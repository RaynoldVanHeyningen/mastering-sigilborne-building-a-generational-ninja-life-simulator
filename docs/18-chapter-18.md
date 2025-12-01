## Chapter 2.3: Shift-Sliding Mechanic - Direction Lock (C#)

Building on our standard movement, this chapter introduces Sigilborne's unique "Shift-Sliding" mechanic. This mechanic allows the player to "lock" their last movement direction by holding a specific key (e.g., `Alt`), causing the character to continue moving in that direction even if the directional input is released. This adds a tactical layer to movement, allowing for precise positioning while maintaining focus on other actions. This feature is directly specified in TDD 12.3.

### 1. Understanding the Shift-Sliding Mechanic

*   **Core Idea**: When the "Shift" key (our `IsShiftHeld` input) is pressed, the player's movement direction becomes "locked" to the last non-zero `MoveVector`.
*   **Behavior**:
    *   If `Shift` is held and directional input is active, the player moves normally, and the locked direction updates.
    *   If `Shift` is held and directional input is *released*, the player continues moving in the *last locked direction* at their current speed, rather than stopping or decelerating due to friction.
    *   If `Shift` is released, movement reverts to normal (friction applies, player stops if no directional input).
*   **Benefit**: Allows players to maintain momentum or precise positioning while freeing up their directional input for other complex actions (e.g., casting glyphs without needing to hold WASD).

### 2. Enhancing `MovementSystem.cs` for Shift-Sliding

We will modify our `MovementSystem` to incorporate this new logic. We'll need a way to store the `LastMoveVector` when the `IsShiftHeld` flag is active.

Open `res://_Brain/Systems/Movement/MovementSystem.cs`:

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
    // ... (VelocityComponent and MovementParametersComponent structs) ...

    /// <summary>
    /// Component to store specific player movement states like the last locked direction for shift-sliding.
    /// </summary>
    public struct PlayerMovementStateComponent
    {
        public Vector2 LastLockedMoveVector; // Stores the last non-zero move vector when shift is held
        public bool IsShiftSliding; // True if currently in shift-sliding mode (shift is held and moving)

        public PlayerMovementStateComponent(Vector2 lastLockedMoveVector = default, bool isShiftSliding = false)
        {
            LastLockedMoveVector = lastLockedMoveVector;
            IsShiftSliding = isShiftSliding;
        }
    }


    /// <summary>
    /// Manages the movement logic for entities.
    /// Reads input, calculates velocity, and updates TransformComponents.
    /// </summary>
    public class MovementSystem
    {
        private EntityManager _entityManager;
        private InputSystem _inputSystem;
        private EventBus _eventBus;
        private TransformSystem _transformSystem; 

        private Dictionary<EntityID, VelocityComponent> _velocities = new Dictionary<EntityID, VelocityComponent>();
        private Dictionary<EntityID, MovementParametersComponent> _movementParams = new Dictionary<EntityID, MovementParametersComponent>();
        private Dictionary<EntityID, PlayerMovementStateComponent> _playerMovementStates = new Dictionary<EntityID, PlayerMovementStateComponent>(); // New: For player-specific movement states

        public MovementSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus, TransformSystem transformSystem)
        {
            _entityManager = entityManager;
            _inputSystem = inputSystem;
            _eventBus = eventBus;
            _transformSystem = transformSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("MovementSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player) // Only player gets movement for now
            {
                _velocities.Add(id, new VelocityComponent(Vector2.Zero));
                _movementParams.Add(id, new MovementParametersComponent(150f, 1.5f, 0.8f, 10f));
                _playerMovementStates.Add(id, new PlayerMovementStateComponent()); // Add player-specific state
                GD.Print($"MovementSystem: Added movement components for Player {id}");
            }
        }

        private void OnEntityDespawned(EntityID id)
        {
            _velocities.Remove(id);
            _movementParams.Remove(id);
            _playerMovementStates.Remove(id); // Remove player-specific state
            GD.Print($"MovementSystem: Removed movement components for {id}");
        }

        /// <summary>
        /// Main update loop for the MovementSystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
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

                // --- Player-specific Movement Logic (Shift-Sliding) ---
                if (id == _entityManager.GetPlayerEntityID()) // Assuming we have a way to get player's ID
                {
                    ref PlayerMovementStateComponent playerMoveState = ref _playerMovementStates.GetValueRef(id);

                    // TDD 12.3: Shift-Sliding Logic
                    if (currentInput.IsShiftHeld)
                    {
                        if (targetMoveVector.LengthSquared() > 0.1f)
                        {
                            // If shift is held AND directional input is active, update the locked vector
                            playerMoveState.LastLockedMoveVector = targetMoveVector;
                            playerMoveState.IsShiftSliding = true;
                        }
                        // If shift is held but directional input is released, continue with LastLockedMoveVector
                        targetMoveVector = playerMoveState.LastLockedMoveVector;
                        playerMoveState.IsShiftSliding = (targetMoveVector.LengthSquared() > 0.1f); // Only sliding if there's a locked direction
                    }
                    else
                    {
                        // Shift is not held, reset sliding state
                        playerMoveState.IsShiftSliding = false;
                        playerMoveState.LastLockedMoveVector = Vector2.Zero; // Clear locked vector
                    }
                }
                // --- End Player-specific Movement Logic ---


                if (currentInput.IsSprintHeld && !(_playerMovementStates.TryGetValue(id, out var state) && state.IsShiftSliding)) // Cannot sprint while shift-sliding
                {
                    currentSpeed *= moveParams.SprintMultiplier;
                }

                Vector2 desiredVelocity = targetMoveVector * currentSpeed;

                currentVelocity.Velocity = currentVelocity.Velocity.Lerp(desiredVelocity, moveParams.Acceleration * (float)delta);

                // Apply friction only if not shift-sliding and no input
                if (!(_playerMovementStates.TryGetValue(id, out var state2) && state2.IsShiftSliding) && targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() > 0.1f)
                {
                    currentVelocity.Velocity = currentVelocity.Velocity.Lerp(Vector2.Zero, moveParams.Friction * (float)delta);
                }
                else if (!(_playerMovementStates.TryGetValue(id, out var state3) && state3.IsShiftSliding) && targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() <= 0.1f)
                {
                    currentVelocity.Velocity = Vector2.Zero;
                }

                // --- Update TransformComponent ---
                if (_transformSystem.TryGetTransform(id, out TransformComponent transform))
                {
                    transform.Position += currentVelocity.Velocity * (float)delta;
                    // Rotation follows MoveVector or LastLockedMoveVector if sliding
                    if (targetMoveVector.LengthSquared() > 0.1f)
                    {
                        transform.RotationDegrees = targetMoveVector.Angle() * (180f / Mathf.Pi);
                    }
                    _transformSystem.TrySetTransform(id, transform);
                }
            }
        }
    }
}
```

**Key Changes in `MovementSystem.cs`:**

*   **`PlayerMovementStateComponent`**: A new struct to store `LastLockedMoveVector` and `IsShiftSliding` specifically for the player.
*   **`_playerMovementStates`**: A dictionary to hold `PlayerMovementStateComponent` instances.
*   **`OnEntitySpawned` / `OnEntityDespawned`**: Updated to add/remove `PlayerMovementStateComponent` for the player entity.
*   **Shift-Sliding Logic**:
    *   If `currentInput.IsShiftHeld` is true:
        *   If `MoveVector` is active, `LastLockedMoveVector` is updated.
        *   `targetMoveVector` is then set to `LastLockedMoveVector`, effectively continuing movement in the locked direction even if `currentInput.MoveVector` is zero.
        *   `IsShiftSliding` is set.
    *   If `IsShiftHeld` is false, `IsShiftSliding` is reset, and `LastLockedMoveVector` is cleared.
*   **Friction and Sprinting Conditional**: Friction is only applied if the player is *not* currently shift-sliding. Sprinting is also conditionally applied, preventing sprinting while shift-sliding for a balanced mechanic.

#### 2.1. Update `EntityManager.cs` to Get Player ID

The `MovementSystem` needs to know which `EntityID` belongs to the player. We can add a simple property to `EntityManager` for this.

Open `_Brain/Entities/EntityManager.cs` and add this property:

```csharp
// _Brain/Entities/EntityManager.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;

namespace Sigilborne.Entities
{
    // ... (EntityID, EntityMeta, EntityType structs/enums) ...

    public class EntityManager
    {
        // ... (MAX_ENTITIES, _entityMetas, _freeIndices, _eventBus) ...
        private EntityID _playerEntityID = EntityID.Invalid; // New: Store player's EntityID

        public EntityManager(EventBus eventBus) { /* ... */ }

        public EntityID CreateEntity(EntityType type, string definitionID, Vector2 initialPosition = default, float initialRotation = 0f)
        {
            // ... (slot allocation, generation increment, EntityMeta setup) ...

            EntityID newId = new EntityID(index, _entityMetas[index].Generation);
            GD.Print($"EntityManager: Created {type} entity {newId} (Def: {definitionID})");

            if (type == EntityType.Player) // Store player's ID when created
            {
                _playerEntityID = newId;
            }

            _eventBus.Publish(new EntitySpawnedEvent { ID = newId, Type = type, DefinitionID = definitionID, InitialPosition = initialPosition, InitialRotation = initialRotation });

            return newId;
        }

        public void DestroyEntity(EntityID id)
        {
            // ... (existing destruction logic) ...
            if (id == _playerEntityID) // Clear player's ID if destroyed
            {
                _playerEntityID = EntityID.Invalid;
            }
        }

        public bool IsValid(EntityID id) { /* ... */ }
        public ref EntityMeta GetEntityMeta(EntityID id) { /* ... */ }

        /// <summary>
        /// Returns the EntityID of the player character.
        /// </summary>
        public EntityID GetPlayerEntityID()
        {
            return _playerEntityID;
        }

        // ... (Helper Events for Body Sync) ...
    }
}
```

### 3. Testing the Shift-Sliding Mechanic

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  **Test 1 (Normal Movement)**: Press WASD. The player moves. Release WASD. The player stops due to friction.
5.  **Test 2 (Shift-Sliding)**:
    *   Hold `Alt` (our `shift` key).
    *   Press `W` (move up). The player moves up.
    *   While still holding `Alt`, release `W`. The player should *continue moving up* without friction applying.
    *   Press `A` (move left) while holding `Alt`. The player's direction should change to left, and they continue moving left when `A` is released (as long as `Alt` is held).
    *   Release `Alt`. The player should stop if no directional input is held, or revert to normal movement if directional input is held.
6.  **Test 3 (Sprint vs. Shift-Slide)**:
    *   Hold `Alt` and move (shift-sliding).
    *   Try to press `Shift` (sprint). The player should *not* sprint faster, as we've added a condition to prevent sprinting while shift-sliding.

This confirms the Shift-Sliding mechanic is working as intended, providing the player with a directional lock based on input.

### Summary

You have successfully implemented the **Shift-Sliding Mechanic** in the C# Brain, allowing the player to lock their last movement direction for tactical precision. By introducing the `PlayerMovementStateComponent` and integrating its logic into the `MovementSystem`, you've created a unique traversal ability that dynamically responds to player input, strictly adhering to TDD 12.3's specifications. This enhances player agency and adds depth to movement, which is crucial for Sigilborne's systemic gameplay.

### Next Steps

The next chapter will focus on the **Physics Layer**, detailing how our hybrid physics architecture leverages Godot's `CharacterBody2D` for collision detection and resolution while the C# Brain maintains authoritative control over velocity and position.