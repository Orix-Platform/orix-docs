# Orix Platform Glossary

**Version:** 1.0.0
**Status:** CANONICAL
**Last Updated:** 2026-01-06

This is the **single source of truth** for all Orix terminology. Other documents MUST reference this glossary instead of redefining terms.

---

## Core Concepts

### Determinism
**Definition:** The property that identical inputs and initial state produce bit-identical outputs across all platforms, all runs, forever.
**Context:** Foundation of all Orix simulation, networking, and replay systems. Enforced at compile-time via analyzers.
**Example:** Running the same simulation twice with seed 42 produces the exact same final state hash `0xDEADBEEF` on both Windows and Linux.
**See also:** The Five Laws, Tick, State Hash

### Discreteness
**Definition:** The property that all numeric values are quantized as Q32.32 signed fixed-point, eliminating platform-dependent floating-point behavior.
**Context:** Second Law of Orix. Applies to all simulation math, physics, and game logic.
**Example:** `DFixed64.Parse("1.5")` represents exactly 1.5, not 1.4999999... or 1.5000001.
**See also:** DFixed64, Fixed-Point Arithmetic, The Five Laws

### Causality
**Definition:** The property that time advances only in discrete ticks, with wall-clock time never influencing simulation state.
**Context:** Third Law of Orix. Ensures frame-rate independence and replay determinism.
**Example:** A projectile's position updates based on `Tick` count, not `DateTime.Now`.
**See also:** Tick, The Five Laws, [Ambient]

### Schema Authority
**Definition:** The principle that all data types crossing boundaries (API, storage, network) MUST originate from Axion schema definitions, not hand-written code.
**Context:** Fourth Law of Orix. Enforced via ORIX100-series analyzers.
**Example:** Instead of `class User { public string Name; }`, define `entity User { name: string; }` in an `.axion` file and generate code via Fusion.
**See also:** Axion Schema, Code Generation, Schema Hash

### Traceability
**Definition:** The requirement that every claim about behavior, performance, or correctness has reproducible evidence.
**Context:** Fifth Law of Orix. Applies to all assertions, documentation, and design decisions.
**Example:** "This algorithm is O(n)" requires a benchmark artifact. "It works" requires passing Arbiter scenarios.
**See also:** Arbiter Scenario, Verification Protocol

### The Five Laws
**Definition:** The immutable constitutional rules of Orix: Determinism, Discreteness, Causality, Schema Authority, Traceability. Violation of any law is CRITICAL.
**Context:** Foundational governance enforced across all products.
**Example:** Using `float` violates Discreteness. Using `DateTime.Now` violates Causality.
**See also:** Determinism, Discreteness, Causality, Schema Authority, Traceability

---

## Time & State

### Tick
**Definition:** The discrete unit of simulation time, represented as a 64-bit unsigned integer (`ulong`). Time advances only in tick increments.
**Context:** The only valid time representation in Orix simulation code. Replaces `DateTime`, `TimeSpan`, and wall-clock concepts.
**Example:** `if (clock.CurrentTick > spawnTick + 100) { SpawnEnemy(); }` spawns an enemy 100 ticks after a spawn time.
**See also:** Causality, Timeline, Snapshot

### State
**Definition:** The complete set of simulation data at a given tick, including all entity component values, system state, and random number generator state.
**Context:** Must be fully serializable and deterministically reproducible. Used for snapshots, rollback, and verification.
**Example:** A game's state includes player positions, health values, inventory contents, and enemy AI states.
**See also:** State Hash, Snapshot, World

### State Hash
**Definition:** A cryptographic hash (e.g., SHA-256) computed over the entire simulation state, used to verify determinism and detect divergence.
**Context:** Two simulations with identical inputs should produce identical state hashes at the same tick.
**Example:** After 1000 ticks with seed 42, both client and server compute state hash `0x7A3F...`. If hashes differ, a desync occurred.
**See also:** Determinism, Desync, Verification Protocol

### Snapshot
**Definition:** A serialized copy of the complete simulation state at a specific tick, enabling time-travel, rollback, and persistence.
**Context:** Used by Chronicle for temporal queries, Echo for replay keyframes, and Nexus for client prediction rollback.
**Example:** Chronicle saves a snapshot every 100 ticks, allowing `AsOf(tick - 500)` queries.
**See also:** State, Keyframe, Chronicle, AsOf Query

### Timeline
**Definition:** A linear sequence of ticks representing the progression of simulation state from initialization to current tick.
**Context:** In replay systems, timelines can branch when different inputs are applied at a divergence point.
**Example:** Main timeline runs to tick 1000. Replaying with different inputs creates a branch at tick 500.
**See also:** Tick, Branch, Chronicle

### Branch
**Definition:** A divergent timeline created when replaying from a snapshot with different inputs, used for time-travel debugging and what-if analysis.
**Context:** Echo and Chronicle support branching to explore alternate outcomes without mutating the main timeline.
**Example:** Load snapshot at tick 500, apply different player inputs, create branch timeline to explore alternate strategy.
**See also:** Timeline, Snapshot, Echo

---

## Math & Primitives

### DFixed64
**Definition:** A Q32.32 signed fixed-point numeric type representing values with 32 bits for the integer part and 32 bits for the fractional part.
**Context:** The only numeric type allowed for simulation math. Range: -2,147,483,648.0 to +2,147,483,647.999999999767.
**Example:** `DFixed64 speed = DFixed64.Parse("5.75");` represents exactly 5.75 in all platforms.
**See also:** Discreteness, Fixed-Point Arithmetic, DVector2

### Fixed-Point Arithmetic
**Definition:** A numeric representation where values are stored as integers scaled by a fixed power of 2, ensuring deterministic cross-platform math.
**Context:** Avoids floating-point precision issues and platform-dependent rounding behavior.
**Example:** `DFixed64(5.5) + DFixed64(2.25) = DFixed64(7.75)` produces bit-identical results on ARM, x86, and WASM.
**See also:** DFixed64, Discreteness

### DVector2
**Definition:** A 2D vector with `DFixed64` components (X, Y), used for deterministic 2D positions, velocities, and directions.
**Context:** Replaces `Vector2` or `System.Numerics.Vector2` in simulation code.
**Example:** `DVector2 position = new DVector2(DFixed64.Parse("10.5"), DFixed64.Parse("20.25"));`
**See also:** DFixed64, DVector3, DQuaternion

### DVector3
**Definition:** A 3D vector with `DFixed64` components (X, Y, Z), used for deterministic 3D positions, velocities, and directions.
**Context:** Replaces `Vector3` or `System.Numerics.Vector3` in simulation code.
**Example:** `DVector3 velocity = new DVector3(DFixed64.One, DFixed64.Zero, DFixed64.MinusOne);`
**See also:** DFixed64, DVector2, DQuaternion

### DQuaternion
**Definition:** A quaternion with `DFixed64` components (X, Y, Z, W), used for deterministic 3D rotations.
**Context:** Replaces `Quaternion` or `System.Numerics.Quaternion` in simulation code.
**Example:** `DQuaternion rotation = DQuaternion.FromEuler(DFixed64.Zero, DFixed64.Parse("90"), DFixed64.Zero);`
**See also:** DFixed64, DVector3

### OrixRandom
**Definition:** A seeded deterministic pseudo-random number generator (PCG algorithm) that produces identical sequences given the same seed.
**Context:** The only random number generator allowed in simulation code. Replaces `System.Random`.
**Example:** `var rng = new OrixRandom(42); var roll = rng.NextDFixed64(DFixed64.One, DFixed64.Parse("6"));` generates deterministic dice rolls.
**See also:** Determinism, Seed

---

## Identity

### Entity
**Definition:** A unique 64-bit identifier allocated deterministically from a monotonic counter, used in ECS to identify logical game objects.
**Context:** Replaces `Guid` in simulation code. Consists of an index and a generation counter to detect use-after-free.
**Example:** `Entity player = world.CreateEntity();` allocates entity `#1`, then `#2`, etc., deterministically.
**See also:** Generation, Component, World, ECS

### Generation
**Definition:** A counter embedded in an `Entity` ID that increments when an entity is destroyed and its ID is recycled, detecting stale references.
**Context:** Prevents accessing components of a destroyed entity when its ID is reused.
**Example:** Entity `#1 gen 0` is destroyed. Next entity reusing that slot becomes `#1 gen 1`. Accessing `#1 gen 0` now fails.
**See also:** Entity, Component

### Component
**Definition:** A plain data structure (Axion-generated) attached to an entity, storing specific attributes like position, health, or inventory.
**Context:** In ECS, components are data-only; logic resides in systems. All components must be Axion-defined for serialization.
**Example:** `struct PositionComponent { DVector2 position; }` attached to entity `#5`.
**See also:** Entity, System, World, Schema Authority

### System
**Definition:** A unit of logic that operates on entities with specific component combinations, processing simulation logic each tick.
**Context:** In ECS, systems are stateless (or explicitly managed state). They query the world for matching entities.
**Example:** `MovementSystem` queries entities with `PositionComponent` and `VelocityComponent`, updating positions each tick.
**See also:** Component, Entity, World, Tick

### World
**Definition:** The container for all entities, components, systems, and resources in a simulation instance.
**Context:** Manages entity lifecycle, component storage, system execution order, and snapshot serialization.
**Example:** `var world = new World(); var player = world.CreateEntity(); world.AddComponent(player, new HealthComponent { value = 100 });`
**See also:** Entity, Component, System, State

---

## Collections

### UnsafeMap
**Definition:** A deterministic key-value collection with insertion-ordered iteration, replacing `Dictionary<K,V>` in simulation code.
**Context:** Guarantees stable iteration order across platforms, avoiding non-deterministic hash table ordering.
**Example:** `var scores = new UnsafeMap<Entity, int>(); scores[player] = 100;` iterates in insertion order.
**See also:** UnsafeSet, UnsafeList, Deterministic Iteration

### UnsafeSet
**Definition:** A deterministic unique-value collection with insertion-ordered iteration, replacing `HashSet<T>` in simulation code.
**Context:** Guarantees stable iteration order across platforms.
**Example:** `var tags = new UnsafeSet<string>(); tags.Add("boss"); tags.Add("enemy");` iterates in insertion order.
**See also:** UnsafeMap, UnsafeList, Deterministic Iteration

### UnsafeList
**Definition:** A pre-allocated, low-allocation dynamic array for hot-path simulation code, replacing `List<T>` in performance-critical contexts.
**Context:** Minimizes GC pressure in per-tick operations. Intentionally named "Unsafe" to signal performance-critical usage.
**Example:** `var entities = new UnsafeList<Entity>(capacity: 1000);` pre-allocates to avoid reallocations.
**See also:** UnsafeMap, UnsafeSet, Hot Path

### Deterministic Iteration
**Definition:** The property that iterating a collection produces elements in the same order across all platforms and runs.
**Context:** Required for determinism. `Dictionary<K,V>` and `HashSet<T>` have platform-dependent iteration order and are forbidden.
**Example:** `UnsafeMap<int, string>` iterates in insertion order: add (1, "a"), (3, "c"), (2, "b") → iterates as 1, 3, 2.
**See also:** UnsafeMap, UnsafeSet, Determinism

---

## Schema

### Axion Schema
**Definition:** A declarative schema definition language (.axion files) that defines data structures, constraints, and annotations for code generation.
**Context:** The single source of truth for all Orix data types. Replaces hand-written models.
**Example:** `entity Player { @key id: uuid; name: string; @range(0, 100) health: i32; }`
**See also:** Schema Authority, Code Generation, Schema Hash, Annotation

### Schema Hash
**Definition:** A cryptographic hash of an Axion schema file, embedded in generated code via `[AxionGenerated(Hash = "...")]`, used to detect schema drift.
**Context:** If generated code is modified, the hash no longer matches, triggering ORIX101 analyzer error.
**Example:** Modifying a generated type without regenerating causes build failure: "Schema hash mismatch."
**See also:** Axion Schema, Code Generation, Schema Authority

### Code Generation
**Definition:** The process of transforming Axion schemas into target language code (C#, TypeScript, Python) via Fusion emitters.
**Context:** All application data types must be generated, never hand-written. Ensures schema-code synchronization.
**Example:** `orix axion export csharp schemas/player.axion -o src/Game/Game.Generated` generates C# types.
**See also:** Axion Schema, Fusion, Schema Authority, Schema Hash

### Annotation
**Definition:** Metadata attached to Axion schema fields (e.g., `@key`, `@indexed`, `@encrypted`, `@range`) to specify constraints, indexing, or encryption requirements.
**Context:** Annotations drive code generation, database schema creation, and validation logic.
**Example:** `@range(0, 100) health: i32;` generates validation ensuring health stays between 0-100.
**See also:** Axion Schema, Code Generation

---

## Storage

### Chronicle
**Definition:** Orix's temporal database layer, storing state snapshots at tick granularity with time-travel query capabilities.
**Context:** Built on LatticeDB. Enables querying historical state via `AsOf(tick)` syntax.
**Example:** `chronicle.Query<Player>().AsOf(tick - 500).Where(p => p.Health > 0)` retrieves alive players from 500 ticks ago.
**See also:** Temporal Query, AsOf Query, Snapshot, LatticeDB

### Temporal Query
**Definition:** A query that retrieves data as it existed at a specific tick, enabling time-travel analysis and debugging.
**Context:** Supported by Chronicle via snapshot-based reconstruction.
**Example:** `AsOf(1000)` retrieves state at tick 1000, even if current tick is 5000.
**See also:** Chronicle, AsOf Query, Tick

### AsOf Query
**Definition:** A temporal query syntax (`AsOf(tick)`) specifying the tick at which to retrieve historical data.
**Context:** Used in Chronicle queries to reconstruct state from snapshots and deltas.
**Example:** `db.Query<Inventory>().AsOf(previousTick).Where(i => i.Gold > 100)`
**See also:** Chronicle, Temporal Query, Snapshot

### CRDT
**Definition:** Conflict-Free Replicated Data Type — a data structure designed to merge concurrent updates deterministically without coordination.
**Context:** Used in Lattice for multi-master replication and offline-first sync. Examples: LWW-Register, OR-Set, PN-Counter.
**Example:** Two clients increment a PN-Counter offline; both updates merge without conflict when syncing.
**See also:** LatticeDB, Replication

### LatticeDB
**Definition:** Orix's embedded deterministic database, supporting Axion-typed storage, indexing, CRDT replication, and Chronicle's temporal queries.
**Context:** Foundation for all Orix persistence, replacing traditional SQL/NoSQL for simulation state.
**Example:** `var db = new LatticeDB("game.lattice"); db.Insert(player);`
**See also:** Chronicle, CRDT, Axion Schema

---

## Networking

### Authority (Networking)
**Definition:** The designation of which peer (client or server) has the authoritative version of a specific entity or component, used to resolve conflicts.
**Context:** In client-server architectures, the server is typically authoritative. In peer-to-peer, authority may be distributed.
**Example:** Server is authoritative for player health; client is authoritative for camera position (local-only).
**See also:** Client Prediction, Server Reconciliation, Lockstep

### Delta Compression
**Definition:** A technique that transmits only changed data (deltas) instead of full state, reducing bandwidth.
**Context:** Nexus uses delta compression to sync only modified components since the last acknowledged tick.
**Example:** If only player position changed, send `{ entity: #5, PositionComponent: { x: 10.5, y: 20.0 } }` instead of full state.
**See also:** Snapshot, Nexus

### Interest Management
**Definition:** A technique that filters networked entities based on relevance (e.g., proximity, visibility) to reduce bandwidth and processing.
**Context:** Nexus supports spatial and visibility-based interest management to avoid sending irrelevant entities.
**Example:** Only send entities within 100 units of the player; distant entities are not transmitted.
**See also:** Nexus, Delta Compression

### Lockstep
**Definition:** A networking model where all peers simulate identical inputs on identical ticks, maintaining perfect state synchronization.
**Context:** Requires determinism and low input latency. Used in RTS games and deterministic multiplayer.
**Example:** All clients receive inputs for tick 100, simulate tick 100, verify state hash matches.
**See also:** Determinism, State Hash, Client Prediction

### Client Prediction
**Definition:** A technique where clients optimistically simulate authoritative entities locally to hide latency, later reconciling with server corrections.
**Context:** Used in client-server architectures for responsive player movement despite network delay.
**Example:** Client predicts player movement locally; when server correction arrives, client rolls back and replays inputs.
**See also:** Server Reconciliation, Rollback, Authority (Networking)

### Server Reconciliation
**Definition:** The process of correcting client-predicted state with authoritative server state, ensuring eventual consistency.
**Context:** When server state differs from client prediction, client rewinds to the divergence point and replays inputs.
**Example:** Client predicted position (10, 20); server says (10.5, 19.8). Client snaps or interpolates to server position.
**See also:** Client Prediction, Authority (Networking), Rollback

---

## Replay

### Recording
**Definition:** The process of capturing all simulation inputs (commands, events) and periodic snapshots to enable deterministic replay.
**Context:** Echo records inputs and keyframes; replaying the same inputs from the same snapshot reproduces identical state.
**Example:** Recording stores player inputs at ticks 1-1000 and snapshots every 100 ticks.
**See also:** Playback, Keyframe, Echo, Determinism

### Playback
**Definition:** The process of replaying recorded inputs from a snapshot to reconstruct simulation state for debugging or analysis.
**Context:** Determinism guarantees playback produces bit-identical results to the original run.
**Example:** Load snapshot at tick 0, replay inputs from recording, verify state hash matches original at tick 1000.
**See also:** Recording, Keyframe, Echo, Replay Verification

### Keyframe
**Definition:** A full state snapshot saved during recording at regular intervals, enabling fast-forward and time-travel in replay.
**Context:** Without keyframes, rewinding requires replaying from tick 0. Keyframes allow jumping to tick 900 instantly.
**Example:** Save keyframes every 500 ticks. To replay from tick 1200, load keyframe at tick 1000 and simulate 200 ticks.
**See also:** Snapshot, Recording, Playback

### Desync
**Definition:** A divergence between expected and actual simulation state, detected via state hash mismatch, indicating a determinism violation.
**Context:** Desyncs break replay and networked multiplayer. Cause: floating-point use, unordered iteration, ambient time, etc.
**Example:** Client state hash `0xABCD` != server state hash `0x1234` at tick 500 → desync detected.
**See also:** State Hash, Divergence, Determinism, Replay Verification

### Divergence
**Definition:** A point in simulation where state differs from expected, causing replay to fail or networked peers to desynchronize.
**Context:** Can be intentional (branching) or unintentional (desync). Detected via state hash comparison.
**Example:** Replaying with different inputs creates intentional divergence at tick 300 (branch). Non-deterministic code creates unintentional divergence (bug).
**See also:** Desync, State Hash, Branch

---

## Verification

### Arbiter Scenario
**Definition:** A deterministic test case marked with `[ArbiterScenario(Seed = ...)]`, ensuring reproducible behavior and enabling replay-based verification.
**Context:** All Arbiter tests must be deterministic, using seeded randomness and tick-based time.
**Example:** `[ArbiterScenario(Seed = 42)] public void Combat_DealsDamage() { ... }`
**See also:** Seed, Determinism Test, Replay Verification

### Seed
**Definition:** An integer used to initialize a deterministic random number generator, ensuring identical random sequences across runs.
**Context:** All randomness in Orix must be seeded. Arbiter scenarios specify seeds explicitly.
**Example:** `OrixRandom rng = new OrixRandom(42);` produces the same sequence every run.
**See also:** OrixRandom, Arbiter Scenario, Determinism

### Replay Verification
**Definition:** A test that records a scenario, replays it, and verifies the final state hash matches the original, proving determinism.
**Context:** Arbiter's determinism test suite uses replay verification to detect non-deterministic code.
**Example:** `orix arbiter determinism` runs all scenarios twice and compares state hashes.
**See also:** Arbiter Scenario, State Hash, Playback

### Determinism Test
**Definition:** A test specifically designed to verify that identical inputs produce identical outputs, enforcing the First Law.
**Context:** Includes replay verification, state hash comparison, and analyzer checks.
**Example:** Run simulation with seed 42 → state hash `0xDEADBEEF`. Repeat → state hash `0xDEADBEEF`. Pass.
**See also:** Determinism, Arbiter Scenario, Replay Verification

---

## Observability

### Tick Correlation
**Definition:** The practice of associating logs, metrics, and traces with the simulation tick at which they occurred, enabling time-travel debugging.
**Context:** Lumen automatically tags all observability data with `Tick` for correlation with Chronicle and Echo.
**Example:** Log entry `[Tick 1234] Player died` allows jumping to tick 1234 in replay to investigate.
**See also:** Tick, Lumen, Chronicle

### Ambient
**Definition:** An attribute (`[Ambient]`) marking code that uses non-deterministic operations (I/O, wall-clock time, true randomness), explicitly documenting the violation.
**Context:** Required for all non-deterministic code. Must include `Reason` parameter explaining necessity.
**Example:** `[Ambient(Reason = "External payment API requires real timestamps")] public async Task Charge() { ... }`
**See also:** [Ambient] Operations, Determinism, Causality

### [Ambient] Operations
**Definition:** Operations marked with the `[Ambient]` attribute, explicitly allowed to violate determinism for I/O, external APIs, or infrastructure concerns.
**Context:** Forbidden in simulation namespaces. Allowed only at system boundaries (file I/O, network calls, database writes).
**Example:** `[Ambient] DateTime.UtcNow` for logging timestamps (not simulation time).
**See also:** Ambient, Determinism, Causality

---

## Product-Specific Terms

### Fusion
**Definition:** The code generation subsystem of Axion, transforming `.axion` schemas into target language code (C#, TypeScript, Python, etc.).
**Context:** Implements emitters for each language, handling serialization, validation, and schema hash embedding.
**Example:** `orix axion export typescript schemas/game.axion -o sdk/ts/generated`
**See also:** Axion Schema, Code Generation, Schema Authority

### Flux
**Definition:** Orix's deterministic simulation runtime, providing tick-based ECS, lifecycle management, and system execution.
**Context:** Foundation for all game logic and simulation. Enforces determinism at runtime.
**Example:** `var world = new FluxWorld(); world.Tick();` advances simulation by one tick.
**See also:** Tick, World, System, Component

### Nexus
**Definition:** Orix's deterministic networking layer, providing state synchronization, delta compression, and interest management.
**Context:** Supports client-server and lockstep models. Built on deterministic serialization.
**Example:** `nexus.Sync(world, tick);` synchronizes world state with remote peers.
**See also:** Delta Compression, Authority (Networking), Lockstep

### Echo
**Definition:** Orix's replay and time-travel debugging system, recording inputs and snapshots for deterministic playback.
**Context:** Enables bug reproduction, regression testing, and what-if analysis.
**Example:** `echo.Record(world); ... echo.Replay(recording);` captures and replays simulation.
**See also:** Recording, Playback, Keyframe, Replay Verification

### Lumen
**Definition:** Orix's observability platform, providing tick-correlated logging, metrics, and tracing.
**Context:** All logs and metrics are tagged with tick, enabling time-travel correlation with Chronicle and Echo.
**Example:** `lumen.Log(LogLevel.Info, "Player spawned", tick);`
**See also:** Tick Correlation, Chronicle, Echo

### Arbiter
**Definition:** Orix's verification framework, providing deterministic testing, replay verification, benchmarks, and property-based tests.
**Context:** Enforces the Five Laws via automated testing and CI integration.
**Example:** `orix arbiter run` executes all test scenarios and verifies determinism.
**See also:** Arbiter Scenario, Determinism Test, Replay Verification

---

## Related Concepts

### Hot Path
**Definition:** Code executed with high frequency (e.g., per-tick, per-frame), where allocations and inefficiencies severely impact performance.
**Context:** Hot paths must use pre-allocated collections (UnsafeList), avoid boxing, and minimize GC pressure.
**Example:** `MovementSystem.Update()` runs every tick for thousands of entities → hot path.
**See also:** UnsafeList, Tick, System

### Rollback
**Definition:** The process of rewinding simulation state to a previous tick, typically from a snapshot, to replay with corrected inputs.
**Context:** Used in client prediction, server reconciliation, and time-travel debugging.
**Example:** Server correction arrives; client loads snapshot at tick 95, replays inputs 96-100 with server correction.
**See also:** Snapshot, Client Prediction, Server Reconciliation

### Replication
**Definition:** The process of synchronizing data across multiple peers (clients, servers, databases) while maintaining consistency.
**Context:** Lattice uses CRDTs for conflict-free replication. Nexus uses delta compression for state replication.
**Example:** Two clients edit the same document offline; CRDT replication merges changes without conflicts.
**See also:** CRDT, LatticeDB, Nexus

### ECS (Entity Component System)
**Definition:** An architectural pattern separating identity (entities), data (components), and logic (systems) for performance and flexibility.
**Context:** Flux implements deterministic ECS with Axion-typed components and tick-based systems.
**Example:** Entity #5 has `PositionComponent`, `HealthComponent`. `MovementSystem` updates positions each tick.
**See also:** Entity, Component, System, Flux

### Verification Protocol
**Definition:** The mandatory four-step validation process (V0: schemas, V1: build, V2: tests, V3: determinism) required before claiming work complete.
**Context:** Enforced by agents and CI. Ensures all changes meet Orix standards.
**Example:**
```bash
orix check schemas           # V0
dotnet build /warnaserror    # V1
orix arbiter run             # V2
orix arbiter determinism     # V3
```
**See also:** The Five Laws, Arbiter, Schema Authority

---

## Terminology Contrasts

### State vs Snapshot
- **State:** The current, live simulation data in memory.
- **Snapshot:** A serialized copy of state at a specific tick, saved for persistence or rollback.

### Authority (Schema) vs Authority (Networking)
- **Schema Authority:** Principle that schemas define data types.
- **Networking Authority:** Designation of which peer's version of data is correct.

### Tick vs Timestamp
- **Tick:** Discrete simulation time unit (ulong).
- **Timestamp:** Wall-clock time (DateTime). Forbidden in simulation, allowed with `[Ambient]`.

### Desync vs Divergence
- **Desync:** Unintentional state mismatch (bug).
- **Divergence:** Any difference in state (can be intentional, e.g., branching).

---

## Document History

| Version | Date       | Changes                          |
|---------|------------|----------------------------------|
| 1.0.0   | 2026-01-06 | Initial canonical glossary       |

---

**End of Glossary**

For questions or missing terms, open an issue or submit a PR.
