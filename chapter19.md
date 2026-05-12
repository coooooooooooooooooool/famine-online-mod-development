# 第19章 死亡与复活系统

## 19.1 死亡流程：Health:Kill() → 掉落物品 → 生成幽灵

### 本节导读

"死亡"在饥荒里不只是"实体消失"——它是一条**完整的、带有多个阶段的管线**：损伤计算 → 事件推送 → 状态图转换 → 动画播放 → 物品掉落 → 骨架/尸体生成 → 幽灵变换。每个阶段都有明确的代码入口，都可以被 mod 拦截或扩展。

理解这条管线对 mod 开发至关重要：
- 你想**在玩家死亡时执行自定义逻辑**——需要知道监听哪个事件
- 你想**让某件装备在死亡后保留**——需要知道 `keepondeath` 字段
- 你想**阻止玩家死亡**——需要知道 `redirect` 和 `invincible` 机制
- 你想**自定义死亡产物**（骨架改成别的）——需要知道 `skeleton_prefab` 和 `SpawnDeathProduct`

> **新手**先看 19.1.1-19.1.3——用 5 步流程图理解整个死亡过程、知道物品是在哪一步掉落的、知道骨架/幽灵是什么时候出现的；**进阶读者**继续看 19.1.4-19.1.8，深入 `Health:DoDelta` 的伤害削减机制、`Health:SetVal` 的双事件推送、SGwilson 死亡状态的动画时间轴、`OnPlayerDeath` 和 `OnMakePlayerGhost` 的分叉逻辑；**老手**跳到 19.1.9-19.1.11，掌握 mod 监听死亡事件的正确姿势、`redirect` / `keepondeath` / `skeleton_prefab` 的自定义接口，以及五个常见坑。

读完本节，你能**精确地在死亡流程的任意一个阶段插入 mod 逻辑**，而不是靠猜测哪个地方能用。

---

### 19.1.1 快速入门：死亡流程 5 步全景图

玩家从满血到变成幽灵，要经历以下 5 个阶段：

```
①  Health:DoDelta(amount, ...)
        ↓ 当 health ≤ 0
②  Health:SetVal 推两个事件：
        TheWorld:PushEvent("entity_death", {inst, cause, afflicter, corpsing})
        inst:PushEvent("death", {cause, afflicter, corpsing})
        ↓
③  player_common.lua 监听 "death" → OnPlayerDeath
        记录死因、公告、触发 stategraph
        ↓
④  SGwilson death 状态（onenter）
        停止移动 → DropAllItemsForDeath → 播死亡动画
        ↓（等动画播完，animover 事件）
        ┌─────────────────────┬────────────────────────┐
        │ ghostenabled = true  │ ghostenabled = false    │
        ↓                     ↓
⑤a  PushEvent("makeplayerghost")       ⑤b  PushEvent("playerdied")
        ↓                               ↓
    OnMakePlayerGhost               OnPlayerDied
    [生成骨架→变幽灵]                [淡出→删除实体]
```

**关键区分**：
- `ghostenabled`（`inst.ghostenabled`）是**是否允许玩家进入幽灵状态**的总开关——由 `POPUPFLAGS.GHOSTENABLED` 等世界设置控制。联机版默认开启；荒野模式默认关闭（死亡直接重开）
- `revivablecorpse` 组件——某些特殊情况（如连接到影子寄生虫时）玩家死亡会变成可复活尸体，走另一条分支（进 `"corpse"` 状态）

> **新手记忆**：物品在**第④步（进入 death 状态 onenter 时）**掉落，骨架在**第⑤步（幽灵变换时）**生成。在幽灵变换前，物品已经全部在地上了。

---

### 19.1.2 快速入门：物品掉落——`DropAllItemsForDeath` 与 `Inventory:DropEverything`

物品掉落由 SGwilson death 状态的 `onenter` 调用：

```225:233:scripts/stategraphs/SGwilson.lua
local function DropAllItemsForDeath(inst)
    inst.components.inventory:DropEverything(true)
    if inst.components.socketholder then
        local items = inst.components.socketholder:UnsocketEverything()
        for _, item in ipairs(items) do
            Launch2(item, inst, 1, 1, 0.2, 0, 4)
        end
    end
end
```

两件事：
1. 调 `inventory:DropEverything(true)` 把背包 + 装备槽所有物品弹出
2. 如果有 `socketholder`（饰品插槽），也把那些物品甩出去

`Inventory:DropEverything(ondeath, keepequip)` 的核心逻辑：

```1730:1774:scripts/components/inventory.lua
function Inventory:DropEverything(ondeath, keepequip)
    if self.inst:HasTag("player") and not GetGhostEnabled() and not GetGameModeProperty("revivable_corpse") then
        -- 荒野模式：强制掉所有东西
        ondeath = false
    end
    if self.activeitem ~= nil and not (ondeath and self.activeitem.components.inventoryitem.keepondeath) then
        self:DropItem(self.activeitem, true, true)
        -- ...
    end

    for k = 1, self.maxslots do
        local v = self.itemslots[k]
        if v ~= nil and not (ondeath and v.components.inventoryitem.keepondeath) and not v.components.curseditem then
            if not v.components.inventoryitem.islockedinslot then
                self:DropItem(v, true, true)
            elseif v.components.container then
                -- 锁定槽里的容器，展开其内容物
                -- ...
            end
        end
    end

    if not keepequip then
        -- 装备槽也掉
        for k, v in pairs(self.equipslots) do
            if not (ondeath and v.components.inventoryitem.keepondeath) then
                self:DropItem(v, true, true)
            end
        end
    end
end
```

**三种物品会在死亡时保留（不掉落）**：

| 条件 | 含义 |
|------|------|
| `v.components.inventoryitem.keepondeath = true` | 该物品被标记为"死亡保留"（如 Wanda 的手表）|
| `v.components.curseditem` 存在 | 被诅咒的物品（在荒野模式下也强制保留）|
| `v.components.inventoryitem.islockedinslot = true` + 非容器 | 被锁定在格子里的物品 |

> **新手记忆**：**正常联机服死亡时，背包 + 装备槽的所有物品都会掉落**（除非有 `keepondeath=true`）；荒野模式下，即使有 `keepondeath` 标记，也会强制掉落所有物品（`ondeath` 被强设为 `false`）。

---

### 19.1.3 快速入门：生成遗体产物——骨架、尸体与浅坑

在 `OnMakePlayerGhost` 里，第一件事就是生成遗体产物：

```673:700:scripts/prefabs/player_common_extensions.lua
local function OnMakePlayerGhost(inst, data)
    if inst:HasTag("playerghost") then
        return
    end

    local x, y, z = inst.Transform:GetWorldPosition()
    local death_product_type

    -- Spawn post death item
    if inst.skeleton_prefab ~= nil and data ~= nil and data.skeleton then
        death_product_type = SpawnDeathProduct(inst)
    end

    if data ~= nil and data.loading then
        -- 加载存档时恢复幽灵状态，不生成遗体
        inst.loading_ghost = true
    else
        -- ...死亡特效 die_fx...
        if death_product_type ~= DEATH_PRODUCTS.CORPSE then
            SpawnPrefab("die_fx").Transform:SetPosition(x, y, z)
        end
    end
```

两个条件缺一不可才会生成遗体：
1. `inst.skeleton_prefab ~= nil` —— 玩家 prefab 设置了骨架类型
2. `data.skeleton = true` —— 死亡位置**在可通行的地面上**（从 SGwilson 传入的 `TheWorld.Map:IsPassableAtPoint(...)` 结果）

`SpawnDeathProduct` 决定生成什么：

```104:145:scripts/prefabs/player_common_extensions.lua
local function SpawnDeathProduct(inst)
    -- WX-78 备份身体特例
    if inst.wx78_backupbody_save_inst then
        -- ...（不生成骨架，让备份身体接管）
        return
    end

    local x, y, z = inst.Transform:GetWorldPosition()
    local can_corpse = CanEntityBecomeCorpse(inst)
    if can_corpse then
        -- 生成可复活尸体（game mode 开启了 revivable_corpse 功能）
        local corpse = SpawnPrefab("playercorpse")
        -- ... 复制皮肤、设置描述...
        return DEATH_PRODUCTS.CORPSE
    else
        -- 正常情况：生成骨架或浅坑
        local has_skeletons = TheSim:HasPlayerSkeletons()
        local skel = SpawnPrefab(has_skeletons and inst.skeleton_prefab or "shallow_grave_player")
        if skel ~= nil then
            -- ...设置位置、描述、皮肤...
        end
        return has_skeletons and DEATH_PRODUCTS.SKELETON or DEATH_PRODUCTS.SHALLOW_GRAVE
    end
end
```

**三种遗体产物**：

| 产物 | 生成条件 | 功能 |
|------|----------|------|
| `playercorpse`（尸体）| `CanEntityBecomeCorpse` 返回 true（revivable_corpse 模式）| 可以被其他玩家手动复活 |
| `inst.skeleton_prefab`（骨架）| `TheSim:HasPlayerSkeletons()` 返回 true（PC/Mac） | 装饰性骨架，记录死因、皮肤，可以被幽灵交互 |
| `shallow_grave_player`（浅坑）| 移动设备（没有骨架特性）| 简化版的遗体标记 |

`inst.skeleton_prefab` 通常在各角色 prefab 中赋值，如：

```lua
-- 在角色 fn 里：
inst.skeleton_prefab = "wilson_skeleton"
```

如果玩家死在水里/不可通行的地块上（`data.skeleton = false`），**完全不生成任何遗体产物**。

> **新手记忆**：死在地上 → 生成骨架（PC）或浅坑（手机），然后变成幽灵；死在水里 → 直接变成幽灵，没有遗体。

---

### 19.1.4 进阶：`Health:DoDelta` 的伤害削减机制

在血量真正减少之前，`DoDelta` 有一整条**拦截和削减管线**：

```613:646:scripts/components/health.lua
function Health:DoDelta(amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb)
    local old_percent = self:GetPercent()

    -- 拦截 1：非致死温度/饥饿伤害（blood pct 低于阈值时停止）
    if old_percent <= self.nonlethal_pct and
		(((cause == "cold" or cause == "hot") and self.nonlethal_temperature) or
		(cause == "hunger" and self.nonlethal_hunger)) then
        return 0
    end

    -- 拦截 2：自定义 redirect 函数（返回 true 则完全跳过）
    if self.redirect ~= nil and self.redirect(self.inst, amount, ...) then
        return 0
    -- 拦截 3：无敌状态 + 正在传送
    elseif not ignore_invincible and (self:IsInvincible() or self.inst.is_teleporting) then
        return 0
    elseif amount < 0 and not ignore_absorb then
        -- 伤害削减 1：absorb（吸收率）
        amount = amount * math.clamp(1 - (...absorb...), 0, 1) * math.max(1 - self.externalabsorbmodifiers:Get(), 0)
        -- 伤害削减 2：externalreductionmodifiers（减伤值）
        amount = afflicter ~= nil and math.min(0, amount + self.externalreductionmodifiers:Get()) or amount
    end

    -- 伤害削减 3：deltamodifierfn（自定义修改器）
    if self.deltamodifierfn ~= nil then
        amount = self.deltamodifierfn(self.inst, amount, ...)
    end

    -- 伤害削减 4：maxdamagetakenperhit（单次伤害上限）
    if self.maxdamagetakenperhit ~= nil and amount < self.maxdamagetakenperhit and not self._ignore_maxdamagetakenperhit then
        amount = self.maxdamagetakenperhit
    end

    self:SetVal(self.currenthealth + amount, cause, afflicter)

    -- 事件推送
    self.inst:PushEvent("healthdelta", { oldpercent, newpercent, overtime, cause, afflicter, amount })
    if self.ondelta ~= nil then
        self.ondelta(self.inst, old_percent, self:GetPercent(), overtime, cause, afflicter, amount)
    end
end
```

**六层削减/拦截，按顺序**：

| 层 | 字段/机制 | 作用 |
|---|-----------|------|
| 1 | `nonlethal_pct` + `nonlethal_temperature` / `nonlethal_hunger` | 低血时体温/饥饿伤害不致死（Wilson 默认低于 5% 不扣血）|
| 2 | `health.redirect(inst, amount, ...)` | **完全接管伤害**——用于特殊机制（如月亮印记保护）|
| 3 | `health.invincible` / `inst.is_teleporting` | 无敌/传送时无敌 |
| 4 | `absorb` + `externalabsorbmodifiers` | 百分比吸收（0-1 区间，越高越少受伤）|
| 5 | `deltamodifierfn` | 自定义伤害修改函数 |
| 6 | `maxdamagetakenperhit` | 单次伤害上限（注意值是负数，如 `-100`）|

**`Health:Kill()` vs `Health:ForceKill()`**：

```528:544:scripts/components/health.lua
function Health:Kill()
    if self.currenthealth > 0 then
		self._ignore_maxdamagetakenperhit = true
        self:DoDelta(-self.currenthealth, nil, nil, nil, nil, true)
        -- ignore_invincible = nil（会被无敌拦截！）
    end
end

function Health:ForceKill() -- To bypass invincible
    if self.currenthealth > 0 then
		self._ignore_maxdamagetakenperhit = true
        self:DoDelta(-self.currenthealth, nil, nil, true, nil, true)
        -- ignore_invincible = true（绕过无敌）
    end
end
```

**关键区别**：
- `Kill()` —— 会被 `invincible`（无敌状态）拦截。如果玩家是无敌的，`Kill()` 无效
- `ForceKill()` —— **绕过无敌**，直接杀死。用于剧情触发死亡、传送门惩罚等

---

### 19.1.5 进阶：`Health:SetVal` 的死亡分支——双事件与 nofadeout 机制

当 `currenthealth` 从 > 0 降到 ≤ 0 时，`SetVal` 触发：

```583:610:scripts/components/health.lua
    if old_health > 0 and self.currenthealth <= 0 then
        -- 记录死亡原因
        self.causeofdeath = afflicter or nil
        local is_corpsing = CanEntityBecomeCorpse(self.inst)
        
        -- 推送两个事件（顺序很重要！）
        TheWorld:PushEvent("entity_death", { inst = self.inst, cause = cause, afflicter = afflicter, corpsing = is_corpsing })
        self.inst:PushEvent("death", { cause = cause, afflicter = afflicter, corpsing = is_corpsing })
        
        self.is_corpsing = is_corpsing

        -- 统计系统
        local notify_type = (self.inst.isplayer and "TotalPlayersKilled") or "TotalEnemiesKilled"
        NotifyPlayerProgress(notify_type, 1, afflicter)

        -- 渐出/消失处理（非玩家实体）
        if self:CanFadeOut() then
            self.inst:AddTag("NOCLICK")
            self.inst.persists = false
            self.inst.erode_task = self.inst:DoTaskInTime(self.destroytime or 2, ErodeAway)
        elseif self.is_corpsing then
            -- 等待实体进入睡眠再切换到 corpse 状态
            if self.inst:IsAsleep() then
                OnCorpsingEntitySleep(self.inst)
            else
                self.inst:ListenForEvent("entitysleep", OnCorpsingEntitySleep)
            end
        end
    end
```

**两个事件的区别**：

| 事件 | 推给谁 | 主要监听者 |
|------|--------|-----------|
| `"entity_death"` | `TheWorld`（全局）| 统计系统、任务系统、成就系统 |
| `"death"` | `inst`（死亡实体）| `player_common.lua` 的 `OnPlayerDeath`、combat 组件、各种 mod |

**为什么先推 world 事件**？注释说：
> "Push world event first, because the entity event may invalidate itself"

即实体的 `"death"` 事件处理可能把实体本身 Remove 掉（比如 `nofadeout` 物品在死亡时调用 `:Remove()`），导致 world 事件没机会触发。先推 world 事件更安全。

**`nofadeout`** 字段：
- `health.nofadeout = false`（默认）→ 死亡后实体会**自动 2 秒后淡出消失**（`ErodeAway`）
- `health.nofadeout = true` → **不自动淡出**，由事件监听者自己控制（常见于玩家 prefab、有特殊死亡动画的 boss）

---

### 19.1.6 进阶：SGwilson death 状态——动画时间轴与掉落时机

死亡状态机是死亡流程的**心脏**——物品掉落、动画播放、幽灵生成全在这里：

```3568:3652:scripts/stategraphs/SGwilson.lua
State{
    name = "death",
    tags = { "busy", "dead", "pausepredict", "nomorph" },

    onenter = function(inst, data)
        assert(inst.deathcause ~= nil, "Entered death state without cause.")

        ClearStatusAilments(inst)  -- 清除冻结/着火/潮湿等状态
        ForceStopHeavyLifting(inst)

        inst.components.locomotor:Stop()
        inst.components.locomotor:Clear()
        inst:ClearBufferedAction()

        if inst.components.rider:IsRiding() then
            -- 骑乘时：先播摔落动画
            inst.AnimState:PlayAnimation("fall_off")
            inst.sg:AddStateTag("dismounting")
        else
            -- 播死亡音效（角色专属音 + 通用死亡音）
            inst.SoundEmitter:PlaySound("dontstarve/wilson/death")
            inst.SoundEmitter:PlaySound(...prefab.."/death_voice")

            -- 特殊分支：vine_save（维诺娜跳绳被查理抓走）、wx78_backupbody_save、revivablecorpse
            -- 正常路径：
            DropAllItemsForDeath(inst)
            inst.AnimState:PlayAnimation(inst.deathanimoverride or "death")
        end

        inst.components.burnable:Extinguish()  -- 熄灭身上的火
        inst.components.playercontroller:Enable(false)  -- 禁止输入
        inst.sg:ClearBufferedEvents()
    end,
```

**物品掉落发生在 `onenter` 里**——即进入死亡状态**的第一帧**就掉了，不是等动画播完。

动画播完后（`animover` EventHandler），根据模式分叉：

```3724:3800:scripts/stategraphs/SGwilson.lua
    events = {
        EventHandler("animover", function(inst)
            if inst.AnimState:AnimDone() then
                local skeleton = TheWorld.Map:IsPassableAtPoint(inst.Transform:GetWorldPosition())

                -- ...特殊情况处理...

                if inst.components.revivablecorpse ~= nil then
                    inst.sg:GoToState("corpse")     -- 尸体模式
                elseif inst.ghostenabled then
                    -- 各角色专属死亡处理
                    inst.components.cursable:Died()
                    -- 检查 wonkey 变回人、WX-78 底盘还原等
                    inst:PushEvent("makeplayerghost", { skeleton = skeleton })
                else
                    inst:PushEvent("playerdied", { skeleton = skeleton })
                end
            end
        end),
    },
```

`skeleton` 参数 = `TheWorld.Map:IsPassableAtPoint(x, y, z)`——**当前位置是否可通行**。如果死在水里，这里是 `false`，`OnMakePlayerGhost` 里就不会生成骨架。

---

### 19.1.7 进阶：`OnPlayerDeath` 与 `OnMakePlayerGhost` 的职责分工

这是最容易混淆的两个函数：

| 函数 | 触发时机 | 职责 |
|------|----------|------|
| `OnPlayerDeath`（`player_common_extensions.lua:192`）| `"death"` 事件 → 即 **血量变为 0 的那一帧** | 记录死因、公告死亡、存档会话删除（荒野模式） |
| `OnMakePlayerGhost`（`player_common_extensions.lua:673`）| `"makeplayerghost"` 事件 → 即 **死亡动画播完后** | 生成骨架、变幽灵外观、切换状态图、初始化幽灵属性 |

`OnPlayerDeath` 的核心工作（`player_common_extensions.lua:192-265`）：

```192:265:scripts/prefabs/player_common_extensions.lua
local function OnPlayerDeath(inst, data)
    if inst:HasTag("playerghost") then
        return  -- 幽灵不能再死
    end
    -- ...（各角色特殊技能检查，如维诺娜藤蔓救援、WX备份身体）

    inst:PushEvent("ms_closepopups")  -- 关闭所有弹窗

    -- 记录死亡信息到 inst 字段（后续 OnMakePlayerGhost 会用）
    inst.deathclientobj = TheNet:GetClientTableForUser(inst.userid)
    inst.deathcause = data ~= nil and data.cause or "unknown"
    inst.last_death_position = Vector3(inst.Transform:GetWorldPosition())
    inst.last_death_shardid = TheShard:GetShardId()

    -- 处理 PK 信息（是谁杀死的玩家）
    -- inst.deathpkname = ...（杀手名字，如果是玩家）

    if not (inst.ghostenabled or inst.components.revivablecorpse or inst.charlie_vinesave) then
        if inst.deathcause ~= "file_load" then
            -- 公告死亡
            inst.player_classified:AddMorgueRecord()
            TheNet:AnnounceDeath(announcement_string, inst.entity)
        end
        -- 荒野模式：提前删除会话（如果客户端断线，不会留下游魂）
        DeleteUserSession(inst)
    end
end
```

`OnMakePlayerGhost` 在动画播完后做的事（`player_common_extensions.lua:673-747`）：

```632:671:scripts/prefabs/player_common_extensions.lua
local function CommonPlayerDeath(inst)
    -- 注意：这个函数在 OnMakePlayerGhost 和 OnMakePlayerCorpse 里都会调
    inst.player_classified.MapExplorer:EnableUpdate(false)  -- 停止地图更新
    inst:RemoveComponent("burnable")
    inst:RemoveComponent("freezable")
    inst:RemoveComponent("propagator")
    inst:RemoveComponent("grogginess")
    inst:RemoveComponent("slipperyfeet")

    inst.components.moisture:ForceDry(true, inst)
    inst.components.sheltered:Stop()
    inst.components.debuffable:Enable(false)

    inst.components.age:PauseAging()
    inst.components.health:SetInvincible(true)  -- 幽灵不能再被击杀
    inst.components.health.canheal = false

    -- 死亡时精神恢复到 50%，饥饿恢复到 2/3
    inst.components.sanity:SetPercent(.5, true)
    inst.components.sanity.ignore = true
    inst.components.hunger:SetPercent(2 / 3, true)
    inst.components.hunger:Pause()
    inst.components.temperature:SetTemp(TUNING.STARTING_TEMP)
    inst.components.frostybreather:Disable()
end
```

**`CommonPlayerDeath` 的重要工作**：
- 精神重置为 **50%**（幽灵状态下精神不变）
- 饥饿重置为 **2/3**（幽灵状态下饥饿不消耗）
- 血量设置为无敌（`SetInvincible(true)`）
- 移除会受到环境影响的组件（`burnable`、`freezable` 等）
- 暂停老化

---

### 19.1.8 进阶：`OnMakePlayerGhost` 的幽灵变换详解

```673:747:scripts/prefabs/player_common_extensions.lua
local function OnMakePlayerGhost(inst, data)
    -- ...（防重复执行检查）...

    -- 1. 生成遗体产物（骨架/尸体）
    if inst.skeleton_prefab ~= nil and data ~= nil and data.skeleton then
        death_product_type = SpawnDeathProduct(inst)
    end

    -- 2. 播放死亡特效
    SpawnPrefab("die_fx").Transform:SetPosition(x, y, z)

    -- 3. 切换外观到幽灵
    inst.AnimState:SetBank("ghost")
    inst.components.skinner:SetSkinMode("ghost_skin")
    inst.components.bloomer:PushBloom("playerghostbloom", "shaders/anim_bloom_ghost.ksh", 100)
    inst.AnimState:SetLightOverride(TUNING.GHOST_LIGHT_OVERRIDE)

    -- 4. 切换状态图到幽灵状态图
    inst:SetStateGraph("SGwilsonghost")

    -- 5. 设置幽灵光源参数
    inst.Light:SetIntensity(.6)
    inst.Light:SetRadius(.5)
    inst.Light:SetFalloff(.6)
    inst.Light:SetColour(180/255, 195/255, 225/255)
    inst.Light:Enable(true)
    inst.DynamicShadow:Enable(false)

    -- 6. 通用死亡状态重置
    CommonPlayerDeath(inst)

    -- 7. 物理设置（幽灵可以穿过障碍物）
    MakeGhostPhysics(inst, 1, .5)
    inst.Physics:Teleport(x, y, z)  -- 确保位置正确

    -- 8. 添加幽灵标签，更新网络标志
    inst:AddTag("playerghost")
    inst.Network:AddUserFlag(USERFLAGS.IS_GHOST)

    -- 9. 重置幽灵的血量（有复活乘数加成的角色此处生效）
    inst.components.health:SetCurrentHealth(TUNING.RESURRECT_HEALTH * (inst.resurrect_multiplier or 1))

    -- 10. 重新启用输入，配置幽灵移动速度和可执行动作
    inst.components.playercontroller:Enable(true)
    inst.player_classified:SetGhostMode(true)
    ConfigureGhostLocomotor(inst)
    ConfigureGhostActions(inst)

    -- 11. 推送 ms_becameghost 事件（通知全局）
    inst:PushEvent("ms_becameghost")

    -- 12. 记录死亡档案、序列化会话
    inst.player_classified:AddMorgueRecord()
    SerializeUserSession(inst)
end
```

**14 步变换，换装换物理换状态图一气呵成**。

**幽灵血量**：`TUNING.RESURRECT_HEALTH * (inst.resurrect_multiplier or 1)`——这就是复活后的初始血量（通常是 50 HP），也受角色专属乘数影响（如威尔逊有胡子加成）。

---

### 19.1.9 老手：mod 钩子——监听死亡事件的正确姿势

#### 钩子 1：监听 `"death"` 事件（最早触发）

在玩家 prefab 的 `fn`（通过 `AddClassPostConstruct` 注入）里监听：

```lua
-- modmain.lua
AddClassPostConstruct("prefabs/wilson", function(inst)
    -- 已经是玩家实例，在这里加监听
end)

-- 更通用的方式：监听所有玩家
-- 在 player_common.lua 的玩家创建流程里加入
-- 实际操作：用 AddPlayerPostInit
AddPlayerPostInit(function(inst)
    inst:ListenForEvent("death", function(inst, data)
        print("玩家死亡！原因:", data.cause, "凶手:", data.afflicter)
    end)
end)
```

**时机**：血量刚变 0 的那一帧，物品还没掉落，动画还没播。

#### 钩子 2：监听 `"entity_death"` 全局事件

```lua
-- 监听任何实体死亡（包括怪物、NPC）
TheWorld:ListenForEvent("entity_death", function(world, data)
    if data.inst:HasTag("player") then
        print("玩家死了:", data.inst)
    end
end)
```

#### 钩子 3：监听 `"makeplayerghost"` 事件（死亡动画播完后）

```lua
AddPlayerPostInit(function(inst)
    inst:ListenForEvent("makeplayerghost", function(inst, data)
        print("即将变成幽灵! 位置可通行:", data.skeleton)
    end)
end)
```

**时机**：死亡动画播完后，骨架刚生成/尚未生成，幽灵变换刚开始。

#### 钩子 4：监听 `"ms_becameghost"` 事件（幽灵变换完成后）

```lua
AddPlayerPostInit(function(inst)
    inst:ListenForEvent("ms_becameghost", function(inst)
        print("已变成幽灵！playerghost tag:", inst:HasTag("playerghost"))
        -- 此时 inst:HasTag("playerghost") == true
        -- 可以安全地修改幽灵属性
    end)
end)
```

**时机**：幽灵变换**完全完成**后——此时所有标签已加、状态图已切换、外观已变，是最安全的"玩家变成幽灵了"的信号。

---

### 19.1.10 老手：自定义死亡行为的三个接口

#### 接口 1：`health.redirect` —— 拦截伤害

```lua
-- 示例：给玩家一个"死亡拦截"效果，一次免死
inst.components.health.redirect = function(inst, amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb)
    if amount < 0 and inst.components.health:GetPercent() + amount / inst.components.health.maxhealth <= 0 then
        -- 致命伤害！触发免死逻辑
        inst.components.health:SetPercent(0.1)  -- 恢复到 10% 血量
        inst.components.health.redirect = nil   -- 移除免死，只能用一次
        SpawnPrefab("statue_transition_2").Transform:SetPosition(inst.Transform:GetWorldPosition())
        return true  -- 返回 true = 本次伤害被完全消耗
    end
    return false  -- 返回 false = 正常处理
end
```

**注意**：`redirect` 在 `invincible` 检查**之前**执行——即使玩家无敌，`redirect` 也会被调用。

#### 接口 2：`inventoryitem.keepondeath` —— 死亡保留物品

```lua
-- 让某件物品在死亡时不掉落
local item = SpawnPrefab("my_special_item")
item.components.inventoryitem.keepondeath = true
```

**荒野模式例外**：在 `DropEverything(ondeath)` 里，当服务器没有 `GhostEnabled` 且没有 `revivable_corpse` 时，`ondeath` 被强制设为 `false`，导致 `keepondeath` 失效。mod 给玩家的"灵魂绑定"物品在荒野模式下**无法保证保留**。

#### 接口 3：`inst.skeleton_prefab` —— 自定义遗体产物

自定义角色 mod 通常在角色 fn 里设置：

```lua
-- 在角色 fn 里（通过 AddCharacter 或直接修改）
inst.skeleton_prefab = "wilson_skeleton"  -- 使用 Wilson 的骨架
-- 或者
inst.skeleton_prefab = "my_custom_skeleton"  -- 使用自定义骨架
```

如果不设置 `skeleton_prefab`（或设为 `nil`），死亡时**不会生成任何遗体产物**。这可以用于特殊角色（如机器人）的死亡效果。

---

### 19.1.11 老手：五个常见坑

#### 坑 1：监听 `"death"` 事件时判断玩家 tag 错误

**症状**：在 `"death"` 事件回调里调用 `inst.components.health:SetPercent(1)` 无效。

**原因**：`"death"` 事件推送时，`SetVal` 已经把血量设为 0，此时 `IsDead()` 返回 true。调 `SetPercent` 确实会设值，但这会触发 `oncurrenthealth` 回调，进而触发另一轮 `death` 事件——导致**无限循环**！

**修复**：在监听器里使用**延迟执行**（`DoTaskInTime(0, fn)`）来在下一帧恢复血量；或者在 `DoDelta` 之前拦截（使用 `redirect`）而不是事后补救。

#### 坑 2：`ForceKill` 没有效果——实体没有 `health` 组件

**症状**：调用 `inst.components.health:ForceKill()` 报错 `attempt to index nil`。

**原因**：不是所有实体都有 `health` 组件——树、草等环境 prefab 通常只有 `workable` 组件，没有 `health`。

**修复**：先检查 `if inst.components.health ~= nil then`。

#### 坑 3：`keepondeath` 在服务端无效——设置时机错误

**症状**：`item.components.inventoryitem.keepondeath = true` 设了，死亡时物品还是掉了。

**原因**：在客户端设置了 `keepondeath`，但 `DropEverything` 在**服务端**执行——客户端的修改不会自动同步到服务端的物品组件。

**修复**：在服务端的 prefab fn 里直接设置，或者在服务端通过 RPC 触发设置；不要在客户端代码里修改物品组件数据。

#### 坑 4：自定义死亡逻辑在存档加载时重复触发

**症状**：玩家上次死亡后关闭游戏，重新加载存档时触发了死亡事件的 mod 回调。

**原因**：存档记录了玩家处于幽灵状态，加载时会调用 `OnMakePlayerGhost(inst, { loading = true })`。`data.loading = true` 是区分"真死亡"和"加载恢复幽灵状态"的标志。

**修复**：在 `"makeplayerghost"` 事件回调里检查 `data.loading`：

```lua
inst:ListenForEvent("makeplayerghost", function(inst, data)
    if data.loading then return end  -- 加载恢复，不执行死亡逻辑
    -- 真死亡逻辑...
end)
```

#### 坑 5：`deathcause` 是字符串，不是实体

**症状**：`data.cause == pig` 不生效，死因判断总是 false。

**原因**：`deathcause` / `data.cause` 是**字符串**（如 `"fire"`, `"cold"`, `"hunger"`, `"unknown"`），不是杀死玩家的实体。杀手实体是 `data.afflicter`。

```lua
inst:ListenForEvent("death", function(inst, data)
    print("死因字符串:", data.cause)         -- "fire" / "cold" / "unknown" / ...
    print("凶手实体:", data.afflicter)       -- 可能是 nil（环境死亡）或 EntityScript
    if data.afflicter and data.afflicter:HasTag("player") then
        print("被玩家 PK！")
    end
end)
```

---

### 19.1 小结

```
伤害管线（DoDelta）：
  redirect → invincible/teleporting → absorb → deltamodifierfn → maxdamagetakenperhit → SetVal

死亡管线（SetVal 触发）：
  ① PushEvent("entity_death") → TheWorld（全局监听者）
  ② PushEvent("death")        → inst（OnPlayerDeath）
          ↓
  OnPlayerDeath：记录死因、公告、删会话（荒野）
          ↓（等 SGwilson "death" 状态动画播完）
  ③ DropAllItemsForDeath：inventory:DropEverything(true)
     [keepondeath 物品保留，curseditem 保留，水中不生成骨架]
          ↓
  ④ ghostenabled == true → PushEvent("makeplayerghost")
     OnMakePlayerGhost：生成骨架 → die_fx → 切外观/状态图 → CommonPlayerDeath
                       → 加 playerghost tag → PushEvent("ms_becameghost")
     
     ghostenabled == false → PushEvent("playerdied") → OnPlayerDied → 淡出删除

mod 钩子时机：
  "death"         → 血量变0那一帧（物品还在背包，animate未播）
  "makeplayerghost" → 动画播完，骨架生成/变幽灵前
  "ms_becameghost"  → 幽灵变换完成（最安全的"玩家成为幽灵"信号）
  data.loading == true → 存档加载恢复幽灵，不是真死亡
```

**新手核心三句**：物品掉落在**进入 death 状态的瞬间**（不是动画播完），骨架生成在**动画播完后的幽灵变换时**；`ghostenabled` 控制死亡是变幽灵还是直接消失；死在水里不生成骨架。

**进阶核心三句**：`DoDelta` 有 6 层伤害拦截，`Kill()` 被无敌拦截而 `ForceKill()` 不会；`"death"` 事件先推给 `TheWorld` 再推给实体，防止实体自我销毁导致 world 事件丢失；`CommonPlayerDeath` 统一重置精神/饥饿/体温并移除会被环境伤害的组件。

**老手核心三句**：`health.redirect` 是最优雅的免死拦截接口，在 invincible 之前执行；`keepondeath` 在荒野模式下失效；监听 `"makeplayerghost"` 时必须检查 `data.loading` 区分加载恢复与真死亡。


## 19.2 幽灵状态：PlayerGhost 的交互限制与能力

### 本节导读

19.1 讲了**如何变成幽灵**——本节讲**变成幽灵之后发生了什么**。

幽灵状态（`playerghost` tag）是饥荒联机版独有的"软重生"机制：玩家不是立刻重开游戏，而是**继续在游戏世界里以透明身体漂浮**，能够行走、闹鬼、等待复活。这个状态有严格的交互白名单——大部分普通动作被禁止，只有幽灵专属行为可以执行。

理解幽灵状态对 mod 开发的意义：
- 你想**给幽灵加新的交互行为**——需要知道 `ghost_valid` 和 `ghost_exclusive` 字段的用法
- 你想**屏蔽幽灵对某些物体的骚扰**——需要知道 `hauntable` 组件
- 你想**在活着的玩家侧感知幽灵附近**——需要知道精神消耗公式
- 你想**监控玩家进入/离开幽灵状态**——需要知道 `ms_becameghost` / `ghostdissipated` / `respawnfromghost` 事件

> **新手**先看 19.2.1-19.2.3——用一张表看清幽灵能做什么不能做什么、了解幽灵的移动特性（穿墙/穿障碍物）、知道幽灵状态如何结束（复活或消散）；**进阶读者**继续看 19.2.4-19.2.6，深入 `GhostActionFilter` 机制与 `ghost_valid` / `ghost_exclusive` 字段、SGwilsonghost 的全部状态、幽灵对活着玩家的精神消耗公式；**老手**跳到 19.2.7-19.2.9，掌握幽灵的物理碰撞配置、mod 给幽灵添加专属行为的接口、常见的幽灵相关 mod 坑。

---

### 19.2.1 快速入门：幽灵能做什么——ghost_valid 行为白名单

幽灵的行为限制通过 **Action 过滤器**实现。`OnMakePlayerGhost` 会调用 `ConfigureGhostActions`：

```66:70:scripts/prefabs/player_common_extensions.lua
local function ConfigureGhostActions(inst)
    if inst.components.playeractionpicker ~= nil then
		inst.components.playeractionpicker:PushActionFilter(GhostActionFilter, ACTION_FILTER_PRIORITIES.ghost)
    end
end
```

过滤函数本身非常简洁：

```55:58:scripts/prefabs/player_common_extensions.lua
local function GhostActionFilter(inst, action)
    return action.ghost_valid
end
```

**所有 Action 只要 `ghost_valid = false`，就对幽灵不可见**。查阅 `scripts/actions.lua` 中带有 `ghost_valid=true` 的动作：

| 动作 | ghost_exclusive | 含义 |
|------|-----------------|------|
| `LOOKAT` | 否 | 检视实体（仅观察，不做任何事）|
| `WALKTO` | 否 | 移动到某个位置 |
| `JUMPIN` | 否 | 进入虫洞 |
| `JUMPIN_MAP` | 否 | 从地图跳进虫洞 |
| `HAUNT` | **是** | 骚扰（闹鬼）实体 |
| `REMOTERESURRECT` | **是** | 使用复活道具自我复活 |
| `MIGRATE` | 否 | 传送到其他分片（地下等）|

**`ghost_exclusive = true`** 意味着这个动作**只有幽灵才能执行**——活着的玩家看不到这个选项。

**没有 ghost_valid 的动作（幽灵完全无法做）**：
- 拾取物品（`PICKUP`）
- 攻击（`ATTACK`）——stategraph 里有明确的"ignore"处理
- 合成（`CRAFT`）
- 使用（`USEITEM`）、装备（`EQUIP`）
- 喂食（`FEED`）、采摘（`PICK`）、砍伐（`CHOP`）……等所有普通交互

> **新手记忆**：幽灵能做的事只有 **7 种**：看（LOOKAT）、走（WALKTO）、进虫洞（JUMPIN/MAP/MIGRATE）、闹鬼（HAUNT）、用复活道具（REMOTERESURRECT）。无法捡东西、攻击或与大多数实体交互。

---

### 19.2.2 快速入门：幽灵的移动特性——穿越障碍物的物理配置

幽灵的物理碰撞体积由 `MakeGhostPhysics` 设置：

```476:489:scripts/standardcomponents.lua
function MakeGhostPhysics(inst, mass, rad)
    local phys = inst.entity:AddPhysics()
    phys:SetMass(mass)
    phys:SetFriction(0)
    phys:SetDamping(5)
    phys:SetCollisionGroup(COLLISION.CHARACTERS)
	phys:SetCollisionMask(
		TheWorld:CanFlyingCrossBarriers() and COLLISION.GROUND or COLLISION.WORLD,
		--COLLISION.OBSTACLES,  ← 注意这行被注释掉了！
		COLLISION.CHARACTERS,
		COLLISION.GIANTS
	)
    phys:SetCapsule(rad, 1)
    return phys
end
```

**关键**：碰撞遮罩里**没有 `COLLISION.OBSTACLES`**——幽灵不与障碍物（树、建筑、石头）碰撞，可以直接**穿过**它们。

玩家幽灵参数为 `MakeGhostPhysics(inst, 1, .5)`：
- `mass = 1`——正常质量，会被 GIANTS（boss 类）推开
- `rad = 0.5`——碰撞半径（比普通玩家的 0.3 略大，更容易触发角色间碰撞）
- `friction = 0` + `damping = 5`——零摩擦但高阻尼，使移动流畅且不会滑溜

幽灵**还是会**和其他玩家、巨型生物发生碰撞（`COLLISION.CHARACTERS + COLLISION.GIANTS`）。

幽灵的移动速度与普通玩家相同（由 `ConfigureGhostLocomotor` 设置）：

```40:49:scripts/prefabs/player_common_extensions.lua
local function ConfigureGhostLocomotor(inst)
    inst.components.locomotor:SetSlowMultiplier(0.6)
    inst.components.locomotor.pathcaps = { player = true, ignorecreep = true }
    inst.components.locomotor.walkspeed = TUNING.WILSON_WALK_SPEED -- 4
    inst.components.locomotor.runspeed = TUNING.WILSON_RUN_SPEED   -- 6
    inst.components.locomotor.fasteronroad = false  -- 幽灵不因道路加速
    inst.components.locomotor:SetTriggersCreep(false)  -- 不触发蜘蛛怪绒
    inst.components.locomotor:SetAllowPlatformHopping(false)  -- 不能跳跃平台间
end
```

与活着时的差异：
- **不因道路加速**（`fasteronroad = false`）
- **不触发蜘蛛蛛网**（`SetTriggersCreep(false)`）
- **不能在船板间跳跃**（`SetAllowPlatformHopping(false)`）

> **新手记忆**：幽灵可以穿越树木、建筑等障碍物，但速度和活着时相同（4/6）；不因道路加速；不能跳上船。

---

### 19.2.3 快速入门：幽灵状态的结束——复活与消散

幽灵状态有两种结束方式：

| 结束方式 | 触发 | 结果 |
|----------|------|------|
| **复活** | 使用复活道具 / 其他玩家复活 | 变回活人 |
| **消散（dissipate）** | 特殊情形（血量耗尽 / 游戏模式计时）| 彻底"死亡"，退出游戏 |

**复活路径**：
```
幽灵使用 REMOTERESURRECT 动作
    → SGwilsonghost.lua: "remoteresurrect" 状态
    → 动画播完后调 PerformBufferedAction()
    → PushEvent("respawnfromghost", {source = item})
    → OnRespawnFromGhost 处理复活流程
```

**消散路径**：
```
特殊情形触发 → SGwilsonghost: "dissipate" 状态
    → 动画播完 → PushEvent("ghostdissipated")
    → player_common.lua 监听 → OnPlayerDied
    → FadeOutDeadPlayer → RemoveDeadPlayer → 删除实体
```

通常情况下，幽灵**不会自然消散**——联机版的幽灵可以无限漂浮等待复活。`dissipate` 状态更多用于复活动画（`remoteresurrect` 状态也会播放 `dissipate` 动画，但走的是复活路径）。

> **新手记忆**：正常联机游玩，幽灵会一直漂浮直到被复活；使用复活道具（告密的心、肉雕像等）触发 `REMOTERESURRECT` 动作，走 `respawnfromghost` 路径回到活人状态。

---

### 19.2.4 进阶：SGwilsonghost 的完整状态集

幽灵专用状态图 `SGwilsonghost.lua` 包含以下状态：

```18:31:scripts/stategraphs/SGwilsonghost.lua
local actionhandlers =
{
    ActionHandler(ACTIONS.HAUNT, "haunt_pre"),
    ActionHandler(ACTIONS.JUMPIN, "jumpin_pre"),
    ActionHandler(ACTIONS.JUMPIN_MAP, "jumpin_pre"),
    ActionHandler(ACTIONS.ATTACK,
        function()
            --dummy handler in case any attack controls came through network
            print("Player ghost ignored attack control")
        end),
    ActionHandler(ACTIONS.REMOTERESURRECT, "remoteresurrect"),
    ActionHandler(ACTIONS.MIGRATE, "migrate"),
}
```

**6 个 ActionHandler**：
- `HAUNT` → 进 `haunt_pre` 状态
- `JUMPIN` / `JUMPIN_MAP` → 进 `jumpin_pre` 状态（准备进入虫洞）
- `ATTACK` → 被忽略（打印警告，不执行）
- `REMOTERESURRECT` → 进 `remoteresurrect` 状态
- `MIGRATE` → 进 `migrate` 状态（传送到其他分片）

**全部状态列表**：

| 状态 | tags | 作用 |
|------|------|------|
| `idle` | `idle, canrotate` | 漂浮静止 |
| `run` | `moving, running, canrotate, autopredict` | 漂浮移动 |
| `appear` | `nopredict` | 变成幽灵的出现动画 + 嚎叫声 |
| `haunt_pre` | `doing, busy` | 闹鬼预备动画（dissipate 动画 + 音效）|
| `haunt` | `doing, busy, nopredict` | 闹鬼动画（appear 动画）|
| `remoteresurrect` | `doing, busy` | 使用复活道具——dissipate 动画 + 淡出 |
| `hit` | `busy, pausepredict` | 被击中反应（播 hit 动画，但不受伤）|
| `dissipate` | `busy, pausepredict` | 消散动画（最终退出）|
| `talk` | `idle, talking` | 说话动画 |
| `jumpin_pre` | `doing, busy, canrotate` | 进入虫洞准备 |

**`appear` 状态**（变成幽灵时的第一个状态）：

```123:151:scripts/stategraphs/SGwilsonghost.lua
State{
    name = "appear",
    tags = { "nopredict" },

    onenter = function(inst)
        if inst.loading_ghost then
            inst.sg:GoToState("idle")  -- 加载存档时直接跳 idle，不播出现动画
            return
        end

        inst.AnimState:PlayAnimation("appear")
        if not inst:HasTag("mime") then
            inst.SoundEmitter:PlaySound(
                inst:HasTag("girl") and
                "dontstarve/ghost/ghost_girl_howl" or
                "dontstarve/ghost/ghost_howl"
            )
        end
    end,
```

**关键**：存档加载时恢复幽灵，`inst.loading_ghost = true`，不播出现动画直接进 `idle`。

---

### 19.2.5 进阶：ghost_valid 与 ghost_exclusive 的区别——给幽灵添加新行为

`Action` 类的定义：

```275:296:scripts/actions.lua
Action = Class(function(self, data, ...)
    -- ...
    self.ghost_exclusive = data.ghost_exclusive or false
    self.ghost_valid = self.ghost_exclusive or data.ghost_valid or false
    -- If it's ghost-exclusive, then it must be ghost-valid
end)
```

**两个字段的语义**：

| 字段 | 含义 | 典型用途 |
|------|------|---------|
| `ghost_valid = true` | 幽灵**可以**执行这个动作（活人也行）| `WALKTO`, `JUMPIN`, `LOOKAT` |
| `ghost_exclusive = true` | **只有**幽灵才能执行（自动设 ghost_valid = true）| `HAUNT`, `REMOTERESURRECT` |

**mod 如何给幽灵添加专属交互**：

```lua
-- 在 modmain.lua 里注册新动作
ACTIONS.MY_GHOST_ACT = Action({ ghost_valid = true, ghost_exclusive = true })
ACTIONS.MY_GHOST_ACT.id = "MY_GHOST_ACT"
ACTIONS.MY_GHOST_ACT.str = "特殊幽灵行为"
ACTIONS.MY_GHOST_ACT.fn = function(act)
    -- act.doer = 执行者（幽灵玩家）
    -- act.target = 目标实体
    if act.target.components.my_ghost_target then
        act.target.components.my_ghost_target:OnGhostInteract(act.doer)
        return true
    end
end

-- 在 SGwilsonghost 里添加状态处理器（通过 AddStategraphActionHandler）
AddStategraphActionHandler("wilsonghost", ActionHandler(ACTIONS.MY_GHOST_ACT, "haunt_pre"))
-- 复用 haunt_pre 状态即可——它会播动画然后 PerformBufferedAction
```

**在目标实体上启用这个动作**（通过 `componentactions`）：

```lua
-- 在目标 prefab 的 fn 里
local function my_ghost_target_fn(act)
    if act.doer and act.doer:HasTag("playerghost") then
        return "特殊幽灵交互"  -- 返回字符串 = 显示行为选项
    end
end
inst:AddComponent("my_ghost_target")
-- 注册到 componentactions：
-- AddComponentAction("SCENE", "my_ghost_target", my_ghost_target_fn)
```

---

### 19.2.6 进阶：幽灵对活着玩家的精神消耗

当世界里有幽灵玩家时，**活着的玩家会持续失去精神**：

```562:565:scripts/components/sanity.lua
    self:RecalcGhostDrain()
    local ghost_delta = TUNING.SANITY_GHOST_PLAYER_DRAIN * self.ghost_drain_mult
```

`ghost_drain_mult` 由 `RecalcGhostDrain` 计算：

```456:466:scripts/components/sanity.lua
function Sanity:RecalcGhostDrain()
	if GetGhostSanityDrain(TheNet:GetServerGameMode()) and not self.player_ghost_immune_sources:Get() then
        local num_ghosts = TheWorld.shard.components.shard_players:GetNumGhosts()
        local num_alive = TheWorld.shard.components.shard_players:GetNumAlive()
        local group_resist = num_alive > num_ghosts and 1 - num_ghosts / num_alive or 0

        self.ghost_drain_mult = math.min(num_ghosts, TUNING.MAX_SANITY_GHOST_PLAYER_DRAIN_MULT) * (1 - group_resist * group_resist)
    else
        self.ghost_drain_mult = 0
    end
end
```

**公式解析**：

```
基础消耗率 = SANITY_GHOST_PLAYER_DRAIN = -100/(night_time*30)
             ≈ 每天消耗约 1.1% 精神（很慢）

ghost_drain_mult = min(num_ghosts, MAX_MULT) × (1 - group_resist²)
  其中:
  group_resist = max(0, 1 - num_ghosts / num_alive)
  MAX_MULT = 3（最多叠加 3 倍）
```

**几个场景计算**：

| 场景（4人服）| num_ghosts | num_alive | group_resist | ghost_drain_mult |
|-------------|-----------|-----------|--------------|------------------|
| 1人死亡     | 1         | 3         | 0.67         | 0.55             |
| 2人死亡     | 2         | 2         | 0            | 2.0              |
| 3人死亡     | 3         | 1         | 0（alive ≤ ghosts）| 3.0（上限）|

**两个前提条件**：
1. `GetGhostSanityDrain(gamemode)` 返回 true——不是所有游戏模式都有幽灵精神消耗
2. `self.player_ghost_immune_sources:Get()` 为 0——玩家没有精神免疫效果（如洁白护符）

> **进阶记忆**：幽灵多了活人精神损失就多，但有上限（3倍基础）；当活着的人和幽灵数量相等时，幽灵精神消耗最大；特定游戏模式或护符可以免疫此效果。

---

### 19.2.7 老手：幽灵的物理碰撞细节与标签系统

#### 幽灵碰撞遮罩

从 `MakeGhostPhysics` 可以看出幽灵的碰撞遮罩包含：
- `COLLISION.WORLD`（或 `COLLISION.GROUND`）—— 不能出地图边界
- `COLLISION.CHARACTERS` —— 和其他玩家/生物碰撞
- `COLLISION.GIANTS` —— 和 boss 碰撞

**不包含**：
- `COLLISION.OBSTACLES` —— 不和树、建筑、障碍物碰撞
- `COLLISION.SMALLOBSTACLES` —— 不和小障碍物碰撞

这意味着幽灵在 AI 寻路上也有区别——寻路时使用 `pathcaps = { player = true, ignorecreep = true }`，与活人的路径相同（都标记为 player），但由于没有障碍物碰撞，实际上可以穿越物理障碍。

#### 幽灵独有的标签和网络标志

变成幽灵时：
```lua
inst:AddTag("playerghost")
inst.Network:AddUserFlag(USERFLAGS.IS_GHOST)
```

- `"playerghost"` tag —— 用于 AI 判断（很多怪不攻击有此 tag 的实体）、action 过滤（`ATTACK_PROP_CANT_TAGS` 包含 `"playerghost"`，阻止攻击幽灵）、客户端 HUD 判断
- `USERFLAGS.IS_GHOST` —— 网络侧的幽灵标志，用于同步幽灵状态给其他客户端

复活时这两个都会被移除：
```lua
inst:RemoveTag("playerghost")
inst.Network:RemoveUserFlag(USERFLAGS.IS_GHOST)
```

#### `"ATTACK_PROP_CANT_TAGS"` 保护幽灵不被攻击

```8:8:scripts/stategraphs/SGwilson.lua
local ATTACK_PROP_CANT_TAGS = { "flying", "shadow", "ghost", "FX", "NOCLICK", "DECOR", "INLIMBO", "playerghost" }
```

玩家 AI 攻击时会检查 `ATTACK_PROP_CANT_TAGS`——有 `playerghost` tag 的实体**无法被玩家选中攻击**。这是幽灵不被普通攻击伤害的**行为层保护**，比 `invincible` 更早拦截。

---

### 19.2.8 老手：幽灵状态的事件钩子

#### 事件时间轴

```
变成幽灵：PushEvent("ms_becameghost")    ← 幽灵完全初始化后
复活开始：PushEvent("respawnfromghost")  ← 玩家确认使用复活道具时
复活完成：PushEvent("ms_respawnedfromghost", {corpse=false, reviver=source})
消散完成：PushEvent("ghostdissipated")   ← 走 dissipate 状态结束
```

`OnRespawnFromGhost` 的触发流程（`player_common_extensions.lua:569`）：

```569:631:scripts/prefabs/player_common_extensions.lua
local function OnRespawnFromGhost(inst, data)
    if not inst:HasTag("playerghost") then
        return  -- 防止重复触发
    end

    inst:AddTag("reviving")  -- 标记正在复活中

    inst.deathclientobj = nil
    inst.deathcause = nil     -- 清空死亡信息
    inst.deathpkname = nil
    inst.deathbypet = nil
    inst:ShowHUD(false)       -- 隐藏 HUD（复活过渡动画期间）
    inst.components.playercontroller:Enable(false)  -- 禁止输入
    -- ...（根据复活来源选择复活方式）
    -- a. 无来源 → 立刻复活（DoActualRez）
    -- b. 远程复活（remoteresurrect 状态）→ 飞到复活道具位置再复活
    -- c. 普通复活物品（reviver tag）→ 在物品位置复活
    -- d. 否则 → 在死亡位置复活（DoMoveToRezPosition）
end
```

#### mod 监听幽灵相关事件

```lua
AddPlayerPostInit(function(inst)
    -- 变成幽灵
    inst:ListenForEvent("ms_becameghost", function(inst)
        -- 幽灵完全初始化，可以安全修改幽灵属性
        if inst.components.sanity then
            inst.components.sanity.ignore = true  -- 已经是 true，只是演示
        end
    end)

    -- 开始复活（使用了复活道具）
    inst:ListenForEvent("respawnfromghost", function(inst, data)
        print("幽灵开始复活，复活来源:", data and data.source)
    end)

    -- 复活完成（已经变回活人）
    inst:ListenForEvent("ms_respawnedfromghost", function(inst, data)
        print("复活完成！用了尸体复活吗:", data.corpse)
        -- 此时 inst:HasTag("playerghost") == false
    end)

    -- 消散（彻底"死亡"）
    inst:ListenForEvent("ghostdissipated", function(inst)
        -- 此后玩家实体会被删除
    end)
end)
```

---

### 19.2.9 老手：五个常见的幽灵相关 mod 坑

#### 坑 1：在 `ms_becameghost` 之前修改幽灵属性——初始化未完成

**症状**：在 `makeplayerghost` 事件里修改 locomotor 速度，结果被 `ConfigureGhostLocomotor` 覆盖。

**原因**：事件推送顺序是 `makeplayerghost` → `OnMakePlayerGhost`（里面调 `ConfigureGhostLocomotor`）→ `ms_becameghost`。如果在 `makeplayerghost` 回调里修改速度，后续 `ConfigureGhostLocomotor` 会覆盖它。

**修复**：在 `ms_becameghost` 事件里修改——此时 `ConfigureGhostLocomotor` 已经执行完毕。

#### 坑 2：给幽灵的动作判断用了错误的 tag 检查

**症状**：`if inst:HasTag("ghost") then` 返回 false，玩家幽灵没有被识别。

**原因**：玩家幽灵的 tag 是 `"playerghost"`，而不是 `"ghost"`。`"ghost"` 是 `scripts/prefabs/ghost.lua` 里的阿比盖尔/普通幽灵实体用的 tag。

**修复**：`if inst:HasTag("playerghost") then`。

#### 坑 3：用 `health.redirect` 拦截了幽灵的伤害——但幽灵有 invincible

**症状**：给幽灵设置了 `redirect` 函数防止"骚扰伤害"，但实际上没有效果，因为伤害根本没来。

**原因**：幽灵的 `health.invincible = true`——`DoDelta` 在 `redirect` 检查之后马上检查 `invincible`，伤害被直接拦截，`redirect` 永远不会被调用（因为 redirect 在 invincible 检查**之前**执行）。

实际上，幽灵的无敌是在 `invincible` 层拦截的，`redirect` 层确实会先执行。但因为幽灵已经无敌，伤害不会到达 `SetVal`，所以无论怎样都没有死亡风险。

**真正的问题**：开发者可能误以为给幽灵加 redirect 可以阻止某种特殊"穿透无敌"的伤害（ForceKill），这是可能的——`ForceKill` 使用 `ignore_invincible = true`，而 `redirect` 在 invincible 之前检查，所以 `redirect` 能拦截 `ForceKill`。

#### 坑 4：`AddStategraphActionHandler` 的目标状态图名字写错

**症状**：`AddStategraphActionHandler("wilsonghost", ...)` 没有生效，幽灵的新行为不触发。

**原因**：状态图名字是 `"wilsonghost"`（没有 "SG" 前缀），但有些开发者误写成 `"SGwilsonghost"`。

**验证**：`inst.sg.sg.name` 在幽灵状态时为 `"wilsonghost"`（注意大小写）。

#### 坑 5：`ghost_drain_mult` 计算的 `num_ghosts` 是分片全局的

**症状**：mod 里计算幽灵数量和 `sanity.ghost_drain_mult` 不一致。

**原因**：`num_ghosts` 来自 `TheWorld.shard.components.shard_players:GetNumGhosts()`——这是**整个分片**（包括地下、洞穴）的幽灵总数，不只是当前维度的。如果地下有一个幽灵，地上玩家也会受精神消耗影响。

---

### 19.2 小结

```
幽灵状态特征：
  标签：playerghost（玩家端），USERFLAGS.IS_GHOST（网络端）
  物理：无障碍物碰撞（无 COLLISION.OBSTACLES），可穿墙
  无敌：health.invincible = true，health.canheal = false
  状态图：SGwilsonghost（"wilsonghost"）

可执行动作（ghost_valid=true）：
  LOOKAT / WALKTO / JUMPIN / JUMPIN_MAP / MIGRATE（活人也可以）
  HAUNT / REMOTERESURRECT（ghost_exclusive，仅幽灵）

精神消耗（活人受到的影响）：
  ghost_delta = SANITY_GHOST_PLAYER_DRAIN * ghost_drain_mult
  ghost_drain_mult = min(num_ghosts, 3) × (1 - group_resist²)
  group_resist = max(0, 1 - num_ghosts/num_alive)

生命周期事件：
  makeplayerghost（动画播完 → 开始变幽灵）
  ms_becameghost（幽灵初始化完成）
  respawnfromghost（使用了复活道具）
  ms_respawnedfromghost（复活完成，已变回活人）
  ghostdissipated（消散，走 OnPlayerDied 路径）
```

**新手核心三句**：幽灵只有 7 种可用行为（看/走/虫洞/骚扰/用复活道具）；可以穿越障碍物但不能跳船；正常联机游戏中幽灵不会自然消散，只要等待复活即可。

**进阶核心三句**：`GhostActionFilter` 通过 `action.ghost_valid` 决定幽灵可见行为；`ghost_exclusive=true` 是幽灵专属行为（如 HAUNT）；幽灵精神消耗与 num_ghosts/num_alive 比例相关，上限 3 倍基础值。

**老手核心三句**：幽灵 tag 是 `"playerghost"` 不是 `"ghost"`；给幽灵添加新动作需要 `ghost_valid=true` + `AddStategraphActionHandler("wilsonghost", ...)`；`ms_becameghost` 是修改幽灵属性的最安全时机（`makeplayerghost` 时内部初始化未完成）。


## 19.3 Hauntable 组件——幽灵作祟机制详解

### 本节导读

19.2 我们知道幽灵能执行 `HAUNT`（骚扰）动作——但骚扰之后会发生什么？答案由目标实体身上的 `hauntable` 组件决定。

`hauntable` 组件是饥荒里最有趣的系统之一：它用一套**回调 + 冷却 + 特效**的机制，让任何 prefab 都能响应幽灵的骚扰。从篝火被加燃料到墓碑改变碑文，从生物四散奔逃到传送门直接复活——都通过这个组件实现。

**本节讲三件事**：
1. `Hauntable:DoHaunt` 的执行流程——从幽灵点击到效果触发的完整链路
2. 冷却/特效/`hauntvalue` 系统——"骚扰成功度"与防刷机制
3. 标准工厂函数 `MakeHauntableXxx`——批量快速给 prefab 加骚扰行为

> **新手**先看 19.3.1-19.3.3——理解骚扰的触发条件（不能 haunted/catchable）、最重要的特殊值 `HAUNT_INSTANT_REZ`（触碰传送门/触手石复活）、5 行代码给自己的 mod prefab 加骚扰响应；**进阶读者**继续看 19.3.4-19.3.7，深入 `DoHaunt` 的完整执行流程、冷却计时系统与 Shader 闪烁特效、`hauntvalue` 的含义与历史背景、以及标准工厂函数家族的现状（很多已被 #HAUNTFIX 注释掉）；**老手**跳到 19.3.8-19.3.10，掌握 Panic 模式（骚扰引发生物奔逃）、自定义 Shader FX 绑定、五个常见坑。

---

### 19.3.1 快速入门：HAUNT 动作的触发条件

幽灵执行闹鬼动作时，先由 `componentactions.lua` 决定目标是否显示 HAUNT 选项：

```410:414:scripts/componentactions.lua
        hauntable = function(inst, doer, actions)
            if not (inst:HasTag("haunted") or inst:HasTag("catchable")) then
                table.insert(actions, ACTIONS.HAUNT)
            end
        end,
```

**两个阻止显示 HAUNT 的条件**：
1. 实体有 `"haunted"` tag ——表示当前正处于骚扰冷却期
2. 实体有 `"catchable"` tag —— 虫子/蝴蝶等可直接捕捉的实体（给活人用，幽灵不能捕捉）

**`ACTIONS.HAUNT.fn`** 在幽灵点击确认后执行：

```3100:3111:scripts/actions.lua
ACTIONS.HAUNT.fn = function(act)
    if act.target ~= nil and
        act.target:IsValid() and
        not act.target:IsInLimbo() and
        act.target.components.hauntable ~= nil and
        not (act.target.components.inventoryitem ~= nil and act.target.components.inventoryitem:IsHeld()) and
        not (act.target:HasTag("haunted") or act.target:HasTag("catchable")) then
        act.doer:PushEvent("haunt", { target = act.target })
        act.target.components.hauntable:DoHaunt(act.doer)
        return true
    end
end
```

执行步骤：
1. 最终验证（防止从 componentactions 到 action.fn 之间状态变化）
2. 给幽灵（doer）推 `"haunt"` 事件——通知 SGwilsonghost 进入 `haunt` 动画状态
3. 调 `target.components.hauntable:DoHaunt(act.doer)`——执行骚扰效果

> **新手记忆**：有 `hauntable` 组件的实体 + 没有 `haunted` 或 `catchable` tag → 幽灵可以骚扰。骚扰时同时触发幽灵的动画和目标的响应函数。

---

### 19.3.2 快速入门：最重要的特殊闹鬼——`HAUNT_INSTANT_REZ` 触发复活

`HAUNT_INSTANT_REZ = 9999` 是 hauntvalue 的一个特殊标记值，当骚扰成功后的 hauntvalue 等于这个值时，**立刻触发幽灵复活**：

```82:95:scripts/components/hauntable.lua
function Hauntable:DoHaunt(doer)
    if self.onhaunt ~= nil then
        -- ...
        self.haunted = self.onhaunt(self.inst, doer)
        if self.haunted then
            if doer ~= nil then
                if self.hauntvalue == TUNING.HAUNT_INSTANT_REZ and doer:HasTag("playerghost") then
                    doer:PushEvent("respawnfromghost", { source = self.inst })
                end
                -- ...
            end
```

**会触发即时复活的实体**：

1. **传送门（multiplayer_portal）**——被骚扰时 `hauntvalue = HAUNT_INSTANT_REZ`：
   ```lua
   -- scripts/prefabs/multiplayer_portal.lua 第 48 行
   inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
   ```

2. **触手石（resurrectionstone）**——被激活后骚扰：
   ```lua
   -- scripts/prefabs/resurrectionstone.lua 第 91 行
   inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
   ```

3. **WX-78 备份身体（wx78_backupbody）**——死亡后备份身体可以被骚扰复活。

这就是为什么幽灵可以**直接骚扰传送门复活**——本质上就是 hauntvalue == HAUNT_INSTANT_REZ 触发了 `respawnfromghost` 事件。

> **新手记忆**：`HAUNT_INSTANT_REZ = 9999` 是特殊复活触发值；设置了这个值的实体（传送门、触手石等），幽灵骚扰后**立刻复活**，不需要额外的复活道具。

---

### 19.3.3 快速入门：给 mod prefab 加一个骚扰响应——5 行代码

最简单的骚扰响应：

```lua
-- 在 prefab fn 里（服务端侧）
inst:AddComponent("hauntable")

-- 设置骚扰回调
inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
    print("我被骚扰了！骚扰者:", haunter)
    -- 执行效果，比如随机移动
    inst.Transform:SetPosition(
        inst.Transform:GetWorldPosition() + math.random(-2, 2),
        0,
        inst.Transform:GetWorldPosition() + math.random(-2, 2)
    )
    return true  -- 返回 true = 骚扰成功；返回 false = 骚扰失败
end)

-- 设置骚扰积分（可选）
inst.components.hauntable:SetHauntValue(TUNING.HAUNT_SMALL)

-- 设置冷却时间（可选，默认 HAUNT_COOLDOWN_SMALL = 3 秒）
inst.components.hauntable.cooldown = TUNING.HAUNT_COOLDOWN_MEDIUM
```

**`onhaunt` 回调的返回值决定成功/失败**：
- 返回 `true` → 骚扰成功，触发标准冷却（`cooldown_on_successful_haunt = true`）
- 返回 `false` → 骚扰失败，触发较短的冷却（`HAUNT_COOLDOWN_SMALL`），`hauntvalue` 不生效

**参考：篝火（campfire）的骚扰响应**：

```101:110:scripts/prefabs/campfire.lua
local function OnHaunt(inst)
    if inst.components.fueled ~= nil and
        inst.components.fueled.accepting and
        math.random() <= TUNING.HAUNT_CHANCE_OCCASIONAL then
        inst.components.fueled:DoDelta(TUNING.TINY_FUEL)
        inst.components.hauntable.hauntvalue = TUNING.HAUNT_SMALL
        return true
    end
    return false
end
```

**参考：墓碑（gravestone）的骚扰响应**：

```48:64:scripts/prefabs/gravestone.lua
local function OnHaunt(inst)
    if not inst.setepitaph and #STRINGS.EPITAPHS > 1 then
        -- 随机换一条碑文（保证不和之前相同）
        local oldepitaph = inst.components.inspectable.description
        inst._epitaph_index = math.random(#STRINGS.EPITAPHS - 1)
        local newepitaph = STRINGS.EPITAPHS[inst._epitaph_index]
        if newepitaph == oldepitaph then
            newepitaph = STRINGS.EPITAPHS[#STRINGS.EPITAPHS]
        end
        inst.components.inspectable:SetDescription(newepitaph)
        inst.components.hauntable.hauntvalue = TUNING.HAUNT_SMALL
    else
        inst.components.hauntable.hauntvalue = TUNING.HAUNT_TINY
    end
    return true
end
```

> **新手记忆**：骚扰响应 = `AddComponent("hauntable")` + `SetOnHauntFn(fn)` + 回调里返回 true（成功）或 false（失败）。不设 `hauntvalue` 没关系，但设了可以调整成功积分。

---

### 19.3.4 进阶：`Hauntable:DoHaunt` 完整执行流程

`DoHaunt` 是骚扰的核心——让我们逐行分析：

```82:115:scripts/components/hauntable.lua
function Hauntable:DoHaunt(doer)
    if self.onhaunt ~= nil then
        -- 特判：如果有 itemmimic（物品拟态）组件，由它处理
        if self.inst.components.itemmimic then
            self.inst.components.itemmimic:TurnEvil(doer)
            return
        end

        -- 调用骚扰回调
        self.haunted = self.onhaunt(self.inst, doer)
        
        if self.haunted then
            -- 骚扰成功
            if doer ~= nil then
                -- 检查是否触发即时复活
                if self.hauntvalue == TUNING.HAUNT_INSTANT_REZ and doer:HasTag("playerghost") then
                    doer:PushEvent("respawnfromghost", { source = self.inst })
                end
                -- 清除 hauntvalue（防止重复使用）
                if not self.no_wipe_value then
                    self.hauntvalue = nil
                end
            end
            -- 触发冷却（如果 cooldown_on_successful_haunt = true，默认是）
            if self.cooldown_on_successful_haunt then
                self.cooldowntimer = self.cooldown or TUNING.HAUNT_COOLDOWN_MEDIUM
                self:StartFX(true)
                self:StartShaderFx()
                self.inst:StartUpdatingComponent(self)
            end
        else
            -- 骚扰失败：也有短暂冷却，防止连续尝试
            if self.inst:IsValid() then
                self.haunted = true
                self.cooldowntimer = self.cooldown or TUNING.HAUNT_COOLDOWN_SMALL
                self:StartFX(true)
                self:StartShaderFx()
                self.inst:StartUpdatingComponent(self)
            end
        end
    end
    -- 无论成功失败，都推送 haunted 事件
    self.inst:PushEvent("haunted")
end
```

**关键细节**：

1. **`itemmimic` 组件特判**：如果实体是物品拟态（某些装饰性道具变成怪物），骚扰触发 `TurnEvil`，**不走 onhaunt**

2. **hauntvalue 清除**：每次成功骚扰后，`hauntvalue` 被设为 nil（除非 `no_wipe_value = true`）。这意味着 `HAUNT_INSTANT_REZ` 效果通常**只能触发一次**

3. **失败也有冷却**：即使骚扰失败（onhaunt 返回 false），也会设置短冷却时间，防止玩家连点

4. **`"haunted"` 事件**：无论成功失败，都给实体推送这个事件——任何监听 `"haunted"` 的代码都会被触发

---

### 19.3.5 进阶：冷却系统——cooldown、haunted tag 与 Shader FX

#### 冷却时间常量

```lua
-- scripts/tuning.lua
HAUNT_COOLDOWN_TINY   = 1,   -- 1 秒
HAUNT_COOLDOWN_SMALL  = 3,   -- 3 秒（默认失败冷却）
HAUNT_COOLDOWN_MEDIUM = 5,   -- 5 秒（默认成功冷却）
HAUNT_COOLDOWN_LARGE  = 7,   -- 7 秒
HAUNT_COOLDOWN_HUGE   = 10,  -- 10 秒
```

#### 冷却计时逻辑

```160:179:scripts/components/hauntable.lua
function Hauntable:OnUpdate(dt)
    if self.cooldowntimer <= 0 then
        self:StopHaunt()     -- 冷却结束：移除 haunted 状态
    else
        self.cooldowntimer = self.cooldowntimer - dt
        -- 闪烁特效的最后 0.4 秒进入淡出阶段
        if self.cooldowntimer < .4 and self.flickering == "on" then
            self:AdvanceFlickerState()
        end
    end
    -- ...panictimer 计时...
end
```

`StopHaunt` 被调用时：

```144:152:scripts/components/hauntable.lua
function Hauntable:StopHaunt()
    self.cooldowntimer = 0
    self.haunted = false
    if self.onunhaunt then
        self.onunhaunt(self.inst)  -- 可选的"骚扰结束"回调
    end
    self:StopShaderFX()
    self:TryStopUpdating()
end
```

冷却结束时：`haunted = false` → `"haunted"` tag 被移除（通过 `haunted` 网络变量触发）→ componentactions 再次允许 HAUNT 动作显示。

#### Shader FX——闪烁效果

骚扰冷却期间实体会**闪烁**（使用专用 shader）：

```128:131:scripts/components/hauntable.lua
function Hauntable:StartShaderFx()
    local AnimState = self:GetAnimState()
    AnimState:SetHaunted(true)
end
```

`AnimState:SetHaunted(true)` 启用幽灵骚扰 shader——通常表现为淡蓝色闪烁光芒，代表物体刚被骚扰过。

`SetAnimStateGetterFn` 允许自定义哪个 AnimState 接收闪烁效果（见 19.3.9 老手部分）。

---

### 19.3.6 进阶：hauntvalue 系统——骚扰积分表

`hauntvalue` 是一个数值，代表骚扰效果的"分量"。常量定义：

```lua
HAUNT_TINY    = 1,
HAUNT_SMALL   = 3,
HAUNT_MEDIUM  = 5,
HAUNT_LARGE   = 10,
HAUNT_INSTANT_REZ = 9999,  -- 特殊：即时复活
```

在 DST 中，**`hauntvalue` 的实际游戏意义较有限**——没有像单机版那样的"积累骚扰值解锁特效"系统。它在 DST 里主要用途是：
- **判断 HAUNT_INSTANT_REZ**——值 = 9999 时触发复活
- **作为日志/调试信息**——知道这次骚扰"有多成功"

在单机版 DS 里，hauntvalue 累积到一定程度可以触发特殊事件（如骚扰火堆达到一定积分会引爆）——但这在 DST 里被大量的 `#HAUNTFIX` 注释掉了。

**`no_wipe_value = true`**：
```lua
function Hauntable:SetHauntValue(val)
    if not val then return end
    self.hauntvalue = val
    self.no_wipe_value = true
end
```

调用 `SetHauntValue(val)` 会同时设置 `no_wipe_value = true`——意味着每次成功骚扰后，`hauntvalue` **不会被清除**，可以持续触发相同效果（比如传送门每次骚扰都能复活）。

如果直接赋值 `inst.components.hauntable.hauntvalue = xxx`（不用 SetHauntValue），`no_wipe_value` 默认是 `false`——用一次就清空。

---

### 19.3.7 进阶：MakeHauntableXxx 工厂函数家族——重要注意事项

`scripts/standardcomponents.lua` 提供了一系列 `MakeHauntableXxx` 快速配置函数：

```987:991:scripts/standardcomponents.lua
function MakeHauntable(inst, cooldown, haunt_value)
    if not inst.components.hauntable then inst:AddComponent("hauntable") end
    inst.components.hauntable.cooldown = cooldown or TUNING.HAUNT_COOLDOWN_SMALL
    inst.components.hauntable:SetHauntValue(haunt_value or TUNING.HAUNT_TINY)
end
```

**家族成员**：

| 函数名 | 设计效果 | 当前状态 |
|--------|----------|---------|
| `MakeHauntable(inst, cooldown, haunt_value)` | 最基础骚扰（无具体效果）| ✅ 活跃（无 #HAUNTFIX）|
| `MakeHauntableLaunch(inst, chance, speed, ...)` | 概率飞出 | ✅ 活跃 |
| `MakeHauntableLaunchAndSmash(...)` | 飞出 + 砸碎 | ⚠️ 砸碎部分被 #HAUNTFIX 注释 |
| `MakeHauntableWork(...)` | 加工（砍伐/挖掘）| ❌ 整体 #HAUNTFIX 注释 |
| `MakeHauntableWorkAndIgnite(...)` | 加工 + 点燃 | ❌ 整体 #HAUNTFIX 注释 |
| `MakeHauntableFreeze(inst, chance, ...)` | 概率冻结 | ✅ 活跃 |
| `MakeHauntableIgnite(...)` | 概率点燃 | ❌ 整体 #HAUNTFIX 注释 |
| `MakeHauntableLaunchAndIgnite(...)` | 飞出 + 点燃 | ⚠️ 点燃部分 #HAUNTFIX 注释 |
| `MakeHauntableChangePrefab(...)` | 变成另一种 prefab | ✅ 活跃 |
| `MakeHauntablePerish(...)` | 加速腐烂 | ❌ 整体 #HAUNTFIX 注释 |
| `MakeHauntableLaunchAndPerish(...)` | 飞出 + 腐烂 | ⚠️ 腐烂部分 #HAUNTFIX 注释 |
| `MakeHauntablePanic(...)` | 引发 Panic 逃跑 | ✅ 活跃 |
| `MakeHauntablePanicAndIgnite(...)` | Panic + 点燃 | ⚠️ 点燃部分 #HAUNTFIX 注释 |

**⚠️ 重要警告**：带有 `#HAUNTFIX` 注释的函数，其原设计效果**在 DST 中不生效**——对应的代码被注释掉了，调用这些函数后骚扰什么都不会发生（只有飞出/Panic 等基础效果保留）。

**`#HAUNTFIX` 的背景**：这些效果在单机版 DS 中存在，但 Klei 在 DST 移植时认为这些效果影响平衡（比如幽灵可以烧树等），决定暂时关闭等待重新设计，留下了 `#HAUNTFIX` 标记。截至目前（2026年）这些效果仍然注释掉。

**示例：`MakeHauntableLaunch` 的完整实现**：

```993:1009:scripts/standardcomponents.lua
function MakeHauntableLaunch(inst, chance, speed, cooldown, haunt_value)
    if not inst.components.hauntable then inst:AddComponent("hauntable") end
    inst.components.hauntable.cooldown = cooldown or TUNING.HAUNT_COOLDOWN_SMALL
    inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
        chance = chance or TUNING.HAUNT_CHANCE_ALWAYS
        if math.random() <= chance then
            Launch(inst, haunter, speed or TUNING.LAUNCH_SPEED_SMALL)
            inst.components.hauntable.hauntvalue = haunt_value or TUNING.HAUNT_TINY
            -- 标记为已落地=false，防止物品在飞行中不触发各种检测
            if inst.components.inventoryitem ~= nil and inst.components.inventoryitem.is_landed then
                inst.components.inventoryitem:SetLanded(false, true)
            end
            return true
        end
        return false
    end)
end
```

> **进阶记忆**：`MakeHauntableXxx` 函数家族很多效果被 #HAUNTFIX 关闭；活跃的有：基础 MakeHauntable、Launch（飞出）、Freeze（冻结）、Panic（逃跑）、ChangePrefab（变形）；`SetHauntValue` 会设置 `no_wipe_value=true`（不清除），直接赋值则每次用完清空。

---

### 19.3.8 老手：Panic 模式——骚扰引发生物奔逃

`MakeHauntablePanic` 是一个常见的骚扰效果——骚扰后生物会四处乱跑（如猪人被骚扰后慌乱逃窜）：

```1279:1297:scripts/standardcomponents.lua
function MakeHauntablePanic(inst, panictime, chance, cooldown, haunt_value)
    if not inst.components.hauntable then inst:AddComponent("hauntable") end
    inst.components.hauntable.panicable = true
    inst.components.hauntable.cooldown = cooldown or TUNING.HAUNT_COOLDOWN_MEDIUM
    inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
        if inst.components.sleeper then -- 唤醒睡眠中的生物！
            inst.components.sleeper:WakeUp()
        end

        chance = chance or TUNING.HAUNT_CHANCE_ALWAYS
        if math.random() <= chance then
            inst.components.hauntable.panic = true
            inst.components.hauntable.panictimer = panictime or TUNING.HAUNT_PANIC_TIME_SMALL
            inst.components.hauntable.hauntvalue = haunt_value or TUNING.HAUNT_SMALL
            return true
        end
        return false
    end)
end
```

**panic 机制**：`hauntable.panic = true` + `panictimer > 0` 时，hauntable 的 `OnUpdate` 会持续计时。生物的 AI（brain/stategraph）通常监听 `"haunted"` 事件或检查 `hauntable.panic` 字段来触发逃跑行为。

**`Hauntable:Panic` 直接调用**：
```52:58:scripts/components/hauntable.lua
function Hauntable:Panic(panictime)
    self.haunted = true
    self.panic = true
    panictime = panictime or TUNING.HAUNT_PANIC_TIME_SMALL
    self.panictimer = math.max(self.panictimer, panictime)
    self.cooldowntimer = math.max(self.cooldowntimer, panictime)
    self.inst:StartUpdatingComponent(self)
end
```

可以在 AI 层或事件中直接调用 `inst.components.hauntable:Panic(duration)` 触发逃跑——不需要经过 `DoHaunt`。

**Panic 持续时间常量**：
```lua
HAUNT_PANIC_TIME_SMALL  = 3,   -- 3 秒恐慌
HAUNT_PANIC_TIME_MEDIUM = 5,
HAUNT_PANIC_TIME_LARGE  = 7,
```

---

### 19.3.9 老手：`SetAnimStateGetterFn` 与多实体 Shader FX 定制

默认情况下，骚扰特效（`SetHaunted(true)`）作用在 `self.inst.AnimState` 上。但某些实体有多个视觉部件，希望特效作用在不同的 AnimState。

`SetAnimStateGetterFn` 允许自定义返回哪个 AnimState 接受特效：

```117:126:scripts/components/hauntable.lua
function Hauntable:SetAnimStateGetterFn(fn)
    self.animstatefn = fn
end

function Hauntable:GetAnimState()
    if self.animstatefn then
        return self.animstatefn(self.inst)
    end
    return self.inst.AnimState
end
```

**使用示例**（让特效作用在子实体的 AnimState 上）：

```lua
-- 假设实体有一个 child_fx 子实体
inst.components.hauntable:SetAnimStateGetterFn(function(inst)
    if inst.child_fx ~= nil and inst.child_fx:IsValid() then
        return inst.child_fx.AnimState
    end
    return inst.AnimState  -- 后备
end)
```

这在有多层动画的复杂建筑（如多段式机器）中很有用——骚扰特效只出现在"发光部件"而不是整个建筑。

---

### 19.3.10 老手：五个常见坑

#### 坑 1：骚扰成功但幽灵没有即时复活——`SetHauntValue` 与直接赋值的区别

**症状**：设置了 `inst.components.hauntable.hauntvalue = TUNING.HAUNT_INSTANT_REZ`，但骚扰后幽灵没有复活。

**原因**：直接赋值 `hauntvalue`，`no_wipe_value` 默认是 `false`；第一次骚扰成功后，`hauntvalue` 被 `DoHaunt` 清零 (`hauntvalue = nil`)，第二次骚扰就无效了。

但如果这是**第一次**骚扰就无效，可能是因为 `onhaunt` 回调没返回 `true`——`DoHaunt` 在 `haunted = true` 分支才检查 `hauntvalue`。

**修复**：用 `SetHauntValue(TUNING.HAUNT_INSTANT_REZ)` 而不是直接赋值——这会设置 `no_wipe_value = true`，每次骚扰都能触发复活。

#### 坑 2：`MakeHauntableIgnite` 调了但骚扰点不着火——#HAUNTFIX

**症状**：调用 `MakeHauntableIgnite` 后，幽灵骚扰实体，火没有被点燃。

**原因**：查看源码会发现 `MakeHauntableIgnite` 的点燃代码被 `#HAUNTFIX` 注释掉，函数目前**什么都不做**（返回 false）。

**修复**：不要依赖这个函数——自己实现 `SetOnHauntFn`：

```lua
inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
    if inst.components.burnable and not inst.components.burnable:IsBurning() then
        inst.components.burnable:Ignite()
        return true
    end
    return false
end)
```

#### 坑 3：骚扰被阻止——实体有 `catchable` tag

**症状**：实体有 `hauntable` 组件，但幽灵对它没有任何骚扰选项。

**原因**：如果实体同时有 `"catchable"` tag，骚扰选项被隐藏（`componentactions.lua:411`）。这是为了让蝴蝶等可捕捉实体不被幽灵骚扰混淆——但如果 mod 给自己的 prefab 加了 `catchable`（不小心），就会看不到骚扰选项。

**检查**：`print(inst:HasTag("catchable"))` 排查。

#### 坑 4：`"haunted"` 事件被误用——它不代表"正在被骚扰"

**症状**：用 `ListenForEvent("haunted", fn)` 想在骚扰成功时执行，但实际上每次骚扰（包括失败）都触发。

**原因**：`"haunted"` 事件在 `DoHaunt` **末尾**无条件推送（不管 onhaunt 返回 true 还是 false）。它表示"发生了骚扰动作"，不是"骚扰成功"。

**修复**：在 `onhaunt` 回调里直接处理成功/失败逻辑；或者在 `"haunted"` 回调里检查 `self.components.hauntable.haunted`（骚扰成功则为 true）。

#### 坑 5：骚扰响应在客户端执行——对游戏逻辑无效

**症状**：在 `onhaunt` 回调里修改 `health`、生成实体等，但没有效果。

**原因**：`ACTIONS.HAUNT.fn` 和 `DoHaunt` 运行在**服务端**——没问题。但如果通过 `AddClassPostConstruct` 或客户端代码给实体添加了 `hauntable` 组件，这个组件存在于客户端，`DoHaunt` 只在客户端执行，效果不会同步到服务器。

**修复**：确保 `hauntable` 组件在 `if not TheWorld.ismastersim then return inst end` 之后（服务端）添加：

```lua
local function fn()
    local inst = CreateEntity()
    -- ... 客户端配置 ...
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end
    -- 下面是服务端逻辑
    inst:AddComponent("hauntable")
    inst.components.hauntable:SetOnHauntFn(...)
    return inst
end
```

---

### 19.3 小结

```
骚扰触发链：
  幽灵点击有 hauntable 的实体（无 haunted/catchable tag）
      ↓ componentactions.lua → 显示 HAUNT 动作
      ↓ ACTIONS.HAUNT.fn
      → doer:PushEvent("haunt")      ← 幽灵进入 haunt_pre/haunt 动画
      → target.hauntable:DoHaunt(doer)
            ↓ onhaunt(inst, doer) 调用
            → true（成功）：
                hauntvalue == HAUNT_INSTANT_REZ → respawnfromghost
                no_wipe_value = false → hauntvalue 清零
                cooldown_on_successful_haunt → 启动 cooldown + Shader FX
            → false（失败）：
                启动短 cooldown + Shader FX
            → 无论成功失败：PushEvent("haunted")

核心数据：
  cooldown       实例字段，控制冷却时间（default: MEDIUM=5s）
  cooldowntimer  计时器
  haunted        是否在冷却（对应 "haunted" tag）
  hauntvalue     骚扰积分（TINY=1 SMALL=3 MEDIUM=5 LARGE=10 INSTANT_REZ=9999）
  no_wipe_value  hauntvalue 是否在成功后清零（SetHauntValue 设置为 true）
  panic/panictimer 恐慌状态计时

活跃的工厂函数（DST 未 #HAUNTFIX）：
  MakeHauntable（无效果基础版）
  MakeHauntableLaunch（飞出）
  MakeHauntableFreeze（冻结）
  MakeHauntablePanic（逃跑）
  MakeHauntableChangePrefab（变形）
```

**新手核心三句**：有 `hauntable` 组件 + 不处于冷却中 → 幽灵可骚扰；`HAUNT_INSTANT_REZ = 9999` 是即时复活触发值（传送门/触手石）；`SetOnHauntFn` 回调返回 true = 成功，false = 失败。

**进阶核心三句**：`SetHauntValue` 设置 `no_wipe_value=true`（持久），直接赋值则骚扰一次后清零；骚扰成功和失败都会有冷却期，失败的冷却更短；`#HAUNTFIX` 注释掉了大量效果（点燃、腐烂、加工），只有飞出/冻结/变形/Panic 保留。

**老手核心三句**：想持续触发 INSTANT_REZ 用 `SetHauntValue`；`"haunted"` 事件无条件触发（成功失败都有），不能作为"成功信号"；确保 `hauntable` 组件在服务端添加（`if not TheWorld.ismastersim then return end` 之后）。


## 19.4 复活机制全解：复活护符、肉雕像、告密的心、触手石

### 本节导读

19.2 我们知道幽灵通过 `REMOTERESURRECT` 动作复活，19.3 知道骚扰传送门/触手石也能复活。本节把五种复活途径全部理清——**从 hauntable 骚扰到 attunable 共鸣，从红色护符到传送门**，每一种都有各自的代码路径和特殊规则。

**本节讲五种复活途径**：
1. **复活护符（红色护符 amulet）**——死亡后掉落，幽灵骚扰它复活
2. **肉雕像（resurrectionstatue）**——活着时建造并绑定，幽灵使用 REMOTERESURRECT 复活
3. **告密的心（reviver）**——活着的队友手持用于复活
4. **触手石（resurrectionstone）**——世界上发现，激活后骚扰一次复活
5. **传送门（multiplayer_portal）**——幽灵骚扰激活的传送门复活（有血量惩罚）

> **新手**先看 19.4.1-19.4.3——一张对比表看清五种复活的特点差异、`respawnfromghost` 与 `REMOTERESURRECT` 两条触发路径、以及复活后通用的状态还原流程；**进阶读者**继续看 19.4.4-19.4.8，深入每种复活方式的源码实现——护符的 HAUNT_INSTANT_REZ 路径、肉雕像的 `attunable` 绑定系统、心脏的 `reviver` tag、触手石的一次性激活、传送门的血量惩罚；**老手**跳到 19.4.9-19.4.11，掌握 `DoActualRez` 的通用流程与各复活来源的动画分支、mod 添加自定义复活道具的两条路、以及五个常见坑。

---

### 19.4.1 快速入门：五种复活途径对比表

| 复活道具 | prefab | 触发方式 | 携带者 | 复活来源动画 | 血量惩罚 |
|---------|--------|----------|--------|------------|----------|
| **复活护符（红色护符）** | `amulet` | 幽灵骚扰掉落的护符 | 无（地上） | `amulet_rebirth`（自动装备） | 无 |
| **肉雕像** | `resurrectionstatue` | 幽灵使用 REMOTERESURRECT | 无（建筑） | `rebirth` | 无 |
| **告密的心** | `reviver` | 活人对幽灵使用 | 活着的队友 | `reviver_rebirth` | 无 |
| **触手石** | `resurrectionstone` | 幽灵骚扰已激活的石头 | 无（建筑） | `wakeup` | 无 |
| **传送门** | `*_portal`（有 `multiplayer_portal` tag）| 幽灵骚扰 | 无（建筑）| `portal_rez` | +25% 最大血量惩罚 |

**两种触发路径**：

```
路径 A：hauntable 骚扰触发（护符、触手石、传送门）
  幽灵骚扰 → ACTIONS.HAUNT.fn → DoHaunt
  → hauntvalue == HAUNT_INSTANT_REZ
  → PushEvent("respawnfromghost", { source = self.inst })

路径 B：REMOTERESURRECT 动作触发（肉雕像）
  幽灵使用 REMOTERESURRECT
  → attuner:GetAttunedTarget("remoteresurrector")
  → PushEvent("respawnfromghost", { source = target })

路径 C：活人道具使用（告密的心）
  活人对幽灵使用 reviver 物品
  → OnRespawnFromGhost，source:HasTag("reviver")
  → DoActualRez(inst, nil, source)（item 参数）
```

> **新手记忆**：护符/石头/传送门靠骚扰（HAUNT）复活；肉雕像靠幽灵主动使用特殊动作（REMOTERESURRECT）；心脏靠活人队友对幽灵使用；只有传送门有血量惩罚（累积最大血量 -25%）。

---

### 19.4.2 快速入门：`respawnfromghost` 与 `REMOTERESURRECT` 的两条触发路径

所有复活最终都经过 `respawnfromghost` 事件和 `OnRespawnFromGhost` 函数（`player_common_extensions.lua:569`）。区别只在于事件的 `source` 参数和触发条件。

**路径 A：hauntable 骚扰触发（护符、石头、传送门）**

在 `Hauntable:DoHaunt` 里（`hauntable.lua:91`）：
```lua
if self.hauntvalue == TUNING.HAUNT_INSTANT_REZ and doer:HasTag("playerghost") then
    doer:PushEvent("respawnfromghost", { source = self.inst })
end
```
`source` 是被骚扰的物体本身。

**路径 B：`ACTIONS.REMOTERESURRECT` 触发（肉雕像）**

```4054:4066:scripts/actions.lua
ACTIONS.REMOTERESURRECT.fn = function(act)
    if act.doer == nil then return end

    local doer_attuner = act.doer.components.attuner
    if doer_attuner and act.doer:HasTag("playerghost") then
        local target = doer_attuner:GetAttunedTarget("remoteresurrector")
            or doer_attuner:GetAttunedTarget("gravestoneresurrector")
        if target ~= nil then
            act.doer:PushEvent("respawnfromghost", { source = target })
            return true
        end
    end
end
```

`source` 是绑定的肉雕像实体。幽灵的 `attuner` 组件里存着绑定关系，通过 `GetAttunedTarget("remoteresurrector")` 找到对应的雕像。

**什么时候 REMOTERESURRECT 动作可见？**

查找 `componentactions.lua` 里的 `attunable_classified` 处理：幽灵的 HUD 中有一个特殊的复活按钮区域，当 `attuner:HasAttunement("remoteresurrector")` 返回 true 时（即已经建造了肉雕像并绑定），REMOTERESURRECT 选项出现。

> **新手记忆**：幽灵复活的核心事件是 `respawnfromghost`，`source` 字段告诉系统是用什么道具复活的，进而决定播放哪个复活动画和触发哪些特殊逻辑。

---

### 19.4.3 快速入门：复活后的状态还原——CommonActualRez

无论哪种复活方式，最终都调 `CommonActualRez`（`player_common_extensions.lua:267`）还原存活状态：

```267:325:scripts/prefabs/player_common_extensions.lua
local function CommonActualRez(inst)
    inst.player_classified.MapExplorer:EnableUpdate(true)

    -- 重新打开背包/恢复老化
    if inst.components.revivablecorpse ~= nil then
        inst.components.inventory:Show()
    else
        inst.components.inventory:Open()
        inst.components.age:ResumeAging()
    end

    -- 重新允许治疗
    inst.components.health.canheal = true
    -- 恢复饥饿/体温计时
    inst.components.hunger:Resume()
    inst.components.temperature:SetTemp()  -- nil 参数 = 恢复体温更新
    inst.components.frostybreather:Enable()

    -- 重新加回被移除的组件
    MakeMediumBurnableCharacter(inst, "torso")  -- burnable
    MakeLargeFreezableCharacter(inst, "torso")  -- freezable
    inst:AddComponent("grogginess")
    inst:AddComponent("slipperyfeet")

    inst.components.moisture:ForceDry(false, inst)
    inst.components.sheltered:Start()
    inst.components.debuffable:Enable(true)

    -- 恢复精神计算
    inst.components.sanity.ignore = GetGameModeProperty("no_sanity")

    -- 恢复移动/行动
    ConfigurePlayerLocomotor(inst)
    ConfigurePlayerActions(inst)

    -- 发复活公告
    if inst.rezsource ~= nil then
        TheNet:AnnounceResurrect(...)
    end
    inst.remoterezsource = nil
    inst.last_death_position = nil
    -- ...
end
```

**复活后的初始血量**：

- 通用路径：复活时血量 = 幽灵的当前血量（`RESURRECT_HEALTH = 50`）
- 但每种来源可能在 `DoActualRez` 里额外设置血量或惩罚（见 19.4.4-19.4.8）

> **新手记忆**：复活后系统会自动恢复饥饿计时、体温、精神计算等；不需要 mod 手动恢复这些状态；复活血量基础为 50 HP，传送门复活有额外的最大血量惩罚。

---

### 19.4.4 进阶：复活护符（红色护符 amulet）

红色护符是最经典的复活道具，也是游戏里最早的复活机制。

#### 工作原理

护符在玩家死亡时**随物品一起掉落**（注意注释里的说明）：

```424:450:scripts/prefabs/amulet.lua
local function red()
    local inst = commonfn("redamulet", "resurrector", true)
    -- ...

    -- red amulet now falls off on death, so you HAVE to haunt it
    -- This is more straightforward for prototype purposes, but has side effect of allowing amulet steals
    -- inst.components.inventoryitem.keepondeath = true

    inst.components.equippable:SetOnEquip(onequip_red)
    inst.components.equippable:SetOnUnequip(onunequip_red)
    inst.components.equippable:SetOnEquipToModel(onequiptomodel_red)

    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetOnFinished(inst.Remove)
    inst.components.finiteuses:SetMaxUses(TUNING.REDAMULET_USES)
    inst.components.finiteuses:SetUses(TUNING.REDAMULET_USES)

    inst:AddComponent("hauntable")
    inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)

    return inst
end
```

关键设计：
1. **`keepondeath = true` 被注释掉**——护符会随其他物品一起掉落（不保留在身上）
2. **`hauntable:SetHauntValue(HAUNT_INSTANT_REZ)`**——骚扰后立即复活，可多次使用（`no_wipe_value = true`）
3. 护符有耐久（`REDAMULET_USES`），每次复活消耗一次

#### DoActualRez 里的护符分支

```368:370:scripts/prefabs/player_common_extensions.lua
        if source.prefab == "amulet" then
            inst.components.inventory:Equip(source)
            inst.sg:GoToState("amulet_rebirth")
```

**特殊逻辑**：复活时**直接把护符装备到身上**——这就是为什么你复活后护符已经穿在身上了。

> **进阶记忆**：护符会在死亡时掉落（不 keepondeath），幽灵骚扰后通过 `HAUNT_INSTANT_REZ` 触发复活，复活时自动穿上护符。因为 `no_wipe_value=true`，多耐久护符可以多次复活。

---

### 19.4.5 进阶：肉雕像（resurrectionstatue）——`attunable` 绑定系统

肉雕像是唯一使用 **`attunable`（绑定）组件**的复活方式——玩家活着时"充能"（绑定自己与雕像），死亡后才能远程使用。

#### attunable 系统

```168:172:scripts/prefabs/resurrectionstatue.lua
    inst:AddComponent("attunable")
    inst.components.attunable:SetAttunableTag("remoteresurrector")
    inst.components.attunable:SetOnAttuneCostFn(onattunecost)
    inst.components.attunable:SetOnLinkFn(onlink)
    inst.components.attunable:SetOnUnlinkFn(onunlink)
```

- `attunable` 组件：实体可以和玩家建立绑定关系
- `SetAttunableTag("remoteresurrector")`：绑定后，玩家的 `attuner` 组件里记录一个 `"remoteresurrector"` 标记的目标

#### 绑定代价——血量消耗

```61:73:scripts/prefabs/resurrectionstatue.lua
local function onattunecost(inst, player)
    local amount_required = player:HasTag("health_as_oldage")
        and math.ceil(TUNING.EFFIGY_HEALTH_PENALTY * TUNING.OLDAGE_HEALTH_SCALE)
        or TUNING.EFFIGY_HEALTH_PENALTY

    if player.components.health == nil or math.ceil(player.components.health.currenthealth) <= amount_required then
        --Don't die from attunement!
        return false, "NOHEALTH"
    end

    player:PushEvent("consumehealthcost")
    player.components.health:DoDelta(-TUNING.EFFIGY_HEALTH_PENALTY, false, "statue_attune", true, inst, true)
    return true
end
```

绑定代价：扣除 `EFFIGY_HEALTH_PENALTY = 40` 点血量（但有保护：血量低于 40 + 1 时拒绝绑定）。

#### 建造时自动绑定

```97:114:scripts/prefabs/resurrectionstatue.lua
local function onbuilt(inst, data)
    --Hack to auto-link without triggering fx or paying the cost again
    inst.components.attunable:SetOnAttuneCostFn(nil)
    -- ...
    if inst.components.attunable:LinkToPlayer(data.builder) then
        -- ...自动绑定到建造者
    end
    -- ...恢复原来的 cost fn
    inst.components.attunable:SetOnAttuneCostFn(onattunecost)
```

建造时**不收费**自动绑定到建造者——这就是为什么建完雕像不扣血（建造本身已经花了材料），但**之后别人想绑定才需要扣 40 血**。

#### DoActualRez 里的雕像分支

```375:376:scripts/prefabs/player_common_extensions.lua
        elseif source.prefab == "resurrectionstatue" then
            inst.sg:GoToState("rebirth", source)
```

进入 `rebirth` 状态，播放从雕像复活的动画。雕像本身在复活后被**销毁**（`inst:ListenForEvent("activateresurrection", inst.Remove)`）。

> **进阶记忆**：肉雕像通过 `attunable` 系统与玩家绑定；绑定需要消耗 40 血（建造不收）；REMOTERESURRECT.fn 通过 `attuner:GetAttunedTarget("remoteresurrector")` 找到雕像；用一次后雕像销毁。

---

### 19.4.6 进阶：告密的心（reviver）——活人使用的复活道具

告密的心是唯一**需要另一位活着的玩家**操作的复活方式。

#### reviver prefab

```48:79:scripts/prefabs/reviver.lua
local function fn()
    local inst = CreateEntity()
    -- ...
    inst:AddTag("reviver")  -- ← 关键 tag
    -- ...
    inst:AddComponent("inventoryitem")
    inst.components.inventoryitem:SetOnDroppedFn(ondropped)
    inst.components.inventoryitem:SetOnPutInInventoryFn(onpickup)
    -- ...
    MakeHauntableLaunch(inst)  -- 地上时可以被幽灵弹飞
    -- ...
```

心脏的核心是 `"reviver"` tag——`OnRespawnFromGhost` 里检查：

```593:594:scripts/prefabs/player_common_extensions.lua
    elseif data.source:HasTag("reviver") then
        inst:DoTaskInTime(0, DoActualRez, nil, data.source)
```

当 source 有 `reviver` tag 时，调用 `DoActualRez(inst, nil, data.source)`——注意 source=nil，data.source 作为 `item` 传入。

#### DoActualRez 里的心脏分支

```410:412:scripts/prefabs/player_common_extensions.lua
        else -- Telltale Heart
            inst.sg:GoToState("reviver_rebirth", item)
        end
```

这是 `item` 参数不为 nil 且不是 pocketwatch 时的分支——进入 `reviver_rebirth` 状态。

#### 活人如何对幽灵使用心脏？

活人拿着 `reviver` 物品，走近幽灵，会出现"使用"选项（通过 `ACTIONS.GIVE` 或专用 componentaction）。这最终触发幽灵的 `respawnfromghost` 事件。

**心脏的跳动特效**：
```10:14:scripts/prefabs/reviver.lua
local function beat(inst)
    inst:PlayBeatAnimation()
    inst.SoundEmitter:PlaySound("dontstarve/ghost/bloodpump")
    inst.beattask = inst:DoTaskInTime(.75 + math.random() * .75, beat)
end
```
在地上时，心脏会以随机间隔跳动播放动画和音效——这是纯粹的视觉/听觉效果。

> **进阶记忆**：心脏靠 `"reviver"` tag 被识别；需要活着的玩家手持对幽灵使用；DoActualRez 里作为 `item` 参数传入（不是 source），进入 `reviver_rebirth` 动画。

---

### 19.4.7 进阶：触手石（resurrectionstone）——一次性激活后骚扰

触手石是世界中自然生成的复活建筑——需要先激活，然后才能被幽灵骚扰。

```87:94:scripts/prefabs/resurrectionstone.lua
local function OnAnimOver(inst)
    if inst.components.hauntable == nil and
        inst.AnimState:IsCurrentAnimation("idle_activate") then
        inst:AddComponent("hauntable")
        inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
        inst.components.hauntable:SetOnHauntFn(OnHaunt)
    end
end
```

**激活后才加 hauntable**——未激活的触手石没有 `hauntable` 组件，幽灵完全无法骚扰它。

激活触手石是通过玩家（活人）对其进行激活动作（靠近使用），播放 `idle_activate` 动画，动画结束后加上 `hauntable` 组件。

#### DoActualRez 里的触手石分支

```371:374:scripts/prefabs/player_common_extensions.lua
        elseif source.prefab == "resurrectionstone" then
            inst.components.inventory:Hide()
            inst:PushEvent("ms_closepopups")
            inst.sg:GoToState("wakeup")
```

进入 `wakeup` 状态——播放从地面觉醒的动画。触手石使用后激活器（cooldown 或摧毁）——玩家只能用一次，之后需要再次激活。

> **进阶记忆**：触手石分两步——活人激活（加 hauntable）+ 幽灵骚扰（触发复活）；只有激活过的石头才有 `hauntable` 组件；用一次后需再次激活。

---

### 19.4.8 进阶：传送门（multiplayer_portal）——带血量惩罚的复活

传送门是最容易获取但有代价的复活方式。

```45:53:scripts/prefabs/multiplayer_portal.lua
local function OnGetPortalRez(inst, portalrez)
    if portalrez then
        inst:AddComponent("hauntable")
        inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
        inst:AddTag("resurrector")
    elseif inst.components.hauntable then
        inst:RemoveComponent("hauntable")
        inst:RemoveTag("resurrector")
    end
end
```

`portalrez` 是一个网络变量——当服务器确认传送门可以复活时设为 true（例如传送门已建立、本服有槽位）。

#### 血量惩罚

```385:389:scripts/prefabs/player_common_extensions.lua
        elseif source:HasTag("multiplayer_portal") then
            inst.components.health:DeltaPenalty(TUNING.PORTAL_HEALTH_PENALTY)

            source:PushEvent("rez_player")
            inst.sg:GoToState("portal_rez")
```

`inst.components.health:DeltaPenalty(TUNING.PORTAL_HEALTH_PENALTY)` 中，`PORTAL_HEALTH_PENALTY = 0.25`——给玩家增加 **25% 的血量惩罚**（不是直接扣血，而是降低最大血量上限）。

血量惩罚是**累积的**——如果你一直靠传送门复活，每次加 25% 惩罚，最大血量越来越低（通常通过特定道具消除惩罚）。

> **进阶记忆**：传送门复活会累积 25% 最大血量惩罚（`DeltaPenalty(0.25)`）；`portalrez` 网络变量控制传送门是否有 hauntable 组件；`source:HasTag("multiplayer_portal")` 是判断依据。

---

### 19.4.9 老手：DoActualRez 通用流程与各复活来源的动画分支

`DoActualRez` 是所有复活路径的**最终公共函数**，让我们完整看它的逻辑：

```327:431:scripts/prefabs/player_common_extensions.lua
local function DoActualRez(inst, source, item)
    -- 1. 在复活位置生成 die_fx 特效
    local x, y, z = (source or inst).Transform:GetWorldPosition()
    SpawnPrefab("die_fx").Transform:SetPosition(x, y, z)

    -- 2. 恢复外观（隐藏帽子特定部件，显示头发）
    inst.AnimState:Hide("HAT")
    inst.AnimState:Show("HAIR_NOHAT")
    -- ...（更多 Show/Hide）

    inst:Show()

    -- 3. 切换状态图回 SGwilson
    inst:SetStateGraph("SGwilson")

    -- 4. 传送到来源位置
    inst.Physics:Teleport(x, y, z)

    -- 5. 取消幽灵模式
    inst.player_classified:SetGhostMode(false)

    -- 6. 根据来源 prefab 分支
    if source ~= nil then
        -- 护符、触手石、肉雕像、温蒂墓碑、WX备份、传送门……
        inst.DynamicShadow:Enable(true)
        inst.AnimState:SetBank("wilson")
        -- 恢复皮肤、光照等
        source:PushEvent("activateresurrection", inst)  -- 通知来源"复活了"

        if source.prefab == "amulet" then
            inst.components.inventory:Equip(source)
            inst.sg:GoToState("amulet_rebirth")
        elseif source.prefab == "resurrectionstone" then
            inst.sg:GoToState("wakeup")
        elseif source.prefab == "resurrectionstatue" then
            inst.sg:GoToState("rebirth", source)
        elseif source:HasTag("multiplayer_portal") then
            inst.components.health:DeltaPenalty(TUNING.PORTAL_HEALTH_PENALTY)
            source:PushEvent("rez_player")
            inst.sg:GoToState("portal_rez")
        -- ...其他分支
        end
    else
        -- item 分支（心脏、口袋表等）
        if item ~= nil and (item.prefab == "pocketwatch_revive" ...) then
            -- 口袋表特殊处理
        else  -- Telltale Heart
            inst.sg:GoToState("reviver_rebirth", item)
        end
    end

    -- 7. 恢复物理
    MakeCharacterPhysics(inst, 75, .5)
    inst.Physics:Stop()

    -- 8. 调用通用状态还原
    CommonActualRez(inst)

    -- 9. 移除 playerghost 标签
    inst:RemoveTag("playerghost")
    inst.Network:RemoveUserFlag(USERFLAGS.IS_GHOST)

    -- 10. 推送复活完成事件
    inst:PushEvent("ms_respawnedfromghost")
end
```

**`activateresurrection` 事件**：每当复活发生，`source:PushEvent("activateresurrection", inst)` 会通知复活来源（触手石/雕像等）"我被用了"——各来源在这里做自我销毁或冷却逻辑。

---

### 19.4.10 老手：mod 自定义复活道具

#### 方法 1：添加 `reviver` tag（活人对幽灵使用）

最简单。让你的 mod 道具成为一个活人可以对幽灵使用的复活工具：

```lua
-- 在 prefab fn 里
inst:AddTag("reviver")  -- 标记为复活道具

-- 这样 OnRespawnFromGhost 里就能识别：
-- elseif data.source:HasTag("reviver") then
--     inst:DoTaskInTime(0, DoActualRez, nil, data.source)
```

复活动画会走 `reviver_rebirth` 分支。

#### 方法 2：添加 `hauntable:SetHauntValue(HAUNT_INSTANT_REZ)`（幽灵自己骚扰复活）

让你的 mod 建筑/道具可以被幽灵骚扰后自动复活：

```lua
-- 在 prefab fn 里（服务端侧）
inst:AddComponent("hauntable")
inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
-- 可选：添加 onhaunt 做额外处理
inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
    -- 自定义特效等
    return true
end)
```

复活动画会走 `source` 是你的 prefab 的分支——如果没有专用分支，可能会走默认逻辑（什么动画都不播，直接变回来）。如果想要专用动画，需要在 `DoActualRez` 里添加分支（通过 `AddClassPostConstruct`）。

#### 方法 3：使用 `attunable` 系统（远程绑定，幽灵使用 REMOTERESURRECT）

和肉雕像一样的系统：

```lua
inst:AddComponent("attunable")
inst.components.attunable:SetAttunableTag("remoteresurrector")
-- 绑定代价
inst.components.attunable:SetOnAttuneCostFn(function(inst, player)
    -- ...消耗资源
    return true  -- 允许绑定
end)
-- 绑定成功回调
inst.components.attunable:SetOnLinkFn(function(inst, player, isloading)
    -- ...
end)
```

绑定后，玩家的 `attuner` 会记录这个建筑，幽灵时可以通过 `REMOTERESURRECT` 动作使用。

---

### 19.4.11 老手：五个常见坑

#### 坑 1：mod 复活道具（reviver tag）无法使用——缺少 componentaction

**症状**：给道具加了 `reviver` tag，但活人拿着靠近幽灵没有"使用"选项。

**原因**：`reviver` tag 只是让 `OnRespawnFromGhost` 能识别这个道具，但**让选项出现**需要 componentaction 注册。查看 `componentactions.lua` 里有没有 reviver 的处理——如果没有，需要自己加。

**修复**：

```lua
-- modmain.lua 或相关文件
AddComponentAction("SCENE", "my_reviver_component", function(inst, doer, actions, right)
    if doer:HasTag("player") and not doer:HasTag("playerghost") then
        local ghost = doer:GetNearbyGhost()  -- 你需要实现这个查找
        if ghost then
            table.insert(actions, ACTIONS.GIVE)  -- 或自定义动作
        end
    end
end)
```

实际上 reviver 的使用通过普通的 GIVE 或 USEON 动作实现，需要配合 `useableitem` 或类似组件。

#### 坑 2：传送门复活的血量惩罚没有被清除

**症状**：多次靠传送门复活后，最大血量越来越低，但 mod 里增加了清除惩罚的道具却不起效。

**原因**：`DeltaPenalty(0.25)` 会累积，`health.penalty` 字段记录总惩罚。清除需要调 `inst.components.health:SetPenalty(0)` 或 `DeltaPenalty(-amount)` 来减少惩罚。

#### 坑 3：肉雕像被砸毁后绑定没解除——`attuner` 残留

**症状**：玩家的肉雕像被砸了，但游戏里幽灵时还能使用 REMOTERESURRECT（结果因为来源无效而失效）。

**原因**：雕像被砸毁时调了 `inst:Remove()`，但如果没有正确 unlink，玩家的 `attuner.attuned` 里还留着死亡实体的引用。

**修复**：雕像代码里有 `inst:ListenForEvent("activateresurrection", inst.Remove)` 监听复活时销毁；被砸也有 `onhammered`。但检查 `attunable:OnRemoveFromEntity` 是否调用 `UnlinkFromPlayer` 确保清理。

#### 坑 4：`respawnfromghost` 事件没有 `source`——无复活动画

**症状**：自定义复活道具触发了 `respawnfromghost`，但没有传 source，玩家复活时没有复活动画（直接变回来）。

**原因**：`DoActualRez(inst, nil, nil)` 的情况（source=nil, item=nil）在 DoActualRez 里直接跳过了所有 source 分支，没有任何复活动画——玩家瞬间变回来没有视觉反馈。

**修复**：如果是 hauntable 路径（骚扰触发），source 会是被骚扰的实体，通常有效；如果是自定义路径，确保在 `respawnfromghost` 的 data 里传入 source：

```lua
doer:PushEvent("respawnfromghost", { source = my_revival_item })
```

#### 坑 5：`REMOTERESURRECT` 不显示——`attuner:HasAttunement("remoteresurrector")` 返回 false

**症状**：建造了肉雕像、也绑定了，但幽灵时看不到 REMOTERESURRECT 选项。

**原因**：REMOTERESURRECT 的显示是通过 UI 里检查 `attuner:HasAttunement("remoteresurrector")`——这在**客户端**执行，而 `attuner.attuned` 在客户端是通过 `attunable_classified` 同步的代理对象，不是直接的实体引用。如果 attunable_classified 同步失败，客户端就看不到绑定。

**排查**：检查 `inst.components.attuner.attuned` 在服务端是否有值；确认 `attunable` 组件的网络同步正常。

---

### 19.4 小结

```
五种复活机制对比：

复活护符（amulet）:
  hauntable → HAUNT_INSTANT_REZ → respawnfromghost
  DoActualRez: source.prefab=="amulet" → Equip + amulet_rebirth
  
肉雕像（resurrectionstatue）:
  attunable("remoteresurrector") ← 玩家建造时绑定（-40血）
  REMOTERESURRECT.fn → attuner:GetAttunedTarget → respawnfromghost
  DoActualRez: source.prefab=="resurrectionstatue" → rebirth(source)
  
告密的心（reviver）:
  活人手持 → OnRespawnFromGhost: source:HasTag("reviver") → DoActualRez(nil, source_as_item)
  DoActualRez: item+no_source → reviver_rebirth(item)
  
触手石（resurrectionstone）:
  活人激活（加hauntable）→ 幽灵骚扰 → HAUNT_INSTANT_REZ → respawnfromghost
  DoActualRez: source.prefab=="resurrectionstone" → wakeup

传送门（multiplayer_portal）:
  portalrez网络变量→ 加hauntable → HAUNT_INSTANT_REZ → respawnfromghost
  DoActualRez: source:HasTag("multiplayer_portal") → DeltaPenalty(0.25) + portal_rez

通用流程：
  所有复活 → CommonActualRez（恢复饥饿/体温/精神/组件）
  → RemoveTag("playerghost")
  → PushEvent("ms_respawnedfromghost")
```

**新手核心三句**：护符/石头/传送门靠骚扰复活，肉雕像靠 REMOTERESURRECT 动作，心脏靠队友对你使用；只有传送门复活有 25% 最大血量惩罚（累积）；所有复活最终都通过 `respawnfromghost` 事件触发。

**进阶核心三句**：护符用 `SetHauntValue(HAUNT_INSTANT_REZ)` + `no_wipe_value=true` 可多次复活；肉雕像的 `attunable` 系统在建造时自动绑定（不扣血），后续再绑需要 -40 HP；`DoActualRez` 根据 `source.prefab` 分支到不同的复活动画。

**老手核心三句**：mod 复活道具最简单的是加 `reviver` tag（活人使用）或 `HAUNT_INSTANT_REZ`（幽灵骚扰）；`activateresurrection` 事件通知来源物品"你被用掉了"；自定义复活必须在 `respawnfromghost` 的 data 里传入 source，否则没有复活动画。


## 19.5 Ghostlybond——温蒂与阿比盖尔的特殊复活系统

### 本节导读

19.4 讲了五种通用的复活道具——它们对所有角色都有效。本节聚焦于**温蒂（Wendy）的专属复活系统**：她的妹妹阿比盖尔（Abigail）的幽灵与她形成了"灵魂纽带（Ghostlybond）"，这个纽带不仅决定阿比盖尔的强度，还赋予温蒂独特的复活途径——**温蒂的长明灯**（wendy_resurrectiongrave）。

**本节讲三件事**：
1. `ghostlybond` 组件——一个通用的"主人 + 幽灵伙伴"框架，阿比盖尔只是其最典型的使用案例
2. 纽带等级系统（Bond Level 1→3）——共生时间积累强度、死亡时重置
3. 温蒂的专属复活——`wendy_resurrectiongrave` 与普通肉雕像的相似与不同

> **新手**先看 19.5.1-19.5.3——理解阿比盖尔如何以"存储在 LIMBO 里的幽灵"的形式依附于温蒂、纽带等级如何随时间提升、温蒂的长明灯如何代替普通肉雕像；**进阶读者**继续看 19.5.4-19.5.6，深入 `GhostlyBond:Init` 初始化流程、`Summon/Recall` 召唤回收机制、纽带等级计时与外部时间倍率；**老手**跳到 19.5.7-19.5.9，掌握温蒂伤害转移（`health.redirect`）机制、mod 如何复用 `GhostlyBond` 组件为自定义角色制作幽灵同伴、以及五个常见坑。

---

### 19.5.1 快速入门：阿比盖尔的存储方式——LIMBO 中的幽灵伙伴

游戏在启动时就给温蒂创建了一个 `abigail` 实体，但这个实体并不是"存在于世界里"的——它被**存放在 LIMBO（场外空间）**中，以温蒂为父实体挂载：

召唤前（LIMBO 状态）：
```
温蒂（inst）
  └─ abigail（ghost）
        entity:SetParent(inst.entity)
        ghost:RemoveFromScene()
```

召唤后（世界中）：
```
温蒂（inst）              阿比盖尔（ghost）
  inst._playerlink           inst._playerlink = 温蒂
                              inst.Transform:SetPosition(世界坐标)
                              ghost:ReturnToScene()
```

这就是为什么温蒂进游戏后你就能立刻看到阿比盖尔（她一直"跟着"）——阿比盖尔从未真正消失，只是切换了存在状态。

**`GhostlyBond:Init`** 在 `wendy.lua` 里调用初始化：

```393:393:scripts/prefabs/wendy.lua
		inst.components.ghostlybond:Init("abigail", TUNING.ABIGAIL_BOND_LEVELUP_TIME)
```

- `"abigail"` → 生成的幽灵 prefab 名
- `TUNING.ABIGAIL_BOND_LEVELUP_TIME = total_day_time * 1` → **每升一级需要 1 个游戏日**的共生时间

`GhostlyBond:Init` 的实现：

```147:153:scripts/components/ghostlybond.lua
function GhostlyBond:Init(ghost_prefab, bond_levelup_time)
	self.bondleveltimer = 0
	self.bondlevelmaxtime = bond_levelup_time
	self.ghost_prefab = ghost_prefab

	self.spawnghosttask = self.inst:DoTaskInTime(0, function() self:SpawnGhost() end)
end
```

初始化后**延迟 0 秒**（下一帧）生成幽灵，并立刻放入 LIMBO（`RecallComplete`）。

> **新手记忆**：阿比盖尔从游戏开始就一直存在（存在于 LIMBO 里），不需要"召唤"一个新实体——只需要从 LIMBO 里取出放到世界。温蒂每次复活后，阿比盖尔也跟着在 LIMBO 里复位。

---

### 19.5.2 快速入门：纽带等级（Bond Level 1~3）

纽带等级决定阿比盖尔的强度：

| 等级 | 条件 | 效果 |
|------|------|------|
| Level 1 | 初始状态 / 死亡后重置 | 基础血量 `ABIGAIL_HEALTH_LEVEL1`，基础攻击 |
| Level 2 | 阿比盖尔在世界中共生 1 天（Level 1→2）| 血量提升，攻击变强 |
| Level 3 | 再共生 1 天（Level 2→3）| 最高血量 `ABIGAIL_HEALTH_LEVEL3`，最强攻击 |

等级计时逻辑（`GhostlyBond:OnUpdate`）：

```99:113:scripts/components/ghostlybond.lua
function GhostlyBond:OnUpdate(dt)
	if self.bondleveltimer == nil or self.pause then
		self.inst:StopUpdatingComponent(self)
		return
	end

	self.bondleveltimer = self.bondleveltimer + (dt * self.externalbondtimemultipliers:Get())
	if self.bondleveltimer >= self.bondlevelmaxtime then
		self:SetBondLevel(self.bondlevel + 1, self.bondleveltimer - self.bondlevelmaxtime)
	end
end
```

- 每帧累加 `dt × externalbondtimemultipliers`（外部时间倍率）
- 当计时器超过 `bondlevelmaxtime` 时，升级并把剩余时间带入下一级
- **Level 3 时 `bondleveltimer = nil`**——不再计时，`OnUpdate` 停止

**外部时间倍率**：

```262:263:scripts/prefabs/wendy.lua
inst.components.ghostlybond:SetBondTimeMultiplier("sisturn", is_active and TUNING.ABIGAIL_BOND_LEVELUP_TIME_MULT or nil)
```

- `ABIGAIL_BOND_LEVELUP_TIME_MULT = 4`——摆渡机（Sisturn）激活时，升级速度**变为 4 倍**

**死亡时重置**：

```191:198:scripts/prefabs/wendy.lua
local function ondeath(inst)
	inst.components.ghostlybond:Recall()
	inst.components.ghostlybond:PauseBonding()
end

local function onresurrection(inst)
	inst.components.ghostlybond:SetBondLevel(1)
	inst.components.ghostlybond:ResumeBonding()
end
```

温蒂死亡（或变幽灵）时：阿比盖尔被召回（进 LIMBO），纽带计时暂停，等级重置为 1。复活后从等级 1 重新开始计时。

> **新手记忆**：阿比盖尔和温蒂共生越久越强，但温蒂死亡会重置为 1 级；摆渡机（Sisturn）可以把升级速度提高 4 倍。

---

### 19.5.3 快速入门：温蒂的长明灯（wendy_resurrectiongrave）

温蒂有她自己版本的"肉雕像"——**长明灯（wendy_resurrectiongrave）**，外观是一块发光的墓碑，功能与肉雕像相似但有独立的配方和专属外观。

关键区别：

| 特性 | 肉雕像（resurrectionstatue） | 长明灯（wendy_resurrectiongrave）|
|------|---------|---------|
| 合成材料 | 木头 + 材料（常规）| 鬼花 × 10 + 切石 × 1 + 40 血 |
| 专属角色 | 所有人可建 | **仅温蒂**（`builder_skill="wendy_ghostflower_grave"`）|
| attunable tag | `"remoteresurrector"` | **`"gravestoneresurrector"`** |
| 复活动画 | `rebirth` | `wendy_gravestone_rebirth` 或 `gravestone_rebirth` |

REMOTERESURRECT 动作同时支持两种 tag：

```4058:4064:scripts/actions.lua
        local target = doer_attuner:GetAttunedTarget("remoteresurrector")
            or doer_attuner:GetAttunedTarget("gravestoneresurrector")
        if target ~= nil then
            act.doer:PushEvent("respawnfromghost", { source = target })
```

温蒂作为幽灵使用 REMOTERESURRECT 时，优先找 `remoteresurrector`（普通肉雕像），找不到再找 `gravestoneresurrector`（长明灯）。

**长明灯的 attunable 注册**：

```174:178:scripts/prefabs/wendy_resurrectiongrave.lua
    local attunable = inst:AddComponent("attunable")
    attunable:SetAttunableTag("gravestoneresurrector")
    attunable:SetOnAttuneCostFn(onattunecost)
    attunable:SetOnLinkFn(onlink)
    attunable:SetOnUnlinkFn(onunlink)
```

绑定代价和普通肉雕像相同（`EFFIGY_HEALTH_PENALTY = 40`），建造时也自动免费绑定（同肉雕像的 hack 逻辑）。

**复活时的动画分支**（来自 `DoActualRez`）：

```377:382:scripts/prefabs/player_common_extensions.lua
        elseif source.prefab == "wendy_resurrectiongrave" then
            if inst.prefab == "wendy" then
                inst.sg:GoToState("wendy_gravestone_rebirth", source)
            else
                inst.sg:GoToState("gravestone_rebirth", source)
            end
```

温蒂本人复活时使用专属的 `wendy_gravestone_rebirth` 动画（阿比盖尔参与动画）；其他角色用它复活时（如果绑定了温蒂的长明灯，理论上不可能，但代码有保护）则用通用 `gravestone_rebirth`。

> **新手记忆**：温蒂的长明灯是温蒂专属的肉雕像替代品，用鬼花合成，使用 `gravestoneresurrector` tag；复活时有特殊动画（阿比盖尔从地面升起帮助温蒂复活）。

---

### 19.5.4 进阶：GhostlyBond 的 Summon / Recall 机制

召唤（Summon）和回收（Recall）是 GhostlyBond 的核心操作。

**召唤流程**（`GhostlyBond:Summon`）：

```167:189:scripts/components/ghostlybond.lua
function GhostlyBond:Summon( summoningitem, pos )
	if self.ghost ~= nil and self.notsummoned then
		self.ghost.entity:SetParent(nil)  -- 从温蒂身上解绑
		if pos then
			self.ghost.Transform:SetPosition(pos:Get())
		else
			self.ghost.Transform:SetPosition(self.inst.Transform:GetWorldPosition())
		end
		self.ghost:ReturnToScene()  -- 重新出现在世界

		-- 同步皮肤（花的皮肤 → 阿比盖尔皮肤）
		TheSim:ReskinEntity( self.ghost.GUID, self.ghost.skinname, summoningitem.linked_skinname, summoningitem.skin_id )
		self.inst.components.pethealthbar:SetPetSkin( summoningitem.linked_skinname )

		self.notsummoned = false

		if self.onsummonfn ~= nil then
			self.onsummonfn(self.inst, self.ghost)  -- 回调：温蒂获得精神值
		end
		return true
	end
	return false
end
```

两个条件：`ghost ~= nil`（阿比盖尔存在）且 `notsummoned = true`（当前在 LIMBO 里）。召唤时还同步了皮肤（让阿比盖尔匹配玩家装备的花的皮肤）。

**召唤完成**（`GhostlyBond:SummonComplete`）：

```192:200:scripts/components/ghostlybond.lua
function GhostlyBond:SummonComplete()
	self.notsummoned = false
	self.summoned = true

	if self.onsummoncompletefn ~= nil then
		self.onsummoncompletefn(self.inst, self.ghost)
	end
	self.inst:PushEvent("ghostlybond_summoncomplete", self.ghost)
end
```

这个函数在召唤动画完成后调用——推送 `ghostlybond_summoncomplete` 事件通知花（abigail_flower）刷新 UI 状态。

**回收流程**（`GhostlyBond:Recall`）：

```202:212:scripts/components/ghostlybond.lua
function GhostlyBond:Recall(was_killed)
	if self.ghost ~= nil and self.summoned and not self.inst.sg:HasStateTag("dissipate") then
		self.summoned = false

		if self.onrecallfn ~= nil then
			self.onrecallfn(self.inst, self.ghost, was_killed)
		end
		return true
	end
end
```

**回收完成**（`GhostlyBond:RecallComplete`）：

```214:225:scripts/components/ghostlybond.lua
function GhostlyBond:RecallComplete()
    self.ghost:RemoveFromScene()  -- 从世界消失
	self.ghost.entity:SetParent(self.inst.entity)  -- 重新挂载到温蒂
	self.ghost.Transform:SetPosition(0, 0, 0)  -- 本地坐标归零

	self.summoned = false
	self.notsummoned = true

	if self.onrecallcompletefn ~= nil then
		self.onrecallcompletefn(self.inst, self.ghost)
	end
	self.inst:PushEvent("ghostlybond_recallcomplete", self.ghost)
end
```

**状态标志对比**：

| 状态 | `summoned` | `notsummoned` | 实际位置 |
|------|-----------|--------------|---------|
| 召唤进行中 | false | false | 正在出现 |
| 在世界中 | **true** | false | 世界中 |
| 在 LIMBO 中 | false | **true** | 挂载到温蒂 |

两个字段分别触发不同的网络同步（`ghostfriend_summoned` / `ghostfriend_notsummoned` tag）。

---

### 19.5.5 进阶：SetBondLevel 与等级变化事件

```131:145:scripts/components/ghostlybond.lua
function GhostlyBond:SetBondLevel(level, time, isloading)
	time = time or 0
	local prev_level = self.bondlevel
	self.bondlevel = math.min(level, self.maxbondlevel)
	self.bondleveltimer = level < self.maxbondlevel and time or nil
	if self.bondleveltimer ~= nil and not self.paused then
		self.inst:StartUpdatingComponent(self)
	end
	if self.bondlevel ~= prev_level then
		if self.onbondlevelchangefn ~= nil then
			self.onbondlevelchangefn(self.inst, self.ghost, level, prev_level, isloading)
		end
		self.inst:PushEvent("ghostlybond_level_change", {ghost = self.ghost, level = level, prev_level = prev_level, isloading = isloading})
	end
end
```

**关键细节**：
- `bondlevel = math.min(level, maxbondlevel = 3)`——不能超过 3 级
- `bondleveltimer = level < maxbondlevel and time or nil`——到达最高级时 timer = nil，停止计时
- 只有等级**真正改变**时才触发 `ghostlybond_level_change` 事件

**温蒂对等级变化的响应**：

```201:208:scripts/prefabs/wendy.lua
local function ghostlybond_onlevelchange(inst, ghost, level, prev_level, isloading)
	inst._bondlevel:set(level)  -- 同步到网络（用于 UI 显示）

	if not isloading and inst.components.talker ~= nil and level > 1 then
		inst.components.talker:Say(GetString(inst, "ANNOUNCE_GHOSTLYBOND_LEVELUP", "LEVEL"..tostring(level)))
		OnBondLevelDirty(inst)
	end
end
```

等级提升时：
1. 更新 `_bondlevel` 网络变量（同步给所有客户端用于 UI）
2. 温蒂说一句台词公告升级
3. 阿比盖尔进入 `ghostlybond_levelup` 状态播放升级动画

**阿比盖尔对等级变化的响应**：

```352:357:scripts/prefabs/abigail.lua
local function on_ghostlybond_level_change(inst, player, data)
	if not inst.inlimbo and data.level > 1 and not inst.sg:HasStateTag("busy") and (inst.components.health == nil or not inst.components.health:IsDead()) then
		inst.sg:GoToState("ghostlybond_levelup", {level = data.level})
	end

	UpdateGhostlyBondLevel(inst, data.level)
end
```

不在 LIMBO 且不在 busy 状态时，阿比盖尔播放升级动画，并更新自身属性（`UpdateGhostlyBondLevel` 会调整血量/攻击等）。

---

### 19.5.6 进阶：温蒂的伤害转移——`health.redirect` + `ghostlybond_redirect` tag

温蒂有一个技能树技能——激活后，温蒂受到的伤害会**被转移到阿比盖尔身上**（代替温蒂承受）。这通过 `health.redirect` 实现：

```348:356:scripts/prefabs/wendy.lua
local function redirect_to_abigail(inst, amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb)
	if inst.components.ghostlybond ~= nil
			and inst.components.ghostlybond.ghost ~= nil
			and not inst.components.ghostlybond.ghost:IsInLimbo()
			and inst:HasTag("ghostlybond_redirect") then
		inst.components.ghostlybond.ghost.components.health:DoDelta(amount)
		return true  -- 返回 true = 原始伤害被完全消耗
	end
end
```

安装方式：

```398:398:scripts/prefabs/wendy.lua
		inst.components.health.redirect = redirect_to_abigail
```

**触发条件**：
1. 温蒂的 `ghostlybond` 组件存在
2. 阿比盖尔存在（不为 nil）
3. 阿比盖尔不在 LIMBO
4. 温蒂有 `"ghostlybond_redirect"` tag（由技能树控制添加/移除）

当所有条件满足时，给温蒂的伤害**直接转给阿比盖尔的 health:DoDelta**，函数返回 true（取消温蒂的原始伤害处理）。

**温蒂的精神免疫**（独特特性）：

```370:371:scripts/prefabs/wendy.lua
	inst.components.sanity:AddSanityAuraImmunity("ghost", inst)
	inst.components.sanity:SetPlayerGhostImmunity(true, inst)
```

- `AddSanityAuraImmunity("ghost")` → 幽灵生物（阿比盖尔本身是幽灵）不对温蒂产生精神负面光环
- `SetPlayerGhostImmunity(true)` → **幽灵玩家不对温蒂造成精神消耗**

这就是为什么其他玩家变成幽灵在温蒂旁边转悠，温蒂的精神不会下降——她已经习惯了死亡的存在。

---

### 19.5.7 老手：GhostlyBond 作为通用框架——为 mod 角色添加幽灵同伴

`ghostlybond` 组件是通用的"主人 + 幽灵伙伴"框架，可以用于**任何 mod 角色**：

#### 第一步：添加组件并初始化

```lua
-- 在角色的 master_postinit 里
inst:AddComponent("ghostlybond")

-- 注册回调（可选）
inst.components.ghostlybond.onbondlevelchangefn = my_onlevelchange
inst.components.ghostlybond.onsummonfn = my_onsummon
inst.components.ghostlybond.onrecallfn = my_onrecall
inst.components.ghostlybond.onsummoncompletefn = my_onsummoncomplete
inst.components.ghostlybond.changebehaviourfn = my_changebehaviour

-- 初始化（生成幽灵 prefab + 开始计时）
inst.components.ghostlybond:Init("my_ghost_prefab", TUNING.TOTAL_DAY_TIME * 0.5)
```

#### 第二步：对应的幽灵 prefab 要调 `LinkToPlayer`

在幽灵的 `linktoplayer` 函数（或 `fn`）里：

```lua
-- 幽灵 prefab 的 fn 里（模仿 abigail.lua:405-419）
local function linktoplayer(inst, player)
    inst.persists = false
    inst._playerlink = player
    -- ...（跟随 AI、监听事件）
end

-- 幽灵 prefab 需要暴露 LinkToPlayer 方法
inst.LinkToPlayer = linktoplayer
```

`GhostlyBond:SpawnGhost` 会调用 `ghost:LinkToPlayer(self.inst)` —— 幽灵 prefab 必须有这个函数。

#### 第三步：召唤/回收工具（需要一个召唤道具）

模仿 `abigail_flower.lua`，给主角提供一个道具，道具的使用触发 `ghostlybond:Summon(summoningitem, pos)` 或 `ghostlybond:Recall()`。

**注意**：如果不需要皮肤同步，`Summon` 里的 `TheSim:ReskinEntity` 会报错（因为 `summoningitem.linked_skinname` 可能为 nil）——需要确保传入的 item 有 `linked_skinname` 属性，或者重写 `Summon` 函数跳过皮肤同步。

#### 第四步：监听生命周期事件

```lua
-- 在角色身上监听
inst:ListenForEvent("death", ondeath_fn)
inst:ListenForEvent("ms_becameghost", ondeath_fn)
inst:ListenForEvent("ms_respawnedfromghost", onresurrection_fn)

-- ondeath_fn
local function ondeath_fn(inst)
    inst.components.ghostlybond:Recall()
    inst.components.ghostlybond:PauseBonding()
end

-- onresurrection_fn
local function onresurrection_fn(inst)
    inst.components.ghostlybond:SetBondLevel(1)
    inst.components.ghostlybond:ResumeBonding()
end
```

---

### 19.5.8 老手：阿比盖尔的死亡处理——`_ghost_death` 回调

阿比盖尔被杀死（血量为 0）时，GhostlyBond 会响应：

```14:17:scripts/components/ghostlybond.lua
local function _ghost_death(self)
	self:SetBondLevel(1)
	self:Recall(true)
end
```

- **等级重置为 1**——阿比盖尔死亡后，纽带强度归零，下次召唤要重新培养
- **`Recall(was_killed = true)`**——触发 `onrecallfn(inst, ghost, true)`

温蒂的 `ghostlybond_onrecall` 响应：

```216:228:scripts/prefabs/wendy.lua
local function ghostlybond_onrecall(inst, ghost, was_killed)
	if inst.migration == nil then
		if inst.components.sanity ~= nil then
			inst.components.sanity:DoDelta(was_killed and (-TUNING.SANITY_MED * 2) or -TUNING.SANITY_MED)
		end
		-- 说话台词
		inst.components.talker:Say(GetString(inst, was_killed and "ANNOUNCE_ABIGAIL_DEATH" or "ANNOUNCE_ABIGAIL_RETRIEVE"))
	end
	inst.components.ghostlybond.ghost.sg:GoToState("dissipate")
end
```

**精神惩罚区分**：
- 阿比盖尔被杀死（`was_killed = true`）→ 精神 `-SANITY_MED × 2`（双倍惩罚）
- 主动回收（`was_killed = false`）→ 精神 `-SANITY_MED`（正常惩罚）

---

### 19.5.9 老手：五个常见坑

#### 坑 1：mod 幽灵 prefab 没有 `LinkToPlayer` 方法——报错

**症状**：`GhostlyBond:SpawnGhost` 调用 `ghost:LinkToPlayer(self.inst)` 报错 "attempt to call nil value"。

**原因**：mod 的幽灵 prefab 没有定义 `LinkToPlayer` 函数（这不是组件，而是 prefab 上直接挂的函数）。

**修复**：在 mod 幽灵 prefab 的 `fn` 里定义：
```lua
inst.LinkToPlayer = function(inst, player)
    -- 建立跟随关系、设置 _playerlink 等
end
```

#### 坑 2：召唤时皮肤同步报错——`summoningitem.linked_skinname` 为 nil

**症状**：调用 `ghostlybond:Summon(item, pos)` 时报错，因为 `item.linked_skinname` 不存在。

**原因**：`GhostlyBond:Summon` 里调用 `TheSim:ReskinEntity( ghost.GUID, ghost.skinname, summoningitem.linked_skinname, summoningitem.skin_id )`——`linked_skinname` 是皮肤系统特有字段，普通 mod 道具没有。

**修复**：对于不需要皮肤的 mod，在调用 `Summon` 前临时给 item 加上空值，或者直接写自定义召唤逻辑绕开 GhostlyBond:Summon，改成手动调 `SpawnGhost` 和 `SummonComplete`。

#### 坑 3：纽带等级计时在 LIMBO 状态下也在积累

**症状**：温蒂手动回收阿比盖尔后，发现计时器还在走。

**原因**：`GhostlyBond:OnUpdate` 的计时**不区分阿比盖尔是否在世界里**——只要计时器不为 nil 且不暂停，就一直计时。

**这是设计**，不是 bug——温蒂和阿比盖尔的纽带是基于"共存时间"，不是"阿比盖尔在世界里的时间"。如果你的 mod 想按照"在世界里的时间"计时，需要在 Summon/Recall 时手动调 `PauseBonding` / `ResumeBonding`。

#### 坑 4：`ghostlybond_level_change` 事件在加载存档时也触发

**症状**：监听 `ghostlybond_level_change` 的代码在加载存档时执行了（如弹出 UI 通知），实际上是恢复状态不是真正升级。

**原因**：`GhostlyBond:OnLoad` 调 `SetBondLevel`，如果等级和默认值（1）不同就会触发事件。`data.isloading = true` 是区分标志。

**修复**：在事件回调里检查 `data.isloading`：

```lua
inst:ListenForEvent("ghostlybond_level_change", function(inst, data)
    if data.isloading then return end  -- 加载恢复，不处理
    -- 真实升级逻辑...
end)
```

#### 坑 5：`ghostlybond_redirect` tag 没有效果——皮肤系统标签冲突

**症状**：给 mod 角色设置了 `health.redirect = redirect_to_abigail`，并手动加了 `"ghostlybond_redirect"` tag，但伤害转移没有生效。

**原因**：检查 `redirect_to_abigail` 函数里还需要 `inst.components.ghostlybond.ghost` 不为 nil 且**不在 LIMBO**（`not ghost:IsInLimbo()`）。如果阿比盖尔在 LIMBO 里，伤害不会转移。

**修复**：召唤阿比盖尔后，确认 `inst.components.ghostlybond.ghost.inlimbo == false`（`not ghost:IsInLimbo()`）。

---

### 19.5 小结

```
GhostlyBond 系统架构：
  温蒂（主人）
    └─ ghostlybond 组件
         └─ ghost = abigail 实体（始终存在，LIMBO 中或世界中）
         └─ bondlevel (1-3)：随时间累积，死亡重置
         └─ bondleveltimer：计时到 bondlevelmaxtime 升级

状态机：
  LIMBO（notsummoned=true）
    ↓ Summon(item, pos)
  出现中（summoned=false, notsummoned=false）
    ↓ SummonComplete()
  世界中（summoned=true）
    ↓ Recall(was_killed?)
  回收中（summoned=false）
    ↓ RecallComplete()
  LIMBO（notsummoned=true）

事件：
  ghostlybond_level_change  → 等级变化（升级/重置）
  ghostlybond_summoncomplete → 召唤动画完成
  ghostlybond_recallcomplete → 回收动画完成

温蒂特有：
  health.redirect → 转移伤害到阿比盖尔（需要 ghostlybond_redirect tag）
  sanity immunity from ghost 玩家 + ghost aura
  wendy_resurrectiongrave（attunable tag = "gravestoneresurrector"）
```

**新手核心三句**：阿比盖尔从游戏开始就在 LIMBO 里（不需要"创建"），召唤 = 从 LIMBO 取出，回收 = 放回 LIMBO；纽带等级 1→3 随共生时间积累，死亡重置；温蒂的长明灯使用 `gravestoneresurrector` tag，和普通肉雕像并行支持 REMOTERESURRECT。

**进阶核心三句**：`GhostlyBond:OnUpdate` 的计时不区分 LIMBO/世界，始终累积（除非调 PauseBonding）；`SetBondLevel` 达到 maxbondlevel(3) 后 timer = nil，停止计时更新；等级变化通过 `ghostlybond_level_change` 事件通知，`data.isloading` 区分加载恢复和真实升级。

**老手核心三句**：mod 幽灵 prefab 必须有 `LinkToPlayer(inst, player)` 函数；`GhostlyBond:Summon` 需要 item 有 `linked_skinname` 和 `skin_id` 字段（皮肤系统），不需要皮肤的 mod 要绕开这部分；阿比盖尔被杀时触发 `Recall(true)` 回调，精神惩罚是主动回收的 2 倍。


## 19.6 Revivable / Rez 组件——复活的技术实现

### 本节导读

19.1-19.5 我们讲了从普通死亡到幽灵状态、再到各种复活道具的全流程——这些都是"有幽灵模式（ghostenabled = true）"的情况。但饥荒还有另一套死亡机制：**尸体可复活（Revivable Corpse）**——玩家死亡后不变幽灵，而是**维持尸体状态**，等待队友手动复活。

这套机制用于特殊游戏模式（熔岩竞技场 lavaarena、沼泽商旅 quagmire 等），通过 `revivablecorpse` 组件实现。与幽灵系统的关键区别：

| 特性 | 幽灵模式（ghostenabled）| 尸体模式（revivable_corpse）|
|-----|----------------------|--------------------------|
| 死后形态 | 幽灵（可自由行动）| 尸体（无法移动）|
| 复活条件 | 自己找道具/骚扰 | **队友手动施救** |
| 复活动作 | REMOTERESURRECT / HAUNT | `REVIVE_CORPSE`（长按进度条）|
| 恢复血量 | RESURRECT_HEALTH (50) | `revive_health_percet × maxhp` |
| 游戏模式 | 普通生存 | lavaarena / quagmire |

> **新手**先看 19.6.1-19.6.3——了解尸体系统与幽灵系统的区别、`REVIVE_CORPSE` 进度条动作的触发条件、复活后血量如何计算；**进阶读者**继续看 19.6.4-19.6.7，深入 `revivablecorpse` 组件的全部 API、`OnMakePlayerCorpse` 与 `OnRespawnFromPlayerCorpse` 的完整流程、游戏模式属性如何控制两套系统切换；**老手**跳到 19.6.8-19.6.10，了解复活速度倍率（revivespeedmult 与 corpsereviver）、mod 扩展接口（自定义复活条件）、五个常见坑。

---

### 19.6.1 快速入门：尸体系统与幽灵系统的切换——`GetGhostEnabled()`

在加载玩家时，游戏根据游戏模式属性决定用哪套死亡系统：

```2639:2650:scripts/prefabs/player_common.lua
        inst:ListenForEvent("death", ex_fns.OnPlayerDeath)
        if inst.ghostenabled then
            inst:ListenForEvent("makeplayerghost", ex_fns.OnMakePlayerGhost)
            inst:ListenForEvent("respawnfromghost", ex_fns.OnRespawnFromGhost)
            inst:ListenForEvent("ghostdissipated", ex_fns.OnPlayerDied)
        elseif inst.components.revivablecorpse ~= nil then
            inst:ListenForEvent("respawnfromcorpse", ex_fns.OnRespawnFromPlayerCorpse)
            inst:ListenForEvent("playerdied", ex_fns.OnMakePlayerCorpse)
        else
            inst:ListenForEvent("playerdied", ex_fns.OnPlayerDied)
        end
```

**三条死亡路径**：
1. `inst.ghostenabled = true` → 幽灵系统（普通生存模式）
2. `inst.components.revivablecorpse ~= nil` → 尸体系统（特殊竞技模式）
3. 否则 → 直接消失（荒野模式等，`OnPlayerDied`）

`GetGhostEnabled()` 的逻辑：

```290:292:scripts/gamemodes.lua
function GetGhostEnabled()
    return GetWorldSetting("ghost_enabled", true) and not GetGameModeProperty("revivable_corpse")
    --revivablecorpse forces ghosts to be disabled.
end
```

当游戏模式有 `revivable_corpse = true` 时，`GetGhostEnabled()` 强制返回 false——**两套系统互斥**。

**哪些游戏模式使用 `revivable_corpse`**：

```70:70:scripts/gamemodes.lua
-- lavaarena
        revivable_corpse = true,
```

```111:111:scripts/gamemodes.lua
-- quagmire
        revivable_corpse = true,
```

> **新手记忆**：普通生存 = 幽灵系统；熔岩竞技场/沼泽商旅 = 尸体系统。两套系统**完全互斥**，不能同时存在。

---

### 19.6.2 快速入门：尸体状态——"corpse" tag 与视觉表现

当玩家死亡进入尸体状态时，`OnMakePlayerCorpse` 被调用：

```774:800:scripts/prefabs/player_common_extensions.lua
local function OnMakePlayerCorpse(inst, data)
    if inst:HasTag("corpse") then
        return  -- 防止重复
    elseif data == nil or not data.loading then
        -- 公告死亡
        TheNet:AnnounceDeath(announcement_string, inst.entity)
    end

    RemovePhysicsColliders(inst)  -- 移除物理碰撞

    inst.components.revivablecorpse:SetCorpse(true)  -- 添加 "corpse" tag
    inst:RemoveTag("NOCLICK")  -- 允许点击（让队友能选中并复活）

    CommonPlayerDeath(inst)  -- 通用死亡状态重置（同幽灵系统）

    inst.player_classified:SetGhostMode(true)  -- 进入"幽灵模式" UI 状态

    inst:PushEvent("ms_becameghost", { corpse = true })  -- 通知变成了尸体
    -- ...
end
```

**关键细节**：
- `RemovePhysicsColliders(inst)` → 移除物理碰撞，其他实体可以穿过尸体
- `SetCorpse(true)` → 给玩家添加 `"corpse"` tag
- `"NOCLICK"` 被移除 → **尸体可以被点击**（区别于其他死亡路径会加 NOCLICK）
- `player_classified:SetGhostMode(true)` → 即使是尸体，也设置了 ghost mode（用于客户端 UI 响应）
- `ms_becameghost` 的 data 里带 `corpse = true` → 可以区分"变成了幽灵"vs"变成了尸体"

**尸体的动画状态**：
从 SGwilson 的 `"death"` 状态（19.1 已讲）可以看到：如果 `inst.components.revivablecorpse ~= nil`，死亡动画 `onenter` 会播放 `"death2"` 动画（不是标准 `"death"`），并在动画结束后 `sg:GoToState("corpse")` 进入尸体等待状态。

> **新手记忆**：尸体玩家有 `"corpse"` tag，被移除了物理碰撞，但可以被点击。即使是尸体，客户端 UI 也以"幽灵模式"处理（因为玩家同样失去了控制权）。

---

### 19.6.3 快速入门：`REVIVE_CORPSE` 动作——按住复活进度条

当活着的队友靠近有 `revivablecorpse` 组件且有 `"corpse"` tag 的玩家时，会出现"复活"选项：

```736:739:scripts/componentactions.lua
        revivablecorpse = function(inst, doer, actions, right)
            if inst.components.revivablecorpse:CanBeRevivedBy(doer) then
                table.insert(actions, ACTIONS.REVIVE_CORPSE)
            end
        end,
```

`REVIVE_CORPSE` 动作的定义：

```485:485:scripts/actions.lua
    REVIVE_CORPSE = Action({ rmb=false, actionmeter=true }),
```

关键参数 `actionmeter=true`——这个动作需要**持续按住**，有进度条，不是瞬间完成的。

**`CanBeRevivedBy` 的检查**：

```24:27:scripts/components/revivablecorpse.lua
function RevivableCorpse:CanBeRevivedBy(reviver)
    return self.inst:HasTag("corpse")
        and (self.canberevivedbyfn == nil or self.canberevivedbyfn(self.inst, reviver))
end
```

两个条件：
1. 目标有 `"corpse"` tag（正在尸体状态）
2. 自定义过滤函数通过（如果有）

**`ACTIONS.REVIVE_CORPSE.fn`** 在动作完成时执行：

```4068:4076:scripts/actions.lua
ACTIONS.REVIVE_CORPSE.fn = function(act)
    if act.doer ~= nil and act.target ~= nil and act.target.components.revivablecorpse ~= nil then
        --Silent fail
        if act.target.components.revivablecorpse:CanBeRevivedBy(act.doer) then
            act.target.components.revivablecorpse:Revive(act.doer)
        end
        return true
    end
end
```

注意 `--Silent fail`——如果期间尸体已经被别人复活了（`CanBeRevivedBy` 返回 false），动作依然返回 true（不报错），只是静默跳过 `Revive`。

**复活动作的持续时间**：

进入 `revivecorpse` 状态后跳转到 `dolongaction`，时间计算：

```7992:8003:scripts/stategraphs/SGwilson.lua
    State{
        name = "revivecorpse",

        onenter = function(inst)
            inst.components.talker:Say(GetString(inst, "ANNOUNCE_REVIVING_CORPSE"))
            local buffaction = inst:GetBufferedAction()
            local target = buffaction ~= nil and buffaction.target or nil
            inst.sg:GoToState("dolongaction",
                TUNING.REVIVE_CORPSE_ACTION_TIME *
                (inst.components.corpsereviver ~= nil and inst.components.corpsereviver:GetReviverSpeedMult(target) or 1) *
                (target ~= nil and target.components.revivablecorpse ~= nil and target.components.revivablecorpse:GetReviveSpeedMult(inst) or 1)
            )
        end,
    },
```

**复活时间 = 6秒 × 施救者速度倍率 × 目标速度倍率**：
- `REVIVE_CORPSE_ACTION_TIME = 6` 秒（基础时间）
- `corpsereviver:GetReviverSpeedMult(target)` → 施救者的速度倍率（通常为 1，特殊情况下更快）
- `revivablecorpse:GetReviveSpeedMult(inst)` → 目标的速度倍率（也通常为 1）

> **新手记忆**：复活需要**持续按住 6 秒**（进度条动作），期间施救者不能移动；期间目标尸体保持 `"corpse"` tag；动作完成后调用 `Revive(doer)`。

---

### 19.6.4 进阶：`RevivableCorpse` 组件的完整 API

`revivablecorpse` 组件有两部分——客户端 + 服务端共用部分 vs 仅服务端部分：

```1:15:scripts/components/revivablecorpse.lua
--This component runs on client as well
local RevivableCorpse = Class(function(self, inst)
    self.inst = inst

    --Common
    self.ismastersim = TheWorld.ismastersim
    --self.canberevivedbyfn = nil

    --Master simulation
    if self.ismastersim then
        self.revive_health_percet = .5
        self.revivespeedmult = 1
        --self.tagmults = nil
    end
end)
```

**全部方法**：

| 方法 | 端 | 说明 |
|------|---|------|
| `SetCanBeRevivedByFn(fn)` | 共用 | 设置自定义"能否被复活"判定函数 |
| `CanBeRevivedBy(reviver)` | 共用 | 判断特定玩家能否复活此尸体 |
| `SetReviveSpeedMult(mult)` | 仅服务端 | 设置复活速度倍率（base = 1）|
| `SetReviveSpeedMultForTag(tag, mult)` | 仅服务端 | 为带特定 tag 的施救者设置速度倍率 |
| `GetReviveSpeedMult(reviver)` | 仅服务端 | 获取当前有效速度倍率（考虑 tag 加速）|
| `SetCorpse(bool)` | 仅服务端 | 添加/移除 "corpse" tag |
| `Revive(reviver)` | 仅服务端 | 触发 `respawnfromcorpse` 事件 |
| `SetReviveHealthPercent(percent)` | 仅服务端 | 设置复活后血量比例（0-1）|
| `GetReviveHealthPercent()` | 仅服务端 | 获取复活后血量比例 |

**`SetReviveSpeedMultForTag` 使用示例**：

```lua
-- 让有 "medic" tag 的玩家复活速度快 2 倍
inst.components.revivablecorpse:SetReviveSpeedMultForTag("medic", 0.5)
-- 倍率 < 1 表示更快（时间 × 0.5 = 快 2 倍）
```

实现细节：

```53:63:scripts/components/revivablecorpse.lua
function RevivableCorpse:GetReviveSpeedMult(reviver)
    local mult = self.revivespeedmult
    if self.tagmults ~= nil then
        for k, v in pairs(self.tagmults) do
            if reviver:HasTag(k) then
                mult = mult * v
            end
        end
    end
    return mult
end
```

复活速度倍率是**可叠加**的——如果施救者同时有多个加速 tag，效果相乘。

---

### 19.6.5 进阶：`OnRespawnFromPlayerCorpse` 完整流程

当尸体被复活（`respawnfromcorpse` 事件触发）时：

```749:771:scripts/prefabs/player_common_extensions.lua
local function OnRespawnFromPlayerCorpse(inst, data)
    if not inst:HasTag("corpse") then
        return  -- 防止重复
    end

    -- 清除死亡信息
    inst.deathclientobj = nil
    inst.deathcause = nil
    inst.deathpkname = nil
    inst.deathbypet = nil

    inst:DoTaskInTime(0, DoActualRezFromCorpse, data and data.source or nil)

    -- 设置"谁复活了我"的信息（用于公告）
    inst.rezsource =
        data ~= nil and (
            (data.source ~= nil and not data.source:HasTag("reviver") and data.source.name) or
            (data.user ~= nil and data.user:GetDisplayName())
        ) or STRINGS.NAMES.SHENANIGANS
end
```

紧接着调用 `DoActualRezFromCorpse`：

```433:479:scripts/prefabs/player_common_extensions.lua
local function DoActualRezFromCorpse(inst, source)
    if not inst:HasTag("corpse") then
        return
    end

    -- 1. 生成复活特效
    SpawnPrefab("lavaarena_player_revive_from_corpse_fx").entity:SetParent(inst.entity)

    inst.components.inventory:Hide()
    inst:PushEvent("ms_closepopups")

    -- 2. 切换回普通状态图
    inst:SetStateGraph("SGwilson")
    inst.sg:GoToState("corpse_rebirth")  -- 从尸体状态起身动画

    -- 3. 关闭幽灵模式
    inst.player_classified:SetGhostMode(false)

    -- 4. 计算复活血量
    local respawn_health_precent = inst.components.revivablecorpse ~= nil
        and inst.components.revivablecorpse:GetReviveHealthPercent() or 1

    if source ~= nil and source:IsValid() then
        if source.components.talker ~= nil then
            source.components.talker:Say(GetString(source, "ANNOUNCE_REVIVED_OTHER_CORPSE"))
        end
        -- 施救者 corpsereviver 组件可以额外增加复活血量
        if source.components.corpsereviver ~= nil then
            respawn_health_precent = respawn_health_precent
                + source.components.corpsereviver:GetAdditionalReviveHealthPercent()
        end
    end

    -- 5. 通用状态还原
    CommonActualRez(inst)

    -- 6. 设置复活血量（比例 × 最大血量）
    inst.components.health:SetCurrentHealth(
        inst.components.health:GetMaxWithPenalty() * math.clamp(respawn_health_precent, 0, 1)
    )
    inst.components.health:ForceUpdateHUD(true)

    -- 7. 移除 corpse 状态
    inst.components.revivablecorpse:SetCorpse(false)

    -- 8. 推送复活完成事件（注意：corpse=true 标记）
    inst:PushEvent("ms_respawnedfromghost", { corpse = true, reviver = source })
end
```

**复活血量计算**：

```
复活血量 = (revivablecorpse.revive_health_percet + corpsereviver额外) × maxhp
         = (0.5 + additional) × maxhp
         = 最大血量的 50%（默认），可能被 corpsereviver 提高
```

**与幽灵复活的对比**：

| 对比项 | 幽灵复活（DoActualRez）| 尸体复活（DoActualRezFromCorpse）|
|--------|----------------------|----------------------------------|
| 触发事件 | `respawnfromghost` | `respawnfromcorpse` |
| 复活特效 | `die_fx` | `lavaarena_player_revive_from_corpse_fx` |
| 复活动画 | 多种（根据来源）| `corpse_rebirth` |
| 血量设置 | ghost health（50 HP）| `revive_health_percet × maxhp` |
| 结束事件 | `ms_respawnedfromghost`（corpse=false）| `ms_respawnedfromghost`（**corpse=true**）|

---

### 19.6.6 进阶：`corpsepersistmanager` 组件——尸体的超时管理

在特殊游戏模式里，尸体不能永远等待，通常有超时机制。这由 `corpsepersistmanager` 组件实现（`scripts/components/corpsepersistmanager.lua`）：

```lua
-- 简化描述
-- 该组件挂在世界（TheWorld）上
-- 追踪所有处于 "corpse" 状态的玩家
-- 当某个玩家尸体超时（竞技场回合结束等），自动触发清理
```

此组件是 `lavaarena` 等特殊游戏模式的内部实现，普通 mod 开发一般不需要直接操作它。它的核心作用是：**在回合结束/超时时，强制处理所有尸体状态的玩家**（要么强制复活，要么进入下一轮重置）。

---

### 19.6.7 进阶：游戏模式属性的完整复活控制

理解游戏模式如何控制复活行为，需要关注几个关键属性：

```lua
-- gamemodes.lua 里每个游戏模式的属性
ghost_enabled = true/false    -- 是否允许幽灵模式
revivable_corpse = true/false -- 是否使用尸体复活系统
portal_rez = true/false       -- 是否允许传送门复活
ghost_sanity_drain = true/false -- 幽灵是否对活人造成精神消耗
```

**生存模式（survival）的默认配置**：

```14:28:scripts/gamemodes.lua
    survival =
    {
        -- ...
        ghost_sanity_drain = true,
        ghost_enabled = true,
        portal_rez = false,
        -- ...
    },
```

**熔岩竞技场（lavaarena）的配置**：

```59:74:scripts/gamemodes.lua
    lavaarena =
    {
        -- ...
        ghost_sanity_drain = false,
        ghost_enabled = false,
        revivable_corpse = true,
        spectator_corpse = true,
        portal_rez = false,
        -- ...
        no_hunger = true,
        no_sanity = true,
    },
```

`GetGhostEnabled()` 检查 `ghost_enabled` AND NOT `revivable_corpse`——确保两套系统互斥。

**mod 设置自己的游戏模式**：

如果 mod 想实现一个"队友复活"式的游戏模式，可以通过 `AddGameMode` 注册游戏模式，设置 `revivable_corpse = true` 并给玩家添加 `revivablecorpse` 组件。

---

### 19.6.8 老手：corpsereviver 组件——施救者专属加速

`corpsereviver` 组件可以挂在施救者身上，给他们加快复活速度或增加复活血量的能力。这个组件不在基础脚本里，是事件特定的扩展点。

从代码引用可以看出它的接口：

```lua
-- 在 SGwilson.lua:8000 引用
inst.components.corpsereviver:GetReviverSpeedMult(target)

-- 在 player_common_extensions.lua:456 引用
source.components.corpsereviver:GetAdditionalReviveHealthPercent()
```

如果你的 mod 想让某些角色/装备加快复活速度，可以创建一个自定义组件并实现这两个方法：

```lua
-- 自定义 corpsereviver 组件（示例）
local Corpsereviver = Class(function(self, inst)
    self.inst = inst
    self.speed_mult = 0.5      -- 速度倍率（0.5 = 快 2 倍）
    self.health_bonus = 0.2   -- 额外复活血量（20%）
end)

function Corpsereviver:GetReviverSpeedMult(target)
    return self.speed_mult     -- 时间 × 0.5 = 比基础快 2 倍
end

function Corpsereviver:GetAdditionalReviveHealthPercent()
    return self.health_bonus   -- 复活血量额外 + 20%
end

return Corpsereviver
```

然后在角色或装备的某个激活条件下加上这个组件：

```lua
inst:AddComponent("corpsereviver")
inst.components.corpsereviver.speed_mult = 0.5
```

---

### 19.6.9 老手：mod 给玩家添加 `revivablecorpse` 组件

如果 mod 想让玩家在普通游戏模式里也有"尸体可复活"的能力，需要：

#### 第一步：给玩家添加组件

```lua
-- 在角色的 master_postinit 或通过 AddClassPostConstruct
AddPlayerPostInit(function(inst)
    if TheWorld.ismastersim then
        inst:AddComponent("revivablecorpse")
        inst.components.revivablecorpse:SetReviveHealthPercent(0.5)
        -- 可选：设置谁能复活
        inst.components.revivablecorpse:SetCanBeRevivedByFn(function(corpse, reviver)
            return reviver:HasTag("player")  -- 所有玩家都能复活
        end)
    end
end)
```

#### 第二步：关闭幽灵模式

`revivablecorpse` 系统需要 `inst.ghostenabled = false`——否则死亡时走的是幽灵路径，不会进入尸体状态。

但是**直接修改 `inst.ghostenabled` 可能有副作用**——这个字段在玩家创建时根据 `GetGhostEnabled()` 设置，改变它会影响所有幽灵相关的功能。

更安全的做法是**只对特定情境（如特殊技能激活时）临时切换**，而不是全局关闭幽灵模式。

#### 第三步：监听复活事件

```lua
inst:ListenForEvent("respawnfromcorpse", ex_fns.OnRespawnFromPlayerCorpse)
inst:ListenForEvent("playerdied", ex_fns.OnMakePlayerCorpse)
```

但要小心——这些 ex_fns 函数是在 `player_common.lua` 里的特定逻辑体系里注册的，如果改变监听组合可能破坏死亡管线。

**建议**：不要在普通生存模式里混用两套系统；如果想做队友复活 mod，考虑使用独立的"救援动作"而不是修改底层死亡系统。

---

### 19.6.10 老手：五个常见坑

#### 坑 1：给玩家添加了 `revivablecorpse` 但死后仍然变成幽灵

**原因**：`inst.ghostenabled` 仍然为 true，`player_common.lua` 注册的是幽灵路径监听器而不是尸体路径监听器。

**原理回顾**：监听器在玩家 `master_postinit` 时**一次性注册**，之后不会重新判断——你在之后添加 `revivablecorpse` 组件不会改变已注册的监听器。

**修复**：需要**在玩家初始化时**（`player_common.lua` 的 `master_postinit`）就有 `revivablecorpse` 组件，或者使用 `AddClassPostConstruct` 替换监听器。

#### 坑 2：`CanBeRevivedBy` 返回 true 但 REVIVE_CORPSE 选项不出现

**原因**：`componentactions.lua:736` 里的 `revivablecorpse` action 只在**服务端**触发 componentactions 时执行，但客户端也需要知道能否显示这个选项——这依赖于 `revivablecorpse` 组件在客户端的可见性（`--This component runs on client as well`）。

**检查**：确保 `CanBeRevivedBy` 在客户端也能正确返回 true；确保目标的 `"corpse"` tag 同步到了客户端（通过 `revivablecorpse:SetCorpse(true)` 触发 tag 同步）。

#### 坑 3：`REVIVE_CORPSE` 动作被中断——尸体移动了

**原因**：`REVIVE_CORPSE` 是进度条动作（`actionmeter=true`），如果目标在施救过程中移动出范围，动作会中断。尸体状态下玩家通常不会移动，但如果有推动力（爆炸等），可能移出施救范围。

**修复**：在 `revivablecorpse` 组件中或物理层面阻止尸体被推动；或者使用更大的动作范围。

#### 坑 4：复活血量错误——`revive_health_percet` 字段名拼写

**症状**：修改 `revivablecorpse.revive_health_percent` 没有效果。

**原因**：源码里的字段名是 `revive_health_percet`（注意：是 `percet` 不是 `percent`——这是代码里的拼写错误但已成固定）。必须用 `SetReviveHealthPercent(percent)` 方法，不要直接访问字段。

```lua
-- 正确
inst.components.revivablecorpse:SetReviveHealthPercent(0.7)

-- 错误（字段名有拼写问题）
inst.components.revivablecorpse.revive_health_percent = 0.7  -- 无效
```

#### 坑 5：`ms_respawnedfromghost` 事件的 `corpse` 字段被忽略

**症状**：监听 `"ms_respawnedfromghost"` 的代码在尸体复活时也触发，但原本只想处理幽灵复活。

**原因**：无论是幽灵复活还是尸体复活，最终都推 `ms_respawnedfromghost` 事件。区别在于 `data.corpse` 字段：
- 幽灵复活：`data = nil` 或 `data.corpse = nil`
- 尸体复活：`data.corpse = true`

**修复**：

```lua
inst:ListenForEvent("ms_respawnedfromghost", function(inst, data)
    if data ~= nil and data.corpse then
        -- 从尸体复活
    else
        -- 从幽灵复活
    end
end)
```

---

### 19.6 小结

```
两套死亡系统（互斥）：

幽灵系统（ghostenabled=true）            尸体系统（revivable_corpse=true）
  死亡 → makeplayerghost                  死亡 → playerdied
       → OnMakePlayerGhost                      → OnMakePlayerCorpse
       → 变幽灵（SetCorpse=false）               → 变尸体（SetCorpse(true)）
       → 自行找道具复活                          → 等队友手动复活（REVIVE_CORPSE 6秒进度条）
       → respawnfromghost                       → respawnfromcorpse
       → DoActualRez                            → DoActualRezFromCorpse
       → ms_respawnedfromghost（corpse=nil）    → ms_respawnedfromghost（corpse=true）

RevivableCorpse API：
  SetCanBeRevivedByFn(fn)      → 自定义"能否复活"条件
  CanBeRevivedBy(reviver)      → 检查（客户端共用）
  SetReviveSpeedMult(mult)     → 基础复活速度倍率
  SetReviveSpeedMultForTag(tag, mult) → 特定 tag 施救者的速度倍率（可叠加）
  SetReviveHealthPercent(pct)  → 复活后血量 = pct × maxhp（默认 0.5）
  SetCorpse(bool)              → 添加/移除 "corpse" tag
  Revive(reviver)              → 触发 respawnfromcorpse

REVIVE_CORPSE 时长 = 6s × GetReviverSpeedMult × GetReviveSpeedMult
```

**新手核心三句**：普通生存用幽灵系统，竞技场/沼泽用尸体系统，两者互斥；尸体系统需要队友持续按住 6 秒复活（进度条动作）；复活血量 = `revive_health_percet × maxhp`（默认 50%）。

**进阶核心三句**：尸体状态靠 `"corpse"` tag 标识（`SetCorpse(true)`）；`GetGhostEnabled()` 检查 `revivable_corpse` 属性互斥；`DoActualRezFromCorpse` 与 `DoActualRez` 流程相似但复活血量和特效不同，均以 `ms_respawnedfromghost` 结束。

**老手核心三句**：监听器在玩家初始化时一次性注册——后添加 `revivablecorpse` 组件不会改变死亡路径；`revive_health_percet` 是代码里的历史拼写错误，应用 `SetReviveHealthPercent` 方法；区分幽灵复活和尸体复活靠 `ms_respawnedfromghost` 事件的 `data.corpse = true` 标志。


## 19.7 死亡惩罚与状态重置

### 本节导读

"死亡 → 复活"看似只是一次画面切换，但其背后**牵动了玩家身上几乎所有的核心组件**。死亡瞬间，`burnable`、`freezable`、`grogginess`、`slipperyfeet`、`propagator` 都会被移除；`hunger`、`temperature`、`moisture` 被强制归零；`debuffable` 被禁用；`age` 被暂停——这是一次彻底的"状态清场"。而复活则要把这些**重新装回去**，并且要承担"血量上限被削减"这种永久惩罚。

理解死亡惩罚和状态重置机制，对 mod 开发非常关键：
- 你想**让某个 mod 复活道具不扣血量上限**——需要知道 `noreviverhealthpenalty` tag 和 `DeltaPenalty` 调用时机
- 你想**做一个移除血量惩罚的道具**——需要知道 `maxhealer` 组件
- 你想**给 mod 角色添加专属的复活后清理**（如 wormwood 重新观察植物）——需要监听 `ms_respawnedfromghost`
- 你想**让你的 mod 角色完全免疫血量惩罚**——需要知道 `disable_penalty = true` 这个开关
- 你想**让 mod 在死亡时给玩家加一个永久减 mod 资源的惩罚**——需要在 `ms_becameghost` 或 `death` 事件钩入

> **新手**先看 19.7.1-19.7.3——三类惩罚速览、死亡瞬间清场、复活时还原；**进阶**读者继续看 19.7.4-19.7.7，深入 `Health.penalty` 的工作原理、各复活源的惩罚数值对照表、`maxhealer` 组件实现、`Sanity` 的双轨惩罚系统；**老手**跳到 19.7.8-19.7.11，掌握自定义复活后清理、自定义死亡惩罚、跳过血量惩罚的三种方法，以及五个常见坑。

读完本节，你能**精确把控玩家"死亡 → 复活"两个事件中所有状态字段的变化时序**，并能为你的 mod 自定义任意维度的死亡惩罚或复活恢复逻辑。

---

### 19.7.1 快速入门：死亡惩罚三大类速览

死亡给玩家带来的影响分为**三大类**，理解它们是否"永久"、是否"可逆"、靠什么"消除"，是 mod 开发的基础：

| 类别 | 代表性影响 | 是否可逆 | 消除方式 |
|------|-----------|---------|----------|
| **1. 持久属性惩罚** | `health.penalty`（血量上限减少）<br>`sanity.penalty`（精神上限减少）| 可逆 | 专用道具（`lifeinjector` 等）<br>移除惩罚源 |
| **2. 临时状态重置** | 血量重置为 50<br>精神 → 50% / 饥饿 → 66.7% / 温度 → 35<br>湿度归零 / debuff 全暂停 | 自动恢复 | 复活后正常进食、烤火等 |
| **3. 数据记录** | Morgue 死亡记录<br>死亡公告（聊天频道）<br>`last_death_position` / `deathcause` | 永久（但会被下次死亡覆盖）| 无（只能避免再次死亡）|

**注意**：第 1 类惩罚里，**只有少数复活源会增加 `health.penalty`**——其他复活源虽然不加惩罚，但通常需要付出"前置代价"（消耗护符、消耗当前血量绑定雕像/坟墓等）。第 2 类的"重置"在 `CommonPlayerDeath` 函数中统一处理；第 3 类则在 `OnMakePlayerGhost` / `OnMakePlayerCorpse` / `OnPlayerDied` 三个分支里分别添加。

> **新手记忆**：死亡掉血量上限的源头只有 3 种——告密的心、沃尔托克斯复活骨头、传送门。**红色护符、肉雕像、触手石、长明灯、怀表复活、WX-78 备份身体——都不加血量上限惩罚**（但它们各自有不同的前置代价）。

---

### 19.7.2 快速入门：死亡瞬间的状态重置——CommonPlayerDeath 全表

**所有死亡分支**（变幽灵 `OnMakePlayerGhost` / 变尸体 `OnMakePlayerCorpse`）的状态清理都走同一个函数 `CommonPlayerDeath`：

```632:671:scripts/prefabs/player_common_extensions.lua
local function CommonPlayerDeath(inst)
    inst.player_classified.MapExplorer:EnableUpdate(false)

    inst:RemoveComponent("burnable")

    inst.components.freezable:Reset()
    inst:RemoveComponent("freezable")
    inst:RemoveComponent("propagator")

    inst:RemoveComponent("grogginess")
	inst:RemoveComponent("slipperyfeet")

    inst.components.moisture:ForceDry(true, inst)

    inst.components.sheltered:Stop()

    inst.components.debuffable:Enable(false)

    if inst.components.revivablecorpse == nil then
        inst.components.age:PauseAging()
    end

    inst.components.health:SetInvincible(true)
    inst.components.health.canheal = false

    if not GetGameModeProperty("no_sanity") then
        inst.components.sanity:SetPercent(.5, true)
    end
    inst.components.sanity.ignore = true

    if not GetGameModeProperty("no_hunger") then
        inst.components.hunger:SetPercent(2 / 3, true)
    end
    inst.components.hunger:Pause()

    if not GetGameModeProperty("no_temperature") then
        inst.components.temperature:SetTemp(TUNING.STARTING_TEMP)
    end
    inst.components.frostybreather:Disable()
end
```

**死亡时的所有状态变化一览**：

| 操作 | 涉及组件/字段 | 含义 |
|------|--------------|------|
| `MapExplorer:EnableUpdate(false)` | player_classified | 暂停地图迷雾更新（幽灵不揭露地图）|
| `RemoveComponent("burnable")` | burnable | 不再可燃 |
| `freezable:Reset()` + 移除 | freezable | 解除冰冻、不再可冻 |
| `RemoveComponent("propagator")` | propagator | 不再传播火焰 |
| `RemoveComponent("grogginess")` | grogginess | 移除头晕状态、不再可击晕 |
| `RemoveComponent("slipperyfeet")` | slipperyfeet | 不再有冰滑动效果 |
| `moisture:ForceDry(true, inst)` | moisture | 湿度强制归零，**silent=true 不通知** |
| `sheltered:Stop()` | sheltered | 不再受庇护遮挡判定 |
| `debuffable:Enable(false)` | debuffable | **禁用所有 debuff 更新**（不是清除，是暂停）|
| `age:PauseAging()` | age（如果不是 corpse 模式）| 年龄停止计数 |
| `health:SetInvincible(true)`<br>`health.canheal = false` | health | 无敌（防二次伤害）、无法治疗 |
| `sanity:SetPercent(.5, true)` | sanity | 精神设置为 50% |
| `sanity.ignore = true` | sanity | 忽略所有精神变化 |
| `hunger:SetPercent(2/3, true)` | hunger | 饥饿设置为 66.7% |
| `hunger:Pause()` | hunger | 饥饿停止流失 |
| `temperature:SetTemp(35)` | temperature | 温度设置为 `TUNING.STARTING_TEMP = 35` |
| `frostybreather:Disable()` | frostybreather | 关闭呼吸雾气特效 |

紧接着 `OnMakePlayerGhost` 还做了如下"幽灵专用"配置：

```724:728:scripts/prefabs/player_common_extensions.lua
    inst:AddTag("playerghost")
    inst.Network:AddUserFlag(USERFLAGS.IS_GHOST)

    inst.components.health:SetCurrentHealth(TUNING.RESURRECT_HEALTH * (inst.resurrect_multiplier or 1))
    inst.components.health:ForceUpdateHUD(true)
```

也就是说：变幽灵时**当前血量被设为 `RESURRECT_HEALTH = 50`** （乘以可选倍率 `resurrect_multiplier`，仅 Wanda 用 `OLDAGE_HEALTH_SCALE`）——这是为复活后的"50 血起步"做准备。

> **新手记忆**：变幽灵后**饥饿不再扣**、**温度不变化**、**精神固定 50%**——这是为什么幽灵状态下你不会再因为这些机制再次"死亡"（因为已经死了）。所有 debuff（如毒、燃烧、冰冻、流血等）都被"按暂停键"，但**不会清除**——复活后会重新启用，但绝大多数 debuff 会因为组件被重新添加而失效。

---

### 19.7.3 快速入门：复活后的状态恢复——CommonActualRez 全表

无论从哪种途径复活（护符 / 雕像 / 触手石 / 告密的心 / 传送门 / 怀表 / 长明灯 / WX-78 备份身体），最终都会调用 `CommonActualRez`：

```267:325:scripts/prefabs/player_common_extensions.lua
local function CommonActualRez(inst)
    inst.player_classified.MapExplorer:EnableUpdate(true)

    if inst.components.revivablecorpse ~= nil then
        inst.components.inventory:Show()
    else
        inst.components.inventory:Open()
        inst.components.age:ResumeAging()
    end

    inst.components.health.canheal = true
    if not GetGameModeProperty("no_hunger") then
        inst.components.hunger:Resume()
    end
    if not GetGameModeProperty("no_temperature") then
        inst.components.temperature:SetTemp() --nil param will resume temp
    end
    inst.components.frostybreather:Enable()

    MakeMediumBurnableCharacter(inst, "torso")
    inst.components.burnable:SetBurnTime(TUNING.PLAYER_BURN_TIME)
    inst.components.burnable.nocharring = true

    MakeLargeFreezableCharacter(inst, "torso")
    inst.components.freezable:SetResistance(4)
    inst.components.freezable:SetDefaultWearOffTime(TUNING.PLAYER_FREEZE_WEAR_OFF_TIME)

    inst:AddComponent("grogginess")
    inst.components.grogginess:SetResistance(3)
    inst.components.grogginess:SetKnockOutTest(ShouldKnockout)

	inst:AddComponent("slipperyfeet")

    inst.components.moisture:ForceDry(false, inst)

    inst.components.sheltered:Start()

    inst.components.debuffable:Enable(true)

    --don't ignore sanity any more
    inst.components.sanity.ignore = GetGameModeProperty("no_sanity")

    ConfigurePlayerLocomotor(inst)
    ConfigurePlayerActions(inst)

    if inst.rezsource ~= nil then
        local announcement_string = GetNewRezAnnouncementString(inst, inst.rezsource)
        if announcement_string ~= "" then
            TheNet:AnnounceResurrect(announcement_string, inst.entity)
        end
        inst.rezsource = nil
    end
    inst.remoterezsource = nil

	inst.last_death_position = nil
	inst.last_death_shardid = nil

	inst:RemoveTag("reviving")
end
```

**复活时的所有恢复操作一览**：

| 操作 | 涉及组件/字段 | 含义 |
|------|--------------|------|
| `MapExplorer:EnableUpdate(true)` | player_classified | 恢复地图迷雾更新 |
| `inventory:Open()` / `Show()` | inventory | 打开/显示背包（取决于是否 corpse）|
| `age:ResumeAging()` | age（如果不是 corpse 模式）| 年龄继续计数 |
| `health.canheal = true` | health | 允许治疗 |
| `hunger:Resume()` | hunger | 饥饿恢复流失 |
| `temperature:SetTemp()` | temperature | 恢复温度更新（nil 参数表示恢复）|
| `frostybreather:Enable()` | frostybreather | 启用呼吸雾气 |
| `MakeMediumBurnableCharacter` | burnable | **重新添加** 中等可燃组件 |
| `MakeLargeFreezableCharacter` | freezable | **重新添加** 大型可冻组件 |
| `AddComponent("grogginess")` | grogginess | **重新添加** 头晕组件 |
| `AddComponent("slipperyfeet")` | slipperyfeet | **重新添加** 滑倒组件 |
| `moisture:ForceDry(false, inst)` | moisture | 通知"已干燥"（传 false 即从死亡状态切回）|
| `sheltered:Start()` | sheltered | 重新启用庇护检测 |
| `debuffable:Enable(true)` | debuffable | 重新启用 debuff 更新 |
| `sanity.ignore = no_sanity ?` | sanity | 恢复精神变化监听 |
| `ConfigurePlayerLocomotor` / `Actions` | locomotor / playercontroller | 切回活人移动/动作配置 |
| `TheNet:AnnounceResurrect(...)` | TheNet | 广播复活公告 |
| `last_death_position = nil`<br>`last_death_shardid = nil` | inst | 清除死亡位置记录 |
| `RemoveTag("reviving")` | inst | 移除"复活中"标记 |

注意 `health:SetInvincible(false)` 并**不在 CommonActualRez 中**——它是在 SGwilson 的 `amulet_rebirth` / `rebirth` / `wakeup` / `rewindtime_rebirth` 等状态退出时统一恢复的。

> **新手记忆**：死亡时是**移除组件**、复活时是**重新添加组件**。所以 mod 如果给玩家添加了"自定义可燃组件配置"或"自定义状态组件配置"，在 `ms_respawnedfromghost` 中需要**重新应用配置**（不能依赖死亡前的状态），这是一个常见坑——见 19.7.11 坑 5。

---

### 19.7.4 进阶：Health.penalty 详解——0~0.75 的可恢复惩罚

`health.penalty` 是 `Health` 组件上的一个浮点字段，表示**当前最大血量被削减的比例**：

```86:88:scripts/components/health.lua
    self.penalty = 0
    self.disable_penalty = not TUNING.HEALTH_PENALTY_ENABLED
```

**有效范围**：`[0, TUNING.MAXIMUM_HEALTH_PENALTY]` = `[0, 0.75]`。也就是说，**最大血量最多只能被削减 75%**（保留 25%）。

**核心 API**：

```460:475:scripts/components/health.lua
function Health:SetPenalty(penalty)
    --print("Health:SetPenalty", self.disable_penalty)
	if not self.disable_penalty then
		--Penalty should never be less than 0% or ever above 75%.
		self.penalty = math.clamp(penalty, 0, TUNING.MAXIMUM_HEALTH_PENALTY)
	end
end

function Health:DeltaPenalty(delta)
    self:SetPenalty(self.penalty + delta)
    self:ForceUpdateHUD(false) --handles capping health at max with penalty
end

function Health:GetPenaltyPercent()
    return self.penalty
end
```

**有效血量上限**：

```524:526:scripts/components/health.lua
function Health:GetMaxWithPenalty()
    return self.maxhealth - self.maxhealth * self.penalty
end
```

也就是 `currenthealth` 永远不能超过 `maxhealth × (1 - penalty)`——这就是死亡惩罚最直观的体现。

**几个重要细节**：

1. **`disable_penalty = true` 完全冻结 penalty**：调用 `SetPenalty` / `DeltaPenalty` 都没有效果，但 `GetPenaltyPercent` 仍返回当前 `self.penalty` 值（如果之前曾被设置过）
2. **`TUNING.HEALTH_PENALTY_ENABLED`**：tuning 表中的全局开关，默认 `true`，关闭后所有玩家初始 `disable_penalty=true`
3. **`SetPenalty` 不调用 `ForceUpdateHUD`**，只有 `DeltaPenalty` 会调用——所以**单纯 `SetPenalty` 后必须手动调用 `ForceUpdateHUD(false)` 才能让 HUD 正确显示**
4. **存档读写**：`OnSave` 只保存 `penalty > 0` 时的值；`OnLoad` 仅在 `penalty > 0 and penalty < 1` 时调用 `SetPenalty`

```154:172:scripts/components/health.lua
function Health:OnSave()
    return
    {
        health = self.currenthealth,
        penalty = self.penalty > 0 and self.penalty or nil,
		maxhealth = self.save_maxhealth and self.maxhealth or nil
    }
end

function Health:OnLoad(data)
	if data.maxhealth ~= nil then
		self.maxhealth = data.maxhealth
	end

    local haspenalty = data.penalty ~= nil and data.penalty > 0 and data.penalty < 1
    if haspenalty then
        self:SetPenalty(data.penalty)
    end
```

> **进阶记忆**：`penalty` 是 `[0, 0.75]` 的浮点比例，**不是绝对血量**——`DeltaPenalty(0.25)` 不是"减 25 血"，而是"上限减 25%"。如果你想给 mod 角色添加"绝对血量惩罚"，需要用 `SetMaxHealth(newMax)` 而不是 `penalty`。

---

### 19.7.5 进阶：各复活源的血量惩罚数值表

不同复活源的代价不同。下面是**完整的复活源对照表**：

| 复活源 | 复活时 `penalty` 增量 | 前置代价 | 代码位置 |
|--------|---------------------|---------|----------|
| **红色护符 `amulet`** | **0**（无惩罚）| 消耗护符道具（finiteuses）| `DoActualRez` 行 368-370 |
| **肉雕像 `resurrectionstatue`** | **0**（无惩罚）| **绑定时**消耗 `EFFIGY_HEALTH_PENALTY = 40` 当前血量 | `resurrectionstatue.lua` `onattunecost` |
| **触手石 `resurrectionstone`** | **0**（无惩罚）| 一次性激活后销毁石头 | `DoActualRez` 行 371-374 |
| **长明灯 `wendy_resurrectiongrave`** | **0**（无惩罚）| **建造时**消耗 `EFFIGY_HEALTH_PENALTY = 40` 当前血量 | `recipes.lua` 行 155 |
| **传送门 `multiplayer_portal`** | **+0.25** | 无前置（建图时存在）| `DoActualRez` 行 386 |
| **告密的心 `reviver`** | **+0.25** | 消耗心、施救者扣精神 | `OnGetItem` 行 357-360 |
| **沃尔托克斯复活骨头 `wortox_reviver`** | **+0.25**（默认）<br>**0**（有 `wortox_lifebringer_2` 技能时）| 同 `reviver` | `OnGetItem` 行 346-349 |
| **鬼魂复仇灵药 `ghostlyelixir_retaliation`** | `DeltaPenalty(-0.25)`<br>实际是**减少** 25% 惩罚 | 灵药一次性消耗 | `ghostly_elixirs.lua` 行 312 |
| **怀表复活 `pocketwatch_revive`** | **0**（无惩罚）| 消耗怀表 | `DoActualRez` 行 392-413 |
| **WX-78 备份身体 `wx78_backupbody`** | **0**（无惩罚）| 消耗备份身体 | `DoActualRez` 行 383-384 |

**对应的源码片段**：

**传送门复活的 +0.25 惩罚**（在 `DoActualRez` 中）：

```385:390:scripts/prefabs/player_common_extensions.lua
        elseif source:HasTag("multiplayer_portal") then
            inst.components.health:DeltaPenalty(TUNING.PORTAL_HEALTH_PENALTY)

            source:PushEvent("rez_player")
            inst.sg:GoToState("portal_rez")
        end
```

**告密的心 / 沃尔托克斯骨头的 +0.25 惩罚**（在 `OnGetItem` 中）：

```341:366:scripts/prefabs/player_common.lua
local function OnGetItem(inst, giver, item)
    if item ~= nil and item:HasTag("reviver") and inst:HasTag("playerghost") then
        if item.skin_sound then
            item.SoundEmitter:PlaySound(item.skin_sound)
        end
        local dohealthpenalty = not item:HasTag("noreviverhealthpenalty")
        if item.prefab == "wortox_reviver" and giver.components.skilltreeupdater and giver.components.skilltreeupdater:IsActivated("wortox_lifebringer_2") then
            dohealthpenalty = false
        end

        item:PushEvent("usereviver", { user = giver })
        giver.hasRevivedPlayer = true
        AwardPlayerAchievement("hasrevivedplayer", giver)
        item:Remove()
        inst:PushEvent("respawnfromghost", { source = item, user = giver })

        if dohealthpenalty then
            inst.components.health:DeltaPenalty(TUNING.REVIVE_HEALTH_PENALTY)
        end
        giver.components.sanity:DoDelta(TUNING.REVIVE_OTHER_SANITY_BONUS)
    elseif item ~= nil and giver.components.age ~= nil then
		if giver.components.age:GetAgeInDays() >= TUNING.ACHIEVEMENT_HELPOUT_GIVER_MIN_AGE and inst.components.age:GetAgeInDays() <= TUNING.ACHIEVEMENT_HELPOUT_RECEIVER_MAX_AGE then
			AwardPlayerAchievement("helping_hand", giver)
		end
    end
end
```

**两个跳过血量惩罚的 tag/技能机制**：

1. **`noreviverhealthpenalty` tag**：物品（reviver 类型）带这个 tag 就免疫血量惩罚——mod 自制复活道具时给 prefab 添加 `inst:AddTag("noreviverhealthpenalty")` 即可
2. **`wortox_lifebringer_2` 技能**：沃尔托克斯专用技能树解锁后，他扔出的 `wortox_reviver` 不再加惩罚

**关键 TUNING 数值**（见 `tuning.lua` 行 2150-2161、2345）：

```lua
PORTAL_HEALTH_PENALTY = 0.25,                     -- 传送门复活
HEART_HEALTH_PENALTY = 0.125,                     -- （未使用，保留字段）
MAXIMUM_HEALTH_PENALTY = 0.75,                    -- penalty 上限
MAXIMUM_SANITY_PENALTY = 0.9,                     -- （字段保留，实际精神上限由 1-5/max 决定）
EFFIGY_HEALTH_PENALTY = 40,                       -- 雕像/坟墓绑定消耗的当前血量
REVIVE_HEALTH_PENALTY_AS_MULTIPLE_OF_EFFIGY = 1,  -- （未实际使用的旧字段）
REVIVE_SHADOW_SANITY_PENALTY = -40,               -- 复活别人时影子仆人的精神惩罚
REVIVE_OTHER_SANITY_BONUS = 80,                   -- 复活别人时的精神奖励
REVIVE_HEALTH_PENALTY = 0.25,                     -- 告密的心、wortox_reviver 复活惩罚
RESURRECT_HEALTH = 50,                            -- 复活后初始血量
```

> **进阶记忆**：**复活后初始血量永远是 50**（除非有 `inst.resurrect_multiplier` 倍率，如 Wanda 用 `OLDAGE_HEALTH_SCALE`）。**血量惩罚增量只有 3 种来源**：告密的心、wortox_reviver（无技能时）、传送门。

---

### 19.7.6 进阶：恢复血量惩罚的途径——maxhealer 组件

`Health.penalty` 是可以**主动消除**的——通过 `DeltaPenalty(负数)` 即可。游戏自带一个**专门的组件**叫 `MaxHealer`，专门用于"减少血量惩罚"：

```1:24:scripts/components/maxhealer.lua
local MaxHealer = Class(function(self, inst)
    self.inst = inst
    self.healamount = TUNING.MAX_HEALING_NORMAL
end)

--NOTE: This is set as a factor of num revives! not an HP amount
function MaxHealer:SetHealthAmount(health)
    self.healamount = health
end

function MaxHealer:Heal(target)
    if target.components.health ~= nil then
        target.components.health:DeltaPenalty(self.healamount) --remove x% from the penalty.
        --print(target.components.health.penalty)
        if self.inst.components.stackable ~= nil and self.inst.components.stackable:IsStack() then
            self.inst.components.stackable:Get():Remove()
        else
            self.inst:Remove()
        end
        return true
    end
end

return MaxHealer
```

注意 `TUNING.MAX_HEALING_NORMAL = -0.25`（行 2002）——**负值**！它会被传给 `DeltaPenalty`，从而**减少** `penalty` 字段。

**生命注射器（lifeinjector）** 是这个组件的典型用例：

```1:42:scripts/prefabs/lifeinjector.lua
local assets =
{
    Asset("ANIM", "anim/lifepen.zip"),
}

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("lifepen")
    inst.AnimState:SetBuild("lifepen")
    inst.AnimState:PlayAnimation("idle")

    MakeInventoryFloatable(inst)

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("stackable")
    inst.components.stackable.maxsize = TUNING.STACK_SIZE_SMALLITEM

    inst:AddComponent("inspectable")

    inst:AddComponent("inventoryitem")

    inst:AddComponent("maxhealer")

    MakeHauntableLaunch(inst)

    return inst
end

return Prefab("lifeinjector", fn, assets)
```

整个 lifeinjector 的"减惩罚"逻辑就一行：`inst:AddComponent("maxhealer")`——其余的 `Heal` 调用由 Action 系统通过使用动作触发。

**其他可以减少血量惩罚的来源**：

**温蒂的鬼魂复仇灵药 `ghostlyelixir_retaliation`**（施加在玩家身上时）：

```300:315:scripts/prefabs/ghostly_elixirs.lua
		DURATION_PLAYER = TUNING.GHOSTLYELIXIR_PLAYER_REVIVE_DURATION,
		ONAPPLY_PLAYER = function(inst, target)
			target.components.talker:Say(GetString(target, "ANNOUNCE_ELIXIR_BOOSTED"))

			if target.components.sanity then
				target.components.sanity:DoDelta(TUNING.SANITY_TINY)
			end
			if target.components.hunger then
				target.components.hunger:DoDelta(TUNING.CALORIES_SMALL)
			end

			if target.components.health ~= nil then
				target.components.health:DeltaPenalty(TUNING.MAX_HEALING_NORMAL)
			end
		end,
```

施加在玩家身上的"复仇灵药"会自动调用 `DeltaPenalty(-0.25)`，回血量上限 25%。

> **进阶记忆**：**减少 penalty 用的还是 `DeltaPenalty`，只是传负数**。如果你做一个 mod"血量上限恢复药水"，只需要 `target.components.health:DeltaPenalty(-yourAmount)` 即可，等同于"减负"。`DeltaPenalty` 会调用 `SetPenalty`，后者用 `math.clamp` 把结果限制在 `[0, 0.75]`，因此**不可能减成负数**。

---

### 19.7.7 进阶：Sanity 的双轨惩罚系统——penalty 字段 vs sanity_penalties 字典

`Sanity` 组件的惩罚机制比 `Health` 复杂——它有**两套并存**的系统：

```99:104:scripts/components/sanity.lua
	self.neg_aura_immune_sources = SourceModifierList(inst, false, SourceModifierList.boolean)
    self.dapperness_mult = 1
    self.penalty = 0

    self.sanity_penalties = {}
```

#### 轨道 A：`self.penalty` 字段

- 类型：浮点数，范围 `[0, 1-(5/self.max)]`（保留至少 5 max sanity）
- 通过 `RecalculatePenalty` 由 `sanity_penalties` 字典累加产生
- **没有公开的 `SetPenalty` 方法**——外部无法直接设置 `penalty` 字段，只能通过字典 API

#### 轨道 B：`self.sanity_penalties` 字典

外部接入的标准方式：

```189:211:scripts/components/sanity.lua
function Sanity:AddSanityPenalty(key, mod)
    self.sanity_penalties[key] = mod
    self:RecalculatePenalty()
end

function Sanity:RemoveSanityPenalty(key)
    self.sanity_penalties[key] = nil
    self:RecalculatePenalty()
end

function Sanity:RecalculatePenalty()
    local penalty = 0

    for k,v in pairs(self.sanity_penalties) do
        penalty = penalty + v
    end

    -- players cannot go lower than 5 max sanity. The sanity_penalties penalty will actually go beyond,
    -- so they will still have to remove enough sanity_penalties to get back above the 5 max sanity cap
    self.penalty = math.min(penalty, 1-(5/self.max))

    self:DoDelta(0)
end
```

**机制特点**：

1. **键 → 值字典**：可以同时存在多个 penalty 来源，每个用唯一的 key 标识
2. **可叠加**：所有来源的值会**相加**作为最终 penalty 比例
3. **底线保护**：实际生效的 penalty 不超过 `1 - 5/max`——保证 max sanity 始终 ≥ 5
4. **`mod` 值**：是 0~1 的浮点比例，如 0.1 表示削减 10%

**最大经典使用案例**——麦斯威尔的影子仆人（`scripts/prefabs/waxwell.lua`）：

```54:64:scripts/prefabs/waxwell.lua
local function OnSpawnPet(inst, pet)
    if pet:HasTag("shadowminion") then
        if not (inst.components.health:IsDead() or inst:HasTag("playerghost")) then
			--if not inst.components.builder.freebuildmode then
	            inst.components.sanity:AddSanityPenalty(pet, TUNING.SHADOWWAXWELL_SANITY_PENALTY[string.upper(pet.prefab)])
			--end
            inst:ListenForEvent("onremove", inst._onpetlost, pet)
            pet.components.skinner:CopySkinsFromPlayer(inst)
        elseif pet._killtask == nil then
            pet._killtask = pet:DoTaskInTime(math.random(), KillPet)
        end
```

每召唤一个影子仆人，就用 `pet`（实体引用）作为 key 添加一份精神惩罚。仆人死亡/被解除时，用 `RemoveSanityPenalty(pet)` 移除。

> **进阶记忆**：mod 给玩家精神加惩罚，**永远用 `AddSanityPenalty(key, mod)`**，不要直接改 `self.penalty`——直接改会被 `RecalculatePenalty` 重置回字典累加值。**`key` 选用一个稳定且唯一的引用**（实体、字符串常量），方便后续 `RemoveSanityPenalty` 移除。

---

### 19.7.8 老手：监听复活事件的正确姿势——ms_becameghost、ms_respawnedfromghost

mod 开发中最常用的"死亡/复活钩子"是这两个事件：

| 事件 | 推送时机 | 推送的 data |
|------|---------|-----------|
| `"ms_becameghost"` | `OnMakePlayerGhost` 末尾（成为幽灵后）<br>`OnMakePlayerCorpse` 末尾（成为尸体后）| `nil`（变幽灵时）<br>`{ corpse = true }`（变尸体时）|
| `"ms_respawnedfromghost"` | `DoActualRez` 末尾（从幽灵复活）<br>`DoActualRezFromCorpse` 末尾（从尸体复活）| `nil`（从幽灵复活）<br>`{ corpse = true, reviver = source }`（从尸体复活）|

**注册方式**：

```lua
inst:ListenForEvent("ms_becameghost", function(inst, data)
    if data ~= nil and data.corpse then
    else
    end
end)

inst:ListenForEvent("ms_respawnedfromghost", function(inst, data)
    if data ~= nil and data.corpse then
    else
    end
end)
```

**官方案例 1 ——Wormwood 的植物观察**（`prefabs/wormwood.lua` 行 584-599）：

```584:599:scripts/prefabs/wormwood.lua
local function OnBecameGhost(inst)
    inst.components.bloomness:SetLevel(0)
    StopWatchingWorldPlants(inst)

    inst:UpdatePhotosynthesisState(TheWorld.state.isday)
end

local function OnRespawnedFromGhost(inst)
    inst.sg.mem.nocorpse = true -- No flesh inside us.
    if TheWorld.state.isspring then
        inst.components.bloomness:Fertilize()
    end
    WatchWorldPlants(inst)

    inst:UpdatePhotosynthesisState(TheWorld.state.isday)
end
```

- 死亡时：清零开花等级、停止监听世界植物
- 复活时：重新监听植物、春天则触发施肥

**官方案例 2 ——Wolfgang 的力量重置**（`prefabs/wolfgang.lua` 行 112-127）：

```112:127:scripts/prefabs/wolfgang.lua
local function onbecamehuman(inst, data)
    inst.components.mightiness:Resume()
    inst.components.mightiness:SetPercent(0.5, true, true)

    StartPlayerCheck(inst)
end

local function onbecameghost(inst, data)
    inst.components.mightiness:Pause()
	inst.hurtsoundoverride = nil

    if inst.playercheck_task ~= nil then
        inst.playercheck_task:Cancel()
        inst.playercheck_task = nil
    end
end
```

- 死亡时：暂停力量、清除受伤声音覆盖
- 复活时：恢复力量、重置为 50%、重新启动检测任务

**官方案例 3 ——Wickerbottom 的不可击晕**（`prefabs/wickerbottom.lua` 行 62-90）：

```62:90:scripts/prefabs/wickerbottom.lua
local function OnRespawnedFromGhost(inst)
    inst.components.grogginess:SetKnockOutTest(KnockOutTest)
end

local function master_postinit(inst)
    inst.starting_inventory = start_inv[TheNet:GetServerGameMode()] or start_inv.default

    inst.customidleanim = customidleanimfn

    inst.soundsname = "wickerbottom"
    --inst.talker_path_override = "dontstarve_DLC001/characters/"

    inst.components.eater:SetDiet({ FOODGROUP.OMNI }, { FOODGROUP.OMNI })

    inst.components.foodaffinity:AddPrefabAffinity("trailmix", TUNING.AFFINITY_15_CALORIES_MEDIUM)
    inst.components.foodaffinity:AddPrefabAffinity("kabobs", TUNING.AFFINITY_15_CALORIES_HUGE)
    inst.components.foodaffinity:AddPrefabAffinity("surfnturf", TUNING.AFFINITY_15_CALORIES_LARGE)

    inst.components.health:SetMaxHealth(TUNING.WICKERBOTTOM_HEALTH)
    inst.components.hunger:SetMax(TUNING.WICKERBOTTOM_HUNGER)
    inst.components.sanity:SetMax(TUNING.WICKERBOTTOM_SANITY)

    inst.components.builder.science_bonus = 1

    inst:ListenForEvent("ms_respawnedfromghost", OnRespawnedFromGhost)
    OnRespawnedFromGhost(inst)

    if TheNet:GetServerGameMode() == "lavaarena" then
        event_server_data("lavaarena", "prefabs/wickerbottom").master_postinit(inst)
    elseif TheNet:GetServerGameMode() == "quagmire" then
        event_server_data("quagmire", "prefabs/wickerbottom").master_postinit(inst)
    end
end
```

由于 `grogginess` 组件在死亡时被移除、复活时**重新添加**（CommonActualRez 中），所以薇克巴顿"永不被击晕"的特性必须在每次复活后**重新设置** `SetKnockOutTest`——这就是为什么 `OnRespawnedFromGhost` 不仅在事件中触发，初始化时也立即调用一次。

> **老手记忆**：监听 `ms_respawnedfromghost` 时**必须考虑** "CommonActualRez 刚刚重新添加了 burnable / freezable / grogginess / slipperyfeet 四个组件"——这意味着你之前对它们做的任何自定义配置都会**丢失**。常见做法是把"对这四个组件的配置"封装成一个函数，初始化时调用一次，`ms_respawnedfromghost` 时再调用一次。

---

### 19.7.9 老手：mod 自定义死亡惩罚的三种方式

如果你的 mod 想给玩家死亡添加一个"自定义惩罚"，主要有三种切入点：

#### 方式 1：监听 `"death"` 事件（最早，玩家刚死的那一刻）

```lua
local function OnMyModDeathPenalty(inst, data)
    inst.mymod_skillpoints = math.max(0, (inst.mymod_skillpoints or 0) - 1)
end

inst:ListenForEvent("death", OnMyModDeathPenalty)
```

**时机**：在 `Health:SetVal` 推送 `death` 事件时（即血量归零后立即）；此时还没进入死亡状态图。

#### 方式 2：监听 `"ms_becameghost"` 事件（已经变成幽灵/尸体）

```lua
local function OnMyModBecameGhost(inst, data)
    inst.components.health:DeltaPenalty(0.1)
    inst.components.sanity:AddSanityPenalty("mymod_death", 0.05)
end

inst:ListenForEvent("ms_becameghost", OnMyModBecameGhost)
```

**时机**：在 `OnMakePlayerGhost` / `OnMakePlayerCorpse` 末尾。

#### 方式 3：监听 `"ms_respawnedfromghost"` 事件（复活后）

```lua
local function OnMyModRespawned(inst, data)
    inst.components.locomotor:SetExternalSpeedMultiplier(inst, "mymod_rez_slow", 0.5)
    inst:DoTaskInTime(5, function()
        inst.components.locomotor:RemoveExternalSpeedMultiplier(inst, "mymod_rez_slow")
    end)
end

inst:ListenForEvent("ms_respawnedfromghost", OnMyModRespawned)
```

**时机**：在 `DoActualRez` / `DoActualRezFromCorpse` 末尾。

**完整 mod 模板**（一个"死亡积分"惩罚 mod）：

```lua
AddPlayerPostInit(function(inst)
    if not TheWorld.ismastersim then return end

    inst:ListenForEvent("death", function(inst, data)
        inst.mymod_deaths = (inst.mymod_deaths or 0) + 1

        if inst.components.health then
            inst.components.health:DeltaPenalty(0.05)
        end
    end)

    inst:ListenForEvent("ms_respawnedfromghost", function(inst, data)
        if inst.components.talker and inst.mymod_deaths then
            inst.components.talker:Say(string.format("我已经死了 %d 次了……", inst.mymod_deaths))
        end
    end)
end)
```

> **老手记忆**：**事件触发顺序是 `death` → `ms_becameghost`（或 `playerdied`）→ `ms_respawnedfromghost`**。如果你想"在 death 事件中扣 penalty"，要注意此时 `Health` 组件的 `disable_penalty` 仍然起作用——更稳妥的做法是用 `inst:DoTaskInTime(0, ...)` 延迟一帧，或直接选择在 `ms_becameghost` 中扣除。

---

### 19.7.10 老手：跳过死亡惩罚的几种姿势

游戏里有三种"跳过血量惩罚"的标准机制，mod 可以借鉴：

#### 姿势 1：`health.disable_penalty = true`（Wanda 的做法）

完全冻结 `penalty` 字段——任何 `SetPenalty` / `DeltaPenalty` 都不生效：

```412:415:scripts/prefabs/wanda.lua
    inst.components.health.redirect = redirect_to_oldager
	inst.components.health.canheal = false
	inst.components.health.disable_penalty = true
    inst.resurrect_multiplier = TUNING.OLDAGE_HEALTH_SCALE
```

Wanda 用此机制，因为她的"血量"实际上是"年龄"——传统的血量惩罚机制对她没有意义。注意还设置了 `resurrect_multiplier = OLDAGE_HEALTH_SCALE`——这会让 `RESURRECT_HEALTH × resurrect_multiplier` 成为她的复活初始血量（不是 50）。

#### 姿势 2：物品 tag `"noreviverhealthpenalty"`

让一个 `reviver` 类型的复活道具**不扣血量上限**：

```346:359:scripts/prefabs/player_common.lua
        local dohealthpenalty = not item:HasTag("noreviverhealthpenalty")
        if item.prefab == "wortox_reviver" and giver.components.skilltreeupdater and giver.components.skilltreeupdater:IsActivated("wortox_lifebringer_2") then
            dohealthpenalty = false
        end

        item:PushEvent("usereviver", { user = giver })
        giver.hasRevivedPlayer = true
        AwardPlayerAchievement("hasrevivedplayer", giver)
        item:Remove()
        inst:PushEvent("respawnfromghost", { source = item, user = giver })

        if dohealthpenalty then
            inst.components.health:DeltaPenalty(TUNING.REVIVE_HEALTH_PENALTY)
        end
```

**mod 用法**：自制一个"友好的复活道具"，在 `master_postinit` 中给 prefab 添加该 tag：

```lua
inst:AddTag("reviver")
inst:AddTag("noreviverhealthpenalty")
```

#### 姿势 3：技能树解锁（沃尔托克斯专属）

```346:349:scripts/prefabs/player_common.lua
        local dohealthpenalty = not item:HasTag("noreviverhealthpenalty")
        if item.prefab == "wortox_reviver" and giver.components.skilltreeupdater and giver.components.skilltreeupdater:IsActivated("wortox_lifebringer_2") then
            dohealthpenalty = false
        end
```

`wortox_lifebringer_2` 是沃尔托克斯的"生命使者 II"技能；解锁后他的 `wortox_reviver`（白色骨头）不再扣血量惩罚。

**mod 类似实现**：

```lua
inst:ListenForEvent("death", function(inst, data)
    if inst.components.skilltreeupdater and inst.components.skilltreeupdater:IsActivated("mymod_immortal_skill") then
        inst.mymod_skip_next_penalty = true
    end
end)

inst:ListenForEvent("ms_becameghost", function(inst, data)
    if inst.mymod_skip_next_penalty then
        inst.mymod_skip_next_penalty = nil
        return
    end
    inst.components.health:DeltaPenalty(0.1)
end)
```

> **老手记忆**：**`disable_penalty` 是最彻底的"免疫"**——既不能加也不能减（包括 lifeinjector 也无效）。**`noreviverhealthpenalty` 只针对 reviver 类型道具**——传送门复活仍会扣。**技能树**最灵活但只对特定 prefab 生效。

---

### 19.7.11 老手：五个常见坑

#### 坑 1：在 `death` 事件中直接修改组件，可能被后续 CommonPlayerDeath 覆盖

```lua
-- 错误示范
inst:ListenForEvent("death", function(inst, data)
    inst.components.hunger:SetPercent(0.1)
end)
```

`death` 事件发生在 `Health:SetVal` 推送的时刻，紧接着 SGwilson `death` 状态触发 `OnMakePlayerGhost` / `OnMakePlayerCorpse`，其中调用 `CommonPlayerDeath` 把饥饿设回 66.7%——你的修改无效。

**正确做法**：在 `ms_becameghost` 事件中修改（此时 `CommonPlayerDeath` 已经执行完毕）：

```lua
inst:ListenForEvent("ms_becameghost", function(inst, data)
    inst.components.hunger:SetPercent(0.1)
end)
```

#### 坑 2：在 `ms_respawnedfromghost` 中重复调用 Resume

```lua
-- 错误示范——多此一举
inst:ListenForEvent("ms_respawnedfromghost", function(inst, data)
    inst.components.hunger:Resume()
    inst.components.hunger:SetPercent(0.8)
end)
```

`CommonActualRez` 已经调用过 `hunger:Resume()`（行 279）、`temperature:SetTemp()`（行 282）、`debuffable:Enable(true)`（行 304）等等。可以直接 `SetPercent`，不需要再 Resume。

#### 坑 3：`disable_penalty=true` 时，`DeltaPenalty` 静默失败

```lua
inst.components.health.disable_penalty = true
inst.components.health:DeltaPenalty(0.25)
print(inst.components.health:GetPenaltyPercent())  -- 依然是 0
```

**调试建议**：在 mod 添加自定义惩罚前，先 `print(inst.components.health.disable_penalty)` 检查。也要注意：如果世界设置了 `TUNING.HEALTH_PENALTY_ENABLED = false`，所有玩家默认 `disable_penalty=true`。

#### 坑 4：`AddSanityPenalty(key, mod)` 使用相同 key 会覆盖前值

```lua
-- 错误示范
inst.components.sanity:AddSanityPenalty("mymod", 0.1)
inst.components.sanity:AddSanityPenalty("mymod", 0.05)  -- 覆盖了 0.1，最终只剩 0.05
```

`AddSanityPenalty` 内部是 `self.sanity_penalties[key] = mod`——同 key 就是覆盖。

**正确做法**：每次累加都用唯一 key：

```lua
inst.components.sanity:AddSanityPenalty("mymod_death_1", 0.1)
inst.components.sanity:AddSanityPenalty("mymod_death_2", 0.05)
-- 现在最终 penalty = 0.15
```

或者**自己维护累加逻辑**，每次只用一个 key 但传新累计值：

```lua
inst.mymod_total = (inst.mymod_total or 0) + 0.05
inst.components.sanity:AddSanityPenalty("mymod", inst.mymod_total)
```

#### 坑 5：`CommonActualRez` 重新添加 `burnable` 等组件后，mod 配置丢失

```lua
-- 错误示范
local function masterpostinit(inst)
    MakeLargeBurnableCharacter(inst, "torso")
    inst.components.burnable:SetBurnTime(100)
end
```

**后果**：第一次死亡复活后，`CommonActualRez` 用 `MakeMediumBurnableCharacter` 重置了 burnable 组件，自定义的 100 秒燃烧时间没了。

**正确做法**：把配置封装成函数，并在 `ms_respawnedfromghost` 时**再次调用**：

```lua
local function ConfigureMyBurnable(inst)
    inst.components.burnable:SetBurnTime(100)
end

local function masterpostinit(inst)
    MakeMediumBurnableCharacter(inst, "torso")
    ConfigureMyBurnable(inst)
    inst:ListenForEvent("ms_respawnedfromghost", ConfigureMyBurnable)
end
```

同样的注意事项也适用于 `freezable`、`grogginess`、`slipperyfeet` 三个组件。

---

### 19.7 小结

```
死亡 → 复活 状态变化时序：

[活]                            [死]                              [活]
                死亡瞬间          幽灵/尸体期             复活瞬间
                ↓                                       ↓
        CommonPlayerDeath                       CommonActualRez
        -------------------                     -------------------
        ① 移除组件:                              ① 重新添加组件:
           burnable/freezable                      burnable/freezable
           propagator/grogginess                   grogginess/slipperyfeet
           slipperyfeet
        ② 强制干燥(moisture)                     ② 切回干燥（湿度刷新）
        ③ 暂停 debuffable                        ③ 启用 debuffable
        ④ 暂停 age (非 corpse)                    ④ 恢复 age
        ⑤ SetInvincible(true)                    ⑤（在 SG 中)解除无敌
        ⑥ canheal=false                          ⑥ canheal=true
        ⑦ sanity → 50%, ignore=true              ⑦ sanity.ignore = no_sanity 设置
        ⑧ hunger → 66.7%, Pause                  ⑧ hunger:Resume()
        ⑨ temp → STARTING_TEMP=35                ⑨ temp:SetTemp() 恢复
        ⑩ frostybreather:Disable                 ⑩ frostybreather:Enable

血量惩罚 (health.penalty)：
  范围 [0, 0.75]                       由 SetPenalty/DeltaPenalty 操作
  GetMaxWithPenalty() = max × (1-p)    复活后初始血量 = RESURRECT_HEALTH = 50

精神惩罚 (双轨)：
  sanity.penalty 字段                   私有，外部不可直接设置
  sanity.sanity_penalties 字典          AddSanityPenalty(key, mod) 累加
  RecalculatePenalty() 重新计算          上限 = 1 - 5/max（保留最少 5）

复活源 → penalty 增量：
  红色护符 / 触手石 / 肉雕像 / 长明灯 / 怀表 / WX 备份身体 → 0（无惩罚）
  传送门                                                 → +0.25
  告密的心 / wortox_reviver(默认)                          → +0.25
  wortox_reviver + 沃尔托克斯生命使者 II 技能                → 0
  鬼魂复仇灵药                                            → -0.25（减惩罚）

恢复 penalty：
  maxhealer 组件: DeltaPenalty(MAX_HEALING_NORMAL=-0.25)
  lifeinjector 道具: 内置 maxhealer 组件
  ghostlyelixir_retaliation: ONAPPLY_PLAYER 中 DeltaPenalty(-0.25)
```

**新手核心三句**：死亡时几乎所有状态都被"清场"（移除组件、归零计数、暂停）；复活后这些状态被"重新装回"（多数组件是新实例）；只有 3 种复活源会扣血量上限——告密的心、wortox_reviver、传送门。

**进阶核心三句**：`health.penalty` 范围 `[0, 0.75]`，用 `DeltaPenalty`（含负数）调整，`GetMaxWithPenalty` 是真正的有效血量上限；`sanity` 走双轨——直接 `self.penalty` 字段（私有）和 `sanity_penalties` 字典（公开 API），永远通过字典 API 操作；`MaxHealer` 组件就是用 `DeltaPenalty(负数)` 减惩罚。

**老手核心三句**：自定义死亡逻辑监听 `"ms_becameghost"`（设置）和 `"ms_respawnedfromghost"`（重置）两个事件配对使用；`CommonActualRez` 会"重新创建" `burnable / freezable / grogginess / slipperyfeet` 四个组件，所以对它们的 mod 配置必须在 `ms_respawnedfromghost` 中重新应用；`disable_penalty=true`（Wanda）、`"noreviverhealthpenalty"` tag（针对 reviver 物品）、技能树（沃尔托克斯）是三种跳过血量惩罚的官方姿势。


## 19.8 实战：自定义复活道具与死亡效果

### 本节导读

19.1 到 19.7 把"死亡—幽灵—复活"这条链路完整拆开了：从 `Health:DoDelta` 触发 `entity_death` 事件，到 SGwilson 的 `death` 状态掉物品，再到 `OnMakePlayerGhost` 生成骨架并切换幽灵状态，再到 `respawnfromghost` → `DoActualRez` → `CommonActualRez` 还原生存状态——每一步对应哪些代码、哪些事件、哪些字段，前几节都讲清楚了。

**本节不再讲机制**，而是把这些零件**拼回成 mod**。我会给出 9 个可以直接复制到自己 mod 工程里的实战案例，从"5 行代码的复活药水"到"自定义复活动画分支"，每一个都标注它**复用了 19.1-19.7 的哪一段官方代码**，让你清楚地看到：教程拆出来的不只是理论，而是可以立即落地的工程模块。

**本节的 9 个实战目标**：

| 难度 | 编号 | 实战目标 | 复用机制 |
|------|------|---------|----------|
| 新手 | 19.8.2 | 5 行核心代码：能被队友"喂"给幽灵的复活药 | `reviver` tag + `MakeHauntableLaunch` |
| 新手 | 19.8.3 | 骚扰即复活的"灵魂符石" | `hauntable:SetHauntValue(HAUNT_INSTANT_REZ)` |
| 进阶 | 19.8.4 | 多次使用的复活罐——耐久 + 不消耗 hauntvalue | `finiteuses` + `no_wipe_value = true` |
| 进阶 | 19.8.5 | 绑定式复活神龛——绑定花精神而不是血 | `attunable` + 自定义 `OnAttuneCostFn` |
| 进阶 | 19.8.6 | 复活后给玩家 5 秒无敌 + 满血回归 buff | 监听 `ms_respawnedfromghost` |
| 进阶 | 19.8.7 | 死亡时永久扣除一个 mod 货币 | 监听 `ms_becameghost` / `death` |
| 老手 | 19.8.8 | 替换骨架 prefab + 自定义掉落 | `inst.skeleton_prefab` + `master_postinit` |
| 老手 | 19.8.9 | 给 `DoActualRez` 加分支：自定义复活动画 | `AddClassPostConstruct` + `respawnfromghost` |
| 老手 | 19.8.10 | 网络同步：复活道具的客户端表现 | `net_*` 网络变量 + `OnUpdate` |

读完本节，你**应该能独立做出一套完整的死亡—复活相关 mod**：自定义道具、自定义建筑、自定义动画、自定义惩罚——所有官方功能能做的，你都能做。

> **新手**先看 19.8.1-19.8.3——理清 mod 工程目录结构、用 5 行代码复刻告密的心、用 5 行代码做一块骚扰即复活的符石；**进阶读者**继续看 19.8.4-19.8.7，给道具加耐久和"不消耗 hauntvalue"、复用 `attunable` 系统、用 `ms_respawnedfromghost` 给复活后挂 buff、用 `ms_becameghost` 加自定义惩罚；**老手**跳到 19.8.8-19.8.10 + 19.8.11，自定义骨架 prefab、用 `AddClassPostConstruct` 给 `DoActualRez` 加分支以实现专属复活动画、做服务端/客户端网络同步、以及五个最容易栽跟头的陷阱。

---

### 19.8.1 实战准备：mod 工程结构与必备文件

在开始写代码前，先确认你的 mod 工程**至少**有这些文件（饥荒联机版 mod 的最简结构）：

```
mymod/
├── modinfo.lua           （mod 元信息：名称、版本、配置）
├── modmain.lua           （mod 主入口：AddPrefab、PrefabPostInit 等）
└── scripts/
    └── prefabs/
        └── myreviver.lua  （你自定义的 prefab）
```

#### modinfo.lua 最小模板

```lua
name = "我的复活 mod"
description = "教程实战 19.8"
author = "your_name"
version = "1.0.0"

forumthread = ""

api_version = 10
dst_compatible = true
all_clients_require_mod = true   -- 涉及新 prefab，所有客户端都要加载
client_only_mod = false
server_only_mod = false

icon_atlas = "modicon.xml"
icon = "modicon.tex"

priority = 0

configuration_options = {}
```

**关键点**：
- `api_version = 10`：联机版当前 mod API 版本，单机版是 6
- `dst_compatible = true`：联机版兼容标志
- `all_clients_require_mod = true`：因为我们会添加新 prefab（带动画/物理），所有客户端必须加载

#### modmain.lua 最小模板

```lua
PrefabFiles =
{
    "myreviver",   -- 对应 scripts/prefabs/myreviver.lua
}

Assets =
{
    -- 如果有自定义贴图/动画再加
    -- Asset("ANIM", "anim/myreviver.zip"),
}

-- 注册到 STRINGS（让游戏能显示物品名称和描述）
STRINGS.NAMES.MYREVIVER = "灵魂之心"
STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYREVIVER = "或许能让朋友再回来一次。"

-- 注册配方（让玩家能合成它）
local Recipe = GLOBAL.Recipe2
local TECH = GLOBAL.TECH
local Ingredient = GLOBAL.Ingredient

AddRecipe2("myreviver",
    { Ingredient("redgem", 1), Ingredient("nightmarefuel", 4), Ingredient("ghostflower", 1) },
    TECH.MAGIC_TWO,
    { atlas = "images/inventoryimages/myreviver.xml" }
)
```

#### scripts/prefabs/myreviver.lua 最小骨架（占位）

```lua
local assets =
{
    Asset("ANIM", "anim/bloodpump.zip"),  -- 暂时借用告密的心的动画
}

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("bloodpump")
    inst.AnimState:SetBuild("bloodpump")
    inst.AnimState:PlayAnimation("idle")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")

    return inst
end

return Prefab("myreviver", fn, assets)
```

#### 服务端/客户端代码分离——`TheWorld.ismastersim`

仔细看上面骨架里的这一行：

```lua
inst.entity:SetPristine()
if not TheWorld.ismastersim then
    return inst
end
```

这是**饥荒联机版 prefab 最关键的"客户端早返回"模式**：

| 阶段 | 谁执行 |
|------|--------|
| `entity:AddXxx()`（Transform/AnimState/Network 等）| 服务端 + 客户端**都执行** |
| `AnimState:SetBank/SetBuild/PlayAnimation` | 服务端 + 客户端**都执行**（客户端要画图）|
| `MakeInventoryPhysics`（物理）| 服务端 + 客户端**都执行**（客户端要碰撞）|
| `entity:SetPristine()` | 标记"以上是 pristine 状态"，**网络变量基础值在此固定** |
| `if not TheWorld.ismastersim then return inst end` | 客户端到此结束 |
| `AddComponent("inventoryitem")` 等组件 | **只在服务端执行**——组件大都是纯服务端逻辑 |

> **新手记忆**：`SetPristine()` 之前的代码是"客户端也要跑的"（视觉/物理/网络变量声明），之后的代码是"只有服务端要跑的"（组件、tag、监听器）。**不分离会导致客户端崩溃或表现异常**。

完成 mod 工程准备后，下面 19.8.2 开始写真正的实战代码。

---

### 19.8.2 实战 1：5 行核心代码做"复活药水"——reviver tag

**目标**：做一个可以被**活着的玩家拿在手里、对幽灵使用**就让他复活的物品（功能等同于告密的心）。

#### 核心原理回顾

19.4.6 已经讲过，告密的心的"被识别为复活道具"是靠**一个 tag**——`"reviver"`：

```48:79:scripts/prefabs/reviver.lua
local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("bloodpump")
    inst.AnimState:SetBuild("bloodpump")
    inst.AnimState:PlayAnimation("idle")

    inst:AddTag("reviver")
```

注意第 62 行 `inst:AddTag("reviver")`——这个 tag 是**在 `SetPristine` 之前添加**的，也就是说 tag 是 pristine 状态的一部分，客户端也能看到。

为什么必须客户端也能看到？因为**给幽灵选择"使用此物"动作的判断在客户端先做一次**（`componentactions.lua` 里检查），如果 tag 在客户端不可见，就无法弹出动作菜单。

而真正的"被使用后让幽灵复活"，则发生在服务端 `OnRespawnFromGhost`：

```593:594:scripts/prefabs/player_common_extensions.lua
    elseif data.source:HasTag("reviver") then
        inst:DoTaskInTime(0, DoActualRez, nil, data.source)
```

只要 source 有 `"reviver"` tag，就走"道具复活"分支，最终进入 `reviver_rebirth` 状态：

```410:412:scripts/prefabs/player_common_extensions.lua
		else -- Telltale Heart
	        inst.sg:GoToState("reviver_rebirth", item)
		end
```

#### 完整 prefab 代码

把 `scripts/prefabs/myreviver.lua` 改成下面这样：

```lua
local assets =
{
    Asset("ANIM", "anim/bloodpump.zip"),
}

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("bloodpump")
    inst.AnimState:SetBuild("bloodpump")
    inst.AnimState:PlayAnimation("idle")

    inst:AddTag("reviver")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")
    inst:AddComponent("tradable")

    MakeHauntableLaunch(inst)

    return inst
end

return Prefab("myreviver", fn, assets)
```

整段代码**实质上的 mod 逻辑只有 1 行**：`inst:AddTag("reviver")`。其他都是任何 inventoryitem 都需要的标准模板代码。

#### 为什么不需要"使用"组件？

你可能困惑：游戏里好像没有一个 `useableitem` 之类的组件被加到道具上，那玩家怎么"使用"它对幽灵复活？

答案在游戏引擎层面——**`reviver` tag 的存在让幽灵的 BufferedAction 系统自动识别它**。具体的"对幽灵使用"逻辑封装在 SGwilsonghost 状态图里以及 `playercontroller` 的右键 / 长按动作处理里，**只要你的物品有 `reviver` tag，游戏会自动把它当成"可以喂幽灵"的物品**。

#### `MakeHauntableLaunch(inst)` 的意义

```77:77:scripts/prefabs/reviver.lua
    MakeHauntableLaunch(inst)
```

`MakeHauntableLaunch` 是 `scripts/prefabs/winter_ornaments.lua` 之外更常见的 `scripts/components/hauntable.lua` 工厂函数（见 `scripts/components/hauntable.lua` 末尾）。它的效果是：**当幽灵骚扰你这个掉在地上的物品时，物品会被弹飞一段距离**（而不是触发复活——因为我们没加 HAUNT_INSTANT_REZ）。

这是一个"幽灵能玩弄它一下"的视觉趣味，不是必须的——但加了能让你的复活道具掉在地上时不那么呆板。

#### 完整流程

1. 玩家 A 用 `myreviver` 配方合成出一个"灵魂之心"
2. 玩家 B 死亡，变成幽灵
3. 玩家 A 拿着"灵魂之心"靠近幽灵 B，右键/长按目标
4. A 的客户端 → 服务端发起 `ACTIONS.GIVE`（或类似）行动 → 物品被给到 B（幽灵）
5. B 的 `OnRespawnFromGhost` 检测 `data.source:HasTag("reviver")` → 调 `DoActualRez(B, nil, source)`
6. DoActualRez → `inst.sg:GoToState("reviver_rebirth", item)` → 播放复活动画 → B 恢复人形

#### 三种身份核心提示

> **新手**：要做"队友能喂给幽灵复活"的道具，**只需要 1 个 tag**：`inst:AddTag("reviver")`。这个 tag 必须在 `SetPristine()` 之前添加。
> 
> **进阶**：`reviver` tag 是 `OnRespawnFromGhost`（`player_common_extensions.lua:593`）里检查的，命中后调 `DoActualRez(inst, nil, source)`——`source` 作为 `item` 参数传入。最终复活状态是 `reviver_rebirth`。
> 
> **老手**：这种 tag 驱动的设计意味着**它会自动消失（被消耗）**——因为 `reviver_rebirth` 状态会调用 `item:Remove()`（见 SGwilson.lua 中 `reviver_rebirth` 状态的 onexit）；如果想做"耐久版"复活心，需要在 `respawnfromghost` 事件中拦截、消耗 finiteuses 后阻止移除。

---

### 19.8.3 实战 2：骚扰即复活的"灵魂符石"——hauntable + HAUNT_INSTANT_REZ

**目标**：做一块**幽灵自己骚扰就能立即复活**的建筑/物品（功能等同于红色护符 + 触手石的复活路径）。

#### 核心原理回顾

19.3.2 和 19.4.4 都讲过，幽灵骚扰带 `hauntable` 组件、并且 `hauntvalue == TUNING.HAUNT_INSTANT_REZ` 的物体时，会触发 `respawnfromghost` 事件：

```82:97:scripts/components/hauntable.lua
function Hauntable:DoHaunt(doer)
    if self.onhaunt ~= nil then
        if self.inst.components.itemmimic then
            self.inst.components.itemmimic:TurnEvil(doer)
            return
        end
        self.haunted = self.onhaunt(self.inst, doer)
        if self.haunted then
            if doer ~= nil then
                if self.hauntvalue == TUNING.HAUNT_INSTANT_REZ and doer:HasTag("playerghost") then
                    doer:PushEvent("respawnfromghost", { source = self.inst })
                end
                if not self.no_wipe_value then
                    self.hauntvalue = nil
                end
            end
```

注意两个关键细节：

1. **必须 `onhaunt` 回调返回 true**——`hauntable` 组件初始化时默认 `self.onhaunt = DefaultOnHauntFn`（直接返回 true），所以**只要不主动 `SetOnHauntFn(nil)`，默认就是返回 true 的**：

```1:3:scripts/components/hauntable.lua
local function DefaultOnHauntFn(inst, haunter)
    return true
end
```

2. **`SetHauntValue` 自动设 `no_wipe_value = true`**——意味着调一次 `SetHauntValue(HAUNT_INSTANT_REZ)`，物品**可以被反复骚扰复活**（除非用 `finiteuses` 限制次数或 mod 自己拦截）：

```46:50:scripts/components/hauntable.lua
function Hauntable:SetHauntValue(val)
    if not val then return end
    self.hauntvalue = val
    self.no_wipe_value = true
end
```

> **注意点**：直接给 `hauntvalue` 字段赋值（如 `inst.components.hauntable.hauntvalue = X`）**不会**自动设置 `no_wipe_value`——只有走 `SetHauntValue` 的封装才会自动开启。这是新手最容易踩的坑（详见 19.8.11）。

#### 完整 prefab 代码

把 `scripts/prefabs/myreviver.lua` 改成下面这样（也可以再开一个 `mysoulstone.lua`）：

```lua
local assets =
{
    Asset("ANIM", "anim/orangestaff.zip"),
}

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("orangestaff")
    inst.AnimState:SetBuild("orangestaff")
    inst.AnimState:PlayAnimation("idle")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")

    inst:AddComponent("hauntable")
    inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)

    return inst
end

return Prefab("mysoulstone", fn, assets)
```

整段代码**实质上的 mod 逻辑只有 2 行**：

```lua
inst:AddComponent("hauntable")
inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
```

#### 工作流程

1. 玩家 A 制作"灵魂符石"，把它丢在地上
2. 玩家 B 死亡 → 变成幽灵
3. 幽灵 B 飘到地上的符石旁，发起 HAUNT 动作（左键点击）
4. `Hauntable:DoHaunt(B)` 触发 → `onhaunt` 默认返回 true → `haunted = true`
5. 检测 `hauntvalue == HAUNT_INSTANT_REZ` 且 `doer:HasTag("playerghost")` → `B:PushEvent("respawnfromghost", { source = self.inst })`
6. B 的 `OnRespawnFromGhost` 检测 `data.source` 不是 reviver、不是雕像、不是石头…… → 走 unsupported rez source 分支 `inst:DoTaskInTime(0, DoActualRez)`（source=nil, item=nil）

**等等！**这意味着用上面的代码 mod，玩家会"瞬间变回来但没有任何复活动画"！

因为 `DoActualRez` 在 source=nil 且 item=nil 的情况下，**不会进入任何 `source:HasTag("xxx")` 分支**，导致 `inst.sg` 不会切到任何 `*_rebirth` 状态，玩家就直接变回来了——很突兀。

#### 修正：让 source 进入兼容的分支

最简单的修正办法：**在 source 进入 OnRespawnFromGhost 之前，先把它伪造成"触手石"或"传送门"的 tag**。但更优雅的做法是看看现成分支的具体条件：

```607:611:scripts/prefabs/player_common_extensions.lua
    elseif data.source.prefab == "amulet"
        or data.source.prefab == "resurrectionstone"
        or data.source.prefab == "resurrectionstatue"
        or data.source:HasTag("multiplayer_portal") then
        inst:DoTaskInTime(9 * FRAMES, DoMoveToRezSource, data.source, --[[60-9]] 51 * FRAMES)
```

可以看到，**`HasTag("multiplayer_portal")` 是 tag 判断**——只要我们给自己的 prefab 加上 `"multiplayer_portal"` tag，OnRespawnFromGhost 就会进入"传送门复活"分支（有 9 帧延迟移动到符石的动画过渡）。

但这样有副作用：进入了传送门分支后，`DoActualRez` 会执行：

```385:389:scripts/prefabs/player_common_extensions.lua
        elseif source:HasTag("multiplayer_portal") then
            inst.components.health:DeltaPenalty(TUNING.PORTAL_HEALTH_PENALTY)

            source:PushEvent("rez_player")
            inst.sg:GoToState("portal_rez")
```

**会扣 25% 最大血量惩罚 + 进入 `portal_rez` 状态**。如果你不希望有惩罚——参考 19.4.10 给玩家加 `"noreviverhealthpenalty"` tag 是无效的（这个 tag 只对 reviver 物品分支有效）；正确做法是改 mod 配方/价格平衡设计，或者用 19.8.9 的 `AddClassPostConstruct` 注入自定义分支。

##### 简版折中：补一个 onhaunt 弹起特效，绕过"无动画"问题

如果 mod 设计上不在乎"复活动画"这一点视觉差异，可以**自己在 `SetOnHauntFn` 里加一个特效**，至少让骚扰那一刻有反馈：

```lua
inst:AddComponent("hauntable")
inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
    local x, y, z = inst.Transform:GetWorldPosition()
    SpawnPrefab("statue_transition").Transform:SetPosition(x, y, z)
    inst.SoundEmitter:PlaySound("dontstarve/common/touchstone_activate")
    inst:Remove()  -- 用完即销毁，免得 no_wipe_value 让它可以无限用
    return true     -- 必须返回 true，否则后续 HAUNT_INSTANT_REZ 不会触发
end)
```

**`return true` 极为关键**——如果忘了，`Hauntable:DoHaunt` 第 88 行的 `self.haunted = self.onhaunt(...)` 会得到 nil/false，**整段 if self.haunted then 跳过**，最终 respawnfromghost 不会推送。

#### 三种身份核心提示

> **新手**：要做"幽灵骚扰即复活"的物品，最少 2 行代码：`inst:AddComponent("hauntable")` + `inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)`。
> 
> **进阶**：`SetHauntValue(...)` 自动把 `no_wipe_value` 设为 true（可重复使用）；如果想用一次后消失，应该在 `SetOnHauntFn` 里手动 `inst:Remove()`，且回调必须 `return true` 才能让 hauntvalue 触发复活。
> 
> **老手**：源码里 `OnRespawnFromGhost` 没有针对自定义 prefab 的分支，因此自定义 mod 道具会走 "unsupported rez source"——玩家无复活动画瞬间复活。要解决这点，要么加 `"multiplayer_portal"` tag（有 25% 血量惩罚副作用），要么走 19.8.9 的 `AddClassPostConstruct` 注入自定义分支。

---

### 19.8.4 实战 3：多次使用的复活罐——finiteuses + no_wipe_value

**目标**：做一个有 5 次复活耐久的复活罐——可以被幽灵骚扰复活 5 次，第 6 次时罐子会消失。

#### 核心原理回顾

红色护符的设计就是"多次使用"的样板。看下它的核心代码：

```424:450:scripts/prefabs/amulet.lua
local function red()
    local inst = commonfn("redamulet", "resurrector", true)

    inst.scrapbook_specialinfo = "REDAMULET"

    if not TheWorld.ismastersim then
        return inst
    end

    -- red amulet now falls off on death, so you HAVE to haunt it
    -- This is more straightforward for prototype purposes, but has side effect of allowing amulet steals
    -- inst.components.inventoryitem.keepondeath = true

    inst.components.equippable:SetOnEquip(onequip_red)
    inst.components.equippable:SetOnUnequip(onunequip_red)
    inst.components.equippable:SetOnEquipToModel(onequiptomodel_red)

    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetOnFinished(inst.Remove)
    inst.components.finiteuses:SetMaxUses(TUNING.REDAMULET_USES)
    inst.components.finiteuses:SetUses(TUNING.REDAMULET_USES)

    inst:AddComponent("hauntable")
    inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)

    return inst
end
```

**几个关键设计**：

1. **`finiteuses` 组件**——给道具加耐久值
2. **`SetOnFinished(inst.Remove)`**——耐久归零时自动销毁
3. **`SetHauntValue(HAUNT_INSTANT_REZ)`**——隐含 `no_wipe_value = true`，骚扰复活后 hauntvalue 不会消失

#### 一个关键陷阱：finiteuses 不会自动随复活消耗

仔细看护符的代码，**`finiteuses` 实际上不是在骚扰复活时被消耗的**——它是在**装备状态下，每隔 `REDAMULET_CONVERSION_TIME` 秒回血时消耗 1 次**：

```11:18:scripts/prefabs/amulet.lua
local function healowner(inst, owner)
    if (owner.components.health and owner.components.health:IsHurt() and not owner.components.oldager)
    and (owner.components.hunger and owner.components.hunger.current > 5 )then
        owner.components.health:DoDelta(TUNING.REDAMULET_CONVERSION,false,"redamulet")
        owner.components.hunger:DoDelta(-TUNING.REDAMULET_CONVERSION)
        inst.components.finiteuses:Use(1)
    end
end
```

所以护符的"5 次复活"其实是因为**复活后会自动穿上一个新的全新护符**（DoActualRez 里 `inventory:Equip(source)`），穿上后这个新护符开始持续消耗——只要伤害足够它每秒都在被消耗，最终自然耗尽。

但**如果你只是骚扰复活、立即把护符再脱下来**，护符不会消耗任何耐久——这是红色护符的一个细节"BUG"或者说"特性"。

#### mod 实战：在骚扰回调中手动消耗 finiteuses

我们要做一个真正"每复活一次扣 1 点耐久"的道具，需要**自己在 onhaunt 回调里调 `finiteuses:Use(1)`**：

```lua
local assets =
{
    Asset("ANIM", "anim/feather_yellow.zip"),
}

local function OnHaunt(inst, haunter)
    if inst.components.finiteuses ~= nil and inst.components.finiteuses:GetUses() > 0 then
        local x, y, z = inst.Transform:GetWorldPosition()
        SpawnPrefab("statue_transition_2").Transform:SetPosition(x, y, z)
        inst.SoundEmitter:PlaySound("dontstarve/common/touchstone_activate")
        inst.components.finiteuses:Use(1)
        return true  -- 关键：必须返回 true，触发 HAUNT_INSTANT_REZ 分支
    end
    return false     -- 耐久用完不让复活
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("feather_yellow")
    inst.AnimState:SetBuild("feather_yellow")
    inst.AnimState:PlayAnimation("idle")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")
    inst:AddComponent("tradable")

    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(5)
    inst.components.finiteuses:SetUses(5)
    inst.components.finiteuses:SetOnFinished(inst.Remove)

    inst:AddComponent("hauntable")
    inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
    inst.components.hauntable:SetOnHauntFn(OnHaunt)

    return inst
end

return Prefab("myrevivejar", fn, assets)
```

#### 验证一遍 hauntable 调用流程

回到 `Hauntable:DoHaunt`（19.3.4 详解过）：

```82:97:scripts/components/hauntable.lua
function Hauntable:DoHaunt(doer)
    if self.onhaunt ~= nil then
        if self.inst.components.itemmimic then
            self.inst.components.itemmimic:TurnEvil(doer)
            return
        end
        self.haunted = self.onhaunt(self.inst, doer)
        if self.haunted then
            if doer ~= nil then
                if self.hauntvalue == TUNING.HAUNT_INSTANT_REZ and doer:HasTag("playerghost") then
                    doer:PushEvent("respawnfromghost", { source = self.inst })
                end
                if not self.no_wipe_value then
                    self.hauntvalue = nil
                end
            end
```

- 第 7 行：`self.haunted = self.onhaunt(self.inst, doer)`——我们的 `OnHaunt` 返回 true，所以 `self.haunted = true`
- 第 9 行：进入 if 块
- 第 11 行：`hauntvalue == HAUNT_INSTANT_REZ` 且 doer 是 playerghost——推送 `respawnfromghost`
- 第 14 行：`if not self.no_wipe_value then` ——因为我们调了 `SetHauntValue`，`no_wipe_value = true`，所以**这里不会清空 hauntvalue**——下次还能复活

#### 三种身份核心提示

> **新手**：要做"多次复活"的道具，组合 `finiteuses` + `hauntable:SetHauntValue(HAUNT_INSTANT_REZ)`，并把"消耗耐久"逻辑写到 `SetOnHauntFn` 的回调里。
> 
> **进阶**：`SetHauntValue` 自动开 `no_wipe_value = true`，所以 hauntvalue 不会因为一次骚扰就被清空；但红色护符的 finiteuses 不是骚扰时消耗的，而是装备后持续消耗，要做真正的"骚扰一次扣 1 耐久"必须在 `SetOnHauntFn` 里手动 `finiteuses:Use(1)`。
> 
> **老手**：要在耐久归零时阻止复活，让 `SetOnHauntFn` 在 `GetUses() <= 0` 时 return false——这样 `DoHaunt` 里的 if self.haunted 整块被跳过，respawnfromghost 不会推送，符合"耐久没了就不能复活"的逻辑。

---

### 19.8.5 实战 4：绑定式复活神龛——attunable 系统

**目标**：做一个仿肉雕像的复活建筑，但**绑定代价是消耗精神（sanity）而不是血量**——既能展示 `attunable` 系统的使用、又能体现"自定义绑定代价"。

#### 核心原理回顾

肉雕像的实现已经在 19.4.5 拆过了。看下完整的 attunable 注册部分：

```168:172:scripts/prefabs/resurrectionstatue.lua
    inst:AddComponent("attunable")
    inst.components.attunable:SetAttunableTag("remoteresurrector")
    inst.components.attunable:SetOnAttuneCostFn(onattunecost)
    inst.components.attunable:SetOnLinkFn(onlink)
    inst.components.attunable:SetOnUnlinkFn(onunlink)
```

四个回调分别是：
- `SetAttunableTag("remoteresurrector")`：**复用 vanilla 的 tag**——这样幽灵的 `REMOTERESURRECT` 动作能找到它
- `SetOnAttuneCostFn`：玩家请求绑定时的代价计算（可拒绝）
- `SetOnLinkFn`：绑定成功的回调（视觉效果等）
- `SetOnUnlinkFn`：解除绑定的回调

#### 关键代码片段：肉雕像的血量代价

```61:73:scripts/prefabs/resurrectionstatue.lua
local function onattunecost(inst, player)
    --round up health to match UI display
    local amount_required = player:HasTag("health_as_oldage") and math.ceil(TUNING.EFFIGY_HEALTH_PENALTY * TUNING.OLDAGE_HEALTH_SCALE) or TUNING.EFFIGY_HEALTH_PENALTY

    if player.components.health == nil or math.ceil(player.components.health.currenthealth) <= amount_required then
        --Don't die from attunement!
        return false, "NOHEALTH"
    end

    player:PushEvent("consumehealthcost")
    player.components.health:DoDelta(-TUNING.EFFIGY_HEALTH_PENALTY, false, "statue_attune", true, inst, true)
    return true
end
```

**两个返回值**：
- `return true` 表示"代价已扣，可以绑定"
- `return false, reason` 表示"代价不够，不能绑定"——`reason` 通常是字符串如 `"NOHEALTH"`，用于显示提示语

#### 完整 mod 代码：精神代价的复活神龛

`scripts/prefabs/myshrine.lua`：

```lua
require "prefabutil"

local assets =
{
    Asset("ANIM", "anim/wilsonstatue.zip"),  -- 暂时借用肉雕像的动画
    Asset("MINIMAP_IMAGE", "resurrect"),
}

local prefabs =
{
    "collapse_small",
    "collapse_big",
}

local SANITY_COST = 80  -- 绑定代价：80 点精神

local function onhammered(inst, worker)
    if inst.components.lootdropper ~= nil then
        inst.components.lootdropper:DropLoot()
    end
    local fx = SpawnPrefab("collapse_big")
    fx.Transform:SetPosition(inst.Transform:GetWorldPosition())
    fx:SetMaterial("wood")
    inst:Remove()
end

local function onattunecost(inst, player)
    if player.components.sanity == nil then
        return false, "NOSANITY"
    end
    -- 精神不足以支付时拒绝
    if player.components.sanity.current <= SANITY_COST then
        return false, "NOSANITY"
    end
    player.components.sanity:DoDelta(-SANITY_COST, false)
    return true
end

local function onlink(inst, player, isloading)
    if not isloading then
        inst.SoundEmitter:PlaySound("dontstarve/common/together/meat_effigy_attune/on")
        inst.AnimState:PlayAnimation("attune_on")
        inst.AnimState:PushAnimation("idle", false)
    end
end

local function onunlink(inst, player, isloading)
    if not (isloading or inst.AnimState:IsCurrentAnimation("attune_on")) then
        inst.SoundEmitter:PlaySound("dontstarve/common/together/meat_effigy_attune/off")
        inst.AnimState:PlayAnimation("attune_off")
        inst.AnimState:PushAnimation("idle", false)
    end
end

local function onbuilt(inst, data)
    -- 复用肉雕像的"建造时自动绑定不收费"hack
    inst.components.attunable:SetOnAttuneCostFn(nil)
    inst.components.attunable:SetOnLinkFn(nil)
    inst.components.attunable:SetOnUnlinkFn(nil)

    inst.AnimState:PlayAnimation("place")
    if inst.components.attunable:LinkToPlayer(data.builder) then
        inst.AnimState:PushAnimation("attune_on")
    end
    inst.AnimState:PushAnimation("idle", false)

    inst.components.attunable:SetOnAttuneCostFn(onattunecost)
    inst.components.attunable:SetOnLinkFn(onlink)
    inst.components.attunable:SetOnUnlinkFn(onunlink)
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddMiniMapEntity()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    inst:SetDeploySmartRadius(1)
    MakeObstaclePhysics(inst, .3)
    inst.MiniMapEntity:SetIcon("resurrect.png")

    inst:AddTag("structure")
    inst:AddTag("resurrector")

    inst.AnimState:SetBank("wilsonstatue")
    inst.AnimState:SetBuild("wilsonstatue")
    inst.AnimState:PlayAnimation("idle")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inspectable")
    inst:AddComponent("lootdropper")

    inst:AddComponent("workable")
    inst.components.workable:SetWorkAction(ACTIONS.HAMMER)
    inst.components.workable:SetWorkLeft(4)
    inst.components.workable:SetOnFinishCallback(onhammered)
    inst:ListenForEvent("onbuilt", onbuilt)

    -- attunable 配置——核心
    inst:AddComponent("attunable")
    inst.components.attunable:SetAttunableTag("remoteresurrector")  -- 复用 vanilla tag
    inst.components.attunable:SetOnAttuneCostFn(onattunecost)
    inst.components.attunable:SetOnLinkFn(onlink)
    inst.components.attunable:SetOnUnlinkFn(onunlink)

    -- 用过即销毁
    inst:ListenForEvent("activateresurrection", inst.Remove)

    return inst
end

return Prefab("myshrine", fn, assets, prefabs),
    MakePlacer("myshrine_placer", "wilsonstatue", "wilsonstatue", "idle")
```

#### 工作流程

1. 玩家 A 建造 `myshrine`，建造时**不扣精神**（建造材料本身已经是代价）
   - `onbuilt` 临时把 `OnAttuneCostFn` 设为 nil，调 `LinkToPlayer(builder)` → 不进入 `onattunecost` 分支
2. 玩家 B 想绑定同一座神龛 → 走 `LinkToPlayer(B)` → 触发 `onattunecost(inst, B)`
   - 检查 B 的精神 > 80 → 扣 80 精神 → return true → 绑定成功
   - 否则 return false, "NOSANITY" → 弹提示"精神不足"
3. 玩家 A 死亡变幽灵 → 在 HUD 看到 `REMOTERESURRECT` 选项（因为他绑定过这个 `"remoteresurrector"` tag 的实体）
4. A 选择 REMOTERESURRECT → `ACTIONS.REMOTERESURRECT.fn` → `attuner:GetAttunedTarget("remoteresurrector")` → 找到 myshrine
5. PushEvent `respawnfromghost`，source = myshrine → `OnRespawnFromGhost` 走 `data.source.prefab == "resurrectionstatue"` 吗？**NO！**——这里 source 的 prefab 是 `"myshrine"`，不匹配任何分支
6. 进入 unsupported rez source 分支，瞬间复活无动画
7. `DoActualRez(inst)` 内 `source` 参数没传（nil），但又因为 `data.source.components.attunable:GetAttunableTag() == "remoteresurrector"` → **设置 `inst.remoterezsource = true`**

如何修复"无动画"？两条路：
- 改 `SetAttunableTag` 用一个新 tag（如 `"my_shrine_resurrector"`），然后 mod 自己实现一个新的 REMOTERESURRECT action？太复杂。
- 或者**就用现有 vanilla tag `"remoteresurrector"`**，让玩家自然能用 REMOTERESURRECT 动作；只是没有 `rebirth` 动画——可以接受。
- 又或者用 19.8.9 的 `AddClassPostConstruct` 给 `DoActualRez` 加自定义分支。

#### attunable 系统的几个隐含规则

| 规则 | 来源 |
|------|------|
| 同一玩家在同一 tag 下**只能绑定一个**实体——绑定新的会自动解除旧的 | `attunable.lua:20-30` `onplayerattuned` |
| 玩家离线时绑定信息保存为 `attuned_userids[userid]`，玩家上线时自动重连 | `attunable.lua:38-43` `onplayerjoined` |
| 实体销毁（Remove）时，所有绑定它的玩家会自动 UnlinkFromPlayer | `attunable.lua:48-58` `OnRemoveEntity` |
| 玩家移除（Remove）时（如重连断线），绑定信息从 `attuned_players` 转入 `attuned_userids` | `attunable.lua:33-36` `onplayerremoved` |
| 客户端通过 `attunable_classified` 代理实体看到绑定关系，**不直接访问实体** | `attunable.lua:105` `SpawnPrefab("attunable_classified")` |
| 一次性使用：监听 `activateresurrection` 事件自动 Remove | `resurrectionstatue.lua:176` |

#### 三种身份核心提示

> **新手**：用 `attunable` 系统的 5 行核心代码：`AddComponent` + `SetAttunableTag("remoteresurrector")` + `SetOnAttuneCostFn` + `SetOnLinkFn` + `SetOnUnlinkFn`。再加上 `ListenForEvent("activateresurrection", inst.Remove)` 让它"用过即销毁"。
> 
> **进阶**：`SetOnAttuneCostFn` 必须返回 `true` 或 `false, reason`；reason 是字符串，用来显示"代价不足"提示语。建造时通过临时清除 `OnAttuneCostFn` 实现"建造者免费自动绑定"。
> 
> **老手**：`attunable` 的网络同步靠 `attunable_classified` 代理实体——客户端通过 `attuner.attuned[guid]` 拿到的是代理而不是实际实体；离线玩家的绑定靠 `attuned_userids` 字典保存 userid，登录时 `onplayerjoined` 自动重链；`UnlinkFromPlayer(p, true)` 第二个参数 isloading 用于区分"加载存档触发的解链"和"主动解链"——后者会播音效，前者不会。

---

### 19.8.6 实战 5：复活后给玩家 5 秒无敌 + 满血——监听 ms_respawnedfromghost

**目标**：做一个**全局生效的"复活保护"系统**——不论玩家用什么方式复活，复活完成后给他 5 秒无敌 + 满血。

#### 核心原理回顾

19.7.8 讲过，复活完成后会推送 `ms_respawnedfromghost` 事件——这是给 mod 挂"复活后清理/初始化"的标准钩子：

```424:430:scripts/prefabs/player_common_extensions.lua
    CommonActualRez(inst)

    inst:RemoveTag("playerghost")
    inst.Network:RemoveUserFlag(USERFLAGS.IS_GHOST)

    inst:PushEvent("ms_respawnedfromghost")
```

事件推送在 `DoActualRez` 的最末尾——`CommonActualRez` 已经跑完、所有组件已经恢复、`playerghost` tag 已经移除——这时挂上的 buff 不会被"二次清理"。

#### 写在 modmain.lua 的两种实现

**方案 1：用 `PrefabPostInit("player_common", ...)`**——给所有玩家 prefab 都注入

```lua
-- 在 modmain.lua 里
local INVINCIBLE_TIME = 5

local function OnRespawnedFromGhost(inst)
    inst.components.health:SetInvincible(true)
    inst.components.health:DoDelta(inst.components.health.maxhealth, true, "myreviveprotection", true)
    
    -- 5 秒后取消无敌
    inst:DoTaskInTime(INVINCIBLE_TIME, function()
        if inst:IsValid() and inst.components.health then
            inst.components.health:SetInvincible(false)
        end
    end)
end

AddPrefabPostInit("player_common", function(inst)
    if not GLOBAL.TheWorld.ismastersim then
        return
    end
    -- 仅在服务端添加监听器（这是 player 通用 prefab 后处理）
    inst:ListenForEvent("ms_respawnedfromghost", OnRespawnedFromGhost)
end)
```

**为什么用 `AddPrefabPostInit("player_common", ...)` 而不是 `PrefabPostInit` 各角色？**

因为 `player_common.lua` 是**所有角色的共通构造**——`wilson.lua`、`wendy.lua`、`willow.lua` 等都是基于它扩展。在 `player_common` 上挂监听，所有玩家角色都能继承。

**方案 2：监听 `ms_playerjoined`（更精确）**

```lua
local function OnPlayerJoined(world, player)
    player:ListenForEvent("ms_respawnedfromghost", OnRespawnedFromGhost)
end

AddPrefabPostInit("world", function(inst)
    if not GLOBAL.TheWorld.ismastersim then
        return
    end
    inst:ListenForEvent("ms_playerjoined", OnPlayerJoined)
end)
```

每当玩家加入（包括首次连接、断线重连）就给他挂监听——比 PrefabPostInit 更稳定，因为后者偶尔会在某些 mod 顺序下不触发。

#### 详细 API 解释

| API 调用 | 作用 |
|---------|------|
| `health:SetInvincible(true)` | 把 `health.invincible` 设为 true——所有 `health:DoDelta` 调用如果 `ignore_invincible` 不为 true，都会被无视 |
| `health:DoDelta(maxhealth, true, "myreviveprotection", true)` | 第 1 参数：变化量（满血量）；第 2 参数：overtime（瞬间生效）；第 3 参数：cause（用于 entity_death 事件）；第 4 参数：ignore_invincible（**必须 true**，否则上一行 SetInvincible 会让本次 DoDelta 静默失败）|
| `DoTaskInTime(time, fn)` | 注册一次性延时任务——5 秒后再调 SetInvincible(false) |

**关键陷阱**：`DoDelta` 第 4 个参数 ignore_invincible 必须 true，否则"先 SetInvincible(true) 再 DoDelta(maxhealth)"的"满血"会失败——`DoDelta` 内部检查 `if not ignore_invincible and self.invincible then return end`。这也是 19.6.5 `OnRespawnFromPlayerCorpse` 里所有 DoDelta 都带 `ignore_invincible=true` 的原因。

#### 完整 modmain.lua 示例

```lua
PrefabFiles = {
    -- 你的 prefab 文件
}

Assets = {}

local INVINCIBLE_TIME = 5

local function OnRespawnedFromGhost(inst)
    if inst.components.health == nil then return end

    inst.components.health:SetInvincible(true)
    inst.components.health:DoDelta(
        inst.components.health.maxhealth,
        true,                       -- overtime
        "myreviveprotection",       -- cause
        true,                       -- ignore_invincible（必须为 true）
        nil,                        -- afflicter
        true                        -- silent（不弹"挨打"数字）
    )

    -- 视觉效果：闪烁防止队友打你
    if inst.AnimState then
        inst.AnimState:SetMultColour(1, 1, 1, 0.6)
    end

    inst:DoTaskInTime(INVINCIBLE_TIME, function()
        if inst:IsValid() and inst.components.health then
            inst.components.health:SetInvincible(false)
            if inst.AnimState then
                inst.AnimState:SetMultColour(1, 1, 1, 1)
            end
        end
    end)
end

local function OnPlayerJoined(world, player)
    player:ListenForEvent("ms_respawnedfromghost", OnRespawnedFromGhost)
end

AddPrefabPostInit("world", function(inst)
    if not GLOBAL.TheWorld.ismastersim then
        return
    end
    inst:ListenForEvent("ms_playerjoined", OnPlayerJoined)
end)
```

#### 配对使用：监听 `ms_becameghost` 添加复活前 buff

实战中，"复活无敌 + 满血"通常和"死亡时记录什么状态"配对——比如**死亡时记录玩家死前的 buff**，复活时**自动复原**：

```lua
local function OnBecameGhost(inst)
    -- 记录死亡时的关键状态（mod 自定义字段）
    inst._mybuffsnapshot = {
        was_buffed = inst:HasTag("my_super_buff"),
        last_resource = inst.my_mod_resource,
    }
end

local function OnRespawnedFromGhost(inst)
    -- 还原快照
    if inst._mybuffsnapshot ~= nil then
        if inst._mybuffsnapshot.was_buffed then
            inst:AddTag("my_super_buff")
        end
        inst.my_mod_resource = inst._mybuffsnapshot.last_resource
        inst._mybuffsnapshot = nil
    end
    -- ... 上面的无敌 + 满血代码 ...
end

local function OnPlayerJoined(world, player)
    player:ListenForEvent("ms_becameghost", OnBecameGhost)
    player:ListenForEvent("ms_respawnedfromghost", OnRespawnedFromGhost)
end
```

`ms_becameghost` 推送时机比 `CommonPlayerDeath` 略晚——它在 `OnMakePlayerGhost` 最后推送，此时玩家已经是幽灵了，但所有 mod-state 还在。

#### 三种身份核心提示

> **新手**：复活后挂 buff 用 `inst:ListenForEvent("ms_respawnedfromghost", fn)`；事件在 `DoActualRez` 末尾推送，所有组件都已恢复。
> 
> **进阶**：用 `AddPrefabPostInit("world", ...)` 监听 `ms_playerjoined` 是最稳定的挂监听姿势——比直接 `AddPrefabPostInit("player_common", ...)` 更不容易被其他 mod 干扰。`DoDelta(maxhealth, true, cause, true)` 第 4 参数 `ignore_invincible` 必须 true，否则 SetInvincible(true) 后的 DoDelta 会被静默吃掉。
> 
> **老手**：`ms_becameghost` 和 `ms_respawnedfromghost` 必须配对使用——任何"死亡时记录"和"复活时还原"的 mod 状态都靠这一对事件。监听器记得做 `nil` 防御：玩家在事件触发时刚刚 RemoveTag("playerghost") 还没退出 `respawnedfromghost` 处理流程，组件理论上都在；但 DoTaskInTime 延时回调可能跨多次实体生命周期，所以 5 秒后回调里**必须 `IsValid` + 组件存在性检查**。

---

### 19.8.7 实战 6：死亡时永久扣 mod 货币——监听 ms_becameghost

**目标**：做一个"死亡惩罚"系统——玩家死亡后，永久扣除 50 点 mod 自定义货币（如果不够则归零）。

#### 核心原理回顾

19.7.8 讲过两个事件的精确触发时机：

| 事件 | 推送时机 | 状态 |
|------|---------|------|
| `"death"` | `Health:SetVal` 内血量降到 0 | inst 还是活的，组件全在 |
| `"ms_becameghost"` | `OnMakePlayerGhost` 末尾 | inst 已变成幽灵，部分组件已被 CommonPlayerDeath 移除 |
| `"playerdied"` | `OnPlayerDied`（ghost_enabled=false 时）| 玩家即将 fadeout 然后被删 |
| `"ms_respawnedfromghost"` | `DoActualRez` 末尾 | 玩家已经复活、playerghost tag 已 RemoveTag |

**选哪个？**

- 如果是"幽灵期间也要扣"——用 `"ms_becameghost"`（保证只在变幽灵时扣一次）
- 如果是"无论变幽灵还是直接消失都要扣"——用 `"death"`（更通用）
- 如果只想在"真正复活时扣"——用 `"ms_respawnedfromghost"`（玩家可能死了再也不上线，这种情况不扣）

下面用 `"death"` 实现，这是最通用的选择。

#### 完整 modmain.lua 示例

```lua
PrefabFiles = {}
Assets = {}

local PENALTY_AMOUNT = 50

local function OnDeath(inst, data)
    -- data.cause: 死因（字符串，如 "spider" 或 "starve"）
    -- data.afflicter: 凶手实体（可能是 nil）
    -- data.corpsing: bool，如果走 corpse 模式则为 true

    if inst.my_mod_currency == nil then
        inst.my_mod_currency = 0
    end

    inst.my_mod_currency = math.max(0, inst.my_mod_currency - PENALTY_AMOUNT)

    -- 即时显示提示
    if inst.components.talker then
        inst.components.talker:Say("我失去了 "..PENALTY_AMOUNT.." 灵魂币……")
    end
end

local function OnSaveCurrency(inst, data)
    data.my_mod_currency = inst.my_mod_currency or 0
end

local function OnLoadCurrency(inst, data)
    if data ~= nil and data.my_mod_currency ~= nil then
        inst.my_mod_currency = data.my_mod_currency
    else
        inst.my_mod_currency = 100  -- 初始值
    end
end

local function OnPlayerJoined(world, player)
    player:ListenForEvent("death", OnDeath)
    -- 也可以监听 ms_becameghost 实现"变幽灵才扣"的语义
    -- player:ListenForEvent("ms_becameghost", OnDeath)
end

AddPrefabPostInit("world", function(inst)
    if not GLOBAL.TheWorld.ismastersim then
        return
    end
    inst:ListenForEvent("ms_playerjoined", OnPlayerJoined)
end)

-- 给玩家 prefab 注入存档/读档逻辑
AddPlayerPostInit(function(inst)
    if not GLOBAL.TheWorld.ismastersim then
        return
    end
    -- 用 AddComponentPostInit 或直接挂 OnSave / OnLoad 钩子
    local oldOnSave = inst.OnSave
    inst.OnSave = function(self, data)
        if oldOnSave then oldOnSave(self, data) end
        OnSaveCurrency(self, data)
    end
    local oldOnLoad = inst.OnLoad
    inst.OnLoad = function(self, data)
        if oldOnLoad then oldOnLoad(self, data) end
        OnLoadCurrency(self, data)
    end
end)
```

#### "data" 字段说明（`death` 事件）

从 19.1.5 我们知道，`death` 事件的 data 是这样推送的：

```scripts/components/health.lua
self.inst:PushEvent("death", { cause = cause, afflicter = afflicter, corpsing = corpsing })
```

| 字段 | 类型 | 含义 |
|------|------|------|
| `cause` | string | 死因字符串，比如 `"spider"`、`"hunger"`、`"redamulet"`（最近一次伤害的 cause 参数）|
| `afflicter` | Entity \| nil | 最后一击的攻击者（可能是 nil，比如饥饿死亡）|
| `corpsing` | bool | 是否走 corpse 模式——通常用 `revivable_corpse` 游戏模式时为 true |

**`cause` 字段可以用于差异化惩罚**：

```lua
local function OnDeath(inst, data)
    local cause = data and data.cause or "unknown"

    local penalty = 50
    if cause == "drowning" then
        penalty = 100   -- 淹死惩罚加倍
    elseif cause == "hunger" then
        penalty = 30    -- 饿死惩罚减少
    elseif cause == "boss_kill" then
        penalty = 0     -- 自定义 cause："被 boss 杀死不算"
    end

    inst.my_mod_currency = math.max(0, (inst.my_mod_currency or 0) - penalty)
end
```

#### 注意点：玩家网络变量同步

如果 `my_mod_currency` 需要在客户端 UI 显示，**不能直接挂在 `inst.my_mod_currency`** 上——这只在服务端有效。需要用 `net_int`：

```lua
-- 在 PrefabPostInit 或者一个 PlayerPostInit 里
local function OnCurrencyDirty(inst)
    -- 客户端：从网络变量读取并更新 UI
    inst:PushEvent("my_currency_changed", { value = inst._my_mod_currency:value() })
end

AddPlayerPostInit(function(inst)
    inst._my_mod_currency = net_int(inst.GUID, "my_mod_currency", "my_currency_dirty")
    inst:ListenForEvent("my_currency_dirty", OnCurrencyDirty)
    
    if GLOBAL.TheWorld.ismastersim then
        -- 服务端：监听字段变化，自动同步网络变量
        inst._my_mod_currency:set(0)
    end
end)
```

然后服务端的 OnDeath 改成：

```lua
local function OnDeath(inst, data)
    if inst._my_mod_currency == nil then return end
    local current = inst._my_mod_currency:value()
    local new_value = math.max(0, current - PENALTY_AMOUNT)
    inst._my_mod_currency:set(new_value)  -- 自动 dirty，客户端会收到事件
end
```

详细的网络变量用法见 19.8.10（实战 9）。

#### 三种身份核心提示

> **新手**：用 `inst:ListenForEvent("death", fn)` 监听玩家死亡——data 字段含 `cause`、`afflicter`、`corpsing`。在回调里改 inst 上的自定义字段，OnSave / OnLoad 负责持久化。
> 
> **进阶**：选事件要看语义——`death` 是"血量归零的瞬间"（最通用，含变幽灵和直接消失两种情况），`ms_becameghost` 是"成功变幽灵后"（不含 ghost_enabled=false 的情况）；用 `data.cause` 字段可以差异化处理不同死因。
> 
> **老手**：mod 状态字段如果要客户端可见（UI 显示等），必须用 `net_int / net_string / net_uint / net_byte / net_bool` 网络变量同步——直接 `inst.my_field` 只在服务端可见。OnSave/OnLoad 需要钩在 inst 的 OnSave/OnLoad 上（包装旧函数），或者把字段交给一个组件管理（更工程化）。

---

### 19.8.8 实战 7：自定义骨架 prefab——替换 mod 角色的死亡遗体

**目标**：给一个 mod 角色配置专属骨架——死亡后掉的不是普通的 `skeleton_player`，而是 mod 自定义的 `myhero_skeleton`，可以有不同的外观、特殊掉落或彩蛋。

#### 核心原理回顾

19.1.3 讲过，`OnMakePlayerGhost` 里两个条件都满足才会生成骨架：

```104:145:scripts/prefabs/player_common_extensions.lua
local function SpawnDeathProduct(inst)
    -- ...
    if can_corpse then
        local corpse = SpawnPrefab("playercorpse")
        -- ...
        return DEATH_PRODUCTS.CORPSE
    else
        local has_skeletons = TheSim:HasPlayerSkeletons()
        local skel = SpawnPrefab(has_skeletons and inst.skeleton_prefab or "shallow_grave_player")
```

关键字段是 **`inst.skeleton_prefab`**——`player_common.lua:2910` 默认设为 `"skeleton_player"`：

```2910:2910:scripts/prefabs/player_common.lua
		inst.skeleton_prefab = "skeleton_player"
```

如果 mod 角色想要专属骨架，**在 `master_postinit` 里把 `inst.skeleton_prefab` 重新赋值**即可：

```scripts/prefabs/wanda.lua:434
inst.skeleton_prefab = nil
```

例如 Wanda 设为 `nil`——意味着 Wanda 死后不留骨架（因为她有 `wanda_diary` 机制）。

#### 完整实战代码

**步骤 1：定义 mod 角色的 `skeleton_prefab`**

`scripts/prefabs/myhero.lua`（角色 prefab）：

```lua
local MakePlayerCharacter = require "prefabs/player_common"

local prefabs = { "myhero_skeleton" }  -- 注册依赖关系，让游戏知道有这个prefab

local function master_postinit(inst)
    inst.skeleton_prefab = "myhero_skeleton"  -- 关键：替换骨架 prefab

    -- 其他角色逻辑（属性、皮肤等）
    inst.components.health:SetMaxHealth(150)
    -- ...
end

return MakePlayerCharacter("myhero",
    prefabs,                   -- prefabs
    {},                        -- assets
    common_postinit,           -- 客户端初始化（可省略）
    master_postinit)           -- 服务端初始化
```

**步骤 2：实现自定义骨架 prefab**

`scripts/prefabs/myhero_skeleton.lua`：

```lua
local assets =
{
    Asset("ANIM", "anim/skeletons.zip"),
}

local prefabs =
{
    "boneshard",
    "redgem",          -- 自定义掉落：红宝石
    "collapse_small",
}

-- 复用普通骨架的掉落表格式
SetSharedLootTable('myhero_skeleton',
{
    { 'boneshard', 1.00 },
    { 'boneshard', 1.00 },
    { 'redgem',    0.30 },  -- 30% 概率掉红宝石（"心碎了"的彩蛋）
})

local function Player_SetSkeletonDescription(inst, char, playername, cause, pkname, userid)
    inst.char = char
    inst.playername = playername
    inst.userid = userid
    inst.pkname = pkname
    inst.cause = pkname == nil and cause:lower() or nil
    inst.components.inspectable.getspecialdescription = GetPlayerDeathDescription
end

local function OnHammered(inst)
    inst.components.lootdropper:DropLoot()
    local fx = SpawnPrefab("collapse_small")
    fx.Transform:SetPosition(inst.Transform:GetWorldPosition())
    fx:SetMaterial("rock")
    inst:Remove()
end

local function Player_OnSave(inst, data)
    data.char = inst.char
    data.playername = inst.playername
    data.userid = inst.userid
    data.pkname = inst.pkname
    data.cause = inst.cause
end

local function Player_OnLoad(inst, data)
    if not data or not data.char then
        return
    end
    Player_SetSkeletonDescription(inst, data.char, data.playername, data.cause, data.pkname, data.userid)
end

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeObstaclePhysics(inst, .25)

    inst:AddTag("skeleton")
    inst:AddTag("playerskeleton")  -- 关键：让幽灵能交互
    inst:AddTag("scarytoprey")      -- 兔子等怕骷髅
    inst:AddTag("structure")

    inst.AnimState:SetBank("skeletons")
    inst.AnimState:SetBuild("skeletons")
    inst.AnimState:PlayAnimation("player1")

    -- 自定义颜色让它显眼一点
    inst.AnimState:SetMultColour(1, 0.7, 0.7, 1)

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inspectable")

    inst:AddComponent("lootdropper")
    inst.components.lootdropper:SetChanceLootTable('myhero_skeleton')

    inst:AddComponent("workable")
    inst.components.workable:SetWorkAction(ACTIONS.HAMMER)
    inst.components.workable:SetWorkLeft(3)
    inst.components.workable:SetOnFinishCallback(OnHammered)

    -- 幽灵可以骚扰骨架（标准玩家骨架行为）
    MakeHauntable(inst, 6, TUNING.HAUNT_COOLDOWN_TINY)

    -- 提供"设置死亡描述"的方法供 player_common_extensions 调用
    inst.SetSkeletonDescription = Player_SetSkeletonDescription

    -- 提供"设置头像数据"的方法（可选）
    inst:AddComponent("playeravatardata")
    inst.SetSkeletonAvatarData = function(inst, client_obj)
        inst.components.playeravatardata:SetData(client_obj)
    end

    inst.OnSave = Player_OnSave
    inst.OnLoad = Player_OnLoad

    return inst
end

return Prefab("myhero_skeleton", fn, assets, prefabs)
```

**步骤 3：在 modmain.lua 注册**

```lua
PrefabFiles = {
    "myhero",
    "myhero_skeleton",
}
```

#### 验证：SpawnDeathProduct 如何使用这个 prefab？

回到 `player_common_extensions.lua`：

```135:144:scripts/prefabs/player_common_extensions.lua
    else
        local has_skeletons = TheSim:HasPlayerSkeletons()
        local skel = SpawnPrefab(has_skeletons and inst.skeleton_prefab or "shallow_grave_player")
        if skel ~= nil then
            skel.Transform:SetPosition(x, y, z)
            -- Set the description
            skel:SetSkeletonDescription(inst.prefab, inst:GetDisplayName(), inst.deathcause, inst.deathpkname, inst.userid)
            skel:SetSkeletonAvatarData(inst.deathclientobj)
        end
        return has_skeletons and DEATH_PRODUCTS.SKELETON or DEATH_PRODUCTS.SHALLOW_GRAVE
    end
```

游戏会**直接调用** `skel:SetSkeletonDescription(...)` 和 `skel:SetSkeletonAvatarData(...)`——注意源码**没有任何 nil 检查**！如果你的自定义骨架 prefab 没实现这两个方法，游戏会直接抛出 `attempt to call method 'SetSkeletonDescription' (a nil value)`。

这就是为什么前面的实战代码里**必须**写：

```lua
inst.SetSkeletonDescription = Player_SetSkeletonDescription
inst.SetSkeletonAvatarData = function(inst, client_obj)
    inst.components.playeravatardata:SetData(client_obj)
end
```

#### 进阶：自定义骨架的额外效果

如果你想让骨架有更多 mod 特性——比如"骚扰它发光 5 秒"或"死亡 7 天后自动消散"——可以在 fn() 里加更多组件：

```lua
-- 让骨架在 7 天（默认游戏内 7 天 = 7 * TUNING.TOTAL_DAY_TIME 秒）后自动消失
inst:DoTaskInTime(7 * TUNING.TOTAL_DAY_TIME, function(inst)
    if inst:IsValid() then
        local x, y, z = inst.Transform:GetWorldPosition()
        SpawnPrefab("ash").Transform:SetPosition(x, y, z)
        SpawnPrefab("collapse_small").Transform:SetPosition(x, y, z)
        inst:Remove()
    end
end)
```

参考 `scripts/prefabs/skeleton.lua` 的 `Player_Decay` 实现。

#### 关键 tag 总结

| tag | 作用 | 必须 |
|-----|------|------|
| `"skeleton"` | 标记为骨架类型 | 强烈推荐 |
| `"playerskeleton"` | 玩家骨架——某些 mod 和 AI 会区分 | 强烈推荐 |
| `"scarytoprey"` | 让兔子、海狸等怕它逃跑 | 看 mod 需要 |
| `"structure"` | 标记为建筑类（不被推动、可铺路）| 看 mod 需要 |
| `"haunted"` | 由 hauntable 组件自动管理 | 不要手动加 |

#### 三种身份核心提示

> **新手**：替换 mod 角色的骨架只需 1 行：`inst.skeleton_prefab = "myhero_skeleton"`（在 `master_postinit` 里）。然后实现这个 prefab。
> 
> **进阶**：自定义骨架 prefab 必须实现 `SetSkeletonDescription(inst, char, playername, cause, pkname, userid)` 和 `SetSkeletonAvatarData(inst, client_obj)` 两个方法——游戏会主动调用它们。
> 
> **老手**：设 `inst.skeleton_prefab = nil` 表示"死亡不留骨架"（Wanda 的设计）；移动端没有玩家骨架（`TheSim:HasPlayerSkeletons() == false`），此时 fallback 到 `"shallow_grave_player"`——如果想让移动端也用自定义浅坑，需要再做一个 `myhero_shallow_grave` prefab 并替换 fallback（但 fallback 在 vanilla 是硬编码的，要 mod 进 player_common_extensions 较复杂，不推荐）。

---

### 19.8.9 实战 8：监听 `respawnfromghost` 给玩家加自定义后处理

**目标**：在玩家用自定义复活道具复活后，**抢在 `DoActualRez` 之前/之后**做一些特殊事情——比如**给玩家挂一个 mod buff、改变复活位置、强制切换角色形态**。

#### 为什么不能直接用 `AddClassPostConstruct`？

19.8.6 已经讲了 `ms_respawnedfromghost` 监听——但它在 `DoActualRez` 末尾才触发，**复活动画已经结束、所有组件已经恢复**——晚了。

`DoActualRez` 本身是 `player_common_extensions.lua` 里的 **local function**：

```327:431:scripts/prefabs/player_common_extensions.lua
local function DoActualRez(inst, source, item)
    -- ...
```

`local function` **不能被 `AddClassPostConstruct` 修改**——它是模块作用域的私有函数，外部完全访问不到。

如果你硬要"插队"到 DoActualRez 中间，**只能监听 `respawnfromghost` 事件本身**：

```569:594:scripts/prefabs/player_common_extensions.lua
local function OnRespawnFromGhost(inst, data) -- from ListenForEvent "respawnfromghost"
    if not inst:HasTag("playerghost") then
        return
    end

	inst:AddTag("reviving")

    inst.deathclientobj = nil
    inst.deathcause = nil
    inst.deathpkname = nil
    inst.deathbypet = nil
    inst:ShowHUD(false)
    -- ...
    if data == nil or data.source == nil then
        inst:DoTaskInTime(0, DoActualRez)
    elseif inst.sg.currentstate.name == "remoteresurrect" then
        inst:DoTaskInTime(0, DoMoveToRezSource, data.source, 24 * FRAMES)
    elseif data.source:HasTag("reviver") then
        inst:DoTaskInTime(0, DoActualRez, nil, data.source)
```

`OnRespawnFromGhost` 在 `player_common.lua` 里通过 `inst:ListenForEvent("respawnfromghost", OnRespawnFromGhost)` 挂上。注意 `DoTaskInTime(0, DoActualRez)` ——延后到下一帧执行，给了 mod **同一帧"早一步处理"**的窗口。

#### 实战方案：在 `respawnfromghost` 监听里抢先做事

```lua
-- modmain.lua

local function OnRespawnFromGhostPre(inst, data)
    -- 因为 vanilla 是 inst:DoTaskInTime(0, DoActualRez)
    -- mod 的监听器和 vanilla 的监听器都立即执行
    -- vanilla 的 DoActualRez 被延迟到下一帧
    -- 因此 mod 可以在同一帧抢先改变 data 或者添加效果

    if data == nil or data.source == nil then
        return
    end

    -- 例 1：自定义 prefab 的复活道具——给玩家保留 50% 饥饿（默认是 66.7%）
    if data.source.prefab == "myreviver" then
        inst._mod_post_resurrect_hunger = 0.5
    end

    -- 例 2：在自定义条件下，让玩家在固定坐标复活（无视 source 位置）
    if data.source:HasTag("homebound_reviver") and inst._mod_home_position ~= nil then
        -- 直接传送（DoActualRez 内的 Physics:Teleport 会在下一帧覆盖这个位置——参见 line 354）
        -- 所以这里改 source 的位置或者改 data.source 引用
        -- 更可靠的办法：把 source 替换为一个临时 prefab，位于 home 位置
    end
end

local function OnRespawnedFromGhost(inst, data)
    -- 这是 ms_respawnedfromghost 事件——DoActualRez 末尾推送
    -- 此时 inst 已经复活、所有组件恢复
    if inst._mod_post_resurrect_hunger ~= nil and inst.components.hunger ~= nil then
        inst.components.hunger:SetPercent(inst._mod_post_resurrect_hunger)
        inst._mod_post_resurrect_hunger = nil
    end
end

local function OnPlayerJoined(world, player)
    player:ListenForEvent("respawnfromghost", OnRespawnFromGhostPre)
    player:ListenForEvent("ms_respawnedfromghost", OnRespawnedFromGhost)
end

AddPrefabPostInit("world", function(inst)
    if not GLOBAL.TheWorld.ismastersim then
        return
    end
    inst:ListenForEvent("ms_playerjoined", OnPlayerJoined)
end)
```

#### 监听器的执行顺序——为什么 mod 监听能"先"于 DoActualRez？

`ListenForEvent` 注册的所有回调，**在事件推送时按注册顺序执行**——但都在**同一帧**完成。

vanilla 的 `OnRespawnFromGhost` 在 `player_common.lua` 里通过 `inst:ListenForEvent("respawnfromghost", OnRespawnFromGhost)` 注册——注册时机是**玩家 prefab 构造时**。

mod 的监听器在 `ms_playerjoined` 时注册——**晚于** vanilla。

按 ListenForEvent 顺序，**vanilla 的回调先执行**——但是它内部用 `DoTaskInTime(0, DoActualRez)` **延迟一帧**——所以 mod 监听器实际上**在同一帧执行**完，然后**下一帧** DoActualRez 才执行。

这给 mod 一个抢跑窗口：

```
帧 N：
  respawnfromghost 事件推送
    ├─ vanilla OnRespawnFromGhost 执行（设 inst.rezsource、DoTaskInTime(0, DoActualRez)）
    └─ mod 监听器执行（设 inst._mod_post_resurrect_hunger = 0.5）

帧 N+1：
  DoActualRez 执行（CommonActualRez 把 hunger 设为 66.7%）
  事件 ms_respawnedfromghost 推送
    ├─ vanilla 监听器（如有）
    └─ mod 监听器 OnRespawnedFromGhost
         └─ 把 hunger 改回 50%（盖掉 CommonActualRez 的 66.7%）
```

#### 复活源限定：判断 source 类型

回顾下 OnRespawnFromGhost 里对 source 的判断：

```607:614:scripts/prefabs/player_common_extensions.lua
    elseif data.source.prefab == "amulet"
        or data.source.prefab == "resurrectionstone"
        or data.source.prefab == "resurrectionstatue"
        or data.source:HasTag("multiplayer_portal") then
        inst:DoTaskInTime(9 * FRAMES, DoMoveToRezSource, data.source, --[[60-9]] 51 * FRAMES)
    else
        --unsupported rez source...
        inst:DoTaskInTime(0, DoActualRez)
    end
```

vanilla 的"已知 source"判断完全靠 **`prefab == "xxx"`** 或 **`HasTag("xxx")`**。

**自定义 mod prefab 不会出现在这个列表里**——所以会走 `unsupported rez source` 分支，无动画瞬间复活。

如果你想让自定义 prefab 也有"幽灵移动到 source 位置 + 等 51 帧再复活"的过渡，**最简单的方法**是让你的 prefab 加上 `"multiplayer_portal"` tag（隐含 25% 血量惩罚，已在 19.8.3 讨论）；**完美的方法**是用监听器拦截：

```lua
local function OnRespawnFromGhostPre(inst, data)
    if data and data.source and data.source.prefab == "myreviver_building" then
        -- 模拟传送门的 9-frame 延迟 + 51-frame 移动
        -- 但要避免真正进入 DoActualRez 的 multiplayer_portal 分支（不想要 DeltaPenalty）
        -- ……非常麻烦，需要 Hook
    end
end
```

不幸的是，因为 `DoActualRez` 是 local function，**不存在干净的拦截办法**。三种"够用"的折中：

1. **接受 vanilla 的 unsupported 分支**——玩家瞬间复活无动画，但代码简单可控
2. **加 `"multiplayer_portal"` tag**——有动画但有血量惩罚，再用 mod 监听 `ms_respawnedfromghost` 调 `health:DeltaPenalty(-0.25)` **抵消**
3. **mod 内重新实现 DoActualRez**——监听到 `respawnfromghost` 后，**自己接管**整个复活流程（不靠 vanilla）

#### 方案 2 的"抵消传送门惩罚"实现

```lua
local function OnRespawnedFromGhost(inst, data)
    -- 如果上一次复活源是我们的自定义建筑（mod 自己记录），抵消传送门惩罚
    if inst._mod_no_penalty_pending then
        if inst.components.health and inst.components.health.penalty > 0 then
            inst.components.health:DeltaPenalty(-TUNING.PORTAL_HEALTH_PENALTY)
        end
        inst._mod_no_penalty_pending = nil
    end
end

local function OnRespawnFromGhostPre(inst, data)
    if data and data.source and data.source.prefab == "myreviver_building" then
        -- prefab 添加 "multiplayer_portal" tag——见你的 myreviver_building.lua
        inst._mod_no_penalty_pending = true
    end
end
```

然后在 `myreviver_building.lua` 里加：

```lua
inst:AddTag("multiplayer_portal")
```

这样玩家会走 vanilla 的传送门动画分支（`portal_rez` 状态），但 mod 监听器抵消掉 25% 惩罚——视觉上美观、玩法上无副作用。

#### 三种身份核心提示

> **新手**：`respawnfromghost` 事件可以监听——它在 vanilla `OnRespawnFromGhost` 之前/同时触发；mod 监听器抢在 `DoActualRez` 之前修改 inst 字段，再在 `ms_respawnedfromghost` 监听里"补打"逻辑。
> 
> **进阶**：vanilla 的 `OnRespawnFromGhost` 用 `DoTaskInTime(0, DoActualRez)` 延迟一帧——这给了 mod 监听器同一帧抢跑的窗口。利用这个窗口设置 mod 字段，DoActualRez 跑完后用 `ms_respawnedfromghost` 监听器读取这些字段做最终修正。
> 
> **老手**：`DoActualRez` 是 local function 无法 hook；想让自定义 prefab 有复活动画的最简洁姿势是加 `"multiplayer_portal"` tag（走 portal_rez 动画）+ mod 监听器抵消血量惩罚。`prefab == "xxx"` 和 `HasTag("xxx")` 是 vanilla 唯二判断复活源类型的方式，没有官方扩展点。

---

### 19.8.10 实战 9：客户端网络同步——让复活道具的 UI 表现一致

**目标**：让自定义复活道具的耐久/状态在**客户端 UI** 上能正确显示——通过 `net_*` 网络变量同步。

#### 为什么需要网络同步？

19.8.1 已经讲过服务端/客户端代码分离。一个常见误区是：**`inst:AddComponent("finiteuses")` 是服务端的**——所以客户端访问 `inst.components.finiteuses` 是 `nil`。

但是 `finiteuses` 是**特殊的**：它通过 `inventoryitem_classified` 中的网络变量 `percentused` 同步耐久百分比到客户端：

```163:165:scripts/prefabs/inventoryitem_classified.lua
    inst.percentused = net_byte(inst.GUID, "inventoryitem.percentused", "percentuseddirty")
    inst.recharge = net_byte(inst.GUID, "inventoryitem.recharge", "rechargedirty")
```

`net_byte`：占 1 字节，值范围 0~255，自动同步到所有客户端，**变化时推送 `percentuseddirty` 事件**。

#### 实战目标：自定义"灵魂能量"状态显示

我们希望做一个**复活之刃**——它有一个 mod 独有的"灵魂能量"字段（0~100），需要在客户端 UI 上显示。

**步骤 1：定义网络变量（prefab 内）**

```lua
local function OnSoulEnergyDirty(inst)
    -- 客户端：从网络变量读取并触发自定义事件
    inst:PushEvent("soulenergy_changed", { value = inst._soulenergy:value() })
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)
    inst.AnimState:SetBank("nightsword")
    inst.AnimState:SetBuild("nightsword")
    inst.AnimState:PlayAnimation("idle")

    -- 关键：网络变量必须在 SetPristine 之前定义
    inst._soulenergy = net_byte(inst.GUID, "myrevivalsword.soulenergy", "soulenergydirty")
    inst._soulenergy:set(100)  -- 默认满能量

    inst:ListenForEvent("soulenergydirty", OnSoulEnergyDirty)

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    -- 服务端逻辑
    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")

    -- 模拟服务端不断消耗
    inst:DoPeriodicTask(1, function(inst)
        local cur = inst._soulenergy:value()
        if cur > 0 then
            inst._soulenergy:set(cur - 1)  -- 自动 dirty，客户端收到事件
        end
    end)

    return inst
end

return Prefab("myrevivalsword", fn, assets)
```

#### 网络变量的几个关键规则

```lua
inst._soulenergy = net_byte(inst.GUID, "myrevivalsword.soulenergy", "soulenergydirty")
```

| 参数 | 含义 |
|------|------|
| `inst.GUID` | 实体的唯一 ID，让网络变量绑定到这个实体 |
| `"myrevivalsword.soulenergy"` | 字段名（必须**全局唯一**——所有 prefab 不能重名）|
| `"soulenergydirty"` | 当服务端调用 `set` 改变值时，**客户端**收到的事件名 |

**网络变量类型对照**：

| 类型 | 范围 | 字节数 |
|------|------|--------|
| `net_bool` | true / false | < 1 字节 |
| `net_byte` | 0~255 | 1 字节 |
| `net_tinybyte` | 0~7 | 3 bit |
| `net_smallbyte` | 0~63 | 6 bit |
| `net_shortint` | -32768~32767 | 2 字节 |
| `net_ushortint` | 0~65535 | 2 字节 |
| `net_int` | int32 范围 | 4 字节 |
| `net_uint` | uint32 范围 | 4 字节 |
| `net_float` | float32 | 4 字节 |
| `net_string` | 字符串（注意有最大长度）| 变长 |
| `net_hash` | 字符串 hash（更省带宽）| 4 字节 |
| `net_entity` | 引用另一个实体 | 4 字节（GUID）|

> **优化提示**：能用 `net_byte` 别用 `net_int`——带宽差 4 倍；能用 `net_hash` 别用 `net_string`——字符串同步开销大。

#### 客户端读取方式

```lua
-- 客户端任意时机：
local energy = inst._soulenergy:value()  -- 返回当前同步过来的值

-- 监听变化（在 ListenForEvent 中订阅 dirty 事件）：
inst:ListenForEvent("soulenergydirty", function(inst)
    local new_energy = inst._soulenergy:value()
    print("能量变了：", new_energy)
end)
```

#### UI 显示——基于网络变量驱动

对于 inventoryitem 上的 mod 数值显示，一个常见做法是**用 `inst.components.inventoryitem.getstatusoverride`**（在客户端的 inventoryitem_replica 上）或者**自定义 widget**。

简单方案：用 `inst.components.inspectable.getspecialdescription`：

```lua
-- 服务端 fn 里：
inst.components.inspectable.getspecialdescription = function(inst, viewer)
    return "灵魂能量: "..tostring(inst._soulenergy:value()).."/100"
end
```

这样玩家"检查"道具时会显示当前能量。

#### 用 `replica` 同步复杂数据

如果要同步**结构化数据**（如一个表 `{ users_revived = {…}, last_used_time = … }`），不能用网络变量——需要用 **`replica` 系统**或者把数据序列化为 string。

最简单的方式：

```lua
-- 序列化字典为字符串
local data_str = string.format("%d|%d", users_revived_count, last_used_time)
inst._mod_data:set(data_str)
```

注意 `net_string` 有长度限制（具体值随版本不同，一般是 200 字符以内）。

#### 三种身份核心提示

> **新手**：要让 mod 字段在客户端可见，用 `net_byte/net_int/net_string` 网络变量；网络变量必须在 `SetPristine()` **之前**定义，否则客户端拿不到初始值。
> 
> **进阶**：网络变量字段名（第 2 个参数）必须**全局唯一**——所有 prefab 不能重名，否则 dirty 事件会串。建议加 prefab 前缀避免冲突，如 `"myrevivalsword.soulenergy"`。
> 
> **老手**：能用 `net_byte` 不要用 `net_int`——带宽差 4 倍；能用 `net_hash` 不要用 `net_string`——字符串同步开销大；变化频繁的字段考虑用 `replica` 代理对象（如 `inventoryitem_classified`）做缓存而不是每个变化都同步；做"结构化数据"的同步，要么序列化为字符串，要么用多个网络变量分散同步。

---

### 19.8.11 老手：五个最常见的实战陷阱

#### 坑 1：忘记 `inst:AddTag("reviver")` 必须在 `SetPristine()` 之前

```lua
-- 错误示范
local function fn()
    local inst = CreateEntity()
    -- ... transform / animstate / network ...
    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddTag("reviver")  -- ❌ 这一行只在服务端生效
end
```

**症状**：服务端能 `inst:HasTag("reviver")` 检查到，但客户端 `HasTag` 返回 false——结果**幽灵看不到"使用此道具"的右键菜单**（因为客户端 componentactions 校验失败）。

**修复**：

```lua
inst:AddTag("reviver")    -- ✅ 在 SetPristine 之前
inst.entity:SetPristine()
if not TheWorld.ismastersim then
    return inst
end
```

> **规则**：**所有需要客户端能 HasTag 到的 tag** 都必须在 `SetPristine()` 之前添加；只在服务端逻辑里检查的 tag 可以放后面（但通常没必要）。

#### 坑 2：`SetOnHauntFn` 没有返回 true，导致 HAUNT_INSTANT_REZ 不生效

```lua
-- 错误示范
inst:AddComponent("hauntable")
inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)
inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
    inst.SoundEmitter:PlaySound("dontstarve/common/touchstone_activate")
    -- ❌ 忘了 return true
end)
```

**症状**：幽灵骚扰道具，能听到音效，但**玩家没有复活**——hauntvalue 检查整段被跳过。

**修复**：

```lua
inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
    inst.SoundEmitter:PlaySound("dontstarve/common/touchstone_activate")
    return true   -- ✅ 必须显式返回 true
end)
```

回顾 `Hauntable:DoHaunt`：

```scripts/components/hauntable.lua
self.haunted = self.onhaunt(self.inst, doer)
if self.haunted then
    if doer ~= nil then
        if self.hauntvalue == TUNING.HAUNT_INSTANT_REZ and doer:HasTag("playerghost") then
            doer:PushEvent("respawnfromghost", { source = self.inst })
        end
```

`self.haunted` 是 onhaunt 的返回值——为 false 时整段被跳过。

> **规则**：自定义 `SetOnHauntFn` 回调**必须显式 `return true`**——Lua 函数无 return 等价于 `return nil`，等价于 false。

#### 坑 3：直接给 `hauntvalue` 字段赋值，导致 `no_wipe_value` 没有被设置

```lua
-- 错误示范
inst:AddComponent("hauntable")
inst.components.hauntable.hauntvalue = TUNING.HAUNT_INSTANT_REZ  -- ❌
```

**症状**：第一次骚扰复活成功，第二次骚扰**hauntvalue 已经被清空**，无法复活。

**原因**：直接赋值不会设置 `no_wipe_value = true`。`Hauntable:DoHaunt` 在成功 onhaunt 之后：

```scripts/components/hauntable.lua
if not self.no_wipe_value then
    self.hauntvalue = nil
end
```

如果 `no_wipe_value = false`（默认），hauntvalue 会被一次性清空。

**修复**：

```lua
inst.components.hauntable:SetHauntValue(TUNING.HAUNT_INSTANT_REZ)  -- ✅ 调用 setter
```

回顾 `SetHauntValue`：

```46:50:scripts/components/hauntable.lua
function Hauntable:SetHauntValue(val)
    if not val then return end
    self.hauntvalue = val
    self.no_wipe_value = true
end
```

**只有走 setter 才会自动设置 `no_wipe_value`**。直接给字段赋值漏掉了这一关键步骤。

> **规则**：**永远用 setter API**——`SetHauntValue`、`SetOnHauntFn`、`SetOnUnHauntFn`。直接字段赋值会跳过组件的封装逻辑。

#### 坑 4：`attunable` 系统的 `LinkToPlayer` 在 master_postinit 里调用，玩家还没加载完

```lua
-- 错误示范（modmain.lua）
AddPrefabPostInit("myshrine", function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end
    -- 想让某个固定 player_id 自动绑定
    local player = GLOBAL.LookupPlayerInstByUserID("KU_xxx")
    if player and inst.components.attunable then
        inst.components.attunable:LinkToPlayer(player)  -- ❌ 可能 player 是 nil
    end
end)
```

**症状**：服务器启动时，建筑 prefab post init 时玩家还没加入，`LookupPlayerInstByUserID` 返回 nil，绑定失败。

**修复**：在 `ms_playerjoined` 事件里再尝试：

```lua
-- ✅ 正确做法
AddPrefabPostInit("myshrine", function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end

    local function TryLinkToTargetPlayer(world, player)
        if player.userid == "KU_xxx" and inst.components.attunable then
            inst.components.attunable:LinkToPlayer(player)
            GLOBAL.TheWorld:RemoveEventCallback("ms_playerjoined", TryLinkToTargetPlayer)
        end
    end

    -- 已在线的玩家？
    for _, p in ipairs(GLOBAL.AllPlayers) do
        if p.userid == "KU_xxx" then
            inst.components.attunable:LinkToPlayer(p)
            return
        end
    end

    -- 没在线就等 ms_playerjoined
    GLOBAL.TheWorld:ListenForEvent("ms_playerjoined", TryLinkToTargetPlayer)
end)
```

> **规则**：涉及"按玩家 userid 操作"的代码，必须考虑 **玩家不在线** 的情况——监听 `ms_playerjoined` 是最稳健的兜底；或者用 `attuned_userids` 字段（attunable 系统已经替你做了离线/在线的过渡，见 `attunable.lua:38-43`）。

#### 坑 5：复活后 mod 重新加 burnable / freezable，配置在第二次死亡复活时丢失

```lua
-- 错误示范
local function masterpostinit(inst)
    GLOBAL.MakeLargeBurnableCharacter(inst, "torso")
    inst.components.burnable:SetBurnTime(100)   -- 自定义 100 秒烧时间
end
```

**症状**：玩家第一次出生时燃烧时间 100 秒；死亡复活后变回 10 秒（vanilla 默认）。

**原因**：19.7 已经详细讲过——`CommonActualRez` 会**重新创建** `burnable / freezable / grogginess / slipperyfeet` 四个组件：

```scripts/prefabs/player_common_extensions.lua
-- CommonActualRez 内部
MakeMediumBurnableCharacter(inst, "torso")  -- 这一行会覆盖 mod 配置
MakeLargeFreezableCharacter(inst, "torso")
inst:AddComponent("grogginess")
inst:AddComponent("slipperyfeet")
```

**修复**：把 mod 配置封装为函数，在 `ms_respawnedfromghost` 中重新调用：

```lua
local function ConfigureMyBurnable(inst)
    if inst.components.burnable then
        inst.components.burnable:SetBurnTime(100)
    end
end

local function masterpostinit(inst)
    GLOBAL.MakeLargeBurnableCharacter(inst, "torso")
    ConfigureMyBurnable(inst)
    inst:ListenForEvent("ms_respawnedfromghost", ConfigureMyBurnable)
end
```

> **规则**：对 vanilla 的 `burnable/freezable/grogginess/slipperyfeet` 四个组件做 mod 配置，**必须在 `ms_respawnedfromghost` 里再来一遍**——这四个是死亡时移除、复活时重新创建的特殊组件。

---

### 19.8 小结

```
本节实战回顾：

┌────────────────────────────────────────────────────────────────┐
│ 9 个实战的"最小化代码量"对照表                                 │
├────────────────────────────────────────────────────────────────┤
│ 19.8.2 reviver tag 复活药：    inst:AddTag("reviver")           │
│ 19.8.3 骚扰即复活灵符：        hauntable:SetHauntValue(...)     │
│ 19.8.4 5 次耐久复活罐：        + finiteuses + 自定义 OnHauntFn  │
│ 19.8.5 attunable 神龛：        attunable + Set 4 个回调          │
│ 19.8.6 复活后无敌+满血：       监听 ms_respawnedfromghost       │
│ 19.8.7 死亡扣 mod 货币：       监听 death / ms_becameghost      │
│ 19.8.8 自定义骨架 prefab：     inst.skeleton_prefab = "xxx"    │
│ 19.8.9 监听 respawnfromghost： 在 DoActualRez 之前抢跑          │
│ 19.8.10 客户端 UI 网络同步：   net_byte/int/hash + dirty event │
└────────────────────────────────────────────────────────────────┘

复活道具的"识别机制"汇总：
  reviver tag                → OnRespawnFromGhost 走 reviver 分支
  hauntable + HAUNT_INSTANT_REZ → 骚扰时 PushEvent("respawnfromghost")
  attunable("remoteresurrector") → REMOTERESURRECT 动作 + attuner 查找
  multiplayer_portal tag     → DoActualRez 走 portal_rez 动画（+25% penalty）

复活道具的"动画分支"对照：
  prefab == "amulet"            → amulet_rebirth + Equip
  prefab == "resurrectionstone" → wakeup + Hide(inventory)
  prefab == "resurrectionstatue" → rebirth(source)
  HasTag("multiplayer_portal")  → portal_rez + DeltaPenalty(0.25)
  HasTag("reviver")             → reviver_rebirth(item)
  pocketwatch_revive             → rewindtime_rebirth
  其他自定义 prefab              → 无动画瞬间复活

事件钩子的标准用法：
  PrefabPostInit("world") + ms_playerjoined → 给每个玩家挂监听
    ├─ death (玩家挂掉瞬间)
    ├─ ms_becameghost (变幽灵完成)
    ├─ respawnfromghost (复活流程开始，可抢跑)
    └─ ms_respawnedfromghost (复活流程结束，最稳定的"复活后"挂载点)

服务端/客户端代码分离：
  inst.entity:SetPristine()
  if not TheWorld.ismastersim then return inst end
    ↑ 之前：transform / animstate / physics / network / tag / 网络变量
    ↓ 之后：组件 / OnSave / OnLoad / 监听器

网络变量速选：
  bool          → net_bool
  小整数 0-255  → net_byte
  整数 ±32k     → net_shortint
  整数更大      → net_int / net_uint
  字符串 hash   → net_hash（首选）
  字符串原文    → net_string
  实体引用      → net_entity
```

**新手核心三句**：要做"队友能用"的复活道具就加 `inst:AddTag("reviver")`，要做"幽灵自己用"的就加 `hauntable:SetHauntValue(HAUNT_INSTANT_REZ)`，要在复活后加 buff 就监听 `ms_respawnedfromghost`——三种最常用 mod 复活模式都只需 1-2 行核心代码。

**进阶核心三句**：`finiteuses` 不会自动随骚扰复活消耗，必须在 `SetOnHauntFn` 里手动 `Use(1)` 并显式 `return true`；`attunable` 系统通过 `SetAttunableTag` 把建筑加入玩家 `attuner` 的"绑定池"，复用 `"remoteresurrector"` tag 能直接让幽灵的 REMOTERESURRECT 动作识别它；mod 的 `respawnfromghost` 监听利用 vanilla 的 `DoTaskInTime(0, DoActualRez)` 一帧延迟做抢跑——同一帧改 inst 字段、下一帧 DoActualRez 跑完后 `ms_respawnedfromghost` 监听器做最后修正。

**老手核心三句**：自定义骨架 prefab 必须实现 `SetSkeletonDescription` 和 `SetSkeletonAvatarData` 两个函数，否则 `SpawnDeathProduct` 调用时报 nil；`DoActualRez` 是 local function 无法 hook，"完美的"自定义复活动画只能通过加 `"multiplayer_portal"` tag 走 portal_rez + mod 监听抵消惩罚的方式实现；客户端 UI 显示 mod 字段必须用 `net_*` 网络变量同步——字段名要全局唯一，能用 `net_byte` 不用 `net_int`，能用 `net_hash` 不用 `net_string`。

---


