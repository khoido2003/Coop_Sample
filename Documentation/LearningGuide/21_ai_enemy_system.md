# 21: AI & Enemy System Deep Dive

> **Purpose:** Understand how Boss Room implements enemy AI using state machines, the same Action system as players, and how enemies are spawned via wave spawners.

> 📍 **Key Files:**
> - [AIBrain.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/AI/AIBrain.cs) — AI decision-making
> - [AIState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/AI/AIState.cs) — Base state class
> - [AttackAIState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Character/AI/AttackAIState.cs) — Attack behavior
> - [ServerWaveSpawner.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/ServerWaveSpawner.cs) — Wave spawning

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [AIBrain Deep Dive](#aibrain-deep-dive)
3. [AI State Machine](#ai-state-machine)
4. [Attack State Logic](#attack-state-logic)
5. [Enemy Configuration](#enemy-configuration)
6. [Wave Spawner System](#wave-spawner-system)
7. [Creating New Enemies](#creating-new-enemies)
8. [Key Takeaways](#key-takeaways)

---

## Architecture Overview

Boss Room's AI system demonstrates a key architectural insight:

> **Enemies use the SAME systems as players!**

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SHARED ARCHITECTURE                                │
└──────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────┐
                    │     ServerCharacter     │
                    │   (shared by all)       │
                    └───────────┬─────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            │                                       │
            ▼                                       ▼
    ┌───────────────┐                       ┌───────────────┐
    │    PLAYER     │                       │    ENEMY      │
    │               │                       │               │
    │ ClientInput   │                       │   AIBrain     │ ← Replaces input
    │ Sender        │                       │               │
    └───────┬───────┘                       └───────┬───────┘
            │                                       │
            └───────────────────┬───────────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │   ServerActionPlayer    │
                    │   (same for all!)       │
                    └─────────────────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │     Action System       │
                    │   (MeleeAction, etc.)   │
                    └─────────────────────────┘
```

**Benefits:**
- All action logic works for both players and enemies
- Abilities, cooldowns, damage work identically
- Only the "brain" (input vs AI) differs

---

## AIBrain Deep Dive

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/AI/AIBrain.cs`

### Class Structure

```csharp
public class AIBrain
{
    private enum AIStateType
    {
        ATTACK,
        // WANDER, (not yet implemented)
        IDLE,
    }

    private ServerCharacter m_ServerCharacter;
    private ServerActionPlayer m_ServerActionPlayer;
    private AIStateType m_CurrentState;
    private Dictionary<AIStateType, AIState> m_Logics;
    private List<ServerCharacter> m_HatedEnemies;  // Aggro list
    private float m_DetectRangeOverride = -1;      // Spawner can override
}
```

### Initialization

```csharp
public AIBrain(ServerCharacter me, ServerActionPlayer myServerActionPlayer)
{
    m_ServerCharacter = me;
    m_ServerActionPlayer = myServerActionPlayer;

    // Create all state instances (no allocation during gameplay)
    m_Logics = new Dictionary<AIStateType, AIState>
    {
        [AIStateType.IDLE] = new IdleAIState(this),
        [AIStateType.ATTACK] = new AttackAIState(this, m_ServerActionPlayer),
    };
    
    m_HatedEnemies = new List<ServerCharacter>();
    m_CurrentState = AIStateType.IDLE;
}
```

### Update Loop

```csharp
public void Update()
{
    // 1. Find best available state
    AIStateType newState = FindBestEligibleAIState();
    
    // 2. On state change, initialize new state
    if (m_CurrentState != newState)
    {
        m_Logics[newState].Initialize();
    }
    
    // 3. Update current state
    m_CurrentState = newState;
    m_Logics[m_CurrentState].Update();
}
```

### Hate (Aggro) System

```csharp
// Called when enemy takes damage
public void ReceiveHP(ServerCharacter inflicter, int amount)
{
    if (inflicter != null && amount < 0)
    {
        Hate(inflicter);  // Add attacker to hate list
    }
}

public void Hate(ServerCharacter character)
{
    if (!m_HatedEnemies.Contains(character))
    {
        m_HatedEnemies.Add(character);
    }
}

public List<ServerCharacter> GetHatedEnemies()
{
    // Clean list - remove dead/null enemies
    for (int i = m_HatedEnemies.Count - 1; i >= 0; i--)
    {
        if (!IsAppropriateFoe(m_HatedEnemies[i]))
        {
            m_HatedEnemies.RemoveAt(i);
        }
    }
    return m_HatedEnemies;
}
```

### Target Validation

```csharp
public bool IsAppropriateFoe(ServerCharacter potentialFoe)
{
    if (potentialFoe == null ||
        potentialFoe.IsNpc ||              // Don't attack other NPCs
        potentialFoe.LifeState != LifeState.Alive ||
        potentialFoe.IsStealthy.Value)     // Can't see stealthy players
    {
        return false;
    }
    return true;
}
```

---

## AI State Machine

### State Priority

```csharp
private AIStateType FindBestEligibleAIState()
{
    // States are checked in order - first eligible wins
    foreach (AIStateType aiStateType in k_AIStates)
    {
        if (m_Logics[aiStateType].IsEligible())
        {
            return aiStateType;
        }
    }
    return AIStateType.IDLE;  // Fallback
}
```

**State order:** ATTACK → IDLE

So if attack is eligible (has a foe), it takes priority.

### Base AIState Class

```csharp
public abstract class AIState
{
    /// <summary>
    /// Returns true if this state could be entered right now
    /// </summary>
    public abstract bool IsEligible();

    /// <summary>
    /// Called when entering this state
    /// </summary>
    public abstract void Initialize();

    /// <summary>
    /// Called every Update while in this state
    /// </summary>
    public abstract void Update();
}
```

### IdleAIState

```csharp
public class IdleAIState : AIState
{
    private AIBrain m_Brain;

    public override bool IsEligible()
    {
        // Idle is always eligible (fallback state)
        return true;
    }

    public override void Initialize()
    {
        // Nothing special to do
    }

    public override void Update()
    {
        // Check for players in detection range
        foreach (var character in PlayerServerCharacter.GetPlayerServerCharacters())
        {
            if (m_Brain.IsAppropriateFoe(character))
            {
                float distance = Vector3.Distance(
                    m_Brain.GetMyServerCharacter().Position,
                    character.Position);
                    
                if (distance <= m_Brain.DetectRange)
                {
                    m_Brain.Hate(character);  // Player detected!
                }
            }
        }
    }
}
```

---

## Attack State Logic

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Character/AI/AttackAIState.cs`

### Eligibility

```csharp
public override bool IsEligible()
{
    // Eligible if we have a foe OR can find one
    return m_Foe != null || ChooseFoe() != null;
}
```

### Initialization

```csharp
public override void Initialize()
{
    // Gather available attacks from CharacterClass
    m_AttackActions = new List<Action>();
    if (m_Brain.CharacterData.Skill1 != null)
        m_AttackActions.Add(m_Brain.CharacterData.Skill1);
    if (m_Brain.CharacterData.Skill2 != null)
        m_AttackActions.Add(m_Brain.CharacterData.Skill2);
    if (m_Brain.CharacterData.Skill3 != null)
        m_AttackActions.Add(m_Brain.CharacterData.Skill3);

    // Pick random starting attack
    m_CurAttackAction = m_AttackActions[Random.Range(0, m_AttackActions.Count)];
    m_Foe = null;  // Will choose in Update
}
```

### Update Logic

```csharp
public override void Update()
{
    // 1. Validate current foe
    if (!m_Brain.IsAppropriateFoe(m_Foe))
    {
        m_Foe = ChooseFoe();
        m_ServerActionPlayer.ClearActions(true);  // Cancel current actions
    }

    // 2. No foe? State becomes ineligible
    if (!m_Foe)
        return;

    // 3. Already attacking this foe? Don't spam
    if (m_ServerActionPlayer.GetActiveActionInfo(out var info))
    {
        // Check if chasing or attacking current foe
        if (IsAttackingOrChasingFoe(info))
            return;
            
        // Check if stunned
        if (IsStunned(info))
            return;
    }

    // 4. Choose attack (respects cooldowns)
    m_CurAttackAction = ChooseAttack();
    if (m_CurAttackAction == null)
        return;  // All attacks on cooldown

    // 5. ATTACK!
    var attackData = new ActionRequestData
    {
        ActionID = m_CurAttackAction.ActionID,
        TargetIds = new ulong[] { m_Foe.NetworkObjectId },
        ShouldClose = true,  // Auto-generate chase action if out of range
        Direction = m_Brain.GetMyServerCharacter().physicsWrapper.Transform.forward
    };
    m_ServerActionPlayer.PlayAction(ref attackData);
}
```

### Target Selection

```csharp
private ServerCharacter ChooseFoe()
{
    // Choose closest enemy from hate list
    Vector3 myPosition = m_Brain.GetMyServerCharacter().physicsWrapper.Transform.position;
    
    float closestDistanceSqr = int.MaxValue;
    ServerCharacter closestFoe = null;
    
    foreach (var foe in m_Brain.GetHatedEnemies())
    {
        float distanceSqr = (myPosition - foe.physicsWrapper.Transform.position).sqrMagnitude;
        if (distanceSqr < closestDistanceSqr)
        {
            closestDistanceSqr = distanceSqr;
            closestFoe = foe;
        }
    }
    return closestFoe;
}
```

### Attack Selection (With Cooldowns)

```csharp
private Action ChooseAttack()
{
    // Random selection among available attacks
    int idx = Random.Range(0, m_AttackActions.Count);
    
    bool anyUsable;
    do
    {
        anyUsable = false;
        foreach (var attack in m_AttackActions)
        {
            // Check cooldown
            if (m_ServerActionPlayer.IsReuseTimeElapsed(attack.ActionID))
            {
                anyUsable = true;
                if (idx == 0)
                    return attack;
                --idx;
            }
        }
    } while (anyUsable);

    return null;  // All actions on cooldown
}
```

---

## Enemy Configuration

Enemies are configured like players via **CharacterClass** ScriptableObjects:

```csharp
// CharacterClass.cs (simplified)
public class CharacterClass : ScriptableObject
{
    // Stats
    public IntVariable BaseHP;
    public float DetectRange;        // How far AI can see
    
    // Abilities (used by AI)
    public ActionPrototype Skill1;   // Primary attack
    public ActionPrototype Skill2;   // Secondary attack
    public ActionPrototype Skill3;   // Special ability
    
    // Visual
    public GameObject CharacterPrefab;
}
```

**Location:** `Assets/GameData/Character/` contains enemy CharacterClass assets.

---

## Wave Spawner System

**File:** `Assets/Scripts/Gameplay/GameplayObjects/ServerWaveSpawner.cs`

### Configuration Options

```csharp
[Header("Wave parameters")]
[SerializeField] int m_NumberOfWaves = 2;
[SerializeField] int m_SpawnsPerWave = 2;
[SerializeField] float m_TimeBetweenSpawns = 0.5f;
[SerializeField] float m_TimeBetweenWaves = 5;
[SerializeField] float m_RestartDelay = 10;
[SerializeField] float m_ProximityDistance = 30;

[Header("Spawn Cap")]
[SerializeField] int m_MinSpawnCap = 2;
[SerializeField] int m_MaxSpawnCap = 10;
[SerializeField] float m_SpawnCapIncreasePerPlayer = 1;
```

### Spawn Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                     WAVE SPAWNER FLOW                                 │
└──────────────────────────────────────────────────────────────────────┘

OnNetworkSpawn() (server only)
         │
         ▼
StartWaveSpawning()
         │
         ▼
TriggerSpawnWhenPlayersNear()  ← Coroutine
         │
         ├─► Every 2 seconds: IsAnyPlayerNearbyAndVisible()?
         │   │
         │   ├─► NO: Keep waiting
         │   │
         │   └─► YES: Start SpawnWaves() coroutine
         │
         ▼
SpawnWaves()
         │
         ├─► Wave 1: SpawnWave() → spawn N enemies
         ├─► Wait TimeBetweenWaves
         ├─► Wave 2: SpawnWave() → spawn N enemies
         ├─► ...
         ├─► Last wave complete
         └─► Wait RestartDelay → loop back
```

### Player Detection

```csharp
bool IsAnyPlayerNearbyAndVisible()
{
    foreach (var serverCharacter in PlayerServerCharacter.GetPlayerServerCharacters())
    {
        // Skip stealthy players if configured
        if (!m_DetectStealthyPlayers && serverCharacter.IsStealthy.Value)
            continue;

        var direction = playerPosition - spawnerPosition;
        
        // Distance check
        if (direction.sqrMagnitude > squaredProximityDistance)
            continue;

        // Line of sight check (raycast)
        var hit = Physics.RaycastNonAlloc(ray, m_Hit, distance, m_BlockingMask);
        if (hit == 0)  // No obstacles
            return true;
    }
    return false;
}
```

### Dynamic Spawn Cap

```csharp
int GetCurrentSpawnCap()
{
    int numPlayers = 0;
    foreach (var serverCharacter in PlayerServerCharacter.GetPlayerServerCharacters())
    {
        if (serverCharacter.NetLifeState.LifeState.Value == LifeState.Alive)
            ++numPlayers;
    }

    // Scale with player count
    return Mathf.CeilToInt(
        Mathf.Min(
            m_MinSpawnCap + (numPlayers * m_SpawnCapIncreasePerPlayer), 
            m_MaxSpawnCap));
}
```

### Spawn with Detection Override

```csharp
NetworkObject SpawnPrefab()
{
    int posIdx = m_SpawnedCount++ % m_SpawnPositions.Count;
    var clone = Instantiate(m_NetworkedPrefab, 
        m_SpawnPositions[posIdx].position, 
        m_SpawnPositions[posIdx].rotation);
        
    clone.Spawn(true);  // Network spawn
    
    // Override detection range if configured
    if (m_SpawnedEntityDetectDistance > -1)
    {
        var serverChar = clone.GetComponent<ServerCharacter>();
        if (serverChar?.AIBrain != null)
        {
            serverChar.AIBrain.DetectRange = m_SpawnedEntityDetectDistance;
        }
    }

    return clone;
}
```

---

## Creating New Enemies

### Step 1: Create CharacterClass

1. Right-click in Project → Create → Boss Room → Character Class
2. Set:
   - BaseHP
   - DetectRange (how far AI can see)
   - Skill1, Skill2, Skill3 (ActionPrototypes)

### Step 2: Create Enemy Prefab

1. Create prefab with:
   - `ServerCharacter` component
   - `ClientCharacter` component
   - Animator
   - Collider
   
2. Set `IsNpc = true` in ServerCharacter
3. Assign CharacterClass reference

### Step 3: Create Action Configs (optional)

If enemy needs unique abilities:
1. Create ActionConfig ScriptableObjects
2. Create Action classes if needed
3. Reference in CharacterClass.Skill1/2/3

### Step 4: Setup Wave Spawner

1. Create empty GameObject
2. Add `ServerWaveSpawner` component
3. Configure:
   - NetworkedPrefab = your enemy prefab
   - SpawnPositions = spawn point transforms
   - Wave parameters
   - Detection settings

---

## Key Takeaways

| Concept | Explanation |
|---------|-------------|
| **Shared Systems** | Enemies use same ServerActionPlayer and Actions as players |
| **AIBrain** | Replaces player input, runs on server only |
| **State Machine** | ATTACK and IDLE states, expandable |
| **Hate List** | Tracks who attacked the enemy (aggro) |
| **Target Selection** | Closest valid target from hate list |
| **Attack Selection** | Random from available skills, respects cooldowns |
| **Wave Spawner** | Proximity-triggered, player-scaled, with line of sight |

### Design Patterns Used

- **State Machine** - AI states with IsEligible/Initialize/Update
- **Shared Components** - Actions work for both players and AI
- **Data-Driven** - CharacterClass defines enemy abilities
- **Dynamic Scaling** - Spawn cap adjusts to player count

---

## Exercises

1. **Trace Boss AI:** Find the boss prefab and its CharacterClass. What skills does it have?

2. **Add a State:** Create a `FleeAIState` that makes the enemy run away when HP < 20%.

3. **Modify Target Selection:** Change `ChooseFoe()` to prefer the player who dealt the most damage.

---

> **Related Guides:**
> - [09_action_system_deepdive.md](./09_action_system_deepdive.md) — How actions work
> - [18_character_system_deepdive.md](./18_character_system_deepdive.md) — ServerCharacter/ClientCharacter
> - [25_level_gameplay_objects.md](./25_level_gameplay_objects.md) — Interactive objects and portals
