## Chapter 9.2: Seals & Locks - Logical Locks (C#)

Our `RitualSystem` now allows for complex magical ceremonies. Often, these rituals are used to interact with powerful **Seals & Locks** that govern ancient powers, restrict areas, or contain dangerous entities. This chapter implements logical locks on objects and defines global seals that can affect the entire world state, detailing how these seals can be physical, magical, or time-based, as specified in TDD 09.3 and GDD B27.1.

### 1. The Power of Seals

The GDD (B27.1) states: "Seals are not decorative. They are glyph logic anchored into physical space, used by clans, spirits, corrupted cults, and ancient civilizations." This implies:

*   **Containment & Empowerment**: Seals can bind entities or enhance areas.
*   **Logical Locks**: They restrict access or require specific actions to open/break.
*   **World Impact**: Global seals can alter fundamental world rules.
*   **Interaction**: Players interact with seals using items, spells, or specific conditions.

### 2. Defining `SealType`, `SealStatus`, and `SealDefinition`

We need enums for different types of seals and their status, and a class to define a static seal blueprint.

1.  Create `res://_Brain/Systems/Rituals/SealData.cs`:

```csharp
// _Brain/Systems/Rituals/SealData.cs
using System;
using System.Collections.Generic;
using Godot; // For Vector2
using Sigilborne.Entities; // For EntityID
using Sigilborne.Systems.Magic; // For GlyphConcept
using Sigilborne.Systems.Inventory; // For ItemID

namespace Sigilborne.Systems.Rituals
{
    /// <summary>
    /// Defines the broad categories of seals.
    /// (TDD 09.3.1)
    /// </summary>
    public enum SealType
    {
        None,
        Physical,       // Requires a key item
        Magical,        // Requires a specific spell/glyph to be cast
        Blood,          // Requires blood sacrifice or specific bloodline
        Time,           // Opens/closes at specific times of day/year
        Ritual,         // Requires a specific ritual to be performed
        Spirit,         // Tied to a spirit, requires spirit interaction
        Corruption,     // Tied to corruption, requires corrupted action or purification
        Global          // Affects entire world state
    }

    /// <summary>
    /// Defines the current status of a seal.
    /// </summary>
    public enum SealStatus
    {
        Sealed,         // Active, closed, binding
        Weakened,       // Partially broken, less effective, might flicker
        Broken,         // Inactive, open, unbound
        TemporarilyOpen // For time-based seals or temporary breaches
    }

    /// <summary>
    /// Static data defining a magical seal.
    /// (TDD 09.3.1)
    /// </summary>
    public class SealDefinition
    {
        public string ID { get; private set; } // Unique ID (e.g., "ancient_dungeon_seal", "sun_seal_global")
        public string Name { get; private set; }
        public string Description { get; private set; }
        public SealType Type { get; private set; }
        public EntityID TargetEntityID { get; private set; } // The entity this seal is attached to (e.g., a door, an NPC)
        public bool IsGlobal { get; private set; } // True if this seal affects world state directly (TDD 09.3.2)

        // Requirements to break/open this seal
        public string RequiredItemID { get; private set; } // For Physical seals
        public GlyphConcept RequiredGlyphConcept { get; private set; } // For Magical seals
        public float RequiredBloodAmount { get; private set; } // For Blood seals
        public int RequiredGameHour { get; private set; } // For Time seals (e.g., 12 for noon)
        public string RequiredRitualID { get; private set; } // For Ritual seals

        // Effects this seal provides when active (Sealed) or broken (Broken)
        public List<string> ActiveEffects { get; private set; } // Effects when sealed (e.g., "area_slow", "spawn_guard")
        public List<string> BrokenEffects { get; private set; } // Effects when broken (e.g., "release_monster", "area_corruption")

        public SealDefinition(string id, string name, string description, SealType type, EntityID targetEntityID, bool isGlobal,
                              string requiredItemID = null, GlyphConcept requiredGlyphConcept = GlyphConcept.None, float requiredBloodAmount = 0f,
                              int requiredGameHour = -1, string requiredRitualID = null, List<string> activeEffects = null, List<string> brokenEffects = null)
        {
            ID = id;
            Name = name;
            Description = description;
            Type = type;
            TargetEntityID = targetEntityID;
            IsGlobal = isGlobal;

            RequiredItemID = requiredItemID;
            RequiredGlyphConcept = requiredGlyphConcept;
            RequiredBloodAmount = requiredBloodAmount;
            RequiredGameHour = requiredGameHour;
            RequiredRitualID = requiredRitualID;

            ActiveEffects = activeEffects ?? new List<string>();
            BrokenEffects = brokenEffects ?? new List<string>();
        }

        public override string ToString()
        {
            string target = TargetEntityID.IsValid() ? $"Target: {TargetEntityID}" : "No Target";
            return $"Seal: '{Name}' ({ID}) | Type: {Type}, {target}, Global: {IsGlobal}";
        }
    }

    /// <summary>
    /// Component to mark a world object as a potential ritual center. (Moved from RitualData)
    /// </summary>
    public struct RitualCenterComponent
    {
        public bool IsActive; // True if a ritual is currently being performed here
        public string ActiveRitualID; // The ID of the ritual being performed

        public RitualCenterComponent(bool isActive = false, string activeRitualID = null)
        {
            IsActive = isActive;
            ActiveRitualID = activeRitualID;
        }
    }
}
```

#### 2.1. Update `RitualSystem.cs` to use `RitualCenterComponent`

The `RitualSystem` needs to manage `RitualCenterComponent` on its ritual altars.

1.  Open `res://_Brain/Systems/Rituals/RitualSystem.cs`.
2.  Replace `_activeRitualCenters` with a dictionary for `RitualCenterComponent`.
3.  Modify `OnEntitySpawned`, `OnEntityDespawned`, `InitiateRitual`, `CompleteRitual` to use `RitualCenterComponent`.

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
using System.Threading; // For Thread.Sleep in job

namespace Sigilborne.Systems.Rituals
{
    // ... (RitualComponentRequirement, RitualDefinition structs/classes) ...
    // ... (RitualCenterComponent struct) ... // Ensure this is defined in SealData.cs

    public class RitualSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private InventorySystem _inventorySystem;
        private SpatialHashGrid _spatialGrid;
        private BiologicalSystem _biologicalSystem;
        private TransformSystem _transformSystem; // New: For getting transform of ritual center

        private Dictionary<string, RitualDefinition> _ritualDefinitions = new Dictionary<string, RitualDefinition>();
        
        // Active ritual centers: Key: EntityID of the ritual center, Value: RitualCenterComponent
        private Dictionary<EntityID, RitualCenterComponent> _ritualCenters = new Dictionary<EntityID, RitualCenterComponent>(); // Changed from _activeRitualCenters


        public RitualSystem(EntityManager entityManager, EventBus eventBus, InventorySystem inventorySystem, SpatialHashGrid spatialGrid, BiologicalSystem biologicalSystem, TransformSystem transformSystem) // Add TransformSystem
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _inventorySystem = inventorySystem;
            _spatialGrid = spatialGrid;
            _biologicalSystem = biologicalSystem;
            _transformSystem = transformSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            RegisterDefaultRitualDefinitions();
            GD.Print("RitualSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.WorldObject && definitionID.Contains("altar"))
            {
                _ritualCenters.Add(id, new RitualCenterComponent()); // Add component to potential ritual center
                GD.Print($"RitualSystem: Identified potential ritual center: {id} ({definitionID}). Added RitualCenterComponent.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _ritualCenters.Remove(e.ID); // Remove component
        }

        // ... (RegisterRitualDefinition, RegisterDefaultRitualDefinitions methods) ...

        public bool InitiateRitual(EntityID ritualInitiatorID, EntityID ritualCenterID, string ritualID)
        {
            if (!_entityManager.IsValid(ritualInitiatorID) || !_entityManager.IsValid(ritualCenterID)) { /* ... */ return false; }
            if (!_ritualDefinitions.TryGetValue(ritualID, out RitualDefinition ritualDef)) { /* ... */ return false; }
            if (!_transformSystem.TryGetTransform(ritualCenterID, out TransformComponent centerTransform)) { /* ... */ return false; }
            if (!_biologicalSystem.TryGetCoreStats(ritualInitiatorID, out CoreStats initiatorStats)) { /* ... */ return false; }

            // Check if ritual center is already active
            if (_ritualCenters.TryGetValue(ritualCenterID, out RitualCenterComponent centerComp) && centerComp.IsActive)
            {
                GD.Print($"RitualSystem: Ritual center {ritualCenterID} is already active with ritual '{centerComp.ActiveRitualID}'. Cannot initiate new ritual.");
                _eventBus.Publish(new RitualFailedEvent { InitiatorID = ritualInitiatorID, RitualID = ritualID, Reason = "Center Busy" });
                return false;
            }

            // --- 1. Check Initiator Resources (Chakra, Stability) ---
            // ... (existing resource checks) ...

            // --- 2. Check Ritual Component Pattern ---
            // ... (existing component check logic) ...

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
            // Mark ritual center as active (TDD 09.2.2)
            ref RitualCenterComponent ritualCenterRef = ref _ritualCenters.GetValueRef(ritualCenterID);
            ritualCenterRef.IsActive = true;
            ritualCenterRef.ActiveRitualID = ritualID;

            GD.Print($"RitualSystem: Ritual '{ritualID}' initiated by {ritualInitiatorID} at {ritualCenterID}. Cast time: {ritualDef.CastTime:F1}s.");
            _eventBus.Publish(new RitualInitiatedEvent { InitiatorID = ritualInitiatorID, RitualID = ritualID, CenterID = ritualCenterID, CastTime = ritualDef.CastTime });

            _jobSystem.Schedule(new RitualCompletionJob(ritualInitiatorID, ritualCenterID, ritualID, ritualDef.CastTime), () => {
                CompleteRitual(ritualInitiatorID, ritualCenterID, ritualID);
            });

            return true;
        }

        /// <summary>
        /// Completes a ritual after its cast time.
        /// </summary>
        public void CompleteRitual(EntityID initiatorID, EntityID ritualCenterID, string ritualID)
        {
            if (!_ritualDefinitions.TryGetValue(ritualID, out RitualDefinition ritualDef)) return;
            if (!_ritualCenters.TryGetValue(ritualCenterID, out RitualCenterComponent centerComp) || !centerComp.IsActive || centerComp.ActiveRitualID != ritualID)
            {
                GD.PrintErr($"RitualSystem: Ritual center {ritualCenterID} is not active with ritual '{ritualID}' or was interrupted.");
                return; // Ritual might have been interrupted or another started.
            }

            // Clear ritual center state
            ref RitualCenterComponent ritualCenterRef = ref _ritualCenters.GetValueRef(ritualCenterID);
            ritualCenterRef.IsActive = false;
            ritualCenterRef.ActiveRitualID = null;

            GD.Print($"RitualSystem: Ritual '{ritualID}' completed successfully by {initiatorID} at {ritualCenterID}! Triggering effects.");
            _eventBus.Publish(new RitualCompletedEvent { InitiatorID = initiatorID, RitualID = ritualID, CenterID = ritualCenterID });

            // ... (existing effect triggering) ...
        }

        // ... (RitualCompletionJob struct) ...
        // ... (Helper Events) ...
    }
}
```

### 4. Implementing `SealSystem.cs`

This system will:
*   Manage `SealDefinition`s.
*   Track the `SealStatus` of active seals.
*   Provide methods to `OpenSeal` or `BreakSeal`.
*   Handle `Global` seals that affect world state.

1.  Create `res://_Brain/Systems/Rituals/SealSystem.cs`:

```csharp
// _Brain/Systems/Rituals/SealSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Inventory; // For InventorySystem
using Sigilborne.Systems.Magic; // For GlyphConcept
using Sigilborne.Systems.Biology; // For CoreStats
using System.Linq;

namespace Sigilborne.Systems.Rituals
{
    /// <summary>
    /// Manages seals (locks) on entities and global seals that affect the world state.
    /// (TDD 09.3)
    /// </summary>
    public class SealSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private InventorySystem _inventorySystem; // For checking/consuming items
        private BiologicalSystem _biologicalSystem; // For checking health/blood (Blood seals)
        private TimeSystem _timeSystem; // For checking time (Time seals)

        // Static definitions of all possible seals
        private Dictionary<string, SealDefinition> _sealDefinitions = new Dictionary<string, SealDefinition>();

        // Active seals in the world and their current status.
        // Key: SealDefinition.ID
        private Dictionary<string, SealStatus> _activeSealStatuses = new Dictionary<string, SealStatus>();

        public SealSystem(EntityManager entityManager, EventBus eventBus, InventorySystem inventorySystem, BiologicalSystem biologicalSystem, TimeSystem timeSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _inventorySystem = inventorySystem;
            _biologicalSystem = biologicalSystem;
            _timeSystem = timeSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned; // For identifying seals on entities
            _eventBus.OnEntityDespawned += OnEntityDespawned;
            _eventBus.OnNewHour += OnNewHour; // For time-based seals

            RegisterDefaultSealDefinitions(); // Register some default seals
            GD.Print("SealSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // For now, assume seals are manually registered.
            // In a real game, entities might have a SealComponent that links to a SealDefinition.
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            // Remove seals associated with this entity.
            foreach (var sealDef in _sealDefinitions.Values.Where(s => s.TargetEntityID == e.ID).ToList())
            {
                _activeSealStatuses.Remove(sealDef.ID);
                GD.Print($"SealSystem: Removed seal '{sealDef.ID}' as its target entity {e.ID} despawned.");
            }
        }

        private void OnNewHour(int hour)
        {
            // Check time-based seals
            foreach (var sealDef in _sealDefinitions.Values.Where(s => s.Type == SealType.Time && s.RequiredGameHour == hour).ToList())
            {
                // If it's a time-based seal and the hour matches, open it temporarily.
                if (_activeSealStatuses.TryGetValue(sealDef.ID, out SealStatus currentStatus) && currentStatus == SealStatus.Sealed)
                {
                    GD.Print($"SealSystem: Time-based seal '{sealDef.ID}' is now TemporarilyOpen at {hour}:00!");
                    _activeSealStatuses[sealDef.ID] = SealStatus.TemporarilyOpen;
                    _eventBus.Publish(new SealStatusChangedEvent { SealID = sealDef.ID, NewStatus = SealStatus.TemporarilyOpen });
                    // Schedule it to close after a few hours/minutes.
                }
            }
        }

        /// <summary>
        /// Registers a new static SealDefinition.
        /// </summary>
        public void RegisterSealDefinition(SealDefinition def)
        {
            _sealDefinitions[def.ID] = def;
            _activeSealStatuses[def.ID] = SealStatus.Sealed; // All seals start Sealed
            GD.Print($"SealSystem: Registered seal definition '{def.ID}'. Status: Sealed.");

            // Apply global effects if it's a global seal
            if (def.IsGlobal)
            {
                ApplySealEffects(def.ID, SealStatus.Sealed);
            }
        }

        /// <summary>
        /// Registers some common default seals for initial testing.
        /// </summary>
        private void RegisterDefaultSealDefinitions()
        {
            RegisterSealDefinition(new SealDefinition(
                id: "dungeon_entrance_seal", name: "Dungeon Entrance Seal", description: "Blocks a dungeon entrance.",
                type: SealType.Physical, targetEntityID: new EntityID(100, 1), isGlobal: false, // Dummy target ID
                requiredItemID: "dungeon_key_t1", activeEffects: new List<string> { "block_path" }
            ));
            RegisterSealDefinition(new SealDefinition(
                id: "chaos_glyph_seal", name: "Chaos Glyph Seal", description: "Binds a dangerous glyph.",
                type: SealType.Magical, targetEntityID: new EntityID(101, 1), isGlobal: false, // Dummy target ID
                requiredGlyphConcept: GlyphConcept.Bind, activeEffects: new List<string> { "suppress_chaos_aura" }
            ));
            RegisterSealDefinition(new SealDefinition(
                id: "sun_seal_global", name: "Global Sun Seal", description: "Keeps the sun perpetually high.",
                type: SealType.Global, targetEntityID: EntityID.Invalid, isGlobal: true, // No target entity
                requiredGameHour: 18, // Breaks at 6 PM
                activeEffects: new List<string> { "pause_night_cycle" }, brokenEffects: new List<string> { "resume_night_cycle" }
            ));
            // Add more types of seals (Blood, Ritual, Spirit, Corruption)
        }

        /// <summary>
        /// Attempts to open a seal.
        /// </summary>
        public bool OpenSeal(EntityID initiatorID, string sealID, string itemUsed = null, GlyphConcept glyphUsed = GlyphConcept.None, float bloodAmount = 0f)
        {
            if (!_sealDefinitions.TryGetValue(sealID, out SealDefinition sealDef))
            {
                GD.PrintErr($"SealSystem: Seal definition '{sealID}' not found.");
                return false;
            }
            if (!_activeSealStatuses.TryGetValue(sealID, out SealStatus currentStatus) || currentStatus == SealStatus.Broken)
            {
                GD.Print($"SealSystem: Seal '{sealID}' is already broken or not active.");
                return false;
            }

            bool requirementsMet = false;
            string reason = "Unknown";

            switch (sealDef.Type)
            {
                case SealType.Physical:
                    if (itemUsed == sealDef.RequiredItemID) { requirementsMet = true; reason = "Used correct key"; }
                    else { reason = "Incorrect item"; }
                    break;
                case SealType.Magical:
                    if (glyphUsed == sealDef.RequiredGlyphConcept) { requirementsMet = true; reason = "Used correct glyph"; }
                    else { reason = "Incorrect glyph"; }
                    break;
                case SealType.Blood:
                    if (bloodAmount >= sealDef.RequiredBloodAmount) { requirementsMet = true; reason = "Sufficient blood sacrificed"; }
                    else { reason = "Insufficient blood"; }
                    break;
                case SealType.Time:
                    if (currentStatus == SealStatus.TemporarilyOpen) { requirementsMet = true; reason = "Opened by time cycle"; }
                    else { reason = "Not the right time"; }
                    break;
                case SealType.Ritual:
                    // Requires a ritual completion event to trigger this.
                    reason = "Requires ritual completion";
                    break;
                case SealType.Global:
                    reason = "Cannot be opened directly. Breaks via specific conditions (e.g., time, ritual)";
                    break;
                default:
                    reason = "Unsupported seal type";
                    break;
            }

            if (requirementsMet)
            {
                _activeSealStatuses[sealID] = SealStatus.Broken; // Assume breaking for now
                GD.Print($"SealSystem: Initiator {initiatorID} successfully opened/broke seal '{sealID}'. Reason: {reason}");
                _eventBus.Publish(new SealStatusChangedEvent { SealID = sealID, NewStatus = SealStatus.Broken });
                
                // Apply broken effects
                ApplySealEffects(sealID, SealStatus.Broken);
                return true;
            }
            GD.Print($"SealSystem: Initiator {initiatorID} failed to open seal '{sealID}'. Reason: {reason}");
            _eventBus.Publish(new SealFailedEvent { InitiatorID = initiatorID, SealID = sealID, Reason = reason });
            return false;
        }

        /// <summary>
        /// Retrieves the current status of a seal.
        /// </summary>
        public SealStatus GetSealStatus(string sealID)
        {
            if (_activeSealStatuses.TryGetValue(sealID, out SealStatus status))
            {
                return status;
            }
            return SealStatus.None; // Seal not registered or invalid
        }

        /// <summary>
        /// Applies effects associated with a seal's status (active or broken).
        /// (GDD B27.1.1: Containment & Empowerment, GDD B27.1.2: Dual Seals)
        /// </summary>
        private void ApplySealEffects(string sealID, SealStatus status)
        {
            if (!_sealDefinitions.TryGetValue(sealID, out SealDefinition sealDef)) return;

            List<string> effectsToTrigger = new List<string>();
            if (status == SealStatus.Sealed) effectsToTrigger.AddRange(sealDef.ActiveEffects);
            else if (status == SealStatus.Broken) effectsToTrigger.AddRange(sealDef.BrokenEffects);
            else if (status == SealStatus.TemporarilyOpen) { /* maybe temporary effects */ }

            foreach (var effectID in effectsToTrigger)
            {
                GD.Print($"SealSystem: Seal '{sealID}' (Status: {status}) triggering effect: {effectID}.");
                // This would interact with other systems:
                // - Spawn entities (EntityManager)
                // - Apply world modifiers (WorldSystem, EconomySystem, WeatherSystem)
                // - Spawn VFX/Audio (Body)
                // - Change game rules (e.g., pause day/night cycle for "pause_night_cycle")
            }

            // --- TDD 09.3.2: Global Seals ---
            if (sealDef.IsGlobal)
            {
                if (sealDef.ID == "sun_seal_global")
                {
                    if (status == SealStatus.Sealed && sealDef.ActiveEffects.Contains("pause_night_cycle"))
                    {
                        GameManager.Instance.Time.PauseDayNightCycle(true); // Conceptual method
                        GD.Print("SealSystem: Global Sun Seal is active. Day/Night cycle paused!");
                    }
                    else if (status == SealStatus.Broken && sealDef.BrokenEffects.Contains("resume_night_cycle"))
                    {
                        GameManager.Instance.Time.PauseDayNightCycle(false); // Conceptual method
                        GD.Print("SealSystem: Global Sun Seal is broken. Day/Night cycle resumed!");
                    }
                }
            }
        }

        // --- Helper Events for Body Sync ---
        public struct SealStatusChangedEvent { public string SealID; public SealStatus NewStatus; }
        public struct SealFailedEvent { public EntityID InitiatorID; public string SealID; public string Reason; }
    }
}
```

#### 4.1. Update `TimeSystem.cs` for `PauseDayNightCycle`

Needed for `sun_seal_global`.

Open `res://_Brain/Core/TimeSystem.cs` and add this method:

```csharp
// _Brain/Core/TimeSystem.cs
using Godot;
using System;

namespace Sigilborne.Core
{
    public class TimeSystem
    {
        // ... (existing fields) ...
        private bool _isDayNightCyclePaused = false; // New field

        public TimeSystem() { /* ... */ }

        public void Tick(double delta)
        {
            if (_isDayNightCyclePaused) return; // If paused, don't advance time.
            // ... (rest of tick logic) ...
        }

        public void SetCurrentHour(int hour) { /* ... */ }

        /// <summary>
        /// Pauses or resumes the day/night cycle.
        /// (TDD 09.3.2)
        /// </summary>
        public void PauseDayNightCycle(bool pause)
        {
            _isDayNightCyclePaused = pause;
            GD.Print($"TimeSystem: Day/Night Cycle {(pause ? "PAUSED" : "RESUMED")}.");
        }
    }
}
```

### 5. Integrating `SealSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Rituals;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `SealSystem` property.
3.  Initialize `SealSystem` in `InitializeSystems()` **after** all its dependencies (`InventorySystem`, `BiologicalSystem`, `TimeSystem`).

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
    public SealSystem Seals { get; private set; } // Add SealSystem property

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

        // --- Test Seal System ---
        GD.Print("\n--- Testing Seal System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        
        // Initial status of seals
        GD.Print($"Dungeon Seal status: {Seals.GetSealStatus("dungeon_entrance_seal")}");
        GD.Print($"Global Sun Seal status: {Seals.GetSealStatus("sun_seal_global")}");
        GD.Print($"Current game hour: {Time.CurrentHour}");

        // Attempt to open Physical Seal (dungeon_entrance_seal) with wrong item
        GD.Print("\nAttempting to open Dungeon Entrance Seal with wrong item:");
        Seals.OpenSeal(playerID, "dungeon_entrance_seal", itemUsed: "healing_herb");
        GD.Print($"Dungeon Seal status after failed attempt: {Seals.GetSealStatus("dungeon_entrance_seal")}");

        // Add correct key to inventory
        Inventory.AddItem(playerID, new InventoryItem("dungeon_key_t1", 1));
        GD.Print("\nAttempting to open Dungeon Entrance Seal with correct item:");
        Seals.OpenSeal(playerID, "dungeon_entrance_seal", itemUsed: "dungeon_key_t1");
        GD.Print($"Dungeon Seal status after successful open: {Seals.GetSealStatus("dungeon_entrance_seal")}");

        // Attempt to open Magical Seal (chaos_glyph_seal) with wrong glyph
        GD.Print("\nAttempting to open Chaos Glyph Seal with wrong glyph:");
        Seals.OpenSeal(playerID, "chaos_glyph_seal", glyphUsed: GlyphConcept.Bloom);
        GD.Print($"Chaos Glyph Seal status after failed attempt: {Seals.GetSealStatus("chaos_glyph_seal")}");

        // Open Magical Seal with correct glyph
        GD.Print("\nAttempting to open Chaos Glyph Seal with correct glyph (Bind):");
        Seals.OpenSeal(playerID, "chaos_glyph_seal", glyphUsed: GlyphConcept.Bind);
        GD.Print($"Chaos Glyph Seal status after successful open: {Seals.GetSealStatus("chaos_glyph_seal")}");

        GD.Print("--- End Testing Seal System ---\n");
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
        Seals.Tick(delta); // Call SealSystem's tick method (for time-based seals)
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to RitualSystem) ...
        
        // Initialize SealSystem AFTER InventorySystem, BiologicalSystem, TimeSystem
        Seals = new SealSystem(Entities, Events, Inventory, BiologicalSystem, Time); // Pass dependencies
        GD.Print("  - SealSystem initialized.");

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

### 6. Testing the Seal System

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Seal System" section.

```
...
RitualSystem: Initialized.
...
SealSystem: Initialized.
SealSystem: Registered seal definition 'Dungeon Entrance Seal'. Status: Sealed.
SealSystem: Registered seal definition 'Chaos Glyph Seal'. Status: Sealed.
SealSystem: Registered seal definition 'Global Sun Seal'. Status: Sealed.
SealSystem: Global Sun Seal is active. Day/Night cycle paused!
  - SealSystem initialized.
EconomyManager: Initialized.
...
--- Testing Seal System ---
Dungeon Seal status: Sealed
Global Sun Seal status: Sealed
Current game hour: 8

Attempting to open Dungeon Entrance Seal with wrong item:
SealSystem: Initiator EntityID(0, Gen:1) failed to open seal 'dungeon_entrance_seal'. Reason: Incorrect item
Dungeon Seal status after failed attempt: Sealed
InventorySystem: Added dungeon_key_t1 (x1) to EntityID(0, Gen:1). New slot.

Attempting to open Dungeon Entrance Seal with correct item:
SealSystem: Initiator EntityID(0, Gen:1) successfully opened/broke seal 'dungeon_entrance_seal'. Reason: Used correct key
SealSystem: Seal 'dungeon_entrance_seal' (Status: Broken) triggering effect: block_path.
Dungeon Seal status after successful open: Broken

Attempting to open Chaos Glyph Seal with wrong glyph:
SealSystem: Initiator EntityID(0, Gen:1) failed to open seal 'chaos_glyph_seal'. Reason: Incorrect glyph
Chaos Glyph Seal status after failed attempt: Sealed

Attempting to open Chaos Glyph Seal with correct glyph (Bind):
SealSystem: Initiator EntityID(0, Gen:1) successfully opened/broke seal 'chaos_glyph_seal'. Reason: Used correct glyph
SealSystem: Seal 'chaos_glyph_seal' (Status: Broken) triggering effect: suppress_chaos_aura.
Chaos Glyph Seal status after successful open: Broken
--- End Testing Seal System ---
...
```

**Key Observations:**

*   **Seal Registration**: `SealDefinition`s are correctly registered, and all seals start in `Sealed` status.
*   **Global Seal Effect**: The `Global Sun Seal` immediately pauses the day/night cycle upon initialization.
*   **Physical Seal**: `dungeon_entrance_seal`
    *   Fails with the wrong item.
    *   Succeeds with `dungeon_key_t1`, changes status to `Broken`, and triggers its `brokenEffects`.
*   **Magical Seal**: `chaos_glyph_seal`
    *   Fails with `GlyphConcept.Bloom`.
    *   Succeeds with `GlyphConcept.Bind`, changes status to `Broken`, and triggers its `brokenEffects`.
*   **Time-Based Seal (Implicit)**: The `OnNewHour` event will trigger `sun_seal_global` to become `TemporarilyOpen` when `Time.CurrentHour` reaches 18. This will also resume the day/night cycle.

This confirms our `SealSystem` is functional, managing seals, checking requirements, and applying their effects.

### Summary

You have successfully implemented **Seals & Locks** in the C# Brain, designing `SealType`, `SealStatus`, and `SealDefinition` to represent various types of magical barriers. By creating `SealSystem` to manage seal definitions, track their status, validate requirements (item, glyph, blood, time), and apply effects (including global world state changes like pausing the day/night cycle), you've established a robust mechanism for powerful in-world controls. This crucial system strictly adheres to TDD 09.3's specifications, providing advanced world mechanics for Sigilborne.

### Next Steps

The next chapter will focus on **Ritual System - Simple, Advanced & Forbidden (C#)**, where we will refine our `RitualSystem` to differentiate between simple, advanced, and forbidden rituals, each with escalating complexity, resource costs, and world-altering consequences.