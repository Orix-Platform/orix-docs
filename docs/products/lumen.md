# Lumen: Tick-Aware Observability

**Version:** 1.0.0
**Status:** IMPLEMENTED
**Layer:** Observability

Lumen is Orix's observability system designed specifically for deterministic, tick-based simulations. It provides logging, metrics, and distributed tracing with first-class support for simulation time correlation.

---

## Why Lumen?

### The Problem

Traditional observability tools face fundamental challenges when applied to deterministic simulations:

**1. Time Mismatch**
- Traditional logging uses wall-clock time (`DateTime.Now`)
- Simulations run in discrete ticks, not continuous time
- Wall-clock time tells you WHEN code executed, not WHERE in the simulation

**2. Debugging Challenges**
- "Bug happens at tick 5,432" - where are those logs?
- Replaying from a specific tick requires tick-correlated data
- Performance metrics need tick-level granularity, not wall-clock

**3. Distributed Tracing Complexity**
- Operations span multiple ticks (e.g., pathfinding takes 3 ticks)
- Parent-child relationships must preserve tick boundaries
- Traditional spans use wall-clock start/end times

**4. Determinism Violations**
- `DateTime.Now` breaks determinism
- `Guid.NewGuid()` for correlation IDs is non-deterministic
- Logging shouldn't affect simulation behavior

### The Solution

Lumen provides:

1. **Tick-Correlated Logging** - Every log entry has a tick number
2. **Deterministic by Default** - Wall-clock time is opt-in via `[Ambient]`
3. **Tick-Based Metrics** - Record values per tick, not per second
4. **Span Tracing with Ticks** - Distributed tracing that respects simulation time
5. **Pluggable Sinks** - Console, memory, file, or custom destinations

---

## How Lumen Works

### Tick-First Philosophy

Lumen inverts traditional logging:

| Aspect | Traditional | Lumen |
|--------|-------------|-------|
| Default timestamp | DateTime.Now | Tick (simulation time) |
| Wall-clock time | Built-in | Opt-in via [Ambient] |
| Determinism | Non-deterministic | Deterministic by default |
| Replay | Impossible | Perfect replay |

### Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Lumen API Layer                         │
│  Log.Info() | Log.For(category) | Log.AtTick(tick)          │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────────────────┐
│                    LogEntry (struct)                         │
│  Tick + Level + Category + Message + Payload                │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────────────────┐
│                      Sink System                             │
│  Console | Memory | File | Binary | [Ambient] Network       │
└─────────────────────────────────────────────────────────────┘
```

### Tick Correlation Mechanism

```csharp
// Simulation sets current tick
Log.SetTick(currentTick);

// All logs automatically use this tick
Log.Info("Entity spawned");  // Tagged with currentTick

// Explicit tick override
Log.AtTick(Tick.FromRaw(100)).Warn("Historical event");
```

The `Log` static class maintains a current tick value that gets embedded in every log entry. This creates perfect correlation between simulation state and log output.

### Two-Layer Design

**Runtime Layer (`Lumen` namespace)**
- `LogEntry` - Performance-optimized struct for logging
- `Log` - Static API for writing logs
- `ILumenSink` - Extensible output destinations
- Uses `Tick` for deterministic time

**Schema Layer (`Lumen.Observability.Generated` namespace)**
- Axion-generated types for storage and transport
- Includes wall-clock time for external correlation
- Serializable with `IPackable<T>` interface
- Used for Chronicle storage and network transmission

---

## What Lumen Provides

### Core Types

#### LogLevel

Six standard severity levels:

```csharp
public enum LogLevel : byte
{
    Trace = 0,   // Fine-grained debugging
    Debug = 1,   // Development diagnostics
    Info = 2,    // General information
    Warn = 3,    // Warning conditions
    Error = 4,   // Error conditions
    Fatal = 5    // Critical failures
}
```

#### LogEntry (Runtime)

Deterministic log entry for in-memory use:

```csharp
public readonly struct LogEntry
{
    public Tick Tick { get; }              // Required: simulation time
    public DateTime? WallTime { get; }     // Optional: wall-clock time
    public LogLevel Level { get; }         // Severity
    public string Category { get; }        // Source component
    public string Message { get; }         // Log message
    public ReadOnlyMemory<byte> Payload { get; } // Optional structured data
}
```

**Key Features:**
- Tick is always present (deterministic)
- WallTime is null by default (opt-in for `[Ambient]` contexts)
- Zero-allocation `ReadOnlyMemory<byte>` for structured data
- `ToString()` formats as `[T{tick}] [LEVEL] Category: Message`

#### MetricType

Four standard metric types:

```csharp
public enum MetricType
{
    Gauge = 0,       // Instantaneous value (entity count, memory usage)
    Counter = 1,     // Monotonically increasing (total events)
    Histogram = 2,   // Distribution of values (latency percentiles)
    Meter = 3        // Rate of events (events per tick)
}
```

#### MetricDefinition (Schema-Generated)

Defines what a metric measures:

```csharp
[AxionGenerated]
public struct MetricDefinition : IPackable<MetricDefinition>
{
    public Entity Id { get; set; }           // Unique identifier
    public string Name { get; set; }         // Metric name (e.g., "entity_count")
    public MetricType MetricType { get; set; } // Type of metric
    public string? Description { get; set; } // Human-readable description
    public string? Unit { get; set; }        // Unit of measurement
    public List<string>? Tags { get; set; }  // Dimension names (e.g., ["world", "region"])
}
```

#### MetricValue (Schema-Generated)

A recorded metric value at a specific tick:

```csharp
[AxionGenerated]
public struct MetricValue : IPackable<MetricValue>
{
    public Guid Id { get; set; }             // Unique value identifier
    public string MetricName { get; set; }   // Links to MetricDefinition
    public long Tick { get; set; }           // When recorded
    public long Value { get; set; }          // Q32.32 fixed-point value
    public List<string>? TagValues { get; set; } // Tag values (parallel to definition)
}
```

**Important:** `Value` is stored as Q32.32 fixed-point (`DFixed64.Raw`), not floating-point.

#### Span (Schema-Generated)

Distributed trace span with tick boundaries:

```csharp
[AxionGenerated]
public struct Span : IPackable<Span>
{
    public Guid Id { get; set; }           // Unique span ID
    public Guid TraceId { get; set; }      // Trace this span belongs to
    public Guid? ParentId { get; set; }    // Parent span (null = root)
    public string Name { get; set; }       // Operation name
    public long StartTick { get; set; }    // When operation started
    public long? EndTick { get; set; }     // When completed (null = in progress)
    public SpanStatus Status { get; set; } // Ok, Error, or Cancelled
    public string? Attributes { get; set; } // JSON key-value attributes
}

public enum SpanStatus
{
    Ok = 0,         // Completed successfully
    Error = 1,      // Failed with error
    Cancelled = 2   // Operation cancelled
}
```

### Logging API

**Basic Logging:**
```csharp
using Lumen;
using Atom.Primitives;

// Set current tick (typically done by Flux runtime)
Log.SetTick(Tick.FromRaw(100));

// Log at different levels
Log.Trace("Detailed trace information");
Log.Debug("Debugging context");
Log.Info("System started");
Log.Warn("Configuration issue detected");
Log.Error("Failed to process entity");
Log.Fatal("Unrecoverable error");
```

**Category-Scoped Logging:**
```csharp
var physics = Log.For("Physics");
var network = Log.For("Network");

physics.Info("Collision detected");
network.Debug("Packet received");
```

**Historical Logging (Specific Tick):**
```csharp
var historicalLog = Log.AtTick(Tick.FromRaw(5000));
historicalLog.Info("This log appears to be from tick 5000");
```

**With Wall-Clock Time (Ambient):**
```csharp
[Ambient(Reason = "External API requires real timestamps")]
public void ProcessExternalEvent()
{
    Log.WriteWithWallTime(LogLevel.Info, "External", "Event from payment gateway");
}
```

**Structured Logging:**
```csharp
// Serialize data to byte array
var data = new { EntityId = 42, Position = new { X = 10.5, Y = 20.3 } };
var payload = SerializeToBytes(data);

Log.Write(
    level: LogLevel.Info,
    category: "Gameplay",
    message: "Entity spawned",
    payload: payload
);
```

### Sink System

**Built-in Sinks:**

**Console Sink:**
```csharp
using Lumen.Sinks;

var consoleSink = new ConsoleSink(useColors: true);
Log.AddSink(consoleSink);

Log.SetTick(Tick.FromRaw(100));
Log.Info("Hello, Lumen!");
// Output: [T100] [INFO] App: Hello, Lumen!
```

**Color Scheme:**
- Trace: Dark Gray
- Debug: Gray
- Info: White
- Warn: Yellow
- Error: Red
- Fatal: Dark Red

**Memory Sink:**
```csharp
using Lumen.Sinks;

var memorySink = new MemorySink(maxEntries: 1000);
Log.AddSink(memorySink);

// Write some logs
Log.Info("Test message 1");
Log.Error("Test error");

// Query logs
var errors = memorySink.AtLevel(LogLevel.Error);
var physicLogs = memorySink.ForCategory("Physics");
var searchResults = memorySink.ContainingMessage("collision");

// Get all entries
IReadOnlyList<LogEntry> allLogs = memorySink.GetEntries();
```

**Query Methods:**
- `AtLevel(LogLevel)` - Filter by minimum level
- `ForCategory(string)` - Filter by category
- `ContainingMessage(string)` - Text search
- `Where(Func<LogEntry, bool>)` - Custom predicate

**Custom Sink Interface:**
```csharp
public interface ILumenSink : IDisposable
{
    string Id { get; }
    void Write(in LogEntry entry);
    void Flush();
}
```

**Ambient Sinks:**
```csharp
// Marker interface for non-deterministic sinks
public interface IAmbientSink : ILumenSink { }

// Example: Database sink
public class DatabaseSink : IAmbientSink
{
    public string Id => "database";

    [Ambient(Reason = "Database I/O")]
    public void Write(in LogEntry entry)
    {
        var wallTime = DateTime.UtcNow;
        _db.InsertLog(entry.Tick, wallTime, entry.Level, entry.Message);
    }

    public void Flush() { }
    public void Dispose() => _db.Close();
}
```

### Configuration

**LumenConfig (Schema-Generated):**
```csharp
using Lumen.Observability.Generated;

var config = new LumenConfig
{
    MinLogLevel = LogLevel.Debug,           // Minimum level to log
    TickCorrelation = true,                  // Enable tick tracking
    MetricsInterval = 60,                    // Record metrics every 60 ticks
    TracingEnabled = true,                   // Enable span tracing
    TraceSampleRate = DFixed64.One.Raw,      // 100% sampling rate
    LogBufferSize = 10000                    // Buffer 10k entries
};

// Default configuration
var config = LumenConfig.Default;
```

**Runtime Configuration:**
```csharp
// Set minimum level
Log.MinimumLevel = LogLevel.Debug;

// Set default category
Log.DefaultCategory = "Simulation";

// Manage sinks
Log.AddSink(new ConsoleSink());
Log.RemoveSink(consoleSink);
Log.ClearSinks();
Log.Flush();
```

---

## Advantages

### 1. Perfect Determinism

```csharp
// Run 1
Log.SetTick(Tick.FromRaw(0));
for (int i = 0; i < 100; i++)
{
    Log.SetTick(Tick.FromRaw((ulong)i));
    Log.Info($"Processing tick {i}");
}

// Run 2 (identical seed)
Log.SetTick(Tick.FromRaw(0));
for (int i = 0; i < 100; i++)
{
    Log.SetTick(Tick.FromRaw((ulong)i));
    Log.Info($"Processing tick {i}");
}

// Both runs produce identical tick-stamped logs
```

### 2. Tick-Level Correlation

Query logs by exact simulation time:
```csharp
var memorySink = new MemorySink();

// After simulation runs
var logsAtTick100 = memorySink.Where(e => e.Tick.Value == 100);
var logsInRange = memorySink.Where(e =>
    e.Tick.Value >= 50 && e.Tick.Value <= 150);
```

### 3. Replay-Friendly

Logs replay identically because they're tied to simulation ticks, not wall-clock time:
```csharp
// Original run
RunSimulation(seed: 42);
// Logs: [T100] Entity spawned at (10.5, 20.0)

// Replay (same seed)
RunSimulation(seed: 42);
// Logs: [T100] Entity spawned at (10.5, 20.0)  // Identical
```

### 4. Testing Integration

Works seamlessly with Arbiter:
```csharp
[ArbiterScenario(Seed = 42, Category = "Lumen.Logging")]
public void LogEntry_Creation_HasCorrectFields()
{
    var tick = Tick.FromRaw(100);
    var logEntry = new LogEntry(tick, LogLevel.Info,
        "TestComponent", "Test log message");

    Assert.Equal(100u, logEntry.Tick.Value);
    Assert.Equal("Test log message", logEntry.Message);
    Assert.False(logEntry.HasWallTime);  // No wall-clock by default
}
```

### 5. Schema Authority

All types generated from Axion schemas guarantee:
- Type safety across system boundaries
- Automatic serialization/deserialization
- Schema evolution without breaking changes
- Hash verification of generated code

### 6. Zero-Allocation Logging (Hot Path)

Struct-based design with value semantics:
```csharp
// LogEntry is a struct - no heap allocation
public readonly struct LogEntry { /* ... */ }

// Passed by reference to sinks
public void Write(in LogEntry entry);  // No copy
```

---

## Disadvantages

### 1. Tick Context Required

Lumen requires simulation to set current tick:
```csharp
// Must be called by simulation runtime
Log.SetTick(currentTick);

// Otherwise logs have stale tick
```

**Mitigation:** Simulation frameworks automatically call SetTick() each frame.

### 2. Additional Overhead

Every log entry carries tick information:
```csharp
// 64-byte minimum per entry
LogEntry: {
    Tick (8 bytes) +
    WallTime? (8 bytes nullable) +
    Level (1 byte) +
    Category (reference) +
    Message (reference) +
    Payload (ReadOnlyMemory)
}
```

**Impact:** Negligible for most use cases, but matters at extreme scale (millions of logs per second).

### 3. Storage for Tick-Level Data

- Metrics per tick can grow large
- 60 FPS = 3600 metric samples per minute per metric
- Span storage scales with operation count

**Mitigation:**
- Aggregate metrics (e.g., per 60 ticks = per second at 60 FPS)
- Use Chronicle's time-windowed storage
- Sample spans (e.g., 10% of operations)

### 4. Two-Layer Complexity

- Runtime types (`Lumen.LogEntry`) vs Generated types (`Lumen.Observability.Generated.LogEntry`)
- Must convert for storage/transmission
- Different APIs for logging vs storage

**Mitigation:**
- Runtime layer optimized for hot-path performance
- Generated layer handles serialization automatically
- Conversion is explicit and rare

### 5. Limited High-Level Metrics API

Current implementation defines metric schemas but lacks high-level runtime API:
```csharp
// Schema exists
public struct MetricValue { /* ... */ }

// But no high-level API yet for:
var counter = Metrics.Counter("entity_count");  // Not implemented
counter.Increment();
```

**Status:** Metrics are schema-complete but API-incomplete.

---

## Comparison

### vs. Serilog

| Feature | Lumen | Serilog |
|---------|-------|---------|
| **Time Model** | Tick-based | Wall-clock only |
| **Determinism** | Yes (default) | No |
| **Simulation Focus** | Yes | No |
| **Structured Logging** | Byte arrays | JSON, objects |
| **Sinks** | Custom interface | Rich ecosystem |
| **Performance** | Zero-allocation struct | Object allocations |

**Use Lumen when:** Deterministic simulation, tick correlation critical
**Use Serilog when:** Traditional applications, rich sink ecosystem needed

### vs. OpenTelemetry

| Feature | Lumen | OpenTelemetry |
|---------|-------|---------------|
| **Tracing Model** | Tick-based spans | Time-based spans |
| **Metrics** | Per-tick values | Per-second rates |
| **Timestamp** | Tick + optional wall-clock | Unix epoch |
| **Determinism** | Yes | No |
| **Exporters** | Ambient sinks | Many |
| **Standards** | Orix-specific | Industry standard |

**Use Lumen when:** Tick-native tracing, deterministic replay
**Use OpenTelemetry when:** External service integration, standard compliance

**Note:** Lumen can export to OpenTelemetry via ambient sinks.

### vs. Datadog

| Feature | Lumen | Datadog |
|---------|-------|---------|
| **Type** | Local library | SaaS monitoring |
| **Hosting** | Self-hosted | Cloud |
| **Time Model** | Tick-based | Wall-clock |
| **Real-time** | Via sinks | Built-in |
| **Cost** | Free | Subscription |
| **Dashboards** | External | Built-in |
| **Alerting** | External | Built-in |
| **Determinism** | Yes | No |
| **Tick correlation** | Built-in | No |

**Use Lumen when:** Development, testing, deterministic debugging
**Use Datadog when:** Production monitoring, alerting, dashboards

**Integration:** Use Lumen locally, export to Datadog via NetworkSink for production.

---

## Performance

### Logging Overhead (Measured)

From test scenarios:

```csharp
[ArbiterScenario(Seed = 42, Category = "Lumen.Performance")]
public void MetricAggregation_PerTick_IsEfficient()
{
    // 1000 metric values generated and aggregated
    // Observed: < 1ms total for 1000 values
    var values = new List<MetricValue>();
    var aggregated = new Dictionary<long, long>();

    for (int i = 0; i < 1000; i++)
    {
        var tick = i / 10;
        var value = new MetricValue
        {
            Id = Guid.NewGuid(),
            MetricName = "processing_time",
            Tick = tick,
            Value = 100 + (i % 50),
            TagValues = null
        };
        values.Add(value);

        if (!aggregated.ContainsKey(tick))
            aggregated[tick] = 0;
        aggregated[tick] += value.Value;
    }

    // 100 ticks aggregated, all values > 0
    Assert.Equal(100, aggregated.Count);
}
```

### Estimated Overhead

Based on struct layout and sink implementations:

| Operation | Estimate | Notes |
|-----------|----------|-------|
| LogEntry creation | ~50 ns | Struct initialization |
| ConsoleSink write | ~1-5 μs | Console I/O bottleneck |
| MemorySink write | ~100 ns | ConcurrentQueue enqueue |
| Tick correlation | ~10 ns | Simple property access |
| Category scoping | ~50 ns | String comparison |

### Memory Profile

```csharp
// LogEntry memory layout
struct LogEntry
{
    Tick Tick;              // 8 bytes (ulong wrapper)
    DateTime? WallTime;     // 8 bytes (nullable struct)
    LogLevel Level;         // 1 byte (enum)
    string Category;        // 8 bytes (reference)
    string Message;         // 8 bytes (reference)
    ReadOnlyMemory<byte>;   // 16 bytes (struct)
}
// Total: ~49 bytes + string heap allocations
```

### Best Practices

**1. Set Tick Early:**
```csharp
public void OnTickStart(Tick tick)
{
    Log.SetTick(tick);  // First thing in tick processing
}
```

**2. Use Category Loggers:**
```csharp
// Create once, reuse
private static readonly CategoryLogger _log = Log.For("Physics");

public void Update()
{
    _log.Debug("Processing collisions");
}
```

**3. Avoid Logging in Hot Loops:**
```csharp
// BAD: Log per entity
foreach (var entity in entities)
{
    Log.Trace($"Processing entity {entity.Id}");
}

// GOOD: Log summary
Log.Debug($"Processed {entities.Count} entities in {elapsed} ticks");
```

**4. Configure Minimum Level for Production:**
```csharp
#if DEBUG
    Log.MinimumLevel = LogLevel.Debug;
#else
    Log.MinimumLevel = LogLevel.Info;
#endif
```

---

## Real-World Usage Examples

### Example 1: Game Simulation

```csharp
public class CombatSystem
{
    private readonly CategoryLogger _log = Log.For("Combat");

    public void Execute(Tick currentTick)
    {
        Log.SetTick(currentTick);

        foreach (var (attacker, target, damage) in GetAttacks())
        {
            _log.Info($"Entity {attacker} attacks {target} for {damage}");

            if (WillKill(target, damage))
            {
                _log.Warn($"Entity {target} killed by {attacker}");
            }
        }
    }
}
```

### Example 2: Testing with MemorySink

```csharp
[ArbiterScenario(Seed = 42)]
public void Combat_LogsCorrectly()
{
    var sink = new MemorySink();
    Log.AddSink(sink);

    var combat = new CombatSystem();
    combat.Execute(Tick.FromRaw(100));

    // Verify logs
    var logs = sink.GetEntries();
    Assert.True(logs.Any(l => l.Message.Contains("attacks")));
    Assert.Equal(100u, logs.First().Tick.Value);
}
```

### Example 3: Metrics Collection

```csharp
// Using schema-defined MetricValue
var metrics = new List<MetricValue>();

for (long tick = 0; tick < 1000; tick++)
{
    metrics.Add(new MetricValue
    {
        Id = Guid.NewGuid(),
        MetricName = "entity_count",
        Tick = tick,
        Value = GetEntityCount(),
        TagValues = new List<string> { "world_1" }
    });
}

// Aggregate by tick range
var avgEntityCount = metrics
    .Where(m => m.Tick >= 100 && m.Tick <= 200)
    .Average(m => m.Value);
```

### Example 4: Distributed Tracing

```csharp
var traceId = Guid.NewGuid();

var rootSpan = new Span
{
    Id = Guid.NewGuid(),
    TraceId = traceId,
    ParentId = null,
    Name = "GameLoop",
    StartTick = 0,
    EndTick = 100,
    Status = SpanStatus.Ok,
    Attributes = "{\"player_count\": 4}"
};

var childSpan = new Span
{
    Id = Guid.NewGuid(),
    TraceId = traceId,
    ParentId = rootSpan.Id,
    Name = "PhysicsUpdate",
    StartTick = 10,
    EndTick = 15,
    Status = SpanStatus.Ok
};

// Verify span hierarchy
Assert.Equal(rootSpan.TraceId, childSpan.TraceId);
Assert.Equal(rootSpan.Id, childSpan.ParentId);
Assert.True(childSpan.StartTick >= rootSpan.StartTick);
Assert.True(childSpan.EndTick <= rootSpan.EndTick);
```

---

## Related Documents

- [Flux Simulation](./flux.md) - The tick-based runtime Lumen observes
- [Echo Replay](./echo.md) - Replay system that benefits from tick-correlated logs
- [Arbiter Testing](./arbiter.md) - Testing with MemorySink for log verification
- [Axion Schema](./axion.md) - Schema definitions for observability types

---

## Current Implementation Status

### Completed
- LogEntry struct (deterministic, tick-stamped)
- LogLevel enum with 6 levels
- Log static API (Info, Debug, Error, etc.)
- Category-scoped logging (Log.For(category))
- Tick-scoped logging (Log.AtTick(tick))
- ConsoleSink with color support
- MemorySink with query methods
- Axion schema for observability types
- Generated types for LogEntry, MetricDefinition, MetricValue, Span
- Arbiter integration tests
- LumenConfig with defaults

### In Progress
- File sink implementation
- Binary sink implementation
- Network sink (ambient)

### Planned
- High-level Metrics API (Counter, Gauge, Histogram)
- High-level Tracing API (Trace.StartSpan)
- OpenTelemetry exporter sink
- CLI commands (orix lumen)

---

## Summary

Lumen solves the fundamental problem of observability in deterministic systems: **wall-clock time is non-deterministic**.

By making simulation tick the default timestamp and wall-clock time opt-in, Lumen enables:
- Perfect replay of logged events
- Exact tick-level correlation
- Deterministic testing with Arbiter
- Schema-driven type safety

The tradeoff is that Lumen requires tick context and is not a complete observability stack. It excels at internal simulation debugging and testing, but external dashboards and alerting require integration via sinks.

**Use Lumen when:** Building deterministic simulations, tick-based systems, or replay-enabled games.
**Integrate with:** Serilog/OpenTelemetry for rich ecosystem, Datadog/Grafana for production monitoring.

**Key Insight:** By making simulation time (ticks) the primary timestamp, Lumen enables debugging, profiling, and tracing that aligns perfectly with deterministic replay and time-travel.
