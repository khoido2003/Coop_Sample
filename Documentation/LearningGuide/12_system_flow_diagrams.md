# 12: System Flow Diagrams

> **Purpose:** Visual reference showing how the major systems work. Use these diagrams when you need to quickly understand a flow or when debugging.

---

## Table of Contents

1. [Game Startup Sequence](#game-startup-sequence)
2. [Host Connection Flow](#host-connection-flow)
3. [Client Connection Flow](#client-connection-flow)
4. [Character Selection Sync](#character-selection-sync)
5. [Combat Action Flow](#combat-action-flow)
6. [Health Damage Flow](#health-damage-flow)
7. [Reconnection Flow](#reconnection-flow)

---

## Game Startup Sequence

What happens from Unity launch to the main menu appearing.

```mermaid
sequenceDiagram
    participant Unity
    participant AppController as ApplicationController
    participant Container as DI Container
    participant Services as Unity Services
    participant SceneLoader

    Unity->>AppController: Awake()
    AppController->>AppController: Configure() - Build DI Container
    
    Note over AppController,Container: Register all services:<br/>LocalSession, ProfileManager,<br/>ConnectionManager, MessageChannels
    
    AppController->>Container: Build()
    Container->>Container: Resolve dependencies
    
    AppController->>AppController: Start()
    AppController->>Services: InitializeUnityServicesAsync()
    Services-->>AppController: Services ready
    
    AppController->>Services: AuthenticateAsync()
    Services-->>AppController: Authenticated (or anonymous)
    
    AppController->>SceneLoader: LoadScene("MainMenu")
    SceneLoader-->>Unity: Main Menu Loaded
```

### Text Diagram

```
Unity Launch
     │
     ▼
┌─────────────────────────────────────┐
│  ApplicationController.Awake()      │
│  └─ DontDestroyOnLoad               │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  Configure() - DI Container Setup   │
│  ├─ Register LocalSession           │
│  ├─ Register ProfileManager         │
│  ├─ Register ConnectionManager      │
│  ├─ Register MessageChannels        │
│  └─ Register UnityServices Facades  │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  Start()                            │
│  ├─ Initialize Unity Services       │
│  ├─ Authenticate (anonymous OK)     │
│  └─ Load Main Menu Scene            │
└─────────────────┬───────────────────┘
                  │
                  ▼
         [ Main Menu Ready ]
```

---

## Host Connection Flow

What happens when a player clicks "Host Game."

```mermaid
stateDiagram-v2
    [*] --> Offline: Game starts
    
    Offline --> StartingHost: User clicks Host
    
    state StartingHost {
        [*] --> SetupRelay: Using Unity Relay
        SetupRelay --> CreateSession: Relay allocated
        CreateSession --> StartNetworkManager: Session created
        StartNetworkManager --> [*]: NetworkManager started
    }
    
    StartingHost --> Hosting: OnServerStarted
    StartingHost --> Offline: Failed
    
    Hosting --> LoadCharSelect: Enter()
    LoadCharSelect --> WaitingForPlayers: Scene loaded
    
    Hosting --> Offline: Shutdown requested
```

### Detailed Steps

```
User clicks "Host"
        │
        ▼
┌──────────────────────────────────────────────┐
│  OfflineState.StartHostSession()             │
│  ├─ Create ConnectionMethodRelay             │
│  └─ ChangeState(StartingHost)                │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  StartingHostState.Enter()                   │
│  ├─ SetupHostConnection()                    │
│  │   ├─ Set connection payload               │
│  │   └─ Configure transport (Relay or IP)   │
│  └─ NetworkManager.StartHost()               │
└──────────────────────┬───────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
    Success                        Failure
        │                             │
        ▼                             ▼
┌───────────────────┐        ┌───────────────────┐
│ OnServerStarted() │        │ OnTransportFailure│
│ ChangeState       │        │ ChangeState       │
│   (Hosting)       │        │   (Offline)       │
└────────┬──────────┘        └───────────────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│  HostingState.Enter()                        │
│  ├─ LoadScene("CharSelect", network: true)   │
│  └─ BeginTracking() if using Session         │
└──────────────────────────────────────────────┘
```

---

## Client Connection Flow

What happens when a player joins a game.

```mermaid
sequenceDiagram
    participant Client
    participant OfflineState
    participant ClientConnecting
    participant Server as Server (Hosting)
    participant ClientConnected

    Client->>OfflineState: StartClientSession(name)
    OfflineState->>ClientConnecting: ChangeState
    
    ClientConnecting->>ClientConnecting: SetupClientConnection()
    Note over ClientConnecting: Set payload<br/>(playerId, playerName, isDebug)
    
    ClientConnecting->>Server: NetworkManager.StartClient()
    
    Server->>Server: ApprovalCheck()
    
    alt Approved
        Server-->>ClientConnecting: OnClientConnected
        ClientConnecting->>ClientConnected: ChangeState
        Note over ClientConnected: Sync to CharSelect scene
    else Rejected
        Server-->>ClientConnecting: OnClientDisconnect(reason)
        ClientConnecting->>OfflineState: ChangeState
        Note over OfflineState: Show error message
    end
```

### Approval Checks

```
Client connection request arrives
              │
              ▼
┌───────────────────────────────────────────────┐
│  HostingState.ApprovalCheck()                 │
└────────────────────┬──────────────────────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │ Payload > 1024 bytes?  │
        └────────┬───────────────┘
                 │ No
                 ▼
        ┌────────────────────────┐
        │ Server full?           │
        │ (count >= MaxPlayers)  │
        └────────┬───────────────┘
                 │ No
                 ▼
        ┌────────────────────────┐
        │ Build type mismatch?   │
        │ (Debug vs Release)     │
        └────────┬───────────────┘
                 │ No
                 ▼
        ┌────────────────────────┐
        │ Duplicate connection?  │
        │ (same playerId)        │
        └────────┬───────────────┘
                 │ No
                 ▼
        ┌────────────────────────┐
        │       APPROVED         │
        │  CreatePlayerObject    │
        └────────────────────────┘
```

---

## Character Selection Sync

How character selection stays synchronized across all players.

```mermaid
sequenceDiagram
    participant Player1 as Player 1 (Client)
    participant Server
    participant Player2 as Player 2 (Client)
    participant NetworkList

    Note over Server,NetworkList: NetworkList<LobbyPlayerState><br/>syncs to all clients

    Player1->>Server: SelectSeatRpc(seatIdx)
    Server->>Server: Validate seat available
    Server->>NetworkList: Update player's seat
    NetworkList-->>Player1: OnListChanged
    NetworkList-->>Player2: OnListChanged
    
    Player1->>Player1: Update UI
    Player2->>Player2: Update UI
    
    Note over Player1,Player2: Both see same selection

    Player1->>Server: LockedInRpc()
    Server->>NetworkList: Set player.IsLockedIn = true
    NetworkList-->>Player1: OnListChanged
    NetworkList-->>Player2: OnListChanged
    
    Note over Server: When all locked in
    Server->>Server: Load BossRoom scene
```

### Key Insight: Client Never Modifies Directly

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ WRONG: Client modifies local state, hopes it syncs         │
│  ✅ RIGHT: Client sends RPC, server validates, NetworkList     │
│            automatically syncs to everyone                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Client clicks character
        │
        ▼
┌─────────────────────┐
│ Send RPC to server  │ ◄── Client does NOT update local state yet
└─────────┬───────────┘
          │
          ▼ (travels over network)
          
┌─────────────────────┐
│ Server validates    │
│ ├─ Is seat taken?   │
│ ├─ Is player valid? │
│ └─ etc.             │
└─────────┬───────────┘
          │ If valid
          ▼
┌─────────────────────┐
│ Server updates      │
│ NetworkList         │
└─────────┬───────────┘
          │
          ▼ (auto-syncs)
          
┌─────────────────────┐
│ ALL clients receive │ ◄── NOW client updates local state
│ OnListChanged       │
└─────────────────────┘
```

---

## Combat Action Flow

Complete flow from player input to damage applied.

```mermaid
sequenceDiagram
    participant Input
    participant ClientChar as ClientCharacter
    participant Server as ServerCharacter
    participant ActionPlayer as ServerActionPlayer
    participant Action as MeleeAction
    participant Target

    Input->>ClientChar: Ability button pressed
    
    Note over ClientChar: Anticipation!
    ClientChar->>ClientChar: AnticipateActionClient()
    ClientChar->>ClientChar: Play attack animation
    
    ClientChar->>Server: PlayActionRpc(actionData)
    
    Server->>ActionPlayer: PlayAction(actionData)
    ActionPlayer->>Action: CreateActionFromData()
    ActionPlayer->>ActionPlayer: Add to queue
    
    alt Queue was empty
        ActionPlayer->>Action: OnStart()
        Action->>Action: DetectFoe() - provisional
        Action->>Server: Play animation (server)
        Action->>ClientChar: ClientPlayActionRpc (to all)
    end
    
    loop Every frame
        ActionPlayer->>Action: OnUpdate()
        alt TimeRunning >= ExecTimeSeconds
            Action->>Action: DetectFoe() - real check
            Action->>Target: ReceiveHitPoints(-damage)
        end
    end
    
    Note over Action: Duration expires
    ActionPlayer->>Action: End()
    Action-->>ActionPlayer: ChainIntoNewAction?
```

### Timeline View

```
T+0ms:     Player presses attack button
           │
           ├─► Client: AnticipateActionClient()
           │   └─ Animation starts IMMEDIATELY (feels responsive!)
           │
           └─► RPC sent to server
           
T+50ms:    Server receives RPC
           │
           ├─► ServerActionPlayer.PlayAction()
           ├─► MeleeAction.OnStart()
           │   ├─ DetectFoe() → provisional target
           │   ├─ Snap to face target
           │   └─ Send ClientPlayActionRpc to all clients
           │
           └─► Action added to queue, starts running

T+100ms:   Other clients receive ClientPlayActionRpc
           └─► OnStartClient() → spawn VFX on target

T+400ms:   TimeRunning >= ExecTimeSeconds (e.g., 0.4s)
           │
           └─► MeleeAction.OnUpdate()
               ├─ DetectFoe() → real hit check
               └─ Target.ReceiveHitPoints(-25)
                  │
                  └─► NetworkVariable<HitPoints> syncs to all clients

T+450ms:   All clients see health bar update

T+1000ms:  TimeRunning >= DurationSeconds (e.g., 1.0s)
           │
           └─► Action ends
               ├─ MeleeAction.End()
               ├─ ServerActionPlayer.AdvanceQueue()
               └─ Next action starts (if any)
```

---

## Health Damage Flow

From damage received to UI update on all clients.

```mermaid
flowchart TB
    subgraph Server
        A[Damage Source] -->|ReceiveHitPoints| B[ServerCharacter]
        B --> C{Apply buffs}
        C -->|Modified damage| D[Update health]
        D --> E[NetworkVariable.HitPoints]
    end
    
    subgraph AllClients["All Clients"]
        E -->|Auto-sync| F[OnHealthChanged callback]
        F --> G[ClientCharacter]
        G --> H[Update health bar]
        G --> I[Play hit VFX]
        G --> J[Flash UI]
    end
```

### Code Flow

```
ServerCharacter.ReceiveHP(source, -damage)
        │
        ▼
┌─────────────────────────────────────────────────┐
│  1. Notify active actions (might modify damage) │
│     serverActionPlayer.OnGameplayActivity(     │
│         GameplayActivity.AttackedByEnemy)      │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│  2. Apply buff modifiers                        │
│     float modifier = serverActionPlayer        │
│         .GetBuffedValue(PercentDamageReceived) │
│     finalDamage = damage * modifier            │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│  3. Apply damage to NetworkVariable            │
│     m_NetworkHealthState.HitPoints.Value       │
│         += finalDamage                         │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│  4. Check death                                │
│     if (HitPoints <= 0)                        │
│         m_NetworkLifeState.LifeState.Value     │
│             = LifeStateType.Fainted            │
└─────────────────────┬───────────────────────────┘
                      │
                      │ (NetworkVariable syncs automatically)
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│  All Clients: OnValueChanged callback          │
│  ├─ ClientCharacter receives callback          │
│  ├─ Update health bar UI                       │
│  ├─ Play damage VFX                            │
│  └─ Trigger hit react animation                │
└─────────────────────────────────────────────────┘
```

---

## Reconnection Flow

What happens when a client loses connection.

```mermaid
stateDiagram-v2
    [*] --> ClientConnected: Playing game
    
    ClientConnected --> ClientReconnecting: Connection lost
    
    state ClientReconnecting {
        [*] --> Wait1s: First attempt
        Wait1s --> Attempt1: Delay passed
        Attempt1 --> Wait5s: Failed
        Wait5s --> Attempt2: Delay passed
        Attempt2 --> Wait5s2: Failed
        Wait5s2 --> Attempt3: Delay passed
        Attempt3 --> [*]: Success or max attempts
    }
    
    ClientReconnecting --> ClientConnected: Reconnected!
    ClientReconnecting --> Offline: Max attempts reached
    ClientReconnecting --> Offline: Session ended
```

### Decision Logic

```
Connection Lost
      │
      ▼
┌───────────────────────────────────────┐
│  Enter ClientReconnectingState        │
│  m_NbAttempts = 0                     │
│  Start ReconnectCoroutine()           │
└──────────────────┬────────────────────┘
                   │
                   ▼
         ┌─────────────────┐
         │ First attempt?  │
         └────────┬────────┘
                  │
        ┌─────────┴─────────┐
       Yes                  No
        │                   │
        ▼                   ▼
   Wait 1 second      Wait 5 seconds
        │                   │
        └─────────┬─────────┘
                  │
                  ▼
┌───────────────────────────────────────┐
│  SetupClientReconnectionAsync()       │
│  (differs by connection method)       │
└──────────────────┬────────────────────┘
                   │
        ┌──────────┴──────────┐
     Success               Failed
        │                     │
        ▼                     ▼
┌─────────────────┐  ┌─────────────────────┐
│ ConnectClient() │  │ attempts < max?     │
└────────┬────────┘  └──────────┬──────────┘
         │                      │
         │             ┌────────┴────────┐
         │            Yes               No
         │             │                 │
         │             ▼                 ▼
         │        Try again        Go to Offline
         │             │
         │             └─────────────────┐
         │                               │
         ▼                               │
┌─────────────────┐                      │
│ OnClient        │                      │
│ Connected?      │                      │
└────────┬────────┘                      │
         │                               │
    ┌────┴────┐                          │
   Yes        No                         │
    │         │                          │
    ▼         └──────────────────────────┘
Go to ClientConnected
```

---

## Quick Reference Card

### State Transitions

| From | Event | To |
|------|-------|-----|
| Offline | StartHost | StartingHost |
| Offline | StartClient | ClientConnecting |
| StartingHost | Server started | Hosting |
| StartingHost | Failed | Offline |
| Hosting | Shutdown | Offline |
| ClientConnecting | Connected | ClientConnected |
| ClientConnecting | Failed | Offline |
| ClientConnected | Disconnect | ClientReconnecting |
| ClientConnected | User quit | Offline |
| ClientReconnecting | Connected | ClientConnected |
| ClientReconnecting | Max attempts | Offline |

### Action Lifecycle

| Phase | Server Method | Client Method |
|-------|--------------|---------------|
| Anticipation | — | AnticipateActionClient |
| Start | OnStart | OnStartClient |
| Update | OnUpdate | OnUpdateClient |
| End | End | EndClient |
| Cancel | Cancel | CancelClient |

---

> **Next:** [13_code_reading_walkthroughs.md](./13_code_reading_walkthroughs.md) — Guided exercises to read the code yourself
