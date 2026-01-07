# 03: Networking Essentials

> **Goal:** Understand multiplayer networking patterns that work for ANY online game. These principles are universal across game engines and languages.

---

## The Golden Rules of Multiplayer

### Rule 1: Server is Authority

**The server is ALWAYS right. Clients only REQUEST, never DEMAND.**

```
CLIENT: "I want to move to position X"
SERVER: "Approved, you are now at X" (or "Denied, you can't move there")
CLIENT: Updates visual to match server
```

**Why:**
- Prevents cheating (client can't lie about position)
- Consistent game state for all players
- Single source of truth

**In Boss Room:**
```csharp
// Client requests action
void OnAbilityButtonPressed() {
    // Send request to server (RPC)
    ServerCharacter.PlayActionRpc(actionData);
}

// Server validates and executes
[Rpc(SendTo.Server)]
void PlayActionRpc(ActionData data) {
    if (CanPerformAction(data)) {  // Validate!
        ExecuteAction(data);
    }
}
```

---

### Rule 2: Hide Latency with Prediction

**Start visual feedback BEFORE server confirms.**

```
T+0ms:   Player presses attack
         → Immediately play attack animation (anticipation)
T+50ms:  Request reaches server
T+55ms:  Server validates and executes
T+105ms: Confirmation reaches client
         → Client already showing attack, feels instant!
```

**In Boss Room:**
```csharp
// Client anticipates immediately
public void AnticipateActionClient(ClientCharacter client) {
    client.Animator.SetTrigger("Attack");  // Play immediately
    IsAnticipated = true;
}

// When server confirms
public void OnStartClient(ClientCharacter client) {
    if (!IsAnticipated) {
        // Only start animation if we weren't anticipating
        client.Animator.SetTrigger("Attack");
    }
}
```

---

### Rule 3: Minimize Network Traffic

**Send only what's necessary, as small as possible.**

| Instead of | Send |
|------------|------|
| "Player moved to (125.456, 0.123, 789.012)" | Delta from last position |
| Full object every frame | Only when changed |
| String "FireballSpell" | Int ID: 3 |
| Entire inventory | Just the changed slot |

**In Boss Room:**
```csharp
// ActionID is just an int, not a string
public struct ActionID : INetworkSerializeByMemcpy {
    public int ID;  // Small, fast to send
}

// NetworkVariables only sync when changed
public NetworkVariable<int> HitPoints = new();
// Setting same value = no network traffic
```

---

### Rule 4: Handle Disconnects Gracefully

**Network WILL fail. Plan for it.**

```
Scenarios to handle:
1. Client loses connection mid-game
2. Host quits while clients are connected
3. Network too slow (timeout)
4. Player joins while game in progress
```

**In Boss Room:**
```csharp
// Reconnection state machine
Offline → Connecting → Connected → (disconnect) → Reconnecting → Connected
                                                 ↓ (failed)
                                              Offline

// Reconnection attempts
while (attempts < maxAttempts) {
    TryReconnect();
    WaitForTimeout();
    attempts++;
}
```

---

## Synchronization Patterns

### Pattern 1: State Sync (NetworkVariable)

**Use for:** Values that change and all clients need to see.

```csharp
public class Enemy : NetworkBehaviour {
    // Automatically synced to all clients
    public NetworkVariable<int> Health = new();
    public NetworkVariable<bool> IsAggro = new();
    
    void TakeDamage(int amount) {
        if (!IsServer) return;  // Only server modifies
        Health.Value -= amount;
        // All clients automatically receive update
    }
}

// Clients react to changes
void Start() {
    Health.OnValueChanged += (old, current) => {
        UpdateHealthBar(current);
    };
}
```

**When to use:**
- Health, mana, score
- Object states (door open/closed)
- Player position (via NetworkTransform)

---

### Pattern 2: Remote Procedure Calls (RPC)

**Use for:** One-time events/actions.

```csharp
// Client → Server: Request something
[Rpc(SendTo.Server)]
void RequestAttackRpc(int targetId) {
    if (CanAttack(targetId)) {
        PerformAttack(targetId);
    }
}

// Server → All Clients: Tell everyone about event
[Rpc(SendTo.Everyone)]
void PlayExplosionRpc(Vector3 position) {
    Instantiate(explosionVFX, position, Quaternion.identity);
}

// Server → Specific Client: Private message
[Rpc(SendTo.ClientsAndHost)]
void ShowPrivateMessageRpc(string message, RpcParams rpcParams = default) {
    ShowMessage(message);
}
```

**When to use:**
- Ability activation
- Chat messages
- Visual effects
- Sound triggers

---

### Pattern 3: NetworkList

**Use for:** Collections that change (player list, inventory).

```csharp
public NetworkList<PlayerData> Players;

void Awake() {
    Players = new NetworkList<PlayerData>();
    Players.OnListChanged += OnPlayersChanged;
}

void OnPlayersChanged(NetworkListEvent<PlayerData> change) {
    switch (change.Type) {
        case NetworkListEvent<PlayerData>.EventType.Add:
            AddPlayerToUI(change.Value);
            break;
        case NetworkListEvent<PlayerData>.EventType.Remove:
            RemovePlayerFromUI(change.Value);
            break;
    }
}
```

---

### Pattern 4: Spawning Network Objects

```csharp
// Spawning (server only)
void SpawnEnemy(EnemyData data) {
    var enemy = Instantiate(enemyPrefab);
    enemy.GetComponent<NetworkObject>().Spawn();  // Makes it visible to all
}

// Despawning
void KillEnemy(NetworkObject enemy) {
    enemy.Despawn();  // Removes from all clients
}

// Spawning as player object (owned by specific client)
void SpawnPlayer(ulong clientId) {
    var player = Instantiate(playerPrefab);
    player.SpawnAsPlayerObject(clientId);  // Client owns this object
}
```

---

## Connection Approval Pattern

**Validate clients before they can join.**

```csharp
NetworkManager.ConnectionApprovalCallback += ApprovalCheck;

void ApprovalCheck(
    NetworkManager.ConnectionApprovalRequest request,
    NetworkManager.ConnectionApprovalResponse response)
{
    // Decode payload
    var payload = JsonUtility.FromJson<ConnectionPayload>(
        Encoding.UTF8.GetString(request.Payload));
    
    // Check 1: Server full?
    if (ConnectedClients.Count >= MaxPlayers) {
        response.Approved = false;
        response.Reason = "Server is full";
        return;
    }
    
    // Check 2: Valid version?
    if (payload.version != CurrentVersion) {
        response.Approved = false;
        response.Reason = "Version mismatch";
        return;
    }
    
    // Check 3: Banned?
    if (IsBanned(payload.playerId)) {
        response.Approved = false;
        response.Reason = "You are banned";
        return;
    }
    
    // All good!
    response.Approved = true;
    response.CreatePlayerObject = true;
}
```

---

## Quick Reference

| Need | Solution |
|------|----------|
| Sync a value | NetworkVariable |
| Sync a list | NetworkList |
| One-time event | RPC |
| Sync position | NetworkTransform |
| Sync physics | NetworkRigidbody |
| Sync animation | NetworkAnimator |
| Player input | Owner sends RPC to server |
| Visual effect | Server broadcasts RPC to all |

---

## Common Mistakes to Avoid

| Mistake | Why It's Bad | Fix |
|---------|--------------|-----|
| Client modifying server state | Creates desync | Only server modifies state |
| Sending every frame | Wastes bandwidth | Send only on change |
| Trusting client input | Enables cheating | Validate everything on server |
| No disconnect handling | Game breaks | State machine for connection |
| Large payloads | High latency | Use IDs, compression |

---

## Exercise

Answer these questions (check the code after):

1. How does Boss Room handle a client who tries to move where they shouldn't?
2. What happens if you spam the attack button? Does it send 100 RPCs?
3. How does reconnection restore your character data?
4. Find where connection approval happens - what checks are performed?
