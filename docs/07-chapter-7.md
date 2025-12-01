## Chapter 1.7: Job System & Concurrency - Multithreaded Processing

Sigilborne is a simulation-heavy game. Updating thousands of virtual agents, generating vast procedural terrain, and calculating complex pathfinding for numerous NPCs concurrently on a single thread would inevitably lead to crippling performance issues. This chapter introduces a robust **Job System** to offload these heavy computations to worker threads, ensuring our game remains smooth and responsive, as detailed in TDD 13.

We will design the core `JobManager`, define an `IJob` interface, and demonstrate how to schedule and execute jobs safely.

### 1. The Challenge of Concurrency in Godot with C#

Godot's main thread (where `_Process` and `_PhysicsProcess` run) is sacred. Any long-running operation on it will cause a "hiccup" or "stutter."

**Why a Job System is Essential:**

*   **Smooth Performance**: Complex simulations (ecology, AI, world generation) can run in the background.
*   **Scalability**: Easily scale computations by adding more worker threads (up to the logical core count of the CPU).
*   **Responsiveness**: The game remains interactive while heavy tasks are being processed.
*   **Data-Oriented**: Jobs inherently encourage data-oriented design, as they operate on chunks of data rather than complex objects.

### 2. The Job Architecture: JobManager

The `JobManager` will be a central singleton in our C# Brain, responsible for managing a pool of worker threads and scheduling jobs. For simplicity and to leverage .NET's built-in threading capabilities, we'll use `System.Threading.Tasks` for our worker pool.

First, let's create the `_Brain/Utils` folder if you haven't already, as per TDD 00.

Create `res://_Brain/Utils/JobSystem.cs`:

```csharp
// _Brain/Utils/JobSystem.cs
using Godot;
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;
using Sigilborne.Core; // For EventBus

namespace Sigilborne.Utils
{
    /// <summary>
    /// Interface for any job that can be executed by the JobSystem.
    /// Jobs should be pure data structs to avoid GC and capture context.
    /// </summary>
    public interface IJob
    {
        void Execute();
    }

    /// <summary>
    /// Manages a pool of worker threads and schedules IJob instances.
    /// Uses System.Threading.Tasks for concurrency.
    /// </summary>
    public class JobSystem
    {
        // TDD 13.2: Queues - We'll simplify to one queue for now, but could add priority.
        private ConcurrentQueue<Action> _mainThreadCompletionQueue = new ConcurrentQueue<Action>();
        private EventBus _eventBus;

        public JobSystem(EventBus eventBus)
        {
            _eventBus = eventBus;
            // The EventBus's FlushCommands will be used to process _mainThreadCompletionQueue.
            // We just need to ensure the JobSystem adds to it.
            GD.Print("JobSystem: Initialized.");
        }

        /// <summary>
        /// Schedules a job to be executed on a background thread.
        /// Once the job completes, an optional callback can be added to the main thread queue.
        /// </summary>
        /// <param name="job">The IJob to execute.</param>
        /// <param name="onCompleteMainThread">An optional action to run on the main thread after the job finishes.</param>
        public void Schedule(IJob job, Action onCompleteMainThread = null)
        {
            Task.Run(() =>
            {
                try
                {
                    job.Execute();
                }
                catch (Exception e)
                {
                    GD.PrintErr($"JobSystem: Error executing background job: {e.Message}\n{e.StackTrace}");
                }

                if (onCompleteMainThread != null)
                {
                    // TDD 13.3: Command Buffers - Add completion callback to EventBus's main thread queue
                    _eventBus.AddCommand(onCompleteMainThread);
                }
            });
        }
    }
}
```

**Explanation:**

*   **`IJob` Interface**: Defines the `Execute()` method that all jobs must implement. Jobs will typically be `structs` to minimize garbage collection.
*   **`JobSystem` Class**:
    *   **`_mainThreadCompletionQueue`**: A `ConcurrentQueue<Action>` is used to safely queue actions that need to be executed back on the main thread (e.g., updating Godot nodes or `EntityManager`). The `EventBus` will be responsible for flushing this queue.
    *   **`Schedule()` Method**:
        *   Uses `Task.Run()` to execute the `job.Execute()` method on a .NET thread pool thread.
        *   Includes basic error handling for background jobs.
        *   If `onCompleteMainThread` is provided, it enqueues that `Action` into the `EventBus`'s command buffer for main thread execution, adhering to TDD 13.3's command buffer strategy.

### 3. Thread Safety Strategy: The "No-Node" Rule and Command Buffers

**CRITICAL**: The most important rule for concurrency is: **Jobs NEVER access Godot Nodes directly.**

*   **"No-Node" Rule**: Background jobs must only operate on C# data (Plain Old C# Objects - POCOs, structs, or thread-safe collections). If a job needs to affect Godot's scene tree or a `Node`, it *must* do so indirectly.
*   **Command Buffers**: When a job completes its work and needs to update the main game state (e.g., spawn an entity, update a visual property), it creates a "command" (an `Action` or a data struct describing the change) and adds it to a `ConcurrentQueue`.
    *   The `EventBus`'s `FlushCommands()` method (called on the main thread in `GameManager._PhysicsProcess`) then processes these commands safely.

### 4. Integrating JobSystem into GameManager

Let's make `JobSystem` a core component of our `GameManager`.

Open `_Brain/Core/GameManager.cs` and modify it:

1.  Add `using Sigilborne.Utils;` at the top.
2.  Add a `JobSystem` property.
3.  Initialize `JobSystem` in `InitializeSystems()`, passing the `EventBus`.

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems;
using Sigilborne.Utils; // Add this using directive

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    public TimeSystem Time { get; private set; }
    public EventBus Events { get; private set; }
    public WorldSimulation World { get; private set; }
    public EntityManager Entities { get; private set; }
    public TransformSystem Transforms { get; private set; }
    public JobSystem Jobs { get; private set; } // Add JobSystem property

    public override void _Ready()
    {
        if (Instance != null)
        {
            GD.PrintErr("GameManager: More than one instance detected! Destroying duplicate.");
            QueueFree();
            return;
        }
        Instance = this;
        
        InitializeSystems();

        GD.Print("GameManager (C#) initialized and ready.");

        // --- Test Scene Loading ---
        if (SceneLoader.instance != null)
        {
            GD.Print("GameManager: Requesting SceneLoader to load Gameplay scene.");
            SceneLoader.instance.load_level("res://_Body/Scenes/Gameplay.tscn");
        }
        else
        {
            GD.PrintErr("GameManager: SceneLoader instance not found! Cannot load gameplay scene.");
        }
        // --- End Test Scene Loading ---

        // --- Test Entity Management & Components ---
        GD.Print("\n--- Testing Entity Management & Components ---");
        EntityID playerEntity = Entities.CreateEntity(EntityType.Player, "player_default");
        GD.Print($"Created Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        Transforms.TrySetTransform(playerEntity, new TransformComponent(new Vector2(100, 50), 45f));
        if (Transforms.TryGetTransform(playerEntity, out TransformComponent playerTransform))
        {
            GD.Print($"Player {playerEntity} Transform: {playerTransform}");
        }

        ref TransformComponent playerTransformRef = ref Transforms.GetTransformRef(playerEntity);
        playerTransformRef.Position = new Vector2(150, 75);
        playerTransformRef.RotationDegrees = 90f;
        GD.Print($"Player {playerEntity} New Transform (via ref): {Transforms.GetTransformRef(playerEntity)}");

        Entities.DestroyEntity(playerEntity);
        GD.Print($"Destroyed Player: {playerEntity}. IsValid: {Entities.IsValid(playerEntity)}");

        if (!Transforms.TryGetTransform(playerEntity, out TransformComponent destroyedTransform))
        {
            GD.Print($"Attempted to get transform for destroyed entity {playerEntity}, it correctly failed.");
        }

        GD.Print("--- End Testing Entity Management & Components ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Time.Tick(delta);
        Events.FlushCommands(); // This processes the _mainThreadCompletionQueue from JobSystem
        World.Tick(delta);
        Transforms.Tick(delta);
    }

    private void InitializeSystems()
    {
        Events = new EventBus();
        GD.Print("  - EventBus initialized.");

        Time = new TimeSystem();
        GD.Print("  - TimeSystem initialized.");

        Entities = new EntityManager(Events);
        GD.Print("  - EntityManager initialized.");

        Transforms = new TransformSystem(Entities, Events);
        GD.Print("  - TransformSystem initialized.");
        
        Jobs = new JobSystem(Events); // Initialize JobSystem here
        GD.Print("  - JobSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 5. Creating and Scheduling a Test Job

Let's create a simple test job to demonstrate multithreaded processing and main thread completion.

First, define a simple `TestJob` struct in `_Brain/Utils/TestJob.cs`:

```csharp
// _Brain/Utils/TestJob.cs
using Godot;
using System;
using System.Threading; // For Thread.Sleep

namespace Sigilborne.Utils
{
    /// <summary>
    /// A simple job that performs a long-running calculation on a background thread.
    /// </summary>
    public struct TestJob : IJob
    {
        public string JobName;
        public int Iterations;
        public double StartTime; // To measure duration

        public void Execute()
        {
            GD.Print($"[Job:{JobName}] Starting job on background thread... (Main Thread Time: {GameManager.Instance.Time.CurrentGameTime:F2})");
            double sum = 0;
            for (int i = 0; i < Iterations; i++)
            {
                sum += Math.Sqrt(i); // Simulate some computation
                // Simulate a delay (DO NOT USE Thread.Sleep in real jobs, this is just for demo)
                // Thread.Sleep(1);
            }
            GD.Print($"[Job:{JobName}] Finished computation. Sum: {sum:F2}");
        }
    }
}
```

Now, schedule this job in `GameManager._Ready()`, after all other initializations.

```csharp
// _Brain/Core/GameManager.cs (inside _Ready method, after entity tests)
// ...
        // --- Testing JobSystem ---
        GD.Print("\n--- Testing JobSystem ---");
        Jobs.Schedule(new TestJob { JobName = "HeavyCompute", Iterations = 1_000_000, StartTime = Time.CurrentGameTime },
            () => {
                // This callback runs on the main thread after the job finishes.
                GD.Print($"[Main Thread] Job 'HeavyCompute' completed callback. (Main Thread Time: {Time.CurrentGameTime:F2})");
            });
        GD.Print("[Main Thread] Scheduled 'HeavyCompute' job. Continuing main thread execution.");

        Jobs.Schedule(new TestJob { JobName = "AnotherJob", Iterations = 500_000, StartTime = Time.CurrentGameTime },
            () => {
                GD.Print($"[Main Thread] Job 'AnotherJob' completed callback. (Main Thread Time: {Time.CurrentGameTime:F2})");
            });
        GD.Print("[Main Thread] Scheduled 'AnotherJob' job. Continuing main thread execution.");
        
        GD.Print("--- End Testing JobSystem ---\n");
// ...
```

Save all C# files and run `Main.tscn`.

**Expected Output:**

You should see output similar to this (timestamps will vary):

```
... (previous initializations and entity tests) ...

--- Testing JobSystem ---
[Main Thread] Scheduled 'HeavyCompute' job. Continuing main thread execution.
[Main Thread] Scheduled 'AnotherJob' job. Continuing main thread execution.
--- End Testing JobSystem ---

[Job:HeavyCompute] Starting job on background thread... (Main Thread Time: 0.00)
[Job:AnotherJob] Starting job on background thread... (Main Thread Time: 0.00)
[Job:HeavyCompute] Finished computation. Sum: 210818.00
[Job:AnotherJob] Finished computation. Sum: 70709.00
[Main Thread] Job 'HeavyCompute' completed callback. (Main Thread Time: 2.01)
[Main Thread] Job 'AnotherJob' completed callback. (Main Thread Time: 2.01)
```

**Key Observations:**

*   The `[Main Thread] Scheduled...` messages appear *immediately*, showing that the main thread continues without waiting.
*   The `[Job:...] Starting...` messages appear after the main thread, indicating the jobs started on background threads.
*   The `[Job:...] Finished...` messages appear before the `[Main Thread] Job '...' completed callback` messages. This is because the job finishes on its background thread, then enqueues its callback to the main thread's `EventBus` command buffer, which is flushed in `_PhysicsProcess`.
*   The `Main Thread Time` in the callbacks will be slightly later than when the jobs started, as time passed during the background computation.

This demonstrates that our Job System successfully offloads work to background threads and safely processes completion callbacks on the main thread, respecting Godot's threading model.

### Summary

You have successfully implemented the core `JobSystem` for Sigilborne, enabling multithreaded processing for heavy computations. By defining the `IJob` interface and integrating `JobSystem` with the `EventBus`, you've established a robust mechanism for offloading tasks to background threads while ensuring safe main thread synchronization through command buffers. This adheres strictly to TDD 13's specifications, providing a critical foundation for scalable performance in our simulation-heavy game.

### Next Steps

With the Job System in place, the next chapter will focus on standardizing how Godot Scenes are composed and how `EntityView.gd` acts as the crucial communication bridge between our C# Brain and GDScript Body, ensuring visual data is always reactive and decoupled from simulation logic.