## Chapter 5.6: Status Effect Data - Definition vs. Instance (C#)

Our `DamageSystem` can now trigger `StatusEffect`s (wounds), and our `BiologicalSystem` manages `CoreStats`. This chapter focuses on fully implementing the **Status Effect System**, starting with defining `EffectDefinition` (static data) and `ActiveEffect` (runtime instance) to manage buffs, debuffs, and wounds. This standardized lifecycle ensures effects are applied, tick, and are cleaned up correctly, preventing bugs like "infinite burn," as specified in TDD 18.2.

### 1. The Dual Nature of Status Effects

TDD 18.2 distinguishes between:

*   **`EffectDefinition`**: The static blueprint of an effect (e.g., "Bleed_T1" always lasts 5 seconds, ticks for X damage). This is data.
*   **`ActiveEffect`**: A live instance of an effect applied to a specific entity, with its current `TimeRemaining`, `Stacks`, and `SourceID`. This is runtime state.

This separation allows for flexible data-driven effect creation and efficient runtime management.

### 2. Defining `EffectDefinition`

This will be our static blueprint for all status effects.

1.  Open `res://_Brain/Systems/StatusEffects/StatusEffectSystem.cs` and add `EffectDefinition` as a nested class (or in its own file if preferred, but for now, nested is fine for cohesion).

```csharp
// _Brain/Systems/StatusEffects/StatusEffectSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic.Components; // For StatusEffectData
using Sigilborne.Systems.Biology; // For CoreStats
using Sigilborne.Core; // For EventBus

namespace Sigilborne.Systems.StatusEffects
{
    /// <summary>
    /// Defines the types of status effects.
    /// </summary>
    public enum EffectType
    {
        None,
        DamageOverTime, // DoT
        StatModifier,   // Modifies a stat (e.g., speed, armor)
        Buff,           // Positive stat modifier
        Debuff,         // Negative stat modifier
        HealingOverTime, // HoT
        ChakraDrain,    // Drains chakra over time
        StabilityDrain, // Drains stability over time
        ControlImpair,  // Stun, Root, Slow
        VoidSickness    // Permanent/persistent corruption effect (C04)
    }

    /// <summary>
    /// Static data defining a status effect.
    /// (TDD 18.2.1)
    /// </summary>
    public class EffectDefinition
    {
        public string ID { get; private set; } // "burn_t1", "slow_short"
        public EffectType Type { get; private set; }
        public int MaxStacks { get; private set; }
        public float BaseDuration { get; private set; } // Base duration before modifiers
        public float TickRate { get; private set; } // How often the effect "ticks" (for DoT, HoT, etc.)
        public float BaseStrength { get; private set; } // Base magnitude of the effect
        public bool IsPermanent { get; private set; } // TDD 18.4.1: VoidSickness is permanent until cured.
        public string VisualFXID { get; private set; } // ID for Body to play VFX (e.g., "burn_particles")
        public string SoundFXID { get; private set; } // ID for Body to play SFX

        // For StatModifier effects: which CoreStats field to modify and how
        public string StatToModify { get; private set; } // e.g., "MoveSpeed", "Armor"
        public float StatModifierValue { get; private set; } // e.g., -0.2 (20% reduction), 10.0 (flat increase)
        public bool IsPercentageModifier { get; private set; } // true for % mod, false for flat

        public EffectDefinition(string id, EffectType type, int maxStacks, float baseDuration, float tickRate, float baseStrength, bool isPermanent = false, string visualFxId = null, string soundFxId = null, string statToModify = null, float statModifierValue = 0f, bool isPercentageModifier = false)
        {
            ID = id;
            Type = type;
            MaxStacks = maxStacks;
            BaseDuration = baseDuration;
            TickRate = tickRate;
            BaseStrength = baseStrength;
            IsPermanent = isPermanent;
            VisualFXID = visualFxId;
            SoundFXID = soundFxId;
            StatToModify = statToModify;
            StatModifierValue = statModifierValue;
            IsPercentageModifier = isPercentageModifier;
        }

        public override string ToString()
        {
            return $"EffectDef: '{ID}' ({Type}) | Str: {BaseStrength}, Dur: {BaseDuration}, Stacks: {MaxStacks}";
        }
    }
    // ... (rest of StatusEffectSystem class) ...
}
```

### 3. Defining `ActiveEffect`

This represents a live instance of an effect on an entity.

1.  Open `res://_Brain/Systems/StatusEffects/StatusEffectSystem.cs` and add `ActiveEffect` as a nested struct.

```csharp
// _Brain/Systems/StatusEffects/StatusEffectSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic.Components; // For StatusEffectData
using Sigilborne.Systems.Biology; // For CoreStats
using Sigilborne.Core; // For EventBus

namespace Sigilborne.Systems.StatusEffects
{
    // ... (EffectType enum) ...
    // ... (EffectDefinition class) ...

    /// <summary>
    /// Represents an active instance of a status effect applied to an entity.
    /// (TDD 18.2.2)
    /// </summary>
    public struct ActiveEffect
    {
        public EffectDefinition Definition; // Reference to the static definition
        public int Stacks;                  // Current number of stacks (TDD 18.2.2)
        public float TimeRemaining;         // Remaining duration of the effect (TDD 18.2.2)
        public float TimeSinceLastTick;     // For effects that tick periodically
        public EntityID SourceID;           // Who applied this effect (for kill credit, etc.)
        public EntityID TargetID;           // Who has this effect

        public ActiveEffect(EffectDefinition definition, EntityID targetID, EntityID sourceID, float initialDuration, float initialStrength, int initialStacks = 1)
        {
            Definition = definition;
            TargetID = targetID;
            SourceID = sourceID;
            Stacks = initialStacks;
            TimeRemaining = initialDuration;
            TimeSinceLastTick = 0; // Start at 0 for first tick
            // Initial strength from EffectDefinition is used, initialStrength parameter
            // allows for overrides (e.g., spell specific strength).
        }

        public override string ToString()
        {
            return $"ActiveEffect: '{Definition.ID}' | Stacks: {Stacks}, Rem: {TimeRemaining:F1}s";
        }
    }
    // ... (rest of StatusEffectSystem class) ...
}
```

### 4. Enhancing `StatusEffectSystem.cs` with Data Management

Now, let's update `StatusEffectSystem` to manage a collection of `EffectDefinition`s and `ActiveEffect`s.

1.  Open `res://_Brain/Systems/StatusEffects/StatusEffectSystem.cs`.
2.  Modify the class to include dictionaries for definitions and active effects.
3.  Implement `RegisterEffectDefinition` and `ApplyEffect`.
4.  Implement the `Tick` method for the `StatusEffectSystem` itself.

```csharp
// _Brain/Systems/StatusEffects/StatusEffectSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic.Components;
using Sigilborne.Systems.Biology;
using Sigilborne.Core;
using System.Linq; // For Any()

namespace Sigilborne.Systems.StatusEffects
{
    // ... (EffectType enum) ...
    // ... (EffectDefinition class) ...
    // ... (ActiveEffect struct) ...

    /// <summary>
    /// Manages the lifecycle and application of all status effects (buffs, debuffs, wounds).
    /// (TDD 03.4, TDD 18)
    /// </summary>
    public class StatusEffectSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem; // To modify CoreStats

        // Static definitions of all possible effects
        private Dictionary<string, EffectDefinition> _effectDefinitions = new Dictionary<string, EffectDefinition>();

        // Active effects on entities: Dictionary<EntityID, List<ActiveEffect>>
        // Using a List<ActiveEffect> as an entity can have multiple effects.
        private Dictionary<EntityID, List<ActiveEffect>> _activeEffects = new Dictionary<EntityID, List<ActiveEffect>>();

        public StatusEffectSystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            GD.Print("StatusEffectSystem: Initialized.");

            // Subscribe to entity despawn events to clean up effects
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            RegisterDefaultEffects(); // Register some default effects for testing
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _activeEffects.Remove(e.ID); // Clean up all effects on a despawned entity
            GD.Print($"StatusEffectSystem: Cleared effects for despawned entity {e.ID}.");
        }

        /// <summary>
        /// Registers a new static EffectDefinition.
        /// </summary>
        public void RegisterEffectDefinition(EffectDefinition def)
        {
            _effectDefinitions[def.ID] = def;
            GD.Print($"StatusEffectSystem: Registered effect definition '{def.ID}'.");
        }

        /// <summary>
        /// Registers some common default effects for initial testing.
        /// </summary>
        private void RegisterDefaultEffects()
        {
            RegisterEffectDefinition(new EffectDefinition("bleed_t1", EffectType.DamageOverTime, 3, 5f, 1f, 2f, visualFxId: "bleed_vfx"));
            RegisterEffectDefinition(new EffectDefinition("burn_t1", EffectType.DamageOverTime, 5, 3f, 0.5f, 3f, visualFxId: "burn_vfx"));
            RegisterEffectDefinition(new EffectDefinition("chakra_drain_t1", EffectType.ChakraDrain, 1, 2f, 0.5f, 5f, visualFxId: "drain_vfx"));
            RegisterEffectDefinition(new EffectDefinition("void_sickness_t1", EffectType.VoidSickness, 100, float.MaxValue, 10f, 1f, isPermanent: true, visualFxId: "void_aura_vfx")); // TDD 18.4.1
            RegisterEffectDefinition(new EffectDefinition("slow_short", EffectType.ControlImpair, 1, 3f, 0f, 0.5f, statToModify: "MoveSpeed", isPercentageModifier: true)); // 50% slow
            RegisterEffectDefinition(new EffectDefinition("regen_t1", EffectType.HealingOverTime, 1, 5f, 1f, 5f));
            // Add more as needed
        }

        /// <summary>
        /// Applies a status effect to a target entity.
        /// (TDD 18.3.1: Application)
        /// </summary>
        public void ApplyEffect(EntityID targetID, StatusEffectData effectData, EntityID sourceID = default)
        {
            if (!_entityManager.IsValid(targetID))
            {
                GD.PrintErr($"StatusEffectSystem: Attempted to apply effect to invalid entity {targetID}.");
                return;
            }
            if (!_effectDefinitions.TryGetValue(effectData.EffectID, out EffectDefinition def))
            {
                GD.PrintErr($"StatusEffectSystem: Effect definition '{effectData.EffectID}' not found.");
                return;
            }

            if (!_activeEffects.ContainsKey(targetID))
            {
                _activeEffects[targetID] = new List<ActiveEffect>();
            }

            // Check if the target already has this effect (TDD 18.3.1)
            int existingEffectIndex = _activeEffects[targetID].FindIndex(ae => ae.Definition.ID == def.ID);

            if (existingEffectIndex != -1)
            {
                // Effect already exists: Refresh duration or stack (TDD 18.3.1)
                ActiveEffect existingEffect = _activeEffects[targetID][existingEffectIndex];
                
                // Refresh duration
                existingEffect.TimeRemaining = def.BaseDuration;
                
                // Stack if allowed
                if (existingEffect.Stacks < def.MaxStacks)
                {
                    existingEffect.Stacks++;
                    GD.Print($"StatusEffectSystem: Stacked '{def.ID}' on {targetID}. New stacks: {existingEffect.Stacks}");
                }
                _activeEffects[targetID][existingEffectIndex] = existingEffect; // Update struct in list

                // Emit event for UI/VFX update (e.g., stack count changed)
                _eventBus.Publish(new EffectUpdatedEvent { TargetID = targetID, Effect = existingEffect, IsNew = false });
            }
            else
            {
                // New effect: Create and add
                ActiveEffect newEffect = new ActiveEffect(def, targetID, sourceID, def.BaseDuration, effectData.Strength);
                _activeEffects[targetID].Add(newEffect);
                GD.Print($"StatusEffectSystem: Applied new effect '{def.ID}' to {targetID}.");

                // Emit event for UI/VFX update (e.g., new icon, particle effect)
                _eventBus.Publish(new EffectAppliedEvent { TargetID = targetID, Effect = newEffect });
            }
        }

        /// <summary>
        /// Main update loop for the StatusEffectSystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// (TDD 18.3.2: The Tick Loop)
        /// </summary>
        public void Tick(double delta)
        {
            List<EntityID> entitiesToRemoveEffectsFrom = new List<EntityID>();

            foreach (var kvp in _activeEffects)
            {
                EntityID targetID = kvp.Key;
                List<ActiveEffect> effectsOnTarget = kvp.Value;

                if (!_entityManager.IsValid(targetID))
                {
                    entitiesToRemoveEffectsFrom.Add(targetID); // Mark for cleanup
                    continue;
                }

                List<ActiveEffect> effectsToRemove = new List<ActiveEffect>();
                for (int i = effectsOnTarget.Count - 1; i >= 0; i--)
                {
                    ActiveEffect effect = effectsOnTarget[i];
                    effect.TimeRemaining -= (float)delta;
                    effect.TimeSinceLastTick += (float)delta;

                    // Apply effect tick (TDD 18.3.2)
                    if (effect.Definition.TickRate > 0 && effect.TimeSinceLastTick >= effect.Definition.TickRate)
                    {
                        ApplyEffectTick(targetID, effect);
                        effect.TimeSinceLastTick = 0; // Reset tick timer
                    }

                    // Check for expiration (TDD 18.3.2)
                    if (!effect.Definition.IsPermanent && effect.TimeRemaining <= 0)
                    {
                        effectsToRemove.Add(effect);
                        GD.Print($"StatusEffectSystem: Effect '{effect.Definition.ID}' expired on {targetID}.");
                    }
                    else
                    {
                        effectsOnTarget[i] = effect; // Update the struct in the list
                    }
                }

                // Remove expired effects
                foreach (var expiredEffect in effectsToRemove)
                {
                    effectsOnTarget.Remove(expiredEffect);
                    _eventBus.Publish(new EffectRemovedEvent { TargetID = targetID, EffectID = expiredEffect.Definition.ID });
                }

                if (effectsOnTarget.Count == 0)
                {
                    entitiesToRemoveEffectsFrom.Add(targetID);
                }
            }

            // Clean up entities with no active effects
            foreach (var id in entitiesToRemoveEffectsFrom)
            {
                _activeEffects.Remove(id);
            }
        }

        /// <summary>
        /// Applies the periodic effect of an active status effect.
        /// (TDD 18.3.2)
        /// </summary>
        private void ApplyEffectTick(EntityID targetID, ActiveEffect effect)
        {
            if (!_biologicalSystem.TryGetCoreStats(targetID, out CoreStats coreStats)) return;

            float strength = effect.Definition.BaseStrength * effect.Stacks; // Strength scales with stacks

            switch (effect.Definition.Type)
            {
                case EffectType.DamageOverTime:
                    // Apply direct damage (GDD B12.4: Cumulative Injuries)
                    GameManager.Instance.Combat.ApplyDamage(effect.SourceID, targetID, strength, DamageType.Physical); // DoT is typically physical/elemental
                    break;
                case EffectType.HealingOverTime:
                    // Apply healing (GDD B12.11: Natural Healing Speed)
                    ref CoreStats targetStatsHeal = ref _biologicalSystem.GetCoreStatsRef(targetID);
                    targetStatsHeal.Health = Mathf.Min(targetStatsHeal.MaxHealth, targetStatsHeal.Health + strength);
                    _eventBus.Publish(new PlayerStatSystem.PlayerHealthChangedEvent { PlayerID = targetID, NewValue = targetStatsHeal.Health, MaxValue = targetStatsHeal.MaxHealth });
                    break;
                case EffectType.ChakraDrain:
                    ref CoreStats targetStatsChakra = ref _biologicalSystem.GetCoreStatsRef(targetID);
                    targetStatsChakra.Chakra = Mathf.Max(0, targetStatsChakra.Chakra - strength);
                     _eventBus.Publish(new PlayerStatSystem.PlayerChakraChangedEvent { PlayerID = targetID, NewValue = targetStatsChakra.Chakra, MaxValue = targetStatsChakra.MaxChakra });
                    break;
                case EffectType.StabilityDrain:
                    ref CoreStats targetStatsStability = ref _biologicalSystem.GetCoreStatsRef(targetID);
                    targetStatsStability.Stability = Mathf.Max(0, targetStatsStability.Stability - strength);
                    _eventBus.Publish(new PlayerStatSystem.PlayerStabilityChangedEvent { PlayerID = targetID, NewValue = targetStatsStability.Stability, MaxValue = targetStatsStability.MaxStability });
                    break;
                case EffectType.VoidSickness: // TDD 18.4.1
                    // Apply long-term void sickness effects here (e.g., reduce MaxHealth/Chakra, random miscasts)
                    GD.Print($"StatusEffectSystem: {targetID} suffering Void Sickness tick (Stacks: {effect.Stacks}).");
                    // Example: reduce max health permanently
                    // ref CoreStats voidStats = ref _biologicalSystem.GetCoreStatsRef(targetID);
                    // voidStats.MaxHealth -= strength * 0.1f; // Small permanent max health reduction per tick
                    break;
                // StatModifier and ControlImpair effects are typically applied continuously/on demand, not per tick.
            }
        }

        /// <summary>
        /// Retrieves all active effects on a specific entity.
        /// </summary>
        public IReadOnlyList<ActiveEffect> GetActiveEffects(EntityID targetID)
        {
            if (_activeEffects.TryGetValue(targetID, out List<ActiveEffect> effects))
            {
                return effects.AsReadOnly();
            }
            return new List<ActiveEffect>().AsReadOnly();
        }

        // --- Helper Events for Body Sync ---
        public struct EffectAppliedEvent { public EntityID TargetID; public ActiveEffect Effect; }
        public struct EffectUpdatedEvent { public EntityID TargetID; public ActiveEffect Effect; public bool IsNew; } // IsNew for UI to know if to create/update
        public struct EffectRemovedEvent { public EntityID TargetID; public string EffectID; }
    }
}
```

### 5. Integrating `StatusEffectSystem` into `GameManager`

We already added the `StatusEffectSystem` property and initialized it. We just need to ensure `BiologicalSystem` is passed to its constructor.

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        BiologicalSystem = new BiologicalSystem(Entities, Events, Transforms, Movement, Weather);
        GD.Print("  - BiologicalSystem initialized.");

        // Initialize StatusEffectSystem, passing BiologicalSystem
        StatusEffects = new StatusEffectSystem(Entities, Events, BiologicalSystem); // Pass BiologicalSystem
        GD.Print("  - StatusEffectSystem initialized.");
// ...
```

### 6. Updating `DamageSystem.cs` to use `StatusEffectSystem`

Our `DamageSystem` placeholder now needs to use the fully implemented `StatusEffectSystem`.

1.  Open `res://_Brain/Systems/Combat/DamageSystem.cs`.
2.  Uncomment the `_statusEffectSystem.ApplyEffect()` call in `GenerateWounds()`.

```csharp
// _Brain/Systems/Combat/DamageSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.StatusEffects; // Ensure this is present

namespace Sigilborne.Systems.Combat
{
    // ... (DamageType, DamageResult structs/enums) ...

    public class DamageSystem
    {
        // ... (existing fields and constructor) ...

        public DamageResult ApplyDamage(EntityID attackerID, EntityID defenderID, float rawDamage, DamageType damageType, float penetration = 0f)
        {
            // ... (existing damage calculation and health application) ...

            // 4. Wound Generation (TDD 03.4) - Chance to create a StatusEffect
            GenerateWounds(defenderID, result);

            // ... (existing defeat logic) ...
        }

        private void GenerateWounds(EntityID targetID, DamageResult result)
        {
            if (result.FinalDamage <= 0) return;

            float woundChance = result.FinalDamage / _biologicalSystem.GetCoreStatsRef(targetID).MaxHealth;
            if (woundChance > 0.1f)
            {
                Random rand = new Random(targetID.Index + (int)GameManager.Instance.Time.CurrentGameTime);
                if (rand.NextDouble() < woundChance)
                {
                    string effectId = "";
                    float duration = 0;
                    float strength = 0;

                    switch (result.Type)
                    {
                        case DamageType.Physical:
                            effectId = "bleed_t1";
                            duration = 5f;
                            strength = result.FinalDamage * 0.05f;
                            break;
                        case DamageType.Elemental:
                            effectId = "burn_t1";
                            duration = 3f;
                            strength = result.FinalDamage * 0.03f;
                            break;
                        case DamageType.Chakra:
                            effectId = "chakra_drain_t1";
                            duration = 2f;
                            strength = result.FinalDamage * 0.02f;
                            break;
                        case DamageType.Residue:
                            effectId = "void_sickness_t1";
                            duration = 10f; // TDD 18.4.1 says permanent, but for test, give a duration.
                            strength = result.FinalDamage * 0.01f;
                            break;
                        default:
                            return;
                    }

                    if (!string.IsNullOrEmpty(effectId))
                    {
                        // Now calling the real ApplyEffect method (uncommented)
                        _statusEffectSystem.ApplyEffect(targetID, new StatusEffectData(effectId, duration, strength), result.AttackerID);
                        // GD.Print($"DamageSystem: {targetID} received wound: {effectId} (Strength: {strength:F1})"); // StatusEffectSystem prints this now.
                    }
                }
            }
        }
    }
}
```

### 7. Update `EventBus.cs` for Status Effect Events

Open `res://_Brain/Core/EventBus.cs` and add delegates for `StatusEffectSystem`'s new events.

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.StatusEffects; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Status Effect Events (TDD 18.3.1)
        public event Action<EntityID, ActiveEffect> OnEffectApplied;
        public event Action<EntityID, ActiveEffect, bool> OnEffectUpdated; // TargetID, Effect, IsNew (true if new, false if just stacked/refreshed)
        public event Action<EntityID, string> OnEffectRemoved; // TargetID, EffectID

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is StatusEffectSystem.EffectAppliedEvent effectAppliedEvent) // New condition
            {
                OnEffectApplied?.Invoke(effectAppliedEvent.TargetID, effectAppliedEvent.Effect);
            }
            else if (eventData is StatusEffectSystem.EffectUpdatedEvent effectUpdatedEvent) // New condition
            {
                OnEffectUpdated?.Invoke(effectUpdatedEvent.TargetID, effectUpdatedEvent.Effect, effectUpdatedEvent.IsNew);
            }
            else if (eventData is StatusEffectSystem.EffectRemovedEvent effectRemovedEvent) // New condition
            {
                OnEffectRemoved?.Invoke(effectRemovedEvent.TargetID, effectRemovedEvent.EffectID);
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

### 8. Testing Status Effect Lifecycle

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output. The `DamageSystem` will apply damage and now call `StatusEffectSystem.ApplyEffect`.
    *   You should see messages like:
        *   `StatusEffectSystem: Applied new effect 'bleed_t1' to EntityID(1, Gen:1).`
        *   `StatusEffectSystem: Effect 'bleed_t1' expired on EntityID(1, Gen:1).` (after 5 seconds for bleed_t1)
        *   `StatusEffectSystem: Applied new effect 'burn_t1' to EntityID(1, Gen:1).` (after high elemental damage)
        *   `StatusEffectSystem: Applied new effect 'void_sickness_t1' to EntityID(0, Gen:1).` (after residue damage to player)
        *   You'll also see `DamageOverTime` ticks reducing health, `ChakraDrain` ticks reducing chakra, etc.

This confirms that:
*   `EffectDefinition`s are registered.
*   `DamageSystem` correctly calls `StatusEffectSystem.ApplyEffect()`.
*   `StatusEffectSystem` manages `ActiveEffect` instances, refreshes/stacks them, and ticks their effects.
*   Effects expire after their `TimeRemaining` (unless `IsPermanent`).
*   The `ApplyEffectTick` correctly applies damage, healing, or resource drain.

### Summary

You have successfully implemented the **Status Effect System** for Sigilborne, defining `EffectDefinition` as static blueprints and `ActiveEffect` as runtime instances. By designing `StatusEffectSystem` to manage their lifecycleâ€”from application and stacking/refreshing to periodic ticking and expirationâ€”you've created a robust mechanism for handling buffs, debuffs, and wounds. This crucial system correctly integrates with `DamageSystem` and `BiologicalSystem` to modify `CoreStats`, strictly adhering to TDD 18.2's specifications and laying the groundwork for complex, lasting combat consequences.

### Next Steps

The next chapter will focus on **Recovery Logic**, detailing how entities heal naturally, manage stamina and chakra regeneration, and how these processes interact with status effects and environmental conditions.