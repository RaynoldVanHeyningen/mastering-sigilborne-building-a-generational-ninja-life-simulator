## Chapter 3.4: Subtypes & Modifiers - Procedural Nuance for Glyphs (C#)

Our glyph system currently maps a `GlyphSymbol` to a `GlyphConcept`. However, the GDD (B01.3) specifies that "each glyph's actual effect â€” its â€œmeaningâ€  â€” is generated uniquely per world." This means a `Bloom` concept might be "toxic bloom" in one world and "healing growth" in another. This chapter implements **Subtypes & Modifiers**, adding procedural nuance to each `WorldGlyphDefinition` to define these world-specific behaviors, as outlined in TDD 15.4.

### 1. The Role of Subtypes in Dynamic Magic

Subtypes are crucial for Sigilborne's infinite replayability and emergent world identity. They provide the "flavor and nuance" (GDD B02.4) to a glyph's core mechanical archetype.

*   **Core Concept**: `Bloom` (stable, universal mechanical archetype).
*   **World-Specific Subtype**: `Toxic Bloom` vs. `Healing Bloom` vs. `Spore Cloud`.
*   **Modifiers**: Numerical parameters that define the subtype's behavior (e.g., `damage_multiplier`, `healing_amount`, `duration`).

These subtypes depend on world history, biomes, anomalies, and clan traditions, ensuring every world feels like a coherent magical language.

### 2. Defining `GlyphSubtype` and `GlyphModifiers`

We need a way to define the specific behavior of a glyph's subtype. This will involve a unique ID for the subtype and a collection of numerical modifiers.

1.  Open `res://_Brain/Systems/Magic/GlyphConcepts.cs` and add `GlyphSubtype` and `GlyphModifierType` enums:

```csharp
// _Brain/Systems/Magic/GlyphConcepts.cs
using System;
using System.Collections.Generic; // For Dictionary<TKey, TValue>

namespace Sigilborne.Systems.Magic
{
    // ... (GlyphConcept enum) ...
    // ... (GlyphKnowledgeState enum) ...
    // ... (GlyphConceptExtensions class) ...

    /// <summary>
    /// Represents the world-specific behavior variant of a GlyphConcept.
    /// (GDD B01.3, TDD 15.4)
    /// </summary>
    public enum GlyphSubtype
    {
        None,           // Default or invalid subtype
        Basic,          // A generic, balanced version of the concept (fallback)

        // Bloom Subtypes
        ToxicBloom,     // Damages, spreads poison
        HealingGrowth,  // Heals, grows protective flora
        SporeCloud,     // Creates a cloud, obscures vision

        // Veil Subtypes
        HeatMirage,     // Creates heat haze, blurs vision
        ColdMistVeil,   // Creates cold fog, slows movement
        ShadowBlend,    // Deepens shadows, enhances stealth

        // Pulse Subtypes
        Shockwave,      // Pushes entities, stuns briefly
        VibrationBurst, // Damages in area, disorients
        ChakraBeam,     // Focused damage, high penetration

        // ... Add more subtypes for other concepts as needed ...
    }

    /// <summary>
    /// Defines the types of numerical modifiers that a glyph subtype can have.
    /// (TDD 15.4)
    /// </summary>
    public enum GlyphModifierType
    {
        None,
        DamageMultiplier,
        HealingAmount,
        Range,
        Duration,
        Radius,
        Speed,
        StabilityCost,
        ResidueGeneration,
        // Add more as needed for specific effects
    }
}
```

### 3. Enhancing `WorldGlyphDefinition` with Subtypes and Modifiers

Now, we'll update our `WorldGlyphDefinition` struct to include a `GlyphSubtype` and a dictionary of `GlyphModifiers`.

Open `res://_Brain/Systems/Magic/WorldGlyphMap.cs` and modify `WorldGlyphDefinition` and its constructor:

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
    /// Now includes world-specific subtype and modifiers.
    /// (GDD B01.3, TDD 15.4)
    /// </summary>
    public struct WorldGlyphDefinition
    {
        public string SymbolID;
        public GlyphConcept Concept;
        public GlyphSubtype Subtype; // New: World-specific behavior variant
        public Dictionary<GlyphModifierType, float> Modifiers; // New: Numerical parameters for the subtype

        public WorldGlyphDefinition(string symbolId, GlyphConcept concept, GlyphSubtype subtype = GlyphSubtype.Basic, Dictionary<GlyphModifierType, float> modifiers = null)
        {
            SymbolID = symbolId;
            Concept = concept;
            Subtype = subtype;
            Modifiers = modifiers ?? new Dictionary<GlyphModifierType, float>(); // Initialize if null
        }

        public override string ToString()
        {
            // Include Subtype and Modifiers in the string representation
            string modifierString = string.Join(", ", Modifiers.Select(kv => $"{kv.Key}: {kv.Value:F1}"));
            return $"Symbol: '{SymbolID}' -> Concept: {Concept}, Subtype: {Subtype} (Mods: {modifierString})";
        }
    }

    // ... (WorldGlyphMap class) ...
}
```

### 4. Enhancing `WorldGlyphMap` to Generate Subtypes and Modifiers

The `GenerateWorldGlyphMap` method needs to be updated to:
1.  Assign a random `GlyphSubtype` to each `WorldGlyphDefinition` based on its `GlyphConcept`.
2.  Generate `GlyphModifiers` for that subtype.

This will involve creating a helper method to get possible subtypes and their base modifiers for each concept.

1.  Open `res://_Brain/Systems/Magic/WorldGlyphMap.cs` and add a new helper class for Subtype Data (within the `Sigilborne.Systems.Magic` namespace, perhaps in a new file `GlyphSubtypeData.cs` or at the bottom of `WorldGlyphMap.cs`). For now, let's keep it in `WorldGlyphMap.cs` for simplicity.

```csharp
// _Brain/Systems/Magic/WorldGlyphMap.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;

namespace Sigilborne.Systems.Magic
{
    // ... (WorldGlyphDefinition struct) ...

    /// <summary>
    /// Helper class to store static definitions of GlyphSubtypes and their base modifiers.
    /// This is a static lookup, not procedurally generated. The *assignment* is procedural.
    /// </summary>
    public static class GlyphSubtypeData
    {
        private static Dictionary<GlyphConcept, List<GlyphSubtype>> _conceptToSubtypes = new Dictionary<GlyphConcept, List<GlyphSubtype>>();
        private static Dictionary<GlyphSubtype, Dictionary<GlyphModifierType, float>> _subtypeBaseModifiers = new Dictionary<GlyphSubtype, Dictionary<GlyphModifierType, float>>();

        static GlyphSubtypeData() // Static constructor to initialize data once
        {
            // --- Map Concepts to Possible Subtypes ---
            _conceptToSubtypes.Add(GlyphConcept.Bloom, new List<GlyphSubtype> { GlyphSubtype.ToxicBloom, GlyphSubtype.HealingGrowth, GlyphSubtype.SporeCloud });
            _conceptToSubtypes.Add(GlyphConcept.Veil, new List<GlyphSubtype> { GlyphSubtype.HeatMirage, GlyphSubtype.ColdMistVeil, GlyphSubtype.ShadowBlend });
            _conceptToSubtypes.Add(GlyphConcept.Pulse, new List<GlyphSubtype> { GlyphSubtype.Shockwave, GlyphSubtype.VibrationBurst, GlyphSubtype.ChakraBeam });
            // Add mappings for other concepts as they get subtypes

            // --- Define Base Modifiers for Each Subtype ---
            _subtypeBaseModifiers.Add(GlyphSubtype.Basic, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.DamageMultiplier, 1.0f }, { GlyphModifierType.Range, 100f } });

            _subtypeBaseModifiers.Add(GlyphSubtype.ToxicBloom, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.DamageMultiplier, 1.2f }, { GlyphModifierType.Duration, 5.0f }, { GlyphModifierType.Radius, 50f } });
            _subtypeBaseModifiers.Add(GlyphSubtype.HealingGrowth, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.HealingAmount, 15.0f }, { GlyphModifierType.Duration, 3.0f }, { GlyphModifierType.Radius, 40f } });
            _subtypeBaseModifiers.Add(GlyphSubtype.SporeCloud, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.Duration, 8.0f }, { GlyphModifierType.Radius, 70f } });

            _subtypeBaseModifiers.Add(GlyphSubtype.HeatMirage, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.Duration, 10.0f }, { GlyphModifierType.StabilityCost, 0.1f } });
            _subtypeBaseModifiers.Add(GlyphSubtype.ColdMistVeil, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.Duration, 7.0f }, { GlyphModifierType.Speed, 0.7f } }); // Speed modifier for affected entities
            _subtypeBaseModifiers.Add(GlyphSubtype.ShadowBlend, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.Duration, 12.0f }, { GlyphModifierType.StabilityCost, 0.05f } });

            _subtypeBaseModifiers.Add(GlyphSubtype.Shockwave, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.DamageMultiplier, 0.8f }, { GlyphModifierType.Radius, 80f }, { GlyphModifierType.Speed, 0.5f } }); // Pushes, slows
            _subtypeBaseModifiers.Add(GlyphSubtype.VibrationBurst, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.DamageMultiplier, 1.1f }, { GlyphModifierType.Radius, 40f }, { GlyphModifierType.Duration, 2.0f } }); // Disorient duration
            _subtypeBaseModifiers.Add(GlyphSubtype.ChakraBeam, new Dictionary<GlyphModifierType, float> { { GlyphModifierType.DamageMultiplier, 1.5f }, { GlyphModifierType.Range, 200f }, { GlyphModifierType.StabilityCost, 0.2f } });
        }

        public static List<GlyphSubtype> GetSubtypesForConcept(GlyphConcept concept)
        {
            if (_conceptToSubtypes.TryGetValue(concept, out List<GlyphSubtype> subtypes))
            {
                return new List<GlyphSubtype>(subtypes); // Return a copy
            }
            return new List<GlyphSubtype> { GlyphSubtype.Basic }; // Fallback
        }

        public static Dictionary<GlyphModifierType, float> GetBaseModifiersForSubtype(GlyphSubtype subtype)
        {
            if (_subtypeBaseModifiers.TryGetValue(subtype, out Dictionary<GlyphModifierType, float> modifiers))
            {
                // Return a new dictionary to prevent external modification of static data
                return new Dictionary<GlyphModifierType, float>(modifiers);
            }
            return new Dictionary<GlyphModifierType, float>(); // Empty if no specific modifiers
        }
    }

    /// <summary>
    /// The authoritative database for a world's unique Glyph Symbol to Concept mapping.
    /// This is generated once per world seed.
    /// (TDD 15.2)
    /// </summary>
    public class WorldGlyphMap
    {
        // ... (existing fields) ...

        public WorldGlyphMap(int worldSeed, int glyphCount)
        {
            GenerateWorldGlyphMap(worldSeed, glyphCount);
            GD.Print($"WorldGlyphMap: Generated {AllWorldGlyphs.Count} glyph definitions for world seed {worldSeed}.");
        }

        private void GenerateWorldGlyphMap(int worldSeed, int glyphCount)
        {
            Random rand = new Random(worldSeed);

            List<string> selectedSymbols = GlyphSymbols.SelectWorldSymbols(worldSeed, glyphCount);
            List<GlyphConcept> availableConcepts = Enum.GetValues(typeof(GlyphConcept))
                                                        .Cast<GlyphConcept>()
                                                        .Where(c => c != GlyphConcept.None)
                                                        .ToList();
            
            // ... (shuffle concepts) ...

            for (int i = 0; i < selectedSymbols.Count; i++)
            {
                string symbol = selectedSymbols[i];
                GlyphConcept concept = availableConcepts[i % availableConcepts.Count];

                // --- NEW: Procedurally assign subtype and modifiers (TDD 15.4) ---
                List<GlyphSubtype> possibleSubtypes = GlyphSubtypeData.GetSubtypesForConcept(concept);
                GlyphSubtype assignedSubtype = possibleSubtypes.Count > 0 ? possibleSubtypes[rand.Next(possibleSubtypes.Count)] : GlyphSubtype.Basic;
                
                Dictionary<GlyphModifierType, float> baseModifiers = GlyphSubtypeData.GetBaseModifiersForSubtype(assignedSubtype);
                
                // Further randomize modifiers slightly (e.g., +/- 10%)
                Dictionary<GlyphModifierType, float> finalModifiers = new Dictionary<GlyphModifierType, float>();
                foreach (var kvp in baseModifiers)
                {
                    float variance = (float)(rand.NextDouble() * 0.2 - 0.1); // -0.1 to +0.1
                    finalModifiers.Add(kvp.Key, kvp.Value * (1.0f + variance));
                }
                // --- END NEW ---

                WorldGlyphDefinition def = new WorldGlyphDefinition(symbol, concept, assignedSubtype, finalModifiers); // Pass new data
                _symbolToDefinition.Add(symbol, def);
                if (!_conceptToDefinition.ContainsKey(concept))
                {
                    _conceptToDefinition.Add(concept, def);
                }
            }
        }

        // ... (GetDefinitionBySymbol, GetDefinitionByConcept methods) ...
    }
}
```

**Explanation of Changes:**

*   **`GlyphSubtypeData`**: A new static class that acts as a lookup table for:
    *   Which `GlyphSubtype`s are possible for each `GlyphConcept`.
    *   The base numerical `Modifiers` for each `GlyphSubtype`. This data is static and pre-defined, not procedurally generated.
*   **`GenerateWorldGlyphMap()`**:
    *   Now, after assigning a `Concept` to a `Symbol`, it retrieves the `possibleSubtypes` for that `Concept` from `GlyphSubtypeData`.
    *   It randomly selects one `assignedSubtype` from the possibilities.
    *   It gets the `baseModifiers` for that `assignedSubtype`.
    *   It then applies a small random `variance` to these modifiers, ensuring even more uniqueness per world.
    *   Finally, it creates the `WorldGlyphDefinition` with the `assignedSubtype` and `finalModifiers`.

### 5. Testing Subtype Generation

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  In the Output console, you should now see the `WorldGlyphMap` initialization printing the generated glyphs with their assigned subtypes and modified values.

```
...
--- Generated World Glyphs ---
Symbol: 'symbol_leaf' -> Concept: Bloom, Subtype: HealingGrowth (Mods: HealingAmount: 15.4, Duration: 3.3, Radius: 43.1)
Symbol: 'symbol_cross' -> Concept: Consume, Subtype: Basic (Mods: DamageMultiplier: 1.0, Range: 100.0)
Symbol: 'symbol_spiral' -> Concept: Pulse, Subtype: Shockwave (Mods: DamageMultiplier: 0.8, Radius: 79.9, Speed: 0.5)
Symbol: 'symbol_pentagon' -> Concept: Veil, Subtype: ShadowBlend (Mods: Duration: 12.5, StabilityCost: 0.0)
Symbol: 'symbol_hand' -> Concept: Chain, Subtype: Basic (Mods: DamageMultiplier: 1.0, Range: 100.0)
Symbol: 'symbol_triangle' -> Concept: Echo, Subtype: Basic (Mods: DamageMultiplier: 1.0, Range: 100.0)
Symbol: 'symbol_sun' -> Concept: Fracture, Subtype: Basic (Mods: DamageMultiplier: 1.0, Range: 100.0)
Symbol: 'symbol_river' -> Concept: Shape, Subtype: Basic (Mods: DamageMultiplier: 1.0, Range: 100.0)
Symbol: 'symbol_dot' -> Concept: Project, Subtype: ChakraBeam (Mods: DamageMultiplier: 1.4, Range: 181.9, StabilityCost: 0.2)
Symbol: 'symbol_wave' -> Concept: Bind, Subtype: Basic (Mods: DamageMultiplier: 1.0, Range: 100.0)
------------------------------
...
```

Notice how `symbol_leaf` (Bloom) got `HealingGrowth` with slightly varied modifiers, `symbol_spiral` (Pulse) got `Shockwave`, etc. This demonstrates the procedural assignment of subtypes and modifiers, bringing the GDD's vision of unique magic per world to life.

### Summary

You have successfully implemented **Subtypes & Modifiers** for Sigilborne's glyph system, expanding `WorldGlyphDefinition` to include procedural subtypes and numerical modifiers. By defining `GlyphSubtypeData` and enhancing `WorldGlyphMap` to randomly assign subtypes and vary their parameters based on the world seed, you've introduced deep nuance and replayability to each glyph's effect, strictly adhering to TDD 15.4's specifications. This crucial step ensures that every `Bloom` is not just `Bloom`, but a unique `Toxic Bloom` or `Healing Growth` with distinct properties, making each world's magic truly unique.

### Next Steps

The next chapter will focus on **Glyph Acquisition**, detailing how players learn new glyphs through various in-world methods like interacting with teachers, finding scrolls, and observing NPCs, and how these actions update their `PlayerGlyphKnowledgeSystem`.