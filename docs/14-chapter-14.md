## Chapter 1.14: Debugging Tools - Console & State Inspector

Developing a complex, systemic game like Sigilborne, where thousands of entities interact and the world state is constantly evolving, demands powerful debugging tools. Without them, understanding and fixing issues in a hybrid C# Brain / GDScript Body architecture would be nearly impossible. This chapter implements a basic **Debug Console** and lays the conceptual groundwork for a **State Inspector**, as specified in TDD 01.5.

### 1. The Need for Robust Debugging Tools

*   **Visibility into the Brain**: The C# Brain operates on pure data. We need ways to inspect and manipulate this data at runtime without stopping the game.
*   **Runtime Control**: Quickly test game mechanics, spawn items, change time, or toggle debug flags without recompiling or restarting.
*   **Hybrid Complexity**: Traditional debugger breakpoints might not be enough when issues span C# logic and GDScript visuals.

### 2. The Debug Console: Quake-Style Input

A Quake-style console (activated by a key like `~`) allows players (and developers) to execute commands at runtime.

**Implementation Plan (GDScript UI + C# Backend):**

1.  **GDScript UI**: A `Control` node with a `LineEdit` for input and `RichTextLabel` for output. It will capture input and forward commands to C#.
2.  **C# Backend**: A `DebugCommandSystem` that parses commands and executes corresponding actions on the Brain's systems.

#### 2.1. C# `DebugCommandSystem` (Backend)

This system will reside in our `_Brain/Utils` folder.

1.  Create a new C# script `res://_Brain/Utils/DebugCommandSystem.cs`:

```csharp
// _Brain/Utils/DebugCommandSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using System.Linq;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Biology;

namespace Sigilborne.Utils
{
    /// <summary>
    /// Represents a debug command that can be executed.
    /// </summary>
    public struct DebugCommand
    {
        public string Name; // e.g., "give"
        public string Description; // e.g., "Gives item to player: /give [item_id] [amount]"
        public Action<string[]> Action; // The method to execute, with arguments
    }

    /// <summary>
    /// Manages and executes debug commands from the console.
    /// This is the C# backend for our Quake-style debug console.
    /// </summary>
    public class DebugCommandSystem
    {
        private Dictionary<string, DebugCommand> _commands = new Dictionary<string, DebugCommand>();
        private GameManager _gameManager; // Reference to access other systems

        public DebugCommandSystem(GameManager gameManager)
        {
            _gameManager = gameManager;
            RegisterDefaultCommands();
            GD.Print("DebugCommandSystem: Initialized.");
        }

        private void RegisterCommand(string name, string description, Action<string[]> action)
        {
            _commands[name.ToLower()] = new DebugCommand { Name = name, Description = description, Action = action };
        }

        private void RegisterDefaultCommands()
        {
            // TDD 01.5: Example Commands
            RegisterCommand("help", "Displays all available commands.", (args) =>
            {
                GD.Print("--- Available Commands ---");
                foreach (var cmd in _commands.Values.OrderBy(c => c.Name))
                {
                    GD.Print($"/ {cmd.Name}: {cmd.Description}");
                }
                GD.Print("--------------------------");
            });

            RegisterCommand("time", "Sets the in-game hour: /time set [hour]", (args) =>
            {
                if (args.Length != 2 || args[0].ToLower() != "set" || !int.TryParse(args[1], out int hour))
                {
                    GD.PrintErr("Usage: /time set [hour (0-23)]");
                    return;
                }
                _gameManager.Time.SetCurrentHour(hour); // We'll add SetCurrentHour to TimeSystem
                GD.Print($"Time set to {hour}:00.");
            });

            RegisterCommand("damage", "Damages the player: /damage [amount]", (args) =>
            {
                if (args.Length != 1 || !float.TryParse(args[0], out float amount))
                {
                    GD.PrintErr("Usage: /damage [amount]");
                    return;
                }
                _gameManager.PlayerStats.TakeDamage(amount);
                GD.Print($"Player took {amount} damage.");
            });

            RegisterCommand("godmode", "Toggles god mode (placeholder).", (args) =>
            {
                // Implement actual god mode logic later (e.g., set player health to infinite, disable damage)
                GD.Print("God mode toggled (placeholder).");
            });

            RegisterCommand("spawn", "Spawns an entity: /spawn [type] [def_id] [x] [y]", (args) =>
            {
                if (args.Length != 4 || !Enum.TryParse(args[0], true, out EntityType type) ||
                    !float.TryParse(args[2], out float x) || !float.TryParse(args[3], out float y))
                {
                    GD.PrintErr("Usage: /spawn [EntityType] [definition_id] [x] [y]");
                    GD.PrintErr($"Available types: {string.Join(", ", Enum.GetNames(typeof(EntityType)))}");
                    return;
                }
                string defId = args[1];
                _gameManager.Entities.CreateEntity(type, defId, new Vector2(x, y));
                GD.Print($"Spawned {type} '{defId}' at ({x},{y}).");
            });
        }

        /// <summary>
        /// Executes a command string received from the console UI.
        /// </summary>
        /// <param name="commandLine">The full command string (e.g., "/time set 12").</param>
        public void ExecuteCommand(string commandLine)
        {
            commandLine = commandLine.Trim();
            if (string.IsNullOrEmpty(commandLine) || !commandLine.StartsWith("/"))
            {
                GD.Print("DebugCommandSystem: Commands must start with '/'.");
                return;
            }

            string[] parts = commandLine.Substring(1).Split(' ', StringSplitOptions.RemoveEmptyEntries);
            if (parts.Length == 0) return;

            string commandName = parts[0].ToLower();
            string[] args = parts.Skip(1).ToArray();

            if (_commands.TryGetValue(commandName, out DebugCommand cmd))
            {
                try
                {
                    cmd.Action.Invoke(args);
                }
                catch (Exception e)
                {
                    GD.PrintErr($"Error executing command '{commandName}': {e.Message}");
                }
            }
            else
            {
                GD.Print($"Unknown command: '{commandName}'. Type /help for a list of commands.");
            }
        }
    }
}
```

#### 2.2. Integrate `DebugCommandSystem` into `GameManager`

1.  Add `using Sigilborne.Utils;` at the top of `_Brain/Core/GameManager.cs`.
2.  Add a `DebugCommandSystem` property.
3.  Initialize `DebugCommandSystem` in `InitializeSystems()`, passing `this` (the `GameManager` instance) so it can access other systems.

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems;
using Sigilborne.Systems.Biology;
using Sigilborne.Utils; // Add this using directive

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
    public DebugCommandSystem DebugCommands { get; private set; } // Add DebugCommandSystem property

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        
        // --- Test PlayerStatSystem (Damage) ---
        // Let's remove the test damage from _Ready for now, so we can control it via console.
        // GD.Print("\n--- Testing PlayerStatSystem ---");
        // PlayerStats.TakeDamage(10f); // Player takes damage
        // PlayerStats.TakeDamage(20f);
        // PlayerStats.TakeDamage(70f); // Should kill the player
        // GD.Print("--- End Testing PlayerStatSystem ---\n");
    }

    public override void _PhysicsProcess(double delta)
    {
        Time.Tick(delta);
        Events.FlushCommands();
        World.Tick(delta);
        Transforms.Tick(delta);
        // DebugCommandSystem doesn't have a Tick method, its operations are command driven.
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

        DebugCommands = new DebugCommandSystem(this); // Initialize DebugCommandSystem here, passing GameManager itself
        GD.Print("  - DebugCommandSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

#### 2.3. Add `SetCurrentHour` to `TimeSystem.cs`

The `DebugCommandSystem` needs a way to modify the game hour.

Open `_Brain/Core/TimeSystem.cs` and add this method:

```csharp
// _Brain/Core/TimeSystem.cs
using Godot;
using System;

namespace Sigilborne.Core
{
    public class TimeSystem
    {
        // ... (existing properties and constants) ...

        public TimeSystem() { /* ... */ }

        public void Tick(double delta) { /* ... */ }

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
            // Optionally, publish an event: _eventBus.Publish(new TimeChangedEvent { Hour = CurrentHour });
        }
    }
}
```

#### 2.4. GDScript `DebugConsole` (UI)

This is the visual part of the console. It will be a `CanvasLayer` that toggles visibility.

1.  Create a new folder `res://_Body/Scenes/UI/Debug/`.
2.  Create a new scene `res://_Body/Scenes/UI/Debug/DebugConsole.tscn`:
    *   Root node: `CanvasLayer`. Rename it `DebugConsole`.
    *   Child node: `PanelContainer`. Rename it `ConsolePanel`.
        *   In Inspector, set `Layout > Anchors Presets` to `Full Rect`.
        *   Set `Margin > Top` to `0`, `Bottom` to `50%` (covers top half of screen).
        *   Set `Theme Overrides > StyleBoxes > Panel` to `New StyleBoxFlat`.
            *   Set `Bg Color` to `rgba(0, 0, 0, 0.8)`.
    *   Child of `ConsolePanel`: `VBoxContainer`. Rename it `ConsoleVBox`.
        *   `Layout > Anchors Presets` to `Full Rect`.
        *   `Margin > All` to `5`.
    *   Child of `ConsoleVBox`: `RichTextLabel`. Rename it `OutputLabel`.
        *   `Size Flags > Vertical` to `Expand`.
        *   `Scroll Fit Content` to `true`.
        *   `Selection Enabled` to `true`.
        *   `Autowrap Mode` to `Word`.
    *   Child of `ConsoleVBox`: `LineEdit`. Rename it `InputLine`.
        *   `Placeholder Text` to `Type /help for commands...`.
        *   `Release Focus on Echo` to `false` (so it stays focused).
        *   `Context Menu Enabled` to `true`.
3.  Attach a new GDScript to the `DebugConsole` node: `res://_Body/Scripts/UI/Debug/DebugConsoleController.gd`.

```gdscript
# _Body/Scripts/UI/Debug/DebugConsoleController.gd
class_name DebugConsoleController extends CanvasLayer

@onready var console_panel: PanelContainer = $ConsolePanel
@onready var output_label: RichTextLabel = $ConsolePanel/ConsoleVBox/OutputLabel
@onready var input_line: LineEdit = $ConsolePanel/ConsoleVBox/InputLine

var is_open: bool = false:
    set(value):
        is_open = value
        console_panel.visible = is_open
        if is_open:
            input_line.grab_focus()
        else:
            input_line.release_focus()

func _ready():
    is_open = false # Start closed
    console_panel.visible = false # Ensure it's hidden initially
    input_line.text_submitted.connect(Callable(self, "_on_input_line_text_submitted"))
    
    # Redirect Godot's GD.print output to our console
    # This captures all GD.print calls, including C# ones.
    # Note: For production, you might want to filter this.
    GD.push_error("DebugConsoleController: Capturing GD.print output. Use GD.print_raw for unfiltered output.")
    
    # Capture existing output (if any)
    for i in range(OS.get_stdout_line_count()):
        _append_output(OS.get_stdout_line(i))

func _notification(what: int):
    # This is a hacky way to capture GD.print, as there's no direct signal.
    # In a real game, you might use a custom Logger that prints to both console and UI.
    if what == NOTIFICATION_WM_SIZE_CHANGED or what == NOTIFICATION_VISIBILITY_CHANGED:
        output_label.scroll_to_line(output_label.get_line_count() - 1)

func _append_output(text: String):
    output_label.append_text(text + "\n")
    output_label.scroll_to_line(output_label.get_line_count() - 1)

func _input(event: InputEvent):
    # TDD 01.5: Quake-style console (tilde key)
    if event.is_action_pressed("toggle_console"): # Map "toggle_console" to '~' in Project Settings
        is_open = not is_open
        get_viewport().set_input_as_handled() # Prevent input from going to game

func _on_input_line_text_submitted(text: String) -> void:
    _append_output("> " + text) # Echo command to output
    input_line.clear()
    
    if text.is_empty():
        return
        
    # Forward command to C# DebugCommandSystem
    if GameManager.Instance != null and GameManager.Instance.DebugCommands != null:
        GameManager.Instance.DebugCommands.ExecuteCommand(text)
    else:
        _append_output("Error: C# DebugCommandSystem not ready.")
```

#### 2.5. Integrate `DebugConsole` into `Main.tscn`

1.  Open `res://Main.tscn`.
2.  Add an instance of `res://_Body/Scenes/UI/Debug/DebugConsole.tscn` as a child of the `Main` root node.
3.  Save `Main.tscn`.

#### 2.6. Map the Toggle Console Input Action

1.  Go to `Project > Project Settings... > Input Map` tab.
2.  In the "Action" field, type `toggle_console`.
3.  Click `Add`.
4.  Click the `+` icon next to `toggle_console`.
5.  Press the `~` (tilde) key.
6.  Close Project Settings.

### 3. Testing the Debug Console

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  The game loads `Gameplay.tscn`.
4.  Press the `~` key. The console should appear.
5.  Type `/help` and press Enter. You should see the list of commands.
6.  Type `/time set 12` and press Enter. You should see "Time set to 12:00."
7.  Type `/damage 50` and press Enter. The HUD health bar should update.
8.  Type `/spawn Animal deer 300 300` and press Enter. A new entity should be spawned visually (though it won't move yet, as it lacks a movement component).
9.  Press `~` again to close the console.

This confirms our Debug Console is functional, allowing us to interact with the C# Brain's systems at runtime.

### 4. State Inspector (Conceptual)

TDD 01.5 also mentions a **State Inspector**. This tool would allow inspecting the raw data of an entity (its components) by hovering the mouse over its visual representation.

**Conceptual Implementation:**

1.  **Body (GDScript)**:
    *   A `Control` UI element for displaying data.
    *   A `_Process()` method in a `DebugInputHandler.gd` script that performs a `raycast` from the mouse position.
    *   If the raycast hits an `EntityView.gd`, it gets its `entity_id`.
    *   It then calls a C# method: `GameManager.Instance.DebugCommands.GetEntityDebugInfo(entity_id)`.
2.  **Brain (C#)**:
    *   `DebugCommandSystem.GetEntityDebugInfo(EntityID id)`: This method would query the `EntityManager` and all other relevant systems (e.g., `Transforms`, `PlayerStats`) for all components associated with that `EntityID`.
    *   It would serialize this data into a `Dictionary<string, string>` or a custom debug struct.
    *   It would then emit a C# event: `OnEntityDebugInfoReceived(EntityID id, Dictionary<string, string> info)`.
3.  **Body (GDScript UI)**:
    *   The `State Inspector` UI listens to `OnEntityDebugInfoReceived` and populates its fields.

This would provide invaluable insight into the live state of any entity in the simulation. We won't implement this fully in this course, but understanding its architecture is key.

### Summary

You have successfully implemented a functional **Debug Console**, providing a Quake-style interface to execute commands and interact with the C# Brain's systems at runtime. By creating a C# `DebugCommandSystem` and a reactive GDScript `DebugConsoleController`, you've established a vital tool for inspecting and manipulating game state, adhering to TDD 01.5's specifications. This significantly enhances your ability to develop and troubleshoot Sigilborne's complex simulation.

### Next Steps

The next chapter will focus on refining the **Main Loop Execution Order**, detailing how `_Process` (GDScript) and `_PhysicsProcess` (C#) are synchronized, and how input processing, simulation ticks, and visual interpolation work together to maintain game integrity and performance.