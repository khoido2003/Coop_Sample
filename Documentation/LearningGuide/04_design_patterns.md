# 04: Design Patterns Catalog

> **Goal:** Recognize and apply design patterns used in Boss Room. These patterns work in ANY programming context.

---

## Pattern 1: State Machine

**Problem:** Object behavior depends on its current state.

**Solution:** Encapsulate each state in a class with Enter, Update, Exit methods.

```csharp
// Base state
public abstract class State {
    public virtual void Enter() { }
    public virtual void Update() { }
    public virtual void Exit() { }
}

// Concrete states
public class IdleState : State {
    public override void Update() {
        if (Input.GetKeyDown(KeyCode.Space)) {
            stateMachine.ChangeState(jumpState);
        }
    }
}

// Controller
public class StateMachine {
    State currentState;
    
    public void ChangeState(State newState) {
        currentState?.Exit();
        currentState = newState;
        currentState.Enter();
    }
    
    public void Update() {
        currentState?.Update();
    }
}
```

**In Boss Room:** `ConnectionState`, `GameStateBehaviour`

**Use when:** Player states, AI states, menu flows, game phases

---

## Pattern 2: Observer (PubSub)

**Problem:** Objects need to react to events without tight coupling.

**Solution:** Publishers send events, subscribers receive them.

```csharp
// Simple implementation
public class GameEvents {
    public static event Action<int> OnScoreChanged;
    public static event Action OnPlayerDeath;
    
    public static void RaiseScoreChanged(int score) {
        OnScoreChanged?.Invoke(score);
    }
}

// Publisher
GameEvents.RaiseScoreChanged(100);

// Subscriber
void Start() {
    GameEvents.OnScoreChanged += UpdateScoreUI;
}
void OnDestroy() {
    GameEvents.OnScoreChanged -= UpdateScoreUI;  // Always unsubscribe!
}
```

**In Boss Room:** `MessageChannel<T>`, `ISubscriber<T>`

**Use when:** UI updates, achievements, sound triggers, analytics

---

## Pattern 3: Factory

**Problem:** Object creation is complex or needs to be abstracted.

**Solution:** Centralized factory creates objects.

```csharp
public class EnemyFactory {
    static Dictionary<string, GameObject> prefabs = new();
    
    public static Enemy Create(string type, Vector3 position) {
        var prefab = prefabs[type];
        var enemy = Object.Instantiate(prefab, position, Quaternion.identity);
        return enemy.GetComponent<Enemy>();
    }
}

// Usage
var goblin = EnemyFactory.Create("Goblin", spawnPoint);
```

**In Boss Room:** `ActionFactory`

**Use when:** Many object types, pooling needed, complex initialization

---

## Pattern 4: Object Pool

**Problem:** Creating/destroying objects causes garbage collection spikes.

**Solution:** Reuse objects from a pool.

```csharp
public class ObjectPool<T> where T : Component {
    Queue<T> pool = new();
    T prefab;
    
    public T Get() {
        if (pool.Count > 0) {
            var obj = pool.Dequeue();
            obj.gameObject.SetActive(true);
            return obj;
        }
        return Object.Instantiate(prefab);
    }
    
    public void Return(T obj) {
        obj.gameObject.SetActive(false);
        pool.Enqueue(obj);
    }
}
```

**In Boss Room:** `NetworkObjectPool`

**Use when:** Bullets, particles, UI elements, enemies

---

## Pattern 5: Singleton

**Problem:** Need exactly one instance accessible globally.

**Solution:** Static instance with controlled access.

```csharp
public class GameManager : MonoBehaviour {
    public static GameManager Instance { get; private set; }
    
    void Awake() {
        if (Instance != null) {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }
}
```

**⚠️ Warning:** Overuse creates tight coupling. Prefer DI when possible.

**In Boss Room:** Uses DI instead of singletons for most systems.

**Use when:** Truly global (audio, save system), but prefer DI.

---

## Pattern 6: Command

**Problem:** Need to encapsulate actions as objects (for queue, undo, replay).

**Solution:** Wrap action data and execution in a class.

```csharp
public interface ICommand {
    void Execute();
    void Undo();
}

public class MoveCommand : ICommand {
    Vector3 previousPosition;
    Transform target;
    Vector3 newPosition;
    
    public void Execute() {
        previousPosition = target.position;
        target.position = newPosition;
    }
    
    public void Undo() {
        target.position = previousPosition;
    }
}

// Usage
commandQueue.Add(new MoveCommand(player, destination));
```

**In Boss Room:** `Action` system is similar (actions queued and executed)

**Use when:** Undo systems, input buffering, action queues, replays

---

## Pattern 7: Strategy

**Problem:** Need interchangeable algorithms.

**Solution:** Define algorithm interface, create implementations.

```csharp
// Strategy interface
public interface IMovementStrategy {
    void Move(Transform transform, Vector3 input);
}

// Implementations
public class WalkStrategy : IMovementStrategy {
    public void Move(Transform t, Vector3 input) {
        t.position += input * walkSpeed * Time.deltaTime;
    }
}

public class FlyStrategy : IMovementStrategy {
    public void Move(Transform t, Vector3 input) {
        t.position += input * flySpeed * Time.deltaTime;
        // No gravity
    }
}

// Usage
public class Character {
    IMovementStrategy movement;
    
    public void SetMovement(IMovementStrategy strategy) {
        movement = strategy;
    }
    
    void Update() {
        movement.Move(transform, inputVector);
    }
}
```

**In Boss Room:** `ConnectionMethod` (IP vs Relay strategies)

**Use when:** Multiple algorithms for same task, swappable behaviors

---

## Pattern 8: Mediator

**Problem:** Many objects need to communicate, but direct connections create spaghetti.

**Solution:** Central mediator coordinates communication.

```csharp
public class UIMediator : MonoBehaviour {
    [SerializeField] Button startButton;
    [SerializeField] InputField nameInput;
    [SerializeField] GameManager gameManager;
    
    void Start() {
        startButton.onClick.AddListener(OnStartClicked);
    }
    
    void OnStartClicked() {
        // Mediator coordinates between UI and game
        string name = nameInput.text;
        if (ValidateName(name)) {
            gameManager.StartGame(name);
        }
    }
}
```

**In Boss Room:** `IPUIMediator`, `SessionUIMediator`

**Use when:** Complex UI, many interdependent components

---

## Pattern Cheat Sheet

| Pattern | Use When | Boss Room Example |
|---------|----------|-------------------|
| State Machine | Complex state transitions | ConnectionState |
| Observer | Decoupled events | MessageChannel |
| Factory | Object creation | ActionFactory |
| Object Pool | Frequent create/destroy | NetworkObjectPool |
| Singleton | Global access (sparingly) | (Uses DI instead) |
| Command | Action queue/undo | Action system |
| Strategy | Swappable algorithms | ConnectionMethod |
| Mediator | Complex coordination | UI Mediators |

---

## Exercise

For each scenario, which pattern would you use?

1. Player can walk, run, swim, or fly - different physics each
2. Game state: Menu → Playing → Paused → GameOver
3. Spawning hundreds of bullets per second
4. Health bar needs to update when player takes damage
5. Building an action queue for a turn-based game

Write your answers, then find examples in Boss Room code.
