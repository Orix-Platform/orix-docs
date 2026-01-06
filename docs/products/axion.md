# Axion: Schema-First Data Contracts

**What it is:** A type-safe schema language and code generation system for defining deterministic data structures across multiple languages.

**Status:** Production-ready. Used across all Orix products (Airlock, Lattice, Flux, Echo, Nexus, Lumen).

---

## Why Axion Exists

### The Problem: Schema Drift and Data Chaos

Hand-written data models create multiple sources of truth:

```
TYPICAL PROJECT TODAY:
┌────────────────────────────────────────────────────────┐
│  Backend:     C# classes (hand-written)                │
│  Frontend:    TypeScript interfaces (hand-written)     │
│  Database:    SQL schema (hand-written)                │
│  API Docs:    OpenAPI spec (hand-written)              │
│  Validation:  Separate validator code (hand-written)   │
│                                                         │
│  Result: 5 sources of truth, constant drift            │
└────────────────────────────────────────────────────────┘
```

**Consequences:**
- Type mismatches between client and server
- Serialization bugs from manual coding
- No single source of truth for data structures
- Cross-language incompatibilities
- Breaking changes not caught until runtime

---

## How Axion Solves It

### Single Source of Truth

```
AXION APPROACH:
┌────────────────────────────────────────────────────────┐
│                                                         │
│               schema/user.axion                         │
│               (SINGLE SOURCE)                           │
│                      │                                  │
│          ┌───────────┼───────────┐                     │
│          │           │           │                     │
│          ▼           ▼           ▼                     │
│       C# code   TypeScript   SQL DDL                   │
│                  interfaces  (optional)                 │
│                                                         │
│  Result: 1 source of truth, zero drift                 │
└────────────────────────────────────────────────────────┘
```

### Technical Approach

1. **Define once** in `.axion` schema files
2. **Generate code** for C#, TypeScript, Python, Rust
3. **Guaranteed type safety** across all languages
4. **Binary serialization** with schema versioning
5. **Annotations** for validation, encryption, indexing

---

## What Axion Provides

### 1. Schema Language (.axion files)

Human-readable schema definition with:
- Type declarations (`entity`, `component`, `enum`, `config`)
- Field annotations (`@key`, `@indexed`, `@encrypted`, `@range`)
- Built-in validation constraints
- Documentation as first-class citizen

### 2. Code Generation (Fusion)

Multi-target code generation from single schema:
- **C#**: Structs with serialization, equality, Lattice ORM
- **TypeScript**: Interfaces with validation, Zod schemas
- **Python**: Dataclasses with validation (planned)
- **Rust**: Structs with serde (planned)

### 3. Binary Serialization Format

Deterministic binary encoding with:
- 90%+ compression vs JSON
- Schema hash validation
- Field-level encryption support
- Varint encoding for compact integers
- Ordered iteration for determinism

### 4. Runtime Validation

Schema-driven validation at runtime:
- Type checking
- Range constraints (`@range(min, max)`)
- String patterns (`@pattern(regex)`)
- Custom validation rules

---

## Schema Language Example

```axion
/// =============================================================================
/// Airlock Secrets Schema
/// =============================================================================
///
/// Core data models for the Airlock secrets manager.
/// Defines Secret entity with field-level encryption support.

namespace airlock.secrets

// =============================================================================
// Secret Types
// =============================================================================

/// Classification of secret types
enum SecretType {
    GenericSecret = 0,
    Password = 1,
    ApiKey = 2,
    Token = 3,
    SshKey = 4,
    DatabaseCred = 5,
    Certificate = 6,
    SecureNote = 7
}

// =============================================================================
// Secret Entity
// =============================================================================

/// A secret stored in the vault.
/// Supports field-level encryption, indexing, and versioning.
entity Secret {
    /// Unique identifier
    @key
    id: uuid,

    /// Display name (encrypted, indexed for blind search)
    @encrypted(group: "metadata")
    @indexed
    name: string,

    /// Type classification
    secretType: SecretType = GenericSecret,

    /// The actual secret value (encrypted at rest)
    @encrypted(group: "sensitive")
    value: string,

    /// Tags for organization (encrypted, indexed)
    @encrypted(group: "metadata")
    @indexed
    tags: list<string>,

    /// Optional notes (encrypted)
    @encrypted(group: "metadata")
    notes: string?,

    /// Creation timestamp
    created_at: timestamp,

    /// Last modification timestamp
    updated_at: timestamp,

    /// Optional expiration timestamp
    expires_at: timestamp?
}
```

### Key Features Demonstrated

| Feature | Example | Purpose |
|---------|---------|---------|
| **Triple-slash comments** | `/// A secret stored...` | Documentation |
| **Type safety** | `id: uuid` | Explicit types |
| **Annotations** | `@key`, `@encrypted` | Metadata |
| **Default values** | `= GenericSecret` | Sensible defaults |
| **Optional fields** | `notes: string?` | Nullability |
| **Collections** | `list<string>` | Complex types |
| **Enums** | `SecretType` | Constrained values |

---

## Generated Code Example

### Input Schema (Flux Simulation)

```axion
namespace flux.simulation

/// Represents a discrete unit of simulation time
entity Tick {
    /// Tick number (monotonically increasing)
    @key
    value: int64,

    /// Hash of world state at this tick
    state_hash: bytes?,

    /// Number of entities active at this tick
    entity_count: int32 = 0,

    /// Whether this tick has been verified deterministic
    verified: bool = false
}
```

### Generated C# Code

```csharp
// <auto-generated>
// Generated by Forge from flux.simulation
// Do not edit directly.
// </auto-generated>

#nullable enable

using System;
using System.Runtime.InteropServices;

namespace Flux.Generated;

/// <summary>
/// Represents a discrete unit of simulation time
/// </summary>
[StructLayout(LayoutKind.Sequential)]
public struct Tick : IEquatable<Tick>
{
    /// <summary>
    /// Tick number (monotonically increasing)
    /// </summary>
    public long Value;

    /// <summary>
    /// Hash of world state at this tick
    /// </summary>
    public byte[]? StateHash;

    /// <summary>
    /// Number of entities active at this tick
    /// </summary>
    public int EntityCount;

    /// <summary>
    /// Whether this tick has been verified deterministic
    /// </summary>
    public bool Verified;

    // IEquatable<Tick> implementation
    public bool Equals(Tick other) { /* ... */ }
    public override bool Equals(object? obj) { /* ... */ }
    public override int GetHashCode() { /* ... */ }

    // Serialization methods
    public void Pack(ref MessagePackWriter writer) { /* ... */ }
    public static Tick Unpack(ref MessagePackReader reader) { /* ... */ }
}
```

### What Gets Generated

- **Struct definition** with correct layout
- **XML documentation** from schema comments
- **IEquatable implementation** for value semantics
- **Serialization methods** (Pack/Unpack)
- **Default values** applied in constructors
- **Validation** (if constraints specified)

---

## Axion vs Alternatives

### vs Protobuf

| Feature | Protobuf | Axion |
|---------|----------|-------|
| Human-readable schema | Yes | Yes |
| Human-readable data | No (binary only) | Yes (.axiomdata format) |
| Comments in data | N/A | Yes |
| Annotations | Limited | Extensive (`@encrypted`, `@indexed`, etc.) |
| Native encryption | No | Yes (field-level) |
| Validation rules | Limited | Extensive (`@range`, `@pattern`) |
| Cross-language | Yes | Yes |
| Determinism | Mostly | Guaranteed |
| Wire format efficiency | Excellent | Excellent |

**Key difference:** Protobuf is schema → binary. Axion is schema → human-readable data → binary.

### vs JSON Schema

| Feature | JSON Schema | Axion |
|---------|-------------|-------|
| Schema language | JSON (verbose) | Axion (concise) |
| Comments | No | Yes |
| Code generation | Third-party tools | Built-in (Fusion) |
| Binary format | No | Yes (90%+ compression) |
| Determinism | No (key order) | Yes (bit-perfect) |
| Encryption | No | Native |
| Type safety | Validation only | Compile-time types |

**Key difference:** JSON Schema validates existing JSON. Axion generates typed code and validates.

### vs TypeScript Interfaces

| Feature | TypeScript | Axion |
|---------|------------|-------|
| Cross-language | No (TS only) | Yes (C#, TS, Rust, etc.) |
| Runtime validation | Requires Zod/io-ts | Built-in |
| Binary serialization | Manual | Automatic |
| Determinism | No | Yes |
| Encryption | Manual | Declarative (`@encrypted`) |
| Server-side types | Duplicate in C# | Single source |

**Key difference:** TypeScript interfaces are frontend-only. Axion generates for all languages.

### vs GraphQL Schema

| Feature | GraphQL | Axion |
|---------|---------|-------|
| Purpose | API queries | Data contracts |
| Binary format | No | Yes |
| Determinism | No | Yes |
| Encryption | No | Native |
| Storage format | Separate concern | Integrated |
| Time-travel queries | No | Via Chronicle |

**Key difference:** GraphQL defines API shape. Axion defines data contracts + storage + encryption.

---

## Real-World Schema: Lattice Storage

```axion
namespace lattice.storage

/// Available storage engine backends
enum StorageEngine {
    Lmdb = 0,      // Lightning Memory-Mapped Database (default)
    Sqlite = 1,    // SQLite for compatibility
    RocksDb = 2,   // RocksDB for high-throughput
    Memory = 3     // In-memory for testing
}

/// Transaction isolation levels
enum IsolationLevel {
    ReadUncommitted = 0,
    ReadCommitted = 1,
    RepeatableRead = 2,
    Serializable = 3,
    Snapshot = 4   // MVCC
}

/// Represents a database transaction
entity Transaction {
    /// Unique transaction identifier
    @key
    id: uuid,

    /// Transaction state
    state: TransactionState,

    /// Isolation level for this transaction
    isolation: IsolationLevel = Snapshot,

    /// Tick at which transaction started
    start_tick: int64,

    /// Tick at which transaction committed (if applicable)
    commit_tick: int64?,

    /// Read set version for MVCC
    read_version: int64,

    /// Number of operations in this transaction
    operation_count: int32 = 0,

    /// Whether this is a read-only transaction
    read_only: bool = false
}

/// Runtime storage configuration
config StorageConfig {
    /// Storage engine to use
    engine: StorageEngine = Lmdb,

    /// Maximum database size in bytes (0 = unlimited)
    max_size_bytes: int64 = 0,

    /// Page size in bytes (4096-65536)
    page_size: int32 = 4096,

    /// Enable memory-mapped I/O
    mmap: bool = true,

    /// Sync mode for durability
    sync_mode: SyncMode = Normal,

    /// Enable write-ahead logging
    wal_enabled: bool = true
}
```

### Demonstrates

- **Enums** for constrained choices
- **Entities** for mutable data
- **Config** for runtime settings
- **Default values** for sensible behavior
- **Optional fields** (`commit_tick: int64?`)
- **Documentation** as schema comments

---

## Performance: Binary Format

### Size Comparison

Example: User record with 8 fields (id, email, name, age, role, settings, created, updated)

```
┌──────────────────────┬──────────┬─────────────┐
│ Format               │ Size     │ vs JSON     │
├──────────────────────┼──────────┼─────────────┤
│ JSON (formatted)     │ 412 B    │ Baseline    │
│ JSON (minified)      │ 268 B    │ -35%        │
│ MessagePack          │ 180 B    │ -56%        │
│ Protobuf             │  95 B    │ -77%        │
│ Axion Binary (.axb)  │  45 B    │ -89%        │
└──────────────────────┴──────────┴─────────────┘
```

### Why So Compact?

1. **No field names** (schema defines order)
2. **Varint encoding** (28 = 1 byte, not "28" = 2 bytes)
3. **Enum as ordinal** (1 byte, not "Admin" = 5 bytes)
4. **Bool as bit** (not "true" = 4 bytes)
5. **Timestamp as tick** (8 bytes, not ISO string = 24 bytes)
6. **Schema hash** validates structure

### Parse Speed

From actual benchmarks:

```
┌──────────────────────┬──────────────┬───────────┐
│ Operation            │ Speed        │ vs JSON   │
├──────────────────────┼──────────────┼───────────┤
│ Decode (cold)        │ 5-20x faster │ 5-20x     │
│ Decode (hot, cached) │ ~7μs / 1M    │ 100x+     │
│ Encode               │ 2-10x faster │ 2-10x     │
└──────────────────────┴──────────────┴───────────┘
```

---

## Annotations Reference

### Field Annotations

| Annotation | Purpose | Example |
|------------|---------|---------|
| `@key` | Primary key | `id: uuid @key` |
| `@indexed` | Create index | `username: string @indexed` |
| `@encrypted(group)` | Field-level encryption | `ssn: string @encrypted(group: "pii")` |
| `@range(min, max)` | Numeric constraint | `age: int32 @range(0, 150)` |
| `@pattern(regex)` | String validation | `email: string @pattern("^.*@.*$")` |
| `@deprecated` | Mark for removal | `old_field: int32 @deprecated` |
| `@default(value)` | Default value | `role: Role @default(User)` |

### Type Annotations

| Annotation | Purpose | Example |
|------------|---------|---------|
| `@version(M, m, p)` | Schema version | `@version(1, 0, 0)` |
| `namespace` | Package namespace | `namespace flux.simulation` |

### Implementation Notes

- **Multiple annotations** allowed per field
- **Annotations generate code** (validation, encryption, indexing)
- **Custom annotations** via plugin system
- **Domain-specific annotations** (gaming, finance, IoT packs)

---

## Advantages

### 1. Single Source of Truth

**Before Axion:**
```
user.cs (C#)         ≠  user.ts (TypeScript)  ≠  users.sql (Database)
↓                        ↓                        ↓
Runtime errors           Type mismatches          Schema drift
```

**With Axion:**
```
user.axion (SINGLE SOURCE)
    ↓
    ├→ user.cs (generated)
    ├→ user.ts (generated)
    └→ users.sql (generated, optional)

Result: Guaranteed consistency
```

### 2. Type Safety Across Languages

- Schema defines types once
- Generated code is strongly typed in target language
- Compile-time errors instead of runtime bugs
- Refactoring is safe (rename in schema, regenerate)

### 3. Automatic Serialization

- No hand-written serialization code
- Zero-allocation in hot paths (C#)
- Deterministic byte output (same input = same bytes)
- Versioning support built-in

### 4. Schema Evolution

- Add optional fields (backward compatible)
- Deprecate fields (`@deprecated`)
- Version tracking (`@version`)
- Migration tools (planned)

### 5. First-Class Documentation

- Comments in schema become XML docs (C#), JSDoc (TS)
- Single place to document data structures
- Always in sync with code

---

## Disadvantages

### 1. Additional Build Step

- Requires code generation before compilation
- Need to regenerate when schema changes
- Tool dependency (orix CLI)

**Mitigation:** IDE integration, watch mode, CI/CD automation

### 2. Learning New Syntax

- Developers must learn `.axion` syntax
- Not as ubiquitous as JSON or Protobuf

**Mitigation:** Similar to TypeScript/Rust, excellent error messages, LSP support

### 3. Generated Code Not Always Idiomatic

- Generated C# may not match hand-written style
- Limited customization without modifying generator

**Mitigation:** Plugin system for domain-specific code generation

### 4. Orix-Specific (Currently)

- Not industry-standard (yet)
- Limited third-party tooling
- Smaller ecosystem than Protobuf

**Mitigation:** Open-source, extensible, interoperates with JSON/Protobuf

---

## Workflow: Schema-First Development

### Step 1: Define Schema

```axion
// schemas/game/player.axion
namespace game.player

entity Player {
    @key
    id: uuid,

    username: string,

    @range(min: 0, max: 100)
    health: int32,

    position: vec3,
    inventory: list<Item>
}
```

### Step 2: Generate Code

```bash
# From Orix CLI
orix axion export csharp schemas/game/player.axion -o src/Game/Generated

# Or via build tool integration
```

### Step 3: Use Generated Types

```csharp
using Game.Generated;

public class PlayerService
{
    public Player CreatePlayer(string username, Vector3 position)
    {
        return new Player
        {
            Id = EntityAllocator.Allocate(),  // NOT Guid.NewGuid()
            Username = username,
            Health = 100,  // Range validation in schema
            Position = position,
            Inventory = new UnsafeList<Item>()
        };
    }
}
```

### Step 4: Serialization

```csharp
// Serialize to binary
var writer = new MessagePackWriter();
player.Pack(ref writer);
byte[] binary = writer.FlushAndGetArray();

// Deserialize from binary
var reader = new MessagePackReader(binary);
var loaded = Player.Unpack(ref reader);

// Guaranteed: loaded.Equals(player) == true
```

---

## CLI Commands

```bash
# Validate schema syntax
orix check schemas

# Generate C# code
orix axion export csharp schemas/user.axion -o src/Generated

# Generate TypeScript code
orix axion export typescript schemas/user.axion -o src/generated

# Generate all schemas in directory
orix axion export csharp schemas/**/*.axion -o src/Generated

# Get schema hash (for version tracking)
orix axion hash schemas/user.axion
# Output: sha256:a1b2c3d4e5f6...
```

---

## Integration: Lattice ORM

Axion schemas can generate Lattice ORM Active Record methods:

```csharp
// Enable Lattice code generation
var options = new EmitterOptions
{
    GenerateLattice = true  // Enables Find, Save, Delete, Where methods
};

// Generated code includes:
public partial struct User : ILatticeEntity
{
    // Active Record methods
    public static User? Find(Guid id);
    public static List<User> Where(Expression<Func<User, bool>> predicate);
    public void Save();
    public void Delete();

    // Query builder
    public static UserQuery Query();
}

// Usage
var user = User.Find(userId);
if (user != null)
{
    user.Name = "Updated Name";
    user.Save();  // Automatically persisted
}

var admins = User.Where(u => u.Role == Role.Admin);
```

---

## Future: VSCode Extension

**Planned features:**
- Syntax highlighting for `.axion` files
- IntelliSense (field completion, type suggestions)
- Real-time validation
- Go to definition
- Hover documentation
- Code actions (generate sample data, fix errors)
- Integrated code generation

**Status:** Language Server Protocol (LSP) implementation in progress

---

## Comparison Summary

| Criterion | JSON Schema | Protobuf | GraphQL | Axion |
|-----------|-------------|----------|---------|-------|
| **Schema language** | JSON (verbose) | .proto | SDL | .axion |
| **Human-readable data** | Yes (JSON) | No | N/A | Yes (.axiomdata) |
| **Code generation** | Third-party | Built-in | Third-party | Built-in (Fusion) |
| **Binary format** | No | Yes | No | Yes (90%+) |
| **Determinism** | No | Mostly | No | Guaranteed |
| **Encryption** | No | No | No | Native |
| **Cross-language** | Validation only | Yes | Yes | Yes |
| **Type safety** | Runtime only | Compile-time | Runtime | Compile-time |
| **Annotations** | Limited | Limited | Directives | Extensive |
| **Ecosystem** | Large | Massive | Large | Growing |

---

## Conclusion

**Axion is schema-first data contracts for the Orix platform.**

It solves the "schema drift" problem by providing:
- Single source of truth (`.axion` files)
- Multi-language code generation (C#, TypeScript, Rust, Python)
- Deterministic binary serialization (90%+ compression)
- Native encryption support (field-level)
- Built-in validation constraints

**When to use Axion:**
- Need type safety across multiple languages
- Require deterministic serialization
- Want declarative encryption
- Building with Lattice/Flux/Nexus
- Need schema evolution support

**When not to use Axion:**
- Simple JSON APIs (use JSON directly)
- One-off scripts (overhead not worth it)
- Ecosystem maturity critical (use Protobuf)
- No code generation in build pipeline

**Current status:** Production-ready. 13 schemas across Orix products. Performance validated in benchmarks.

---

## Example Schemas in Orix

| Product | Schema | Purpose |
|---------|--------|---------|
| **Airlock** | `secrets.axion` | Vault secrets with encryption |
| **Lattice** | `storage.axion` | Transaction and snapshot types |
| **Lattice** | `chronicle.axion` | Time-travel query types |
| **Flux** | `simulation.axion` | Tick and entity lifecycle |
| **Nexus** | `network.axion` | Network messages and sync |
| **Echo** | `replay.axion` | Recording and playback |
| **Lumen** | `observability.axion` | Logging and metrics |

## Related Documents

- [Atom Foundation](./atom.md) - Deterministic primitives Axion generates code for
- [Serialization](../reference/serialization.md) - Binary encoding of Axion types
- [Technical Deep-Dive](../reference/technical-deep-dive.md) - Implementation patterns

---

**Documentation version:** 1.0
**Last updated:** 2026-01-06
**Platform:** Orix
**Component:** Axion Schema & Code Generation
