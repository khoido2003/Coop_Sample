# 22: UI Architecture Deep Dive

> **Purpose:** Understand how Boss Room structures its UI using mediators, event-driven updates, state binding, and reusable components.

> 📍 **Key Files:**
> - [IPUIMediator.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/UI/IPUIMediator.cs) — Mediator pattern example
> - [HeroActionBar.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/UI/HeroActionBar.cs) — Action bar with dynamic updates
> - [PartyHUD.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/UI/PartyHUD.cs) — Party health display
> - [UIStateDisplayHandler.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/UI/UIStateDisplayHandler.cs) — State-driven UI
> - [PopupManager.cs](file:///d:/unity_projects/com.unity.multiplayer.samples.coop/Assets/Scripts/Gameplay/UI/PopupManager.cs) — Modal popups

---

## Table of Contents

1. [UI Architecture Principles](#ui-architecture-principles)
2. [Mediator Pattern](#mediator-pattern)
3. [Event-Driven UI Updates](#event-driven-ui-updates)
4. [Hero Action Bar](#hero-action-bar)
5. [State-Driven UI](#state-driven-ui)
6. [Popup System](#popup-system)
7. [Tooltip System](#tooltip-system)
8. [Best Practices](#best-practices)
9. [Creating New UI](#creating-new-ui)

---

## UI Architecture Principles

Boss Room's UI follows key architectural patterns:

```
┌──────────────────────────────────────────────────────────────────────┐
│                      UI ARCHITECTURE                                  │
└──────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────┐
                    │      Game Systems       │
                    │  (ConnectionManager,    │
                    │   ServerCharacter, etc) │
                    └───────────┬─────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
            ┌───────────────┐       ┌───────────────┐
            │   Messages    │       │ NetworkVars   │
            │  (PubSub)     │       │ OnValueChange │
            └───────┬───────┘       └───────┬───────┘
                    │                       │
                    └───────────┬───────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │       UI MEDIATOR       │
                    │  Coordinates components │
                    └───────────┬─────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
    ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
    │  UI Element   │   │  UI Element   │   │  UI Element   │
    │  (Button)     │   │  (InputField) │   │  (Text)       │
    └───────────────┘   └───────────────┘   └───────────────┘
```

### Key Principles

1. **UI doesn't call game logic directly** — uses mediators
2. **UI listens to game state** — doesn't poll for changes
3. **One mediator per screen** — coordinates multiple elements
4. **Reusable components** — buttons, tooltips, health bars

---

## Mediator Pattern

The **Mediator Pattern** prevents UI elements from knowing about each other or game systems.

### IPUIMediator Example

**File:** `Assets/Scripts/Gameplay/UI/IPUIMediator.cs`

```csharp
public class IPUIMediator : MonoBehaviour
{
    // UI element references
    [SerializeField] TextMeshProUGUI m_PlayerNameLabel;
    [SerializeField] IPJoiningUI m_IPJoiningUI;
    [SerializeField] IPHostingUI m_IPHostingUI;
    [SerializeField] GameObject m_SignInSpinner;
    [SerializeField] IPConnectionWindow m_IPConnectionWindow;

    // Injected dependencies (NOT FindObjectOfType!)
    [Inject] ConnectionManager m_ConnectionManager;
    ISubscriber<ConnectStatus> m_ConnectStatusSubscriber;

    [Inject]
    void InjectDependencies(ISubscriber<ConnectStatus> connectStatusSubscriber)
    {
        m_ConnectStatusSubscriber = connectStatusSubscriber;
        m_ConnectStatusSubscriber.Subscribe(OnConnectStatusMessage);
    }
}
```

### Mediator Responsibilities

```csharp
// 1. Coordinate between UI panels
public void ToggleJoinIPUI()
{
    m_IPJoiningUI.Show();
    m_IPHostingUI.Hide();
    m_JoinTabButtonHighlightTinter.SetToColor(1);  // Visual feedback
    m_HostTabButtonHighlightTinter.SetToColor(0);
}

// 2. Bridge UI actions to game systems
public void HostIPRequest(string ip, string port)
{
    // Validate and normalize input
    int.TryParse(port, out var portNum);
    if (portNum <= 0) portNum = k_DefaultPort;
    ip = string.IsNullOrEmpty(ip) ? k_DefaultIP : ip;

    // Show loading state
    m_SignInSpinner.SetActive(true);
    
    // Call game system
    m_ConnectionManager.StartHostIp(m_PlayerNameLabel.text, ip, portNum);
}

// 3. React to game events
void OnConnectStatusMessage(ConnectStatus connectStatus)
{
    // Hide spinner when connection attempt completes
    DisableSignInSpinner();
}

// 4. Handle cleanup
void OnDestroy()
{
    m_ConnectStatusSubscriber?.Unsubscribe(OnConnectStatusMessage);
}
```

### Input Validation

```csharp
// Sanitize IP input (only numbers and dots)
public static string SanitizeIP(string dirtyString)
{
    return Regex.Replace(dirtyString, "[^0-9.]", "");
}

// Sanitize port input (only numbers)
public static string SanitizePort(string dirtyString)
{
    return Regex.Replace(dirtyString, "[^0-9]", "");
}

// Validate combined IP:Port
public static bool AreIpAddressAndPortValid(string ipAddress, string port)
{
    var portValid = ushort.TryParse(port, out var portNum);
    return portValid && NetworkEndpoint.TryParse(ipAddress, portNum, out _);
}
```

---

## Event-Driven UI Updates

UI listens to game state changes rather than polling.

### Pattern: Subscribe in Start, Unsubscribe in OnDestroy

```csharp
public class ConnectionStatusUI : MonoBehaviour
{
    ISubscriber<ConnectStatus> m_StatusSubscriber;

    [Inject]
    void InjectDependencies(ISubscriber<ConnectStatus> subscriber)
    {
        m_StatusSubscriber = subscriber;
    }

    void Start()
    {
        m_StatusSubscriber.Subscribe(OnStatusChanged);
    }

    void OnDestroy()
    {
        m_StatusSubscriber?.Unsubscribe(OnStatusChanged);
    }

    void OnStatusChanged(ConnectStatus status)
    {
        // Update UI based on status
        UpdateStatusText(status);
    }
}
```

### Pattern: NetworkVariable Binding

```csharp
public class UIHealth : MonoBehaviour
{
    [SerializeField] Slider m_HealthSlider;
    ServerCharacter m_Character;

    public void Initialize(ServerCharacter character)
    {
        m_Character = character;
        
        // Subscribe to NetworkVariable changes
        m_Character.NetHealthState.HitPoints.OnValueChanged += OnHealthChanged;
        
        // Set initial value
        UpdateUI(m_Character.NetHealthState.HitPoints.Value);
    }

    void OnDestroy()
    {
        if (m_Character != null)
        {
            m_Character.NetHealthState.HitPoints.OnValueChanged -= OnHealthChanged;
        }
    }

    void OnHealthChanged(int oldValue, int newValue)
    {
        UpdateUI(newValue);
    }

    void UpdateUI(int health)
    {
        m_HealthSlider.value = health / (float)m_Character.CharacterClass.BaseHP.Value;
    }
}
```

---

## Hero Action Bar

**File:** `Assets/Scripts/Gameplay/UI/HeroActionBar.cs`

The action bar demonstrates dynamic UI that responds to player character state.

### Button Registration

```csharp
void Awake()
{
    // Create button info for each button type
    m_ButtonInfo = new Dictionary<ActionButtonType, ActionButtonInfo>()
    {
        [ActionButtonType.BasicAction] = new ActionButtonInfo(ActionButtonType.BasicAction, m_BasicActionButton, this),
        [ActionButtonType.Special1] = new ActionButtonInfo(ActionButtonType.Special1, m_SpecialAction1Button, this),
        [ActionButtonType.Special2] = new ActionButtonInfo(ActionButtonType.Special2, m_SpecialAction2Button, this),
        [ActionButtonType.EmoteBar] = new ActionButtonInfo(ActionButtonType.EmoteBar, m_EmoteBarButton, this),
    };

    // Subscribe to player spawn events
    ClientPlayerAvatar.LocalClientSpawned += RegisterInputSender;
    ClientPlayerAvatar.LocalClientDespawned += DeregisterInputSender;
}
```

### Dynamic Button Updates

```csharp
void RegisterInputSender(ClientPlayerAvatar clientPlayerAvatar)
{
    m_InputSender = clientPlayerAvatar.GetComponent<ClientInputSender>();
    m_InputSender.action1ModifiedCallback += Action1ModifiedCallback;

    // Get actions from input sender
    Action action1 = null;
    if (m_InputSender.actionState1 != null)
    {
        GameDataSource.Instance.TryGetActionPrototypeByID(
            m_InputSender.actionState1.actionID, out action1);
    }

    // Update button appearance
    UpdateActionButton(m_ButtonInfo[ActionButtonType.BasicAction], action1);
    // ... repeat for other buttons
}

void UpdateActionButton(ActionButtonInfo buttonInfo, Action action, bool isClickable = true)
{
    if (action == null)
    {
        buttonInfo.Button.gameObject.SetActive(false);
        return;
    }
    
    buttonInfo.Button.gameObject.SetActive(true);
    buttonInfo.Button.interactable = isClickable;
    buttonInfo.Button.image.sprite = action.Config.Icon;      // From ActionConfig
    buttonInfo.Tooltip.SetText(action.Config.Description);    // Tooltip text
    buttonInfo.CurAction = action;
}
```

### Click Handling

```csharp
void OnButtonClickedDown(ActionButtonType buttonType)
{
    if (buttonType == ActionButtonType.EmoteBar)
        return;  // Special handling
        
    if (m_InputSender == null)
        return;

    // Request action through input sender
    m_InputSender.RequestAction(
        m_ButtonInfo[buttonType].CurAction.ActionID, 
        SkillTriggerStyle.UI);
}

void OnButtonClickedUp(ActionButtonType buttonType)
{
    if (buttonType == ActionButtonType.EmoteBar)
    {
        // Toggle emote panel
        m_EmotePanel.SetActive(!m_EmotePanel.activeSelf);
        return;
    }
    
    // Complete action
    m_InputSender.RequestAction(
        m_ButtonInfo[buttonType].CurAction.ActionID, 
        SkillTriggerStyle.UIRelease);
}
```

---

## State-Driven UI

**File:** `Assets/Scripts/Gameplay/UI/UIStateDisplayHandler.cs`

This pattern shows/hides UI elements based on character state.

```csharp
public class UIStateDisplayHandler : MonoBehaviour
{
    [SerializeField] List<UIStateDisplay> m_StateDisplays;
    
    ServerCharacter m_Character;

    public void Initialize(ServerCharacter character)
    {
        m_Character = character;
        m_Character.NetLifeState.LifeState.OnValueChanged += OnLifeStateChanged;
        
        UpdateStateDisplay(m_Character.NetLifeState.LifeState.Value);
    }

    void OnLifeStateChanged(LifeState oldState, LifeState newState)
    {
        UpdateStateDisplay(newState);
    }

    void UpdateStateDisplay(LifeState state)
    {
        foreach (var display in m_StateDisplays)
        {
            // Show/hide based on state
            display.gameObject.SetActive(display.ApplicableState == state);
        }
    }
}
```

---

## Popup System

**File:** `Assets/Scripts/Gameplay/UI/PopupManager.cs`

Centralized popup/modal management.

```csharp
public class PopupManager : MonoBehaviour
{
    [SerializeField] GameObject m_PopupPrefab;
    [SerializeField] Transform m_PopupContainer;
    
    PopupPanel m_CurrentPopup;

    public void ShowPopup(string title, string message, string buttonText, Action onConfirm)
    {
        // Close existing popup
        if (m_CurrentPopup != null)
        {
            ClosePopup();
        }
        
        // Create new popup
        var popupGO = Instantiate(m_PopupPrefab, m_PopupContainer);
        m_CurrentPopup = popupGO.GetComponent<PopupPanel>();
        m_CurrentPopup.Initialize(title, message, buttonText, () => {
            onConfirm?.Invoke();
            ClosePopup();
        });
    }

    public void ClosePopup()
    {
        if (m_CurrentPopup != null)
        {
            Destroy(m_CurrentPopup.gameObject);
            m_CurrentPopup = null;
        }
    }
}
```

---

## Tooltip System

**File:** `Assets/Scripts/Gameplay/UI/UITooltipDetector.cs`

Attach to any UI element that needs a tooltip.

```csharp
public class UITooltipDetector : MonoBehaviour, IPointerEnterHandler, IPointerExitHandler
{
    [SerializeField] string m_TooltipText;
    
    UITooltipPopup m_TooltipPopup;

    void Awake()
    {
        m_TooltipPopup = FindObjectOfType<UITooltipPopup>();
    }

    public void SetText(string text)
    {
        m_TooltipText = text;
    }

    public void OnPointerEnter(PointerEventData eventData)
    {
        if (!string.IsNullOrEmpty(m_TooltipText))
        {
            m_TooltipPopup.ShowTooltip(m_TooltipText, transform.position);
        }
    }

    public void OnPointerExit(PointerEventData eventData)
    {
        m_TooltipPopup.HideTooltip();
    }
}
```

---

## Best Practices

### DO ✅

```csharp
// 1. Use DI for dependencies
[Inject] ConnectionManager m_ConnectionManager;

// 2. Subscribe/Unsubscribe properly
void Start() { subscriber.Subscribe(OnEvent); }
void OnDestroy() { subscriber?.Unsubscribe(OnEvent); }

// 3. Use CanvasGroup for show/hide
public void Show()
{
    m_CanvasGroup.alpha = 1f;
    m_CanvasGroup.interactable = true;
    m_CanvasGroup.blocksRaycasts = true;
}

// 4. Cache component references
[SerializeField] TextMeshProUGUI m_Label;  // Not GetComponent every frame
```

### DON'T ❌

```csharp
// 1. Don't call game systems directly from button OnClick
void OnButtonClick()
{
    // BAD: Direct reference to game system
    FindObjectOfType<ConnectionManager>().StartHost();
}

// 2. Don't poll for state
void Update()
{
    // BAD: Checking every frame
    if (player.Health != lastHealth)
    {
        UpdateHealthBar();
    }
}

// 3. Don't forget to unsubscribe
void Start()
{
    // BAD: No matching Unsubscribe
    subscriber.Subscribe(OnEvent);
}
```

---

## Creating New UI

### Step 1: Create the Canvas/Prefab

1. Create UI elements in Unity Editor
2. Add `CanvasGroup` for show/hide control
3. Reference child elements via `[SerializeField]`

### Step 2: Create Mediator Script

```csharp
public class MyUIMediator : MonoBehaviour
{
    // 1. References to UI elements
    [SerializeField] Button m_ConfirmButton;
    [SerializeField] TextMeshProUGUI m_StatusText;
    [SerializeField] CanvasGroup m_CanvasGroup;
    
    // 2. Injected dependencies
    [Inject] ISubscriber<MyMessage> m_Subscriber;
    
    // 3. Subscribe on enable
    void OnEnable()
    {
        m_ConfirmButton.onClick.AddListener(OnConfirmClicked);
        m_Subscriber.Subscribe(OnMessageReceived);
    }
    
    // 4. Unsubscribe on disable/destroy
    void OnDisable()
    {
        m_ConfirmButton.onClick.RemoveListener(OnConfirmClicked);
        m_Subscriber?.Unsubscribe(OnMessageReceived);
    }
    
    // 5. Handle UI events
    void OnConfirmClicked()
    {
        // Coordinate with game systems
    }
    
    // 6. Handle game events
    void OnMessageReceived(MyMessage msg)
    {
        m_StatusText.text = msg.Status;
    }
    
    // 7. Show/Hide via CanvasGroup
    public void Show() { /* set alpha, interactable, blocksRaycasts */ }
    public void Hide() { /* clear alpha, interactable, blocksRaycasts */ }
}
```

### Step 3: Register in DI (if needed)

```csharp
// In ApplicationController.Configure()
builder.RegisterComponentInHierarchy<MyUIMediator>();
```

---

## Key Takeaways

| Concept | Description |
|---------|-------------|
| **Mediator Pattern** | One script coordinates multiple UI elements |
| **Event Binding** | UI listens to PubSub and NetworkVariable changes |
| **No Polling** | Never check state in Update(), subscribe instead |
| **CanvasGroup** | Use for clean show/hide with fade, interactability |
| **Cleanup** | Always unsubscribe in OnDestroy |
| **DI for Systems** | Inject ConnectionManager, not FindObjectOfType |

---

> **Related Guides:**
> - [04_design_patterns.md](./04_design_patterns.md) — Mediator pattern details
> - [11_infrastructure_patterns.md](./11_infrastructure_patterns.md) — PubSub system
> - [03_networking_essentials.md](./03_networking_essentials.md) — NetworkVariable binding
