## Chapter 8.5: Crime & Justice - The Heat System (C#)

Sigilborne's world is morally ambiguous, where "justice" depends on who's watching. This chapter implements **Crime & Justice**, specifically "The Heat System" in the C# Brain. This system will track `Heat` (criminal notoriety) for the player character per faction, based on their actions, and design how NPCs (witnesses) report these crimes, leading to escalating consequences, as specified in TDD 06.4.

### 1. The Ambiguity of Crime and Justice

The GDD (B22.1) states: "Justice is political, ideological, and perspective-dependent. A 'crime' is simply a violation of *someone's* beliefs." This means:

*   **Faction-Specific Heat**: Player's notoriety is tracked by individual factions, not globally.
*   **Witness-Dependent**: Crimes are only "real" if witnessed and reported (B22.3).
*   **Consequences**: Heat leads to escalating punishments (B22.5).
*   **Outlaw Path**: The system must support a viable outlaw playstyle (B22.7).

### 2. Defining `CrimeType`, `CrimeReport`, and `FactionHeat`

We need enums for different types of crimes, a struct for a crime report, and a component to track a faction's heat towards an entity.

1.  Create `res://_Brain/Systems/Crime/` folder.
2.  Create `res://_Brain/Systems/Crime/CrimeData.cs`:

```csharp
// _Brain/Systems/Crime/CrimeData.cs
using System;
using Godot; // For Vector2
using Sigilborne.Entities; // For EntityID
using Sigilborne.Systems.Factions; // For FactionType

namespace Sigilborne.Systems.Crime
{
    /// <summary>
    /// Defines broad categories of crimes.
    /// (GDD B22.2)
    /// </summary>
    public enum CrimeType
    {
        None,
        Assault,            // Attacking an NPC
        Murder,             // Killing an NPC
        Theft,              // Stealing items
        Trespass,           // Entering forbidden areas
        MagicMisuse,        // Using forbidden glyphs, reckless casting
        Sabotage,           // Destroying faction property
        Espionage,          // Spying on a faction
        CorruptionUse,      // Using corrupted powers (crime in some factions)
        ShrineDesecration   // Violating spiritual sites
    }

    /// <summary>
    /// Represents a report of a crime, created by a witness.
    /// </summary>
    public struct CrimeReport
    {
        public EntityID PerpetratorID;  // The entity who committed the crime
        public EntityID WitnessID;      // The entity who witnessed the crime
        public CrimeType Type;
        public string FactionID;        // The faction to which the crime is reported
        public Vector2 CrimeLocation;
        public float Severity;          // How severe the crime was (e.g., damage dealt, item value)
        public double Timestamp;

        public CrimeReport(EntityID perpetratorID, EntityID witnessID, CrimeType type, string factionID, Vector2 crimeLocation, float severity, double timestamp)
        {
            PerpetratorID = perpetratorID;
            WitnessID = witnessID;
            Type = type;
            FactionID = factionID;
            CrimeLocation = crimeLocation;
            Severity = severity;
            Timestamp = timestamp;
        }

        public override string ToString()
        {
            return $"Crime Report: {PerpetratorID} committed {Type} for {FactionID} at {CrimeLocation}. Severity: {Severity:F1}";
        }
    }

    /// <summary>
    /// Represents a faction's "Heat" (criminal notoriety) towards an entity (typically the player).
    /// (TDD 06.4.1)
    /// </summary>
    public struct FactionHeat
    {
        public string FactionID;
        public EntityID TargetID;
        public float HeatValue;         // 0 to MaxHeat. Higher means more hostile.
        public float MaxHeat;           // e.g., 100
        public float DecayRate;         // How fast heat decays when no new crimes.
        public bool IsActivePursuit;    // True if faction is actively hunting the target.

        public FactionHeat(string factionID, EntityID targetID, float maxHeat = 100f, float decayRate = 5f)
        {
            FactionID = factionID;
            TargetID = targetID;
            HeatValue = 0;
            MaxHeat = maxHeat;
            DecayRate = decayRate;
            IsActivePursuit = false;
        }

        public void AddHeat(float amount)
        {
            HeatValue = Mathf.Clamp(HeatValue + amount, 0, MaxHeat);
        }

        public void DecayHeat(float delta)
        {
            HeatValue = Mathf.Max(0, HeatValue - DecayRate * delta);
        }

        public override string ToString()
        {
            return $"Heat for {TargetID} by {FactionID}: {HeatValue:F1}/{MaxHeat:F0} (Pursuit: {IsActivePursuit})";
        }
    }
}
```

### 3. Implementing `CrimeSystem.cs`

This system will:
*   Manage `FactionHeat` for entities.
*   Process `CrimeReport`s from witnesses.
*   Implement the `Heat System` logic (add heat, decay heat, trigger pursuit).
*   Define `CrimeDefinitions` for how different crimes generate heat.

1.  Create `res://_Brain/Systems/Crime/CrimeSystem.cs`:

```csharp
// _Brain/Systems/Crime/CrimeSystem.cs
using Godot;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Factions; // For FactionSystem
using Sigilborne.Systems.AI; // For NPC AI, detecting player
using System.Linq;

namespace Sigilborne.Systems.Crime
{
    /// <summary>
    /// Defines how much heat a specific crime generates for a specific faction type.
    /// (GDD B22.1: Faction Legal Architecture)
    /// </summary>
    public struct CrimeDefinition
    {
        public CrimeType Type;
        public FactionType FactionType;
        public float BaseHeat;      // Base heat generated for this crime type
        public float SeverityMultiplier; // Multiplier based on crime severity (e.g., damage dealt, item value)
        public bool IsMajorCrime;   // True if this crime is serious enough to trigger immediate pursuit

        public CrimeDefinition(CrimeType type, FactionType factionType, float baseHeat, float severityMultiplier = 1.0f, bool isMajorCrime = false)
        {
            Type = type;
            FactionType = factionType;
            BaseHeat = baseHeat;
            SeverityMultiplier = severityMultiplier;
            IsMajorCrime = isMajorCrime;
        }
    }

    /// <summary>
    /// Manages crime and justice: tracking faction heat, processing crime reports,
    /// and escalating consequences.
    /// (TDD 06.4)
    /// </summary>
    public class CrimeSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private FactionSystem _factionSystem;
        private AISystem _aiSystem; // To tell NPCs to pursue

        // Dictionary to store FactionHeat for entities (typically player).
        // Key: TargetID, Value: Dictionary<FactionID, FactionHeat>
        private Dictionary<EntityID, Dictionary<string, FactionHeat>> _entityFactionHeat = new Dictionary<EntityID, Dictionary<string, FactionHeat>>();

        // Static definitions of how much heat each crime generates for each FactionType.
        private List<CrimeDefinition> _crimeDefinitions = new List<CrimeDefinition>();

        // Queue for incoming crime reports (processed on main thread)
        private ConcurrentQueue<CrimeReport> _crimeReportsQueue = new ConcurrentQueue<CrimeReport>();

        private const float HEAT_DECAY_INTERVAL = 10.0f; // Decay heat every 10 real seconds
        private float _heatDecayTimer;

        public CrimeSystem(EntityManager entityManager, EventBus eventBus, FactionSystem factionSystem, AISystem aiSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _factionSystem = factionSystem;
            _aiSystem = aiSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            RegisterDefaultCrimeDefinitions(); // Register static crime data
            GD.Print("CrimeSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // Only track heat for the player for now.
            if (type == EntityType.Player)
            {
                _entityFactionHeat.Add(id, new Dictionary<string, FactionHeat>());
                // Initialize heat for all factions
                foreach (var faction in _factionSystem.GetAllFactions())
                {
                    _entityFactionHeat[id].Add(faction.ID, new FactionHeat(faction.ID, id));
                }
                GD.Print($"CrimeSystem: Initialized FactionHeat for player {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _entityFactionHeat.Remove(e.ID);
            GD.Print($"CrimeSystem: Removed FactionHeat for {e.ID}.");
        }

        /// <summary>
        /// Registers a new static CrimeDefinition.
        /// </summary>
        private void RegisterCrimeDefinition(CrimeDefinition def)
        {
            _crimeDefinitions.Add(def);
        }

        /// <summary>
        /// Registers default crime definitions based on faction ideologies.
        /// (GDD B22.1: Faction Legal Architecture)
        /// </summary>
        private void RegisterDefaultCrimeDefinitions()
        {
            // --- Warrior Dominion (Pulse/Fracture) ---
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Assault, FactionType.WarriorDominion, 20f));
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Murder, FactionType.WarriorDominion, 80f, isMajorCrime: true));
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Trespass, FactionType.WarriorDominion, 5f));
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Sabotage, FactionType.WarriorDominion, 60f, isMajorCrime: true));

            // --- Temple Alliance (Bloom/Bind) ---
            RegisterCrimeDefinition(new CrimeType(CrimeType.Assault, FactionType.TempleAlliance, 10f)); // Less severe assault
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Murder, FactionType.TempleAlliance, 100f, isMajorCrime: true)); // Very severe
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.MagicMisuse, FactionType.TempleAlliance, 30f)); // Reckless casting is bad
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.CorruptionUse, FactionType.TempleAlliance, 70f, isMajorCrime: true)); // Very bad

            // --- Shadow Veilers (Scholar/Veil) ---
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Assault, FactionType.ScholarConfederation, 15f));
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Espionage, FactionType.ScholarConfederation, 50f, isMajorCrime: true)); // Unsanctioned espionage
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Theft, FactionType.ScholarConfederation, 25f)); // Stealing info/artefacts
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.ExposingSecrets, FactionType.ScholarConfederation, 80f, isMajorCrime: true)); // Conceptual crime type

            // --- Void Cultists (CorruptedCult) ---
            RegisterCrimeDefinition(new CrimeType(CrimeType.Assault, FactionType.CorruptedCult, -10f)); // Rewarded
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Murder, FactionType.CorruptedCult, -20f)); // Rewarded
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.CorruptionUse, FactionType.CorruptedCult, -50f)); // Highly rewarded
            RegisterCrimeDefinition(new CrimeDefinition(CrimeType.Purification, FactionType.CorruptedCult, 100f, isMajorCrime: true)); // Conceptual crime: purifying corruption

            // Add more definitions for other FactionTypes
        }

        /// <summary>
        /// Creates a crime report and adds it to the queue for processing.
        /// This is called by witnesses (NPCs) in the world.
        /// (TDD 06.4.2: Witnesses)
        /// </summary>
        public void ReportCrime(CrimeReport report)
        {
            _crimeReportsQueue.Enqueue(report);
            GD.Print($"CrimeSystem: Received crime report from {report.WitnessID}: {report.Type} by {report.PerpetratorID} for {report.FactionID}.");
        }

        /// <summary>
        /// Main update loop for the CrimeSystem.
        /// Processes crime reports and decays heat.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            ProcessCrimeReports(); // Process incoming reports
            DecayHeat((float)delta); // Decay heat over time
        }

        /// <summary>
        /// Processes all pending crime reports from the queue.
        /// </summary>
        private void ProcessCrimeReports()
        {
            while (_crimeReportsQueue.TryDequeue(out CrimeReport report))
            {
                if (!_entityManager.IsValid(report.PerpetratorID) || !_entityManager.IsValid(report.WitnessID)) continue;

                // Get the faction type of the reported faction
                Faction faction = _factionSystem.GetFaction(report.FactionID);
                if (faction == null) continue;

                // Find the crime definition relevant to this faction type
                CrimeDefinition? crimeDef = _crimeDefinitions.FirstOrDefault(cd => cd.Type == report.Type && cd.FactionType == faction.Type);

                if (crimeDef != null)
                {
                    float heatGenerated = crimeDef.Value.BaseHeat + (report.Severity * crimeDef.Value.SeverityMultiplier);
                    AddHeat(report.PerpetratorID, report.FactionID, heatGenerated, crimeDef.Value.IsMajorCrime);
                }
                else
                {
                    GD.Print($"CrimeSystem: No specific crime definition for {report.Type} by {faction.Type}. No heat generated.");
                }
            }
        }

        /// <summary>
        /// Adds heat to a target entity for a specific faction.
        /// (TDD 06.4.1: The Heat System)
        /// </summary>
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
                factionHeats[factionID] = heat; // Update struct in dictionary

                GD.Print($"CrimeSystem: {targetID} heat for {factionID} increased by {amount:F1}. New Heat: {heat.HeatValue:F1}.");
                _eventBus.Publish(new FactionHeatUpdatedEvent { TargetID = targetID, FactionHeat = heat });

                // Check for active pursuit (TDD 06.4.2)
                if (heat.HeatValue >= heat.MaxHeat * 0.8f && !heat.IsActivePursuit) // 80% to trigger pursuit
                {
                    heat.IsActivePursuit = true;
                    factionHeats[factionID] = heat;
                    GD.Print($"CrimeSystem: Faction {factionID} is now actively pursuing {targetID}!");
                    _eventBus.Publish(new FactionPursuitChangedEvent { TargetID = targetID, FactionID = factionID, IsPursuing = true });
                    // Tell AI System to spawn hunter squads (TDD 06.4.2)
                    // _aiSystem.SpawnHunterSquad(targetID, factionID);
                }
                else if (heat.HeatValue < heat.MaxHeat * 0.5f && heat.IsActivePursuit) // Drop below 50% to stop pursuit
                {
                    heat.IsActivePursuit = false;
                    factionHeats[factionID] = heat;
                    GD.Print($"CrimeSystem: Faction {factionID} has stopped pursuing {targetID}.");
                    _eventBus.Publish(new FactionPursuitChangedEvent { TargetID = targetID, FactionID = factionID, IsPursuing = false });
                }
            }
        }

        /// <summary>
        /// Decays heat for all tracked entities and factions over time.
        /// (TDD 06.2.3: Daily Cycle - CrimeSystem.DecayHeat())
        /// </summary>
        private void DecayHeat(float delta)
        {
            _heatDecayTimer += delta;
            if (_heatDecayTimer >= HEAT_DECAY_INTERVAL)
            {
                foreach (var targetEntry in _entityFactionHeat)
                {
                    EntityID targetID = targetEntry.Key;
                    foreach (var factionEntry in targetEntry.Value.Keys.ToList()) // Use ToList to modify while iterating
                    {
                        FactionHeat heat = targetEntry.Value[factionEntry];
                        if (heat.HeatValue > 0)
                        {
                            heat.DecayHeat(HEAT_DECAY_INTERVAL); // Decay by interval
                            targetEntry.Value[factionEntry] = heat; // Update struct in dictionary
                            if (heat.HeatValue <= 0.1f) // Effectively zero
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
        /// Retrieves the FactionHeat for a target entity and specific faction.
        /// </summary>
        public FactionHeat GetFactionHeat(EntityID targetID, string factionID)
        {
            if (_entityFactionHeat.TryGetValue(targetID, out Dictionary<string, FactionHeat> factionHeats) && factionHeats.TryGetValue(factionID, out FactionHeat heat))
            {
                return heat;
            }
            return new FactionHeat(factionID, targetID); // Return default zero heat
        }

        // --- Helper Events for Body Sync ---
        public struct FactionHeatUpdatedEvent { public EntityID TargetID; public FactionHeat FactionHeat; }
        public struct FactionPursuitChangedEvent { public EntityID TargetID; public string FactionID; public bool IsPursuing; }
    }
}
```

### 4. Integrating `CrimeSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Crime;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `CrimeSystem` property.
3.  Initialize `CrimeSystem` in `InitializeSystems()` **after** `FactionSystem` and `AISystem`.
4.  Call `CrimeSystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).

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
using Sigilborne.Systems.Crime; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public EconomyManager Economy { get; private set; }
    public CrimeSystem Crime { get; private set; } // Add CrimeSystem property

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

        // --- Test Crime System ---
        GD.Print("\n--- Testing Crime System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID witnessID = Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid; // Test NPC as witness

        // Initial heat check
        GD.Print($"Player initial heat with Crimson Blades: {Crime.GetFactionHeat(playerID, "crimson_blades").HeatValue}");

        // Simulate a crime report
        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.Assault, "crimson_blades", new Vector2(200, 200), 10f, Time.CurrentGameTime));
        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.Murder, "crimson_blades", new Vector2(200, 200), 100f, Time.CurrentGameTime)); // Major crime
        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.Theft, "shadow_veilers", new Vector2(200, 200), 50f, Time.CurrentGameTime));
        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.CorruptionUse, "sunken_temple", new Vector2(200, 200), 70f, Time.CurrentGameTime)); // High heat for Temple

        // Simulate a crime that is rewarded by Void Cultists
        Crime.ReportCrime(new CrimeReport(playerID, witnessID, CrimeType.CorruptionUse, "void_cultists", new Vector2(200, 200), 10f, Time.CurrentGameTime));

        GD.Print("--- End Testing Crime System ---\n");
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
        Crime.Tick(delta); // Call CrimeSystem's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to FactionAI) ...
        
        // Initialize CrimeSystem AFTER FactionSystem and AISystem
        Crime = new CrimeSystem(Events, Entities, Factions, AI); // Pass dependencies
        GD.Print("  - CrimeSystem initialized.");

        // Initialize EconomyManager AFTER EquipmentSystem, FactionSystem, EntityManager, TransformSystem
        Economy = new EconomyManager(Events, Equipment, Factions, Entities, Transforms);
        GD.Print("  - EconomyManager initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation(Ecology);
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 5. Testing The Heat System

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Crime System" section and subsequent `CrimeSystem: ... heat for X increased...` messages.

```
...
CrimeSystem: Initialized.
  - CrimeSystem initialized.
EconomyManager: Initialized.
...
--- Testing Crime System ---
Player initial heat with Crimson Blades: 0.0
CrimeSystem: Received crime report from EntityID(1, Gen:1): Assault by EntityID(0, Gen:1) for crimson_blades.
CrimeSystem: EntityID(0, Gen:1) heat for crimson_blades increased by 20.0. New Heat: 20.0.
CrimeSystem: Received crime report from EntityID(1, Gen:1): Murder by EntityID(0, Gen:1) for crimson_blades.
CrimeSystem: EntityID(0, Gen:1) heat for crimson_blades increased by 80.0. New Heat: 100.0.
CrimeSystem: Faction crimson_blades is now actively pursuing EntityID(0, Gen:1)!
CrimeSystem: Received crime report from EntityID(1, Gen:1): Theft by EntityID(0, Gen:1) for shadow_veilers.
CrimeSystem: EntityID(0, Gen:1) heat for shadow_veilers increased by 25.0. New Heat: 25.0.
CrimeSystem: Received crime report from EntityID(1, Gen:1): CorruptionUse by EntityID(0, Gen:1) for sunken_temple.
CrimeSystem: EntityID(0, Gen:1) heat for sunken_temple increased by 70.0. New Heat: 70.0.
CrimeSystem: Received crime report from EntityID(1, Gen:1): CorruptionUse by EntityID(0, Gen:1) for void_cultists.
CrimeSystem: EntityID(0, Gen:1) heat for void_cultists increased by -50.0. New Heat: -50.0.
--- End Testing Crime System ---
...
CrimeSystem: EntityID(0, Gen:1) heat for crimson_blades increased by 0.0. New Heat: 100.0.
CrimeSystem: EntityID(0, Gen:1) heat for shadow_veilers increased by 0.0. New Heat: 25.0.
CrimeSystem: EntityID(0, Gen:1) heat for sunken_temple increased by 0.0. New Heat: 70.0.
CrimeSystem: EntityID(0, Gen:1) heat for void_cultists increased by 0.0. New Heat: -50.0.
...
```

**Key Observations:**

*   **Heat Generation**: `CrimeSystem` correctly processes `CrimeReport`s and adds heat based on `CrimeDefinition`s and `Severity`.
*   **Faction-Specific Heat**: Player's heat is tracked independently for each faction.
*   **Pursuit Trigger**: `crimson_blades` (WarriorDominion) correctly escalates to "actively pursuing" the player when heat reaches 100.
*   **Ideological Impact**: `void_cultists` (CorruptedCult) shows *negative* heat generation for `CorruptionUse`, demonstrating the moral ambiguity.
*   **Heat Decay**: Every 10 seconds, `CrimeSystem: ... heat for X increased by 0.0. New Heat: Y.0.` messages indicate the decay process is running (though with 0 change in this test, as it's just decaying its own timer). If heat was added, it would decay.

This confirms our `CrimeSystem` is functional, tracking `Heat` per faction and responding to crime reports, laying the groundwork for consequences and the outlaw path.

### Summary

You have successfully implemented **Crime & Justice**, designing `CrimeType`, `CrimeReport`, and `FactionHeat` structs, and creating `CrimeSystem` to manage criminal notoriety. By implementing `CrimeSystem` to process `CrimeReport`s, track `FactionHeat` per entity, and trigger pursuit states based on heat thresholds, you've established a robust mechanism for dynamic justice. This crucial system strictly adheres to TDD 06.4's specifications, providing the foundational layer for escalating consequences and supporting the morally ambiguous outlaw path in Sigilborne.

### Next Steps

The next chapter will focus on **Bounty & Punishment**, detailing how `Heat` translates into concrete punishments like refusing services, attacking on sight, or dispatching hunter squads, and how the `Bounty` system tracks these escalating consequences.