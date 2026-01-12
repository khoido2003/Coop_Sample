# 28: Scene Management & Loading

> **Purpose:** Understand how Boss Room manages scene transitions, the bootstrap pattern, and networked scene loading.

> 📍 **Key Files:**
> - [SceneBootstrapper.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Editor/SceneBootstrapper.cs) — Editor bootstrap helper
> - [ApplicationController.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/ApplicationLifecycle/ApplicationController.cs) — App entry point
> - [ServerBossRoomState.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameState/ServerBossRoomState.cs) — Scene transitions
> - [GameStateBehaviour.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/GameState/GameStateBehaviour.cs) — Scene-bound state

---

## Table of Contents

1. [Scene Architecture](#scene-architecture)
2. [Bootstrap Pattern](#bootstrap-pattern)
3. [Scene Flow](#scene-flow)
4. [Networked Scene Loading](#networked-scene-loading)
5. [Editor Scene Bootstrapping](#editor-scene-bootstrapping)
6. [GameStateBehaviour](#gamestatebehaviour)
7. [Best Practices](#best-practices)
8. [Key Takeaways](#key-takeaways)

---

## Scene Architecture

Boss Room uses a **bootstrap scene** pattern:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SCENE ARCHITECTURE                                 │
└──────────────────────────────────────────────────────────────────────┘

BOOTSTRAP SCENE (always loaded first)
┌─────────────────────────────────────────────────────────────────────┐
│  - ApplicationController (DontDestroyOnLoad)                        │
│  - ConnectionManager (DontDestroyOnLoad)                            │
│  - NetworkManager (DontDestroyOnLoad)                               │
│  - Dependency Injection container                                    │
└─────────────────────────────────────────────────────────────────────┘
         │
         │ Loads additively or replaces
         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   MainMenu      │ ──► │   CharSelect    │ ──► │    BossRoom     │
│   Scene         │     │   Scene         │     │    Scene        │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                         │
                                                         ▼
                                                ┌─────────────────┐
                                                │    PostGame     │
                                                │    Scene        │
                                                └─────────────────┘
```

### Why Bootstrap?

| Reason | Explanation |
|--------|-------------|
| **Persistent services** | NetworkManager survives scene loads |
| **Single initialization** | DI container created once |
| **Clean state** | Each scene starts fresh |
| **Consistent entry** | Always same initialization path |

---

## Bootstrap Pattern

### DontDestroyOnLoad Pattern

```csharp
public class ApplicationController : MonoBehaviour
{
    void Awake()
    {
        // Keep this object across scene loads
        DontDestroyOnLoad(gameObject);
        
        // Initialize core services
        InitializeDependencyInjection();
        InitializeNetworkManager();
    }

    void Start()
    {
        // Load first gameplay scene
        SceneManager.LoadScene("MainMenu");
    }
}
```

### Service Initialization Order

```csharp
// In Bootstrap scene's ApplicationController
async void Start()
{
    // 1. Initialize Unity Gaming Services (optional)
    await TryInitializeUGS();
    
    // 2. DI container is already configured in Awake
    
    // 3. Load main menu
    SceneManager.LoadScene("MainMenu");
}
```

---

## Scene Flow

### Complete Scene Transition Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SCENE TRANSITION FLOW                              │
└──────────────────────────────────────────────────────────────────────┘

1. Bootstrap → MainMenu
   └─► SceneManager.LoadScene("MainMenu")
   
2. MainMenu → CharSelect (on successful connection)
   └─► SceneLoaderWrapper.LoadScene("CharSelect", useNetworkSceneManager: true)
   
3. CharSelect → BossRoom (when all players ready)
   └─► SceneLoaderWrapper.LoadScene("BossRoom", useNetworkSceneManager: true)
   
4. BossRoom → PostGame (on boss defeat or party wipe)
   └─► SceneLoaderWrapper.LoadScene("PostGame", useNetworkSceneManager: true)
   
5. PostGame → CharSelect (play again)
   └─► SceneLoaderWrapper.LoadScene("CharSelect", useNetworkSceneManager: true)
   
6. Any → MainMenu (disconnect)
   └─► SceneLoaderWrapper.LoadScene("MainMenu", useNetworkSceneManager: false)
```

### Triggering Scene Transitions

```csharp
// In ServerCharSelectState - when all players ready
void OnAllPlayersReady()
{
    SceneLoaderWrapper.Instance.LoadScene("BossRoom", useNetworkSceneManager: true);
}

// In ServerBossRoomState - on game end
void EndGame()
{
    SceneLoaderWrapper.Instance.LoadScene("PostGame", useNetworkSceneManager: true);
}

// In OfflineState - returning to menu
public override void Enter()
{
    SceneLoaderWrapper.Instance.LoadScene("MainMenu", useNetworkSceneManager: false);
}
```

---

## Networked Scene Loading

### Server vs Client Control

```
┌──────────────────────────────────────────────────────────────────────┐
│                NETWORKED SCENE LOADING                                │
└──────────────────────────────────────────────────────────────────────┘

                    SERVER
                      │
                      │ LoadScene("BossRoom", networkSceneManager: true)
                      ▼
    ┌─────────────────────────────────────┐
    │     NetworkManager.SceneManager     │
    │                                     │
    │  1. Server loads scene first        │
    │  2. Notifies all clients            │
    │  3. Waits for clients to load       │
    │  4. Spawns network objects          │
    └───────────────────┬─────────────────┘
                        │
          ┌─────────────┼─────────────────┐
          ▼             ▼                 ▼
      CLIENT 1      CLIENT 2          CLIENT 3
      Loading       Loading           Loading
```

### Scene Load Completion Callback

```csharp
public override void OnNetworkSpawn()
{
    if (IsServer)
    {
        // Subscribe to scene load completion
        NetworkManager.SceneManager.OnLoadEventCompleted += OnLoadEventCompleted;
    }
}

void OnLoadEventCompleted(string sceneName, LoadSceneMode loadSceneMode, 
    List<ulong> clientsCompleted, List<ulong> clientsTimedOut)
{
    if (!InitialSpawnDone && loadSceneMode == LoadSceneMode.Single)
    {
        // All clients loaded - spawn players
        SpawnAllPlayers();
        InitialSpawnDone = true;
    }
}
```

---

## Editor Scene Bootstrapping

**File:** `Assets/Scripts/Editor/SceneBootstrapper.cs`

This editor script ensures play mode always starts from Bootstrap scene.

### Purpose

```
PROBLEM: Developer opens BossRoom scene and clicks Play
─────────────────────────────────────────────────────

Without bootstrapper:
  ❌ No NetworkManager (it's in Bootstrap)
  ❌ No DI container
  ❌ Null reference errors

With bootstrapper:
  ✅ Redirects to Bootstrap scene
  ✅ Initializes everything properly
  ✅ Returns to original scene after play
```

### How It Works

```csharp
[InitializeOnLoad]
public class SceneBootstrapper
{
    static SceneBootstrapper()
    {
        EditorApplication.playModeStateChanged += EditorApplicationOnplayModeStateChanged;
    }

    static void EditorApplicationOnplayModeStateChanged(PlayModeStateChange playModeStateChange)
    {
        if (playModeStateChange == PlayModeStateChange.ExitingEditMode)
        {
            // Save current scene to return later
            PreviousScene = EditorSceneManager.GetActiveScene().path;

            if (ShouldLoadBootstrapScene)
            {
                // Stop play mode
                EditorApplication.isPlaying = false;
                
                // Switch to Bootstrap scene
                EditorSceneManager.OpenScene(BootstrapScene);
                
                // Resume play mode
                EditorApplication.isPlaying = true;
            }
        }
        else if (playModeStateChange == PlayModeStateChange.EnteredEditMode)
        {
            // Return to the scene we were editing
            if (!string.IsNullOrEmpty(PreviousScene))
            {
                EditorSceneManager.OpenScene(PreviousScene);
            }
        }
    }
}
```

### Toggle Options

Menu: **Boss Room → Load Bootstrap Scene On Play** (toggle on/off)

---

## GameStateBehaviour

**File:** `Assets/Scripts/Gameplay/GameState/GameStateBehaviour.cs`

Scene-specific logic tied to NetworkBehaviour lifecycle.

```csharp
/// <summary>
/// A: They are driven implicitly by calling NetworkManager.SceneManager.LoadScene in server code.
/// </summary>
public abstract class GameStateBehaviour : NetworkBehaviour
{
    [SerializeField]
    protected NetworkObject m_NetworkObjectPrefab;

    public abstract GameState ActiveState { get; }

    public override void OnNetworkSpawn()
    {
        if (IsServer)
        {
            ConfigureServer();
        }
        
        if (IsClient)
        {
            ConfigureClient();
        }
    }

    protected abstract void ConfigureServer();
    protected abstract void ConfigureClient();
}
```

### Scene-Specific States

```csharp
// Each scene has a ServerXXXState:
public class ServerCharSelectState : GameStateBehaviour { }
public class ServerBossRoomState : GameStateBehaviour { }
public class ServerPostGameState : GameStateBehaviour { }
```

---

## Best Practices

### DO ✅

```csharp
// 1. Use NetworkManager.SceneManager for multiplayer scenes
SceneLoaderWrapper.LoadScene("BossRoom", useNetworkSceneManager: true);

// 2. Handle clients that fail to load
void OnLoadEventCompleted(List<ulong> clientsTimedOut)
{
    foreach (var clientId in clientsTimedOut)
    {
        Debug.LogWarning($"Client {clientId} timed out loading scene");
        NetworkManager.DisconnectClient(clientId);
    }
}

// 3. Use DontDestroyOnLoad for persistent objects
void Awake()
{
    DontDestroyOnLoad(gameObject);
}

// 4. Clean up subscriptions on scene unload
void OnDestroy()
{
    NetworkManager.SceneManager.OnLoadEventCompleted -= OnLoadEventCompleted;
}
```

### DON'T ❌

```csharp
// 1. Don't use SceneManager.LoadScene for networked scenes
SceneManager.LoadScene("BossRoom");  // ❌ Clients won't follow!

// 2. Don't spawn network objects before scene load completes
void Awake()
{
    SpawnPlayers();  // ❌ Scene might not be ready
}

// 3. Don't forget to restore previous scene in editor
// SceneBootstrapper handles this automatically

// 4. Don't assume scene is loaded synchronously
LoadScene("BossRoom");
SpawnPlayers();  // ❌ Scene not loaded yet!
```

---

## Key Takeaways

| Concept | Description |
|---------|-------------|
| **Bootstrap Scene** | First scene, persistent services |
| **DontDestroyOnLoad** | Services survive scene transitions |
| **NetworkManager.SceneManager** | Server controls scene loading |
| **SceneBootstrapper** | Editor tool for correct play mode |
| **OnLoadEventCompleted** | Callback when all clients loaded |
| **GameStateBehaviour** | Scene-specific networked state |

---

## Exercises

1. **Trace Scene Load:** Put breakpoints in scene transition code and trace from CharSelect to BossRoom.

2. **Add Scene:** Create a new "VictoryScreen" scene and add it to the scene flow.

3. **Handle Timeout:** Add better handling for clients that timeout during scene load.

---

> **Related Guides:**
> - [19_game_flow_deepdive.md](./19_game_flow_deepdive.md) — Complete game state flow
> - [10_connection_state_machine.md](./10_connection_state_machine.md) — Connection states
> - [03_networking_essentials.md](./03_networking_essentials.md) — NetworkManager basics
