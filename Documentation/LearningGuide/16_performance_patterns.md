# 16: Performance Patterns

> **Purpose:** Memory management, garbage collection avoidance, and efficiency patterns that apply to ALL Unity games.

---

## Table of Contents

1. [Understanding Unity's Memory Model](#understanding-unitys-memory-model)
2. [Garbage Collection: The Enemy](#garbage-collection-the-enemy)
3. [Object Pooling Deep Dive](#object-pooling-deep-dive)
4. [Struct vs Class Decisions](#struct-vs-class-decisions)
5. [Caching Patterns](#caching-patterns)
6. [Update() Optimization](#update-optimization)
7. [Async/Await vs Coroutines](#asyncawait-vs-coroutines)
8. [Network Bandwidth Optimization](#network-bandwidth-optimization)
9. [Profiling Workflow](#profiling-workflow)

---

## Understanding Unity's Memory Model

### The Two Memory Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                        UNITY MEMORY MODEL                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MANAGED MEMORY (C#)                 NATIVE MEMORY (Unity Engine)   │
│  ─────────────────────               ─────────────────────────────  │
│  • Your C# objects                   • GameObjects                  │
│  • Strings, arrays                   • Components                   │
│  • Closures (lambdas)                • Textures, meshes             │
│  • Boxed value types                 • Audio clips                  │
│                                                                     │
│  Garbage Collected                   Reference Counted              │
│  (GC can cause stutters)             (Destroy() immediately)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### When GC Runs

The garbage collector runs when:
1. Managed heap is full
2. You call `GC.Collect()` (don't!)
3. Scene loads (Unity triggers it)

**The problem:** GC pauses your game to scan all objects.

---

## Garbage Collection: The Enemy

### Common GC Triggers

| Pattern | Problem | Solution |
|---------|---------|----------|
| String concatenation | Creates new strings | StringBuilder |
| `foreach` on some collections | Creates enumerator | `for` loop |
| LINQ | Creates iterators | Manual loops |
| Boxing | Value type → heap | Use generics |
| Closures (lambdas) | Captures create allocations | Cache delegates |
| `new` in Update | Allocation every frame | Pool or cache |

### BAD: Allocations Every Frame

```csharp
void Update()
{
    // BAD: New string every frame (3 allocations!)
    debugText.text = "Score: " + score + " / " + maxScore;
    
    // BAD: New array every frame
    var enemies = FindObjectsOfType<Enemy>();
    
    // BAD: New list every frame
    var nearbyEnemies = new List<Enemy>();
    
    // BAD: LINQ creates enumerators
    var filtered = enemies.Where(e => e.IsAlive).ToList();
    
    // BAD: Boxing (int → object)
    object boxed = score;
}
```

### GOOD: Zero-Allocation Patterns

```csharp
// Pre-allocate reusable objects
private StringBuilder m_StringBuilder = new StringBuilder(256);
private List<Enemy> m_EnemyCache = new List<Enemy>(100);
private Collider[] m_HitBuffer = new Collider[32];

void Update()
{
    // GOOD: Reuse StringBuilder
    m_StringBuilder.Clear();
    m_StringBuilder.Append("Score: ").Append(score).Append(" / ").Append(maxScore);
    debugText.text = m_StringBuilder.ToString();
    
    // GOOD: Non-allocating physics
    int hitCount = Physics.OverlapSphereNonAlloc(pos, radius, m_HitBuffer);
    
    // GOOD: Manual filtering without LINQ
    m_EnemyCache.Clear();
    for (int i = 0; i < allEnemies.Count; i++)
    {
        if (allEnemies[i].IsAlive)
            m_EnemyCache.Add(allEnemies[i]);
    }
}
```

---

## Object Pooling Deep Dive

### Why Pooling?

```
WITHOUT POOLING:
Spawn → Allocate → Play → Destroy → GC Eventually → Stutter!

WITH POOLING:
First spawn → Allocate once → Play → Return to pool (no destroy) → Reuse
```

### Boss Room's NetworkObjectPool

**File:** [NetworkObjectPool.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/NetworkObjectPool.cs)

```csharp
public class NetworkObjectPool : NetworkBehaviour
{
    // Pool storage per prefab type
    Dictionary<GameObject, ObjectPool<NetworkObject>> m_PooledObjects;
    
    void RegisterPrefabInternal(GameObject prefab, int prewarmCount)
    {
        // Define pool callbacks
        NetworkObject CreateFunc() 
            => Instantiate(prefab).GetComponent<NetworkObject>();
        
        void ActionOnGet(NetworkObject obj) 
            => obj.gameObject.SetActive(true);
        
        void ActionOnRelease(NetworkObject obj) 
            => obj.gameObject.SetActive(false);
        
        void ActionOnDestroy(NetworkObject obj) 
            => Destroy(obj.gameObject);
        
        // Create pool with Unity's built-in ObjectPool
        m_PooledObjects[prefab] = new ObjectPool<NetworkObject>(
            CreateFunc, 
            ActionOnGet, 
            ActionOnRelease, 
            ActionOnDestroy,
            defaultCapacity: prewarmCount
        );
        
        // PREWARM: Create objects now, before gameplay
        for (int i = 0; i < prewarmCount; i++)
        {
            var obj = m_PooledObjects[prefab].Get();
            m_PooledObjects[prefab].Release(obj);
        }
    }
}
```

### Generic Pool for Any Game

```csharp
public class SimplePool<T> where T : Component
{
    private readonly Stack<T> m_Available = new Stack<T>();
    private readonly Func<T> m_CreateFunc;
    
    public SimplePool(Func<T> createFunc, int prewarmCount = 10)
    {
        m_CreateFunc = createFunc;
        
        // Prewarm
        for (int i = 0; i < prewarmCount; i++)
        {
            var obj = createFunc();
            obj.gameObject.SetActive(false);
            m_Available.Push(obj);
        }
    }
    
    public T Get()
    {
        T obj = m_Available.Count > 0 ? m_Available.Pop() : m_CreateFunc();
        obj.gameObject.SetActive(true);
        return obj;
    }
    
    public void Release(T obj)
    {
        obj.gameObject.SetActive(false);
        m_Available.Push(obj);
    }
}

// Usage
private SimplePool<Bullet> m_BulletPool;

void Start()
{
    m_BulletPool = new SimplePool<Bullet>(
        () => Instantiate(bulletPrefab).GetComponent<Bullet>(),
        prewarmCount: 50
    );
}

void Shoot()
{
    var bullet = m_BulletPool.Get();
    bullet.Fire(direction);
}

// When bullet hits something
public void OnBulletHit(Bullet bullet)
{
    m_BulletPool.Release(bullet);
}
```

### What to Pool

| Pool | Don't Pool |
|------|------------|
| Bullets/projectiles | Player (only 1-4) |
| Particles (pooled by system) | Bosses (few, unique) |
| Enemies (if many spawn/die) | Static level geometry |
| Pickup items | Singleton managers |
| Damage numbers | Scene-persistent objects |
| Network objects (spawn traffic) | |

---

## Struct vs Class Decisions

### When to Use Struct

```csharp
// GOOD struct usage: small, immutable, value semantics
public struct DamageInfo
{
    public readonly int Amount;
    public readonly DamageType Type;
    public readonly Vector3 HitPoint;
    
    public DamageInfo(int amount, DamageType type, Vector3 hitPoint)
    {
        Amount = amount;
        Type = type;
        HitPoint = hitPoint;
    }
}

// Boss Room uses structs for network messages
public struct ConnectionPayload
{
    public string playerId;
    public string playerName;
    public bool isDebug;
}
```

### Struct Rules

| Use Struct When | Use Class When |
|-----------------|----------------|
| Size < 16 bytes | Larger objects |
| Immutable (readonly) | Mutable state |
| Short-lived | Long-lived |
| Passed by value makes sense | Need reference semantics |
| No inheritance needed | Need polymorphism |

### Warning: Boxing

```csharp
// BAD: Boxing allocation
void Process(object data)  // struct becomes boxed
{
    var damage = (DamageInfo)data;  // Unboxing
}

// GOOD: Use generics
void Process<T>(T data) where T : struct
{
    // No boxing
}
```

---

## Caching Patterns

### Component Caching

```csharp
// BAD: GetComponent every access
void Update()
{
    GetComponent<Rigidbody>().AddForce(Vector3.up);  // Slow!
}

// GOOD: Cache in Awake
private Rigidbody m_Rigidbody;

void Awake()
{
    m_Rigidbody = GetComponent<Rigidbody>();
}

void Update()
{
    m_Rigidbody.AddForce(Vector3.up);  // Fast!
}
```

### Transform Caching

```csharp
// Unity 2020+: transform is already cached, but...
// Accessing child/parent still allocates

// BAD: Finding children repeatedly
void Update()
{
    transform.Find("Weapon").position = newPos;  // Allocates!
}

// GOOD: Cache references
private Transform m_WeaponTransform;

void Awake()
{
    m_WeaponTransform = transform.Find("Weapon");
}
```

### Animator Parameter Caching

```csharp
// BAD: String comparison every time
animator.SetTrigger("Attack");

// GOOD: Cache the hash
private static readonly int AttackTrigger = Animator.StringToHash("Attack");

void Attack()
{
    animator.SetTrigger(AttackTrigger);  // Integer comparison
}
```

### Boss Room's Caching

```csharp
// Boss Room caches animation hashes in ActionConfig
public class ActionConfig : ScriptableObject
{
    public string AnimTrigger;
    
    // Cached on first access
    private int? m_AnimTriggerHash;
    public int AnimTriggerHash
    {
        get
        {
            m_AnimTriggerHash ??= Animator.StringToHash(AnimTrigger);
            return m_AnimTriggerHash.Value;
        }
    }
}
```

---

## Update() Optimization

### The UpdateManager Pattern

Instead of 1000 objects with Update(), have 1 manager updating 1000 objects:

```csharp
// BAD: Every enemy has Update()
public class Enemy : MonoBehaviour
{
    void Update()
    {
        // 1000 enemies = 1000 Update calls = 1000 native→managed transitions
    }
}

// GOOD: One manager updates all
public class EnemyManager : MonoBehaviour
{
    private List<Enemy> m_Enemies = new List<Enemy>();
    
    void Update()
    {
        // One Update call, process all
        for (int i = 0; i < m_Enemies.Count; i++)
        {
            m_Enemies[i].DoUpdate();  // Regular method, not Unity callback
        }
    }
}
```

### Staggered Updates

Not everything needs to update every frame:

```csharp
public class AIManager : MonoBehaviour
{
    private List<Enemy> m_Enemies;
    private int m_CurrentIndex;
    
    void Update()
    {
        // Update 10 enemies per frame instead of all
        int count = Mathf.Min(10, m_Enemies.Count);
        for (int i = 0; i < count; i++)
        {
            int index = (m_CurrentIndex + i) % m_Enemies.Count;
            m_Enemies[index].UpdateAI();
        }
        m_CurrentIndex = (m_CurrentIndex + count) % m_Enemies.Count;
    }
}
```

### Event-Driven Instead of Polling

```csharp
// BAD: Polling every frame
void Update()
{
    if (health != lastHealth)  // Check every frame
    {
        UpdateHealthBar();
        lastHealth = health;
    }
}

// GOOD: Event-driven
public event Action<int> OnHealthChanged;

public int Health
{
    get => m_Health;
    set
    {
        if (m_Health != value)
        {
            m_Health = value;
            OnHealthChanged?.Invoke(m_Health);  // Only when changed
        }
    }
}
```

---

## Async/Await vs Coroutines

### Coroutines

```csharp
// Unity's traditional approach
IEnumerator LoadAsync()
{
    Debug.Log("Starting load");
    
    yield return new WaitForSeconds(1f);  // Wait
    
    var operation = SceneManager.LoadSceneAsync("Level1");
    yield return operation;  // Wait for completion
    
    Debug.Log("Load complete");
}

// Start coroutine
StartCoroutine(LoadAsync());
```

### Async/Await (Modern C#)

```csharp
// Modern approach (Unity 2023+, or with UniTask)
async Task LoadAsync()
{
    Debug.Log("Starting load");
    
    await Task.Delay(1000);  // Wait 1 second
    
    var operation = SceneManager.LoadSceneAsync("Level1");
    while (!operation.isDone)
    {
        await Task.Yield();
    }
    
    Debug.Log("Load complete");
}
```

### When to Use Each

| Coroutines | Async/Await |
|------------|-------------|
| Simple timing | Complex async logic |
| WaitForSeconds | Multiple parallel operations |
| Existing Unity APIs | Cancellation needed |
| Quick prototyping | Error handling with try/catch |

### Boss Room's Async Pattern

```csharp
// Boss Room uses async for service initialization
async void Start()
{
    await InitializeServicesAsync();
    await AuthenticateAsync();
    LoadMainMenu();
}

// With proper error handling
async Task AuthenticateAsync()
{
    try
    {
        await AuthenticationService.Instance.SignInAnonymouslyAsync();
    }
    catch (AuthenticationException e)
    {
        Debug.LogError($"Auth failed: {e.Message}");
        // Handle gracefully
    }
}
```

---

## Network Bandwidth Optimization

### Send Less Data

```csharp
// BAD: Syncing full transform every frame
public class NetworkTransform : NetworkBehaviour
{
    [SyncVar] Vector3 position;      // 12 bytes
    [SyncVar] Quaternion rotation;   // 16 bytes
    [SyncVar] Vector3 scale;         // 12 bytes
    // 40 bytes per sync!
}

// GOOD: Quantize and compress
public class OptimizedNetworkTransform : NetworkBehaviour
{
    [SyncVar] short posX, posY, posZ;  // 6 bytes (quantized)
    [SyncVar] byte rotationY;           // 1 byte (just Y rotation)
    // 7 bytes per sync!
}
```

### Send Less Often

```csharp
// Boss Room's interpolation approach
public class ServerCharacterMovement : NetworkBehaviour
{
    // Server sends position updates at reduced rate
    // Client interpolates between received positions
    
    void FixedUpdate()
    {
        if (IsServer && frameCount % 3 == 0)  // Every 3rd frame
        {
            SyncPositionRpc(transform.position);
        }
    }
}
```

### Delta Compression

```csharp
// Only send what changed
public struct PlayerState
{
    public byte Flags;  // Which fields are included
    // Conditional fields based on flags
}

// If position didn't change, don't include it
```

---

## Profiling Workflow

### Step 1: Capture Profile

1. Open **Window → Analysis → Profiler**
2. Connect to target device
3. Enable categories: CPU, Memory, Rendering
4. **Deep Profile** for detailed callstacks (slow but thorough)

### Step 2: Identify Bottlenecks

**CPU Usage:**
- Sort by **Time ms** (self)
- Look for methods taking > 1ms
- Check call count (high count × small time = problem)

**Memory:**
- Look for **GC.Alloc** events
- Sort by **GC Alloc** column
- Ignore one-time allocations, focus on per-frame

### Step 3: Common Fixes

| Problem | Profile Shows | Fix |
|---------|---------------|-----|
| GC stutters | GC.Alloc in Update | Pool, cache, reuse |
| Slow physics | Physics.X taking too long | Simplify colliders, layers |
| Render bound | Camera.Render high | LOD, culling, batching |
| Scripts slow | Your method high | Optimize algorithm |

### Step 4: Verify Fix

Profile again after optimization to confirm improvement.

---

## Performance Checklist

### Pre-Release Audit

- [ ] No `FindObjectOfType` in Update
- [ ] No `GetComponent` in Update (cache in Awake)
- [ ] No string concatenation in Update (use StringBuilder)
- [ ] No `new` allocations in Update (pool or cache)
- [ ] No LINQ in Update (manual loops)
- [ ] Frequently spawned objects use pooling
- [ ] Animator parameters use hash caching
- [ ] Physics queries use NonAlloc versions

---

## Key Takeaways

1. **Profile first** - Don't optimize blindly
2. **Avoid allocations in hot paths** - Update, FixedUpdate, OnTrigger
3. **Pool frequently spawned objects** - Bullets, particles, enemies
4. **Cache expensive lookups** - GetComponent, Find, StringToHash
5. **Event-driven > polling** - Only process when things change
6. **Batch operations** - One manager updating many objects

---

> **Next:** [17_architecture_decision_framework.md](./17_architecture_decision_framework.md) — When to use which pattern
