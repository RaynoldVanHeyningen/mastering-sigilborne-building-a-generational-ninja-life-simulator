## Chapter 4.4: Casting State Machine - Player Casting Flow (C#)

Our `ComboResolver` can now identify a `SpellDefinition` from player input. However, casting a spell isn't instantaneous; it involves a sequence of states: preparing the glyphs, channeling energy, and then executing the technique. This chapter implements a **Casting State Machine** in the C# Brain to manage the player's casting flow (Idle, Channeling, Casting, Recovery), governing when spells can be input, executed, and how chakra costs and stability are managed, as specified in TDD 02.4.

### 1. The Importance of a Casting State Machine

The GDD (B05.6) describes dynamic timing windows, and (B03.2) details chakra strain. A state machine provides:

*   **Clear State Management**: Precisely define what the player can do in each phase of casting.
*   **Timing Control**: Enforce `CastTime` and `Recovery` periods.
*   **Resource Management**: Deduct `ChakraCost` and `StabilityCost` at the appropriate time.
*   **Feedback**: Emit events for the Body (UI, animations) to react to casting states.
*   **Interruptibility**: Define when a cast can be interrupted (covered later).

### 2. Defining `CastState` and `PlayerCastingState`

We need an enum for the casting states and a component-like struct to hold the player's current casting data.

1.  Create `res://_Brain/Systems/Magic/PlayerCastingState.cs`:

```csharp
// _Brain/Systems/Magic/PlayerCastingState.cs
using System;
using Godot; // For Vector2 if needed

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Defines the possible states of a player's spellcasting flow.
    /// (TDD 02.4)
    /// </summary>
    public enum CastState
    {
        Idle,           // Not casting, ready for new input.
        Channeling,     // Glyph sequence started, waiting for more inputs within combo window.
        CastStart,      // Sequence complete, spell identified, resources deducted, preparing to execute.
        Casting,        // Spell is actively being cast (e.g., projectile traveling, AoE active).
        Recovery,       // Post-cast delay, player is briefly unable to act.
        Interrupted     // Cast was interrupted (e.g., by damage, status effect).
    }

    /// <summary>
    /// Stores the player's current casting state and related data.
    /// This is a component-like data structure managed by the CastingSystem.
    /// </summary>
    public struct PlayerCastingState
    {
        public CastState CurrentState;
        public SpellDefinition CurrentSpell; // The spell being cast.
        public double StateTimer;           // How long we've been in the current state.
        public EntityID CasterID;           // The entity performing the cast.

        public PlayerCastingState(EntityID casterID)
        {
            CasterID = casterID;
            CurrentState = CastState.Idle;
            CurrentSpell = null;
            StateTimer = 0;
        }

        public override string ToString()
        {
            return $"State: {CurrentState}, Spell: {CurrentSpell?.ID ?? "None"}, Timer: {StateTimer:F2}";
        }
    }
}
```

### 3. Implementing `CastingSystem.cs`

This system will manage the `PlayerCastingState` struct, process state transitions, deduct resources, and interact with other systems.

1.  Create `res://_Brain/Systems/Magic/CastingSystem.cs`:

```csharp
// _Brain/Systems/Magic/CastingSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology; // For PlayerStatSystem
using Sigilborne.Systems.Magic.Components; // For ProjectileData, AoEData etc.

namespace Sigilborne.Systems.Magic
{
    /// <summary>
    /// Manages the player's spellcasting state machine and executes spell effects.
    /// (TDD 02.4)
    /// </summary>
    public class CastingSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private PlayerStatSystem _playerStatSystem; // To deduct chakra/stability
        private TransformSystem _transformSystem;   // To get caster position for effects

        private EntityID _playerEntityID;
        private PlayerCastingState _playerCastingState; // The player's casting state machine

        // Recovery time after a spell cast (GDD B05.5)
        private const float DEFAULT_RECOVERY_TIME = 0.3f; // Small default recovery

        public CastingSystem(EntityManager entityManager, EventBus eventBus, PlayerStatSystem playerStatSystem, TransformSystem transformSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _playerStatSystem = playerStatSystem;
            _transformSystem = transformSystem;

            _playerEntityID = _entityManager.GetPlayerEntityID();
            _playerCastingState = new PlayerCastingState(_playerEntityID); // Initialize casting state

            GD.Print("CastingSystem: Initialized.");
        }

        /// <summary>
        /// Main update loop for the CastingSystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// (TDD 02.4: Update(double delta) logic)
        /// </summary>
        public void Tick(double delta)
        {
            _playerCastingState.StateTimer += delta; // Increment timer for current state

            switch (_playerCastingState.CurrentState)
            {
                case CastState.Idle:
                    // Do nothing, waiting for MagicSystem to initiate Channeling
                    break;

                case CastState.Channeling:
                    // Channeling is managed by MagicSystem (combo delay window)
                    // If MagicSystem doesn't transition to CastStart, it will fizzle.
                    break;

                case CastState.CastStart:
                    // Resources deducted, now waiting for CastTime to elapse
                    if (_playerCastingState.StateTimer >= _playerCastingState.CurrentSpell.CastTime)
                    {
                        ExecuteSpellEffect(_playerCastingState.CurrentSpell);
                        TransitionToState(CastState.Recovery, DEFAULT_RECOVERY_TIME); // Enter recovery after execution
                    }
                    break;

                case CastState.Casting:
                    // For instant spells, this state might be very short or skipped.
                    // For sustained spells (e.g., beam), this state would persist.
                    // For now, most spells are instant or have effects triggered in CastStart.
                    break;

                case CastState.Recovery:
                    // TDD 02.4: The "End Lag". Player cannot move/act.
                    if (_playerCastingState.StateTimer >= DEFAULT_RECOVERY_TIME)
                    {
                        TransitionToState(CastState.Idle); // Back to idle after recovery
                    }
                    break;

                case CastState.Interrupted:
                    // Stays in interrupted state until reset (e.g., player input, timer).
                    // For now, it will transition to Idle after a short delay.
                    if (_playerCastingState.StateTimer >= 0.5f) // Short interruption recovery
                    {
                        TransitionToState(CastState.Idle);
                    }
                    break;
            }
        }

        /// <summary>
        /// Initiates a spell cast. Called by MagicSystem when a combo is resolved.
        /// </summary>
        /// <param name="spell">The resolved spell to cast.</param>
        /// <returns>True if cast initiated, false if resources insufficient or not in Idle state.</returns>
        public bool InitiateCast(SpellDefinition spell)
        {
            if (_playerCastingState.CurrentState != CastState.Idle)
            {
                GD.Print($"CastingSystem: Cannot cast spell '{spell.ID}', not in Idle state ({_playerCastingState.CurrentState}).");
                return false;
            }

            PlayerStats currentStats = _playerStatSystem.GetPlayerStats();
            if (currentStats.Chakra < spell.ChakraCost)
            {
                GD.Print($"CastingSystem: Insufficient Chakra to cast '{spell.ID}'. (Need: {spell.ChakraCost}, Have: {currentStats.Chakra})");
                // Emit event for UI feedback (e.g., "Not enough Chakra")
                _eventBus.Publish(new CastFailedEvent { PlayerID = _playerEntityID, Reason = "Insufficient Chakra" });
                return false;
            }
            // Add stability check here (GDD B03.2)
            // If currentStats.Stability < spell.StabilityCost... fail or miscast.

            // Deduct resources immediately (TDD 02.4: Mana deducted)
            _playerStatSystem.TakeChakra(spell.ChakraCost); // We'll add TakeChakra to PlayerStatSystem
            // Deduct stability (GDD B03.2)
            // _playerStatSystem.TakeStability(spell.StabilityCost);

            _playerCastingState.CurrentSpell = spell;
            TransitionToState(CastState.CastStart, spell.CastTime); // Transition to CastStart, timer is CastTime

            GD.Print($"CastingSystem: Initiated cast for '{spell.ID}'. Chakra cost: {spell.ChakraCost}.");
            _eventBus.Publish(new CastStateChangedEvent { PlayerID = _playerEntityID, NewState = CastState.CastStart, SpellID = spell.ID });

            return true;
        }

        /// <summary>
        /// Transitions the casting state machine to a new state.
        /// </summary>
        private void TransitionToState(CastState newState, double timerDuration = 0)
        {
            _playerCastingState.CurrentState = newState;
            _playerCastingState.StateTimer = 0; // Reset timer for the new state
            GD.Print($"CastingSystem: Player {_playerEntityID} transitioned to state: {newState}.");
            _eventBus.Publish(new CastStateChangedEvent { PlayerID = _playerEntityID, NewState = newState, SpellID = _playerCastingState.CurrentSpell?.ID });
        }

        /// <summary>
        /// Executes the actual effects of the spell.
        /// (TDD 02.4: Casting: The "Active Frames". Projectile spawns or effect applies.)
        /// </summary>
        private void ExecuteSpellEffect(SpellDefinition spell)
        {
            GD.Print($"CastingSystem: Executing effect for spell '{spell.ID}'!");
            _eventBus.Publish(new SpellEffectExecutedEvent { PlayerID = _playerEntityID, Spell = spell });

            // --- Placeholder for actual effect logic ---
            // In later chapters, this would interact with other systems:
            // - Spawn projectile entities (ProjectileSystem)
            // - Apply AoE effects (AoESystem)
            // - Apply status effects (StatusEffectSystem)
            // - Play specific VFX/Audio (Body via EventBus)

            // For now, let's just use the TransformSystem to get the caster's position for context.
            if (_transformSystem.TryGetTransform(_playerEntityID, out TransformComponent casterTransform))
            {
                GD.Print($"  Caster Position: {casterTransform.Position}");
                if (spell.Projectile.HasValue)
                {
                    GD.Print($"  Spawn Projectile: {spell.Projectile.Value}");
                    // _eventBus.Publish(new SpawnProjectileEvent { CasterID = _playerEntityID, ProjectileData = spell.Projectile.Value, SpawnPosition = casterTransform.Position });
                }
                if (spell.Explosion.HasValue)
                {
                    GD.Print($"  Apply AoE: {spell.Explosion.Value}");
                    // _eventBus.Publish(new ApplyAoEEvent { CasterID = _playerEntityID, AoEData = spell.Explosion.Value, CenterPosition = casterTransform.Position });
                }
                if (spell.Effects.Any())
                {
                    GD.Print($"  Apply Effects: {string.Join(", ", spell.Effects)}");
                    // _eventBus.Publish(new ApplyStatusEffectsEvent { CasterID = _playerEntityID, TargetID = ..., Effects = spell.Effects });
                }
            }
        }

        /// <summary>
        /// Gets the player's current casting state.
        /// </summary>
        public PlayerCastingState GetPlayerCastingState()
        {
            return _playerCastingState;
        }

        // --- Helper Events for Body Sync ---
        public struct CastStateChangedEvent { public EntityID PlayerID; public CastState NewState; public string SpellID; }
        public struct SpellEffectExecutedEvent { public EntityID PlayerID; public SpellDefinition Spell; }
        public struct CastFailedEvent { public EntityID PlayerID; public string Reason; }
    }
}
```

#### 3.1. Update `PlayerStatSystem.cs` with `TakeChakra`

The `CastingSystem` needs to deduct chakra.

Open `res://_Brain/Systems/Biology/PlayerStatSystem.cs` and add this method:

```csharp
// _Brain/Systems/Biology/PlayerStatSystem.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;

namespace Sigilborne.Systems.Biology
{
    // ... (PlayerStats struct) ...

    public class PlayerStatSystem
    {
        // ... (existing fields and constructor) ...

        public PlayerStats GetPlayerStats() { return _playerStats; }

        public void TakeDamage(float amount) { /* ... */ }

        /// <summary>
        /// Authoritatively deducts chakra from the player.
        /// </summary>
        public void TakeChakra(float amount)
        {
            if (amount < 0) return;

            _playerStats.Chakra -= amount;
            if (_playerStats.Chakra < 0) _playerStats.Chakra = 0;

            GD.Print($"PlayerStatSystem: Player {_playerStats.PlayerID} used {amount} chakra. New Chakra: {_playerStats.Chakra}");

            // Publish an event for the Body (UI) to update chakra display
            _eventBus.Publish(new PlayerChakraChangedEvent { PlayerID = _playerStats.PlayerID, NewChakra = _playerStats.Chakra, MaxChakra = _playerStats.MaxChakra });
        }

        // --- Helper Events for Body Sync ---
        public struct PlayerHealthChangedEvent { public EntityID PlayerID; public float NewHealth; public float MaxHealth; }
        public struct PlayerDiedEvent { public EntityID PlayerID; }
        public struct PlayerChakraChangedEvent { public EntityID PlayerID; public float NewChakra; public float MaxChakra; } // New event
    }
}
```

### 4. Integrating `CastingSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Magic;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `CastingSystem` property.
3.  Initialize `CastingSystem` in `InitializeSystems()` **before** `MagicSystem`.
4.  Call `CastingSystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).
5.  Pass `CastingSystem` to `MagicSystem`'s constructor.

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
using Sigilborne.Systems.Magic.Components; // Needed for ProjectileData etc.

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public ComboResolver ComboResolver { get; private set; }
    public CastingSystem Casting { get; private set; } // Add CastingSystem property
    public MagicSystem Magic { get; private set; }

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        
        // --- Test PlayerStatSystem (Damage) ---
        // ... (existing test damage code) ...
        // --- End Testing PlayerStatSystem ---
        
        // --- Test Glyph Discovery System ---
        // ... (existing glyph discovery tests) ...
        
        // --- Test PlayerHotbarSystem ---
        // ... (existing hotbar tests) ...

        // --- Test Glyph Acquisition System ---
        // ... (existing acquisition tests) ...

        // --- Test Combo Resolver and Spell Definitions ---
        GD.Print("\n--- Testing Combo Resolver and Spell Definitions ---");
        
        WorldGlyphDefinition def0 = GlyphMap.GetDefinitionBySymbol(GlyphMap.AllWorldGlyphs[0].SymbolID); // Bloom
        WorldGlyphDefinition def1 = GlyphMap.GetDefinitionBySymbol(GlyphMap.AllWorldGlyphs[1].SymbolID); // Consume
        WorldGlyphDefinition def2 = GlyphMap.GetDefinitionBySymbol(GlyphMap.AllWorldGlyphs[2].SymbolID); // Pulse
        WorldGlyphDefinition def3 = GlyphMap.GetDefinitionBySymbol(GlyphMap.AllWorldGlyphs[3].SymbolID); // e.g., Bind

        Func<WorldGlyphDefinition, GlyphModifierType, float, float> getMod = (def, type, defaultValue) => 
            def.Modifiers.TryGetValue(type, out float val) ? val : defaultValue;

        // Spell 1: Bloom -> Consume (Example: Toxic Projectile)
        float baseDmg1 = 10f * getMod(def0, GlyphModifierType.DamageMultiplier, 1.0f) * getMod(def1, GlyphModifierType.DamageMultiplier, 1.0f);
        float chakraCost1 = 5f + getMod(def0, GlyphModifierType.StabilityCost, 0f) + getMod(def1, GlyphModifierType.StabilityCost, 0f);
        float stabilityCost1 = 0.1f + getMod(def0, GlyphModifierType.ResidueGeneration, 0f) + getMod(def1, GlyphModifierType.ResidueGeneration, 0f);
        ProjectileData projData1 = new ProjectileData(speed: 200f * getMod(def0, GlyphModifierType.Speed, 1.0f), size: 10f * getMod(def1, GlyphModifierType.Radius, 1.0f), pierce: 1, visualId: "projectile_basic", offset: Vector2.Zero);
        List<StatusEffectData> effects1 = new List<StatusEffectData>();
        if (def0.Subtype == GlyphSubtype.ToxicBloom) effects1.Add(new StatusEffectData("poison_t1", getMod(def0, GlyphModifierType.Duration, 3f), getMod(def0, GlyphModifierType.DamageMultiplier, 1f)));
        List<GlyphConcept> spell1Sequence = new List<GlyphConcept> { def0.Concept, def1.Concept };
        SpellDefinition spell1 = new SpellDefinition("Test_BloomConsume", spell1Sequence, baseDmg1, chakraCost1, 0.5f, stabilityCost1, 0.05f, false, false,
                                                    projectile: projData1, effects: effects1);
        ComboResolver.RegisterSpell(spell1);

        // Spell 2: Bloom -> Consume -> Pulse (Example: Healing AoE)
        float baseHeal2 = 15f * getMod(def0, GlyphModifierType.HealingAmount, 1.0f) + getMod(def2, GlyphModifierType.HealingAmount, 0f);
        float chakraCost2 = 15f + getMod(def0, GlyphModifierType.StabilityCost, 0f) + getMod(def1, GlyphModifierType.StabilityCost, 0f) + getMod(def2, GlyphModifierType.StabilityCost, 0f);
        float stabilityCost2 = 0.3f + getMod(def0, GlyphModifierType.ResidueGeneration, 0f) + getMod(def1, GlyphModifierType.ResidueGeneration, 0f) + getMod(def2, GlyphModifierType.ResidueGeneration, 0f);
        AoEData aoeData2 = new AoEData(radius: 50f * getMod(def0, GlyphModifierType.Radius, 1.0f) * getMod(def2, GlyphModifierType.Radius, 1.0f), falloff: 0.5f, visualId: "aoe_healing_pulse", duration: getMod(def0, GlyphModifierType.Duration, 0f) + getMod(def2, GlyphModifierType.Duration, 0f));
        List<StatusEffectData> effects2 = new List<StatusEffectData>();
        if (def0.Subtype == GlyphSubtype.HealingGrowth) effects2.Add(new StatusEffectData("regen_t1", aoeData2.Duration, getMod(def0, GlyphModifierType.HealingAmount, 1f)));
        List<GlyphConcept> spell2Sequence = new List<GlyphConcept> { def0.Concept, def1.Concept, def2.Concept };
        SpellDefinition spell2 = new SpellDefinition("Test_BloomConsumePulse", spell2Sequence, baseHeal2, chakraCost2, 1.0f, stabilityCost2, 0.1f, false, false,
                                                    explosion: aoeData2, effects: effects2);
        ComboResolver.RegisterSpell(spell2);

        // Spell 3: Pulse -> Bloom (Example: Stun/Slow)
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
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Casting.Tick(delta); // Call CastingSystem's tick method
        Magic.Tick(delta); // MagicSystem is called after CastingSystem to allow it to initiate casts
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to PlayerStatSystem) ...

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

        PlayerGlyphKnowledge = new PlayerGlyphKnowledgeSystem(Entities.GetPlayerEntityID(), GlyphMap, Events);
        GD.Print("  - PlayerGlyphKnowledgeSystem initialized.");

        PlayerHotbar = new PlayerHotbarSystem(Entities.GetPlayerEntityID(), Events, PlayerGlyphKnowledge, GlyphMap);
        GD.Print("  - PlayerHotbarSystem initialized.");

        ComboResolver = new ComboResolver();
        GD.Print("  - ComboResolver initialized.");

        // Initialize CastingSystem BEFORE MagicSystem
        Casting = new CastingSystem(Entities, Events, PlayerStats, Transforms); // Initialize CastingSystem here
        GD.Print("  - CastingSystem initialized.");

        // Initialize MagicSystem, passing CastingSystem
        Magic = new MagicSystem(Entities, Input, Events, PlayerHotbar, PlayerGlyphKnowledge, GlyphMap, this, ComboResolver, Casting); // Pass CastingSystem
        GD.Print("  - MagicSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `MagicSystem.cs` to Pass `CastingSystem`

Open `res://_Brain/Systems/Magic/MagicSystem.cs` and modify its constructor and `ResolveCombo` to interact with `CastingSystem`.

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
    public class MagicSystem
    {
        // ... (existing fields) ...
        private CastingSystem _castingSystem; // New: Reference to CastingSystem

        public MagicSystem(EntityManager entityManager, InputSystem inputSystem, EventBus eventBus,
                           PlayerHotbarSystem playerHotbar, PlayerGlyphKnowledgeSystem playerGlyphKnowledge,
                           WorldGlyphMap worldGlyphMap, GameManager gameManager, ComboResolver comboResolver,
                           CastingSystem castingSystem) // Add CastingSystem parameter
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
            _comboResolver = comboResolver;
            _castingSystem = castingSystem; // Store CastingSystem reference
            GD.Print("MagicSystem: Initialized.");
        }

        // ... (Tick, ProcessGlyphInput methods) ...

        private void ResolveCombo()
        {
            ReadOnlySpan<GlyphInputFrame> recentInputsSpan = _glyphInputBuffer.GetRecentUnconsumed(_gameManager.Time.CurrentGameTime - MAX_COMBO_DELAY);
            
            if (recentInputsSpan.IsEmpty) return;

            // Check if player is currently in a state that prevents new casts (TDD 02.4)
            if (_castingSystem.GetPlayerCastingState().CurrentState != CastState.Idle)
            {
                GD.Print($"MagicSystem: Player is not Idle ({_castingSystem.GetPlayerCastingState().CurrentState}), cannot initiate new cast.");
                return; // Cannot initiate a new cast if not idle
            }

            GD.Print($"MagicSystem: Attempting to resolve combo with {recentInputsSpan.Length} recent inputs.");
            
            SpellDefinition resolvedSpell = _comboResolver.ResolveCombo(recentInputsSpan);

            if (resolvedSpell != null)
            {
                // Found a known spell! Now initiate the cast via CastingSystem.
                if (_castingSystem.InitiateCast(resolvedSpell))
                {
                    GD.Print($"MagicSystem: Resolved and initiated cast for known spell: '{resolvedSpell.ID}'!");
                    _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = false, ResultText = $"Cast {resolvedSpell.ID}!", ConceptSequence = resolvedSpell.Sequence });
                    _glyphInputBuffer.MarkConsumed(recentInputsSpan.ToArray());
                }
                else
                {
                    GD.Print($"MagicSystem: Failed to initiate cast for '{resolvedSpell.ID}' (e.g., insufficient resources).");
                    // CastingSystem already published a CastFailedEvent.
                }
                return;
            }
            // ... (existing placeholder resolution logic) ...
            // Simplified logic for single unknown glyph discovery
            if (recentInputsSpan.Length == 1 && recentInputsSpan[0].KnowledgeState < GlyphKnowledgeState.Known)
            {
                _playerGlyphKnowledge.UpdateGlyphKnowledge(recentInputsSpan[0].SymbolID, GlyphKnowledgeState.Known);
                GD.Print($"MagicSystem: Player successfully experimented with '{recentInputsSpan[0].SymbolID}' and now KNOWS its concept: {recentInputsSpan[0].Concept}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Discovered {recentInputsSpan[0].Concept}!", ConceptSequence = new List<GlyphConcept> { recentInputsSpan[0].Concept } });
                _glyphInputBuffer.MarkConsumed(recentInputsSpan.ToArray());
                return;
            }
            
            // Plausible experiment for 2+ known glyphs
            if (recentInputsSpan.Length >= 2 && recentInputsSpan.ToArray().All(f => f.KnowledgeState >= GlyphKnowledgeState.Known))
            {
                List<GlyphConcept> conceptSequence = recentInputsSpan.ToArray().Select(f => f.Concept).ToList();
                GD.Print($"MagicSystem: Player experimented with a plausible sequence: {string.Join(" -> ", conceptSequence)}.");
                _eventBus.Publish(new ComboResolvedEvent { PlayerID = _playerEntityID, IsSuccess = true, IsDiscovery = true, ResultText = $"Plausible experiment! {string.Join(" ", conceptSequence)}", ConceptSequence = conceptSequence });
                _glyphInputBuffer.MarkConsumed(recentInputsSpan.ToArray());
                return;
            }
        }
        // ... (Helper Events) ...
    }
}
```

#### 4.2. Update `EventBus.cs` for CastingSystem Events

Our `CastingSystem` publishes `CastStateChangedEvent`, `SpellEffectExecutedEvent`, and `CastFailedEvent`. Our `PlayerStatSystem` also now publishes `PlayerChakraChangedEvent`. We need to define these `Action` delegates in `EventBus`.

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

        // Player Stat Events (TDD 01.4) - updated with Chakra
        public event Action<EntityID, float, float> OnPlayerHealthChanged;
        public event Action<EntityID> OnPlayerDied;
        public event Action<EntityID, float, float> OnPlayerChakraChanged; // New event

        // Casting System Events (TDD 02.4)
        public event Action<EntityID, CastState, string> OnCastStateChanged; // PlayerID, NewState, SpellID
        public event Action<EntityID, SpellDefinition> OnSpellEffectExecuted; // PlayerID, Spell
        public event Action<EntityID, string> OnCastFailed; // PlayerID, Reason

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is PlayerStatSystem.PlayerChakraChangedEvent chakraEvent) // New condition
            {
                OnPlayerChakraChanged?.Invoke(chakraEvent.PlayerID, chakraEvent.NewChakra, chakraEvent.MaxChakra);
            }
            else if (eventData is CastingSystem.CastStateChangedEvent castStateEvent) // New condition
            {
                OnCastStateChanged?.Invoke(castStateEvent.PlayerID, castStateEvent.NewState, castStateEvent.SpellID);
            }
            else if (eventData is CastingSystem.SpellEffectExecutedEvent spellEffectEvent) // New condition
            {
                OnSpellEffectExecuted?.Invoke(spellEffectEvent.PlayerID, spellEffectEvent.Spell);
            }
            else if (eventData is CastingSystem.CastFailedEvent castFailedEvent) // New condition
            {
                OnCastFailed?.Invoke(castFailedEvent.PlayerID, castFailedEvent.Reason);
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

### 5. Testing the Casting State Machine

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  The player should have 50 Chakra initially.
5.  **Test 1 (Known Spell - `testSymbol` -> `testSymbol2`)**:
    *   Quickly press `0` (for `testSymbol`, which is `Bloom`)
    *   Then quickly press `1` (for `testSymbol2`, which is `Consume`)
    *   This sequence should match `Test_BloomConsume` (Chakra Cost: `chakraCost1` from `GameManager._Ready()`, e.g., 5).
    *   You should see:
        *   `MagicSystem: Resolved and initiated cast for known spell: 'Test_BloomConsume'!`
        *   `CastingSystem: Player EntityID(0, Gen:1) transitioned to state: CastStart.`
        *   `PlayerStatSystem: Player EntityID(0, Gen:1) used X chakra. New Chakra: Y` (Chakra deducted)
        *   After `CastTime` (0.5s): `CastingSystem: Executing effect for spell 'Test_BloomConsume'!`
        *   `CastingSystem: Player EntityID(0, Gen:1) transitioned to state: Recovery.`
        *   After `Recovery` (0.3s): `CastingSystem: Player EntityID(0, Gen:1) transitioned to state: Idle.`
6.  **Test 2 (Insufficient Chakra)**:
    *   Repeatedly cast `Test_BloomConsume` until your Chakra is below its `chakraCost1`.
    *   The next attempt should result in: `CastingSystem: Insufficient Chakra to cast 'Test_BloomConsume'. (Need: X, Have: Y)` and `MagicSystem: Failed to initiate cast...`
7.  **Test 3 (Casting While Not Idle)**:
    *   Quickly press `0` then `1` (initiates cast).
    *   While in `CastStart` or `Recovery` states, quickly press `0` then `1` again.
    *   You should see: `CastingSystem: Cannot cast spell 'Test_BloomConsume', not in Idle state (CastStart/Recovery).`

This confirms the Casting State Machine is correctly managing the player's casting flow, deducting resources, and enforcing state transitions.

### Summary

You have successfully implemented the **Casting State Machine** in the C# Brain, managing the player's spellcasting flow through `Idle`, `Channeling`, `CastStart`, `Casting`, and `Recovery` states. By creating `CastingSystem` to handle state transitions, enforce `CastTime` and `Recovery` periods, and deduct `ChakraCost` (via `PlayerStatSystem`), you've established precise control over spell execution. This crucial system strictly adheres to TDD 02.4's specifications, providing a robust foundation for Sigilborne's dynamic magic.

### Next Steps

The next chapter will focus on **Visual Feedback - Particles & Animations**, where we will create a `ParticleManager` and `AnimationController` in GDScript to react to the C# Brain's casting state changes and spell effect execution events, bringing our spells to life visually.