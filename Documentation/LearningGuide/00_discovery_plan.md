# 🗓️ 30-Day Discovery Plan

> **This is your step-by-step guide to mastering this codebase.** Each day has specific files, exact instructions, and exercises. Just open the day and follow.

---

## How to Use This Plan

1. **One day = 1-2 hours of study** (adjust to your pace)
2. **Keep a notebook** - write down everything asked
3. **Don't skip exercises** - they cement understanding
4. **If stuck, re-read the previous day**
5. **Reference the other docs** in this folder for context

---

## Week 1: Foundation & Entry Point

---

### 📅 Day 1: Play the Game & Overview

**Goal:** Understand what this game does before reading code.

**Steps:**

1. **Open Unity** and run the project
2. **Host a game** - go through the full flow:
   - Main Menu → Character Select → Play → Win or Die
3. **Run a second instance** (Build → Run, or use Multiplayer Play Mode):
   - Join the first game
   - See how character select syncs
   - Fight together
4. **Write in notebook:**
   - What scenes exist? (Check Assets/Scenes folder)
   - What characters can you pick?
   - What abilities did you see?
   - What happens when you disconnect?

**Read:**
- `README.md` (root folder) - 30 minutes

**✅ Done when:** You've played as both host and client, and listed all scenes.

---

### 📅 Day 2: Project Structure

**Goal:** Know where everything is.

**Steps:**

1. **Open** `Assets/Scripts/` in your file explorer
2. **For each subfolder**, open ONE file and read the namespace:
   ```
   Assets/Scripts/
   ├── ApplicationLifecycle/ → Open ApplicationController.cs, read namespace
   ├── ConnectionManagement/ → Open ConnectionManager.cs, read namespace
   ├── Gameplay/             → Open any file, read namespace
   ├── Infrastructure/       → Open any file, read namespace
   └── UnityServices/        → Open any file, read namespace
   ```
3. **Write in notebook:**
   - What does each folder contain?
   - What namespace pattern do they use?

**Read:**
- `Documentation/LearningGuide/05_project_structure.md` - 20 minutes

**✅ Done when:** You can name all 5 script folders and their purpose without looking.

---

### 📅 Day 3: Entry Point (ApplicationController)

**Goal:** Understand how the game boots.

**Open file:** `Assets/Scripts/ApplicationLifecycle/ApplicationController.cs`

**Steps:**

1. **Line 1-30:** Read the imports. What namespaces does this file use?
2. **Line 37-79 (Configure method):** This is Dependency Injection setup
   - For each `builder.Register...` line, write what is being registered
   - Count how many different things are registered
3. **Line 81-97 (Start method):**
   - What scene does it load at the end?
   - What is `Container.Resolve<>` doing?
4. **Line 99-112 (OnDestroy):**
   - What cleanup happens?

**Write in notebook:**
```
ApplicationController is responsible for:
1. ...
2. ...
3. ...

It registers these services:
- LocalSession
- ProfileManager
- MessageChannel<...>
- etc.

Startup flow:
1. Configure() runs
2. Start() runs
3. Loads scene: ____
```

**Read:**
- `Documentation/LearningGuide/01_architecture_principles.md` (Section: Dependency Inversion)

**✅ Done when:** You can explain what DI is and list 5 things registered in Configure().

---

### 📅 Day 4: Message System (PubSub)

**Goal:** Understand decoupled communication.

**Open files in order:**

1. `Assets/Scripts/Infrastructure/PubSub/IMessageChannel.cs`
   - What interfaces are defined?
   - Why does Subscribe return IDisposable?

2. `Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs`
   - How does Publish() work?
   - Why is there a try-catch in Publish()?

3. `Assets/Scripts/Infrastructure/PubSub/NetworkedMessageChannel.cs`
   - What's different from regular MessageChannel?
   - Why does it check IsServer before sending RPC?

**Exercise:**
```
Search the codebase (Ctrl+Shift+F) for "Subscribe("
Find 3 examples of subscription.
For each, write:
- What message is subscribed to?
- What happens when the message arrives?
```

**Write in notebook:**
```
PubSub Pattern:
- Publishers: [don't know about subscribers]
- Subscribers: [don't know about publishers]
- Benefit: [loose coupling]

MessageChannel types:
1. MessageChannel - ...
2. BufferedMessageChannel - ...
3. NetworkedMessageChannel - ...
```

**Read:**
- `Documentation/LearningGuide/04_design_patterns.md` (Pattern 2: Observer)

**✅ Done when:** You can explain PubSub to someone who doesn't know it.

---

### 📅 Day 5: Connection Manager Overview

**Goal:** See the State Machine pattern in action.

**Open file:** `Assets/Scripts/ConnectionManagement/ConnectionManager.cs`

**Steps:**

1. **Lines 1-50:** Read the enums and structs
   - What is `ConnectStatus`? List all values
   - What is `ConnectionPayload`?

2. **Lines 56-80:** Study the class fields
   - What are the state instances? (m_Offline, m_ClientConnecting, etc.)
   - Why are they created upfront, not on demand?

3. **Lines 86-101 (Start):**
   - How are states injected with dependencies?
   - What is the initial state?

4. **Lines 112-122 (ChangeState):**
   - What happens when state changes? (Exit → assign → Enter)

**Draw in notebook:**
```
State Machine:

[Offline] → StartHost → [StartingHost] → Success → [Hosting]
    ↓
StartClient
    ↓
[Connecting] → Success → [Connected]
    ↓
Disconnect
    ↓
[Reconnecting] → Failed → [Offline]
```

**Read:**
- `Documentation/LearningGuide/04_design_patterns.md` (Pattern 1: State Machine)

**✅ Done when:** You can draw the state diagram from memory.

---

### 📅 Day 6: Connection States

**Goal:** Understand each connection state.

**Open each file and fill the table:**

| File | Enter() does | Exit() does | Key Methods |
|------|--------------|-------------|-------------|
| `ConnectionState/OfflineState.cs` | | | |
| `ConnectionState/StartingHostState.cs` | | | |
| `ConnectionState/HostingState.cs` | | | |
| `ConnectionState/ClientConnectingState.cs` | | | |
| `ConnectionState/ClientConnectedState.cs` | | | |
| `ConnectionState/ClientReconnectingState.cs` | | | |

**Focus on HostingState.cs:**
- Find `ApprovalCheck()` method
- What validations happen?
- What happens when rejected?

**Write in notebook:**
```
Connection approval checks:
1. Server full? → Reject with "ServerFull"
2. ...
3. ...

Why server-side validation matters:
- Prevents cheating
- Single source of truth
```

**✅ Done when:** You can trace what happens when a client clicks "Join".

---

### 📅 Day 7: Week 1 Review

**Goal:** Consolidate knowledge.

**Exercises:**

1. **Without looking**, write down:
   - Entry point class name
   - 3 services registered in Configure()
   - 3 connection states

2. **Trace the flow** (write each step):
   - User opens game → ... → Main Menu appears

3. **Trace the flow:**
   - User clicks "Host" → ... → Game starts

**If you can't answer, re-read the relevant day.**

---

## Week 2: Game States & Action System

---

### 📅 Day 8: Game State Behaviour

**Goal:** Understand how game flow works.

**Open file:** `Assets/Scripts/Gameplay/GameState/GameStateBehaviour.cs`

**Steps:**

1. **Read the comments** at the top (lines 14-36) - very valuable!
2. **Study the Start() method:**
   - What is `s_ActiveStateGO`?
   - Why check if previous state `Persists`?
   - What happens when two game states exist?

**Write in notebook:**
```
GameState enum values:
1. MainMenu
2. ...
3. ...
4. ...

Persists property:
- If true: stays across scenes (DontDestroyOnLoad)
- If false: destroyed on scene change

One-at-a-time rule:
- Only ONE game state object per scene
- New state destroys old state (unless both persistent + same type)
```

**✅ Done when:** You can explain why `Persists` matters.

---

### 📅 Day 9: Character Selection

**Goal:** Understand networked lobby.

**Open files:**

1. `Gameplay/GameState/NetworkCharSelection.cs`
   - What is `NetworkList<LobbyPlayerState>`?
   - What data is in `LobbyPlayerState`?

2. `Gameplay/GameState/ServerCharSelectState.cs`
   - How does server handle seat selection?
   - When does it load the game scene?

3. `Gameplay/GameState/ClientCharSelectState.cs`
   - How does client react to NetworkList changes?
   - Does client ever modify the list directly?

**Write in notebook:**
```
Character Selection Flow:
1. Player clicks character
2. Client sends RPC to server
3. Server validates (is character available?)
4. Server updates NetworkList
5. All clients receive update via OnListChanged
6. All clients update UI

Key insight: Client never modifies NetworkList directly!
```

**Read:**
- `Documentation/LearningGuide/03_networking_essentials.md` (Rule 1: Server is Authority)

**✅ Done when:** You can explain why clients don't modify NetworkList directly.

---

### 📅 Day 10: Action System Overview

**Goal:** Understand combat ability architecture.

**Open file:** `Assets/Scripts/Gameplay/Action/Action.cs`

**Steps:**

1. **Lines 1-80:** Read fields and setup
   - What is `Config`?
   - What is `m_Parent`?
   - What tracking exists? (TimeRunning, etc.)

2. **Lines 90-150:** Server lifecycle methods
   - `OnStart()` - when called?
   - `OnUpdate()` - what does return value mean?
   - `End()` vs `Cancel()` - difference?

3. **Lines 200-280:** Client lifecycle methods
   - What is `AnticipateActionClient()`?
   - Why separate server and client methods?

**Draw in notebook:**
```
Action Lifecycle:

SERVER SIDE:
OnStart() → [returns true to continue]
    ↓
OnUpdate() ← [called every frame]
    ↓
[returns false]
    ↓
End()
    ↓
ChainIntoNewAction()?

CLIENT SIDE:
AnticipateActionClient() ← [called IMMEDIATELY on input]
    ↓
OnStartClient() ← [called when server confirms]
    ↓
OnUpdateClient()
    ↓
CancelClient()
```

**✅ Done when:** You can explain anticipation and why it exists.

---

### 📅 Day 11: Action Config (Data-Driven Design)

**Goal:** See data configuration pattern.

**Open file:** `Assets/Scripts/Gameplay/Action/ActionConfig.cs`

**Then open:** `Assets/GameData/Actions/` in Unity (find ScriptableObjects)

**Steps:**

1. Study ActionConfig fields:
   - What timing fields exist?
   - What combat fields exist?
   - What visual fields exist?

2. Open 3 different action configs in Unity Inspector:
   - Compare their values
   - Note the differences

**Write in notebook:**
```
ActionConfig fields:
- ExecTimeSeconds: how long action runs
- EffectTimeSeconds: when damage/effect applies
- Damage: ...
- Range: ...
- Projectile: ...

Data-Driven Benefits:
1. Designers change values, not code
2. Easy to create variations
3. Can balance without recompiling
```

**Read:**
- `Documentation/LearningGuide/01_architecture_principles.md` (Section: Data-Driven Design)

**✅ Done when:** You know why ActionConfig is a ScriptableObject.

---

### 📅 Day 12: Melee Action Deep Dive

**Goal:** Trace a complete action implementation.

**Open files:**

1. `Gameplay/Action/ConcreteActions/MeleeAction.cs`
2. `Gameplay/Action/ConcreteActions/MeleeAction.Client.cs`

**For MeleeAction.cs (Server):**

1. Find `OnStart()`:
   - What does it do?
   - What is `m_DidDamage`?

2. Find `OnUpdate()`:
   - When does damage apply?
   - How does it check timing?

3. Find `DoDamage()`:
   - How does it find targets?
   - What layer mask is used?

**For MeleeAction.Client.cs (Client):**

1. Find `AnticipateActionClient()`:
   - What visual starts immediately?

2. Find `OnStartClient()`:
   - What happens if already anticipated?

**Write in notebook:**
```
MeleeAction Timeline:
T+0.0s: Player presses button
        → AnticipateActionClient() runs
        → Animation starts

T+0.05s: Request reaches server
         → OnStart() runs
         → Server animation starts

T+0.4s: EffectTimeSeconds reached
        → DoDamage() runs
        → OverlapSphere finds targets
        → Targets receive damage

T+1.0s: ExecTimeSeconds reached
        → OnUpdate() returns false
        → Action ends
```

**✅ Done when:** You can trace melee from button press to damage.

---

### 📅 Day 13: Projectile Actions

**Goal:** Understand two projectile approaches.

**Open files:**

1. `ConcreteActions/LaunchProjectileAction.cs`
   - Spawns actual NetworkObject
   - Projectile syncs via NetworkTransform

2. `ConcreteActions/FXProjectileTargetedAction.cs`
   - Uses RPC for visual
   - Damage applies instantly

**Write in notebook:**
```
LaunchProjectile vs FXProjectile:

| Aspect | Launch | FX |
|--------|--------|-----|
| NetworkObject? | Yes | No |
| Damage When? | On collision | Immediately |
| Can Dodge? | Yes | No |
| Use for | Arrows, slow | Magic bolts |
| Network cost | Higher | Lower |
```

**Question to answer:**
Why would you choose one over the other?

**✅ Done when:** You know when to use each approach.

---

### 📅 Day 14: Week 2 Review

**Goal:** Consolidate action system knowledge.

**Exercises:**

1. List all lifecycle methods (server and client) for Action
2. Explain anticipation in 3 sentences
3. Compare blocking modes (find in `BlockingModeType.cs`)
4. Trace: Player clicks Fireball button → explosion on enemy

---

## Week 3: Characters & Gameplay Objects

---

### 📅 Day 15: ServerCharacter

**Goal:** Understand server-side character logic.

**Open file:** `Gameplay/GameplayObjects/Character/ServerCharacter.cs`

**Focus areas:**
1. What components does it reference?
2. How does action queue work?
3. Find `ReceiveHP()` - how does damage work?

**Write in notebook:**
```
ServerCharacter responsibilities:
1. Owns action queue
2. Processes damage
3. Tracks life state
4. Runs on server/host only

Key methods:
- PlayAction(): adds action to queue
- ReceiveHP(): processes damage through buffs
```

**Read:**
- `Documentation/LearningGuide/02_clean_code_patterns.md` (Single Responsibility)

**✅ Done when:** You know what ServerCharacter does NOT handle (visuals!).

---

### 📅 Day 16: ClientCharacter

**Goal:** Understand client-side character display.

**Open file:** `Gameplay/GameplayObjects/Character/ClientCharacter.cs`

**Focus areas:**
1. What does it subscribe to?
2. How does it know when health changes?
3. What visual effects does it control?

**Write in notebook:**
```
ClientCharacter responsibilities:
1. Subscribes to NetworkVariables
2. Updates visuals based on state
3. Plays VFX and animations
4. Runs on all clients

Server/Client Split Pattern:
- Server: game logic (authority)
- Client: visual feedback (display)
```

**✅ Done when:** You can explain why Server/Client are separate classes.

---

### 📅 Day 17: Health & Life States

**Goal:** Understand health synchronization.

**Open files:**

1. `GameplayObjects/NetworkHealthState.cs`
   - What is the NetworkVariable?
   - What is MaxHitPoints?

2. `GameplayObjects/NetworkLifeState.cs`
   - What states exist? (Alive, Fainted, Dead)
   - When does each state apply?

3. `GameplayObjects/IDamageable.cs`
   - What interface method exists?
   - Who implements this?

**Write in notebook:**
```
Health System:
- NetworkVariable<int> HitPoints - synced to all clients
- OnValueChanged event - clients react to changes

Life States:
1. Alive - normal
2. Fainted - can be revived
3. Dead - permanent

IDamageable interface:
- ReceiveHP(source, amount)
- Implemented by: ServerCharacter, Breakable, ...
```

**✅ Done when:** You can trace health damage from hit to UI update.

---

### 📅 Day 18: Movement System

**Goal:** Understand character movement.

**Open file:** `Character/ServerCharacterMovement.cs`

**Focus areas:**
1. How does SetDestination() work?
2. What blocks movement?
3. How are buffs applied to speed?

**Write in notebook:**
```
Movement Flow:
1. Player clicks ground
2. Client sends position to server
3. Server validates (can move? not stunned?)
4. Server calls NavMeshAgent.SetDestination()
5. Position syncs via NetworkTransform
6. Clients see smooth movement
```

**✅ Done when:** You know why movement is server-authoritative.

---

### 📅 Day 19: Interactive Objects

**Goal:** Understand game world objects.

**Open files:**

1. `GameplayObjects/Breakable.cs` - destructible objects
2. `GameplayObjects/FloorSwitch.cs` - pressure plates
3. `GameplayObjects/SwitchedDoor.cs` - doors

**Write in notebook:**
```
Interactive Object Pattern:
1. NetworkVariable for state (IsOpen, IsBroken)
2. Server modifies state (authority)
3. Clients react via OnValueChanged
4. Visual updates on all clients

Example - Door:
- Server: Sets IsOpen when switch pressed
- Clients: Play open animation when IsOpen changes
```

**✅ Done when:** You see the same Server-modifies, Client-reacts pattern.

---

### 📅 Day 20: Spawning & Pooling

**Goal:** Understand object creation.

**Open files:**

1. `GameplayObjects/ServerWaveSpawner.cs` - wave spawning
2. `Infrastructure/NetworkObjectPool.cs` - object pooling

**Write in notebook:**
```
Wave Spawner:
1. Has list of wave configs
2. Spawns enemies with delay
3. Waits for wave clear
4. Tracks alive enemies

Object Pooling:
- Pre-instantiate objects
- Reuse instead of destroy
- Reduces garbage collection
- Critical for performance

Pool Pattern:
Get() - retrieve from pool (or create new)
Return() - put back in pool (deactivate)
```

**Read:**
- `Documentation/LearningGuide/04_design_patterns.md` (Pattern 4: Object Pool)

**✅ Done when:** You know why pooling matters for multiplayer.

---

### 📅 Day 21: Week 3 Review

**Goal:** Consolidate character & object knowledge.

**Exercises:**

1. Explain Server/Client character split in 3 sentences
2. Draw health damage flow diagram
3. List 3 types of interactive objects and their patterns
4. Explain when to use object pooling

---

## Week 4: UI, Services & Synthesis

---

### 📅 Day 22: UI Mediator Pattern

**Goal:** Understand UI architecture.

**Open file:** `Gameplay/UI/IPUIMediator.cs`

**Focus areas:**
1. What UI elements does it reference?
2. What game systems does it call?
3. How does it handle feedback?

**Write in notebook:**
```
Mediator Pattern:
- UI elements don't know about game systems
- Game systems don't know about UI
- Mediator coordinates between them

Benefits:
- Easy to swap UI
- Easy to test
- Decoupled code
```

**Read:**
- `Documentation/LearningGuide/04_design_patterns.md` (Pattern 8: Mediator)

**✅ Done when:** You can explain why UI doesn't call ConnectionManager directly.

---

### 📅 Day 23: HUD & Action Bar

**Goal:** Understand gameplay UI.

**Open files:**

1. `Gameplay/UI/HeroActionBar.cs` - ability buttons
2. `Gameplay/UI/PartyHUD.cs` - team health bars
3. `Gameplay/UI/UIHealth.cs` - single health bar

**Write in notebook:**
```
HeroActionBar:
- Shows ability buttons
- Tracks cooldowns
- Sends action requests

PartyHUD:
- Subscribes to player spawns
- Tracks all player health
- Updates bars on NetworkVariable changes
```

**✅ Done when:** You can trace ability button click to action execution.

---

### 📅 Day 24: Authentication

**Goal:** Understand Unity Services auth.

**Open file:** `UnityServices/Auth/AuthenticationServiceFacade.cs`

**Write in notebook:**
```
Authentication Flow:
1. Initialize Unity Services
2. Check if already signed in
3. If not, sign in anonymously
4. Get PlayerId for session

Why facade?
- Simplifies complex service
- Handles errors
- Provides clean API
```

**Read:**
- `Documentation/LearningGuide/04_design_patterns.md` (Pattern notes about Facade)

**✅ Done when:** You know what anonymous authentication means.

---

### 📅 Day 25: Sessions & Relay

**Goal:** Understand multiplayer services.

**Open file:** `ConnectionManagement/ConnectionMethod.cs`

**Study both classes:**
1. `ConnectionMethodIP` - direct connection
2. `ConnectionMethodRelay` - relay service

**Write in notebook:**
```
ConnectionMethodIP:
- Direct connection
- Requires port forwarding
- Good for LAN

ConnectionMethodRelay:
- Uses Unity Relay
- Works through NAT
- No port forwarding needed
- Creates allocation, gets join code
```

**Read:**
- `Documentation/LearningGuide/03_networking_essentials.md` (All sections)

**✅ Done when:** You can explain when to use IP vs Relay.

---

### 📅 Day 26: Full Flow Trace

**Goal:** Connect all knowledge.

**Exercise:** Trace complete flow for each scenario:

**Scenario 1: Host starts game**
```
1. User clicks "Host"
2. [Fill each step]
3. ...
10. Players are in BossRoom scene
```

**Scenario 2: Player attacks enemy**
```
1. Player presses ability key
2. [Fill each step]
3. ...
8. Enemy health bar updates
```

**Scenario 3: Player disconnects mid-game**
```
1. Network drops
2. [Fill each step]
3. ...
6. Player either reconnects or goes to menu
```

**✅ Done when:** You can trace all 3 scenarios completely.

---

### 📅 Day 27: Pattern Identification

**Goal:** See patterns everywhere.

**Exercise:** For each pattern, find 2 examples in code:

| Pattern | Example 1 | Example 2 |
|---------|-----------|-----------|
| State Machine | ConnectionState | |
| PubSub | MessageChannel | |
| Factory | ActionFactory | |
| Object Pool | NetworkObjectPool | |
| Mediator | IPUIMediator | |
| Strategy | ConnectionMethod | |

**✅ Done when:** Table is complete.

---

### 📅 Day 28: Clean Code Review

**Goal:** Identify clean code practices.

**Exercise:** Find examples of:

1. **Single Responsibility** - find 2 classes that do ONE thing
2. **Interface Segregation** - find 2 small interfaces
3. **Guard Clauses** - find 3 early returns
4. **Naming Conventions** - list the prefixes used (m_, s_, etc.)

**Read:**
- `Documentation/LearningGuide/02_clean_code_patterns.md` (entire document)

**✅ Done when:** You have examples for each practice.

---

### 📅 Day 29: Your Architecture Document

**Goal:** Create your own reference.

**Create a new file:** `Documentation/MyNotes/architecture_summary.md`

**Write your own summary covering:**
1. Project structure (folders and purpose)
2. Key patterns used (with one-sentence explanations)
3. Main systems and how they connect
4. Things you'd do differently for YOUR game

**✅ Done when:** You have a 1-2 page summary you fully understand.

---

### 📅 Day 30: Capstone & Next Steps

**Goal:** Apply knowledge.

**Choose ONE:**

**Option A: Add a new ability**
- Create new ActionConfig
- Create new Action class
- Add to character

**Option B: Modify existing system**
- Add a new connection status
- Add a new life state
- Add a new UI element

**Option C: Document for your game**
- Write a plan for your own game
- Choose which patterns to use
- Sketch the architecture

**✅ Done when:** You've built or planned something NEW.

---

## 🎓 Congratulations!

You've completed the discovery plan. You now understand:

- ✅ Clean architecture principles
- ✅ Multiplayer networking patterns  
- ✅ Common design patterns
- ✅ Professional code organization
- ✅ How to read and understand large codebases

**What's next?**
- Build your own game using these patterns
- Return to specific days when you need a refresher
- Use `08_checklist.md` when starting new projects
