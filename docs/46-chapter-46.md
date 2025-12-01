## Chapter 7.3: Stealth Mechanics - Visibility & Detection Meter (C#)

With our `PerceptionSystem` allowing entities to see and hear, it's time to introduce **Stealth Mechanics**. This chapter focuses on implementing `Visibility` as a float value for entities, influenced by factors like light level, movement speed, and (conceptually) camouflage. We'll also design a `Detection Meter` that, when filled, transitions NPCs into an "Alert" state, creating a tactical interplay between stealth and perception, as specified in TDD 05.2.3 and GDD B10.

### 1. The Dynamic Nature of Stealth

The GDD (B10.2) states: "Stealth is weak early, viable mid-game, and powerful at high mastery, but always counterable." This implies:

*   **Non-Binary Stealth**: Not just "hidden" or "detected," but a continuous `Visibility` score.
*   **Environmental Influence**: Light, shadows, and terrain affect visibility.
*   **Player Skill**: Movement control and glyphs (Veil, Echo) can reduce visibility.
*   **NPC Detection**: NPCs accumulate "detection" over time, leading to alert states.

### 2. Defining `StealthComponent` and `DetectionComponent`

We need components to store an entity's stealth properties (what makes it hidden) and its detection properties (how much it's currently detected).

1.  Create `res://_Brain/Systems/AI/StealthComponent.cs`:

```csharp
// _Brain/Systems/AI/StealthComponent.cs
using System;
using Godot; // For Vector2

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Stores the parameters defining an entity's stealth capabilities and current visibility.
    /// (TDD 05.2.3)
    /// </summary>
    public struct StealthComponent
    {
        public float BaseVisibility;    // Base visibility (1.0 = fully visible, 0.0 = invisible)
        public float MovementNoise;     // How much noise this entity generates when moving (0-1)
        public float CamouflageBonus;   // Environmental camouflage bonus (e.g., -0.2 for 20% less visible)
        public float ChakraSignatureMod; // How much this entity's chakra signature is dampened (0-1)

        public StealthComponent(float baseVisibility = 1.0f, float movementNoise = 0.5f, float camouflageBonus = 0f, float chakraSignatureMod = 0f)
        {
            BaseVisibility = baseVisibility;
            MovementNoise = movementNoise;
            CamouflageBonus = camouflageBonus;
            ChakraSignatureMod = chakraSignatureMod;
        }

        public override string ToString()
        {
            return $"Vis: {BaseVisibility:F2}, Noise: {MovementNoise:F2}, Camo: {CamouflageBonus:F2}";
        }
    }
}
```

2.  Create `res://_Brain/Systems/AI/DetectionComponent.cs`:

```csharp
// _Brain/Systems/AI/DetectionComponent.cs
using System;
using Godot; // For Vector2

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Stores an entity's current detection state by a specific perceiver.
    /// (TDD 05.2.3: Detection Meter)
    /// </summary>
    public struct DetectionComponent
    {
        public EntityID PerceiverID;       // The entity doing the detecting (e.g., an NPC looking at the player)
        public float DetectionMeter;        // Accumulates detection (0-100). If > Threshold, target is Alerted.
        public float MaxDetectionMeter;     // Max value of the meter (e.g., 100)
        public float DecayRate;             // How fast the meter decays when not actively detected
        public bool IsAlerted;             // True if the target is fully detected and the perceiver is alerted.

        public DetectionComponent(EntityID perceiverID, float maxDetectionMeter = 100f, float decayRate = 5f)
        {
            PerceiverID = perceiverID;
            DetectionMeter = 0;
            MaxDetectionMeter = maxDetectionMeter;
            DecayRate = decayRate;
            IsAlerted = false;
        }

        public override string ToString()
        {
            return $"Det: {DetectionMeter:F1}/{MaxDetectionMeter:F0} (Alerted: {IsAlerted})";
        }
    }
}
```

### 3. Implementing `StealthSystem.cs`

This system will:
*   Manage `StealthComponent`s.
*   Calculate an entity's `CurrentVisibility` and `CurrentNoise` based on its state.
*   Manage `DetectionComponent`s, updating meters and triggering `Alerted` states.

1.  Create `res://_Brain/Systems/AI/StealthSystem.cs`:

```csharp
// _Brain/Systems/AI/StealthSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components; // For TransformComponent
using Sigilborne.Systems.Biology; // For CoreStats
using Sigilborne.Systems.Movement; // For movement speed
using Sigilborne.Systems.Weather; // For light level / environmental effects
using System.Linq;

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Manages stealth mechanics, including calculating entity visibility/noise and
    /// handling detection meters for perceivers.
    /// (TDD 05.2.3, GDD B10)
    /// </summary>
    public class StealthSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private TransformSystem _transformSystem;
        private BiologicalSystem _biologicalSystem; // For CoreStats (movement speed)
        private MovementSystem _movementSystem;     // For actual velocity
        private WeatherSystem _weatherSystem;       // For light level / environmental effects
        private PerceptionSystem _perceptionSystem; // To query perceivers

        // Dictionary to store StealthComponent for entities that can be stealthy.
        private Dictionary<EntityID, StealthComponent> _stealthComponents = new Dictionary<EntityID, StealthComponent>();
        
        // Dictionary to store DetectionComponents (perceiver -> target -> detection state)
        // For simplicity, we'll store per-target detection in a list on the target for now,
        // rather than per-perceiver. This means a target knows *how much* it's detected.
        private Dictionary<EntityID, List<DetectionComponent>> _targetDetectionStates = new Dictionary<EntityID, List<DetectionComponent>>();


        public StealthSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, BiologicalSystem biologicalSystem, MovementSystem movementSystem, WeatherSystem weatherSystem, PerceptionSystem perceptionSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _biologicalSystem = biologicalSystem;
            _movementSystem = movementSystem;
            _weatherSystem = weatherSystem;
            _perceptionSystem = perceptionSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("StealthSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // Only add StealthComponent to entities that can be stealthy (Player, NPC, some Animals)
            if (type == EntityType.Player || type == EntityType.NPC || type == EntityType.Animal)
            {
                // Player has better base stealth
                if (type == EntityType.Player)
                {
                    _stealthComponents.Add(id, new StealthComponent(baseVisibility: 0.8f, movementNoise: 0.3f));
                }
                else // NPC / Animal
                {
                    _stealthComponents.Add(id, new StealthComponent(baseVisibility: 1.0f, movementNoise: 0.5f));
                }
                _targetDetectionStates.Add(id, new List<DetectionComponent>()); // Add detection states for this target
                GD.Print($"StealthSystem: Added StealthComponent for {type} entity {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _stealthComponents.Remove(e.ID);
            _targetDetectionStates.Remove(e.ID);
            GD.Print($"StealthSystem: Removed StealthComponent for {e.ID}.");
        }

        /// <summary>
        /// Main update loop for the StealthSystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            foreach (var kvp in _stealthComponents)
            {
                EntityID targetID = kvp.Key;
                ref StealthComponent stealth = ref _stealthComponents.GetValueRef(targetID);

                if (!_entityManager.IsValid(targetID)) continue;

                // --- Calculate Current Visibility (TDD 05.2.3) ---
                float currentVisibility = CalculateCurrentVisibility(targetID, stealth);
                // GD.Print($"StealthSystem: {targetID} Current Visibility: {currentVisibility:P1}");

                // --- Update Detection Meters ---
                // For each target, iterate through all potential perceivers.
                // This is a simplified O(N*M) loop. In a real game, only relevant perceivers would update.
                // For now, let's just make the player the target and update its detection by an "abstract" NPC.
                if (targetID == _entityManager.GetPlayerEntityID())
                {
                    UpdatePlayerDetection(targetID, currentVisibility, delta);
                }
            }
        }

        /// <summary>
        /// Calculates the current effective visibility of an entity.
        /// (TDD 05.2.3: Factors: Light Level (from TileMap), Movement Speed, Camouflage)
        /// </summary>
        /// <param name="entityID">The entity whose visibility is being calculated.</param>
        /// <param name="stealthComponent">The entity's StealthComponent.</param>
        /// <returns>A float from 0.0 (invisible) to 1.0 (fully visible).</returns>
        private float CalculateCurrentVisibility(EntityID entityID, StealthComponent stealthComponent)
        {
            float visibility = stealthComponent.BaseVisibility;

            // 1. Movement Speed (TDD 05.2.3)
            if (_movementSystem.TryGetCoreStats(entityID, out CoreStats coreStats))
            {
                Vector2 velocity = _movementSystem.GetVelocity(entityID);
                float speedRatio = velocity.Length() / coreStats.MoveSpeed; // 0 for idle, 1 for max speed
                visibility += speedRatio * stealthComponent.MovementNoise; // Moving faster increases visibility
            }

            // 2. Light Level (from WeatherSystem, conceptual local light) (TDD 05.2.3)
            // if (_transformSystem.TryGetTransform(entityID, out TransformComponent transform))
            // {
            //     float lightLevel = _weatherSystem.GetLightLevelAt(transform.Position); // Conceptual
            //     visibility *= lightLevel; // Darker areas reduce visibility
            // }

            // 3. Camouflage Bonus (TDD 05.2.3)
            visibility += stealthComponent.CamouflageBonus; // Negative bonus reduces visibility

            // Later: Chakra signature (GDD B10.3.B)
            // visibility += stealthComponent.ChakraSignatureMod * _biologicalSystem.GetChakraSignatureStrength(entityID);

            return Mathf.Clamp(visibility, 0f, 1f);
        }

        /// <summary>
        /// Calculates the current effective noise generated by an entity.
        /// </summary>
        public float CalculateCurrentNoise(EntityID entityID, StealthComponent stealthComponent)
        {
            if (!_biologicalSystem.TryGetCoreStats(entityID, out CoreStats coreStats)) return 0;
            Vector2 velocity = _movementSystem.GetVelocity(entityID);
            float speedRatio = velocity.Length() / coreStats.MoveSpeed;
            return stealthComponent.MovementNoise * speedRatio;
        }

        /// <summary>
        /// Updates the player's detection meter by an abstract NPC perceiver.
        /// (TDD 05.2.3: Detection Meter)
        /// </summary>
        private void UpdatePlayerDetection(EntityID playerID, float playerVisibility, double delta)
        {
            // For testing, let's use a dummy NPC perceiver.
            // In a real game, this would loop through actual NPCs perceiving the player.
            EntityID dummyNpcID = GameManager.Instance.Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid;
            if (!dummyNpcID.IsValid()) return;

            // Find or create the detection component for this perceiver-target pair
            List<DetectionComponent> detections = _targetDetectionStates[playerID];
            int detectionIndex = detections.FindIndex(d => d.PerceiverID == dummyNpcID);

            if (detectionIndex == -1)
            {
                // Create a new detection entry if none exists for this perceiver
                detections.Add(new DetectionComponent(dummyNpcID));
                detectionIndex = detections.Count - 1;
            }

            ref DetectionComponent playerDetection = ref detections.GetValueRef(detectionIndex);

            // --- Detection Meter Logic ---
            // If the player is 'seen' by the dummy NPC (conceptual)
            bool isPlayerCurrentlySeen = _perceptionSystem.CanSee(dummyNpcID, playerID); // Use NPC's vision to see player

            if (isPlayerCurrentlySeen)
            {
                // Increase detection meter based on player's visibility and perceiver's sensitivity
                float detectionGain = playerVisibility * _perceptionSystem.GetPerceptionComponentRef(dummyNpcID).VisionRange * 0.05f * (float)delta; // Gain faster if more visible
                playerDetection.DetectionMeter += detectionGain;
                // GD.Print($"StealthSystem: {dummyNpcID} gains detection on {playerID}. Gain: {detectionGain:F2}. Current: {playerDetection.DetectionMeter:F1}");
            }
            else
            {
                // Decay detection meter if not currently seen
                playerDetection.DetectionMeter -= playerDetection.DecayRate * (float)delta;
            }

            playerDetection.DetectionMeter = Mathf.Clamp(playerDetection.DetectionMeter, 0f, playerDetection.MaxDetectionMeter);

            // Check for alert state transition (TDD 05.2.3)
            if (playerDetection.DetectionMeter >= playerDetection.MaxDetectionMeter * 0.8f && !playerDetection.IsAlerted) // 80% to alert
            {
                playerDetection.IsAlerted = true;
                GD.Print($"StealthSystem: Player {playerID} is now ALERTED by {dummyNpcID}!");
                _eventBus.Publish(new PlayerAlertedEvent { PlayerID = playerID, PerceiverID = dummyNpcID, IsAlerted = true });
            }
            else if (playerDetection.DetectionMeter < playerDetection.MaxDetectionMeter * 0.5f && playerDetection.IsAlerted) // Drop below 50% to de-alert
            {
                playerDetection.IsAlerted = false;
                GD.Print($"StealthSystem: Player {playerID} is no longer ALERTED by {dummyNpcID}.");
                _eventBus.Publish(new PlayerAlertedEvent { PlayerID = playerID, PerceiverID = dummyNpcID, IsAlerted = false });
            }

            // Publish event for UI (e.g., stealth meter, alert indicator)
            _eventBus.Publish(new PlayerDetectionUpdatedEvent { PlayerID = playerID, DetectionMeter = playerDetection.DetectionMeter, MaxDetectionMeter = playerDetection.MaxDetectionMeter, IsAlerted = playerDetection.IsAlerted });
        }


        // --- Helper Events for Body Sync ---
        public struct PlayerDetectionUpdatedEvent { public EntityID PlayerID; public float DetectionMeter; public float MaxDetectionMeter; public bool IsAlerted; }
        public struct PlayerAlertedEvent { public EntityID PlayerID; public EntityID PerceiverID; public bool IsAlerted; }
    }
}
```

### 4. Integrating `StealthSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.AI;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `StealthSystem` property.
3.  Initialize `StealthSystem` in `InitializeSystems()` **after** `PerceptionSystem`.
4.  Call `StealthSystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).

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
using Sigilborne.Systems.Weather;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Combat;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.AI;
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public PerceptionSystem Perception { get; private set; }
    public StealthSystem Stealth { get; private set; } // Add StealthSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        // ... (existing inventory/equipment tests) ...
        // ... (existing spatial grid tests) ...
        // ... (existing perception system tests) ...

        // --- Test Stealth System ---
        GD.Print("\n--- Testing Stealth System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID testNpcID = Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid; // Assuming NPC is ID 1, Gen 1
        
        if (playerID.IsValid() && testNpcID.IsValid())
        {
            // Place player and NPC close for detection. Player is facing NPC.
            Transforms.GetTransformRef(playerID).Position = new Vector2(200, 200);
            Transforms.GetTransformRef(testNpcID).Position = new Vector2(250, 200);
            Transforms.GetTransformRef(playerID).RotationDegrees = 0; // Face right

            GD.Print($"Player {playerID} initial stealth: {Stealth.GetStealthComponentRef(playerID)}");
            GD.Print($"NPC {testNpcID} initial perception: {Perception.GetPerceptionComponentRef(testNpcID)}");
        }
        else
        {
            GD.PrintErr("Player or Test NPC not valid for stealth tests.");
        }

        GD.Print("--- End Testing Stealth System ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta);
        Casting.Tick(delta);
        Weather.Tick(delta);
        Biology.Tick(delta);
        TitanAI.Tick(delta);
        StatusEffects.Tick(delta);
        Perception.Tick(delta); // PerceptionSystem needs to tick if it's continuously querying, or query here.
        Stealth.Tick(delta); // Call StealthSystem's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to PerceptionSystem) ...
        
        // Initialize StealthSystem AFTER PerceptionSystem and other dependencies
        Stealth = new StealthSystem(Entities, Events, Transforms, BiologicalSystem, Movement, Weather, Perception); // Pass dependencies
        GD.Print("  - StealthSystem initialized.");

        // ... (existing system initializations after StealthSystem) ...
    }
}
```

#### 4.1. Update `EventBus.cs` for Stealth Events

Open `res://_Brain/Core/EventBus.cs` and add `OnPlayerDetectionUpdated` and `OnPlayerAlerted` delegates.

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Combat;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.AI; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Stealth System Events (TDD 05.2.3)
        public event Action<EntityID, float, float, bool> OnPlayerDetectionUpdated; // PlayerID, DetectionMeter, MaxDetectionMeter, IsAlerted
        public event Action<EntityID, EntityID, bool> OnPlayerAlerted; // PlayerID, PerceiverID, IsAlerted

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is StealthSystem.PlayerDetectionUpdatedEvent detectionEvent) // New condition
            {
                OnPlayerDetectionUpdated?.Invoke(detectionEvent.PlayerID, detectionEvent.DetectionMeter, detectionEvent.MaxDetectionMeter, detectionEvent.IsAlerted);
            }
            else if (eventData is StealthSystem.PlayerAlertedEvent alertedEvent) // New condition
            {
                OnPlayerAlerted?.Invoke(alertedEvent.PlayerID, alertedEvent.PerceiverID, alertedEvent.IsAlerted);
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

### 5. Displaying Detection Meter in the Body (GDScript UI)

Let's add a simple UI to display the player's detection meter.

1.  Open `res://_Body/Scenes/UI/HUD.tscn` and modify it:
    *   Add a `Label` node for `DetectionLabel`.
    *   Arrange it (e.g., in `VBoxNeeds`).

2.  Open `res://_Body/Scripts/UI/HUDController.gd` and modify it:

```gdscript
# _Body/Scripts/UI/HUDController.gd
class_name HUDController extends CanvasLayer

@onready var health_label: Label = $MainHBox/VBoxStats/HealthLabel
@onready var chakra_label: Label = $MainHBox/VBoxStats/ChakraLabel
@onready var stamina_label: Label = $MainHBox/VBoxStats/StaminaLabel
@onready var stability_label: Label = $MainHBox/VBoxStats/StabilityLabel
@onready var hunger_label: Label = $MainHBox/VBoxNeeds/HungerLabel
@onready var thirst_label: Label = $MainHBox/VBoxNeeds/ThirstLabel
@onready var weather_label: Label = $MainHBox/VBoxNeeds/WeatherLabel
@onready var temperature_label: Label = $MainHBox/VBoxNeeds/TemperatureLabel
@onready var detection_label: Label = $MainHBox/VBoxNeeds/DetectionLabel # New

var player_entity_id: int = -1

func _ready():
    GD.print("HUDController: Initialized. Connecting to C# PlayerStatSystem & BiologicalSystem events.")
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        # ... (existing event connections) ...
        GameManager.Instance.Events.OnPlayerDetectionUpdated.connect(Callable(self, "_on_player_detection_updated")) # New
        GameManager.Instance.Events.OnPlayerAlerted.connect(Callable(self, "_on_player_alerted")) # New
        GD.print("HUDController: Successfully connected to C# PlayerStatSystem & BiologicalSystem events.")
    else:
        push_error("HUDController: GameManager or EventBus not ready! Cannot connect C# events.")

func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    if type == 0:
        player_entity_id = id
        GD.print("HUDController: Detected player entity with ID: %s" % player_entity_id)

# ... (existing stat update handlers) ...

## Handler for C# OnPlayerDetectionUpdated event.
func _on_player_detection_updated(id: int, detection_meter: float, max_detection_meter: float, is_alerted: bool) -> void:
    if id == player_entity_id:
        var alert_status = "Hidden"
        if is_alerted:
            alert_status = "ALERTED!"
        elif detection_meter > max_detection_meter * 0.5:
            alert_status = "Suspicious"
        elif detection_meter > 0:
            alert_status = "Detected"
        
        detection_label.text = "DET: %s/%s (%s)" % [int(detection_meter), int(max_detection_meter), alert_status]

## Handler for C# OnPlayerAlerted event.
func _on_player_alerted(id: int, perceiver_id: int, is_alerted: bool) -> void:
    if id == player_entity_id:
        GD.print("HUDController: Player %s is now %s by Perceiver %s!" % [id, "ALERTED" if is_alerted else "NOT ALERTED", perceiver_id])
        # Play a sound, flash screen, etc.
```

### 6. Testing Stealth Mechanics

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the new Detection label on the HUD.
    *   **Initial State**: The player starts near an NPC. The detection meter should slowly increase as the NPC "sees" the player.
    *   **Alerted**: Once the meter reaches 80, it should switch to "ALERTED!".
    *   **Movement**: Move the player. The detection gain should be higher because `playerVisibility` increases with movement speed.
    *   **Decay**: Stand still. The detection meter should slowly decay. If it drops below 50, the "ALERTED!" status should clear.
    *   **Hide**: Move the player far away from the NPC (e.g., to 1000,1000 using WASD). The detection meter should quickly decay to 0.

This confirms that the `StealthSystem` is correctly calculating `Visibility` based on movement and managing the `Detection Meter`, providing dynamic feedback on the player's stealth status.

### Summary

You have successfully implemented **Stealth Mechanics** in the C# Brain, designing `StealthComponent` and `DetectionComponent` to manage entity visibility and detection states. By creating `StealthSystem` to calculate `CurrentVisibility` based on movement and managing `DetectionMeter` accumulation and decay, you've established a dynamic and tactical interplay between stealth and perception. This crucial system strictly adheres to TDD 05.2.3 and GDD B10's specifications, providing continuous feedback on the player's stealth status and driving NPC alert states.

### Next Steps

The next chapter will focus on **Decision Making: Utility AI (C#)**, where we will design a flexible Utility AI system for NPCs, allowing them to select actions (e.g., attack, flee, eat, sleep) based on a scoring mechanism that evaluates their needs and environmental context.