## Chapter 1.12: Interop Layer - C# to GDScript Communication

The "Interop Layer" is the backbone of our hybrid architecture, enabling seamless communication between the C# Brain and the GDScript Body. TDD 01.3 specifies that the Brain does not call functions on the Body directly; instead, it emits **Signals**. This chapter will formalize how C# `Action<T>` delegates in our `EventBus` are exposed to GDScript, allowing GDScript to connect to them and react to state changes, strictly adhering to TDD 01.3's event bus contract.

### 1. Understanding the Interop Challenge

Godot's C# integration uses `P/Invoke` (Platform Invoke) to bridge the .NET runtime with the Godot engine's native C++ API. While C# can directly call GDScript methods using `Call()`, the TDD explicitly favors a **signal-based approach** for C# to GDScript communication.

**Why a Signal-Based Interop?**

*   **Decoupling**: The C# Brain doesn't need to know *which* GDScript node is listening or *how* it will react. It just broadcasts an event.
*   **Flexibility**: Multiple GDScript nodes can listen to the same C# event, allowing for easy expansion (e.g., UI, VFX, Audio all react to `OnPlayerHealthChanged`).
*   **Thread Safety**: Events emitted from the C# `EventBus` (which runs on the main thread during `FlushCommands()`) are safe for GDScript to receive and process.
*   **Maintainability**: Clear separation of responsibilities.

### 2. Exposing C# Events to GDScript

Godot's C# binding allows GDScript to connect to C# delegates (events) that are exposed as `public event Action<T>`.

**Steps:**

1.  **Define C# `Action`s**: Our `EventBus` already has these (e.g., `OnEntitySpawned`, `OnEntityMoved`).
2.  **Make `EventBus` Accessible**: The `GameManager` singleton makes `EventBus` globally accessible.
3.  **GDScript Connection**: GDScript uses `GameManager.Instance.Events.Connect("EventName", Callable(self, "_on_event_handler"))`.

Let's refine our `EventBus` and `GameManager` to make these connections as smooth as possible.

#### 2.1. Refine `EventBus.cs` for GDScript Compatibility

We need to ensure our `Action` parameters are simple types that Godot's C# binding can easily marshal to GDScript. `EntityID` is a struct, which works well. `Dictionary<string, Variant>` also works for payloads.

Open `_Brain/Core/EventBus.cs`:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // --- C# to C# / C# to GDScript Events ---
        // TDD 01.3: C# Action<T> delegates (Synchronous)
        // These are public events that GDScript can connect to.

        public event Action<string> OnGameStateChanged;

        // Entity Lifecycle Events (TDD 11.4)
        public event Action<EntityID, EntityType, string, Vector2, float> OnEntitySpawned; // Explicit parameters
        public event Action<EntityID> OnEntityDespawned;
        public event Action<EntityID, Vector2, float> OnEntityMoved;

        // Animation Event (TDD 16.5)
        public event Action<EntityID, string, Dictionary<string, Variant>> OnAnimationEvent; // Explicit parameters

        // --- Command Buffer for Main Thread Execution (TDD 13.3) ---
        private ConcurrentQueue<Action> _commandBuffer = new ConcurrentQueue<Action>();

        /// <summary>
        /// Publishes an event to all C# and GDScript subscribers.
        /// GDScript connects to these events by name (e.g., "OnEntitySpawned").
        /// </summary>
        /// <typeparam name="TEvent">The type of the event data struct.</typeparam>
        /// <param name="eventData">The event data struct.</param>
        public void Publish<TEvent>(TEvent eventData)
        {
            // The Publish method now explicitly unpacks event data structs
            // and invokes the corresponding Action with individual parameters.
            // This makes the GDScript connection cleaner (no need to unpack structs in GDScript).

            if (eventData is EntityManager.EntitySpawnedEvent spawnedEvent)
            {
                OnEntitySpawned?.Invoke(spawnedEvent.ID, spawnedEvent.Type, spawnedEvent.DefinitionID, spawnedEvent.InitialPosition, spawnedEvent.InitialRotation);
            }
            else if (eventData is EntityManager.EntityDespawnedEvent despawnedEvent)
            {
                OnEntityDespawned?.Invoke(despawnedEvent.ID);
            }
            else if (eventData is EntityManager.EntityMovedEvent movedEvent)
            {
                OnEntityMoved?.Invoke(movedEvent.ID, movedEvent.NewPosition, movedEvent.NewRotation);
            }
            else if (eventData is EntityManager.AnimationEvent animEvent)
            {
                OnAnimationEvent?.Invoke(animEvent.ID, animEvent.Type, animEvent.Payload);
            }
            else if (eventData is string gameState)
            {
                OnGameStateChanged?.Invoke(gameState);
            }
            else
            {
                GD.PrintErr($"EventBus: Attempted to publish unknown event type: {typeof(TEvent).Name}. Ensure it's handled in Publish<TEvent>.");
            }
        }

        // ... (AddCommand and FlushCommands methods unchanged) ...
    }
}
```

**Key Change**: The `Publish<TEvent>` method now explicitly unpacks the event data structs (e.g., `EntityManager.EntitySpawnedEvent`) and invokes the corresponding `Action` with its individual parameters. This is crucial because Godot's `Connect` method for C# events maps to the *parameters of the `Action`*, not the struct itself.

#### 2.2. Update `EntityManager.cs` to reflect `EntitySpawnedEvent` changes

We made `EntitySpawnedEvent` include `InitialPosition` and `InitialRotation` in the previous chapter. The `EntityManager.CreateEntity` method needs to pass these values.

Open `_Brain/Entities/EntityManager.cs` and ensure `CreateEntity` passes these:

```csharp
// _Brain/Entities/EntityManager.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;

namespace Sigilborne.Entities
{
    // ... (EntityID, EntityMeta, EntityType, EntityManager class) ...

    public class EntityManager
    {
        // ... (MAX_ENTITIES, _entityMetas, _freeIndices, _eventBus) ...

        public EntityManager(EventBus eventBus) { /* ... */ }

        public EntityID CreateEntity(EntityType type, string definitionID, Vector2 initialPosition = default, float initialRotation = 0f)
        {
            // ... (slot allocation, generation increment, EntityMeta setup) ...

            EntityID newId = new EntityID(index, _entityMetas[index].Generation);
            GD.Print($"EntityManager: Created {type} entity {newId} (Def: {definitionID})");

            // Publish with initial position and rotation
            _eventBus.Publish(new EntitySpawnedEvent { ID = newId, Type = type, DefinitionID = definitionID, InitialPosition = initialPosition, InitialRotation = initialRotation });

            return newId;
        }

        // ... (DestroyEntity, IsValid, GetEntityMeta methods) ...

        // --- Helper Events for Body Sync (TDD 11.4) ---
        // Note: These structs are just internal data carriers for the generic Publish method.
        // The actual Action delegates in EventBus define the signature GDScript connects to.
        public struct EntitySpawnedEvent { public EntityID ID; public EntityType Type; public string DefinitionID; public Vector2 InitialPosition; public float InitialRotation; }
        public struct EntityDespawnedEvent { public EntityID ID; }
        public struct EntityMovedEvent { public EntityID ID; public Vector2 NewPosition; public float NewRotation; }
        public struct AnimationEvent { public EntityID ID; public string Type; public Dictionary<string, Variant> Payload; }
    }
}
```

This change ensures the `EntitySpawnedEvent` carries the necessary `Vector2` and `float` data for initialization.

### 3. GDScript Connecting to C# Events

Now, our `EntityViewManager.gd` will connect to these C# events.

Open `res://_Body/Scripts/Visuals/EntityViewManager.gd`:

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
    GD.print("EntityViewManager: Initialized. Connecting to C# events...")
    
    # --- Connect to C# Events from GameManager.Instance.Events (TDD 01.3) ---
    # We use Callable(self, "_on_...") to bind the GDScript method.
    # The string names (e.g., "OnEntitySpawned") must match the C# event names in EventBus.
    # The parameters of the GDScript method must match the parameters of the C# Action delegate.
    
    # Check if GameManager and its Events are ready before connecting
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        GameManager.Instance.Events.OnEntitySpawned.connect(Callable(self, "_on_entity_spawned"))
        GameManager.Instance.Events.OnEntityDespawned.connect(Callable(self, "_on_entity_despawned"))
        GameManager.Instance.Events.OnEntityMoved.connect(Callable(self, "_on_entity_moved"))
        # Animation Events: EntityView's anim_event is local, then EntityViewManager forwards to C#
        # We will connect EntityView's anim_event to a C# receiving method in the next step.
        GD.print("EntityViewManager: Successfully connected to C# EventBus events.")
    else:
        push_error("EntityViewManager: GameManager or EventBus not ready! Cannot connect C# events.")
    
    # For testing, we'll add a timer to play an animation after entities are spawned.
    await get_tree().create_timer(3.0).timeout # Wait 3 seconds after scene loads

    for id in _active_entity_views:
        var entity_view: EntityView = _active_entity_views[id]
        if entity_view.entity_id == 1: # NPC entity has ID 1 from previous tests
            entity_view.play_animation("test_attack")
            break
    pass

## Handler for C# EntitySpawnedEvent.
## Parameters must match Action<EntityID, EntityType, string, Vector2, float> OnEntitySpawned in C#.
func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    if ENTITY_ROOT_SCENE == null:
        push_error("EntityViewManager: ENTITY_ROOT_SCENE not loaded!")
        return

    var entity_view: EntityView = ENTITY_ROOT_SCENE.instantiate()
    add_child(entity_view)
    
    entity_view.setup(id, initial_position, initial_rotation, definition_id)
    
    # Connect the EntityView's anim_event to our manager's conceptual handler
    entity_view.anim_event.connect(Callable(self, "_on_entity_view_anim_event"))

    _active_entity_views[id] = entity_view
    GD.print("EntityViewManager: Spawned visual for C# EntityID %s (Type: %s, Def: %s) at %s" % [id, type, definition_id, initial_position])

## Handler for C# EntityDespawnedEvent.
## Parameters must match Action<EntityID> OnEntityDespawned in C#.
func _on_entity_despawned(id: int) -> void:
    if _active_entity_views.has(id):
        var entity_view: EntityView = _active_entity_views[id]
        entity_view.queue_free()
        _active_entity_views.erase(id)
        GD.print("EntityViewManager: Despawned visual for C# EntityID %s" % id)
    else:
        push_warning("EntityViewManager: Attempted to despawn non-existent visual for C# EntityID %s" % id)

## Handler for C# EntityMovedEvent.
## Parameters must match Action<EntityID, Vector2, float> OnEntityMoved in C#.
func _on_entity_moved(id: int, new_position: Vector2, new_rotation: float) -> void:
    if _active_entity_views.has(id):
        var entity_view: EntityView = _active_entity_views[id]
        entity_view.brain_target_position = new_position
        entity_view.visuals.rotation_degrees = new_rotation # Directly set rotation for now. Interpolation can be added later.
    # else:
        # push_warning("EntityViewManager: Received move event for non-existent visual for C# EntityID %s" % id)

## Handler for animation events from EntityView.
## This will forward the event to the C# EventBus.
func _on_entity_view_anim_event(type: String, payload: Dictionary) -> void:
    GD.print("EntityViewManager: Caught anim_event from EntityView. Type: '%s', Payload: %s" % [type, payload])
    # This is where GDScript calls back into C#
    # TDD 01.3: GDScript to C# (Method Calls)
    # GameManager.Instance.Events.ReceiveAnimationEvent(entity_view.entity_id, type, payload)
    # For now, we'll just print, the C# receiving method will be implemented next.
    pass
```

**Key Changes in `EntityViewManager.gd`:**

*   **`_ready()`**: Now contains `GameManager.Instance.Events.OnEntitySpawned.connect(...)` calls. This is the core interop connection.
*   **GDScript Method Signatures**: The parameters for `_on_entity_spawned`, `_on_entity_despawned`, `_on_entity_moved` now precisely match the parameters of their corresponding C# `Action` delegates in `EventBus`.

### 4. GDScript Calling C# Methods

While C# to GDScript uses signals, GDScript can directly call C# methods. This is useful for forwarding input or specific requests from the Body to the Brain. TDD 01.3 mentions this pattern.

Let's implement a C# method in `EventBus` that `EntityViewManager.gd` can call to send animation events back to the Brain.

Open `_Brain/Core/EventBus.cs` and add this method:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events and command buffer) ...

        /// <summary>
        /// GDScript calls this method to send animation events back to the C# Brain.
        /// </summary>
        /// <param name="entityIdIndex">The EntityID.Index of the visual entity.</param>
        /// <param name="type">The type of animation event.</param>
        /// <param name="payload">Additional data as a Godot Dictionary.</param>
        public void ReceiveAnimationEvent(int entityIdIndex, string type, Dictionary<string, Variant> payload)
        {
            // We need to reconstruct the full EntityID for safety.
            // This assumes EntityManager is available and entityIdIndex is valid.
            // For now, let's just create a dummy EntityID for the event.
            // In a real scenario, you'd use GameManager.Instance.Entities.GetEntityMeta(entityIdIndex).Generation
            // to create a fully valid EntityID.
            EntityID id = new EntityID(entityIdIndex, 1); // Placeholder generation

            // Publish as an internal C# event
            Publish(new EntityManager.AnimationEvent { ID = id, Type = type, Payload = payload });
            GD.Print($"EventBus: C# received animation event from GDScript: EntityID Index {entityIdIndex}, Type '{type}', Payload {payload}");
        }

        // ... (Publish, AddCommand, FlushCommands methods) ...
    }
}
```

Now, modify `_Body/Scripts/Visuals/EntityViewManager.gd`'s `_on_entity_view_anim_event` to call this C# method:

```gdscript
# _Body/Scripts/Visuals/EntityViewManager.gd
class_name EntityViewManager extends Node

# ... (existing code) ...

func _on_entity_view_anim_event(type: String, payload: Dictionary) -> void:
    # Get the EntityID.Index from the EntityView that emitted the signal
    var entity_view: EntityView = get_tree().get_last_notified_signal_source()
    if entity_view == null or not entity_view is EntityView:
        push_error("EntityViewManager: Could not get source EntityView for anim_event.")
        return

    GD.print("EntityViewManager: Caught anim_event from EntityView %s. Type: '%s', Payload: %s. Forwarding to C#." % [entity_view.entity_id, type, payload])
    
    # --- GDScript Calling C# Method (TDD 01.3) ---
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        # Pass the entity's ID (index), event type, and payload.
        # Note: Godot automatically converts GDScript Dictionary to C# Dictionary<string, Variant>.
        GameManager.Instance.Events.ReceiveAnimationEvent(entity_view.entity_id, type, payload)
    else:
        push_error("EntityViewManager: GameManager or EventBus not ready! Cannot forward anim_event to C#.")
```

### 5. Testing the Full Interop Layer

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.

**Expected Output:**

*   The scene loads.
*   You'll see messages confirming C# and GDScript systems initialized and connected.
*   The NPC will move and rotate.
*   After ~3 seconds, the NPC's `test_attack` animation plays.
*   When the "hit\_frame" event triggers:
    *   `EntityView 1: Emitted anim_event: Type 'hit_frame', Payload {hitbox_id:test_hitbox_01}` (from `EntityView.gd`)
    *   `EntityViewManager: Caught anim_event from EntityView 1. Type: 'hit_frame', Payload: {hitbox_id:test_hitbox_01}. Forwarding to C#.` (from `EntityViewManager.gd`)
    *   `EventBus: C# received animation event from GDScript: EntityID Index 1, Type 'hit_frame', Payload {hitbox_id:test_hitbox_01}` (from `EventBus.cs`)

This confirms the complete bidirectional interop: C# events triggering GDScript visuals, and GDScript animation events calling C# logic.

### Summary

You have successfully implemented Sigilborne's Interop Layer, formalizing the communication bridge between the C# Brain and GDScript Body. By leveraging C# `Action<T>` delegates in the `EventBus` for Brain-to-Body signals and direct method calls for Body-to-Brain requests, you've established a seamless, type-safe, and decoupled communication pipeline. This crucial step adheres strictly to TDD 01.3's specifications, ensuring that simulation logic and visual presentation can evolve independently while remaining perfectly synchronized.

### Next Steps

The next module will focus on **Global State Management**, defining how the C# Brain maintains authoritative control over all game data, and how the GDScript Body consistently displays this data without owning it.