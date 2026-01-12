# 30: Debugging Multiplayer Games

> **Purpose:** Learn practical techniques for debugging Unity Netcode for GameObjects (NGO) multiplayer games, including logging, network profiling, and common pitfalls.

---

## Table of Contents

1. [Debugging Challenges](#debugging-challenges)
2. [Logging Strategies](#logging-strategies)
3. [Multiple Editor Instances](#multiple-editor-instances)
4. [Network Profiler](#network-profiler)
5. [Common Pitfalls](#common-pitfalls)
6. [State Synchronization Issues](#state-synchronization-issues)
7. [Connection Problems](#connection-problems)
8. [Best Practices](#best-practices)
9. [Key Takeaways](#key-takeaways)

---

## Debugging Challenges

Multiplayer games are harder to debug:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SINGLE-PLAYER vs MULTIPLAYER                       │
└──────────────────────────────────────────────────────────────────────┘

SINGLE-PLAYER:
┌──────────────┐
│   Client     │  ← One process, one state, predictable
└──────────────┘

MULTIPLAYER:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Server     │ ←─► │   Client 1   │ ←─► │   Client 2   │
│  (Host)      │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
      ↑
      └── Different states, race conditions, network delays
```

### Why It's Hard

| Challenge | Description |
|-----------|-------------|
| **Multiple processes** | Need to debug host AND clients simultaneously |
| **Non-determinism** | Network delays cause timing variations |
| **Race conditions** | Operations may complete in different orders |
| **State divergence** | Server and client states can mismatch |
| **Reproduction** | Hard to recreate exact network conditions |

---

## Logging Strategies

### Contextual Logging

```csharp
public class ServerCharacter : NetworkBehaviour
{
    void TakeDamage(int damage, ulong attackerClientId)
    {
        // Include context: who, what, when, where
        Debug.Log($"[Server] Character {OwnerClientId} took {damage} damage " +
                  $"from client {attackerClientId} at {Time.time:F2}s " +
                  $"(HP: {m_Health.Value} → {m_Health.Value - damage})");
    }
}

public class ClientCharacter : NetworkBehaviour
{
    void OnHealthChanged(int oldValue, int newValue)
    {
        // Mark as client log
        Debug.Log($"[Client:{OwnerClientId}] Health changed: {oldValue} → {newValue}");
    }
}
```

### Prefixed Logging Pattern

```csharp
// Create consistent log prefixes
public static class NetLog
{
    public static void Server(string message)
    {
        Debug.Log($"[SERVER] {Time.time:F2}s: {message}");
    }
    
    public static void Client(ulong clientId, string message)
    {
        Debug.Log($"[CLIENT:{clientId}] {Time.time:F2}s: {message}");
    }
    
    public static void RPC(string rpcName, string direction, string data = "")
    {
        Debug.Log($"[RPC] {rpcName} ({direction}) {data}");
    }
}

// Usage
NetLog.Server("Spawning player for client 2");
NetLog.Client(NetworkManager.LocalClientId, "Received spawn confirmation");
NetLog.RPC("TakeDamageClientRpc", "Server→All", $"damage={damage}");
```

### Conditional Compilation

```csharp
public class DebugNetworking : MonoBehaviour
{
    // Only compile in development builds
    [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
    [System.Diagnostics.Conditional("UNITY_EDITOR")]
    public static void LogNetworkState(ServerCharacter character)
    {
        Debug.Log($"NetState: HP={character.NetHealthState.HitPoints.Value}, " +
                  $"Pos={character.transform.position}, " +
                  $"Action={character.CurrentAction}");
    }
}
```

---

## Multiple Editor Instances

### ParrelSync (Recommended)

ParrelSync creates linked project clones for testing:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PARRELSYNC SETUP                                   │
└──────────────────────────────────────────────────────────────────────┘

1. Install ParrelSync from Package Manager
2. ParrelSync → Clones Manager → Create Clone
3. Open clone in separate Unity Editor
4. Run host in main project, client in clone

MAIN PROJECT              CLONE PROJECT
┌──────────────┐          ┌──────────────┐
│  Host/Server │ ←──────► │   Client     │
│  (Play mode) │  Network │  (Play mode) │
└──────────────┘          └──────────────┘
```

### Build and Run Approach

```
1. Build standalone player
2. Run build as client
3. Run Editor as host/server

EDITOR (Host)              BUILD (Client)
┌──────────────┐          ┌──────────────┐
│   Debugger   │ ←──────► │   No debug   │
│   available  │  Network │   (logs only)│
└──────────────┘          └──────────────┘
```

---

## Network Profiler

### Unity Profiler - Network Module

```
Window → Analysis → Profiler → Add Profiler Module → Network Messages

KEY METRICS:
┌────────────────────────────────────────────────────────────────┐
│ Bytes Sent/Received    │ Total network traffic                 │
│ Messages Sent/Received │ Number of RPC/NetworkVariable updates │
│ Message Types          │ Breakdown by type                     │
└────────────────────────────────────────────────────────────────┘
```

### Multiplayer Tools Package

```
Install: com.unity.multiplayer.tools

Features:
- Network Simulator (artificial latency/packet loss)
- Runtime Net Stats Monitor
- Network Profiler with detailed breakdown
```

### Network Simulator Settings

```csharp
// Simulate poor network conditions
NetworkManager.Singleton.NetworkConfig.NetworkTransport
    .GetComponent<UnityTransport>()
    .SetDebugSimulatorParameters(
        packetDelayMS: 100,      // 100ms latency
        packetJitterMS: 25,       // ±25ms jitter
        packetDropRate: 5         // 5% packet loss
    );
```

---

## Common Pitfalls

### 1. Forgetting IsServer/IsClient Checks

```csharp
// ❌ BAD: Runs on all instances
void Update()
{
    MoveCharacter();  // Clients moving server objects!
}

// ✅ GOOD: Only server moves
void Update()
{
    if (!IsServer) return;
    MoveCharacter();
}
```

### 2. Modifying NetworkVariables on Client

```csharp
// ❌ BAD: Client tries to modify
public void TakeDamage(int damage)
{
    m_Health.Value -= damage;  // Only works on server!
}

// ✅ GOOD: Client requests, server modifies
[ServerRpc]
void TakeDamageServerRpc(int damage)
{
    m_Health.Value -= damage;  // Server-side only
}
```

### 3. RPC Called Before Spawn

```csharp
// ❌ BAD: RPC in Awake/Start
void Start()
{
    InitializeClientRpc();  // Object not spawned yet!
}

// ✅ GOOD: RPC in OnNetworkSpawn
public override void OnNetworkSpawn()
{
    if (IsServer)
    {
        InitializeClientRpc();  // Safe now
    }
}
```

### 4. Not Handling Late Joiners

```csharp
// ❌ BAD: Initial state only set once
void Start()
{
    SetInitialStateClientRpc(m_Config);  // Late joiners miss this!
}

// ✅ GOOD: Use NetworkVariable for state
NetworkVariable<int> m_State = new();  // Auto-syncs to late joiners
```

### 5. Comparing NetworkObject References

```csharp
// ❌ BAD: Reference comparison across network
if (targetObject == myObject)  // Different instances!

// ✅ GOOD: Compare NetworkObjectIds
if (targetObject.NetworkObjectId == myObject.NetworkObjectId)
```

---

## State Synchronization Issues

### Diagnosing Desync

```csharp
// Add debug visualization
void OnGUI()
{
    if (!IsOwner) return;
    
    GUILayout.Label($"Local Position: {transform.position}");
    GUILayout.Label($"Network Position: {m_NetworkPosition.Value}");
    GUILayout.Label($"Delta: {Vector3.Distance(transform.position, m_NetworkPosition.Value)}");
}
```

### Common Causes

| Symptom | Possible Cause |
|---------|----------------|
| Position rubber-banding | Client prediction without server reconciliation |
| Health mismatch | Client-side health modification |
| Missing objects | Spawn before NetworkVariable set |
| Duplicate actions | RPC called on both server and client |

### NetworkVariable Debugging

```csharp
public class DebugNetworkVariable<T> : NetworkVariable<T> where T : unmanaged
{
    public DebugNetworkVariable(T value = default) : base(value)
    {
        OnValueChanged += (old, current) =>
        {
            Debug.Log($"[NetVar] {typeof(T).Name} changed: {old} → {current}");
        };
    }
}
```

---

## Connection Problems

### Connection Failure Checklist

```
1. FIREWALL
   □ Port open? (default 7777)
   □ UDP allowed?
   
2. NAT/ROUTER
   □ Port forwarded?
   □ Use Relay for NAT traversal
   
3. NETWORK TRANSPORT
   □ UnityTransport configured?
   □ Correct connection data set?
   
4. AUTHENTICATION
   □ UGS signed in?
   □ Session/lobby exists?
```

### Debug Connection Setup

```csharp
public void DebugConnectionInfo()
{
    var transport = NetworkManager.Singleton.NetworkConfig.NetworkTransport 
                    as UnityTransport;
    
    Debug.Log($"Connection Type: {transport.Protocol}");
    Debug.Log($"Server Address: {transport.ConnectionData.Address}");
    Debug.Log($"Server Port: {transport.ConnectionData.Port}");
    Debug.Log($"Is Server: {NetworkManager.Singleton.IsServer}");
    Debug.Log($"Is Client: {NetworkManager.Singleton.IsClient}");
    Debug.Log($"Connected Clients: {NetworkManager.Singleton.ConnectedClientsIds.Count}");
}
```

---

## Best Practices

### DO ✅

```csharp
// 1. Log with context
Debug.Log($"[{(IsServer ? "SERVER" : "CLIENT")}] Action {action} on object {NetworkObjectId}");

// 2. Validate RPC parameters
[ServerRpc]
void DoActionServerRpc(ActionID actionId)
{
    if (!IsValidAction(actionId))
    {
        Debug.LogWarning($"Invalid action {actionId} from client {OwnerClientId}");
        return;
    }
}

// 3. Use Network Simulator for testing
// Test with 100ms latency, 5% packet loss

// 4. Test with minimum 3 players
// Host + Client reveals different issues

// 5. Add debug commands
[DebugCheat]
void ForceDesync()
{
    transform.position += Vector3.up * 10;  // Test resync
}
```

### DON'T ❌

```csharp
// 1. Don't assume operations are instant
SpawnObject();
GetComponent<...>();  // ❌ May not exist yet!

// 2. Don't ignore network callbacks
public override void OnNetworkSpawn()
{
    // ❌ Forgetting base call
    base.OnNetworkSpawn();  // ✅ Always call base
}

// 3. Don't test only in editor
// Build and test on actual hardware
```

---

## Key Takeaways

| Technique | Description |
|-----------|-------------|
| **Prefixed Logging** | [SERVER]/[CLIENT:id] for filtering |
| **ParrelSync** | Multiple editors for local testing |
| **Network Profiler** | Monitor bandwidth, RPCs, syncs |
| **Network Simulator** | Test with artificial latency |
| **IsServer/IsClient** | Always check before operations |
| **OnNetworkSpawn** | Safe place for network init |
| **NetworkObjectId** | Compare IDs, not references |

---

## Debugging Checklist

When something isn't working:

1. [ ] Check Console for errors on BOTH server and client
2. [ ] Verify IsServer/IsClient checks
3. [ ] Confirm NetworkObject is spawned
4. [ ] Check NetworkVariable write permissions
5. [ ] Verify RPC target (Server vs Client)
6. [ ] Test with multiple clients
7. [ ] Add logging at key points
8. [ ] Use Network Profiler

---

## Exercises

1. **Add Logging:** Add contextual logging to a NetworkBehaviour and trace an action from client input to server execution.

2. **Test With Latency:** Use Network Simulator to add 200ms latency and observe how the game behaves.

3. **Debug Desync:** Intentionally create a desync and practice diagnosing it.

---

> **Related Guides:**
> - [03_networking_essentials.md](./03_networking_essentials.md) — NGO basics
> - [15_testability_debugging.md](./15_testability_debugging.md) — General debugging
> - [27_error_handling_resilience.md](./27_error_handling_resilience.md) — Error patterns
