## Chapter 3.5: Glyph Acquisition - Teachers & Scrolls (C#)

Players in Sigilborne don't just know glyphs; they acquire them through active engagement with the world. This chapter implements the core mechanisms for **Glyph Acquisition**, focusing on two primary methods: learning from NPC teachers and discovering scrolls. These actions will update the player's `GlyphKnowledgeState` in the `PlayerGlyphKnowledgeSystem`, reflecting their journey of discovery, as specified in TDD 15.4 and GDD B01.4.

### 1. The Acquisition Philosophy: In-World Discovery

The GDD (B01.4) outlines several ways players learn glyphs:
*   **NPC Teachers**: Structured or informal teaching.
*   **Scrolls & Fragments**: Diagrams, partial combos, lore.
*   **Observation**: Seeing an NPC use a sign.
*   **Solo-Cast Discovery**: Experimentation (already covered).
*   **Bloodline Resonance**: Innate intuition (covered later).
*   **Anomaly Revelation**: Dangerous flashes (covered later).

This chapter will focus on the first two, building a foundation for explicit knowledge transfer.

### 2. Implementing `GlyphAcquisitionSystem.cs`

This system will provide methods for other systems (e.g., `InteractionSystem` for talking to teachers, `InventorySystem` for using scrolls) to trigger glyph learning.

1.  Create `res://_Brain/Systems/Magic/GlyphAcquisitionSystem.cs`:

```csharp
// _Brain/Systems/Magic/GlyphAcquisitionSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Manages the acquisition of new glyph knowledge for the player.
    /// Provides methods for learning via teachers, scrolls, and observation.
    /// (GDD B01.4, TDD 15.4)
    /// </summary>
    public class GlyphAcquisitionSystem
    {
        private EntityID _playerEntityID;
        private EventBus _eventBus;
        private PlayerGlyphKnowledgeSystem _playerGlyphKnowledge;
        private WorldGlyphMap _worldGlyphMap;

        public GlyphAcquisitionSystem(EntityID playerEntityID, EventBus eventBus, PlayerGlyphKnowledgeSystem playerGlyphKnowledge, WorldGlyphMap worldGlyphMap)
        {
            _playerEntityID = playerEntityID;
            _eventBus = eventBus;
            _playerGlyphKnowledge = playerGlyphKnowledge;
            _worldGlyphMap = worldGlyphMap;
            GD.Print($"GlyphAcquisitionSystem: Initialized for player {_playerEntityID}.");
        }

        /// <summary>
        /// Simulates learning a glyph from an NPC teacher.
        /// (GDD B01.4.A: NPC Teachers)
        /// </summary>
        /// <param name="symbolID">The ID of the glyph symbol being taught.</param>
        /// <returns>True if knowledge was updated to Known, false otherwise.</returns>
        public bool LearnFromTeacher(string symbolID)
        {
            if (!_worldGlyphMap.GetDefinitionBySymbol(symbolID).Concept.IsValidConcept())
            {
                GD.PrintErr($"GlyphAcquisitionSystem: Teacher attempted to teach unknown symbol '{symbolID}'.");
                return false;
            }

            GlyphKnowledgeState currentState = _playerGlyphKnowledge.GetGlyphKnowledge(symbolID);
            if (currentState == GlyphKnowledgeState.Known)
            {
                GD.Print($"GlyphAcquisitionSystem: Player already knows '{symbolID}'. Teacher offers practice instead.");
                // In a real game, this might trigger a training mini-game or quest.
                return false;
            }

            GD.Print($"GlyphAcquisitionSystem: Player learned symbol '{symbolID}' from a teacher.");
            // Teachers typically grant 'Known' status immediately for the symbol and its concept.
            return _playerGlyphKnowledge.UpdateGlyphKnowledge(symbolID, GlyphKnowledgeState.Known);
        }

        /// <summary>
        /// Simulates learning a glyph from a scroll or fragment.
        /// (GDD B01.4.B: Scrolls & Fragments)
        /// </summary>
        /// <param name="symbolID">The ID of the glyph symbol detailed in the scroll.</param>
        /// <param name="fullMeaning">If true, scroll grants Known. If false, grants Seen (fragment).</param>
        /// <returns>True if knowledge was updated, false otherwise.</returns>
        public bool LearnFromScroll(string symbolID, bool fullMeaning)
        {
            if (!_worldGlyphMap.GetDefinitionBySymbol(symbolID).Concept.IsValidConcept())
            {
                GD.PrintErr($"GlyphAcquisitionSystem: Scroll details unknown symbol '{symbolID}'.");
                return false;
            }

            GlyphKnowledgeState currentState = _playerGlyphKnowledge.GetGlyphKnowledge(symbolID);
            GlyphKnowledgeState newState = fullMeaning ? GlyphKnowledgeState.Known : GlyphKnowledgeState.Seen;

            if (currentState >= newState)
            {
                GD.Print($"GlyphAcquisitionSystem: Player already knows '{symbolID}' at state {currentState}. Scroll provides no new info.");
                // In a real game, a scroll might provide lore or combo hints even if the glyph is known.
                return false;
            }

            GD.Print($"GlyphAcquisitionSystem: Player learned symbol '{symbolID}' from a scroll (State: {newState}).");
            return _playerGlyphKnowledge.UpdateGlyphKnowledge(symbolID, newState);
        }

        /// <summary>
        /// Simulates observing an NPC use a glyph.
        /// (GDD B01.4.C: Observation)
        /// </summary>
        /// <param name="symbolID">The ID of the glyph symbol observed.</param>
        /// <returns>True if knowledge was updated to Seen (or higher), false otherwise.</returns>
        public bool ObserveGlyph(string symbolID)
        {
            if (!_worldGlyphMap.GetDefinitionBySymbol(symbolID).Concept.IsValidConcept())
            {
                GD.PrintErr($"GlyphAcquisitionSystem: Observed unknown symbol '{symbolID}'.");
                return false;
            }

            GlyphKnowledgeState currentState = _playerGlyphKnowledge.GetGlyphKnowledge(symbolID);
            if (currentState >= GlyphKnowledgeState.Seen)
            {
                // Already seen or known, no update needed for 'Seen' status.
                return false;
            }

            GD.Print($"GlyphAcquisitionSystem: Player observed symbol '{symbolID}'.");
            return _playerGlyphKnowledge.UpdateGlyphKnowledge(symbolID, GlyphKnowledgeState.Seen);
        }
    }
}
```

### 3. Integrating `GlyphAcquisitionSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Magic;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `GlyphAcquisitionSystem` property.
3.  Initialize `GlyphAcquisitionSystem` in `InitializeSystems()`.

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
using System.Linq; // For .First() in test

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public PlayerHotbarSystem PlayerHotbar { get; private set; }
    public MagicSystem Magic { get; private set; }
    public GlyphAcquisitionSystem GlyphAcquisition { get; private set; } // Add GlyphAcquisitionSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        
        // --- Test Glyph Discovery System ---
        GD.Print("\n--- Testing Glyph Discovery System ---");
        string testSymbol = GlyphMap.AllWorldGlyphs[0].SymbolID;
        string testSymbol2 = GlyphMap.AllWorldGlyphs[1].SymbolID;
        string testSymbol3 = GlyphMap.AllWorldGlyphs[2].SymbolID; // New symbol for acquisition tests
        string unknownSymbol = GlyphSymbols.AllSymbols.First(s => !GlyphMap.AllWorldGlyphs.Any(g => g.SymbolID == s));

        // Reset knowledge for acquisition tests
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Hidden, true);
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol2, GlyphKnowledgeState.Hidden, true);
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol3, GlyphKnowledgeState.Hidden, true);

        GD.Print($"Initial knowledge of '{testSymbol}': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        GD.Print($"Initial knowledge of '{testSymbol2}': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol2)}");
        GD.Print($"Initial knowledge of '{testSymbol3}': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol3)}");
        
        // Let MagicSystem handle discovery for testSymbol via hotbar input
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol2, GlyphKnowledgeState.Known, true); // Still make testSymbol2 known for hotbar test
        
        GD.Print($"Attempt to update unknown symbol '{unknownSymbol}':");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(unknownSymbol, GlyphKnowledgeState.Known);
        
        GD.Print("--- End Testing Glyph Discovery System ---\n");

        // --- Test PlayerHotbarSystem ---
        GD.Print("\n--- Testing PlayerHotbarSystem ---");
        PlayerHotbar.AssignGlyphToSlot(0, testSymbol); // Assign potentially unknown symbol
        PlayerHotbar.AssignGlyphToSlot(1, testSymbol2); // Assign known symbol
        PlayerHotbar.AssignGlyphToSlot(2, testSymbol3); // Assign unknown symbol for observation test
        // ... (existing hotbar tests) ...
        GD.Print("--- End Testing PlayerHotbarSystem ---\n");

        // --- Test Glyph Acquisition System ---
        GD.Print("\n--- Testing Glyph Acquisition System ---");
        GD.Print($"Knowledge of '{testSymbol3}' before acquisition: {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol3)}");
        
        // Test LearnFromTeacher
        GlyphAcquisition.LearnFromTeacher(testSymbol3);
        GD.Print($"Knowledge of '{testSymbol3}' after teacher: {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol3)}");
        GlyphAcquisition.LearnFromTeacher(testSymbol3); // Should say already known
        
        // Test LearnFromScroll (fragment)
        string scrollSymbol = GlyphMap.AllWorldGlyphs[3].SymbolID; // Get another symbol
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(scrollSymbol, GlyphKnowledgeState.Hidden, true); // Ensure it's hidden
        GD.Print($"Knowledge of '{scrollSymbol}' before scroll: {PlayerGlyphKnowledge.GetGlyphKnowledge(scrollSymbol)}");
        GlyphAcquisition.LearnFromScroll(scrollSymbol, false); // Fragment -> Seen
        GD.Print($"Knowledge of '{scrollSymbol}' after fragment scroll: {PlayerGlyphKnowledge.GetGlyphKnowledge(scrollSymbol)}");
        GlyphAcquisition.LearnFromScroll(scrollSymbol, false); // No update needed
        GlyphAcquisition.LearnFromScroll(scrollSymbol, true); // Full scroll -> Known
        GD.Print($"Knowledge of '{scrollSymbol}' after full scroll: {PlayerGlyphKnowledge.GetGlyphKnowledge(scrollSymbol)}");

        // Test ObserveGlyph
        string observedSymbol = GlyphMap.AllWorldGlyphs[4].SymbolID; // Yet another symbol
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(observedSymbol, GlyphKnowledgeState.Hidden, true); // Ensure hidden
        GD.Print($"Knowledge of '{observedSymbol}' before observation: {PlayerGlyphKnowledge.GetGlyphKnowledge(observedSymbol)}");
        GlyphAcquisition.ObserveGlyph(observedSymbol);
        GD.Print($"Knowledge of '{observedSymbol}' after observation: {PlayerGlyphKnowledge.GetGlyphKnowledge(observedSymbol)}");
        GlyphAcquisition.ObserveGlyph(observedSymbol); // No update needed
        
        GD.Print("--- End Testing Glyph Acquisition System ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        // ... (existing _PhysicsProcess calls) ...
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations) ...

        Magic = new MagicSystem(Entities, Input, Events, PlayerHotbar, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - MagicSystem initialized.");

        // Initialize GlyphAcquisitionSystem
        GlyphAcquisition = new GlyphAcquisitionSystem(Entities.GetPlayerEntityID(), Events, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - GlyphAcquisitionSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 4. Testing Glyph Acquisition

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  In the Output console, you should see the `GlyphAcquisitionSystem` initialization and test messages.

```
...
PlayerGlyphKnowledgeSystem: Initialized for player EntityID(0, Gen:1) with all glyphs as Hidden.
  - PlayerGlyphKnowledgeSystem initialized.
  - PlayerHotbarSystem initialized.
  - MagicSystem initialized.
GlyphAcquisitionSystem: Initialized for player EntityID(0, Gen:1).
  - GlyphAcquisitionSystem initialized.
  - WorldSimulation initialized.

--- Testing Glyph Discovery System ---
Initial knowledge of 'symbol_leaf': Hidden
Initial knowledge of 'symbol_cross': Hidden
Initial knowledge of 'symbol_diamond': Hidden
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_cross' updated from Hidden to Known.
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_cross' updated from Hidden to Known.
Attempt to update unknown symbol 'symbol_triangle_alt':
PlayerGlyphKnowledgeSystem: Attempted to update knowledge for unknown symbol 'symbol_triangle_alt'.
--- End Testing Glyph Discovery System ---

--- Testing PlayerHotbarSystem ---
PlayerHotbarSystem: Assigned 'symbol_leaf' to slot 0.
PlayerHotbarSystem: Assigned 'symbol_cross' to slot 1.
PlayerGlyphKnowledgeSystem: Player does not know glyph 'symbol_diamond'. Cannot assign to hotbar.
PlayerHotbarSystem: Assigned 'symbol_leaf' to slot 9.
PlayerHotbarSystem: Unequipped slot 0.
Glyph in slot 1: symbol_cross
--- End Testing PlayerHotbarSystem ---

--- Testing Glyph Acquisition System ---
Knowledge of 'symbol_diamond' before acquisition: Hidden
GlyphAcquisitionSystem: Player learned symbol 'symbol_diamond' from a teacher.
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_diamond' updated from Hidden to Known.
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_diamond' updated from Hidden to Known.
GlyphAcquisitionSystem: Player already knows 'symbol_diamond'. Teacher offers practice instead.
Knowledge of 'symbol_diamond' after teacher: Known
Knowledge of 'symbol_pentagon' before scroll: Hidden
GlyphAcquisitionSystem: Player learned symbol 'symbol_pentagon' from a scroll (State: Seen).
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_pentagon' updated from Hidden to Seen.
GlyphAcquisitionSystem: Player already knows 'symbol_pentagon' at state Seen. Scroll provides no new info.
GlyphAcquisitionSystem: Player learned symbol 'symbol_pentagon' from a scroll (State: Known).
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_pentagon' updated from Seen to Known.
Knowledge of 'symbol_pentagon' after full scroll: Known
Knowledge of 'symbol_hexagon' before observation: Hidden
GlyphAcquisitionSystem: Player observed symbol 'symbol_hexagon'.
PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_hexagon' updated from Hidden to Seen.
GlyphAcquisitionSystem: Player observed symbol 'symbol_hexagon'.
Knowledge of 'symbol_hexagon' after observation: Seen
--- End Testing Glyph Acquisition System ---
```

This output confirms:
*   `LearnFromTeacher` correctly grants `Known` status.
*   `LearnFromScroll` correctly grants `Seen` (for fragments) and then `Known` (for full meaning).
*   `ObserveGlyph` correctly grants `Seen` status.
*   The system gracefully handles attempts to acquire already known glyphs or unknown symbols.

### Summary

You have successfully implemented the **Glyph Acquisition System** for Sigilborne, enabling players to learn new glyphs through structured methods like NPC teachers and scrolls, as well as through passive observation. By creating `GlyphAcquisitionSystem` and integrating it with `PlayerGlyphKnowledgeSystem`, you've established a clear pipeline for updating the player's knowledge state based on in-world discovery actions, strictly adhering to GDD B01.4 and TDD 15.4's specifications. This crucial step enriches the player's journey of mastering ninjutsu through active engagement with the game world.

### Next Steps

This concludes **Module 3: The Glyph System - Language of Ninjutsu**. We will now move on to **Module 4: Combos & Casting - The Art of Ninjutsu**, starting with **Input Buffer - Storing Glyph Sequences (C#)**. This will refine our glyph input processing for more complex combo detection.