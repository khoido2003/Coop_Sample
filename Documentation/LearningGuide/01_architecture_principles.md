# 01: Core Architecture Principles

> **Goal:** Understand the fundamental architecture decisions in this project and WHY they were made. These principles apply to ANY game.

---

## The 5 Core Principles

### 1. Separation of Concerns (SoC)

**What it means:** Each class/system does ONE thing well.

**In Boss Room:**
```
ServerCharacter    → Game logic (health, damage, actions)
ClientCharacter    → Visual feedback (animations, effects)
NetworkHealthState → Data synchronization only
```

**Why it matters:**
- Easier to modify one part without breaking others
- Easier to test individual components
- Clearer mental model of the codebase

**How to apply in YOUR game:**
```
BAD:  PlayerController that handles input, movement, health, animation, UI, saving
GOOD: 
├── PlayerInput        → Reads controller/keyboard
├── PlayerMovement     → Moves character
├── PlayerHealth       → Tracks hit points
├── PlayerAnimator     → Plays animations  
└── PlayerUI           → Shows health bar
```

---

### 2. Dependency Inversion (DI)

**What it means:** Classes depend on ABSTRACTIONS (interfaces), not concrete implementations.

**In Boss Room:**
```csharp
// BAD - Direct dependency
public class HeroActionBar 
{
    private ConnectionManager connectionManager; // Concrete class
    
    void Start() 
    {
        connectionManager = FindObjectOfType<ConnectionManager>(); // Hard to test!
    }
}

// GOOD - Dependency Injection (what Boss Room does)
public class HeroActionBar 
{
    [Inject] IConnectionManager connectionManager; // Interface
    // VContainer injects this automatically
}
```

**Why it matters:**
- Easy to swap implementations (e.g., real network vs fake for testing)
- Clearer dependencies (you can see what a class needs)
- Testable (inject mock objects)

**How to apply in YOUR game:**
```csharp
// 1. Define interfaces for major systems
public interface ISaveSystem {
    void Save(GameData data);
    GameData Load();
}

// 2. Create implementation
public class JsonSaveSystem : ISaveSystem { ... }
public class CloudSaveSystem : ISaveSystem { ... }

// 3. Inject where needed
public class GameManager {
    [Inject] ISaveSystem saveSystem; // Works with any implementation
}
```

---

### 3. State Machine Pattern

**What it means:** Complex objects have explicit STATES with defined TRANSITIONS.

**In Boss Room:**
```
Connection States:
    Offline → Connecting → Connected → Reconnecting → Offline

Game States:
    MainMenu → CharSelect → BossRoom → PostGame
```

**Why it matters:**
- Clear understanding of what's possible in each state
- Prevents illegal state combinations
- Easy to add new states later

**How to apply in YOUR game (even offline):**
```csharp
// Player states for a platformer
public enum PlayerState { Idle, Running, Jumping, Falling, Climbing, Dead }

public class PlayerStateMachine {
    PlayerState currentState;
    
    public void ChangeState(PlayerState newState) {
        ExitState(currentState);
        currentState = newState;
        EnterState(currentState);
    }
    
    public void Update() {
        // Each state handles its own logic
        switch(currentState) {
            case PlayerState.Jumping: UpdateJumping(); break;
            // ...
        }
    }
}
```

---

### 4. Event-Driven Communication (PubSub)

**What it means:** Objects communicate through events/messages, not direct calls.

**In Boss Room:**
```csharp
// Publisher (doesn't know who's listening)
healthChannel.Publish(new HealthChangedMessage { newHealth = 50 });

// Subscriber (doesn't know who's publishing)
healthChannel.Subscribe(msg => UpdateHealthBar(msg.newHealth));
```

**Why it matters:**
- Publishers and subscribers don't know about each other
- Easy to add new reactions to events
- No circular dependencies

**How to apply in YOUR game:**
```csharp
// Simple event system for offline games
public static class GameEvents {
    public static event Action<int> OnScoreChanged;
    public static event Action OnPlayerDied;
    public static event Action<string> OnLevelCompleted;
    
    public static void ScoreChanged(int score) => OnScoreChanged?.Invoke(score);
    public static void PlayerDied() => OnPlayerDied?.Invoke();
}

// Usage
GameEvents.OnPlayerDied += ShowGameOverScreen;
GameEvents.OnPlayerDied += SaveHighScore;
GameEvents.OnPlayerDied += PlayDeathSound;
```

---

### 5. Data-Driven Design

**What it means:** Game behavior is controlled by DATA (config files), not hard-coded values.

**In Boss Room:**
```
ActionConfig (ScriptableObject):
├── Damage: 25
├── Cooldown: 2.5
├── Range: 5.0
├── Animation: "Attack01"
└── VFX: FireballPrefab
```

**Why it matters:**
- Designers can tweak without programmers
- Easy to create variations (Fireball Level 1, 2, 3)
- Can change balance without recompiling

**How to apply in YOUR game:**
```csharp
[CreateAssetMenu(menuName = "Game/Enemy Config")]
public class EnemyConfig : ScriptableObject {
    public string enemyName;
    public int maxHealth;
    public float moveSpeed;
    public int damage;
    public float attackRange;
    public GameObject prefab;
}

public class Enemy : MonoBehaviour {
    [SerializeField] EnemyConfig config;
    
    void Start() {
        health = config.maxHealth;
        // All behavior reads from config
    }
}
```

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                          │
│  Entry Point: ApplicationController                               │
│  Responsibility: Bootstrap, DI setup, scene management           │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                         DOMAIN LAYER                              │
│  Game Rules: Actions, Characters, GameState                       │
│  Responsibility: Core game logic (platform-independent)          │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                      INFRASTRUCTURE LAYER                         │
│  Services: Networking, Saving, Audio, Pooling                    │
│  Responsibility: Technical services, external APIs               │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                       PRESENTATION LAYER                          │
│  UI, VFX, Audio, Animations                                       │
│  Responsibility: Everything the player sees/hears                │
└──────────────────────────────────────────────────────────────────┘
```

**Rule:** Upper layers can call lower layers. Never the reverse.

---

## Quick Reference

| Principle | One-Line Summary | Boss Room Example |
|-----------|------------------|-------------------|
| Separation of Concerns | One class = one job | Server/Client split |
| Dependency Inversion | Depend on interfaces | VContainer injection |
| State Machine | Explicit states & transitions | ConnectionState classes |
| Event-Driven | Communicate through events | MessageChannel system |
| Data-Driven | Config in data, not code | ActionConfig ScriptableObjects |

---

## Exercise

Open these files and identify which principles they demonstrate:

1. `ConnectionManager.cs` - Which principle?
2. `ActionConfig.cs` - Which principle?
3. `MessageChannel.cs` - Which principle?
4. `ServerCharacter.cs` vs `ClientCharacter.cs` - Which principle?

Write your answers in a notebook before checking the code comments.
