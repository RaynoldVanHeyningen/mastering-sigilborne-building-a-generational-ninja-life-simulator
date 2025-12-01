## Chapter 2.4: Physics Layer - Hybrid Movement Pipeline

In Sigilborne, accurate and responsive physics are crucial for combat, environmental interaction, and satisfying movement. However, directly managing complex physics simulations in our C# Brain for thousands of entities would be an immense undertaking. This chapter details our **Hybrid Physics Architecture**, where Godot's `CharacterBody2D` (Body) is authoritative for collision detection and separation, while the C# Brain maintains authoritative control over velocity and position. This approach, outlined in TDD 17.1, balances performance with reliable collision resolution.

### 1. The Hybrid Physics Architecture Philosophy

*   **C# Brain**:
    *   **Authoritative for Logic**: Calculates `TargetVelocity`, `ProposedPosition`, and applies all game-specific forces (e.g., knockback, movement input).
    *   **"Wishes" a Position**: It computes where an entity *should* be if there were no obstacles.
*   **GDScript Body**:
    *   **Authoritative for Collision Resolution**: Takes the Brain's "wish" (velocity) and uses Godot's built-in physics engine (`move_and_slide()`) to resolve actual collisions with walls, terrain, and other entities.
    *   **"Reports" Actual Position**: After `move_and_slide()` completes, it reports the *final, collision-resolved position* back to the Brain.
*   **Reconciliation**: The Brain then reconciles its `CurrentPosition` with the Body's `FinalPosition`.

This way, we get Godot's optimized and robust physics engine for collision handling, while keeping our core game logic and state deterministic in C#.

### 2. The Movement Pipeline (TDD 17.2)

Let's formalize the steps involved in each physics frame:

1.  **Brain Calculates Proposed Movement**:
    *   `MovementSystem` (C#) calculates the `TargetVelocity` based on input, speed, friction, etc.
    *   It then calculates `ProposedPosition = CurrentPosition + (TargetVelocity * Delta)`.
    *   **Crucially**: The Brain does *not* apply this `ProposedPosition` directly as the new `CurrentPosition` yet. It just generates the `TargetVelocity` and emits it.
2.  **Brain Emits `EntityPhysicsUpdate`**: The `TransformSystem` (C#) will emit a specific event containing the `EntityID` and the `TargetVelocity` for the Body.
3.  **Body Receives and Resolves Collisions**:
    *   `EntityView.gd` (Body) receives `EntityPhysicsUpdate`.
    *   It calls `CharacterBody2D.move_and_slide()` using the `TargetVelocity` received from the Brain.
    *   Godot's physics engine handles all collisions, sliding along walls, pushing other `CharacterBody2D`s, etc.
4.  **Body Reports Actual Position Back to Brain**:
    *   After `move_and_slide()` completes, `EntityView.gd` gets its new `global_position`.
    *   It calls a C# method (e.g., `GameManager.Instance.Physics.ReportFinalPosition()`) to send this `FinalPosition` back.
5.  **Brain Reconciles**:
    *   A `PhysicsSystem` (C#) receives `FinalPosition`.
    *   It *overwrites* the entity's `TransformComponent.Position` with this `FinalPosition`.

### 3. Implementing the C# `PhysicsSystem` (Brain)

This system will handle the reconciliation step.

1.  Create a new folder `res://_Brain/Systems/Physics/`.
2.  Create `res://_Brain/Systems/Physics/PhysicsSystem.cs`:

```csharp
// _Brain/Systems/Physics/PhysicsSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Movement; // To get VelocityComponent

namespace Sigilborne.Systems.Physics
{
    /// <summary>
    /// Manages the reconciliation of physics positions between the Godot Body and C# Brain.
    /// It receives the final, collision-resolved position from the Body and updates the Brain's authoritative TransformComponent.
    /// </summary>
    public class PhysicsSystem
    {
        private EntityManager _entityManager;
        private TransformSystem _transformSystem;
        private EventBus _eventBus;

        public PhysicsSystem(EntityManager entityManager, TransformSystem transformSystem, EventBus eventBus)
        {
            _entityManager = entityManager;
            _transformSystem = transformSystem;
            _eventBus = eventBus;
            GD.Print("PhysicsSystem: Initialized.");
        }

        /// <summary>
        /// Receives the final, collision-resolved position from the GDScript Body
        /// and updates the Brain's authoritative TransformComponent.
        /// (TDD 17.2.2: Body Reports Actual Position Back to Brain)
        /// </summary>
        /// <param name="idIndex">The index of the entity.</param>
        /// <param name="generation">The generation of the entity (for validation).</param>
        /// <param name="finalPosition">The actual position after Godot's physics resolution.</param>
        /// <param name="finalRotation">The actual rotation after Godot's physics resolution.</param>
        public void ReportFinalPosition(int idIndex, int generation, Vector2 finalPosition, float finalRotation)
        {
            EntityID id = new EntityID(idIndex, generation);

            if (!_entityManager.IsValid(id))
            {
                // GD.PrintErr($"PhysicsSystem: Received final position for invalid or stale entity {id}. Ignoring.");
                return;
            }

            // TDD 17.2.3: Reconciliation - Overwrite Brain's internal CurrentPosition
            if (_transformSystem.TryGetTransform(id, out TransformComponent transform))
            {
                transform.Position = finalPosition;
                transform.RotationDegrees = finalRotation;
                _transformSystem.TrySetTransform(id, transform);
                // GD.Print($"PhysicsSystem: Reconciled {id} to {finalPosition}");
            }
            else
            {
                GD.PrintErr($"PhysicsSystem: Entity {id} has no TransformComponent to reconcile.");
            }
        }
    }
}
```

### 4. Integrating `PhysicsSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Physics;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `PhysicsSystem` property.
3.  Initialize `PhysicsSystem` in `InitializeSystems()`.

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
using Sigilborne.Systems.Physics; // Add this using directive
using Sigilborne.Utils;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public MovementSystem Movement { get; private set; }
    public PhysicsSystem Physics { get; private set; } // Add PhysicsSystem property

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
        Movement.Tick(delta); // Calculates target velocities and updates transforms (proposed positions)
        // Transforms.Tick(delta); // This is no longer needed here. TransformSystem only manages data, doesn't 'tick' in this context.
                                // Its data is updated by MovementSystem, and then reconciled by PhysicsSystem.

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

        Transforms = new TransformSystem(Entities, Events); // Keep TransformSystem initialized
        GD.Print("  - TransformSystem initialized.");
        
        Jobs = new JobSystem(Events);
        GD.Print("  - JobSystem initialized.");

        PlayerStats = new PlayerStatSystem(Events, Entities);
        GD.Print("  - PlayerStatSystem initialized.");

        DebugCommands = new DebugCommandSystem(this);
        GD.Print("  - DebugCommandSystem initialized.");

        Input = new InputSystem();
        GD.Print("  - InputSystem initialized.");

        Movement = new MovementSystem(Entities, Input, Events, Transforms);
        GD.Print("  - MovementSystem initialized.");

        // Initialize PhysicsSystem, passing dependencies
        Physics = new PhysicsSystem(Entities, Transforms, Events); // Initialize PhysicsSystem here
        GD.Print("  - PhysicsSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```
**Important Correction**: `TransformSystem.Tick(delta)` was previously updating transforms based on simulated velocity *and* emitting `EntityMovedEvent`. This is fine for testing simple movement, but for a full physics pipeline, `TransformSystem` should *not* be directly ticking movement after `MovementSystem` calculates the velocity. Instead, `MovementSystem` should calculate the velocity, and `EntityView.gd` should apply it via `move_and_slide()`, then report back. The `TransformSystem`'s `Tick` method can be removed or repurposed for other transform-related logic (e.g., updating children transforms if we had a hierarchy).

Let's adjust `TransformSystem.cs` to remove its internal movement logic, as movement is now owned by `MovementSystem` and resolved by `EntityView`'s `move_and_slide`. `TransformSystem`'s `Tick` will be empty for now.

```csharp
// _Brain/Systems/TransformSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;

namespace Sigilborne.Systems
{
    public class TransformSystem
    {
        private Dictionary<EntityID, TransformComponent> _transforms = new Dictionary<EntityID, TransformComponent>();
        private EntityManager _entityManager;
        private EventBus _eventBus;

        public TransformSystem(EntityManager entityManager, EventBus eventBus)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            eventBus.OnEntitySpawned += OnEntitySpawned;
            eventBus.OnEntityDespawned += OnEntityDespawned;
            GD.Print("TransformSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityManager.EntitySpawnedEvent e)
        {
            _transforms.Add(e.ID, new TransformComponent(e.InitialPosition, e.InitialRotation));
            GD.Print($"TransformSystem: Added transform for {e.ID}");
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            if (_transforms.ContainsKey(e.ID))
            {
                _transforms.Remove(e.ID);
                GD.Print($"TransformSystem: Removed transform for {e.ID}");
            }
        }

        // ... (TryGetTransform, TrySetTransform, GetTransformRef methods) ...

        public void Tick(double delta)
        {
            // TDD 17.2: TransformSystem no longer directly updates positions or rotations based on its own tick.
            // Movement is now handled by MovementSystem calculating velocity, and EntityView resolving collisions.
            // TransformSystem's job is to store and provide the authoritative transform data,
            // and emit events when that data is externally changed (e.g., by PhysicsSystem after collision).
            // For now, this method will be empty.
        }
    }
}
```

### 5. Modifying `MovementSystem` to Emit `EntityPhysicsUpdate`

Instead of directly modifying `TransformComponent.Position`, `MovementSystem` should now calculate the `TargetVelocity` and emit it. The `TransformSystem` will then publish `EntityMovedEvent` when its data is *set* (e.g., by `PhysicsSystem`).

Let's introduce a new event for the Body to receive the velocity.

1.  Add `EntityVelocityUpdateEvent` struct to `_Brain/Entities/EntityManager.cs`:

```csharp
// _Brain/Entities/EntityManager.cs (inside Sigilborne.Entities namespace)
// ...
        // --- Helper Events for Body Sync (TDD 11.4) ---
        // ... (existing events) ...
        public struct EntityVelocityUpdateEvent { public EntityID ID; public Vector2 TargetVelocity; public float TargetRotationDegrees; } // New event
    }
}
```
2.  Add `OnEntityVelocityUpdate` delegate to `_Brain/Core/EventBus.cs`:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...
        public event Action<EntityID, Vector2, float> OnEntityVelocityUpdate; // New event

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is EntityManager.EntityVelocityUpdateEvent velUpdateEvent) // New condition
            {
                OnEntityVelocityUpdate?.Invoke(velUpdateEvent.ID, velUpdateEvent.TargetVelocity, velUpdateEvent.TargetRotationDegrees);
            }
            else if (eventData is EntityManager.EntityMovedEvent movedEvent) // Existing, but now published by PhysicsSystem
            {
                OnEntityMoved?.Invoke(movedEvent.ID, movedEvent.NewPosition, movedEvent.NewRotation);
            }
            // ... (rest of conditions) ...
        }
        // ... (AddCommand and FlushCommands methods) ...
    }
}
```

3.  Modify `_Brain/Systems/Movement/MovementSystem.cs`:
    *   Remove the `transform.Position` and `transform.RotationDegrees` updates.
    *   Instead, publish `EntityVelocityUpdateEvent`.

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
    // ... (VelocityComponent, MovementParametersComponent, PlayerMovementStateComponent structs) ...

    public class MovementSystem
    {
        private EntityManager _entityManager;
        private InputSystem _inputSystem;
        private EventBus _eventBus;
        private TransformSystem _transformSystem; // Still needed for TryGetTransform etc.

        // ... (dictionaries, constructor) ...

        public void Tick(double delta)
        {
            PlayerInputFrame currentInput = _inputSystem.GetLatestInput();

            foreach (var kvp in _velocities)
            {
                EntityID id = kvp.Key;

                if (!_entityManager.IsValid(id) || !_movementParams.TryGetValue(id, out MovementParametersComponent moveParams))
                {
                    continue;
                }

                ref VelocityComponent currentVelocity = ref _velocities.GetValueRef(id);
                Vector2 targetMoveVector = currentInput.MoveVector;
                float currentSpeed = moveParams.BaseSpeed;
                float targetRotationDegrees = _transformSystem.GetTransformRef(id).RotationDegrees; // Keep current rotation

                // --- Player-specific Movement Logic (Shift-Sliding) ---
                if (id == _entityManager.GetPlayerEntityID())
                {
                    ref PlayerMovementStateComponent playerMoveState = ref _playerMovementStates.GetValueRef(id);

                    if (currentInput.IsShiftHeld)
                    {
                        if (targetMoveVector.LengthSquared() > 0.1f)
                        {
                            playerMoveState.LastLockedMoveVector = targetMoveVector;
                            playerMoveState.IsShiftSliding = true;
                        }
                        targetMoveVector = playerMoveState.LastLockedMoveVector;
                        playerMoveState.IsShiftSliding = (targetMoveVector.LengthSquared() > 0.1f);
                    }
                    else
                    {
                        playerMoveState.IsShiftSliding = false;
                        playerMoveState.LastLockedMoveVector = Vector2.Zero;
                    }
                }
                // --- End Player-specific Movement Logic ---

                if (currentInput.IsSprintHeld && !(_playerMovementStates.TryGetValue(id, out var state) && state.IsShiftSliding))
                {
                    currentSpeed *= moveParams.SprintMultiplier;
                }

                Vector2 desiredVelocity = targetMoveVector * currentSpeed;

                currentVelocity.Velocity = currentVelocity.Velocity.Lerp(desiredVelocity, moveParams.Acceleration * (float)delta);

                if (!(_playerMovementStates.TryGetValue(id, out var state2) && state2.IsShiftSliding) && targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() > 0.1f)
                {
                    currentVelocity.Velocity = currentVelocity.Velocity.Lerp(Vector2.Zero, moveParams.Friction * (float)delta);
                }
                else if (!(_playerMovementStates.TryGetValue(id, out var state3) && state3.IsShiftSliding) && targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() <= 0.1f)
                {
                    currentVelocity.Velocity = Vector2.Zero;
                }

                // --- Determine Target Rotation ---
                if (targetMoveVector.LengthSquared() > 0.1f)
                {
                    targetRotationDegrees = targetMoveVector.Angle() * (180f / Mathf.Pi);
                }


                // TDD 17.2.1: Brain calculates TargetVelocity and emits it.
                // It does NOT modify TransformComponent.Position directly here anymore.
                _eventBus.Publish(new EntityManager.EntityVelocityUpdateEvent { ID = id, TargetVelocity = currentVelocity.Velocity, TargetRotationDegrees = targetRotationDegrees });
            }
        }
    }
}
```

### 6. Modifying `EntityView.gd` to use `move_and_slide()` and Report Back

Now, `EntityView.gd` needs to:
1.  Connect to `OnEntityVelocityUpdate`.
2.  Store the `TargetVelocity` from C#.
3.  In `_physics_process`, call `move_and_slide()` with this velocity.
4.  After `move_and_slide()`, call `GameManager.Instance.Physics.ReportFinalPosition()` to send the actual, collision-resolved position back to the C# Brain.

Open `res://_Body/Scripts/Visuals/EntityView.gd`:

```gdscript
# _Body/Scripts/Visuals/EntityView.gd
class_name EntityView extends CharacterBody2D

# ... (existing properties and node references) ...

# --- Visual Interpolation Parameters (TDD 16.4) ---
var brain_target_position: Vector2 = Vector2.ZERO # For visual interpolation
var brain_target_velocity: Vector2 = Vector2.ZERO # New: Velocity received from Brain
var brain_target_rotation_degrees: float = 0.0 # New: Rotation received from Brain
const SMOOTHING_SPEED: float = 10.0

# ... (existing signals) ...

func setup(id: int, initial_position: Vector2, initial_rotation: float, definition_id: String) -> void:
    entity_id = id
    global_position = initial_position
    brain_target_position = initial_position
    brain_target_rotation_degrees = initial_rotation
    visuals.rotation_degrees = initial_rotation # Set initial visual rotation
    
    # ... (animation_player setup) ...

    GD.print("EntityView: Setup for C# EntityID %s (Def: %s) at %s" % [entity_id, definition_id, global_position])
    
    # --- Connect to Brain Signals ---
    # Connect to the new velocity update event
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        GameManager.Instance.Events.OnEntityVelocityUpdate.connect(Callable(self, "_on_entity_velocity_update"))
        # We also need to connect to the OnEntityMoved for reconciliation to get the final authoritative position
        GameManager.Instance.Events.OnEntityMoved.connect(Callable(self, "_on_entity_moved_reconciliation"))
    pass

## Called every physics frame (fixed rate) to update the visual position.
## (TDD 17.2.2: Body Receives and Resolves Collisions)
func _physics_process(delta: float) -> void:
    # Interpolate visual position towards the Brain's authoritative position for smoothness
    # (This is separate from the physics movement, just for visual lag compensation)
    global_position = global_position.lerp(brain_target_position, delta * SMOOTHING_SPEED)
    visuals.rotation_degrees = lerp_angle(visuals.rotation_degrees, brain_target_rotation_degrees, delta * SMOOTHING_SPEED)


    # TDD 17.2.2: Body calls CharacterBody2D.move_and_slide() using Brain's velocity.
    # Note: CharacterBody2D.velocity is implicitly used by move_and_slide().
    # We set this CharacterBody2D's velocity to the one received from the Brain.
    velocity = brain_target_velocity
    move_and_slide()

    # TDD 17.2.2: Body Reports Actual Position Back to Brain.
    # After move_and_slide, global_position is the actual, collision-resolved position.
    if GameManager.Instance != null and GameManager.Instance.Physics != null:
        # We need the EntityID's generation for validation in C#
        var entity_id_struct = GameManager.Instance.Entities.GetEntityMeta(entity_id).Generation
        # This assumes entity_id is the index.
        GameManager.Instance.Physics.ReportFinalPosition(entity_id, GameManager.Instance.Entities.GetEntityMeta(entity_id).Generation, global_position, visuals.rotation_degrees)
    # else:
        # push_error("EntityView: GameManager or PhysicsSystem not ready to report final position.")


## Handler for C# EntityMovedEvent. This is for reconciliation.
## (TDD 17.2.3: Brain Reconciles)
func _on_entity_moved_reconciliation(id: int, new_position: Vector2, new_rotation: float) -> void:
    if entity_id == id:
        # This is the authoritative position from the Brain after reconciliation.
        # We set our visual interpolation target to this.
        brain_target_position = new_position
        brain_target_rotation_degrees = new_rotation
        # GD.print("EntityView %s: Reconciled to Brain position %s" % [entity_id, new_position])

## Handler for C# EntityVelocityUpdateEvent.
## Parameters must match Action<EntityID, Vector2, float> OnEntityVelocityUpdate in C#.
func _on_entity_velocity_update(id: int, target_velocity: Vector2, target_rotation_degrees: float) -> void:
    if entity_id == id:
        # Store the target velocity and rotation from the Brain.
        # move_and_slide() will use this in the next _physics_process call.
        brain_target_velocity = target_velocity
        brain_target_rotation_degrees = target_rotation_degrees
        # GD.print("EntityView %s: Received target velocity %s" % [entity_id, target_velocity])
```

**Key Changes in `EntityView.gd`:**

*   **`brain_target_velocity`**: New property to store the velocity received from the Brain.
*   **`_physics_process`**:
    *   Now sets `velocity = brain_target_velocity`.
    *   Calls `move_and_slide()`.
    *   Calls `GameManager.Instance.Physics.ReportFinalPosition()` after `move_and_slide()` to report the actual position.
*   **`_on_entity_velocity_update`**: Connects to the new C# event and stores the `target_velocity`.
*   **`_on_entity_moved_reconciliation`**: Renamed the old `_on_entity_moved` to clarify its role in reconciliation.

### 7. Creating a Collision Layer (Body)

For `move_and_slide()` to work, we need a simple collision object. Let's add a `StaticBody2D` with a `CollisionShape2D` to `Gameplay.tscn`.

1.  Open `res://Gameplay.tscn`.
2.  Add a `StaticBody2D` node. Rename it `Wall`.
3.  Add a `CollisionShape2D` as a child of `Wall`.
4.  In the `CollisionShape2D`'s Inspector, set its `Shape` to `New RectangleShape2D`.
5.  Select the `RectangleShape2D` resource. Set its `Size` to `Vector2(20, 200)` (or any size for a vertical wall).
6.  Position the `Wall` (e.g., `x=400, y=300`) so the player can collide with it.
7.  Save `Gameplay.tscn`.

### 8. Testing the Hybrid Physics Pipeline

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Move the player (WASD).
    *   Observe the player moving smoothly.
    *   Try to move the player into the `Wall` you created. The player should slide along the wall, not pass through it.
    *   Use Shift-Sliding (`Alt` + WASD, then release WASD while holding `Alt`). The player should continue sliding in the locked direction, even against the wall.

**Expected Output (in debug console, if you print `_latestInput` in `InputSystem`):**

You'll see `InputSystem: Processed latest input...` showing your WASD and Shift/Alt states. You should also see many `PhysicsSystem: Reconciled...` messages, confirming the Body is reporting back to the Brain.

This confirms the hybrid physics pipeline is functional: the C# Brain dictates the intent (velocity), Godot's Body resolves the actual collisions, and the Body reports the final position back to the Brain for reconciliation.

### Summary

You have successfully implemented Sigilborne's **Hybrid Physics Architecture**, leveraging Godot's `CharacterBody2D` for collision detection and resolution while the C# Brain maintains authoritative control over velocity and position. By designing a `PhysicsSystem` for reconciliation and modifying `MovementSystem` and `EntityView.gd` to work in concert, you've established a robust pipeline where the Brain calculates intent, the Body resolves physical interactions via `move_and_slide()`, and the final position is reported back to the Brain, strictly adhering to TDD 17's specifications.

### Next Steps

This concludes **Module 2: Player Input & Core Movement**. We will now move on to **Module 3: The Glyph System - Language of Ninjutsu**, where we will begin to design and implement the procedural generation of glyph meanings and the player's discovery mechanics.