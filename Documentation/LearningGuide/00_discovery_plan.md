# 🗓️ 30-Day Comprehensive Learning Journey (Enhanced)

> **This is your complete guided path through Boss Room architecture.** Each day includes prerequisite reading, hands-on exercises, AND answers to questions. By Day 30, you'll have read all guides and mastered the codebase.

---

## 📚 How This Guide Works

### For Each Day You'll Get:
1. **📖 Prerequisites** - Which guide(s) to read BEFORE starting
2. **🎯 Learning Objectives** - What you'll master
3. **💻 Hands-On Work** - Specific files to study with line numbers
4. **✅ Checkpoints** - Verify your understanding
5. **💡 Answers** - Solutions to exercises and reflection questions

### Time Commitment
- **Daily study time:** 2-3 hours recommended
- **Required guides:** Read the prerequisite guide FULLY before each day

### Materials
- [ ] Unity Editor with Boss Room project open
- [ ] Code editor (Rider, VS Code, or Visual Studio)
- [ ] This guide open alongside the code

---

# WEEK 1: Foundation & Mental Models
> **Goal:** Understand the big picture before diving into specifics.

---

## 📅 Day 1: Experiencing the Game as a Player

### 📖 Prerequisites
**Read first:** No prerequisite guides - just play!

### 🎯 Learning Objectives
- Understand what Boss Room is from a player's perspective
- Identify all major game flows and states
- Form questions that the code will answer

### 💻 Hands-On Work

#### Part 1: Play as Host (45 min)
1. Open Unity Hub → Boss Room project → Play Mode (Ctrl+P)
2. Click **Host (IP)** or **Host (Session)**
3. Observe the **Character Select** screen
4. Pick a character, click **Lock In**
5. Play through to defeat the boss (or die trying!)

**Record in your notes:**
- How many characters are available? (Answer: 8 - Tank, Healer, Mage, Rogue, and 4 enemy types)
- What scenes did you see? (Answer: MainMenu → CharSelect → BossRoom → PostGame)

#### Part 2: Play as Client (30 min)
1. Build the game OR use Multiplayer Play Mode
2. Run a second instance and **Join** the host
3. Notice: Can you pick the same character as host? (Answer: No, seats are exclusive)

#### Part 3: Write Investigation Questions
Write 5 questions you want answered. Examples:
1. How does character selection sync between players?
2. How does damage get from attacker to target?
3. What happens when a player disconnects and reconnects?

### ✅ Checkpoints
- [ ] Played full game as host
- [ ] Played as client connecting to host
- [ ] Listed all 4 scenes (MainMenu, CharSelect, BossRoom, PostGame)
- [ ] Wrote 5 investigation questions

### 💡 Answers to Reflection Questions

**Q: What did you find surprising?**
> Many players are surprised that character selection prevents duplicates - this requires network synchronization of seat states.

**Q: What would be hardest to implement yourself?**
> The action system and reconnection handling are typically the most complex systems.

---

## 📅 Day 2: Project Structure & Code Navigation

### 📖 Prerequisites
**Read FULLY:**
- [05_project_structure.md](./05_project_structure.md) - Project organization
- [A1_code_navigation_guide.md](./A1_code_navigation_guide.md) - File lookup reference

### 🎯 Learning Objectives
- Know every major folder and its purpose
- Navigate to any feature without searching
- Understand namespace organization

### 💻 Hands-On Work

#### Part 1: Folder Exploration (30 min)
Open `Assets/Scripts/` and explore each folder:

| Folder | Purpose | Key File to Open |
|--------|---------|------------------|
| `ApplicationLifecycle/` | Game startup, DI | `ApplicationController.cs` |
| `ConnectionManagement/` | Network connection | `ConnectionManager.cs` |
| `Gameplay/Action/` | Combat abilities | `Action.cs` |
| `Gameplay/GameplayObjects/Character/` | Player/NPC logic | `ServerCharacter.cs` |
| `Gameplay/GameState/` | Scene management | `ServerBossRoomState.cs` |
| `Infrastructure/PubSub/` | Event system | `MessageChannel.cs` |

#### Part 2: Namespace Pattern (20 min)
Open 5 different files. Notice the pattern:
```csharp
namespace Unity.BossRoom.Gameplay.GameplayObjects.Character
// Matches: Assets/Scripts/Gameplay/GameplayObjects/Character/
```

**Key insight:** Namespaces mirror folder structure - this is intentional!

#### Part 3: Key File Memorization (30 min)
Using A1_code_navigation_guide.md, find these files WITHOUT searching:

| Feature | Find This File |
|---------|---------------|
| Entry point | `ApplicationController.cs` |
| Player health | `NetworkHealthState.cs` |
| Combat action base | `Action.cs` |
| Connection state | `HostingState.cs` |

### ✅ Checkpoints
- [ ] Read A1_code_navigation_guide.md completely
- [ ] Can navigate to ConnectionManager in 3 clicks
- [ ] Know the namespace pattern
- [ ] Located all key files from table above

### 💡 Answers to Reflection Questions

**Q: Why is the project organized this way?**
> It uses a **hybrid organization** - technical role at top level (Infrastructure, Gameplay) with features underneath. This scales well for large teams where different developers own different systems.

**Q: Where would you add Inventory?**
> `Assets/Scripts/Gameplay/Inventory/` with namespace `Unity.BossRoom.Gameplay.Inventory`

---

## 📅 Day 3: The Entry Point - ApplicationController

### 📖 Prerequisites
**Read FULLY:**
- [01_architecture_principles.md](./01_architecture_principles.md) - Section on Dependency Inversion
- [19_game_flow_deepdive.md](./19_game_flow_deepdive.md) - Application Startup Sequence

### 🎯 Learning Objectives
- Understand Dependency Injection (DI) and why it's used
- Trace the complete startup sequence
- Know all services registered at startup

### 💻 Hands-On Work

#### Part 1: Study ApplicationController (60 min)
**File:** `Assets/Scripts/ApplicationLifecycle/ApplicationController.cs`

**Read these sections in order:**

1. **Lines 1-30:** Class declaration and Serializefields
   - What base class does it inherit from? (Answer: `LifetimeScope` from VContainer)

2. **Configure() method:** DI registrations
   - Count how many `builder.Register<>` calls there are
   - Note which have `Lifetime.Singleton`

3. **Start() method:** Bootstrap sequence
   - What happens at the end? (Answer: `SceneManager.LoadScene("MainMenu")`)

#### Part 2: Create Registration Table (30 min)

Fill this table by reading Configure():

| Registration | Lifetime | Purpose |
|--------------|----------|---------|
| `UpdateRunner` | Component | Non-MonoBehaviour update loop |
| `ConnectionManager` | Component | Network state machine |
| `LocalSession` | Singleton | Current session data |
| `ProfileManager` | Singleton | Player profile |
| `MessageChannel<ConnectStatus>` | Instance | Connection status events |
| *(continue for all registrations)* | | |

#### Part 3: Draw Startup Flow (30 min)

Draw in your notebook:
```
Unity loads Startup scene
        ↓
ApplicationController.Awake() [VContainer]
        ↓
Configure() - Register all services
        ↓
Container.Build()
        ↓
Start() - Resolve services, LoadScene("MainMenu")
        ↓
MainMenu scene loads
```

### ✅ Checkpoints
- [ ] Read 19_game_flow_deepdive.md completely
- [ ] Listed all service registrations
- [ ] Drew startup sequence diagram
- [ ] Understand Singleton vs Scoped vs Transient

### 💡 Answers to Reflection Questions

**Q: Why use interfaces (IPublisher<T>) instead of concrete classes?**
> Interfaces allow swapping implementations. For testing, you can inject a mock publisher. For different platforms, you could inject platform-specific implementations.

**Q: What would break if you removed one registration?**
> Any class that has `[Inject] SomeService` would fail at runtime with a VContainer exception saying it couldn't resolve the dependency.

**Q: How would you add a new service?**
> 1. Create your service class
> 2. Add `builder.Register<YourService>(Lifetime.Singleton);` in Configure()
> 3. Use `[Inject] YourService` anywhere you need it

---

## 📅 Day 4: Event Communication - The PubSub Pattern

### 📖 Prerequisites
**Read FULLY:**
- [04_design_patterns.md](./04_design_patterns.md) - Pattern 2: Observer
- [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - PubSub section

### 🎯 Learning Objectives
- Understand Publish-Subscribe pattern deeply
- Trace message flow through the system
- Know when to use events vs direct calls

### 💻 Hands-On Work

#### Part 1: Interface Study (30 min)
**File:** `Assets/Scripts/Infrastructure/PubSub/IMessageChannel.cs`

Understand the three interfaces:
```csharp
IPublisher<T>    // Can send messages: Publish(message)
ISubscriber<T>   // Can receive messages: Subscribe(handler)
IMessageChannel<T> // Both publish and subscribe
```

**Key insight:** Why return `IDisposable` from Subscribe()?
> To enable automatic cleanup: `subscription.Dispose()` unsubscribes!

#### Part 2: Find Real Usage (45 min)
Search the codebase (Ctrl+Shift+F):

**Search 1:** `IPublisher<`
- Find: `HostingState.cs` publishes `ConnectionEventMessage`
- Find: `ConnectionManager.cs` publishes `ConnectStatus`

**Search 2:** `Subscribe(`
- Find: `MainMenuUI.cs` subscribes to connection events
- Find: How they dispose subscriptions

**Fill this table:**

| Publisher | Message Type | Subscriber | What Happens |
|-----------|--------------|------------|--------------|
| HostingState | LifeStateChangedEventMessage | ServerBossRoomState | Checks for game over |
| ConnectionManager | ConnectStatus | UI components | Show status messages |
| (find 3 more) | | | |

#### Part 3: Pending Handlers Pattern (30 min)
**File:** `Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs`

Study the `m_PendingHandlers` dictionary. This solves:
> **Problem:** If a handler unsubscribes during Publish(), modifying the collection crashes!
> **Solution:** Queue add/remove operations, apply them before next Publish()

### ✅ Checkpoints
- [ ] Read sections on PubSub completely
- [ ] Found 5 publisher/subscriber pairs
- [ ] Understand why pending handlers exist
- [ ] Know the IDisposable pattern for cleanup

### 💡 Answers to Reflection Questions

**Q: When would you NOT use PubSub?**
> When you have a 1:1 relationship and direct reference is simpler. Don't over-engineer - if only one class ever needs to react, direct call is fine.

**Q: How would you debug "message not received"?**
> 1. Check subscriber called Subscribe() before publisher called Publish()
> 2. Check subscription wasn't disposed early
> 3. Add debug logs in MessageChannel.Publish()
> 4. Verify both sides use the same MessageChannel instance (same DI registration)

---

## 📅 Day 5: Connection State Machine

### 📖 Prerequisites
**Read FULLY:**
- [10_connection_state_machine.md](./10_connection_state_machine.md) - Complete deep dive
- [04_design_patterns.md](./04_design_patterns.md) - Pattern 1: State Machine

### 🎯 Learning Objectives
- Understand the State Machine pattern
- Know all 6 connection states
- Trace state transitions for host and client

### 💻 Hands-On Work

#### Part 1: ConnectionManager Analysis (45 min)
**File:** `Assets/Scripts/ConnectionManagement/ConnectionManager.cs`

**Find and study:**
1. **Lines 74-79:** Pre-created state instances
2. **ChangeState() method:** How transitions work
3. **OnConnectionEvent():** How events route to current state

#### Part 2: State Table (60 min)
Open each state file and fill this table:

| State | Enter() Does | When Reached | Transitions To |
|-------|--------------|--------------|----------------|
| OfflineState | Load MainMenu | App start, disconnect | ClientConnecting, StartingHost |
| StartingHostState | StartHost async | User clicks Host | Hosting or Offline |
| HostingState | Load CharSelect | Host started | Offline (on shutdown) |
| ClientConnectingState | StartClient async | User clicks Join | ClientConnected or Offline |
| ClientConnectedState | (nothing) | Connected | ClientReconnecting or Offline |
| ClientReconnectingState | Start reconnect loop | Unexpected disconnect | ClientConnected or Offline |

**Files:**
- `ConnectionState/OfflineState.cs`
- `ConnectionState/StartingHostState.cs`
- `ConnectionState/HostingState.cs`
- `ConnectionState/ClientConnectingState.cs`
- `ConnectionState/ClientConnectedState.cs`
- `ConnectionState/ClientReconnectingState.cs`

#### Part 3: Draw State Diagram (30 min)
Draw from memory, then verify against 10_connection_state_machine.md

### ✅ Checkpoints
- [ ] Read 10_connection_state_machine.md completely
- [ ] Filled state table for all 6 states
- [ ] Can draw state diagram from memory
- [ ] Understand why states are pre-created

### 💡 Answers to Reflection Questions

**Q: What would happen if you forgot Exit() before Enter()?**
> Resources might leak. For example, a coroutine started in Enter() wouldn't be stopped, leading to duplicate coroutines running.

**Q: How would you add a "Matchmaking" state?**
> 1. Create `MatchmakingState.cs` extending `ConnectionState`
> 2. Add `internal readonly MatchmakingState m_Matchmaking` in ConnectionManager
> 3. Add transitions from OfflineState to MatchmakingState
> 4. In MatchmakingState.Enter(), start matchmaking, then transition to ClientConnecting when match found

---

## 📅 Day 6: Connection Approval & Reconnection

### 📖 Prerequisites
**Read FULLY:**
- [20_session_reconnection.md](./20_session_reconnection.md) - Session management

### 🎯 Learning Objectives
- Understand PlayerId vs ClientId
- Know the approval flow for connections
- Trace reconnection logic

### 💻 Hands-On Work

#### Part 1: Approval Flow (60 min)
**File:** `Assets/Scripts/ConnectionManagement/ConnectionState/HostingState.cs`

Find `ApprovalCheck()` method. Trace the checks:
1. Payload size validation (DOS protection)
2. Parse `ConnectionPayload`
3. IsDuplicateConnection check
4. Server full check
5. Build type compatibility

**Draw the flowchart in your notes.**

#### Part 2: SessionManager Deep Dive (45 min)
**File:** `Packages/com.unity.multiplayer.samples.coop/Utilities/Net/SessionManager.cs`

Study the two dictionaries:
```csharp
Dictionary<string, T> m_ClientData;           // PlayerId → Session Data
Dictionary<ulong, string> m_ClientIDToPlayerId; // ClientId → PlayerId
```

**Answer:**
- Why two dictionaries? (PlayerId is stable, ClientId changes on reconnect)
- What does `IsDuplicateConnection` check? (Same PlayerId already connected)

#### Part 3: Reconnection Trace (30 min)
Using 20_session_reconnection.md, trace what happens when:
1. Player connects first time
2. Player disconnects mid-game
3. Player reconnects

### ✅ Checkpoints
- [ ] Read 20_session_reconnection.md completely
- [ ] Drew approval flowchart
- [ ] Understand PlayerId vs ClientId
- [ ] Can explain reconnection data preservation

### 💡 Answers to Day 1 Questions (Finally!)

Now you can answer your Day 1 questions:

**Q: What happens when a player disconnects and reconnects?**
> SessionManager keeps their data (position, HP, character) keyed by PlayerId. When they reconnect with the same PlayerId, SessionManager recognizes them and restores their state.

---

## 📅 Day 7: Week 1 Review & Assessment

### 📖 Prerequisites
**Re-read key sections of:**
- [01_architecture_principles.md](./01_architecture_principles.md)
- [19_game_flow_deepdive.md](./19_game_flow_deepdive.md)

### 🎯 Learning Objectives
- Consolidate Week 1 knowledge
- Identify any gaps
- Prepare for Week 2

### 💻 Self-Assessment Quiz

**Answer WITHOUT looking at notes, then check:**

1. What is the entry point class? 
   > ApplicationController

2. Name 5 services registered in DI container.
   > LocalSession, ProfileManager, ConnectionManager, UpdateRunner, MessageChannel<ConnectStatus>

3. What does PubSub solve?
   > Decouples publishers from subscribers - publisher doesn't need to know who listens

4. Why do states have Enter() and Exit()?
   > To setup/cleanup resources when transitioning, avoiding if-else spaghetti

5. List all 6 connection states.
   > Offline, StartingHost, Hosting, ClientConnecting, ClientConnected, ClientReconnecting

6. What's the difference between PlayerId and ClientId?
   > PlayerId is stable (device-based), ClientId changes each connection (network-assigned)

### Mastery Criteria
You've mastered Week 1 if you can:
- [ ] Explain DI, PubSub, State Machine patterns without notes
- [ ] Navigate to any system without searching
- [ ] Draw connection state diagram from memory
- [ ] Trace startup from launch to MainMenu

---

# WEEK 2: Character System & Combat
> **Goal:** Master the ServerCharacter/ClientCharacter split and Action system.

---

## 📅 Day 8: The Character Split - THE Core Pattern

### 📖 Prerequisites
**Read FULLY:**
- [18_character_system_deepdive.md](./18_character_system_deepdive.md) - **CRITICAL READING**
- [03_networking_essentials.md](./03_networking_essentials.md) - NetworkVariables and RPCs

### 🎯 Learning Objectives
- Understand WHY ServerCharacter and ClientCharacter are separate
- Know what each is responsible for
- Trace communication between them

### 💻 Hands-On Work

#### Part 1: ServerCharacter Study (60 min)
**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs`

**Find and document:**
1. All `NetworkVariable<>` declarations (lines ~50-80)
2. The `OnNetworkSpawn()` pattern - what gets disabled?
3. All `[Rpc(SendTo.Server)]` methods - what do they receive?
4. The `ReceiveHP()` method - trace damage flow

#### Part 2: ClientCharacter Study (45 min)
**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/ClientCharacter.cs`

**Find and document:**
1. How it subscribes to NetworkVariable changes
2. The position lerping pattern (smooth movement)
3. How ClientActionPlayer handles visual effects
4. The `OnNetworkSpawn()` pattern

#### Part 3: Communication Diagram (30 min)
Draw how player presses attack:
```
ClientInputSender → [RPC] → ServerCharacter.ServerPlayActionRpc()
        ↓
ServerActionPlayer.PlayAction()
        ↓
Action.OnStart() (server logic)
        ↓
ClientCharacter.ClientPlayActionRpc() → [RPC] → All clients
        ↓
ClientActionPlayer plays visuals
```

### ✅ Checkpoints
- [ ] Read 18_character_system_deepdive.md completely
- [ ] Listed all NetworkVariables in ServerCharacter
- [ ] Understand the `enabled = false` pattern
- [ ] Drew communication flow diagram

### 💡 Answers to Reflection Questions

**Q: Why not just one PlayerCharacter class with if(IsServer) checks?**
> 1. Constant conditional checks clutter code
> 2. Easy to accidentally run server code on client
> 3. Hard to test either side in isolation
> 4. Mental overhead: "Is this line running on server?"

**Q: How does client show immediate feedback if server is authoritative?**
> Client Anticipation - ClientCharacter plays actions immediately when owner presses button, before server confirms. If server rejects, visual is cancelled.

---

## 📅 Day 9-10: Action System - Part 1 (Architecture)

### 📖 Prerequisites
**Read FULLY:**
- [09_action_system_deepdive.md](./09_action_system_deepdive.md) - Sections 1-5

### 🎯 Learning Objectives (Day 9)
- Understand Action class hierarchy
- Know ActionConfig data-driven design
- Understand ServerActionPlayer queue management

### 🎯 Learning Objectives (Day 10)
- Trace a MeleeAction from start to finish
- Understand blocking vs non-blocking actions
- Know synthesized actions (Chase, Target)

### 💻 Hands-On Work

#### Day 9: Core Classes
**Files to study:**
- `Action.cs` - Base class, lifecycle methods
- `ActionConfig.cs` - ScriptableObject data
- `ServerActionPlayer.cs` - Queue management (focus on PlayAction, StartAction)

**Key insight:** Actions are DATA-DRIVEN - behavior comes from ActionConfig, not code!

#### Day 10: Concrete Actions
**Files to study:**
- `ConcreteActions/MeleeAction.cs` - Simple attack
- `ConcreteActions/LaunchProjectileAction.cs` - Ranged attack
- `ConcreteActions/ChaseAction.cs` - Auto-generated movement

**Trace MeleeAction:**
1. OnStart() - Start animation, reset hit list
2. OnUpdate() - Check for hits, deal damage
3. OnEnd() - Cleanup

### ✅ Checkpoints
- [ ] Read 09_action_system_deepdive.md sections 1-5
- [ ] Understand Action lifecycle (OnStart → OnUpdate → OnEnd)
- [ ] Know what ActionConfig contains
- [ ] Can explain blocking modes

### 💡 Answers to Reflection Questions

**Q: What is the Action lifecycle?**
> OnStart() → OnUpdate() (repeats each frame) → OnEnd() when action completes, or OnCancel() if interrupted.

**Q: What are blocking modes?**
> - **BlocksAllButMovement** - Character can't do other actions while this runs
> - **OnlyDuringExecTime** - Blocks only during execution phase, then becomes non-blocking
> - **NonBlocking** - Runs alongside other actions (e.g., buffs)

**Q: What are synthesized actions?**
> Chase and Target actions auto-inserted by ServerActionPlayer when you use an ability on a target out of range. System automatically moves you in range first.

---

## 📅 Day 11-12: Action System - Part 2 (Implementation)

### 📖 Prerequisites
**Read FULLY:**
- [09_action_system_deepdive.md](./09_action_system_deepdive.md) - Sections 6-12

### 🎯 Learning Objectives
- Understand client anticipation pattern
- Know how ActionFactory pools actions
- Be able to create a new action

### 💻 Hands-On Work

#### Day 11: Client Side & Pooling
**Files to study:**
- `ClientActionPlayer.cs` - Visual playback
- `ActionFactory.cs` - Object pooling

**Key insight:** Actions are POOLED - created once, reused to avoid GC!

#### Day 12: Creating Your Own Action
Follow the guide in 09_action_system_deepdive.md section 11 to:
1. Create a new ActionConfig ScriptableObject
2. Create a new Action class
3. Wire it up and test

### ✅ Checkpoints
- [ ] Finished reading 09_action_system_deepdive.md
- [ ] Understand client anticipation
- [ ] Know action pooling pattern
- [ ] Created (or planned) a custom action

### 💡 Answers to Reflection Questions

**Q: Why pool actions?**
> Actions are created frequently during combat. Without pooling, each action would allocate memory and trigger garbage collection. ActionFactory maintains a pool of pre-created action instances that get reused.

**Q: How does client anticipation work?**
> When the owning client presses an action button, ClientCharacter immediately plays the action visually (via ClientActionPlayer) BEFORE the server confirms. If server rejects the action, the visual is cancelled. This hides network latency.

---

## 📅 Day 13: Character Components & Damage System

### 📖 Prerequisites
**Re-read:**
- [18_character_system_deepdive.md](./18_character_system_deepdive.md) - Component Composition section

### 🎯 Learning Objectives
- Understand component composition pattern
- Trace complete damage flow
- Know health/life state system

### 💻 Hands-On Work

#### Part 1: Component Map (45 min)
ServerCharacter delegates to these components:
- `ServerCharacterMovement` - Physics
- `ServerActionPlayer` - Action queue
- `NetworkHealthState` - HP sync
- `NetworkLifeState` - Alive/Fainted/Dead
- `AIBrain` (NPCs only) - AI decisions

**For each component, find the file and understand its role.**

#### Part 2: Damage Flow Trace (45 min)
Trace from attack to death:
1. `MeleeAction.OnUpdate()` detects hit
2. Calls `IDamageable.ReceiveHP(-25)`
3. `ServerCharacter.ReceiveHP()` applies damage
4. Sets `NetHealthState.HitPoints.Value`
5. NetworkVariable syncs to clients
6. If HP <= 0, sets `LifeState = Fainted/Dead`

#### Part 3: Life State Events (30 min)
Find where `LifeStateChangedEventMessage` is:
- Published (ServerCharacter)
- Subscribed (ServerBossRoomState - for game over check)

### ✅ Checkpoints
- [ ] Mapped all character components
- [ ] Traced complete damage flow
- [ ] Understand health → life state → game over chain

### 💡 Answers to Reflection Questions

**Q: Why use component composition instead of inheritance?**
> Composition is more flexible. You can mix and match components (NPCs have AIBrain, players don't). Inheritance would create rigid hierarchies that are hard to change.

**Q: Where is LifeStateChangedEventMessage published?**
> In `ServerCharacter.cs` when LifeState changes (via NetworkVariable callback).

**Q: What subscribes to it?**
> `ServerBossRoomState` subscribes to check if all players fainted (game over) or boss died (win).

---

## 📅 Day 14: Week 2 Review

### Self-Assessment
- [ ] Can explain ServerCharacter vs ClientCharacter split
- [ ] Can trace action from button press to visual effect
- [ ] Understand blocking vs non-blocking actions
- [ ] Know how damage flows through the system
- [ ] Can draw character component diagram from memory

### 💡 Answers:

1. **Why are ServerCharacter and ClientCharacter separate files?**
   > To separate authority (server) from visualization (client). Server handles truth, client handles display. No if(IsServer) checks needed.

2. **What does ServerActionPlayer's queue manage?**
   > A list of pending actions. Blocking actions execute one at a time. Non-blocking actions run in parallel. Queue handles timing, cooldowns, and synthesized actions.

3. **How does client anticipation work?**
   > Owner's client plays action visually immediately when button pressed, before server confirms. If server rejects, visual cancelled.

4. **What triggers a game over check?**
   > LifeStateChangedEventMessage. ServerBossRoomState checks if all players fainted (loss) or boss died (win).

---

# WEEK 3: Game Flow, UI & Scenes
> **Goal:** Master scene transitions, game states, and networked UI.

---

## 📅 Day 15: Game States & Scene Flow

### 📖 Prerequisites
**Read FULLY:**
- [19_game_flow_deepdive.md](./19_game_flow_deepdive.md) - Scene Flow & GameStateBehaviour sections
- [12_system_flow_diagrams.md](./12_system_flow_diagrams.md) - Scene diagrams

### 🎯 Learning Objectives
- Understand GameStateBehaviour pattern
- Know all scene transitions and triggers
- Understand Server vs Client state files

### 💻 Hands-On Work

**Files to study:**
- `GameStateBehaviour.cs` - Base class
- `ServerBossRoomState.cs` - Server gameplay logic
- `ServerCharSelectState.cs` - Server lobby logic
- `ClientCharSelectState.cs` - Client lobby UI

**Map all scene transitions:**
| From | To | Trigger | Who Triggers |
|------|----|---------| -------------|
| MainMenu | CharSelect | Host/Join success | HostingState.Enter() |
| CharSelect | BossRoom | All locked in | ServerCharSelectState |
| BossRoom | PostGame | Boss dies / All fainted | ServerBossRoomState |
| PostGame | MainMenu | Player clicks Return | ClientPostGameState |

### ✅ Checkpoints
- [ ] Read 19_game_flow_deepdive.md completely
- [ ] Understand GameStateBehaviour pattern
- [ ] Mapped all scene transitions
- [ ] Know difference between Server and Client states

### 💡 Answers to Reflection Questions

**Q: Why separate Server and Client state files?**
> Same reason as characters! ServerBossRoomState handles spawning, game over logic. ClientBossRoomState would handle UI. Keeps responsibilities clear.

**Q: How does NetworkSceneManager differ from regular SceneManager?**
> NetworkSceneManager synchronizes scene loads across all connected clients. Regular SceneManager only loads locally.

---

## 📅 Day 16-17: Character Selection Deep Dive

### 📖 Prerequisites
**Reference:**
- [03_networking_essentials.md](./03_networking_essentials.md)

### 🎯 Learning Objectives
- Understand seat/lobby synchronization
- Know LobbyPlayerState pattern
- Trace character lock-in flow

### 💻 Hands-On Work
**Files:**
- `ServerCharSelectState.cs` - Seat management
- `ClientCharSelectState.cs` - UI updates
- `LobbyPlayerState` related files

**Trace these flows:**
1. Player A selects Tank → How do other clients see it?
2. Player A locks in → What prevents changing?
3. All players locked → What triggers scene load?

### 💡 Answers:

1. **Player A selects Tank → How do other clients see it?**
   > NetworkVariable in LobbyPlayerState syncs seat selection. ClientCharSelectState subscribes to changes and updates UI.

2. **Player A locks in → What prevents changing?**
   > Server sets a `IsLockedIn` flag. Server rejects any further seat change requests from that client.

3. **All players locked → What triggers scene load?**
   > ServerCharSelectState checks after each lock-in. When all connected players are locked, it calls NetworkSceneManager.LoadScene("BossRoom").

---

## 📅 Day 18-19: UI Patterns

### 📖 Prerequisites
**Reference:**
- [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - For event patterns

### 🎯 Learning Objectives
- Understand UI → Game communication
- Know how UI subscribes to game events
- Understand networked UI updates

### 💻 Hands-On Work
**Explore:** `Assets/Scripts/Gameplay/UI/`
- How does health bar know when health changes?
- How does lobby UI know seat state?
- Find examples of UI subscribing to MessageChannels

### 💡 Answers:

**Q: How does health bar know when health changes?**
> It subscribes to `NetworkHealthState.HitPoints.OnValueChanged`. When server updates HP, the NetworkVariable syncs and triggers the callback.

**Q: How does lobby UI know seat state?**
> It subscribes to NetworkVariables in LobbyPlayerState, or uses PubSub messages for seat changes.

---

## 📅 Day 20-21: Week 3 Review & Practice

### Practical Exercise
Trace the complete flow:
1. Two players launch game
2. Both click Host/Join
3. Both select characters and lock in
4. Game loads, they fight boss
5. Boss dies, game ends

### 💡 Complete Flow Answer:

| Step | Files & Methods |
|------|----------------|
| 1. Launch | ApplicationController.Start() → LoadScene("MainMenu") |
| 2. Host/Join | ConnectionManager → HostingState or ClientConnectingState |
| 3. CharSelect | ServerCharSelectState manages seats, all lock → LoadScene("BossRoom") |
| 4. Combat | ServerBossRoomState.SpawnPlayer(), Actions via ServerActionPlayer |
| 5. Boss dies | LifeStateChangedEventMessage → ServerBossRoomState.BossDefeated() → LoadScene("PostGame") |

---

# WEEK 4: Infrastructure, Polish & Mastery
> **Goal:** Master remaining patterns, create your reference, and prove mastery.

---

## 📅 Day 22: Infrastructure Patterns

### 📖 Prerequisites
**Read FULLY:**
- [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - Complete guide
- [16_performance_patterns.md](./16_performance_patterns.md) - Object pooling section

### 🎯 Learning Objectives
- Master NetworkObjectPool pattern
- Understand DisposableGroup cleanup
- Know UpdateRunner for non-MonoBehaviours

### 💻 Hands-On Work
**Files:**
- `NetworkObjectPool.cs`
- `DisposableGroup.cs`
- `UpdateRunner.cs`

### 💡 Answers:

**Q: What does NetworkObjectPool do?**
> Pre-instantiates networked prefabs (like projectiles) at startup. Instead of Instantiate/Destroy, it reuses pooled objects to avoid GC.

**Q: What is DisposableGroup?**
> Collects multiple IDisposable subscriptions so you can dispose them all at once in OnDestroy(). Prevents forgetting to unsubscribe.

**Q: Why UpdateRunner?**
> Non-MonoBehaviour classes can't use Update(). UpdateRunner provides a centralized update loop they can subscribe to.

---

## 📅 Day 23: Clean Code & SOLID

### 📖 Prerequisites
**Read FULLY:**
- [02_clean_code_patterns.md](./02_clean_code_patterns.md)
- [14_antipatterns_guide.md](./14_antipatterns_guide.md)

### 🎯 Learning Objectives
- Identify SOLID principles in Boss Room
- Recognize antipatterns and their fixes
- Apply principles to your own code

### 💡 Answers:

**Q: Find Single Responsibility example?**
> ServerCharacter handles game logic, ClientCharacter handles visuals. Each has one job.

**Q: Find Dependency Inversion example?**
> Classes inject IPublisher<T> interface, not concrete MessageChannel. Allows mock injection for tests.

**Q: Common antipattern to avoid?**
> God class - putting all game logic in one GameManager. Boss Room splits into ConnectionManager, ActionPlayer, etc.

---

## 📅 Day 24: Performance Patterns

### 📖 Prerequisites
**Read FULLY:**
- [16_performance_patterns.md](./16_performance_patterns.md)

### 🎯 Learning Objectives
- Understand GC avoidance techniques
- Know when to use struct vs class
- Master caching patterns

### 💡 Answers:

**Q: When use struct vs class?**
> Struct for small, short-lived data (ActionRequestData). Class for long-lived objects with identity (ServerCharacter).

**Q: How to avoid GC in Update loops?**
> Cache references, use object pools, avoid string concatenation, pre-allocate lists.

**Q: Example of caching in Boss Room?**
> ActionFactory caches created Action instances instead of creating new ones.

---

## 📅 Day 25: Architecture Decisions

### 📖 Prerequisites
**Read FULLY:**
- [17_architecture_decision_framework.md](./17_architecture_decision_framework.md)
- [06_offline_game_guide.md](./06_offline_game_guide.md)

### 🎯 Learning Objectives
- Know which pattern to use when
- Understand what applies to offline games
- Make architecture decisions confidently

### 💡 Answers:

**Q: Which patterns apply to offline games?**
> DI, PubSub, State Machine, Data-Driven Design, Object Pooling. Skip: NetworkVariables, RPCs, server/client split.

**Q: When to use State Machine vs if/else?**
> State Machine when you have 3+ states with different behaviors. If/else for 2 simple states.

**Q: When to use PubSub vs direct call?**
> PubSub when multiple systems react to one event. Direct call for 1:1 relationships.

---

## 📅 Day 26-27: Code Reading Exercises

### 📖 Prerequisites
**Complete ALL exercises in:**
- [13_code_reading_walkthroughs.md](./13_code_reading_walkthroughs.md)

### 🎯 Learning Objectives
- Navigate code independently
- Trace unfamiliar flows
- Build investigation skills

### 💡 Tip for Code Reading:

**Investigation pattern:**
1. Start at the "trigger" (button click, RPC receive)
2. Follow method calls with F12/Go to Definition
3. Note class names and their patterns (Server*, Client*, *State)
4. Draw the flow as you go
5. Verify in debugger if unsure

---

## 📅 Day 28: Create Your Reference

### Exercise
Create `Documentation/MyNotes/my_reference.md` containing:
1. **Pattern Cheat Sheet** - One-sentence explanations
2. **File Quick Reference** - Where to find each feature
3. **Decision Guide** - Which pattern for which problem
4. **Personal Insights** - What you learned

---

## 📅 Day 29: Capstone Project

### Choose One:

**Option A: Add an Ability**
- Create ActionConfig for a new spell
- Implement Action class
- Test in multiplayer

**Option B: Add a Feature**
- Implement a scoreboard
- Add a countdown timer
- Create networked UI

**Option C: Design Your Game**
- Draw architecture diagram
- List patterns you'd use
- Write DI container setup

---

## 📅 Day 30: Graduation

### Final Assessment
You should be able to:
- [ ] Explain any Boss Room system without notes
- [ ] Navigate to any feature in under 30 seconds
- [ ] Justify architectural decisions
- [ ] Apply patterns to new problems
- [ ] Teach someone else what you learned

### 🎓 Congratulations!
You now possess professional game architecture knowledge. Go build something amazing!

---

## 📚 Complete Guide Reference

### Guides Read During This 30-Day Journey:

| Day | Guides |
|-----|--------|
| 2 | 05_project_structure, A1_code_navigation_guide |
| 3 | 01_architecture_principles, 19_game_flow_deepdive |
| 4 | 04_design_patterns, 11_infrastructure_patterns |
| 5 | 10_connection_state_machine, 04_design_patterns |
| 6 | 20_session_reconnection |
| 8 | 18_character_system_deepdive, 03_networking_essentials |
| 9-12 | 09_action_system_deepdive |
| 15 | 19_game_flow_deepdive, 12_system_flow_diagrams |
| 22 | 11_infrastructure_patterns, 16_performance_patterns |
| 23 | 02_clean_code_patterns, 14_antipatterns_guide |
| 24 | 16_performance_patterns |
| 25 | 17_architecture_decision_framework, 06_offline_game_guide |
| 26-27 | 13_code_reading_walkthroughs |

**Additional references (read as needed):**
- [07_implementation_templates.md](./07_implementation_templates.md)
- [08_checklist.md](./08_checklist.md)
- [15_testability_debugging.md](./15_testability_debugging.md)
