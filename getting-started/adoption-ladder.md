# Incremental Adoption Guide

**Last Updated:** 2026-01-06
**Status:** Living Document

---

## Philosophy

You don't have to use everything. Each layer provides standalone value. Adopt what you need, when you need it.

Orix is designed as a **layered platform**, not an all-or-nothing framework. Start small, validate value, expand when ready.

---

## The Adoption Ladder

### Level 0: Just the Math

**What you adopt:** Atom (DFixed64, DVector2/3, DQuaternion, OrixRandom)

**What you get:**
- Cross-platform deterministic calculations
- Bit-identical results on all platforms (Windows, Linux, macOS, WebAssembly)
- Fixed-point math that never drifts
- Seeded random that replays perfectly

**Integration effort:**
- Drop-in replacement for floats in critical paths
- Change `float` to `DFixed64`, `Vector2` to `DVector2`
- Approximately 1-3 days for typical physics system

**Risk:** Near zero - it's just math types

**Use case:** "We just need deterministic physics"

**Example:**
```csharp
// Before
float distance = Vector2.Distance(playerPos, enemyPos);

// After
DFixed64 distance = DVector2.Distance(playerPos, enemyPos);
```

**Who does this:**
- Game studios needing lockstep multiplayer
- Financial systems requiring exact decimal math
- Scientific simulations needing reproducibility

---

### Level 1: Schema-Driven Types

**What you adopt:** Atom + Axion

**What you get:**
- Type-safe serialization across languages (C#, TypeScript, Python)
- Schema evolution without breaking changes
- Cross-language type definitions (define once, use everywhere)
- Binary serialization faster than JSON, safer than hand-rolled formats

**Integration effort:**
- Add `.axion` schema files to your project
- Run `orix axion export csharp` to generate types
- Use generated types for API contracts, storage models
- Approximately 1-2 weeks for typical API surface

**Risk:** Low - schemas are additive, don't replace existing code

**Use case:** "We want type-safe APIs and storage"

**Example:**
```axion
// schemas/game/player.axion
namespace game.player;

entity Player {
    @key
    id: uuid;

    position: vec2;
    health: fixed64;
    inventory: Item[];
}

entity Item {
    id: uuid;
    name: string;
    quantity: int32;
}
```

**Who does this:**
- Teams with multiple client platforms (web, mobile, desktop)
- API-first organizations
- Anyone tired of hand-writing serialization code

---

### Level 2: Deterministic Storage

**What you adopt:** Atom + Axion + Lattice

**What you get:**
- Time-travel queries ("show me the state at 3pm yesterday")
- Chronicle history (every change is an immutable event)
- CRDT support for distributed writes
- Full-text search, spatial indexing
- Audit trails for free

**Integration effort:**
- Replace or augment existing database layer
- Migrate data from old schema to Axion schemas
- Update queries to use LatticeQL
- Approximately 1-3 months depending on data complexity

**Risk:** Medium - storage is foundational, test thoroughly

**Use case:** "We need audit trails and temporal queries"

**Example:**
```csharp
// Time-travel query
var db = LatticeDB.Open("game.db");
var snapshotAt = db.AsOf(Tick.FromDateTime(new DateTime(2026, 1, 1)));
var player = snapshotAt.Query<Player>().Where(p => p.Id == playerId).First();

// Current state
var currentPlayer = db.Query<Player>().Where(p => p.Id == playerId).First();
```

**Who does this:**
- Financial applications (regulatory audit requirements)
- Healthcare systems (HIPAA compliance)
- Games needing replay/spectator modes
- Any system where "show me what happened" is valuable

---

### Level 3: Simulation Runtime

**What you adopt:** Atom + Axion + Flux

**What you get:**
- ECS (Entity Component System) architecture
- Tick-based execution (deterministic frame loop)
- Entity lifecycle management
- System ordering and dependencies
- Deterministic simulation guaranteed

**Integration effort:**
- Restructure game loop / simulation core
- Convert objects to entities + components
- Rewrite update logic as systems
- Approximately 2-6 months for greenfield, longer for migration

**Risk:** Medium-high - architectural change

**Use case:** "We're building a new simulation from scratch" or "We need deterministic replay"

**Example:**
```csharp
// Define components (via Axion schemas)
// schemas/game/components.axion
namespace game.components;

component Position {
    value: vec2;
}

component Velocity {
    value: vec2;
}

// Define system
public class MovementSystem : FluxSystem
{
    public override void Update(Tick tick)
    {
        var entities = World.Query<Position, Velocity>();
        foreach (var (entity, pos, vel) in entities)
        {
            pos.Value += vel.Value * DFixed64.One; // 1 unit per tick
            World.Set(entity, pos);
        }
    }
}
```

**Who does this:**
- Game studios building multiplayer games
- Real-time strategy games
- Physics simulations
- Anyone needing "rewind and replay" functionality

---

### Level 4: Networked Simulation

**What you adopt:** Atom + Axion + Flux + Nexus

**What you get:**
- Multiplayer state sync (deterministic lockstep or client-server)
- Delta compression (only send what changed)
- Authority management (who controls what)
- Input delay, rollback, prediction support

**Integration effort:**
- Replace or augment networking layer
- Design authority model (client-server or peer-to-peer)
- Handle latency, packet loss, cheating
- Approximately 3-6 months

**Risk:** High - affects distributed architecture, test with real network conditions

**Use case:** "We need deterministic multiplayer"

**Example:**
```csharp
// Server broadcasts state delta
var delta = NexusServer.ComputeDelta(previousTick, currentTick);
NexusServer.Broadcast(delta);

// Clients apply delta
NexusClient.OnReceiveDelta(delta => {
    World.ApplyDelta(delta);
});
```

**Who does this:**
- Multiplayer game studios
- Real-time collaboration tools
- Distributed simulations (IoT, robotics)

---

### Level 5: Full Platform

**What you adopt:** Everything (Atom + Axion + Lattice + Flux + Nexus + Echo + Lumen + Arbiter + Airlock)

**What you get:**
- Recording and replay (Echo)
- Time-travel debugging
- Observability (Lumen - logs, metrics, traces)
- Deterministic testing (Arbiter)
- Secrets management (Airlock)

**Integration effort:** Full commitment (6-12+ months)

**Risk:** High investment, high reward

**Use case:** "We want the complete deterministic platform"

**Who does this:**
- Game studios building AAA multiplayer titles
- Financial institutions with strict compliance requirements
- Teams committed to deterministic-first architecture

---

## Mixing and Matching

### You CAN Use:

| Combination | Why It Works |
|-------------|--------------|
| **Atom alone** | Math is standalone, no dependencies |
| **Atom + Axion** | Schemas don't require simulation runtime |
| **Lattice without Flux** | Storage is independent of simulation |
| **Echo without Nexus** | Single-player replay doesn't need networking |
| **Arbiter for any project** | Testing is universal |
| **Lumen for any project** | Observability is universal |

### You CANNOT Use:

| Combination | Why It Fails |
|-------------|--------------|
| **Nexus without Flux** | Networking assumes ECS entity model |
| **Echo without deterministic simulation** | Replay requires bit-identical results |
| **Flux without Atom** | Simulation requires deterministic math |

---

## Migration Paths

### From Unity/Unreal

**Recommended approach:**
1. Start with **Level 0** (Atom math in physics/AI systems)
2. Add **Axion** for save data, network packets
3. Consider **Flux** only if determinism is mandatory (e.g., competitive multiplayer)

**Why incremental?** Unity/Unreal have their own ECS, networking. Replacing them is risky. Use Orix for determinism where it matters.

**Timeline:** 1-3 months (Level 0-1), 6-12 months (Level 3+)

---

### From Custom Engine

**Recommended approach:**
1. Start with **Level 0-1** (math + schemas)
2. **Lattice** can replace existing storage incrementally (dual-write pattern)
3. **Flux** if you're willing to restructure simulation core

**Why incremental?** Custom engines often have technical debt. Prove value before rewriting.

**Timeline:** 2-6 months (Level 0-2), 6-18 months (Level 3+)

---

### Greenfield Project

**Recommended approach:**
- Consider **Level 3+** from the start
- Architectural decisions are easier before code exists
- Start with determinism, don't retrofit later

**Why go big?** No legacy constraints, no migration risk.

**Timeline:** 3-6 months to first playable (Level 3-4)

---

## The Off-Ramp

We want you to succeed, even if you leave.

| If you adopted... | Your exit strategy... |
|-------------------|------------------------|
| **Atom** | Code still works, swap `DFixed64` back to `float`. Determinism is lost, but logic remains. |
| **Axion** | Export schemas to Protobuf or JSON Schema. Generated code becomes reference implementation. |
| **Lattice** | Data is queryable. Export to Postgres, MySQL, or any SQL database. Chronicle events are portable. |
| **Flux** | ECS concepts transfer to Unity DOTS, Bevy, Flecs. Entity/component/system pattern is universal. |
| **Nexus** | Networking code is standard socket/TCP/UDP. Rollback/prediction concepts transfer to other engines. |

**No lock-in.** We believe great platforms are chosen, not mandated.

---

## Risk vs. Reward Matrix

| Level | Integration Effort | Risk | Value if Successful | Recommended For |
|-------|-------------------|------|---------------------|-----------------|
| 0 | Days | Very Low | Medium | Anyone needing deterministic math |
| 1 | Weeks | Low | High | API-first teams, multi-platform |
| 2 | Months | Medium | Very High | Compliance, audit, time-travel |
| 3 | Months | Medium-High | Very High | New simulations, replays |
| 4 | Months | High | Extreme | Multiplayer games |
| 5 | Year+ | Very High | Extreme | Full platform commitment |

---

## Common Questions

### "Can I use Orix in just one service/module?"

Yes. Many teams use Atom + Axion for a critical calculation service, while the rest of the system uses standard tools.

### "What if I need a feature Orix doesn't have?"

1. Check if it's on the roadmap
2. Build it yourself (Orix is open-source)
3. Use `[Ambient]` to integrate external libraries (with caution)

### "How do I convince my team to adopt incrementally?"

Show, don't tell:
1. Build a small proof-of-concept (Level 0 or 1)
2. Measure the value (performance, correctness, time saved)
3. Expand if successful

### "What's the minimum viable adoption?"

**Level 0 (Atom alone).** DFixed64 is a drop-in replacement for float. If it doesn't work, you've lost days, not months.

---

## Success Stories (Hypothetical)

### Game Studio: Started with Level 0

**Problem:** Multiplayer physics desyncs
**Solution:** Replaced `float` with `DFixed64` in physics engine
**Outcome:** Desyncs eliminated, adopted Flux 6 months later
**Timeline:** 1 week proof-of-concept, 3 months full physics migration

### Fintech Startup: Started with Level 1

**Problem:** Price calculations differed across mobile/web
**Solution:** Used Axion schemas for pricing models
**Outcome:** Bit-identical prices, adopted Lattice for audit trails
**Timeline:** 2 weeks schema creation, 1 month integration

### Simulation Company: Started with Level 3

**Problem:** Building new vehicle simulator from scratch
**Solution:** Built on Flux from day one
**Outcome:** Deterministic replay, added Nexus for multi-user
**Timeline:** 6 months to first demo, 12 months to production

---

## Decision Framework

**Ask yourself:**

1. **Do I need determinism?**
   - No → Maybe Orix isn't for you (yet)
   - Yes, in specific areas → Start with Level 0-1
   - Yes, everywhere → Consider Level 3+

2. **Am I migrating or greenfield?**
   - Migrating → Go slow (Level 0 → 1 → 2...)
   - Greenfield → Can jump to Level 3+

3. **What's my risk tolerance?**
   - Low → Start with Atom (days of effort)
   - Medium → Axion + Lattice (weeks to months)
   - High → Full platform (months to year)

4. **What's my timeline?**
   - Short (weeks) → Level 0-1 only
   - Medium (months) → Level 2-3
   - Long (year+) → Level 4-5

---

## Next Steps

1. **Read:** [Getting Started Guide](../getting-started.md)
2. **Try:** Download Orix CLI and run `orix new demo`
3. **Decide:** Which level matches your needs?
4. **Start small:** Prove value before committing
5. **Ask:** Join Discord/Slack if you have questions

---

**Remember:** You're not making a permanent decision. Adopt incrementally, validate value, expand when confident.

The best adoption path is the one that minimizes risk while proving value quickly.

---

**End of Document**
