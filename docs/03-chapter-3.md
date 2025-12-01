## Chapter 1.3: The Body (GDScript) - Reactive Presentation Layer

In Sigilborne's hybrid architecture, the "Body" is our GDScript layer, solely responsible for the game's visual presentation. It's a reactive layer, meaning it doesn't calculate game outcomes or own authoritative data; instead, it listens for state updates from the C# Brain and visualizes them. This separation ensures optimal framerates, allows for rapid iteration on visuals, and keeps our codebase clean and modular.

This chapter will focus on implementing the `SceneLoader` in GDScript, as outlined in TDD 01. `SceneLoader` will manage high-level game state transitions, such as moving between the main menu, loading screens, and core gameplay.

### 1. Understanding the Role of the Body (GDScript)

The GDScript layer is the "Body" of our game, the puppet show that brings the Brain's simulation to life.

**Key Characteristics of the Body:**

*   **Reactive**: It primarily listens to events and signals emitted by the C# Brain. It should not initiate game logic that affects the core simulation state.
*   **Visual-Centric**: Its responsibilities include rendering sprites, playing animations, handling UI, managing particle effects, and playing audio.
*   **Input Capture**: It captures raw player input and forwards it to the C# Brain for processing.
*   **Performance-Oriented**: GDScript excels at rapid prototyping and visual scripting. Keeping its logic focused on presentation ensures Godot can render frames as fast as possible.

### 2. Implementing the SceneLoader

The `SceneLoader` will be a central GDScript singleton that orchestrates scene changes. It will handle the visual aspects of loading, like showing a loading screen, fading transitions, and informing the C# Brain when a new level is requested.

First, let's create the necessary directory structure for our GDScript core components, as per TDD 00.

1.  In the Godot editor's FileSystem dock, if you haven't already, ensure you have:
    *   `res://_Body/Scripts/Core/`

2.  Now, create a new GDScript in `res://_Body/Scripts/Core/`:
    *   Right-click on `res://_Body/Scripts/Core/`.
    *   Select `New Script...`.
    *   **Language**: Choose `GDScript`.
    *   **Class Name**: Enter `SceneLoader`.
    *   **Inherits**: Ensure it inherits `Node`.
    *   **Path**: Confirm the path is `res://_Body/Scripts/Core/SceneLoader.gd`.
    *   Click `Create`.

Open `SceneLoader.gd` and modify it as follows:

```gdscript
# _Body/Scripts/Core/SceneLoader.gd
class_name SceneLoader extends Node

# --- Signals ---
# Emitted when a scene has finished loading and is ready to be displayed.
# This signal can be listened to by UI elements (e.g., to hide a loading screen).
signal scene_loaded(scene_path: String)

# --- Singleton Instance ---
# Provides global access to the SceneLoader from other GDScript classes.
static var instance: SceneLoader

func _init():
    if instance != null:
        # This prevents multiple instances if SceneLoader is accidentally added to multiple scenes.
        # In our setup, it will be a child of Main.tscn, ensuring only one instance.
        push_error("SceneLoader: More than one instance detected! This should not happen.")
        queue_free()
        return
    instance = self

func _ready():
    # Connect to C# events (if any are relevant for scene loading, e.g., world_gen_complete)
    # For now, we'll assume the C# GameManager is already present in the scene.
    # We will establish this connection more formally in Chapter 1.12 (Interop Layer).
    pass

# --- Public Methods ---

## Loads a new game level or scene.
## This method handles the visual transition, but the actual world generation/setup
## is expected to be handled by the C# Brain (e.g., GameManager.World.GenerateWorld()).
##
## @param level_path: The resource path to the target scene (e.g., "res://_Body/Scenes/Gameplay.tscn").
func load_level(level_path: String) -> void:
    if not ResourceLoader.exists(level_path):
        push_error("SceneLoader: Cannot load level. Resource does not exist: %s" % level_path)
        return

    GD.print("SceneLoader: Initiating level load for: %s" % level_path)

    # 1. Show Loading Screen (Placeholder)
    # In a real game, you would instantiate and add a loading screen UI scene here.
    # Example: var loading_screen = preload("res://_Body/Scenes/UI/LoadingScreen.tscn").instantiate()
    # add_child(loading_screen)
    # await get_tree().create_timer(0.5).timeout # Simulate loading screen delay

    # 2. Request World Generation from Brain (Placeholder)
    # This is where we'd call a C# method to start the heavy lifting.
    # Example: GameManager.Instance.World.RequestWorldGeneration(level_path)
    # We await a signal from C# when it's done.
    # await GameManager.Instance.Events.world_gen_complete # This will be set up later.
    
    # For now, we simulate the C# Brain's work with a simple delay.
    GD.print("SceneLoader: (Simulating C# Brain's world generation...)")
    await get_tree().create_timer(2.0).timeout # Simulate heavy world generation

    # 3. Load the actual scene resource
    var next_scene: PackedScene = ResourceLoader.load(level_path)
    if next_scene == null:
        push_error("SceneLoader: Failed to load PackedScene for %s" % level_path)
        # Handle error, maybe go back to main menu
        return

    # 4. Change the current scene in the scene tree
    # This frees the current scene and replaces it with the new one.
    get_tree().change_scene_to_packed(next_scene)
    
    # 5. Hide Loading Screen (Placeholder) and Fade In
    # Example: loading_screen.queue_free()
    # (Perform a fade-in animation here)

    GD.print("SceneLoader: Level loaded successfully: %s" % level_path)
    scene_loaded.emit(level_path) # Emit signal to notify other parts of the game
```

**Explanation of Changes:**

*   **`class_name SceneLoader extends Node`**: This declares `SceneLoader` as a custom type, making it easier to reference in other scripts and in the editor.
*   **`signal scene_loaded(scene_path: String)`**: This signal is emitted when a scene successfully loads. Other GDScript nodes (e.g., UI elements that show/hide loading screens) can connect to this.
*   **`static var instance: SceneLoader`**: Implements the Singleton pattern in GDScript, providing global access via `SceneLoader.instance`. The `_init()` method includes a check for duplicate instances.
*   **`_ready()`**: A placeholder for future connections to C# events.
*   **`load_level(level_path: String)`**:
    *   Takes a `level_path` (e.g., `res://_Body/Scenes/Gameplay.tscn`).
    *   Includes placeholders for showing a loading screen and awaiting C# world generation. For now, it uses `await get_tree().create_timer(X).timeout` to simulate these asynchronous operations.
    *   `ResourceLoader.load()` loads the scene resource.
    *   `get_tree().change_scene_to_packed(next_scene)` performs the actual scene switch, replacing the current scene with the new one.
    *   Emits `scene_loaded` upon completion.

### 3. Integrating SceneLoader into the Main Scene

For `SceneLoader` to function, it needs to be part of our initial scene. We'll add it as a child of our `Main` scene.

1.  Open `res://Main.tscn` in Godot.
2.  Select the `Main` node.
3.  Click the "Add Child Node" button (the `+` icon).
4.  Search for `SceneLoader` and add it.
5.  Save `Main.tscn`.

Your `Main.tscn` scene tree should now look like this:

```
Main (Node)
└── GameManager (GameManager.cs)
└── SceneLoader (SceneLoader.gd)
```

### 4. Creating a Placeholder Gameplay Scene

To test our `SceneLoader`, we need a target scene to load. Let's create a very simple "Gameplay" scene.

1.  In Godot, go to `Scene > New Scene`.
2.  Add a `Node2D` as the root. Rename it `Gameplay`.
3.  Add a `Label` node as a child of `Gameplay`.
4.  In the `Label`'s Inspector, set its `Text` property to "Welcome to Sigilborne's World!".
5.  Center the label roughly on the screen by setting its `Position` (e.g., `x=200, y=100`).
6.  Save the scene as `res://_Body/Scenes/Gameplay.tscn`.

### 5. Testing SceneLoader Functionality

Now, let's modify `GameManager.cs` to trigger the `SceneLoader` to load our `Gameplay` scene after initialization. This demonstrates the C# Brain initiating a visual command on the GDScript Body.

Open `_Brain/Core/GameManager.cs` and add the following to its `_Ready()` method, **after** `InitializeSystems();`:

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    public TimeSystem Time { get; private set; }
    public EventBus Events { get; private set; }
    public WorldSimulation World { get; private set; }

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
        // This demonstrates the C# Brain requesting a scene change from the GDScript Body.
        // We ensure SceneLoader is ready before calling it.
        if (SceneLoader.instance != null)
        {
            GD.Print("GameManager: Requesting SceneLoader to load Gameplay scene.");
            // We cannot directly await GDScript methods from C# like this in _Ready,
            // as _Ready is synchronous. We'll queue it or use await for later.
            // For now, let's just call it. The GDScript side handles the await.
            SceneLoader.instance.load_level("res://_Body/Scenes/Gameplay.tscn");
        }
        else
        {
            GD.PrintErr("GameManager: SceneLoader instance not found! Cannot load gameplay scene.");
        }
    }

    // ... (rest of GameManager.cs) ...
}
```

Save `GameManager.cs`. Godot will recompile.

Now, run the `Main.tscn` scene.

**Expected Outcome:**

1.  Godot starts with `Main.tscn`.
2.  `GameManager` and `SceneLoader` get initialized.
3.  `GameManager` calls `SceneLoader.instance.load_level()`.
4.  You'll see "SceneLoader: (Simulating C# Brain's world generation...)" in the output for 2 seconds.
5.  The scene should then switch to `Gameplay.tscn`, displaying "Welcome to Sigilborne's World!".

This confirms that our GDScript `SceneLoader` is working correctly as the visual orchestrator, responding to requests from the C# Brain.

### Summary

You have successfully implemented the `SceneLoader` in GDScript, establishing it as the primary component for managing high-level visual game state transitions. This `SceneLoader` now acts as the "Body's" orchestrator, responding to the C# "Brain's" commands to load new scenes and handle visual feedback. This crucial step solidifies the separation of concerns, ensuring that our GDScript layer remains focused on presentation while the C# layer drives the core simulation.

### Next Steps

In the next chapter, we will formalize our project's directory structure, enforcing the strict separation of concerns that is fundamental to our Brain & Body architecture. This will be critical for maintaining a clean, scalable, and manageable codebase as Sigilborne grows in complexity.