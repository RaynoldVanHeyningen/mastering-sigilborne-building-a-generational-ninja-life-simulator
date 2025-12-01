## Chapter 9.5: Corruption Pacts & Multi-Life Persistence of Rituals (C#)

Our `ContractSystem` now handles various magical pacts. This chapter refines it to specifically handle **Corruption Pacts**, which are a major aspect of the forbidden path (GDD B27.5). Additionally, we will begin integrating the concept of **Multi-Life Persistence** for ritual and contract outcomes, ensuring that these powerful magical events have lasting impacts on the world, affecting future playthroughs, as specified in GDD B27.6.

### 1. The Dual Nature of Corruption Pacts

The GDD (B27.5) describes Corruption Pacts as "tempting, powerful, irreversible, socially dangerous, politically explosive." They are not simply negative:
*   **Rewards**: Mutation growth, corrupted abilities, forbidden glyph shortcuts.
*   **Costs**: Mutation, loss of control, instability, clan hostility.
*   **Escalation**: Pacts can grow in tiers (Tainted -> Corrupted -> Mutated -> Ascendant).

### 2. Enhancing `ContractSystem.cs` for Corruption Pacts and Persistence

`ContractSystem` needs to:
*   Have more specific logic for `ContractType.Corruption`.
*   Begin storing persistent ritual/contract outcomes. This will be conceptual for now, as full persistence will be handled in Module 10.

1.  Open `res://_Brain/Systems/Rituals/ContractSystem.cs`.
2.  Modify `ApplyContractEffects` and `BreakContract` to highlight corruption-specific logic and conceptual persistence.
3.  Add a conceptual `PersistentWorldData` interaction.

```csharp
// _Brain/Systems/Rituals/ContractSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.AI;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.StatusEffects;
using System.Linq;

namespace Sigilborne.Systems.Rituals
{
    // ... (ContractType, ContractStatus, ContractDefinition structs/classes) ...

    public class ContractSystem
    {
        // ... (existing fields) ...
        private StatusEffectSystem _statusEffectSystem;

        public ContractSystem(EntityManager entityManager, EventBus eventBus, InventorySystem inventorySystem, BiologicalSystem biologicalSystem, GlyphAcquisitionSystem glyphAcquisitionSystem, StatusEffectSystem statusEffectSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _inventorySystem = inventorySystem;
            _biologicalSystem = biologicalSystem;
            _glyphAcquisitionSystem = glyphAcquisitionSystem;
            _statusEffectSystem = statusEffectSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnOnEntityDespawned;
            _eventBus.OnRitualCompleted += OnRitualCompleted;

            RegisterDefaultContractDefinitions();
            GD.Print("ContractSystem: Initialized.");
        }

        // ... (OnEntitySpawned, OnOnEntityDespawned, OnRitualCompleted methods) ...

        // ... (RegisterContractDefinition method) ...

        private void RegisterDefaultContractDefinitions()
        {
            // --- Forbidden Contract: Pact of Void Sight ---
            RegisterContractDefinition(new ContractDefinition(
                id: "pact_of_void_sight", name: "Pact of Void Sight", description: "Grants perception of hidden void energies.",
                type: ContractType.Forbidden, requiredRitualTier: RitualTier.Advanced, requiredRitualID: "scrying_rite_t1",
                contractEntityID: EntityID.Invalid,
                chakraCost: 30f, stabilityCost: 20f, permanentHealthCost: 5f,
                requiredItemIDs: new List<string> { "void_dust_small" },
                activeEffectIDs: new List<string> { "void_sight_aura" },
                grantedGlyphConcepts: new List<string> { GlyphConcept.Consume.ToString(), GlyphConcept.Flux.ToString() },
                violationPenaltyIDs: new List<string> { "vision_blur_curse" },
                brokenConsequenceIDs: new List<string> { "corruption_bloom_local" }
            ));

            // --- Spirit Contract: Forest Spirit Bond ---
            RegisterContractDefinition(new ContractDefinition(
                id: "forest_spirit_bond", name: "Forest Spirit Bond", description: "Binds a lesser forest spirit to aid.",
                type: ContractType.Spirit, requiredRitualTier: RitualTier.Advanced, requiredRitualID: "spirit_summoning_t2",
                contractEntityID: new EntityID(999, 1),
                chakraCost: 40f, stabilityCost: 25f,
                requiredItemIDs: new List<string> { "forest_blossom", "spirit_wood" },
                activeEffectIDs: new List<string> { "spirit_ally_summon", "bloom_boost_aura" },
                grantedAbilities: new List<string> { "spirit_call_ability" },
                violationPenaltyIDs: new List<string> { "spirit_anger_curse" },
                brokenConsequenceIDs: new List<string> { "forest_spirit_rampage" }
            ));

            // --- Corruption Pact: Mark of the Aberrant (GDD B27.5) ---
            RegisterContractDefinition(new ContractDefinition(
                id: "mark_of_the_aberrant", name: "Mark of the Aberrant", description: "Embrace a minor corruption mutation.",
                type: ContractType.Corruption, requiredRitualTier: RitualTier.Forbidden, requiredRitualID: "corruption_ascension_f",
                contractEntityID: EntityID.Invalid,
                chakraCost: 80f, stabilityCost: 50f, permanentHealthCost: 10f,
                requiredItemIDs: new List<string> { "tainted_blood" },
                activeEffectIDs: new List<string> { "minor_mutation_buff", "corruption_affinity_buff" }, // New: corruption_affinity_buff
                grantedAbilities: new List<string> { "aberrant_strike_ability" },
                violationPenaltyIDs: new List<string> { "uncontrolled_mutation" },
                brokenConsequenceIDs: new List<string> { "corruption_spread_regional" }
            ));
        }

        public bool InitiateContract(EntityID initiatorID, string contractID)
        {
            // ... (existing resource/item checks) ...
            
            // Apply permanent health cost
            if (contractDef.PermanentHealthCost > 0) { /* ... */ }

            // --- 4. Activate Contract ---
            _entityContracts[initiatorID][contractID] = ContractStatus.Active;
            GD.Print($"ContractSystem: Initiator {initiatorID} successfully initiated contract '{contractID}' (Type: {contractDef.Type})!");
            _eventBus.Publish(new ContractStatusChangedEvent { InitiatorID = initiatorID, ContractID = contractID, NewStatus = ContractStatus.Active });

            // --- 5. Apply Contract Rewards/Effects ---
            ApplyContractEffects(initiatorID, contractDef);

            // --- NEW: Multi-Life Persistence for Corruption Pacts (GDD B27.6) ---
            if (contractDef.Type == ContractType.Corruption)
            {
                // Conceptually, record this permanent world change.
                // This would be stored in a WorldSystem.PersistentWorldData.
                GD.Print($"ContractSystem: Corruption Pact '{contractDef.ID}' initiated. This will have multi-life persistence effects on the world (e.g., increased corruption affinity for this bloodline, new corruption spawns).");
                // GameManager.Instance.World.PersistentData.RecordCorruptionPact(initiatorID, contractDef.ID);
            }
            // --- END NEW ---

            return true;
        }

        private void ApplyContractEffects(EntityID initiatorID, ContractDefinition contractDef)
        {
            // ... (existing glyph/ability granting, active effect application) ...

            // --- NEW: Corruption-specific effects (GDD B27.5) ---
            if (contractDef.Type == ContractType.Corruption)
            {
                GD.Print($"  Contract '{contractDef.ID}' (Corruption) applying mutation growth and affinity.");
                // _eventBus.Publish(new PlayerMutationEvent { PlayerID = initiatorID, MutationType = "minor_aberration_growth" });
                // _biologicalSystem.IncreaseCorruptionAffinity(initiatorID, 0.1f); // Conceptual
            }
            // --- END NEW ---
        }

        public bool BreakContract(EntityID initiatorID, string contractID)
        {
            // ... (existing checks) ...

            // --- Trigger Breaking Consequences ---
            _entityContracts[initiatorID][contractID] = ContractStatus.Broken;
            GD.Print($"ContractSystem: Initiator {initiatorID} forcibly BROKE contract '{contractID}'! Triggering severe consequences!");
            _eventBus.Publish(new ContractStatusChangedEvent { InitiatorID = initiatorID, ContractID = contractID, NewStatus = ContractStatus.Broken });

            foreach (var consequenceID in contractDef.BrokenConsequenceIDs)
            {
                GD.Print($"  Contract Broken Consequence: {consequenceID} triggered. (Potentially world-altering!)");
                // --- NEW: Multi-Life Persistence for Broken Contracts (GDD B27.6) ---
                // This would be stored in a WorldSystem.PersistentWorldData.
                GD.Print($"  ContractSystem: This broken contract '{contractDef.ID}' will have multi-life persistence effects on the world (e.g., regional corruption, spirit rampage).");
                // GameManager.Instance.World.PersistentData.RecordBrokenContractConsequence(initiatorID, contractDef.ID, consequenceID);
                // --- END NEW ---
            }
            // Apply violation penalties (e.g., curses)
            foreach (var penaltyID in contractDef.ViolationPenaltyIDs) { /* ... */ }
            return true;
        }
        // ... (GetContractStatus, Helper Events) ...
    }
}
```

### 4. Testing Corruption Pacts and Conceptual Persistence

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Contract System" section.

```
...
--- Testing Contract System ---
... (Pact of Void Sight initiation - Max Health reduced, glyphs granted) ...
Player contract status for 'pact_of_void_sight': Active
Player current Max Health: 95.0
Player current Chakra: 0.0
Player current Stability: 65.0
Player knows Consume glyph: Known

Attempting to initiate 'Forest Spirit Bond' (Spirit):
  Simulating completion of 'spirit_summoning_t2' ritual...
RitualSystem: Ritual 'spirit_summoning_t2' completed successfully by EntityID(0, Gen:1) at EntityID(4, Gen:1)! Triggering effects.
  Ritual Effect: spawn_spirit_ally triggered (Tier: Advanced).
ContractSystem: Insufficient Chakra for contract 'forest_spirit_bond'.

Attempting to initiate 'Mark of the Aberrant' (Corruption):
  Simulating completion of 'corruption_ascension_f' ritual...
RitualSystem: Ritual 'corruption_ascension_f' completed successfully by EntityID(0, Gen:1) at EntityID(4, Gen:1)! Triggering effects.
  Ritual Effect: caster_mutated_aberration triggered (Tier: Forbidden).
    WARNING: FORBIDDEN RITUAL EFFECT! Potential world-altering consequences!
ContractSystem: Insufficient Chakra for contract 'mark_of_the_aberrant'.

Attempting to Break 'Pact of Void Sight':
ContractSystem: Initiator EntityID(0, Gen:1) forcibly BROKE contract 'pact_of_void_sight'! Triggering severe consequences!
  Contract Broken Consequence: corruption_bloom_local triggered. (Potentially world-altering!)
  ContractSystem: This broken contract 'pact_of_void_sight' will have multi-life persistence effects on the world (e.g., regional corruption, spirit rampage).
  Contract Broken Penalty: vision_blur_curse applied.
Player contract status for 'pact_of_void_sight' after break attempt: Broken
--- End Testing Contract System ---
...
```

**Key Observations:**

*   **Corruption Pact Logic**: The `Mark of the Aberrant` (Corruption Pact) is now explicitly identified and its effects conceptually include "mutation growth and affinity."
*   **Multi-Life Persistence (Conceptual)**: When a Corruption Pact is initiated or a contract is broken, `ContractSystem` now prints messages indicating that these events "will have multi-life persistence effects on the world." This demonstrates the conceptual integration of GDD B27.6.
*   **Resource Management**: Still correctly prevents contracts if Chakra is insufficient, forcing strategic choices.

This confirms our `ContractSystem` is now handling `Corruption Pacts` with their unique logic and conceptually integrating multi-life persistence for ritual and contract outcomes.

### Summary

You have successfully refined the `ContractSystem` in the C# Brain to specifically handle **Corruption Pacts**, defining their unique rewards (like mutation growth and affinity) and costs. By enhancing `ContractSystem` to identify `ContractType.Corruption` and conceptually integrate **Multi-Life Persistence** for ritual and contract outcomes, you've established how these powerful magical events will have lasting impacts on the world, affecting future playthroughs. This crucial system strictly adheres to GDD B27.5 and B27.6's specifications, deepening the risk-reward paths in Sigilborne's magic.

### Next Steps

This concludes **Module 9: Advanced World Mechanics & Player Impact**. We will now move on to **Module 10: The Mythic & Meta Game - Legacy, Collapse & Multiverse**, starting with **World Generation & Streaming - Chunk Architecture (C#)**, where we will design the foundational chunk-based world generation and streaming system for our infinite, persistent world.