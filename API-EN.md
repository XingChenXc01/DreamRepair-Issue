# DreamRepair Developer API

> API version: `1.0.0`  
> Plugin version: `1.0.1`

[Chinese](API.md) · **English**

DreamRepair exposes a public API that lets other plugins:

- Quote the repair cost of an item
- Repair directly (charged or free)
- Listen to quote, pre-repair and post-repair events
- Build blacksmith NPCs, repair wands, GUIs, event discounts, etc.
- Split NPC revenue between the NPC owner and the system
- Enforce NPC expiry through `RepairShare.expiresAt`

API package: `plugins.xcmc.top.api`

---

## 1. Getting the API

### 1.1 Compile dependency

Add the DreamRepair jar to your compile classpath (`provided` is enough; the server provides it at runtime):

```xml
<dependency>
    <groupId>top.xcmc.plugins</groupId>
    <artifactId>DreamRepair</artifactId>
    <version>1.0.1</version>
    <scope>provided</scope>
</dependency>
```

If the artifact is not in your local Maven repository, add `DreamRepair-1.0.1.jar` as a `provided` / `compileOnly` library in your IDE instead.

> Do not shade or inline the `plugins.xcmc.top.api` classes into your own plugin. Use a `provided` dependency so your plugin and DreamRepair reference the same API interface class.

### 1.2 Runtime lookup

```java
import plugins.xcmc.top.api.DreamRepairAPI;

DreamRepairAPI api = DreamRepairAPI.get();
if (api == null || !api.isAvailable()) {
    // DreamRepair is missing, disabled, or still loading
    return;
}
```

Equivalent:

```java
DreamRepairAPI api = Bukkit.getServicesManager().load(DreamRepairAPI.class);
```

Declare a soft dependency in your `plugin.yml` to avoid load-order issues:

```yaml
softdepend: [DreamRepair]
```

If your plugin must work without DreamRepair, use `softdepend` and null-check the API before every use. If DreamRepair is mandatory, use `depend: [DreamRepair]`.

---

## 2. Quick Start

### 2.1 Quote the held item

```java
import plugins.xcmc.top.api.RepairQuote;
import plugins.xcmc.top.api.RepairSource;

RepairQuote quote = api.quoteHeldItem(player, RepairSource.NPC);

if (!quote.isValid()) {
    // quote.getFailureReason() explains why:
    // EMPTY_HAND / NOT_REPAIRABLE / NO_DAMAGE / UNBREAKABLE / CANCELLED / NPC_EXPIRED ...
    return;
}

player.sendMessage("Repair cost: " + quote.getCost() + " " + quote.getCurrency());
player.sendMessage("Durability lost: " + quote.getDamage());
```

### 2.2 Repair the held item

```java
import plugins.xcmc.top.api.RepairResult;

RepairResult result = api.repairHeldItem(player, true, RepairSource.NPC);

if (result.isSuccess()) {
    player.sendMessage("Repaired for " + result.getCost() + " " + result.getCurrency());
} else {
    player.sendMessage("Repair failed: " + result.getFailureReason());
}
```

- `repairHeldItem` writes the repaired item back to the player's main hand.
- `repair(player, item, charge, source)` only modifies the provided `ItemStack`; it does **not** write it back to the main hand. Use it for GUI/inventory items and write the stack back yourself.
- `charge = true` charges through Vault; `charge = false` repairs for free.
- The API repairs immediately and does **not** open DreamRepair's double-confirmation flow. If you need confirmation, call `quote` first, implement your own confirmation, then call `repair`.

---

## 3. API Methods

| Method | Description |
|---|---|
| `getApiVersion()` | API version, e.g. `1.0.0` |
| `isAvailable()` | Whether DreamRepair is enabled and the API is registered |
| `quote(player, item, source)` | Quote an item |
| `quote(player, item, source, share)` | Quote with owner split / NPC expiry |
| `quoteHeldItem(player, source)` | Quote the held item |
| `quoteHeldItem(player, source, share)` | Quote the held item with owner split / NPC expiry |
| `repair(player, item, charge, source)` | Repair an item (no main-hand write-back) |
| `repair(player, item, charge, source, share)` | Repair an item with owner split |
| `repairHeldItem(player, charge, source)` | Repair the held item and write it back |
| `repairHeldItem(player, charge, source, share)` | Repair the held item with owner split |
| `calculateCost(int damage)` | Calculate cost from current config |
| `isEconomyAvailable()` | Whether Vault is available |
| `getCurrencyName()` | Current currency name (e.g. Coins) |
| `getLanguageCode()` | Current language code (en_us / zh_cn) |
| `getPluginVersion()` | DreamRepair plugin version |

`source` can be `RepairSource.COMMAND / API / NPC / OTHER`; it defaults to `API`. Use `NPC` so event listeners can identify NPC/wand interactions and apply discounts, stats, or extra logic.

---

## 4. RepairQuote

`RepairQuote` is an immutable snapshot of one quote:

| Method | Description |
|---|---|
| `isValid()` | Whether the item can be repaired |
| `getFailureReason()` | Failure reason, see `RepairFailureReason` |
| `getSource()` | Request source |
| `getMaterial()` | Item material |
| `getItemName()` | Display item name; may contain color codes |
| `getDamage()` | Durability lost before repair |
| `getCost()` | Repair cost |
| `getCurrency()` | Currency name |
| `isEconomyAvailable()` | Whether Vault was available at quote time |
| `getShare()` | Revenue split / NPC expiry |
| `getOwnerAmount()` | Amount the owner should receive |
| `getSystemAmount()` | Amount the system should keep |

Common `RepairFailureReason` values:

`NONE`, `EMPTY_HAND`, `NOT_REPAIRABLE`, `NO_DAMAGE`, `UNBREAKABLE`, `ECONOMY_UNAVAILABLE`, `INSUFFICIENT_FUNDS`, `WITHDRAW_FAILED`, `CANCELLED`, `NPC_EXPIRED`, `INVALID_ITEM`, `PLUGIN_DISABLED`, `UNKNOWN`

---

## 5. RepairResult

`RepairResult` is the result of one repair call:

| Method | Description |
|---|---|
| `isSuccess()` | Whether the repair succeeded |
| `getFailureReason()` | Failure reason; `NONE` on success |
| `getQuote()` | Quote snapshot for this call |
| `getCost()` | Quoted cost (kept even for free repairs) |
| `getDamage()` | Durability lost before repair |
| `getCurrency()` | Currency name |
| `isCharged()` | Whether money was actually withdrawn |
| `getBalance()` | Player balance after the call; 0 when economy is unavailable |
| `getShare()` | Revenue split / NPC expiry for this call |
| `getOwnerAmount()` | Amount actually paid to the owner (0 if unpaid) |
| `getSystemAmount()` | Amount the system actually kept |
| `isOwnerPaid()` | Whether the owner share was paid successfully |

Note: with `charge = false`, `isCharged()` is false, but `getCost()` may still contain the original quote. Use `isCharged()` to check whether money was actually withdrawn.

---

## 6. NPC Revenue Split & Expiry

`RepairShare` describes who summoned the NPC, how much the owner receives, and when the NPC expires.

```java
import plugins.xcmc.top.api.RepairShare;

import java.util.concurrent.TimeUnit;

long expiresAt = System.currentTimeMillis() + TimeUnit.DAYS.toMillis(days);
RepairShare share = RepairShare.of(ownerId, ownerName, 30.0, expiresAt);
```

- `ownerPercent`: 0–100; the system receives the remainder.
- `expiresAt`: epoch milliseconds; 0 means no expiry. Expired shares cause quote/repair to fail with `NPC_EXPIRED`.
- `RepairShare.none()` gives the system 100%.
- **The system share is not deposited into any player account.** It is a coin sink. If your server needs a dedicated system account, this can be extended with an owner-like system account.
- `ownerId` is the summoner UUID; `ownerName` is only used for display/logging.

### 6.1 Quote and repair

```java
import plugins.xcmc.top.api.RepairQuote;
import plugins.xcmc.top.api.RepairResult;
import plugins.xcmc.top.api.RepairSource;

RepairQuote quote = api.quoteHeldItem(player, RepairSource.NPC, share);
if (!quote.isValid()) {
    player.sendMessage("Cannot repair: " + quote.getFailureReason());
    return;
}

player.sendMessage("Total cost: " + quote.getCost() + " " + quote.getCurrency());
player.sendMessage("NPC owner receives: " + quote.getOwnerAmount());
player.sendMessage("System keeps: " + quote.getSystemAmount());

RepairResult result = api.repairHeldItem(player, true, RepairSource.NPC, share);
if (result.isSuccess()) {
    player.sendMessage("Repaired. Owner paid: " + result.getOwnerAmount()
            + ", system kept: " + result.getSystemAmount()
            + ", owner paid successfully: " + result.isOwnerPaid());
}
```

Behavior:

- The player is charged the **full cost** first, then DreamRepair deposits the owner share into the summoner's account.
- Offline owners can still be paid (Vault `depositPlayer(OfflinePlayer, amount)`).
- If the owner is the paying player: the full cost is charged, then the owner share is returned; the net cost is the system share.
- If the owner payout fails: `isOwnerPaid()` is false, `getOwnerAmount()` is 0, and `getSystemAmount()` is the full cost. No money is created.
- `charge = false` free repairs do not generate owner or system income.
- NPC spawning, duration display, and despawn are still the wand plugin's responsibility. DreamRepair only validates `expiresAt` at transaction time and rejects expired NPCs.

### 6.2 Shares in events

```java
@EventHandler
public void onQuote(DreamRepairQuoteEvent event) {
    // During an event, the NPC owner only receives 10%
    event.setShare(RepairShare.of(ownerId, ownerName, 10.0, expiresAt));
}
```

`DreamRepairQuoteEvent` also provides `getOwnerAmount()` / `getSystemAmount()`, which are recalculated when `setCost` or `setShare` changes the quote.

`DreamRepairPostRepairEvent` exposes the actual result: `getOwnerAmount()`, `getSystemAmount()`, and `isOwnerPaid()`.

---

## 7. Events

All events are fired synchronously on the thread that calls the API. On Folia, call the API on the player's region thread.

### 7.1 Quote event (modify price / cancel)

```java
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import plugins.xcmc.top.api.RepairSource;
import plugins.xcmc.top.api.event.DreamRepairQuoteEvent;

public class MyListener implements Listener {

    @EventHandler
    public void onQuote(DreamRepairQuoteEvent event) {
        if (event.getSource() == RepairSource.NPC) {
            // 20% discount at the blacksmith NPC
            event.setCost(event.getCost() * 0.8);
        }
    }
}
```

- `setCost(0)` makes the repair free.
- `setCancelled(true)` turns the quote into `CANCELLED`; `repair` will not proceed.

### 7.2 Pre-repair event (cancellable)

```java
import plugins.xcmc.top.api.event.DreamRepairPreRepairEvent;

@EventHandler
public void onPreRepair(DreamRepairPreRepairEvent event) {
    if (event.getPlayer().hasPermission("myserver.no-repair")) {
        event.setCancelled(true);
    }
}
```

Cancelling it prevents the charge and the durability change.

### 7.3 Post-repair event

```java
import plugins.xcmc.top.api.event.DreamRepairPostRepairEvent;

@EventHandler
public void onPostRepair(DreamRepairPostRepairEvent event) {
    // Stats, achievements, announcements, extra rewards
    event.getPlayer().sendMessage("Repair complete!");
}
```

---

## 8. Blacksmith Wand / NPC Example

DreamRepair handles **quoting and repairing**. It does **not** spawn NPCs or delete them on expiry. NPC spawning, lifetime display, and interaction are managed by the caller's plugin (Citizens, Sentinel, custom entities, etc.).

Pseudo-code for a "blacksmith wand summons a temporary NPC" flow:

```java
// 1) The player uses the wand: summon the NPC and store the owner + expiry
UUID ownerId = summoner.getUniqueId();
String ownerName = summoner.getName();
long expiresAt = System.currentTimeMillis() + TimeUnit.DAYS.toMillis(days);
RepairShare share = RepairShare.of(ownerId, ownerName, 30.0, expiresAt);

// 2) Another player right-clicks the NPC
if (System.currentTimeMillis() > expiresAt) {
    despawnNpc();
    player.sendMessage("This blacksmith has already left.");
    return;
}

if (!player.hasPermission("dreamrepair.use")) {
    player.sendMessage("You do not have permission to use the repair service.");
    return;
}

DreamRepairAPI api = DreamRepairAPI.get();
if (api == null || !api.isAvailable()) {
    player.sendMessage("DreamRepair is temporarily unavailable.");
    return;
}

RepairQuote quote = api.quoteHeldItem(player, RepairSource.NPC, share);
if (!quote.isValid()) {
    player.sendMessage("Cannot repair: " + quote.getFailureReason());
    return;
}

// 3) Show the price; the NPC plugin handles its own confirmation
//    (GUI / second right-click / dialogue)
player.sendMessage("Repairing " + quote.getItemName()
        + " costs " + quote.getCost() + " " + quote.getCurrency()
        + " and will restore " + quote.getDamage() + " durability.");
player.sendMessage(quote.getOwnerAmount() + " goes to the blacksmith owner, "
        + quote.getSystemAmount() + " is kept by the system.");

// 4) After the player confirms
RepairResult result = api.repairHeldItem(player, true, RepairSource.NPC, share);
if (result.isSuccess()) {
    player.sendMessage("Repaired. Owner received: " + result.getOwnerAmount()
            + ", system kept: " + result.getSystemAmount());
} else {
    player.sendMessage("Repair failed: " + result.getFailureReason());
}
```

If the owner is the paying player, DreamRepair charges the full amount and then returns the owner share; the net effect is paying only the system share. If the owner payout fails, `result.isOwnerPaid()` is false, `getOwnerAmount()` is 0, and `getSystemAmount()` is the full cost.

To add a limited-time discount, listen to `DreamRepairQuoteEvent` and modify `cost` or call `setShare` when `source == NPC`. Persist the NPC's owner, expiry, and location in your own plugin (YAML / database); DreamRepair does not store that state, but it uses `share.expiresAt` as the final expiry check at transaction time.

---

## 9. Notes

- **Permissions**: The API does not check `dreamrepair.use` or `dreamrepair.admin`. NPC and wand entry points should check permissions themselves to avoid bypassing your server's permission design.
- **Confirmation**: The API repairs immediately. If you need confirmation, call `quote` first, implement your own confirmation, then call `repair`.
- **Threading**: Call all API methods on the player's thread. On Folia this is the player's region thread; do not call the API from an async thread.
- **Item write-back**: `repairHeldItem` writes back to the main hand; `repair(player, item, ...)` only modifies the provided `ItemStack`. GUI/inventory callers must write it back to the correct slot.
- **Economy**: `charge = true` with a cost above 0 requires Vault to be available; `charge = false` repairs for free and does not require the economy.
- **NPC revenue**: The owner share is deposited through Vault `depositPlayer(OfflinePlayer, amount)`; the system share is not deposited into any player account. If the owner payout fails, the full amount stays with the system to avoid creating money.
- **NPC expiry**: `RepairShare.expiresAt` is checked for both quote and repair, returning `NPC_EXPIRED` after expiry. Your wand plugin should still despawn the NPC promptly.
- **Events**: Do not throw from event listeners. DreamRepair catches and logs exceptions, but an exception may leave your own logic incomplete.
- **Versioning**: Use `getApiVersion()` to check the API version. Incompatible changes will bump the API version.
- **Optional dependency**: Prefer `softdepend: [DreamRepair]` and null-check the API before every use so your plugin still starts when DreamRepair is not installed.

---

[Chinese](API.md) · **English**
