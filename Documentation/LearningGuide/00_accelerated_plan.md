# 🚀 7-Day Accelerated Learning Path

> **For intermediate/advanced developers.** You know Unity. You know C#. You want the architecture patterns fast but thorough.

---

## How This Works

| 30-Day Plan | 7-Day Plan |
|-------------|------------|
| 2-3 hours/day | 4-6 hours/day |
| Detailed theory for beginners | Concise theory, more code |
| Step-by-step guidance | Direct file references with line numbers |
| Basic exercises | Real implementation challenges |

**Prerequisites:** Unity experience, C# proficiency, basic networking concepts.

---

# Day 1: Architecture Overview & DI Foundation

### 📖 Required Reading (2 hours)
1. [A1_code_navigation_guide.md](./A1_code_navigation_guide.md) - File lookup reference
2. [01_architecture_principles.md](./01_architecture_principles.md) - Core architectural patterns
3. [19_game_flow_deepdive.md](./19_game_flow_deepdive.md) - Startup and scene flow

---

### 💻 Deep Dive: ApplicationController (1.5 hours)

**File:** `Assets/Scripts/ApplicationLifecycle/ApplicationController.cs`

This is THE entry point. Study these sections:

#### Section 1: DI Registration (Configure method)
```csharp
protected override void Configure(IContainerBuilder builder)
{
    // Components from Inspector
    builder.RegisterComponent(m_UpdateRunner);
    builder.RegisterComponent(m_ConnectionManager);
    builder.RegisterComponent(m_NetworkManager);

    // Singletons - one instance for entire app lifetime
    builder.Register<LocalSessionUser>(Lifetime.Singleton);
    builder.Register<LocalSession>(Lifetime.Singleton);
    builder.Register<ProfileManager>(Lifetime.Singleton);
    builder.Register<PersistentGameState>(Lifetime.Singleton);

    // Message Channels - PubSub infrastructure
    builder.RegisterInstance(new MessageChannel<QuitApplicationMessage>()).AsImplementedInterfaces();
    builder.RegisterInstance(new MessageChannel<ConnectStatus>()).AsImplementedInterfaces();
    
    // Networked channels - sync across network
    builder.RegisterComponent(new NetworkedMessageChannel<LifeStateChangedEventMessage>()).AsImplementedInterfaces();
}
```

**Key insight:** Everything here is available via `[Inject]` anywhere in the app.

#### Section 2: Bootstrap (Start method)
```csharp
private void Start()
{
    m_LocalSession = Container.Resolve<LocalSession>();
    m_MultiplayerServicesFacade = Container.Resolve<MultiplayerServicesFacade>();
    
    // Subscribe to quit event
    var quitApplicationSub = Container.Resolve<ISubscriber<QuitApplicationMessage>>();
    subHandles.Add(quitApplicationSub.Subscribe(QuitGame));
    
    DontDestroyOnLoad(gameObject);
    Application.targetFrameRate = 120;
    
    SceneManager.LoadScene("MainMenu");  // ← First scene load
}
```

---

### 💻 Deep Dive: Connection State Machine (1.5 hours)

**File:** `Assets/Scripts/ConnectionManagement/ConnectionManager.cs`

#### The State Pattern Implementation

```csharp
// Pre-created states (no allocation during gameplay)
internal readonly OfflineState m_Offline = new OfflineState();
internal readonly ClientConnectingState m_ClientConnecting = new ClientConnectingState();
internal readonly ClientConnectedState m_ClientConnected = new ClientConnectedState();
internal readonly ClientReconnectingState m_ClientReconnecting = new ClientReconnectingState();
internal readonly StartingHostState m_StartingHost = new StartingHostState();
internal readonly HostingState m_Hosting = new HostingState();

// State transition (the ONLY way states change)
internal void ChangeState(ConnectionState nextState)
{
    Debug.Log($"Changed state from {m_CurrentState} to {nextState}");
    m_CurrentState?.Exit();
    m_CurrentState = nextState;
    m_CurrentState.Enter();
}

// Event routing to current state
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

**File:** `Assets/Scripts/ConnectionManagement/ConnectionState/HostingState.cs`

#### Connection Approval (lines 90-165)
```csharp
public override void ApprovalCheck(Request request, Response response)
{
    // 1. DOS protection
    if (payload.Length > MaxConnectPayload) { reject; }
    
    // 2. Parse connection data
    var connectionPayload = JsonUtility.FromJson<ConnectionPayload>(payloadString);
    
    // 3. Check if already connected elsewhere
    if (IsDuplicateConnection(connectionPayload.playerId)) { reject ServerFull; }
    
    // 4. Check server capacity
    if (connectedClients >= maxPlayers) { reject ServerFull; }
    
    // 5. Check build compatibility
    if (payload.isDebug != Debug.isDebugBuild) { reject IncompatibleBuildType; }
    
    // 6. Approve
    response.Approved = true;
    response.CreatePlayerObject = true;
    
    // 7. Setup session data
    SessionManager<SessionPlayerData>.Instance.SetupConnectingPlayerSessionData(...);
}
```

---

### ✅ Day 1 Checkpoint

**Answer these from memory:**

1. **Where are all services registered?**
   > `ApplicationController.Configure()` method

2. **Why pre-create state instances?**
   > No memory allocation during gameplay (GC avoidance), states can hold injected dependencies

3. **What's the complete startup flow?**
   > ApplicationController.Configure() → DI container built → Start() → Resolve services → LoadScene("MainMenu")

4. **What happens in HostingState.ApprovalCheck?**
   > DOS check → Parse payload → Duplicate check → Capacity check → Build compatibility → Approve → Setup session data

---

# Day 2: Character System - The Core Architecture

### 📖 Required Reading (1.5 hours)
1. [18_character_system_deepdive.md](./18_character_system_deepdive.md) - **CRITICAL**
2. [03_networking_essentials.md](./03_networking_essentials.md) - NetworkVariables, RPCs

---

### 💻 Deep Dive: ServerCharacter (2 hours)

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs`

#### NetworkVariables (lines 50-80)
```csharp
public NetworkVariable<MovementStatus> MovementStatus { get; } = new();
public NetworkVariable<ulong> HeldNetworkObject { get; } = new();
public NetworkVariable<bool> IsStealthy { get; } = new();
public NetworkVariable<ulong> TargetId { get; } = new();

// Health and Life state are in separate components
public NetworkHealthState NetHealthState { get; private set; }
public NetworkLifeState NetLifeState { get; private set; }
```

#### The Critical OnNetworkSpawn Pattern (lines 100-140)
```csharp
public override void OnNetworkSpawn()
{
    // CRITICAL: Disable on clients - this runs ONLY on server
    if (!IsServer)
    {
        enabled = false;
        return;
    }
    
    // Subscribe to state changes
    NetLifeState.LifeState.OnValueChanged += OnLifeStateChanged;
    m_DamageReceiver.DamageReceived += ReceiveHP;
    
    // Initialize AI for NPCs
    if (IsNpc)
    {
        m_AIBrain = new AIBrain(this, m_ServerActionPlayer);
    }
    
    InitializeHitPoints();
}
```

#### Server RPCs (receive from clients)
```csharp
[Rpc(SendTo.Server)]
public void ServerSendCharacterInputRpc(Vector3 movementTarget)
{
    // Validate state
    if (LifeState != LifeState.Alive) return;
    if (m_Movement.IsPerformingForcedMovement()) return;
    
    // Cancel interruptible actions
    if (m_ServerActionPlayer.GetActiveActionInfo(out var data))
    {
        if (GameDataSource.Instance.GetActionPrototypeByID(data.ActionID).Config.ActionInterruptible)
        {
            m_ServerActionPlayer.ClearActions(false);
        }
    }
    
    m_Movement.SetMovementTarget(movementTarget);
}

[Rpc(SendTo.Server)]
public void ServerPlayActionRpc(ActionRequestData data)
{
    // Notify running actions (e.g., Stealth cancels on attack)
    if (!GameDataSource.Instance.GetActionPrototypeByID(data.ActionID).Config.IsFriendly)
    {
        ActionPlayer.OnGameplayActivity(Action.GameplayActivity.UsingAttackAction);
    }
    PlayAction(ref data);
}
```

#### Damage System (ReceiveHP method)
```csharp
void ReceiveHP(ServerCharacter inflicter, int HP)
{
    if (HP > 0) // Healing
    {
        m_ServerActionPlayer.OnGameplayActivity(Action.GameplayActivity.Healed);
        float healingMod = m_ServerActionPlayer.GetBuffedValue(Action.BuffableValue.PercentHealingReceived);
        HP = (int)(HP * healingMod);
    }
    else // Damage
    {
        m_ServerActionPlayer.OnGameplayActivity(Action.GameplayActivity.AttackedByEnemy);
        float damageMod = m_ServerActionPlayer.GetBuffedValue(Action.BuffableValue.PercentDamageReceived);
        HP = (int)(HP * damageMod);
        serverAnimationHandler.NetworkAnimator.SetTrigger("HitReact1");
    }
    
    HitPoints = Mathf.Clamp(HitPoints + HP, 0, CharacterClass.BaseHP.Value);
    
    if (HitPoints <= 0)
    {
        LifeState = IsNpc ? LifeState.Dead : LifeState.Fainted;
        m_ServerActionPlayer.ClearActions(false);
    }
}
```

---

### 💻 Deep Dive: ClientCharacter (1 hour)

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/ClientCharacter.cs`

#### Subscribing to NetworkVariable Changes
```csharp
public override void OnNetworkSpawn()
{
    if (!IsClient || transform.parent == null) return;
    
    enabled = true;
    m_ServerCharacter = GetComponentInParent<ServerCharacter>();
    
    // Subscribe to server state changes
    m_ServerCharacter.IsStealthy.OnValueChanged += OnStealthyChanged;
    m_ServerCharacter.MovementStatus.OnValueChanged += OnMovementStatusChanged;
    
    // Initialize lerping for smooth movement
    m_PositionLerper = new PositionLerper(serverCharacter.physicsWrapper.Transform.position, k_LerpTime);
    
    // For owning player only
    if (m_ServerCharacter.IsOwner)
    {
        gameObject.AddComponent<CameraController>();
        
        // Client anticipation: play actions before server confirms
        if (!IsServer)
        {
            inputSender.ActionInputEvent += OnActionInput;
        }
    }
}

void OnActionInput(ActionRequestData data)
{
    // ANTICIPATION: Play action visually immediately
    m_ClientActionViz.AnticipateAction(ref data);
}
```

#### Client RPCs (receive from server)
```csharp
[Rpc(SendTo.ClientsAndHost)]
public void ClientPlayActionRpc(ActionRequestData data)
{
    m_ClientActionViz.PlayAction(ref data);
}

[Rpc(SendTo.ClientsAndHost)]
public void ClientCancelAllActionsRpc()
{
    m_ClientActionViz.CancelAllActions();
}
```

---

### ✅ Day 2 Checkpoint

**The Core Communication Flow:**
```
┌─────────────────────────────────────────────────────────────────────┐
│  CLIENT (Input)                                                      │
│  ClientInputSender.OnClick()                                         │
│      ↓                                                              │
│  AnticipateAction() - show visual immediately                        │
│      ↓                                                              │
│  ServerPlayActionRpc() ─────────[RPC]─────────────────────────────► │
└─────────────────────────────────────────────────────────────────────┘
                                                                       │
┌──────────────────────────────────────────────────────────────────────┘
│  SERVER                                                              │
│  ServerCharacter.ServerPlayActionRpc()                               │
│      ↓                                                              │
│  ServerActionPlayer.PlayAction()                                     │
│      ↓                                                              │
│  Action.OnStart() → game logic executes                              │
│      ↓                                                              │
│  ClientCharacter.ClientPlayActionRpc() ◄─────[RPC]──────────────────│
└─────────────────────────────────────────────────────────────────────┘
                                                                       │
┌──────────────────────────────────────────────────────────────────────┘
│  ALL CLIENTS                                                         │
│  ClientCharacter.ClientPlayActionRpc()                               │
│      ↓                                                              │
│  ClientActionPlayer.PlayAction() → visuals for everyone              │
└─────────────────────────────────────────────────────────────────────┘
```

**Answer these:**

1. **Why `if (!IsServer) { enabled = false; }`?**
   > Ensures ServerCharacter.Update() never runs on clients. Server logic stays on server.

2. **How does HP change sync to clients?**
   > `HitPoints` is a NetworkVariable. Setting its value on server auto-syncs to all clients.

3. **What is client anticipation?**
   > Owning client plays action visually before server confirms, hiding network latency.

---

# Day 3: Action System Complete

### 📖 Required Reading (2 hours)
1. [09_action_system_deepdive.md](./09_action_system_deepdive.md) - Complete system analysis

---

### 💻 Deep Dive: Action Base Class (1 hour)

**File:** `Assets/Scripts/Gameplay/Action/Action.cs`

#### Lifecycle Methods
```csharp
public abstract class Action
{
    // Configuration (from ScriptableObject)
    public ActionConfig Config;
    public ActionRequestData Data;
    
    // Lifecycle
    public virtual bool OnStart(ServerCharacter parent)
    {
        // Return false to cancel action before it starts
        return true;
    }
    
    public virtual bool OnUpdate(ServerCharacter parent)
    {
        // Return false when action should end
        return TimeRunning < Config.ExecTimeSeconds;
    }
    
    public virtual void OnEnd(ServerCharacter parent)
    {
        // Cleanup when action completes normally
    }
    
    public virtual void Cancel(ServerCharacter parent)
    {
        // Cleanup when action is interrupted
    }
    
    // Called for gameplay events (e.g., getting attacked while stealthed)
    public virtual void OnGameplayActivity(ServerCharacter parent, GameplayActivity activity) { }
    
    // For buff actions to modify values
    public virtual float GetBuffedValue(BuffableValue value) => 1f;
}
```

---

### 💻 Deep Dive: ServerActionPlayer (1.5 hours)

**File:** `Assets/Scripts/Gameplay/Action/ActionPlayers/ServerActionPlayer.cs`

#### Core Data Structures
```csharp
public class ServerActionPlayer
{
    private List<Action> m_Queue;              // Blocking actions (one at a time)
    private List<Action> m_NonBlockingActions; // Run in parallel (buffs, etc.)
    private Dictionary<ActionID, float> m_LastUsedTimestamps; // Cooldowns
    
    private const float k_MaxQueueTimeDepth = 1.6f; // Max seconds of queued actions
}
```

#### PlayAction Entry Point
```csharp
public void PlayAction(ref ActionRequestData data)
{
    // Create action from pool
    Action action = ActionFactory.CreateActionFromData(ref data);
    
    // Check queue depth (prevent spamming)
    if (GetQueueTimeDepth() > k_MaxQueueTimeDepth)
    {
        ReturnActionToPool(action);
        return;
    }
    
    // Add to queue
    m_Queue.Add(action);
    
    // If nothing running, start immediately
    if (m_Queue.Count == 1)
    {
        StartAction();
    }
}
```

#### StartAction with Synthesized Actions
```csharp
private void StartAction()
{
    // Check cooldown
    if (Time.time - lastTimeUsed < reuseTime)
    {
        AdvanceQueue(false);
        return;
    }
    
    // SYNTHESIZE helper actions if needed
    SynthesizeTargetIfNecessary(0);  // Auto-target enemy
    SynthesizeChaseIfNecessary(0);   // Auto-move in range
    
    // Start the action
    m_Queue[0].TimeStarted = Time.time;
    bool play = m_Queue[0].OnStart(m_ServerCharacter);
    
    if (!play)
    {
        AdvanceQueue(false);
        return;
    }
    
    // Record cooldown
    m_LastUsedTimestamps[action.ActionID] = Time.time;
    
    // Notify client
    m_ClientCharacter.ClientPlayActionRpc(data);
}
```

#### Synthesized Actions Explained
```csharp
private int SynthesizeChaseIfNecessary(int baseIndex)
{
    Action baseAction = m_Queue[baseIndex];
    
    // Does this action need to close distance?
    if (baseAction.Data.ShouldClose && baseAction.Data.TargetIds != null)
    {
        // Are we out of range?
        if (Vector3.Distance(position, targetPosition) > baseAction.Config.Range)
        {
            // Insert Chase action BEFORE the queued action
            ActionRequestData chaseData = new ActionRequestData
            {
                ActionID = GameDataSource.Instance.GeneralChaseActionPrototype.ActionID,
                TargetIds = baseAction.Data.TargetIds,
                Amount = baseAction.Config.Range  // Stop when in range
            };
            
            Action chaseAction = ActionFactory.CreateActionFromData(ref chaseData);
            m_Queue.Insert(baseIndex, chaseAction);
            
            return baseIndex + 1;  // Base action is now one position later
        }
    }
    return baseIndex;
}
```

---

### 💻 Deep Dive: Concrete Action Example (30 min)

**File:** `Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.cs`

```csharp
public partial class MeleeAction : Action
{
    private HashSet<ulong> m_HitTargets;  // Prevent hitting same target twice
    
    public override bool OnStart(ServerCharacter parent)
    {
        m_HitTargets = new HashSet<ulong>();
        parent.serverAnimationHandler.NetworkAnimator.SetTrigger(Config.Anim);
        return true;
    }
    
    public override bool OnUpdate(ServerCharacter parent)
    {
        // Get all colliders in attack range
        var colliders = Physics.OverlapSphere(position, Config.Radius, layerMask);
        
        foreach (var collider in colliders)
        {
            var target = collider.GetComponent<IDamageable>();
            if (target == null) continue;
            
            ulong targetId = target.NetworkObjectId;
            if (m_HitTargets.Contains(targetId)) continue;  // Already hit
            
            // Deal damage!
            target.ReceiveHP(parent, -Config.Amount);
            m_HitTargets.Add(targetId);
        }
        
        return base.OnUpdate(parent);  // Continue until ExecTime expires
    }
    
    public override void OnEnd(ServerCharacter parent)
    {
        m_HitTargets = null;
    }
}
```

---

### ✅ Day 3 Checkpoint

**Action System Flow:**
```
ActionConfig (ScriptableObject)
    ↓ defines
Action (class)
    ↓ created by
ActionFactory (pooled)
    ↓ managed by
ServerActionPlayer (queue)
    ↓ lifecycle
OnStart() → OnUpdate() → OnEnd()
    ↓ visual sync
ClientActionPlayer (RPC)
```

**Answer these:**

1. **What are blocking modes?**
   > `BlocksAllButMovement`: Can't do other actions during this
   > `OnlyDuringExecTime`: Blocks during exec, then becomes non-blocking
   > `NonBlocking`: Runs in parallel with other actions (buffs)

2. **What are synthesized actions?**
   > Auto-inserted Chase (move in range) and Target (change target) actions when player uses ability on distant/different target

3. **Why pool actions?**
   > Actions created frequently during combat. Pooling avoids GC spikes.

---

# Day 4: Game States & Session Management

### 📖 Required Reading (1.5 hours)
1. [19_game_flow_deepdive.md](./19_game_flow_deepdive.md) - GameStateBehaviour section
2. [20_session_reconnection.md](./20_session_reconnection.md) - Session persistence

---

### 💻 Deep Dive: GameStateBehaviour (1 hour)

**File:** `Assets/Scripts/Gameplay/GameState/GameStateBehaviour.cs`

```csharp
public abstract class GameStateBehaviour : MonoBehaviour
{
    public abstract GameState ActiveState { get; }
    public static GameStateBehaviour ActiveInstance { get; private set; }
    
    protected virtual void Awake()
    {
        if (ActiveInstance != null && ActiveInstance != this)
        {
            Destroy(gameObject);
            return;
        }
        ActiveInstance = this;
    }
}
```

**File:** `Assets/Scripts/Gameplay/GameState/ServerBossRoomState.cs`

#### Player Spawning
```csharp
void SpawnPlayer(ulong clientId, bool lateJoin)
{
    // 1. Pick spawn point
    int index = Random.Range(0, m_PlayerSpawnPointsList.Count);
    Transform spawnPoint = m_PlayerSpawnPointsList[index];
    m_PlayerSpawnPointsList.RemoveAt(index);
    
    // 2. Get persistent player data
    var persistentPlayer = NetworkManager.Singleton.SpawnManager
        .GetPlayerNetworkObject(clientId).GetComponent<PersistentPlayer>();
    
    // 3. Instantiate character
    var newPlayer = Instantiate(m_PlayerPrefab, Vector3.zero, Quaternion.identity);
    
    // 4. Restore position for reconnecting players
    if (lateJoin)
    {
        var sessionData = SessionManager<SessionPlayerData>.Instance.GetPlayerData(clientId);
        if (sessionData is { HasCharacterSpawned: true })
        {
            physicsTransform.SetPositionAndRotation(
                sessionData.Value.PlayerPosition,
                sessionData.Value.PlayerRotation);
        }
    }
    
    // 5. Spawn with ownership
    newPlayer.SpawnWithOwnership(clientId, destroyWithScene: true);
}
```

#### Game Over Detection
```csharp
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
    foreach (var serverCharacter in PlayerServerCharacter.GetPlayerServerCharacters())
    {
        if (serverCharacter.LifeState == LifeState.Alive)
            return;  // Someone still alive
    }
    
    // All players fainted - game over
    StartCoroutine(CoroGameOver(k_LoseDelay, gameWon: false));
}
```

---

### 💻 Deep Dive: SessionManager (1 hour)

**File:** `Packages/com.unity.multiplayer.samples.coop/Utilities/Net/SessionManager.cs`

#### The Two-Dictionary Pattern
```csharp
public class SessionManager<T> where T : struct, ISessionPlayerData
{
    // PlayerId (stable, per device) → Session Data
    Dictionary<string, T> m_ClientData;
    
    // ClientId (changes each connection) → PlayerId
    Dictionary<ulong, string> m_ClientIDToPlayerId;
    
    bool m_HasSessionStarted;
}
```

#### Reconnection Logic
```csharp
public void SetupConnectingPlayerSessionData(ulong clientId, string playerId, T sessionPlayerData)
{
    // Duplicate check
    if (IsDuplicateConnection(playerId)) return;
    
    // Reconnection detection
    bool isReconnecting = m_ClientData.ContainsKey(playerId) 
                       && !m_ClientData[playerId].IsConnected;
    
    if (isReconnecting)
    {
        // Use OLD data (preserve position, HP, etc.)
        sessionPlayerData = m_ClientData[playerId];
        sessionPlayerData.ClientID = clientId;  // Update to new ClientId
        sessionPlayerData.IsConnected = true;
    }
    
    m_ClientIDToPlayerId[clientId] = playerId;
    m_ClientData[playerId] = sessionPlayerData;
}

public void DisconnectClient(ulong clientId)
{
    if (m_HasSessionStarted)
    {
        // Game in progress - KEEP data for reconnection
        clientData.IsConnected = false;
    }
    else
    {
        // Still in lobby - DELETE data
        m_ClientData.Remove(playerId);
    }
}
```

---

### ✅ Day 4 Checkpoint

**Scene Flow:**
```
MainMenu → CharSelect → BossRoom → PostGame
              ↑                        │
              └────────────────────────┘
```

**Answer these:**

1. **PlayerId vs ClientId?**
   > PlayerId: Stable per device/install, survives reconnection
   > ClientId: Assigned by Netcode each connection, changes on reconnect

2. **Why keep session data on disconnect?**
   > To restore player position, HP, character when they reconnect mid-game

3. **What triggers scene loads?**
   > HostingState.Enter() → CharSelect
   > ServerCharSelectState (all locked) → BossRoom
   > ServerBossRoomState (win/loss) → PostGame

---

# Day 5: Infrastructure Patterns

### 📖 Required Reading (1.5 hours)
1. [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) - All patterns
2. [16_performance_patterns.md](./16_performance_patterns.md) - Optimization

---

### 💻 Deep Dive: MessageChannel (1 hour)

**File:** `Assets/Scripts/Infrastructure/PubSub/MessageChannel.cs`

#### The Pending Handlers Pattern
```csharp
public class MessageChannel<T> : IMessageChannel<T>
{
    private HashSet<Action<T>> m_MessageHandlers = new();
    private Dictionary<Action<T>, bool> m_PendingHandlers = new();  // true=add, false=remove
    
    public void Publish(T message)
    {
        // 1. Apply pending modifications FIRST
        foreach (var pending in m_PendingHandlers)
        {
            if (pending.Value)
                m_MessageHandlers.Add(pending.Key);
            else
                m_MessageHandlers.Remove(pending.Key);
        }
        m_PendingHandlers.Clear();
        
        // 2. Now safe to invoke handlers
        foreach (var handler in m_MessageHandlers)
        {
            handler?.Invoke(message);
        }
    }
    
    public IDisposable Subscribe(Action<T> handler)
    {
        // Defer addition until next Publish
        m_PendingHandlers[handler] = true;
        return new DisposableSubscription<T>(this, handler);
    }
    
    public void Unsubscribe(Action<T> handler)
    {
        // Defer removal until next Publish
        m_PendingHandlers[handler] = false;
    }
}
```

**Why this pattern?** If a handler unsubscribes during Publish(), modifying the collection would crash. Deferring changes makes it safe.

---

### 💻 Deep Dive: Object Pooling (1 hour)

**File:** `Assets/Scripts/Infrastructure/NetworkObjectPool.cs`

```csharp
public class NetworkObjectPool : MonoBehaviour
{
    [SerializeField] List<PoolConfigObject> PooledPrefabsList;
    
    private Dictionary<GameObject, Queue<NetworkObject>> m_PooledObjects = new();
    
    public NetworkObject GetNetworkObject(GameObject prefab, Vector3 position, Quaternion rotation)
    {
        var queue = m_PooledObjects[prefab];
        
        if (queue.Count > 0)
        {
            // Reuse from pool
            var obj = queue.Dequeue();
            obj.transform.SetPositionAndRotation(position, rotation);
            obj.gameObject.SetActive(true);
            return obj;
        }
        
        // Pool empty - instantiate new
        return Instantiate(prefab, position, rotation).GetComponent<NetworkObject>();
    }
    
    public void ReturnNetworkObject(NetworkObject obj, GameObject prefab)
    {
        obj.gameObject.SetActive(false);
        m_PooledObjects[prefab].Enqueue(obj);
    }
}
```

**File:** `Assets/Scripts/Utils/DisposableGroup.cs`

```csharp
public class DisposableGroup : IDisposable
{
    private List<IDisposable> m_Disposables = new();
    
    public void Add(IDisposable disposable) => m_Disposables.Add(disposable);
    
    public void Dispose()
    {
        foreach (var d in m_Disposables)
            d.Dispose();
        m_Disposables.Clear();
    }
}

// Usage:
class MyComponent : MonoBehaviour
{
    DisposableGroup m_Subscriptions = new();
    
    void Start()
    {
        m_Subscriptions.Add(channel1.Subscribe(OnEvent1));
        m_Subscriptions.Add(channel2.Subscribe(OnEvent2));
    }
    
    void OnDestroy()
    {
        m_Subscriptions.Dispose();  // Unsubscribes everything
    }
}
```

---

### ✅ Day 5 Checkpoint

**Answer these:**

1. **Why pending handlers?**
   > Safe to unsubscribe during Publish() - changes deferred until next Publish

2. **Why pool network objects?**
   > Instantiate/Destroy causes GC. Pooling reuses objects.

3. **What is DisposableGroup for?**
   > Collect multiple subscriptions, dispose all at once in OnDestroy

---

# Day 6: Design Patterns & Clean Code

### 📖 Required Reading (2 hours)
1. [02_clean_code_patterns.md](./02_clean_code_patterns.md) - SOLID principles
2. [04_design_patterns.md](./04_design_patterns.md) - Pattern catalog
3. [17_architecture_decision_framework.md](./17_architecture_decision_framework.md) - When to use what

---

### Pattern Quick Reference

| Pattern | When | Boss Room Example |
|---------|------|-------------------|
| **State Machine** | 3+ states, different behavior each | ConnectionManager, Action BlockingModes |
| **PubSub/Observer** | Multiple listeners for one event | MessageChannel, NetworkVariable.OnValueChanged |
| **Factory** | Complex creation, pooling | ActionFactory |
| **Object Pool** | Frequent create/destroy | NetworkObjectPool, ActionFactory |
| **Command** | Queue, undo, or log operations | Action system queue |
| **Mediator** | Many-to-many communication | (Not heavily used) |
| **Strategy** | Swap algorithms at runtime | ConnectionMethod (IP vs Relay) |

---

### SOLID in Boss Room

```
S - Single Responsibility
    ServerCharacter: game logic only
    ClientCharacter: visuals only
    
O - Open/Closed
    New actions without modifying ActionPlayer
    Just create new Action class + ActionConfig
    
L - Liskov Substitution
    All Actions work with ServerActionPlayer
    All ConnectionStates work with ConnectionManager
    
I - Interface Segregation
    IPublisher<T> and ISubscriber<T> are separate
    Classes only depend on what they need
    
D - Dependency Inversion
    Inject IPublisher<T>, not MessageChannel<T>
    Inject interfaces, swap implementations for testing
```

---

### When NOT to Use Patterns

- **Don't use State Machine** for 2 simple boolean states (if/else is fine)
- **Don't use PubSub** for 1:1 direct relationships (just call the method)
- **Don't use Factory** for simple `new MyClass()` creation
- **Don't use Pool** for rarely-created objects

---

### ✅ Day 6 Checkpoint

**Answer these:**

1. **Which patterns apply to offline games?**
   > DI, PubSub, State Machine, Data-Driven, Object Pool, Factory
   > Skip: NetworkVariables, RPCs, Server/Client split

2. **When State Machine vs if/else?**
   > State Machine: 3+ states, complex transitions, different methods per state
   > If/else: 2 states, simple conditions

3. **When PubSub vs direct call?**
   > PubSub: Multiple listeners, decoupling needed
   > Direct: Single listener, simple relationship

---

# Day 7: Integration & Mastery

### 📖 Final Reading (1 hour)
1. [13_code_reading_walkthroughs.md](./13_code_reading_walkthroughs.md) - Practice exercises
2. [06_offline_game_guide.md](./06_offline_game_guide.md) - Offline applicability

---

### Complete System Trace (2 hours)

Trace through the entire codebase for this scenario:

**Scenario:** Player A hosts, Player B joins, both pick characters, fight boss, boss dies

| Step | Key Files | Key Methods |
|------|-----------|-------------|
| A clicks Host | ConnectionManager | ChangeState(m_StartingHost) |
| Host ready | StartingHostState | OnServerStarted() → ChangeState(m_Hosting) |
| Scene loads | HostingState | Enter() → LoadScene("CharSelect") |
| B clicks Join | ConnectionManager | StartClientIP() |
| B approved | HostingState | ApprovalCheck() → CreatePlayerObject |
| A picks Tank | ServerCharSelectState | ChangeSeatRpc() |
| Both lock in | ServerCharSelectState | CheckForAllLockedIn() → LoadScene("BossRoom") |
| Characters spawn | ServerBossRoomState | OnLoadEventCompleted() → SpawnPlayer() |
| A attacks | ClientInputSender | ServerPlayActionRpc() |
| Action plays | ServerActionPlayer | PlayAction() → MeleeAction.OnStart() |
| Damage dealt | MeleeAction | OnUpdate() → IDamageable.ReceiveHP() |
| Boss HP = 0 | ServerCharacter | ReceiveHP() → LifeState = Dead |
| Game over | ServerBossRoomState | OnLifeStateChanged() → BossDefeated() |
| PostGame | ServerBossRoomState | LoadScene("PostGame") |

---

### Self-Assessment

Without notes, explain:
- [ ] The complete startup flow from launch to gameplay
- [ ] Why ServerCharacter and ClientCharacter are separate
- [ ] How actions flow from input to visual effect
- [ ] The connection state machine transitions
- [ ] How reconnection preserves player state
- [ ] When to use each design pattern

---

### Apply Your Knowledge

**Challenge 1: Add an Ability**
1. Create ActionConfig ScriptableObject
2. Create Action class inheriting from Action
3. Implement OnStart(), OnUpdate(), OnEnd()
4. Test in multiplayer

**Challenge 2: Trace a New Flow**
Pick any system you haven't fully explored:
- AI Brain decision making
- Character Selection seat synchronization
- Lobby services integration

---

## 🎓 Graduation

You've completed the 7-day accelerated path. You now understand:
- ✅ VContainer DI and service registration
- ✅ State machine pattern for connections and actions
- ✅ Server/Client character separation
- ✅ Action system architecture
- ✅ Session management and reconnection
- ✅ Infrastructure patterns (PubSub, pooling, cleanup)
- ✅ Clean code principles in practice

**Go build something amazing!**
