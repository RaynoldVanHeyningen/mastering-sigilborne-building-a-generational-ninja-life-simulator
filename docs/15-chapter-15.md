## Chapter 1.15: Main Loop Execution Order - Tick vs. Frame

Understanding the main loop is fundamental to developing a high-performance, deterministic game like Sigilborne, especially with our hybrid C# Brain / GDScript Body architecture. This chapter details the precise execution order of Godot's `_Process` and `_PhysicsProcess` methods, and how we leverage them to synchronize input, advance the simulation tick, and perform visual interpolation, as specified in TDD 01.6.

### 1. The "Tick" vs. "Frame" Distinction

*   **Simulation Tick (Brain)**:
    *   **Fixed Timestep**: Occurs at a consistent rate (e.g., 20Hz, or every 0.05 seconds), regardless of frame rate.
    *   **Deterministic**: Ideal for all game logic, physics calculations, and AI decisions. Ensures consistent results across different machines.
    *   **Owner**: C# Brain (`GameManager._PhysicsProcess`).
*   **Render Frame (Body)**:
    *   **Variable Timestep**: Occurs as fast as the hardware can render (e.g., 60Hz, 144Hz, or more), tied to the monitor's refresh rate.
    *   **Non-Deterministic**: Used for visual updates, animations, and smooth interpolation.
    *   **Owner**: GDScript Body (`_Process` method in visual scripts).

The challenge is to connect these two distinct loops seamlessly.

### 2. Main Loop Execution Order (TDD 01.6)

Godot's engine processes its main loop in a specific order. We're fitting our Brain and Body logic into this.

1.  **`_Process(delta)` (GDScript Body)**:
    *   **Input Capture**: Captures raw player input (keyboard, mouse, gamepad).
    *   **Input Buffering**: Sends a snapshot of this input to the C# Brain's input buffer.
    *   **Visual Interpolation**: Smoothly moves visual elements (like `EntityView` sprites) between known simulation states.
    *   **UI Updates**: Updates UI elements based on the current visual state.
    *   **Audio/VFX**: Triggers non-gameplay-critical visual/audio effects.

2.  **`_PhysicsProcess(delta)` (C# Brain)**:
    *   **Phase 1: Process Input Buffer**: Reads the latest input snapshots from the buffer.
    *   **Phase 2: Run Simulation Systems**: This is the "Tick" where all core game logic runs (AI, Biology, Combat, Magic, WorldSimulation).
    *   **Phase 3: Resolve Collisions/Events**: Handles collision responses and processes any pending commands from the Job System's buffer.
    *   **Phase 4: Emit State Update Signals**: Broadcasts the new authoritative game state to the GDScript Body via the `EventBus`.

### 3. Implementing the Input Buffer (Body -> Brain)

To pass input from GDScript (`_Process`) to C# (`_PhysicsProcess`), we need an input buffer. TDD 02.1 specifies `InputBuffer` in C#, and TDD 12.2 defines `PlayerInputFrame`.

#### 3.1. C# `PlayerInputFrame` and `InputBuffer`

1.  Create a new folder `res://_Brain/Systems/Input/`.
2.  Create `res://_Brain/Systems/Input/PlayerInputFrame.cs`:

```csharp
// _Brain/Systems/Input/PlayerInputFrame.cs
using Godot;
using System;

namespace Sigilborne.Systems.Input
{
    /// <summary>
    /// Represents a snapshot of player input at a specific moment.
    /// This is sent from the GDScript Body to the C# Brain.
    /// </summary>
    public struct PlayerInputFrame
    {
        public Vector2 MoveVector;      // X,Y (-1 to 1)
        public Vector2 LookVector;      // Mouse position or Stick (e.g., normalized direction or screen coords)
        public bool IsSprintHeld;
        public bool IsShiftHeld;        // For "Sliding" mechanic
        public bool[] HotbarKeys;       // 1-0 (array of 10 booleans)
        public bool InteractPressed;
        public double Timestamp;        // When this input was captured (real time)

        public PlayerInputFrame(Vector2 moveVector, Vector2 lookVector, bool isSprintHeld, bool isShiftHeld, bool[] hotbarKeys, bool interactPressed, double timestamp)
        {
            MoveVector = moveVector;
            LookVector = lookVector;
            IsSprintHeld = isSprintHeld;
            IsShiftHeld = isShiftHeld;
            HotbarKeys = hotbarKeys ?? new bool[10]; // Ensure it's not null
            InteractPressed = interactPressed;
            Timestamp = timestamp;
        }

        public override string ToString()
        {
            return $"Move: {MoveVector}, Sprint: {IsSprintHeld}, Shift: {IsShiftHeld}, Interact: {InteractPressed}, T: {Timestamp:F3}";
        }
    }
}
```

3.  Create `res://_Brain/Systems/Input/InputSystem.cs`:

```csharp
// _Brain/Systems/Input/InputSystem.cs
using Godot;
using System;
using System.Collections.Concurrent; // For ConcurrentQueue
using Sigilborne.Core;

namespace Sigilborne.Systems.Input
{
    /// <summary>
    /// Manages player input, receiving frames from the GDScript Body and providing
    /// the latest input to other Brain systems.
    /// </summary>
    public class InputSystem
    {
        // TDD 02.1: InputBuffer - A ConcurrentQueue to safely receive input frames from GDScript.
        private ConcurrentQueue<PlayerInputFrame> _inputQueue = new ConcurrentQueue<PlayerInputFrame>();
        private PlayerInputFrame _latestInput; // The input frame processed in the current tick

        public InputSystem()
        {
            // Initialize with a default (empty) input frame
            _latestInput = new PlayerInputFrame(Vector2.Zero, Vector2.Zero, false, false, new bool[10], false, 0);
            GD.Print("InputSystem: Initialized.");
        }

        /// <summary>
        /// Receives an input frame from the GDScript Body and adds it to the queue.
        /// This method is called from GDScript on the main thread.
        /// </summary>
        public void EnqueueInput(PlayerInputFrame inputFrame)
        {
            _inputQueue.Enqueue(inputFrame);
        }

        /// <summary>
        /// Processes the input queue, updating the latest input for the current simulation tick.
        /// Called during GameManager._PhysicsProcess (Phase 1).
        /// </summary>
        public void ProcessInputBuffer()
        {
            // Always take the latest input available in the queue for the current tick.
            // If multiple inputs arrived since the last tick, we only care about the most recent one.
            PlayerInputFrame newLatest = _latestInput;
            bool updated = false;
            while (_inputQueue.TryDequeue(out PlayerInputFrame inputFrame))
            {
                newLatest = inputFrame;
                updated = true;
            }

            if (updated)
            {
                _latestInput = newLatest;
                // GD.Print($"InputSystem: Processed latest input: {_latestInput}");
            }
        }

        /// <summary>
        /// Returns the latest processed input frame for other systems to read.
        /// </summary>
        public PlayerInputFrame GetLatestInput()
        {
            return _latestInput;
        }
    }
}
```

#### 3.2. Integrate `InputSystem` into `GameManager`

1.  Add `using Sigilborne.Systems.Input;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add an `InputSystem` property.
3.  Initialize `InputSystem` in `InitializeSystems()`.
4.  Call `InputSystem.ProcessInputBuffer()` in `_PhysicsProcess` (Phase 1).

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Input; // Add this using directive
using Sigilborne.Utils;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    public TimeSystem Time { get; private set; }
    public EventBus Events { get; private set; }
    public WorldSimulation World { get; private set; }
    public EntityManager Entities { get; private set; }
    public TransformSystem Transforms { get; private set; }
    public JobSystem Jobs { get; private set; }
    public PlayerStatSystem PlayerStats { get; private set; }
    public DebugCommandSystem DebugCommands { get; private set; }
    public InputSystem Input { get; private set; } // Add InputSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
    }

    public override void _PhysicsProcess(double delta)
    {
        // TDD 01.6: Phase 1: Process Input Buffer.
        Input.ProcessInputBuffer(); 

        // TDD 01.6: Phase 2: Run Simulation Systems.
        Time.Tick(delta);
        World.Tick(delta); // WorldSimulation might contain AI, Biology, Combat, Magic updates
        Transforms.Tick(delta); // Transforms are updated by simulation logic (e.g., movement system)

        // TDD 01.6: Phase 3: Resolve Collisions/Events.
        Events.FlushCommands(); // Process any batched events from background jobs and deferred commands.

        // TDD 01.6: Phase 4: Emit State Update Signals.
        // These are implicitly handled by systems calling Events.Publish within their Tick methods.
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
        
        Jobs = new JobSystem(Events);
        GD.Print("  - JobSystem initialized.");

        PlayerStats = new PlayerStatSystem(Events, Entities);
        GD.Print("  - PlayerStatSystem initialized.");

        DebugCommands = new DebugCommandSystem(this);
        GD.Print("  - DebugCommandSystem initialized.");

        Input = new InputSystem(); // Initialize InputSystem here
        GD.Print("  - InputSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 3.3. GDScript `InputManager` (Body)

This script will capture input in `_Process` and send it to the C# `InputSystem`.

1.  Create `res://_Body/Scripts/Core/InputManager.gd`:

```gdscript
# _Body/Scripts/Core/InputManager.gd
class_name InputManager extends Node

# --- Singleton Instance ---
static var instance: InputManager

# --- Hotbar Key Mapping ---
const HOTBAR_KEYS: Array[int] = [
    KEY_1, KEY_2, KEY_3, KEY_4, KEY_5,
    KEY_6, KEY_7, KEY_8, KEY_9, KEY_0
]

func _init():
    if instance != null:
        push_error("InputManager: More than one instance detected!")
        queue_free()
        return
    instance = self

func _ready():
    GD.print("InputManager: Initialized. Capturing input for C# Brain.")
    pass # No connections needed here, we poll input in _process

## Captures raw input events and sends a snapshot to the C# InputSystem.
## (TDD 01.6: _Process (GDScript) - Capture Input -> Send to C# Buffer)
func _process(delta: float) -> void:
    # Only send input if GameManager and InputSystem are ready
    if GameManager.Instance == null or GameManager.Instance.Input == null:
        return

    # Check if console is open; if so, consume input
    if DebugConsoleController.instance != null and DebugConsoleController.instance.is_open:
        # Input is handled by the console, don't send to game
        return

    var move_vector: Vector2 = Input.get_vector("move_left", "move_right", "move_up", "move_down")
    var look_vector: Vector2 = Vector2.ZERO # Placeholder for mouse/stick look
    var is_sprint_held: bool = Input.is_action_pressed("sprint")
    var is_shift_held: bool = Input.is_action_pressed("shift") # Placeholder for shift-slide
    var interact_pressed: bool = Input.is_action_just_pressed("interact")

    var hotbar_keys_pressed: Array[bool] = []
    for i in range(HOTBAR_KEYS.size()):
        hotbar_keys_pressed.append(Input.is_key_pressed(HOTBAR_KEYS[i]))

    var input_frame = PlayerInputFrame.new() # Create a new C# struct instance via Godot's binding
    input_frame.MoveVector = move_vector
    input_frame.LookVector = look_vector
    input_frame.IsSprintHeld = is_sprint_held
    input_frame.IsShiftHeld = is_shift_held
    input_frame.HotbarKeys = hotbar_keys_pressed.to_array() # Convert GDScript Array to C# Array
    input_frame.InteractPressed = interact_pressed
    input_frame.Timestamp = Time.get_ticks_msec() / 1000.0 # Real-time timestamp

    GameManager.Instance.Input.EnqueueInput(input_frame)
    # GD.print("InputManager: Sent input frame: %s" % input_frame.to_string()) # Excessive printing, use for debug
```

#### 3.4. Update `Main.tscn`

1.  Open `res://Main.tscn`.
2.  Add an instance of `res://_Body/Scripts/Core/InputManager.gd` as a child of the `Main` root node.
3.  Save `Main.tscn`.

#### 3.5. Map Movement Input Actions

1.  Go to `Project > Project Settings... > Input Map` tab.
2.  Add the following actions and map keys:
    *   `move_up`: `W`
    *   `move_down`: `S`
    *   `move_left`: `A`
    *   `move_right`: `D`
    *   `sprint`: `Shift` (Left)
    *   `shift`: `Alt` (Left) (for our shift-slide mechanic, distinct from sprint)
    *   `interact`: `E`
3.  Close Project Settings.

### 4. Visual Interpolation (Body)

Our `EntityView.gd` already handles this (Chapter 1.9). The `_physics_process` method in `EntityView.gd` `lerps` the visual position towards `brain_target_position`, which is updated by C# `OnEntityMoved` events. This ensures smooth movement regardless of the Brain's fixed tick rate.

### 5. Testing the Main Loop Integration

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  You should see the NPC moving and rotating as before.
5.  Now, the console output will be flooded with `InputManager: Sent input frame...` if you uncommented the print statement.
6.  Press WASD. You won't see any immediate movement from the player yet, because we haven't implemented a `MovementSystem` in C# that *uses* the input. However, the input frames are correctly being sent to the C# Brain.

To confirm the input is reaching the C# Brain, you can temporarily add a print statement to `InputSystem.ProcessInputBuffer()`:

```csharp
// _Brain/Systems/Input/InputSystem.cs (inside ProcessInputBuffer)
// ...
            if (updated)
            {
                _latestInput = newLatest;
                GD.Print($"InputSystem: Processed latest input: {_latestInput}"); // Temp debug print
            }
// ...
```
Run again, and you'll see the input frames being processed by the C# Brain.

### Summary

You have successfully established the **Main Loop Execution Order** for Sigilborne, clearly distinguishing between the fixed-timestep simulation tick (C# Brain in `_PhysicsProcess`) and the variable-timestep render frame (GDScript Body in `_Process`). By implementing a C# `InputSystem` to receive `PlayerInputFrame` snapshots from the GDScript `InputManager`, and confirming `EntityView.gd`'s role in visual interpolation, you've created a robust pipeline for handling input, advancing simulation, and maintaining smooth visuals, strictly adhering to TDD 01.6's specifications.

### Next Steps

This concludes **Module 1: Core Architecture - The Brain & Body Paradigm**. You now have a fully functional architectural backbone for Sigilborne. We will now move on to **Module 2: Player Input & Core Movement**, where we will implement the actual movement logic in the C# Brain, using the input frames we just set up.