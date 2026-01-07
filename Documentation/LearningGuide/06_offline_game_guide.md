# 06: Applying Patterns to Offline Games

> **Goal:** These patterns aren't just for multiplayer - they make SINGLE-PLAYER games better too.

---

## Which Patterns Still Apply?

| Pattern | Multiplayer Use | Offline Use |
|---------|-----------------|-------------|
| State Machine | Connection states | Player states, AI, menus |
| PubSub Events | Network events | Any decoupled events |
| Dependency Injection | Service injection | Testable code |
| Object Pool | Network spawns | Bullets, particles |
| Factory | Create network objects | Create any objects |
| ScriptableObject Config | Synced game data | All game data |
| Server/Client Split | Network authority | Logic/View separation |

---

## Logic/View Separation (Offline Version of Server/Client)

Even without networking, separate LOGIC from PRESENTATION:

```csharp
// LOGIC (like ServerCharacter)
public class CharacterLogic : MonoBehaviour {
    public int Health { get; private set; }
    public event Action<int> OnHealthChanged;
    
    public void TakeDamage(int amount) {
        Health -= amount;
        OnHealthChanged?.Invoke(Health);
        
        if (Health <= 0) Die();
    }
}

// VIEW (like ClientCharacter)
public class CharacterView : MonoBehaviour {
    [SerializeField] CharacterLogic logic;
    [SerializeField] Slider healthBar;
    [SerializeField] Animator animator;
    
    void Start() {
        logic.OnHealthChanged += UpdateHealthBar;
    }
    
    void UpdateHealthBar(int health) {
        healthBar.value = health;
        animator.SetTrigger("Hit");
    }
}
```

**Benefits:**
- Logic works without visuals (for testing, headless mode)
- Can swap visual styles without touching game logic
- Easier to add multiplayer later

---

## Offline State Machine Example

```csharp
public class GameFlowManager : MonoBehaviour {
    enum GameState { MainMenu, Playing, Paused, GameOver }
    
    GameState currentState;
    
    public void ChangeState(GameState newState) {
        ExitState(currentState);
        currentState = newState;
        EnterState(currentState);
    }
    
    void EnterState(GameState state) {
        switch(state) {
            case GameState.MainMenu:
                Time.timeScale = 1;
                SceneManager.LoadScene("MainMenu");
                break;
            case GameState.Paused:
                Time.timeScale = 0;
                ShowPauseMenu();
                break;
        }
    }
}
```

---

## Offline Event System

```csharp
// Simple event hub
public static class GameEvents {
    public static event Action<int> OnCoinCollected;
    public static event Action OnPlayerDied;
    public static event Action<string> OnLevelCompleted;
    
    public static void CoinCollected(int value) => OnCoinCollected?.Invoke(value);
    public static void PlayerDied() => OnPlayerDied?.Invoke();
    public static void LevelCompleted(string levelId) => OnLevelCompleted?.Invoke(levelId);
}

// Coin publishes
public class Coin : MonoBehaviour {
    void OnTriggerEnter(Collider other) {
        GameEvents.CoinCollected(10);
        Destroy(gameObject);
    }
}

// UI subscribes
public class ScoreUI : MonoBehaviour {
    int score;
    void Start() => GameEvents.OnCoinCollected += AddScore;
    void OnDestroy() => GameEvents.OnCoinCollected -= AddScore;
    void AddScore(int value) => score += value;
}

// Audio subscribes
public class AudioManager : MonoBehaviour {
    void Start() => GameEvents.OnCoinCollected += _ => PlaySound("Coin");
}

// Analytics subscribes
public class Analytics : MonoBehaviour {
    void Start() => GameEvents.OnCoinCollected += TrackCoin;
}
```

---

## Adding Multiplayer Later

If you follow these patterns, adding multiplayer becomes much easier:

| Current (Offline) | Change for Multiplayer |
|-------------------|------------------------|
| CharacterLogic | Becomes ServerCharacter, add NetworkBehaviour |
| CharacterView | Becomes ClientCharacter, reads NetworkVariables |
| GameEvents | Replace with NetworkedMessageChannel |
| LocalSave | Add server-side validation |

**Key:** Logic/View separation is the same as Server/Client separation!
