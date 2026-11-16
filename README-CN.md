# DreamRepair 开发者 API

> API 版本：`1.0.0`  
> 插件版本：`1.0.1`

**简体中文** · [English](API-en.md)

DreamRepair 提供一套公开 API，方便其他插件：

- 查询“这件物品修好要多少钱”
- 直接修理（扣款 / 免费）
- 监听报价、修理前、修理后事件
- 做铁匠 NPC、修理魔杖、GUI、活动折扣等功能
- NPC 收益分成：owner（召唤者）拿一部分，系统回收剩余部分
- NPC 有效期：过期后自动拒绝报价/修理（`NPC_EXPIRED`）

API 包：`plugins.xcmc.top.api`

---

## 1. 获取 API

### 1.1 编译依赖

把 DreamRepair 的 jar 加入编译 classpath（`provided` 即可，运行时由服务器提供）：

```xml
<dependency>
    <groupId>top.xcmc.plugins</groupId>
    <artifactId>DreamRepair</artifactId>
    <version>1.0.1</version>
    <scope>provided</scope>
</dependency>
```

如果本地 Maven 仓库没有该构件，也可以直接把 `DreamRepair-1.0.1.jar` 作为 IDE 的 `provided` / `compileOnly` 库。

> 请勿把 `plugins.xcmc.top.api` 下的类 shade/内联到自己的插件里；请使用 `provided` 依赖，确保调用方与 DreamRepair 注册的是同一个 API 接口 Class。

### 1.2 运行时获取

```java
import plugins.xcmc.top.api.DreamRepairAPI;

DreamRepairAPI api = DreamRepairAPI.get();
if (api == null || !api.isAvailable()) {
    // DreamRepair 未安装、未启用，或尚未完成加载
    return;
}
```

等价写法：

```java
DreamRepairAPI api = Bukkit.getServicesManager().load(DreamRepairAPI.class);
```

建议在插件的 `plugin.yml` 中声明软依赖，避免加载顺序问题：

```yaml
softdepend: [DreamRepair]
```

如果希望“没有 DreamRepair 也能正常启动”，使用 `softdepend` + 每次调用前判空；如果必须依赖，使用 `depend: [DreamRepair]`。

---

## 2. 快速开始

### 2.1 查询主手物品报价

```java
import plugins.xcmc.top.api.RepairQuote;
import plugins.xcmc.top.api.RepairSource;

RepairQuote quote = api.quoteHeldItem(player, RepairSource.NPC);

if (!quote.isValid()) {
    // quote.getFailureReason() 说明原因：
    // EMPTY_HAND / NOT_REPAIRABLE / NO_DAMAGE / UNBREAKABLE / CANCELLED ...
    return;
}

player.sendMessage("修理费用：" + quote.getCost() + " " + quote.getCurrency());
player.sendMessage("损耗耐久：" + quote.getDamage() + " 点");
```

### 2.2 立即修理主手物品

```java
import plugins.xcmc.top.api.RepairResult;

RepairResult result = api.repairHeldItem(player, true, RepairSource.NPC);

if (result.isSuccess()) {
    player.sendMessage("修理完成，扣款 " + result.getCost() + " " + result.getCurrency());
} else {
    player.sendMessage("修理失败：" + result.getFailureReason());
}
```

- `repairHeldItem` 会把修好的物品写回玩家主手。
- `repair(player, item, charge, source)` 只修改传入的 `ItemStack`，**不会**自动写回主手；适合 GUI / 背包物品，由调用方负责写回。
- `charge = true` 走 Vault 扣款；`charge = false` 表示免费修理。
- API 立即修理，不会弹出 DreamRepair 的二次确认；需要确认的场景请先 `quote`，由调用方实现确认，再 `repair`。

---

## 3. API 方法

| 方法 | 说明 |
|---|---|
| `getApiVersion()` | API 版本，例如 `1.0.0` |
| `isAvailable()` | DreamRepair 是否已启用并注册 API |
| `quote(player, item, source)` | 为指定物品报价 |
| `quote(player, item, source, share)` | 带 owner 分成 / NPC 有效期的报价 |
| `quoteHeldItem(player, source)` | 为主手物品报价 |
| `quoteHeldItem(player, source, share)` | 带 owner 分成 / NPC 有效期的主手报价 |
| `repair(player, item, charge, source)` | 修理指定物品（不写回主手） |
| `repair(player, item, charge, source, share)` | 带 owner 分成的修理 |
| `repairHeldItem(player, charge, source)` | 修理主手物品并写回 |
| `repairHeldItem(player, charge, source, share)` | 带 owner 分成的主手修理 |
| `calculateCost(int damage)` | 按当前配置计算费用 |
| `isEconomyAvailable()` | Vault 经济是否可用 |
| `getCurrencyName()` | 当前货币名（Coins / 金币） |
| `getLanguageCode()` | 当前语言码（en_us / zh_cn） |
| `getPluginVersion()` | DreamRepair 插件版本 |

`source` 可传 `RepairSource.COMMAND / API / NPC / OTHER`；不传时默认 `API`。使用 `NPC` 后，事件监听器可以据此给 NPC 交互做折扣、统计或额外逻辑。

---

## 4. RepairQuote

`RepairQuote` 是一次报价的不可变快照：

| 方法 | 说明 |
|---|---|
| `isValid()` | 是否可以修理 |
| `getFailureReason()` | 失败原因，见 `RepairFailureReason` |
| `getSource()` | 来源 |
| `getMaterial()` | 物品材质 |
| `getItemName()` | 展示用物品名，可能包含颜色代码 |
| `getDamage()` | 修复前损耗点数 |
| `getCost()` | 费用 |
| `getCurrency()` | 货币名 |
| `isEconomyAvailable()` | 报价时 Vault 是否可用 |
| `getShare()` | 收益分成 / NPC 有效期 |
| `getOwnerAmount()` | owner 应得金额 |
| `getSystemAmount()` | 系统保留金额 |

`RepairFailureReason` 常见值：

`NONE`、`EMPTY_HAND`、`NOT_REPAIRABLE`、`NO_DAMAGE`、`UNBREAKABLE`、`ECONOMY_UNAVAILABLE`、`INSUFFICIENT_FUNDS`、`WITHDRAW_FAILED`、`CANCELLED`、`NPC_EXPIRED`、`INVALID_ITEM`、`PLUGIN_DISABLED`、`UNKNOWN`

---

## 5. RepairResult

`RepairResult` 是一次修理调用的结果：

| 方法 | 说明 |
|---|---|
| `isSuccess()` | 是否修理成功 |
| `getFailureReason()` | 失败原因；成功时为 `NONE` |
| `getQuote()` | 本次报价快照 |
| `getCost()` | 报价费用（免费修理也保留原报价） |
| `getDamage()` | 修复前损耗 |
| `getCurrency()` | 货币名 |
| `isCharged()` | 是否实际扣款 |
| `getBalance()` | 操作后余额；经济不可用时为 0 |
| `getShare()` | 本次收益分成 / NPC 有效期 |
| `getOwnerAmount()` | 实际打给 owner 的金额（未支付为 0） |
| `getSystemAmount()` | 系统实际保留金额 |
| `isOwnerPaid()` | owner 分成是否成功到账 |

注意：`charge = false` 时 `isCharged()` 为 false，但 `getCost()` 仍可能是原报价金额；用 `isCharged()` 判断是否真的扣了钱。

---

## 6. NPC 收益分成与有效期

`RepairShare` 用来描述“谁召唤了 NPC、owner 拿多少、NPC 什么时候过期”。

```java
import plugins.xcmc.top.api.RepairShare;

import java.util.concurrent.TimeUnit;

long expiresAt = System.currentTimeMillis() + TimeUnit.DAYS.toMillis(days);
RepairShare share = RepairShare.of(ownerId, ownerName, 30.0, expiresAt);
```

- `ownerPercent`：0~100；系统获得剩余部分。
- `expiresAt`：epoch 毫秒；0 表示不过期。过期后报价/修理返回 `NPC_EXPIRED`。
- 没有 owner（`RepairShare.none()`）时系统获得 100%。
- **系统份额不会存入任何玩家账户**，相当于金币回收（sink）；如果服务器需要“系统账户”，可以后续用 owner 机制扩展。
- `ownerId` 为召唤者 UUID；`ownerName` 只用于展示/日志。

### 6.1 报价与修理

```java
import plugins.xcmc.top.api.RepairQuote;
import plugins.xcmc.top.api.RepairResult;
import plugins.xcmc.top.api.RepairSource;

RepairQuote quote = api.quoteHeldItem(player, RepairSource.NPC, share);
if (!quote.isValid()) {
    player.sendMessage("无法修理：" + quote.getFailureReason());
    return;
}

player.sendMessage("总费用：" + quote.getCost() + " " + quote.getCurrency());
player.sendMessage("NPC 主人可得：" + quote.getOwnerAmount());
player.sendMessage("系统回收：" + quote.getSystemAmount());

RepairResult result = api.repairHeldItem(player, true, RepairSource.NPC, share);
if (result.isSuccess()) {
    player.sendMessage("修理完成；owner 到账：" + result.getOwnerAmount()
            + "，系统回收：" + result.getSystemAmount()
            + "，owner 是否到账：" + result.isOwnerPaid());
}
```

行为说明：

- 玩家先被扣除**全额费用**，然后 DreamRepair 把 owner 份额存入召唤者账户。
- owner 离线也可以收款（Vault `depositPlayer(OfflinePlayer, amount)`）。
- owner 就是付款玩家时：先扣全额，再返还 owner 份额，净效果是只付了系统份额。
- 如果 owner 打款失败：`isOwnerPaid()` 为 false，`getOwnerAmount()` 为 0，`getSystemAmount()` 为全额；金币不会凭空增加，留在系统。
- `charge = false` 的免费修理不会产生 owner/系统收益。
- NPC 的生成、存活天数展示、到期删除仍由魔杖插件负责；DreamRepair 只负责在交易时校验 `expiresAt` 并拒绝过期 NPC。

### 6.2 事件中的分成

```java
@EventHandler
public void onQuote(DreamRepairQuoteEvent event) {
    // 活动期间 NPC 主人只拿 10%
    event.setShare(RepairShare.of(ownerId, ownerName, 10.0, expiresAt));
}
```

`DreamRepairQuoteEvent` 还提供 `getOwnerAmount()` / `getSystemAmount()`，会随 `setCost` / `setShare` 动态计算。

`DreamRepairPostRepairEvent` 提供实际结果：`getOwnerAmount()`、`getSystemAmount()`、`isOwnerPaid()`。

---

## 7. 事件

所有事件都在调用 API 的线程同步触发。Folia 下请确保在玩家所属区域线程调用。

### 7.1 报价事件（可改价 / 可取消）

```java
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import plugins.xcmc.top.api.RepairSource;
import plugins.xcmc.top.api.event.DreamRepairQuoteEvent;

public class MyListener implements Listener {

    @EventHandler
    public void onQuote(DreamRepairQuoteEvent event) {
        if (event.getSource() == RepairSource.NPC) {
            // 铁匠 NPC 八折
            event.setCost(event.getCost() * 0.8);
        }
    }
}
```

- `setCost(0)` 可以做成免费。
- `setCancelled(true)` 会让报价变成 `CANCELLED`，`repair` 也不会继续。

### 7.2 修理前事件（可取消）

```java
import plugins.xcmc.top.api.event.DreamRepairPreRepairEvent;

@EventHandler
public void onPreRepair(DreamRepairPreRepairEvent event) {
    if (event.getPlayer().hasPermission("myserver.no-repair")) {
        event.setCancelled(true);
    }
}
```

取消后不会扣款，也不会修改耐久。

### 7.3 修理后事件

```java
import plugins.xcmc.top.api.event.DreamRepairPostRepairEvent;

@EventHandler
public void onPostRepair(DreamRepairPostRepairEvent event) {
    // 统计、成就、公告、额外奖励
    event.getPlayer().sendMessage("修理完成！");
}
```

---

## 8. 铁匠魔杖 / NPC 示例

DreamRepair 只负责“报价 + 修理”，**不负责生成 NPC、不负责到期删除**。NPC 的生成、存活时间和交互由调用方插件（Citizens、Sentinel、自定义实体等）管理。

下面是一个“铁匠魔杖召唤限时 NPC”的伪代码：

```java
// 1) 玩家使用魔杖：召唤 NPC，并记录召唤者与到期时间
UUID ownerId = summoner.getUniqueId();
String ownerName = summoner.getName();
long expiresAt = System.currentTimeMillis() + TimeUnit.DAYS.toMillis(days);
RepairShare share = RepairShare.of(ownerId, ownerName, 30.0, expiresAt);

// 2) 其他玩家右键 NPC
if (System.currentTimeMillis() > expiresAt) {
    despawnNpc();
    player.sendMessage("这位铁匠已经离开了。");
    return;
}

if (!player.hasPermission("dreamrepair.use")) {
    player.sendMessage("你没有使用修理服务的权限。");
    return;
}

DreamRepairAPI api = DreamRepairAPI.get();
if (api == null || !api.isAvailable()) {
    player.sendMessage("DreamRepair 暂不可用。");
    return;
}

RepairQuote quote = api.quoteHeldItem(player, RepairSource.NPC, share);
if (!quote.isValid()) {
    player.sendMessage("无法修理：" + quote.getFailureReason());
    return;
}

// 3) 展示价格，由 NPC 插件自己做确认（GUI / 第二次右键 / 对话）
player.sendMessage("修理 " + quote.getItemName()
        + " 需要 " + quote.getCost() + " " + quote.getCurrency()
        + "，损耗 " + quote.getDamage() + " 点。");
player.sendMessage("其中 " + quote.getOwnerAmount() + " 归铁匠主人，"
        + quote.getSystemAmount() + " 由系统回收。");

// 4) 玩家确认后
RepairResult result = api.repairHeldItem(player, true, RepairSource.NPC, share);
if (result.isSuccess()) {
    player.sendMessage("修理完成！铁匠主人到账：" + result.getOwnerAmount()
            + "，系统回收：" + result.getSystemAmount());
} else {
    player.sendMessage("修理失败：" + result.getFailureReason());
}
```

如果 owner 是付款玩家自己，DreamRepair 会先扣全额、再把 owner 份额返还，净效果是只付系统份额。如果 owner 打款失败，`result.isOwnerPaid()` 为 false，`getOwnerAmount()` 为 0，`getSystemAmount()` 为全额。

如果要让 NPC 提供“限时折扣”，监听 `DreamRepairQuoteEvent`，在 `source == NPC` 时修改 `cost` 或 `setShare` 即可。NPC 到期时间、所有者、召唤位置等信息建议由调用方插件持久化（例如 YAML / 数据库），DreamRepair 不保存这些状态；但交易时会把 `share.expiresAt` 作为最后一道过期校验。

---

## 9. 注意事项

- **权限**：API 不检查 `dreamrepair.use` / `dreamrepair.admin`。NPC、魔杖等入口请自行检查权限，避免绕过服务器的权限设计。
- **二次确认**：API 是立即修理。需要确认时，先 `quote` 展示费用，由调用方完成确认后再 `repair`。
- **线程**：所有方法都应在玩家所在线程调用。Folia 下是玩家所属区域线程；不要把 API 调用丢到异步线程。
- **物品写回**：`repairHeldItem` 会写回主手；`repair(player, item, ...)` 只修改传入的 `ItemStack`，GUI/背包场景由调用方写回对应槽位。
- **经济**：`charge = true` 且费用大于 0 时需要 Vault 经济可用；`charge = false` 可免费修理，不依赖经济。
- **NPC 收益**：owner 份额通过 Vault `depositPlayer(OfflinePlayer, amount)` 打入召唤者账户；系统份额直接留在系统，不进入任何玩家账户。owner 打款失败时全额留在系统，避免造币。
- **NPC 过期**：`RepairShare.expiresAt` 会在报价和修理时校验，过期返回 `NPC_EXPIRED`；仍建议魔杖插件自己及时删除 NPC。
- **事件**：事件监听器请勿抛异常；DreamRepair 会捕获并记录，但异常可能导致调用方逻辑不完整。
- **版本**：通过 `getApiVersion()` 判断 API 版本；未来若有不兼容改动会提升 API 版本号。
- **可选依赖**：其他插件建议使用 `softdepend: [DreamRepair]`，并在每次使用前判空，避免 DreamRepair 未安装时无法启动。

---

**简体中文** · [English](API-en.md)
