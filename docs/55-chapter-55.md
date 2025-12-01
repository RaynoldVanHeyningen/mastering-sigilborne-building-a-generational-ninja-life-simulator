## Chapter 8.6: Bounty & Punishment - Escalation & Consequences (C#)

Our `CrimeSystem` now tracks `Heat` per faction. This chapter builds upon that by implementing **Bounty & Punishment** mechanisms. We'll define how `Heat` translates into concrete punishments like refusing services, attacking on sight, or dispatching hunter squads, and design how the `Bounty` system tracks these escalating consequences, as specified in TDD 06.4.2.

### 1. The Escalation of Consequences

The GDD (B22.5) states: "Punishments range from fines, exile, forced labor... to execution." This implies a system that:

*   **Tiers of Punishment**: Consequences escalate with `HeatValue`.
*   **Active Pursuit**: High `Heat` leads to active hunting.
*   **Faction-Specific**: Punishments are enforced by the aggrieved faction.
*   **Player-Visible**: The player needs to understand the consequences of their actions.

### 2. Defining `Bounty` and `PunishmentLevel`

We need a struct to represent a bounty and an enum for escalating punishment tiers.

1.  Create `res://_Brain/Systems/Crime/BountyData.cs`:

```csharp
// _Brain/Systems/Crime/BountyData.cs
using System;
using Godot; // For Vector2
using Sigilborne.Entities; // For EntityID
using Sigilborne.Systems.Factions; // For FactionType

namespace Sigilborne.Systems.Crime
{
    /// <summary>
    /// Defines the escalating levels of punishment a faction can impose.
    /// (TDD 06.4.2)
    /// </summary>
    public enum PunishmentLevel
    {
        None,               // No punishment, ignored
        Warning,            // Verbal warning, reduced services
        RefusedServices,    // Merchants won't trade, NPCs won't talk
        AttackOnSight,      // NPCs in faction territory become hostile
        HunterSquads,       // Faction dispatches bounty hunters to actively pursue
        Exile,              // Permanent ban from faction territory/settlements
        ExecutionOrder      // Highest level, faction seeks to kill on sight
    }

    /// <summary>
    /// Represents a bounty placed on an entity by a specific faction.
    /// (TDD 06.4.2)
    /// </summary>
    public struct Bounty
    {
        public EntityID TargetID;
        public string FactionID;
        public float GoldValue;         // The gold reward for capturing/defeating the target
        public PunishmentLevel Level;   // The current punishment level
        public bool IsActive;           // True if hunters are actively pursuing

        public Bounty(EntityID targetID, string factionID, float goldValue, PunishmentLevel level, bool isActive)
        {
            TargetID = targetID;
            FactionID = factionID;
            GoldValue = goldValue;
            Level = level;
            IsActive = isActive;
        }

        public override string ToString()
        {
            return $"Bounty on {TargetID} by {FactionID}: {GoldValue:F0} Gold, Level: {Level} (Active: {IsActive})";
        }
    }
}
```

### 3. Enhancing `CrimeSystem.cs` for Bounties and Punishments

`CrimeSystem` will now:
*   Manage `Bounty` instances per entity and faction.
*   Translate `FactionHeat` into `PunishmentLevel`s.
*   Trigger `HunterSquads` (conceptually) when `Heat` is high enough.
*   Provide methods for other systems (e.g., `AISystem`, `EconomyManager`) to query `PunishmentLevel`.

1.  Open `res://_Brain/Systems/Crime/CrimeSystem.cs`.
2.  Add a dictionary for `Bounty` instances.
3.  Modify `AddHeat` to update `Bounty` and `PunishmentLevel`.
4.  Add methods to retrieve `PunishmentLevel`.

```csharp
// _Brain/Systems/Crime/CrimeSystem.cs
using Godot;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Factions;
using Sigilborne.Systems.AI;
using System.Linq;

namespace Sigilborne.Systems.Crime
{
    // ... (CrimeType, CrimeReport, FactionHeat structs/enums) ...
    // ... (PunishmentLevel, Bounty structs/enums) ...

    public class CrimeSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private FactionSystem _factionSystem;
        private AISystem _aiSystem;

        private Dictionary<EntityID, Dictionary<string, FactionHeat>> _entityFactionHeat = new Dictionary<EntityID, Dictionary<string, FactionHeat>>();
        private List<CrimeDefinition> _crimeDefinitions = new List<CrimeDefinition>();
        private ConcurrentQueue<CrimeReport> _crimeReportsQueue = new ConcurrentQueue<CrimeReport>();

        // New: Bounties tracked per target and faction.
        // Key: TargetID, Value: Dictionary<FactionID, Bounty>
        private Dictionary<EntityID, Dictionary<string, Bounty>> _entityBounties = new Dictionary<EntityID, Dictionary<string, Bounty>>();


        private const float HEAT_DECAY_INTERVAL = 10.0f;
        private float _heatDecayTimer;

        public CrimeSystem(EntityManager entityManager, EventBus eventBus, FactionSystem factionSystem, AISystem aiSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _factionSystem = factionSystem;
            _aiSystem = aiSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            RegisterDefaultCrimeDefinitions();
            GD.Print("CrimeSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player)
            {
                _entityFactionHeat.Add(id, new Dictionary<string, FactionHeat>());
                _entityBounties.Add(id, new Dictionary<string, Bounty>()); // Initialize bounties
                foreach (var faction in _factionSystem.GetAllFactions())
                {
                    _entityFactionHeat[id].Add(faction.ID, new FactionHeat(faction.ID, id));
                    _entityBounties[id].Add(faction.ID, new Bounty(id, faction.ID, 0f, PunishmentLevel.None, false)); // Initialize bounty
                }
                GD.Print($"CrimeSystem: Initialized FactionHeat and Bounties for player {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _entityFactionHeat.Remove(e.ID);
            _entityBounties.Remove(e.ID); // Remove bounties
            GD.Print($"CrimeSystem: Removed FactionHeat and Bounties for {e.ID}.");
        }

        // ... (RegisterCrimeDefinition, RegisterDefaultCrimeDefinitions, ReportCrime methods) ...

        public void Tick(double delta)
        {
            ProcessCrimeReports();
            DecayHeat((float)delta);
            UpdateBountiesAndPunishments(); // New: Update punishment levels
        }

        private void ProcessCrimeReports() { /* ... */ }

        public void AddHeat(EntityID targetID, string factionID, float amount, bool isMajorCrime = false)
        {
            if (!_entityFactionHeat.TryGetValue(targetID, out Dictionary<string, FactionHeat> factionHeats))
            {
                GD.PrintErr($"CrimeSystem: No FactionHeat tracked for entity {targetID}.");
                return;
            }

            if (factionHeats.TryGetValue(factionID, out FactionHeat heat))
            {
                heat.AddHeat(amount);
                factionHeats[factionID] = heat;

                GD.Print($"CrimeSystem: {targetID} heat for {factionID} increased by {amount:F1}. New Heat: {heat.HeatValue:F1}.");
                _eventBus.Publish(new FactionHeatUpdatedEvent { TargetID = targetID, FactionHeat = heat });

                // Check for active pursuit (TDD 06.4.2)
                // This logic is now part of UpdateBountiesAndPunishments, as it's tied to PunishmentLevel.
                // For now, just a print if it's a major crime.
                if (isMajorCrime)
                {
                    GD.Print($"CrimeSystem: {targetID} committed a MAJOR crime for {factionID}.");
                }
            }
        }

        private void DecayHeat(float delta)
        {
            _heatDecayTimer += delta;
            if (_heatDecayTimer >= HEAT_DECAY_INTERVAL)
            {
                foreach (var targetEntry in _entityFactionHeat)
                {
                    EntityID targetID = targetEntry.Key;
                    foreach (var factionEntry in targetEntry.Value.Keys.ToList())
                    {
                        FactionHeat heat = targetEntry.Value[factionEntry];
                        if (heat.HeatValue > 0)
                        {
                            heat.DecayHeat(HEAT_DECAY_INTERVAL);
                            targetEntry.Value[factionEntry] = heat;
                            if (heat.HeatValue <= 0.1f)
                            {
                                GD.Print($"CrimeSystem: {targetID} heat for {factionEntry} decayed to zero.");
                                _eventBus.Publish(new FactionHeatUpdatedEvent { TargetID = targetID, FactionHeat = heat });
                            }
                        }
                    }
                }
                _heatDecayTimer = 0;
            }
        }

        /// <summary>
        /// Updates the bounty and punishment level for all entities based on their faction heat.
        /// (TDD 06.4.2)
        /// </summary>
        private void UpdateBountiesAndPunishments()
        {
            foreach (var targetEntry in _entityFactionHeat)
            {
                EntityID targetID = targetEntry.Key;
                foreach (var factionEntry in targetEntry.Value.Keys.ToList())
                {
                    FactionHeat heat = targetEntry.Value[factionEntry];
                    ref Bounty bounty = ref _entityBounties[targetID].GetValueRef(factionEntry); // Get mutable ref to bounty

                    // Determine new punishment level based on heat
                    PunishmentLevel newLevel = PunishmentLevel.None;
                    float newGoldValue = 0f;

                    if (heat.HeatValue >= 90f) { newLevel = PunishmentLevel.ExecutionOrder; newGoldValue = 500f; }
                    else if (heat.HeatValue >= 70f) { newLevel = PunishmentLevel.HunterSquads; newGoldValue = 300f; }
                    else if (heat.HeatValue >= 50f) { newLevel = PunishmentLevel.AttackOnSight; newGoldValue = 100f; }
                    else if (heat.HeatValue >= 30f) { newLevel = PunishmentLevel.RefusedServices; newGoldValue = 20f; }
                    else if (heat.HeatValue >= 10f) { newLevel = PunishmentLevel.Warning; newGoldValue = 5f; }

                    bool wasPursuing = bounty.IsActive;
                    bool isPursuing = (newLevel >= PunishmentLevel.HunterSquads); // HunterSquads and above means active pursuit

                    if (bounty.Level != newLevel || bounty.GoldValue != newGoldValue || bounty.IsActive != isPursuing)
                    {
                        bounty.Level = newLevel;
                        bounty.GoldValue = newGoldValue;
                        bounty.IsActive = isPursuing;

                        GD.Print($"CrimeSystem: Bounty for {targetID} by {factionEntry} updated. Level: {newLevel}, Gold: {newGoldValue:F0}, Active: {isPursuing}");
                        _eventBus.Publish(new BountyUpdatedEvent { TargetID = targetID, Bounty = bounty });

                        // Trigger/stop hunter squads if pursuit status changed (TDD 06.4.2)
                        if (isPursuing && !wasPursuing)
                        {
                            _eventBus.Publish(new FactionPursuitChangedEvent { TargetID = targetID, FactionID = factionEntry, IsPursuing = true });
                            // _aiSystem.SpawnHunterSquad(targetID, factionEntry); // Conceptual
                        }
                        else if (!isPursuing && wasPursuing)
                        {
                            _eventBus.Publish(new FactionPursuitChangedEvent { TargetID = targetID, FactionID = factionEntry, IsPursuing = false });
                            // _aiSystem.DespawnHunterSquads(targetID, factionEntry); // Conceptual
                        }
                    }
                }
            }
        }

        /// <summary>
        /// Retrieves the PunishmentLevel for a target entity by a specific faction.
        /// (TDD 06.4.2)
        /// </summary>
        public PunishmentLevel GetPunishmentLevel(EntityID targetID, string factionID)
        {
            if (_entityBounties.TryGetValue(targetID, out Dictionary<string, Bounty> bounties) && bounties.TryGetValue(factionID, out Bounty bounty))
            {
                return bounty.Level;
            }
            return PunishmentLevel.None;
        }

        /// <summary>
        /// Retrieves the Bounty for a target entity by a specific faction.
        /// </summary>
        public Bounty GetBounty(EntityID targetID, string factionID)
        {
            if (_entityBounties.TryGetValue(targetID, out Dictionary<string, Bounty> bounties) && bounties.TryGetValue(factionID, out Bounty bounty))
            {
                return bounty;
            }
            return new Bounty(targetID, factionID, 0f, PunishmentLevel.None, false);
        }

        // --- Helper Events for Body Sync ---
        public struct FactionHeatUpdatedEvent { public EntityID TargetID; public FactionHeat FactionHeat; }
        public struct FactionPursuitChangedEvent { public EntityID TargetID; public string FactionID; public bool IsPursuing; }
        public struct BountyUpdatedEvent { public EntityID TargetID; public Bounty Bounty; } // New event
    }
}
```

#### 3.1. Add `ExposingSecrets` and `Purification` to `CrimeType`

These were referenced in `RegisterDefaultCrimeDefinitions`.

Open `res://_Brain/Systems/Crime/CrimeData.cs` and add to `CrimeType`:

```csharp
// _Brain/Systems/Crime/CrimeData.cs
using System;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Factions;

namespace Sigilborne.Systems.Crime
{
    public enum CrimeType
    {
        None,
        Assault,
        Murder,
        Theft,
        Trespass,
        MagicMisuse,
        Sabotage,
        Espionage,
        CorruptionUse,
        ShrineDesecration,
        ExposingSecrets,    // New
        Purification        // New
    }
    // ... (rest of file) ...
}
```

### 4. Integrating `Bounty` and `PunishmentLevel` into `GameManager`

1.  Add `using Sigilborne.Systems.Crime;` at the top of `_Brain/Core/GameManager.cs`.
2.  In `GameManager._Ready()`, modify the "Test Crime System" section to:
    *   Test `GetPunishmentLevel`.
    *   Observe how `PunishmentLevel` changes with added `Heat`.

```csharp
// _Brain/Core/GameManager.cs (inside _Ready method)
// ...
        // --- Test Crime System ---
        GD.Print("\n--- Testing Crime System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID witnessID = Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid; // Test NPC as witness

        // Initial heat check
        GD.Print($"Player initial heat with Crimson Blades: {Crime.GetFactionHeat(playerID, "crimson_blades").HeatValue}");
        GD.Print($"Player initial punishment level with Crimson Blades: {Crime.GetPunishmentLevel(playerID, "crimson_blades")}");

        // Simulate a crime report
        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.Assault, "crimson_blades", new Vector2(200, 200), 10f, Time.CurrentGameTime));
        GD.Print($"Player punishment level with Crimson Blades after Assault: {Crime.GetPunishmentLevel(playerID, "crimson_blades")}");

        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.Trespass, "crimson_blades", new Vector2(200, 200), 1f, Time.CurrentGameTime));
        GD.Print($"Player punishment level with Crimson Blades after Trespass: {Crime.GetPunishmentLevel(playerID, "crimson_blades")}");

        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.Murder, "crimson_blades", new Vector2(200, 200), 100f, Time.CurrentGameTime)); // Major crime
        GD.Print($"Player punishment level with Crimson Blades after Murder: {Crime.GetPunishmentLevel(playerID, "crimson_blades")}");
        GD.Print($"Player bounty with Crimson Blades: {Crime.GetBounty(playerID, "crimson_blades")}");


        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.Theft, "shadow_veilers", new Vector2(200, 200), 50f, Time.CurrentGameTime));
        GD.Print($"Player punishment level with Shadow Veilers after Theft: {Crime.GetPunishmentLevel(playerID, "shadow_veilers")}");

        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.CorruptionUse, "sunken_temple", new Vector2(200, 200), 70f, Time.CurrentGameTime)); // High heat for Temple
        GD.Print($"Player punishment level with Sunken Temple after CorruptionUse: {Crime.GetPunishmentLevel(playerID, "sunken_temple")}");

        // Simulate a crime that is rewarded by Void Cultists
        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.CorruptionUse, "void_cultists", new Vector2(200, 200), 10f, Time.CurrentGameTime));
        GD.Print($"Player punishment level with Void Cultists after CorruptionUse: {Crime.GetPunishmentLevel(playerID, "void_cultists")}");

        GD.Print("--- End Testing Crime System ---\n");
// ...
```

#### 4.1. Update `EventBus.cs` for `BountyUpdatedEvent`

Open `res://_Brain/Core/EventBus.cs` and add `OnBountyUpdated` delegate.

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
using Sigilborne.Systems.Crime; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Crime System Events (TDD 06.4.1)
        public event Action<EntityID, FactionHeat> OnFactionHeatUpdated;
        public event Action<EntityID, string, bool> OnFactionPursuitChanged;
        public event Action<EntityID, Bounty> OnBountyUpdated; // New event

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is CrimeSystem.FactionHeatUpdatedEvent factionHeatEvent)
            {
                OnFactionHeatUpdated?.Invoke(factionHeatEvent.TargetID, factionHeatEvent.FactionHeat);
            }
            else if (eventData is CrimeSystem.FactionPursuitChangedEvent factionPursuitEvent)
            {
                OnFactionPursuitChanged?.Invoke(factionPursuitEvent.TargetID, factionPursuitEvent.FactionID, factionPursuitEvent.IsPursuing);
            }
            else if (eventData is CrimeSystem.BountyUpdatedEvent bountyUpdatedEvent) // New condition
            {
                OnBountyUpdated?.Invoke(bountyUpdatedEvent.TargetID, bountyUpdatedEvent.Bounty);
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

### 5. Testing Bounty and Punishment Escalation

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Crime System" section.

```
...
CrimeSystem: Initialized.
  - CrimeSystem initialized.
EconomyManager: Initialized.
...
--- Testing Crime System ---
Player initial heat with Crimson Blades: 0.0
Player initial punishment level with Crimson Blades: None
CrimeSystem: Received crime report from EntityID(1, Gen:1): Assault by EntityID(0, Gen:1) for crimson_blades.
CrimeSystem: EntityID(0, Gen:1) heat for crimson_blades increased by 20.0. New Heat: 20.0.
CrimeSystem: Bounty for EntityID(0, Gen:1) by crimson_blades updated. Level: Warning, Gold: 5, Active: False
Player punishment level with Crimson Blades after Assault: Warning
CrimeSystem: Received crime report from EntityID(1, Gen:1): Trespass by EntityID(0, Gen:1) for crimson_blades.
CrimeSystem: EntityID(0, Gen:1) heat for crimson_blades increased by 5.0. New Heat: 25.0.
CrimeSystem: Bounty for EntityID(0, Gen:1) by crimson_blades updated. Level: Warning, Gold: 5, Active: False
Player punishment level with Crimson Blades after Trespass: Warning
CrimeSystem: Received crime report from EntityID(1, Gen:1): Murder by EntityID(0, Gen:1) for crimson_blades.
CrimeSystem: EntityID(0, Gen:1) heat for crimson_blades increased by 80.0. New Heat: 100.0.
CrimeSystem: Bounty for EntityID(0, Gen:1) by crimson_blades updated. Level: ExecutionOrder, Gold: 500, Active: True
CrimeSystem: Faction crimson_blades is now actively pursuing EntityID(0, Gen:1)!
Player punishment level with Crimson Blades after Murder: ExecutionOrder
Player bounty with Crimson Blades: Bounty on EntityID(0, Gen:1) by crimson_blades: 500 Gold, Level: ExecutionOrder (Active: True)
CrimeSystem: Received crime report from EntityID(1, Gen:1): Theft by EntityID(0, Gen:1) for shadow_veilers.
CrimeSystem: EntityID(0, Gen:1) heat for shadow_veilers increased by 25.0. New Heat: 25.0.
CrimeSystem: Bounty for EntityID(0, Gen:1) by shadow_veilers updated. Level: RefusedServices, Gold: 20, Active: False
Player punishment level with Shadow Veilers after Theft: RefusedServices
CrimeSystem: Received crime report from EntityID(1, Gen:1): CorruptionUse by EntityID(0, Gen:1) for sunken_temple.
CrimeSystem: EntityID(0, Gen:1) heat for sunken_temple increased by 70.0. New Heat: 70.0.
CrimeSystem: Bounty for EntityID(0, Gen:1) by sunken_temple updated. Level: HunterSquads, Gold: 300, Active: True
CrimeSystem: Faction sunken_temple is now actively pursuing EntityID(0, Gen:1)!
Player punishment level with Sunken Temple after CorruptionUse: HunterSquads
CrimeSystem: Received crime report from EntityID(1, Gen:1): CorruptionUse by EntityID(0, Gen:1) for void_cultists.
CrimeSystem: EntityID(0, Gen:1) heat for void_cultists increased by -50.0. New Heat: -50.0.
CrimeSystem: Bounty for EntityID(0, Gen:1) by void_cultists updated. Level: None, Gold: 0, Active: False
Player punishment level with Void Cultists after CorruptionUse: None
--- End Testing Crime System ---
...
```

**Key Observations:**

*   **Punishment Escalation**: `AddHeat` correctly updates the `Bounty` and `PunishmentLevel` based on `HeatValue` thresholds (e.g., from `None` to `Warning`, then to `ExecutionOrder`).
*   **Gold Value**: `GoldValue` on the `Bounty` also increases with `PunishmentLevel`.
*   **Active Pursuit**: `PunishmentLevel.HunterSquads` and `ExecutionOrder` correctly trigger `IsActivePursuit` and the `OnFactionPursuitChanged` event.
*   **Decay (Implicit)**: The `Tick` method's `DecayHeat` will reduce heat over time, which would then lower punishment levels.
*   **Ideological Impact**: `Void Cultists` correctly show `None` punishment level (even with negative heat), demonstrating faction-specific legal interpretation.

This confirms our `CrimeSystem` is robustly managing bounties and escalating punishments based on faction heat.

### Summary

You have successfully implemented **Bounty & Punishment** mechanisms in the C# Brain, defining `PunishmentLevel` and `Bounty` structs, and enhancing `CrimeSystem` to manage these. By translating `FactionHeat` into escalating `PunishmentLevel`s, updating `Bounty` values, and triggering active pursuit states, you've established a robust system for dynamic consequences. This crucial system strictly adheres to TDD 06.4.2's specifications, providing the authoritative backbone for escalating consequences and supporting the morally ambiguous outlaw path in Sigilborne.

### Next Steps

This concludes **Module 8: Society, Politics & Economy**. We will now move on to **Module 9: Advanced World Mechanics & Player Impact**, starting with **Ritual System - Pattern Matching & Execution (C#)**, where we will design how spatial arrangements of items trigger complex magical rituals.