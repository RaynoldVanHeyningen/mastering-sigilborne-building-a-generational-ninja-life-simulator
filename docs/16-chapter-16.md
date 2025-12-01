## Chapter 2.1: Input Manager - Capturing Raw Input (GDScript)

Welcome to **Module 2: Player Input & Core Movement**! In our hybrid architecture, the GDScript Body is the primary interface for player interaction. This chapter focuses on the `InputManager.gd` script, which is solely responsible for capturing raw hardware input events (keyboard, mouse, gamepad) from Godot's `_Process` loop, normalizing them, and forwarding them as a concise snapshot to the C# Brain. This adheres to TDD 12.2's `PlayerInputFrame` and `InputManager` specifications.

### 1. The Role of `InputManager.gd`

*   **Capture**: Listens to Godot's input events.
*   **Normalize**: Translates various hardware inputs (e.g., "W" key, "Up Arrow", Left Stick Up) into unified game actions (e.g., `move_up`).
*   **Snapshot**: Collects all relevant input states into a single `PlayerInputFrame` struct.
*   **Forward**: Sends this `PlayerInputFrame` to the C# `InputSystem` for processing by the Brain.
*   **Filter**: Prevents game input from being sent when UI (like the Debug Console) is active.

It does NOT interpret *what* the input means for game logic (e.g., "move character," "cast spell"). That's the Brain's job.

### 2. Reviewing and Enhancing `InputManager.gd`

We've already created a basic `InputManager.gd` in Chapter 1.15. Let's review its structure and enhance it to fully meet the requirements for capturing raw input.

Open `res://_Body/Scripts/Core/InputManager.gd`:

```gdscript
# _Body/Scripts/Core/InputManager.gd
class_name InputManager extends Node

# --- Singleton Instance (TDD 12.1) ---
static var instance: InputManager

# --- Hotbar Key Mapping (TDD 12.5) ---
# Maps Godot key codes to hotbar slots 0-9.
# The C# PlayerInputFrame.HotbarKeys expects an array of 10 booleans.
const HOTBAR_KEYS: Array[int] = [
    KEY_1, KEY_2, KEY_3, KEY_4, KEY_5,
    KEY_6, KEY_7, KEY_8, KEY_9, KEY_0
]

func _init():
    if instance != null:
        push_error("InputManager: More than one instance detected! This should not happen.")
        queue_free()
        return
    instance = self

func _ready():
    GD.print("InputManager: Initialized. Capturing input for C# Brain.")
    # No explicit connections here, as we poll input state in _process.
    # Input actions are defined in Project Settings -> Input Map.
    pass

## Captures raw input events and sends a snapshot to the C# InputSystem.
## This runs every render frame (variable timestep).
## (TDD 01.6: _Process (GDScript) - Capture Input -> Send to C# Buffer)
func _process(delta: float) -> void:
    # Ensure GameManager and its InputSystem are initialized before sending input.
    if GameManager.Instance == null or GameManager.Instance.Input == null:
        return

    # TDD 12.4: Action Routing & Priority - If UI (Debug Console) is open, consume input.
    if DebugConsoleController.instance != null and DebugConsoleController.instance.is_open:
        # Input is handled by the console, don't send to game.
        return

    # --- Capture Movement Input (TDD 12.2: PlayerInputFrame.MoveVector) ---
    var move_vector: Vector2 = Input.get_vector("move_left", "move_right", "move_up", "move_down")

    # --- Capture Look Input (TDD 12.2: PlayerInputFrame.LookVector) ---
    # For a 2D game, LookVector might be mouse position, or a normalized direction from a gamepad stick.
    # For now, let's capture mouse position relative to the viewport center.
    var look_vector: Vector2 = Vector2.ZERO
    if get_viewport() != null:
        var mouse_pos: Vector2 = get_viewport().get_mouse_position()
        var viewport_center: Vector2 = get_viewport().size / 2.0
        look_vector = (mouse_pos - viewport_center).normalized() # Normalized direction from center

    # --- Capture Action Inputs (TDD 12.2: PlayerInputFrame flags) ---
    var is_sprint_held: bool = Input.is_action_pressed("sprint")
    var is_shift_held: bool = Input.is_action_pressed("shift")
    var interact_pressed: bool = Input.is_action_just_pressed("interact") # Use just_pressed for single-trigger actions
    
    # --- Capture Hotbar Inputs (TDD 12.2: PlayerInputFrame.HotbarKeys) ---
    var hotbar_keys_pressed: Array[bool] = []
    # Ensure the array is always 10 elements long to match C# struct
    hotbar_keys_pressed.resize(HOTBAR_KEYS.size()) 
    for i in range(HOTBAR_KEYS.size()):
        hotbar_keys_pressed[i] = Input.is_key_pressed(HOTBAR_KEYS[i])

    # --- Create PlayerInputFrame Snapshot (TDD 12.2) ---
    # PlayerInputFrame is a C# struct, so we instantiate it using `new()`.
    var input_frame = PlayerInputFrame.new() 
    input_frame.MoveVector = move_vector
    input_frame.LookVector = look_vector
    input_frame.IsSprintHeld = is_sprint_held
    input_frame.IsShiftHeld = is_shift_held
    input_frame.HotbarKeys = hotbar_keys_pressed.to_array() # Convert GDScript Array to C# Array
    input_frame.InteractPressed = interact_pressed
    input_frame.Timestamp = Time.get_ticks_msec() / 1000.0 # Real-time timestamp (TDD 12.2)

    # --- Enqueue Input to C# Brain (TDD 12.2) ---
    GameManager.Instance.Input.EnqueueInput(input_frame)
    # GD.print("InputManager: Sent input frame: %s" % input_frame.to_string()) # Keep commented for performance
```

**Key Enhancements:**

*   **`look_vector`**: Now captures a normalized direction from the mouse position relative to the viewport center. This is a common pattern for 2D games where the player "looks" in the direction of the mouse.
*   **`interact_pressed`**: Uses `is_action_just_pressed` to ensure it only triggers once per press, not every frame the button is held.
*   **`hotbar_keys_pressed.resize(HOTBAR_KEYS.size())`**: Explicitly resizes the array to 10 elements, matching the C# `PlayerInputFrame.HotbarKeys` array size expectation.
*   **`PlayerInputFrame.new()`**: Correctly instantiates the C# struct via Godot's binding.
*   **`hotbar_keys_pressed.to_array()`**: Converts the GDScript `Array[bool]` to a C# compatible array.

### 3. Defining Input Actions in Project Settings

For `Input.get_vector()` and `Input.is_action_pressed()`, we rely on Godot's Input Map. We started this in Chapter 1.15. Let's ensure all necessary actions are defined.

1.  Go to `Project > Project Settings... > Input Map` tab.
2.  **Verify/Add Actions and Map Keys:**
    *   `move_up`: `W`
    *   `move_down`: `S`
    *   `move_left`: `A`
    *   `move_right`: `D`
    *   `sprint`: `Shift` (Left)
    *   `shift`: `Alt` (Left) (for our shift-slide mechanic)
    *   `interact`: `E`
    *   `toggle_console`: `~` (tilde)
    *   `hotbar_1`: `1` (Key 1)
    *   `hotbar_2`: `2` (Key 2)
    *   ...
    *   `hotbar_0`: `0` (Key 0)
        *   **Note**: For `hotbar_X` actions, you'll need to add each individually and map the corresponding key. `Input.is_key_pressed(KEY_1)` in GDScript directly checks the key, so defining these actions isn't strictly necessary for `InputManager.gd`'s `HOTBAR_KEYS` array, but it's good practice for consistency and potential UI binding. `Input.get_vector` and `is_action_pressed` *do* require actions.

3.  Close Project Settings.

### 4. Testing Raw Input Capture

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Interact with movement keys (WASD), sprint (Left Shift), and our custom shift key (Left Alt).
5.  Open the debug console (`~`) and type `/help` to ensure console input is still isolated.
6.  Close the console and move the mouse around.

**Expected Output (in the Godot Output console, if you temporarily uncomment the `GD.print` in `InputManager.gd`):**

You should see `InputManager: Sent input frame...` messages, with `MoveVector`, `LookVector`, `IsSprintHeld`, `IsShiftHeld`, `InteractPressed`, and `HotbarKeys` accurately reflecting your inputs. This confirms that `InputManager.gd` is correctly capturing and formatting all raw player input.

### Summary

You have successfully implemented and enhanced `InputManager.gd` to reliably capture and normalize raw hardware input events from Godot's `_Process` loop. By collecting all relevant input states into a `PlayerInputFrame` snapshot and forwarding it to the C# `InputSystem`, you've established the foundational pipeline for player control, strictly adhering to TDD 12.2's specifications. This ensures that the C# Brain receives consistent and timely player commands, decoupled from the Body's rendering.

### Next Steps

The next chapter will focus on **Movement Logic (Brain)**, where we will use the `PlayerInputFrame` data in the C# `InputSystem` to implement standard character movement and our unique "Shift-Sliding" mechanic, bringing our player entity to life in the simulation.