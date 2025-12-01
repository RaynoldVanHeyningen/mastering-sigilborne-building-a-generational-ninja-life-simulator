## Chapter 3.1: Glyph Database - Concepts vs. Symbols (C#)

Welcome to **Module 3: The Glyph System - Language of Ninjutsu**! This module dives into Sigilborne's core magic system. Unlike traditional RPGs with fixed spell lists, Sigilborne's magic is dynamic: the meaning of a "hand sign" (glyph symbol) is randomized per world. This chapter lays the groundwork by separating abstract `Concepts` (the mechanical effects like Fire, Water) from `Symbols` (the visual hand signs like Triangle, Circle) and establishing a database to manage them in the C# Brain, as specified in TDD 15.2.

### 1. The Philosophical Core: Dynamic Magic

The GDD (B01.3) states that each glyph's actual effect (its "meaning") is generated uniquely per world. This is achieved by decoupling the visual representation from its mechanical function.

*   **Concepts**: These are the underlying, stable magical archetypes (e.g., Bloom, Veil, Pulse, Bind). They are universal across all worlds.
*   **Symbols**: These are the visual hand signs the player sees and performs (e.g., a specific hand gesture, a rune-like graphic). They are also universal in their visual form, but their *meaning* changes.

The "magic" happens when a `Symbol` is *mapped* to a `Concept` for a specific world.

### 2. Defining Glyph `Concepts` (Enums)

`Concepts` represent the fundamental magical archetypes. They are stable and unchanging, providing long-term player intuition.

1.  Create a new folder `res://_Brain/Systems/Magic/`.
2.  Create `res://_Brain/Systems/Magic/GlyphConcepts.cs`:

```csharp
// _Brain/Systems/Magic/GlyphConcepts.cs
using System;

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents the fundamental, stable magical archetypes (mechanical categories).
    /// These concepts are universal across all worlds.
    /// (GDD B01.2, TDD 15.2)
    /// </summary>
    public enum GlyphConcept
    {
        None,       // Default or invalid concept
        Bloom,      // Growth, expansion, healing
        Veil,       // Concealment, distortion, illusion
        Pulse,      // Shock, bursts, force
        Bind,       // Control, restraint, entanglement
        Consume,    // Drain, corruption, decay
        Fracture,   // Division, instability, breaking
        Echo,       // Duplication, resonance, memory
        Project,    // Beams, projectiles, focused energy
        Conjure,    // Materialization, summoning
        Shape,      // Form manipulation, alteration
        Flux,       // Transformation, chaotic change
        Chain,      // Propagation, linking, spreading
        // Add more concepts as defined in GDD B01.2 if needed
    }
}
```

### 3. Defining Glyph `Symbols` (IDs)

`Symbols` are the visual hand signs. They are unique and fixed in their visual appearance across the global database, but their *assigned meaning* (to a `GlyphConcept`) changes per world.

We'll define these as simple string IDs for now, representing distinct visual assets.

1.  Create `res://_Brain/Systems/Magic/GlyphSymbols.cs`:

```csharp
// _Brain/Systems/Magic/GlyphSymbols.cs
using System;
using System.Collections.Generic;

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents the static, visual hand-sign symbols available in the global pool.
    /// Each symbol has a unique string ID corresponding to a visual asset.
    /// (GDD B01.1, TDD 15.2)
    /// </summary>
    public static class GlyphSymbols
    {
        // Example static list of all available glyph symbol IDs.
        // In a real game, these might be loaded from a config file or asset manifest.
        public static readonly IReadOnlyList<string> AllSymbols = new List<string>
        {
            "symbol_triangle",
            "symbol_circle",
            "symbol_square",
            "symbol_spiral",
            "symbol_dot",
            "symbol_cross",
            "symbol_wave",
            "symbol_star",
            "symbol_moon",
            "symbol_sun",
            // Add more symbols to reach the GDD's 100+ global pool
            "symbol_diamond", "symbol_pentagon", "symbol_hexagon", "symbol_infinity",
            "symbol_lightning", "symbol_leaf", "symbol_drop", "symbol_fire",
            "symbol_eye", "symbol_mouth", "symbol_hand", "symbol_foot",
            "symbol_mountain", "symbol_river", "symbol_cloud", "symbol_tree",
            // ... and so on, up to 100+ unique visual identifiers.
        };

        /// <summary>
        /// Returns a subset of all available symbols, simulating a world drawing 50-80 glyphs
        /// from the global pool (GDD B01.2).
        /// </summary>
        /// <param name="worldSeed">The seed for world generation.</param>
        /// <param name="count">The number of symbols to select.</param>
        /// <returns>A list of selected symbol IDs for a specific world.</returns>
        public static List<string> SelectWorldSymbols(int worldSeed, int count)
        {
            if (count > AllSymbols.Count) count = AllSymbols.Count;
            if (count < 0) count = 0;

            Random rand = new Random(worldSeed);
            List<string> selected = new List<string>(AllSymbols);

            // Shuffle and take the first 'count' elements
            for (int i = selected.Count - 1; i > 0; i--)
            {
                int j = rand.Next(i + 1);
                string temp = selected[i];
                selected[i] = selected[j];
                selected[j] = temp;
            }

            return selected.Take(count).ToList();
        }
    }
}
```

### 4. The Glyph Database: `WorldGlyphMap`

This is the core data structure that stores the unique `Symbol` to `Concept` mapping for a given world. TDD 15.2 specifies that this mapping is stored in `WorldData.GlyphMap`. For now, we'll implement it as a standalone class.

1.  Create `res://_Brain/Systems/Magic/WorldGlyphMap.cs`:

```csharp
// _Brain/Systems/Magic/WorldGlyphMap.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents a single glyph definition for a specific world, mapping a visual symbol to a mechanical concept.
    /// (TDD 15.2)
    /// </summary>
    public struct WorldGlyphDefinition
    {
        public string SymbolID;     // e.g., "symbol_triangle"
        public GlyphConcept Concept; // e.g., GlyphConcept.Fire
        // Subtype information will be added in a later chapter (GDD B01.3, TDD 15.4)

        public WorldGlyphDefinition(string symbolId, GlyphConcept concept)
        {
            SymbolID = symbolId;
            Concept = concept;
        }

        public override string ToString()
        {
            return $"Symbol: '{SymbolID}' -> Concept: {Concept}";
        }
    }

    /// <summary>
    /// The authoritative database for a world's unique Glyph Symbol to Concept mapping.
    /// This is generated once per world seed.
    /// (TDD 15.2)
    /// </summary>
    public class WorldGlyphMap
    {
        private Dictionary<string, WorldGlyphDefinition> _symbolToDefinition = new Dictionary<string, WorldGlyphDefinition>();
        private Dictionary<GlyphConcept, WorldGlyphDefinition> _conceptToDefinition = new Dictionary<GlyphConcept, WorldGlyphDefinition>(); // Optional reverse lookup
        
        // Expose the definitions as a read-only list for iteration or UI display
        public IReadOnlyList<WorldGlyphDefinition> AllWorldGlyphs => _symbolToDefinition.Values.ToList().AsReadOnly();

        public WorldGlyphMap(int worldSeed, int glyphCount)
        {
            GenerateWorldGlyphMap(worldSeed, glyphCount);
            GD.Print($"WorldGlyphMap: Generated {AllWorldGlyphs.Count} glyph definitions for world seed {worldSeed}.");
        }

        /// <summary>
        /// Generates the unique Symbol to Concept mapping for a new world.
        /// (TDD 15.2: Generation Logic)
        /// </summary>
        private void GenerateWorldGlyphMap(int worldSeed, int glyphCount)
        {
            Random rand = new Random(worldSeed);

            // 1. Select a subset of symbols for this world (GDD B01.2)
            List<string> selectedSymbols = GlyphSymbols.SelectWorldSymbols(worldSeed, glyphCount);

            // 2. Get all available concepts (excluding 'None')
            List<GlyphConcept> availableConcepts = Enum.GetValues(typeof(GlyphConcept))
                                                        .Cast<GlyphConcept>()
                                                        .Where(c => c != GlyphConcept.None)
                                                        .ToList();
            
            // Ensure we don't try to map more symbols than concepts available
            if (selectedSymbols.Count > availableConcepts.Count)
            {
                GD.PrintErr($"WorldGlyphMap: Not enough unique concepts ({availableConcepts.Count}) for {selectedSymbols.Count} symbols. Some symbols will share concepts.");
                // For now, we'll allow sharing concepts, or just map fewer symbols.
                // A more robust system might disallow this or add "minor" concepts.
            }

            // Shuffle concepts to ensure random assignment
            for (int i = availableConcepts.Count - 1; i > 0; i--)
            {
                int j = rand.Next(i + 1);
                GlyphConcept temp = availableConcepts[i];
                availableConcepts[i] = availableConcepts[j];
                availableConcepts[j] = temp;
            }

            // 3. Assign each selected symbol to a unique concept (or reuse if concepts run out)
            for (int i = 0; i < selectedSymbols.Count; i++)
            {
                string symbol = selectedSymbols[i];
                GlyphConcept concept = availableConcepts[i % availableConcepts.Count]; // Cycle through concepts if more symbols than unique concepts

                WorldGlyphDefinition def = new WorldGlyphDefinition(symbol, concept);
                _symbolToDefinition.Add(symbol, def);
                // For reverse lookup, store only if concept isn't already mapped (i.e., unique concept mapping)
                if (!_conceptToDefinition.ContainsKey(concept))
                {
                    _conceptToDefinition.Add(concept, def);
                }
            }
        }

        /// <summary>
        /// Retrieves the WorldGlyphDefinition for a given SymbolID.
        /// </summary>
        public WorldGlyphDefinition GetDefinitionBySymbol(string symbolID)
        {
            if (_symbolToDefinition.TryGetValue(symbolID, out WorldGlyphDefinition def))
            {
                return def;
            }
            GD.PrintErr($"WorldGlyphMap: No definition found for symbol '{symbolID}'.");
            return new WorldGlyphDefinition(symbolID, GlyphConcept.None);
        }

        /// <summary>
        /// Retrieves the WorldGlyphDefinition for a given GlyphConcept.
        /// (Useful for AI or systems that know the concept but need the symbol)
        /// </summary>
        public WorldGlyphDefinition GetDefinitionByConcept(GlyphConcept concept)
        {
            if (_conceptToDefinition.TryGetValue(concept, out WorldGlyphDefinition def))
            {
                return def;
            }
            GD.PrintErr($"WorldGlyphMap: No definition found for concept '{concept}'.");
            return new WorldGlyphDefinition("unknown_symbol", GlyphConcept.None);
        }
    }
}
```

### 5. Integrating `WorldGlyphMap` into `GameManager`

Now, let's make `WorldGlyphMap` a core system of our `GameManager`.

1.  Add `using Sigilborne.Systems.Magic;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `WorldGlyphMap` property.
3.  Initialize `WorldGlyphMap` in `InitializeSystems()`. We'll pass a `worldSeed` (e.g., from a future `WorldGenerationSystem`) and a `glyphCount`.

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
using Sigilborne.Systems.Magic; // Add this using directive
using Sigilborne.Utils;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public PhysicsSystem Physics { get; private set; }
    public WorldGlyphMap GlyphMap { get; private set; } // Add WorldGlyphMap property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Events.FlushCommands();
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

        // Initialize WorldGlyphMap (TDD 15.2)
        // For now, use a fixed seed and glyph count. These will come from world generation later.
        int currentWorldSeed = 12345;
        int numGlyphsForWorld = 10; // GDD B01.2: Player typically learns 8-12 early, 20-30 late.
                                   // This is the number available in *this* world.
        GlyphMap = new WorldGlyphMap(currentWorldSeed, numGlyphsForWorld);
        GD.Print("  - WorldGlyphMap initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 6. Testing Glyph Database Generation

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  In the Output console, you should see messages indicating the `WorldGlyphMap` initialization.

```
...
  - PhysicsSystem initialized.
WorldGlyphMap: Generated 10 glyph definitions for world seed 12345.
  - WorldGlyphMap initialized.
  - WorldSimulation initialized.
...
```

You can also use the debug console to inspect the generated map (we'll add a command for this later). For now, you can temporarily add a print loop in `GameManager._Ready()` to see the generated definitions:

```csharp
// _Brain/Core/GameManager.cs (inside _Ready, after GlyphMap initialization)
        GD.Print("\n--- Generated World Glyphs ---");
        foreach (var glyphDef in GlyphMap.AllWorldGlyphs)
        {
            GD.Print(glyphDef.ToString());
        }
        GD.Print("------------------------------\n");
```

Run again, and you'll see a list like:

```
--- Generated World Glyphs ---
Symbol: 'symbol_leaf' -> Concept: Bloom
Symbol: 'symbol_cross' -> Concept: Consume
Symbol: 'symbol_spiral' -> Concept: Pulse
Symbol: 'symbol_pentagon' -> Concept: Veil
Symbol: 'symbol_hand' -> Concept: Chain
Symbol: 'symbol_triangle' -> Concept: Echo
Symbol: 'symbol_sun' -> Concept: Fracture
Symbol: 'symbol_river' -> Concept: Shape
Symbol: 'symbol_dot' -> Concept: Project
Symbol: 'symbol_wave' -> Concept: Bind
------------------------------
```

The specific mappings will vary if you change `currentWorldSeed`. This demonstrates the procedural generation of glyph meanings for each world.

### Summary

You have successfully implemented the foundational **Glyph Database** for Sigilborne, clearly separating `GlyphConcept` (mechanical archetypes) from `GlyphSymbol` (visual hand signs). By designing `WorldGlyphMap` to procedurally generate a unique `Symbol` to `Concept` mapping for each world seed, you've established the core mechanism for dynamic magic, strictly adhering to TDD 15.2's specifications. This crucial step enables infinite replayability and ensures every playthrough feels like a coherent, yet unique, magical language.

### Next Steps

The next chapter will focus on the **Glyph Discovery System**, implementing how players learn these randomized glyph meanings through experimentation, observation, and reading scrolls, and how their `KnowledgeState` progresses over time.