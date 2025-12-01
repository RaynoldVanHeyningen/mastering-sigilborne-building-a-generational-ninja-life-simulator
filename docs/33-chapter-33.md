## Chapter 5.4: Weather & Environment - Global Environmental State (C#)

Our `BiologicalSystem` is now dynamically adjusting metabolism based on activity. The next step is to integrate environmental factors. This chapter focuses on fully implementing the **WeatherSystem** in the C# Brain to manage global and local weather states, and designing how environmental pressure (temperature, specific weather types) directly impacts `CoreStats` and biological processes, as specified in TDD 03.3 and GDD B14.7.

### 1. The Dynamic Nature of Sigilborne's Environment

The GDD (B14.7, B14.8) emphasizes that weather and temperature alter gameplay, affecting:
*   Visibility, stealth difficulty.
*   Chakra stability.
*   Movement.
*   Residue spread.
*   Sensing accuracy.
*   And, crucially, fatigue rate (which ties into metabolism).

Our `WeatherSystem` will be the authoritative source for all environmental conditions.

### 2. Enhancing `WeatherSystem.cs`

Our `WeatherSystem` currently exists as a placeholder. We'll expand it to:
*   Track `CurrentWeather`, `Temperature`, `WindDirection`, and `Humidity` globally.
*   Implement a simple update loop for weather transitions.
*   Provide methods for other systems to query environmental conditions.

Open `res://_Brain/Systems/Weather/WeatherSystem.cs`:

```csharp
// _Brain/Systems/Weather/WeatherSystem.cs
using Godot;
using System;
using Sigilborne.Core; // For EventBus, TimeSystem

namespace Sigilborne.Systems.Weather
{
    /// <summary>
    /// Stores the global weather state.
    /// (TDD 03.3)
    /// </summary>
    public struct GlobalWeatherState
    {
        public WeatherType CurrentWeather;
        public Vector2 WindDirection; // Normalized vector
        public float Temperature;     // Ambient Celsius
        public float Humidity;        // 0-1 (0=dry, 1=humid)
        public float Precipitation;   // 0-1 (0=none, 1=heavy)

        public GlobalWeatherState(WeatherType currentWeather, Vector2 windDirection, float temperature, float humidity, float precipitation)
        {
            CurrentWeather = currentWeather;
            WindDirection = windDirection;
            Temperature = temperature;
            Humidity = humidity;
            Precipitation = precipitation;
        }

        public override string ToString()
        {
            return $"Weather: {CurrentWeather}, Temp: {Temperature:F1}°C, Wind: {WindDirection}, Hum: {Humidity:P0}, Precip: {Precipitation:P0}";
        }
    }

    /// <summary>
    /// Manages the global and local weather states, and their transitions.
    /// (TDD 03.3)
    /// </summary>
    public class WeatherSystem
    {
        private EventBus _eventBus;
        private TimeSystem _timeSystem; // To react to day/night, seasons (conceptual)

        private GlobalWeatherState _currentGlobalWeather;
        private const float WEATHER_UPDATE_INTERVAL = 60.0f; // Update global weather every 60 game minutes (1 real minute)
        private float _weatherUpdateTimer;

        public WeatherSystem(EventBus eventBus, TimeSystem timeSystem)
        {
            _eventBus = eventBus;
            _timeSystem = timeSystem;
            
            // Initialize with default clear weather
            _currentGlobalWeather = new GlobalWeatherState(WeatherType.Clear, Vector2.Down, 20.0f, 0.5f, 0f);
            GD.Print($"WeatherSystem: Initialized. Current: {_currentGlobalWeather}");

            // Publish initial weather state for Body (VFX, UI)
            _eventBus.Publish(new GlobalWeatherChangedEvent { NewWeatherState = _currentGlobalWeather });
        }

        /// <summary>
        /// Main update loop for the WeatherSystem.
        /// Called during GameManager._PhysicsProcess (Phase 2).
        /// </summary>
        public void Tick(double delta)
        {
            _weatherUpdateTimer += (float)delta;
            if (_weatherUpdateTimer >= WEATHER_UPDATE_INTERVAL)
            {
                UpdateGlobalWeather();
                _weatherUpdateTimer = 0;
            }
        }

        /// <summary>
        /// Simulates transitions in global weather.
        /// For now, a simple cycle. Later, this would be based on biome, season, anomalies.
        /// </summary>
        private void UpdateGlobalWeather()
        {
            // Simple sequential transition for testing
            WeatherType nextWeather = _currentGlobalWeather.CurrentWeather switch
            {
                WeatherType.Clear => WeatherType.Rain,
                WeatherType.Rain => WeatherType.Fog,
                WeatherType.Fog => WeatherType.Clear,
                _ => WeatherType.Clear, // Fallback
            };

            // Randomize temperature slightly
            Random rand = new Random(GameManager.Instance.WorldSeed + (int)_timeSystem.CurrentGameTime); // Use world seed and time for determinism
            float newTemperature = Mathf.Clamp(_currentGlobalWeather.Temperature + (float)(rand.NextDouble() * 5 - 2.5), -10f, 40f); // +/- 2.5 C
            
            _currentGlobalWeather = new GlobalWeatherState(nextWeather, Vector2.Down, newTemperature, 0.5f, nextWeather == WeatherType.Rain ? 0.7f : 0f); // Update precipitation for rain

            GD.Print($"WeatherSystem: Global weather updated to: {_currentGlobalWeather}");
            _eventBus.Publish(new GlobalWeatherChangedEvent { NewWeatherState = _currentGlobalWeather });
        }

        /// <summary>
        /// Returns the current global weather state.
        /// For more complex worlds, this would take a position and return local weather.
        /// </summary>
        public GlobalWeatherState GetCurrentGlobalWeather()
        {
            return _currentGlobalWeather;
        }
        
        // --- Helper Events for Body Sync ---
        public struct GlobalWeatherChangedEvent { public GlobalWeatherState NewWeatherState; }
    }
}
```

### 3. Integrating `WeatherSystem` into `GameManager`

We already added the `WeatherSystem` property and initialized it. We just need to ensure `TimeSystem` is passed to its constructor.

```csharp
// _Brain/Core/GameManager.cs (inside InitializeSystems)
// ...
        // Initialize WeatherSystem BEFORE BiologicalSystem
        Weather = new WeatherSystem(Events, Time); // Pass TimeSystem
        GD.Print("  - WeatherSystem initialized.");
// ...
```

### 4. Enhancing `BiologicalSystem.cs` with Environmental Pressure

Now, `BiologicalSystem` will use `WeatherSystem` to get the `WeatherMultiplier` and apply environmental pressure to `CoreStats`.

Open `res://_Brain/Systems/Biology/BiologicalSystem.cs`:

```csharp
// _Brain/Systems/Biology/BiologicalSystem.cs
using Godot;
using System;
using System.Collections.Generic;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems.Physics;
using Sigilborne.Systems.Movement;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.Weather; // Ensure this is present

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        // ... (existing fields) ...
        private WeatherSystem _weatherSystem;

        public BiologicalSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, MovementSystem movementSystem, WeatherSystem weatherSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _movementSystem = movementSystem;
            _weatherSystem = weatherSystem;

            _eventBus.OnEntitySpawned += OnEntitySpawned;
            _eventBus.OnEntityDespawned += OnEntityDespawned;

            GD.Print("BiologicalSystem: Initialized.");
        }

        // ... (OnEntitySpawned, OnEntityDespawned, Tick methods) ...

        private void ProcessBioTick()
        {
            foreach (var kvp in _entityCoreStats)
            {
                EntityID id = kvp.Key;
                ref CoreStats coreStats = ref _entityCoreStats.GetValueRef(id);

                if (!_entityManager.IsValid(id)) continue;

                float activityMultiplier = GetActivityLevel(id);
                
                // --- NEW: Get weather multiplier and apply environmental pressure (TDD 03.3) ---
                float weatherMetabolismMultiplier = 1.0f;
                float ambientTemperature = coreStats.NormalBodyTemp; // Default to normal temp
                
                if (_transformSystem.TryGetTransform(id, out TransformComponent transform))
                {
                    // For now, assume global weather applies everywhere. Later, this would be local.
                    GlobalWeatherState currentGlobalWeather = _weatherSystem.GetCurrentGlobalWeather();
                    ambientTemperature = currentGlobalWeather.Temperature; // Get ambient temp

                    // Colder/hotter weather increases metabolism (GDD B14.8)
                    float tempDiff = Mathf.Abs(ambientTemperature - coreStats.NormalBodyTemp);
                    if (tempDiff > 5f) // If temperature deviates by more than 5 degrees
                    {
                        weatherMetabolismMultiplier += tempDiff * 0.05f; // 5% increase per 5 degrees diff
                    }
                    if (currentGlobalWeather.Precipitation > 0.5f) // Rain/snow increases energy expenditure
                    {
                        weatherMetabolismMultiplier += 0.1f;
                    }

                    // TDD 03.3: AshFall (C04) increases Chakra instability (conceptual)
                    // if (currentGlobalWeather.CurrentWeather == WeatherSystem.WeatherType.AshFall)
                    // {
                    //     // Apply chakra stability penalty here
                    //     coreStats.Stability -= 0.5f * BIO_TICK_RATE;
                    // }
                }
                // --- END NEW ---

                // Hunger decay (TDD 03.2)
                coreStats.Hunger -= 1.0f * activityMultiplier * weatherMetabolismMultiplier;
                coreStats.Hunger = Mathf.Max(0, coreStats.Hunger);
                
                // Thirst decay (TDD 03.2)
                coreStats.Thirst -= 1.5f * activityMultiplier * weatherMetabolismMultiplier;
                coreStats.Thirst = Mathf.Max(0, coreStats.Thirst);

                // --- Body temperature adjustment (TDD 03.2) ---
                // Slowly drift body temp towards ambient temp
                coreStats.BodyTemp = Mathf.Lerp(coreStats.BodyTemp, ambientTemperature, 0.1f * BIO_TICK_RATE);
                // Apply penalties for extreme body temp (GDD B14.8)
                if (Mathf.Abs(coreStats.BodyTemp - coreStats.NormalBodyTemp) > 10f)
                {
                    // GD.Print($"BiologicalSystem: {id} has extreme body temperature ({coreStats.BodyTemp:F1}°C)!");
                    // Apply fatigue, stat penalties, or even health damage if severe.
                }


                // --- Regeneration (Health, Stamina, Chakra, Stability) ---
                // ... (existing regen logic) ...

                // ... (Apply consequences of low stats) ...

                // Publish updates for the Body (e.g., UI for player)
                if (id == _entityManager.GetPlayerEntityID())
                {
                    _eventBus.Publish(new PlayerCoreStatsChangedEvent { PlayerID = id, NewCoreStats = coreStats });
                }
            }
        }
        // ... (GetActivityLevel, GetCoreStatsRef, TryGetCoreStats, Helper Events methods) ...
    }
}
```

### 5. Displaying Weather and Updated BioState in the Body (GDScript UI)

Let's update our `HUDController.gd` to display the current weather and the player's Body Temperature.

1.  Open `res://_Body/Scenes/UI/HUD.tscn` and modify it:
    *   Add a `Label` node for `WeatherLabel`.
    *   Add a `Label` node for `TemperatureLabel`.
    *   Arrange them (e.g., within `VBoxNeeds` or a new container).

2.  Open `res://_Body/Scripts/UI/HUDController.gd` and modify it:

```gdscript
# _Body/Scripts/UI/HUDController.gd
class_name HUDController extends CanvasLayer

@onready var health_label: Label = $MainHBox/VBoxStats/HealthLabel
@onready var chakra_label: Label = $MainHBox/VBoxStats/ChakraLabel
@onready var stamina_label: Label = $MainHBox/VBoxStats/StaminaLabel
@onready var stability_label: Label = $MainHBox/VBoxStats/StabilityLabel
@onready var hunger_label: Label = $MainHBox/VBoxNeeds/HungerLabel
@onready var thirst_label: Label = $MainHBox/VBoxNeeds/ThirstLabel
@onready var weather_label: Label = $MainHBox/VBoxNeeds/WeatherLabel # New
@onready var temperature_label: Label = $MainHBox/VBoxNeeds/TemperatureLabel # New

var player_entity_id: int = -1

func _ready():
    GD.print("HUDController: Initialized. Connecting to C# PlayerStatSystem & BiologicalSystem events.")
    if GameManager.Instance != null and GameManager.Instance.Events != null:
        GameManager.Instance.Events.OnPlayerHealthChanged.connect(Callable(self, "_on_player_health_changed"))
        GameManager.Instance.Events.OnPlayerChakraChanged.connect(Callable(self, "_on_player_chakra_changed"))
        GameManager.Instance.Events.OnPlayerStaminaChanged.connect(Callable(self, "_on_player_stamina_changed"))
        GameManager.Instance.Events.OnPlayerStabilityChanged.connect(Callable(self, "_on_player_stability_changed"))
        GameManager.Instance.Events.OnPlayerDied.connect(Callable(self, "_on_player_died"))
        GameManager.Instance.Events.OnPlayerCoreStatsChanged.connect(Callable(self, "_on_player_core_stats_changed"))
        GameManager.Instance.Events.OnEntitySpawned.connect(Callable(self, "_on_entity_spawned"))
        GameManager.Instance.Events.OnGlobalWeatherChanged.connect(Callable(self, "_on_global_weather_changed")) # New
        GD.print("HUDController: Successfully connected to C# PlayerStatSystem & BiologicalSystem events.")
    else:
        push_error("HUDController: GameManager or EventBus not ready! Cannot connect C# events.")

func _on_entity_spawned(id: int, type: int, definition_id: String, initial_position: Vector2, initial_rotation: float) -> void:
    if type == 0:
        player_entity_id = id
        GD.print("HUDController: Detected player entity with ID: %s" % player_entity_id)

func _on_player_health_changed(id: int, new_value: float, max_value: float) -> void:
    if id == player_entity_id: health_label.text = "HP: %s/%s" % [int(new_value), int(max_value)]
func _on_player_chakra_changed(id: int, new_value: float, max_value: float) -> void:
    if id == player_entity_id: chakra_label.text = "CHA: %s/%s" % [int(new_value), int(max_value)]
func _on_player_stamina_changed(id: int, new_value: float, max_value: float) -> void:
    if id == player_entity_id: stamina_label.text = "STA: %s/%s" % [int(new_value), int(max_value)]
func _on_player_stability_changed(id: int, new_value: float, max_value: float) -> void:
    if id == player_entity_id: stability_label.text = "STAB: %s/%s" % [int(new_value), int(max_value)]

## Handler for C# PlayerCoreStatsChangedEvent (for Hunger, Thirst, Temp)
func _on_player_core_stats_changed(id: int, new_core_stats: Variant) -> void:
    if id == player_entity_id:
        hunger_label.text = "HUN: %s" % int(new_core_stats.Hunger)
        thirst_label.text = "THI: %s" % int(new_core_stats.Thirst)
        temperature_label.text = "TEMP: %s°C" % int(new_core_stats.BodyTemp) # Update temperature

## Handler for C# OnGlobalWeatherChanged event.
func _on_global_weather_changed(new_weather_state: Variant) -> void: # Variant to receive C# struct
    weather_label.text = "WTH: %s" % new_weather_state.CurrentWeather # Access enum name directly
    temperature_label.text = "TEMP: %s°C" % int(new_weather_state.Temperature) # Update temperature
    GD.print("HUDController: Global Weather Updated to %s" % new_weather_state.CurrentWeather)


func _on_player_died(id: int) -> void:
    if id == player_entity_id:
        health_label.text = "HP: DEAD"
        chakra_label.text = "CHA: DEAD"
        stamina_label.text = "STA: DEAD"
        stability_label.text = "STAB: DEAD"
        hunger_label.text = "HUN: DEAD"
        thirst_label.text = "THI: DEAD"
        temperature_label.text = "TEMP: N/A"
        weather_label.text = "WTH: N/A"
        GD.print("HUDController: Player %s has died visually!" % id)
```

### 6. Testing Weather and Environmental Pressure

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the new Weather and Temperature labels on the HUD.
    *   The `WeatherSystem` will update every 60 game minutes (1 real minute). You should see the weather cycle through Clear, Rain, Fog.
    *   Temperature will fluctuate.
    *   When the weather changes (especially to Rain), you should observe Hunger and Thirst decay rates increase slightly due to `weatherMetabolismMultiplier`.
    *   `BodyTemp` will slowly drift towards `AmbientTemperature`.

This confirms that the `WeatherSystem` is active, updating the global environment, and `BiologicalSystem` is correctly applying environmental pressure to `CoreStats`.

### Summary

You have successfully implemented the **WeatherSystem** in the C# Brain, establishing it as the authoritative source for global environmental conditions. By designing its update loop and integrating it with `BiologicalSystem`, you've ensured that environmental pressures (like temperature and precipitation) dynamically impact `CoreStats` and metabolic rates, strictly adhering to TDD 03.3's specifications. Furthermore, `HUDController.gd` now reactively displays these environmental factors, enhancing player immersion and strategic awareness.

### Next Steps

The next chapter will focus on **Damage & Recovery Pipeline**, detailing how entities take damage, how mitigation is applied, and the process of generating `StatusEffect`s (like Bleed or Fracture) from injuries.