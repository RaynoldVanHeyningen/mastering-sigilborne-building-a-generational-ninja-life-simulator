## Chapter 5.8: Weather & Environment - Global Environmental State (C#)

Our `BiologicalSystem` is now dynamically adjusting metabolism based on activity. The next step is to integrate environmental factors. This chapter focuses on fully implementing the **WeatherSystem** in the C# Brain to manage global and local weather states, and designing how environmental pressure (temperature, specific weather types) directly impacts `CoreStats` and biological processes, as specified in TDD 03.3 and GDD B14.7.

### 1. The Dynamic Nature of Sigilborne's Environment

The GDD (B14.7, B14.8) emphasizes that weather and temperature alter gameplay, affecting:
*   Visibility, stealth difficulty.
*   Chakra stability.
*   Movement.
*   Residue spread.
*   Sensing accuracy.
*   And, crucially, fatigue rate (which ties into metabolism).

Our `WeatherSystem` will be the authoritative source for all environmental conditions, providing a `GlobalWeatherState` that `BiologicalSystem` will use to modify entity `CoreStats`.

### 2. Enhancing `WeatherSystem.cs`

Our `WeatherSystem` currently exists as a placeholder. We'll expand it to:
*   Use a `Random` instance for more dynamic transitions.
*   Implement a more realistic (though still simple) cycle for `CurrentWeather`, `Temperature`, `WindDirection`, and `Humidity`.
*   Ensure it publishes `GlobalWeatherChangedEvent` for the Body to react.

Open `res://_Brain/Systems/Weather/WeatherSystem.cs`:

```csharp
// _Brain/Systems/Weather/WeatherSystem.cs
using Godot;
using System;
using Sigilborne.Core; // For EventBus, TimeSystem
using System.Linq; // For random weather selection

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
        private TimeSystem _timeSystem;
        private Random _rand; // New: Random instance for dynamic weather

        private GlobalWeatherState _currentGlobalWeather;
        private const float WEATHER_UPDATE_INTERVAL = 60.0f; // Update global weather every 60 game minutes (1 real minute)
        private float _weatherUpdateTimer;

        public WeatherSystem(EventBus eventBus, TimeSystem timeSystem, int worldSeed) // Add worldSeed for deterministic random
        {
            _eventBus = eventBus;
            _timeSystem = timeSystem;
            _rand = new Random(worldSeed + (int)timeSystem.CurrentGameTime); // Seed with world seed + current time

            // Initialize with default clear weather
            _currentGlobalWeather = new GlobalWeatherState(WeatherType.Clear, Vector2.Down, 20.0f, 0.5f, 0f);
            GD.Print($"WeatherSystem: Initialized. Current: {_currentGlobalWeather}");

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
        /// (TDD 03.3)
        /// </summary>
        private void UpdateGlobalWeather()
        {
            // --- Dynamic Weather Transition Logic ---
            WeatherType newWeather = _currentGlobalWeather.CurrentWeather;
            float newTemperature = _currentGlobalWeather.Temperature;
            float newHumidity = _currentGlobalWeather.Humidity;
            float newPrecipitation = _currentGlobalWeather.Precipitation;
            Vector2 newWindDirection = _currentGlobalWeather.WindDirection;

            // Simple chance to change weather
            if (_rand.NextDouble() < 0.3) // 30% chance to change every interval
            {
                // Select a random weather type (excluding None)
                WeatherType[] possibleWeathers = Enum.GetValues(typeof(WeatherType))
                                                    .Cast<WeatherType>()
                                                    .Where(w => w != WeatherType.None)
                                                    .ToArray();
                newWeather = possibleWeathers[_rand.Next(possibleWeathers.Length)];
            }

            // Adjust temperature based on current weather and time of day (conceptual)
            float baseTempChange = (float)(_rand.NextDouble() * 4 - 2); // +/- 2 C
            newTemperature = Mathf.Clamp(newTemperature + baseTempChange, -10f, 40f);

            // Adjust humidity and precipitation based on weather
            switch (newWeather)
            {
                case WeatherType.Clear:
                    newHumidity = Mathf.Lerp(newHumidity, 0.4f, 0.2f);
                    newPrecipitation = Mathf.Lerp(newPrecipitation, 0f, 0.2f);
                    break;
                case WeatherType.Rain:
                    newHumidity = Mathf.Lerp(newHumidity, 0.8f, 0.2f);
                    newPrecipitation = Mathf.Lerp(newPrecipitation, 0.7f, 0.2f);
                    break;
                case WeatherType.Storm:
                    newHumidity = Mathf.Lerp(newHumidity, 0.9f, 0.2f);
                    newPrecipitation = Mathf.Lerp(newPrecipitation, 1.0f, 0.2f);
                    newWindDirection = new Vector2((float)(_rand.NextDouble() * 2 - 1), (float)(_rand.NextDouble() * 2 - 1)).Normalized();
                    break;
                case WeatherType.Snow:
                    newTemperature = Mathf.Min(newTemperature, 5f); // Ensure cold
                    newHumidity = Mathf.Lerp(newHumidity, 0.6f, 0.2f);
                    newPrecipitation = Mathf.Lerp(newPrecipitation, 0.8f, 0.2f);
                    break;
                case WeatherType.Fog:
                    newHumidity = Mathf.Lerp(newHumidity, 0.9f, 0.2f);
                    newPrecipitation = Mathf.Lerp(newPrecipitation, 0.1f, 0.2f);
                    break;
                case WeatherType.AshFall: // C04 specific weather
                    newHumidity = Mathf.Lerp(newHumidity, 0.1f, 0.2f);
                    newPrecipitation = Mathf.Lerp(newPrecipitation, 0.5f, 0.2f);
                    break;
            }

            _currentGlobalWeather = new GlobalWeatherState(newWeather, newWindDirection, newTemperature, newHumidity, newPrecipitation);

            GD.Print($"WeatherSystem: Global weather updated to: {_currentGlobalWeather}");
            _eventBus.Publish(new GlobalWeatherChangedEvent { NewWeatherState = _currentGlobalWeather });
        }

        /// <summary>
        /// Returns the current global weather state.
        /// (TDD 03.3)
        /// For more complex worlds, this would take a position and return local weather,
        /// but for now, global applies everywhere.
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

We already added the `WeatherSystem` property and initialized it. We just need to ensure `WorldSeed` is passed to its constructor.

1.  Add a `WorldSeed` property to `GameManager`.
2.  Modify the `WeatherSystem` initialization in `InitializeSystems()` to pass `GameManager.Instance.WorldSeed`.

```csharp
// _Brain/Core/GameManager.cs
using Godot;
using System;
using Sigilborne.Core;
using Sigilborne.Entities;
using Sigilborne.Systems;
using Sigilborne.Systems.Biology;
using Sigilborne.Systems.Input;
using Sigilborne.Systems.Movement;
using Sigilborne.Systems.Physics;
using Sigilborne.Systems.Magic;
using Sigilborne.Systems.Weather;
using Sigilborne.Utils;
using System.Linq;
using Sigilborne.Systems.Magic.Components;

public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }
    
    // ... (existing system properties) ...
    public WeatherSystem Weather { get; private set; }
    public int WorldSeed { get; private set; } = 12345; // New: Store world seed

    public override void _Ready()
    {
        // ... (existing Instance check and InitializeSystems call) ...
        // ... (existing test entity/job code) ...
        // ... (existing glyph and magic system tests) ...
        
        // --- Test Damage & Recovery Pipeline ---
        // ... (existing damage tests) ...
        // --- End Testing Damage & Recovery Pipeline ---
    }

    public override void _PhysicsProcess(double delta)
    {
        Input.ProcessInputBuffer(); 
        Time.Tick(delta);
        World.Tick(delta);
        Movement.Tick(delta);
        Magic.Tick(delta);
        Casting.Tick(delta);
        Weather.Tick(delta); // Call WeatherSystem's tick method
        Biology.Tick(delta);
        Events.FlushCommands();
    }

    private void InitializeSystems()
    {
        // ... (existing system initializations up to PhysicsSystem) ...

        // ... (existing magic system initializations) ...
        
        // Initialize WeatherSystem BEFORE BiologicalSystem, pass WorldSeed
        Weather = new WeatherSystem(Events, Time, WorldSeed); // Pass WorldSeed
        GD.Print("  - WeatherSystem initialized.");

        // Initialize BiologicalSystem, passing WeatherSystem
        Biology = new BiologicalSystem(Entities, Events, Transforms, Movement, Weather, StatusEffects);
        GD.Print("  - BiologicalSystem initialized.");

        // Initialize PlayerStatSystem, passing BiologicalSystem
        PlayerStats = new PlayerStatSystem(Events, Entities, BiologicalSystem);
        GD.Print("  - PlayerStatSystem initialized.");

        World = new WorldSimulation();
        GD.Print("  - WorldSimulation initialized.");
    }
}
```

### 4. Enhancing `BiologicalSystem.cs` with Full Environmental Pressure

Now, `BiologicalSystem` will fully leverage the `WeatherSystem`'s data in `ProcessBioTick` to apply environmental pressure to `CoreStats`.

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
using Sigilborne.Systems.Weather;
using Sigilborne.Systems.StatusEffects;

namespace Sigilborne.Systems.Biology
{
    public class BiologicalSystem
    {
        // ... (existing fields) ...
        private WeatherSystem _weatherSystem;
        private StatusEffectSystem _statusEffectSystem;

        public BiologicalSystem(EntityManager entityManager, EventBus eventBus, TransformSystem transformSystem, MovementSystem movementSystem, WeatherSystem weatherSystem, StatusEffectSystem statusEffectSystem)
        {
            _entityManager = entityManager;
            _eventBus = eventBus;
            _transformSystem = transformSystem;
            _movementSystem = movementSystem;
            _weatherSystem = weatherSystem;
            _statusEffectSystem = statusEffectSystem;

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

                if (!_entityManager.IsValid(id) || coreStats.Health <= 0) continue;

                float activityMultiplier = GetActivityLevel(id);
                
                // --- Get Weather Multipliers (TDD 03.3) ---
                float weatherMetabolismMultiplier = 1.0f; // Influences hunger/thirst decay
                float weatherChakraStabilityPenalty = 0.0f; // Influences chakra stability (GDD B14.7)

                GlobalWeatherState currentGlobalWeather = _weatherSystem.GetCurrentGlobalWeather();
                float ambientTemperature = currentGlobalWeather.Temperature;

                // Metabolism based on temperature deviation (GDD B14.8)
                float tempDiff = Mathf.Abs(ambientTemperature - coreStats.NormalBodyTemp);
                if (tempDiff > 5f)
                {
                    weatherMetabolismMultiplier += tempDiff * 0.02f; // 2% increase per degree of deviation > 5 deg
                }
                if (currentGlobalWeather.Precipitation > 0.5f) // Rain/snow increases energy expenditure
                {
                    weatherMetabolismMultiplier += 0.1f;
                }

                // TDD 03.3: AshFall (C04) increases Chakra instability (GDD B14.7)
                if (currentGlobalWeather.CurrentWeather == WeatherSystem.WeatherType.AshFall)
                {
                    weatherChakraStabilityPenalty += 1.0f; // Significant penalty
                }
                // --- END Weather Multipliers ---

                // Hunger decay
                coreStats.Hunger -= 1.0f * activityMultiplier * weatherMetabolismMultiplier;
                coreStats.Hunger = Mathf.Max(0, coreStats.Hunger);
                
                // Thirst decay
                coreStats.Thirst -= 1.5f * activityMultiplier * weatherMetabolismMultiplier;
                coreStats.Thirst = Mathf.Max(0, coreStats.Thirst);

                // --- Body temperature adjustment (TDD 03.2) ---
                // Slowly drift body temp towards ambient temp
                coreStats.BodyTemp = Mathf.Lerp(coreStats.BodyTemp, ambientTemperature, 0.1f * BIO_TICK_RATE);
                // Apply penalties for extreme body temp (GDD B14.8)
                if (Mathf.Abs(coreStats.BodyTemp - coreStats.NormalBodyTemp) > 10f)
                {
                    // Apply fatigue, stat penalties, or even health damage if severe.
                    // coreStats.Stamina -= 1.0f * BIO_TICK_RATE; // Example: lose stamina from extreme temp
                }

                // --- Regeneration (Health, Stamina, Chakra, Stability) ---
                float regenRateMultiplier = 1.0f;
                if (activityMultiplier > 1.0f) regenRateMultiplier = 0.5f;
                if (activityMultiplier > 4.0f) regenRateMultiplier = 0.0f;

                // Impair regeneration from status effects
                if (_statusEffectSystem.HasEffectOfType(id, EffectType.ChakraDrain) || _statusEffectSystem.HasEffectOfType(id, EffectType.StabilityDrain))
                {
                    regenRateMultiplier *= 0.1f;
                }
                if (_statusEffectSystem.HasEffectOfType(id, EffectType.ControlImpair))
                {
                    regenRateMultiplier *= 0.5f;
                }
                if (_statusEffectSystem.HasEffectOfType(id, EffectType.DamageOverTime))
                {
                    regenRateMultiplier *= 0.2f;
                }

                // Health Regen
                if (coreStats.Hunger > 10 && coreStats.Thirst > 10 && coreStats.Health < coreStats.MaxHealth)
                {
                    coreStats.Health += coreStats.HealthRegenRate * regenRateMultiplier * BIO_TICK_RATE;
                    coreStats.Health = Mathf.Min(coreStats.MaxHealth, coreStats.Health);
                }

                // Stamina Regen
                if (coreStats.Stamina < coreStats.MaxStamina)
                {
                    coreStats.Stamina += coreStats.StaminaRegenRate * regenRateMultiplier * BIO_TICK_RATE;
                    coreStats.Stamina = Mathf.Min(coreStats.MaxStamina, coreStats.Stamina);
                }

                // Chakra Regen
                if (coreStats.Chakra < coreStats.MaxChakra)
                {
                    coreStats.Chakra += coreStats.ChakraRegenRate * regenRateMultiplier * BIO_TICK_RATE;
                    coreStats.Chakra = Mathf.Min(coreStats.MaxChakra, coreStats.Chakra);
                }

                // Stability Regen (GDD B03.6)
                // Apply weather penalty here
                coreStats.Stability += (coreStats.StabilityRegenRate * regenRateMultiplier - weatherChakraStabilityPenalty) * BIO_TICK_RATE;
                coreStats.Stability = Mathf.Min(coreStats.MaxStability, coreStats.Stability);
                coreStats.Stability = Mathf.Max(0, coreStats.Stability); // Clamp at 0 for now


                // --- Apply consequences of low stats ---
                // ... (existing low stat consequences) ...

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

### 5. Testing Weather and Environmental Pressure

1.  Save all C# and GDScript files.
2.  Run `Main.tscn`.
3.  Load `Gameplay.tscn`.
4.  Observe the Weather and Temperature labels on the HUD.
    *   Every 1 real minute (60 game minutes), the weather should dynamically change (e.g., Clear -> Rain -> Fog -> Clear).
    *   Temperature will fluctuate.
    *   When the weather changes (e.g., to Rain or if temperature deviates significantly), you should observe Hunger and Thirst decay rates increase due to `weatherMetabolismMultiplier`.
    *   `BodyTemp` will slowly drift towards `AmbientTemperature`.
    *   (Optional Test): Use the debug console to force `AshFall` weather (e.g., `/weather set AshFall` - this command needs to be added to `DebugCommandSystem`). Observe Chakra stability decreasing due to `weatherChakraStabilityPenalty`.

This confirms that the `WeatherSystem` is active, dynamically updating the global environment, and `BiologicalSystem` is correctly applying environmental pressure to `CoreStats`'s metabolism and regeneration, including chakra stability.

### Summary

You have successfully implemented the **Weather & Environment** system, fully enhancing `WeatherSystem` in the C# Brain to manage dynamic global weather states. By integrating this system with `BiologicalSystem`, you've ensured that environmental pressures (temperature, precipitation, and specific weather types like AshFall) directly impact entity `CoreStats`'s metabolism, body temperature, and chakra stability. This crucial step strictly adheres to TDD 03.3 and GDD B14.7's specifications, creating a living, reactive environment that influences character well-being and strategic gameplay.

### Next Steps

This concludes **Module 5: Chakra & Life Systems**. We will now move on to **Module 6: Combat & Tactical Engagement**, starting with **Physics Layer - Hitbox & Hurtbox Detection (GDScript)**, where we will set up Godot's `Area2D` nodes for collision detection in combat.