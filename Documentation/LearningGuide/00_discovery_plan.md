# 🗓️ 30-Day Comprehensive Learning Journey

> **This is your complete book-like guide to mastering multiplayer game architecture.** Each day is a full lesson with theory, concepts, code analysis, exercises, and reflection. Treat this as your textbook.

---

## 📚 How to Get the Most from This Guide

### Time Commitment
- **Daily study time:** 2-3 hours recommended
- **Best approach:** Study in the morning when your mind is fresh
- **Don't rush:** Understanding deeply is better than covering quickly

### Materials Needed
- [ ] Unity Editor with Boss Room project open
- [ ] Code editor (VS Code, Rider, or Visual Studio)
- [ ] Notebook (physical or digital) for notes and diagrams
- [ ] Drawing tool (paper or digital) for architecture sketches

### Study Method (SQ3R)
For each day:
1. **Survey** - Skim the content first
2. **Question** - Write down what you want to learn
3. **Read** - Study the material carefully
4. **Recite** - Explain what you learned (out loud if possible)
5. **Review** - Complete all exercises

### Self-Assessment
At the end of each week, you should be able to:
- Explain concepts without looking at notes
- Draw diagrams from memory
- Find related code without guidance
- Apply patterns to hypothetical scenarios

---

# WEEK 1: Foundation & Mental Models

> **Week Goal:** Build a solid mental model of the project. You should understand the "big picture" before diving into specifics.

---

## 📅 Day 1: Experiencing the Game as a Player

### Learning Objectives
By the end of today, you will:
- Understand what Boss Room is from a player's perspective
- Identify all major game flows and states
- Create a mental map of the user experience
- Know what problems the code needs to solve

### 🎓 Theory: Why Play Before Reading Code?

Before reading any code, you must understand **what problem the code solves**. This is a fundamental principle of software learning:

```
Understanding = Context + Code
```

Without context, code is just syntax. With context, code becomes a solution to a problem you understand.

**The "What Before How" Principle:**
- **What:** The game lets multiple players cooperate to defeat a boss
- **How:** Networked state synchronization, action system, etc.

When you understand the "what," the "how" makes sense.

### 📋 Detailed Instructions

#### Part 1: Play as Host (45 minutes)

1. **Open Unity Hub** → Open Boss Room project
2. **Enter Play Mode** (Ctrl+P or ▶ button)
3. **Navigate the Main Menu**
   - What options do you see?
   - What's the difference between "Host" options?
   - Write down every button and what it seems to do

4. **Host a Game**
   - Click Host (IP or Session)
   - Watch the loading carefully - what scenes load?
   - Arrive at Character Select

5. **Study Character Select**
   - How many characters are available?
   - What information is shown for each?
   - What happens when you click a character?
   - What's the "Lock In" button do?
   - Can you change your selection after locking?

6. **Play the Game**
   - Move around (note the controls)
   - Use abilities (which keys?)
   - Find enemies
   - Take damage (watch your health bar)
   - Use different abilities
   - Try to defeat the boss

#### Part 2: Play as Client (30 minutes)

1. **Build the game** (File → Build and Run) or use **Multiplayer Play Mode**
2. **Run a second instance**
3. **Join the first game**
   - What happens in Character Select?
   - Can you select the same character as the host?
   - Do you see the host's selection?

4. **Play together**
   - Does combat feel synchronized?
   - What happens if you walk away from the host?
   - What happens if you disconnect?

#### Part 3: Documentation (30 minutes)

In your notebook, create these sections:

**Section 1: Game Scenes**
```
List every scene you encountered:
1. Main Menu - purpose: _____
2. Character Select - purpose: _____
3. Boss Room - purpose: _____
4. (any others?)
```

**Section 2: Player Experience Flow**
```
Draw a flowchart:
[Launch Game] → [Main Menu] → [?] → [?] → [Playing] → [?]
                     ↓
              [Join a Game]
                     ↓
                   [?]
```

**Section 3: Questions for Code Investigation**
```
Write at least 5 questions you want answered:
1. How does character selection sync between players?
2. How does damage get from attacker to target?
3. How does the game know when boss is defeated?
4. What happens when a player disconnects?
5. How are abilities configured?
```

### 📖 Supplementary Reading

Open and skim these files (don't read deeply yet):
- `README.md` (project root) - Project overview
- `CONTRIBUTING.md` - How the project is organized

### ✅ Completion Checklist

- [ ] Played as host through full game
- [ ] Played as client connected to host
- [ ] Listed all scenes in notebook
- [ ] Drew player flow diagram
- [ ] Wrote at least 5 investigation questions
- [ ] Can explain what Boss Room is to someone unfamiliar

### 🤔 Reflection Questions

1. What was surprising about the game?
2. What felt polished? What felt rough?
3. If you were making a similar game, what would be hardest?

---

## 📅 Day 2: Project Structure & Mental Map

### Learning Objectives
By the end of today, you will:
- Know every major folder and its purpose
- Understand namespace organization patterns
- Have a reference map of the codebase
- Know where to look for specific functionality

### 🎓 Theory: Why Structure Matters

In professional codebases, **organization is not accidental**. It reflects architectural decisions:

```
Folder Structure = Architecture Made Visible
```

**Common Organization Strategies:**

| Strategy | Example |
|----------|---------|
| **By Layer** | `UI/`, `Logic/`, `Data/` |
| **By Feature** | `Combat/`, `Inventory/`, `UI/` |
| **By Technical Role** | `Server/`, `Client/`, `Shared/` |
| **Hybrid** | Combine above approaches |

Boss Room uses a **hybrid** approach, primarily organized by technical concern with gameplay grouped together.

### 📋 Detailed Instructions

#### Part 1: Top-Level Exploration (30 minutes)

Open `Assets/Scripts/` in your file explorer. Study this structure:

```
Assets/Scripts/
├── ApplicationLifecycle/    ← App startup, DI container
├── ConnectionManagement/    ← Multiplayer connection logic
├── Gameplay/                ← Core game systems
│   ├── Action/              ← Ability system
│   ├── Configuration/       ← ScriptableObjects
│   ├── GameState/           ← Scene/state management
│   ├── GameplayObjects/     ← Characters, entities
│   └── UI/                  ← Game UI
├── Infrastructure/          ← Reusable utilities
│   ├── PubSub/              ← Event system
│   └── (others)             ← Pooling, etc.
├── UnityServices/           ← Unity backend services
└── Utils/                   ← Helper classes
```

For EACH folder:
1. Open it
2. Open ONE file
3. Read the `namespace` declaration at the top
4. Write what the namespace tells you

#### Part 2: Namespace Analysis (30 minutes)

Fill this table in your notebook:

| Folder | Namespace | Purpose |
|--------|-----------|---------|
| ApplicationLifecycle | Unity.BossRoom.ApplicationLifecycle | Game startup |
| ConnectionManagement | Unity.BossRoom.ConnectionManagement | ? |
| Gameplay/Action | Unity.BossRoom.Gameplay.Actions | ? |
| Infrastructure | Unity.BossRoom.Infrastructure | ? |
| UnityServices | Unity.BossRoom.UnityServices | ? |

**Key Insight:** Namespaces follow the folder structure. This is intentional—it makes finding code easy.

#### Part 3: Create Your Reference Map (45 minutes)

In your notebook, create a **visual map** of the codebase. This will be your reference for the entire 30 days.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         BOSS ROOM CODEBASE                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ LIFECYCLE                                                            │
│ ┌─────────────────────┐                                             │
│ │ ApplicationController│ ← THE entry point                          │
│ │ - Sets up DI        │                                             │
│ │ - Initializes game  │                                             │
│ └─────────────────────┘                                             │
└─────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ CONNECTION                                                           │
│ ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │
│ │ConnectionManager│  │ ConnectionState │  │ ConnectionMethod│       │
│ │(state machine)  │  │(6 states)       │  │(IP vs Relay)    │       │
│ └─────────────────┘  └─────────────────┘  └─────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ GAMEPLAY                                                             │
│ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│ │ Actions     │  │ Characters  │  │ GameState   │  │ UI          │  │
│ │(abilities)  │  │(server/     │  │(scenes,     │  │             │  │
│ │             │  │ client)     │  │ flow)       │  │             │  │
│ └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ INFRASTRUCTURE                                                       │
│ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│ │ PubSub      │  │ ObjectPool  │  │ Utils       │                   │
│ │(events)     │  │(netcode)    │  │             │                   │
│ └─────────────┘  └─────────────┘  └─────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

#### Part 4: Key File Identification (30 minutes)

Find and note the location of these important files:

| What | File | Path |
|------|------|------|
| Entry point | ApplicationController.cs | Scripts/ApplicationLifecycle/ |
| Connection manager | ConnectionManager.cs | Scripts/ConnectionManagement/ |
| Base action class | Action.cs | Scripts/Gameplay/Action/ |
| Character (server) | ServerCharacter.cs | Scripts/Gameplay/GameplayObjects/Character/ |
| Character (client) | ClientCharacter.cs | Scripts/Gameplay/GameplayObjects/Character/ |
| Object pool | NetworkObjectPool.cs | Scripts/Infrastructure/ |

### 📖 Supplementary Reading

Open and read:
- [05_project_structure.md](./05_project_structure.md) - Project organization guide

### ✅ Completion Checklist

- [ ] Explored all top-level folders
- [ ] Identified namespace pattern
- [ ] Created visual reference map
- [ ] Located all key files
- [ ] Can navigate to any component without searching

### 🤔 Reflection Questions

1. Why might the team have chosen this structure?
2. What would change if this were a single-player game?
3. Where would you add a new feature (like an Inventory)?

---

## 📅 Day 3: The Entry Point - ApplicationController

### Learning Objectives
By the end of today, you will:
- Understand how the game bootstraps
- Know what Dependency Injection is and why it's used
- Trace the startup sequence
- Identify all services registered at startup

### 🎓 Theory: Dependency Injection Explained

**What is Dependency Injection?**

Imagine you're building a house. You need a hammer. Two approaches:

**Without DI (hard-coded dependencies):**
```csharp
class Worker {
    private Hammer hammer = new Hammer(); // Creates its own tools
    
    void DoWork() {
        hammer.Hit(nail);
    }
}
// Problem: Can't easily swap for a different hammer
// Problem: Hard to test (can't use a fake hammer)
```

**With DI (injected dependencies):**
```csharp
class Worker {
    [Inject] private IHammer hammer; // Receives tools from outside
    
    void DoWork() {
        hammer.Hit(nail);
    }
}
// Benefit: Can inject any hammer that implements IHammer
// Benefit: Can inject a mock for testing
```

**Why does Boss Room use DI?**

1. **Testability** - Can inject mock services for testing
2. **Flexibility** - Can swap implementations easily
3. **Clarity** - Dependencies are explicit
4. **Decoupling** - Classes don't create their own dependencies

**VContainer:**

Boss Room uses VContainer, a fast DI framework for Unity. The basic pattern:

```csharp
// 1. REGISTRATION (in ApplicationController.Configure)
builder.Register<MyService>(Lifetime.Singleton);

// 2. INJECTION (anywhere in code)
public class SomeClass {
    [Inject] private MyService myService; // VContainer fills this in
}
```

### 📋 Detailed Instructions

#### Part 1: Reading ApplicationController (60 minutes)

**File:** [ApplicationController.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ApplicationLifecycle/ApplicationController.cs)

Open this file and study it section by section:

**Section A: Class Declaration (lines 1-25)**

Read the imports and class declaration. Notice:
- It inherits from a VContainer class
- It's in the ApplicationLifecycle namespace
- What attributes are on the class?

**Section B: The Configure Method (main DI setup)**

This is where all services are registered. For EACH registration line, understand:

```csharp
// Pattern:
builder.Register<ConcreteClass>(Lifetime.___);
// or
builder.Register<IInterface, ConcreteClass>(Lifetime.___);
```

Create this table in your notebook:

| Registration | Lifetime | Purpose |
|--------------|----------|---------|
| LocalSession | Singleton | Stores current session info |
| ProfileManager | ? | ? |
| ConnectionManager | ? | ? |
| MessageChannel<ConnectStatus> | ? | ? |
| (continue for ALL registrations) | | |

**Lifetime meanings:**
- **Singleton** - ONE instance for entire app lifetime
- **Scoped** - One instance per scope (e.g., per scene)
- **Transient** - New instance every time requested

**Section C: The Start Method**

Trace what happens:
1. What gets resolved from the container?
2. What asynchronous operations occur?
3. What scene loads at the end?

**Section D: OnDestroy**

What cleanup happens when the app closes?

#### Part 2: Tracing the Startup Flow (30 minutes)

Draw this sequence in your notebook:

```
Unity loads scene with ApplicationController
                    │
                    ▼
            Awake() runs
            (inherited from VContainer)
                    │
                    ▼
          Configure() is called
          ┌────────────────────────────────────────┐
          │ Register all services:                 │
          │ - LocalSession                         │
          │ - ProfileManager                       │
          │ - ConnectionManager                    │
          │ - MessageChannels                      │
          │ - UnityServices facades                │
          └────────────────────────────────────────┘
                    │
                    ▼
          Container is built
          (all registrations become available)
                    │
                    ▼
            Start() runs
          ┌────────────────────────────────────────┐
          │ 1. Resolve services from container     │
          │ 2. Initialize Unity Services           │
          │ 3. Authenticate (async)                │
          │ 4. Load MainMenu scene                 │
          └────────────────────────────────────────┘
                    │
                    ▼
          MainMenu scene loads
          (game is ready for player)
```

#### Part 3: Understanding the Container Pattern (30 minutes)

The DI container is like a "service locator" that knows how to create everything:

```csharp
// WITHOUT container (hard to maintain):
var localSession = new LocalSession();
var profileManager = new ProfileManager();
var connectionManager = new ConnectionManager(localSession, profileManager);
// ... more and more constructor arguments ...

// WITH container (clean):
var connectionManager = Container.Resolve<ConnectionManager>();
// Container automatically injects all dependencies!
```

**Exercise:** Find 3 classes that have `[Inject]` attributes. List what they inject.

### 📖 Supplementary Reading

- [01_architecture_principles.md](./01_architecture_principles.md) - Section on Dependency Inversion
- [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - Section on DI

### ✅ Completion Checklist

- [ ] Can explain DI in your own words
- [ ] Listed all service registrations
- [ ] Drew the startup sequence diagram
- [ ] Found 3 [Inject] examples
- [ ] Understand the difference between Singleton/Scoped/Transient

### 🤔 Reflection Questions

1. Why register as interfaces (IPublisher<T>) instead of concrete classes?
2. What would break if you removed one registration?
3. How would you add a new service to the container?

---

## 📅 Day 4: Event Communication - The PubSub Pattern

### Learning Objectives
By the end of today, you will:
- Understand the Publish-Subscribe pattern deeply
- Know when to use events vs direct calls
- Trace message flow through the system
- Implement a mental model of decoupled communication

### 🎓 Theory: The Problem PubSub Solves

**Scenario without PubSub:**

```csharp
class HealthSystem {
    void TakeDamage(int amount) {
        currentHealth -= amount;
        
        // HealthSystem must know about ALL these systems!
        healthBarUI.UpdateDisplay(currentHealth);
        soundSystem.PlayHurtSound();
        achievementSystem.TrackDamageTaken(amount);
        analyticsSystem.LogEvent("damage_taken", amount);
        combatLog.AddEntry($"Took {amount} damage");
        cameraShake.Shake(0.5f);
        particleSystem.SpawnBloodSplatter();
        // ... and more ...
    }
}
```

**Problems:**
1. HealthSystem knows too much about other systems
2. Adding new reactions requires modifying HealthSystem
3. Hard to test HealthSystem in isolation
4. Circular dependencies possible

**With PubSub:**

```csharp
class HealthSystem {
    [Inject] IPublisher<HealthChangedMessage> healthPublisher;
    
    void TakeDamage(int amount) {
        currentHealth -= amount;
        
        // Just publish - don't care who listens!
        healthPublisher.Publish(new HealthChangedMessage { 
            CurrentHealth = currentHealth,
            DamageAmount = amount 
        });
    }
}

// Anyone can subscribe independently:
class HealthBarUI {
    [Inject] ISubscriber<HealthChangedMessage> healthSubscriber;
    
    void Start() {
        healthSubscriber.Subscribe(OnHealthChanged);
    }
    
    void OnHealthChanged(HealthChangedMessage msg) {
        UpdateDisplay(msg.CurrentHealth);
    }
}
```

### 📋 Detailed Instructions

#### Part 1: Interface Study (30 minutes)

**File:** [IMessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/IMessageChannel.cs)

Study each interface:

```csharp
// PUBLISHER - can send messages
public interface IPublisher<T> {
    void Publish(T message);
}

// SUBSCRIBER - can receive messages
public interface ISubscriber<T> {
    IDisposable Subscribe(Action<T> handler);
    void Unsubscribe(Action<T> handler);
}

// CHANNEL - can do both (usually used internally)
public interface IMessageChannel<T> : IPublisher<T>, ISubscriber<T>, IDisposable {
    bool IsDisposed { get; }
}
```

**Key insight:** Why return `IDisposable` from Subscribe()?

```csharp
// Pattern for proper cleanup:
class SomeComponent : MonoBehaviour {
    IDisposable subscription;
    
    void Start() {
        subscription = channel.Subscribe(OnMessage);
    }
    
    void OnDestroy() {
        subscription?.Dispose();  // Automatically unsubscribe!
    }
}
```

#### Part 2: MessageChannel Implementation (45 minutes)

**File:** [MessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs)

Study the implementation carefully:

**The "Pending Handlers" Problem:**

```csharp
// PROBLEM: Modifying collection while iterating crashes!
foreach (var handler in handlers) {
    handler.Invoke(message);  // What if handler unsubscribes here?
}

// SOLUTION: Defer modifications!
Dictionary<Action<T>, bool> m_PendingHandlers; // true=add, false=remove

void Publish(T message) {
    // 1. First, process pending changes
    foreach (var pending in m_PendingHandlers) {
        if (pending.Value)
            m_MessageHandlers.Add(pending.Key);
        else
            m_MessageHandlers.Remove(pending.Key);
    }
    m_PendingHandlers.Clear();
    
    // 2. Then invoke handlers (now safe!)
    foreach (var handler in m_MessageHandlers) {
        handler?.Invoke(message);
    }
}
```

In your notebook, trace this scenario:
1. Handler A is subscribed
2. Publish() is called
3. Handler A receives message
4. Handler A calls Unsubscribe(A) inside its callback
5. What happens next?

#### Part 3: NetworkedMessageChannel (30 minutes)

**File:** [NetworkedMessageChannel.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/PubSub/NetworkedMessageChannel.cs)

This extends MessageChannel to work across the network:

```csharp
public override void Publish(T message) {
    if (m_NetworkManager.IsServer) {
        // Server: send to all clients, then publish locally
        SendMessageThroughNetwork(message);
        base.Publish(message);
    } else {
        Debug.LogError("Only server can publish!");
    }
}
```

**Key constraint:** `where T : unmanaged, INetworkSerializeByMemcpy`

This means the message must be:
- `unmanaged` - No reference types (string, class)
- `INetworkSerializeByMemcpy` - Can be copied byte-by-byte

**Why?** Network messages must be serializable to binary!

#### Part 4: Find Real Usage (30 minutes)

Search the codebase (Ctrl+Shift+F) for each:

1. Search: `IPublisher<`
   - Find 3 classes that publish messages
   - What messages do they publish?

2. Search: `ISubscriber<`
   - Find 3 classes that subscribe
   - What do they do when receiving?

3. Search: `Subscribe(`
   - Find 3 subscription calls
   - Are they properly disposed?

Fill this table:

| Publisher Class | Message Type | Subscriber Class | Action |
|-----------------|--------------|------------------|--------|
| HostingState | ConnectionEventMessage | ? | ? |
| ? | ConnectStatus | MainMenuUI | Show error |
| ? | ? | ? | ? |

### 📖 Supplementary Reading

- [04_design_patterns.md](./04_design_patterns.md) - Pattern 2: Observer
- [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - PubSub section

### ✅ Completion Checklist

- [ ] Can explain PubSub without notes
- [ ] Understand why Subscribe returns IDisposable
- [ ] Know why pending handlers exist
- [ ] Found real publisher/subscriber pairs
- [ ] Understand NetworkedMessageChannel constraint

### 🤔 Reflection Questions

1. When would you NOT use PubSub?
2. Could you have too many message channels?
3. How would you debug a "message not received" bug?

---

## 📅 Day 5: Connection State Machine - Part 1

### Learning Objectives
By the end of today, you will:
- Understand the State Machine pattern in depth
- Know all 6 connection states
- Trace state transitions
- Understand why state machines make complex logic manageable

### 🎓 Theory: The State Machine Pattern

**What is a State Machine?**

A state machine is an object that:
1. Has a **finite set of states** it can be in
2. Has **transitions** between states triggered by events
3. **Behaves differently** depending on current state

**Why use state machines?**

Without state machine:
```csharp
void OnDisconnect() {
    if (wasConnected && !wasHost && shouldReconnect && 
        reconnectAttempts < maxAttempts && !userRequestedDisconnect) {
        // Try to reconnect
    } else if (wasHost && wasInGame) {
        // Shut down server
    } else if (wasConnecting && !wasHost) {
        // Show connection failed
    }
    // ... more and more conditions ...
}
```

With state machine:
```csharp
// In ClientReconnectingState:
public override void OnDisconnect() {
    if (attempts < maxAttempts) {
        TryReconnect();
    } else {
        ChangeState(offlineState);
    }
}
// Clean! This state only handles reconnection logic.
```

**The State Pattern Structure:**

```
┌─────────────────────────────────────────────────────────┐
│  STATE MACHINE (ConnectionManager)                       │
├─────────────────────────────────────────────────────────┤
│  - Holds current state                                   │
│  - Routes events to current state                        │
│  - Handles state transitions                             │
└─────────────────────────────────────────────────────────┘
                        │
           ┌────────────┼────────────┐
           │            │            │
           ▼            ▼            ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ Offline  │  │Connecting│  │ Hosting  │
    │ State    │  │ State    │  │ State    │
    └──────────┘  └──────────┘  └──────────┘
    Each state handles events differently
```

### 📋 Detailed Instructions

#### Part 1: ConnectionManager Analysis (45 minutes)

**File:** [ConnectionManager.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionManager.cs)

**Section A: State Instances (lines 74-79)**

```csharp
internal readonly OfflineState m_Offline = new OfflineState();
internal readonly ClientConnectingState m_ClientConnecting = new ClientConnectingState();
internal readonly ClientConnectedState m_ClientConnected = new ClientConnectedState();
internal readonly ClientReconnectingState m_ClientReconnecting = new ClientReconnectingState();
internal readonly StartingHostState m_StartingHost = new StartingHostState();
internal readonly HostingState m_Hosting = new HostingState();
```

**Why pre-create states?**
1. No allocation during gameplay (no GC spikes)
2. States can be configured before use
3. Dependencies injected once at startup

**Section B: ChangeState Method (lines 112-122)**

```csharp
internal void ChangeState(ConnectionState nextState)
{
    Debug.Log($"Changed state from {m_CurrentState} to {nextState}");
    
    if (m_CurrentState != null)
        m_CurrentState.Exit();  // Let old state clean up
    
    m_CurrentState = nextState;
    m_CurrentState.Enter();     // Let new state initialize
}
```

This is the **only** way states change. Every transition goes through here.

**Section C: Event Routing (lines 124-155)**

NetworkManager fires callbacks. ConnectionManager routes them to current state:

```csharp
void OnConnectionEvent(NetworkManager nm, ConnectionEventData data)
{
    switch (data.EventType) {
        case ConnectionEvent.ClientConnected:
            m_CurrentState.OnClientConnected(data.ClientId);
            break;
        case ConnectionEvent.ClientDisconnected:
            m_CurrentState.OnClientDisconnect(data.ClientId);
            break;
    }
}
```

Why routing? Because **the same event means different things in different states!**

| Event | In OfflineState | In ConnectingState | In HostingState |
|-------|-----------------|--------------------| --------------- |
| OnClientConnected | (ignored) | "I connected! → Connected" | "A player joined" |
| OnClientDisconnect | (ignored) | "Failed → Offline" | "A player left" |

#### Part 2: Base State Class (30 minutes)

**File:** [ConnectionState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/ConnectionState.cs)

```csharp
abstract class ConnectionState
{
    [Inject] protected ConnectionManager m_ConnectionManager;
    [Inject] protected IPublisher<ConnectStatus> m_ConnectStatusPublisher;
    
    // Lifecycle
    public abstract void Enter();
    public abstract void Exit();
    
    // Events (default: do nothing)
    public virtual void OnClientConnected(ulong clientId) { }
    public virtual void OnClientDisconnect(ulong clientId) { }
    public virtual void OnServerStarted() { }
    public virtual void StartClientIP(string playerName, string ip, int port) { }
    public virtual void StartClientSession(string playerName) { }
    public virtual void StartHostIP(string playerName, string ip, int port) { }
    public virtual void StartHostSession(string playerName) { }
    public virtual void OnUserRequestedShutdown() { }
    public virtual void ApprovalCheck(...) { }
    public virtual void OnTransportFailure() { }
    public virtual void OnServerStopped() { }
}
```

**Key insight:** Virtual methods with empty bodies mean "ignore this event by default." States only override what they care about.

#### Part 3: Draw the State Diagram (30 minutes)

In your notebook, draw a complete state diagram:

```
                    ┌───────────────────────────────┐
                    │          OFFLINE              │
                    │  - Initial state              │
                    │  - Can start host or client   │
                    └───────────────┬───────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       │                       ▼
┌───────────────────────┐           │           ┌───────────────────────┐
│    STARTING HOST      │           │           │   CLIENT CONNECTING   │
│ - Setting up relay    │           │           │ - Connecting to host  │
│ - Starting NetworkMgr │           │           │ - Waiting for approve │
└───────────┬───────────┘           │           └───────────┬───────────┘
            │                       │                       │
    ┌───────┴───────┐               │           ┌───────────┴───────────┐
  Success         Fail              │         Success                 Fail
    │               │               │           │                       │
    ▼               ▼               │           ▼                       ▼
┌─────────────┐     └───────────────┼───────────┘           ┌───────────────┐
│   HOSTING   │                     │                       │   (back to    │
│ - Accepting │                     │                       │    Offline)   │
│   clients   │                     │                       └───────────────┘
└─────────────┘                     │
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │   CLIENT CONNECTED    │──── disconnect ────►
                        │ - Normal gameplay     │                     │
                        └───────────────────────┘                     │
                                    │                                  │
                               disconnect                              │
                               (unexpected)                            │
                                    │                                  │
                                    ▼                                  │
                        ┌───────────────────────┐                     │
                        │  CLIENT RECONNECTING  │◄────────────────────┘
                        │ - Attempting reconnect│
                        └───────────────────────┘
                                    │
                            ┌───────┴───────┐
                          Success         Max attempts
                            │                 │
                            ▼                 ▼
                    (ClientConnected)    (Offline)
```

### 📖 Supplementary Reading

- [10_connection_state_machine.md](./10_connection_state_machine.md) - Complete deep dive
- [04_design_patterns.md](./04_design_patterns.md) - Pattern 1: State Machine

### ✅ Completion Checklist

- [ ] Can explain state machine pattern
- [ ] Know all 6 connection states
- [ ] Drew complete state diagram
- [ ] Understand why states are pre-created
- [ ] Know how event routing works

### 🤔 Reflection Questions

1. What would happen if you forgot to call Exit() before Enter()?
2. How would you add a "Matchmaking" state?
3. Why use virtual methods instead of abstract?

---

## 📅 Day 6: Connection States - Part 2 (Each State Deep Dive)

### Learning Objectives
By the end of today, you will:
- Understand what each state does in Enter() and Exit()
- Know the approval flow for incoming connections
- Trace a complete client join sequence
- Understand reconnection logic

### 📋 Detailed Instructions

#### Part 1: Fill the State Table (60 minutes)

For EACH state file, open it and complete this table:

| State | Enter() Does | Exit() Does | Key Methods | Transitions To |
|-------|--------------|-------------|-------------|----------------|
| OfflineState | Shutdown network, load MainMenu | (nothing) | StartClientIP, StartHostSession | ClientConnecting, StartingHost |
| StartingHostState | | | | |
| HostingState | | | | |
| ClientConnectingState | | | | |
| ClientConnectedState | | | | |
| ClientReconnectingState | | | | |

**Files to open:**
- [OfflineState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/OfflineState.cs)
- [StartingHostState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/StartingHostState.cs)
- [HostingState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/HostingState.cs)
- [ClientConnectingState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/ClientConnectingState.cs)
- [ClientConnectedState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/ClientConnectedState.cs)
- [ClientReconnectingState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/ClientReconnectingState.cs)

#### Part 2: Connection Approval Deep Dive (45 minutes)

**File:** [HostingState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/HostingState.cs)

Focus on the `ApprovalCheck` method. Trace the validation steps:

```csharp
public override void ApprovalCheck(Request request, Response response) {
    // 1. DOS Protection - reject huge payloads
    if (payload.Length > 1024) {
        response.Approved = false;
        return;
    }
    
    // 2. Parse the connection data
    var connectionPayload = JsonUtility.FromJson<ConnectionPayload>(payloadString);
    
    // 3. Run game-specific checks
    var status = GetConnectStatus(connectionPayload);
    
    // 4. Approve or reject
    if (status == ConnectStatus.Success) {
        response.Approved = true;
        response.CreatePlayerObject = true;
    } else {
        response.Approved = false;
        response.Reason = JsonUtility.ToJson(status);
    }
}
```

**GetConnectStatus checks:**
1. Is server full? → `ConnectStatus.ServerFull`
2. Build type mismatch? → `ConnectStatus.IncompatibleBuildType`
3. Already connected elsewhere? → `ConnectStatus.LoggedInAgain`
4. Otherwise → `ConnectStatus.Success`

Draw a flowchart of the approval process.

#### Part 3: Reconnection Deep Dive (30 minutes)

**File:** [ClientReconnectingState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/ClientReconnectingState.cs)

Study the reconnection coroutine:

```csharp
IEnumerator ReconnectCoroutine() {
    // Wait before retrying (exponential backoff could be added here)
    if (m_NbAttempts > 0)
        yield return new WaitForSeconds(k_TimeBetweenAttempts);
    
    // Shutdown and wait for completion
    NetworkManager.Shutdown();
    yield return new WaitWhile(() => NetworkManager.ShutdownInProgress);
    
    // Wait before first attempt (give services time to update)
    if (m_NbAttempts == 0)
        yield return new WaitForSeconds(k_TimeBeforeFirstAttempt);
    
    m_NbAttempts++;
    
    // Try to reconnect through the connection method
    var task = m_ConnectionMethod.SetupClientReconnectionAsync();
    yield return new WaitUntil(() => task.IsCompleted);
    
    if (task.Result.success)
        ConnectClientAsync();  // Will trigger OnClientConnected or OnClientDisconnect
    else
        OnClientDisconnect(0);  // Trigger retry or give up
}
```

### ✅ Completion Checklist

- [ ] Completed state table for all 6 states
- [ ] Drew approval flowchart
- [ ] Understand reconnection timing
- [ ] Can trace full join sequence

---

## 📅 Day 7: Week 1 Comprehensive Review

### Goal
Consolidate all knowledge from Week 1 without referring to notes.

### Self-Assessment Quiz

**Answer these without looking at your notes (then check):**

1. What is the entry point class called?
2. Name 5 services registered in the DI container.
3. What does PubSub stand for and what problem does it solve?
4. Why do states have Enter() and Exit() methods?
5. What are the 6 connection states?
6. What happens during connection approval?
7. Why use interfaces with DI instead of concrete classes?
8. What is the "pending handlers" pattern in MessageChannel?

### Practical Exercises

**Exercise 1: Draw from Memory**
Without looking, draw:
- The startup sequence
- The state diagram for connections
- The PubSub flow

**Exercise 2: Trace These Flows**
Write step-by-step what happens for:
1. User launches game → Main Menu appears
2. User clicks "Host" → Game is ready for players
3. User clicks "Join" → Enters the game

**Exercise 3: Find in Code**
Without guidance, find:
1. Where scene loading happens
2. Where network messages are sent
3. Where player objects are created

### Week 1 Mastery Criteria

You've mastered Week 1 if you can:
- [ ] Explain the project structure to a colleague
- [ ] Navigate to any system without searching
- [ ] Explain DI, PubSub, and State Machine patterns
- [ ] Draw the connection state diagram from memory
- [ ] Trace startup from launch to MainMenu

---

# WEEK 2: Action System & Combat

> **Week Goal:** Master the action/ability system - the heart of gameplay.

*(Continue with Days 8-14 covering the Action system in depth...)*

---

## 📅 Day 8: The Character Split (Server vs Client)

### Learning Objectives
- Understand why characters are split into Server and Client components
- Know what each component is responsible for
- Trace communication between them

### 🎓 Theory: Why Split Characters?

In multiplayer games, we need to separate:
- **Authority (Server):** What is TRUE about the game state
- **Visualization (Client):** What the player SEES

```
┌──────────────────────────────────────────────────────────────────┐
│                        THE SEPARATION                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SERVER SIDE (Truth)                CLIENT SIDE (Display)        │
│  ─────────────────────              ─────────────────────        │
│  - Health value                     - Health bar UI              │
│  - Position (authoritative)         - Position (interpolated)    │
│  - Damage calculations              - Damage numbers              │
│  - Action validation                - Action animations           │
│  - Hit detection                    - Hit effects                 │
│  - Death determination              - Death animation             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 📋 Detailed Instructions

#### Part 1: ServerCharacter Analysis (60 minutes)

**File:** [ServerCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs)

Study these key areas:

1. **Network Variables** (find all NetworkVariable declarations)
   - What data is synchronized?
   - Why NetworkVariable and not regular variables?

2. **Action Player** (find ServerActionPlayer)
   - How are actions queued?
   - Who controls action execution?

3. **Damage Interface** (find IDamageable implementation)
   - How does ReceiveHP work?
   - What validation happens?

#### Part 2: ClientCharacter Analysis (45 minutes)

**File:** [ClientCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ClientCharacter.cs)

Study:

1. **Visual Updates**
   - How does it know when to update visuals?
   - How does it subscribe to NetworkVariable changes?

2. **Animation**
   - Who triggers animations?
   - How are animations synchronized?

3. **Client Anticipation**
   - What is anticipation?
   - Why is it important for responsiveness?

### ✅ Completion Checklist

- [ ] Listed all NetworkVariables in ServerCharacter
- [ ] Understand ActionPlayer relationship
- [ ] Know how damage flows
- [ ] Understand visual update pattern

---

## 📅 Day 9-14: Action System Deep Dive

*(Days 9-14 cover: Action.cs base class, ActionConfig data-driven design, MeleeAction walkthrough, Projectile actions, ActionFactory and pooling, Custom action creation)*

---

# WEEK 3: UI, Game States, and Scenes

*(Days 15-21 cover: GameStateBehaviour, Character Selection sync, Scene management, UI Mediator pattern, Networked UI updates)*

---

# WEEK 4: Services, Polish, and Mastery

*(Days 22-28 cover: Unity Services integration, Authentication, Object Pooling, Full flow traces, Pattern identification exercises)*

---

## 📅 Day 29: Create Your Architecture Reference

### Goal
Synthesize everything into your personal reference.

### Create Your Document

Create `Documentation/MyNotes/my_architecture_reference.md` with:

1. **Project Map** - Your version of the codebase structure
2. **Pattern Glossary** - Each pattern with one-sentence explanation
3. **Quick Reference** - Where to find key files
4. **Decision Guide** - Which pattern to use when
5. **Lessons Learned** - What surprised you, what you'd do differently

---

## 📅 Day 30: Apply Your Knowledge

### Goal
Prove mastery by doing.

### Choose One Capstone Project

**Option A: Add a New Ability**
- Create a new ActionConfig
- Implement the Action class (server and client)
- Test in multiplayer

**Option B: Add a New Game Feature**
- Implement a simple scoreboard
- Add a countdown timer
- Create a new UI panel

**Option C: Architecture for Your Game**
- Design the folder structure
- List the patterns you'd use
- Draw the main system diagrams
- Write the DI container setup

---

## 🎓 Graduation

You've completed the comprehensive 30-day journey. You now possess:

- ✅ Deep understanding of professional game architecture
- ✅ Mastery of essential design patterns
- ✅ Knowledge of multiplayer networking best practices
- ✅ Skills to read and understand large codebases
- ✅ Foundation to build your own games using proven patterns

**Your next steps:**
1. Build something! Apply what you learned
2. Return to specific days when you need a refresher
3. Keep your personal reference updated
4. Share knowledge with others - teaching reinforces learning

---

> 📚 **Cross-Reference Guides:**
> - [09_action_system_deepdive.md](./09_action_system_deepdive.md) - Complete action system analysis
> - [10_connection_state_machine.md](./10_connection_state_machine.md) - Connection management deep dive
> - [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - Infrastructure patterns
> - [12_system_flow_diagrams.md](./12_system_flow_diagrams.md) - Visual diagrams
> - [13_code_reading_walkthroughs.md](./13_code_reading_walkthroughs.md) - Guided code exploration
