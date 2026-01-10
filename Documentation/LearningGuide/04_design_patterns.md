# 04: Design Patterns Catalog

> **Goal:** Master the design patterns used in Boss Room with detailed code examples, real file references, and practical exercises.

> 📍 **Key Implementation Files:**
> - [ConnectionState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/ConnectionState.cs) — State Machine base
> - [MessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs) — PubSub/Observer
> - [ActionFactory.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ActionFactory.cs) — Factory + Object Pool
> - [NetworkObjectPool.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/NetworkObjectPool.cs) — Object Pool
> - [ConnectionMethod.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionMethod.cs) — Strategy Pattern

---

## Table of Contents

1. [State Machine](#pattern-1-state-machine)
2. [Observer (PubSub)](#pattern-2-observer-pubsub)
3. [Factory](#pattern-3-factory)
4. [Object Pool](#pattern-4-object-pool)
5. [Singleton](#pattern-5-singleton)
6. [Command](#pattern-6-command)
7. [Strategy](#pattern-7-strategy)
8. [Mediator](#pattern-8-mediator)
9. [When NOT to Use Patterns](#when-not-to-use-patterns)
10. [Pattern Decision Guide](#pattern-decision-guide)
11. [Exercises with Answers](#exercises-with-answers)

---

## Pattern 1: State Machine

### What It Solves
Object behavior depends on its current state, and you're drowning in if-else spaghetti.

### ❌ Without State Machine
```csharp
// BAD: Unmaintainable mess
void OnDisconnect() {
    if (isHost && wasInGame && !userRequestedShutdown) {
        // cleanup server
    } else if (isClient && wasConnected && shouldReconnect) {
        // try reconnect
    } else if (isConnecting && !isHost) {
        // show error
    }
    // ... more and more conditions ...
}
```

### ✅ With State Machine
```csharp
// GOOD: Each state handles its own logic
public class ClientReconnectingState : ConnectionState
{
    public override void OnClientDisconnect(ulong clientId)
    {
        if (m_NbAttempts < k_MaxReconnectAttempts)
        {
            TryReconnect();
        }
        else
        {
            m_ConnectionManager.ChangeState(m_ConnectionManager.m_Offline);
        }
    }
}
```

### Boss Room Implementation

**File:** `Assets/Scripts/ConnectionManagement/ConnectionState/ConnectionState.cs` (lines 12-44)
```csharp
abstract class ConnectionState
{
    [Inject] protected ConnectionManager m_ConnectionManager;
    [Inject] protected IPublisher<ConnectStatus> m_ConnectStatusPublisher;
    
    // Lifecycle
    public abstract void Enter();
    public abstract void Exit();
    
    // Events - override only what you care about
    public virtual void OnClientConnected(ulong clientId) { }
    public virtual void OnClientDisconnect(ulong clientId) { }
    public virtual void OnServerStarted() { }
    public virtual void StartClientIP(string playerName, string ip, int port) { }
    public virtual void StartHostIP(string playerName, string ip, int port) { }
    public virtual void OnUserRequestedShutdown() { }
    public virtual void ApprovalCheck(...) { }
}
```

**File:** `Assets/Scripts/ConnectionManagement/ConnectionManager.cs` (lines 74-79, 112-122)
```csharp
// Pre-created states (no allocation during gameplay)
internal readonly OfflineState m_Offline = new OfflineState();
internal readonly ClientConnectingState m_ClientConnecting = new ClientConnectingState();
internal readonly ClientConnectedState m_ClientConnected = new ClientConnectedState();
internal readonly ClientReconnectingState m_ClientReconnecting = new ClientReconnectingState();
internal readonly StartingHostState m_StartingHost = new StartingHostState();
internal readonly HostingState m_Hosting = new HostingState();

// The ONLY way states change
internal void ChangeState(ConnectionState nextState)
{
    Debug.Log($"Changed state from {m_CurrentState} to {nextState}");
    m_CurrentState?.Exit();
    m_CurrentState = nextState;
    m_CurrentState.Enter();
}
```

### When to Use
✅ Player states (Idle, Walking, Jumping, Attacking, Dead)
✅ AI states (Patrol, Chase, Attack, Flee)
✅ Game phases (Menu, Loading, Playing, Paused, GameOver)
✅ Connection states (Offline, Connecting, Connected)
✅ UI screen flows

### When NOT to Use
❌ Only 2 simple states (use bool)
❌ States don't have different behavior (use enum)
❌ Overkill for simple toggles

> 📚 **Deep Dive:** [10_connection_state_machine.md](./10_connection_state_machine.md)

---

## Pattern 2: Observer (PubSub)

### What It Solves
Multiple objects need to react to events without tight coupling.

### ❌ Without Observer
```csharp
// BAD: Health knows TOO MUCH
class Health {
    void TakeDamage(int amount) {
        currentHealth -= amount;
        
        // Health must know about ALL these!
        healthBar.UpdateDisplay(currentHealth);
        soundManager.PlayHurtSound();
        achievementSystem.TrackDamageTaken(amount);
        cameraShake.Shake(0.5f);
        analyticsSystem.Log("damage_taken", amount);
        // ... adding more means modifying Health ...
    }
}
```

### ✅ With Observer
```csharp
// GOOD: Health only manages health, publishes event
class Health {
    [Inject] IPublisher<HealthChangedMsg> publisher;
    
    void TakeDamage(int amount) {
        currentHealth -= amount;
        publisher.Publish(new HealthChangedMsg { HP = currentHealth });
        // Don't know, don't care who listens!
    }
}

// Anyone subscribes independently
class HealthBar {
    [Inject] ISubscriber<HealthChangedMsg> subscriber;
    void Start() => subscriber.Subscribe(OnHealthChanged);
    void OnHealthChanged(HealthChangedMsg msg) => UpdateUI(msg.HP);
}
```

### Boss Room Implementation

**File:** `Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs` (lines 28-75)
```csharp
public class MessageChannel<T> : IMessageChannel<T>
{
    readonly HashSet<Action<T>> m_MessageHandlers = new();
    readonly Dictionary<Action<T>, bool> m_PendingHandlers = new();  // Deferred changes
    
    public void Publish(T message)
    {
        // 1. Apply pending changes first (safe modification)
        foreach (var pending in m_PendingHandlers)
        {
            if (pending.Value)
                m_MessageHandlers.Add(pending.Key);
            else
                m_MessageHandlers.Remove(pending.Key);
        }
        m_PendingHandlers.Clear();
        
        // 2. Now safe to invoke
        foreach (var handler in m_MessageHandlers)
        {
            handler?.Invoke(message);
        }
    }
    
    public IDisposable Subscribe(Action<T> handler)
    {
        m_PendingHandlers[handler] = true;  // Deferred add
        return new DisposableSubscription<T>(this, handler);
    }
}
```

**Why pending handlers?** If a handler unsubscribes during Publish(), modifying the collection would crash!

### Real Usage Example
```csharp
// In HostingState.cs - publishes connection events
m_ConnectionEventPublisher.Publish(new ConnectionEventMessage {
    ClientId = clientId,
    EventType = ConnectionEventType.ClientConnected
});

// In ServerBossRoomState.cs - subscribes to life state changes
m_LifeStateSubscriber.Subscribe(OnLifeStateChanged);

void OnLifeStateChanged(LifeStateChangedEventMessage msg)
{
    if (msg.LifeState == LifeState.Dead && msg.CharacterType == CharacterTypeEnum.ImpBoss)
    {
        BossDefeated();
    }
}
```

### When to Use
✅ UI updates from game events
✅ Achievement/analytics systems
✅ Sound triggers
✅ Multiple systems react to same event
✅ Decoupling modules

### When NOT to Use
❌ Only one listener (direct call is simpler)
❌ Synchronous request/response (just call the method)
❌ Performance-critical hot paths (slight overhead)

> 📚 **Deep Dive:** [11_infrastructure_patterns.md](./11_infrastructure_patterns.md)

---

## Pattern 3: Factory

### What It Solves
Object creation is complex, needs pooling, or should be abstracted.

### ❌ Without Factory
```csharp
// BAD: Creation logic scattered everywhere
void SpawnEnemy() {
    var prefab = Resources.Load<GameObject>("Goblin");
    var enemy = Instantiate(prefab, spawnPoint, Quaternion.identity);
    enemy.GetComponent<Health>().SetMax(100);
    enemy.GetComponent<AI>().Initialize(patrolRoute);
    // Duplicated everywhere goblins spawn
}
```

### ✅ With Factory
```csharp
// GOOD: Centralized creation
public class EnemyFactory {
    public Enemy CreateEnemy(EnemyType type, Vector3 position) {
        var prefab = GetPrefab(type);
        var enemy = Instantiate(prefab, position, Quaternion.identity);
        InitializeEnemy(enemy, type);
        return enemy;
    }
}

// Usage - clean and consistent
var goblin = enemyFactory.CreateEnemy(EnemyType.Goblin, spawnPoint);
```

### Boss Room Implementation

**File:** `Assets/Scripts/Gameplay/Action/ActionFactory.cs`
```csharp
public class ActionFactory
{
    // Pool of action instances (avoids GC)
    private Dictionary<ActionID, Stack<Action>> m_ActionPool = new();
    
    public Action CreateActionFromData(ref ActionRequestData data)
    {
        var prototype = GameDataSource.Instance.GetActionPrototypeByID(data.ActionID);
        
        // Try to get from pool
        if (m_ActionPool.TryGetValue(data.ActionID, out var pool) && pool.Count > 0)
        {
            var action = pool.Pop();
            action.Initialize(prototype.Config, data);
            return action;
        }
        
        // Create new if pool empty
        var newAction = Activator.CreateInstance(prototype.ActionType) as Action;
        newAction.Initialize(prototype.Config, data);
        return newAction;
    }
    
    public void ReturnAction(Action action)
    {
        action.Reset();
        if (!m_ActionPool.ContainsKey(action.ActionID))
            m_ActionPool[action.ActionID] = new Stack<Action>();
        m_ActionPool[action.ActionID].Push(action);
    }
}
```

### When to Use
✅ Complex object creation logic
✅ Need to hide concrete types
✅ Combined with object pooling
✅ Multiple similar object types

### When NOT to Use
❌ Simple `new MyClass()` is enough
❌ Object has no special initialization
❌ Only one type ever created

---

## Pattern 4: Object Pool

### What It Solves
Frequent Instantiate/Destroy causes garbage collection spikes.

### ❌ Without Pool
```csharp
// BAD: GC spike every bullet
void Fire() {
    var bullet = Instantiate(bulletPrefab);  // Allocates memory
    StartCoroutine(DestroyAfter(bullet, 3f));
}

IEnumerator DestroyAfter(GameObject obj, float delay) {
    yield return new WaitForSeconds(delay);
    Destroy(obj);  // GC will collect later (spike!)
}
```

### ✅ With Pool
```csharp
// GOOD: Reuse bullets
void Fire() {
    var bullet = bulletPool.Get();  // Fast - no allocation
    bullet.transform.position = firePoint.position;
    StartCoroutine(ReturnAfter(bullet, 3f));
}

IEnumerator ReturnAfter(Bullet bullet, float delay) {
    yield return new WaitForSeconds(delay);
    bulletPool.Return(bullet);  // Back to pool, no GC
}
```

### Boss Room Implementation

**File:** `Assets/Scripts/Infrastructure/NetworkObjectPool.cs` (lines 45-95)
```csharp
public class NetworkObjectPool : MonoBehaviour
{
    [SerializeField] List<PoolConfigObject> PooledPrefabsList;
    
    Dictionary<GameObject, Queue<NetworkObject>> m_PooledObjects = new();
    
    // Called at startup - prewarm pools
    void InitializePool()
    {
        foreach (var configObject in PooledPrefabsList)
        {
            RegisterPrefab(configObject.Prefab, configObject.PrewarmCount);
        }
    }
    
    public NetworkObject GetNetworkObject(GameObject prefab, Vector3 pos, Quaternion rot)
    {
        var queue = m_PooledObjects[prefab];
        
        if (queue.Count > 0)
        {
            // Reuse from pool
            var obj = queue.Dequeue();
            obj.transform.SetPositionAndRotation(pos, rot);
            obj.gameObject.SetActive(true);
            return obj;
        }
        
        // Pool empty - create new (fallback)
        return Instantiate(prefab, pos, rot).GetComponent<NetworkObject>();
    }
    
    public void ReturnNetworkObject(NetworkObject obj, GameObject prefab)
    {
        obj.gameObject.SetActive(false);
        m_PooledObjects[prefab].Enqueue(obj);
    }
}
```

### When to Use
✅ Bullets/projectiles
✅ Particle effects
✅ Damage numbers/floating text
✅ Spawned enemies
✅ Any frequently created objects

### When NOT to Use
❌ Objects created rarely (once per level)
❌ Objects with unique state that can't be reset
❌ Pool management overhead exceeds creation cost

> 📚 **Deep Dive:** [16_performance_patterns.md](./16_performance_patterns.md)

---

## Pattern 5: Singleton (Use Sparingly!)

### What It Solves
Need exactly one instance accessible globally.

### ⚠️ Warning
Singletons are often **overused**. They create tight coupling and make testing difficult.

### ❌ Overused Singleton
```csharp
// BAD: Everything becomes a singleton
GameManager.Instance.DoThing();
AudioManager.Instance.PlaySound();
UIManager.Instance.ShowScreen();
SaveManager.Instance.Save();
// Now everything is coupled to everything!
```

### ✅ Boss Room's Better Approach: Dependency Injection
```csharp
// GOOD: Inject dependencies instead
public class SomeComponent : MonoBehaviour
{
    [Inject] private IAudioService m_Audio;
    [Inject] private ISaveService m_Save;
    
    // Dependencies are explicit, testable, swappable
}
```

### When Singleton IS Acceptable
✅ Truly stateless utilities (math helpers)
✅ Single access point required by framework (Unity's NetworkManager)
✅ When DI is overkill for small project

### When to Avoid (Use DI Instead)
❌ Most game systems (use DI)
❌ When you need to swap implementations
❌ When testing matters

---

## Pattern 6: Command

### What It Solves
Encapsulate actions as objects for queuing, undo/redo, or replay.

### Boss Room's Action System as Command Pattern

**File:** `Assets/Scripts/Gameplay/Action/Action.cs`
```csharp
public abstract class Action
{
    // Command data
    public ActionConfig Config;
    public ActionRequestData Data;
    
    // Execute
    public virtual bool OnStart(ServerCharacter parent) => true;
    public virtual bool OnUpdate(ServerCharacter parent) => TimeRunning < Config.ExecTimeSeconds;
    public virtual void OnEnd(ServerCharacter parent) { }
    
    // Undo/Cancel
    public virtual void Cancel(ServerCharacter parent) { }
}
```

**File:** `Assets/Scripts/Gameplay/Action/ActionPlayers/ServerActionPlayer.cs`
```csharp
public class ServerActionPlayer
{
    private List<Action> m_Queue;  // Command queue!
    
    public void PlayAction(ref ActionRequestData data)
    {
        Action action = ActionFactory.CreateActionFromData(ref data);
        m_Queue.Add(action);  // Queue the command
        
        if (m_Queue.Count == 1)
            StartAction();  // Execute immediately if first
    }
}
```

### When to Use
✅ Undo/redo systems
✅ Input buffering (fighting games)
✅ Action queues (RPGs, strategy games)
✅ Replay systems
✅ Transaction logging

### When NOT to Use
❌ Simple fire-and-forget operations
❌ When undo isn't needed
❌ Performance-critical code (object overhead)

---

## Pattern 7: Strategy

### What It Solves
Need to swap algorithms at runtime without changing the code that uses them.

### Boss Room Implementation

**File:** `Assets/Scripts/ConnectionManagement/ConnectionMethod.cs`
```csharp
// Strategy interface
public abstract class ConnectionMethod
{
    public abstract Task SetupHostConnectionAsync();
    public abstract Task SetupClientConnectionAsync();
    public abstract Task SetupClientReconnectionAsync();
}

// Concrete strategies
public class IPConnectionMethod : ConnectionMethod
{
    public override async Task SetupHostConnectionAsync()
    {
        // Direct IP hosting
        NetworkManager.Singleton.StartHost();
    }
}

public class UnityRelayConnectionMethod : ConnectionMethod
{
    public override async Task SetupHostConnectionAsync()
    {
        // Use Unity Relay service
        var allocation = await RelayService.CreateAllocationAsync(maxPlayers);
        NetworkManager.Singleton.StartHost();
    }
}
```

**Usage:** (in ConnectionState files)
```csharp
// The connection method can be swapped at runtime
m_ConnectionMethod = useRelay 
    ? new UnityRelayConnectionMethod() 
    : new IPConnectionMethod();

// Code doesn't care which strategy is used
await m_ConnectionMethod.SetupHostConnectionAsync();
```

### When to Use
✅ Multiple algorithms for same task
✅ Need to swap behavior at runtime
✅ Avoid complex conditionals
✅ Different behaviors per game mode/platform

### When NOT to Use
❌ Only one algorithm exists
❌ Algorithms never change
❌ Simple if/else is clearer

---

## Pattern 8: Mediator

### What It Solves
Complex many-to-many communication simplified through a central coordinator.

### Boss Room Implementation

**File:** `Assets/Scripts/Gameplay/UI/Lobby/IPUIMediator.cs`
```csharp
public class IPUIMediator : MonoBehaviour
{
    [SerializeField] InputField m_IPInputField;
    [SerializeField] InputField m_PortInputField;
    [SerializeField] InputField m_PlayerNameField;
    [SerializeField] Button m_HostButton;
    [SerializeField] Button m_JoinButton;
    
    [Inject] ConnectionManager m_ConnectionManager;
    
    void Start()
    {
        // Mediator coordinates between UI elements and game systems
        m_HostButton.onClick.AddListener(OnHostClicked);
        m_JoinButton.onClick.AddListener(OnJoinClicked);
    }
    
    void OnHostClicked()
    {
        // Mediator validates and coordinates
        if (ValidateInputs())
        {
            m_ConnectionManager.StartHostIP(
                m_PlayerNameField.text,
                m_IPInputField.text,
                int.Parse(m_PortInputField.text));
        }
    }
}
```

### When to Use
✅ Complex UI with many interdependent elements
✅ Dialog systems (choices affect other UI)
✅ Form validation with multiple fields
✅ Reduce component coupling

### When NOT to Use
❌ Simple UI with few elements
❌ Creates a "god class" mediator
❌ When direct references are clearer

---

## When NOT to Use Patterns

Patterns are tools, not rules. Overuse creates complexity.

| Don't Use | When |
|-----------|------|
| State Machine | Only 2 states (use bool) |
| Observer | Single listener (call directly) |
| Factory | Simple `new MyClass()` works |
| Object Pool | Objects created rarely |
| Singleton | DI is available |
| Command | No undo/queue needed |
| Strategy | Only one algorithm |
| Mediator | Simple relationships |

### Signs of Over-Engineering
- Pattern adds more code than it saves
- You're the only one who understands it
- Simple tasks require navigating many files
- "It might be useful someday" reasoning

---

## Pattern Decision Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WHICH PATTERN DO I NEED?                          │
└─────────────────────────────────────────────────────────────────────┘

"I need to..."

├── React to events without knowing who fires them
│   └── OBSERVER / PUBSUB
│
├── Manage complex state with transitions
│   └── STATE MACHINE
│
├── Create objects with complex setup
│   └── FACTORY
│
├── Avoid GC from frequent create/destroy
│   └── OBJECT POOL
│
├── Queue actions for later / undo
│   └── COMMAND
│
├── Swap algorithms at runtime
│   └── STRATEGY
│
├── Coordinate many UI elements
│   └── MEDIATOR
│
└── Have global access (think twice!)
    └── SINGLETON (or better: DI)
```

---

## Exercises with Answers

### Exercise 1: Identify the Pattern

For each scenario, which pattern fits best?

**Q1:** Player has states: Idle, Walking, Jumping, Climbing, Swimming
> **A:** State Machine - multiple states with different behaviors

**Q2:** Health bar, damage numbers, and achievement system all need to know when damage happens
> **A:** Observer/PubSub - multiple independent listeners for one event

**Q3:** Firing 100 bullets per second in a bullet-hell game
> **A:** Object Pool - frequent create/destroy would cause GC spikes

**Q4:** Turn-based game needs to queue player actions and undo last move
> **A:** Command - encapsulates actions for queue and undo

**Q5:** Game can connect via LAN, Internet Relay, or Dedicated Server
> **A:** Strategy - multiple algorithms for same task (connection)

**Q6:** Character creation screen with name field, class selector, and start button
> **A:** Mediator - coordinates multiple UI elements

### Exercise 2: Find in Code

Using your IDE's search, find these patterns in Boss Room:

1. **State Machine:** Search `ChangeState` - find in `ConnectionManager.cs`
2. **Observer:** Search `IPublisher<` - find publishers throughout
3. **Factory:** Search `ActionFactory` - find action creation
4. **Pool:** Search `NetworkObjectPool` - find the pool implementation
5. **Strategy:** Search `ConnectionMethod` - find IP vs Relay strategies

### Exercise 3: Apply a Pattern

**Scenario:** You're building an inventory system. Items can be added, removed, and moved. The UI, save system, and quest tracker all need to know when inventory changes.

**What pattern(s) would you use?**

> **Answer:**
> 1. **Observer/PubSub** - Inventory publishes `InventoryChangedMessage`, UI/save/quest subscribe
> 2. **Command** (optional) - If you need undo, encapsulate add/remove/move as commands
> 3. **Factory** (optional) - If items have complex creation logic

---

## Key Takeaways

1. **Patterns solve specific problems** - Don't use them just because they exist
2. **Boss Room demonstrates real usage** - Study the actual files
3. **DI > Singleton** - Boss Room uses VContainer, almost no singletons
4. **Pool high-frequency objects** - Actions, projectiles, effects
5. **Combine patterns** - Factory + Pool, State Machine + Observer

---

> **Related Guides:**
> - [10_connection_state_machine.md](./10_connection_state_machine.md) - State Machine deep dive
> - [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - PubSub implementation
> - [09_action_system_deepdive.md](./09_action_system_deepdive.md) - Command pattern in actions
> - [16_performance_patterns.md](./16_performance_patterns.md) - Object pooling details
