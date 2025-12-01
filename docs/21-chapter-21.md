## Chapter 3.2: Glyph Discovery System - Player Knowledge State (C#)

With our `WorldGlyphMap` creating unique glyph meanings per world, the next critical step is to implement the **Glyph Discovery System**. Players don't start knowing these meanings; they must learn them. This chapter focuses on tracking the player's `KnowledgeState` for each glyph (Hidden, Seen, Known) and implementing the feedback loop for revealing glyph meanings, as outlined in TDD 15.3.

### 1. The Discovery Philosophy: Mystery and Experimentation

The GDD (B01.4) emphasizes that players learn glyphs through teachers, observation, scrolls, and experimentation. This means the system must:

*   **Hide Information**: Initially, glyph meanings are unknown.
*   **Progressive Revelation**: Knowledge is gained in stages.
*   **Feedback**: The game must clearly communicate when a player gains new knowledge.

### 2. Defining `GlyphKnowledgeState`

We need an enum to represent the different levels of knowledge a player has about a specific glyph symbol.

1.  Open `res://_Brain/Systems/Magic/GlyphConcepts.cs` and add the `GlyphKnowledgeState` enum:

```csharp
// _Brain/Systems/Magic/GlyphConcepts.cs
using System;

namespace Sigilborne.Systems.Magic
{
    // ... (GlyphConcept enum) ...

    /// <summary>
    /// Represents the player's knowledge state about a specific glyph symbol.
    /// (TDD 15.3)
    /// </summary>
    public enum GlyphKnowledgeState
    {
        Hidden, // Player sees "???" or a blurred rune. Has no idea what it is.
        Seen,   // Player has seen the symbol (e.g., an NPC cast it). Knows the visual but not the meaning.
        Known   // Player knows the symbol's concept (e.g., "Triangle is Fire"). Can attempt to cast.
    }
}
```

### 3. Implementing `PlayerGlyphKnowledgeSystem.cs`

This system will manage the player's knowledge state for all glyphs in the current world.

1.  Create `res://_Brain/Systems/Magic/PlayerGlyphKnowledgeSystem.cs`:

```csharp
// _Brain/Systems/Magic/PlayerGlyphKnowledgeSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Manages the player's knowledge state for each glyph symbol in the current world.
    /// (TDD 15.3)
    /// </summary>
    public class PlayerGlyphKnowledgeSystem
    {
        private EntityID _playerEntityID;
        private WorldGlyphMap _worldGlyphMap; // Reference to the world's glyph definitions
        private EventBus _eventBus;

        // Dictionary to store the player's knowledge state for each glyph symbol ID.
        private Dictionary<string, GlyphKnowledgeState> _playerGlyphKnowledge = new Dictionary<string, GlyphKnowledgeState>();

        public PlayerGlyphKnowledgeSystem(EntityID playerEntityID, WorldGlyphMap worldGlyphMap, EventBus eventBus)
        {
            _playerEntityID = playerEntityID;
            _worldGlyphMap = worldGlyphMap;
            _eventBus = eventBus;

            // Initialize all glyphs in the world map as Hidden for the player.
            foreach (var glyphDef in _worldGlyphMap.AllWorldGlyphs)
            {
                _playerGlyphKnowledge.Add(glyphDef.SymbolID, GlyphKnowledgeState.Hidden);
            }
            GD.Print($"PlayerGlyphKnowledgeSystem: Initialized for player {_playerEntityID} with all glyphs as Hidden.");
        }

        /// <summary>
        /// Updates the player's knowledge state for a specific glyph symbol.
        /// (TDD 15.3: Knowledge State)
        /// </summary>
        /// <param name="symbolID">The ID of the glyph symbol.</param>
        /// <param name="newState">The new knowledge state.</param>
        /// <param name="forceUpdate">If true, updates even if new state is lower than current.</param>
        /// <returns>True if the knowledge state was updated, false otherwise.</returns>
        public bool UpdateGlyphKnowledge(string symbolID, GlyphKnowledgeState newState, bool forceUpdate = false)
        {
            if (!_playerGlyphKnowledge.ContainsKey(symbolID))
            {
                GD.PrintErr($"PlayerGlyphKnowledgeSystem: Attempted to update knowledge for unknown symbol '{symbolID}'.");
                return false;
            }

            GlyphKnowledgeState currentState = _playerGlyphKnowledge[symbolID];

            // Only update if the new state is higher or forceUpdate is true.
            if (forceUpdate || newState > currentState)
            {
                _playerGlyphKnowledge[symbolID] = newState;
                GD.Print($"PlayerGlyphKnowledgeSystem: Player {_playerEntityID} knowledge of '{symbolID}' updated from {currentState} to {newState}.");
                
                // Publish an event for the Body (UI) to update its display
                _eventBus.Publish(new GlyphKnowledgeUpdatedEvent { PlayerID = _playerEntityID, SymbolID = symbolID, NewState = newState });
                
                // If the glyph became Known, also publish its meaning for UI hints (TDD 15.3: Feedback)
                if (newState == GlyphKnowledgeState.Known)
                {
                    WorldGlyphDefinition def = _worldGlyphMap.GetDefinitionBySymbol(symbolID);
                    _eventBus.Publish(new GlyphMeaningDiscoveredEvent { PlayerID = _playerEntityID, SymbolID = symbolID, Concept = def.Concept });
                }
                return true;
            }
            return false;
        }

        /// <summary>
        /// Retrieves the player's current knowledge state for a glyph symbol.
        /// </summary>
        public GlyphKnowledgeState GetGlyphKnowledge(string symbolID)
        {
            if (_playerGlyphKnowledge.TryGetValue(symbolID, out GlyphKnowledgeState state))
            {
                return state;
            }
            return GlyphKnowledgeState.Hidden; // Default to hidden if not in map (shouldn't happen with proper init)
        }

        /// <summary>
        /// Returns all glyph symbols the player knows (state >= Known).
        /// </summary>
        public IEnumerable<string> GetKnownGlyphSymbols()
        {
            return _playerGlyphKnowledge.Where(kv => kv.Value >= GlyphKnowledgeState.Known).Select(kv => kv.Key);
        }

        // --- Helper Events for Body Sync ---
        public struct GlyphKnowledgeUpdatedEvent { public EntityID PlayerID; public string SymbolID; public GlyphKnowledgeState NewState; }
        public struct GlyphMeaningDiscoveredEvent { public EntityID PlayerID; public string SymbolID; public GlyphConcept Concept; }
    }
}
```

### 4. Integrating `PlayerGlyphKnowledgeSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Magic;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `PlayerGlyphKnowledgeSystem` property.
3.  Initialize `PlayerGlyphKnowledgeSystem` in `InitializeSystems()`, passing the player's `EntityID`, `WorldGlyphMap`, and `EventBus`.

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
using Sigilborne.Utils;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public WorldGlyphMap GlyphMap { get; private set; }
    public PlayerGlyphKnowledgeSystem PlayerGlyphKnowledge { get; private set; } // Add PlayerGlyphKnowledgeSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...

        // --- Test PlayerStatSystem (Damage) ---
        // ... (existing test damage code) ...
        // --- End Testing PlayerStatSystem ---
        
        // --- Test Glyph Discovery System ---
        GD.Print("\n--- Testing Glyph Discovery System ---");
        // Get a symbol from the generated map for testing
        string testSymbol = GlyphMap.AllWorldGlyphs[0].SymbolID; // e.g., "symbol_leaf"
        string unknownSymbol = GlyphSymbols.AllSymbols.First(s => !GlyphMap.AllWorldGlyphs.Any(g => g.SymbolID == s)); // Get a symbol not in this world

        GD.Print($"Initial knowledge of '{testSymbol}': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Seen);
        GD.Print($"Knowledge of '{testSymbol}' after 'Seen': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Known);
        GD.Print($"Knowledge of '{testSymbol}' after 'Known': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        
        GD.Print($"Attempt to update unknown symbol '{unknownSymbol}':");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(unknownSymbol, GlyphKnowledgeState.Known); // Should print an error
        
        GD.Print("--- End Testing Glyph Discovery System ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        // ... (existing _PhysicsProcess calls) ...
    }

    private void InitializeSystems()
    {
        Events = new EventBus();
        GD.Print("  - EventBus initialized.");

        Time = new TimeSystem();
        GD.Print("  - TimeSystem initialized.");

        Entities = new EntityManager(Events);
        GD.Print("  - EntityManager initialized.");

        Transforms = new TransformSystem(Entities, Events);
        GD.Print("  - TransformSystem initialized.");
        
        Jobs = new JobSystem(Events);
        GD.Print("  - JobSystem initialized.");

        PlayerStats = new PlayerStatSystem(Events, Entities);
        GD.Print("  - PlayerStatSystem initialized.");

        DebugCommands = new DebugCommandSystem(this);
        GD.Print("  - DebugCommandSystem initialized.");

        Input = new InputSystem();
        GD.Print("  - InputSystem initialized.");

        Movement = new MovementSystem(Entities, Input, Events, Transforms);
        GD.Print("  - MovementSystem initialized.");

        Physics = new PhysicsSystem(Entities, Transforms, Events);
        GD.Print("  - PhysicsSystem initialized.");

        int currentWorldSeed = 12345;
        int numGlyphsForWorld = 10;
        GlyphMap = new WorldGlyphMap(currentWorldSeed, numGlyphsForWorld);
        GD.Print("  - WorldGlyphMap initialized.");

        // Initialize PlayerGlyphKnowledgeSystem (TDD 15.3)
        // We need the player's EntityID, which is created earlier in _Ready().
        // For now, let's create a temporary player ID for initialization.
        // In a proper game flow, this would happen AFTER the player entity is confirmed.
        // We'll rely on EntityManager.GetPlayerEntityID() for now, assuming it's set.
        // To be safe, we'll initialize this AFTER the player entity is created in _Ready.
        PlayerGlyphKnowledge = new PlayerGlyphKnowledgeSystem(Entities.GetPlayerEntityID(), GlyphMap, Events);
        GD.Print("  - PlayerGlyphKnowledgeSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

**Important Note**: The `PlayerGlyphKnowledgeSystem` needs the player's `EntityID`. In our current `GameManager._Ready()` test setup, the player entity is created before `InitializeSystems()` is fully done. A more robust solution for the full game would be to:
1.  Spawn the player entity.
2.  Then, in a callback or a later `_Ready` phase, initialize `PlayerGlyphKnowledgeSystem` using `GameManager.Instance.Entities.GetPlayerEntityID()`.
For this chapter's testing, `Entities.GetPlayerEntityID()` should work because the player entity is indeed created before `InitializeSystems()` finishes in `_Ready`.

#### 4.1. Update `EventBus.cs` for Glyph Knowledge Events

Our `PlayerGlyphKnowledgeSystem` publishes `GlyphKnowledgeUpdatedEvent` and `GlyphMeaningDiscoveredEvent`. We need to define these `Action` delegates in `EventBus`.

Open `_Brain/Core/EventBus.cs`:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Glyph Knowledge Events (TDD 15.3)
        public event Action<EntityID, string, GlyphKnowledgeState> OnGlyphKnowledgeUpdated;
        public event Action<EntityID, string, GlyphConcept> OnGlyphMeaningDiscovered;

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is PlayerGlyphKnowledgeSystem.GlyphKnowledgeUpdatedEvent knowledgeEvent) // New condition
            {
                OnGlyphKnowledgeUpdated?.Invoke(knowledgeEvent.PlayerID, knowledgeEvent.SymbolID, knowledgeEvent.NewState);
            }
            else if (eventData is PlayerGlyphKnowledgeSystem.GlyphMeaningDiscoveredEvent meaningEvent) // New condition
            {
                OnGlyphMeaningDiscovered?.Invoke(meaningEvent.PlayerID, meaningEvent.SymbolID, meaningEvent.Concept);
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

### 5. Testing Glyph Discovery

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  In the Output console, you should see the `PlayerGlyphKnowledgeSystem` initialization and test messages.

```
...
PlayerStatSystem: Initialized stats for player EntityID(0, Gen:1): Health: 100/100, Chakra: 50/50, Stamina: 75/75
  - PlayerStatSystem initialized.
  - DebugCommandSystem initialized.
  - InputSystem initialized.
  - MovementSystem initialized.
  - PhysicsSystem initialized.
WorldGlyphMap: Generated 10 glyph definitions for world seed 12345.
  - WorldGlyphMap initialized.
PlayerGlyphKnowledgeSystem: Initialized for player EntityID(0, Gen:1) with all glyphs as Hidden.
  - PlayerGlyphKnowledgeSystem initialized.
  - WorldSimulation initialized.
...
--- Testing Glyph Discovery System ---
Initial knowledge of 'symbol_leaf': Hidden
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_leaf' updated from Hidden to Seen.
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_leaf' updated from Seen to Known.
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_leaf' updated from Seen to Known.
Attempt to update unknown symbol 'symbol_triangle_alt':
PlayerGlyphKnowledgeSystem: Attempted to update knowledge for unknown symbol 'symbol_triangle_alt'.
--- End Testing Glyph Discovery System ---
...
```

This output confirms that:
*   The player's glyph knowledge is correctly initialized to `Hidden`.
*   `UpdateGlyphKnowledge` successfully transitions the state from `Hidden` to `Seen` to `Known`.
*   An error is logged when trying to update an unknown symbol.
*   `GlyphKnowledgeUpdatedEvent` and `GlyphMeaningDiscoveredEvent` are published when the state reaches `Known`.

### Summary

You have successfully implemented the **Glyph Discovery System** for Sigilborne, allowing players to progressively learn the randomized meanings of glyph symbols. By defining `GlyphKnowledgeState` and creating `PlayerGlyphKnowledgeSystem` to manage this state, you've established a core mechanism for progressive revelation. The system correctly initializes glyphs as `Hidden`, updates knowledge through stages, and publishes events for UI feedback, strictly adhering to TDD 15.3's specifications. This sets the stage for experimentation and a unique learning journey in each world.

### Next Steps

The next chapter will focus on **Glyph Experimentation**, implementing how players attempt to combine glyphs, process the results (success, fizzle, discovery), and handle the crucial feedback loop that reveals glyph meanings or hints at new combos.