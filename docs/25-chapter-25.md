## Chapter 4.1: Input Buffer - Storing Glyph Sequences (C#)

Welcome to **Module 4: Combos & Casting - The Art of Ninjutsu**! This module focuses on how glyphs combine into powerful techniques. The foundation of any combo system is a robust way to store and retrieve a sequence of player inputs. This chapter refines our `MagicSystem` by implementing a dedicated **Input Buffer** to store `GlyphInputFrame` structs, allowing for efficient detection of glyph sequences within a time window, as specified in TDD 02.1.

### 1. The Need for a Dedicated Glyph Input Buffer

Our `MagicSystem` currently uses a `List<GlyphInputFrame>` to store recent glyph inputs. While functional, we need to formalize this into a proper circular buffer (or a `List` managed like one) for clarity and to adhere to TDD 02.1's specification.

**Key Requirements for the Input Buffer:**

*   **Order Preservation**: Inputs must be stored in the order they occurred.
*   **Timestamping**: Each input needs a precise timestamp to determine if it's within a combo window.
*   **Fixed Size / Max Count**: We only care about the most recent inputs relevant for combos.
*   **Efficient Retrieval**: Quickly get a `ReadOnlySpan<GlyphInputFrame>` of recent inputs.

### 2. Refining `GlyphInputFrame`

We already defined `GlyphInputFrame` in `MagicSystem.cs` in Chapter 3.3. Let's move it to its own file (`GlyphInputFrame.cs`) for better modularity and ensure it includes the `Consumed` flag as per TDD 02.1.

1.  Create `res://_Brain/Systems/Magic/GlyphInputFrame.cs`:

```csharp
// _Brain/Systems/Magic/GlyphInputFrame.cs
using System;
using Godot; // For Vector2 if needed, though not directly in this struct

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents a single glyph input by the player as part of a sequence.
    /// This struct is stored in the MagicSystem's input buffer.
    /// (TDD 02.1)
    /// </summary>
    public struct GlyphInputFrame : IEquatable<GlyphInputFrame>
    {
        public string SymbolID;     // The visual symbol ID (e.g., "symbol_triangle")
        public GlyphConcept Concept; // The resolved mechanical concept (e.g., GlyphConcept.Fire)
        public double Timestamp;    // When this input occurred (real time)
        public bool Consumed;       // Whether this input has been part of a successfully resolved combo (TDD 02.1)

        public GlyphInputFrame(string symbolID, GlyphConcept concept, double timestamp, bool consumed = false)
        {
            SymbolID = symbolID;
            Concept = concept;
            Timestamp = timestamp;
            Consumed = consumed;
        }

        public override string ToString()
        {
            return $"'{SymbolID}' ({Concept}) @ {Timestamp:F3} (Consumed: {Consumed})";
        }

        // --- IEquatable Implementation ---
        public bool Equals(GlyphInputFrame other)
        {
            // For equality, we typically check all fields, or just the unique identifier if one exists.
            // For the purpose of tracking inputs, all fields should match for true equality.
            return SymbolID == other.SymbolID &&
                   Concept == other.Concept &&
                   Timestamp == other.Timestamp &&
                   Consumed == other.Consumed;
        }

        public override bool Equals(object obj)
        {
            return obj is GlyphInputFrame other && Equals(other);
        }

        public override int GetHashCode()
        {
            return HashCode.Combine(SymbolID, Concept, Timestamp, Consumed);
        }
    }
}
```

2.  Now, open `res://_Brain/Systems/Magic/MagicSystem.cs` and remove the `GlyphInputFrame` struct definition from it. Add `using Sigilborne.Systems.Magic;` at the top if it's not already there.

### 3. Implementing the `GlyphInputBuffer` Class

TDD 02.1 suggests a circular buffer. While a `List` with `RemoveAt(0)` can simulate this, a dedicated `GlyphInputBuffer` class (or even a `Queue` or `LinkedList`) provides clearer semantics and better control. For efficiency with `ReadOnlySpan<T>`, a raw array or a `List` managed carefully is often preferred. We'll stick with a `List` and manage its size, as it's flexible for `ReadOnlySpan<T>` conversion.

1.  Create `res://_Brain/Systems/Magic/GlyphInputBuffer.cs`:

```csharp
// _Brain/Systems/Magic/GlyphInputBuffer.cs
using System;
using System.Collections.Generic;
using System.Linq; // For debugging output

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// A circular buffer that stores recent GlyphInputFrames with precise timestamps.
    /// (TDD 02.1)
    /// </summary>
    public class GlyphInputBuffer
    {
        private List<GlyphInputFrame> _buffer;
        private int _maxSize;

        public GlyphInputBuffer(int maxSize)
        {
            _maxSize = maxSize;
            _buffer = new List<GlyphInputFrame>(maxSize);
        }

        /// <summary>
        /// Pushes a new GlyphInputFrame into the buffer.
        /// If the buffer is full, the oldest unconsumed input is removed.
        /// </summary>
        public void Push(GlyphInputFrame frame)
        {
            _buffer.Add(frame);
            if (_buffer.Count > _maxSize)
            {
                // Remove the oldest unconsumed input, or just the oldest if all are consumed/no unconsumed
                // This logic might need refinement based on exact combo rules (e.g., if consumed inputs are needed for historical context)
                int oldestUnconsumedIndex = _buffer.FindIndex(f => !f.Consumed);
                if (oldestUnconsumedIndex != -1 && _buffer.Count > _maxSize)
                {
                    _buffer.RemoveAt(oldestUnconsumedIndex);
                }
                else if (_buffer.Count > _maxSize) // If all are consumed or no unconsumed, just remove the oldest
                {
                    _buffer.RemoveAt(0);
                }
            }
        }

        /// <summary>
        /// Retrieves a ReadOnlySpan of recent, unconsumed glyph inputs within a specified time window.
        /// (TDD 02.1)
        /// </summary>
        /// <param name="windowStartTime">The minimum timestamp for inputs to be considered.</param>
        public ReadOnlySpan<GlyphInputFrame> GetRecentUnconsumed(double windowStartTime)
        {
            // Filter and order inputs from the current buffer
            // For performance, avoid Linq in hot paths. This is a simple example.
            var recent = _buffer.Where(f => !f.Consumed && f.Timestamp >= windowStartTime).OrderBy(f => f.Timestamp).ToList();
            
            // To return a ReadOnlySpan, we need to ensure the underlying data is contiguous.
            // Converting to an array is the safest way for a List<T>.
            return new ReadOnlySpan<GlyphInputFrame>(recent.ToArray());
        }

        /// <summary>
        /// Marks a set of inputs as consumed.
        /// </summary>
        public void MarkConsumed(IEnumerable<GlyphInputFrame> inputsToConsume)
        {
            foreach (var inputToConsume in inputsToConsume)
            {
                for (int i = 0; i < _buffer.Count; i++)
                {
                    if (_buffer[i].Equals(inputToConsume)) // Use struct equality
                    {
                        _buffer[i] = new GlyphInputFrame(inputToConsume.SymbolID, inputToConsume.Concept, inputToConsume.Timestamp, true);
                        break; // Found and updated
                    }
                }
            }
            // After marking, optionally remove consumed inputs to keep buffer clean
            _buffer.RemoveAll(f => f.Consumed);
        }

        /// <summary>
        /// Clears all inputs from the buffer.
        /// </summary>
        public void Clear()
        {
            _buffer.Clear();
        }

        public int Count => _buffer.Count;

        public override string ToString()
        {
            return $"Buffer ({_buffer.Count}/{_maxSize}): [{string.Join(", ", _buffer.Select(f => f.ToString()))}]";
        }
    }
}
```

### 4. Integrating `GlyphInputBuffer` into `MagicSystem`

Now, `MagicSystem` will use this dedicated `GlyphInputBuffer` class.

1.  Open `res://_Brain/Systems/Magic/MagicSystem.cs`.
2.  Replace `private List<GlyphInputFrame> _glyphInputBuffer = new List<GlyphInputFrame>();` with:
    `private GlyphInputBuffer _glyphInputBuffer;`
3.  Modify the `MagicSystem` constructor to initialize `_glyphInputBuffer`.
4.  Update `ProcessGlyphInput` and `ResolveCombo` to use the `GlyphInputBuffer`'s `Push`, `GetRecentUnconsumed`, `MarkConsumed`, and `Clear` methods.

```csharp
// _Brain/Systems/Magic/MagicSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Input;

namespace Sigilborne.Systems.Magic
{
    // ... (removed GlyphInputFrame struct definition) ...

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
        private GameManager _gameManager; // Reference to GameManager for CurrentGameTime

        private EntityID _playerEntityID;

        // TDD 02.1: InputBuffer - Now using our dedicated GlyphInputBuffer class.
        private GlyphInputBuffer _glyphInputBuffer;
        private const int MAX_GLYPH_BUFFER_SIZE = 10; // Max glyphs in a sequence (GDD B02.3)
        private const float MAX_COMBO_DELAY = 1.5f; // Max time between glyph inputs for a combo (GDD B05.6)
        private const float GLYPH_DEBOUNCE_TIME = 0.1f; // Prevents multiple inputs from a single key press

        public MagicSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus,
                           PlayerHotbarSystem playerHotbar, PlayerGlyphKnowledgeSystem playerGlyphKnowledge,
                           WorldGlyphMap worldGlyphMap, GameManager gameManager) // Add GameManager
        {
            _entityManager = entityManager;
            _inputSystem = inputSystem;
            _eventBus = eventBus;
            _playerHotbar = playerHotbar;
            _playerGlyphKnowledge = playerGlyphKnowledge;
            _worldGlyphMap = worldGlyphMap;
            _gameManager = gameManager; // Store GameManager reference

            _playerEntityID = _entityManager.GetPlayerEntityID();

            _glyphInputBuffer = new GlyphInputBuffer(MAX_GLYPH_BUFFER_SIZE); // Initialize the new buffer
            GD.Print("MagicSystem: Initialized.");
        }

        public void Tick(double delta)
        {
            PlayerInputFrame currentInput = _inputSystem.GetLatestInput();
            double currentTime = _gameManager.Time.CurrentGameTime;

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

            // No longer need to manually remove old inputs here, GlyphInputBuffer manages its size
            // However, we might still want to trigger a fizzle if the oldest input is too old and no combo has resolved.
            // This is part of the combo timing window logic.
            if (_glyphInputBuffer.Count > 0 && (currentTime - _glyphInputBuffer.GetRecentUnconsumed(0).ToArray().First().Timestamp) > MAX_COMBO_DELAY)
            {
                // Oldest input expired, and no combo resolved. Fizzle.
                GD.Print($"MagicSystem: Combo fizzled (inputs too slow). Buffer cleared.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = false, IsDiscovery = false, ResultText = "Fizzle!", ConceptSequence = new List<GlyphConcept>() });
                _glyphInputBuffer.Clear();
            }
        }

        private void ProcessGlyphInput(string symbolID, double timestamp)
        {
            // Debounce (GDD B05.6) - Check if this glyph was already input very recently
            if (_glyphInputBuffer.GetRecentUnconsumed(timestamp - GLYPH_DEBOUNCE_TIME).Any(f => f.SymbolID == symbolID))
            {
                return;
            }

            GlyphKnowledgeState knowledge = _playerGlyphKnowledge.GetGlyphKnowledge(symbolID);
            if (knowledge == GlyphKnowledgeState.Hidden)
            {
                _playerGlyphKnowledge.UpdateGlyphKnowledge(symbolID, GlyphKnowledgeState.Seen);
                GD.Print($"MagicSystem: Player saw symbol '{symbolID}'.");
                _eventBus.Publish(new GlyphInputEvent { PlayerID = _playerEntityID, SymbolID = symbolID, Concept = GlyphConcept.None, KnowledgeState = knowledge, IsKnown = false, IsValid = false });
                return;
            }

            WorldGlyphDefinition glyphDef = _worldGlyphMap.GetDefinitionBySymbol(symbolID);
            if (!glyphDef.Concept.IsValidConcept())
            {
                GD.PrintErr($"MagicSystem: Glyph symbol '{symbolID}' has no valid concept in this world. Cannot process.");
                _eventBus.Publish(new GlyphInputEvent { PlayerID = _playerEntityID, SymbolID = symbolID, Concept = GlyphConcept.None, KnowledgeState = knowledge, IsKnown = true, IsValid = false });
                return;
            }

            // Add to input buffer (TDD 02.1)
            _glyphInputBuffer.Push(new GlyphInputFrame(symbolID, glyphDef.Concept, timestamp));

            GD.Print($"MagicSystem: Player input glyph: {glyphDef}. Buffer size: {_glyphInputBuffer.Count}");
            _eventBus.Publish(new GlyphInputEvent { PlayerID = _playerEntityID, SymbolID = symbolID, Concept = glyphDef.Concept, KnowledgeState = knowledge, IsKnown = true, IsValid = true });

            // Attempt to resolve combo (TDD 02.2)
            ResolveCombo();
        }

        private void ResolveCombo()
        {
            // TDD 02.1: Get recent inputs within the combo delay window.
            // ReadOnlySpan<GlyphInputFrame> recentInputsSpan = _glyphInputBuffer.GetRecentUnconsumed(GameManager.Instance.Time.CurrentGameTime - MAX_COMBO_DELAY);
            // Convert to List for easier manipulation in this simple combo resolver.
            List<GlyphInputFrame> recentInputs = _glyphInputBuffer.GetRecentUnconsumed(_gameManager.Time.CurrentGameTime - MAX_COMBO_DELAY).ToList();

            if (recentInputs.Count == 0) return;

            GD.Print($"MagicSystem: Attempting to resolve combo with {recentInputs.Count} recent inputs.");
            
            // --- Combo Resolution Logic (Placeholder for now) ---
            // Simplified logic from previous chapter.
            // TDD 15.3: Feedback - If player inputs a single known glyph, mark it as known.
            if (recentInputs.Count == 1 && recentInputs[0].KnowledgeState < GlyphKnowledgeState.Known)
            {
                _playerGlyphKnowledge.UpdateGlyphKnowledge(recentInputs[0].SymbolID, GlyphKnowledgeState.Known);
                GD.Print($"MagicSystem: Player successfully experimented with '{recentInputs[0].SymbolID}' and now KNOWS its concept: {recentInputs[0].Concept}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Discovered {recentInputs[0].Concept}!", ConceptSequence = new List<GlyphConcept> { recentInputs[0].Concept } });
                _glyphInputBuffer.MarkConsumed(recentInputs);
                return;
            }
            
            // Simple check: if two known glyphs are pressed, it's a plausible experiment (GDD B05.7)
            if (recentInputs.Count >= 2 && recentInputs.All(f => f.KnowledgeState >= GlyphKnowledgeState.Known))
            {
                List<GlyphConcept> conceptSequence = recentInputs.Select(f => f.Concept).ToList();
                GD.Print($"MagicSystem: Player experimented with a plausible sequence: {string.Join(" -> ", conceptSequence)}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Plausible experiment! {string.Join(" ", conceptSequence)}", ConceptSequence = conceptSequence });
                _glyphInputBuffer.MarkConsumed(recentInputs);
                return;
            }

            // Fizzle logic is now primarily handled in Tick() if inputs expire, or if explicit invalid combo.
            // This placeholder resolver does not handle complex invalid combos yet.
        }
        
        // Removed MarkInputsConsumed, it's now part of GlyphInputBuffer.MarkConsumed
        // Removed Clear, it's now part of GlyphInputBuffer.Clear
    }
}
```

**Key Changes in `MagicSystem.cs`:**

*   Now uses `GlyphInputBuffer _glyphInputBuffer` instead of a raw `List`.
*   Constructor now takes `GameManager` to access `Time.CurrentGameTime`.
*   `Push` and `MarkConsumed` are delegated to the `_glyphInputBuffer` instance.
*   The `Tick` method now includes logic to fizzle combos if the oldest input expires due to `MAX_COMBO_DELAY`.
*   `ProcessGlyphInput` includes `GLYPH_DEBOUNCE_TIME` to prevent accidental double-inputs.

### 5. Update `GameManager` to Pass `GameManager` to `MagicSystem`

1.  Open `res://_Brain/Core/GameManager.cs`.
2.  Modify the `MagicSystem` initialization in `InitializeSystems()` to pass `this`.

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        PlayerHotbar = new PlayerHotbarSystem(Entities.GetPlayerEntityID(), Events, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - PlayerHotbarSystem initialized.");

        // Initialize MagicSystem, passing GameManager itself for CurrentGameTime access
        Magic = new MagicSystem(Entities, Input, Events, PlayerHotbar, PlayerGlyphKnowledge, GlyphMap, this); // Pass 'this'
        GD.Print("  - MagicSystem initialized.");
// ...
```

### 6. Testing the Input Buffer

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Perform the same tests as in Chapter 3.3:
    *   **Test 1 (Single Unknown Glyph)**: Press `1` (hotbar slot 0). This is our `testSymbol` (initially `Hidden`). You should see it become `Seen`, then `Known`, and a "Discovered Concept!" message.
    *   **Test 2 (Two Known Glyphs - Plausible Experiment)**: Quickly press `0` then `1`. You should see a "Plausible experiment!" message.
    *   **Test 3 (Fizzle)**: Press `0`. Wait for more than `MAX_COMBO_DELAY` (1.5 seconds). Press `0` again. You should see a `MagicSystem: Combo fizzled (inputs too slow)...` message.

The output and behavior should be consistent with the previous chapter, but now the underlying `GlyphInputBuffer` is handling the sequence storage more robustly. The explicit `_glyphInputBuffer.ToString()` can be used in debug to see the buffer's state.

### Summary

You have successfully implemented a dedicated **Input Buffer** for storing `GlyphInputFrame` sequences, enhancing the `MagicSystem`'s ability to process player input for combos. By creating the `GlyphInputBuffer` class and integrating it into `MagicSystem`, you've established a robust mechanism for managing glyph input order, timestamps, and consumption, strictly adhering to TDD 02.1's specifications. This refined input pipeline is crucial for the efficient and accurate detection of complex glyph combos.

### Next Steps

The next chapter will implement the **Combo Resolver** using a Trie structure (TDD 02.2) to efficiently detect known spell sequences from the glyph input buffer, moving us closer to a fully functional magic system.