# 05: Project Structure Guide

> **Goal:** Learn how to organize a scalable Unity project based on Boss Room's structure.

---

## Boss Room Structure Overview

```
Assets/
├── Scripts/                    # All C# code
│   ├── ApplicationLifecycle/   # Entry point, DI setup
│   ├── ConnectionManagement/   # Network connection
│   ├── Gameplay/               # Game logic
│   │   ├── Action/            # Combat system
│   │   ├── GameState/         # Game flow
│   │   ├── GameplayObjects/   # Entities
│   │   └── UI/                # UI scripts
│   ├── Infrastructure/         # Utilities
│   └── UnityServices/          # External services
├── Prefabs/                    # Prefabricated objects
├── ScriptableObjects/          # Data assets
├── Scenes/                     # Unity scenes
└── GameData/                   # Config files
```

---

## Recommended Structure for YOUR Game

```
Assets/
├── _Project/                   # Your game-specific code
│   ├── Scripts/
│   │   ├── Core/              # Entry, managers, DI
│   │   ├── Gameplay/          # Game mechanics
│   │   ├── UI/                # UI logic
│   │   └── Data/              # Data structures
│   ├── Prefabs/
│   ├── Scenes/
│   └── Config/                # ScriptableObjects
├── _Common/                    # Reusable across projects
│   ├── Utilities/
│   ├── Extensions/
│   └── ThirdParty/
└── Resources/                  # Dynamically loaded
```

---

## Assembly Definitions

Boss Room uses **Assembly Definitions** (`.asmdef`) to:
- Speed up compilation (only recompile changed assemblies)
- Enforce boundaries between systems
- Prevent circular dependencies

```
Unity.BossRoom.Gameplay.asmdef
├── References: Infrastructure, Utils
└── Cannot reference: ConnectionManagement (one-way)

Unity.BossRoom.ConnectionManagement.asmdef
├── References: Infrastructure
└── Cannot reference: Gameplay
```

**Rule:** Dependencies flow ONE direction (down the layer hierarchy).

---

## Folder Naming Conventions

| Prefix | Purpose |
|--------|---------|
| `_` | Your project-specific folders (appear at top) |
| `ThirdParty/` | External assets/plugins |
| `Editor/` | Editor-only scripts |
| `Tests/` | Unit/integration tests |
| `Resources/` | Dynamically loaded assets |

---

## File Organization Rules

1. **One class per file** (with same name)
2. **Folder = namespace** (Scripts/Gameplay → namespace Gameplay)
3. **Related files together** (MeleeAction.cs, MeleeAction.Client.cs)
4. **Tests mirror structure** (Scripts/Gameplay/Action → Tests/Gameplay/ActionTests)

---

## Quick Setup Checklist

- [ ] Create `_Project` folder structure
- [ ] Add assembly definitions for major systems
- [ ] Create `_Common` for reusable code
- [ ] Set up ScriptableObject config folder
- [ ] Create entry point scene (Startup/Bootstrap)
