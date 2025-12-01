## Chapter 1.11: Animation Event Protocol - Syncing Brain & Body

In a game like Sigilborne, where combat is systemic and timing-sensitive, precise synchronization between visual animations (Body) and simulation logic (Brain) is crucial. A sword swing's "hit frame" must trigger damage calculation in the C# Brain at the exact moment the animation visually connects. This chapter implements the **Animation Event Protocol**, a strict standard for using Godot's `AnimationPlayer` to emit signals that the Brain can interpret, as detailed in TDD 16.5.

### 1. The Challenge of Animation Synchronization

*   **Decoupling**: The Brain doesn't know about `AnimationPlayer` nodes. The Body plays animations.
*   **Precision**: We need to trigger C# logic (e.g., "activate hitbox," "spawn projectile") at specific frames of an animation.
*   **Reliability**: The communication must be robust and thread-safe.

Our solution is to use Godot's built-in `AnimationPlayer` functionality to emit custom signals, which our `EntityView.gd` script will then forward to the C# Brain via our `EventBus`.

### 2. The Signal Standard: `anim_event(type: String, payload: Dictionary)`

TDD 16.5 specifies a standard signal format for animation events:

*   **Signal Name**: `anim_event` (declared in `EntityView.gd`).
*   **Parameter 1 (`type: String`)**: A string identifying the type of event (e.g., `"hit_frame"`, `"footstep"`, `"cast_release"`).
*   **Parameter 2 (`payload: Dictionary`)**: A dictionary containing any additional data relevant to the event (e.g., `{"hitbox_id": "sword_slash_01"}`, `{"socket": "right_hand"}`).

This generic format allows us to handle a wide range of animation-driven events with a single signal.

### 3. Modifying `EntityView.gd` to Emit Animation Events

Our `EntityView.gd` script, attached to `EntityRoot.tscn`, will be the intermediary. It will listen for a generic `animation_finished` signal from an `AnimationPlayer` (if present) or specific `AnimationPlayer` track events, and then re-emit them using our standardized `anim_event` signal.

First, let's update `EntityView.gd` to declare the `anim_event` signal and prepare to handle animation events.

Open `res://_Body/Scripts/Visuals/EntityView.gd`:

```gdscript
# _Body/Scripts/Visuals/EntityView.gd
class_name EntityView extends CharacterBody2D

# ... (existing properties) ...

# --- Node References ---
@onready var visuals: Node2D = $Visuals
@onready var sprite: Sprite2D = $Visuals/Sprite
@onready var animation_player: AnimationPlayer = null # Initialize as null, find in _ready if exists
@onready var vfx_attachment_point: Node2D = $Visuals/VFX
@onready var audio_attachment_point: Node2D = $Audio
@onready var ui_attachment_point: Control = $UI

# ... (existing interpolation parameters) ...

# --- Signals (TDD 16.5) ---
# This signal is emitted when an animation event occurs.
# It's caught by the Brain (C#) via GameEvents.
signal anim_event(type: String, payload: Dictionary)


## Initializes this EntityView with its corresponding C# EntityID.
func setup(id: int, initial_position: Vector2, initial_rotation: float, definition_id: String) -> void:
    entity_id = id
    global_position = initial_position
    brain_target_position = initial_position
    visuals.rotation_degrees = initial_rotation
    
    # Try to find an AnimationPlayer if it's a child of Visuals
    # This assumes AnimationPlayer is a direct child of Visuals, which is a common pattern.
    if visuals.has_node("AnimationPlayer"):
        animation_player = visuals.get_node("AnimationPlayer")
        # Connect to the AnimationPlayer's 'animation_changed' signal to react to new animations
        # and 'animation_finished' for general animation completion.
        # For specific frame events, we'll use AnimationPlayer's Call Method Track.
        if animation_player != null:
            animation_player.animation_finished.connect(Callable(self, "_on_animation_finished"))
            GD.print("EntityView %s: Connected to AnimationPlayer." % entity_id)

    GD.print("EntityView: Setup for C# EntityID %s (Def: %s) at %s" % [entity_id, definition_id, global_position])
    pass

## Placeholder for reacting to C# entity movement signals.
func _on_entity_moved(id: int, new_position: Vector2, new_rotation: float) -> void:
    if entity_id == id:
        brain_target_position = new_position
        visuals.rotation_degrees = new_rotation
        # GD.print("EntityView %s: Received new position %s" % [entity_id, new_position])

# --- Animation Event Handlers ---

## Generic handler for when any animation finishes playing.
func _on_animation_finished(anim_name: String) -> void:
    # Example: Emit a generic 'animation_finished' event.
    anim_event.emit("animation_finished", {"anim_name": anim_name})
    GD.print("EntityView %s: Animation '%s' finished." % [entity_id, anim_name])

## This function will be called by an AnimationPlayer's Call Method Track.
## (TDD 16.5: The Signal Standard)
## @param type: String - The type of animation event (e.g., "hit_frame", "footstep").
## @param payload: Dictionary - Additional data for the event.
func _on_anim_track_event(type: String, payload: Dictionary) -> void:
    anim_event.emit(type, payload)
    GD.print("EntityView %s: Emitted anim_event: Type '%s', Payload %s" % [entity_id, type, payload])

# --- Public Methods for Animation Control ---
## Plays a specific animation on this entity's AnimationPlayer.
func play_animation(anim_name: String, blend_time: float = -1, custom_speed: float = 1.0, from_end: bool = false) -> void:
    if animation_player != null and animation_player.has_animation(anim_name):
        animation_player.play(anim_name, blend_time, custom_speed, from_end)
        GD.print("EntityView %s: Playing animation '%s'." % [entity_id, anim_name])
    else:
        push_warning("EntityView %s: AnimationPlayer or animation '%s' not found." % [entity_id, anim_name])
```

**Key Changes in `EntityView.gd`:**

*   **`@onready var animation_player`**: Added to get a reference to the `AnimationPlayer` node.
*   **`setup()`**: Now checks for `AnimationPlayer` and connects its `animation_finished` signal.
*   **`_on_animation_finished()`**: A generic handler for when an animation completes.
*   **`_on_anim_track_event(type: String, payload: Dictionary)`**: This is the crucial method. It's designed to be called directly from an `AnimationPlayer` using a "Call Method Track." It then emits our standardized `anim_event` signal.
*   **`play_animation()`**: A public method that other GDScript (or C# via interop) can call to trigger animations.

### 4. Setting up `AnimationPlayer` Tracks in `EntityRoot.tscn`

Now, let's add an `AnimationPlayer` to our `EntityRoot.tscn` and configure it to emit events.

1.  Open `res://_Body/Scenes/Entities/EntityRoot.tscn`.
2.  Select the `Visuals` node.
3.  Add a child node: `AnimationPlayer`.
    *   Rename it `AnimationPlayer`.
4.  In the `AnimationPlayer` dock (usually at the bottom), click `Animation > New`.
    *   Name the new animation `test_attack`.
5.  Set the animation `Length` to `1.0` seconds (or any duration).
6.  Click the "Add Track" button (the `+` icon).
    *   Select `Call Method Track`.
    *   Click `Select Node` and choose `EntityRoot` (our `EntityView.gd` script is attached here).
    *   Click `Ok`.
7.  In the `test_attack` animation timeline:
    *   Move the timeline cursor to `0.5` seconds (this will be our "hit frame").
    *   Click the "Add Key" button (the diamond icon) on the `Call Method Track`.
    *   In the "Insert Key" dialog:
        *   **Method**: Select `_on_anim_track_event`.
        *   **Parameters**: Click the `+` twice to add two parameters.
            *   Parameter 0 (type `String`): Enter `"hit_frame"`.
            *   Parameter 1 (type `Dictionary`): Click `New Dictionary`. In the dictionary editor, add a key `hitbox_id` (String) with value `"test_hitbox_01"` (String).
        *   Click `Ok`.
8.  Save `EntityRoot.tscn`.

Now, when `test_attack` animation plays, at `0.5` seconds, it will call `EntityView._on_anim_track_event("hit_frame", {"hitbox_id": "test_hitbox_01"})`, which then emits our `anim_event` signal.

### 5. Conceptually Connecting `anim_event` to the C# Brain

The `anim_event` signal is a GDScript signal. To reach the C# Brain, we need to connect it through our Interop Layer (Chapter 1.12). For now, we'll conceptually show how the C# side would subscribe.

First, let's modify `_Brain/Entities/EntityManager.cs` to add a specific event struct for animation events.

```csharp
// _Brain/Entities/EntityManager.cs (inside Sigilborne.Entities namespace)
// ...
        // --- Helper Events for Body Sync (TDD 11.4) ---
        // ... (existing events) ...
        public struct AnimationEvent { public EntityID ID; public string Type; public Dictionary<string, Variant> Payload; } // New event
    }
}
```
**Note**: `Variant` is Godot's universal type. Using `Dictionary<string, Variant>` here allows us to pass any GDScript dictionary payload.

Now, modify `_Brain/Core/EventBus.cs` to publish this new event type:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using System.Collections.Generic; // For Dictionary<string, Variant>

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing event Action definitions) ...
        public event Action<EntityManager.AnimationEvent> OnAnimationEvent; // New event

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is EntityManager.AnimationEvent animEvent) // New condition
            {
                OnAnimationEvent?.Invoke(animEvent);
            }
            else
            {
                GD.PrintErr($"EventBus: Attempted to publish unknown event type: {typeof(TEvent).Name}");
            }
        }
        // ... (AddCommand and FlushCommands methods) ...
    }
}
```

Next, in our conceptual `EntityViewManager.gd` (which will eventually handle the actual `connect()` calls), we'd connect `EntityView.anim_event` to the C# `EventBus`.

**Conceptual `EntityViewManager.gd` (for context, no code change needed here yet):**

```gdscript
# _Body/Scripts/Visuals/EntityViewManager.gd (Conceptual)
# ...
func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    # ... (instantiate entity_view) ...
    entity_view.setup(...)
    
    # --- Conceptual Connection to C# Brain ---
    # This connection will be handled in Chapter 1.12 (Interop Layer).
    # For now, imagine GameManager.Instance.Events.Connect is the C# EventBus itself.
    # The C# side will expose a method to connect to its Action<AnimationEvent> delegate.
    # We would connect EntityView's anim_event to that C# method.
    # entity_view.anim_event.connect(Callable(GameManager.Instance.Events, "ReceiveAnimationEvent"))
    
    _active_entity_views[id] = entity_view
# ...
```

### 6. Testing the Animation Event Protocol

To test this, we need to make `EntityViewManager.gd` play the `test_attack` animation.

Modify `_Body/Scripts/Visuals/EntityViewManager.gd` to play an animation for the NPC shortly after it's spawned:

```gdscript
# _Body/Scripts/Visuals/EntityViewManager.gd
class_name EntityViewManager extends Node

static var instance: EntityViewManager
var _active_entity_views: Dictionary = {}
const ENTITY_ROOT_SCENE: PackedScene = preload("res://_Body/Scenes/Entities/EntityRoot.tscn")

func _init():
    if instance != null:
        push_error("EntityViewManager: More than one instance detected!")
        queue_free()
        return
    instance = self

func _ready():
    GD.print("EntityViewManager: Initialized. Waiting for C# entity events.")
    # For testing, we'll add a timer to play an animation after entities are spawned.
    # This would normally be triggered by C# combat logic.
    await get_tree().create_timer(3.0).timeout # Wait 3 seconds after scene loads

    # Find an NPC entity view and play its attack animation
    for id in _active_entity_views:
        var entity_view: EntityView = _active_entity_views[id]
        # Assuming we only have one NPC moving from previous chapter
        if entity_view.entity_id == 1: # NPC entity has ID 1 from previous tests
            entity_view.play_animation("test_attack")
            break
    pass

func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    if ENTITY_ROOT_SCENE == null:
        push_error("EntityViewManager: ENTITY_ROOT_SCENE not loaded!")
        return

    var entity_view: EntityView = ENTITY_ROOT_SCENE.instantiate()
    add_child(entity_view)
    
    entity_view.setup(id, initial_position, initial_rotation, definition_id)
    
    # Crucial: Connect the EntityView's anim_event signal to our manager's conceptual handler
    # This allows EntityViewManager to "catch" events locally before forwarding to C#
    entity_view.anim_event.connect(Callable(self, "_on_entity_view_anim_event"))

    _active_entity_views[id] = entity_view
    GD.print("EntityViewManager: Spawned visual for C# EntityID %s (Type: %s)" % [id, type])

func _on_entity_despawned(id: int) -> void:
    if _active_entity_views.has(id):
        var entity_view: EntityView = _active_entity_views[id]
        entity_view.queue_free()
        _active_entity_views.erase(id)
        GD.print("EntityViewManager: Despawned visual for C# EntityID %s" % id)
    else:
        push_warning("EntityViewManager: Attempted to despawn non-existent visual for C# EntityID %s" % id)

func _on_entity_moved(id: int, new_position: Vector2, new_rotation: float) -> void:
    if _active_entity_views.has(id):
        var entity_view: EntityView = _active_entity_views[id]
        entity_view.brain_target_position = new_position
        entity_view.visuals.rotation_degrees = new_rotation

## Conceptual handler for animation events from EntityView.
## In Chapter 1.12, this is where we would forward the event to the C# EventBus.
func _on_entity_view_anim_event(type: String, payload: Dictionary) -> void:
    GD.print("EntityViewManager: Caught anim_event from EntityView. Type: '%s', Payload: %s" % [type, payload])
    # Conceptual: GameManager.Instance.Events.Publish(new AnimationEvent { ID = entity_view.entity_id, Type = type, Payload = payload });
    pass
```

Save all C# and GDScript files. Run `Main.tscn`.

**Expected Output:**

1.  The game loads `Gameplay.tscn`.
2.  The NPC entity (ID 1) starts moving right and rotating.
3.  After ~3 seconds, you'll see in the Output console:
    *   `EntityView 1: Playing animation 'test_attack'.`
    *   Then, shortly after (at the 0.5s mark of the animation):
        *   `EntityView 1: Emitted anim_event: Type 'hit_frame', Payload {hitbox_id:test_hitbox_01}`
        *   `EntityViewManager: Caught anim_event from EntityView. Type: 'hit_frame', Payload: {hitbox_id:test_hitbox_01}`
    *   Finally: `EntityView 1: Animation 'test_attack' finished.`
    *   `EntityViewManager: Caught anim_event from EntityView. Type: 'animation_finished', Payload: {anim_name:test_attack}`

This confirms that:
*   The `AnimationPlayer` is correctly configured to call `_on_anim_track_event`.
*   `EntityView.gd` correctly emits its `anim_event` signal.
*   `EntityViewManager.gd` correctly catches this signal.

This pipeline is now ready for the formal C# to GDScript interop connection in the next chapter.

### Summary

You have successfully implemented the **Animation Event Protocol** for Sigilborne, establishing a precise synchronization mechanism between visual animations in the GDScript Body and simulation logic in the C# Brain. By standardizing the `anim_event` signal and configuring `AnimationPlayer` tracks to call `EntityView.gd` methods, you've created a reliable pipeline for triggering simulation events (like "hit frames") at exact moments within animations, adhering to TDD 16.5's specifications.

### Next Steps

The next crucial module will focus on the **Interop Layer**, where we will formalize the communication bridge between our C# Brain and GDScript Body, ensuring seamless and type-safe exchange of data and events.