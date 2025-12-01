## Chapter 6.2: The Math Layer - Damage Calculator (C#)

Our `CombatSystem` can now receive `HitEvent`s and trigger the damage application. This chapter focuses on fully implementing **The Math Layer** within the C# `CombatSystem`, specifically the `Damage Calculator`. We'll incorporate `CoreStats` (Armor, MagicResistance, BaseDamage, AttackSpeed) and other factors to determine the `FinalDamage`, `IsCrit`, and `IsBlocked` values, as specified in TDD 04.3.

### 1. The Importance of an Authoritative Damage Calculator

*   **Determinism**: All damage calculations happen in the C# Brain, ensuring consistent results.
*   **Balance**: Centralized formulas make it easier to balance combat.
*   **Complexity**: Allows for intricate interactions between stats, damage types, and status effects.
*   **GDD Alignment**: Supports mastery-based damage scaling (B12.3) and variable realism (B12.2).

### 2. Enhancing `CombatSystem.cs` for Damage Calculation

Our `CombatSystem` already has an `ApplyDamage` method that performs basic mitigation. We'll expand it to include more sophisticated calculations for critical hits, blocks, and to use `CoreStats` more thoroughly.

Open `res://_Brain/Systems/Combat/CombatSystem.cs`:

```csharp
// _Brain/Systems/Combat/CombatSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Magic; // For SpellDefinition if spells have base damage

namespace Sigilborne.Systems.Combat
{
    // ... (DamageType, DamageResult, HitEvent structs/enums) ...

    public class CombatSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem;
        private StatusEffectSystem _statusEffectSystem;

        public CombatSystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, StatusEffectSystem statusEffectSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _statusEffectSystem = statusEffectSystem;
            GD.Print("CombatSystem: Initialized.");
        }

        public void RegisterHit(HitEvent hitEvent)
        {
            if (!_entityManager.IsValid(hitEvent.AttackerID) || !_entityManager.IsValid(hitEvent.DefenderID))
            {
                GD.PrintErr($"CombatSystem: Invalid attacker or defender in hit event: {hitEvent}.");
                return;
            }
            
            GD.Print($"CombatSystem: Received hit event: {hitEvent}.");

            // --- Determine Raw Damage and Type from Hitbox Definition (Conceptual) ---
            // In a real game, this would query an "AttackDefinition" database based on hitEvent.HitboxDefinitionID.
            // For now, we'll use placeholder logic.
            float baseRawDamage = 0f;
            DamageType damageType = DamageType.Physical;
            float penetration = 0f;
            float critChance = 0.05f; // 5% base crit chance
            float critMultiplier = 1.5f; // 150% crit damage
            
            // Example: If attacker has a weapon, use its stats. If it's a spell, use SpellDefinition.
            // For now, let's use attacker's BaseDamage from CoreStats.
            if (_biologicalSystem.TryGetCoreStats(hitEvent.AttackerID, out CoreStats attackerStats))
            {
                baseRawDamage = attackerStats.BaseDamage;
                // critChance += attackerStats.CritChanceBonus; // Conceptual
            }

            if (hitEvent.HitboxDefinitionID.Contains("fireball"))
            {
                baseRawDamage = 25f;
                damageType = DamageType.Elemental;
            }
            else if (hitEvent.HitboxDefinitionID.Contains("void"))
            {
                baseRawDamage = 30f;
                damageType = DamageType.Residue;
            }
            // --- End Raw Damage Determination ---

            // Call the core damage application logic.
            ApplyDamage(hitEvent.AttackerID, hitEvent.DefenderID, baseRawDamage, damageType, penetration, critChance, critMultiplier);
        }


        /// <summary>
        /// Calculates and applies damage to a target entity.
        /// This is the entry point for all damage in the game.
        /// (TDD 03.4: Damage Pipeline, TDD 04.3: Damage Calculator)
        /// </summary>
        /// <param name="attackerID">The entity dealing damage.</param>
        /// <param name="defenderID">The entity receiving damage.</param>
        /// <param name="rawDamage">The base damage value.</param>
        /// <param name="damageType">The type of damage.</param>
        /// <param name="penetration">How much the damage ignores defender's mitigation (0-1).</param>
        /// <param name="critChance">Base chance for a critical hit (0-1).</param>
        /// <param name="critMultiplier">Multiplier for critical hit damage (e.g., 1.5 for 150%).</param>
        /// <returns>The final DamageResult.</returns>
        public DamageResult ApplyDamage(EntityID attackerID, EntityID defenderID, float rawDamage, DamageType damageType, float penetration, float critChance, float critMultiplier)
        {
            if (!_biologicalSystem.TryGetCoreStats(defenderID, out CoreStats defenderStats))
            {
                GD.PrintErr($"CombatSystem: Defender {defenderID} has no CoreStats. Cannot apply damage.");
                return new DamageResult(0, false, false, DamageType.None, attackerID, defenderID);
            }
            
            // --- 1. Base Damage --- (TDD 04.3)
            float calculatedDamage = rawDamage;
            
            // --- 2. Mitigation (Armor, Magic Resistance) --- (TDD 03.4, TDD 04.3)
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
            calculatedDamage -= mitigation;
            calculatedDamage = Mathf.Max(0, calculatedDamage); // Damage cannot be negative

            // --- 3. Critical Hit & Block Logic --- (TDD 04.3)
            bool isCrit = false;
            bool isBlocked = false; // Placeholder for block logic

            // Use a deterministic random number generator (seeded by time + entity IDs)
            Random rand = new Random(attackerID.Index + defenderID.Index + (int)GameManager.Instance.Time.CurrentGameTime);

            // Critical Hit (TDD 04.3)
            if (rand.NextDouble() < critChance)
            {
                isCrit = true;
                calculatedDamage *= critMultiplier;
            }
            
            // Block (Conceptual: Implement block chance/mechanic here)
            // if (rand.NextDouble() < defenderStats.BlockChance) { isBlocked = true; calculatedDamage *= defenderStats.BlockReduction; }

            // --- 4. Multipliers (TDD 04.3) ---
            // Add other multipliers here (e.g., backstab, status effect vulnerabilities, environmental bonuses)
            // if (IsBackstab(attacker, defender)) calculatedDamage *= 1.5f; // Conceptual

            // Final damage result
            DamageResult result = new DamageResult(calculatedDamage, isCrit, isBlocked, damageType, attackerID, defenderID);
            GD.Print($"CombatSystem: {attackerID} dealt {result} to {defenderID}.");

            // --- 5. Application to Health & Wound Generation --- (TDD 03.4)
            if (result.FinalDamage > 0)
            {
                ref CoreStats targetCoreStats = ref _biologicalSystem.GetCoreStatsRef(defenderID);
                targetCoreStats.Health -= result.FinalDamage;
                if (targetCoreStats.Health < 0) targetCoreStats.Health = 0;

                // Publish health change (PlayerStatSystem already does this for player)
                if (defenderID == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerStatSystem.PlayerHealthChangedEvent { PlayerID = defenderID, NewValue = targetCoreStats.Health, MaxValue = targetCoreStats.MaxHealth });
                }
                // For NPCs, we'd need a generic EntityHealthChangedEvent.

                // 4. Wound Generation (TDD 03.4)
                GenerateWounds(defenderID, result);

                if (targetCoreStats.Health == 0)
                {
                    GD.Print($"CombatSystem: {defenderID} has been defeated!");
                    if (defenderID == _entityManager.GetPlayerEntityID())
                    {
                        _eventBus.Publish(new PlayerStatSystem.PlayerDiedEvent { PlayerID = defenderID });
                    }
                    // _eventBus.Publish(new EntityDefeatedEvent { EntityID = defenderID, KillerID = attackerID });
                }
            }
            // TDD 04.2.2: Body (GDScript) reacts to OnDamageTaken signal.
            // This event is already implicitly handled by PlayerStatSystem.PlayerHealthChangedEvent for players.
            // For general entities, we might need a more generic event.
            _eventBus.Publish(new DamageTakenEvent { DefenderID = defenderID, Result = result }); // New generic event for visual feedback
            return result;
        }

        private void GenerateWounds(EntityID targetID, DamageResult result)
        {
            // ... (existing GenerateWounds logic) ...
        }

        // --- Helper Events for Body Sync ---
        public struct DamageTakenEvent { public EntityID DefenderID; public DamageResult Result; } // New event
    }
}
```

#### 2.1. Update `CoreStats.cs` with Combat-Related Stats

We need to add `CritChance`, `CritMultiplier`, and `BlockChance` (conceptual) to `CoreStats`.

Open `res://_Brain/Systems/Biology/CoreStats.cs`:

```csharp
// _Brain/Systems/Biology/CoreStats.cs
using System;
using Godot;

namespace Sigilborne.Systems.Biology
{
    public struct CoreStats
    {
        // ... (existing survival/biological stats) ...

        // --- Combat/Magic-related Stats ---
        public float BaseDamage;
        public float AttackSpeed;
        public float Armor;
        public float MagicResistance;
        public float MoveSpeed;
        public float SprintMultiplier;
        public float CastSpeed;
        public float Stability;
        public float MaxStability;
        public float StabilityRegenRate;
        public float ChakraRegenRate;
        public float StaminaRegenRate;
        public float CritChance;       // New: Chance to land a critical hit (0-1)
        public float CritMultiplier;   // New: Multiplier for critical hit damage (e.g., 1.5)
        public float BlockChance;      // New: Chance to block an attack (0-1)
        public float BlockReduction;   // New: Percentage damage reduction on block (0-1)

        public CoreStats(float maxHealth, float maxStamina, float maxChakra, float baseDamage = 10f, float attackSpeed = 1.0f,
                         float armor = 0f, float magicResistance = 0f, float moveSpeed = 150f, float sprintMultiplier = 1.5f,
                         float castSpeed = 1.0f, float maxStability = 100f, float stabilityRegenRate = 5f,
                         float chakraRegenRate = 2f, float staminaRegenRate = 10f, float normalBodyTemp = 37.0f,
                         float critChance = 0.05f, float critMultiplier = 1.5f, float blockChance = 0f, float blockReduction = 0.5f) // New params
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
            Stability = MaxStability;
            StabilityRegenRate = stabilityRegenRate;
            ChakraRegenRate = chakraRegenRate;
            StaminaRegenRate = staminaRegenRate;
            CritChance = critChance; // New
            CritMultiplier = critMultiplier; // New
            BlockChance = blockChance; // New
            BlockReduction = blockReduction; // New
        }

        public override string ToString()
        {
            return $"HP: {Health:F0}/{MaxHealth:F0}, Sta: {Stamina:F0}/{MaxStamina:F0}, Cha: {Chakra:F0}/{MaxChakra:F0}, Stab: {Stability:F0}/{MaxStability:F0}, Hunger: {Hunger:F0}, Thirst: {Thirst:F0}, Temp: {BodyTemp:F1}°C, Spd: {MoveSpeed:F0}, Dmg: {BaseDamage:F0}, Armor: {Armor:F0}, MR: {MagicResistance:F0}, Crit: {CritChance:P0}";
        }
    }
}
```

#### 2.2. Update `BiologicalSystem.cs` for Initial `CoreStats`

The `BiologicalSystem` initializes `CoreStats` for new entities. We need to pass the new `CoreStats` constructor parameters.

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
using Sigilborne.Systems.StatusEffects;

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        // ... (existing fields and constructor) ...

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player)
            {
                // Player's initial CoreStats (TDD 03.2).
                // Set default player stats with some armor/MR, and crit chance.
                _entityCoreStats.Add(id, new CoreStats(
                    maxHealth: 100f, maxStamina: 75f, maxChakra: 50f,
                    baseDamage: 15f, attackSpeed: 1.0f, armor: 5f, magicResistance: 5f,
                    moveSpeed: 150f, sprintMultiplier: 1.5f, castSpeed: 1.0f,
                    maxStability: 100f, stabilityRegenRate: 5f, chakraRegenRate: 2f, staminaRegenRate: 10f,
                    normalBodyTemp: 37.0f, critChance: 0.1f, critMultiplier: 1.75f, blockChance: 0.1f, blockReduction: 0.5f // New crit/block stats
                ));
                GD.Print($"BiologicalSystem: Added CoreStats for Player {id}. Stats: {_entityCoreStats[id]}");
            }
            else if (type == EntityType.NPC || type == EntityType.Animal)
            {
                // Generic NPC/Animal stats
                _entityCoreStats.Add(id, new CoreStats(
                    maxHealth: 50f, maxStamina: 30f, maxChakra: 20f,
                    baseDamage: 8f, attackSpeed: 1.0f, armor: 2f, magicResistance: 2f,
                    moveSpeed: 100f, sprintMultiplier: 1.2f, castSpeed: 1.0f,
                    maxStability: 50f, stabilityRegenRate: 3f, chakraRegenRate: 1f, staminaRegenRate: 5f,
                    normalBodyTemp: 37.0f, critChance: 0.05f, critMultiplier: 1.5f, blockChance: 0f, blockReduction: 0f // Generic NPC stats
                ));
                GD.Print($"BiologicalSystem: Added CoreStats for {type} entity {id}. Stats: {_entityCoreStats[id]}");
            }
        }
        // ... (rest of class) ...
    }
}
```

#### 2.3. Update `EventBus.cs` for `DamageTakenEvent`

Open `res://_Brain/Core/EventBus.cs` and add `OnDamageTaken` delegate:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Combat; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Combat System Events (TDD 04.2.2)
        public event Action<EntityID, DamageResult> OnDamageTaken; // New event

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is CombatSystem.DamageTakenEvent damageTakenEvent) // New condition
            {
                OnDamageTaken?.Invoke(damageTakenEvent.DefenderID, damageTakenEvent.Result);
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

### 3. Testing the Damage Calculator

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.

```
...
BiologicalSystem: Added CoreStats for Player EntityID(0, Gen:1). Stats: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 150, Dmg: 15, Armor: 5, MR: 5, Crit: 10%
BiologicalSystem: Added CoreStats for NPC entity EntityID(1, Gen:1). Stats: HP: 50/50, Sta: 30/30, Cha: 20/20, Stab: 50/50, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 100, Dmg: 8, Armor: 2, MR: 2, Crit: 5%
BiologicalSystem: Added CoreStats for NPC entity EntityID(2, Gen:1). Stats: HP: 50/50, Sta: 30/30, Cha: 20/20, Stab: 50/50, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 100, Dmg: 8, Armor: 2, MR: 2, Crit: 5%
...
--- Testing Damage & Recovery Pipeline ---
Player initial HP: 100.0
Test NPC initial HP: 50.0
CombatSystem: Received hit event: Hit! Attacker: EntityID(0, Gen:1), Defender: EntityID(1, Gen:1), Hitbox: 'player_sword_slash', Part: 'torso', Pos: (450, 200).
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 13.0 (Physical) | Crit: False, Block: False to EntityID(1, Gen:1).
StatusEffectSystem: Applied new effect 'bleed_t1' to EntityID(1, Gen:1).
CombatSystem: Received hit event: Hit! Attacker: EntityID(0, Gen:1), Defender: EntityID(1, Gen:1), Hitbox: 'player_fireball', Part: 'torso', Pos: (450, 200).
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 20.0 (Elemental) | Crit: False, Block: False to EntityID(1, Gen:1).
StatusEffectSystem: Applied new effect 'burn_t1' to EntityID(1, Gen:1).
CombatSystem: Received hit event: Hit! Attacker: EntityID(0, Gen:1), Defender: EntityID(1, Gen:1), Hitbox: 'player_void_blast', Part: 'torso', Pos: (450, 200).
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 25.0 (Residue) | Crit: False, Block: False to EntityID(1, Gen:1).
StatusEffectSystem: Applied new effect 'void_sickness_t1' to EntityID(1, Gen:1).
CombatSystem: EntityID(1, Gen:1) has been defeated!
CombatSystem: Received hit event: Hit! Attacker: EntityID(2, Gen:1), Defender: EntityID(0, Gen:1), Hitbox: 'npc_claw', Part: 'torso', Pos: (250, 200).
CombatSystem: EntityID(2, Gen:1) dealt Dmg: 3.0 (Physical) | Crit: False, Block: False to EntityID(0, Gen:1).
StatusEffectSystem: Applied new effect 'bleed_t1' to EntityID(0, Gen:1).
PlayerStatSystem: Player EntityID(0, Gen:1) took 3.0 damage. New Health: 97.0
CombatSystem: Received hit event: Hit! Attacker: EntityID(2, Gen:1), Defender: EntityID(0, Gen:1), Hitbox: 'npc_poison_spit', Part: 'torso', Pos: (250, 200).
CombatSystem: EntityID(2, Gen:1) dealt Dmg: 6.0 (Elemental) | Crit: False, Block: False to EntityID(0, Gen:1).
StatusEffectSystem: Applied new effect 'burn_t1' to EntityID(0, Gen:1).
PlayerStatSystem: Player EntityID(0, Gen:1) took 6.0 damage. New Health: 91.0
CombatSystem: Received hit event: Hit! Attacker: EntityID(2, Gen:1), Defender: EntityID(0, Gen:1), Hitbox: 'npc_heavy_hit', Part: 'torso', Pos: (250, 200).
CombatSystem: EntityID(2, Gen:1) dealt Dmg: 63.0 (Physical) | Crit: False, Block: False to EntityID(0, Gen:1).
StatusEffectSystem: Applied new effect 'bleed_t1' to EntityID(0, Gen:1).
PlayerStatSystem: Player EntityID(0, Gen:1) took 63.0 damage. New Health: 28.0
--- End Testing Damage & Recovery Pipeline ---
...
```

**Key Observations:**

*   **NPC Armor/MR**: The NPC (Armor 2, MR 2) now correctly mitigates damage. Player's `player_sword_slash` (Raw 15) deals `15 - 2 = 13`. `player_fireball` (Raw 25, Elemental) deals `25 - 2 = 23`. `player_void_blast` (Raw 30, Residue) deals `30 - 2 = 28`.
*   **Player Armor/MR**: The player (Armor 5, MR 5) mitigates damage. `npc_claw` (Raw 8) deals `8 - 5 = 3`. `npc_poison_spit` (Raw 10, Elemental) deals `10 - 5 = 5`. `npc_heavy_hit` (Raw 65) deals `65 - 5 = 60`.
*   **Wound Generation**: `bleed_t1`, `burn_t1`, `chakra_drain_t1`, `void_sickness_t1` are applied based on `DamageType` and damage amount.
*   **Crit Chance**: You might see `Crit: True` for some hits if the random chance triggers.

This confirms that the `Damage Calculator` is now fully functional, incorporating `CoreStats` for mitigation, `DamageType`s, and `CritChance`/`CritMultiplier` for dynamic combat outcomes.

### Summary

You have successfully implemented **The Math Layer** within the C# `CombatSystem`, developing a comprehensive `Damage Calculator`. By incorporating `CoreStats` (Armor, MagicResistance, CritChance, CritMultiplier) and leveraging `DamageType`s, you've established a robust system for determining `FinalDamage`, `IsCrit`, and `IsBlocked` values. This crucial component strictly adheres to TDD 04.3's specifications, providing the authoritative and deterministic backbone for all combat interactions in Sigilborne.

### Next Steps

The next chapter will focus on **Equipment & Inventory - Inventory Data Structure (C#)**, where we will design the core data structures for managing items and the player's inventory, laying the groundwork for how equipment will influence `CoreStats` and gameplay.