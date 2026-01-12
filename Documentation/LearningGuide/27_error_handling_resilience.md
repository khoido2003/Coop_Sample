# 27: Error Handling & Resilience Patterns

> **Purpose:** Understand how Boss Room handles errors gracefully, recovers from failures, and builds resilient multiplayer systems.

> 📍 **Key Files:**
> - [ClientConnectingState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/ClientConnectingState.cs) — Connection error handling
> - [StartingHostState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/StartingHostState.cs) — Host startup errors
> - [ConnectionManager.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionManager.cs) — Connection lifecycle
> - [ConnectStatus.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectStatus.cs) — Error codes

---

## Table of Contents

1. [Error Handling Philosophy](#error-handling-philosophy)
2. [Connection Error Handling](#connection-error-handling)
3. [Defensive Coding Patterns](#defensive-coding-patterns)
4. [Graceful Degradation](#graceful-degradation)
5. [Retry Strategies](#retry-strategies)
6. [User Feedback](#user-feedback)
7. [Best Practices](#best-practices)
8. [Key Takeaways](#key-takeaways)

---

## Error Handling Philosophy

Boss Room follows a **fail-safe** approach:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ERROR HANDLING PRINCIPLES                          │
└──────────────────────────────────────────────────────────────────────┘

1. Expect failures     → Network connections WILL fail
2. Fail gracefully     → Return to safe state (Offline)
3. Inform the user     → Publish error via ConnectStatus
4. Log for debugging   → Debug.LogError with context
5. Don't crash         → Catch and recover
```

---

## Connection Error Handling

### Try-Catch Pattern

**File:** `Assets/Scripts/ConnectionManagement/ConnectionState/ClientConnectingState.cs`

```csharp
internal void ConnectClientAsync()
{
    try
    {
        // Attempt connection setup
        m_ConnectionMethod.SetupClientConnection();

        if (m_ConnectionMethod is ConnectionMethodIP)
        {
            if (!m_ConnectionManager.NetworkManager.StartClient())
            {
                throw new Exception("NetworkManager StartClient failed");
            }
        }
    }
    catch (Exception e)
    {
        // 1. Log the error for developers
        Debug.LogError("Error connecting client, see following exception");
        Debug.LogException(e);
        
        // 2. Transition to safe state
        StartingClientFailed();
        
        // 3. Re-throw for upstream handling
        throw;
    }
}
```

### State Transition on Failure

```csharp
void StartingClientFailed()
{
    // 1. Check for specific disconnect reason from server
    var disconnectReason = m_ConnectionManager.NetworkManager.DisconnectReason;
    
    if (string.IsNullOrEmpty(disconnectReason))
    {
        // Generic failure
        m_ConnectStatusPublisher.Publish(ConnectStatus.StartClientFailed);
    }
    else
    {
        // Server provided reason (e.g., "lobby full")
        var connectStatus = JsonUtility.FromJson<ConnectStatus>(disconnectReason);
        m_ConnectStatusPublisher.Publish(connectStatus);
    }

    // 2. Always return to safe state
    m_ConnectionManager.ChangeState(m_ConnectionManager.m_Offline);
}
```

### Error Code Pattern

```csharp
public enum ConnectStatus
{
    Undefined,
    Success,
    
    // Connection failures
    ServerFull,
    LoggedInAgain,
    UserRequestedDisconnect,
    GenericDisconnect,
    
    // Startup failures
    StartClientFailed,
    StartHostFailed,
    
    // Permission failures
    IncompatibleBuildType,
    HostEndedSession
}
```

---

## Defensive Coding Patterns

### Null Checks

```csharp
// Pattern 1: Early return
public void ProcessCharacter(ServerCharacter character)
{
    if (character == null)
    {
        Debug.LogWarning("ProcessCharacter called with null character");
        return;
    }
    
    // Safe to proceed
    character.TakeDamage(10);
}

// Pattern 2: Null-conditional operator
void OnLifeStateChanged()
{
    m_Character?.NetLifeState?.LifeState?.OnValueChanged -= OnLifeStateChanged;
}

// Pattern 3: TryGet pattern
if (GameDataSource.Instance.TryGetActionPrototypeByID(actionID, out var action))
{
    ExecuteAction(action);
}
else
{
    Debug.LogWarning($"Action {actionID} not found");
}
```

### Validation Before Operations

```csharp
public void JoinSession(string sessionCode)
{
    // Validate input
    if (string.IsNullOrWhiteSpace(sessionCode))
    {
        m_ConnectStatusPublisher.Publish(ConnectStatus.StartClientFailed);
        return;
    }
    
    // Validate state
    if (m_ConnectionManager.CurrentState != ConnectionState.Offline)
    {
        Debug.LogWarning("Cannot join session while already connected");
        return;
    }
    
    // Proceed with valid input and state
    StartJoinSession(sessionCode);
}
```

### Assert for Debug Builds

```csharp
public void OnStateEnter(Animator animator)
{
    Debug.Assert(m_Animator == animator, "Animator mismatch!");
    
    // Continue with assured state
}
```

---

## Graceful Degradation

### Feature Fallbacks

```csharp
protected string GetPlayerId()
{
    // Try UGS authentication first
    if (Services.Core.UnityServices.State == ServicesInitializationState.Initialized 
        && AuthenticationService.Instance.IsSignedIn)
    {
        return AuthenticationService.Instance.PlayerId;
    }
    
    // Fallback to local GUID
    return ClientPrefs.GetGuid() + m_ProfileManager.Profile;
}
```

### Service Availability Checks

```csharp
public async Task<bool> TryInitializeServices()
{
    try
    {
        await UnityServices.InitializeAsync();
        await AuthenticationService.Instance.SignInAnonymouslyAsync();
        return true;
    }
    catch (Exception e)
    {
        Debug.LogWarning($"UGS unavailable, running in offline mode: {e.Message}");
        return false;
    }
}
```

---

## Retry Strategies

### Reconnection with Retry

```csharp
class ClientReconnectingState : ClientConnectingState
{
    const int k_NbReconnectAttempts = 2;
    int m_NbAttempts;

    public override void Enter()
    {
        m_NbAttempts = 0;
        TryReconnect();
    }

    async void TryReconnect()
    {
        // Attempt reconnection setup
        var (success, shouldRetry) = await m_ConnectionMethod.SetupClientReconnectionAsync();

        if (success)
        {
            // Try to start client
            if (!m_ConnectionManager.NetworkManager.StartClient())
            {
                HandleReconnectFailed(shouldRetry);
            }
        }
        else
        {
            HandleReconnectFailed(shouldRetry);
        }
    }

    void HandleReconnectFailed(bool shouldRetry)
    {
        if (shouldRetry && m_NbAttempts < k_NbReconnectAttempts)
        {
            m_NbAttempts++;
            TryReconnect();  // Retry
        }
        else
        {
            // Give up, go offline
            m_ConnectionManager.ChangeState(m_ConnectionManager.m_Offline);
        }
    }
}
```

### Exponential Backoff Pattern

```csharp
async Task RetryWithBackoff<T>(Func<Task<T>> operation, int maxRetries = 3)
{
    int delay = 1000;  // Start with 1 second
    
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            return await operation();
        }
        catch (Exception e)
        {
            Debug.LogWarning($"Attempt {i + 1} failed: {e.Message}");
            
            if (i < maxRetries - 1)
            {
                await Task.Delay(delay);
                delay *= 2;  // Double the delay each time
            }
            else
            {
                throw;  // Final attempt failed
            }
        }
    }
}
```

---

## User Feedback

### Error to UI Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ERROR → USER FEEDBACK FLOW                         │
└──────────────────────────────────────────────────────────────────────┘

1. Error occurs in connection code
           │
           ▼
2. Publish ConnectStatus via IPublisher
           │
           ▼
3. UI subscribes to ConnectStatus messages
           │
           ▼
4. UI shows appropriate error message to user
```

### Implementation

```csharp
// In connection state
m_ConnectStatusPublisher.Publish(ConnectStatus.ServerFull);

// In UI
[Inject]
void InjectDependencies(ISubscriber<ConnectStatus> subscriber)
{
    subscriber.Subscribe(OnConnectStatusChanged);
}

void OnConnectStatusChanged(ConnectStatus status)
{
    switch (status)
    {
        case ConnectStatus.ServerFull:
            ShowError("Server is full. Please try again later.");
            break;
        case ConnectStatus.StartClientFailed:
            ShowError("Failed to connect. Check your network.");
            break;
        case ConnectStatus.IncompatibleBuildType:
            ShowError("Version mismatch. Please update your game.");
            break;
    }
}
```

---

## Best Practices

### DO ✅

```csharp
// 1. Always have a safe fallback state
catch (Exception e)
{
    m_ConnectionManager.ChangeState(m_ConnectionManager.m_Offline);
}

// 2. Log errors with context
Debug.LogError($"Failed to spawn player {playerId} at position {position}");
Debug.LogException(e);

// 3. Use specific error codes
m_ConnectStatusPublisher.Publish(ConnectStatus.ServerFull);

// 4. Validate inputs before operations
if (string.IsNullOrEmpty(playerName))
{
    return ValidationResult.InvalidName;
}

// 5. Clean up resources on failure
catch (Exception)
{
    CleanupPartialState();
    throw;
}
```

### DON'T ❌

```csharp
// 1. Don't swallow exceptions silently
catch (Exception) { }  // ❌ Error hidden!

// 2. Don't show raw exceptions to users
catch (Exception e)
{
    ShowUI(e.ToString());  // ❌ User sees stack trace
}

// 3. Don't leave app in undefined state
catch (Exception)
{
    // ❌ No state transition, app stuck
}

// 4. Don't retry forever
while (true)
{
    try { Connect(); break; }
    catch { }  // ❌ Infinite loop possible
}
```

---

## Key Takeaways

| Concept | Description |
|---------|-------------|
| **Safe States** | Always transition to known-good state on error |
| **Error Codes** | Use enums for typed error handling |
| **PubSub for UI** | Publish errors, UI subscribes |
| **Defensive Coding** | Null checks, validation, assertions |
| **Graceful Degradation** | Fallbacks when services unavailable |
| **Limited Retries** | Retry with backoff, then fail |
| **Log + Inform** | Log for devs, message for users |

---

## Exercises

1. **Add Error Handling:** Find a place in the codebase that could benefit from try-catch and add appropriate error handling.

2. **Create Error Flow:** Trace what happens when a client tries to join a full server.

3. **Implement Retry:** Add retry logic to a network operation of your choice.

---

> **Related Guides:**
> - [10_connection_state_machine.md](./10_connection_state_machine.md) — Connection states
> - [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) — PubSub messaging
> - [20_session_reconnection.md](./20_session_reconnection.md) — Reconnection handling
