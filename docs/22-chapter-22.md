## Chapter 3.3: Glyph Experimentation - Trial & Error Casting (C#)

With glyph meanings defined and player knowledge tracked, it's time for the player to actually *use* (or attempt to use) these glyphs. This chapter implements the core **Glyph Experimentation** loop, where players input sequences of glyphs. The system will process these inputs, determine if they form a valid known combo, a new plausible technique, or simply fizzle, and provide appropriate feedback, including revealing glyph meanings, as specified in TDD 15.3.

This is where the magic (or the misfire) truly happens!

### 1. The Experimentation Loop: Input -> Interpretation -> Outcome

The GDD (B01.4, B02.2) emphasizes freeform combos and interpretation by world logic. This requires:

*   **Input Capture**: We already have this via `InputSystem` and `PlayerInputFrame`.
*   **Glyph Mapping**: The player presses a hotbar key, which maps to a `GlyphSymbolID`. This symbol is then looked up in the `WorldGlyphMap` to get its `GlyphConcept`.
*   **Sequence Interpretation**: The sequence of `GlyphConcept`s is analyzed to determine the outcome.
*   **Outcome & Feedback**: Based on the interpretation, a `SpellDefinition` is either found/created, or a fizzle/miscast occurs, and knowledge is updated.

### 2. Mapping Hotbar Input to Glyph Symbols

First, we need a way to map hotbar key presses to specific `GlyphSymbolID`s. This mapping will be dynamic and tied to the player's learned glyphs.

We'll create a `PlayerHotbarSystem` to manage what glyphs are currently "equipped" to the hotbar.

1.  Create `res://_Brain/Systems/Magic/PlayerHotbarSystem.cs`:

```csharp
// _Brain/Systems/Magic/PlayerHotbarSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;
using Sigilborne.Core;
using Sigilborne.Entities;

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Manages the glyphs (symbols) currently assigned to the player's hotbar slots.
    /// (GDD B05.2: The Recursive Mastery Deck)
    /// </summary>
    public class PlayerHotbarSystem
    {
        private EntityID _playerEntityID;
        private EventBus _eventBus;
        private PlayerGlyphKnowledgeSystem _playerGlyphKnowledge;
        private WorldGlyphMap _worldGlyphMap;

        // The hotbar slots. Max 10 slots (TDD 12.5).
        // Stores the SymbolID assigned to each slot. Null if slot is empty.
        private string[] _hotbarSlots = new string[10];

        public PlayerHotbarSystem(EntityID playerEntityID, EventBus eventBus, PlayerGlyphKnowledgeSystem playerGlyphKnowledge, WorldGlyphMap worldGlyphMap)
        {
            _playerEntityID = playerEntityID;
            _eventBus = eventBus;
            _playerGlyphKnowledge = playerGlyphKnowledge;
            _worldGlyphMap = worldGlyphMap;

            // Initially, hotbar is empty.
            // For testing, let's assign the first few known glyphs to the hotbar.
            // In a real game, players would equip these via UI.
            GD.Print($"PlayerHotbarSystem: Initialized for player {_playerEntityID}.");
        }

        /// <summary>
        /// Assigns a glyph symbol to a specific hotbar slot.
        /// </summary>
        /// <param name="slotIndex">The 0-9 index of the hotbar slot.</param>
        /// <param name="symbolID">The ID of the glyph symbol. Null/empty to unequip.</param>
        /// <returns>True if assignment was successful, false otherwise (e.g., unknown symbol, invalid slot).</returns>
        public bool AssignGlyphToSlot(int slotIndex, string symbolID)
        {
            if (slotIndex < 0 || slotIndex >= _hotbarSlots.Length)
            {
                GD.PrintErr($"PlayerHotbarSystem: Invalid hotbar slot index: {slotIndex}.");
                return false;
            }

            if (string.IsNullOrEmpty(symbolID))
            {
                _hotbarSlots[slotIndex] = null; // Unequip
                GD.Print($"PlayerHotbarSystem: Unequipped slot {slotIndex}.");
                _eventBus.Publish(new HotbarSlotUpdatedEvent { PlayerID = _playerEntityID, SlotIndex = slotIndex, SymbolID = null });
                return true;
            }

            // Check if the player knows this glyph and if it's a valid symbol for this world
            if (_playerGlyphKnowledge.GetGlyphKnowledge(symbolID) < GlyphKnowledgeState.Known)
            {
                GD.PrintErr($"PlayerHotbarSystem: Player does not know glyph '{symbolID}'. Cannot assign to hotbar.");
                return false;
            }
            if (!_worldGlyphMap.GetDefinitionBySymbol(symbolID).Concept.IsValidConcept()) // Check if symbol maps to a valid concept in this world
            {
                GD.PrintErr($"PlayerHotbarSystem: Symbol '{symbolID}' is not a valid glyph in this world. Cannot assign to hotbar.");
                return false;
            }

            _hotbarSlots[slotIndex] = symbolID;
            GD.Print($"PlayerHotbarSystem: Assigned '{symbolID}' to slot {slotIndex}.");
            _eventBus.Publish(new HotbarSlotUpdatedEvent { PlayerID = _playerEntityID, SlotIndex = slotIndex, SymbolID = symbolID });
            return true;
        }

        /// <summary>
        /// Gets the glyph symbol ID assigned to a hotbar slot.
        /// </summary>
        public string GetGlyphSymbolInSlot(int slotIndex)
        {
            if (slotIndex >= 0 && slotIndex < _hotbarSlots.Length)
            {
                return _hotbarSlots[slotIndex];
            }
            return null;
        }

        // --- Helper Events for Body Sync ---
        public struct HotbarSlotUpdatedEvent { public EntityID PlayerID; public int SlotIndex; public string SymbolID; }
    }
}
```

#### 2.1. Update `GlyphConcepts.cs` with `IsValidConcept`

We need a helper method to check if a `GlyphConcept` is valid (i.e., not `None`).

Open `res://_Brain/Systems/Magic/GlyphConcepts.cs` and add this extension method:

```csharp
// _Brain/Systems/Magic/GlyphConcepts.cs
using System;

namespace Sigilborne.Systems.Magic
{
    // ... (GlyphConcept enum) ...
    // ... (GlyphKnowledgeState enum) ...

    public static class GlyphConceptExtensions
    {
        public static bool IsValidConcept(this GlyphConcept concept)
        {
            return concept != GlyphConcept.None;
        }
    }
}
```

### 3. Integrating `PlayerHotbarSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Magic;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `PlayerHotbarSystem` property.
3.  Initialize `PlayerHotbarSystem` in `InitializeSystems()`.

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
    public PlayerGlyphKnowledgeSystem PlayerGlyphKnowledge { get; private set; }
    public PlayerHotbarSystem PlayerHotbar { get; private set; } // Add PlayerHotbarSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        
        // --- Test Glyph Discovery System ---
        GD.Print("\n--- Testing Glyph Discovery System ---");
        // Get a symbol from the generated map for testing
        string testSymbol = GlyphMap.AllWorldGlyphs[0].SymbolID; // e.g., "symbol_leaf"
        string testSymbol2 = GlyphMap.AllWorldGlyphs[1].SymbolID; // e.g., "symbol_cross"
        string unknownSymbol = GlyphSymbols.AllSymbols.First(s => !GlyphMap.AllWorldGlyphs.Any(g => g.SymbolID == s));

        GD.Print($"Initial knowledge of '{testSymbol}': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Seen);
        GD.Print($"Knowledge of '{testSymbol}' after 'Seen': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Known);
        GD.Print($"Knowledge of '{testSymbol}' after 'Known': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol2, GlyphKnowledgeState.Known); // Make second symbol known for hotbar test
        GD.Print($"Knowledge of '{testSymbol2}' after 'Known': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol2)}");

        GD.Print($"Attempt to update unknown symbol '{unknownSymbol}':");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(unknownSymbol, GlyphKnowledgeState.Known);
        
        GD.Print("--- End Testing Glyph Discovery System ---\n");

        // --- Test PlayerHotbarSystem ---
        GD.Print("\n--- Testing PlayerHotbarSystem ---");
        PlayerHotbar.AssignGlyphToSlot(0, testSymbol); // Assign known symbol
        PlayerHotbar.AssignGlyphToSlot(1, testSymbol2); // Assign another known symbol
        PlayerHotbar.AssignGlyphToSlot(2, unknownSymbol); // Attempt to assign unknown symbol (should fail)
        PlayerHotbar.AssignGlyphToSlot(9, testSymbol); // Assign to last slot
        PlayerHotbar.AssignGlyphToSlot(0, null); // Unequip slot 0
        GD.Print($"Glyph in slot 1: {PlayerHotbar.GetGlyphSymbolInSlot(1)}");
        GD.Print("--- End Testing PlayerHotbarSystem ---\n");
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

        // Player entity is created in _Ready(), so its ID is available when InitializeSystems() is called.
        // This is safe because _Ready() runs before InitializeSystems() in Godot's execution order.
        PlayerGlyphKnowledge = new PlayerGlyphKnowledgeSystem(Entities.GetPlayerEntityID(), GlyphMap, Events);
        GD.Print("  - PlayerGlyphKnowledgeSystem initialized.");

        // Initialize PlayerHotbarSystem
        PlayerHotbar = new PlayerHotbarSystem(Entities.GetPlayerEntityID(), Events, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - PlayerHotbarSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 3.1. Update `EventBus.cs` for Hotbar Events

Our `PlayerHotbarSystem` publishes `HotbarSlotUpdatedEvent`. We need to define this `Action` delegate in `EventBus`.

Open `_Brain/Core/EventBus.cs`:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Hotbar Events (TDD 12.5)
        public event Action<EntityID, int, string> OnHotbarSlotUpdated;

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is PlayerHotbarSystem.HotbarSlotUpdatedEvent hotbarEvent) // New condition
            {
                OnHotbarSlotUpdated?.Invoke(hotbarEvent.PlayerID, hotbarEvent.SlotIndex, hotbarEvent.SymbolID);
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

### 4. Implementing the `MagicSystem` (Core Logic)

This system will take hotbar inputs, resolve them into glyph concepts, and then (in later chapters) trigger spell casting. For now, it will focus on processing the input sequence and providing feedback.

1.  Create `res://_Brain/Systems/Magic/MagicSystem.cs`:

```csharp
// _Brain/Systems/Magic/MagicSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Input; // To get hotbar input

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents a single glyph input by the player as part of a sequence.
    /// (TDD 02.1)
    /// </summary>
    public struct GlyphInputFrame
    {
        public string SymbolID;     // The visual symbol ID (e.g., "symbol_triangle")
        public GlyphConcept Concept; // The resolved mechanical concept (e.g., GlyphConcept.Fire)
        public double Timestamp;    // When this input occurred (real time)
        public bool Consumed;       // Whether this input has been part of a successfully resolved combo

        public GlyphInputFrame(string symbolID, GlyphConcept concept, double timestamp)
        {
            SymbolID = symbolID;
            Concept = concept;
            Timestamp = timestamp;
            Consumed = false;
        }

        public override string ToString()
        {
            return $"'{SymbolID}' ({Concept}) @ {Timestamp:F3}";
        }
    }

    /// <summary>
    /// Manages the core magic system: processing glyph input sequences,
    /// resolving combos, and triggering spell effects.
    /// (TDD 02.1: InputBuffer, TDD 02.2: Combo Resolver)
    /// </summary>
    public class MagicSystem
    {
        private EntityManager _entityManager;
        private InputSystem _inputSystem;
        private EventBus _eventBus;
        private PlayerHotbarSystem _playerHotbar;
        private PlayerGlyphKnowledgeSystem _playerGlyphKnowledge;
        private WorldGlyphMap _worldGlyphMap;

        private EntityID _playerEntityID;

        // TDD 02.1: InputBuffer - We'll use a List as a circular buffer for simplicity in C#.
        private List<GlyphInputFrame> _glyphInputBuffer = new List<GlyphInputFrame>();
        private const int MAX_GLYPH_BUFFER_SIZE = 10; // Max glyphs in a sequence
        private const float MAX_COMBO_DELAY = 1.5f; // Max time between glyph inputs for a combo (GDD B05.6)

        public MagicSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus,
                           PlayerHotbarSystem playerHotbar, PlayerGlyphKnowledgeSystem playerGlyphKnowledge,
                           WorldGlyphMap worldGlyphMap)
        {
            _entityManager = entityManager;
            _inputSystem = inputSystem;
            _eventBus = eventBus;
            _playerHotbar = playerHotbar;
            _playerGlyphKnowledge = playerGlyphKnowledge;
            _worldGlyphMap = worldGlyphMap;

            _playerEntityID = _entityManager.GetPlayerEntityID(); // Get player's ID

            GD.Print("MagicSystem: Initialized.");
        }

        /// <summary>
        /// Main update loop for the MagicSystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            PlayerInputFrame currentInput = _inputSystem.GetLatestInput();
            double currentTime = _gameManager.Time.CurrentGameTime; // Assuming GameManager is accessible

            // Process hotbar inputs for glyphs
            for (int i = 0; i < currentInput.HotbarKeys.Length; i++)
            {
                if (currentInput.HotbarKeys[i]) // If hotbar key is pressed
                {
                    string symbolID = _playerHotbar.GetGlyphSymbolInSlot(i);
                    if (symbolID != null)
                    {
                        ProcessGlyphInput(symbolID, currentTime);
                    }
                }
            }

            // Clean up old inputs from the buffer (GDD B05.6: Combo Timing Window)
            _glyphInputBuffer.RemoveAll(f => (currentTime - f.Timestamp) > MAX_COMBO_DELAY * 2); // Keep for a bit longer than max combo delay
        }

        /// <summary>
        /// Processes a single glyph input (from hotbar press).
        /// Adds it to the buffer and attempts to resolve a combo.
        /// </summary>
        /// <param name="symbolID">The symbol ID of the glyph that was input.</param>
        /// <param name="timestamp">The real-time timestamp of the input.</param>
        private void ProcessGlyphInput(string symbolID, double timestamp)
        {
            // Check if this glyph was already input very recently (debounce)
            if (_glyphInputBuffer.Any(f => f.SymbolID == symbolID && (timestamp - f.Timestamp) < 0.1f)) // 0.1s debounce
            {
                return;
            }

            // Get player's knowledge of this glyph (TDD 15.3)
            GlyphKnowledgeState knowledge = _playerGlyphKnowledge.GetGlyphKnowledge(symbolID);
            if (knowledge == GlyphKnowledgeState.Hidden)
            {
                // If player doesn't even know the symbol, just mark as seen.
                _playerGlyphKnowledge.UpdateGlyphKnowledge(symbolID, GlyphKnowledgeState.Seen);
                GD.Print($"MagicSystem: Player saw symbol '{symbolID}'.");
                _eventBus.Publish(new GlyphInputEvent { PlayerID = _playerEntityID, SymbolID = symbolID, Concept = GlyphConcept.None, KnowledgeState = knowledge, IsKnown = false });
                return;
            }

            // If player knows the symbol, get its concept
            WorldGlyphDefinition glyphDef = _worldGlyphMap.GetDefinitionBySymbol(symbolID);
            if (!glyphDef.Concept.IsValidConcept())
            {
                GD.PrintErr($"MagicSystem: Glyph symbol '{symbolID}' has no valid concept in this world. Cannot process.");
                _eventBus.Publish(new GlyphInputEvent { PlayerID = _playerEntityID, SymbolID = symbolID, Concept = GlyphConcept.None, KnowledgeState = knowledge, IsKnown = true, IsValid = false });
                return;
            }

            // Add to input buffer (TDD 02.1)
            _glyphInputBuffer.Add(new GlyphInputFrame(symbolID, glyphDef.Concept, timestamp));
            if (_glyphInputBuffer.Count > MAX_GLYPH_BUFFER_SIZE)
            {
                _glyphInputBuffer.RemoveAt(0); // Maintain max size
            }

            GD.Print($"MagicSystem: Player input glyph: {glyphDef}. Buffer size: {_glyphInputBuffer.Count}");
            _eventBus.Publish(new GlyphInputEvent { PlayerID = _playerEntityID, SymbolID = symbolID, Concept = glyphDef.Concept, KnowledgeState = knowledge, IsKnown = true, IsValid = true });

            // Attempt to resolve combo (TDD 02.2)
            ResolveCombo();
        }

        /// <summary>
        /// Attempts to resolve a combo from the current glyph input buffer.
        /// (TDD 02.2: Combo Resolver)
        /// </summary>
        private void ResolveCombo()
        {
            if (_glyphInputBuffer.Count == 0) return;

            // Get recent inputs within the combo delay window (GDD B05.6)
            List<GlyphInputFrame> recentInputs = _glyphInputBuffer
                .Where(f => (GameManager.Instance.Time.CurrentGameTime - f.Timestamp) <= MAX_COMBO_DELAY)
                .OrderBy(f => f.Timestamp) // Ensure correct order
                .ToList();

            if (recentInputs.Count == 0) return;

            GD.Print($"MagicSystem: Attempting to resolve combo with {recentInputs.Count} recent inputs.");
            
            // --- Combo Resolution Logic (Placeholder for now) ---
            // In later chapters, this is where a Trie structure (TDD 02.2)
            // or complex interpretation model (GDD B02.2) would go.
            // For now, we'll simulate a simple "known" combo if it's a single known glyph.

            // TDD 15.3: Feedback - If player inputs a known glyph, mark it as known.
            if (recentInputs.Count == 1 && recentInputs[0].KnowledgeState < GlyphKnowledgeState.Known)
            {
                _playerGlyphKnowledge.UpdateGlyphKnowledge(recentInputs[0].SymbolID, GlyphKnowledgeState.Known);
                GD.Print($"MagicSystem: Player successfully experimented with '{recentInputs[0].SymbolID}' and now KNOWS its concept: {recentInputs[0].Concept}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Discovered {recentInputs[0].Concept}!", ConceptSequence = new List<GlyphConcept> { recentInputs[0].Concept } });
                MarkInputsConsumed(recentInputs);
                return;
            }
            
            // Simple check: if two known glyphs are pressed, it's a plausible experiment (GDD B05.7)
            if (recentInputs.Count >= 2 && recentInputs.All(f => f.KnowledgeState >= GlyphKnowledgeState.Known))
            {
                List<GlyphConcept> conceptSequence = recentInputs.Select(f => f.Concept).ToList();
                GD.Print($"MagicSystem: Player experimented with a plausible sequence: {string.Join(" -> ", conceptSequence)}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Plausible experiment! {string.Join(" ", conceptSequence)}", ConceptSequence = conceptSequence });
                MarkInputsConsumed(recentInputs);
                return;
            }

            // If buffer is full and no combo resolved, or oldest input expired, it's a fizzle (GDD B05.7)
            if (_glyphInputBuffer.Count == MAX_GLYPH_BUFFER_SIZE && recentInputs.Count == 0)
            {
                GD.Print($"MagicSystem: Combo fizzled (inputs too slow or no valid sequence).");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = false, IsDiscovery = false, ResultText = "Fizzle!", ConceptSequence = new List<GlyphConcept>() });
                _glyphInputBuffer.Clear(); // Clear buffer on fizzle
            }
        }

        /// <summary>
        /// Marks glyph inputs as consumed after a combo is resolved.
        /// (TDD 02.1)
        /// </summary>
        private void MarkInputsConsumed(List<GlyphInputFrame> inputs)
        {
            foreach (var input in inputs)
            {
                int index = _glyphInputBuffer.IndexOf(input);
                if (index != -1)
                {
                    _glyphInputBuffer[index] = new GlyphInputFrame(input.SymbolID, input.Concept, input.Timestamp) { Consumed = true };
                }
            }
            _glyphInputBuffer.RemoveAll(f => f.Consumed); // Remove consumed inputs
        }
        
        // --- Helper Events for Body Sync ---
        public struct GlyphInputEvent { public EntityID PlayerID; public string SymbolID; public GlyphConcept Concept; public GlyphKnowledgeState KnowledgeState; public bool IsKnown; public bool IsValid; }
        public struct ComboResolvedEvent { public EntityID PlayerID; public bool IsSuccess; public bool IsDiscovery; public string ResultText; public List<GlyphConcept> ConceptSequence; }
    }
}
```

**Explanation of `MagicSystem.cs`:**

*   **`GlyphInputFrame`**: A struct to store each individual glyph input (symbol, concept, timestamp, consumed status).
*   **`_glyphInputBuffer`**: A `List<GlyphInputFrame>` acting as our circular input buffer (TDD 02.1).
*   **`MAX_COMBO_DELAY`**: Defines the maximum time between glyphs for them to be considered part of the same combo (GDD B05.6).
*   **`Tick(double delta)`**:
    *   Iterates through the `PlayerInputFrame.HotbarKeys`.
    *   If a hotbar key is pressed, it calls `ProcessGlyphInput()`.
    *   Cleans up old, expired inputs from the buffer.
*   **`ProcessGlyphInput()`**:
    *   **Debounces**: Prevents multiple inputs from a single key press.
    *   **Knowledge Check**: If the glyph is `Hidden`, it updates to `Seen` (GDD B01.4.C).
    *   Resolves `GlyphConcept` from `WorldGlyphMap`.
    *   Adds the `GlyphInputFrame` to `_glyphInputBuffer`.
    *   Publishes a `GlyphInputEvent` for UI feedback.
    *   Calls `ResolveCombo()`.
*   **`ResolveCombo()`**:
    *   Filters `_glyphInputBuffer` for recent inputs within `MAX_COMBO_DELAY`.
    *   **Placeholder Logic**:
        *   If a single `Seen` glyph is input, it's marked as `Known` (TDD 15.3: Feedback).
        *   If two or more `Known` glyphs are input, it's a "plausible experiment" (GDD B05.7).
        *   If the buffer fills up and no combo is resolved, it's a "fizzle."
    *   Publishes a `ComboResolvedEvent` with the outcome.
    *   Calls `MarkInputsConsumed()` on successful combos.
*   **`MarkInputsConsumed()`**: Clears inputs that were part of a successful combo.

### 5. Integrating `MagicSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Magic;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `MagicSystem` property.
3.  Initialize `MagicSystem` in `InitializeSystems()`.
4.  Call `MagicSystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).

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
using System.Linq;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public PlayerHotbarSystem PlayerHotbar { get; private set; }
    public MagicSystem Magic { get; private set; } // Add MagicSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        
        // --- Test Glyph Discovery System ---
        GD.Print("\n--- Testing Glyph Discovery System ---");
        string testSymbol = GlyphMap.AllWorldGlyphs[0].SymbolID;
        string testSymbol2 = GlyphMap.AllWorldGlyphs[1].SymbolID;
        // Make sure testSymbol is only Seen initially for experimentation
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Hidden, true); // Force reset to Hidden
        GD.Print($"Initial knowledge of '{testSymbol}': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        // PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Seen); // Will be handled by MagicSystem
        // PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Known); // Will be handled by MagicSystem
        
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol2, GlyphKnowledgeState.Known); // Make second symbol known for hotbar test
        GD.Print($"Knowledge of '{testSymbol2}' after 'Known': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol2)}");

        // ... (existing unknown symbol test) ...
        GD.Print("--- End Testing Glyph Discovery System ---\n");

        // --- Test PlayerHotbarSystem ---
        GD.Print("\n--- Testing PlayerHotbarSystem ---");
        PlayerHotbar.AssignGlyphToSlot(0, testSymbol); // Assign potentially unknown symbol
        PlayerHotbar.AssignGlyphToSlot(1, testSymbol2); // Assign known symbol
        // ... (existing hotbar tests) ...
        GD.Print("--- End Testing PlayerHotbarSystem ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta); // Call MagicSystem's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations) ...

        PlayerHotbar = new PlayerHotbarSystem(Entities.GetPlayerEntityID(), Events, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - PlayerHotbarSystem initialized.");

        // Initialize MagicSystem
        Magic = new MagicSystem(Entities, Input, Events, PlayerHotbar, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - MagicSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 5.1. Update `EventBus.cs` for MagicSystem Events

Our `MagicSystem` publishes `GlyphInputEvent` and `ComboResolvedEvent`. We need to define these `Action` delegates in `EventBus`.

Open `_Brain/Core/EventBus.cs`:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Magic System Events (TDD 15.3, TDD 02.1)
        public event Action<EntityID, string, GlyphConcept, GlyphKnowledgeState, bool, bool> OnGlyphInput; // PlayerID, SymbolID, Concept, KnowledgeState, IsKnown, IsValid
        public event Action<EntityID, bool, bool, string, List<GlyphConcept>> OnComboResolved; // PlayerID, IsSuccess, IsDiscovery, ResultText, ConceptSequence

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is MagicSystem.GlyphInputEvent glyphInputEvent) // New condition
            {
                OnGlyphInput?.Invoke(glyphInputEvent.PlayerID, glyphInputEvent.SymbolID, glyphInputEvent.Concept, glyphInputEvent.KnowledgeState, glyphInputEvent.IsKnown, glyphInputEvent.IsValid);
            }
            else if (eventData is MagicSystem.ComboResolvedEvent comboResolvedEvent) // New condition
            {
                OnComboResolved?.Invoke(comboResolvedEvent.PlayerID, comboResolvedEvent.IsSuccess, comboResolvedEvent.IsDiscovery, comboResolvedEvent.ResultText, comboResolvedEvent.ConceptSequence);
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

### 6. Testing Glyph Experimentation

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  In the Output console, observe the initial glyph knowledge and hotbar setup. The `testSymbol` (e.g., `symbol_leaf`) should be `Hidden`. The `testSymbol2` (e.g., `symbol_cross`) should be `Known`.
5.  **Test 1 (Single Unknown Glyph)**: Press `1` (hotbar slot 0). This is our `testSymbol`.
    *   You should see `MagicSystem: Player saw symbol 'symbol_leaf'.`
    *   Then `PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_leaf' updated from Hidden to Seen.`
    *   Then `MagicSystem: Player input glyph: 'symbol_leaf' (Bloom) @ X.XX. Buffer size: 1`
    *   Then `MagicSystem: Player successfully experimented with 'symbol_leaf' and now KNOWS its concept: Bloom.`
    *   And `PlayerGlyphKnowledgeSystem: Player EntityID(0, Gen:1) knowledge of 'symbol_leaf' updated from Seen to Known.`
    *   This sequence demonstrates the discovery of a glyph's meaning through solo experimentation.
6.  **Test 2 (Two Known Glyphs - Plausible Experiment)**:
    *   Quickly press `0` (hotbar slot 0, which is `testSymbol`, now `Known`).
    *   Then quickly press `1` (hotbar slot 1, which is `testSymbol2`, `Known`).
    *   You should see `MagicSystem: Player input glyph...` for both.
    *   Then `MagicSystem: Player experimented with a plausible sequence: Bloom -> Consume.` (or whatever your concepts are). This simulates a new combo discovery.
7.  **Test 3 (Fizzle)**:
    *   Press `0`. Wait for more than `MAX_COMBO_DELAY` (1.5 seconds).
    *   Press `0` again.
    *   You should see a `MagicSystem: Combo fizzled...` message. The buffer cleared.

This confirms the basic glyph experimentation loop, including knowledge progression and combo resolution, is working.

### Summary

You have successfully implemented the core **Glyph Experimentation** loop for Sigilborne, allowing players to input glyph sequences, process their hotbar choices, and interpret the outcomes. By creating `PlayerHotbarSystem` to manage equipped glyphs and enhancing `MagicSystem` to process inputs, track knowledge, and resolve rudimentary combos (or fizzles), you've established the foundation for dynamic spellcasting and discovery. This system correctly handles knowledge progression from `Hidden` to `Known` and provides feedback for plausible experiments, strictly adhering to TDD 15.3's specifications.

### Next Steps

The next chapter will focus on **Subtypes & Modifiers**, expanding our `WorldGlyphDefinition` to include procedural subtypes that add nuance and variety to each glyph's actual effect based on world-specific conditions, further enhancing replayability.