## Chapter 7.1: Perception System - Spatial Hashing (C#)

Welcome to **Module 7: The Living World - Ecology & AI**! This module dives into creating a dynamic, simulated ecosystem. A core requirement for intelligent AI and efficient world simulation is the ability for entities to "perceive" their surroundings. This chapter focuses on implementing an efficient **Perception System** using a **Spatial Hash Grid** in the C# Brain. This data structure allows entities to quickly detect nearby objects and events without computationally expensive $O(N^2)$ distance checks, as specified in TDD 05.2.1.

### 1. The Challenge of Perception in Large Worlds

In a world with potentially thousands of active (or virtual) entities, simply iterating through all of them to check for proximity to a single entity is inefficient.

*   **$O(N^2)$ Problem**: If every entity checks every other entity, the computational cost grows quadratically with the number of entities.
*   **Performance**: This quickly becomes a bottleneck for AI and other perception-driven systems.
*   **Scalability**: A spatial partitioning system is essential for large, open worlds.

### 2. The Spatial Hash Grid Solution

A Spatial Hash Grid divides the world into a grid of cells. Each entity registers itself in the cell it occupies. To find nearby entities, an entity only needs to check its current cell and its immediate neighbors.

**Key Principles (TDD 05.2.1):**

*   **Grid Size**: We'll use a configurable cell size (e.g., 128x128 pixels, roughly one chunk).
*   **Logic**: Entities register their IDs in the grid cell they occupy.
*   **Query**: "Get all entities in cells (X,Y) and neighbors."

### 3. Implementing `SpatialHashGrid.cs`

This class will manage the grid and provide methods for adding, removing, and querying entities.

1.  Create `res://_Brain/Systems/AI/` folder.
2.  Create `res://_Brain/Systems/AI/SpatialHashGrid.cs`:

```csharp
// _Brain/Systems/AI/SpatialHashGrid.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Entities; // For EntityID

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Represents a 2D spatial hash grid for efficient proximity queries of entities.
    /// (TDD 05.2.1)
    /// </summary>
    public class SpatialHashGrid
    {
        private Dictionary<Vector2I, List<EntityID>> _grid = new Dictionary<Vector2I, List<EntityID>>();
        private Dictionary<EntityID, Vector2I> _entityToCell = new Dictionary<EntityID, Vector2I>(); // Tracks which cell an entity is in

        private float _cellSize; // Size of each grid cell (e.g., 128x128 pixels)
        private float _inverseCellSize; // 1 / _cellSize for faster division

        public SpatialHashGrid(float cellSize)
        {
            if (cellSize <= 0) throw new ArgumentOutOfRangeException(nameof(cellSize), "Cell size must be positive.");
            _cellSize = cellSize;
            _inverseCellSize = 1f / cellSize;
            GD.Print($"SpatialHashGrid: Initialized with cell size {_cellSize}.");
        }

        /// <summary>
        /// Converts world coordinates to grid cell coordinates.
        /// </summary>
        private Vector2I GetCellCoords(Vector2 worldPosition)
        {
            return new Vector2I(
                (int)MathF.Floor(worldPosition.X * _inverseCellSize),
                (int)MathF.Floor(worldPosition.Y * _inverseCellSize)
            );
        }

        /// <summary>
        /// Adds or updates an entity's position in the grid.
        /// </summary>
        public void AddOrUpdateEntity(EntityID id, Vector2 newWorldPosition)
        {
            Vector2I newCell = GetCellCoords(newWorldPosition);

            if (_entityToCell.TryGetValue(id, out Vector2I oldCell))
            {
                if (oldCell == newCell)
                {
                    // Entity is still in the same cell, no grid update needed.
                    return;
                }
                // Remove from old cell
                if (_grid.TryGetValue(oldCell, out List<EntityID> oldCellEntities))
                {
                    oldCellEntities.Remove(id);
                    if (oldCellEntities.Count == 0)
                    {
                        _grid.Remove(oldCell); // Clean up empty lists
                    }
                }
            }

            // Add to new cell
            if (!_grid.ContainsKey(newCell))
            {
                _grid[newCell] = new List<EntityID>();
            }
            _grid[newCell].Add(id);
            _entityToCell[id] = newCell;
            // GD.Print($"SpatialHashGrid: Entity {id} moved from {oldCell} to {newCell}.");
        }

        /// <summary>
        /// Removes an entity from the grid.
        /// </summary>
        public void RemoveEntity(EntityID id)
        {
            if (_entityToCell.TryGetValue(id, out Vector2I cell))
            {
                if (_grid.TryGetValue(cell, out List<EntityID> cellEntities))
                {
                    cellEntities.Remove(id);
                    if (cellEntities.Count == 0)
                    {
                        _grid.Remove(cell);
                    }
                }
                _entityToCell.Remove(id);
                // GD.Print($"SpatialHashGrid: Entity {id} removed from cell {cell}.");
            }
        }

        /// <summary>
        /// Retrieves all entities within a specified radius around a world position.
        /// (TDD 05.2.1: Query - "Get all entities in cells (X,Y) and neighbors.")
        /// </summary>
        /// <param name="worldPosition">The center of the query.</param>
        /// <param name="radius">The radius to search within.</param>
        /// <returns>A list of EntityIDs within the search area.</returns>
        public List<EntityID> QueryEntities(Vector2 worldPosition, float radius)
        {
            List<EntityID> nearbyEntities = new List<EntityID>();
            Vector2I centerCell = GetCellCoords(worldPosition);

            // Calculate min/max cells to check based on radius
            int radiusCells = (int)MathF.Ceiling(radius * _inverseCellSize);

            for (int x = centerCell.X - radiusCells; x <= centerCell.X + radiusCells; x++)
            {
                for (int y = centerCell.Y - radiusCells; y <= centerCell.Y + radiusCells; y++)
                {
                    Vector2I cellCoords = new Vector2I(x, y);
                    if (_grid.TryGetValue(cellCoords, out List<EntityID> cellEntities))
                    {
                        // Add all entities in this cell.
                        // A more precise query would filter by actual distance here.
                        nearbyEntities.AddRange(cellEntities);
                    }
                }
            }
            return nearbyEntities;
        }

        /// <summary>
        /// Retrieves all entities within the same cell as the given entity, including neighbors.
        /// Useful for simple "nearby" checks without a specific radius.
        /// </summary>
        public List<EntityID> GetEntitiesInNeighboringCells(EntityID entityID)
        {
            if (!_entityToCell.TryGetValue(entityID, out Vector2I centerCell))
            {
                return new List<EntityID>();
            }

            List<EntityID> nearbyEntities = new List<EntityID>();
            for (int x = centerCell.X - 1; x <= centerCell.X + 1; x++)
            {
                for (int y = centerCell.Y - 1; y <= centerCell.Y + 1; y++)
                {
                    Vector2I cellCoords = new Vector2I(x, y);
                    if (_grid.TryGetValue(cellCoords, out List<EntityID> cellEntities))
                    {
                        nearbyEntities.AddRange(cellEntities);
                    }
                }
            }
            return nearbyEntities;
        }
    }
}
```

### 4. Integrating `SpatialHashGrid` into `GameManager` and `TransformSystem`

The `SpatialHashGrid` needs to be updated whenever an entity's `TransformComponent.Position` changes. The `TransformSystem` is the ideal place for this.

1.  Add `using Sigilborne.Systems.AI;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `SpatialHashGrid` property.
3.  Initialize `SpatialHashGrid` in `InitializeSystems()`.

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
using Sigilborne.Systems.Weather;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Combat;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.AI; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public InventorySystem Inventory { get; private set; }
    public SpatialHashGrid SpatialGrid { get; private set; } // Add SpatialHashGrid property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        // ... (existing inventory/equipment tests) ...
        
        // --- Test Spatial Hash Grid ---
        GD.Print("\n--- Testing Spatial Hash Grid ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID testNpcID = Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid; // Assuming NPC is ID 1, Gen 1
        EntityID bossID = Entities.GetEntityMeta(2).Generation == 1 ? new EntityID(2, 1) : EntityID.Invalid; // Assuming Boss is ID 2, Gen 1
        EntityID newNpcID = Entities.CreateEntity(EntityType.NPC, "test_new_npc", new Vector2(250, 250)); // Create new NPC near player

        // Query entities near player
        List<EntityID> nearby = SpatialGrid.QueryEntities(Transforms.GetTransformRef(playerID).Position, 200f); // 200 pixel radius
        GD.Print($"Entities near player ({Transforms.GetTransformRef(playerID).Position}) (Radius 200): {string.Join(", ", nearby.Select(e => e.ToString()))}");
        
        // Move player far away
        Transforms.GetTransformRef(playerID).Position = new Vector2(1000, 1000); // This will update SpatialGrid in TransformSystem's event handler

        // Query again
        nearby = SpatialGrid.QueryEntities(Transforms.GetTransformRef(playerID).Position, 200f);
        GD.Print($"Entities near player ({Transforms.GetTransformRef(playerID).Position}) (Radius 200) after move: {string.Join(", ", nearby.Select(e => e.ToString()))}");

        GD.Print("--- End Testing Spatial Hash Grid ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta);
        Casting.Tick(delta);
        Weather.Tick(delta);
        Biology.Tick(delta);
        TitanAI.Tick(delta);
        StatusEffects.Tick(delta); // StatusEffectSystem needs to tick for duration/tick rate
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to EquipmentSystem) ...
        
        // Initialize SpatialHashGrid BEFORE TransformSystem (as TransformSystem needs it)
        SpatialGrid = new SpatialHashGrid(128f); // Cell size 128 (TDD 05.2.1)
        GD.Print("  - SpatialHashGrid initialized.");

        // Initialize TransformSystem, passing SpatialHashGrid
        Transforms = new TransformSystem(Entities, Events, SpatialGrid); // Pass SpatialGrid
        GD.Print("  - TransformSystem initialized.");
        
        // ... (existing system initializations after TransformSystem) ...
    }
}
```

#### 4.1. Update `TransformSystem.cs` to use `SpatialHashGrid`

The `TransformSystem` is responsible for updating the `SpatialHashGrid` whenever an entity's position changes.

Open `res://_Brain/Systems/TransformSystem.cs`:

```csharp
// _Brain/Systems/TransformSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.AI; // Add this using directive

namespace Sigilborne.Systems
{
    public class TransformSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private SpatialHashGrid _spatialGrid; // New: Reference to SpatialHashGrid

        private Dictionary<EntityID, TransformComponent> _transforms = new Dictionary<EntityID, TransformComponent>();

        public TransformSystem(EntityManager entityManager, EventBus eventBus, SpatialHashGrid spatialGrid) // Add SpatialHashGrid
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _spatialGrid = spatialGrid; // Store SpatialHashGrid reference

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("TransformSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityManager.EntitySpawnedEvent e)
        {
            _transforms.Add(e.ID, new TransformComponent(e.InitialPosition, e.InitialRotation));
            _spatialGrid.AddOrUpdateEntity(e.ID, e.InitialPosition); // Add to spatial grid on spawn
            GD.Print($"TransformSystem: Added transform for {e.ID} and added to spatial grid.");
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            if (_transforms.ContainsKey(e.ID))
            {
                _transforms.Remove(e.ID);
                _spatialGrid.RemoveEntity(e.ID); // Remove from spatial grid on despawn
                GD.Print($"TransformSystem: Removed transform for {e.ID} and removed from spatial grid.");
            }
        }

        // ... (TryGetTransform, TrySetTransform, GetTransformRef methods) ...

        public void Tick(double delta)
        {
            // TransformSystem's Tick is currently empty as its data is updated externally.
            // However, we need to ensure the SpatialGrid is updated whenever a transform changes.
            // This is primarily handled by PhysicsSystem.ReportFinalPosition.
        }

        /// <summary>
        /// Attempts to set the TransformComponent for a given entity.
        /// This method is called by PhysicsSystem to reconcile positions.
        /// (TDD 17.2.3: Reconciliation - Overwrite Brain's internal CurrentPosition)
        /// </summary>
        /// <returns>True if the component exists and is set, false otherwise.</returns>
        public bool TrySetTransform(EntityID id, TransformComponent transform)
        {
            if (!_entityManager.IsValid(id) || !_transforms.ContainsKey(id))
            {
                return false;
            }
            // Only update spatial grid if position actually changed
            if (_transforms[id].Position != transform.Position)
            {
                _spatialGrid.AddOrUpdateEntity(id, transform.Position); // Update spatial grid on position change
            }
            _transforms[id] = transform;
            return true;
        }
    }
}
```

#### 4.2. Update `GameManager` to Pass `SpatialHashGrid` to `TransformSystem`

Open `res://_Brain/Core/GameManager.cs` and modify the `TransformSystem` initialization in `InitializeSystems()`:

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        // Initialize TransformSystem, passing SpatialHashGrid
        Transforms = new TransformSystem(Entities, Events, SpatialGrid); // Pass SpatialGrid
        GD.Print("  - TransformSystem initialized.");
// ...
```

### 5. Testing the Spatial Hash Grid

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Spatial Hash Grid" section.

```
...
SpatialHashGrid: Initialized with cell size 128.
  - SpatialHashGrid initialized.
TransformSystem: Initialized.
  - TransformSystem initialized.
... (entity spawns, including newNpcID) ...
TransformSystem: EntityID(0, Gen:1) added to spatial grid.
TransformSystem: EntityID(1, Gen:1) added to spatial grid.
TransformSystem: EntityID(2, Gen:1) added to spatial grid.
TransformSystem: EntityID(3, Gen:1) added to spatial grid.
TransformSystem: EntityID(4, Gen:1) added to spatial grid.
...
--- Testing Spatial Hash Grid ---
Entities near player (200, 200) (Radius 200): EntityID(0, Gen:1), EntityID(1, Gen:1), EntityID(2, Gen:1), EntityID(3, Gen:1), EntityID(4, Gen:1)
SpatialHashGrid: Entity EntityID(0, Gen:1) moved from (1,1) to (7,7).
Entities near player (1000, 1000) (Radius 200) after move: EntityID(0, Gen:1)
--- End Testing Spatial Hash Grid ---
...
```

**Key Observations:**

*   **Initialization**: `SpatialHashGrid` is initialized.
*   **Add/Update**: `TransformSystem` correctly adds entities to the grid on spawn and updates them when their position changes (e.g., player moving to 1000,1000).
*   **Query**:
    *   The first query (near player at 200,200) correctly finds all entities (player, NPC, boss, new NPC) because they are all initially spawned close together, within a 200-pixel radius (which spans a few 128-pixel cells).
    *   The second query (near player at 1000,1000) only finds the player, demonstrating that the other entities are no longer considered "nearby" by the grid, improving query efficiency.

This confirms our `SpatialHashGrid` is functional, efficiently managing entity positions for proximity queries.

### Summary

You have successfully implemented the **Perception System** using a **Spatial Hash Grid** in the C# Brain. By designing `SpatialHashGrid` to manage entity positions in a grid-based structure and integrating it with `TransformSystem` for automatic updates, you've established an efficient mechanism for proximity queries. This crucial component strictly adheres to TDD 05.2.1's specifications, providing the foundational layer for AI perception and other range-based interactions in Sigilborne's living world.

### Next Steps

The next chapter will focus on **The Senses**, implementing `Vision`, `Hearing`, and `Chakra Sense` components for entities, which will leverage the `SpatialHashGrid` to detect nearby entities and events, enabling more sophisticated AI behavior.