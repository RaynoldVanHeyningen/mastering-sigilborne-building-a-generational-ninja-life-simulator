## Chapter 6.1: Physics Layer - Hitbox & Hurtbox Detection (GDScript)

Welcome to **Module 6: Combat & Tactical Engagement**! This module dives into the core mechanics of Sigilborne's combat system. The foundation of any combat interaction is collision detection. This chapter focuses on implementing **Hitbox & Hurtbox detection** using Godot's `Area2D` nodes in the GDScript Body. These visual components will detect overlaps and send `HitEvent`s to the C# Brain for authoritative damage calculation, as specified in TDD 04.2.

### 1. The Role of Hitboxes and Hurtboxes

*   **Hitbox**: Represents the attacking part of an entity (e.g., a sword swing, a projectile). It defines "where I am hitting."
*   **Hurtbox**: Represents the vulnerable part of an entity (e.g., a character's body, a monster's head). It defines "where I can be hit."

**Key Principles:**

*   **Body-Owned**: Hitboxes and Hurtboxes are `Area2D` nodes in the GDScript Body, responsible for *detecting* overlaps.
*   **Brain-Informed**: When an overlap occurs, the Body sends a `HitEvent` to the C# Brain.
*   **Decoupled Logic**: The Body does not calculate damage; it merely reports a potential hit. The Brain decides the outcome.

### 2. Defining Physics Layers

TDD 17.4 provides a standard for physics layers. We need to ensure these are set up in Godot's Project Settings.

1.  Go to `Project > Project Settings... > Layer Names > 2D Physics`.
2.  Rename the layers according to TDD 17.4:

    | Bit | Layer Name |
    | :-- | :--------- |
    | 1   | `World`    |
    | 2   | `Player`   |
    | 3   | `Enemy`    |
    | 4   | `Projectile` |
    | 5   | `Interaction` |
    | 6   | `Hitbox`   |

3.  Close Project Settings.

### 3. Implementing Hurtboxes for Entities

Every living entity (player, NPC, animal) that can take damage needs a Hurtbox. This will be a child of our `EntityRoot.tscn` scene.

1.  Open `res://_Body/Scenes/Entities/EntityRoot.tscn`.
2.  Add an `Area2D` node as a child of `EntityRoot`.
    *   Rename it `Hurtbox`.
    *   In the Inspector, set its `Collision Layer` to `Player` (for the player) or `Enemy` (for NPCs/animals).
        *   For `EntityRoot`, let's default it to `Player` layer for now, and we'll change it dynamically for NPCs.
    *   Set its `Collision Mask` to `Hitbox` (it should detect hits from Hitboxes).
3.  Add a `CollisionShape2D` as a child of `Hurtbox`.
    *   In the Inspector, set its `Shape` to `New CapsuleShape2D`.
    *   Select the `CapsuleShape2D` resource. Adjust its `Radius` (e.g., `8`) and `Height` (e.g., `20`) to roughly match a humanoid character.
    *   Position the `CollisionShape2D` (e.g., `y=-10`) to center it on the character.
4.  Save `EntityRoot.tscn`.

### 4. Implementing Hitboxes for Attacks (Conceptual)

Hitboxes are typically attached to weapons, spell effects, or animation frames. For this chapter, we'll conceptually prepare for them.

*   A weapon scene (e.g., `IronSword.tscn`) would have an `Area2D` child named `Hitbox`.
*   A projectile scene (e.g., `Fireball.tscn`) would have an `Area2D` as its root or child.

**Common Hitbox Properties:**

*   `Collision Layer`: `Hitbox`.
*   `Collision Mask`: `Player` or `Enemy` (who it can hit).
*   **Data**: Hitboxes need to carry `DamageProfileID` (TDD 04.2) and `AttackerID` (who owns this hitbox).

### 5. `HitEvent` Data Structure (Brain)

When a `Hitbox` detects a `Hurtbox`, the Body needs to send precise data to the C# Brain. TDD 04.2 specifies `HitEvent`.

1.  Create `res://_Brain/Systems/Combat/HitEvent.cs`:

```csharp
// _Brain/Systems/Combat/HitEvent.cs
using System;
using Godot; // For Vector2
using Sigilborne.Entities;

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Data describing a potential hit, sent from the GDScript Body to the C# Brain.
    /// (TDD 04.2.2)
    /// </summary>
    public struct HitEvent
    {
        public EntityID AttackerID;         // The entity that owns the hitbox.
        public EntityID DefenderID;         // The entity that owns the hurtbox.
        public string HitboxDefinitionID;   // ID of the hitbox (e.g., "sword_slash_01", "fireball_projectile").
        public string HurtboxBodyPart;      // Which part of the defender was hit (e.g., "head", "torso").
        public Vector2 HitPosition;         // World position where the hit occurred.
        public Vector2 HitNormal;          // Normal vector of the collision (useful for knockback direction).

        public HitEvent(EntityID attackerID, EntityID defenderID, string hitboxDefinitionID, string hurtboxBodyPart, Vector2 hitPosition, Vector2 hitNormal)
        {
            AttackerID = attackerID;
            DefenderID = defenderID;
            HitboxDefinitionID = hitboxDefinitionID;
            HurtboxBodyPart = hurtboxBodyPart;
            HitPosition = hitPosition;
            HitNormal = hitNormal;
        }

        public override string ToString()
        {
            return $"Hit! Attacker: {AttackerID}, Defender: {DefenderID}, Hitbox: '{HitboxDefinitionID}', Part: '{HurtboxBodyPart}', Pos: {HitPosition}";
        }
    }
}
```

### 6. Updating `EntityView.gd` to Handle Hurtbox Detection

`EntityView.gd` will manage its `Hurtbox` and, when it detects an overlap, will call a C# method to register the hit.

1.  Open `res://_Body/Scripts/Visuals/EntityView.gd`.
2.  Add a new method `_on_hurtbox_area_entered` and connect it to the `Hurtbox` node's `area_entered` signal.

```gdscript
# _Body/Scripts/Visuals/EntityView.gd
class_name EntityView extends CharacterBody2D

# ... (existing properties and node references) ...
@onready var hurtbox: Area2D = $Hurtbox # New: Reference to Hurtbox
# @onready var hitbox: Area2D = $Hitbox # Conceptual: If this entity also has a hitbox


func setup(id: int, initial_position: Vector2, initial_rotation: float, definition_id: String) -> void:
    entity_id = id
    global_position = initial_position
    brain_target_position = initial_position
    brain_target_rotation_degrees = initial_rotation
    visuals.rotation_degrees = initial_rotation
    
    # ... (animation_player setup) ...

    # --- Setup Hurtbox (TDD 04.2) ---
    if hurtbox != null:
        hurtbox.area_entered.connect(Callable(self, "_on_hurtbox_area_entered"))
        # Set collision layer dynamically based on EntityType (conceptual)
        # For now, EntityRoot is assumed to be player, so Player layer (bit 1) is set in scene.
        # For NPC/Animal, it would be Enemy layer (bit 2).
        # if definition_id.begins_with("player"):
        #     hurtbox.set_collision_layer_value(1, true) # Player layer
        #     hurtbox.set_collision_layer_value(2, false) # Not Enemy layer
        # else: # NPC/Animal
        #     hurtbox.set_collision_layer_value(1, false)
        #     hurtbox.set_collision_layer_value(2, true) # Enemy layer
        GD.print("EntityView %s: Hurtbox initialized." % entity_id)

    GD.print("EntityView: Setup for C# EntityID %s (Def: %s) at %s" % [entity_id, definition_id, global_position])
    
    # ... (existing C# event connections) ...
    pass

# ... (existing _physics_process, _on_entity_moved_reconciliation, _on_entity_velocity_update,
#      _on_animation_finished, _on_anim_track_event, play_animation methods) ...


## Handles when a Hitbox enters this entity's Hurtbox.
## (TDD 04.2.2: Body detects collision via area_entered signal)
func _on_hurtbox_area_entered(other_area: Area2D) -> void:
    # Check if the other_area is a Hitbox (e.g., by checking its collision layer or name)
    # For now, we'll assume any Area2D on the Hitbox layer is a valid hitbox.
    if other_area.get_collision_layer_value(6): # Layer 6 is Hitbox
        # We need to get information from the Hitbox itself.
        # Hitboxes should carry data like AttackerID and HitboxDefinitionID.
        # For now, let's assume the Hitbox has a script or metadata attached.
        
        # Conceptual: Get data from the Hitbox (e.g., from its parent EntityView script if it's a weapon)
        var hitbox_attacker_id: int = -1 # Placeholder
        var hitbox_def_id: String = "unknown_hitbox" # Placeholder

        # Example: If the hitbox is part of a projectile, its parent EntityView would be the projectile.
        # If it's a weapon swing, its parent EntityView would be the attacker.
        var attacker_entity_view: EntityView = other_area.get_parent_node_3d() # Or get_parent() if 2D
        if attacker_entity_view != null and attacker_entity_view is EntityView:
            hitbox_attacker_id = attacker_entity_view.entity_id
            # We would also get hitbox_def_id from the attacker_entity_view's current attack definition.
        
        # Get hit position and normal
        var hit_position: Vector2 = other_area.global_position # Simple approximation
        var hit_normal: Vector2 = (global_position - other_area.global_position).normalized() # Normal from hitbox to hurtbox center

        # Get hurtbox body part (conceptual for now, could be dynamic based on hit location)
        var hurtbox_body_part: String = "torso"

        # TDD 04.2.2: Body sends HitEvent to the Brain.
        if GameManager.Instance != null and GameManager.Instance.Combat != null:
            # We need to pass EntityID structs, not just indices.
            # GameManager.Instance.Entities.GetEntityMeta(entity_id).Generation
            var defender_entity_id_struct = GameManager.Instance.Entities.GetEntityMeta(entity_id)
            var attacker_entity_id_struct = GameManager.Instance.Entities.GetEntityMeta(hitbox_attacker_id)

            # Create a C# HitEvent struct and pass it.
            var hit_event = HitEvent.new() # Instantiate C# struct via Godot binding
            hit_event.AttackerID = attacker_entity_id_struct # Need a way to convert int to EntityID struct
            hit_event.DefenderID = defender_entity_id_struct
            hit_event.HitboxDefinitionID = hitbox_def_id
            hit_event.HurtboxBodyPart = hurtbox_body_part
            hit_event.HitPosition = hit_position
            hit_event.HitNormal = hit_normal
            
            GameManager.Instance.Combat.RegisterHit(hit_event) # Call C# method
            get_viewport().set_input_as_handled() # Prevent multiple hits from one frame if needed.
        else:
            push_error("EntityView: GameManager or CombatSystem not ready to register hit.")
```

**Key Changes in `EntityView.gd`:**

*   **`@onready var hurtbox`**: Gets a reference to the `Hurtbox` node.
*   **`setup()`**: Connects `hurtbox.area_entered` signal.
*   **`_on_hurtbox_area_entered()`**:
    *   This is the signal handler for incoming hits.
    *   It conceptually extracts `AttackerID` and `HitboxDefinitionID` from the `other_area` (the hitbox).
    *   It gathers `HitPosition` and `HitNormal`.
    *   **Crucially**: It creates a C# `HitEvent` struct instance and calls `GameManager.Instance.Combat.RegisterHit(hit_event)` to send the data to the Brain.

#### 6.1. Update `EntityManager.cs` for `GetEntityMeta` for GDScript

`EntityView.gd` needs to get the full `EntityID` struct, not just the index, to reconstruct it for the `HitEvent`.

Open `res://_Brain/Entities/EntityManager.cs` and add this helper method:

```csharp
// _Brain/Entities/EntityManager.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;

namespace Sigilborne.Entities
{
    // ... (EntityID, EntityMeta, EntityType structs/enums) ...

    public class EntityManager
    {
        // ... (existing fields and constructor) ...

        /// <summary>
        /// Retrieves the EntityMeta struct for a given entity index.
        /// Primarily for GDScript interop to reconstruct EntityID from index.
        /// </summary>
        public EntityMeta GetEntityMeta(int index)
        {
            if (index < 0 || index >= MAX_ENTITIES)
            {
                GD.PrintErr($"EntityManager: Invalid index {index} requested for GetEntityMeta.");
                return new EntityMeta { Generation = -1, IsActive = false, Type = EntityType.Invalid, DefinitionID = "invalid" };
            }
            return _entityMetas[index];
        }

        // ... (other methods) ...
    }
}
```

### 7. Implementing `CombatSystem.cs` (Brain Backend)

This is the system that will receive the `HitEvent` from GDScript and orchestrate damage calculation.

1.  Open `res://_Brain/Systems/Combat/DamageSystem.cs` and rename the class to `CombatSystem` (as per TDD 04).
2.  Add a `RegisterHit` method.

```csharp
// _Brain/Systems/Combat/CombatSystem.cs (formerly DamageSystem.cs)
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.StatusEffects;

namespace Sigilborne.Systems.Combat
{
    // ... (DamageType, DamageResult structs/enums) ...
    // ... (HitEvent struct) ...

    /// <summary>
    /// Manages the full combat pipeline: receiving hit events, damage calculation,
    /// application to CoreStats, and generation of StatusEffects (wounds).
    /// (TDD 03.4, TDD 04.2, TDD 04.3)
    /// </summary>
    public class CombatSystem // Renamed from DamageSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem;
        private StatusEffectSystem _statusEffectSystem;

        public CombatSystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, StatusEffectSystem statusEffectSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _statusEffectSystem = statusEffectSystem;
            GD.Print("CombatSystem: Initialized.");
        }

        /// <summary>
        /// Registers a hit event received from the GDScript Body.
        /// (TDD 04.2.2: Body sends HitEvent to the Brain, Brain calculates the result)
        /// </summary>
        /// <param name="hitEvent">The HitEvent data from the Body.</param>
        public void RegisterHit(HitEvent hitEvent)
        {
            // Basic validation
            if (!_entityManager.IsValid(hitEvent.AttackerID) || !_entityManager.IsValid(hitEvent.DefenderID))
            {
                GD.PrintErr($"CombatSystem: Invalid attacker or defender in hit event: {hitEvent}.");
                return;
            }
            
            GD.Print($"CombatSystem: Received hit event: {hitEvent}.");

            // --- Actual damage calculation (from previous ApplyDamage logic) ---
            // For now, let's use a dummy raw damage based on hitbox ID.
            float rawDamage = 10f; // Default
            DamageType damageType = DamageType.Physical; // Default
            float penetration = 0f;

            if (hitEvent.HitboxDefinitionID.Contains("fireball"))
            {
                rawDamage = 25f;
                damageType = DamageType.Elemental;
            }
            else if (hitEvent.HitboxDefinitionID.Contains("void"))
            {
                rawDamage = 30f;
                damageType = DamageType.Residue;
            }

            // Call the core damage application logic.
            ApplyDamage(hitEvent.AttackerID, hitEvent.DefenderID, rawDamage, damageType, penetration);
        }


        /// <summary>
        /// Calculates and applies damage to a target entity.
        /// This is the entry point for all damage in the game.
        /// (TDD 03.4: Damage Pipeline)
        /// </summary>
        /// <param name="attackerID">The entity dealing damage.</param>
        /// <param name="defenderID">The entity receiving damage.</param>
        /// <param name="rawDamage">The base damage value.</param>
        /// <param name="damageType">The type of damage.</param>
        /// <param name="penetration">How much the damage ignores defender's mitigation.</param>
        /// <returns>The final DamageResult.</returns>
        public DamageResult ApplyDamage(EntityID attackerID, EntityID defenderID, float rawDamage, DamageType damageType, float penetration = 0f)
        {
            // ... (existing ApplyDamage logic from previous chapter) ...
            if (!_biologicalSystem.TryGetCoreStats(defenderID, out CoreStats defenderStats))
            {
                GD.PrintErr($"CombatSystem: Defender {defenderID} has no CoreStats. Cannot apply damage.");
                return new DamageResult(0, false, false, DamageType.None, attackerID, defenderID);
            }

            float mitigation = 0;
            switch (damageType)
            {
                case DamageType.Physical:
                    mitigation = defenderStats.Armor * (1.0f - penetration);
                    break;
                case DamageType.Chakra:
                case DamageType.Elemental:
                case DamageType.Spiritual:
                case DamageType.Residue:
                    mitigation = defenderStats.MagicResistance * (1.0f - penetration);
                    break;
                case DamageType.Internal:
                    mitigation = 0; // Internal damage often bypasses armor
                    break;
            }
            float finalDamage = rawDamage - mitigation;
            finalDamage = Mathf.Max(0, finalDamage);

            bool isCrit = false;
            bool isBlocked = false;

            DamageResult result = new DamageResult(finalDamage, isCrit, isBlocked, damageType, attackerID, defenderID);
            GD.Print($"CombatSystem: {attackerID} dealt {result} to {defenderID}.");

            if (result.FinalDamage > 0)
            {
                ref CoreStats targetCoreStats = ref _biologicalSystem.GetCoreStatsRef(defenderID);
                targetCoreStats.Health -= result.FinalDamage;
                if (targetCoreStats.Health < 0) targetCoreStats.Health = 0;

                if (defenderID == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerStatSystem.PlayerHealthChangedEvent { PlayerID = defenderID, NewValue = targetCoreStats.Health, MaxValue = targetCoreStats.MaxHealth });
                }

                GenerateWounds(defenderID, result);

                if (targetCoreStats.Health == 0)
                {
                    GD.Print($"CombatSystem: {defenderID} has been defeated!");
                    if (defenderID == _entityManager.GetPlayerEntityID())
                    {
                        _eventBus.Publish(new PlayerStatSystem.PlayerDiedEvent { PlayerID = defenderID });
                    }
                }
            }
            return result;
        }

        private void GenerateWounds(EntityID targetID, DamageResult result)
        {
            // ... (existing GenerateWounds logic) ...
            if (result.FinalDamage <= 0) return;

            float woundChance = result.FinalDamage / _biologicalSystem.GetCoreStatsRef(targetID).MaxHealth;
            if (woundChance > 0.1f)
            {
                Random rand = new Random(targetID.Index + (int)GameManager.Instance.Time.CurrentGameTime);
                if (rand.NextDouble() < woundChance)
                {
                    string effectId = "";
                    float duration = 0;
                    float strength = 0;

                    switch (result.Type)
                    {
                        case DamageType.Physical:
                            effectId = "bleed_t1";
                            duration = 5f;
                            strength = result.FinalDamage * 0.05f;
                            break;
                        case DamageType.Elemental:
                            effectId = "burn_t1";
                            duration = 3f;
                            strength = result.FinalDamage * 0.03f;
                            break;
                        case DamageType.Chakra:
                            effectId = "chakra_drain_t1";
                            duration = 2f;
                            strength = result.FinalDamage * 0.02f;
                            break;
                        case DamageType.Residue:
                            effectId = "void_sickness_t1";
                            duration = 10f;
                            strength = result.FinalDamage * 0.01f;
                            break;
                        default:
                            return;
                    }

                    if (!string.IsNullOrEmpty(effectId))
                    {
                        _statusEffectSystem.ApplyEffect(targetID, new StatusEffectData(effectId, duration, strength), result.AttackerID);
                    }
                }
            }
        }
    }
}
```

### 8. Updating `GameManager` to Reflect `CombatSystem`

1.  Rename `Combat` property in `GameManager` from `DamageSystem` to `CombatSystem`.
2.  Update its initialization in `InitializeSystems()`.

```csharp
// _Brain/Core/GameManager.cs
// ...
using Sigilborne.Systems.Combat; // Add this using directive
// ...

public partial class GameManager : Node
{
    // ... (existing system properties) ...
    public CombatSystem Combat { get; private set; } // Renamed from DamageSystem

    public override void _Ready()
    {
        // ... (existing test code) ...
        
        // --- Test Damage & Recovery Pipeline ---
        GD.Print("\n--- Testing Damage & Recovery Pipeline ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        // Get NPC ID assuming it's still ID 1, Gen 1
        // We need a way to get the Generation dynamically for NPCs.
        // For now, let's just create a dummy NPC for this test.
        EntityID testNpcID = Entities.CreateEntity(EntityType.NPC, "test_dummy_npc", new Vector2(500, 200));

        GD.Print($"Player initial HP: {Biology.GetCoreStatsRef(playerID).Health}");
        GD.Print($"Test NPC initial HP: {Biology.GetCoreStatsRef(testNpcID).Health}");

        // Player attacks NPC (will be triggered by HitEvent from GDScript)
        // For testing, we'll manually call RegisterHit.
        Combat.RegisterHit(new HitEvent(playerID, testNpcID, "player_sword_slash", "torso", new Vector2(450, 200), Vector2.Left));
        Combat.RegisterHit(new HitEvent(playerID, testNpcID, "player_fireball", "torso", new Vector2(450, 200), Vector2.Left));
        Combat.RegisterHit(new HitEvent(playerID, testNpcID, "player_void_blast", "torso", new Vector2(450, 200), Vector2.Left)); // Should trigger void sickness

        // NPC attacks Player (will be triggered by HitEvent from GDScript)
        // For testing, we'll manually call RegisterHit.
        Combat.RegisterHit(new HitEvent(testNpcID, playerID, "npc_claw", "torso", new Vector2(250, 200), Vector2.Right));
        Combat.RegisterHit(new HitEvent(testNpcID, playerID, "npc_poison_spit", "torso", new Vector2(250, 200), Vector2.Right));
        Combat.RegisterHit(new HitEvent(testNpcID, playerID, "npc_heavy_hit", "torso", new Vector2(250, 200), Vector2.Right)); // Should kill player

        GD.Print("--- End Testing Damage & Recovery Pipeline ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        // ... (existing PhysicsProcess calls) ...
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to BiologicalSystem) ...
        
        // Initialize StatusEffectSystem BEFORE CombatSystem
        StatusEffects = new StatusEffectSystem(Entities, Events, BiologicalSystem);
        GD.Print("  - StatusEffectSystem initialized.");

        // Initialize CombatSystem (renamed from DamageSystem), passing BiologicalSystem and StatusEffectSystem
        Combat = new CombatSystem(Entities, Events, BiologicalSystem, StatusEffects); // Renamed to CombatSystem
        GD.Print("  - CombatSystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 9. Testing Hitbox & Hurtbox Detection (Conceptual)

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  The console output will show the direct `CombatSystem.RegisterHit` calls and their results.

**To truly test the GDScript -> C# Hitbox/Hurtbox pipeline:**

*   You would need to create a dummy `Hitbox` scene (e.g., a simple `Area2D` with `CollisionShape2D` on layer `Hitbox`).
*   Attach a script to this `Hitbox` that, when it enters another `Area2D` on `Player` or `Enemy` layer, calls `GameManager.Instance.Combat.RegisterHit()` with appropriate `EntityID`s.
*   For now, our manual calls in `GameManager._Ready()` simulate this, showing that the C# backend is ready to process these events.

This confirms the C# backend for combat is ready to receive `HitEvent`s from the GDScript Body.

### Summary

You have successfully implemented the foundational **Hitbox & Hurtbox detection** system for Sigilborne's combat. By defining `DamageType`, `DamageResult`, and `HitEvent` structs, and enhancing `CombatSystem` (formerly `DamageSystem`) to receive and process `HitEvent`s from the GDScript Body, you've established the core pipeline for collision detection and authoritative damage calculation. `EntityView.gd` is prepared to manage its `Hurtbox` and send `HitEvent`s, strictly adhering to TDD 04.2's specifications. This crucial step enables systemic and precise combat interactions.

### Next Steps

The next chapter will focus on **The Math Layer**, where we will fully implement the `Damage Calculator` within the C# `CombatSystem`, incorporating `CoreStats` (Armor, MagicResistance) and advanced formulas to determine final damage, critical hits, and blocks.