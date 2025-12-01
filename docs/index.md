# Mastering Sigilborne: Building a Generational Ninja Life Simulator

This course guides you through the technical and design journey of creating "Sigilborne," a unique ninja-life simulator where player characters are ephemeral, but the world and its history are persistent across generations. We will build a deeply simulated, procedurally generated magical world, focusing on systemic gameplay, emergent narratives, and a composition-first, production-grade architecture using Godot 4.5 and C#.

---

## Table of Contents

### Module 1: Core Architecture - The Brain & Body Paradigm
This module lays the foundational architectural principles, establishing the strict separation of simulation logic (C# Brain) from visual presentation (GDScript Body). You'll learn how to set up this hybrid environment for optimal performance and maintainability.

*   **Chapter 1.1: Project Setup for Godot 4.5 with C#**
    *   Initialize a new Godot 4.5 project with C# support.
    *   Configure the C# solution and build pipeline.
*   **Chapter 1.2: The Brain (C#) - Headless Simulation Layer**
    *   Understand the role of C# for heavy simulation, data, and AI.
    *   Implement the `GameManager` as the C# entry point and system bootstrapper.
*   **Chapter 1.3: The Body (GDScript) - Reactive Presentation Layer**
    *   Understand the role of GDScript for rendering, UI, and animation.
    *   Implement `SceneLoader` for managing high-level game state transitions.
*   **Chapter 1.4: Directory Structure - Enforcing Separation of Concerns**
    *   Establish the `_Brain/`, `_Body/`, and `_Shared/` directory structure.
    *   Detail naming conventions for C# and GDScript files and assets.
*   **Chapter 1.5: Entity Model - Lightweight ECS-lite in C#**
    *   Define the `EntityID` struct for unique entity identification.
    *   Implement the `EntityManager` for entity creation, destruction, and validation.
*   **Chapter 1.6: Component Architecture - Composition over Inheritance**
    *   Design `TransformComponent`, `TypeComponent`, and `StateComponent` as core data structs.
    *   Understand component storage patterns (SoA vs. Dictionary).
*   **Chapter 1.7: Job System & Concurrency - Multithreaded Processing**
    *   Implement the `JobManager` with worker thread pools and priority queues.
    *   Design `IJob` interface and `EcologyUpdateJob` example.
*   **Chapter 1.8: Thread Safety - Command Buffers & Double Buffering**
    *   Enforce the "No-Node" rule for jobs.
    *   Implement `CommandBuffer` for main thread synchronization.
    *   Understand double buffering for concurrent read/write operations.
*   **Chapter 1.9: Scene Composition - Standardizing Godot Scenes**
    *   Define the `EntityRoot` hierarchy for all active entity scenes.
    *   Establish `EntityView.gd` as the sole script communicating with the Brain.
*   **Chapter 1.10: Naming Conventions & Scene Interfaces**
    *   Standardize naming for scenes, scripts, and nodes.
    *   Implement `setup(id: int)` function for scene initialization.
*   **Chapter 1.11: Animation Event Protocol - Syncing Brain & Body**
    *   Define the `anim_event(type: String, payload: Dictionary)` signal standard.
    *   Implement standard event types like "hit_frame" and "cast_release".
*   **Chapter 1.12: Interop Layer - C# to GDScript Communication**
    *   Implement the EventBus using C# `Action<T>` delegates for Brain-to-Body signals.
    *   Detail thread-safe event batching and error handling.
*   **Chapter 1.13: Global State Management - Data Ownership**
    *   Establish the Brain as the sole owner of authoritative game state.
    *   Detail how the Body only displays interpolated visual states.
*   **Chapter 1.14: Debugging Tools - Console & State Inspector**
    *   Implement a Quake-style debug console for runtime command execution.
    *   Develop a runtime state inspector for entity data visualization.
*   **Chapter 1.15: Main Loop Execution Order - Tick vs. Frame**
    *   Detail the `_Process` (GDScript) and `_PhysicsProcess` (C#) execution flow.
    *   Explain the roles of Input Processing, Simulation Tick, and Visual Interpolation.

### Module 2: Player Input & Core Movement
This module focuses on how player input is captured and translated into character actions, including standard movement, unique "Shift-Sliding," and the hybrid physics system that ensures smooth, responsive control.

*   **Chapter 2.1: Input Manager - Capturing Raw Input (GDScript)**
    *   Implement `InputManager.gd` to capture hardware events.
    *   Normalize input (e.g., WASD to `Vector2`).
*   **Chapter 2.2: Input Frame - Snapshot for the Brain (C#)**
    *   Define the `PlayerInputFrame` struct for sending input snapshots to C#.
    *   Buffer input for processing in the Brain's tick.
*   **Chapter 2.3: Standard Movement Logic - Velocity & Friction (C#)**
    *   Implement basic `MoveVector` to `Velocity` calculations.
    *   Apply friction for snappy movement stops.
*   **Chapter 2.4: Shift-Sliding Mechanic - Direction Lock (C#)**
    *   Implement the "Shift-Sliding" logic to maintain movement direction.
    *   Manage `LastMoveVector` for persistent directional input.
*   **Chapter 2.5: Physics Layer - Hybrid Movement Pipeline**
    *   Understand the Brain's role in calculating `TargetVelocity` and `ProposedPosition`.
    *   Understand the Body's role in `move_and_slide()` for collision resolution.
*   **Chapter 2.6: Physics Reconciliation - Syncing Brain & Body Positions**
    *   Implement the reconciliation step where the Brain overwrites its position with the Body's `FinalPosition`.
    *   Explain why Godot handles collision resolution locally.
*   **Chapter 2.7: Interactions & Pushes - Knockback & Entity-vs-Entity**
    *   Implement knockback by modifying the Brain's `Velocity`.
    *   Leverage `CharacterBody2D` for automatic soft collision pushes.
*   **Chapter 2.8: Physics Layers - Standardizing Collision Interactions**
    *   Define physics layers for World, Player, Enemy, Projectile, Interaction, and Hitbox.
    *   Configure collision masks for proper interaction.

### Module 3: The Glyph System - Language of Ninjutsu
Dive deep into Sigilborne's unique glyph-based magic system. This module covers the procedural generation of glyph meanings, player discovery mechanics, and the underlying logic that makes every world's magic feel unique.

*   **Chapter 3.1: Glyph Database - Concepts vs. Symbols (C#)**
    *   Separate abstract `Concepts` (Fire, Water) from `Symbols` (Triangle, Circle).
    *   Store procedural mappings in `WorldData.GlyphMap`.
*   **Chapter 3.2: Glyph Generation Logic - World Seed Mapping (C#)**
    *   Implement the world-seed-based shuffling and assignment of Symbols to Concepts.
    *   Ensure unique glyph meanings per playthrough.
*   **Chapter 3.3: Glyph Discovery System - Player Knowledge State (C#)**
    *   Track player `KnowledgeState` for each glyph (Hidden, Seen, Known).
    *   Implement the feedback loop for revealing glyph meanings upon successful cast.
*   **Chapter 3.4: Glyph Experimentation - Trial & Error Casting (C#)**
    *   Process player input to attempt combos with unknown glyphs.
    *   Provide feedback for successful discovery or fizzle/stability damage.
*   **Chapter 3.5: Subtypes & Modifiers - Procedural Nuance for Glyphs (C#)**
    *   Implement procedural generation of `Subtypes` for each glyph variant.
    *   Assign float modifiers to define unique behaviors (e.g., Blue Flame vs. Red Flame).
*   **Chapter 3.6: Glyph Acquisition - Teachers & Scrolls (C#)**
    *   Design `Scroll` items to grant `Known` status for glyphs.
    *   Implement `Teacher` NPCs that demonstrate combos, enabling player learning.
*   **Chapter 3.7: Glyph Stability & Category Resonance (GDD B01)**
    *   Define per-glyph `Stability Rating` and its effects on casting.
    *   Implement `Category Resonance` logic for environmental interactions (e.g., Bloom in forests).
*   **Chapter 3.8: World Mapping Logic - Glyph to Subtype Table (C#)**
    *   Generate the global `glyph -> category -> subtype` table per world.
    *   Implement environmental modulation of subtype behavior.
*   **Chapter 3.9: Glyph Interaction Rules - High-Level Principles (GDD B01)**
    *   Implement glyph interactions based on category relationships, subtypes, and sequence order.
    *   Ensure world-specific variations via subtypes.
*   **Chapter 3.10: Forbidden Glyphs & Glyph Loss (GDD B01)**
    *   Define `Forbidden Glyphs` with inherent instability and danger.
    *   Implement mechanics for glyph loss or corruption due to residue, miscasts, or bloodline mutations.

### Module 4: Combos & Casting - The Art of Ninjutsu
This module focuses on the dynamic combo system, where glyph sequences transform into powerful techniques. You'll build the input pipeline, spell resolution, and the mastery system that allows players to discover and evolve their unique ninjutsu.

*   **Chapter 4.1: Input Buffer - Storing Glyph Sequences (C#)**
    *   Implement a circular `InputBuffer` to store `InputFrame` structs with timestamps.
    *   Efficiently retrieve recent inputs for combo detection.
*   **Chapter 4.2: Combo Resolver - Trie Structure for Spell Detection (C#)**
    *   Implement a Prefix Tree (Trie) to efficiently detect glyph sequences.
    *   Optimize Trie storage for cache locality.
*   **Chapter 4.3: Spell Data Architecture - Data-Driven Definitions (C#)**
    *   Define `SpellDefinition` as a data resource, including sequence, costs, and component data.
    *   Support `Mythic Evolution` through data mutation or swapping.
*   **Chapter 4.4: Casting State Machine - Player Casting Flow (C#)**
    *   Implement a `CastState` enum (Idle, Channeling, Casting, Recovery).
    *   Manage state transitions and timers for spell execution.
*   **Chapter 4.5: Visual Feedback - Particles & Animations (GDScript)**
    *   Implement a `ParticleManager` and `AnimationController` to react to casting signals.
    *   Use `GPUParticles2D` pools for optimized visual effects.
*   **Chapter 4.6: Hotbar Input & Casting - Recursive Mastery Deck (GDD B05)**
    *   Implement the 10-slot hotbar system for glyphs and shortcuts.
    *   Design the "Recursive Chaining" macro system for complex combos.
*   **Chapter 4.7: Combo Input Rhythm & Dynamic Timing Windows (GDD B05)**
    *   Implement rhythmic input processing for glyph sequences.
    *   Dynamically adjust `Combo Timing Windows` based on player mastery.
*   **Chapter 4.8: Combo Resolution Model - Outcomes & Experimentation (GDD B05)**
    *   Implement the H1 Combo Resolution Model (Full, Partial, Experimental, Solo-Cast, Fizzle).
    *   Support player experimentation and accidental technique discovery.
*   **Chapter 4.9: Interruptibility & Mobility While Casting (GDD B05)**
    *   Implement dynamic interruptibility based on mastery and combo length.
    *   Enable fully mobile casting with mastery-dependent penalties.
*   **Chapter 4.10: Combo System Overview & Interpreter (GDD B02)**
    *   Implement the `Combo Interpreter` to evaluate sequences based on order, categories, subtypes, and external factors.
    *   Define outcomes like Full Technique, Partial, Fizzle, Miscast, Forbidden Surge.
*   **Chapter 4.11: Combo Formation Rules & Repetition (GDD B02)**
    *   Enforce rules that any sequence can be attempted, order matters, and repetition matters.
    *   Implement how categories define structure and subtypes define nuance.
*   **Chapter 4.12: Combo Length Tiers & Discovery Through Repetition (GDD B02)**
    *   Define `Combo Length Tiers` (Single-Glyph to 7+ Glyphs).
    *   Implement `Discovery Through Repetition` for natural combo improvement.
*   **Chapter 4.13: Player-Invented Techniques & The Combo Journal (GDD B02)**
    *   Allow players to name and record unique, discovered techniques.
    *   Implement the `Combo Journal` for tracking attempted sequences and annotations.
*   **Chapter 4.14: Mastery Structure & Glyph Mastery (GDD B04)**
    *   Implement `Internal` (continuous) and `External` (tiered) mastery.
    *   Track `Glyph Mastery` and its effects on strain, stability, and visuals.
*   **Chapter 4.15: Combo Mastery & Unlocking Shortcut Glyphs (GDD B04)**
    *   Implement `Combo Mastery` and its benefits (rhythm, flow, reliability).
    *   Design the system for unlocking `Shortcut Glyphs` through combined mastery and bloodline resonance.
*   **Chapter 4.16: Shortcut Glyph Behavior & Evolution Paths (GDD B04)**
    *   Define `Shortcut Glyph` properties (visuals, hotbar integration, benefits).
    *   Implement `Shortcut Evolution Paths` influenced by residue, bloodline, and world events.
*   **Chapter 4.17: Legendary Techniques & NPC Combo Literacy (GDD B02/B04)**
    *   Define conditions for `Legendary Techniques` (long sequence, high stability, bloodline).
    *   Implement `NPC Combo Literacy` with tiers and evolving knowledge.

### Module 5: Chakra & Life Systems
Explore the intricate biological and environmental simulation that governs character well-being, chakra flow, and the dangerous accumulation of chaos residue.

*   **Chapter 5.1: Biological Simulation - The Bio-Tick (C#)**
    *   Implement a `BiologicalSim` with a slower `Bio-Tick` rate (e.g., 1Hz).
    *   Define `BioState` struct for core stats (Health, Stamina, Chakra, Hunger, Thirst, BodyTemp).
*   **Chapter 5.2: Metabolism System - Dynamic Decay Rates (C#)**
    *   Implement `Hunger` and `Thirst` decay based on `ActivityLevel` and `WeatherMultiplier`.
    *   Integrate Chakra consumption with Hunger spikes.
*   **Chapter 5.3: Weather System - Global Environmental State (C#)**
    *   Implement `WeatherSystem` as a global singleton with `WeatherType`, `WindDirection`, `Temperature`, `Humidity`.
    *   Define `Environmental Pressure` effects on `BioState` (e.g., Cold increases Hunger).
*   **Chapter 5.4: Damage & Recovery Pipeline - Wounds & Status Effects (C#)**
    *   Implement the `Damage Pipeline` (Raw, Mitigation, Application, Wound Generation).
    *   Define `Recovery Logic` for healing and status effect management.
*   **Chapter 5.5: Status Effect Data - Definition vs. Instance (C#)**
    *   Define `EffectDefinition` for static effect data (ID, Type, Stacks, Duration).
    *   Manage `ActiveEffect` instances on entities with `TimeRemaining` and `Stacks`.
*   **Chapter 5.6: Status Effect Lifecycle - Application & Tick Loop (C#)**
    *   Implement `ApplyEffect()` logic (refresh, stack).
    *   Design the `StatusEffectSystem` tick loop for DoT and StatMod effects.
*   **Chapter 5.7: Void Sickness & Bleed Mechanics (C#)**
    *   Implement `Void Sickness` (C04) with permanent effects and escalating consequences.
    *   Implement `Bleed` (B12) as a movement-based DoT.
*   **Chapter 5.8: Chakra Strain System - Accumulation & Consequences (GDD B03)**
    *   Implement `Per-Glyph Strain` and `Per-Technique Final Spike` accumulation.
    *   Define `High Strain Consequences` (Fatigue, Distortion, Unreliability, Burnout).
*   **Chapter 5.9: Chakra Regeneration & Pool Growth (GDD B03)**
    *   Implement `Strain Reset Mechanisms` (Natural, Meditation, Food/Rest, Sacred Locations).
    *   Design `Chakra Pool Growth` over lifespan (Age, Training, Bloodline).
*   **Chapter 5.10: Chaos Residue System - Accumulation & Distribution (GDD B03)**
    *   Implement `Residue Accumulation` from miscasts, forbidden sequences, etc.
    *   Distribute `Residue` to `Player Residue` and `World Residue` (by chunk).
*   **Chapter 5.11: Effects of Residue on the World & Player (GDD B03)**
    *   Implement `World Residue` effects (Biome Distortion, Anomaly Formation, Wildlife Corruption).
    *   Implement `Player Residue` effects (Miscast Chance, Subtype Mutations, Power Increases).
*   **Chapter 5.12: Chakra Biology - Aging, Training & Bloodline Influence (GDD B08)**
    *   Implement `Natural Aging Curve` for chakra capacity.
    *   Define `Training` as the primary growth driver and `Bloodline Influence` on quirks.
*   **Chapter 5.13: Chakra Network Injuries & Recovery (GDD B08/B12)**
    *   Implement `Chakra Network Injuries` (micro-tears, residue burns) and their effects.
    *   Design `Injury Repair` mechanisms (discipline, healing, rituals).
*   **Chapter 5.14: Pain System & Pain Suppression (GDD B12)**
    *   Implement `Pain` as a mechanical factor affecting movement, casting, and focus.
    *   Design `Pain Suppression` with its associated strain and instability costs.
*   **Chapter 5.15: Medical Ninjutsu & Bloodline Healing (GDD B12)**
    *   Implement `Medical Ninjutsu` as a skill-based discipline with chakra costs and miscast risks.
    *   Define `Bloodline Healing Influence` (regenerative, cursed lines).
*   **Chapter 5.16: Survival & Environmental Discipline - Fatigue, Sleep & Weather (GDD B14)**
    *   Implement `Sleep & Fatigue` as core survival mechanics (no hunger/thirst).
    *   Integrate `Weather Effects` on visibility, stealth, and chakra stability.
*   **Chapter 5.17: Sickness, Poison & Disease - Dynamic Threats (GDD B14)**
    *   Implement `Sickness, Poison & Disease` mechanics (sources, effects, recovery).
    *   Integrate `Mental Fatigue` from trauma and chakra overuse.

### Module 6: Combat & Tactical Engagement
Master the fast-paced, systemic combat of Sigilborne. This module covers physics-based detection, detailed damage calculation, stealth and sensing mechanics, and the strategic use of weapons and tools.

*   **Chapter 6.1: Physics Layer - Hitbox & Hurtbox Detection (GDScript)**
    *   Implement `Area2D` nodes for `Hitbox` and `Hurtbox` detection.
    *   Define `BodyPart` and `DamageProfileID` for hit data.
*   **Chapter 6.2: Detection Logic - HitEvent Pipeline (Brain & Body)**
    *   Implement `area_entered` signal to send `HitEvent` to the Brain.
    *   Design `CombatSystem` to calculate damage and emit `OnDamageTaken` signals.
*   **Chapter 6.3: Damage Calculator - Complex Math Layer (C#)**
    *   Implement `DamageResult` struct for final damage, crit, block, and type.
    *   Develop `Calculate()` function for base damage, mitigation, and multipliers.
*   **Chapter 6.4: Combat Fundamentals - Pace, Lethality & Reading Opponents (GDD B09)**
    *   Implement fast, high-pressure, reactive combat mechanics.
    *   Design `Lethality & Danger Gradient` based on enemy type and context.
*   **Chapter 6.5: Ninja Movement & Chakra-Enhanced Mobility (GDD B09)**
    *   Implement `Movement` as a core skill with dodges, slides, and aerial control.
    *   Design `Chakra-Enhanced Mobility` (Surge dash, Veil slide) with mastery reduction.
*   **Chapter 6.6: Interruptibility & Casting Mobility (GDD B09)**
    *   Implement universal `Interruptibility` for non-instant techniques.
    *   Design `Casting Mobility` allowing movement while weaving signs, with mastery reducing penalties.
*   **Chapter 6.7: Taijutsu Integration & Priority Rules (GDD B09)**
    *   Implement `Taijutsu` as a non-chakra-based martial art.
    *   Design `Chakra-Enhanced Taijutsu` variants and `Casting vs. Taijutsu Priority Rules`.
*   **Chapter 6.8: Combat Symmetry & Enemy Intelligence (GDD B09)**
    *   Implement `Enemy Intelligence Spectrum` from low-tier to legendary.
    *   Ensure `Combat Symmetry` where enemies follow player rules (signs, strain, miscast).
*   **Chapter 6.9: Stealth Progression & Types (GDD B10)**
    *   Implement `Stealth Progression Curve` (weak early, powerful late).
    *   Design `Physical Stealth`, `Chakra Stealth`, and `Corrupted Stealth` modes.
*   **Chapter 6.10: Detection Systems & Enemy Learning (GDD B10)**
    *   Implement `Detection Types` (Sight, Sound, Chakra Sensing, Resonance, Bloodline).
    *   Design `Enemy Learning Behavior` for intelligent adaptation and coordination.
*   **Chapter 6.11: Glyph-Based Stealth Techniques (GDD B10)**
    *   Implement `Veil`, `Echo`, `Flux`, `Bloom`, and `Consume` techniques for stealth.
    *   Integrate `Chakra Usage` and `Residue Interaction` with stealth.
*   **Chapter 6.12: Stealth Failure & Environmental Influence (GDD B10)**
    *   Implement a `Multi-State Detection Model` for stealth failure.
    *   Design `Environmental Influence` on stealth (buffs/penalties from terrain, weather, resonance).
*   **Chapter 6.13: Chakra Sensing Fundamentals & Trainable Skill (GDD B11)**
    *   Implement `Chakra Sensing` as a trainable skill with tiers.
    *   Define `Who Can Sense Chakra` and how sensing ability improves.
*   **Chapter 6.14: Sensing Range, Clarity & Combat Intuition (GDD B11)**
    *   Implement `Sensing Range Tiers` and `What Can Be Sensed` (Chakra Quantity, Residue, Bloodline Signatures).
    *   Design `Combat Intuition` to provide subtle anticipatory feedback.
*   **Chapter 6.15: Specialized Sensing Schools & Residue Interaction (GDD B11)**
    *   Implement `Echo Sensing`, `Veil Sensing`, `Flux Sensing`, `Consume Sensing`, `Bloom Sensing`.
    *   Integrate `Residue Corruption` effects on sensing.
*   **Chapter 6.16: Weapon Philosophy & Mastery (GDD B13)**
    *   Define `Weapon Philosophy` as supplemental, not central to power.
    *   Implement `Weapon Mastery` for movement style, rhythm, and chakra infusion.
*   **Chapter 6.17: Chakra Interaction with Weapons & Types (GDD B13)**
    *   Implement `Chakra Infusion System` for weapon enhancement (Bloom, Veil, Echo, Surge).
    *   Categorize `Weapon Types` (Traditional, Clan, Chakra-Forged, Residue-Corrupted).
*   **Chapter 6.18: Ninja Tools & The Tool Belt (GDD B13)**
    *   Implement `Ninja Tools` (smoke bombs, tags, traps, sensors).
    *   Design the `Tool Belt` with dedicated slots separate from glyphs.

### Module 7: The Living World - Ecology & AI
Build the dynamic, simulated ecosystem where NPCs, animals, and spirits coexist and evolve. This module covers perception, decision-making, and the background simulation that makes the world feel alive even when the player isn't directly observing it.

*   **Chapter 7.1: Perception System - Spatial Hashing (C#)**
    *   Implement a `Spatial Hash Grid` for efficient entity detection.
    *   Optimize queries for entities within range.
*   **Chapter 7.2: The Senses - Vision, Hearing & Chakra Sense (C#)**
    *   Implement `PerceptionComponent` with `Vision` (cone check), `Hearing` (circle check).
    *   Design `Chakra Sense` (B11) for detecting active spells through walls.
*   **Chapter 7.3: Stealth Mechanics - Visibility & Detection Meter (C#)**
    *   Implement `Visibility` as a float value (light, movement, camouflage).
    *   Design `Detection Meter` to trigger NPC alert states.
*   **Chapter 7.4: Decision Making - Utility AI (C#)**
    *   Implement `Utility AI` (Scoring) for NPC action selection.
    *   Design `Scorer` functions for actions like EatFood, AttackEnemy, Sleep.
*   **Chapter 7.5: The Scheduler - Daily Routines & Overrides (C#)**
    *   Implement `Daily Routines` for NPCs (Work, Socialize, Sleep).
    *   Design `Override` logic for threats or major events.
*   **Chapter 7.6: Ecology Simulation - Virtual Agents (C#)**
    *   Implement `Virtual Agents` for entities in unloaded chunks.
    *   Design `Background Simulation` for updating virtual agents at a slow tick rate.
*   **Chapter 7.7: Spawning & Despawning - Hydration/Dehydration (Brain & Body)**
    *   Implement `Hydration` (Virtual to Active) when player approaches chunks.
    *   Design `Dehydration` (Active to Virtual) when player leaves chunks.
*   **Chapter 7.8: Natural Animals & Ecosystem Interaction (GDD B15)**
    *   Implement `Full Wildlife Ecosystems` with realistic behavior.
    *   Design `Animal Interaction` with stealth and combat (flee, warn, attack).
*   **Chapter 7.9: Spirit Creatures - Taxonomy & Visibility Tiers (GDD B15)**
    *   Implement `Spirit Creatures` as a layered ecosystem (Forest, Shadow, Echo, Guardians).
    *   Design `Spirit Visibility Tiers` (resonance zones, trained sensors, rituals).
*   **Chapter 7.10: Spirit Intelligence & Resonance Interaction (GDD B15)**
    *   Implement `Spirit Intelligence` (negotiate, form pacts, teach, threaten).
    *   Design `Resonance Interaction` for spirits based on their nature.
*   **Chapter 7.11: Corrupted & Mythical Creatures (GDD B15)**
    *   Implement `Corrupted Creatures` (mutated animals/spirits) from residue.
    *   Design `Mythical Creatures` (titanic guardians, sky serpents) as rare, late-game entities.
*   **Chapter 7.12: Procedural Species Variability & Territory Behavior (GDD B15)**
    *   Implement `Procedural Species` generation for unique fauna per world.
    *   Design `Territory Behavior` for creatures (dens, nests, patrol routes).
*   **Chapter 7.13: NPC Identity Architecture - Personality & Social Bonds (GDD B18)**
    *   Define `Core Personality` (courage, curiosity, discipline) for NPCs.
    *   Implement `Social Bonds` (friends, rivals, mentors) affecting behavior.
*   **Chapter 7.14: NPC Profession Roles & Clan Alignment (GDD B18)**
    *   Define `Profession Roles` (farmer, guard, monk) for daily life.
    *   Implement `Clan Alignment` (loyal, clanless, defect) affecting literacy and training.
*   **Chapter 7.15: Autonomy & Decision-Making - Personality & Social Drivers (GDD B18)**
    *   Implement `NPC Decision-Making` driven by personality weighting and social bonds.
    *   Design `Anomaly Influence` and `Faction Politics` on NPC choices.
*   **Chapter 7.16: Emergent NPC Learning - Observation & Social Transmission (GDD B18)**
    *   Implement `Observation Learning` (watching player/NPC casts, anomalies).
    *   Design `Social Transmission` (rumors, training tips) for knowledge spread.
*   **Chapter 7.17: NPC Reaction Model - Instant, Gradual & Slow (GDD B18)**
    *   Implement `NPC Reactions` at three speeds: instant (danger), gradual (politics), slow (culture).
    *   Ensure NPCs adapt schedules dynamically.
*   **Chapter 7.18: NPC Mastery Speed & Glyph Progression (GDD B19)**
    *   Implement `NPC Mastery Speed` with personality, clan, and environmental modifiers.
    *   Define `Five Stages of NPC Glyph Mastery` (Unstable to Shortcut).
*   **Chapter 7.19: NPC Glyph Learning - Clan Training & Anomaly-Based (GDD B19)**
    *   Implement `Observation Learning`, `Social Transmission`, `Clan Training Programs`.
    *   Design `Anomaly-Based Learning` and `Trauma-Based Learning`.
*   **Chapter 7.20: NPC Combo Development & Mutation-Driven Growth (GDD B19)**
    *   Implement `NPC Combo Development` from clan standards to personal experiments.
    *   Design `Mutation-Driven Skill Growth` from residue and forbidden events.

### Module 8: Society, Politics & Economy
Construct the complex social and economic fabric of Sigilborne. This module covers faction dynamics, settlement structures, a knowledge-centric economy, and how crime and justice operate in a morally ambiguous world.

*   **Chapter 8.1: Faction System - The Relationship Graph (C#)**
    *   Implement `FactionRelation` struct for tracking inter-faction relationships.
    *   Design `Faction AI` with goals and actions (DeclareWar, SendCaravan).
*   **Chapter 8.2: The Simulation Clock - Game Time & Load Balancing (C#)**
    *   Implement a "Game Time" system with a slower tick rate (e.g., 1 min = 1 real sec).
    *   Stagger heavy system updates across the daily cycle (`FactionAI`, `MarketSim`).
*   **Chapter 8.3: Market Simulation - Supply & Demand (C#)**
    *   Implement `MarketSim` where prices are local and dynamic.
    *   Design `Price` calculation based on `Demand / Supply` and event modifiers (War, Famine).
*   **Chapter 8.4: Trade Routes - Caravans as Arteries (C#)**
    *   Implement `CaravanSystem` for NPC caravans moving between settlements.
    *   Design `Trade Interruption` logic (player/bandit destruction affecting prices).
*   **Chapter 8.5: Crime & Justice - The Heat System (C#)**
    *   Implement `Heat System` to track crime per faction.
    *   Design `Witness` mechanics for crime recording.
*   **Chapter 8.6: Bounty & Punishment - Escalation & Consequences (C#)**
    *   Implement `Bounty` system with escalating hunter squad responses.
    *   Define `Consequences` for crimes (refusal to talk, attack on sight).
*   **Chapter 8.7: Settlement Formation Logic - Templates & Inputs (GDD B16)**
    *   Implement procedural `Settlement Formation` based on biome, faction ancestry, anomalies.
    *   Design `Settlement Templates` (Village, Clan District, Monastic, Trade Hub).
*   **Chapter 8.8: District System - Modular Subdivisions (GDD B16)**
    *   Implement `District Types` (Common, Clan, Monastic, Marketplace, Spirit, Corrupted).
    *   Assign unique rules, hierarchies, and literacy patterns per district.
*   **Chapter 8.9: NPC Population Model - Individuated Agents (GDD B16)**
    *   Implement NPCs as individual agents with literacy, relationships, political alignment.
    *   Ensure `Dynamic Village Story` through NPC actions and memories.
*   **Chapter 8.10: Internal Social Structure - Multi-Hierarchy Stack (GDD B16)**
    *   Implement `Civic`, `Clan`, `Monastic`, `Spirit`, and `Corruption` hierarchies.
    *   Design interactions where hierarchies can support, compete, or undermine each other.
*   **Chapter 8.11: Faction Presence in Settlements - Dynamic Tension (GDD B16)**
    *   Implement `Multiple Factions` competing over knowledge, anomaly management, and succession within settlements.
    *   Ensure `Constant Dynamic Tension` through their interactions.
*   **Chapter 8.12: Economic System - Knowledge-Centric Economy (GDD B16/B23)**
    *   Implement an economy built around `Magical Scarcity` and `Knowledge Exchange`.
    *   Define `Cores of the Economy` (Scroll fragments, Glyph diagrams, Anomaly reagents).
*   **Chapter 8.13: Settlement Borders & Influence Zones (GDD B16)**
    *   Implement `Settlement Influence Zones` with patrols, wards, and resonance markers.
    *   Design `Magical Gradients` for borders that affect glyph behavior.
*   **Chapter 8.14: Dynamic Settlement Events & Multigenerational Evolution (GDD B16)**
    *   Implement `Procedural Story Engine` for settlement events (corruption outbreak, ward collapse).
    *   Design `Multigenerational Evolution` for knowledge drift, leadership cycles, and legacy interactions.
*   **Chapter 8.15: Clans, Factions & Political Actors (GDD B17)**
    *   Define `Clans` as micro-factions, `Major Factions` as macro powers, and `Supra-Faction Forces`.
    *   Implement `Faction AI` for strategic moves, alliances, and conflicts.
*   **Chapter 8.16: Clan Architecture - Ideology & Leadership (GDD B17)**
    *   Implement `Clan Identity` across `Category Ideology` (Bloom+Bind) and `Leadership Structure`.
    *   Design dynamic leadership changes via trials, corruption, or anomaly events.
*   **Chapter 8.17: Clan Lifecycle - Formation, Growth, Merging & Dissolution (GDD B17)**
    *   Implement `Clan Formation` (schism, charismatic founder, player influence).
    *   Design `Clan Growth`, `Merging`, and `Dissolution` mechanics.
*   **Chapter 8.18: Diplomacy & Relationships - Cross-Faction Web (GDD B17)**
    *   Implement `Diplomatic State` (Ally, Neutral, Hostile) with internal numeric meters.
    *   Design `Diplomacy Drivers` (ideological conflict, territory disputes, player involvement).
*   **Chapter 8.19: Political Heat & Volatility (GDD B17)**
    *   Implement `Political Heat` system from corruption, forbidden arts, NPC deaths.
    *   Design `Heat Triggers` for diplomatic shifts, clan splits, and revolts.
*   **Chapter 8.20: Economy Core Principles & Currency (GDD B23)**
    *   Implement a `Local, Systemic Economy` where knowledge, materials, and relationships outweigh gold.
    *   Define `Gold Usage` (bribery, travel) vs. non-gold transactions (glyphs, rituals).
*   **Chapter 8.21: Categories of Goods - Basic to Illicit (GDD B23)**
    *   Categorize `Goods` into Basic, Crafting, Magical Reagents, Anomaly-Derived, Corrupted, Spiritual, Illicit.
    *   Implement their usage and value within the economy.
*   **Chapter 8.22: Trade Systems - Ideological Markets & Caravans (GDD B23)**
    *   Implement `Settlement Economies` as ideological, with preferred/banned goods.
    *   Design `Caravans` as arteries with dynamic routes, risks, and consequences.
*   **Chapter 8.23: Black Markets & Goods Flow Simulation (GDD B23)**
    *   Implement `Black Markets` for outlaw networks (corruption, assassins-for-hire).
    *   Design `Goods Flow Simulation` for resource movement, shortages, and NPC reactions.
*   **Chapter 8.24: Player Housing Types & Availability (GDD B24)**
    *   Implement `Dynamic Housing` (Purchasable, Clan-Granted, Outlaw, Spirit, Claimed).
    *   Design `Realistic Availability` based on population, disasters, and politics.
*   **Chapter 8.25: Housing Functions & Upgrades (GDD B24)**
    *   Implement `Functional Perks` (meditation, training, crafting, rituals, storage).
    *   Design `Medium-Depth Upgrades` (Meditation Alcove, Ritual Circle, Corruption Chamber).

### Module 9: Advanced World Mechanics & Player Impact
This module delves into the complex systems of seals, rituals, contracts, and how player actions drive emergent missions, crime, and justice in the world.

*   **Chapter 9.1: Ritual System - Pattern Matching & Execution (C#)**
    *   Implement `RitualManager` to detect spatial arrangement of items for rituals.
    *   Design `Execution` logic for consuming components and triggering effects.
*   **Chapter 9.2: Seals & Locks - Logical Locks (C#)**
    *   Implement `Seal` as a logical lock component on objects (Physical, Magical, Blood, Time).
    *   Design `Global Seals` that affect the entire world state.
*   **Chapter 9.3: Ritual System - Simple, Advanced & Forbidden (GDD B27)**
    *   Implement `Simple/Medium Rituals` (healing, cleansing, ward maintenance).
    *   Design `Advanced Rituals` (anomaly stabilization, spirit summoning) with complex requirements.
    *   Implement `Forbidden Rituals` (resurrection, corruption ascension) with irreversible consequences.
*   **Chapter 9.4: Forbidden Contracts & Spirit Contracts (GDD B27)**
    *   Implement `Forbidden Contracts` as pacts with powerful forces (spirits, corruption, anomalies).
    *   Design `Spirit Contracts` (C3) with deep relational bonds and high risk.
*   **Chapter 9.5: Corruption Pacts & Multi-Life Persistence of Rituals (GDD B27)**
    *   Implement `Corruption Pacts` (C4) for mutation growth and corrupted abilities.
    *   Ensure `Multi-Life Persistence` for seals, rituals, and contracts across generations.
*   **Chapter 9.6: Mission Categories - Morally Ambiguous Tasks (GDD B20)**
    *   Implement `Mission Categories` (Settlement, Clan, NPC, Anomaly, Corruption, Political, Forbidden).
    *   Ensure tasks emerge from `Systemic World State` and are morally ambiguous.
*   **Chapter 9.7: Mission Delivery Methods & Tracking (GDD B20)**
    *   Implement `Diegetic Delivery` via NPC dialogue, overheard conversations, environmental clues.
    *   Design `Mission Tracking` solely through a `Diegetic Journal` (no UI markers).
*   **Chapter 9.8: Mission Rewards & Group Missions (GDD B20)**
    *   Implement `Rewards` (Gold, Materials, Knowledge, Access, Political Favor, Reputation Echo).
    *   Design `Group Missions` where NPCs join based on trust, clan orders, or crisis.
*   **Chapter 9.9: Crime & Justice Legal Architecture (GDD B21)**
    *   Implement `Regional Faction Codes` and `Clan-Level Variation` for laws.
    *   Define `Crime Categories` (Martial, Political, Magical, Economic, Spiritual, Corruption).
*   **Chapter 9.10: Detection, Suspicion & Witnesses (GDD B21)**
    *   Implement `Crime Detection` based on witnesses, evidence, rumors, and magical traces.
    *   Design `NPC Personality` and ideology to affect reporting behavior.
*   **Chapter 9.11: Bounty System & Prison Simulation (GDD B21)**
    *   Implement a `Dynamic Multi-Faction Bounty Network` (kill, capture, humiliate).
    *   Design a `Full Prison World-Simulation` with prison factions, dynamics, and gameplay loops.
*   **Chapter 9.12: Outlaw Path & Intergenerational Crime Memory (GDD B21)**
    *   Implement `Fully Viable Outlaw Gameplay Loop` (black markets, rogue clans, assassination).
    *   Ensure `Intergenerational Crime Memory` where past crimes shape future NPC reactions.

### Module 10: The Mythic & Meta Game - Legacy, Collapse & Multiverse
This final module integrates all systems into the grand vision of Sigilborne. You'll explore how player actions lead to world-shaping events, mythic transformations, and the generational persistence that defines the game's unique meta-narrative.

*   **Chapter 10.1: World Generation & Streaming - Chunk Architecture (C#)**
    *   Implement `Chunk Architecture` (64x64 tiles) with `Active Radius` loading.
    *   Design `Threaded Loading` for efficient chunk generation and mesh application.
*   **Chapter 10.2: World Persistence - The "Diff" Strategy (C#)**
    *   Implement `Delta Compression` for saving only chunk modifications.
    *   Design `Serialization Format` (binary Protobuf) and `Save Versioning`.
*   **Chapter 10.3: Virtual Agent Sync - Handover & Restoration (C#)**
    *   Implement `Handover` (Active to Virtual) and `Restoration` (Virtual to Active) for NPCs.
    *   Ensure persistence of NPC stats across loaded/unloaded states.
*   **Chapter 10.4: World Instability Index (WII) & Collapse Seeds (GDD B29)**
    *   Implement `WorldStability` metric (Doom Clock) influenced by corruption, anomalies, clan wars.
    *   Design `Collapse Seeds` (Corruption, Anomaly, Spirit, Political) as early threats.
*   **Chapter 10.5: Regional Threats & Global Collapse States (GDD B29)**
    *   Implement `Regional Threats` (Corruption Bloom, Anomaly Storm, Spirit Awakening).
    *   Design `Global Collapse States` (Corruption Surge, Anomaly Cascade, Spirit Cataclysm) triggered by converging crises.
*   **Chapter 10.6: Player Influence & Recovery Logic for Collapse (GDD B29)**
    *   Implement `Player Influence` to prevent, delay, redirect, or accelerate collapse.
    *   Design `Manual, Slow, Multi-Generation Recovery` requiring deliberate effort.
*   **Chapter 10.7: Mythic Evolution Philosophy & The Evolution Engine (GDD C01)**
    *   Define `Mythic Evolution` as emergent metamorphosis, not content to chase.
    *   Implement the `Evolution Engine` combining chakra signature, world resonance, and instability.
*   **Chapter 10.8: Three Stages of Mythic Evolution - Distortion to Emergence (GDD C01)**
    *   Implement `Resonant Distortion`, `Metaphysical Fracture`, and `Mythic Emergence`.
    *   Design `Catastrophic Instability` and `World Reaction` during evolution.
*   **Chapter 10.9: High-Tier Techniques & Emergent Evolution (GDD C02)**
    *   Implement `10+ Sign Combo Complexity` for reality-adjacent power.
    *   Design `Technique Evolution` based on internal/external factors and world state.
*   **Chapter 10.10: Spirit Contracts & Ancestral Pacts - Tiered Relationships (GDD C03)**
    *   Implement `Spirit Taxonomy` (Lesser, Domain, Greater, Forbidden Echo Entities).
    *   Design `Contract Arcs` with environmental alignment, ritual components, and dream/spirit trials.
*   **Chapter 10.11: Forbidden Arts & Aberration Metamorphosis (GDD C04)**
    *   Define `Corrupted Chakra` as a volatile overclocking phenomenon.
    *   Implement `Aberration Forms` as dangerous, ninja-shaped metamorphosis.
*   **Chapter 10.12: Residue Overgrowth & Forbidden Glyph Sequences (GDD C04)**
    *   Implement `Residue Overgrowth` as a world-affecting spread.
    *   Design `Forbidden Glyph Sequences` as unstable, painful, chaotic combos.
*   **Chapter 10.13: World Bosses, Titans & Primordial Entities (GDD C05)**
    *   Define `Titans` as hybrid spirit-anomaly-corruption entities.
    *   Implement `Behavioral Identity` (ecological/political actors) and `Procedural Mutation`.
*   **Chapter 10.14: Titan Domains & Reincarnation (GDD C05)**
    *   Implement `Titan Domains` as living regions reshaping terrain and ecology.
    *   Design `Titan Reincarnation` (mutated return) for persistent challenges.
*   **Chapter 10.15: Climax & Catastrophe - Collapse Triggers & Recovery (GDD C06)**
    *   Implement `Collapse Types` (Corruption Cascade, Anomaly Surge, Spirit Uprising).
    *   Design `Political Response` and `Player Influence` during collapse.
*   **Chapter 10.16: Meta-System: Reincarnation & Legacy - World Memory (GDD B30/C07)**
    *   Implement `Total World Persistence` (no mechanical inheritance for new characters).
    *   Design `Social, Mythic, and Corruption Memory` for past lives.
*   **Chapter 10.17: Legacy Influence & Past-Life Echoes (GDD B30/C07)**
    *   Implement `Legacy Influence` across NPC dialogue, political shifts, spiritual reactions.
    *   Design `Past-Life Echoes` as common dream illusions and rare manifestations.
*   **Chapter 10.18: Endless Mode & The Multiverse (GDD C08)**
    *   Implement `Hybrid Infinite Worlds` (unlimited seeds, parallel existing worlds, independent timelines).
    *   Design `Cross-World Influence` (Echo bleedthrough) and `Player Identity Across Worlds` (Echo awareness).
*   **Chapter 10.19: The Echo Layer & Cosmology - Foundation of All Worlds (GDD C09)**
    *   Define the `Echo Layer` as the metaphysical substrate for all reality.
    *   Explain its role in the `Origin of Chakra, Spirits, Corruption, Anomalies, Titans, Glyphs, Collapse, and Infinite Worlds`.
*   **Chapter 10.20: Meaning of Sigilborne & Final Canon (GDD C09)**
    *   Define a `Sigilborne` as one attuned to Echo resonance.
    *   Establish `No Single Canon` but an infinite multiverse of player-created stories.
*   **Chapter 10.21: System Integration & World Simulation (GDD B31)**
    *   Implement `Staggered Global Tick Architecture` with cross-system influence.
    *   Design `Multi-Layer Simulation` (Political, Magical, Ecological, Corruption, Anomaly) with player agency.

---
[[BLUEPRINT_GENERATED]]