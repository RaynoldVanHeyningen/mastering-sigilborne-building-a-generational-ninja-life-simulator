## Chapter 7.4: Decision Making: Utility AI (C#)

With entities now capable of perceiving their environment and managing their core stats, it's time to give NPCs the ability to make intelligent decisions. This chapter focuses on implementing **Utility AI** (scoring-based AI) in the C# Brain, allowing NPCs to select actions (e.g., attack, flee, eat, sleep) based on a scoring mechanism that evaluates their current needs and environmental context, as specified in TDD 05.3.

### 1. The Power of Utility AI

Traditional Finite State Machines (FSMs) can become complex and rigid for games with many interconnected systems. Utility AI offers a more flexible, emergent, and scalable approach:

*   **Dynamic Decision-Making**: NPCs evaluate all possible actions and choose the one with the highest "utility score" for their current situation.
*   **Emergent Behavior**: Complex behaviors arise from simple scoring rules, leading to less predictable and more lifelike NPCs.
*   **Scalability**: Easily add new actions or modify priorities by adjusting scores, without rewriting state transitions.
*   **GDD Alignment**: Supports NPCs with evolving literacy, clan loyalties, and dynamic schedules (B18).

### 2. Defining `AIComponent` and `UtilityAction`

We need a component to hold an entity's AI parameters and a struct to represent a single action the AI can take.

1.  Create `res://_Brain/Systems/AI/AIComponent.cs`:

```csharp
// _Brain/Systems/AI/AIComponent.cs
using System;
using Godot; // For Vector2

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Stores the parameters defining an entity's AI behavior.
    /// This is a component-like data structure managed by the AISystem.
    /// </summary>
    public struct AIComponent
    {
        public float Aggression;        // How likely to attack (0-1)
        public float Curiosity;         // How likely to investigate anomalies (0-1)
        public float HungerThreshold;   // Below this hunger, prioritize eating
        public float ThirstThreshold;   // Below this thirst, prioritize drinking
        public float FearLevel;         // How easily scared (0-1)
        public float CombatRange;       // How close to get to target in combat
        public float FleeThreshold;     // Below this health, prioritize fleeing (0-1)

        public AIComponent(float aggression = 0.5f, float curiosity = 0.5f, float hungerThreshold = 30f, float thirstThreshold = 20f, float fearLevel = 0.5f, float combatRange = 100f, float fleeThreshold = 0.2f)
        {
            Aggression = aggression;
            Curiosity = curiosity;
            HungerThreshold = hungerThreshold;
            ThirstThreshold = thirstThreshold;
            FearLevel = fearLevel;
            CombatRange = combatRange;
            FleeThreshold = fleeThreshold;
        }

        public override string ToString()
        {
            return $"Agg: {Aggression:F2}, Cur: {Curiosity:F2}, Flee: {FleeThreshold:P0}";
        }
    }
}
```

2.  Create `res://_Brain/Systems/AI/UtilityAction.cs`:

```csharp
// _Brain/Systems/AI/UtilityAction.cs
using System;
using System.Collections.Generic;

namespace Sigilborne.Systems.AI
{
    /// <summary>
    /// Represents a single action an AI entity can take, along with its scoring function.
    /// (TDD 05.3: The Scorer)
    /// </summary>
    public class UtilityAction
    {
        public string Name { get; private set; }
        public Func<EntityID, float> ScoreFunction { get; private set; } // Function to calculate action's utility score
        public Action<EntityID> ExecuteAction { get; private set; } // Function to execute the action

        public UtilityAction(string name, Func<EntityID, float> scoreFunction, Action<EntityID> executeAction)
        {
            Name = name;
            ScoreFunction = scoreFunction;
            ExecuteAction = executeAction;
        }

        public override string ToString()
        {
            return $"Action: '{Name}'";
        }
    }
}
```

### 4. Implementing `AISystem.cs`

This system will:
*   Manage `AIComponent`s.
*   Register a set of `UtilityAction`s.
*   In its `Tick` method, for each AI entity, it will:
    *   Query nearby entities using `PerceptionSystem`.
    *   Score all available `UtilityAction`s.
    *   Execute the highest-scoring action.

1.  Create `res://_Brain/Systems/AI/AISystem.cs`:

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
        private EntityManager _entityManager;
        private EventBus _eventBus;
        private BiologicalSystem _biologicalSystem; // For CoreStats
        private PerceptionSystem _perceptionSystem; // For detecting targets
        private CombatSystem _combatSystem;         // For attacking
        private TransformSystem _transformSystem;   // For movement

        // Dictionary to store AIComponent for entities that have AI.
        private Dictionary<EntityID, AIComponent> _aiComponents = new Dictionary<EntityID, AIComponent>();

        // List of all available UtilityActions for NPCs to choose from.
        private List<UtilityAction> _availableActions = new List<UtilityAction>();

        // To prevent NPCs from making decisions every single tick (performance)
        private const float AI_DECISION_INTERVAL = 0.5f; // Every 0.5 real seconds
        private Dictionary<EntityID, float> _aiDecisionTimers = new Dictionary<EntityID, float>();


        public AISystem(EntityManager entityManager, EventBus eventBus, BiologicalSystem biologicalSystem, PerceptionSystem perceptionSystem, CombatSystem combatSystem, TransformSystem transformSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _biologicalSystem = biologicalSystem;
            _perceptionSystem = perceptionSystem;
            _combatSystem = combatSystem;
            _transformSystem = transformSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;
            // AISystem needs to know when player is alerted for NPC reactions
            _eventBus.OnPlayerAlerted += OnPlayerAlerted;

            RegisterDefaultActions(); // Register the AI's repertoire of actions
            GD.Print("AISystem: Initialized.");
        }

        private void OnEntitySpawned(EntityID id, EntityType type, string definitionID, Vector2 initialPosition, float initialRotation)
        {
            // Only add AIComponent to NPCs and some Animals
            if (type == EntityType.NPC || type == EntityType.Animal)
            {
                // Default AI for NPCs/Animals
                _aiComponents.Add(id, new AIComponent());
                _aiDecisionTimers.Add(id, (float)GameManager.Instance.Time.CurrentGameTime); // Initialize timer
                GD.Print($"AISystem: Added AIComponent for {type} entity {id}.");
            }
        }

        private void OnEntityDespawned(EntityManager.EntityDespawnedEvent e)
        {
            _aiComponents.Remove(e.ID);
            _aiDecisionTimers.Remove(e.ID);
            GD.Print($"AISystem: Removed AIComponent for {e.ID}.");
        }

        private void OnPlayerAlerted(EntityID playerID, EntityID perceiverID, bool isAlerted)
        {
            if (_aiComponents.ContainsKey(perceiverID))
            {
                // If this NPC is the perceiver, react to player being alerted.
                // This might override their current action to prioritize combat/flee.
                GD.Print($"AISystem: NPC {perceiverID} detected player {playerID} and is now alerted: {isAlerted}.");
                // For now, this is just a print. Later, it would force an immediate AI decision.
            }
        }

        /// <summary>
        /// Registers a new UtilityAction into the AI's repertoire.
        /// </summary>
        private void RegisterAction(UtilityAction action)
        {
            _availableActions.Add(action);
        }

        /// <summary>
        /// Registers the default set of actions available to NPCs.
        /// (TDD 05.3: The Scorer - Example Actions)
        /// </summary>
        private void RegisterDefaultActions()
        {
            // --- Action: Flee (TDD 05.3) ---
            RegisterAction(new UtilityAction("Flee", (npcID) =>
            {
                if (!_biologicalSystem.TryGetCoreStats(npcID, out CoreStats stats) || !_aiComponents.TryGetValue(npcID, out AIComponent ai)) return 0;
                // Score higher if health is low (below flee threshold) and fear is high.
                float healthScore = (1f - (stats.Health / stats.MaxHealth)); // 1 if 0 HP, 0 if 100 HP
                return healthScore > ai.FleeThreshold ? healthScore * ai.FearLevel * 2 : 0; // Double score if below threshold
            }, (npcID) =>
            {
                GD.Print($"AISystem: NPC {npcID} is fleeing!");
                // Conceptual: Tell MovementSystem to move away from player
                // _movementSystem.MoveAwayFrom(npcID, _entityManager.GetPlayerEntityID(), 200f);
            }));

            // --- Action: Attack Player (TDD 05.3) ---
            RegisterAction(new UtilityAction("AttackPlayer", (npcID) =>
            {
                if (!_biologicalSystem.TryGetCoreStats(npcID, out CoreStats stats) || !_aiComponents.TryGetValue(npcID, out AIComponent ai)) return 0;
                // Score higher if player is detected and NPC is aggressive.
                List<EntityID> detected = _perceptionSystem.GetDetectedEntities(npcID);
                if (detected.Contains(_entityManager.GetPlayerEntityID()))
                {
                    // Score higher if player is close enough to combat range and NPC is aggressive.
                    if (_transformSystem.TryGetTransform(npcID, out TransformComponent npcT) && _transformSystem.TryGetTransform(_entityManager.GetPlayerEntityID(), out TransformComponent playerT))
                    {
                        float distance = npcT.Position.DistanceTo(playerT.Position);
                        if (distance <= ai.CombatRange)
                        {
                            return ai.Aggression * (1f - (stats.Health / stats.MaxHealth) * ai.FleeThreshold); // Aggression, reduced if health too low
                        }
                    }
                }
                return 0;
            }, (npcID) =>
            {
                GD.Print($"AISystem: NPC {npcID} is attacking player {_entityManager.GetPlayerEntityID()}!");
                // Conceptual: Tell CombatSystem to perform an attack
                // _combatSystem.PerformNPCAttack(npcID, _entityManager.GetPlayerEntityID());
            }));

            // --- Action: Eat Food (TDD 05.3) ---
            RegisterAction(new UtilityAction("EatFood", (npcID) =>
            {
                if (!_biologicalSystem.TryGetCoreStats(npcID, out CoreStats stats) || !_aiComponents.TryGetValue(npcID, out AIComponent ai)) return 0;
                // Score higher if hunger is low (below threshold)
                return stats.Hunger < ai.HungerThreshold ? (ai.HungerThreshold - stats.Hunger) / ai.HungerThreshold : 0;
            }, (npcID) =>
            {
                GD.Print($"AISystem: NPC {npcID} is eating food!");
                // Conceptual: Find nearby food item, move to it, consume it.
                // _inventorySystem.UseItem(npcID, "generic_food");
            }));

            // --- Action: Drink Water (TDD 05.3) ---
            RegisterAction(new UtilityAction("DrinkWater", (npcID) =>
            {
                if (!_biologicalSystem.TryGetCoreStats(npcID, out CoreStats stats) || !_aiComponents.TryGetValue(npcID, out AIComponent ai)) return 0;
                return stats.Thirst < ai.ThirstThreshold ? (ai.ThirstThreshold - stats.Thirst) / ai.ThirstThreshold : 0;
            }, (npcID) =>
            {
                GD.Print($"AISystem: NPC {npcID} is drinking water!");
                // Conceptual: Find nearby water source, move to it, drink.
            }));

            // --- Action: Wander (Default/Idle) ---
            RegisterAction(new UtilityAction("Wander", (npcID) =>
            {
                // Low base score, increases if no higher priority actions are scored.
                return 0.1f;
            }, (npcID) =>
            {
                GD.Print($"AISystem: NPC {npcID} is wandering aimlessly.");
                // Conceptual: Tell MovementSystem to move to a random nearby point.
                // _movementSystem.MoveToRandom(npcID, 50f);
            }));
        }

        /// <summary>
        /// Main update loop for the AISystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            foreach (var kvp in _aiComponents)
            {
                EntityID npcID = kvp.Key;
                ref float decisionTimer = ref _aiDecisionTimers.GetValueRef(npcID); // Get mutable timer

                if (!_entityManager.IsValid(npcID)) continue;

                decisionTimer += (float)delta;
                if (decisionTimer >= AI_DECISION_INTERVAL)
                {
                    MakeDecision(npcID);
                    decisionTimer = 0;
                }
            }
        }

        /// <summary>
        /// Evaluates all available actions for an NPC and executes the highest-scoring one.
        /// (TDD 05.3: Algorithm)
        /// </summary>
        private void MakeDecision(EntityID npcID)
        {
            UtilityAction bestAction = null;
            float highestScore = 0;

            foreach (UtilityAction action in _availableActions)
            {
                float score = action.ScoreFunction.Invoke(npcID);
                if (score > highestScore)
                {
                    highestScore = score;
                    bestAction = action;
                }
            }

            if (bestAction != null)
            {
                GD.Print($"AISystem: NPC {npcID} chose action '{bestAction.Name}' with score {highestScore:F2}.");
                bestAction.ExecuteAction.Invoke(npcID);
            }
            else
            {
                GD.Print($"AISystem: NPC {npcID} found no suitable action.");
            }
        }
    }
}
```

### 5. Integrating `AISystem` into `GameManager`

1.  Add `using Sigilborne.Systems.AI;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add an `AISystem` property.
3.  Initialize `AISystem` in `InitializeSystems()` **after** `PerceptionSystem`, `CombatSystem`, and `BiologicalSystem`.
4.  Call `AISystem.Tick(delta)` in `_PhysicsProcess` (Phase 2).

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
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public StealthSystem Stealth { get; private set; }
    public AISystem AI { get; private set; } // Add AISystem property

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
        
        // --- Test AI System ---
        GD.Print("\n--- Testing AI System ---");
        EntityID playerID = Entities.GetPlayerEntityID();
        EntityID testNpcID = Entities.GetEntityMeta(1).Generation == 1 ? new EntityID(1, 1) : EntityID.Invalid; // Assuming NPC is ID 1, Gen 1
        
        if (playerID.IsValid() && testNpcID.IsValid())
        {
            // Place NPC near player for initial detection/attack behavior
            Transforms.GetTransformRef(playerID).Position = new Vector2(200, 200);
            Transforms.GetTransformRef(testNpcID).Position = new Vector2(250, 200); // Within combat range
            Transforms.GetTransformRef(playerID).RotationDegrees = 0; // Player faces right

            GD.Print($"NPC {testNpcID} initial stats: {Biology.GetCoreStatsRef(testNpcID)}");
            GD.Print($"Player {playerID} initial stealth: {Stealth.GetStealthComponentRef(playerID)}");
            GD.Print($"NPC {testNpcID} initial perception: {Perception.GetPerceptionComponentRef(testNpcID)}");
        }
        else
        {
            GD.PrintErr("Player or Test NPC not valid for AI tests.");
        }

        GD.Print("--- End Testing AI System ---\n");
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
        AI.Tick(delta); // Call AISystem's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to StealthSystem) ...
        
        // Initialize AISystem AFTER all its dependencies
        AI = new AISystem(Entities, Events, BiologicalSystem, Perception, Combat, Transforms); // Pass dependencies
        GD.Print("  - AISystem initialized.");

        // ... (existing system initializations after AISystem) ...
    }
}
```

### 6. Testing Utility AI

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output for the "Testing AI System" section and subsequent `AISystem: NPC X chose action...` messages.

**Expected Output:**

*   **Initial State**: The NPC (`testNpcID`) is spawned near the player.
*   **Attack Behavior**: Since the player is detected and within combat range, the NPC should repeatedly choose `AttackPlayer` (every 0.5 seconds, `AI_DECISION_INTERVAL`).
    *   `AISystem: NPC EntityID(1, Gen:1) chose action 'AttackPlayer' with score X.XX.`
    *   `AISystem: NPC EntityID(1, Gen:1) is attacking player EntityID(0, Gen:1)!`
*   **Hunger/Thirst (if low)**: If you manually reduce the NPC's hunger/thirst (e.g., `/hunger -50` on `testNpcID` using debug console - this command needs to be added to `DebugCommandSystem`), the NPC might temporarily prioritize `EatFood` or `DrinkWater` over attacking, if its hunger/thirst score becomes higher than the attack score.
*   **Flee (if low HP)**: If you damage the NPC (e.g., `/damage 40` on `testNpcID`), its health will drop. If it falls below its `FleeThreshold`, it might switch to `Flee` action.
*   **Wander (if no other actions)**: If the player moves far away (out of detection range) and the NPC has no pressing needs, it will default to `Wander`.

This confirms our `AISystem` is functional, using Utility AI to make dynamic decisions based on the NPC's `AIComponent` and current `CoreStats` / `Perception`.

### Summary

You have successfully implemented **Decision Making** for NPCs using a **Utility AI** (scoring) system in the C# Brain. By designing `AIComponent` to hold AI parameters and `UtilityAction` to represent scoreable actions, and creating `AISystem` to evaluate and execute the highest-scoring action, you've established a flexible and emergent AI framework. This crucial system strictly adheres to TDD 05.3's specifications, allowing NPCs to dynamically respond to their needs and environmental context in Sigilborne's living world.

### Next Steps

The next chapter will focus on **Ecology Simulation**, where we will design how animals and NPCs exist and behave even when the player is far away, by using "Virtual Agents" that are simulated at a very slow tick rate in unloaded chunks.