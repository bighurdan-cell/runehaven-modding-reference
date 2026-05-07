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
│ NOTABLE CAPABILITIES (less obvious from the example modpack)   │
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
│ EXTENDED VIA CODE OR SCRIPTING                                 │
│   - new rune effects (extend via BepInEx today, MiniScript     │
│     once it ships)                                             │
│   - new mob AI archetypes                                      │
│   - new prop interaction types                                 │
│   - new status effects                                         │
│   - new player classes and skills                              │
│   - new console commands                                       │
└────────────────────────────────────────────────────────────────┘
```

## Where vanilla content lives

Worth knowing up front: vanilla items, mobs, props, projectiles, and modifiers don't ship as JSON files alongside the game. They live inside the game's compiled assets. The loose-file folders shipped in `StreamingAssets/` are:

```
biomes/         demo/           mods/           nodes/
ParticleSystems/ RuneSigns/     Sounds/         Soundtracks/
sprites/        Testing/        themes/         worlds/
zones/          config.json (empty placeholder)
```

What this means in practice: the `mod_example` modpack is the canonical reference implementation for items, mobs, props, projectiles, and modifiers. A modpack is the only place where these JSONs exist on disk, and authoring new ones is how you add them to the game.

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

## Beyond data modding

Some kinds of changes go past what JSON modpacks can do. The natural paths are BepInEx (C# patching at runtime, what you'd reach for today) and the in-progress MiniScript integration (sandboxed scripting tied to game events, more accessible once it ships).

Things that fall in this territory:

- **New rune effect logic.** Authoring new rune *items* in JSON works, but they reference existing effect classes by ID. A genuinely new effect (a new element, new projectile shape, new modifier) is code or scripting work.
- **New mob AI behavior.** Mob `type` selects from a set of behavior classes. Reusing them with new stats and visuals is fully data-driven, but a new behavior shape is code or scripting.
- **New prop interaction types.** Same pattern. Doors, levers, runestones, etc., are reusable; new interaction logic is code.
- **New status effects.** The status effect IDs you can reference from projectiles map to specific behaviors. New ones need code.
- **Player classes and skills.** These are baked into the game's compiled assets, not JSON-driven.
- **Console commands.** Hardcoded.
- **Combat balance, save format, level-up curve, loot pool weighting.**

JustAHarmlessCat's [NoRuneRedo](https://github.com/JustAHarmlessCat/NoRuneRedo) mod is a working example of BepInEx-based code modding for Runehaven if you're looking for a starting reference.

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

## mod.json

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

## items/*.json

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

Confirmed item `type` values: `weapon`, `consumable`, `rune`, `key`, `armor`.

Confirmed `weapon_type` values: `melee_weapon`, `staff`, `bow`, `crossbow`, `throwing_weapon`.

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

## Rune effect registry

Rune items in your inventory reference an effect by string ID. The IDs below are the rune effects available in the game. Items can stack multiple of these in a `runes[]` array to combine behaviors (a `["missile_rune", "fire_rune"]` weapon shoots fire missiles, etc).

### Confirmed spawnable as items

These rune items can be added to your inventory via the `add_item` console command, which means they exist as fully-formed items in the game.

| ID | Effect category | Notes |
|---|---|---|
| `fire_rune` | Element | Fire damage, applies burning |
| `air_rune` | Element | Radial push |
| `missile_rune` | Delivery | Single projectile |
| `explosive_rune` | Delivery | AOE on impact |
| `explosion_rune` | Delivery | Alias of `explosive_rune` |

### Likely spawnable (untested via console)

The pattern strongly suggests these work the same way. Try them and see.

`earth_rune`, `water_rune`, `light_rune`, `scatter_rune`, `seek_rune`

### Effect classes that exist but didn't spawn from the console

These show up in the game's effect class set but didn't add to inventory when probed via `add_item`. Three possibilities: the rune item ships under a different ID, the item exists but isn't reachable through `add_item`, or the effect is referenced internally rather than as a player-facing rune. They may still be referenceable from item `runes[]` arrays in JSON; that's an empirical test worth running.

`orb_rune`, `ring_rune`, `circle_rune`, `fragment_rune`, `bounce_rune`, `returning_rune`, `chaos_rune`, `attraction_rune`, `repulsion_rune`, `gravity_rune`, `impulse_rune`, `momentum_rune`, `permeability_rune`, `radial_damage_rune`, `knockback_rune`, `slow_rune`, `fast_rune`, `longevity_rune`, `luck_rune`, `lifesteal_rune`, `manasteal_rune`, `harmful_rune`, `elemental_rune`, `ascension_rune`, `split_rune`

If you confirm any of these work (or don't), please contribute back.

---

## mobs/*.json

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

| `type` | Description |
|---|---|
| `character` | Generic humanoid |
| `crawler` | Crawling enemy |
| `dragonfly` | Flying insect |
| `elemental` | Air/water/fire/earth elementals |
| `flying` | Generic flying |
| `flying_ranged` | Ranged flying caster |
| `friendly_humanoid` | NPCs, merchants |
| `jumping_insect` | Flea, frog, etc |
| `magic_human` | Caster humanoid |
| `melee_human` | Melee humanoid |
| `bomber` | Cyclops bomber |
| `pinnsvain` | Boss |
| `rauk` | Boss |
| `wraith` | Boss |

### Steering behaviors (composable, weighted)

| `type` | Notes |
|---|---|
| `avoid_obstacles` | Avoids static geometry |
| `avoid_player` | Moves away from the player |
| `avoid_target` | Moves away from current target |
| `avoid_terrain` | Avoids walls, drops, etc |
| `flying_avoid_player` | Flying-mob variant of avoid_player |
| `forward` | Moves forward |
| `patrol` | Patrols an area |
| `random` | Random heading |
| `seek` | Pursues |
| `seek_path` | Pathfinds toward a target |
| `seek_target` | Pursues current target |
| `wander` | Idle wandering |

---

## props/*.json

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

| `type` | Notes |
|---|---|
| `bear_trap` | Trap that snaps shut |
| `billboard` | Static billboard prop |
| `door` | Openable door (carries `is_open`, `is_unlocked`, `is_gate`, `key`) |
| `info` | Info display |
| `lever` | Toggleable lever |
| `marker` | Marker/waypoint |
| `note` | Readable note |
| `portal_room` | Portal-room prop |
| `portal_room_crystal_stand` | Crystal stand variant |
| `end_portal` | End-of-run portal |
| `end_demo_portal` | Demo end portal |
| `portcullis` | Portcullis gate |
| `rotator` | Rotating prop |
| `runestone` | Interactive runestone |
| `shape_prop` | Default for destructibles |
| `spell_trap` | Magic trap |
| `spikes` | Spike trap |
| `turning_wheel` | Crank/wheel |
| `item_spawner` | Spawns items on a schedule |

---

## projectiles/*.json

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

## modifiers/*.json

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

`operation` accepts `add` (confirmed). `multiply` and `set` are likely but untested.

---

## biomes/*.json

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

## themes/*.json

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

## worlds/*.json

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

## nodes/*.node

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

## Status effect IDs

Reference these by `type` from a projectile's `status_effects[]` array.

| ID | Effect |
|---|---|
| `aloe_vera` | Healing-related |
| `burning` | Fire damage over time |
| `cikoria` | Mana regeneration |
| `dragonroot` | Buff |
| `foxglove` | Buff/debuff |
| `frozen` | Freeze movement |
| `poison` | Poison DoT |
| `poison_new` | Newer poison variant |
| `slimy` | Slow movement |
| `stats` | Generic stat modifier |
| `web` | Slow/stuck |
| `wet` | Lightning conductivity |

---

## Element types

The damage types referenced in `elemental_weaknesses` and `elemental_resistances` on mobs:

`fire`, `earth`, `water`, `air`, `magma`, `ice`

Note that `magic` and `light` aren't elements, despite the rune names suggesting they might be. They're handled separately.

---

## Console commands (F11)

Pasting may not work in the console UI; type commands manually. Up arrow recalls history (last 100 commands persisted).

| Command | Notes |
|---|---|
| `add_item <id> [<id>...]` | Spawns items into inventory |
| `break_weapons` | Damages held/equipped weapons |
| `collapse` | Triggers ceiling collapse |
| `destroy_chunk_meshes` | Debug |
| `fly` | Toggle flight |
| `heal` | Restore health |
| `kill_mobs` | Kill all mobs |
| `level_up` | Gain a level |
| `message <text>` | On-screen message |
| `play <sound>` | Play sound |
| `remove_floating_props` | Cleanup |
| `spawn <id>` | Generic spawn |
| `spawn_mob <id>` | Spawn enemy |
| `spawn_of_type <type>` | Spawn by category |
| `spawn_prop <id>` | Spawn prop |
| `take_damage <amount>` | Damage player |
| `teleport <x> <y> <z>` | Teleport player |

There's no built-in `help` command. Probing IDs you suspect (e.g. `add_item my_guess`) is how to discover what's actually registered.

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

### Items
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
- **MagicaVoxel 0.99.7.1** (https://ephtracy.github.io). For `.vox` models. The version matters: `.vox` files saved in different MagicaVoxel versions can fail to load. Mob animations use multi-frame `.vox` files.
- **Runehaven's in-game level editor** for `.node` files.
- **Text editor** for JSON.

### Recommended
- **VS Code** with JSON Schema validation.
- **Git** for versioning.
- **Aseprite** for sprite/9-slice work.
- **Audacity** for `.wav` editing.

### For code modding (BepInEx)
- **BepInEx 6 IL2CPP** for runtime patching.
- **Cpp2IL** (https://github.com/SamboyCoding/Cpp2IL) for decompilation.
- **dnSpy** or **ILSpy** to browse decompiled assemblies.

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

## Questions for the community

Things that aren't yet confirmed and are worth verifying as people build:

1. Whether the rune effects that didn't spawn from console can still be referenced from item `runes[]` arrays. Test by adding one to a custom weapon and seeing whether it fires.
2. The full set of values for `ItemModifier.parameters[].operation`. `add` is confirmed; `multiply` and `set` are likely.
3. The full set of `shape` values for particle systems. `sphere` and `cone` are confirmed; `box` and `hemisphere` are likely.
4. The full set of `simulation_space` values. `world` is confirmed; `local` is likely.
5. Whether mod-local `RuneSigns/` folders load, or only the top-level one.
6. How a theme gets selected per world or per layer (not obviously visible in the world or layer schemas).
7. Whether `worlds[].layers[].zone[]` and `biomes[]` arrays pick one entry per run or merge them.
8. Zone action `type` values beyond `node` and `loop`.
9. Built-in L-system commands beyond `loop` and `push`.
10. Per-instance fields for prop types beyond `door` in `.node` files. Doors carry `is_open`, `is_unlocked`, `is_gate`, `key`; other prop types probably have their own.
11. Full vanilla mob `type` registry. `dragonfly` is the only confirmed pairing of JSON `type` to behavior; others haven't been matched 1:1.

If you confirm any of these or find something else worth noting, contributions are welcome.
