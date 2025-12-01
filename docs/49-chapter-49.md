## Chapter 7.6: Spawning & Despawning - Hydration/Dehydration (Brain & Body)

Building on our `EcologyManager`'s concept of `VirtualAgent`s, this chapter focuses on the seamless transition of entities between "Virtual Agent" (data-only in C# Brain) and "Active Entity" (fully simulated with Godot Nodes in GDScript Body) states. We'll refine the hydration (spawning) and dehydration (despawning) process, ensuring entities appear and disappear gracefully as the player enters and leaves chunks, as specified in TDD 05.4.2 and TDD 05.4.3.

### 1. The Seamless World: Hydration & Dehydration

The illusion of a persistent world relies on entities appearing and disappearing smoothly.

*   **Dehydration**: When an active entity (NPC, animal) moves out of the player's loaded chunk radius, its visual Godot Node is removed, and its `CoreStats` and `TransformComponent` data are converted into a `VirtualAgent` in `EcologyManager`.
*   **Hydration**: When the player moves into a chunk containing a `VirtualAgent`, that `VirtualAgent`'s data is used to `CreateEntity()` in `EntityManager`, and a corresponding visual Godot Node is spawned by `EntityViewManager.gd`.

This process is critical for performance and maintaining consistency between the Brain and Body.

### 2. Refining `EcologyManager.cs` for Dehydration

The `HydrateAndDehydrateChunks()` method currently handles both. Let's make sure the dehydration part is robust.

Open `res://_Brain/Systems/Ecology/EcologyManager.cs`:

```csharp
// _Brain/Systems/Ecology/EcologyManager.cs
using Godot;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.AI;
using Sigilborne.Systems.Movement;
using Sigilborne.Systems.Weather;
using Sigilborne.Utils;
using System.Linq;

namespace Sigilborne.Systems.Ecology
{
    public class EcologyManager
    {
        // ... (existing fields) ...

        // Player's current loaded chunk (used for hydration/dehydration radius)
        private Vector2I _playerLoadedChunk = Vector2I.Zero;
        private const int LOAD_RADIUS_CHUNKS = 2; // TDD 08.2.1: Player loads chunks within a radius (e.g., 2 chunks)

        public EcologyManager(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, TransformSystem transformSystem, AISystem aiSystem, MovementSystem movementSystem, WeatherSystem weatherSystem, JobSystem jobSystem, SpatialHashGrid spatialGrid)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _transformSystem = transformSystem;
            _aiSystem = aiSystem;
            _movementSystem = movementSystem;
            _weatherSystem = weatherSystem;
            _jobSystem = jobSystem;
            _spatialGrid = spatialGrid;

            // Subscribe to entity lifecycle events
            // We'll manage OnEntitySpawned/Despawned internally for hydration/dehydration
            // This manager *is* the source of truth for spawning/despawning game entities.
            _eventBus.OnPlayerCoreStatsChanged += OnPlayerCoreStatsChanged;

            GD.Print("EcologyManager: Initialized.");
        }

        // ... (OnPlayerCoreStatsChanged, Tick, ProcessVirtualAgentTick methods) ...

        /// <summary>
        /// Manages the transition of entities between Active (full simulation) and Virtual (lightweight) states.
        /// (TDD 05.4.3: Transition (Hydration/Dehydration))
        /// </summary>
        private void HydrateAndDehydrateChunks()
        {
            // --- Dehydration: Convert active entities in unloaded chunks to virtual agents ---
            // Iterate all active entities (excluding player)
            List<EntityID> activeEntitiesToDehydrate = new List<EntityID>();
            EntityID playerID = _entityManager.GetPlayerEntityID();

            foreach (EntityID activeID in _entityManager.GetAllActiveEntityIDs())
            {
                if (activeID == playerID) continue; // Never dehydrate player

                if (_transformSystem.TryGetTransform(activeID, out TransformComponent transform))
                {
                    Vector2I entityChunk = GameManager.Instance.World.GetChunkCoords(transform.Position); // New: Use WorldSimulation helper

                    // If entity is outside the loaded radius, dehydrate it
                    if (!IsChunkInLoadRadius(entityChunk))
                    {
                        if (_biologicalSystem.TryGetCoreStats(activeID, out CoreStats stats) && _entityManager.TryGetEntityMeta(activeID, out EntityMeta meta))
                        {
                            // Create a new VirtualAgent from the active entity's current state
                            _virtualAgents.Add(new VirtualAgent(activeID, meta.Type, meta.DefinitionID, transform.Position, transform.RotationDegrees, stats, _aiSystem.GetSimplifiedAIState(activeID), _aiSystem.GetTargetID(activeID)));
                            activeEntitiesToDehydrate.Add(activeID);
                            GD.Print($"EcologyManager: Dehydrating {activeID} (Type: {meta.Type}).");
                        }
                    }
                }
            }
            // Destroy active entities that were dehydrated (must be done on main thread)
            foreach (EntityID id in activeEntitiesToDehydrate)
            {
                _entityManager.DestroyEntity(id); // This also triggers OnEntityDespawned event for Body
            }

            // --- Hydration: Convert virtual agents in loaded chunks to active entities ---
            List<VirtualAgent> remainingVirtualAgents = new List<VirtualAgent>();

            foreach (VirtualAgent agent in _virtualAgents)
            {
                Vector2I agentChunk = GameManager.Instance.World.GetChunkCoords(agent.Position); // New: Use WorldSimulation helper

                if (IsChunkInLoadRadius(agentChunk))
                {
                    // Hydrate this agent (spawn active entity)
                    // TDD 05.4.2: Spawning - Instantiate actual Godot Nodes.
                    // When CreateEntity is called, it triggers OnEntitySpawned, which
                    // EntityViewManager.gd listens to.
                    _entityManager.CreateEntity(agent.Type, agent.DefinitionID, agent.Position, agent.RotationDegrees);
                    // Update the newly created active entity's stats from the virtual agent (TDD 05.4.2)
                    ref CoreStats hydratedStats = ref _biologicalSystem.GetCoreStatsRef(agent.ID);
                    hydratedStats = agent.CurrentStats; // Copy all stats from virtual agent
                    // Set AI state as well
                    _aiSystem.SetSimplifiedAIState(agent.ID, agent.SimplifiedAIState);
                    _aiSystem.SetTargetID(agent.ID, agent.TargetID);

                    GD.Print($"EcologyManager: Hydrating {agent.ID} (Type: {agent.Type}).");
                }
                else
                {
                    remainingVirtualAgents.Add(agent); // Keep it virtual
                }
            }
            // Replace the ConcurrentBag with only the remaining virtual agents
            _virtualAgents = new ConcurrentBag<VirtualAgent>(remainingVirtualAgents);
        }

        /// <summary>
        /// Checks if a given chunk is within the player's loaded radius.
        /// </summary>
        private bool IsChunkInLoadRadius(Vector2I chunkCoords)
        {
            return Mathf.Abs(chunkCoords.X - _playerLoadedChunk.X) <= LOAD_RADIUS_CHUNKS &&
                   Mathf.Abs(chunkCoords.Y - _playerLoadedChunk.Y) <= LOAD_RADIUS_CHUNKS;
        }
        
        // --- Helper: Get a list of all current virtual agents ---
        public IReadOnlyList<VirtualAgent> GetAllVirtualAgents()
        {
            return _virtualAgents.ToList().AsReadOnly();
        }
    }
}
```

#### 2.1. Update `AISystem.cs` for Simplified AI State

`EcologyManager` needs to get and set a simplified AI state for virtual agents.

1.  Open `res://_Brain/Systems/AI/AISystem.cs` and add `_simplifiedAIStates` dictionary and `GetSimplifiedAIState` / `SetSimplifiedAIState` / `GetTargetID` / `SetTargetID` methods.

```csharp
// _Brain/Systems/AI/AISystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Combat;
using System.Linq;

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Manages AI decision-making using a Utility AI (scoring) system.
    /// NPCs evaluate actions based on needs and context and execute the highest-scoring one.
    /// (TDD 05.3)
    /// </summary>
    public class AISystem
    {
        // ... (existing fields) ...
        // Simplified AI state for when an entity is virtual
        private Dictionary<EntityID, string> _simplifiedAIStates = new Dictionary<EntityID, string>();
        private Dictionary<EntityID, EntityID> _aiTargets = new Dictionary<EntityID, EntityID>();

        // ... (constructor) ...

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.NPC || type == EntityType.Animal)
            {
                _aiComponents.Add(id, new AIComponent());
                _aiDecisionTimers.Add(id, (float)GameManager.Instance.Time.CurrentGameTime);
                _simplifiedAIStates.Add(id, "Wandering"); // Default simplified state
                _aiTargets.Add(id, EntityID.Invalid); // Default target
                GD.Print($"AISystem: Added AIComponent for {type} entity {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _aiComponents.Remove(e.ID);
            _aiDecisionTimers.Remove(e.ID);
            _simplifiedAIStates.Remove(e.ID);
            _aiTargets.Remove(e.ID);
            GD.Print($"AISystem: Removed AIComponent for {e.ID}.");
        }

        // ... (OnPlayerAlerted, RegisterAction, RegisterDefaultActions methods) ...

        public void Tick(double delta) { /* ... */ }
        private void MakeDecision(EntityID npcID) { /* ... */ }

        /// <summary>
        /// Gets the simplified AI state for a given entity.
        /// </summary>
        public string GetSimplifiedAIState(EntityID id)
        {
            return _simplifiedAIStates.TryGetValue(id, out string state) ? state : "None";
        }

        /// <summary>
        /// Sets the simplified AI state for a given entity. Used during hydration.
        /// </summary>
        public void SetSimplifiedAIState(EntityID id, string state)
        {
            if (_simplifiedAIStates.ContainsKey(id)) _simplifiedAIStates[id] = state;
        }

        /// <summary>
        /// Gets the target ID for a given entity.
        /// </summary>
        public EntityID GetTargetID(EntityID id)
        {
            return _aiTargets.TryGetValue(id, out EntityID target) ? target : EntityID.Invalid;
        }

        /// <summary>
        /// Sets the target ID for a given entity. Used during hydration.
        /// </summary>
        public void SetTargetID(EntityID id, EntityID target)
        {
            if (_aiTargets.ContainsKey(id)) _aiTargets[id] = target;
        }
    }
}
```

#### 2.2. Update `GameManager` to Pass `AISystem` to `EcologyManager`

Open `res://_Brain/Core/GameManager.cs` and modify the `EcologyManager` initialization in `InitializeSystems()`:

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        // Initialize EcologyManager AFTER all its dependencies, including AI
        Ecology = new EcologyManager(Entities, Events, BiologicalSystem, Transforms, AI, Movement, Weather, Jobs, SpatialGrid); // Pass AI
        GD.Print("  - EcologyManager initialized.");
// ...
```

### 3. Refining `WorldSimulation.cs` for Chunk Coordinates

`EcologyManager` needs to convert world positions to chunk coordinates. This logic should be in `WorldSimulation`.

Open `res://_Brain/Core/WorldSimulation.cs` and add `GetChunkCoords` method:

```csharp
// _Brain/Core/WorldSimulation.cs
using Godot;
using System;
using Sigilborne.Systems.Ecology;

namespace Sigilborne.Core
{
    public class WorldSimulation
    {
        public EcologyManager Ecology { get; private set; }
        public float ChunkSize { get; private set; } = 128f;

        public WorldSimulation(EcologyManager ecologyManager) { /* ... */ }
        public void Tick(double delta) { /* ... */ }

        /// <summary>
        /// Converts world coordinates to grid cell (chunk) coordinates.
        /// </summary>
        public Vector2I GetChunkCoords(Vector2 worldPosition)
        {
            return new Vector2I(
                (int)MathF.Floor(worldPosition.X / ChunkSize),
                (int)MathF.Floor(worldPosition.Y / ChunkSize)
            );
        }
    }
}
```

### 4. Testing Hydration/Dehydration

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.
    *   **Initial Spawns**: The player, test NPC, and boss are spawned as active entities.
    *   **Dehydration**: Move the player far away (e.g., use WASD to move the player sprite to the edge of the screen, or use the debug console `/spawn Player player_default 1000 1000`).
        *   You should see `EcologyManager: Dehydrating EntityID(...) (Type: NPC/Animal).` messages. The NPC and boss active entities should visually disappear from the screen, and their `EntityView`s will be `queue_free()`d.
    *   **Hydration**: Move the player back towards the original spawn location (e.g., `/spawn Player player_default 200 200`).
        *   You should see `EcologyManager: Hydrating EntityID(...) (Type: NPC/Animal).` messages. The NPC and boss entities should visually reappear.
    *   **Virtual Agent Tick**: While entities are dehydrated, `EcologyManager: Processing Virtual Agent Tick.` messages will continue every 10 seconds, showing their simplified simulation.

This confirms our `EcologyManager` is effectively handling the hydration and dehydration process, transitioning entities between active and virtual states based on player proximity.

### Summary

You have successfully refined the **Spawning & Despawning** mechanics, implementing a robust hydration (spawning) and dehydration (despawning) process for entities in Sigilborne. By enhancing `EcologyManager` to seamlessly convert entities between "Virtual Agent" (data-only in C# Brain) and "Active Entity" (fully simulated with Godot Nodes in GDScript Body) states, you've ensured efficient resource management and a persistent world illusion. This crucial system strictly adheres to TDD 05.4.2 and TDD 05.4.3's specifications, allowing entities to appear and disappear gracefully as the player navigates the world.

### Next Steps

This concludes **Module 7: The Living World - Ecology & AI**. We will now move on to **Module 8: Society, Politics & Economy**, starting with **Faction System - The Relationship Graph (C#)**, where we will design the core data structures for managing faction relationships and their dynamic interactions.