## Chapter 1.2: The Brain (C#) - Headless Simulation Layer

In Sigilborne, the "Brain" is our C# simulation layer, operating largely independently of Godot's visual nodes. Its primary responsibilities are heavy simulation, managing game data, driving AI decisions, and handling all procedural generation. This headless approach is crucial for performance, determinism, and future scalability, as it allows the core logic to run efficiently without being tied to the rendering pipeline.

This chapter will focus on implementing the `GameManager` class in C#. As per TDD 01, `GameManager` acts as the single entry point for all C# logic, a central singleton, and the bootstrapper for our core systems.

### 1. Understanding the Role of the Brain (C#)

The C# layer embodies the "Brain" of our game. It's where all the fundamental rules, calculations, and state management occur. Think of it as a local server running your game's universe.

**Key Characteristics of the Brain:**

*   **Headless**: Ideally, it should not directly interact with Godot's visual nodes (`Sprite2D`, `Node2D`, `Control`, etc.). Its world is one of pure data: structs, classes, and arrays.
*   **Authoritative**: All game state (player health, NPC positions, inventory, world time, faction relationships) is owned and managed by the Brain.
*   **Deterministic**: For a simulation-heavy game, especially with potential future multiplayer, the Brain's logic should be deterministic, meaning the same inputs always produce the same outputs.
*   **Performance-Oriented**: C# allows for efficient data structures (structs, `Span<T>`, `NativeArray` equivalents via `Unsafe` code if needed) and multithreading (via the Job System, TDD 13) to handle complex simulations without bogging down the main thread.

### 2. Implementing the GameManager Singleton

The `GameManager` is the heart of our C# architecture. It will be a `Node` in the Godot scene tree, but its primary purpose is to bootstrap and manage other C# systems, not to perform visual tasks. It adheres to the Singleton pattern, providing global access to core systems.

Open `_Brain/Core/GameManager.cs` in your IDE and modify it as follows:

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core; // Assuming Core namespace for TimeSystem, EventBus

public partial class GameManager : Node
{
    // Static instance for global access (Singleton pattern)
    public static GameManager Instance { get; private set; }
    
    // --- Core Systems ---
    // These will be initialized here and accessible globally.
    public TimeSystem Time { get; private set; }
    public EventBus Events { get; private set; }
    public WorldSimulation World { get; private set; } // Represents the main world simulation logic

    // --- Godot Lifecycle Methods ---
    public override void _Ready()
    {
        // Ensure only one instance of GameManager exists
        if (Instance != null)
        {
            GD.PrintErr("GameManager: More than one instance detected! Destroying duplicate.");
            QueueFree(); // Remove this duplicate node
            return;
        }
        Instance = this;
        
        // Initialize all core C# systems
        InitializeSystems();

        GD.Print("GameManager (C#) initialized and ready.");
    }

    // _PhysicsProcess runs on a fixed timestep, making it suitable for our simulation tick.
    public override void _PhysicsProcess(double delta)
    {
        // The Heartbeat of the Brain: Advance the world simulation
        // The TDD specifies World.Tick(delta), which we'll implement in TDD 01.3
        // For now, let's ensure our core systems get their update calls.
        Time.Tick(delta); // Update game time
        Events.FlushCommands(); // Process any batched events from background jobs
        World.Tick(delta); // Main world simulation update
    }

    /// <summary>
    /// Initializes all core C# systems managed by the GameManager.
    /// </summary>
    private void InitializeSystems()
    {
        // 1. Initialize EventBus first, as other systems may need to register/emit events.
        Events = new EventBus();
        GD.Print("  - EventBus initialized.");

        // 2. Initialize TimeSystem (manages game time, day/night cycle, etc.)
        Time = new TimeSystem();
        GD.Print("  - TimeSystem initialized.");

        // 3. Initialize the main WorldSimulation (this will encompass other systems like Ecology, AI, etc.)
        // We'll flesh out WorldSimulation in a later chapter.
        World = new WorldSimulation(); // Placeholder for now
        GD.Print("  - WorldSimulation initialized.");

        // Add more system initializations here as we build them
        // Example: CombatSystem, EcologySystem, FactionSystem, etc.
        // Combat = new CombatSystem();
        // Ecology = new EcologySystem();
    }
}
```

**Explanation of Changes:**

*   **`Instance` Property**: This static property implements the Singleton pattern, allowing any other C# class to access core systems via `GameManager.Instance.Time` or `GameManager.Instance.Events`.
*   **System Properties**: `Time`, `Events`, and `World` are declared as properties, making them accessible after initialization.
*   **`_Ready()` Method**:
    *   Contains a check to ensure only one `GameManager` instance exists, preventing potential bugs from duplicate nodes.
    *   Calls `InitializeSystems()` to set up all our C# backend logic.
*   **`_PhysicsProcess(double delta)`**:
    *   As per TDD 01, this Godot method, which runs on a fixed timestep, is designated as the "Heartbeat of the Brain."
    *   Here, we call `Tick()` or `FlushCommands()` on our core systems, ensuring they update in a consistent order.
*   **`InitializeSystems()` Method**: A private method responsible for creating instances of our various C# systems. This keeps `_Ready()` clean and organized.

### 3. Creating Placeholder Core Systems

The `GameManager` now expects `TimeSystem`, `EventBus`, and `WorldSimulation` to exist. Let's create minimal placeholder classes for these in the `_Brain/Core` directory.

#### 3.1 `EventBus.cs`

This will be our central hub for C# to C# and C# to GDScript communication.

Create a new C# script `_Brain/Core/EventBus.cs`:

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent; // For ConcurrentQueue
using Godot; // For GD.PrintErr in error handling

namespace Sigilborne.Core
{
    // This namespace helps organize our C# code as per TDD 00
    // and prevents naming conflicts.
    public class EventBus
    {
        // TDD 01 specifies Action<T> delegates for C# to GDScript (and C# to C#)
        // For simplicity in this early stage, we'll use a generic Action for C# to C#
        // and define specific signals for C# to GDScript later.

        // Example: A generic event for when a game state changes
        public event Action<string> OnGameStateChanged;

        // TDD 01 also mentions batching events from background jobs.
        // This ConcurrentQueue will hold commands to be processed on the main thread.
        private ConcurrentQueue<Action> _commandBuffer = new ConcurrentQueue<Action>();

        /// <summary>
        /// Publishes an event for C# subscribers.
        /// </summary>
        /// <typeparam name="TEvent">The type of the event.</typeparam>
        /// <param name="eventData">The event data.</param>
        public void Publish<TEvent>(TEvent eventData)
        {
            // This is a simplified generic publish.
            // In a real system, you'd have a dictionary of event types to their delegates.
            // For now, we'll rely on specific named events (like OnGameStateChanged).
            if (eventData is string gameState)
            {
                OnGameStateChanged?.Invoke(gameState);
            }
            // Add more specific event types here as needed.
        }

        /// <summary>
        /// Adds a command to be executed on the main thread during the next flush.
        /// Used by background jobs.
        /// </summary>
        /// <param name="command">The action to execute.</param>
        public void AddCommand(Action command)
        {
            _commandBuffer.Enqueue(command);
        }

        /// <summary>
        /// Flushes all commands from the buffer, executing them on the main thread.
        /// Called once per main thread tick (e.g., in _PhysicsProcess).
        /// </summary>
        public void FlushCommands()
        {
            while (_commandBuffer.TryDequeue(out var command))
            {
                try
                {
                    command.Invoke();
                }
                catch (Exception e)
                {
                    GD.PrintErr($"EventBus: Error executing batched command: {e.Message}\n{e.StackTrace}");
                }
            }
        }
    }
}
```

**Note on Namespaces**: We've introduced the `Sigilborne.Core` namespace. This is a good practice in C# to organize code and prevent naming collisions, especially in larger projects. Remember to add `using Sigilborne.Core;` at the top of any C# file that needs to access classes from this namespace.

#### 3.2 `TimeSystem.cs`

This system will manage the in-game clock, day/night cycle, and other time-related logic.

Create a new C# script `_Brain/Core/TimeSystem.cs`:

```csharp
// _Brain/Core/TimeSystem.cs
using Godot;
using System;

namespace Sigilborne.Core
{
    public class TimeSystem
    {
        public double CurrentGameTime { get; private set; } // Total game time in seconds
        public int CurrentDay { get; private set; }
        public int CurrentHour { get; private set; }
        public int CurrentMinute { get; private set; }

        private const float REAL_SECONDS_PER_GAME_MINUTE = 1.0f; // 1 real second = 1 game minute
        private const float GAME_MINUTES_PER_HOUR = 60.0f;
        private const float GAME_HOURS_PER_DAY = 24.0f;

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
            // Convert real delta time to game minutes
            double gameMinutesToAdd = delta / REAL_SECONDS_PER_GAME_MINUTE;
            CurrentGameTime += gameMinutesToAdd;

            // Update game clock
            CurrentMinute += (int)Math.Floor(gameMinutesToAdd);
            if (CurrentMinute >= GAME_MINUTES_PER_HOUR)
            {
                CurrentHour += CurrentMinute / (int)GAME_MINUTES_PER_HOUR;
                CurrentMinute %= (int)GAME_MINUTES_PER_HOUR;

                if (CurrentHour >= GAME_HOURS_PER_DAY)
                {
                    CurrentDay += CurrentHour / (int)GAME_HOURS_PER_DAY;
                    CurrentHour %= (int)GAME_HOURS_PER_DAY;
                    // GameManager.Instance.Events.Publish("NewDay", CurrentDay); // Example event
                }
                // GameManager.Instance.Events.Publish("NewHour", CurrentHour); // Example event
            }
        }
    }
}
```

#### 3.3 `WorldSimulation.cs`

This will be the parent system for all other simulation systems (Ecology, Combat, Economy, etc.).

Create a new C# script `_Brain/Core/WorldSimulation.cs`:

```csharp
// _Brain/Core/WorldSimulation.cs
using Godot;
using System;

namespace Sigilborne.Core
{
    public class WorldSimulation
    {
        // Placeholder for other systems
        // public EcologySystem Ecology { get; private set; }
        // public CombatSystem Combat { get; private set; }

        public WorldSimulation()
        {
            // Initialize child systems here
            // Ecology = new EcologySystem();
            // Combat = new CombatSystem();
            GD.Print("WorldSimulation: Initialized.");
        }

        public void Tick(double delta)
        {
            // Update child systems here
            // Ecology.Tick(delta);
            // Combat.Tick(delta);
            // GD.Print($"WorldSimulation: Tick at {GameManager.Instance.Time.CurrentGameTime:F2}"); // Example
        }
    }
}
```

### 4. Testing the Initial Brain Setup

Now, let's run the game again to ensure our `GameManager` correctly initializes and ticks these systems.

1.  Save all your C# files.
2.  Go back to the Godot editor. It might prompt you to re-import C# projects. Allow it.
3.  Ensure `Main.tscn` is still set as your main scene (Project Settings -> Application -> Run).
4.  Click the "Play Scene" button.

In the Output console, you should now see:

```
GameManager (C#) initialized and ready.
  - EventBus initialized.
  - TimeSystem initialized. Day 1, 08:00
  - WorldSimulation initialized.
```

And as the game runs, you'll see repeated output from `TimeSystem` (if you uncommented the `GD.Print` in `TimeSystem.Tick`). This confirms that our `GameManager` is acting as the central orchestrator for our C# simulation.

### Summary

You have successfully implemented the `GameManager` as the entry point for the C# Brain, establishing it as a global singleton and the bootstrapper for core systems like `EventBus`, `TimeSystem`, and `WorldSimulation`. This lays a robust foundation for our hybrid architecture, ensuring that all complex simulation logic is managed in C# on a fixed timestep, ready to be decoupled from Godot's visual layer.

### Next Steps

In the next chapter, we will implement the "Body" (GDScript) presentation layer and begin to define how it reacts to the state provided by the Brain.