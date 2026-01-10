# 19: Game Flow Deep Dive

> **Purpose:** Understand how Boss Room flows from application start to gameplay, and how scenes and states are managed in a networked context.

---

## Table of Contents

1. [The Entry Point Question](#the-entry-point-question)
2. [Application Startup Sequence](#application-startup-sequence)
3. [Scene Flow Overview](#scene-flow-overview)
4. [GameStateBehaviour Pattern](#gamestatebehaviour-pattern)
5. [Connection State Machine](#connection-state-machine)
6. [Scene Loading in Multiplayer](#scene-loading-in-multiplayer)
7. [Server vs Client States](#server-vs-client-states)
8. [Key Takeaways](#key-takeaways)

---

## The Entry Point Question

> **"Where does the game actually start? What's the first code that runs?"**

Unlike simple Unity projects where you might have a `GameManager.Start()`, Boss Room uses a **Dependency Injection container** as its entry point.

### The Bootstrap Scene

1. Open Unity and look at **Build Settings**
2. The first scene is `Startup`
3. This scene contains `ApplicationController` GameObject

---

## Application Startup Sequence

**File:** [ApplicationController.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ApplicationLifecycle/ApplicationController.cs)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         STARTUP SEQUENCE                             │
└─────────────────────────────────────────────────────────────────────┘

Unity loads Startup scene
         │
         ▼
ApplicationController (LifetimeScope)
         │
         ├─► Configure() - Register all dependencies
         │       ├─ UpdateRunner
         │       ├─ ConnectionManager
         │       ├─ NetworkManager
         │       ├─ LocalSession / LocalSessionUser
         │       ├─ ProfileManager
         │       ├─ Message Channels (PubSub)
         │       └─ AuthenticationServiceFacade
         │
         └─► Start() - Bootstrap the game
                 ├─ Container.Resolve<LocalSession>()
                 ├─ Subscribe to quit messages
                 ├─ DontDestroyOnLoad(this)
                 ├─ Application.targetFrameRate = 120
                 └─ SceneManager.LoadScene("MainMenu")  ◄─── FIRST SCENE LOAD
```

### The Configure Method (Dependency Registration)

```csharp
protected override void Configure(IContainerBuilder builder)
{
    base.Configure(builder);
    
    // Core components (referenced from Inspector)
    builder.RegisterComponent(m_UpdateRunner);
    builder.RegisterComponent(m_ConnectionManager);
    builder.RegisterComponent(m_NetworkManager);

    // Singletons that persist across scenes
    builder.Register<LocalSessionUser>(Lifetime.Singleton);
    builder.Register<LocalSession>(Lifetime.Singleton);
    builder.Register<ProfileManager>(Lifetime.Singleton);
    builder.Register<PersistentGameState>(Lifetime.Singleton);

    // Message channels for event-driven communication
    builder.RegisterInstance(new MessageChannel<QuitApplicationMessage>())
           .AsImplementedInterfaces();
    builder.RegisterInstance(new MessageChannel<ConnectStatus>())
           .AsImplementedInterfaces();
    
    // Networked message channels (sync across network)
    builder.RegisterComponent(new NetworkedMessageChannel<LifeStateChangedEventMessage>())
           .AsImplementedInterfaces();
}
```

> **Key Insight:** Everything registered here is available via `[Inject]` throughout the application lifetime.

### The Start Method (Bootstrap)

```csharp
private void Start()
{
    // Resolve dependencies from container
    m_LocalSession = Container.Resolve<LocalSession>();
    m_MultiplayerServicesFacade = Container.Resolve<MultiplayerServicesFacade>();

    // Subscribe to quit events
    var quitApplicationSub = Container.Resolve<ISubscriber<QuitApplicationMessage>>();
    subHandles.Add(quitApplicationSub.Subscribe(QuitGame));
    
    // Keep these alive forever
    DontDestroyOnLoad(gameObject);
    DontDestroyOnLoad(m_UpdateRunner.gameObject);
    
    // Lock to 120 FPS
    Application.targetFrameRate = 120;
    
    // GO TO MAIN MENU!
    SceneManager.LoadScene("MainMenu");
}
```

---

## Scene Flow Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         COMPLETE GAME FLOW                           │
└─────────────────────────────────────────────────────────────────────┘

    ┌──────────┐
    │ Startup  │  (Bootstrap, DI container setup)
    └────┬─────┘
         │ SceneManager.LoadScene("MainMenu")
         ▼
    ┌──────────┐
    │ MainMenu │  (UI for Host/Join/Direct IP)
    └────┬─────┘
         │ User clicks "Host" or "Join"
         │ ConnectionManager.StartHostSession() or StartClientSession()
         ▼
    ┌──────────────────────────────────────────────────────────────┐
    │              CONNECTION STATE MACHINE TAKES OVER              │
    │                                                              │
    │  OfflineState → StartingHostState → HostingState            │
    │       or                                                     │
    │  OfflineState → ClientConnectingState → ClientConnectedState │
    └──────────────────────────────────────────────────────────────┘
         │ HostingState.Enter() calls:
         │ SceneLoaderWrapper.Instance.LoadScene("CharSelect", useNetworkSceneManager: true)
         ▼
    ┌────────────┐
    │ CharSelect │  (All players pick characters)
    └────┬───────┘
         │ All players ready
         │ ServerCharSelectState triggers scene load
         ▼
    ┌──────────┐
    │ BossRoom │  (Actual gameplay)
    └────┬─────┘
         │ Boss dies or all players die
         │ ServerBossRoomState triggers scene load
         ▼
    ┌──────────┐
    │ PostGame │  (Win/Lose screen)
    └────┬─────┘
         │ Return to menu
         ▼
    ┌──────────┐
    │ MainMenu │
    └──────────┘
```

---

## GameStateBehaviour Pattern

**File:** [GameStateBehaviour.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameState/GameStateBehaviour.cs)

Each scene has a `GameStateBehaviour` that manages scene-specific logic.

```csharp
public abstract class GameStateBehaviour : MonoBehaviour
{
    // Which game state this behaviour represents
    public abstract GameState ActiveState { get; }
    
    // Singleton pattern for current state
    public static GameStateBehaviour ActiveInstance { get; private set; }
    
    protected virtual void Awake()
    {
        if (ActiveInstance != null && ActiveInstance != this)
        {
            // Only one state active at a time
            Destroy(gameObject);
            return;
        }
        ActiveInstance = this;
    }
}

public enum GameState
{
    MainMenu,
    CharSelect,
    BossRoom,
    PostGame
}
```

### Server vs Client States

Each scene has SEPARATE behaviours for server and client:

| Scene | Server | Client |
|-------|--------|--------|
| MainMenu | - | ClientMainMenuState |
| CharSelect | ServerCharSelectState | ClientCharSelectState |
| BossRoom | ServerBossRoomState | (in scene) |
| PostGame | ServerPostGameState | (in scene) |

**Why separate?**
- Server handles game logic, spawning, win conditions
- Client handles UI, local input, visual feedback
- Same pattern as ServerCharacter/ClientCharacter!

---

## Connection State Machine

**File:** [ConnectionManager.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionManager.cs)

The connection is managed as a **state machine** with 6 states:

```
┌───────────────────────────────────────────────────────────────────┐
│                    CONNECTION STATE MACHINE                        │
└───────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │  OfflineState   │ ◄─── Initial state
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
   ┌───────────────┐  ┌─────────────┐  ┌─────────────────┐
   │StartingHost   │  │ClientConnec-│  │ (IP Connect)    │
   │    State      │  │ tingState   │  │                 │
   └───────┬───────┘  └──────┬──────┘  └────────┬────────┘
           │                 │                  │
           ▼                 ▼                  │
   ┌───────────────┐  ┌─────────────┐          │
   │ HostingState  │  │ClientConnec-│ ◄────────┘
   │               │  │ tedState    │
   └───────────────┘  └──────┬──────┘
                             │
                             ▼ (disconnect)
                    ┌─────────────────┐
                    │ClientReconnect- │
                    │   ingState      │
                    └─────────────────┘
```

### How States Work

```csharp
// Base class for all connection states
abstract class ConnectionState
{
    [Inject] protected ConnectionManager m_ConnectionManager;
    [Inject] protected IPublisher<ConnectStatus> m_ConnectStatusPublisher;

    public abstract void Enter();
    public abstract void Exit();
    
    // Override these to handle events
    public virtual void OnClientConnected(ulong clientId) { }
    public virtual void OnClientDisconnect(ulong clientId) { }
    public virtual void OnServerStarted() { }
    public virtual void OnUserRequestedShutdown() { }
    public virtual void ApprovalCheck(...) { }
}
```

### HostingState Example

**File:** [HostingState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ConnectionManagement/ConnectionState/HostingState.cs)

```csharp
class HostingState : OnlineState
{
    public override void Enter()
    {
        // Immediately load CharSelect when hosting starts
        SceneLoaderWrapper.Instance.LoadScene("CharSelect", useNetworkSceneManager: true);
        
        if (m_MultiplayerServicesFacade.CurrentUnitySession != null)
        {
            m_MultiplayerServicesFacade.BeginTracking();
        }
    }

    public override void ApprovalCheck(
        NetworkManager.ConnectionApprovalRequest request,
        NetworkManager.ConnectionApprovalResponse response)
    {
        // Validate connection payload
        var payload = JsonUtility.FromJson<ConnectionPayload>(connectionData);
        
        // Check server full
        if (m_ConnectionManager.NetworkManager.ConnectedClientsIds.Count >= MaxConnectedPlayers)
        {
            response.Approved = false;
            response.Reason = JsonUtility.ToJson(ConnectStatus.ServerFull);
            return;
        }
        
        // Check build type match
        if (payload.isDebug != Debug.isDebugBuild)
        {
            response.Approved = false;
            response.Reason = JsonUtility.ToJson(ConnectStatus.IncompatibleBuildType);
            return;
        }
        
        // Approve and create player object
        response.Approved = true;
        response.CreatePlayerObject = true;
    }
}
```

---

## Scene Loading in Multiplayer

### The Problem

```csharp
// ❌ WRONG: Regular scene loading doesn't sync
SceneManager.LoadScene("BossRoom");
// Result: Only local machine loads, other clients don't!
```

### The Solution: NetworkSceneManager

```csharp
// ✅ RIGHT: Use NetworkSceneManager for synchronized loading
NetworkManager.Singleton.SceneManager.LoadScene("BossRoom", LoadSceneMode.Single);

// Or use the wrapper:
SceneLoaderWrapper.Instance.LoadScene("BossRoom", useNetworkSceneManager: true);
```

### Scene Loading Events

```csharp
// In ServerBossRoomState.OnNetworkSpawn()
NetworkManager.Singleton.SceneManager.OnLoadEventCompleted += OnLoadEventCompleted;
NetworkManager.Singleton.SceneManager.OnSynchronizeComplete += OnSynchronizeComplete;

void OnLoadEventCompleted(string sceneName, LoadSceneMode loadSceneMode, 
    List<ulong> clientsCompleted, List<ulong> clientsTimedOut)
{
    if (!InitialSpawnDone && loadSceneMode == LoadSceneMode.Single)
    {
        InitialSpawnDone = true;
        // Spawn all connected players
        foreach (var kvp in NetworkManager.Singleton.ConnectedClients)
        {
            SpawnPlayer(kvp.Key, false);
        }
    }
}

void OnSynchronizeComplete(ulong clientId)
{
    // Handle late joiners
    if (InitialSpawnDone && !PlayerServerCharacter.GetPlayerServerCharacter(clientId))
    {
        SpawnPlayer(clientId, true);  // Late join spawn
    }
}
```

---

## Server vs Client States

### ServerBossRoomState

**File:** [ServerBossRoomState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameState/ServerBossRoomState.cs)

**Responsibilities:**
- Spawn players at game start
- Handle late joiners
- Monitor life state changes
- Detect win/lose conditions
- Trigger scene transitions

```csharp
public class ServerBossRoomState : GameStateBehaviour
{
    public override GameState ActiveState => GameState.BossRoom;
    
    void OnNetworkSpawn()
    {
        if (!NetworkManager.Singleton.IsServer)
        {
            enabled = false;  // Only runs on server
            return;
        }
        
        // Subscribe to life state changes
        m_LifeStateChangedEventMessageSubscriber.Subscribe(OnLifeStateChangedEventMessage);
        
        // Subscribe to network events
        NetworkManager.Singleton.SceneManager.OnLoadEventCompleted += OnLoadEventCompleted;
    }
    
    void OnLifeStateChangedEventMessage(LifeStateChangedEventMessage message)
    {
        switch (message.CharacterType)
        {
            case CharacterTypeEnum.Tank:
            case CharacterTypeEnum.Archer:
            case CharacterTypeEnum.Mage:
            case CharacterTypeEnum.Rogue:
                if (message.NewLifeState == LifeState.Fainted)
                    CheckForGameOver();
                break;
            case CharacterTypeEnum.ImpBoss:
                if (message.NewLifeState == LifeState.Dead)
                    BossDefeated();
                break;
        }
    }
    
    void CheckForGameOver()
    {
        // If all players fainted, game over
        foreach (var serverCharacter in PlayerServerCharacter.GetPlayerServerCharacters())
        {
            if (serverCharacter.LifeState == LifeState.Alive)
                return;  // Someone still alive
        }
        
        StartCoroutine(CoroGameOver(k_LoseDelay, false));
    }
    
    IEnumerator CoroGameOver(float wait, bool gameWon)
    {
        m_PersistentGameState.SetWinState(gameWon ? WinState.Win : WinState.Loss);
        yield return new WaitForSeconds(wait);
        SceneLoaderWrapper.Instance.LoadScene("PostGame", useNetworkSceneManager: true);
    }
}
```

### ServerCharSelectState

**File:** [ServerCharSelectState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameState/ServerCharSelectState.cs)

**Responsibilities:**
- Manage lobby slots
- Validate character selections
- Detect when all players ready
- Start the game

---

## The NetcodeHooks Pattern

**Problem:** NetworkBehaviour lifecycle is different from MonoBehaviour.

**Solution:** NetcodeHooks provides familiar callbacks.

```csharp
[RequireComponent(typeof(NetcodeHooks))]
public class ServerBossRoomState : GameStateBehaviour
{
    [SerializeField] NetcodeHooks m_NetcodeHooks;

    protected override void Awake()
    {
        base.Awake();
        // Subscribe to network lifecycle
        m_NetcodeHooks.OnNetworkSpawnHook += OnNetworkSpawn;
        m_NetcodeHooks.OnNetworkDespawnHook += OnNetworkDespawn;
    }
    
    void OnNetworkSpawn()
    {
        // Safe to use NetworkManager here
    }
}
```

---

## Key Takeaways

### Startup Flow
```
Startup Scene → ApplicationController.Configure() → DI Setup
              → ApplicationController.Start() → Load MainMenu
```

### Scene Transitions
```
MainMenu → (Host/Join) → ConnectionState changes → CharSelect → BossRoom → PostGame
```

### Important Patterns

| Pattern | Purpose |
|---------|---------|
| **GameStateBehaviour** | Scene-specific logic |
| **ConnectionState** | Connection state machine |
| **Server/Client States** | Separate server and client logic |
| **NetcodeHooks** | Bridge MonoBehaviour and NetworkBehaviour |
| **NetworkSceneManager** | Synchronized scene loading |

### The Golden Rules

1. **DI container is the root.** Everything starts from ApplicationController.
2. **Connection states control flow.** Scene transitions happen in state Enter/Exit.
3. **Use NetworkSceneManager.** Never use raw SceneManager for multiplayer scenes.
4. **Separate server/client states.** Same pattern as character classes.
5. **Subscribe to scene events.** OnLoadEventCompleted, OnSynchronizeComplete.

---

## Exercises

1. **Trace a Host Flow:** Start at MainMenu, trace through to character spawn in BossRoom. List every state transition.

2. **Add a New Scene:** Create a "Tutorial" scene that loads after CharSelect for new players only.

3. **Find Scene Transitions:** Search for `LoadScene` in the codebase. List all scene transitions and what triggers them.

---

> **Next:** [20_session_reconnection.md](./20_session_reconnection.md) — Session management and reconnection
