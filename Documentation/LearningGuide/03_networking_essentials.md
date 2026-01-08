# 03: Networking Essentials

> **Purpose:** Understand multiplayer networking patterns that work for ANY online game. These principles are universal across game engines, languages, and networking libraries.

---

## Table of Contents

1. [The Four Golden Rules](#the-four-golden-rules)
2. [Network Architectures](#network-architectures)
3. [Synchronization Patterns](#synchronization-patterns)
4. [Latency Compensation](#latency-compensation)
5. [Connection Management](#connection-management)
6. [Security Considerations](#security-considerations)
7. [Bandwidth Optimization](#bandwidth-optimization)
8. [Common Pitfalls](#common-pitfalls)
9. [Boss Room Implementation Examples](#boss-room-implementation-examples)

---

## The Four Golden Rules

### Rule 1: Server is Authority

**The server is ALWAYS right. Clients only REQUEST, never DEMAND.**

```
Timeline of a player action:

CLIENT                                SERVER
   │                                     │
   │ "I want to attack"                  │
   │ ───────────────────────────────────►│
   │                                     │ Validate: Can this player attack?
   │                                     │ - Is player alive?
   │                                     │ - Is ability off cooldown?
   │                                     │ - Has enough mana?
   │                                     │
   │         "Attack approved"           │
   │◄─────────────────────────────────── │
   │                                     │
   │ Show attack animation               │ Apply damage to target
   │                                     │
```

**Why Server Authority?**

| Without Server Authority | With Server Authority |
|--------------------------|----------------------|
| Client says "I have 999999 health" | Server tracks real health |
| Client says "I teleported to flag" | Server validates movement |
| Client says "I dealt 1000 damage" | Server calculates actual damage |
| Cheater wins | Everyone plays fair |

**Implementation Pattern:**

```csharp
// Client sends REQUEST, not command
[Rpc(SendTo.Server)]
void RequestAttackServerRpc(ulong targetId)
{
    // Server code - runs on server
    if (!ValidateAttackRequest(targetId))
        return;
    
    // Server performs the actual attack
    PerformAttack(targetId);
}

// Never trust client data!
bool ValidateAttackRequest(ulong targetId)
{
    if (!IsAlive) return false;
    if (!IsOffCooldown) return false;
    if (mana < attackCost) return false;
    if (!IsValidTarget(targetId)) return false;
    
    return true;
}
```

---

### Rule 2: Hide Latency with Prediction

**Start visual feedback BEFORE server confirms to make the game feel responsive.**

```
WITHOUT PREDICTION (feels laggy):
─────────────────────────────────────────────────────────────────
T+0ms:   Player presses attack button
T+50ms:  Request reaches server
T+55ms:  Server processes, sends confirmation
T+105ms: Client receives confirmation
T+105ms: Client STARTS playing animation
─────────────────────────────────────────────────────────────────
Perceived delay: 105ms 😞


WITH PREDICTION (feels instant):
─────────────────────────────────────────────────────────────────
T+0ms:   Player presses attack button
T+0ms:   Client STARTS playing animation (anticipation)
T+50ms:  Request reaches server
T+55ms:  Server processes, sends confirmation
T+105ms: Client receives confirmation (animation already playing!)
─────────────────────────────────────────────────────────────────
Perceived delay: 0ms 😊
```

**Boss Room's Anticipation Pattern:**

```csharp
// Client-side: Anticipate immediately when button pressed
public void AnticipateAction(ActionRequestData data)
{
    // Start visual feedback NOW, before server confirms
    m_ClientCharacter.Animator.SetTrigger(action.Config.AnimationTrigger);
    m_IsAnticipated = true;
}

// Later, when server confirms:
public void OnStartClient(ClientCharacter client)
{
    if (!m_IsAnticipated)
    {
        // Only play if we weren't already anticipating
        client.Animator.SetTrigger(Config.AnimationTrigger);
    }
}
```

**What Can Be Predicted?**

| Safe to Predict | Needs Server Authority |
|-----------------|----------------------|
| Animations | Damage dealt |
| Sound effects | Health changes |
| Particle effects | Item pickups |
| UI feedback | Game state changes |
| Visual position | Actual position |

---

### Rule 3: Minimize Network Traffic

**Every byte counts. Send only what's necessary.**

```
BAD: Sending every frame
─────────────────────────────────────────────────────────────────
Frame 1: { position: (100.123, 50.456, 200.789), health: 100, mana: 50, ... }
Frame 2: { position: (100.234, 50.456, 200.901), health: 100, mana: 50, ... }
Frame 3: { position: (100.345, 50.456, 201.012), health: 100, mana: 50, ... }
... 60 packets per second x 100 bytes = 6000 bytes/sec per player!

GOOD: Sending only changes
─────────────────────────────────────────────────────────────────
Frame 1: { position: (100, 50, 200) }  // Initial
Frame 15: { position: (115, 50, 215) } // Only when moved significantly
Frame 42: { health: 85 }               // Only when health changed
... Much less data!
```

**Optimization Techniques:**

| Technique | Before | After |
|-----------|--------|-------|
| Use IDs instead of strings | "FireballSpell" (12 bytes) | ID: 3 (4 bytes) |
| Quantize floats | 100.12345f (4 bytes) | short: 1001 (2 bytes) |
| Delta compression | Full position (12 bytes) | Delta from last (6 bytes) |
| Bit packing | bool as byte (1 byte) | 8 bools in 1 byte |
| Only send on change | Every frame | On value change |

**Boss Room Example:**

```csharp
// ActionID is a simple struct with an int
public struct ActionID : INetworkSerializeByMemcpy
{
    public int ID;  // 4 bytes instead of string
}

// NetworkVariable only syncs when value actually changes
public NetworkVariable<int> HitPoints = new();

void TakeDamage(int amount)
{
    HitPoints.Value -= amount; // Syncs only because value changed
    HitPoints.Value -= 0;      // Does NOT sync - same value
}
```

---

### Rule 4: Handle Disconnects Gracefully

**Network WILL fail. Your game must handle it.**

```
Possible failure scenarios:
┌────────────────────────────────────────────────────────────────┐
│ 1. Client loses internet mid-game                              │
│ 2. Host quits while clients are connected                      │
│ 3. Network too slow (timeout)                                  │
│ 4. Packet loss corrupts messages                               │
│ 5. IP changes (mobile switching wifi/cellular)                 │
│ 6. Server crashes                                              │
└────────────────────────────────────────────────────────────────┘
```

**Boss Room's Connection State Machine:**

```
┌──────────┐    ┌────────────┐    ┌────────────────┐
│ Offline  │───►│ Connecting │───►│ Connected      │
└──────────┘    └────────────┘    └────────────────┘
     ▲                │                   │
     │                │ (failed)          │ (disconnect)
     │                ▼                   ▼
     │          ┌──────────┐    ┌────────────────┐
     └──────────│ Offline  │◄───│ Reconnecting   │
                └──────────┘    └────────────────┘
                                      │  ▲
                                      │  │ (retry)
                                      └──┘
```

**Key Principles:**

1. **Save session data** - So player can reconnect
2. **Retry with backoff** - Don't spam reconnect attempts
3. **Show feedback** - Player needs to know connection state
4. **Graceful host migration** - Or clear messaging if not supported

---

## Network Architectures

### Client-Server (Most Common)

```
       ┌─────────────────┐
       │     SERVER      │
       │  (Authoritative)│
       └────────┬────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│Client 1│ │Client 2│ │Client 3│
└────────┘ └────────┘ └────────┘

Pros: Easy to secure, consistent state
Cons: Requires dedicated server or player-host
```

### Listen Server / Host-Client (Boss Room Uses This)

```
┌────────────────────────────────┐
│     HOST (Server + Client)     │
│  Server logic │ Client logic   │
└──────────────┬─────────────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│Client 1│ │Client 2│ │Client 3│
└────────┘ └────────┘ └────────┘

Pros: No dedicated server needed
Cons: Host advantage, host quits = game ends
```

### Peer-to-Peer (Not Recommended for Games)

```
┌────────┐     ┌────────┐
│Client 1│────▶│Client 2│
└───┬────┘     └────┬───┘
    │               │
    ▼               ▼
┌────────┐     ┌────────┐
│Client 3│◄───▶│Client 4│
└────────┘     └────────┘

Pros: No server needed
Cons: Hard to secure, state conflicts, NAT issues
```

---

## Synchronization Patterns

### Pattern 1: NetworkVariable (State Sync)

**Use for:** Continuous state that clients need to track.

```csharp
// Health, mana, score, etc.
public class Character : NetworkBehaviour
{
    // Automatically synced to all clients
    public NetworkVariable<int> Health = new(
        100,                                    // Default value
        NetworkVariableReadPermission.Everyone, // Who can read
        NetworkVariableWritePermission.Server   // Who can write
    );
    
    // React to changes
    public override void OnNetworkSpawn()
    {
        Health.OnValueChanged += OnHealthChanged;
    }
    
    void OnHealthChanged(int previousValue, int newValue)
    {
        UpdateHealthBar(newValue);
        
        if (newValue < previousValue)
        {
            PlayHitEffect();
        }
    }
    
    // Only server modifies
    [Rpc(SendTo.Server)]
    public void TakeDamageServerRpc(int damage)
    {
        Health.Value -= damage;  // Automatically synced!
    }
}
```

### Pattern 2: RPC (Remote Procedure Call)

**Use for:** One-time events and actions.

```csharp
// Client → Server: Request action
[Rpc(SendTo.Server)]
void RequestActionServerRpc(ActionRequestData data, RpcParams rpcParams = default)
{
    // Validate request
    var clientId = rpcParams.Receive.SenderClientId;
    if (!ValidateAction(clientId, data))
        return;
    
    // Perform action
    PerformAction(data);
    
    // Notify all clients
    NotifyActionClientRpc(data);
}

// Server → All Clients: Notify of action
[Rpc(SendTo.ClientsAndHost)]
void NotifyActionClientRpc(ActionRequestData data)
{
    // Show visual feedback on all clients
    PlayActionVisuals(data);
}

// Server → Specific Client: Private message
[Rpc(SendTo.SpecificClients)]
void PrivateMessageClientRpc(string message, RpcParams rpcParams = default)
{
    ShowPrivateMessage(message);
}
```

### Pattern 3: NetworkList

**Use for:** Collections that change (player list, inventory).

```csharp
public class LobbyManager : NetworkBehaviour
{
    public NetworkList<PlayerInfo> Players;
    
    void Awake()
    {
        Players = new NetworkList<PlayerInfo>();
    }
    
    public override void OnNetworkSpawn()
    {
        Players.OnListChanged += OnPlayersChanged;
    }
    
    void OnPlayersChanged(NetworkListEvent<PlayerInfo> evt)
    {
        switch (evt.Type)
        {
            case NetworkListEvent<PlayerInfo>.EventType.Add:
                OnPlayerJoined(evt.Value);
                break;
            case NetworkListEvent<PlayerInfo>.EventType.Remove:
                OnPlayerLeft(evt.Value);
                break;
            case NetworkListEvent<PlayerInfo>.EventType.Value:
                OnPlayerUpdated(evt.Index, evt.Value);
                break;
        }
    }
}

[System.Serializable]
public struct PlayerInfo : INetworkSerializable, IEquatable<PlayerInfo>
{
    public ulong ClientId;
    public FixedString64Bytes Name;
    public int CharacterClass;
    
    public void NetworkSerialize<T>(BufferSerializer<T> serializer) where T : IReaderWriter
    {
        serializer.SerializeValue(ref ClientId);
        serializer.SerializeValue(ref Name);
        serializer.SerializeValue(ref CharacterClass);
    }
    
    public bool Equals(PlayerInfo other) => ClientId == other.ClientId;
}
```

### Pattern 4: Network Spawning

```csharp
// Server spawns objects - visible to all
void SpawnEnemy(Vector3 position)
{
    // Only server can spawn
    if (!IsServer) return;
    
    var enemy = Instantiate(enemyPrefab, position, Quaternion.identity);
    enemy.GetComponent<NetworkObject>().Spawn();
}

// Spawn as player object (owned by client)
void SpawnPlayerCharacter(ulong clientId)
{
    var player = Instantiate(playerPrefab);
    player.GetComponent<NetworkObject>().SpawnAsPlayerObject(clientId);
    
    // This client now owns the object
}

// Despawn
void DespawnEnemy(NetworkObject enemy)
{
    enemy.Despawn();  // Removed from all clients
    // Or: enemy.Despawn(destroy: true) to also destroy locally
}
```

---

## Latency Compensation

### Input Buffering

```csharp
// Buffer inputs to smooth out network jitter
public class InputBuffer
{
    private Queue<InputFrame> buffer = new Queue<InputFrame>();
    private const int BUFFER_SIZE = 3;
    
    public void AddInput(InputFrame input)
    {
        buffer.Enqueue(input);
        
        // Keep buffer at target size
        while (buffer.Count > BUFFER_SIZE)
        {
            buffer.Dequeue();
        }
    }
    
    public InputFrame GetInput()
    {
        return buffer.Count > 0 ? buffer.Dequeue() : default;
    }
}
```

### Entity Interpolation

```csharp
// Smooth other players' movement between network updates
public class NetworkPositionInterpolator : MonoBehaviour
{
    private Vector3 targetPosition;
    private Vector3 previousPosition;
    private float interpolationTime;
    private float lastUpdateTime;
    
    public void OnPositionReceived(Vector3 newPosition)
    {
        previousPosition = targetPosition;
        targetPosition = newPosition;
        lastUpdateTime = Time.time;
    }
    
    void Update()
    {
        // Interpolate between last two known positions
        float t = (Time.time - lastUpdateTime) / interpolationTime;
        transform.position = Vector3.Lerp(previousPosition, targetPosition, t);
    }
}
```

---

## Connection Management

### Connection Approval

```csharp
void Start()
{
    NetworkManager.Singleton.ConnectionApprovalCallback = ApproveConnection;
}

void ApproveConnection(
    NetworkManager.ConnectionApprovalRequest request,
    NetworkManager.ConnectionApprovalResponse response)
{
    // Decode client's payload
    var payload = JsonUtility.FromJson<ConnectionPayload>(
        System.Text.Encoding.UTF8.GetString(request.Payload));
    
    // Check 1: Server full?
    if (NetworkManager.Singleton.ConnectedClientsIds.Count >= maxPlayers)
    {
        response.Approved = false;
        response.Reason = "Server is full";
        return;
    }
    
    // Check 2: Version match?
    if (payload.gameVersion != Application.version)
    {
        response.Approved = false;
        response.Reason = "Version mismatch";
        return;
    }
    
    // Check 3: Banned?
    if (IsBanned(payload.playerId))
    {
        response.Approved = false;
        response.Reason = "You are banned";
        return;
    }
    
    // Approved!
    response.Approved = true;
    response.CreatePlayerObject = true;
    response.Position = GetSpawnPosition();
}
```

### Session Data (For Reconnection)

```csharp
// Store player data on server for reconnection
public class SessionManager
{
    private Dictionary<string, PlayerSessionData> sessions = new();
    
    public void SaveSession(string playerId, PlayerSessionData data)
    {
        sessions[playerId] = data;
    }
    
    public bool TryRestoreSession(string playerId, out PlayerSessionData data)
    {
        return sessions.TryGetValue(playerId, out data);
    }
}

public struct PlayerSessionData
{
    public int characterClass;
    public int health;
    public int experience;
    public Vector3 lastPosition;
}
```

---

## Security Considerations

### Never Trust the Client

```csharp
// ❌ BAD: Trust client data
[Rpc(SendTo.Server)]
void DealDamageServerRpc(int damage)
{
    target.Health -= damage;  // Client controls damage amount!
}

// ✅ GOOD: Server calculates
[Rpc(SendTo.Server)]
void RequestAttackServerRpc(ulong targetId)
{
    // Server calculates damage based on attacker's stats
    int damage = CalculateDamage(attackerStats, targetDefense);
    target.Health -= damage;
}
```

### Validate Everything

```csharp
bool ValidateMovement(Vector3 requestedPosition)
{
    // Check distance from current position
    float maxMoveDistance = moveSpeed * Time.deltaTime * 1.5f; // With margin
    if (Vector3.Distance(currentPosition, requestedPosition) > maxMoveDistance)
        return false;
    
    // Check for walls/obstacles
    if (Physics.Linecast(currentPosition, requestedPosition))
        return false;
    
    // Check for bounds
    if (!IsWithinPlayArea(requestedPosition))
        return false;
    
    return true;
}
```

### Rate Limiting

```csharp
private Dictionary<ulong, float> lastActionTime = new();

bool IsActionRateLimited(ulong clientId)
{
    if (lastActionTime.TryGetValue(clientId, out float lastTime))
    {
        if (Time.time - lastTime < minActionInterval)
            return true;  // Too fast!
    }
    
    lastActionTime[clientId] = Time.time;
    return false;
}
```

---

## Bandwidth Optimization

### Quantization

```csharp
// Compress position to fewer bytes
public struct CompressedPosition : INetworkSerializable
{
    // Use shorts instead of floats (2 bytes vs 4 bytes each)
    // Precision: 0.1 units
    public short x, y, z;
    
    public CompressedPosition(Vector3 position)
    {
        x = (short)(position.x * 10);
        y = (short)(position.y * 10);
        z = (short)(position.z * 10);
    }
    
    public Vector3 ToVector3()
    {
        return new Vector3(x / 10f, y / 10f, z / 10f);
    }
    
    public void NetworkSerialize<T>(BufferSerializer<T> serializer) where T : IReaderWriter
    {
        serializer.SerializeValue(ref x);
        serializer.SerializeValue(ref y);
        serializer.SerializeValue(ref z);
    }
}

// 12 bytes → 6 bytes (50% reduction!)
```

### Delta Compression

```csharp
// Only send what changed
[Flags]
public enum DirtyFlags : byte
{
    None = 0,
    Position = 1 << 0,
    Rotation = 1 << 1,
    Health = 1 << 2,
    Animation = 1 << 3
}

// Include flags in packet, only serialize dirty fields
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Client modifying server state | State desync | Only server modifies |
| Sending every frame | Wastes bandwidth | Send on change |
| Trusting client data | Enables cheating | Validate on server |
| No disconnect handling | Game breaks | State machine |
| Large payloads | High latency | Use IDs, compression |
| Synchronous operations | Game freezes | Use async/await |
| No lag compensation | Feels unresponsive | Prediction, interpolation |

---

## Boss Room Implementation Examples

### Action Request Flow

```csharp
// File: ServerCharacter.cs
[Rpc(SendTo.Server)]
public void RecvDoActionServerRPC(ActionRequestData data)
{
    ActionPlayer.PlayAction(ref data);
}
```

### NetworkVariable for Health

```csharp
// File: NetworkHealthState.cs
public NetworkVariable<int> HitPoints { get; } = new NetworkVariable<int>();

// Only server writes
public void ApplyDamage(int damage)
{
    if (!IsServer) return;
    HitPoints.Value = Mathf.Max(0, HitPoints.Value - damage);
}
```

### Connection State Machine

```csharp
// File: ConnectionManager.cs
internal void ChangeState(ConnectionState nextState)
{
    Debug.Log($"Changed connection state from {m_CurrentState.GetType().Name} " +
              $"to {nextState.GetType().Name}.");
    
    m_CurrentState?.Exit();
    m_CurrentState = nextState;
    m_CurrentState.Enter();
}
```

---

## Quick Reference

| Need | Solution | Details |
|------|----------|---------|
| Sync a value | `NetworkVariable<T>` | Auto-syncs on change |
| Sync a list | `NetworkList<T>` | With change events |
| One-time event | RPC | Client/Server/Everyone |
| Sync position | `NetworkTransform` | Built-in component |
| Sync physics | `NetworkRigidbody` | Built-in component |
| Sync animation | `NetworkAnimator` | Built-in component |
| Player action | Client RPC → Server | Server validates, executes |
| Visual effect | Server RPC → Clients | Server triggers, clients show |

---

## Exercises

1. **Find validation code:** Open `HostingState.cs` and find where connection approval happens
2. **Trace an action:** Follow what happens when attack button is pressed to damage dealt
3. **Study reconnection:** Read `ClientReconnectingState.cs` and understand the retry logic
4. **Identify anticipation:** Find where client anticipates actions before server confirms

---

> **Cross-Reference:**
> - [10_connection_state_machine.md](./10_connection_state_machine.md) - Deep dive on connection management
> - [09_action_system_deepdive.md](./09_action_system_deepdive.md) - Action/ability system details
> - [12_system_flow_diagrams.md](./12_system_flow_diagrams.md) - Visual flow diagrams
