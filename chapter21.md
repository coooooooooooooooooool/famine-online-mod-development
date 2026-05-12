# 第21章 战斗系统（Combat 全解）

## 21.1 架构基础：Master / Replica / Classified

## 21.1 架构基础：Master / Replica / Classified

### 本节导读

很多刚开始写战斗 mod 的玩家会被一个问题困住：**为什么我在客户端调 `inst.components.combat:SetTarget(...)` 会报 nil？** 答案是——在饥荒联机版里，**`components.combat` 只存在于服务端（Master）**；客户端拿到的是它的"网络镜像"`replica.combat`，而玩家这种特殊实体还多了一层"私有同步通道"`player_classified`。这三件东西分工不同、字段集不同、调用方法也不同。

> **三者职责一句话总结**：
>
> - **`inst.components.combat`（Master 端）**：战斗逻辑的真身，所有伤害计算、目标维护、攻击冷却都跑在这里——**只服务端有**。
> - **`inst.replica.combat`（Replica 端）**：战斗状态的轻量镜像，给客户端做攻击判定、目标判定、UI 显示——**所有客户端都有**（包括主机自己）。
> - **`inst.player_classified`（Classified 端，仅玩家）**：玩家专属的"私房网络对象"，承载只让本玩家看到的细节字段（小地图、配方表、战斗冷却等）——**只有玩家 prefab 有**，并且只在该玩家自己的客户端有完整可读权限。

读完本节，你将能回答：

- "我在 `Combat:SetTarget(target)` 里改了 `self.target`，客户端是怎么知道的？"
- "为什么 `combat_replica.lua` 顶部只有 3 个 `net_*` 变量，但客户端能查到 6 个战斗状态？"
- "我自己的怪物 prefab，需要写 `combat_replica` 吗？要主动 `AddTag('_combat')` 吗？"
- "为什么 mod 给 `combat` 加新字段，客户端读不到？该怎么解决？"

> **新手**先看 21.1.1-21.1.3 ——理解为什么需要这套架构、记住"客户端用 replica、服务端用 components"的口诀；**进阶读者**继续看 21.1.4-21.1.7，深入 `_combat` 标签的作用、`ReplicateComponent` 的工作流程、Classified 的特殊定位；**老手**跳到 21.1.8-21.1.10，掌握 Class setter 自动同步的内部机制、跨端调用的常见陷阱、mod 扩展 combat 时正确的 replica/classified 写法。

---

### 21.1.1 快速入门：一张图看懂 Master / Replica / Classified

先看一张全景图，再看代码就有方向感：

```
┌─────────────────────────────────────────────────────────────────┐
│                       服务端（Master Sim）                       │
│                                                                  │
│   inst.components.combat        ← 真身：战斗逻辑、伤害计算       │
│   ├─ self.target                                                 │
│   ├─ self.attackrange                                            │
│   ├─ self.canattack                                              │
│   ├─ self.min_attack_period                                      │
│   └─ self.panic_thresh                                           │
│                  │                                               │
│                  │ Class setter (props 钩子)                     │
│                  ▼                                               │
│   inst.replica.combat       ← 镜像（master 也有，方便统一调用）  │
│   ├─ self._target           : net_entity                         │
│   ├─ self._ispanic          : net_bool                           │
│   └─ self._attackrange      : net_float                          │
│         │                                                        │
│         │ 仅玩家：通过 self.classified 跨实体写                  │
│         ▼                                                        │
│   inst.player_classified         ← 玩家专属的"私房网络包"        │
│   ├─ inst.lastcombattarget   : net_entity                        │
│   ├─ inst.canattack          : net_bool                          │
│   ├─ inst.minattackperiod    : net_float                         │
│   ├─ inst.attackedpulseevent : net_event                         │
│   ├─ inst.isattackedbydanger : net_bool                          │
│   └─ inst.isattackredirected : net_bool                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │  网络同步（dirty -> tick）
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        客户端（Client）                          │
│                                                                  │
│   inst.components.combat = nil   ← 真身不存在！                  │
│                                                                  │
│   inst.replica.combat        ← 客户端唯一能直接用的"战斗对象"    │
│   ├─ self._target:value()         读到 master 同步过来的值       │
│   ├─ self._ispanic:value()                                       │
│   └─ self._attackrange:value()                                   │
│                                                                  │
│   inst.player_classified     ← 只有玩家 prefab 才有              │
│   └─ 同上 6 字段，都通过 :value() 读                             │
└─────────────────────────────────────────────────────────────────┘
```

**为什么要拆三份**？两个核心原因：

1. **带宽**——战斗组件里有大量服务端专用字段（`bonusdamagefn`、`keeptargetfn`、`externaldamagemultipliers` 一堆函数和大表），全同步过去既浪费带宽，客户端也根本用不到。所以**只把"客户端要做判定/显示用的几个值"挑出来**放到 replica。
2. **预测**——客户端在收到服务器响应之前要能立刻判断"我点这只猪能不能打到"，所以必须有一份本地可读的攻击距离、目标、能否攻击的状态。这就是 replica 的核心使命。

而 **Classified 的存在是因为**：有些战斗状态（比如玩家自己的攻击冷却、是否被强敌攻击的"音乐脉冲"）只该让**本玩家自己的客户端**知道，不能广播给其他玩家。于是这些字段不能挂在 `replica.combat` 上（那是对所有人可见），而是挂在 `player_classified` 上——它是一个独立的网络实体，由 `Network:SetClassifiedTarget` 绑定到玩家身上。

---

### 21.1.2 三者代码地图：先看在哪、后看是啥

| 文件路径 | 角色 | 关键类 |
|---|---|---|
| `scripts/components/combat.lua` | Master 端组件本体 | `Combat`（Class，1300+ 行） |
| `scripts/components/combat_replica.lua` | Replica 端镜像 | `Combat`（Class，491 行） |
| `scripts/prefabs/player_classified.lua` | 玩家私房网络实体 | 一个 prefab，含 1500+ 行 netvar |
| `scripts/entityreplica.lua` | replica 挂载机制 | `EntityScript:ReplicateComponent` |

> 注意：`components/combat.lua` 和 `components/combat_replica.lua` **都把 Class 取名 `Combat`**，但它们是两个完全独立的 Class，分别挂在 `inst.components.combat` 和 `inst.replica.combat`。这是饥荒里 Master/Replica 配对的通用命名习惯——`xxx.lua` 配 `xxx_replica.lua`。

`entityreplica.lua` 顶部维护着一张"可被 replicate 的组件白名单"：

```5:26:scripts/entityreplica.lua
local REPLICATABLE_COMPONENTS =
{
    builder = true,
    combat = true,
    container = true,
    constructionsite = true,
    equippable = true,
    fishingrod = true,
    follower = true,
    health = true,
    hunger = true,
    inventory = true,
    inventoryitem = true,
    moisture = true,
    named = true,
    oceanfishingrod = true,
    rider = true,
    sanity = true,
    sheltered = true,
    stackable = true,
    writeable = true,
}
```

`combat` 在白名单里，意味着只要服务端 `inst:AddComponent("combat")`，**框架会自动**：

1. 给实体打上 `_combat` 标签（让客户端反序列化时识别）；
2. 实例化 `components/combat_replica.lua` 并挂到 `inst.replica.combat`。

具体看 `entityscript.lua`：

```610:646:scripts/entityscript.lua
function EntityScript:AddComponent(name)
    local lower_name = string.lower(name)
    if self.lower_components_shadow[lower_name] ~= nil then
        print("component "..name.." already exists on entity "..tostring(self).."!"..debugstack_oneline(3))
        ...
    end

    local cmp = LoadComponent(name)
    if not cmp then
        error("component ".. name .. " does not exist!")
    end

    self:ReplicateComponent(name)

    local loadedcmp = cmp(self)
    self.components[name] = loadedcmp
    ...
end
```

注意第 631 行：`self:ReplicateComponent(name)` 是在挂载 `components.combat` **之前**先创建好 `replica.combat`。这也是为什么 `combat.lua` 的构造函数里能立刻安全地写 `self.inst.replica.combat:SetXxx(...)`——那时 replica 已经存在了。

---

### 21.1.3 Master 端 `Combat` 组件：核心字段速查

下面这张表只列**新手最常用 + replica 会同步**的字段，完整版（带 `bonusdamagefn`、`shouldrecoilfn` 等高级钩子）后续小节再细讲。

| 字段 | 类型 | 默认值 | 含义 | 会同步到 replica？ |
|---|---|---|---|---|
| `target` | Entity / nil | nil | 当前锁定的目标 | ✅ → `_target` |
| `attackrange` | number | 3 | 基础攻击距离（不含武器加成） | ✅ → `_attackrange` |
| `hitrange` | number | 3 | 基础命中距离（攻击落地判定，通常 ≥ attackrange） | ❌ |
| `min_attack_period` | number | 4 | 攻击间隔（秒） | ✅ → `classified.minattackperiod`（仅玩家） |
| `canattack` | bool | true | 是否允许发起攻击 | ✅ → `classified.canattack`（仅玩家） |
| `panic_thresh` | number / nil | nil | 血量百分比阈值，低于则触发恐慌（停止主动攻击） | ✅ → `_ispanic`（计算后） |
| `defaultdamage` | number | 0 | 徒手伤害值（武器会覆盖） | ❌ |
| `areahitrange` | number / nil | nil | 范围伤害半径 | ❌ |
| `areahitdamagepercent` | number / nil | nil | 范围伤害衰减系数 | ❌ |
| `lasttargetGUID` | number / nil | nil | 上一个目标 GUID | ✅ → `classified.lastcombattarget`（仅玩家） |
| `laststartattacktime` | number | 0 | 上次开始攻击的时间戳 | ❌（仅服务端冷却用） |
| `externaldamagemultipliers` | SourceModifierList | — | 外部伤害倍率叠加表 | ❌ |
| `externaldamagetakenmultipliers` | SourceModifierList | — | 外部受伤倍率叠加表 | ❌ |

最关键的入口在 `combat.lua` 顶部：

```24:110:scripts/components/combat.lua
local Combat = Class(function(self, inst)
    self.inst = inst

    self.nextbattlecrytime = nil
    self.battlecryenabled = true
    self.attackrange = 3
    self.hitrange = 3
    ...
    self.min_attack_period = 4
    ...
    self.canattack = true
    self.lasttargetGUID = nil
    self.target = nil
    self.panic_thresh = nil
    self.forcefacing = true
    ...
end,
nil,
{
    attackrange = onattackrange,
    min_attack_period = onminattackperiod,
    canattack = oncanattack,
    target = ontarget,
    panic_thresh = onpanicthresh,
})
```

**这里有个魔法**：`Class(...)` 的第 3 个参数 `{attackrange = onattackrange, ...}` 叫 **props（setter 钩子表）**。它的作用是：**任何时候你给 `self.attackrange = X` 赋值，都会自动触发 `onattackrange(self, X)`**——而后者正是把值写到 replica 的桥梁。21.1.8 会单独剖析这个机制。

5 个 setter 钩子就在文件顶部：

```4:22:scripts/components/combat.lua
local function onattackrange(self, attackrange)
    self.inst.replica.combat:SetAttackRange(attackrange)
end

local function onminattackperiod(self, minattackperiod)
    self.inst.replica.combat:SetMinAttackPeriod(minattackperiod)
end

local function oncanattack(self, canattack)
    self.inst.replica.combat:SetCanAttack(canattack)
end

local function ontarget(self, target)
    self.inst.replica.combat:SetTarget(target)
end

local function onpanicthresh(self, panicthresh)
    self.inst.replica.combat:SetIsPanic(panicthresh ~= nil and self.inst.components.health ~= nil and panicthresh > self.inst.components.health:GetPercent())
end
```

每一条都是一行——拿到新值，立刻喊 `inst.replica.combat:SetXxx(value)` 把值写进网络变量。**对开发者透明，对网络层透明。**

> ⚠️ `lasttargetGUID` 没在 props 里——因为它的同步需要"把实体引用转成 GUID"再传，没法用简单 setter 表达，所以 `combat.lua` 提供了一个显式的 `Combat:SetLastTarget` 方法：

```114:117:scripts/components/combat.lua
function Combat:SetLastTarget(target)
    self.lasttargetGUID = target ~= nil and target:IsValid() and target.GUID or nil
    self.inst.replica.combat:SetLastTarget(target ~= nil and target:IsValid() and target or nil)
end
```

---

### 21.1.4 Replica 端 `combat_replica`：客户端唯一的战斗入口

打开 `combat_replica.lua` 你会看到一个非常精炼的构造函数：

```1:15:scripts/components/combat_replica.lua
local Combat = Class(function(self, inst)
    self.inst = inst

    self._target = net_entity(inst.GUID, "combat._target")
    self._ispanic = net_bool(inst.GUID, "combat._ispanic")
    self._attackrange = net_float(inst.GUID, "combat._attackrange")
    self._laststartattacktime = nil

    if TheWorld.ismastersim then
        self.classified = inst.player_classified
    elseif self.classified == nil and inst.player_classified ~= nil then
        self:AttachClassified(inst.player_classified)
    end
end)
```

**只有 3 个 `net_*` 变量**——这就是所有非玩家实体（猪、蜘蛛、巨鹿……）能让客户端看到的全部战斗状态：

| Replica 字段 | 类型 | 同步内容 | 客户端典型用途 |
|---|---|---|---|
| `_target` | `net_entity` | 当前目标 | 显示战斗 UI、判定友军伤害、`CanBeAttacked` 检查 |
| `_ispanic` | `net_bool` | 是否处于"恐慌脱战"状态 | `CanTarget` 时跳过目标 |
| `_attackrange` | `net_float` | 基础攻击距离 | 客户端攻击预测、`CanAttack` 判定 |

它们都是用 `inst.GUID` 作为命名空间，名称用 `"combat._xxx"` 作 key——这是 netvar 在 C++ 网络层唯一定位的依据，**所以不能重复**（连同 mod 写的 netvar）。

**`_laststartattacktime` 不是 `net_*`**！它是一个普通字段，纯粹的客户端本地变量，用来做攻击冷却预测：

```136:159:scripts/components/combat_replica.lua
function Combat:StartAttack()
    if self.inst.components.combat ~= nil then
        self.inst.components.combat:StartAttack()
    elseif self.classified ~= nil then
        self._laststartattacktime = GetTime()
    end
end

function Combat:CancelAttack()
    if self.inst.components.combat ~= nil then
        self.inst.components.combat:CancelAttack()
    elseif self.classified ~= nil then
        self._laststartattacktime = nil
    end
end

function Combat:InCooldown()
    if self.inst.components.combat ~= nil then
        return self.inst.components.combat:InCooldown()
    elseif self.classified ~= nil then
        return self._laststartattacktime ~= nil and self._laststartattacktime + self.classified.minattackperiod:value() > GetTime()
    end
    return false
end
```

注意每个方法都有这个二段式：

```lua
if self.inst.components.combat ~= nil then
    -- 服务端：直接调真身
    return self.inst.components.combat:XxxXxx(...)
elseif self.classified ~= nil then
    -- 客户端 + 玩家：走 classified 读 netvar
    ...
end
```

这就是 replica 最经典的"中间层"写法——**无论调用方在客户端还是服务端，都用同一个方法名 `inst.replica.combat:XxxXxx(...)`**，replica 内部自动分流。这让上层（stategraph、动作系统、UI）可以写一份代码同时跑在两端。

`Combat:GetWeapon` 是个很好的例子：

```97:112:scripts/components/combat_replica.lua
function Combat:GetWeapon()
    if self.inst.components.combat ~= nil then
        return self.inst.components.combat:GetWeapon()
    elseif self.inst.replica.inventory ~= nil then
        local item = self.inst.replica.inventory:GetEquippedItem(EQUIPSLOTS.HANDS)
        if item ~= nil and item:HasTag("weapon") then
            if (item:HasTag("projectile") and not item:HasTag("complexprojectile")) or
                item:HasTag("rangedweapon")
            then
                return item
            end
            local rider = self.inst.replica.rider
            return not (rider ~= nil and rider:IsRiding()) and item or nil
        end
    end
end
```

服务端直接拿真组件的 `:GetWeapon()`；客户端没有 `components.inventory`，所以转去问 `replica.inventory`，并用 `weapon` 标签判定。

---

### 21.1.5 `player_classified`：玩家专属的"私房网络包"

`player_classified` 是个独立 prefab，主玩家创建时一并实例化，并把它绑给玩家：

```1006:1009:scripts/prefabs/player_common.lua
if TheWorld.ismastersim then
    TheNet:SetIsClientInWorld(inst.userid, true)
    inst.player_classified.Network:SetClassifiedTarget(inst)
end
```

`SetClassifiedTarget(inst)` 告诉网络层："这个 classified 实体的'网络可见性'跟它绑定的玩家走"——只有玩家自己的客户端能读到完整字段。

战斗相关的 6 个 netvar 集中定义在文件末尾：

```1560:1568:scripts/prefabs/player_classified.lua
--Combat variables
inst.lastcombattarget = net_entity(inst.GUID, "combat.lasttarget")
inst.canattack = net_bool(inst.GUID, "combat.canattack")
inst.minattackperiod = net_float(inst.GUID, "combat.minattackperiod")
inst.attackedpulseevent = net_event(inst.GUID, "combat.attackedpulse")
inst.isattackedbydanger = net_bool(inst.GUID, "combat.isattackedbydanger")
inst.isattackredirected = net_bool(inst.GUID, "combat.isattackredirected")
inst.canattack:set(true)
inst.minattackperiod:set(4)
```

| Classified 字段 | 类型 | 用途 |
|---|---|---|
| `lastcombattarget` | `net_entity` | 玩家上一个交战目标——给 `IsRecentTarget` 用 |
| `canattack` | `net_bool` | 玩家是否能发起攻击（`BlankOutAttacks` 会把它设 false） |
| `minattackperiod` | `net_float` | 玩家攻击冷却时长（武器/技能可改） |
| `attackedpulseevent` | `net_event` | "受击脉冲"——只触发一次的客户端事件 |
| `isattackedbydanger` | `net_bool` | 攻击者是否构成"危险"（用于动态音乐） |
| `isattackredirected` | `net_bool` | 这次伤害是否被重定向（吉祥物挡刀等） |

> **为什么放 classified 而不是 replica.combat？** 因为 `replica.combat` 是**所有客户端都能读**的（用来判断别人能不能打你），如果把 `canattack` 放进去，等于把"玩家 A 的攻击冷却"广播给地图上所有玩家，浪费带宽且毫无意义。放进 classified 后，**只有玩家 A 自己的客户端**能读到。

`replica.combat` 怎么"借用"这些字段？看构造函数最后：

```9:14:scripts/components/combat_replica.lua
if TheWorld.ismastersim then
    self.classified = inst.player_classified
elseif self.classified == nil and inst.player_classified ~= nil then
    self:AttachClassified(inst.player_classified)
end
```

只要这个实体身上有 `player_classified`（也就是它是玩家），replica 就把它存到 `self.classified`，于是 `SetCanAttack` 这种方法实际是写到 classified：

```130:134:scripts/components/combat_replica.lua
function Combat:SetCanAttack(canattack)
    if self.classified ~= nil then
        self.classified.canattack:set(canattack)
    end
end
```

**普通生物（猪、蜘蛛）没有 `player_classified`**，所以 `self.classified == nil`，`SetCanAttack` 直接静默返回——猪根本没有"是否能攻击"这个需要同步的字段，因为 AI 完全跑在服务端，客户端不需要知道。

### 21.1.5.1 `attackedpulseevent` 的事件链：写给进阶读者

`net_event` 是一种特殊的 netvar——它**不存值**，只触发一次"dirty"，让客户端跑一遍对应的监听器。受击脉冲的完整链路：

```
服务端：玩家被攻击
    ↓
inst:PushEvent("attacked", {attacker=..., redirected=...})
    ↓
player_classified.lua 在 RegisterNetListeners_mastersim 注册的 OnAttacked 触发
    ↓
inst.attackedpulseevent:push()           ← net_event 标 dirty
inst.isattackedbydanger:set(...)         ← net_bool 同步
inst.isattackredirected:set(...)         ← net_bool 同步
    ↓
（下一个网络 tick）
    ↓
玩家自己的客户端：触发 "combat.attackedpulse" dirty 事件
    ↓
RegisterNetListeners_local 里挂的 OnAttackedPulseEvent
    ↓
inst._parent:PushEvent("attacked", { isattackedbydanger=..., redirected=... })
```

源码：

```113:121:scripts/prefabs/player_classified.lua
local NON_DANGER_TAGS = {"noepicmusic", "shadow", "shadowchesspiece", "smolder", "thorny", "nodangermusic"}
local function OnAttacked(parent, data)
    parent.player_classified.attackedpulseevent:push()
    parent.player_classified.isattackedbydanger:set(
        data ~= nil and
        data.attacker ~= nil and
        not data.attacker:HasAnyTag(NON_DANGER_TAGS)
    )
    parent.player_classified.isattackredirected:set(data ~= nil and data.redirected ~= nil)
end
```

```326:330:scripts/prefabs/player_classified.lua
local function OnAttackedPulseEvent(inst)
    if inst._parent ~= nil then
        inst._parent:PushEvent("attacked", { isattackedbydanger = inst.isattackedbydanger:value(), redirected = inst.isattackredirected:value() })
    end
end
```

这就是为什么 **同一个 `"attacked"` 事件**，在服务端和客户端都能监听到——但数据结构不同：

- 服务端版本：`{attacker, damage, weapon, stimuli, redirected, original_damage, damageresolved, ...}`
- 客户端版本：`{isattackedbydanger, redirected}`

> **mod 开发提示**：客户端版的 `attacked` 事件不包含 `attacker`、不包含 `damage`。如果你要做"客户端 UI 显示伤害数字"，得自己用一个新的 `net_event` 把伤害值（floor 取整后）通过 classified 同步过去。

---

### 21.1.6 `_combat` 标签：客户端怎么知道这个实体要加 `combat_replica`

客户端拿到一个实体（比如玩家加入时 Spawn 出的猪）时，它**没有跑过服务端的 `master_postinit`**，所以并不知道这只猪有 `combat` 组件。`_combat` 标签就是这两端之间的"暗号"。

工作流程：

**服务端**

```34:60:scripts/entityreplica.lua
function EntityScript:ReplicateComponent(name)
    if not REPLICATABLE_COMPONENTS[name] then
        return
    end

    if TheWorld.ismastersim then
        self:AddTag("_"..name)
        if self:HasTag("__"..name) then
            self:RemoveTag("__"..name)
            return
        end
    end

    if rawget(self.replica, "_")[name] ~= nil then
        print("replica "..name.." already exists! "..debugstack_oneline(3))
    end

    local filename = name.."_replica"
    local cmp = Replicas[filename]
    if cmp == nil then
        cmp = require("components/"..filename)
        Replicas[filename] = cmp
    end
    assert(cmp ~= nil, "replica "..name.." does not exist!")

    rawset(self.replica._, name, cmp(self))
end
```

第 40 行 `self:AddTag("_"..name)` 就是给实体打上 `_combat` 标签。这个标签会序列化进实体的 pristine state，跟随网络包到达客户端。

**客户端**

实体到达客户端、反序列化完 tag 后，`EntityScript:ReplicateEntity` 自动跑：

```75:85:scripts/entityreplica.lua
function EntityScript:ReplicateEntity()
    for k, v in pairs(REPLICATABLE_COMPONENTS) do
        if v and (self:HasTag("_"..k) or self:HasTag("__"..k)) then
            self:ReplicateComponent(k)
        end
    end

    if self.OnEntityReplicated ~= nil then
        self:OnEntityReplicated()
    end
end
```

——遍历白名单，凡是看到 `_combat`/`__combat` 标签的，就 `ReplicateComponent("combat")` 把 replica 挂上来。

> **`__combat`（双下划线）是什么**？看 `UnreplicateComponent`：服务端如果在 `master_postinit` 之后又 `RemoveComponent("combat")`，会去掉 `_combat`、加上 `__combat`。这种"曾经有过、现在没了"的实体在客户端仍然需要保留 replica（因为可能还要展示之前的状态），所以也算入白名单。`PrereplicateComponent` 就是手动玩这一招——本节末尾再讲。

**玩家比普通生物多一手优化**：玩家的 `_combat` 标签在 `common_postinit` 阶段就提前打上：

```2537:2544:scripts/prefabs/player_common.lua
--Sneak these into pristine state for optimization
inst:AddTag("_health")
inst:AddTag("_hunger")
inst:AddTag("_sanity")
inst:AddTag("_builder")
inst:AddTag("_combat")
inst:AddTag("_moisture")
inst:AddTag("_sheltered")
inst:AddTag("_rider")
```

注释写得很直白——`Sneak these into pristine state for optimization`：抢在 `entity:SetPristine()` 之前打标签，让标签随首次同步包一起发出去，省一次 dirty-tick。

---

### 21.1.7 玩家 vs 普通生物：架构差异对比表

|  | 玩家（wilson、wendy 等） | 普通生物（猪、蜘蛛、巨鹿） |
|---|---|---|
| 服务端是否有 `components.combat` | ✅ | ✅ |
| 是否有 `_combat` 标签 | ✅（在 common_postinit 预先打上） | ✅（AddComponent 时自动打上） |
| 客户端是否有 `replica.combat` | ✅ | ✅ |
| 是否有 `player_classified` | ✅ | ❌ |
| `replica.combat.classified` 指向 | `inst.player_classified` | nil |
| `SetCanAttack` / `SetMinAttackPeriod` 是否能同步 | ✅ | ❌（静默丢弃） |
| 客户端 `CanAttack` 用什么数据 | 走 classified 的 `canattack` / `minattackperiod` | 不走这条路（只能问 master） |
| 受击脉冲 `attackedpulseevent` | ✅ | ❌ |

举个非常具体的差异——你在 client 端调 `inst.replica.combat:CanAttack(target)`：

- 如果 `inst` 是**玩家**且**客户端就是该玩家**：`replica.combat` 能通过 classified 读到 `canattack:value()`、`minattackperiod:value()`，进而预测能否攻击。
- 如果 `inst` 是**别人**（其他玩家）或者**怪物**：`self.classified == nil`，`CanAttack` 直接返回 false。

这就是为什么饥荒里**只有"本地玩家"能做攻击预测**——你看不到其他玩家或怪物什么时候出手，必须等服务器同步动作。

> 历史小知识：早期没有 `player_classified` 时，玩家的攻击冷却也跑过"客户端等服务器"路线，导致延迟下挥空感很强。后来把这部分搬进 classified 才让客户端能流畅地预攻击。

---

### 21.1.8 老手必读：Class setter 钩子的内部机制

新手看 `combat.lua` 顶部那个奇怪的"第 3 个参数"会一头雾水。看完这一小节你能彻底理解。

打开 `class.lua`，关键就 4 个函数：

```28:45:scripts/class.lua
local function __index(t, k)
    local p = rawget(t, "_")[k]
    if p ~= nil then
        return p[1]
    end
    return getmetatable(t)[k]
end

local function __newindex(t, k, v)
    local p = rawget(t, "_")[k]
    if p == nil then
        rawset(t, k, v)
    else
        local old = p[1]
        p[1] = v
        p[2](t, v, old)
    end
end
```

理解这一段是关键：

- 当一个 Class 带了 props（第 3 个参数），它的元表会装上 `__index` 和 `__newindex`。
- 对象内部多了一个 `_` 表，把每个 prop key 映射成 `{ currentValue, setterFn }` 的二元组。
- 读：`self.attackrange` → 走 `__index` → 从 `_` 表取出当前值。
- 写：`self.attackrange = 5` → 走 `__newindex` → **先更新存储值，再调用 setter 函数**`p[2](self, 5, old)`。

回到 `combat.lua`：

```104:110:scripts/components/combat.lua
nil,
{
    attackrange = onattackrange,
    min_attack_period = onminattackperiod,
    canattack = oncanattack,
    target = ontarget,
    panic_thresh = onpanicthresh,
})
```

构造时：`obj._.attackrange = { nil, onattackrange }`、`obj._.target = { nil, ontarget }` ……

然后构造函数体内：

```28:80:scripts/components/combat.lua
    self.attackrange = 3        -- 触发 onattackrange(self, 3, nil) → 写入 replica._attackrange
    ...
    self.min_attack_period = 4  -- 触发 onminattackperiod
    ...
    self.canattack = true       -- 触发 oncanattack
    ...
    self.target = nil           -- 触发 ontarget（这里写 nil 也会被同步）
    self.panic_thresh = nil     -- 触发 onpanicthresh
```

——**初始化的同时就完成了所有 5 个字段的网络同步**。后续 mod 调用 `inst.components.combat.attackrange = 7` 时，**根本不用关心 replica**，框架自动同步。

> **mod 实践陷阱**：如果你想给 combat 加一个新字段 `extra_pierce`，单纯写：
>
> ```lua
> inst.components.combat.extra_pierce = 5
> ```
>
> ——客户端**收不到**！因为这个字段不在 props 表里、没 setter。要让客户端读到，必须：
>
> 1. 自己定义一个 netvar（最好在 mod 自己的 replica 或 classified 上）。
> 2. 写一个 setter 函数把值写进 netvar。
> 3. 通过 `AddComponentPostInit("combat", fn)` 在组件构造后用 `addsetter(self, "extra_pierce", mySetterFn)` 把字段升级成"带钩子的字段"。
>
> 21.1.10 会给出完整示例。

---

### 21.1.9 mod 开发实战姿势（新手 / 进阶 / 老手三档）

#### 新手档：给你的怪物 prefab 加 combat 组件

最经典的"新手模板"——参考 mod 中的雪梅大头领（见 `mods/联机版mod/万物书/scripts/prefabs/11_tbat_animals/01_snow_plum_chieftain.lua`）：

```lua
local function master_postinit(inst)
    inst:AddComponent("health")
    inst.components.health:SetMaxHealth(POG_HEALTH)

    inst:AddComponent("combat")
    inst.components.combat:SetDefaultDamage(POG_DAMAGE)
    inst.components.combat:SetRange(POG_ATTACK_RANGE, POG_ATTACK_RANGE)
    inst.components.combat:SetAttackPeriod(POG_ATTACK_PERIOD)
    inst.components.combat:SetKeepTargetFunction(KeepTargetFn)
    inst.components.combat:SetRetargetFunction(3, RetargetFn)
    inst.components.combat:SetHurtSound("dontstarve_DLC003/creatures/pog/hit")
    inst:ListenForEvent("attacked", OnAttacked)
end
```

**为什么不用手动管 replica**？因为：

1. `inst:AddComponent("combat")` → 自动 `ReplicateComponent("combat")` → 自动打 `_combat` tag、自动挂 `replica.combat`；
2. `SetRange` 等方法内部最终会改 `self.attackrange`，触发 setter → 写 replica；
3. 客户端拿到 tag 后，`ReplicateEntity` 自动给客户端版的实体挂上 `replica.combat`。

**你什么都不用做**。

#### 进阶档：让客户端预读"自定义子弹剩余"

假设你给一个远程武器加了"弹药数"，希望客户端 UI 实时显示。

错误写法（客户端读不到）：

```lua
inst.components.weapon.bullets = 30
```

正确写法（用 inventoryitem replica 的 classified 或自建 netvar）：

```lua
-- 服务端：构造时建 netvar、在 master_postinit 维护
local function fn()
    local inst = CreateEntity()
    ...
    inst._bullets = net_byte(inst.GUID, "myweapon.bullets", "bulletsdirty")
    inst._bullets:set(30)
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end
    ...
end

-- 客户端：监听 dirty 事件
inst:ListenForEvent("bulletsdirty", function(i)
    if i.HUD ~= nil then
        i.HUD.controls.bullet_label:SetString(tostring(i._bullets:value()))
    end
end)
```

注意第二个参数 `"bulletsdirty"` 是 **dirty 事件名**。客户端 `:value()` 拿到的永远是最新已同步值；想知道"什么时候变了"，监听 dirty 事件即可。

#### 老手档：扩展玩家的战斗 classified

如果你的 mod 给玩家加了一个"暴击率"字段，希望本玩家的客户端 UI 显示。最佳实践是**直接往 `player_classified` 上扩展 netvar**，因为它本来就是"私房包"。

```lua
-- modmain.lua
AddPrefabPostInit("player_classified", function(inst)
    inst.mod_critrate = net_float(inst.GUID, "mymod.critrate", "critratedirty")
    inst.mod_critrate:set(0)
end)

-- 服务端 mod 在玩家身上挂的组件：
function MyCritComponent:SetCritRate(rate)
    self.critrate = rate
    if self.inst.player_classified ~= nil then
        self.inst.player_classified.mod_critrate:set(rate)
    end
end

-- 客户端：HUD 监听
function PlayerHud:AttachCritDisplay(owner)
    owner:ListenForEvent("critratedirty", function(p)
        self.crit_label:SetString(string.format("%.1f%%", p.player_classified.mod_critrate:value() * 100))
    end, owner.player_classified)
end
```

这种写法比"自己 hack 一个 net_*"干净得多——配合 `player_classified` 的生命周期，玩家断线/换皮肤/复活，你的 netvar 也会跟着正确销毁/重建。

---

### 21.1.10 五个常见坑

1. **`inst.components.combat` 在客户端是 nil**
   一定要先判 `if TheWorld.ismastersim then ... else ... end`，或者用 `inst.replica.combat`。把客户端要用的逻辑写到 replica 里。

2. **`AddTag("_combat")` 之后再 `AddComponent("combat")` 会报 "replica already exists"**
   不要自己手动打 `_combat` 标签。除非你在 `common_postinit` 里有特殊需求（比如玩家那种 pristine 优化），否则交给 `AddComponent` 自动处理。

3. **`PrereplicateComponent` 是干嘛的**
   看 `deciduoustrees.lua`、`antlion.lua`、`wagboss_robot.lua` 的用法——它们在 prefab 处于"非战斗形态"时仍然让客户端有 replica（因为可能会变成战斗形态）。
   ```lua
   inst:PrereplicateComponent("combat")
   ```
   内部等价于"先 `ReplicateComponent` 再 `UnreplicateComponent`"，结果是标签从 `_combat` 切到 `__combat`，客户端仍然会挂 replica，但服务端没有真组件。后续真要打架时再 `AddComponent("combat")`，标签会从 `__combat` 切回 `_combat`，replica 不重建。

4. **客户端读自定义字段读不到**
   见 21.1.8 末尾。`inst.components.combat.my_field = X` 这种写法**永远不会同步**——必须走 netvar。

5. **改 `attackrange` 不生效是因为武器叠加**
   `Combat:GetAttackRange()` 返回的是 `self.attackrange + weapon.attackrange`。客户端这边 `combat_replica:GetAttackRangeWithWeapon()` 也是同样的叠加。**所以你给玩家加 +2 攻击距离 buff，要改 `inst.components.combat.attackrange`，不能改武器的**——否则裸手时不生效。

   ```984:990:scripts/components/combat.lua
   function Combat:GetAttackRange()
       local weapon = self:GetWeapon()
       return weapon ~= nil
           and weapon.components.weapon.attackrange ~= nil
           and math.max(0, self.attackrange + weapon.components.weapon.attackrange)
           or self.attackrange
   end
   ```

---

### 本节涉及的标签

| 标签 | 含义 | 作用 |
|---|---|---|
| `_combat` | "本实体有 combat 组件" | 由 `ReplicateComponent("combat")` 自动打上，客户端反序列化时识别并挂载 `replica.combat` |
| `__combat` | "本实体曾有 combat 组件，现已移除" | 由 `UnreplicateComponent("combat")` 在服务端打上，客户端仍要挂 replica 用于记录历史状态 |
| `weapon` | "这是武器" | `combat_replica:GetWeapon` 判定手持物是否能作为武器 |
| `projectile` / `complexprojectile` / `rangedweapon` | 远程武器分类 | 决定是否允许骑乘状态下挥击 |
| `propweapon` | "道具武器"（如玩具锤子） | PVP 关闭时仍允许玩家间互打 |
| `playerghost` | 鬼魂状态 | `CanBeAttacked` 中过滤鬼魂 |
| `noattack` / `invisible` / `flight` | 无敌 / 隐身 / 飞行 | `CanBeAttacked` 中过滤 |
| `notarget` / `INLIMBO` / `debugnoattack` | 不可被选为目标 | `CanTarget` 中过滤 |
| `spawnprotection` | 复活保护 | `IsValidTarget` 中过滤 |
| `crazy` | 攻击者疯狂状态 | 影响 `CanBeAttacked` 对暗影生物的判定 |
| `shadowcreature` / `nightmarecreature` | 暗影 / 噩梦生物 | 非疯狂玩家无法攻击它们 |
| `alwayshostile` | 始终敌对 | `CanBeAlly` 中过滤 |
| `domesticated` | 已驯化的生物 | 视作玩家阵营 |
| `companion` | 玩家伙伴 | 视作玩家阵营 |
| `hostile` | 敌对状态 | `ShouldAggro` 中影响最小血量过滤 |
| `stealth` | 潜行 | `ShouldAggro` 中过滤 |
| `monster` / `pig` | 怪物 / 猪人 | 用于"危险目标"音乐和 AI 选择 |
| `_inventory` / `_rider` / `_health` 等 | 对应组件存在的 replica 标记 | 同 `_combat` 机制，框架自动管理 |

### 本节涉及的事件

| 事件名 | 推送对象 | 推送时机 | 数据键值 |
|---|---|---|---|
| `attacked`（服务端版） | 被攻击实体 | `Combat:GetAttacked` 中未 blocked 时（`combat.lua` 第 686 行） | `attacker`(Entity) / `damage`(number) / `damageresolved`(number) / `original_damage`(number) / `weapon`(Entity) / `stimuli`(string) / `spdamage`(table\|nil) / `redirected`(Entity\|nil) / `noimpactsound`(bool\|nil)（21.8 详解） |
| `blocked` | 被攻击实体 | `Combat:GetAttacked` 中 blocked 时（recoil / 0 伤害） | `attacker`(Entity) / `damage`(number) / `spdamage`(table\|nil) / `original_damage`(number) |
| `onhitother` | 攻击者本身 | 成功打中目标时（`combat.lua` 第 693 行） | `target`(Entity) / `damage`(number) / `damageresolved`(number) / `stimuli`(string) / `spdamage`(table\|nil) / `weapon`(Entity) / `redirected`(Entity\|nil) |
| `killed` | 攻击者本身 | 攻击导致目标死亡（`combat.lua` 第 652 行） | `victim`(Entity) / `attacker`(Entity) |
| `attacked`（客户端版） | 被攻击的玩家实体（仅本玩家自己的客户端能接收，因 classified 私有） | classified `OnAttackedPulseEvent` 触发 | `isattackedbydanger`(bool) / `redirected`(bool) |
| `newcombattarget` | 攻击者本身 | `Combat:EngageTarget` 切换目标后 | `target`(Entity) / `oldtarget`(Entity) |
| `droppedtarget` | 攻击者本身 | `Combat:DropTarget` 主动放弃目标时 | `target`(Entity) |
| `losttarget` | 攻击者本身 | `Combat:OnUpdate` 中 `keeptargetfn` 判定失败导致脱战 | 无 |
| `giveuptarget` | 攻击者本身 | `Combat:GiveUp` 因脱战放弃目标 | `target`(Entity) |
| `doattack` | 攻击者本身 | `Combat:TryAttack` / `ForceAttack` 通过 stategraph 发起攻击 | `target`(Entity) |
| `combat.attackedpulse` | 玩家的 `player_classified` 实体 | `attackedpulseevent:push()` | 无（事件型 netvar） |
| `enterlimbo` | 目标实体 | 目标进入 limbo（被放入容器、被吞食等） | 无（用于自动 DropTarget） |
| `onremove` | 目标实体 | 目标被销毁 | 无（用于自动 DropTarget） |
| `transfercombattarget` | 目标实体 | 触发"仇恨转移"（如某些 boss 召唤分身） | `newtarget`(Entity) |
| `leaderchanged` | 目标实体 | 跟随者的领主变更 | （随实现） |
| `knockback` | 被击退的实体本身 | 由 `knockback` 组件 / 弹反逻辑推送 | combat 组件构造时挂监听 `CommonHandlers.ResetHitRecoveryDelay`，用于重置硬直时长 |

---



## 21.2 初始化与属性

### 本节导读

21.1 把"战斗组件分三份"的全景介绍过了，现在我们进入第一节真正的实操：**怎么从无到有给一个实体配上 combat，并把所有关键属性调对**。这一节看似平淡，但 80% 的 mod 战斗 bug 都来自这一步——`SetRange` 和 `SetAttackPeriod` 顺序写反、把武器伤害写在 `defaultdamage` 上、修改 `attackrange` 之后客户端读不到、忘了 `SetKeepTargetFunction` 导致脱战立刻丢目标……都属于初始化阶段的"小手抖、大灾难"。

读完本节，你将能回答：

- "`SetDefaultDamage(60)` 和 `defaultdamage = 60` 一样吗？哪个会同步到 replica？"
- "`SetRange(3, 5)` 第二个参数是什么意思？什么时候要用？"
- "我给一个怪物加了 `damagemultiplier = 2`，为什么对玩家伤害不是 2 倍？"
- "`bonusdamagefn` 和 `customdamagemultfn` 都是改伤害，到底有什么区别？什么时候用哪一个？"
- "`externaldamagemultipliers` 这种 SourceModifierList 怎么用？为什么不直接乘 `damagemultiplier`？"
- "为什么 `combat.lua` 顶部有一堆 `--self.xxx = ...` 注释掉的字段？我能不能用？"

> **新手**先看 21.2.1-21.2.4——掌握"5 行模板配齐怪物战斗"、构造函数读图、基础 setter 用法；**进阶读者**继续看 21.2.5-21.2.8，吃透距离系字段、伤害修正栈、四种钩子函数；**老手**直接跳到 21.2.9-21.2.13，掌握硬直/仇恨过滤/状态字段、mod 扩展正确姿势、五个常见坑。

---

### 21.2.1 快速入门：5 行代码让一只猪会打架

最简洁的"会战斗的怪物"模板：

```lua
inst:AddComponent("combat")
inst.components.combat:SetDefaultDamage(20)           -- 徒手攻击伤害
inst.components.combat:SetAttackPeriod(2)             -- 攻击冷却 2 秒
inst.components.combat:SetRange(3)                    -- 攻击距离 3 格（同时设置命中距离）
inst.components.combat:SetRetargetFunction(2, RetargetFn)  -- 每 2 秒重新搜索目标
inst.components.combat:SetKeepTargetFunction(KeepTargetFn) -- 是否继续锁定当前目标
```

这 5 行就能让一只生物：

1. 有完整的战斗逻辑真身（`components.combat`）和客户端镜像（`replica.combat`）；
2. 拥有徒手伤害、攻击冷却、攻击距离；
3. 每 2 秒主动搜目标；
4. 锁定目标后用 `KeepTargetFn` 持续判定是否要脱战；
5. 客户端能自动通过 `replica.combat` 读到攻击距离做攻击预测、UI 显示。

完整例子见万物书的雪梅大头领（`mods/联机版mod/万物书/scripts/prefabs/11_tbat_animals/01_snow_plum_chieftain.lua` 第 253-262 行）：

```253:262:mods/联机版mod/万物书/scripts/prefabs/11_tbat_animals/01_snow_plum_chieftain.lua
		--- 战斗
			inst:AddComponent("combat")
			inst.components.combat:SetDefaultDamage(POG_DAMAGE)
			inst.components.combat:SetRange(POG_ATTACK_RANGE,POG_ATTACK_RANGE)
			inst.components.combat:SetAttackPeriod(POG_ATTACK_PERIOD)
			inst.components.combat:SetKeepTargetFunction(KeepTargetFn)
			inst.components.combat:SetRetargetFunction(3, RetargetFn)
			inst.components.combat:SetHurtSound("dontstarve_DLC003/creatures/pog/hit")
			inst:ListenForEvent("attacked", OnAttacked)
			inst.components.combat.battlecryinterval = 20
```

——这就是雪梅大头领"被打→设目标→分享仇恨给同伴"的完整初始化。注意它**没有手动 `AddTag("_combat")`、没有写 `combat_replica`、没有在 common_postinit 里碰任何战斗代码**——一切都是 `AddComponent("combat")` 触发了 21.1.6 讲过的"自动 ReplicateComponent"。

> 但是注意一点：`SetRange` 第二个参数（hitrange）这里和第一个相等。绝大多数怪物都是这么设的——`attackrange == hitrange`。等到 21.2.5 你会看到为什么近战刺客需要 `hitrange > attackrange`。

---

### 21.2.2 构造函数全景：每一行都在做什么

打开 `scripts/components/combat.lua`，构造函数从第 24 行开始：

```24:110:scripts/components/combat.lua
local Combat = Class(function(self, inst)
    self.inst = inst

    self.nextbattlecrytime = nil
    self.battlecryenabled = true
    self.attackrange = 3
    self.hitrange = 3
    self.areahitrange = nil
    self.temprange = nil
	--self.areahitcheck = nil
    self.areahitdamagepercent = nil
    --self.areahitdisabled = nil
    self.defaultdamage = 0
    --
    --use nil for defaults
    --self.playerdamagepercent = 1 --modifier for NPC dmg on players, only works with NO WEAPON
    --self.pvp_damagemod = 1
    --self.damagemultiplier = 1
    --self.damagebonus = 0
    --self.ignorehitrange = false
    --self.noimpactsound = false
    --

    -- NOTES(JBK): Aggro system for combat to help make things untargetable temporarily.
    self.shouldaggrofn = nil
    self.shouldavoidaggro = nil
    self.forbiddenaggrotags = nil
	self.lastwasattackedbytargettime = 0
	--self.lastattacker = nil
	--self.lastattacktype = nil
	--self.laststimuli = nil

    --self.tough = false
	--self.workmultiplierfn = nil
	--self.shouldrecoilfn = nil

	self.externaldamagemultipliers = SourceModifierList(self.inst) -- damage dealt to others multiplier
	self.externaldamagetakenmultipliers = SourceModifierList(self.inst) -- my damage taken multiplier (post armour reduction)
    -- self.conditionexternaldamagetakenmultipliers = {} -- extra damage taken on certain conditions (post armour reduction)

    self.min_attack_period = 4
    self.onhitfn = nil
    self.onhitotherfn = nil
    self.laststartattacktime = 0
    self.lastwasattackedtime = 0
    self.keeptargetfn = nil
    self.keeptargettimeout = 0
    self.hiteffectsymbol = "marker"
    self.canattack = true
    self.lasttargetGUID = nil
    self.target = nil
    self.panic_thresh = nil
    self.forcefacing = true
    self.bonusdamagefn = nil
    --self.playerstunlock = PLAYERSTUNLOCK.ALWAYS --nil for default

	self.losetargetcallback = function() self:DropTarget() end
	self.transfertargetcallback = function(target, newtarget) ... end
	self.allycheckcallback = function(target) ... end

	inst:ListenForEvent("knockback", CommonHandlers.ResetHitRecoveryDelay)
end,
nil,
{
    attackrange = onattackrange,
    min_attack_period = onminattackperiod,
    canattack = oncanattack,
    target = ontarget,
    panic_thresh = onpanicthresh,
})
```

把它分四块看：

**第一块：直接赋的初始值**

| 字段 | 默认值 | 是否有 setter 钩子 | 说明 |
|---|---|---|---|
| `attackrange` | 3 | ✅ `onattackrange` | 基础攻击距离，构造时赋 3 立刻同步到 replica |
| `hitrange` | 3 | ❌ | 基础命中判定距离，**不会同步** |
| `min_attack_period` | 4 | ✅ `onminattackperiod` | 攻击冷却，仅玩家会同步到 classified |
| `canattack` | true | ✅ `oncanattack` | 是否允许发起攻击，仅玩家同步到 classified |
| `target` | nil | ✅ `ontarget` | 当前目标，赋 nil 也会同步 |
| `panic_thresh` | nil | ✅ `onpanicthresh` | 恐慌阈值，结合 health 计算后写入 `_ispanic` |
| `defaultdamage` | 0 | ❌ | 徒手伤害 |
| `battlecryenabled` | true | ❌ | 是否允许喊战吼 |
| `nextbattlecrytime` | nil | ❌ | 下次能喊战吼的时间戳 |
| `laststartattacktime` | 0 | ❌ | 上次开始攻击时间戳（冷却用） |
| `lastwasattackedtime` | 0 | ❌ | 上次被攻击时间戳 |
| `keeptargettimeout` | 0 | ❌ | KeepTarget 判定下一次执行的剩余时间 |
| `hiteffectsymbol` | `"marker"` | ❌ | 命中特效要附着的 anim symbol（猪是 `pig_torso`、玩家是 `torso`） |
| `forcefacing` | true | ❌ | 攻击瞬间是否强制朝向目标 |
| `lastwasattackedbytargettime` | 0 | ❌ | 上次被"当前目标"打中的时间（用于"它打我我就还手"逻辑） |

**第二块：被注释掉的"隐含字段"**

这堆带 `--` 注释的字段（`playerdamagepercent`、`pvp_damagemod`、`damagemultiplier`、`damagebonus`、`ignorehitrange`、`noimpactsound`、`areahitcheck`、`areahitdisabled`、`tough`、`workmultiplierfn`、`shouldrecoilfn`、`conditionexternaldamagetakenmultipliers`、`playerstunlock`、`lastattacker`、`lastattacktype`、`laststimuli`）**全部用 `nil` 作默认值**。

它们在 `CalcDamage`/`GetAttacked`/`DoAttack` 里都做了 `xxx or 1`、`xxx or 0`、`xxx ~= nil and ...` 这种 nil-safe 处理，所以"不写就是默认值"——但**只要你赋一次值，这个字段就活了**。

> 注释掉的字段写在源码里干嘛？两个原因：① **占位文档**——告诉读者"哦原来还能写这些字段"；② **省内存**——一个怪物 prefab 同时有几百个实例，少存几个 `nil` 字段就少几百份指针。

**第三块：SourceModifierList（StackingModifier 容器）**

```64:65:scripts/components/combat.lua
	self.externaldamagemultipliers = SourceModifierList(self.inst) -- damage dealt to others multiplier
	self.externaldamagetakenmultipliers = SourceModifierList(self.inst) -- my damage taken multiplier (post armour reduction)
```

这两个不是 number，是 `SourceModifierList` 实例——能同时叠加多个倍率源，21.2.7 详细讲。

**第四块：回调闭包 + knockback 监听**

```84:101:scripts/components/combat.lua
	self.losetargetcallback = function() self:DropTarget() end
	self.transfertargetcallback = function(target, newtarget)
		if newtarget ~= nil and self:CanTarget(newtarget) then
			self:SetTarget(newtarget)
			if self.target == target and self.target ~= newtarget then
				self:DropTarget()
			end
		else
			self:DropTarget()
		end
	end
	self.allycheckcallback = function(target)
		if self:CanBeAlly(target) then
			self:DropTarget()
		end
	end

	inst:ListenForEvent("knockback", CommonHandlers.ResetHitRecoveryDelay)
end,
```

构造函数里就预先建好三个闭包——`losetargetcallback` 用于"目标进 limbo / 被销毁就脱战"、`transfertargetcallback` 用于"目标转移仇恨"、`allycheckcallback` 用于"目标变成盟友就脱战"。这三个闭包稍后在 `StartTrackingTarget`/`StopTrackingTarget` 时挂到事件上（21.3 详解）。

最后挂的 `knockback` 监听是给"被击退时重置硬直恢复时长"用的——所有挂了 combat 的实体都自动有这个能力。

**第五块（构造尾巴）：props 表**

```104:110:scripts/components/combat.lua
nil,
{
    attackrange = onattackrange,
    min_attack_period = onminattackperiod,
    canattack = oncanattack,
    target = ontarget,
    panic_thresh = onpanicthresh,
})
```

这就是 21.1.8 讲过的 Class setter 钩子表。5 个字段一旦被赋值都会自动同步到 replica。

---

### 21.2.3 完整属性字段速查表

下面这张表是"所有可读写的 self.xxx 字段"，按用途分组。**带 ✅ 同步的字段，是 21.1.3 那张表的扩展版**：

#### 战斗参数（伤害 / 距离 / 冷却）

| 字段 | 类型 | 默认 | 含义 | replica 同步 |
|---|---|---|---|---|
| `defaultdamage` | number | 0 | 徒手伤害（无武器时使用） | ❌ |
| `attackrange` | number | 3 | 基础攻击距离 | ✅ `_attackrange` |
| `hitrange` | number | 3 | 基础命中距离（落地判定，通常 ≥ attackrange） | ❌ |
| `temprange` | number / nil | nil | 临时攻击距离覆盖（`DoAttack` 单次调用用，结束自动清） | ❌ |
| `min_attack_period` | number | 4 | 攻击冷却秒数 | ✅ `classified.minattackperiod`（仅玩家） |
| `areahitrange` | number / nil | nil | 范围伤害半径 | ❌ |
| `areahitdamagepercent` | number / nil | nil | 范围伤害衰减系数（0~1） | ❌ |
| `areahitcheck` | function / nil | nil | 范围伤害命中过滤函数 `(ent, self.inst) -> bool` | ❌ |
| `areahitdisabled` | bool | nil (false) | true 表示禁用 AOE | ❌ |
| `ignorehitrange` | bool | nil (false) | true 表示无视命中距离（投射物常用） | ❌ |
| `hiteffectsymbol` | string | `"marker"` | 命中特效附着的 anim symbol | ❌ |
| `noimpactsound` | bool | nil | true 表示不播命中音 | ❌ |

#### 状态字段（运行时维护）

| 字段 | 类型 | 默认 | 含义 | replica 同步 |
|---|---|---|---|---|
| `target` | Entity / nil | nil | 当前锁定目标 | ✅ `_target` |
| `lasttargetGUID` | number / nil | nil | 上一目标 GUID（IsRecentTarget 用） | ✅ `classified.lastcombattarget`（仅玩家） |
| `canattack` | bool | true | 当前是否能发起攻击 | ✅ `classified.canattack`（仅玩家） |
| `laststartattacktime` | number | 0 | 上次开始攻击时间戳（冷却计算） | ❌ |
| `lastwasattackedtime` | number | 0 | 上次被攻击时间戳 | ❌ |
| `lastwasattackedbytargettime` | number | 0 | 上次被"当前目标"打中的时间戳 | ❌ |
| `lastattacker` | Entity / nil | nil | 最近一次攻击我的人 | ❌ |
| `lastattacktype` | string / nil | nil | 最近一次攻击类型（`"projectile"` 或 nil） | ❌ |
| `laststimuli` | string / nil | nil | 最近一次伤害刺激类型（如 `"electric"`） | ❌ |
| `lastdoattacktime` | number / nil | nil | 上次成功打中目标的时间戳（DoAttack 末尾设） | ❌ |
| `panic_thresh` | number / nil | nil | 恐慌血量阈值（百分比） | ✅ → `_ispanic`（与 health 比较后） |
| `redirected_from` | Entity / nil | nil | "我是被谁的伤害重定向到的"（蜈蚣等用） | ❌ |

#### AI / 仇恨控制

| 字段 | 类型 | 默认 | 含义 | replica 同步 |
|---|---|---|---|---|
| `targetfn` | function / nil | nil | 主动搜目标函数 `(inst) -> Entity?, forcechange?` | ❌ |
| `retargetperiod` | number / nil | nil | 主动搜目标的周期（秒） | ❌ |
| `retargettask` | task / nil | — | 当前的 retarget 定时器 | ❌ |
| `keeptargetfn` | function / nil | nil | 判定是否继续锁定目标 `(inst, target) -> bool` | ❌ |
| `keeptargettimeout` | number | 0 | KeepTarget 下次执行剩余秒 | ❌ |
| `shouldaggrofn` | function / nil | nil | 自定义"是否要仇恨这个目标" | ❌ |
| `shouldavoidaggrofn` | function / nil | nil | 自定义"是否不让别人仇恨我" | ❌ |
| `shouldavoidaggro` | table / nil | nil | 临时不仇恨的目标白名单（按引用计数） | ❌ |
| `forbiddenaggrotags` | table / nil | nil | 禁止仇恨的标签列表 | ❌ |
| `cansuggesttargetfn` | function / nil | nil | "外部建议的目标"是否接受 | ❌ |
| `forcefacing` | bool | true | 攻击瞬间是否强制朝向目标 | ❌ |
| `tough` | bool | false | "硬度"——非 `toughfighter` 标签的攻击者会被弹反 | ❌ |
| `battlecryenabled` | bool | true | 是否允许喊战吼 | ❌ |
| `battlecryinterval` | number / nil | nil（默认 5） | 战吼间隔（秒） | ❌ |
| `nextbattlecrytime` | number / nil | nil | 下次能喊战吼的时间 | ❌ |

#### 伤害公式参数

| 字段 | 类型 | 默认 | 含义 | replica 同步 |
|---|---|---|---|---|
| `damagemultiplier` | number / nil | nil (1) | 基础伤害倍率（全局） | ❌ |
| `damagebonus` | number / nil | nil (0) | 基础伤害加成（直接 + 到结果） | ❌ |
| `playerdamagepercent` | number / nil | nil (1) | NPC 攻击玩家的伤害折扣（只在徒手时生效） | ❌ |
| `pvp_damagemod` | number / nil | nil (1) | 玩家 PVP 伤害倍率 | ❌ |
| `externaldamagemultipliers` | SourceModifierList | (1) | 外部伤害倍率叠加表（造成伤害） | ❌ |
| `externaldamagetakenmultipliers` | SourceModifierList | (1) | 外部受伤倍率叠加表（受到伤害） | ❌ |
| `conditionexternaldamagetakenmultipliers` | array / nil | nil | 条件性受伤倍率函数列表 | ❌ |
| `bonusdamagefn` | function / nil | nil | 条件性 bonus 伤害（穿过护甲后才加） | ❌ |
| `customdamagemultfn` | function / nil | nil | 自定义伤害倍率函数（最后乘） | ❌ |
| `customspdamagemultfn` | function / nil | nil | 自定义特殊伤害倍率函数 | ❌ |

#### 受击 / 死亡 / 反射

| 字段 | 类型 | 默认 | 含义 | replica 同步 |
|---|---|---|---|---|
| `onhitfn` | function / nil | nil | 我被打中时的服务端回调 `(inst, attacker, damage, spdamage)` | ❌ |
| `onhitotherfn` | function / nil | nil | 我打中别人时的服务端回调 | ❌ |
| `onkilledbyother` | function / nil | nil | 我被某人打死时回调 | ❌ |
| `redirectdamagefn` | function / nil | nil | 伤害重定向（吉祥物挡刀、骑乘等） | ❌ |
| `shouldrecoilfn` | function / nil | nil | 是否触发弹反（攻击者武器太软） | ❌ |
| `ignoredamagereflect` | bool | nil | true 表示忽略目标的伤害反射 | ❌ |
| `hurtsound` | string / nil | nil | 受击播放的额外音效（除 hitsound 之外） | ❌ |
| `playerstunlock` | enum / nil | nil（= ALWAYS） | 对玩家施加硬直的等级 | ❌ |
| `blanktask` | task / nil | nil | `BlankOutAttacks` 创建的暂时禁用攻击任务 | ❌ |

> **怎么判断一个字段"能同步"**？最简单：看 `combat.lua` 顶部 `Class(...)` 第 3 个 `props` 表里有没有它的 setter。**只有 5 个有 setter 的会自动同步**：`attackrange`、`min_attack_period`、`canattack`、`target`、`panic_thresh`。其他都得自己 `inst.replica.combat:SetXxx` 或者根本不同步。

---

### 21.2.4 基础 Setter 方法详解

构造函数里 5 个 prop setter 会自动同步，但**真正能让你"安全配置 combat"的入口**是一组显式的 Set 方法。下面挑重点详解。

#### `SetDefaultDamage(damage)`

```217:219:scripts/components/combat.lua
function Combat:SetDefaultDamage(damage)
    self.defaultdamage = damage
end
```

最简单：把徒手伤害设为 `damage`。注意：

- **只在"无武器"时使用**——拿了武器后 `CalcDamage` 会优先用 `weapon.components.weapon:GetDamage()`。
- **玩家也用这个**！`scripts/prefabs/player_common.lua` 第 2675 行写的就是 `SetDefaultDamage(TUNING.UNARMED_DAMAGE)`，所以你裸手打猪伤害 = `TUNING.UNARMED_DAMAGE`。
- **骑乘时被替换**——`CalcDamage` 第 877 行：如果骑着可战斗坐骑，用坐骑的 `defaultdamage`。

#### `SetAttackPeriod(period)`

```119:121:scripts/components/combat.lua
function Combat:SetAttackPeriod(period)
    self.min_attack_period = period
end
```

——一行，把攻击冷却赋值。**注意是先赋值后才走 setter 钩子**，所以会触发 `onminattackperiod` → `SetMinAttackPeriod` → 写入 `classified.minattackperiod`（仅玩家）。

直接给字段赋值 `inst.components.combat.min_attack_period = 2` 也等效——因为 props 钩子机制（21.1.8）。但**推荐用 Set 方法**，更显式且不容易写错字段名（`min_attack_period` 中间是下划线，不少 mod 错写成 `minattackperiod` 不会报错但完全不生效）。

#### `SetRange(attack, hit)`

```147:150:scripts/components/combat.lua
function Combat:SetRange(attack, hit)
    self.attackrange = attack
    self.hitrange = (hit or self.attackrange)
end
```

最容易踩坑的一个！它有两个参数：

- `attack`（**必填**）：基础攻击距离（决定"你能否走到攻击范围内开始攻击"），赋值后通过 props 钩子自动同步到 replica。
- `hit`（**可选**）：基础命中判定距离（决定"挥击落地时还能不能打到目标"），**不同步**。**如果不传，自动等于 `attack`**。

什么时候 `hit > attack`？两种典型场景：

1. **被击退后仍能命中**——比如猪人战士冲上去攻击，玩家被打中后退一点，原本 `attackrange = 3` 可能就够不着了，但 `hitrange = 4` 让"已经发动的这一击"仍能命中。这种"挥击拖尾"是手感的核心。
2. **动作位移**——某些怪物攻击时自己会前冲一段距离（如蜘蛛勇士的扑击），`hitrange` 要覆盖前冲后的位置。

蜘蛛勇士就是典型——它的 `SetRange(TUNING.SPIDER_WARRIOR_ATTACK_RANGE, TUNING.SPIDER_WARRIOR_HIT_RANGE)` 明确给了两个不同值。

> ⚠️ **常见坑**：很多 mod 写 `SetRange(3, 3)` —— 这是无意义的，因为不传第二个参数默认就是相等。但写 `SetRange(3)` 后再单独 `combat.hitrange = 5`，效果同 `SetRange(3, 5)`，也常见。

#### `SetHurtSound(sound)`

```559:561:scripts/components/combat.lua
function Combat:SetHurtSound(sound)
    self.hurtsound = sound
end
```

直接赋值 `self.hurtsound`。它会在 `GetAttacked` 末尾被 SoundEmitter 播放（如果实体没在 limbo）。

> 注意它**不会替换"hit 命中音"**——hit 音是攻击者发出的，由 `GetImpactSound` 根据武器锐度/目标类型动态选择。`hurtsound` 是被攻击者自己额外播的一份"喊疼声"。

#### `SetAreaDamage(range, percent, areahitcheck)`

```156:164:scripts/components/combat.lua
function Combat:SetAreaDamage(range, percent, areahitcheck)
    self.areahitrange = range
	self.areahitcheck = areahitcheck
    if self.areahitrange then
        self.areahitdamagepercent = percent or 1
    else
        self.areahitdamagepercent = nil
    end
end
```

- `range`（数字）：范围伤害半径，`nil` 表示关闭 AOE。
- `percent`（数字，可选）：AOE 伤害衰减系数，默认 1（与主伤害相等）。
- `areahitcheck`（函数，可选）：进一步的命中过滤 `(ent, attacker) -> bool`，返回 false 跳过。

注意第 3 行：**`areahitcheck` 没有 nil 检查就被直接赋值**——所以你想清除 AOE 时调用 `SetAreaDamage(nil)` 也会顺便清掉 areahitcheck（变成 nil）。

#### `SetKeepTargetFunction(fn)` / `SetRetargetFunction(period, fn)`

这两个属于"目标系统"范畴，21.3 详讲，这里只列签名：

```lua
function Combat:SetKeepTargetFunction(fn)
    self.keeptargetfn = fn
end

function Combat:SetRetargetFunction(period, fn)
    self.targetfn = fn
    self.retargetperiod = period
    -- 如果实体未休眠，立刻开一个定时器
end
```

`fn(inst, target) -> bool` 决定 OnUpdate 时是否继续锁目标；`SetRetargetFunction` 同时设置周期搜索目标的定时器。

#### `SetOnHit(fn)`

```221:223:scripts/components/combat.lua
function Combat:SetOnHit(fn)
    self.onhitfn = fn
end
```

——`fn(inst, attacker, damage, spdamage)` 在我被打中时调用。

蜘蛛巢就用了这个：`scripts/prefabs/spider.lua` 第 656 行的 `inst.components.combat:SetOnHit(SummonFriends)` 让蜘蛛被打就召唤同伴。

---

### 21.2.5 进阶档：距离系字段深度剖析

新手只需要会 `SetRange`，进阶读者要弄懂下面这套"距离-范围-命中"的三层结构：

#### 距离最终如何计算

服务端的 `GetAttackRange` 和 `GetHitRange`：

```984:1005:scripts/components/combat.lua
function Combat:GetAttackRange()
    local weapon = self:GetWeapon()
    return weapon ~= nil
        and weapon.components.weapon.attackrange ~= nil
        and math.max(0, self.attackrange + weapon.components.weapon.attackrange)
        or self.attackrange
end

function Combat:CalcAttackRangeSq(target)
    local range = (target or self.target):GetPhysicsRadius(0) + self:GetAttackRange()
    return range * range
end

function Combat:GetHitRange()
    local weapon = self:GetWeapon()
    return self.temprange or weapon ~= nil and weapon.components.weapon.hitrange ~= nil and self.hitrange + weapon.components.weapon.hitrange or self.hitrange
end

function Combat:CalcHitRangeSq(target)
    local range = (target or self.target):GetPhysicsRadius(0) + self:GetHitRange()
    return range * range
end
```

总结成公式：

```
实际攻击距离² = (目标.PhysicsRadius + (self.attackrange + (武器.attackrange 或 0)))²
实际命中距离² = (目标.PhysicsRadius + (self.temprange 或 (self.hitrange + (武器.hitrange 或 0))))²
```

**三个关键点**：

1. **武器距离是相加的**——不是替换。所以"给生物加 +2 攻击距离 buff，应该改 `self.attackrange`"（21.1.10 的最后一坑就是这个）。
2. **目标物理半径会加上去**——所以打一只半径 0.5 的猪和打一只半径 2 的巨鹿，实际生效距离不同。
3. **`temprange` 优先级最高**——它只在 `DoAttack` 单次调用里被设，`ClearAttackTemps` 调用后清空（看 `combat.lua:1055-1058`）。这是给"特殊攻击动作"用的，比如某些 boss 的横扫攻击单次有更大命中。

#### 客户端的距离计算

`combat_replica.lua:82-90` 的 `GetAttackRangeWithWeapon`：

```82:90:scripts/components/combat_replica.lua
function Combat:GetAttackRangeWithWeapon()
    if self.inst.components.combat ~= nil then
        return self.inst.components.combat:GetAttackRange()
    end
    local weapon = self:GetWeapon()
    return weapon ~= nil
        and math.max(0, self._attackrange:value() + weapon.replica.inventoryitem:AttackRange())
        or self._attackrange:value()
end
```

注意它走的是 `weapon.replica.inventoryitem:AttackRange()` —— **因为客户端没有 `weapon.components.weapon`，只有 replica**。这就是为什么武器 prefab 必须把 attackrange 同步到 inventoryitem replica。

#### `areahitrange` 和 `areahitdamagepercent`

```1252:1280:scripts/components/combat.lua
function Combat:DoAreaAttack(target, range, weapon, validfn, stimuli, excludetags, onlyontarget)
    local hitcount = 0
    local x, y, z = target.Transform:GetWorldPosition()
    if onlyontarget then
        local ent = target
        if self:IsValidTarget(ent) and
            (validfn == nil or validfn(ent, self.inst)) then
            self.inst:PushEvent("onareaattackother", { target = ent, weapon = weapon, stimuli = stimuli })
            local dmg, spdmg = self:CalcDamage(ent, weapon, self.areahitdamagepercent)
            ent.components.combat:GetAttacked(self.inst, dmg, weapon, stimuli, spdmg)
            hitcount = hitcount + 1
        end
    else
        local ents = TheSim:FindEntities(x, y, z, range, AREAATTACK_MUST_TAGS, excludetags)
        for i, ent in ipairs(ents) do
            ...
            local dmg, spdmg = self:CalcDamage(ent, weapon, self.areahitdamagepercent)
            ...
        end
    end

    return hitcount
end
```

关键点：

- **AOE 调用 `CalcDamage` 时把 `areahitdamagepercent` 当作 `multiplier` 传进去**——所以 `percent = 0.5` 意味着 AOE 伤害是单次伤害的一半。
- **AREAATTACK_MUST_TAGS = `{"_combat"}`** —— 只命中带 `_combat` 标签的实体。注意 AOE 不要求"目标周围有 combat 组件"，但被命中的必须有 combat。
- **excludetags 默认是 `{"INLIMBO", "notarget", "noattack", "flight", "invisible", "playerghost"}`** —— 这串就是 `combat.lua:112` 的 `AREA_EXCLUDE_TAGS`。

#### `temprange` 和 `temppos`：单次攻击的临时覆盖

```1055:1058:scripts/components/combat.lua
function Combat:ClearAttackTemps()
    self.temppos = nil
    self.temprange = nil
end
```

`DoAttack` 接收 `instrangeoverride` 和 `instpos`，存到 `temprange`/`temppos`，跑完 `ClearAttackTemps` 清掉。**用途**：投射物落地时"命中检测用投射物当前位置 + 自定义半径"，而不是用攻击者位置/距离。

---

### 21.2.6 进阶档：伤害修正栈完全剖析

这是 mod 开发者最容易迷茫的地方——"我要给玩家加 50% 伤害，到底改哪个字段？"

先看 `CalcDamage` 的完整公式（`combat.lua` 第 908-916 行）：

```908:916:scripts/components/combat.lua
	local damage = (basedamage or 0)
        * (basemultiplier or 1)
        * externaldamagemultipliers:Get()
		* damagetypemult
        * (multiplier or 1)
        * playermultiplier
        * pvpmultiplier
		* (self.customdamagemultfn ~= nil and self.customdamagemultfn(self.inst, target, weapon, multiplier, mount) or 1)
        + (bonus or 0)
```

七个乘数 + 一个加成：

| 因子 | 来源字段 | 何时生效 |
|---|---|---|
| `basedamage` | `weapon.components.weapon:GetDamage()` 或 `self.defaultdamage` | 总在生效 |
| `basemultiplier` | `self.damagemultiplier` | nil 时退化为 1 |
| `externaldamagemultipliers:Get()` | `self.externaldamagemultipliers`（SourceModifierList） | 总在生效（默认 1） |
| `damagetypemult` | `self.inst.components.damagetypebonus:GetBonus(target)` | 有 damagetypebonus 组件时 |
| `multiplier` | `DoAttack` 传入或 `areahitdamagepercent` | 调用方决定 |
| `playermultiplier` | `self.playerdamagepercent`（仅徒手 NPC 打玩家） | 见 `CanApplyPlayerDamageMod` |
| `pvpmultiplier` | `self.pvp_damagemod`（仅玩家攻击玩家） | 同时是玩家时 |
| `customdamagemultfn` | `self.customdamagemultfn(inst, target, weapon, multiplier, mount)` | 设了就生效 |
| `+ bonus` | `self.damagebonus` | nil 时退化为 0 |

#### `damagemultiplier` vs `externaldamagemultipliers`

两个都是"全局乘伤害"，区别：

| | `damagemultiplier`（number） | `externaldamagemultipliers`（SourceModifierList） |
|---|---|---|
| 类型 | 单个 number | 多源叠加容器 |
| 同时只能存一个值？ | ✅ 是 | ❌ 否，可以同时挂多个 source 的修饰 |
| 适合谁用 | prefab 本体（角色固有 1.x 倍） | 装备/buff/法术等动态来源 |
| 来源被 Remove 后是否自动清理 | ❌ 需要手动 reset | ✅ 监听 onremove 自动清 |
| 计算方式 | 直接乘 | 默认乘法叠加（也支持加法） |

——所以你写一个"暴击爪子+25% 伤害"的装备：

```lua
-- 装备时
owner.components.combat.externaldamagemultipliers:SetModifier(inst, 1.25)
-- 卸下时
owner.components.combat.externaldamagemultipliers:RemoveModifier(inst)
```

——绝对不要写 `owner.components.combat.damagemultiplier = 1.25`，因为：

1. 其他装备/buff 会冲掉你的值；
2. 卸下时你需要记住"卸下前是多少"才能恢复；
3. 装备 Remove 时无法自动清理。

#### `playerdamagepercent` 的微妙特性

```851:852:scripts/components/combat.lua
	local playermultiplier = CanApplyPlayerDamageMod(target)
	local pvpmultiplier = playermultiplier and self.inst.isplayer and self.pvp_damagemod or 1
```

```862:872:scripts/components/combat.lua
    if weapon ~= nil then
        --No playermultiplier when using weapons
		basedamage, spdamage = weapon.components.weapon:GetDamage(self.inst, target)
        playermultiplier = 1
        ...
    else
        basedamage = self.defaultdamage
        playermultiplier = playermultiplier and self.playerdamagepercent or 1
        ...
    end
```

**两个关键限制**：

1. **手持武器时强制设为 1**——所以"NPC 拿着武器打玩家"不会享受 `playerdamagepercent` 折扣。这就是为什么源码注释写 `only works with NO WEAPON`。
2. **`CanApplyPlayerDamageMod`** 判定目标是玩家或带 `player_damagescale` 标签时才生效（见 `componentutil.lua` 第 1345 行）。

蜂卫兵就用了这个：`scripts/prefabs/beeguard.lua` 第 536 行写 `inst.components.combat.playerdamagepercent = .5` —— 蜂卫兵徒手打玩家伤害减半（但它会拿武器吗？不会，所以一直生效）。

#### `damagebonus`：加法不是乘法

```916:scripts/components/combat.lua
        + (bonus or 0)
```

**这个 `+` 是在所有乘法之后才加的**。所以你设 `damagebonus = 10` 表示"无论怎么乘，最后再加 10 伤害"。

应用场景：固定额外伤害（例如某武器附魔"+10 火焰附伤"）。注意它**也只是 `self.damagebonus`**，没有 SourceModifierList 版——所以多个 buff 想叠加 bonus 需要自己设计。

#### `bonusdamagefn`：穿透护甲后的条件加伤

```630:639:scripts/components/combat.lua
			if damage > 0 then
				--Bonus damage only applies after unabsorbed damage gets through your armor
				if attacker ~= nil and attacker.components.combat ~= nil and attacker.components.combat.bonusdamagefn ~= nil then
					damage = damage + damagetypemult * (attacker.components.combat.bonusdamagefn(attacker, self.inst, damage, weapon) or 0)
				end
				--Planar entities dampen regular damage
				if self.inst.components.planarentity ~= nil then
					damage, spdamage = self.inst.components.planarentity:AbsorbDamage(damage, attacker, weapon, spdamage)
				end
            end
```

**和 `damagebonus` 的区别**：

| | `damagebonus`（字段） | `bonusdamagefn`（函数） |
|---|---|---|
| 静态/动态 | 静态数字 | 动态计算（每次攻击调用） |
| 何时生效 | `CalcDamage` 里，乘法后加 | `GetAttacked` 里，**穿过护甲后**加 |
| 能否依赖目标 | ❌ 攻击者自己的字段 | ✅ 可以根据 target 决定加多少 |
| 例子 | 全局 +10 火焰附伤 | "打过敏目标 +30、否则 0" |

**典型实现——蜜蜂的过敏额外伤害**：

```56:58:scripts/prefabs/bee.lua
local function bonus_damage_via_allergy(inst, target, damage, weapon)
    return (target:HasTag("allergictobees") and TUNING.BEE_ALLERGY_EXTRADAMAGE) or 0
end
```

```215:scripts/prefabs/bee.lua
    inst.components.combat.bonusdamagefn = bonus_damage_via_allergy
```

——攻击对象带 `allergictobees` 标签时返回 `TUNING.BEE_ALLERGY_EXTRADAMAGE`，否则返回 0。注意函数签名是 `(attacker, target, damage, weapon)`，第 3 个 damage 是"已经穿透护甲的"伤害值。

伍迪变海狸时也用 `bonusdamagefn` 给"砍树更狠"加成：

```618:620:scripts/prefabs/woodie.lua
local function beaverbonusdamagefn(inst, target, damage, weapon)
    return (target:HasTag("tree") or target:HasTag("beaverchewable")) and TUNING.BEAVER_WOOD_DAMAGE or 0
end
```

```1255:scripts/prefabs/woodie.lua
    inst.components.combat.bonusdamagefn = beaverbonusdamagefn
```

——变身海狸时 `bonusdamagefn` 设置，变回人形时设回 nil（`woodie.lua` 第 1198 行）。

#### `customdamagemultfn` / `customspdamagemultfn`：最后一个倍率

```915:scripts/components/combat.lua
		* (self.customdamagemultfn ~= nil and self.customdamagemultfn(self.inst, target, weapon, multiplier, mount) or 1)
```

```918:932:scripts/components/combat.lua
	if spdamage ~= nil then
		local spmult =
			damagetypemult *
			pvpmultiplier

        if self.customspdamagemultfn then
            spmult = spmult * (self.customspdamagemultfn(self.inst, target, weapon, multiplier, mount) or 1)
        end

		if spmult ~= 1 then
			spdamage = SpDamageUtil.ApplyMult(spdamage, spmult)
		end
	end
	return damage, spdamage
end
```

**两个关键区别**：

1. **`customdamagemultfn` 作用在 `CalcDamage` 里，最后乘进总伤害**——所以是攻击者自己的乘数。
2. **`bonusdamagefn` 作用在 `GetAttacked` 里，穿透护甲后加**——所以是攻击者对**这次具体攻击**的条件加成。

Wendy 的经典用法：

```277:283:scripts/prefabs/wendy.lua
local function CustomCombatDamage(inst, target)
	local vex_debuff = target:GetDebuff("abigail_vex_debuff")
	return (vex_debuff ~= nil and ( vex_debuff.prefab == "abigail_vex_debuff" or vex_debuff.prefab == "abigail_vex_shadow_debuff" ) and TUNING.ABIGAIL_VEX_GHOSTLYFRIEND_DAMAGE_MOD)
		or (target == inst.components.ghostlybond.ghost and target:HasTag("abigail") and 0)
		or 1
end
```

——目标身上挂着 Abigail 的"激怒 debuff" → 返回伤害倍率；目标是 Wendy 的 Abigail → 返回 0（不能误伤自家姐妹）。

#### 受伤侧：`externaldamagetakenmultipliers`

```628:scripts/components/combat.lua
		damage = damage * damagetypemult * self.externaldamagetakenmultipliers:Get()
```

——它出现在 `GetAttacked` 里（护甲处理之后）。挂个"+50% 受伤"的 debuff，往这里写：

```lua
inst.components.combat.externaldamagetakenmultipliers:SetModifier(buff_source, 1.5, "weakened")
```

#### `conditionexternaldamagetakenmultipliers`：条件式受伤

```1213:1233:scripts/components/combat.lua
function Combat:AddConditionExternalDamageTakenMultiplier(fn)
    self.conditionexternaldamagetakenmultipliers = self.conditionexternaldamagetakenmultipliers or {}
    self:RemoveConditionExternalDamageTakenMultiplier(fn)
    table.insert(self.conditionexternaldamagetakenmultipliers, fn)
end

function Combat:ApplyConditionExternalDamageTakenMultiplier(damage, attacker, weapon)
    local damagetakenmult = 1

    for k, fn in ipairs(self.conditionexternaldamagetakenmultipliers) do
        damagetakenmult = damagetakenmult * (fn(self.inst, attacker, weapon) or 1)
    end

    return damage * damagetakenmult
end
```

——和 `externaldamagetakenmultipliers` 类似，但每个 modifier 是一个**函数**，可根据攻击者、武器动态决定倍率。Add/Remove 配对使用。

完整顺序（受伤侧）：

```
原始伤害 → 闪避（attackdodger 设为 0）→ 装备护甲（inventory:ApplyDamage）
       → 鞍具吸收 → damagetypemult × condition × external → planarentity 吸收
       → spdamage 加进来 → 扣血
```

---

### 21.2.7 进阶档：状态字段与冷却机制

#### `canattack`：能否发起攻击

这是 5 个有 props setter 的字段之一，会同步到 `classified.canattack`（仅玩家）。

`CanAttack`/`LocomotorCanAttack` 都读它（`combat.lua` 第 755 行、`combat_replica.lua` 第 167 行）。注意 replica 版只对玩家有效——其他客户端从来不能预测怪物能不能攻击。

#### `BlankOutAttacks(fortime)`：临时禁用攻击

```170:182:scripts/components/combat.lua
local function OnBlankOutOver(inst, self)
    self.blanktask = nil
    self.canattack = true
end

function Combat:BlankOutAttacks(fortime)
    self.canattack = false

    if self.blanktask ~= nil then
        self.blanktask:Cancel()
    end
    self.blanktask = self.inst:DoTaskInTime(fortime, OnBlankOutOver, self)
end
```

——把 `canattack` 设为 `false`，`fortime` 秒后自动恢复为 `true`。常用场景：

- 玩家刚复活有 N 秒"不能攻击"保护；
- 某些状态（如冻僵）期间禁止攻击；
- 法术施法时强制收刀。

注意它**直接覆盖**`canattack`——所以如果原本 `canattack` 已经是 false，过了 fortime 秒后会被强制设为 true，可能引入 bug。

#### `laststartattacktime` 与 `min_attack_period`：冷却三件套

```127:145:scripts/components/combat.lua
function Combat:InCooldown()
    return self.laststartattacktime ~= nil and self.laststartattacktime + self.min_attack_period > GetTime()
end

function Combat:GetCooldown()
    return self.laststartattacktime ~= nil and math.max(0, self.min_attack_period - GetTime() + self.laststartattacktime) or 0
end

function Combat:ResetCooldown()
    self.laststartattacktime = nil
end

function Combat:RestartCooldown()
    self.laststartattacktime = GetTime()
end

function Combat:OverrideCooldown(cd)
	self.laststartattacktime = GetTime() - self.min_attack_period + cd
end
```

**冷却计算原理**：

- `InCooldown()` 判定"现在到下次能攻击之间是否还有时间"。
- `RestartCooldown()` = `laststartattacktime = GetTime()`，开始一次新冷却。
- `ResetCooldown()` = `laststartattacktime = nil`，立刻清掉冷却。
- `OverrideCooldown(cd)` = "强制设置当前剩余冷却为 cd 秒"——往回拨 `laststartattacktime` 让"距离下次能攻击只剩 cd 秒"。

`StartAttack` 自动调 `RestartCooldown`：

```733:738:scripts/components/combat.lua
function Combat:StartAttack()
    if self.forcefacing and self.target ~= nil and self.target:IsValid() then
        self.inst:ForceFacePoint(self.target:GetPosition())
    end
	self:RestartCooldown()
end

Combat.CancelAttack = Combat.ResetCooldown
```

注意第 740 行是个**直接复用赋值**——`Combat.CancelAttack = Combat.ResetCooldown` —— 这两个方法本质就是同一个东西，"取消攻击" = "重置冷却"。

#### 客户端冷却的差异

```136:159:scripts/components/combat_replica.lua
function Combat:StartAttack()
    if self.inst.components.combat ~= nil then
        self.inst.components.combat:StartAttack()
    elseif self.classified ~= nil then
        self._laststartattacktime = GetTime()
    end
end

function Combat:CancelAttack()
    if self.inst.components.combat ~= nil then
        self.inst.components.combat:CancelAttack()
    elseif self.classified ~= nil then
        self._laststartattacktime = nil
    end
end

function Combat:InCooldown()
    if self.inst.components.combat ~= nil then
        return self.inst.components.combat:InCooldown()
    elseif self.classified ~= nil then
        return self._laststartattacktime ~= nil and self._laststartattacktime + self.classified.minattackperiod:value() > GetTime()
    end
    return false
end
```

**关键差异**：

- 服务端：`laststartattacktime` 是服务端字段，所有客户端都读不到。
- 客户端：`_laststartattacktime` **是 replica 上的本地字段**（21.1.4 讲过），**不是 netvar**，只在该客户端有效。`classified.minattackperiod` 才是同步过来的。

这就是为什么客户端冷却是"预测式"——玩家本地点击攻击的瞬间就把 `_laststartattacktime` 设为当前时间，下次点击时本地判定 InCooldown 就能立刻拒绝，**不用等服务器回包**。如果服务器后来回拒（比如 `canattack` 变成 false），就走 `CancelAttack` 清掉本地冷却。

---

### 21.2.8 老手必读：仇恨系统初始化字段

```47:54:scripts/components/combat.lua
    -- NOTES(JBK): Aggro system for combat to help make things untargetable temporarily.
    -- None of this is saved in a restart and should be instantiated as things go.
    -- shouldaggrofn returns if the owner of this combat component should target the passed in target.
    -- shouldavoidaggro is a table that holds entity references as keys in it for temporary combat no target.
    -- forbiddenaggrotags is an ipairs table that holds tag strings as values in it to avoid targeting.
    self.shouldaggrofn = nil
    self.shouldavoidaggro = nil
    self.forbiddenaggrotags = nil
```

这三个字段在 `ShouldAggro` 里组合使用（`combat.lua` 第 419-445 行）：

```419:445:scripts/components/combat.lua
function Combat:ShouldAggro(target, ignore_forbidden)
    if target ~= nil and
        (self.shouldaggrofn == nil or self.shouldaggrofn(self.inst, target)) and
        (self.shouldavoidaggro == nil or not self.shouldavoidaggro[target]) and
        (target.components.combat == nil or target.components.combat.shouldavoidaggrofn == nil or target.components.combat.shouldavoidaggrofn(self.inst, target))
        then
        if not ignore_forbidden and self.forbiddenaggrotags ~= nil then
            for _, tag in ipairs(self.forbiddenaggrotags) do
                if target:HasTag(tag) then
                    return false
                end
            end
        end
		if target:HasTag("stealth") then
			return false
		end
		if target.components.health ~= nil and (target.components.health.minhealth or 0) > 0 and not target:HasTag("hostile") then
			target = target.components.follower ~= nil and target.components.follower:GetLeader() or target
			if not target.isplayer then
				--npc should not aggro on things that can't be killed (unless hostile!)
				return false
			end
		end
        return true
    end
    return false
end
```

#### `shouldaggrofn`：自定义"要不要主动恨它"

`fn(self.inst, target) -> bool`。返回 false 表示拒绝。

例如：你写一只"和平鸽"的怪物，永远不主动仇恨，除非被打：

```lua
inst.components.combat:SetShouldAggroFn(function(inst, target)
    return false  -- 永远拒绝主动仇恨
end)
```

#### `shouldavoidaggrofn`：自定义"不要让别人恨我"

注意它**写在目标身上**，由攻击者的 `ShouldAggro` 调用：

```423:scripts/components/combat.lua
        (target.components.combat == nil or target.components.combat.shouldavoidaggrofn == nil or target.components.combat.shouldavoidaggrofn(self.inst, target))
```

返回 false → 攻击者不会恨上你。

例如：玩家进入特殊状态"伪装"，让所有怪物都不恨我：

```lua
self.components.combat.shouldavoidaggrofn = function(attacker, self_target)
    return self_target.is_disguised  -- 自己设了 is_disguised 就拒绝被恨
end
```

#### `shouldavoidaggro`：基于引用计数的临时白名单

```397:417:scripts/components/combat.lua
function Combat:SetShouldAggroFn(fn)
    self.shouldaggrofn = fn
end

function Combat:SetShouldAvoidAggro(target)
    self.shouldavoidaggro = self.shouldavoidaggro or {}
    self.shouldavoidaggro[target] = (self.shouldavoidaggro[target] or 0) + 1
end

function Combat:RemoveShouldAvoidAggro(target)
    if self.shouldavoidaggro == nil then
        return
    end
    self.shouldavoidaggro[target] = (self.shouldavoidaggro[target] or 1) - 1
    if self.shouldavoidaggro[target] == 0 then
        self.shouldavoidaggro[target] = nil
    end
    if next(self.shouldavoidaggro) == nil then
        self.shouldavoidaggro = nil
    end
end
```

**用引用计数**！同一个 buff 来源可以多次 `SetShouldAvoidAggro(target)`，多次 `RemoveShouldAvoidAggro(target)` 配对。表为空时自动 nil 化（省内存）。

适合"叠加 buff"场景：A 法术让我不仇恨 X、B 法术也让我不仇恨 X，A 失效时不该把 B 的也擦掉。

#### `forbiddenaggrotags`：禁止仇恨某些标签

```447:464:scripts/components/combat.lua
function Combat:AddNoAggroTag(tag)
    self.forbiddenaggrotags = self.forbiddenaggrotags or {}
    table.insert(self.forbiddenaggrotags, tag)
end

function Combat:RemoveNoAggroTag(tag)
    if self.forbiddenaggrotags == nil then
        return
    end
    table.removearrayvalue(self.forbiddenaggrotags, tag)
    if self.forbiddenaggrotags[1] == nil then
        self.forbiddenaggrotags = nil
    end
end

function Combat:SetNoAggroTags(tags) -- Wants a table in an ipairs table format.
    self.forbiddenaggrotags = tags
end
```

`SetNoAggroTags` 一次性覆盖整个列表，`AddNoAggroTag` / `RemoveNoAggroTag` 增删单个。pigman.lua 就用 `SetNoAggroTags(RETARGET_GUARD_CANT_TAGS)` 区分普通猪和守卫猪的仇恨过滤标签。

> **`ShouldAggro` 第二个参数 `ignore_forbidden`** 是给"分享仇恨"（`ShareTarget`）用的——别人把仇恨目标传给我时**忽略我自己的 forbiddenaggrotags**，因为是"友军提示我打这个"。

---

### 21.2.9 老手必读：硬直、弹反、特殊钩子

#### `playerstunlock`：对玩家的硬直等级

```2421:2429:scripts/constants.lua
-- How does this creature apply stunlock to the player
PLAYERSTUNLOCK =
{
    ALWAYS = nil,--0,
    OFTEN = 1,
    SOMETIMES = 2,
    RARELY = 3,
    NEVER = 4,
}
```

注意 **ALWAYS = nil**，这是默认值。在 `SGwilson.lua` 里被读：

```1867:1870:scripts/stategraphs/SGwilson.lua
                    (   (stunlock == PLAYERSTUNLOCK.NEVER and math.huge) or
                        (stunlock == PLAYERSTUNLOCK.RARELY and TUNING.STUNLOCK_TIMES.RARELY) or
                        (stunlock == PLAYERSTUNLOCK.SOMETIMES and TUNING.STUNLOCK_TIMES.SOMETIMES) or
                        (stunlock == PLAYERSTUNLOCK.OFTEN and TUNING.STUNLOCK_TIMES.OFTEN) or
```

——级别越高（RARELY/NEVER），玩家被这个怪物打中后能更快从受击状态恢复。

蜜蜂、蚊子用 RARELY（小怪不容易让玩家硬直）：

```214:scripts/prefabs/bee.lua
    inst.components.combat:SetPlayerStunlock(PLAYERSTUNLOCK.RARELY)
```

蜂卫兵用 OFTEN（精英怪威胁感强）：

```239:scripts/prefabs/beeguard.lua
                        inst.components.combat:SetPlayerStunlock(PLAYERSTUNLOCK.OFTEN)
```

#### `shouldrecoilfn` / `tough`：弹反与"皮糙肉厚"

```1188:1211:scripts/components/combat.lua
function Combat:SetShouldRecoilFn(fn)
	self.shouldrecoilfn = fn
end

function Combat:ShouldRecoil(attacker, weapon, damage)
	if self.shouldrecoilfn ~= nil then
		local recoil, remaining_damage = self.shouldrecoilfn(self.inst, attacker, weapon, damage)
		if recoil ~= nil then
			if recoil then
				return true, remaining_damage or nil
			end
			return false, remaining_damage or damage
		end
	end

	if self.tough and
		not (attacker ~= nil and attacker:HasTag("toughfighter")) and --TODO reuse toughworker?
		not (weapon ~= nil and weapon.components.weapon ~= nil and weapon.components.weapon:CanDoToughFight())
		then
		return true, nil
	end

	return false, damage
end
```

**优先级**：

1. `shouldrecoilfn` 返回 `(true, dmg)` → 触发弹反，伤害被替换为 `dmg`（可以是 nil，表示完全免伤）；
2. `shouldrecoilfn` 返回 `(false, dmg)` → 不弹反，但伤害仍可被替换；
3. `shouldrecoilfn` 返回 `(nil, ...)` → 走 `tough` 判定；
4. `tough = true` 且攻击者没 `toughfighter` 标签、武器也不能 `CanDoToughFight` → 触发弹反。

弹反会在 `GetAttacked` 里把 `blocked = true`，于是不会 push `attacked` 事件，而是 push `blocked` 事件。

#### `redirectdamagefn`：伤害重定向

```574:scripts/components/combat.lua
    local damageredirecttarget = self.redirectdamagefn ~= nil and self.redirectdamagefn(self.inst, attacker, damage, weapon, stimuli, spdamage) or nil
```

签名 `fn(self.inst, attacker, damage, weapon, stimuli, spdamage) -> redirect_target_entity?`。返回非 nil 时把这次伤害转给目标实体的 combat（递归调用 `:GetAttacked`）。

经典场景：

- **骑乘**：玩家骑着野牛，伤害先打到野牛身上（`SGwilson.lua` 第 5633 行设置）。
- **格挡盾**：玩家正在格挡，把伤害挡掉一部分。
- **影子蜈蚣**：`shadowthrall_centipede.lua` 第 248 行让蜈蚣的每节伤害都重定向到本体。

---

### 21.2.10 mod 实战：扩展 Combat 字段的三种正确姿势

#### 姿势 1：通过 SourceModifierList 加 Buff（推荐）

最常见——做"狂战士 buff +30% 伤害"：

```lua
local function ApplyBerserkBuff(inst, duration)
    -- 加 buff
    inst.components.combat.externaldamagemultipliers:SetModifier(inst, 1.3, "berserk")
    -- 持续 duration 秒后自动撤销
    inst:DoTaskInTime(duration, function()
        if inst:IsValid() and inst.components.combat then
            inst.components.combat.externaldamagemultipliers:RemoveModifier(inst, "berserk")
        end
    end)
end
```

**好处**：

- 多个 buff 同时挂不会冲突（key 不同）；
- 实体被销毁时自动清理（SourceModifierList 监听 onremove）；
- 卸下时一行 RemoveModifier 搞定。

#### 姿势 2：通过钩子函数实现条件式加伤（推荐进阶）

做"对甲虫打额外 50 伤害"——参考蜜蜂的 `bonusdamagefn`：

```lua
local function AntiBeetleBonus(inst, target, damage, weapon)
    return target:HasTag("beetle") and 50 or 0
end

-- 装备时
inst.components.combat.bonusdamagefn = AntiBeetleBonus
-- 卸下时
inst.components.combat.bonusdamagefn = nil
```

**坑**：`bonusdamagefn` **只有一个槽位**！如果你的 buff 和角色自带的 bonusdamagefn 冲突，需要写"合并器"：

```lua
local original_fn = inst.components.combat.bonusdamagefn
inst.components.combat.bonusdamagefn = function(...)
    local original = original_fn and original_fn(...) or 0
    local mine = AntiBeetleBonus(...)
    return original + mine
end
-- 卸下时恢复
inst.components.combat.bonusdamagefn = original_fn
```

这种"链式钩子"是 mod 兼容性的核心技巧——千万不要无脑覆盖角色原有的 bonusdamagefn，会让伍迪砍树海狸 buff 失效！

#### 姿势 3：定义全新的 mod 私有字段 + 网络同步

如果你要给玩家新增"暴击率"字段并让 UI 显示——21.1.9 的"老手档"已经讲过完整代码，核心思路：

```lua
-- modmain.lua 给 player_classified 加 netvar
AddPrefabPostInit("player_classified", function(inst)
    inst.mod_critrate = net_float(inst.GUID, "mymod.critrate", "critratedirty")
    inst.mod_critrate:set(0)
end)

-- 在玩家的 combat 上加自定义函数
inst.components.combat.mod_critrate = 0
function inst.components.combat:SetModCritRate(rate)
    self.mod_critrate = rate
    if self.inst.player_classified ~= nil then
        self.inst.player_classified.mod_critrate:set(rate)
    end
end
```

——**关键是不要直接覆盖 combat 的 `customdamagemultfn`**，否则会和 Wendy/Wanda/WX78 的内置实现冲突。最好用 `externaldamagemultipliers` 配合 dirty 事件。

---

### 21.2.11 老手必读：OnRemoveFromEntity 与生命周期清理

```1299:1312:scripts/components/combat.lua
function Combat:OnRemoveFromEntity()
    if self.target ~= nil then
        self:StopTrackingTarget(self.target)
    end
    if self.blanktask ~= nil then
        self.blanktask:Cancel()
        self.blanktask = nil
    end
    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end
	self.inst:RemoveEventCallback("knockback", CommonHandlers.ResetHitRecoveryDelay)
end
```

当 `inst:RemoveComponent("combat")` 或实体被销毁时，组件框架会调用这个方法。它清理三件事：

1. **停止跟踪当前目标**（移除 enterlimbo/onremove/transfercombattarget/leaderchanged 监听）；
2. **取消 BlankOutAttacks 的定时器**；
3. **取消 SetRetargetFunction 的周期定时器**；
4. **移除自己挂在 knockback 上的监听**。

> **注意 replica 没有 `OnRemoveFromEntity`！** 看 `combat_replica.lua` 第 19-31 行那段被注释掉的代码——`V2C: OnRemoveFromEntity not supported`。也就是说**你不能在运行中安全地 `UnreplicateComponent("combat")`**——这是引擎已知的限制。

#### 实体休眠/唤醒：`OnEntitySleep` / `OnEntityWake`

```284:304:scripts/components/combat.lua
function Combat:OnEntitySleep()
    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end
end

function Combat:OnEntityWake()
    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end

    if self.retargetperiod ~= nil then
        self.retargettask = self.inst:DoPeriodicTask(self.retargetperiod, dotryretarget, self.retargetperiod*math.random(), self)
    end

    if self.target ~= nil and self.keeptargetfn ~= nil then
        self.inst:StartUpdatingComponent(self)
    end
end
```

**关键点**：

- 睡眠时取消主动搜目标的定时器（省 CPU）；
- 唤醒时重启定时器，并恢复 OnUpdate（如果当前还有目标且需要 keeptarget 判定）。

——所以如果你在 mod 里写了"周期任务"挂在 combat 上，记得遵守同样的 sleep/wake 规则，否则远离玩家的实体会一直跑后台任务、卡服务器。

---

### 21.2.12 五个初始化常见坑

#### 坑 1：在 `master_postinit` 之前调用 `inst.components.combat:Set...`

```lua
local function fn()
    local inst = CreateEntity()
    -- ...
    inst.entity:SetPristine()

    -- ❌ 在这之前还没 AddComponent！
    inst.components.combat:SetDefaultDamage(20)  -- nil error

    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("combat")  -- 应该先这里
end
```

**正确顺序**：所有 `AddComponent` 必须在 `if not TheWorld.ismastersim then return inst end` **之后**——因为客户端没有 `inst.components`。同时 `Set` 调用必须在 `AddComponent` 之后。

#### 坑 2：`SetRange` 第二个参数想用变量，结果传成 nil

```lua
local hit = config.hitrange  -- 配置可能返回 nil
inst.components.combat:SetRange(3, hit)  -- 如果 hit 是 nil，hitrange 会被设成 attackrange (3)
```

`SetRange` 内部：`self.hitrange = (hit or self.attackrange)`。**nil 会被 fallback 成 attack 值，不会报错**——但很多时候你想要的是"用之前的 hitrange"，结果被悄悄改了。

正确：

```lua
inst.components.combat:SetRange(3)
if hit ~= nil then
    inst.components.combat.hitrange = hit
end
```

#### 坑 3：`bonusdamagefn` 覆盖了角色原有钩子

特别针对玩家：Wendy 设置了 `customdamagemultfn`、Woodie 用 `bonusdamagefn`、伍尔夫用 `customdamagemultfn`……你的 mod 如果无脑覆盖：

```lua
AddPrefabPostInit("woodie", function(inst)
    if not TheWorld.ismastersim then return end
    inst:DoTaskInTime(0, function()
        inst.components.combat.bonusdamagefn = my_bonus  -- ❌ 覆盖了海狸砍树加成！
    end)
end)
```

**正确做法**：链式包裹（见 21.2.10 姿势 2）。

#### 坑 4：把武器伤害写在 `defaultdamage` 上

```lua
inst.components.combat:SetDefaultDamage(60)  -- 武器伤害？
-- 玩家拿着武器伤害还是按 weapon.components.weapon.damage 算，defaultdamage 完全不生效！
```

**`defaultdamage` 只在"无武器"时生效**。武器伤害应该写在 `weapon.components.weapon:SetDamage()`。

#### 坑 5：客户端读不到自定义 combat 字段

```lua
-- 服务端
inst.components.combat.my_critrate = 0.5

-- 客户端
local rate = inst.replica.combat.my_critrate  -- nil，永远读不到
```

`replica.combat` 是**另一个 Class 实例**（21.1.4 讲过），跟 `components.combat` 完全无关。要让客户端读到自定义字段，必须走 netvar（见 21.1.9 老手档 + 21.2.10 姿势 3）。

---

### 21.2.13 五个属性查询/调试小技巧

**1. 在控制台快速查目标和距离**

```lua
local p = ConsoleCommandPlayer()
local c = p.components.combat
print("target:", c.target, "attackrange:", c:GetAttackRange(), "hitrange:", c:GetHitRange())
```

**2. 看 InCooldown 状态**

```lua
print(p.components.combat:InCooldown(), p.components.combat:GetCooldown())
```

**3. 查看 SourceModifierList 内容**

```lua
local list = p.components.combat.externaldamagemultipliers
print("current:", list:Get())
for source, src_params in pairs(list._modifiers) do
    print("  ", tostring(source), src_params.modifiers)
end
```

**4. 看实体的 combat debug 字符串**

```lua
print(p.components.combat:GetDebugString())
```

——内置的 `GetDebugString`（`combat.lua` 第 491-508 行）会打印 target、damage、距离、冷却、能否攻击、能否被攻击。

**5. 临时禁用 AOE 调试**

```lua
p.components.combat:EnableAreaDamage(false)  -- 暂时关 AOE
-- ...
p.components.combat:EnableAreaDamage(true)   -- 恢复
```

---

### 本节涉及的标签

| 标签 | 含义 | 作用 |
|---|---|---|
| `_combat` | "本实体有 combat 组件" | `AddComponent("combat")` 时自动添加（见 21.1.6） |
| `allergictobees` | "对蜂蜇过敏" | 蜜蜂 / 蜂卫兵 / 蜂后的 `bonusdamagefn` 用它加额外伤害 |
| `tree` / `beaverchewable` | 树 / 海狸可啃 | 伍迪海狸形态的 `bonusdamagefn` 用它加伤害 |
| `toughfighter` | "强力战士" | 能突破 `tough = true` 的弹反 |
| `player_damagescale` | "适用玩家伤害折扣" | `CanApplyPlayerDamageMod` 用它扩展非玩家但也享受/承担 playerdamagepercent |
| `propweapon` | "道具武器"（如玩具锤子） | PVP 关闭时仍允许 PVP 互打 |
| `INLIMBO` / `notarget` / `noattack` / `flight` / `invisible` / `playerghost` | 战斗豁免标签 | `AREA_EXCLUDE_TAGS` 中过滤 |
| `stealth` | "潜行" | `ShouldAggro` 直接返回 false |
| `hostile` | "敌对状态" | `ShouldAggro` 中影响"最小血量过滤" |

### 本节涉及的事件

| 事件名 | 推送对象 | 推送时机 | 数据键值 |
|---|---|---|---|
| `knockback` | combat 所属实体 | knockback 组件 / 弹反逻辑推送 | （随推送方）—— Combat 构造时挂监听 `CommonHandlers.ResetHitRecoveryDelay` 重置硬直 |
| `attacked`（服务端） | 被攻击实体 | `Combat:GetAttacked` 未 blocked 时（详 21.8） | `attacker` / `damage` / `damageresolved` / `original_damage` / `weapon` / `stimuli` / `spdamage` / `redirected` / `noimpactsound` |
| `blocked` | 被攻击实体 | `GetAttacked` 被 blocked 时 | `attacker` / `damage` / `spdamage` / `original_damage` |
| `onhitother` | 攻击者本身 | 成功打中目标后 | `target` / `damage` / `damageresolved` / `stimuli` / `spdamage` / `weapon` / `redirected` |
| `killed` | 攻击者本身 | 攻击导致目标死亡 | `victim` / `attacker` |
| `onmissother` | 攻击者本身 | DoAttack 中 `CanHitTarget` 失败 | `target` / `weapon` |
| `onattackother` | 攻击者本身 | DoAttack 中即将攻击时 | `target` / `weapon` / `projectile` / `stimuli` |
| `onareaattackother` | 攻击者本身 | DoAreaAttack 命中每个范围目标时 | `target` / `weapon` / `stimuli` |
| `weapontooweak` | 攻击者本身 | `tough` 弹反触发且伤害为 0/nil 时 | 无（在 stategraph 里被监听显示"武器太弱"提示） |
| `recoil_off` | 攻击者本身 | DoAttack 中触发弹反时 | `target` |
| `onreflectdamage` | 装备/角色被反弹时 | 反射伤害应用后 | `inst` / `attacker` / `reflected_dmg` / `reflected_spdmg` |

---



## 21.3 目标系统：寻找、锁定、保持、丢弃

### 本节导读

21.1 把战斗组件的"分布"讲清楚了，21.2 把战斗组件的"属性"讲清楚了，但战斗的真正灵魂是——**怪物怎么知道要打谁？打到一半玩家跑远了要不要追？目标死了/进容器了/变成盟友了怎么处理？**这就是"目标系统"。

很多 mod 开发者写怪物时遇到的第一个百思不解的 bug 就在这里：

- "我怎么 `SetTarget` 都没反应，怪物站着不动？"
- "目标进了王国/被吞食了，怪物还在原地疯狂攻击空气！"
- "为什么我的怪物刚锁定玩家，下一秒又把目标换成同伴？"
- "明明 `RetargetFn` 写对了，为啥我的猪几分钟都不主动找目标？"

这些都属于"目标系统"的核心问题。

> **目标流转一句话总结**：
>
> - **找**（Retarget）：周期性主动搜索目标 → `targetfn` + `retargetperiod`
> - **设**（SetTarget / SuggestTarget / EngageTarget）：把找到的目标"挂"到 `self.target` 上
> - **跟**（StartTrackingTarget）：监听目标的 enterlimbo/onremove/transfercombattarget/leaderchanged 四个事件
> - **保**（KeepTargetFn / OnUpdate）：每秒判断"是否还应该锁着这个目标"
> - **弃**（DropTarget / GiveUp）：主动放弃或被迫放弃

读完本节，你将能回答：

- "`SetTarget` 和 `SuggestTarget` 有什么区别？什么时候用哪个？"
- "`SetRetargetFunction(3, fn)` 里的 `3` 到底是什么周期？怪物会不会一离玩家视野就停下来？"
- "目标进了 limbo（被吃掉/被装进容器），为啥怪物会自动脱战？"
- "`keeptargetfn` 返回 false 是立刻脱战吗？为什么有时候要等好几秒？"
- "`ShareTarget`、`SuggestTarget`、`SetTarget` 到底是怎样的调用层级？"
- "`iframeskeepaggro` 标签是干嘛的？"
- "我自定义的怪物为什么休眠后再唤醒就不主动搜目标了？"

> **新手**先看 21.3.1-21.3.5——理解四阶段流程、记住"SetTarget 不是简单赋值，是组合操作"、掌握最常用 5 个 API；**进阶读者**继续看 21.3.6-21.3.9，深入跟踪机制、SuggestTarget 链路、ShareTarget 分享算法、休眠唤醒对搜目标的影响；**老手**跳到 21.3.10-21.3.13，掌握 keeptargettimeout 精细控制、iframeskeepaggro 特殊机制、与 brain 节点的协作、mod 扩展正确姿势。

---

### 21.3.1 快速入门：目标系统的四阶段全景图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         服务端 Combat 组件的"目标流水线"                       │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  阶段一：寻找（Retarget）                                                ││
│  │                                                                          ││
│  │  SetRetargetFunction(period, targetfn)                                   ││
│  │   └→ DoPeriodicTask(period, dotryretarget)                              ││
│  │       └→ TryRetarget()                                                  ││
│  │           └→ local newtarget = targetfn(self.inst)                      ││
│  │               └→ SetTarget(newtarget) 或 SuggestTarget(newtarget)       ││
│  │                                                                          ││
│  │  + brain 节点（ChaseAndAttack）可能也会主动找目标                       ││
│  │  + 战斗时被攻击 → OnAttacked 回调 → SetTarget(attacker)                 ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│                                ↓                                             │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  阶段二：锁定（Engage）                                                  ││
│  │                                                                          ││
│  │  SetTarget(target)                                                       ││
│  │   └→ 1. IsValidTarget 过滤、ShouldAggro 过滤、hiding 状态过滤            ││
│  │      2. DropTarget(hasnexttarget = true) 卸掉旧目标的事件监听             ││
│  │      3. EngageTarget(target, oldtarget)                                  ││
│  │          ├→ self.target = target                                         ││
│  │          ├→ PushEvent("newcombattarget", { target, oldtarget })          ││
│  │          ├→ StartTrackingTarget(target) ← 挂4个事件监听                  ││
│  │          ├→ StartUpdatingComponent ← 启动 OnUpdate                       ││
│  │          └→ 如果目标是我的 leader：自动 RemoveFollower（撕破关系）       ││
│  │                                                                          ││
│  │  → ontarget(self, target) ← Class setter 钩子                            ││
│  │      └→ inst.replica.combat:SetTarget(target) ← 同步给客户端             ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│                                ↓                                             │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  阶段三：保持（Keep）                                                    ││
│  │                                                                          ││
│  │  每帧 OnUpdate(dt)：                                                     ││
│  │   ├→ 如果 self.target == nil → StopUpdatingComponent                     ││
│  │   ├→ keeptargettimeout -= dt                                             ││
│  │   ├→ 如果还 > 0，等下次                                                  ││
│  │   ├→ ≤ 0 时：                                                            ││
│  │   │   - 目标 IsInLimbo / 不 IsValid → drop                              ││
│  │   │   - keeptargetfn(inst, target) 返回 false → drop                    ││
│  │   │   - target.combat:CanBeAttacked(inst) 返回 false → drop             ││
│  │   └→ 否则继续锁定                                                        ││
│  │                                                                          ││
│  │  四个事件回调（StartTrackingTarget 挂的）：                              ││
│  │   ├→ enterlimbo  → losetargetcallback → DropTarget()                    ││
│  │   ├→ onremove    → losetargetcallback → DropTarget()                    ││
│  │   ├→ transfercombattarget → transfertargetcallback → 切换/丢弃           ││
│  │   └→ leaderchanged → allycheckcallback → 检查是否变盟友 → 可能 DropTarget││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│                                ↓                                             │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  阶段四：丢弃（Drop / GiveUp）                                           ││
│  │                                                                          ││
│  │  DropTarget(hasnexttarget?)                                              ││
│  │   ├→ SetLastTarget(self.target) ← 记录到 lasttargetGUID                  ││
│  │   ├→ StopTrackingTarget(target) ← 卸4个事件监听                          ││
│  │   ├→ StopUpdatingComponent                                               ││
│  │   ├→ self.target = nil                                                   ││
│  │   ├→ 不是切换目标的情况下：PushEvent("droppedtarget", { target })        ││
│  │   └→ lastwasattackedbytargettime = 0                                     ││
│  │                                                                          ││
│  │  GiveUp()                                                                ││
│  │   ├→ talker:Say(...) 喊放弃台词                                          ││
│  │   ├→ PushEvent("giveuptarget", { target })                               ││
│  │   └→ DropTarget()                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────┘
```

**理解这张图的关键三句话**：

1. **目标变化的所有入口最终都会汇集到 `SetTarget`**——`SetTarget` 才是"真正改变 `self.target` 的地方"，而它内部又调用 `DropTarget` + `EngageTarget` 两个子步骤。所以**你直接 `self.target = X` 是错的**（虽然 props setter 钩子会触发同步，但跳过了所有过滤逻辑），永远走 `SetTarget`。
2. **目标的"死亡通知"是事件机制，不是轮询**——`enterlimbo`/`onremove` 是事件，目标自己一被销毁/进容器，立刻通知所有锁定它的怪物。这就是为什么"猪进了宝箱、所有打它的怪物会立刻停手"。
3. **`OnUpdate` 是兜底机制**——事件通知抓不到的情况（比如目标隔得太远、状态变了、变成了盟友），靠每秒一次的 `keeptargetfn` 检查兜底。

---

### 21.3.2 五个核心方法速查表

| 方法 | 签名 | 作用 | 何时调用 |
|---|---|---|---|
| `SetTarget(target)` | `Combat:SetTarget(target?)` | **设置新目标，最常用入口**；内部走 IsValidTarget + ShouldAggro + hiding 过滤、然后切换 | OnAttacked、Brain 节点、RetargetFn 找到目标后 |
| `SuggestTarget(target)` | `Combat:SuggestTarget(target?)` | **建议目标**——仅当 `self.target == nil` 且 `cansuggesttargetfn` 通过时才接受；返回 true/nil 表示是否被采纳 | ShareTarget 内部、外部 prefab/staff 给生物建议目标 |
| `DropTarget(hasnexttarget?)` | `Combat:DropTarget(hasnexttarget?: bool)` | **放弃当前目标**——卸事件监听、清 `self.target`、push `droppedtarget` 事件 | 目标失效、KeepTarget 判定失败、SetTarget 切换时内部调用 |
| `GiveUp()` | `Combat:GiveUp()` | **优雅放弃**——播台词 + push `giveuptarget` + 再调 DropTarget | brain 失去耐心、追击超时 |
| `EngageTarget(target, oldtarget?)` | `Combat:EngageTarget(target, oldtarget?)` | **底层锁定操作**——push `newcombattarget`、StartTrackingTarget、启动 OnUpdate | **不建议直接调用**，由 SetTarget 内部分派 |

辅助方法：

| 方法 | 签名 | 作用 |
|---|---|---|
| `HasTarget()` | `Combat:HasTarget() -> bool` | `self.target ~= nil` |
| `TargetIs(target)` | `Combat:TargetIs(target) -> bool` | `target ~= nil and self.target == target` |
| `IsRecentTarget(target)` | `Combat:IsRecentTarget(target) -> bool` | 当前目标 **或** `target.GUID == self.lasttargetGUID` |
| `ValidateTarget()` | `Combat:ValidateTarget() -> bool` | 当前目标若失效则立刻 DropTarget |
| `SetLastTarget(target)` | `Combat:SetLastTarget(target?)` | 写入 `self.lasttargetGUID` + 同步给 classified（仅玩家） |

配置方法：

| 方法 | 签名 | 作用 |
|---|---|---|
| `SetRetargetFunction(period, fn)` | `Combat:SetRetargetFunction(period: number?, fn: function?)` | 设置周期搜目标——会立即启动 DoPeriodicTask |
| `SetKeepTargetFunction(fn)` | `Combat:SetKeepTargetFunction(fn: function?)` | 设置"是否继续锁定"判定 |
| `SetCanSuggestTargetFn(fn)` | `Combat:SetCanSuggestTargetFn(fn: function?)` | 设置"是否接受外部建议"判定 |
| `SetShouldAggroFn(fn)` | `Combat:SetShouldAggroFn(fn: function?)` | 设置主动仇恨过滤（详见 21.2.8） |
| `TryRetarget()` | `Combat:TryRetarget()` | 立刻尝试一次搜目标（不等周期到） |

---

### 21.3.3 SetTarget：所有目标变化的中央入口

源码非常精炼：

```466:475:scripts/components/combat.lua
function Combat:SetTarget(target)
    if target ~= self.target and
        (target == nil or (self:IsValidTarget(target) and self:ShouldAggro(target))) and
		not (target and target.isplayer and target.sg and target.sg:HasStateTag("hiding"))
	then
		local oldtarget = self.target
        self:DropTarget(target ~= nil)
		self:EngageTarget(target, oldtarget)
    end
end
```

读这九行代码要看到三个守卫层：

#### 守卫 1：`target ~= self.target`

**重复设置同一个目标会被无视**。看似简单但很重要——避免每次 OnUpdate 触发的"还要继续打"误触发一次 `newcombattarget` 事件、误打事件监听卸了又挂浪费 CPU。

#### 守卫 2：`target == nil or (IsValidTarget and ShouldAggro)`

**两个 nil-safe 分支**：

- `target == nil`：跳过验证，直接进入 DropTarget+EngageTarget(nil)。等同"手动清除目标"。
- `target ~= nil`：必须**同时通过** `IsValidTarget` 和 `ShouldAggro` 才能设置。

`IsValidTarget` 内部代理到 `replica.combat:IsValidTarget`（见 21.1.4 + `combat_replica.lua:288-305`），检查：

- 目标存在、不是自己、entity 可见
- 不是死亡、不在 spawnprotection、不是不可见的暗影/鬼魂（除非攻击者疯狂）
- PVP 关闭时不打玩家（除非装备 propweapon）
- 高度 ≤ attackrange（飞行高度的怪物可豁免）

`ShouldAggro` 检查（见 `combat.lua:419-445`）：

- `shouldaggrofn` 自定义过滤
- `shouldavoidaggro` 临时白名单
- `target.shouldavoidaggrofn` 目标拒绝
- `forbiddenaggrotags` 标签黑名单
- target 有 `stealth` 标签 → 拒绝
- target 不能被杀（minhealth > 0、非 hostile、非 player） → 拒绝

注意第三个守卫前的 **and 短路**——只要 `IsValidTarget` 或 `ShouldAggro` 任一返回 false，整个 `SetTarget` 静默失败。这是初学者常见困惑点：**"我调了 SetTarget 没生效，但没报错"**——99% 都是这两个守卫之一拒绝了。

#### 守卫 3：`hiding` 状态过滤

```469:scripts/components/combat.lua
		not (target and target.isplayer and target.sg and target.sg:HasStateTag("hiding"))
```

——玩家用万圣节南瓜灯、藏在草丛等状态里时不被设为目标。这是个**很容易被忽视的细节**：你写的怪物如果"奇怪地无视玩家"，先查一下玩家是不是处于 `hiding` 状态。

#### 内部分派：DropTarget + EngageTarget

```472:473:scripts/components/combat.lua
        self:DropTarget(target ~= nil)
		self:EngageTarget(target, oldtarget)
```

`DropTarget(target ~= nil)` 这一行很关键：

- `target ~= nil` → `hasnexttarget = true` → **不 push `droppedtarget` 事件**（因为马上要锁新目标）
- `target == nil` → `hasnexttarget = false` → **push `droppedtarget` 事件**（真的脱战了）

为什么要这样区分？因为 `droppedtarget` 事件常被 brain 节点监听用作"主动脱战的信号"。如果切换目标时也 push 一次 `droppedtarget`，brain 会误判"脱战了"做出错误反应（比如停下来啃食、晃悠）。这是个**对外行为契约**：`droppedtarget` 仅在真正没目标时推送。

#### 完整 EngageTarget 内部

```381:395:scripts/components/combat.lua
function Combat:EngageTarget(target, oldtarget)
    if target then
		oldtarget = self.target or oldtarget
        self.target = target
        self.inst:PushEvent("newcombattarget", {target=target, oldtarget=oldtarget})
        self:StartTrackingTarget(target)
        if self.keeptargetfn then
            self.inst:StartUpdatingComponent(self)
        end
        local leader = self.inst.components.follower and self.inst.components.follower:GetLeader()
        if leader and leader == target and leader.components.leader and not self.inst.components.follower.keepleaderonattacked then
            leader.components.leader:RemoveFollower(self.inst)
        end
    end
end
```

**五件事**：

1. **赋值 `self.target`**——触发 props setter `ontarget` → 同步给 replica → 客户端能立刻知道；
2. **推 `newcombattarget` 事件**——brain 可以监听到目标变化；
3. **`StartTrackingTarget`**——挂 4 个事件监听（详见 21.3.6）；
4. **启动 `OnUpdate`**——仅当 `keeptargetfn` 存在时（21.3.5 详讲）；
5. **跟随者反叛**——如果新目标是自己的 leader，且 follower 没有 `keepleaderonattacked = true`，**强制脱离 leader**！这是一段很多 mod 开发者不知道的逻辑——"你打你领主，关系就破裂了"。

> **mod 开发提示**：你做"忠心永不背叛"的宠物，需要在 follower 上设 `follower.keepleaderonattacked = true`，否则宠物一打主人就脱离队伍，主人就找不回宠物了。

---

### 21.3.4 SetRetargetFunction：让生物自己主动搜目标

#### 配置方法的源码

```270:282:scripts/components/combat.lua
function Combat:SetRetargetFunction(period, fn)
    self.targetfn = fn
    self.retargetperiod = period

    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end

	if period and fn and not self.inst:IsAsleep() then
        self.retargettask = self.inst:DoPeriodicTask(period, dotryretarget, period*math.random(), self)
    end
end
```

四步：

1. 存 `targetfn`、`retargetperiod` 到字段；
2. 取消旧的定时器；
3. 仅当 period **和** fn 都有值 **且** 实体没在休眠时，开新定时器；
4. **初始延迟是 `period * math.random()`**——这是个非常重要的细节，避免同时刷出的 100 只猪都在同一帧搜目标，让 CPU 抖动。每个实例的延迟独立随机，自然错峰。

> **mod 实践**：你想暂时禁用搜目标？最简洁的方式是 `SetRetargetFunction(nil, nil)`——同时清掉两个字段，相当于"忘了搜索"，brain 还能用 `findnewtargetfn` 兜底（21.3.11 老手实战会展开）。

#### TryRetarget 内部

```245:268:scripts/components/combat.lua
function Combat:TryRetarget()
    if self.targetfn ~= nil
        and not (self.inst.components.health ~= nil and
                self.inst.components.health:IsDead())
        and not (self.inst.components.sleeper ~= nil and
                self.inst.components.sleeper:IsInDeepSleep()) then

        local newtarget, forcechange = self.targetfn(self.inst)
        if newtarget ~= nil and newtarget ~= self.target and not newtarget:HasTag("notarget") then

            if forcechange then
                self:SetTarget(newtarget)
			    self.lastwasattackedbytargettime = GetTime()
            elseif self.target ~= nil and self.target:HasTag("structure") and not newtarget:HasTag("structure") then
                self:SetTarget(newtarget)
			    self.lastwasattackedbytargettime = GetTime()
            else
                if self:SuggestTarget(newtarget) then
					self.lastwasattackedbytargettime = GetTime()
				end
            end
        end
    end
end
```

**注意几个细节**：

1. **死亡 / 沉睡时跳过**——这是为了避免"死掉的怪物仍在搜目标 / 沉睡的怪物醒来一帧就被吵醒"。
2. **`targetfn` 返回两个值**——`(newtarget, forcechange)`。第二个 `forcechange = true` 意思是"强制覆盖现有目标"。否则走"建议"路线（SuggestTarget）。
3. **新目标带 `notarget` 标签直接跳过**——双层安全保障。
4. **三段式决策**：
   - `forcechange = true` → 直接 `SetTarget(newtarget)`（覆盖）；
   - 当前目标是 structure（墙、储物箱等） 而新目标不是 → 直接 `SetTarget(newtarget)`（生物优先于结构）；
   - 否则 → `SuggestTarget(newtarget)`（仅当 `self.target == nil` 时才接受）。

第二条非常有意思——很多怪物会"先恨上路上挡路的墙，但只要看到玩家就立刻转向打玩家"。比如海盗船边的猴子，会先攻击船板，看到玩家就忘记船板。这条逻辑就让 RetargetFn 不必显式判断。

#### dotryretarget 是定时器调度入口

```241:243:scripts/components/combat.lua
local function dotryretarget(inst, self)
    self:TryRetarget()
end
```

——只是个跳板，把 self 闭包进去。`DoPeriodicTask` 的第三个参数是初始延迟、第四个开始才是回调参数。

#### 典型 RetargetFn 模板：以猪人为例

```217:247:scripts/prefabs/pigman.lua
local function NormalRetargetFn(inst)
    if inst:HasTag("NPC_contestant") then
        return nil
    end

	local exclude_tags = { "playerghost", "INLIMBO" , "NPC_contestant" }
	if inst.components.follower:GetLeader() ~= nil then
		table.insert(exclude_tags, "abigail")
	end
	if inst.components.minigame_spectator ~= nil then
		table.insert(exclude_tags, "player") -- prevent spectators from auto-targeting webber
	end

    local oneof_tags = {"monster","wonkey","pirate"}
    if not inst:HasTag("merm") then
        table.insert(oneof_tags, "merm")
    end

    return not inst:IsInLimbo()
        and FindEntity(
                inst,
                TUNING.PIG_TARGET_DIST,
                function(guy)
                    return guy:IsInLight() and inst.components.combat:CanTarget(guy)
                end,
                RETARGET_MUST_TAGS, -- see entityreplica.lua
                exclude_tags,
                oneof_tags
            )
        or nil
end
```

学习四个模式：

**模式 1：早期返回——某些"豁免状态"直接 nil**

```lua
if inst:HasTag("NPC_contestant") then
    return nil
end
```

参加迷你游戏（NPC_contestant 是参赛者标签）的猪不应该主动找目标。

**模式 2：动态构造 exclude_tags / oneof_tags**

```lua
local exclude_tags = { "playerghost", "INLIMBO" , "NPC_contestant" }
if inst.components.follower:GetLeader() ~= nil then
    table.insert(exclude_tags, "abigail")
end
```

——根据实体当前状态调整搜索条件。"猪有 leader 时不要打温蒂的姐姐"。

**模式 3：FindEntity + 自定义过滤 + 标签三件套**

`FindEntity(inst, radius, fn, musttags, canttags, mustoneoftags)` 是搜目标的"瑞士军刀"：

| 参数 | 类型 | 作用 |
|---|---|---|
| `inst` | Entity | 搜索中心 |
| `radius` | number | 搜索半径 |
| `fn` | function `(guy, inst) -> bool` 或 nil | 进一步自定义过滤（最贵） |
| `musttags` | table 或 nil | 必须**全部**包含的标签 |
| `canttags` | table 或 nil | 必须**全部**不包含的标签 |
| `mustoneoftags` | table 或 nil | 必须**至少**包含一个的标签 |

性能优化：**优先用标签过滤**（C++ 层加速），自定义 fn 是最后的兜底。

源码：

```31:42:scripts/simutil.lua
function FindEntity(inst, radius, fn, musttags, canttags, mustoneoftags)
    if inst ~= nil and inst:IsValid() then
        local x, y, z = inst.Transform:GetWorldPosition()
        local ents = TheSim:FindEntities(x, y, z, radius, musttags, canttags, mustoneoftags)
        for i, v in ipairs(ents) do
            if v ~= inst and v.entity:IsVisible() and (fn == nil or fn(v, inst)) then
                return v
            end
        end
    end
end
```

— `FindEntity` 是返回**第一个**符合条件的实体（不是最近的）。如果要找最近的，用 `FindClosestEntity`。

**模式 4：`RETARGET_MUST_TAGS = { "_combat" }`**

```450:scripts/prefabs/pigman.lua
local RETARGET_MUST_TAGS = { "_combat" }
```

——**用 `_combat` 标签限定"只搜索有 combat 组件的实体"**。这是 21.1.6 讲过的关键标签：`AddComponent("combat")` 会自动打。**几乎所有 RetargetFn 都该用这条 musttag**，否则会跑遍场景里的食物、家具浪费 CPU。

> 注释 `-- See entityreplica.lua (re: "_combat" tag)` 频繁出现在源码里，正是为了提醒读者这个隐式契约。

---

### 21.3.5 SetKeepTargetFunction：当前目标该不该继续打

#### 配置方法

```237:239:scripts/components/combat.lua
function Combat:SetKeepTargetFunction(fn)
    self.keeptargetfn = fn
end
```

——一行，记下函数。**`fn(inst, target) -> bool` 返回 false 表示"放弃"**。

但注意：**只有当 `keeptargetfn ~= nil` 时，`EngageTarget` 才会启动 `OnUpdate`**：

```387:389:scripts/components/combat.lua
        if self.keeptargetfn then
            self.inst:StartUpdatingComponent(self)
        end
```

——这是个性能优化。**不需要 keeptarget 判定的怪物（比如石虾，被吸引就一直挂着）根本不开 OnUpdate**，省下每秒一次的回调。

#### OnUpdate 完整逻辑

```306:341:scripts/components/combat.lua
function Combat:OnUpdate(dt)
    if self.target == nil then
        self.inst:StopUpdatingComponent(self)
        return
    end

    if self.keeptargetfn ~= nil then
        self.keeptargettimeout = self.keeptargettimeout - dt
        if self.keeptargettimeout < 0 then
            if self.inst:IsAsleep() then
                self.inst:StopUpdatingComponent(self)
                return
            end
            self.keeptargettimeout = 1

			local drop
			if not self.target:IsValid() or self.target:IsInLimbo() then
				drop = true
			else
				local iframeskeepaggro_combat = self.target.sg and self.target.sg:HasStateTag("iframeskeepaggro") and self.inst.replica.combat or nil --V2C: intentionally using replica on server
				if iframeskeepaggro_combat then
					iframeskeepaggro_combat.temp_iframes_keep_aggro = true
				end
				drop = not (self.target.components.combat and self.keeptargetfn(self.inst, self.target) and self.target.components.combat:CanBeAttacked(self.inst))
				if iframeskeepaggro_combat then
					iframeskeepaggro_combat.temp_iframes_keep_aggro = nil
				end
			end

			if drop then
                self.inst:PushEvent("losttarget")
                self:DropTarget()
            end
        end
    end
end
```

**五段式拆解**：

1. **第一道关卡：`self.target == nil` 立刻停 OnUpdate**——常发生在 DropTarget 之后 OnUpdate 还排在队列里。
2. **`keeptargettimeout -= dt`**——这就是"每秒判一次"的实现。`OnUpdate` 实际每帧（30Hz）跑，但只有 timeout < 0 才执行真正的判定。
3. **沉睡时停止 OnUpdate**——重要！实体进入 sleep 后没必要每秒判定。OnEntityWake 会重新拉起来（见 21.3.6）。
4. **目标失效 / IsInLimbo → 立刻 drop**——immediate 判定，不需要走 keeptargetfn。
5. **三重叠加判定**——必须**同时**满足：
   - `target.components.combat ~= nil`（目标还有战斗组件）
   - `keeptargetfn(self.inst, self.target)` 返回 true
   - `target.components.combat:CanBeAttacked(self.inst)` 返回 true（21.1 + 21.4 的逻辑）
   
   任一为 false → drop。

#### keeptargettimeout 的"防抖动"特性

注意这一行：

```319:scripts/components/combat.lua
			self.keeptargettimeout = 1
```

每次判定完都重置为 **1 秒**。这意味着：

- 怪物锁定目标后，**至少 1 秒内不会脱战**（即使 KeepTargetFn 在这秒内会失败）；
- 这是为了避免"小步抖动"——比如玩家短暂跨出光圈一瞬，被一帧判定为"目标进黑暗"立刻脱战，又一帧"回光"重新锁定，让 brain 状态机疯狂切换；
- 默认 1 秒粒度是一个"心跳间隔"，给世界状态稳定的时间。

> ⚠️ **mod 实践陷阱**：你写 `keeptargetfn` 时**不要假定它每帧调用**——它最快每秒一次。所以"目标进黑暗立刻松手"是做不到的（除非用事件机制如 `onenterdark` 触发 `DropTarget`）。

#### 典型 KeepTargetFn：从简到繁

**极简**（万物书雪梅大头领）：

```91:106:mods/联机版mod/万物书/scripts/prefabs/11_tbat_animals/01_snow_plum_chieftain.lua
	local function KeepTargetFn(inst, target)
		if target:HasTag("tbat_animal_snow_plum_chieftain") then
			return (target
				and target.components.combat
				and target.components.health
				and not target.components.health:IsDead()
				and not ( get_leader(inst) ~= nil and get_leader(inst) == get_leader(target))
				and not ( get_leader(inst) == target))
		else
			return (target
				and target.components.combat
				and target.components.health
				and not target.components.health:IsDead()
				and not (get_leader(inst) == target)) and not (get_player_leader(inst) == target)
		end
	end
```

读这个函数学到三件事：

1. **target.components.combat / health 判存活**——keeptargetfn 内可以读 target.components（因为这里跑在服务端 OnUpdate 里）。
2. **特殊关系排除**——同一 leader 的同伴、leader 本人、leader 持有 eyebone 的 grand owner 都不打。
3. **同种生物间不打**——雪梅大头领不打雪梅大头领。

**官方版猪人**：

```249:253:scripts/prefabs/pigman.lua
local function NormalKeepTargetFn(inst, target)
    --give up on dead guys, or guys in the dark, or werepigs
    return inst.components.combat:CanTarget(target) and target:IsInLight()
        and not (target.sg ~= nil and target.sg:HasStateTag("transform"))
end
```

——三条件：能打（CanTarget 包含所有 ShouldAggro + IsValidTarget 检查）、目标在亮处、目标没在变身。

> **要点**：在 KeepTargetFn 中调用 `inst.components.combat:CanTarget(target)` 是个非常常见的模式——它会自动重新跑一遍 IsValidTarget + ShouldAggro，确保"目标的所有状态约束都还成立"。

#### 不写 KeepTargetFn 会怎样？

不写 = `keeptargetfn = nil` = **`EngageTarget` 不启动 OnUpdate** = **永远不会主动脱战**。

这适合两种场景：

1. **boss 之类锁定即"终身仇恨"的实体**：钟摆似的攻击模式，开战即终结，不需要"心累了换目标"；
2. **简单工具实体**：陷阱、固定炮台之类的，被攻击者一旦失效（target 进 limbo / 被销毁）由 `StartTrackingTarget` 的事件兜底就够了。

> 但**真要做这种"永不放手"怪物**，还是建议写一个最简单的 KeepTargetFn：
>
> ```lua
> local function KeepTargetFn(inst, target)
>     return inst.components.combat:CanTarget(target)
> end
> ```
>
> ——至少保证"被冻僵、变成盟友、被吃掉"这类状态变化能在 1 秒内反应。

---

### 21.3.6 进阶档：StartTrackingTarget 与四个回调

锁定目标时，`EngageTarget` 会调 `StartTrackingTarget(target)` 挂 4 个事件监听：

```347:354:scripts/components/combat.lua
function Combat:StartTrackingTarget(target)
    if target then
        self.inst:ListenForEvent("enterlimbo", self.losetargetcallback, target)
        self.inst:ListenForEvent("onremove", self.losetargetcallback, target)
		self.inst:ListenForEvent("transfercombattarget", self.transfertargetcallback, target)
		self.inst:ListenForEvent("leaderchanged", self.allycheckcallback, target)
    end
end

function Combat:StopTrackingTarget(target)
	self.inst:RemoveEventCallback("leaderchanged", self.allycheckcallback, target)
	self.inst:RemoveEventCallback("transfercombattarget", self.transfertargetcallback, target)
    self.inst:RemoveEventCallback("enterlimbo", self.losetargetcallback, target)
    self.inst:RemoveEventCallback("onremove", self.losetargetcallback, target)
end
```

注意 `ListenForEvent` 的第三个参数 `target`——这意味着监听的是**目标实体上**的事件，而不是 self 上的。所以"目标被销毁，我立刻知道"。

#### 四个回调详解

回到 21.2.2 看构造函数：

```84:99:scripts/components/combat.lua
	self.losetargetcallback = function() self:DropTarget() end
	self.transfertargetcallback = function(target, newtarget)
		if newtarget ~= nil and self:CanTarget(newtarget) then
			self:SetTarget(newtarget)
			if self.target == target and self.target ~= newtarget then
				self:DropTarget()
			end
		else
			self:DropTarget()
		end
	end
	self.allycheckcallback = function(target)
		if self:CanBeAlly(target) then
			self:DropTarget()
		end
	end
```

| 回调 | 触发事件 | 行为 |
|---|---|---|
| `losetargetcallback` | `enterlimbo` / `onremove` | 直接 `DropTarget()` |
| `transfertargetcallback` | `transfercombattarget` | 检查能否打 newtarget → `SetTarget(newtarget)` 或 `DropTarget` |
| `allycheckcallback` | `leaderchanged` | 检查 target 是否变成盟友 → `DropTarget` |

**为什么要在构造函数里预先创建闭包**？因为这三个闭包都"捕获了 self"——你需要每个 Combat 实例有自己的回调，而不是用一个全局函数。同时，预先建好后，`StartTracking`/`StopTracking` 用同一个闭包引用，才能正确 RemoveEventCallback（事件回调用 lambda 时**必须用同一个引用**才能解绑）。

#### `transfercombattarget` 事件：仇恨转移

——这是个**boss 专属**的事件，由攻击者主动推送给当前目标。常见场景：

- **某 boss 死前召唤分身**：死之前 push `transfercombattarget` 给所有锁定它的怪物，新目标设为它的分身；
- **传送召唤**：本体跑了，但留下一个"假身"接打。

源码搜一下：

```mods\联机版mod\...\蜈蚣\centipede.lua 等
inst:PushEvent("transfercombattarget", { newtarget = some_body })
```

——`{ newtarget = ... }` 是数据约定。回调里读 `newtarget`，能打就 `SetTarget(newtarget)`，不能就 `DropTarget`。

> **设计建议**：你做"二阶段 boss"的时候，可以利用这个事件优雅地让所有仇恨该 boss 的怪物都转向新阶段的本体。

#### `leaderchanged` 事件：盟友身份切换

——当目标的 leader 变化时，目标会 push 这个事件。`allycheckcallback` 收到后调用 `CanBeAlly(target)`（实际走 `replica.combat:CanBeAlly`，见 `combat_replica.lua:322-372`）判断目标是否变成自己阵营。如果是 → DropTarget。

经典场景：玩家用"友善之笛"驯化敌对生物，让它的 leader 切到自己 → 所有正在打它的玩家阵营生物自动停手。

---

### 21.3.7 进阶档：SuggestTarget 与 cansuggesttargetfn

`SuggestTarget` 是"温和"的目标设置——**仅当我现在没目标时才接受**。

```229:235:scripts/components/combat.lua
function Combat:SuggestTarget(target)
    if self.target == nil and target ~= nil and (self.cansuggesttargetfn == nil or self.cansuggesttargetfn(self.inst, target)) then
        --print("Combat:SuggestTarget", self.inst, target)
        self:SetTarget(target)
        return true
    end
end
```

**三层守卫**：

1. `self.target == nil`（已经在打别人就拒绝）
2. `target ~= nil`
3. `cansuggesttargetfn` 通过（可选）

**注意它不返回 false——只返回 true 或 nil**。所以调用方习惯写 `if combat:SuggestTarget(t) then ... end`。

#### `SetCanSuggestTargetFn`

```225:227:scripts/components/combat.lua
function Combat:SetCanSuggestTargetFn(fn)
    self.cansuggesttargetfn = fn
end
```

——设置过滤函数。`fn(inst, target) -> bool`。

#### 实战：变异秃鹰的"分散仇恨"

```365:367:scripts/prefabs/buzzard.lua
local function Mutated_CanSuggestTargetFn(inst, target)
    return GetTableSize(BUZZARD_SHARED_TARGETS[target]) < TUNING.MUTATEDBUZZARD_MAX_TARGET_COUNT
end
```

——某个 target 已经被超过 N 只秃鹰盯上了，拒绝再分配秃鹰。**避免所有秃鹰扎堆打同一个人**，保留生态平衡。

这是 `cansuggesttargetfn` 最经典的使用场景：**"基于全局状态的目标分配"**。`shouldaggrofn`（21.2.8）是"个体决定要不要打"，`cansuggesttargetfn` 是"被推荐目标时再过一遍审"。

#### 谁会调用 SuggestTarget？

```scripts\prefabs\evergreens.lua:413
v.components.combat:SuggestTarget(chopper)

scripts\prefabs\evergreens.lua:445
leif.components.combat:SuggestTarget(target.chopper)

scripts\prefabs\warg.lua:346
inst.components.combat:SuggestTarget(player)

scripts\prefabs\hats.lua:805
k.components.combat:SuggestTarget(owner)

scripts\stategraphs\SGtornado.lua:84
v.components.combat:SuggestTarget(inst.WINDSTAFF_CASTER)

scripts\prefabs\staff.lua:114/178
target.components.combat:SuggestTarget(attacker)

scripts\prefabs\tree_rocks.lua:611
v.components.combat:SuggestTarget(chopper)
```

——总结四类：

1. **被砍的树**：唤醒附近的 Leif → `SuggestTarget(chopper)`，让森林精灵盯上砍树的人；
2. **环境效果**：龙卷风、风暴法杖召唤的范围效果 → 对所有受影响生物 SuggestTarget；
3. **法杖**：传送法杖、骨杖等让目标自动反击施法者；
4. **宠物帽**：让宠物自动反击主人的敌人。

注意它们都是"事件触发型"——某个动作发生 → SuggestTarget。**不是"我主动找目标"，而是"别人告诉我有目标"**。这是 SuggestTarget 与 SetTarget 的本质区别：

| | `SetTarget(t)` | `SuggestTarget(t)` |
|---|---|---|
| 当前已有目标 | **强制覆盖** | 拒绝 |
| 过滤层级 | IsValidTarget + ShouldAggro + hiding | 同上 + cansuggesttargetfn |
| 典型调用方 | OnAttacked、RetargetFn、Brain 节点 | 环境效果、法杖、其他生物 ShareTarget |
| 返回值 | 无（隐式 boolean: target ~= self.target） | true 表示采纳，nil 表示拒绝 |

---

### 21.3.8 进阶档：ShareTarget 仇恨分享算法

群体怪物（蜘蛛、猪、蜜蜂）有个共同需求——"被打就喊群"。`ShareTarget` 是这种群体仇恨的标准 API。

```184:215:scripts/components/combat.lua
local DEFAULT_SHARE_TARGET_MUST_TAGS = { "_combat" }
function Combat:ShareTarget(target, range, fn, maxnum, musttags)
    --NOTE: true param ignores my own forbidden tags when sharing someone else's aggro
    if not self:ShouldAggro(target, true) then
        return
    end
    if maxnum <= 0 then
        return
    end

    --print("Combat:ShareTarget", self.inst, target)

    local x, y, z = self.inst.Transform:GetWorldPosition()
    local ents = TheSim:FindEntities(x, y, z, SpringCombatMod(range), musttags or DEFAULT_SHARE_TARGET_MUST_TAGS)

    local num_helpers = 0
    for i, v in ipairs(ents) do
        if v ~= self.inst
            and not (v.components.health ~= nil and
                    v.components.health:IsDead())
            and (fn == nil or fn(v, self.inst))
            and v.components.combat:SuggestTarget(target) then

            --print("    share with", v)
            num_helpers = num_helpers + 1

            if num_helpers >= maxnum then
                return
            end
        end
    end
end
```

#### 参数全表

| 参数 | 类型 | 是否可选 | 默认值 | 作用 |
|---|---|---|---|---|
| `target` | Entity | 必填 | — | 要分享的目标 |
| `range` | number | 必填 | — | 搜索半径（**会自动套 SpringCombatMod 春季放大**） |
| `fn` | function `(v, inst) -> bool` | 可选 | nil | 进一步过滤"谁能接收分享" |
| `maxnum` | number | 必填 | — | 最多通知多少个同伴 |
| `musttags` | table | 可选 | `{"_combat"}` | 搜索的必须标签 |

#### 流程拆解

1. **守卫 1：`self:ShouldAggro(target, true)`**——自己先看下能不能仇恨这个目标。**第二个参数 `true`** 表示"忽略自己的 forbiddenaggrotags"（因为是给别人分享，别人有自己的 forbidden 列表）。如果**自己**就不该恨这个 target，那也别告诉别人。
2. **守卫 2：`maxnum <= 0`** 直接返回。
3. **`SpringCombatMod(range)`** 自动放大搜索半径——春季生物更暴躁（饥荒春季战斗增益）。
4. **遍历 + 三层过滤**：
   - `v ~= self.inst`（别广播给自己）
   - 不是死亡
   - `fn(v, self.inst)` 通过（自定义"谁能收"的过滤）
   - `v.components.combat:SuggestTarget(target)` 返回 true（接收方实际接受了）
5. **计数器**——接受方数到 maxnum 立刻停止。

#### 实战：蜘蛛 OnAttacked 分享

```306:330:scripts/prefabs/spider.lua
local function OnAttacked(inst, data)
    if inst.no_targeting then
        return
    end

    inst.defensive = false
    inst.components.combat:SetTarget(data.attacker)

    if inst:HasTag("shadowthrall_parasite_hosted") then
        inst.components.combat:ShareTarget(data.attacker, 30, IsHost, 10)
    else
        inst.components.combat:ShareTarget(data.attacker, 30, function(dude)
                local should_share = dude:HasTag("spider")
                    and not dude.components.health:IsDead()
                    and dude.components.follower ~= nil
                    and dude.components.follower:GetLeader() == inst.components.follower:GetLeader()

                if should_share and dude.defensive and not dude.no_targeting then
                    dude.defensive = false
                end

                return should_share
            end, 10)
    end
end
```

**精彩点**：

1. **自己先 `SetTarget`**——主角先反击。
2. **`ShareTarget(attacker, 30, ...)`**——半径 30、最多 10 个同伴。
3. **fn 不只是过滤，还顺带做"副作用"**——`dude.defensive = false` 让防御型蜘蛛也变进攻型！这是 fn 参数的隐藏用法：**遍历过程中可以修改 dude 的状态**。
4. **同 leader 限制**——只通知"和我同一个母蜘蛛巢的蜘蛛"，避免敌对蜘蛛阵营互相帮助。

#### 万物书雪梅大头领的 ShareTarget

```67:78:mods/联机版mod/万物书/scripts/prefabs/11_tbat_animals/01_snow_plum_chieftain.lua
	local function OnAttacked(inst, data)
		local attacker = data.attacker
		inst:ClearBufferedAction()

		if inst.components.combat and not inst.components.combat.target then
		--	inst.sg:GoToState("hiss")
		end
		if inst.components.combat then 
			inst.components.combat:SetTarget(data.attacker) 
			inst.components.combat:ShareTarget(attacker, SHARE_TARGET_DIST, function(dude) return dude:HasTag("tbat_animal_snow_plum_chieftain") end, MAX_TARGET_SHARES)
		end		
	end
```

——同样"先自己反击 → 再广播"的标准模式，过滤简单只看 prefab 标签。**最少代码的"仇恨广播"实现**，30 行新手都能套着写。

---

### 21.3.9 进阶档：休眠/唤醒对目标系统的影响

```284:304:scripts/components/combat.lua
function Combat:OnEntitySleep()
    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end
end

function Combat:OnEntityWake()
    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end

    if self.retargetperiod ~= nil then
        self.retargettask = self.inst:DoPeriodicTask(self.retargetperiod, dotryretarget, self.retargetperiod*math.random(), self)
    end

    if self.target ~= nil and self.keeptargetfn ~= nil then
        self.inst:StartUpdatingComponent(self)
    end
end
```

#### 休眠时

**只取消 `retargettask`，不动 `self.target`**！这意味着：

- 怪物休眠了，但**当前锁定的目标仍在**；
- 唤醒后能继续打（StartUpdatingComponent 重新拉起 OnUpdate）。

但是要注意——`OnUpdate` 在休眠时也被停掉（见 `OnUpdate` 内的 `if self.inst:IsAsleep() then StopUpdatingComponent`）。

#### 唤醒时

**三件事**：

1. **重启 retargettask**——只要 `retargetperiod ~= nil`（即配置过 SetRetargetFunction）；
2. **初始延迟仍是 `retargetperiod * math.random()`**——让多只同时唤醒的怪物错峰搜目标；
3. **如果有目标 + 有 keeptargetfn → 重启 OnUpdate**。

#### 为什么这么设计？

饥荒的"休眠机制"是**性能优化**：远离玩家的实体冻结所有更新。如果怪物休眠时还能跑 RetargetFn，会让"地图角落的猪 30 秒一次搜目标"白白消耗 CPU。

但**当前目标不动**——保证玩家"远走重新回来时怪物还在追他"。这种"假休眠"是饥荒的核心一致性保证。

> **mod 实践陷阱**：你写一个"怪物休眠时也要 hover 搜目标"的逻辑（比如自动巡逻），需要**重写 `OnEntitySleep`/`OnEntityWake`**（用 AddComponentPostInit 包装），否则你的定时器会被原生逻辑无情取消。

---

### 21.3.10 老手必读：keeptargettimeout 与 iframeskeepaggro

这两个细节是 boss 战手感的根基。

#### `keeptargettimeout` 的可控性

回顾代码：

```312:319:scripts/components/combat.lua
    if self.keeptargetfn ~= nil then
        self.keeptargettimeout = self.keeptargettimeout - dt
        if self.keeptargettimeout < 0 then
            ...
			self.keeptargettimeout = 1
```

每次判定完重置为 **1 秒**。但**你可以在外部强制设置 `self.keeptargettimeout = 0.1`** 让下次判定提前——这是"立即重新检查目标合法性"的方式。

```lua
-- 例：玩家被强力 buff 隐身，要让怪物立刻丢目标
function ApplyInvisibleBuff(target)
    for _, inst in pairs(TheSim:FindEntities(...)) do
        if inst.components.combat and inst.components.combat.target == target then
            inst.components.combat.keeptargettimeout = 0  -- 下一帧就判定
        end
    end
end
```

虽然你也可以直接调 `combat:DropTarget()`，但用 `keeptargettimeout` 更优雅——**让 KeepTargetFn 自己决定"还该不该打"**，而不是粗暴脱战。

#### `iframeskeepaggro`：无敌帧的特殊机制

```324:332:scripts/components/combat.lua
			else
				local iframeskeepaggro_combat = self.target.sg and self.target.sg:HasStateTag("iframeskeepaggro") and self.inst.replica.combat or nil --V2C: intentionally using replica on server
				if iframeskeepaggro_combat then
					iframeskeepaggro_combat.temp_iframes_keep_aggro = true
				end
				drop = not (self.target.components.combat and self.keeptargetfn(self.inst, self.target) and self.target.components.combat:CanBeAttacked(self.inst))
				if iframeskeepaggro_combat then
					iframeskeepaggro_combat.temp_iframes_keep_aggro = nil
				end
			end
```

**这段在做什么**？

某些 boss / 玩家进入"无敌帧"状态时（比如 wanda 时间穿梭、wickerbottom 翻书的 invulnerable 几帧、wagstaff 的护盾），它们的 `CanBeAttacked` 会返回 false——按正常 KeepTarget 逻辑，所有锁定它们的怪物会"丢目标"。

但**这违背设计预期**：玩家用 1 帧无敌帧躲攻击，不该让 boss 直接停下来转向别人。

于是引入了 `iframeskeepaggro` 状态标签：**目标的 stategraph 有这个 state tag 时，临时把 `temp_iframes_keep_aggro = true` 写到攻击者的 replica 上**——`CanBeAttacked` 检测到这个标记，会**绕过 invisible 检查**：

```408:416:scripts/components/combat_replica.lua
function Combat:CanBeAttacked(attacker)
	if self.inst:HasAnyTag("playerghost", "flight") or
		(	not self.temp_iframes_keep_aggro and
			self.inst:HasAnyTag("noattack", "invisible")
		)
	then
        --Can't be attacked by anyone
        return false
```

——`not self.temp_iframes_keep_aggro` 是关键，启用时即使有 `noattack`/`invisible` 标签也允许"维持锁定"（这只是给 KeepTarget 判定用，不允许实际伤害）。

#### 注释里的 `V2C: intentionally using replica on server`

第 325 行特意走了 `self.inst.replica.combat`——服务端走 replica 看似奇怪，实际是因为 **`CanBeAttacked` 的逻辑在 replica 上**（服务端共用 replica，见 21.1.4）。这种"服务端故意走 replica"的注释在饥荒源码里有几十处，每次都是因为该方法的实现只在 replica 上有一份。

#### mod 实践：让自己的 buff 也享受 iframeskeepaggro

如果你写一个"短暂无敌"buff，希望怪物不要因此脱战，**给玩家的 stategraph state 加 `iframeskeepaggro` 标签**：

```lua
-- 在某个新建的"小翻滚"state 里
addstategraph.lua

State {
    name = "dodge_roll",
    tags = { "busy", "invisible", "iframeskeepaggro" }, -- 关键
    ...
}
```

——这样玩家做翻滚时**有 invisible 让伤害打不到**，但 **keeptargetfn 仍把目标钉死**——boss 翻滚完玩家立刻还能被打。

---

### 21.3.11 mod 开发实战姿势（新手 / 进阶 / 老手三档）

#### 新手档：30 行写一个会找目标的小怪

```lua
local POG_TARGET_DIST = 20

local RETARGET_MUST_TAGS = { "_combat" }
local RETARGET_CANT_TAGS = { "INLIMBO", "notarget", "playerghost", "noattack" }

local function RetargetFn(inst)
    return FindEntity(
        inst,
        POG_TARGET_DIST,
        function(guy)
            return inst.components.combat:CanTarget(guy)
                and guy.components.health and not guy.components.health:IsDead()
        end,
        RETARGET_MUST_TAGS,
        RETARGET_CANT_TAGS,
        { "monster", "player" }  -- 要打怪和玩家
    )
end

local function KeepTargetFn(inst, target)
    return inst.components.combat:CanTarget(target)
        and inst:IsNear(target, POG_TARGET_DIST + 5) -- 出离 25 格脱战
end

local function OnAttacked(inst, data)
    if data.attacker and inst.components.combat then
        inst.components.combat:SetTarget(data.attacker)
        inst.components.combat:ShareTarget(data.attacker, 20, function(dude)
            return dude.prefab == inst.prefab and not dude.components.health:IsDead()
        end, 5)
    end
end

local function fn()
    local inst = CreateEntity()
    ...
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("combat")
    inst.components.combat:SetDefaultDamage(20)
    inst.components.combat:SetAttackPeriod(2)
    inst.components.combat:SetRange(3)
    inst.components.combat:SetRetargetFunction(3, RetargetFn)
    inst.components.combat:SetKeepTargetFunction(KeepTargetFn)
    inst:ListenForEvent("attacked", OnAttacked)
    ...
end
```

——这是个**完整能跑**的怪物战斗骨架。包含：

1. **每 3 秒主动搜目标**（半径 20）；
2. **被攻击时立刻反击 + 喊群**（半径 20、最多 5 个同伴）；
3. **目标超过 25 格自动脱战**（KeepTargetFn 距离检查）；
4. **目标被销毁/进 limbo/变盟友**——自动 DropTarget（StartTrackingTarget 内置）；
5. **目标死亡** → ChaseAndAttack brain 节点会自动处理。

新手不要往里加复杂逻辑——先用骨架跑通"接战 / 脱战 / 喊群"，再迭代。

#### 进阶档：动态切换 RetargetFn / KeepTargetFn

像猪人那样"白天和睦、夜晚警卫、被守卫后野化"——通过 `SetRetargetFunction` 直接覆盖即可：

```lua
local function SetGuardMode(inst)
    inst.components.combat:SetRetargetFunction(1, GuardRetargetFn) -- 警戒期更频繁搜
    inst.components.combat:SetKeepTargetFunction(GuardKeepTargetFn)
    inst.components.combat:SetNoAggroTags({ "guard", "INLIMBO" }) -- 不打其他守卫
end

local function SetPeacefulMode(inst)
    inst.components.combat:SetRetargetFunction(nil, nil) -- 完全停止主动搜目标
    inst.components.combat:SetKeepTargetFunction(nil)    -- 永不丢目标（已锁的会一直打）
    inst.components.combat:SetTarget(nil)                -- 清掉当前目标
end
```

注意**两件事**：

1. `SetRetargetFunction(nil, nil)` 内部会取消 retargettask（见 21.3.4），切换非常干净；
2. `SetKeepTargetFunction(nil)` **不会** 停止现有的 OnUpdate——下一帧 OnUpdate 会发现 `keeptargetfn == nil`，但**仍然不 drop**（因为没有判定）。如果你确实要"切到和平模式时立刻清目标"，记得加 `SetTarget(nil)`。

#### 老手档：自定义 ChaseAndAttack 行为

`ChaseAndAttack` 是 brain 库的标准追击节点，但它有一个 `findnewtargetfn` 参数——在当前目标失效时**调用一次找新目标**，**不依赖 `retargettask`**：

```110:scripts/behaviours/chaseandattack.lua
self.findnewtargetfn(self.inst)  -- 见 chaseandattack.lua:42
```

经典用例：

```lua
-- 在 brain.lua
local function FindNewTargetForChase(inst)
    -- 不要走 RetargetFn，因为 brain 节点是更"积极"的搜索
    return FindEntity(inst, 15, function(guy)
        return inst.components.combat:CanTarget(guy)
            and not inst.components.combat:IsRecentTarget(guy)  -- 不重复打刚打过的
    end, { "_combat" }, { "INLIMBO" })
end

local node = ChaseAndAttack(self.inst, 10, 25, nil, FindNewTargetForChase)
```

——这种"双层目标系统"很常见：

- **`SetRetargetFunction`** 负责"和平时期 / 散步时" 5-10 秒一次的主动找目标；
- **`ChaseAndAttack.findnewtargetfn`** 负责"战斗中目标突然没了"的紧急找新目标。

两者用不同的搜索条件——前者注重"是否值得仇恨"，后者注重"是否能立刻打到"。

#### 老手档：拦截 newcombattarget / droppedtarget 实现"目标日志"

某些 mod 需要追踪怪物的目标变化历史。可以挂事件：

```lua
inst:ListenForEvent("newcombattarget", function(inst, data)
    inst.target_history = inst.target_history or {}
    table.insert(inst.target_history, {
        time = GetTime(),
        target = data.target,
        oldtarget = data.oldtarget,
    })
end)

inst:ListenForEvent("droppedtarget", function(inst, data)
    print(inst, "dropped", data.target)
end)
```

——`newcombattarget` / `droppedtarget` / `losttarget` / `giveuptarget` 这 4 个事件构成完整的"目标生命周期"日志。结合 `lastwasattackedbytargettime` 字段可以做出"怪物对玩家的注意力数据图"，是后期平衡性测试的好工具。

---

### 21.3.12 五个常见坑

#### 坑 1：直接给 self.target 赋值绕过验证

```lua
inst.components.combat.target = some_target  -- ❌ 跳过了所有验证
```

虽然 props setter 钩子 `ontarget` 会触发同步（21.1.8），但你**跳过了**：

- `IsValidTarget` / `ShouldAggro` / `hiding` 守卫
- `DropTarget` 旧目标的事件监听卸载
- `EngageTarget` 的 `StartTrackingTarget` + `OnUpdate` 启动
- `RemoveFollower` 跟随者反叛逻辑

结果：**旧目标的 enterlimbo 监听还挂着**——下次旧目标进 limbo，你**已经换的新目标会被错误 drop**！

正确：永远走 `combat:SetTarget(target)`。

#### 坑 2：忘记 `_combat` 标签导致 RetargetFn 慢得离谱

```lua
local function BadRetargetFn(inst)
    return FindEntity(inst, 20, function(guy)
        return inst.components.combat:CanTarget(guy)
            and guy.components.combat ~= nil  -- ← 慢！
    end)  -- ← 没有 musttags！
end
```

`FindEntity` 没有 musttags 时会遍历**搜索范围内的所有实体**——包括地上的石头、树、草、便便。20 格内可能有几百个实体，每个都跑一遍 `guy.components.combat ~= nil` 检查——CPU 杀手。

正确：

```lua
local RETARGET_MUST_TAGS = { "_combat" }
local function GoodRetargetFn(inst)
    return FindEntity(inst, 20, function(guy)
        return inst.components.combat:CanTarget(guy)
    end, RETARGET_MUST_TAGS)
end
```

`_combat` 在 C++ 层做空间索引，几乎 O(1)。

#### 坑 3：KeepTargetFn 写得太苛刻导致瞬间脱战

```lua
local function BadKeepTargetFn(inst, target)
    -- 目标在 attackrange 内才继续打
    return inst:GetDistanceSqToInst(target) < inst.components.combat:CalcAttackRangeSq(target)
end
```

——这种写法的怪物**追不上**任何会跑的目标。玩家一跑出攻击距离，下一秒 OnUpdate 判定 keeptargetfn = false，立刻 drop。

正确的"距离判定"要用**比攻击距离更大的"放弃距离"**：

```lua
local function GoodKeepTargetFn(inst, target)
    return inst.components.combat:CanTarget(target)
        and inst:GetDistanceSqToInst(target) < (TUNING.MY_GIVEUP_DIST ^ 2)
end
```

或者更优雅地用 `inst:IsNear(target, dist)`：

```lua
local function GoodKeepTargetFn(inst, target)
    return inst.components.combat:CanTarget(target)
        and inst:IsNear(target, TUNING.MY_GIVEUP_DIST)
end
```

#### 坑 4：mod 给玩家加 buff 时忘记目标过滤

```lua
-- 玩家用了"和平 buff"——但身边有怪物锁定他
AddPrefabPostInit("wilson", function(inst)
    inst:AddTag("notarget")  -- ❌ 仅打新标签
end)
```

这只阻止**新的目标设置**（`ShouldAggro` 中 `notarget` 触发拒绝），但**已经锁定的怪物**呢？

它们的 `OnUpdate` 会等 keeptargettimeout 到，跑 `keeptargetfn(inst, target)` ——大多数生物的 keeptargetfn 调用 `CanTarget`，而 `CanTarget` 会检查 target 的 `notarget` 标签。

但**等 keeptargettimeout 是最多 1 秒**！这 1 秒内怪物还在打你。

更激进的做法：

```lua
AddPrefabPostInit("wilson", function(inst)
    if not TheWorld.ismastersim then return end
    inst:AddTag("notarget")
    -- 强制所有锁定我的怪物立刻丢目标
    inst:PushEvent("transfercombattarget", { newtarget = nil })
end)
```

——`transfercombattarget` 配合 `newtarget = nil` 触发 `transfertargetcallback` 内的 `DropTarget` 分支。

#### 坑 5：mod 在 OnEntitySleep 里清掉 combat.target

```lua
-- 你 mod 的某怪物
inst:ListenForEvent("entitysleep", function(inst)
    inst.components.combat:SetTarget(nil)  -- ❌ 别这么干
end)
```

——这违背了 21.3.9 讲的设计意图。怪物休眠时**应该保留 target**，让玩家"回头还能继续打这只怪"。

如果你的 mod 真的需要"远离玩家则忘记仇恨"，请改用 `keeptargetfn`：

```lua
local function KeepTargetFn(inst, target)
    return inst:IsNear(target, GIVEUP_DIST)
end
```

——让饥荒的"距离判定"自然处理。

---

### 21.3.13 调试技巧

**1. 看实体当前目标和距离**

```lua
local mob = c_find("pigman")  -- 控制台找最近的猪
print(mob.components.combat:GetDebugString())
```

`GetDebugString` 输出 target + damage + dist/range + cooldown + canattack + canbeattacked，一行看尽（见 `combat.lua:491-508`）。

**2. 监听目标变化事件**

```lua
local mob = c_find("pigman")
mob:ListenForEvent("newcombattarget", function(inst, data)
    print(inst, "锁定", data.target, "旧目标", data.oldtarget)
end)
mob:ListenForEvent("droppedtarget", function(inst, data)
    print(inst, "脱战", data.target)
end)
mob:ListenForEvent("losttarget", function(inst)
    print(inst, "因 KeepTarget 失败脱战")
end)
mob:ListenForEvent("giveuptarget", function(inst, data)
    print(inst, "主动放弃", data.target)
end)
```

——4 个事件覆盖了所有目标变化。如果你的怪物不主动脱战、又或者一直在脱战，立刻就能看出哪种触发。

**3. 强制触发一次 retarget**

```lua
mob.components.combat:TryRetarget()
```

——不用等 retargetperiod，立刻调一次 targetfn。

**4. 看 retargettask 的状态**

```lua
if mob.components.combat.retargettask then
    print("retarget 周期：", mob.components.combat.retargetperiod)
    print("正在跑：", mob.components.combat.retargettask)
else
    print("没有 retarget（可能在休眠 / 没设 SetRetargetFunction / 或者刚被清掉）")
end
```

**5. 临时切换搜目标周期**

```lua
mob.components.combat:SetRetargetFunction(0.5, mob.components.combat.targetfn)
```

——把周期从 3 秒压到 0.5 秒，方便观察 Retarget 触发频率。注意：复用了原来的 `targetfn`，只是改了周期。

**6. 强制让所有同类怪物锁定某玩家**

```lua
local target = ConsoleCommandPlayer()
for _, ent in ipairs(TheSim:FindEntities(target.Transform:GetWorldPosition(), 30, {"pigman"})) do
    if ent.components.combat then
        ent.components.combat:SetTarget(target)
    end
end
```

——压力测试群体仇恨用。

---

### 本节涉及的标签

| 标签 | 含义 | 作用 |
|---|---|---|
| `_combat` | "本实体有 combat 组件" | RetargetFn / ShareTarget / FindEntity 的标准 musttag，让搜索 O(1) |
| `notarget` | "不可被选为目标" | `TryRetarget` 跳过；`CanTarget` 拒绝；mod buff/状态切换时常用 |
| `INLIMBO` | "实体在 limbo（容器中、隐藏）" | enterlimbo 事件触发自动 DropTarget；RetargetFn 标准 excludetag |
| `playerghost` | "玩家鬼魂" | `CanBeAttacked` 直接拒绝，RetargetFn 通常排除 |
| `noattack` | "不能被攻击" | `CanBeAttacked` 拒绝（除非 iframeskeepaggro） |
| `invisible` | "隐身" | `CanTarget` 拒绝（除非 iframeskeepaggro） |
| `flight` | "飞行" | `CanBeAttacked` 直接拒绝 |
| `hiding` | "藏匿状态" | `SetTarget` 守卫——藏匿的玩家不可被设为目标 |
| `iframeskeepaggro` | "无敌帧但保持仇恨"（state tag） | `OnUpdate` 临时设 `temp_iframes_keep_aggro = true` 绕过 invisible 检查 |
| `structure` | "建筑" | `TryRetarget` 中"生物优先于结构"的判定标签 |
| `stealth` | "潜行" | `ShouldAggro` 直接拒绝 |
| `hostile` | "敌对" | `ShouldAggro` 影响"最小血量"判定 |
| `alwayshostile` | "始终敌对" | `CanBeAlly` 拒绝（永远不是盟友） |
| `companion` / `domesticated` / `saltlicker_salted` | 玩家阵营标签 | `CanBeAlly` 用以确定盟友身份 |
| `crazy` | "疯狂"状态 | `CanBeAttacked` 可绕过暗影生物豁免 |
| `shadowcreature` / `nightmarecreature` | 暗影 / 噩梦生物 | `CanBeAttacked` 中非疯狂攻击者无法打 |
| `wonkey` / `pirate` / `monster` / `merm` / `abigail` | 物种 / 阵营标签 | RetargetFn 的 mustoneoftags / canttags 常见组合 |
| `transform` | "正在变身"（state tag） | 多个 KeepTargetFn 用以"变身中暂时停手" |

### 本节涉及的事件

| 事件名 | 推送对象 | 推送时机 | 数据键值 |
|---|---|---|---|
| `newcombattarget` | 攻击者本身（`self.inst`） | `EngageTarget` 中切换/锁定新目标后 | `target`(Entity) - 新目标 / `oldtarget`(Entity\|nil) - 旧目标 |
| `droppedtarget` | 攻击者本身 | `DropTarget(false)`——真正脱战（**非切换目标**）时 | `target`(Entity) - 被放弃的目标 |
| `losttarget` | 攻击者本身 | `OnUpdate` 中 keeptargetfn / CanBeAttacked 判定失败导致脱战 | 无（数据为 nil） |
| `giveuptarget` | 攻击者本身 | `Combat:GiveUp` 主动放弃（带台词） | `target`(Entity) - 被放弃的目标 |
| `enterlimbo` | 目标实体 | 目标进入 limbo（被放入容器、被吞食等） | 无 —— Combat 的 `losetargetcallback` 触发 DropTarget |
| `onremove` | 目标实体 | 目标被销毁 | 无 —— Combat 的 `losetargetcallback` 触发 DropTarget |
| `transfercombattarget` | 目标实体 | "仇恨转移"（boss 二阶段、传送替身等） | `newtarget`(Entity\|nil) - 转给的新目标，nil 表示让所有仇恨者脱战 |
| `leaderchanged` | 目标实体 | 目标的 follower leader 变化 | （随推送方）—— Combat 的 `allycheckcallback` 检查目标是否变盟友 |
| `attacked`（服务端版） | 被攻击实体 | `Combat:GetAttacked` 未 blocked 时（详 21.8） | `attacker` / `damage` / `weapon` / `stimuli` / `spdamage` / `redirected` / `noimpactsound` / `original_damage` / `damageresolved` —— 触发 OnAttacked 是设新目标的标准入口 |
| `onattackother` | 攻击者本身 | DoAttack 中即将攻击时 | `target` / `weapon` / `projectile` / `stimuli` —— ChaseAndAttack 节点用以重置 max_chase_time |
| `onmissother` | 攻击者本身 | DoAttack 中 `CanHitTarget` 失败 | `target` / `weapon` —— 同上，重置 max_chase_time |
| `knockback` | 被击退实体 | knockback 组件 / 弹反逻辑 | （随推送方）—— Combat 构造时挂监听 `CommonHandlers.ResetHitRecoveryDelay` |
| `doattack` | 攻击者本身 | `TryAttack` / `ForceAttack` 触发攻击 stategraph | `target`(Entity\|nil) - 攻击目标 |

---



## 21.4 仇恨与敌对判定

### 本节导读

21.3 把目标系统的**流程**讲清楚了——"找、设、跟、保、弃"。但流程的每个环节都需要一个核心问题的答案：**这个东西，我现在能不能打？**

```lua
combat:SetTarget(some_player)         -- 这个玩家能被设为目标吗？
combat:SuggestTarget(beefalo)         -- 这只野牛允许仇恨吗？
inst:DoAttack(structure_target)       -- 这个建筑能被普通攻击吗？
```

每一个这样的"能不能"，都对应一组**判定函数**：

| 问题 | 判定函数 | 主要在 |
|---|---|---|
| "目标存在/可见/不在保护期吗？" | `IsValidTarget` | `combat_replica.lua` |
| "目标符合所有战斗规则吗？" | `CanTarget` | `combat_replica.lua` |
| "我作为攻击者，主观上想恨它吗？" | `ShouldAggro` | `combat.lua` |
| "目标能不能被任何人攻击？" | `CanBeAttacked` | `combat_replica.lua` |
| "目标是不是我的盟友？" | `CanBeAlly` / `IsAlly` | `combat_replica.lua` |
| "暗影生物对玩家是否敌对？" | `HostileToPlayerTest` | prefab 文件 |

这些函数构成了饥荒战斗系统的"宪法"——所有目标变化的允许/拒绝都在这里盖章。

读完本节，你将能回答：

- "`IsValidTarget` 和 `CanTarget` 有什么区别？为什么有时候只查 IsValidTarget？"
- "PVP 关闭时玩家为什么还能用'玩具锤子'互打？"
- "为啥我用 mod 加了一个怪物，它居然会主动攻击玩家的猪兵？"
- "暗影生物有时候明明站我面前，我点击却'打不到'，怎么回事？"
- "`shadowsubmissive` 状态下的暗影生物到底打不打玩家？"
- "PVP 关闭时跟随玩家的非敌对生物（猪、猫狸）会不会被路过的玩家误伤？"
- "我做了个无敌 buff，加哪个标签效果最好？"

> **新手**先看 21.4.1-21.4.4——理解六大判定函数及它们的调用顺序、记住"客户端用 replica、服务端用 components"的判定差异；**进阶读者**继续看 21.4.5-21.4.8，深入 PVP 规则、玩家阵营、暗影生物特殊判定、`shadowsubmissive` 机制；**老手**跳到 21.4.9-21.4.13，掌握 `HostileToPlayerTest` 钩子机制、与 brain / PlayerController 的协作、mod 扩展正确姿势。

---

### 21.4.1 一张图看清判定函数的调用层级

```
┌──────────────────────────────────────────────────────────────────────┐
│                         判定函数依赖关系                                │
│                                                                       │
│   ┌────────────────────────────────────────────────────────────────┐ │
│   │  IsValidTarget(target)            最底层——基础校验               │ │
│   │  - target != nil && != self                                     │ │
│   │  - target.entity:IsValid() && :IsVisible()                      │ │
│   │  - 灭火/点火豁免                                                │ │
│   │  - target 有 replica.combat                                     │ │
│   │  - target 不死亡、不在 spawnprotection、不是不可见暗影/鬼魂       │ │
│   │  - PVP 关闭时玩家不打玩家（除非 propweapon）                     │ │
│   │  - target.y ≤ self._attackrange:value()                         │ │
│   └────────────────────────────────────────────────────────────────┘ │
│                              ↑                                        │
│                              │ 被以下两层调用                          │
│                              │                                        │
│   ┌────────────────────────┐ │ ┌────────────────────────────────────┐│
│   │  CanTarget(target)     │ │ │  CanAttack(target) / CanHitTarget  ││
│   │  - IsValidTarget       │ │ │  - IsValidTarget                   ││
│   │  - 我没在 panic         │ │ │  - canattack && 不在冷却            ││
│   │  - 目标不带特殊标签：    │ │ │  - 不在 busy 状态                   ││
│   │    INLIMBO/notarget/   │ │ │  - 距离 ≤ attack/hitrange           ││
│   │    debugnoattack/      │ │ └────────────────────────────────────┘│
│   │    invisible           │ │              ↑                        │
│   │  - target.CanBeAttacked│ │              │ 攻击瞬间用              │
│   │  - 平静坐骑骑乘豁免     │ │                                       │
│   └────────────────────────┘ │                                       │
│           ↑                  │                                       │
│           │ 仇恨 + 锁定时用    │                                       │
│           │                  │                                       │
│   ┌─────────────────────────┴────────────────────────┐               │
│   │  ShouldAggro(target, ignore_forbidden?)          │               │
│   │  - shouldaggrofn 自定义过滤                       │               │
│   │  - shouldavoidaggro 临时白名单                    │               │
│   │  - target.shouldavoidaggrofn 拒绝                 │               │
│   │  - forbiddenaggrotags 标签黑名单（可绕过）         │               │
│   │  - target 不带 stealth                            │               │
│   │  - target 能死（除非 hostile）                     │               │
│   └──────────────────────────────────────────────────┘               │
│           ↑                                                          │
│           │ 主动设置目标时用                                          │
│           │                                                          │
│   ┌───────┴────────────────────────────────────────────┐             │
│   │  SetTarget(target)                                 │             │
│   │  - target != self.target                           │             │
│   │  - target == nil || (IsValidTarget && ShouldAggro) │             │
│   │  - target 没在 hiding 状态（玩家）                  │             │
│   └────────────────────────────────────────────────────┘             │
│                                                                       │
│   ┌────────────────────────────────────────────────────────────────┐ │
│   │  CanBeAttacked(attacker)        目标方主观——"我让你打吗"        │ │
│   │  - 不是 playerghost / flight                                    │ │
│   │  - 没 noattack / invisible（iframeskeepaggro 豁免）             │ │
│   │  - PVP 规则：玩家+玩家+无 propweapon → 拒绝                     │ │
│   │  - 玩家跟随者保护：攻击者是玩家小弟 → 拒绝                       │ │
│   │  - 攻击者疯狂（crazy/IsCrazy）→ 允许打几乎所有东西                │ │
│   │  - 暗影/噩梦生物对非疯狂玩家不可打（除非已锁定该玩家）            │ │
│   └────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│   ┌────────────────────────────────────────────────────────────────┐ │
│   │  CanBeAlly(guy) / IsAlly(guy)   "guy 是我的朋友吗"              │ │
│   │  - guy 带 alwayshostile → 永远不是                              │ │
│   │  - guy 是自己/我的 leader/我的小弟/共同 leader                   │ │
│   │  - 玩家阵营互认（bedazzled/companion/domesticated 等）           │ │
│   │  - PVP 关闭 + 玩家相关链 → 是盟友                                │ │
│   └────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│   ┌────────────────────────────────────────────────────────────────┐ │
│   │  HostileToPlayerTest(player)    暗影生物特有钩子                 │ │
│   │  - 玩家带 shadowdominance → false（玩家可"控制"它）              │ │
│   │  - 已锁定该玩家 → true                                          │ │
│   │  - 玩家疯狂（IsCrazy）→ true                                     │ │
│   │  - 否则视具体怪物而定                                            │ │
│   └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

**这张图的关键三个角度**：

1. **判定函数有两个视角**——**攻击者视角**（IsValidTarget / CanTarget / ShouldAggro / SetTarget）和**被攻击者视角**（CanBeAttacked、HostileToPlayerTest）。它们要**双向都通过**才算"打得到"。
2. **服务端的 IsValidTarget / CanTarget / CanBeAttacked / CanBeAlly 都委托给 replica**——21.3 反复见过这个模式（`combat.lua` 第 477-479、742-743、1282-1297 行），原因是这些规则在客户端预测和服务端执行时**必须用完全相同的代码**，所以集中放在 replica。
3. **`ShouldAggro` 只在服务端有**——因为客户端不会"主动设置目标"，所有客户端能做的就是"点击攻击"（走 CanAttack/CanHitTarget），不需要这层主观过滤。

---

### 21.4.2 IsValidTarget：最底层的基础校验

源码完整：

```288:305:scripts/components/combat_replica.lua
function Combat:IsValidTarget(target)
    if target == nil or
        target == self.inst or
        not (target.entity:IsValid() and target.entity:IsVisible()) then
        return false
    end

    local weapon = self:GetWeapon()
    return self:CanExtinguishTarget(target, weapon)
        or self:CanLightTarget(target, weapon)
        or (target.replica.combat ~= nil and
            not IsEntityDead(target, true) and
            not target:HasTag("spawnprotection") and
            not (target:HasTag("shadow") and self.inst.replica.sanity == nil and not self.inst:HasTag("crazy")) and
            not (target:HasTag("playerghost") and (self.inst.replica.sanity == nil or self.inst.replica.sanity:IsSane()) and not self.inst:HasTag("crazy")) and
			(TheNet:GetPVPEnabled() or not (self.inst.isplayer and target.isplayer) or (weapon and weapon:HasTag("propweapon"))) and
            target:GetPosition().y <= self._attackrange:value())
end
```

#### 分三段读

**第一段：三个 nil-safe 守卫**

```lua
if target == nil or
    target == self.inst or
    not (target.entity:IsValid() and target.entity:IsVisible()) then
    return false
end
```

最简单——目标存在、不是自己、entity 有效且可见。这是任何后续判定的前提。

**第二段：灭火/点火豁免**

```lua
local weapon = self:GetWeapon()
return self:CanExtinguishTarget(target, weapon)
    or self:CanLightTarget(target, weapon)
    or ...
```

——拿着灭火器对着冒烟物体、拿着点火器对着可点燃物体，**直接通过验证**！不走后面的战斗规则。

这是为什么你能"用水袋'攻击'营火"——本质上不是攻击，是利用攻击交互完成灭火工作。`CanExtinguishTarget` / `CanLightTarget` 见 `combat_replica.lua:224-261`。

**第三段：6 条战斗规则（AND 短路）**

只有这六条全部通过才算"有效战斗目标"：

| 条件 | 含义 |
|---|---|
| `target.replica.combat ~= nil` | 目标必须有 combat replica（即有 `_combat` 或 `__combat` 标签） |
| `not IsEntityDead(target, true)` | 目标不能死亡（`require_health = true` 表示没有 health 组件也算"未死亡"） |
| `not target:HasTag("spawnprotection")` | 目标不在出生保护 |
| `not (target:HasTag("shadow") && 我没 sanity && 我不疯狂)` | 暗影生物对没 sanity 的攻击者（如 NPC）默认隐身，疯狂时可见 |
| `not (target:HasTag("playerghost") && 我清醒 && 我不疯狂)` | 玩家鬼魂对清醒攻击者隐身，疯狂时可见 |
| PVP 规则（详见 21.4.5） | 玩家不打玩家，除非 PVP 开/拿 propweapon |
| `target.y ≤ self._attackrange` | 目标高度（飞行物）在攻击距离内 |

#### 注意几个微妙细节

**`IsEntityDead(target, true)` 的第二参数**——见 `componentutil.lua:7-13`：

```lua
function IsEntityDead(inst, require_health)
	local health = inst.replica.health
	if health == nil then
        return require_health == true
    end
	return health:IsDead()
end
```

——传 `true` 表示"目标没有 health 时算未死亡"（合理，因为没 health 本来就打不死它，但有些没 health 的物体（如墙）可以打）。

**`shadow` 标签和 `playerghost` 的视觉规则**：

- 攻击者**没有 sanity** + 攻击者**不疯狂** → shadow / playerghost 隐形（NPC 看不到）
- 攻击者**清醒（IsSane）** + 攻击者**不疯狂** → playerghost 隐形（清醒的玩家看不到鬼魂玩家）
- 攻击者**疯狂**（任何途径） → 全部可见

这就是为什么"玩家死后变鬼魂、清醒队友会无法把他设为目标"。

> **mod 实践陷阱**：你做"暗影鬼魂"风格的怪物，加 `shadow` 标签会让 NPC 看不到它——这通常是你想要的。但**`spawnprotection` 标签会让所有人都看不到**，注意它通常是玩家复活时短暂的，长期挂着会变成"无敌"。

**target.y 检查**——这是飞行 / 跳跃单位的硬豁免。空中蝙蝠飞太高就打不到，必须等它落下。

---

### 21.4.3 CanTarget：主动攻击的综合判定

```307:320:scripts/components/combat_replica.lua
function Combat:CanTarget(target)
    local rider = self.inst.replica.rider
    local weapon = self:GetWeapon()
	local is_ranged_weapon = weapon ~= nil and weapon:HasAnyTag("projectile", "rangedweapon")

    return self:IsValidTarget(target)
		and not (	self._ispanic:value() or
					target:HasAnyTag("INLIMBO", "notarget", "debugnoattack")
				)
		and (self.temp_iframes_keep_aggro or not target:HasTag("invisible"))
        and (target.replica.combat == nil
            or target.replica.combat:CanBeAttacked(self.inst))
        and (rider == nil or (not rider:IsRiding() or (not rider:GetMount():HasTag("peacefulmount") or is_ranged_weapon)))
end
```

#### 五道关卡

1. **IsValidTarget**（21.4.2）——前置基础校验。
2. **我没 panic + 目标无禁标签**：
   - `_ispanic:value()` 是恐慌状态（21.1.3 - 21.1.5）；
   - `INLIMBO` 目标进了 limbo；
   - `notarget` 目标禁选；
   - `debugnoattack` 调试期间禁用。
3. **目标不 invisible**（除非有 `iframeskeepaggro` 临时豁免，详 21.3.10）。
4. **`target.replica.combat:CanBeAttacked(self.inst)`**——**目标主观允许我打它**（21.4.6）。
5. **骑乘平静坐骑限制**：骑着 `peacefulmount`（如沃比、温迪伊吉巴）且非远程武器 → 不能主动攻击。这是温迪坐骑的"和平骑兽"特性——骑着小宝不能挥棒打人。但**拿了弓 / 弹弓**这种 `rangedweapon` 可以。

#### CanTarget vs CanAttack vs IsValidTarget 三者对比

| | `IsValidTarget` | `CanTarget` | `CanAttack` |
|---|---|---|---|
| 用途 | 基础校验，作为内部前置 | "是否可以仇恨/锁定" | "现在能否立刻发起攻击" |
| 调用方 | SetTarget、ValidateTarget、CanTarget、CanAttack | RetargetFn、KeepTargetFn、SuggestTarget | PlayerController 攻击瞬间、DoAttack 前 |
| 关键检查 | 死亡/可见/PVP | + panic/invisible/CanBeAttacked/坐骑 | + canattack/InCooldown/distance/busy |
| 谁拒绝？ | 攻击者视角 | 攻击者+目标 | 攻击者+目标+空间 |

**常见使用模式**：

- **RetargetFn**：用 `inst.components.combat:CanTarget(guy)` —— 检查"我能不能主动恨它"；
- **KeepTargetFn**：用 `inst.components.combat:CanTarget(target)` —— 检查"我能不能继续锁它"（注意：不是 CanAttack，因为目标暂时不在范围内不该立刻脱战）；
- **PlayerController.GetAttackTarget**：用 `combat:CanTarget(v)` —— 玩家点击攻击键时找"能否选为目标"；
- **CanAttack**：仅在 DoAttack 前的一刻验证"距离够吗、冷却好了吗"。

---

### 21.4.4 ShouldAggro：主动仇恨的最终过滤

`ShouldAggro` 在 21.2.8 已经简介过，这里讲完整决策树。

```419:445:scripts/components/combat.lua
function Combat:ShouldAggro(target, ignore_forbidden)
    if target ~= nil and
        (self.shouldaggrofn == nil or self.shouldaggrofn(self.inst, target)) and
        (self.shouldavoidaggro == nil or not self.shouldavoidaggro[target]) and
        (target.components.combat == nil or target.components.combat.shouldavoidaggrofn == nil or target.components.combat.shouldavoidaggrofn(self.inst, target))
        then
        if not ignore_forbidden and self.forbiddenaggrotags ~= nil then
            for _, tag in ipairs(self.forbiddenaggrotags) do
                if target:HasTag(tag) then
                    return false
                end
            end
        end
		if target:HasTag("stealth") then
			return false
		end
		if target.components.health ~= nil and (target.components.health.minhealth or 0) > 0 and not target:HasTag("hostile") then
			target = target.components.follower ~= nil and target.components.follower:GetLeader() or target
			if not target.isplayer then
				--npc should not aggro on things that can't be killed (unless hostile!)
				return false
			end
		end
        return true
    end
    return false
end
```

#### 决策树

```
ShouldAggro(target, ignore_forbidden):
│
├─ target == nil → false
│
├─ shouldaggrofn ~= nil && !shouldaggrofn(self, target) → false
│
├─ shouldavoidaggro[target] ~= nil → false  (临时白名单)
│
├─ target.combat 有 shouldavoidaggrofn && !shouldavoidaggrofn(self, target) → false
│
├─ !ignore_forbidden && forbiddenaggrotags 中任一 tag 在 target 上 → false
│
├─ target has "stealth" → false
│
├─ target 有 health && minhealth > 0 && 不带 hostile:
│   │   "目标实际上打不死（minhealth 兜底）且不是敌对生物"
│   │
│   ├─ target 有 follower：递归到 leader
│   │
│   └─ leader/target 不是玩家 → false
│      "npc 不该去恨永远打不死的非玩家东西"
│
└─ 通过所有检查 → true
```

#### 几个关键细节

**`ignore_forbidden` 的语义**——21.3.8 提到过：`ShareTarget` 调用 `ShouldAggro(target, true)` 时跳过 `forbiddenaggrotags`，因为"同伴让我打这家伙，我自己的 forbidden 不该影响"。

**`shouldavoidaggrofn` 写在目标身上**——这就是为什么 21.2.8 强调"目标可以主动拒绝被仇恨"。常见用途：玩家施法"伪装"，所有怪物的 ShouldAggro 都不通过。

**`minhealth > 0` 的玄机**——`health` 组件的 `minhealth` 表示"血量再低也不能死到这个值以下"。某些不死实体（如温迪的鬼姐姐 Abigail、伍迪海狸形态、wagstaff 等）会设 minhealth > 0 表示"打不死"。

这条规则的意思是：

- 一个生物**有 health**（不是无生命物体）；
- 它**minhealth > 0**（实际上打不死）；
- 它**没 hostile 标签**（即不是敌对怪物，而是友善实体如 Abigail）；
- 它的 leader/自己**不是玩家**；

→ NPC 怪物**不应该主动仇恨**这种"打不死也不敌对的东西"。

但**例外**：

- 如果它的 follower leader 是**玩家**（比如 Abigail，她跟着温迪），那 ShouldAggro 仍然返回 true，因为攻击 leader 跟随者间接攻击玩家。
- 如果它带了 `hostile` 标签（即"虽然打不死但敌对"），返回 true。

#### `forbiddenaggrotags` 的"覆盖式" vs "增量式"

`SetNoAggroTags` 是**覆盖整个数组**：

```462:464:scripts/components/combat.lua
function Combat:SetNoAggroTags(tags) -- Wants a table in an ipairs table format.
    self.forbiddenaggrotags = tags
end
```

`AddNoAggroTag` / `RemoveNoAggroTag` 是**增量**：

```447:460:scripts/components/combat.lua
function Combat:AddNoAggroTag(tag)
    self.forbiddenaggrotags = self.forbiddenaggrotags or {}
    table.insert(self.forbiddenaggrotags, tag)
end

function Combat:RemoveNoAggroTag(tag)
    if self.forbiddenaggrotags == nil then
        return
    end
    table.removearrayvalue(self.forbiddenaggrotags, tag)
    if self.forbiddenaggrotags[1] == nil then
        self.forbiddenaggrotags = nil
    end
end
```

**实战**：

```lua
-- 守卫猪：不打其他猪、其他守卫、不在 limbo 的实体
inst.components.combat:SetNoAggroTags({"pig", "guard", "INLIMBO"})

-- 加 buff：临时让我也不恨蜘蛛
inst.components.combat:AddNoAggroTag("spider")
-- buff 结束
inst.components.combat:RemoveNoAggroTag("spider")
```

> **常见坑**：`forbiddenaggrotags` 是**ipairs 数组**，不是 set。多次 AddNoAggroTag 同一个标签会重复追加，RemoveNoAggroTag 只删一次。mod 开发者要自己维护"是否已加"。

---

### 21.4.5 进阶档：PVP 规则与玩家阵营

PVP 关闭时，玩家之间不能攻击。但实现远比"加个 if 判断"复杂——涉及三个独立的 PVP 检查点：

#### 检查点 1：IsValidTarget 内（攻击者视角）

```303:scripts/components/combat_replica.lua
(TheNet:GetPVPEnabled() or not (self.inst.isplayer and target.isplayer) or (weapon and weapon:HasTag("propweapon")))
```

——读起来：**PVP 开**，**或**（我不是玩家**或**目标不是玩家），**或** 武器是道具武器。

只要这三个条件之一为真，玩家就能尝试攻击玩家。

**`propweapon`**——典型的"玩具锤子"、"小弟拳"道具武器。让 PVP 关闭时仍能"打闹"，主要用于喜剧效果或表演性挑战。

#### 检查点 2：CanBeAttacked 内（被攻击者视角）

```427:435:scripts/components/combat_replica.lua
if attacker.isplayer and not TheNet:GetPVPEnabled() then
    --PVP check
    local combat = attacker.replica.combat
    local weapon = combat ~= nil and combat:GetWeapon() or nil
    if weapon == nil or not weapon:HasTag("propweapon") then
        --Allow friendly fire with props
        return false
    end
end
```

——同样的规则，但从**目标的视角**：目标是玩家，攻击者是玩家，PVP 关闭，且没拿 propweapon → 拒绝被攻击。

**双重检查的设计意图**：

1. **客户端预测和服务端执行都跑同一份代码**，避免误差；
2. **服务端有最终决定权**——客户端可能因为延迟用了旧的 PVP 状态，服务端再过滤一次。

#### 检查点 3：玩家跟随者保护

```436:450:scripts/components/combat_replica.lua
if self._target:value() ~= attacker then
    local follower = attacker.replica.follower
    if follower ~= nil then
        local leader = follower:GetLeader()
        if leader ~= nil and
            leader ~= self._target:value() and
			leader.isplayer and not attacker:HasTag("alwayshostile") then
            local combat = attacker.replica.combat
            if combat ~= nil and combat:GetTarget() ~= self.inst then
                --Follower check
                return false
            end
        end
    end
end
```

**条件**：

- 我（目标）是玩家；
- 我**当前没锁定攻击者**（即不是"我打他他还手"的局面）；
- 攻击者**有 follower 组件**；
- 攻击者的 **leader 是玩家**且**不是我**；
- 攻击者**不带 `alwayshostile` 标签**；
- 攻击者**也没锁定我**作为目标。

→ **拒绝被这只小弟打**。

#### 这是干嘛？

这段保护**"玩家 A 的小弟（猪/猫狸/招募的怪物）不会误伤玩家 B"**。比如玩家 A 招了一只猪，猪跟着 A 跑来跑去；玩家 B 路过，猪不会主动打 B，**B 也无法手动让小弟打自己**（除非 PVP 开或猪已经在追 B）。

**注意双向豁免**：

- "我打他他还手" → `self._target:value() == attacker` → 跳过此检查；
- "他锁定了我" → `combat:GetTarget() == self.inst` → 跳过；
- 攻击者带 `alwayshostile` → 跳过（敌对生物谁都打）。

#### CanBeAlly / IsAlly 的玩家阵营互认

```322:380:scripts/components/combat_replica.lua
function Combat:CanBeAlly(guy)
    if guy:HasTag("alwayshostile") then
        return false
    end

    if guy == self.inst then
        return true -- It's me.
    end

	local follower = self.inst.replica.follower
	local myleader = follower and follower:GetLeader()
    if myleader and guy == myleader then
        return true -- It's my leader.
    end

	local guy_follower = guy.replica.follower
	local theirleader = guy_follower and guy_follower:GetLeader()
    if self.inst == theirleader then
        return true -- It's my follower.
    end

    if myleader and myleader == theirleader then
        return true -- Same leader we should be friends.
    end

	-- Are we player aligned?
	if self.inst.isplayer or
		self.inst.bedazzled or
		(myleader and (myleader.isplayer or myleader.replica.inventoryitem)) or
		self.inst:HasAnyTag("domesticated", "saltlicker_salted")
	then
		if guy.bedazzled or
			guy:HasTag("companion") or
			(theirleader and theirleader.replica.inventoryitem)
		then
			return true -- Consider it aligned to all players.
		end

		if not TheNet:GetPVPEnabled() then
			if guy.isplayer or (theirleader and theirleader.isplayer) then
				return true -- They're aligned to another player.
			end

			if guy:HasAnyTag("domesticated", "saltlicker_salted") then
				return true -- No current leader, but still considered aligned to players.
			end
		end
	end

	return false
end
```

#### 五道盟友判定关卡

```
CanBeAlly(guy):
│
├─ guy has "alwayshostile" → false  (始终敌对)
│
├─ guy == self.inst → true  (我自己)
│
├─ myleader && guy == myleader → true  (guy 是我的 leader)
│
├─ self.inst == guy.leader → true  (guy 是我的小弟)
│
├─ myleader && myleader == theirleader → true  (同 leader)
│
└─ 玩家阵营互认（关键）：
    │
    ├─ 我属于玩家阵营？
    │   - isplayer
    │   - bedazzled（被蜘蛛蛛网帽迷惑）
    │   - 我的 leader 是玩家或玩家持有物（小弟的小弟）
    │   - 我带 domesticated（已驯化）或 saltlicker_salted（盐舔）
    │
    └─ 是的话，guy 也是玩家阵营？
        - guy bedazzled
        - guy 带 companion
        - guy 的 leader 是玩家持有物
        - PVP 关闭时：guy 是玩家、guy 的 leader 是玩家、guy 带 domesticated/saltlicker_salted
        
        → true
```

#### IsAlly vs CanBeAlly

```374:380:scripts/components/combat_replica.lua
function Combat:IsAlly(guy)
	if not self:CanBeAlly(guy) then
		return false
	end
	local guy_combat = guy.replica.combat
	return guy_combat == nil or guy_combat:GetTarget() ~= self.inst
end
```

**`IsAlly = CanBeAlly && guy 没锁定我**`

——区别在于：**guy 已经把我当成目标，他就不是我的盟友了**（即使阵营上是）。这是反击逻辑：被自家人锁定（如反水的猪），我可以攻击他。

#### 实战：群体反水

经典例子是猪在月圆变野猪后会攻击玩家——它的 `combat:GetTarget()` 已经是玩家，所以 `IsAlly` 返回 false，玩家可以反击。

---

### 21.4.6 进阶档：CanBeAttacked 完整决策树

这是最复杂的判定函数之一，把 21.4.5 的 PVP 检查点 + 暗影规则 + 疯狂攻击者豁免全融合在内。

```408:489:scripts/components/combat_replica.lua
function Combat:CanBeAttacked(attacker)
	if self.inst:HasAnyTag("playerghost", "flight") or
		(	not self.temp_iframes_keep_aggro and
			self.inst:HasAnyTag("noattack", "invisible")
		)
	then
        return false
	end

	local sanity

	if attacker ~= nil then
		if attacker.isplayer and self.inst:HasTag("noplayertarget") then
            return false
        elseif attacker ~= self.inst and self.inst.isplayer then
			if attacker.isplayer and not TheNet:GetPVPEnabled() then
                local combat = attacker.replica.combat
                local weapon = combat ~= nil and combat:GetWeapon() or nil
                if weapon == nil or not weapon:HasTag("propweapon") then
                    return false
                end
            end
            if self._target:value() ~= attacker then
                local follower = attacker.replica.follower
                if follower ~= nil then
                    local leader = follower:GetLeader()
                    if leader ~= nil and
                        leader ~= self._target:value() and
						leader.isplayer and not attacker:HasTag("alwayshostile") then
                        local combat = attacker.replica.combat
                        if combat ~= nil and combat:GetTarget() ~= self.inst then
                            return false
                        end
                    end
                end
            end
        end

		sanity = attacker.replica.sanity

        if sanity ~= nil and sanity:IsCrazy() or attacker:HasTag("crazy") then
            return true
        end
    end

	if self.inst:HasAnyTag("shadowcreature", "nightmarecreature") and
		(	self._target:value() == nil
			or (
				attacker ~= nil and
				self._target:value() ~= attacker and
				(self.inst.HostileToPlayerTest ~= nil and not self.inst:HostileToPlayerTest(self._target:value()))
				)
		) and
        (attacker ~= nil or self.inst:HasTag("locomotor")) then
        return false
    end

    return true
end
```

#### 决策树（从顶向下）

```
CanBeAttacked(attacker):
│
├─ 1. 我自己豁免：
│  - playerghost / flight → false
│  - noattack / invisible（且不是 iframeskeepaggro）→ false
│
├─ 2. 攻击者非 nil 时的玩家相关检查：
│  ├─ "我是 noplayertarget 且攻击者是玩家" → false
│  │     （波利鹦鹉等同伴宠物用此标签拒绝玩家攻击）
│  │
│  └─ "我是玩家 + 攻击者不是我"：
│      ├─ PVP 检查（见 21.4.5 检查点 2）
│      └─ 玩家跟随者保护（见 21.4.5 检查点 3）
│
├─ 3. 疯狂攻击者豁免：
│  - attacker.replica.sanity:IsCrazy() OR attacker HasTag("crazy") → true (跳出所有后续检查！)
│
├─ 4. 暗影/噩梦生物的特殊保护：
│  "我是 shadowcreature 或 nightmarecreature 且 我没目标 / 我目标不是攻击者且 HostileToPlayerTest 返回 false"
│  + "有攻击者，或 我是 locomotor（暗影生物本体）"
│  → false
│  （非疯狂攻击者无法主动打不仇恨自己的暗影生物）
│
└─ 5. 所有检查通过 → true
```

#### 第 3 点的疯狂豁免特别重要

```lua
if sanity ~= nil and sanity:IsCrazy() or attacker:HasTag("crazy") then
    return true  -- 跳出后续 shadow/nightmare 检查
end
```

——疯狂攻击者绕过暗影生物的"半透明保护"。这是饥荒的核心机制：**山姆模糊的疯狂状态让你能反击暗影怪**。

> **注释提到的 V2C TODO**——`combat_replica.lua:469-471` 写着"原本设计是要求攻击者低于 50% 理智才能帮队友打暗影怪"，但因为"能选暗影生物当目标本来就不该允许"被改了。这个注释保留至今，看到时不必惊讶。

#### 第 4 点的暗影生物保护

```lua
if self.inst:HasAnyTag("shadowcreature", "nightmarecreature") and
    (	self._target:value() == nil
        or (
            attacker ~= nil and
            self._target:value() ~= attacker and
            (self.inst.HostileToPlayerTest ~= nil and not self.inst:HostileToPlayerTest(self._target:value()))
            )
    ) and
    (attacker ~= nil or self.inst:HasTag("locomotor")) then
    return false
end
```

**用人话翻译**：

> "如果我是暗影/噩梦生物，且（我没在打人 或 我在打的不是 attacker 且 我不仇恨我当前的目标），那么不能被攻击。"
> "但 AOE 攻击（attacker = nil）只对会动的暗影生物（locomotor）有效，对'看不见的手'这类静态暗影无效。"

**逻辑原理**：

- 我是暗影生物，但我已经锁定了 attacker → 允许 attacker 反击我（因为我在打他了）；
- 我是暗影生物，没有目标 → 不允许任何人打我（隐身保护）；
- 我有目标但不是 attacker，但 `HostileToPlayerTest(目标)` 返回 true → 允许（因为我对玩家敌对）；
- 我有目标但不是 attacker，且 `HostileToPlayerTest(目标)` 返回 false → 不允许（我对目标"温和"，路人不该过来打我）。

#### `HostileToPlayerTest` 钩子（在 prefab 里挂）

```170:181:scripts/prefabs/shadowcreature.lua
local function CLIENT_ShadowSubmissive_HostileToPlayerTest(inst, player)
	if player:HasTag("shadowdominance") then
		return false
	end
	local combat = inst.replica.combat
	if combat ~= nil and combat:GetTarget() == player then
		return true
	end
	local sanity = player.replica.sanity
	if sanity ~= nil and sanity:IsCrazy() then
		return true
	end
```

—— `shadowcreature.lua` 完整版还有更多分支，最后默认 return false（如果是 shadowsubmissive 形态）。

**`HostileToPlayerTest` 函数签名**：`fn(inst, player) -> bool`，返回"这个生物对该玩家敌对吗"。它**挂在生物自己身上**，被 `CanBeAttacked` 调用。

**典型实现的三条规则**：

1. 玩家带 `shadowdominance`（如装备影刺）→ false（玩家"支配"暗影，暗影不敌对）；
2. 我已经在打这个玩家 → true（敌对中）；
3. 玩家疯狂 → true（疯狂状态暗影才显露）；
4. 否则 → false（半透明状态，不敌对）。

`shadowsubmissive` 状态下的暗影怪就用这套规则——它能"被支配"，玩家挂着 `shadowdominance` 标签时它不主动敌对，**而且玩家也不能直接打它**（CanBeAttacked 内 shadow 保护层就用这个钩子）。

---

### 21.4.7 进阶档：玩家攻击玩家的完整决策链

来个综合案例：**Wilson 攻击 Wickerbottom**，PVP 关闭，Wilson 拿着普通斧子。看完整决策链：

**步骤 1**：Wilson 点击 Wickerbottom，PlayerController 调 `ValidateAttackTarget`：

```1741:1779:scripts/components/playercontroller.lua
local function ValidateAttackTarget(combat, target, force_attack, x, z, has_weapon, reach)
	if not combat:CanTarget(target) or target:HasTag("stealth") then
        return false
    end

    local targetcombat = target.replica.combat
    if targetcombat ~= nil then
        if combat:IsAlly(target) then
            return false
        elseif not (force_attack or combat:IsRecentTarget(target)) then
            ...
		end
    end
    ...
end
```

——先 `combat:CanTarget(target)` → 走到 `IsValidTarget` 内。

**步骤 2**：`IsValidTarget` 检查 PVP 规则：

```lua
(TheNet:GetPVPEnabled() or not (self.inst.isplayer and target.isplayer) or (weapon and weapon:HasTag("propweapon")))
-- = false or false or false = false
```

——PVP 关闭、双方都是玩家、武器（斧子）不是 propweapon → **IsValidTarget 返回 false**。

**步骤 3**：CanTarget 返回 false → `ValidateAttackTarget` 返回 false → 玩家点击无效。

#### 同一个 Wilson 但拿了"小弟拳"（propweapon）

**步骤 2'**：IsValidTarget 检查 PVP：

```lua
(false or false or (weapon and true)) = true
```

——`propweapon = true`，IsValidTarget 通过！

**步骤 3'**：`CanTarget` 检查 `target.replica.combat:CanBeAttacked(self.inst)`：

```lua
-- 进入 CanBeAttacked，Wickerbottom 视角：
-- 攻击者是玩家、我是玩家、PVP 关闭：
local weapon = attacker_combat:GetWeapon()  -- 小弟拳
if weapon == nil or not weapon:HasTag("propweapon") then
    return false
end
-- 走到这里说明武器是 propweapon → 没拒绝
```

——CanBeAttacked 通过！

**步骤 4'**：`combat:IsAlly(target)`？

走 `CanBeAlly` 流程：

- 双方都是玩家 → 都属于"玩家阵营"；
- PVP 关闭 → guy.isplayer = true → 返回 true (盟友！)；

所以 `IsAlly` 返回 true → `ValidateAttackTarget` 拒绝。

**结论**：拿 propweapon 也**不能直接打**自己阵营玩家。需要**强制攻击**（强行按攻击键 / Ctrl+F）。

#### `force_attack` 路径

```lua
elseif not (force_attack or combat:IsRecentTarget(target)) then
    ...
end
```

——`force_attack = true` 时跳过盟友/敌对检查，让玩家强制攻击同伴。这是"友军误伤"的实现方式。

---

### 21.4.8 进阶档：暗影生物的多层判定

汇总暗影生物相关的标签和钩子：

| 标签 / 钩子 | 含义 | 影响 |
|---|---|---|
| `shadowcreature` | 暗影生物本体 | `CanBeAttacked` 第 4 层保护；`IsValidTarget` 中 NPC 看不见 |
| `nightmarecreature` | 噩梦生物（恶梦门狗等） | 同上 |
| `shadow` | 通用暗影标签 | `IsValidTarget` 中无 sanity 的 NPC 看不见 |
| `shadowsubmissive` | 半透明状态（被玩家支配） | 必须配合 `shadowsubmissive` 组件使用 |
| `shadowdominance` | "支配暗影"（玩家） | 玩家挂此标签时影刺等装备让暗影怪 HostileToPlayerTest = false |
| `HostileToPlayerTest` | 函数钩子 | 自定义"对该玩家是否敌对" |
| `crazy` | "疯狂"状态 | `CanBeAttacked` 中直接 return true 跳过所有保护 |

#### `shadowsubmissive` 组件

```lua
inst:AddComponent("shadowsubmissive")
```

——这个组件由项目内置（`scripts/components/shadowsubmissive.lua`），它做的就是：

- 把 `inst.HostileToPlayerTest` 挂上去；
- 跟踪 nightmare creature 等是否"被支配"；
- 跟 `targetfn` 配合：在仇恨判定时优先打"支配自己的玩家"。

如 `nightmarecreature.lua`：

```22:43:scripts/prefabs/nightmarecreature.lua
local function targetfn(inst)
    local x, y, z = inst.Transform:GetWorldPosition()
    ...
        for i, v in ipairs(TheSim:FindEntities(x, y, z, TUNING.SHADOWCREATURE_TARGET_DIST, nil, NO_TAGS, INSANITY_TAGS)) do
            if v.entity:IsVisible() and not v.components.health:IsDead() then
                local distsq = inst:GetDistanceSqToInst(v)
                if inst.components.shadowsubmissive:TargetHasDominance(v) then
                    if distsq < rangesq1 and inst.components.combat:CanTarget(v) then
                        target1 = v
                        rangesq1 = distsq
                    end
                ...
```

——它们的 RetargetFn 会优先打"有 dominance 的玩家"（实际上是因为玩家挂着 shadowdominance 标签）。当玩家"控制"暗影时，它不会主动恨别人，只会等候命令。

---

### 21.4.9 老手必读：HostileToPlayerTest 的实现模式

#### 模式 1：基本暗影生物

```lua
local function HostileToPlayerTest(inst, player)
    -- 玩家携带"影刺装备"
    if player:HasTag("shadowdominance") then
        return false
    end
    -- 我已在攻击该玩家
    if inst.components.combat and inst.components.combat:TargetIs(player) then
        return true
    end
    -- 玩家疯狂
    if player.components.sanity and player.components.sanity:IsCrazy() then
        return true
    end
    -- 默认：对清醒玩家半透明，不敌对
    return false
end
```

#### 模式 2：噩梦生物（永远敌对）

```lua
local function HostileToPlayerTest(inst, player)
    if player:HasTag("shadowdominance") then
        return false
    end
    -- 默认 true：永远敌对（噩梦门狗等）
    return true
end
```

#### 模式 3：服务端版本（用 components）

```lua
-- master_postinit
inst.HostileToPlayerTest = HostileToPlayerTest_Server

local function HostileToPlayerTest_Server(inst, player)
    -- 服务端可以读 components.combat, components.sanity 等
    if player:HasTag("shadowdominance") then
        return false
    end
    local sanity = player.components.sanity
    return sanity and sanity:IsCrazy()
end
```

#### 模式 4：客户端版本（用 replica）

```lua
-- common_postinit（pristine 之前）
inst.HostileToPlayerTest = HostileToPlayerTest_Client

local function HostileToPlayerTest_Client(inst, player)
    -- 客户端只能用 replica（player.components.sanity 是 nil）
    if player:HasTag("shadowdominance") then
        return false
    end
    local sanity = player.replica.sanity
    return sanity and sanity:IsCrazy()
end
```

**对比 `oceanshadowcreature.lua:320-334`**：

```320:332:scripts/prefabs/oceanshadowcreature.lua
local function CLIENT_ShadowSubmissive_HostileToPlayerTest(inst, player)
	if player:HasTag("shadowdominance") then
		return false
	end
	local combat = inst.replica.combat
	if combat ~= nil and combat:GetTarget() == player then
		return true
	end
	local sanity = player.replica.sanity
	if sanity ~= nil and sanity:IsCrazy() then
		return true
	end
```

——典型客户端实现，全走 replica。

#### 为什么挂 `HostileToPlayerTest` 在两端都要？

`CanBeAttacked` 既在客户端预测、又在服务端执行。客户端看不到 `inst.components`，必须用 replica。所以**实现要在两端都安装一份**。

`nightmarecreature.lua` 的最佳实践：

```225:226:scripts/prefabs/nightmarecreature.lua
		--shadowsubmissive (from shadowsubmissive component) added to pristine state for optimization
		inst:AddTag("shadowsubmissive")
		inst.HostileToPlayerTest = CLIENT_ShadowSubmissive_HostileToPlayerTest
```

——`common_postinit`（pristine 前）就挂上**客户端版本**。然后服务端 `AddComponent("shadowsubmissive")` 时，组件**内部**会覆盖成服务端版本（如果有）。

> **mod 实践**：你做新的暗影怪物，**始终挂客户端版本**就够了——除非你的判定逻辑必须读 `components`（极少数情况）。

---

### 21.4.10 mod 实战姿势

#### 新手档：写一个"对村民友好的怪物"

```lua
local function ShouldAggroFn(inst, target)
    -- 不打村民（用 prefab 判断）
    if target.prefab == "pigman" or target.prefab == "merm" then
        return false
    end
    -- 不打玩家阵营
    if inst.components.combat:CanBeAlly(target) then
        return false
    end
    return true
end

local function fn()
    local inst = CreateEntity()
    ...
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("combat")
    inst.components.combat:SetShouldAggroFn(ShouldAggroFn)
    ...
end
```

**注意细节**：

- 用 `prefab` 字段判断比标签判断**更精确**——`pig` 标签所有猪类都有（守卫、野猪），但 `prefab == "pigman"` 只匹配普通猪人。
- 用 `CanBeAlly` 一次性排除所有玩家阵营（含跟随玩家的小弟）——比挨个判定省心。

#### 进阶档：实现"敌人盟友判定"

你做一个**有阵营的敌方头目**——它和小弟（mod 自定义的怪兵）算同一阵营，互相不打：

```lua
-- 给所有同阵营单位挂标签
local function MakeFaction(inst)
    inst:AddTag("my_faction_member")
end

-- ShouldAggroFn
local function ShouldAggroFn(inst, target)
    -- 不打同阵营
    if target:HasTag("my_faction_member") then
        return false
    end
    return true
end

-- 兜底：CanBeAlly 钩子（如果你的怪物有 combat 组件）
local function OnInit(inst)
    -- Hijack CanBeAlly：用 shouldavoidaggrofn 拒绝被敌阵营仇恨
    inst.components.combat.shouldavoidaggrofn = function(attacker, self_target)
        -- 如果攻击者也是同阵营 → 拒绝
        return not attacker:HasTag("my_faction_member")
    end
end
```

注意**双向豁免**：

- 用 `ShouldAggroFn` 让自己不主动恨同阵营；
- 用 `shouldavoidaggrofn` 让同阵营不来恨我。

这两个一起用，才能彻底"阵营互不攻击"。

#### 老手档：实现 PVP buff（让玩家间无视 PVP 关闭限制）

这种 buff 很特殊——你**不能改 `TheNet:GetPVPEnabled()`** 因为它是全局开关。但你能：

```lua
-- 给玩家加一个"决斗状态"标签
inst:AddTag("dueling_mode")

-- AddPrefabPostInit 给所有玩家修改判定
AddPrefabPostInit("wilson", function(inst)
    if not TheWorld.ismastersim then return end
    
    -- 服务端 hook
    local OldIsValidTarget = inst.replica.combat.IsValidTarget
    function inst.replica.combat:IsValidTarget(target)
        -- 攻击者和目标都在决斗模式时，临时把 PVP 视为开
        if self.inst:HasTag("dueling_mode") and target:HasTag("dueling_mode") then
            local original_pvp = TheNet.GetPVPEnabled
            TheNet.GetPVPEnabled = function() return true end
            local result = OldIsValidTarget(self, target)
            TheNet.GetPVPEnabled = original_pvp
            return result
        end
        return OldIsValidTarget(self, target)
    end
end)
```

> **极其危险**：上面的 hook 模式会**全局**短暂改 `TheNet:GetPVPEnabled()`！如果有 race condition（异步事件触发其他玩家的判定）会出错。更安全的是**重写整个 IsValidTarget**，明确条件分支。但这超出新手范围——记得测试。

更优雅的方式：**让 mod 武器带 `propweapon` 标签**，已有的 PVP 豁免就能让玩家互打。

#### 老手档：让 NPC 仇恨"打不死"的目标

21.4.4 提到 ShouldAggro 会跳过 `minhealth > 0 && 非 hostile && 非玩家相关` 的目标。你做一个"必须击败但有 minhealth"的精英怪（像剧情 boss），希望路过的 NPC 也仇恨它：

```lua
-- 给精英怪加 hostile 标签
inst:AddTag("hostile")

-- 或：直接修改 health 让 minhealth = 0（但血量很低也不死的逻辑要走别处）
inst.components.health.minhealth = 0
inst.components.health:SetInvincible(true)  -- 用 invincible 替代 minhealth
```

#### 实战：万物书桂花猫的友好性

参考万物书的桂花猫 (`02_osmanthus_cat.lua`)：

```lua
-- (大致结构，简化版)
local function ShouldAggroFn(inst, target)
    if target:HasTag("cat") and target.prefab ~= inst.prefab then
        return false  -- 不打其他猫
    end
    if not TheNet:GetPVPEnabled() and target.isplayer then
        return false  -- PVP 关闭时不打玩家
    end
    return true
end
```

—— 这种 "PVP-aware" 的 ShouldAggroFn 是 mod 良好兼容性的标志。

---

### 21.4.11 老手必读：判定函数的客户端 vs 服务端差异

虽然 `combat_replica.lua` 里的 IsValidTarget / CanTarget / CanBeAttacked / CanBeAlly 服务端和客户端共用，但它们**读取的状态可能不同**：

| 状态 | 服务端 | 客户端 |
|---|---|---|
| `target.replica.combat:GetTarget()` | 实时最新（无网络延迟） | 滞后 1-2 tick（netvar 同步） |
| `self._ispanic:value()` | 同上 | 同上 |
| `target:GetPhysicsRadius(0)` | 立即 | 立即（物理在两端都准） |
| `TheNet:GetPVPEnabled()` | 真实值 | 客户端本地缓存（极少差异） |
| `attacker.replica.sanity:IsCrazy()` | 服务端的 sanity（精确） | classified 同步过来的 |
| `inst.components.combat`（用于直接调原组件方法） | 服务端有 | 客户端 nil |

**实战影响**：

- **mod 的 IsValidTarget 钩子里读 `inst.components.xxx`**：客户端会崩溃！必须用 `inst.replica.xxx`。
- **客户端预测拒绝攻击，但服务端可能接受**：例如客户端 sanity 同步晚一步，本地判断 sanity 不疯狂、不让点击；服务端实际已疯狂，但因客户端没点击，根本没传请求过来。结果：玩家觉得"打不了"。

> **mod 实践**：扩展 IsValidTarget 时，**优先用标签而不是组件**——标签同步早、判定快、客户端服务端一致。

#### `Combat:IsValidTarget`（服务端 combat.lua）只是包装

```477:479:scripts/components/combat.lua
function Combat:IsValidTarget(target)
    return self.inst.replica.combat:IsValidTarget(target)
end
```

服务端的 `Combat:IsValidTarget` **委托给 replica**——所以即使你在服务端 mod 修改了 `inst.components.combat.IsValidTarget`，**不会生效**！必须修改 `inst.replica.combat.IsValidTarget`。

```422:423:scripts/components/combat.lua
function Combat:ShouldAggro(target, ignore_forbidden)
    ...
    (target.components.combat == nil or target.components.combat.shouldavoidaggrofn == nil or target.components.combat.shouldavoidaggrofn(self.inst, target))
```

——而 `ShouldAggro` **是服务端 combat.lua 内的方法**，可以读 `target.components.combat.shouldavoidaggrofn`。

这种**有些方法在 replica、有些在 components** 的设计是饥荒源码"实战经验沉淀"——具体原则：

- **要让客户端预测的方法**（点击攻击的瞬间）→ **放 replica**；
- **服务端独有的状态决策**（搜目标、KeepTarget 周期）→ **放 components**。

---

### 21.4.12 五个常见坑

#### 坑 1：在 IsValidTarget 钩子里用 components

```lua
-- AddPrefabPostInit("wilson", ...)
local old = inst.replica.combat.IsValidTarget
function inst.replica.combat:IsValidTarget(target)
    if target.components.health:IsDead() then  -- ❌ 客户端崩溃
        return false
    end
    return old(self, target)
end
```

正确：

```lua
function inst.replica.combat:IsValidTarget(target)
    if IsEntityDead(target) then  -- 自动走 replica.health
        return false
    end
    return old(self, target)
end
```

#### 坑 2：误以为 IsValidTarget 检查 panic

```lua
-- 怪物处于 panic 状态，玩家想点击攻击它
combat:IsValidTarget(target)  -- 通过！
-- 但 CanTarget 会拒绝（panic 检查在那里）
```

——`IsValidTarget` 不检查我自己的状态（只检查目标），所以即使我 panic 也通过。**panic 检查在 `CanTarget` 里**。这是 21.4.3 强调的细节。

如果你做"恐慌时无法攻击"的逻辑，必须用 `CanTarget` 或者**显式**检查 `combat._ispanic:value()`。

#### 坑 3：用 `IsAlly` 判定时忘记考虑目标已锁定

```lua
local function ShouldAggroFn(inst, target)
    -- 不打盟友
    if inst.components.combat:IsAlly(target) then
        return false
    end
    return true
end
```

——看起来对，但**漏了一个细节**：`IsAlly` 内部已检查"target 是否锁定我"：

```374:380:scripts/components/combat_replica.lua
function Combat:IsAlly(guy)
	if not self:CanBeAlly(guy) then
		return false
	end
	local guy_combat = guy.replica.combat
	return guy_combat == nil or guy_combat:GetTarget() ~= self.inst
end
```

→ `IsAlly = CanBeAlly && !target.target == self.inst`

意思是：如果 target 已经在打我，它**不是**我的盟友，我可以反击。这正是你要的！但**如果你用 `CanBeAlly` 而不是 `IsAlly`**：

```lua
if inst.components.combat:CanBeAlly(target) then  -- ❌
    return false
end
```

——这就 bug 了：反水的盟友（如月圆变野猪、被催眠的猪）会一直被你视为盟友，你不会反击。

**记住**：**仇恨判定用 `IsAlly`，纯阵营查询用 `CanBeAlly`**。

#### 坑 4：noplayertarget 的反向效果

某些 mod 想做"NPC 永不被攻击"，加了 `noplayertarget` 标签：

```lua
inst:AddTag("noplayertarget")  -- 玩家无法选为目标
```

——但**仅阻止玩家**！其他怪物**仍能仇恨**这个 NPC。如果你想完全防御：

```lua
inst:AddTag("noplayertarget")  -- 玩家不打
inst:AddTag("notarget")        -- 其他怪物不主动找
inst:AddTag("noattack")        -- 真受到攻击时不进入 GetAttacked
```

三连击——但注意 `noattack` 会让玩家**点击**这个 NPC 时直接没反应（不是"打不到"而是"看不到"作为目标）。

#### 坑 5：在 CanBeAlly 钩子里假定 PVP

某些 mod 给玩家加自定义 buff，希望"buff 期间不被自家小弟打"。改 CanBeAlly：

```lua
local old = inst.replica.combat.CanBeAlly
function inst.replica.combat:CanBeAlly(guy)
    -- 我有 buff 时，自家小弟视为盟友
    if self.inst:HasTag("my_special_buff") then
        if guy.replica.follower and guy.replica.follower:GetLeader() == self.inst then
            return true  -- 总是盟友
        end
    end
    return old(self, guy)
end
```

——看起来对，但**PVP 开时**，`CanBeAlly` 的内置玩家阵营互认会**返回 false**（因为 PVP 开了，玩家间不互认）。所以你的小弟在 PVP 开时**仍然能打你**！

正确做法：

```lua
function inst.replica.combat:CanBeAlly(guy)
    -- 显式提前优先：自家小弟永远是盟友
    if self.inst:HasTag("my_special_buff") then
        local f = guy.replica.follower
        if f and f:GetLeader() == self.inst then
            return true
        end
    end
    return old(self, guy)
end
```

**关键**：判定函数的扩展应该**优先于原逻辑**而非"原逻辑包装"，避免被 PVP 等全局开关绕过。

---

### 21.4.13 调试技巧

**1. 控制台快查目标的可打性**

```lua
local player = ConsoleCommandPlayer()
local target = c_select()  -- 鼠标选中的实体
local combat = player.components.combat

print("IsValidTarget:", combat:IsValidTarget(target))
print("CanTarget:", combat:CanTarget(target))
print("ShouldAggro:", combat:ShouldAggro(target))
print("CanBeAttacked:", target.replica.combat:CanBeAttacked(player))
print("IsAlly:", combat:IsAlly(target))
print("CanBeAlly:", combat:CanBeAlly(target))
```

——一行行检查"卡在哪一关"。

**2. 临时关闭某个守卫看看**

```lua
-- 让所有怪物把玩家视为可打
local p = ConsoleCommandPlayer()
p:RemoveTag("playerghost")
p:RemoveTag("spawnprotection")
```

**3. 打印 forbiddenaggrotags**

```lua
local mob = c_find("pigman")
print(mob.components.combat.forbiddenaggrotags)
-- 期望：{"guard", "INLIMBO"}
```

**4. 测试 HostileToPlayerTest**

```lua
local shadow = c_select()  -- 选中一只暗影怪
local player = ConsoleCommandPlayer()
if shadow.HostileToPlayerTest then
    print("HostileToPlayerTest:", shadow:HostileToPlayerTest(player))
end
```

**5. 改 sanity 模拟疯狂攻击**

```lua
local p = ConsoleCommandPlayer()
p.components.sanity:SetPercent(0)  -- 拉到最低
-- 这时所有暗影生物对我都"可打"了
```

**6. 自定义 ShouldAggro 调试日志**

```lua
local mob = c_find("pigman")
local old_aggro = mob.components.combat.ShouldAggro
mob.components.combat.ShouldAggro = function(self, target, ignore_forbidden)
    local result = old_aggro(self, target, ignore_forbidden)
    print(self.inst, "ShouldAggro", target, result)
    return result
end
```

——临时 hook 看哪些目标被仇恨/被拒。

---

### 本节涉及的标签

| 标签 | 含义 | 作用 |
|---|---|---|
| `_combat` | "本实体有 combat 组件" | `IsValidTarget` 中 `target.replica.combat ~= nil` 隐含检查 |
| `spawnprotection` | "出生保护"（玩家复活短暂） | `IsValidTarget` 拒绝（不能被攻击） |
| `shadow` | "暗影类型"（通用） | `IsValidTarget` 中无 sanity 攻击者 + 非疯狂 → 看不见 |
| `playerghost` | "玩家鬼魂" | `IsValidTarget` 中清醒/无 sanity 攻击者 → 看不见；`CanBeAttacked` 拒绝 |
| `flight` | "飞行（不可被打）" | `CanBeAttacked` 直接拒绝 |
| `noattack` | "无敌" | `CanBeAttacked` 拒绝（除非 iframeskeepaggro） |
| `invisible` | "隐身" | `CanBeAttacked` + `CanTarget` 拒绝（除非 iframeskeepaggro） |
| `noplayertarget` | "玩家不可选为目标" | `CanBeAttacked` 拒绝玩家攻击；polly_rogers / shadowwaxwell 等同伴用 |
| `INLIMBO` | "在 limbo（容器/隐藏）" | `CanTarget` 拒绝 |
| `notarget` | "不可被选为目标" | `CanTarget` 拒绝；`TryRetarget` 跳过 |
| `debugnoattack` | "调试期间禁用" | `CanTarget` 拒绝 |
| `stealth` | "潜行" | `ShouldAggro` 拒绝；玩家攻击 `ValidateAttackTarget` 也拒绝 |
| `hostile` | "敌对状态" | `ShouldAggro` 中绕过"打不死 + 非玩家相关"的拒绝逻辑 |
| `alwayshostile` | "始终敌对" | `CanBeAlly` 拒绝（不会被视为盟友）；玩家跟随者保护跳过 |
| `propweapon` | "道具武器"（玩具锤等） | `IsValidTarget` + `CanBeAttacked` 中允许 PVP 关闭时玩家互打 |
| `crazy` | "疯狂"状态 | `CanBeAttacked` 中允许打几乎所有东西，包括暗影生物 |
| `shadowcreature` / `nightmarecreature` | 暗影/噩梦生物 | `CanBeAttacked` 中受多层保护（除非攻击者疯狂 / 已仇恨 / HostileToPlayerTest） |
| `shadowsubmissive` | "可被支配的暗影"（state tag/普通 tag） | 配合 `HostileToPlayerTest` 钩子，对玩家半透明 |
| `shadowdominance` | "支配暗影"（玩家） | `HostileToPlayerTest` 中让暗影对该玩家不敌对 |
| `bedazzled` / `companion` / `domesticated` / `saltlicker_salted` | 玩家阵营标签 | `CanBeAlly` 中决定盟友身份 |
| `player_damagescale` | "适用玩家伤害折扣" | `CanApplyPlayerDamageMod` 把非玩家视为玩家（如 wendy 的鬼姐姐） |
| `player` | "属于玩家"（玩家原型基础标签） | 与 `isplayer` 字段配合用于多种判定 |
| `peacefulmount` | "和平坐骑" | `CanTarget` 中限制骑乘攻击 |
| `iframeskeepaggro` | "无敌帧仍保持仇恨"（state tag） | `CanBeAttacked` 中临时绕过 `noattack`/`invisible` 检查 |

### 本节涉及的事件

| 事件名 | 推送对象 | 推送时机 | 数据键值 |
|---|---|---|---|
| `attacked` | 被攻击实体 | `Combat:GetAttacked` 未 blocked 时 | `attacker` / `damage` / `weapon` / `stimuli` / `spdamage` / `redirected` / `noimpactsound` / `original_damage` / `damageresolved`（详 21.8） |
| `blocked` | 被攻击实体 | `Combat:GetAttacked` 被 blocked 时（弹反、零伤害） | `attacker` / `damage` / `spdamage` / `original_damage` |
| `newcombattarget` | 攻击者本身 | `EngageTarget` 锁定新目标后（与 21.3.6 一致） | `target` / `oldtarget` |
| `droppedtarget` | 攻击者本身 | `DropTarget` 真正脱战时 | `target` |
| `losttarget` | 攻击者本身 | `OnUpdate` 中判定失败导致脱战 | 无 |
| `transfercombattarget` | 目标实体 | "仇恨转移"事件 | `newtarget`(Entity\|nil) |
| `leaderchanged` | 目标实体 | follower leader 变化 | （随推送方）—— Combat 用以重新检查 `CanBeAlly` |

> **注**：本节聚焦"判定函数"，事件相对较少。事件的完整列表更多集中在 21.3（流程）和 21.8（伤害结算）。

---



## 21.5 攻击系统：冷却、武器、投射物

（待编写）

## 21.6 Weapon 组件——武器属性与攻击回调

（待编写）

## 21.7 伤害计算：CalcDamage 全公式

（待编写）

## 21.8 受击处理：GetAttacked 全流程

（待编写）

## 21.9 Armor 组件——防具减伤计算与耐久

（待编写）

## 21.10 范围攻击（AOE）

（待编写）

## 21.11 位面伤害与伤害类型系统

（待编写）

### PlanarDamage/PlanarDefense

（待编写）

### PlanarEntity

（待编写）

### DamageTypeBonus/DamageTypeResist

（待编写）

### 位面伤害与普通伤害在CalcDamage中的叠加流程

（待编写）

## 21.12 社会关系与 PVP

（待编写）

## 21.13 辅助系统：战吼、弹反、伤害反射

（待编写）

## 21.14 暗影装备控制——ShadowDominance / ShadowSubmissive / ShadowLevel 机制

（待编写）

## 21.15 Replica 与客户端判定

（待编写）

## 21.16 OnUpdate 与生命周期

（待编写）

## 21.17 与其他系统的集成

（待编写）
