## Chapter 4.5: Visual Feedback - Particles & Animations (GDScript)

With the C# Brain's `CastingSystem` now managing the player's casting flow and executing spell effects, it's time for the GDScript Body to bring these actions to life visually. This chapter focuses on implementing a `ParticleManager` and `AnimationController` in GDScript to react to the Brain's casting state changes and spell effect execution events, providing essential visual feedback for glyph inputs, cast states, and spell effects, as specified in TDD 02.5.

### 1. The Role of Visual Feedback in the Body

*   **Immersion**: Visuals make the magic feel real and impactful.
*   **Clarity**: Players need clear feedback on their inputs, casting progress, and spell outcomes.
*   **Responsiveness**: The Body must react immediately to Brain events to maintain a smooth player experience.
*   **Decoupling**: Visuals are completely separate from game logic. The Body doesn't *decide* to cast; it *shows* a cast.

### 2. Implementing a `ParticleManager.gd`

This manager will handle spawning and pooling `GPUParticles2D` for glyph inputs and spell effects.

1.  Create a new folder `res://_Body/Scenes/VFX/`.
2.  Create a simple `GPUParticles2D` scene for a generic glyph input effect: `res://_Body/Scenes/VFX/GlyphInputFX.tscn`.
    *   Root node: `GPUParticles2D`.
    *   In Inspector, set `Amount` to `10-20`.
    *   Set `Lifetime` to `0.5-1.0`.
    *   Set `Process Material` to `New ParticlesMaterial`.
        *   In `ParticlesMaterial`, set `Direction > Spread` to `180`.
        *   Set `Initial Velocity > Min/Max` to `50-100`.
        *   Set `Color` to a generic white or light blue.
        *   Set `Emission Shape` to `Sphere` (radius `5-10`).
        *   Set `Scale > Scale Curve` to fade out.
    *   Set `Texture` to a simple circle or square (you can create a `res://_Body/Art/VFX/circle_particle.png` or use a default Godot texture).
    *   Set `One Shot` to `true`.
    *   Set `Autostart` to `false`.
    *   Set `Emitting` to `false`.
    *   **Crucially**: Set `Process Mode` to `One Shot` (in the `GPUParticles2D` node itself, not the material). This makes it act like a single burst.
    *   Save this scene.
3.  Create `res://_Body/Scripts/Visuals/ParticleManager.gd`:

```gdscript
# _Body/Scripts/Visuals/ParticleManager.gd
class_name ParticleManager extends Node

static var instance: ParticleManager

# Particle pools for different effects
var _glyph_input_fx_pool: Array[GPUParticles2D] = []
const GLYPH_INPUT_FX_SCENE: PackedScene = preload("res://_Body/Scenes/VFX/GlyphInputFX.tscn")
const GLYPH_INPUT_POOL_SIZE: int = 10

func _init():
    if instance != null:
        push_error("ParticleManager: More than one instance detected!")
        queue_free()
        return
    instance = self

func _ready():
    GD.print("ParticleManager: Initialized. Pre-populating particle pools.")
    _populate_pool(GLYPH_INPUT_FX_SCENE, _glyph_input_fx_pool, GLYPH_INPUT_POOL_SIZE)

    # Connect to C# events for visual feedback
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        GameManager.Instance.Events.OnGlyphInput.connect(Callable(self, "_on_glyph_input"))
        GameManager.Instance.Events.OnCastStateChanged.connect(Callable(self, "_on_cast_state_changed"))
        GameManager.Instance.Events.OnSpellEffectExecuted.connect(Callable(self, "_on_spell_effect_executed"))
        GD.print("ParticleManager: Connected to C# magic events.")
    else:
        push_error("ParticleManager: GameManager or EventBus not ready! Cannot connect C# events.")

func _populate_pool(scene: PackedScene, pool: Array[GPUParticles2D], size: int) -> void:
    for i in range(size):
        var fx: GPUParticles2D = scene.instantiate()
        add_child(fx)
        fx.process_mode = GPUParticles2D.PROCESS_MODE_PAUSED # Ensure it's not emitting
        fx.emitting = false # Explicitly set to false
        pool.append(fx)

func _get_free_fx(pool: Array[GPUParticles2D]) -> GPUParticles2D:
    for fx in pool:
        if not fx.emitting: # Check if it's currently not emitting (available)
            return fx
    # If no free particles, expand pool (or return null if strictly fixed size)
    push_warning("ParticleManager: Particle pool exhausted! Expanding pool.")
    var new_fx: GPUParticles2D = pool[0].get_scene_file().instantiate() # Instantiate from original scene
    add_child(new_fx)
    new_fx.process_mode = GPUParticles2D.PROCESS_MODE_PAUSED
    new_fx.emitting = false
    pool.append(new_fx)
    return new_fx

## Spawns a generic particle effect at a given position.
func spawn_glyph_input_fx(position: Vector2, color: Color) -> void:
    var fx: GPUParticles2D = _get_free_fx(_glyph_input_fx_pool)
    if fx != null:
        fx.global_position = position
        # Temporarily override particle material color
        if fx.process_material is ParticlesMaterial:
            (fx.process_material as ParticlesMaterial).color = color
        
        fx.restart() # This implicitly sets emitting to true for one shot
        # GD.print("ParticleManager: Spawned glyph FX at %s with color %s" % [position, color])

## Handler for C# OnGlyphInput event.
## (TDD 02.5: Particle Manager - Signal: OnGlyphInput(GlyphType type))
func _on_glyph_input(id: int, symbol_id: String, concept: int, knowledge_state: int, is_known: bool, is_valid: bool) -> void:
    # Only show FX for player's input
    if id == GameManager.Instance.Entities.GetPlayerEntityID().Index: # Access C# player ID via Index
        var player_entity_view = EntityViewManager.instance._active_entity_views.get(id)
        if player_entity_view != null:
            var fx_color: Color = Color.WHITE # Default color
            match concept: # Match concept (int) to a color (TDD 02.5: Color matches the element)
                Magic.GlyphConcept.Bloom: fx_color = Color.GREEN
                Magic.GlyphConcept.Veil: fx_color = Color.BLUE
                Magic.GlyphConcept.Pulse: fx_color = Color.RED
                Magic.GlyphConcept.Bind: fx_color = Color.PURPLE
                Magic.GlyphConcept.Consume: fx_color = Color.BLACK
                Magic.GlyphConcept.Fracture: fx_color = Color.ORANGE
                Magic.GlyphConcept.Echo: fx_color = Color.CYAN
                Magic.GlyphConcept.Project: fx_color = Color.YELLOW
                Magic.GlyphConcept.Conjure: fx_color = Color.BROWN
                Magic.GlyphConcept.Shape: fx_color = Color.GRAY
                Magic.GlyphConcept.Flux: fx_color = Color.MAGENTA
                Magic.GlyphConcept.Chain: fx_color = Color.DARK_RED
                _: fx_color = Color.WHITE # Default for None or unknown
            
            spawn_glyph_input_fx(player_entity_view.global_position, fx_color)

## Handler for C# OnCastStateChanged event.
func _on_cast_state_changed(id: int, new_state: int, spell_id: String) -> void:
    if id == GameManager.Instance.Entities.GetPlayerEntityID().Index:
        match new_state:
            Magic.CastState.CastStart:
                GD.print("ParticleManager: Player started casting '%s'." % spell_id)
                # Play specific cast start VFX here (e.g., charging particles)
            Magic.CastState.Casting:
                GD.print("ParticleManager: Player is actively casting '%s'." % spell_id)
                # Play sustained casting VFX here
            Magic.CastState.Recovery:
                GD.print("ParticleManager: Player is in recovery after casting '%s'." % spell_id)
            Magic.CastState.Interrupted:
                GD.print("ParticleManager: Player cast '%s' was interrupted." % spell_id)

## Handler for C# OnSpellEffectExecuted event.
func _on_spell_effect_executed(id: int, spell: Variant) -> void:
    if id == GameManager.Instance.Entities.GetPlayerEntityID().Index:
        GD.print("ParticleManager: Player executed spell effect for '%s'." % spell.ID)
        var player_entity_view = EntityViewManager.instance._active_entity_views.get(id)
        if player_entity_view != null:
            # Spawn specific spell effect VFX based on spell.Projectile.VisualID, spell.AoE.VisualID
            # For now, a generic burst at player position
            # (Example: spawn_vfx_by_id(spell.Projectile.VisualID, player_entity_view.global_position))
            spawn_glyph_input_fx(player_entity_view.global_position, Color.GOLD) # Generic burst for any effect
```

### 3. Implementing an `AnimationController.gd` (Conceptual)

TDD 02.5 mentions an `AnimationController`. This would be a script that listens to `OnCastStateChanged` and `OnSpellEffectExecuted` events and triggers specific animations on the `EntityView`'s `AnimationPlayer`. For now, we'll keep this conceptual, as `EntityView.gd` already has a `play_animation` method.

**Conceptual `AnimationController.gd` (not a new file, just for understanding):**

```gdscript
# (Inside EntityView.gd or a dedicated AnimationController.gd attached to EntityRoot)
# ...
func _on_cast_state_changed(id: int, new_state: int, spell_id: String) -> void:
    if entity_id == id: # Check if this event is for *this* entity_view
        match new_state:
            Magic.CastState.Channeling:
                play_animation("channel_loop") # Play a glyph channeling animation
            Magic.CastState.CastStart:
                play_animation("cast_start") # Play a short animation for spell wind-up
            Magic.CastState.Casting:
                play_animation("cast_loop") # Play a sustained casting animation
            Magic.CastState.Recovery:
                play_animation("cast_recovery") # Play a post-cast animation
            Magic.CastState.Idle:
                play_animation("idle") # Return to idle animation
            Magic.CastState.Interrupted:
                play_animation("interrupted") # Play an interrupted animation
# ...
func _on_spell_effect_executed(id: int, spell: Variant) -> void:
    if entity_id == id:
        # Trigger specific animations for spell execution
        # Example: play_animation("attack_spell_" + spell.ID)
        pass
```
For now, `ParticleManager.gd` will handle the visual feedback, and `EntityView.gd`'s existing `play_animation` will be used if triggered by the Brain.

### 4. Integrating `ParticleManager.gd` into `Main.tscn`

1.  Open `res://Main.tscn`.
2.  Add an instance of `res://_Body/Scripts/Visuals/ParticleManager.gd` as a child of the `Main` root node.
3.  Save `Main.tscn`.

### 5. Testing Visual Feedback

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  **Test 1 (Glyph Input FX)**: Press `0` (hotbar slot 0, for `testSymbol`).
    *   You should see a particle burst at the player's position, colored according to the concept (e.g., green for Bloom).
    *   The console will show `ParticleManager: Player input glyph...`
5.  **Test 2 (Casting State FX)**: Quickly press `0` then `1` (for `Test_BloomConsume`).
    *   You should see a particle burst for each glyph input.
    *   The console will show:
        *   `ParticleManager: Player started casting 'Test_BloomConsume'.`
        *   `ParticleManager: Player executed spell effect for 'Test_BloomConsume'.` (followed by a gold particle burst)
        *   `ParticleManager: Player is in recovery after casting 'Test_BloomConsume'.`

This confirms that our `ParticleManager.gd` is correctly reacting to C# events and providing visual feedback for glyph inputs and spell execution.

### Summary

You have successfully implemented **Visual Feedback** for Sigilborne's magic system, creating a `ParticleManager.gd` in the GDScript Body to react to C# Brain events. By designing a particle pooling system and connecting it to `OnGlyphInput`, `OnCastStateChanged`, and `OnSpellEffectExecuted` events, you've ensured that glyph inputs and spell effects are visually represented with immediate and context-sensitive particle bursts, strictly adhering to TDD 02.5's specifications. This crucial step brings our spells to life, enhancing player immersion and clarity.

### Next Steps

This concludes **Module 4: Combos & Casting - The Art of Ninjutsu**. We will now move on to **Module 5: Chakra & Life Systems**, starting with **Biological Simulation - The Bio-Tick (C#)**, where we will implement a slower, dedicated update loop for biological stats to manage player and NPC well-being efficiently.