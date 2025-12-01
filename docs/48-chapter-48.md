## Chapter 7.5: Ecology Simulation - Virtual Agents (C#)

Sigilborne's world is a living, breathing ecosystem, even beyond the player's immediate vicinity. This chapter implements the **Ecology Simulation** system, designing how animals and NPCs exist and behave as "Virtual Agents" even when the player is far away in unloaded chunks. These virtual agents will be simulated at a very slow tick rate, maintaining the illusion of a persistent, evolving world, as specified in TDD 05.4.

### 1. The Challenge of a "Living World"

The GDD (B03) states: "The world exists **with or without the player**." This means:

*   **Persistence**: NPCs don't just disappear when off-screen; their lives continue.
*   **Performance**: Simulating every detail of every entity in the entire world all the time is impossible.
*   **Abstraction**: We need a lightweight representation for distant entities.
*   **Hydration/Dehydration**: Seamlessly transition entities between full simulation and lightweight virtual states.

### 2. Defining `VirtualAgent`

TDD 05.4.1 defines `VirtualAgent` as a lightweight data struct. We'll enhance it slightly to include more relevant data for ecology simulation.

1.  Create `res://_Brain/Systems/Ecology/` folder.
2.  Create `res://_Brain/Systems/Ecology/VirtualAgent.cs`:

```csharp
// _Brain/Systems/Ecology/VirtualAgent.cs
using System;
using Godot; // For Vector2
using Sigilborne.Entities; // For EntityID, EntityType
using Sigilborne.Systems.Biology; // For CoreStats

namespace Sigilborne.Systems.Ecology
{
    /// <summary>
    /// A lightweight data representation of an entity in an unloaded chunk.
    /// (TDD 05.4.1)
    /// </summary>
    public struct VirtualAgent
    {
        public EntityID ID;                 // The original EntityID
        public EntityType Type;             // Player, NPC, Animal
        public string DefinitionID;         // e.g., "goblin_grunt", "deer_male"
        public Vector2 Position;            // World position
        public float RotationDegrees;       // Rotation
        public CoreStats CurrentStats;      // Core stats (health, hunger, thirst, etc.)
        public int CurrentChunkX;           // Which chunk this agent is in
        public int CurrentChunkY;

        // --- AI State for Virtual Agents ---
        // Simplified AI state, e.g., "Wandering", "Hunting", "Eating"
        public string SimplifiedAIState;    
        public EntityID TargetID;           // Simplified target (e.g., player, food source)

        public VirtualAgent(EntityID id, EntityType type, string definitionID, Vector2 position, float rotationDegrees, CoreStats currentStats, string simplifiedAIState = "Wandering", EntityID targetID = default)
        {
            ID = id;
            Type = type;
            DefinitionID = definitionID;
            Position = position;
            RotationDegrees = rotationDegrees;
            CurrentStats = currentStats;
            CurrentChunkX = (int)MathF.Floor(position.X / GameManager.Instance.World.ChunkSize); // Assuming WorldSystem has ChunkSize
            CurrentChunkY = (int)MathF.Floor(position.Y / GameManager.Instance.World.ChunkSize);
            SimplifiedAIState = simplifiedAIState;
            TargetID = targetID;
        }

        public override string ToString()
        {
            return $"VA {ID} ({DefinitionID}) Pos: {Position} HP: {CurrentStats.Health:F0} HUN: {CurrentStats.Hunger:F0} State: {SimplifiedAIState}";
        }
    }
}
```

### 3. Implementing `EcologyManager.cs`

This system will manage the collection of `VirtualAgent`s, simulate them at a slow tick, and handle the hydration/dehydration (spawning/despawning) process.

1.  Create `res://_Brain/Systems/Ecology/EcologyManager.cs`:

```csharp
// _Brain/Systems/Ecology/EcologyManager.cs
using Godot;
using System;
using System.Collections.Concurrent; // For ConcurrentBag
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Entities.Components;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.AI; // For AI components, SpatialGrid
using Sigilborne.Systems.Movement; // For movement simulation
using Sigilborne.Systems.Weather; // For environmental impact
using Sigilborne.Utils; // For JobSystem
using System.Linq;

namespace Sigilborne.Systems.Ecology
{
    /// <summary>
    /// Manages the ecology simulation, including Virtual Agents in unloaded chunks.
    /// Handles hydration (spawning) and dehydration (despawning) of entities.
    /// (TDD 05.4)
    /// </summary>
    public class EcologyManager
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem;
        private TransformSystem _transformSystem;
        private AISystem _aiSystem;
        private MovementSystem _movementSystem;
        private WeatherSystem _weatherSystem;
        private JobSystem _jobSystem;
        private SpatialHashGrid _spatialGrid; // Needed for querying nearby virtual agents

        // TDD 05.4.1: Stores Virtual Agents in unloaded chunks.
        // Using ConcurrentBag for thread-safe adding/removing if jobs interact directly.
        private ConcurrentBag<VirtualAgent> _virtualAgents = new ConcurrentBag<VirtualAgent>();

        // TDD 13.3: Double Buffering for Ecology Simulation (Conceptual)
        // For simplicity in this chapter, we'll directly modify _virtualAgents.
        // A full implementation would use two ConcurrentBags and swap them.

        private const float VIRTUAL_AGENT_TICK_RATE = 10.0f; // Simulate virtual agents every 10 real seconds
        private float _virtualAgentTickTimer;

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

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;
            _eventBus.OnPlayerCoreStatsChanged += OnPlayerCoreStatsChanged; // To track player's chunk

            GD.Print("EcologyManager: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // For entities spawned by the GameManager (e.g., player, test NPC),
            // ensure they are *not* added to virtual agents initially if they are in loaded chunks.
            // This method is mainly for entities that *transition* from virtual to active.
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            // This is primarily for entities explicitly destroyed.
            // Entities dehydrating (transitioning from active to virtual) will be handled separately.
        }

        private void OnPlayerCoreStatsChanged(EntityID playerID, CoreStats newCoreStats)
        {
            // TDD 08.2.1: Track player's current chunk to manage load radius.
            Vector2I newPlayerChunk = new Vector2I(
                (int)MathF.Floor(newCoreStats.Position.X / GameManager.Instance.World.ChunkSize),
                (int)MathF.Floor(newCoreStats.Position.Y / GameManager.Instance.World.ChunkSize)
            );

            if (newPlayerChunk != _playerLoadedChunk)
            {
                _playerLoadedChunk = newPlayerChunk;
                // GD.Print($"EcologyManager: Player entered new chunk: {_playerLoadedChunk}. Triggering Hydration/Dehydration.");
                HydrateAndDehydrateChunks();
            }
        }

        /// <summary>
        /// Main update loop for the EcologyManager.
        /// Manages the Virtual Agent Tick.
        /// </summary>
        public void Tick(double delta)
        {
            _virtualAgentTickTimer += (float)delta;
            if (_virtualAgentTickTimer >= VIRTUAL_AGENT_TICK_RATE)
            {
                ProcessVirtualAgentTick();
                _virtualAgentTickTimer = 0;
            }
        }

        /// <summary>
        /// Processes a slow tick for all virtual agents.
        /// (TDD 05.4.1: Simulation - Update Virtual Agents at a very slow tick rate)
        /// </summary>
        private void ProcessVirtualAgentTick()
        {
            GD.Print($"EcologyManager: Processing Virtual Agent Tick. ({_virtualAgents.Count} virtual agents)");

            // TDD 13.4.1: Ecology Simulation - Task: Update hunger/movement for virtual animals.
            // For simplicity, we'll do this on the main thread for now.
            // A full implementation would use the JobSystem to process chunks of _virtualAgents.

            List<VirtualAgent> updatedAgents = new List<VirtualAgent>();
            foreach (VirtualAgent agent in _virtualAgents)
            {
                VirtualAgent updatedAgent = agent; // Create a mutable copy

                // --- Simulate simplified biological processes ---
                updatedAgent.CurrentStats.Hunger = Mathf.Max(0, updatedAgent.CurrentStats.Hunger - 5); // Hunger always decays
                updatedAgent.CurrentStats.Thirst = Mathf.Max(0, updatedAgent.CurrentStats.Thirst - 7); // Thirst always decays

                // --- Simulate simplified AI/movement ---
                // For now, simple random wander or move towards player if hungry/thirsty
                if (updatedAgent.CurrentStats.Hunger <= 20 || updatedAgent.CurrentStats.Thirst <= 20)
                {
                    updatedAgent.SimplifiedAIState = "SeekingSustenance";
                    // Move towards a conceptual food/water source (placeholder)
                    updatedAgent.Position += new Vector2((float)_rand.NextDouble() * 10 - 5, (float)_rand.NextDouble() * 10 - 5);
                }
                else
                {
                    updatedAgent.SimplifiedAIState = "Wandering";
                    updatedAgent.Position += new Vector2((float)_rand.NextDouble() * 2 - 1, (float)_rand.NextDouble() * 2 - 1); // Small random walk
                }

                updatedAgents.Add(updatedAgent);
            }
            // Replace old virtual agents with updated ones (conceptual double buffering)
            _virtualAgents = new ConcurrentBag<VirtualAgent>(updatedAgents);
        }

        /// <summary>
        /// Manages the transition of entities between Active (full simulation) and Virtual (lightweight) states.
        /// (TDD 05.4.3: Transition (Hydration/Dehydration))
        /// </summary>
        private void HydrateAndDehydrateChunks()
        {
            // --- Dehydration: Convert active entities in unloaded chunks to virtual agents ---
            // Iterate all active entities
            List<EntityID> activeEntitiesToDehydrate = new List<EntityID>();
            foreach (EntityID activeID in _entityManager.GetAllActiveEntityIDs()) // Assuming GetAllActiveEntityIDs exists
            {
                if (activeID == _entityManager.GetPlayerEntityID()) continue; // Never dehydrate player

                if (_transformSystem.TryGetTransform(activeID, out TransformComponent transform))
                {
                    Vector2I entityChunk = new Vector2I(
                        (int)MathF.Floor(transform.Position.X / GameManager.Instance.World.ChunkSize),
                        (int)MathF.Floor(transform.Position.Y / GameManager.Instance.World.ChunkSize)
                    );

                    // If entity is outside the loaded radius, dehydrate it
                    if (!IsChunkInLoadRadius(entityChunk))
                    {
                        if (_biologicalSystem.TryGetCoreStats(activeID, out CoreStats stats) && _entityManager.TryGetEntityMeta(activeID, out EntityMeta meta))
                        {
                            _virtualAgents.Add(new VirtualAgent(activeID, meta.Type, meta.DefinitionID, transform.Position, transform.RotationDegrees, stats));
                            activeEntitiesToDehydrate.Add(activeID);
                            // GD.Print($"EcologyManager: Dehydrating {activeID} (Type: {meta.Type}).");
                        }
                    }
                }
            }
            // Destroy active entities that were dehydrated (must be done on main thread)
            foreach (EntityID id in activeEntitiesToDehydrate)
            {
                _entityManager.DestroyEntity(id);
            }

            // --- Hydration: Convert virtual agents in loaded chunks to active entities ---
            List<VirtualAgent> virtualAgentsToHydrate = new List<VirtualAgent>();
            List<VirtualAgent> remainingVirtualAgents = new List<VirtualAgent>();

            foreach (VirtualAgent agent in _virtualAgents)
            {
                Vector2I agentChunk = new Vector2I(
                    (int)MathF.Floor(agent.Position.X / GameManager.Instance.World.ChunkSize),
                    (int)MathF.Floor(agent.Position.Y / GameManager.Instance.World.ChunkSize)
                );

                if (IsChunkInLoadRadius(agentChunk))
                {
                    // Hydrate this agent (spawn active entity)
                    // TDD 05.4.2: Spawning - Instantiate actual Godot Nodes.
                    // This creates a new active entity, but uses the old ID.
                    _entityManager.CreateEntity(agent.Type, agent.DefinitionID, agent.Position, agent.RotationDegrees);
                    // Update the new active entity's stats from the virtual agent (TDD 05.4.2)
                    ref CoreStats hydratedStats = ref _biologicalSystem.GetCoreStatsRef(agent.ID);
                    hydratedStats = agent.CurrentStats;
                    // GD.Print($"EcologyManager: Hydrating {agent.ID} (Type: {agent.Type}).");
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

        private Random _rand = new Random(); // For simple virtual agent movement
    }
}
```

#### 3.1. Update `EntityManager.cs` for `GetAllActiveEntityIDs` and `TryGetEntityMeta`

`EcologyManager` needs to iterate active entities and safely get `EntityMeta`.

1.  Open `res://_Brain/Entities/EntityManager.cs` and add these methods:

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
        // ... (existing fields) ...

        /// <summary>
        /// Retrieves all currently active EntityIDs.
        /// (TDD 05.4.3)
        /// </summary>
        public IEnumerable<EntityID> GetAllActiveEntityIDs()
        {
            for (int i = 0; i < MAX_ENTITIES; i++)
            {
                if (_entityMetas[i].IsActive)
                {
                    yield return new EntityID(i, _entityMetas[i].Generation);
                }
            }
        }

        /// <summary>
        /// Safely attempts to get the EntityMeta for a given EntityID.
        /// </summary>
        public bool TryGetEntityMeta(EntityID id, out EntityMeta meta)
        {
            if (IsValid(id))
            {
                meta = _entityMetas[id.Index];
                return true;
            }
            meta = default;
            return false;
        }

        // ... (other methods) ...
    }
}
```

#### 3.2. Update `CoreStats.cs` for Position

`CoreStats` needs a `Position` field for `VirtualAgent`s. This is a simplification; ideally, `VirtualAgent` would store `TransformComponent` data directly.

1.  Open `res://_Brain/Systems/Biology/CoreStats.cs` and add a `Position` field.

```csharp
// _Brain/Systems/Biology/CoreStats.cs
using System;
using Godot; // For Vector2

namespace Sigilborne.Systems.Biology
{
    public struct CoreStats
    {
        // ... (existing stats) ...
        public Vector2 Position; // New: For VirtualAgent tracking

        public CoreStats(float maxHealth, float maxStamina, float maxChakra, float baseDamage = 10f, float attackSpeed = 1.0f,
                         float armor = 0f, float magicResistance = 0f, float moveSpeed = 150f, float sprintMultiplier = 1.5f,
                         float castSpeed = 1.0f, float maxStability = 100f, float stabilityRegenRate = 5f,
                         float chakraRegenRate = 2f, float staminaRegenRate = 10f, float normalBodyTemp = 37.0f,
                         float critChance = 0.05f, float critMultiplier = 1.5f, float blockChance = 0f, float blockReduction = 0.5f,
                         Vector2 initialPosition = default) // Add initialPosition
        {
            // ... (existing initializations) ...
            Position = initialPosition; // Initialize position
        }

        public override string ToString() { /* ... */ }
    }
}
```

2.  Update `BiologicalSystem.OnEntitySpawned` to pass `initialPosition` to `CoreStats`.

```csharp
// _Brain/Systems/Biology/BiologicalSystem.cs (in OnEntitySpawned)
// ...
            if (type == EntityType.Player)
            {
                _entityCoreStats.Add(id, new CoreStats(
                    maxHealth: 100f, maxStamina: 75f, maxChakra: 50f,
                    baseDamage: 15f, attackSpeed: 1.0f, armor: 5f, magicResistance: 5f,
                    moveSpeed: 150f, sprintMultiplier: 1.5f, castSpeed: 1.0f,
                    maxStability: 100f, stabilityRegenRate: 5f, chakraRegenRate: 2f, staminaRegenRate: 10f,
                    normalBodyTemp: 37.0f, critChance: 0.1f, critMultiplier: 1.75f, blockChance: 0.1f, blockReduction: 0.5f,
                    initialPosition: initialPosition // Pass initialPosition
                ));
                GD.Print($"BiologicalSystem: Added CoreStats for Player {id}. Stats: {_entityCoreStats[id]}");
            }
            else if (type == EntityType.NPC || type == EntityType.Animal)
            {
                _entityCoreStats.Add(id, new CoreStats(
                    maxHealth: 50f, maxStamina: 30f, maxChakra: 20f,
                    baseDamage: 8f, attackSpeed: 1.0f, armor: 2f, magicResistance: 2f,
                    moveSpeed: 100f, sprintMultiplier: 1.2f, castSpeed: 1.0f,
                    maxStability: 50f, stabilityRegenRate: 3f, chakraRegenRate: 1f, staminaRegenRate: 5f,
                    normalBodyTemp: 37.0f, critChance: 0.05f, critMultiplier: 1.5f, blockChance: 0f, blockReduction: 0f,
                    initialPosition: initialPosition // Pass initialPosition
                ));
                GD.Print($"BiologicalSystem: Added CoreStats for {type} entity {id}. Stats: {_entityCoreStats[id]}");
            }
// ...
```

#### 3.3. Update `GameManager` to Pass `EcologyManager` to `WorldSimulation`

`WorldSimulation` will oversee the `EcologyManager`. Also, `WorldSimulation` needs a `ChunkSize` property.

1.  Add `using Sigilborne.Systems.Ecology;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `EcologyManager` property.
3.  Initialize `EcologyManager` in `InitializeSystems()` **after** all its dependencies.
4.  Modify `WorldSimulation` to have a `ChunkSize` property and constructor, and to take `EcologyManager`.

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        // Initialize EcologyManager AFTER all its dependencies
        Ecology = new EcologyManager(Entities, Events, BiologicalSystem, Transforms, AI, Movement, Weather, Jobs, SpatialGrid); // Pass dependencies
        GD.Print("  - EcologyManager initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        // Initialize WorldSimulation, passing EcologyManager
        World = new WorldSimulation(Ecology); // Pass EcologyManager
        GD.Print("  - WorldSimulation initialized.");
// ...
```

#### 3.4. Update `_Brain/Core/WorldSimulation.cs`

```csharp
// _Brain/Core/WorldSimulation.cs
using Godot;
using System;
using Sigilborne.Systems.Ecology; // Add this using directive

namespace Sigilborne.Core
{
    public class WorldSimulation
    {
        public EcologyManager Ecology { get; private set; } // New: Reference to EcologyManager
        public float ChunkSize { get; private set; } = 128f; // TDD 05.4.1: Chunk size (e.g., 128x128 pixels)

        public WorldSimulation(EcologyManager ecologyManager) // Add EcologyManager parameter
        {
            Ecology = ecologyManager; // Store EcologyManager reference
            GD.Print("WorldSimulation: Initialized.");
        }

        public void Tick(double delta)
        {
            // Update child systems here
            Ecology.Tick(delta); // EcologyManager needs to tick for virtual agents
            // GD.Print($"WorldSimulation: Tick at {GameManager.Instance.Time.CurrentGameTime:F2}"); // Example
        }
    }
}
```

### 4. Testing Ecology Simulation

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.
    *   You'll see `EcologyManager: Initialized.` and `WorldSimulation: Initialized.`
    *   Every 10 seconds (our `VIRTUAL_AGENT_TICK_RATE`), you should see `EcologyManager: Processing Virtual Agent Tick.` messages.
    *   The `VirtualAgent`s' hunger/thirst will decay, and their simplified AI states (`Wandering`, `SeekingSustenance`) will be printed.
    *   Move the player around. When the player crosses a chunk boundary (e.g., from `(0,0)` to `(1,0)` if `ChunkSize` is 128), `EcologyManager` should print `Hydrating/Dehydrating` messages.
        *   `EcologyManager: Dehydrating EntityID(1, Gen:1) (Type: NPC).`
        *   `EcologyManager: Hydrating EntityID(1, Gen:1) (Type: NPC).`

This confirms our `EcologyManager` is correctly simulating `VirtualAgent`s in the background and handling the hydration/dehydration process as the player moves between chunks.

### Summary

You have successfully implemented the **Ecology Simulation** system, designing `VirtualAgent` as a lightweight data representation for entities in unloaded chunks and creating `EcologyManager` to manage their lifecycle. By implementing a slow "Virtual Agent Tick" for background simulation and orchestrating the hydration/dehydration process based on the player's loaded chunk radius, you've ensured a persistent and evolving world, strictly adhering to TDD 05.4's specifications. This crucial step maintains the illusion of a living ecosystem without compromising performance.

### Next Steps

The next chapter will focus on **Spawning & Despawning**, detailing how `EcologyManager` and `EntityManager` seamlessly convert entities between "Virtual Agent" (data-only) and "Active Entity" (Godot node) states as the player enters and leaves chunks.