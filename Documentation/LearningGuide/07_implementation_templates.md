# 07: Implementation Templates

> **Purpose:** Production-ready, copy-paste templates for common game systems. Each template includes usage examples, extension points, and best practices.

---

## Table of Contents

1. [Event System](#template-1-event-system)
2. [Object Pool](#template-2-object-pool)
3. [State Machine](#template-3-state-machine)
4. [Service Locator](#template-4-service-locator)
5. [ScriptableObject Patterns](#template-5-scriptableobject-patterns)
6. [Save System](#template-6-save-system)
7. [Audio Manager](#template-7-audio-manager)
8. [Scene Loader](#template-8-scene-loader)
9. [Input Handler](#template-9-input-handler)
10. [UI Manager](#template-10-ui-manager)

---

## Template 1: Event System

### Basic Event Hub

```csharp
// GameEvents.cs
using System;

/// <summary>
/// Centralized event hub for game-wide events.
/// Publishers don't know about subscribers, and vice versa.
/// </summary>
public static class GameEvents
{
    // ═══════════════════════════════════════════════════════════
    // GAMEPLAY EVENTS
    // ═══════════════════════════════════════════════════════════
    
    public static event Action OnGameStart;
    public static event Action OnGameOver;
    public static event Action OnGamePaused;
    public static event Action OnGameResumed;
    public static event Action<string> OnLevelCompleted;
    
    // ═══════════════════════════════════════════════════════════
    // PLAYER EVENTS
    // ═══════════════════════════════════════════════════════════
    
    public static event Action<int, int> OnHealthChanged;       // current, max
    public static event Action<int> OnScoreChanged;
    public static event Action<int> OnCoinsChanged;
    public static event Action OnPlayerDied;
    public static event Action OnPlayerRespawned;
    public static event Action<float> OnExperienceGained;
    public static event Action<int> OnLevelUp;
    
    // ═══════════════════════════════════════════════════════════
    // COMBAT EVENTS
    // ═══════════════════════════════════════════════════════════
    
    public static event Action<int, Vector3> OnDamageDealt;     // amount, position
    public static event Action<string, Vector3> OnAbilityUsed; // ability name, position
    public static event Action<GameObject> OnEnemyKilled;
    
    // ═══════════════════════════════════════════════════════════
    // UI EVENTS
    // ═══════════════════════════════════════════════════════════
    
    public static event Action<string> OnShowMessage;
    public static event Action<string, float> OnShowToast;      // message, duration
    public static event Action<string> OnShowScreen;            // screen name
    public static event Action OnHideAllScreens;
    
    // ═══════════════════════════════════════════════════════════
    // RAISE METHODS (call these to trigger events)
    // ═══════════════════════════════════════════════════════════
    
    public static void RaiseGameStart() => OnGameStart?.Invoke();
    public static void RaiseGameOver() => OnGameOver?.Invoke();
    public static void RaiseGamePaused() => OnGamePaused?.Invoke();
    public static void RaiseGameResumed() => OnGameResumed?.Invoke();
    public static void RaiseLevelCompleted(string levelId) => OnLevelCompleted?.Invoke(levelId);
    
    public static void RaiseHealthChanged(int current, int max) => OnHealthChanged?.Invoke(current, max);
    public static void RaiseScoreChanged(int score) => OnScoreChanged?.Invoke(score);
    public static void RaiseCoinsChanged(int coins) => OnCoinsChanged?.Invoke(coins);
    public static void RaisePlayerDied() => OnPlayerDied?.Invoke();
    public static void RaisePlayerRespawned() => OnPlayerRespawned?.Invoke();
    public static void RaiseExperienceGained(float xp) => OnExperienceGained?.Invoke(xp);
    public static void RaiseLevelUp(int level) => OnLevelUp?.Invoke(level);
    
    public static void RaiseDamageDealt(int amount, Vector3 position) => OnDamageDealt?.Invoke(amount, position);
    public static void RaiseAbilityUsed(string name, Vector3 pos) => OnAbilityUsed?.Invoke(name, pos);
    public static void RaiseEnemyKilled(GameObject enemy) => OnEnemyKilled?.Invoke(enemy);
    
    public static void RaiseShowMessage(string msg) => OnShowMessage?.Invoke(msg);
    public static void RaiseShowToast(string msg, float duration = 2f) => OnShowToast?.Invoke(msg, duration);
    public static void RaiseShowScreen(string screen) => OnShowScreen?.Invoke(screen);
    public static void RaiseHideAllScreens() => OnHideAllScreens?.Invoke();
    
    // ═══════════════════════════════════════════════════════════
    // CLEANUP (call when loading new scene)
    // ═══════════════════════════════════════════════════════════
    
    public static void ClearAll()
    {
        OnGameStart = null;
        OnGameOver = null;
        OnGamePaused = null;
        OnGameResumed = null;
        OnLevelCompleted = null;
        OnHealthChanged = null;
        OnScoreChanged = null;
        OnCoinsChanged = null;
        OnPlayerDied = null;
        OnPlayerRespawned = null;
        OnExperienceGained = null;
        OnLevelUp = null;
        OnDamageDealt = null;
        OnAbilityUsed = null;
        OnEnemyKilled = null;
        OnShowMessage = null;
        OnShowToast = null;
        OnShowScreen = null;
        OnHideAllScreens = null;
    }
}
```

### Usage Examples

```csharp
// Publisher (doesn't know who listens)
public class PlayerHealth : MonoBehaviour
{
    [SerializeField] int maxHealth = 100;
    private int currentHealth;
    
    void Start() => currentHealth = maxHealth;
    
    public void TakeDamage(int amount)
    {
        currentHealth = Mathf.Max(0, currentHealth - amount);
        GameEvents.RaiseHealthChanged(currentHealth, maxHealth);
        
        if (currentHealth <= 0)
        {
            GameEvents.RaisePlayerDied();
        }
    }
}

// Subscriber (doesn't know who publishes)
public class HealthBarUI : MonoBehaviour
{
    [SerializeField] Slider healthSlider;
    
    void OnEnable()
    {
        GameEvents.OnHealthChanged += UpdateHealthBar;
    }
    
    void OnDisable()
    {
        GameEvents.OnHealthChanged -= UpdateHealthBar;  // ALWAYS unsubscribe!
    }
    
    void UpdateHealthBar(int current, int max)
    {
        healthSlider.value = (float)current / max;
    }
}

// Multiple subscribers - all react independently
public class AudioManager : MonoBehaviour
{
    void OnEnable()
    {
        GameEvents.OnPlayerDied += () => PlaySound("Death");
        GameEvents.OnDamageDealt += (amount, pos) => PlaySoundAt("Hit", pos);
        GameEvents.OnLevelUp += level => PlaySound("LevelUp");
    }
}

public class AchievementTracker : MonoBehaviour
{
    private int enemiesKilled;
    
    void OnEnable()
    {
        GameEvents.OnEnemyKilled += OnEnemyKilled;
    }
    
    void OnEnemyKilled(GameObject enemy)
    {
        enemiesKilled++;
        if (enemiesKilled >= 100)
            UnlockAchievement("Slayer");
    }
}
```

---

## Template 2: Object Pool

### Full-Featured Object Pool

```csharp
// ObjectPool.cs
using System;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Generic object pool for frequently spawned/despawned objects.
/// Avoids GC overhead from constant Instantiate/Destroy.
/// </summary>
public class ObjectPool<T> where T : Component
{
    private readonly Stack<T> m_Available = new Stack<T>();
    private readonly HashSet<T> m_InUse = new HashSet<T>();
    private readonly Func<T> m_CreateFunc;
    private readonly Action<T> m_OnGet;
    private readonly Action<T> m_OnRelease;
    private readonly Action<T> m_OnDestroy;
    private readonly Transform m_Parent;
    private readonly int m_MaxSize;
    
    public int CountAvailable => m_Available.Count;
    public int CountInUse => m_InUse.Count;
    public int CountTotal => CountAvailable + CountInUse;
    
    /// <summary>
    /// Create an object pool.
    /// </summary>
    /// <param name="createFunc">How to create new instances</param>
    /// <param name="onGet">Called when object is retrieved (activation)</param>
    /// <param name="onRelease">Called when object is returned (deactivation)</param>
    /// <param name="onDestroy">Called when object is destroyed</param>
    /// <param name="prewarmCount">Number to pre-create</param>
    /// <param name="maxSize">Maximum pool size (0 = unlimited)</param>
    /// <param name="parent">Optional parent transform for pooled objects</param>
    public ObjectPool(
        Func<T> createFunc,
        Action<T> onGet = null,
        Action<T> onRelease = null,
        Action<T> onDestroy = null,
        int prewarmCount = 10,
        int maxSize = 0,
        Transform parent = null)
    {
        m_CreateFunc = createFunc ?? throw new ArgumentNullException(nameof(createFunc));
        m_OnGet = onGet ?? DefaultOnGet;
        m_OnRelease = onRelease ?? DefaultOnRelease;
        m_OnDestroy = onDestroy ?? DefaultOnDestroy;
        m_MaxSize = maxSize;
        m_Parent = parent;
        
        // Prewarm the pool
        for (int i = 0; i < prewarmCount; i++)
        {
            var obj = Create();
            m_OnRelease(obj);
            m_Available.Push(obj);
        }
    }
    
    private T Create()
    {
        var obj = m_CreateFunc();
        if (m_Parent != null)
            obj.transform.SetParent(m_Parent);
        return obj;
    }
    
    private void DefaultOnGet(T obj) => obj.gameObject.SetActive(true);
    private void DefaultOnRelease(T obj) => obj.gameObject.SetActive(false);
    private void DefaultOnDestroy(T obj) => UnityEngine.Object.Destroy(obj.gameObject);
    
    /// <summary>
    /// Get an object from the pool.
    /// </summary>
    public T Get()
    {
        T obj = m_Available.Count > 0 ? m_Available.Pop() : Create();
        m_InUse.Add(obj);
        m_OnGet(obj);
        return obj;
    }
    
    /// <summary>
    /// Get an object and position it.
    /// </summary>
    public T Get(Vector3 position, Quaternion rotation)
    {
        T obj = Get();
        obj.transform.SetPositionAndRotation(position, rotation);
        return obj;
    }
    
    /// <summary>
    /// Return an object to the pool.
    /// </summary>
    public void Release(T obj)
    {
        if (obj == null || !m_InUse.Contains(obj))
            return;
        
        m_InUse.Remove(obj);
        
        // If at max size, destroy instead of returning
        if (m_MaxSize > 0 && CountTotal > m_MaxSize)
        {
            m_OnDestroy(obj);
            return;
        }
        
        m_OnRelease(obj);
        m_Available.Push(obj);
    }
    
    /// <summary>
    /// Release all in-use objects back to pool.
    /// </summary>
    public void ReleaseAll()
    {
        foreach (var obj in new List<T>(m_InUse))
        {
            Release(obj);
        }
    }
    
    /// <summary>
    /// Destroy all pooled objects.
    /// </summary>
    public void Clear()
    {
        foreach (var obj in m_Available)
            m_OnDestroy(obj);
        foreach (var obj in m_InUse)
            m_OnDestroy(obj);
        
        m_Available.Clear();
        m_InUse.Clear();
    }
}
```

### Pool Manager (MonoBehaviour Wrapper)

```csharp
// PoolManager.cs
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Central manager for multiple object pools.
/// </summary>
public class PoolManager : MonoBehaviour
{
    public static PoolManager Instance { get; private set; }
    
    [System.Serializable]
    public class PoolConfig
    {
        public string key;
        public GameObject prefab;
        public int prewarmCount = 10;
        public int maxSize = 50;
    }
    
    [SerializeField] List<PoolConfig> pools;
    
    private Dictionary<string, ObjectPool<Transform>> m_Pools = new();
    
    void Awake()
    {
        Instance = this;
        
        foreach (var config in pools)
        {
            CreatePool(config);
        }
    }
    
    void CreatePool(PoolConfig config)
    {
        var pool = new ObjectPool<Transform>(
            createFunc: () => Instantiate(config.prefab).transform,
            prewarmCount: config.prewarmCount,
            maxSize: config.maxSize,
            parent: transform
        );
        
        m_Pools[config.key] = pool;
    }
    
    public Transform Get(string key, Vector3 position, Quaternion rotation)
    {
        if (!m_Pools.TryGetValue(key, out var pool))
        {
            Debug.LogError($"Pool '{key}' not found!");
            return null;
        }
        
        return pool.Get(position, rotation);
    }
    
    public void Release(string key, Transform obj)
    {
        if (m_Pools.TryGetValue(key, out var pool))
        {
            pool.Release(obj);
        }
    }
}

// Usage
// var bullet = PoolManager.Instance.Get("Bullet", spawnPos, spawnRot);
// PoolManager.Instance.Release("Bullet", bullet);
```

---

## Template 3: State Machine

### Generic State Machine

```csharp
// StateMachine.cs
using System;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Generic state machine with enter/exit/update lifecycle.
/// </summary>
public abstract class StateMachine<TOwner> : MonoBehaviour where TOwner : class
{
    public State<TOwner> CurrentState { get; private set; }
    public State<TOwner> PreviousState { get; private set; }
    
    protected abstract TOwner Owner { get; }
    
    private Dictionary<Type, State<TOwner>> m_States = new();
    
    /// <summary>
    /// Register a state (call in Awake).
    /// </summary>
    protected void RegisterState<TState>(TState state) where TState : State<TOwner>
    {
        state.Initialize(Owner, this);
        m_States[typeof(TState)] = state;
    }
    
    /// <summary>
    /// Change to a new state by type.
    /// </summary>
    public void ChangeState<TState>() where TState : State<TOwner>
    {
        if (m_States.TryGetValue(typeof(TState), out var state))
        {
            ChangeState(state);
        }
        else
        {
            Debug.LogError($"State {typeof(TState).Name} not registered!");
        }
    }
    
    /// <summary>
    /// Change to a new state instance.
    /// </summary>
    public void ChangeState(State<TOwner> newState)
    {
        if (newState == CurrentState) return;
        
        PreviousState = CurrentState;
        
        CurrentState?.Exit();
        CurrentState = newState;
        CurrentState?.Enter();
        
        OnStateChanged(PreviousState, CurrentState);
    }
    
    /// <summary>
    /// Return to the previous state.
    /// </summary>
    public void RevertToPreviousState()
    {
        if (PreviousState != null)
            ChangeState(PreviousState);
    }
    
    protected virtual void Update()
    {
        CurrentState?.Update();
    }
    
    protected virtual void FixedUpdate()
    {
        CurrentState?.FixedUpdate();
    }
    
    protected virtual void OnStateChanged(State<TOwner> from, State<TOwner> to)
    {
        Debug.Log($"[State] {from?.GetType().Name ?? "None"} → {to.GetType().Name}");
    }
}

/// <summary>
/// Base state class.
/// </summary>
public abstract class State<TOwner> where TOwner : class
{
    protected TOwner Owner { get; private set; }
    protected StateMachine<TOwner> StateMachine { get; private set; }
    
    public void Initialize(TOwner owner, StateMachine<TOwner> stateMachine)
    {
        Owner = owner;
        StateMachine = stateMachine;
    }
    
    public virtual void Enter() { }
    public virtual void Update() { }
    public virtual void FixedUpdate() { }
    public virtual void Exit() { }
}
```

### Example: Player State Machine

```csharp
// PlayerStateMachine.cs
public class PlayerStateMachine : StateMachine<PlayerController>
{
    [SerializeField] PlayerController player;
    
    protected override PlayerController Owner => player;
    
    void Awake()
    {
        RegisterState(new IdleState());
        RegisterState(new RunState());
        RegisterState(new JumpState());
        RegisterState(new FallState());
        RegisterState(new AttackState());
    }
    
    void Start()
    {
        ChangeState<IdleState>();
    }
}

// IdleState.cs
public class IdleState : State<PlayerController>
{
    public override void Enter()
    {
        Owner.Animator.Play("Idle");
    }
    
    public override void Update()
    {
        if (Owner.Input.JumpPressed)
        {
            StateMachine.ChangeState<JumpState>();
            return;
        }
        
        if (Owner.Input.MoveDirection.magnitude > 0.1f)
        {
            StateMachine.ChangeState<RunState>();
        }
    }
}

// JumpState.cs
public class JumpState : State<PlayerController>
{
    public override void Enter()
    {
        Owner.Animator.Play("Jump");
        Owner.Rigidbody.AddForce(Vector3.up * Owner.JumpForce, ForceMode.Impulse);
    }
    
    public override void Update()
    {
        if (Owner.Rigidbody.velocity.y < 0)
        {
            StateMachine.ChangeState<FallState>();
        }
    }
}
```

---

## Template 4: Service Locator

### Simple Dependency Injection Alternative

```csharp
// Services.cs
using System;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Simple service locator for loose coupling.
/// Alternative to full DI framework for smaller projects.
/// </summary>
public static class Services
{
    private static Dictionary<Type, object> s_Services = new();
    private static Dictionary<Type, Func<object>> s_LazyServices = new();
    
    /// <summary>
    /// Register a service instance.
    /// </summary>
    public static void Register<T>(T service) where T : class
    {
        s_Services[typeof(T)] = service;
    }
    
    /// <summary>
    /// Register a lazy-loaded service (created on first access).
    /// </summary>
    public static void RegisterLazy<T>(Func<T> factory) where T : class
    {
        s_LazyServices[typeof(T)] = () => factory();
    }
    
    /// <summary>
    /// Get a registered service.
    /// </summary>
    public static T Get<T>() where T : class
    {
        // Check concrete instances
        if (s_Services.TryGetValue(typeof(T), out var service))
            return (T)service;
        
        // Check lazy factories
        if (s_LazyServices.TryGetValue(typeof(T), out var factory))
        {
            var instance = (T)factory();
            s_Services[typeof(T)] = instance;
            s_LazyServices.Remove(typeof(T));
            return instance;
        }
        
        Debug.LogError($"Service {typeof(T).Name} not registered!");
        return null;
    }
    
    /// <summary>
    /// Try to get a service (returns false if not registered).
    /// </summary>
    public static bool TryGet<T>(out T service) where T : class
    {
        if (s_Services.TryGetValue(typeof(T), out var obj))
        {
            service = (T)obj;
            return true;
        }
        
        service = null;
        return false;
    }
    
    /// <summary>
    /// Check if a service is registered.
    /// </summary>
    public static bool Has<T>() where T : class
    {
        return s_Services.ContainsKey(typeof(T)) || s_LazyServices.ContainsKey(typeof(T));
    }
    
    /// <summary>
    /// Unregister a service.
    /// </summary>
    public static void Unregister<T>() where T : class
    {
        s_Services.Remove(typeof(T));
        s_LazyServices.Remove(typeof(T));
    }
    
    /// <summary>
    /// Clear all services (call when changing scenes).
    /// </summary>
    public static void Clear()
    {
        s_Services.Clear();
        s_LazyServices.Clear();
    }
}
```

### Bootstrap Example

```csharp
// Bootstrap.cs
public class Bootstrap : MonoBehaviour
{
    void Awake()
    {
        // Register services at startup
        Services.Register<ISaveSystem>(new LocalSaveSystem());
        Services.Register<IAudioManager>(GetComponent<AudioManager>());
        Services.Register<ISceneLoader>(GetComponent<SceneLoader>());
        
        // Lazy registration (created when first needed)
        Services.RegisterLazy<IAnalytics>(() => new UnityAnalytics());
    }
}

// Usage anywhere in the codebase
public class GameManager : MonoBehaviour
{
    void SaveGame()
    {
        Services.Get<ISaveSystem>().Save(gameData);
    }
    
    void PlaySound(string name)
    {
        Services.Get<IAudioManager>().Play(name);
    }
}
```

---

## Template 5: ScriptableObject Patterns

### Runtime Set

```csharp
// RuntimeSet.cs
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Set of objects tracked at runtime.
/// No FindObjectsOfType needed!
/// </summary>
public abstract class RuntimeSet<T> : ScriptableObject
{
    private List<T> m_Items = new List<T>();
    
    public IReadOnlyList<T> Items => m_Items;
    public int Count => m_Items.Count;
    
    public void Add(T item)
    {
        if (!m_Items.Contains(item))
            m_Items.Add(item);
    }
    
    public void Remove(T item)
    {
        m_Items.Remove(item);
    }
    
    public void Clear()
    {
        m_Items.Clear();
    }
    
    void OnEnable()
    {
        // Clear when entering play mode
        m_Items.Clear();
    }
}

// Example
[CreateAssetMenu(menuName = "Game/Runtime Sets/Enemy Set")]
public class EnemySet : RuntimeSet<Enemy> { }

// Usage in Enemy.cs
public class Enemy : MonoBehaviour
{
    [SerializeField] EnemySet enemySet;
    
    void OnEnable() => enemySet.Add(this);
    void OnDisable() => enemySet.Remove(this);
}

// Usage elsewhere
public class EnemyCounter : MonoBehaviour
{
    [SerializeField] EnemySet enemies;
    [SerializeField] Text countText;
    
    void Update()
    {
        countText.text = $"Enemies: {enemies.Count}";
        
        // Iterate without allocations
        foreach (var enemy in enemies.Items)
        {
            // Process each enemy
        }
    }
}
```

### Game Config

```csharp
// GameConfig.cs
using UnityEngine;

[CreateAssetMenu(menuName = "Game/Game Config")]
public class GameConfig : ScriptableObject
{
    [Header("Player")]
    public int playerMaxHealth = 100;
    public float playerMoveSpeed = 5f;
    public float playerJumpForce = 12f;
    
    [Header("Combat")]
    public float attackCooldown = 0.5f;
    public int baseDamage = 10;
    
    [Header("Economy")]
    public int startingCoins = 0;
    public int coinsPerKill = 10;
    
    [Header("Progression")]
    public AnimationCurve xpCurve;
    
    public int GetXpForLevel(int level)
    {
        return Mathf.RoundToInt(xpCurve.Evaluate(level) * 1000);
    }
}

// Usage
public class Player : MonoBehaviour
{
    [SerializeField] GameConfig config;
    
    void Start()
    {
        maxHealth = config.playerMaxHealth;
        moveSpeed = config.playerMoveSpeed;
    }
}
```

---

## Template 6: Save System

```csharp
// ISaveSystem.cs
public interface ISaveSystem
{
    void Save<T>(string key, T data);
    T Load<T>(string key);
    bool HasKey(string key);
    void Delete(string key);
    void DeleteAll();
}

// LocalSaveSystem.cs
using System.IO;
using UnityEngine;

public class LocalSaveSystem : ISaveSystem
{
    private string GetPath(string key) => 
        Path.Combine(Application.persistentDataPath, $"{key}.json");
    
    public void Save<T>(string key, T data)
    {
        string json = JsonUtility.ToJson(data, prettyPrint: true);
        File.WriteAllText(GetPath(key), json);
    }
    
    public T Load<T>(string key)
    {
        string path = GetPath(key);
        if (!File.Exists(path))
            return default;
        
        string json = File.ReadAllText(path);
        return JsonUtility.FromJson<T>(json);
    }
    
    public bool HasKey(string key) => File.Exists(GetPath(key));
    
    public void Delete(string key)
    {
        string path = GetPath(key);
        if (File.Exists(path))
            File.Delete(path);
    }
    
    public void DeleteAll()
    {
        foreach (var file in Directory.GetFiles(Application.persistentDataPath, "*.json"))
        {
            File.Delete(file);
        }
    }
}

// SaveData.cs
[System.Serializable]
public class SaveData
{
    public string playerName;
    public int level;
    public int experience;
    public int coins;
    public Vector3 lastPosition;
    public string currentScene;
    public System.DateTime saveTime;
}

// Usage
Services.Get<ISaveSystem>().Save("slot1", saveData);
var loaded = Services.Get<ISaveSystem>().Load<SaveData>("slot1");
```

---

## Template 7: Audio Manager

```csharp
// AudioManager.cs
using System.Collections.Generic;
using UnityEngine;

public class AudioManager : MonoBehaviour, IAudioManager
{
    [System.Serializable]
    public class Sound
    {
        public string name;
        public AudioClip clip;
        [Range(0f, 1f)] public float volume = 1f;
        [Range(0.5f, 1.5f)] public float pitch = 1f;
        public bool loop;
    }
    
    [SerializeField] Sound[] sounds;
    [SerializeField] int poolSize = 10;
    
    private Dictionary<string, Sound> m_SoundDict = new();
    private Queue<AudioSource> m_SourcePool = new();
    private AudioSource m_MusicSource;
    
    void Awake()
    {
        // Build dictionary
        foreach (var sound in sounds)
            m_SoundDict[sound.name] = sound;
        
        // Create audio source pool
        for (int i = 0; i < poolSize; i++)
        {
            var go = new GameObject($"AudioSource_{i}");
            go.transform.SetParent(transform);
            var source = go.AddComponent<AudioSource>();
            source.playOnAwake = false;
            m_SourcePool.Enqueue(source);
        }
        
        // Music source
        m_MusicSource = gameObject.AddComponent<AudioSource>();
        m_MusicSource.loop = true;
    }
    
    public void Play(string name)
    {
        if (!m_SoundDict.TryGetValue(name, out var sound))
        {
            Debug.LogWarning($"Sound '{name}' not found!");
            return;
        }
        
        var source = GetSource();
        source.clip = sound.clip;
        source.volume = sound.volume;
        source.pitch = sound.pitch;
        source.loop = sound.loop;
        source.Play();
        
        if (!sound.loop)
        {
            StartCoroutine(ReturnToPool(source, sound.clip.length));
        }
    }
    
    public void PlayAt(string name, Vector3 position)
    {
        if (!m_SoundDict.TryGetValue(name, out var sound))
            return;
        
        AudioSource.PlayClipAtPoint(sound.clip, position, sound.volume);
    }
    
    public void PlayMusic(string name)
    {
        if (!m_SoundDict.TryGetValue(name, out var sound))
            return;
        
        m_MusicSource.clip = sound.clip;
        m_MusicSource.volume = sound.volume;
        m_MusicSource.Play();
    }
    
    public void StopMusic() => m_MusicSource.Stop();
    
    private AudioSource GetSource()
    {
        if (m_SourcePool.Count == 0)
        {
            // Expand pool if needed
            var go = new GameObject("AudioSource_Extra");
            go.transform.SetParent(transform);
            return go.AddComponent<AudioSource>();
        }
        return m_SourcePool.Dequeue();
    }
    
    private System.Collections.IEnumerator ReturnToPool(AudioSource source, float delay)
    {
        yield return new WaitForSeconds(delay);
        source.Stop();
        m_SourcePool.Enqueue(source);
    }
}
```

---

## Template 8: Scene Loader

```csharp
// SceneLoader.cs
using System;
using System.Collections;
using UnityEngine;
using UnityEngine.SceneManagement;

public class SceneLoader : MonoBehaviour, ISceneLoader
{
    [SerializeField] GameObject loadingScreen;
    [SerializeField] UnityEngine.UI.Slider progressBar;
    [SerializeField] float minimumLoadTime = 0.5f;
    
    public float Progress { get; private set; }
    public event Action OnLoadComplete;
    
    public void LoadScene(string sceneName, bool showLoading = true)
    {
        StartCoroutine(LoadSceneAsync(sceneName, showLoading));
    }
    
    public void LoadScene(int buildIndex, bool showLoading = true)
    {
        StartCoroutine(LoadSceneAsync(buildIndex, showLoading));
    }
    
    public void ReloadCurrentScene(bool showLoading = true)
    {
        LoadScene(SceneManager.GetActiveScene().buildIndex, showLoading);
    }
    
    private IEnumerator LoadSceneAsync(string sceneName, bool showLoading)
    {
        yield return LoadSceneAsync(SceneManager.LoadSceneAsync(sceneName), showLoading);
    }
    
    private IEnumerator LoadSceneAsync(int buildIndex, bool showLoading)
    {
        yield return LoadSceneAsync(SceneManager.LoadSceneAsync(buildIndex), showLoading);
    }
    
    private IEnumerator LoadSceneAsync(AsyncOperation operation, bool showLoading)
    {
        if (showLoading && loadingScreen != null)
            loadingScreen.SetActive(true);
        
        operation.allowSceneActivation = false;
        float startTime = Time.time;
        
        while (!operation.isDone)
        {
            Progress = Mathf.Clamp01(operation.progress / 0.9f);
            
            if (progressBar != null)
                progressBar.value = Progress;
            
            if (operation.progress >= 0.9f)
            {
                // Ensure minimum load time for smooth transition
                float elapsed = Time.time - startTime;
                if (elapsed < minimumLoadTime)
                {
                    yield return new WaitForSeconds(minimumLoadTime - elapsed);
                }
                
                operation.allowSceneActivation = true;
            }
            
            yield return null;
        }
        
        if (showLoading && loadingScreen != null)
            loadingScreen.SetActive(false);
        
        OnLoadComplete?.Invoke();
    }
}
```

---

## Template 9: Input Handler

```csharp
// InputData.cs
using UnityEngine;

public struct InputData
{
    public Vector2 MoveDirection;
    public Vector2 LookDirection;
    public bool JumpPressed;
    public bool JumpHeld;
    public bool AttackPressed;
    public bool InteractPressed;
    public bool PausePressed;
}

// IInputHandler.cs
public interface IInputHandler
{
    InputData GetInput();
}

// KeyboardInputHandler.cs
public class KeyboardInputHandler : MonoBehaviour, IInputHandler
{
    public InputData GetInput()
    {
        return new InputData
        {
            MoveDirection = new Vector2(
                Input.GetAxisRaw("Horizontal"),
                Input.GetAxisRaw("Vertical")
            ),
            LookDirection = new Vector2(
                Input.GetAxis("Mouse X"),
                Input.GetAxis("Mouse Y")
            ),
            JumpPressed = Input.GetButtonDown("Jump"),
            JumpHeld = Input.GetButton("Jump"),
            AttackPressed = Input.GetMouseButtonDown(0),
            InteractPressed = Input.GetKeyDown(KeyCode.E),
            PausePressed = Input.GetKeyDown(KeyCode.Escape)
        };
    }
}
```

---

## Template 10: UI Manager

```csharp
// UIManager.cs
using System.Collections.Generic;
using UnityEngine;

public class UIManager : MonoBehaviour
{
    [SerializeField] List<UIScreen> screens;
    
    private Dictionary<string, UIScreen> m_Screens = new();
    private Stack<UIScreen> m_ScreenStack = new();
    
    void Awake()
    {
        foreach (var screen in screens)
        {
            m_Screens[screen.ScreenName] = screen;
            screen.gameObject.SetActive(false);
        }
    }
    
    public void ShowScreen(string name, bool addToStack = true)
    {
        if (!m_Screens.TryGetValue(name, out var screen))
        {
            Debug.LogError($"Screen '{name}' not found!");
            return;
        }
        
        // Hide current if stacking
        if (addToStack && m_ScreenStack.Count > 0)
        {
            m_ScreenStack.Peek().gameObject.SetActive(false);
        }
        
        if (addToStack)
            m_ScreenStack.Push(screen);
        
        screen.gameObject.SetActive(true);
        screen.OnShow();
    }
    
    public void HideCurrentScreen()
    {
        if (m_ScreenStack.Count == 0) return;
        
        var current = m_ScreenStack.Pop();
        current.OnHide();
        current.gameObject.SetActive(false);
        
        // Show previous
        if (m_ScreenStack.Count > 0)
        {
            m_ScreenStack.Peek().gameObject.SetActive(true);
        }
    }
    
    public void HideAllScreens()
    {
        while (m_ScreenStack.Count > 0)
        {
            HideCurrentScreen();
        }
    }
}

// UIScreen.cs
public abstract class UIScreen : MonoBehaviour
{
    public abstract string ScreenName { get; }
    
    public virtual void OnShow() { }
    public virtual void OnHide() { }
}
```

---

## How to Use These Templates

1. **Copy the code** to your project's `_Common/Templates/` folder
2. **Modify to fit your needs** - these are starting points
3. **Keep interfaces separate** for flexibility
4. **Reference Boss Room** for production versions

---

> **Cross-Reference:**
> - [06_offline_game_guide.md](./06_offline_game_guide.md) - See these in context
> - [04_design_patterns.md](./04_design_patterns.md) - Pattern explanations
> - [16_performance_patterns.md](./16_performance_patterns.md) - Optimization tips
