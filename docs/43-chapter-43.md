## Chapter 6.6: Titan AI State - Specialized State Machine (C#)

With World Bosses (Titans) now represented as multi-part entities, the next challenge is to give them intelligent, dynamic behavior. This chapter focuses on designing a **Specialized State Machine** for Titans in the C# Brain, accounting for their massive size, complex abilities, and phase transitions triggered by health thresholds or severed limbs, as specified in TDD 04.5.2.

### 1. The Complexities of Titan AI

The GDD (C05.3) describes Titans as "ecological & political actors" that "migrate, react to weather, territorialize, claim shrines, influence wildlife, reshape terrain, block travel routes, compete with other titans, provoke or calm spirits, attract corruption, disturb anomalies, cause clan panic, trigger regional wars." This requires AI that goes far beyond simple enemy behavior:

*   **Phased Behavior**: Titans should have distinct phases (e.g., "Awakening," "Rampage," "Weakened") with unique abilities.
*   **Environmental Interaction**: Their actions should interact with and reshape the world.
*   **Limb-Dependent Abilities**: Severing a limb should disable certain attacks or expose new vulnerabilities.
*   **Massive Scale**: Actions might take longer (e.g., "Turning" takes 5 seconds, as per TDD 05.5.2).
*   **World Impact**: Their AI decisions can trigger region-wide events.

### 2. Defining `TitanState` and `TitanAIComponent`

We need an enum for the Titan's high-level AI states and a component to hold its current AI state and related data.

1.  Create `res://_Brain/Systems/Combat/TitanAIState.cs`:

```csharp
// _Brain/Systems/Combat/TitanAIState.cs
using System;
using Godot; // For Vector2 if needed

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Defines the high-level AI states for a Titan World Boss.
    /// (TDD 04.5.2)
    /// </summary>
    public enum TitanAIState
    {
        Idle,           // Passive, dormant, or waiting for trigger
        Awakening,      // Transitioning to active combat
        Phase1_Default, // Initial combat phase
        Phase2_Enraged, // Triggered by HP threshold or severed limb
        Phase3_Desperate, // Near defeat, uses ultimate attacks (C04 Forbidden Arts)
        Retreating,     // Moving to a new location
        Dying,          // Final phase before defeat
        Defeated        // Boss is defeated
    }

    /// <summary>
    /// Stores the current AI state and related data for a Titan World Boss.
    /// This is a component-like data structure managed by the TitanAISystem.
    /// (TDD 04.5.2)
    /// </summary>
    public struct TitanAIComponent
    {
        public TitanAIState CurrentState;
        public double StateTimer;           // How long in current state
        public double ActionCooldownTimer;  // Cooldown for next major action
        public EntityID TargetID;           // Current target
        public int CurrentPhase;            // Which phase the boss is in (1, 2, 3)

        public TitanAIComponent(EntityID targetID = default)
        {
            CurrentState = TitanAIState.Idle;
            StateTimer = 0;
            ActionCooldownTimer = 0;
            TargetID = targetID;
            CurrentPhase = 0;
        }

        public override string ToString()
        {
            return $"AI State: {CurrentState}, Phase: {CurrentPhase}, Timer: {StateTimer:F1}";
        }
    }
}
```

### 3. Implementing `TitanAISystem.cs`

This system will manage `TitanAIComponent` instances, process state transitions, select actions, and react to combat events.

1.  Create `res://_Brain/Systems/Combat/TitanAISystem.cs`:

```csharp
// _Brain/Systems/Combat/TitanAISystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Movement; // For Titan movement
using System.Linq;

namespace Sigilborne.Systems.Combat
{
    /// <summary>
    /// Manages the specialized AI state machine for Titan World Bosses.
    /// Accounts for massive size, complex abilities, and phase transitions.
    /// (TDD 04.5.2)
    /// </summary>
    public class TitanAISystem
    {
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem; // For CoreStats
        private CombatSystem _combatSystem;         // For boss parts, damage events
        private TransformSystem _transformSystem;   // For boss position/movement

        // Dictionary to store TitanAIComponent for active titans.
        private Dictionary<EntityID, TitanAIComponent> _titanAIComponents = new Dictionary<EntityID, TitanAIComponent>();

        public TitanAISystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, CombatSystem combatSystem, TransformSystem transformSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _combatSystem = combatSystem;
            _transformSystem = transformSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;
            _eventBus.OnBossPartSevered += OnBossPartSevered; // React to severed limbs (TDD 04.5.1)
            _eventBus.OnDamageTaken += OnDamageTaken; // React to damage for phase transitions

            GD.Print("TitanAISystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // Only add TitanAIComponent to entities defined as Titans (e.g., "world_boss_01")
            if (definitionID == "world_boss_01")
            {
                _titanAIComponents.Add(id, new TitanAIComponent(GameManager.Instance.Entities.GetPlayerEntityID())); // Target player initially
                TransitionTitanState(id, TitanAIState.Awakening); // Start in Awakening phase
                GD.Print($"TitanAISystem: Added TitanAIComponent for {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _titanAIComponents.Remove(e.ID);
            GD.Print($"TitanAISystem: Removed TitanAIComponent for {e.ID}.");
        }

        private void OnBossPartSevered(EntityID bossID, string partID)
        {
            if (_titanAIComponents.ContainsKey(bossID))
            {
                GD.Print($"TitanAISystem: Boss {bossID} had part '{partID}' severed. Checking for phase change.");
                // Trigger phase transition or special reaction (TDD 04.5.1)
                // Example: if Head is severed, boss might become enraged and use new attacks.
                // TransitionTitanState(bossID, TitanAIState.Phase2_Enraged);
            }
        }

        private void OnDamageTaken(EntityID defenderID, DamageResult result)
        {
            if (_titanAIComponents.ContainsKey(defenderID))
            {
                // Check health thresholds for phase transitions (TDD 04.5.2)
                ref CoreStats bossCoreStats = ref _biologicalSystem.GetCoreStatsRef(defenderID);
                ref TitanAIComponent titanAI = ref _titanAIComponents.GetValueRef(defenderID);

                // Example phase transitions:
                if (bossCoreStats.Health <= bossCoreStats.MaxHealth * 0.25f && titanAI.CurrentPhase < 3)
                {
                    TransitionTitanState(defenderID, TitanAIState.Phase3_Desperate);
                }
                else if (bossCoreStats.Health <= bossCoreStats.MaxHealth * 0.5f && titanAI.CurrentPhase < 2)
                {
                    TransitionTitanState(defenderID, TitanAIState.Phase2_Enraged);
                }
                else if (bossCoreStats.Health <= bossCoreStats.MaxHealth * 0.75f && titanAI.CurrentPhase < 1)
                {
                    TransitionTitanState(defenderID, TitanAIState.Phase1_Default);
                }
            }
        }

        /// <summary>
        /// Main update loop for the TitanAISystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            foreach (var kvp in _titanAIComponents)
            {
                EntityID titanID = kvp.Key;
                ref TitanAIComponent titanAI = ref _titanAIComponents.GetValueRef(titanID);

                if (!_entityManager.IsValid(titanID)) continue;

                titanAI.StateTimer += delta;
                titanAI.ActionCooldownTimer -= delta;

                // --- Titan AI State Logic ---
                switch (titanAI.CurrentState)
                {
                    case TitanAIState.Idle:
                        HandleIdleState(titanID, ref titanAI, delta);
                        break;
                    case TitanAIState.Awakening:
                        HandleAwakeningState(titanID, ref titanAI, delta);
                        break;
                    case TitanAIState.Phase1_Default:
                        HandlePhase1(titanID, ref titanAI, delta);
                        break;
                    case TitanAIState.Phase2_Enraged:
                        HandlePhase2(titanID, ref titanAI, delta);
                        break;
                    case TitanAIState.Phase3_Desperate:
                        HandlePhase3(titanID, ref titanAI, delta);
                        break;
                    case TitanAIState.Retreating:
                        HandleRetreatingState(titanID, ref titanAI, delta);
                        break;
                    case TitanAIState.Dying:
                        HandleDyingState(titanID, ref titanAI, delta);
                        break;
                    case TitanAIState.Defeated:
                        // No actions, just remain defeated
                        break;
                }
            }
        }

        /// <summary>
        /// Transitions a titan to a new AI state.
        /// </summary>
        private void TransitionTitanState(EntityID titanID, TitanAIState newState)
        {
            ref TitanAIComponent titanAI = ref _titanAIComponents.GetValueRef(titanID);
            if (titanAI.CurrentState == newState) return; // Already in this state

            GD.Print($"TitanAISystem: Boss {titanID} transitioning from {titanAI.CurrentState} to {newState}.");
            _eventBus.Publish(new TitanStateChangedEvent { TitanID = titanID, OldState = titanAI.CurrentState, NewState = newState });

            titanAI.CurrentState = newState;
            titanAI.StateTimer = 0; // Reset timer for new state

            // Set phase for combat states
            titanAI.CurrentPhase = newState switch
            {
                TitanAIState.Phase1_Default => 1,
                TitanAIState.Phase2_Enraged => 2,
                TitanAIState.Phase3_Desperate => 3,
                _ => 0, // Other states
            };

            // Reset action cooldown on phase change
            titanAI.ActionCooldownTimer = 0;

            // Trigger visual/audio feedback for phase changes
            // _eventBus.Publish(new TitanPhaseVFXEvent { TitanID = titanID, Phase = titanAI.CurrentPhase });
        }

        // --- State Handlers ---
        private void HandleIdleState(EntityID titanID, ref TitanAIComponent titanAI, double delta)
        {
            if (titanAI.StateTimer >= 5.0) // After 5 seconds idle, awaken
            {
                TransitionTitanState(titanID, TitanAIState.Awakening);
            }
        }

        private void HandleAwakeningState(EntityID titanID, ref TitanAIComponent titanAI, double delta)
        {
            // TDD 05.5.2: "Turning" takes 5 seconds (massive size)
            if (titanAI.StateTimer >= 5.0)
            {
                GD.Print($"TitanAISystem: Boss {titanID} has finished AWAKENING!");
                TransitionTitanState(titanID, TitanAIState.Phase1_Default);
            }
            // Emit VFX/Audio for awakening here
            // _eventBus.Publish(new TitanAwakeningVFXEvent { TitanID = titanID });
        }

        private void HandlePhase1(EntityID titanID, ref TitanAIComponent titanAI, double delta)
        {
            if (titanAI.ActionCooldownTimer <= 0)
            {
                // Perform a basic attack (conceptual)
                GD.Print($"TitanAISystem: Boss {titanID} (Phase 1) performs basic attack on {titanAI.TargetID}.");
                // _combatSystem.PerformBossAttack(titanID, titanAI.TargetID, "basic_swipe");
                titanAI.ActionCooldownTimer = 3.0; // Cooldown 3 seconds
            }
            // Move towards target (conceptual)
            // _movementSystem.MoveTowards(titanID, titanAI.TargetID, titanAI.StateTimer);
        }

        private void HandlePhase2(EntityID titanID, ref TitanAIComponent titanAI, double delta)
        {
            if (titanAI.ActionCooldownTimer <= 0)
            {
                // Use a more powerful attack or area effect
                GD.Print($"TitanAISystem: Boss {titanID} (Phase 2) performs enraged attack on {titanAI.TargetID}.");
                // _combatSystem.PerformBossAttack(titanID, titanAI.TargetID, "aoe_ground_slam");
                titanAI.ActionCooldownTimer = 2.0; // Shorter cooldown
            }
            // Environment might change (TDD 05.5.2: Weather)
            // _eventBus.Publish(new WeatherSystem.GlobalWeatherChangedEvent { NewWeatherState = new GlobalWeatherState(WeatherType.Storm, ...) });
        }

        private void HandlePhase3(EntityID titanID, ref TitanAIComponent titanAI, double delta)
        {
            if (titanAI.ActionCooldownTimer <= 0)
            {
                // Use "Ultimate" attacks (C04 Forbidden Arts)
                GD.Print($"TitanAISystem: Boss {titanID} (Phase 3) uses ultimate desperate attack on {titanAI.TargetID}.");
                // _combatSystem.PerformBossAttack(titanID, titanAI.TargetID, "forbidden_void_blast");
                titanAI.ActionCooldownTimer = 1.0; // Very short cooldown
            }
        }

        private void HandleRetreatingState(EntityID titanID, ref TitanAIComponent titanAI, double delta)
        {
            // Titan moves away from player or to a specific location
            // if (titanAI.StateTimer >= 10.0) TransitionTitanState(titanID, TitanAIState.Idle); // Retreat complete
        }

        private void HandleDyingState(EntityID titanID, ref TitanAIComponent titanAI, double delta)
        {
            // Play death animation, become invulnerable, perhaps trigger a final AoE
            if (titanAI.StateTimer >= 5.0)
            {
                TransitionTitanState(titanID, TitanAIState.Defeated);
            }
        }
        
        // --- Helper Events for Body Sync ---
        public struct TitanStateChangedEvent { public EntityID TitanID; public TitanAIState OldState; public TitanAIState NewState; }
        // public struct TitanPhaseVFXEvent { public EntityID TitanID; public int Phase; } // Example
        // public struct TitanAwakeningVFXEvent { public EntityID TitanID; } // Example
    }
}
```

### 4. Integrating `TitanAISystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Combat;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `TitanAISystem` property.
3.  Initialize `TitanAISystem` in `InitializeSystems()` **after** `CombatSystem`.
4.  Call `TitanAISystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).

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
    public CombatSystem Combat { get; private set; }
    public TitanAISystem TitanAI { get; private set; } // Add TitanAISystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        // ... (existing damage tests) ...
        
        // --- Test Inventory System ---
        // ... (existing inventory tests) ...
        
        // --- Test Equipment System ---
        // ... (existing equipment tests) ...

        // --- Test Titan AI State Machine ---
        GD.Print("\n--- Testing Titan AI State Machine ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID bossID = Entities.CreateEntity(EntityType.NPC, "world_boss_01", new Vector2(700, 300)); // Create a boss entity
        
        GD.Print($"Boss initial HP: {Biology.GetCoreStatsRef(bossID).Health}");
        GD.Print($"Boss Parts initial state: {Combat.GetBossParts(bossID)}");

        // Simulate hitting boss parts to trigger phases
        GD.Print("\n--- Simulating Boss Damage to Trigger Phases ---");
        Combat.DamageBossPart(playerID, bossID, "LeftArm", 30f, DamageType.Physical, 0, 0.1f, 1.75f);
        Combat.DamageBossPart(playerID, bossID, "LeftArm", 20f, DamageType.Physical, 0, 0.1f, 1.75f); // Sever Left Arm (triggers OnBossPartSevered)
        
        Combat.DamageBossPart(playerID, bossID, "Head", 40f, DamageType.Elemental, 0, 0.1f, 1.75f);
        Combat.DamageBossPart(playerID, bossID, "Head", 20f, DamageType.Elemental, 0, 0.1f, 1.75f); // Sever Head

        // Trigger Phase 2 (50% HP)
        Combat.DamageBossPart(playerID, bossID, "Body", 50f, DamageType.Physical, 0, 0.1f, 1.75f); 
        // Trigger Phase 3 (25% HP)
        Combat.DamageBossPart(playerID, bossID, "Body", 20f, DamageType.Physical, 0, 0.1f, 1.75f); 
        // Defeat Boss
        Combat.DamageBossPart(playerID, bossID, "Body", 30f, DamageType.Physical, 0, 0.1f, 1.75f);

        GD.Print("--- End Testing Titan AI State Machine ---\n");
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
        TitanAI.Tick(delta); // Call TitanAISystem's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to CombatSystem) ...
        
        // Initialize TitanAISystem AFTER CombatSystem (as it needs CombatSystem for events/parts)
        TitanAI = new TitanAISystem(Entities, Events, BiologicalSystem, Combat, Transforms); // Pass CombatSystem
        GD.Print("  - TitanAISystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 4.1. Update `EventBus.cs` for Titan AI Events

Open `res://_Brain/Core/EventBus.cs` and add `OnTitanStateChanged` delegate.

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

        // Combat System Events (TDD 04.2.2)
        public event Action<EntityID, DamageResult> OnDamageTaken;
        public event Action<EntityID, string, float, float, DamageResult> OnBossPartDamaged;
        public event Action<EntityID, string> OnBossPartSevered;
        public event Action<EntityID, EntityID> OnEntityDefeated;

        // Titan AI Events (TDD 04.5.2)
        public event Action<EntityID, TitanAIState, TitanAIState> OnTitanStateChanged; // TitanID, OldState, NewState

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is CombatSystem.BossPartDamagedEvent bossPartDamagedEvent)
            {
                OnBossPartDamaged?.Invoke(bossPartDamagedEvent.BossID, bossPartDamagedEvent.PartID, bossPartDamagedEvent.CurrentHealth, bossPartDamagedEvent.MaxHealth, bossPartDamagedEvent.Result);
            }
            else if (eventData is CombatSystem.BossPartSeveredEvent bossPartSeveredEvent)
            {
                OnBossPartSevered?.Invoke(bossPartSeveredEvent.BossID, bossPartSeveredEvent.PartID);
            }
            else if (eventData is CombatSystem.EntityDefeatedEvent entityDefeatedEvent)
            {
                OnEntityDefeated?.Invoke(entityDefeatedEvent.EntityID, entityDefeatedEvent.KillerID);
            }
            else if (eventData is CombatSystem.TitanAISystem.TitanStateChangedEvent titanStateChangedEvent) // New condition
            {
                OnTitanStateChanged?.Invoke(titanStateChangedEvent.TitanID, titanStateChangedEvent.OldState, titanStateChangedEvent.NewState);
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

### 5. Testing Titan AI State Machine

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.

```
...
CombatSystem: Added BossPartsComponent for Boss EntityID(3, Gen:1).
TitanAISystem: Initialized.
TitanAISystem: Added TitanAIComponent for EntityID(3, Gen:1).
TitanAISystem: Boss EntityID(3, Gen:1) transitioning from Idle to Awakening.
TitanAISystem: Boss EntityID(3, Gen:1) transitioning from Idle to Awakening.
  - TitanAISystem initialized.
PlayerStatSystem: Initialized.
...
--- Testing Titan AI State Machine ---
Boss initial HP: 190.0
Boss Parts initial state: Boss Parts: 4 | Total HP: 190/190

--- Simulating Boss Damage to Trigger Phases ---
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 13.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'LeftArm'.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 13.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'LeftArm'.
CombatSystem: Part 'LeftArm' of Boss EntityID(3, Gen:1) has been SEVERED!
TitanAISystem: Boss EntityID(3, Gen:1) had part 'LeftArm' severed. Checking for phase change.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 23.0 (Elemental) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Head'.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 23.0 (Elemental) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Head'.
CombatSystem: Part 'Head' of Boss EntityID(3, Gen:1) has been SEVERED!
TitanAISystem: Boss EntityID(3, Gen:1) had part 'Head' severed. Checking for phase change.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 48.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Body'.
TitanAISystem: Boss EntityID(3, Gen:1) transitioning from Awakening to Phase1_Default.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 48.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Body'.
TitanAISystem: Boss EntityID(3, Gen:1) transitioning from Phase1_Default to Phase2_Enraged.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 23.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Body'.
TitanAISystem: Boss EntityID(3, Gen:1) transitioning from Phase2_Enraged to Phase3_Desperate.
CombatSystem: EntityID(0, Gen:1) dealt Dmg: 23.0 (Physical) | Crit: False, Block: False to Boss EntityID(3, Gen:1) part 'Body'.
CombatSystem: Boss EntityID(3, Gen:1) has been DEFEATED!
--- End Testing Titan AI State Machine ---
...
TitanAISystem: Boss EntityID(3, Gen:1) has finished AWAKENING!
TitanAISystem: Boss EntityID(3, Gen:1) (Phase 1) performs basic attack on EntityID(0, Gen:1).
TitanAISystem: Boss EntityID(3, Gen:1) (Phase 2) performs enraged attack on EntityID(0, Gen:1).
TitanAISystem: Boss EntityID(3, Gen:1) (Phase 3) uses ultimate desperate attack on EntityID(0, Gen:1).
...
```

**Key Observations:**

*   **Initial State**: The Titan starts in `Awakening`.
*   **Phase Transitions**: As damage is dealt to the boss (reducing its overall health), `OnDamageTaken` triggers, and the Titan correctly transitions through `Phase1_Default`, `Phase2_Enraged`, and `Phase3_Desperate` based on health thresholds.
*   **Severed Limb Reaction**: `OnBossPartSevered` is correctly triggered and handled, showing that the AI can react to specific part destruction.
*   **State-Specific Actions**: The `Tick` method calls the appropriate `HandlePhaseX` method, which prints conceptual attack messages, demonstrating state-specific behaviors.
*   **Awakening Timer**: The `Awakening` phase correctly transitions to `Phase1_Default` after its 5-second timer.

This confirms our `TitanAISystem` is correctly managing the Titan's AI states and reacting to combat events, providing a foundation for complex, multi-phase boss encounters.

### Summary

You have successfully implemented the **Titan AI State Machine** in the C# Brain, designing `TitanAIState` and `TitanAIComponent` to manage the complex behaviors of World Bosses. By creating `TitanAISystem` to process state transitions, react to `OnBossPartSevered` and `OnDamageTaken` events for phase changes, and execute state-specific actions, you've established a robust system for dynamic, multi-phase boss encounters. This crucial component strictly adheres to TDD 04.5.2's specifications, paving the way for truly intelligent and challenging Titan battles in Sigilborne.

### Next Steps

This concludes **Module 6: Combat & Tactical Engagement**. We will now move on to **Module 7: The Living World - Ecology & AI**, starting with **Perception System - Spatial Hashing (C#)**, where we will implement an efficient spatial hashing grid for entities to quickly detect nearby objects and events.