## Chapter 1.8: Thread Safety - Command Buffers & Double Buffering

In the previous chapter, we established the `JobSystem` for multithreaded processing. However, concurrency introduces complex challenges, primarily **thread safety**. Background threads cannot directly manipulate Godot's scene tree or the main simulation state without risking race conditions, crashes, or data corruption. This chapter details our strategies for safe multithreaded operations: the "No-Node" Rule, Command Buffers, and an introduction to Double Buffering, as specified in TDD 13.3.

### 1. The "No-Node" Rule: Foundation of Thread Safety

The most fundamental rule in our Brain & Body architecture for thread safety is:

> **Background Jobs (C#) MUST NEVER directly access Godot Nodes (GDScript or C#).**

*   **Why?**: Godot's API is generally not thread-safe. Accessing a `Node2D.position` or calling `Node.AddChild()` from a background thread can lead to undefined behavior, crashes, or corrupt the scene tree.
*   **What this means**: Any code within an `IJob.Execute()` method, or any code called by it, must operate purely on C# data structures (structs, POCOs, thread-safe collections).

If a job needs to affect something visual or part of the scene tree, it must do so *indirectly* via a message or command that is processed on the **main thread**.

### 2. Command Buffers: Safe Main Thread Updates

Command Buffers are our primary mechanism for background threads to request changes to the main game state or visuals.

**How it Works:**

1.  **Job (Background Thread)**: Performs its computation. When it needs to make a change that affects the main thread (e.g., spawn a visual entity, update an NPC's on-screen health bar, mark an entity as dead), it doesn't do it directly. Instead, it creates a small, serializable "command" (often an `Action` or a data struct) and adds it to a `ConcurrentQueue`.
2.  **EventBus (Main Thread)**: Our `EventBus` (`_Brain/Core/EventBus.cs`) already has a `_commandBuffer` (`ConcurrentQueue<Action>`) and a `FlushCommands()` method. This is precisely what TDD 13.3 refers to for processing batched commands.
3.  **Main Thread Execution**: During `GameManager._PhysicsProcess()`, `EventBus.FlushCommands()` is called. This method dequeues and executes all pending commands, ensuring they run safely on the main thread.

We already implemented this basic structure in Chapter 1.7. Let's ensure our `EventBus` is robust enough for this.

**Review `_Brain/Core/EventBus.cs`:**

```csharp
// _Brain/Core/EventBus.cs
using System;
using System.Collections.Concurrent;
using Godot; // For GD.PrintErr in error handling

namespace Sigilborne.Core
{
    public class EventBus
    {
        // ... (existing event Action definitions) ...

        // TDD 01.3 specifies a ConcurrentQueue for batching events from Background Jobs.
        // TDD 13.3 specifies this as the Command Buffer.
        private ConcurrentQueue<Action> _commandBuffer = new ConcurrentQueue<Action>();

        /// <summary>
        /// Adds a command to be executed on the main thread during the next flush.
        /// Used by background jobs or any code that needs to defer execution to the main thread.
        /// </summary>
        /// <param name="command">The action to execute.</param>
        public void AddCommand(Action command)
        {
            _commandBuffer.Enqueue(command);
        }

        /// <summary>
        /// Flushes all commands from the buffer, executing them on the main thread.
        /// Called once per main thread tick (e.g., in GameManager._PhysicsProcess).
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
                    // TDD 01.3: Error Handling - Listeners are wrapped in try/catch.
                    // This ensures one command's error doesn't crash the entire flush.
                    GD.PrintErr($"EventBus: Error executing batched command: {e.Message}\n{e.StackTrace}");
                }
            }
        }
    }
}
```

This `EventBus` setup correctly implements the command buffer pattern for safe main thread updates. Our `JobSystem.Schedule()` method already uses `_eventBus.AddCommand()` for job completion callbacks, so this pipeline is functional.

### 3. Double Buffering: Safe Read/Write for Concurrent State

For systems where background jobs need to *read* a state while simultaneously *writing* a *new* state (e.g., updating thousands of virtual agents in the `EcologyManager`), direct modification can lead to race conditions (job A reads old data while job B writes new data to the same location, leading to corrupted state).

**Double Buffering** solves this by providing two copies of the state:

1.  **`State_Read` (Current)**: The authoritative state that all jobs and the main thread read from. This state is immutable during a frame/tick.
2.  **`State_Write` (Next)**: A buffer where background jobs write their changes.
3.  **Swap**: At a safe synchronization point (e.g., the end of the simulation tick on the main thread), `State_Read` is replaced with `State_Write`.

This ensures that:

*   All readers (jobs, main thread) always see a consistent, complete snapshot of the data.
*   Writers (jobs) never interfere with readers.

**Example: Ecology Simulation (TDD 13.4)**

Let's illustrate how `EcologyManager` (which we'll implement later) might use double buffering for its `VirtualAgent` data.

**Conceptual `EcologyManager` (will be implemented in Module 7):**

```csharp
// _Brain/Systems/Ecology/EcologyManager.cs (Conceptual)
using System.Collections.Generic;
using System.Linq;
using System.Threading; // For Interlocked if needed for index swaps

namespace Sigilborne.Systems.Ecology
{
    // A simplified struct for a virtual agent
    public struct VirtualAgent
    {
        public int DefID;
        public Vector2 Pos; // Godot.Vector2 for simplicity
        public float Health;
        public float Hunger;
    }

    public class EcologyManager
    {
        // Two buffers for VirtualAgent data
        private List<VirtualAgent>[] _agentBuffers = new List<VirtualAgent>[2];
        private int _readBufferIndex = 0; // Index of the buffer currently being read from
        private int _writeBufferIndex = 1; // Index of the buffer currently being written to

        public EcologyManager()
        {
            _agentBuffers[0] = new List<VirtualAgent>();
            _agentBuffers[1] = new List<VirtualAgent>();
            // Initialize with some agents
            _agentBuffers[0].Add(new VirtualAgent { DefID = 1, Pos = new Vector2(10, 10), Health = 100, Hunger = 50 });
            _agentBuffers[0].Add(new VirtualAgent { DefID = 2, Pos = new Vector2(20, 20), Health = 80, Hunger = 60 });
        }

        /// <summary>
        /// Returns the current read-only list of virtual agents.
        /// Jobs will read from this.
        /// </summary>
        public IReadOnlyList<VirtualAgent> GetReadAgents()
        {
            return _agentBuffers[_readBufferIndex];
        }

        /// <summary>
        /// Returns the list where jobs should write their modified agents.
        /// </summary>
        public List<VirtualAgent> GetWriteAgents()
        {
            // Clear the write buffer before jobs start writing to it
            _agentBuffers[_writeBufferIndex].Clear();
            return _agentBuffers[_writeBufferIndex];
        }

        /// <summary>
        /// Swaps the read and write buffers. Called on the main thread after all jobs complete.
        /// </summary>
        public void SwapBuffers()
        {
            // Atomically swap indices to prevent race conditions if multiple threads were involved
            // For simple List swapping, direct assignment is fine if called from main thread.
            int temp = _readBufferIndex;
            _readBufferIndex = _writeBufferIndex;
            _writeBufferIndex = temp;
            GD.Print($"EcologyManager: Swapped buffers. New read index: {_readBufferIndex}");
        }

        // --- Example Job for Ecology Update ---
        public struct EcologyUpdateJob : IJob
        {
            public IReadOnlyList<VirtualAgent> ReadAgents; // Read from here
            public List<VirtualAgent> WriteAgents;      // Write to here
            public int StartIndex;
            public int EndIndex;

            public void Execute()
            {
                for (int i = StartIndex; i < EndIndex; i++)
                {
                    VirtualAgent agent = ReadAgents[i];
                    // Simulate agent logic: move, get hungry
                    agent.Pos += new Vector2(1, 0); // Simple movement
                    agent.Hunger = Math.Min(100, agent.Hunger + 5); // Get hungrier
                    WriteAgents.Add(agent); // Add to the write buffer
                }
                GD.Print($"EcologyUpdateJob: Processed agents {StartIndex}-{EndIndex-1}.");
            }
        }
    }
}
```

**Conceptual Integration into `GameManager._PhysicsProcess`:**

```csharp
// _Brain/Core/GameManager.cs (Conceptual _PhysicsProcess)
// ...
    public EcologyManager Ecology { get; private set; } // Add EcologyManager property

    public override void _PhysicsProcess(double delta)
    {
        // ... (existing calls) ...

        // --- Ecology Double Buffer Process (Conceptual) ---
        // 1. Get the write buffer and clear it
        List<VirtualAgent> writeBuffer = Ecology.GetWriteAgents();
        // 2. Get the read buffer (snapshot of current state)
        IReadOnlyList<VirtualAgent> readBuffer = Ecology.GetReadAgents();
        
        // 3. Schedule jobs to process agents
        int agentsPerJob = readBuffer.Count / Environment.ProcessorCount; // Divide work
        for (int i = 0; i < readBuffer.Count; i += agentsPerJob)
        {
            int startIndex = i;
            int endIndex = Math.Min(i + agentsPerJob, readBuffer.Count);
            Jobs.Schedule(new EcologyManager.EcologyUpdateJob
            {
                ReadAgents = readBuffer,
                WriteAgents = writeBuffer,
                StartIndex = startIndex,
                EndIndex = endIndex
            },
            // The last job to complete would trigger the buffer swap
            // This requires more complex job completion tracking for multiple jobs.
            // For simplicity, let's assume one job for now or a simpler swap logic.
            null // No direct callback here; swap happens after ALL jobs are done.
            );
        }

        // 4. After all jobs *complete* (this is the tricky part for multiple jobs), swap buffers.
        // A robust solution would involve a countdown latch or similar mechanism.
        // For now, we'll just demonstrate the swap, assuming jobs are fast enough.
        // In reality, you'd track job completion and then call swap.
        // For this chapter, we're focusing on the *concept* of the buffers.
        // Ecology.SwapBuffers(); // This would be called AFTER all jobs finish and their results are written.
        // ...
```

**Key Takeaways for Double Buffering:**

*   **Snapshot Consistency**: Jobs always operate on a consistent snapshot of the data.
*   **No Concurrent Writes to Same Location**: Jobs write to a separate buffer, preventing direct write conflicts.
*   **Synchronization Point**: The buffer swap is a single, atomic operation on the main thread, making the new state authoritative.

### 4. Summary

You have solidified your understanding of thread safety in Sigilborne's hybrid architecture. By strictly adhering to the "No-Node" Rule and implementing Command Buffers via the `EventBus`, you've created a safe pipeline for background jobs to communicate changes to the main thread. Furthermore, you've grasped the conceptual framework of Double Buffering, a critical strategy for managing concurrent read/write operations on shared data, particularly for large simulation sets like ecology. These principles, as detailed in TDD 13.3, are fundamental for building a performant and stable game.

### Next Steps

With our thread safety mechanisms in place, the next chapter will focus on standardizing how Godot Scenes are composed and how `EntityView.gd` acts as the crucial communication bridge between our C# Brain and GDScript Body, ensuring visual data is always reactive and decoupled from simulation logic.