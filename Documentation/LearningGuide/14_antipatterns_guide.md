# 14: Anti-Patterns Guide

> **Purpose:** Knowing what NOT to do is as important as knowing what TO do. This guide covers common mistakes and how the Boss Room architecture avoids them.

---

## Table of Contents

1. [Why Study Anti-Patterns?](#why-study-anti-patterns)
2. [The God Class](#the-god-class)
3. [Singleton Abuse](#singleton-abuse)
4. [Finding Objects at Runtime](#finding-objects-at-runtime)
5. [Tight Coupling](#tight-coupling)
6. [Leaky Subscriptions](#leaky-subscriptions)
7. [Magic Numbers and Strings](#magic-numbers-and-strings)
8. [Premature Optimization vs Ignoring Performance](#premature-optimization-vs-ignoring-performance)
9. [Unity-Specific Anti-Patterns](#unity-specific-anti-patterns)
10. [The Refactoring Checklist](#the-refactoring-checklist)

---

## Why Study Anti-Patterns?

An **anti-pattern** is a common solution that seems reasonable but creates more problems than it solves. Studying anti-patterns helps you:

1. **Recognize bad code** before it becomes your habit
2. **Refactor legacy code** by identifying what's wrong
3. **Understand WHY good patterns exist** (they solve anti-pattern problems)
4. **Avoid technical debt** that slows development later

---

## The God Class

### What It Is

A class that does too many things, knows too much, and has too many responsibilities.

### ❌ Anti-Pattern Example

```csharp
// BAD: "PlayerController" that does EVERYTHING
public class PlayerController : MonoBehaviour
{
    // Movement
    public float moveSpeed = 5f;
    private Rigidbody rb;
    
    // Health
    public int maxHealth = 100;
    private int currentHealth;
    
    // Combat  
    public int damage = 10;
    private float attackCooldown;
    
    // Inventory
    public List<Item> inventory = new List<Item>();
    private int gold;
    
    // UI References
    public Slider healthBar;
    public Text goldText;
    public GameObject inventoryPanel;
    
    // Audio
    public AudioClip walkSound;
    public AudioClip attackSound;
    public AudioClip hurtSound;
    
    // Save/Load
    public string saveFileName = "player.save";
    
    void Update()
    {
        HandleInput();
        HandleMovement();
        HandleCombat();
        UpdateUI();
        CheckAchievements();
        AutoSave();
    }
    
    // ... hundreds of lines of mixed responsibilities ...
}
```

**Problems:**
- Changing UI requires modifying player logic
- Can't reuse combat system for enemies
- Can't test movement without loading everything
- One bug can break everything
- Multiple developers can't work simultaneously

### ✅ Boss Room Solution

```csharp
// GOOD: Single Responsibility - Each class does ONE thing

// Just movement
public class ServerCharacterMovement : NetworkBehaviour { }

// Just health state
public class NetworkHealthState : NetworkBehaviour { }

// Just actions/combat
public class ServerActionPlayer { }

// Just visual feedback
public class ClientCharacter : NetworkBehaviour { }

// Just animation handling
public class ClientAnimationHandler { }
```

**Key files to study:**
- [ServerCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs) - Orchestrates, delegates to specialists
- [ServerCharacterMovement.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacterMovement.cs) - Only movement

### How to Fix God Classes

1. **Identify responsibilities** - List everything the class does
2. **Group related items** - Movement, Combat, UI, etc.
3. **Extract classes** - One class per group
4. **Use composition** - Main class holds references to specialists
5. **Define interfaces** - Let specialists communicate via contracts

---

## Singleton Abuse

### What It Is

Using singletons for everything because they're "easy" - creating hidden dependencies everywhere.

### ❌ Anti-Pattern Example

```csharp
// BAD: Singletons everywhere
public class GameManager : MonoBehaviour
{
    public static GameManager Instance;
    void Awake() => Instance = this;
}

public class AudioManager : MonoBehaviour
{
    public static AudioManager Instance;
    void Awake() => Instance = this;
}

public class UIManager : MonoBehaviour
{
    public static UIManager Instance;
    void Awake() => Instance = this;
}

// Now code everywhere accesses these directly:
public class Enemy
{
    void Die()
    {
        GameManager.Instance.AddScore(100);           // Hidden dependency
        AudioManager.Instance.PlaySound("death");     // Hidden dependency
        UIManager.Instance.ShowKillNotification();    // Hidden dependency
        AnalyticsManager.Instance.TrackKill();        // Hidden dependency
        AchievementManager.Instance.CheckKillCount(); // Hidden dependency
    }
}
```

**Problems:**
- Hidden dependencies (not visible in constructor/inspector)
- Hard to test (can't mock singletons easily)
- Order of initialization matters (race conditions)
- Tight coupling to specific implementations
- Can't have multiple instances when needed

### ✅ Boss Room Solution

Boss Room uses **Dependency Injection** instead of singletons:

```csharp
// GOOD: Dependencies are explicit and injectable
public class HostingState : ConnectionState
{
    // Dependencies are DECLARED explicitly
    [Inject] protected ConnectionManager m_ConnectionManager;
    [Inject] protected IPublisher<ConnectionEventMessage> m_ConnectionEventPublisher;
    [Inject] protected SessionManager m_SessionManager;
    
    // Now it's clear what this class needs to work
    // AND we can inject mocks for testing
}
```

**Key insight:** VContainer manages creation and injection, so you never need:
- `FindObjectOfType<T>()`
- `T.Instance`
- `new T()`

### When Singletons ARE Acceptable

Singletons are okay for:
- **Truly global, stateless utilities** (e.g., math helpers)
- **Hardware abstractions** (e.g., InputSystem, AudioListener)
- **During rapid prototyping** (refactor before production)

---

## Finding Objects at Runtime

### What It Is

Using `FindObjectOfType`, `FindWithTag`, or `GetComponent` frequently at runtime.

### ❌ Anti-Pattern Example

```csharp
// BAD: Finding things every frame
public class Enemy : MonoBehaviour
{
    void Update()
    {
        // SLOW: Searches entire scene!
        var player = FindObjectOfType<Player>();
        
        if (player != null)
        {
            // SLOW: GetComponent every frame!
            var playerHealth = player.GetComponent<Health>();
            
            if (Vector3.Distance(transform.position, player.transform.position) < 10f)
            {
                Attack(playerHealth);
            }
        }
    }
}
```

**Problems:**
- `FindObjectOfType` is O(n) - scans all objects
- Creates garbage from search results
- Fails silently if object not found
- No compile-time safety

### ✅ Boss Room Solutions

**Solution 1: Cache references**
```csharp
// Cache in Start(), use repeatedly
private Player m_Player;
private Health m_PlayerHealth;

void Start()
{
    m_Player = FindObjectOfType<Player>();  // Once only
    m_PlayerHealth = m_Player.GetComponent<Health>();
}

void Update()
{
    if (m_Player != null) { /* use cached */ }
}
```

**Solution 2: Dependency Injection (Boss Room's approach)**
```csharp
public class HeroActionBar : MonoBehaviour
{
    [Inject] ServerCharacter m_ServerCharacter;  // Injected, not found
    
    void Start()
    {
        // m_ServerCharacter is already available
    }
}
```

**Solution 3: RuntimeSet pattern**
```csharp
// Track all enemies without searching
[CreateAssetMenu]
public class EnemySet : RuntimeSet<Enemy> { }

public class Enemy : MonoBehaviour
{
    [SerializeField] EnemySet allEnemies;
    
    void OnEnable() => allEnemies.Add(this);
    void OnDisable() => allEnemies.Remove(this);
}

// Now anywhere:
foreach (var enemy in allEnemies.Items)
{
    // No FindObjectsOfType needed!
}
```

---

## Tight Coupling

### What It Is

Classes that directly reference and depend on specific other classes.

### ❌ Anti-Pattern Example

```csharp
// BAD: Health directly calls specific classes
public class Health : MonoBehaviour
{
    public int currentHealth;
    public HealthBarUI healthBar;           // Direct reference to UI
    public AudioSource audioSource;         // Direct reference to audio
    public DeathParticleEffect deathEffect; // Direct reference to VFX
    
    public void TakeDamage(int amount)
    {
        currentHealth -= amount;
        
        healthBar.UpdateDisplay(currentHealth);     // Coupled to UI
        audioSource.PlayOneShot(hurtSound);         // Coupled to audio
        
        if (currentHealth <= 0)
        {
            deathEffect.Play();                     // Coupled to VFX
            GetComponent<EnemyAI>()?.OnDeath();     // Coupled to AI
            FindObjectOfType<ScoreManager>()?.AddScore(100); // Coupled to score
        }
    }
}
```

**Problems:**
- Health knows about UI, audio, VFX, AI, scoring...
- Can't use Health without all dependencies
- Changes to UI require changes to Health
- Testing requires all systems present

### ✅ Boss Room Solution: Events/PubSub

```csharp
// GOOD: Health just manages health, publishes events
public class ServerCharacter : NetworkBehaviour
{
    [Inject] IPublisher<HealthChangedMessage> m_HealthPublisher;
    
    public void ReceiveHP(int amount)
    {
        m_Health.Value += amount;
        
        // Just publish - don't know or care who listens
        m_HealthPublisher.Publish(new HealthChangedMessage {
            CharacterId = NetworkObjectId,
            NewHealth = m_Health.Value
        });
    }
}

// Subscribers handle their own logic:
public class HealthBarUI { /* subscribes to update display */ }
public class AudioManager { /* subscribes to play sound */ }
public class AchievementSystem { /* subscribes to track */ }
```

---

## Leaky Subscriptions

### What It Is

Subscribing to events but forgetting to unsubscribe, causing memory leaks and ghost callbacks.

### ❌ Anti-Pattern Example

```csharp
// BAD: Subscribe without unsubscribe
public class ScoreUI : MonoBehaviour
{
    void Start()
    {
        GameEvents.OnScoreChanged += UpdateScore;  // Subscribed!
        // But never unsubscribed...
    }
    
    void UpdateScore(int score)
    {
        // This gets called even after object is destroyed!
        scoreText.text = score.ToString();  // NullReferenceException!
    }
    
    // Missing: void OnDestroy() { GameEvents.OnScoreChanged -= UpdateScore; }
}
```

**Problems:**
- Object destroyed but callback still registered
- Memory leak (object can't be garbage collected)
- Callback invoked on destroyed object (exception)
- Hard to debug (no obvious error)

### ✅ Boss Room Solution: IDisposable Pattern

```csharp
// GOOD: Subscription returns IDisposable for easy cleanup
public class SomeComponent : MonoBehaviour
{
    IDisposable m_Subscription;
    
    void Start()
    {
        // Subscribe returns disposable
        m_Subscription = m_MessageChannel.Subscribe(OnMessage);
    }
    
    void OnDestroy()
    {
        // Clean disposal - impossible to forget pattern
        m_Subscription?.Dispose();
    }
}

// EVEN BETTER: DisposableGroup for multiple subscriptions
public class ComplexComponent : MonoBehaviour
{
    DisposableGroup m_Subscriptions = new DisposableGroup();
    
    void Start()
    {
        m_Subscriptions.Add(channel1.Subscribe(Handler1));
        m_Subscriptions.Add(channel2.Subscribe(Handler2));
        m_Subscriptions.Add(channel3.Subscribe(Handler3));
    }
    
    void OnDestroy()
    {
        m_Subscriptions.Dispose();  // Cleans up ALL subscriptions
    }
}
```

---

## Magic Numbers and Strings

### What It Is

Using unexplained literal values scattered throughout code.

### ❌ Anti-Pattern Example

```csharp
// BAD: What do these numbers mean?
public void Attack()
{
    if (Time.time - lastAttack > 0.5f)  // What's 0.5?
    {
        animator.SetTrigger("Attack1");  // Magic string
        
        var hits = Physics.OverlapSphere(transform.position, 2.5f);  // What's 2.5?
        foreach (var hit in hits)
        {
            var health = hit.GetComponent<Health>();
            health?.TakeDamage(25);  // What's 25?
        }
        
        if (Random.value < 0.15f)  // What's 0.15?
        {
            DoCriticalHit();
        }
    }
}
```

**Problems:**
- No idea what values represent
- To change attack range, must search entire codebase
- Easy to use wrong value
- Designer can't tweak without programmer

### ✅ Boss Room Solution: ActionConfig ScriptableObject

```csharp
// GOOD: All values in data, with meaningful names
[CreateAssetMenu(menuName = "Game/Action Config")]
public class ActionConfig : ScriptableObject
{
    [Tooltip("Time before this action can be used again")]
    public float ReuseTimeSeconds;
    
    [Tooltip("How far the attack reaches")]
    public float Range;
    
    [Tooltip("Base damage dealt")]
    public int Amount;
    
    [Tooltip("Animation trigger name")]
    public string AnimationTrigger;
    
    [Tooltip("Chance for critical hit (0-1)")]
    [Range(0f, 1f)]
    public float CriticalChance;
}

// Now code reads from config:
public override bool OnUpdate(ServerCharacter parent)
{
    if (TimeRunning >= Config.ReuseTimeSeconds)
    {
        var hits = Physics.OverlapSphere(parent.Position, Config.Range);
        // ...
    }
}
```

---

## Premature Optimization vs Ignoring Performance

### Two Opposite Anti-Patterns

**Anti-Pattern 1: Premature Optimization**
```csharp
// BAD: Micro-optimizing before profiling
public class Enemy
{
    // "I'll cache everything just in case!"
    private static readonly int AttackHash = Animator.StringToHash("Attack");
    private Transform cachedTransform;
    private Vector3[] preallocatedPathBuffer = new Vector3[100];
    private StringBuilder reusableStringBuilder = new StringBuilder(256);
    
    // Code is now complex and hard to read for marginal gain
}
```

**Anti-Pattern 2: Ignoring Performance**
```csharp
// BAD: Doing expensive things every frame
void Update()
{
    var enemies = FindObjectsOfType<Enemy>();  // O(n) search every frame!
    
    foreach (var enemy in enemies)
    {
        var distance = Vector3.Distance(transform.position, enemy.transform.position);
        // Creates garbage on every iteration...
    }
}
```

### ✅ The Balanced Approach

1. **Write clean code first**
2. **Profile to find actual bottlenecks**
3. **Optimize only what matters**
4. **Know common pitfalls** (FindObjectOfType, GetComponent in Update)

Boss Room's balance:
```csharp
// Clean code that avoids KNOWN performance issues
public class NetworkObjectPool : NetworkBehaviour
{
    // Object pooling for networked objects (known to be spawned frequently)
    // This is a justified optimization - spawning causes network traffic AND GC
}
```

---

## Unity-Specific Anti-Patterns

### 1. Update() Overuse

```csharp
// BAD: Logic in Update that doesn't need to run every frame
void Update()
{
    if (Input.GetKeyDown(KeyCode.Space))  // Fine
    {
        DoSomething();
    }
    
    healthBar.value = currentHealth;  // BAD: Only needs to update when health changes!
}

// GOOD: Event-driven updates
void OnHealthChanged(int newHealth)
{
    healthBar.value = newHealth;  // Only when actually changed
}
```

### 2. Coroutine Abuse

```csharp
// BAD: Starting coroutines repeatedly
void Update()
{
    if (shouldAnimate)
    {
        StartCoroutine(AnimateCoroutine());  // New coroutine every frame!
    }
}

// GOOD: Track coroutine state
private Coroutine m_AnimationCoroutine;

void StartAnimation()
{
    if (m_AnimationCoroutine == null)
    {
        m_AnimationCoroutine = StartCoroutine(AnimateCoroutine());
    }
}
```

### 3. String Concatenation in Hot Paths

```csharp
// BAD: Creates garbage every frame
void Update()
{
    debugText.text = "Health: " + health + " / " + maxHealth;  // 3 allocations!
}

// GOOD: Use StringBuilder or avoid in Update
private StringBuilder sb = new StringBuilder();

void UpdateDebugText()  // Called only when needed
{
    sb.Clear();
    sb.Append("Health: ").Append(health).Append(" / ").Append(maxHealth);
    debugText.text = sb.ToString();
}
```

---

## The Refactoring Checklist

When reviewing code, check for:

| Anti-Pattern | Warning Signs | Fix |
|--------------|---------------|-----|
| God Class | Class > 500 lines, many responsibilities | Extract classes |
| Singleton Abuse | `Instance` everywhere | Use DI |
| Runtime Finding | `FindObjectOfType` in Update | Cache or inject |
| Tight Coupling | Many direct references | Use events/interfaces |
| Leaky Subscriptions | Subscribe without unsubscribe | Use IDisposable |
| Magic Values | Unexplained literals | Use config/constants |
| Update Overuse | Logic in Update that's event-driven | Use events |

---

## Key Takeaways

1. **Single Responsibility** - Each class should have ONE reason to change
2. **Explicit Dependencies** - If a class needs something, make it obvious
3. **Decouple with Events** - Publishers and subscribers don't know each other
4. **Always Clean Up** - Unsubscribe, dispose, release
5. **Configure, Don't Hardcode** - Values in data, not code
6. **Profile Before Optimizing** - But know common pitfalls

---

> **Next:** [15_testability_debugging.md](./15_testability_debugging.md) — How architecture enables testing
