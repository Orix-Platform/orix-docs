# Chronicle: Time-Travel Database

**Part of:** Lattice (Storage & Query Layer)
**Status:** Implemented
**Version:** 1.0.0

Chronicle is Orix's time-travel database, enabling queries of historical state at any tick, branching for parallel timelines, and complete audit trails. It makes "What was the game state when the bug occurred?" as natural as querying current state.

---

## Why Time-Travel?

Traditional databases give you now. Chronicle gives you any moment in history.

### The Problems

**Debugging distributed systems requires seeing past state**
```
User: "My player lost 500 gold at 3:42pm yesterday."
Dev: "Let me check the database..."
Database: "Here's the current gold: 1,200"
Dev: "That doesn't help. What was it yesterday?"
Database: "I don't remember yesterday."
```

With Chronicle:
```csharp
var state = chronicle.TravelTo(tick: 186420); // 3:42pm yesterday
var player = state.GetEntity<Player>(playerId);
// player.Gold = 500 (we can see exactly what happened)
```

**Audit requirements need complete history**

Regulatory compliance often requires: "Show me all changes to this account."

Traditional approach: Custom audit tables, triggers, separate logging infrastructure.

Chronicle approach: Built-in. Every change is automatically versioned.

```csharp
var history = chronicle.Query<Account>()
    .ForEntity(accountId)
    .FullHistory()
    .ExecuteHistory();

foreach (var version in history.Versions)
{
    Console.WriteLine($"Tick {version.Tick}: Balance = {version.Value.Balance}");
}
```

**"What-if" analysis needs state branching**

Testing a new feature without affecting production:
- Create a branch from production state
- Run experiments on the branch
- Discard if it fails, merge if successful

Traditional approach: Copy entire database, manual reconciliation.

Chronicle approach:
```csharp
// Branch from current state
var branch = chronicle.Branch(currentSnapshot, new BranchOptions
{
    Name = "experiment-new-matchmaking"
});

// Run tests on branch
RunMatchmakingExperiment();

// If successful, merge back
if (success)
    chronicle.Merge(branch.Timeline.Id, into: "main");
```

**Traditional logging loses context**

Logs capture events: "Player attacked", "Gold changed".

They don't capture state: "What was the player's inventory?" "Which enemies were nearby?"

Chronicle captures complete state at every tick, making historical debugging trivial.

---

## How Chronicle Solves It

### Technical Approach

**1. Append-Only Storage for Immutability**

```
Every write is a new version. Nothing is ever deleted or overwritten.

Tick 100: Player gold = 1000
Tick 101: Player gold = 1200  (previous version still exists)
Tick 102: Player gold = 900   (both previous versions still exist)
```

This enables:
- Time-travel queries (ask for state at tick 100)
- Complete audit trail (see all changes)
- Deterministic replay (same inputs = same history)

**2. Temporal Indexing by Tick**

Every entity version is indexed by tick, enabling O(log n) lookups:

```csharp
// Find player state at tick 5000
var version = versionStore.GetAt(playerId, tick: 5000);
```

Binary search through versions:
```
Versions: [tick: 100] [tick: 500] [tick: 5000] [tick: 6000]
Query: tick 5500
Result: version at tick 5000 (latest before 5500)
```

**3. Branching for Parallel Timelines**

```
Main Timeline:  [tick 0] → [tick 100] → [tick 200] → [tick 300]
                                 ↓
                    Branch "experiment":
                    [tick 100] → [tick 150] → [tick 200]
```

Each timeline is independent:
- Changes on branch don't affect main
- Can compare states between timelines
- Can merge successful experiments back

**4. Efficient Snapshot Management**

Full snapshots at intervals, deltas in between:

```
[Full Snapshot at tick 0]
  + delta at tick 1
  + delta at tick 2
  + delta at tick 3
  ...
[Full Snapshot at tick 100]
  + delta at tick 101
  + delta at tick 102
```

Reconstruct state at tick 52:
1. Load full snapshot at tick 0
2. Apply deltas 1-52
3. Return reconstructed state

Benefits:
- Storage efficiency (deltas are smaller than full snapshots)
- Fast reconstruction (don't replay entire history)
- Configurable trade-off (more snapshots = faster queries, more storage)

---

## What Chronicle Provides

### 1. Time-Travel Queries

Access any historical state:

```csharp
// Query state at specific tick
var result = chronicle.TravelTo(tick: 5000);

if (result.Success)
{
    var playerData = result.StateData;
    // playerData contains the complete game state at tick 5000
}
```

With temporal query builder:

```csharp
var state = chronicle.Query<Player>(deserializer)
    .ForEntity(playerId)
    .AsOf(tick: 5000)
    .Execute();

if (state.Found)
{
    Console.WriteLine($"Player gold at tick 5000: {state.Value.Gold}");
}
```

### 2. Branches

Create parallel timelines for experimentation:

```csharp
// Create a snapshot to branch from
var snapshot = chronicle.CreateSnapshot(
    tick: currentTick,
    stateData: gameState,
    stateHash: ComputeHash(gameState)
);

// Branch from this snapshot
var branchResult = chronicle.Branch(snapshot.Id, new BranchOptions
{
    Name = "test-new-feature",
    Description = "Testing double XP weekend",
    SwitchToNewTimeline = true
});

if (branchResult.Success)
{
    // Now operating on branch timeline
    // Changes here don't affect main timeline
}
```

Switch between timelines:

```csharp
// Work on main timeline
chronicle.SwitchTimeline(timelineId: 1); // main timeline

// Work on experiment
chronicle.SwitchTimeline(branchResult.Timeline.Id);
```

### 3. Snapshots

Efficient state checkpoints:

```csharp
var snapshot = chronicle.CreateSnapshot(
    tick: currentTick,
    stateData: SerializeGameState(),
    stateHash: ComputeStateHash(),
    options: new SnapshotOptions
    {
        Label = "before-boss-fight",
        Compress = true,
        IncludeTables = new[] { "players", "inventory" }
    }
);

Console.WriteLine($"Created snapshot {snapshot.Id} at tick {snapshot.Tick}");
```

Retrieve snapshots:

```csharp
var snapshot = chronicle.GetSnapshot(snapshotId);
var decompressedData = chronicle.GetSnapshotData(snapshotId);

// Find nearest snapshot before a tick
var nearest = chronicle.FindLatestSnapshotBefore(timelineId: 1, tick: 5000);
```

### 4. History Queries

Query changes over time:

```csharp
// Get all versions of an entity between two ticks
var history = chronicle.Query<Player>(deserializer)
    .ForEntity(playerId)
    .Between(startTick: 1000, endTick: 2000)
    .ExecuteHistory();

foreach (var version in history.Versions)
{
    Console.WriteLine($"Tick {version.Tick}: {version.Value}");
}
```

Full entity history:

```csharp
var fullHistory = chronicle.Query<Account>(deserializer)
    .ForEntity(accountId)
    .FullHistory()
    .ExecuteHistory();

Console.WriteLine($"Found {fullHistory.Versions.Count} versions");
Console.WriteLine($"First change: tick {fullHistory.StartTick}");
Console.WriteLine($"Last change: tick {fullHistory.EndTick}");
```

### 5. Comparison Operations

Compare states between timelines:

```csharp
var comparison = chronicle.Compare(
    timelineA: 1,        // main timeline
    timelineB: branch.Timeline.Id,
    tick: 5000
);

if (comparison.AreIdentical)
{
    Console.WriteLine("States are identical at tick 5000");
}
else
{
    Console.WriteLine($"Hash A: {comparison.StateHashA}");
    Console.WriteLine($"Hash B: {comparison.StateHashB}");

    foreach (var diff in comparison.Differences)
    {
        Console.WriteLine($"{diff.Type}: {diff.TableName}[{diff.EntityId}]");
    }
}
```

### 6. Verification

Verify snapshot integrity:

```csharp
var isValid = chronicle.VerifySnapshot(
    snapshotId,
    hashFunction: data => ComputeXxHash(data)
);

if (!isValid)
{
    Console.WriteLine("WARNING: Snapshot integrity check failed!");
}
```

---

## Core Operations

### Creating a Chronicle

```csharp
// In-memory storage
var chronicle = new ChronicleEngine();

// Persistent storage
var storage = new LatticeStorageEngine();
storage.Open("game.chronicle");
var chronicle = new ChronicleEngine(storage);
```

### Recording Changes

```csharp
// Ensure main timeline exists
var mainTimeline = chronicle.EnsureMainTimeline();

// Record state at each tick
for (ulong tick = 0; tick < 10000; tick++)
{
    // Simulate game tick
    UpdateGameState(tick);

    // Create snapshot every 100 ticks
    if (tick % 100 == 0)
    {
        var stateData = SerializeGameState();
        var stateHash = ComputeHash(stateData);

        chronicle.CreateSnapshot(tick, stateData, stateHash);
    }
}
```

### Time-Travel Debugging

```csharp
// Bug occurred at tick 7,342
var result = chronicle.TravelTo(tick: 7342);

if (result.Success)
{
    Console.WriteLine($"Restored state from snapshot at tick {result.Snapshot.Tick}");
    Console.WriteLine($"Need to replay {result.TicksReplayed} ticks");

    // Deserialize and inspect state
    var gameState = DeserializeGameState(result.StateData);
    InspectBugState(gameState);
}
```

### Branch Testing

```csharp
// Save current state
var snapshot = chronicle.CreateSnapshot(
    tick: currentTick,
    stateData: currentState,
    stateHash: currentHash
);

// Create experiment branch
var branch = chronicle.Branch(snapshot.Id, new BranchOptions
{
    Name = "test-balance-changes",
    Description = "Testing weapon damage rebalance",
    SwitchToNewTimeline = true
});

// Run experiment
for (int i = 0; i < 1000; i++)
{
    SimulateGameTick();
}

// Analyze results
var experimentResult = AnalyzeBalance();

if (experimentResult.IsAcceptable)
{
    // Success - could merge or extract changes
    Console.WriteLine("Experiment successful");
}
else
{
    // Failure - just switch back to main
    chronicle.SwitchTimeline(1); // back to main
    Console.WriteLine("Experiment failed, discarded");
}
```

---

## Use Cases

### Debugging: "What was the game state when the bug occurred?"

```csharp
// Player reports: "I lost my legendary sword at timestamp 2026-01-06 15:42:00"
var bugTick = TimestampToTick("2026-01-06 15:42:00");

// Time-travel to that moment
var state = chronicle.Query<Player>(deserializer)
    .ForEntity(playerId)
    .AsOf(tick: bugTick)
    .Execute();

if (state.Found)
{
    var player = state.Value;
    Console.WriteLine($"Player inventory at bug tick: {player.Inventory}");

    // Check history around that time
    var history = chronicle.Query<Player>(deserializer)
        .ForEntity(playerId)
        .Between(bugTick - 100, bugTick + 100)
        .ExecuteHistory();

    // Find when sword disappeared
    foreach (var version in history.Versions)
    {
        if (!version.Value.Inventory.Contains("legendary_sword"))
        {
            Console.WriteLine($"Sword disappeared at tick {version.Tick}");
            break;
        }
    }
}
```

### Auditing: "Show all changes to this account"

```csharp
// Compliance requirement: Show complete account history
var accountHistory = chronicle.Query<BankAccount>(deserializer)
    .ForEntity(accountId)
    .FullHistory()
    .ExecuteHistory();

Console.WriteLine($"Account {accountId} History:");
Console.WriteLine($"Created at tick {accountHistory.StartTick}");
Console.WriteLine($"Last modified at tick {accountHistory.EndTick}");
Console.WriteLine($"Total versions: {accountHistory.Versions.Count}");

foreach (var version in accountHistory.Versions)
{
    Console.WriteLine($"Tick {version.Tick}: Balance = {version.Value.Balance}");
}
```

### Testing: "Branch, run experiment, discard if failed"

```csharp
// Save production state
var productionSnapshot = chronicle.CreateSnapshot(
    tick: currentTick,
    stateData: productionState,
    stateHash: productionHash,
    options: new SnapshotOptions { Label = "before-experiment" }
);

// Branch for testing
var testBranch = chronicle.Branch(productionSnapshot.Id, new BranchOptions
{
    Name = "test-new-matchmaking",
    Description = "Testing skill-based matchmaking algorithm"
});

// Run experiment
var metrics = new MatchmakingMetrics();
for (int i = 0; i < 10000; i++)
{
    RunMatchmakingTick();
    metrics.RecordResults();
}

// Evaluate
if (metrics.PlayerSatisfaction > 0.8 && metrics.MatchQuality > 0.75)
{
    Console.WriteLine("Experiment successful - could deploy");
    // In practice: extract changes, apply to main timeline
}
else
{
    Console.WriteLine("Experiment failed - discarding branch");
    chronicle.SwitchTimeline(1); // back to main
}
```

### Replay: "Restart from tick 10000"

```csharp
// Game crashed at tick 15,000
// Restart from last stable snapshot
var snapshot = chronicle.FindLatestSnapshotBefore(timelineId: 1, tick: 10000);

if (snapshot != null)
{
    var stateData = chronicle.GetSnapshotData(snapshot.Id);
    var gameState = DeserializeGameState(stateData);

    Console.WriteLine($"Restored from snapshot at tick {snapshot.Tick}");
    Console.WriteLine($"Replaying {15000 - snapshot.Tick} ticks...");

    // Replay from snapshot to crash point
    for (ulong tick = snapshot.Tick; tick < 15000; tick++)
    {
        SimulateGameTick(gameState, tick);
    }
}
```

---

## Advantages

### Complete History Available

Every state change is preserved:
- Debug issues from hours/days ago
- Analyze player behavior over time
- Reproduce exact conditions of bugs

### Non-Destructive Experimentation

Branch without risk:
- Test balance changes without affecting players
- A/B test features on historical data
- Validate fixes before deploying

### Powerful Debugging Tool

Time-travel makes impossible debugging possible:
- "What was the state when X happened?" → Just query it
- "When did this value change?" → Query history
- "What caused this cascade?" → Replay with inspection

### Audit Compliance Built-In

Regulatory requirements met automatically:
- Complete change history
- Immutable audit trail
- Cryptographic integrity verification

### Deterministic Replay

Same inputs = same history:
- Reproduce bugs exactly
- Validate fixes with historical data
- Test changes against real player behavior

---

## Disadvantages

### Storage Grows Over Time

Every version is stored:
- 10,000 ticks × 100 entities × 1KB = 1GB
- Need retention policies for long-running systems
- Storage costs proportional to state change rate

Mitigation:
```csharp
var config = new ChronicleConfig
{
    RetentionTicks = 86400,  // Keep 24 hours
    MaxVersionsPerEntity = 1000,
    VacuumAfterTicks = 86400
};
```

### Query Complexity for Deep History

Reconstructing state at tick 50,000 when last snapshot is tick 0:
- Must apply 50,000 deltas
- O(n) reconstruction time

Mitigation:
- More frequent snapshots (trade storage for query speed)
- Snapshot every N ticks (configurable threshold)
- LRU cache for recent snapshots

### Branch Management Overhead

Each branch:
- Separate timeline metadata
- Snapshot references
- Potential storage duplication

Mitigation:
- Prune inactive branches
- Merge successful experiments promptly
- Archive instead of keeping live

---

## Comparison to Alternatives

### vs Event Sourcing

**Event Sourcing:** Store events, reconstruct state by replaying

```csharp
// Event Sourcing
events = [PlayerCreated, GoldAdded(500), GoldSpent(100)]
state = events.Aggregate(initialState, applyEvent)
```

**Chronicle:** Store state versions, query-friendly

```csharp
// Chronicle
state = chronicle.GetAt(playerId, tick: 5000)
// Direct state access, no replay needed for queries
```

Chronicle advantages:
- Faster queries (O(log n) vs O(n))
- State-oriented (matches game model)
- Built-in branching

Event Sourcing advantages:
- Captures intent ("why" not just "what")
- Smaller storage (events < state)
- Better for business logic replay

### vs Datomic

**Similar philosophy:** Immutable facts, temporal queries, time-travel

```clojure
; Datomic
(d/q '[:find ?balance
       :in $ ?account
       :where [?account :account/balance ?balance]]
     (d/as-of db #inst "2026-01-06")
     account-id)
```

Chronicle differences:
- Orix-integrated (uses Tick, Entity, DFixed64)
- Gaming-optimized (fast snapshot/restore)
- ECS-friendly (entity-component model)
- Deterministic (required for game simulation)

### vs Temporal Tables (SQL)

**SQL Temporal Tables:** SYSTEM_VERSIONING with history table

```sql
SELECT balance FROM accounts
FOR SYSTEM_TIME AS OF '2026-01-06 15:42:00'
WHERE account_id = 123;
```

Chronicle differences:
- Richer branching (parallel timelines)
- Tick-based (not wall-clock time)
- Deterministic (no time zone issues)
- Snapshot-based (not transaction logs)

### vs Git (for data)

**Git:** Version control for code, sometimes used for data

Chronicle differences:
- Optimized for frequent changes (per-tick)
- Query-oriented (not diff-oriented)
- Binary-efficient (not text-oriented)
- Gaming semantics (tick, entity, timeline)

---

## Performance

### Snapshot Restoration

**O(1) with periodic snapshots**

```csharp
// Query at tick 5,342
// Nearest snapshot: tick 5,300
// Apply deltas: 5,300 → 5,342 (42 deltas)

var result = chronicle.TravelTo(tick: 5342);
// Time: 42 delta applications ≈ 1-2ms
```

With snapshots every 100 ticks:
- Worst case: 99 delta applications
- Typical case: 50 delta applications
- Best case: 0 deltas (exact snapshot match)

### Point-in-Time Query

**O(log n) for tick lookup**

```csharp
// 10,000 versions stored
// Binary search: log₂(10,000) ≈ 13 comparisons

var version = versionStore.GetAt(entityId, tick: 7,500);
```

Benchmark (10,000 versions):
- Binary search: ~13 comparisons
- Reconstruction: 0-100 delta applications
- Total: 1-5ms typical

### Storage

**Compressed deltas between snapshots**

Example (Player entity, 256 bytes):
```
Full Snapshot (tick 0):     256 bytes
Delta (tick 1):             12 bytes (only gold changed)
Delta (tick 2):             8 bytes (only position changed)
...
Full Snapshot (tick 100):   256 bytes
```

Compression ratio (typical):
- With deltas: 30-50% of full storage
- With LZ4 compression: 60-80% compression ratio
- Combined: 15-40% of naive full-snapshot storage

Storage calculation:
```
Entities: 1,000
State size: 256 bytes/entity
Ticks: 10,000
Snapshot interval: 100 ticks

Naive: 1,000 × 256 × 10,000 = 2.56 GB
Chronicle: 1,000 × 256 × (10,000/100 + 10,000×0.3) = 844 MB
Savings: 67%
```

### Stats Monitoring

```csharp
var stats = chronicle.GetStats();

Console.WriteLine($"Timelines: {stats.TimelineCount}");
Console.WriteLine($"Snapshots: {stats.SnapshotCount}");
Console.WriteLine($"Total data: {stats.TotalDataBytes / 1024 / 1024} MB");
Console.WriteLine($"Uncompressed: {stats.TotalUncompressedBytes / 1024 / 1024} MB");
Console.WriteLine($"Compression: {stats.CompressionRatio:P2}");
```

---

## Implementation Details

### Architecture

```
ChronicleEngine
├── Timeline Management (branching, switching)
├── Snapshot Management (create, retrieve, cache)
├── Version Store (entity versions, delta compression)
├── Temporal Queries (AsOf, Between, History)
└── Storage Backend (persistence, indexes)
```

### Key Components

| Component | Purpose |
|-----------|---------|
| **ChronicleEngine** | Core engine for time-travel and branching, timeline/snapshot management |
| **Timeline** | Timeline/branch representation, parent-child relationships |
| **Snapshot** | Immutable state capture with compression support |
| **VersionStore** | Per-entity version history with delta compression |
| **TemporalQuery** | Fluent query builder for AsOf, Between, History modes |

### Schema Example

Chronicle types are defined in Axion schema:
```axion
enum TemporalMode {
    Current = 0,
    AsOf = 1,
    Between = 2,
    History = 3
}

entity EntityVersion {
    @key id: uuid,
    @indexed entity_id: uuid,
    version: int64,
    created_tick: int64,
    superseded_tick: int64?,
    operation: VersionOperation,
    transaction_id: uuid
}

config ChronicleConfig {
    enabled: bool = true,
    retention_ticks: int64 = 0,
    max_versions_per_entity: int32 = 0,
    vacuum_after_ticks: int64 = 86400,
    delta_compression: bool = true
}
```

---

## Best Practices

### Snapshot Frequency

Balance query speed vs storage:

```csharp
// High-frequency (every 10 ticks)
// + Fast queries
// - More storage
chronicle.CreateSnapshot(tick, state, hash);  // every 10 ticks

// Low-frequency (every 1000 ticks)
// + Less storage
// - Slower queries
chronicle.CreateSnapshot(tick, state, hash);  // every 1000 ticks
```

Recommendation: Snapshot every 100-500 ticks for typical games.

### Branch Hygiene

Clean up completed experiments:

```csharp
// After experiment completes
if (experimentSuccessful)
{
    // Extract changes for deployment
    ExtractAndDeploy(branch);
}

// Mark branch inactive (archive)
branch.IsActive = false;

// Prune inactive branches periodically
PruneInactiveBranches(olderThan: 7 * 86400); // 7 days
```

### Retention Policies

Don't keep history forever:

```csharp
var config = new ChronicleConfig
{
    // Keep last 24 hours
    RetentionTicks = 24 * 60 * 60,

    // Max 1000 versions per entity
    MaxVersionsPerEntity = 1000,

    // Vacuum deleted after 24 hours
    VacuumAfterTicks = 24 * 60 * 60
};
```

### State Hashing

Verify integrity with cryptographic hashing:

```csharp
using Atom.Crypto;

var stateHash = XxHash3.Hash64(stateData);

var snapshot = chronicle.CreateSnapshot(
    tick: currentTick,
    stateData: stateData,
    stateHash: stateHash
);

// Later: verify integrity
var isValid = chronicle.VerifySnapshot(
    snapshot.Id,
    hashFunction: XxHash3.Hash64
);
```

---

## Related Documents

- [LatticeDB Storage](./lattice.md) - Underlying storage layer
- [Flux Simulation](./flux.md) - Entity-component simulation
- [Axion Schema](./axion.md) - Schema definition language
- [Executive Overview](../overview/executive.md) - Platform overview and Five Laws

---

**End of Document**

Chronicle brings time-travel to data. What was becomes as accessible as what is.
