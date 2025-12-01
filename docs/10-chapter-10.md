## Chapter 1.10: Naming Conventions & Scene Interfaces

Consistency in naming and structure is a hallmark of a professional, maintainable codebase. In Sigilborne, with its hybrid C# and GDScript architecture, strict adherence to conventions is even more critical to avoid confusion and streamline collaboration. This chapter formalizes the naming conventions for all files, classes, methods, variables, and nodes, as well as the basic interfaces for Godot Scenes, based on TDD 00 and TDD 16.

### 1. The Importance of Naming Conventions

*   **Readability**: Code is read far more often than it's written. Clear, consistent names make it easier to understand intent.
*   **Maintainability**: Reduces cognitive load when navigating a large project.
*   **Collaboration**: Ensures all team members follow the same rules, preventing "wild west" naming.
*   **Tooling**: Improves IDE autocompletion, search, and refactoring.
*   **Separation of Concerns**: Naming helps reinforce which layer (Brain/Body) a file belongs to.

### 2. Naming Conventions: The Canonical Rules

#### 2.1. C# (The Brain) - TDD 00, TDD 01.1

*   **Files/Classes (`.cs`)**: `PascalCase`.
    *   *Example*: `GameManager.cs`, `TimeSystem.cs`, `SpellDefinition.cs`, `EntityID.cs`, `TransformComponent.cs`, `JobSystem.cs`.
*   **Methods/Properties**: `PascalCase`.
    *   *Example*: `InitializeSystems()`, `CurrentGameTime`, `CreateEntity()`, `Execute()`.
*   **Local Variables/Method Parameters**: `camelCase`.
    *   *Example*: `double delta`, `string definitionID`, `int index`.
*   **Structs**: `PascalCase`.
    *   *Example*: `InputFrame`, `EntityMeta`, `TestJob`.
*   **Enums**: `PascalCase`.
    *   *Example*: `GlyphType`, `CastState`, `EntityType`.
*   **Interfaces**: `IPascalCase` (starting with `I`).
    *   *Example*: `IJob`.
*   **Namespaces**: `PascalCase`, typically `Sigilborne.Subsystem`.
    *   *Example*: `Sigilborne.Core`, `Sigilborne.Entities`, `Sigilborne.Entities.Components`, `Sigilborne.Systems`, `Sigilborne.Utils`.

#### 2.2. GDScript (The Body) - TDD 00, TDD 16.2

*   **Files/Classes (`.gd`) (custom types)**: `PascalCase` using `class_name`.
    *   *Example*: `SceneLoader.gd`, `EntityView.gd`, `InputManager.gd`.
*   **Variables/Functions**: `snake_case`.
    *   *Example*: `var current_health: float`, `func play_animation(anim_name: String)`.
*   **Signals**: `snake_case`.
    *   *Example*: `signal scene_loaded`, `signal anim_event`.
*   **Node Paths (in code)**: `PascalCase` matching the scene tree.
    *   *Example*: `$Visuals/Sprite`, `get_node("MainCamera")`.
*   **Typing**: **ALWAYS** use static typing (`var health: float = 100.0`). This is a hard rule.
*   **Constants**: `SCREAMING_SNAKE_CASE`.
    *   *Example*: `const SMOOTHING_SPEED: float = 10.0`, `const MAX_HEALTH: int = 100`.

#### 2.3. Godot Assets (Scenes, Resources, Art) - TDD 16.1

*   **Scenes (`.tscn`)**: `PascalCase`.
    *   *Example*: `PlayerCharacter.tscn`, `IronSword.tscn`, `MainMenu.tscn`, `Gameplay.tscn`, `EntityRoot.tscn`.
*   **Resources (`.tres`, `.res`)**: `PascalCase`.
    *   *Example*: `PlayerStats.tres`, `FireballEffect.tres`, `DefaultMaterial.tres`.
*   **Art Assets (`.png`, `.wav`, `.ogg`, `.shader`, `.material`, etc.)**: `snake_case`.
    *   *Example*: `icon_fire_ball.png`, `sfx_footstep_grass.wav`, `bgm_forest_loop.ogg`, `shader_water_flow.gdshader`.
*   **Nodes in Scene Tree**: `PascalCase`.
    *   *Functional/Generic Nodes*: `Sprite`, `CollisionShape2D`, `AnimationPlayer`.
    *   *Unique/Specific Nodes*: `MainCamera`, `WorldEnvironment`, `PlayerCharacter`, `EnemyGoblin`.
    *   *Our EntityRoot children*: `Visuals`, `Sprite`, `VFX`, `Collision`, `Audio`, `UI`.

### 3. Scene Interfaces: The "Setup" Function

TDD 16.4 introduces the concept of a "Setup" function for scenes. This is a standardized way for the C# Brain to initialize a newly spawned GDScript Body scene with its authoritative data.

**Rule**: Every `EntityView.gd` (or similar top-level script on a visual entity scene) **MUST** have a `setup()` function.

*   **Purpose**: This function is called by the `EntityViewManager.gd` (which is reacting to a C# `EntitySpawnedEvent`) immediately after `instantiate()` and `add_child()`.
*   **Parameters**: It should accept the necessary C# data (`id`, `initial_position`, `initial_rotation`, `definition_id`) to initialize its visual state.
*   **Example from `EntityView.gd` (Chapter 1.9):**

```gdscript
# _Body/Scripts/Visuals/EntityView.gd
class_name EntityView extends CharacterBody2D

# ... (properties and node references) ...

## Initializes this EntityView with its corresponding C# EntityID.
## This function is called by the C# Brain via the EventBus when an entity is spawned.
## (TDD 16.4: The "Setup" Function)
func setup(id: int, initial_position: Vector2, initial_rotation: float, definition_id: String) -> void:
    entity_id = id
    global_position = initial_position
    brain_target_position = initial_position
    visuals.rotation_degrees = initial_rotation
    # ... (set sprite, etc.) ...
```

This `setup()` function acts as a formal contract between the Brain's data and the Body's visual initialization.

### 4. Visual Interpolation: Smoothness in the Body

TDD 16.4 also highlights visual interpolation. This is a core responsibility of the Body to ensure smooth animations even if the Brain updates at a lower tick rate.

**Rule**: `EntityView.gd` (and similar scripts for moving visuals) **MUST** interpolate their `global_position` towards a `brain_target_position`.

*   **Mechanism**: The `_physics_process()` (or `_process()` for non-physics-based movement) method is used to `lerp()` (linear interpolate) the current visual position towards the last authoritative position received from the Brain.
*   **Example from `EntityView.gd` (Chapter 1.9):**

```gdscript
# _Body/Scripts/Visuals/EntityView.gd
class_name EntityView extends CharacterBody2D

# ... (properties) ...
var brain_target_position: Vector2 = Vector2.ZERO
const SMOOTHING_SPEED: float = 10.0

func _physics_process(delta: float) -> void:
    global_position = global_position.lerp(brain_target_position, delta * SMOOTHING_SPEED)
    # ...
```

This ensures that even if the C# Brain sends position updates at 20Hz, the GDScript Body renders smooth movement at 60Hz or higher.

### 5. Reviewing Our Current Project Against Standards

Let's do a quick check of our existing files to ensure they meet these standards.

*   **Directory Structure**: Fully created in Chapter 1.4.
*   **C# Naming**:
    *   `GameManager.cs`, `TimeSystem.cs`, `EventBus.cs`, `WorldSimulation.cs` (classes): PascalCase.
    *   `EntityManager.cs`, `EntityID.cs`, `TransformComponent.cs`, `StateComponent.cs` (classes/structs): PascalCase.
    *   `IJob.cs`, `TestJob.cs`, `JobSystem.cs` (interface/struct/class): PascalCase, `IJob` starts with I.
    *   All internal methods/properties/variables follow PascalCase/camelCase.
    *   Namespaces `Sigilborne.Core`, `Sigilborne.Entities`, `Sigilborne.Entities.Components`, `Sigilborne.Systems`, `Sigilborne.Utils` are PascalCase. **All good.**
*   **GDScript Naming**:
    *   `SceneLoader.gd`, `EntityView.gd`, `EntityViewManager.gd` (class_name): PascalCase.
    *   Variables/functions like `_active_entity_views`, `_on_entity_spawned`, `brain_target_position`, `SMOOTHING_SPEED` (constants) follow snake_case/SCREAMING_SNAKE_CASE. **All good.**
    *   Static typing is used in `EntityView.gd` and `SceneLoader.gd`. **All good.**
*   **Scene Naming**:
    *   `Main.tscn`, `Gameplay.tscn`, `EntityRoot.tscn` are PascalCase. **All good.**
*   **Node Naming in `EntityRoot.tscn`**:
    *   `EntityRoot`, `Visuals`, `Sprite`, `VFX`, `Collision`, `Audio`, `UI` are PascalCase. **All good.**
*   **Scene Interfaces**:
    *   `EntityView.gd` has `func setup(...)`. **All good.**
    *   `EntityView.gd` has `brain_target_position` and `_physics_process` for interpolation. **All good.**

Our current project perfectly adheres to the defined standards. This early commitment to consistency will greatly simplify future development.

### Summary

You have thoroughly reviewed and committed to the comprehensive naming conventions and scene interface standards for Sigilborne, as defined in TDD 00 and TDD 16. By understanding the rationale behind these rules and verifying their application in your existing codebase, you've reinforced the clarity, maintainability, and scalability of your hybrid Godot/C# project. The `setup()` function and visual interpolation mechanisms in `EntityView.gd` now serve as formal contracts between the C# Brain's data and the GDScript Body's presentation.

### Next Steps

The next chapter will delve into the **Animation Event Protocol**, a critical system for synchronizing complex animations in the GDScript Body with the precise logic in the C# Brain, ensuring that visual events (like a weapon's "hit frame") accurately trigger simulation events.