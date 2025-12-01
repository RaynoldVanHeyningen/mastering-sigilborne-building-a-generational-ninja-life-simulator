## Chapter 1.1: Project Setup for Godot 4.5 with C#

Welcome to the first chapter of building Sigilborne! This foundational step is crucial for establishing a robust development environment. We'll set up a new Godot 4.5 project with C# support, laying the groundwork for our hybrid "Brain & Body" architecture. A correct initial setup ensures smooth compilation, efficient workflow, and adherence to the architectural principles outlined in our Technical Design Document (TDD).

### 1. Downloading and Installing Godot 4.5

First, ensure you have the correct version of the Godot Engine.

1.  Navigate to the [official Godot Engine website](https://godotengine.org/download).
2.  Download the **.NET version** of Godot 4.5 for your operating system. This is crucial as it includes the necessary .NET runtime and C# tooling.
3.  Extract the downloaded archive to a convenient location on your machine. This executable is all you need to run Godot.

### 2. Creating a New Godot Project with C# Support

Now, let's create our project.

1.  Launch the Godot Engine executable.
2.  In the Project Manager, click the **"New Project"** button.
3.  **Project Name**: Enter `Sigilborne`.
4.  **Project Path**: Choose a suitable directory where you want your project files to reside. Godot will create a `Sigilborne` folder within this path.
5.  **Renderer**: For a 2D game like Sigilborne, `Compatibility` is often sufficient and offers broader hardware support. `Forward+` is also an option if you plan for more advanced 2D lighting effects, but it's not strictly necessary for our core mechanics. Choose `Compatibility` for now to keep things lean.
6.  **Language**: This is the most critical step for our hybrid architecture. Select **"C#"** from the dropdown menu.
7.  Click **"Create & Edit"**.

Godot will now create the project, including the necessary C# solution (`.sln`) and project (`.csproj`) files at the root of your `res://` directory.

### 3. Confirming Initial Project Structure

After creation, your project directory (visible in Godot's FileSystem dock) should look something like this:

```
res://
├── .godot/           # Godot's internal cache and configuration
├── project.godot     # Godot project settings
├── Sigilborne.csproj # C# project file (defines C# source, references, etc.)
├── Sigilborne.sln    # C# solution file (groups one or more .csproj files)
└── (other auto-generated files/folders)
```

The presence of `Sigilborne.csproj` and `Sigilborne.sln` confirms that C# support has been correctly integrated.

### 4. Installing .NET SDK (If Not Already Present)

Godot 4.5 with C# requires a compatible .NET SDK (Software Development Kit) to compile your C# code. Godot 4.x typically uses **.NET 6.0 SDK** or **.NET 7.0 SDK**.

1.  To check if you have it, open your system's command prompt or terminal and type:
    `dotnet --list-sdks`
2.  If you see an entry for `6.0.x` or `7.0.x`, you're likely good to go.
3.  If not, download and install the latest .NET 6.0 SDK from the [official .NET website](https://dotnet.microsoft.com/download).

### 5. Opening the C# Solution in an IDE

For a productive C# development experience, you'll need an Integrated Development Environment (IDE).

*   **Visual Studio Code (Recommended for Cross-Platform)**:
    1.  Install Visual Studio Code.
    2.  Install the **"C# Dev Kit"** extension from the VS Code Marketplace. This bundle includes C# language support, debugger, and project management features.
    3.  Open VS Code and go to `File > Open Folder...`. Select your `Sigilborne` project folder.
    4.  VS Code should automatically detect the `Sigilborne.sln` file and prompt you to load the project.
*   **Visual Studio (Windows/Mac)**:
    1.  Install Visual Studio (Community Edition is free). Ensure you select the ".NET desktop development" workload during installation.
    2.  Open Visual Studio and go to `File > Open > Project/Solution...`.
    3.  Navigate to your `Sigilborne` project folder and select `Sigilborne.sln`.

Once opened, your IDE should display the C# project structure, and you should be able to build it without errors (though there's no C# code yet to actually compile).

### 6. First C# Script and Build Test

Let's create our first C# script to confirm everything is working. This will also be the first step in organizing our project according to the TDD's `_Brain/` structure.

1.  In the Godot editor's FileSystem dock, right-click on the `res://` root.
2.  Select `New Folder` and name it `_Brain`.
3.  Inside `_Brain`, create another new folder named `Core`.
4.  Right-click on `res://_Brain/Core/`.
5.  Select `New Script...`.
6.  **Language**: Choose `C#`.
7.  **Class Name**: Enter `GameManager`.
8.  **Inherits**: Ensure it inherits `Node`.
9.  **Path**: Confirm the path is `res://_Brain/Core/GameManager.cs`.
10. Click `Create`.

Now, open `GameManager.cs` in your IDE or Godot's script editor. It should look like this (as per TDD 01):

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;

public partial class GameManager : Node
{
    // We'll add more to this class in later chapters.
    public override void _Ready()
    {
        GD.Print("GameManager (C#) is ready!");
    }
}
```

Next, let's attach this script to a scene to run it:

1.  In the Godot editor, go to `Scene > New Scene`.
2.  Add a `Node` as the root. Rename it `Main`.
3.  Save the scene as `res://Main.tscn`.
4.  Select the `Main` node in the Scene dock.
5.  In the Inspector dock, click the script icon next to "Script".
6.  Load `res://_Brain/Core/GameManager.cs`.
7.  Go to `Project > Project Settings... > Application > Run`.
8.  Set the `Main Scene` to `res://Main.tscn`.
9.  Close Project Settings.
10. Click the "Play Scene" button (the button with a film strip and play icon).

Godot should compile your C# project, and if successful, you'll see "GameManager (C#) is ready!" printed in the Output console at the bottom of the editor. This confirms that your C# environment is correctly set up and integrated with Godot.

### 7. Integrating TDD Directory Structure

With C# compilation confirmed, let's fully establish the project structure as defined in TDD 00 for better organization and adherence to the Brain/Body paradigm.

1.  In the Godot editor's FileSystem dock, if you haven't already:
    *   Create a new folder `res://_Body/`.
    *   Inside `res://_Body/`, create `Scenes/`, `Scripts/`, `Art/`, `Audio/`.
    *   Inside `res://_Body/Scripts/`, create `Core/`.
    *   Inside `res://_Brain/`, you already have `Core/`. Now add `Systems/`, `Data/`, `Utils/`.
    *   Create a new folder `res://_Shared/`.
    *   Inside `res://_Shared/`, create `Config/`.

Your `res://` directory should now visually reflect the TDD's structure:

```
res://
├── _Brain/                 # C# Source Code (The Simulation)
│   ├── Core/               # Main Loop, Event Bus, Time
│   │   └── GameManager.cs  # Our first C# script
│   ├── Systems/            # Magic, Combat, Ecology, Economy
│   ├── Data/               # Static Data (Items, Spells)
│   └── Utils/              # Math, Extensions
├── _Body/                  # Godot Assets (The Visuals)
│   ├── Scenes/             # .tscn files
│   ├── Scripts/            # .gd files (Visual logic only)
│   │   └── Core/           # Core GDScript
│   ├── Art/                # Sprites, Textures, Shaders
│   └── Audio/              # Banks, Streams
├── _Shared/                # Resources used by both (if any)
│   └── Config/             # Settings, Constants
├── Main.tscn               # Entry Point
├── .godot/
├── project.godot
├── Sigilborne.csproj
└── Sigilborne.sln
```

This completes the initial project setup. You now have a Godot 4.5 project with a fully functional C# backend, organized according to the TDD's core architectural principles.

### Next Steps

In the next chapter, we will begin implementing the core components of the `GameManager` in C#, establishing the Brain's main loop and essential systems.