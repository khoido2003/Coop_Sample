# 05: Project Structure Guide

> **Purpose:** Learn how to organize a scalable Unity project that supports growth, team collaboration, and maintainability. This guide covers folder organization, namespaces, assembly definitions, and scaling strategies.

---

## Table of Contents

1. [Why Structure Matters](#why-structure-matters)
2. [Boss Room Structure Analysis](#boss-room-structure-analysis)
3. [Folder Organization Principles](#folder-organization-principles)
4. [Namespace Strategy](#namespace-strategy)
5. [Assembly Definitions In-Depth](#assembly-definitions-in-depth)
6. [Scaling Strategies](#scaling-strategies)
7. [Template Structures by Project Size](#template-structures-by-project-size)
8. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
9. [Migration Guide](#migration-guide)

---

## Why Structure Matters

### The Real Cost of Poor Structure

```
WEEK 1:  "I'll just put everything in Scripts folder for now"
WEEK 4:  "Where's that utility class again?"
WEEK 8:  "I'm scared to touch this file, it might break everything"
WEEK 16: "We need to rewrite everything because we can't add features"
```

### Benefits of Good Structure

| Benefit | Description |
|---------|-------------|
| **Findability** | Anyone can locate code quickly |
| **Ownership** | Clear who's responsible for what |
| **Isolation** | Changes in one area don't break others |
| **Compilation** | Assembly definitions speed up builds |
| **Testability** | Easy to test systems in isolation |
| **Reusability** | Easy to extract for other projects |

---

## Boss Room Structure Analysis

### Complete Folder Tree

```
Assets/
├── Art/                           # All visual assets
│   ├── Materials/
│   ├── Models/
│   ├── Textures/
│   └── VFX/
│
├── Audio/                         # All audio assets
│   ├── Music/
│   └── SFX/
│
├── GameData/                      # ScriptableObject instances
│   ├── Actions/                   # ActionConfig instances
│   ├── Characters/                # CharacterConfig instances
│   └── Abilities/                 # Ability definitions
│
├── Prefabs/                       # All prefabs
│   ├── Characters/
│   ├── UI/
│   ├── VFX/
│   └── Environment/
│
├── Scenes/                        # Unity scenes
│   ├── Startup.unity             # Bootstrap/entry scene
│   ├── MainMenu.unity
│   ├── CharSelect.unity
│   └── BossRoom.unity
│
├── Scripts/                       # All C# code
│   ├── ApplicationLifecycle/     # Entry point, DI container
│   │   └── ApplicationController.cs
│   │
│   ├── ConnectionManagement/     # Network connection
│   │   ├── ConnectionManager.cs
│   │   ├── ConnectionMethod.cs
│   │   └── ConnectionState/      # State machine states
│   │       ├── ConnectionState.cs
│   │       ├── OfflineState.cs
│   │       ├── HostingState.cs
│   │       └── ...
│   │
│   ├── Gameplay/                 # Core game logic
│   │   ├── Action/               # Ability system
│   │   │   ├── Action.cs
│   │   │   ├── ActionConfig.cs
│   │   │   ├── ActionFactory.cs
│   │   │   └── ConcreteActions/
│   │   │       ├── MeleeAction.cs
│   │   │       ├── MeleeAction.Client.cs
│   │   │       └── ...
│   │   │
│   │   ├── Configuration/        # Config ScriptableObjects
│   │   │   ├── CharacterClass.cs
│   │   │   └── AvatarConfiguration.cs
│   │   │
│   │   ├── GameplayObjects/      # Game entities
│   │   │   ├── Character/
│   │   │   │   ├── ServerCharacter.cs
│   │   │   │   ├── ClientCharacter.cs
│   │   │   │   └── ...
│   │   │   ├── RuntimeDataContainers/
│   │   │   └── Pickups/
│   │   │
│   │   ├── GameState/            # Game flow management
│   │   │   ├── GameStateBehaviour.cs
│   │   │   ├── ServerBossRoomState.cs
│   │   │   └── ...
│   │   │
│   │   └── UI/                   # Game UI
│   │       ├── HeroActionBar.cs
│   │       └── ...
│   │
│   ├── Infrastructure/           # Reusable utilities
│   │   ├── PubSub/               # Event system
│   │   │   ├── IMessageChannel.cs
│   │   │   ├── MessageChannel.cs
│   │   │   └── NetworkedMessageChannel.cs
│   │   ├── NetworkObjectPool.cs
│   │   ├── DisposableGroup.cs
│   │   └── UpdateRunner.cs
│   │
│   ├── UnityServices/            # External services
│   │   ├── Auth/
│   │   ├── Lobbies/
│   │   └── Vivox/
│   │
│   └── Utils/                    # Helper utilities
│       ├── Extensions/
│       └── SceneUtils.cs
│
└── ThirdParty/                   # External packages
    └── VContainer/
```

### Key Observations

1. **Separation by Technical Layer**
   - `Scripts/` contains only code
   - `Prefabs/` contains only prefabs
   - `GameData/` contains only ScriptableObjects

2. **Separation by Feature** (within Scripts/)
   - `Gameplay/Action/` - All action system code
   - `ConnectionManagement/` - All networking code

3. **Separation by Role**
   - `Server*.cs` - Server logic
   - `Client*.cs` - Client visuals
   - `Network*.cs` - Sync code

---

## Folder Organization Principles

### Principle 1: Logical Grouping

Group files by **what they do together**, not by **what type they are**.

```
❌ BAD: By file type
Scripts/
├── Interfaces/
│   ├── IDamageable.cs
│   ├── IPoolable.cs
│   └── ISaveable.cs
├── Classes/
│   ├── Enemy.cs
│   ├── Player.cs
│   └── Projectile.cs
└── ScriptableObjects/
    ├── EnemyConfig.cs
    └── PlayerConfig.cs

✅ GOOD: By feature
Scripts/
├── Combat/
│   ├── IDamageable.cs       # Interface used by combat
│   ├── DamageCalculator.cs  # Combat logic
│   └── CombatConfig.cs      # Combat settings
├── Enemies/
│   ├── Enemy.cs
│   ├── EnemyConfig.cs
│   └── EnemyAI.cs
└── Player/
    ├── Player.cs
    ├── PlayerConfig.cs
    └── PlayerInput.cs
```

### Principle 2: Depth Matches Complexity

Simple projects = shallow folders. Complex projects = deeper folders.

```
// Small game (2-month project)
Scripts/
├── Core/
├── Gameplay/
├── UI/
└── Utils/

// Large game (12+ month project)
Scripts/
├── Core/
│   ├── Bootstrap/
│   ├── DI/
│   ├── Events/
│   └── State/
├── Gameplay/
│   ├── Combat/
│   │   ├── Actions/
│   │   ├── Damage/
│   │   └── Targeting/
│   ├── Progression/
│   │   ├── Experience/
│   │   ├── Leveling/
│   │   └── Skills/
│   └── AI/
│       ├── Behaviors/
│       └── Pathfinding/
└── ...
```

### Principle 3: Clear Entry Points

Every major folder should have an obvious "main" file.

```
ConnectionManagement/
├── ConnectionManager.cs      # ← THE entry point for this system
├── ConnectionMethod.cs
├── ConnectionPayload.cs
└── ConnectionState/
    └── ...

Gameplay/Action/
├── Action.cs                 # ← THE base class
├── ActionConfig.cs
├── ActionFactory.cs
└── ConcreteActions/
    └── ...
```

### Principle 4: Keep Related Files Together

Files that change together should live together.

```
// MeleeAction has server and client parts - keep them together!
ConcreteActions/
├── MeleeAction.cs           # Server logic
├── MeleeAction.Client.cs    # Client logic (partial class)
├── MeleeAction.asset        # Config (ScriptableObject instance)
└── MeleeActionVFX.prefab    # Visual effects
```

---

## Namespace Strategy

### Follow Folder Structure

```csharp
// File: Scripts/Gameplay/Action/MeleeAction.cs
namespace Unity.BossRoom.Gameplay.Actions
{
    public class MeleeAction : Action { }
}

// File: Scripts/Infrastructure/PubSub/MessageChannel.cs
namespace Unity.BossRoom.Infrastructure
{
    public class MessageChannel<T> : IMessageChannel<T> { }
}
```

### Namespace Hierarchy

```
Unity.BossRoom                       # Root namespace
├── Unity.BossRoom.ApplicationLifecycle
├── Unity.BossRoom.ConnectionManagement
├── Unity.BossRoom.Gameplay
│   ├── Unity.BossRoom.Gameplay.Actions
│   ├── Unity.BossRoom.Gameplay.GameState
│   └── Unity.BossRoom.Gameplay.UI
├── Unity.BossRoom.Infrastructure
└── Unity.BossRoom.UnityServices
```

### Benefits of Namespaces

```csharp
// Without namespaces - collisions!
class Player { }  // Your code
class Player { }  // Some asset you imported

// With namespaces - no problem
MyGame.Gameplay.Player { }        // Your code
ThirdParty.Networking.Player { }  // Imported asset

// Clean imports
using MyGame.Gameplay;
using static MyGame.Gameplay.GameEvents;  // Static usings
```

---

## Assembly Definitions In-Depth

### What Are Assembly Definitions?

Assembly Definitions (`.asmdef`) split your code into separate compiled units.

```
Without ASMDEF:                  With ASMDEF:
┌─────────────────────┐         ┌─────────────┐   ┌─────────────┐
│  Assembly-CSharp    │         │  Gameplay   │   │ Connection  │
│  (ALL your code)    │         │  .asmdef    │   │ .asmdef     │
└─────────────────────┘         └─────────────┘   └─────────────┘
                                ┌─────────────┐   ┌─────────────┐
One change = recompile ALL      │Infrastructure│   │  UI         │
                                │  .asmdef    │   │  .asmdef    │
                                └─────────────┘   └─────────────┘
                                
                                Change Gameplay = only recompile
                                Gameplay.asmdef (fast!)
```

### Boss Room's Assembly Structure

```
Unity.BossRoom.ApplicationLifecycle.asmdef
├── References: Infrastructure, ConnectionManagement
└── Purpose: Entry point, DI container setup

Unity.BossRoom.ConnectionManagement.asmdef
├── References: Infrastructure, UnityServices
└── Purpose: Network connection handling

Unity.BossRoom.Gameplay.asmdef
├── References: Infrastructure, ConnectionManagement
└── Purpose: Core game logic

Unity.BossRoom.Infrastructure.asmdef
├── References: (none - base layer)
└── Purpose: Reusable utilities

Unity.BossRoom.UnityServices.asmdef
├── References: Infrastructure
└── Purpose: Unity Authentication, Lobbies, Relay
```

### Creating Assembly Definitions

```
1. Right-click folder → Create → Assembly Definition
2. Name it: [CompanyName].[ProjectName].[System].asmdef
3. Configure references (only what's needed)
4. Set platforms if applicable
```

### Assembly Reference Rules

```
┌─────────────────────────────────────────────────────────────────┐
│                     DEPENDENCY DIRECTION                         │
│                                                                 │
│   Application Layer                                              │
│        ↓ (uses)                                                 │
│   Feature Layers (Gameplay, Connection, etc.)                    │
│        ↓ (uses)                                                 │
│   Infrastructure Layer                                           │
│        ↓ (uses)                                                 │
│   Unity APIs / Third Party                                       │
│                                                                 │
│   NEVER reference upward! (circular dependency)                  │
└─────────────────────────────────────────────────────────────────┘
```

### Common ASMDEF Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| "Type not found" | Missing reference | Add asmdef to references |
| Circular dependency | Both reference each other | Extract shared code to lower layer |
| MonoBehaviour not found | Different assemblies | Add reference or move code |

---

## Scaling Strategies

### Strategy 1: Start Simple, Grow Intentionally

```
PHASE 1 (Prototype):
Scripts/
├── Core/
├── Gameplay/
└── UI/

PHASE 2 (MVP):
Scripts/
├── Core/
│   └── Bootstrap.cs
├── Gameplay/
│   ├── Player/
│   └── Combat/
├── UI/
└── Utils/

PHASE 3 (Production):
Scripts/
├── Core/
│   ├── Bootstrap/
│   ├── DI/
│   └── Events/
├── Gameplay/
│   ├── Player/
│   ├── Combat/
│   │   ├── Actions/
│   │   ├── Damage/
│   │   └── AI/
│   └── Progression/
├── UI/
│   ├── Screens/
│   └── Widgets/
├── Infrastructure/
├── Services/
└── Utils/
```

### Strategy 2: Feature Folders for Team Development

When multiple developers work simultaneously:

```
Scripts/
├── Core/                    # Senior dev / architect owns
│   └── ...
├── Features/                # Each dev owns their feature
│   ├── Combat/              # Dev A owns
│   │   └── ...
│   ├── Inventory/           # Dev B owns
│   │   └── ...
│   └── Quests/              # Dev C owns
│       └── ...
├── Shared/                  # Shared utilities
└── ThirdParty/
```

### Strategy 3: Domain-Driven Structure

For complex games with clear domains:

```
Scripts/
├── Domain/                  # Pure game logic (no Unity)
│   ├── Combat/
│   ├── Inventory/
│   └── Progression/
├── Application/             # Use cases, orchestration
│   ├── GameFlow/
│   └── Services/
├── Infrastructure/          # Persistence, networking
│   ├── Save/
│   └── Network/
└── Presentation/            # UI, View components
    ├── HUD/
    └── Menus/
```

---

## Template Structures by Project Size

### Solo Prototype (1-2 months)

```
Assets/
├── _Game/
│   ├── Scripts/
│   │   ├── Core/
│   │   │   └── GameManager.cs
│   │   ├── Gameplay/
│   │   │   ├── Player.cs
│   │   │   └── Enemy.cs
│   │   └── UI/
│   │       └── HUD.cs
│   ├── Prefabs/
│   ├── Scenes/
│   └── Art/
└── Resources/
```

### Small Team (2-5 devs, 3-6 months)

```
Assets/
├── _Project/
│   ├── Scripts/
│   │   ├── Core/
│   │   │   ├── Bootstrap/
│   │   │   └── Events/
│   │   ├── Gameplay/
│   │   │   ├── Player/
│   │   │   ├── Enemies/
│   │   │   ├── Combat/
│   │   │   └── Pickups/
│   │   ├── UI/
│   │   │   ├── Screens/
│   │   │   └── Widgets/
│   │   └── Services/
│   │       ├── Audio/
│   │       ├── Save/
│   │       └── Analytics/
│   ├── Prefabs/
│   │   ├── Characters/
│   │   ├── UI/
│   │   └── VFX/
│   ├── GameData/
│   ├── Scenes/
│   └── Art/
├── _Common/             # Reusable across projects
│   └── Utils/
└── ThirdParty/
```

### Large Team (5+ devs, 12+ months)

```
Assets/
├── _BossRoom/                # Project-specific
│   ├── Art/
│   ├── Audio/
│   ├── GameData/
│   ├── Prefabs/
│   ├── Scenes/
│   └── Scripts/
│       ├── ApplicationLifecycle/
│       ├── ConnectionManagement/
│       ├── Gameplay/
│       │   ├── Actions/
│       │   ├── AI/
│       │   ├── Characters/
│       │   ├── GameState/
│       │   ├── Pickups/
│       │   └── UI/
│       └── UnityServices/
├── _Common/                  # Shared infrastructure
│   ├── Infrastructure/
│   │   ├── DI/
│   │   ├── Events/
│   │   ├── Pooling/
│   │   └── State/
│   └── Utils/
│       ├── Extensions/
│       └── Helpers/
├── Packages/                 # Package manager packages
├── ThirdParty/              # Imported assets
└── Tests/                   # Unit & integration tests
    ├── EditMode/
    └── PlayMode/
```

---

## Common Mistakes to Avoid

### Mistake 1: Everything in One Folder

```
❌ BAD:
Scripts/
├── Player.cs
├── Enemy.cs
├── GameManager.cs
├── UIController.cs
├── SaveSystem.cs
├── AudioManager.cs
├── ... (200 more files)

✅ FIX: Group by feature/responsibility
```

### Mistake 2: Too Many Levels of Nesting

```
❌ BAD:
Scripts/Gameplay/Systems/Combat/Damage/Types/Physical/Modifiers/Armor/...

✅ FIX: Flatten to 3-4 levels maximum
```

### Mistake 3: Mixing Assets and Code

```
❌ BAD:
Player/
├── Player.cs
├── Player.prefab
├── PlayerTexture.png
├── PlayerConfig.asset
└── PlayerAudio.wav

✅ FIX: Separate by asset type in the main hierarchy
Scripts/Player/Player.cs
Prefabs/Player.prefab
Art/Characters/PlayerTexture.png
```

### Mistake 4: No Clear Entry Point

```
❌ BAD: 50 files, no idea where to start reading

✅ FIX: 
Scripts/
├── Bootstrap/
│   └── GameBootstrap.cs    # READ THIS FIRST (clear entry point)
├── Core/
└── ...
```

---

## Migration Guide

### From Messy to Organized

**Step 1: Audit current state**
```
Count: How many files in Scripts?
├── < 50: One afternoon to reorganize
├── 50-200: 1-2 days
└── > 200: Plan carefully, do incrementally
```

**Step 2: Define target structure**
```
Draw your ideal folder structure on paper first
Don't move any files yet!
```

**Step 3: Move in waves**
```
Wave 1: Create folders, move obvious files
Wave 2: Fix compile errors
Wave 3: Move edge cases
Wave 4: Add assembly definitions
```

**Step 4: Update all references**
```
Unity handles script moves automatically, but:
- Check Prefab references
- Check ScriptableObject references  
- Check Addressables if used
```

### Script to Audit Current Structure

```csharp
// Editor script to analyze your project
[MenuItem("Tools/Audit Project Structure")]
static void AuditStructure()
{
    var scripts = Directory.GetFiles("Assets/Scripts", "*.cs", 
        SearchOption.AllDirectories);
    
    Debug.Log($"Total scripts: {scripts.Length}");
    
    var byFolder = scripts
        .GroupBy(s => Path.GetDirectoryName(s))
        .OrderByDescending(g => g.Count());
    
    foreach (var group in byFolder.Take(10))
    {
        Debug.Log($"{group.Count()} files in: {group.Key}");
    }
}
```

---

## Quick Reference Checklist

### New Project Setup

- [ ] Create `_Project` or `_[GameName]` folder
- [ ] Create folder structure matching team size
- [ ] Add entry point script in `Core/` or `Bootstrap/`
- [ ] Create `GameData/` for ScriptableObjects
- [ ] Set up `ThirdParty/` for external assets
- [ ] Add `.asmdef` for major systems (when needed)

### Before Each Feature

- [ ] Does this feature fit an existing folder?
- [ ] If new folder needed, where does it belong?
- [ ] What namespace should the code use?
- [ ] Who owns this folder? (team context)

### Code Review Checklist

- [ ] File is in appropriate folder for its responsibility
- [ ] Namespace matches folder path
- [ ] No circular dependencies introduced
- [ ] Related files are together

---

> **Cross-Reference:**
> - [01_architecture_principles.md](./01_architecture_principles.md) - Why these structures matter
> - [07_implementation_templates.md](./07_implementation_templates.md) - Copy-paste starters
> - [17_architecture_decision_framework.md](./17_architecture_decision_framework.md) - Decision guidance
