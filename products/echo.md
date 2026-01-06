# Echo - Deterministic Replay

**Echo** is Orix's deterministic replay and time-travel debugging system. It records simulation traces and enables perfect playback, debugging, and divergence analysis.

---

## 1. Why Echo?

Traditional recording approaches fall short for deterministic simulations:

### The Problems

| Approach | Problem |
|----------|---------|
| **Video Recording** | Massive files (GB per minute), loses interactivity, no state inspection |
| **Save States** | Entire state snapshots are large, can't seek efficiently |
| **Manual Logging** | Incomplete, ad-hoc, hard to reproduce bugs |
| **Live Debugging** | Race conditions make bugs unreproducible |

### Real-World Needs

- **Bug Reproduction**: Player reports desync, need exact conditions
- **Spectating**: Watch matches without streaming full state
- **Debugging**: Step through simulation tick-by-tick
- **Testing**: Verify determinism across platforms
- **Highlights**: Extract key moments from sessions
- **Analytics**: Analyze player behavior patterns

---

## 2. How Echo Solves It

Echo leverages Orix's determinism guarantee:

```
SAME SEED + SAME INPUTS = IDENTICAL STATE
```

### Core Insight

Because simulations are deterministic, you don't need to record every state change. Record:

1. **Initial seed** - Starting random state
2. **Player inputs** - Commands issued each tick
3. **Periodic snapshots** - Full state every N ticks (for fast seeking)

The simulation will reproduce identical results.

### Technical Approach

```
┌────────────────────────────────────────────────────────────┐
│                    Echo Architecture                        │
│                                                             │
│  Record:    Seed + Inputs → Tiny trace file (1KB/min)     │
│  Replay:    Load seed → Apply inputs → Identical state     │
│  Seek:      Load snapshot → Replay from there              │
│  Verify:    Compare state hashes → Detect divergence       │
└────────────────────────────────────────────────────────────┘
```

---

## 3. What Echo Provides

### 3.1 Recording Types

```csharp
// Recording - A recorded simulation session
public struct Recording
{
    public Guid Id;              // Unique identifier
    public RecordingState State; // Stopped/Recording/Paused/Finalizing
    public long StartTick;       // When recording started
    public long? EndTick;        // When recording ended (null if ongoing)
    public long Seed;            // Initial random seed
    public long FrameCount;      // Total frames recorded
    public long SizeBytes;       // Recording size
    public DateTimeOffset CreatedAt;
    public string? Description;
    public int FormatVersion;    // Versioning for compatibility
}

// RecordingFrame - A single tick's data
public struct RecordingFrame
{
    public Guid Id;
    public Guid RecordingId;     // Which recording this belongs to
    public long Tick;            // Tick number
    public byte[] Inputs;        // Serialized player inputs
    public byte[]? StateHash;    // Hash for verification (optional)
    public byte[]? Snapshot;     // Full state snapshot (if keyframe)
}
```

### 3.2 Recording States

```csharp
public enum RecordingState
{
    Stopped = 0,     // Not recording
    Recording = 1,   // Actively recording
    Paused = 2,      // Recording paused
    Finalizing = 3   // Writing final data
}
```

### 3.3 Playback Types

```csharp
// PlaybackSession - Controls replay
public struct PlaybackSession
{
    public Guid Id;
    public Guid RecordingId;     // Which recording to play
    public PlaybackState State;  // Stopped/Playing/Paused/Seeking
    public long CurrentTick;     // Current playback position
    public PlaybackSpeed Speed;  // Playback speed multiplier
    public bool Looping;         // Loop when reaching end
    public bool Verify;          // Check state hashes during playback
}
```

### 3.4 Playback States

```csharp
public enum PlaybackState
{
    Stopped = 0,  // Not playing
    Playing = 1,  // Actively playing
    Paused = 2,   // Playback paused
    Seeking = 3   // Jumping to different tick
}
```

### 3.5 Playback Speeds

```csharp
public enum PlaybackSpeed
{
    Quarter = 0,     // 0.25x (slow motion)
    Half = 1,        // 0.5x (slow)
    Normal = 2,      // 1.0x (real-time)
    Double = 3,      // 2.0x (fast)
    Quad = 4,        // 4.0x (very fast)
    Unlimited = 5    // As fast as possible
}
```

### 3.6 Configuration

```csharp
public struct EchoConfig
{
    public int KeyframeInterval;    // Ticks between snapshots (1-1000)
    public bool Compression;         // Enable compression
    public long MaxSizeBytes;        // Size limit (0 = unlimited)
    public int AutoSaveInterval;     // Auto-save every N seconds (0 = disabled)
    public string StoragePath;       // Where to store recordings

    public static EchoConfig Default => new()
    {
        KeyframeInterval = 60,       // 1 snapshot per second at 60Hz
        Compression = true,
        MaxSizeBytes = 0,
        AutoSaveInterval = 0,
        StoragePath = "./recordings"
    };
}
```

---

## 4. Recording Flow

### 4.1 Starting a Recording

```csharp
// Create recording
var recording = new Recording
{
    Id = Guid.NewGuid(),
    State = RecordingState.Recording,
    StartTick = 0,
    EndTick = null,              // Still recording
    Seed = 42,                   // For determinism
    FrameCount = 0,
    SizeBytes = 0,
    CreatedAt = DateTimeOffset.UtcNow,
    Description = "Player vs AI match",
    FormatVersion = 1
};
```

### 4.2 Recording Each Tick

```csharp
// During simulation, record each tick
for (long tick = 0; tick < 10000; tick++)
{
    // Collect player inputs this tick
    var inputs = GetPlayerInputs(tick);

    // Create frame
    var frame = new RecordingFrame
    {
        Id = Guid.NewGuid(),
        RecordingId = recording.Id,
        Tick = tick,
        Inputs = SerializeInputs(inputs),
        StateHash = ComputeStateHash(),  // For verification
        Snapshot = null                   // Not a keyframe
    };

    // Every 60 ticks, create snapshot (keyframe)
    if (tick % 60 == 0)
    {
        frame.Snapshot = SerializeFullState();
    }

    SaveFrame(frame);
    recording.FrameCount++;
}
```

### 4.3 Finalizing Recording

```csharp
// Stop recording
recording.State = RecordingState.Finalizing;
recording.EndTick = currentTick;
recording.SizeBytes = CalculateTotalSize();

// Write final metadata
recording.State = RecordingState.Stopped;
SaveRecording(recording);
```

---

## 5. Playback Flow

### 5.1 Loading a Recording

```csharp
// Load recording
var recording = LoadRecording(recordingId);

// Create playback session
var session = new PlaybackSession
{
    Id = Guid.NewGuid(),
    RecordingId = recording.Id,
    State = PlaybackState.Stopped,
    CurrentTick = recording.StartTick,
    Speed = PlaybackSpeed.Normal,
    Looping = false,
    Verify = true                    // Check hashes
};
```

### 5.2 Playing Back

```csharp
// Initialize simulation with same seed
var simulation = new Simulation(recording.Seed);

// Start playback
session.State = PlaybackState.Playing;

while (session.CurrentTick <= recording.EndTick)
{
    // Load frame
    var frame = LoadFrame(recording.Id, session.CurrentTick);

    // Apply inputs
    var inputs = DeserializeInputs(frame.Inputs);
    simulation.QueueInputs(inputs);

    // Tick simulation
    simulation.Tick();

    // Verify if enabled
    if (session.Verify && frame.StateHash != null)
    {
        var actualHash = ComputeStateHash(simulation);
        if (!actualHash.SequenceEqual(frame.StateHash))
        {
            throw new Exception($"Replay diverged at tick {session.CurrentTick}!");
        }
    }

    session.CurrentTick++;

    // Render or process
    RenderFrame(simulation);

    // Respect playback speed
    ApplySpeedDelay(session.Speed);
}

session.State = PlaybackState.Stopped;
```

### 5.3 Seeking (Time-Travel)

```csharp
// Jump to specific tick
public void SeekTo(long targetTick)
{
    session.State = PlaybackState.Seeking;

    // Find nearest keyframe before target
    var keyframe = FindNearestKeyframe(targetTick);

    if (keyframe != null)
    {
        // Load snapshot
        var state = DeserializeSnapshot(keyframe.Snapshot);
        simulation.RestoreState(state);
        session.CurrentTick = keyframe.Tick;
    }
    else
    {
        // No snapshot, restart from beginning
        simulation = new Simulation(recording.Seed);
        session.CurrentTick = recording.StartTick;
    }

    // Replay from keyframe to target
    while (session.CurrentTick < targetTick)
    {
        var frame = LoadFrame(recording.Id, session.CurrentTick);
        var inputs = DeserializeInputs(frame.Inputs);
        simulation.QueueInputs(inputs);
        simulation.Tick();
        session.CurrentTick++;
    }

    session.State = PlaybackState.Paused;
}
```

---

## 6. Example: Bug Reproduction

Let me show you an example.

### The Bug Report

Player reports: "Desync occurred 5 minutes into match, turn 18000."

### Recording

```csharp
// Player's game automatically recorded
var recording = new Recording
{
    Id = Guid.NewGuid(),
    Seed = 987654321,
    StartTick = 0,
    Description = "Match where desync occurred"
};

// Every tick was recorded with state hashes
for (long tick = 0; tick < 18000; tick++)
{
    var frame = new RecordingFrame
    {
        Tick = tick,
        Inputs = GetInputs(tick),
        StateHash = ComputeHash(tick)  // Verification hash
    };
    SaveFrame(frame);
}
```

### Debugging

```csharp
// Load player's recording
var recording = LoadRecording(bugReportId);

// Create debug session
var session = new PlaybackSession
{
    RecordingId = recording.Id,
    Verify = true,           // Check hashes
    Speed = PlaybackSpeed.Unlimited  // Go fast
};

// Run through recording
var simulation = new Simulation(recording.Seed);
session.State = PlaybackState.Playing;

for (long tick = 0; tick < 18000; tick++)
{
    var frame = LoadFrame(recording.Id, tick);
    simulation.QueueInputs(DeserializeInputs(frame.Inputs));
    simulation.Tick();

    // Verify hash
    var actualHash = ComputeHash(simulation);
    if (!actualHash.SequenceEqual(frame.StateHash))
    {
        // Found the divergence!
        Console.WriteLine($"Divergence at tick {tick}!");

        // Seek back and step through slowly
        SeekTo(tick - 100);
        session.Speed = PlaybackSpeed.Quarter;

        // Step-by-step debugging
        for (long t = tick - 100; t <= tick; t++)
        {
            DumpState(simulation);  // Log everything
            simulation.Tick();
        }

        break;
    }
}
```

This example doesn't compile - it's pseudocode showing the concept.

Here's why this works:

1. **Exact Reproduction**: Same seed + same inputs = identical bug
2. **Efficient**: Only inputs recorded, tiny file
3. **Seekable**: Jump to moments before the bug
4. **Verifiable**: State hashes prove correct replay

---

## 7. Advantages

### 7.1 File Size

| Recording Type | Size per Minute |
|----------------|-----------------|
| **Video (1080p60)** | ~100 MB |
| **Full State Snapshots** | ~10 MB |
| **Echo (inputs only)** | ~0.1 MB (1000x smaller!) |

### 7.2 Functionality

| Feature | Echo | Video | Save States |
|---------|------|-------|-------------|
| **Interactivity** | Full simulation access | Watch only | Full access |
| **Seeking** | Fast (keyframes) | Slow | Not supported |
| **State Inspection** | Yes | No | Yes |
| **Time-Travel** | Yes | No | Limited |
| **File Size** | Tiny | Huge | Large |
| **Bug Reproduction** | Perfect | No state | Snapshots only |

### 7.3 Performance

| Operation | Performance |
|-----------|-------------|
| **Recording overhead** | ~1% CPU (input serialization only) |
| **Storage** | ~1 KB per minute of gameplay |
| **Seek time** | O(1) to keyframe + resim |
| **Verification** | Optional, ~5% overhead |

---

## 8. Disadvantages

### 8.1 Requires Determinism

Echo ONLY works with deterministic simulations:

- ❌ Non-deterministic physics engines
- ❌ Random without seed
- ❌ Floating-point arithmetic
- ❌ Wall-clock dependencies

Orix guarantees determinism, so this isn't a problem.

### 8.2 Simulation Changes Break Replays

If you change simulation logic:

```csharp
// Version 1.0
player.Speed = 10;

// Version 1.1 - BREAKS OLD REPLAYS
player.Speed = 15;  // Different behavior!
```

**Solutions**:
- Version recordings with simulation version
- Keep old simulation code paths
- Convert recordings to new format

### 8.3 Initial Load Requires Resimulation

Seeking to tick 5000 requires:
1. Load nearest snapshot (say tick 4800)
2. Resimulate 200 ticks to reach 5000

This is fast (microseconds per tick) but not instant.

---

## 9. Comparison to Alternatives

### 9.1 vs Video Recording

| Aspect | Echo | Video |
|--------|------|-------|
| **Size** | 1 KB/min | 100 MB/min |
| **Interactive** | Yes (full state) | No (watch only) |
| **Debugging** | Step through code | Can't see internals |
| **Seeking** | Fast | Slow |

### 9.2 vs Save States

| Aspect | Echo | Save States |
|--------|------|-------------|
| **Storage** | Inputs + snapshots | Full state |
| **Seeking** | Keyframes + resim | Load snapshot |
| **History** | Complete timeline | Discrete points |
| **Size** | Small | Large |

### 9.3 vs Event Sourcing

| Aspect | Echo | Event Sourcing |
|--------|------|----------------|
| **Domain** | Simulation-specific | General purpose |
| **Snapshots** | Periodic keyframes | Optional |
| **Verification** | State hashes | Business logic |
| **Time-travel** | Tick-level | Event-level |

Echo is specialized event sourcing for deterministic simulations.

---

## 10. Use Cases

### 10.1 Competitive Gaming

```csharp
// Record tournament matches
var tournament = new Recording
{
    Description = "Championship Finals",
    Seed = GenerateTournamentSeed()
};

// Stream highlights later
foreach (var highlight in tournament.Highlights)
{
    SeekTo(highlight.Tick);
    PlayFor(highlight.Duration);
}
```

### 10.2 AI Training

```csharp
// Record human gameplay
var sessions = RecordHumanPlayers(1000);

// Replay for AI to learn from
foreach (var session in sessions)
{
    var replay = new PlaybackSession
    {
        RecordingId = session.Id,
        Speed = PlaybackSpeed.Unlimited,
        Verify = false  // Skip verification for speed
    };

    aiTrainer.Learn(replay);
}
```

### 10.3 Testing

```csharp
// Verify determinism across platforms
var windowsTrace = RecordOnWindows(seed: 42);
var linuxTrace = RecordOnLinux(seed: 42);
var macTrace = RecordOnMac(seed: 42);

// Compare state hashes
VerifyIdentical(windowsTrace, linuxTrace);
VerifyIdentical(windowsTrace, macTrace);
```

### 10.4 Customer Support

```csharp
// Player files bug report with attached recording
var bugReport = LoadRecording("bug_12345.otr");

// Support engineer reproduces
var session = new PlaybackSession
{
    RecordingId = bugReport.Id,
    Verify = true
};

// Find exact moment of issue
SeekTo(bugReport.IssueTimestamp);
InspectState();  // See exactly what player saw
```

---

## 11. Chronicle Integration

Echo is for **in-memory replay**. For **persistent time-travel database**, use Lattice.Chronicle:

| System | Purpose | Scope |
|--------|---------|-------|
| **Echo** | Recording & playback | Session/match level |
| **Chronicle** | Time-travel queries | Database-wide |

```csharp
// Echo: Replay a game session
var recording = LoadRecording("match.otr");
ReplaySession(recording);

// Chronicle: Query database history
var asOfTick = 5000;
var player = db.Query<Player>()
    .AsOf(asOfTick)
    .Where(p => p.Id == playerId)
    .FirstOrDefault();  // State at tick 5000
```

---

## 12. Key Takeaways

1. **Tiny Files**: 1000x smaller than video, inputs-only recording
2. **Perfect Reproduction**: Determinism guarantees identical replay
3. **Time-Travel**: Seek to any tick via keyframes
4. **Verification**: State hashes detect divergence
5. **Debugging**: Step through simulation tick-by-tick
6. **Specialized**: Built for deterministic simulations (Orix)

---

## Related Documents

- [Chronicle Time-Travel](./05-chronicle.md) - Database-level time-travel queries
- [Flux Simulation](./07-simulation-flux.md) - The deterministic runtime Echo records
- [Nexus Networking](./08-networking-nexus.md) - Multiplayer state synchronization
- [Arbiter Testing](./11-testing-arbiter.md) - Determinism verification tests

---

**Next**: [Lumen Observability](./10-observability-lumen.md) - Tick-aware logging and metrics
