## Chapter 1.9: Scene Composition - Standardizing Godot Scenes

With our C# Brain's architecture taking shape, it's time to formalize how the GDScript Body visualizes its data. This chapter focuses on standardizing Godot Scene composition, particularly for entities, as outlined in TDD 16. The goal is to create a predictable, maintainable, and efficient structure where `EntityView.gd` acts as the sole communication point between the Brain's data and the Body's visuals. This prevents "Scene Spaghetti" and ensures a clean separation of concerns.

### 1. The "View" vs. "Controller" Philosophy

In our Brain & Body architecture:

*   **Brain (C#)**: Is the **Controller** (and Model). It dictates what *should* happen.
*   **Body (GDScript)**: Is the **View**. It displays what *is* happening.

This means Godot scenes and their scripts should primarily focus on:

*   **Visuals**: Sprites, animations, particles, UI.
*   **Audio**: Sound effects, music.
*   **Input Capture**: Forwarding raw input to the Brain.
*   **Interpolation**: Smoothly animating between Brain states.

**Scenes should NOT contain logic for:**

*   Calculating damage.
*   Determining AI decisions.
*   Managing inventory data.
*   Storing authoritative game state (e.g., actual health values).

### 2. The Canonical Entity Root Hierarchy

TDD 16.2 provides a strict template for every active entity scene. This consistency is vital for systems that need to generically interact with entity visuals (e.g., `EntityManager` spawning, `VFXManager` attaching particles).

Let's create a generic `EntityRoot` scene that other entity types (Player, NPC, Projectile) will inherit from or instantiate.

1.  In Godot, go to `Scene > New Scene`.
2.  Add a `CharacterBody2D` as the root node.
    *   Rename this node to `EntityRoot`.
    *   **Reasoning**: `CharacterBody2D` is ideal for entities that move and collide in 2D space, providing built-in `move_and_slide()` functionality.
3.  As a child of `EntityRoot`, add a `Node2D`.
    *   Rename this node to `Visuals`.
    *   **Reasoning**: This node will contain all visual elements (sprites, animations). Rotating or flipping this node will affect all visuals without changing the `CharacterBody2D`'s collision or physics.
4.  As a child of `Visuals`, add a `Sprite2D`.
    *   Rename this node to `Sprite`.
    *   **Reasoning**: The main visual representation.
5.  As a child of `Visuals`, add another `Node2D`.
    *   Rename this node to `VFX`.
    *   **Reasoning**: A dedicated attachment point for particle effects, ensuring they move and rotate with the main visual.
6.  As a child of `EntityRoot`, add a `CollisionShape2D`.
    *   Rename this node to `Collision`.
    *   **Reasoning**: Defines the entity's physical collision bounds. You'll need to assign a `Shape2D` (e.g., `RectangleShape2D`, `CapsuleShape2D`) to it in the Inspector.
7.  As a child of `EntityRoot`, add a `Node2D`.
    *   Rename this node to `Audio`.
    *   **Reasoning**: A dedicated attachment point for `AudioStreamPlayer2D` nodes, allowing sounds to originate from the entity's position.
8.  As a child of `EntityRoot`, add a `Control` node.
    *   Rename this node to `UI`.
    *   **Reasoning**: An attachment point for UI elements like health bars, name tags, or interaction prompts, ensuring they stay positioned relative to the entity.

Your scene tree for `EntityRoot.tscn` should look like this:

```
EntityRoot (CharacterBody2D)
├── Visuals (Node2D)
│   ├── Sprite (Sprite2D)
│   └── VFX (Node2D)
├── Collision (CollisionShape2D)
├── Audio (Node2D)
└── UI (Control)
```

Save this scene as `res://_Body/Scenes/Entities/EntityRoot.tscn`.

### 3. Script Responsibilities: `EntityView.gd`

TDD 16.2 explicitly states: "`EntityView.gd`: The ONLY script that talks to the Brain." This script is the entity's "puppet master" in the Body, listening to C# signals and updating the visual scene.

Create a new GDScript and attach it to the `EntityRoot` node in `res://_Body/Scenes/Entities/EntityRoot.tscn`.

1.  Select the `EntityRoot` node.
2.  Click the script icon in the Inspector.
3.  Create a new script:
    *   **Language**: `GDScript`.
    *   **Class Name**: `EntityView`.
    *   **Inherits**: `CharacterBody2D` (this should be auto-filled).
    *   **Path**: `res://_Body/Scripts/Visuals/EntityView.gd` (create `Visuals` folder if it doesn't exist).
    *   Click `Create`.

Open `EntityView.gd` and modify it:

```gdscript
# _Body/Scripts/Visuals/EntityView.gd
class_name EntityView extends CharacterBody2D

# --- Public Properties ---
# The EntityID from the C# Brain that this visual representation corresponds to.
var entity_id: int = -1 # Store as int for simplicity, but conceptually it's EntityID.Index

# --- Node References ---
@onready var visuals: Node2D = $Visuals
@onready var sprite: Sprite2D = $Visuals/Sprite
# @onready var animation_player: AnimationPlayer = $Visuals/AnimationPlayer # Add this if you have an AnimationPlayer
@onready var vfx_attachment_point: Node2D = $Visuals/VFX
@onready var audio_attachment_point: Node2D = $Audio
@onready var ui_attachment_point: Control = $UI

# --- Visual Interpolation Parameters (TDD 16.4) ---
# The target position received from the Brain.
var brain_target_position: Vector2 = Vector2.ZERO
# The smoothing speed for visual interpolation.
const SMOOTHING_SPEED: float = 10.0 # Adjust as needed

# --- Signals ---
# This signal is emitted when an animation event occurs.
# It's caught by the Brain (C#) via GameEvents.
signal anim_event(type: String, payload: Dictionary)


## Initializes this EntityView with its corresponding C# EntityID.
## This function is called by the C# Brain via the EventBus when an entity is spawned.
## (TDD 16.4: The "Setup" Function)
func setup(id: int, initial_position: Vector2, initial_rotation: float, definition_id: String) -> void:
    entity_id = id
    global_position = initial_position
    brain_target_position = initial_position
    visuals.rotation_degrees = initial_rotation
    
    # Example: Set sprite texture based on definition_id
    # var texture_path = "res://_Body/Art/Characters/%s.png" % definition_id
    # if ResourceLoader.exists(texture_path):
    #     sprite.texture = load(texture_path)
    # else:
    #     push_warning("EntityView: No sprite found for definition_id: %s" % definition_id)

    GD.print("EntityView: Setup for C# EntityID %s (Def: %s) at %s" % [entity_id, definition_id, global_position])
    
    # --- Connect to Brain Signals (TDD 16.4) ---
    # We will establish these connections more formally in Chapter 1.12 (Interop Layer).
    # For now, conceptual connections:
    # GameManager.Instance.Events.Connect("entity_moved", Callable(self, "_on_entity_moved"))
    # GameManager.Instance.Events.Connect("entity_health_changed", Callable(self, "_on_entity_health_changed"))
    pass

## Called every physics frame (fixed rate) to update the visual position.
## (TDD 16.4: Visual Interpolation)
func _physics_process(delta: float) -> void:
    # Interpolate towards the Brain's authoritative target position.
    # We use global_position here to affect the CharacterBody2D directly.
    global_position = global_position.lerp(brain_target_position, delta * SMOOTHING_SPEED)
    
    # If this entity needs to send input to the Brain, it would do so here.
    # Example: if entity_id == GameManager.Instance.Player.PlayerEntityID:
    #    InputManager.instance.send_player_input_to_brain()
    pass

## Placeholder for reacting to C# entity movement signals.
## This method would be connected to an event from the C# Brain.
func _on_entity_moved(id: int, new_position: Vector2, new_rotation: float) -> void:
    if entity_id == id:
        brain_target_position = new_position
        visuals.rotation_degrees = new_rotation
        # GD.print("EntityView %s: Received new position %s" % [entity_id, new_position])
```

**Explanation of `EntityView.gd`:**

*   **`entity_id: int`**: This `EntityView` stores the `Index` of its corresponding C# `EntityID`.
*   **`@onready var ...`**: Efficiently gets references to child nodes.
*   **`brain_target_position`**: This `Vector2` stores the *last authoritative position* received from the C# Brain.
*   **`_physics_process(delta: float)`**: This is where `Visual Interpolation` happens (TDD 16.4). The `EntityView` smoothly moves towards `brain_target_position`, providing fluid visuals even if the C# Brain updates at a lower tick rate.
*   **`setup(id: int, ...)`**: This is the "Setup" function (TDD 16.4). The C# Brain will call this via the `EventBus` when it spawns a new entity, passing its `EntityID` and initial state.
*   **`_on_entity_moved(...)`**: A placeholder for a signal handler. In Chapter 1.12, we'll connect this to a C# event that the `TransformSystem` will emit.

### 4. Integrating EntityView with the C# Brain

For the `EntityView` to be useful, the C# Brain needs to:
1.  **Emit events** when entities are spawned, moved, or destroyed.
2.  The GDScript `EntityView` needs to **listen** to these events.

We already set up `EntityManager` to emit `EntitySpawnedEvent` and `EntityDespawnedEvent`. We will modify `TransformSystem` to emit `EntityMovedEvent`.

First, let's define the `EntityMovedEvent` struct in `_Brain/Entities/EntityManager.cs` so our `EventBus` can handle it:

```csharp
// _Brain/Entities/EntityManager.cs (inside Sigilborne.Entities namespace)
// ...
        // --- Helper Events for Body Sync (TDD 11.4) ---
        public struct EntitySpawnedEvent { public EntityID ID; public EntityType Type; public string DefinitionID; public Vector2 InitialPosition; public float InitialRotation; }
        public struct EntityDespawnedEvent { public EntityID ID; }
        public struct EntityMovedEvent { public EntityID ID; public Vector2 NewPosition; public float NewRotation; } // Add this struct
    }
}
```
**Note**: We updated `EntitySpawnedEvent` to include initial position and rotation, as the `EntityView.gd` `setup` function now expects it.

Now, modify `_Brain/Systems/TransformSystem.cs` to emit this event:

```csharp
// _Brain/Systems/TransformSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;

namespace Sigilborne.Systems
{
    public class TransformSystem
    {
        private Dictionary<EntityID, TransformComponent> _transforms = new Dictionary<EntityID, TransformComponent>();
        private EntityManager _entityManager;
        private EventBus _eventBus; // Store EventBus reference

        public TransformSystem(EntityManager entityManager, EventBus eventBus)
        {
            _entityManager = entityManager;
            _eventBus = eventBus; // Store EventBus reference
            eventBus.OnEntitySpawned += OnEntitySpawned;
            eventBus.OnEntityDespawned += OnEntityDespawned;
            GD.Print("TransformSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityManager.EntitySpawnedEvent e)
        {
            // All entities get a TransformComponent by default for simplicity in this example
            _transforms.Add(e.ID, new TransformComponent(e.InitialPosition, e.InitialRotation)); // Use initial position from event
            GD.Print($"TransformSystem: Added transform for {e.ID}");
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            if (_transforms.ContainsKey(e.ID))
            {
                _transforms.Remove(e.ID);
                GD.Print($"TransformSystem: Removed transform for {e.ID}");
            }
        }

        // ... (TryGetTransform, TrySetTransform, GetTransformRef methods) ...

        public void Tick(double delta)
        {
            // Example: Simple movement for all entities with a transform
            foreach (var kvp in _transforms)
            {
                EntityID id = kvp.Key;
                ref TransformComponent transform = ref _transforms.GetValueRef(id);

                // Simulate simple movement for testing: move right
                transform.Position += new Vector2(10 * (float)delta, 0); // Move 10 units/sec to the right
                transform.RotationDegrees += 50 * (float)delta; // Rotate 50 degrees/sec

                // TDD 11.4: Emit OnEntityMoved for Body sync
                _eventBus.Publish(new EntityManager.EntityMovedEvent { ID = id, NewPosition = transform.Position, NewRotation = transform.RotationDegrees });
            }
        }
    }
}
```

Now, we need to adjust `GameManager.CreateEntity` to pass initial position and rotation.

```csharp
// _Brain/Core/GameManager.cs (inside CreateEntity and test section)
// ...
        public EntityID CreateEntity(EntityType type, string definitionID, Vector2 initialPosition = default, float initialRotation = 0f)
        {
            if (_freeIndices.Count == 0)
            {
                GD.PrintErr("EntityManager: No free entity slots available!");
                return EntityID.Invalid;
            }

            int index = _freeIndices[_freeIndices.Count - 1];
            _freeIndices.RemoveAt(_freeIndices.Count - 1);

            _entityMetas[index].Generation++;
            _entityMetas[index].IsActive = true;
            _entityMetas[index].Type = type;
            _entityMetas[index].DefinitionID = definitionID;

            EntityID newId = new EntityID(index, _entityMetas[index].Generation);
            GD.Print($"EntityManager: Created {type} entity {newId} (Def: {definitionID})");

            // Publish with initial position and rotation
            _eventBus.Publish(new EntitySpawnedEvent { ID = newId, Type = type, DefinitionID = definitionID, InitialPosition = initialPosition, InitialRotation = initialRotation });

            return newId;
        }
// ...
// Inside GameManager._Ready():
// ...
        // --- Test Entity Management & Components ---
        GD.Print("\n--- Testing Entity Management & Components ---");
        // Pass initial position and rotation
        EntityID playerEntity = Entities.CreateEntity(EntityType.Player, "player_default", new Vector2(200, 200), 0f);
        GD.Print($"Created Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        // The TransformSystem will pick up this initial position from the EntitySpawnedEvent.
        // No need to manually set it here, as Transforms.TrySetTransform will override.
        // Let's remove the manual set/get for playerTransform here to show event-driven init.
        // The TransformSystem.Tick will then immediately start moving it.

        EntityID npcEntity = Entities.CreateEntity(EntityType.NPC, "goblin_grunt", new Vector2(100, 100), 90f);
        GD.Print($"Created NPC: {npcEntity}. IsValid: {Entities.IsValid(npcEntity)}");
        
        Entities.DestroyEntity(playerEntity);
        GD.Print($"Destroyed Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        if (!Transforms.TryGetTransform(playerEntity, out TransformComponent destroyedTransform))
        {
            GD.Print($"Attempted to get transform for destroyed entity {playerEntity}, it correctly failed.");
        }

        GD.Print("--- End Testing Entity Management & Components ---\n");
// ...
```

Finally, we need to modify `EventBus.cs` to declare the new event types.

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities; // Add this using directive for EntityID, EntityType, Vector2

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing OnGameStateChanged event) ...

        // Declare events for entity lifecycle and movement
        public event Action<EntityManager.EntitySpawnedEvent> OnEntitySpawned;
        public event Action<EntityManager.EntityDespawnedEvent> OnEntityDespawned;
        public event Action<EntityManager.EntityMovedEvent> OnEntityMoved; // New event

        private ConcurrentQueue<Action> _commandBuffer = new ConcurrentQueue<Action>();

        public void Publish<TEvent>(TEvent eventData)
        {
            // Use specific event types for publishing
            if (eventData is EntityManager.EntitySpawnedEvent spawnedEvent)
            {
                OnEntitySpawned?.Invoke(spawnedEvent);
            }
            else if (eventData is EntityManager.EntityDespawnedEvent despawnedEvent)
            {
                OnEntityDespawned?.Invoke(despawnedEvent);
            }
            else if (eventData is EntityManager.EntityMovedEvent movedEvent) // New condition
            {
                OnEntityMoved?.Invoke(movedEvent);
            }
            else if (eventData is string gameState) // Existing example
            {
                OnGameStateChanged?.Invoke(gameState);
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

### 5. Connecting the Body to the Brain's Events (Conceptually)

The actual connection between C# `Action` delegates and GDScript signals is handled by the Interop Layer (Chapter 1.12). For now, conceptually, `EntityView.gd` would connect to these events.

We can illustrate this by adding a placeholder `EntityViewManager.gd` to manage the spawning and despawning of `EntityView` scenes.

Create `res://_Body/Scripts/Visuals/EntityViewManager.gd`:

```gdscript
# _Body/Scripts/Visuals/EntityViewManager.gd
class_name EntityViewManager extends Node

# --- Singleton Instance ---
static var instance: EntityViewManager

# Dictionary to hold active EntityView instances, mapped by C# EntityID.Index
var _active_entity_views: Dictionary = {}

# Preload our generic EntityRoot scene
const ENTITY_ROOT_SCENE: PackedScene = preload("res://_Body/Scenes/Entities/EntityRoot.tscn")

func _init():
    if instance != null:
        push_error("EntityViewManager: More than one instance detected!")
        queue_free()
        return
    instance = self

func _ready():
    # Connect to C# Events from GameManager.Instance.Events (will be done in Chapter 1.12)
    # For now, we'll simulate the connection.
    GD.print("EntityViewManager: Initialized. Waiting for C# entity events.")
    # This is a conceptual connection. In Chapter 1.12, we'll write the C# side to expose these.
    # GameManager.Instance.Events.OnEntitySpawned += Callable(self, "_on_entity_spawned")
    # GameManager.Instance.Events.OnEntityDespawned += Callable(self, "_on_entity_despawned")
    # GameManager.Instance.Events.OnEntityMoved += Callable(self, "_on_entity_moved")
    pass

## Handler for C# EntitySpawnedEvent.
func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    if ENTITY_ROOT_SCENE == null:
        push_error("EntityViewManager: ENTITY_ROOT_SCENE not loaded!")
        return

    var entity_view: EntityView = ENTITY_ROOT_SCENE.instantiate()
    add_child(entity_view) # Add to the scene tree
    
    # Call the setup function on the EntityView
    entity_view.setup(id, initial_position, initial_rotation, definition_id)
    
    _active_entity_views[id] = entity_view
    GD.print("EntityViewManager: Spawned visual for C# EntityID %s (Type: %s)" % [id, type])

## Handler for C# EntityDespawnedEvent.
func _on_entity_despawned(id: int) -> void:
    if _active_entity_views.has(id):
        var entity_view: EntityView = _active_entity_views[id]
        entity_view.queue_free() # Remove from scene tree
        _active_entity_views.erase(id)
        GD.print("EntityViewManager: Despawned visual for C# EntityID %s" % id)
    else:
        push_warning("EntityViewManager: Attempted to despawn non-existent visual for C# EntityID %s" % id)

## Handler for C# EntityMovedEvent.
func _on_entity_moved(id: int, new_position: Vector2, new_rotation: float) -> void:
    if _active_entity_views.has(id):
        var entity_view: EntityView = _active_entity_views[id]
        entity_view.brain_target_position = new_position
        entity_view.visuals.rotation_degrees = new_rotation # Directly set rotation for now. Interpolation can be added later.
    # else:
        # push_warning("EntityViewManager: Received move event for non-existent visual for C# EntityID %s" % id)
```

Now, add `EntityViewManager` to `Main.tscn` as a child of `Main`:

```
Main (Node)
├── GameManager (GameManager.cs)
├── SceneLoader (SceneLoader.gd)
└── EntityViewManager (EntityViewManager.gd)
```

Finally, let's test this.

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.

**Expected Output:**

*   The game will load `Gameplay.tscn`.
*   In the Output console, you'll see:
    *   `EntityViewManager: Spawned visual for C# EntityID 0 (Type: 0)` (Player)
    *   `EntityViewManager: Spawned visual for C# EntityID 1 (Type: 1)` (NPC)
    *   `EntityViewManager: Despawned visual for C# EntityID 0` (Player)
*   On the `Gameplay.tscn` scene, you should see one `EntityRoot` (the NPC) moving to the right and rotating. The player entity will briefly appear and then be removed.

This confirms the `EntityViewManager` is correctly spawning and despawning `EntityView` instances based on C# events, and the `TransformSystem` is driving their movement.

### Summary

You have successfully standardized Godot Scene composition for entities, defining a canonical `EntityRoot` hierarchy and implementing `EntityView.gd` as the dedicated communication interface for the Body. By integrating `EntityViewManager.gd` to dynamically spawn, manage, and despawn these visual representations based on C# events, you've established a robust, reactive presentation layer that adheres strictly to TDD 16's specifications. This setup ensures a clean separation between simulation logic and visual feedback, paving the way for efficient development and high performance.

### Next Steps

In the next chapter, we will formalize our naming conventions across all project assets and scripts, ensuring consistency and readability, which are crucial for maintaining a large-scale project like Sigilborne.