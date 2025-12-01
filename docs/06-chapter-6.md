## Chapter 1.6: Component Architecture - Composition over Inheritance

Building upon our `EntityID` and `EntityManager`, this chapter dives into the "Component" aspect of our ECS-lite architecture. Components are pure data structures that describe an entity's attributes. By attaching various components to an `EntityID`, we define its behavior and properties through composition rather than complex inheritance chains. This approach, as per TDD 11.3, promotes flexibility, reusability, and efficient data processing.

We will design and implement `TransformComponent`, `TypeComponent`, and `StateComponent` as our foundational components.

### 1. Understanding Component-Based Design

In an ECS-lite, an entity is just an ID. Its capabilities come from the components attached to it.

**Key Principles of Components:**

*   **Pure Data**: Components should primarily contain data, not logic. Logic resides in "Systems" that operate on components.
*   **Small and Focused**: Each component should represent a single, clear aspect of an entity (e.g., `TransformComponent` for position, `HealthComponent` for health).
*   **Composition**: An entity gets its behavior by composing multiple components. A `Player` entity might have `TransformComponent`, `HealthComponent`, `InputComponent`, and `InventoryComponent`. A `Projectile` might only have `TransformComponent` and `DamageComponent`.
*   **Reusability**: Components can be reused across vastly different entity types.
*   **Cache-Friendliness**: Storing components of the same type together (e.g., all `TransformComponents` in one array) improves CPU cache performance when systems iterate over them.

### 2. Component Storage Strategy

TDD 11.3 outlines two primary patterns for component storage:

*   **Structure of Arrays (SoA)**: For "hot path" components that are frequently iterated over (like `TransformComponent`, `VelocityComponent`), we store them in large arrays. This is excellent for cache locality.
*   **Dictionary**: For "sparse" components that only a subset of entities possess (like `InventoryComponent`, `ManaComponent`), a `Dictionary<EntityID, ComponentData>` is more memory-efficient.

For our core components, `TransformComponent` will likely be SoA-like (stored in a system that iterates all transforms), while `TypeComponent` and `StateComponent` will be direct metadata or part of the `EntityManager`'s internal structure.

Let's create the `_Brain/Entities/Components` folder if you haven't already.

### 3. Core Components Implementation

#### 3.1 `TransformComponent.cs`

Every entity in our game world will have a position and rotation. This component is fundamental for movement, rendering, and spatial queries.

Create `res://_Brain/Entities/Components/TransformComponent.cs`:

```csharp
// _Brain/Entities/Components/TransformComponent.cs
using Godot; // For Vector2

namespace Sigilborne.Entities.Components
{
    /// <summary>
    /// Stores the position and rotation of an entity in world space.
    /// This is a common component for all spatially aware entities.
    /// </summary>
    public struct TransformComponent
    {
        public Vector2 Position;
        public float RotationDegrees; // Rotation in degrees

        public TransformComponent(Vector2 position, float rotationDegrees = 0f)
        {
            Position = position;
            RotationDegrees = rotationDegrees;
        }

        public override string ToString()
        {
            return $"Pos: {Position}, Rot: {RotationDegrees}°";
        }
    }
}
```

**Note**: We use Godot's `Vector2` struct directly here. While the TDD mentions the Brain should be headless, `Vector2` is a fundamental mathematical type for 2D games, and its inclusion doesn't tie the Brain to Godot's visual nodes.

#### 3.2 `TypeComponent.cs`

This component defines the broad category of an entity. We've already implicitly defined this with `EntityType` in `EntityManager.cs`, so we can consider `EntityType` itself as the `TypeComponent` data, stored directly within `EntityMeta`.

To formalize this, ensure your `EntityManager.cs` explicitly uses the `EntityType` enum. (We already did this in Chapter 1.5).

```csharp
// Excerpt from _Brain/Entities/EntityManager.cs
// ...
    public struct EntityMeta
    {
        public int Generation;
        public bool IsActive;
        public EntityType Type; // This is our TypeComponent data
        public string DefinitionID;
    }
// ...
    public enum EntityType // This enum defines the possible types
    {
        Player,
        NPC,
        Animal,
        Projectile,
        Item,
        WorldObject,
        Effect,
        Invalid
    }
// ...
```

#### 3.3 `StateComponent.cs`

This component holds various boolean flags that describe an entity's current state (e.g., active, sleeping, dead).

Create `res://_Brain/Entities/Components/StateComponent.cs`:

```csharp
// _Brain/Entities/Components/StateComponent.cs
namespace Sigilborne.Entities.Components
{
    /// <summary>
    /// Stores various boolean flags representing an entity's current state.
    /// </summary>
    public struct StateComponent
    {
        public bool IsActive;    // Whether the entity is currently active in the simulation (beyond EntityManager's IsActive)
        public bool IsSleeping;  // For NPCs/Animals
        public bool IsDead;      // For all living entities
        public bool IsStunned;   // For combat
        public bool IsInvisible; // For stealth

        public StateComponent(bool isActive = true)
        {
            IsActive = isActive;
            IsSleeping = false;
            IsDead = false;
            IsStunned = false;
            IsInvisible = false;
        }

        public override string ToString()
        {
            return $"Active: {IsActive}, Sleeping: {IsSleeping}, Dead: {IsDead}, Stunned: {IsStunned}, Invisible: {IsInvisible}";
        }
    }
}
```

### 4. Component Storage and Retrieval within Systems

Now, let's consider how a "System" would manage these components. For `TransformComponent`, which almost all entities will have, a dedicated `TransformSystem` would store them efficiently. For `StateComponent`, we can store it similarly, or integrate some flags directly into `EntityMeta` for very common states.

Let's create a placeholder `TransformSystem` to demonstrate component management.

Create `res://_Brain/Systems/TransformSystem.cs`:

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
    /// <summary>
    /// Manages all TransformComponents for active entities.
    /// Example of a system that holds its own component data.
    /// </summary>
    public class TransformSystem
    {
        // TDD 11.3: SoA Pattern - Dictionary for sparse data, but still efficient for many entities.
        // For very high performance (millions of entities), this might be a NativeArray<TransformComponent>
        // and a separate NativeArray<EntityID> for indices.
        private Dictionary<EntityID, TransformComponent> _transforms = new Dictionary<EntityID, TransformComponent>();
        private EntityManager _entityManager; // Reference to validate EntityIDs

        public TransformSystem(EntityManager entityManager, EventBus eventBus)
        {
            _entityManager = entityManager;
            // Subscribe to entity lifecycle events to manage transforms
            eventBus.OnEntitySpawned += OnEntitySpawned;
            eventBus.OnEntityDespawned += OnEntityDespawned;
            GD.Print("TransformSystem: Initialized.");
        }

        // --- Event Handlers for Entity Lifecycle ---
        private void OnEntitySpawned(EntityManager.EntitySpawnedEvent e)
        {
            // All entities get a TransformComponent by default for simplicity in this example
            // In a real game, only spatially aware entities would get one.
            _transforms.Add(e.ID, new TransformComponent(Vector2.Zero)); // Default position
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

        // --- Public API for Systems to Access/Modify Transforms ---

        /// <summary>
        /// Attempts to get the TransformComponent for a given entity.
        /// </summary>
        /// <returns>True if the component exists and is retrieved, false otherwise.</returns>
        public bool TryGetTransform(EntityID id, out TransformComponent transform)
        {
            if (!_entityManager.IsValid(id))
            {
                transform = default;
                return false;
            }
            return _transforms.TryGetValue(id, out transform);
        }

        /// <summary>
        /// Attempts to set the TransformComponent for a given entity.
        /// </summary>
        /// <returns>True if the component exists and is set, false otherwise.</returns>
        public bool TrySetTransform(EntityID id, TransformComponent transform)
        {
            if (!_entityManager.IsValid(id) || !_transforms.ContainsKey(id))
            {
                return false;
            }
            _transforms[id] = transform;
            return true;
        }

        /// <summary>
        /// Provides a mutable reference to the TransformComponent for direct modification.
        /// Use with caution and ensure the ID is valid.
        /// </summary>
        /// <returns>A ref to the TransformComponent.</returns>
        /// <exception cref="InvalidOperationException">If the entity is invalid or doesn't have a transform.</exception>
        public ref TransformComponent GetTransformRef(EntityID id)
        {
            if (!_entityManager.IsValid(id) || !_transforms.ContainsKey(id))
            {
                throw new InvalidOperationException($"Entity {id} is invalid or does not have a TransformComponent.");
            }
            return ref _transforms.GetValueRef(id); // GetValueRef is a C# 7.0+ feature for ref return from Dictionary
        }

        // --- Tick Method (if this system needs to update transforms) ---
        public void Tick(double delta)
        {
            // Example: A simple movement system would iterate here
            // foreach (var kvp in _transforms)
            // {
            //     // Get velocity component from another system
            //     // kvp.Value.Position += velocity * delta;
            // }
        }
    }
}
```

**Note on `GetValueRef()`**: This is a C# 7.0+ feature that allows returning a `ref` to a value in a `Dictionary`. If you're using an older C# version or prefer not to use `ref` returns, you would get a copy of the struct, modify it, and then set it back: `_transforms[id] = modifiedTransform;`.

### 5. Integrating TransformSystem into GameManager

Now, let's add `TransformSystem` to our `GameManager`.

Open `_Brain/Core/GameManager.cs` and modify it:

1.  Add `using Sigilborne.Systems;` at the top.
2.  Add a `TransformSystem` property.
3.  Initialize `TransformSystem` in `InitializeSystems()`, passing `Entities` and `Events`.
4.  Call `TransformSystem.Tick(delta)` in `_PhysicsProcess`.

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems; // Add this using directive

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    public TimeSystem Time { get; private set; }
    public EventBus Events { get; private set; }
    public WorldSimulation World { get; private set; }
    public EntityManager Entities { get; private set; }
    public TransformSystem Transforms { get; private set; } // Add TransformSystem property

    public override void _Ready()
    {
        if (Instance != null)
        {
            GD.PrintErr("GameManager: More than one instance detected! Destroying duplicate.");
            QueueFree();
            return;
        }
        Instance = this;
        
        InitializeSystems();

        GD.Print("GameManager (C#) initialized and ready.");

        // --- Test Scene Loading ---
        if (SceneLoader.instance != null)
        {
            GD.Print("GameManager: Requesting SceneLoader to load Gameplay scene.");
            SceneLoader.instance.load_level("res://_Body/Scenes/Gameplay.tscn");
        }
        else
        {
            GD.PrintErr("GameManager: SceneLoader instance not found! Cannot load gameplay scene.");
        }
        // --- End Test Scene Loading ---
    }

    public override void _PhysicsProcess(double delta)
    {
        Time.Tick(delta);
        Events.FlushCommands();
        World.Tick(delta);
        Transforms.Tick(delta); // Call TransformSystem's tick method
    }

    private void InitializeSystems()
    {
        Events = new EventBus();
        GD.Print("  - EventBus initialized.");

        Time = new TimeSystem();
        GD.Print("  - TimeSystem initialized.");

        Entities = new EntityManager(Events);
        GD.Print("  - EntityManager initialized.");

        // Initialize TransformSystem, passing EntityManager and EventBus
        Transforms = new TransformSystem(Entities, Events); // Initialize TransformSystem here
        GD.Print("  - TransformSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 6. Testing Component Attachment and Retrieval

Let's modify our entity testing in `GameManager._Ready()` to demonstrate setting and getting a `TransformComponent`.

```csharp
// _Brain/Core/GameManager.cs (inside _Ready method, replace previous test entity code)
// ...
        // --- Test Entity Management & Components ---
        GD.Print("\n--- Testing Entity Management & Components ---");
        EntityID playerEntity = Entities.CreateEntity(EntityType.Player, "player_default");
        GD.Print($"Created Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        // Set player's initial position
        Transforms.TrySetTransform(playerEntity, new TransformComponent(new Vector2(100, 50), 45f));
        if (Transforms.TryGetTransform(playerEntity, out TransformComponent playerTransform))
        {
            GD.Print($"Player {playerEntity} Transform: {playerTransform}");
        }

        // Get a ref and modify directly
        ref TransformComponent playerTransformRef = ref Transforms.GetTransformRef(playerEntity);
        playerTransformRef.Position = new Vector2(150, 75);
        playerTransformRef.RotationDegrees = 90f;
        GD.Print($"Player {playerEntity} New Transform (via ref): {Transforms.GetTransformRef(playerEntity)}");


        EntityID npcEntity = Entities.CreateEntity(EntityType.NPC, "goblin_grunt");
        GD.Print($"Created NPC: {npcEntity}. IsValid: {Entities.IsValid(npcEntity)}");
        // NPCs also get a default transform via OnEntitySpawned

        Entities.DestroyEntity(playerEntity);
        GD.Print($"Destroyed Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        // Attempt to get transform for destroyed entity (should fail gracefully)
        if (!Transforms.TryGetTransform(playerEntity, out TransformComponent destroyedTransform))
        {
            GD.Print($"Attempted to get transform for destroyed entity {playerEntity}, it correctly failed.");
        }

        GD.Print("--- End Testing Entity Management & Components ---\n");
// ...
```

Save all C# files and run `Main.tscn`. The output should now reflect the component creation, modification, and proper removal upon entity destruction.

```
--- Testing Entity Management & Components ---
EntityManager: Created Player entity EntityID(0, Gen:1) (Def: player_default)
TransformSystem: Added transform for EntityID(0, Gen:1)
Created Player: EntityID(0, Gen:1). IsValid: True
Player EntityID(0, Gen:1) Transform: Pos: (100, 50), Rot: 45°
Player EntityID(0, Gen:1) New Transform (via ref): Pos: (150, 75), Rot: 90°
EntityManager: Created NPC entity EntityID(1, Gen:1) (Def: goblin_grunt)
TransformSystem: Added transform for EntityID(1, Gen:1)
Created NPC: EntityID(1, Gen:1). IsValid: True
EntityManager: Destroyed entity EntityID(0, Gen:1).
TransformSystem: Removed transform for EntityID(0, Gen:1)
Destroyed Player: EntityID(0, Gen:1). IsValid: False
Attempted to get transform for destroyed entity EntityID(0, Gen:1), it correctly failed.
--- End Testing Entity Management & Components ---
```

### Summary

You have successfully implemented the foundational components (`TransformComponent`, `TypeComponent`, `StateComponent`) for Sigilborne's ECS-lite architecture, emphasizing composition over inheritance. By designing a `TransformSystem` that manages `TransformComponent` data and integrates with `EntityManager`'s lifecycle events, you've established a flexible and efficient system for defining entity properties. This crucial step adheres to TDD 11's specifications, providing a scalable and performant way to manage thousands of entities in our simulation.

### Next Steps

With our core entity and component architecture in place, the next module will tackle the Job System and Concurrency, enabling our Brain to perform heavy tasks like ecology simulation and world generation across multiple threads without impacting performance.