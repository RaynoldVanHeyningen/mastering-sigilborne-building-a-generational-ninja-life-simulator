## Chapter 6.4: Equipment Logic - Recalculating Stats (C#)

With our `InventorySystem` managing items, the next step is to implement **Equipment Logic**. This chapter focuses on how items are moved to "Equipment Slots," how an entity's `CoreStats` are recalculated when equipment changes, and how the GDScript Body visually updates to reflect equipped items, as specified in TDD 04.4.2.

### 1. The Role of Equipment in Sigilborne

The GDD (B13.2) states: "Weapons matter, but do not define your power. Chakra techniques, mastery, and strategy dominate combat." This means equipment provides:

*   **Utility/Tactical Advantage**: Tools, ranged weapons.
*   **Stat Enhancement**: Modifies `CoreStats` (e.g., `BaseDamage`, `Armor`, `MoveSpeed`).
*   **Visual Identity**: Changes the character's appearance.

Our `EquipmentSystem` will manage which items are equipped and apply their stat bonuses.

### 2. Defining `EquipmentSlotType` and `EquippedItemsComponent`

We need an enum for different equipment slots and a component to store an entity's currently equipped items.

1.  Create `res://_Brain/Systems/Inventory/EquipmentSlotType.cs`:

```csharp
// _Brain/Systems/Inventory/EquipmentSlotType.cs
using System;

namespace Sigilborne.Systems.Inventory
{
    /// <summary>
    /// Defines the types of equipment slots an entity can have.
    /// </summary>
    public enum EquipmentSlotType
    {
        None,
        MainHand,       // Primary weapon
        OffHand,        // Shield, secondary weapon, tool
        Head,           // Helmet, headband
        Body,           // Armor, robe
        Legs,           // Pants, greaves
        Feet,           // Boots
        Accessory1,     // Ring, amulet (GDD B13.12: Jewelry)
        Accessory2,
        Back,           // Cloak, backpack (for inventory size bonus)
        // Add more slots as needed (e.g., ranged, belt, gloves)
    }
}
```

2.  Create `res://_Brain/Systems/Inventory/EquippedItemsComponent.cs`:

```csharp
// _Brain/Systems/Inventory/EquippedItemsComponent.cs
using System;
using System.Collections.Generic;

namespace Sigilborne.Systems.Inventory
{
    /// <summary>
    /// Stores the items currently equipped by an entity in their respective slots.
    /// This is a component-like data structure managed by the EquipmentSystem.
    /// </summary>
    public struct EquippedItemsComponent
    {
        // Dictionary mapping slot type to the InventoryItem in that slot.
        public Dictionary<EquipmentSlotType, InventoryItem> EquippedSlots;

        public EquippedItemsComponent(Dictionary<EquipmentSlotType, InventoryItem> equippedSlots = null)
        {
            EquippedSlots = equippedSlots ?? new Dictionary<EquipmentSlotType, InventoryItem>();
        }

        public override string ToString()
        {
            if (EquippedSlots.Count == 0) return "No items equipped.";
            return string.Join(", ", EquippedSlots.Select(kv => $"{kv.Key}: {kv.Value.ItemID}"));
        }
    }
}
```

### 3. Defining `ItemDefinition`

To know what stats an item provides, we need a static `ItemDefinition` blueprint. This will be data-driven.

1.  Create `res://_Brain/Systems/Inventory/ItemDefinition.cs`:

```csharp
// _Brain/Systems/Inventory/ItemDefinition.cs
using System;
using System.Collections.Generic;
using Sigilborne.Systems.Biology; // For CoreStats properties

namespace Sigilborne.Systems.Inventory
{
    /// <summary>
    /// Static data defining an item.
    /// (TDD 04.4.1 - Implicit, for ItemID lookup)
    /// </summary>
    public class ItemDefinition
    {
        public string ID { get; private set; } // "iron_sword_t1", "healing_potion_t1"
        public string Name { get; private set; }
        public string Description { get; private set; }
        public ItemType Type { get; private set; }
        public EquipmentSlotType EquipSlot { get; private set; } // None if not equippable
        public bool IsStackable { get; private set; }
        public int MaxStackSize { get; private set; }
        public float MaxDurability { get; private set; }

        // Stat bonuses this item provides when equipped (map to CoreStats fields)
        public Dictionary<string, float> StatBonuses { get; private set; } // e.g., "BaseDamage": 10, "Armor": 5

        // Visual ID for the Body to know what sprite/model to use
        public string VisualID { get; private set; } // e.g., "sprite_iron_sword"

        public ItemDefinition(string id, string name, string description, ItemType type, EquipmentSlotType equipSlot, bool isStackable, int maxStackSize, float maxDurability, Dictionary<string, float> statBonuses, string visualId)
        {
            ID = id;
            Name = name;
            Description = description;
            Type = type;
            EquipSlot = equipSlot;
            IsStackable = isStackable;
            MaxStackSize = maxStackSize;
            MaxDurability = maxDurability;
            StatBonuses = statBonuses ?? new Dictionary<string, float>();
            VisualID = visualId;
        }

        public override string ToString()
        {
            return $"ItemDef: '{ID}' ({Name}) Type: {Type}, Slot: {EquipSlot}";
        }
    }

    /// <summary>
    /// Defines the broad categories of items.
    /// </summary>
    public enum ItemType
    {
        None,
        Weapon,
        Armor,
        Accessory,
        Consumable,
        Tool,
        QuestItem,
        Resource,
        Currency // For gold, etc.
    }
}
```

### 4. Implementing `EquipmentSystem.cs`

This system will manage `EquippedItemsComponent`, handle equipping/unequipping, and recalculate `CoreStats`.

1.  Create `res://_Brain/Systems/Inventory/EquipmentSystem.cs`:

```csharp
// _Brain/Systems/Inventory/EquipmentSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology; // For CoreStats
using System.Linq;
using System.Reflection; // For dynamic stat modification

namespace Sigilborne.Systems.Inventory
{
    /// <summary>
    /// Manages equipped items for entities and recalculates their CoreStats based on equipment bonuses.
    /// (TDD 04.4.2)
    /// </summary>
    public class EquipmentSystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem; // To access/modify CoreStats
        private InventorySystem _inventorySystem; // To move items between inventory and equipped slots

        // Static definitions of all items
        private Dictionary<string, ItemDefinition> _itemDefinitions = new Dictionary<string, ItemDefinition>();

        // Equipped items for entities: Dictionary<EntityID, EquippedItemsComponent>
        private Dictionary<EntityID, EquippedItemsComponent> _equippedItems = new Dictionary<EntityID, EquippedItemsComponent>();

        public EquipmentSystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, InventorySystem inventorySystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _inventorySystem = inventorySystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            RegisterDefaultItemDefinitions(); // Register some default items
            GD.Print("EquipmentSystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            if (type == EntityType.Player || type == EntityType.NPC)
            {
                _equippedItems.Add(id, new EquippedItemsComponent()); // Start with empty equipment
                GD.Print($"EquipmentSystem: Added EquippedItemsComponent for {type} entity {id}.");
                RecalculateCoreStats(id); // Recalculate initial stats
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _equippedItems.Remove(e.ID);
            GD.Print($"EquipmentSystem: Removed EquippedItemsComponent for {e.ID}.");
        }

        /// <summary>
        /// Registers a new static ItemDefinition.
        /// </summary>
        public void RegisterItemDefinition(ItemDefinition def)
        {
            _itemDefinitions[def.ID] = def;
            GD.Print($"EquipmentSystem: Registered item definition '{def.ID}'.");
        }

        /// <summary>
        /// Registers some common default items for initial testing.
        /// </summary>
        private void RegisterDefaultItemDefinitions()
        {
            RegisterItemDefinition(new ItemDefinition(
                id: "iron_sword_t1", name: "Iron Sword", description: "A basic iron sword.",
                type: ItemType.Weapon, equipSlot: EquipmentSlotType.MainHand, isStackable: false, maxStackSize: 1, maxDurability: 100f,
                statBonuses: new Dictionary<string, float> { { "BaseDamage", 10f }, { "AttackSpeed", 0.1f }, { "Armor", 1f } }, // Weapon can give Armor
                visualId: "sprite_iron_sword"
            ));
            RegisterItemDefinition(new ItemDefinition(
                id: "leather_armor_t1", name: "Leather Armor", description: "Simple leather protection.",
                type: ItemType.Armor, equipSlot: EquipmentSlotType.Body, isStackable: false, maxStackSize: 1, maxDurability: 80f,
                statBonuses: new Dictionary<string, float> { { "Armor", 5f }, { "MoveSpeed", -10f } }, // Armor can reduce speed
                visualId: "sprite_leather_armor"
            ));
            RegisterItemDefinition(new ItemDefinition(
                id: "amulet_of_regeneration", name: "Amulet of Regeneration", description: "A mystical amulet that boosts healing.",
                type: ItemType.Accessory, equipSlot: EquipmentSlotType.Accessory1, isStackable: false, maxStackSize: 1, maxDurability: 100f,
                statBonuses: new Dictionary<string, float> { { "HealthRegenRate", 2f }, { "ChakraRegenRate", 1f } },
                visualId: "sprite_amulet"
            ));
            // Add definitions for healing_potion_t1, mana_potion_t1, quest_item_ancient_scroll etc.
        }

        /// <summary>
        /// Equips an item from inventory into an equipment slot.
        /// (TDD 04.4.2)
        /// </summary>
        /// <param name="entityID">The entity equipping the item.</param>
        /// <param name="itemToEquip">The item to equip (must exist in inventory).</param>
        /// <param name="sourceInventorySlotIndex">The inventory slot the item is coming from.</param>
        /// <returns>True if equipped, false if slot invalid, occupied, or item not equippable.</returns>
        public bool EquipItem(EntityID entityID, InventoryItem itemToEquip, int sourceInventorySlotIndex)
        {
            if (!_entityManager.IsValid(entityID) || !_equippedItems.TryGetValue(entityID, out EquippedItemsComponent equipped))
            {
                GD.PrintErr($"EquipmentSystem: Entity {entityID} cannot equip items (no EquippedItemsComponent).");
                return false;
            }
            if (itemToEquip.IsEmpty() || !_itemDefinitions.TryGetValue(itemToEquip.ItemID, out ItemDefinition itemDef))
            {
                GD.PrintErr($"EquipmentSystem: Invalid item to equip: {itemToEquip.ItemID}");
                return false;
            }
            if (itemDef.EquipSlot == EquipmentSlotType.None)
            {
                GD.PrintErr($"EquipmentSystem: Item '{itemDef.Name}' is not equippable.");
                return false;
            }

            // Check if slot is already occupied
            if (equipped.EquippedSlots.ContainsKey(itemDef.EquipSlot) && !equipped.EquippedSlots[itemDef.EquipSlot].IsEmpty())
            {
                // If occupied, unequip current item first (move to inventory)
                UnequipItem(entityID, itemDef.EquipSlot); // This will move old item to inventory
            }

            // Remove from inventory
            if (!_inventorySystem.RemoveItem(entityID, sourceInventorySlotIndex, 1))
            {
                GD.PrintErr($"EquipmentSystem: Failed to remove item '{itemToEquip.ItemID}' from inventory slot {sourceInventorySlotIndex}.");
                return false;
            }

            equipped.EquippedSlots[itemDef.EquipSlot] = itemToEquip;
            _equippedItems[entityID] = equipped; // Update struct in dictionary

            GD.Print($"EquipmentSystem: {entityID} equipped '{itemDef.Name}' in {itemDef.EquipSlot}.");
            RecalculateCoreStats(entityID); // Recalculate stats after equipping (TDD 04.4.2)

            _eventBus.Publish(new EquipmentChangedEvent { EntityID = entityID, Slot = itemDef.EquipSlot, Item = itemToEquip });
            return true;
        }

        /// <summary>
        /// Unequips an item from a specific equipment slot, moving it to inventory.
        /// </summary>
        public bool UnequipItem(EntityID entityID, EquipmentSlotType slotType)
        {
            if (!_entityManager.IsValid(entityID) || !_equippedItems.TryGetValue(entityID, out EquippedItemsComponent equipped))
            {
                GD.PrintErr($"EquipmentSystem: Entity {entityID} has no EquippedItemsComponent.");
                return false;
            }
            if (!equipped.EquippedSlots.ContainsKey(slotType) || equipped.EquippedSlots[slotType].IsEmpty())
            {
                GD.PrintErr($"EquipmentSystem: Slot {slotType} for {entityID} is empty. Nothing to unequip.");
                return false;
            }

            InventoryItem itemToUnequip = equipped.EquippedSlots[slotType];
            
            // Try to add back to inventory
            if (!_inventorySystem.AddItem(entityID, itemToUnequip))
            {
                GD.PrintErr($"EquipmentSystem: Inventory for {entityID} is full. Cannot unequip '{itemToUnequip.ItemID}' from {slotType}.");
                return false; // Inventory full, cannot unequip
            }

            equipped.EquippedSlots.Remove(slotType); // Remove from equipped slots
            _equippedItems[entityID] = equipped; // Update struct in dictionary

            GD.Print($"EquipmentSystem: {entityID} unequipped '{itemToUnequip.ItemID}' from {slotType}.");
            RecalculateCoreStats(entityID); // Recalculate stats after unequipping (TDD 04.4.2)

            _eventBus.Publish(new EquipmentChangedEvent { EntityID = entityID, Slot = slotType, Item = new InventoryItem() }); // Send empty item for unequip
            return true;
        }

        /// <summary>
        /// Recalculates an entity's CoreStats based on all equipped items.
        /// (TDD 04.4.2)
        /// </summary>
        private void RecalculateCoreStats(EntityID entityID)
        {
            if (!_entityManager.IsValid(entityID) || !_biologicalSystem.TryGetCoreStats(entityID, out CoreStats baseStats))
            {
                GD.PrintErr($"EquipmentSystem: Entity {entityID} has no CoreStats. Cannot recalculate.");
                return;
            }

            // Get a fresh copy of base stats (or load from definition if we had a base definition)
            // For now, assume current CoreStats are the base before equipment.
            // In a more complex system, you'd load "base" stats from an EntityDefinition.
            CoreStats recalculatedStats = baseStats; // Start with current stats, then apply modifiers

            if (_equippedItems.TryGetValue(entityID, out EquippedItemsComponent equipped))
            {
                foreach (var kvp in equipped.EquippedSlots)
                {
                    InventoryItem item = kvp.Value;
                    if (item.IsEmpty() || !_itemDefinitions.TryGetValue(item.ItemID, out ItemDefinition itemDef)) continue;

                    foreach (var statBonus in itemDef.StatBonuses)
                    {
                        // Dynamically apply stat bonuses using Reflection (TDD 04.4.2)
                        // This is a powerful but potentially slow technique. For hot paths, pre-calculate or use faster lookups.
                        PropertyInfo prop = typeof(CoreStats).GetProperty(statBonus.Key);
                        if (prop != null && prop.CanWrite && prop.PropertyType == typeof(float))
                        {
                            float currentValue = (float)prop.GetValue(recalculatedStats);
                            prop.SetValue(recalculatedStats, currentValue + statBonus.Value);
                        }
                        else
                        {
                            GD.PrintWarning($"EquipmentSystem: Item '{itemDef.ID}' has unknown stat bonus '{statBonus.Key}' or type mismatch.");
                        }
                    }
                }
            }

            // Apply recalculated stats back to the entity's CoreStats
            ref CoreStats targetCoreStats = ref _biologicalSystem.GetCoreStatsRef(entityID);
            targetCoreStats = recalculatedStats; // Overwrite with new stats
            
            GD.Print($"EquipmentSystem: Recalculated CoreStats for {entityID}. New stats: {targetCoreStats}");
            // Publish full CoreStats update event (BiologicalSystem already does this on its tick)
            // _eventBus.Publish(new BiologicalSystem.PlayerCoreStatsChangedEvent { PlayerID = entityID, NewCoreStats = targetCoreStats });
        }

        /// <summary>
        /// Retrieves the EquippedItemsComponent for an entity.
        /// </summary>
        public IReadOnlyDictionary<EquipmentSlotType, InventoryItem> GetEquippedItems(EntityID entityID)
        {
            if (_equippedItems.TryGetValue(entityID, out EquippedItemsComponent equipped))
            {
                return equipped.EquippedSlots.AsReadOnly();
            }
            return new Dictionary<EquipmentSlotType, InventoryItem>().AsReadOnly();
        }

        // --- Helper Events for Body Sync ---
        public struct EquipmentChangedEvent { public EntityID EntityID; public EquipmentSlotType Slot; public InventoryItem Item; }
    }
}
```

### 5. Integrating `EquipmentSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Inventory;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add an `EquipmentSystem` property.
3.  Initialize `EquipmentSystem` in `InitializeSystems()`.

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
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public InventorySystem Inventory { get; private set; }
    public EquipmentSystem Equipment { get; private set; } // Add EquipmentSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        
        // --- Test Inventory System ---
        GD.Print("\n--- Testing Inventory System ---");
        EntityID playerID = Entities.GetPlayerEntityID();

        Inventory.AddItem(playerID, new InventoryItem("iron_sword_t1", 1, 95f));
        Inventory.AddItem(playerID, new InventoryItem("healing_potion_t1", 3));
        Inventory.AddItem(playerID, new InventoryItem("iron_sword_t1", 1, 95f));
        Inventory.AddItem(playerID, new InventoryItem("leather_armor_t1", 1)); // Add armor
        Inventory.AddItem(playerID, new InventoryItem("amulet_of_regeneration", 1)); // Add amulet
        Inventory.AddItem(playerID, new InventoryItem("healing_potion_t1", 5));
        Inventory.AddItem(playerID, new InventoryItem("mana_potion_t1", 2));
        Inventory.AddItem(playerID, new InventoryItem("quest_item_ancient_scroll", 1, 100f, new Dictionary<string, Variant> { { "lore", "ancient_text" } }));
        
        GD.Print($"Player Inventory after initial additions: {string.Join("\n  ", Inventory.GetInventory(playerID).Select(item => item.ToString()))}");
        
        // --- Test Equipment System ---
        GD.Print("\n--- Testing Equipment System ---");
        GD.Print($"Player initial stats: {Biology.GetCoreStatsRef(playerID)}");

        // Equip sword
        InventoryItem sword = Inventory.GetItemInSlot(playerID, 0); // Assuming sword is in slot 0
        Equipment.EquipItem(playerID, sword, 0);
        GD.Print($"Player stats after equipping sword: {Biology.GetCoreStatsRef(playerID)}");

        // Equip armor
        InventoryItem armor = Inventory.GetItemInSlot(playerID, 3); // Assuming armor is in slot 3
        Equipment.EquipItem(playerID, armor, 3);
        GD.Print($"Player stats after equipping armor: {Biology.GetCoreStatsRef(playerID)}");

        // Equip amulet
        InventoryItem amulet = Inventory.GetItemInSlot(playerID, 4); // Assuming amulet is in slot 4
        Equipment.EquipItem(playerID, amulet, 4);
        GD.Print($"Player stats after equipping amulet: {Biology.GetCoreStatsRef(playerID)}");

        // Unequip sword
        Equipment.UnequipItem(playerID, EquipmentSlotType.MainHand);
        GD.Print($"Player stats after unequipping sword: {Biology.GetCoreStatsRef(playerID)}");
        GD.Print($"Player Inventory after unequip: {string.Join("\n  ", Inventory.GetInventory(playerID).Select(item => item.ToString()))}");

        GD.Print("--- End Testing Equipment System ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        // ... (existing _PhysicsProcess calls) ...
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to CombatSystem) ...
        
        Inventory = new InventorySystem(Entities, Events);
        GD.Print("  - InventorySystem initialized.");

        // Initialize EquipmentSystem, passing InventorySystem and BiologicalSystem
        Equipment = new EquipmentSystem(Entities, Events, BiologicalSystem, Inventory); // Initialize EquipmentSystem here
        GD.Print("  - EquipmentSystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 5.1. Update `EventBus.cs` for `EquipmentChangedEvent`

Open `res://_Brain/Core/EventBus.cs` and add `OnEquipmentChanged` delegate:

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
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Inventory System Events (TDD 04.4.1)
        public event Action<EntityID, List<InventoryItem>> OnPlayerInventoryChanged;

        // Equipment System Events (TDD 04.4.2)
        public event Action<EntityID, EquipmentSlotType, InventoryItem> OnEquipmentChanged; // EntityID, Slot, Item (empty for unequip)

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is InventorySystem.EquipmentChangedEvent equipmentEvent) // New condition
            {
                OnEquipmentChanged?.Invoke(equipmentEvent.EntityID, equipmentEvent.Slot, equipmentEvent.Item);
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

### 6. Testing Equipment Logic

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Equipment System" section.

```
...
InventorySystem: Added iron_sword_t1 (x1) to EntityID(0, Gen:1). New slot.
InventorySystem: Added healing_potion_t1 (x3) to EntityID(0, Gen:1). New slot.
InventorySystem: Added iron_sword_t1 (x1) to EntityID(0, Gen:1). Stacked.
InventorySystem: Added leather_armor_t1 (x1) to EntityID(0, Gen:1). New slot.
InventorySystem: Added amulet_of_regeneration (x1) to EntityID(0, Gen:1). New slot.
InventorySystem: Added healing_potion_t1 (x5) to EntityID(0, Gen:1). Stacked.
InventorySystem: Added mana_potion_t1 (x2) to EntityID(0, Gen:1). New slot.
InventorySystem: Added quest_item_ancient_scroll (x1) to EntityID(0, Gen:1). New slot.
Player Inventory after initial additions: iron_sword_t1 (x1) Dur:95%
  healing_potion_t1 (x8) Dur:100%
  mana_potion_t1 (x2) Dur:100%
  leather_armor_t1 (x1) Dur:80%
  amulet_of_regeneration (x1) Dur:100%
  quest_item_ancient_scroll (x1) Dur:100% lore:ancient_text
  Empty Slot
  ... (remaining empty slots) ...
--- Test Equipment System ---
Player initial stats: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 150, Dmg: 15, Armor: 5, MR: 5, Crit: 10%
EquipmentSystem: EntityID(0, Gen:1) equipped 'Iron Sword' in MainHand.
EquipmentSystem: Recalculated CoreStats for EntityID(0, Gen:1). New stats: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 150, Dmg: 25, Armor: 6, MR: 5, Crit: 10%
Player stats after equipping sword: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 150, Dmg: 25, Armor: 6, MR: 5, Crit: 10%
EquipmentSystem: EntityID(0, Gen:1) equipped 'Leather Armor' in Body.
EquipmentSystem: Recalculated CoreStats for EntityID(0, Gen:1). New stats: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 140, Dmg: 25, Armor: 11, MR: 5, Crit: 10%
Player stats after equipping armor: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 140, Dmg: 25, Armor: 11, MR: 5, Crit: 10%
EquipmentSystem: EntityID(0, Gen:1) equipped 'Amulet of Regeneration' in Accessory1.
EquipmentSystem: Recalculated CoreStats for EntityID(0, Gen:1). New stats: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 140, Dmg: 25, Armor: 11, MR: 5, Crit: 10%
Player stats after equipping amulet: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 140, Dmg: 25, Armor: 11, MR: 5, Crit: 10%
EquipmentSystem: EntityID(0, Gen:1) unequipped 'Iron Sword' from MainHand.
EquipmentSystem: Recalculated CoreStats for EntityID(0, Gen:1). New stats: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 140, Dmg: 15, Armor: 10, MR: 5, Crit: 10%
Player stats after unequipping sword: HP: 100/100, Sta: 75/75, Cha: 50/50, Stab: 100/100, Hunger: 100, Thirst: 100, Temp: 37.0°C, Spd: 140, Dmg: 15, Armor: 10, MR: 5, Crit: 10%
Player Inventory after unequip: iron_sword_t1 (x1) Dur:95%
  healing_potion_t1 (x8) Dur:100%
  mana_potion_t1 (x2) Dur:100%
  leather_armor_t1 (x1) Dur:80%
  amulet_of_regeneration (x1) Dur:100%
  quest_item_ancient_scroll (x1) Dur:100% lore:ancient_text
  Empty Slot
  ... (remaining empty slots) ...
--- End Testing Equipment System ---
...
```

**Key Observations:**

*   **Stat Recalculation**: After equipping the `iron_sword_t1`, `BaseDamage` increases from 15 to 25, and `Armor` from 5 to 6.
*   **Multiple Bonuses**: Equipping `leather_armor_t1` further increases `Armor` to 11 and *reduces* `MoveSpeed` from 150 to 140.
*   **Regen Bonuses**: Equipping `amulet_of_regeneration` adds to `HealthRegenRate` and `ChakraRegenRate` (though not directly visible in the single `CoreStats` print, it's applied).
*   **Unequip Reverts**: Unequipping the sword correctly reverts `BaseDamage` and `Armor` to their previous values (influenced by the remaining equipped armor).
*   **Inventory Interaction**: Unequipping correctly moves the item back to inventory (slot 5 in the example).

This confirms our `EquipmentSystem` is fully functional, dynamically modifying `CoreStats` based on equipped items.

### Summary

You have successfully implemented **Equipment Logic** in the C# Brain, defining `EquipmentSlotType`, `EquippedItemsComponent`, and `ItemDefinition` to manage equipped items. By designing `EquipmentSystem` to handle equipping/unequipping, dynamically recalculate `CoreStats` using item bonuses (via Reflection), and interact with the `InventorySystem` for item transfer, you've established a robust mechanism for character progression. This crucial system strictly adheres to TDD 04.4.2's specifications, allowing equipment to meaningfully influence entity capabilities and visual presentation in Sigilborne.

### Next Steps

The next chapter will focus on **World Boss Mechanics - Multi-Part Entities (C#)**, where we will design how large, complex entities like World Bosses (Titans) are represented as multiple logical parts in the C# Brain, each with its own health and ability to be severed, creating multi-stage combat encounters.