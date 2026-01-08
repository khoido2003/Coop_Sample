# 15: Testability & Debugging Patterns

> **Purpose:** How Boss Room's architecture enables testing, and how to debug effectively in well-architected code.

---

## Table of Contents

1. [Why Testability Matters](#why-testability-matters)
2. [How DI Enables Testing](#how-di-enables-testing)
3. [Creating Mock Implementations](#creating-mock-implementations)
4. [Unit Testing Patterns](#unit-testing-patterns)
5. [Integration Testing Approaches](#integration-testing-approaches)
6. [Debug Logging Patterns](#debug-logging-patterns)
7. [Debugging Complex Systems](#debugging-complex-systems)
8. [Performance Profiling Workflow](#performance-profiling-workflow)

---

## Why Testability Matters

### The Cost of Untestable Code

```
Without Tests:
- Change code → Manual testing → Miss edge case → Bug in production
- Time: 5 minutes to change, 2 hours to manually test, 1 day to fix bug

With Tests:
- Change code → Run tests → Fail instantly → Fix immediately
- Time: 5 minutes to change, 10 seconds to test, 2 minutes to fix
```

### What Makes Code Testable?

| Testable | Untestable |
|----------|------------|
| Dependencies injected | Dependencies created internally |
| Uses interfaces | Uses concrete classes |
| Pure logic (no side effects) | Mixed logic and I/O |
| Small, focused methods | Large, complex methods |
| Deterministic | Depends on random/time |

---

## How DI Enables Testing

### The Problem with Hard-Coded Dependencies

```csharp
// UNTESTABLE: Creates its own dependencies
public class DamageCalculator
{
    public int CalculateDamage(Player attacker, Enemy target)
    {
        // Can't test without real Player and Enemy objects!
        int baseDamage = attacker.GetComponent<Stats>().Attack;  // Needs Unity
        int defense = target.GetComponent<Stats>().Defense;       // Needs Unity
        
        // Can't test random behavior deterministically
        if (Random.value < 0.1f)  // 10% crit chance
        {
            baseDamage *= 2;
        }
        
        return Mathf.Max(1, baseDamage - defense);
    }
}
```

### The Solution: Inject Dependencies

```csharp
// TESTABLE: Dependencies are injected
public interface IRandomProvider
{
    float Value { get; }
}

public class DamageCalculator
{
    private readonly IRandomProvider m_Random;
    
    // Constructor injection
    public DamageCalculator(IRandomProvider random)
    {
        m_Random = random;
    }
    
    public int CalculateDamage(int attackPower, int defense, float critChance)
    {
        int baseDamage = attackPower;
        
        if (m_Random.Value < critChance)
        {
            baseDamage *= 2;
        }
        
        return Mathf.Max(1, baseDamage - defense);
    }
}

// In production:
var calculator = new DamageCalculator(new UnityRandomProvider());

// In tests:
var mockRandom = new MockRandomProvider(0.5f);  // Always returns 0.5
var calculator = new DamageCalculator(mockRandom);
```

### Boss Room's DI Pattern

```csharp
// Boss Room injects publishers/subscribers
public class HostingState : ConnectionState
{
    [Inject] protected IPublisher<ConnectionEventMessage> m_ConnectionEventPublisher;
    
    // In tests, inject a mock publisher that records what was published
}
```

---

## Creating Mock Implementations

### What is a Mock?

A **mock** is a fake implementation of an interface used for testing.

### Simple Mock Example

```csharp
// The interface
public interface ISaveSystem
{
    void Save(string key, object data);
    T Load<T>(string key);
    bool HasKey(string key);
}

// Production implementation
public class PlayerPrefsSaveSystem : ISaveSystem
{
    public void Save(string key, object data) 
        => PlayerPrefs.SetString(key, JsonUtility.ToJson(data));
    
    public T Load<T>(string key) 
        => JsonUtility.FromJson<T>(PlayerPrefs.GetString(key));
    
    public bool HasKey(string key) 
        => PlayerPrefs.HasKey(key);
}

// Mock for testing
public class MockSaveSystem : ISaveSystem
{
    public Dictionary<string, object> SavedData = new Dictionary<string, object>();
    public int SaveCallCount = 0;
    
    public void Save(string key, object data)
    {
        SavedData[key] = data;
        SaveCallCount++;
    }
    
    public T Load<T>(string key) 
        => (T)SavedData[key];
    
    public bool HasKey(string key) 
        => SavedData.ContainsKey(key);
}
```

### Mock MessageChannel for Testing

```csharp
// Mock for Boss Room's PubSub
public class MockPublisher<T> : IPublisher<T>
{
    public List<T> PublishedMessages = new List<T>();
    
    public void Publish(T message)
    {
        PublishedMessages.Add(message);
    }
    
    // Test helper
    public void AssertPublished(T expected)
    {
        Assert.Contains(expected, PublishedMessages);
    }
    
    public void AssertNothingPublished()
    {
        Assert.IsEmpty(PublishedMessages);
    }
}

// Usage in test:
[Test]
public void HostingState_OnClientConnected_PublishesEvent()
{
    // Arrange
    var mockPublisher = new MockPublisher<ConnectionEventMessage>();
    var hostingState = new HostingState();
    // Inject mock (using reflection or test container)
    
    // Act
    hostingState.OnClientConnected(clientId: 123);
    
    // Assert
    Assert.AreEqual(1, mockPublisher.PublishedMessages.Count);
    Assert.AreEqual(123, mockPublisher.PublishedMessages[0].ClientId);
}
```

---

## Unit Testing Patterns

### The AAA Pattern

Every test should follow **Arrange-Act-Assert**:

```csharp
[Test]
public void ActionConfig_CalculateCooldown_ReturnsCorrectValue()
{
    // ARRANGE - Set up the test
    var config = ScriptableObject.CreateInstance<ActionConfig>();
    config.ReuseTimeSeconds = 2.5f;
    
    // ACT - Perform the action
    float cooldown = config.ReuseTimeSeconds;
    
    // ASSERT - Verify the result
    Assert.AreEqual(2.5f, cooldown, 0.001f);
}
```

### Testing State Machines

```csharp
[Test]
public void ConnectionStateMachine_StartHost_TransitionsToStartingHost()
{
    // Arrange
    var connectionManager = CreateTestConnectionManager();
    Assert.IsInstanceOf<OfflineState>(connectionManager.CurrentState);
    
    // Act
    connectionManager.CurrentState.StartHostSession("TestPlayer");
    
    // Assert
    Assert.IsInstanceOf<StartingHostState>(connectionManager.CurrentState);
}

[Test]
public void OfflineState_Enter_LoadsMainMenu()
{
    // Arrange
    var mockSceneLoader = new MockSceneLoader();
    var offlineState = new OfflineState(mockSceneLoader);
    
    // Act
    offlineState.Enter();
    
    // Assert
    Assert.AreEqual("MainMenu", mockSceneLoader.LastLoadedScene);
}
```

### Testing Actions

```csharp
[Test]
public void MeleeAction_OnUpdate_DealsDamageAtExecTime()
{
    // Arrange
    var config = CreateMeleeConfig(execTimeSeconds: 0.4f, damage: 25);
    var action = new MeleeAction();
    action.Initialize(config);
    
    var mockTarget = new MockDamageable();
    
    // Act - Simulate time passing
    action.OnStart(mockCharacter);
    
    // Before exec time - no damage
    SimulateTime(0.3f);
    action.OnUpdate(mockCharacter);
    Assert.AreEqual(0, mockTarget.DamageReceived);
    
    // After exec time - damage dealt
    SimulateTime(0.5f);
    action.OnUpdate(mockCharacter);
    Assert.AreEqual(25, mockTarget.DamageReceived);
}
```

---

## Integration Testing Approaches

### What is Integration Testing?

**Unit tests** test a single class in isolation.
**Integration tests** test multiple classes working together.

### Testing Networked Code

```csharp
// Integration test for connection flow
[UnityTest]
public IEnumerator ClientCanConnectToHost()
{
    // Arrange
    var host = CreateTestHost();
    var client = CreateTestClient();
    
    // Act - Host starts
    host.StartHost();
    yield return new WaitUntil(() => host.IsListening);
    
    // Client connects
    client.StartClient(host.Address);
    yield return new WaitUntil(() => client.IsConnected || timeout);
    
    // Assert
    Assert.IsTrue(client.IsConnected);
    Assert.AreEqual(2, host.ConnectedClientCount);  // Host + Client
}
```

### Testing with PlayMode Tests

Unity PlayMode tests run in an actual game environment:

```csharp
[UnityTest]
public IEnumerator Player_TakesDamage_HealthDecreases()
{
    // Arrange - Spawn a player
    var playerPrefab = Resources.Load<GameObject>("Player");
    var player = Object.Instantiate(playerPrefab);
    var health = player.GetComponent<ServerCharacter>();
    
    int initialHealth = health.HitPoints;
    
    yield return null;  // Wait one frame
    
    // Act
    health.ReceiveHP(null, -10);
    
    yield return null;
    
    // Assert
    Assert.AreEqual(initialHealth - 10, health.HitPoints);
    
    // Cleanup
    Object.Destroy(player);
}
```

---

## Debug Logging Patterns

### Structured Logging

```csharp
// BAD: Unstructured logs
Debug.Log("Something happened");
Debug.Log("Player did something with " + someValue);

// GOOD: Structured, contextual logs
[System.Diagnostics.Conditional("UNITY_EDITOR")]
public static class GameLog
{
    public static void Connection(string message)
    {
        Debug.Log($"[CONNECTION] {message}");
    }
    
    public static void Action(string actionName, ulong characterId, string details = "")
    {
        Debug.Log($"[ACTION] {actionName} by {characterId}: {details}");
    }
    
    public static void State(string from, string to, string reason = "")
    {
        Debug.Log($"[STATE] {from} → {to} {reason}");
    }
}

// Usage
GameLog.Connection($"Client {clientId} connected");
GameLog.Action("MeleeAction", characterId, $"target={targetId}");
GameLog.State("Offline", "StartingHost", "User clicked Host button");
```

### Conditional Compilation

```csharp
// Only compile logging in Editor/Debug builds
public class VerboseLogger
{
    [System.Diagnostics.Conditional("DEBUG")]
    public static void Log(string message)
    {
        Debug.Log(message);
    }
    
    [System.Diagnostics.Conditional("DEBUG")]
    public static void LogWarning(string message)
    {
        Debug.LogWarning(message);
    }
}

// In release builds, these calls are completely removed
VerboseLogger.Log("This won't exist in release builds");
```

### Boss Room's Logging Pattern

```csharp
// Boss Room logs state transitions
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

## Debugging Complex Systems

### 1. State Machine Debugging

When state machines misbehave:

```csharp
// Add logging to every transition
public void ChangeState(State newState)
{
    Debug.Log($"[StateMachine] Transition: {currentState?.Name} → {newState.Name}");
    Debug.Log($"[StateMachine] Stack trace:\n{Environment.StackTrace}");
    
    currentState?.Exit();
    currentState = newState;
    currentState.Enter();
}
```

### 2. Event/PubSub Debugging

When events aren't received:

```csharp
public class DebugMessageChannel<T> : MessageChannel<T>
{
    public override void Publish(T message)
    {
        Debug.Log($"[PubSub] Publishing {typeof(T).Name}: {message}");
        Debug.Log($"[PubSub] Subscribers: {m_MessageHandlers.Count}");
        
        base.Publish(message);
    }
    
    public override IDisposable Subscribe(Action<T> handler)
    {
        Debug.Log($"[PubSub] New subscriber for {typeof(T).Name}: {handler.Method.Name}");
        return base.Subscribe(handler);
    }
}
```

### 3. Network Debugging

When networked data seems wrong:

```csharp
// Log NetworkVariable changes
public class DebuggableNetworkVariable<T> : NetworkVariable<T>
{
    public override T Value
    {
        get => base.Value;
        set
        {
            Debug.Log($"[Network] {Name}: {base.Value} → {value}");
            base.Value = value;
        }
    }
}
```

### 4. Action System Debugging

```csharp
public override bool OnStart(ServerCharacter parent)
{
    Debug.Log($"[Action] {GetType().Name}.OnStart - Character: {parent.NetworkObjectId}");
    Debug.Log($"[Action] Config: Duration={Config.DurationSeconds}, ExecTime={Config.ExecTimeSeconds}");
    Debug.Log($"[Action] Data: TargetId={Data.TargetIds?.FirstOrDefault()}");
    
    // ... actual logic ...
    
    return result;
}
```

---

## Performance Profiling Workflow

### Step 1: Identify the Symptom

- Frame rate drops?
- Memory growing?
- Hitches/stutters?
- Load time too long?

### Step 2: Profile (Don't Guess!)

```
Unity Profiler (Window → Analysis → Profiler)
├── CPU Usage → Find slow methods
├── GPU Usage → Find rendering issues
├── Memory → Find allocations
└── Network → Find bandwidth issues
```

### Step 3: Common Culprits

| Symptom | Likely Cause | How to Check |
|---------|--------------|--------------|
| Regular stutters | Garbage collection | Memory profiler, look for GC.Alloc |
| Slow loading | Synchronous loading | CPU profiler during load |
| Frame drops | Too much in Update() | CPU profiler, sort by time |
| Memory growth | Leaking references | Memory snapshot comparison |

### Step 4: Boss Room's Performance Patterns

```csharp
// Pattern 1: Object Pooling (avoids GC)
var projectile = NetworkObjectPool.Singleton.GetNetworkObject(prefab, pos, rot);

// Pattern 2: Cached components (avoids GetComponent)
private Health m_CachedHealth;
void Awake() => m_CachedHealth = GetComponent<Health>();

// Pattern 3: Pre-created states (avoids allocation during play)
internal readonly OfflineState m_Offline = new OfflineState();

// Pattern 4: StringToHash for animations (avoids string comparison)
private static readonly int AttackTrigger = Animator.StringToHash("Attack");
```

---

## Key Takeaways

1. **DI enables testing** - Inject dependencies to swap with mocks
2. **Interfaces enable mocking** - Code to interfaces, not implementations
3. **Test behavior, not implementation** - What it does, not how
4. **Log strategically** - Categories, levels, conditionals
5. **Profile before optimizing** - Measure, don't guess
6. **Use Unity's tools** - Profiler, Frame Debugger, Memory Profiler

---

## Testing Checklist

For any new feature:

- [ ] Can I test this without running the game?
- [ ] Are dependencies injectable?
- [ ] Are there interfaces I can mock?
- [ ] Have I written at least one happy-path test?
- [ ] Have I tested edge cases?
- [ ] Is there logging for debugging?

---

> **Next:** [16_performance_patterns.md](./16_performance_patterns.md) — Memory, GC, and efficiency patterns
