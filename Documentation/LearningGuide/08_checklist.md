# 08: Comprehensive Checklists

> **Purpose:** Quick reference checklists for every stage of game development. Print these and keep them at your desk.

---

## Table of Contents

1. [New Project Setup](#-new-project-setup)
2. [Architecture Decisions](#-architecture-decisions)
3. [Core Systems Checklist](#-core-systems-checklist)
4. [Per-Feature Checklist](#-per-feature-checklist)
5. [Clean Code Checklist](#-clean-code-checklist)
6. [Performance Checklist](#-performance-checklist)
7. [Multiplayer Checklist](#-multiplayer-checklist)
8. [Pre-Commit Checklist](#-pre-commit-checklist)
9. [Code Review Checklist](#-code-review-checklist)
10. [Release Readiness](#-release-readiness)
11. [Boss Room Key Files Reference](#-boss-room-key-files-reference)

---

## 🚀 New Project Setup

### Day 1: Foundation
- [ ] Create project with Unity version appropriate for target platforms
- [ ] Set up version control (Git) with .gitignore
- [ ] Create folder structure (see [05_project_structure.md](./05_project_structure.md))
- [ ] Set up namespaces matching folder structure
- [ ] Create `Bootstrap.cs` entry point in a dedicated startup scene

### Week 1: Core Infrastructure
- [ ] Event system (see Template 1 in [07_implementation_templates.md](./07_implementation_templates.md))
- [ ] Service locator or DI container setup
- [ ] Scene loading system with loading screen
- [ ] Audio manager with pooled sources
- [ ] Object pool for frequently spawned objects
- [ ] Input handling abstraction

### Week 2: Game Framework
- [ ] Save/Load system interface
- [ ] UI manager with screen stack
- [ ] Settings/preferences system
- [ ] Analytics integration (if needed)
- [ ] Error handling and logging

---

## 🏗️ Architecture Decisions

### Before Starting Development

| Decision | Options | Boss Room Uses |
|----------|---------|----------------|
| DI Framework | VContainer / Zenject / ServiceLocator / Manual | VContainer |
| Event System | C# events / Unity Events / MessageChannel | MessageChannel |
| State Management | Enum+Switch / State Pattern / FSM Library | State Pattern |
| Object Pooling | Unity ObjectPool / Custom / AddressablePool | Custom + Unity |
| Save Format | JSON / Binary / PlayerPrefs / Cloud | JSON |

### Questions to Answer

- [ ] What's the target platform(s)?
- [ ] Solo or team development?
- [ ] Will it need multiplayer (now or later)?
- [ ] What's the expected project duration?
- [ ] What's the test strategy?

---

## 🔧 Core Systems Checklist

### Event System
- [ ] Central event hub or message channels created
- [ ] Raise methods for each event type
- [ ] Clear method for scene transitions
- [ ] Documentation of all events

### Object Pooling
- [ ] Pool for bullets/projectiles
- [ ] Pool for particle systems
- [ ] Pool for damage numbers/floating text
- [ ] Pool for frequently spawned enemies
- [ ] Prewarm pools during loading

### State Machines
- [ ] Game flow state machine (Menu → Playing → Paused → GameOver)
- [ ] Player state machine (Idle → Move → Jump → Attack)
- [ ] AI state machines
- [ ] UI screen state machine

### Data Management
- [ ] ScriptableObject configs for all tweakable values
- [ ] Runtime sets for object tracking
- [ ] GameConfig for global settings
- [ ] Level/stage data structures

### Save System
- [ ] ISaveSystem interface defined
- [ ] Local save implementation
- [ ] Auto-save on key events
- [ ] Save slot management
- [ ] Corruption handling

### Audio
- [ ] Sound effect playback
- [ ] Music playback with crossfade
- [ ] Volume controls (master/music/sfx)
- [ ] Audio occlusion (if 3D)
- [ ] Pooled audio sources

---

## 📦 Per-Feature Checklist

### When Adding Any New Feature

#### Planning
- [ ] Feature requirements documented
- [ ] Data structures defined (what needs to be stored?)
- [ ] Events identified (what notifies what?)
- [ ] UI requirements identified
- [ ] Dependencies on other systems identified

#### Implementation
- [ ] Create configuration ScriptableObject
- [ ] Create logic class (rules, state, no visuals)
- [ ] Create view class (visuals, reads from logic)
- [ ] Create UI components
- [ ] Wire up events
- [ ] Handle edge cases (null, invalid input)

#### Polish
- [ ] Add to object pool if spawned frequently
- [ ] Add visual feedback (particles, screen shake)
- [ ] Add audio feedback
- [ ] Add save/load support if needed
- [ ] Add analytics tracking if needed

#### Quality
- [ ] Unit tests for logic
- [ ] Manual testing complete
- [ ] Edge cases tested
- [ ] Performance checked

---

## ✨ Clean Code Checklist

### For Every Class

#### Responsibility
- [ ] Class has ONE reason to change
- [ ] Class name clearly describes its purpose
- [ ] No "And" or "Manager" in name (unless truly managing)
- [ ] < 300 lines of code (guideline)

#### Dependencies
- [ ] Dependencies are explicit (injected or serialized)
- [ ] No `FindObjectOfType` in runtime code
- [ ] No `GetComponent` in Update (cache in Awake)
- [ ] Uses interfaces for major dependencies

#### Naming
- [ ] Class: `PascalCase` (noun)
- [ ] Method: `PascalCase` (verb)
- [ ] Private field: `m_CamelCase`
- [ ] Constant: `UPPER_CASE`
- [ ] Boolean: starts with Is/Has/Can
- [ ] Event handler: `On` prefix

#### Documentation
- [ ] Public API has XML comments (`///`)
- [ ] Complex algorithms explained
- [ ] Workarounds documented with why

#### Safety
- [ ] Null checks before dereferencing
- [ ] Event subscriptions have matching unsubscriptions
- [ ] Collections initialized before use
- [ ] TryGet patterns used where appropriate

---

## ⚡ Performance Checklist

### Avoid in Update/FixedUpdate
- [ ] No `FindObjectOfType`
- [ ] No `GetComponent` (cache in Awake)
- [ ] No `new` allocations (pool or cache)
- [ ] No string concatenation with `+`
- [ ] No LINQ queries
- [ ] No `foreach` on non-array collections

### Memory Management
- [ ] Frequently spawned objects use pools
- [ ] Strings use `StringBuilder`
- [ ] Physics queries use `NonAlloc` versions
- [ ] Animator parameters use `StringToHash`
- [ ] Large data uses structs where appropriate

### Best Practices
- [ ] Event-driven > polling
- [ ] Batch updates (UpdateManager pattern)
- [ ] Profile before optimizing
- [ ] Verify fixes with profiler

---

## 🌐 Multiplayer Checklist

### Architecture
- [ ] Logic/View separation in place
- [ ] All game state in centralized location
- [ ] No client-only state that should be synced
- [ ] Server authoritative design

### Synchronization
- [ ] NetworkVariables for persistent state
- [ ] RPCs for one-time events
- [ ] NetworkList for collections
- [ ] Appropriate sync granularity

### Security
- [ ] Server validates ALL client requests
- [ ] No trust in client-sent data
- [ ] Rate limiting on actions
- [ ] Sanity checks on values

### User Experience
- [ ] Client anticipation for actions
- [ ] Latency compensation
- [ ] Reconnection support
- [ ] Graceful disconnect handling
- [ ] Connection state feedback to player

### Testing
- [ ] Tested with artificial latency
- [ ] Tested with packet loss
- [ ] Tested with multiple clients
- [ ] Tested disconnect scenarios

---

## 📝 Pre-Commit Checklist

### Before Every Commit

#### Code Quality
- [ ] No hardcoded magic numbers
- [ ] No `FindObjectOfType` in gameplay code
- [ ] No empty catch blocks
- [ ] No commented-out code
- [ ] Events properly unsubscribed
- [ ] Objects returned to pools

#### Standards
- [ ] Naming follows conventions
- [ ] Code is formatted consistently
- [ ] No compiler warnings
- [ ] Public methods documented

#### Functionality
- [ ] Feature works as intended
- [ ] Edge cases handled
- [ ] No obvious bugs introduced
- [ ] Existing tests still pass

---

## 👀 Code Review Checklist

### For Reviewers

#### Architecture
- [ ] Changes align with project architecture
- [ ] No unnecessary coupling introduced
- [ ] Dependencies flow downward only
- [ ] Separation of concerns maintained

#### Quality
- [ ] Single responsibility per class
- [ ] Code is testable
- [ ] Naming is clear and consistent
- [ ] No duplicate code

#### Performance
- [ ] No obvious performance issues
- [ ] Allocations appropriate
- [ ] Resources properly disposed

#### Edge Cases
- [ ] Null handling
- [ ] Error handling
- [ ] Boundary conditions

---

## 🚢 Release Readiness

### Code Complete
- [ ] All planned features implemented
- [ ] All known bugs fixed
- [ ] Performance acceptable on target platforms
- [ ] Memory usage within bounds

### Testing
- [ ] All automated tests passing
- [ ] Manual testing completed
- [ ] Platform-specific testing done
- [ ] Edge case testing done
- [ ] Localization tested (if applicable)

### Polish
- [ ] All placeholder content replaced
- [ ] Audio complete and mixed
- [ ] Visual effects complete
- [ ] UI/UX reviewed

### Documentation
- [ ] README up to date
- [ ] Build instructions documented
- [ ] Known issues documented
- [ ] Changelog updated

---

## 📚 Boss Room Key Files Reference

### Entry Points
| Purpose | File |
|---------|------|
| Application startup | `ApplicationController.cs` |
| DI container setup | `GameObjectContext.cs` |
| Scene bootstrap | Scene prefabs |

### State Management
| Purpose | File |
|---------|------|
| Connection states | `ConnectionState.cs` |
| Game states | `GameStateBehaviour.cs` |
| State transitions | `ConnectionManager.cs` |

### Core Patterns
| Purpose | File |
|---------|------|
| PubSub | `MessageChannel.cs` |
| Object pool | `NetworkObjectPool.cs` |
| Action system | `Action.cs`, `ActionFactory.cs` |

### Characters
| Purpose | File |
|---------|------|
| Server logic | `ServerCharacter.cs` |
| Client visuals | `ClientCharacter.cs` |
| Movement | `ServerCharacterMovement.cs` |
| Health | `NetworkHealthState.cs` |

### Data
| Purpose | File |
|---------|------|
| Action config | `ActionConfig.cs` |
| Character class | `CharacterClass.cs` |
| Avatar config | `AvatarConfiguration.cs` |

---

## 🎯 Pattern Quick Reference

| Need | Pattern | Template |
|------|---------|----------|
| Decoupled communication | PubSub/Events | Template 1 |
| Efficient spawning | Object Pool | Template 2 |
| Complex state logic | State Machine | Template 3 |
| Service access | Service Locator | Template 4 |
| Configurable data | ScriptableObject | Template 5 |
| Persistence | Save System | Template 6 |
| Audio playback | Audio Manager | Template 7 |
| Scene transitions | Scene Loader | Template 8 |
| Input abstraction | Input Handler | Template 9 |
| Screen management | UI Manager | Template 10 |

---

## 💡 Golden Rules to Remember

1. **Logic/View Separation** = Server/Client separation = Testability
2. **Events for Communication** = Loose coupling = Maintainability
3. **Data Drives Behavior** = ScriptableObjects = Designer-friendly
4. **Single Responsibility** = Small classes = Readability
5. **Explicit Dependencies** = Injected/Serialized = No surprises
6. **Pool Frequently Spawned** = No GC = Smooth gameplay
7. **Cache Expensive Lookups** = Fast = Happy players

---

## 📋 Print-Friendly Summary

```
╔══════════════════════════════════════════════════════════════════╗
║                     GAME DEV QUICK REFERENCE                      ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ARCHITECTURE PILLARS:                                           ║
║  □ Separate Logic from View                                      ║
║  □ Events for communication                                      ║
║  □ Data drives behavior (ScriptableObjects)                      ║
║  □ Pool frequently spawned objects                               ║
║  □ State machines for complex states                             ║
║                                                                  ║
║  EVERY CLASS SHOULD:                                             ║
║  □ Have ONE responsibility                                       ║
║  □ Name describe its purpose                                     ║
║  □ Dependencies be explicit                                      ║
║  □ Events be unsubscribed                                        ║
║  □ Handle null cases                                             ║
║                                                                  ║
║  NEVER IN UPDATE():                                              ║
║  □ FindObjectOfType                                              ║
║  □ GetComponent (cache instead)                                  ║
║  □ new allocations                                               ║
║  □ String concatenation                                          ║
║  □ LINQ queries                                                  ║
║                                                                  ║
║  BEFORE COMMIT:                                                  ║
║  □ No magic numbers                                              ║
║  □ No commented code                                             ║
║  □ Events unsubscribed                                           ║
║  □ Pools used properly                                           ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

> **Cross-Reference:**
> - [07_implementation_templates.md](./07_implementation_templates.md) - Copy-paste code
> - [14_antipatterns_guide.md](./14_antipatterns_guide.md) - What NOT to do
> - [17_architecture_decision_framework.md](./17_architecture_decision_framework.md) - Decision guidance
