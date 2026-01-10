# 20: Session Management and Reconnection

> **Purpose:** Understand how Boss Room handles player sessions, reconnection, and state persistence across disconnects.

---

## Table of Contents

1. [The Core Problem](#the-core-problem)
2. [PlayerID vs ClientID](#playerid-vs-clientid)
3. [SessionManager Deep Dive](#sessionmanager-deep-dive)
4. [SessionPlayerData Structure](#sessionplayerdata-structure)
5. [The Reconnection Flow](#the-reconnection-flow)
6. [PersistentPlayer Pattern](#persistentplayer-pattern)
7. [Late Join Handling](#late-join-handling)
8. [Key Takeaways](#key-takeaways)

---

## The Core Problem

> **"If a player disconnects and reconnects, how do we know it's the same player?"**

In networked games, players disconnect all the time:
- Network hiccups
- Alt+Tab during loading
- Intentional disconnect to change settings

Without proper session management:
- Player loses all progress
- Re-spawns with wrong character
- HP resets to full
- Position resets to spawn point

**Boss Room's solution:** Use a **persistent PlayerId** that survives reconnection.

---

## PlayerID vs ClientID

This is the most confusing concept for newcomers:

| ID Type | What It Is | When It Changes | Who Assigns It |
|---------|------------|-----------------|----------------|
| **ClientID** | Network transport ID | Every connection | Netcode (automatic) |
| **PlayerId** | Unique install ID | Never (per device) | Client generates it |

### Example Scenario

```
First Connection:
  PlayerId = "player_abc123"  ← Generated from device GUID
  ClientID = 1                ← Assigned by Netcode

Player Disconnects...

Reconnection:
  PlayerId = "player_abc123"  ← SAME! (this is how we know it's them)
  ClientID = 5                ← DIFFERENT (new connection)
```

### Where PlayerId Comes From

```csharp
// ConnectionPayload sent during connection
[Serializable]
public class ConnectionPayload
{
    public string playerId;    // Unique to this device/install
    public string playerName;  // Display name
    public bool isDebug;       // Build type check
}
```

---

## SessionManager Deep Dive

**File:** [SessionManager.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Packages/com.unity.multiplayer.samples.coop/Utilities/Net/SessionManager.cs)

### The Two Dictionaries

```csharp
public class SessionManager<T> where T : struct, ISessionPlayerData
{
    // Lookup 1: PlayerId → Session Data
    Dictionary<string, T> m_ClientData;
    
    // Lookup 2: ClientID → PlayerId (mapping table)
    Dictionary<ulong, string> m_ClientIDToPlayerId;
    
    bool m_HasSessionStarted;
}
```

**Why two dictionaries?**

1. **m_ClientData** - The actual player data, keyed by stable PlayerId
2. **m_ClientIDToPlayerId** - Maps ephemeral ClientID to stable PlayerId

This allows looking up player data from either ID type.

### Connection Flow

```csharp
public void SetupConnectingPlayerSessionData(ulong clientId, string playerId, T sessionPlayerData)
{
    // Check: Is this playerId already connected?
    if (IsDuplicateConnection(playerId))
    {
        Debug.LogError("Duplicate connection - rejecting");
        return;
    }
    
    // Check: Is this a RECONNECTION?
    bool isReconnecting = m_ClientData.ContainsKey(playerId) 
                       && !m_ClientData[playerId].IsConnected;
    
    if (isReconnecting)
    {
        // Use OLD data, update NEW clientId
        sessionPlayerData = m_ClientData[playerId];
        sessionPlayerData.ClientID = clientId;
        sessionPlayerData.IsConnected = true;
    }
    
    // Store the data
    m_ClientIDToPlayerId[clientId] = playerId;
    m_ClientData[playerId] = sessionPlayerData;
}
```

### Disconnect Handling

```csharp
public void DisconnectClient(ulong clientId)
{
    if (m_HasSessionStarted)
    {
        // Session in progress - KEEP their data for reconnection
        var clientData = m_ClientData[playerId];
        clientData.IsConnected = false;  // Mark as disconnected only
        m_ClientData[playerId] = clientData;
    }
    else
    {
        // Still in lobby - REMOVE their data completely
        m_ClientIDToPlayerId.Remove(clientId);
        m_ClientData.Remove(playerId);
    }
}
```

**Key insight:** `m_HasSessionStarted` determines whether we keep or delete data!

---

## SessionPlayerData Structure

**File:** [SessionPlayerData.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/SessionPlayerData.cs)

```csharp
public struct SessionPlayerData : ISessionPlayerData
{
    // Display info
    public string PlayerName;
    public int PlayerNumber;
    
    // Character state (for reconnection)
    public Vector3 PlayerPosition;
    public Quaternion PlayerRotation;
    public NetworkGuid AvatarNetworkGuid;  // Which character class
    public int CurrentHitPoints;
    public bool HasCharacterSpawned;
    
    // Connection state
    public bool IsConnected { get; set; }
    public ulong ClientID { get; set; }
    
    // Reset for new game
    public void Reinitialize()
    {
        HasCharacterSpawned = false;
    }
}
```

### What Gets Preserved on Reconnection

| Data | Preserved? | Why |
|------|------------|-----|
| Character class | ✅ Yes | So they rejoin as same character |
| Position | ✅ Yes | So they resume where they were |
| HP | ✅ Yes | So they don't get free healing |
| Player name | ✅ Yes | Display consistency |
| HasCharacterSpawned | ✅ Yes | Know to restore vs fresh spawn |

---

## The Reconnection Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                       RECONNECTION FLOW                              │
└─────────────────────────────────────────────────────────────────────┘

Player connects (first time)
        │
        ▼
HostingState.ApprovalCheck()
        │
        ├─► IsDuplicateConnection(playerId)? → NO
        │
        ▼
SetupConnectingPlayerSessionData()
        │
        ├─► m_ClientData[playerId] = new data
        ├─► m_ClientIDToPlayerId[clientId] = playerId
        │
        ▼
SessionManager.OnSessionStarted()  ← Game begins
        │
        ▼
Player DISCONNECTS
        │
        ▼
DisconnectClient(clientId)
        │
        ├─► m_HasSessionStarted == true
        └─► clientData.IsConnected = false  ← Data KEPT!
        
═══════════════════════════════════════════════════════════════════════

Player RECONNECTS (same playerId, new clientId)
        │
        ▼
HostingState.ApprovalCheck()
        │
        ├─► IsDuplicateConnection(playerId)? → NO (not connected)
        │
        ▼
SetupConnectingPlayerSessionData()
        │
        ├─► isReconnecting = true (playerId exists, not connected)
        ├─► sessionPlayerData = m_ClientData[playerId]  ← Old data!
        ├─► sessionPlayerData.ClientID = newClientId
        │
        ▼
ServerBossRoomState.OnSynchronizeComplete()
        │
        ├─► SpawnPlayer(clientId, lateJoin: true)
        │
        ▼
Character spawns at OLD position with OLD HP!
```

---

## PersistentPlayer Pattern

**File:** [PersistentPlayer.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/PersistentPlayer.cs)

The `PersistentPlayer` is the "Default Player Prefab" in NetworkManager. It:
- Survives scene loads
- Holds NetworkVariables for name and avatar
- Is separate from the in-game character

### Why Separate from Character?

```
┌─────────────────────────────────────────────────────────────────────┐
│  PersistentPlayer                                                    │
│  (survives all scenes)                                               │
│  ├── NetworkNameState     → Player display name                      │
│  └── NetworkAvatarGuidState → Which character class selected         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ References when spawning
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ServerCharacter / ClientCharacter                                   │
│  (exists only in BossRoom scene)                                     │
│  ├── Copies AvatarGuid from PersistentPlayer                         │
│  └── destroyWithScene: true                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### OnNetworkSpawn Logic

```csharp
public override void OnNetworkSpawn()
{
    if (IsServer)
    {
        var sessionPlayerData = SessionManager<SessionPlayerData>.Instance.GetPlayerData(OwnerClientId);
        if (sessionPlayerData.HasValue)
        {
            var playerData = sessionPlayerData.Value;
            
            // Restore name
            m_NetworkNameState.Name.Value = playerData.PlayerName;
            
            // Check if this is a reconnection with existing character
            if (playerData.HasCharacterSpawned)
            {
                // Restore saved avatar
                m_NetworkAvatarGuidState.AvatarGuid.Value = playerData.AvatarNetworkGuid;
            }
            else
            {
                // First time - assign random avatar
                m_NetworkAvatarGuidState.SetRandomAvatar();
            }
        }
    }
}
```

---

## Late Join Handling

When a player joins after the game has started:

**File:** [ServerBossRoomState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameState/ServerBossRoomState.cs)

```csharp
void OnSynchronizeComplete(ulong clientId)
{
    // Check: Initial spawn already done? Player doesn't have character?
    if (InitialSpawnDone && !PlayerServerCharacter.GetPlayerServerCharacter(clientId))
    {
        // This is a LATE JOINER - spawn them now
        SpawnPlayer(clientId, lateJoin: true);
    }
}

void SpawnPlayer(ulong clientId, bool lateJoin)
{
    // ... spawn point selection ...
    
    if (lateJoin)
    {
        // Check for reconnection data
        SessionPlayerData? sessionPlayerData = SessionManager<SessionPlayerData>
            .Instance.GetPlayerData(clientId);
            
        if (sessionPlayerData is { HasCharacterSpawned: true })
        {
            // RECONNECTION - restore position
            physicsTransform.SetPositionAndRotation(
                sessionPlayerData.Value.PlayerPosition,
                sessionPlayerData.Value.PlayerRotation);
        }
    }
    
    // ... rest of spawn logic ...
}
```

---

## Session Lifecycle

```
OnSessionStarted()       ← Game begins, keep disconnect data
        │
        ▼
    [GAMEPLAY]
        │
        ▼
OnSessionEnded()         ← Game ends, clear disconnected, reinitialize
        │
        ▼
    [RETURN TO LOBBY]
        │
        ▼
OnServerEnded()          ← Server shuts down, clear everything
```

---

## Key Takeaways

| Concept | Explanation |
|---------|-------------|
| **PlayerId** | Stable ID per device, survives reconnection |
| **ClientID** | Ephemeral network ID, changes each connection |
| **SessionManager** | Maps ClientID → PlayerId → Session Data |
| **m_HasSessionStarted** | Determines if disconnect data is kept |
| **PersistentPlayer** | NetworkObject that survives scene loads |
| **Late Join** | OnSynchronizeComplete handles spawn |

### The Golden Rules

1. **Always use PlayerId for player identity**, not ClientID
2. **Keep session data on disconnect** during gameplay
3. **Clear session data on disconnect** during lobby
4. **Separate persistent data from scene objects**
5. **Handle late join explicitly** in game state

---

## Exercises

1. **Trace a Reconnection:** Disconnect a player during gameplay, reconnect, and watch the console logs. Follow the data through SessionManager.

2. **Add Reconnection Data:** Add a new field to SessionPlayerData (e.g., last action timestamp) and verify it persists across reconnection.

3. **Find the Duplicate Check:** Search for `IsDuplicateConnection` and understand when a connection is rejected.

---

> **Related:** [10_connection_state_machine.md](./10_connection_state_machine.md) — Connection states in detail
