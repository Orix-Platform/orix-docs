# Testing with Arbiter

**Meeting 11 - Orix Developer Guide**

Arbiter is Orix's deterministic testing framework. Every test runs with an explicit seed, ensuring perfect reproducibility. Same seed = same result, every time.

---

## Why Arbiter?

Traditional testing frameworks weren't designed for deterministic systems:

| Problem | Traditional Test | Arbiter Solution |
|---------|-----------------|------------------|
| **Non-deterministic failures** | `Random()`, `DateTime.Now` | Explicit seeds, `ScenarioContext` |
| **Unreproducible bugs** | "Works on my machine" | Seed in every test, hash verification |
| **Time-dependent code** | Flaky CI/CD | Tick-based simulation |
| **Unverified determinism** | Manual checking | Automatic replay verification |
| **Scattered test organization** | Folder hierarchies | Category-based filtering |

**The core problem:** You can't prove determinism with non-deterministic tests.

**Arbiter's answer:** Every scenario is a reproducible experiment. Same inputs = same outputs, always.

---

## How Arbiter Works

### 1. Seeded Scenarios

Every test declares its seed explicitly:

```csharp
[ArbiterScenario(Seed = 42, Category = "Math.DFixed64")]
public void DFixed64_Addition_IsCommutative()
{
    var a = DFixed64.FromInt(10);
    var b = DFixed64.FromInt(20);

    Assert.Equal(a + b, b + a);
}
```

**Key insight:** The seed isn't hidden in a setup method. It's visible in the attribute. Every developer knows exactly what inputs produced this result.

### 2. Automatic Replay

Arbiter can run determinism checks automatically:

```bash
orix arbiter determinism
```

This:
1. Runs each scenario
2. Records the final state hash
3. Runs it again with the same seed
4. Verifies identical hash

**If determinism breaks, the test fails immediately.**

### 3. Category Organization

Tests organize by feature, not folder structure:

```csharp
[ArbiterScenario(Seed = 42, Category = "Flux.World")]
[ArbiterScenario(Seed = 42, Category = "Flux.EntityAllocator")]
[ArbiterScenario(Seed = 42, Category = "Chronicle.TimeTravel")]
[ArbiterScenario(Seed = 42, Category = "Nexus.Sync")]
```

Run specific categories:

```bash
orix arbiter run --category Flux.World
orix arbiter run --category Chronicle
```

---

## What Arbiter Provides

### The `[ArbiterScenario]` Attribute

```csharp
public sealed class ArbiterScenarioAttribute : Attribute
{
    public ulong Seed { get; set; } = 1;
    public uint MaxTicks { get; set; } = 10000;
    public int TimeoutMs { get; set; } = 30000;
    public string? ExpectedHash { get; set; }
    public string? Category { get; set; }
    public string? Description { get; set; }
    public bool AllowAmbient { get; set; } = false;
    public bool Skip { get; set; } = false;
    public string? SkipReason { get; set; }
}
```

**Parameters:**

- `Seed` - Deterministic seed (default: 1)
- `MaxTicks` - Timeout for tick-based tests (default: 10,000)
- `TimeoutMs` - Wall-clock timeout for CI (default: 30,000ms)
- `ExpectedHash` - Verify exact final state (optional)
- `Category` - Group tests (e.g., "Flux.World")
- `AllowAmbient` - Allow non-deterministic operations (testing external APIs)
- `Skip` - Skip this test
- `SkipReason` - Explanation for skip

### The `Assert` Class

Arbiter assertions are Orix-aware:

```csharp
// Basic assertions
Assert.True(condition);
Assert.False(condition);
Assert.Equal(expected, actual);
Assert.NotEqual(expected, actual);
Assert.Null(value);
Assert.NotNull(value);

// Orix-aware assertions
Assert.ApproxEqual(expected, actual, tolerance);  // DFixed64
Assert.ApproxEqual(expectedVec, actualVec, tolerance);  // DVector2/3
Assert.InRange(value, min, max);  // DFixed64 range
Assert.Positive(value);  // DFixed64 > 0
Assert.Negative(value);  // DFixed64 < 0
Assert.Zero(value);  // DFixed64 == 0

// Tick assertions
Assert.AtOrAfter(tick, minimum);
Assert.Before(tick, maximum);

// Collection assertions
Assert.Empty(collection);
Assert.NotEmpty(collection);
Assert.Count(expected, collection);
Assert.Contains(item, collection);

// Exception assertions
var ex = Assert.Throws&lt;InvalidOperationException&gt;(() => DoSomething());
Assert.DoesNotThrow(() => DoSomething());

// Determinism verification
Assert.Deterministic(() => ComputeSomething(), runs: 10);
```

### The `ScenarioContext`

For tick-based simulations, use `ScenarioContext`:

```csharp
[ArbiterScenario(Seed = 42, MaxTicks = 1000, Category = "Simulation")]
public void PhysicsSimulation_TickBased(ScenarioContext ctx)
{
    var position = DVector3.Zero;
    var velocity = new DVector3(DFixed64.One, DFixed64.Zero, DFixed64.Zero);
    var gravity = new DVector3(DFixed64.Zero, -DFixed64.FromDouble(9.8), DFixed64.Zero);

    // Simulate 100 ticks
    for (int i = 0; i < 100; i++)
    {
        velocity += gravity * DFixed64.FromDouble(0.016);
        position += velocity * DFixed64.FromDouble(0.016);

        ctx.HashState(position);  // Track state for determinism
        ctx.Advance();  // Next tick
    }

    Assert.True(position.Y < DFixed64.Zero, "Object should have fallen");
}
```

**ScenarioContext API:**

```csharp
// Properties
ctx.Seed                  // ulong - The seed for this run
ctx.CurrentTick          // Tick - Current simulation tick
ctx.MaxTicks             // uint - Maximum allowed ticks
ctx.IsTimeout            // bool - Exceeded max ticks?
ctx.StateHash            // ulong - Accumulated state hash
ctx.Random               // OrixRandom - Deterministic RNG

// Methods
ctx.Advance()            // Advance one tick
ctx.Advance(count)       // Advance multiple ticks
ctx.HashState(value)     // Track state (ulong, DFixed64, DVector2/3)
ctx.RandomValue()        // DFixed64 in [0, 1)
ctx.RandomRange(min, max) // DFixed64 in [min, max)
ctx.RandomInt(min, max)  // int in [min, max)
ctx.RandomBool()         // Random bool
ctx.Reset()              // Reset to initial state
```

---

## Basic Scenario Patterns

### 1. Simple Assertion Test

```csharp
[ArbiterScenario(Seed = 42, Category = "Math.DFixed64")]
public void DFixed64_Addition_IsCommutative()
{
    var a = DFixed64.FromInt(10);
    var b = DFixed64.FromInt(20);

    Assert.Equal(a + b, b + a);
}
```

**When to use:** Pure functions, mathematical properties, invariants.

### 2. Determinism Verification Test

```csharp
[ArbiterScenario(Seed = 42, Category = "Flux.Determinism")]
public void Simulation_SameSeed_SameResult()
{
    // Run 1
    uint finalRandom1;
    long finalTick1;
    using (var world1 = new World(12345))
    {
        world1.Start();
        for (int i = 0; i < 10; i++)
        {
            world1.CreateEntity();
        }
        world1.ProcessTicks(100);
        finalRandom1 = world1.Random.NextUInt();
        finalTick1 = world1.CurrentTick.Value;
    }

    // Run 2 with same seed
    uint finalRandom2;
    long finalTick2;
    using (var world2 = new World(12345))
    {
        world2.Start();
        for (int i = 0; i < 10; i++)
        {
            world2.CreateEntity();
        }
        world2.ProcessTicks(100);
        finalRandom2 = world2.Random.NextUInt();
        finalTick2 = world2.CurrentTick.Value;
    }

    Assert.Equal(finalRandom1, finalRandom2, "Same seed = same random state");
    Assert.Equal(finalTick1, finalTick2, "Same operations = same tick count");
}
```

**When to use:** Verify entire subsystems are deterministic.

### 3. Tick-Based Simulation Test

```csharp
[ArbiterScenario(Seed = 42, MaxTicks = 1000, Category = "Simulation")]
public void PhysicsSimulation_TickBased(ScenarioContext ctx)
{
    var position = DVector3.Zero;
    var velocity = new DVector3(DFixed64.One, DFixed64.Zero, DFixed64.Zero);
    var gravity = new DVector3(DFixed64.Zero, -DFixed64.FromDouble(9.8), DFixed64.Zero);

    for (int i = 0; i < 100; i++)
    {
        velocity += gravity * DFixed64.FromDouble(0.016);
        position += velocity * DFixed64.FromDouble(0.016);
        ctx.HashState(position);
        ctx.Advance();
    }

    Assert.True(position.Y < DFixed64.Zero, "Object should have fallen");
}
```

**When to use:** Time-stepped simulations, physics, gameplay logic.

### 4. Property-Based Test

```csharp
[ArbiterScenario(Seed = 42, Category = "Math.Random")]
public void OrixRandom_Range_WithinBounds(ScenarioContext ctx)
{
    var min = DFixed64.FromInt(-10);
    var max = DFixed64.FromInt(10);

    for (int i = 0; i < 1000; i++)
    {
        var value = ctx.RandomRange(min, max);
        Assert.InRange(value, min, max);
    }
}
```

**When to use:** Verify properties hold across many random inputs.

### 5. Hash Verification Test

```csharp
[ArbiterScenario(Seed = 42, ExpectedHash = "A1B2C3D4E5F6", Category = "Regression")]
public void KnownBehavior_MatchesSnapshot(ScenarioContext ctx)
{
    // Run some complex simulation
    var result = RunComplexSimulation(ctx);

    // Hash final state
    ctx.HashState(result);

    // Arbiter automatically verifies ctx.StateHash == ExpectedHash
}
```

**When to use:** Regression tests - verify exact behavior doesn't change.

---

## Real-World Examples from Orix

### Example 1: Flux World Lifecycle

```csharp
// From tests/Flux.Tests/TickScenarios.cs

[ArbiterScenario(Seed = 42, Category = "Flux.World")]
public void World_StartStop_TransitionsCorrectly()
{
    using var world = new World(42);

    Assert.True(world.State == WorldState.Stopped, "Initial: Stopped");

    world.Start();
    Assert.True(world.State == WorldState.Running, "After Start: Running");

    world.Pause();
    Assert.True(world.State == WorldState.Paused, "After Pause: Paused");

    world.Resume();
    Assert.True(world.State == WorldState.Running, "After Resume: Running");

    world.Stop();
    Assert.True(world.State == WorldState.Stopped, "After Stop: Stopped");
}
```

**Pattern:** State machine testing - verify all transitions work correctly.

### Example 2: Entity Allocation Determinism

```csharp
// From tests/Flux.Tests/TickScenarios.cs

[ArbiterScenario(Seed = 42, Category = "Flux.EntityAllocator")]
public void EntityAllocator_Allocate_ProducesUniqueEntities()
{
    var allocator = new EntityAllocator(1000);
    var entities = new HashSet&lt;uint&gt;();

    for (int i = 0; i < 100; i++)
    {
        var entity = allocator.Allocate();
        Assert.True(!entities.Contains(entity.PackedValue),
            $"Entity {i} should be unique");
        entities.Add(entity.PackedValue);
    }

    Assert.Equal(100, allocator.AllocatedCount);
}
```

**Pattern:** Uniqueness verification across many allocations.

### Example 3: Math Properties

```csharp
// From tests/Atom.Tests/MathScenarios.cs

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

**Pattern:** Mathematical identity verification with tolerance for fixed-point approximation.

### Example 4: Deterministic RNG

```csharp
// From tests/Atom.Tests/MathScenarios.cs

[ArbiterScenario(Seed = 42, Category = "Math.Random")]
public void OrixRandom_Deterministic_SameSeedSameSequence(ScenarioContext ctx)
{
    // Generate sequence
    var values = new DFixed64[100];
    for (int i = 0; i < 100; i++)
    {
        values[i] = ctx.RandomValue();
        ctx.HashState(values[i]);
    }

    // Reset and verify same sequence
    ctx.Reset();
    for (int i = 0; i < 100; i++)
    {
        Assert.Equal(values[i], ctx.RandomValue(), $"Mismatch at index {i}");
    }
}
```

**Pattern:** Record-and-replay to verify determinism.

### Example 5: Chronicle Time Travel

```csharp
// From tests/Echo.Tests/ReplayScenarios.cs

[ArbiterScenario(Seed = 42, Category = "Echo.TimeTravel")]
public void TimeTravel_JumpToTick_RestoresState()
{
    var recording = CreateTestRecording();
    var player = new ReplayPlayer(recording);

    // Play forward to tick 50
    player.PlayTo(50);
    var state50 = player.GetCurrentStateHash();

    // Continue to tick 100
    player.PlayTo(100);
    var state100 = player.GetCurrentStateHash();

    // Jump back to tick 50
    player.JumpTo(50);
    var state50Again = player.GetCurrentStateHash();

    Assert.Equal(state50, state50Again, "Jumping back should restore exact state");
    Assert.NotEqual(state50, state100, "Different ticks should have different states");
}
```

**Pattern:** Time-travel verification - ensure exact state restoration.

---

## Test Categories in Orix

Real categories from the codebase:

### Atom (Foundation)

- `Math.DFixed64` - Fixed-point arithmetic
- `Math.Vector` - Vector operations
- `Math.Random` - Deterministic RNG
- `Determinism` - Core determinism verification
- `Simulation` - Tick-based simulation tests

### Flux (ECS Runtime)

- `Flux.Tick` - Tick advancement
- `Flux.Lifecycle` - Entity create/destroy
- `Flux.World` - World state management
- `Flux.EntityAllocator` - Entity ID allocation
- `Flux.Determinism` - Full simulation determinism

### Lattice (Storage)

- `Lattice.CRUD` - Create/Read/Update/Delete
- `Lattice.Query` - Query execution
- `Chronicle.StateHasher` - State hashing
- `Chronicle.MerkleTree` - Merkle proof verification
- `Chronicle.Snapshot` - Snapshot creation/restoration
- `Chronicle.TimeTravel` - Time-travel API
- `CRDT.Register` - LWW-Register tests
- `CRDT.Counter` - G-Counter, PN-Counter
- `CRDT.Set` - OR-Set tests
- `CRDT.Map` - OR-Map tests

### Nexus (Networking)

- `Nexus.Sync` - State synchronization
- `Nexus.Delta` - Delta compression
- `Nexus.Authority` - Authority resolution
- `Nexus.Determinism` - Network determinism

### Echo (Replay)

- `Echo.Recording` - Recording creation/state
- `Echo.Playback` - Replay playback
- `Echo.TimeTravel` - Jump to tick
- `Echo.Determinism` - Replay determinism

### Lumen (Observability)

- `Lumen.Logging` - Structured logging
- `Lumen.Metrics` - Metric collection
- `Lumen.Tracing` - Distributed tracing
- `Lumen.Determinism` - Logging determinism

### Crypto

- `Crypto.Envelope` - Envelope encryption
- `Crypto.TimeLock` - Time-locked encryption
- `Crypto.StructuredEncryption` - Searchable encryption
- `PQC.MlKem` - Post-quantum KEM
- `PQC.MlDsa` - Post-quantum signatures

---

## Running Tests

### Run All Tests

```bash
orix arbiter run
```

### Run by Category

```bash
orix arbiter run --category Flux
orix arbiter run --category Chronicle.TimeTravel
orix arbiter run --category Math
```

### Run Specific Test File

```bash
cd tests/Flux.Tests
dotnet test
```

### Verify Determinism

```bash
orix arbiter determinism
```

This runs each test twice with the same seed and verifies identical results.

### Verbose Output

```bash
orix arbiter run --verbose
```

### With Custom Seed Override

```bash
orix arbiter run --seed 99999
```

Runs all tests with seed 99999 instead of their declared seeds.

---

## Advantages

### 1. Perfect Reproducibility

```csharp
[ArbiterScenario(Seed = 42, Category = "Bugs.DesyncIssue123")]
public void Reproduce_DesyncBug_FromIssue123()
{
    // This test will ALWAYS reproduce the exact bug
    // because the seed is fixed
}
```

Ship this test with a bug report. Anyone can reproduce it.

### 2. Determinism Verification Built-In

```bash
orix arbiter determinism
```

Automatic verification that all tests are actually deterministic.

### 3. Clear Failure Reproduction

```
FAIL: Simulation_SameSeed_SameResult
  Seed: 42
  Tick: 150
  Hash: Expected A1B2C3D4, got A1B2C3D5

To reproduce:
  orix arbiter run --seed 42 --test Simulation_SameSeed_SameResult
```

### 4. Category-Based Organization

```bash
# Test just Chronicle
orix arbiter run --category Chronicle

# Test all CRDT types
orix arbiter run --category CRDT

# Test determinism across all products
orix arbiter run --category Determinism
```

### 5. State Hash Tracking

```csharp
ctx.HashState(position);
ctx.HashState(velocity);
ctx.HashState(entityCount);

// At end: ctx.StateHash contains all accumulated state
```

Arbiter tracks state hashes automatically. Use `ExpectedHash` for regression tests.

---

## Disadvantages

### 1. Requires Seed Management

Every test needs an explicit seed. Can't just write `new Random()`.

**Mitigation:** ScenarioContext provides seeded RNG automatically.

### 2. Not Suitable for Non-Deterministic Tests

Testing external APIs, network calls, file I/O requires `AllowAmbient`:

```csharp
[ArbiterScenario(Seed = 42, AllowAmbient = true, Category = "Integration")]
public void ExternalAPI_ResponseParsing()
{
    // This test is allowed to be non-deterministic
}
```

### 3. Learning Curve

Property-based testing and seed management are unfamiliar to many developers.

**Mitigation:** Clear examples, documentation, and patterns.

### 4. Slower Than Unit Tests

Determinism verification requires running tests multiple times.

**Mitigation:** Run determinism checks in CI, not locally every time.

---

## Arbiter vs. Other Frameworks

### vs. xUnit/NUnit

| Feature | xUnit/NUnit | Arbiter |
|---------|-------------|---------|
| **Seed management** | Manual | Built-in attribute |
| **Determinism verification** | Manual | Automatic |
| **Tick-based simulation** | Manual | ScenarioContext |
| **State hashing** | Manual | Built-in |
| **Category filtering** | Traits/Categories | Category attribute |
| **Fixed-point aware** | No | Yes (DFixed64, DVector) |
| **Time abstraction** | No | Tick-based |

**Use xUnit/NUnit when:** Testing infrastructure, I/O, external integrations.

**Use Arbiter when:** Testing deterministic simulation, game logic, math.

### vs. QuickCheck/Hypothesis

| Feature | QuickCheck | Arbiter |
|---------|------------|---------|
| **Property testing** | Yes | Yes (via ScenarioContext) |
| **Shrinking** | Yes | No (explicit seeds) |
| **Type generators** | Automatic | Manual (via OrixRandom) |
| **Determinism focus** | No | Yes |
| **Orix primitives** | No | Yes |

**Use QuickCheck when:** Need automatic input generation and shrinking.

**Use Arbiter when:** Need deterministic, reproducible tests with explicit seeds.

---

## Best Practices

### 1. Always Use Explicit Seeds

```csharp
// Good
[ArbiterScenario(Seed = 42, Category = "Math")]
public void Test_Something() { }

// Bad - uses default seed (1)
[ArbiterScenario(Category = "Math")]
public void Test_Something() { }
```

**Why:** Explicit seeds make tests reproducible and debuggable.

### 2. Use ScenarioContext for Simulations

```csharp
// Good
[ArbiterScenario(Seed = 42, MaxTicks = 1000, Category = "Simulation")]
public void Simulate_Physics(ScenarioContext ctx)
{
    for (int i = 0; i < 100; i++)
    {
        // ... simulation logic ...
        ctx.HashState(state);
        ctx.Advance();
    }
}

// Bad - manual tick tracking
public void Simulate_Physics()
{
    var tick = 0;
    for (int i = 0; i < 100; i++)
    {
        tick++;
        // ... simulation logic ...
    }
}
```

**Why:** ScenarioContext provides state tracking, timeout detection, and reset.

### 3. Hash State at Key Points

```csharp
ctx.HashState(position);
ctx.HashState(velocity);
ctx.HashState(entityCount);
```

**Why:** State hashing enables determinism verification and regression detection.

### 4. Use Categories Consistently

```csharp
// Good - hierarchical categories
"Flux.World"
"Flux.EntityAllocator"
"Chronicle.TimeTravel"
"CRDT.Register"

// Bad - flat categories
"Test1"
"Test2"
```

**Why:** Hierarchical categories enable filtering by subsystem.

### 5. Write Determinism Tests for New Features

```csharp
[ArbiterScenario(Seed = 42, Category = "MyFeature.Determinism")]
public void MyFeature_SameSeed_SameResult()
{
    var result1 = RunMyFeature(42);
    var result2 = RunMyFeature(42);
    Assert.Equal(result1, result2);
}
```

**Why:** Catch non-determinism early, before it ships.

---

## Common Patterns

### Pattern: Two-Run Determinism Check

```csharp
[ArbiterScenario(Seed = 42, Category = "Determinism")]
public void Feature_IsDeterministic()
{
    var result1 = RunComplexOperation(seed: 12345);
    var result2 = RunComplexOperation(seed: 12345);
    Assert.Equal(result1, result2);
}
```

### Pattern: Record-Replay

```csharp
[ArbiterScenario(Seed = 42, Category = "Replay")]
public void Record_AndReplay_ProducesSameResult(ScenarioContext ctx)
{
    var values = new List&lt;DFixed64&gt;();

    // Record
    for (int i = 0; i < 100; i++)
    {
        values.Add(ctx.RandomValue());
    }

    // Replay
    ctx.Reset();
    for (int i = 0; i < 100; i++)
    {
        Assert.Equal(values[i], ctx.RandomValue());
    }
}
```

### Pattern: Property Verification

```csharp
[ArbiterScenario(Seed = 42, Category = "Properties")]
public void Property_HoldsForRandomInputs(ScenarioContext ctx)
{
    for (int i = 0; i < 1000; i++)
    {
        var input = ctx.RandomRange(min, max);
        var output = ProcessInput(input);

        // Verify property
        Assert.True(PropertyHolds(input, output));
    }
}
```

### Pattern: State Machine Testing

```csharp
[ArbiterScenario(Seed = 42, Category = "StateMachine")]
public void StateMachine_AllTransitions_AreValid()
{
    var sm = new StateMachine();
    Assert.Equal(State.Initial, sm.Current);

    sm.Transition(Event.Start);
    Assert.Equal(State.Running, sm.Current);

    sm.Transition(Event.Pause);
    Assert.Equal(State.Paused, sm.Current);

    // ... test all transitions
}
```

### Pattern: Regression with Hash

```csharp
[ArbiterScenario(
    Seed = 42,
    ExpectedHash = "A1B2C3D4E5F6",
    Category = "Regression")]
public void KnownBehavior_MatchesSnapshot(ScenarioContext ctx)
{
    var result = RunComplexSimulation(ctx);
    ctx.HashState(result);

    // Arbiter automatically verifies final hash
}
```

---

## Summary

Arbiter is Orix's answer to deterministic testing:

- **Explicit seeds** - Every test declares its seed
- **Automatic replay** - Determinism verification built-in
- **Tick-aware** - ScenarioContext for simulations
- **State tracking** - Hash accumulation for regression detection
- **Category-based** - Organize tests by feature, not folder

**The golden rule:** Same seed = same result, always.

Write tests that prove it.

---

## Related Documents

- [Atom Foundation](./01-foundation-atom.md) - DFixed64 and deterministic primitives tested by Arbiter
- [Flux Simulation](./07-simulation-flux.md) - ECS runtime with determinism tests
- [Echo Replay](./09-replay-echo.md) - Replay verification tests
- [Chronicle Time-Travel](./05-chronicle.md) - Time-travel tests

---

**Next:** [Technical Deep-Dive](./12-technical-deep-dive.md) - Architecture details for engineers
