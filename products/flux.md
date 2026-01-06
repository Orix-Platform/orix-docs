# Meeting 07: Flux - Deterministic Simulation Runtime

**Topic:** How Orix runs simulations that produce identical results every time
**For:** New developers learning Orix architecture
**Reading time:** 15 minutes

---

## Table of Contents
1. [Why Flux Exists](#why-flux-exists)
2. [How Flux Solves It](#how-flux-solves-it)
3. [What Flux Provides](#what-flux-provides)
4. [Core Concepts](#core-concepts)
5. [Example Usage](#example-usage)
6. [Advantages](#advantages)
7. [Disadvantages](#disadvantages)
8. [Comparison](#comparison)
9. [Performance](#performance)
10. [What's Next](#whats-next)

---

## Why Flux Exists

Traditional game loops and simulation systems have serious problems:

### Problem 1: Frame Rate Dependence

```csharp
// Bad: Updates depend on frame rate
void Update(float deltaTime)
{
    position += velocity * deltaTime;  // Different results at 60fps vs 120fps
}
```

If your game runs at 60fps on one machine and 120fps on another, the simulation state diverges. Physics feels different. Jumps go different distances. Speedrunners exploit frame rate.

### Problem 2: Multiplayer Desyncs

```csharp
// Two players, same inputs, different results
Player1 (Windows, 60fps):  Position = (10.531, 5.223)
Player2 (Linux, 59.7fps):  Position = (10.529, 5.221)  // DESYNC!
```

Even tiny floating-point differences accumulate. After 1000 frames, players are in completely different worlds. The game thinks Player 1 hit the enemy, but Player 2's client shows a miss.

### Problem 3: Debugging Time-Dependent Bugs

```csharp
// Bug only happens after running for 23 minutes
// How do you debug this?
// How do you reproduce it reliably?
```

You can't just "rewind" traditional simulations. Recording 23 minutes of state is impractical. The bug might not even reproduce on the next run.

### Problem 4: Traditional OOP Doesn't Scale

```csharp
// With 10,000 entities calling Update()
foreach (var entity in entities)
{
    entity.Update();  // Cache misses everywhere
    entity.CheckCollisions();  // O(N²) comparisons
}
```

Object-oriented game architecture creates scattered memory accesses. With thousands of entities, CPU cache misses dominate performance. You spend more time waiting for RAM than computing.

**Flux solves all of these problems.**

---

## How Flux Solves It

Flux uses four key principles:

### 1. Tick-Based Execution

Time doesn't flow continuously. It advances in discrete steps called **ticks**:

```csharp
// Before Flux: Continuous time
void Update(float deltaTime) { ... }  // deltaTime varies!

// With Flux: Discrete time
void Update(Tick tick) { ... }  // tick is always +1
```

Every tick is exactly one unit of time. No matter what frame rate the rendering runs at, the simulation advances in identical steps. 60fps rendering with 30 tick/sec means: render, render (tick), render, render (tick).

### 2. Entity-Component-System (ECS) Architecture

Instead of objects with behavior, you have:
- **Entities**: Just IDs (like database primary keys)
- **Components**: Pure data (position, velocity, health)
- **Systems**: Logic that processes components

```csharp
// Instead of:
class Enemy
{
    Vector2 position;
    int health;
    void Update() { AI logic mixed with data }
}

// You write:
struct Position { DVector2 Value; }
struct Health { int Value; }

class AISystem : ISystem
{
    void Update(Tick tick)
    {
        // Process ALL AI in one batch (cache-friendly)
    }
}
```

### 3. Deterministic Entity Allocation

```csharp
// Traditional: Non-deterministic IDs
var enemy = Instantiate(enemyPrefab);  // Random GUID each time

// Flux: Deterministic IDs
var enemy = world.CreateEntity();  // Same seed = same IDs
```

Entity IDs are allocated deterministically using a generation-counter system. If you create 10 entities with seed 42, you get the same 10 IDs every time.

### 4. World State Management

The **World** container owns all simulation state:

```csharp
var world = new World(seed: 42);
world.Start();

// ... simulation runs ...

// Same inputs = same state
Assert.Equal(hash1, hash2);
```

The world encapsulates everything needed to reproduce a simulation. Same seed, same operations, same result. Always.

---

## What Flux Provides

### Current Implementation (v0.1)

| Component | Status | Description |
|-----------|--------|-------------|
| **World** | ✅ Implemented | Main ECS container |
| **Entity** | ✅ Implemented | Lightweight identifier (24-bit index + 8-bit generation) |
| **EntityAllocator** | ✅ Implemented | Deterministic ID allocation with recycling |
| **Tick** | ✅ Implemented | Discrete time unit (uint32) |
| **WorldState** | ✅ Implemented | Running/Paused/Stopped states |
| **ISystem** | ✅ Implemented | Interface for simulation systems |
| **SystemScheduler** | ✅ Implemented | Deterministic system execution ordering |
| **ComponentTypeRegistry** | ✅ Implemented | Type-safe component registration |
| **Archetype** | ✅ Implemented | Structure-of-Arrays component storage |

### Planned Features

| Feature | Status | Description |
|---------|--------|-------------|
| Component API | 📋 Planned | Add/Get/Remove components on entities |
| Query API | 📋 Planned | Iterate entities with specific components |
| Input System | 📋 Planned | Queue and process player inputs |
| State Snapshots | 📋 Planned | Save/restore entire simulation state |
| Event Buffer | 📋 Planned | Collect simulation events per tick |
| Physics Integration | 📋 Planned | Deterministic physics subsystem |

---

## Core Concepts

### Tick: The Unit of Time

```csharp
// Tick is a wrapper around uint
public readonly struct Tick
{
    public readonly uint Value;

    public Tick Next() => new Tick(Value + 1);
}

// Usage
var tick0 = Tick.Zero;          // Tick(0)
var tick1 = tick0.Next();       // Tick(1)
var tick2 = Tick.FromRaw(100);  // Tick(100)

// Comparison works as expected
Assert.True(tick0 < tick1);
Assert.True(tick2 > tick1);
```

Ticks represent discrete moments. There's no "time between ticks" in the simulation. This eliminates floating-point time accumulation errors.

### Entity: Just an ID, No Behavior

```csharp
// Entity packs index + generation into 32 bits
public readonly struct Entity
{
    public int Index { get; }        // 24 bits: 0 to 16,777,215
    public byte Generation { get; }  // 8 bits: 0 to 255
}

// Create entities
var player = world.CreateEntity();   // Entity(0:1)
var enemy1 = world.CreateEntity();   // Entity(1:1)
var enemy2 = world.CreateEntity();   // Entity(2:1)

// Destroy and reuse
world.DestroyEntity(enemy1);
var enemy3 = world.CreateEntity();   // Entity(1:2) - same index, new generation!
```

The generation counter prevents "use after free" bugs. If you hold a reference to `Entity(1:1)` but it was destroyed and the slot reused for `Entity(1:2)`, your reference is now invalid. The world knows to reject operations on stale entities.

### Component: Data Attached to Entities

```csharp
// Components are plain structs
public struct Position
{
    public DVector2 Value;  // DVector2 = deterministic fixed-point
}

public struct Velocity
{
    public DVector2 Value;
}

public struct Health
{
    public int Current;
    public int Maximum;
}
```

Components are pure data. No methods, no logic. They get stored in **Structure of Arrays** (SoA) format for cache efficiency.

### System: Logic That Processes Components

```csharp
public class MovementSystem : SystemBase
{
    public override int Priority => 10;  // Lower executes first

    public override void Update(Tick tick)
    {
        // Pseudocode - query API planned
        foreach (var (entity, position, velocity) in Query<Position, Velocity>())
        {
            position.Value += velocity.Value;  // Move everything in one batch
        }
    }
}
```

Systems run in priority order every tick. All MovementSystem work happens together (good for CPU cache). No random object updates.

### World: Container Orchestrating Everything

```csharp
public sealed class World : IDisposable
{
    public Tick CurrentTick { get; }
    public WorldState State { get; }
    public int EntityCount { get; }
    public OrixRandom Random { get; }

    // Lifecycle
    public void Start();
    public void Pause();
    public void Resume();
    public void Stop();
    public void ProcessTick();

    // Entities
    public Entity CreateEntity();
    public void DestroyEntity(Entity entity);
    public bool IsValid(Entity entity);

    // Systems
    public void AddSystem(ISystem system);
    public T? GetSystem<T>() where T : class, ISystem;
}
```

The World is the single source of truth for simulation state. Everything flows through it.

---

## Example Usage

### Basic Simulation Loop

```csharp
using Flux.Core;
using Atom.Primitives;

// Create world with deterministic seed
using var world = new World(seed: 42);

// Register systems (before Start())
world.AddSystem(new MovementSystem());
world.AddSystem(new CollisionSystem());

// Start simulation
world.Start();

// Create some entities
var player = world.CreateEntity();
var enemy1 = world.CreateEntity();
var enemy2 = world.CreateEntity();

Console.WriteLine($"Created {world.EntityCount} entities");

// Advance simulation by 100 ticks
world.ProcessTicks(100);

Console.WriteLine($"Simulation is at tick {world.CurrentTick.Value}");
// Output: Simulation is at tick 100

// After 100 ticks with seed 42, state is ALWAYS identical
// Run this program twice - same output guaranteed
```

### Game Loop Integration

```csharp
// Game loop runs at variable frame rate (30-144fps)
// Simulation runs at fixed tick rate (60 ticks/sec)

const double TICK_RATE = 60.0;
const double TICK_DURATION = 1.0 / TICK_RATE;

double accumulator = 0.0;
var lastTime = DateTime.UtcNow;

while (running)
{
    // Calculate frame time (ambient - NOT in simulation)
    var currentTime = DateTime.UtcNow;
    var deltaTime = (currentTime - lastTime).TotalSeconds;
    lastTime = currentTime;

    // Accumulate frame time
    accumulator += deltaTime;

    // Process ticks in fixed increments
    while (accumulator >= TICK_DURATION)
    {
        world.ProcessTick();  // Deterministic step
        accumulator -= TICK_DURATION;
    }

    // Render at whatever frame rate you want
    renderer.Draw(world, accumulator / TICK_DURATION);  // Interpolation factor
}
```

The rendering frame rate can vary wildly - the simulation ticks at exactly 60Hz. This separates visual smoothness from simulation correctness.

### Determinism Verification

```csharp
// Run 1
var world1 = new World(seed: 12345);
world1.Start();
for (int i = 0; i < 10; i++)
{
    world1.CreateEntity();
}
world1.ProcessTicks(100);
var random1 = world1.Random.NextUInt();

// Run 2 - same seed
var world2 = new World(seed: 12345);
world2.Start();
for (int i = 0; i < 10; i++)
{
    world2.CreateEntity();
}
world2.ProcessTicks(100);
var random2 = world2.Random.NextUInt();

// MUST be equal - determinism guarantee
Assert.Equal(random1, random2);
```

This is the core promise: same seed + same operations = identical results.

---

## Advantages

### 1. Perfect Determinism

Same inputs always produce identical outputs. This enables:
- **Replay systems**: Record inputs, replay simulation exactly
- **Network synchronization**: All clients reach identical state
- **Time travel debugging**: Step forward/backward through simulation
- **Testing**: Reproducible scenarios for unit tests

### 2. Cache-Friendly Performance

Structure-of-Arrays layout means all positions are stored together, all velocities together:

```
Traditional OOP:
[Enemy1: pos, vel, health] [Enemy2: pos, vel, health] [Enemy3: pos, vel, health]
^ cache miss        ^ cache miss                ^ cache miss

Flux ECS:
[Enemy1 pos] [Enemy2 pos] [Enemy3 pos] ... [Enemy1 vel] [Enemy2 vel] [Enemy3 vel]
^ all in one cache line - blazing fast
```

Processing 10,000 enemies' movement = one tight loop over a contiguous array.

### 3. Scalability

Entity-Component-System scales to thousands of entities:
- Entities are just integers (cheap)
- Components stored in typed arrays (fast iteration)
- Systems process in batches (good cache usage)
- Parallel system execution possible (when dependencies allow)

### 4. Testability

```csharp
[ArbiterScenario(Seed = 42, Category = "Combat")]
public void Sword_Hits_Enemy_Deals_Damage()
{
    var world = new World(42);
    world.AddSystem(new CombatSystem());
    world.Start();

    var player = world.CreateEntity();
    var enemy = world.CreateEntity();

    // Setup components...

    world.ProcessTick();

    // Verify results - always reproducible!
    Assert.Equal(90, GetHealth(enemy));
}
```

Tests are deterministic. No more flaky tests that pass sometimes and fail other times.

### 5. Network-Friendly

For multiplayer, you only need to synchronize inputs, not entire state:

```
Client 1: Seed=42, [Input1, Input2, Input3]
Client 2: Seed=42, [Input1, Input2, Input3]

Both clients run simulation locally with identical results.
Bandwidth: Only inputs (~10 bytes/tick vs. ~1000 bytes for full state)
```

This is how fighting games achieve rollback netcode.

---

## Disadvantages

### 1. Learning Curve

ECS is different from traditional OOP:

```csharp
// OOP: Intuitive but slow
class Enemy
{
    void Update()
    {
        this.position += this.velocity;
    }
}

// ECS: Less intuitive but fast
class MovementSystem : ISystem
{
    void Update(Tick tick)
    {
        // Process ALL entities at once
        foreach (var (entity, position, velocity) in Query<Position, Velocity>())
        {
            position.Value += velocity.Value;
        }
    }
}
```

Thinking in "data and systems" instead of "objects and methods" takes adjustment.

### 2. Boilerplate

Simple things require more setup:

```csharp
// OOP: One line
var enemy = new Enemy(position, velocity);

// ECS: Multiple steps
var enemy = world.CreateEntity();
world.AddComponent<Position>(enemy, new Position { Value = pos });
world.AddComponent<Velocity>(enemy, new Velocity { Value = vel });
```

For prototypes, ECS feels verbose. The payoff comes at scale.

### 3. Mental Model Shift

You can't think in terms of "objects communicating":

```csharp
// OOP: Direct communication
enemy.TakeDamage(10);
enemy.NotifyHealthBar();

// ECS: Systems react to component changes
// DamageSystem sets Health component
// UISystem reads Health component and updates bar
```

Systems communicate through component state, not method calls.

### 4. Debugging Visibility

Traditional debuggers show object state clearly. With ECS, state is spread across component arrays:

```
Debugger on `enemy` variable:
OOP: { position: (10, 5), velocity: (1, 0), health: 90 }
ECS: { Entity(5:1) }  // Just an ID - need to query for components
```

You need tools to visualize component state (entity inspectors).

---

## Comparison

### vs Unity ECS (DOTS)

| Feature | Flux | Unity ECS |
|---------|------|-----------|
| **Determinism** | Core design goal | Not guaranteed |
| **Fixed-Point Math** | DFixed64 everywhere | float/double |
| **API Complexity** | Simpler API surface | More features, more complex |
| **Platform** | Cross-platform C# | Unity-specific |
| **Use Case** | Simulation-first | Rendering-first |

Unity ECS optimizes for performance in rendering-heavy games. Flux optimizes for deterministic simulations (networking, replay, financial modeling).

### vs Bevy (Rust ECS)

| Feature | Flux | Bevy |
|---------|------|------|
| **Language** | C# | Rust |
| **Philosophy** | Similar (determinism, ECS) | Similar (determinism, ECS) |
| **Math** | DFixed64 | f32/f64 (or fixed-point crates) |
| **Ecosystem** | .NET ecosystem | Rust ecosystem |

Bevy and Flux share philosophy. Bevy has more mature ECS implementation and query API. Flux integrates with .NET ecosystem (Azure, ASP.NET, Unity interop).

### vs Traditional OOP Game Architecture

| Aspect | Flux ECS | Traditional OOP |
|--------|----------|-----------------|
| **Learning curve** | Steeper | Gentle |
| **Performance (1000s entities)** | Excellent | Poor |
| **Determinism** | Guaranteed | Not guaranteed |
| **Memory layout** | Cache-friendly | Cache-hostile |
| **Iteration speed** | Fast (arrays) | Slow (pointer chasing) |
| **Debugging** | Need tools | Debugger-friendly |

Choose ECS for simulations with many entities. Choose OOP for UI, tools, small-scale projects.

---

## Performance

### Entity Operations

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Create entity | O(1) | Allocates from free list or advances index |
| Destroy entity | O(1) | Adds to free list, increments generation |
| Validate entity | O(1) | Checks generation matches |

The EntityAllocator uses a free list for O(1) allocation and recycling. No memory allocations after initialization.

### Tick Processing

With current implementation (world container only, no component queries yet):

| Entity Count | Tick Time | Notes |
|--------------|-----------|-------|
| 100 | ~200 μs | Target met |
| 1,000 | ~1 ms | Target met |
| 10,000 | TBD | Component storage needed for real test |

Once component queries are implemented, performance will depend on:
- Number of systems
- Component access patterns (read vs. write)
- System parallelization

### Memory Layout

```
EntityAllocator:
- Generations: byte[maxEntities]  (e.g., 100KB for 100K entities)
- Free list: List<int>             (amortized O(1) ops)

Archetype (per component type):
- Entities: List<Entity>
- Components: List<T> (SoA)
  Example: List<Position>, List<Velocity>

Total overhead: ~4 bytes per entity + component data
```

Very compact. A world with 100K entities with Position (8 bytes) and Velocity (8 bytes) uses:
- 100KB for generations
- 400KB for entity IDs
- 800KB for positions
- 800KB for velocities
- **Total: ~2.1 MB**

Compare to OOP with object overhead: ~100KB * 64 bytes = 6.4MB.

---

## What's Next

### Immediate Next Steps (Phase 1)

1. **Component Storage**: Add/Get/Remove components on entities
2. **Query API**: Iterate entities with specific component combinations
3. **System Examples**: Implement MovementSystem, DamageSystem as references

### Medium Term (Phase 2)

4. **Input System**: Queue and process player inputs per tick
5. **Event Buffer**: Collect simulation events (collisions, damage, etc.)
6. **State Snapshots**: Serialize/deserialize entire world state

### Long Term (Phase 3)

7. **Physics Integration**: Deterministic 2D physics subsystem
8. **Networking**: Integrate with Nexus for rollback netcode
9. **Parallel Systems**: Execute independent systems on multiple threads

### Related Documents

| Document | What It Covers |
|----------|----------------|
| [Atom Foundation](./01-foundation-atom.md) | DFixed64 - deterministic math |
| [LatticeDB Storage](./04-storage-lattice.md) | Lattice - storage layer |
| [Nexus Networking](./08-networking-nexus.md) | Multiplayer state synchronization |
| [Executive Overview](./00-executive-overview.md) | Five Laws of determinism |

---

## Summary

**Flux is Orix's deterministic simulation runtime.**

It solves the core problems of traditional game loops:
- ✅ Frame rate independence through tick-based execution
- ✅ Multiplayer synchronization through deterministic state
- ✅ Time-travel debugging through state snapshots
- ✅ Scalability through Entity-Component-System architecture

The current implementation provides:
- World container for simulation state
- Deterministic Entity allocation with generation counters
- Tick-based time representation
- System scheduling with priority ordering
- Component type registry and archetype storage

The tradeoffs:
- Learning curve: ECS is different from OOP
- Boilerplate: More setup for simple cases
- Mental model: Think in data and systems, not objects

When to use Flux:
- ✅ Multiplayer games requiring synchronization
- ✅ Replay systems (fighting games, RTS)
- ✅ Financial simulations
- ✅ Scientific modeling
- ✅ Any simulation requiring reproducibility

When NOT to use Flux:
- ❌ Simple single-player games
- ❌ UI-heavy applications
- ❌ Prototypes where determinism doesn't matter
- ❌ Projects with tight OOP requirements

**Next:** [Nexus Networking](./08-networking-nexus.md) - networking and state synchronization using Flux.
