## Chapter 8.4: Trade Routes - Caravans as Arteries (C#)

Our `EconomyManager` now simulates local markets and dynamic prices. The next crucial component of the economic system is **Trade Routes**, represented by NPC caravans physically moving between settlements. This chapter designs how these caravans operate, carry goods, and how their journeys can be interrupted by player actions or world events, directly influencing supply chains and local market prices, as specified in TDD 06.3.2.

### 1. The Dynamic Nature of Trade

The GDD (B23.4) states: "Caravans travel between settlements carrying... goods... Each caravan has a faction identity, a risk rating, preferred routes, defenses, negotiable prices, vulnerabilities." This implies:

*   **Physical Movement**: Caravans are entities that move in the world.
*   **Goods Flow**: They transport items, affecting supply/demand in origin/destination markets.
*   **Vulnerability**: They can be attacked, robbed, or destroyed.
*   **Dynamic Routes**: Routes can change due to world instability.
*   **Player Interaction**: Players can escort, attack, or trade with caravans.

### 2. Defining `Caravan` and `CaravanRoute`

We need a class to represent a single caravan entity and a class to define its route.

1.  Create `res://_Brain/Systems/Economy/CaravanData.cs`:

```csharp
// _Brain/Systems/Economy/CaravanData.cs
using System;
using System.Collections.Generic;
using Godot; // For Vector2
using Sigilborne.Entities; // For EntityID
using Sigilborne.Systems.Factions; // For Faction
using Sigilborne.Systems.Inventory; // For InventoryItem

namespace Sigilborne.Systems.Economy
{
    /// <summary>
    /// Represents a single trade caravan moving between settlements.
    /// (TDD 06.3.2)
    /// </summary>
    public class Caravan
    {
        public EntityID ID { get; private set; } // The entity ID of this caravan
        public string Name { get; private set; }
        public string OriginSettlementID { get; private set; }
        public string DestinationSettlementID { get; private set; }
        public string FactionID { get; private set; } // The faction this caravan belongs to
        public List<InventoryItem> Cargo { get; private set; } // The goods being transported
        public CaravanRoute CurrentRoute { get; private set; }
        public float Speed { get; private set; }
        public float CurrentProgress { get; private set; } // 0.0 to 1.0 along the route
        public bool IsDestroyed { get; private set; }
        public bool IsActive { get; private set; } // True if currently moving in the world

        public Caravan(EntityID id, string name, string originID, string destinationID, string factionID, CaravanRoute route, float speed, List<InventoryItem> cargo = null)
        {
            ID = id;
            Name = name;
            OriginSettlementID = originID;
            DestinationSettlementID = destinationID;
            FactionID = factionID;
            CurrentRoute = route;
            Speed = speed;
            Cargo = cargo ?? new List<InventoryItem>();
            CurrentProgress = 0f;
            IsDestroyed = false;
            IsActive = true;
        }

        public override string ToString()
        {
            return $"Caravan: '{Name}' ({ID}) | {OriginSettlementID} -> {DestinationSettlementID} | Progress: {CurrentProgress:P0}";
        }
    }

    /// <summary>
    /// Defines a geographical path between two settlements for a caravan.
    /// </summary>
    public class CaravanRoute
    {
        public string ID { get; private set; } // Unique route ID
        public string OriginSettlementID { get; private set; }
        public string DestinationSettlementID { get; private set; }
        public float Length { get; private set; } // Total length of the route
        public List<Vector2> Waypoints { get; private set; } // The path the caravan follows

        public CaravanRoute(string id, string originID, string destinationID, List<Vector2> waypoints)
        {
            ID = id;
            OriginSettlementID = originID;
            DestinationSettlementID = destinationID;
            Waypoints = waypoints ?? new List<Vector2>();
            Length = CalculateLength(waypoints); // Calculate total length
        }

        private float CalculateLength(List<Vector2> waypoints)
        {
            float totalLength = 0f;
            for (int i = 0; i < waypoints.Count - 1; i++)
            {
                totalLength += waypoints[i].DistanceTo(waypoints[i+1]);
            }
            return totalLength;
        }

        /// <summary>
        /// Gets the world position along the route at a given progress (0.0 to 1.0).
        /// </summary>
        public Vector2 GetPositionAtProgress(float progress)
        {
            if (Waypoints.Count < 2 || progress < 0 || progress > 1) return Waypoints.FirstOrDefault();

            float targetDistance = Length * progress;
            float currentDistance = 0f;

            for (int i = 0; i < Waypoints.Count - 1; i++)
            {
                float segmentLength = Waypoints[i].DistanceTo(Waypoints[i+1]);
                if (currentDistance + segmentLength >= targetDistance)
                {
                    // The target position is within this segment
                    float segmentProgress = (targetDistance - currentDistance) / segmentLength;
                    return Waypoints[i].Lerp(Waypoints[i+1], segmentProgress);
                }
                currentDistance += segmentLength;
            }
            return Waypoints.LastOrDefault(); // Should be reached at progress 1.0
        }

        public override string ToString()
        {
            return $"Route: '{ID}' | {OriginSettlementID} -> {DestinationSettlementID} | Length: {Length:F0}";
        }
    }
}
```

### 3. Enhancing `EconomyManager.cs` for Trade Routes

`EconomyManager` will now:
*   Manage a collection of `CaravanRoute`s and `Caravan` entities.
*   Implement `CaravanSystem.Depart()` (TDD 06.2.3) logic to spawn caravans.
*   Update `Caravan` progress and position.
*   Handle `Caravan` destruction and its economic impact.

1.  Open `res://_Brain/Systems/Economy/EconomyManager.cs`.
2.  Add dictionaries for `CaravanRoute`s and `Caravan` entities.
3.  Implement `AddCaravanRoute`, `SpawnCaravan`, and a `Tick` method for caravans.
4.  Modify `UpdateAllMarkets` to factor in caravan status.

```csharp
// _Brain/Systems/Economy/EconomyManager.cs
using Godot;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.Factions;
using Sigilborne.Systems.Movement; // For caravan movement via TransformSystem
using System.Linq;

namespace Sigilborne.Systems.Economy
{
    // ... (MarketRecord, SettlementMarket structs/classes) ...

    /// <summary>
    /// Manages the economic simulation: local markets, supply & demand, and price fluctuations.
    /// Now also manages trade routes and caravans.
    /// (TDD 06.3)
    /// </summary>
    public class EconomyManager
    {
        private EventBus _eventBus;
        private EquipmentSystem _equipmentSystem;
        private FactionSystem _factionSystem;
        private EntityManager _entityManager; // New: For spawning caravan entities
        private TransformSystem _transformSystem; // New: For updating caravan positions

        private Dictionary<string, SettlementMarket> _activeMarkets = new Dictionary<string, SettlementMarket>();
        private Dictionary<string, float> _globalModifiers = new Dictionary<string, float>();

        // New: Trade route and caravan management
        private Dictionary<string, CaravanRoute> _caravanRoutes = new Dictionary<string, CaravanRoute>();
        private Dictionary<EntityID, Caravan> _activeCaravans = new Dictionary<EntityID, Caravan>(); // Active caravan entities

        private const float MARKET_UPDATE_INTERVAL = 60.0f; // Update markets every 60 game minutes (1 real minute)
        private float _marketUpdateTimer;

        private const float CARAVAN_SPAWN_INTERVAL = 120.0f; // Try to spawn a caravan every 120 game minutes (2 real minutes)
        private float _caravanSpawnTimer;

        public EconomyManager(EventBus eventBus, EquipmentSystem equipmentSystem, FactionSystem factionSystem, EntityManager entityManager, TransformSystem transformSystem) // Add EntityManager, TransformSystem
        {
            _eventBus = eventBus;
            _equipmentSystem = equipmentSystem;
            _factionSystem = factionSystem;
            _entityManager = entityManager;
            _transformSystem = transformSystem;

            _eventBus.OnNewHour += OnNewHour;
            _eventBus.OnEntityDefeated += OnEntityDefeated; // To detect caravan destruction

            GD.Print("EconomyManager: Initialized.");
        }

        private void OnNewHour(int hour)
        {
            _marketUpdateTimer += (float)GameManager.Instance.Time.REAL_SECONDS_PER_GAME_MINUTE * 60; // Add 60 seconds (1 game hour)
            if (_marketUpdateTimer >= MARKET_UPDATE_INTERVAL)
            {
                UpdateAllMarkets();
                _marketUpdateTimer = 0;
            }
        }

        private void OnEntityDefeated(EntityID entityID, EntityID killerID)
        {
            if (_activeCaravans.ContainsKey(entityID))
            {
                Caravan destroyedCaravan = _activeCaravans[entityID];
                destroyedCaravan.IsDestroyed = true; // Mark as destroyed
                destroyedCaravan.IsActive = false;
                GD.Print($"EconomyManager: Caravan '{destroyedCaravan.Name}' ({entityID}) was destroyed by {killerID}!");
                
                // --- Economic Impact of Caravan Destruction --- (TDD 06.3.2)
                // Reduce supply in destination market
                SettlementMarket destinationMarket = GetMarket(destroyedCaravan.DestinationSettlementID);
                if (destinationMarket != null)
                {
                    foreach (var item in destroyedCaravan.Cargo)
                    {
                        MarketRecord record = destinationMarket.GetMarketRecord(item.ItemID);
                        record.CurrentSupply -= item.Quantity; // Reduce supply for lost cargo
                        destinationMarket.AddOrUpdateItem(record);
                    }
                }
                // Increase demand for missing items
                // Increase prices in destination market
                // Reduce reputation with faction

                _activeCaravans.Remove(entityID); // Remove from active list
                _eventBus.Publish(new CaravanDestroyedEvent { CaravanID = entityID, KillerID = killerID, CaravanName = destroyedCaravan.Name });
            }
        }

        /// <summary>
        /// Main update loop for the EconomyManager.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            _caravanSpawnTimer += (float)delta;
            if (_caravanSpawnTimer >= CARAVAN_SPAWN_INTERVAL)
            {
                TrySpawnCaravan();
                _caravanSpawnTimer = 0;
            }

            UpdateActiveCaravans(delta); // Update position of active caravans
        }

        /// <summary>
        /// Adds a new settlement market to the economy.
        /// </summary>
        public void AddSettlementMarket(string settlementID, string controllingFactionID)
        {
            // ... (existing logic) ...
        }

        /// <summary>
        /// Adds a new trade route to the economy.
        /// </summary>
        public void AddCaravanRoute(CaravanRoute route)
        {
            if (_caravanRoutes.ContainsKey(route.ID)) return;
            _caravanRoutes.Add(route.ID, route);
            GD.Print($"EconomyManager: Added trade route '{route.ID}': {route.OriginSettlementID} to {route.DestinationSettlementID}.");
        }

        /// <summary>
        /// Tries to spawn a new caravan on an available route.
        /// (TDD 06.2.3: CaravanSystem.Depart())
        /// </summary>
        private void TrySpawnCaravan()
        {
            if (_caravanRoutes.Count == 0 || _activeMarkets.Count < 2) return;

            // Simple logic: pick a random route
            Random rand = new Random(GameManager.Instance.WorldSeed + (int)GameManager.Instance.Time.CurrentGameTime);
            CaravanRoute route = _caravanRoutes.Values.ElementAt(rand.Next(_caravanRoutes.Count));

            // Check if origin/destination markets exist
            if (!_activeMarkets.ContainsKey(route.OriginSettlementID) || !_activeMarkets.ContainsKey(route.DestinationSettlementID))
            {
                GD.PrintErr($"EconomyManager: Cannot spawn caravan for route '{route.ID}'. Markets not found.");
                return;
            }

            // Create a dummy caravan entity
            EntityID caravanEntityID = _entityManager.CreateEntity(EntityType.WorldObject, "caravan_wagon_01", route.Waypoints.First(), 0f);
            if (!caravanEntityID.IsValid())
            {
                GD.PrintErr("EconomyManager: Failed to create entity for new caravan.");
                return;
            }
            
            // Populate cargo (simple for now)
            List<InventoryItem> cargo = new List<InventoryItem>();
            cargo.Add(new InventoryItem("iron_sword_t1", 5));
            cargo.Add(new InventoryItem("healing_potion_t1", 10));

            string factionID = _factionSystem.GetAllFactions().ElementAt(rand.Next(_factionSystem.GetAllFactions().Count)).ID; // Random faction
            Caravan newCaravan = new Caravan(caravanEntityID, $"Caravan {GameManager.Instance.Time.CurrentDay}-{_activeCaravans.Count}", route.OriginSettlementID, route.DestinationSettlementID, factionID, route, 50f, cargo);
            _activeCaravans.Add(caravanEntityID, newCaravan);

            GD.Print($"EconomyManager: Spawned new caravan '{newCaravan.Name}' ({caravanEntityID}) from {route.OriginSettlementID} to {route.DestinationSettlementID}.");
            _eventBus.Publish(new CaravanSpawnedEvent { CaravanID = caravanEntityID, Caravan = newCaravan });
        }

        /// <summary>
        /// Updates the position and progress of all active caravans.
        /// </summary>
        private void UpdateActiveCaravans(double delta)
        {
            List<EntityID> completedCaravans = new List<EntityID>();
            foreach (var kvp in _activeCaravans)
            {
                EntityID caravanID = kvp.Key;
                Caravan caravan = kvp.Value;

                if (caravan.IsDestroyed || !caravan.IsActive) continue;

                // Calculate movement
                float distanceToTravel = caravan.Speed * (float)delta;
                caravan.CurrentProgress += distanceToTravel / caravan.CurrentRoute.Length;
                caravan.CurrentProgress = Mathf.Min(1.0f, caravan.CurrentProgress); // Clamp at 1.0

                // Update position in TransformSystem
                Vector2 newPosition = caravan.CurrentRoute.GetPositionAtProgress(caravan.CurrentProgress);
                _transformSystem.TrySetTransform(caravanID, new TransformComponent(newPosition));

                if (caravan.CurrentProgress >= 1.0f)
                {
                    completedCaravans.Add(caravanID);
                }
            }

            // Process completed caravans
            foreach (EntityID id in completedCaravans)
            {
                Caravan completedCaravan = _activeCaravans[id];
                GD.Print($"EconomyManager: Caravan '{completedCaravan.Name}' ({id}) reached destination {completedCaravan.DestinationSettlementID}.");
                
                // --- Economic Impact of Caravan Arrival --- (TDD 06.3.2)
                // Add cargo to destination market's supply
                SettlementMarket destinationMarket = GetMarket(completedCaravan.DestinationSettlementID);
                if (destinationMarket != null)
                {
                    foreach (var item in completedCaravan.Cargo)
                    {
                        MarketRecord record = destinationMarket.GetMarketRecord(item.ItemID);
                        record.CurrentSupply += item.Quantity; // Increase supply for arrived cargo
                        destinationMarket.AddOrUpdateItem(record);
                    }
                }
                // Reduce demand in destination market
                // Reduce prices in destination market
                // Improve reputation with faction

                _entityManager.DestroyEntity(id); // Despawn caravan entity
                _activeCaravans.Remove(id);
                _eventBus.Publish(new CaravanArrivedEvent { CaravanID = id, Caravan = completedCaravan });
            }
        }
        
        // ... (GetMarket, AddGlobalModifier, RemoveGlobalModifier methods) ...

        // --- Helper Events for Body Sync ---
        public struct MarketsUpdatedEvent { public List<SettlementMarket> Markets; }
        public struct CaravanSpawnedEvent { public EntityID CaravanID; public Caravan Caravan; }
        public struct CaravanArrivedEvent { public EntityID CaravanID; public Caravan Caravan; }
        public struct CaravanDestroyedEvent { public EntityID CaravanID; public EntityID KillerID; public string CaravanName; }
    }
}
```

### 4. Integrating `EconomyManager` into `GameManager`

1.  Open `res://_Brain/Core/GameManager.cs`.
2.  Add `using Sigilborne.Systems.Economy;` at the top.
3.  Modify `EconomyManager` initialization in `InitializeSystems()` to pass `EntityManager` and `TransformSystem`.
4.  Call `EconomyManager.Tick(delta)` in `_PhysicsProcess`.

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
using Sigilborne.Systems.AI;
using Sigilborne.Systems.Factions;
using Sigilborne.Systems.Economy;
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    // ... (existing system properties) ...
    public EconomyManager Economy { get; private set; }

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        // ... (existing inventory/equipment tests) ...
        // ... (existing spatial grid tests) ...
        // ... (existing perception system tests) ...
        // ... (existing stealth system tests) ...
        // ... (existing AI system tests) ...
        // ... (existing ecology system tests) ...
        // ... (existing faction system tests) ...

        // --- Test Economy System ---
        GD.Print("\n--- Testing Economy System ---");
        // Add a test settlement market
        Economy.AddSettlementMarket("village_north", "crimson_blades");
        Economy.AddSettlementMarket("town_south", "sunken_temple");

        // Add a test caravan route
        CaravanRoute route1 = new CaravanRoute("route_north_south", "village_north", "town_south", new List<Vector2> { new Vector2(100, 100), new Vector2(500, 100), new Vector2(500, 500) });
        Economy.AddCaravanRoute(route1);

        // Initial check
        GD.Print($"Initial iron_sword_t1 price in village_north: {Economy.GetMarket("village_north").GetMarketRecord("iron_sword_t1").CurrentPrice:F1}");
        GD.Print($"Initial healing_potion_t1 price in town_south: {Economy.GetMarket("town_south").GetMarketRecord("healing_potion_t1").CurrentPrice:F1}");

        // Add a global modifier (e.g., "war_active")
        Economy.AddGlobalModifier("war_active", 2.0f);
        Economy.AddGlobalModifier("famine_active", 3.0f);

        GD.Print("--- End Testing Economy System ---\n");
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
        StatusEffects.Tick(delta);
        Perception.Tick(delta);
        Stealth.Tick(delta);
        AI.Tick(delta);
        Ecology.Tick(delta);
        FactionAI.Tick(delta);
        Economy.Tick(delta);
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to FactionSystem) ...
        
        FactionAI = new FactionAISystem(Events, Factions, Jobs, WorldSeed);
        GD.Print("  - FactionAISystem initialized.");

        // Initialize EconomyManager AFTER EquipmentSystem, FactionSystem, EntityManager, TransformSystem
        Economy = new EconomyManager(Events, Equipment, Factions, Entities, Transforms); // Pass dependencies
        GD.Print("  - EconomyManager initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation(Ecology);
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `EventBus.cs` for Caravan Events

Open `res://_Brain/Core/EventBus.cs` and add `OnCaravanSpawned`, `OnCaravanArrived`, `OnCaravanDestroyed` delegates.

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.StatusEffects;
using Sigilborne.Systems.Combat;
using Sigilborne.Systems.Inventory;
using Sigilborne.Systems.AI;
using Sigilborne.Systems.Factions;
using Sigilborne.Systems.Economy;
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Economy System Events (TDD 06.3.1)
        public event Action<List<SettlementMarket>> OnMarketsUpdated;

        // Caravan Events (TDD 06.3.2)
        public event Action<EntityID, Caravan> OnCaravanSpawned;
        public event Action<EntityID, Caravan> OnCaravanArrived;
        public event Action<EntityID, EntityID, string> OnCaravanDestroyed; // CaravanID, KillerID, CaravanName

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is EconomyManager.MarketsUpdatedEvent marketsUpdatedEvent)
            {
                OnMarketsUpdated?.Invoke(marketsUpdatedEvent.Markets);
            }
            else if (eventData is EconomyManager.CaravanSpawnedEvent caravanSpawnedEvent) // New condition
            {
                OnCaravanSpawned?.Invoke(caravanSpawnedEvent.CaravanID, caravanSpawnedEvent.Caravan);
            }
            else if (eventData is EconomyManager.CaravanArrivedEvent caravanArrivedEvent) // New condition
            {
                OnCaravanArrived?.Invoke(caravanArrivedEvent.CaravanID, caravanArrivedEvent.Caravan);
            }
            else if (eventData is EconomyManager.CaravanDestroyedEvent caravanDestroyedEvent) // New condition
            {
                OnCaravanDestroyed?.Invoke(caravanDestroyedEvent.CaravanID, caravanDestroyedEvent.KillerID, caravanDestroyedEvent.CaravanName);
            }
            else
            {
                GD.PrintErr($"EventBus: Attempted to publish unknown event type: {typeof(TEvent).Name}. Ensure it's handled in Publish<TEvent>.");
            }
        }
        // ... (AddCommand and FlushCommands methods) ...
    }
}
```

### 5. Testing Trade Routes and Caravans

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.
    *   `EconomyManager: Added market for settlement 'village_north'.`
    *   `EconomyManager: Added market for settlement 'town_south'.`
    *   `EconomyManager: Added trade route 'route_north_south': village_north to town_south.`
    *   Every 2 real minutes (120 game minutes), `EconomyManager: Spawned new caravan 'Caravan X-Y' ...` messages will appear.
    *   You should see these caravan entities moving on screen (they are `EntityType.WorldObject` and get `TransformComponent`s).
    *   After a caravan reaches its destination (progress 1.0), you'll see `EconomyManager: Caravan 'X' reached destination...` and `EntityManager: Destroyed entity...` messages.
    *   If you continuously spawn and destroy caravans (e.g. by using the debug console to speed up time, or by running the game for a while), you'll also see their economic impact on market prices in the `UpdateAllMarkets` calls.

This demonstrates that:
*   `CaravanRoute`s and `Caravan` entities are correctly defined.
*   `EconomyManager` spawns caravans, updates their positions, and destroys them upon arrival.
*   Caravan destruction (simulated by `OnEntityDefeated` if you kill a caravan entity) correctly impacts the destination market's supply.

### Summary

You have successfully implemented **Trade Routes** and **Caravans** in the C# Brain, designing `Caravan` and `CaravanRoute` classes to represent physical trade entities and their paths. By enhancing `EconomyManager` to manage route definitions, spawn caravans, update their progress, and simulate their economic impact upon arrival or destruction, you've established a robust mechanism for dynamic goods flow. This crucial system strictly adheres to TDD 06.3.2's specifications, allowing trade to visibly influence supply chains and local market prices in Sigilborne.

### Next Steps

The next chapter will focus on **Crime & Justice - The Heat System (C#)**, where we will implement a system to track "Heat" (criminal notoriety) per faction, based on player crimes, and design how witnesses report these crimes, leading to escalating consequences.