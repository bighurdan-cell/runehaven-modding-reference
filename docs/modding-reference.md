# Runehaven Modding Reference

A reference for what's possible when modding [Runehaven](https://store.steampowered.com/app/2676030/). Covers schemas, the rune system, the L-system zone format, the node object types, asset registries, and the line between data modding and code modding.

This doc is not official. It complements the mod system's example modpack and the modding guide currently being written by the dev. Where the official docs cover something, prefer those. This is for everything else.

## Reading this doc

- **Confirmed** facts come from running examples through the loader, console probing, or direct dev confirmation
- **Inferred** facts come from reading the decompile but haven't been exercised in-game
- **Untested** items are flagged in-place; those are good places to contribute corrections

## Game and tooling versions

- Game: Runehaven by Cikoria Studio, released April 20 2026
- Engine: Unity 6000.2.7f2, IL2CPP backend (metadata v31.1)
- JSON serializer: Newtonsoft.Json
- Mod loader path: `StreamingAssets/mods/<mod_id>/mod.json`
- MagicaVoxel version (for `.vox` files): 0.99.7.1
- In-game console: F11 (pasting may not work; type commands; up arrow recalls history)

## At a glance

```
┌────────────────────────────────────────────────────────────────┐
│ FULLY DATA-MODDABLE                                            │
│   mod.json       manifest with optional path overrides         │
│   items/         weapons, runes, consumables, ammo, keys       │
│   mobs/          stats, AI archetype, composable steering      │
│   props/         destructibles, doors, levers, traps, portals  │
│   biomes/        lighting, fog, prop/item/material/mob mix     │
│   themes/        UI restyling                                  │
│   worlds/        run config, layers, tunnel gen                │
│   zones/         L-system grammar, custom action IDs           │
│   projectiles/   damage, lights, particles, status, physics    │
│   modifiers/     stat-modifying suffix variants                │
│   particles/     emission shapes, colors, lifetime, bursts     │
│                                                                │
│ ASSETS                                                         │
│   models/        .vox (MagicaVoxel 0.99.7.1, multi-frame anim) │
│   sprites/       .png                                          │
│   samples/       .wav                                          │
│                                                                │
│ AUTHORED IN GAME EDITOR                                        │
│   nodes/         5 object types: Shape, Connector, Prop,       │
│                  Item, PlayerSpawnpoint                        │
│                                                                │
│ HIDDEN CAPABILITIES                                            │
│   items[].runes              pre-bake spell into weapon        │
│   items[].light_source       items emit light                  │
│   items[].audio_source       items emit looping audio          │
│   mobs[].steering_behaviors  composable AI primitives          │
│   props[].on_interact        interaction hooks                 │
│   props[].explode_on_destroy explosive containers              │
│   biomes[].item_map          biomes spawn items                │
│   biomes[].material_map      biomes override voxel materials   │
│   zone actions[].hallway     per-action corridor materials     │
│   zone "ambient_color"       override biome ambient color      │
│   node Item.modifier         per-instance item modifier        │
│   node Item.runes            per-instance pre-bound spell      │
│   node Item.respawn          item respawning loot              │
│   node Prop.is_open/key      door state, lock keys             │
│                                                                │
│ REQUIRES CODE MODDING (BepInEx)                                │
│   - new rune effects (33 hardcoded classes)                    │
│   - new mob AI archetypes (13 hardcoded)                       │
│   - new prop interaction types (~20 hardcoded)                 │
│   - new status effects (12 hardcoded)                          │
│   - new player classes (Unity ScriptableObjects)               │
│   - new player skills (Unity ScriptableObjects)                │
│   - new console commands                                       │
│                                                                │
│ DOES NOT EXIST                                                 │
│   - quest system                                               │
│   - dialogue system                                            │
│   - NPC scripting beyond friendly_humanoid behavior            │
└────────────────────────────────────────────────────────────────┘
```

## Where vanilla content lives

Worth knowing up front: vanilla items, mobs, props, projectiles, and modifiers do not ship as JSON files. They live inside Unity asset bundles. The only loose-file folders shipped in `StreamingAssets/` are:

```
biomes/         demo/           mods/           nodes/
ParticleSystems/ RuneSigns/     Sounds/         Soundtracks/
sprites/        Testing/        themes/         worlds/
zones/          config.json (empty placeholder)
```

This means:
- Schemas in this doc come from the IL2CPP decompile, not from reading vanilla JSONs (because there aren't any to read for some categories)
- The `mod_example` modpack is the only loose-file reference implementation for items, mobs, props, projectiles, and modifiers
- A modpack is the only place where item/mob/prop/projectile/modifier JSONs exist on disk

## What's moddable

| Category | Folder default | Per-mod path field | Loaded from |
|---|---|---|---|
| Mod manifest | `/` | n/a | required at root |
| Items (weapons, runes, consumables, ammo, keys, armor) | `items/` | `item_path` | mod only |
| Mobs (enemies) | `mobs/` | `mob_path` | mod only |
| Props (destructibles, doors, etc) | `props/` | `prop_path` | mod only |
| Projectiles | `projectiles/` | (no override seen) | mod only |
| Modifiers (item suffix variants) | `modifiers/` | (no override seen) | mod only |
| Biomes | `biomes/` | `biome_path` | vanilla + mods |
| Themes | `themes/` | `theme_path` | vanilla + mods |
| Worlds | `worlds/` | `world_path` | vanilla + mods |
| Zones | `zones/` | `zone_path` | vanilla + mods |
| Nodes (rooms) | `nodes/` | `node_path` | vanilla + mods |
| Particle systems | `ParticleSystems/` | (no override seen) | vanilla only? |
| Models (`.vox`) | `models/` | `model_path` | vanilla + mods |
| Sprites (`.png`) | `sprites/` | `sprite_path` | vanilla + mods |
| Audio samples (`.wav`) | `samples/` | `sample_path` | vanilla + mods |
| Rune signs (gestures) | `RuneSigns/` | (top-level only) | vanilla + (mod untested) |

## What is NOT moddable without code (BepInEx)

Confirmed from decompile:
- **Quest systems do not exist.** No quest classes anywhere in the code.
- **Rune effects.** 33 hardcoded `Rune` subclasses. Constants baked in
  (FireRune burns at 85% probability for 4-7s, AirRune has 4.5m radius
  push, etc). New rune *items* can be authored as JSON, but they must
  reference one of the 33 existing effect classes by string ID.
- **Mob AI archetypes.** 13 hardcoded behavior types. New mobs reuse them.
  Steering is composable from 12 hardcoded primitives.
- **Prop interaction logic.** ~20 hardcoded behavior types.
- **Status effects.** 12 hardcoded subclasses.
- **Player classes** and **player skills.** Both are Unity
  `ScriptableObject`s, baked into asset bundles.
- **Console command set.** Hardcoded.
- **Combat balance, save format, level-up curve, loot pool weighting.**

## How systems connect

```
mod.json (manifest with optional path overrides)
    │
    ├── worlds/*.json (WorldData3) ← appears in New World dropdown
    │       │
    │       ├── soundtracks, mob_count, merchants
    │       ├── tunnel_settings, stairs_*, place_runes
    │       └── layers[]
    │               │
    │               ├── zone[] → zones/*.json
    │               │       │
    │               │       ├── axiom + rules (Lindenmayer)
    │               │       └── actions[] (type: node | loop | push?)
    │               │               ├── node_paths[] (folder OR file)
    │               │               └── hallway: { floor/wall/ceiling_material }
    │               │                       │
    │               │                       └── nodes/*.node
    │               │                               ├── Shape (CSG)
    │               │                               ├── Prop (typed instances)
    │               │                               ├── Item (placed loot)
    │               │                               ├── Connector (entrance/exit)
    │               │                               └── PlayerSpawnpoint
    │               │
    │               ├── biomes[] → biomes/*.json (Biome2)
    │               │       └── prop_map, item_map, material_map, mobs
    │               └── mobs[] / ascension_mobs[] → mobs/*.json
    │
    ├── items/*.json (Item, Rune subclass)
    ├── projectiles/*.json
    ├── modifiers/*.json (ItemModifier)
    ├── props/*.json
    └── themes/*.json (ThemeData)
```

Loading order from `Game.cs`:

```
LoadConfig → LoadFonts → LoadVoxelData → LoadMaterials → LoadSprites →
LoadSamples → LoadRuneSigns → LoadThemes → LoadProjectiles → LoadProps →
LoadItems → LoadModifiers → LoadMobs → LoadBiomes2 → LoadZones → LoadNodes →
LoadWorlds3 → LoadModpacks → LoadClasses → LoadSkills → LoadSaves
```

Modpacks load LAST so they can reference vanilla content but vanilla
cannot reference modpack content.

### Vanilla content replacement via ID override

Confirmed by the dev: a modpack can override vanilla content by shipping
a JSON with the same ID as the vanilla version. So `items/sunblade.json`
in your mod replaces the vanilla `sunblade`. This applies to any category
where the loader uses ID-based registration (items, mobs, props,
projectiles, biomes, zones, themes, worlds, particle systems).

---

## mod.json (Modpack)

```json
{
    "id": "mod_example",
    "title": "Mod example",
    "description": "...",
    "version": "1.0.0",

    "sample_path": "samples",
    "model_path": "models",
    "item_path": "items",
    "sprite_path": "sprites",
    "mob_path": "mobs",
    "prop_path": "props",
    "node_path": "nodes",
    "zone_path": "zones",
    "biome_path": "biomes",
    "world_path": "worlds",
    "theme_path": "themes",
    "debug": false
}
```

Required: `id`, `title`. Path fields default to the matching folder name and can be omitted unless you want a non-standard layout.

---

## items/*.json (Item)

Full field set, drawn from the example mod and the decompiled `Item` class. Items load from the modpack's `items/` folder by default.

```json
{
    "id": "blue_sword",
    "title": "Blue sword",
    "description": "...",
    "type": "weapon",
    "level": 1,

    "model": "blue_sword",
    "voxel_resolution": 16,
    "scale": 0.6,
    "pivot": { "x": 1.5, "y": 5.0, "z": 4.5 },
    "offset": { "x": 0, "y": 0, "z": 0 },

    "quantity": 1,
    "probability": 1.0,
    "is_loot": true,
    "stackable": false,
    "droppable": true,
    "identified": true,
    "has_condition": false,
    "snap_to_terrain": false,

    "weapon_type": "melee_weapon",
    "min_damage": 2,
    "max_damage": 4,
    "min_knockback": 1,
    "max_knockback": 2,
    "critical_hit_chance": 0.05,
    "range": 2.5,
    "melee_attack_type": "swing",
    "projectile_source": { "x": 14.5, "y": 3.5, "z": 26 },

    "loot_tier": 2,
    "damage_tier": 1,
    "range_tier": 1,
    "speed_tier": 3,
    "knockback_tier": 1,

    "fire_sound": "dagger_swing",
    "destruct_sound": "metal_weapon_break",
    "parry_sound": "parry",
    "pick_up_sound": "...",
    "use_sound": "...",
    "use_color": { "r": 1, "g": 1, "b": 1, "a": 1 },

    "audio_source": { "sample": "...", "range": 5.0, "volume": 0.5 },
    "light_source": { "intensity": 1.5, "color": { "r": 1, "g": 0.5, "b": 0.2, "a": 1 } },

    "runes": ["missile_rune", "fire_rune"],
    "tags": ["warrior", "melee_weapon"],
    "stats": []
}
```

Confirmed item `type` values: `weapon`, `consumable`, `rune`, `key`,
`armor` (likely, untested).

Confirmed `weapon_type` values from existing items and behaviors:
`melee_weapon`, `staff`. Almost certainly also: `bow`, `crossbow`,
`throwing_weapon`.

### Runes are items

A rune in your inventory is an `Item` of type `rune` deserialized into
the `Rune` subclass:

```json
{
    "id": "fire_rune",
    "type": "rune",
    "rune_type": "fire_rune",
    "rune_sign": "rune_3",
    "mana_cost": 5,
    "sound": "magic_2",
    "sample": "fire_cast",
    "is_standalone": false
}
```

`rune_sign` references a file in `RuneSigns/` (gesture pattern).
`rune_type` selects the hardcoded effect class.

---

## Rune Effect Registry (33 classes, partial item availability)

All 33 classes exist in code. Only a subset ship as spawnable inventory
items in vanilla. Whether the rest can be referenced from item `runes`
arrays is empirically untested and likely no.

### Confirmed spawnable as items (via `add_item` console)

| ID | Class | Effect |
|---|---|---|
| `fire_rune` | FireRune | Burning status (85% chance, 4-7s) |
| `air_rune` | AirRune | Push: 4.5m radius, 45 strength |
| `missile_rune` | MissileRune | Single projectile delivery |
| `explosive_rune` | ExplosiveRune | AOE on impact |
| `explosion_rune` | ExplosiveRune (alias) | Same as above |

### Likely spawnable as items (confirmed class, untested)

| ID | Class |
|---|---|
| `earth_rune` | EarthRune |
| `water_rune` | WaterRune |
| `light_rune` | LightRune |
| `scatter_rune` | ScatterRune |
| `seek_rune` | SeekRune |

### Class exists but does NOT spawn from console

These didn't add to inventory when probed. The class is in code, but
either no rune item ships with that ID, or the ID differs from the class
name pattern. May still be referenceable from item `runes[]` arrays in
JSON, untested.

`orb_rune`, `ring_rune`, `circle_rune`, `fragment_rune`, `bounce_rune`,
`returning_rune`, `chaos_rune`, `attraction_rune`, `repulsion_rune`,
`gravity_rune`, `impulse_rune`, `momentum_rune`, `permeability_rune`,
`radial_damage_rune`, `knockback_rune`, `slow_rune`, `fast_rune`,
`longevity_rune`, `luck_rune`, `lifesteal_rune`, `manasteal_rune`,
`harmful_rune`, `elemental_rune`, `ascension_rune`, `split_rune`

For modding purposes, design around the confirmed-spawnable set.

---

## mobs/*.json (Mob)

```json
{
    "id": "giant_dragonfly",
    "type": "dragonfly",
    "size": "small",
    "scale": 2.0,
    "voxel_resolution": 16,
    "flying": true,
    "mass": 1.0,

    "collider_center": { "x": 0, "y": 0.5, "z": 0 },
    "collider_height": 1.0,
    "collider_radius": 0.4,
    "rig_center": 0.5,
    "view_distance": 12.0,

    "health": 14.0,
    "min_damage": 3, "max_damage": 6,
    "min_coins": 0, "max_coins": 5,
    "xp": 20,
    "color": { "r": 0.45, "g": 0.4, "b": 0.7 },
    "blood_color": { "r": 0.45, "g": 0.5, "b": 0.29 },

    "active_sound": "insect_fly_1",
    "attack_sound": "insect_attack",
    "hurt_sound": "hurt_1",
    "death_sound": "death_1",
    "grunt_sounds": ["grunt_1"],

    "elemental_weaknesses": [{ "element": "fire", "value": 1.0 }],
    "elemental_resistances": [{ "element": "air", "value": 1.0 }],

    "steering_behaviors": [
        { "type": "seek_target", "weight": 1.0 },
        { "type": "avoid_obstacles", "weight": 0.8 },
        { "type": "wander", "weight": 0.2 }
    ],

    "attack_animation": "attack",
    "idle_animation": "idle",
    "walk_animation": "fly",
    "walk_animation_speed": 1.0,

    "animator": { ... },
    "loot": { ... },
    "point_lights": [ ... ]
}
```

### Mob behavior types (`type` field)

| `type` | Class | Description |
|---|---|---|
| `character` | CharacterMobBehavior | Generic humanoid |
| `crawler` | CrawlerMobBehavior | Crawling enemy |
| `dragonfly` | DragonflyMobBehavior | Flying insect |
| `elemental` | ElementalMobBehavior | Air/water/fire/earth elementals |
| `flying` | FlyingMobBehavior | Generic flying |
| `flying_ranged` | FlyingRangedMobBehavior | Ranged flying caster |
| `friendly_humanoid` | FriendlyHumanoidMobBehavior | NPCs, merchants |
| `jumping_insect` | JumpingInsectMobBehavior | Flea, frog, etc |
| `magic_human` | MagicHumanMobBehavior | Caster humanoid |
| `melee_human` | MeleeHumanMobBehavior | Melee humanoid |
| `bomber` | (Bomber) | Cyclops bomber |
| `pinnsvain` | PinnsvainBehavior | Boss |
| `rauk` | RaukBehavior | Boss |
| `wraith` | WraithBehavior | Boss |

### Steering behaviors (composable, weighted)

| `type` | Class |
|---|---|
| `avoid_obstacles` | AvoidObstaclesSteeringBehavior |
| `avoid_player` | AvoidPlayerSteeringBehavior |
| `avoid_target` | AvoidTargetSteeringBehavior |
| `avoid_terrain` | AvoidTerrainSteeringBehavior |
| `flying_avoid_player` | FlyingAvoidPlayerSteeringBehavior |
| `forward` | ForwardSteeringBehavior |
| `patrol` | PatrolSteeringBehavior |
| `random` | RandomSteeringBehavior |
| `seek` | SeekSteeringBehavior |
| `seek_path` | SeekPathSteeringBehavior |
| `seek_target` | SeekTargetSteeringBehavior |
| `wander` | WanderSteeringBehavior |

---

## props/*.json (Prop)

```json
{
    "type": "destructible",
    "health": 30.0,
    "scale": 0.75,
    "voxel_resolution": 16,
    "pivot": { "x": 8, "y": 0, "z": 8 },

    "dynamic": true,
    "use_collider_physics": true,
    "destructible": true,
    "destructible_from_impacts": true,
    "create_model": true,
    "use_instancing": false,
    "spawn_small_mob": false,
    "can_pick_up": true,
    "snap_to_terrain": true,
    "terrain_snap_dir": { "x": 0, "y": -1, "z": 0 },
    "add_collider_to_debris": true,

    "active_sound": "barrel_creak",
    "damage_sound": "wood_hit",
    "destruct_sound": "wood_break",

    "hover_text": "A green barrel",
    "tooltip_offset": { "x": 0, "y": 1, "z": 0 },
    "on_interact": "...",

    "destroy_probability": 0.5,
    "explode_on_destroy": false,
    "explosion_radius": 3.0,
    "explosion_damage": 10.0,
    "destruct_particle_system": "fire_explosion",

    "point_lights": [ ... ],
    "particle_systems": [ ... ],

    "loot": { ... },
    "tags": ["container"]
}
```

### Prop behavior types (`type` field)

| `type` | Class |
|---|---|
| `bear_trap` | BearTrapBehavior |
| `billboard` | BillboardPropBehavior |
| `door` | DoorBehavior |
| `info` | InfoBehavior |
| `lever` | LeverBehavior |
| `marker` | MarkerBehavior |
| `note` | NoteBehavior |
| `portal_room` | PortalRoomBehavior |
| `portal_room_crystal_stand` | PortalRoomCrystalStandBehavior |
| `end_portal` | EndPortalBehavior |
| `end_demo_portal` | EndDemoPortalBehavior |
| `portcullis` | PortcullisBehavior |
| `rotator` | RotatorBehavior |
| `runestone` | RunestoneBehavior |
| `shape_prop` | ShapePropBehavior (default for destructibles) |
| `spell_trap` | SpellTrapBehavior |
| `spikes` | SpikesBehavior |
| `turning_wheel` | TurningWheelBehavior |
| `item_spawner` | (uses ItemSpawner) |

---

## projectiles/*.json (Projectile)

```json
{
    "id": "fire_missile",
    "type": "magic",
    "model": "fire_missile",
    "voxel_resolution": 16,
    "scale": 1.0,
    "pivot": { "x": 0, "y": 0, "z": 0 },
    "radius": 0.2,
    "lifetime": 5.0,
    "damage_delay": 0.0,

    "min_damage": 5.0, "max_damage": 8.0,
    "min_knockback": 1.0, "max_knockback": 3.0,

    "rotation_speed": { "x": 0, "y": 360, "z": 0 },
    "use_velocity_as_direction": true,
    "use_rotation_speed": false,

    "create_light": true,
    "light_intensity": 2.0,
    "light_radius": 4.0,
    "light_color": { "r": 1, "g": 0.5, "b": 0.1, "a": 1 },
    "point_light": { ... },

    "particle_systems": ["fire"],
    "destruct_particle_system": "fire_explosion",

    "impact_sound": "fire_impact",
    "stick_sound": "arrow_stick",

    "destroy_on_collision": true,
    "stick_on_collision": false,
    "explode_on_destroy": true,
    "explode_radius": 2.0,
    "explosion_damage": 6.0,
    "damage_destructibles": true,
    "bounciness": 0.0,
    "gravity": 0.0,

    "status_effects": [
        { "type": "burning", "duration": { "min": 3.0, "max": 5.0 }, "probability": 0.85 }
    ]
}
```

---

## modifiers/*.json (ItemModifier)

The "Sword of Burning +2" system.

```json
{
    "id": "burning",
    "suffix": "of Burning",
    "title_color": { "r": 1, "g": 0.5, "b": 0, "a": 1 },
    "probability": 0.1,
    "parameters": [
        {
            "type": "damage",
            "key": "fire_damage",
            "value": 2.0,
            "operation": "add",
            "description": "+2 fire damage",
            "color": { "r": 1, "g": 0.4, "b": 0.1, "a": 1 }
        }
    ]
}
```

`operation` confirmed `add`. Likely also `multiply`, `set` (untested).

---

## biomes/*.json (Biome2)

```json
{
    "id": "biome_example",
    "description": "...",

    "ambient_light_hue": { "min": 0.0, "max": 0.1 },
    "ambient_light_value": { "min": 0.4, "max": 0.5 },
    "ambient_light_saturation": { "min": 0.7, "max": 0.8 },

    "fog_start_distance": { "min": 2, "max": 3 },
    "fog_end_distance": { "min": 15, "max": 22 },
    "fog_hue": { "min": 0.0, "max": 0.1 },
    "fog_value": { "min": 0.3, "max": 0.5 },
    "fog_saturation": { "min": 0.6, "max": 0.8 },

    "floor_details": ["small_rocks_1"],
    "ceiling_details": ["cobweb_1"],
    "floor_edge_props": ["wooden_barrel"],

    "prop_map": [
        { "tag": "floor_detail", "props": ["mushroom_1"] }
    ],
    "item_map": [ ... ],
    "material_map": [ ... ],

    "spawners": [
        { "props": ["small_rocks_1"], "target": "floor", "density": 1 }
    ],

    "mobs": [ { "id": "...", "probability": 1.0 } ]
}
```

`spawners[].target` confirmed: `"floor"`, `"ceiling"`. Likely also `"wall"`.

---

## themes/*.json (ThemeData)

UI restyling. Replaces panel/tooltip/minimap/bar sprites and font.

```json
{
    "name": "Gothic",
    "description": "Gothic theme.",
    "panel_sprite": "gothic_sprite_panel",
    "panel_border": { "x": 7, "y": 7, "z": 7, "w": 7 },
    "panel_spacing": 4.0,
    "panel_padding": 24,
    "font": "alagard",
    "font_size": 24,
    "font_color": { "r": 0.8, "g": 0.8, "b": 1.0, "a": 1.0 },
    "pixel_scale": 0.35,
    "slot_sprite": "default_panel_sprite",
    "slot_border": { "x": 2, "y": 2, "z": 2, "w": 2 },
    "tooltip_sprite": "default_panel_sprite",
    "tooltip_border": { "x": 2, "y": 2, "z": 2, "w": 2 },
    "tooltip_padding": 15,
    "minimap_sprite": "gothic_minimap_sprite",
    "minimap_border": { "x": 7, "y": 7, "z": 7, "w": 7 },
    "bar_sprite": "default_panel_sprite",
    "bar_border": { "x": 2, "y": 2, "z": 2, "w": 2 }
}
```

`*_border` is a 9-slice border definition (left, right, top, bottom in pixels). Sprite IDs reference either `sprites/<name>.png` in the same mod or vanilla sprites like `default_panel_sprite`.

How a theme actually gets applied to a world or world layer is not yet documented. Worth verifying empirically.

---

## worlds/*.json (WorldData3)

```json
{
    "id": "runehaven",
    "name": "Runehaven",
    "soundtracks": [...],
    "merchants": ["merchant_1", "merchant_2", "merchant_3"],
    "mob_count": { "min": 240, "max": 260 },
    "place_runes": true,
    "tunnel_count": { "min": 4, "max": 8 },
    "exit_tunnel_probability": 0.4,
    "tunnel_settings": { ... },
    "stairs_width": { "min": 2.0, "max": 4.0 },
    "stairs_height": { "min": 3.0, "max": 4.0 },
    "stairs_step_length": { "min": 0.3, "max": 0.6 },
    "floor_detail_count": { "min": 600, "max": 600 },
    "ceiling_detail_count": { "min": 500, "max": 500 },
    "wall_prop_count": { "min": 220, "max": 220 },
    "beartrap_count": { "min": 50, "max": 75 },
    "debug": false,
    "layers": [ ... ]
}
```

---

## zones/*.json

The L-system grammar with named actions. Real vanilla zones use more
features than `mod_example` showed.

```json
{
    "id": "dungeon",
    "axiom": "entrance intermediate exit loop push",
    "rules": [
        {
            "predecessor": "intermediate",
            "successor": "intermediate intermediate"
        },
        {
            "predecessor": "intermediate",
            "successor": "prefab_intermediate intermediate",
            "probability": 0.6
        }
    ],
    "actions": [
        {
            "id": "entrance",
            "type": "node",
            "node_paths": [ "nodes/dungeon/entrance" ],
            "hallway": {
                "floor_material": "stone_tiles",
                "wall_material": "bricks",
                "ceiling_material": "rock"
            }
        },
        {
            "id": "exit",
            "type": "node",
            "node_attach_type": "exit",
            "node_paths": [ "nodes/dungeon/exit" ],
            "hallway": { ... }
        },
        {
            "id": "intermediate",
            "type": "node",
            "node_paths": [ "nodes/dungeon/intermediate" ]
        },
        {
            "id": "prefab_intermediate",
            "type": "node",
            "allow_duplicates": false,
            "node_paths": [ "nodes/dungeon/prefab_intermediate" ]
        },
        {
            "id": "loop",
            "type": "loop",
            "loop_floor_material": "sand"
        }
    ],
    "ambience": ["dungeon_ambience_1"],
    "music": ["music_2"],
    "ambient_color": { "r": 0.1, "g": 0.1, "b": 0.1 },
    "biomes": ["mystical", "dark"],
    "mobs": [ { "id": "flea" } ],
    "spawners": [
        { "props": ["small_rocks_1"], "target": "floor", "density": 1 }
    ]
}
```

### Zone action types

| `type` | Notes |
|---|---|
| `node` | Pick a `.node` file from `node_paths`. Most common. |
| `loop` | Closes a loop in the dungeon graph. Takes `loop_floor_material`. |

The `push` symbol that appears in axioms is likely a built-in
L-system stack command, not a custom action. Not seen as an action `type`.

### Action fields

| Field | Notes |
|---|---|
| `id` | Symbol this action handles. Custom IDs supported (village uses `part_1`, `part_2`, `part_3`). |
| `type` | `node` or `loop`. |
| `node_paths` | Array of folder paths OR specific `.node` file paths. |
| `node_attach_type` | Confirmed: `"exit"`. |
| `allow_duplicates` | If false, same node won't repeat in run. |
| `hallway` | `{ floor_material, wall_material, ceiling_material }` for connecting corridors. |
| `loop_floor_material` | For `type: loop`. |
| `probability` | (on rules) Probabilistic application. |

### Rule fields

`predecessor` (symbol to match), `successor` (space-separated expansion),
optional `probability`. Probabilistic rules are non-exclusive (a symbol
can match multiple rules and the engine chooses one weighted).

### Custom action IDs

Zones can use any symbol names, not just `entrance`/`intermediate`/`exit`.
Example from `village.json`:

```json
"axiom": "entrance part_1 part_2 part_3 exit loop push",
"actions": [
    { "id": "part_1", "type": "node",
      "node_paths": [ "nodes/village/intermediate/wight_village_1.node" ] },
    ...
]
```

This pattern lets you control the exact sequence of rooms in a zone.

---

## nodes/*.node (DataFile format)

Custom text format produced by the in-game level editor. Each room is a
list of typed objects. Five object types confirmed.

### Shape (CSG geometry)

```
$BEGIN_OBJECT Shape
$string name "Shape"
$int mode 0
$int type 1
$float3 pos (2.89 3.91 12.32)
$float3 rot (0 0 0)
$float3 scale (17.39 8.82 25.64)
$int operator 0
$int floor_mat 3
$int wall_mat 6
$int ceiling_mat 0
$float smoothness 0
$float noise_strength 0
$float noise_scale 2.0
$float radius 0.5
$float start_radius 0.5
$float end_radius 0.5
$float3 start (0 0 0)
$float3 end (0 0 0)
$string node ""
$END_OBJECT
```

`type` and `operator` are integer enums for primitive shape and CSG
operation (union/subtract). `floor_mat`/`wall_mat`/`ceiling_mat` are
int indices into the global material palette.

### Connector (entrance/exit anchors)

```
$BEGIN_OBJECT Connector
$int type 0
$string guid ""
$float3 pos (1.06 -0.50 -2.66)
$float3 rot (0 0 0)
$END_OBJECT
```

`type 0` = entrance, `type 1` = exit. Connector count and arrangement
determine how the L-system can wire rooms together.

| Configuration | Use case |
|---|---|
| Zero connectors | Standalone room (e.g. spawn point) |
| One type-0 only | Terminal node (player spawn, treasure room) |
| One type-0 + one type-1 | Linear corridor segment |
| One type-0 + multiple type-1 | Branching room |
| Multiple type-0 + multiple type-1 | Loop segment |

### Prop (placed instances with type-specific extensions)

```
$BEGIN_OBJECT Prop
$string id "wooden_door_3"
$string name "wooden_door_3"
$string type "door"
$string guid "..."
$string parent_guid ""
$float3 local_pos (-0.54 -0.50 2.38)
$string node ""
$float3 pos (-0.54 -0.50 2.38)
$float3 rot (0 0 0)
$float3 scale (1 1 1)
$int is_open 0
$int is_unlocked 0
$int is_gate 0
$string key "silver_key"
$END_OBJECT
```

The `type` field selects the prop behavior class. Per-instance
behavior-specific fields can extend the placement. For doors:
`is_open`, `is_unlocked`, `is_gate` (int booleans), `key` (item ID
required to open).

Other prop types likely accept their own per-instance fields. Worth
testing by inspecting nodes that include levers, traps, runestones, etc.

### Item (placed loot)

```
$BEGIN_OBJECT Item
$string id "silver_key"
$string type "key"
$string modifier ""
$string runes ""
$string node ""
$string custom_data ""
$int quantity 1
$float condition 10
$float3 pos (-0.05 -0.46 0.98)
$float3 rot (0 27.98 0)
$float3 scale (1 1 1)
$int respawn 0
$float respawn_duration 15
$END_OBJECT
```

Each placed item can carry a per-instance `modifier`, a pre-bound
`runes` string, a `condition` (durability), and respawn behavior
(`respawn` 0/1, `respawn_duration` seconds).

### PlayerSpawnpoint

```
$BEGIN_OBJECT PlayerSpawnpoint
$float3 pos (0.45 -0.28 0.42)
$float3 rot (0 0 0)
$int type 0
$END_OBJECT
```

---

## Particle systems

Reference set in `StreamingAssets/ParticleSystems/`: `bolt_trail`,
`fire`, `fire_explosion`, `hand_torch_fire`, `torch_fire`. Vanilla files
use a smaller field set than the C# schema's full surface.

```json
{
    "looping": true,
    "min_start_size": 0.08,
    "max_start_size": 0.15,
    "min_color": { "r": 1, "g": 0, "b": 0 },
    "max_color": { "r": 1, "g": 0.6, "b": 0 },
    "emission_color": { "r": 1, "g": 0.5, "b": 0 },
    "use_emission_color": true,
    "min_color_over_lifetime": { "r": 2, "g": 2, "b": 0.3 },
    "max_color_over_lifetime": { "r": 2, "g": 0.3, "b": 0 },
    "noise_strength": 0.2,
    "shape": "sphere",
    "angle": 10,
    "min_start_speed": 0.3,
    "max_start_speed": 2,
    "min_lifetime": 0.5,
    "max_lifetime": 1.0,
    "simulation_space": "world",
    "rate": 30,
    "bursts": [{ "x": 100, "y": 200 }],
    "gravity": 0.1
}
```

Confirmed values:
- `shape`: `"sphere"`, `"cone"`. Likely also `"box"`, `"hemisphere"`.
- `simulation_space`: `"world"`. Likely also `"local"`.
- `bursts`: array of `{x: minCount, y: maxCount}` pairs. When
  `looping: false` and `rate: 0`, particle is one-shot via bursts.
- Color values can exceed 1.0 (e.g. `{"r": 2, "g": 0.3}`) for HDR/bloom
  effects.

---

## Status Effect Registry (12 hardcoded)

Reference by `type` from projectile `status_effects[]`.

| ID (probable) | Class | Effect |
|---|---|---|
| `aloe_vera` | AloeVeraStatusEffect | Healing-related |
| `burning` | BurningStatusEffect | Fire damage over time |
| `cikoria` | CikoriaStatusEffect | Mana regeneration |
| `dragonroot` | DragonrootStatusEffect | Buff |
| `foxglove` | FoxgloveStatusEffect | Buff/debuff |
| `frozen` | FrozenStatusEffect | Freeze movement |
| `poison` | PoisonStatusEffect | Poison DoT |
| `poison_new` | PoisonStatusEffectNew | Newer poison variant |
| `slimy` | SlimyStatusEffect | Slow movement |
| `stats` | StatsStatusEffect | Generic stat modifier |
| `web` | WebStatusEffect | Slow/stuck |
| `wet` | WetStatusEffect | Lightning conductivity |

---

## Element Type Enum

```
none = 0, fire = 1, earth = 2, water = 3, air = 4, magma = 5, ice = 6
```

Six damage types. Used in `elemental_weaknesses` and
`elemental_resistances` on mobs. Note: no `magic` or `light` element
despite rune names.

---

## Console commands (F11)

Pasting may not work in the console UI. Type commands manually.
Up arrow recalls history (last 100 commands persisted).

| Command (probable) | Class | Notes |
|---|---|---|
| `add_item <id> [<id>...]` | AddItemCommand | Spawns items into inventory |
| `break_weapons` | BreakWeaponsCommand | Damages held/equipped weapons |
| `collapse` | CollapseCommand | Triggers ceiling collapse |
| `destroy_chunk_meshes` | DestroyChunkMeshesCommand | Debug |
| `fly` | FlyCommand | Toggle flight (confirmed) |
| `heal` | HealCommand | Restore health |
| `kill_mobs` | KillMobsCommand | Kill all mobs |
| `level_up` | LevelUpCommand | Gain a level |
| `message <text>` | MessageCommand | On-screen message |
| `play <sound>` | PlayCommand | Play sound |
| `remove_floating_props` | RemoveFloatingPropsCommand | Cleanup |
| `spawn <id>` | SpawnCommand | Generic spawn |
| `spawn_mob <id>` | SpawnMobCommand | Spawn enemy (confirmed) |
| `spawn_of_type <type>` | SpawnOfTypeCommand | Spawn by category |
| `spawn_prop <id>` | SpawnPropCommand | Spawn prop |
| `take_damage <amount>` | TakeDamageCommand | Damage player |
| `teleport <x> <y> <z>` | TeleportCommand | Teleport player |

No built-in `help` command. Empirical probing (`add_item <guess>`) is
how to discover IDs.

The console also has `EvaluateCustomCommand` and
`EvaluateScript(string, target)` methods, suggesting custom command
registration and script execution. Hooks unknown.

---

## Vanilla asset registry

### Mobs
```
flea, bat, spitter, crawler, spider_warrior,
undead_warrior, undead_mage, undead_arbalist, undead_rogue, undead_hunter,
air_elemental, water_elemental, earth_elemental, fire_elemental, magic_elemental,
cyclops_bomber, harvestman, dragonfly, scorpion,
imp, fiend, lich, wight,
merchant_1, merchant_2, merchant_3
```

### Props
```
mushroom_1, mushroom_2, mushroom_3, small_rocks_1,
cobweb_1, chain_1, ceiling_roots_1,
wooden_barrel, wooden_crate, green_barrel,
grass_1, grass_2, grass_3,
vines_ceiling_1, vines_ceiling_2, vines_ceiling_3,
wooden_door_3
```

### Items confirmed
```
silver_key (key),
fire_rune, air_rune, missile_rune, explosive_rune (runes)
```

### Materials (named, used in zone hallways and node mat indices)
```
stone_tiles, bricks, rock,
sand, sandstone, sandstone_bricks
```

### Sounds
```
dagger_swing, metal_weapon_break, parry, wood_break,
insect_fly_1, hurt_1, death_1, eat_herb, magic_2
```

### Soundtracks
```
shroom, lost, mist, tunnels, ravine, the_maze, tikku, gate, dungeon
(suffixed with _soundtrack)
```

### Music
```
music_2, music_8 (likely music_1 through music_N exist)
```

### Ambience
```
cave_ambience_1, dungeon_ambience_1, temple_ambience_1
```

### Particle systems
```
bolt_trail, fire, fire_explosion, hand_torch_fire, torch_fire
```

### Biomes
```
cave, dark, desert, grown, mines, misty,
mystical, mystical_cave, sinister, toxic, tropical
```

### Zones
```
cave_entrance, cave, mine, village, sandy_tunnels,
temple, dungeon, halls,
pinnsvain_arena, rauk_arena, wraith_arena
```

### Loot tags
```
healing_potion, mana_potion,
common_melee_weapon, common_ranged_weapon, common_magic_weapon,
uncommon_ranged_weapon,
throwing_weapon, ammo, scroll, coins, rope
```

### Item models
```
wooden_staff, longsword, cikoria, blue_sword, green_barrel,
giant_dragonfly_1
```

---

## Tooling

### Required
- **MagicaVoxel 0.99.7.1** (https://ephtracy.github.io). For `.vox`
  models. The version matters: `.vox` files saved in newer or older
  MagicaVoxel versions may fail to load. Confirmed by the dev as a
  real-world issue. Mob animations use multi-frame `.vox` files.
- **Runehaven's in-game level editor** for `.node` files.
- **Text editor** for JSON.

### Recommended
- **VS Code** with JSON Schema validation.
- **Git** for versioning.
- **Aseprite** for sprite/9-slice work.
- **Audacity** for `.wav` editing.

### For deep modding (BepInEx)
- **Cpp2IL** (https://github.com/SamboyCoding/Cpp2IL) for decompilation.
  Already run, dummy DLL output available if needed.
- **dnSpy** or **ILSpy** to browse decompiled assemblies.
- **BepInEx 6 IL2CPP** for runtime patching.

### Workspace layout

Mods live at:
```
C:\Program Files (x86)\Steam\steamapps\common\Runehaven\
    Runehaven_Data\StreamingAssets\mods\<mod_id>\
```

Don't put a git repo there. Use a directory junction:
```cmd
mklink /J "C:\...\StreamingAssets\mods\my_mod" "C:\dev\runehaven-mods\my_mod"
```

The `debug: true` field in `mod.json` may surface verbose logging.
Worth setting during development.

---

## Open questions

Things the doc can't yet resolve. Each can be answered empirically. Contributions welcome.

1. Whether the ~24 non-spawnable rune classes can still be referenced from item `runes[]` arrays. Test by adding one to a custom weapon and observing whether it functions.
2. The full set of `ItemModifier.parameters[].operation` values. `add` is confirmed; `multiply` and `set` are likely.
3. The full set of `shape` values for particle systems. `sphere` and `cone` are confirmed in vanilla files; `box` and `hemisphere` are likely.
4. The full set of `simulation_space` values for particle systems. `world` is confirmed; `local` is likely.
5. Whether mod-local `RuneSigns/` folders load, or only the top-level one.
6. How a theme gets selected per world or per layer (not visible in the world or layer schemas).
7. Whether `worlds[].layers[].zone[]` and `biomes[]` arrays pick one entry or merge them.
8. The full set of zone action `type` values. `node` and `loop` are confirmed; others may exist.
9. Built-in L-system commands beyond `loop` and `push`.
10. Behavior-specific extensions for non-door props in `.node` files. Doors carry `is_open`, `is_unlocked`, `is_gate`, `key`. Other prop types likely have their own per-instance fields.
11. The full vanilla mob `type` registry. `dragonfly` is the only confirmed example; the other 12 archetypes have classes but the JSON `type` strings haven't been matched 1:1.
