# 29: Unity-Specific Clean Code Patterns

> **Purpose:** Learn Unity-specific clean code practices including assembly definitions, namespaces, editor/runtime separation, and project organization.

> 📍 **Key Files:**
> - `Assets/Scripts/*/Unity.BossRoom.*.asmdef` — Assembly definitions
> - [ApplicationController.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ApplicationLifecycle/ApplicationController.cs) — Namespace example
> - Various `Editor/` subdirectories — Editor-only code

---

## Table of Contents

1. [Project Organization](#project-organization)
2. [Assembly Definitions](#assembly-definitions)
3. [Namespace Conventions](#namespace-conventions)
4. [Editor vs Runtime Code](#editor-vs-runtime-code)
5. [MonoBehaviour Best Practices](#monobehaviour-best-practices)
6. [Serialization Patterns](#serialization-patterns)
7. [Unity API Patterns](#unity-api-patterns)
8. [Key Takeaways](#key-takeaways)

---

## Project Organization

Boss Room uses feature-based folder organization:

```
Assets/Scripts/
├── ApplicationLifecycle/     # App startup, DI container
│   └── Messages/             # Lifecycle events
├── Audio/                    # Audio management
├── CameraUtils/              # Camera systems
├── ConnectionManagement/     # Network connection
│   └── ConnectionState/      # State machine states
├── Editor/                   # Editor-only tools
├── Gameplay/                 # Core game logic
│   ├── Actions/              # Ability system
│   ├── Configuration/        # ScriptableObjects
│   ├── GameplayObjects/      # Characters, enemies
│   ├── GameState/            # Game flow states
│   ├── UI/                   # UI components
│   └── UserInput/            # Input handling
├── Infrastructure/           # Core utilities
│   ├── PubSub/               # Messaging
│   └── ScriptableObjectArchitecture/
├── Navigation/               # Pathfinding
├── UnityServices/            # UGS integration
│   └── Sessions/             # Lobby/session
├── Utils/                    # Helpers
└── VisualEffects/            # VFX system
```

### Organization Principles

| Principle | Example |
|-----------|---------|
| **Feature-based** | `ConnectionManagement/` contains all connection code |
| **Cohesive modules** | Related code stays together |
| **Clear dependencies** | Infrastructure has no gameplay deps |
| **Editor separation** | `Editor/` folders for editor-only |

---

## Assembly Definitions

Boss Room uses **14 assembly definitions** for:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ASSEMBLY DEFINITION BENEFITS                       │
└──────────────────────────────────────────────────────────────────────┘

1. FASTER COMPILATION
   ─────────────────
   Change in Audio/ → Only recompile Unity.BossRoom.Audio
   Not the entire project!

2. EXPLICIT DEPENDENCIES
   ──────────────────────
   Gameplay → can reference → Infrastructure
   Infrastructure → cannot reference → Gameplay
   
3. CLEAR BOUNDARIES
   ─────────────────
   Forces clean architecture
   No accidental cross-references

4. PLATFORM FILTERING
   ───────────────────
   Editor assemblies → only in editor
   Don't ship editor code in builds
```

### Assembly Structure

```
Unity.BossRoom.Gameplay.asmdef
├── References:
│   ├── Unity.BossRoom.Infrastructure
│   ├── Unity.BossRoom.ConnectionManagement
│   ├── Unity.Netcode.Runtime
│   └── VContainer
└── Defines: BOSS_ROOM_GAMEPLAY

Unity.BossRoom.Infrastructure.asmdef
├── References:
│   └── (minimal - core utilities only)
└── Defines: BOSS_ROOM_INFRASTRUCTURE

Unity.BossRoom.Editor.asmdef
├── References:
│   └── Unity.BossRoom.Gameplay
├── Include Platforms: Editor only
└── Defines: BOSS_ROOM_EDITOR
```

### Creating Assembly Definitions

1. Right-click folder → Create → Assembly Definition
2. Name it: `CompanyName.ProjectName.Feature.asmdef`
3. Add references to dependencies
4. Set platform filtering if needed

---

## Namespace Conventions

Boss Room uses consistent namespace hierarchy:

```csharp
// Root namespace
namespace Unity.BossRoom { }

// Feature namespaces
namespace Unity.BossRoom.Gameplay { }
namespace Unity.BossRoom.Gameplay.Actions { }
namespace Unity.BossRoom.Gameplay.GameplayObjects { }
namespace Unity.BossRoom.Gameplay.GameplayObjects.Character { }

// Infrastructure
namespace Unity.BossRoom.Infrastructure { }

// Connection
namespace Unity.BossRoom.ConnectionManagement { }
```

### Naming Rules

```csharp
// ✅ GOOD: Matches folder structure
// File: Assets/Scripts/Gameplay/Actions/MeleeAction.cs
namespace Unity.BossRoom.Gameplay.Actions
{
    public class MeleeAction : Action { }
}

// ❌ BAD: Doesn't match folder
namespace BossRoom.Combat  // Wrong hierarchy
{
    public class MeleeAction { }
}
```

---

## Editor vs Runtime Code

### Folder-Based Separation

```
Assets/Scripts/
├── Gameplay/
│   ├── RuntimCode.cs           # Included in build
│   └── Editor/                  # NOT in build
│       └── EditorOnlyCode.cs
└── Editor/                      # Project-wide editor tools
    └── SceneBootstrapper.cs
```

### Preprocessor Directives

```csharp
public class CharacterClass : ScriptableObject
{
    // Runtime code always runs
    public int BaseHP;

    // Editor-only code
#if UNITY_EDITOR
    void OnValidate()
    {
        // Validation that only runs in editor
        BaseHP = Mathf.Max(1, BaseHP);
    }
#endif
}
```

### Editor Assembly Example

```csharp
// File: Assets/Scripts/Editor/SceneBootstrapper.cs
// Assembly: Unity.BossRoom.Editor (Editor-only platform)

using UnityEditor;  // Only available in editor

namespace Unity.BossRoom.Editor
{
    [InitializeOnLoad]  // Editor-only attribute
    public class SceneBootstrapper
    {
        static SceneBootstrapper()
        {
            EditorApplication.playModeStateChanged += OnPlayModeChanged;
        }
    }
}
```

---

## MonoBehaviour Best Practices

### Lifecycle Method Order

```csharp
public class MyComponent : MonoBehaviour
{
    // 1. AWAKE - Called once when instantiated
    void Awake()
    {
        // Initialize self, cache components
        m_Rigidbody = GetComponent<Rigidbody>();
    }

    // 2. ONENABLE - Called when enabled (can be multiple times)
    void OnEnable()
    {
        // Subscribe to events
        EventBus.Subscribe(OnSomeEvent);
    }

    // 3. START - Called once before first Update
    void Start()
    {
        // Initialize based on other objects
        m_Target = FindObjectOfType<Target>();
    }

    // 4. ONDISABLE - Called when disabled
    void OnDisable()
    {
        // Unsubscribe from events
        EventBus.Unsubscribe(OnSomeEvent);
    }

    // 5. ONDESTROY - Called when destroyed
    void OnDestroy()
    {
        // Final cleanup
    }
}
```

### SerializeField vs Public

```csharp
public class MyComponent : MonoBehaviour
{
    // ✅ GOOD: Private field exposed to Inspector
    [SerializeField] int m_Health;
    
    // ❌ AVOID: Public field (exposes internal state)
    public int Health;
    
    // ✅ GOOD: Public property with private backing
    [SerializeField] int m_MaxHealth;
    public int MaxHealth => m_MaxHealth;
}
```

### Cache Component References

```csharp
public class MyComponent : MonoBehaviour
{
    // ✅ GOOD: Cached reference
    [SerializeField] Rigidbody m_Rigidbody;
    
    void FixedUpdate()
    {
        m_Rigidbody.AddForce(Vector3.up);  // Fast
    }
}

// ❌ BAD: GetComponent every frame
void Update()
{
    GetComponent<Rigidbody>().AddForce(Vector3.up);  // Slow!
}
```

---

## Serialization Patterns

### Tooltip and Header

```csharp
public class WeaponConfig : ScriptableObject
{
    [Header("Combat Stats")]
    [Tooltip("Damage dealt per hit")]
    [SerializeField] int m_Damage = 10;
    
    [Tooltip("Attacks per second")]
    [Range(0.1f, 10f)]
    [SerializeField] float m_AttackSpeed = 1f;

    [Header("Visual")]
    [SerializeField] Sprite m_Icon;
    [SerializeField] GameObject m_Prefab;
}
```

### Validation in OnValidate

```csharp
void OnValidate()
{
    // Clamp values to valid ranges
    m_Damage = Mathf.Max(0, m_Damage);
    m_AttackSpeed = Mathf.Clamp(m_AttackSpeed, 0.1f, 10f);
    
    // Auto-generate dependent values
    if (string.IsNullOrEmpty(m_DisplayName))
    {
        m_DisplayName = name;
    }
}
```

### HideInInspector for Internal State

```csharp
public class GuidScriptableObject : ScriptableObject
{
    [HideInInspector]  // Don't show in Inspector
    [SerializeField]   // But do serialize
    byte[] m_Guid;
    
    public Guid Guid => new Guid(m_Guid);
}
```

---

## Unity API Patterns

### RequireComponent

```csharp
// Automatically adds Rigidbody when this component is added
[RequireComponent(typeof(Rigidbody))]
[RequireComponent(typeof(Collider))]
public class PhysicsCharacter : MonoBehaviour
{
    Rigidbody m_Rigidbody;
    
    void Awake()
    {
        // Guaranteed to exist
        m_Rigidbody = GetComponent<Rigidbody>();
    }
}
```

### DisallowMultipleComponent

```csharp
// Only one allowed per GameObject
[DisallowMultipleComponent]
public class PlayerController : MonoBehaviour { }
```

### ExecuteAlways

```csharp
// Runs in edit mode too (for preview)
[ExecuteAlways]
public class LightProbe : MonoBehaviour
{
    void Update()
    {
        #if UNITY_EDITOR
        if (!Application.isPlaying)
        {
            UpdatePreview();
        }
        #endif
    }
}
```

### CreateAssetMenu

```csharp
[CreateAssetMenu(
    fileName = "New Weapon", 
    menuName = "GameData/Weapon Config", 
    order = 1)]
public class WeaponConfig : ScriptableObject { }
```

---

## Key Takeaways

| Pattern | Description |
|---------|-------------|
| **Assembly Definitions** | Faster builds, explicit dependencies |
| **Namespace = Folder Path** | Consistent organization |
| **Editor/ Folders** | Separate editor-only code |
| **SerializeField + Private** | Expose to Inspector, hide from code |
| **Cache Components** | GetComponent in Awake, not Update |
| **OnValidate** | Editor-time validation |
| **RequireComponent** | Enforce dependencies |

### Project Checklist

- [ ] Create assembly definition for each major feature
- [ ] Match namespaces to folder structure
- [ ] Put editor code in `Editor/` folders
- [ ] Use `[SerializeField]` instead of public fields
- [ ] Add `[Tooltip]` to all serialized fields
- [ ] Cache component references in Awake
- [ ] Use `OnValidate` for data validation

---

## Exercises

1. **Create Assembly Definition:** Add a new feature folder with its own asmdef and proper references.

2. **Refactor Public Fields:** Find a class with public fields and refactor to SerializeField + properties.

3. **Add Validation:** Add OnValidate to a ScriptableObject to ensure all values are valid.

---

> **Related Guides:**
> - [02_clean_code_patterns.md](./02_clean_code_patterns.md) — General clean code
> - [05_project_structure.md](./05_project_structure.md) — Project organization
> - [26_scriptableobject_architecture.md](./26_scriptableobject_architecture.md) — SO patterns
