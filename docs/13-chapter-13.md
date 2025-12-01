## Chapter 1.13: Global State Management - Data Ownership

In Sigilborne's hybrid architecture, maintaining a consistent and authoritative game state is paramount. This chapter formalizes the principle of **data ownership**: the C# Brain is the sole owner of all authoritative game data, while the GDScript Body never "owns" data, it only displays it. This strict separation, as outlined in TDD 01.4, prevents data desynchronization, simplifies debugging, and reinforces the Brain & Body paradigm.

### 1. The Principle of Authoritative State

*   **Brain (C#)**: Owns the **Model** (the data) and the **Controller** (the logic that changes the data).
    *   *Examples*: Player's actual health, inventory contents, NPC positions, world time, faction relationships, quest states.
    *   *Rule*: All changes to the game state *must* originate from or be validated by the Brain.
*   **Body (GDScript)**: Owns the **View**. It receives updates from the Brain and renders them.
    *   *Examples*: Player's visual health bar, inventory UI, animated NPC sprite at an interpolated position, UI display of world time.
    *   *Rule*: The Body never modifies the authoritative game state directly. It only requests changes from the Brain (e.g., via GDScript-to-C# method calls for player input).

This single source of truth is crucial for determinism, especially if we were to introduce multiplayer.

### 2. Global State Examples and Their Brain Owners

Let's look at key game data and identify their authoritative owners within our C# Brain, as per TDD 01.4.

| Data Type        | Owner (C# System)         | Example Data                               |
| :--------------- | :------------------------ | :----------------------------------------- |
| **Player Stats** | `BiologicalSystem` (Soon) | `Health: 100`, `Chakra: 50`, `Stamina: 75` |
| **Inventory**    | `InventorySystem` (Soon)  | `[ItemID: "iron_sword", Qty: 1]`           |
| **World Time**   | `TimeSystem` (Implemented) | `Day: 4`, `Hour: 14`, `Minute: 0`          |
| **NPC Positions**| `TransformSystem` (Implemented) | `EntityID(1).Position: (40, 20)`           |
| **Weather**      | `WeatherSystem` (Soon)    | `CurrentWeather: Rain`, `Temperature: 15`  |
| **Quest State**  | `QuestSystem` (Soon)      | `QuestID("main_01").Status: Active`        |

The `GameManager.Instance` acts as the central hub to access these owning systems.

### 3. Implementing a Basic Player Stat System (Brain)

To demonstrate data ownership, let's create a very simple `PlayerStatSystem` in C# that manages the player's health. This will be an early step towards our `BiologicalSystem` (TDD 03.2).

1.  Create a new folder `res://_Brain/Systems/Biology/`.
2.  Create a new C# script `res://_Brain/Systems/Biology/PlayerStatSystem.cs`:

```csharp
// _Brain/Systems/Biology/PlayerStatSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;

namespace Sigilborne.Systems.Biology
{
    /// <summary>
    /// Stores the core stats for the player entity.
    /// This is a component-like data structure managed by the PlayerStatSystem.
    /// </summary>
    public struct PlayerStats
    {
        public EntityID PlayerID;
        public float Health;
        public float MaxHealth;
        public float Chakra;
        public float MaxChakra;
        public float Stamina;
        public float MaxStamina;

        public PlayerStats(EntityID id, float maxHealth, float maxChakra, float maxStamina)
        {
            PlayerID = id;
            MaxHealth = maxHealth;
            Health = maxHealth;
            MaxChakra = maxChakra;
            Chakra = maxChakra;
            MaxStamina = maxStamina;
            Stamina = maxStamina;
        }

        public override string ToString()
        {
            return $"Health: {Health}/{MaxHealth}, Chakra: {Chakra}/{MaxChakra}, Stamina: {Stamina}/{MaxStamina}";
        }
    }

    /// <summary>
    /// Manages the player's core biological stats.
    /// This is the authoritative owner of the player's health, chakra, etc.
    /// </summary>
    public class PlayerStatSystem
    {
        private PlayerStats _playerStats;
        private EventBus _eventBus;
        private EntityManager _entityManager;

        public PlayerStatSystem(EventBus eventBus, EntityManager entityManager)
        {
            _eventBus = eventBus;
            _entityManager = entityManager;
            GD.Print("PlayerStatSystem: Initialized.");
            
            // Listen for when the player entity is actually spawned
            _eventBus.OnEntitySpawned += OnEntitySpawned;
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player)
            {
                // Initialize player stats when the player entity is created
                _playerStats = new PlayerStats(id, 100f, 50f, 75f);
                GD.Print($"PlayerStatSystem: Initialized stats for player {id}: {_playerStats}");
                
                // Publish an event for the Body to display the initial health
                _eventBus.Publish(new PlayerHealthChangedEvent { PlayerID = id, NewHealth = _playerStats.Health, MaxHealth = _playerStats.MaxHealth });
            }
        }

        /// <summary>
        /// Retrieves a copy of the player's current stats.
        /// </summary>
        public PlayerStats GetPlayerStats()
        {
            return _playerStats;
        }

        /// <summary>
        /// Authoritatively applies damage to the player's health.
        /// </summary>
        public void TakeDamage(float amount)
        {
            if (_playerStats.Health <= 0) return; // Already dead

            _playerStats.Health -= amount;
            if (_playerStats.Health < 0) _playerStats.Health = 0;

            GD.Print($"PlayerStatSystem: Player {_playerStats.PlayerID} took {amount} damage. New Health: {_playerStats.Health}");

            // Publish an event for the Body (UI) to update the health bar
            _eventBus.Publish(new PlayerHealthChangedEvent { PlayerID = _playerStats.PlayerID, NewHealth = _playerStats.Health, MaxHealth = _playerStats.MaxHealth });

            if (_playerStats.Health == 0)
            {
                GD.Print($"PlayerStatSystem: Player {_playerStats.PlayerID} has died!");
                _eventBus.Publish(new PlayerDiedEvent { PlayerID = _playerStats.PlayerID });
            }
        }

        // --- Helper Events for Body Sync ---
        public struct PlayerHealthChangedEvent { public EntityID PlayerID; public float NewHealth; public float MaxHealth; }
        public struct PlayerDiedEvent { public EntityID PlayerID; }
    }
}
```

### 4. Integrating `PlayerStatSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Biology;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `PlayerStatSystem` property.
3.  Initialize `PlayerStatSystem` in `InitializeSystems()`.

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems;
using Sigilborne.Systems.Biology; // Add this using directive
using Sigilborne.Utils;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    public TimeSystem Time { get; private set; }
    public EventBus Events { get; private set; }
    public WorldSimulation World { get; private set; }
    public EntityManager Entities { get; private set; }
    public TransformSystem Transforms { get; private set; }
    public JobSystem Jobs { get; private set; }
    public PlayerStatSystem PlayerStats { get; private set; } // Add PlayerStatSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        
        // --- Test Scene Loading ---
        // ... (existing SceneLoader call) ...
        // --- End Test Scene Loading ---

        // --- Test Entity Management & Components ---
        GD.Print("\n--- Testing Entity Management & Components ---");
        // Create the player entity (this will trigger PlayerStatSystem to initialize stats)
        EntityID playerEntity = Entities.CreateEntity(EntityType.Player, "player_default", new Vector2(200, 200), 0f);
        GD.Print($"Created Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        EntityID npcEntity = Entities.CreateEntity(EntityType.NPC, "goblin_grunt", new Vector2(100, 100), 90f);
        GD.Print($"Created NPC: {npcEntity}. IsValid: {Entities.IsValid(npcEntity)}");
        
        // Let's not destroy playerEntity immediately to test damage later
        // Entities.DestroyEntity(playerEntity);
        // GD.Print($"Destroyed Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        // if (!Transforms.TryGetTransform(playerEntity, out TransformComponent destroyedTransform))
        // {
        //     GD.Print($"Attempted to get transform for destroyed entity {playerEntity}, it correctly failed.");
        // }

        GD.Print("--- End Testing Entity Management & Components ---\n");

        // --- Test JobSystem ---
        // ... (existing JobSystem tests) ...
        // --- End Testing JobSystem ---
        
        // --- Test PlayerStatSystem (Damage) ---
        GD.Print("\n--- Testing PlayerStatSystem ---");
        PlayerStats.TakeDamage(10f); // Player takes damage
        PlayerStats.TakeDamage(20f);
        PlayerStats.TakeDamage(70f); // Should kill the player
        GD.Print("--- End Testing PlayerStatSystem ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Time.Tick(delta);
        Events.FlushCommands();
        World.Tick(delta);
        Transforms.Tick(delta);
        // PlayerStatSystem doesn't have a Tick method for now, its updates are event/command driven.
    }

    private void InitializeSystems()
    {
        Events = new EventBus();
        GD.Print("  - EventBus initialized.");

        Time = new TimeSystem();
        GD.Print("  - TimeSystem initialized.");

        Entities = new EntityManager(Events);
        GD.Print("  - EntityManager initialized.");

        Transforms = new TransformSystem(Entities, Events);
        GD.Print("  - TransformSystem initialized.");
        
        Jobs = new JobSystem(Events);
        GD.Print("  - JobSystem initialized.");

        PlayerStats = new PlayerStatSystem(Events, Entities); // Initialize PlayerStatSystem here
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `EventBus.cs` for PlayerStatSystem Events

Our `PlayerStatSystem` publishes `PlayerHealthChangedEvent` and `PlayerDiedEvent`. We need to define these `Action` delegates in `EventBus`.

Open `_Brain/Core/EventBus.cs`:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Player Stat Events (TDD 01.4)
        public event Action<EntityID, float, float> OnPlayerHealthChanged;
        public event Action<EntityID> OnPlayerDied;

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is PlayerStatSystem.PlayerHealthChangedEvent healthEvent) // New condition
            {
                OnPlayerHealthChanged?.Invoke(healthEvent.PlayerID, healthEvent.NewHealth, healthEvent.MaxHealth);
            }
            else if (eventData is PlayerStatSystem.PlayerDiedEvent diedEvent) // New condition
            {
                OnPlayerDied?.Invoke(diedEvent.PlayerID);
            }
            else
            {
                GD.PrintErr($"EventBus: Attempted to publish unknown event type: {typeof(TEvent).Name}. Ensure it's handled in Publish<TEvent>.");
            }
        }
        // ... (AddCommand and FlushCommands methods) ...
    }
}
```

### 5. Displaying Player Stats in the Body (GDScript UI)

Now, let's create a simple UI (Body) to display the player's health, reacting to `OnPlayerHealthChanged` events.

1.  Create a new folder `res://_Body/Scenes/UI/`.
2.  Create a new scene `res://_Body/Scenes/UI/HUD.tscn`:
    *   Root node: `CanvasLayer`. Rename it `HUD`.
    *   Child node: `HBoxContainer`. Rename it `HealthBarContainer`.
        *   In Inspector, set `Anchor Preset` to `Top Wide` (or adjust layout as desired).
        *   Set `Position` (e.g., `x=10, y=10`).
    *   Child of `HealthBarContainer`: `Label`. Rename it `HealthLabel`.
        *   Set `Text` to `Health: 100/100`.
3.  Attach a new GDScript to the `HUD` node: `res://_Body/Scripts/UI/HUDController.gd`.

```gdscript
# _Body/Scripts/UI/HUDController.gd
class_name HUDController extends CanvasLayer

@onready var health_label: Label = $HealthBarContainer/HealthLabel

var player_entity_id: int = -1 # Store the player's EntityID.Index

func _ready():
    GD.print("HUDController: Initialized. Connecting to C# PlayerStatSystem events.")
    # Connect to C# PlayerStatSystem events
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        GameManager.Instance.Events.OnPlayerHealthChanged.connect(Callable(self, "_on_player_health_changed"))
        GameManager.Instance.Events.OnPlayerDied.connect(Callable(self, "_on_player_died"))
        # We also need to know the player's EntityID when it spawns
        GameManager.Instance.Events.OnEntitySpawned.connect(Callable(self, "_on_entity_spawned"))
        GD.print("HUDController: Successfully connected to C# PlayerStatSystem events.")
    else:
        push_error("HUDController: GameManager or EventBus not ready! Cannot connect C# events.")

func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    # Only care about the Player entity here
    # Godot's C# binding marshals EntityType enum as an int, so we compare to its integer value.
    if type == 0: # EntityType.Player has value 0
        player_entity_id = id
        GD.print("HUDController: Detected player entity with ID: %s" % player_entity_id)

## Handler for C# PlayerHealthChangedEvent.
func _on_player_health_changed(id: int, new_health: float, max_health: float) -> void:
    # Only update if it's our player's health
    if id == player_entity_id:
        health_label.text = "Health: %s/%s" % [int(new_health), int(max_health)]
        GD.print("HUDController: Player %s Health Updated to %s/%s" % [id, int(new_health), int(max_health)])

## Handler for C# PlayerDiedEvent.
func _on_player_died(id: int) -> void:
    if id == player_entity_id:
        health_label.text = "Health: DEAD"
        GD.print("HUDController: Player %s has died visually!" % id)
```

Now, instantiate the `HUD.tscn` scene into `res://Gameplay.tscn`:

1.  Open `res://Gameplay.tscn`.
2.  Add a new instance of `res://_Body/Scenes/UI/HUD.tscn` as a child of the `Gameplay` root node.
3.  Save `Gameplay.tscn`.

### 6. Testing Global State Management

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.

**Expected Output:**

*   The game loads `Gameplay.tscn`.
*   The `HUD` will appear with "Health: 100/100".
*   In the Output console, you'll see the `PlayerStatSystem` messages about damage taken.
*   The `HUD`'s health label will update `Health: 90/100`, then `Health: 70/100`, and finally `Health: DEAD`.

This demonstrates:

*   The C# Brain (`PlayerStatSystem`) authoritatively owns and modifies player health.
*   The C# Brain publishes events when health changes.
*   The GDScript Body (`HUDController`) listens to these events and reactively updates its UI.
*   The Body does not directly modify player health; it only displays the state provided by the Brain.

This is a perfect example of global state management and data ownership in action.

### Summary

You have successfully implemented a core aspect of Global State Management, ensuring that the C# Brain authoritatively owns all game data, specifically demonstrated with a `PlayerStatSystem`. By designing `PlayerStats` and integrating the system with `GameManager` and `EventBus`, you've established a clear pipeline for modifying game state in C#. Furthermore, you've created a reactive GDScript `HUDController` that listens to C# events and displays this state without owning it, strictly adhering to TDD 01.4's principles of data ownership and separation of concerns.

### Next Steps

The next chapter will focus on **Debugging Tools**, implementing a console and state inspector to help visualize and manipulate our complex simulation data at runtime, which is essential for developing a large-scale, systemic game like Sigilborne.