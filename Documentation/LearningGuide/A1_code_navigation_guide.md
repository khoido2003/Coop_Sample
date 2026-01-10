# A1: Code Navigation Quick Reference

> **Purpose:** Answer "I want to understand X, where should I look?" in seconds.

---

## Quick Lookup Table

| Feature | Start Here | Also Check |
|---------|------------|------------|
| **Game Startup** | `ApplicationController.cs` | `ConnectionManager.cs` |
| **Main Menu UI** | `MainMenuUI.cs` | `ClientMainMenuState.cs` |
| **Host/Join Flow** | `ConnectionManager.cs` | `HostingState.cs`, `ClientConnectingState.cs` |
| **Character Select** | `ServerCharSelectState.cs` | `ClientCharSelectState.cs`, `CharSelectData.cs` |
| **Player Movement** | `ServerCharacterMovement.cs` | `ClientInputSender.cs` |
| **Combat Actions** | `ServerActionPlayer.cs` | `Action.cs`, `ActionConfig.cs` |
| **Specific Abilities** | `Assets/Scripts/Gameplay/Action/ConcreteActions/` | `ActionConfig` ScriptableObjects |
| **Health/Damage** | `ServerCharacter.ReceiveHP()` | `NetworkHealthState.cs`, `DamageReceiver.cs` |
| **AI/Enemies** | `AIBrain.cs` | `AIBrainState*.cs` files |
| **Player Spawn** | `ServerBossRoomState.SpawnPlayer()` | `PersistentPlayer.cs` |
| **Win/Lose** | `ServerBossRoomState.CheckForGameOver()` | `PersistentGameState.cs` |
| **Networking Basics** | `ConnectionManager.cs` | `ConnectionState.cs` (base class) |
| **RPCs** | `ServerCharacter.cs` (search `[Rpc(`) | `ClientCharacter.cs` |
| **NetworkVariables** | `ServerCharacter.cs` | `NetworkHealthState.cs`, `NetworkLifeState.cs` |
| **PubSub Messaging** | `MessageChannel.cs` | `NetworkedMessageChannel.cs` |
| **Object Pooling** | `NetworkObjectPool.cs` | `Projectile.cs` (example usage) |
| **Session/Lobby** | `SessionManager.cs` | `LocalSession.cs`, `HostingState.ApprovalCheck()` |

---

## Directory Map

```
Assets/Scripts/
├── ApplicationLifecycle/        ← Game startup, quit handling
│   ├── ApplicationController.cs   [ENTRY POINT]
│   └── Messages/                   [Quit messages]
│
├── ConnectionManagement/        ← Host/Join, connection states
│   ├── ConnectionManager.cs        [State machine owner]
│   └── ConnectionState/            [Individual states]
│       ├── OfflineState.cs
│       ├── HostingState.cs
│       ├── ClientConnectingState.cs
│       └── ...
│
├── Gameplay/
│   ├── Action/                  ← Combat system
│   │   ├── Action.cs               [Base action class]
│   │   ├── ActionConfig.cs         [ScriptableObject config]
│   │   ├── ActionPlayers/          [Queue management]
│   │   │   ├── ServerActionPlayer.cs
│   │   │   └── ClientActionPlayer.cs
│   │   ├── ConcreteActions/        [All abilities]
│   │   │   ├── MeleeAction.cs
│   │   │   ├── LaunchProjectileAction.cs
│   │   │   └── ... (48 actions)
│   │   └── Input/                  [User input handling]
│   │       └── ClientInputSender.cs
│   │
│   ├── Configuration/           ← ScriptableObject data
│   │   ├── CharacterClass.cs
│   │   ├── Avatar.cs
│   │   └── GameDataSource.cs       [Registry of all data]
│   │
│   ├── GameplayObjects/         ← Game entities
│   │   ├── Character/              [Player/NPC characters]
│   │   │   ├── ServerCharacter.cs
│   │   │   ├── ClientCharacter.cs
│   │   │   ├── ServerCharacterMovement.cs
│   │   │   └── AI/                 [Enemy AI]
│   │   ├── PersistentPlayer.cs     [Survives scene loads]
│   │   └── NetworkHealthState.cs
│   │
│   ├── GameState/               ← Scene-specific logic
│   │   ├── GameStateBehaviour.cs   [Base class]
│   │   ├── ServerBossRoomState.cs
│   │   ├── ServerCharSelectState.cs
│   │   └── Client*State.cs files
│   │
│   └── UI/                      ← All UI components
│
├── Infrastructure/              ← Reusable systems
│   ├── PubSub/                     [Message channels]
│   │   ├── MessageChannel.cs
│   │   └── NetworkedMessageChannel.cs
│   └── NetworkObjectPool.cs
│
└── UnityServices/               ← Cloud services integration
    ├── Sessions/
    └── Auth/
```

---

## Common Search Patterns

### Find all RPCs
```
Search: [Rpc(SendTo.
Files: *.cs
```

### Find all NetworkVariables
```
Search: NetworkVariable<
Files: *.cs
```

### Find where damage is dealt
```
Search: ReceiveHP
Files: *.cs
```

### Find where actions are played
```
Search: PlayAction
Files: ServerActionPlayer.cs, ServerCharacter.cs
```

### Find state machine transitions
```
Search: ChangeState
Files: ConnectionManager.cs
```

### Find scene loads
```
Search: LoadScene
Files: *.cs
```

---

## Key Class Relationships

```
ApplicationController
    └─► ConnectionManager
           └─► ConnectionState (current)
                  │
    ┌──────────────┴──────────────┐
    ▼                             ▼
HostingState                 ClientConnectedState
    │                             │
    ▼                             ▼
ServerBossRoomState          (client-side state)
    │
    ▼
ServerCharacter ◄──────────► ClientCharacter
    │                             │
    ▼                             ▼
ServerActionPlayer           ClientActionPlayer
    │                             │
    ▼                             ▼
Action (server logic)        Action (client visuals)
```

---

## File Naming Conventions

| Prefix/Suffix | Meaning |
|---------------|---------|
| `Server*` | Runs only on server |
| `Client*` | Runs only on clients |
| `Network*` | Contains NetworkVariables |
| `*State` | State in a state machine |
| `*Action` | Combat ability |
| `*Config` | ScriptableObject configuration |
| `*Behaviour` | Base class for scene logic |
| `*Manager` | Orchestrates subsystem |
| `*Facade` | Simplifies complex subsystem |

---

## "I want to add..." Quick Paths

### Add a new ability
1. Create `Assets/Scripts/Gameplay/Action/ConcreteActions/MyAction.cs`
2. Create `ActionConfig` ScriptableObject in `Assets/GameData/Actions/`
3. Register in `GameDataSource`

### Add a new character class
1. Create `CharacterClass` ScriptableObject
2. Create character prefab with `ServerCharacter` + `ClientCharacter`
3. Add to `GameDataSource.CharacterDataList`

### Add a new UI element
1. Create component in `Assets/Scripts/Gameplay/UI/`
2. Subscribe to relevant `ISubscriber<>` for events

### Add a new enemy type
1. Duplicate existing enemy prefab
2. Configure `CharacterClass` ScriptableObject
3. Add `AIBrain` configuration

### Add a new game mode
1. Create new scene
2. Create `Server*State.cs` and `Client*State.cs`
3. Add to `GameState` enum
4. Wire up scene transitions

---

> **Tip:** Use Ctrl+Shift+F in your IDE to search across all files. The patterns above will help you find what you need.
