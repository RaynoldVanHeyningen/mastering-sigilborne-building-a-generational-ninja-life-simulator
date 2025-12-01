## Chapter 4.2: Combo Resolver - Trie Structure for Spell Detection (C#)

With our `GlyphInputBuffer` accurately storing glyph sequences, the next challenge is efficiently detecting if these sequences correspond to known spells or combo patterns. This chapter implements the **Combo Resolver** using a **Prefix Tree (Trie)** structure, as specified in TDD 02.2. This data structure is ideal for quickly matching input sequences and is fundamental for Sigilborne's dynamic combo system.

### 1. The Need for a Trie-Based Combo Resolver

The GDD (B02.2) emphasizes that "any sequence can be attempted" and "the world's logic interprets them." A Trie allows for:

*   **Efficient Prefix Matching**: Quickly determine if a partial input sequence could lead to a valid combo.
*   **Dynamic Combo Registration**: Easily add or remove new combos (spells) to the system.
*   **Hierarchical Structure**: Naturally represents sequences of glyphs.

### 2. Defining `SpellDefinition`

Before we build the Trie, we need a way to define what a "spell" or "combo" actually *is*. TDD 02.3 specifies `SpellDefinition` as data, not code.

1.  Create `res://_Brain/Systems/Magic/SpellDefinition.cs`:

```csharp
// _Brain/Systems/Magic/SpellDefinition.cs
using System;
using System.Collections.Generic;
using Godot; // For Vector2, though not directly used in this struct yet
using Sigilborne.Entities; // For EntityID if spell targets an entity

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents the static data defining a single spell or combo technique.
    /// Spells are data-driven to allow for thousands of variations and mythic evolution.
    /// (TDD 02.3)
    /// </summary>
    public class SpellDefinition
    {
        public string ID { get; private set; } // Unique ID (e.g., "fireball_01", "bloom_veil_healing_mist")
        public List<GlyphConcept> Sequence { get; private set; } // The required sequence of GlyphConcepts
        public float BaseDamage { get; private set; }
        public float ChakraCost { get; private set; } // TDD 02.3: ManaCost renamed to ChakraCost
        public float CastTime { get; private set; }
        public float StabilityCost { get; private set; } // GDD B03.2: Strain Accumulation Model
        public float ResidueGeneration { get; private set; } // GDD B03.7: Chaos Residue System
        public bool IsMythic { get; private set; } // TDD 02.3: For Mythic Evolution (C01)
        public bool IsForbidden { get; private set; } // GDD B01.10: Forbidden Glyphs (C04)

        // Components (ECS-lite) - TDD 02.3
        // These will be actual structs or classes in later chapters.
        public ProjectileData Projectile { get; private set; } // Speed, Size, Pierce
        public AoEData Explosion { get; private set; } // Radius, Falloff
        public List<StatusEffectData> Effects { get; private set; } // Burn, Stun

        public SpellDefinition(string id, List<GlyphConcept> sequence, float baseDamage, float chakraCost, float castTime, float stabilityCost, float residueGeneration, bool isMythic, bool isForbidden, ProjectileData projectile = null, AoEData explosion = null, List<StatusEffectData> effects = null)
        {
            ID = id;
            Sequence = sequence ?? new List<GlyphConcept>();
            BaseDamage = baseDamage;
            ChakraCost = chakraCost;
            CastTime = castTime;
            StabilityCost = stabilityCost;
            ResidueGeneration = residueGeneration;
            IsMythic = isMythic;
            IsForbidden = isForbidden;
            Projectile = projectile;
            Explosion = explosion;
            Effects = effects ?? new List<StatusEffectData>();
        }

        public override string ToString()
        {
            string seq = string.Join("->", Sequence.Select(c => c.ToString()));
            return $"Spell: '{ID}' ({seq}) | Dmg: {BaseDamage}, Chakra: {ChakraCost}, StabCost: {StabilityCost}";
        }

        // --- Placeholder Component Data Structs (TDD 02.3) ---
        public class ProjectileData { public float Speed; public float Size; public int Pierce; public override string ToString() => $"Proj(Spd:{Speed},Sz:{Size})"; }
        public class AoEData { public float Radius; public float Falloff; public override string ToString() => $"AoE(Rad:{Radius})"; }
        public class StatusEffectData { public string EffectID; public float Duration; public override string ToString() => $"Effect({EffectID})"; }
    }
}
```

### 3. The `ComboResolver` (Trie Structure)

Now, let's implement the Trie. Each node in the Trie will represent a `GlyphConcept`. Leaf nodes will store the `SpellDefinition` that corresponds to the completed sequence.

1.  Create `res://_Brain/Systems/Magic/ComboResolver.cs`:

```csharp
// _Brain/Systems/Magic/ComboResolver.cs
using System;
using System.Collections.Generic;
using System.Linq;
using Godot;

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents a node in the Trie structure for combo resolution.
    /// (TDD 02.2)
    /// </summary>
    public class TrieNode
    {
        public Dictionary<GlyphConcept, TrieNode> Children { get; } = new Dictionary<GlyphConcept, TrieNode>();
        public SpellDefinition Spell { get; set; } // Null if not a complete spell, otherwise the SpellDefinition

        public override string ToString()
        {
            return $"Node (Children: {Children.Count}) {(Spell != null ? $"[Spell: {Spell.ID}]" : "")}";
        }
    }

    /// <summary>
    /// Uses a Prefix Tree (Trie) to efficiently detect spells from glyph input sequences.
    /// (TDD 02.2)
    /// </summary>
    public class ComboResolver
    {
        private TrieNode _root = new TrieNode();

        public ComboResolver()
        {
            GD.Print("ComboResolver: Initialized.");
        }

        /// <summary>
        /// Registers a new SpellDefinition into the Trie structure.
        /// </summary>
        public void RegisterSpell(SpellDefinition spell)
        {
            if (spell.Sequence == null || spell.Sequence.Count == 0)
            {
                GD.PrintErr($"ComboResolver: Cannot register spell '{spell.ID}' with empty sequence.");
                return;
            }

            TrieNode currentNode = _root;
            foreach (GlyphConcept concept in spell.Sequence)
            {
                if (!currentNode.Children.ContainsKey(concept))
                {
                    currentNode.Children[concept] = new TrieNode();
                }
                currentNode = currentNode.Children[concept];
            }
            // If a spell already exists at this node, it means a shorter sequence
            // has the same effect, or we're overwriting. Log a warning.
            if (currentNode.Spell != null)
            {
                GD.PrintWarning($"ComboResolver: Overwriting spell '{currentNode.Spell.ID}' with '{spell.ID}' for sequence '{string.Join("->", spell.Sequence)}'.");
            }
            currentNode.Spell = spell;
            GD.Print($"ComboResolver: Registered spell '{spell.ID}' with sequence '{string.Join("->", spell.Sequence)}'.");
        }

        /// <summary>
        /// Attempts to find the longest matching spell in the Trie for a given sequence of glyph inputs.
        /// (TDD 02.2: Algorithm)
        /// </summary>
        /// <param name="glyphInputs">The sequence of glyph inputs to check.</param>
        /// <returns>The longest matching SpellDefinition, or null if no spell is found.</returns>
        public SpellDefinition ResolveCombo(ReadOnlySpan<GlyphInputFrame> glyphInputs)
        {
            if (glyphInputs.IsEmpty) return null;

            SpellDefinition bestMatch = null;
            TrieNode currentNode = _root;
            int longestMatchLength = 0;

            for (int i = 0; i < glyphInputs.Length; i++)
            {
                GlyphConcept concept = glyphInputs[i].Concept;

                if (currentNode.Children.TryGetValue(concept, out TrieNode nextNode))
                {
                    currentNode = nextNode;
                    // If this node completes a spell, it's a potential match.
                    // We keep searching for a longer match.
                    if (currentNode.Spell != null)
                    {
                        bestMatch = currentNode.Spell;
                        longestMatchLength = i + 1; // Length of the sequence that matched
                    }
                }
                else
                {
                    // No further match for this sequence, break.
                    break;
                }
            }
            return bestMatch;
        }

        // --- Debugging / Utility ---
        public void PrintTrie(TrieNode node = null, string prefix = "")
        {
            if (node == null) node = _root;
            
            foreach (var childEntry in node.Children)
            {
                string newPrefix = prefix + childEntry.Key.ToString() + "->";
                GD.Print($"{newPrefix}{(childEntry.Value.Spell != null ? $" [Spell: {childEntry.Value.Spell.ID}]" : "")}");
                PrintTrie(childEntry.Value, newPrefix);
            }
        }
    }
}
```

### 4. Integrating `ComboResolver` into `MagicSystem`

Now, `MagicSystem` will use `ComboResolver` to detect actual spells.

1.  Open `res://_Brain/Systems/Magic/MagicSystem.cs`.
2.  Add a `ComboResolver` property.
3.  Modify the `MagicSystem` constructor to initialize `_comboResolver`.
4.  Update `ResolveCombo` to use `_comboResolver.ResolveCombo()`.

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
    // ... (GlyphInputFrame struct) ...

    public class MagicSystem
    {
        private EntityManager _entityManager;
        private InputSystem _inputSystem;
        private EventBus _eventBus;
        private PlayerHotbarSystem _playerHotbar;
        private PlayerGlyphKnowledgeSystem _playerGlyphKnowledge;
        private WorldGlyphMap _worldGlyphMap;
        private GameManager _gameManager;

        private EntityID _playerEntityID;

        private GlyphInputBuffer _glyphInputBuffer;
        private ComboResolver _comboResolver; // New: ComboResolver instance
        
        private const int MAX_GLYPH_BUFFER_SIZE = 10;
        private const float MAX_COMBO_DELAY = 1.5f;
        private const float GLYPH_DEBOUNCE_TIME = 0.1f;

        public MagicSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus,
                           PlayerHotbarSystem playerHotbar, PlayerGlyphKnowledgeSystem playerGlyphKnowledge,
                           WorldGlyphMap worldGlyphMap, GameManager gameManager, ComboResolver comboResolver) // Add ComboResolver
        {
            _entityManager = entityManager;
            _inputSystem = inputSystem;
            _eventBus = eventBus;
            _playerHotbar = playerHotbar;
            _playerGlyphKnowledge = playerGlyphKnowledge;
            _worldGlyphMap = worldGlyphMap;
            _gameManager = gameManager;

            _playerEntityID = _entityManager.GetPlayerEntityID();

            _glyphInputBuffer = new GlyphInputBuffer(MAX_GLYPH_BUFFER_SIZE);
            _comboResolver = comboResolver; // Initialize ComboResolver
            GD.Print("MagicSystem: Initialized.");
        }

        // ... (Tick, ProcessGlyphInput methods) ...

        private void ResolveCombo()
        {
            // TDD 02.1: Get recent inputs within the combo delay window.
            ReadOnlySpan<GlyphInputFrame> recentInputsSpan = _glyphInputBuffer.GetRecentUnconsumed(_gameManager.Time.CurrentGameTime - MAX_COMBO_DELAY);
            
            if (recentInputsSpan.IsEmpty) return;

            GD.Print($"MagicSystem: Attempting to resolve combo with {recentInputsSpan.Length} recent inputs.");
            
            // --- NEW: Use ComboResolver to find a known spell (TDD 02.2) ---
            SpellDefinition resolvedSpell = _comboResolver.ResolveCombo(recentInputsSpan);

            if (resolvedSpell != null)
            {
                // Found a known spell!
                GD.Print($"MagicSystem: Resolved known spell: '{resolvedSpell.ID}'!");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = false, ResultText = $"Cast {resolvedSpell.ID}!", ConceptSequence = resolvedSpell.Sequence });
                _glyphInputBuffer.MarkConsumed(recentInputsSpan.ToArray()); // Mark inputs as consumed
                return;
            }
            // --- END NEW ---

            // --- Existing Placeholder Combo Resolution Logic ---
            // Simplified logic from previous chapter.
            // If player inputs a single unknown glyph, mark it as known.
            if (recentInputsSpan.Length == 1 && recentInputsSpan[0].KnowledgeState < GlyphKnowledgeState.Known)
            {
                _playerGlyphKnowledge.UpdateGlyphKnowledge(recentInputsSpan[0].SymbolID, GlyphKnowledgeState.Known);
                GD.Print($"MagicSystem: Player successfully experimented with '{recentInputsSpan[0].SymbolID}' and now KNOWS its concept: {recentInputsSpan[0].Concept}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Discovered {recentInputsSpan[0].Concept}!", ConceptSequence = new List<GlyphConcept> { recentInputsSpan[0].Concept } });
                _glyphInputBuffer.MarkConsumed(recentInputsSpan.ToArray());
                return;
            }
            
            // Simple check: if two known glyphs are pressed, it's a plausible experiment (GDD B00.7)
            if (recentInputsSpan.Length >= 2 && recentInputsSpan.ToArray().All(f => f.KnowledgeState >= GlyphKnowledgeState.Known))
            {
                List<GlyphConcept> conceptSequence = recentInputsSpan.ToArray().Select(f => f.Concept).ToList();
                GD.Print($"MagicSystem: Player experimented with a plausible sequence: {string.Join(" -> ", conceptSequence)}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Plausible experiment! {string.Join(" ", conceptSequence)}", ConceptSequence = conceptSequence });
                _glyphInputBuffer.MarkConsumed(recentInputsSpan.ToArray());
                return;
            }

            // Fizzle logic is now primarily handled in Tick() if inputs expire, or if explicit invalid combo.
        }
    }
}
```

### 5. Integrating `ComboResolver` into `GameManager`

1.  Add `using Sigilborne.Systems.Magic;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `ComboResolver` property.
3.  Initialize `ComboResolver` in `InitializeSystems()` **before** `MagicSystem`.
4.  Pass `ComboResolver` to `MagicSystem`'s constructor.

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
    public MagicSystem Magic { get; private set; }
    public ComboResolver ComboResolver { get; private set; } // Add ComboResolver property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        
        // --- Test Glyph Discovery System ---
        GD.Print("\n--- Testing Glyph Discovery System ---");
        string testSymbol = GlyphMap.AllWorldGlyphs[0].SymbolID;
        string testSymbol2 = GlyphMap.AllWorldGlyphs[1].SymbolID;
        string testSymbol3 = GlyphMap.AllWorldGlyphs[2].SymbolID;
        string testSymbol4 = GlyphMap.AllWorldGlyphs[3].SymbolID; // New for multi-glyph combo
        string unknownSymbol = GlyphSymbols.AllSymbols.First(s => !GlyphMap.AllWorldGlyphs.Any(g => g.SymbolID == s));

        // Reset knowledge for acquisition tests
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol, GlyphKnowledgeState.Hidden, true);
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol2, GlyphKnowledgeState.Hidden, true);
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol3, GlyphKnowledgeState.Hidden, true);
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol4, GlyphKnowledgeState.Hidden, true); // Reset

        GD.Print($"Initial knowledge of '{testSymbol}': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol)}");
        // Let MagicSystem handle discovery for testSymbol via hotbar input
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol2, GlyphKnowledgeState.Known, true); // Still make testSymbol2 known for hotbar test
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol3, GlyphKnowledgeState.Known, true); // Make testSymbol3 known
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(testSymbol4, GlyphKnowledgeState.Known, true); // Make testSymbol4 known
        
        GD.Print($"Knowledge of '{testSymbol2}' after 'Known': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol2)}");
        GD.Print($"Knowledge of '{testSymbol3}' after 'Known': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol3)}");
        GD.Print($"Knowledge of '{testSymbol4}' after 'Known': {PlayerGlyphKnowledge.GetGlyphKnowledge(testSymbol4)}");
        
        GD.Print($"Attempt to update unknown symbol '{unknownSymbol}':");
        PlayerGlyphKnowledge.UpdateGlyphKnowledge(unknownSymbol, GlyphKnowledgeState.Known);
        
        GD.Print("--- End Testing Glyph Discovery System ---\n");

        // --- Test PlayerHotbarSystem ---
        GD.Print("\n--- Testing PlayerHotbarSystem ---");
        PlayerHotbar.AssignGlyphToSlot(0, testSymbol); // Assign potentially unknown symbol
        PlayerHotbar.AssignGlyphToSlot(1, testSymbol2); // Assign known symbol
        PlayerHotbar.AssignGlyphToSlot(2, testSymbol3); // Assign known symbol
        PlayerHotbar.AssignGlyphToSlot(3, testSymbol4); // Assign known symbol
        // ... (existing hotbar tests) ...
        GD.Print("--- End Testing PlayerHotbarSystem ---\n");

        // --- Test Glyph Acquisition System ---
        // ... (existing acquisition tests) ...
        GD.Print("--- End Testing Glyph Acquisition System ---\n");

        // --- Test Combo Resolver and Spell Definitions ---
        GD.Print("\n--- Testing Combo Resolver and Spell Definitions ---");
        // Register some dummy spells for testing the Trie
        // Spell 1: Bloom -> Consume
        List<GlyphConcept> spell1Sequence = new List<GlyphConcept> { GlyphMap.GetDefinitionBySymbol(testSymbol).Concept, GlyphMap.GetDefinitionBySymbol(testSymbol2).Concept };
        SpellDefinition spell1 = new SpellDefinition("Test_BloomConsume", spell1Sequence, 10f, 5f, 0.5f, 0.1f, 0.05f, false, false,
                                                    projectile: new SpellDefinition.ProjectileData { Speed = 200, Size = 10 });
        ComboResolver.RegisterSpell(spell1);

        // Spell 2: Bloom -> Consume -> Pulse (longer version)
        List<GlyphConcept> spell2Sequence = new List<GlyphConcept> { GlyphMap.GetDefinitionBySymbol(testSymbol).Concept, GlyphMap.GetDefinitionBySymbol(testSymbol2).Concept, GlyphMap.GetDefinitionBySymbol(testSymbol3).Concept };
        SpellDefinition spell2 = new SpellDefinition("Test_BloomConsumePulse", spell2Sequence, 25f, 15f, 1.0f, 0.3f, 0.1f, false, false,
                                                    explosion: new SpellDefinition.AoEData { Radius = 50 });
        ComboResolver.RegisterSpell(spell2);

        // Spell 3: Pulse -> Bloom
        List<GlyphConcept> spell3Sequence = new List<GlyphConcept> { GlyphMap.GetDefinitionBySymbol(testSymbol3).Concept, GlyphMap.GetDefinitionBySymbol(testSymbol).Concept };
        SpellDefinition spell3 = new SpellDefinition("Test_PulseBloom", spell3Sequence, 15f, 8f, 0.7f, 0.2f, 0.0f, false, false,
                                                    effects: new List<SpellDefinition.StatusEffectData> { new SpellDefinition.StatusEffectData { EffectID = "slow", Duration = 3f } });
        ComboResolver.RegisterSpell(spell3);
        
        GD.Print("\n--- Combo Resolver Trie Structure ---");
        ComboResolver.PrintTrie();
        GD.Print("-------------------------------------\n");
        
        GD.Print("--- End Testing Combo Resolver and Spell Definitions ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        // ... (existing _PhysicsProcess calls) ...
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to GlyphMap) ...
        
        PlayerGlyphKnowledge = new PlayerGlyphKnowledgeSystem(Entities.GetPlayerEntityID(), GlyphMap, Events);
        GD.Print("  - PlayerGlyphKnowledgeSystem initialized.");

        PlayerHotbar = new PlayerHotbarSystem(Entities.GetPlayerEntityID(), Events, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - PlayerHotbarSystem initialized.");

        // Initialize ComboResolver BEFORE MagicSystem
        ComboResolver = new ComboResolver(); // Initialize ComboResolver here
        GD.Print("  - ComboResolver initialized.");

        // Initialize MagicSystem, passing GameManager and ComboResolver
        Magic = new MagicSystem(Entities, Input, Events, PlayerHotbar, PlayerGlyphKnowledge, GlyphMap, this, ComboResolver); // Pass 'this' and ComboResolver
        GD.Print("  - MagicSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 6. Testing Combo Resolution

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for `ComboResolver` initialization and spell registration. The `PrintTrie()` output will show the structure.
5.  **Test 1 (Single Unknown Glyph)**: Press `0` (hotbar slot 0, `testSymbol`, initially `Hidden`).
    *   This will still trigger the glyph discovery logic: `MagicSystem: Player successfully experimented with 'symbol_leaf' and now KNOWS its concept: Bloom.`
6.  **Test 2 (Known Spell - `testSymbol` -> `testSymbol2`)**:
    *   Quickly press `0` (for `testSymbol`, which is `Bloom` in our seed)
    *   Then quickly press `1` (for `testSymbol2`, which is `Consume` in our seed)
    *   This sequence should match `Test_BloomConsume`.
    *   You should see: `MagicSystem: Resolved known spell: 'Test_BloomConsume'!`
    *   `MagicSystem: Combo fizzled...` should NOT appear for these resolved combos.
7.  **Test 3 (Longer Known Spell - `testSymbol` -> `testSymbol2` -> `testSymbol3`)**:
    *   Quickly press `0` (Bloom)
    *   Then `1` (Consume)
    *   Then `2` (Pulse)
    *   This sequence should match `Test_BloomConsumePulse`.
    *   You should see: `MagicSystem: Resolved known spell: 'Test_BloomConsumePulse'!` (The Trie correctly finds the *longest* match, even if a shorter prefix also matches a spell).
8.  **Test 4 (Fizzle)**:
    *   Press `0`. Wait for more than `MAX_COMBO_DELAY` (1.5 seconds).
    *   Press `0` again. You should see a `MagicSystem: Combo fizzled...` message.
9.  **Test 5 (Different Order - `testSymbol3` -> `testSymbol`)**:
    *   Quickly press `2` (Pulse)
    *   Then `0` (Bloom)
    *   This should match `Test_PulseBloom`.
    *   You should see: `MagicSystem: Resolved known spell: 'Test_PulseBloom'!`

This confirms the `ComboResolver` (Trie) is correctly identifying known spell sequences, respecting order, and finding the longest match.

### Summary

You have successfully implemented the **Combo Resolver** using a **Prefix Tree (Trie) structure**, efficiently detecting known spell sequences from the glyph input buffer. By defining `SpellDefinition` as data and integrating `ComboResolver` into `MagicSystem`, you've established a robust and flexible system for matching glyph input patterns, strictly adhering to TDD 02.2's specifications. This crucial component is the brain of Sigilborne's dynamic combo system, enabling complex spellcasting and emergent technique discovery.

### Next Steps

The next chapter will focus on **Spell Data Architecture**, expanding our `SpellDefinition` with more detailed data for projectiles, AoE, and status effects, and laying the groundwork for how these definitions will support `Mythic Evolution`.