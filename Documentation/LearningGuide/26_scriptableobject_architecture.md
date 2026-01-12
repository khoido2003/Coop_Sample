# 26: ScriptableObject Architecture

> **Purpose:** Understand how Boss Room uses ScriptableObjects for data-driven design, configuration, and runtime data management.

> 📍 **Key Files:**
> - [CharacterClass.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Configuration/CharacterClass.cs) — Character config
> - [GuidScriptableObject.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Infrastructure/ScriptableObjectArchitecture/GuidScriptableObject.cs) — Base class
> - [VisualizationConfiguration.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Configuration/VisualizationConfiguration.cs) — Visual settings
> - [NameGenerationData.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/Configuration/NameGenerationData.cs) — Data assets

---

## Table of Contents

1. [Why ScriptableObjects?](#why-scriptableobjects)
2. [Configuration Pattern](#configuration-pattern)
3. [Data Registry Pattern](#data-registry-pattern)
4. [GuidScriptableObject](#guidscriptableobject)
5. [Runtime Variables](#runtime-variables)
6. [Best Practices](#best-practices)
7. [Creating New SOs](#creating-new-sos)
8. [Key Takeaways](#key-takeaways)

---

## Why ScriptableObjects?

ScriptableObjects (SOs) provide data-driven design benefits:

```
┌──────────────────────────────────────────────────────────────────────┐
│           WITHOUT SCRIPTABLEOBJECTS (HARDCODED)                       │
└──────────────────────────────────────────────────────────────────────┘

public class Warrior
{
    public int MaxHP = 100;        // ❌ Hardcoded
    public float Speed = 5f;       // ❌ Can't change without code
    public string DisplayName;     // ❌ Designers can't edit
}

┌──────────────────────────────────────────────────────────────────────┐
│           WITH SCRIPTABLEOBJECTS (DATA-DRIVEN)                        │
└──────────────────────────────────────────────────────────────────────┘

public class Character
{
    public CharacterClass Config;  // ✅ All data in asset file
}

// CharacterClass.asset (editable in Inspector)
// - MaxHP: 100
// - Speed: 5
// - DisplayName: "Warrior"
```

### Benefits

| Benefit | Explanation |
|---------|-------------|
| **No code changes** | Designers edit assets, not code |
| **Version control** | Each config is a separate file |
| **Reusability** | Same config for multiple instances |
| **Memory efficiency** | Shared between all instances |
| **Hot reload** | Some changes apply without restart |

---

## Configuration Pattern

### CharacterClass Example

**File:** `Assets/Scripts/Gameplay/Configuration/CharacterClass.cs`

```csharp
[CreateAssetMenu(menuName = "GameData/CharacterClass", order = 1)]
public class CharacterClass : ScriptableObject
{
    [Tooltip("which character this data represents")]
    public CharacterTypeEnum CharacterType;

    [Tooltip("skill1 is usually the character's default attack")]
    public Action Skill1;

    [Tooltip("skill2 is usually the character's secondary attack")]
    public Action Skill2;

    [Tooltip("skill3 is usually the character's unique or special attack")]
    public Action Skill3;

    [Tooltip("Starting HP of this character class")]
    public IntVariable BaseHP;

    [Tooltip("Starting Mana of this character class")]
    public int BaseMana;

    [Tooltip("Base movement speed of this character class (in meters/sec)")]
    public float Speed;

    [Tooltip("Set to true if this represents an NPC, as opposed to a player.")]
    public bool IsNpc;

    [Tooltip("For NPCs, this will be used as the aggro radius")]
    public float DetectRange;

    [Tooltip("For players, this is the displayed class name")]
    public string DisplayedName;

    [Tooltip("For players, this is the class banner (when active)")]
    public Sprite ClassBannerLit;

    [Tooltip("For players, this is the class banner (when inactive)")]
    public Sprite ClassBannerUnlit;
}
```

### Usage

```csharp
public class ServerCharacter : NetworkBehaviour
{
    [SerializeField]
    CharacterClass m_CharacterClass;  // Assigned in Inspector

    void Start()
    {
        // Use config data
        m_CurrentHP = m_CharacterClass.BaseHP.Value;
        m_MovementSpeed = m_CharacterClass.Speed;
        
        // Check if NPC
        if (m_CharacterClass.IsNpc)
        {
            m_AIBrain = new AIBrain(this);
        }
    }
}
```

---

## Data Registry Pattern

### Centralized Asset Access

```csharp
public class GameDataSource : ScriptableObject
{
    [SerializeField] List<CharacterClass> m_CharacterClasses;
    [SerializeField] List<ActionPrototype> m_Actions;

    // Cached lookup dictionaries
    Dictionary<CharacterTypeEnum, CharacterClass> m_CharacterDict;
    Dictionary<ActionID, ActionPrototype> m_ActionDict;

    public void Initialize()
    {
        // Build lookup dictionaries once
        m_CharacterDict = m_CharacterClasses.ToDictionary(c => c.CharacterType);
        m_ActionDict = m_Actions.ToDictionary(a => a.ActionID);
    }

    public CharacterClass GetCharacterClass(CharacterTypeEnum type)
    {
        return m_CharacterDict[type];
    }

    public bool TryGetActionPrototypeByID(ActionID id, out ActionPrototype action)
    {
        return m_ActionDict.TryGetValue(id, out action);
    }
}
```

### Singleton Access

```csharp
public class GameDataSource : ScriptableObject
{
    static GameDataSource s_Instance;
    
    public static GameDataSource Instance
    {
        get
        {
            if (s_Instance == null)
            {
                s_Instance = Resources.Load<GameDataSource>("GameData/GameDataSource");
                s_Instance.Initialize();
            }
            return s_Instance;
        }
    }
}
```

---

## GuidScriptableObject

**File:** `Assets/Scripts/Infrastructure/ScriptableObjectArchitecture/GuidScriptableObject.cs`

A base class that gives each SO a unique identifier for network synchronization.

```csharp
[Serializable]
public abstract class GuidScriptableObject : ScriptableObject
{
    [HideInInspector]
    [SerializeField]
    byte[] m_Guid;

    public Guid Guid => new Guid(m_Guid);

    void OnValidate()
    {
        if (m_Guid.Length == 0)
        {
            // Auto-generate GUID on first use
            m_Guid = Guid.NewGuid().ToByteArray();
        }
    }
}
```

### Why GUIDs?

```
PROBLEM: How to sync "which action" across network?
─────────────────────────────────────────────────

Option 1: Send object reference
    ❌ Can't serialize ScriptableObject refs over network

Option 2: Send asset name
    ❌ Fragile, name changes break sync

Option 3: Send GUID ✅
    ✅ Stable identifier
    ✅ Survives renames
    ✅ Fast dictionary lookup
```

### Usage in Networking

```csharp
// Server sends action GUID
public struct ActionRequestData : INetworkSerializable
{
    public Guid ActionID;  // GUID from GuidScriptableObject
    
    public void NetworkSerialize<T>(BufferSerializer<T> serializer) where T : IReaderWriter
    {
        // GUIDs serialize as 16 bytes
        serializer.SerializeValue(ref ActionID);
    }
}

// Client looks up action by GUID
var action = GameDataSource.Instance.GetActionByGuid(requestData.ActionID);
```

---

## Runtime Variables

### IntVariable Pattern

```csharp
[CreateAssetMenu(menuName = "Variables/IntVariable")]
public class IntVariable : ScriptableObject
{
    [SerializeField] int m_Value;

    public int Value => m_Value;

    // Optional: Runtime override (reset on play)
    [NonSerialized] int? m_RuntimeValue;

    public int RuntimeValue
    {
        get => m_RuntimeValue ?? m_Value;
        set => m_RuntimeValue = value;
    }
}
```

### Benefits

```csharp
// In CharacterClass
public IntVariable BaseHP;  // Reference to shared asset

// Character1 and Character2 both reference "WarriorHP" asset
// Change asset once → all warriors updated
```

---

## Best Practices

### DO ✅

```csharp
// 1. Use CreateAssetMenu for easy creation
[CreateAssetMenu(menuName = "GameData/MyConfig", order = 1)]
public class MyConfig : ScriptableObject { }

// 2. Add tooltips for designers
[Tooltip("Damage dealt per hit")]
public int Damage;

// 3. Validate data in OnValidate
void OnValidate()
{
    Damage = Mathf.Max(0, Damage);  // Clamp to valid range
}

// 4. Use references to other SOs
public CharacterClass Character;  // Not CharacterTypeEnum

// 5. Cache lookups at startup
void Initialize()
{
    m_LookupDict = m_List.ToDictionary(x => x.Id);
}
```

### DON'T ❌

```csharp
// 1. Don't modify SO values at runtime (they persist!)
public void TakeDamage(int damage)
{
    m_Config.CurrentHP -= damage;  // ❌ Modifies asset!
}

// 2. Don't use Resources.Load frequently
void Update()
{
    var config = Resources.Load<MyConfig>("Config");  // ❌ Slow!
}

// 3. Don't hardcode data that could be configurable
public int MaxHP = 100;  // ❌ Put in SO instead
```

---

## Creating New SOs

### Step 1: Define the Class

```csharp
using UnityEngine;

[CreateAssetMenu(menuName = "GameData/WeaponConfig", fileName = "New Weapon")]
public class WeaponConfig : ScriptableObject
{
    [Header("Stats")]
    public int Damage;
    public float AttackSpeed;
    public float Range;

    [Header("Visuals")]
    public Sprite Icon;
    public GameObject Prefab;

    [Header("Audio")]
    public AudioClip AttackSound;
}
```

### Step 2: Create Asset

1. Right-click in Project → Create → GameData → WeaponConfig
2. Name it (e.g., "Sword_Iron")
3. Fill in values in Inspector

### Step 3: Reference in Code

```csharp
public class Weapon : MonoBehaviour
{
    [SerializeField] WeaponConfig m_Config;

    public void Attack()
    {
        // Use config values
        DealDamage(m_Config.Damage);
        PlaySound(m_Config.AttackSound);
    }
}
```

---

## Key Takeaways

| Concept | Description |
|---------|-------------|
| **Data-Driven** | Config in assets, not hardcoded |
| **CreateAssetMenu** | Easy asset creation for designers |
| **GuidScriptableObject** | Unique IDs for network sync |
| **Registry Pattern** | Centralized lookup with caching |
| **Runtime Variables** | Shared values across instances |
| **Don't Modify at Runtime** | SO changes persist to asset |
| **Tooltips** | Document fields for designers |

---

## Exercises

1. **Create a Weapon SO:** Define a `WeaponConfig` SO and create 3 different weapon assets.

2. **Add GUID Support:** Extend an existing SO to inherit from `GuidScriptableObject`.

3. **Build a Registry:** Create a `WeaponRegistry` that loads all weapon configs and provides lookup by ID.

---

> **Related Guides:**
> - [02_clean_code_patterns.md](./02_clean_code_patterns.md) — Data separation principles
> - [04_design_patterns.md](./04_design_patterns.md) — Registry and Factory patterns
> - [09_action_system_deepdive.md](./09_action_system_deepdive.md) — Actions as ScriptableObjects
