# 01: Core Architecture Principles

> **Goal:** Master the fundamental architecture decisions in Boss Room with detailed code examples. These principles apply to ANY game, online or offline.

> 📍 **Key Files to Study:**
> - [ApplicationController.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ApplicationLifecycle/ApplicationController.cs) — DI container setup
> - [ServerCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs) — Server-side game logic
> - [ClientCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ClientCharacter.cs) — Client-side visuals
> - [ConnectionManager.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionManager.cs) — State machine pattern
> - [MessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs) — Event system

---

## Table of Contents

1. [Separation of Concerns](#principle-1-separation-of-concerns)
2. [Dependency Inversion (DI)](#principle-2-dependency-inversion-di)
3. [State Machine Pattern](#principle-3-state-machine-pattern)
4. [Event-Driven Communication](#principle-4-event-driven-communication)
5. [Data-Driven Design](#principle-5-data-driven-design)
6. [Architecture Layers](#architecture-layers)
7. [Exercises with Answers](#exercises-with-answers)

---

## Principle 1: Separation of Concerns

### What It Means
Each class/system does **ONE thing** well. Split responsibilities clearly.

### ❌ Without Separation
```csharp
// BAD: PlayerController that does EVERYTHING
public class PlayerController : MonoBehaviour
{
    // Movement - concern 1
    public float moveSpeed = 5f;
    void HandleMovement() { /* physics code */ }
    
    // Health - concern 2
    public int maxHealth = 100;
    void TakeDamage(int amount) { /* health code */ }
    
    // Combat - concern 3
    public int damage = 10;
    void Attack() { /* combat code */ }
    
    // Animation - concern 4
    void PlayAnimation(string name) { /* animation code */ }
    
    // UI - concern 5
    void UpdateHealthBar() { /* UI code */ }
    
    // Audio - concern 6
    void PlaySound(string name) { /* audio code */ }
    
    // Saving - concern 7
    void SaveGame() { /* save code */ }
    
    // This class will grow to 2000+ lines...
}
```

### ✅ With Separation
```
PlayerCharacter/
├── PlayerInput         → Reads controller/keyboard
├── PlayerMovement      → Moves character (physics)
├── PlayerHealth        → Tracks hit points
├── PlayerCombat        → Attacks and abilities
├── PlayerAnimator      → Plays animations
├── PlayerUI            → Shows health bar
└── PlayerAudio         → Sound effects
```

### Boss Room Implementation

Boss Room takes this to the extreme with **Server/Client split**:

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs`
```csharp
// SERVER-ONLY: Game logic runs here
public class ServerCharacter : NetworkBehaviour
{
    // State (authoritative)
    public NetworkVariable<int> HitPoints;
    public NetworkVariable<LifeState> LifeState;
    
    // Logic
    public void ReceiveHP(ServerCharacter inflicter, int HP)
    {
        // Apply damage/healing modifiers
        // Update health
        // Check for death
    }
    
    public override void OnNetworkSpawn()
    {
        if (!IsServer)
        {
            enabled = false;  // Disable on clients!
            return;
        }
        // Server-only initialization
    }
}
```

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/ClientCharacter.cs`
```csharp
// CLIENT-ONLY: Visuals run here
public class ClientCharacter : NetworkBehaviour
{
    // References to visual components
    private ClientActionPlayer m_ClientActionViz;
    private Animator m_Animator;
    
    // Listens to server state changes
    public override void OnNetworkSpawn()
    {
        if (!IsClient) return;
        
        // Subscribe to NetworkVariable changes
        m_ServerCharacter.NetHealthState.HitPoints.OnValueChanged += OnHealthChanged;
        m_ServerCharacter.IsStealthy.OnValueChanged += OnStealthChanged;
    }
    
    void OnHealthChanged(int oldHP, int newHP)
    {
        // Update health bar, play damage VFX
        // Pure visuals - no game logic!
    }
}
```

### Why This Matters

| Aspect | With SoC | Without SoC |
|--------|----------|-------------|
| Testing | Test movement alone | Must test entire player |
| Debugging | Know exactly where to look | Bug could be anywhere |
| Reuse | Movement works for NPCs too | Movement tied to player |
| Team Work | Different devs own components | Constant merge conflicts |
| Maintenance | Change one thing safely | Changes break unknown things |

### Apply to Your Game

Even in **single-player offline games**, split concerns:

```csharp
// Separation for a simple platformer
public class Mario : MonoBehaviour
{
    [SerializeField] MarioInput input;       // Reads buttons
    [SerializeField] MarioMovement movement; // Physics
    [SerializeField] MarioHealth health;     // Lives/damage
    [SerializeField] MarioAnimation anim;    // Sprites/animations
    [SerializeField] MarioAudio audio;       // Sounds
}
```

> 📚 **Deep Dive:** [18_character_system_deepdive.md](./18_character_system_deepdive.md)

---

## Principle 2: Dependency Inversion (DI)

### What It Means
Classes depend on **abstractions** (interfaces), not **concrete implementations**. Dependencies are **injected**, not created internally.

### ❌ Without DI
```csharp
// BAD: Creates its own dependencies
public class Enemy : MonoBehaviour
{
    private AudioManager audioManager;
    private ScoreSystem scoreSystem;
    
    void Start()
    {
        // Finds or creates dependencies internally
        audioManager = FindObjectOfType<AudioManager>();   // Hard to test!
        scoreSystem = ScoreSystem.Instance;                 // Tight coupling!
    }
    
    void Die()
    {
        audioManager.PlaySound("EnemyDeath");
        scoreSystem.AddPoints(100);
    }
}

// Problems:
// - Can't test without AudioManager and ScoreSystem
// - Can't swap implementations (mock, different platform)
// - Dependencies are hidden
```

### ✅ With DI
```csharp
// GOOD: Dependencies injected
public class Enemy : MonoBehaviour
{
    [Inject] private IAudioService m_Audio;    // Interface!
    [Inject] private IScoreService m_Score;    // Interface!
    
    // Dependencies are:
    // - Visible (you see what this class needs)
    // - Swappable (different implementations)
    // - Testable (inject mocks)
    
    void Die()
    {
        m_Audio.PlaySound("EnemyDeath");
        m_Score.AddPoints(100);
    }
}

// In tests:
var mockAudio = new MockAudioService();  // Doesn't play real sounds
var mockScore = new MockScoreService();  // Just counts calls
// Inject mocks, test behavior, no side effects!
```

### Boss Room Implementation

**File:** `Assets/Scripts/ApplicationLifecycle/ApplicationController.cs` (Configure method)
```csharp
protected override void Configure(IContainerBuilder builder)
{
    // Register services - WHAT to inject
    
    // Components from Inspector
    builder.RegisterComponent(m_UpdateRunner);
    builder.RegisterComponent(m_ConnectionManager);
    builder.RegisterComponent(m_NetworkManager);

    // Singletons
    builder.Register<LocalSessionUser>(Lifetime.Singleton);
    builder.Register<LocalSession>(Lifetime.Singleton);
    builder.Register<ProfileManager>(Lifetime.Singleton);
    builder.Register<PersistentGameState>(Lifetime.Singleton);

    // Message Channels (PubSub)
    builder.RegisterInstance(new MessageChannel<QuitApplicationMessage>())
           .AsImplementedInterfaces();
    builder.RegisterInstance(new MessageChannel<ConnectStatus>())
           .AsImplementedInterfaces();
}
```

**Usage in any class:**
```csharp
public class HostingState : ConnectionState
{
    // VContainer automatically injects these
    [Inject] protected IPublisher<ConnectionEventMessage> m_ConnectionEventPublisher;
    [Inject] protected SessionManager m_SessionManager;
    [Inject] protected LocalSession m_LocalSession;
    
    // Now use them - they're guaranteed to be set!
    void OnClientConnected(ulong clientId)
    {
        m_ConnectionEventPublisher.Publish(new ConnectionEventMessage { ... });
    }
}
```

### The DI Container Flow

```
1. Application starts
   ↓
2. ApplicationController.Awake() (VContainer LifetimeScope)
   ↓
3. Configure() registers all services
   ↓
4. Container.Build() - creates the dependency graph
   ↓
5. Start() - resolve and use services
   ↓
6. Any class with [Inject] gets dependencies automatically
```

### Benefits

| Benefit | Example |
|---------|---------|
| **Testable** | Inject mock audio that doesn't play sounds |
| **Swappable** | Inject CloudSave instead of LocalSave |
| **Visible** | See what a class needs by its [Inject] fields |
| **Decoupled** | Change SaveSystem without touching users |

### Apply to Your Game

Even without VContainer, you can use **poor man's DI**:

```csharp
// Simple constructor injection
public class GameManager
{
    private readonly ISaveService m_Save;
    private readonly IAudioService m_Audio;
    
    // Dependencies passed in, not created
    public GameManager(ISaveService save, IAudioService audio)
    {
        m_Save = save;
        m_Audio = audio;
    }
}

// In your bootstrap code
var save = new JsonSaveService();
var audio = new UnityAudioService();
var game = new GameManager(save, audio);  // Inject!
```

> 📚 **Deep Dive:** [15_testability_debugging.md](./15_testability_debugging.md)

---

## Principle 3: State Machine Pattern

### What It Means
Complex objects have **explicit states** with **defined transitions**. No more if-else spaghetti.

### ❌ Without State Machine
```csharp
// BAD: State tracked by booleans, logic everywhere
public class ConnectionHandler
{
    bool isConnected;
    bool isConnecting;
    bool isHost;
    bool isReconnecting;
    
    void HandleDisconnect()
    {
        if (isHost && isConnected && !isReconnecting)
        {
            // Server shutdown logic
        }
        else if (!isHost && isConnected && shouldTryReconnect)
        {
            // Client reconnect logic
            isReconnecting = true;
            isConnected = false;
        }
        else if (isConnecting && !isHost)
        {
            // Connection failed logic
        }
        // ... 20 more conditions ...
    }
}
```

### ✅ With State Machine
```csharp
// GOOD: Each state handles its own logic
public class ClientConnectedState : ConnectionState
{
    public override void OnClientDisconnect(ulong clientId)
    {
        if (ShouldTryReconnect())
        {
            m_ConnectionManager.ChangeState(m_ConnectionManager.m_ClientReconnecting);
        }
        else
        {
            m_ConnectionManager.ChangeState(m_ConnectionManager.m_Offline);
        }
    }
}

public class ClientReconnectingState : ConnectionState
{
    public override void OnClientDisconnect(ulong clientId)
    {
        if (m_AttemptCount < k_MaxAttempts)
        {
            TryReconnect();  // Different behavior in this state!
        }
        else
        {
            m_ConnectionManager.ChangeState(m_ConnectionManager.m_Offline);
        }
    }
}
```

### Boss Room Implementation

**File:** `Assets/Scripts/ConnectionManagement/ConnectionManager.cs`
```csharp
public class ConnectionManager : MonoBehaviour
{
    // All states pre-created (no GC during play)
    internal readonly OfflineState m_Offline = new OfflineState();
    internal readonly ClientConnectingState m_ClientConnecting = new ClientConnectingState();
    internal readonly ClientConnectedState m_ClientConnected = new ClientConnectedState();
    internal readonly ClientReconnectingState m_ClientReconnecting = new ClientReconnectingState();
    internal readonly StartingHostState m_StartingHost = new StartingHostState();
    internal readonly HostingState m_Hosting = new HostingState();

    private ConnectionState m_CurrentState;

    // THE ONLY WAY states change
    internal void ChangeState(ConnectionState nextState)
    {
        Debug.Log($"Changed state from {m_CurrentState} to {nextState}");
        m_CurrentState?.Exit();
        m_CurrentState = nextState;
        m_CurrentState.Enter();
    }
    
    // Events route to current state
    void OnConnectionEvent(NetworkManager nm, ConnectionEventData data)
    {
        switch (data.EventType)
        {
            case ConnectionEvent.ClientConnected:
                m_CurrentState.OnClientConnected(data.ClientId);
                break;
            case ConnectionEvent.ClientDisconnected:
                m_CurrentState.OnClientDisconnect(data.ClientId);
                break;
        }
    }
}
```

**State Diagram:**
```
                    ┌─────────────────┐
                    │    OFFLINE      │ ◄──────────────────────┐
                    └────────┬────────┘                        │
                             │                                 │
        ┌────────────────────┼────────────────────┐           │
        ▼                    ▼                    │           │
┌───────────────┐    ┌───────────────┐           │           │
│ STARTING_HOST │    │  CLIENT_      │           │           │
│               │    │  CONNECTING   │           │           │
└───────┬───────┘    └───────┬───────┘           │           │
        │                    │                    │           │
        ▼                    ▼                    │           │
┌───────────────┐    ┌───────────────┐           │           │
│   HOSTING     │    │   CLIENT_     │───────────┼───────────┘
│               │    │   CONNECTED   │           │
└───────┬───────┘    └───────┬───────┘           │
        │                    │                    │
        │                    ▼                    │
        │            ┌───────────────┐           │
        │            │   CLIENT_     │───────────┘
        │            │ RECONNECTING  │
        │            └───────────────┘
        │
        └─────────────────────────────────────────┘
```

### Why Pre-Create States?

```csharp
// Why this?
internal readonly OfflineState m_Offline = new OfflineState();

// Instead of this?
internal void ChangeState(ConnectionStateType type)
{
    m_CurrentState = new OfflineState();  // Allocates memory!
}
```

**Reasons:**
1. **No GC during gameplay** - states created once at startup
2. **States can hold dependencies** - injected once, reused
3. **States can hold temporary data** - persists across Enter/Exit

### Apply to Your Game

```csharp
// Simple player state machine for platformer
public enum PlayerState { Idle, Running, Jumping, Falling, Climbing, Dead }

public class PlayerStateMachine : MonoBehaviour
{
    private PlayerState currentState = PlayerState.Idle;
    
    public void ChangeState(PlayerState newState)
    {
        ExitState(currentState);
        currentState = newState;
        EnterState(currentState);
    }
    
    void EnterState(PlayerState state)
    {
        switch (state)
        {
            case PlayerState.Jumping:
                GetComponent<Rigidbody2D>().AddForce(Vector2.up * jumpForce);
                animator.Play("Jump");
                break;
            // ...
        }
    }
    
    void Update()
    {
        UpdateState(currentState);
    }
}
```

> 📚 **Deep Dive:** [10_connection_state_machine.md](./10_connection_state_machine.md)

---

## Principle 4: Event-Driven Communication

### What It Means
Objects communicate through **events/messages**, not direct calls. Publisher doesn't know who listens.

### ❌ Without Events
```csharp
// BAD: Health must know about ALL listeners
public class Health : MonoBehaviour
{
    public HealthBarUI healthBar;           // UI reference
    public AudioSource audioSource;         // Audio reference
    public AchievementSystem achievements;  // Achievement reference
    public AnalyticsSystem analytics;       // Analytics reference
    public QuestTracker quests;             // Quest reference
    
    public void TakeDamage(int amount)
    {
        currentHealth -= amount;
        
        // Health knows too much!
        healthBar.UpdateDisplay(currentHealth);
        audioSource.PlayOneShot(hurtSound);
        achievements.TrackDamageTaken(amount);
        analytics.LogEvent("damage", amount);
        quests.OnPlayerDamaged(amount);
        // Adding new listeners means modifying Health!
    }
}
```

### ✅ With Events
```csharp
// GOOD: Health just publishes, doesn't know who listens
public class Health : MonoBehaviour
{
    [Inject] IPublisher<HealthChangedMessage> m_Publisher;
    
    public void TakeDamage(int amount)
    {
        currentHealth -= amount;
        
        // Just publish - don't know, don't care who listens
        m_Publisher.Publish(new HealthChangedMessage {
            EntityId = gameObject.GetInstanceID(),
            OldHealth = currentHealth + amount,
            NewHealth = currentHealth
        });
    }
}

// Each listener subscribes independently
public class HealthBarUI : MonoBehaviour
{
    [Inject] ISubscriber<HealthChangedMessage> m_Subscriber;
    
    void Start() => m_Subscriber.Subscribe(OnHealthChanged);
    void OnHealthChanged(HealthChangedMessage msg) => UpdateUI(msg);
}

public class DamageSound : MonoBehaviour
{
    [Inject] ISubscriber<HealthChangedMessage> m_Subscriber;
    
    void Start() => m_Subscriber.Subscribe(OnHealthChanged);
    void OnHealthChanged(HealthChangedMessage msg) => PlayHurtSound();
}

// Add new listeners WITHOUT modifying Health!
```

### Boss Room Implementation

**File:** `Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs`
```csharp
public class MessageChannel<T> : IMessageChannel<T>
{
    readonly HashSet<Action<T>> m_MessageHandlers = new();
    readonly Dictionary<Action<T>, bool> m_PendingHandlers = new();
    
    public void Publish(T message)
    {
        // Apply pending changes (safe modification during iteration)
        foreach (var pending in m_PendingHandlers)
        {
            if (pending.Value)
                m_MessageHandlers.Add(pending.Key);
            else
                m_MessageHandlers.Remove(pending.Key);
        }
        m_PendingHandlers.Clear();
        
        // Invoke all handlers
        foreach (var handler in m_MessageHandlers)
        {
            handler?.Invoke(message);
        }
    }
    
    public IDisposable Subscribe(Action<T> handler)
    {
        m_PendingHandlers[handler] = true;
        return new DisposableSubscription<T>(this, handler);
    }
}
```

**Real Usage:**
```csharp
// Publisher (in ServerBossRoomState.cs)
m_LifeStatePublisher.Publish(new LifeStateChangedEventMessage {
    CharacterType = characterType,
    NewLifeState = newLifeState
});

// Subscriber (in ServerBossRoomState.cs)
m_LifeStateSubscriber.Subscribe(OnLifeStateChanged);

void OnLifeStateChanged(LifeStateChangedEventMessage msg)
{
    if (msg.NewLifeState == LifeState.Dead && 
        msg.CharacterType == CharacterTypeEnum.ImpBoss)
    {
        BossDefeated();  // Trigger win condition
    }
}
```

### Apply to Your Game

Simple event system for offline games:

```csharp
// Static events (simplest approach)
public static class GameEvents
{
    public static event Action<int> OnScoreChanged;
    public static event Action OnPlayerDied;
    public static event Action<string> OnLevelCompleted;
    
    public static void ScoreChanged(int score) => OnScoreChanged?.Invoke(score);
    public static void PlayerDied() => OnPlayerDied?.Invoke();
    public static void LevelCompleted(string level) => OnLevelCompleted?.Invoke(level);
}

// Publisher
void CollectCoin()
{
    score += 100;
    GameEvents.ScoreChanged(score);
}

// Subscribers (each in their own class)
void Start()
{
    GameEvents.OnScoreChanged += UpdateScoreUI;
    GameEvents.OnPlayerDied += ShowGameOverScreen;
}

void OnDestroy()
{
    GameEvents.OnScoreChanged -= UpdateScoreUI;  // Always unsubscribe!
    GameEvents.OnPlayerDied -= ShowGameOverScreen;
}
```

> 📚 **Deep Dive:** [11_infrastructure_patterns.md](./11_infrastructure_patterns.md)

---

## Principle 5: Data-Driven Design

### What It Means
Game behavior is controlled by **data** (config files, ScriptableObjects), not hard-coded values.

### ❌ Without Data-Driven
```csharp
// BAD: Values hard-coded in class
public class Fireball : Ability
{
    public override void Cast()
    {
        // Magic numbers everywhere!
        int damage = 25;
        float range = 10f;
        float cooldown = 2.5f;
        string animation = "CastFireball";
        
        PlayAnimation(animation);
        DamageInRange(range, damage);
        StartCooldown(cooldown);
    }
}

// Creating Fireball Level 2 = copy entire class, change numbers
// Balancing = recompile after every change
// Designer wants to tweak = needs programmer
```

### ✅ With Data-Driven
```csharp
// GOOD: Values from config
[CreateAssetMenu(menuName = "Game/Ability Config")]
public class AbilityConfig : ScriptableObject
{
    public string abilityName;
    public int damage;
    public float range;
    public float cooldownSeconds;
    public string animationTrigger;
    public GameObject vfxPrefab;
    public AudioClip soundEffect;
}

public class Ability : MonoBehaviour
{
    [SerializeField] AbilityConfig config;
    
    public void Cast()
    {
        PlayAnimation(config.animationTrigger);
        DamageInRange(config.range, config.damage);
        StartCooldown(config.cooldownSeconds);
        PlayVFX(config.vfxPrefab);
        PlaySound(config.soundEffect);
    }
}
```

### Boss Room Implementation

**File:** `Assets/Scripts/Gameplay/Action/ActionConfig.cs`
```csharp
[CreateAssetMenu(menuName = "Boss Room/Game Data/Action Config")]
public class ActionConfig : ScriptableObject
{
    [Header("Basic Info")]
    public ActionID ActionID;
    public string DisplayName;
    
    [Header("Timing")]
    public float DurationSeconds;      // How long action runs
    public float ExecTimeSeconds;      // When effect happens
    public float ReuseTimeSeconds;     // Cooldown
    
    [Header("Target")]
    public float Range;                // Attack range
    public bool IsFriendly;            // Buff vs attack
    
    [Header("Effect")]
    public int Amount;                 // Damage/healing amount
    public BlockingModeType BlockingMode;
    
    [Header("Visuals")]
    public string Anim;                // Animation trigger
    public GameObject Spawns;          // VFX prefab
}
```

**Usage in Action classes:**
```csharp
public class MeleeAction : Action
{
    public override bool OnUpdate(ServerCharacter parent)
    {
        // All values from Config - no magic numbers!
        if (TimeRunning >= Config.ExecTimeSeconds)
        {
            var hits = Physics.OverlapSphere(
                parent.Position, 
                Config.Range,     // From config
                k_LayerMask);
            
            foreach (var hit in hits)
            {
                var damageable = hit.GetComponent<IDamageable>();
                damageable?.ReceiveHP(parent, -Config.Amount);  // From config
            }
        }
        
        return TimeRunning < Config.DurationSeconds;  // From config
    }
}
```

### Benefits

| Benefit | Example |
|---------|---------|
| **Designers edit without code** | Change damage in Inspector |
| **Easy variations** | Fireball, IceBlast, Lightning = same class, different config |
| **Balance without recompile** | Tweak cooldown, hit Play immediately |
| **Version control** | Config changes tracked separately from code |

### Apply to Your Game

```csharp
// Enemy configuration for any game
[CreateAssetMenu(menuName = "Game/Enemy Config")]
public class EnemyConfig : ScriptableObject
{
    [Header("Stats")]
    public string enemyName;
    public int maxHealth;
    public float moveSpeed;
    public int contactDamage;
    
    [Header("Behavior")]
    public float detectionRange;
    public float attackCooldown;
    
    [Header("Loot")]
    public int scoreValue;
    public LootTable lootTable;
    
    [Header("Visuals")]
    public GameObject prefab;
    public RuntimeAnimatorController animator;
}

// Usage
public class Enemy : MonoBehaviour
{
    [SerializeField] EnemyConfig config;
    
    void Awake()
    {
        health = config.maxHealth;
        agent.speed = config.moveSpeed;
        // All behavior reads from config!
    }
}
```

> 📚 **Deep Dive:** [09_action_system_deepdive.md](./09_action_system_deepdive.md)

---

## Architecture Layers

Boss Room follows a **layered architecture**:

```
┌──────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                          │
│  Entry Point: ApplicationController                               │
│  Responsibility: Bootstrap, DI setup, scene management           │
│  Files: ApplicationController.cs, Startup scene                  │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                         DOMAIN LAYER                              │
│  Game Rules: Actions, Characters, GameState                       │
│  Responsibility: Core game logic (platform-independent)          │
│  Files: ServerCharacter, Action, ServerBossRoomState             │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                      INFRASTRUCTURE LAYER                         │
│  Services: Networking, Saving, Audio, Pooling                    │
│  Responsibility: Technical services, external APIs               │
│  Files: ConnectionManager, MessageChannel, NetworkObjectPool     │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                       PRESENTATION LAYER                          │
│  UI, VFX, Audio, Animations                                       │
│  Responsibility: Everything the player sees/hears                │
│  Files: ClientCharacter, UI scripts, VFX prefabs                 │
└──────────────────────────────────────────────────────────────────┘
```

### The Direction Rule

**Upper layers can call lower layers. Never the reverse.**

```csharp
// GOOD: Domain layer uses Infrastructure
public class ServerCharacter
{
    [Inject] IPublisher<LifeStateChangedEventMessage> m_Publisher;  // Infrastructure
    
    void OnDeath()
    {
        m_Publisher.Publish(...);  // Domain → Infrastructure ✅
    }
}

// GOOD: Presentation listens to Domain
public class HealthBarUI
{
    void OnHealthChanged(HealthChangedMessage msg)  // Presentation ← Domain ✅
    {
        UpdateUI(msg.NewHealth);
    }
}

// BAD: Infrastructure calling Domain directly
public class NetworkManager
{
    void OnMessage()
    {
        ServerCharacter.Instance.DoThing();  // Infrastructure → Domain ❌
    }
}
```

---

## Quick Reference

| Principle | One-Line Summary | Boss Room Example |
|-----------|------------------|-------------------|
| Separation of Concerns | One class = one job | ServerCharacter vs ClientCharacter |
| Dependency Inversion | Depend on interfaces, inject deps | VContainer [Inject] |
| State Machine | Explicit states & transitions | ConnectionState classes |
| Event-Driven | Communicate through events | MessageChannel PubSub |
| Data-Driven | Config in data, not code | ActionConfig ScriptableObjects |

---

## Exercises with Answers

### Exercise 1: Identify Principles

For each file, which principle(s) does it demonstrate?

**Q1:** `ServerCharacter.cs` vs `ClientCharacter.cs`
> **A:** Separation of Concerns - server handles logic, client handles visuals

**Q2:** `ApplicationController.cs` with `builder.Register<>()`
> **A:** Dependency Inversion - services registered for injection

**Q3:** `ConnectionManager.cs` with `ChangeState()`
> **A:** State Machine Pattern - states with Enter/Exit/transitions

**Q4:** `MessageChannel.cs` with Publish/Subscribe
> **A:** Event-Driven Communication - decoupled publishers and subscribers

**Q5:** `ActionConfig.cs` ScriptableObject
> **A:** Data-Driven Design - behavior defined by data, not code

### Exercise 2: Apply Principles

**Scenario:** You're building an inventory system. How would you apply each principle?

> **Separation of Concerns:**
> - `InventoryData` - stores item list
> - `InventoryLogic` - add/remove/stack logic
> - `InventoryUI` - displays items
> - `InventoryAudio` - plays sounds

> **Dependency Inversion:**
> - `IInventoryService` interface
> - Inject into any class that needs inventory access

> **Event-Driven:**
> - Publish `InventoryChangedMessage` when items change
> - UI, quest system, achievements subscribe

> **Data-Driven:**
> - `ItemConfig` ScriptableObject with stats
> - `InventorySlotConfig` for slot behavior

### Exercise 3: Find in Code

Using your IDE, find examples of:

1. **State transition:** Search `ChangeState` in ConnectionManager.cs
2. **Event publish:** Search `Publish(` anywhere
3. **DI registration:** Look at ApplicationController.Configure()
4. **Config usage:** Find ActionConfig being read

---

> **Next Steps:**
> - [04_design_patterns.md](./04_design_patterns.md) — Design patterns catalog
> - [10_connection_state_machine.md](./10_connection_state_machine.md) — State machine deep dive
> - [18_character_system_deepdive.md](./18_character_system_deepdive.md) — Server/Client split explained
