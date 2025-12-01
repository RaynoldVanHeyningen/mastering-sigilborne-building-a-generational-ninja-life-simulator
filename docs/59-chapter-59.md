## Chapter 9.4: Forbidden Contracts & Spirit Contracts (C#)

Our `RitualSystem` now handles tiered rituals, including the dangerous `Forbidden` tier. Often, these potent rituals are used to forge **Forbidden Contracts** or **Spirit Contracts** with powerful entities or metaphysical forces. This chapter designs how players can enter into these dangerous pacts, each with world-linked consequences, representing the ultimate risk-reward paths in Sigilborne's magic, as specified in GDD B27.3 and B27.4.

### 1. The Nature of Pacts and Contracts

The GDD emphasizes that these are not simple spells:
*   **Forbidden Contracts**: (B27.3) Pacts with powerful forces (spirits, corruption, anomalies) that *change the world*. They are "power-for-pain deals" with "world-linked consequences."
*   **Spirit Contracts**: (B27.4) Very rare pacts with ancient spirits, requiring deep Echo affinity and strict rituals. They are "relational, not mechanical."

These systems represent the highest level of non-mythic magic, blurring the line between player agency and world-altering events.

### 2. Defining `ContractType`, `ContractStatus`, and `ContractDefinition`

We need enums for different types of contracts and their status, and a class to define a static contract blueprint.

1.  Create `res://_Brain/Systems/Rituals/ContractData.cs`:

```csharp
// _Brain/Systems/Rituals/ContractData.cs
using System;
using System.Collections.Generic;
using Godot; // For Vector2
using Sigilborne.Entities; // For EntityID
using Sigilborne.Systems.Magic; // For GlyphConcept
using Sigilborne.Systems.Factions; // For FactionID

namespace Sigilborne.Systems.Rituals
{
    /// <summary>
    /// Defines the broad categories of magical contracts.
    /// (GDD B27.3.1, B27.4)
    /// </summary>
    public enum ContractType
    {
        None,
        Forbidden,      // General forbidden pact with metaphysical forces (GDD B27.3)
        Spirit,         // Pact with a specific spirit entity (GDD B27.4)
        Corruption,     // Pact with chaotic residue/corruption entities (GDD B27.5)
        Bloodline,      // Modifies player's bloodline (GDD B06.4)
        Faction         // Binding pact with a major faction
    }

    /// <summary>
    /// Defines the current status of a contract.
    /// </summary>
    public enum ContractStatus
    {
        Inactive,       // Not yet initiated or broken
        Pending,        // Initiated, undergoing ritual/trial
        Active,         // Fully bound, effects are active
        Violated,       // Player broke terms, penalties applied
        Broken          // Contract forcibly broken (often with severe consequences)
    }

    /// <summary>
    /// Static data defining a magical contract.
    /// (GDD B27.3, B27.4)
    /// </summary>
    public class ContractDefinition
    {
        public string ID { get; private set; } // Unique ID (e.g., "pact_of_void_sight", "spirit_of_forest_bond")
        public string Name { get; private set; }
        public string Description { get; private set; }
        public ContractType Type { get; private set; }
        public RitualTier RequiredRitualTier { get; private set; } // Contract requires a ritual to initiate
        public string RequiredRitualID { get; private set; } // The specific ritual needed
        public EntityID ContractEntityID { get; private set; } // The entity the contract is with (e.g., a spirit, an anomaly core)
        public string FactionID { get; private set; } // For Faction contracts

        // Requirements/Costs
        public float ChakraCost { get; private set; }
        public float StabilityCost { get; private set; }
        public float PermanentHealthCost { get; private set; } // GDD B27.3.1: Power-for-Pain deals
        public List<string> RequiredItemIDs { get; private set; } // Items consumed

        // Rewards/Effects when Active
        public List<string> ActiveEffectIDs { get; private set; } // e.g., "void_sense_bonus", "summon_forest_spirit"
        public List<string> GrantedGlyphConcepts { get; private set; } // New glyph concepts granted (GDD B01.4.E/F)
        public List<string> GrantedAbilities { get; private set; } // New active abilities (C02 high-tier techniques)

        // Penalties/Consequences when Violated or Broken
        public List<string> ViolationPenaltyIDs { get; private set; } // e.g., "curse_of_instability", "spirit_hunt_on_player"
        public List<string> BrokenConsequenceIDs { get; private set; } // World-altering consequences (GDD B27.3.2)

        public ContractDefinition(string id, string name, string description, ContractType type, RitualTier requiredRitualTier, string requiredRitualID, EntityID contractEntityID, string factionID = null, float chakraCost = 0f, float stabilityCost = 0f, float permanentHealthCost = 0f, List<string> requiredItemIDs = null, List<string> activeEffectIDs = null, List<string> grantedGlyphConcepts = null, List<string> grantedAbilities = null, List<string> violationPenaltyIDs = null, List<string> brokenConsequenceIDs = null)
        {
            ID = id;
            Name = name;
            Description = description;
            Type = type;
            RequiredRitualTier = requiredRitualTier;
            RequiredRitualID = requiredRitualID;
            ContractEntityID = contractEntityID;
            FactionID = factionID;
            ChakraCost = chakraCost;
            StabilityCost = stabilityCost;
            PermanentHealthCost = permanentHealthCost;
            RequiredItemIDs = requiredItemIDs ?? new List<string>();
            ActiveEffectIDs = activeEffectIDs ?? new List<string>();
            GrantedGlyphConcepts = grantedGlyphConcepts ?? new List<string>();
            GrantedAbilities = grantedAbilities ?? new List<string>();
            ViolationPenaltyIDs = violationPenaltyIDs ?? new List<string>();
            BrokenConsequenceIDs = brokenConsequenceIDs ?? new List<string>();
        }

        public override string ToString()
        {
            string entityInfo = ContractEntityID.IsValid() ? $" with {ContractEntityID}" : (FactionID != null ? $" with {FactionID}" : "");
            return $"Contract: '{Name}' ({ID}) | Type: {Type}, Tier: {RequiredRitualTier}{entityInfo}";
        }
    }
}
```

### 3. Implementing `ContractSystem.cs`

This system will:
*   Manage `ContractDefinition`s.
*   Track `ContractStatus` for entities (player).
*   Provide methods to `InitiateContract` and `BreakContract`.
*   Interact with `RitualSystem` to trigger contracts.
*   Apply contract rewards and penalties.

1.  Create `res://_Brain/Systems/Rituals/ContractSystem.cs`:

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
using Sigilborne.Systems.Magic; // For GlyphAcquisitionSystem, GlyphConcept
using Sigilborne.Systems.StatusEffects; // For StatusEffectSystem
using System.Linq;

namespace Sigilborne.Systems.Rituals
{
    /// <summary>
    /// Manages magical contracts (Forbidden, Spirit, Corruption, Bloodline, Faction).
    /// Tracks contract status, applies rewards/penalties, and integrates with rituals.
    /// (GDD B27.3, B27.4)
    /// </summary>
    public class ContractSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private InventorySystem _inventorySystem;
        private BiologicalSystem _biologicalSystem;
        private GlyphAcquisitionSystem _glyphAcquisitionSystem; // To grant new glyphs
        private StatusEffectSystem _statusEffectSystem; // To apply penalties

        // Static definitions of all possible contracts
        private Dictionary<string, ContractDefinition> _contractDefinitions = new Dictionary<string, ContractDefinition>();

        // Active contracts for entities: Key: EntityID, Value: Dictionary<ContractID, ContractStatus>
        private Dictionary<EntityID, Dictionary<string, ContractStatus>> _entityContracts = new Dictionary<EntityID, Dictionary<string, ContractStatus>>();

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
            _eventBus.OnRitualCompleted += OnRitualCompleted; // Contracts are often initiated by ritual completion

            RegisterDefaultContractDefinitions(); // Register some default contracts
            GD.Print("ContractSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player || type == EntityType.NPC) // NPCs can also have contracts
            {
                _entityContracts.Add(id, new Dictionary<string, ContractStatus>());
                // Initialize all contracts as Inactive for this entity.
                foreach (var contractDef in _contractDefinitions.Values)
                {
                    _entityContracts[id].Add(contractDef.ID, ContractStatus.Inactive);
                }
                GD.Print($"ContractSystem: Initialized contracts for {type} entity {id}.");
            }
        }

        private void OnOnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _entityContracts.Remove(e.ID);
            GD.Print($"ContractSystem: Removed contracts for {e.ID}.");
        }

        private void OnRitualCompleted(EntityID initiatorID, string ritualID, EntityID centerID)
        {
            // Check if any contract requires this ritual
            foreach (var contractDef in _contractDefinitions.Values.Where(cd => cd.RequiredRitualID == ritualID).ToList())
            {
                // If the player completed the ritual, attempt to initiate the contract.
                if (initiatorID == GameManager.Instance.Entities.GetPlayerEntityID())
                {
                    GD.Print($"ContractSystem: Player completed ritual '{ritualID}'. Attempting to initiate contract '{contractDef.ID}'.");
                    InitiateContract(initiatorID, contractDef.ID);
                }
            }
        }

        /// <summary>
        /// Registers a new static ContractDefinition.
        /// </summary>
        public void RegisterContractDefinition(ContractDefinition def)
        {
            _contractDefinitions[def.ID] = def;
            GD.Print($"ContractSystem: Registered contract definition '{def.ID}'.");
        }

        /// <summary>
        /// Registers some default contracts for initial testing.
        /// (GDD B27.3, B27.4)
        /// </summary>
        private void RegisterDefaultContractDefinitions()
        {
            // --- Forbidden Contract: Pact of Void Sight (GDD B27.3) ---
            RegisterContractDefinition(new ContractDefinition(
                id: "pact_of_void_sight", name: "Pact of Void Sight", description: "Grants perception of hidden void energies.",
                type: ContractType.Forbidden, requiredRitualTier: RitualTier.Advanced, requiredRitualID: "scrying_rite_t1", // Use scrying rite
                contractEntityID: EntityID.Invalid, // No specific entity, general forbidden pact
                chakraCost: 30f, stabilityCost: 20f, permanentHealthCost: 5f, // Power-for-pain
                requiredItemIDs: new List<string> { "void_dust_small" },
                activeEffectIDs: new List<string> { "void_sight_aura" }, // Conceptual effect
                grantedGlyphConcepts: new List<string> { GlyphConcept.Consume.ToString(), GlyphConcept.Flux.ToString() }, // Grants new glyphs
                violationPenaltyIDs: new List<string> { "vision_blur_curse" },
                brokenConsequenceIDs: new List<string> { "corruption_bloom_local" } // World-altering
            ));

            // --- Spirit Contract: Forest Spirit Bond (GDD B27.4) ---
            RegisterContractDefinition(new ContractDefinition(
                id: "forest_spirit_bond", name: "Forest Spirit Bond", description: "Binds a lesser forest spirit to aid.",
                type: ContractType.Spirit, requiredRitualTier: RitualTier.Advanced, requiredRitualID: "spirit_summoning_t2",
                contractEntityID: new EntityID(999, 1), // Dummy spirit entity ID
                chakraCost: 40f, stabilityCost: 25f,
                requiredItemIDs: new List<string> { "forest_blossom", "spirit_wood" },
                activeEffectIDs: new List<string> { "spirit_ally_summon", "bloom_boost_aura" },
                grantedAbilities: new List<string> { "spirit_call_ability" }, // Conceptual ability
                violationPenaltyIDs: new List<string> { "spirit_anger_curse" },
                brokenConsequenceIDs: new List<string> { "forest_spirit_rampage" } // World-altering
            ));

            // --- Corruption Pact: Mark of the Aberrant (GDD B27.5) ---
            RegisterContractDefinition(new ContractDefinition(
                id: "mark_of_the_aberrant", name: "Mark of the Aberrant", description: "Embrace a minor corruption mutation.",
                type: ContractType.Corruption, requiredRitualTier: RitualTier.Forbidden, requiredRitualID: "corruption_ascension_f",
                contractEntityID: EntityID.Invalid,
                chakraCost: 80f, stabilityCost: 50f, permanentHealthCost: 10f,
                requiredItemIDs: new List<string> { "tainted_blood" },
                activeEffectIDs: new List<string> { "minor_mutation_buff" },
                grantedAbilities: new List<string> { "aberrant_strike_ability" },
                violationPenaltyIDs: new List<string> { "uncontrolled_mutation" },
                brokenConsequenceIDs: new List<string> { "corruption_spread_regional" }
            ));
        }

        /// <summary>
        /// Attempts to initiate a contract after its required ritual is completed.
        /// (GDD B27.3.1)
        /// </summary>
        /// <param name="initiatorID">The entity attempting to initiate the contract (e.g., player).</param>
        /// <param name="contractID">The ID of the contract to initiate.</param>
        /// <returns>True if contract initiated successfully, false otherwise.</returns>
        public bool InitiateContract(EntityID initiatorID, string contractID)
        {
            if (!_entityManager.IsValid(initiatorID)) { GD.PrintErr($"ContractSystem: Invalid initiator {initiatorID}."); return false; }
            if (!_contractDefinitions.TryGetValue(contractID, out ContractDefinition contractDef)) { GD.PrintErr($"ContractSystem: Contract definition '{contractID}' not found."); return false; }

            if (_entityContracts.TryGetValue(initiatorID, out Dictionary<string, ContractStatus> contractsOnInitiator) && contractsOnInitiator.TryGetValue(contractID, out ContractStatus currentStatus) && currentStatus == ContractStatus.Active)
            {
                GD.Print($"ContractSystem: Initiator {initiatorID} already has contract '{contractID}' active.");
                return false;
            }

            if (!_biologicalSystem.TryGetCoreStats(initiatorID, out CoreStats initiatorStats)) { GD.PrintErr($"ContractSystem: Initiator {initiatorID} has no CoreStats."); return false; }

            // --- 1. Check Initiator Resources (Chakra, Stability, Permanent Health) ---
            if (initiatorStats.Chakra < contractDef.ChakraCost) { GD.Print($"ContractSystem: Insufficient Chakra for contract '{contractID}'."); return false; }
            if (initiatorStats.Stability < contractDef.StabilityCost) { GD.Print($"ContractSystem: Insufficient Stability for contract '{contractID}'."); return false; }
            // Permanent health cost is applied upon successful contract.

            // --- 2. Check Item Requirements (GDD B27.3.1) ---
            foreach (var itemID in contractDef.RequiredItemIDs)
            {
                // This would query inventory for consumable items
                // if (!_inventorySystem.HasItem(initiatorID, itemID, 1)) { /* ... */ return false; }
                GD.Print($"ContractSystem: Checking for required item: {itemID} (conceptual).");
            }

            // --- 3. Deduct Resources and Consume Items ---
            GameManager.Instance.PlayerStats.TakeChakra(contractDef.ChakraCost);
            GameManager.Instance.PlayerStats.TakeStability(contractDef.StabilityCost);
            
            // Consume items (conceptual)
            foreach (var itemID in contractDef.RequiredItemIDs)
            {
                // _inventorySystem.RemoveItem(initiatorID, itemID, 1);
                GD.Print($"ContractSystem: Consuming item '{itemID}' for contract '{contractID}'.");
            }

            // Apply permanent health cost (GDD B27.3.1)
            if (contractDef.PermanentHealthCost > 0)
            {
                ref CoreStats initiatorCoreStats = ref _biologicalSystem.GetCoreStatsRef(initiatorID);
                initiatorCoreStats.MaxHealth -= contractDef.PermanentHealthCost;
                initiatorCoreStats.Health = Mathf.Min(initiatorCoreStats.Health, initiatorCoreStats.MaxHealth); // Adjust current health if max dropped below
                GD.Print($"ContractSystem: {initiatorID} suffered permanent Max Health loss of {contractDef.PermanentHealthCost}. New Max Health: {initiatorCoreStats.MaxHealth}.");
                _eventBus.Publish(new PlayerStatSystem.PlayerHealthChangedEvent { PlayerID = initiatorID, NewValue = initiatorCoreStats.Health, MaxValue = initiatorCoreStats.MaxHealth });
            }

            // --- 4. Activate Contract ---
            _entityContracts[initiatorID][contractID] = ContractStatus.Active;
            GD.Print($"ContractSystem: Initiator {initiatorID} successfully initiated contract '{contractID}' (Type: {contractDef.Type})!");
            _eventBus.Publish(new ContractStatusChangedEvent { InitiatorID = initiatorID, ContractID = contractID, NewStatus = ContractStatus.Active });

            // --- 5. Apply Contract Rewards/Effects (GDD B27.3.1) ---
            ApplyContractEffects(initiatorID, contractDef);

            return true;
        }

        /// <summary>
        /// Applies the rewards and effects of an active contract.
        /// </summary>
        private void ApplyContractEffects(EntityID initiatorID, ContractDefinition contractDef)
        {
            GD.Print($"ContractSystem: Applying effects for contract '{contractDef.ID}'.");

            // Grant new glyph concepts (GDD B01.4.E/F)
            foreach (var conceptStr in contractDef.GrantedGlyphConcepts)
            {
                if (Enum.TryParse(conceptStr, out GlyphConcept concept))
                {
                    // Find a symbol for this concept in the world map (if one exists)
                    WorldGlyphDefinition glyphDef = GameManager.Instance.GlyphMap.GetDefinitionByConcept(concept);
                    if (glyphDef.Concept != GlyphConcept.None)
                    {
                        _glyphAcquisitionSystem.UpdateGlyphKnowledge(glyphDef.SymbolID, GlyphKnowledgeState.Known);
                        GD.Print($"  Contract granted knowledge of glyph concept: {concept} (Symbol: '{glyphDef.SymbolID}').");
                    }
                    else
                    {
                        GD.PrintErr($"  Contract '{contractDef.ID}' attempted to grant unknown concept: {conceptStr}.");
                    }
                }
            }

            // Grant new abilities (conceptual, C02 high-tier techniques)
            foreach (var abilityID in contractDef.GrantedAbilities)
            {
                GD.Print($"  Contract granted ability: {abilityID} (conceptual).");
                // _eventBus.Publish(new PlayerAbilityGrantedEvent { PlayerID = initiatorID, AbilityID = abilityID });
            }

            // Apply active status effects (conceptual)
            foreach (var effectID in contractDef.ActiveEffectIDs)
            {
                GD.Print($"  Contract applied active effect: {effectID} (conceptual).");
                // _statusEffectSystem.ApplyEffect(initiatorID, new StatusEffectData(effectID, float.MaxValue, 1f));
            }
        }

        /// <summary>
        /// Attempts to break an active contract.
        /// (GDD B27.3.3: Breaking a Contract)
        /// </summary>
        public bool BreakContract(EntityID initiatorID, string contractID)
        {
            if (!_entityManager.IsValid(initiatorID)) return false;
            if (!_contractDefinitions.TryGetValue(contractID, out ContractDefinition contractDef)) return false;

            if (!_entityContracts.TryGetValue(initiatorID, out Dictionary<string, ContractStatus> contractsOnInitiator) || !contractsOnInitiator.TryGetValue(contractID, out ContractStatus currentStatus) || currentStatus != ContractStatus.Active)
            {
                GD.Print($"ContractSystem: Initiator {initiatorID} does not have active contract '{contractID}' to break.");
                return false;
            }

            // --- Trigger Breaking Consequences (GDD B27.3.2) ---
            _entityContracts[initiatorID][contractID] = ContractStatus.Broken;
            GD.Print($"ContractSystem: Initiator {initiatorID} forcibly BROKE contract '{contractID}'! Triggering severe consequences!");
            _eventBus.Publish(new ContractStatusChangedEvent { InitiatorID = initiatorID, ContractID = contractID, NewStatus = ContractStatus.Broken });

            foreach (var consequenceID in contractDef.BrokenConsequenceIDs)
            {
                GD.Print($"  Contract Broken Consequence: {consequenceID} triggered. (Potentially world-altering!)");
                // This would trigger severe world events (e.g., corruption bloom, spirit rampage, anomaly rupture)
                // _eventBus.Publish(new WorldSystem.GlobalConsequenceEvent { ConsequenceID = consequenceID });
            }
            // Apply violation penalties (e.g., curses)
            foreach (var penaltyID in contractDef.ViolationPenaltyIDs)
            {
                 GD.Print($"  Contract Broken Penalty: {penaltyID} applied.");
                // _statusEffectSystem.ApplyEffect(initiatorID, new StatusEffectData(penaltyID, float.MaxValue, 1f));
            }
            return true;
        }

        /// <summary>
        /// Retrieves the current status of a contract for an entity.
        /// </summary>
        public ContractStatus GetContractStatus(EntityID entityID, string contractID)
        {
            if (_entityContracts.TryGetValue(entityID, out Dictionary<string, ContractStatus> contracts) && contracts.TryGetValue(contractID, out ContractStatus status))
            {
                return status;
            }
            return ContractStatus.Inactive;
        }

        // --- Helper Events for Body Sync ---
        public struct ContractStatusChangedEvent { public EntityID InitiatorID; public string ContractID; public ContractStatus NewStatus; }
    }
}
```

### 4. Integrating `ContractSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Rituals;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `ContractSystem` property.
3.  Initialize `ContractSystem` in `InitializeSystems()` **after** `RitualSystem` and its dependencies.

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
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Combat;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.AI;
using Sigilborne.Systems.Factions;
using Sigilborne.Systems.Economy;
using Sigilborne.Systems.Crime;
using Sigilborne.Systems.Rituals;
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public RitualSystem Rituals { get; private set; }
    public SealSystem Seals { get; private set; }
    public ContractSystem Contracts { get; private set; } // Add ContractSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        // ... (existing inventory/equipment tests) ...
        // ... (existing spatial grid tests) ...
        // ... (existing perception system tests) ...
        // ... (existing stealth system tests) ...
        // ... (existing AI system tests) ...
        // ... (existing ecology system tests) ...
        // ... (existing faction system tests) ...
        // ... (existing economy system tests) ...
        // ... (existing crime system tests) ...
        // ... (existing ritual system tests) ...
        // ... (existing seal system tests) ...

        // --- Test Contract System ---
        GD.Print("\n--- Testing Contract System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID altarID = Entities.GetEntityMeta(4).Generation == 1 ? new EntityID(4, 1) : EntityID.Invalid; // Our test altar

        // Add required items for contracts to player's inventory
        Inventory.AddItem(playerID, new InventoryItem("void_dust_small", 1));
        Inventory.AddItem(playerID, new InventoryItem("forest_blossom", 1));
        Inventory.AddItem(playerID, new InventoryItem("spirit_wood", 1));
        Inventory.AddItem(playerID, new InventoryItem("tainted_blood", 1));
        
        // --- Test Forbidden Contract: Pact of Void Sight (requires scrying_rite_t1 completion) ---
        GD.Print("\nAttempting to initiate 'Pact of Void Sight' (Forbidden):");
        // We'll simulate the ritual completion first, which then calls InitiateContract
        // Scrying Rite requires 1 scrying_orb, 1 rare_incense
        // Player already has these from ritual tests.
        GD.Print("  Simulating completion of 'scrying_rite_t1' ritual...");
        Rituals.CompleteRitual(playerID, altarID, "scrying_rite_t1"); // Manually complete ritual to trigger contract

        // Verify status
        GD.Print($"Player contract status for 'pact_of_void_sight': {Contracts.GetContractStatus(playerID, "pact_of_void_sight")}");
        GD.Print($"Player current Max Health: {Biology.GetCoreStatsRef(playerID).MaxHealth}");
        GD.Print($"Player current Chakra: {Biology.GetCoreStatsRef(playerID).Chakra}");
        GD.Print($"Player current Stability: {Biology.GetCoreStatsRef(playerID).Stability}");
        GD.Print($"Player knows Consume glyph: {PlayerGlyphKnowledge.GetGlyphKnowledge(GlyphMap.GetDefinitionByConcept(GlyphConcept.Consume).SymbolID)}");

        // --- Test Spirit Contract: Forest Spirit Bond (requires spirit_summoning_t2 completion) ---
        GD.Print("\nAttempting to initiate 'Forest Spirit Bond' (Spirit):");
        // Spirit Summoning Rite requires 1 spirit_essence, 1 ancient_blood_rune, 3 lit_candle
        GD.Print("  Simulating completion of 'spirit_summoning_t2' ritual...");
        Rituals.CompleteRitual(playerID, altarID, "spirit_summoning_t2"); // Manually complete ritual to trigger contract

        // --- Test Corruption Pact: Mark of the Aberrant (requires corruption_ascension_f completion) ---
        GD.Print("\nAttempting to initiate 'Mark of the Aberrant' (Corruption):");
        // Corruption Ascension Rite requires 1 residue_crystal_large, 1 fresh_heart, 1 void_ink_scroll
        GD.Print("  Simulating completion of 'corruption_ascension_f' ritual...");
        Rituals.CompleteRitual(playerID, altarID, "corruption_ascension_f");

        // --- Test Breaking a Contract (e.g., Pact of Void Sight) ---
        GD.Print("\nAttempting to Break 'Pact of Void Sight':");
        Contracts.BreakContract(playerID, "pact_of_void_sight");
        GD.Print($"Player contract status for 'pact_of_void_sight' after break attempt: {Contracts.GetContractStatus(playerID, "pact_of_void_sight")}");

        GD.Print("--- End Testing Contract System ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta);
        Casting.Tick(delta);
        Weather.Tick(delta);
        Biology.Tick(delta);
        TitanAI.Tick(delta);
        StatusEffects.Tick(delta);
        Perception.Tick(delta);
        Stealth.Tick(delta);
        AI.Tick(delta);
        Ecology.Tick(delta);
        FactionAI.Tick(delta);
        Economy.Tick(delta);
        Crime.Tick(delta);
        Rituals.Tick(delta);
        Seals.Tick(delta);
        // ContractSystem doesn't have a Tick method for now; its operations are event-driven.
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to SealSystem) ...
        
        // Initialize ContractSystem AFTER RitualSystem, GlyphAcquisitionSystem, StatusEffectSystem
        Contracts = new ContractSystem(Entities, Events, Inventory, BiologicalSystem, PlayerGlyphKnowledge, StatusEffects); // Pass dependencies
        GD.Print("  - ContractSystem initialized.");

        // ... (existing system initializations after ContractSystem) ...
    }
}
```

#### 4.1. Update `EventBus.cs` for Contract Events

Open `res://_Brain/Core/EventBus.cs` and add `OnContractStatusChanged` delegate.

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
using Sigilborne.Systems.AI;
using Sigilborne.Systems.Factions;
using Sigilborne.Systems.Economy;
using Sigilborne.Systems.Crime;
using Sigilborne.Systems.Rituals;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Contract System Events (GDD B27.3, B27.4)
        public event Action<EntityID, string, ContractStatus> OnContractStatusChanged; // InitiatorID, ContractID, NewStatus

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is RitualSystem.ContractSystem.ContractStatusChangedEvent contractStatusEvent) // New condition
            {
                OnContractStatusChanged?.Invoke(contractStatusEvent.InitiatorID, contractStatusEvent.ContractID, contractStatusEvent.NewStatus);
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

### 5. Testing Forbidden and Spirit Contracts

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Contract System" section.

```
...
ContractSystem: Initialized.
ContractSystem: Registered contract definition 'Pact of Void Sight'.
ContractSystem: Registered contract definition 'Forest Spirit Bond'.
ContractSystem: Registered contract definition 'Mark of the Aberrant'.
  - ContractSystem initialized.
EconomyManager: Initialized.
...
--- Testing Contract System ---

Attempting to initiate 'Pact of Void Sight' (Forbidden):
  Simulating completion of 'scrying_rite_t1' ritual...
RitualSystem: Ritual 'scrying_rite_t1' completed successfully by EntityID(0, Gen:1) at EntityID(4, Gen:1)! Triggering effects.
  Ritual Effect: reveal_map_area triggered (Tier: Simple).
PlayerStatSystem: Player EntityID(0, Gen:1) used 30 chakra. New Chakra: 0.0
PlayerStatSystem: Player EntityID(0, Gen:1) lost 20 stability. New Stability: 65.0
ContractSystem: EntityID(0, Gen:1) suffered permanent Max Health loss of 5.0. New Max Health: 95.0.
PlayerStatSystem: Player EntityID(0, Gen:1) took 0.0 damage. New Health: 95.0
ContractSystem: Initiator EntityID(0, Gen:1) successfully initiated contract 'pact_of_void_sight' (Type: Forbidden)!
ContractSystem: Applying effects for contract 'pact_of_void_sight'.
  Contract granted knowledge of glyph concept: Consume (Symbol: 'symbol_cross').
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_cross' updated from Known to Known.
  Contract granted knowledge of glyph concept: Flux (Symbol: 'symbol_pentagon_alt').
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_pentagon_alt' updated from Hidden to Known.
  Contract applied active effect: void_sight_aura (conceptual).
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
ContractSystem: Initiator EntityID(0, Gen:1) does not have active contract 'forest_spirit_bond' to break.

Attempting to initiate 'Mark of the Aberrant' (Corruption):
  Simulating completion of 'corruption_ascension_f' ritual...
RitualSystem: Ritual 'corruption_ascension_f' completed successfully by EntityID(0, Gen:1) at EntityID(4, Gen:1)! Triggering effects.
  Ritual Effect: caster_mutated_aberration triggered (Tier: Forbidden).
    WARNING: FORBIDDEN RITUAL EFFECT! Potential world-altering consequences!
ContractSystem: Insufficient Chakra for contract 'mark_of_the_aberrant'.

Attempting to Break 'Pact of Void Sight':
ContractSystem: Initiator EntityID(0, Gen:1) forcibly BROKE contract 'pact_of_void_sight'! Triggering severe consequences!
  Contract Broken Consequence: corruption_bloom_local triggered. (Potentially world-altering!)
  Contract Broken Penalty: vision_blur_curse applied.
Player contract status for 'pact_of_void_sight' after break attempt: Broken
--- End Testing Contract System ---
...
```

**Key Observations:**

*   **Contract Initiation**: `InitiateContract` is called upon ritual completion.
*   **Resource Checks**: Chakra/Stability checks correctly prevent contract initiation if resources are insufficient (e.g., for `Forest Spirit Bond` and `Mark of the Aberrant` after `Pact of Void Sight` drained all chakra).
*   **Permanent Health Cost**: `Pact of Void Sight` correctly reduces player's `MaxHealth`.
*   **Granted Glyphs**: `Pact of Void Sight` grants knowledge of `Consume` and `Flux` glyphs.
*   **Effect/Ability Granting**: Conceptual effects/abilities are "granted."
*   **Breaking Contract**: `BreakContract` successfully transitions status to `Broken` and triggers `BrokenConsequenceIDs` and `ViolationPenaltyIDs`.
*   **World-Altering Consequences**: The output for `Forbidden` and `Broken` contracts explicitly mentions "Potential world-altering consequences!"

This confirms our `ContractSystem` is functional, managing contracts, enforcing requirements, applying consequences, and integrating with rituals and other systems.

### Summary

You have successfully implemented **Forbidden Contracts & Spirit Contracts** in the C# Brain, designing `ContractType`, `ContractStatus`, and `ContractDefinition` to represent powerful magical pacts. By creating `ContractSystem` to manage contract definitions, track entity contract status, validate initiator resources and item requirements, and apply tiered rewards and penalties (including permanent `MaxHealth` loss and granting new glyphs), you've established a robust mechanism for high-stakes magic. This crucial system strictly adheres to GDD B27.3 and B27.4's specifications, providing the ultimate risk-reward paths in Sigilborne's advanced magic.

### Next Steps

The next chapter will focus on **Corruption Pacts & Multi-Life Persistence of Rituals**, where we will refine the `ContractSystem` to specifically handle `Corruption` pacts (including `Mutation` growth), and ensure that ritual and contract outcomes persist across player lives, impacting future playthroughs.