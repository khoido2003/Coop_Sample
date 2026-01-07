# 08: Quick Reference Checklist

> **Use this checklist when starting YOUR game projects.**

---

## 🚀 New Project Setup

### Architecture Decisions
- [ ] Define major systems (Input, Gameplay, UI, Audio, Save)
- [ ] Create folder structure matching architecture
- [ ] Set up assembly definitions for each system
- [ ] Decide on DI approach (VContainer, simple ServiceLocator, or manual)
- [ ] Create entry point/bootstrap scene

### Core Systems to Build First
- [ ] Event system (simple events or MessageChannel)
- [ ] Object pool for frequently spawned objects
- [ ] Data config system (ScriptableObjects)
- [ ] Save/Load system interface

---

## 📁 Folder Structure Checklist

```
Assets/
├── _Project/
│   ├── Scripts/
│   │   ├── Core/           ← Entry point, DI, lifecycle
│   │   ├── Gameplay/       ← Game mechanics
│   │   ├── UI/             ← UI scripts
│   │   └── Data/           ← Data structures
│   ├── Prefabs/
│   ├── Scenes/
│   └── Config/             ← ScriptableObjects
└── _Common/
    └── Utilities/          ← Reusable code
```

---

## ✨ Clean Code Checklist (Per Class)

- [ ] Class has ONE responsibility
- [ ] Name clearly describes purpose
- [ ] Dependencies are injected (not FindObjectOfType)
- [ ] Uses interfaces for major dependencies
- [ ] Public API is documented
- [ ] Null checks in place
- [ ] Events are unsubscribed in OnDestroy

---

## 🎮 Game Feature Checklist

When adding a new feature:

- [ ] Define data structure (ScriptableObject config)
- [ ] Create logic class (rules, no visuals)
- [ ] Create view class (visuals, reads from logic)
- [ ] Add events for state changes
- [ ] Connect to UI via events
- [ ] Add to object pool if spawned frequently

---

## 🌐 Multiplayer Checklist

If adding networking:

- [ ] Identify what needs to sync (health, position, etc.)
- [ ] Choose sync method for each (NetworkVariable, RPC, NetworkList)
- [ ] Server validates ALL client inputs
- [ ] Client anticipates actions for responsiveness
- [ ] Handle disconnect/reconnect gracefully
- [ ] Test with artificial latency

---

## 🔍 Code Review Checklist

Before committing:

- [ ] No hardcoded magic numbers (use constants/config)
- [ ] No `FindObjectOfType` in gameplay code
- [ ] No empty catch blocks
- [ ] No commented-out code
- [ ] Events unsubscribed properly
- [ ] Objects returned to pools
- [ ] Naming matches conventions

---

## 📚 Key Files to Study in Boss Room

| What to Learn | Start With |
|---------------|------------|
| Application entry | `ApplicationController.cs` |
| State machines | `ConnectionState.cs`, `GameStateBehaviour.cs` |
| Event system | `MessageChannel.cs` |
| Action system | `Action.cs`, `MeleeAction.cs` |
| Server/Client split | `ServerCharacter.cs`, `ClientCharacter.cs` |
| Data-driven design | `ActionConfig.cs` |
| Object pooling | `NetworkObjectPool.cs` |
| UI patterns | `IPUIMediator.cs` |

---

## 🎯 Patterns to Remember

| Pattern | When to Use |
|---------|-------------|
| **State Machine** | Complex object states |
| **PubSub/Observer** | Decoupled communication |
| **Factory** | Object creation |
| **Object Pool** | Frequent spawn/destroy |
| **Mediator** | Complex UI coordination |
| **Strategy** | Swappable algorithms |
| **Logic/View Split** | All game objects |

---

## 💡 Remember

1. **Server is Authority** (or Logic class in offline games)
2. **Separate What from How** (interfaces from implementations)
3. **Data drives behavior** (configs, not hardcoded)
4. **Events for communication** (loose coupling)
5. **One class, one job** (single responsibility)

---

*Print this page and keep it next to your desk!*
