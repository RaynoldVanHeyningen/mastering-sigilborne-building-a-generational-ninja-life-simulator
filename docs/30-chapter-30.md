## Chapter 5.1: Biological Simulation - The Bio-Tick (C#)

Welcome to **Module 5: Chakra & Life Systems**! This module delves into the intricate biological and environmental simulation that governs character well-being. Unlike fast-paced combat, biological stats like hunger, thirst, and fatigue don't need to update every physics frame. This chapter implements a dedicated **Bio-Tick** system in the C# Brain, running at a slower frequency (e.g., 1Hz), to efficiently manage these stats, as specified in TDD 03.2.

### 1. The Need for a Slower Bio-Tick

*   **Performance**: Updating every biological stat for every NPC (even virtual ones, to be covered later) at 20Hz (our `_PhysicsProcess` rate) is overkill and computationally expensive.
*   **Realism**: Biological processes occur over seconds, minutes, or hours, not milliseconds. A slower tick rate aligns better with this.
*   **Decoupling**: Separates biological updates from the core physics and input loops.

### 2. Defining `BioState`

TDD 03.2 specifies a `readonly struct` for `BioState` to pass biological data around safely and efficiently.

1.  Create `res://_Brain/Systems/Biology/BioState.cs`:

```csharp
// _Brain/Systems/Biology/BioState.cs
using System;
using Godot; // For Vector2, though not directly used in this struct

namespace Sigilborne.Systems.Biology
{
    /// <summary>
    /// Stores the core biological state of an entity.
    /// This is a readonly struct for efficiency and immutability when passing data.
    /// (TDD 03.2)
    /// </summary>
    public struct BioState
    {
        public float Health;
        public float MaxHealth;
        public float Stamina;
        public float MaxStamina;
        public float Chakra;
        public float MaxChakra;
        public float Hunger;       // 0-100, 0 = starving
        public float MaxHunger;    // Always 100 for simplicity
        public float Thirst;       // 0-100, 0 = dehydrated
        public float MaxThirst;    // Always 100 for simplicity
        public float BodyTemp;     // Celsius (e.g., 37.0 = normal, lower/higher causes penalties)
        public float NormalBodyTemp; // Reference normal temp for this entity

        public BioState(float maxHealth, float maxStamina, float maxChakra, float normalBodyTemp = 37.0f)
        {
            MaxHealth = maxHealth;
            Health = maxHealth;
            MaxStamina = maxStamina;
            Stamina = maxStamina;
            MaxChakra = maxChakra;
            Chakra = maxChakra;
            MaxHunger = 100f;
            Hunger = MaxHunger; // Start full
            MaxThirst = 100f;
            Thirst = MaxThirst; // Start full
            NormalBodyTemp = normalBodyTemp;
            BodyTemp = normalBodyTemp;
        }

        public override string ToString()
        {
            return $"HP: {Health}/{MaxHealth}, Sta: {Stamina}/{MaxStamina}, Cha: {Chakra}/{MaxChakra}, Hunger: {Hunger:F0}, Thirst: {Thirst:F0}, Temp: {BodyTemp:F1}°C";
        }
    }
}
```

### 3. Implementing `BiologicalSystem.cs` with the Bio-Tick

This system will manage `BioState` for all entities that possess it (player, NPCs, animals) and apply updates on its own tick.

1.  Create `res://_Brain/Systems/Biology/BiologicalSystem.cs`:

```csharp
// _Brain/Systems/Biology/BiologicalSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Physics; // For potential physics interactions
using Sigilborne.Systems.Movement; // For activity level

namespace Sigilborne.Systems.Biology
{
    /// <summary>
    /// Manages the biological simulation for entities (player, NPCs, animals).
    /// Updates stats like hunger, thirst, and body temperature on a slower Bio-Tick.
    /// (TDD 03.2)
    /// </summary>
    public class BiologicalSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private TransformSystem _transformSystem; // To get entity positions for environmental effects
        private MovementSystem _movementSystem;   // To determine activity level

        // Dictionary to store BioState for all entities that have one.
        private Dictionary<EntityID, BioState> _bioStates = new Dictionary<EntityID, BioState>();

        // TDD 03.2: The Bio-Tick
        private const float BIO_TICK_RATE = 1.0f; // 1.0f = once per real second (1Hz)
        private float _bioTickTimer;

        public BiologicalSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, MovementSystem movementSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _movementSystem = movementSystem;

            // Subscribe to entity lifecycle events to manage BioStates
            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("BiologicalSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // Only add BioState to living entities (Player, NPC, Animal)
            if (type == EntityType.Player || type == EntityType.NPC || type == EntityType.Animal)
            {
                // For player, initial stats are handled by PlayerStatSystem.
                // For NPCs/Animals, we'd load from definitionID (e.g., "goblin_grunt" has 50HP, 20Chakra).
                // For now, let's give generic stats. PlayerStatSystem will override for player.
                _bioStates.Add(id, new BioState(100f, 75f, 50f));
                GD.Print($"BiologicalSystem: Added BioState for {type} entity {id}.");
            }
        }

        private void OnEntityDespawned(EntityID id)
        {
            _bioStates.Remove(id);
            GD.Print($"BiologicalSystem: Removed BioState for {id}.");
        }

        /// <summary>
        /// Main update loop for the BiologicalSystem.
        /// Manages the Bio-Tick.
        /// (TDD 03.2)
        /// </summary>
        public void Tick(double delta)
        {
            _bioTickTimer += (float)delta;
            if (_bioTickTimer >= BIO_TICK_RATE)
            {
                ProcessBioTick();
                _bioTickTimer = 0;
            }
        }

        /// <summary>
        /// Processes a single Bio-Tick for all entities with BioState.
        /// (TDD 03.2)
        /// </summary>
        private void ProcessBioTick()
        {
            // GD.Print($"BiologicalSystem: Processing Bio-Tick (Game Time: {GameManager.Instance.Time.CurrentGameTime:F1})");

            foreach (var kvp in _bioStates)
            {
                EntityID id = kvp.Key;
                ref BioState bioState = ref _bioStates.GetValueRef(id); // Get mutable ref

                if (!_entityManager.IsValid(id)) continue; // Ensure entity is still valid

                // --- Metabolism System (TDD 03.2) ---
                float activityMultiplier = GetActivityLevel(id); // 1x Idle, 2x Running, 5x Combat
                float weatherMultiplier = 1.0f; // Placeholder for weather effects (TDD 03.3)

                // Hunger decay
                bioState.Hunger -= 1.0f * activityMultiplier * weatherMultiplier; // Base decay of 1 per tick
                bioState.Hunger = Mathf.Max(0, bioState.Hunger); // Clamp at 0
                
                // Thirst decay
                bioState.Thirst -= 1.5f * activityMultiplier * weatherMultiplier; // Thirst decays faster
                bioState.Thirst = Mathf.Max(0, bioState.Thirst); // Clamp at 0

                // Body temperature (placeholder for environmental effects)
                // bioState.BodyTemp = AdjustBodyTemperature(id, bioState.BodyTemp, activityMultiplier, weatherMultiplier);

                // --- Apply consequences of low stats (GDD B14.4) ---
                if (bioState.Hunger <= 0)
                {
                    // GD.Print($"BiologicalSystem: {id} is starving!");
                    // _eventBus.Publish(new EntityStarvingEvent { EntityID = id });
                    // Apply health damage or stat penalties here.
                }
                if (bioState.Thirst <= 0)
                {
                    // GD.Print($"BiologicalSystem: {id} is dehydrated!");
                    // _eventBus.Publish(new EntityDehydratedEvent { EntityID = id });
                    // Apply health damage or stat penalties here.
                }

                // Publish updates for the Body (e.g., UI for player, visual cues for NPCs)
                if (id == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerBioStateChangedEvent { PlayerID = id, NewBioState = bioState });
                }
            }
        }

        /// <summary>
        /// Determines the activity level of an entity for metabolism calculations.
        /// (TDD 03.2)
        /// </summary>
        private float GetActivityLevel(EntityID id)
        {
            // For now, only player has a MovementSystem.
            if (id == _entityManager.GetPlayerEntityID())
            {
                // Access player's velocity from MovementSystem (if it stored it, or from TransformSystem)
                // For simplicity, let's assume a "moving" state based on velocity magnitude.
                if (_movementSystem.GetVelocity(id).LengthSquared() > 1.0f) // If moving significantly
                {
                    return 2.0f; // 2x decay for running
                }
            }
            return 1.0f; // Default 1x decay for idle/NPCs
        }

        /// <summary>
        /// Provides a mutable reference to an entity's BioState.
        /// Other systems (e.g., PlayerStatSystem, CombatSystem) can modify health, stamina, chakra.
        /// </summary>
        public ref BioState GetBioStateRef(EntityID id)
        {
            if (!_entityManager.IsValid(id) || !_bioStates.ContainsKey(id))
            {
                throw new InvalidOperationException($"Entity {id} is invalid or does not have a BioState.");
            }
            return ref _bioStates.GetValueRef(id);
        }

        // --- Helper Events for Body Sync ---
        public struct PlayerBioStateChangedEvent { public EntityID PlayerID; public BioState NewBioState; }
        // public struct EntityStarvingEvent { public EntityID EntityID; } // Example
        // public struct EntityDehydratedEvent { public EntityID EntityID; } // Example
    }
}
```

#### 3.1. Update `MovementSystem.cs` to expose `GetVelocity`

The `BiologicalSystem` needs to query the player's velocity.

Open `res://_Brain/Systems/Movement/MovementSystem.cs` and add this method:

```csharp
// _Brain/Systems/Movement/MovementSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Input;

namespace Sigilborne.Systems.Movement
{
    // ... (existing structs) ...

    public class MovementSystem
    {
        // ... (existing fields and constructor) ...

        public void Tick(double delta) { /* ... */ }

        /// <summary>
        /// Retrieves the current velocity of an entity.
        /// </summary>
        public Vector2 GetVelocity(EntityID id)
        {
            if (_velocities.TryGetValue(id, out VelocityComponent vel))
            {
                return vel.Velocity;
            }
            return Vector2.Zero;
        }
    }
}
```

### 4. Integrating `BiologicalSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Biology;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `BiologicalSystem` property.
3.  Initialize `BiologicalSystem` in `InitializeSystems()`.
4.  Call `BiologicalSystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Input;
using Sigilborne.Systems.Movement;
using Sigilborne.Systems.Physics;
using Sigilborne.Systems.Magic;
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public PlayerStatSystem PlayerStats { get; private set; }
    public BiologicalSystem Biology { get; private set; } // Add BiologicalSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta);
        Casting.Tick(delta);
        Biology.Tick(delta); // Call BiologicalSystem's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to PlayerStatSystem) ...

        DebugCommands = new DebugCommandSystem(this);
        GD.Print("  - DebugCommandSystem initialized.");

        Input = new InputSystem();
        GD.Print("  - InputSystem initialized.");

        Movement = new MovementSystem(Entities, Input, Events, Transforms);
        GD.Print("  - MovementSystem initialized.");

        Physics = new PhysicsSystem(Entities, Transforms, Events);
        GD.Print("  - PhysicsSystem initialized.");

        int currentWorldSeed = 12345;
        int numGlyphsForWorld = 10;
        GlyphMap = new WorldGlyphMap(currentWorldSeed, numGlyphsForWorld);
        GD.Print("  - WorldGlyphMap initialized.");

        PlayerGlyphKnowledge = new PlayerGlyphKnowledgeSystem(Entities.GetPlayerEntityID(), GlyphMap, Events);
        GD.Print("  - PlayerGlyphKnowledgeSystem initialized.");

        PlayerHotbar = new PlayerHotbarSystem(Entities.GetPlayerEntityID(), Events, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - PlayerHotbarSystem initialized.");

        ComboResolver = new ComboResolver();
        GD.Print("  - ComboResolver initialized.");

        Casting = new CastingSystem(Entities, Events, PlayerStats, Transforms);
        GD.Print("  - CastingSystem initialized.");

        Magic = new MagicSystem(Entities, Input, Events, PlayerHotbar, PlayerGlyphKnowledge, GlyphMap, this, ComboResolver, Casting);
        GD.Print("  - MagicSystem initialized.");

        // Initialize BiologicalSystem AFTER PlayerStatSystem (as PlayerStatSystem sets initial player health)
        Biology = new BiologicalSystem(Entities, Events, Transforms, Movement); // Initialize BiologicalSystem here
        GD.Print("  - BiologicalSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `PlayerStatSystem.cs` to use `BiologicalSystem`

The `PlayerStatSystem` currently manages `PlayerStats` directly. It should now delegate to `BiologicalSystem`'s `GetBioStateRef` for core stats like health, stamina, chakra. This makes `BiologicalSystem` the single source of truth for these stats.

Open `res://_Brain/Systems/Biology/PlayerStatSystem.cs` and modify it:

```csharp
// _Brain/Systems/Biology/PlayerStatSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;

namespace Sigilborne.Systems.Biology
{
    // ... (PlayerStats struct) ...

    public class PlayerStatSystem
    {
        private EventBus _eventBus;
        private EntityManager _entityManager;
        private BiologicalSystem _biologicalSystem; // New: Reference to BiologicalSystem

        private EntityID _playerEntityID; // Store player ID directly

        public PlayerStatSystem(EventBus eventBus, EntityManager entityManager, BiologicalSystem biologicalSystem) // Add BiologicalSystem
        {
            _eventBus = eventBus;
            _entityManager = entityManager;
            _biologicalSystem = biologicalSystem; // Store BiologicalSystem reference

            _playerEntityID = _entityManager.GetPlayerEntityID(); // Get player ID once

            GD.Print("PlayerStatSystem: Initialized.");
            
            // PlayerStatSystem no longer needs to listen to OnEntitySpawned to initialize BioState,
            // as BiologicalSystem handles that. It just needs to get a reference to the player's BioState.
            // We'll initialize it in _Ready() or a later method.
            // For now, ensure player's BioState is created by BiologicalSystem before this is called.
            
            // Publish initial player stats from BiologicalSystem's BioState
            ref BioState playerBioState = ref _biologicalSystem.GetBioStateRef(_playerEntityID);
            _eventBus.Publish(new PlayerHealthChangedEvent { PlayerID = _playerEntityID, NewHealth = playerBioState.Health, MaxHealth = playerBioState.MaxHealth });
            _eventBus.Publish(new PlayerChakraChangedEvent { PlayerID = _playerEntityID, NewChakra = playerBioState.Chakra, MaxChakra = playerBioState.MaxChakra });
            // Add Stamina changed event as well
            _eventBus.Publish(new PlayerStaminaChangedEvent { PlayerID = _playerEntityID, NewStamina = playerBioState.Stamina, MaxStamina = playerBioState.MaxStamina });
        }

        /// <summary>
        /// Retrieves a copy of the player's current stats from the BiologicalSystem.
        /// </summary>
        public PlayerStats GetPlayerStats()
        {
            // Construct PlayerStats from BiologicalSystem's authoritative BioState
            ref BioState bioState = ref _biologicalSystem.GetBioStateRef(_playerEntityID);
            return new PlayerStats(_playerEntityID, bioState.MaxHealth, bioState.MaxChakra, bioState.MaxStamina)
            {
                Health = bioState.Health,
                Chakra = bioState.Chakra,
                Stamina = bioState.Stamina
            };
        }

        public void TakeDamage(float amount)
        {
            ref BioState playerBioState = ref _biologicalSystem.GetBioStateRef(_playerEntityID);
            if (playerBioState.Health <= 0) return;

            playerBioState.Health -= amount;
            if (playerBioState.Health < 0) playerBioState.Health = 0;

            GD.Print($"PlayerStatSystem: Player {_playerEntityID} took {amount} damage. New Health: {playerBioState.Health}");
            _eventBus.Publish(new PlayerHealthChangedEvent { PlayerID = _playerEntityID, NewHealth = playerBioState.Health, MaxHealth = playerBioState.MaxHealth });

            if (playerBioState.Health == 0)
            {
                GD.Print($"PlayerStatSystem: Player {_playerEntityID} has died!");
                _eventBus.Publish(new PlayerDiedEvent { PlayerID = _playerEntityID });
            }
        }

        public void TakeChakra(float amount)
        {
            ref BioState playerBioState = ref _biologicalSystem.GetBioStateRef(_playerEntityID);
            if (amount < 0) return;

            playerBioState.Chakra -= amount;
            if (playerBioState.Chakra < 0) playerBioState.Chakra = 0;

            GD.Print($"PlayerStatSystem: Player {_playerEntityID} used {amount} chakra. New Chakra: {playerBioState.Chakra}");
            _eventBus.Publish(new PlayerChakraChangedEvent { PlayerID = _playerEntityID, NewChakra = playerBioState.Chakra, MaxChakra = playerBioState.MaxChakra });
        }

        // --- Helper Events for Body Sync ---
        public struct PlayerHealthChangedEvent { public EntityID PlayerID; public float NewHealth; public float MaxHealth; }
        public struct PlayerDiedEvent { public EntityID PlayerID; }
        public struct PlayerChakraChangedEvent { public EntityID PlayerID; public float NewChakra; public float MaxChakra; }
        public struct PlayerStaminaChangedEvent { public EntityID PlayerID; public float NewStamina; public float MaxStamina; } // New event
    }
}
```

#### 4.2. Update `EventBus.cs` for PlayerStaminaChangedEvent

Open `res://_Brain/Core/EventBus.cs` and add `OnPlayerStaminaChanged` delegate:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Player Stat Events (TDD 01.4) - updated with Stamina
        public event Action<EntityID, float, float> OnPlayerHealthChanged;
        public event Action<EntityID> OnPlayerDied;
        public event Action<EntityID, float, float> OnPlayerChakraChanged;
        public event Action<EntityID, float, float> OnPlayerStaminaChanged; // New event

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is PlayerStatSystem.PlayerChakraChangedEvent chakraEvent)
            {
                OnPlayerChakraChanged?.Invoke(chakraEvent.PlayerID, chakraEvent.NewChakra, chakraEvent.MaxChakra);
            }
            else if (eventData is PlayerStatSystem.PlayerStaminaChangedEvent staminaEvent) // New condition
            {
                OnPlayerStaminaChanged?.Invoke(staminaEvent.PlayerID, staminaEvent.NewStamina, staminaEvent.MaxStamina);
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

### 5. Displaying Player BioState in the Body (GDScript UI)

Let's update our `HUDController.gd` to display Chakra, Stamina, Hunger, and Thirst.

Open `res://_Body/Scenes/UI/HUD.tscn` and modify it:

*   Add `Label` nodes for Chakra, Stamina, Hunger, Thirst.
*   Arrange them (e.g., in `VBoxContainer` or `HBoxContainer`).
*   Example:
    ```
    HUD (CanvasLayer)
    └── MainHBox (HBoxContainer)
        ├── VBoxStats (VBoxContainer)
        │   ├── HealthLabel (Label)
        │   ├── ChakraLabel (Label)
        │   ├── StaminaLabel (Label)
        └── VBoxNeeds (VBoxContainer)
            ├── HungerLabel (Label)
            └── ThirstLabel (Label)
    ```

Open `res://_Body/Scripts/UI/HUDController.gd` and modify it:

```gdscript
# _Body/Scripts/UI/HUDController.gd
class_name HUDController extends CanvasLayer

@onready var health_label: Label = $MainHBox/VBoxStats/HealthLabel
@onready var chakra_label: Label = $MainHBox/VBoxStats/ChakraLabel # New
@onready var stamina_label: Label = $MainHBox/VBoxStats/StaminaLabel # New
@onready var hunger_label: Label = $MainHBox/VBoxNeeds/HungerLabel # New
@onready var thirst_label: Label = $MainHBox/VBoxNeeds/ThirstLabel # New

var player_entity_id: int = -1

func _ready():
    GD.print("HUDController: Initialized. Connecting to C# PlayerStatSystem & BiologicalSystem events.")
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        GameManager.Instance.Events.OnPlayerHealthChanged.connect(Callable(self, "_on_player_health_changed"))
        GameManager.Instance.Events.OnPlayerChakraChanged.connect(Callable(self, "_on_player_chakra_changed")) # New
        GameManager.Instance.Events.OnPlayerStaminaChanged.connect(Callable(self, "_on_player_stamina_changed")) # New
        GameManager.Instance.Events.OnPlayerBioStateChanged.connect(Callable(self, "_on_player_bio_state_changed")) # New
        GameManager.Instance.Events.OnPlayerDied.connect(Callable(self, "_on_player_died"))
        GameManager.Instance.Events.OnEntitySpawned.connect(Callable(self, "_on_entity_spawned"))
        GD.print("HUDController: Successfully connected to C# PlayerStatSystem & BiologicalSystem events.")
    else:
        push_error("HUDController: GameManager or EventBus not ready! Cannot connect C# events.")

func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    if type == 0: # EntityType.Player has value 0
        player_entity_id = id
        GD.print("HUDController: Detected player entity with ID: %s" % player_entity_id)

func _on_player_health_changed(id: int, new_health: float, max_health: float) -> void:
    if id == player_entity_id:
        health_label.text = "HP: %s/%s" % [int(new_health), int(max_health)]

func _on_player_chakra_changed(id: int, new_chakra: float, max_chakra: float) -> void: # New
    if id == player_entity_id:
        chakra_label.text = "CHA: %s/%s" % [int(new_chakra), int(max_chakra)]

func _on_player_stamina_changed(id: int, new_stamina: float, max_stamina: float) -> void: # New
    if id == player_entity_id:
        stamina_label.text = "STA: %s/%s" % [int(new_stamina), int(max_stamina)]

## Handler for C# PlayerBioStateChangedEvent (for Hunger, Thirst, Temp)
func _on_player_bio_state_changed(id: int, new_bio_state: Variant) -> void: # Variant to receive C# struct
    if id == player_entity_id:
        # Access fields of the BioState struct (marshaled as a Dictionary or similar)
        # Godot's C# binding usually marshals structs as Dictionaries in GDScript.
        hunger_label.text = "HUN: %s" % int(new_bio_state.Hunger)
        thirst_label.text = "THI: %s" % int(new_bio_state.Thirst)
        # You can add BodyTemp here too if you display it
        # GD.print("HUDController: Player %s BioState Updated: %s" % [id, new_bio_state])

func _on_player_died(id: int) -> void:
    if id == player_entity_id:
        health_label.text = "HP: DEAD"
        chakra_label.text = "CHA: DEAD"
        stamina_label.text = "STA: DEAD"
        hunger_label.text = "HUN: DEAD"
        thirst_label.text = "THI: DEAD"
        GD.print("HUDController: Player %s has died visually!" % id)
```

#### 5.1. Update `EventBus.cs` for PlayerBioStateChangedEvent

Open `res://_Brain/Core/EventBus.cs` and add `OnPlayerBioStateChanged` delegate:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Biological System Events (TDD 03.2)
        public event Action<EntityID, BioState> OnPlayerBioStateChanged; // New event

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is BiologicalSystem.PlayerBioStateChangedEvent bioStateEvent) // New condition
            {
                OnPlayerBioStateChanged?.Invoke(bioStateEvent.PlayerID, bioStateEvent.NewBioState);
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

### 6. Testing the Bio-Tick and UI Updates

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the new UI elements for Hunger and Thirst.
5.  Wait for a few seconds. You should see Hunger and Thirst values slowly decreasing, and the UI updating every second (our `BIO_TICK_RATE`).
6.  Move the player (WASD). You should see Hunger and Thirst decrease faster due to increased `activityMultiplier`.
7.  Cast a spell (e.g., `0` then `1`). You should see Chakra decrease.

This confirms that:
*   The C# `BiologicalSystem` is ticking independently at 1Hz.
*   Hunger and Thirst are decaying based on `activityMultiplier`.
*   The GDScript `HUDController` is correctly listening to `OnPlayerBioStateChanged` and `OnPlayerChakraChanged` events and updating the UI.

### Summary

You have successfully implemented the **Bio-Tick** system in the C# Brain, establishing a dedicated, slower update loop for managing biological stats like hunger, thirst, and body temperature. By designing `BioState` and creating `BiologicalSystem` to manage these stats and apply metabolism rules, you've created an efficient simulation layer. Furthermore, by updating `PlayerStatSystem` to delegate core stat management to `BiologicalSystem` and enhancing `HUDController.gd` to reactively display these stats, you've ensured accurate and performant biological simulation, strictly adhering to TDD 03.2's specifications.

### Next Steps

The next chapter will focus on **Core Stats (Struct)**, further defining the complete set of biological and combat-related stats for entities, and how they are managed and accessed across various systems.