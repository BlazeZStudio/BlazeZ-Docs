# Nemesis

Packet-level world security built on PacketEvents. Nemesis hides what cheats rely
on — ores, containers, underground caves and off-screen entities — by rewriting
chunk and entity packets on the way to each client, so vanilla players see nothing
different while X-ray, freecam and ESP/radar see obfuscated data.

Everything is toggled and tuned live from the `/nemesis settings` GUI, then
persisted back to `config.yml`.

## Features

- **Anti-XRay** — obfuscates ores and containers into surrounding stone (per-world,
  per-block `transformations`). Ores reveal only when a player legitimately sees
  them, with options to keep them visible afterwards and reveal the whole vein.
- **Anti-Freecam** — fills the world below a `hide-level` with solid blocks (and
  optionally strips light), revealing again above `reveal-level`. Per-world levels
  and fill blocks, with a fall-damage guard for the hidden layer.
- **Hide Entities (Anti-ESP / Radar)** — hides entities a player can't actually see
  (line-of-sight checked from the eye and/or third-person view) and reveals them
  when looked at. Configurable entity list and re-hide behaviour.
- **Hide Structures** *(experimental)* — hides whitelisted structures below a level.
- **Vision modes** — `performance` / `medium` / `hard` presets trade CPU for
  refresh rate and render distance.
- **Integrations** — WorldEdit / FAWE awareness (so `//cut` etc. update the cache),
  a WorldGuard/NPC-friendly soft-depend set, a Bedrock-ignore option, and an
  optional Discord `webhook`.

## Commands

Base command `/nemesis`. No aliases.

| Command | Description | Permission |
| --- | --- | --- |
| `/nemesis` | Show the command help. | `nemesis.admin` |
| `/nemesis settings` | Open the Vision Module GUI (toggles and settings for every feature). | `nemesis.admin` |
| `/nemesis reload` | Reload `config.yml` and `gui.yml`. | `nemesis.admin` |

> The `messages.anti-esp-*` keys mention an `/antiesp` command — that's legacy
> text; only `/nemesis` is registered.

## Permissions

| Permission | Effect |
| --- | --- |
| `nemesis.admin` | Use `/nemesis` and all subcommands. |
| `nemesis.vision.bypass.hide-entities` | This player is **not** subject to entity hiding (others' entities are always shown to them). |

## Files

| File | Purpose |
| --- | --- |
| `config.yml` | All settings (below). |
| `gui.yml` | The `/nemesis settings` menus. |

## config.yml

### General

| Key | Description |
| --- | --- |
| `license-key` | Your Lukittu license key (leave the placeholder for the build to inject). |
| `addons` | Reminder text about enabling addon licenses. |
| `bstats` | Send anonymous bStats metrics. |
| `debug.storage` | Verbose storage logging. |
| `blacklisted-worlds` | Worlds where anti-xray does **not** run. |
| `bedrock-ignore.enabled` / `bedrock-prefix` | Skip obfuscation for Bedrock players (matched by the floodgate prefix). |
| `webhook` | Discord webhook URL (empty = off). |

### Anti-XRay

| Key | Description |
| --- | --- |
| `anti-xray.enabled` | Master switch for ore/container obfuscation. |
| `hidden-block-types` | Block types hidden from the map (ores, containers, spawners…). |
| `transformations.environment-default` | What hidden blocks become per dimension (`normal`, `nether`, `the_end`). |
| `transformations.other` | Per-block overrides (e.g. `CHEST: AIR`, `DIAMOND_ORE: STONE`). |
| `keep-ores-in-view` | Once revealed, keep an ore visible until you're ~3 chunks away. |
| `reveal-whole-ore-stack` | Revealing one ore reveals the connected vein. |
| `max-hide-section` | How many 16-block sections (from the bottom up) obfuscation affects. |
| `world-edit-support` | Sync the cache with WorldEdit/FAWE edits (see the config note for the FAWE allow-list entry). |

### Vision mode

| Key | Description |
| --- | --- |
| `vision-mode.selected-mode` | Active preset: `performance`, `medium` or `hard`. |
| `vision-mode.modes.<mode>.refresh-rate-milliseconds` | How often nearby ores are re-checked. |
| `vision-mode.modes.<mode>.chunk-render-distance` | Chunk radius scanned for ores. |

### Anti-Freecam

| Key | Description |
| --- | --- |
| `anti-freecam.enabled` | Master switch. |
| `cancel-block-update-below` | Suppress block updates below the hide level. |
| `hide-all-sections` | Hide every section (under development). |
| `rehide-section` | Re-hide after the player drops back below the reveal level. |
| `vertical.*` | *(beta)* Vertical hiding: `chunk-distance`, `max-hide-section`, `refresh-time-in-seconds`, `fill-out-block`. |
| `remove-light-below-hide-level` | Strip light below the hide level. |
| `lower-chunk-distance-underground.*` | Lower the client view distance underground (`enabled`, `distance`). |
| `cancel-fall-damage-on-hidden-layer` | Cancel fall damage when a player lands on the hidden layer (lag safety). |
| `default-enabled` | If off, anti-freecam runs **only** in the listed `worlds`. |
| `default.hide-level` / `reveal-level` / `fill-out` | Fallback levels and fill block. |
| `worlds.<n>.world/hide-level/reveal-level/fill-out` | Per-world overrides (delete a section to disable it there). |

### Hide structures (experimental)

| Key | Description |
| --- | --- |
| `hide-structure.enabled` | Master switch. |
| `hide-structure.highest-hide-level` | Highest Y at which structures are hidden. |
| `hide-structure.whitelist` | Structure types to hide (e.g. `BURIED_TREASURE`, `MINESHAFT`, `STRONGHOLD`). |

### Hide entities (Anti-ESP / Radar)

| Key | Description |
| --- | --- |
| `hidden-entity-types` | Entity types eligible to be hidden when off-screen. |
| `disable-hide-entity-completely` | Turn entity hiding off entirely. |
| `re-hide-entities` | Re-hide entities once they leave view (not recommended on PvP servers). |
| `hide-entity-check-types` | Which views count as "seen": `PLAYER_EYE_VIEW`, `THIRD_PERSON_VIEW` (both on by default). |

### Messages

`messages.*` — player-facing text. Placeholder: `%ms%` (reload time). Keys:
`anti-esp-reload`, `anti-esp-toggle-on`, `anti-esp-toggle-off`, `anti-esp-usage`.

## gui.yml

| Menu | Purpose |
| --- | --- |
| `vision-module-inventory` | Main `/nemesis settings` menu — toggles (Anti-XRay, Anti-Freecam, Hide Structures, Keep Ores) and entries into the settings sub-menus (Vision Mode, Hide Level, Blocks, Entities, General, Anti-XRay, Anti-Freecam, Transformations, Worlds). |
| `setting-inventory` | The General settings sub-menu. |
| `setting-entity-inventory` | The hidden-entities editor. |
| `setting-blocks-inventory` | The hidden-blocks editor. |

Icons use `type`, `slot`, `name`, `lore`. Lore placeholders include `%enabled%`,
`%mode%`, `%refresh_rate%`, `%chunk_distance%`, `%level%`, `%block_count%` and
`%entity_count%`. Colours use `&` codes.

## Dependencies

**Required:** PacketEvents. **Optional (soft-depend):** PlaceholderAPI, WorldEdit
or FastAsyncWorldEdit (for `world-edit-support`), WorldGuard, ServersNPC,
FancyNpcs, GSit. Folia is supported.
