## Chapter 5.2: Core Stats (Struct)

In Sigilborne, every living entity (player, NPC, animal) possesses a set of fundamental statistics that define its capabilities and well-being. This chapter focuses on thoroughly defining the **Core Stats** struct, building upon our existing `BioState` and `PlayerStats` concepts. This will centralize stat management, making it efficient to access and modify these values across various systems in the C# Brain, as specified in TDD 03.2.

### 1. The Need for a Unified Core Stats Definition

We currently have `BioState` (for hunger, thirst, temp) and `PlayerStats` (for health, chakra, stamina). While `BioState` is authoritative for biological needs, `PlayerStats` is a bit redundant now that `BiologicalSystem` is the source of truth for the player's core combat stats.

A single, comprehensive `CoreStats` struct will:

*   **Centralize Data**: All critical stats in one place.
*   **Efficiency**: `struct` for value semantics and cache-friendliness.
*   **Clarity**: Clear definition of an entity's capabilities.
*   **Flexibility**: Easily extendable for new stats.

### 2. Defining `CoreStats.cs`

This `CoreStats` struct will replace the role of `PlayerStats` and be the primary stat container within `BiologicalSystem` for all entities.

1.  Create `res://_Brain/Systems/Biology/CoreStats.cs`:

```csharp
// _Brain/Systems/Biology/CoreStats.cs
using System;
using Godot; // For Vector2 if needed

namespace Sigilborne.Systems.Biology
{
    /// <summary>
    /// Stores the core statistics for any entity that has biological or combat capabilities.
    /// This struct serves as the comprehensive stat container for entities.
    /// (TDD 03.2 - Expanding on BioState concept)
    /// </summary>
    public struct CoreStats
    {
        // --- Core Survival/Biological Stats (from BioState) ---
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
        public float BodyTemp;     // Celsius
        public float NormalBodyTemp; // Reference normal temp for this entity

        // --- Combat/Magic-related Stats (new or consolidated) ---
        public float BaseDamage;       // Base physical damage for attacks
        public float AttackSpeed;      // How fast an entity attacks
        public float Armor;            // Physical damage reduction
        public float MagicResistance;  // Magical damage reduction
        public float MoveSpeed;        // Base movement speed
        public float SprintMultiplier; // Multiplier for sprint speed
        public float CastSpeed;        // Multiplier for spell cast time (lower = faster)
        public float Stability;        // Current Chakra Stability (0-100) (GDD B03.6)
        public float MaxStability;     // Max Chakra Stability (e.g., 100)
        public float StabilityRegenRate; // How fast stability recovers
        public float ChakraRegenRate;  // How fast chakra recovers
        public float StaminaRegenRate; // How fast stamina recovers

        public CoreStats(float maxHealth, float maxStamina, float maxChakra, float baseDamage = 10f, float attackSpeed = 1.0f,
                         float armor = 0f, float magicResistance = 0f, float moveSpeed = 150f, float sprintMultiplier = 1.5f,
                         float castSpeed = 1.0f, float maxStability = 100f, float stabilityRegenRate = 5f,
                         float chakraRegenRate = 2f, float staminaRegenRate = 10f, float normalBodyTemp = 37.0f)
        {
            MaxHealth = maxHealth;
            Health = maxHealth;
            MaxStamina = maxStamina;
            Stamina = maxStamina;
            MaxChakra = maxChakra;
            Chakra = maxChakra;
            MaxHunger = 100f;
            Hunger = MaxHunger;
            MaxThirst = 100f;
            Thirst = MaxThirst;
            NormalBodyTemp = normalBodyTemp;
            BodyTemp = normalBodyTemp;

            BaseDamage = baseDamage;
            AttackSpeed = attackSpeed;
            Armor = armor;
            MagicResistance = magicResistance;
            MoveSpeed = moveSpeed;
            SprintMultiplier = sprintMultiplier;
            CastSpeed = castSpeed;
            MaxStability = maxStability;
            Stability = MaxStability; // Start full
            StabilityRegenRate = stabilityRegenRate;
            ChakraRegenRate = chakraRegenRate;
            StaminaRegenRate = staminaRegenRate;
        }

        public override string ToString()
        {
            return $"HP: {Health:F0}/{MaxHealth:F0}, Sta: {Stamina:F0}/{MaxStamina:F0}, Cha: {Chakra:F0}/{MaxChakra:F0}, Stab: {Stability:F0}/{MaxStability:F0}, Hunger: {Hunger:F0}, Thirst: {Thirst:F0}, Temp: {BodyTemp:F1}°C, Spd: {MoveSpeed:F0}";
        }
    }
}
```

### 3. Refactoring `BiologicalSystem.cs` to use `CoreStats`

The `BiologicalSystem` will now store `CoreStats` for each entity instead of `BioState`. This means `PlayerStatSystem` will also directly interact with `CoreStats`.

1.  Open `res://_Brain/Systems/Biology/BiologicalSystem.cs`.
2.  Replace `private Dictionary<EntityID, BioState> _bioStates = new Dictionary<EntityID, BioState>();` with:
    `private Dictionary<EntityID, CoreStats> _entityCoreStats = new Dictionary<EntityID, CoreStats>();`
3.  Modify `OnEntitySpawned` to add `CoreStats` instances.
4.  Modify `OnEntityDespawned` to remove `CoreStats` instances.
5.  Update `ProcessBioTick` to operate on `CoreStats` and include regeneration logic for Health, Stamina, Chakra, and Stability.
6.  Update `GetBioStateRef` to `GetCoreStatsRef`.
7.  Update `PlayerBioStateChangedEvent` to `PlayerCoreStatsChangedEvent` and pass `CoreStats`.

```csharp
// _Brain/Systems/Biology/BiologicalSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Physics;
using Sigilborne.Systems.Movement;
using Sigilborne.Systems.Magic; // For GlyphConcept if environmental resonance affects regen

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private TransformSystem _transformSystem;
        private MovementSystem _movementSystem;

        // Dictionary to store CoreStats for all entities that have one.
        private Dictionary<EntityID, CoreStats> _entityCoreStats = new Dictionary<EntityID, CoreStats>(); // Changed from _bioStates

        private const float BIO_TICK_RATE = 1.0f; // 1.0f = once per real second (1Hz)
        private float _bioTickTimer;

        public BiologicalSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, MovementSystem movementSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _movementSystem = movementSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("BiologicalSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player || type == EntityType.NPC || type == EntityType.Animal)
            {
                // Initialize CoreStats (TDD 03.2)
                // For player, PlayerStatSystem will further initialize/override this.
                // For NPCs/Animals, load from definitionID (e.g., "goblin_grunt" has specific stats).
                // For now, let's give generic stats.
                _entityCoreStats.Add(id, new CoreStats(100f, 75f, 50f)); // HP, Stamina, Chakra
                GD.Print($"BiologicalSystem: Added CoreStats for {type} entity {id}.");
            }
        }

        private void OnEntityDespawned(EntityID id)
        {
            _entityCoreStats.Remove(id); // Changed from _bioStates
            GD.Print($"BiologicalSystem: Removed CoreStats for {id}.");
        }

        public void Tick(double delta)
        {
            _bioTickTimer += (float)delta;
            if (_bioTickTimer >= BIO_TICK_RATE)
            {
                ProcessBioTick();
                _bioTickTimer = 0;
            }
        }

        private void ProcessBioTick()
        {
            // GD.Print($"BiologicalSystem: Processing Bio-Tick (Game Time: {GameManager.Instance.Time.CurrentGameTime:F1})");

            foreach (var kvp in _entityCoreStats) // Changed from _bioStates
            {
                EntityID id = kvp.Key;
                ref CoreStats coreStats = ref _entityCoreStats.GetValueRef(id); // Get mutable ref

                if (!_entityManager.IsValid(id)) continue;

                float activityMultiplier = GetActivityLevel(id);
                float weatherMultiplier = 1.0f;

                // --- Hunger and Thirst Decay ---
                coreStats.Hunger -= 1.0f * activityMultiplier * weatherMultiplier;
                coreStats.Hunger = Mathf.Max(0, coreStats.Hunger);
                
                coreStats.Thirst -= 1.5f * activityMultiplier * weatherMultiplier;
                coreStats.Thirst = Mathf.Max(0, coreStats.Thirst);

                // --- Regeneration (Health, Stamina, Chakra, Stability) (GDD B03.4, B03.6) ---
                // Health Regen (only if not starving/dehydrated, and not in combat)
                if (coreStats.Hunger > 10 && coreStats.Thirst > 10 && coreStats.Health < coreStats.MaxHealth)
                {
                    coreStats.Health += coreStats.HealthRegenRate * BIO_TICK_RATE;
                    coreStats.Health = Mathf.Min(coreStats.MaxHealth, coreStats.Health);
                }

                // Stamina Regen
                if (coreStats.Stamina < coreStats.MaxStamina)
                {
                    coreStats.Stamina += coreStats.StaminaRegenRate * BIO_TICK_RATE;
                    coreStats.Stamina = Mathf.Min(coreStats.MaxStamina, coreStats.Stamina);
                }

                // Chakra Regen
                if (coreStats.Chakra < coreStats.MaxChakra)
                {
                    coreStats.Chakra += coreStats.ChakraRegenRate * BIO_TICK_RATE;
                    coreStats.Chakra = Mathf.Min(coreStats.MaxChakra, coreStats.Chakra);
                }

                // Stability Regen (GDD B03.6)
                if (coreStats.Stability < coreStats.MaxStability)
                {
                    coreStats.Stability += coreStats.StabilityRegenRate * BIO_TICK_RATE;
                    coreStats.Stability = Mathf.Min(coreStats.MaxStability, coreStats.Stability);
                }

                // --- Apply consequences of low stats (GDD B14.4) ---
                if (coreStats.Hunger <= 0) { /* GD.Print($"BiologicalSystem: {id} is starving!"); */ }
                if (coreStats.Thirst <= 0) { /* GD.Print($"BiologicalSystem: {id} is dehydrated!"); */ }

                // Publish updates for the Body (e.g., UI for player)
                if (id == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerCoreStatsChangedEvent { PlayerID = id, NewCoreStats = coreStats }); // Changed event
                }
            }
        }

        public float GetActivityLevel(EntityID id) { /* ... */ return 1.0f; }

        /// <summary>
        /// Provides a mutable reference to an entity's CoreStats.
        /// Other systems (e.g., PlayerStatSystem, CombatSystem) can modify these.
        /// </summary>
        public ref CoreStats GetCoreStatsRef(EntityID id) // Changed from GetBioStateRef
        {
            if (!_entityManager.IsValid(id) || !_entityCoreStats.ContainsKey(id)) // Changed from _bioStates
            {
                throw new InvalidOperationException($"Entity {id} is invalid or does not have CoreStats.");
            }
            return ref _entityCoreStats.GetValueRef(id); // Changed from _bioStates
        }

        // --- Helper Events for Body Sync ---
        public struct PlayerCoreStatsChangedEvent { public EntityID PlayerID; public CoreStats NewCoreStats; } // Changed event
    }
}
```

### 4. Refactoring `PlayerStatSystem.cs` to use `BiologicalSystem`'s `CoreStats`

The `PlayerStatSystem` no longer needs its own `PlayerStats` struct. It will directly access and modify the player's `CoreStats` via `BiologicalSystem`.

1.  Open `res://_Brain/Systems/Biology/PlayerStatSystem.cs`.
2.  Remove the `PlayerStats` struct definition.
3.  Modify the `PlayerStatSystem` class to interact with `BiologicalSystem`'s `CoreStats` directly.
4.  Remove `OnEntitySpawned` logic, as `BiologicalSystem` handles initial `CoreStats` creation.
5.  Update `GetPlayerStats`, `TakeDamage`, `TakeChakra` to use `_biologicalSystem.GetCoreStatsRef()`.
6.  Add `TakeStamina` method.
7.  Update event structs to pass individual float values, as `CoreStats` is too large for frequent full struct passing.

```csharp
// _Brain/Systems/Biology/PlayerStatSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;

namespace Sigilborne.Systems.Biology
{
    // Removed PlayerStats struct definition.

    public class PlayerStatSystem
    {
        private EventBus _eventBus;
        private EntityManager _entityManager;
        private BiologicalSystem _biologicalSystem;

        private EntityID _playerEntityID;

        public PlayerStatSystem(EventBus eventBus, EntityManager entityManager, BiologicalSystem biologicalSystem)
        {
            _eventBus = eventBus;
            _entityManager = entityManager;
            _biologicalSystem = biologicalSystem;

            _playerEntityID = _entityManager.GetPlayerEntityID();

            GD.Print("PlayerStatSystem: Initialized.");
            
            // Publish initial player stats from BiologicalSystem's CoreStats
            ref CoreStats playerCoreStats = ref _biologicalSystem.GetCoreStatsRef(_playerEntityID);
            _eventBus.Publish(new PlayerHealthChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Health, MaxValue = playerCoreStats.MaxHealth });
            _eventBus.Publish(new PlayerChakraChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Chakra, MaxValue = playerCoreStats.MaxChakra });
            _eventBus.Publish(new PlayerStaminaChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Stamina, MaxValue = playerCoreStats.MaxStamina });
            _eventBus.Publish(new PlayerStabilityChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Stability, MaxValue = playerCoreStats.MaxStability }); // New event
        }

        // Removed GetPlayerStats(), as systems should access CoreStats via BiologicalSystem directly or use events.

        public void TakeDamage(float amount)
        {
            ref CoreStats playerCoreStats = ref _biologicalSystem.GetCoreStatsRef(_playerEntityID);
            if (playerCoreStats.Health <= 0) return;

            playerCoreStats.Health -= amount;
            if (playerCoreStats.Health < 0) playerCoreStats.Health = 0;

            GD.Print($"PlayerStatSystem: Player {_playerEntityID} took {amount} damage. New Health: {playerCoreStats.Health}");
            _eventBus.Publish(new PlayerHealthChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Health, MaxValue = playerCoreStats.MaxHealth });

            if (playerCoreStats.Health == 0)
            {
                GD.Print($"PlayerStatSystem: Player {_playerEntityID} has died!");
                _eventBus.Publish(new PlayerDiedEvent { PlayerID = _playerEntityID });
            }
        }

        public void TakeChakra(float amount)
        {
            ref CoreStats playerCoreStats = ref _biologicalSystem.GetCoreStatsRef(_playerEntityID);
            if (amount < 0) return;

            playerCoreStats.Chakra -= amount;
            if (playerCoreStats.Chakra < 0) playerCoreStats.Chakra = 0;

            GD.Print($"PlayerStatSystem: Player {_playerEntityID} used {amount} chakra. New Chakra: {playerCoreStats.Chakra}");
            _eventBus.Publish(new PlayerChakraChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Chakra, MaxValue = playerCoreStats.MaxChakra });
        }

        /// <summary>
        /// Authoritatively deducts stamina from the player.
        /// </summary>
        public void TakeStamina(float amount) // New method
        {
            ref CoreStats playerCoreStats = ref _biologicalSystem.GetCoreStatsRef(_playerEntityID);
            if (amount < 0) return;

            playerCoreStats.Stamina -= amount;
            if (playerCoreStats.Stamina < 0) playerCoreStats.Stamina = 0;

            GD.Print($"PlayerStatSystem: Player {_playerEntityID} used {amount} stamina. New Stamina: {playerCoreStats.Stamina}");
            _eventBus.Publish(new PlayerStaminaChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Stamina, MaxValue = playerCoreStats.MaxStamina });
        }

        /// <summary>
        /// Authoritatively deducts stability from the player.
        /// (GDD B03.6)
        /// </summary>
        public void TakeStability(float amount) // New method
        {
            ref CoreStats playerCoreStats = ref _biologicalSystem.GetCoreStatsRef(_playerEntityID);
            if (amount < 0) return;

            playerCoreStats.Stability -= amount;
            if (playerCoreStats.Stability < 0) playerCoreStats.Stability = 0; // Stability can potentially go negative for chaos residue effects later.

            GD.Print($"PlayerStatSystem: Player {_playerEntityID} lost {amount} stability. New Stability: {playerCoreStats.Stability}");
            _eventBus.Publish(new PlayerStabilityChangedEvent { PlayerID = _playerEntityID, NewValue = playerCoreStats.Stability, MaxValue = playerCoreStats.MaxStability });
        }


        // --- Helper Events for Body Sync ---
        // Changed event structs to pass NewValue and MaxValue for generic display.
        public struct PlayerHealthChangedEvent { public EntityID PlayerID; public float NewValue; public float MaxValue; }
        public struct PlayerChakraChangedEvent { public EntityID PlayerID; public float NewValue; public float MaxValue; }
        public struct PlayerStaminaChangedEvent { public EntityID PlayerID; public float NewValue; public float MaxValue; }
        public struct PlayerStabilityChangedEvent { public EntityID PlayerID; public float NewValue; public float MaxValue; } // New event
        public struct PlayerDiedEvent { public EntityID PlayerID; }
    }
}
```

#### 4.1. Update `GameManager` to Pass `BiologicalSystem` to `PlayerStatSystem`

Open `res://_Brain/Core/GameManager.cs` and modify the `PlayerStatSystem` initialization in `InitializeSystems()`:

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        BiologicalSystem = new BiologicalSystem(Entities, Events, Transforms, Movement);
        GD.Print("  - BiologicalSystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem); // Pass BiologicalSystem
        GD.Print("  - PlayerStatSystem initialized.");
// ...
```

#### 4.2. Update `EventBus.cs` for New PlayerStatSystem Events

Open `res://_Brain/Core/EventBus.cs` and modify/add delegates for `PlayerStatSystem`'s new event signatures.

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

        // Player Stat Events (TDD 01.4) - updated signatures
        public event Action<EntityID, float, float> OnPlayerHealthChanged;
        public event Action<EntityID> OnPlayerDied;
        public event Action<EntityID, float, float> OnPlayerChakraChanged;
        public event Action<EntityID, float, float> OnPlayerStaminaChanged;
        public event Action<EntityID, float, float> OnPlayerStabilityChanged; // New event

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is PlayerStatSystem.PlayerHealthChangedEvent healthEvent) // Updated signature
            {
                OnPlayerHealthChanged?.Invoke(healthEvent.PlayerID, healthEvent.NewValue, healthEvent.MaxValue);
            }
            else if (eventData is PlayerStatSystem.PlayerChakraChangedEvent chakraEvent) // Updated signature
            {
                OnPlayerChakraChanged?.Invoke(chakraEvent.PlayerID, chakraEvent.NewValue, chakraEvent.MaxValue);
            }
            else if (eventData is PlayerStatSystem.PlayerStaminaChangedEvent staminaEvent) // Updated signature
            {
                OnPlayerStaminaChanged?.Invoke(staminaEvent.PlayerID, staminaEvent.NewValue, staminaEvent.MaxValue);
            }
            else if (eventData is PlayerStatSystem.PlayerStabilityChangedEvent stabilityEvent) // New condition
            {
                OnPlayerStabilityChanged?.Invoke(stabilityEvent.PlayerID, stabilityEvent.NewValue, stabilityEvent.MaxValue);
            }
            else if (eventData is BiologicalSystem.PlayerCoreStatsChangedEvent bioStateEvent) // Updated to CoreStats
            {
                // This event is now for the *full* CoreStats update, used for UI that displays all.
                OnPlayerCoreStatsChanged?.Invoke(bioStateEvent.PlayerID, bioStateEvent.NewCoreStats);
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

### 5. Refactoring `MovementSystem.cs` for `CoreStats`

The `MovementSystem` needs to get `MoveSpeed` and `SprintMultiplier` from `CoreStats`.

1.  Open `res://_Brain/Systems/Movement/MovementSystem.cs`.
2.  Remove the `MovementParametersComponent` struct definition.
3.  Remove `private Dictionary<EntityID, MovementParametersComponent> _movementParams`.
4.  Modify `OnEntitySpawned` to remove `_movementParams.Add()`.
5.  Update `Tick` to retrieve `MoveSpeed` and `SprintMultiplier` directly from `BiologicalSystem.GetCoreStatsRef(id)`.

```csharp
// _Brain/Systems/Movement/MovementSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Input;
using Sigilborne.Systems.Biology; // Add this using directive

namespace Sigilborne.Systems.Movement
{
    // ... (VelocityComponent struct) ...
    // Removed MovementParametersComponent struct definition.

    public class MovementSystem
    {
        private EntityManager _entityManager;
        private InputSystem _inputSystem;
        private EventBus _eventBus;
        private TransformSystem _transformSystem;
        private BiologicalSystem _biologicalSystem; // New: Reference to BiologicalSystem

        private Dictionary<EntityID, VelocityComponent> _velocities = new Dictionary<EntityID, VelocityComponent>();
        // Removed _movementParams dictionary.
        private Dictionary<EntityID, PlayerMovementStateComponent> _playerMovementStates = new Dictionary<EntityID, PlayerMovementStateComponent>();

        public MovementSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus, TransformSystem transformSystem, BiologicalSystem biologicalSystem) // Add BiologicalSystem
        {
            _entityManager = entityManager;
            _inputSystem = inputSystem;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _biologicalSystem = biologicalSystem; // Store BiologicalSystem reference

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("MovementSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player)
            {
                _velocities.Add(id, new VelocityComponent(Vector2.Zero));
                // MovementParametersComponent is no longer added here.
                _playerMovementStates.Add(id, new PlayerMovementStateComponent());
                GD.Print($"MovementSystem: Added movement components for Player {id}");
            }
        }

        private void OnEntityDespawned(EntityID id)
        {
            _velocities.Remove(id);
            // Removed _movementParams.Remove(id);
            _playerMovementStates.Remove(id);
            GD.Print($"MovementSystem: Removed movement components for {id}");
        }

        public void Tick(double delta)
        {
            PlayerInputFrame currentInput = _inputSystem.GetLatestInput();

            foreach (var kvp in _velocities)
            {
                EntityID id = kvp.Key;

                // Get CoreStats for movement parameters (TDD 03.2)
                if (!_entityManager.IsValid(id) || !_biologicalSystem.TryGetCoreStats(id, out CoreStats coreStats)) // Use TryGetCoreStats for safety
                {
                    continue;
                }

                ref VelocityComponent currentVelocity = ref _velocities.GetValueRef(id);
                Vector2 targetMoveVector = currentInput.MoveVector;
                float currentSpeed = coreStats.MoveSpeed; // Get from CoreStats
                float friction = 0.8f; // Default friction, could be part of CoreStats or external
                float acceleration = 10f; // Default acceleration, could be part of CoreStats or external
                float targetRotationDegrees = _transformSystem.GetTransformRef(id).RotationDegrees;

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
                    currentSpeed *= coreStats.SprintMultiplier; // Get from CoreStats
                }

                Vector2 desiredVelocity = targetMoveVector * currentSpeed;

                currentVelocity.Velocity = currentVelocity.Velocity.Lerp(desiredVelocity, acceleration * (float)delta);

                if (!(_playerMovementStates.TryGetValue(id, out var state2) && state2.IsShiftSliding) && targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() > 0.1f)
                {
                    currentVelocity.Velocity = currentVelocity.Velocity.Lerp(Vector2.Zero, friction * (float)delta);
                }
                else if (!(_playerMovementStates.TryGetValue(id, out var state3) && state3.IsShiftSliding) && targetMoveVector == Vector2.Zero && currentVelocity.Velocity.LengthSquared() <= 0.1f)
                {
                    currentVelocity.Velocity = Vector2.Zero;
                }

                if (targetMoveVector.LengthSquared() > 0.1f)
                {
                    targetRotationDegrees = targetMoveVector.Angle() * (180f / Mathf.Pi);
                }

                _eventBus.Publish(new EntityManager.EntityVelocityUpdateEvent { ID = id, TargetVelocity = currentVelocity.Velocity, TargetRotationDegrees = targetRotationDegrees });
            }
        }

        public Vector2 GetVelocity(EntityID id) { /* ... */ return Vector2.Zero; }
    }
}
```

#### 5.1. Add `TryGetCoreStats` to `BiologicalSystem.cs`

The `MovementSystem` needs a safe way to get `CoreStats`.

Open `res://_Brain/Systems/Biology/BiologicalSystem.cs` and add this method:

```csharp
// _Brain/Systems/Biology/BiologicalSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Physics;
using Sigilborne.Systems.Movement;
using Sigilborne.Systems.Magic;

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        // ... (existing fields and constructor) ...

        public ref CoreStats GetCoreStatsRef(EntityID id) { /* ... */ return ref _entityCoreStats.GetValueRef(id); }

        /// <summary>
        /// Safely attempts to get a copy of an entity's CoreStats.
        /// </summary>
        public bool TryGetCoreStats(EntityID id, out CoreStats coreStats)
        {
            if (_entityManager.IsValid(id) && _entityCoreStats.TryGetValue(id, out coreStats))
            {
                return true;
            }
            coreStats = default;
            return false;
        }

        // ... (Helper Events) ...
    }
}
```

#### 5.2. Update `GameManager` to Pass `BiologicalSystem` to `MovementSystem`

Open `res://_Brain/Core/GameManager.cs` and modify the `MovementSystem` initialization in `InitializeSystems()`:

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        Movement = new MovementSystem(Entities, Input, Events, Transforms, BiologicalSystem); // Pass BiologicalSystem
        GD.Print("  - MovementSystem initialized.");
// ...
```

### 6. Refactor `MagicSystem.cs` for `CoreStats`

The `MagicSystem` needs to deduct chakra and stability, and potentially check `CastSpeed` from `CoreStats`.

1.  Open `res://_Brain/Systems/Magic/MagicSystem.cs`.
2.  Modify `MagicSystem` constructor to pass `BiologicalSystem`.
3.  Update `ProcessGlyphInput` and `ResolveCombo` to use `BiologicalSystem.GetCoreStatsRef()`.

```csharp
// _Brain/Systems/Magic/MagicSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Input;
using Sigilborne.Systems.Biology; // Add this using directive

namespace Sigilborne.Systems.Magic
{
    public class MagicSystem
    {
        // ... (existing fields) ...
        private BiologicalSystem _biologicalSystem; // New: Reference to BiologicalSystem

        public MagicSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus,
                           PlayerHotbarSystem playerHotbar, PlayerGlyphKnowledgeSystem playerGlyphKnowledge,
                           WorldGlyphMap worldGlyphMap, GameManager gameManager, ComboResolver comboResolver,
                           CastingSystem castingSystem, BiologicalSystem biologicalSystem) // Add BiologicalSystem
        {
            _entityManager = entityManager;
            _inputSystem = inputSystem;
            _eventBus = eventBus;
            _playerHotbar = playerHotbar;
            _playerGlyphKnowledge = playerGlyphKnowledge;
            _worldGlyphMap = worldGlyphMap;
            _gameManager = gameManager;

            _playerEntityID = _entityManager.GetPlayerEntityID();

            _glyphInputBuffer = new GlyphInputBuffer(MAX_GLYPH_BUFFER_SIZE);
            _comboResolver = comboResolver;
            _castingSystem = castingSystem;
            _biologicalSystem = biologicalSystem; // Store BiologicalSystem reference
            GD.Print("MagicSystem: Initialized.");
        }

        // ... (Tick, ProcessGlyphInput methods) ...

        private void ResolveCombo()
        {
            ReadOnlySpan<GlyphInputFrame> recentInputsSpan = _glyphInputBuffer.GetRecentUnconsumed(_gameManager.Time.CurrentGameTime - MAX_COMBO_DELAY);
            
            if (recentInputsSpan.IsEmpty) return;

            // Check if player is currently in a state that prevents new casts (TDD 02.4)
            if (_castingSystem.GetPlayerCastingState().CurrentState != CastState.Idle)
            {
                GD.Print($"MagicSystem: Player is not Idle ({_castingSystem.GetPlayerCastingState().CurrentState}), cannot initiate new cast.");
                return; // Cannot initiate a new cast if not idle
            }

            GD.Print($"MagicSystem: Attempting to resolve combo with {recentInputsSpan.Length} recent inputs.");
            
            SpellDefinition resolvedSpell = _comboResolver.ResolveCombo(recentInputsSpan);

            if (resolvedSpell != null)
            {
                // Get player's current CoreStats for resource checks (Chakra, Stability, CastSpeed)
                ref CoreStats playerCoreStats = ref _biologicalSystem.GetCoreStatsRef(_playerEntityID);

                // Check Chakra cost
                if (playerCoreStats.Chakra < resolvedSpell.ChakraCost)
                {
                    GD.Print($"CastingSystem: Insufficient Chakra to cast '{resolvedSpell.ID}'. (Need: {resolvedSpell.ChakraCost}, Have: {playerCoreStats.Chakra})");
                    _eventBus.Publish(new CastingSystem.CastFailedEvent { PlayerID = _playerEntityID, Reason = "Insufficient Chakra" });
                    return;
                }
                // Check Stability cost (GDD B03.2)
                if (playerCoreStats.Stability < resolvedSpell.StabilityCost)
                {
                    GD.Print($"CastingSystem: Insufficient Stability to cast '{resolvedSpell.ID}'. (Need: {resolvedSpell.StabilityCost}, Have: {playerCoreStats.Stability})");
                    _eventBus.Publish(new CastingSystem.CastFailedEvent { PlayerID = _playerEntityID, Reason = "Insufficient Stability" });
                    return;
                }

                // If resources are sufficient, initiate the cast via CastingSystem.
                // CastingSystem will deduct resources and manage state.
                if (_castingSystem.InitiateCast(resolvedSpell)) // InitiateCast will now deduct resources
                {
                    GD.Print($"MagicSystem: Resolved and initiated cast for known spell: '{resolvedSpell.ID}'!");
                    _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = false, ResultText = $"Cast {resolvedSpell.ID}!", ConceptSequence = resolvedSpell.Sequence });
                    _glyphInputBuffer.MarkConsumed(recentInputsSpan.ToArray());
                }
                else
                {
                    GD.Print($"MagicSystem: Failed to initiate cast for '{resolvedSpell.ID}' (e.g., CastingSystem rejected).");
                }
                return;
            }
            // ... (existing placeholder resolution logic) ...
        }
    }
}
```

#### 6.1. Update `CastingSystem.cs` to use `BiologicalSystem`'s `CoreStats`

The `CastingSystem` needs to deduct chakra and stability.

1.  Open `res://_Brain/Systems/Magic/CastingSystem.cs`.
2.  Modify `CastingSystem` constructor to pass `BiologicalSystem`.
3.  Remove direct `_playerStatSystem` calls for chakra/stability deduction. Delegate to `PlayerStatSystem` which in turn uses `BiologicalSystem`.

```csharp
// _Brain/Systems/Magic/CastingSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic.Components;

namespace Sigilborne.Systems.Magic
{
    public class CastingSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private PlayerStatSystem _playerStatSystem; // Still needed for damage/chakra/stamina deduction methods
        private TransformSystem _transformSystem;
        private BiologicalSystem _biologicalSystem; // New: Reference to BiologicalSystem

        private EntityID _playerEntityID;
        private PlayerCastingState _playerCastingState;

        private const float DEFAULT_RECOVERY_TIME = 0.3f;

        public CastingSystem(EntityManager entityManager, EventBus eventBus, PlayerStatSystem playerStatSystem, TransformSystem transformSystem, BiologicalSystem biologicalSystem) // Add BiologicalSystem
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _playerStatSystem = playerStatSystem;
            _transformSystem = transformSystem;
            _biologicalSystem = biologicalSystem; // Store BiologicalSystem reference

            _playerEntityID = _entityManager.GetPlayerEntityID();
            _playerCastingState = new PlayerCastingState(_playerEntityID);

            GD.Print("CastingSystem: Initialized.");
        }

        public void Tick(double delta) { /* ... */ }

        public bool InitiateCast(SpellDefinition spell)
        {
            if (_playerCastingState.CurrentState != CastState.Idle)
            {
                GD.Print($"CastingSystem: Cannot cast spell '{spell.ID}', not in Idle state ({_playerCastingState.CurrentState}).");
                return false;
            }

            // Resource checks are now done in MagicSystem before calling InitiateCast.
            // So, if we reach here, resources are sufficient.

            // Deduct resources (TDD 02.4: Mana deducted)
            _playerStatSystem.TakeChakra(spell.ChakraCost); // Deduct chakra
            _playerStatSystem.TakeStability(spell.StabilityCost); // Deduct stability (GDD B03.2)

            // Adjust CastTime based on player's CastSpeed (from CoreStats)
            ref CoreStats playerCoreStats = ref _biologicalSystem.GetCoreStatsRef(_playerEntityID);
            double finalCastTime = spell.CastTime / playerCoreStats.CastSpeed; // Faster CastSpeed reduces cast time

            _playerCastingState.CurrentSpell = spell;
            TransitionToState(CastState.CastStart, finalCastTime); // Use finalCastTime

            GD.Print($"CastingSystem: Initiated cast for '{spell.ID}'. Chakra cost: {spell.ChakraCost}, Stability cost: {spell.StabilityCost}. Final Cast Time: {finalCastTime:F2}s.");
            _eventBus.Publish(new CastStateChangedEvent { PlayerID = _playerEntityID, NewState = CastState.CastStart, SpellID = spell.ID });

            return true;
        }

        public PlayerCastingState GetPlayerCastingState() { return _playerCastingState; }

        private void TransitionToState(CastState newState, double timerDuration = 0) { /* ... */ }
        private void ExecuteSpellEffect(SpellDefinition spell) { /* ... */ }
    }
}
```

#### 6.2. Update `GameManager` to Pass `BiologicalSystem` to `CastingSystem`

Open `res://_Brain/Core/GameManager.cs` and modify the `CastingSystem` initialization in `InitializeSystems()`:

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        // Initialize CastingSystem, passing BiologicalSystem
        Casting = new CastingSystem(Entities, Events, PlayerStats, Transforms, BiologicalSystem); // Pass BiologicalSystem
        GD.Print("  - CastingSystem initialized.");

        // Initialize MagicSystem, passing CastingSystem and BiologicalSystem
        Magic = new MagicSystem(Entities, Input, Events, PlayerHotbar, PlayerGlyphKnowledge, GlyphMap, this, ComboResolver, Casting, BiologicalSystem); // Pass BiologicalSystem
        GD.Print("  - MagicSystem initialized.");
// ...
```

### 7. Update Debug Console

Open `res://_Brain/Utils/DebugCommandSystem.cs` and update the `damage` command to use `PlayerStats.TakeDamage` (which is now updated). Also add commands for `chakra` and `stability`.

```csharp
// _Brain/Utils/DebugCommandSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;

namespace Sigilborne.Utils
{
    // ... (DebugCommand struct) ...

    public class DebugCommandSystem
    {
        // ... (fields and constructor) ...

        private void RegisterDefaultCommands()
        {
            // ... (existing commands) ...

            RegisterCommand("damage", "Damages the player: /damage [amount]", (args) =>
            {
                if (args.Length != 1 || !float.TryParse(args[0], out float amount))
                {
                    GD.PrintErr("Usage: /damage [amount]");
                    return;
                }
                _gameManager.PlayerStats.TakeDamage(amount);
                GD.Print($"Player took {amount} damage.");
            });

            RegisterCommand("chakra", "Adds/removes chakra: /chakra [amount]", (args) =>
            {
                if (args.Length != 1 || !float.TryParse(args[0], out float amount))
                {
                    GD.PrintErr("Usage: /chakra [amount]");
                    return;
                }
                // For adding chakra, we'll need a different method or a direct CoreStats access.
                // For now, let's just make it take chakra (negative amount means adding).
                _gameManager.PlayerStats.TakeChakra(-amount); // Negative amount to add
                GD.Print($"Player chakra adjusted by {amount}.");
            });

            RegisterCommand("stability", "Adds/removes stability: /stability [amount]", (args) =>
            {
                if (args.Length != 1 || !float.TryParse(args[0], out float amount))
                {
                    GD.PrintErr("Usage: /stability [amount]");
                    return;
                }
                _gameManager.PlayerStats.TakeStability(-amount); // Negative amount to add
                GD.Print($"Player stability adjusted by {amount}.");
            });

            RegisterCommand("spawn", "Spawns an entity: /spawn [type] [def_id] [x] [y]", (args) =>
            {
                // ... (existing spawn command) ...
            });
        }
    }
}
```

### 8. Update GDScript HUD

Open `res://_Body/Scripts/UI/HUDController.gd` and add a `stability_label` and connect to `OnPlayerStabilityChanged`.

```gdscript
# _Body/Scripts/UI/HUDController.gd
class_name HUDController extends CanvasLayer

@onready var health_label: Label = $MainHBox/VBoxStats/HealthLabel
@onready var chakra_label: Label = $MainHBox/VBoxStats/ChakraLabel
@onready var stamina_label: Label = $MainHBox/VBoxStats/StaminaLabel
@onready var stability_label: Label = $MainHBox/VBoxStats/StabilityLabel # New
@onready var hunger_label: Label = $MainHBox/VBoxNeeds/HungerLabel
@onready var thirst_label: Label = $MainHBox/VBoxNeeds/ThirstLabel

var player_entity_id: int = -1

func _ready():
    GD.print("HUDController: Initialized. Connecting to C# PlayerStatSystem & BiologicalSystem events.")
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        GameManager.Instance.Events.OnPlayerHealthChanged.connect(Callable(self, "_on_player_health_changed"))
        GameManager.Instance.Events.OnPlayerChakraChanged.connect(Callable(self, "_on_player_chakra_changed"))
        GameManager.Instance.Events.OnPlayerStaminaChanged.connect(Callable(self, "_on_player_stamina_changed"))
        GameManager.Instance.Events.OnPlayerStabilityChanged.connect(Callable(self, "_on_player_stability_changed")) # New
        GameManager.Instance.Events.OnPlayerDied.connect(Callable(self, "_on_player_died"))
        GameManager.Instance.Events.OnPlayerCoreStatsChanged.connect(Callable(self, "_on_player_core_stats_changed")) # Updated from BioStateChanged
        GameManager.Instance.Events.OnEntitySpawned.connect(Callable(self, "_on_entity_spawned"))
        GD.print("HUDController: Successfully connected to C# PlayerStatSystem & BiologicalSystem events.")
    else:
        push_error("HUDController: GameManager or EventBus not ready! Cannot connect C# events.")

func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    if type == 0: # EntityType.Player has value 0
        player_entity_id = id
        GD.print("HUDController: Detected player entity with ID: %s" % player_entity_id)

func _on_player_health_changed(id: int, new_value: float, max_value: float) -> void:
    if id == player_entity_id:
        health_label.text = "HP: %s/%s" % [int(new_value), int(max_value)]

func _on_player_chakra_changed(id: int, new_value: float, max_value: float) -> void:
    if id == player_entity_id:
        chakra_label.text = "CHA: %s/%s" % [int(new_value), int(max_value)]

func _on_player_stamina_changed(id: int, new_value: float, max_value: float) -> void:
    if id == player_entity_id:
        stamina_label.text = "STA: %s/%s" % [int(new_value), int(max_value)]

func _on_player_stability_changed(id: int, new_value: float, max_value: float) -> void: # New
    if id == player_entity_id:
        stability_label.text = "STAB: %s/%s" % [int(new_value), int(max_value)]

func _on_player_core_stats_changed(id: int, new_core_stats: Variant) -> void: # Updated to CoreStats
    if id == player_entity_id:
        hunger_label.text = "HUN: %s" % int(new_core_stats.Hunger)
        thirst_label.text = "THI: %s" % int(new_core_stats.Thirst)

func _on_player_died(id: int) -> void:
    if id == player_entity_id:
        health_label.text = "HP: DEAD"
        chakra_label.text = "CHA: DEAD"
        stamina_label.text = "STA: DEAD"
        stability_label.text = "STAB: DEAD"
        hunger_label.text = "HUN: DEAD"
        thirst_label.text = "THI: DEAD"
        GD.print("HUDController: Player %s has died visually!" % id)
```

Also, update `res://_Body/Scenes/UI/HUD.tscn` to add a `StabilityLabel` to `VBoxStats`.

### 9. Testing the Casting State Machine with CoreStats

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the new Stability UI element. It should start at 100/100.
5.  **Test 1 (Known Spell - `testSymbol` -> `testSymbol2`)**:
    *   Quickly press `0` then `1`.
    *   Observe the Chakra and Stability values decreasing on the HUD as the spell casts.
6.  **Test 2 (Insufficient Chakra/Stability)**:
    *   Use the debug console `/chakra -50` (to reduce chakra) or `/stability -100` (to reduce stability).
    *   Attempt to cast `Test_BloomConsume`.
    *   You should see messages about insufficient chakra/stability in the console, and the spell should not initiate.
7.  **Test 3 (CastSpeed effect)**:
    *   The player's default `CastSpeed` is 1.0.
    *   If you could modify `PlayerStats.CastSpeed` (e.g., via a debug command: `/castspeed 2.0`), the `CastStart` duration would halve.

This comprehensive test confirms that `CastingSystem` correctly manages the player's casting flow, interacts with `PlayerStatSystem` to deduct chakra and stability from `CoreStats`, and uses `CoreStats.CastSpeed` to determine spell `CastTime`.

### Summary

You have successfully implemented the **Casting State Machine** in the C# Brain, governing the player's spellcasting flow through distinct states (Idle, Channeling, CastStart, Casting, Recovery). By designing `CastingSystem` to manage these transitions, deduct `ChakraCost` and `StabilityCost` from `CoreStats` (via `PlayerStatSystem`), and utilize `CoreStats.CastSpeed` for `CastTime` adjustments, you've established precise control over spell execution. This crucial system strictly adheres to TDD 02.4's specifications, providing a robust foundation for Sigilborne's dynamic magic, with all core stats now unified under `CoreStats`.

### Next Steps

This concludes **Module 4: Combos & Casting - The Art of Ninjutsu**. We will now move back to **Module 5: Chakra & Life Systems**, starting with **Metabolism System - Dynamic Decay Rates (C#)**, where we will refine how hunger, thirst, and other biological needs dynamically decay based on player activity and environmental factors.