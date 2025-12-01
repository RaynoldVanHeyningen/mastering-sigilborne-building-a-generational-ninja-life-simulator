## Chapter 8.1: Faction System - The Relationship Graph (C#)

Welcome to **Module 8: Society, Politics & Economy**! Sigilborne's world is a complex tapestry of competing clans and factions. This module begins by implementing the **Faction System** in the C# Brain, designing the core data structures for managing faction relationships as a "Relationship Graph" and defining their dynamic interactions, as specified in TDD 06.2.

### 1. The Dynamic Nature of Faction Politics

The GDD (B17.1) states: "Every political actor is a living organism. Factions and clans evolve, fracture, merge, mutate, and occasionally die." This requires a system that:

*   **Models Relationships**: Tracks trust, fear, and hostility between groups.
*   **Dynamic States**: Relationships change over time due to events.
*   **Hierarchical Structure**: Clans belong to major factions (B17.2).
*   **Player Influence**: Player actions can shift political alignments (B17.10).

### 2. Defining `FactionType`, `FactionRelationState`, and `Faction`

We need enums for different types of factions and relationship states, and a class to represent a single faction.

1.  Create `res://_Brain/Systems/Factions/` folder.
2.  Create `res://_Brain/Systems/Factions/FactionEnums.cs`:

```csharp
// _Brain/Systems/Factions/FactionEnums.cs
using System;

namespace Sigilborne.Systems.Factions
{
    /// <summary>
    /// Defines the broad categories of major factions in the world.
    /// (GDD B17.2.2: Major Faction Archetypes)
    /// </summary>
    public enum FactionType
    {
        None,
        WarriorDominion,    // Pulse/Fracture aligned
        TempleAlliance,     // Bloom/Bind aligned
        SpiritWovenTribe,   // Echo/Flux aligned
        ScholarConfederation, // Veil/Conjure aligned
        CorruptedCult,      // Consume/Chain aligned (C04)
        TradeConsortium,    // Neutral/Economic
        BorderCoalition,    // Defensive
        RogueGroup          // Outlaw (GDD B22.7)
    }

    /// <summary>
    /// Defines the visible state of a relationship between two factions.
    /// (GDD B17.5: Diplomacy & Relationships)
    /// </summary>
    public enum FactionRelationState
    {
        Ally,       // Strongly positive, cooperative
        Friendly,   // Positive, generally helpful
        Neutral,    // Indifferent, no strong feelings
        Tense,      // Negative, distrustful, cautious
        Hostile,    // Actively aggressive, will attack
        War         // Open warfare, full conflict
    }
}
```

3.  Create `res://_Brain/Systems/Factions/Faction.cs`:

```csharp
// _Brain/Systems/Factions/Faction.cs
using System;
using System.Collections.Generic;
using Godot; // For GD.Print

namespace Sigilborne.Systems.Factions
{
    /// <summary>
    /// Represents a single major faction in the world.
    /// (TDD 06.2.1: The Relationship Graph)
    /// </summary>
    public class Faction
    {
        public string ID { get; private set; } // Unique ID (e.g., "CrimsonBladeClan", "SunkenTemple")
        public string Name { get; private set; }
        public FactionType Type { get; private set; }
        public string Description { get; private set; }

        // Core ideological glyph concepts (GDD B17.7)
        public List<GlyphConcept> PrimaryConcepts { get; private set; }
        public List<GlyphConcept> TabooConcepts { get; private set; }

        public Faction(string id, string name, FactionType type, string description, List<GlyphConcept> primaryConcepts = null, List<GlyphConcept> tabooConcepts = null)
        {
            ID = id;
            Name = name;
            Type = type;
            Description = description;
            PrimaryConcepts = primaryConcepts ?? new List<GlyphConcept>();
            TabooConcepts = tabooConcepts ?? new List<GlyphConcept>();
        }

        public override string ToString()
        {
            return $"Faction: '{Name}' ({Type}) | Primary: {string.Join(", ", PrimaryConcepts)}";
        }
    }
}
```

### 3. Implementing `FactionSystem.cs`

This system will manage all `Faction` instances and their relationships.

1.  Create `res://_Brain/Systems/Factions/FactionSystem.cs`:

```csharp
// _Brain/Systems/Factions/FactionSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic; // For GlyphConcept
using System.Linq;

namespace Sigilborne.Systems.Factions
{
    /// <summary>
    /// Represents the relationship between two factions.
    /// (TDD 06.2.1: The Relationship Graph)
    /// </summary>
    public struct FactionRelation
    {
        public EntityID FactionA_ID; // Using EntityID for factions too, or string ID? TDD says string. Let's use string.
        public string FactionA_ID_Str;
        public string FactionB_ID_Str;
        public float Value; // -100 to 100 (TDD 06.2.1)
        public FactionRelationState State; // Ally/Friendly/Neutral/Tense/Hostile/War (TDD 06.2.1)

        public FactionRelation(string factionA, string factionB, float value = 0f)
        {
            FactionA_ID_Str = factionA;
            FactionB_ID_Str = factionB;
            Value = value;
            State = GetStateFromValue(value);
        }

        public void UpdateValue(float change)
        {
            Value = Mathf.Clamp(Value + change, -100f, 100f);
            State = GetStateFromValue(Value);
        }

        public static FactionRelationState GetStateFromValue(float value)
        {
            if (value >= 80) return FactionRelationState.Ally;
            if (value >= 40) return FactionRelationState.Friendly;
            if (value > -40) return FactionRelationState.Neutral;
            if (value > -80) return FactionRelationState.Tense;
            return FactionRelationState.Hostile; // Below -80, consider War
        }

        public override string ToString()
        {
            return $"Rel: {FactionA_ID_Str} <-> {FactionB_ID_Str} | Val: {Value:F0} ({State})";
        }
    }


    /// <summary>
    /// Manages all Faction instances and their relationships within the world.
    /// (TDD 06.2.1)
    /// </summary>
    public class FactionSystem
    {
        private EventBus _eventBus;
        private Dictionary<string, Faction> _factions = new Dictionary<string, Faction>();
        // The relationship graph: Key is a combined string "FactionA_FactionB" (alphabetically sorted)
        private Dictionary<string, FactionRelation> _relationships = new Dictionary<string, FactionRelation>();

        public FactionSystem(EventBus eventBus)
        {
            _eventBus = eventBus;
            RegisterDefaultFactions(); // Register some default factions
            GD.Print("FactionSystem: Initialized.");
        }

        /// <summary>
        /// Registers a new static Faction definition.
        /// </summary>
        public void RegisterFaction(Faction faction)
        {
            _factions[faction.ID] = faction;
            GD.Print($"FactionSystem: Registered faction '{faction.Name}'.");
        }

        /// <summary>
        /// Registers some common default factions for initial testing.
        /// (GDD B17.2.2: Examples of major faction archetypes)
        /// </summary>
        private void RegisterDefaultFactions()
        {
            RegisterFaction(new Faction("crimson_blades", "Crimson Blades", FactionType.WarriorDominion, "A martial clan focused on combat mastery.", new List<GlyphConcept> { GlyphConcept.Pulse, GlyphConcept.Fracture }));
            RegisterFaction(new Faction("sunken_temple", "Sunken Temple", FactionType.TempleAlliance, "An ancient order dedicated to healing and stability.", new List<GlyphConcept> { GlyphConcept.Bloom, GlyphConcept.Bind }));
            RegisterFaction(new Faction("shadow_veilers", "Shadow Veilers", FactionType.ScholarConfederation, "Masters of espionage and information control.", new List<GlyphConcept> { GlyphConcept.Veil, GlyphConcept.Echo }));
            RegisterFaction(new Faction("void_cultists", "Void Cultists", FactionType.CorruptedCult, "Worshippers of chaos and forbidden arts.", new List<GlyphConcept> { GlyphConcept.Consume, GlyphConcept.Flux }, new List<GlyphConcept> { GlyphConcept.Bloom, GlyphConcept.Bind }));
            RegisterFaction(new Faction("free_wanderers", "Free Wanderers", FactionType.RogueGroup, "Independent nomads, valuing freedom above all.", new List<GlyphConcept> { GlyphConcept.Flux }));

            // Establish initial relationships
            SetRelationship("crimson_blades", "sunken_temple", 50f); // Friendly
            SetRelationship("crimson_blades", "void_cultists", -80f); // Hostile
            SetRelationship("sunken_temple", "void_cultists", -90f); // Hostile
            SetRelationship("shadow_veilers", "crimson_blades", 10f); // Neutral
            SetRelationship("shadow_veilers", "void_cultists", -20f); // Tense
            SetRelationship("free_wanderers", "crimson_blades", -10f); // Tense
        }

        /// <summary>
        /// Sets or updates the relationship value between two factions.
        /// (TDD 06.2.1: Dynamics)
        /// </summary>
        public void SetRelationship(string factionA_ID, string factionB_ID, float value)
        {
            if (factionA_ID == factionB_ID) return; // Cannot set relationship with self

            string relationshipKey = GetRelationshipKey(factionA_ID, factionB_ID);
            if (!_relationships.ContainsKey(relationshipKey))
            {
                _relationships[relationshipKey] = new FactionRelation(factionA_ID, factionB_ID, value);
            }
            else
            {
                FactionRelation rel = _relationships[relationshipKey];
                rel.Value = value;
                rel.State = FactionRelation.GetStateFromValue(value);
                _relationships[relationshipKey] = rel; // Update struct in dictionary
            }
            GD.Print($"FactionSystem: Relationship {relationshipKey} set to {value:F0} ({_relationships[relationshipKey].State}).");
            _eventBus.Publish(new FactionRelationshipChangedEvent { FactionA_ID = factionA_ID, FactionB_ID = factionB_ID, NewRelation = _relationships[relationshipKey] });
        }

        /// <summary>
        /// Adjusts the relationship value between two factions by a given change.
        /// </summary>
        public void AdjustRelationship(string factionA_ID, string factionB_ID, float change)
        {
            if (factionA_ID == factionB_ID) return;

            string relationshipKey = GetRelationshipKey(factionA_ID, factionB_ID);
            if (!_relationships.ContainsKey(relationshipKey))
            {
                // Create with initial change
                _relationships[relationshipKey] = new FactionRelation(factionA_ID, factionB_ID, change);
            }
            else
            {
                FactionRelation rel = _relationships[relationshipKey];
                rel.UpdateValue(change);
                _relationships[relationshipKey] = rel;
            }
            GD.Print($"FactionSystem: Relationship {relationshipKey} adjusted by {change:F0}. New Value: {_relationships[relationshipKey].Value:F0} ({_relationships[relationshipKey].State}).");
            _eventBus.Publish(new FactionRelationshipChangedEvent { FactionA_ID = factionA_ID, FactionB_ID = factionB_ID, NewRelation = _relationships[relationshipKey] });
        }

        /// <summary>
        /// Retrieves the relationship between two factions.
        /// </summary>
        public FactionRelation GetRelationship(string factionA_ID, string factionB_ID)
        {
            if (factionA_ID == factionB_ID) return new FactionRelation(factionA_ID, factionB_ID, 100f); // Self is always Ally
            string relationshipKey = GetRelationshipKey(factionA_ID, factionB_ID);
            if (_relationships.TryGetValue(relationshipKey, out FactionRelation rel))
            {
                return rel;
            }
            return new FactionRelation(factionA_ID, factionB_ID, 0f); // Default Neutral
        }

        /// <summary>
        /// Helper to create a consistent key for relationship dictionary.
        /// </summary>
        private string GetRelationshipKey(string factionA_ID, string factionB_ID)
        {
            // Sort alphabetically to ensure key is consistent regardless of order (A_B vs B_A)
            return string.CompareOrdinal(factionA_ID, factionB_ID) < 0
                ? $"{factionA_ID}_{factionB_ID}"
                : $"{factionB_ID}_{factionA_ID}";
        }

        /// <summary>
        /// Retrieves a faction by its ID.
        /// </summary>
        public Faction GetFaction(string factionID)
        {
            _factions.TryGetValue(factionID, out Faction faction);
            return faction;
        }

        /// <summary>
        /// Retrieves all registered factions.
        /// </summary>
        public IReadOnlyList<Faction> GetAllFactions()
        {
            return _factions.Values.ToList().AsReadOnly();
        }

        // --- Helper Events for Body Sync ---
        public struct FactionRelationshipChangedEvent { public string FactionA_ID; public string FactionB_ID; public FactionRelation NewRelation; }
    }
}
```

### 4. Integrating `FactionSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Factions;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `FactionSystem` property.
3.  Initialize `FactionSystem` in `InitializeSystems()`.

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
using Sigilborne.Systems.Factions; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public EcologyManager Ecology { get; private set; }
    public FactionSystem Factions { get; private set; } // Add FactionSystem property

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

        // --- Test Faction System ---
        GD.Print("\n--- Testing Faction System ---");
        GD.Print("All Factions:");
        foreach (var faction in Factions.GetAllFactions())
        {
            GD.Print($"  - {faction}");
        }

        // Test relationships
        GD.Print($"Relationship Crimson Blades <-> Sunken Temple: {Factions.GetRelationship("crimson_blades", "sunken_temple")}");
        GD.Print($"Relationship Crimson Blades <-> Void Cultists: {Factions.GetRelationship("crimson_blades", "void_cultists")}");
        GD.Print($"Relationship Sunken Temple <-> Shadow Veilers: {Factions.GetRelationship("sunken_temple", "shadow_veilers")}"); // Default neutral

        // Adjust a relationship
        Factions.AdjustRelationship("shadow_veilers", "sunken_temple", 30f); // Make them friendly
        GD.Print($"Relationship Sunken Temple <-> Shadow Veilers (adjusted): {Factions.GetRelationship("sunken_temple", "shadow_veilers")}");

        // Adjust to hostile
        Factions.AdjustRelationship("free_wanderers", "void_cultists", -70f); // Make them hostile
        GD.Print($"Relationship Free Wanderers <-> Void Cultists (adjusted): {Factions.GetRelationship("free_wanderers", "void_cultists")}");

        GD.Print("--- End Testing Faction System ---\n");
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
        // FactionSystem doesn't have a Tick method for now; its operations are event/command driven.
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to EcologyManager) ...
        
        // Initialize FactionSystem
        Factions = new FactionSystem(Events); // Pass EventBus
        GD.Print("  - FactionSystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation(Ecology);
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `EventBus.cs` for `FactionRelationshipChangedEvent`

Open `res://_Brain/Core/EventBus.cs` and add `OnFactionRelationshipChanged` delegate.

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
using Sigilborne.Systems.Factions; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Faction System Events (TDD 06.2.1)
        public event Action<string, string, FactionRelation> OnFactionRelationshipChanged; // FactionA_ID, FactionB_ID, NewRelation

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is FactionSystem.FactionRelationshipChangedEvent factionRelEvent) // New condition
            {
                OnFactionRelationshipChanged?.Invoke(factionRelEvent.FactionA_ID, factionRelEvent.FactionB_ID, factionRelEvent.NewRelation);
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

### 5. Testing the Faction System

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Faction System" section.

```
...
FactionSystem: Initialized.
FactionSystem: Registered faction 'Crimson Blades'.
FactionSystem: Registered faction 'Sunken Temple'.
FactionSystem: Registered faction 'Shadow Veilers'.
FactionSystem: Registered faction 'Void Cultists'.
FactionSystem: Registered faction 'Free Wanderers'.
FactionSystem: Relationship crimson_blades_sunken_temple set to 50 (Friendly).
FactionSystem: Relationship crimson_blades_void_cultists set to -80 (Hostile).
FactionSystem: Relationship sunken_temple_void_cultists set to -90 (Hostile).
FactionSystem: Relationship crimson_blades_shadow_veilers set to 10 (Neutral).
FactionSystem: Relationship shadow_veilers_void_cultists set to -20 (Tense).
FactionSystem: Relationship free_wanderers_crimson_blades set to -10 (Tense).
  - FactionSystem initialized.
PlayerStatSystem: Initialized.
...
--- Testing Faction System ---
All Factions:
  Faction: 'Crimson Blades' (WarriorDominion) | Primary: Pulse, Fracture
  Faction: 'Sunken Temple' (TempleAlliance) | Primary: Bloom, Bind
  Faction: 'Shadow Veilers' (ScholarConfederation) | Primary: Veil, Echo
  Faction: 'Void Cultists' (CorruptedCult) | Primary: Consume, Flux
  Faction: 'Free Wanderers' (RogueGroup) | Primary: Flux
Relationship Crimson Blades <-> Sunken Temple: Rel: crimson_blades <-> sunken_temple | Val: 50 (Friendly)
Relationship Crimson Blades <-> Void Cultists: Rel: crimson_blades <-> void_cultists | Val: -80 (Hostile)
Relationship Sunken Temple <-> Shadow Veilers: Rel: shadow_veilers <-> sunken_temple | Val: 0 (Neutral)
FactionSystem: Relationship shadow_veilers_sunken_temple adjusted by 30. New Value: 30 (Neutral).
Relationship Sunken Temple <-> Shadow Veilers (adjusted): Rel: shadow_veilers <-> sunken_temple | Val: 30 (Neutral)
FactionSystem: Relationship free_wanderers_void_cultists adjusted by -70. New Value: -70 (Tense).
Relationship Free Wanderers <-> Void Cultists (adjusted): Rel: free_wanderers <-> void_cultists | Val: -70 (Tense)
--- End Testing Faction System ---
...
```

**Key Observations:**

*   **Faction Registration**: `Faction` objects are correctly registered and printed.
*   **Initial Relationships**: Default relationships are set, and their `FactionRelationState` is correctly derived from the `Value`.
*   **Relationship Adjustment**: `AdjustRelationship` correctly updates the `Value` and transitions the `State` (e.g., from `Neutral` to `Friendly`).
*   **Key Generation**: The `GetRelationshipKey` ensures consistent key generation (e.g., `crimson_blades_sunken_temple` is the same as `sunken_temple_crimson_blades`).

This confirms our `FactionSystem` is functional, managing factions and their dynamic relationships using a relationship graph.

### Summary

You have successfully implemented the **Faction System** in the C# Brain, designing `FactionType`, `FactionRelationState`, and `Faction` classes to represent political actors and their relationships. By creating `FactionSystem` to manage faction definitions and their dynamic `FactionRelation`s using a relationship graph, you've established the core mechanism for political simulation. This crucial system strictly adheres to TDD 06.2's specifications, providing the foundational layer for complex societal interactions and emergent political narratives in Sigilborne.

### Next Steps

The next chapter will focus on **The Simulation Clock**, detailing how the world's internal "Game Time" operates at a slower tick rate than the rendering frame, and how heavy system updates (like Faction AI planning) are load-balanced across this daily cycle.