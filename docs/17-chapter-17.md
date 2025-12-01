## Chapter 2.2: Standard Movement Logic - Velocity & Friction (C#)

Now that our GDScript `InputManager` is reliably sending player input snapshots to the C# Brain's `InputSystem`, it's time to translate that input into actual character movement within our simulation. This chapter focuses on implementing the standard movement logic in C#, defining how `MoveVector` from the `PlayerInputFrame` is converted into a `Velocity`, and how friction is applied for a snappy, responsive feel, as specified in TDD 12.3.

### 1. The Movement System's Role in the Brain

Our movement logic resides entirely in the C# Brain. This ensures:

*   **Authoritative Movement**: The Brain dictates the character's true position and velocity.
*   **Determinism**: Movement calculations are consistent across all runs.
*   **Decoupling**: The visual movement (interpolation in `EntityView.gd`) is separate from the logical movement.

We will create a new `MovementSystem` in C# that processes the `PlayerInputFrame` and updates the `TransformComponent` of the player entity.

### 2. Implementing `MovementSystem.cs`

This system will manage `TransformComponent` (position, rotation) and potentially a new `VelocityComponent` (current speed and direction).

1.  Create a new folder `res://_Brain/Systems/Movement/`.
2.  Create `res://_Brain/Systems/Movement/MovementSystem.cs`:

```csharp
// _Brain/Systems/Movement/MovementSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Input; // To read PlayerInputFrame

namespace Sigilborne.Systems.Movement
{
    /// <summary>
    /// Component to store an entity's current velocity.
    /// </summary>
    public struct VelocityComponent
    {
        public Vector2 Velocity;

        public VelocityComponent(Vector2 velocity)
        {
            Velocity = velocity;
        }

        public override string ToString()
        {
            return $"Vel: {Velocity}";
        }
    }

    /// <summary>
    /// Component to store an entity's movement parameters.
    /// </summary>
    public struct MovementParametersComponent
    {
        public float BaseSpeed;
        public float SprintMultiplier;
        public float Friction; // For snappy stops
        public float Acceleration; // How fast to reach max speed

        public MovementParametersComponent(float baseSpeed, float sprintMultiplier, float friction, float acceleration)
        {
            BaseSpeed = baseSpeed;
            SprintMultiplier = sprintMultiplier;
            Friction = friction;
            Acceleration = acceleration;
        }
    }


    /// <summary>
    /// Manages the movement logic for entities.
    /// Reads input, calculates velocity, and updates TransformComponents.
    /// </summary>
    public class MovementSystem
    {
        private EntityManager _entityManager;
        private InputSystem _inputSystem; // To get player input
        private EventBus _eventBus;

        // Store VelocityComponents and MovementParametersComponents for entities that move.
        private Dictionary<EntityID, VelocityComponent> _velocities = new Dictionary<EntityID, VelocityComponent>();
        private Dictionary<EntityID, MovementParametersComponent> _movementParams = new Dictionary<EntityID, MovementParametersComponent>();

        // Reference to the TransformSystem to update positions
        private TransformSystem _transformSystem; 

        public MovementSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus, TransformSystem transformSystem)
        {
            _entityManager = entityManager;
            _inputSystem = inputSystem;
            _eventBus = eventBus;
            _transformSystem = transformSystem; // Get reference to TransformSystem

            // Subscribe to entity lifecycle events to manage components
            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("MovementSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player) // Only player gets movement for now
            {
                _velocities.Add(id, new VelocityComponent(Vector2.Zero));
                _movementParams.Add(id, new MovementParametersComponent(150f, 1.5f, 0.8f, 10f)); // BaseSpeed 150, Sprint 1.5x, Friction 0.8, Accel 10
                GD.Print($"MovementSystem: Added movement components for Player {id}");
            }
            // NPCs/Animals will have their own AI-driven movement, handled later.
        }

        private void OnEntityDespawned(EntityID id)
        {
            _velocities.Remove(id);
            _movementParams.Remove(id);
            GD.Print($"MovementSystem: Removed movement components for {id}");
        }

        /// <summary>
        /// Main update loop for the MovementSystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            // Get the latest input from the InputSystem
            PlayerInputFrame currentInput = _inputSystem.GetLatestInput();

            // Iterate over all entities that have movement components (currently just player)
            foreach (var kvp in _velocities)
            {
                EntityID id = kvp.Key;

                // Ensure entity is valid and has movement parameters
                if (!_entityManager.IsValid(id) || !_movementParams.TryGetValue(id, out MovementParametersComponent moveParams))
                {
                    continue;
                }

                // Get current velocity (mutable reference)
                ref VelocityComponent currentVelocity = ref _velocities.GetValueRef(id);

                // --- Standard Movement Logic (TDD 12.3) ---
                Vector2 targetMoveVector = currentInput.MoveVector;
                float currentSpeed = moveParams.BaseSpeed;

                if (currentInput.IsSprintHeld)
                {
                    currentSpeed *= moveParams.SprintMultiplier;
                }

                Vector2 desiredVelocity = targetMoveVector * currentSpeed;

                // Apply acceleration to smoothly reach desired velocity
                currentVelocity.Velocity = currentVelocity.Velocity.Lerp(desiredVelocity, moveParams.Acceleration * (float)delta);

                // If no input, apply friction for snappy stops (TDD 12.3)
                if (targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() > 0.1f)
                {
                    currentVelocity.Velocity = currentVelocity.Velocity.Lerp(Vector2.Zero, moveParams.Friction * (float)delta);
                }
                else if (targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() <= 0.1f)
                {
                    currentVelocity.Velocity = Vector2.Zero; // Snap to zero to prevent tiny residual movement
                }

                // --- Update TransformComponent ---
                if (_transformSystem.TryGetTransform(id, out TransformComponent transform))
                {
                    // ProposedPosition = CurrentPosition + (Velocity * Delta) (TDD 17.2.1)
                    transform.Position += currentVelocity.Velocity * (float)delta;
                    // For player, rotation should follow LookVector or MoveVector if no LookVector.
                    // For now, let's make it follow MoveVector if moving.
                    if (targetMoveVector.LengthSquared() > 0.1f)
                    {
                        transform.RotationDegrees = targetMoveVector.Angle() * (180f / Mathf.Pi); // Convert radians to degrees
                    }
                    _transformSystem.TrySetTransform(id, transform);
                }
            }
        }
    }
}
```

**Explanation of `MovementSystem.cs`:**

*   **`VelocityComponent`**: A new struct to hold the entity's current velocity.
*   **`MovementParametersComponent`**: A new struct to define an entity's speed, sprint multiplier, friction, and acceleration.
*   **`_velocities` / `_movementParams`**: Dictionaries to store these components, mapped by `EntityID`.
*   **`OnEntitySpawned` / `OnEntityDespawned`**: Event handlers to add/remove movement components when entities (currently just the player) are created/destroyed.
*   **`Tick(double delta)`**:
    *   Retrieves the `PlayerInputFrame` from `_inputSystem`.
    *   Iterates over entities with movement components.
    *   Calculates `desiredVelocity` based on `MoveVector` and `sprint`.
    *   Applies `acceleration` for smooth start/stop.
    *   Applies `friction` when `MoveVector` is zero (TDD 12.3).
    *   Updates the `TransformComponent.Position` by adding `currentVelocity * delta`.
    *   Updates `TransformComponent.RotationDegrees` to face the movement direction.

### 3. Integrating `MovementSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Movement;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `MovementSystem` property.
3.  Initialize `MovementSystem` in `InitializeSystems()`.
4.  Call `MovementSystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Input;
using Sigilborne.Systems.Movement; // Add this using directive
using Sigilborne.Utils;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public InputSystem Input { get; private set; }
    public MovementSystem Movement { get; private set; } // Add MovementSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
    }

    public override void _PhysicsProcess(double delta)
    {
        // TDD 01.6: Phase 1: Process Input Buffer.
        Input.ProcessInputBuffer(); 

        // TDD 01.6: Phase 2: Run Simulation Systems.
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta); // Call MovementSystem's tick method
        Transforms.Tick(delta); // Transforms are updated by MovementSystem, but also other systems, so it ticks.

        // TDD 01.6: Phase 3: Resolve Collisions/Events.
        Events.FlushCommands();

        // TDD 01.6: Phase 4: Emit State Update Signals. (Implicitly handled)
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

        PlayerStats = new PlayerStatSystem(Events, Entities);
        GD.Print("  - PlayerStatSystem initialized.");

        DebugCommands = new DebugCommandSystem(this);
        GD.Print("  - DebugCommandSystem initialized.");

        Input = new InputSystem();
        GD.Print("  - InputSystem initialized.");

        // Initialize MovementSystem, passing all necessary dependencies
        Movement = new MovementSystem(Entities, Input, Events, Transforms); // Initialize MovementSystem here
        GD.Print("  - MovementSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 4. Testing Standard Movement

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Use WASD keys to move the player entity.
    *   You should see the player sprite (the one that briefly appears and then doesn't get destroyed if you commented out `Entities.DestroyEntity(playerEntity)` in `GameManager._Ready()`) moving and rotating.
    *   The movement should feel responsive, accelerating and decelerating smoothly due to `acceleration` and `friction`.
    *   Press Left Shift to sprint, and the movement speed should increase.

**Important Note on Visuals**: The player entity (`EntityID 0`) is still the generic `EntityRoot.tscn`. It doesn't have a specific player sprite yet. It's just a placeholder circle. In a later chapter (e.g., in Module 1.9 or Module 6), we'd swap this to a proper player character scene. For now, observe its movement behavior.

Also, remember the NPC (`EntityID 1`) is still moving right and rotating as per our `TransformSystem.Tick` test from Chapter 1.9. Its movement is independent of player input.

### Summary

You have successfully implemented the **Standard Movement Logic** in the C# Brain, translating player input from the `InputSystem` into authoritative `Velocity` and `TransformComponent` updates. By creating a `MovementSystem` that incorporates `MovementParametersComponent` and applies `acceleration` and `friction`, you've achieved responsive and smooth character movement, strictly adhering to TDD 12.3's specifications. This is a crucial step in bringing our player entity to life within Sigilborne's simulation.

### Next Steps

The next chapter will build on this by implementing the **Shift-Sliding Mechanic**, a unique movement ability that allows the player to lock their last movement direction, adding a tactical layer to traversal.