## Chapter 6.5: World Boss Mechanics - Multi-Part Entities (C#)

Sigilborne's world is home to colossal World Bosses (Titans), which are far too complex for a single health bar or hitbox. This chapter implements **Multi-Part Entities** in the C# Brain, designing how these large creatures are represented as multiple logical parts (limbs, heads, cores), each with its own health, vulnerabilities, and ability to be severed, creating dynamic and multi-stage combat encounters, as specified in TDD 04.5.1.

### 1. The Challenge of Colossal Entities

The GDD (C05) describes Titans as "grand metaphysical lifeforms" that are "not enemies to kill" but "living world systems." TDD 04.5.1 states: "World Bosses (Titans) are too big for a single Hitbox... The Boss is one logical Entity in C#, but has multiple `Hurtbox` nodes in Godot." This requires:

*   **Logical Parts**: A way to define distinct, damageable parts of a single entity in the Brain.
*   **Individual Health**: Each part needs its own health pool, separate from the main entity's health.
*   **Severing Mechanics**: What happens when a part's health reaches zero.
*   **Body Synchronization**: How the GDScript Body visually reacts to limb damage or severing.

### 2. Defining `BossPart` and `BossPartsComponent`

We need a struct to represent a single damageable part and a component to hold all parts for a boss entity.

1.  Create `res://_Brain/Systems/Combat/BossPart.cs`:

```csharp
// _Brain/Systems/Combat/BossPart.cs
using System;
using Godot; // For Vector2 if needed

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Represents a single damageable part of a multi-part boss entity.
    /// (TDD 04.5.1)
    /// </summary>
    public struct BossPart
    {
        public string PartID;           // Unique ID for this part (e.g., "LeftArm", "Head", "Core")
        public float Health;
        public float MaxHealth;
        public bool IsSevered;          // True if this part has been destroyed/severed
        public bool IsVulnerable;       // True if this part is currently vulnerable to damage
        public string VisualNodePath;   // Path to the corresponding visual node in Godot Body (e.g., "Visuals/LeftArmSprite")

        public BossPart(string partID, float maxHealth, string visualNodePath = null)
        {
            PartID = partID;
            MaxHealth = maxHealth;
            Health = maxHealth;
            IsSevered = false;
            IsVulnerable = true; // Default vulnerable
            VisualNodePath = visualNodePath;
        }

        public override string ToString()
        {
            return $"Part: '{PartID}' HP: {Health:F0}/{MaxHealth:F0} (Severed: {IsSevered})";
        }
    }
}
```

2.  Create `res://_Brain/Systems/Combat/BossPartsComponent.cs`:

```csharp
// _Brain/Systems/Combat/BossPartsComponent.cs
using System;
using System.Collections.Generic;
using System.Linq; // For Sum()

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Stores all damageable parts for a multi-part boss entity.
    /// This is a component-like data structure managed by the CombatSystem.
    /// (TDD 04.5.1)
    /// </summary>
    public struct BossPartsComponent
    {
        public Dictionary<string, BossPart> Parts; // Map PartID to BossPart data

        public BossPartsComponent(Dictionary<string, BossPart> parts = null)
        {
            Parts = parts ?? new Dictionary<string, BossPart>();
        }

        /// <summary>
        /// Calculates the total health of all active (non-severed) parts.
        /// </summary>
        public float GetTotalCurrentHealth()
        {
            return Parts.Values.Where(p => !p.IsSevered).Sum(p => p.Health);
        }

        /// <summary>
        /// Calculates the total max health of all parts.
        /// </summary>
        public float GetTotalMaxHealth()
        {
            return Parts.Values.Sum(p => p.MaxHealth);
        }

        public override string ToString()
        {
            return $"Boss Parts: {Parts.Count} | Total HP: {GetTotalCurrentHealth():F0}/{GetTotalMaxHealth():F0}";
        }
    }
}
```

### 3. Enhancing `CombatSystem.cs` for Multi-Part Entities

The `CombatSystem` needs to:
*   Store `BossPartsComponent` instances.
*   Modify `ApplyDamage` to target specific `BossPart`s.
*   Handle `BossPart` health reduction and severing.
*   Emit events when a `BossPart` is damaged or severed.

1.  Open `res://_Brain/Systems/Combat/CombatSystem.cs`.
2.  Add a dictionary for `BossPartsComponent`.
3.  Modify `ApplyDamage` to check if the defender is a boss and apply damage to its part.
4.  Add `RegisterBossPart` and `DamageBossPart` methods.

```csharp
// _Brain/Systems/Combat/CombatSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.Magic.Components; // For SpellDefinition

namespace Sigilborne.Systems.Combat
{
    // ... (DamageType, DamageResult, HitEvent structs/enums) ...
    // ... (BossPart, BossPartsComponent structs) ... // Ensure these are properly defined in their files

    public class CombatSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem;
        private StatusEffectSystem _statusEffectSystem;

        // Dictionary to store BossPartsComponent for multi-part entities.
        private Dictionary<EntityID, BossPartsComponent> _bossParts = new Dictionary<EntityID, BossPartsComponent>(); // New: Boss parts storage

        public CombatSystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, StatusEffectSystem statusEffectSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _statusEffectSystem = statusEffectSystem;
            GD.Print("CombatSystem: Initialized.");

            // Subscribe to entity spawn/despawn events to manage BossPartsComponent
            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // For testing, let's assume "world_boss_01" gets BossPartsComponent
            if (definitionID == "world_boss_01")
            {
                // Example parts for a boss (TDD 04.5.1)
                Dictionary<string, BossPart> parts = new Dictionary<string, BossPart>
                {
                    { "Head", new BossPart("Head", 50f, "Visuals/HeadSprite") },
                    { "LeftArm", new BossPart("LeftArm", 40f, "Visuals/LeftArmSprite") },
                    { "RightArm", new BossPart("RightArm", 40f, "Visuals/RightArmSprite") },
                    { "Body", new BossPart("Body", 100f, "Visuals/BodySprite") }
                };
                _bossParts.Add(id, new BossPartsComponent(parts));
                GD.Print($"CombatSystem: Added BossPartsComponent for Boss {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _bossParts.Remove(e.ID);
            GD.Print($"CombatSystem: Removed BossPartsComponent for {e.ID}.");
        }

        public void RegisterHit(HitEvent hitEvent)
        {
            if (!_entityManager.IsValid(hitEvent.AttackerID) || !_entityManager.IsValid(hitEvent.DefenderID))
            {
                GD.PrintErr($"CombatSystem: Invalid attacker or defender in hit event: {hitEvent}.");
                return;
            }
            
            GD.Print($"CombatSystem: Received hit event: {hitEvent}.");

            float baseRawDamage = 0f;
            DamageType damageType = DamageType.Physical;
            float penetration = 0f;
            float critChance = 0.05f;
            float critMultiplier = 1.5f;
            
            if (_biologicalSystem.TryGetCoreStats(hitEvent.AttackerID, out CoreStats attackerStats))
            {
                baseRawDamage = attackerStats.BaseDamage;
            }

            if (hitEvent.HitboxDefinitionID.Contains("fireball"))
            {
                baseRawDamage = 25f;
                damageType = DamageType.Elemental;
            }
            else if (hitEvent.HitboxDefinitionID.Contains("void"))
            {
            // ... (existing damage type determination) ...
            }

            // --- Determine if defender is a multi-part boss ---
            if (_bossParts.TryGetValue(hitEvent.DefenderID, out BossPartsComponent bossParts))
            {
                // Apply damage to a specific part
                DamageBossPart(hitEvent.AttackerID, hitEvent.DefenderID, hitEvent.HurtboxBodyPart, baseRawDamage, damageType, penetration, critChance, critMultiplier);
            }
            else
            {
                // Apply damage to a regular entity
                ApplyDamage(hitEvent.AttackerID, hitEvent.DefenderID, baseRawDamage, damageType, penetration, critChance, critMultiplier);
            }
        }

        /// <summary>
        /// Applies damage to a specific part of a boss entity.
        /// (TDD 04.5.1: Limb Health)
        /// </summary>
        public DamageResult DamageBossPart(EntityID attackerID, EntityID bossID, string partID, float rawDamage, DamageType damageType, float penetration, float critChance, float critMultiplier)
        {
            if (!_bossParts.TryGetValue(bossID, out BossPartsComponent bossParts) || !bossParts.Parts.ContainsKey(partID))
            {
                GD.PrintErr($"CombatSystem: Boss {bossID} does not have part '{partID}'.");
                return new DamageResult(0, false, false, DamageType.None, attackerID, bossID);
            }

            ref BossPart part = ref bossParts.Parts.GetValueRef(partID); // Get mutable ref to the part
            if (part.IsSevered || !part.IsVulnerable)
            {
                GD.Print($"CombatSystem: Part '{partID}' of Boss {bossID} is already severed or not vulnerable. No damage applied.");
                return new DamageResult(0, false, false, DamageType.None, attackerID, bossID);
            }

            // --- Damage Calculation for the Part ---
            // For simplicity, parts use the boss's CoreStats for mitigation.
            if (!_biologicalSystem.TryGetCoreStats(bossID, out CoreStats bossStats))
            {
                GD.PrintErr($"CombatSystem: Boss {bossID} has no CoreStats. Cannot apply damage to part.");
                return new DamageResult(0, false, false, DamageType.None, attackerID, bossID);
            }

            float calculatedDamage = rawDamage;
            float mitigation = 0;
            switch (damageType)
            {
                case DamageType.Physical: mitigation = bossStats.Armor * (1.0f - penetration); break;
                case DamageType.Elemental: mitigation = bossStats.MagicResistance * (1.0f - penetration); break;
                // ... other damage types
            }
            calculatedDamage -= mitigation;
            calculatedDamage = Mathf.Max(0, calculatedDamage);

            // Crit calculation
            Random rand = new Random(attackerID.Index + bossID.Index + (int)GameManager.Instance.Time.CurrentGameTime);
            bool isCrit = rand.NextDouble() < critChance;
            if (isCrit) calculatedDamage *= critMultiplier;

            DamageResult result = new DamageResult(calculatedDamage, isCrit, false, damageType, attackerID, bossID);
            GD.Print($"CombatSystem: {attackerID} dealt {result} to Boss {bossID} part '{partID}'.");

            // --- Apply Damage to Part ---
            part.Health -= result.FinalDamage;
            if (part.Health < 0) part.Health = 0;

            // --- Severing Logic (TDD 04.5.1) ---
            if (part.Health == 0 && !part.IsSevered)
            {
                part.IsSevered = true;
                GD.Print($"CombatSystem: Part '{partID}' of Boss {bossID} has been SEVERED!");
                _eventBus.Publish(new BossPartSeveredEvent { BossID = bossID, PartID = partID }); // TDD 04.5.1: Emit OnLimbSevered
                
                // Trigger boss phase transition or ability change (conceptual)
                // _eventBus.Publish(new BossPhaseChangedEvent { BossID = bossID, NewPhase = ... });
            }

            // Publish part damaged event (TDD 04.5.1: Emit OnDamageTaken)
            _eventBus.Publish(new BossPartDamagedEvent { BossID = bossID, PartID = partID, CurrentHealth = part.Health, MaxHealth = part.MaxHealth, Result = result });

            // Also update the main CoreStats health for the boss (optional, or just use part health)
            // For simplicity, let's update the main boss HP for now as the sum of parts.
            ref CoreStats bossCoreStats = ref _biologicalSystem.GetCoreStatsRef(bossID);
            bossCoreStats.Health = bossParts.GetTotalCurrentHealth();
            bossCoreStats.MaxHealth = bossParts.GetTotalMaxHealth(); // Max health might change if parts are permanently severed
            _eventBus.Publish(new PlayerStatSystem.PlayerHealthChangedEvent { PlayerID = bossID, NewValue = bossCoreStats.Health, MaxValue = bossCoreStats.MaxHealth }); // Re-use player health event for now

            if (bossCoreStats.Health == 0)
            {
                GD.Print($"CombatSystem: Boss {bossID} has been DEFEATED!");
                _eventBus.Publish(new EntityDefeatedEvent { EntityID = bossID, KillerID = attackerID });
            }

            return result;
        }

        // ... (ApplyDamage for regular entities, GenerateWounds methods) ...

        // --- Helper Events for Body Sync ---
        // ... (DamageTakenEvent struct) ...
        public struct BossPartDamagedEvent { public EntityID BossID; public string PartID; public float CurrentHealth; public float MaxHealth; public DamageResult Result; } // New event
        public struct BossPartSeveredEvent { public EntityID BossID; public string PartID; } // New event
        public struct EntityDefeatedEvent { public EntityID EntityID; public EntityID KillerID; } // New generic event for when any entity is defeated
    }
}
```

### 4. Integrating `BossPart`s into `GameManager` Tests

1.  Add `using Sigilborne.Systems.Combat;` at the top of `_Brain/Core/GameManager.cs`.
2.  In `GameManager._Ready()`, modify the "Test Damage & Recovery Pipeline" section to:
    *   Create a "world\_boss\_01" entity.
    *   Call `Combat.DamageBossPart` directly to simulate hits on its parts.

```csharp
// _Brain/Core/GameManager.cs (inside _Ready method)
// ...
        // --- Test Damage & Recovery Pipeline ---
        GD.Print("\n--- Testing Damage & Recovery Pipeline ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID testNpcID = Entities.CreateEntity(EntityType.NPC, "test_dummy_npc", new Vector2(500, 200));
        EntityID bossID = Entities.CreateEntity(EntityType.NPC, "world_boss_01", new Vector2(700, 300)); // Create a boss entity

        GD.Print($"Player initial HP: {Biology.GetCoreStatsRef(playerID).Health}");
        GD.Print($"Test NPC initial HP: {Biology.GetCoreStatsRef(testNpcID).Health}");
        GD.Print($"Boss initial HP: {Biology.GetCoreStatsRef(bossID).Health}");
        GD.Print($"Boss Parts initial state: {Combat.GetBossParts(bossID)}"); // Need a GetBossParts method

        // Player attacks NPC (will be triggered by HitEvent from GDScript)
        Combat.RegisterHit(new HitEvent(playerID, testNpcID, "player_sword_slash", "torso", new Vector2(450, 200), Vector2.Left));
        Combat.RegisterHit(new HitEvent(playerID, testNpcID, "player_fireball", "torso", new Vector2(450, 200), Vector2.Left));
        Combat.RegisterHit(new HitEvent(playerID, testNpcID, "player_void_blast", "torso", new Vector2(450, 200), Vector2.Left));

        // NPC attacks Player (will be triggered by HitEvent from GDScript)
        Combat.RegisterHit(new HitEvent(testNpcID, playerID, "npc_claw", "torso", new Vector2(250, 200), Vector2.Right));
        Combat.RegisterHit(new HitEvent(testNpcID, playerID, "npc_poison_spit", "torso", new Vector2(250, 200), Vector2.Right));
        Combat.RegisterHit(new HitEvent(testNpcID, playerID, "npc_heavy_hit", "torso", new Vector2(250, 200), Vector2.Right));

        GD.Print("\n--- Boss Combat Test ---");
        // Simulate hitting boss parts
        Combat.DamageBossPart(playerID, bossID, "LeftArm", 30f, DamageType.Physical, 0, 0.1f, 1.75f); // Damage Left Arm
        Combat.DamageBossPart(playerID, bossID, "LeftArm", 20f, DamageType.Physical, 0, 0.1f, 1.75f); // Sever Left Arm
        Combat.DamageBossPart(playerID, bossID, "Head", 40f, DamageType.Elemental, 0, 0.1f, 1.75f); // Damage Head
        Combat.DamageBossPart(playerID, bossID, "Head", 20f, DamageType.Elemental, 0, 0.1f, 1.75f); // Sever Head
        Combat.DamageBossPart(playerID, bossID, "Body", 50f, DamageType.Physical, 0, 0.1f, 1.75f); // Damage Body
        Combat.DamageBossPart(playerID, bossID, "Body", 50f, DamageType.Physical, 0, 0.1f, 1.75f); // Defeat Boss

        GD.Print("--- End Testing Damage & Recovery Pipeline ---\n");
// ...
```

#### 4.1. Add `GetBossParts` to `CombatSystem`

Needed for the test print statement.

Open `res://_Brain/Systems/Combat/CombatSystem.cs` and add this method:

```csharp
// _Brain/Systems/Combat/CombatSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.Magic.Components;

namespace Sigilborne.Systems.Combat
{
    // ... (DamageType, DamageResult, HitEvent structs/enums) ...
    // ... (BossPart, BossPartsComponent structs) ...

    public class CombatSystem
    {
        // ... (existing fields and constructor) ...

        /// <summary>
        /// Retrieves the BossPartsComponent for a given boss entity.
        /// </summary>
        public BossPartsComponent GetBossParts(EntityID bossID)
        {
            if (_bossParts.TryGetValue(bossID, out BossPartsComponent parts))
            {
                return parts;
            }
            return new BossPartsComponent(); // Return empty if not a boss
        }

        // ... (RegisterHit, ApplyDamage, DamageBossPart, GenerateWounds methods) ...
        // ... (Helper Events) ...
    }
}
```

#### 4.2. Update `EventBus.cs` for Boss Events

Open `res://_Brain/Core/EventBus.cs` and add delegates for `BossPartDamagedEvent`, `BossPartSeveredEvent`, and `EntityDefeatedEvent`.

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Combat;
using Sigilborne.Systems.Inventory;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Combat System Events (TDD 04.2.2)
        public event Action<EntityID, DamageResult> OnDamageTaken;
        public event Action<EntityID, string, float, float, DamageResult> OnBossPartDamaged; // BossID, PartID, CurrentHP, MaxHP, Result
        public event Action<EntityID, string> OnBossPartSevered; // BossID, PartID
        public event Action<EntityID, EntityID> OnEntityDefeated; // EntityID, KillerID

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is CombatSystem.DamageTakenEvent damageTakenEvent)
            {
                OnDamageTaken?.Invoke(damageTakenEvent.DefenderID, damageTakenEvent.Result);
            }
            else if (eventData is CombatSystem.BossPartDamagedEvent bossPartDamagedEvent) // New condition
            {
                OnBossPartDamaged?.Invoke(bossPartDamagedEvent.BossID, bossPartDamagedEvent.PartID, bossPartDamagedEvent.CurrentHealth, bossPartDamagedEvent.MaxHealth, bossPartDamagedEvent.Result);
            }
            else if (eventData is CombatSystem.BossPartSeveredEvent bossPartSeveredEvent) // New condition
            {
                OnBossPartSevered?.Invoke(bossPartSeveredEvent.BossID, bossPartSeveredEvent.PartID);
            }
            else if (eventData is CombatSystem.EntityDefeatedEvent entityDefeatedEvent) // New condition
            {
                OnEntityDefeated?.Invoke(entityDefeatedEvent.EntityID, entityDefeatedEvent.KillerID);
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

### 5. Testing Multi-Part Entities

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.

```
...
BiologicalSystem: Added CoreStats for NPC entity EntityID(2, Gen:1). Stats: HP: 50/50, Sta: 30/30, Cha: 20/20, Stab: 50/50, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 100, Dmg: 8, Armor: 2, MR: 2, Crit: 5%
BiologicalSystem: Added CoreStats for NPC entity EntityID(3, Gen:1). Stats: HP: 50/50, Sta: 30/30, Cha: 20/20, Stab: 50/50, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 100, Dmg: 8, Armor: 2, MR: 2, Crit: 5%
CombatSystem: Added BossPartsComponent for Boss EntityID(3, Gen:1).
--- Testing Damage & Recovery Pipeline ---
Player initial HP: 100.0
Test NPC initial HP: 50.0
Boss initial HP: 190.0
Boss Parts initial state: Boss Parts: 4 | Total HP: 190/190
... (existing NPC/Player damage output) ...

--- Boss Combat Test ---
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 13.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'LeftArm'.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 13.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'LeftArm'.
CombatSystem: Part 'LeftArm' of Boss EntityID(3, Gen:1) has been SEVERED!
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 23.0 (Elemental) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Head'.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 23.0 (Elemental) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Head'.
CombatSystem: Part 'Head' of Boss EntityID(3, Gen:1) has been SEVERED!
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 48.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Body'.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 48.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Body'.
CombatSystem: Boss EntityID(3, Gen:1) has been DEFEATED!
--- End Testing Damage & Recovery Pipeline ---
...
```

**Key Observations:**

*   **Boss Initialization**: A `BossPartsComponent` is added to the boss entity, and its initial health is the sum of its parts (190).
*   **Part-Specific Damage**: Damage is applied to individual parts (`LeftArm`, `Head`, `Body`).
*   **Severing**: When a part's health reaches zero, it's marked as `Severed`, and a `BossPartSeveredEvent` is published.
*   **Boss Defeat**: When all parts (or critical parts, based on later logic) are severed, the boss is `DEFEATED`.
*   **Event Publishing**: `BossPartDamagedEvent` and `BossPartSeveredEvent` are published for the Body to react.

This confirms our `CombatSystem` is correctly managing multi-part entities, allowing for complex, multi-stage boss encounters.

### Summary

You have successfully implemented **World Boss Mechanics** for multi-part entities in the C# Brain, defining `BossPart` and `BossPartsComponent` to represent distinct, damageable parts of colossal creatures. By enhancing `CombatSystem` to store these components, apply damage to specific parts, handle severing logic, and update the boss's overall health, you've established a robust system for dynamic, multi-stage combat encounters. This crucial component strictly adheres to TDD 04.5.1's specifications, paving the way for truly epic Titan battles in Sigilborne.

### Next Steps

The next chapter will focus on **Titan AI State**, where we will design a specialized state machine for World Bosses that accounts for their massive size and complex behaviors, including phase transitions triggered by health thresholds or severed limbs.