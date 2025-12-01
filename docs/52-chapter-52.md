## Chapter 8.3: Market Simulation - Supply & Demand (C#)

Our world now has dynamic factions and a ticking clock. The next layer of complexity is the **Economy Simulation**. This chapter focuses on implementing **Market Simulation** in the C# Brain, defining how prices for goods are local to each settlement and dynamically fluctuate based on supply, demand, and global modifiers like war or famine, as specified in TDD 06.3.1.

### 1. The Dynamic Nature of Sigilborne's Economy

The GDD (B23.1) states: "Economy is local, not global. No unified marketplace. Every settlement has different prices, resources, taboos, and needs." This requires a system that:

*   **Local Prices**: Prices vary by `Settlement`.
*   **Supply & Demand**: Prices respond to item availability and need.
*   **Global Modifiers**: World events (war, famine) can dramatically shift prices.
*   **Player Interaction**: Player actions (selling large quantities, destroying caravans) can influence local markets.

### 2. Defining `MarketItem`, `MarketRecord`, and `SettlementMarket`

We need a way to track items in a market, their current supply/demand, and the market itself for each settlement.

1.  Create `res://_Brain/Systems/Economy/` folder.
2.  Create `res://_Brain/Systems/Economy/MarketData.cs`:

```csharp
// _Brain/Systems/Economy/MarketData.cs
using System;
using System.Collections.Generic;
using Godot; // For GD.Print
using Sigilborne.Systems.Inventory; // For ItemDefinition

namespace Sigilborne.Systems.Economy
{
    /// <summary>
    /// Represents a single item in a market, with its supply, demand, and price.
    /// </summary>
    public struct MarketRecord
    {
        public string ItemID;
        public int CurrentSupply;   // Actual quantity available in this market
        public int CurrentDemand;   // How much the market "wants" this item
        public float BasePrice;     // Base price from ItemDefinition
        public float CurrentPrice;  // Calculated dynamic price

        public MarketRecord(string itemID, float basePrice, int initialSupply, int initialDemand)
        {
            ItemID = itemID;
            BasePrice = basePrice;
            CurrentSupply = initialSupply;
            CurrentDemand = initialDemand;
            CurrentPrice = basePrice; // Initial price
        }

        public override string ToString()
        {
            return $"Item: '{ItemID}' | Supply: {CurrentSupply}, Demand: {CurrentDemand}, Price: {CurrentPrice:F1}";
        }
    }

    /// <summary>
    /// Represents the local market of a single settlement.
    /// (TDD 06.3.1: Market Simulation - Prices are local to each Settlement)
    /// </summary>
    public class SettlementMarket
    {
        public string SettlementID { get; private set; }
        private Dictionary<string, MarketRecord> _marketRecords = new Dictionary<string, MarketRecord>();
        
        // Modifiers for this specific market (e.g., "war_modifier", "famine_modifier")
        private Dictionary<string, float> _localModifiers = new Dictionary<string, float>();

        public SettlementMarket(string settlementID)
        {
            SettlementID = settlementID;
            GD.Print($"SettlementMarket: Initialized for '{settlementID}'.");
        }

        /// <summary>
        /// Adds or updates an item's record in this market.
        /// </summary>
        public void AddOrUpdateItem(MarketRecord record)
        {
            _marketRecords[record.ItemID] = record;
        }

        /// <summary>
        /// Retrieves a market record for a given item.
        /// </summary>
        public MarketRecord GetMarketRecord(string itemID)
        {
            if (_marketRecords.TryGetValue(itemID, out MarketRecord record))
            {
                return record;
            }
            return new MarketRecord(itemID, 0f, 0, 0); // Return empty record
        }

        /// <summary>
        /// Applies a local modifier to this market.
        /// </summary>
        public void AddLocalModifier(string modifierID, float value)
        {
            _localModifiers[modifierID] = value;
        }

        /// <summary>
        /// Removes a local modifier from this market.
        /// </summary>
        public void RemoveLocalModifier(string modifierID)
        {
            _localModifiers.Remove(modifierID);
        }

        /// <summary>
        /// Retrieves all local modifiers.
        /// </summary>
        public IReadOnlyDictionary<string, float> GetLocalModifiers()
        {
            return _localModifiers.AsReadOnly();
        }

        /// <summary>
        /// Retrieves all market records for this settlement.
        /// </summary>
        public IReadOnlyList<MarketRecord> GetAllMarketRecords()
        {
            return _marketRecords.Values.ToList().AsReadOnly();
        }
    }
}
```

### 3. Implementing `EconomyManager.cs`

This system will:
*   Manage all `SettlementMarket` instances.
*   Register `ItemDefinition`s (using `EquipmentSystem`'s definitions).
*   Implement the `Market Simulation` algorithm for price calculation.
*   Apply global modifiers to markets.

1.  Create `res://_Brain/Systems/Economy/EconomyManager.cs`:

```csharp
// _Brain/Systems/Economy/EconomyManager.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Inventory; // For ItemDefinition
using Sigilborne.Systems.Factions; // For FactionType (to influence demand)
using System.Linq;

namespace Sigilborne.Systems.Economy
{
    /// <summary>
    /// Manages the economic simulation: local markets, supply & demand, and price fluctuations.
    /// (TDD 06.3.1)
    /// </summary>
    public class EconomyManager
    {
        private EventBus _eventBus;
        private EquipmentSystem _equipmentSystem; // To access ItemDefinitions
        private FactionSystem _factionSystem;     // To get faction types for demand modifiers

        // All active markets, mapped by SettlementID (string for now)
        private Dictionary<string, SettlementMarket> _activeMarkets = new Dictionary<string, SettlementMarket>();

        // Global economic modifiers (e.g., "war_active", "famine_event")
        private Dictionary<string, float> _globalModifiers = new Dictionary<string, float>();

        private const float MARKET_UPDATE_INTERVAL = 60.0f; // Update markets every 60 game minutes (1 real minute)
        private float _marketUpdateTimer;

        public EconomyManager(EventBus eventBus, EquipmentSystem equipmentSystem, FactionSystem factionSystem)
        {
            _eventBus = eventBus;
            _equipmentSystem = equipmentSystem;
            _factionSystem = factionSystem;

            // Subscribe to hourly updates for market simulation
            _eventBus.OnNewHour += OnNewHour;

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

        /// <summary>
        /// Adds a new settlement market to the economy.
        /// </summary>
        public void AddSettlementMarket(string settlementID, string controllingFactionID)
        {
            if (_activeMarkets.ContainsKey(settlementID)) return;

            SettlementMarket newMarket = new SettlementMarket(settlementID);
            _activeMarkets.Add(settlementID, newMarket);

            // Initialize market with all known items (from EquipmentSystem's definitions)
            foreach (var itemDef in _equipmentSystem.GetAllItemDefinitions())
            {
                // Initial supply and demand based on item type and controlling faction
                int initialSupply = 10;
                int initialDemand = 10;

                // Example: Warrior factions demand weapons, Temple factions demand healing items
                Faction controllingFaction = _factionSystem.GetFaction(controllingFactionID);
                if (controllingFaction != null)
                {
                    if (itemDef.Type == ItemType.Weapon && controllingFaction.Type == FactionType.WarriorDominion) initialDemand += 20;
                    if (itemDef.Type == ItemType.Consumable && itemDef.Name.Contains("healing") && controllingFaction.Type == FactionType.TempleAlliance) initialDemand += 15;
                    // Add more complex logic here
                }

                newMarket.AddOrUpdateItem(new MarketRecord(itemDef.ID, itemDef.BasePrice, initialSupply, initialDemand));
            }

            GD.Print($"EconomyManager: Added market for settlement '{settlementID}'.");
        }

        /// <summary>
        /// Updates prices, supply, and demand for all active markets.
        /// (TDD 06.3.1: Market Simulation - Algorithm)
        /// </summary>
        private void UpdateAllMarkets()
        {
            GD.Print($"EconomyManager: Updating all markets. (Game Time: {GameManager.Instance.Time.CurrentGameTime:F1})");

            foreach (var market in _activeMarkets.Values)
            {
                // Apply global modifiers (e.g., famine, war)
                float globalFamineMultiplier = _globalModifiers.TryGetValue("famine_active", out float famineVal) ? famineVal : 1.0f;
                float globalWarMultiplier = _globalModifiers.TryGetValue("war_active", out float warVal) ? warVal : 1.0f;

                List<MarketRecord> updatedRecords = new List<MarketRecord>();
                foreach (var record in market.GetAllMarketRecords())
                {
                    MarketRecord newRecord = record;

                    // --- Adjust Supply & Demand ---
                    // Simulate natural replenishment/consumption
                    newRecord.CurrentSupply += 1; // Basic replenishment
                    newRecord.CurrentDemand += 1; // Basic consumption
                    
                    // Apply global modifiers to supply/demand
                    if (newRecord.ItemID.Contains("food")) newRecord.CurrentSupply = (int)(newRecord.CurrentSupply / globalFamineMultiplier);
                    if (newRecord.ItemID.Contains("weapon")) newRecord.CurrentDemand = (int)(newRecord.CurrentDemand * globalWarMultiplier);

                    // Apply local modifiers (e.g., a siege might increase demand for weapons in one market)
                    // if (market.GetLocalModifiers().TryGetValue("siege_active", out float siegeVal) && newRecord.ItemID.Contains("weapon")) { newRecord.CurrentDemand += (int)(newRecord.CurrentDemand * siegeVal); }


                    // --- Calculate Price (TDD 06.3.1: Price = BasePrice * (Demand / Supply)) ---
                    // Ensure supply is never zero to avoid division by zero
                    float effectiveSupply = Mathf.Max(1, newRecord.CurrentSupply);
                    newRecord.CurrentPrice = newRecord.BasePrice * (newRecord.CurrentDemand / effectiveSupply);
                    
                    // Clamp prices to prevent extreme fluctuations
                    newRecord.CurrentPrice = Mathf.Clamp(newRecord.CurrentPrice, newRecord.BasePrice * 0.2f, newRecord.BasePrice * 5.0f);

                    updatedRecords.Add(newRecord);
                }
                // Update market records
                foreach (var record in updatedRecords)
                {
                    market.AddOrUpdateItem(record);
                }
                // GD.Print($"EconomyManager: Market '{market.SettlementID}' updated. Example price: {market.GetMarketRecord("iron_sword_t1").CurrentPrice:F1}");
            }
            _eventBus.Publish(new MarketsUpdatedEvent { Markets = _activeMarkets.Values.ToList() });
        }

        /// <summary>
        /// Retrieves a specific settlement market.
        /// </summary>
        public SettlementMarket GetMarket(string settlementID)
        {
            _activeMarkets.TryGetValue(settlementID, out SettlementMarket market);
            return market;
        }

        /// <summary>
        /// Applies a global economic modifier.
        /// </summary>
        public void AddGlobalModifier(string modifierID, float value)
        {
            _globalModifiers[modifierID] = value;
            GD.Print($"EconomyManager: Added global modifier '{modifierID}' with value {value}.");
        }

        /// <summary>
        /// Removes a global economic modifier.
        /// </summary>
        public void RemoveGlobalModifier(string modifierID)
        {
            _globalModifiers.Remove(modifierID);
            GD.Print($"EconomyManager: Removed global modifier '{modifierID}'.");
        }

        // --- Helper Events for Body Sync ---
        public struct MarketsUpdatedEvent { public List<SettlementMarket> Markets; }
    }
}
```

### 4. Integrating `EconomyManager` into `GameManager`

1.  Add `using Sigilborne.Systems.Economy;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add an `EconomyManager` property.
3.  Initialize `EconomyManager` in `InitializeSystems()` **after** `EquipmentSystem` and `FactionSystem`.

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
using Sigilborne.Systems.Economy; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public FactionSystem Factions { get; private set; }
    public EconomyManager Economy { get; private set; } // Add EconomyManager property

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
        Economy.AddSettlementMarket("village_north", "crimson_blades"); // Controlled by Crimson Blades
        Economy.AddSettlementMarket("town_south", "sunken_temple"); // Controlled by Sunken Temple

        // Initial check
        GD.Print($"Initial iron_sword_t1 price in village_north: {Economy.GetMarket("village_north").GetMarketRecord("iron_sword_t1").CurrentPrice:F1}");
        GD.Print($"Initial healing_potion_t1 price in town_south: {Economy.GetMarket("town_south").GetMarketRecord("healing_potion_t1").CurrentPrice:F1}");

        // Add a global modifier (e.g., "war_active")
        Economy.AddGlobalModifier("war_active", 2.0f); // Weapons demand x2
        Economy.AddGlobalModifier("famine_active", 3.0f); // Food supply /3

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
        Economy.Tick(delta); // Call EconomyManager's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to FactionSystem) ...
        
        FactionAI = new FactionAISystem(Events, Factions, Jobs, WorldSeed);
        GD.Print("  - FactionAISystem initialized.");

        // Initialize EconomyManager AFTER EquipmentSystem and FactionSystem
        Economy = new EconomyManager(Events, Equipment, Factions); // Pass dependencies
        GD.Print("  - EconomyManager initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation(Ecology);
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `EquipmentSystem.cs` to expose `GetAllItemDefinitions`

The `EconomyManager` needs to get all item definitions.

Open `res://_Brain/Systems/Inventory/EquipmentSystem.cs` and add this method:

```csharp
// _Brain/Systems/Inventory/EquipmentSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using System.Linq;
using System.Reflection;

namespace Sigilborne.Systems.Inventory
{
    // ... (EquipmentSlotType, EquippedItemsComponent, ItemDefinition, ItemType structs/classes) ...

    public class EquipmentSystem
    {
        // ... (existing fields and constructor) ...

        /// <summary>
        /// Retrieves all registered item definitions.
        /// </summary>
        public IReadOnlyList<ItemDefinition> GetAllItemDefinitions()
        {
            return _itemDefinitions.Values.ToList().AsReadOnly();
        }

        // ... (other methods) ...
    }
}
```

#### 4.2. Update `EventBus.cs` for `MarketsUpdatedEvent`

Open `res://_Brain/Core/EventBus.cs` and add `OnMarketsUpdated` delegate.

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
using Sigilborne.Systems.Economy; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Economy System Events (TDD 06.3.1)
        public event Action<List<SettlementMarket>> OnMarketsUpdated; // List of all markets

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is EconomyManager.MarketsUpdatedEvent marketsUpdatedEvent) // New condition
            {
                OnMarketsUpdated?.Invoke(marketsUpdatedEvent.Markets);
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

### 5. Testing the Market Simulation

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Economy System" section and subsequent `EconomyManager: Updating all markets.` messages.

```
...
EconomyManager: Initialized.
  - EconomyManager initialized.
PlayerStatSystem: Initialized.
...
--- Testing Economy System ---
EconomyManager: Added market for settlement 'village_north'.
EconomyManager: Added market for settlement 'town_south'.
Initial iron_sword_t1 price in village_north: 10.0
Initial healing_potion_t1 price in town_south: 10.0
EconomyManager: Added global modifier 'war_active' with value 2.
EconomyManager: Added global modifier 'famine_active' with value 3.
--- End Testing Economy System ---
...
EconomyManager: Updating all markets. (Game Time: 0.0)
EconomyManager: Updating all markets. (Game Time: 60.0)
EconomyManager: Updating all markets. (Game Time: 120.0)
...
```

After a few `EconomyManager: Updating all markets.` ticks (which happen every 1 real minute):

*   **`village_north` (WarriorDominion)**: Prices for `iron_sword_t1` should increase (due to `war_active` increasing demand, and `WarriorDominion` having high base demand for weapons). Food prices will also increase due to `famine_active` reducing supply.
*   **`town_south` (TempleAlliance)**: Prices for `healing_potion_t1` should increase (due to `TempleAlliance` having high base demand). Food prices will also increase.

This demonstrates that:
*   `EconomyManager` correctly initializes markets for settlements.
*   Initial supply/demand is influenced by `FactionType`.
*   Global modifiers (`war_active`, `famine_active`) are applied.
*   Market prices dynamically fluctuate based on supply/demand and modifiers, updating on a regular interval.

### Summary

You have successfully implemented **Market Simulation** in the C# Brain, defining `MarketRecord` and `SettlementMarket` to track local market conditions. By designing `EconomyManager` to manage active markets, register item definitions, and dynamically calculate prices based on supply, demand, and global modifiers (like war or famine), you've established a robust mechanism for a fluctuating economy. This crucial system strictly adheres to TDD 06.3.1's specifications, providing a dynamic and localized economic backbone for Sigilborne.

### Next Steps

The next chapter will focus on **Trade Routes**, detailing how NPC caravans physically move between settlements, carrying goods, and how their journeys can be interrupted by player actions or world events, influencing supply chains and local market prices.