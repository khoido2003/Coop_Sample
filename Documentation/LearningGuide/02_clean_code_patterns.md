# 02: Clean Code Patterns

> **Goal:** Learn clean code practices demonstrated in Boss Room that make code readable, maintainable, and testable.

---

## The SOLID Principles in Boss Room

### S - Single Responsibility Principle

**Each class has ONE reason to change.**

```csharp
// Boss Room does this:
ServerCharacter       → Only game logic
ClientCharacter       → Only visual feedback  
NetworkHealthState    → Only health sync
ServerCharacterMovement → Only movement

// NOT this (bad):
GodCharacter {
    void HandleInput() { }
    void Move() { }
    void Attack() { }
    void TakeDamage() { }
    void PlayAnimation() { }
    void UpdateUI() { }
    void SaveGame() { }
    void PlayAudio() { }
}
```

**Rule of thumb:** If your class name needs "And" or "Manager", it's too big.

---

### O - Open/Closed Principle

**Open for extension, closed for modification.**

```csharp
// Boss Room - Adding new actions doesn't modify existing code
public abstract class Action {
    public abstract bool OnStart(ServerCharacter c);
    public abstract bool OnUpdate(ServerCharacter c);
}

public class MeleeAction : Action { /* new action */ }
public class ProjectileAction : Action { /* new action */ }
public class HealAction : Action { /* new action */ }
// Each new action is a NEW file, doesn't touch Action.cs
```

**For your games:**
```csharp
// Adding enemies without modifying EnemyManager
public interface IEnemy {
    void Attack();
    void TakeDamage(int amount);
}

public class Goblin : IEnemy { }
public class Dragon : IEnemy { }
// EnemyManager works with IEnemy, never needs to change
```

---

### L - Liskov Substitution Principle

**Derived classes can replace base classes without breaking code.**

```csharp
// Boss Room - Any Action can be used where Action is expected
void ExecuteAction(Action action) {
    if (action.OnStart(character)) {
        activeActions.Add(action);
    }
}
// Works with MeleeAction, ProjectileAction, any Action subclass
```

---

### I - Interface Segregation Principle

**Many small interfaces > one big interface.**

```csharp
// Boss Room does this:
public interface IDamageable {
    void ReceiveHP(ServerCharacter source, int amount);
}

public interface ITargetable {
    bool IsValidTarget { get; }
}

// NOT this (bad):
public interface IGameObject {
    void ReceiveHP(...);
    void Move(...);
    void Attack(...);
    void PlaySound(...);
    // Forces every class to implement everything
}
```

**For your games:**
```csharp
public interface ISaveable { void Save(); void Load(); }
public interface IPoolable { void OnSpawn(); void OnDespawn(); }
public interface IPausable { void OnPause(); void OnResume(); }

// A class only implements what it needs
public class Enemy : MonoBehaviour, IDamageable, IPoolable { }
public class Decoration : MonoBehaviour, ISaveable { }
```

---

### D - Dependency Inversion Principle

**Depend on abstractions, not concretions.**

```csharp
// Boss Room uses VContainer for this:
public class SomeSystem {
    [Inject] ISubscriber<HealthChangedMessage> healthSubscriber;
    [Inject] IPublisher<DamageMessage> damagePublisher;
    // Depends on interfaces, actual implementation injected
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
| `*State` | A state in state machine | `OfflineState`, `HostingState` |
| `*Manager` | Coordinates multiple things | `ConnectionManager`, `PopupManager` |
| `*Facade` | Simplifies complex subsystem | `AuthenticationServiceFacade` |
| `*Config` | Data-only, ScriptableObject | `ActionConfig`, `CharacterConfig` |
| `I*` | Interface | `IDamageable`, `ITargetable` |

### Methods

```csharp
// Start with verb
ReceiveHP(), PlayAction(), StartHost(), CreateSession()

// Boolean methods/properties start with Is/Has/Can
IsServer, IsConnected, HasTarget, CanMove()

// Event handlers start with On
OnHealthChanged(), OnClientConnected(), OnButtonClicked()

// Private methods start with lowercase
calculateDamage(), spawnEnemy(), updateUI()
```

### Variables

```csharp
// Private fields with m_ prefix
private int m_CurrentHealth;
private bool m_IsActive;

// Serialized fields for inspector
[SerializeField] GameObject m_Prefab;
[SerializeField] float m_Speed;

// Constants UPPER_CASE
const int MAX_PLAYERS = 8;
const float RESPAWN_TIME = 3.0f;

// Static with s_ prefix
static ConnectionManager s_Instance;
```

---

## Code Organization Patterns

### Partial Classes for Client/Server Split

```csharp
// MeleeAction.cs - Server logic
public partial class MeleeAction : Action {
    public override bool OnStart(ServerCharacter c) { }
    public override bool OnUpdate(ServerCharacter c) { }
}

// MeleeAction.Client.cs - Client visual logic
public partial class MeleeAction {
    public override void OnStartClient(ClientCharacter c) { }
    public override void OnUpdateClient(ClientCharacter c) { }
}
```

**Why:** Keeps server logic separate from client visuals, easier to read.

---

### Region-Free, Comment-Light Code

Boss Room uses **self-documenting code** instead of comments:

```csharp
// BAD - Needs comment to explain
int x = 5; // Max number of retries

// GOOD - Self-documenting
int maxReconnectAttempts = 5;

// BAD - Comment explains what code does
// Check if player is dead and respawn them
if (h <= 0) { Respawn(); }

// GOOD - Code explains itself
if (health <= 0) { 
    HandleDeath(); 
}
```

**When to comment:**
- WHY, not WHAT (code shows what, comments explain why)
- Complex algorithms
- Workarounds for bugs/limitations
- Public API documentation (`///`)

---

### Guard Clauses (Early Return)

```csharp
// BAD - Deep nesting
void ProcessDamage(int damage) {
    if (isAlive) {
        if (!isInvincible) {
            if (damage > 0) {
                health -= damage;
            }
        }
    }
}

// GOOD - Guard clauses (Boss Room style)
void ProcessDamage(int damage) {
    if (!isAlive) return;
    if (isInvincible) return;
    if (damage <= 0) return;
    
    health -= damage;
}
```

---

### Null Safety

```csharp
// Always check before using
if (target != null) {
    target.ReceiveHP(this, -damage);
}

// Or use null-conditional
target?.ReceiveHP(this, -damage);

// TryGet pattern
if (m_ClientIdToSlot.TryGetValue(clientId, out var slot)) {
    slot.UpdateHealth(health);
}
```

---

## Quick Reference Checklist

When writing code, ask yourself:

- [ ] Does this class have only ONE responsibility?
- [ ] Can I add new features without modifying existing code?
- [ ] Am I depending on interfaces, not concrete classes?
- [ ] Is the naming clear without needing comments?
- [ ] Are my methods short and focused?
- [ ] Did I handle null cases?
- [ ] Would someone else understand this in 6 months?

---

## Exercise

Find these patterns in the codebase:

1. Find a `Server*` and `Client*` pair - how do they differ?
2. Find a partial class split across files
3. Find an interface with multiple implementations
4. Find a guard clause (early return)
5. Find a ScriptableObject config

Write down the file names and line numbers.
