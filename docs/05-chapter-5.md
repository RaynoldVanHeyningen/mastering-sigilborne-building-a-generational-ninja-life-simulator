## Chapter 1.5: Entity Model - Lightweight ECS-lite in C#

In Sigilborne, our world will be teeming with thousands of dynamic elements: players, NPCs, animals, projectiles, items, and more. Managing these efficiently requires a robust "atomic unit" for our simulation. This chapter introduces a lightweight Entity-Component-System (ECS-lite) architecture in C#, as outlined in TDD 11. This approach prioritizes performance, flexibility, and data-oriented design, moving away from heavy Godot nodes for our core simulation data.

### 1. The Need for an ECS-lite

While Godot's Node system is excellent for visuals and hierarchical scenes, it's not ideal for managing thousands of pure data-driven entities in a simulation.

**Why ECS-lite in the Brain?**

*   **Performance**: Nodes carry a lot of overhead (scene tree management, signals, visual properties). Our C# entities will be pure data, making them cache-friendly and faster to process in bulk.
*   **Flexibility**: Entities are just IDs. Components (pure data) are attached to them, allowing for highly flexible combinations of behaviors without complex inheritance hierarchies.
*   **Data-Oriented Design (DOD)**: ECS encourages thinking about data first, leading to more efficient memory access and easier multithreading (TDD 13).
*   **Decoupling**: Simulation logic is completely separate from visual representation. An entity can exist in the Brain without a visual counterpart in the Body (e.g., an "unloaded" NPC, TDD 11.5).

### 2. Entity ID Structure

An "Entity" in our ECS-lite is simply an `EntityID`: a unique identifier, not an object or a class. As per TDD 11.2, it's a `readonly struct` for efficiency.

Let's create the `EntityID` struct.

1.  In `res://_Brain/Entities/`, create a new folder named `Components/`.
2.  In `res://_Brain/Entities/`, create a new C# script named `EntityID.cs`:

```csharp
// _Brain/Entities/EntityID.cs
using System;

namespace Sigilborne.Entities
{
    /// <summary>
    /// Represents a unique identifier for an entity in the ECS-lite.
    /// It's a readonly struct for efficiency and immutability.
    /// </summary>
    public readonly struct EntityID : IEquatable<EntityID>
    {
        // TDD 11.2: Index - The slot in the global entity array.
        public readonly int Index;
        // TDD 11.2: Generation - A version counter to detect "Use After Free" bugs.
        public readonly int Generation;

        // A special ID to represent a non-existent or invalid entity.
        public static readonly EntityID Invalid = new EntityID(-1, -1);

        public EntityID(int index, int generation)
        {
            Index = index;
            Generation = generation;
        }

        // --- IEquatable Implementation ---
        public bool Equals(EntityID other)
        {
            return Index == other.Index && Generation == other.Generation;
        }

        public override bool Equals(object obj)
        {
            return obj is EntityID other && Equals(other);
        }

        public override int GetHashCode()
        {
            return HashCode.Combine(Index, Generation);
        }

        public static bool operator ==(EntityID left, EntityID right)
        {
            return left.Equals(right);
        }

        public static bool operator !=(EntityID left, EntityID right)
        {
            return !(left == right);
        }

        public override string ToString()
        {
            return $"EntityID({Index}, Gen:{Generation})";
        }

        public bool IsValid()
        {
            return Index != -1 && Generation != -1;
        }
    }
}
```

**Explanation:**

*   **`readonly struct`**: Ensures immutability and memory efficiency, as structs are value types.
*   **`Index`**: Points to the entity's actual data in an array.
*   **`Generation`**: A crucial safety mechanism. When an entity is destroyed, its `Index` slot might be reused. Incrementing `Generation` for the new entity in that slot prevents old `EntityID` references from accidentally accessing the wrong (or new) entity.
*   **`IEquatable<EntityID>`**: Provides efficient comparisons for `EntityID`s.
*   **`Invalid`**: A static property for representing a null or non-existent entity.

### 3. The Entity Registry (EntityManager)

The `EntityManager` is the central component responsible for creating, destroying, and managing the lifecycle of `EntityID`s. It will also hold basic metadata about each entity. As per TDD 11.2, it avoids `Dictionary` for core lookup, preferring direct array access.

Create a new C# script `_Brain/Entities/EntityManager.cs`:

```csharp
// _Brain/Entities/EntityManager.cs
using Godot;
using System;
using System.Collections.Generic; // For List<int> and Dictionary (for sparse data)
using Sigilborne.Core; // For EventBus

namespace Sigilborne.Entities
{
    /// <summary>
    /// Represents the core metadata for an entity.
    /// This is stored directly in the EntityManager's array.
    /// </summary>
    public struct EntityMeta
    {
        public int Generation; // Current generation of the entity in this slot
        public bool IsActive;   // Whether this slot is currently in use
        public EntityType Type; // Player, NPC, Animal, Projectile, etc.
        public string DefinitionID; // Reference to static data (e.g., "goblin_scout", "player_default")
    }

    /// <summary>
    /// Defines the broad categories of entities in the simulation.
    /// </summary>
    public enum EntityType
    {
        Player,
        NPC,
        Animal,
        Projectile,
        Item,
        WorldObject, // Interactable objects like Chests, Doors
        Effect,      // Visual effects that are entities (e.g., a persistent poison cloud)
        Invalid
    }

    /// <summary>
    /// Manages the creation, destruction, and basic lifecycle of all entities.
    /// Provides direct O(1) access to entity metadata via Index.
    /// </summary>
    public class EntityManager
    {
        private const int MAX_ENTITIES = 10_000; // TDD 11.2: Fixed size entity array
        private EntityMeta[] _entityMetas = new EntityMeta[MAX_ENTITIES];
        private List<int> _freeIndices = new List<int>(MAX_ENTITIES); // List of available slots

        private EventBus _eventBus; // To emit events like OnEntitySpawned, OnEntityDespawned

        public EntityManager(EventBus eventBus)
        {
            _eventBus = eventBus;

            // Populate free indices initially
            for (int i = 0; i < MAX_ENTITIES; i++)
            {
                _freeIndices.Add(i);
            }
            GD.Print($"EntityManager: Initialized with {MAX_ENTITIES} slots.");
        }

        /// <summary>
        /// Creates a new entity and returns its unique EntityID.
        /// </summary>
        /// <param name="type">The type of entity to create.</param>
        /// <param name="definitionID">The ID of the static definition for this entity.</param>
        /// <returns>A valid EntityID if successful, otherwise EntityID.Invalid.</returns>
        public EntityID CreateEntity(EntityType type, string definitionID)
        {
            if (_freeIndices.Count == 0)
            {
                GD.PrintErr("EntityManager: No free entity slots available!");
                return EntityID.Invalid;
            }

            int index = _freeIndices[_freeIndices.Count - 1]; // Get last free index
            _freeIndices.RemoveAt(_freeIndices.Count - 1);    // Remove from free list

            // Increment generation for safety (TDD 11.2)
            _entityMetas[index].Generation++;
            _entityMetas[index].IsActive = true;
            _entityMetas[index].Type = type;
            _entityMetas[index].DefinitionID = definitionID;

            EntityID newId = new EntityID(index, _entityMetas[index].Generation);
            GD.Print($"EntityManager: Created {type} entity {newId} (Def: {definitionID})");

            // TDD 11.4: Emit OnEntitySpawned for Body sync
            _eventBus.Publish(new EntitySpawnedEvent { ID = newId, Type = type, DefinitionID = definitionID });

            return newId;
        }

        /// <summary>
        /// Destroys an entity, making its ID invalid and its slot available for reuse.
        /// </summary>
        /// <param name="id">The EntityID to destroy.</param>
        public void DestroyEntity(EntityID id)
        {
            if (!IsValid(id))
            {
                GD.PrintErr($"EntityManager: Attempted to destroy invalid entity {id}.");
                return;
            }

            int index = id.Index;
            _entityMetas[index].IsActive = false;
            // Generation is not incremented here, but when a new entity reuses the slot.
            // This ensures old IDs point to an "invalid" generation.

            _freeIndices.Add(index); // Add back to free list
            GD.Print($"EntityManager: Destroyed entity {id}.");

            // TDD 11.4: Emit OnEntityDespawned for Body sync
            _eventBus.Publish(new EntityDespawnedEvent { ID = id });
        }

        /// <summary>
        /// Checks if an EntityID is currently valid and refers to an active entity.
        /// </summary>
        /// <param name="id">The EntityID to validate.</param>
        /// <returns>True if the ID is valid and active, false otherwise.</returns>
        public bool IsValid(EntityID id)
        {
            if (!id.IsValid() || id.Index >= MAX_ENTITIES || id.Index < 0)
            {
                return false;
            }
            // TDD 11.2: Validation - Generation check is critical
            return _entityMetas[id.Index].IsActive && _entityMetas[id.Index].Generation == id.Generation;
        }

        /// <summary>
        /// Retrieves the metadata for a given EntityID.
        /// Throws an exception if the ID is invalid, so always call IsValid() first.
        /// </summary>
        public ref EntityMeta GetEntityMeta(EntityID id)
        {
            if (!IsValid(id))
            {
                throw new InvalidOperationException($"Attempted to get metadata for invalid entity {id}.");
            }
            return ref _entityMetas[id.Index];
        }

        // --- Helper Events for Body Sync (TDD 11.4) ---
        // These are simple structs to pass data through the EventBus.
        public struct EntitySpawnedEvent { public EntityID ID; public EntityType Type; public string DefinitionID; }
        public struct EntityDespawnedEvent { public EntityID ID; }
    }
}
```

**Explanation:**

*   **`EntityMeta` struct**: Stores essential metadata for each entity directly in the `_entityMetas` array.
*   **`EntityType` enum**: Categorizes our entities for better organization and system processing.
*   **`MAX_ENTITIES`**: A fixed-size array (`_entityMetas`) for fast O(1) access by index.
*   **`_freeIndices`**: A `List<int>` to quickly find available slots for new entities.
*   **`CreateEntity()`**:
    *   Finds a free index.
    *   Increments the `Generation` counter for that slot.
    *   Initializes `EntityMeta`.
    *   Returns a new `EntityID`.
    *   **Crucially**: Publishes an `EntitySpawnedEvent` via the `EventBus` so the GDScript Body can create a visual representation.
*   **`DestroyEntity()`**:
    *   Marks the slot as inactive.
    *   Adds the index back to `_freeIndices`.
    *   **Crucially**: Publishes an `EntityDespawnedEvent` for the GDScript Body to remove its visual representation.
*   **`IsValid()`**: The core validation method, using both `Index` and `Generation` to prevent stale references.
*   **`GetEntityMeta()`**: Provides safe, direct access to the `EntityMeta` struct.
*   **`EntitySpawnedEvent` / `EntityDespawnedEvent`**: These nested structs define the data payload for our `EventBus` signals, enabling the Body to react to entity lifecycle changes.

### 4. Integrating EntityManager into GameManager

Now, we need to make `EntityManager` a core system managed by our `GameManager`.

Open `_Brain/Core/GameManager.cs` and modify it:

1.  Add `using Sigilborne.Entities;` at the top.
2.  Add an `EntityManager` property.
3.  Initialize `EntityManager` in `InitializeSystems()`.

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities; // Add this using directive

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    public TimeSystem Time { get; private set; }
    public EventBus Events { get; private set; }
    public WorldSimulation World { get; private set; }
    public EntityManager Entities { get; private set; } // Add EntityManager property

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

        // --- Test Scene Loading (from previous chapter) ---
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
        // EntityManager doesn't have a Tick method, its operations are event/command driven.
    }

    private void InitializeSystems()
    {
        Events = new EventBus();
        GD.Print("  - EventBus initialized.");

        Time = new TimeSystem();
        GD.Print("  - TimeSystem initialized.");

        // Initialize EntityManager, passing the EventBus so it can publish events
        Entities = new EntityManager(Events); // Initialize EntityManager here
        GD.Print("  - EntityManager initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 5. Testing Entity Creation

Let's add a simple test to `GameManager` to create and destroy an entity.

Add the following to the end of `GameManager`'s `_Ready()` method (after the `SceneLoader` call):

```csharp
// _Brain/Core/GameManager.cs (inside _Ready method)
// ...
        // --- Test Entity Management ---
        GD.Print("\n--- Testing Entity Management ---");
        EntityID playerEntity = Entities.CreateEntity(EntityType.Player, "player_default");
        GD.Print($"Created Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        EntityID npcEntity = Entities.CreateEntity(EntityType.NPC, "goblin_grunt");
        GD.Print($"Created NPC: {npcEntity}. IsValid: {Entities.IsValid(npcEntity)}");
        
        Entities.DestroyEntity(playerEntity);
        GD.Print($"Destroyed Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        // Attempt to create a new entity, reusing the slot
        EntityID newPlayerEntity = Entities.CreateEntity(EntityType.Player, "player_reborn");
        GD.Print($"Created New Player: {newPlayerEntity}. IsValid: {Entities.IsValid(newPlayerEntity)}");
        GD.Print($"Old Player ID {playerEntity} now IsValid: {Entities.IsValid(playerEntity)}"); // Should be false!

        GD.Print("--- End Testing Entity Management ---\n");
// ...
```

Save all C# files and run `Main.tscn`. In the Output console, after the scene loads, you should see output similar to this:

```
--- Testing Entity Management ---
EntityManager: Created Player entity EntityID(0, Gen:1) (Def: player_default)
Created Player: EntityID(0, Gen:1). IsValid: True
EntityManager: Created NPC entity EntityID(1, Gen:1) (Def: goblin_grunt)
Created NPC: EntityID(1, Gen:1). IsValid: True
EntityManager: Destroyed entity EntityID(0, Gen:1).
Destroyed Player: EntityID(0, Gen:1). IsValid: False
EntityManager: Created Player entity EntityID(0, Gen:2) (Def: player_reborn)
Created New Player: EntityID(0, Gen:2). IsValid: True
Old Player ID EntityID(0, Gen:1) now IsValid: False
--- End Testing Entity Management ---
```

Notice how `EntityID(0, Gen:1)` becomes invalid after destruction, and the new entity reusing index `0` gets `Gen:2`. This confirms our generation counter mechanism is working as intended.

### Summary

You have successfully implemented the core Entity model for Sigilborne using a lightweight ECS-lite architecture in C#. This includes the `EntityID` struct for unique identification and the `EntityManager` for handling entity lifecycle. By integrating `EntityManager` into `GameManager` and confirming its functionality, you've established a performant, flexible, and data-oriented foundation for all simulation entities, strictly adhering to TDD 11's specifications.

### Next Steps

In the next chapter, we will build upon this by designing and implementing core components, focusing on `TransformComponent`, `TypeComponent`, and `StateComponent`, and explore how they are stored and accessed within our ECS-lite framework.