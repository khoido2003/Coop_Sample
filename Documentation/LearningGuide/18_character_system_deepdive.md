# 18: Character System Deep Dive

> **Purpose:** Understand the heart of Boss Room's networked character architecture—why ServerCharacter and ClientCharacter are separate, how they communicate, and how to extend them.

---

## Table of Contents

1. [The Fundamental Question](#the-fundamental-question)
2. [Why Two Character Classes?](#why-two-character-classes)
3. [ServerCharacter Anatomy](#servercharacter-anatomy)
4. [ClientCharacter Anatomy](#clientcharacter-anatomy)
5. [The Communication Flow](#the-communication-flow)
6. [Component Composition Pattern](#component-composition-pattern)
7. [Character Spawn Flow](#character-spawn-flow)
8. [Common Newcomer Mistakes](#common-newcomer-mistakes)
9. [Extending the Character System](#extending-the-character-system)
10. [Key Takeaways](#key-takeaways)

---

## The Fundamental Question

> **"Why are there TWO character classes? Why not just one PlayerCharacter?"**

This is the most common question newcomers ask. The answer is **THE** foundational concept of networked game architecture.

### The Problem with One Class

```csharp
// ❌ BAD: Single class tries to do everything
public class PlayerCharacter : NetworkBehaviour
{
    void Update()
    {
        if (IsServer)
        {
            // Server logic here
            ProcessDamage();
            ValidateMovement();
        }
        
        if (IsClient)
        {
            // Client logic here
            PlayAnimations();
            UpdateHealthBar();
        }
        
        // Wait... what about IsHost? That's both!
        // This gets confusing FAST.
    }
}
```

**Problems:**
- Constant `if (IsServer)` / `if (IsClient)` checks everywhere
- Easy to accidentally run server code on client
- Hard to test either side in isolation
- Mental overhead: "Is this line running on server or client?"

### The Boss Room Solution

```csharp
// ✅ GOOD: Separate files, separate concerns
// ServerCharacter.cs - ONLY server logic
public class ServerCharacter : NetworkBehaviour
{
    void Update()
    {
        // Everything here runs on server. No checks needed.
        m_ServerActionPlayer.OnUpdate();
        m_AIBrain?.Update();
    }
}

// ClientCharacter.cs - ONLY client visuals
public class ClientCharacter : NetworkBehaviour
{
    void Update()
    {
        // Everything here runs on client. No checks needed.
        OurAnimator.SetFloat("Speed", m_CurrentSpeed);
        m_ClientActionViz.OnUpdate();
    }
}
```

---

## Why Two Character Classes?

### Separation of Concerns

| Concern | ServerCharacter | ClientCharacter |
|---------|-----------------|-----------------|
| **Health** | Stores and modifies HP | Displays health bar |
| **Movement** | Validates and applies | Smoothly interpolates |
| **Actions** | Executes logic, deals damage | Plays animations, VFX |
| **AI** | Runs brain, makes decisions | Shows behavior visually |
| **Targeting** | Validates targets | Shows reticle |

### The Mental Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GAME WORLD (Authority)                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ServerCharacter                           │   │
│  │  • NetworkVariables (HitPoints, LifeState, TargetId)        │   │
│  │  • ServerActionPlayer (action queue)                         │   │
│  │  • ServerCharacterMovement (physics)                         │   │
│  │  • AIBrain (for NPCs)                                        │   │
│  │  • Receives RPCs from clients                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              │ NetworkVariables auto-sync           │
│                              │ + explicit RPCs for actions          │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ClientCharacter                           │   │
│  │  • Animator (plays animations)                               │   │
│  │  • ClientActionPlayer (visual effects)                       │   │
│  │  • Position/Rotation lerping (smooth movement)               │   │
│  │  • CharacterSwap (model switching)                           │   │
│  │  • Receives RPCs from server                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ServerCharacter Anatomy

**File:** [ServerCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ServerCharacter.cs)

### Key Fields

```csharp
// Network-synchronized state (auto-synced to all clients)
public NetworkVariable<MovementStatus> MovementStatus { get; }
public NetworkVariable<ulong> HeldNetworkObject { get; }
public NetworkVariable<bool> IsStealthy { get; }
public NetworkVariable<ulong> TargetId { get; }

// Components for delegation
public NetworkHealthState NetHealthState { get; }      // HitPoints
public NetworkLifeState NetLifeState { get; }          // Alive/Fainted/Dead
private ServerActionPlayer m_ServerActionPlayer;       // Action queue
private ServerCharacterMovement m_Movement;            // Physics movement
private AIBrain m_AIBrain;                             // NPC AI (null for players)
```

### The OnNetworkSpawn Pattern

```csharp
public override void OnNetworkSpawn()
{
    // CRITICAL: Disable this component on clients!
    if (!IsServer) { enabled = false; }
    else
    {
        // Subscribe to events
        NetLifeState.LifeState.OnValueChanged += OnLifeStateChanged;
        m_DamageReceiver.DamageReceived += ReceiveHP;
        
        // Initialize AI for NPCs
        if (IsNpc)
        {
            m_AIBrain = new AIBrain(this, m_ServerActionPlayer);
        }
        
        // Play starting action if configured
        if (m_StartingAction != null)
        {
            var startingAction = new ActionRequestData() { ActionID = m_StartingAction.ActionID };
            PlayAction(ref startingAction);
        }
        
        InitializeHitPoints();
    }
}
```

> **Key Insight:** The `if (!IsServer) { enabled = false; }` pattern ensures this component's `Update()` never runs on clients.

### Server RPCs (Client → Server)

```csharp
// Movement input from owning client
[Rpc(SendTo.Server)]
public void ServerSendCharacterInputRpc(Vector3 movementTarget)
{
    if (LifeState == LifeState.Alive && !m_Movement.IsPerformingForcedMovement())
    {
        // Interrupt interruptible actions
        if (m_ServerActionPlayer.GetActiveActionInfo(out ActionRequestData data))
        {
            if (GameDataSource.Instance.GetActionPrototypeByID(data.ActionID).Config.ActionInterruptible)
            {
                m_ServerActionPlayer.ClearActions(false);
            }
        }
        m_Movement.SetMovementTarget(movementTarget);
    }
}

// Action request from owning client
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

### The Damage System

```csharp
void ReceiveHP(ServerCharacter inflicter, int HP)
{
    if (HP > 0)
    {
        // Healing - apply buff modifiers
        m_ServerActionPlayer.OnGameplayActivity(Action.GameplayActivity.Healed);
        float healingMod = m_ServerActionPlayer.GetBuffedValue(Action.BuffableValue.PercentHealingReceived);
        HP = (int)(HP * healingMod);
    }
    else
    {
        // Damage - apply buff modifiers
        m_ServerActionPlayer.OnGameplayActivity(Action.GameplayActivity.AttackedByEnemy);
        float damageMod = m_ServerActionPlayer.GetBuffedValue(Action.BuffableValue.PercentDamageReceived);
        HP = (int)(HP * damageMod);
        
        // Play hit reaction
        serverAnimationHandler.NetworkAnimator.SetTrigger("HitReact1");
    }
    
    // Clamp and apply
    HitPoints = Mathf.Clamp(HitPoints + HP, 0, CharacterClass.BaseHP.Value);
    
    // Check for death
    if (HitPoints <= 0)
    {
        LifeState = IsNpc ? LifeState.Dead : LifeState.Fainted;
        m_ServerActionPlayer.ClearActions(false);
    }
}
```

---

## ClientCharacter Anatomy

**File:** [ClientCharacter.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/ClientCharacter.cs)

### Key Fields

```csharp
[SerializeField] Animator m_ClientVisualsAnimator;        // Animation control
[SerializeField] VisualizationConfiguration m_VisualizationConfiguration; // Speed values, etc.

private ClientActionPlayer m_ClientActionViz;  // Visual action playback
private PositionLerper m_PositionLerper;       // Smooth position interpolation
private RotationLerper m_RotationLerper;       // Smooth rotation interpolation
private ServerCharacter m_ServerCharacter;     // Reference to server component
```

### The OnNetworkSpawn Pattern

```csharp
public override void OnNetworkSpawn()
{
    if (!IsClient || transform.parent == null)
    {
        return;  // Only run on clients with valid parent
    }

    enabled = true;
    m_ClientActionViz = new ClientActionPlayer(this);
    m_ServerCharacter = GetComponentInParent<ServerCharacter>();

    // Subscribe to NetworkVariable changes
    m_ServerCharacter.IsStealthy.OnValueChanged += OnStealthyChanged;
    m_ServerCharacter.MovementStatus.OnValueChanged += OnMovementStatusChanged;

    // Initialize position lerping
    m_PositionLerper = new PositionLerper(serverCharacter.physicsWrapper.Transform.position, k_LerpTime);
    
    // For owning player only
    if (m_ServerCharacter.IsOwner)
    {
        // Start targeting action
        ActionRequestData data = new ActionRequestData { ActionID = GameDataSource.Instance.GeneralTargetActionPrototype.ActionID };
        m_ClientActionViz.PlayAction(ref data);
        
        // Add camera
        gameObject.AddComponent<CameraController>();
        
        // Subscribe to input (for anticipation)
        if (!IsServer)  // Non-host clients only
        {
            inputSender.ActionInputEvent += OnActionInput;
        }
    }
}
```

### Client RPCs (Server → Client)

```csharp
// Server tells all clients to play an action visually
[Rpc(SendTo.ClientsAndHost)]
public void ClientPlayActionRpc(ActionRequestData data)
{
    m_ClientActionViz.PlayAction(ref data);
}

// Server tells all clients to cancel actions
[Rpc(SendTo.ClientsAndHost)]
public void ClientCancelAllActionsRpc()
{
    m_ClientActionViz.CancelAllActions();
}

// Server tells clients charging stopped
[Rpc(SendTo.ClientsAndHost)]
public void ClientStopChargingUpRpc(float percentCharged)
{
    m_ClientActionViz.OnStoppedChargingUp(percentCharged);
}
```

### Position Smoothing (For Host)

```csharp
void Update()
{
    // On host, smooth the visual position
    if (IsHost)
    {
        m_LerpedPosition = m_PositionLerper.LerpPosition(
            m_LerpedPosition,
            serverCharacter.physicsWrapper.Transform.position);
        m_LerpedRotation = m_RotationLerper.LerpRotation(
            m_LerpedRotation,
            serverCharacter.physicsWrapper.Transform.rotation);
        transform.SetPositionAndRotation(m_LerpedPosition, m_LerpedRotation);
    }

    // Update animator
    if (m_ClientVisualsAnimator)
    {
        OurAnimator.SetFloat(m_VisualizationConfiguration.SpeedVariableID, m_CurrentSpeed);
    }

    m_ClientActionViz.OnUpdate();
}
```

---

## The Communication Flow

### Player Presses Attack

```
┌─────────────────────────────────────────────────────────────────────┐
│  CLIENT (Owning Player)                                              │
├─────────────────────────────────────────────────────────────────────┤
│  1. ClientInputSender detects click                                  │
│  2. Creates ActionRequestData                                        │
│  3. Calls AnticipateAction() for immediate feedback                  │
│  4. Sends ServerPlayActionRpc()                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ RPC (network)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SERVER                                                              │
├─────────────────────────────────────────────────────────────────────┤
│  5. ServerCharacter.ServerPlayActionRpc() receives request           │
│  6. Validates action can be performed                                │
│  7. ServerActionPlayer.PlayAction() queues it                        │
│  8. Action.OnStart() executes game logic                             │
│  9. Calls ClientCharacter.ClientPlayActionRpc()                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ RPC (network)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ALL CLIENTS                                                         │
├─────────────────────────────────────────────────────────────────────┤
│  10. ClientCharacter.ClientPlayActionRpc() receives                  │
│  11. ClientActionPlayer.PlayAction() starts visuals                  │
│  12. Animation plays, VFX spawn                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Health Changes

```
┌─────────────────────────────────────────────────────────────────────┐
│  SERVER                                                              │
├─────────────────────────────────────────────────────────────────────┤
│  1. MeleeAction.OnUpdate() detects hit                               │
│  2. Calls IDamageable.ReceiveHP(-25)                                 │
│  3. ServerCharacter.ReceiveHP() applies damage                       │
│  4. Sets NetHealthState.HitPoints.Value -= 25                        │
│     (NetworkVariable automatically syncs!)                           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ NetworkVariable sync (automatic)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ALL CLIENTS                                                         │
├─────────────────────────────────────────────────────────────────────┤
│  5. NetHealthState.HitPoints.OnValueChanged triggers                 │
│  6. UI components subscribed to this event update                    │
│  7. Health bar reflects new value                                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Composition Pattern

ServerCharacter doesn't do everything itself. It delegates to specialized components:

```
ServerCharacter (orchestrator)
├── ServerCharacterMovement    → Physics, pathfinding
├── ServerActionPlayer         → Action queue management
├── NetworkHealthState         → HP synchronization
├── NetworkLifeState           → Alive/Fainted/Dead
├── NetworkAvatarGuidState     → Character class identity
├── ServerAnimationHandler     → Animation triggers on server
├── PhysicsWrapper             → Collision/transform access
├── DamageReceiver             → Collision damage detection
└── AIBrain (NPCs only)        → AI decision making
```

### Why Composition?

```csharp
// ❌ BAD: Everything in one file
public class ServerCharacter
{
    // 2000+ lines of code
    void UpdateMovement() { }
    void UpdateActions() { }
    void UpdateAI() { }
    void UpdateHealth() { }
    // ... impossible to maintain
}

// ✅ GOOD: Delegation to specialists
public class ServerCharacter
{
    void Update()
    {
        m_ServerActionPlayer.OnUpdate();     // ~500 lines in own file
        m_AIBrain?.Update();                 // ~300 lines in own file
        // Movement handled by its own component
    }
}
```

---

## Character Spawn Flow

**File:** [ServerBossRoomState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameState/ServerBossRoomState.cs)

```csharp
void SpawnPlayer(ulong clientId, bool lateJoin)
{
    // 1. Pick spawn point
    int index = Random.Range(0, m_PlayerSpawnPointsList.Count);
    Transform spawnPoint = m_PlayerSpawnPointsList[index];
    m_PlayerSpawnPointsList.RemoveAt(index);

    // 2. Get persistent player data (from CharSelect)
    var playerNetworkObject = NetworkManager.Singleton.SpawnManager.GetPlayerNetworkObject(clientId);
    var persistentPlayer = playerNetworkObject.GetComponent<PersistentPlayer>();

    // 3. Instantiate the player prefab
    var newPlayer = Instantiate(m_PlayerPrefab, Vector3.zero, Quaternion.identity);
    var newPlayerCharacter = newPlayer.GetComponent<ServerCharacter>();

    // 4. Set position
    newPlayerCharacter.physicsWrapper.Transform.SetPositionAndRotation(
        spawnPoint.position, spawnPoint.rotation);

    // 5. Copy avatar GUID from persistent player
    networkAvatarGuidState.AvatarGuid = new NetworkVariable<NetworkGuid>(
        persistentPlayer.NetworkAvatarGuidState.AvatarGuid.Value);

    // 6. Spawn with ownership
    newPlayer.SpawnWithOwnership(clientId, destroyWithScene: true);
}
```

### The Spawn Sequence

```
Scene Load Complete
        │
        ▼
OnLoadEventCompleted
        │
        ▼
foreach (connectedClient)
    SpawnPlayer(clientId)
        │
        ├─► Instantiate prefab
        ├─► Set position/rotation
        ├─► Copy player data
        └─► SpawnWithOwnership()
                │
                ▼
        ServerCharacter.OnNetworkSpawn()
                │
                ▼
        ClientCharacter.OnNetworkSpawn() (all clients)
```

---

## Common Newcomer Mistakes

### Mistake 1: Running Client Code on Server

```csharp
// ❌ WRONG: This is in ServerCharacter but plays animations
void ReceiveHP(...)
{
    PlayHurtAnimation();  // Animations are CLIENT responsibility!
}

// ✅ RIGHT: Server triggers the animation via NetworkAnimator
serverAnimationHandler.NetworkAnimator.SetTrigger("HitReact1");
// NetworkAnimator automatically syncs to clients
```

### Mistake 2: Modifying NetworkVariables on Client

```csharp
// ❌ WRONG: Client tries to change HP
// In ClientCharacter:
void OnPlayerClickedHealButton()
{
    m_ServerCharacter.NetHealthState.HitPoints.Value += 10;  // ERROR!
    // Only server can modify NetworkVariables by default
}

// ✅ RIGHT: Client sends RPC, server modifies
[Rpc(SendTo.Server)]
void ServerRequestHealRpc()
{
    // Server validates and applies
    if (CanHeal())
    {
        HitPoints += 10;
    }
}
```

### Mistake 3: Forgetting to Disable on Wrong Side

```csharp
// ❌ WRONG: No check, runs everywhere
public class ServerCharacter : NetworkBehaviour
{
    void Update()
    {
        ProcessServerLogic();  // Runs on clients too!
    }
}

// ✅ RIGHT: Disable on clients
public override void OnNetworkSpawn()
{
    if (!IsServer) { enabled = false; return; }
    // ... server-only initialization
}
```

### Mistake 4: Direct References Instead of RPCs

```csharp
// ❌ WRONG: Directly calling across network boundary
// In ServerCharacter:
void DealDamage()
{
    m_ClientCharacter.PlayDamageVFX();  // Not networked!
}

// ✅ RIGHT: Use RPC
void DealDamage()
{
    clientCharacter.ClientPlayActionRpc(damageData);
}
```

---

## Extending the Character System

### Adding a New Ability

1. **Create Action ScriptableObject** (see action_system_deepdive.md)
2. **Server logic goes in Action.OnStart/OnUpdate**
3. **Client visuals go in Action.OnStartClient/OnUpdateClient**
4. **No changes to ServerCharacter/ClientCharacter needed!**

### Adding a New Character Stat

```csharp
// 1. Add NetworkVariable to ServerCharacter
public NetworkVariable<int> Mana { get; } = new NetworkVariable<int>();

// 2. Subscribe in ClientCharacter for UI updates
void OnNetworkSpawn()
{
    m_ServerCharacter.Mana.OnValueChanged += OnManaChanged;
}

// 3. Modify on server only
void UseManaAbility(int cost)
{
    if (Mana.Value >= cost)
    {
        Mana.Value -= cost;  // Auto-syncs to clients
    }
}
```

### Adding a New Component

```csharp
// 1. Create ServerXxx component for server logic
public class ServerCharacterInventory : NetworkBehaviour
{
    public override void OnNetworkSpawn()
    {
        if (!IsServer) { enabled = false; }
    }
}

// 2. Create ClientXxx component for client visuals
public class ClientCharacterInventory : NetworkBehaviour
{
    public override void OnNetworkSpawn()
    {
        if (!IsClient) { return; }
    }
}

// 3. Reference from ServerCharacter/ClientCharacter as needed
```

---

## Key Takeaways

| Concept | Server | Client |
|---------|--------|--------|
| **Purpose** | Game logic, authority | Visualization, feedback |
| **Modifies state** | Yes (NetworkVariables) | No (read-only) |
| **Runs Update** | `if (IsServer)` | `if (IsClient)` |
| **Receives input from** | Client RPCs | Local user |
| **Sends data to** | Clients via RPCs/NetworkVars | Server via RPCs |

### The Golden Rules

1. **Server = Truth.** Server decides what happens.
2. **Client = Display.** Client shows what happened.
3. **NetworkVariables = Automatic Sync.** For state that changes often.
4. **RPCs = Explicit Messages.** For events and commands.
5. **Disable on Wrong Side.** First line of OnNetworkSpawn.

---

## Exercises

1. **Trace a Damage Event:** Find where MeleeAction deals damage, trace through ReceiveHP, and find where the health bar updates on clients.

2. **Add a Stamina System:** Create a NetworkVariable for stamina, consume it when sprinting, recover it over time, display it on the UI.

3. **Find All RPCs:** Search for `[Rpc(` in ServerCharacter and ClientCharacter. List what each one does and which direction it goes.

---

> **Next:** [19_game_flow_deepdive.md](./19_game_flow_deepdive.md) — Complete startup to gameplay flow
