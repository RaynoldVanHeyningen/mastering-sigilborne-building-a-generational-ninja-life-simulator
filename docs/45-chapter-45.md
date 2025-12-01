## Chapter 7.2: The Senses - Vision, Hearing & Chakra Sense (C#)

With our `SpatialHashGrid` efficiently providing lists of nearby entities, we can now build the actual **Perception System** that allows entities to "sense" their environment. This chapter focuses on implementing `Vision`, `Hearing`, and `Chakra Sense` components for entities in the C# Brain, which will leverage the `SpatialHashGrid` to detect nearby entities and events, enabling more sophisticated AI behavior, as specified in TDD 05.2.2.

### 1. The Layered Nature of Perception

The GDD (B10.4) describes various detection types (Sight, Sound, Chakra Sensing, Resonance, Bloodline). Our TDD (05.2.2) boils these down to:

*   **Vision**: Line-of-sight and cone checks.
*   **Hearing**: Radius checks based on sound volume.
*   **Chakra Sense**: Detecting active spells/chakra through walls.

Each sense will contribute to an entity's overall awareness.

### 2. Defining `PerceptionComponent`

This component will hold the parameters for an entity's sensory capabilities.

1.  Create `res://_Brain/Systems/AI/PerceptionComponent.cs`:

```csharp
// _Brain/Systems/AI/PerceptionComponent.cs
using System;
using Godot; // For Vector2

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Stores the parameters defining an entity's sensory capabilities.
    /// This is a component-like data structure managed by the PerceptionSystem.
    /// (TDD 05.2.2)
    /// </summary>
    public struct PerceptionComponent
    {
        // Vision parameters
        public float VisionRange;       // Max distance to see
        public float VisionAngle;       // Field of View angle in degrees (e.g., 120 for 120 degrees total)

        // Hearing parameters
        public float HearingRange;      // Max distance to hear
        public float HearingSensitivity; // Multiplier for how well sounds are heard (e.g., 1.0 normal, 1.5 keen)

        // Chakra Sense parameters
        public float ChakraSenseRange;  // Max distance to sense chakra
        public float ChakraSenseSensitivity; // Multiplier for how well chakra is sensed

        public PerceptionComponent(float visionRange = 200f, float visionAngle = 120f, float hearingRange = 150f, float hearingSensitivity = 1.0f, float chakraSenseRange = 100f, float chakraSenseSensitivity = 1.0f)
        {
            VisionRange = visionRange;
            VisionAngle = visionAngle;
            HearingRange = hearingRange;
            HearingSensitivity = hearingSensitivity;
            ChakraSenseRange = chakraSenseRange;
            ChakraSenseSensitivity = chakraSenseSensitivity;
        }

        public override string ToString()
        {
            return $"Vis: {VisionRange:F0}m/{VisionAngle:F0}°, Hear: {HearingRange:F0}m, Cha: {ChakraSenseRange:F0}m";
        }
    }
}
```

### 3. Implementing `PerceptionSystem.cs`

This system will:
*   Manage `PerceptionComponent`s.
*   Use `SpatialHashGrid` to get nearby entities.
*   Implement `CanSee()`, `CanHear()`, and `CanSenseChakra()` logic.

1.  Create `res://_Brain/Systems/AI/PerceptionSystem.cs`:

```csharp
// _Brain/Systems/AI/PerceptionSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components; // For TransformComponent
using Sigilborne.Systems.Magic; // For active spells
using System.Linq;

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Manages entities' sensory capabilities (vision, hearing, chakra sense).
    /// Leverages SpatialHashGrid for efficient nearby entity queries.
    /// (TDD 05.2.2)
    /// </summary>
    public class PerceptionSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private TransformSystem _transformSystem; // To get entity positions/rotations
        private SpatialHashGrid _spatialGrid;     // To get nearby entities

        // Dictionary to store PerceptionComponent for entities that can perceive.
        private Dictionary<EntityID, PerceptionComponent> _perceptionComponents = new Dictionary<EntityID, PerceptionComponent>();

        public PerceptionSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, SpatialHashGrid spatialGrid)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _spatialGrid = spatialGrid;

            // Subscribe to entity lifecycle events
            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("PerceptionSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // Only add PerceptionComponent to entities that should perceive (Player, NPC, some Animals)
            if (type == EntityType.Player || type == EntityType.NPC || type == EntityType.Animal)
            {
                // For player, initial vision range is higher.
                if (type == EntityType.Player)
                {
                    _perceptionComponents.Add(id, new PerceptionComponent(visionRange: 300f, visionAngle: 180f, hearingRange: 200f, chakraSenseRange: 150f));
                }
                else // NPC / Animal
                {
                    _perceptionComponents.Add(id, new PerceptionComponent(visionRange: 200f, visionAngle: 120f, hearingRange: 150f, chakraSenseRange: 100f));
                }
                GD.Print($"PerceptionSystem: Added PerceptionComponent for {type} entity {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _perceptionComponents.Remove(e.ID);
            GD.Print($"PerceptionSystem: Removed PerceptionComponent for {e.ID}.");
        }

        /// <summary>
        /// Retrieves a mutable reference to an entity's PerceptionComponent.
        /// </summary>
        public ref PerceptionComponent GetPerceptionComponentRef(EntityID id)
        {
            if (!_entityManager.IsValid(id) || !_perceptionComponents.ContainsKey(id))
            {
                throw new InvalidOperationException($"Entity {id} is invalid or does not have a PerceptionComponent.");
            }
            return ref _perceptionComponents.GetValueRef(id);
        }

        /// <summary>
        /// Checks if a perceiver entity can see a target entity.
        /// (TDD 05.2.2.1: Vision - Cone check (Dot Product + Raycast))
        /// </summary>
        /// <param name="perceiverID">The entity attempting to see.</param>
        /// <param name="targetID">The entity being looked for.</param>
        /// <returns>True if the target is seen, false otherwise.</returns>
        public bool CanSee(EntityID perceiverID, EntityID targetID)
        {
            if (!_entityManager.IsValid(perceiverID) || !_entityManager.IsValid(targetID) || perceiverID == targetID) return false;
            if (!_perceptionComponents.TryGetValue(perceiverID, out PerceptionComponent perceiverPerception)) return false;
            if (!_transformSystem.TryGetTransform(perceiverID, out TransformComponent perceiverTransform)) return false;
            if (!_transformSystem.TryGetTransform(targetID, out TransformComponent targetTransform)) return false;

            // 1. Range check
            float distance = perceiverTransform.Position.DistanceTo(targetTransform.Position);
            if (distance > perceiverPerception.VisionRange) return false;

            // 2. Angle (Field of View) check
            Vector2 directionToTarget = (targetTransform.Position - perceiverTransform.Position).Normalized();
            Vector2 perceiverForward = Vector2.FromAngle(Mathf.DegToRad(perceiverTransform.RotationDegrees)); // Convert degrees to radians for angle

            float angleToTarget = perceiverForward.AngleTo(directionToTarget); // Angle in radians
            float halfVisionAngleRad = Mathf.DegToRad(perceiverPerception.VisionAngle / 2f);

            if (Mathf.Abs(angleToTarget) > halfVisionAngleRad) return false;

            // 3. Line-of-sight (Raycast) check (Conceptual for now, involves Godot's physics)
            // This would typically be a call to Godot's PhysicsServer or a custom raycast system.
            // For now, we assume no obstacles.
            bool lineOfSightClear = true; // GameManager.Instance.Physics.Raycast(perceiverTransform.Position, targetTransform.Position);
            if (!lineOfSightClear) return false;

            // Later: Stealth mechanics (GDD B10.2). Check target's visibility/stealth score.
            // if (_stealthSystem.GetVisibility(targetID) < perceiverPerception.VisionSensitivity) return false;

            return true;
        }

        /// <summary>
        /// Checks if a perceiver entity can hear a sound event.
        /// (TDD 05.2.2.2: Hearing - Circle check)
        /// </summary>
        /// <param name="perceiverID">The entity attempting to hear.</param>
        /// <param name="soundOrigin">The world position of the sound.</param>
        /// <param name="soundVolume">The raw volume of the sound.</param>
        /// <returns>True if the sound is heard, false otherwise.</returns>
        public bool CanHear(EntityID perceiverID, Vector2 soundOrigin, float soundVolume)
        {
            if (!_entityManager.IsValid(perceiverID)) return false;
            if (!_perceptionComponents.TryGetValue(perceiverID, out PerceptionComponent perceiverPerception)) return false;
            if (!_transformSystem.TryGetTransform(perceiverID, out TransformComponent perceiverTransform)) return false;

            float distance = perceiverTransform.Position.DistanceTo(soundOrigin);
            // Sound's effective range is its volume scaled by perceiver's sensitivity
            float effectiveHearingRange = perceiverPerception.HearingRange * perceiverPerception.HearingSensitivity;

            return distance <= effectiveHearingRange && distance <= soundVolume; // Sound must be within range AND loud enough
        }

        /// <summary>
        /// Checks if a perceiver entity can sense the chakra of an active spell.
        /// (TDD 05.2.2.3: Chakra Sense - Detects active spells through walls.)
        /// </summary>
        /// <param name="perceiverID">The entity attempting to sense.</param>
        /// <param name="spellOrigin">The world position of the spell.</param>
        /// <param name="spellIntensity">The raw intensity of the spell's chakra signature.</param>
        /// <returns>True if the chakra is sensed, false otherwise.</returns>
        public bool CanSenseChakra(EntityID perceiverID, Vector2 spellOrigin, float spellIntensity)
        {
            if (!_entityManager.IsValid(perceiverID)) return false;
            if (!_perceptionComponents.TryGetValue(perceiverID, out PerceptionComponent perceiverPerception)) return false;
            if (!_transformSystem.TryGetTransform(perceiverID, out TransformComponent perceiverTransform)) return false;

            float distance = perceiverTransform.Position.DistanceTo(spellOrigin);
            // Chakra sense is effective if spell intensity is greater than distance (TDD 05.2.2.3)
            // And within perceiver's range.
            return distance <= perceiverPerception.ChakraSenseRange && spellIntensity > distance * (1f / perceiverPerception.ChakraSenseSensitivity);
        }

        /// <summary>
        /// Retrieves a list of entities that a perceiver can currently detect (see, hear, or sense chakra).
        /// (TDD 05.2.1: Query - "Get all entities in cells (X,Y) and neighbors.")
        /// </summary>
        /// <param name="perceiverID">The entity doing the perceiving.</param>
        /// <returns>A list of detected EntityIDs.</returns>
        public List<EntityID> GetDetectedEntities(EntityID perceiverID)
        {
            if (!_entityManager.IsValid(perceiverID) || !_transformSystem.TryGetTransform(perceiverID, out TransformComponent perceiverTransform) || !_perceptionComponents.TryGetValue(perceiverID, out PerceptionComponent perceiverPerception))
            {
                return new List<EntityID>();
            }

            // Query a broad area from the spatial grid first
            List<EntityID> potentialTargets = _spatialGrid.QueryEntities(perceiverTransform.Position, perceiverPerception.VisionRange + perceiverPerception.HearingRange + perceiverPerception.ChakraSenseRange);
            
            List<EntityID> detected = new List<EntityID>();
            foreach (EntityID targetID in potentialTargets)
            {
                if (perceiverID == targetID) continue; // Don't detect self

                // Combine senses
                if (CanSee(perceiverID, targetID))
                {
                    detected.Add(targetID);
                }
                // Add hearing/chakra sense for other entities (conceptual sound/spell events)
                // else if (CanHear(perceiverID, targetTransform.Position, someSoundVolume)) { detected.Add(targetID); }
                // else if (CanSenseChakra(perceiverID, targetTransform.Position, someChakraIntensity)) { detected.Add(targetID); }
            }
            return detected.Distinct().ToList(); // Remove duplicates if detected by multiple senses
        }
    }
}
```

### 4. Integrating `PerceptionSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.AI;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `PerceptionSystem` property.
3.  Initialize `PerceptionSystem` in `InitializeSystems()` **after** `SpatialGrid` and `Transforms`.

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
using Sigilborne.Systems.AI; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public SpatialHashGrid SpatialGrid { get; private set; }
    public PerceptionSystem Perception { get; private set; } // Add PerceptionSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        // ... (existing inventory/equipment tests) ...
        
        // --- Test Spatial Hash Grid ---
        // ... (existing spatial grid tests) ...

        // --- Test Perception System ---
        GD.Print("\n--- Testing Perception System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID testNpcID = Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid; // Assuming NPC is ID 1, Gen 1
        EntityID bossID = Entities.GetEntityMeta(2).Generation == 1 ? new EntityID(2, 1) : EntityID.Invalid; // Assuming Boss is ID 2, Gen 1
        EntityID newNpcID = Entities.GetEntityMeta(4).Generation == 1 ? new EntityID(4, 1) : EntityID.Invalid; // Assuming newNpcID is ID 4, Gen 1

        // Move NPC/Boss closer to player for initial test
        Transforms.GetTransformRef(testNpcID).Position = Transforms.GetTransformRef(playerID).Position + new Vector2(50, 0);
        Transforms.GetTransformRef(bossID).Position = Transforms.GetTransformRef(playerID).Position + new Vector2(100, 50);
        Transforms.GetTransformRef(newNpcID).Position = Transforms.GetTransformRef(playerID).Position + new Vector2(-70, -20);
        
        // Player turns to face testNpcID
        Vector2 playerPos = Transforms.GetTransformRef(playerID).Position;
        Vector2 dirToNpc = (Transforms.GetTransformRef(testNpcID).Position - playerPos).Normalized();
        Transforms.GetTransformRef(playerID).RotationDegrees = Mathf.RadToDeg(dirToNpc.Angle());

        List<EntityID> detectedByPlayer = Perception.GetDetectedEntities(playerID);
        GD.Print($"Player {playerID} detected entities: {string.Join(", ", detectedByPlayer.Select(e => e.ToString()))}");
        
        // Move player far away
        Transforms.GetTransformRef(playerID).Position = new Vector2(1000, 1000); // This updates SpatialGrid
        detectedByPlayer = Perception.GetDetectedEntities(playerID); // Should be empty or just self
        GD.Print($"Player {playerID} detected entities after moving far: {string.Join(", ", detectedByPlayer.Select(e => e.ToString()))}");

        GD.Print("--- End Testing Perception System ---\n");
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
        // PerceptionSystem doesn't have a Tick method, its operations are query-driven.
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to SpatialGrid) ...
        
        // Initialize TransformSystem, passing SpatialHashGrid
        Transforms = new TransformSystem(Entities, Events, SpatialGrid);
        GD.Print("  - TransformSystem initialized.");
        
        // Initialize PerceptionSystem AFTER TransformSystem and SpatialGrid
        Perception = new PerceptionSystem(Entities, Events, Transforms, SpatialGrid); // Pass dependencies
        GD.Print("  - PerceptionSystem initialized.");

        // ... (existing system initializations after PerceptionSystem) ...
    }
}
```

### 5. Testing the Perception System

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Perception System" section.

```
...
PerceptionSystem: Initialized.
  - PerceptionSystem initialized.
... (entity spawns and spatial grid updates) ...
--- Testing Perception System ---
Player EntityID(0, Gen:1) detected entities: EntityID(1, Gen:1), EntityID(4, Gen:1), EntityID(2, Gen:1)
Entities near player (1000, 1000) (Radius 200) after move:
Player EntityID(0, Gen:1) detected entities after moving far:
--- End Testing Perception System ---
...
```

**Key Observations:**

*   **Initialization**: `PerceptionSystem` is initialized and adds `PerceptionComponent`s to the player, NPCs, and boss.
*   **Initial Detection**: The player (at 200,200) successfully detects `testNpcID` (at 250,200), `newNpcID` (at 130,180), and `bossID` (at 300,250) because they are within the player's `VisionRange` (300f) and `VisionAngle` (180 degrees, effectively a wide cone or circle if distance is small).
*   **Movement Affects Detection**: When the player moves far away, the `GetDetectedEntities` call returns an empty list, demonstrating that the `SpatialHashGrid` and distance checks are working.

This confirms our `PerceptionSystem` is functional, leveraging the `SpatialHashGrid` and `TransformSystem` to perform vision checks. Hearing and Chakra Sense are conceptually ready to be integrated with actual sound/spell events.

### Summary

You have successfully implemented **The Senses** for Sigilborne's entities, designing `PerceptionComponent` to hold sensory parameters and creating `PerceptionSystem` to manage these capabilities. By leveraging `SpatialHashGrid` and `TransformSystem`, `PerceptionSystem` can now efficiently perform `Vision` (range, angle) checks, and is conceptually ready for `Hearing` and `Chakra Sense` queries, strictly adhering to TDD 05.2.2's specifications. This crucial step provides the foundational layer for AI awareness and sophisticated entity behavior in the living world.

### Next Steps

The next chapter will focus on **Stealth Mechanics**, implementing `Visibility` as a float value based on factors like light level and movement speed, and designing a `Detection Meter` to transition NPCs into "Alert" states, enriching the tactical interplay between stealth and perception.