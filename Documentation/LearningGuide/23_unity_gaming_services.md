# 23: Unity Gaming Services Integration

> **Purpose:** Understand how Boss Room integrates Unity Gaming Services (UGS) including Authentication, Sessions (Lobby), and Relay for production multiplayer.

> 📍 **Key Files:**
> - [MultiplayerServicesInterface.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/UnityServices/Sessions/MultiplayerServicesInterface.cs) — Sessions API wrapper
> - [MultiplayerServicesFacade.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/UnityServices/Sessions/MultiplayerServicesFacade.cs) — High-level session operations
> - [ConnectionMethod.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionMethod.cs) — IP vs Relay strategies
> - [LocalSession.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/UnityServices/Sessions/LocalSession.cs) — Local session state

---

## Table of Contents

1. [Overview](#overview)
2. [Unity Authentication](#unity-authentication)
3. [Sessions (Lobby) System](#sessions-lobby-system)
4. [Unity Relay](#unity-relay)
5. [Connection Method Strategy](#connection-method-strategy)
6. [Session Lifecycle](#session-lifecycle)
7. [Error Handling](#error-handling)
8. [Configuration](#configuration)
9. [Key Takeaways](#key-takeaways)

---

## Overview

Unity Gaming Services (UGS) provides backend services for multiplayer games:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    UNITY GAMING SERVICES                              │
└──────────────────────────────────────────────────────────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Authentication  │     │    Sessions     │     │      Relay      │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│ - Anonymous     │     │ - Create lobby  │     │ - NAT punchthru │
│ - Persistent ID │     │ - Join by code  │     │ - No port fwd   │
│ - Cross-device  │     │ - Quick match   │     │ - Global relays │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │      Boss Room          │
                    │  ConnectionManager      │
                    └─────────────────────────┘
```

### Why UGS?

| Problem | UGS Solution |
|---------|--------------|
| Players can't connect (NAT) | Relay handles network traversal |
| Need stable player ID | Authentication provides persistent ID |
| Need to find games | Sessions provides lobby/matchmaking |
| Port forwarding | Not needed with Relay |

---

## Unity Authentication

### Anonymous Authentication

Boss Room uses anonymous auth so players don't need accounts:

```csharp
// In initialization code
await UnityServices.InitializeAsync();

if (!AuthenticationService.Instance.IsSignedIn)
{
    await AuthenticationService.Instance.SignInAnonymouslyAsync();
}

// Get persistent PlayerId
string playerId = AuthenticationService.Instance.PlayerId;
```

### PlayerId Retrieval with Fallback

**File:** `ConnectionMethod.cs` (lines 71-79)

```csharp
protected string GetPlayerId()
{
    // Check if UGS is initialized
    if (Services.Core.UnityServices.State != ServicesInitializationState.Initialized)
    {
        // Fallback: Use local GUID if UGS unavailable
        return ClientPrefs.GetGuid() + m_ProfileManager.Profile;
    }

    // Use UGS PlayerId if signed in, otherwise fallback
    return AuthenticationService.Instance.IsSignedIn 
        ? AuthenticationService.Instance.PlayerId 
        : ClientPrefs.GetGuid() + m_ProfileManager.Profile;
}
```

**Key insight:** Boss Room works offline too! UGS is optional for development.

---

## Sessions (Lobby) System

**File:** `Assets/Scripts/UnityServices/Sessions/MultiplayerServicesInterface.cs`

### Creating a Session

```csharp
public async Task<ISession> CreateSession(
    string sessionName, 
    int maxPlayers, 
    bool isPrivate, 
    Dictionary<string, PlayerProperty> playerProperties, 
    Dictionary<string, SessionProperty> sessionProperties)
{
    var sessionOptions = new SessionOptions
    {
        Name = sessionName,
        MaxPlayers = maxPlayers,
        IsPrivate = isPrivate,
        IsLocked = false,
        PlayerProperties = playerProperties,
        SessionProperties = sessionProperties
    }.WithRelayNetwork();  // Automatically sets up Relay!

    return await MultiplayerService.Instance.CreateSessionAsync(sessionOptions);
}
```

### Joining by Code

```csharp
public async Task<ISession> JoinSessionByCode(
    string sessionCode, 
    Dictionary<string, PlayerProperty> localUserData)
{
    var joinSessionOptions = new JoinSessionOptions
    {
        PlayerProperties = localUserData
    };
    return await MultiplayerService.Instance.JoinSessionByCodeAsync(sessionCode, joinSessionOptions);
}
```

### Quick Join (Matchmaking)

```csharp
public async Task<ISession> QuickJoinSession(Dictionary<string, PlayerProperty> localUserData)
{
    var quickJoinOptions = new QuickJoinOptions
    {
        Filters = m_FilterOptions,  // Only open sessions
        CreateSession = true        // Create if none found
    };

    var sessionOptions = new SessionOptions
    {
        MaxPlayers = k_MaxPlayers,
        PlayerProperties = localUserData
    }.WithRelayNetwork();

    return await MultiplayerService.Instance.MatchmakeSessionAsync(quickJoinOptions, sessionOptions);
}
```

### Querying Available Sessions

```csharp
public async Task<QuerySessionsResults> QueryAllSessions()
{
    var querySessionOptions = new QuerySessionsOptions
    {
        Count = k_MaxSessionsToShow,
        FilterOptions = m_FilterOptions,   // Only open slots
        SortOptions = m_SortOptions        // Newest first
    };
    return await MultiplayerService.Instance.QuerySessionsAsync(querySessionOptions);
}
```

### Filtering and Sorting

```csharp
public MultiplayerServicesInterface()
{
    // Filter: Only show sessions with available slots
    m_FilterOptions = new List<FilterOption>
    {
        new(FilterField.AvailableSlots, "0", FilterOperation.Greater)
    };

    // Sort: Newest sessions first
    m_SortOptions = new List<SortOption>
    {
        new(SortOrder.Descending, SortField.CreationTime)
    };
}
```

---

## Unity Relay

Relay enables connections without port forwarding or NAT configuration.

### How It Works

```
WITHOUT RELAY (Direct IP):
┌──────────┐                    ┌──────────┐
│  Host    │◄──── NAT ────X────►│  Client  │
│          │    (blocked!)      │          │
└──────────┘                    └──────────┘

WITH RELAY:
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Host    │◄───►│ Unity Relay  │◄───►│  Client  │
│          │     │   Server     │     │          │
└──────────┘     └──────────────┘     └──────────┘
                       ▲
        All traffic goes through Relay
        No NAT issues!
```

### Relay with Sessions

When using Sessions, Relay is integrated automatically:

```csharp
var sessionOptions = new SessionOptions
{
    Name = sessionName,
    MaxPlayers = maxPlayers,
    // ... other options
}.WithRelayNetwork();  // This line enables Relay!
```

The Session service handles:
1. Allocating a Relay server
2. Distributing join codes
3. Managing connectivity

---

## Connection Method Strategy

**File:** `Assets/Scripts/ConnectionManagement/ConnectionMethod.cs`

Boss Room uses the **Strategy Pattern** to support both IP and Relay connections.

### Base Class

```csharp
public abstract class ConnectionMethodBase
{
    protected ConnectionManager m_ConnectionManager;
    readonly ProfileManager m_ProfileManager;
    protected readonly string m_PlayerName;

    public abstract void SetupHostConnection();
    public abstract void SetupClientConnection();
    public abstract Task<(bool success, bool shouldTryAgain)> SetupClientReconnectionAsync();
    
    protected void SetConnectionPayload(string playerId, string playerName)
    {
        var payload = JsonUtility.ToJson(new ConnectionPayload
        {
            playerId = playerId,
            playerName = playerName,
            isDebug = Debug.isDebugBuild
        });

        m_ConnectionManager.NetworkManager.NetworkConfig.ConnectionData = 
            System.Text.Encoding.UTF8.GetBytes(payload);
    }
}
```

### IP Connection (Direct)

```csharp
class ConnectionMethodIP : ConnectionMethodBase
{
    string m_Ipaddress;
    ushort m_Port;

    public override void SetupClientConnection()
    {
        SetConnectionPayload(GetPlayerId(), m_PlayerName);
        var utp = (UnityTransport)m_ConnectionManager.NetworkManager.NetworkConfig.NetworkTransport;
        utp.SetConnectionData(m_Ipaddress, m_Port);
    }

    public override void SetupHostConnection()
    {
        SetConnectionPayload(GetPlayerId(), m_PlayerName);
        var utp = (UnityTransport)m_ConnectionManager.NetworkManager.NetworkConfig.NetworkTransport;
        utp.SetConnectionData(m_Ipaddress, m_Port);
    }

    public override Task<(bool success, bool shouldTryAgain)> SetupClientReconnectionAsync()
    {
        return Task.FromResult((true, true));  // Just retry
    }
}
```

### Relay Connection

```csharp
class ConnectionMethodRelay : ConnectionMethodBase
{
    MultiplayerServicesFacade m_MultiplayerServicesFacade;

    public override void SetupClientConnection()
    {
        // Relay setup is handled by Session service
        SetConnectionPayload(GetPlayerId(), m_PlayerName);
    }

    public override void SetupHostConnection()
    {
        Debug.Log("Setting up Unity Relay host");
        SetConnectionPayload(GetPlayerId(), m_PlayerName);
    }

    public override async Task<(bool success, bool shouldTryAgain)> SetupClientReconnectionAsync()
    {
        if (m_MultiplayerServicesFacade.CurrentUnitySession == null)
        {
            Debug.Log("Session does not exist anymore, stopping reconnection attempts.");
            return (false, false);  // Can't reconnect, don't retry
        }

        // Reconnect via Session service
        var session = await m_MultiplayerServicesFacade.ReconnectToSessionAsync();
        var success = session != null;
        Debug.Log(success ? "Successfully reconnected to Session." : "Failed to reconnect.");
        return (success, true);
    }
}
```

### Usage

```csharp
// In ConnectionManager or state
ConnectionMethodBase connectionMethod;

if (useRelay)
{
    connectionMethod = new ConnectionMethodRelay(
        multiplayerServicesFacade, connectionManager, profileManager, playerName);
}
else
{
    connectionMethod = new ConnectionMethodIP(
        ip, port, connectionManager, profileManager, playerName);
}

// Use the method (doesn't matter which type)
connectionMethod.SetupHostConnection();
NetworkManager.Singleton.StartHost();
```

---

## Session Lifecycle

```
┌──────────────────────────────────────────────────────────────────────┐
│                     SESSION LIFECYCLE                                 │
└──────────────────────────────────────────────────────────────────────┘

1. HOST CREATES SESSION
         │
         ├─► MultiplayerService.CreateSessionAsync()
         ├─► Session created with Relay allocation
         └─► Host gets session code to share
         
2. CLIENT JOINS
         │
         ├─► MultiplayerService.JoinSessionByCodeAsync()
         ├─► Client connects via Relay
         └─► Player added to session
         
3. GAMEPLAY
         │
         ├─► Session tracks connected players
         ├─► Host can lock session (no more joins)
         └─► Properties can be updated
         
4. PLAYER DISCONNECTS
         │
         ├─► Session marks player as disconnected
         ├─► Player has time window to reconnect
         └─► ReconnectToSessionAsync() to rejoin
         
5. SESSION ENDS
         │
         ├─► Host leaves or session destroyed
         └─► All players disconnected
```

### Reconnection

```csharp
public async Task<ISession> ReconnectToSession(string sessionId)
{
    return await MultiplayerService.Instance.ReconnectToSessionAsync(sessionId);
}
```

The Session service remembers disconnected players for a configurable time, allowing seamless reconnection.

---

## Error Handling

### Connection Errors

```csharp
public async Task<ISession> TryCreateSession()
{
    try
    {
        return await m_ServicesInterface.CreateSession(...);
    }
    catch (SessionServiceException e)
    {
        Debug.LogError($"Session creation failed: {e.Message}");
        
        // Handle specific errors
        switch (e.Reason)
        {
            case SessionError.SessionFull:
                ShowError("Session is full");
                break;
            case SessionError.SessionNotFound:
                ShowError("Session no longer exists");
                break;
            default:
                ShowError("Connection failed");
                break;
        }
        return null;
    }
}
```

### Authentication Errors

```csharp
try
{
    await AuthenticationService.Instance.SignInAnonymouslyAsync();
}
catch (AuthenticationException e)
{
    Debug.LogError($"Auth failed: {e.Message}");
    // Fall back to local-only mode
}
```

---

## Configuration

### Unity Dashboard Setup

1. Go to [dashboard.unity3d.com](https://dashboard.unity3d.com)
2. Create/select project
3. Enable services:
   - **Authentication** (for player IDs)
   - **Multiplayer** (for Sessions/Lobby)
   - **Relay** (for NAT traversal)

### In-Code Configuration

```csharp
// Session configuration
const int k_MaxSessionsToShow = 16;
const int k_MaxPlayers = 8;

// Transport configuration (in NetworkManager)
var utp = GetComponent<UnityTransport>();
utp.SetRelayServerData(...);  // Set by Session service
```

---

## Key Takeaways

| Concept | Description |
|---------|-------------|
| **Authentication** | Anonymous sign-in for persistent PlayerId |
| **Sessions** | Create/join/query game lobbies |
| **Relay** | Network traversal without port forwarding |
| **WithRelayNetwork()** | Automatic Relay setup for sessions |
| **Strategy Pattern** | IP vs Relay as swappable connection methods |
| **Offline Fallback** | Works without UGS for development |
| **Reconnection** | Session remembers disconnected players |

### When to Use Each

| Scenario | Use |
|----------|-----|
| Development/testing | IP connection (simpler) |
| LAN party | IP connection (lower latency) |
| Internet play | Relay (handles NAT) |
| Production | Relay + Sessions (full features) |

---

## Exercises

1. **Trace Session Creation:** Set a breakpoint in `CreateSession()` and trace the full flow from UI button to session created.

2. **Add Session Property:** Add a custom session property (e.g., "mapName") and display it in the lobby list.

3. **Implement Kick:** Add functionality for the host to kick a player from the session.

---

> **Related Guides:**
> - [10_connection_state_machine.md](./10_connection_state_machine.md) — Connection states
> - [20_session_reconnection.md](./20_session_reconnection.md) — PlayerId and reconnection
> - [04_design_patterns.md](./04_design_patterns.md) — Strategy pattern details
