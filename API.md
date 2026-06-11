# DonutSell Developer API

The DonutSell API lets other plugins query prices, categories and seller statistics, sell items
programmatically, listen to sell-related events, and control the global sell-event.

- **Package:** `net.luxcube.minecraft.api`
- **Entry point:** `DonutSellProvider.get()` → `DonutSellAPI`
- **Events:** `net.luxcube.minecraft.api.event.*`
- **Artifact:** `donut-sell-<version>-api.jar` (compile-time only; the real classes ship inside the DonutSell plugin at runtime)

---

## 1. Setup

### 1.1 Add the API jar to your build

The `-api` jar contains only the API classes — add it as a **compile-only** dependency. It must
**not** be shaded into your plugin; the implementation is provided by the DonutSell plugin at runtime.

**Gradle**
```groovy
repositories {
    mavenCentral()
    maven { url = "https://repo.papermc.io/repository/maven-public/" } // for paper-api
}

dependencies {
    compileOnly files("libs/donut-sell-1.5.0-api.jar")
    compileOnly "io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT"
    compileOnly "com.github.MilkBowl:VaultAPI:1.7" // only if you call getEconomy()
}
```

**Maven**
```xml
<dependency>
    <groupId>net.luxcube</groupId>
    <artifactId>donut-sell</artifactId>
    <version>1.5.0</version>
    <classifier>api</classifier>
    <scope>provided</scope>
</dependency>
```

### 1.2 Declare the dependency in `plugin.yml`

So your plugin loads **after** DonutSell and the API is ready by the time you use it:

```yaml
name: MyPlugin
main: com.example.MyPlugin
version: 1.0.0
depend: [DonutSell]        # hard dependency — your plugin won't load without DonutSell
# softdepend: [DonutSell]  # use this instead if DonutSell is optional
```

---

## 2. Getting the API

Obtain the singleton through `DonutSellProvider`. The API becomes available once DonutSell has
enabled.

```java
import net.luxcube.minecraft.api.DonutSellAPI;
import net.luxcube.minecraft.api.DonutSellProvider;

// Throws IllegalStateException if DonutSell isn't enabled yet:
DonutSellAPI api = DonutSellProvider.get();

// Or, the safe variants for softdepend:
if (DonutSellProvider.isAvailable()) {
    DonutSellAPI api2 = DonutSellProvider.get();
}

DonutSellAPI maybe = DonutSellProvider.getOrNull(); // null if not enabled

// Shortcut equivalent to DonutSellProvider.get():
DonutSellAPI same = DonutSellAPI.get();
```

The API is also registered with Bukkit's `ServicesManager`:

```java
DonutSellAPI api = getServer().getServicesManager().load(DonutSellAPI.class);
```

> **Tip:** with `softdepend`, always guard with `isAvailable()` / `getOrNull()`. With `depend`,
> `DonutSellProvider.get()` is safe to call from `onEnable()` onward.

---

## 3. Core concepts

| Term | Meaning |
|------|---------|
| **Category** (`CategoryModel`) | A group of sellable materials with its own pricing/leveling. |
| **Seller** (`SellerEntity`) | A player's DonutSell profile: per-category levels and lifetime stats. Only **loaded** sellers (usually online players) are returned. |
| **Unit price** | Price of one item, before amount and the player's category-level multiplier. |
| **Sell value** | `unit price × amount × category-level multiplier`. Excludes any active global sell-event boost. |
| **Global sell-event** | A temporary server-wide payout boost (e.g. 2× for 5 minutes). |

---

## 4. Querying prices (read-only)

These never pay out or change state — safe to call freely (e.g. to preview a value in your own GUI).

```java
Player player = ...;
ItemStack stack = new ItemStack(Material.DIAMOND, 64);

double unit  = api.getUnitPrice(player, stack);          // price of ONE diamond for this player
double value = api.getSellValue(player, stack);          // full value of the 64 stack
double total = api.getSellValue(player, player.getInventory()); // value of everything sellable

if (api.isSellable(Material.DIAMOND)) { /* ... */ }
if (api.isSellable(stack))            { /* ... */ }
```

---

## 5. Selling programmatically

All `sell*` methods value the items, fire a cancellable `PlayerSellEvent`, pay the player through
Vault, then fire `PlayerSoldEvent`. They return a `SellResult`.

```java
SellResult result = api.sellAll(player);                 // everything in their inventory
// SellResult r = api.sellHand(player);                  // item in main hand
// SellResult r = api.sellInventory(player, someChest);  // sellables in any Inventory (removed from it)
// SellResult r = api.sellItem(player, stack);           // a stack you already hold (inventory untouched)

if (result.isSuccess()) {
    player.sendMessage("Sold " + result.getItemCount() + " items for $" + result.getIncome());
    Map<Material, Integer> sold = result.getSoldItems();
} else if (result.isCancelled()) {
    player.sendMessage("Your sale was blocked.");
} else { // result == SellResult.empty()
    player.sendMessage("Nothing to sell.");
}
```

### `SellResult`

| Method | Returns |
|--------|---------|
| `isSuccess()` | `true` if items were sold and paid out. |
| `isCancelled()` | `true` if a `PlayerSellEvent` vetoed the sale. |
| `getIncome()` | Final money paid to the player. |
| `getItemCount()` | Total number of items sold. |
| `getSoldItems()` | Unmodifiable `Map<Material, Integer>` of what was sold. |

> **Note:** `sellInventory` / `sellHand` / `sellAll` remove sold items from the inventory.
> `sellItem` does **not** touch any inventory — use it when you already hold the stack.

---

## 6. Sellers & statistics

Only loaded sellers are returned; expect `null` / defaults for offline players.

```java
SellerEntity seller = api.getSeller(player);             // or getSeller(uuid)
Collection<SellerEntity> online = api.getLoadedSellers();

double lifetime = api.getTotalMoneyMade(player.getUniqueId());
int level       = api.getCategoryLevel(player.getUniqueId(), category); // -1 if not loaded
long serverTotal = api.getTotalSoldItems();              // server-wide items ever sold
```

---

## 7. Categories

```java
Collection<CategoryModel> all = api.getCategories();     // unmodifiable
CategoryModel ores = api.getCategory("ores");            // case-insensitive, null if absent
CategoryModel cat  = api.getCategoryOf(Material.DIAMOND); // null if not sellable

api.registerCategory(myCategory);                        // add at runtime
boolean removed = api.unregisterCategory("ores");
```

---

## 8. The global sell-event (payout boost)

```java
if (!api.isSellEventActive()) {
    api.startSellEvent();                 // configured defaults
    api.startSellEvent(300, 2.0);         // custom: 300 seconds at 2× boost
}

double boost = api.getSellEventBoost();   // 1.0 when no event is active
api.stopSellEvent();
```

`startSellEvent(...)` fires a cancellable `SellEventStartEvent` and returns `false` if an event was
already running or a listener cancelled it.

---

## 9. Events

Listen with a normal Bukkit `Listener`. Register it the usual way:
`getServer().getPluginManager().registerEvents(new MyListener(), this);`

> ⚠️ **Threading:** DonutSell events auto-detect their thread. Because commands run on an async
> scheduler, **your handler may be called asynchronously.** Never touch non-thread-safe Bukkit state
> directly from a handler — schedule back onto the main/region thread first
> (`Bukkit.getScheduler().runTask(plugin, ...)`).

### `PlayerSellEvent` — cancellable, fires *before* payout

Adjust the payout or veto the sale.

```java
@EventHandler
public void onPreSell(PlayerSellEvent event) {
    Player player = event.getPlayer();
    Map<Material, Integer> items = event.getItems();   // unmodifiable
    int count        = event.getItemCount();
    double base      = event.getBaseIncome();          // value before your multiplier
    double projected = event.getProjectedIncome();     // base × current multiplier

    event.setMultiplier(event.getMultiplier() * 1.5);  // +50% for this sale
    // event.setCancelled(true);                       // block the sale entirely
}
```

| Method | Description |
|--------|-------------|
| `getItems()` | Materials/amounts about to be sold (unmodifiable). |
| `getItemCount()` | Total item count. |
| `getBaseIncome()` | Income before the multiplier. |
| `getMultiplier()` / `setMultiplier(double)` | Your payout multiplier (default `1.0`). |
| `getProjectedIncome()` | `baseIncome × multiplier`. |
| `isCancelled()` / `setCancelled(boolean)` | Veto the sale. |

### `PlayerSoldEvent` — fires *after* a successful sale (not cancellable)

```java
@EventHandler
public void onSold(PlayerSoldEvent event) {
    double paid = event.getIncome();
    Map<Material, Integer> sold = event.getItems();
    if (event.isSellEventActive()) {
        double boost = event.getSellEventBoost();
    }
}
```

Methods: `getPlayer()`, `getItems()`, `getItemCount()`, `getIncome()`, `isSellEventActive()`,
`getSellEventBoost()`.

### `SellerLevelUpEvent` — a player leveled up a category

```java
@EventHandler
public void onLevelUp(SellerLevelUpEvent event) {
    Player player = event.getPlayer();
    CategoryModel category = event.getCategory();
    int from = event.getPreviousLevel();
    int to   = event.getNewLevel();
    int gained = event.getLevelsGained();
}
```

### `SellEventStartEvent` — cancellable, before a global sell-event starts

```java
@EventHandler
public void onSellEventStart(SellEventStartEvent event) {
    boolean auto = event.isAutomatic();            // scheduled vs. manually triggered
    event.setDurationSeconds(600);                 // override duration
    event.setBoost(3.0);                           // override boost
    // event.setCancelled(true);                   // prevent it from starting
}
```

Methods: `getDurationSeconds()`/`setDurationSeconds(int)`, `getBoost()`/`setBoost(double)`,
`isAutomatic()`, `isCancelled()`/`setCancelled(boolean)`.

### `SellEventEndEvent` — a global sell-event ended (not cancellable)

```java
@EventHandler
public void onSellEventEnd(SellEventEndEvent event) {
    int seconds = event.getDurationSeconds();
    double boost = event.getBoost();
}
```

---

## 10. Full example plugin

```java
package com.example;

import net.luxcube.minecraft.api.DonutSellAPI;
import net.luxcube.minecraft.api.DonutSellProvider;
import net.luxcube.minecraft.api.SellResult;
import net.luxcube.minecraft.api.event.PlayerSellEvent;
import net.luxcube.minecraft.api.event.SellerLevelUpEvent;
import org.bukkit.command.*;
import org.bukkit.entity.Player;
import org.bukkit.event.*;
import org.bukkit.plugin.java.JavaPlugin;

public final class MyPlugin extends JavaPlugin implements Listener {

    private DonutSellAPI sell;

    @Override
    public void onEnable() {
        if (!DonutSellProvider.isAvailable()) {
            getLogger().warning("DonutSell not found — disabling.");
            getServer().getPluginManager().disablePlugin(this);
            return;
        }
        this.sell = DonutSellProvider.get();
        getServer().getPluginManager().registerEvents(this, this);
    }

    // /sellhand — sell the held item via the API
    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        if (!(sender instanceof Player player)) return true;
        SellResult result = sell.sellHand(player);
        player.sendMessage(result.isSuccess()
            ? "Sold for $" + result.getIncome()
            : "Nothing to sell.");
        return true;
    }

    // VIPs earn 25% more
    @EventHandler
    public void onPreSell(PlayerSellEvent event) {
        if (event.getPlayer().hasPermission("myplugin.vip")) {
            event.setMultiplier(event.getMultiplier() * 1.25);
        }
    }

    @EventHandler
    public void onLevelUp(SellerLevelUpEvent event) {
        event.getPlayer().sendMessage(
            "You reached level " + event.getNewLevel() + " in " + event.getCategory().getName() + "!");
    }
}
```

---

## 11. API reference summary

**`DonutSellProvider`** — `get()`, `getOrNull()`, `isAvailable()`

**`DonutSellAPI`**
- Access: `getPlugin()`, `getEconomy()`
- Categories: `getCategories()`, `getCategory(name)`, `getCategoryOf(material)`, `isSellable(material/item)`, `registerCategory(c)`, `unregisterCategory(name)`
- Pricing: `getUnitPrice(player, item)`, `getSellValue(player, item)`, `getSellValue(player, inventory)`
- Selling: `sellItem`, `sellInventory`, `sellHand`, `sellAll`
- Sellers: `getSeller(uuid/player)`, `getLoadedSellers()`, `getTotalMoneyMade(uuid)`, `getCategoryLevel(uuid, category)`, `getTotalSoldItems()`
- Sell-event: `isSellEventActive()`, `getSellEventBoost()`, `startSellEvent()`, `startSellEvent(seconds, boost)`, `stopSellEvent()`

**Events:** `PlayerSellEvent` (cancellable), `PlayerSoldEvent`, `SellerLevelUpEvent`,
`SellEventStartEvent` (cancellable), `SellEventEndEvent`.

> `DonutSellEvents` and the `@ApiStatus.Internal` provider methods are internal — do not call them.
