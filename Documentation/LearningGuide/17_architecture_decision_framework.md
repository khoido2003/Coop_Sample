# 17: Architecture Decision Framework

> **Purpose:** The "meta-skill" of knowing WHEN to use WHICH pattern. This guide helps you make architectural decisions for ANY game project.

---

## Table of Contents

1. [The Decision-Making Process](#the-decision-making-process)
2. [Pattern Selection Flowcharts](#pattern-selection-flowcharts)
3. [Project Scale Considerations](#project-scale-considerations)
4. [Genre-Specific Guidance](#genre-specific-guidance)
5. [Team Size Considerations](#team-size-considerations)
6. [When to Add Complexity](#when-to-add-complexity)
7. [Migration Paths](#migration-paths)
8. [New Project Checklist](#new-project-checklist)

---

## The Decision-Making Process

### The Three Questions

Before choosing any pattern, ask:

```
1. WHAT PROBLEM AM I SOLVING?
   - Be specific about the actual issue
   - Don't add patterns "just in case"

2. WHAT ARE THE TRADEOFFS?
   - Every pattern has costs (complexity, learning curve)
   - Benefits must outweigh costs

3. WHAT'S THE SIMPLEST SOLUTION?
   - Start simple, add complexity when needed
   - "Premature abstraction" is as bad as "no abstraction"
```

### The YAGNI Principle

**You Aren't Gonna Need It** - Don't add patterns for hypothetical future needs.

```csharp
// BAD: "We might need multiple save systems someday"
interface ISaveSystem { }
class CloudSaveSystem : ISaveSystem { }
class LocalSaveSystem : ISaveSystem { }
class SaveSystemFactory { }
class SaveSystemRouter { }
// ... for a game that will only ever use local saves

// GOOD: Just build what you need now
class SaveSystem
{
    public void Save(GameData data) { /* local save */ }
    public GameData Load() { /* local load */ }
}
// Refactor to interface IF/WHEN you actually need cloud saves
```

---

## Pattern Selection Flowcharts

### Communication Between Objects

```
Objects need to communicate
              │
              ▼
┌─────────────────────────────────┐
│ Do they know about each other? │
└─────────────┬───────────────────┘
              │
    ┌─────────┴─────────┐
   Yes                  No
    │                   │
    ▼                   ▼
┌─────────────┐   ┌─────────────────────────────────┐
│ Direct call │   │ How many listeners?             │
│ A.DoThing() │   └─────────────┬───────────────────┘
└─────────────┘                 │
                    ┌───────────┴───────────┐
                   One                    Many
                    │                       │
                    ▼                       ▼
              ┌───────────┐         ┌─────────────────┐
              │ Callback/ │         │ PubSub/Events   │
              │ Delegate  │         │ MessageChannel  │
              └───────────┘         └─────────────────┘
```

### State Management

```
Object has different modes/states
              │
              ▼
┌────────────────────────────────────┐
│ How many states?                   │
└─────────────┬──────────────────────┘
              │
    ┌─────────┴─────────┬──────────────────┐
   2-3                 4-10               >10
    │                   │                   │
    ▼                   ▼                   ▼
┌─────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Simple bool │   │ Enum + switch   │   │ State Machine   │
│ or enum     │   │ or State Machine│   │ (definitely)    │
└─────────────┘   └─────────────────┘   └─────────────────┘

Consider State Machine if:
- Complex transitions (not just any → any)
- States have entry/exit logic
- States have different update behavior
```

### Object Creation

```
Need to create objects
              │
              ▼
┌────────────────────────────────────┐
│ How often?                         │
└─────────────┬──────────────────────┘
              │
    ┌─────────┴─────────┬──────────────────┐
  Rarely             Often              Very Often
   (once)          (per level)        (per frame)
    │                   │                   │
    ▼                   ▼                   ▼
┌─────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Direct new/ │   │ Factory         │   │ Object Pool     │
│ Instantiate │   │ (centralize)    │   │ (must have!)    │
└─────────────┘   └─────────────────┘   └─────────────────┘
```

### Configuration/Data

```
Need configurable values
              │
              ▼
┌────────────────────────────────────┐
│ Who changes them?                  │
└─────────────┬──────────────────────┘
              │
    ┌─────────┴─────────┬──────────────────┐
 Programmer        Designer           Player
    │                   │                   │
    ▼                   ▼                   ▼
┌─────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Constants/  │   │ ScriptableObj   │   │ Save System     │
│ readonly    │   │ or JSON         │   │ PlayerPrefs     │
└─────────────┘   └─────────────────┘   └─────────────────┘
```

---

## Project Scale Considerations

### Small Project (1-3 months, solo dev)

**Recommended complexity level:** LOW

```
DO:
├── Simple folder structure
├── Direct references (SerializeField)
├── C# events for decoupling
├── MonoBehaviour singletons (sparingly)
└── ScriptableObjects for data

DON'T:
├── Full DI framework
├── Complex message bus
├── Over-abstracted factories
└── 10 design patterns for 5 features
```

### Medium Project (3-12 months, 2-5 devs)

**Recommended complexity level:** MODERATE

```
DO:
├── Clear folder structure with namespaces
├── Interfaces for major systems
├── Event system (PubSub)
├── Consider DI for testability
├── Object pooling where needed
└── State machines for complex flows

DON'T:
├── Abstract everything
├── Pattern for every problem
└── Reinvent the wheel
```

### Large Project (12+ months, 5+ devs)

**Recommended complexity level:** HIGH (justified)

```
DO:
├── Full DI framework
├── Comprehensive interface design
├── Layered architecture
├── Automated testing
├── Event-driven communication
├── Object pooling everywhere
└── Design patterns applied consistently

ALSO:
├── Code review standards
├── Architecture documentation
└── Pattern guidelines for team
```

---

## Genre-Specific Guidance

### RPG / Action RPG (like Boss Room)

**Key patterns needed:**
- **Action System** - Abilities, cooldowns, effects
- **State Machines** - Player states, AI states
- **Data-Driven** - Items, skills, enemies from config
- **PubSub** - Combat events, UI updates
- **Object Pool** - Projectiles, damage numbers

### Platformer

**Key patterns needed:**
- **State Machine** - Player movement states (idle, run, jump, wall-slide)
- **Factory** - Enemy spawning
- **Data-Driven** - Level data from ScriptableObjects
- **Event System** - Collectibles, checkpoints

```csharp
// Minimal platformer architecture
class Player : MonoBehaviour
{
    enum State { Idle, Running, Jumping, Falling, WallSlide }
    State currentState;
    
    void Update() { HandleStateLogic(); }
}

class LevelManager : MonoBehaviour
{
    [SerializeField] LevelData levelData;  // ScriptableObject
    void SpawnEnemies() { /* from levelData */ }
}

static class GameEvents  // Simple events
{
    public static event Action<int> OnCoinCollected;
}
```

### Puzzle Game

**Key patterns needed:**
- **Command** - Undo/redo moves
- **State Machine** - Puzzle state
- **Data-Driven** - Puzzle definitions

```csharp
// Puzzle-specific: Command pattern for undo
interface ICommand
{
    void Execute();
    void Undo();
}

class MoveCommand : ICommand
{
    public void Execute() { /* make move */ }
    public void Undo() { /* reverse move */ }
}

class GameHistory
{
    Stack<ICommand> history = new Stack<ICommand>();
    
    public void Execute(ICommand cmd)
    {
        cmd.Execute();
        history.Push(cmd);
    }
    
    public void Undo()
    {
        if (history.Count > 0)
            history.Pop().Undo();
    }
}
```

### FPS / Shooter

**Key patterns needed:**
- **Object Pool** - Bullets (CRITICAL)
- **State Machine** - Player states, weapon states
- **Event System** - Kill events, achievements
- **Factory** - Weapon creation

### RTS / Strategy

**Key patterns needed:**
- **Command** - Unit orders (move, attack)
- **State Machine** - Unit AI states
- **Object Pool** - Many units
- **Event System** - Selection events, resource updates
- **Data-Driven** - Unit stats, building costs

---

## Team Size Considerations

### Solo Developer

```
Priority: Speed of development

Use:
- SerializeField for references
- Simple MonoBehaviour patterns
- Quick iteration over "perfect" architecture

Avoid:
- Over-engineering
- Patterns you don't fully understand
- Abstractions for "future flexibility"
```

### Small Team (2-5)

```
Priority: Code ownership clarity

Use:
- Clear system boundaries
- Interfaces between systems
- Documented patterns
- Code review for consistency

Communication:
- Regular architecture discussions
- Shared coding standards
```

### Large Team (5+)

```
Priority: Independent work without conflicts

Use:
- Full DI for decoupling
- Strict interfaces between systems
- Events/messages over direct calls
- Automated testing
- Architecture documentation

Structure:
- Each dev owns specific systems
- Interfaces are contracts
- Changes to interfaces need approval
```

---

## When to Add Complexity

### Signs You NEED More Architecture

| Symptom | Solution |
|---------|----------|
| Changing one thing breaks another | Add interfaces, decouple |
| Can't test without playing | Add DI, enable mocking |
| Same code in many places | Extract common behavior |
| New features require many file changes | Better separation of concerns |
| Merge conflicts everywhere | Better system boundaries |

### Signs of OVER-Architecture

| Symptom | Solution |
|---------|----------|
| More infrastructure than features | Simplify |
| Can't find where logic actually is | Reduce layers |
| Takes forever to add simple features | Reduce abstraction |
| Interfaces with one implementation | Remove interface |
| Patterns for hypothetical features | Apply YAGNI |

### The Refactoring Approach

```
1. START SIMPLE
   - Direct code, minimal abstraction
   - Get features working

2. NOTICE PAIN POINTS
   - What's hard to change?
   - What causes bugs?
   
3. REFACTOR WITH PURPOSE
   - Add pattern to solve SPECIFIC problem
   - One refactor at a time

4. REPEAT
   - Continuous improvement
   - Architecture evolves with the game
```

---

## Migration Paths

### Adding Multiplayer to Single-Player

If you followed good architecture:

| Single-Player | Multiplayer Equivalent |
|---------------|----------------------|
| Logic class | Server component + NetworkBehaviour |
| View class | Client component |
| C# events | NetworkedMessageChannel |
| Local save | Server-side validation |
| Direct calls | RPCs |

### From Prototype to Production

```
PROTOTYPE:
├── Everything in one script
├── Public fields everywhere
├── God objects
└── No structure

MIGRATION STEPS:
1. Extract responsibilities into separate classes
2. Add interfaces for major systems
3. Replace direct references with events
4. Add configuration (ScriptableObjects)
5. Add DI if testing is important
```

---

## New Project Checklist

### Day 1: Project Setup

- [ ] Create folder structure
- [ ] Set up namespaces
- [ ] Create ApplicationController (entry point)
- [ ] Decide on DI approach (VContainer, manual, none)
- [ ] Create basic event system

### Before Building Features

For each major system, answer:

1. **What's the API?** (public methods/events)
2. **Who are the clients?** (what calls it)
3. **What's the data?** (config, runtime state)
4. **What's the lifecycle?** (initialization, updates, cleanup)

### Folder Structure Template

```
Assets/
├── Scripts/
│   ├── Core/              # Application lifecycle, DI
│   ├── Gameplay/          # Game-specific logic
│   │   ├── Player/
│   │   ├── Enemies/
│   │   ├── Combat/
│   │   └── Progression/
│   ├── UI/                # All UI code
│   ├── Infrastructure/    # Reusable utilities
│   │   ├── Events/        # Event system
│   │   ├── Pooling/       # Object pools
│   │   └── Save/          # Save/load
│   └── Utils/             # Helpers
├── GameData/              # ScriptableObjects
├── Prefabs/
├── Scenes/
└── Art/
```

---

## Quick Reference: Pattern Selection

| Need | Pattern | Complexity |
|------|---------|------------|
| Objects communicate without knowing each other | PubSub / Events | Low |
| Object has distinct modes | State Machine | Medium |
| Centralized object creation | Factory | Low |
| Reuse frequently created objects | Object Pool | Medium |
| Swappable implementations | Strategy | Medium |
| Undo/redo or command queuing | Command | Medium |
| Centralized access point | Singleton (careful!) | Low |
| Testable, decoupled code | Dependency Injection | High |
| Separate data from behavior | ScriptableObject | Low |
| Separate logic from visuals | Logic/View split | Medium |

---

## The Golden Rules

1. **Solve the problem you have**, not the problem you might have
2. **Simple code that works** beats complex code that's "elegant"
3. **Refactoring later is okay** - architecture evolves
4. **Every pattern is a tradeoff** - know the costs
5. **Consistency matters more than perfection**
6. **Copy Boss Room patterns** when appropriate, adapt when not

---

## Final Checklist: Project Architecture Review

Use this when reviewing your architecture:

- [ ] Every class has a single, clear responsibility
- [ ] Major systems communicate through interfaces or events
- [ ] Configuration is in data, not hard-coded
- [ ] Frequently spawned objects use pooling
- [ ] Complex state logic uses state machines
- [ ] References are injected or serialized, not found at runtime
- [ ] Event subscriptions are properly cleaned up
- [ ] Code is testable (could you write a unit test?)

---

> **Congratulations!** You've now covered all the universal architecture patterns from Boss Room. Return to the [README](./README.md) for the complete documentation index.
