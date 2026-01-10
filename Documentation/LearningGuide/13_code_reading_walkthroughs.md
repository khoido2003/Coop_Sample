# 13: Code Reading Walkthroughs

> **Purpose:** Guided exercises to help you read and understand the codebase yourself. Each walkthrough poses questions, gives hints, and provides answers you can check after attempting.

---

## Table of Contents

1. [How to Use This Guide](#how-to-use-this-guide)
2. [Walkthrough 1: Trace a Character Spawn](#walkthrough-1-trace-a-character-spawn)
3. [Walkthrough 2: Follow the Damage](#walkthrough-2-follow-the-damage)
4. [Walkthrough 3: Understand Ability Cooldowns](#walkthrough-3-understand-ability-cooldowns)
5. [Walkthrough 4: Session Player Data Flow](#walkthrough-4-session-player-data-flow)
6. [Walkthrough 5: Find All Subscribers to an Event](#walkthrough-5-find-all-subscribers-to-an-event)
7. [Walkthrough 6: Adding a New Character Class](#walkthrough-6-adding-a-new-character-class)
8. [Self-Assessment Checklist](#self-assessment-checklist)

---

## How to Use This Guide

Each walkthrough follows this structure:

1. **The Question** - What you're trying to understand
2. **Starting Point** - Where to begin looking
3. **Hints** - Tips if you get stuck (try without first!)
4. **Key Findings** - What you should discover
5. **Deep Dive Questions** - Extra challenges

**Pro Tips:**
- Use your IDE's "Find Usages" (F12 / Ctrl+Click) extensively
- Search the codebase with terms you find in the code
- Read comments—they often explain the "why"
- Draw diagrams as you explore

---

## Walkthrough 1: Trace a Character Spawn

### The Question

When a player selects a character and the game starts, how does their character prefab get spawned? Trace from character selection to the character appearing in the game scene.

### Starting Point

Open `CharSelectData.cs` in the CharacterSelect folder. Find the `LobbyPlayerState` struct.

**File:** `Assets/Scripts/Gameplay/UI/Lobby/CharSelectData.cs`

### Hints

<details>
<summary>Hint 1: Where is seat selection tracked?</summary>

Look for `SeatIdx` in `LobbyPlayerState`. This is stored in a `NetworkList<LobbyPlayerState>`.

</details>

<details>
<summary>Hint 2: Who owns the NetworkList?</summary>

Search for classes that declare `NetworkList<LobbyPlayerState>`. There's a server-side component that manages this.

</details>

<details>
<summary>Hint 3: How does seat index become a prefab?</summary>

The seat index maps to an Avatar in a list. Search for "Avatar" and find how it connects to character data.

</details>

<details>
<summary>Hint 4: Where does spawning happen?</summary>

Look for `ServerBossRoomState` or similar. The scene transition handler is responsible for spawning.

</details>

### Key Findings

You should discover:

1. **LobbyPlayerState** stores `SeatIdx` which corresponds to a character class
2. **ServerCharSelectState** manages a `NetworkList<LobbyPlayerState>`
3. **CharacterClass** ScriptableObjects define each character's prefab, abilities, etc.
4. **ServerBossRoomState** reads player data and spawns the correct prefab

```
CharSelect Scene                          BossRoom Scene
                                         
LobbyPlayerState.SeatIdx ─────────────────► CharacterClass[SeatIdx]
                   │                                │
                   ▼                                ▼
     Which avatar?                          Which prefab?
                                                   │
                                                   ▼
                                          Spawn NetworkObject
```

### Deep Dive Questions

1. What happens if a player disconnects after selecting but before spawning?
2. How does the spawn position get determined?
3. Can you find where the player's NetworkGuid (for reconnection) is assigned?

---

## Walkthrough 2: Follow the Damage

### The Question

When an enemy hits a player, trace the complete path from collision detection to the health bar updating on all clients.

### Starting Point

Open `ServerCharacter.cs` and find `ReceiveHitPoints`.

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs`

### Hints

<details>
<summary>Hint 1: Where is damage calculated?</summary>

The action that deals damage calls `ReceiveHitPoints`. Look at `MeleeAction.OnUpdate()` to see where it applies damage.

</details>

<details>
<summary>Hint 2: How does health sync?</summary>

Search for `NetworkVariable` and "HitPoints" to find where health state is stored.

</details>

<details>
<summary>Hint 3: Who listens to health changes?</summary>

Find `OnValueChanged` subscriptions for the health NetworkVariable. There's a client-side component that updates UI.

</details>

### Key Findings

You should discover:

1. **IDamageable** interface defines `ReceiveHitPoints`
2. **ServerCharacter** implements IDamageable
3. **NetworkHealthState** contains `NetworkVariable<int> HitPoints`
4. **ClientCharacter** subscribes to `OnValueChanged` and updates visuals
5. Damage can be modified by buffs via `GetBuffedValue(BuffableValue.PercentDamageReceived)`

```
MeleeAction.OnUpdate()
        │
        ▼ (server)
IDamageable.ReceiveHitPoints(-25)
        │
        ▼ (server)
ServerCharacter: Apply buff modifiers
        │
        ▼ (server)
NetworkHealthState.HitPoints.Value += damage
        │
        ▼ (automatic sync)
[All Clients] OnValueChanged callback
        │
        ▼ (client)
ClientCharacter: Update health bar, play VFX
```

### Deep Dive Questions

1. What happens when health reaches 0? (Search for `LifeStateType.Fainted`)
2. How does healing work? Is it the same path with positive numbers?
3. Can you find where damage numbers are shown on screen?

---

## Walkthrough 3: Understand Ability Cooldowns

### The Question

After using an ability, how does the game prevent you from using it again immediately? Find the cooldown system.

### Starting Point

Open `ActionConfig.cs` and find `ReuseTimeSeconds`.

**File:** `Assets/Scripts/Gameplay/Action/ActionConfig.cs`

### Hints

<details>
<summary>Hint 1: Who checks cooldowns?</summary>

Search for where `ReuseTimeSeconds` is read. There's a player class that tracks when actions were last used.

</details>

<details>
<summary>Hint 2: How are timestamps stored?</summary>

Look for `Dictionary` in `ServerActionPlayer`. You'll find a mapping from ActionID to last-used time.

</details>

<details>
<summary>Hint 3: When is the check performed?</summary>

Find `IsReuseTimeElapsed` method. See when it's called in the action flow.

</details>

### Key Findings

You should discover:

1. **ActionConfig.ReuseTimeSeconds** defines cooldown duration
2. **ServerActionPlayer** has `m_LastUsedTimestamps` dictionary
3. **StartAction()** checks `if (Time.time - lastTimeUsed < ReuseTimeSeconds)`
4. Timestamp is recorded when action successfully starts
5. If action is **cancelled**, the timestamp is removed (no cooldown)

```csharp
// From ServerActionPlayer.cs
private Dictionary<ActionID, float> m_LastUsedTimestamps;

public bool IsReuseTimeElapsed(ActionID actionID)
{
    if (m_LastUsedTimestamps.TryGetValue(actionID, out float lastTimeUsed))
    {
        var abilityConfig = GameDataSource.Instance.GetActionPrototypeByID(actionID).Config;
        float reuseTime = abilityConfig.ReuseTimeSeconds;
        
        if (reuseTime > 0 && Time.time - lastTimeUsed < reuseTime)
        {
            return false;  // Still on cooldown!
        }
    }
    return true;  // Ready to use
}
```

### Deep Dive Questions

1. What happens to the cooldown UI on the client? (Hint: Look for ability bar scripts)
2. Is cooldown validated server-side, client-side, or both?
3. What happens if an action with cooldown is queued behind another action?

---

## Walkthrough 4: Session Player Data Flow

### The Question

When a player joins mid-game or reconnects, how does the system know which character was theirs? Trace the session/player data system.

### Starting Point

Open `SessionManager.cs` in the UnityServices folder.

**File:** `Assets/Scripts/UnityServices/Sessions/SessionManager.cs`

### Hints

<details>
<summary>Hint 1: What is a "PlayerId"?</summary>

Search for where PlayerId comes from. It can be from Unity Authentication or a local GUID.

</details>

<details>
<summary>Hint 2: How is player data stored?</summary>

Look for `SessionPlayerData` struct. Find what data it contains and how it's keyed.

</details>

<details>
<summary>Hint 3: How does reconnection use this?</summary>

When a client connects, find where `IsDuplicateConnection` is checked and what happens.

</details>

### Key Findings

You should discover:

1. **PlayerId** is persistent (survives app restart)—from Auth or local GUID
2. **SessionPlayerData** contains: playerId, playerName, NetworkGuid, characterIndex
3. **SessionManager<T>** maps PlayerId → SessionPlayerData
4. On connect, server checks if PlayerId already exists
5. If reconnecting, player gets their **same NetworkGuid** back

```
First Connection:
ClientId (temporary) ───► SessionManager ───► Assign new NetworkGuid
                               │
                               └─ Store: PlayerId → SessionPlayerData

Reconnection:
ClientId (new, temporary) ───► SessionManager ───► Find by PlayerId
                                    │
                                    └─ Return SAME NetworkGuid
                                         │
                                         └─ Reconnect to existing character!
```

### Deep Dive Questions

1. What's the difference between `ClientId` (Netcode) and `PlayerId` (Session)?
2. How long does session data persist after disconnect?
3. Find where `SetupConnectingPlayerSessionData` is called.

---

## Walkthrough 5: Find All Subscribers to an Event

### The Question

The `ConnectStatus` enum is used for connection events. Find ALL places that publish AND subscribe to these messages.

### Starting Point

Search the codebase for `IPublisher<ConnectStatus>` and `ISubscriber<ConnectStatus>`.

### Hints

<details>
<summary>Hint 1: Where are message channels registered?</summary>

Look in `ApplicationController.cs` for DI container setup. Find where MessageChannels are registered.

</details>

<details>
<summary>Hint 2: What do publishers look like?</summary>

Classes that want to publish inject `IPublisher<ConnectStatus>`. Search for this pattern.

</details>

<details>
<summary>Hint 3: What do subscribers look like?</summary>

Classes that want to listen inject `ISubscriber<ConnectStatus>`. They call `Subscribe()` in their initialization.

</details>

### Key Findings

**Publishers** (send messages):

| Class | When it publishes |
|-------|-------------------|
| HostingState | Client connected/disconnected |
| ClientReconnectingState | Failed to reconnect |
| ClientConnectedState | Disconnected unexpectedly |
| ClientConnectingState | Connection failed |

**Subscribers** (receive messages):

| Class | What it does |
|-------|--------------|
| MainMenuUI | Shows error messages |
| ConnectionStatusUI | Updates status display |
| LobbyUIMediator | Handles lobby-specific reactions |

### Deep Dive Questions

1. Is there a `NetworkedMessageChannel<ConnectStatus>`? Why or why not?
2. Trace what UI appears when `ConnectStatus.ServerFull` is published.
3. How would you add a new ConnectStatus value?

---

## Walkthrough 6: Adding a New Character Class

### The Question

If you wanted to add a new playable character class (e.g., "Healer"), what files would you need to create or modify?

### Starting Point

Examine an existing character class. Find the Archer or Mage ScriptableObject assets.

**Folder:** `Assets/GameData/Character/`

### Hints

<details>
<summary>Hint 1: What defines a character class?</summary>

Look for `CharacterClass` ScriptableObject. What fields does it have?

</details>

<details>
<summary>Hint 2: What about their abilities?</summary>

Each character has Action assets. Find where these are assigned.

</details>

<details>
<summary>Hint 3: What about the character select?</summary>

The lobby needs to know about available characters. Search for how the seat count is determined.

</details>

### Key Findings

To add a new character, you need:

1. **Character prefab** (with ServerCharacter, ClientCharacter components)
2. **CharacterClass ScriptableObject** (stats, base actions, prefab reference)
3. **Action ScriptableObjects** for their abilities
4. **UI assets** (portrait, lobby icon)
5. Register in **GameDataSource** or character registry
6. Add seat in **CharSelectData** configuration

```
New Character Checklist:
├─ Prefabs/
│   └─ Characters/
│       └─ Healer.prefab
├─ GameData/
│   ├─ Character/
│   │   └─ Healer_CharacterClass.asset
│   └─ Actions/
│       ├─ Healer_BasicHeal.asset
│       └─ Healer_AreaHeal.asset
├─ Art/
│   └─ UI/
│       └─ HealerPortrait.png
└─ Update GameDataSource.InjectModule()
```

### Deep Dive Questions

1. How are character-specific animations handled?
2. What determines the character's base attack pattern?
3. How would you balance a new character's stats?

---

## Self-Assessment Checklist

After completing these walkthroughs, you should be able to:

### Core Understanding

- [ ] Explain how a player's input becomes an action on the server
- [ ] Trace data flow from server to all clients via NetworkVariables
- [ ] Describe the state machine pattern used for connections
- [ ] Explain why client anticipation exists and how it works

### Code Navigation

- [ ] Find where any Action is configured (ScriptableObject location)
- [ ] Locate the server-side logic for any gameplay feature
- [ ] Identify which component is responsible for a given behavior
- [ ] Trace a message from publisher to all subscribers

### Patterns Recognition

- [ ] Identify State Machine pattern in code
- [ ] Recognize Factory pattern usage
- [ ] Find examples of Object Pooling
- [ ] Locate Strategy pattern implementations

### Modification Skills

- [ ] Know what files to modify to add a new action
- [ ] Understand how to add a new connection state
- [ ] Know how to create a new message channel
- [ ] Understand how to add a new character class

---

## Additional Walkthroughs

---

## Walkthrough 7: Understanding the Input → Action Flow

### The Question

When a player clicks to move or presses an ability key, what is the complete path from input to the action executing on the server?

### Starting Point

**File:** `Assets/Scripts/Gameplay/Input/ClientInputSender.cs`

### Hints

<details>
<summary>Hint 1: Where does input get captured?</summary>

Look for Unity's new Input System callbacks or `Update()` reading input. `ClientInputSender` is attached to the player's character.

</details>

<details>
<summary>Hint 2: How does it reach the server?</summary>

Look for `Rpc` attributes or methods that call RPCs on `ServerCharacter`.

</details>

<details>
<summary>Hint 3: What happens on the server?</summary>

Find `ServerPlayActionRpc` in ServerCharacter. It calls into ServerActionPlayer.

</details>

### Key Findings

```
┌──────────────────────────────────────────────────────────────────────┐
│                       INPUT → ACTION FLOW                             │
└──────────────────────────────────────────────────────────────────────┘

1. Player presses ability key
         │
         ▼
2. ClientInputSender.OnAction1()
         │
         ├─► Create ActionRequestData
         │
         ▼
3. ClientCharacter.AnticipateAction()     ← Client anticipation (visuals)
         │
         ▼
4. ServerCharacter.ServerPlayActionRpc()  ← RPC to server
         │
         ▼
5. ServerCharacter.PlayAction()
         │
         ▼
6. ServerActionPlayer.PlayAction()
         │
         ├─► Check cooldown
         ├─► Synthesize Chase/Target if needed
         │
         ▼
7. Action.OnStart() → OnUpdate() → OnEnd()  ← Server logic runs
         │
         ▼
8. ClientCharacter.ClientPlayActionRpc()  ← RPC to all clients
         │
         ▼
9. ClientActionPlayer.PlayAction()        ← All clients see visuals
```

### Key Code Locations

| Step | File | Method |
|------|------|--------|
| Input capture | ClientInputSender.cs | OnAction1(), OnClick() |
| Anticipation | ClientCharacter.cs | AnticipateAction() |
| Server RPC | ServerCharacter.cs | ServerPlayActionRpc() |
| Queue management | ServerActionPlayer.cs | PlayAction() |
| Action execution | Action.cs (subclasses) | OnStart/OnUpdate/OnEnd |
| Client sync | ClientCharacter.cs | ClientPlayActionRpc() |

### Answer: The Complete Flow

```csharp
// 1. ClientInputSender reads input
void OnAction1(InputAction.CallbackContext context)
{
    var data = new ActionRequestData { ActionID = ability1ID };
    m_ServerCharacter.ServerPlayActionRpc(data);
    m_ClientCharacter.AnticipateAction(ref data);  // Local feedback
}

// 2. ServerCharacter receives RPC
[Rpc(SendTo.Server)]
public void ServerPlayActionRpc(ActionRequestData data)
{
    PlayAction(ref data);
}

// 3. ServerActionPlayer queues and executes
public void PlayAction(ref ActionRequestData data)
{
    var action = ActionFactory.CreateActionFromData(ref data);
    m_Queue.Add(action);
    StartAction();
}

// 4. Action runs on server
public override bool OnUpdate(ServerCharacter parent)
{
    // Deal damage, apply effects - SERVER ONLY
}

// 5. Server notifies clients
m_ClientCharacter.ClientPlayActionRpc(data);

// 6. ClientActionPlayer shows visuals on ALL clients
public void PlayAction(ref ActionRequestData data)
{
    // Animation, VFX - CLIENT ONLY
}
```

---

## Walkthrough 8: Connection Approval Deep Dive

### The Question

When a client tries to connect, what checks does the server perform to decide whether to approve or reject the connection?

### Starting Point

**File:** `Assets/Scripts/ConnectionManagement/ConnectionState/HostingState.cs`

Search for `ApprovalCheck`

### Key Findings

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CONNECTION APPROVAL CHECKS                         │
└──────────────────────────────────────────────────────────────────────┘

Client connects with ConnectionPayload
         │
         ▼
1. Payload size check (DOS protection)
         │
         ├─► Too big? → REJECT (ServerFull as generic error)
         │
         ▼
2. Parse ConnectionPayload (JSON)
         │
         ├─► Parse failed? → REJECT
         │
         ▼
3. IsDuplicateConnection(playerId)
         │
         ├─► Same playerId already connected? → REJECT (ServerFull)
         │
         ▼
4. Server capacity check
         │
         ├─► ConnectedClients >= MaxPlayers? → REJECT (ServerFull)
         │
         ▼
5. Build type compatibility
         │
         ├─► Debug client + Release server? → REJECT (IncompatibleBuildType)
         │
         ▼
6. APPROVED!
         │
         ├─► response.Approved = true
         ├─► response.CreatePlayerObject = true
         └─► SetupConnectingPlayerSessionData(...)
```

### Answer: The Code

```csharp
public override void ApprovalCheck(Request request, Response response)
{
    // 1. DOS check
    if (request.Payload.Length > MaxConnectPayload)
    {
        response.Approved = false;
        return;
    }
    
    // 2. Parse payload
    var payload = JsonUtility.FromJson<ConnectionPayload>(payloadString);
    
    // 3. Duplicate check
    if (IsDuplicateConnection(payload.playerId))
    {
        response.Approved = false;
        response.Reason = ConnectStatus.ServerFull;
        return;
    }
    
    // 4. Capacity check
    if (NetworkManager.ConnectedClients.Count >= m_ConnectionManager.MaxConnectedPlayers)
    {
        response.Approved = false;
        response.Reason = ConnectStatus.ServerFull;
        return;
    }
    
    // 5. Build check
    if (payload.isDebug != Debug.isDebugBuild)
    {
        response.Approved = false;
        response.Reason = ConnectStatus.IncompatibleBuildType;
        return;
    }
    
    // 6. APPROVE!
    response.Approved = true;
    response.CreatePlayerObject = true;
    
    SessionManager<SessionPlayerData>.Instance.SetupConnectingPlayerSessionData(
        request.ClientNetworkId, payload.playerId, playerData);
}
```

---

## Walkthrough 9: Tracing the Game State Flow

### The Question

How does the game transition from MainMenu → CharSelect → BossRoom → PostGame? Who initiates each transition?

### Starting Point

**File:** `Assets/Scripts/Gameplay/GameState/ServerBossRoomState.cs`

Also look at: `HostingState.cs`, `ServerCharSelectState.cs`

### Key Findings

```
┌──────────────────────────────────────────────────────────────────────┐
│                      SCENE TRANSITION MAP                             │
└──────────────────────────────────────────────────────────────────────┘

┌─────────────┐       ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  MainMenu   │ ────► │  CharSelect │ ────► │  BossRoom   │ ────► │  PostGame   │
└─────────────┘       └─────────────┘       └─────────────┘       └─────────────┘
       │                     │                     │                     │
       ▼                     ▼                     ▼                     ▼
   WHO TRIGGERS?          WHO?                  WHO?                  WHO?
   HostingState.Enter()   ServerCharSelectState ServerBossRoomState   User clicks
   after StartHost        after AllLockedIn    after Win/Loss        ReturnToMenu
```

### Answer: Transition Details

| From | To | Triggered By | File | Condition |
|------|----| -------------|------|-----------|
| MainMenu | CharSelect | HostingState | HostingState.cs | Host starts successfully |
| CharSelect | BossRoom | ServerCharSelectState | ServerCharSelectState.cs | All players locked in |
| BossRoom | PostGame | ServerBossRoomState | ServerBossRoomState.cs | Boss dies OR all players fainted |
| PostGame | MainMenu | User | ClientPostGameState.cs | User clicks button |

### Code Traces

```csharp
// Transition 1: MainMenu → CharSelect
// File: HostingState.cs
public override void Enter()
{
    await m_ConnectionMethod.SetupHostConnectionAsync();
    NetworkManager.Singleton.StartHost();
    SceneLoaderWrapper.Instance.LoadScene("CharSelect", true);  // ← HERE
}

// Transition 2: CharSelect → BossRoom
// File: ServerCharSelectState.cs
void OnClientChangedSeat(ulong clientId, int newSeatIdx, bool lockedIn)
{
    if (AllSeatsLockedIn())
    {
        SaveLobbyResults();
        SceneLoaderWrapper.Instance.LoadScene("BossRoom", true);  // ← HERE
    }
}

// Transition 3: BossRoom → PostGame
// File: ServerBossRoomState.cs
void OnLifeStateChanged(LifeStateChangedEventMessage msg)
{
    if (msg.CharacterType == CharacterTypeEnum.ImpBoss && msg.LifeState == LifeState.Dead)
    {
        BossDefeated();  // → LoadScene("PostGame")  // ← HERE
    }
    
    if (AllPlayersFainted())
    {
        GameOver(false);  // → LoadScene("PostGame")  // ← HERE
    }
}
```

---

## Walkthrough 10: Object Pooling Investigation

### The Question

How does Boss Room implement object pooling for networked objects? Trace how a projectile is spawned from the pool and returned.

### Starting Point

**File:** `Assets/Scripts/Infrastructure/NetworkObjectPool.cs`

### Key Findings

```
┌──────────────────────────────────────────────────────────────────────┐
│                    OBJECT POOL LIFECYCLE                              │
└──────────────────────────────────────────────────────────────────────┘

INITIALIZATION (on scene load)
         │
         ▼
PooledPrefabsList (configured in Inspector)
         │
         ├─► For each prefab: Create N instances
         ├─► SetActive(false)
         └─► Queue into m_PooledObjects[prefab]

═══════════════════════════════════════════════════════════════════════

SPAWN REQUEST (projectile fired)
         │
         ▼
GetNetworkObject(prefab, position, rotation)
         │
         ├─► Is queue empty?
         │   │
         │   ├─► NO: Dequeue, SetActive(true), return
         │   │
         │   └─► YES: Instantiate new (fallback)
         │
         ▼
Projectile flies, hits something
         │
         ▼
ReturnNetworkObject(obj, prefab)
         │
         ├─► SetActive(false)
         └─► Enqueue back into pool
```

### Answer: The Implementation

```csharp
public class NetworkObjectPool : NetworkBehaviour
{
    [SerializeField] List<PoolConfigObject> PooledPrefabsList;
    
    Dictionary<GameObject, Queue<NetworkObject>> m_PooledObjects = new();
    
    // Called at initialization
    void RegisterPrefab(GameObject prefab, int prewarmCount)
    {
        m_PooledObjects[prefab] = new Queue<NetworkObject>();
        
        for (int i = 0; i < prewarmCount; i++)
        {
            var obj = Instantiate(prefab).GetComponent<NetworkObject>();
            obj.gameObject.SetActive(false);
            m_PooledObjects[prefab].Enqueue(obj);
        }
    }
    
    // Get from pool
    public NetworkObject GetNetworkObject(GameObject prefab, Vector3 pos, Quaternion rot)
    {
        var queue = m_PooledObjects[prefab];
        
        if (queue.Count > 0)
        {
            var obj = queue.Dequeue();
            obj.transform.SetPositionAndRotation(pos, rot);
            obj.gameObject.SetActive(true);
            return obj;
        }
        
        // Fallback: create new
        return Instantiate(prefab, pos, rot).GetComponent<NetworkObject>();
    }
    
    // Return to pool
    public void ReturnNetworkObject(NetworkObject obj, GameObject prefab)
    {
        obj.gameObject.SetActive(false);
        m_PooledObjects[prefab].Enqueue(obj);
    }
}

// Usage in LaunchProjectileAction:
var projectile = NetworkObjectPool.Singleton.GetNetworkObject(
    Config.Spawns, 
    spawnPoint.position, 
    Quaternion.identity);
    
projectile.Spawn();  // Network spawn

// When projectile hits or expires:
projectile.Despawn();
NetworkObjectPool.Singleton.ReturnNetworkObject(projectile, Config.Spawns);
```

---

## Exercises with Full Answers

### Exercise A: Network Variable Debugging

**Task:** Find all `NetworkVariable` declarations in ServerCharacter.

**Answer:**
```csharp
// In ServerCharacter.cs and related components:
public NetworkVariable<MovementStatus> MovementStatus { get; }
public NetworkVariable<ulong> HeldNetworkObject { get; }
public NetworkVariable<bool> IsStealthy { get; }
public NetworkVariable<ulong> TargetId { get; }

// In NetworkHealthState.cs:
public NetworkVariable<int> HitPoints { get; }

// In NetworkLifeState.cs:
public NetworkVariable<LifeState> LifeState { get; }
```

**Where modified (server):** 
- `ServerCharacter.ReceiveHP()` modifies HitPoints
- `ServerCharacter.SetLifeState()` modifies LifeState
- `Action` subclasses modify other state

**Where callbacks (client):**
- `ClientCharacter.OnNetworkSpawn()` subscribes to `.OnValueChanged`

---

### Exercise B: Scene Loading

**Task:** Trace what happens when host moves from CharSelect to BossRoom.

**Answer:**
```
1. ServerCharSelectState.OnClientChangedSeat() detects all locked in
         │
         ▼
2. SceneLoaderWrapper.Instance.LoadScene("BossRoom", useNetworkSceneManager: true)
         │
         ▼
3. NetworkSceneManager synchronizes load across all clients
         │
         ▼
4. All clients load BossRoom scene
         │
         ▼
5. ServerBossRoomState.OnNetworkSpawn() called
         │
         ▼
6. ServerBossRoomState subscribes to LifeStateChangedEventMessage
         │
         ▼
7. SceneLoaderWrapper.OnSceneEvent() fires OnSceneLoaded
         │
         ▼
8. ServerBossRoomState.OnLoadEventCompleted() spawns players
```

---

### Exercise C: Boss AI

**Task:** Find the boss AI logic.

**Answer:**
- **Boss prefab:** Look in `Assets/Prefabs/Characters/Enemies/`
- **AI logic:** `Assets/Scripts/Gameplay/GameplayObjects/Character/AI/`
- **AIBrain** class handles decision-making
- The boss uses the same `ServerActionPlayer` as players
- AI decides which action to play based on target distance, health, etc.

```csharp
// In AIBrain.cs:
public void Update()
{
    if (m_ServerCharacter.LifeState != LifeState.Alive) return;
    
    var target = FindTarget();
    var action = SelectAction(target);
    
    m_ServerActionPlayer.PlayAction(ref action);
}
```

---

### Exercise D: Projectile Lifecycle (Full Answer)

**Task:** Trace how projectiles work.

**Answer:**

```
1. LaunchProjectileAction.OnStart()
         │
         ├─► Get projectile from NetworkObjectPool
         ├─► Set position, rotation
         └─► Spawn as NetworkObject
         
2. Projectile.Update() (on server)
         │
         ├─► Move forward
         ├─► Check for collision (Physics.OverlapSphere or Raycast)
         └─► On hit: Apply damage, Despawn
         
3. Projectile.OnDespawn()
         │
         └─► Return to pool via NetworkObjectPool.ReturnNetworkObject()
```

**Key files:**
- `LaunchProjectileAction.cs` - Creates projectile
- `PhysicsProjectile.cs` or similar - Movement and hit detection
- `NetworkObjectPool.cs` - Pooling

---

## Self-Assessment Checklist

After completing these walkthroughs, you should be able to:

### Core Understanding

- [x] Explain how a player's input becomes an action on the server
- [x] Trace data flow from server to all clients via NetworkVariables
- [x] Describe the state machine pattern used for connections
- [x] Explain why client anticipation exists and how it works
- [x] Trace connection approval checks
- [x] Understand scene transition triggers

### Code Navigation

- [x] Find where any Action is configured (ScriptableObject location)
- [x] Locate the server-side logic for any gameplay feature
- [x] Identify which component is responsible for a given behavior
- [x] Trace a message from publisher to all subscribers
- [x] Find the object pool implementation

### Patterns Recognition

- [x] Identify State Machine pattern in code
- [x] Recognize Factory pattern usage
- [x] Find examples of Object Pooling
- [x] Locate Strategy pattern implementations

### Modification Skills

- [x] Know what files to modify to add a new action
- [x] Understand how to add a new connection state
- [x] Know how to create a new message channel
- [x] Understand how to add a new character class

---

> **Congratulations!** You've completed the guided walkthroughs. Return to the [README](./README.md) for the full learning path.

