# 11: Infrastructure Patterns Deep Dive

> **Purpose:** A comprehensive guide to the reusable infrastructure systems that power Boss Room. These patterns are useful in ANY game—multiplayer or single-player.

---

## Table of Contents

1. [Infrastructure Overview](#infrastructure-overview)
2. [PubSub Message System](#pubsub-message-system)
3. [NetworkObjectPool](#networkobjectpool)
4. [DisposableGroup Pattern](#disposablegroup-pattern)
5. [UpdateRunner for Non-MonoBehaviours](#updaterunner-for-non-monobehaviours)
6. [ScriptableObject Architecture](#scriptableobject-architecture)
7. [Key Takeaways](#key-takeaways)

---

## Infrastructure Overview

The Infrastructure layer contains **reusable utilities** that aren't specific to Boss Room's gameplay. You could copy these into any project.

```
Assets/Scripts/Infrastructure/
├── PubSub/                         ← Event messaging system
│   ├── IMessageChannel.cs          ← Interfaces
│   ├── MessageChannel.cs           ← Basic pub/sub
│   ├── BufferedMessageChannel.cs   ← Remembers last message
│   ├── NetworkedMessageChannel.cs  ← Server broadcasts to clients
│   └── DisposableSubscription.cs   ← Auto-cleanup subscriptions
├── NetworkObjectPool.cs            ← Object pooling for Netcode
├── DisposableGroup.cs              ← Group multiple disposables
├── UpdateRunner.cs                 ← Provides Update() to non-MonoBehaviours
└── ScriptableObjectArchitecture/   ← Data-driven events and sets
```

---

## PubSub Message System

The Publish-Subscribe pattern decouples senders from receivers. Publishers don't know who listens; subscribers don't know who publishes.

### Why PubSub?

```csharp
// WITHOUT PubSub (tight coupling)
class HealthSystem {
    void TakeDamage(int amount) {
        health -= amount;
        
        // HealthSystem must KNOW about all these classes!
        healthUI.UpdateBar(health);
        audioManager.PlayHitSound();
        achievementSystem.CheckDamageAchievement(amount);
        analyticsSystem.TrackDamage(amount);
    }
}

// WITH PubSub (loose coupling)
class HealthSystem {
    void TakeDamage(int amount) {
        health -= amount;
        
        // Just publish the event - don't care who listens
        healthChannel.Publish(new HealthChangedMessage { health = health });
    }
}

// Anyone can subscribe independently
class HealthUI {
    void Start() => healthChannel.Subscribe(msg => UpdateBar(msg.health));
}
class AudioManager {
    void Start() => healthChannel.Subscribe(_ => PlayHitSound());
}
```

### The Interface Hierarchy

**File:** [IMessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/IMessageChannel.cs)

```csharp
namespace Unity.BossRoom.Infrastructure
{
    // Publisher interface - can send messages
    public interface IPublisher<T>
    {
        void Publish(T message);
    }
    
    // Subscriber interface - can receive messages
    public interface ISubscriber<T>
    {
        IDisposable Subscribe(Action<T> handler);
        void Unsubscribe(Action<T> handler);
    }
    
    // Full channel - can do both
    public interface IMessageChannel<T> : IPublisher<T>, ISubscriber<T>, IDisposable
    {
        bool IsDisposed { get; }
    }
    
    // Buffered channel - remembers last message for late subscribers
    public interface IBufferedMessageChannel<T> : IMessageChannel<T>
    {
        bool HasBufferedMessage { get; }
        T BufferedMessage { get; }
    }
}
```

**Dependency Injection Tip:** Inject `IPublisher<T>` to classes that only publish, `ISubscriber<T>` to classes that only subscribe. This enforces separation of concerns.

### MessageChannel Implementation

**File:** [MessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs)

```csharp
public class MessageChannel<T> : IMessageChannel<T>
{
    // Active handlers
    readonly List<Action<T>> m_MessageHandlers = new List<Action<T>>();
    
    // Pending adds/removes (deferred to avoid modification during iteration)
    readonly Dictionary<Action<T>, bool> m_PendingHandlers = new Dictionary<Action<T>, bool>();
    
    public bool IsDisposed { get; private set; } = false;
    
    public virtual void Publish(T message)
    {
        // 1. Process pending adds/removes FIRST
        foreach (var handler in m_PendingHandlers.Keys)
        {
            if (m_PendingHandlers[handler])
                m_MessageHandlers.Add(handler);     // true = add
            else
                m_MessageHandlers.Remove(handler);  // false = remove
        }
        m_PendingHandlers.Clear();
        
        // 2. Invoke all handlers
        foreach (var messageHandler in m_MessageHandlers)
        {
            if (messageHandler != null)
            {
                messageHandler.Invoke(message);
            }
        }
    }
    
    public virtual IDisposable Subscribe(Action<T> handler)
    {
        // Mark for addition (deferred)
        if (m_PendingHandlers.ContainsKey(handler))
        {
            if (!m_PendingHandlers[handler])  // Was pending remove
                m_PendingHandlers.Remove(handler);  // Cancel the remove
        }
        else
        {
            m_PendingHandlers[handler] = true;  // Mark for add
        }
        
        // Return disposable for auto-cleanup
        return new DisposableSubscription<T>(this, handler);
    }
    
    public void Unsubscribe(Action<T> handler)
    {
        // Mark for removal (deferred)
        if (m_PendingHandlers.ContainsKey(handler))
        {
            if (m_PendingHandlers[handler])  // Was pending add
                m_PendingHandlers.Remove(handler);  // Cancel the add
        }
        else
        {
            m_PendingHandlers[handler] = false;  // Mark for remove
        }
    }
}
```

**Key Insight: Deferred Modification**

The `m_PendingHandlers` dictionary solves a common bug: modifying a collection while iterating it.

```csharp
// BUG: This crashes if handler unsubscribes during iteration
foreach (var handler in handlers)
{
    handler.Invoke(msg);  // Handler calls Unsubscribe → modifies handlers!
}

// FIX: Defer modifications until next Publish()
```

### NetworkedMessageChannel

**File:** [NetworkedMessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/NetworkedMessageChannel.cs)

This extends MessageChannel to broadcast messages from server to all clients.

```csharp
public class NetworkedMessageChannel<T> : MessageChannel<T> 
    where T : unmanaged, INetworkSerializeByMemcpy  // Must be blittable!
{
    NetworkManager m_NetworkManager;
    string m_Name;  // Used as custom message identifier
    
    public override void Publish(T message)
    {
        if (m_NetworkManager.IsServer)
        {
            // 1. Send to all clients via custom named message
            SendMessageThroughNetwork(message);
            
            // 2. Also publish locally (for host)
            base.Publish(message);
        }
        else
        {
            Debug.LogError("Only a server can publish in a NetworkedMessageChannel");
        }
    }
    
    void SendMessageThroughNetwork(T message)
    {
        // Serialize message to buffer
        var writer = new FastBufferWriter(FastBufferWriter.GetWriteSize<T>(), Allocator.Temp);
        writer.WriteValueSafe(message);
        
        // Send to all connected clients
        m_NetworkManager.CustomMessagingManager.SendNamedMessageToAll(m_Name, writer);
    }
    
    void ReceiveMessageThroughNetwork(ulong clientID, FastBufferReader reader)
    {
        // Deserialize and publish locally
        reader.ReadValueSafe(out T message);
        base.Publish(message);  // Call base to invoke local subscribers
    }
}
```

### Usage Example

```csharp
// Define a message type
public struct PlayerJoinedMessage : INetworkSerializeByMemcpy
{
    public FixedString64Bytes PlayerName;
    public int PlayerCount;
}

// In DI setup (ApplicationController)
builder.Register<NetworkedMessageChannel<PlayerJoinedMessage>>(Lifetime.Singleton)
       .AsImplementedInterfaces();

// Server publishes when player joins
[Inject] IPublisher<PlayerJoinedMessage> m_PlayerJoinedPublisher;

void OnPlayerJoined(string name, int count)
{
    m_PlayerJoinedPublisher.Publish(new PlayerJoinedMessage {
        PlayerName = name,
        PlayerCount = count
    });
}

// Any client can subscribe
[Inject] ISubscriber<PlayerJoinedMessage> m_PlayerJoinedSubscriber;
IDisposable m_Subscription;

void Start()
{
    m_Subscription = m_PlayerJoinedSubscriber.Subscribe(OnPlayerJoinedReceived);
}

void OnPlayerJoinedReceived(PlayerJoinedMessage msg)
{
    Debug.Log($"{msg.PlayerName} joined! {msg.PlayerCount} players now.");
}

void OnDestroy()
{
    m_Subscription?.Dispose();  // Always unsubscribe!
}
```

---

## NetworkObjectPool

Object pooling is critical for performance in any game that spawns many objects (bullets, particles, enemies). For networked games, it's even more important because spawning triggers network traffic.

**File:** [NetworkObjectPool.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/NetworkObjectPool.cs)

### Why Pool NetworkObjects?

```
WITHOUT POOLING:
Spawn → Allocate memory → Add to network spawn list → Send spawn message → GC pressure
Despawn → Remove from network → Destroy → Garbage collection spike!

WITH POOLING:
First spawn → Allocate once
Despawn → Deactivate, return to pool
Respawn → Reactivate from pool → No allocation!
```

### Implementation

```csharp
public class NetworkObjectPool : NetworkBehaviour
{
    public static NetworkObjectPool Singleton { get; private set; }
    
    [SerializeField]
    List<PoolConfigObject> PooledPrefabsList;  // Configure in Inspector
    
    // Pool storage
    Dictionary<GameObject, ObjectPool<NetworkObject>> m_PooledObjects 
        = new Dictionary<GameObject, ObjectPool<NetworkObject>>();
    
    public override void OnNetworkSpawn()
    {
        // Register all prefabs with their pools
        foreach (var config in PooledPrefabsList)
        {
            RegisterPrefabInternal(config.Prefab, config.PrewarmCount);
        }
    }
    
    void RegisterPrefabInternal(GameObject prefab, int prewarmCount)
    {
        // Define pool callbacks
        NetworkObject CreateFunc() => Instantiate(prefab).GetComponent<NetworkObject>();
        void ActionOnGet(NetworkObject obj) => obj.gameObject.SetActive(true);
        void ActionOnRelease(NetworkObject obj) => obj.gameObject.SetActive(false);
        void ActionOnDestroy(NetworkObject obj) => Destroy(obj.gameObject);
        
        // Create the pool
        m_PooledObjects[prefab] = new ObjectPool<NetworkObject>(
            CreateFunc, ActionOnGet, ActionOnRelease, ActionOnDestroy,
            defaultCapacity: prewarmCount
        );
        
        // Prewarm: create objects now, return to pool
        var prewarmObjects = new List<NetworkObject>();
        for (var i = 0; i < prewarmCount; i++)
        {
            prewarmObjects.Add(m_PooledObjects[prefab].Get());
        }
        foreach (var obj in prewarmObjects)
        {
            m_PooledObjects[prefab].Release(obj);
        }
        
        // Hook into Netcode's spawn system
        NetworkManager.Singleton.PrefabHandler.AddHandler(
            prefab, new PooledPrefabInstanceHandler(prefab, this));
    }
    
    // Public API
    public NetworkObject GetNetworkObject(GameObject prefab, Vector3 pos, Quaternion rot)
    {
        var networkObject = m_PooledObjects[prefab].Get();
        networkObject.transform.SetPositionAndRotation(pos, rot);
        return networkObject;
    }
    
    public void ReturnNetworkObject(NetworkObject obj, GameObject prefab)
    {
        m_PooledObjects[prefab].Release(obj);
    }
}
```

### Netcode Integration

The magic is in `INetworkPrefabInstanceHandler`. This hooks into Netcode so that when the server spawns a NetworkObject, clients use the pool too:

```csharp
class PooledPrefabInstanceHandler : INetworkPrefabInstanceHandler
{
    GameObject m_Prefab;
    NetworkObjectPool m_Pool;
    
    // Called when Netcode needs to instantiate this prefab
    NetworkObject INetworkPrefabInstanceHandler.Instantiate(
        ulong ownerClientId, Vector3 position, Quaternion rotation)
    {
        return m_Pool.GetNetworkObject(m_Prefab, position, rotation);
    }
    
    // Called when Netcode despawns this object
    void INetworkPrefabInstanceHandler.Destroy(NetworkObject networkObject)
    {
        m_Pool.ReturnNetworkObject(networkObject, m_Prefab);
    }
}
```

### Usage

```csharp
// Server spawns a projectile
var projectile = NetworkObjectPool.Singleton.GetNetworkObject(
    bulletPrefab, spawnPosition, spawnRotation);
projectile.Spawn();  // This triggers the pool on clients too!

// When projectile hits something
projectile.Despawn();  // Returns to pool automatically
```

---

## DisposableGroup Pattern

When you have multiple subscriptions or resources to clean up, `DisposableGroup` bundles them together.

**File:** [DisposableGroup.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/DisposableGroup.cs)

```csharp
public class DisposableGroup : IDisposable
{
    List<IDisposable> m_Disposables = new List<IDisposable>();
    
    public void Add(IDisposable disposable)
    {
        m_Disposables.Add(disposable);
    }
    
    public void Dispose()
    {
        foreach (var disposable in m_Disposables)
        {
            disposable.Dispose();
        }
        m_Disposables.Clear();
    }
}
```

### Usage

```csharp
class PlayerUI : MonoBehaviour
{
    DisposableGroup m_Subscriptions = new DisposableGroup();
    
    void Start()
    {
        // Add multiple subscriptions to the group
        m_Subscriptions.Add(healthChannel.Subscribe(OnHealthChanged));
        m_Subscriptions.Add(manaChannel.Subscribe(OnManaChanged));
        m_Subscriptions.Add(scoreChannel.Subscribe(OnScoreChanged));
    }
    
    void OnDestroy()
    {
        // One call cleans up everything
        m_Subscriptions.Dispose();
    }
}
```

---

## UpdateRunner for Non-MonoBehaviours

Plain C# classes don't get Unity's `Update()` callback. `UpdateRunner` provides this.

**File:** [UpdateRunner.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/UpdateRunner.cs)

```csharp
public class UpdateRunner : MonoBehaviour
{
    static UpdateRunner s_Instance;
    
    List<Action> m_UpdateCallbacks = new List<Action>();
    List<Action> m_PendingAdds = new List<Action>();
    List<Action> m_PendingRemoves = new List<Action>();
    
    public static void Subscribe(Action callback)
    {
        EnsureInstance();
        s_Instance.m_PendingAdds.Add(callback);
    }
    
    public static void Unsubscribe(Action callback)
    {
        if (s_Instance != null)
            s_Instance.m_PendingRemoves.Add(callback);
    }
    
    void Update()
    {
        // Process pending changes
        foreach (var add in m_PendingAdds)
            m_UpdateCallbacks.Add(add);
        m_PendingAdds.Clear();
        
        foreach (var remove in m_PendingRemoves)
            m_UpdateCallbacks.Remove(remove);
        m_PendingRemoves.Clear();
        
        // Run all callbacks
        foreach (var callback in m_UpdateCallbacks)
            callback?.Invoke();
    }
}
```

### Usage

```csharp
public class MyService  // Not a MonoBehaviour!
{
    public MyService()
    {
        UpdateRunner.Subscribe(OnUpdate);
    }
    
    void OnUpdate()
    {
        // Called every frame!
    }
    
    public void Dispose()
    {
        UpdateRunner.Unsubscribe(OnUpdate);
    }
}
```

---

## ScriptableObject Architecture

The `ScriptableObjectArchitecture` folder contains patterns for data-driven game design.

### RuntimeSet

A set of objects that exist at runtime (e.g., all enemies, all players).

```csharp
public class RuntimeSet<T> : ScriptableObject
{
    List<T> m_Items = new List<T>();
    
    public void Add(T item) => m_Items.Add(item);
    public void Remove(T item) => m_Items.Remove(item);
    public IReadOnlyList<T> Items => m_Items;
}

// Usage
[CreateAssetMenu(menuName = "Game/Enemy Set")]
public class EnemySet : RuntimeSet<Enemy> { }

// Enemy.cs
[SerializeField] EnemySet allEnemies;

void OnEnable() => allEnemies.Add(this);
void OnDisable() => allEnemies.Remove(this);

// Any system can now iterate all enemies without FindObjectsOfType
foreach (var enemy in allEnemies.Items)
{
    enemy.TakeDamage(10);
}
```

### GameEvent

ScriptableObject-based events that work in the Inspector.

```csharp
[CreateAssetMenu(menuName = "Game/Event")]
public class GameEvent : ScriptableObject
{
    List<GameEventListener> m_Listeners = new List<GameEventListener>();
    
    public void Raise()
    {
        foreach (var listener in m_Listeners)
            listener.OnEventRaised();
    }
    
    public void RegisterListener(GameEventListener listener) 
        => m_Listeners.Add(listener);
    public void UnregisterListener(GameEventListener listener) 
        => m_Listeners.Remove(listener);
}

// Put this on any GameObject
public class GameEventListener : MonoBehaviour
{
    [SerializeField] GameEvent m_Event;
    [SerializeField] UnityEvent m_Response;
    
    void OnEnable() => m_Event.RegisterListener(this);
    void OnDisable() => m_Event.UnregisterListener(this);
    
    public void OnEventRaised() => m_Response.Invoke();
}
```

**Benefits:**
- Designers can wire up events in Inspector
- No code changes needed to add reactions
- Events are assets, can be referenced across scenes

---

## Key Takeaways

### Patterns Summary

| Pattern | Use Case | Boss Room Implementation |
|---------|----------|--------------------------|
| **PubSub** | Decoupled communication | MessageChannel<T> |
| **Networked PubSub** | Server broadcasts | NetworkedMessageChannel<T> |
| **Object Pool** | Avoid GC for spawns | NetworkObjectPool |
| **Disposable Group** | Cleanup multiple resources | DisposableGroup |
| **Update Injection** | Update for non-MonoBehaviours | UpdateRunner |
| **Runtime Set** | Track live objects | RuntimeSet<T> |
| **SO Events** | Designer-friendly events | GameEvent |

### When to Use Each

| Need | Solution |
|------|----------|
| "Health changed, update UI" | MessageChannel<HealthChangedMessage> |
| "Server tells all clients player joined" | NetworkedMessageChannel<PlayerJoinedMessage> |
| "Spawn 100 bullets per second" | NetworkObjectPool |
| "Clean up 5 subscriptions at once" | DisposableGroup |
| "Find all enemies without FindObjectsOfType" | RuntimeSet<Enemy> |

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forget to unsubscribe | Memory leak, ghost callbacks | Use IDisposable pattern |
| Modify collection while iterating | Crash | Use pending add/remove pattern |
| Spawn without pooling | GC spikes, lag | Use ObjectPool |
| FindObjectsOfType in Update | Major performance hit | Use RuntimeSet |

---

## Exercises

1. **Trace a Message:** Find where `ConnectStatus` messages are published and all places that subscribe to them.

2. **Add a Buffered Channel:** Look at BufferedMessageChannel. When would you use this? (Hint: late-joining clients)

3. **Pool Something New:** Configure NetworkObjectPool to pool enemy prefabs. Profile before/after.

4. **Create a RuntimeSet:** Make a `PlayerSet` that tracks all active players. Use it to display a player list UI.

---

> **Next:** [12_system_flow_diagrams.md](./12_system_flow_diagrams.md) — Visual diagrams of all major flows
