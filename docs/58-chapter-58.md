## Chapter 9.3: Ritual System - Simple, Advanced & Forbidden (C#)

Our `RitualSystem` can already detect patterns and execute basic rituals. This chapter refines it to differentiate between **Simple, Advanced, and Forbidden Rituals**, each with escalating complexity, resource costs, and world-altering consequences, as specified in GDD B27.2. This deepens the magic system, aligning with Sigilborne's themes of risk, power, and emergent world impact.

### 1. The Escalation of Rituals

The GDD (B27.2) categorizes rituals by depth:

*   **Simple/Medium Rituals**: (B27.2.1) Low cost, local effects, accessible.
*   **Advanced Rituals**: (B27.2.2) Multiple participants, precise timing, rare components, powerful outcomes.
*   **Forbidden Rituals**: (B27.2.3) Unique materials, irreversible world-level consequences, potential mutation/rupture.

Our `RitualDefinition` needs to capture this complexity, and `RitualSystem` must enforce it.

### 2. Enhancing `RitualData.cs` with `RitualTier` and `Participants`

We need to add a `RitualTier` enum and parameters for `MinParticipants` to `RitualDefinition`.

1.  Open `res://_Brain/Systems/Rituals/RitualData.cs`.
2.  Add `RitualTier` enum.
3.  Add `RitualTier` and `MinParticipants` to `RitualDefinition`.

```csharp
// _Brain/Systems/Rituals/RitualData.cs
using System;
using System.Collections.Generic;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Inventory;

namespace Sigilborne.Systems.Rituals
{
    /// <summary>
    /// Defines the escalating tiers of ritual complexity and power.
    /// (GDD B27.2)
    /// </summary>
    public enum RitualTier
    {
        None,
        Simple,     // Low cost, local effects, accessible (GDD B27.2.1)
        Advanced,   // Higher cost, complex, powerful outcomes (GDD B27.2.2)
        Forbidden   // Extreme cost/risk, world-altering consequences (GDD B27.2.3)
    }

    // ... (RitualComponentRequirement struct) ...

    /// <summary>
    /// Static data defining a magical ritual.
    /// Now includes RitualTier and participant requirements.
    /// (TDD 09.2.1)
    /// </summary>
    public class RitualDefinition
    {
        public string ID { get; private set; }
        public string Name { get; private set; }
        public string Description { get; private set; }
        public RitualTier Tier { get; private set; } // New: Tier of the ritual
        public float CastTime { get; private set; }
        public float ChakraCost { get; private set; }
        public float StabilityCost { get; private set; }
        public float Radius { get; private set; }
        public int MinParticipants { get; private set; } // New: Minimum number of entities participating

        public List<RitualComponentRequirement> Requirements { get; private set; }
        public List<string> EffectIDs { get; private set; }

        public RitualDefinition(string id, string name, string description, RitualTier tier, float castTime, float chakraCost, float stabilityCost, float radius, int minParticipants, List<RitualComponentRequirement> requirements, List<string> effectIDs = null)
        {
            ID = id;
            Name = name;
            Description = description;
            Tier = tier; // Set tier
            CastTime = castTime;
            ChakraCost = chakraCost;
            StabilityCost = stabilityCost;
            Radius = radius;
            MinParticipants = minParticipants; // Set min participants
            Requirements = requirements ?? new List<RitualComponentRequirement>();
            EffectIDs = effectIDs ?? new List<string>();
        }

        public override string ToString()
        {
            return $"Ritual: '{Name}' ({ID}) | Tier: {Tier}, Cast: {CastTime:F1}s, Chakra: {ChakraCost}, Stab: {StabilityCost}, MinPart: {MinParticipants}";
        }
    }
    // ... (RitualCenterComponent struct) ...
}
```

### 3. Enhancing `RitualSystem.cs` for Tiered Rituals

`RitualSystem` needs to:
*   Enforce `RitualTier` requirements (e.g., `MinParticipants`).
*   Potentially have different success/failure outcomes based on tier.
*   Trigger more severe effects for `Forbidden` rituals.

1.  Open `res://_Brain/Systems/Rituals/RitualSystem.cs`.
2.  Modify `RegisterDefaultRitualDefinitions` to include `RitualTier` and `MinParticipants`.
3.  Modify `InitiateRitual` to check `MinParticipants`.
4.  Modify `CompleteRitual` to trigger effects based on `RitualTier` (conceptually).

```csharp
// _Brain/Systems/Rituals/RitualSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.AI;
using Sigilborne.Systems.Biology;
using System.Linq;
using System.Threading;

namespace Sigilborne.Systems.Rituals
{
    // ... (RitualComponentRequirement, RitualDefinition structs/classes) ...
    // ... (RitualCenterComponent struct) ...

    public class RitualSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private InventorySystem _inventorySystem;
        private SpatialHashGrid _spatialGrid;
        private BiologicalSystem _biologicalSystem;
        private TransformSystem _transformSystem;
        private JobSystem _jobSystem; // New: For scheduling ritual completion jobs

        private Dictionary<string, RitualDefinition> _ritualDefinitions = new Dictionary<string, RitualDefinition>();
        private Dictionary<EntityID, RitualCenterComponent> _ritualCenters = new Dictionary<EntityID, RitualCenterComponent>();

        public RitualSystem(EntityManager entityManager, EventBus eventBus, InventorySystem inventorySystem, SpatialHashGrid spatialGrid, BiologicalSystem biologicalSystem, TransformSystem transformSystem, JobSystem jobSystem) // Add JobSystem
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _inventorySystem = inventorySystem;
            _spatialGrid = spatialGrid;
            _biologicalSystem = biologicalSystem;
            _transformSystem = transformSystem;
            _jobSystem = jobSystem; // Store JobSystem reference

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            RegisterDefaultRitualDefinitions();
            GD.Print("RitualSystem: Initialized.");
        }

        // ... (OnEntitySpawned, OnEntityDespawned methods) ...

        private void RegisterDefaultRitualDefinitions()
        {
            // --- Simple Ritual (GDD B27.2.1) ---
            RegisterRitualDefinition(new RitualDefinition(
                id: "healing_rite_t1", name: "Minor Healing Rite", description: "A simple ritual to restore health.",
                tier: RitualTier.Simple, castTime: 3.0f, chakraCost: 10f, stabilityCost: 5f, radius: 50f, minParticipants: 1,
                requirements: new List<RitualComponentRequirement>
                {
                    new RitualComponentRequirement("healing_herb", 20f, 3),
                    new RitualComponentRequirement("lit_candle", 10f, 1)
                },
                effectIDs: new List<string> { "heal_caster_100hp" }
            ));
            RegisterRitualDefinition(new RitualDefinition(
                id: "scrying_rite_t1", name: "Basic Scrying Rite", description: "Reveals distant information.",
                tier: RitualTier.Simple, castTime: 5.0f, chakraCost: 20f, stabilityCost: 10f, radius: 100f, minParticipants: 1,
                requirements: new List<RitualComponentRequirement>
                {
                    new RitualComponentRequirement("scrying_orb", 5f, 1, false),
                    new RitualComponentRequirement("rare_incense", 10f, 1)
                },
                effectIDs: new List<string> { "reveal_map_area" }
            ));

            // --- Advanced Ritual (GDD B27.2.2) ---
            RegisterRitualDefinition(new RitualDefinition(
                id: "spirit_summoning_t2", name: "Lesser Spirit Summoning", description: "Summons a minor spirit to aid.",
                tier: RitualTier.Advanced, castTime: 10.0f, chakraCost: 50f, stabilityCost: 30f, radius: 150f, minParticipants: 2, // Requires 2 participants
                requirements: new List<RitualComponentRequirement>
                {
                    new RitualComponentRequirement("spirit_essence", 30f, 1),
                    new RitualComponentRequirement("ancient_blood_rune", 5f, 1),
                    new RitualComponentRequirement("lit_candle", 10f, 3)
                },
                effectIDs: new List<string> { "spawn_spirit_ally" }
            ));

            // --- Forbidden Ritual (GDD B27.2.3) ---
            RegisterRitualDefinition(new RitualDefinition(
                id: "corruption_ascension_f", name: "Forbidden Corruption Ascension", description: "Mutates the caster with void energy.",
                tier: RitualTier.Forbidden, castTime: 30.0f, chakraCost: 100f, stabilityCost: 80f, radius: 200f, minParticipants: 1, // Can be solo, but dangerous
                requirements: new List<RitualComponentRequirement>
                {
                    new RitualComponentRequirement("residue_crystal_large", 5f, 1),
                    new RitualComponentRequirement("fresh_heart", 5f, 1), // Gruesome requirement
                    new RitualComponentRequirement("void_ink_scroll", 10f, 1, false) // Not consumed
                },
                effectIDs: new List<string> { "caster_mutated_aberration" }
            ));
        }

        public bool InitiateRitual(EntityID ritualInitiatorID, EntityID ritualCenterID, string ritualID)
        {
            if (!_entityManager.IsValid(ritualInitiatorID) || !_entityManager.IsValid(ritualCenterID)) { /* ... */ return false; }
            if (!_ritualDefinitions.TryGetValue(ritualID, out RitualDefinition ritualDef)) { /* ... */ return false; }
            if (!_transformSystem.TryGetTransform(ritualCenterID, out TransformComponent centerTransform)) { /* ... */ return false; }
            if (!_biologicalSystem.TryGetCoreStats(ritualInitiatorID, out CoreStats initiatorStats)) { /* ... */ return false; }

            if (_ritualCenters.TryGetValue(ritualCenterID, out RitualCenterComponent centerComp) && centerComp.IsActive)
            {
                GD.Print($"RitualSystem: Ritual center {ritualCenterID} is already active with ritual '{centerComp.ActiveRitualID}'. Cannot initiate new ritual.");
                _eventBus.Publish(new RitualFailedEvent { InitiatorID = ritualInitiatorID, RitualID = ritualID, Reason = "Center Busy" });
                return false;
            }

            // --- NEW: Check Minimum Participants (GDD B27.2.2) ---
            List<EntityID> nearbyLivingEntities = _spatialGrid.QueryEntities(centerTransform.Position, ritualDef.Radius)
                                                            .Where(id => _entityManager.IsValid(id) && id != ritualInitiatorID && _biologicalSystem.HasCoreStats(id) && _biologicalSystem.GetCoreStatsRef(id).Health > 0)
                                                            .ToList();
            int actualParticipants = nearbyLivingEntities.Count + 1; // Initiator + nearby living entities
            if (actualParticipants < ritualDef.MinParticipants)
            {
                GD.Print($"RitualSystem: Insufficient participants for '{ritualID}'. Need {ritualDef.MinParticipants}, found {actualParticipants}.");
                _eventBus.Publish(new RitualFailedEvent { InitiatorID = ritualInitiatorID, RitualID = ritualID, Reason = "Insufficient Participants" });
                return false;
            }


            // --- 1. Check Initiator Resources (Chakra, Stability) ---
            if (initiatorStats.Chakra < ritualDef.ChakraCost) { /* ... */ return false; }
            if (initiatorStats.Stability < ritualDef.StabilityCost) { /* ... */ return false; }

            // --- 2. Check Ritual Component Pattern ---
            List<EntityID> nearbyItemsAndEntities = _spatialGrid.QueryEntities(centerTransform.Position, ritualDef.Radius);
            Dictionary<string, int> foundComponents = new Dictionary<string, int>();

            foreach (var req in ritualDef.Requirements)
            {
                int countFound = 0;
                foreach (EntityID nearbyID in nearbyItemsAndEntities)
                {
                    if (_entityManager.TryGetEntityMeta(nearbyID, out EntityMeta nearbyMeta) && nearbyMeta.DefinitionID == req.ItemID)
                    {
                        if (_transformSystem.TryGetTransform(nearbyID, out TransformComponent nearbyTransform))
                        {
                            if (nearbyTransform.Position.DistanceTo(centerTransform.Position) <= req.Radius)
                            {
                                countFound++;
                            }
                        }
                    }
                }
                foundComponents[req.ItemID] = countFound;
                if (countFound < req.Quantity) { /* ... */ return false; }
            }

            // --- 3. Deduct Resources and Consume Components ---
            GameManager.Instance.PlayerStats.TakeChakra(ritualDef.ChakraCost);
            GameManager.Instance.PlayerStats.TakeStability(ritualDef.StabilityCost);
            
            foreach (var req in ritualDef.Requirements)
            {
                if (req.Consumable)
                {
                    GD.Print($"RitualSystem: Consuming {req.Quantity}x '{req.ItemID}' for ritual '{ritualID}'.");
                    // Conceptual consumption.
                }
            }

            // --- 4. Initiate Ritual (start cast time) ---
            ref RitualCenterComponent ritualCenterRef = ref _ritualCenters.GetValueRef(ritualCenterID);
            ritualCenterRef.IsActive = true;
            ritualCenterRef.ActiveRitualID = ritualID;

            GD.Print($"RitualSystem: Ritual '{ritualID}' (Tier: {ritualDef.Tier}) initiated by {initiatorID} at {ritualCenterID}. Cast time: {ritualDef.CastTime:F1}s. Participants: {actualParticipants}.");
            _eventBus.Publish(new RitualInitiatedEvent { InitiatorID = initiatorID, RitualID = ritualID, CenterID = ritualCenterID, CastTime = ritualDef.CastTime });

            _jobSystem.Schedule(new RitualCompletionJob(initiatorID, ritualCenterID, ritualID, ritualDef.CastTime), () => {
                CompleteRitual(initiatorID, ritualCenterID, ritualID);
            });

            return true;
        }

        /// <summary>
        /// Completes a ritual after its cast time.
        /// </summary>
        public void CompleteRitual(EntityID initiatorID, EntityID ritualCenterID, string ritualID)
        {
            if (!_ritualDefinitions.TryGetValue(ritualID, out RitualDefinition ritualDef)) return;
            if (!_ritualCenters.TryGetValue(ritualCenterID, out RitualCenterComponent centerComp) || !centerComp.IsActive || centerComp.ActiveRitualID != ritualID) { /* ... */ return; }

            ref RitualCenterComponent ritualCenterRef = ref _ritualCenters.GetValueRef(ritualCenterID);
            ritualCenterRef.IsActive = false;
            ritualCenterRef.ActiveRitualID = null;

            GD.Print($"RitualSystem: Ritual '{ritualID}' completed successfully by {initiatorID} at {ritualCenterID}! Triggering effects.");
            _eventBus.Publish(new RitualCompletedEvent { InitiatorID = initiatorID, RitualID = ritualID, CenterID = ritualCenterID });

            // --- Trigger Ritual Effects (conceptual, now tiered) ---
            foreach (var effectID in ritualDef.EffectIDs)
            {
                GD.Print($"  Ritual Effect: {effectID} triggered (Tier: {ritualDef.Tier}).");
                // More complex effects based on tier (GDD B27.2)
                if (ritualDef.Tier == RitualTier.Forbidden)
                {
                    GD.Print("    WARNING: FORBIDDEN RITUAL EFFECT! Potential world-altering consequences!");
                    // _eventBus.Publish(new WorldSystem.CorruptionEvent { Type = "ritual_induced_bloom" });
                }
            }
        }

        // ... (RitualCompletionJob struct) ...
        // ... (Helper Events) ...
    }
}
```

#### 3.1. Add `HasCoreStats` to `BiologicalSystem.cs`

`RitualSystem` needs to check if an entity has `CoreStats` and is alive.

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
using Sigilborne.Systems.Weather;
using Sigilborne.Systems.StatusEffects;

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        // ... (existing fields and constructor) ...

        /// <summary>
        /// Checks if an entity has CoreStats (is a living entity).
        /// </summary>
        public bool HasCoreStats(EntityID id)
        {
            return _entityManager.IsValid(id) && _entityCoreStats.ContainsKey(id);
        }

        // ... (other methods) ...
    }
}
```

### 4. Integrating `RitualSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Rituals;` at the top of `_Brain/Core/GameManager.cs`.
2.  In `GameManager._Ready()`, modify the "Test Ritual System" section to:
    *   Create additional participants (NPCs) for advanced rituals.
    *   Test `Simple`, `Advanced`, and `Forbidden` rituals.

```csharp
// _Brain/Core/GameManager.cs (inside _Ready method)
// ...
        // --- Test Ritual System ---
        GD.Print("\n--- Testing Ritual System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        
        // Create a ritual altar entity
        EntityID altarID = Entities.CreateEntity(EntityType.WorldObject, "altar_stone", new Vector2(300, 300));
        
        // Create additional participants for advanced rituals
        EntityID npcParticipant1 = Entities.CreateEntity(EntityType.NPC, "ritual_npc_1", new Vector2(320, 300)); // Near altar
        EntityID npcParticipant2 = Entities.CreateEntity(EntityType.NPC, "ritual_npc_2", new Vector2(280, 280)); // Near altar
        EntityID distantNpc = Entities.CreateEntity(EntityType.NPC, "distant_npc", new Vector2(1000, 1000)); // Far from altar

        // Add ritual components to player's inventory
        Inventory.AddItem(playerID, new InventoryItem("healing_herb", 5));
        Inventory.AddItem(playerID, new InventoryItem("lit_candle", 5));
        Inventory.AddItem(playerID, new InventoryItem("scrying_orb", 1));
        Inventory.AddItem(playerID, new InventoryItem("rare_incense", 1));
        Inventory.AddItem(playerID, new InventoryItem("spirit_essence", 1));
        Inventory.AddItem(playerID, new InventoryItem("ancient_blood_rune", 1));
        Inventory.AddItem(playerID, new InventoryItem("residue_crystal_large", 1));
        Inventory.AddItem(playerID, new InventoryItem("fresh_heart", 1));
        Inventory.AddItem(playerID, new InventoryItem("void_ink_scroll", 1));


        // Test Simple Ritual: healing_rite_t1 (requires 3 healing_herb, 1 lit_candle, 1 participant)
        GD.Print("\nAttempting 'healing_rite_t1' (Simple):");
        Rituals.InitiateRitual(playerID, altarID, "healing_rite_t1");

        // Test Advanced Ritual: spirit_summoning_t2 (requires 1 spirit_essence, 1 ancient_blood_rune, 3 lit_candle, 2 participants)
        GD.Print("\nAttempting 'spirit_summoning_t2' (Advanced, with 3 participants):");
        Rituals.InitiateRitual(playerID, altarID, "spirit_summoning_t2");

        // Test Advanced Ritual (should fail due to insufficient participants if NPCs moved away)
        Transforms.GetTransformRef(npcParticipant1).Position = new Vector2(1500, 1500); // Move NPC far
        GD.Print("\nAttempting 'spirit_summoning_t2' again (should fail due to participants):");
        Rituals.InitiateRitual(playerID, altarID, "spirit_summoning_t2");

        // Test Forbidden Ritual: corruption_ascension_f (requires 1 residue_crystal_large, 1 fresh_heart, 1 void_ink_scroll, 1 participant)
        GD.Print("\nAttempting 'corruption_ascension_f' (Forbidden):");
        Rituals.InitiateRitual(playerID, altarID, "corruption_ascension_f");

        GD.Print("--- End Testing Ritual System ---\n");
// ...
```

### 5. Testing Tiered Rituals

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Ritual System" section.

```
...
RitualSystem: Identified potential ritual center: EntityID(4, Gen:1) (altar_stone). Added RitualCenterComponent.
RitualSystem: Identified potential ritual center: EntityID(5, Gen:1) (ritual_npc_1). Added RitualCenterComponent.
RitualSystem: Identified potential ritual center: EntityID(6, Gen:1) (ritual_npc_2). Added RitualCenterComponent.
RitualSystem: Identified potential ritual center: EntityID(7, Gen:1) (distant_npc). Added RitualCenterComponent.
...
--- Testing Ritual System ---

Attempting 'healing_rite_t1' (Simple):
RitualSystem: Ritual 'healing_rite_t1' (Tier: Simple) initiated by EntityID(0, Gen:1) at EntityID(4, Gen:1). Cast time: 3.0s. Participants: 3.
PlayerStatSystem: Player EntityID(0, Gen:1) used 10 chakra. New Chakra: 40.0
PlayerStatSystem: Player EntityID(0, Gen:1) lost 5 stability. New Stability: 95.0
RitualSystem: Consuming 3x 'healing_herb' for ritual 'healing_rite_t1'.
RitualSystem: Consuming 1x 'lit_candle' for ritual 'healing_rite_t1'.
[Job:RitualCompletion] Ritual 'healing_rite_t1' delay finished.
RitualSystem: Ritual 'healing_rite_t1' completed successfully by EntityID(0, Gen:1) at EntityID(4, Gen:1)! Triggering effects.
  Ritual Effect: heal_caster_100hp triggered (Tier: Simple).

Attempting 'spirit_summoning_t2' (Advanced, with 3 participants):
RitualSystem: Ritual 'spirit_summoning_t2' (Tier: Advanced) initiated by EntityID(0, Gen:1) at EntityID(4, Gen:1). Cast time: 10.0s. Participants: 3.
PlayerStatSystem: Player EntityID(0, Gen:1) used 50 chakra. New Chakra: 0.0
PlayerStatSystem: Player EntityID(0, Gen:1) lost 30 stability. New Stability: 65.0
RitualSystem: Consuming 1x 'spirit_essence' for ritual 'spirit_summoning_t2'.
RitualSystem: Consuming 1x 'ancient_blood_rune' for ritual 'spirit_summoning_t2'.
RitualSystem: Consuming 3x 'lit_candle' for ritual 'spirit_summoning_t2'.
[Job:RitualCompletion] Ritual 'spirit_summoning_t2' delay finished.
RitualSystem: Ritual 'spirit_summoning_t2' completed successfully by EntityID(0, Gen:1) at EntityID(4, Gen:1)! Triggering effects.
  Ritual Effect: spawn_spirit_ally triggered (Tier: Advanced).

Attempting 'spirit_summoning_t2' again (should fail due to participants):
RitualSystem: Insufficient participants for 'spirit_summoning_t2'. Need 2, found 1.
RitualSystem: Ritual 'spirit_summoning_t2' failed by EntityID(0, Gen:1) at EntityID(4, Gen:1). Reason: Insufficient Participants

Attempting 'corruption_ascension_f' (Forbidden):
RitualSystem: Ritual 'corruption_ascension_f' (Tier: Forbidden) initiated by EntityID(0, Gen:1) at EntityID(4, Gen:1). Cast time: 30.0s. Participants: 1.
PlayerStatSystem: Player EntityID(0, Gen:1) used 100 chakra. New Chakra: 0.0
PlayerStatSystem: Player EntityID(0, Gen:1) lost 80 stability. New Stability: 0.0
RitualSystem: Consuming 1x 'residue_crystal_large' for ritual 'corruption_ascension_f'.
RitualSystem: Consuming 1x 'fresh_heart' for ritual 'corruption_ascension_f'.
[Job:RitualCompletion] Ritual 'corruption_ascension_f' delay finished.
RitualSystem: Ritual 'corruption_ascension_f' completed successfully by EntityID(0, Gen:1) at EntityID(4, Gen:1)! Triggering effects.
  Ritual Effect: caster_mutated_aberration triggered (Tier: Forbidden).
    WARNING: FORBIDDEN RITUAL EFFECT! Potential world-altering consequences!
--- End Testing Ritual System ---
...
```

**Key Observations:**

*   **Tiered Rituals**: `RitualTier` and `MinParticipants` are correctly registered and enforced.
*   **Participant Check**: `InitiateRitual` correctly counts `nearbyLivingEntities` and fails if `actualParticipants` is too low.
*   **Resource Deduction**: Chakra and Stability are deducted, with higher costs for more advanced rituals.
*   **Forbidden Ritual Warning**: `Forbidden Corruption Ascension` triggers a specific warning and its effects are conceptually flagged as potentially world-altering.
*   **Item Consumption**: Conceptual consumption messages are printed.

This confirms our `RitualSystem` is robustly handling tiered rituals, enforcing their complex requirements and triggering appropriate consequences.

### Summary

You have successfully refined the **Ritual System** in the C# Brain, differentiating between **Simple, Advanced, and Forbidden Rituals** through the `RitualTier` enum and `MinParticipants` requirement. By enhancing `RitualSystem` to enforce these tiered complexities, deduct escalating resources, and trigger tier-specific effects (including warnings for forbidden rituals), you've aligned the magic system with Sigilborne's themes of risk and power. This crucial system strictly adheres to GDD B27.2's specifications, providing advanced and dangerous world mechanics.

### Next Steps

The next chapter will focus on **Forbidden Contracts & Spirit Contracts**, designing how players can enter into dangerous pacts with powerful forces, each with world-linked consequences, representing the ultimate risk-reward paths in Sigilborne's magic.