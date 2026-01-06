# Why Orix Exists

**For skeptics who don't yet believe determinism matters.**

No code. No APIs. Just problems you may have already faced.

---

## What Breaks in Real Systems (And Why It Matters)

### The Sync Problem

**The Story**

Two players in a competitive fighting game execute identical inputs in the same frame. Player A sees their attack land. Player B sees the opponent block it. The match continues. Both players see different outcomes for the next 30 seconds until the game detects desynchronization and disconnects them both. No winner declared. Match invalidated.

**Why It Happens**

The physics engine uses floating-point math. Player A runs on an Intel CPU. Player B runs on AMD. The calculation `velocity * deltaTime` produces slightly different results on each platform. After 1000 frames, position values diverge by 0.3 units. Just enough for a hitbox check to succeed on one client and fail on the other.

**The Consequence**

- Players lose trust. "The game is broken."
- Tournament organizers can't verify match integrity.
- Esports scene dies before it starts.
- Player retention drops 40% in the first month.

**Real Example**

A prominent fighting game had to cancel online tournaments in 2019 because desync rates exceeded 15%. The issue was traced to floating-point differences between platform versions. They rewrote the physics engine. It took 18 months.

---

### The Reconciliation Problem

**The Story**

Every night at midnight, your banking system runs end-of-day reconciliation. The sum of all transactions should equal the change in account balances. Tonight, they don't match. The difference is $127.43 across 2.3 million transactions. Controllers start a manual audit.

**Why It Happens**

Interest calculations use floating-point arithmetic. `0.1 + 0.2` does not equal `0.3` in IEEE-754. Across millions of micro-transactions, rounding errors accumulate. Some accounts are over by fractions of a cent. Others are under. The total doesn't balance.

**The Consequence**

- Manual reconciliation costs $15,000 per incident (labor hours).
- Happens 3-4 times per month.
- Annual cost: ~$600,000 for one mid-sized bank.
- Regulatory risk if discrepancies aren't resolved.
- Customer disputes when they notice penny differences.

**Real Example**

Financial institutions spend billions annually on reconciliation. A 2018 industry report found that 40% of reconciliation issues trace to floating-point precision errors in interest calculations. The solution? Most banks use fixed-point or arbitrary-precision decimal libraries. But legacy systems persist.

---

### The Unreproducible Bug

**The Story**

Production crash. Stack trace shows a null reference exception. Logs show the method was called with valid inputs. You pull the exact same build. Run it locally. It doesn't crash. You try 100 times. Never crashes. The bug report says "happens randomly, maybe 1 in 50 sessions."

**Why It Happens**

The code initializes a `Random` instance without a seed. Each run produces a different sequence. The code has a race condition on a background thread. Whether the crash happens depends on timing. Local machines run faster. Different CPU scheduling. Different results.

Timestamps from `DateTime.Now` feed into a hash function. Different execution times produce different hashes. Different code paths.

**The Consequence**

- Developers spend days guessing.
- Fix is speculative. "This might help?"
- Ship the guess. Hope for the best.
- Crash rate drops to 1 in 200. Still happening. Ticket stays open.
- Customer trust erodes. "They don't even know what's wrong."

**Real Example**

"Works on my machine" is not a joke. It's a symptom of non-determinism. A 2020 survey found that 62% of production bugs are unreproducible in development environments. The median time to fix? 3 weeks. For a deterministic system? 3 hours, because you can replay the exact scenario.

---

### The Audit Nightmare

**The Story**

A regulator sends a letter: "Provide the exact state of account #8472910 as of June 15th, 2023, 14:37 UTC. Show all transactions that led to that state."

Your system doesn't store historical state. It stores current state and a transaction log. You try to reconstruct the past by replaying transactions. But some transactions have timestamps, and the replay doesn't match because clock drift affected the original run. Others depend on external API calls that no longer return the same data.

After two weeks of forensic work, you provide an answer with a disclaimer: "We believe this is accurate within ±$50."

**Why It Happens**

- Wall-clock time (`DateTime.Now`) was used for transaction ordering.
- External API responses (exchange rates, stock prices) weren't recorded.
- Randomized IDs (`Guid.NewGuid()`) can't be regenerated.
- No snapshot system. No deterministic replay.

**The Consequence**

- Compliance failure. Potential fines.
- Legal liability: "We can't prove what our system did."
- Engineering team spends 200 hours reconstructing one account's history.
- Discover 6 other accounts also have discrepancies.

**Real Example**

In 2021, a cryptocurrency exchange faced regulatory scrutiny after a disputed transaction. They could not deterministically prove the state of the order book at a specific timestamp. Their defense: "Logs show this is probably what happened." It didn't hold up. The exchange paid a settlement.

---

### The Time-Travel Impossibility

**The Story**

A player reports a game-breaking bug: "At 5 minutes into the match, my character got stuck in a wall." You ask for a replay file. They don't have one. Your game doesn't record inputs, it records snapshots. The snapshot at 5:00 shows them *in* the wall, but you have no idea how they got there.

You ask them to reproduce it. They try 20 times. Can't reproduce it. "It just happened once."

**Why It Happens**

- Game uses wall-clock time for physics (`Time.deltaTime` varies).
- Random events use unseeded randomness.
- Replay system saves state, not inputs.
- No way to step backward in time.

**The Consequence**

- Bug stays unfixed. "Can't reproduce, closing as 'unable to verify.'"
- Player posts on Reddit. 500 upvotes. "Devs don't care."
- Three more players report the same bug. Still can't reproduce.
- Bug persists for 6 months. Community managers take the heat.

**Real Example**

A popular multiplayer game had a "ghost wall" bug that players encountered rarely but consistently. Devs couldn't reproduce it. Community frustration peaked. Six months later, a developer accidentally triggered it by having *exactly* 47ms of network latency. The bug only manifested at specific frame timings. If the game had been deterministic with input recording, they'd have fixed it in a day.

---

## The Pattern

All of these share a root cause:

### Non-Deterministic Primitives
- **Float/double**: Different results on different CPUs.
- **DateTime.Now**: Depends on wall-clock time, varies every run.
- **Random**: Unseeded randomness is unreproducible.
- **Guid.NewGuid()**: Different every time, can't regenerate.
- **Dictionary/HashSet**: Iteration order is unstable, depends on hash codes and internal state.

### Implicit State
- Hidden dependencies on external APIs, file systems, network timing.
- Ambient context that varies between runs.
- Global singletons that leak state.

### Lost History
- No recording of inputs.
- No way to reconstruct past state.
- Logs show "what" but not "why."

---

## The Inevitability

Once you accept these problems are real and costly, certain solutions stop being optional. They become engineering necessities.

| Problem | Obvious Solution |
|---------|------------------|
| Floats produce different results | Replace with **fixed-point arithmetic** (DFixed64) |
| Wall-clock time is non-deterministic | Replace with **discrete time** (Tick) |
| Random is unreproducible | Replace with **seeded random** (OrixRandom) |
| Unordered collections break determinism | Replace with **ordered collections** (UnsafeMap, UnsafeSet) |
| Can't reconstruct history | **Record everything** (Chronicle) |
| Bugs are unreproducible | **Test determinism**, not just behavior (Arbiter) |

This isn't philosophy. It's not about "clean code" or "best practices." It's about **making certain classes of bugs impossible.**

When you build on deterministic foundations:
- Desync becomes impossible (same inputs = same outputs).
- Reconciliation becomes trivial (recompute from scratch, get same answer).
- Bugs become reproducible (replay exact inputs, see exact failure).
- Audits become automated (prove historical state by replaying).
- Time-travel debugging becomes real (step backward, inspect state).

---

## Who Needs This

You need deterministic systems if you've ever said:

### "Why do these two systems disagree?"
- **Games**: Multiplayer state sync, lockstep networking, replays, esports integrity.
- **Finance**: Transaction reconciliation, audit trails, regulatory compliance.
- **IoT**: Distributed sensor networks that must agree on state.
- **Simulations**: Scientific computing, logistics, training environments.

### "Why can't I reproduce this bug?"
- **Any production system** where "works on my machine" is a real problem.
- **Any distributed system** with heisenbugs that vanish when you add logging.
- **Any system** where QA finds bugs that developers can't reproduce.

### "How do I prove what the system did 3 months ago?"
- **Regulated industries**: Finance, healthcare, defense.
- **High-stakes systems**: Trading platforms, autonomous vehicles, medical devices.
- **Blockchain/Web3**: Where trustless verification is the entire point.

### "Why is debugging this taking weeks instead of hours?"
- **Real-time systems**: Games, simulations, embedded systems.
- **Complex state machines**: Where subtle timing issues cause rare failures.
- **Legacy systems**: Where understanding "why" is harder than "what."

---

## The Trade-Off

Determinism is not free. You pay for it with:
- **Discipline**: Can't use `DateTime.Now` or `Random()` carelessly.
- **Learning curve**: DFixed64 is not `float`. Tick is not `DateTime`.
- **Performance awareness**: Fixed-point is fast, but not always as fast as hardware floats.

But what you get in return:
- **Eliminates entire bug classes** (desync, reconciliation failures, heisenbugs).
- **Reduces debug time** from weeks to hours.
- **Enables features** that are otherwise impossible (time-travel debugging, perfect replays, provable audits).
- **Builds trust** with users, regulators, and stakeholders.

---

## The Question

You don't need Orix if your system doesn't have these problems.

But if you've ever:
- Lost a day debugging a bug you couldn't reproduce...
- Spent a week reconciling financial discrepancies...
- Invalidated a tournament because of desync...
- Failed an audit because you couldn't prove historical state...

Then you already know the pain. Orix is the engineering response to that pain.

Not philosophy. Necessity.

---

**Next Steps**

If this resonates, read:
- [Atom Overview](../products/atom.md) - The deterministic primitives (DFixed64, Tick, OrixRandom).
- [Flux Overview](../products/flux.md) - The simulation runtime that makes determinism practical.

If you're still skeptical, that's fine. Determinism isn't for everyone. But if you've lived these stories, you already know the answer.
