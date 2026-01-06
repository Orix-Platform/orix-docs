# Orix Platform

**Deterministic Simulation Platform**

Orix guarantees identical behavior across all devices, operating systems, and runs when given the same inputs.

---

## Quick Start

!!! question "Why Orix?"
    Understand the problems Orix solves before diving into the solution.

    **[Read Why Orix →](getting-started/why-orix.md)**

!!! example "Hello World"
    Prove determinism in 5 minutes with 42 lines of code.

    **[Try Hello World →](getting-started/hello-world.md)**

!!! tip "Adoption Guide"
    Start small. Adopt incrementally. No "all or nothing."

    **[See Adoption Ladder →](getting-started/adoption-ladder.md)**

---

## The Five Laws

Orix is built on five immutable laws:

| Law | Statement |
|-----|-----------|
| **Determinism** | Same inputs → identical outputs, always |
| **Discreteness** | All values as Q32.32 fixed-point (DFixed64) |
| **Causality** | Time = Ticks only, no wall-clock in simulation |
| **Schema Authority** | All types from .axion schemas |
| **Traceability** | Every claim needs reproducible evidence |

---

## Platform Architecture

```
+-----------------------------------------------------------+
|                         PACKS                             |
|                    (Domain Extensions)                    |
+-----------------------------------------------------------+
| AIRLOCK | FLUX | NEXUS | ECHO | LUMEN | ARBITER           |
+-----------------------------------------------------------+
|                        LATTICE                            |
|                   (Storage + Query)                       |
+-----------------------------------------------------------+
|                         AXION                             |
|                   (Schema + Codegen)                      |
+-----------------------------------------------------------+
|                          ATOM                             |
|                 (Deterministic Primitives)                |
+-----------------------------------------------------------+
```

---

## Products at a Glance

| Product | Purpose | Key Feature |
|---------|---------|-------------|
| [**Atom**](products/atom.md) | Foundation | DFixed64 replaces float/double |
| [**Axion**](products/axion.md) | Schema | Code generation from `.axion` schemas |
| [**Lattice**](products/lattice.md) | Storage | Chronicle time-travel queries |
| [**Flux**](products/flux.md) | Simulation | Tick-based ECS runtime |
| [**Nexus**](products/nexus.md) | Networking | Delta compression + deterministic sync |
| [**Echo**](products/echo.md) | Replay | Record once, replay identically |
| [**Lumen**](products/lumen.md) | Observability | Tick-correlated logging |
| [**Arbiter**](products/arbiter.md) | Testing | Determinism verification |
| [**Airlock**](products/airlock.md) | Secrets | AES-256-GCM vault |

---

## Who Is This For?

- **Gaming**: Multiplayer sync, replay systems, esports
- **Finance**: Auditable calculations, reconciliation
- **IoT**: Distributed simulations, digital twins
- **Distributed Systems**: Reproducible behavior, time-travel debugging

---

## Reading Paths

| I am a... | Start here |
|-----------|------------|
| Skeptic | [Why Orix?](getting-started/why-orix.md) |
| Executive | [Business Overview](overview/business.md) |
| Developer | [Hello World](getting-started/hello-world.md) |
| Evaluator | [Adoption Guide](getting-started/adoption-ladder.md) |

For detailed navigation, see the [Glossary](reference/glossary.md) for terminology.

---

!!! tip "Core Insight"
    Orix is a **deterministic execution substrate** - a new kind of runtime for systems where reproducibility matters more than best-effort performance.
