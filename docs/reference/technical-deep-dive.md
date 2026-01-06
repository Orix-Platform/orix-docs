# Technical Deep-Dive: Orix Platform Implementation

**Audience:** Technical developers implementing with Orix
**Purpose:** Implementation details, code patterns, and architectural deep-dive
**Prerequisites:** Understanding of C#, ECS, and distributed systems

---

## Table of Contents

1. [The Five Laws Explained](#the-five-laws-explained)
2. [Layer Architecture](#layer-architecture)
3. [Code Examples by Product](#code-examples-by-product)
4. [Integration Patterns](#integration-patterns)
5. [Performance Considerations](#performance-considerations)
6. [Common Pitfalls](#common-pitfalls)
7. [Developer Workflow](#developer-workflow)

---

## The Five Laws Explained

### Law 1: DETERMINISM

**Same inputs → identical outputs, always**

Every computation must produce bit-identical results across all platforms, all compiler versions, all runs.

#### Implementation Requirements

```csharp
// ❌ FORBIDDEN - Platform-dependent floating-point
public float CalculateDistance(Vector3 a, Vector3 b)
{
    return (a - b).magnitude; // Different results on x86 vs ARM
}

// ✅ REQUIRED - Deterministic fixed-point
public DFixed64 CalculateDistance(DVector3 a, DVector3 b)
{
    return (a - b).Magnitude; // Identical results everywhere
}
```

**Forbidden Patterns:**
- `float`, `double` → Use `DFixed64`
- `DateTime.Now` → Use `Tick`
- `System.Random` → Use `OrixRandom(seed)`
- `Dictionary<K,V>` → Use `UnsafeMap<K,V>` (ordered iteration)
- `HashSet<T>` → Use `UnsafeSet<T>` (ordered iteration)
- `async/await` in simulation → Single-threaded tick processing

### Law 2: DISCRETENESS

**All values quantized as Q32.32 fixed-point**

DFixed64 is a 64-bit signed fixed-point number with 32 bits for the integer part and 32 bits for the fractional part.

```csharp
// Q32.32 format: Value = Raw / 2^32
public readonly struct DFixed64
{
    public readonly long Raw; // Internal representation

    // Range: -2,147,483,648.0 to +2,147,483,647.999999999767
    // Precision: ~0.00000000023 (2^-32)
}
```

**Real Implementation (Atom.Math/DFixed64.cs):**

```csharp
// Arithmetic uses 128-bit intermediate for precision
public static DFixed64 operator *(DFixed64 left, DFixed64 right)
{
    Int128 product = (Int128)left.Raw * right.Raw;
    Int128 result128 = product >> 32; // Shift by fractional bits

    // Saturate on overflow
    if (result128 > long.MaxValue) return MaxValue;
    if (result128 < long.MinValue) return MinValue;

    return new((long)result128);
}

// Trigonometry via CORDIC + lookup table (fully deterministic)
public static DFixed64 Sin(DFixed64 angle)
{
    return new(SinLookup(angle.Raw)); // Precomputed table, no Math.Sin
}
```

### Law 3: CAUSALITY

**Time = Ticks only, no wall-clock in simulation**

Wall-clock time (`DateTime.Now`, `Stopwatch`, etc.) introduces non-determinism. Simulation time advances in discrete ticks.

```csharp
// Atom.Primitives/Tick.cs
public readonly struct Tick : IEquatable<Tick>, IComparable<Tick>
{
    public readonly uint Value; // Monotonic counter

    public Tick Next() => new(Value + 1); // Saturates at MaxValue

    public static Tick operator +(Tick tick, TickDelta delta) { /*...*/ }
    public static TickDelta operator -(Tick a, Tick b) { /*...*/ }
}
```

**Usage:**

```csharp
public class MovementSystem : ISystem
{
    public void Update(Tick currentTick)
    {
        // ❌ FORBIDDEN
        var elapsed = DateTime.Now - _startTime;

        // ✅ REQUIRED
        var ticksPassed = currentTick - _startTick;
        var deltaFixed = DFixed64.FromInt((int)ticksPassed.Value);
    }
}
```

### Law 4: SCHEMA AUTHORITY

**All types from .axion schemas**

Every data type that crosses a boundary (API, storage, network) MUST originate from an Axion schema.

**Example Schema (schemas/flux/simulation.axion):**

```axion
namespace flux.simulation

entity EntityInfo {
    @key
    id: uuid,

    state: EntityState = Pending,
    created_tick: int64,
    destroyed_tick: int64?,
    archetype_id: int32,
    generation: int32 = 0
}

enum EntityState {
    Pending = 0,
    Active = 1,
    Destroying = 2,
    Destroyed = 3
}
```

**Generated Code (auto-generated, never modify):**

```csharp
[AxionGenerated(SchemaHash = "abc123...")]
public partial class EntityInfo
{
    public Guid Id { get; set; }
    public EntityState State { get; set; }
    public long CreatedTick { get; set; }
    public long? DestroyedTick { get; set; }
    public int ArchetypeId { get; set; }
    public int Generation { get; set; }
}
```

### Law 5: TRACEABILITY

**Every claim needs reproducible evidence**

Code without tests is untrusted. Performance claims need benchmarks. Correctness claims need formal verification.

```csharp
// tests/Atom.Tests/MathScenarios.cs
[ArbiterScenario(Seed = 42, Category = "Math.DFixed64")]
public void DFixed64_Trigonometry_PythagoreanIdentity()
{
    // sin²(x) + cos²(x) = 1 for all x
    var angle = DFixed64.Pi / DFixed64.FromInt(4); // 45 degrees
    var (sin, cos) = DFixed64.SinCos(angle);

    var result = sin * sin + cos * cos;
    Assert.ApproxEqual(DFixed64.One, result, DFixed64.FromDouble(0.0002));
}
```

---

## Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         PACKS                                │
│                    (Domain Extensions)                       │
│                  Gaming | Finance | DeFi | IoT               │
├─────────────────────────────────────────────────────────────┤
│  AIRLOCK  │  FLUX  │  NEXUS  │  ECHO  │  LUMEN  │  ARBITER  │
│ (Secrets) │ (ECS)  │ (Net)   │ (Time) │ (Obs)   │ (Test)    │
├─────────────────────────────────────────────────────────────┤
│                        LATTICE                               │
│      DB | ORM | Chronicle | QL | CRDT | Search | Crypto     │
├─────────────────────────────────────────────────────────────┤
│                         AXION                                │
│         Language | Fusion (Codegen) | Data | Runtime        │
├─────────────────────────────────────────────────────────────┤
│                          ATOM                                │
│     Math | Collections | Serialization | Primitives         │
│    DFixed64 | UnsafeMap | BitWriter | Tick | Entity         │
└─────────────────────────────────────────────────────────────┘
```

### Dependency Rules

1. **Lower layers MUST NOT depend on higher layers**
2. **Horizontal peers MAY depend on each other with explicit declaration**
3. **All products depend on Atom (foundation)**
4. **All products that use schemas depend on Axion**

### Layer Responsibilities

| Layer | Responsibility | Key Concepts |
|-------|----------------|--------------|
| **Atom** | Deterministic primitives | DFixed64, Tick, Entity, UnsafeMap |
| **Axion** | Schema definition & codegen | .axion files → C# types |
| **Lattice** | Storage & time-travel | Chronicle snapshots, CRDT merging |
| **Flux** | Simulation runtime | ECS World, tick processing |
| **Nexus** | Networking | Delta compression, state sync |
| **Echo** | Recording & replay | Time-travel debugging |
| **Lumen** | Observability | Logging, metrics, tracing |
| **Arbiter** | Verification | Deterministic tests, benchmarks |

---

## Code Examples by Product

### ATOM: Foundation Primitives

#### DFixed64: Deterministic Arithmetic

```csharp
// src/atom/Atom.Math/DFixed64.cs
using Atom.Math;

// Creation
var a = DFixed64.FromInt(10);
var b = DFixed64.FromDouble(3.14159); // Boundary only, not in simulation
var c = DFixed64.FromRaw(0x1_0000_0000L); // Raw Q32.32 value = 1.0

// Arithmetic (all saturating)
var sum = a + b;      // Addition
var product = a * b;  // Multiplication (uses Int128 intermediate)
var quotient = a / b; // Division
var sqrt = DFixed64.Sqrt(a); // Square root (CORDIC-based)

// Trigonometry (lookup table + CORDIC)
var angle = DFixed64.Pi / DFixed64.FromInt(4); // 45°
var (sin, cos) = DFixed64.SinCos(angle);
var tan = DFixed64.Tan(angle);

// Vector math
var v1 = new DVector3(DFixed64.One, DFixed64.FromInt(2), DFixed64.FromInt(3));
var v2 = new DVector3(DFixed64.FromInt(4), DFixed64.FromInt(5), DFixed64.FromInt(6));
var dot = DVector3.Dot(v1, v2);
var cross = DVector3.Cross(v1, v2);
var normalized = v1.Normalized;
```

#### UnsafeMap: Deterministic Collections

```csharp
// src/atom/Atom.Collections/UnsafeMap.cs
using Atom.Collections;

// Create with allocator (no GC pressure)
var map = new UnsafeMap<int, DFixed64>(16, Allocator.Persistent);

// Add/update
map.Add(1, DFixed64.FromInt(100));
map[2] = DFixed64.FromInt(200);
map.TryAdd(3, DFixed64.FromInt(300));

// Query
if (map.TryGetValue(1, out var value))
{
    Console.WriteLine($"Found: {value}");
}

// Iterate in INSERTION ORDER (deterministic)
foreach (var kvp in map)
{
    Console.WriteLine($"{kvp.Key} = {kvp.Value}");
}

// Cleanup
map.Dispose();
```

#### OrixRandom: Seeded Randomness

```csharp
// src/atom/Atom.Math/OrixRandom.cs
using Atom.Math;

// Create with seed (deterministic)
var rng = new OrixRandom(42);

// Generate values
var intValue = rng.NextInt(100);           // [0, 100)
var fixedValue = rng.NextFixed();           // [0.0, 1.0)
var rangeValue = rng.NextFixed(DFixed64.FromInt(10), DFixed64.FromInt(20)); // [10, 20)

// Geometric distributions
var unitVec2 = rng.NextUnitVector2();       // Unit circle
var unitVec3 = rng.NextUnitVector3();       // Unit sphere
var insideCircle = rng.NextInsideCircle();  // Random point in circle
var insideSphere = rng.NextInsideSphere();  // Random point in sphere

// Shuffle arrays (Fisher-Yates)
var items = new[] { 1, 2, 3, 4, 5 };
rng.Shuffle(items);
```

### AXION: Schema-Driven Development

#### Define Schema

```axion
// schemas/game/player.axion
namespace game.player

entity Player {
    @key
    id: uuid,

    @indexed
    username: string,

    position: Vector3,
    health: float64,
    score: int32 = 0,
    inventory: list<uuid>?,

    @encrypted(group: "sensitive")
    session_token: bytes
}

entity Vector3 {
    x: float64,
    y: float64,
    z: float64
}
```

#### Generate Code

```bash
orix axion export csharp schemas/game/player.axion -o src/Game/Game.Generated
```

#### Use Generated Types

```csharp
// src/Game/Services/PlayerService.cs
using Game.Generated;
using Atom.Primitives;

public class PlayerService
{
    private readonly EntityAllocator _entityAllocator;

    public Player CreatePlayer(string username, DVector3 position)
    {
        return new Player
        {
            Id = _entityAllocator.Allocate().PackedValue, // NOT Guid.NewGuid()
            Username = username,
            Position = new Vector3
            {
                X = position.X.ToDouble(), // Boundary conversion
                Y = position.Y.ToDouble(),
                Z = position.Z.ToDouble()
            },
            Health = 100.0,
            Score = 0,
            Inventory = new List<Guid>()
        };
    }
}
```

### FLUX: ECS Simulation Runtime

#### World Creation

```csharp
// src/flux/Flux/Core/World.cs
using Flux.Core;
using Atom.Primitives;

// Create world with seed
var world = new World(seed: 42, maxEntities: 100000);

// Register component types
world.RegisterComponent<Position>();
world.RegisterComponent<Velocity>();
world.RegisterComponent<Health>();

// Add systems
world.AddSystem(new MovementSystem());
world.AddSystem(new CombatSystem());

// Start simulation
world.Start();
```

#### Entity Lifecycle

```csharp
// tests/Flux.Tests/TickScenarios.cs
[ArbiterScenario(Seed = 42, Category = "Flux.Lifecycle")]
public void EntityLifecycle_CreateDestroy_IsDeterministic()
{
    using var world = new World(42);

    // Create entities (deterministic allocation)
    var entity1 = world.CreateEntity();
    var entity2 = world.CreateEntity();
    var entity3 = world.CreateEntity();

    Assert.True(world.IsValid(entity1));
    Assert.Equal(3, world.EntityCount);

    // Destroy one entity
    world.DestroyEntity(entity2);
    Assert.False(world.IsValid(entity2));
    Assert.Equal(2, world.EntityCount);

    // Create new entity - reuses freed slot with incremented generation
    var entity4 = world.CreateEntity();
    Assert.True(world.IsValid(entity4));
    Assert.Equal(entity2.Index, entity4.Index); // Same index
    Assert.Equal(entity2.Generation + 1, entity4.Generation); // Incremented
}
```

#### Tick Processing

```csharp
// Custom system implementation
public class MovementSystem : ISystem
{
    public void Initialize(World world) { }

    public void Update(Tick currentTick)
    {
        // Query entities with Position and Velocity components
        foreach (var entity in _entities)
        {
            ref var pos = ref GetComponent<Position>(entity);
            ref var vel = ref GetComponent<Velocity>(entity);

            // Update position (deterministic fixed-point math)
            pos.X += vel.X;
            pos.Y += vel.Y;
            pos.Z += vel.Z;
        }
    }

    public void Shutdown() { }
}

// Main loop
world.Start();

while (running)
{
    world.ProcessTick(); // Single deterministic tick

    // Optional: batch processing
    // world.ProcessTicks(4); // Process 4 ticks at once
}

world.Stop();
```

### NEXUS: Networking & State Sync

```csharp
// Conceptual API (Nexus is being implemented)
using Nexus;

// Server: Broadcast state deltas
var server = new NetworkServer(port: 7777);
server.OnClientConnected += client =>
{
    // Send full state snapshot to new client
    var snapshot = world.CreateSnapshot();
    client.SendSnapshot(snapshot);
};

server.OnTick += (tick) =>
{
    // Broadcast delta since last acknowledged tick
    var delta = world.CreateDelta(since: tick);
    server.BroadcastDelta(delta);
};

// Client: Apply deltas to local world
var client = new NetworkClient("localhost", 7777);
client.OnSnapshot += snapshot =>
{
    world.ApplySnapshot(snapshot);
};

client.OnDelta += delta =>
{
    world.ApplyDelta(delta);
};
```

### ECHO: Recording & Replay

```csharp
// Conceptual API (Echo is being implemented)
using Echo;

// Recording
var recorder = new WorldRecorder(world);
recorder.Start();

// Simulation runs...
for (int i = 0; i < 1000; i++)
{
    world.ProcessTick();
}

recorder.Stop();
recorder.SaveToFile("replay.orix");

// Playback
var playback = new WorldPlayback("replay.orix");
playback.Seek(Tick.FromRaw(500)); // Jump to tick 500
playback.StepBackward();          // Go back one tick
playback.StepForward();           // Advance one tick

// Time-travel debugging
var stateAtTick500 = playback.GetWorldState(Tick.FromRaw(500));
var entityAtTick500 = playback.GetEntityState(entityId, Tick.FromRaw(500));
```

### LUMEN: Observability

```csharp
// Conceptual API (Lumen is being implemented)
using Lumen;

// Structured logging
var logger = new StructuredLogger("GameSimulation");
logger.Info("World started", new { Seed = 42, MaxEntities = 100000 });
logger.Warn("Entity count high", new { Count = 95000, Threshold = 90000 });

// Metrics
var metrics = new MetricsCollector();
metrics.Increment("entities.created");
metrics.Gauge("entities.active", world.EntityCount);
metrics.Histogram("tick.duration_ms", tickDurationMs);

// Tracing
using var span = Tracer.StartSpan("ProcessTick");
span.SetAttribute("tick", currentTick.Value);
world.ProcessTick();
span.End();
```

### ARBITER: Testing & Verification

```csharp
// tests/Atom.Tests/MathScenarios.cs
using Arbiter;

[ArbiterScenario(Seed = 42, Category = "Math.DFixed64")]
public void DFixed64_Addition_IsCommutative()
{
    var a = DFixed64.FromInt(10);
    var b = DFixed64.FromInt(20);

    Assert.Equal(a + b, b + a);
}

[ArbiterScenario(Seed = 42, Category = "Math.DFixed64")]
public void DFixed64_Multiplication_IsAssociative()
{
    var a = DFixed64.FromInt(2);
    var b = DFixed64.FromInt(3);
    var c = DFixed64.FromInt(4);

    Assert.Equal((a * b) * c, a * (b * c));
}

[ArbiterScenario(Seed = 123, Category = "Flux.Determinism")]
public void World_SameSeed_SameResults()
{
    // Run two simulations with same seed
    var world1 = new World(42);
    var world2 = new World(42);

    for (int i = 0; i < 100; i++)
    {
        world1.ProcessTick();
        world2.ProcessTick();
    }

    // Verify identical state
    Assert.Equal(world1.GetStateHash(), world2.GetStateHash());
}
```

### LATTICE: Chronicle Time-Travel Queries

```csharp
// tests/Lattice.Tests/ChronicleScenarios.cs
using Lattice.Chronicle;

[ArbiterScenario(Seed = 42, Category = "Chronicle.StateHasher")]
public void Hash_SameData_SameResult()
{
    var data = new byte[] { 1, 2, 3, 4, 5 };

    var hash1 = StateHasher.Hash(data);
    var hash2 = StateHasher.Hash(data);

    Assert.Equal(hash1, hash2, "Same data should produce same hash");
}

[ArbiterScenario(Seed = 42, Category = "Chronicle.StateHasher")]
public void IncrementalHasher_MatchesDirectHash()
{
    var data = new byte[] { 1, 2, 3, 4, 5, 6, 7, 8 };
    var directHash = StateHasher.Hash(data);

    var incremental = StateHasher.CreateIncremental();
    incremental.Add(new byte[] { 1, 2, 3, 4 });
    incremental.Add(new byte[] { 5, 6, 7, 8 });
    var incrementalHash = incremental.GetHash();

    Assert.Equal(directHash, incrementalHash);
}
```

### AIRLOCK: Secret Storage

```csharp
// Conceptual API (Airlock is being implemented)
using Airlock;

// Store secret
var vault = new SecretVault(masterPassword: "strong-password");
vault.Set("api-key", "sk-1234567890abcdef");
vault.Set("db-password", "super-secret-password");

// Retrieve secret
var apiKey = vault.Get("api-key");

// Share secret with another device
var shareToken = vault.CreateShareToken("api-key", expiresIn: TimeSpan.FromHours(1));
// On other device:
vault.AcceptShare(shareToken, masterPassword: "strong-password");

// Audit log
var auditLog = vault.GetAuditLog();
foreach (var entry in auditLog)
{
    Console.WriteLine($"{entry.Timestamp}: {entry.Action} on {entry.SecretId}");
}
```

---

## Integration Patterns

### How Flux Uses Atom Primitives

```csharp
// Flux depends on Atom for deterministic primitives
namespace Flux.Core;

using Atom.Math;         // DFixed64, OrixRandom, DVector3
using Atom.Primitives;   // Tick, Entity, TickDelta
using Atom.Collections;  // UnsafeMap, UnsafeSet, UnsafeList

public sealed class World : IDisposable
{
    private readonly OrixRandom _random;      // Seeded RNG from Atom
    private readonly EntityAllocator _entityAllocator; // Entity management
    private Tick _currentTick;                // Time tracking via Tick

    public World(ulong seed, int maxEntities = 100000)
    {
        _random = new OrixRandom(seed);       // Atom primitive
        _currentTick = Tick.Zero;              // Atom primitive
        // ...
    }

    public void ProcessTick()
    {
        _currentTick = _currentTick.Next();    // Tick arithmetic
        _scheduler.Update(_currentTick);
    }
}
```

### How Nexus Uses Axion-Generated Types

```csharp
// Nexus uses Axion schemas for network messages
namespace Nexus;

using Nexus.Generated; // Types from schemas/nexus/network.axion

public class NetworkServer
{
    public void BroadcastDelta(WorldDelta delta)
    {
        // delta is an Axion-generated type with built-in serialization
        var bytes = delta.Serialize(); // Efficient binary format

        foreach (var client in _clients)
        {
            client.Send(bytes);
        }
    }

    public void OnReceive(byte[] bytes)
    {
        // Deserialize using Axion runtime
        var delta = WorldDelta.Deserialize(bytes);
        ApplyDelta(delta);
    }
}
```

### How Echo Integrates with Flux Ticks

```csharp
// Echo records Flux ticks for replay
namespace Echo;

using Flux.Core;
using Atom.Primitives;

public class WorldRecorder
{
    private readonly World _world;
    private readonly List<TickSnapshot> _snapshots;

    public WorldRecorder(World world)
    {
        _world = world;
        _snapshots = new List<TickSnapshot>();
    }

    public void OnTick(Tick tick)
    {
        // Capture world state at this tick
        var snapshot = new TickSnapshot
        {
            Tick = tick,
            StateHash = ComputeStateHash(_world),
            EntityCount = _world.EntityCount,
            Entities = CaptureEntities(_world)
        };

        _snapshots.Add(snapshot);
    }
}
```

### How Lumen Correlates with Simulation

```csharp
// Lumen traces simulation events
namespace Lumen;

using Atom.Primitives;
using Flux.Core;

public class SimulationTracer
{
    public void TraceTickProcessing(World world)
    {
        using var span = Tracer.StartSpan("ProcessTick");

        span.SetAttribute("tick", world.CurrentTick.Value);
        span.SetAttribute("entity_count", world.EntityCount);
        span.SetAttribute("seed", world.Random.State);

        var startTime = DateTime.UtcNow; // Ambient time for tracing only
        world.ProcessTick();
        var endTime = DateTime.UtcNow;

        span.SetAttribute("duration_ms", (endTime - startTime).TotalMilliseconds);
        span.End();
    }
}
```

---

## Performance Considerations

### Hot Path Rules (No Allocations)

```csharp
// ❌ FORBIDDEN - Allocates on every call
public void UpdateEntities()
{
    var positions = new List<Position>(); // HEAP ALLOCATION
    foreach (var entity in entities)
    {
        positions.Add(GetPosition(entity));
    }
}

// ✅ REQUIRED - Pre-allocated, reused
private readonly UnsafeList<Position> _positionsBuffer;

public void UpdateEntities()
{
    _positionsBuffer.Clear(); // Reuse existing capacity
    foreach (var entity in entities)
    {
        _positionsBuffer.Add(GetPosition(entity));
    }
}
```

### Entity Allocation Pooling

```csharp
// Flux uses generation counter for safe reuse
public readonly struct Entity
{
    private readonly uint _packed; // 24 bits index, 8 bits generation

    public int Index => (int)(_packed & 0x00FFFFFF);
    public byte Generation => (byte)((_packed >> 24) & 0xFF);

    // When destroying entity, increment generation
    public Entity NextGeneration()
    {
        byte nextGen = (byte)((Generation + 1) & 0xFF);
        return new Entity(Index, nextGen);
    }
}

// Allocator reuses slots with incremented generation
public class EntityAllocator
{
    private readonly Entity[] _entities;
    private readonly Queue<int> _freeIndices;

    public void Free(Entity entity)
    {
        var nextGen = entity.NextGeneration();
        _entities[entity.Index] = nextGen; // Store new generation
        _freeIndices.Enqueue(entity.Index); // Mark index as free
    }

    public Entity Allocate()
    {
        if (_freeIndices.TryDequeue(out var index))
        {
            return _entities[index]; // Reuse with new generation
        }

        // Allocate new slot
        var newIndex = _nextIndex++;
        var newEntity = new Entity(newIndex, generation: 1);
        _entities[newIndex] = newEntity;
        return newEntity;
    }
}
```

### Delta Compression Trade-offs

**Snapshot vs Delta:**
- Snapshot: Full state, large size, fast to apply
- Delta: Only changes, small size, accumulates error

**Strategy (Nexus):**
```csharp
public class StateReplication
{
    private const int SnapshotInterval = 300; // Every 300 ticks

    public void OnTick(Tick tick)
    {
        if (tick.Value % SnapshotInterval == 0)
        {
            // Send full snapshot periodically
            BroadcastSnapshot(CreateSnapshot());
        }
        else
        {
            // Send delta for intermediate ticks
            BroadcastDelta(CreateDelta(since: _lastSnapshotTick));
        }
    }
}
```

### Snapshot Frequency in Chronicle

```csharp
// Lattice.Chronicle configuration
config WorldConfig {
    // Snapshot interval in ticks (0 = disabled)
    snapshot_interval: int32 = 300,

    // Trade-off:
    // - Higher interval = less storage, slower queries
    // - Lower interval = more storage, faster queries
}

// Query performance
public WorldState GetStateAtTick(Tick tick)
{
    // Find nearest snapshot before tick
    var snapshot = FindNearestSnapshot(tick);

    // Apply deltas from snapshot to tick
    for (var t = snapshot.Tick; t < tick; t = t.Next())
    {
        var delta = GetDelta(t);
        snapshot.Apply(delta);
    }

    return snapshot;
}
```

---

## Common Pitfalls

### 1. Using float instead of DFixed64

```csharp
// ❌ WRONG - Platform-dependent
public struct Position
{
    public float X, Y, Z;
}

// ✅ CORRECT - Deterministic
public struct Position
{
    public DFixed64 X, Y, Z;
}
```

### 2. Forgetting Seed in Tests

```csharp
// ❌ WRONG - Non-deterministic test
[Test]
public void TestWorldSimulation()
{
    var world = new World(seed: (ulong)DateTime.Now.Ticks); // RANDOM SEED
    // Test may pass/fail randomly
}

// ✅ CORRECT - Deterministic test
[ArbiterScenario(Seed = 42, Category = "Flux.World")]
public void TestWorldSimulation()
{
    var world = new World(seed: 42); // FIXED SEED
    // Test always behaves the same
}
```

### 3. Dictionary instead of UnsafeMap

```csharp
// ❌ WRONG - Unordered iteration
var entities = new Dictionary<int, Entity>();
foreach (var kvp in entities) // ORDER IS UNDEFINED
{
    ProcessEntity(kvp.Value);
}

// ✅ CORRECT - Ordered iteration
var entities = new UnsafeMap<int, Entity>(16, Allocator.Persistent);
foreach (var kvp in entities) // INSERTION ORDER
{
    ProcessEntity(kvp.Value);
}
```

### 4. DateTime in Simulation Code

```csharp
// ❌ WRONG - Ambient time in simulation
public void Update()
{
    var now = DateTime.Now; // NON-DETERMINISTIC
    if (now - _lastSpawn > TimeSpan.FromSeconds(1))
    {
        SpawnEnemy();
    }
}

// ✅ CORRECT - Tick-based timing
public void Update(Tick currentTick)
{
    var ticksSinceSpawn = currentTick - _lastSpawnTick;
    if (ticksSinceSpawn.Value >= 60) // 60 ticks = 1 second at 60 TPS
    {
        SpawnEnemy();
    }
}
```

### 5. Modifying Generated Code

```csharp
// ❌ WRONG - Modifying generated file
// src/Game/Game.Generated/Player.cs
[AxionGenerated(SchemaHash = "abc123...")]
public partial class Player
{
    // DON'T ADD CODE HERE - will be overwritten on next generation
    public void CustomMethod() { } // WILL BE LOST
}

// ✅ CORRECT - Extend via partial class
// src/Game/Extensions/PlayerExtensions.cs
public partial class Player
{
    // Safe to add extensions here
    public void CustomMethod()
    {
        // Custom logic
    }
}
```

---

## Developer Workflow

### Complete Feature Implementation

```bash
# 1. Define schema
touch schemas/game/weapon.axion
```

```axion
namespace game.weapon

entity Weapon {
    @key
    id: uuid,

    name: string,
    damage: int32,
    range: float64,
    ammo: int32 = 100,
    type: WeaponType
}

enum WeaponType {
    Melee = 0,
    Ranged = 1,
    Magic = 2
}
```

```bash
# 2. Generate code
orix axion export csharp schemas/game/weapon.axion -o src/Game/Game.Generated

# 3. Implement using generated types
# src/Game/Systems/CombatSystem.cs
```

```csharp
using Game.Generated;
using Flux.Core;
using Atom.Math;

public class CombatSystem : ISystem
{
    private readonly UnsafeMap<Entity, Weapon> _weapons;

    public void Update(Tick currentTick)
    {
        foreach (var kvp in _weapons)
        {
            var entity = kvp.Key;
            var weapon = kvp.Value;

            if (weapon.Ammo > 0 && ShouldFire(entity))
            {
                Fire(entity, weapon);
                weapon.Ammo--;
            }
        }
    }
}
```

```bash
# 4. Test with Arbiter scenarios
# tests/Game.Tests/CombatScenarios.cs
```

```csharp
[ArbiterScenario(Seed = 42, Category = "Combat")]
public void Weapon_FiringReducesAmmo()
{
    var weapon = new Weapon
    {
        Id = Guid.NewGuid(),
        Name = "Pistol",
        Damage = 10,
        Range = 50.0,
        Ammo = 100,
        Type = WeaponType.Ranged
    };

    var initialAmmo = weapon.Ammo;
    Fire(weapon);

    Assert.Equal(initialAmmo - 1, weapon.Ammo);
}
```

```bash
# 5. Verify all checks pass
orix check schemas           # V0: Schema validation
dotnet build /warnaserror    # V1: Zero warnings
orix arbiter run             # V2: All tests pass
orix arbiter determinism     # V3: Determinism verified

# 6. Commit with proper format
git add .
git commit -m "feat: add weapon system

Implements weapon firing with ammo consumption.
Uses Axion schema for weapon definition.

Refs: #123"
```

---

## Summary

**The Four Laws in Practice:**

1. **DETERMINISM**: Use DFixed64, Tick, OrixRandom, UnsafeMap
2. **DISCRETENESS**: Q32.32 fixed-point for all numeric values
3. **CAUSALITY**: Tick-based time, no DateTime in simulation
4. **SCHEMA AUTHORITY**: All boundary types from .axion schemas
5. **TRACEABILITY**: Test with Arbiter, benchmark with evidence

**Layer Dependencies:**
```
PACKS → FLUX|NEXUS|ECHO|LUMEN|ARBITER|AIRLOCK → LATTICE → AXION → ATOM
```

**Verification Protocol:**
```bash
V0: orix check schemas
V1: dotnet build /warnaserror
V2: orix arbiter run
V3: orix arbiter determinism
```

**Hot Path Rules:**
- Pre-allocate collections
- Use UnsafeMap/UnsafeSet/UnsafeList
- No LINQ in simulation code
- Entity pooling with generation counter

---

## Related Documents

- [Atom Foundation](./01-foundation-atom.md) - DFixed64 and deterministic primitives
- [Axion Schema](./02-schema-axion.md) - Schema language and code generation
- [Flux Simulation](./07-simulation-flux.md) - ECS runtime details
- [Arbiter Testing](./11-testing-arbiter.md) - Testing framework
- [Executive Overview](./00-executive-overview.md) - Platform overview and Five Laws

---

**End of Technical Deep-Dive**
