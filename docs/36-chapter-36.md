## Chapter 5.7: Recovery Logic - Health, Stamina, Chakra Regeneration (C#)

Our `BiologicalSystem` currently applies hunger/thirst decay and basic regeneration. This chapter refines the **Recovery Logic** for `Health`, `Stamina`, `Chakra`, and `Stability` in the C# Brain. We'll ensure regeneration rates are influenced by `CoreStats`, activity levels, and (conceptually) status effects or environmental conditions, providing a comprehensive system for entity well-being, as specified in TDD 03.4 and GDD B12.11.

### 1. The Importance of Dynamic Recovery

The GDD (B03.4) highlights deliberate recovery mechanisms (meditation, food, rest). This implies that recovery isn't just a static rate but should be influenced by:

*   **CoreStats**: `ChakraRegenRate`, `StaminaRegenRate`, `StabilityRegenRate`.
*   **Activity Level**: Rest should accelerate recovery, high activity should slow or halt it.
*   **Status Effects**: Wounds (e.g., Bleed) or debuffs (e.g., Chakra Drain) should impair regeneration.
*   **Environment**: Resonant locations (GDD B03.4.E) could boost recovery.

### 2. Enhancing `BiologicalSystem.cs` for Regeneration

Our `BiologicalSystem.ProcessBioTick` already has basic regeneration. We'll refine it to use `CoreStats`'s dedicated regeneration rates and factor in activity level.

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
using Sigilborne.Systems.Weather;
using Sigilborne.Systems.StatusEffects; // New: For StatusEffectSystem

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private TransformSystem _transformSystem;
        private MovementSystem _movementSystem;
        private WeatherSystem _weatherSystem;
        private StatusEffectSystem _statusEffectSystem; // New: Reference to StatusEffectSystem

        private Dictionary<EntityID, CoreStats> _entityCoreStats = new Dictionary<EntityID, CoreStats>();

        private const float BIO_TICK_RATE = 1.0f; // 1.0f = once per real second (1Hz)
        private float _bioTickTimer;

        public BiologicalSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, MovementSystem movementSystem, WeatherSystem weatherSystem, StatusEffectSystem statusEffectSystem) // Add StatusEffectSystem
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _movementSystem = movementSystem;
            _weatherSystem = weatherSystem;
            _statusEffectSystem = statusEffectSystem; // Store StatusEffectSystem reference

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

                if (!_entityManager.IsValid(id) || coreStats.Health <= 0) continue; // Don't process dead entities

                float activityMultiplier = GetActivityLevel(id); // 1x Idle, 2x Running, 5x Combat (conceptual)
                float weatherMetabolismMultiplier = GetWeatherMultiplier(id);
                
                // --- Hunger and Thirst Decay ---
                coreStats.Hunger -= 1.0f * activityMultiplier * weatherMetabolismMultiplier;
                coreStats.Hunger = Mathf.Max(0, coreStats.Hunger);
                
                coreStats.Thirst -= 1.5f * activityMultiplier * weatherMetabolismMultiplier;
                coreStats.Thirst = Mathf.Max(0, coreStats.Thirst);

                // --- Body temperature adjustment ---
                // ... (existing temp logic) ...

                // --- Regeneration (Health, Stamina, Chakra, Stability) (TDD 03.4) ---
                // GDD B12.11: Natural Healing Speed (slow)
                // GDD B03.4: Strain Reset Mechanisms

                float regenRateMultiplier = 1.0f;
                if (activityMultiplier > 1.0f) regenRateMultiplier = 0.5f; // Reduce regen if active
                if (activityMultiplier > 4.0f) regenRateMultiplier = 0.0f; // Halt regen if in combat (conceptual)

                // Check for status effects that impair regeneration
                if (_statusEffectSystem.HasEffectOfType(id, EffectType.ChakraDrain) || _statusEffectSystem.HasEffectOfType(id, EffectType.StabilityDrain))
                {
                    regenRateMultiplier *= 0.1f; // Severely impair regen
                }
                if (_statusEffectSystem.HasEffectOfType(id, EffectType.ControlImpair)) // e.g., Stun, Root
                {
                    regenRateMultiplier *= 0.5f; // Reduce regen
                }
                if (_statusEffectSystem.HasEffectOfType(id, EffectType.DamageOverTime)) // Bleeding/Burning
                {
                    regenRateMultiplier *= 0.2f; // Significantly impair regen
                }

                // Health Regen (only if not starving/dehydrated, and not in combat, and not full)
                if (coreStats.Hunger > 10 && coreStats.Thirst > 10 && coreStats.Health < coreStats.MaxHealth)
                {
                    coreStats.Health += coreStats.HealthRegenRate * regenRateMultiplier * BIO_TICK_RATE;
                    coreStats.Health = Mathf.Min(coreStats.MaxHealth, coreStats.Health);
                }

                // Stamina Regen (GDD B03.4)
                if (coreStats.Stamina < coreStats.MaxStamina)
                {
                    coreStats.Stamina += coreStats.StaminaRegenRate * regenRateMultiplier * BIO_TICK_RATE;
                    coreStats.Stamina = Mathf.Min(coreStats.MaxStamina, coreStats.Stamina);
                }

                // Chakra Regen (GDD B03.4)
                if (coreStats.Chakra < coreStats.MaxChakra)
                {
                    coreStats.Chakra += coreStats.ChakraRegenRate * regenRateMultiplier * BIO_TICK_RATE;
                    coreStats.Chakra = Mathf.Min(coreStats.MaxChakra, coreStats.Chakra);
                }

                // Stability Regen (GDD B03.6)
                if (coreStats.Stability < coreStats.MaxStability)
                {
                    coreStats.Stability += coreStats.StabilityRegenRate * regenRateMultiplier * BIO_TICK_RATE;
                    coreStats.Stability = Mathf.Min(coreStats.MaxStability, coreStats.Stability);
                }

                // --- Apply consequences of low stats ---
                // ... (existing low stat consequences) ...

                // Publish updates for the Body (e.g., UI for player)
                if (id == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerCoreStatsChangedEvent { PlayerID = id, NewCoreStats = coreStats });
                }
            }
        }

        public float GetActivityLevel(EntityID id)
        {
            // For player, activity is based on movement.
            if (id == _entityManager.GetPlayerEntityID())
            {
                if (_movementSystem.GetVelocity(id).LengthSquared() > 1.0f)
                {
                    return 2.0f; // 2x decay for running
                }
            }
            return 1.0f; // Default 1x decay for idle/NPCs
        }

        // ... (GetWeatherMultiplier, GetCoreStatsRef, TryGetCoreStats methods) ...
        // ... (Helper Events) ...
    }
}
```

#### 2.1. Add `HasEffectOfType` to `StatusEffectSystem.cs`

`BiologicalSystem` needs to query active effects.

Open `res://_Brain/Systems/StatusEffects/StatusEffectSystem.cs` and add this method:

```csharp
// _Brain/Systems/StatusEffects/StatusEffectSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic.Components;
using Sigilborne.Systems.Biology;
using Sigilborne.Core;
using System.Linq;

namespace Sigilborne.Systems.StatusEffects
{
    // ... (EffectType, EffectDefinition, ActiveEffect structs/classes) ...

    public class StatusEffectSystem
    {
        // ... (existing fields and constructor) ...

        public void Tick(double delta) { /* ... */ }
        private void ApplyEffectTick(EntityID targetID, ActiveEffect effect) { /* ... */ }

        /// <summary>
        /// Checks if an entity has any active effect of a specific type.
        /// </summary>
        public bool HasEffectOfType(EntityID targetID, EffectType type)
        {
            if (_activeEffects.TryGetValue(targetID, out List<ActiveEffect> effects))
            {
                return effects.Any(e => e.Definition.Type == type);
            }
            return false;
        }

        // ... (GetActiveEffects, Helper Events) ...
    }
}
```

### 3. Integrating `StatusEffectSystem` into `BiologicalSystem`

1.  Open `res://_Brain/Core/GameManager.cs`.
2.  Modify the `BiologicalSystem` initialization in `InitializeSystems()` to pass `StatusEffects`.

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        // Initialize BiologicalSystem, passing StatusEffectSystem
        Biology = new BiologicalSystem(Entities, Events, Transforms, Movement, Weather, StatusEffects); // Pass StatusEffects
        GD.Print("  - BiologicalSystem initialized.");
// ...
```

### 4. Testing Dynamic Recovery

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe Health, Stamina, Chakra, and Stability on the HUD.
    *   **Idle Regeneration**: Stand still. All four stats should slowly regenerate towards their max values.
    *   **Active Regeneration**: Move the player (WASD). Regeneration rates for all stats should decrease due to the `regenRateMultiplier` based on `activityMultiplier`.
    *   **Impaired Regeneration (Conceptual)**: Use the debug console to apply a `DamageOverTime` effect (e.g., `/damage 1 Physical`, if `bleed_t1` is generated). You won't see the effect icon yet, but in the console, `DamageSystem` will apply the bleed, and `BiologicalSystem` will show reduced regeneration if `bleed_t1` is applied.

This confirms that the `Recovery Logic` in `BiologicalSystem` is dynamically adjusting regeneration rates based on activity and (conceptually) impairing them with status effects.

### Summary

You have successfully implemented the **Recovery Logic** for `Health`, `Stamina`, `Chakra`, and `Stability` in the C# Brain, ensuring regeneration rates are dynamically influenced by `CoreStats` and `ActivityLevel`. By enhancing `BiologicalSystem` to apply a `regenRateMultiplier` and query `StatusEffectSystem` for impairments, you've created a comprehensive system for entity well-being, strictly adhering to TDD 03.4 and GDD B12.11's specifications. This crucial step adds depth to resource management and interaction with in-game conditions.

### Next Steps

The next chapter will focus on **Environmental Pressure**, where we will fully implement the `WeatherSystem` to manage global and local weather states, and design how environmental pressure (temperature, specific weather types) directly impacts `CoreStats` and biological processes.