## Chapter 5.3: Metabolism System - Dynamic Decay Rates (C#)

Building on our `BiologicalSystem` and `CoreStats`, this chapter refines the management of biological needs by implementing a **Metabolism System**. This system will dynamically adjust the decay rates of `Hunger` and `Thirst` based on the entity's `ActivityLevel` and (conceptually) `WeatherMultiplier`, ensuring a more realistic and engaging survival simulation, as specified in TDD 03.2.

### 1. The Dynamic Nature of Metabolism

The GDD (B14.4) states that lack of sleep/overexertion causes elevated strain and miscast risk. This implies that high activity should have consequences beyond just stamina drain. A dynamic metabolism system ensures:

*   **Realism**: Active entities burn more calories and get thirsty faster.
*   **Player Choice**: Encourages strategic rest and resource management.
*   **Environmental Interaction**: Weather will (conceptually) further influence metabolic rates.

### 2. Enhancing `BiologicalSystem.cs` for Metabolism

Our `BiologicalSystem` already contains the `ProcessBioTick` method where hunger and thirst decay. We'll enhance `GetActivityLevel` and integrate `WeatherSystem` (which we'll define conceptually for now, as TDD 03.3 covers it later) to apply these dynamic modifiers.

Open `res://_Brain/Systems/Biology/BiologicalSystem.cs`:

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
using Sigilborne.Systems.Weather; // New: For WeatherSystem

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private TransformSystem _transformSystem;
        private MovementSystem _movementSystem;
        private WeatherSystem _weatherSystem; // New: Reference to WeatherSystem

        private Dictionary<EntityID, CoreStats> _entityCoreStats = new Dictionary<EntityID, CoreStats>();

        private const float BIO_TICK_RATE = 1.0f; // 1.0f = once per real second (1Hz)
        private float _bioTickTimer;

        public BiologicalSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, MovementSystem movementSystem, WeatherSystem weatherSystem) // Add WeatherSystem
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _movementSystem = movementSystem;
            _weatherSystem = weatherSystem; // Store WeatherSystem reference

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("BiologicalSystem: Initialized.");
        }

        // ... (OnEntitySpawned, OnEntityDespawned, Tick methods) ...

        private void ProcessBioTick()
        {
            foreach (var kvp in _entityCoreStats)
            {
                EntityID id = kvp.Key;
                ref CoreStats coreStats = ref _entityCoreStats.GetValueRef(id);

                if (!_entityManager.IsValid(id)) continue;

                // --- Metabolism System (TDD 03.2) ---
                float activityMultiplier = GetActivityLevel(id); // 1x Idle, 2x Running, 5x Combat (conceptual)
                float weatherMultiplier = GetWeatherMultiplier(id); // New: Get multiplier based on local weather

                // Hunger decay
                coreStats.Hunger -= 1.0f * activityMultiplier * weatherMultiplier; // Base decay of 1 per tick
                coreStats.Hunger = Mathf.Max(0, coreStats.Hunger);
                
                // Thirst decay
                coreStats.Thirst -= 1.5f * activityMultiplier * weatherMultiplier; // Thirst decays faster
                coreStats.Thirst = Mathf.Max(0, coreStats.Thirst);

                // Body temperature (placeholder for environmental effects)
                // coreStats.BodyTemp = AdjustBodyTemperature(id, coreStats.BodyTemp, activityMultiplier, weatherMultiplier);

                // --- Regeneration (Health, Stamina, Chakra, Stability) ---
                // ... (existing regen logic) ...

                // --- Apply consequences of low stats (GDD B14.4) ---
                // ... (existing low stat consequences) ...

                // Publish updates for the Body (e.g., UI for player)
                if (id == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerCoreStatsChangedEvent { PlayerID = id, NewCoreStats = coreStats });
                }
            }
        }

        /// <summary>
        /// Determines the activity level of an entity for metabolism calculations.
        /// (TDD 03.2)
        /// </summary>
        private float GetActivityLevel(EntityID id)
        {
            // For player, activity is based on movement.
            if (id == _entityManager.GetPlayerEntityID())
            {
                // If the player has a significant velocity, consider them "running".
                if (_movementSystem.GetVelocity(id).LengthSquared() > 1.0f)
                {
                    return 2.0f; // 2x decay for running
                }
                // Later: check if in combat for 5x multiplier (conceptual)
                // if (_combatSystem.IsInCombat(id)) return 5.0f;
            }
            // For NPCs/Animals, could be based on their AI state (e.g., fleeing, fighting, patrolling).
            return 1.0f; // Default 1x decay for idle/NPCs
        }

        /// <summary>
        /// Determines the weather-based multiplier for metabolism.
        /// (TDD 03.3) - Conceptual for now, will be fleshed out in Chapter 5.4.
        /// </summary>
        private float GetWeatherMultiplier(EntityID id)
        {
            // For now, a simple placeholder.
            // In Chapter 5.4, this will query the WeatherSystem based on entity's location.
            // Example:
            // if (_transformSystem.TryGetTransform(id, out TransformComponent transform))
            // {
            //     WeatherType currentWeather = _weatherSystem.GetCurrentWeatherAt(transform.Position);
            //     float temperature = _weatherSystem.GetTemperatureAt(transform.Position);
            //     if (temperature < coreStats.NormalBodyTemp - 10) return 1.2f; // Colder increases metabolism
            //     if (temperature > coreStats.NormalBodyTemp + 10) return 1.1f; // Hotter increases thirst decay more
            // }
            return 1.0f; // Default no multiplier
        }

        public ref CoreStats GetCoreStatsRef(EntityID id) { /* ... */ return ref _entityCoreStats.GetValueRef(id); }
        public bool TryGetCoreStats(EntityID id, out CoreStats coreStats) { /* ... */ return false; }
        // ... (Helper Events) ...
    }
}
```

### 3. Implementing a Conceptual `WeatherSystem.cs`

For `BiologicalSystem` to compile, we need a placeholder `WeatherSystem`. We'll flesh this out in Chapter 5.4.

1.  Create a new folder `res://_Brain/Systems/Weather/`.
2.  Create `res://_Brain/Systems/Weather/WeatherSystem.cs`:

```csharp
// _Brain/Systems/Weather/WeatherSystem.cs
using Godot;
using System;

namespace Sigilborne.Systems.Weather
{
    /// <summary>
    /// Placeholder for the WeatherSystem, which will manage global and local weather states.
    /// (TDD 03.3)
    /// </summary>
    public class WeatherSystem
    {
        public enum WeatherType { Clear, Rain, Storm, Snow, Fog, AshFall }

        public WeatherSystem()
        {
            GD.Print("WeatherSystem: Initialized (placeholder).");
        }

        // Placeholder methods for BiologicalSystem to call
        public WeatherType GetCurrentWeatherAt(Vector2 position) => WeatherType.Clear;
        public float GetTemperatureAt(Vector2 position) => 20.0f; // Default 20 C
    }
}
```

### 4. Integrating `WeatherSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Weather;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `WeatherSystem` property.
3.  Initialize `WeatherSystem` in `InitializeSystems()` **before** `BiologicalSystem`.
4.  Pass `WeatherSystem` to `BiologicalSystem`'s constructor.

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
using Sigilborne.Systems.Weather; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public BiologicalSystem Biology { get; private set; }
    public WeatherSystem Weather { get; private set; } // Add WeatherSystem property

    public override void _Ready() { /* ... */ }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta);
        Casting.Tick(delta);
        Biology.Tick(delta);
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to PhysicsSystem) ...

        // ... (existing magic system initializations) ...
        
        // Initialize WeatherSystem BEFORE BiologicalSystem
        Weather = new WeatherSystem(); // Initialize WeatherSystem here
        GD.Print("  - WeatherSystem initialized.");

        // Initialize BiologicalSystem, passing WeatherSystem
        Biology = new BiologicalSystem(Entities, Events, Transforms, Movement, Weather); // Pass WeatherSystem
        GD.Print("  - BiologicalSystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 5. Testing Dynamic Decay Rates

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe Hunger and Thirst values on the HUD.
    *   **Idle**: Stand still. Hunger and Thirst should decay at their base rates (1.0 and 1.5 per second).
    *   **Moving**: Move the player (WASD). Hunger and Thirst should decay faster (2.0 and 3.0 per second) due to the `activityMultiplier` from `GetActivityLevel()`.

This confirms that the `Metabolism System` in `BiologicalSystem` is dynamically adjusting decay rates based on activity, laying the groundwork for more complex environmental interactions.

### Summary

You have successfully implemented the **Metabolism System** in the C# Brain, dynamically adjusting the decay rates of `Hunger` and `Thirst` within `BiologicalSystem`. By enhancing `GetActivityLevel` to factor in movement and integrating a conceptual `WeatherSystem`, you've created a more realistic and engaging survival simulation, strictly adhering to TDD 03.2's specifications. This crucial step ensures biological needs are not static but react to player activity, influencing resource management.

### Next Steps

The next chapter will focus on **Weather & Environment**, where we will fully implement the `WeatherSystem` to manage global and local weather states, and design how environmental pressure directly impacts `CoreStats` and biological processes.