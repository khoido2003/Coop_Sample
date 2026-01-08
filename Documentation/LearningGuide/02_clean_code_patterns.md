# 02: Clean Code Patterns

> **Purpose:** Learn clean code practices demonstrated in Boss Room that make code readable, maintainable, and testable. These principles apply to ANY codebase.

---

## Table of Contents

1. [The SOLID Principles In-Depth](#the-solid-principles-in-depth)
2. [Naming Conventions](#naming-conventions)
3. [Code Organization Patterns](#code-organization-patterns)
4. [Writing Self-Documenting Code](#writing-self-documenting-code)
5. [Error Handling Patterns](#error-handling-patterns)
6. [Code Smells and Fixes](#code-smells-and-fixes)
7. [Boss Room Code Examples](#boss-room-code-examples)

---

## The SOLID Principles In-Depth

### S - Single Responsibility Principle

**Definition:** A class should have only ONE reason to change.

**The Question to Ask:** "If I describe what this class does, do I use the word 'and'?"

```csharp
// ❌ BAD: Multiple responsibilities
public class PlayerController : MonoBehaviour
{
    // Movement responsibility
    public void Move(Vector3 direction) { }
    
    // Health responsibility
    public void TakeDamage(int amount) { }
    public void Heal(int amount) { }
    
    // Combat responsibility
    public void Attack() { }
    public void UseAbility(int index) { }
    
    // Inventory responsibility
    public void AddItem(Item item) { }
    public void RemoveItem(Item item) { }
    
    // Save responsibility
    public void Save() { }
    public void Load() { }
    
    // Animation responsibility
    public void PlayAnimation(string name) { }
}

// ✅ GOOD: Boss Room approach - separate classes
public class ServerCharacter : NetworkBehaviour       // Orchestrates, delegates
public class ServerCharacterMovement : NetworkBehaviour // ONLY movement
public class NetworkHealthState : NetworkBehaviour     // ONLY health sync
public class ServerActionPlayer                        // ONLY action execution
public class ClientCharacter : NetworkBehaviour        // ONLY visual feedback
```

**Benefits:**
- Change health system → only touch health classes
- Different developers can work on different systems
- Easier to test individual responsibilities
- Smaller, more readable files

**Boss Room Examples:**
- [ServerCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs) - Orchestrates but delegates actual work
- [ServerCharacterMovement.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacterMovement.cs) - Only movement

---

### O - Open/Closed Principle

**Definition:** Open for extension, closed for modification.

**The Question to Ask:** "Can I add new behavior without changing existing code?"

```csharp
// ❌ BAD: Must modify this switch for every new action
public class ActionExecutor
{
    public void Execute(string actionName, Character character)
    {
        switch (actionName)
        {
            case "Melee":
                character.PlayAnimation("Melee");
                character.DealDamage(10);
                break;
            case "Fireball":
                character.PlayAnimation("Cast");
                SpawnProjectile("Fireball");
                break;
            case "Heal":
                character.PlayAnimation("Heal");
                character.Heal(20);
                break;
            // Must add case for every new action!
        }
    }
}

// ✅ GOOD: Boss Room approach - abstract base, concrete extensions
public abstract class Action : ScriptableObject
{
    public abstract bool OnStart(ServerCharacter character);
    public abstract bool OnUpdate(ServerCharacter character);
    public virtual void Cancel(ServerCharacter character) { }
}

// Add new actions without touching Action.cs!
public class MeleeAction : Action { /* implementation */ }
public class ProjectileAction : Action { /* implementation */ }
public class HealAction : Action { /* implementation */ }
public class BuffAction : Action { /* implementation */ }
// ... infinite extensibility
```

**Benefits:**
- Adding features = adding files, not modifying existing code
- Reduced risk of breaking existing functionality
- Better for team development

**Boss Room Examples:**
- [Action.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/Action.cs) - Abstract base
- [MeleeAction.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.cs) - Extension

---

### L - Liskov Substitution Principle

**Definition:** Derived classes can replace base classes without breaking code.

**The Question to Ask:** "Can I use any subclass where the base class is expected?"

```csharp
// ❌ BAD: Subclass breaks expected behavior
public class Bird
{
    public virtual void Fly() { /* flap wings */ }
}

public class Penguin : Bird
{
    public override void Fly() 
    { 
        throw new NotImplementedException("Penguins can't fly!");
    }
}

// Code that expects Bird to fly breaks with Penguin!
foreach (Bird bird in birds)
{
    bird.Fly(); // Throws for Penguin!
}

// ✅ GOOD: All subclasses fulfill the contract
public abstract class Action
{
    public abstract bool OnStart(ServerCharacter character);
    public abstract bool OnUpdate(ServerCharacter character);
}

// All action subclasses work the same way:
Action action = GetAction(); // Could be Melee, Fireball, Heal...
if (action.OnStart(character)) // Works for ANY action
{
    activeActions.Add(action);
}
```

**Boss Room Example:**
```csharp
// ActionFactory returns Action instances
// The system doesn't care which concrete type it is
Action action = m_Factory.CreateAction(actionType);
action.OnStart(character); // Works for MeleeAction, HealAction, etc.
```

---

### I - Interface Segregation Principle

**Definition:** Many small interfaces > one big interface.

**The Question to Ask:** "Does implementing this interface force me to add methods I don't need?"

```csharp
// ❌ BAD: One massive interface
public interface IGameObject
{
    void Move(Vector3 direction);
    void Attack();
    void TakeDamage(int amount);
    void ReceiveHP(int amount);
    void UpdateAI();
    void Interact();
    void Save();
    void Load();
}

// A door implements IGameObject but doesn't need Move, Attack, UpdateAI...
public class Door : IGameObject
{
    public void Move(Vector3 direction) { /* not applicable */ }
    public void Attack() { /* not applicable */ }
    public void UpdateAI() { /* not applicable */ }
    // ... lots of empty methods
}

// ✅ GOOD: Boss Room approach - focused interfaces
public interface IDamageable
{
    void ReceiveHP(ServerCharacter source, int amount);
}

public interface ITargetable
{
    bool IsValidTarget { get; }
    Transform transform { get; }
}

public interface IInteractable
{
    void Interact(ServerCharacter character);
}

// Classes implement only what they need:
public class Enemy : MonoBehaviour, IDamageable, ITargetable { }
public class Door : MonoBehaviour, IInteractable { }
public class Decoration : MonoBehaviour { } // No interfaces needed
```

**Boss Room Examples:**
- `IDamageable` - Anything that can take damage
- `ITargetable` - Anything that can be targeted

---

### D - Dependency Inversion Principle

**Definition:** Depend on abstractions, not concretions.

**The Question to Ask:** "Am I depending on a specific implementation or an interface?"

```csharp
// ❌ BAD: Direct dependency on concrete class
public class Enemy
{
    private JsonSaveSystem saveSystem = new JsonSaveSystem();
    
    public void Die()
    {
        saveSystem.Save("kills", GetKillData());
        // What if we want to use CloudSaveSystem instead?
        // Must modify this class!
    }
}

// ✅ GOOD: Depend on abstraction
public interface ISaveSystem
{
    void Save(string key, object data);
    T Load<T>(string key);
}

public class Enemy
{
    private ISaveSystem saveSystem;  // Abstraction!
    
    // Injected via constructor or DI
    public Enemy(ISaveSystem saveSystem)
    {
        this.saveSystem = saveSystem;
    }
    
    public void Die()
    {
        saveSystem.Save("kills", GetKillData());
        // Works with any ISaveSystem implementation!
    }
}
```

**Boss Room Example:**
```csharp
// Boss Room uses VContainer for DI
public class HostingState : ConnectionState
{
    // Injected interfaces - not concrete classes
    [Inject] protected IPublisher<ConnectionEventMessage> m_ConnectionEventPublisher;
    [Inject] protected SessionManager m_SessionManager;
}
```

---

## Naming Conventions

### Classes

| Prefix/Suffix | Meaning | Example |
|---------------|---------|---------|
| `Server*` | Runs on server only | `ServerCharacter`, `ServerWaveSpawner` |
| `Client*` | Runs on client only | `ClientCharacter`, `ClientClickFeedback` |
| `Network*` | Syncs over network | `NetworkHealthState`, `NetworkLifeState` |
| `*State` | State in a state machine | `OfflineState`, `HostingState` |
| `*Manager` | Coordinates multiple entities | `ConnectionManager`, `SessionManager` |
| `*Controller` | Handles input/output | `ApplicationController` |
| `*Handler` | Processes events/messages | `ClientAnimationHandler` |
| `*Factory` | Creates objects | `ActionFactory`, `CharacterFactory` |
| `*Config` | Data-only ScriptableObject | `ActionConfig`, `CharacterClass` |
| `*Facade` | Simplifies complex subsystem | `AuthenticationServiceFacade` |
| `I*` | Interface | `IDamageable`, `IPublisher` |

### Methods

| Pattern | When to Use | Examples |
|---------|-------------|----------|
| `VerbNoun` | Actions | `ReceiveHP`, `PlayAction`, `SpawnCharacter` |
| `Is/Has/Can*` | Boolean returns | `IsServer`, `HasTarget`, `CanMove` |
| `On*` | Event handlers | `OnHealthChanged`, `OnClientConnected` |
| `Try*` | May fail, returns bool | `TryGetValue`, `TryConnect` |
| `Get*` | Returns a value | `GetDamage`, `GetConfig` |
| `Set*` | Sets a value | `SetHealth`, `SetPosition` |

### Variables

```csharp
// Private fields with m_ prefix
private int m_CurrentHealth;
private bool m_IsActive;
private List<Action> m_ActiveActions = new();

// Serialized fields for inspector (also m_ prefix)
[SerializeField] private GameObject m_Prefab;
[SerializeField] private float m_Speed;
[SerializeField] private Transform m_SpawnPoint;

// Constants in UPPER_CASE
private const int MAX_PLAYERS = 8;
private const float RESPAWN_TIME = 3.0f;
private const string PLAYER_TAG = "Player";

// Static with s_ prefix
private static ConnectionManager s_Instance;
private static int s_TotalEnemiesSpawned;

// Parameters and locals in camelCase
public void TakeDamage(int damageAmount)
{
    int finalDamage = damageAmount - armor;
}
```

---

## Code Organization Patterns

### File Organization

```csharp
// Recommended order in a file:
using System;
using System.Collections.Generic;
using UnityEngine;

namespace Unity.BossRoom.Gameplay
{
    /// <summary>
    /// Brief description of what this class does.
    /// </summary>
    public class ExampleClass : MonoBehaviour
    {
        // ═══════════════════════════════════════════════════════════
        // CONSTANTS
        // ═══════════════════════════════════════════════════════════
        
        private const int MAX_VALUE = 100;
        
        // ═══════════════════════════════════════════════════════════
        // SERIALIZED FIELDS
        // ═══════════════════════════════════════════════════════════
        
        [Header("References")]
        [SerializeField] private Transform m_Target;
        
        [Header("Settings")]
        [SerializeField] private float m_Speed = 5f;
        
        // ═══════════════════════════════════════════════════════════
        // PRIVATE FIELDS
        // ═══════════════════════════════════════════════════════════
        
        private int m_CurrentValue;
        
        // ═══════════════════════════════════════════════════════════
        // PROPERTIES
        // ═══════════════════════════════════════════════════════════
        
        public int CurrentValue => m_CurrentValue;
        
        // ═══════════════════════════════════════════════════════════
        // EVENTS
        // ═══════════════════════════════════════════════════════════
        
        public event Action<int> OnValueChanged;
        
        // ═══════════════════════════════════════════════════════════
        // UNITY LIFECYCLE
        // ═══════════════════════════════════════════════════════════
        
        private void Awake() { }
        private void Start() { }
        private void Update() { }
        private void OnDestroy() { }
        
        // ═══════════════════════════════════════════════════════════
        // PUBLIC METHODS
        // ═══════════════════════════════════════════════════════════
        
        public void DoSomething() { }
        
        // ═══════════════════════════════════════════════════════════
        // PRIVATE METHODS
        // ═══════════════════════════════════════════════════════════
        
        private void HelperMethod() { }
    }
}
```

### Partial Classes for Large Files

```csharp
// MeleeAction.cs - Server logic
public partial class MeleeAction : Action
{
    public override bool OnStart(ServerCharacter parent) { /* server code */ }
    public override bool OnUpdate(ServerCharacter parent) { /* server code */ }
}

// MeleeAction.Client.cs - Client visual logic
public partial class MeleeAction
{
    public override void OnStartClient(ClientCharacter client) { /* client visuals */ }
    public override void OnUpdateClient(ClientCharacter client) { /* client visuals */ }
}
```

---

## Writing Self-Documenting Code

### Bad vs Good Examples

```csharp
// ❌ BAD: Needs comment to explain
int t = 30; // timeout in seconds
if (p > 0 && h <= 0) { R(); } // respawn if points positive and health zero

// ✅ GOOD: Self-documenting
int timeoutSeconds = 30;
bool hasPointsButNeedsRespawn = points > 0 && health <= 0;
if (hasPointsButNeedsRespawn) 
{ 
    Respawn(); 
}
```

### Guard Clauses (Early Return)

```csharp
// ❌ BAD: Deep nesting
void ProcessDamage(Character source, int damage)
{
    if (isAlive)
    {
        if (!isInvincible)
        {
            if (damage > 0)
            {
                if (source != null)
                {
                    ApplyDamage(damage);
                    NotifySource(source);
                }
            }
        }
    }
}

// ✅ GOOD: Guard clauses flatten the code
void ProcessDamage(Character source, int damage)
{
    if (!isAlive) return;
    if (isInvincible) return;
    if (damage <= 0) return;
    if (source == null) return;
    
    ApplyDamage(damage);
    NotifySource(source);
}
```

### When to Comment

```csharp
// ✅ DO: Comment WHY, not WHAT
// We add a small delay to prevent race condition with network callbacks
yield return new WaitForSeconds(0.1f);

// ✅ DO: Comment complex algorithms
// Using A* pathfinding with Manhattan distance heuristic for grid-based movement
// Reference: https://www.redblobgames.com/pathfinding/a-star/

// ✅ DO: Comment workarounds
// HACK: Unity's NavMesh has a bug where agents get stuck at corners.
// We force a small offset to work around this.
agent.destination += Vector3.forward * 0.01f;

// ❌ DON'T: Comment obvious code
i++; // increment i
player.Health = 100; // set health to 100
```

---

## Error Handling Patterns

### Null Checking

```csharp
// ❌ BAD: Implicit null assumption
target.TakeDamage(damage); // NullReferenceException if target is null!

// ✅ GOOD: Null-conditional
target?.TakeDamage(damage);

// ✅ GOOD: Explicit check with early return
if (target == null) return;
target.TakeDamage(damage);

// ✅ GOOD: TryGet pattern
if (m_Targets.TryGetValue(targetId, out var target))
{
    target.TakeDamage(damage);
}
```

### Defensive Programming

```csharp
public void SetHealth(int value)
{
    // Validate input
    if (value < 0)
    {
        Debug.LogWarning($"Attempted to set negative health: {value}");
        value = 0;
    }
    
    if (value > maxHealth)
    {
        value = maxHealth;
    }
    
    m_Health = value;
}
```

### Assert for Development

```csharp
void Initialize(GameConfig config)
{
    // Assert catches bugs during development
    Debug.Assert(config != null, "GameConfig is required!");
    Debug.Assert(config.MaxPlayers > 0, "MaxPlayers must be positive!");
    
    m_Config = config;
}
```

---

## Code Smells and Fixes

| Smell | Problem | Fix |
|-------|---------|-----|
| God Class | Class > 500 lines, many responsibilities | Split into smaller classes |
| Long Method | Method > 30 lines | Extract smaller methods |
| Long Parameter List | Method with 5+ parameters | Create parameter object |
| Magic Numbers | `if (x > 42)` | Use named constant |
| Duplicate Code | Same code in multiple places | Extract to shared method |
| Dead Code | Commented-out or unreachable code | Delete it |
| Feature Envy | Method uses another class more than its own | Move method to that class |
| Data Clumps | Same data always passed together | Create a class/struct |

---

## Boss Room Code Examples

### Clean Naming Example
```csharp
// From ServerCharacter.cs
public void ReceiveHP(ServerCharacter inflicter, int HP)
{
    // Clear names: inflicter = who caused this, HP = amount
    // Positive HP = heal, Negative HP = damage
}
```

### Interface Segregation Example
```csharp
// From IDamageable.cs - Single focused interface
public interface IDamageable
{
    void ReceiveHP(ServerCharacter inflicter, int HP);
}

// Characters, enemies, destructibles all implement this ONE method
```

### Guard Clause Example
```csharp
// From Action.cs
public override bool OnUpdate(ServerCharacter parent)
{
    if (!IsClient) return true;
    if (m_ClientVisualization == null) return true;
    
    // Main logic here
}
```

---

## Exercises

1. **Find SOLID violations:** Look through your own code and identify one violation of each SOLID principle
2. **Rename:** Take a class with poor naming and rename all variables/methods to be self-documenting
3. **Extract class:** Take a "God class" and split it into at least 3 smaller classes
4. **Add guard clauses:** Find a deeply nested method and flatten it with guard clauses
5. **Remove magic numbers:** Find all magic numbers in a file and replace with named constants

---

> **Cross-Reference:**
> - [14_antipatterns_guide.md](./14_antipatterns_guide.md) - What NOT to do
> - [01_architecture_principles.md](./01_architecture_principles.md) - Architecture patterns
> - [04_design_patterns.md](./04_design_patterns.md) - Design patterns in detail
