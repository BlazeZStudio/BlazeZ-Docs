# PrivateVaults

Private, **numbered** per-player vaults that follow players across every server in a
network. Vault contents live in **MySQL** (durable source of truth) and are served
through **Redis** (shared cache + a distributed lock that makes the vaults
**dupe-safe** across servers). A player opens a vault with `/pv <number>`; how many
vaults they get and how big each one is comes from permissions.

This file is rewritten every time the plugin loads, so it always matches the running
version. The shipped `config.yml` has **no comments** — this document is the reference
for every key instead.

## How it works

- Each player owns vaults numbered `1..N`. `N` (the amount) and the size (rows) of
  each vault are granted by permission, falling back to the `vaults.default-*` values.
- Opening a vault takes a **cross-server lock** in Redis keyed by owner + vault number.
  While the lock is held no other server can open that same vault, so a vault can never
  be open in two places at once — that is what prevents item duplication.
- Contents are read from MySQL (authoritative) when the vault opens and written back to
  MySQL + Redis when it closes. While a vault is open its lock TTL is refreshed and its
  contents are auto-saved on an interval, so a server crash loses at most one interval
  of edits and the lock frees itself automatically.

## Commands

Base command `/pv` (aliases: `/playervault`, `/playervaults`, `/pvault`, `/vault`).

| Command | Description | Permission |
| --- | --- | --- |
| `/pv` | Open the vault selector menu. | `vault.use` |
| `/pv <number>` | Open one of your own vaults. | `vault.use` + `vault.amount.<n>` |
| `/pv admin <player> [number]` | Open and edit another player's vault. | `vault.admin` |
| `/pv unlock <player> [number]` | Force-clear a stuck cross-server lock. | `vault.unlock` |
| `/pv import [overwrite]` | Import every vault from the PlayerVaults plugin. Add `overwrite` to replace vaults that already exist. | `vault.admin` |
| `/pv reload` | Reload `config.yml`. | `vault.reload` |

## Permissions

| Permission | Effect |
| --- | --- |
| `vault.use` | Open your own vaults. Default `true`. |
| `vault.amount.<n>` | You may use vaults `1..n`. The **highest** `n` you hold wins; falls back to `vaults.default-amount`. |
| `vault.size.<rows>` | Your vaults are `rows` tall (1-6). The **highest** `rows` you hold wins; falls back to `vaults.default-size`. |
| `vault.admin` | Open/edit any other player's vault. |
| `vault.unlock` | Manually clear a stuck lock. |
| `vault.reload` | Reload the config. |
| `vault.bypass.blacklist` | Store blacklisted materials anyway. |

> Vaults never shrink destructively: if a player's permitted size drops below the
> number of slots that already hold items, the vault opens just large enough to keep
> every item, so nothing is ever lost.

## Compatibility

Runs on Paper and on **Folia** (and forks like Leaf/Purpur) — all scheduling uses the
region-threaded scheduler API, so vault I/O happens off-thread and inventory work happens
on each player's region thread. Migrating from another vault plugin? See `/pv import`.

## Files

| File | Purpose |
| --- | --- |
| `config.yml` | All settings (below). No comments — see this file. |
| `private-vaults.md` | This document, re-saved on every load. |
| `vaults/` | Per-player vault files (only when `storage.type: local`). |

## config.yml

### Top level

| Key | Description |
| --- | --- |
| `license-key` | Your Lukittu license key for this product. |
| `server-name` | A unique name for this server in the network. Used as the lock owner so locks can be traced to a server. |

### `storage`

| Key | Description |
| --- | --- |
| `type` | `local`, `mysql`, `redis`, or `mysql+redis` (recommended for cross-server). |
| `mysql.host` / `port` / `database` / `username` / `password` | MySQL connection (used by `mysql` and `mysql+redis`). |
| `redis.host` / `port` / `password` | Redis connection (used by `redis` and `mysql+redis`). Leave `password` empty if unauthenticated. |

> **Which type?**
> - `local` — **single server, no database.** Vaults are stored in YAML files under
>   `plugin/vaults/` and locked in-memory. Zero setup, but **not** shared between servers.
> - `mysql+redis` — the cross-server default: MySQL keeps the data, Redis provides the
>   shared cache and the dupe-prevention lock.
> - `mysql` — also cross-server (uses a MySQL lock table) but with no cache.
> - `redis` — fast but not durable; use it only if you don't mind losing data when Redis
>   is wiped.
>
> Every type except `local` is cross-server: point each server at the same database and
> a vault can never be open on two servers at once.

### `vaults`

| Key | Description |
| --- | --- |
| `default-amount` | Vaults a player gets with no `vault.amount.<n>` permission. |
| `default-size` | Rows (1-6) a vault has with no `vault.size.<rows>` permission. |
| `title` | Inventory title of your own vaults. Placeholder: `%number%`. |
| `admin-title` | Inventory title when staff open someone's vault. Placeholders: `%player%`, `%number%`. |
| `selector-title` | Title of the `/pv` selector menu. |

### `cross-server`

| Key | Description |
| --- | --- |
| `lock-ttl-seconds` | How long a vault lock survives without a refresh (auto-frees a crashed server's lock). |
| `lock-refresh-seconds` | How often an open vault's lock TTL is extended. Keep it below `lock-ttl-seconds`. |
| `autosave-seconds` | How often open vaults are flushed to storage (crash resilience). |

### `blacklist`

| Key | Description |
| --- | --- |
| `enabled` | Block the listed materials from being placed into vaults. |
| `materials` | Bukkit material names that may not be stored (unless the player has `vault.bypass.blacklist`). |

### `sounds`

| Key | Description |
| --- | --- |
| `enabled` | Play a sound when a vault opens/closes. |
| `open` / `close` | Bukkit `Sound` enum names. |

### `buttons`

An optional button bar along the bottom row of the vault.

| Key | Description |
| --- | --- |
| `enabled` | Show the button bar. Default `false`. |
| `sort` | Show the **Sort** button (tidies and stacks the vault). |
| `deposit` | Show the **Deposit Matching** button (pulls items from your inventory that match what's already stored). |

> When the bar is shown the bottom row is reserved for buttons, so storage is capped at
> 5 rows. A vault that actually needs all 6 rows for its items simply opens without the
> bar — items are never lost.

### `audit`

Optional log of staff opening/editing other players' vaults.

| Key | Description |
| --- | --- |
| `enabled` | Turn auditing on. Default `false`. |
| `console` | Also print audit lines to the server console. |
| `webhook-url` | A Discord webhook URL; leave empty to disable Discord posts. |

### `placeholders`

| Key | Description |
| --- | --- |
| `enabled` | Register the PlaceholderAPI expansion when PAPI is installed. |

PlaceholderAPI placeholders: `%privatevaults_max_vaults%`, `%privatevaults_max_size%`,
`%privatevaults_default_size%`, `%privatevaults_server%`.

### `messages`

Every player-facing string. `%prefix%` expands to `messages.prefix`; colour codes use
`&` and `&#RRGGBB` hex. Placeholders per message: `%number%`, `%player%`, `%ms%`.

## License

This plugin is protected by the Lukittu licensing system. Set `license-key` in
`config.yml`; if verification fails on startup the plugin disables itself.
