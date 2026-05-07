# Runehaven Modding Reference

A community reference for what's possible in [Runehaven](https://store.steampowered.com/app/2676030/) mods. Compiled to fill in the gaps between the example modpack and the documentation that's still being written.

→ **[Read the reference doc](docs/modding-reference.md)**

## What's in it

A schema for every modpack category (items, mobs, props, projectiles, modifiers, biomes, themes, worlds, zones, nodes, particle systems), the rune effect registry, the L-system zone grammar, the node object types, the full asset registry of vanilla IDs you can reference from a mod, and an honest "what's data-moddable vs what needs code" section.

## Quick navigation

- [Mod folder structure and `mod.json`](docs/modding-reference.md#modjson-modpack)
- [Items, weapons, runes-as-items](docs/modding-reference.md#itemsjson-item)
- [Mobs and AI archetypes](docs/modding-reference.md#mobsjson-mob)
- [Props and interactive types](docs/modding-reference.md#propsjson-prop)
- [Projectiles](docs/modding-reference.md#projectilesjson-projectile)
- [Item modifiers](docs/modding-reference.md#modifiersjson-itemmodifier)
- [Biomes, themes, worlds, zones](docs/modding-reference.md#biomesjson-biome2)
- [The `.node` room format](docs/modding-reference.md#nodesnode-datafile-format)
- [The 33-class rune effect registry](docs/modding-reference.md#rune-effect-registry-33-classes-partial-item-availability)
- [Console commands](docs/modding-reference.md#console-commands-f11)
- [Vanilla asset registry](docs/modding-reference.md#vanilla-asset-registry)
- [What's NOT moddable without code](docs/modding-reference.md#what-is-not-moddable-without-code-bepinex)

## How this was compiled

A mix of sources, in order of authority:

- The shipped `mod_example` modpack and the example JSONs the dev has shared
- The vanilla JSONs that ship as loose files in `StreamingAssets/`
- The IL2CPP decompile of `GameAssembly.dll` (via Cpp2IL → diffable C# stubs), which reveals class structures, field names, and enum values
- In-game console probing (`add_item <guess>`) to confirm which IDs are actually wired up
- Public Discord conversations with the dev, where they confirmed specific behaviors

The doc distinguishes confirmed facts, inferred behavior, and open questions throughout. When you see an "untested" or "likely" tag, that's a place where the schema exists in code but hasn't been exercised end-to-end.

## Status

This is a working draft. The modding system is also a work in progress, so things will shift. The doc is updated as new findings come in. If you find something that's wrong or missing, open an issue or a PR. Specific fixes welcome: a confirmed ID that the doc lists as untested, a field the doc missed, a corrected value range, anything.

## License

MIT. See [LICENSE](LICENSE). The reference content is shared under the same terms, fork it, contribute back, use it however helps you build things.

## Credits

Runehaven is by [Cikoria Studio](https://store.steampowered.com/developer/cikoria). The architecture is theirs and it's good, this is just notes from someone who wanted to understand it.

Compiled by [Ben Dixon](https://bendixon.dev). Reach out if you want to talk modding.
