# 06: Applying Patterns to Offline/Single-Player Games

> **Purpose:** Every pattern in Boss Room applies to single-player games. This guide shows you how to adapt multiplayer architecture for any game genre.

---

## Table of Contents

1. [Why These Patterns Apply Everywhere](#why-these-patterns-apply-everywhere)
2. [The Logic/View Split (Offline Server/Client)](#the-logicview-split)
3. [Pattern Adaptation Guide](#pattern-adaptation-guide)
4. [Genre-Specific Examples](#genre-specific-examples)
5. [Save System Architecture](#save-system-architecture)
6. [Scene Management Patterns](#scene-management-patterns)
7. [Adding Multiplayer Later](#adding-multiplayer-later)
8. [Complete Offline Architecture Template](#complete-offline-architecture-template)

---

## Why These Patterns Apply Everywhere

### The Universal Truth

The patterns in Boss Room solve **universal problems**:

| Problem | Pattern | Online Use | Offline Use |
|---------|---------|------------|-------------|
| Complex state transitions | State Machine | Connection states | Player states, menus, AI |
| Decoupled communication | PubSub/Events | Network events | Any game events |
| Swappable implementations | Strategy | IP vs Relay | Different AI, save methods |
| Efficient object reuse | Object Pool | Network spawning | Bullets, particles, enemies |
| Configurable behavior | Data-Driven | Synced game data | Any tweakable values |
| Testable, flexible code | DI | Service injection | Mockable systems |

### The Core Insight

**Server/Client separation is the same as Logic/View separation:**

```
MULTIPLAYER:                    SINGLE-PLAYER:
┌──────────────┐               ┌──────────────┐
│    SERVER    │               │    LOGIC     │
│  (Authority) │               │   (Rules)    │
│  Game rules  │               │  Game rules  │
│  Validation  │               │  Validation  │
│  State owner │               │  State owner │
└──────────────┘               └──────────────┘
       │                              │
       │ sync                         │ notify
       ▼                              ▼
┌──────────────┐               ┌──────────────┐
│    CLIENT    │               │     VIEW     │
│   (Display)  │               │  (Display)   │
│  Animations  │               │  Animations  │
│  VFX, Audio  │               │  VFX, Audio  │
│  UI updates  │               │  UI updates  │
└──────────────┘               └──────────────┘
```

---

## The Logic/View Split

### From ServerCharacter to GameLogic

```csharp
// BOSS ROOM (multiplayer)
public class ServerCharacter : NetworkBehaviour
{
    public NetworkVariable<int> HitPoints = new();
    
    public void ReceiveHP(int amount)
    {
        if (!IsServer) return;  // Only server changes state
        
        HitPoints.Value = Mathf.Clamp(HitPoints.Value + amount, 0, MaxHP);
        
        if (HitPoints.Value <= 0)
        {
            Die();
        }
    }
}

// YOUR OFFLINE GAME (same pattern!)
public class CharacterLogic : MonoBehaviour
{
    public int HitPoints { get; private set; }
    public int MaxHP = 100;
    
    public event Action<int, int> OnHealthChanged;  // current, max
    public event Action OnDied;
    
    public void ReceiveHP(int amount)
    {
        int oldHP = HitPoints;
        HitPoints = Mathf.Clamp(HitPoints + amount, 0, MaxHP);
        
        if (HitPoints != oldHP)
        {
            OnHealthChanged?.Invoke(HitPoints, MaxHP);
        }
        
        if (HitPoints <= 0)
        {
            OnDied?.Invoke();
        }
    }
}
```

### From ClientCharacter to GameView

```csharp
// BOSS ROOM (multiplayer)
public class ClientCharacter : NetworkBehaviour
{
    [SerializeField] Slider healthBar;
    [SerializeField] Animator animator;
    
    void Start()
    {
        serverCharacter.HitPoints.OnValueChanged += OnHealthChanged;
    }
    
    void OnHealthChanged(int oldValue, int newValue)
    {
        healthBar.value = (float)newValue / serverCharacter.MaxHP;
        
        if (newValue < oldValue)
        {
            animator.SetTrigger("Hit");
            PlayHitSound();
        }
    }
}

// YOUR OFFLINE GAME (same pattern!)
public class CharacterView : MonoBehaviour
{
    [SerializeField] CharacterLogic logic;
    [SerializeField] Slider healthBar;
    [SerializeField] Animator animator;
    
    void Start()
    {
        logic.OnHealthChanged += OnHealthChanged;
        logic.OnDied += OnDied;
    }
    
    void OnDestroy()
    {
        logic.OnHealthChanged -= OnHealthChanged;
        logic.OnDied -= OnDied;
    }
    
    void OnHealthChanged(int current, int max)
    {
        healthBar.value = (float)current / max;
        animator.SetTrigger("Hit");
        PlayHitSound();
    }
    
    void OnDied()
    {
        animator.SetTrigger("Death");
        PlayDeathSound();
    }
}
```

### Benefits of This Split

1. **Testability** - Test logic without visuals
2. **Replays** - Record/playback logic decisions
3. **Headless Mode** - Run game without rendering
4. **AI Training** - Fast simulation without graphics
5. **Multiplayer Ready** - Easy to add networking later

---

## Pattern Adaptation Guide

### State Machine → Player States

```csharp
// Offline player state machine
public abstract class PlayerState
{
    protected PlayerController player;
    
    public virtual void Enter() { }
    public virtual void Exit() { }
    public virtual void Update() { }
    public virtual void HandleInput(InputData input) { }
}

public class IdleState : PlayerState
{
    public override void HandleInput(InputData input)
    {
        if (input.moveDirection.magnitude > 0.1f)
        {
            player.ChangeState(player.runState);
        }
        else if (input.jumpPressed)
        {
            player.ChangeState(player.jumpState);
        }
    }
}

public class JumpState : PlayerState
{
    public override void Enter()
    {
        player.rb.AddForce(Vector3.up * player.jumpForce, ForceMode.Impulse);
    }
    
    public override void Update()
    {
        if (player.IsGrounded)
        {
            player.ChangeState(player.idleState);
        }
    }
}

public class PlayerController : MonoBehaviour
{
    // Pre-created states (no allocation during gameplay)
    internal readonly IdleState idleState = new IdleState();
    internal readonly RunState runState = new RunState();
    internal readonly JumpState jumpState = new JumpState();
    internal readonly FallState fallState = new FallState();
    internal readonly ClimbState climbState = new ClimbState();
    
    private PlayerState currentState;
    
    public void ChangeState(PlayerState newState)
    {
        currentState?.Exit();
        currentState = newState;
        currentState?.Enter();
    }
}
```

### PubSub → Game Events

```csharp
// Offline event system
public static class GameEvents
{
    // Gameplay
    public static event Action<int> OnScoreChanged;
    public static event Action<int, int> OnHealthChanged;  // current, max
    public static event Action OnPlayerDied;
    public static event Action<string> OnLevelCompleted;
    
    // UI
    public static event Action<string> OnShowMessage;
    public static event Action<string> OnShowTooltip;
    
    // System
    public static event Action OnGamePaused;
    public static event Action OnGameResumed;
    public static event Action OnGameSaved;
    
    // Raise methods
    public static void RaiseScoreChanged(int score) 
        => OnScoreChanged?.Invoke(score);
    
    public static void RaiseHealthChanged(int current, int max) 
        => OnHealthChanged?.Invoke(current, max);
    
    public static void RaisePlayerDied() 
        => OnPlayerDied?.Invoke();
    
    // Clear all (for scene cleanup)
    public static void ClearAll()
    {
        OnScoreChanged = null;
        OnHealthChanged = null;
        OnPlayerDied = null;
        // ... etc
    }
}

// Usage - publisher doesn't know about subscribers
public class Coin : MonoBehaviour
{
    [SerializeField] int value = 10;
    
    void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player"))
        {
            GameEvents.RaiseScoreChanged(value);
            Destroy(gameObject);
        }
    }
}

// Usage - subscriber doesn't know about publisher
public class ScoreUI : MonoBehaviour
{
    [SerializeField] Text scoreText;
    private int totalScore;
    
    void OnEnable()
    {
        GameEvents.OnScoreChanged += AddScore;
    }
    
    void OnDisable()
    {
        GameEvents.OnScoreChanged -= AddScore;  // Always unsubscribe!
    }
    
    void AddScore(int amount)
    {
        totalScore += amount;
        scoreText.text = $"Score: {totalScore}";
    }
}

// Other systems can subscribe too
public class AudioManager : MonoBehaviour
{
    void OnEnable()
    {
        GameEvents.OnScoreChanged += _ => PlaySound("Collect");
        GameEvents.OnPlayerDied += () => PlaySound("Death");
    }
}

public class AchievementSystem : MonoBehaviour
{
    private int totalCoins;
    
    void OnEnable()
    {
        GameEvents.OnScoreChanged += OnCoinCollected;
    }
    
    void OnCoinCollected(int amount)
    {
        totalCoins++;
        if (totalCoins >= 100) UnlockAchievement("Coin Master");
    }
}
```

### Object Pool → Bullets/Particles

```csharp
public class SimplePool<T> where T : Component
{
    private readonly Stack<T> available = new Stack<T>();
    private readonly Func<T> createFunc;
    private readonly Transform parent;
    
    public SimplePool(Func<T> createFunc, int prewarmCount = 10, Transform parent = null)
    {
        this.createFunc = createFunc;
        this.parent = parent;
        
        // Prewarm
        for (int i = 0; i < prewarmCount; i++)
        {
            var obj = createFunc();
            obj.transform.SetParent(parent);
            obj.gameObject.SetActive(false);
            available.Push(obj);
        }
    }
    
    public T Get(Vector3 position, Quaternion rotation)
    {
        T obj = available.Count > 0 ? available.Pop() : createFunc();
        obj.transform.position = position;
        obj.transform.rotation = rotation;
        obj.gameObject.SetActive(true);
        return obj;
    }
    
    public void Release(T obj)
    {
        obj.gameObject.SetActive(false);
        obj.transform.SetParent(parent);
        available.Push(obj);
    }
}

// Usage
public class WeaponController : MonoBehaviour
{
    [SerializeField] Bullet bulletPrefab;
    [SerializeField] Transform bulletContainer;
    
    private SimplePool<Bullet> bulletPool;
    
    void Awake()
    {
        bulletPool = new SimplePool<Bullet>(
            () => Instantiate(bulletPrefab),
            prewarmCount: 50,
            parent: bulletContainer
        );
    }
    
    public void Shoot(Vector3 position, Vector3 direction)
    {
        var bullet = bulletPool.Get(position, Quaternion.LookRotation(direction));
        bullet.Initialize(direction, damage: 10, pool: bulletPool);
    }
}

public class Bullet : MonoBehaviour
{
    private Vector3 direction;
    private int damage;
    private SimplePool<Bullet> pool;
    private float lifeTime = 3f;
    private float timer;
    
    public void Initialize(Vector3 dir, int damage, SimplePool<Bullet> pool)
    {
        this.direction = dir;
        this.damage = damage;
        this.pool = pool;
        this.timer = 0;
    }
    
    void Update()
    {
        transform.position += direction * 20f * Time.deltaTime;
        
        timer += Time.deltaTime;
        if (timer >= lifeTime)
        {
            pool.Release(this);  // Return to pool, don't destroy!
        }
    }
    
    void OnTriggerEnter(Collider other)
    {
        var damageable = other.GetComponent<IDamageable>();
        damageable?.TakeDamage(damage);
        pool.Release(this);
    }
}
```

---

## Genre-Specific Examples

### Platformer

```csharp
// Core player logic
public class PlatformerPlayer : MonoBehaviour
{
    // Logic (could be separate class)
    public int Health { get; private set; } = 3;
    public int Coins { get; private set; }
    
    // State machine for movement
    private PlayerState currentState;
    internal readonly IdleState idleState = new IdleState();
    internal readonly RunState runState = new RunState();
    internal readonly JumpState jumpState = new JumpState();
    internal readonly WallSlideState wallSlideState = new WallSlideState();
    
    // Configuration (data-driven)
    [SerializeField] PlatformerConfig config;
    
    void Start()
    {
        ChangeState(idleState);
    }
}

[CreateAssetMenu(menuName = "Game/Platformer Config")]
public class PlatformerConfig : ScriptableObject
{
    [Header("Movement")]
    public float moveSpeed = 8f;
    public float jumpForce = 12f;
    public float wallSlideSpeed = 2f;
    
    [Header("Combat")]
    public float invincibilityTime = 2f;
    public int maxHealth = 3;
    
    [Header("Feel")]
    public float coyoteTime = 0.1f;
    public float jumpBuffer = 0.1f;
}
```

### RPG / Action RPG

```csharp
// Character system (like Boss Room!)
public class RPGCharacter : MonoBehaviour
{
    // Core stats
    [SerializeField] CharacterStats stats;
    
    // Action system (simplified Boss Room)
    [SerializeField] ActionSlot[] actionSlots;
    
    // Equipment 
    public Weapon EquippedWeapon { get; private set; }
    public Armor EquippedArmor { get; private set; }
    
    // Events
    public event Action<int, int> OnHealthChanged;
    public event Action<int, int> OnManaChanged;
    public event Action<int> OnExperienceGained;
    public event Action<int> OnLevelUp;
}

[CreateAssetMenu(menuName = "Game/Character Stats")]
public class CharacterStats : ScriptableObject
{
    public string characterName;
    public int baseHealth = 100;
    public int baseMana = 50;
    public int baseAttack = 10;
    public int baseDefense = 5;
    public float attackSpeed = 1f;
    
    public AnimationCurve experienceCurve;
    public int GetExpForLevel(int level) 
        => Mathf.RoundToInt(experienceCurve.Evaluate(level) * 1000);
}

// Action system (simplified)
public abstract class GameAction : ScriptableObject
{
    public string actionName;
    public float cooldown;
    public int manaCost;
    public Sprite icon;
    public AnimationClip animation;
    
    public abstract void Execute(RPGCharacter user, Vector3 targetPosition);
}

[CreateAssetMenu(menuName = "Game/Actions/Melee Attack")]
public class MeleeAttack : GameAction
{
    public float range = 2f;
    public int damage = 20;
    
    public override void Execute(RPGCharacter user, Vector3 targetPos)
    {
        // Find targets in range
        var hits = Physics.OverlapSphere(user.transform.position, range);
        foreach (var hit in hits)
        {
            var enemy = hit.GetComponent<IDamageable>();
            if (enemy != null && enemy != user)
            {
                int totalDamage = damage + user.stats.baseAttack;
                enemy.TakeDamage(totalDamage);
            }
        }
    }
}
```

### Puzzle Game

```csharp
// Command pattern for undo/redo
public interface ICommand
{
    void Execute();
    void Undo();
}

public class MoveCommand : ICommand
{
    private PuzzlePiece piece;
    private Vector2Int fromPosition;
    private Vector2Int toPosition;
    
    public MoveCommand(PuzzlePiece piece, Vector2Int from, Vector2Int to)
    {
        this.piece = piece;
        this.fromPosition = from;
        this.toPosition = to;
    }
    
    public void Execute()
    {
        piece.MoveTo(toPosition);
    }
    
    public void Undo()
    {
        piece.MoveTo(fromPosition);
    }
}

public class PuzzleGame : MonoBehaviour
{
    private Stack<ICommand> history = new Stack<ICommand>();
    private Stack<ICommand> redoStack = new Stack<ICommand>();
    
    public void ExecuteMove(ICommand command)
    {
        command.Execute();
        history.Push(command);
        redoStack.Clear();  // Can't redo after new move
        
        CheckWinCondition();
    }
    
    public void Undo()
    {
        if (history.Count > 0)
        {
            var command = history.Pop();
            command.Undo();
            redoStack.Push(command);
        }
    }
    
    public void Redo()
    {
        if (redoStack.Count > 0)
        {
            var command = redoStack.Pop();
            command.Execute();
            history.Push(command);
        }
    }
}
```

### Strategy / RTS

```csharp
// Command pattern for unit orders
public interface IUnitCommand
{
    void Execute(Unit unit);
    bool IsComplete { get; }
}

public class MoveToCommand : IUnitCommand
{
    private Vector3 targetPosition;
    
    public MoveToCommand(Vector3 position)
    {
        targetPosition = position;
    }
    
    public bool IsComplete => /* check if arrived */;
    
    public void Execute(Unit unit)
    {
        unit.NavAgent.SetDestination(targetPosition);
    }
}

public class AttackCommand : IUnitCommand
{
    private Unit target;
    
    public AttackCommand(Unit target)
    {
        this.target = target;
    }
    
    public bool IsComplete => target == null || target.IsDead;
    
    public void Execute(Unit unit)
    {
        // Move to attack range, then attack
        if (unit.IsInAttackRange(target))
            unit.Attack(target);
        else
            unit.NavAgent.SetDestination(target.transform.position);
    }
}

// Unit processes commands in queue
public class Unit : MonoBehaviour
{
    private Queue<IUnitCommand> commandQueue = new Queue<IUnitCommand>();
    private IUnitCommand currentCommand;
    
    public void AddCommand(IUnitCommand command, bool clearQueue = false)
    {
        if (clearQueue) commandQueue.Clear();
        commandQueue.Enqueue(command);
    }
    
    void Update()
    {
        if (currentCommand == null || currentCommand.IsComplete)
        {
            if (commandQueue.Count > 0)
            {
                currentCommand = commandQueue.Dequeue();
                currentCommand.Execute(this);
            }
        }
    }
}
```

---

## Save System Architecture

### Interface-Based Save System

```csharp
// Saveable interface
public interface ISaveable
{
    string SaveId { get; }
    object CaptureState();
    void RestoreState(object state);
}

// Save system
public class SaveSystem : MonoBehaviour
{
    [Inject] ISaveStorage storage;  // Can be local, cloud, etc.
    
    public void Save(string slotName)
    {
        var saveData = new SaveData();
        
        // Find all saveables in scene
        var saveables = FindObjectsOfType<MonoBehaviour>()
            .OfType<ISaveable>();
        
        foreach (var saveable in saveables)
        {
            saveData.states[saveable.SaveId] = saveable.CaptureState();
        }
        
        storage.Save(slotName, saveData);
        GameEvents.RaiseGameSaved();
    }
    
    public void Load(string slotName)
    {
        var saveData = storage.Load<SaveData>(slotName);
        
        var saveables = FindObjectsOfType<MonoBehaviour>()
            .OfType<ISaveable>();
        
        foreach (var saveable in saveables)
        {
            if (saveData.states.TryGetValue(saveable.SaveId, out var state))
            {
                saveable.RestoreState(state);
            }
        }
    }
}

// Storage implementations (Strategy pattern)
public interface ISaveStorage
{
    void Save<T>(string key, T data);
    T Load<T>(string key);
    bool HasSave(string key);
    void Delete(string key);
}

public class LocalSaveStorage : ISaveStorage
{
    public void Save<T>(string key, T data)
    {
        string json = JsonUtility.ToJson(data);
        string path = Path.Combine(Application.persistentDataPath, key + ".json");
        File.WriteAllText(path, json);
    }
    
    public T Load<T>(string key)
    {
        string path = Path.Combine(Application.persistentDataPath, key + ".json");
        string json = File.ReadAllText(path);
        return JsonUtility.FromJson<T>(json);
    }
    
    // ... other methods
}

// Example saveable component
public class PlayerSaveable : MonoBehaviour, ISaveable
{
    [SerializeField] string saveId = "player";
    [SerializeField] CharacterLogic character;
    
    public string SaveId => saveId;
    
    public object CaptureState()
    {
        return new PlayerSaveData
        {
            position = transform.position,
            rotation = transform.rotation,
            health = character.HitPoints,
            level = character.Level,
            experience = character.Experience
        };
    }
    
    public void RestoreState(object state)
    {
        var data = (PlayerSaveData)state;
        transform.position = data.position;
        transform.rotation = data.rotation;
        character.SetHealth(data.health);
        character.SetLevel(data.level);
        character.SetExperience(data.experience);
    }
}

[Serializable]
public class PlayerSaveData
{
    public Vector3 position;
    public Quaternion rotation;
    public int health;
    public int level;
    public int experience;
}
```

---

## Scene Management Patterns

### Scene Loading Service

```csharp
public interface ISceneLoader
{
    void LoadScene(string sceneName, bool showLoadingScreen = true);
    void LoadSceneAdditive(string sceneName);
    void UnloadScene(string sceneName);
    float LoadProgress { get; }
    event Action OnSceneLoadComplete;
}

public class SceneLoaderService : MonoBehaviour, ISceneLoader
{
    [SerializeField] GameObject loadingScreen;
    [SerializeField] Slider progressBar;
    
    public float LoadProgress { get; private set; }
    public event Action OnSceneLoadComplete;
    
    public void LoadScene(string sceneName, bool showLoadingScreen = true)
    {
        StartCoroutine(LoadSceneAsync(sceneName, showLoadingScreen));
    }
    
    private IEnumerator LoadSceneAsync(string sceneName, bool showLoading)
    {
        if (showLoading) loadingScreen.SetActive(true);
        
        var operation = SceneManager.LoadSceneAsync(sceneName);
        operation.allowSceneActivation = false;
        
        while (!operation.isDone)
        {
            LoadProgress = Mathf.Clamp01(operation.progress / 0.9f);
            if (progressBar) progressBar.value = LoadProgress;
            
            if (operation.progress >= 0.9f)
            {
                // Wait a frame for visual feedback
                yield return null;
                operation.allowSceneActivation = true;
            }
            
            yield return null;
        }
        
        if (showLoading) loadingScreen.SetActive(false);
        OnSceneLoadComplete?.Invoke();
    }
}
```

---

## Adding Multiplayer Later

If you follow these patterns, here's how easy it becomes:

| Offline Pattern | Multiplayer Upgrade |
|-----------------|---------------------|
| `CharacterLogic` | Add `NetworkBehaviour`, make properties `NetworkVariable` |
| `CharacterView` | Subscribe to `NetworkVariable.OnValueChanged` |
| `GameEvents` | Replace with `NetworkedMessageChannel` |
| `ISaveStorage` | Add server-side validation before save |
| `ISceneLoader` | Add `NetworkSceneManager` calls |
| `ICommand` | Send as RPC to server, server executes |

```csharp
// Offline version
public class CharacterLogic : MonoBehaviour
{
    public int Health { get; private set; }
    public event Action<int> OnHealthChanged;
}

// Multiplayer upgrade (minimal changes!)
public class CharacterLogic : NetworkBehaviour  // Added
{
    public NetworkVariable<int> Health = new();  // Changed
    
    public override void OnNetworkSpawn()  // Added
    {
        Health.OnValueChanged += (old, current) => OnHealthChanged?.Invoke(current);
    }
    
    public event Action<int> OnHealthChanged;  // Same!
}
```

---

## Complete Offline Architecture Template

```
Assets/
├── _Game/
│   ├── Scripts/
│   │   ├── Core/
│   │   │   ├── Bootstrap.cs           # Entry point
│   │   │   ├── GameEvents.cs          # Static events
│   │   │   ├── ServiceLocator.cs      # Simple DI
│   │   │   └── GameSettings.cs        # Runtime settings
│   │   │
│   │   ├── Gameplay/
│   │   │   ├── Player/
│   │   │   │   ├── PlayerLogic.cs     # Game rules
│   │   │   │   ├── PlayerView.cs      # Visuals
│   │   │   │   ├── PlayerInput.cs     # Input handling
│   │   │   │   └── PlayerState/       # State machine
│   │   │   │       ├── IdleState.cs
│   │   │   │       ├── MoveState.cs
│   │   │   │       └── ...
│   │   │   │
│   │   │   ├── Enemies/
│   │   │   │   ├── EnemyLogic.cs
│   │   │   │   ├── EnemyView.cs
│   │   │   │   └── EnemyAI.cs
│   │   │   │
│   │   │   ├── Combat/
│   │   │   │   ├── IDamageable.cs
│   │   │   │   ├── DamageSystem.cs
│   │   │   │   └── AbilitySystem/
│   │   │   │
│   │   │   └── Pickups/
│   │   │       ├── Coin.cs
│   │   │       └── HealthPickup.cs
│   │   │
│   │   ├── UI/
│   │   │   ├── Screens/
│   │   │   │   ├── MainMenuScreen.cs
│   │   │   │   ├── PauseScreen.cs
│   │   │   │   └── GameOverScreen.cs
│   │   │   └── HUD/
│   │   │       ├── HealthBar.cs
│   │   │       └── ScoreDisplay.cs
│   │   │
│   │   ├── Services/
│   │   │   ├── SaveSystem/
│   │   │   │   ├── ISaveable.cs
│   │   │   │   ├── SaveManager.cs
│   │   │   │   └── LocalSaveStorage.cs
│   │   │   ├── AudioService.cs
│   │   │   └── SceneLoader.cs
│   │   │
│   │   └── Data/
│   │       ├── PlayerConfig.cs
│   │       ├── EnemyConfig.cs
│   │       └── LevelConfig.cs
│   │
│   ├── Prefabs/
│   ├── Scenes/
│   └── GameData/           # ScriptableObject instances
│
└── _Common/
    ├── Patterns/
    │   ├── ObjectPool.cs
    │   ├── StateMachine.cs
    │   └── ServiceLocator.cs
    └── Utils/
        └── Extensions.cs
```

---

## Key Takeaways

1. **Logic/View = Server/Client** - Same separation, different context
2. **Events decouple everything** - Publishers don't know subscribers
3. **Pool frequently spawned objects** - Works for any game
4. **Configure, don't hardcode** - ScriptableObjects everywhere
5. **State machines for complex states** - Player, AI, menus, game flow
6. **Design for multiplayer even if offline** - Easy to upgrade later

---

> **Cross-Reference:**
> - [04_design_patterns.md](./04_design_patterns.md) - Pattern details
> - [07_implementation_templates.md](./07_implementation_templates.md) - Copy-paste code
> - [17_architecture_decision_framework.md](./17_architecture_decision_framework.md) - When to use what
