## Chapter 4.3: Spell Data Architecture - Data-Driven Definitions (C#)

Our `ComboResolver` can now identify known `SpellDefinition`s. This chapter focuses on expanding the `SpellDefinition` itself to be truly data-driven, encompassing detailed parameters for projectiles, AoE effects, and status effects, as specified in TDD 02.3. This data-first approach is crucial for supporting thousands of unique spell variations, dynamic `Mythic Evolution` (C01), and `Forbidden Arts` (C04) without extensive code changes.

### 1. The Power of Data-Driven Spells

*   **Flexibility**: New spells can be created, modified, or evolved simply by changing data, not code.
*   **Scalability**: Supports a vast number of unique techniques, from simple glyphs to legendary combos.
*   **Moddability**: Easier for future content creators or modders to add new magic.
*   **GDD Alignment**: Directly supports the GDD's vision of emergent, world-specific techniques and mythic transformations.

### 2. Refining `SpellDefinition` with Component Data Structs

We briefly defined placeholder `ProjectileData`, `AoEData`, and `StatusEffectData` as nested classes in `SpellDefinition.cs`. While this works, for a truly ECS-lite approach, these should ideally be separate `structs` or `classes` that `SpellDefinition` references. This allows for cleaner data and potential reusability.

Let's move these into their own files in `res://_Brain/Systems/Magic/Components/` and refine their structure.

#### 2.1. `ProjectileData.cs`

1.  Create `res://_Brain/Systems/Magic/Components/ProjectileData.cs`:

```csharp
// _Brain/Systems/Magic/Components/ProjectileData.cs
using System;
using Godot; // For Vector2

namespace Sigilborne.Systems.Magic.Components
{
    /// <summary>
    /// Data for a spell that spawns a projectile.
    /// (TDD 02.3)
    /// </summary>
    public struct ProjectileData
    {
        public float Speed;         // Speed of the projectile
        public float Size;          // Visual size/radius for collision
        public int Pierce;          // Number of enemies it can pierce through
        public string VisualID;     // ID for the Body to know which visual to spawn (e.g., "fireball_vfx")
        public Vector2 Offset;      // Offset from caster's position

        public ProjectileData(float speed, float size, int pierce, string visualId, Vector2 offset)
        {
            Speed = speed;
            Size = size;
            Pierce = pierce;
            VisualID = visualId;
            Offset = offset;
        }

        public override string ToString() => $"Proj(Spd:{Speed},Sz:{Size},Vis:{VisualID})";
    }
}
```

#### 2.2. `AoEData.cs`

1.  Create `res://_Brain/Systems/Magic/Components/AoEData.cs`:

```csharp
// _Brain/Systems/Magic/Components/AoEData.cs
using System;

namespace Sigilborne.Systems.Magic.Components
{
    /// <summary>
    /// Data for a spell that creates an Area of Effect (AoE).
    /// (TDD 02.3)
    /// </summary>
    public struct AoEData
    {
        public float Radius;        // Radius of the AoE
        public float Falloff;       // How damage/effect diminishes with distance from center (0-1)
        public string VisualID;     // ID for the Body to know which visual to spawn (e.g., "explosion_vfx")
        public float Duration;      // How long the AoE persists (if not instantaneous)

        public AoEData(float radius, float falloff, string visualId, float duration = 0f)
        {
            Radius = radius;
            Falloff = falloff;
            VisualID = visualId;
            Duration = duration;
        }

        public override string ToString() => $"AoE(Rad:{Radius},Vis:{VisualID})";
    }
}
```

#### 2.3. `StatusEffectData.cs`

1.  Create `res://_Brain/Systems/Magic/Components/StatusEffectData.cs`:

```csharp
// _Brain/Systems/Magic/Components/StatusEffectData.cs
using System;

namespace Sigilborne.Systems.Magic.Components
{
    /// <summary>
    /// Data for a status effect applied by a spell.
    /// (TDD 02.3)
    /// </summary>
    public struct StatusEffectData
    {
        public string EffectID;     // Unique ID of the status effect (e.g., "burn_t1", "stun_short")
        public float Duration;      // How long the effect lasts
        public float Strength;      // Magnitude of the effect (e.g., damage per tick, slow amount)

        public StatusEffectData(string effectId, float duration, float strength)
        {
            EffectID = effectId;
            Duration = duration;
            Strength = strength;
        }

        public override string ToString() => $"Effect({EffectID},Dur:{Duration},Str:{Strength})";
    }
}
```

#### 2.4. Update `SpellDefinition.cs` to Reference New Structs

Now, modify `res://_Brain/Systems/Magic/SpellDefinition.cs` to use these new structs and remove the nested class definitions.

```csharp
// _Brain/Systems/Magic/SpellDefinition.cs
using System;
using System.Collections.Generic;
using System.Linq;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic.Components; // Add this using directive

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Represents the static data defining a single spell or combo technique.
    /// Spells are data-driven to allow for thousands of variations and mythic evolution.
    /// (TDD 02.3)
    /// </summary>
    public class SpellDefinition
    {
        public string ID { get; private set; }
        public List<GlyphConcept> Sequence { get; private set; }
        public float BaseDamage { get; private set; }
        public float ChakraCost { get; private set; }
        public float CastTime { get; private set; }
        public float StabilityCost { get; private set; }
        public float ResidueGeneration { get; private set; }
        public bool IsMythic { get; private set; }
        public bool IsForbidden { get; private set; }

        // Components (ECS-lite) - Now using our dedicated structs (TDD 02.3)
        // Use nullable types or default values if a spell doesn't always have these.
        public ProjectileData? Projectile { get; private set; } // Nullable struct
        public AoEData? Explosion { get; private set; } // Nullable struct
        public List<StatusEffectData> Effects { get; private set; }

        public SpellDefinition(string id, List<GlyphConcept> sequence, float baseDamage, float chakraCost, float castTime, float stabilityCost, float residueGeneration, bool isMythic, bool isForbidden, ProjectileData? projectile = null, AoEData? explosion = null, List<StatusEffectData> effects = null)
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
            string components = "";
            if (Projectile.HasValue) components += $" {Projectile.Value}";
            if (Explosion.HasValue) components += $" {Explosion.Value}";
            if (Effects.Any()) components += $" {string.Join(", ", Effects.Select(e => e.ToString()))}";

            return $"Spell: '{ID}' ({seq}) | Dmg: {BaseDamage}, Chakra: {ChakraCost}, StabCost: {StabilityCost}{components}";
        }
    }
}
```

### 3. Populating `SpellDefinition`s with Procedural Subtype Modifiers

Now that `WorldGlyphDefinition` has subtypes and modifiers, and `SpellDefinition` has component data, we can leverage this to create more dynamic spells.

When we register a spell, its `BaseDamage`, `ChakraCost`, etc., and the parameters for `ProjectileData`, `AoEData`, `StatusEffectData` can be influenced by the `WorldGlyphDefinition`s that make up its sequence. This is the "emergent behavior" the GDD (B02.1) talks about.

Let's modify `GameManager._Ready()` to register spells that incorporate these modifiers.

Open `res://_Brain/Core/GameManager.cs` and modify the "Test Combo Resolver and Spell Definitions" section:

```csharp
// _Brain/Core/GameManager.cs (inside _Ready method)
// ...
        // --- Test Combo Resolver and Spell Definitions ---
        GD.Print("\n--- Testing Combo Resolver and Spell Definitions ---");
        
        // Get WorldGlyphDefinitions for our test symbols
        WorldGlyphDefinition def0 = GlyphMap.GetDefinitionBySymbol(testSymbol); // Bloom
        WorldGlyphDefinition def1 = GlyphMap.GetDefinitionBySymbol(testSymbol2); // Consume
        WorldGlyphDefinition def2 = GlyphMap.GetDefinitionBySymbol(testSymbol3); // Pulse
        WorldGlyphDefinition def3 = GlyphMap.GetDefinitionBySymbol(testSymbol4); // e.g., Bind

        // Helper to get modifier value safely
        Func<WorldGlyphDefinition, GlyphModifierType, float, float> getMod = (def, type, defaultValue) => 
            def.Modifiers.TryGetValue(type, out float val) ? val : defaultValue;

        // --- Spell 1: Bloom -> Consume (Example: Toxic Projectile) ---
        // Base stats for the spell
        float baseDmg1 = 10f * getMod(def0, GlyphModifierType.DamageMultiplier, 1.0f) * getMod(def1, GlyphModifierType.DamageMultiplier, 1.0f);
        float chakraCost1 = 5f + getMod(def0, GlyphModifierType.StabilityCost, 0f) + getMod(def1, GlyphModifierType.StabilityCost, 0f);
        float stabilityCost1 = 0.1f + getMod(def0, GlyphModifierType.ResidueGeneration, 0f) + getMod(def1, GlyphModifierType.ResidueGeneration, 0f);

        // Projectile data influenced by glyphs
        ProjectileData projData1 = new ProjectileData(
            speed: 200f * getMod(def0, GlyphModifierType.Speed, 1.0f),
            size: 10f * getMod(def1, GlyphModifierType.Radius, 1.0f), // Consume might make it smaller/denser
            pierce: 1,
            visualId: "projectile_basic",
            offset: Vector2.Zero
        );
        List<StatusEffectData> effects1 = new List<StatusEffectData>();
        if (def0.Subtype == GlyphSubtype.ToxicBloom) effects1.Add(new StatusEffectData("poison_t1", getMod(def0, GlyphModifierType.Duration, 3f), getMod(def0, GlyphModifierType.DamageMultiplier, 1f)));

        List<GlyphConcept> spell1Sequence = new List<GlyphConcept> { def0.Concept, def1.Concept };
        SpellDefinition spell1 = new SpellDefinition("Test_BloomConsume", spell1Sequence, baseDmg1, chakraCost1, 0.5f, stabilityCost1, 0.05f, false, false,
                                                    projectile: projData1,
                                                    effects: effects1);
        ComboResolver.RegisterSpell(spell1);

        // --- Spell 2: Bloom -> Consume -> Pulse (Example: Healing AoE) ---
        float baseHeal2 = 15f * getMod(def0, GlyphModifierType.HealingAmount, 1.0f) + getMod(def2, GlyphModifierType.HealingAmount, 0f); // Pulse could add healing
        float chakraCost2 = 15f + getMod(def0, GlyphModifierType.StabilityCost, 0f) + getMod(def1, GlyphModifierType.StabilityCost, 0f) + getMod(def2, GlyphModifierType.StabilityCost, 0f);
        float stabilityCost2 = 0.3f + getMod(def0, GlyphModifierType.ResidueGeneration, 0f) + getMod(def1, GlyphModifierType.ResidueGeneration, 0f) + getMod(def2, GlyphModifierType.ResidueGeneration, 0f);

        AoEData aoeData2 = new AoEData(
            radius: 50f * getMod(def0, GlyphModifierType.Radius, 1.0f) * getMod(def2, GlyphModifierType.Radius, 1.0f),
            falloff: 0.5f,
            visualId: "aoe_healing_pulse",
            duration: getMod(def0, GlyphModifierType.Duration, 0f) + getMod(def2, GlyphModifierType.Duration, 0f) // If pulse has duration
        );
        List<StatusEffectData> effects2 = new List<StatusEffectData>();
        if (def0.Subtype == GlyphSubtype.HealingGrowth) effects2.Add(new StatusEffectData("regen_t1", aoeData2.Duration, getMod(def0, GlyphModifierType.HealingAmount, 1f)));

        List<GlyphConcept> spell2Sequence = new List<GlyphConcept> { def0.Concept, def1.Concept, def2.Concept };
        SpellDefinition spell2 = new SpellDefinition("Test_BloomConsumePulse", spell2Sequence, baseHeal2, chakraCost2, 1.0f, stabilityCost2, 0.1f, false, false,
                                                    explosion: aoeData2, // AoE for healing
                                                    effects: effects2);
        ComboResolver.RegisterSpell(spell2);

        // --- Spell 3: Pulse -> Bloom (Example: Stun/Slow) ---
        float baseDmg3 = 15f * getMod(def2, GlyphModifierType.DamageMultiplier, 1.0f) * getMod(def0, GlyphModifierType.DamageMultiplier, 1.0f);
        float chakraCost3 = 8f + getMod(def2, GlyphModifierType.StabilityCost, 0f) + getMod(def0, GlyphModifierType.StabilityCost, 0f);
        float stabilityCost3 = 0.2f + getMod(def2, GlyphModifierType.ResidueGeneration, 0f) + getMod(def0, GlyphModifierType.ResidueGeneration, 0f);

        List<StatusEffectData> effects3 = new List<StatusEffectData> { new StatusEffectData("stun_short", 1.0f, 1.0f) };
        if (def2.Subtype == GlyphSubtype.Shockwave) effects3.Add(new StatusEffectData("slow_t1", getMod(def2, GlyphModifierType.Duration, 3f), getMod(def2, GlyphModifierType.Speed, 1f)));

        List<GlyphConcept> spell3Sequence = new List<GlyphConcept> { def2.Concept, def0.Concept };
        SpellDefinition spell3 = new SpellDefinition("Test_PulseBloom", spell3Sequence, baseDmg3, chakraCost3, 0.7f, stabilityCost3, 0.0f, false, false,
                                                    effects: effects3);
        ComboResolver.RegisterSpell(spell3);
        
        GD.Print("\n--- Combo Resolver Trie Structure ---");
        ComboResolver.PrintTrie();
        GD.Print("-------------------------------------\n");
        
        GD.Print("--- End Testing Combo Resolver and Spell Definitions ---\n");
// ...
```

**Key Changes in `GameManager._Ready()`:**

*   **`getMod` Helper**: A lambda function `getMod` is used to safely retrieve modifier values from `WorldGlyphDefinition.Modifiers`, providing a default if the modifier isn't present.
*   **Dynamic Spell Stats**: `baseDmg`, `chakraCost`, `stabilityCost`, and the parameters for `ProjectileData`, `AoEData`, and `StatusEffectData` are now calculated by multiplying or adding values from the `WorldGlyphDefinition`'s `Modifiers`. This makes each spell's numerical properties unique to the current world's glyph definitions.
*   **`ProjectileData`, `AoEData`, `StatusEffectData` Instantiation**: Now correctly uses the new struct constructors.

### 4. Testing Data-Driven Spells

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for "Testing Combo Resolver and Spell Definitions".
    *   The `Spell: 'ID' (...)` string representation will now include the details of the `Projectile`, `AoE`, and `StatusEffect` data.
    *   Crucially, the `Dmg`, `Chakra`, `StabCost`, and parameters within the component data (e.g., `Proj(Spd:X,Sz:Y)`) will reflect the values influenced by the `GlyphModifierType`s of the specific `WorldGlyphDefinition`s in your current world seed. These numbers will change if you change the `worldSeed` in `GameManager.InitializeSystems()`.

This output demonstrates that your `SpellDefinition`s are truly data-driven, with their properties dynamically influenced by the world's unique glyph mappings and their associated subtypes and modifiers.

### Summary

You have successfully expanded Sigilborne's `SpellDefinition` to be truly **data-driven**, incorporating detailed parameters for projectiles, AoE effects, and status effects using dedicated structs. By leveraging the procedural `GlyphSubtype`s and `Modifiers` from `WorldGlyphMap` to dynamically calculate spell properties during registration, you've established a powerful mechanism for creating thousands of unique technique variations. This crucial step strictly adheres to TDD 02.3's specifications, laying the groundwork for `Mythic Evolution` and `Forbidden Arts` through flexible, data-centric spell definitions.

### Next Steps

The next chapter will focus on the **Casting State Machine**, implementing the player's casting flow (Idle, Channeling, Casting, Recovery) in the C# Brain, which will govern when spells can be input and executed, and how chakra costs and stability are managed.