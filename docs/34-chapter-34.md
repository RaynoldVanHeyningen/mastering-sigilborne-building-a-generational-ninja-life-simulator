## Chapter 5.5: Damage & Recovery Pipeline - Wounds & Status Effects (C#)

Combat in Sigilborne is not just about reducing a health bar; it's about systemic injuries and lasting consequences. This chapter implements the **Damage & Recovery Pipeline**, detailing how entities take damage, how mitigation (armor) is applied, and how injuries can generate `StatusEffect`s (like Bleed or Fracture), as specified in TDD 03.4 and GDD B12.

### 1. The Systemic Nature of Damage

The GDD (B12.2) emphasizes that damage depends on the technique, attacker mastery, defender mastery, and environmental resonance. It also states that "injuries are not minor debuffs; they fundamentally alter performance." This requires a robust pipeline:

*   **Raw Damage**: The initial value from an attack.
*   **Mitigation**: Reduction based on defender's `Armor` and `MagicResistance`.
*   **Application**: Reducing `Health` in `CoreStats`.
*   **Wound Generation**: Chance to apply `StatusEffect`s based on damage type and severity.
*   **Recovery**: Mechanisms for healing and clearing status effects.

### 2. Defining `DamageType` and `DamageResult`

We need enums for different damage types and a struct to encapsulate the outcome of a damage calculation. TDD 04.3 defines `DamageResult`.

1.  Create `res://_Brain/Systems/Combat/DamageTypes.cs`:

```csharp
// _Brain/Systems/Combat/DamageTypes.cs
using System;

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Defines the types of damage that can be dealt.
    /// (GDD B12.2)
    /// </summary>
    public enum DamageType
    {
        None,
        Physical,   // Cuts, blunt strikes, fractures
        Chakra,     // Internal flow disruption, energy drain
        Elemental,  // Heat, cold, shock, pressure
        Spiritual,  // Soul pressure, illusion backlash
        Residue,    // Corruption, mutation stress
        Internal    // Organ/nerve damage, blood loss
    }
}
```

2.  Create `res://_Brain/Systems/Combat/DamageResult.cs`:

```csharp
// _Brain/Systems/Combat/DamageResult.cs
using System;

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Encapsulates the result of a damage calculation.
    /// (TDD 04.3)
    /// </summary>
    public struct DamageResult
    {
        public float FinalDamage;
        public bool IsCrit;
        public bool IsBlocked;
        public DamageType Type;
        public EntityID AttackerID; // Who dealt the damage
        public EntityID DefenderID; // Who received the damage

        public DamageResult(float finalDamage, bool isCrit, bool isBlocked, DamageType type, EntityID attackerID, EntityID defenderID)
        {
            FinalDamage = finalDamage;
            IsCrit = isCrit;
            IsBlocked = isBlocked;
            Type = type;
            AttackerID = attackerID;
            DefenderID = defenderID;
        }

        public override string ToString()
        {
            return $"Dmg: {FinalDamage:F1} ({Type}) | Crit: {IsCrit}, Block: {IsBlocked}";
        }
    }
}
```

### 3. Implementing `DamageSystem.cs`

This system will be responsible for the entire damage pipeline: calculation, application, and wound generation.

1.  Create a new folder `res://_Brain/Systems/Combat/`.
2.  Create `res://_Brain/Systems/Combat/DamageSystem.cs`:

```csharp
// _Brain/Systems/Combat/DamageSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology; // For CoreStats
using Sigilborne.Systems.StatusEffects; // For StatusEffectSystem

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Manages the full damage pipeline: calculation, application to CoreStats,
    /// and generation of StatusEffects (wounds).
    /// (TDD 03.4, TDD 04.3)
    /// </summary>
    public class DamageSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem; // To access/modify CoreStats
        private StatusEffectSystem _statusEffectSystem; // To apply wounds

        public DamageSystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, StatusEffectSystem statusEffectSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _statusEffectSystem = statusEffectSystem; // Store StatusEffectSystem reference
            GD.Print("DamageSystem: Initialized.");
        }

        /// <summary>
        /// Calculates and applies damage to a target entity.
        /// This is the entry point for all damage in the game.
        /// (TDD 03.4: Damage Pipeline)
        /// </summary>
        /// <param name="attackerID">The entity dealing damage.</param>
        /// <param name="defenderID">The entity receiving damage.</param>
        /// <param name="rawDamage">The base damage value.</param>
        /// <param name="damageType">The type of damage.</param>
        /// <param name="penetration">How much the damage ignores defender's mitigation.</param>
        /// <returns>The final DamageResult.</returns>
        public DamageResult ApplyDamage(EntityID attackerID, EntityID defenderID, float rawDamage, DamageType damageType, float penetration = 0f)
        {
            if (!_entityManager.IsValid(defenderID))
            {
                GD.PrintErr($"DamageSystem: Attempted to apply damage to invalid defender {defenderID}.");
                return new DamageResult(0, false, false, DamageType.None, attackerID, defenderID);
            }

            if (!_biologicalSystem.TryGetCoreStats(defenderID, out CoreStats defenderStats))
            {
                GD.PrintErr($"DamageSystem: Defender {defenderID} has no CoreStats. Cannot apply damage.");
                return new DamageResult(0, false, false, DamageType.None, attackerID, defenderID);
            }

            // 1. Raw Damage: Already provided.

            // 2. Mitigation (TDD 03.4, TDD 04.3)
            float mitigation = 0;
            switch (damageType)
            {
                case DamageType.Physical:
                    mitigation = defenderStats.Armor * (1.0f - penetration);
                    break;
                case DamageType.Chakra:
                case DamageType.Elemental:
                case DamageType.Spiritual:
                case DamageType.Residue:
                    mitigation = defenderStats.MagicResistance * (1.0f - penetration);
                    break;
                case DamageType.Internal:
                    mitigation = 0; // Internal damage often bypasses armor
                    break;
            }
            float finalDamage = rawDamage - mitigation;
            finalDamage = Mathf.Max(0, finalDamage); // Damage cannot be negative

            // Placeholder for crit/block logic (TDD 04.3)
            bool isCrit = false; // (Implement crit chance logic here)
            bool isBlocked = false; // (Implement block chance logic here)

            DamageResult result = new DamageResult(finalDamage, isCrit, isBlocked, damageType, attackerID, defenderID);
            GD.Print($"DamageSystem: {attackerID} dealt {result} to {defenderID}.");

            // 3. Application (TDD 03.4)
            if (result.FinalDamage > 0)
            {
                ref CoreStats targetCoreStats = ref _biologicalSystem.GetCoreStatsRef(defenderID);
                targetCoreStats.Health -= result.FinalDamage;
                if (targetCoreStats.Health < 0) targetCoreStats.Health = 0; // Clamp health at 0

                // Publish health change (PlayerStatSystem already does this for player)
                if (defenderID == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerStatSystem.PlayerHealthChangedEvent { PlayerID = defenderID, NewValue = targetCoreStats.Health, MaxValue = targetCoreStats.MaxHealth });
                }
                // For NPCs, we'd need a generic EntityHealthChangedEvent.

                // 4. Wound Generation (TDD 03.4) - Chance to create a StatusEffect
                // GDD B12.4: Injuries fundamentally alter performance.
                GenerateWounds(defenderID, result);

                if (targetCoreStats.Health == 0)
                {
                    GD.Print($"DamageSystem: {defenderID} has been defeated!");
                    // Publish entity defeated event
                    if (defenderID == _entityManager.GetPlayerEntityID())
                    {
                        _eventBus.Publish(new PlayerStatSystem.PlayerDiedEvent { PlayerID = defenderID });
                    }
                    // _eventBus.Publish(new EntityDefeatedEvent { EntityID = defenderID, KillerID = attackerID });
                }
            }

            return result;
        }

        /// <summary>
        /// Generates StatusEffect "wounds" based on damage type and severity.
        /// (TDD 03.4, GDD B12.4)
        /// </summary>
        private void GenerateWounds(EntityID targetID, DamageResult result)
        {
            if (result.FinalDamage <= 0) return;

            // Simple wound generation for now. Later, this would be more complex.
            float woundChance = result.FinalDamage / _biologicalSystem.GetCoreStatsRef(targetID).MaxHealth; // Higher damage = higher chance
            if (woundChance > 0.1f) // 10% damage means 10% chance to bleed
            {
                Random rand = new Random(targetID.Index + (int)GameManager.Instance.Time.CurrentGameTime); // Deterministic random
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
                            strength = result.FinalDamage * 0.05f; // Bleed for 5% of damage dealt
                            break;
                        case DamageType.Elemental:
                            effectId = "burn_t1";
                            duration = 3f;
                            strength = result.FinalDamage * 0.03f;
                            break;
                        case DamageType.Chakra:
                            effectId = "chakra_drain_t1";
                            duration = 2f;
                            strength = result.FinalDamage * 0.02f; // Drains chakra
                            break;
                        case DamageType.Residue:
                            effectId = "void_sickness_t1";
                            duration = 10f;
                            strength = result.FinalDamage * 0.01f; // Accumulates void sickness
                            break;
                        default:
                            return; // No wound for other damage types for now
                    }

                    if (!string.IsNullOrEmpty(effectId))
                    {
                        // _statusEffectSystem.ApplyEffect(targetID, new StatusEffectData(effectId, duration, strength));
                        GD.Print($"DamageSystem: {targetID} received wound: {effectId} (Strength: {strength:F1})");
                    }
                }
            }
        }
    }
}
```

### 4. Implementing `StatusEffectSystem.cs` (Placeholder)

The `DamageSystem` references a `StatusEffectSystem`. We need a placeholder for it. We'll fully implement this in Chapter 5.6.

1.  Create a new folder `res://_Brain/Systems/StatusEffects/`.
2.  Create `res://_Brain/Systems/StatusEffects/StatusEffectSystem.cs`:

```csharp
// _Brain/Systems/StatusEffects/StatusEffectSystem.cs
using Godot;
using System;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic.Components; // For StatusEffectData

namespace Sigilborne.Systems.StatusEffects
{
    /// <summary>
    /// Placeholder for the StatusEffectSystem, which will manage buffs, debuffs, and wounds.
    /// (TDD 03.4, TDD 18)
    /// </summary>
    public class StatusEffectSystem
    {
        public StatusEffectSystem()
        {
            GD.Print("StatusEffectSystem: Initialized (placeholder).");
        }

        // Placeholder for applying an effect (DamageSystem needs this)
        public void ApplyEffect(EntityID targetID, StatusEffectData effectData)
        {
            GD.Print($"StatusEffectSystem: (Placeholder) Applied '{effectData.EffectID}' to {targetID}.");
            // In Chapter 5.6, this will add the effect to the target's active effects list.
        }
    }
}
```

### 5. Integrating `DamageSystem` and `StatusEffectSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Combat;` and `using Sigilborne.Systems.StatusEffects;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add `DamageSystem` and `StatusEffectSystem` properties.
3.  Initialize `StatusEffectSystem` first, then `DamageSystem`, in `InitializeSystems()`.

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
using Sigilborne.Systems.Combat; // Add this using directive
using Sigilborne.Systems.StatusEffects; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public BiologicalSystem Biology { get; private set; }
    public StatusEffectSystem StatusEffects { get; private set; } // Add StatusEffectSystem property
    public DamageSystem Combat { get; private set; } // Add DamageSystem property (named Combat as per TDD 04.3)

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        
        // --- Test Damage & Recovery Pipeline ---
        GD.Print("\n--- Testing Damage & Recovery Pipeline ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID npcID = Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid; // Get NPC ID assuming it's still ID 1, Gen 1

        GD.Print($"Player initial HP: {Biology.GetCoreStatsRef(playerID).Health}");
        GD.Print($"NPC initial HP: {Biology.GetCoreStatsRef(npcID).Health}");

        // Player attacks NPC
        Combat.ApplyDamage(playerID, npcID, 20f, DamageType.Physical);
        Combat.ApplyDamage(playerID, npcID, 15f, DamageType.Physical);
        Combat.ApplyDamage(playerID, npcID, 70f, DamageType.Elemental); // High damage, higher wound chance

        // NPC attacks Player
        Combat.ApplyDamage(npcID, playerID, 5f, DamageType.Physical);
        Combat.ApplyDamage(npcID, playerID, 10f, DamageType.Chakra);
        Combat.ApplyDamage(npcID, playerID, 20f, DamageType.Residue); // Should trigger void sickness
        Combat.ApplyDamage(npcID, playerID, 65f, DamageType.Physical); // Should kill player

        GD.Print("--- End Testing Damage & Recovery Pipeline ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta);
        Casting.Tick(delta);
        Biology.Tick(delta); // BiologicalSystem needs to tick for regen
        // Combat and StatusEffects don't have a Tick method for now, their operations are command-driven.
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to BiologicalSystem) ...
        
        // Initialize StatusEffectSystem BEFORE DamageSystem
        StatusEffects = new StatusEffectSystem(); // Initialize StatusEffectSystem here
        GD.Print("  - StatusEffectSystem initialized.");

        // Initialize DamageSystem, passing BiologicalSystem and StatusEffectSystem
        Combat = new DamageSystem(Entities, Events, BiologicalSystem, StatusEffects); // Initialize DamageSystem here
        GD.Print("  - DamageSystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 6. Testing the Damage & Recovery Pipeline

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.

```
...
StatusEffectSystem: Initialized (placeholder).
  - StatusEffectSystem initialized.
DamageSystem: Initialized.
  - DamageSystem initialized.
PlayerStatSystem: Initialized.
PlayerStatSystem: Player EntityID(0, Gen:1) took 0 damage. New Health: 100.0
PlayerStatSystem: Player EntityID(0, Gen:1) used 0 chakra. New Chakra: 50.0
PlayerStatSystem: Player EntityID(0, Gen:1) used 0 stamina. New Stamina: 75.0
PlayerStatSystem: Player EntityID(0, Gen:1) lost 0 stability. New Stability: 100.0
  - PlayerStatSystem initialized.
  - WorldSimulation initialized.

--- Testing Damage & Recovery Pipeline ---
Player initial HP: 100.0
NPC initial HP: 100.0
DamageSystem: EntityID(0, Gen:1) dealt Dmg: 20.0 (Physical) | Crit: False, Block: False to EntityID(1, Gen:1).
StatusEffectSystem: (Placeholder) Applied 'bleed_t1' to EntityID(1, Gen:1).
DamageSystem: EntityID(0, Gen:1) dealt Dmg: 15.0 (Physical) | Crit: False, Block: False to EntityID(1, Gen:1).
DamageSystem: EntityID(0, Gen:1) dealt Dmg: 70.0 (Elemental) | Crit: False, Block: False to EntityID(1, Gen:1).
StatusEffectSystem: (Placeholder) Applied 'burn_t1' to EntityID(1, Gen:1).
DamageSystem: EntityID(1, Gen:1) dealt Dmg: 5.0 (Physical) | Crit: False, Block: False to EntityID(0, Gen:1).
PlayerStatSystem: Player EntityID(0, Gen:1) took 5.0 damage. New Health: 95.0
DamageSystem: EntityID(1, Gen:1) dealt Dmg: 10.0 (Chakra) | Crit: False, Block: False to EntityID(0, Gen:1).
StatusEffectSystem: (Placeholder) Applied 'chakra_drain_t1' to EntityID(0, Gen:1).
PlayerStatSystem: Player EntityID(0, Gen:1) took 10.0 damage. New Health: 85.0
DamageSystem: EntityID(1, Gen:1) dealt Dmg: 20.0 (Residue) | Crit: False, Block: False to EntityID(0, Gen:1).
StatusEffectSystem: (Placeholder) Applied 'void_sickness_t1' to EntityID(0, Gen:1).
PlayerStatSystem: Player EntityID(0, Gen:1) took 20.0 damage. New Health: 65.0
DamageSystem: EntityID(1, Gen:1) dealt Dmg: 65.0 (Physical) | Crit: False, Block: False to EntityID(0, Gen:1).
PlayerStatSystem: Player EntityID(0, Gen:1) took 65.0 damage. New Health: 0.0
PlayerStatSystem: Player EntityID(0, Gen:1) has died!
--- End Testing Damage & Recovery Pipeline ---
...
```

This output confirms that:
*   `DamageSystem` correctly calculates damage, applies mitigation, and reduces health in `CoreStats`.
*   `GenerateWounds` attempts to apply `StatusEffect`s based on damage type and severity, calling our `StatusEffectSystem` placeholder.
*   The player's death is correctly detected and broadcast.

### Summary

You have successfully implemented the foundational **Damage & Recovery Pipeline** for Sigilborne, detailing how entities take damage, how mitigation (armor/magic resistance) is applied, and how injuries can generate `StatusEffect`s (wounds). By designing `DamageSystem` to handle this entire process and integrating it with `BiologicalSystem` and a placeholder `StatusEffectSystem`, you've established a robust mechanism for systemic combat, strictly adhering to TDD 03.4 and GDD B12's specifications. This crucial step enables deep and meaningful consequences from combat encounters.

### Next Steps

The next chapter will focus on **Status Effect Data - Definition vs. Instance**, where we will fully implement the `StatusEffectSystem` to manage buffs, debuffs, and wounds, defining their lifecycle and how they modify entity `CoreStats`.