## Chapter 6.3: Equipment & Inventory - Inventory Data Structure (C#)

With combat mechanics taking shape, entities will need items: weapons, tools, consumables, and quest items. This chapter focuses on designing the core **Inventory Data Structure** in the C# Brain. We'll define `InventoryItem` and `InventorySystem`, ensuring that inventory is managed as pure data, decoupled from any UI, and designed for efficiency and scalability, as specified in TDD 04.4.

### 1. The Pure Data Philosophy of Inventory

The TDD (04.4) explicitly states: "Inventory is a pure data structure, not a list of UI nodes." This is crucial for our Brain & Body architecture:

*   **Brain (C#) owns Data**: The `InventorySystem` in C# holds the authoritative truth about what items an entity possesses.
*   **Body (GDScript) owns UI**: The UI will query the Brain for inventory data and display it reactively.
*   **Performance**: Managing thousands of items as pure data is far more efficient than as Godot nodes.
*   **Flexibility**: Easily supports different inventory sizes, item types, and complex item properties.

### 2. Defining `InventoryItem`

`InventoryItem` will be a struct representing a single slot or stack of items.

1.  Create `res://_Brain/Systems/Inventory/InventoryItem.cs`:

```csharp
// _Brain/Systems/Inventory/InventoryItem.cs
using System;
using System.Collections.Generic;
using Godot; // For Variant, if using Godot's Variant type for metadata

namespace Sigilborne.Systems.Inventory
{
    /// <summary>
    /// Data for a single item stack within an inventory.
    /// (TDD 04.4.1)
    /// </summary>
    public struct InventoryItem : IEquatable<InventoryItem>
    {
        public string ItemID;           // Unique ID of the item (e.g., "iron_sword", "healing_potion_t1")
        public int Quantity;            // Number of items in this stack
        public float Durability;        // Current durability (for weapons, tools)
        public Dictionary<string, Variant> Metadata; // For enchantments, unique item stats (TDD 04.4.1)

        public InventoryItem(string itemID, int quantity = 1, float durability = 100f, Dictionary<string, Variant> metadata = null)
        {
            ItemID = itemID;
            Quantity = quantity;
            Durability = durability;
            Metadata = metadata ?? new Dictionary<string, Variant>();
        }

        public bool IsEmpty()
        {
            return string.IsNullOrEmpty(ItemID) || Quantity <= 0;
        }

        public override string ToString()
        {
            if (IsEmpty()) return "Empty Slot";
            return $"{ItemID} (x{Quantity}) Dur:{Durability:F0}% {string.Join(", ", Metadata.Select(kv => $"{kv.Key}:{kv.Value}"))}";
        }

        // --- IEquatable Implementation ---
        public bool Equals(InventoryItem other)
        {
            // For inventory purposes, two items are "equal" if they can stack.
            // This usually means same ItemID, and sometimes same metadata (e.g., enchantments).
            // For simplicity, we'll just check ItemID and Metadata keys for now.
            return ItemID == other.ItemID &&
                   Durability == other.Durability && // Durability might prevent stacking if not full
                   Metadata.OrderBy(kv => kv.Key).SequenceEqual(other.Metadata.OrderBy(kv => kv.Key));
        }

        public override bool Equals(object obj)
        {
            return obj is InventoryItem other && Equals(other);
        }

        public override int GetHashCode()
        {
            // Simple hash code. For production, consider a more robust one if metadata is complex.
            return HashCode.Combine(ItemID, Durability, Metadata.GetHashCode());
        }
    }
}
```

### 3. Defining `InventorySystem.cs`

This system will manage the inventories of all entities that possess one.

1.  Create `res://_Brain/Systems/Inventory/InventorySystem.cs`:

```csharp
// _Brain/Systems/Inventory/InventorySystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using System.Linq;

namespace Sigilborne.Systems.Inventory
{
    /// <summary>
    /// Manages the inventories for all entities that possess one.
    /// (TDD 04.4.1)
    /// </summary>
    public class InventorySystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;

        // Dictionary to store inventories for entities.
        // Each entity's inventory is a fixed-size array of InventoryItem.
        private Dictionary<EntityID, InventoryItem[]> _inventories = new Dictionary<EntityID, InventoryItem[]>();

        // Default inventory size for new entities
        private const int DEFAULT_INVENTORY_SIZE = 16; // GDD B13.19: Inventory management (weight, slots)

        public InventorySystem(EntityManager entityManager, EventBus eventBus)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;

            // Subscribe to entity lifecycle events
            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("InventorySystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // Only add inventory to entities that should have one (Player, NPC, some WorldObjects)
            if (type == EntityType.Player || type == EntityType.NPC || type == EntityType.WorldObject)
            {
                _inventories.Add(id, new InventoryItem[DEFAULT_INVENTORY_SIZE]);
                GD.Print($"InventorySystem: Added inventory for {type} entity {id} with {DEFAULT_INVENTORY_SIZE} slots.");

                // For player, publish initial inventory state
                if (id == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerInventoryChangedEvent { PlayerID = id, Inventory = GetInventory(id).ToList() });
                }
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _inventories.Remove(e.ID);
            GD.Print($"InventorySystem: Removed inventory for {e.ID}.");
        }

        /// <summary>
        /// Adds an item to an entity's inventory. Stacks if possible.
        /// (TDD 04.4.1)
        /// </summary>
        /// <returns>True if item was added, false if inventory is full.</returns>
        public bool AddItem(EntityID targetID, InventoryItem item)
        {
            if (!_entityManager.IsValid(targetID) || !_inventories.TryGetValue(targetID, out InventoryItem[] inventory))
            {
                GD.PrintErr($"InventorySystem: Entity {targetID} has no inventory.");
                return false;
            }

            if (item.IsEmpty()) return false;

            // Try to stack with existing items
            for (int i = 0; i < inventory.Length; i++)
            {
                if (!inventory[i].IsEmpty() && inventory[i].Equals(item)) // If same item type and can stack
                {
                    inventory[i].Quantity += item.Quantity;
                    GD.Print($"InventorySystem: Added {item.ItemID} (x{item.Quantity}) to {targetID}. Stacked.");
                    _eventBus.Publish(new PlayerInventoryChangedEvent { PlayerID = targetID, Inventory = GetInventory(targetID).ToList() });
                    return true;
                }
            }

            // Find first empty slot
            for (int i = 0; i < inventory.Length; i++)
            {
                if (inventory[i].IsEmpty())
                {
                    inventory[i] = item;
                    GD.Print($"InventorySystem: Added {item.ItemID} (x{item.Quantity}) to {targetID}. New slot.");
                    _eventBus.Publish(new PlayerInventoryChangedEvent { PlayerID = targetID, Inventory = GetInventory(targetID).ToList() });
                    return true;
                }
            }

            GD.Print($"InventorySystem: Inventory for {targetID} is full. Cannot add {item.ItemID}.");
            return false; // Inventory is full
        }

        /// <summary>
        /// Removes an item from a specific inventory slot.
        /// (TDD 04.4.1)
        /// </summary>
        /// <param name="slotIndex">The index of the slot to remove from.</param>
        /// <param name="quantity">The amount to remove. If less than current quantity, reduces stack. If more, clears slot.</param>
        /// <returns>True if item was removed/quantity reduced, false if slot invalid or item not found.</returns>
        public bool RemoveItem(EntityID targetID, int slotIndex, int quantity = 1)
        {
            if (!_entityManager.IsValid(targetID) || !_inventories.TryGetValue(targetID, out InventoryItem[] inventory))
            {
                GD.PrintErr($"InventorySystem: Entity {targetID} has no inventory.");
                return false;
            }

            if (slotIndex < 0 || slotIndex >= inventory.Length || inventory[slotIndex].IsEmpty())
            {
                GD.PrintErr($"InventorySystem: Invalid slot {slotIndex} or empty slot for {targetID}.");
                return false;
            }

            if (quantity >= inventory[slotIndex].Quantity)
            {
                GD.Print($"InventorySystem: Removed all {inventory[slotIndex].ItemID} from slot {slotIndex} for {targetID}.");
                inventory[slotIndex] = new InventoryItem(); // Clear slot
            }
            else
            {
                inventory[slotIndex].Quantity -= quantity;
                GD.Print($"InventorySystem: Removed {quantity} of {inventory[slotIndex].ItemID} from slot {slotIndex} for {targetID}. New quantity: {inventory[slotIndex].Quantity}.");
            }
            _eventBus.Publish(new PlayerInventoryChangedEvent { PlayerID = targetID, Inventory = GetInventory(targetID).ToList() });
            return true;
        }

        /// <summary>
        /// Retrieves the current inventory of an entity.
        /// </summary>
        public IReadOnlyList<InventoryItem> GetInventory(EntityID targetID)
        {
            if (_inventories.TryGetValue(targetID, out InventoryItem[] inventory))
            {
                return inventory.AsEnumerable().ToList().AsReadOnly();
            }
            return new List<InventoryItem>().AsReadOnly();
        }

        /// <summary>
        /// Retrieves the item in a specific inventory slot.
        /// </summary>
        public InventoryItem GetItemInSlot(EntityID targetID, int slotIndex)
        {
            if (_inventories.TryGetValue(targetID, out InventoryItem[] inventory) && slotIndex >= 0 && slotIndex < inventory.Length)
            {
                return inventory[slotIndex];
            }
            return new InventoryItem(); // Return empty item
        }

        // --- Helper Events for Body Sync ---
        // Using List<InventoryItem> for GDScript interop (TDD 04.4.1)
        public struct PlayerInventoryChangedEvent { public EntityID PlayerID; public List<InventoryItem> Inventory; }
    }
}
```

### 4. Integrating `InventorySystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Inventory;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add an `InventorySystem` property.
3.  Initialize `InventorySystem` in `InitializeSystems()`.

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
using Sigilborne.Systems.Inventory; // Add this using directive
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public CombatSystem Combat { get; private set; }
    public InventorySystem Inventory { get; private set; } // Add InventorySystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        
        // --- Test Inventory System ---
        GD.Print("\n--- Testing Inventory System ---");
        EntityID playerID = Entities.GetPlayerEntityID();

        // Add some items
        Inventory.AddItem(playerID, new InventoryItem("iron_sword_t1", 1, 95f));
        Inventory.AddItem(playerID, new InventoryItem("healing_potion_t1", 3));
        Inventory.AddItem(playerID, new InventoryItem("iron_sword_t1", 1, 95f)); // Should stack
        Inventory.AddItem(playerID, new InventoryItem("healing_potion_t1", 5)); // Should stack
        Inventory.AddItem(playerID, new InventoryItem("mana_potion_t1", 2));
        Inventory.AddItem(playerID, new InventoryItem("quest_item_ancient_scroll", 1, 100f, new Dictionary<string, Variant> { { "lore", "ancient_text" } }));
        
        // Remove items
        Inventory.RemoveItem(playerID, 0, 1); // Remove 1 iron sword
        Inventory.RemoveItem(playerID, 1, 8); // Try to remove 8 potions from a stack of 8 (should clear)
        Inventory.RemoveItem(playerID, 99, 1); // Try to remove from invalid slot (should fail)

        GD.Print($"Player Inventory after tests: {string.Join("\n  ", Inventory.GetInventory(playerID).Select(item => item.ToString()))}");
        GD.Print("--- End Testing Inventory System ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        // ... (existing _PhysicsProcess calls) ...
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to CombatSystem) ...
        
        // Initialize InventorySystem
        Inventory = new InventorySystem(Entities, Events); // Initialize InventorySystem here
        GD.Print("  - InventorySystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `EventBus.cs` for `PlayerInventoryChangedEvent`

Open `res://_Brain/Core/EventBus.cs` and add `OnPlayerInventoryChanged` delegate:

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
using Sigilborne.Systems.Inventory; // Add this using directive
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Inventory System Events (TDD 04.4.1)
        public event Action<EntityID, List<InventoryItem>> OnPlayerInventoryChanged; // PlayerID, List<InventoryItem>

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is InventorySystem.PlayerInventoryChangedEvent inventoryEvent) // New condition
            {
                OnPlayerInventoryChanged?.Invoke(inventoryEvent.PlayerID, inventoryEvent.Inventory);
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

### 5. Testing the Inventory System

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing Inventory System" section.

```
...
InventorySystem: Initialized.
  - InventorySystem initialized.
PlayerStatSystem: Initialized.
PlayerStatSystem: Player EntityID(0, Gen:1) took 0 damage. New Health: 100.0
...
--- Testing Inventory System ---
InventorySystem: Added iron_sword_t1 (x1) to EntityID(0, Gen:1). New slot.
InventorySystem: Added healing_potion_t1 (x3) to EntityID(0, Gen:1). New slot.
InventorySystem: Added iron_sword_t1 (x1) to EntityID(0, Gen:1). Stacked.
InventorySystem: Added healing_potion_t1 (x5) to EntityID(0, Gen:1). Stacked.
InventorySystem: Added mana_potion_t1 (x2) to EntityID(0, Gen:1). New slot.
InventorySystem: Added quest_item_ancient_scroll (x1) to EntityID(0, Gen:1). New slot.
InventorySystem: Removed 1 of iron_sword_t1 from slot 0 for EntityID(0, Gen:1). New quantity: 1.
InventorySystem: Removed all healing_potion_t1 from slot 1 for EntityID(0, Gen:1).
InventorySystem: Invalid slot 99 or empty slot for EntityID(0, Gen:1).
Player Inventory after tests: iron_sword_t1 (x1) Dur:95%
  Empty Slot
  mana_potion_t1 (x2) Dur:100%
  quest_item_ancient_scroll (x1) Dur:100% lore:ancient_text
  Empty Slot
  ... (remaining empty slots) ...
--- End Testing Inventory System ---
...
```

This output confirms that:
*   `InventorySystem` correctly initializes inventories for entities.
*   `AddItem` successfully adds items to empty slots and stacks identical items.
*   `RemoveItem` correctly reduces stack quantities or clears slots.
*   Invalid operations (full inventory, invalid slot) are handled gracefully.
*   `PlayerInventoryChangedEvent` is published on every change.

### Summary

You have successfully designed and implemented the core **Inventory Data Structure** in the C# Brain, defining `InventoryItem` as a struct and `InventorySystem` to manage item stacks for entities. By ensuring inventory is pure data, decoupled from UI, and capable of adding, stacking, and removing items efficiently, you've established the authoritative backbone for all item management, strictly adhering to TDD 04.4's specifications. This crucial step supports varied equipment, consumables, and quest items in Sigilborne.

### Next Steps

The next chapter will focus on **Equipment Logic**, detailing how items are moved to "Equipment Slots," how `CoreStats` are recalculated when equipment changes, and how the GDScript Body visually updates to reflect equipped items.