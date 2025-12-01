## Chapter 8.2: The Simulation Clock - Game Time & Load Balancing (C#)

Sigilborne's world is a vast simulation, where factions wage war, economies fluctuate, and NPCs live out their lives over in-game days, weeks, and years. Simulating all these complex systems in real-time at 60 frames per second is impractical. This chapter focuses on implementing **The Simulation Clock**, which defines how the world's internal "Game Time" operates at a slower, configurable tick rate than the rendering frame. It also details how heavy system updates (like Faction AI planning) are **load-balanced** across this daily cycle to prevent performance spikes, as specified in TDD 06.2.3 and TDD 06.2.4.

### 1. The Need for a Simulation Clock and Load Balancing

*   **Performance**: Many game systems (AI, economy, ecology) don't need to update every frame. A slower "game tick" reduces computational load.
*   **Realism**: Political and ecological changes happen over longer timescales than combat.
*   **Load Balancing**: Even on a slower game tick, some operations (e.g., a Faction AI planning its day) can be CPU-intensive. Spreading these out prevents noticeable hitches.
*   **Determinism**: The game clock provides a consistent time reference for all simulation logic.

### 2. Enhancing `TimeSystem.cs` for Game Time

Our `TimeSystem` already tracks `CurrentGameTime`, `CurrentDay`, `CurrentHour`, and `CurrentMinute`. We'll enhance it with events for day/hour changes and a mechanism for systems to subscribe to these.

Open `res://_Brain/Core/TimeSystem.cs`:

```csharp
// _Brain/Core/TimeSystem.cs
using Godot;
using System;

namespace Sigilborne.Core
{
    /// <summary>
    /// Manages the in-game clock, day/night cycle, and other time-related logic.
    /// (TDD 01.2, TDD 06.2.3)
    /// </summary>
    public class TimeSystem
    {
        public double CurrentGameTime { get; private set; } // Total game time in seconds (real-time equivalent)
        public int CurrentDay { get; private set; }
        public int CurrentHour { get; private set; }
        public int CurrentMinute { get; private set; }

        // TDD 06.2.3: Tick Rate - 1 Tick = 1 In-Game Minute (approx 1 real second).
        private const float REAL_SECONDS_PER_GAME_MINUTE = 1.0f; // 1 real second = 1 game minute
        private const float GAME_MINUTES_PER_HOUR = 60.0f;
        private const float GAME_HOURS_PER_DAY = 24.0f;

        // Events for other systems to subscribe to (TDD 06.2.3: Daily Cycle)
        public event Action<int> OnNewDay; // Parameter: New day number
        public event Action<int> OnNewHour; // Parameter: New hour number

        public TimeSystem()
        {
            CurrentGameTime = 0.0;
            CurrentDay = 1;
            CurrentHour = 8; // Start at 8 AM
            CurrentMinute = 0;
            GD.Print($"TimeSystem: Initialized. Day {CurrentDay}, {CurrentHour:D2}:{CurrentMinute:D2}");
        }

        public void Tick(double delta)
        {
            double gameMinutesToAdd = delta / REAL_SECONDS_PER_GAME_MINUTE;
            CurrentGameTime += gameMinutesToAdd;

            // Store old values to detect changes
            int oldHour = CurrentHour;
            int oldDay = CurrentDay;

            CurrentMinute += (int)Math.Floor(gameMinutesToAdd);
            if (CurrentMinute >= GAME_MINUTES_PER_HOUR)
            {
                CurrentHour += CurrentMinute / (int)GAME_MINUTES_PER_HOUR;
                CurrentMinute %= (int)GAME_MINUTES_PER_HOUR;

                if (CurrentHour >= GAME_HOURS_PER_DAY)
                {
                    CurrentDay += CurrentHour / (int)GAME_HOURS_PER_DAY;
                    CurrentHour %= (int)GAME_HOURS_PER_DAY;
                }
            }

            // Publish events if hour or day changed
            if (CurrentHour != oldHour)
            {
                OnNewHour?.Invoke(CurrentHour);
                // GD.Print($"TimeSystem: New Hour: {CurrentHour:D2}:00");
            }
            if (CurrentDay != oldDay)
            {
                OnNewDay?.Invoke(CurrentDay);
                GD.Print($"TimeSystem: New Day: {CurrentDay}");
            }
        }

        /// <summary>
        /// Sets the current in-game hour.
        /// </summary>
        /// <param name="hour">The hour to set (0-23).</param>
        public void SetCurrentHour(int hour)
        {
            if (hour < 0 || hour > 23)
            {
                GD.PrintErr($"TimeSystem: Invalid hour '{hour}'. Must be between 0 and 23.");
                return;
            }
            CurrentHour = hour;
            CurrentMinute = 0; // Reset minutes for clean hour setting
            GD.Print($"TimeSystem: Hour manually set to {CurrentHour:D2}:00.");
            OnNewHour?.Invoke(CurrentHour); // Publish event for manual set
        }
    }
}
```

#### 2.1. Update `EventBus.cs` for `TimeSystem` Events

Open `res://_Brain/Core/EventBus.cs` and add `OnNewDay` and `OnNewHour` delegates.

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
using System.Collections.Generic;

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing events) ...

        // Time System Events (TDD 06.2.3)
        public event Action<int> OnNewDay;
        public event Action<int> OnNewHour;

        // ... (existing _commandBuffer) ...

        public void Publish<TEvent>(TEvent eventData)
        {
            // ... (existing publish conditions) ...
            else if (eventData is int newDay && typeof(TEvent) == typeof(int) && CurrentEventType == TimeEventType.NewDay) // Special handling for int events
            {
                OnNewDay?.Invoke(newDay);
            }
            else if (eventData is int newHour && typeof(TEvent) == typeof(int) && CurrentEventType == TimeEventType.NewHour) // Special handling for int events
            {
                OnNewHour?.Invoke(newHour);
            }
            else
            {
                GD.PrintErr($"EventBus: Attempted to publish unknown event type: {typeof(TEvent).Name}. Ensure it's handled in Publish<TEvent>.");
            }
        }
        // Special enum to differentiate int events (can't overload Action<int> easily)
        public enum TimeEventType { None, NewDay, NewHour }
        public TimeEventType CurrentEventType { get; set; } // Set by TimeSystem before publishing int.
        // This is a workaround for C# Action<int> not having distinct types.
    }
}
```

#### 2.2. Modify `TimeSystem.cs` to use `EventBus.CurrentEventType`

```csharp
// _Brain/Core/TimeSystem.cs (inside Tick method)
// ...
            if (CurrentHour != oldHour)
            {
                _eventBus.CurrentEventType = EventBus.TimeEventType.NewHour; // Set type before publishing
                _eventBus.Publish(CurrentHour); // Publish the int directly
                // GD.Print($"TimeSystem: New Hour: {CurrentHour:D2}:00");
            }
            if (CurrentDay != oldDay)
            {
                _eventBus.CurrentEventType = EventBus.TimeEventType.NewDay; // Set type before publishing
                _eventBus.Publish(CurrentDay); // Publish the int directly
                GD.Print($"TimeSystem: New Day: {CurrentDay}");
            }
            _eventBus.CurrentEventType = EventBus.TimeEventType.None; // Reset
// ...
```

### 3. Implementing `FactionAISystem.cs` for Daily Planning & Load Balancing

This system will simulate Faction AI decisions once per game day, and importantly, load-balance its CPU-intensive operations.

1.  Create `res://_Brain/Systems/Factions/FactionAISystem.cs`:

```csharp
// _Brain/Systems/Factions/FactionAISystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Magic; // For GlyphConcept
using Sigilborne.Utils; // For JobSystem
using System.Linq;

namespace Sigilborne.Systems.Factions
{
    /// <summary>
    /// Manages AI decision-making for major factions.
    /// Runs once per game day and load-balances heavy operations.
    /// (TDD 06.2.2: Faction AI, TDD 06.2.4: Load Balancing)
    /// </summary>
    public class FactionAISystem
    {
        private EventBus _eventBus;
        private FactionSystem _factionSystem; // To query factions and relationships
        private JobSystem _jobSystem;         // For load balancing heavy tasks

        private Random _rand; // For deterministic random decisions

        private const int FACTION_AI_PLANNING_HOUR = 6; // TDD 06.2.3: Faction AI plans at 06:00
        private bool _hasPlannedToday = false;

        public FactionAISystem(EventBus eventBus, FactionSystem factionSystem, JobSystem jobSystem, int worldSeed)
        {
            _eventBus = eventBus;
            _factionSystem = factionSystem;
            _jobSystem = jobSystem;
            _rand = new Random(worldSeed);

            _eventBus.OnNewHour += OnNewHour; // Subscribe to hourly updates
            _eventBus.OnNewDay += OnNewDay; // Subscribe to daily updates

            GD.Print("FactionAISystem: Initialized.");
        }

        private void OnNewHour(int hour)
        {
            if (hour == FACTION_AI_PLANNING_HOUR && !_hasPlannedToday)
            {
                GD.Print($"FactionAISystem: It's {FACTION_AI_PLANNING_HOUR}:00. Initiating daily faction planning.");
                _hasPlannedToday = true;
                PlanDailyFactionMoves();
            }
        }

        private void OnNewDay(int day)
        {
            _hasPlannedToday = false; // Reset for the new day
        }

        /// <summary>
        /// Initiates the daily planning for all factions, load-balancing heavy computations.
        /// (TDD 06.2.2: Faction AI Runs once per "Game Day", TDD 06.2.4: Load Balancing)
        /// </summary>
        private void PlanDailyFactionMoves()
        {
            var allFactions = _factionSystem.GetAllFactions();
            int factionsPerJob = 2; // Process 2 factions per job for load balancing

            for (int i = 0; i < allFactions.Count; i += factionsPerJob)
            {
                List<Faction> factionsToProcess = allFactions.Skip(i).Take(factionsPerJob).ToList();
                
                // Schedule a job for each batch of factions
                _jobSystem.Schedule(new FactionPlanningJob(factionsToProcess, _factionSystem, _rand.Next()), () => {
                    // This callback runs on the main thread after each job completes.
                    GD.Print($"FactionAISystem: Completed planning job for {factionsToProcess.Count} factions.");
                    // After all jobs are done, we might trigger a global event.
                });
            }
            GD.Print($"FactionAISystem: Scheduled {MathF.Ceiling((float)allFactions.Count / factionsPerJob)} faction planning jobs.");
        }

        /// <summary>
        /// Represents a background job for a batch of factions to plan their daily moves.
        /// (TDD 13.2: The Job Struct)
        /// </summary>
        public struct FactionPlanningJob : IJob
        {
            public List<Faction> FactionsToPlan;
            public FactionSystem FactionSystem; // Reference to the FactionSystem for updates
            public int Seed; // For deterministic random in job

            public FactionPlanningJob(List<Faction> factions, FactionSystem factionSystem, int seed)
            {
                FactionsToPlan = factions;
                FactionSystem = factionSystem;
                Seed = seed;
            }

            public void Execute()
            {
                Random jobRand = new Random(Seed);
                foreach (Faction faction in FactionsToPlan)
                {
                    // --- Faction AI Logic (TDD 06.2.2) ---
                    // Example: Check relationships and decide to declare war, send caravan, etc.
                    GD.Print($"[Job] Faction '{faction.Name}' is planning. Type: {faction.Type}");

                    // Example: Adjust relationships based on ideology
                    foreach (Faction otherFaction in FactionSystem.GetAllFactions())
                    {
                        if (faction.ID == otherFaction.ID) continue;

                        float currentRelationValue = FactionSystem.GetRelationship(faction.ID, otherFaction.ID).Value;
                        float ideologicalCompatibility = CalculateIdeologicalCompatibility(faction, otherFaction);

                        float baseChange = 0;
                        if (ideologicalCompatibility > 0.7f) baseChange = 1f; // Slowly drift positive
                        else if (ideologicalCompatibility < 0.3f) baseChange = -1f; // Slowly drift negative

                        // Add some random daily fluctuation
                        baseChange += (float)(jobRand.NextDouble() * 2 - 1); // +/- 1

                        // Only adjust if not already in extreme states
                        if ((currentRelationValue > -90 && currentRelationValue < 90))
                        {
                            // TDD 13.3: Command Buffers - Use main thread for actual state changes.
                            // The FactionSystem.AdjustRelationship method already publishes events which
                            // are then picked up by the EventBus's flush.
                            FactionSystem.AdjustRelationship(faction.ID, otherFaction.ID, baseChange);
                        }
                    }

                    // Example: Decide to build outpost (conceptual)
                    if (faction.Type == FactionType.WarriorDominion && jobRand.NextDouble() < 0.1)
                    {
                        GD.Print($"[Job] Faction '{faction.Name}' decided to build an outpost (conceptual).");
                        // _eventBus.Publish(new FactionAction.BuildOutpostEvent { FactionID = faction.ID, Location = ... });
                    }
                }
            }

            /// <summary>
            /// Calculates a compatibility score between two factions based on their primary and taboo glyph concepts.
            /// (GDD B17.7: Faction Identity & Category Alignment)
            /// </summary>
            private float CalculateIdeologicalCompatibility(Faction factionA, Faction factionB)
            {
                float score = 0;
                // Bonus for shared primary concepts
                score += factionA.PrimaryConcepts.Intersect(factionB.PrimaryConcepts).Count() * 0.2f;
                // Penalty for one's primary being another's taboo
                score -= factionA.PrimaryConcepts.Intersect(factionB.TabooConcepts).Count() * 0.5f;
                score -= factionB.PrimaryConcepts.Intersect(factionA.TabooConcepts).Count() * 0.5f;
                // Normalization (simple for now)
                return Mathf.Clamp(score, -1f, 1f);
            }
        }
    }
}
```

### 5. Integrating `FactionAISystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Factions;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `FactionAISystem` property.
3.  Initialize `FactionAISystem` in `InitializeSystems()` **after** `FactionSystem` and `JobSystem`.
4.  Remove the `FactionSystem`'s default relationship setting from `InitializeSystems()` and move it to `FactionAISystem`'s constructor, ensuring `FactionAISystem` controls the initial setup.

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
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public FactionSystem Factions { get; private set; }
    public FactionAISystem FactionAI { get; private set; } // Add FactionAISystem property

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

        // --- Test Faction System ---
        GD.Print("\n--- Testing Faction System ---");
        GD.Print("All Factions:");
        foreach (var faction in Factions.GetAllFactions())
        {
            GD.Print($"  - {faction}");
        }

        // Test relationships
        GD.Print($"Relationship Crimson Blades <-> Sunken Temple: {Factions.GetRelationship("crimson_blades", "sunken_temple")}");
        GD.Print($"Relationship Crimson Blades <-> Void Cultists: {Factions.GetRelationship("crimson_blades", "void_cultists")}");
        GD.Print($"Relationship Sunken Temple <-> Shadow Veilers: {Factions.GetRelationship("sunken_temple", "shadow_veilers")}");

        // Adjust a relationship (will be done by FactionAISystem now)
        // Factions.AdjustRelationship("shadow_veilers", "sunken_temple", 30f);
        // Factions.AdjustRelationship("free_wanderers", "void_cultists", -70f);

        GD.Print("--- End Testing Faction System ---\n");
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
        FactionAI.Tick(delta); // Call FactionAISystem's tick method
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to FactionSystem) ...
        
        // FactionSystem's constructor will now call RegisterDefaultFactions
        Factions = new FactionSystem(Events); 
        GD.Print("  - FactionSystem initialized.");

        // Initialize FactionAISystem AFTER FactionSystem and JobSystem
        FactionAI = new FactionAISystem(Events, Factions, Jobs, WorldSeed); // Pass dependencies
        GD.Print("  - FactionAISystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation(Ecology);
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 6. Testing the Simulation Clock and Load Balancing

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the console output.
    *   **Time Progression**: You'll see `TimeSystem: New Hour: X` messages every real-time minute.
    *   **Faction Planning Trigger**: At `06:00` (in-game time), you should see `FactionAISystem: It's 06:00. Initiating daily faction planning.`
    *   **Job Scheduling**: Immediately after, `FactionAISystem: Scheduled X faction planning jobs.` will appear.
    *   **Load Balancing**: Then, you'll see `[Job] Faction 'X' is planning.` messages, interleaved with other system ticks. These jobs run on background threads.
    *   **Relationship Adjustments**: `FactionSystem: Relationship X adjusted by Y.` messages will appear as these background jobs complete and update relationships on the main thread via the `EventBus`.

This demonstrates that:
*   The `TimeSystem` is correctly ticking and broadcasting hourly/daily events.
*   `FactionAISystem` reacts to these events to trigger daily planning.
*   Heavy planning logic is offloaded to the `JobSystem` for load balancing.
*   Faction relationships dynamically adjust over time based on ideological compatibility and random fluctuations.

### Summary

You have successfully implemented **The Simulation Clock** by enhancing `TimeSystem` to broadcast hourly and daily events, and designed **Load Balancing** for heavy operations. By creating `FactionAISystem` to subscribe to `TimeSystem` events and offload its daily planning logic to the `JobSystem`, you've ensured that complex calculations for faction relationships and strategic moves occur without causing performance spikes. This crucial system strictly adheres to TDD 06.2.3 and TDD 06.2.4's specifications, providing a dynamic and performant backbone for Sigilborne's evolving political landscape.

### Next Steps

The next chapter will focus on **Market Simulation**, where we will define how prices for goods are local to each settlement and dynamically fluctuate based on supply, demand, and global modifiers like war or famine.