# 第18章 生存系统

## 18.1 Health 组件——血量、无敌与伤害处理

### 本节导读

> **一句话定位**：`Health` 是饥荒中**所有可被伤害 entity 的"生命状态机"**——它不只存一个数字，而是管理着当前血量、最大血量、无敌状态、惩罚上限、吸收系数、回血任务、火焰伤害……并在任何血量变化时广播事件驱动后续逻辑——这是 mod 开发中**使用频率最高**的核心组件之一。

#### 一段宏观先讲清楚：一次受伤经历了什么

玩家被蜘蛛攻击，血量从 100 降到 85，游戏里这 15 点伤害是怎么结算的？

```
【蜘蛛 combat:Attack → target.components.health:DoDelta(-15, ...)】
            │
            ▼
    Step 1: nonlethal 检查
            血量 ≤ nonlethal_pct(20%) 且 cause=hunger/cold/hot？→ 直接返回 0
            │ 否则继续
            ▼
    Step 2: redirect 检查
            health.redirect 函数存在？→ 交给 redirect 处理，返回 0
            │ 否则继续
            ▼
    Step 3: 无敌检查
            IsInvincible() or inst.is_teleporting？→ 直接返回 0
            │ 否则继续
            ▼
    Step 4: 吸收系数计算
            amount = amount * (1 - absorb - playerabsorb)
                           * max(1 - externalabsorbmodifiers, 0)
            amount = min(0, amount + externalreductionmodifiers)
            │
            ▼
    Step 5: deltamodifierfn 自定义修饰
            amount = deltamodifierfn(inst, amount, ...) （如有设置）
            │
            ▼
    Step 6: maxdamagetakenperhit 上限
            amount = max(amount, maxdamagetakenperhit) （如有设置）
            │
            ▼
    Step 7: SetVal 写入血量
            currenthealth = currenthealth + amount → 触发 oncurrenthealth
            │
            ▼
    Step 8: 推送事件
            PushEvent("healthdelta", { oldpercent, newpercent, cause, afflicter, ... })
            当 currenthealth ≤ 0：PushEvent("death") + 世界事件 "entity_death"
```

#### 15.4 节回答的 6 个核心问题

```
Q1: ──── 如何设置一个 entity 的初始血量？
         ↓ 答：inst:AddComponent("health") + health:SetMaxHealth(n)

Q2: ──── 如何让 entity 扣血 / 加血？
         ↓ 答：health:DoDelta(-n, ...) 扣血；health:DoDelta(n, true, "regen") 加血

Q3: ──── 如何让 entity 变成无敌状态（godmode）？
         ↓ 答：health:SetInvincible(true)；或 SG 的 temp_invincible 标签

Q4: ──── 死亡惩罚（上限血量降低）是怎么实现的？
         ↓ 答：health.penalty [0~0.75]；GetMaxWithPenalty() = maxhealth * (1-penalty)

Q5: ──── 如何拦截所有伤害，做"转移伤害"效果？
         ↓ 答：health.redirect = function(inst, amount, ...) ... return true end

Q6: ──── 联机时客户端如何读到血量？
         ↓ 答：health_replica 组件 + player_classified 的 net 变量
```

#### 你将看到的核心源码

| 文件 | 行 | 用途 |
|---|---|---|
| `scripts/components/health.lua` | 全文 665 行 | Health 组件主体 |
| `scripts/components/health_replica.lua` | 全文 216 行 | 客户端血量只读接口 |
| `scripts/prefabs/player_common.lua` | 2744-2749 | 玩家 health 初始化 |
| `scripts/tuning.lua` | 38/93/2150-2153 | 关键常量：wilson_health/MAX_FIRE/PENALTY 上限 |
| `scripts/prefabs/wanda.lua` | 412-414 | redirect + canheal=false 实例 |
| `scripts/prefabs/wendy.lua` | 398 | redirect 转移伤害到 Abigail |

#### 本节学习路径

```
18.1.1 新手 ─── Health 三个核心概念 —— currenthealth / maxhealth / penalty
18.1.2 新手 ─── SetMaxHealth / DoDelta —— 最常用的两个函数
18.1.3 新手 ─── healthdelta 事件 —— 监听血量变化
                  ↓
18.1.4 进阶 ─── DoDelta 完整伤害计算链 —— 7 步结算流程
18.1.5 进阶 ─── 无敌机制 —— SetInvincible / temp_invincible / ForceKill
18.1.6 进阶 ─── penalty 惩罚系统 —— 死亡上限惩罚
18.1.7 进阶 ─── 火焰伤害专项 —— fire_damage_scale + DoFireDamage
                  ↓
18.1.8 老手 ─── redirect 钩子 —— 拦截所有伤害
18.1.9 老手 ─── 回血系统 —— StartRegen / AddRegenSource
18.1.10 老手 ─── 联机同步机制 —— health_replica 与 classified
18.1.11     ─── 小结·速查表 + mod 实战 4 模板
```

---

### 18.1.1（新手）Health 三个核心概念

#### 概念一：currenthealth —— 当前血量

```62:66:scripts/components/health.lua
local Health = Class(function(self, inst)
    self.inst = inst
    self.maxhealth = 100
    self.minhealth = 0
    self.currenthealth = self.maxhealth
```

| 字段 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `currenthealth` | number | = maxhealth | 当前血量，≤ 0 时角色死亡 |
| `maxhealth` | number | 100 | 最大血量（不含惩罚） |
| `minhealth` | number | 0 | 血量下限（通常为 0；设正值可让 entity 永不死亡） |

**当 `currenthealth ≤ 0` 时**：
- `SetVal()` 内检测到变化 → 推送 `"death"` 事件
- 世界推送 `"entity_death"`
- 若 `nofadeout = false`（默认）→ entity 2 秒后自动消失

#### 概念二：maxhealth —— 上限血量

`maxhealth` 是纯粹的数值上限。玩家角色的默认上限：

```38:38:scripts/tuning.lua
    local wilson_health = 150
```

所有角色的默认最大血量都等于 `wilson_health = 150`（除特殊角色外）。设置方法：

```503:507:scripts/components/health.lua
function Health:SetMaxHealth(amount)
    self.maxhealth = amount
    self.currenthealth = amount
    self:ForceUpdateHUD(true) --handles capping health at max with penalty
end
```

> **注意**：`SetMaxHealth` 会**同时将 currenthealth 设为 amount**（完全回满血）。如果只想修改上限而不满血，应先调用 `SetMaxHealth`，再用 `DoDelta` 或 `SetPercent` 调整当前血量。

#### 概念三：penalty —— 血量惩罚系统

`penalty` 是 [0, 0.75] 范围内的**百分比惩罚**，使实际可用血量上限降低：

```524:526:scripts/components/health.lua
function Health:GetMaxWithPenalty()
    return self.maxhealth - self.maxhealth * self.penalty
end
```

**示例**：`maxhealth = 150`，`penalty = 0.25`：
- 实际可用上限 = 150 - 150 × 0.25 = **112.5**
- 血量即使满格，UI 显示的最大值也是 112.5
- 最大惩罚为 `MAXIMUM_HEALTH_PENALTY = 0.75`，即最少只剩 25% 上限

```2153:2153:scripts/tuning.lua
        MAXIMUM_HEALTH_PENALTY = 0.75,
```

> **penalty 存档**：penalty 值在 `OnSave`/`OnLoad` 中被序列化，玩家死亡后重新进入游戏依然有惩罚。

#### 三个概念的关系图

```
健康状态示意（maxhealth=150, penalty=0.25, currenthealth=100）：
│
│  maxhealth = 150 ──────────────────────────────────────────────│
│                                                                 │
│  GetMaxWithPenalty() = 112.5 ───────────────────────────────┐  │
│                                                              │  │
│  currenthealth = 100 ──────────────────────────────────┐    │  │
│                                                         │    │  │
│  0 ─────────────────────────────────────────────────────────────│
│
│  HUD 显示：100 / 112.5
│  GetPercent() = 100/150 = 0.667
│  GetPercentWithPenalty() = 100/112.5 = 0.889
```

---

### 18.1.2（新手）SetMaxHealth / DoDelta —— 最常用的两个函数

#### SetMaxHealth —— 初始化血量上限

```503:507:scripts/components/health.lua
function Health:SetMaxHealth(amount)
    self.maxhealth = amount
    self.currenthealth = amount
    self:ForceUpdateHUD(true) --handles capping health at max with penalty
end
```

**参数**：
- `amount`（number）：新的最大血量，同时将 currenthealth 也设为该值

**典型使用位置**：prefab 的 `master_postinit` 中：

```2744:2745:scripts/prefabs/player_common.lua
        inst:AddComponent("health")
        inst.components.health:SetMaxHealth(TUNING.WILSON_HEALTH)
```

#### DoDelta —— 核心伤害/回血函数

```613:649:scripts/components/health.lua
function Health:DoDelta(amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb)
    local old_percent = self:GetPercent()

    if old_percent <= self.nonlethal_pct and
		(((cause == "cold" or cause == "hot") and self.nonlethal_temperature) or
		(cause == "hunger" and self.nonlethal_hunger)) then

        return 0
    end

    if self.redirect ~= nil and self.redirect(self.inst, amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb) then
        return 0
    elseif not ignore_invincible and (self:IsInvincible() or self.inst.is_teleporting) then
        return 0
    elseif amount < 0 and not ignore_absorb then        
        amount = amount * math.clamp(1 - (self.playerabsorb ~= 0 and afflicter ~= nil and afflicter:HasTag("player") and self.playerabsorb + self.absorb or self.absorb), 0, 1) * math.max(1 - self.externalabsorbmodifiers:Get(), 0)
        amount = afflicter ~= nil and math.min(0, amount + self.externalreductionmodifiers:Get()) or amount
    end

    if self.deltamodifierfn ~= nil then
        amount = self.deltamodifierfn(self.inst, amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb)
    end

    if self.maxdamagetakenperhit ~= nil and amount < self.maxdamagetakenperhit and not self._ignore_maxdamagetakenperhit then
        amount = self.maxdamagetakenperhit
    end

    self:SetVal(self.currenthealth + amount, cause, afflicter)

    self.inst:PushEvent("healthdelta", { oldpercent = old_percent, newpercent = self:GetPercent(), overtime = overtime, cause = cause, afflicter = afflicter, amount = amount })
    -- ...
    return amount
end
```

**参数速查**：

| 参数 | 类型 | 含义 |
|---|---|---|
| `amount` | number | 血量变化量：**负数=扣血，正数=加血** |
| `overtime` | bool/nil | `true`=慢慢回血动画（渐变，不是立即），`false`/nil=立即跳变 |
| `cause` | string/nil | 伤害原因（`"cold"`,`"hot"`,`"hunger"`,`"fire"`,`"regen"` 等）|
| `ignore_invincible` | bool/nil | `true`=跳过无敌检查（ForceKill 用此参数）|
| `afflicter` | entity/nil | 造成伤害的 entity（玩家/怪物）|
| `ignore_absorb` | bool/nil | `true`=跳过吸收系数计算 |

**返回值**：实际生效的 amount（经过所有修饰后）

**最简用法**：

```lua
-- 直接扣 20 点血（不触发无敌检查、不计算吸收）
inst.components.health:DoDelta(-20)

-- 治疗 10 点血（overtime=true 触发 UI 渐变动画）
inst.components.health:DoDelta(10, true, "heal_item")

-- 带造成者的伤害
inst.components.health:DoDelta(-35, false, "weapon", false, attacker)
```

#### SetPercent / SetVal —— 直接设置血量百分比/绝对值

```550:553:scripts/components/health.lua
function Health:SetPercent(percent, overtime, cause)
    self:SetVal(self.maxhealth * percent, cause)
    self:DoDelta(0, overtime, cause, true, nil, true)
end
```

```565:611:scripts/components/health.lua
function Health:SetVal(val, cause, afflicter)
    local old_health = self.currenthealth
    local max_health = self:GetMaxWithPenalty()
    local min_health = math.min(self.minhealth or 0, max_health)

    self.inst:PushEvent("pre_health_setval", {val=val, old_health=old_health})

    if val > max_health then
        val = max_health
    end
    -- ... 死亡检查，推送事件
end
```

| 函数 | 参数 | 作用 |
|---|---|---|
| `SetPercent(pct, overtime, cause)` | pct: 0~1 | 设为最大血量的 pct 倍（例 0.5 = 半血）|
| `SetVal(val, cause, afflicter)` | val: 绝对值 | 设为指定血量，但会被 GetMaxWithPenalty 上限截断 |

> **SetPercent vs SetMaxHealth**：`SetMaxHealth` 用于初始化（改上限并满血）；`SetPercent` / `DoDelta` 用于运行时血量变动。

---

### 18.1.3（新手）healthdelta 事件 —— 监听血量变化

每次 `DoDelta` 执行完毕，会推送 `"healthdelta"` 事件：

```642:642:scripts/components/health.lua
    self.inst:PushEvent("healthdelta", { oldpercent = old_percent, newpercent = self:GetPercent(), overtime = overtime, cause = cause, afflicter = afflicter, amount = amount })
```

| 键名 | 类型 | 含义 |
|---|---|---|
| `oldpercent` | number [0~1] | 变化前的血量百分比（相对 maxhealth）|
| `newpercent` | number [0~1] | 变化后的血量百分比 |
| `overtime` | bool | 是否渐变 |
| `cause` | string | 伤害/回血原因 |
| `afflicter` | entity/nil | 造成者 |
| `amount` | number | 实际生效的变化量 |

**在 prefab 中监听血量变化**：

```lua
inst:ListenForEvent("healthdelta", function(inst, data)
    local newpct = data.newpercent
    if newpct < 0.3 then
        -- 血量低于 30%，触发某效果
        inst.components.talker:Say("I'm running out of health!")
    end
end)
```

**关联的其他事件**：

| 事件名 | 触发时机 | 数据 |
|---|---|---|
| `"death"` | currenthealth ≤ 0 | `{cause, afflicter, corpsing}` |
| `"minhealth"` | currenthealth ≤ minhealth | `{cause, afflicter}` |
| `"pre_health_setval"` | SetVal 调用前 | `{val, old_health}` |
| `"invincibletoggle"` | SetInvincible 调用时 | `{invincible: bool}` |

---

### 18.1.4（进阶）DoDelta 完整伤害计算链 —— 7 步结算流程

#### 完整流程图

```
DoDelta(amount=-15, overtime=false, cause="weapon", ignore_invincible=false, afflicter=spider, ignore_absorb=false)
    │
    Step 1: nonlethal 检查（血量 ≤ 20% 且是 cold/hot/hunger 伤害？）
    │       → 满足条件：直接 return 0，本次伤害完全不生效
    │       → 不满足：继续
    │
    Step 2: redirect 检查
    │       → health.redirect 函数存在 AND 返回 true：return 0
    │       → 不存在或返回 false：继续
    │
    Step 3: 无敌检查 (ignore_invincible = false)
    │       → IsInvincible() or inst.is_teleporting：return 0
    │       → 否则：继续
    │
    Step 4: 吸收计算 (amount < 0 AND ignore_absorb = false)
    │       旧式字段（已废弃但兼容）:
    │         factor = 1 - (absorb + playerabsorb) [若 afflicter 是玩家才加 playerabsorb]
    │         amount = amount * clamp(factor, 0, 1)
    │       新式字段（当前推荐）:
    │         amount = amount * max(1 - externalabsorbmodifiers:Get(), 0)
    │       减免（reduction）:
    │         amount = min(0, amount + externalreductionmodifiers:Get())
    │
    Step 5: deltamodifierfn 修饰（若设置）
    │       amount = health.deltamodifierfn(inst, amount, ...)
    │
    Step 6: maxdamagetakenperhit 上限（若设置）
    │       amount = max(amount, maxdamagetakenperhit) [maxdamagetakenperhit 是负数]
    │
    Step 7: 写入血量
            SetVal(currenthealth + amount, cause, afflicter)
            → PushEvent("healthdelta", ...)
            → 若 currenthealth ≤ 0：PushEvent("death", ...)
```

#### 吸收系数详解

| 字段 | 类型 | 废弃？ | 说明 |
|---|---|---|---|
| `health.absorb` | number | 已废弃 | 旧式全局吸收系数（建议不用）|
| `health.playerabsorb` | number | 已废弃 | 旧式玩家造成伤害时的额外吸收 |
| `health.externalabsorbmodifiers` | SourceModifierList | **推荐** | 新式叠加吸收（多来源可叠加）|
| `health.externalreductionmodifiers` | SourceModifierList | **推荐** | 固定减免（如护盾减 5 点）|

**SourceModifierList 的 SetModifier / RemoveModifier**：

```lua
-- 装备某护甲时：增加 20% 吸收
inst.components.health.externalabsorbmodifiers:SetModifier(armor_inst, 0.20, "my_armor")

-- 卸下护甲时：移除
inst.components.health.externalabsorbmodifiers:RemoveModifier(armor_inst, "my_armor")
```

> `externalabsorbmodifiers` 使用 **additive 模式**，多个来源的值直接相加。0.2 + 0.15 = 0.35（即 35% 吸收）。

#### deltamodifierfn —— 全局伤害修饰钩子

```lua
-- 设置：让所有伤害减半
inst.components.health.deltamodifierfn = function(inst, amount, ...)
    if amount < 0 then
        return amount * 0.5
    end
    return amount
end
```

#### maxdamagetakenperhit —— 单次受伤上限

```513:518:scripts/components/health.lua
function Health:SetMaxDamageTakenPerHit(maxdamagetakenperhit)
    if maxdamagetakenperhit ~= nil and maxdamagetakenperhit > 0 then
        maxdamagetakenperhit = -maxdamagetakenperhit
    end
    self.maxdamagetakenperhit = maxdamagetakenperhit
end
```

传入**正数**（如 `50`），内部转为负数（`-50`）——意味着"每次最多受 50 点伤害"，即单次防暴机制。

```lua
-- 每次攻击最多只掉 50 血（类似分层血量怪物如 Gel Blob）
inst.components.health:SetMaxDamageTakenPerHit(50)
-- 取消限制
inst.components.health:SetMaxDamageTakenPerHit(nil)
```

---

### 18.1.5（进阶）无敌机制 —— SetInvincible / temp_invincible / ForceKill

#### 两种无敌方式

| 方式 | 设置方法 | 特点 |
|---|---|---|
| **持久无敌** | `health:SetInvincible(true)` | 写入 `health.invincible = true`，同步到网络 |
| **状态临时无敌** | SG 状态带 `"temp_invincible"` 标签 | 仅在该 SG 状态期间无敌，不同步网络 |

```145:148:scripts/components/health.lua
function Health:SetInvincible(val)
    self.invincible = val
    self.inst:PushEvent("invincibletoggle", { invincible = val })
end
```

```485:489:scripts/components/health.lua
function Health:IsInvincible()
	--V2C: don't use "temp_invincible" for players, since they also need to use
	--     "invincibletoggle" event, which doesn't work with "temp_invincible".
    return self.invincible or (self.inst.sg and self.inst.sg:HasStateTag("temp_invincible"))
end
```

**注意**：对玩家不要用 `"temp_invincible"` SG 标签——玩家需要配合 `invincibletoggle` 事件更新 HUD 显示，而该事件只在 `SetInvincible` 中推送。

#### Kill vs ForceKill

```528:544:scripts/components/health.lua
function Health:Kill()
    if self.currenthealth > 0 then
		self._ignore_maxdamagetakenperhit = true
        self:DoDelta(-self.currenthealth, nil, nil, nil, nil, true)
		self._ignore_maxdamagetakenperhit = nil
    end
end

function Health:ForceKill() -- To bypass invincible
    if self.currenthealth > 0 then
		self._ignore_maxdamagetakenperhit = true
        self:DoDelta(-self.currenthealth, nil, nil, true, nil, true)
		self._ignore_maxdamagetakenperhit = nil
    end
end
```

| 函数 | `ignore_invincible` | 用途 |
|---|---|---|
| `health:Kill()` | **false**（尊重无敌）| 普通死亡（会被无敌拦截）|
| `health:ForceKill()` | **true**（强制）| 强制死亡，无论是否无敌 |

#### 玩家无敌的应用场景（player_common.lua）

```1108:1125:scripts/prefabs/player_common.lua
        inst.components.health:SetInvincible(true)
```

（传送时临时设无敌，传送完成后恢复）

---

### 18.1.6（进阶）penalty 惩罚系统 —— 死亡上限惩罚

#### penalty 是什么

`penalty` 是玩家每次死亡/复活后积累的**血量上限永久降低**机制，影响 HUD 显示的最大血量格子数。

```460:465:scripts/components/health.lua
function Health:SetPenalty(penalty)
    --print("Health:SetPenalty", self.disable_penalty)
	if not self.disable_penalty then
		--Penalty should never be less than 0% or ever above 75%.
		self.penalty = math.clamp(penalty, 0, TUNING.MAXIMUM_HEALTH_PENALTY)
	end
end
```

| 函数 | 参数 | 说明 |
|---|---|---|
| `SetPenalty(penalty)` | 0~0.75 的百分比 | 直接设置惩罚值 |
| `DeltaPenalty(delta)` | 任意数 | 在当前基础上增加/减少惩罚 |
| `GetPenaltyPercent()` | 无 | 返回当前惩罚百分比 |
| `GetMaxWithPenalty()` | 无 | 返回 `maxhealth * (1 - penalty)` |

**各复活方式的惩罚值**（TUNING）：

```2150:2162:scripts/tuning.lua
        PORTAL_HEALTH_PENALTY = 0.25,
        HEART_HEALTH_PENALTY = 0.125,

        MAXIMUM_HEALTH_PENALTY = 0.75,
        MAXIMUM_SANITY_PENALTY = 0.9,

        EFFIGY_HEALTH_PENALTY = 40,
        REVIVE_HEALTH_PENALTY_AS_MULTIPLE_OF_EFFIGY = 1,

        REVIVE_SHADOW_SANITY_PENALTY = -40,
        REVIVE_OTHER_SANITY_BONUS = 80,
        REVIVE_HEALTH_PENALTY = 0.25,
```

| 复活方式 | 惩罚 | 说明 |
|---|---|---|
| 传送门复活 | 25% | `PORTAL_HEALTH_PENALTY = 0.25` |
| 救友复活 | 25% | `REVIVE_HEALTH_PENALTY = 0.25` |
| 心脏复活 | 12.5% | `HEART_HEALTH_PENALTY = 0.125` |
| 肉体稻草人 | 减少惩罚（绝对值）| `EFFIGY_HEALTH_PENALTY = 40` |

#### 禁用 penalty（如 Wanda）

Wanda 使用单独的"老化系统"来替代 penalty，因此禁用了默认 penalty：

```414:414:scripts/prefabs/wanda.lua
	inst.components.health.disable_penalty = true
```

mod 角色如果不希望有血量惩罚，可以在 `master_postinit` 中：

```lua
inst.components.health.disable_penalty = true
```

---

### 18.1.7（进阶）火焰伤害专项 —— fire_damage_scale + DoFireDamage

#### fire_damage_scale —— 火焰伤害倍率

```80:80:scripts/components/health.lua
    self.fire_damage_scale = 1
```

`fire_damage_scale` 是乘以火焰伤害的倍率：
- `1.0`：正常火焰伤害（默认）
- `0.0`：完全免疫火焰（如火焰蜥蜴、部分建筑）
- `>1.0`：对火焰更脆弱（Wormwood = 1.25）

```790:790:scripts/prefabs/wormwood.lua
    inst.components.health.fire_damage_scale = TUNING.WORMWOOD_FIRE_DAMAGE
```

```4537:4537:scripts/tuning.lua
        WORMWOOD_FIRE_DAMAGE = 1.25,
```

**设置 entity 免疫火焰**：

```lua
inst.components.health.fire_damage_scale = 0
```

#### DoFireDamage —— 火焰持续伤害处理

```194:238:scripts/components/health.lua
function Health:DoFireDamage(amount, doer, instant)
    --V2C: "not instant" generally means that we are burning or being set on fire at the same time
    local mult = self:GetFireDamageScale()
    if not self:IsInvincible() and (not instant or mult > 0) then
        -- ... 设置 takingfiredamage 状态
        -- ... 每秒最多 MAX_FIRE_DAMAGE_PER_SECOND 点伤害
        self:DoDelta(-amount * mult, false, doer ~= nil and (doer.nameoverride or doer.prefab) or "fire", nil, doer)
        self.inst:PushEvent("firedamage")
    end
end
```

**每秒伤害上限**：

```93:93:scripts/tuning.lua
        MAX_FIRE_DAMAGE_PER_SECOND = 120,
```

即使着大火，每秒最多扣 120 血——防止瞬间秒杀。

#### 火焰状态相关事件

| 事件 | 触发时机 | 数据 |
|---|---|---|
| `"startfiredamage"` | 开始受火焰伤害 | `{low: bool}` 是否低强度火 |
| `"stopfiredamage"` | 火焰伤害停止（0.5s 超时）| 无 |
| `"changefiredamage"` | 火焰强度变化（低→高 或 高→低）| `{low: bool}` |
| `"firedamage"` | 每次火焰扣血 | 无 |

---

### 18.1.8（老手）redirect 钩子 —— 拦截所有伤害

#### redirect 是什么

`health.redirect` 是一个**可选函数字段**，在 `DoDelta` 执行吸收计算之前调用。如果它返回 `true`，则本次伤害/加血**完全被拦截**，DoDelta 返回 0。

```623:626:scripts/components/health.lua
    if self.redirect ~= nil and self.redirect(self.inst, amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb) then
        return 0
    elseif not ignore_invincible and (self:IsInvincible() or self.inst.is_teleporting) then
        return 0
```

**函数签名**：

```lua
health.redirect = function(inst, amount, overtime, cause, ignore_invincible, afflicter, ignore_absorb)
    -- ...
    return true  -- 拦截本次伤害（DoDelta 返回 0）
    return false -- 不拦截，正常结算
end
```

#### 典型案例 1：Wendy 的伤害转移给 Abigail

```lua
-- scripts/prefabs/wendy.lua（简化）
local function redirect_to_abigail(inst, amount, overtime, cause, ...)
    if amount < 0 and inst.components.follower == nil then
        -- 有 Abigail 时，将伤害转给 Abigail
        local abigail = -- 获取 Abigail 实例
        if abigail ~= nil then
            abigail.components.health:DoDelta(amount * redirect_ratio, ...)
            return true  -- 拦截 Wendy 的伤害
        end
    end
    return false
end

inst.components.health.redirect = redirect_to_abigail
```

#### 典型案例 2：Wanda 的伤害转移给时钟仪

Wanda 的血量是她的年龄（时钟仪的使用次数），真正的 HP 来自老化计算：

```412:413:scripts/prefabs/wanda.lua
    inst.components.health.redirect = redirect_to_oldager
	inst.components.health.canheal = false
```

Wanda 用 `canheal = false` 阻止普通加血（年龄不能"治疗"），所有血量变化都通过 redirect 函数重新路由到她的年龄系统。

#### 在 mod 中使用 redirect

```lua
-- 给 entity 增加"护盾"效果：受到伤害时先吸收护盾值
inst.components.health.redirect = function(target, amount, overtime, cause, ignore_inv, afflicter, ignore_abs)
    if amount < 0 and target.shield_hp ~= nil and target.shield_hp > 0 then
        local absorbed = math.min(target.shield_hp, -amount)
        target.shield_hp = target.shield_hp - absorbed
        local remaining = amount + absorbed  -- 剩余未被护盾吸收的伤害
        if remaining < 0 then
            -- 还有剩余伤害，不拦截，让 DoDelta 继续结算
            -- 但我们需要修改 amount... redirect 无法做到
            -- 建议：用 deltamodifierfn 而不是 redirect
        end
        return target.shield_hp > 0  -- 护盾没空=拦截；护盾耗尽=让伤害穿透
    end
    return false
end
```

> **redirect 的限制**：redirect 只能返回 true/false（完全拦截 or 完全不拦截），**不能修改 amount**。如果需要修改伤害值，应使用 `deltamodifierfn`（在 redirect 之后执行）。

---

### 18.1.9（老手）回血系统 —— StartRegen / AddRegenSource

#### 两种回血方式

| 方式 | 函数 | 适用场景 |
|---|---|---|
| **简单定时回血** | `StartRegen(amount, period)` | 单一回血任务，新的 StartRegen 会覆盖旧的 |
| **多源管理回血** | `AddRegenSource(source, amount, period, key)` | 多种装备/buff 叠加回血，source 自动管理生命周期 |

#### StartRegen —— 简单定时回血

```328:345:scripts/components/health.lua
function Health:StartRegen(amount, period, interruptcurrentregen)
    if interruptcurrentregen ~= false then
        self:StopRegen()
    end

    if self.regen == nil then
        self.regen = {}
    end
    self.regen.amount = amount
    self.regen.period = period

    if self.regen.task == nil then
        self.regen.task = self.inst:DoPeriodicTask(self.regen.period, DoRegen, nil, self)
    end
end
```

**参数**：
- `amount`：每次回复量（正数 = 回血；负数 = 持续掉血）
- `period`：间隔秒数
- `interruptcurrentregen`：是否中断现有任务（默认 `true`）

```lua
-- 每 2 秒回 2 血
inst.components.health:StartRegen(2, 2)

-- 停止回血
inst.components.health:StopRegen()
```

#### AddRegenSource —— 多源管理回血（推荐）

```353:404:scripts/components/health.lua
function Health:AddRegenSource(source, amount, period, key)
    key = key or "key"
    -- ... 以 source+key 为标识建立 DoPeriodicTask
    -- ... 若 source 是 entity，监听 onremove 自动清除
end
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `source` | entity 或任意 key | 回血的来源（通常是提供加成的物品/状态）|
| `amount` | number | 每次回复量 |
| `period` | number | 间隔秒数 |
| `key` | string/nil | 同一 source 下的区分标识（可选）|

```lua
-- 装备某物品，该物品在时提供 1 血/秒回血
local function OnEquip(inst, data)
    local item = data.item
    inst.components.health:AddRegenSource(item, 1, 1, "equip_regen")
end

local function OnUnequip(inst, data)
    local item = data.item
    inst.components.health:RemoveRegenSource(item, "equip_regen")
end

inst:ListenForEvent("equip", OnEquip)
inst:ListenForEvent("unequip", OnUnequip)
```

> **优势**：若 `source` 是一个 entity，当该 entity 被移除时，回血任务会**自动清除**（通过 `"onremove"` 事件监听）——无需手动调用 `RemoveRegenSource`。

---

### 18.1.10（老手）联机同步机制 —— health_replica 与 classified

#### 架构：服务端 vs 客户端

```
┌─────────────────────────────────────────────────────────────────────┐
│  服务端（ismastersim = true）                                         │
│   entity.components.health        ← 完整 Health 组件（主体）          │
│     .currenthealth                 ← 真实血量值                       │
│     .maxhealth                     ← 真实上限                        │
│     .penalty                       ← 真实惩罚值                       │
│     .invincible                    ← 真实无敌状态                     │
│       │                                                              │
│       │ onmaxhealth / oncurrenthealth / onpenalty / oninvincible     │
│       │（SourceField listeners）                                      │
│       ▼                                                              │
│   entity.replica.health            ← health_replica（桥梁）           │
│     → SetMax / SetCurrent / SetPenalty                               │
│       │                                                              │
│       │ 写入 player_classified 的 net_* 变量                          │
│       ▼                                                              │
│   player_classified                                                  │
│     .maxhealth      (net_ushortint)                                  │
│     .currenthealth  (net_ushortint)                                  │
│     .healthpenalty  (net_byte)                                       │
└─────────────────────────────────────────────────────────────────────┘
                         │ 网络同步 │
┌─────────────────────────────────────────────────────────────────────┐
│  客户端（ismastersim = false）                                        │
│   entity.components.health        ← nil（客户端没有完整 health 组件）  │
│   entity.replica.health            ← health_replica（只读）           │
│     .classified = player_classified（对应的 classified entity）       │
│   entity.player_classified                                           │
│     .maxhealth:value()     ← 服务端同步来的最大血量                   │
│     .currenthealth:value() ← 服务端同步来的当前血量                   │
│     .healthpenalty:value() ← 服务端同步来的惩罚（/200 → 百分比）       │
└─────────────────────────────────────────────────────────────────────┘
```

#### health_replica 的 API（服务端 + 客户端通用）

```70:128:scripts/components/health_replica.lua
function Health:Max()
    -- 服务端：直接返回 health.maxhealth
    -- 客户端：从 classified 读 maxhealth:value()
end

function Health:MaxWithPenalty()
    -- 服务端：health:GetMaxWithPenalty()
    -- 客户端：classified.maxhealth * (1 - penalty/200)
end

function Health:GetPercent()
    -- 服务端：health:GetPercent()
    -- 客户端：currenthealth:value() / maxhealth:value()
end

function Health:IsDead()
    return self.inst:HasTag("isdead")  -- 两端通用，靠 tag 同步
end
```

#### 非玩家 entity（怪物、物品）的血量读取

非玩家 entity 没有 `player_classified`，但服务端有 `components.health`。
客户端可以从 `entityreplica` 读取（如果 entity 添加了对应的 replica 字段），否则客户端无法读取非玩家 entity 的血量。

```lua
-- 服务端（安全）
local hp = inst.components.health.currenthealth
local maxhp = inst.components.health.maxhealth

-- 客户端（玩家）
local hp = ThePlayer.replica.health:GetCurrent()   -- health_replica.GetCurrent()
local pct = ThePlayer.replica.health:GetPercent()
```

---

### 18.1.11（小结）速查表 + mod 实战 4 模板

#### Health 组件 API 速查

| 函数/字段 | 参数 | 说明 |
|---|---|---|
| `SetMaxHealth(n)` | number | 设置上限并满血 |
| `DoDelta(amount, overtime, cause, ignore_inv, afflicter, ignore_abs)` | — | 伤害/加血核心 |
| `SetPercent(pct, overtime, cause)` | 0~1 | 设为 maxhealth 的 pct 倍 |
| `SetVal(val, cause, afflicter)` | — | 设为绝对值（受 maxWithPenalty 限制）|
| `Kill()` | 无 | 扣光血（尊重无敌）|
| `ForceKill()` | 无 | 强制扣光血（忽略无敌）|
| `IsDead()` | 无 | currenthealth ≤ 0 |
| `IsHurt()` | 无 | currenthealth < GetMaxWithPenalty() |
| `GetPercent()` | 无 | currenthealth / maxhealth [0~1] |
| `GetMaxWithPenalty()` | 无 | maxhealth * (1 - penalty) |
| `SetInvincible(bool)` | bool | 设置持久无敌 |
| `IsInvincible()` | 无 | invincible OR sg.temp_invincible |
| `SetPenalty(pct)` | 0~0.75 | 设置惩罚 |
| `DeltaPenalty(delta)` | number | 增加/减少惩罚 |
| `StartRegen(amount, period)` | — | 简单定时回血 |
| `StopRegen()` | 无 | 停止简单回血 |
| `AddRegenSource(src, amount, period, key)` | — | 多源回血（推荐）|
| `RemoveRegenSource(src, key)` | — | 移除某来源回血 |
| `SetMaxDamageTakenPerHit(n)` | 正数 | 单次受伤上限（防暴）|

#### 关键字段速查

| 字段 | 类型 | 说明 |
|---|---|---|
| `health.fire_damage_scale` | number | 火焰伤害倍率（0=免疫，1.25=Wormwood）|
| `health.nofadeout` | bool | `true` = 死亡后不自动消失（玩家默认 true）|
| `health.canmurder` | bool | `false` = 不能被杀死（设为 `false` 后 SetCanMurder(false)）|
| `health.canheal` | bool | `false` = DoDelta 正值被忽略（不能被治疗）|
| `health.redirect` | function | 拦截所有伤害的钩子 |
| `health.deltamodifierfn` | function | 修改伤害值的钩子 |
| `health.disable_penalty` | bool | `true` = 禁用惩罚系统（Wanda）|
| `health.save_maxhealth` | bool | `true` = maxhealth 会被存档（Abigail）|

#### 4 个 mod 常用模板

##### 模板 A：设置基础血量（角色 prefab 标准写法）

```lua
local function master_postinit(inst)
    inst.components.health:SetMaxHealth(200)              -- 200 血
    inst.components.health.fire_damage_scale = 0.5        -- 火焰伤害减半
    inst.components.health.nofadeout = true               -- 死亡不自动消失（玩家必须设）
    inst.components.health.disable_penalty = false        -- 启用惩罚（默认）
end
```

##### 模板 B：监听血量变化触发特效

```lua
inst:ListenForEvent("healthdelta", function(inst, data)
    if data.newpercent < 0.3 and data.oldpercent >= 0.3 then
        -- 首次低于 30% 血量时触发
        inst.SoundEmitter:PlaySound("mymod/characters/mychar/low_health_warning")
    end
end)

inst:ListenForEvent("death", function(inst, data)
    -- 死亡时触发特效
    SpawnPrefab("mymod_death_fx").Transform:SetPosition(inst.Transform:GetWorldPosition())
end)
```

##### 模板 C：护盾效果（使用 externalabsorbmodifiers）

```lua
-- 激活护盾：减少 30% 受到的伤害
local function ActivateShield(inst, shield_item)
    inst.components.health.externalabsorbmodifiers:SetModifier(shield_item, 0.30, "shield")
end

-- 护盾破碎/卸下
local function DeactivateShield(inst, shield_item)
    inst.components.health.externalabsorbmodifiers:RemoveModifier(shield_item, "shield")
end
```

##### 模板 D：受伤时转移伤害（redirect）

```lua
-- 有护盾实体时，将伤害重定向给护盾实体
local function SetupDamageRedirect(inst)
    inst.components.health.redirect = function(target, amount, overtime, cause, ...)
        if amount < 0 and target.shield_entity ~= nil and target.shield_entity:IsValid() then
            target.shield_entity.components.health:DoDelta(amount, overtime, cause, ...)
            return true  -- 拦截原来的伤害
        end
        return false
    end
end
```

---


## 18.2 Hunger 组件——饥饿与新陈代谢

### 本节导读

> **一句话定位**：`Hunger` 是饥荒中角色**胃容量与新陈代谢**的管理器——它以固定间隔（每秒）消耗饱食度，耗尽后开始扣血，并在状态变化时广播 `"hungerdelta"`/`"startstarving"`/`"stopstarving"` 事件——相比 Health，Hunger 更简单但也有若干关键细节值得精通。

#### 一段宏观先讲清楚：饱食度是如何流动的

```
【游戏运行：每 1 秒触发一次 OnTaskTick】
            │
            ▼
    Hunger:DoDec(1)   ← dt = 1 (秒)
            │
    ┌───────┴────────────────────────────────────────┐
    │  current > 0（没饿死）                          │  current <= 0（已饥饿）
    │                                                 │
    ▼                                                 ▼
    DoDelta(                                overridestarvefn(inst, dt)
        -hungerrate * dt * burnrate * burnratemodifiers,     或
        overtime=true                    health:DoDelta(-hurtrate * dt, true, "hunger")
    )                                                 │
    │                                                 │
    ▼                                                 ▼
    SetCurrent(current + delta)          玩家每秒因饥饿失血
    │                                    （STARVE_KILL_TIME=120 秒内死亡）
    ▼
    PushEvent("hungerdelta", {...})
    若 current 跨越 0 边界：
      PushEvent("startstarving") 或 PushEvent("stopstarving")
```

**核心公式**：

```
每秒消耗饱食度 = hungerrate × dt × burnrate × burnratemodifiers.Get()
               = WILSON_HUNGER_RATE × 1 × 1 × 1
               = (75/480) × 1 × 1 × 1
               ≈ 0.156 / 秒
               → 150 点饱食度 ÷ 0.156 ≈ 960 秒 ≈ 16 分钟耗尽（一个游戏日）
```

（`total_day_time = seg_time * 16 = 30 × 16 = 480 秒`，即一游戏日 8 分钟现实时间）

#### 18.2 节回答的 5 个核心问题

```
Q1: ──── 如何设置角色的胃容量（最大饱食度）？
         ↓ 答：hunger:SetMax(n)（同时将 current 设为 n）

Q2: ──── 如何控制饥饿消耗速率？
         ↓ 答：hunger:SetRate(rate) 设基础速率；burnratemodifiers 叠加倍率

Q3: ──── 饥饿后如何扣血？扣多少？
         ↓ 答：DoDec → health:DoDelta(-hurtrate * dt, true, "hunger")；hurtrate = health/STARVE_KILL_TIME

Q4: ──── 如何暂停/恢复饥饿消耗？
         ↓ 答：hunger:Pause() / hunger:Resume()（暂停/恢复 DoPeriodicTask）

Q5: ──── 如何监听玩家开始/停止挨饿？
         ↓ 答：ListenForEvent("startstarving") / ListenForEvent("stopstarving")
```

#### 你将看到的核心源码

| 文件 | 行 | 用途 |
|---|---|---|
| `scripts/components/hunger.lua` | 全文 211 行 | Hunger 组件主体 |
| `scripts/components/hunger_replica.lua` | 全文 92 行 | 客户端饥饿只读接口 |
| `scripts/prefabs/player_common.lua` | 2751-2757 | 玩家 hunger 初始化 |
| `scripts/tuning.lua` | 41/120/752-754 | 关键常量：calories_per_day/HUNGER_RATE/STARVE_KILL_TIME |
| `scripts/prefabs/shadow_battleaxe.lua` | 372-414 | overridestarvefn + Pause 实例 |

#### 本节学习路径

```
18.2.1 新手 ─── Hunger 三大核心字段 —— max / current / hungerrate
18.2.2 新手 ─── SetMax / DoDelta / SetCurrent —— 常用 API
18.2.3 新手 ─── hungerdelta / startstarving / stopstarving 事件
                  ↓
18.2.4 进阶 ─── 饱食度消耗机制 —— DoDec 的完整逻辑
18.2.5 进阶 ─── hurtrate —— 挨饿扣血速率
18.2.6 进阶 ─── burnratemodifiers —— 多源消耗速率叠加
                  ↓
18.2.7 老手 ─── SetOverrideStarveFn —— 自定义挨饿行为
18.2.8 老手 ─── Pause / Resume —— 暂停饥饿消耗
18.2.9 老手 ─── 联机同步机制 —— hunger_replica
18.2.10     ─── 小结·速查表 + mod 实战 3 模板
```

---

### 18.2.1（新手）Hunger 三大核心字段

```24:38:scripts/components/hunger.lua
local Hunger = Class(function(self, inst)
    self.inst = inst
    self.max = 100
    self.current = self.max

    self.hungerrate = 1
    self.hurtrate = 1
    self.overridestarvefn = nil

    self.burning = true

    self.burnrate = 1 -- DEPRECATED, please use burnratemodifiers instead.
    self.burnratemodifiers = SourceModifierList(self.inst)

    self.updatetask = self.inst:DoPeriodicTask(UPDATE_PERIOD, OnTaskTick, nil, self)
```

| 字段 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `max` | number | 100 | 最大饱食度（胃容量）|
| `current` | number | = max | 当前饱食度 |
| `hungerrate` | number | 1 | 基础饥饿速率（单位：饱食度/秒）|
| `hurtrate` | number | 1 | 挨饿每秒扣血量 |
| `burning` | bool | true | `true` = 正在消耗饱食度（`false` = 已暂停）|
| `burnrate` | number | 1 | **已废弃**，用 `burnratemodifiers` 代替 |
| `burnratemodifiers` | SourceModifierList | 乘积 = 1 | 多来源饥饿速率乘数 |
| `overridestarvefn` | function/nil | nil | 自定义挨饿行为（nil = 默认扣血）|

#### 玩家的默认值计算

```2751:2754:scripts/prefabs/player_common.lua
        inst:AddComponent("hunger")
        inst.components.hunger:SetMax(TUNING.WILSON_HUNGER)
        inst.components.hunger:SetRate(TUNING.WILSON_HUNGER_RATE)
        inst.components.hunger:SetKillRate(TUNING.WILSON_HEALTH / TUNING.STARVE_KILL_TIME)
```

```
WILSON_HUNGER = 150               ← 胃容量
WILSON_HUNGER_RATE = 75/480       ← ≈ 0.15625 点/秒（一天耗完 150 格）
STARVE_KILL_TIME = 120            ← 挨饿 120 秒死亡
hurtrate = 150/120 = 1.25         ← 挨饿每秒扣 1.25 血
```

---

### 18.2.2（新手）SetMax / DoDelta / SetCurrent —— 常用 API

#### SetMax —— 设置胃容量（同时满胃）

```48:51:scripts/components/hunger.lua
function Hunger:SetMax(amount)
    self.max = amount
    self.current = amount
end
```

> **注意**：`SetMax` 会**同时把 current 设为 amount（满胃）**——与 `health:SetMaxHealth` 行为一致。

#### DoDelta —— 增加/消耗饱食度

```128:144:scripts/components/hunger.lua
function Hunger:DoDelta(delta, overtime, ignore_invincible)
    if self.redirect ~= nil then
        self.redirect(self.inst, delta, overtime)

        return
    end

    if not ignore_invincible and
        self.inst.components.health and
        self.inst.components.health:IsInvincible() or
        self.inst.is_teleporting
    then
        return
    end

    self:SetCurrent(self.current + delta, overtime)
end
```

**参数**：

| 参数 | 类型 | 含义 |
|---|---|---|
| `delta` | number | 饱食度变化量（正数=吃东西、负数=消耗）|
| `overtime` | bool/nil | `true` = HUD 缓慢变化动画，`false`/nil = 立即跳变 |
| `ignore_invincible` | bool/nil | `true` = 无论无敌状态都生效 |

**特殊机制**：
1. 若 `self.redirect ~= nil`，调用 redirect 并直接返回（不执行后续逻辑）
2. 若角色处于无敌状态 (`IsInvincible()`) 或传送中 (`is_teleporting`)，DoDelta 无效

**最简用法**：

```lua
-- 吃食物时（eater 组件内部调用）
inst.components.hunger:DoDelta(25, true)     -- 吃了 25 卡食物，overtime=true 触发渐变

-- 某技能消耗饱食度
inst.components.hunger:DoDelta(-10)          -- 立即消耗 10 点
```

#### SetCurrent —— 直接设置当前饱食度

```104:126:scripts/components/hunger.lua
function Hunger:SetCurrent(current, overtime)
    local old = self.current

    self.current = math.clamp(current, 0, self.max)

    self.inst:PushEvent("hungerdelta", {
        oldpercent = old / self.max,
        newpercent = self.current / self.max,
        overtime = overtime,
        delta = self.current-old
    })

    if old > 0 then
        if self.current <= 0 then
            self.inst:PushEvent("startstarving")
            ProfileStatsSet("started_starving", true)
        end

    elseif self.current > 0 then
        self.inst:PushEvent("stopstarving")
        ProfileStatsSet("stopped_starving", true)
    end
end
```

- `current` 会被 `clamp(current, 0, self.max)` 限制在 [0, max] 范围
- 推送 `"hungerdelta"` 事件
- 跨越 0 边界时推送 `"startstarving"` 或 `"stopstarving"`

#### SetPercent

```98:100:scripts/components/hunger.lua
function Hunger:SetPercent(p, overtime)
    self:SetCurrent(p * self.max, overtime)
end
```

**用途**：以百分比形式设置饱食度，例如 `SetPercent(0.5)` 设为半胃。

---

### 18.2.3（新手）hungerdelta / startstarving / stopstarving 事件

#### hungerdelta 事件

每次调用 `SetCurrent` 时推送，无论是吃食物、自然消耗还是代码直接设值。

| 键名 | 类型 | 含义 |
|---|---|---|
| `oldpercent` | number [0~1] | 变化前百分比（相对 max）|
| `newpercent` | number [0~1] | 变化后百分比 |
| `overtime` | bool | 是否渐变动画 |
| `delta` | number | 实际变化量（current新 - current旧）|

```lua
-- 监听饱食度变化（例如：低于 1/3 时触发饥饿提示）
inst:ListenForEvent("hungerdelta", function(inst, data)
    if data.newpercent < 0.333 and data.oldpercent >= 0.333 then
        inst.components.talker:Say(GetString(inst, "ANNOUNCE_HUNGRY"))
    end
end)
```

> `TUNING.HUNGRY_THRESH = 0.333`（饥饿警告阈值）和 `TUNING.GHOST_THRESH = 0.125`（极度饥饿）是 vanilla 中用于触发特殊效果的两个阈值。

#### startstarving / stopstarving 事件

```lua
-- 开始挨饿（current 从 >0 变为 ≤0）
inst:ListenForEvent("startstarving", function(inst)
    inst.components.talker:Say(GetString(inst, "ANNOUNCE_HUNGRY"))
    -- 或触发特殊动画
end)

-- 吃东西后不再挨饿（current 从 0 变为 >0）
inst:ListenForEvent("stopstarving", function(inst)
    -- 停止饥饿特效
end)
```

---

### 18.2.4（进阶）饱食度消耗机制 —— DoDec 的完整逻辑

`DoDec` 是 Hunger 组件的核心内部更新函数，每 1 秒被 `OnTaskTick` 调用一次（`UPDATE_PERIOD = 1`）：

```146:163:scripts/components/hunger.lua
function Hunger:DoDec(dt, ignore_damage)
    if self:IsPaused() then
        return
    end

    local old = self.current

    if self.current > 0 then
        self:DoDelta(-self.hungerrate * dt * self.burnrate * self.burnratemodifiers:Get(), true)

    elseif not ignore_damage then
        if self.overridestarvefn ~= nil then
            self.overridestarvefn(self.inst, dt)
        else
            self.inst.components.health:DoDelta(-self.hurtrate * dt, true, "hunger")
        end
    end
end
```

**逻辑分支**：

```
DoDec(dt=1, ignore_damage=false)
    │
    ├── 已暂停（IsPaused()）→ 直接返回，什么也不做
    │
    ├── current > 0（未挨饿）
    │       → DoDelta(-hungerrate * 1 * burnrate * burnratemodifiers.Get(), overtime=true)
    │       → 消耗饱食度
    │
    └── current <= 0（正在挨饿）AND ignore_damage = false
            ├── overridestarvefn 存在：调用自定义函数
            └── 否则：health:DoDelta(-hurtrate * 1, true, "hunger")
                      → 每秒扣 hurtrate 点血
```

**消耗速率完整公式**：

```
每秒消耗 = hungerrate × dt × burnrate × burnratemodifiers.Get()
         = 基础速率 × 时间 × 旧式乘数(废弃) × 新式乘数
```

> 注意：`burnrate` 字段已废弃（仍保留用于兼容旧 mod），新代码应使用 `burnratemodifiers`。

#### 游戏暂停时的 LongUpdate

当游戏加载存档或快速时间流逝时，调用 `LongUpdate(dt)`：

```167:169:scripts/components/hunger.lua
function Hunger:LongUpdate(dt)
    self:DoDec(dt, true)
end
```

`LongUpdate` 传入 `ignore_damage=true`，即**快速时间流逝只消耗饱食度，不会因饥饿扣血**——这是为了防止离线时间过长导致玩家一上线就死亡。

---

### 18.2.5（进阶）hurtrate —— 挨饿扣血速率

`hurtrate` 控制挨饿时每秒扣多少血。通过 `SetKillRate` 设置：

```57:59:scripts/components/hunger.lua
function Hunger:SetKillRate(rate)
    self.hurtrate = rate
end
```

**玩家的默认 hurtrate 计算**：

```2754:2754:scripts/prefabs/player_common.lua
        inst.components.hunger:SetKillRate(TUNING.WILSON_HEALTH / TUNING.STARVE_KILL_TIME)
```

```
hurtrate = WILSON_HEALTH / STARVE_KILL_TIME = 150 / 120 = 1.25 点/秒
```

即**饿肚子 120 秒（2 分钟现实时间 / 4 分钟游戏内时间）就会死亡**（不考虑治疗）。

> `TUNING.STARVE_KILL_TIME = 120`（秒）

**调整角色耐饿能力**（mod 中）：

```lua
-- 让角色更耐饿：需要 200 秒才被饿死
inst.components.hunger:SetKillRate(inst.components.health.maxhealth / 200)

-- 让角色饿肚子不会死（但会触发 startstarving）
-- 方法1：让 hurtrate = 0（但 overridestarvefn 更规范）
-- 方法2：health.nonlethal_hunger = true（health 组件的特性）
```

---

### 18.2.6（进阶）burnratemodifiers —— 多源消耗速率叠加

`burnratemodifiers` 是一个 **SourceModifierList**（默认乘法模式），用于叠加多个来源的饥饿速率倍数。

**典型应用：奔跑/骑乘/游泳时加快饥饿消耗**

```lua
-- SGwilson.lua：猴王奔跑时消耗加快
inst.components.hunger.burnratemodifiers:SetModifier(inst, TUNING.WONKEY_RUN_HUNGER_RATE_MULT, "wonkey_run")
-- 停止奔跑
inst.components.hunger.burnratemodifiers:RemoveModifier(inst, "wonkey_run")
```

**帽子减缓饥饿（蘑菇帽）**：

```lua
-- 装备蘑菇帽
owner.components.hunger.burnratemodifiers:SetModifier(inst, TUNING.MUSHROOMHAT_SLOW_HUNGER)
-- 卸下蘑菇帽
owner.components.hunger.burnratemodifiers:RemoveModifier(inst)
```

**burnratemodifiers 的默认模式**：

`SourceModifierList` 默认是乘法模式（multiplicative）——每个来源的值相乘。若两个来源分别是 `1.5` 和 `0.8`：
- `burnratemodifiers.Get() = 1.5 × 0.8 = 1.2`
- 最终消耗 = `hungerrate × 1.2`（加快了 20%）

**mod 中使用 burnratemodifiers**：

```lua
-- 激活效果：饥饿消耗加速 50%
inst.components.hunger.burnratemodifiers:SetModifier(effect_source, 1.5, "my_speed_boost")

-- 移除效果
inst.components.hunger.burnratemodifiers:RemoveModifier(effect_source, "my_speed_boost")

-- 设置为减缓消耗（如修炼状态）
inst.components.hunger.burnratemodifiers:SetModifier(effect_source, 0.5, "meditation")
```

---

### 18.2.7（老手）SetOverrideStarveFn —— 自定义挨饿行为

`overridestarvefn` 在 `current <= 0` 且 `DoDec` 执行时被调用，完全替换默认的扣血逻辑。

```61:63:scripts/components/hunger.lua
function Hunger:SetOverrideStarveFn(fn)
    self.overridestarvefn = fn
end
```

**函数签名**：

```lua
hunger:SetOverrideStarveFn(function(inst, dt)
    -- inst: entity
    -- dt: 时间步长（通常 = UPDATE_PERIOD = 1 秒）
    -- 在这里自定义"挨饿时"的逻辑
end)
```

#### 案例：影子战斧的挨饿行为

影子战斧（一个非玩家 entity）也有 Hunger 组件，挨饿时不扣血而是消耗武器耐久度：

```372:378:scripts/prefabs/shadow_battleaxe.lua
local function OnStarving(inst, dt)
    if inst.components.finiteuses ~= nil then
        inst.components.finiteuses:Use(math.min(dt, inst.components.finiteuses:GetUses()))
    end

    inst:SayRegularChatLine("starving", inst._owner)
end
```

设置方式（`shadow_battleaxe.lua` 第 414 行）：

```414:414:scripts/prefabs/shadow_battleaxe.lua
    inst.components.hunger:SetOverrideStarveFn(OnStarving)
```

#### mod 应用：挨饿时触发特殊变身

```lua
local function OnCharacterStarving(inst, dt)
    -- 挨饿时不直接扣血，而是触发变身状态
    if not inst:HasTag("starving_form") then
        inst:AddTag("starving_form")
        -- 触发变身动画/效果
        inst:PushEvent("enter_starving_form")
    end
    -- 仍然缓慢扣血但速率减半
    if inst.components.health ~= nil then
        inst.components.health:DoDelta(-inst.components.hunger.hurtrate * dt * 0.5, true, "hunger")
    end
end

inst.components.hunger:SetOverrideStarveFn(OnCharacterStarving)
```

---

### 18.2.8（老手）Pause / Resume —— 暂停饥饿消耗

#### Pause —— 停止饥饿消耗

```75:82:scripts/components/hunger.lua
function Hunger:Pause()
    self.burning = false

    if self.updatetask ~= nil then
        self.updatetask:Cancel()
        self.updatetask = nil
    end
end
```

Pause 会**取消 `DoPeriodicTask`**，完全停止饱食度消耗。

```84:90:scripts/components/hunger.lua
function Hunger:Resume()
    self.burning = true

    if self.updatetask == nil then
        self.updatetask = self.inst:DoPeriodicTask(UPDATE_PERIOD, OnTaskTick, nil, self)
    end
end
```

Resume 重新创建 `DoPeriodicTask`，恢复饱食度消耗。

**使用场景**：
1. **特殊游戏模式**：`GetGameModeProperty("no_hunger")` 为 true 时（友好模式）
2. **进入特殊状态**：如变身后不消耗饱食度
3. **非玩家 entity 初始化时**：先 `Pause()` 再按需 `Resume()`

```lua
-- 节日模式下关闭饥饿消耗
if GetGameModeProperty("no_hunger") then
    inst.components.hunger:Pause()
end

-- 角色变身时暂停饥饿
local function OnTransform(inst)
    inst.components.hunger:Pause()
end

-- 变回正常后恢复
local function OnRestore(inst)
    inst.components.hunger:Resume()
end
```

#### IsPaused / IsStarving

```65:71:scripts/components/hunger.lua
function Hunger:IsPaused()
    return not self.burning
end

function Hunger:IsStarving()
    return self.current <= 0
end
```

---

### 18.2.9（老手）联机同步机制 —— hunger_replica

#### 架构

```
┌─────────────────────────────────────────────────────────┐
│  服务端                                                   │
│   entity.components.hunger       ← 完整 Hunger 组件       │
│     .current                       ← 真实饱食度           │
│     .max                           ← 真实胃容量           │
│       │                                                   │
│       │ oncurrent / onmax（SourceField listeners）        │
│       ▼                                                   │
│   entity.replica.hunger           ← hunger_replica（桥梁）│
│     → SetCurrent / SetMax                                 │
│       │                                                   │
│       │ 写入 player_classified 的 net 变量               │
│       ▼                                                   │
│   player_classified                                       │
│     .currenthunger  (net 变量)                            │
│     .maxhunger      (net 变量)                            │
└─────────────────────────────────────────────────────────┘
                      │ 网络同步 │
┌─────────────────────────────────────────────────────────┐
│  客户端                                                   │
│   entity.replica.hunger          ← hunger_replica（只读）│
│     .classified = player_classified                       │
│   player_classified.currenthunger:value()                │
│   player_classified.maxhunger:value()                     │
└─────────────────────────────────────────────────────────┘
```

#### hunger_replica API（两端通用）

```53:90:scripts/components/hunger_replica.lua
function Hunger:Max()
    -- 服务端：hunger.max；客户端：classified.maxhunger:value()
end

function Hunger:GetPercent()
    -- 服务端：hunger:GetPercent()；客户端：currenthunger/maxhunger
end

function Hunger:GetCurrent()
    -- 服务端：hunger.current；客户端：classified.currenthunger:value()
end

function Hunger:IsStarving()
    -- 服务端：hunger:IsStarving()；客户端：currenthunger:value() <= 0
end
```

**在代码中读取饱食度（两端安全）**：

```lua
-- 服务端（直接访问组件）
local hunger_pct = inst.components.hunger:GetPercent()
local is_starving = inst.components.hunger:IsStarving()

-- 客户端（通过 replica）
local hunger_pct = ThePlayer.replica.hunger:GetPercent()
local is_starving = ThePlayer.replica.hunger:IsStarving()
```

---

### 18.2.10（小结）速查表 + mod 实战 3 模板

#### Hunger 组件 API 速查

| 函数/字段 | 参数 | 说明 |
|---|---|---|
| `SetMax(n)` | number | 设置胃容量（同时满胃）|
| `SetRate(rate)` | number | 设置基础饥饿速率（卡/秒）|
| `SetKillRate(rate)` | number | 设置挨饿扣血速率（血量/秒）|
| `DoDelta(delta, overtime, ignore_inv)` | — | 增加/消耗饱食度 |
| `SetCurrent(current, overtime)` | — | 直接设置当前饱食度（自动 clamp）|
| `SetPercent(pct, overtime)` | 0~1 | 设为 max 的 pct 倍 |
| `GetPercent()` | 无 | current / max |
| `IsStarving()` | 无 | current <= 0 |
| `IsPaused()` | 无 | !burning |
| `Pause()` | 无 | 停止饱食度消耗 |
| `Resume()` | 无 | 恢复饱食度消耗 |
| `SetOverrideStarveFn(fn)` | function | 自定义挨饿行为 |

#### 关键字段速查

| 字段 | 类型 | 说明 |
|---|---|---|
| `hunger.hungerrate` | number | 基础饥饿速率（`SetRate` 设置）|
| `hunger.hurtrate` | number | 挨饿扣血速率（`SetKillRate` 设置）|
| `hunger.burnrate` | number | 旧式乘数（已废弃，不推荐使用）|
| `hunger.burnratemodifiers` | SourceModifierList | 新式叠加速率乘数 |
| `hunger.overridestarvefn` | function/nil | 替换默认挨饿扣血逻辑 |
| `hunger.redirect` | function/nil | 拦截所有饱食度变化 |

#### 常见 TUNING 值参考

| 常量 | 值 | 含义 |
|---|---|---|
| `WILSON_HUNGER` | 150 | 默认胃容量 |
| `WILSON_HUNGER_RATE` | 75/480 ≈ 0.156 | 默认饥饿速率（卡/秒）|
| `STARVE_KILL_TIME` | 120 | 默认挨饿死亡时间（秒）|
| `HUNGRY_THRESH` | 0.333 | 饥饿 UI 提示阈值 |
| `GHOST_THRESH` | 0.125 | 极度饥饿阈值 |
| `CALORIES_TINY` | 9.375 | 浆果等小型食物热量 |
| `CALORIES_SMALL` | 12.5 | 蔬菜等小型食物热量 |
| `CALORIES_MED` | 25 | 肉类等中型食物热量 |
| `CALORIES_LARGE` | 37.5 | 烤肉等大型食物热量 |
| `CALORIES_HUGE` | 75 | 锅饭等巨量食物热量 |

#### 3 个 mod 常用模板

##### 模板 A：基础角色饥饿初始化

```lua
local function master_postinit(inst)
    inst.components.hunger:SetMax(200)            -- 胃容量 200
    inst.components.hunger:SetRate(TUNING.WILSON_HUNGER_RATE * 0.8)  -- 比 Wilson 慢 20%
    inst.components.hunger:SetKillRate(
        inst.components.health.maxhealth / 150    -- 挨饿 150 秒才死
    )
end
```

##### 模板 B：使用 burnratemodifiers 做技能消耗加速

```lua
-- 激活冲刺技能：饥饿消耗加快 1.5 倍
local function OnSprintStart(inst)
    inst.components.hunger.burnratemodifiers:SetModifier(inst, 1.5, "sprint")
end

-- 技能结束：恢复正常
local function OnSprintEnd(inst)
    inst.components.hunger.burnratemodifiers:RemoveModifier(inst, "sprint")
end

inst:ListenForEvent("sprint_start", OnSprintStart)
inst:ListenForEvent("sprint_end", OnSprintEnd)
```

##### 模板 C：自定义挨饿不扣血（改为触发特殊状态）

```lua
local function CustomStarveFn(inst, dt)
    -- 不扣血，改为降低移速（通过 tag 控制）
    if not inst:HasTag("hungry_debuff") then
        inst:AddTag("hungry_debuff")
        -- 假设有一个移速组件
        if inst.components.locomotor then
            inst.components.locomotor:SetExternalSpeedMultiplier(inst, 0.7, "hungry")
        end
    end
end

local function OnStopStarving(inst)
    inst:RemoveTag("hungry_debuff")
    if inst.components.locomotor then
        inst.components.locomotor:RemoveExternalSpeedMultiplier(inst, "hungry")
    end
end

inst.components.hunger:SetOverrideStarveFn(CustomStarveFn)
inst:ListenForEvent("stopstarving", OnStopStarving)
```

---


## 18.3 Sanity 组件——理智、光环与暗影生物（sanityaura.lua）

### 本节导读

> **一句话定位**：`Sanity` 是饥荒中最复杂的生存状态组件——它同时管理两种不同模式（`疯狂模式` vs `顿悟模式`）、5 类持续变化来源（光照/装备魅力/湿度/光环/幽灵）、强制精神状态（`SetInducedInsanity`/`SetInducedLunacy`）以及影响范围内所有玩家精神的 `SanityAura` 组件——理解这套系统，你才能让 mod 的 boss、装备、地形正确影响玩家理智。

#### 一段宏观先讲清楚：理智值是怎么变化的

```
【每帧 OnUpdate(dt) 执行】
    │
    ├── 若无敌 / 传送中 / 睡眠中 / ignore → 跳过 Recalc
    │
    └── Recalc(dt)
           │
           ├── dapper_delta   ← 装备魅力值（正负均可）
           │                       = Σ(equippable.dapperness) × dapperness_mult
           │                         × TUNING.SANITY_DAPPERNESS
           │
           ├── moisture_delta ← 潮湿度惩罚（总是负数或 0）
           │                       easing.inSine(moisture, 0, MAX_PENALTY, maxMoisture)
           │
           ├── light_delta    ← 光照变化（白天+/夜晚-）
           │                       免疫条件：light_drain_immune_sources
           │
           ├── aura_delta     ← 范围内带 sanityaura 组件的实体光环之和
           │                       搜索范围：SANITY_AURA_SEACH_RANGE = 20
           │                       免疫条件：sanity_aura_immune_sources
           │
           ├── ghost_delta    ← 其他玩家成为幽灵的惩罚
           │                       免疫条件：player_ghost_immune_sources
           │
           ├── externalmodifiers ← 外部添加的加/减项
           │
           └── custom_rate_fn ← 可选：自定义额外速率函数
                     │
                     ▼
            rate = Σ 以上 × rate_modifier
                     │
                     ▼
            DoDelta(rate × dt, overtime=true)
                     │
                     ▼
            current = clamp(current + delta, 0, max×(1-penalty))
                     │
                     ├── 若跨越疯狂/理智/顿悟阈值
                     │       PushEvent("gosane") / PushEvent("goinsane") / PushEvent("goenlightened")
                     └── PushEvent("sanitydelta", {...})
```

#### 18.3 节回答的 6 个核心问题

```
Q1: ──── 两种模式（疯狂/顿悟）是什么？分别对应什么角色/区域？
         ↓ 答：SANITY_MODE_INSANITY（默认）→ 低理智=疯狂；
                SANITY_MODE_LUNACY  → 高理智=顿悟（月光区域）

Q2: ──── 理智每帧是怎么计算的？5 类来源分别是什么？
         ↓ 答：dapper+moisture+light+aura+ghost, Recalc(dt) 每帧汇总

Q3: ──── 如何让 mod 怪物/物品降低附近玩家理智？
         ↓ 答：inst:AddComponent("sanityaura") + sanityaura.aura = -TUNING.SANITYAURA_MED

Q4: ──── gosane/goinsane 的阈值是多少？
         ↓ 答：insane: <30/200=15%；sane: ≥35/200=17.5%（有滞回区间）

Q5: ──── 如何强制让玩家进入疯狂/顿悟状态（无视实际理智值）？
         ↓ 答：SetInducedInsanity(src, true) / SetInducedLunacy(src, true)

Q6: ──── 装备的 dapperness 值是怎么影响理智的？
         ↓ 答：equippable.dapperness × TUNING.SANITY_DAPPERNESS（=1点/秒/单位）
```

#### 你将看到的核心源码

| 文件 | 行 | 用途 |
|---|---|---|
| `scripts/components/sanity.lua` | 全文 594 行 | Sanity 组件主体 |
| `scripts/components/sanityaura.lua` | 全文 30 行 | 理智光环组件 |
| `scripts/components/sanity_replica.lua` | 全文 240 行 | 客户端理智只读接口 |
| `scripts/tuning.lua` | 2163-2215 | 关键阈值、光照速率、光环速率 |
| `scripts/constants.lua` | 1226-1227/2392-2401 | 模式常量、RATE_SCALE |

#### 本节学习路径

```
18.3.1 新手 ─── 两种模式：INSANITY vs LUNACY
18.3.2 新手 ─── DoDelta / gosane / goinsane / goenlightened 事件
18.3.3 新手 ─── sanityaura 组件 —— 让 entity 影响周围玩家理智
                  ↓
18.3.4 进阶 ─── Recalc：5 类理智变化来源详解
18.3.5 进阶 ─── dapperness 系统 —— 装备魅力值
18.3.6 进阶 ─── 各类免疫源（SourceModifierList 布尔模式）
                  ↓
18.3.7 老手 ─── SetInducedInsanity / SetInducedLunacy —— 强制精神状态
18.3.8 老手 ─── externalmodifiers + custom_rate_fn —— 外部速率修改
18.3.9 老手 ─── 联机同步机制 —— sanity_replica
18.3.10     ─── 小结·速查表 + mod 实战 4 模板
```

---

### 18.3.1（新手）两种模式：INSANITY vs LUNACY

#### 两种模式的语义

```1226:1227:scripts/constants.lua
SANITY_MODE_INSANITY = 0
SANITY_MODE_LUNACY = 1
```

| 模式 | 常量值 | 语义 | 低理智效果 | 高理智效果 |
|---|---|---|---|---|
| **INSANITY**（疯狂）| 0 | 默认模式 | 疯狂（`goinsane`）| 正常（`gosane`）|
| **LUNACY**（顿悟）| 1 | 月光/月形怪区域 | 顿悟消失（`gosane`）| 顿悟（`goenlightened`）|

**关键点**：在疯狂模式下，理智值越低越危险；在顿悟模式下，理智值越**高**越特殊（理智不降反升才能"顿悟"）。

#### 模式切换：EnableLunacy

```184:187:scripts/components/sanity.lua
function Sanity:EnableLunacy(enable, source)
	self._lunacy_sources:SetModifier(self.inst, enable, source)
	self:UpdateMode_Internal()
end
```

`_lunacy_sources` 是布尔 SourceModifierList——任意一个来源为 `true`，则 mode = `SANITY_MODE_LUNACY`。

**游戏中的使用场景**：
- 月形怪聚集区域 → `sanity:EnableLunacy(true, "moon_zone")`
- 离开区域 → `sanity:EnableLunacy(false, "moon_zone")`

#### 三种状态判断函数

```138:157:scripts/components/sanity.lua
function Sanity:IsSane()
	if self.mode == SANITY_MODE_INSANITY then
		return self.sane and not self.inducedinsanity or self.inducedlunacy or false
	else--if self.mode == SANITY_MODE_LUNACY then
		return self.sane and not self.inducedlunacy or self.inducedinsanity or false
	end
end

function Sanity:IsInsane()
	return self.mode == SANITY_MODE_INSANITY and (not self.sane or self.inducedinsanity) and not self.inducedlunacy
end

function Sanity:IsEnlightened()
	return self.mode == SANITY_MODE_LUNACY and (not self.sane or self.inducedlunacy) and not self.inducedinsanity
end
```

| 函数 | 返回 true 的条件 |
|---|---|
| `IsSane()` | 疯狂模式下正常；或顿悟模式下顿悟；简言之"看起来不疯不癫" |
| `IsInsane()` | 疯狂模式 AND 已疯狂（sane=false 或被强制疯狂）AND 非强制顿悟 |
| `IsEnlightened()` | 顿悟模式 AND 已顿悟 AND 非强制疯狂 |

#### IsSane 判断的滞回设计

在 `DoDelta` 内部，阈值比较用的是 `percent_ignoresinduced`（不考虑 inducedinsanity/inducedlunacy 的"真实"百分比）：

```392:402:scripts/components/sanity.lua
	if self.mode == SANITY_MODE_INSANITY then
		if self.sane and percent_ignoresinduced <= TUNING.SANITY_BECOME_INSANE_THRESH then --30
			self.sane = false
		elseif not self.sane and percent_ignoresinduced >= TUNING.SANITY_BECOME_SANE_THRESH then --35
			self.sane = true
		end
	else
		if self.sane and percent_ignoresinduced >= TUNING.SANITY_BECOME_ENLIGHTENED_THRESH then
			self.sane = false
		elseif not self.sane and percent_ignoresinduced <= TUNING.SANITY_LOSE_ENLIGHTENMENT_THRESH then
			self.sane = true
		end
```

**疯狂模式阈值**（以 `maxhealth=200` 为例）：

```
进入疯狂：current / max  ≤  SANITY_BECOME_INSANE_THRESH  = 30/200 = 15%
恢复理智：current / max  ≥  SANITY_BECOME_SANE_THRESH   = 35/200 = 17.5%
```

存在 2.5% 的**滞回区间**（15%~17.5%）——避免临界状态反复横跳触发事件。

**顿悟模式阈值**：

```
进入顿悟：current / max  ≥  SANITY_BECOME_ENLIGHTENED_THRESH = 170/200 = 85%
失去顿悟：current / max  ≤  SANITY_LOSE_ENLIGHTENMENT_THRESH = 165/200 = 82.5%
```

---

### 18.3.2（新手）DoDelta / gosane / goinsane / goenlightened 事件

#### DoDelta —— 增减理智值

```377:381:scripts/components/sanity.lua
function Sanity:DoDelta(delta, overtime)
    if self.redirect ~= nil then
        self.redirect(self.inst, delta, overtime)
        return
    end

    if self.ignore then
        return
    end
```

**参数**：

| 参数 | 类型 | 含义 |
|---|---|---|
| `delta` | number | 理智变化量（正数=恢复，负数=消耗）|
| `overtime` | bool/nil | `true` = HUD 渐变动画 |

**特殊处理**：
1. `redirect` 存在 → 完全交给 redirect（类似 hunger/health）
2. `ignore = true` → 直接返回，理智系统失效（友好模式 `no_sanity`）
3. `current` 被 clamp 到 `[0, max × (1 - penalty)]`

**最简用法**：

```lua
-- 神秘书籍读完后扣 15 点理智
inst.components.sanity:DoDelta(-TUNING.SANITY_MED)

-- 理智回复物品（常以 overtime=true 配合 HUD 渐变动画）
inst.components.sanity:DoDelta(TUNING.SANITY_LARGE, true)
```

#### 常用 TUNING 理智变化量

```2016:2021:scripts/tuning.lua
        SANITY_TINY = 5,
        SANITY_SMALL = 10,
        SANITY_MED = 15,

        SANITY_LARGE = 33,
        SANITY_HUGE = 50,
```

#### sanitydelta / gosane / goinsane / goenlightened 事件

```405:433:scripts/components/sanity.lua
    self.inst:PushEvent("sanitydelta", { oldpercent = self._oldpercent, newpercent = self:GetPercent(), overtime = overtime, sanitymode = self.mode })
    -- ...
    if self:IsSane() ~= self._oldissane or (not self._oldissane and self.mode ~= self._oldmode) then
        -- ...
        if self._oldissane then
            self.inst:PushEvent("gosane")
        else
            if self.mode == SANITY_MODE_INSANITY then
                self.inst:PushEvent("goinsane")
            else
                self.inst:PushEvent("goenlightened")
            end
        end
    end
```

| 事件 | 键名 | 触发时机 |
|---|---|---|
| `"sanitydelta"` | oldpercent, newpercent, overtime, sanitymode | 每次理智变化 |
| `"gosane"` | 无 | IsSane 从 false→true |
| `"goinsane"` | 无 | INSANITY 模式下 IsSane 从 true→false |
| `"goenlightened"` | 无 | LUNACY 模式下 IsSane 从 true→false（即进入顿悟）|
| `"sanitymodechanged"` | mode | 疯狂/顿悟模式切换 |

---

### 18.3.3（新手）sanityaura 组件 —— 让 entity 影响周围玩家理智

#### SanityAura 组件概述

```1:9:scripts/components/sanityaura.lua
local SanityAura = Class(function(self, inst)
    self.inst = inst
    self.aura = 0
    
    --self.max_distsq = nil
    --self.aurafn = nil
    --self.fallofffn = nil

	self.inst:AddTag("sanityaura")
end)
```

添加 `sanityaura` 组件会给 entity 加上 `"sanityaura"` 标签——`Sanity.Recalc` 用 `TheSim:FindEntities` 按此标签搜索附近 entity。

#### 核心函数 GetAura

```21:28:scripts/components/sanityaura.lua
function SanityAura:GetAura(observer)
	local aura_val = 0
	local distsq = observer:GetDistanceSqToInst(self.inst)
	if distsq <= (self.max_distsq or SANITY_EFFECT_RANGE_SQ) then
	    aura_val = (self.aurafn == nil and self.aura or self.aurafn(self.inst, observer)) / (self.fallofffn ~= nil and self.fallofffn(self.inst, observer, distsq) or math.max(1, distsq))
	end
    return aura_val
end
```

**三个可配置字段**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `aura` | number | 固定光环值（正=恢复，负=降低）|
| `aurafn` | function/nil | 动态光环函数 `function(inst, observer) return val end` |
| `max_distsq` | number/nil | 最大作用距离平方（nil = `SANITY_EFFECT_RANGE = 10` 的平方）|
| `fallofffn` | function/nil | 距离衰减函数 `function(inst, observer, distsq) return divisor end` |

**距离衰减公式**：`aura_val = baseAura / max(1, distsq)`——距离越远，影响越小（平方衰减）。

#### 常用光环强度 TUNING 值

```2188:2192:scripts/tuning.lua
        SANITYAURA_TINY = 100/(seg_time*32),       -- 50    per day.
        SANITYAURA_SMALL_TINY = 100/(seg_time*20), -- 80    per day.
        SANITYAURA_SMALL = 100/(seg_time*8),       -- 200   per day.
        SANITYAURA_MED = 100/(seg_time*5),         -- 320   per day.
        SANITYAURA_LARGE = 100/(seg_time*2),       -- 800   per day.
```

（所有值均为"每单位时间的速率"——实际每秒影响 = aura / distsq）

#### 最简用法：给 mod 怪物添加负面光环

```lua
-- 在 prefab 中
inst:AddComponent("sanityaura")
inst.components.sanityaura.aura = -TUNING.SANITYAURA_MED  -- 中等负面光环

-- 更大范围的光环
inst.components.sanityaura.aura = -TUNING.SANITYAURA_LARGE
inst.components.sanityaura.max_distsq = 15 * 15  -- 15 格范围（默认 10 格）
```

**正面光环（提高理智）**：

```lua
inst:AddComponent("sanityaura")
inst.components.sanityaura.aura = TUNING.SANITYAURA_SMALL  -- 小型正面光环（如格洛摩）
```

---

### 18.3.4（进阶）Recalc：5 类理智变化来源详解

`Recalc(dt)` 在每帧 OnUpdate 中被调用，汇总所有理智变化来源：

```490:585:scripts/components/sanity.lua
function Sanity:Recalc(dt)
    -- 1. dapper_delta：装备魅力值
    -- 2. moisture_delta：潮湿度惩罚
    -- 3. light_delta：光照
    -- 4. aura_delta：附近光环
    -- 5. ghost_delta：玩家幽灵惩罚
    -- + externalmodifiers：外部修饰
    -- + custom_rate_fn：自定义速率
    self.rate = Σ × self.rate_modifier
    self:DoDelta(self.rate * dt, true)
end
```

#### 来源 1：dapper_delta —— 装备魅力值

```491:504:scripts/components/sanity.lua
	if self.dapperness_mult ~= 0 then
		local total_dapperness = self.dapperness
		for k, v in pairs(self.inst.components.inventory.equipslots) do
            local equippable = v.components.equippable
            if equippable ~= nil then
                local item_dapperness = self.get_equippable_dappernessfn ~= nil and self.get_equippable_dappernessfn(self.inst, equippable) or equippable:GetDapperness(self.inst, self.no_moisture_penalty)
                total_dapperness = total_dapperness + item_dapperness
            end
		end

		total_dapperness = total_dapperness * self.dapperness_mult
		dapper_delta = total_dapperness * TUNING.SANITY_DAPPERNESS
	end
```

**字段说明**：
- `sanity.dapperness`：角色本身的基础魅力值（不依赖装备）
- `equippable.dapperness`：装备的魅力值
- `sanity.dapperness_mult`：魅力值的全局乘数（默认 1）
- `TUNING.SANITY_DAPPERNESS = 1`：单位魅力对应理智速率的比例系数

**装备魅力 TUNING 值**：

```2197:2200:scripts/tuning.lua
        DAPPERNESS_SMALL = 100/(day_time*10),      -- 16.0 per day.
        DAPPERNESS_MED = 100/(day_time*6),         -- 26.7 per day.
        DAPPERNESS_MED_LARGE = 100/(day_time*4.5), -- 35.5 per day.
        DAPPERNESS_LARGE = 100/(day_time*3),       -- 53.3 per day.
```

负值代表有魅力惩罚（如void cloth系列装备 `dapperness = -TUNING.DAPPERNESS_MED`）。

#### 来源 2：moisture_delta —— 潮湿度惩罚

```506:506:scripts/components/sanity.lua
    local moisture_delta = (self.inst.components.moisture == nil or self.no_moisture_penalty) and 0 or easing.inSine(self.inst.components.moisture:GetMoisture(), 0, TUNING.MOISTURE_SANITY_PENALTY_MAX, self.inst.components.moisture:GetMaxMoisture())
```

使用 `easing.inSine` 缓动函数——潮湿度越大，理智下降越快（但不是线性的）。`MOISTURE_SANITY_PENALTY_MAX` 是潮湿满格时的最大惩罚速率。设置 `sanity.no_moisture_penalty = true` 可免疫湿度理智惩罚。

#### 来源 3：light_delta —— 光照

```508:522:scripts/components/sanity.lua
    local light_sanity_drain = LIGHT_SANITY_DRAINS[self.mode]
	-- 白天（且非洞穴）
    if TheWorld.state.isday and not TheWorld:HasTag("cave") then
        light_delta = light_sanity_drain.DAY
    else
        -- 夜晚/黄昏：根据光照值
        local lightval = CanEntitySeeInDark(self.inst) and .9 or self.inst.LightWatcher:GetLightValue()
        light_delta =
            (lightval > TUNING.SANITY_HIGH_LIGHT and light_sanity_drain.NIGHT_LIGHT) or
            (lightval < TUNING.SANITY_LOW_LIGHT  and light_sanity_drain.NIGHT_DARK) or
            light_sanity_drain.NIGHT_DIM
```

**INSANITY 模式的光照速率**：

```2174:2178:scripts/tuning.lua
        SANITY_DAY_GAIN = 0,          -- 白天无理智变化（地面）
        SANITY_NIGHT_LIGHT = -100/(night_time*20),  -- 夜晚有光 / 黄昏：缓慢下降
        SANITY_NIGHT_MID = -100/(night_time*20),    -- 黄昏（相同速率）
        SANITY_NIGHT_DARK = -100/(night_time*2),    -- 完全黑暗：快速下降（10倍）
```

**光照判断阈值**：
```
SANITY_HIGH_LIGHT = 0.6  →  亮（有光源/篝火附近）
SANITY_LOW_LIGHT  = 0.1  →  暗（远离光源）
```

**免疫光照理智消耗**：

```lua
inst.components.sanity.light_drain_immune_sources:SetModifier(source, true, "reason")
-- 移除
inst.components.sanity.light_drain_immune_sources:RemoveModifier(source, "reason")
```

#### 来源 4：aura_delta —— 附近 sanityaura 实体

```524:560:scripts/components/sanity.lua
	if not self.sanity_aura_immune_sources:Get() then
		local x, y, z = self.inst.Transform:GetWorldPosition()
	    local ents = TheSim:FindEntities(x, y, z, TUNING.SANITY_AURA_SEACH_RANGE, SANITYRECALC_MUST_TAGS, SANITYRECALC_CANT_TAGS)
	    for i, v in ipairs(ents) do
	        if v.components.sanityaura ~= nil and v ~= self.inst then
                -- 检查特定光环免疫
                -- 计算 aura_val（考虑 neg_aura 吸收和乘数）
                aura_delta = aura_delta + aura_val
            end
        end
    end
```

搜索范围：`TUNING.SANITY_AURA_SEACH_RANGE = 20`（格），但实际作用距离受 `sanityaura.max_distsq` 限制。

#### 来源 5：ghost_delta —— 玩家幽灵

```562:563:scripts/components/sanity.lua
    self:RecalcGhostDrain()
    local ghost_delta = TUNING.SANITY_GHOST_PLAYER_DRAIN * self.ghost_drain_mult
```

多个玩家成为幽灵时，联机模式下活着的玩家理智会下降（多人联机的压迫感设计）。`player_ghost_immune_sources` 可免疫此惩罚。

---

### 18.3.5（进阶）dapperness 系统 —— 装备魅力值

#### equippable.dapperness 的设置

在 prefab 的 equippable 初始化中设置：

```lua
inst:AddComponent("equippable")
inst.components.equippable.equipslot = EQUIPSLOTS.HEAD

-- 正值：装备后提高理智回复速率
inst.components.equippable.dapperness = TUNING.DAPPERNESS_LARGE    -- 高魅力（顶帽等）

-- 负值：装备后降低理智回复/加速理智消耗
inst.components.equippable.dapperness = -TUNING.DAPPERNESS_MED     -- 中等负魅力（黑暗装备）
```

#### sanity.dapperness —— 角色基础魅力

```lua
-- 在 master_postinit 中设置角色自带的理智回复（不依赖装备）
inst.components.sanity.dapperness = TUNING.DAPPERNESS_LARGE  -- 如 Maxwell 用影子力量维持理智
```

Waxwell 用此字段设置影子仆人带来的基础魅力：

```303:303:scripts/prefabs/player_common.lua
    inst.components.sanity.dapperness = TUNING.DAPPERNESS_LARGE
```

#### sanity.dapperness_mult —— 全局魅力乘数

```lua
-- 某个 debuff 使角色所有魅力效果减半
inst.components.sanity.dapperness_mult = 0.5
```

---

### 18.3.6（进阶）各类免疫源（SourceModifierList 布尔模式）

Sanity 中有多个以 `_sources` 结尾的 `SourceModifierList`，使用布尔模式（`OR` 语义：任意一个来源为 true 则生效）：

| 字段 | 作用 | 设置方式 |
|---|---|---|
| `sanity_aura_immune_sources` | 完全免疫所有 sanityaura 光环 | `SetFullAuraImmunity(true, source)` |
| `neg_aura_immune_sources` | 免疫负面 sanityaura 光环 | `SetNegativeAuraImmunity(true, source)` |
| `player_ghost_immune_sources` | 免疫玩家幽灵理智惩罚 | `SetPlayerGhostImmunity(true, source)` |
| `light_drain_immune_sources` | 免疫光照理智变化 | `SetLightDrainImmune(true, source)` |
| `_lunacy_sources` | 进入顿悟模式 | `EnableLunacy(true, source)` |

**设置示例**：

```lua
-- 某装备使玩家免疫负面光环
local function OnEquip(inst, data)
    inst.components.sanity:SetNegativeAuraImmunity(true, data.item)
end
local function OnUnequip(inst, data)
    inst.components.sanity:SetNegativeAuraImmunity(false, data.item)
end
inst:ListenForEvent("equip", OnEquip)
inst:ListenForEvent("unequip", OnUnequip)
```

#### neg_aura_modifiers —— 负面光环乘数

```94:96:scripts/components/sanity.lua
    self.neg_aura_mult = 1 -- Deprecated, use the SourceModifier below
    self.neg_aura_modifiers = SourceModifierList(self.inst)
    self.neg_aura_absorb = 0
```

- `neg_aura_modifiers`：乘以负面光环强度（`1.5` = 负面光环效果×1.5 倍，更快失智）
- `neg_aura_absorb`：负面光环的吸收比（`0.5` = 吸收50%负面光环）

---

### 18.3.7（老手）SetInducedInsanity / SetInducedLunacy —— 强制精神状态

#### SetInducedInsanity —— 强制疯狂

```331:351:scripts/components/sanity.lua
function Sanity:SetInducedInsanity(src, val)
    if val then
        if self.inducedinsanity_sources == nil then
            self.inducedinsanity_sources = { [src] = true }
        else
            self.inducedinsanity_sources[src] = true
        end
    elseif self.inducedinsanity_sources ~= nil then
        self.inducedinsanity_sources[src] = nil
        -- 若没有其他来源，清除 inducedinsanity
    end
    if self.inducedinsanity ~= val then
        self.inducedinsanity = val
        self:DoDelta(0)
        self.inst:PushEvent("inducedinsanity", val)
    end
end
```

`inducedinsanity = true` 时：
- `IsInsane()` 返回 `true`（无视实际理智值）
- `GetPercent()` 返回 `0`（UI 显示 0 理智）
- 暗影生物会攻击玩家

**使用场景**：某个物品/状态强制玩家进入疯狂（如某些 boss 技能）：

```lua
-- 进入强制疯狂
inst.components.sanity:SetInducedInsanity(my_source, true)

-- 解除强制疯狂
inst.components.sanity:SetInducedInsanity(my_source, false)
```

> **多来源管理**：`_sources` 表记录所有使其疯狂的来源——只有所有来源都移除，才真正解除强制疯狂。

#### SetInducedLunacy —— 强制顿悟

```354:374:scripts/components/sanity.lua
function Sanity:SetInducedLunacy(src, val)
    -- 类似 SetInducedInsanity 的多来源管理
    if self.inducedlunacy ~= val then
        self.inducedlunacy = val
        self:DoDelta(0)
        self.inst:PushEvent("inducedlunacy", val)
    end
end
```

`inducedlunacy = true` 时：
- `IsEnlightened()` 在 LUNACY 模式下返回 `true`
- `GetPercent()` 返回 `1 - penalty`（UI 显示满理智）
- 月形生物视为友好

---

### 18.3.8（老手）externalmodifiers + custom_rate_fn —— 外部速率修改

#### externalmodifiers —— 外部加法修饰

```87:87:scripts/components/sanity.lua
    self.externalmodifiers = SourceModifierList(self.inst, 0, SourceModifierList.additive)
```

`externalmodifiers` 是 additive SourceModifierList——多个来源的值**直接相加**，最终加到 `rate` 上：

```lua
-- 某增益 buff 提供持续理智回复（+0.05/s）
inst.components.sanity.externalmodifiers:SetModifier(buff_source, 0.05, "sanity_buff")

-- 某 debuff 加速理智下降（-0.1/s）
inst.components.sanity.externalmodifiers:SetModifier(debuff_source, -0.1, "sanity_debuff")

-- 移除
inst.components.sanity.externalmodifiers:RemoveModifier(buff_source)
```

#### custom_rate_fn —— 完全自定义速率

```107:107:scripts/components/sanity.lua
    self.custom_rate_fn = nil
```

```567:571:scripts/components/sanity.lua
    if self.custom_rate_fn ~= nil then
        --NOTE: dt param was added for wormwood's custom rate function
        --      dt shouldn't have been applied to the return value yet
        self.rate = self.rate + self.custom_rate_fn(self.inst, dt)
    end
```

**函数签名**：

```lua
sanity.custom_rate_fn = function(inst, dt)
    -- 返回额外的理智速率（不乘以 dt，内部已处理）
    -- 正数 = 额外回复，负数 = 额外消耗
    return extra_rate
end
```

**使用场景（Wormwood）**：Wormwood 有独特的花期理智回复机制，通过 `custom_rate_fn` 实现，而不是污染通用的理智系统。

```lua
-- mod 角色：在血量低时理智快速下降
inst.components.sanity.custom_rate_fn = function(inst, dt)
    local health_pct = inst.components.health:GetPercent()
    if health_pct < 0.3 then
        return -0.05 * (1 - health_pct / 0.3)  -- 血量越低，理智越快下降
    end
    return 0
end
```

#### rate_modifier —— 全局速率乘数

```lua
-- 将所有理智变化速率乘以 0.5（恢复/消耗都减半）
inst.components.sanity.rate_modifier = 0.5
```

---

### 18.3.9（老手）联机同步机制 —— sanity_replica

#### 架构

```
┌─────────────────────────────────────────────────────────┐
│  服务端                                                   │
│   sanity.current / .max / .penalty / .sane / .mode       │
│     ↓（SourceField listeners: oncurrent/onmax/onsane等）  │
│   replica.sanity（sanity_replica）                        │
│     → SetCurrent/SetMax/SetIsSane/SetSanityMode           │
│     ↓                                                     │
│   player_classified net 变量 + net_bool _issane          │
└─────────────────────────────────────────────────────────┘
                      │ 网络同步 │
┌─────────────────────────────────────────────────────────┐
│  客户端                                                   │
│   replica.sanity（sanity_replica）                        │
│   监听：issanedirty → PushEvent("gosane"/"goinsane"/"goenlightened")
│         isinsanitymodedirty → PushEvent("sanitymodechanged")│
└─────────────────────────────────────────────────────────┘
```

**理智同步的关键特点**：
- `sane`（是否理智）通过独立的 `net_bool _issane` 同步，保证客户端的 `gosane`/`goinsane` 事件触发
- `current`（当前理智值）通过 `classified` 的 net 变量同步，用于 HUD 显示
- 理智模式（INSANITY/LUNACY）通过 `net_bool _isinsanitymode` 同步

**客户端读取理智**：

```lua
-- 通过 replica
local sanity_pct = ThePlayer.replica.sanity:GetPercent()
local is_sane = ThePlayer.replica.sanity:IsSane()
```

---

### 18.3.10（小结）速查表 + mod 实战 4 模板

#### Sanity 组件 API 速查

| 函数/字段 | 参数 | 说明 |
|---|---|---|
| `SetMax(n)` | number | 设置最大理智（同时满理智）|
| `DoDelta(delta, overtime)` | — | 增减理智（正=恢复，负=消耗）|
| `SetPercent(pct, overtime)` | 0~1 | 设为 max 的 pct 倍 |
| `IsSane()` | 无 | 当前是否"正常" |
| `IsInsane()` | 无 | 疯狂模式下是否已疯狂 |
| `IsEnlightened()` | 无 | 顿悟模式下是否已顿悟 |
| `EnableLunacy(enable, source)` | bool, key | 进入/退出顿悟模式 |
| `SetInducedInsanity(src, val)` | entity, bool | 强制疯狂（多来源）|
| `SetInducedLunacy(src, val)` | entity, bool | 强制顿悟（多来源）|
| `AddSanityPenalty(key, mod)` | string, 0~1 | 添加理智上限惩罚 |
| `RemoveSanityPenalty(key)` | string | 移除理智上限惩罚 |
| `SetFullAuraImmunity(b, src)` | bool, key | 免疫所有 sanityaura |
| `SetNegativeAuraImmunity(b, src)` | bool, key | 免疫负面 sanityaura |
| `SetLightDrainImmune(b, src)` | bool, key | 免疫光照理智变化 |
| `SetPlayerGhostImmunity(b, src)` | bool, key | 免疫玩家幽灵惩罚 |

#### 关键字段速查

| 字段 | 类型 | 说明 |
|---|---|---|
| `sanity.ignore` | bool | `true` = 理智系统完全停用（友好模式）|
| `sanity.dapperness` | number | 角色基础魅力（不依赖装备）|
| `sanity.dapperness_mult` | number | 魅力全局乘数 |
| `sanity.rate_modifier` | number | 所有速率的全局乘数 |
| `sanity.no_moisture_penalty` | bool | 免疫湿度惩罚 |
| `sanity.custom_rate_fn` | function | 自定义额外速率 |
| `sanity.night_drain_mult` | number | 夜晚理智消耗的额外乘数 |
| `sanity.externalmodifiers` | SourceModifierList | 外部加法速率修饰 |
| `sanity.neg_aura_modifiers` | SourceModifierList | 负面光环强度乘数 |
| `sanity.neg_aura_absorb` | number | 负面光环吸收比（0~1）|

#### SanityAura 字段速查

| 字段 | 类型 | 说明 |
|---|---|---|
| `sanityaura.aura` | number | 固定光环值（正=恢复，负=消耗）|
| `sanityaura.aurafn` | function | 动态光环函数 |
| `sanityaura.max_distsq` | number | 最大作用距离平方（默认 10^2=100）|
| `sanityaura.fallofffn` | function | 距离衰减函数 |

#### 4 个 mod 常用模板

##### 模板 A：基础角色理智初始化

```lua
local function master_postinit(inst)
    inst.components.sanity:SetMax(200)                     -- 理智上限 200
    inst.components.sanity.dapperness = TUNING.DAPPERNESS_MED  -- 角色自带魅力
    inst.components.sanity.ignore = false                  -- 确保理智系统激活
end
```

##### 模板 B：给 mod 怪物添加理智光环

```lua
-- 中等负面光环 + 更大范围
inst:AddComponent("sanityaura")
inst.components.sanityaura.aura = -TUNING.SANITYAURA_MED
inst.components.sanityaura.max_distsq = 12 * 12  -- 12 格范围
```

##### 模板 C：监听疯狂/恢复事件触发角色反应

```lua
inst:ListenForEvent("goinsane", function(inst)
    -- 进入疯狂时触发变形/特殊状态
    if not inst:HasTag("cursed") then
        inst:AddTag("cursed")
        inst.components.talker:Say(GetString(inst, "ANNOUNCE_INSANE"))
        -- 触发变形特效
        SpawnPrefab("shadowcrumble").Transform:SetPosition(inst.Transform:GetWorldPosition())
    end
end)

inst:ListenForEvent("gosane", function(inst)
    inst:RemoveTag("cursed")
end)
```

##### 模板 D：装备提供理智 buff（externalmodifiers）

```lua
-- 某特殊装备装备时添加持续理智回复
local function OnEquipSanityAmulet(inst, data)
    inst.components.sanity.externalmodifiers:SetModifier(data.item, 0.08, "sanity_amulet")
end

local function OnUnequipSanityAmulet(inst, data)
    inst.components.sanity.externalmodifiers:RemoveModifier(data.item)
end

local function OnInit(inst)
    local owner = inst.components.equippable.owner
    if owner ~= nil then
        OnEquipSanityAmulet(owner, { item = inst })
    end
end

inst:ListenForEvent("equipped", function(inst, data)
    OnEquipSanityAmulet(data.owner, { item = inst })
end)
inst:ListenForEvent("unequipped", function(inst, data)
    OnUnequipSanityAmulet(data.owner, { item = inst })
end)
```

---


## 18.4 Temperature 组件——温度、过热与冻结

### 本节导读

> **一句话定位**：`Temperature` 是饥荒"冬天会冻死、夏天会热死"机制的核心——它每帧计算 entity 的**体温**（受环境温度、保温装备、加热器、饮食、避雨状态、潮湿度影响），当体温跌破 0 或超过 70 时开始扣血，但两套保温公式能减缓温度变化速率——让你在冬天穿厚衣服撑更久。

#### 一段宏观先讲清楚：体温是如何变化的

```
【每帧 OnUpdate(dt)】
    │
    ├── settemp ≠ nil（强制体温）→ 直接跳过计算
    ├── IsInvincible() / is_teleporting → 跳过计算
    │
    └── 正常计算流程
           │
           ├── Step 1: 确定环境温度（ambient_temperature）
           │           = GetTemperatureAtXZ(x, z)  ← 世界当前温度
           │           若在冰箱里：特殊逻辑
           │           若在睡袋里：用睡袋温度
           │
           ├── Step 2: 计算目标温差 delta
           │           = ambient_temperature + totalModifiers + moisturePenalty
           │             - current
           │           + 装备加热器/冷却器增量（equipped heater）
           │           + 携带物加热器/冷却器增量（carried heater）
           │           + 吃食物的体温增量（belly temperature）
           │           + 树荫冷却（sheltered）
           │           + 世界中 ZERO_DISTANCE(10格) 内 HASHEATER entity
           │
           ├── Step 3: 应用保温系数（insulation）
           │           夏/热时（delta > 0）：rate = delta × SEG_TIME/(SEG_TIME + summerInsulation)
           │           冬/冷时（delta < 0）：rate = delta × SEG_TIME/(SEG_TIME + winterInsulation)
           │           解冻/散热时：rate 上限为 THAW_DEGREES_PER_SEC(5) 或 WARM_DEGREES_PER_SEC(1)
           │
           ├── Step 4: clamp 到 [mintemp(-20), maxtemp(90)]
           │           SetTemperature(current + rate * dt)
           │
           └── Step 5: 温度危险检查
                       若 current < 0      → health:DoDelta(-hurtrate * dt, "cold")
                       若 current > overheattemp → health:DoDelta(-overheathurtrate * dt, "hot")
```

#### 18.4 节回答的 5 个核心问题

```
Q1: ──── 体温的正常范围是多少？过热/冻结的临界点？
         ↓ 答：正常 0~70°；STARTING_TEMP=35°；冻结<0°；过热>70°

Q2: ──── 如何让 mod entity 开始冻伤或过热？
         ↓ 答：inst:AddComponent("temperature") 即可，框架自动处理

Q3: ──── 装备是怎么保温的？
         ↓ 答：equippable → insulator 组件，GetInsulation() 汇总后缩小温变速率

Q4: ──── 如何让一个 entity（如火堆）给附近玩家加温？
         ↓ 答：inst:AddComponent("heater") + heater.heat = 90（目标温度）

Q5: ──── 如何强制设置体温（如变身为冰霜兽）？
         ↓ 答：temperature:SetTemp(value)（设后不再随环境变化）
```

#### 本节学习路径

```
18.4.1 新手 ─── Temperature 三个临界点与体温范围
18.4.2 新手 ─── SetTemperature / IsFreezing / IsOverheating / 相关事件
18.4.3 新手 ─── 冻伤与过热扣血 —— hurtrate 计算
                  ↓
18.4.4 进阶 ─── OnUpdate 完整计算流程 —— 5 个体温来源
18.4.5 进阶 ─── SetModifier / RemoveModifier —— 外部温度调整器
18.4.6 进阶 ─── GetInsulation / inherentinsulation —— 保温系数详解
18.4.7 进阶 ─── SetTemperatureInBelly —— 吃热食/冷食物
                  ↓
18.4.8 老手 ─── Heater 组件 —— 主动加热/冷却的 entity
18.4.9 老手 ─── Insulator 组件 —— 装备保温值
18.4.10     ─── 小结·速查表 + mod 实战 3 模板
```

---

### 18.4.1（新手）Temperature 三个临界点与体温范围

#### 关键 TUNING 常量

```2282:2289:scripts/tuning.lua
        STARTING_TEMP = 35,
        OVERHEAT_TEMP = 70,

        MIN_ENTITY_TEMP = -20,
        MAX_ENTITY_TEMP = 90,

        WARM_DEGREES_PER_SEC = 1,
        THAW_DEGREES_PER_SEC = 5,
```

**体温示意图**：

```
MIN_ENTITY_TEMP     0         STARTING_TEMP    OVERHEAT_TEMP    MAX_ENTITY_TEMP
      │             │               │                │                │
     -20            0              35               70               90
      │             │               │                │                │
     极寒           │         舒适温度              │              极热
      │          ←冻结区→        正常区          ←过热区→            │
      │         扣blood hp        无伤害           扣blood hp         │
```

| TUNING 值 | 数值 | 含义 |
|---|---|---|
| `STARTING_TEMP` | 35 | 初始体温 / 舒适温度（偏体温）|
| `OVERHEAT_TEMP` | 70 | 过热临界点（>70 开始扣血）|
| `MIN_ENTITY_TEMP` | -20 | 体温下限（不会低于此）|
| `MAX_ENTITY_TEMP` | 90 | 体温上限（不会高于此）|
| `WARM_DEGREES_PER_SEC` | 1 | 正常温度变化速率上限（°/秒）|
| `THAW_DEGREES_PER_SEC` | 5 | 解冻/散热速率上限（°/秒，更快）|

#### 组件初始化字段

```23:55:scripts/components/temperature.lua
local Temperature = Class(function(self, inst)
    self.inst = inst
    self.settemp = nil
    self.current = TUNING.STARTING_TEMP
    self.maxtemp = TUNING.MAX_ENTITY_TEMP
    self.mintemp = TUNING.MIN_ENTITY_TEMP
    self.overheattemp = TUNING.OVERHEAT_TEMP
    self.hurtrate = TUNING.WILSON_HEALTH / TUNING.FREEZING_KILL_TIME
    --self.overheathurtrate = nil --defaults to use same as .hurtrate (freezing rate)
    self.inherentinsulation = 0
    self.inherentsummerinsulation = 0
    self.shelterinsulation = TUNING.INSULATION_MED_LARGE
    -- ...
```

| 字段 | 默认 | 含义 |
|---|---|---|
| `current` | 35 | 当前体温 |
| `maxtemp` | 90 | 最大体温（可自定义）|
| `mintemp` | -20 | 最小体温（可自定义）|
| `overheattemp` | 70 | 过热临界点（可自定义）|
| `hurtrate` | 150/120=1.25 | 冻结/过热时每秒扣血量 |
| `overheathurtrate` | nil（同 hurtrate）| 过热时专用扣血速率 |
| `inherentinsulation` | 0 | 固有冬季保温值 |
| `inherentsummerinsulation` | 0 | 固有夏季保温值 |
| `settemp` | nil | 强制体温（非 nil 时禁用计算）|

---

### 18.4.2（新手）SetTemperature / IsFreezing / IsOverheating / 相关事件

#### SetTemperature —— 设置当前体温（并触发事件）

```163:176:scripts/components/temperature.lua
function Temperature:SetTemperature(value)
    local last = self.current
    self.current = value

    if (self.current < 0) ~= (last < 0) then
        self.inst:PushEvent(self.current < 0 and "startfreezing" or "stopfreezing")
    end

    if (self.current > self.overheattemp) ~= (last > self.overheattemp) then
        self.inst:PushEvent(self.current > self.overheattemp and "startoverheating" or "stopoverheating")
    end

    self.inst:PushEvent("temperaturedelta", { last = last, new = self.current, hasrate = self.rate ~= 0 })
end
```

**每次体温变化都会触发 `"temperaturedelta"` 事件**，跨越临界点时额外触发状态进出事件。

#### 状态判断函数

```187:193:scripts/components/temperature.lua
function Temperature:IsFreezing()
    return self.current < 0
end

function Temperature:IsOverheating()
    return self.current > self.overheattemp
end
```

#### 相关事件速查

| 事件 | 触发时机 | 数据键名 |
|---|---|---|
| `"temperaturedelta"` | 每次体温变化 | `last`(旧温度), `new`(新温度), `hasrate`(是否有速率) |
| `"startfreezing"` | current 从 ≥0 → <0 | 无 |
| `"stopfreezing"` | current 从 <0 → ≥0 | 无 |
| `"startoverheating"` | current 从 ≤overheattemp → >overheattemp | 无 |
| `"stopoverheating"` | current 从 >overheattemp → ≤overheattemp | 无 |

**监听示例**：

```lua
inst:ListenForEvent("startfreezing", function(inst)
    inst.components.talker:Say(GetString(inst, "ANNOUNCE_COLD"))
    -- 触发冻结特效
end)

inst:ListenForEvent("stopfreezing", function(inst)
    -- 停止冻结特效
end)
```

---

### 18.4.3（新手）冻伤与过热扣血 —— hurtrate 计算

```472:478:scripts/components/temperature.lua
    if applyhealthdelta ~= false and self.inst.components.health ~= nil then
        if self.current < 0 then
            self.inst.components.health:DoDelta(-self.hurtrate * dt, true, "cold")
        elseif self.current > self.overheattemp then
            self.inst.components.health:DoDelta(-(self.overheathurtrate or self.hurtrate) * dt, true, "hot")
        end
    end
```

**关键**：
- 冻结时扣血原因：`"cold"`
- 过热时扣血原因：`"hot"`
- `health.nonlethal_temperature = false`（默认）：冻死/热死致命
- `health.nonlethal_temperature = true`：冻伤/过热在血量低于 20% 时不再扣血（永远不会死）

**默认扣血速率**：

```lua
hurtrate = WILSON_HEALTH / FREEZING_KILL_TIME = 150 / 120 = 1.25 点/秒
-- 120 秒内死亡（与饥饿一样）
```

**设置不同的扣血速率**：

```lua
-- 冻结/过热同一速率（更耐冻）
inst.components.temperature:SetFreezingHurtRate(150 / 180)  -- 需要 180 秒才冻死

-- 分开设置：对过热更脆弱
inst.components.temperature:SetFreezingHurtRate(150 / 120)
inst.components.temperature:SetOverheatHurtRate(150 / 60)   -- 只需 60 秒过热死
```

**Willow 的快速冻结**（相比 Wilson 耐过热但不耐冻）：

```3747:3747:scripts/tuning.lua
        WILLOW_FREEZING_KILL_TIME = 60,
```

---

### 18.4.4（进阶）OnUpdate 完整计算流程 —— 5 个体温来源

`OnUpdate(dt)` 每帧执行，整合所有温度来源：

#### 来源 1：环境温度

```295:295:scripts/components/temperature.lua
    local ambient_temperature = inside_pocket_container and TheWorld.state.temperature or GetTemperatureAtXZ(x, z)
```

`GetTemperatureAtXZ(x, z)` 是 C++ 函数，返回指定坐标的当前环境温度（受季节、地块等影响）。

#### 来源 2：体温修改器（totalModifiers）

`totalmodifiers` 字段是所有通过 `SetModifier/RemoveModifier` 设置的修改器之和，加到环境温度上，直接影响"目标温度"。

#### 来源 3：潮湿度惩罚

```270:271:scripts/components/temperature.lua
function Temperature:GetMoisturePenalty()
    return self.inst.components.moisture ~= nil and -Lerp(0, self.maxmoisturepenalty, self.inst.components.moisture:GetMoisturePercent()) or 0
end
```

`maxmoisturepenalty = TUNING.MOISTURE_TEMP_PENALTY = 30`——满潮湿时，感受到的温度降低 30°（即夏天 35° 感觉只有 5°）。

#### 来源 4：装备/携带物的 Heater

玩家装备槽和背包中有 `heater` 组件的物品（如热石头、冰块）会修改 delta：

```334:367:scripts/components/temperature.lua
        if self.inst.components.inventory ~= nil then
            for k, v in pairs(self.inst.components.inventory.equipslots) do
                if v.components.heater ~= nil then
                    local heat = v.components.heater:GetEquippedHeat()
                    -- 如果目标温度高于当前且 IsExothermic，则增加 delta
                end
            end
            for k, v in pairs(self.inst.components.inventory.itemslots) do
                if v.components.heater ~= nil then
                    local heat, carriedmult = v.components.heater:GetCarriedHeat()
                    -- 携带物也影响体温
                end
            end
        end
```

#### 来源 5：世界中的加热器 entity（火堆、暖石等）

```313:437:scripts/components/temperature.lua
        ents = TheSim:FindEntities(x, y, z, ZERO_DISTANCE, ...)
        for i, v in ipairs(ents) do
            if v ~= self.inst and v.components.heater then
                -- heatfactor（0~1）根据距离线性衰减（0格=1，10格=0）
                -- 湿了时，加热效果打折（WET_HEAT_FACTOR_PENALTY = 0.75）
                -- 外热型（exothermic）：提升体温向 heat 靠近
                -- 内冷型（endothermic）：降低体温向 heat 靠近
            end
        end
```

**搜索范围**：`ZERO_DISTANCE = 10`（格）——超过 10 格的火堆不影响体温。

#### 保温系数的应用

计算出 delta 后，再按保温系数减缓变化速率：

```443:461:scripts/components/temperature.lua
        if ambient_temperature >= TUNING.STARTING_TEMP then
            -- 夏天（热）
            if self.delta > 0 then  -- 在加热，用夏季保温抵抗
                self.rate = math.min(self.delta, TUNING.SEG_TIME / (TUNING.SEG_TIME + summerInsulation))
            else
                self.rate = math.max(self.delta, -(TUNING.THAW_DEGREES_PER_SEC or WARM_DEGREES_PER_SEC))
            end
        elseif self.delta < 0 then  -- 冬天（冷），在降温，用冬季保温抵抗
            self.rate = math.max(self.delta, -TUNING.SEG_TIME / (TUNING.SEG_TIME + winterInsulation))
        else
            self.rate = math.min(self.delta, TUNING.THAW_DEGREES_PER_SEC)
        end
```

**SEG_TIME = 30 秒**——`SEG_TIME / (SEG_TIME + insulation)` 是保温系数的核心公式。`insulation` 越大，比值越小，温度变化越缓慢。

---

### 18.4.5（进阶）SetModifier / RemoveModifier —— 外部温度调整器

#### SetModifier —— 添加/修改温度修改器

```195:209:scripts/components/temperature.lua
function Temperature:SetModifier(name, value)
    if value == nil or value == 0 then
        return self:RemoveModifier(name)
    elseif self.temperature_modifiers == nil then
        self.temperature_modifiers = { [name] = value }
        self.totalmodifiers = value
        return
    end
    local m = self.temperature_modifiers[name]
    if m == value then
        return
    end
    self.temperature_modifiers[name] = value
    self.totalmodifiers = self.totalmodifiers + value - (m or 0)
end
```

**工作原理**：`temperature_modifiers` 是以 `name` 为键的字典，`totalmodifiers` 是所有修改器之和。修改器加到**环境温度**上，使体温"目标值"偏移。

| 用途 | 修改器值 | 效果 |
|---|---|---|
| 加热 buff（如食物温热）| +15 | 目标温度 +15°，在冬天更暖 |
| 冷却 debuff | -15 | 目标温度 -15°，在夏天更凉 |
| 变身为热血怪物 | +50 | 需要更多冷却才能过热 |

```lua
-- 某 buff 给角色提供 +10° 的体温提升（持续期间）
inst.components.temperature:SetModifier("warm_buff", 10)

-- buff 结束
inst.components.temperature:RemoveModifier("warm_buff")
```

#### DoDelta —— 立即改变当前体温

```70:80:scripts/components/temperature.lua
function Temperature:DoDelta(delta)
    local winterInsulation,summerInsulation = self:GetInsulation()

    if delta > 0 then
        delta = delta * (TUNING.SEG_TIME / (TUNING.SEG_TIME + summerInsulation))
    else
        delta = delta * (TUNING.SEG_TIME / (TUNING.SEG_TIME + winterInsulation))
    end

    self:SetTemperature(self.current + delta)
end
```

**注意**：`DoDelta` 会考虑保温系数减弱效果（与 OnUpdate 行为一致）。

```lua
-- 被冰霜攻击时立即降温 20°
inst.components.temperature:DoDelta(-20)

-- 喝热汤立即升温 10°（但保温系数可能衰减）
inst.components.temperature:DoDelta(10)
```

#### SetTemp —— 强制设置体温（禁用自动计算）

```156:161:scripts/components/temperature.lua
function Temperature:SetTemp(temp)
    self.settemp = temp
    if temp ~= nil then
        self:SetTemperature(temp)
    end
end
```

设置后 `settemp ~= nil`，OnUpdate 直接跳过所有计算——体温被"锁定"在指定值。传 `nil` 解除锁定。

```lua
-- 变身后锁定体温（不受环境影响）
inst.components.temperature:SetTemp(35)    -- 锁定为舒适温度

-- 恢复正常（重新受环境影响）
inst.components.temperature:SetTemp(nil)
```

---

### 18.4.6（进阶）GetInsulation / inherentinsulation —— 保温系数详解

#### 保温系数的汇总

```228:266:scripts/components/temperature.lua
function Temperature:GetInsulation()
    local winterInsulation = self.inherentinsulation
    local summerInsulation = self.inherentsummerinsulation

    if self.inst.components.inventory ~= nil then
        for k, v in pairs(self.inst.components.inventory.equipslots) do
            if v.components.insulator ~= nil then
                local insulationValue, insulationType = v.components.insulator:GetInsulation()
                if insulationType == SEASONS.WINTER then
                    winterInsulation = winterInsulation + insulationValue
                elseif insulationType == SEASONS.SUMMER then
                    summerInsulation = summerInsulation + insulationValue
                end
            end
        end
    end

    if self.inst.components.beard ~= nil then
        --Beards help winterInsulation but hurt summerInsulation
        winterInsulation = winterInsulation + self.inst.components.beard:GetInsulation()
        summerInsulation = summerInsulation - self.inst.components.beard:GetInsulation()
    end

    if self.sheltered then
        summerInsulation = summerInsulation + self.shelterinsulation
    end
    -- 黄昏/夜晚也有少量夏季保温
    return math.max(0, winterInsulation), math.max(0, summerInsulation)
end
```

**保温来源**：
1. `temperature.inherentinsulation`：角色/entity 固有冬季保温（如熊怪物形态）
2. `temperature.inherentsummerinsulation`：角色固有夏季保温
3. 装备的 `insulator` 组件（冬帽、防晒伞等）
4. `beard` 组件（Wilson 的胡须：冬天+，夏天-）
5. 有遮蔽（`sheltered = true`）：夏季保温 + `shelterinsulation`
6. 黄昏/夜晚：额外夏季保温（`DUSK_INSULATION_BONUS`/`NIGHT_INSULATION_BONUS`）

**关键常量**：

```2242:2245:scripts/tuning.lua
        INSULATION_SMALL = seg_time*2,
        INSULATION_MED = seg_time*4,
        INSULATION_MED_LARGE = seg_time*6,
        INSULATION_LARGE = seg_time*8,
```

（`seg_time = 30` 秒，所以 `INSULATION_LARGE = 240`）

**保温系数公式**：

```
温变速率 × (SEG_TIME / (SEG_TIME + insulation))
         = rate × (30 / (30 + insulation))

insulation = 0（无保温）：   rate × (30/30) = rate × 1.0  (全速变温)
insulation = 30（SMALL/1级）：rate × (30/60) = rate × 0.5  (半速)
insulation = 120（MED/4级）：  rate × (30/150) = rate × 0.2  (20%速)
insulation = 240（LARGE/8级）：rate × (30/270) ≈ rate × 0.11 (11%速)
```

**设置角色固有保温（如 Woodie 变身）**：

```1265:1266:scripts/prefabs/woodie.lua
    inst.components.temperature.inherentinsulation = TUNING.INSULATION_LARGE
    inst.components.temperature.inherentsummerinsulation = TUNING.INSULATION_LARGE
```

---

### 18.4.7（进阶）SetTemperatureInBelly —— 吃热食/冷食物

```91:98:scripts/components/temperature.lua
function Temperature:SetTemperatureInBelly(delta, duration)
    self.bellytemperaturedelta = delta
    self.bellytime = GetTime() + duration
    if self.bellytask ~= nil then
        self.bellytask:Cancel()
    end
    self.bellytask = self.inst:DoTaskInTime(duration, ClearBellyTemperature, self)
end
```

| 参数 | 含义 |
|---|---|
| `delta` | 每帧额外增加的温度增量（正=加热，负=冷却）|
| `duration` | 效果持续时间（秒），之后自动清除 |

**工作条件**（OnUpdate 中的检查）：

```373:378:scripts/components/temperature.lua
        if self.bellytemperaturedelta ~= nil and (
                (self.bellytemperaturedelta > 0 and self.current < TUNING.HOT_FOOD_WARMING_THRESHOLD) or
                (self.bellytemperaturedelta < 0 and self.current > TUNING.COLD_FOOD_CHILLING_THRESHOLD)
            ) then
            self.delta = self.delta + self.bellytemperaturedelta
        end
```

- 热食物（`delta > 0`）只在体温 `< HOT_FOOD_WARMING_THRESHOLD = 62°` 时生效（不让人过热）
- 冷食物（`delta < 0`）只在体温 `> COLD_FOOD_CHILLING_THRESHOLD = 12°` 时生效（不让人冻死）

**eater 组件中的调用**（当玩家吃食物时自动触发）：

```lua
-- 某热汤食物：吃下后 60 秒内保温
inst.components.temperature:SetTemperatureInBelly(5, 60)

-- 某冰镇饮料：吃下后 30 秒内降温
inst.components.temperature:SetTemperatureInBelly(-8, 30)
```

---

### 18.4.8（老手）Heater 组件 —— 主动加热/冷却的 entity

`heater` 组件让 entity（如火堆、冰箱、暖石）能够影响附近 entity 的体温。

#### Heater 的两种模式

```24:35:scripts/components/heater.lua
function Heater:SetThermics(exo, endo)
	self.exothermic = exo
	self.endothermic = endo
end
```

| 模式 | `exothermic` | `endothermic` | 效果 |
|---|---|---|---|
| 放热（加热）| true | false（默认）| 使周围体温趋向 `heat` 值（加热）|
| 吸热（冷却）| false | true | 使周围体温趋向 `heat` 值（冷却）|

**放热原理（`heat = 90`）**：

```
temperature 组件在 OnUpdate 中：
  warmingtemp = heat × heatfactor
  若 warmingtemp > current：
      delta += warmingtemp - current  （把体温推向 heat）
```

**吸热原理（`heat = 0`，吸热）**：

```
  coolingtemp = (heat - overheattemp) × heatfactor + overheattemp
  若 coolingtemp < current：
      delta += coolingtemp - current  （把体温推向 heat）
```

#### Heater 核心字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `heat` | number | 加热目标温度（放热时是 entity 的"发热温度"）|
| `heatfn` | function | 动态热量函数（替代 heat）|
| `equippedheat` | number | 装备时提供的热量 |
| `equippedheatfn` | function | 装备时动态热量函数 |
| `carriedheat` | number | 携带时提供的热量 |
| `carriedheatmultiplier` | number | 携带热量的衰减倍数 |
| `stop_falloff` | bool | `true` = 距离不衰减 |
| `radius_cutoff` | number | 有效半径（格），超过无效 |

#### 最简用法：给 mod entity 添加加热效果

```lua
-- 一个持续放热的篝火（类似）
inst:AddComponent("heater")
inst.components.heater.heat = 90        -- 目标温度 90°（高于 OVERHEAT_TEMP）
-- 默认 exothermic=true, endothermic=false
-- 距离衰减：10格内有效，越近越暖

-- 一个小型加热器（没有距离衰减，更均匀）
inst:AddComponent("heater")
inst.components.heater.heat = 50
inst.components.heater:SetShouldFalloff(false)  -- 不衰减

-- 一个冷却装置（如冰箱上的外部冷却）
inst:AddComponent("heater")
inst.components.heater.heat = 0
inst.components.heater:SetThermics(false, true)  -- 吸热模式
```

**装备或携带也能加热/冷却**（如热石头）：

```lua
inst:AddComponent("heater")
inst.components.heater.equippedheat = 50    -- 装备时目标温度 50°（只影响装备者）
inst.components.heater.carriedheat = 25     -- 携带时目标温度 25°（弱化版）
inst.components.heater.carriedheatmultiplier = 0.5  -- 携带效果再打折
```

---

### 18.4.9（老手）Insulator 组件 —— 装备保温值

`insulator` 组件让装备提供冬季或夏季保温值。

```1:31:scripts/components/insulator.lua
local Insulator = Class(function(self, inst)
    self.inst = inst
    self.insulation = 0
    self.type = SEASONS.WINTER
end)

function Insulator:SetSummer()
	self.type = SEASONS.SUMMER
end

function Insulator:SetWinter()
	self.type = SEASONS.WINTER
end

function Insulator:SetInsulation(val)
	self.insulation = val
end

function Insulator:GetInsulation()
	return self.insulation, self:GetType()
end
```

#### 在装备 prefab 中添加 Insulator

```lua
-- 冬季帽（防冻）
inst:AddComponent("insulator")
inst.components.insulator:SetWinter()
inst.components.insulator:SetInsulation(TUNING.INSULATION_MED_LARGE)

-- 夏季帽（防暑）
inst:AddComponent("insulator")
inst.components.insulator:SetSummer()
inst.components.insulator:SetInsulation(TUNING.INSULATION_LARGE)

-- 也可直接赋值
inst.components.insulator.insulation = TUNING.INSULATION_SMALL
inst.components.insulator.type = SEASONS.WINTER
```

#### INSULATION 常量速查

```2242:2245:scripts/tuning.lua
        INSULATION_SMALL = seg_time*2,
        INSULATION_MED = seg_time*4,
        INSULATION_MED_LARGE = seg_time*6,
        INSULATION_LARGE = seg_time*8,
```

| 常量 | 值 | 体感效果 |
|---|---|---|
| `INSULATION_SMALL` | 60 | 减缓 67% 温变速率 |
| `INSULATION_MED` | 120 | 减缓 80% 温变速率 |
| `INSULATION_MED_LARGE` | 180 | 减缓 86% 温变速率 |
| `INSULATION_LARGE` | 240 | 减缓 89% 温变速率 |

---

### 18.4.10（小结）速查表 + mod 实战 3 模板

#### Temperature 组件 API 速查

| 函数/字段 | 参数 | 说明 |
|---|---|---|
| `SetTemperature(value)` | number | 设置体温（触发状态事件）|
| `SetTemp(temp)` | number/nil | 强制锁定体温（nil 解除）|
| `DoDelta(delta)` | number | 立即改变体温（考虑保温）|
| `GetCurrent()` | 无 | 返回当前体温 |
| `GetMax()` | 无 | 返回 maxtemp |
| `IsFreezing()` | 无 | current < 0 |
| `IsOverheating()` | 无 | current > overheattemp |
| `GetInsulation()` | 无 | 返回 (winterInsul, summerInsul) |
| `SetModifier(name, value)` | string, number | 添加环境温度偏移 |
| `RemoveModifier(name)` | string | 移除温度偏移 |
| `SetTemperatureInBelly(delta, dur)` | number, number | 吃食物的短暂体温效果 |
| `SetFreezingHurtRate(rate)` | number | 冻结扣血速率 |
| `SetOverheatHurtRate(rate)` | number | 过热扣血速率 |

#### 关键字段速查

| 字段 | 类型 | 说明 |
|---|---|---|
| `temperature.current` | number | 当前体温 |
| `temperature.overheattemp` | number | 过热临界点（默认70）|
| `temperature.hurtrate` | number | 冻结/过热扣血速率 |
| `temperature.overheathurtrate` | number | 过热专用扣血速率 |
| `temperature.inherentinsulation` | number | 固有冬季保温值 |
| `temperature.inherentsummerinsulation` | number | 固有夏季保温值 |
| `temperature.maxmoisturepenalty` | number | 满湿时温度感知降低量（默认30°）|
| `temperature.shelterinsulation` | number | 避雨时额外夏季保温 |

#### 3 个 mod 常用模板

##### 模板 A：设置角色基础耐温属性

```lua
local function master_postinit(inst)
    -- 更耐冷但不耐热（如毛茸茸的角色）
    inst.components.temperature.inherentinsulation = TUNING.INSULATION_MED   -- 冬季保温
    inst.components.temperature.inherentsummerinsulation = 0                  -- 夏季不保温

    -- 过热比正常早（更高临界点才过热 → 改小 overheattemp = 更容易过热）
    -- 此处维持默认 70，不改

    -- 修改冻结扣血速率（更耐冻）
    inst.components.temperature:SetFreezingHurtRate(TUNING.WILSON_HEALTH / 200)  -- 200s
end
```

##### 模板 B：给 mod 道具添加保温和加热功能

```lua
-- 一个"暖石灯"：携带时持续保温，放置时给附近人加热
-- 携带/装备加热
inst:AddComponent("heater")
inst.components.heater.carriedheat = 40        -- 携带时目标温度 40°
inst.components.heater.carriedheatmultiplier = 0.8

-- 放置在世界中加热
inst.components.heater.heat = 70               -- 世界加热目标 70°

-- 装备可提供保温
inst:AddComponent("insulator")
inst.components.insulator:SetWinter()
inst.components.insulator:SetInsulation(TUNING.INSULATION_SMALL)
```

##### 模板 C：监听温度状态触发角色特殊反应

```lua
inst:ListenForEvent("startfreezing", function(inst)
    -- 开始冻结时激活"抗寒"技能形态
    inst:AddTag("frost_form")
    inst.components.combat.defaultdamage = inst.components.combat.defaultdamage * 1.2
end)

inst:ListenForEvent("stopfreezing", function(inst)
    inst:RemoveTag("frost_form")
    inst.components.combat.defaultdamage = inst.components.combat.defaultdamage / 1.2
end)

inst:ListenForEvent("startoverheating", function(inst)
    -- 过热时激活"火焰护盾"
    inst.SoundEmitter:PlaySound("mymod/characters/mychar/overheat_start_LP", "overheat")
end)

inst:ListenForEvent("stopoverheating", function(inst)
    inst.SoundEmitter:KillSound("overheat")
end)
```

---


## 18.5 Moisture 组件——潮湿度、打滑与干燥

### 本节导读

> **一句话定位**：`Moisture` 是饥荒中**潮湿度（淋雨程度）**的管理器——它跟踪 entity 有多湿，`segs ≥ 2` 时标记为"湿了"（`wet=true`），产生温度感知惩罚、理智惩罚、装备打滑等副作用，并在不下雨且靠近热源时缓慢干燥——防雨装备通过降低`waterproofness`来减少淋湿速率。

#### 一段宏观先讲清楚：潮湿度如何变化

```
【每帧 OnUpdate(dt)】
    │
    ├── IsForceDry() → 直接跳过（强制干燥，不受任何影响）
    │
    └── 正常计算：
           │
           ├── 在睡袋里 → rate = -sleepingbagdryingrate（快速烘干）
           │
           └── 正常 rate 计算：
                   rate = moisturerate           ← 雨/海水/泼水
                        + equippedmoisturerate   ← 穿防水装备（负值=保护）
                        - dryingrate             ← 自然/热源干燥
                        + externalbonuses         ← 外部加/减速率
                        - desiccantBonus         ← 干燥剂吸湿
                     │
                     ▼
                   DoDelta(rate * dt)
                     │
                     ├── moisture += delta，clamp [0, maxmoisture(100)]
                     ├── wet = (segs >= 2)
                     └── PushEvent("moisturedelta", {old, new})
```

**"湿了"的三重副作用**：

```
wet = true（segs ≥ 2，moisture ≥ 40/100）
   │
   ├── 体温感受惩罚：temperature.GetMoisturePenalty() 最多 -30°（更容易冷/减缓过热）
   │
   ├── 理智惩罚：sanity.Recalc 中的 moisture_delta（easing.inSine 曲线）
   │
   ├── 装备打滑：equippable 组件的装备等效 dapperness 降低（部分）
   │
   └── 工具打滑：使用工具/武器时有几率滑手掉落（取决于 moisture 量）
```

#### 18.5 节回答的 5 个核心问题

```
Q1: ──── moisture 和 "wet" 状态有什么区别？
         ↓ 答：moisture 是 0~100 的数值；wet = segs ≥ 2（即 moisture ≥ 40/100）

Q2: ──── 防雨伞/雨帽如何减少淋雨？waterproofness 是如何计算的？
         ↓ 答：waterproofer 组件 + waterproofnessmodifiers，淋雨速率 × (1 - waterproofness)

Q3: ──── 自然干燥是怎么工作的？靠近篝火干得更快的原因？
         ↓ 答：GetDryingRate：heater power + 体温 + moisture 量三重因素

Q4: ──── 如何完全免疫淋雨？
         ↓ 答：moisture:ForceDry(true, source) 或 inst:AddComponent("rainimmunity")

Q5: ──── 如何在代码中判断玩家是否"湿了"（用于触发效果）？
         ↓ 答：inst:GetIsWet()（C++ 快捷函数）或 inst.replica.moisture:IsWet()
```

#### 你将看到的核心源码

| 文件 | 行 | 用途 |
|---|---|---|
| `scripts/components/moisture.lua` | 全文 397 行 | Moisture 组件主体 |
| `scripts/components/moisture_replica.lua` | 全文 15 行 | 客户端湿度状态只读接口 |
| `scripts/components/waterproofer.lua` | 全文 30 行 | 防水组件 |
| `scripts/tuning.lua` | 2907-2910/2316 | 关键常量 |

#### 本节学习路径

```
18.5.1 新手 ─── moisture 与 wet —— 两个概念的关系
18.5.2 新手 ─── DoDelta / moisturedelta 事件 / AnnounceMoisture
18.5.3 新手 ─── 湿了的三重副作用（温度/理智/打滑）
                  ↓
18.5.4 进阶 ─── OnUpdate：完整淋湿/干燥计算流程
18.5.5 进阶 ─── waterproofness 防水系数 —— 装备防水原理
18.5.6 进阶 ─── GetDryingRate —— 自然干燥的三重因素
                  ↓
18.5.7 老手 ─── ForceDry —— 强制免疫淋雨
18.5.8 老手 ─── waterproofnessmodifiers + externalbonuses
18.5.9 老手 ─── WaterProofer 组件 + 联机同步机制
18.5.10     ─── 小结·速查表 + mod 实战 3 模板
```

---

### 18.5.1（新手）moisture 与 wet —— 两个概念的关系

#### 两个核心概念

| 概念 | 类型 | 范围 | 含义 |
|---|---|---|---|
| `moisture` | number | 0 ~ 100 | 当前潮湿度（连续数值）|
| `wet` | bool | true/false | 是否处于"湿了"状态 |

**`wet` 的判定条件**：

```142:142:scripts/components/moisture.lua
    self.wet = newSegs >= 2
```

`wet = true` 当且仅当 `segs ≥ 2`。`segs` 是 UI 上的"水滴数"，由以下公式计算：

```201:209:scripts/components/moisture.lua
function Moisture:GetSegs()
    local num = self.moisture / self.maxmoisture * self.numSegs
    local full = math.max(0, math.ceil(num - 1))

    --(num): real number of drops (aka. segs)
    --(full): whole number of full drops for UI
    --(num - full): alpha value of the currently filling drop
    return full, num - full
end
```

**公式**：`num = moisture / maxmoisture × numSegs = moisture / 100 × 5`

| moisture | num（实际水滴数）| full（完整水滴）| wet？|
|---|---|---|---|
| 0 | 0 | 0 | false |
| 20 | 1.0 | 0 | false（full=0，不足1个完整水滴）|
| 40 | 2.0 | 1 | false（full=ceil(2-1)=1，= 1 < 2）|
| 41 | 2.05 | 2 | **true**（full=ceil(1.05)=2 ≥ 2）|
| 60 | 3.0 | 2 | true |
| 80 | 4.0 | 3 | true |
| 100 | 5.0 | 4 | true |

> **关键阈值**：`moisture > 40`（严格大于）时 `wet = true`。

#### 各级潮湿度的台词

```118:130:scripts/components/moisture.lua
function Moisture:AnnounceMoisture(oldSegs, newSegs)
    if self.inst.components.talker then
        if oldSegs < 1 and newSegs >= 1 then
            self.inst.components.talker:Say(GetString(self.inst, "ANNOUNCE_DAMP"))   -- 略湿（segs达到1）
        elseif oldSegs < 2 and newSegs >= 2 then
            self.inst.components.talker:Say(GetString(self.inst, "ANNOUNCE_WET"))    -- 湿了（segs达到2）
        elseif oldSegs < 3 and newSegs >= 3 then
            self.inst.components.talker:Say(GetString(self.inst, "ANNOUNCE_WETTER")) -- 更湿（segs达到3）
        elseif oldSegs < 4 and newSegs >= 4 then
            self.inst.components.talker:Say(GetString(self.inst, "ANNOUNCE_SOAKED")) -- 浑身湿透（segs达到4）
        end
    end
end
```

---

### 18.5.2（新手）DoDelta / moisturedelta 事件 / 关键 API

#### DoDelta —— 直接增加/减少潮湿度

```132:147:scripts/components/moisture.lua
function Moisture:DoDelta(num, no_announce)
	if self:IsForceDry() then
        return
    end

    local oldLevel = self.moisture
    local oldSegs = self:GetSegs()
    self.moisture = math.clamp(self.moisture + num, 0, self.maxmoisture)
    local newSegs = self:GetSegs()
    local delta = self.moisture - oldLevel
    self.wet = newSegs >= 2
	if not no_announce then
	    self:AnnounceMoisture(oldSegs, newSegs)
	end
    self.inst:PushEvent("moisturedelta", { old = oldLevel, new = self.moisture })
end
```

**参数**：
- `num`：潮湿度变化量（正数=变湿，负数=变干）
- `no_announce`：`true` = 不触发台词播报

**注意**：`IsForceDry()` 时直接返回——强制干燥状态下无法变湿。

#### moisturedelta 事件

每次潮湿度变化都触发：

| 键名 | 含义 |
|---|---|
| `old` | 变化前 moisture 值 |
| `new` | 变化后 moisture 值 |

#### 常用 API 速查

| 函数 | 参数 | 返回 | 说明 |
|---|---|---|---|
| `DoDelta(num, no_announce)` | number, bool | — | 增减潮湿度 |
| `SetMoistureLevel(num)` | number | — | 直接设置（不触发台词）|
| `SetPercent(per)` | 0~1 | — | 设为 maxmoisture 的 per 倍 |
| `GetMoisture()` | 无 | number | 当前潮湿度值 |
| `GetMoisturePercent()` | 无 | 0~1 | 当前潮湿百分比 |
| `GetMaxMoisture()` | 无 | number | 最大潮湿度（默认 100）|
| `IsWet()` | 无 | bool | segs >= 2 |
| `IsForceDry()` | 无 | bool | 是否强制干燥 |
| `GetSegs()` | 无 | number, number | (完整水滴数, 当前水滴填充度) |

---

### 18.5.3（新手）湿了的三重副作用

#### 副作用 1：体温感受惩罚（最多 -30°）

这部分在 18.4 节已经讲到——`temperature.GetMoisturePenalty()` 在 Recalc 中减去体感温度：

```lua
-- 满湿时，感受温度 = 真实温度 - 30°
-- 夏天 35° 的沙漠感觉只有 5°，极大降低过热可能性
-- 冬天 -5° 感觉 -35°，极大加速冻结
MOISTURE_TEMP_PENALTY = 30
```

**实际意义**：雨天穿防雨伞不只是"少变湿"，更是避免了体感温度骤降带来的快速冻结。

#### 副作用 2：理智惩罚

`sanity.Recalc` 中：

```lua
local moisture_delta = easing.inSine(moisture, 0, MOISTURE_SANITY_PENALTY_MAX, maxMoisture)
```

`MOISTURE_SANITY_PENALTY_MAX` 是满湿时的最大理智下降速率——潮湿度越高，理智下降越快（`inSine` 曲线，非线性加速）。

#### 副作用 3：工具打滑

在 `player_common.lua` 中，玩家使用工具时有几率打滑（滑手掉落工具）：

```375:375:scripts/prefabs/player_common.lua
	if tool and tool:GetIsWet() and not tool:HasTag("stickygrip") and TryLuckRoll(...)
```

湿了的工具更容易从手中滑落。防止打滑：给工具添加 `"stickygrip"` 标签。

#### 另外：雨天不能灭火

湿身玩家或雨天降低了火焰有效性，`WET_HEAT_FACTOR_PENALTY = 0.75`——湿了时，加热器对你的加热效果降低 25%。

---

### 18.5.4（进阶）OnUpdate：完整淋湿/干燥计算流程

```338:372:scripts/components/moisture.lua
function Moisture:OnUpdate(dt)
	if self:IsForceDry() then
        return
    end

    local sleepingbagdryingrate = self:GetSleepingBagDryingRate()
    if sleepingbagdryingrate ~= nil then
        self.rate = -sleepingbagdryingrate
    else
        local moisturerate = self:GetMoistureRate()
        local dryingrate = self:GetDryingRate(moisturerate)
        local equippedmoisturerate = self:GetEquippedMoistureRate(dryingrate)
        local externalbonuses = self:GetRateBonus()

        self.rate = moisturerate + equippedmoisturerate - dryingrate + externalbonuses
    end

    if self.moisture > 0 or self.rate > 0 then
        local drate = self:GetDesiccantBonus(self.rate, dt)
        self.rate = self.rate - drate
    end
    -- ratescale 更新
    self:DoDelta(self.rate * dt, self:IsInBathingPool())
end
```

**四个速率来源**：

| 变量 | 计算函数 | 含义 |
|---|---|---|
| `moisturerate` | `GetMoistureRate()` | 当前淋湿速率（正值）|
| `equippedmoisturerate` | `GetEquippedMoistureRate()` | 装备带来的额外湿度变化 |
| `dryingrate` | `GetDryingRate()` | 自然干燥速率（正值）|
| `externalbonuses` | `externalbonuses:Get()` | 外部额外加成 |
| `desiccantBonus` | `GetDesiccantBonus()` | 干燥剂额外吸湿 |

**净速率公式**：

```
rate = moisturerate + equippedmoisturerate - dryingrate + externalbonuses - desiccantBonus
```

若 rate > 0：在变湿；rate < 0：在干燥；rate = 0：稳定。

#### GetMoistureRate —— 淋湿速率

```253:263:scripts/components/moisture.lua
function Moisture:GetMoistureRate()
	if self.inst.components.inventory and
		self.inst.components.inventory:IsFloaterHeld() or
		self:IsInBathingPool()
	then
		return self.maxMoistureRate
	elseif not TheWorld.state.israining then
        return 0
    end

    return self:_GetMoistureRateAssumingRain()
end
```

**没有下雨时（不在水里）**：`moisturerate = 0`，不会淋湿。

**下雨时**：

```212:235:scripts/components/moisture.lua
function Moisture:_GetMoistureRateAssumingRain()
	if self.inst.components.rainimmunity ~= nil then
		return 0
	end

    local waterproofmult = sheltered_waterproofness + inventory_waterproofness + inherentWaterproofness + waterproofnessmodifiers

    if waterproofmult >= 1 then
        return 0
    end

    local rate = easing.inSine(TheWorld.state.precipitationrate, self.minMoistureRate, self.maxMoistureRate, 1)
    return rate * (1 - waterproofmult)
end
```

**淋湿速率公式**：

```
moisturerate = easing.inSine(precipitationrate, 0, maxMoistureRate(0.75), 1) × (1 - waterproofmult)
```

- `precipitationrate` 越大（倾盆大雨）→ 淋湿越快
- `waterproofmult` 越大（防水性越强）→ 淋湿越慢
- `waterproofmult ≥ 1` → 完全防水，`moisturerate = 0`

---

### 18.5.5（进阶）waterproofness 防水系数 —— 装备防水原理

#### waterproofness 的四个来源

```239:250:scripts/components/moisture.lua
function Moisture:GetWaterproofness()
    local waterproofness =
        (   self.inst.components.inventory ~= nil and
            self.inst.components.inventory:GetWaterproofness() or 0
        ) +
        (   self.inherentWaterproofness or 0
        ) +
        (
            self.waterproofnessmodifiers:Get() or 0
        )
    
    return math.clamp(waterproofness, 0, 1)
end
```

| 来源 | 字段/函数 | 说明 |
|---|---|---|
| 装备栏防水 | `inventory:GetWaterproofness()` | 装备带 `waterproofer` 组件的物品 |
| 角色固有防水 | `inherentWaterproofness` | 角色本身的防水属性（如 Wurt 水生物）|
| 防水修饰器 | `waterproofnessmodifiers:Get()` | 多来源叠加的防水加成 |
| 遮蔽防水 | `sheltered.waterproofness` | 在遮蔽物/屋檐下 |

**`waterproofness = 1.0`**：完全防水（如穿雨衣，下雨不淋湿）

**`waterproofness = 0.5`**：一半防水（淋湿速率减半）

#### WaterProofer 组件 —— 给物品添加防水

```lua
-- 在物品 prefab 中添加防水组件
inst:AddComponent("waterproofer")
inst.components.waterproofer:SetEffectiveness(1.0)  -- 完全防水（如雨衣）
-- 或部分防水
inst.components.waterproofer:SetEffectiveness(0.5)  -- 50% 防水
```

`waterproofer` 的效果通过 `inventory:GetWaterproofness()` 汇总所有装备的防水值后，被 `GetWaterproofness()` 使用。

#### 通过 waterproofnessmodifiers 动态添加防水

```lua
-- 某 buff 提供额外防水（如涂油后）
inst.components.moisture.waterproofnessmodifiers:SetModifier(buff_source, 0.3, "oil_buff")

-- buff 结束
inst.components.moisture.waterproofnessmodifiers:RemoveModifier(buff_source, "oil_buff")
```

---

### 18.5.6（进阶）GetDryingRate —— 自然干燥的三重因素

```287:301:scripts/components/moisture.lua
function Moisture:GetDryingRate(moisturerate)
    -- Don't dry if it's raining
    if (moisturerate or self:GetMoistureRate()) > 0 then
        return 0
    end

    local heaterPower = self.inst.components.temperature ~= nil and math.clamp(self.inst.components.temperature.externalheaterpower, 0, 1) or 0
    local playerTempDrying = self:GetSegs() < 3 and self.optimalPlayerTempDrying or self.maxPlayerTempDrying

    local rate = self.baseDryingRate
        + easing.linear(heaterPower, self.minPlayerTempDrying, playerTempDrying, 1)
        + easing.linear(GetLocalTemperature(self.inst), self.minDryingRate, self.maxDryingRate, self.optimalDryingTemp)
        + easing.inExpo(self:GetMoisture(), 0, 1, self.maxmoisture)

    return math.clamp(rate, 0, self.maxDryingRate + self.maxPlayerTempDrying)
end
```

**干燥不发生的条件**：正在下雨（`moisturerate > 0`）时，`dryingrate = 0`。

**干燥速率的三重因素**：

| 因素 | 来源 | 效果 |
|---|---|---|
| 火堆/加热器功率 | `temperature.externalheaterpower`（0~1）| 靠近火堆 → 更快干燥（最大 `playerTempDrying = 5`）|
| 环境温度 | `GetLocalTemperature()` | 温度越高越干得快（0 ~ `maxDryingRate=0.1`，最优 `optimalDryingTemp=50°`）|
| 湿度量 | `moisture` | 越湿 → 干燥越快（`inExpo` 曲线，0~1）|
| 睡袋 | `sleepingbag.dryingrate` | 睡袋有额外快速干燥速率 |

**解读**：
- 靠近大篝火（`heaterPower ≈ 1`）：`dryingrate ≈ 5`——大约 20 秒内从满湿（100）干透（100 ÷ 5 = 20）
- 不靠近火，在较热的天气（35°）：`dryingrate ≈ 0.07`——更慢（约 25 分钟）
- 完全浸湿时干得更快（`inExpo` 曲线）

---

### 18.5.7（老手）ForceDry —— 强制免疫淋雨

```74:97:scripts/components/moisture.lua
function Moisture:ForceDry(force, source)
	source = source or self.inst
    if force then
		if self.forcedrysources == nil then
            self.rate = 0
            self.ratescale = RATE_SCALE.NEUTRAL
            self:SetMoistureLevel(0)
            self.inst:StopUpdatingComponent(self)
			self.forcedrysources = { [source] = true }
		elseif not self.forcedrysources[source] then
			self.forcedrysources[source] = true
		end
		-- 若 source 是 entity，监听 onremove 自动清除
	elseif -- 从 forcedrysources 移除 source
    end
end
```

`ForceDry(true, source)` 效果：
1. **立即将 moisture 设为 0**
2. **停止 OnUpdate**（不再计算淋湿/干燥）
3. `DoDelta` 和 `SetMoistureLevel` 调用无效

**多来源管理**：只有所有 source 都调用 `ForceDry(false, source)` 后，才真正恢复湿润能力。

**使用场景**：

```lua
-- 进入特殊区域（如龙之怒地牢）：角色不受雨水影响
inst.components.moisture:ForceDry(true, special_zone)
-- 离开区域
inst.components.moisture:ForceDry(false, special_zone)

-- 变身为火焰形态：不受淋雨影响
inst.components.moisture:ForceDry(true, inst)
-- 恢复
inst.components.moisture:ForceDry(false, inst)
```

**自动回收**：若 `source` 是一个有效 entity，当它被移除时，ForceDry 自动解除（通过 `onremove` 监听）。

#### rainimmunity 组件 —— 轻量级雨水免疫

与 `ForceDry` 不同，`rainimmunity` 组件只阻止雨水淋湿（不强制干），但不影响手动 `DoDelta`：

```213:215:scripts/components/moisture.lua
	if self.inst.components.rainimmunity ~= nil then
		return 0
	end
```

```lua
-- 添加雨水免疫（下雨不会变湿，但可以手动 DoDelta）
inst:AddComponent("rainimmunity")

-- 移除
inst:RemoveComponent("rainimmunity")
```

---

### 18.5.8（老手）waterproofnessmodifiers + externalbonuses

#### waterproofnessmodifiers —— 叠加防水系数

`SourceModifierList`（additive 模式）——多个来源的防水系数相加：

```lua
-- 某装备/buff 提供额外防水
inst.components.moisture.waterproofnessmodifiers:SetModifier(source_inst, 0.5, "waterproof_buff")

-- 移除
inst.components.moisture.waterproofnessmodifiers:RemoveModifier(source_inst, "waterproof_buff")
```

总防水系数 = 固有防水 + 装备防水 + modifiers 之和，`clamp(0, 1)`。

#### externalbonuses / AddRateBonus —— 外部速率加成

```317:327:scripts/components/moisture.lua
function Moisture:AddRateBonus(src, bonus, key)
    self.externalbonuses:SetModifier(src, bonus, key)
end

function Moisture:RemoveRateBonus(src, key)
    self.externalbonuses:RemoveModifier(src, key)
end

function Moisture:GetRateBonus()
    return self.externalbonuses:Get()
end
```

`externalbonuses` 直接加到最终速率——可以做"快速变湿"或"快速干燥"的效果：

```lua
-- 某技能使角色快速变湿
inst.components.moisture:AddRateBonus(skill_inst, 2.0, "soak_skill")  -- rate +2/s

-- 某装备加速干燥
inst.components.moisture:AddRateBonus(equip_inst, -1.5, "quick_dry")  -- rate -1.5/s

-- 移除
inst.components.moisture:RemoveRateBonus(skill_inst, "soak_skill")
```

---

### 18.5.9（老手）WaterProofer 组件 + 联机同步机制

#### WaterProofer 组件详解

```1:30:scripts/components/waterproofer.lua
local WaterProofer = Class(function(self, inst)
    self.inst = inst
    self.effectiveness = 1

    inst:AddTag("waterproofer")

    if inst.components.inventoryitem ~= nil then
        inst.components.inventoryitem:EnableMoisture(false)
    end
end)
-- ...
function WaterProofer:GetEffectiveness()
    return self.effectiveness
end

function WaterProofer:SetEffectiveness(val)
    self.effectiveness = val
end
```

- `effectiveness`：此物品的防水系数（0~1），被 `inventory:GetWaterproofness()` 汇总
- 添加 `"waterproofer"` 标签
- **自动禁用物品自身的湿度**（`EnableMoisture(false)`）——防水伞不会自己被淋湿影响

#### 联机同步

`moisture_replica.lua` 极简，只同步 `wet`（是否湿了）的布尔状态：

```1:14:scripts/components/moisture_replica.lua
local Moisture = Class(function(self, inst)
    self.inst = inst
    self._iswet = net_bool(inst.GUID, "moisture._iswet")
end)

function Moisture:SetIsWet(iswet)
    self._iswet:set(iswet)
end

function Moisture:IsWet()
    return self._iswet:value()
end
```

**客户端只能知道"是否湿了"（`IsWet()`），不能知道精确的 `moisture` 数值**（具体数值通过 `player_classified` 同步，但需要对应的 classified 字段）。

**`inst:GetIsWet()`**（C++ 函数，全平台可用）：

```lua
-- 服务端（直接访问 replica）
local is_wet = inst.replica.moisture:IsWet()

-- C++ 快捷方式（两端均可用）
local is_wet = inst:GetIsWet()  -- 内部用 replica.moisture:IsWet()
```

---

### 18.5.10（小结）速查表 + mod 实战 3 模板

#### Moisture 组件 API 速查

| 函数/字段 | 参数 | 说明 |
|---|---|---|
| `DoDelta(num, no_announce)` | number, bool | 增减潮湿度 |
| `SetMoistureLevel(num)` | number | 直接设置潮湿度（不触发台词）|
| `SetPercent(per)` | 0~1 | 百分比设置 |
| `GetMoisture()` | 无 | 当前潮湿度值 |
| `GetMoisturePercent()` | 无 | 0~1 |
| `GetSegs()` | 无 | (完整水滴数, 填充进度) |
| `IsWet()` | 无 | segs >= 2 |
| `IsForceDry()` | 无 | 是否强制干燥 |
| `ForceDry(force, source)` | bool, key | 设置/解除强制干燥 |
| `GetWaterproofness()` | 无 | 当前总防水系数 (0~1) |
| `GetMoistureRate()` | 无 | 当前淋湿速率 |
| `GetDryingRate()` | 无 | 当前干燥速率 |
| `AddRateBonus(src, bonus, key)` | — | 添加外部速率加成 |
| `RemoveRateBonus(src, key)` | — | 移除外部速率加成 |

#### 关键字段速查

| 字段 | 类型 | 说明 |
|---|---|---|
| `moisture.moisture` | number | 当前潮湿度 (0~maxmoisture) |
| `moisture.maxmoisture` | number | 最大潮湿度（默认 100）|
| `moisture.wet` | bool | 是否"湿了"（segs>=2）|
| `moisture.inherentWaterproofness` | number | 角色固有防水（已废弃，用 modifiers）|
| `moisture.waterproofnessmodifiers` | SourceModifierList | 叠加防水系数 |
| `moisture.externalbonuses` | SourceModifierList | 叠加速率加成 |
| `moisture.maxDryingRate` | number | 最大干燥速率（默认 0.1）|
| `moisture.optimalDryingTemp` | number | 最优干燥温度（默认 50°）|

#### 3 个 mod 常用模板

##### 模板 A：给 mod 角色添加天然防水（水生角色）

```lua
local function master_postinit(inst)
    -- 水生角色：天然 50% 防水
    inst.components.moisture.waterproofnessmodifiers:SetModifier(inst, 0.5, "aquatic")
    -- 完全免疫雨水（只有直接 DoDelta 才能变湿）
    -- inst:AddComponent("rainimmunity")
end
```

##### 模板 B：某技能临时加速角色变湿（如被海浪冲刷）

```lua
local function OnWaveHit(inst)
    -- 立即增加 50 点潮湿度（忽略防水）
    inst.components.moisture:DoDelta(50, false)  -- 触发台词

    -- 或通过 ForceDry 临时解除防水效果
end
```

##### 模板 C：mod 装备添加防水功能

```lua
-- 某防水外套的 prefab
inst:AddComponent("equippable")
inst.components.equippable.equipslot = EQUIPSLOTS.BODY

-- 添加防水组件
inst:AddComponent("waterproofer")
inst.components.waterproofer:SetEffectiveness(0.80)  -- 80% 防水（比雨衣稍差）

-- 可选：同时提供夏季保温（防水外套）
inst:AddComponent("insulator")
inst.components.insulator:SetSummer()
inst.components.insulator:SetInsulation(TUNING.INSULATION_SMALL)
```

---


## 18.6 防护组件联动：Waterproofer、Insulator、Heater、RainDome

### 本节导读

> **一句话定位**：饥荒的"防护系统"不是单一组件，而是六个组件的协同网络——`Waterproofer` 防淋湿、`Insulator` 减缓温变、`Heater` 主动加热/冷却、`RainDome` 给整个范围提供雨幕保护、`Sheltered` 检测遮蔽状态、`RainImmunity` 免疫雨水——前五节学了"被保护者"（Moisture/Temperature），本节聚焦"提供保护的装备/物体"如何工作。

#### 六个防护组件速览

```
┌─────────────────────────────────────────────────────────────────────┐
│  防护体系                                                             │
│                                                                      │
│  Waterproofer（装备/物品）                                            │
│    ↓ 提供 effectiveness（0~1）                                        │
│    ↓ inventory:GetWaterproofness() 汇总                               │
│    ↓ 降低 moisture 组件的淋湿速率                                     │
│                                                                      │
│  Insulator（装备）                                                    │
│    ↓ 提供 insulation 值（WINTER / SUMMER 类型）                       │
│    ↓ temperature:GetInsulation() 汇总                                 │
│    ↓ 降低温变速率（SEG_TIME / (SEG_TIME + insulation)）               │
│                                                                      │
│  Heater（世界/装备/携带物）                                           │
│    ↓ 提供 heat（目标温度）                                             │
│    ↓ 放热：把周围 entity 体温推向 heat（最大范围 10 格衰减）           │
│    ↓ 吸热：把周围 entity 体温推向 heat（降温）                        │
│                                                                      │
│  Sheltered（地形/树木）                                               │
│    ↓ 树荫/屋檐下 → 玩家 sheltered=true                               │
│    ↓ temperature：夏季保温 + INSULATION_MED_LARGE                    │
│    ↓ moisture：waterproofness（由 sheltered.waterproofness 提供）     │
│                                                                      │
│  RainImmunity（临时添加到 entity 上）                                 │
│    ↓ 完全阻止雨水淋湿（moisture:_GetMoistureRateAssumingRain 返回 0） │
│                                                                      │
│  RainDome（世界实体，如灵魂伞）                                       │
│    ↓ 以特定半径保护范围内所有 entity                                  │
│    ↓ 给范围内 entity 添加 rainimmunity（多来源管理）                  │
└─────────────────────────────────────────────────────────────────────┘
```

#### 本节学习路径

```
18.6.1 新手 ─── Waterproofer 防水 —— 装备防水原理与 effectiveness
18.6.2 新手 ─── Insulator 保温 —— 冬季/夏季保温与 INSULATION 值
18.6.3 进阶 ─── Heater 加热/冷却 —— 放热/吸热、距离衰减、携带/装备模式
18.6.4 进阶 ─── Sheltered 遮蔽 —— 树荫/屋檐检测与双重防护
18.6.5 进阶 ─── RainImmunity 雨水免疫 —— 多来源管理
18.6.6 老手 ─── RainDome 穹顶保护 —— 大范围雨幕保护区
18.6.7 老手 ─── 六大组件联动关系图
18.6.8     ─── 小结·速查表 + mod 实战 3 模板
```

---

### 18.6.1（新手）Waterproofer 防水 —— 装备防水原理

**Waterproofer** 是一个轻量级组件，负责给物品赋予"防水能力"。

#### 组件结构

```1:30:scripts/components/waterproofer.lua
local WaterProofer = Class(function(self, inst)
    self.inst = inst
    self.effectiveness = 1

    --V2C: Recommended to explicitly add tag to prefab pristine state
    inst:AddTag("waterproofer")

    if inst.components.inventoryitem ~= nil then
        inst.components.inventoryitem:EnableMoisture(false)
    end
end)

function WaterProofer:GetEffectiveness()
    return self.effectiveness
end

function WaterProofer:SetEffectiveness(val)
    self.effectiveness = val
end
```

| 字段/函数 | 说明 |
|---|---|
| `effectiveness` | 防水系数（0~1），由 `inventory:GetWaterproofness()` 汇总 |
| `"waterproofer"` 标签 | 识别标识 |
| `EnableMoisture(false)` | 自动禁止物品自身被淋湿 |

#### WATERPROOFNESS 常量速查

```2927:2932:scripts/tuning.lua
        WATERPROOFNESS_SMALL = 0.2,
        WATERPROOFNESS_SMALLMED = 0.35,
        WATERPROOFNESS_MED = 0.5,
        WATERPROOFNESS_LARGE = 0.7,
        WATERPROOFNESS_HUGE = 0.9,
        WATERPROOFNESS_ABSOLUTE = 1,
```

| 常量 | 值 | 含义 |
|---|---|---|
| `WATERPROOFNESS_SMALL` | 0.2 | 20% 防水（花圈等装饰品）|
| `WATERPROOFNESS_MED` | 0.5 | 50% 防水（轻便雨衣）|
| `WATERPROOFNESS_LARGE` | 0.7 | 70% 防水（普通雨帽/雨伞）|
| `WATERPROOFNESS_HUGE` | 0.9 | 90% 防水（高级防水装备）|
| `WATERPROOFNESS_ABSOLUTE` | 1 | 完全防水（void cloth 伞）|

#### 在 prefab 中添加防水能力

```lua
-- 雨帽（70% 防水）
inst:AddComponent("waterproofer")
inst.components.waterproofer:SetEffectiveness(TUNING.WATERPROOFNESS_LARGE)

-- 雨伞（完全防水）
inst:AddComponent("waterproofer")
inst.components.waterproofer:SetEffectiveness(TUNING.WATERPROOFNESS_ABSOLUTE)

-- 魔法雨衣（完全防水 + 夏季保温）
inst:AddComponent("waterproofer")
inst.components.waterproofer:SetEffectiveness(TUNING.WATERPROOFNESS_HUGE)
inst:AddComponent("insulator")
inst.components.insulator:SetSummer()
inst.components.insulator:SetInsulation(TUNING.INSULATION_MED)
```

**防水叠加规则**：多件防水物品的 `effectiveness` 由 `inventory:GetWaterproofness()` 叠加——叠加方式取决于 inventory 的 GetWaterproofness 实现（通常是加法，但 `clamp(0, 1)`）。

---

### 18.6.2（新手）Insulator 保温 —— 冬季/夏季保温

**Insulator** 给物品提供保温值，被 `temperature:GetInsulation()` 汇总后减缓体温变化速率。

#### 组件结构

```1:31:scripts/components/insulator.lua
local Insulator = Class(function(self, inst)
    self.inst = inst
    self.insulation = 0
    self.type = SEASONS.WINTER
end)

function Insulator:SetSummer()
	self.type = SEASONS.SUMMER
end

function Insulator:SetWinter()
	self.type = SEASONS.WINTER
end

function Insulator:SetInsulation(val)
	self.insulation = val
end

function Insulator:GetInsulation()
	return self.insulation, self:GetType()
end
```

#### 两种类型：冬季 vs 夏季

| 类型 | 常量 | 生效时机 | 效果 |
|---|---|---|---|
| `SEASONS.WINTER` | （默认）| 冬天（ambient_temp < STARTING_TEMP）时抵抗降温 | 降低"我在变冷"速率 |
| `SEASONS.SUMMER` | `SetSummer()` 设置 | 夏天（ambient_temp ≥ STARTING_TEMP）时抵抗升温 | 降低"我在变热"速率 |

**一件物品只能是一种类型**——要同时保冬天和夏天，需要两个 Insulator 实例（但装备只有一个组件槽，通常用 `temperature.inherentinsulation` 或两件分开的装备解决）。

#### 实用速查：各装备保温值

| 保温值常量 | 数值 | 降温速率倍数 |
|---|---|---|
| `INSULATION_SMALL` | 60 | × 0.33（降低 67%）|
| `INSULATION_MED` | 120 | × 0.20（降低 80%）|
| `INSULATION_MED_LARGE` | 180 | × 0.14（降低 86%）|
| `INSULATION_LARGE` | 240 | × 0.11（降低 89%）|

#### 在 prefab 中添加保温

```lua
-- 冬帽（大保温）
inst:AddComponent("insulator")
inst.components.insulator:SetWinter()
inst.components.insulator:SetInsulation(TUNING.INSULATION_LARGE)

-- 防晒伞（夏季保温）
inst:AddComponent("insulator")
inst.components.insulator:SetSummer()
inst.components.insulator:SetInsulation(TUNING.INSULATION_MED_LARGE)
```

#### 特殊情况：胡须保温（Wilson）

Wilson 的胡须特殊：给冬季保温 `+X`，同时给夏季保温 `-X`（越长的胡须，夏天越热）：

```248:252:scripts/components/temperature.lua
    if self.inst.components.beard ~= nil then
        --Beards help winterInsulation but hurt summerInsulation
        winterInsulation = winterInsulation + self.inst.components.beard:GetInsulation()
        summerInsulation = summerInsulation - self.inst.components.beard:GetInsulation()
    end
```

---

### 18.6.3（进阶）Heater 加热/冷却 —— 主动温度控制

**Heater** 是 6 个组件中最灵活的——它允许 entity 主动影响附近/装备者/携带者的体温。

#### 三种工作场景

| 场景 | 相关字段 | 触发时机 |
|---|---|---|
| **世界加热器**（如火堆）| `heat` / `heatfn` | temperature.OnUpdate：搜索周围 10 格 |
| **装备加热器**（如热石头装备）| `equippedheat` / `equippedheatfn` | temperature.OnUpdate：遍历装备槽 |
| **携带加热器**（如热石头在背包）| `carriedheat` / `carriedheatmultiplier` | temperature.OnUpdate：遍历物品槽 |

#### 放热 vs 吸热

```24:27:scripts/components/heater.lua
function Heater:SetThermics(exo, endo)
	self.exothermic = exo
	self.endothermic = endo
end
```

| 模式 | `exo` | `endo` | 含义 |
|---|---|---|---|
| **放热**（加热）| true | false | 将体温推向 `heat`（加热）|
| **吸热**（冷却）| false | true | 将体温推向 `heat`（冷却）|
| **禁用**（两者关闭）| false | false | 不影响体温（虽然有组件）|

**放热计算**（temperature.OnUpdate）：

```lua
-- 目标加热温度（距离衰减后）
local warmingtemp = heat * heatfactor  -- heatfactor ∈ [0,1]，10格时=0，0格时=1
-- 如果目标温度 > 当前体温，推动升温
if warmingtemp > current then
    delta += warmingtemp - current
end
```

**吸热计算**：

```lua
-- 目标冷却温度（相对于 overheattemp）
local coolingtemp = (heat - overheattemp) * heatfactor + overheattemp
-- 如果目标温度 < 当前体温，推动降温
if coolingtemp < current then
    delta += coolingtemp - current
end
```

#### 湿身惩罚

在雨中（`inst:GetIsWet()`），加热效果打 `WET_HEAT_FACTOR_PENALTY = 0.75` 折：

```lua
if self.inst:GetIsWet() then
    if heat > 0 then          -- 加热时打折（效果减弱）
        heatfactor = heatfactor * 0.75
    else                       -- 冷却时增强（湿了更快冷）
        heatfactor = heatfactor / 0.75
    end
end
```

#### 距离衰减与范围限制

```37:51:scripts/components/heater.lua
function Heater:SetShouldFalloff(should_falloff)
    self.stop_falloff = not should_falloff
end

function Heater:ShouldFalloff()
    return not self.stop_falloff
end

function Heater:SetHeatRadiusCutoff(radius_cutoff)
    self.radius_cutoff = radius_cutoff
end
```

- 默认：有距离衰减（10格=0，0格=1，线性衰减）
- `SetShouldFalloff(false)`：关闭距离衰减（均匀全范围）
- `SetHeatRadiusCutoff(n)`：超过 n 格完全无效

#### 实战：制作一个 mod 发热炉

```lua
local function fn()
    local inst = CreateEntity()
    -- ...entity setup...
    
    inst:AddComponent("heater")
    inst.components.heater.heat = 90          -- 目标温度 90°（接近过热）
    -- exothermic=true, endothermic=false（默认放热）
    -- 默认 10 格范围，有距离衰减
    
    -- 想要更大范围但效果均匀（如地热）：
    -- inst.components.heater.stop_falloff = true
    -- inst.components.heater:SetHeatRadiusCutoff(15)  -- 15 格内有效
    
    return inst
end
```

---

### 18.6.4（进阶）Sheltered 遮蔽 —— 树荫/屋檐检测

**Sheltered** 是一个玩家专用组件，检测玩家是否在遮蔽物下（树冠、屋檐等），并提供保护效果。

#### 组件工作原理

```63:82:scripts/components/sheltered.lua
local SHELTERED_MUST_TAGS = { "shelter" }
local SHELTERED_CANT_TAGS = { "FX", "NOCLICK", "DECOR", "INLIMBO", "stump", "burnt" }
function Sheltered:OnUpdate(dt)
    -- ...
    if self.inst.canopytrees and self.inst.canopytrees > 0 then
        sheltered = true
        level = 2
    else
        local x, y, z = self.inst.Transform:GetWorldPosition()
        local num_sheltered = TheSim:CountEntities(x, y, z, 2, SHELTERED_MUST_TAGS, SHELTERED_CANT_TAGS)
        sheltered = num_sheltered > 0
    end

    self:SetSheltered(sheltered, level)
end
```

**两种遮蔽来源**：
1. **树冠**（`canopytrees > 0`）：玩家周围 2 格内有树冠（level=2）
2. **`"shelter"` 标签 entity**：在 2 格范围内有带 `"shelter"` 标签的 entity（level=1）

哪些 entity 有 `"shelter"` 标签？

- 大树（常绿树、落叶树等）：当树足够大时添加此标签
- 石头树等特殊植被

#### 遮蔽的双重保护效果

**温度保护**（temperature.GetInsulation）：

```254:256:scripts/components/temperature.lua
    if self.sheltered then
        summerInsulation = summerInsulation + self.shelterinsulation
    end
```

遮蔽时额外获得 `shelterinsulation = TUNING.INSULATION_MED_LARGE = 180` 的夏季保温——夏天在树荫下不会那么热。

**防雨保护**（moisture._GetMoistureRateAssumingRain）：

```216:221:scripts/components/moisture.lua
    local waterproofmult =
        (   self.inst.components.sheltered ~= nil and
            self.inst.components.sheltered.sheltered and
            self.inst.components.sheltered.waterproofness or 0
        ) +
        -- ...
```

遮蔽时提供 `sheltered.waterproofness = TUNING.WATERPROOFNESS_SMALLMED = 0.35` 的防雨效果。

**level 2（树冠）的额外效果**：

```327:329:scripts/components/temperature.lua
        if self.sheltered_level > 1 then
            ambient_temperature = math.min(ambient_temperature,  self.overheattemp - 5)
        end
```

level 2 时环境温度被限制在 `OVERHEAT_TEMP - 5 = 65°`——深树林里永远不会太热。

#### 触发的事件

```47:59:scripts/components/sheltered.lua
        if self.sheltered then
            self.sheltered = false
            self.inst:PushEvent("sheltered", { sheltered=false, level=self.sheltered_level })
        end
    -- ...
        self.sheltered = true
        self.inst:PushEvent("sheltered", { sheltered=true, level=self.sheltered_level })
```

| 事件 | 数据 | 含义 |
|---|---|---|
| `"sheltered"` | `{sheltered: bool, level: int}` | 进入/离开遮蔽物 |

**让一个 mod entity 提供遮蔽（仿树荫）**：

```lua
-- 让 mod 建筑提供遮蔽效果（半径 2 格内的玩家认为自己在遮蔽下）
inst:AddTag("shelter")
```

---

### 18.6.5（进阶）RainImmunity 雨水免疫 —— 多来源管理

**RainImmunity** 是一个临时组件，当有任意来源添加时存在，所有来源移除时自动删除组件。

```1:50:scripts/components/rainimmunity.lua
local RainImmunity = Class(function(self, inst)
	self.inst = inst
	self.sources = {}

	inst:AddTag("rainimmunity")
	-- 构造时监听 onremove 并推送 gainrainimmunity 事件
	inst:PushEvent("gainrainimmunity")
end)

function RainImmunity:AddSource(src)
	if not self.sources[src] then
		self.sources[src] = true
		-- 若 src 是 entity，监听其 onremove 自动清理
	end
end

function RainImmunity:RemoveSource(src)
	-- 移除来源；若 sources 为空，inst:RemoveComponent("rainimmunity")
end
```

#### 两种使用场景

**场景 1：手动为 entity 添加永久雨水免疫**

```lua
-- entity 完全免疫雨水（如水生角色）
inst:AddComponent("rainimmunity")
-- 此时 sources 为空 {}，但组件存在且有效
-- moisture._GetMoistureRateAssumingRain() 检查：若 rainimmunity 组件存在 → return 0
```

**场景 2：RainDome 临时为范围内 entity 添加**

RainDome 在 OnUpdate 中自动管理：
- entity 进入范围 → `target.components.rainimmunity:AddSource(dome_inst)`
- entity 离开范围 → `target.components.rainimmunity:RemoveSource(dome_inst)`
- 当没有任何来源时，`rainimmunity` 组件自动删除

**相关事件**：

| 事件 | 触发时机 |
|---|---|
| `"gainrainimmunity"` | RainImmunity 组件被添加到 entity 时 |
| `"loserainimmunity"` | RainImmunity 组件被从 entity 移除时 |

---

### 18.6.6（老手）RainDome 穹顶保护 —— 大范围雨幕保护

**RainDome** 是饥荒中"灵魂伞"等雨幕穹顶的核心组件——它在一定半径内持续为所有 entity 提供雨水免疫。

#### 基本工作原理

```80:216:scripts/components/raindome.lua
-- 服务端字段
self.radius = 16          -- 默认保护半径（格）
self.enabled = false      -- 默认禁用

-- 客户端/服务端共享
self._activeradius = net_float(...)  -- 通过网络同步的活跃半径
```

```132:144:scripts/components/raindome.lua
function RainDome:Enable()
	if self.ismastersim and not self.enabled then
		self.enabled = true
		self:SetActiveRadius_Internal(self.radius, 0)
	end
end

function RainDome:Disable()
	if self.ismastersim and self.enabled then
		self.enabled = false
		self:SetActiveRadius_Internal(0, self.radius)
	end
end
```

#### OnUpdate：持续管理范围内的 entity

```183:216:scripts/components/raindome.lua
function RainDome:OnUpdate(dt)
	-- ...
	for _, target in ipairs(TheSim:FindEntities(x, y, z, self.radius, TAGS, NOTAGS)) do
		if oldtargets[target] then
			oldtargets[target] = nil  -- 已在范围内，继续
		else
			-- 新进入范围的 entity
			if not target.components.rainimmunity then
				target:AddComponent("rainimmunity")
			end
			target.components.rainimmunity:AddSource(self.inst)
		end
		-- ...
	end
	-- 处理离开范围的 entity（移除 rainimmunity source）
	for tgt in pairs(oldtargets) do
		if tgt.components.rainimmunity ~= nil and tgt:IsValid() then
			tgt.components.rainimmunity:RemoveSource(self.inst)
		end
	end
	self.delay = awake and 1 or 3  -- 有玩家活跃时 1s 更新，否则 3s
end
```

**关键设计**：
- RainDome 每 1-3 秒更新一次（不是每帧）
- 当 RainDome 被禁用或销毁时，自动移除所有 entity 的 rainimmunity source
- 支持多个 RainDome 同时存在（全局的 `_maxsize`/`_sizes` 缓存最大半径用于优化搜索）

#### 全局查询函数

```29:59:scripts/components/raindome.lua
function GetRainDomesAtXZ(x, z)
	-- 返回坐标 (x, z) 周围所有活跃的 RainDome 列表
end

function IsUnderRainDomeAtXZ(x, z)
	-- 快速判断 (x, z) 是否在任何 RainDome 保护下
end
```

**使用场景**：`rain.lua` / `caverain.lua` 用 `GetRainDomesAtXZ` 判断雨水是否落在某坐标（被 dome 保护的地方不应有雨水特效）。

#### 在 mod 中使用 RainDome

```lua
-- 创建一个便携雨幕穹顶（类似 voidcloth_umbrella）
local function fn()
    local inst = CreateEntity()
    -- entity 必须在客户端和服务端都添加 raindome（因为有 net 变量）
    inst:AddComponent("raindome")
    
    if not TheWorld.ismastersim then
        return inst
    end
    
    -- 配置（服务端）
    inst.components.raindome:SetRadius(8)    -- 8 格保护半径
    -- 默认 disabled，需要手动启用
    
    return inst
end

-- 激活
inst.components.raindome:Enable()
-- 停用
inst.components.raindome:Disable()
```

**注意**：`RainDome.OnRemoveFromEntity` 直接 `assert(false)` —— 组件一旦添加**不能被移除**，只能通过 `Disable()` 停用，或删除整个 entity。

---

### 18.6.7（老手）六大组件联动关系图

```
┌──────────────────────────────────────────────────────────────────────────┐
│  玩家 entity（被保护方）                                                   │
│                                                                            │
│  moisture 组件                      temperature 组件                       │
│    ↑ 淋湿速率                          ↑ 体温变化速率                       │
│    │                                   │                                   │
│    │ waterproofness（0~1）              │ insulation（冬/夏）               │
│    │     │                             │     │                             │
│    │     ├── Waterproofer（装备）       │     ├── Insulator（装备）         │
│    │     ├── moisture.waterproof       │     ├── temperature.inherent*     │
│    │     │   nessmodifiers             │     ├── beard（Wilson 专有）       │
│    │     └── sheltered.waterproofness  │     ├── sheltered（树荫/屋檐）     │
│    │         （来自 Sheltered 组件）    │     └── dusk/night 时段加成       │
│    │                                   │                                   │
│    ├── RainImmunity（临时组件）         ├── Heater（世界/装备/携带）        │
│    │   → 完全阻断 moisturerate = 0     │   → 修改 delta，推动体温变化       │
│    │   ↑ 由 RainDome 的 OnUpdate 添加  │                                   │
│    │   ↑ 由 ForceDry 的 rate=0 替代    │                                   │
│    │                                   │                                   │
│    └── Sheltered 组件（玩家专用）       └── Sheltered 组件（同上）          │
│        → 周围 2 格内有 "shelter" 标签                                       │
│        → 提供 waterproofness + summerInsulation + 限制 ambientTemp         │
└──────────────────────────────────────────────────────────────────────────┘
```

**总结：防护效果叠加路径**

```
角色不被淋湿（moisture 不上升）的两条路：
  A: waterproofness ≥ 1  → moisturerate × (1 - 1) = 0  [装备叠加到 100%]
  B: rainimmunity 存在    → _GetMoistureRateAssumingRain() 直接返回 0

角色不受冷/热（temperature 变化很慢）的路：
  insulation 足够大     → rate ≈ 0（温变速率极低）
  Heater 存在          → delta 被推向目标温度（对抗环境温度差）
  settemp ≠ nil        → temperature 被锁定（完全忽略所有计算）
```

---

### 18.6.8（小结）速查表 + mod 实战 3 模板

#### 六大组件速查

| 组件 | 主要字段/函数 | 被哪里消费 |
|---|---|---|
| `WaterProofer` | `effectiveness (0~1)` | `moisture._GetMoistureRateAssumingRain` |
| `Insulator` | `insulation`, `type (WINTER/SUMMER)` | `temperature.GetInsulation()` |
| `Heater` | `heat`, `exothermic`, `endothermic`, `heatfn` | `temperature.OnUpdate` |
| `Sheltered` | `sheltered`, `waterproofness`, `sheltered_level` | `moisture` + `temperature` |
| `RainImmunity` | `sources` (多来源管理) | `moisture._GetMoistureRateAssumingRain` |
| `RainDome` | `radius`, `enabled`, `_activeradius` (net) | 自动管理范围内 entity 的 RainImmunity |

#### WATERPROOFNESS / INSULATION 常量速查

| 常量 | 类型 | 值 | 
|---|---|---|
| `WATERPROOFNESS_SMALL` | 防水 | 0.2 |
| `WATERPROOFNESS_SMALLMED` | 防水 | 0.35 |
| `WATERPROOFNESS_MED` | 防水 | 0.5 |
| `WATERPROOFNESS_LARGE` | 防水 | 0.7 |
| `WATERPROOFNESS_HUGE` | 防水 | 0.9 |
| `WATERPROOFNESS_ABSOLUTE` | 防水 | 1.0 |
| `INSULATION_SMALL` | 保温 | 60 |
| `INSULATION_MED` | 保温 | 120 |
| `INSULATION_MED_LARGE` | 保温 | 180 |
| `INSULATION_LARGE` | 保温 | 240 |

#### 3 个 mod 常用模板

##### 模板 A：制作一件综合防护装备（防水+保温）

```lua
-- 神奇外套：80% 防水 + 夏季中级保温 + 冬季小级保温
local function fn()
    local inst = CreateEntity()
    -- ...entity setup...
    
    inst:AddComponent("equippable")
    inst.components.equippable.equipslot = EQUIPSLOTS.BODY

    -- 防水
    inst:AddComponent("waterproofer")
    inst.components.waterproofer:SetEffectiveness(TUNING.WATERPROOFNESS_HUGE)

    -- 夏季保温
    inst:AddComponent("insulator")
    inst.components.insulator:SetSummer()
    inst.components.insulator:SetInsulation(TUNING.INSULATION_MED)
    
    -- 注意：一件装备只能有一个 insulator，若需要同时有冬季保温：
    -- 可使用 temperature.inherentinsulation（在角色 prefab 中设）
    -- 或在装备的 OnEquip/OnUnequip 中动态修改角色 temperature 字段
    
    return inst
end
```

##### 模板 B：制作一个 mod 篝火（会加热的世界 entity）

```lua
local function fn()
    local inst = CreateEntity()
    -- ...
    
    -- 发热模块
    inst:AddComponent("heater")
    inst.components.heater.heat = 90           -- 目标加热温度
    -- exothermic = true（默认），endothermic = false（默认）
    -- 默认 10 格内有距离衰减
    
    -- 若想提供遮蔽（树荫效果）：
    inst:AddTag("shelter")
    
    return inst
end
```

##### 模板 C：制作一个小型 RainDome 物品（携带时保护周围人）

```lua
-- 一个放置型穹顶道具
local function fn()
    local inst = CreateEntity()
    -- ...
    
    -- raindome 需要在客户端和服务端都添加
    inst:AddComponent("raindome")
    
    if not TheWorld.ismastersim then
        return inst
    end
    
    -- 服务端配置
    inst.components.raindome:SetRadius(6)  -- 6 格保护半径
    
    -- 只在满足条件时启用（如需要放置激活）
    -- inst.components.raindome:Enable()   -- 启动时调用
    
    -- 激活/停用时机
    inst:ListenForEvent("activate", function(inst)
        inst.components.raindome:Enable()
    end)
    
    return inst
end
```

---


## 18.7 三维状态的联动设计（饥饿影响血量、理智影响战斗等）

### 本节导读

> **一句话定位**：饥荒的五大生存状态（Health/Hunger/Sanity/Temperature/Moisture）并不是相互隔离的——它们形成了一张**互相影响的网**：饥饿会杀死你、冻死也会、黑暗会同时打击血量和理智、湿了会加快冻死和降低理智……本节从"设计视角"梳理这些联动关系，帮助 mod 开发者理解和复现类似机制。

#### 五大状态的联动全景图

```
                         ┌──────────────────────┐
                         │   5 大生存状态        │
                         └──────────────────────┘
                                   │
    ┌──────────────────────────────┼───────────────────────────────┐
    │                              │                               │
    ▼                              ▼                               ▼
┌────────┐               ┌──────────────┐               ┌──────────────┐
│ Hunger │               │  Temperature │               │   Moisture   │
│ (饥饿) │               │  (体温)      │               │   (潮湿)     │
└────────┘               └──────────────┘               └──────────────┘
    │                          │    │                        │    │
    │ current<=0               │    │ GetMoisturePenalty     │    │ moisture_delta
    │ DoDelta(-hurtrate,"hunger")│  │ -30° at full wet       │    │ (sanity.Recalc)
    ▼                          │    ▼                        │    ▼
┌────────┐                     │ ┌────────┐                  │ ┌────────┐
│ Health │ ◄────────────────── │ │ Health │                  │ │ Sanity │
│ (血量) │   DoDelta(-hurtrate │ │ (血量) │                  │ │ (理智) │
│        │   ,"cold"/"hot")    │ └────────┘                  │ └────────┘
└────────┘                     │   current<0 or >overheat   │
    │                          └────────────────────────────┘
    │ health<=0
    ▼
  死亡

┌────────────────────────────────────────────────────────────────┐
│ Grue 黑暗攻击：同时 DoDelta(-damage, "darkness") + sanity.DoDelta(-SANITY_MEDLARGE)
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Sanity → Combat：                                               │
│   IsCrazy() → 暗影生物/夜魇生物变成"对当前玩家hostile"         │
│   IsInsane() → 暗影战斗！                                      │
└────────────────────────────────────────────────────────────────┘
```

#### 18.7 节回答的 6 个核心问题

```
Q1: ──── 饥饿如何导致死亡？中间经历了哪些步骤？
         ↓ 答：hunger.DoDec → current=0 → startstarving → health:DoDelta(-1.25/s, "hunger")

Q2: ──── 非致死保护（nonlethal）是什么？何时生效？
         ↓ 答：health < 20% 时，hunger/temperature 伤害不再生效（默认关闭）

Q3: ──── 理智如何影响战斗？
         ↓ 答：IsCrazy() → 暗影生物 HostileToPlayerTest 返回 true（变成攻击者）

Q4: ──── 黑暗（Grue）如何同时影响血量和理智？
         ↓ 答：Grue.Attack() 扣血 + sanity.DoDelta(-SANITY_MEDLARGE)

Q5: ──── 潮湿是怎么同时影响体温和理智的？
         ↓ 答：temperature.GetMoisturePenalty（-30°） + sanity.Recalc moisture_delta

Q6: ──── 复活别人有什么联动效果？
         ↓ 答：被救者 +25% health penalty；救人者 sanity +80
```

#### 本节学习路径

```
18.7.1 新手 ─── 饥饿 → 血量：饿死机制
18.7.2 新手 ─── 体温 → 血量：冻死/热死机制  
18.7.3 进阶 ─── 非致死保护 —— nonlethal 机制详解
18.7.4 进阶 ─── 理智 → 战斗：暗影生物的仇恨逻辑
18.7.5 进阶 ─── 黑暗 → 血量+理智：Grue 组件
18.7.6 进阶 ─── 潮湿 → 理智+体温双重惩罚
                  ↓
18.7.7 老手 ─── 复活与血量/理智的互动设计
18.7.8 老手 ─── 行为系统与状态的双向联动（建造/恐慌逃跑）
18.7.9 老手 ─── 角色专属联动（Wolfgang/Wanda/Wendy 等）
18.7.10     ─── 小结·完整联动矩阵 + mod 实战 3 模板
```

---

### 18.7.1（新手）饥饿 → 血量：饿死机制

这是最直接的状态联动——饱食度归零后，每秒扣血，最终死亡。

**触发路径**：

```
hunger.DoDec(dt) 每秒调用
    │
    ├── current > 0：扣饱食度（正常）
    └── current <= 0：
            ├── overridestarvefn 存在 → 调用自定义逻辑
            └── 否则：
                  inst.components.health:DoDelta(-self.hurtrate * dt, true, "hunger")
```

**相关 TUNING 值**：

```
WILSON_HUNGER = 150          ← 最大饱食度
WILSON_HUNGER_RATE = 75/480  ≈ 0.156/s ← 正常消耗速率
STARVE_KILL_TIME = 120       ← 挨饿 120 秒后死亡
hurtrate = 150/120 = 1.25/s  ← 每秒扣 1.25 血
```

**关键设计**：`cause = "hunger"` 传入 `health:DoDelta`——这允许 `health.nonlethal_hunger` 机制拦截（见 18.7.3）。

**监听挨饿事件**（mod 实用）：

```lua
inst:ListenForEvent("startstarving", function(inst)
    inst.components.talker:Say(GetString(inst, "ANNOUNCE_HUNGRY"))
end)
```

---

### 18.7.2（新手）体温 → 血量：冻死/热死机制

体温超出安全范围时，直接调用 health:DoDelta：

```472:478:scripts/components/temperature.lua
    if applyhealthdelta ~= false and self.inst.components.health ~= nil then
        if self.current < 0 then
            self.inst.components.health:DoDelta(-self.hurtrate * dt, true, "cold")
        elseif self.current > self.overheattemp then
            self.inst.components.health:DoDelta(-(self.overheathurtrate or self.hurtrate) * dt, true, "hot")
        end
    end
```

| 状态 | cause | 扣血条件 |
|---|---|---|
| 冻结 | `"cold"` | `temperature.current < 0` |
| 过热 | `"hot"` | `temperature.current > overheattemp (70°)` |

**两个 cause 的意义**：
- `health.nonlethal_temperature = false`（默认）：冷热都能致命
- `health.nonlethal_temperature = true`（友好模式/特殊角色）：低血量时免疫冷热伤害

**默认扣血速率（冻/热相同）**：

```
hurtrate = WILSON_HEALTH / FREEZING_KILL_TIME = 150 / 120 = 1.25/s
```

也就是说，**饿死、冻死、热死的默认速率都是 120 秒**——游戏设计上故意保持一致，让玩家有相同的反应时间。

---

### 18.7.3（进阶）非致死保护 —— nonlethal 机制详解

饥荒有一个"非致死（nonlethal）"系统——某些伤害来源在血量极低时会停止造成伤害，防止玩家被某个特定来源杀死（但其他来源仍然可以）。

#### 触发条件

```616:621:scripts/components/health.lua
    if old_percent <= self.nonlethal_pct and
		(((cause == "cold" or cause == "hot") and self.nonlethal_temperature) or
		(cause == "hunger" and self.nonlethal_hunger)) then

        return 0
    end
```

**三个字段**：

| 字段 | 默认值 | 含义 |
|---|---|---|
| `health.nonlethal_pct` | `NONLETHAL_PERCENT = 0.2` | 触发非致死保护的血量阈值（20%）|
| `health.nonlethal_hunger` | `NONLETHAL_HUNGER = false` | 饥饿伤害是否非致死（默认关，即饥饿能饿死）|
| `health.nonlethal_temperature` | `NONLETHAL_TEMPERATURE = false` | 温度伤害是否非致死（默认关）|

**TUNING 默认值**：

```6823:6827:scripts/tuning.lua
        HEALTH_PENALTY_ENABLED = true,
        NONLETHAL_TEMPERATURE = false,
        NONLETHAL_HUNGER = false,
        NONLETHAL_DARKNESS = false,
        NONLETHAL_PERCENT = 0.2,
```

**Grue（黑暗）的非致死保护**（独立于 health 组件，在 grue 组件中实现）：

```160:178:scripts/components/grue.lua
function Grue:Attack()
    local damage = TUNING.GRUEDAMAGE

    if self.nonlethal then
        local health = self.inst.components.health
        if health then
            local currenthealth = health.currenthealth
            local maxhealth = health.maxhealth

            local damagepercent = (currenthealth - damage) / maxhealth
            if damagepercent <= self.nonlethal_pct then
                local minhealth = maxhealth * self.nonlethal_pct
                damage = math.max(0, currenthealth - minhealth)
            end
        end
    end

    self.inst.components.combat:GetAttacked(nil, damage, nil, "darkness")
end
```

Grue 的非致死保护逻辑：若此次攻击会使血量降到 `nonlethal_pct(20%)` 以下，则将伤害削减到"恰好降到 20%"。

**开启非致死保护（友好模式/特殊角色设计）**：

```lua
-- 让角色的饥饿/体温伤害不致死（但其他伤害仍然可以）
inst.components.health.nonlethal_hunger = true
inst.components.health.nonlethal_temperature = true
inst.components.health.nonlethal_pct = 0.1  -- 可自定义阈值（10%）

-- 黑暗攻击非致死
inst.components.grue.nonlethal = true
```

---

### 18.7.4（进阶）理智 → 战斗：暗影生物的仇恨逻辑

#### 暗影生物的 HostileToPlayerTest

当玩家理智降到阈值以下进入"疯狂"状态（`IsCrazy()` = `IsInsane()` 的别名），暗影生物（shadow creature / nightmare creature）就会对该玩家变成敌对状态：

```170:182:scripts/prefabs/shadowcreature.lua
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
	return false
end
```

**`IsCrazy()`**（被废弃的名称，实际等同于 `IsInsane()`）：

```lua
-- sanity_replica 中
function Sanity:IsCrazy()
    -- 等价于：mode == INSANITY AND (not sane or inducedinsanity) AND not inducedlunacy
    return not self._issane:value() and self._isinsanitymode:value()
end
```

**触发链**：

```
sanity.DoDelta →  percent < SANITY_BECOME_INSANE_THRESH(15%)
  → sane = false
  → PushEvent("goinsane")
  → replica.sanity:SetIsSane(false)
  → 客户端 _issane:set(false)
  → 暗影生物 CLIENT_ShadowSubmissive_HostileToPlayerTest → IsCrazy() = true
  → 暗影生物变为该玩家的敌人！
```

#### DynamicMusic 的联动

理智进入疯狂时，动态音乐系统会切换到战斗/危险 BGM：

```lua
-- dynamicmusic.lua 中，"goinsane" 事件触发
-- → StartDanger("danger") 或播放疯狂主题音乐
```

#### 玩家也判断是否有暗影危险

`fns.IsNearDanger` 中：

```43:43:scripts/prefabs/player_common.lua
                    not (inst.components.sanity:IsSane() and target:HasTag("shadowcreature")))
```

只有当玩家**已疯狂**时，周围的 `"shadowcreature"` 才被计入"危险状态"——这会影响 DynamicMusic 的 danger 音乐触发。

---

### 18.7.5（进阶）黑暗 → 血量+理智：Grue 组件

`Grue` 是饥荒中"黑暗怪"的实现——玩家在黑暗中停留超过 5-10 秒后，Grue 开始攻击。

**Grue 的双重伤害**：

```196:212:scripts/components/grue.lua
        if self.level > (self.resistance or 0) then
            self:Attack()   -- ← 扣血（health:DoDelta，damage = wilson_health*0.667 = 100）
            self.inst.components.sanity:DoDelta(-TUNING.SANITY_MEDLARGE)  -- ← 扣理智 -20
            self.inst:PushEvent("attackedbygrue")
        else
            self.inst:PushEvent("resistedgrue")
        end
```

**同时伤害血量和理智**，是最直接的跨状态联动：

| 伤害类型 | 数值 |
|---|---|
| 血量伤害 | `GRUEDAMAGE = wilson_health * 0.667 ≈ 100` |
| 理智伤害 | `SANITY_MEDLARGE = 20` |

**免疫条件**（任意一个）：
- 在光照下（`IsInLight()`）
- 拥有夜视（`playervision:HasNightVision()`）
- 处于无敌状态（`health:IsInvincible()`）

**非致死设计（NONLETHAL_DARKNESS）**：

```71:72:scripts/components/grue.lua
    self.nonlethal = TUNING.NONLETHAL_DARKNESS
    self.nonlethal_pct = TUNING.NONLETHAL_PERCENT
```

默认 `NONLETHAL_DARKNESS = false`——黑暗可以杀死你；但可以通过设置 `grue.nonlethal = true` 让黑暗只把血量打到 20% 为止。

**设计意义**：黑暗攻击同时威胁血量和理智，造成恐慌——玩家需要一手保健康（逃离或点火）、一手维持理智（避免暗影生物出现）——这是饥荒"多重压力"设计哲学的体现。

---

### 18.7.6（进阶）潮湿 → 理智+体温双重惩罚

潮湿是联动效果最广的状态——它**同时影响理智和体温**：

#### 潮湿 → 理智下降（sanity.Recalc）

```506:506:scripts/components/sanity.lua
    local moisture_delta = (self.inst.components.moisture == nil or self.no_moisture_penalty) and 0 or easing.inSine(self.inst.components.moisture:GetMoisture(), 0, TUNING.MOISTURE_SANITY_PENALTY_MAX, self.inst.components.moisture:GetMaxMoisture())
```

`MOISTURE_SANITY_PENALTY_MAX` 是满湿时的最大理智下降速率——`inSine` 曲线使轻微湿润影响很小，但湿透时影响显著。

#### 潮湿 → 体温降低（temperature.GetMoisturePenalty）

```269:271:scripts/components/temperature.lua
function Temperature:GetMoisturePenalty()
    return self.inst.components.moisture ~= nil and -Lerp(0, self.maxmoisturepenalty, self.inst.components.moisture:GetMoisturePercent()) or 0
end
```

`maxmoisturepenalty = MOISTURE_TEMP_PENALTY = 30°`——满湿时感受温度降低 30°。

**叠加效果（雨夜场景）**：

```
下雨夜晚：
  体温惩罚：-30°（湿）+ 夜晚快速降温 → 极速冻结
  理智惩罚：湿度惩罚 + 夜晚黑暗 → 快速失智
  → 不防水/不保温就是双重死亡威胁
```

---

### 18.7.7（老手）复活与血量/理智的互动设计

复活机制涉及血量惩罚和理智奖励的双向设计：

#### 复活者：理智奖励

```360:360:scripts/prefabs/player_common.lua
        giver.components.sanity:DoDelta(TUNING.REVIVE_OTHER_SANITY_BONUS)
```

**`REVIVE_OTHER_SANITY_BONUS = 80`**——救人者获得 80 点理智奖励。

**设计意义**：鼓励联机合作——救队友不仅道德上"正确"，游戏机制上也会让你理智更好。

#### 被救者：血量上限惩罚

```358:358:scripts/prefabs/player_common.lua
            inst.components.health:DeltaPenalty(TUNING.REVIVE_HEALTH_PENALTY)
```

**各种复活方式的血量惩罚**：

| 复活方式 | 血量惩罚 | 理智惩罚 |
|---|---|---|
| 被队友救活 | `+25%`（REVIVE_HEALTH_PENALTY）| - |
| 传送门复活 | `+25%`（PORTAL_HEALTH_PENALTY）| `REVIVE_SHADOW_SANITY_PENALTY = -40` |
| 心脏复活 | `+12.5%`（HEART_HEALTH_PENALTY）| - |
| 肉体稻草人 | 减少惩罚 | - |

**传送门复活的双重代价**：

```lua
-- REVIVE_SHADOW_SANITY_PENALTY = -40（扣理智）
-- PORTAL_HEALTH_PENALTY = 0.25（血量上限 -25%）
```

传送门复活最"廉价"但代价最重——会影响理智（可能触发暗影生物）和永久降低血量上限。

---

### 18.7.8（老手）行为系统与状态的双向联动

#### 建造 → 饥饿

```699:699:scripts/components/builder.lua
                self.inst.components.hunger:DoDelta(TUNING.HUNGRY_BUILDER_DELTA)
```

快速连续建造时，每次建造消耗 `HUNGRY_BUILDER_DELTA = -5` 点饱食度（有冷却，防止短时间内重复扣）。

**触发条件**（builder.lua 695 行附近）：
- 新的建造位置与上次不同（移动了至少 4 格）
- 或距上次建造超过 60 秒（`HUNGRY_BUILDER_RESET_TIME = seg_time * 2`）

**设计意义**：防止玩家脱离"生存"语境——即使专注建造，也需要保持饥饿度管理。

#### 血量 → 恐慌逃跑（combat.panic_thresh）

`combat.panic_thresh` 字段：当血量百分比低于此值时，`replica.combat:SetIsPanic(true)`——AI 用此判断是否进入逃跑模式。

```20:21:scripts/components/combat.lua
local function onpanicthresh(self, panicthresh)
    self.inst.replica.combat:SetIsPanic(panicthresh ~= nil and self.inst.components.health ~= nil and panicthresh > self.inst.components.health:GetPercent())
end
```

**在 health 变化时自动触发重算**（health.lua 第 4-6 行）：

```3:6:scripts/components/health.lua
local function onpercent(self)
    if self.inst.components.combat ~= nil then
        self.inst.components.combat.panic_thresh = self.inst.components.combat.panic_thresh
    end
end
```

每次血量变化，`onpercent` 触发 SourceField 的 dirty 信号，重新判断是否进入恐慌。

**典型应用（兔人）**：

```lua
inst.components.combat.panic_thresh = TUNING.BUNNYMAN_PANIC_THRESH
-- 兔人血量低于阈值时开始逃跑
```

---

### 18.7.9（老手）角色专属联动设计案例

饥荒的各角色通过重写默认联动来实现独特机制：

#### Wolfgang：力量状态与伤害/饥饿联动

- **体力满格（Mighty）**：攻击伤害×2，饥饿消耗更快
- **体力为零（Wimpy）**：攻击伤害减弱，饥饿消耗减少
- 通过修改 `combat.damagemultiplier` 和 `hunger.burnratemodifiers` 实现

#### Wanda：血量即年龄的联动设计

- `health.redirect = redirect_to_oldager`：所有血量变化重路由到年龄系统
- `health.canheal = false`：阻止普通治疗（年龄不能被"治疗"）
- `health.disable_penalty = true`：禁用死亡血量惩罚

#### Wendy：血量损伤分摊给 Abigail

- `health.redirect = redirect_to_abigail`：部分或全部伤害转到 Abigail
- 理智方面：Abigail 的 sanityaura 提供正面效果

#### WX-78：潮湿造成额外伤害（而非单纯理智/体温惩罚）

```lua
-- 机器人特有：湿了时 health:DoDelta（特殊联动）
TUNING.WX78_MIN_MOISTURE_DAMAGE = -0.60      -- 湿了每秒至少扣 0.6 血
TUNING.WX78_PERCENT_MOISTURE_DAMAGE = -1.2   -- 还有按百分比扣血
```

---

### 18.7.10（小结）完整联动矩阵 + mod 实战 3 模板

#### 五大状态联动矩阵

| 影响来源 → | Health（血量）| Hunger（饥饿）| Sanity（理智）| Temperature（体温）| Moisture（潮湿）|
|---|---|---|---|---|---|
| **Health 低** | — | ×（无直接联动）| ×（无直接联动）| `nonlethal_temperature` 保护 | ×（无直接联动）|
| **Hunger 归零** | `-hurtrate/s`（`"hunger"`）| — | ×（无直接联动）| ×（无直接联动）| ×（无直接联动）|
| **Sanity 低** | ×（暗影直接攻击）| ×（无直接联动）| — | ×（无直接联动）| ×（无直接联动）|
| **Temperature 极值** | `-hurtrate/s`（`"cold"`/`"hot"`）| ×（无直接联动）| ×（无直接联动）| — | ×（无直接联动）|
| **Moisture 高** | ×（间接：温度↓→体温↓→冻死）| ×（无直接联动）| `moisture_delta`（↓）| `GetMoisturePenalty` -30° | — |
| **黑暗（Grue）** | `-100` 每次 | ×（无直接联动）| `-SANITY_MEDLARGE=-20` | ×（无直接联动）| ×（无直接联动）|
| **复活救人** | ×（无变化）| ×（无直接联动）| `+REVIVE_OTHER_SANITY_BONUS=+80` | ×（无直接联动）| ×（无直接联动）|
| **被复活** | `+25% penalty`（惩罚）| ×（无直接联动）| （传送门-40）| ×（无直接联动）| ×（无直接联动）|
| **建造** | ×（无直接联动）| `-HUNGRY_BUILDER_DELTA=-5` | ×（无直接联动）| ×（无直接联动）| ×（无直接联动）|

#### 关键 TUNING 速查

| 常量 | 值 | 联动含义 |
|---|---|---|
| `STARVE_KILL_TIME` | 120s | 饥饿 → 死亡时间 |
| `FREEZING_KILL_TIME` | 120s | 冻结 → 死亡时间 |
| `NONLETHAL_PERCENT` | 0.2 | 非致死保护的血量阈值 |
| `GRUEDAMAGE` | 100（约）| 黑暗每次攻击的血量伤害 |
| `SANITY_MEDLARGE` | 20 | 黑暗每次攻击的理智伤害 |
| `MOISTURE_TEMP_PENALTY` | 30° | 满湿时的体温感知降低 |
| `REVIVE_HEALTH_PENALTY` | 0.25 | 复活时血量上限惩罚 |
| `REVIVE_OTHER_SANITY_BONUS` | 80 | 救人时的理智奖励 |
| `HUNGRY_BUILDER_DELTA` | -5 | 建造时消耗的饱食度 |

#### 3 个 mod 联动设计模板

##### 模板 A：创建"压力越大越强"的机制（低血量增加攻击）

```lua
-- 血量越低，攻击力越高（绝境逢生机制）
inst:ListenForEvent("healthdelta", function(inst, data)
    local health_pct = data.newpercent
    -- 血量 < 30% 时，伤害翻倍
    if health_pct < 0.3 then
        inst.components.combat.damagemultiplier = 2.0
        if not inst:HasTag("desperation") then
            inst:AddTag("desperation")
            inst.components.talker:Say(GetString(inst, "ANNOUNCE_LOWHEALTH") or "Now you've done it!")
        end
    else
        inst.components.combat.damagemultiplier = 1.0
        inst:RemoveTag("desperation")
    end
end)
```

##### 模板 B：创建"潮湿能量"机制（湿了增加魔法效果）

```lua
-- 潮湿时特殊技能冷却更快（水属性角色）
inst:ListenForEvent("moisturedelta", function(inst, data)
    local moisture_pct = data.new / inst.components.moisture:GetMaxMoisture()
    if moisture_pct > 0.5 then
        -- 潮湿时进入"水元素激活"状态
        inst.components.moisture.waterproofnessmodifiers:SetModifier(inst, 0.5, "water_element")
        -- 理智不再受潮湿惩罚（水属性适应）
        inst.components.sanity.no_moisture_penalty = true
    else
        inst.components.moisture.waterproofnessmodifiers:RemoveModifier(inst, "water_element")
        inst.components.sanity.no_moisture_penalty = false
    end
end)
```

##### 模板 C：创建"理智-伤害"联动（失智时伤害更高但自身也更脆）

```lua
-- 进入疯狂时：攻击力提升，但防御降低
inst:ListenForEvent("goinsane", function(inst)
    inst.components.combat.damagemultiplier = 1.5
    -- 进入疯狂形态的吸收变差
    inst.components.health.externalabsorbmodifiers:SetModifier(inst, -0.2, "insane_penalty")
    inst.components.talker:Say(GetString(inst, "ANNOUNCE_INSANE") or "They'll all pay!")
end)

inst:ListenForEvent("gosane", function(inst)
    inst.components.combat.damagemultiplier = 1.0
    inst.components.health.externalabsorbmodifiers:RemoveModifier(inst, "insane_penalty")
end)
```

---

