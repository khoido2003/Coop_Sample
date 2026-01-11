# 25: Level Gameplay Objects

> **Purpose:** Understand how Boss Room implements interactive level elements like portals, switches, doors, and breakable objects.

> 📍 **Key Files:**
> - [EnemyPortal.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/EnemyPortal.cs) — Enemy spawn point
> - [FloorSwitch.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/FloorSwitch.cs) — Pressure plate
> - [SwitchedDoor.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/SwitchedDoor.cs) — Door controlled by switch
> - [Breakable.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/Breakable.cs) — Destructible objects
> - [ServerWaveSpawner.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/ServerWaveSpawner.cs) — Wave spawning

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Enemy Portal](#enemy-portal)
3. [Floor Switch (Pressure Plate)](#floor-switch-pressure-plate)
4. [Switched Door](#switched-door)
5. [Breakable Objects](#breakable-objects)
6. [Common Patterns](#common-patterns)
7. [Creating New Level Objects](#creating-new-level-objects)
8. [Key Takeaways](#key-takeaways)

---

## Architecture Overview

Level objects follow the same server-authoritative pattern:

```
┌──────────────────────────────────────────────────────────────────────┐
│                   LEVEL OBJECT ARCHITECTURE                           │
└──────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────┐
                    │     NetworkBehaviour    │
                    │    (level object)       │
                    └───────────┬─────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            │                                       │
            ▼                                       ▼
    SERVER LOGIC                            CLIENT SYNC
    ┌───────────────┐                       ┌───────────────┐
    │ - Physics     │                       │ - Animation   │
    │ - Triggers    │──NetworkVariable───►  │ - Visual      │
    │ - State       │                       │ - Sound       │
    └───────────────┘                       └───────────────┘
```

---

## Enemy Portal

**File:** `Assets/Scripts/Gameplay/GameplayObjects/EnemyPortal.cs`

Spawns enemies when players are nearby. Has breakable elements that can disable it temporarily.

### Structure

```csharp
[RequireComponent(typeof(ServerWaveSpawner))]
public class EnemyPortal : NetworkBehaviour, ITargetable
{
    [SerializeField]
    public List<Breakable> m_BreakableElements;  // Crystal pillars

    [SerializeField]
    float m_DormantCooldown;  // Time to respawn after broken

    [SerializeField]
    Breakable m_PortalBreakable;  // Portal effect itself

    [SerializeField]
    ServerWaveSpawner m_WaveSpawner;

    public bool IsNpc => true;
    public bool IsValidTarget => !m_PortalBreakable.IsBroken;
}
```

### State Management

```csharp
void MaintainState()
{
    // Check if any breakable elements remain
    bool hasUnbrokenBreakables = false;
    foreach (var breakable in m_BreakableElements)
    {
        if (breakable && !breakable.IsBroken)
        {
            hasUnbrokenBreakables = true;
            break;
        }
    }

    // Update portal visual
    if (!hasUnbrokenBreakables)
        m_PortalBreakable.Break();
    else
        m_PortalBreakable.Unbreak();

    // Enable/disable spawning
    m_WaveSpawner.SetSpawnerEnabled(hasUnbrokenBreakables);

    // Start dormant timer when all broken
    if (!hasUnbrokenBreakables && m_CoroDormant == null)
    {
        m_CoroDormant = StartCoroutine(CoroGoDormantAndThenRestart());
    }
}
```

### Restart After Cooldown

```csharp
IEnumerator CoroGoDormantAndThenRestart()
{
    yield return new WaitForSeconds(m_DormantCooldown);
    Restart();
}

void Restart()
{
    // Repair all breakables
    foreach (var state in m_BreakableElements)
    {
        state.GetComponent<Breakable>().Unbreak();
    }

    m_PortalBreakable.Unbreak();
    m_WaveSpawner.SetSpawnerEnabled(true);
    m_CoroDormant = null;
}
```

---

## Floor Switch (Pressure Plate)

**File:** `Assets/Scripts/Gameplay/GameplayObjects/FloorSwitch.cs`

Activates when a player stands on it.

### Structure

```csharp
[RequireComponent(typeof(Collider))]
public class FloorSwitch : NetworkBehaviour
{
    [SerializeField] Animator m_Animator;
    [SerializeField] Collider m_Collider;

    public NetworkVariable<bool> IsSwitchedOn { get; } = new NetworkVariable<bool>();

    List<Collider> m_RelevantCollidersInTrigger = new List<Collider>();
}
```

### Server Logic

```csharp
void OnTriggerEnter(Collider other)
{
    m_RelevantCollidersInTrigger.Add(other);
}

void OnTriggerExit(Collider other)
{
    m_RelevantCollidersInTrigger.Remove(other);
}

void FixedUpdate()
{
    // Clean up destroyed colliders
    m_RelevantCollidersInTrigger.RemoveAll(col => col == null);
    
    // Update network state
    IsSwitchedOn.Value = m_RelevantCollidersInTrigger.Count > 0;
}
```

### Client Sync

```csharp
public override void OnNetworkSpawn()
{
    if (!IsServer) enabled = false;

    // Initial state
    FloorSwitchStateChanged(false, IsSwitchedOn.Value);
    
    // Subscribe to changes
    IsSwitchedOn.OnValueChanged += FloorSwitchStateChanged;
}

void FloorSwitchStateChanged(bool previousValue, bool newValue)
{
    m_Animator.SetBool(m_AnimatorPressedDownBoolVarID, newValue);
}
```

---

## Switched Door

**File:** `Assets/Scripts/Gameplay/GameplayObjects/SwitchedDoor.cs`

Opens when connected switches are activated.

### Structure

```csharp
public class SwitchedDoor : NetworkBehaviour
{
    [SerializeField] List<FloorSwitch> m_Switches;
    [SerializeField] Animator m_Animator;
    [SerializeField] Collider m_Collider;

    public NetworkVariable<bool> IsOpen { get; } = new NetworkVariable<bool>();
}
```

### Logic

```csharp
void FixedUpdate()
{
    // Check if ALL switches are on
    bool allSwitchesOn = true;
    foreach (var sw in m_Switches)
    {
        if (!sw.IsSwitchedOn.Value)
        {
            allSwitchesOn = false;
            break;
        }
    }
    
    IsOpen.Value = allSwitchesOn;
}

void OnIsOpenChanged(bool wasOpen, bool isOpen)
{
    m_Animator.SetBool("IsOpen", isOpen);
    m_Collider.enabled = !isOpen;  // Disable collision when open
}
```

---

## Breakable Objects

**File:** `Assets/Scripts/Gameplay/GameplayObjects/Breakable.cs`

Objects that can be destroyed by player attacks.

### Structure

```csharp
public class Breakable : NetworkBehaviour, IDamageable
{
    [SerializeField] int m_MaxHealth;
    [SerializeField] GameObject m_IntactVisual;
    [SerializeField] GameObject m_BrokenVisual;
    [SerializeField] ParticleSystem m_BreakParticles;

    public NetworkVariable<bool> IsBrokenNetworkVariable { get; } = new();
    
    public event Action Broken;
    public bool IsBroken => IsBrokenNetworkVariable.Value;
    
    int m_CurrentHealth;
}
```

### Damage Reception (IDamageable)

```csharp
public void ReceiveHP(ServerCharacter inflicter, int HP)
{
    if (IsBroken) return;  // Already broken

    m_CurrentHealth += HP;  // HP is negative for damage

    if (m_CurrentHealth <= 0)
    {
        Break();
    }
}

public void Break()
{
    if (!IsBroken)
    {
        IsBrokenNetworkVariable.Value = true;
        Broken?.Invoke();  // Notify listeners (like EnemyPortal)
    }
}

public void Unbreak()
{
    m_CurrentHealth = m_MaxHealth;
    IsBrokenNetworkVariable.Value = false;
}
```

### Client Visuals

```csharp
void OnBrokenStateChanged(bool wasBroken, bool isBroken)
{
    m_IntactVisual.SetActive(!isBroken);
    m_BrokenVisual.SetActive(isBroken);
    
    if (isBroken && m_BreakParticles != null)
    {
        m_BreakParticles.Play();
    }
}
```

---

## Common Patterns

### 1. Server-Only Logic

```csharp
public override void OnNetworkSpawn()
{
    if (!IsServer)
    {
        enabled = false;  // Disable Update on clients
        return;
    }
    // Server-only initialization
}
```

### 2. NetworkVariable for State Sync

```csharp
public NetworkVariable<bool> SomeState { get; } = new();

// Server modifies
SomeState.Value = true;

// Client reacts
SomeState.OnValueChanged += OnStateChanged;
```

### 3. Trigger-Based Interaction

```csharp
void OnTriggerEnter(Collider other)
{
    // Only server processes
    if (!IsServer) return;
    
    // Do something
}
```

### 4. Event-Based Composition

```csharp
// Breakable raises event
public event Action Broken;

// Portal subscribes
breakable.Broken += OnBreakableBroken;
```

---

## Creating New Level Objects

### Step 1: Create NetworkBehaviour

```csharp
public class MyLevelObject : NetworkBehaviour
{
    // NetworkVariable for synced state
    public NetworkVariable<bool> IsActive { get; } = new();

    public override void OnNetworkSpawn()
    {
        if (!IsServer)
        {
            enabled = false;
        }
        
        IsActive.OnValueChanged += OnActiveChanged;
    }
    
    void OnActiveChanged(bool was, bool now)
    {
        // Update visuals (runs on all clients)
    }
}
```

### Step 2: Add Collider if Needed

```csharp
[RequireComponent(typeof(Collider))]
public class MyTrigger : NetworkBehaviour
{
    void Awake()
    {
        GetComponent<Collider>().isTrigger = true;
    }
    
    void OnTriggerEnter(Collider other)
    {
        if (!IsServer) return;
        // Handle on server
    }
}
```

### Step 3: Add Visual Components

```csharp
[SerializeField] Animator m_Animator;
[SerializeField] ParticleSystem m_Particles;

void OnActiveChanged(bool was, bool now)
{
    m_Animator.SetBool("Active", now);
    if (now) m_Particles.Play();
}
```

### Step 4: Compose with Other Objects

```csharp
// Like EnemyPortal + ServerWaveSpawner + Breakable
[RequireComponent(typeof(SomeOtherComponent))]
public class MyComposedObject : NetworkBehaviour
{
    [SerializeField] SomeOtherComponent m_OtherComponent;
    
    // Coordinate between components
}
```

---

## Key Takeaways

| Concept | Description |
|---------|-------------|
| **Server-Only Logic** | Disable MonoBehaviour on clients |
| **NetworkVariable** | Sync state from server to clients |
| **Physics on Server** | Triggers only processed on server |
| **Event Composition** | Objects communicate via C# events |
| **Visual Updates** | Subscribe to OnValueChanged |
| **ITargetable** | Makes object targetable by players |
| **IDamageable** | Makes object take damage |

### Object Relationship

```
┌─────────────────────────────────────────────────────────────────┐
│                    LEVEL OBJECT RELATIONSHIPS                    │
└─────────────────────────────────────────────────────────────────┘

  FloorSwitch ─────────────► SwitchedDoor
      │                           ▲
      │ IsSwitchedOn.Value       │ Opens when all switches on
      ▼                           │
  Player stands on           Checks switches
      
      
  Breakable ─────────────► EnemyPortal
      │                           │
      │ Broken event              │
      ▼                           ▼
  Fires when HP=0         Disables spawner,
                          starts cooldown
```

---

> **Related Guides:**
> - [21_ai_enemy_system.md](./21_ai_enemy_system.md) — Wave spawner and AI
> - [03_networking_essentials.md](./03_networking_essentials.md) — NetworkVariables
> - [04_design_patterns.md](./04_design_patterns.md) — Observer pattern (events)
