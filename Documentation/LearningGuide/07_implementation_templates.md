# 07: Implementation Templates

> **Goal:** Copy-paste templates for common systems you can use in your games.

---

## Template 1: Simple Event System

```csharp
// GameEvents.cs
using System;

public static class GameEvents
{
    // Define events
    public static event Action OnGameStart;
    public static event Action OnGameOver;
    public static event Action<int> OnScoreChanged;
    public static event Action<int> OnHealthChanged;
    
    // Raise methods
    public static void GameStart() => OnGameStart?.Invoke();
    public static void GameOver() => OnGameOver?.Invoke();
    public static void ScoreChanged(int score) => OnScoreChanged?.Invoke(score);
    public static void HealthChanged(int health) => OnHealthChanged?.Invoke(health);
    
    // Clear all (for scene reload)
    public static void ClearAll()
    {
        OnGameStart = null;
        OnGameOver = null;
        OnScoreChanged = null;
        OnHealthChanged = null;
    }
}
```

---

## Template 2: Object Pool

```csharp
// ObjectPool.cs
using System.Collections.Generic;
using UnityEngine;

public class ObjectPool<T> where T : Component
{
    private Queue<T> pool = new Queue<T>();
    private T prefab;
    private Transform parent;
    
    public ObjectPool(T prefab, int initialSize, Transform parent = null)
    {
        this.prefab = prefab;
        this.parent = parent;
        
        for (int i = 0; i < initialSize; i++)
        {
            CreateNew();
        }
    }
    
    private T CreateNew()
    {
        var obj = Object.Instantiate(prefab, parent);
        obj.gameObject.SetActive(false);
        pool.Enqueue(obj);
        return obj;
    }
    
    public T Get(Vector3 position, Quaternion rotation)
    {
        T obj = pool.Count > 0 ? pool.Dequeue() : CreateNew();
        obj.transform.SetPositionAndRotation(position, rotation);
        obj.gameObject.SetActive(true);
        return obj;
    }
    
    public void Return(T obj)
    {
        obj.gameObject.SetActive(false);
        pool.Enqueue(obj);
    }
}

// Usage:
// ObjectPool<Bullet> bulletPool = new ObjectPool<Bullet>(bulletPrefab, 20);
// Bullet b = bulletPool.Get(spawnPos, Quaternion.identity);
// bulletPool.Return(b);
```

---

## Template 3: State Machine

```csharp
// State.cs
public abstract class State<T> where T : class
{
    protected T owner;
    protected StateMachine<T> stateMachine;
    
    public void Setup(T owner, StateMachine<T> sm)
    {
        this.owner = owner;
        this.stateMachine = sm;
    }
    
    public virtual void Enter() { }
    public virtual void Update() { }
    public virtual void Exit() { }
}

// StateMachine.cs
public class StateMachine<T> where T : class
{
    public State<T> CurrentState { get; private set; }
    private T owner;
    
    public StateMachine(T owner)
    {
        this.owner = owner;
    }
    
    public void ChangeState(State<T> newState)
    {
        CurrentState?.Exit();
        CurrentState = newState;
        CurrentState.Setup(owner, this);
        CurrentState.Enter();
    }
    
    public void Update()
    {
        CurrentState?.Update();
    }
}

// Example state:
public class PlayerIdleState : State<PlayerController>
{
    public override void Enter() => owner.Animator.Play("Idle");
    
    public override void Update()
    {
        if (Input.GetAxis("Horizontal") != 0)
            stateMachine.ChangeState(new PlayerRunState());
    }
}
```

---

## Template 4: Simple Dependency Container

```csharp
// ServiceLocator.cs (simple DI alternative)
using System;
using System.Collections.Generic;

public static class Services
{
    private static Dictionary<Type, object> services = new();
    
    public static void Register<T>(T service)
    {
        services[typeof(T)] = service;
    }
    
    public static T Get<T>()
    {
        if (services.TryGetValue(typeof(T), out var service))
            return (T)service;
        throw new Exception($"Service {typeof(T)} not registered");
    }
    
    public static void Clear() => services.Clear();
}

// Registration (at startup):
Services.Register<ISaveSystem>(new JsonSaveSystem());
Services.Register<IAudioManager>(new AudioManager());

// Usage anywhere:
Services.Get<ISaveSystem>().Save(data);
```

---

## Template 5: ScriptableObject Config

```csharp
// EnemyConfig.cs
using UnityEngine;

[CreateAssetMenu(fileName = "NewEnemy", menuName = "Game/Enemy Config")]
public class EnemyConfig : ScriptableObject
{
    [Header("Stats")]
    public int maxHealth = 100;
    public float moveSpeed = 3f;
    public int damage = 10;
    
    [Header("Combat")]
    public float attackRange = 2f;
    public float attackCooldown = 1.5f;
    
    [Header("Visuals")]
    public GameObject prefab;
    public Color tintColor = Color.white;
}

// Usage:
public class Enemy : MonoBehaviour
{
    [SerializeField] EnemyConfig config;
    
    int currentHealth;
    
    void Start()
    {
        currentHealth = config.maxHealth;
    }
    
    void Attack()
    {
        if (distanceToPlayer < config.attackRange)
            player.TakeDamage(config.damage);
    }
}
```

---

## Template 6: UI Mediator

```csharp
// MainMenuMediator.cs
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.SceneManagement;

public class MainMenuMediator : MonoBehaviour
{
    [SerializeField] Button playButton;
    [SerializeField] Button optionsButton;
    [SerializeField] Button quitButton;
    [SerializeField] GameObject optionsPanel;
    
    void Start()
    {
        playButton.onClick.AddListener(OnPlayClicked);
        optionsButton.onClick.AddListener(OnOptionsClicked);
        quitButton.onClick.AddListener(OnQuitClicked);
    }
    
    void OnPlayClicked()
    {
        SceneManager.LoadScene("Gameplay");
    }
    
    void OnOptionsClicked()
    {
        optionsPanel.SetActive(true);
    }
    
    void OnQuitClicked()
    {
        #if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
        #else
        Application.Quit();
        #endif
    }
}
```

---

## How to Use These Templates

1. Copy the code to your project
2. Modify to fit your needs
3. Keep them in a `_Common/Templates/` folder
4. Reference Boss Room for advanced versions
