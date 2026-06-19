# Shop

A category-based GUI shop. Each category is its own file under `categories/`,
opened from the main shop menu.

## Commands

All commands are player-only.

| Command | Description | Permission |
| --- | --- | --- |
| `/shop` | Open the shop menu. | — |
| `/shop reload` | Reload the shop config and categories. | `shop.reload` |
| `/shop admin` | Open the category editor GUI. | `shop.admin` |
| `/shopitem` | Show the `/shopitem` usage. | `donutshop.admin` |
| `/shopitem set <category> <price> [sell-price]` | Add the item in your hand to `<category>` with the given buy price and optional sell price. | `donutshop.admin` |

## Permissions

| Permission | Grants |
| --- | --- |
| `shop.reload` | `/shop reload`. |
| `shop.admin` | `/shop admin` (the category editor). |
| `donutshop.admin` | `/shopitem` and `/shopitem set`. |

## Files

| File | Purpose |
| --- | --- |
| `config.yml` | Main menu, category list and shared item lore. |
| `categories/<name>.yml` | Items for one category. |

## config.yml

| Key | Description |
| --- | --- |
| `shop-item-lore.buy` / `.sell` | Lore lines appended to every item showing buy/sell price. |
| `main-inventory.title` | Main menu title. |
| `main-inventory.layout` | Main menu layout (`O` category, `<`/`>` pages, `X` filler). |
| `main-inventory.add-stack-item` / `remove-stack-item` | The +/- amount buttons. |
| `categories.<name>` | A category: `rows`, `icon`, `inventory-name` and `layout`. |

## Category files

Each file under `categories/` lists the buyable/sellable items for that category —
their materials, prices and slots. Placeholders: `%price%`, `%amount%`.
