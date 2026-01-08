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

## Additional Exercises

### Exercise A: Network Variable Debugging

1. Find all `NetworkVariable` declarations in ServerCharacter
2. For each one, find where it's modified (server-side)
3. Find where each has a value-changed callback (client-side)

### Exercise B: Scene Loading

1. Find `SceneLoaderWrapper.cs`
2. Trace what happens when the host moves from CharSelect to BossRoom
3. How do clients know to load the scene?

### Exercise C: Boss AI

1. Find the boss enemy prefab
2. Locate its AI/behavior logic
3. How does it decide which player to target?

### Exercise D: Projectile Lifecycle

1. Find `LaunchProjectileAction`
2. Trace how the projectile prefab is spawned
3. How is hit detection performed?
4. When is the projectile despawned?

---

> **Congratulations!** You've completed the guided walkthroughs. Return to the [README](./README.md) for the full learning path.
