# 24: Animation & VFX System

> **Purpose:** Understand how Boss Room handles character animations, animation-triggered effects, particle systems, and visual feedback.

> 📍 **Key Files:**
> - [AnimatorTriggeredSpecialFX.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/AnimationCallbacks/AnimatorTriggeredSpecialFX.cs) — Animation-triggered FX
> - [SpecialFXGraphic.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/VisualEffects/SpecialFXGraphic.cs) — VFX prefab wrapper
> - [AnimatorNodeHook.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/AnimationCallbacks/AnimatorNodeHook.cs) — State machine callbacks
> - [AnimatorFootstepSounds.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameplayObjects/AnimationCallbacks/AnimatorFootstepSounds.cs) — Footstep audio

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Animation-Triggered FX](#animation-triggered-fx)
3. [SpecialFXGraphic](#specialfxgraphic)
4. [Animator Node Hooks](#animator-node-hooks)
5. [Sound Effects](#sound-effects)
6. [Network Considerations](#network-considerations)
7. [Best Practices](#best-practices)
8. [Creating New Effects](#creating-new-effects)

---

## Architecture Overview

Boss Room separates animation/VFX from gameplay logic:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ANIMATION/VFX ARCHITECTURE                         │
└──────────────────────────────────────────────────────────────────────┘

SERVER (gameplay logic)              CLIENT (visuals only)
┌─────────────────────┐              ┌─────────────────────┐
│  ServerCharacter    │──network───►│  ClientCharacter    │
│  - Runs actions     │              │  - Animator         │
│  - Applies damage   │              │  - Particles        │
│  - Game state       │              │  - Sounds           │
└─────────────────────┘              └─────────────────────┘
          │                                    │
          │                                    ▼
          │                          ┌─────────────────────┐
          │                          │AnimatorTriggered    │
          │                          │SpecialFX            │
          │                          │ - Watches anim state│
          │                          │ - Spawns VFX        │
          │                          │ - Plays sounds      │
          └──────────────────────────└─────────────────────┘
```

**Key insight:** Server never touches animations or VFX directly. Client handles all visuals.

---

## Animation-Triggered FX

**File:** `Assets/Scripts/Gameplay/GameplayObjects/AnimationCallbacks/AnimatorTriggeredSpecialFX.cs`

This system spawns effects when entering/exiting specific animation states.

### Configuration

```csharp
[Serializable]
internal class AnimatorNodeEntryEvent
{
    // Which animation node triggers this
    [Tooltip("The name of a node in the Animator's state machine.")]
    public string m_AnimatorNodeName;
    
    [HideInInspector]
    public int m_AnimatorNodeNameHash;  // Pre-computed for performance

    // Particle effect
    public SpecialFXGraphic m_Prefab;
    public float m_PrefabSpawnDelaySeconds;
    public float m_PrefabCanBeAbortedUntilSecs;
    public Transform m_PrefabParent;       // Attach to bone
    public Vector3 m_PrefabParentOffset;
    public bool m_DeParentPrefab;          // Detach after spawn

    // Sound effect
    public AudioClip m_SoundEffect;
    public float m_SoundStartDelaySeconds;
    public float m_VolumeMultiplier = 1;
    public bool m_LoopSound = false;
}
```

### State Enter Handling

```csharp
public void OnStateEnter(Animator animator, AnimatorStateInfo stateInfo, int layerIndex)
{
    m_ActiveNodes.Add(stateInfo.shortNameHash);

    // Check each configured event
    foreach (var info in m_EventsOnNodeEntry)
    {
        if (info.m_AnimatorNodeNameHash == stateInfo.shortNameHash)
        {
            // Spawn FX (with optional delay)
            if (info.m_Prefab)
            {
                StartCoroutine(CoroPlayStateEnterFX(info));
            }
            
            // Play sound (with optional delay)
            if (info.m_SoundEffect)
            {
                StartCoroutine(CoroPlayStateEnterSound(info));
            }
        }
    }
}
```

### Delayed FX Spawning

```csharp
private IEnumerator CoroPlayStateEnterFX(AnimatorNodeEntryEvent eventInfo)
{
    // Wait for spawn delay
    if (eventInfo.m_PrefabSpawnDelaySeconds > 0)
        yield return new WaitForSeconds(eventInfo.m_PrefabSpawnDelaySeconds);

    // Check if still in this animation state
    if (!m_ActiveNodes.Contains(eventInfo.m_AnimatorNodeNameHash))
        yield break;

    // Spawn at parent (bone) or character root
    Transform parent = eventInfo.m_PrefabParent ?? m_ClientCharacter.transform;
    var instantiatedFX = Instantiate(eventInfo.m_Prefab, parent);
    instantiatedFX.transform.localPosition += eventInfo.m_PrefabParentOffset;

    // Optionally detach from parent (for effects that should stay in world space)
    if (eventInfo.m_DeParentPrefab)
    {
        instantiatedFX.transform.SetParent(null);
    }

    // Handle abortion if we leave the state early
    if (eventInfo.m_PrefabCanBeAbortedUntilSecs > 0)
    {
        // ... watch for state exit and call Shutdown() ...
    }
}
```

---

## SpecialFXGraphic

**File:** `Assets/Scripts/VisualEffects/SpecialFXGraphic.cs`

A wrapper for VFX prefabs that handles lifecycle management.

### Modes

1. **Manual shutdown:** Effect runs until `Shutdown()` is called
2. **Auto shutdown:** Effect self-destructs after configured time

```csharp
public class SpecialFXGraphic : MonoBehaviour
{
    [SerializeField]
    public List<ParticleSystem> m_ParticleSystemsToTurnOffOnShutdown;

    // -1 = manual shutdown, >0 = auto after N seconds
    [SerializeField]
    private float m_AutoShutdownTime = -1;

    // How long to wait after shutdown before destroy
    // 0 = immediate, -1 = wait for particles to finish
    [SerializeField]
    private float m_PostShutdownSelfDestructTime = -1;

    [SerializeField]
    bool m_StayAtSpawnRotation;

    private bool m_IsShutdown = false;
}
```

### Shutdown Flow

```csharp
public void Shutdown()
{
    if (!m_IsShutdown)
    {
        // Stop all particle systems
        foreach (var particleSystem in m_ParticleSystemsToTurnOffOnShutdown)
        {
            if (particleSystem)
            {
                particleSystem.Stop();
            }
        }

        // Self-destruct logic
        if (m_PostShutdownSelfDestructTime >= 0)
        {
            // Fixed delay
            Destroy(gameObject, m_PostShutdownSelfDestructTime);
        }
        else if (m_PostShutdownSelfDestructTime == -1)
        {
            // Wait for particles to finish
            StartCoroutine(CoroWaitForParticlesToEnd());
        }

        m_IsShutdown = true;
    }
}
```

### Waiting for Particles

```csharp
private IEnumerator CoroWaitForParticlesToEnd()
{
    bool foundAliveParticles;
    do
    {
        yield return new WaitForEndOfFrame();
        foundAliveParticles = false;
        foreach (var particleSystem in m_ParticleSystemsToTurnOffOnShutdown)
        {
            if (particleSystem.IsAlive())
            {
                foundAliveParticles = true;
            }
        }
    } while (foundAliveParticles);

    Destroy(gameObject);
}
```

---

## Animator Node Hooks

**File:** `Assets/Scripts/Gameplay/GameplayObjects/AnimationCallbacks/AnimatorNodeHook.cs`

This is a `StateMachineBehaviour` attached to animator states to fire callbacks.

```csharp
public class AnimatorNodeHook : StateMachineBehaviour
{
    AnimatorTriggeredSpecialFX[] m_SpecialFX;

    public override void OnStateEnter(Animator animator, AnimatorStateInfo stateInfo, int layerIndex)
    {
        // Find and cache FX handlers on first call
        if (m_SpecialFX == null)
        {
            m_SpecialFX = animator.GetComponentsInChildren<AnimatorTriggeredSpecialFX>();
        }

        // Notify all handlers
        foreach (var fx in m_SpecialFX)
        {
            fx.OnStateEnter(animator, stateInfo, layerIndex);
        }
    }

    public override void OnStateExit(Animator animator, AnimatorStateInfo stateInfo, int layerIndex)
    {
        foreach (var fx in m_SpecialFX)
        {
            fx.OnStateExit(animator, stateInfo, layerIndex);
        }
    }
}
```

### Setup in Animator

1. Select an animation state in the Animator window
2. Add `AnimatorNodeHook` as a State Machine Behaviour
3. The hook will automatically find and notify `AnimatorTriggeredSpecialFX` components

---

## Sound Effects

### Animation-Triggered Sounds

```csharp
private IEnumerator CoroPlayStateEnterSound(AnimatorNodeEntryEvent eventInfo)
{
    // Wait for delay
    if (eventInfo.m_SoundStartDelaySeconds > 0)
        yield return new WaitForSeconds(eventInfo.m_SoundStartDelaySeconds);

    // Check if still in animation state
    if (!m_ActiveNodes.Contains(eventInfo.m_AnimatorNodeNameHash))
        yield break;

    if (!eventInfo.m_LoopSound)
    {
        // One-shot sound
        m_AudioSources[0].PlayOneShot(eventInfo.m_SoundEffect, eventInfo.m_VolumeMultiplier);
    }
    else
    {
        // Looping sound (stops when leaving state)
        AudioSource audioSource = GetAudioSourceForLooping();
        if (!audioSource) yield break;
        
        audioSource.volume = eventInfo.m_VolumeMultiplier;
        audioSource.loop = true;
        audioSource.clip = eventInfo.m_SoundEffect;
        audioSource.Play();
        
        // Stop when leaving animation state
        while (m_ActiveNodes.Contains(eventInfo.m_AnimatorNodeNameHash) && audioSource.isPlaying)
        {
            yield return new WaitForFixedUpdate();
        }
        audioSource.Stop();
    }
}
```

### Footstep Sounds

**File:** `AnimatorFootstepSounds.cs`

Plays footstep sounds based on animation events or root motion.

```csharp
public class AnimatorFootstepSounds : MonoBehaviour
{
    [SerializeField] AudioClip[] m_FootstepSounds;
    [SerializeField] AudioSource m_AudioSource;
    
    // Called from animation event
    public void PlayFootstep()
    {
        if (m_FootstepSounds.Length > 0)
        {
            var clip = m_FootstepSounds[Random.Range(0, m_FootstepSounds.Length)];
            m_AudioSource.PlayOneShot(clip);
        }
    }
}
```

---

## Network Considerations

### Client-Side Only

All animation and VFX code runs on clients only:

```csharp
// In ClientCharacter
public override void OnNetworkSpawn()
{
    if (!IsClient)
    {
        enabled = false;  // Server doesn't run client visuals
        return;
    }
    
    // Subscribe to server state changes
    m_ServerCharacter.ActionRequestData.OnValueChanged += OnActionChanged;
}
```

### Triggering from Server State

```
SERVER                              CLIENT
┌─────────────────┐                ┌─────────────────┐
│ Action starts   │───network───►│ Receive action  │
│ Sets ActionData │      RPC      │ data            │
└─────────────────┘                └────────┬────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │ ClientActionViz │
                                   │ Triggers anim   │
                                   └────────┬────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │ Animator state  │
                                   │ changes         │
                                   └────────┬────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │ AnimatorNodeHook│
                                   │ fires callback  │
                                   └────────┬────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │ AnimatorTrigged │
                                   │ SpecialFX spawns│
                                   │ particles       │
                                   └─────────────────┘
```

---

## Best Practices

### Performance

```csharp
// 1. Pre-compute animation name hashes
private void OnValidate()
{
    foreach (var info in m_EventsOnNodeEntry)
    {
        info.m_AnimatorNodeNameHash = Animator.StringToHash(info.m_AnimatorNodeName);
    }
}

// 2. Use HashSet for O(1) state lookups
private HashSet<int> m_ActiveNodes = new HashSet<int>();

// 3. Cache component references
[SerializeField] private Animator m_Animator;
```

### Effect Cleanup

```csharp
// Always use SpecialFXGraphic for VFX prefabs
// It handles proper cleanup after particles finish

// For pooling (mobile), modify SpecialFXGraphic to return to pool instead of Destroy
```

### Editor Validation

```csharp
// AnimatorTriggeredSpecialFX includes editor validation
// Use "Validate Node Names" button to check configuration
```

---

## Creating New Effects

### Step 1: Create VFX Prefab

1. Create particle system
2. Add `SpecialFXGraphic` component
3. Configure auto-shutdown time or leave at -1 for manual
4. Click "Auto-Add All Particle Systems" button

### Step 2: Configure Animation Trigger

1. Add `AnimatorNodeHook` to the animation state
2. In `AnimatorTriggeredSpecialFX`, add new entry:
   - Set `AnimatorNodeName` (exact state name)
   - Assign VFX prefab
   - Set spawn delay if needed
   - Optionally set sound effect

### Step 3: Test

1. Play game and trigger the animation
2. Watch for Console errors about missing node names
3. Use "Validate Node Names" button in Inspector

---

## Key Takeaways

| Concept | Description |
|---------|-------------|
| **Separation** | Server does logic, client does visuals |
| **AnimatorNodeHook** | Fires callbacks on state enter/exit |
| **AnimatorTriggeredSpecialFX** | Maps states to effects |
| **SpecialFXGraphic** | Manages VFX lifecycle |
| **Hash caching** | Pre-compute animation state hashes |
| **Delayed spawn** | Effects can wait before spawning |
| **Abortion** | Effects can be cancelled if state exits early |

---

> **Related Guides:**
> - [18_character_system_deepdive.md](./18_character_system_deepdive.md) — Server/Client character split
> - [09_action_system_deepdive.md](./09_action_system_deepdive.md) — Action visuals
> - [16_performance_patterns.md](./16_performance_patterns.md) — Object pooling for VFX
