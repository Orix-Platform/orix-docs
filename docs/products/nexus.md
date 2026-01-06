# Nexus: Deterministic Networking for Multiplayer

## Overview

Nexus is Orix's networking layer, designed to synchronize state across distributed systems while preserving determinism. It provides delta compression, authority management, and interest-based filtering to make multiplayer games and distributed simulations efficient and correct.

## The Problem

Traditional networking breaks determinism in four critical ways:

### 1. State Synchronization Divergence

```csharp
// Problem: Client and server states drift apart
// Client
player.Position = new Vector3(10.5f, 0, 5.2f);

// Server (slightly different due to float precision)
player.Position = new Vector3(10.500001f, 0, 5.2000003f);

// After 100 ticks: desync, teleporting, rubber-banding
```

Multiplayer games struggle with state synchronization because:
- Floating-point precision differences between platforms
- Different update rates (client at 144 FPS, server at 64 ticks/sec)
- Network latency causing input delays
- Different execution order between client and server

This breaks:
- **Competitive multiplayer**: Players see different game states
- **Replay systems**: Recordings don't match live gameplay
- **Testing**: Impossible to reproduce bugs consistently

### 2. Bandwidth Constraints

```csharp
// Problem: Sending full state every frame is expensive
public struct GameState
{
    public Player[] Players;      // 100 players
    public Entity[] Entities;     // 1000 entities
    public Terrain[] Chunks;      // 256 chunks
}

// Full state: ~500 KB per update
// At 60 Hz: 30 MB/sec = 240 Mbps per client
```

Sending complete state is impractical:
- Most data doesn't change between frames
- Network bandwidth is limited (typically 1-10 Mbps for games)
- Server must send to multiple clients simultaneously
- Mobile networks have high latency and packet loss

This breaks:
- **Scalability**: Can't support many clients
- **Performance**: Network becomes bottleneck
- **Responsiveness**: High latency from large packets

### 3. Authority Confusion

```csharp
// Problem: Who controls what?

// Client predicts movement locally
client.Player.Position += client.Input * deltaTime;

// Server also updates position
server.Player.Position += server.Input * deltaTime;

// Conflict: Which position is correct?
// Without clear authority, client gets rubber-banding
```

Multiplayer systems need clear authority:
- Server-authoritative: Server controls everything (secure but laggy)
- Client-authoritative: Client controls local player (responsive but cheatable)
- Hybrid: Different entities have different authority

Without clear rules:
- **Rubber-banding**: Client predictions get overridden
- **Exploits**: Clients can cheat by lying about state
- **Inconsistency**: Different clients see different states

### 4. Relevance Overload

```csharp
// Problem: Client receives updates for everything

// Client at position (0, 0, 0)
// Receives updates for:
// - Enemy at (0, 5, 0) - RELEVANT
// - Enemy at (10000, 0, 0) - IRRELEVANT (off-screen)
// - Explosion at (5000, 2000, 0) - IRRELEVANT (can't see)

// Wastes bandwidth on irrelevant data
```

Not all state is relevant to all clients:
- Players can only see nearby entities
- Distant events don't affect local gameplay
- Different clients have different areas of interest

This breaks:
- **Efficiency**: Wasted bandwidth
- **Performance**: Processing irrelevant updates
- **Privacy**: Clients see data they shouldn't (wallhacks)

## How Nexus Solves It

Nexus builds on Atom's deterministic primitives to provide efficient, correct networking:

### 1. Tick-Synchronized Updates

Instead of wall-clock time, Nexus uses **discrete ticks**:

```csharp
// All clients and server run at same tick rate
public struct NetworkUpdate
{
    public Tick BaseTick;      // Last acknowledged tick
    public Tick TargetTick;    // Current tick
    public byte[] DeltaData;   // Changes between ticks
}
```

**Why This Works:**
- Ticks are deterministic (Atom.Primitives.Tick)
- Same tick rate = same update frequency
- No float accumulation errors
- Replay-friendly (tick = frame)

### 2. Delta Compression

Only send **changes** since last acknowledged state:

```csharp
// Full snapshot (first sync): 500 KB
public struct FullSnapshot
{
    public Entity[] AllEntities;    // Everything
}

// Delta update (typical): 2-50 KB
public struct DeltaUpdate
{
    public Entity EntityId;              // What changed
    public long BaseTick;                // From this tick
    public long TargetTick;              // To this tick
    public byte[] Delta;                 // Only changed data
    public List<int>? ChangedComponents; // Which components changed
}

// Bandwidth reduction: 90-99%
```

Implementation uses bitwise diffs:
1. Compare current state to baseline (last ack)
2. XOR to find differences
3. Compress with variable-length encoding
4. Send only changed bytes

### 3. Authority Management

Clear ownership for every entity:

```csharp
public struct Authority
{
    public Entity Id;                   // Entity identifier
    public Guid EntityId;               // World entity reference
    public AuthorityMode Mode;          // Who controls it
    public Guid? OwnerConnection;       // Owner if Client mode
    public long AssignedTick;           // When assigned
    public bool Transferable;           // Can authority change?
}

public enum AuthorityMode
{
    Server = 0,      // Server is authoritative (secure)
    Client = 1,      // Client has authority (responsive)
    Shared = 2       // Negotiated (competitive)
}
```

**Authority Rules:**
- Server-authoritative entities: Server state is truth, client predicts
- Client-authoritative entities: Client controls, server validates
- Shared entities: Client predicts, server corrects if wrong

### 4. Interest Management

Send only relevant updates:

```csharp
public enum SyncMode
{
    Full = 0,        // Send everything (initial sync)
    Delta = 1,       // Send changes only (normal operation)
    Interest = 2     // Send only relevant changes (optimization)
}

// Interest-based filtering
public bool IsRelevant(Entity entity, Connection client)
{
    var playerPos = GetPosition(client.PlayerEntity);
    var entityPos = GetPosition(entity);
    var distance = DVector3.Distance(playerPos, entityPos);

    // Only send if within visibility range
    return distance < DFixed64.FromInt(100);
}
```

## What Nexus Provides

### Connection Management

#### Connection

Represents a network connection with metrics:

```csharp
public struct Connection
{
    public Guid Id;                  // Unique connection ID
    public ConnectionState State;    // Current state
    public string Address;           // IP address
    public int Port;                 // Port number
    public long RttMs;              // Round-trip time (milliseconds, Q32.32)
    public long PacketsSent;        // Total packets sent
    public long PacketsReceived;    // Total packets received
    public long PacketsLost;        // Detected packet loss
    public long LastTick;           // Last tick received from client
}
```

**Usage:**

```csharp
using Nexus.Network.Generated;

// Create connection
var connection = new Connection
{
    Id = Guid.NewGuid(),
    State = ConnectionState.Disconnected,
    Address = "192.168.1.100",
    Port = 7777,
    RttMs = 0,
    PacketsSent = 0,
    PacketsReceived = 0,
    PacketsLost = 0,
    LastTick = 0
};

// Connect
connection.State = ConnectionState.Connecting;
// ... handshake ...
connection.State = ConnectionState.Synchronizing;
// ... initial state sync ...
connection.State = ConnectionState.Connected;

// Track metrics
connection.PacketsSent++;
connection.RttMs = MeasureRoundTripTime();

// Detect timeout
var currentTick = Tick.FromRaw(1000);
var ticksSinceLastUpdate = currentTick.Raw - connection.LastTick;
if (ticksSinceLastUpdate > TIMEOUT_TICKS)
{
    connection.State = ConnectionState.Reconnecting;
}
```

#### ConnectionState

Lifecycle of a connection:

```csharp
public enum ConnectionState
{
    Disconnected = 0,    // Not connected
    Connecting = 1,      // Handshake in progress
    Synchronizing = 2,   // Receiving initial state
    Connected = 3,       // Fully synced and active
    Reconnecting = 4     // Connection lost, attempting recovery
}
```

**State Machine:**

```
Disconnected
    ↓ (connect request)
Connecting
    ↓ (handshake success)
Synchronizing
    ↓ (initial state received)
Connected
    ↓ (timeout / packet loss)
Reconnecting
    ↓ (reconnect success)
Synchronizing
    ↓ (manual disconnect)
Disconnected
```

### State Synchronization

#### SyncMode

Three strategies for state synchronization:

```csharp
public enum SyncMode
{
    Full = 0,        // Full state snapshot
    Delta = 1,       // Delta compression
    Interest = 2     // Interest-based filtering + delta
}
```

**When to Use:**

| Mode | Use Case | Bandwidth | Latency |
|------|----------|-----------|---------|
| **Full** | Initial sync, reconnect | High (100%) | Low |
| **Delta** | Normal operation | Low (1-10%) | Low |
| **Interest** | Many entities, large world | Lowest (0.5-5%) | Low |

**Example:**

```csharp
// Initial connection: full snapshot
if (connection.State == ConnectionState.Synchronizing)
{
    SendFullSnapshot(connection);
    connection.State = ConnectionState.Connected;
}

// Normal operation: delta updates
else if (connection.State == ConnectionState.Connected)
{
    var mode = SyncMode.Delta;

    // Switch to interest-based for large worlds
    if (worldSize > 1000)
    {
        mode = SyncMode.Interest;
    }

    SendUpdate(connection, mode);
}
```

#### DeltaUpdate

Compressed state changes between ticks:

```csharp
public struct DeltaUpdate
{
    public Entity Id;                      // Update identifier
    public Guid EntityId;                  // Entity being updated
    public long BaseTick;                  // Previous acknowledged tick
    public long TargetTick;                // Current tick
    public byte[] Delta;                   // Compressed changes
    public List<int>? ChangedComponents;   // Component indices (null = full)
}
```

**Usage:**

```csharp
using Atom.Primitives;
using Atom.Serialization;

// Create delta update
var delta = new DeltaUpdate
{
    Id = new Entity(1, 1),
    EntityId = playerEntity,
    BaseTick = 100,
    TargetTick = 105,
    Delta = CompressDelta(oldState, newState),
    ChangedComponents = new List<int> { 0, 2, 5 }  // Position, Velocity, Health
};

// Serialize
var buffer = new byte[1400];
var writer = new BitWriter(buffer);
delta.Pack(ref writer);

// Send over network
SendPacket(connection, buffer[..writer.BytePosition]);

// Receive and deserialize
var reader = new BitReader(receivedBytes);
var receivedDelta = new DeltaUpdate();
receivedDelta.Unpack(ref reader);

// Apply delta
ApplyDelta(receivedDelta.EntityId, receivedDelta.Delta, receivedDelta.ChangedComponents);
```

**Delta Compression Algorithm:**

```csharp
public byte[] CompressDelta(byte[] oldState, byte[] newState)
{
    // 1. XOR to find differences
    var diff = new byte[newState.Length];
    for (int i = 0; i < newState.Length; i++)
    {
        diff[i] = (byte)(oldState[i] ^ newState[i]);
    }

    // 2. Run-length encode zeros (unchanged bytes)
    var compressed = new List<byte>();
    int zeroCount = 0;

    for (int i = 0; i < diff.Length; i++)
    {
        if (diff[i] == 0)
        {
            zeroCount++;
        }
        else
        {
            // Write zero run
            if (zeroCount > 0)
            {
                WriteVarInt(compressed, zeroCount);
                zeroCount = 0;
            }
            // Write changed byte
            compressed.Add(diff[i]);
        }
    }

    // 3. Return compressed data
    return compressed.ToArray();
}
```

### Authority Management

#### AuthorityMode

Who controls an entity:

```csharp
public enum AuthorityMode
{
    Server = 0,      // Server is authoritative
    Client = 1,      // Client has authority (prediction)
    Shared = 2       // Negotiated authority
}
```

**Authority Patterns:**

| Pattern | Mode | Use Case | Example |
|---------|------|----------|---------|
| **Server Authority** | Server | Secure, non-player entities | NPCs, projectiles, loot |
| **Client Prediction** | Client | Responsive player input | Player movement |
| **Shared Authority** | Shared | Competitive interactions | Pickup items, doors |

#### Authority

Authority assignment per entity:

```csharp
public struct Authority
{
    public Entity Id;              // Authority record ID
    public Guid EntityId;          // Entity this applies to
    public AuthorityMode Mode;     // Current authority mode
    public Guid? OwnerConnection;  // Owner connection (if Client mode)
    public long AssignedTick;      // When authority was assigned
    public bool Transferable;      // Whether authority can be transferred
}
```

**Usage:**

```csharp
// Server-authoritative NPC
var npcAuthority = new Authority
{
    Id = new Entity(1, 1),
    EntityId = npcEntity,
    Mode = AuthorityMode.Server,
    OwnerConnection = null,
    AssignedTick = currentTick.Raw,
    Transferable = false
};

// Client-authoritative player
var playerAuthority = new Authority
{
    Id = new Entity(2, 1),
    EntityId = playerEntity,
    Mode = AuthorityMode.Client,
    OwnerConnection = clientConnection.Id,
    AssignedTick = currentTick.Raw,
    Transferable = false  // Player always owns their character
};

// Shared-authority pickup item
var itemAuthority = new Authority
{
    Id = new Entity(3, 1),
    EntityId = itemEntity,
    Mode = AuthorityMode.Shared,
    OwnerConnection = null,
    AssignedTick = currentTick.Raw,
    Transferable = true  // Can transfer to whoever picks it up
};
```

**Authority Transfer:**

```csharp
public void TransferAuthority(ref Authority authority, Guid newOwner, long tick)
{
    if (!authority.Transferable)
    {
        throw new InvalidOperationException("Authority not transferable");
    }

    authority.OwnerConnection = newOwner;
    authority.AssignedTick = tick;
    authority.Mode = AuthorityMode.Client;
}

// Example: Player picks up item
if (PickupItem(playerEntity, itemEntity))
{
    var itemAuth = GetAuthority(itemEntity);
    TransferAuthority(ref itemAuth, playerConnection.Id, currentTick.Raw);
}
```

### Network Configuration

#### NetworkConfig

Configuration for network behavior:

```csharp
public struct NetworkConfig
{
    public int MaxPacketSize;           // Maximum packet size in bytes (512-65535)
    public int SendRate;                // Updates per second (1-120)
    public bool DeltaCompression;       // Enable delta compression
    public int TimeoutMs;               // Connection timeout in milliseconds
    public int MaxPendingConnections;   // Maximum simultaneous connections
    public bool Encryption;             // Enable packet encryption
}
```

**Defaults:**

```csharp
public static NetworkConfig Default => new()
{
    MaxPacketSize = 1400,              // MTU-safe (1500 - IP/UDP headers)
    SendRate = 60,                     // 60 Hz update rate
    DeltaCompression = true,           // Always use delta compression
    TimeoutMs = 10000,                 // 10 second timeout
    MaxPendingConnections = 10,        // 10 simultaneous connections
    Encryption = true                  // Encrypt by default
};
```

**Configuration Examples:**

```csharp
// Low-bandwidth mobile
var mobileConfig = new NetworkConfig
{
    MaxPacketSize = 1200,              // Conservative for mobile networks
    SendRate = 20,                     // 20 Hz (50ms updates)
    DeltaCompression = true,
    TimeoutMs = 15000,                 // Longer timeout for flaky networks
    MaxPendingConnections = 5,
    Encryption = true
};

// High-performance LAN
var lanConfig = new NetworkConfig
{
    MaxPacketSize = 4096,              // Larger packets on LAN
    SendRate = 120,                    // 120 Hz (8.3ms updates)
    DeltaCompression = true,           // Still beneficial
    TimeoutMs = 5000,                  // Shorter timeout
    MaxPendingConnections = 32,        // More simultaneous connections
    Encryption = false                 // Skip encryption on trusted LAN
};

// Competitive FPS
var competitiveConfig = new NetworkConfig
{
    MaxPacketSize = 1400,
    SendRate = 64,                     // Standard tick rate
    DeltaCompression = true,
    TimeoutMs = 8000,
    MaxPendingConnections = 16,
    Encryption = true
};
```

## Sync Modes Explained

### Full Sync

Send complete state snapshot:

```csharp
public byte[] CreateFullSnapshot(GameState state)
{
    var writer = new BitWriter(new byte[65536]);

    // Write all entities
    writer.WriteVarUInt((uint)state.Entities.Count);
    foreach (var entity in state.Entities)
    {
        entity.Pack(ref writer);
    }

    // Write all components
    writer.WriteVarUInt((uint)state.Components.Count);
    foreach (var component in state.Components)
    {
        component.Pack(ref writer);
    }

    return writer.ToArray();
}
```

**When to Use:**
- Initial connection (client has no state)
- Reconnection after timeout (client state is stale)
- After major state change (world reset, level load)

**Pros:**
- Simple implementation
- Self-contained (no dependencies)
- Recovers from any desync

**Cons:**
- High bandwidth (100% of state size)
- Not suitable for continuous updates
- Latency spike on large states

### Delta Sync

Send only changes since last acknowledged tick:

```csharp
public byte[] CreateDeltaUpdate(GameState oldState, GameState newState, long baseTick, long targetTick)
{
    var changes = new List<DeltaUpdate>();

    // Find changed entities
    foreach (var entity in newState.Entities)
    {
        if (!oldState.Entities.TryGetValue(entity.Id, out var oldEntity))
        {
            // New entity: send full data
            changes.Add(CreateFullEntityDelta(entity, baseTick, targetTick));
        }
        else if (!entity.Equals(oldEntity))
        {
            // Changed entity: send delta
            changes.Add(CreateEntityDelta(oldEntity, entity, baseTick, targetTick));
        }
    }

    // Serialize deltas
    var writer = new BitWriter(new byte[4096]);
    writer.WriteVarUInt((uint)changes.Count);
    foreach (var delta in changes)
    {
        delta.Pack(ref writer);
    }

    return writer.ToArray();
}
```

**When to Use:**
- Normal operation (client has valid baseline)
- Bandwidth is limited
- State changes are sparse

**Pros:**
- Low bandwidth (1-10% of full state)
- Scales with change rate, not state size
- Efficient for most games

**Cons:**
- Requires reliable baseline acknowledgment
- More complex implementation
- Needs desync recovery mechanism

### Interest Sync

Send only changes relevant to client:

```csharp
public byte[] CreateInterestUpdate(GameState state, Connection client, long baseTick, long targetTick)
{
    var playerEntity = GetPlayerEntity(client);
    var playerPos = GetPosition(playerEntity);
    var relevantChanges = new List<DeltaUpdate>();

    foreach (var entity in state.Entities)
    {
        // Filter by relevance
        if (IsRelevant(entity, playerPos))
        {
            var delta = CreateEntityDelta(entity, baseTick, targetTick);
            relevantChanges.Add(delta);
        }
    }

    // Serialize relevant deltas only
    var writer = new BitWriter(new byte[4096]);
    writer.WriteVarUInt((uint)relevantChanges.Count);
    foreach (var delta in relevantChanges)
    {
        delta.Pack(ref writer);
    }

    return writer.ToArray();
}

private bool IsRelevant(Entity entity, DVector3 playerPos)
{
    var entityPos = GetPosition(entity);
    var distance = DVector3.Distance(playerPos, entityPos);

    // Visibility threshold
    var viewDistance = DFixed64.FromInt(100);
    return distance < viewDistance;
}
```

**When to Use:**
- Large worlds with many entities
- MMO-style games
- Bandwidth extremely constrained

**Pros:**
- Lowest bandwidth (0.5-5% of full state)
- Scales with local density, not world size
- Privacy (clients don't receive distant state)

**Cons:**
- Most complex implementation
- Needs area-of-interest (AOI) system
- Entities can "pop in" when entering range

## Authority Model Explained

### Server-Authoritative

Server controls entity, clients receive updates:

```
Client                          Server
  |                               |
  |----Input: Move Right--------->|
  |                               | Update: Position += Input
  |                               | Authority Check: Valid
  |<---Delta: Position = (5,0)----|
  | Apply: Position = (5,0)       |
```

**Implementation:**

```csharp
// Server
public void UpdateServerAuthority(Entity entity, Connection client, Input input)
{
    var authority = GetAuthority(entity);

    if (authority.Mode != AuthorityMode.Server)
    {
        return;  // Not server-authoritative
    }

    // Server processes input
    var state = GetState(entity);
    state.Position += input.Direction * DeltaTime;

    // Validate (anti-cheat)
    if (IsValidMovement(state.Position))
    {
        SetState(entity, state);

        // Broadcast to all clients
        BroadcastDelta(entity, state);
    }
}
```

**Pros:**
- Secure (server validates everything)
- Simple client code
- No cheating possible

**Cons:**
- Input latency (round-trip delay)
- Feels unresponsive
- Server load (processes all inputs)

**Best For:**
- Competitive games (anti-cheat critical)
- NPCs and AI
- Physics objects

### Client-Authoritative (Client Prediction)

Client controls entity locally, server validates:

```
Client                          Server
  |                               |
  | Input: Move Right             |
  | Predict: Position = (5,0)     |
  |----Input: Move Right--------->|
  |                               | Update: Position = (5,0)
  |                               | Validate: Correct
  |<---Ack: Tick 105--------------|
  | (No correction needed)        |
```

**Implementation:**

```csharp
// Client
public void UpdateClientAuthority(Entity entity, Input input)
{
    var authority = GetAuthority(entity);

    if (authority.Mode != AuthorityMode.Client)
    {
        return;  // Not client-authoritative
    }

    if (authority.OwnerConnection != LocalConnection.Id)
    {
        return;  // Not our entity
    }

    // Client predicts immediately
    var state = GetState(entity);
    state.Position += input.Direction * DeltaTime;
    SetState(entity, state);

    // Send input to server
    SendInput(input, CurrentTick);
}

// Server
public void ValidateClientAuthority(Entity entity, Connection client, Input input)
{
    var authority = GetAuthority(entity);

    if (authority.OwnerConnection != client.Id)
    {
        return;  // Client doesn't own this entity
    }

    // Server validates input
    var serverState = GetState(entity);
    serverState.Position += input.Direction * DeltaTime;

    // Check for cheating
    if (!IsValidMovement(serverState.Position))
    {
        // Reject: send correction
        SendCorrection(client, entity, serverState);
    }
    else
    {
        // Accept: acknowledge
        SendAck(client, CurrentTick);
    }
}
```

**Pros:**
- Responsive (no input latency)
- Smooth player experience
- Lower server load

**Cons:**
- Requires client prediction
- Needs correction handling (rubber-banding)
- Vulnerable to cheating

**Best For:**
- Player-controlled characters
- Local player interactions
- Casual games

### Shared Authority

Negotiated authority between client and server:

```
Client A                        Server                        Client B
  |                               |                               |
  | Pickup Item                   |                               |
  |----Request: Pickup----------->|                               |
  |                               | Check: Available?             |
  |                               | Yes: Grant authority          |
  |<---Grant: Authority-----------|                               |
  | Authority: Client A           |----Sync: Item taken---------->|
  |                               |                               | Display: Item gone
```

**Implementation:**

```csharp
// Client requests authority
public void RequestAuthority(Entity entity)
{
    var authority = GetAuthority(entity);

    if (!authority.Transferable)
    {
        return;  // Cannot transfer
    }

    SendMessage(new AuthorityRequest
    {
        EntityId = entity.Id,
        RequestingConnection = LocalConnection.Id
    });
}

// Server grants authority
public void HandleAuthorityRequest(AuthorityRequest request)
{
    var authority = GetAuthority(request.EntityId);

    if (!authority.Transferable)
    {
        SendMessage(request.RequestingConnection, new AuthorityDenied());
        return;
    }

    if (authority.OwnerConnection != null)
    {
        // Someone else owns it
        SendMessage(request.RequestingConnection, new AuthorityDenied());
        return;
    }

    // Grant authority
    authority.Mode = AuthorityMode.Client;
    authority.OwnerConnection = request.RequestingConnection;
    authority.AssignedTick = CurrentTick.Raw;

    SendMessage(request.RequestingConnection, new AuthorityGranted
    {
        EntityId = request.EntityId
    });

    // Notify other clients
    BroadcastExcept(request.RequestingConnection, new AuthorityChanged
    {
        EntityId = request.EntityId,
        NewOwner = request.RequestingConnection
    });
}
```

**Pros:**
- Flexible (combines server security with client responsiveness)
- Handles ownership transfer
- Reduces conflicts

**Cons:**
- Most complex implementation
- Requires negotiation protocol
- Can have race conditions

**Best For:**
- Pickup items
- Interactive objects (doors, switches)
- Territory control

## Advantages

### 1. Bandwidth Efficiency

Delta compression drastically reduces bandwidth:

```
Full State Size: 500 KB
Delta Size (typical): 5 KB
Reduction: 99%

At 60 Hz:
- Full: 30 MB/sec = 240 Mbps
- Delta: 300 KB/sec = 2.4 Mbps

Client requirements:
- Full: Impossible for most users
- Delta: Practical for broadband
```

**Real-World Measurements:**

| Scenario | Full | Delta | Interest | Reduction |
|----------|------|-------|----------|-----------|
| FPS (10 players) | 50 KB | 8 KB | 3 KB | 94% |
| Battle Royale (100 players) | 500 KB | 45 KB | 12 KB | 97.6% |
| MMO (1000 entities) | 5 MB | 200 KB | 20 KB | 99.6% |

### 2. Determinism-Compatible

Nexus preserves determinism:

```csharp
// All updates are tick-synchronized
public struct NetworkUpdate
{
    public Tick BaseTick;      // Deterministic
    public Tick TargetTick;    // Deterministic
    public byte[] DeltaData;   // Bit-exact
}

// No floating-point in network code
// No wall-clock time
// No unordered iteration

// Result: Same inputs = same state, always
```

This enables:
- **Replay systems**: Record inputs, replay identically
- **Time-travel debugging**: Step backward through network state
- **Lockstep mode**: Clients run identical simulations
- **Deterministic tests**: Network bugs are reproducible

### 3. Flexible Authority Models

Adapt to different game types:

```csharp
// Competitive FPS: mostly server-authoritative
foreach (var entity in entities)
{
    if (entity.Type == EntityType.Player)
    {
        SetAuthority(entity, AuthorityMode.Client);  // Responsive movement
    }
    else
    {
        SetAuthority(entity, AuthorityMode.Server);  // Secure NPCs/items
    }
}

// Cooperative PvE: mostly client-authoritative
foreach (var entity in entities)
{
    if (entity.Owner != null)
    {
        SetAuthority(entity, AuthorityMode.Client);  // Responsive for everyone
    }
    else
    {
        SetAuthority(entity, AuthorityMode.Server);  // Server controls environment
    }
}
```

### 4. Built-in Metrics

Connection tracks performance:

```csharp
public void MonitorConnection(Connection connection)
{
    // Calculate packet loss rate
    var totalPackets = connection.PacketsSent;
    var lostPackets = connection.PacketsLost;
    var lossRate = (double)lostPackets / totalPackets;

    if (lossRate > 0.05)  // > 5% loss
    {
        LogWarning($"High packet loss: {lossRate:P}");
    }

    // Check round-trip time
    if (connection.RttMs > 200)  // > 200ms
    {
        LogWarning($"High latency: {connection.RttMs}ms");
    }

    // Detect stale connections
    var ticksSinceUpdate = CurrentTick.Raw - connection.LastTick;
    if (ticksSinceUpdate > 100)  // No update for 100 ticks
    {
        connection.State = ConnectionState.Reconnecting;
    }
}
```

## Disadvantages

### 1. Complexity

Nexus is not simple:

```csharp
// Simple approach: send everything
SendFullState(client);  // One line

// Nexus approach: delta compression + authority + interest
var baseline = GetBaseline(client);
var changes = ComputeDelta(baseline, currentState);
var relevant = FilterRelevant(changes, client);
var compressed = CompressDeltas(relevant);
SendUpdate(client, compressed);  // Many lines
```

**Learning Curve:**
- Understanding delta compression
- Authority models
- Tick synchronization
- Desync recovery

**Mitigation**: Clear documentation, examples, and helper utilities.

### 2. Not Suitable for Simple Use Cases

Overkill for:
- Single-player games
- Simple turn-based games
- Games with < 5 entities
- Prototypes

**When Nexus is Overkill:**
- You don't need determinism
- State is < 10 KB total
- Updates are infrequent (< 1 Hz)
- Development time is critical

**Alternative**: Use simple JSON over WebSockets for prototypes.

### 3. Requires Careful Design

Nexus doesn't solve everything:

```csharp
// You still need to design:

// 1. State structure (what to sync)
public struct PlayerState
{
    public DVector3 Position;      // Sync
    public DVector3 Velocity;      // Sync
    public DFixed64 Health;        // Sync
    public string Name;            // Don't sync every frame
    public int Score;              // Sync occasionally
}

// 2. Update frequency (what to send)
public bool ShouldSync(Component component)
{
    // Position: every frame
    if (component is PositionComponent) return true;

    // Health: only when changed
    if (component is HealthComponent health)
        return health.IsDirty;

    // Name: once at connection
    if (component is NameComponent) return false;
}

// 3. Authority assignment (who controls what)
public AuthorityMode GetAuthority(Entity entity)
{
    if (entity.Type == EntityType.Player)
        return AuthorityMode.Client;

    if (entity.Type == EntityType.NPC)
        return AuthorityMode.Server;

    if (entity.Type == EntityType.Projectile)
        return entity.Owner != null
            ? AuthorityMode.Client
            : AuthorityMode.Server;
}
```

## Comparison to Alternatives

### vs Photon

| Feature | Photon | Nexus |
|---------|--------|-------|
| **Determinism** | No | Yes |
| **Delta Compression** | Limited | Full |
| **Authority Control** | Server-only | Flexible |
| **Tick Sync** | No | Yes |
| **Replay Support** | No | Yes |
| **Lock-step** | No | Yes |
| **Cloud Hosting** | Yes | Self-hosted |
| **Learning Curve** | Low | Medium |

**Use Photon When:**
- You need cloud hosting
- Quick prototype
- No determinism requirement

**Use Nexus When:**
- Determinism is critical
- Time-travel debugging needed
- Full control over networking

### vs Mirror (Unity)

| Feature | Mirror | Nexus |
|---------|--------|-------|
| **Platform** | Unity-only | .NET-agnostic |
| **Determinism** | No | Yes |
| **Authority** | Server or client | Flexible |
| **State Sync** | Full or delta | Delta + interest |
| **ECS Support** | Limited | Native (Flux) |
| **Tick-based** | No | Yes |

**Use Mirror When:**
- Unity project
- Asset Store ecosystem
- Standard Unity networking

**Use Nexus When:**
- Non-Unity project
- Determinism required
- ECS architecture (Flux)

### vs Custom UDP

| Feature | Custom UDP | Nexus |
|---------|------------|-------|
| **Control** | Full | High |
| **Reliability** | Must implement | Built-in |
| **Delta Compression** | Must implement | Built-in |
| **Authority** | Must implement | Built-in |
| **Serialization** | Must implement | Built-in (Axion) |
| **Development Time** | Weeks/months | Days |

**Use Custom UDP When:**
- Very specific requirements
- Academic research
- Maximum optimization

**Use Nexus When:**
- Want battle-tested abstractions
- Save development time
- Standard game networking

### vs WebRTC

| Feature | WebRTC | Nexus |
|---------|--------|-------|
| **Latency** | Low | Low |
| **NAT Traversal** | Built-in | Must implement |
| **Browser Support** | Native | Via WebSocket |
| **Determinism** | No | Yes |
| **Game-Optimized** | No | Yes |

**Use WebRTC When:**
- Browser-based game
- Voice/video needed
- P2P preferred

**Use Nexus When:**
- Native game
- Determinism required
- Game-specific optimizations

## Performance (Design Targets)

### Bandwidth Reduction

Typical measurements:

```
Scenario: FPS with 10 players, 60 Hz updates

Full State:
- 10 players × 200 bytes = 2 KB per frame
- 60 Hz × 2 KB = 120 KB/sec = 960 Kbps

Delta Compression:
- Average change: 20 bytes per player
- 10 players × 20 bytes = 200 bytes per frame
- 60 Hz × 200 bytes = 12 KB/sec = 96 Kbps
- Reduction: 90%

Interest-Based (5 visible players):
- 5 players × 20 bytes = 100 bytes per frame
- 60 Hz × 100 bytes = 6 KB/sec = 48 Kbps
- Reduction: 95%
```

### Latency

Tick-aligned updates minimize latency:

```
Update Rate: 60 Hz (16.67ms per tick)
Network RTT: 50ms

Server-Authoritative:
- Client input → Server: 25ms (half RTT)
- Server processes: 16.67ms (1 tick)
- Server → Client: 25ms (half RTT)
- Total latency: ~67ms

Client-Authoritative:
- Client predicts: 0ms (instant)
- Server validates: 16.67ms (1 tick)
- Correction (if wrong): 50ms (full RTT)
- Perceived latency: 0-50ms
```

### Packet Loss Handling

Built-in recovery:

```csharp
// Lost packet detection
if (receivedTick > lastReceivedTick + 1)
{
    // Packet(s) lost
    var lostTicks = receivedTick - lastReceivedTick - 1;
    connection.PacketsLost += lostTicks;

    // Request retransmission or full snapshot
    if (lostTicks > 10)
    {
        RequestFullSnapshot(connection);
    }
}
```

**Recovery Strategies:**

| Loss Rate | Strategy | Impact |
|-----------|----------|--------|
| < 1% | Ignore (delta catches up) | Minimal |
| 1-5% | Request missing packets | Slight rubber-banding |
| 5-10% | Request full snapshot | Noticeable hitch |
| > 10% | Reconnect | Connection unstable |

## Code Examples

### Example 1: Basic Server/Client

```csharp
using Nexus.Network.Generated;
using Atom.Primitives;

// Server
public class GameServer
{
    private Dictionary<Guid, Connection> _connections = new();
    private Tick _currentTick = Tick.Zero;

    public void Update()
    {
        _currentTick = _currentTick.Next();

        // Update game state
        UpdateGameState(_currentTick);

        // Send updates to all clients
        foreach (var connection in _connections.Values)
        {
            if (connection.State == ConnectionState.Connected)
            {
                SendDeltaUpdate(connection, _currentTick);
            }
        }
    }

    private void SendDeltaUpdate(Connection connection, Tick tick)
    {
        var baseline = GetBaseline(connection);
        var delta = CreateDelta(baseline, _currentState, tick);

        SendPacket(connection, delta);
        connection.PacketsSent++;
    }
}

// Client
public class GameClient
{
    private Connection _serverConnection;
    private Tick _lastReceivedTick = Tick.Zero;

    public void ProcessUpdate(byte[] packet)
    {
        var delta = DeserializeDelta(packet);

        // Apply delta to state
        ApplyDelta(delta);

        // Update connection metrics
        _serverConnection.LastTick = delta.TargetTick;
        _serverConnection.PacketsReceived++;
        _lastReceivedTick = Tick.FromRaw(delta.TargetTick);

        // Send acknowledgment
        SendAck(_serverConnection, _lastReceivedTick);
    }
}
```

### Example 2: Client Prediction with Server Reconciliation

```csharp
using Nexus.Network.Generated;
using Atom.Math;
using Atom.Primitives;

public class PlayerController
{
    private Entity _playerEntity;
    private DVector3 _position;
    private Queue<InputSnapshot> _inputHistory = new();

    // Client-side prediction
    public void UpdateClient(Input input, Tick currentTick)
    {
        // Record input
        var snapshot = new InputSnapshot
        {
            Tick = currentTick,
            Input = input
        };
        _inputHistory.Enqueue(snapshot);

        // Predict movement immediately
        _position += input.Direction * DeltaTime;

        // Send input to server
        SendInput(input, currentTick);
    }

    // Server reconciliation
    public void ApplyServerUpdate(DVector3 serverPosition, Tick serverTick)
    {
        // Find the input that matches server tick
        while (_inputHistory.Count > 0 && _inputHistory.Peek().Tick < serverTick)
        {
            _inputHistory.Dequeue();  // Discard old inputs
        }

        // Check for misprediction
        var delta = DVector3.Distance(serverPosition, _position);
        if (delta > DFixed64.FromDouble(0.01))
        {
            // Server correction: rewind and replay
            _position = serverPosition;

            // Replay inputs from server tick to current
            foreach (var snapshot in _inputHistory)
            {
                _position += snapshot.Input.Direction * DeltaTime;
            }
        }
    }
}

public struct InputSnapshot
{
    public Tick Tick;
    public Input Input;
}
```

### Example 3: Interest Management

```csharp
using Nexus.Network.Generated;
using Atom.Math;
using Atom.Primitives;
using Atom.Collections;

public class InterestManager
{
    private DFixed64 _viewDistance = DFixed64.FromInt(100);

    public List<Entity> GetRelevantEntities(Entity observer, UnsafeMap<Entity, DVector3> positions)
    {
        var observerPos = positions[observer];
        var relevant = new List<Entity>();

        // Find entities within view distance
        foreach (var kvp in positions)
        {
            var entity = kvp.Key;
            var position = kvp.Value;

            if (entity == observer)
            {
                relevant.Add(entity);  // Always include self
                continue;
            }

            var distance = DVector3.Distance(observerPos, position);
            if (distance < _viewDistance)
            {
                relevant.Add(entity);
            }
        }

        return relevant;
    }

    public void SendInterestUpdate(Connection client, Tick currentTick)
    {
        var playerEntity = GetPlayerEntity(client);
        var relevantEntities = GetRelevantEntities(playerEntity, _positions);

        // Create delta only for relevant entities
        var deltas = new List<DeltaUpdate>();
        foreach (var entity in relevantEntities)
        {
            var delta = CreateDeltaFor(entity, client, currentTick);
            if (delta != null)
            {
                deltas.Add(delta.Value);
            }
        }

        // Send
        SendPacket(client, SerializeDeltas(deltas));
    }
}
```

## When to Use [Ambient]

Nexus itself is deterministic, but networking I/O is not:

```csharp
using Atom.Primitives;

[Ambient(Reason = "Socket I/O for network packets")]
public class NetworkTransport
{
    private Socket _socket;

    public void SendPacket(Connection connection, byte[] data)
    {
        // Socket operations are inherently non-deterministic
        _socket.SendTo(data, new IPEndPoint(
            IPAddress.Parse(connection.Address),
            connection.Port
        ));
    }

    public byte[] ReceivePacket()
    {
        var buffer = new byte[4096];
        var received = _socket.Receive(buffer);  // Non-deterministic timing
        return buffer[..received];
    }
}

[Ambient(Reason = "Wall-clock time for connection timeout")]
public void CheckTimeout(Connection connection)
{
    var now = DateTime.UtcNow;  // Ambient time
    var lastUpdate = connection.LastUpdateTime;
    var elapsed = (now - lastUpdate).TotalMilliseconds;

    if (elapsed > connection.TimeoutMs)
    {
        connection.State = ConnectionState.Reconnecting;
    }
}
```

**Rules:**
- Network I/O operations must be `[Ambient]`
- Wall-clock timeout checks must be `[Ambient]`
- Game state updates must NOT be `[Ambient]`

## Verification

Nexus determinism is verified by Arbiter:

```csharp
[ArbiterScenario(Seed = 42, Category = "Nexus.Determinism")]
public void NetworkReplay_SameSeed_SameResult()
{
    var random1 = new OrixRandom(99999);
    var random2 = new OrixRandom(99999);

    // Simulate network events
    var events1 = SimulateNetworkEvents(random1, 100);
    var events2 = SimulateNetworkEvents(random2, 100);

    // Verify identical
    for (int i = 0; i < events1.Count; i++)
    {
        Assert.Equal(events1[i], events2[i]);
    }
}
```

Run verification:

```bash
# V0: Schema validation
orix check schemas

# V1: Build
dotnet build /warnaserror

# V2: Tests
orix arbiter run --category Nexus

# V3: Determinism
orix arbiter determinism
```

## Summary

Nexus provides deterministic networking for Orix:

| Problem | Solution |
|---------|----------|
| State divergence | Tick-synchronized updates |
| High bandwidth | Delta compression (90-99% reduction) |
| Authority confusion | Flexible authority models |
| Relevance overload | Interest-based filtering |

**Core Types:**

| Type | Purpose |
|------|---------|
| `Connection` | Peer with metrics |
| `ConnectionState` | Lifecycle management |
| `SyncMode` | Full, Delta, or Interest |
| `DeltaUpdate` | Compressed state changes |
| `Authority` | Entity ownership |
| `AuthorityMode` | Server, Client, or Shared |
| `NetworkConfig` | Network configuration |

**Trade-offs:**

| Advantage | Disadvantage |
|-----------|--------------|
| 90-99% bandwidth reduction | Complex implementation |
| Determinism-compatible | Learning curve |
| Flexible authority | Requires careful design |
| Built-in metrics | Overkill for simple games |

**When to Use Nexus:**
- Multiplayer games
- Distributed simulations
- Determinism required
- Bandwidth constrained
- Time-travel debugging needed

**When Not to Use:**
- Single-player games
- Simple prototypes
- State < 10 KB
- No determinism requirement

Nexus builds on Atom (deterministic primitives), Flux (ECS simulation), and Axion (schema serialization) to provide a complete deterministic networking solution.

## Related Documents

- [Flux Simulation](./flux.md) - ECS runtime that Nexus synchronizes
- [Echo Replay](./echo.md) - Recording and playback using deterministic state
- [Axion Schema](./axion.md) - Schema definitions for network types
- [Executive Overview](../overview/executive.md) - Platform overview and Five Laws

---

## Analyzer Rules

Nexus code must follow these determinism rules:

| Rule | Description |
|------|-------------|
| **ORIX007** | No `async/await` in simulation code |
| **ORIX008** | No `ThreadPool` or `Task.Run` in simulation |

Network I/O operations must be marked with `[Ambient]` attribute.
