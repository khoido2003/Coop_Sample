# 09: Action System Deep Dive

> **Purpose:** This is a comprehensive, book-like guide to the Action system—the heart of Boss Room's combat. By the end, you will understand every piece of this architecture and be able to build similar systems in your own games.

---

## Table of Contents

1. [What is the Action System?](#what-is-the-action-system)
2. [Architecture Overview](#architecture-overview)
3. [The Action Class Hierarchy](#the-action-class-hierarchy)
4. [ActionConfig: Data-Driven Design](#actionconfig-data-driven-design)
5. [Server-Side Execution (ServerActionPlayer)](#server-side-execution-serveractionplayer)
6. [Client-Side Visualization](#client-side-visualization)
7. [Blocking Modes Explained](#blocking-modes-explained)
8. [Client Anticipation Pattern](#client-anticipation-pattern)
9. [The Complete MeleeAction Walkthrough](#the-complete-meleeaction-walkthrough)
10. [All Concrete Actions Reference](#all-concrete-actions-reference)
11. [Creating Your Own Action](#creating-your-own-action)
12. [Key Takeaways](#key-takeaways)

---

## What is the Action System?

The Action System is a **generalized mechanism for characters to "do stuff" in a networked game**. This includes:

- **Combat abilities** (melee attacks, ranged spells, area effects)
- **Character states** (stunned, charging up, dashing)
- **Interactions** (picking up items, pulling levers, reviving allies)
- **Targeting** (selecting enemies, aiming projectiles)

**Key insight:** The system separates WHAT happens (game logic on server) from HOW it looks (visuals on client). This is the foundation of networked game architecture.

### Why Study This?

| Concept | What You'll Learn |
|---------|-------------------|
| Server Authority | Server validates and executes all actions |
| Client Prediction | Clients show feedback immediately for responsiveness |
| Data-Driven Design | Actions configured via ScriptableObjects, not code |
| Object Pooling | Actions are pooled and reused for performance |
| Partial Classes | Server/client code split into separate files |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ACTION SYSTEM OVERVIEW                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  CONFIGURATION LAYER                                                         │
│  ┌─────────────────┐                                                         │
│  │  ActionConfig   │ ← ScriptableObject data (damage, timing, animations)    │
│  └────────┬────────┘                                                         │
└───────────┼──────────────────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  ACTION LAYER                                                                │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐           │
│  │   Action.cs     │    │  MeleeAction.cs │    │ LaunchProjectile│           │
│  │   (abstract)    │◄───│  (concrete)     │    │    Action.cs    │           │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘           │
└──────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  EXECUTION LAYER                                                             │
│  ┌─────────────────────────┐         ┌─────────────────────────┐             │
│  │   ServerActionPlayer    │         │  ActionVisualization    │             │
│  │   (server-side queue)   │◄───────►│  (client-side visuals)  │             │
│  └─────────────────────────┘         └─────────────────────────┘             │
└──────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  FACTORY LAYER                                                               │
│  ┌─────────────────┐                                                         │
│  │  ActionFactory  │ ← Creates and pools Action instances                    │
│  └─────────────────┘                                                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Key Files

| File | Purpose | Lines |
|------|---------|-------|
| [Action.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/Action.cs) | Abstract base class with lifecycle methods | ~340 |
| [ActionConfig.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ActionConfig.cs) | Data structure for action configuration | ~110 |
| [ActionFactory.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ActionFactory.cs) | Creates/pools action instances | ~56 |
| [ServerActionPlayer.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ActionPlayers/ServerActionPlayer.cs) | Server-side action queue management | ~479 |
| [BlockingModeType.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/BlockingModeType.cs) | Enum for blocking behavior | ~12 |

---

## The Action Class Hierarchy

### Base Action Class

The `Action` class is a `ScriptableObject` that serves as the base for all actions. This is an interesting design choice—actions are assets that can be configured in the Unity Editor.

**File:** [Action.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/Action.cs)

```csharp
public abstract class Action : ScriptableObject
{
    // Identity
    public ActionID ActionID;           // Unique network identifier
    public ActionConfig Config;          // Configuration data
    
    // Timing
    public float TimeStarted { get; set; }
    public float TimeRunning => Time.time - TimeStarted;
    
    // Request data (passed when action is triggered)
    protected ActionRequestData m_Data;
    public ref ActionRequestData Data => ref m_Data;
}
```

### Server-Side Lifecycle Methods

These methods run ONLY on the server and control game logic:

```csharp
// Called when action starts playing (after queue wait)
// Return false to cancel immediately
public abstract bool OnStart(ServerCharacter serverCharacter);

// Called every frame while running
// Return true to continue, false to end
public abstract bool OnUpdate(ServerCharacter clientCharacter);

// Called when action wants to transition to background
public virtual bool ShouldBecomeNonBlocking();

// Called when action ends naturally
public virtual void End(ServerCharacter serverCharacter);

// Called when action is interrupted/canceled
public virtual void Cancel(ServerCharacter serverCharacter);

// Called after End() to chain into another action
public virtual bool ChainIntoNewAction(ref ActionRequestData newAction);
```

### Client-Side Lifecycle Methods

These methods run on ALL clients and control visuals:

```csharp
// Called when action starts on client
public virtual bool OnStartClient(ClientCharacter clientCharacter);

// Called every frame on client
public virtual bool OnUpdateClient(ClientCharacter clientCharacter);

// Called when action ends on client
public virtual void EndClient(ClientCharacter clientCharacter);

// Called when action is canceled on client
public virtual void CancelClient(ClientCharacter clientCharacter);

// Called IMMEDIATELY on owning client (before server confirms)
public virtual void AnticipateActionClient(ClientCharacter clientCharacter);

// Called when animation event fires
public virtual void OnAnimEventClient(ClientCharacter clientCharacter, string id);
```

### Lifecycle Diagram

```
                     ┌─── SERVER SIDE ────┐   ┌─── CLIENT SIDE ────┐
                     │                    │   │                    │
Player Input ────────┼───► OnStart() ─────┼───┼──► OnStartClient() │
                     │         │          │   │         │          │
                     │         ▼          │   │         ▼          │
                     │    OnUpdate() ◄────┼───┼───► OnUpdateClient()
                     │    (every frame)   │   │    (every frame)   │
                     │         │          │   │         │          │
                     │         ▼          │   │         ▼          │
                     │  ShouldBecomeNon   │   │                    │
                     │    Blocking()?     │   │                    │
                     │    ┌───┴───┐       │   │                    │
                     │   Yes     No       │   │                    │
                     │    │       │       │   │                    │
                     │    ▼       ▼       │   │                    │
                     │  [Move to  [Stay]  │   │                    │
                     │  background]       │   │                    │
                     │         │          │   │         │          │
                     │         ▼          │   │         ▼          │
                     │      End() ────────┼───┼───► EndClient()    │
                     │         │          │   │                    │
                     │         ▼          │   │                    │
                     │  ChainIntoNew      │   │                    │
                     │    Action()?       │   │                    │
                     └────────────────────┘   └────────────────────┘
```

---

## ActionConfig: Data-Driven Design

Every action has a `Config` object containing all its parameters. This is the **data-driven design** pattern—game designers can tweak values without touching code.

**File:** [ActionConfig.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ActionConfig.cs)

### Timing Fields

```csharp
[Tooltip("Duration in seconds that this Action takes to play")]
public float DurationSeconds;

[Tooltip("Time when the Action should do its 'main thing' (e.g. apply damage)")]
public float ExecTimeSeconds;

[Tooltip("How long the effect this Action leaves behind will last")]
public float EffectDurationSeconds;

[Tooltip("Cooldown - time before this action can be used again")]
public float ReuseTimeSeconds;
```

**Visual Timeline:**
```
Time:   0s          ExecTime        DurationSeconds
        |             |                    |
        ▼             ▼                    ▼
        ┌─────────────┬────────────────────┐
        │  WIND-UP    │     RECOVERY       │
        │  (animation │   (can't act yet)  │
        │   plays)    │                    │
        └─────────────┴────────────────────┘
                      ▲
                      │
              DAMAGE APPLIES HERE
```

### Combat Fields

```csharp
[Tooltip("Could be damage, healing, or other effects")]
public int Amount;

[Tooltip("How much mana it costs")]
public int ManaCost;

[Tooltip("How far the performer can be from target")]
public float Range;

[Tooltip("Radius of effect for area attacks")]
public float Radius;

[Tooltip("Damage to non-primary targets")]
public int SplashDamage;
```

### Animation Fields

```csharp
[Tooltip("Animation trigger for anticipation (before server confirms)")]
public string AnimAnticipation;

[Tooltip("Primary animation trigger")]
public string Anim;

[Tooltip("Auxiliary animation trigger (e.g. to end a loop)")]
public string Anim2;

[Tooltip("Reaction animation when target is hit")]
public string ReactAnim;
```

### Behavior Fields

```csharp
[Tooltip("Can this action be interrupted by movement or other actions?")]
public bool ActionInterruptible;

[Tooltip("Specific actions that can interrupt this one")]
public List<Action> IsInterruptableBy;

[Tooltip("How long this action blocks other actions")]
public BlockingModeType BlockingMode;

[Tooltip("Does this affect friendly or enemy targets?")]
public bool IsFriendly;
```

---

## Server-Side Execution (ServerActionPlayer)

The `ServerActionPlayer` is the brain of action execution. It manages a queue of actions, handles blocking/non-blocking logic, and enforces cooldowns.

**File:** [ServerActionPlayer.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ActionPlayers/ServerActionPlayer.cs)

### Core Data Structures

```csharp
public class ServerActionPlayer
{
    private List<Action> m_Queue;              // Blocking actions waiting to play
    private List<Action> m_NonBlockingActions; // Actions running in background
    private Dictionary<ActionID, float> m_LastUsedTimestamps; // Cooldown tracking
    
    private const float k_MaxQueueTimeDepth = 1.6f; // Max seconds of queued actions
}
```

### Action Queue Flow

```
Player presses ability button
            │
            ▼
    ┌───────────────────┐
    │  PlayAction()     │ ◄─── Entry point
    └─────────┬─────────┘
              │
              ▼
    ┌───────────────────┐     ┌─────────────────────┐
    │ Is queue blocking │ No  │ Add to queue        │
    │ action interrupt- ├────►│                     │
    │ ible by this?     │     └──────────┬──────────┘
    └─────────┬─────────┘                │
              │ Yes                      │
              ▼                          │
    ┌───────────────────┐                │
    │  ClearActions()   │                │
    │  (cancel current) │                │
    └─────────┬─────────┘                │
              │                          │
              ▼                          ▼
    ┌───────────────────────────────────────┐
    │          Is queue empty?              │
    │     ┌─────────┴─────────┐             │
    │    Yes                  No            │
    │     │                   │             │
    │     ▼                   ▼             │
    │  StartAction()      Wait...           │
    └───────────────────────────────────────┘
```

### StartAction() Deep Dive

This is where the magic happens. Let's trace through exactly what occurs:

```csharp
private void StartAction()
{
    // 1. Check cooldown (ReuseTimeSeconds)
    if (Time.time - lastTimeUsed < reuseTime)
    {
        AdvanceQueue(false);  // Skip this action, try next
        return;
    }
    
    // 2. Synthesize helper actions if needed
    SynthesizeTargetIfNecessary(0);  // Auto-target if needed
    SynthesizeChaseIfNecessary(0);   // Auto-move in range if needed
    
    // 3. Start the action
    m_Queue[0].TimeStarted = Time.time;
    bool play = m_Queue[0].OnStart(m_ServerCharacter);
    
    if (!play)
    {
        AdvanceQueue(false);  // Action declined to run
        return;
    }
    
    // 4. Stop movement if action is interruptible
    if (action.Config.ActionInterruptible && !forcedMovement)
    {
        m_Movement.CancelMove();
    }
    
    // 5. Record cooldown timestamp
    m_LastUsedTimestamps[action.ActionID] = Time.time;
    
    // 6. Handle instant non-blocking actions
    if (ExecTimeSeconds == 0 && BlockingMode == OnlyDuringExecTime)
    {
        m_NonBlockingActions.Add(m_Queue[0]);
        AdvanceQueue(false);
    }
}
```

### Synthesized Actions

The system automatically inserts helper actions when needed:

**Target Action:** If you use a targeted skill on an enemy that's not your current target, the system first changes your target.

**Chase Action:** If you use a ranged skill but are out of range, the system first moves you into range.

```csharp
private int SynthesizeChaseIfNecessary(int baseIndex)
{
    Action baseAction = m_Queue[baseIndex];
    
    if (baseAction.Data.ShouldClose && baseAction.Data.TargetIds != null)
    {
        // Insert a Chase action before the base action
        ActionRequestData data = new ActionRequestData
        {
            ActionID = GameDataSource.Instance.GeneralChaseActionPrototype.ActionID,
            TargetIds = baseAction.Data.TargetIds,
            Amount = baseAction.Config.Range  // Stop when in range
        };
        
        Action chaseAction = ActionFactory.CreateActionFromData(ref data);
        m_Queue.Insert(baseIndex, chaseAction);
        
        return baseIndex + 1;  // Base action moved down
    }
    return baseIndex;
}
```

### OnUpdate Loop

Every frame, the ServerActionPlayer updates all active actions:

```csharp
public void OnUpdate()
{
    // 1. Handle pending synthesized actions
    if (m_HasPendingSynthesizedAction)
    {
        PlayAction(ref m_PendingSynthesizedAction);
    }
    
    // 2. Check if blocking action should become non-blocking
    if (m_Queue.Count > 0 && m_Queue[0].ShouldBecomeNonBlocking())
    {
        m_NonBlockingActions.Add(m_Queue[0]);
        AdvanceQueue(false);
    }
    
    // 3. Update blocking action
    if (m_Queue.Count > 0)
    {
        if (!UpdateAction(m_Queue[0]))
        {
            AdvanceQueue(true);  // Action finished, call End()
        }
    }
    
    // 4. Update non-blocking actions (reverse order for safe removal)
    for (int i = m_NonBlockingActions.Count - 1; i >= 0; --i)
    {
        if (!UpdateAction(m_NonBlockingActions[i]))
        {
            m_NonBlockingActions[i].End(m_ServerCharacter);
            m_NonBlockingActions.RemoveAt(i);
        }
    }
}
```

---

## Client-Side Visualization

While the server handles logic, clients handle visuals independently. This separation is crucial for responsive gameplay.

### The Split Pattern

Each concrete action uses C# partial classes to split server and client code:

```
ConcreteActions/
├── MeleeAction.cs            ← Server logic (OnStart, OnUpdate)
├── MeleeAction.Client.cs     ← Client visuals (OnStartClient, AnticipateActionClient)
├── TrampleAction.cs
├── TrampleAction.Client.cs
└── ...
```

**Why split files?**
1. **Clarity:** Server logic is separate from visual logic
2. **Testing:** Can test server logic without needing visuals
3. **Debugging:** Easier to find which side has issues
4. **Platform-specific:** Could strip client code from server builds

---

## Blocking Modes Explained

Every action has a `BlockingMode` that determines how it interacts with other actions.

**File:** [BlockingModeType.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/BlockingModeType.cs)

```csharp
public enum BlockingModeType
{
    EntireDuration,      // Blocks queue for full DurationSeconds
    OnlyDuringExecTime,  // Blocks only until ExecTimeSeconds, then continues in background
}
```

### EntireDuration Mode

Used for actions where the character can't do anything else until finished.

```
Time: 0s                                          DurationSeconds
      |                                                  |
      ▼                                                  ▼
      ┌──────────────────────────────────────────────────┐
      │              BLOCKS ALL OTHER ACTIONS            │
      │                                                  │
      │   Player cannot queue new actions during this    │
      └──────────────────────────────────────────────────┘
```

**Examples:** Stunned, reviving ally, certain charge-up abilities

### OnlyDuringExecTime Mode

Used for actions that need a wind-up, then can run in background.

```
Time: 0s          ExecTimeSeconds           DurationSeconds
      |                 |                         |
      ▼                 ▼                         ▼
      ┌─────────────────┐─────────────────────────┐
      │   BLOCKS        │      NON-BLOCKING       │
      │   (wind-up)     │   (runs in background)  │
      └─────────────────┘─────────────────────────┘
                        ▲
                        │
              Player can queue next action
```

**Examples:** Projectile attacks (arrow flies while you can act again)

---

## Client Anticipation Pattern

This is one of the most important patterns for responsive networked games. Without it, every action would feel laggy due to network round-trip time.

### The Problem

```
Without Anticipation:
T+0ms:     Player presses button
T+0ms:     Send request to server
T+50ms:    Request reaches server
T+55ms:    Server validates and executes
T+55ms:    Server sends confirmation
T+105ms:   Confirmation reaches client
T+105ms:   Animation starts ← Player feels 105ms delay!
```

### The Solution

```
With Anticipation:
T+0ms:     Player presses button
T+0ms:     AnticipateActionClient() → Animation starts IMMEDIATELY
T+0ms:     Send request to server
T+50ms:    Request reaches server
T+55ms:    Server validates and executes
T+55ms:    Server sends confirmation
T+105ms:   Confirmation reaches client
T+105ms:   OnStartClient() → Already animating, looks smooth!
```

### How It Works

**File:** [Action.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/Action.cs) (lines 322-336)

```csharp
/// <summary>
/// Called when the action is being "anticipated" on the client.
/// Overriders should always call the base class!
/// </summary>
public virtual void AnticipateActionClient(ClientCharacter clientCharacter)
{
    AnticipatedClient = true;
    TimeStarted = UnityEngine.Time.time;

    // Play anticipation animation if specified
    if (!string.IsNullOrEmpty(Config.AnimAnticipation))
    {
        clientCharacter.OurAnimator.SetTrigger(Config.AnimAnticipation);
    }
}
```

**File:** [Action.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/Action.cs) (lines 219-224)

```csharp
public virtual bool OnStartClient(ClientCharacter clientCharacter)
{
    AnticipatedClient = false;  // No longer anticipated, now "real"
    TimeStarted = UnityEngine.Time.time;
    return true;
}
```

### Should We Anticipate?

Not all actions should be anticipated. The system checks several conditions:

**File:** [Action.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/Action.cs) (lines 255-277)

```csharp
public static bool ShouldClientAnticipate(ClientCharacter clientCharacter, ref ActionRequestData data)
{
    // Can this character perform actions at all?
    if (!clientCharacter.CanPerformActions) { return false; }
    
    // For targeted actions, check if target is in range
    // (Don't anticipate if we need to chase first)
    if (data.ShouldClose == true)
    {
        // Calculate distance to target
        if (distanceToTarget > range)
        {
            return false;  // Will need chase action first
        }
    }
    
    // Target action is special (handles its own anticipation)
    if (actionDescription.Logic == ActionLogic.Target)
    {
        return false;
    }
    
    return true;
}
```

---

## The Complete MeleeAction Walkthrough

Let's trace exactly what happens when a player clicks to perform a melee attack.

### Files Involved

| File | Role |
|------|------|
| [MeleeAction.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.cs) | Server logic |
| [MeleeAction.Client.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.Client.cs) | Client visuals |

### Step 1: Player Input (T+0ms)

Player clicks on an enemy to attack.

```
Client Input System → ActionRequestData created → Sent to server
                                               ↓
                               AnticipateActionClient() called
```

### Step 2: Client Anticipation

**File:** [Action.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/Action.cs) (lines 327-336)

```csharp
public virtual void AnticipateActionClient(ClientCharacter clientCharacter)
{
    AnticipatedClient = true;
    TimeStarted = UnityEngine.Time.time;
    
    // Play attack wind-up animation immediately
    if (!string.IsNullOrEmpty(Config.AnimAnticipation))
    {
        clientCharacter.OurAnimator.SetTrigger(Config.AnimAnticipation);
    }
}
```

### Step 3: Server Receives Request (T+50ms)

Server's `ServerActionPlayer.PlayAction()` is called.

```csharp
// Check if we can interrupt current action
if (!action.ShouldQueue && m_Queue.Count > 0)
{
    if (currentAction.Config.ActionInterruptible)
    {
        ClearActions(false);  // Cancel current
    }
}

// Create action and add to queue
var newAction = ActionFactory.CreateActionFromData(ref action);
m_Queue.Add(newAction);

// If queue was empty, start immediately
if (m_Queue.Count == 1) { StartAction(); }
```

### Step 4: MeleeAction.OnStart()

**File:** [MeleeAction.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.cs) (lines 37-56)

```csharp
public override bool OnStart(ServerCharacter serverCharacter)
{
    // Get target (from request or from character's current target)
    ulong target = (Data.TargetIds != null && Data.TargetIds.Length > 0) 
        ? Data.TargetIds[0] 
        : serverCharacter.TargetId.Value;
    
    // PRE-detect foe for client hint (provisional target)
    IDamageable foe = DetectFoe(serverCharacter, target);
    if (foe != null)
    {
        m_ProvisionalTarget = foe.NetworkObjectId;
        Data.TargetIds = new ulong[] { foe.NetworkObjectId };
    }
    
    // Snap character to face target direction
    if (Data.Direction != Vector3.zero)
    {
        serverCharacter.physicsWrapper.Transform.forward = Data.Direction;
    }
    
    // Play animation on server
    serverCharacter.serverAnimationHandler.NetworkAnimator.SetTrigger(Config.Anim);
    
    // Tell all clients to play the action
    serverCharacter.clientCharacter.ClientPlayActionRpc(Data);
    
    return true;  // Action started successfully
}
```

### Step 5: MeleeAction.OnUpdate() — Waiting for ExecTime

**File:** [MeleeAction.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.cs) (lines 67-80)

```csharp
public override bool OnUpdate(ServerCharacter clientCharacter)
{
    // Wait until execution time (when sword "connects")
    if (!m_ExecutionFired && (Time.time - TimeStarted) >= Config.ExecTimeSeconds)
    {
        m_ExecutionFired = true;
        
        // RE-detect foe at moment of impact (they might have moved!)
        var foe = DetectFoe(clientCharacter, m_ProvisionalTarget);
        if (foe != null)
        {
            // Apply damage!
            foe.ReceiveHitPoints(clientCharacter, -Config.Amount);
        }
    }
    
    return true;  // Keep running until DurationSeconds elapses
}
```

### Step 6: Hit Detection Logic

**Why detect twice?** The comment in the code explains this design decision beautifully:

```csharp
/// Q: Why do we DetectFoe twice, once in Start, once when we actually connect?
/// A: The weapon swing doesn't happen instantaneously. We want to broadcast
///    the action to other clients as fast as possible to minimize latency,
///    but this poses a conundrum. At the moment the swing starts, you don't
///    know for sure if you've hit anybody yet.
///
/// Possible solutions:
///   1. Detect once in Start → Unfair (can dodge but still get hit)
///   2. Detect once in Update, separate hit RPC → More bandwidth, complexity
///   3. Detect twice (Start for hint, Update for real) → Best balance ✓
```

**File:** [MeleeAction.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.cs) (lines 106-145)

```csharp
public static IDamageable GetIdealMeleeFoe(bool isNPC, Collider ourCollider, 
    float meleeRange, float meleeRadius, ulong preferredTargetNetworkId)
{
    // Cast sphere/box to find targets
    int numResults = radius > 0
        ? ActionUtils.DetectNearbyEntitiesUseSphere(...)
        : ActionUtils.DetectNearbyEntities(...);
    
    IDamageable foundFoe = null;
    int maxDamage = int.MinValue;
    
    for (int i = 0; i < numResults; i++)
    {
        var damageable = results[i].collider.GetComponent<IDamageable>();
        
        // Skip invalid targets
        if (damageable == null || !damageable.IsDamageable()) continue;
        
        // Prefer the hinted target (for animation sync)
        if (damageable.NetworkObjectId == preferredTargetNetworkId)
        {
            foundFoe = damageable;
            maxDamage = int.MaxValue;  // Always prefer this one
            continue;
        }
        
        // Otherwise pick highest-damage target
        var totalDamage = damageable.GetTotalDamage();
        if (foundFoe == null || maxDamage < totalDamage)
        {
            foundFoe = damageable;
            maxDamage = totalDamage;
        }
    }
    
    return foundFoe;
}
```

### Step 7: Client-Side Visuals

**File:** [MeleeAction.Client.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Action/ConcreteActions/MeleeAction.Client.cs)

```csharp
public override bool OnStartClient(ClientCharacter clientCharacter)
{
    base.OnStartClient(clientCharacter);
    
    // Spawn special effects on target if in range
    if (targetInRange)
    {
        m_SpawnedGraphics = InstantiateSpecialFXGraphics(targetTransform, true);
    }
    
    return true;
}

public override void OnAnimEventClient(ClientCharacter clientCharacter, string id)
{
    // Animation triggers "impact" event when sword connects
    if (id == "impact" && !m_ImpactPlayed)
    {
        PlayHitReact(clientCharacter);
    }
}

void PlayHitReact(ClientCharacter parent)
{
    // Play hit reaction animation on target
    if (targetInRange && target != null)
    {
        target.Animator.SetTrigger(Config.ReactAnim ?? "HitReact1");
    }
}
```

### Complete Timeline

```
T+0ms:     Player clicks attack
T+0ms:     AnticipateActionClient() → Attack animation starts
T+0ms:     RPC sent to server

T+50ms:    Server receives RPC
T+50ms:    MeleeAction.OnStart() → Provisional target detected
T+50ms:    ClientPlayActionRpc sent to all clients

T+100ms:   Other clients receive RPC
T+100ms:   OnStartClient() → VFX spawned on target

T+400ms:   ExecTimeSeconds reached
T+400ms:   MeleeAction.OnUpdate() → RE-detect target, apply damage
T+400ms:   Animation "impact" event → PlayHitReact()

T+1000ms:  DurationSeconds reached
T+1000ms:  Action ends, queue advances
```

---

## All Concrete Actions Reference

The project includes 15 concrete action implementations:

| Action | Description | Key Features |
|--------|-------------|--------------|
| **MeleeAction** | Close-range weapon swing | Double detection, hit react |
| **LaunchProjectileAction** | Spawns real NetworkObject projectile | Projectile syncs via transform |
| **FXProjectileTargetedAction** | Visual-only projectile, instant damage | Lower bandwidth than LaunchProjectile |
| **ChargedLaunchProjectileAction** | Hold to charge, release to fire | Variable power based on charge time |
| **ChargedShieldAction** | Hold to maintain shield | Blocks damage while held |
| **AOEAction** | Area-of-effect damage | Hits multiple targets in radius |
| **TrampleAction** | Charge forward, damaging enemies | Movement + damage combined |
| **DashAttackAction** | Quick dash with damage | Short distance movement |
| **ChaseAction** | Move toward target until in range | Auto-synthesized by system |
| **TargetAction** | Select an enemy as target | Auto-synthesized by system |
| **StunnedAction** | Character is stunned | Blocks all other actions |
| **ReviveAction** | Revive fallen ally | Channel, interruptible |
| **PickUpAction** | Pick up a world item | Interacts with objects |
| **DropAction** | Drop held item | Spawns object in world |
| **EmoteAction** | Play an emote animation | Pure visual, no gameplay effect |
| **StealthModeAction** | Toggle stealth visibility | Buff-style action |
| **TossAction** | Throw held object | Physics-based |

---

## Creating Your Own Action

Here's a template for creating a new action:

### Step 1: Create the Server Logic

```csharp
// MyNewAction.cs
using Unity.BossRoom.Gameplay.GameplayObjects.Character;
using UnityEngine;

namespace Unity.BossRoom.Gameplay.Actions
{
    [CreateAssetMenu(menuName = "BossRoom/Actions/My New Action")]
    public partial class MyNewAction : Action
    {
        private bool m_EffectApplied;
        
        public override bool OnStart(ServerCharacter serverCharacter)
        {
            // 1. Validate action can start
            if (!SomeCondition()) return false;
            
            // 2. Face target direction
            if (Data.Direction != Vector3.zero)
            {
                serverCharacter.physicsWrapper.Transform.forward = Data.Direction;
            }
            
            // 3. Play animation
            serverCharacter.serverAnimationHandler.NetworkAnimator.SetTrigger(Config.Anim);
            
            // 4. Tell clients
            serverCharacter.clientCharacter.ClientPlayActionRpc(Data);
            
            return true;
        }
        
        public override bool OnUpdate(ServerCharacter serverCharacter)
        {
            // Apply effect at execution time
            if (!m_EffectApplied && TimeRunning >= Config.ExecTimeSeconds)
            {
                m_EffectApplied = true;
                ApplyEffect(serverCharacter);
            }
            
            return true;  // Keep running until duration expires
        }
        
        public override void Reset()
        {
            base.Reset();
            m_EffectApplied = false;
        }
        
        private void ApplyEffect(ServerCharacter serverCharacter)
        {
            // Your effect logic here
        }
    }
}
```

### Step 2: Create the Client Logic

```csharp
// MyNewAction.Client.cs
using Unity.BossRoom.Gameplay.GameplayObjects.Character;

namespace Unity.BossRoom.Gameplay.Actions
{
    public partial class MyNewAction
    {
        public override bool OnStartClient(ClientCharacter clientCharacter)
        {
            base.OnStartClient(clientCharacter);
            
            // Spawn visual effects
            InstantiateSpecialFXGraphics(clientCharacter.transform, true);
            
            return true;
        }
        
        public override void AnticipateActionClient(ClientCharacter clientCharacter)
        {
            base.AnticipateActionClient(clientCharacter);
            
            // Start visual immediately for responsiveness
            // (Base class handles AnimAnticipation trigger)
        }
        
        public override void CancelClient(ClientCharacter clientCharacter)
        {
            // Clean up any visuals
        }
    }
}
```

### Step 3: Create ActionConfig in Unity

1. Right-click in Project → Create → BossRoom → Actions → My New Action
2. Configure the ActionConfig fields in Inspector
3. Assign to a character's action list

---

## Key Takeaways

### Design Patterns Used

| Pattern | Implementation |
|---------|----------------|
| **Factory** | `ActionFactory` creates and pools actions |
| **Object Pool** | Actions are reused via Unity's ObjectPool |
| **Command** | Actions encapsulate requests as objects |
| **State** | Actions have lifecycle states (start, update, end) |
| **Template Method** | Base class defines lifecycle, subclasses override |
| **Partial Class** | Server/client code split for clarity |

### Core Principles

1. **Server Authority:** Server validates and executes all damage/effects
2. **Client Prediction:** Clients anticipate for responsiveness
3. **Data-Driven:** ActionConfig allows designer tuning without code
4. **Separation of Concerns:** Logic separate from visuals
5. **Pool for Performance:** Actions are pooled, not created/destroyed

### When to Use This Pattern

✅ **Use Action System for:**
- Combat abilities with timing
- Interruptible actions
- Actions that need queuing
- Server-validated gameplay

❌ **Don't use for:**
- Simple instant effects
- UI-only feedback
- Purely cosmetic actions
- Client-side-only features

---

## Exercises

1. **Trace a Projectile:** Follow `LaunchProjectileAction` from button press to damage. How does the projectile sync across clients?

2. **Find the Buff System:** Look at `ChargedShieldAction`. How does it modify incoming damage using `BuffValue()`?

3. **Understand Chaining:** Find an action that uses `ChainIntoNewAction()`. What scenario triggers this?

4. **Create a Simple Action:** Make an "EmoteAction" that plays a wave animation with no gameplay effect.

---

> **Next:** [10_connection_state_machine.md](./10_connection_state_machine.md) — Deep dive into connection management
