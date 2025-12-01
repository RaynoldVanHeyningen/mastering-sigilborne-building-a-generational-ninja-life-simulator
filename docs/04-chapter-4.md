## Chapter 1.4: Directory Structure - Enforcing Separation of Concerns

A well-defined and strictly adhered-to directory structure is paramount for any large-scale project, especially one employing a hybrid architecture like Sigilborne. Our TDD 00 outlines a clear separation between the "Brain" (C# simulation) and "Body" (GDScript visuals) layers. This chapter will formalize that structure, explaining the rationale behind each folder and establishing consistent naming conventions.

### 1. The Rationale: Why a Strict Structure?

The core principle behind our directory structure is **separation of concerns**.

*   **Clarity**: Developers can immediately tell if a file belongs to the simulation logic or the visual presentation.
*   **Maintainability**: Changes in one layer (e.g., refactoring C# game logic) are less likely to accidentally break code in another layer (e.g., GDScript UI).
*   **Performance**: C# code focused on data and algorithms can be optimized without concern for rendering, and vice-versa for GDScript.
*   **Scalability**: As the project grows, new systems can be added to their respective layers without cluttering the entire project.
*   **Collaboration**: Multiple developers can work on different parts of the game (e.g., one on C# AI, another on GDScript animations) with minimal merge conflicts and clear responsibilities.

### 2. The Canonical Directory Structure

As defined in TDD 00, our `res://` (project root) directory will be organized as follows:

```text
res://
├── .godot/                 # Godot's internal cache and configuration (managed by Godot)
├── project.godot           # Godot project settings (managed by Godot)
├── Sigilborne.csproj       # C# project file (managed by Godot/IDE)
├── Sigilborne.sln          # C# solution file (managed by Godot/IDE)
├── Main.tscn               # Entry Point (our initial scene)
├── _Brain/                 # C# Source Code (The Simulation)
│   ├── Core/               # Main Loop, Event Bus, Time
│   ├── Systems/            # Magic, Combat, Ecology, Economy
│   ├── Data/               # Static Data (Items, Spells), Save Data
│   └── Utils/              # Math, Extensions, Debugging
├── _Body/                  # Godot Assets & GDScript (The Visuals)
│   ├── Scenes/             # .tscn files (UI, Entities, World Chunks)
│   │   ├── Entities/       # Player, NPCs, Animals, Projectiles
│   │   ├── UI/             # HUD, Menus, Inventory
│   │   └── World/          # Terrain, Environment props
│   ├── Scripts/            # .gd files (Visual logic only)
│   │   ├── Core/           # InputManager, SceneLoader, CameraController
│   │   ├── Visuals/        # EntityView, AnimationControllers, VFX/Audio Hooks
│   │   └── UI/             # UI Controllers, Menu Logic
│   ├── Art/                # Sprites, Textures, Shaders, TileSets
│   │   ├── Characters/     # Player, NPCs, Animals
│   │   ├── Environment/    # Tiles, Props, Backgrounds
│   │   ├── VFX/            # Particle textures, Shader graphs
│   │   └── UI/             # Icons, UI elements
│   └── Audio/              # Banks, Streams, SFX, Music
│       ├── Music/
│       └── SFX/
└── _Shared/                # Resources used by both (if any)
    ├── Config/             # Settings, Constants, Enums
    └── Resources/          # Shared data (e.g., custom resource files)
```

### 3. Creating the Full Structure

Let's ensure your project's `res://` directory matches this canonical structure. You've already started this in Chapter 1.1, but let's complete it.

1.  Open your `Sigilborne` project in Godot.
2.  In the FileSystem dock, right-click on `res://` and select `New Folder` to create any missing top-level folders (`_Brain`, `_Body`, `_Shared`).
3.  Navigate into each of these and create their respective subfolders as listed above.

For example, to create `res://_Body/Scenes/Entities/`:
*   Right-click `res://_Body/`.
*   `New Folder` -> `Scenes`.
*   Right-click `res://_Body/Scenes/`.
*   `New Folder` -> `Entities`.

Repeat this process until your FileSystem dock is fully populated according to the TDD.

### 4. Naming Conventions

Consistent naming is vital for readability and quick navigation.

#### 4.1. C# (The Brain) - TDD 00, TDD 01.1

*   **Files/Classes**: `PascalCase` (e.g., `GameManager.cs`, `TimeSystem.cs`, `SpellDefinition.cs`).
*   **Methods/Properties**: `PascalCase` (e.g., `CalculateDamage()`, `CurrentGameTime`).
*   **Local Variables/Parameters**: `camelCase` (e.g., `currentHealth`, `delta`).
*   **Structs**: `PascalCase` (e.g., `EntityID`, `InputFrame`).
*   **Enums**: `PascalCase` (e.g., `GlyphType`, `CastState`).
*   **Namespaces**: `PascalCase` (e.g., `Sigilborne.Core`, `Sigilborne.Systems.Magic`).

#### 4.2. GDScript (The Body) - TDD 00, TDD 16.2

*   **Files/Classes (custom types)**: `PascalCase` (e.g., `SceneLoader.gd`, `PlayerController.gd`).
*   **Variables/Functions**: `snake_case` (e.g., `play_animation()`, `current_health`).
*   **Signals**: `snake_case` (e.g., `scene_loaded`, `entity_moved`).
*   **Node Paths**: `PascalCase` (e.g., `get_node("PlayerCharacter/Visuals/Sprite")`).
*   **Typing**: **ALWAYS** use static typing (`var health: float = 100.0`).

#### 4.3. Godot Assets (Scenes, Resources, Art) - TDD 16.1

*   **Scenes (`.tscn`)**: `PascalCase` (e.g., `PlayerCharacter.tscn`, `IronSword.tscn`, `MainMenu.tscn`).
*   **Resources (`.tres`, `.res`)**: `PascalCase` (e.g., `PlayerStats.tres`, `FireballEffect.tres`).
*   **Art Assets (`.png`, `.wav`, `.ogg`, etc.)**: `snake_case` (e.g., `icon_fire_ball.png`, `sfx_footstep_grass.wav`, `bgm_forest_loop.ogg`).
*   **Nodes in Scene Tree**: `PascalCase` for functional nodes (`Sprite`, `CollisionShape`, `AnimationPlayer`). `PascalCase` for unique nodes (`MainCamera`, `WorldEnvironment`). Custom nodes should also be `PascalCase` (e.g., `PlayerCharacter`, `EnemyGoblin`).

### 5. Applying Naming Conventions to Existing Files

Let's ensure our existing files follow these conventions.

1.  **`res://Main.tscn`**: This is correct as `PascalCase`.
2.  **`res://_Brain/Core/GameManager.cs`**: Correct as `PascalCase`.
3.  **`res://_Brain/Core/EventBus.cs`**: Correct as `PascalCase`.
4.  **`res://_Brain/Core/TimeSystem.cs`**: Correct as `PascalCase`.
5.  **`res://_Brain/Core/WorldSimulation.cs`**: Correct as `PascalCase`.
6.  **`res://_Body/Scripts/Core/SceneLoader.gd`**: Correct as `PascalCase` for its `class_name`.

### 6. Importance of Godot's Class_Name and Static Typing

*   **`class_name`**: In GDScript, `class_name` (e.g., `class_name SceneLoader extends Node`) registers your script as a global type. This allows you to:
    *   Reference it directly in other GDScripts without `preload()` or `load()`.
    *   Instantiate it directly (e.g., `var loader = SceneLoader.new()`).
    *   See it in the "Add Node" dialog in the editor.
    *   This is why `GameManager.cs` could call `SceneLoader.instance.load_level()` even though `SceneLoader` is a GDScript.
*   **Static Typing**: (e.g., `var health: float = 100.0`) is **mandatory** for all GDScript. It improves:
    *   Code readability and maintainability.
    *   Error detection during development.
    *   Performance (Godot can optimize typed code better).
    *   IDE autocompletion.

### 7. Verifying the Structure and Running the Project

After creating all the folders and confirming naming, try running your project (`Main.tscn`) again. Everything should still work as before. If you encounter any errors, double-check your folder paths and script names.

This structured approach might seem like overhead initially, but it's an investment that pays dividends in large, complex projects like Sigilborne, preventing "spaghetti code" and making the game much easier to develop and maintain in the long run.

### Summary

You have successfully established the canonical directory structure for Sigilborne, strictly separating the C# "Brain" from the GDScript "Body" and other shared resources. You've also reviewed and committed to consistent naming conventions for files, classes, methods, and nodes across both languages and assets. This robust organizational framework is a critical step towards building a scalable, maintainable, and collaborative game.

### Next Steps

With our project structure firmly in place, we will now implement the core `Entity` model using a lightweight Entity-Component-System (ECS-lite) approach in C#, which will serve as the atomic unit of our simulation.