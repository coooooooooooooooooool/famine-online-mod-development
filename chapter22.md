# 第22章 Buff/Debuff 与法术系统

## 22.1 Debuff / Debuffable 组件——增减益效果的挂载与管理

> 涉及源码文件：
> - `scripts/components/debuff.lua`（Debuff 组件本体）
> - `scripts/components/debuffable.lua`（Debuffable 组件本体）
> - `scripts/entityscript.lua` 第 2099–2135 行（实体快捷方法）
> - `scripts/prefabs/player_common.lua` 第 2530–2531、2803–2807、1413 行（玩家挂载与回调示例）
> - `scripts/prefabs/healthregenbuff.lua`、`scripts/prefabs/hungerregenbuff.lua`、`scripts/prefabs/foodbuffs.lua`、`scripts/prefabs/spawnprotectionbuff.lua`、`scripts/prefabs/wx78_shadow_fuel_debuff.lua`、`scripts/prefabs/abigail.lua`（典型 buff 模板）

### 一、为什么需要这套组件？（所有读者）

在饥荒中，“吃了某种食物后一段时间内攻击力提升”“被怪物毒雾笼罩后持续掉血”“喝下灵药后获得一段时间的暗影视觉”——这些**“一段时间内附加效果”**几乎全部由 `debuff` / `debuffable` 这一对组件承担。

它的设计哲学相当反直觉：

- **效果本体是一个独立的 prefab 实体**（通常不可见、不持久化、客户端不可见）；
- 这个实体身上挂着 `debuff` 组件，定义“贴上去做什么”“扯下来做什么”“同名重复贴时做什么”。
- 被影响的目标（玩家、宠物、怪物、训练假人…）身上挂着 `debuffable` 组件，作为一本**“名字 → buff 实体”的小账本**。

> 名字虽然叫 “de”buff，但**没有任何代码上的差别**区分增益还是减益——`buff` 和 `debuff` 完全是同一套机制，命名只是历史遗留。本节里我们也会按习惯把“增益+减益”统称 buff，把组件名按官方写法保留为 `debuff`。

### 二、新手篇：5 分钟上手

#### 2.1 最小三步走

1. **挑一个已有的 buff prefab**（例如 `healthregenbuff`，吃果冻豆时会用到）。
2. 在你想给某个实体加效果的地方，调用：

```lua
target:AddDebuff("healthregenbuff", "healthregenbuff")
```

3. 想提前撤掉效果就调用：

```lua
target:RemoveDebuff("healthregenbuff")
```

就这么简单。`AddDebuff` 的第一个参数是“账本里的键名（你自己起，唯一即可）”，第二个参数是“要 `SpawnPrefab` 的那个 buff prefab”。

> 即使 `target` 上原本没有 `debuffable` 组件，`AddDebuff` 也会自动帮你 `AddComponent("debuffable")`（见 `entityscript.lua:2115-2128`），不必额外手动添加。

#### 2.2 一个完整的“吃果子回血”范例

官方 `preparedfoods.lua` 第 470–474 行就是这么写的：

```lua
prefabs = { "healthregenbuff" },
oneat_desc = STRINGS.UI.COOKBOOK.FOOD_EFFECTS_HEALTH_REGEN,
oneatenfn = function(inst, eater)
    eater:AddDebuff("healthregenbuff", "healthregenbuff")
end,
```

新手只要记住三点：
- **必须把 buff prefab 放进物品的 `prefabs` 列表**，否则世界生成时不会预加载该 prefab。
- **`AddDebuff` 必须在服务端调用**（也就是 `TheWorld.ismastersim` 为真时），客户端调用不会生效。
- buff 的“到期、回调、特效”全都由 buff prefab 内部自己管理，**调用者只负责挂上去**。

#### 2.3 新手最常见的 4 个坑

1. **键名乱起会冲突**。如果两个东西都用 `"buff"` 当 name，第二个会触发**刷新（Extend）**而不是叠加。
2. **不要把同一个 buff prefab 同时给一个人贴两次**——后一次会走 `OnExtended` 而非 `OnAttached`。
3. **客户端不要直接调** `AddDebuff`。所有 buff 都是服务器权威。
4. **不要在 buff prefab 里设置 `inst.entity:AddNetwork()`**（除非需要在屏幕上看到 fx）。绝大多数 buff 实体是“非网络化”的隐形服务端实体，详见进阶篇。

### 三、进阶篇：理解组件结构

> 这一节适合已经会写过简单 prefab、知道 EntityScript 生命周期的同学。我们把官方 `debuff.lua` 与 `debuffable.lua` 整个拆开看一遍。

#### 3.1 Debuff 组件（贴在 buff 实体身上）

源码 `scripts/components/debuff.lua` 一共只有 67 行，结构非常简单：

```lua
local Debuff = Class(function(self, inst)
    self.inst = inst
    self.name = nil                  -- 在 debuffable 账本里的键名（被挂上后才被赋值）
    self.target = nil                -- 当前贴的目标（被挂上后才被赋值）
    self.onattachedfn = nil          -- 贴上去时回调
    self.ondetachedfn = nil          -- 撕下来时回调
    self.onextendedfn = nil          -- 同名再贴时（“延长/刷新”）回调
    self.onchangefollowsymbolfn = nil-- 跟随骨骼符号变化时回调
    --self.keepondespawn = nil       -- 玩家“断线/离开”时是否要保留这条 buff
end)
```

它对外暴露 4 个 Setter 和 5 个核心方法：

| 方法 | 作用 |
| --- | --- |
| `SetAttachedFn(fn)` | 注册 `OnAttached` 回调 |
| `SetDetachedFn(fn)` | 注册 `OnDetached` 回调 |
| `SetExtendedFn(fn)` | 注册 `OnExtended` 回调（同名重新 Add） |
| `SetChangeFollowSymbolFn(fn)` | 跟随符号变更回调（罕用） |
| `Stop()` | **由 buff 实体自己调用**，等价于让目标的 `debuffable:RemoveDebuff(self.name)` |
| `AttachTo(name, target, followsymbol, followoffset, data, buffer)` | **由 debuffable 调用**，不要自己手动调 |
| `OnDetach()` | 同上，仅 debuffable 调用 |
| `Extend(followsymbol, followoffset, data, buffer)` | 同上，仅 debuffable 调用 |
| `ChangeFollowSymbol(followsymbol, followoffset)` | 同上，仅 debuffable 调用 |

回调函数的签名（参数顺序非常重要！）：

```lua
-- attached
onattachedfn(self.inst,  target, followsymbol, followoffset, data, buffer)
-- detached
ondetachedfn(self.inst,  target)
-- extended
onextendedfn(self.inst,  target, followsymbol, followoffset, data, buffer)
-- changefollowsymbol
onchangefollowsymbolfn(self.inst, target, followsymbol, followoffset)
```

> `self.inst` 就是 buff 实体本身（注意不是目标），`target` 才是被影响的人/怪。新手最容易把这两个搞反。

#### 3.2 Debuffable 组件（贴在目标身上）

源码 `scripts/components/debuffable.lua`：

##### 3.2.1 字段表

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `enable` | boolean | 是否允许新增 buff；置为 false 时会**清空已有 buff**（见 `Enable`） |
| `followsymbol` | string | 用于 buff 实体跟随挂点（视觉 buff 用） |
| `followoffset` | Vector3 | 跟随时的相对偏移 |
| `debuffs` | table | 账本本体，`{ [name] = { inst = buffEntity, onremove = fn } }` |
| `ondebuffadded` | function 可选 | `(self.inst, name, ent, data, buffer)`，挂载成功后回调 |
| `ondebuffremoved` | function 可选 | `(self.inst, name, debuffInst)`，移除时回调 |

注意 `enable` 字段通过 `Class` 的 `onset` 回调（构造器第三个参数）自动同步 `"debuffable"` 标签：

```lua
local function onenable(self, enable)
    if enable then
        self.inst:AddTag("debuffable")
    else
        self.inst:RemoveTag("debuffable")
    end
end
```

##### 3.2.2 核心方法

| 方法 | 行为 |
| --- | --- |
| `IsEnabled()` | 返回 `self.enable` |
| `Enable(enable)` | 切换开关；**关掉时会立刻把所有 buff 全部移除** |
| `RemoveOnDespawn()` | 把所有**没有** `keepondespawn` 标志的 buff 移除（玩家下线时由 `player_common` 调用） |
| `SetFollowSymbol(symbol, x, y, z)` | 同步给当前所有 buff 实体 |
| `HasDebuff(name)` | 是否存在指定键名的 buff |
| `GetDebuff(name)` | 返回 buff 实体（不是“是否有”，是要那个 entity 本身） |
| `AddDebuff(name, prefab, data, buffer)` | 添加 buff；同名时走 `Extend` |
| `RemoveDebuff(name)` | 移除指定 buff，会触发 `OnDetach` |
| `OnSave / OnLoad` | 持久化 buff 列表（注意每个 buff 实体都自带 `GetSaveRecord`） |
| `TransferComponent(newinst)` | 角色身体迁移（如复活/换身）时把 buff 整体搬走 |
| `GetDebugString()` | Ctrl+Shift+F1 调试面板里看 buff 列表 |

##### 3.2.3 内部小流程：`RegisterDebuff`

`AddDebuff` 调到 `RegisterDebuff` 函数（局部函数）做了 5 件事：

1. 检查新生成的实体上有没有 `debuff` 组件——**没有则直接 `Remove()`**（防御式编程）。
2. 把 `{ inst = ent, onremove = ... }` 写进账本。
3. 把 `ent.persists = false`：**buff 实体不参与游戏存档**（除非通过 OnSave/OnLoad 间接由 debuffable 持久化）。
4. 监听 buff 实体的 `onremove` 事件——这意味着即使外部直接 `ent:Remove()` 也能正确清账。
5. 调用 `ent.components.debuff:AttachTo(...)`，然后调用 `self.ondebuffadded`（如果有）。

#### 3.3 `EntityScript` 提供的便捷方法

源码 `scripts/entityscript.lua:2099-2135`，本质是 5 个语法糖：

```lua
function EntityScript:DebuffsEnabled()
function EntityScript:HasDebuff(name)
function EntityScript:GetDebuff(name)
function EntityScript:AddDebuff(name, prefab, data, skip_test, pre_buff_fn, buffer)
function EntityScript:RemoveDebuff(name)
```

其中 `AddDebuff` 比 `Debuffable:AddDebuff` 多了 3 个参数，请务必注意区分：

| 位置 | 参数 | 说明 |
| --- | --- | --- |
| 1 | `name` | 账本键名 |
| 2 | `prefab` | buff prefab 名称 |
| 3 | `data` | 透传到 buff 实体的 OnAttached/OnExtended 回调 |
| 4 | `skip_test` | true 则**绕过** `DebuffsEnabled` 和“已死亡”检查 |
| 5 | `pre_buff_fn` | buff 真正贴上前执行的回调（常用于：抹掉旧的、播放一个准备动画等） |
| 6 | `buffer` | 透传到回调，用于标识“施加者”等额外信息 |

注意：`skip_test` 默认会做两件事：
- `self:DebuffsEnabled()`——拒绝在被禁用 buff 的目标上添加；
- `not IsEntityDeadOrGhost(self)`——**死人/幽灵默认不能加 buff**。
  这就是为什么 `healthregenbuff` 内部还要在 `OnTick` 里再判一次 `HasTag("playerghost")`：**变成幽灵后买的果冻豆 buff，不会持续回血**。

### 四、老手篇：底层机制与最佳实践

#### 4.1 “非网络化 buff 实体”的标准模板

服务器权威的 buff prefab，90% 都长这个样子（取自 `hungerregenbuff.lua`）：

```lua
local function fn()
    local inst = CreateEntity()

    if not TheWorld.ismastersim then
        inst:DoTaskInTime(0, inst.Remove)
        return inst
    end

    inst.entity:AddTransform()
    inst.entity:Hide()                  
    inst.persists = false                  

    inst:AddTag("CLASSIFIED")              

    inst:AddComponent("debuff")
    inst.components.debuff:SetAttachedFn(OnAttached)
    inst.components.debuff:SetDetachedFn(inst.Remove)

    return inst
end
```

要点：

- **没有 `AddNetwork()`**——这是“非网络化实体”，客户端根本不知道它的存在，零带宽开销。
- **`entity:Hide()`** 防止它意外地参与渲染（虽然没 AnimState 也看不见）。
- **`CLASSIFIED` 标签**——让它免于参与某些遍历（比如不会被 `FindEntities` 抓到，除非显式带这个 tag）。
- **`SetDetachedFn(inst.Remove)`**——绝大部分 buff 在 Detach 后就应该自我销毁；不写这一行就会**泄露实体**。

> 反例：`healthregenbuff.lua` 设了 `inst.components.debuff.keepondespawn = true`，意味着即使玩家下线、`RemoveOnDespawn` 被调用，这个 buff 也会留下来等玩家上线后继续生效。**只有需要跨断线保留的 buff（食物 buff、皮肤 buff 等）才开**。

#### 4.2 同名 buff 的“延长”语义

`Debuffable:AddDebuff` 里的关键分支：

```lua
if self.debuffs[name] == nil then
    -- 全新挂载，走 OnAttached
else
    self.debuffs[name].inst.components.debuff:Extend(self.followsymbol, self.followoffset, data, buffer)
    return self.debuffs[name].inst
end
```

也就是说 **name 是“同一类 buff”的索引**。这给了你两种典型设计：

- **刷新型**：同名重新加 = 重置时间。在 `OnExtended` 回调里把 `timer:StopTimer + StartTimer` 重启即可（参考 `foodbuffs.lua`、`healthregenbuff.lua`）。
- **互斥型**：你希望“喝了新药水替换旧药水”而不是延长——可以在 `pre_buff_fn` 里手动 `RemoveDebuff`，然后再 Add（参考 `ghostly_elixirs.lua` 第 355–360 行）。

```lua
target:AddDebuff(buff_type, inst.buff_prefab, nil, nil, function()
    local cur_buff = target:GetDebuff(buff_type)
    if cur_buff ~= nil and cur_buff.prefab ~= inst.buff_prefab then
        target:RemoveDebuff(buff_type)
    end
end)
```

#### 4.3 `data` 与 `buffer` 参数怎么用

- `data` 是“**给本次贴标签**”用的数据包，典型例子见 `wx78_shadow_fuel_debuff.lua`：

```lua
local WX78_BUFF_DATA = { duration = TUNING.SKILLS.WX78.SHADOWFUEL_DEBUFF_TIME }
target:AddDebuff("wx78_shadow_fuel_debuff", "wx78_shadow_fuel_debuff", WX78_BUFF_DATA)

-- 在 buff prefab 内部：
local function buff_OnAttached(inst, target, followsymbol, followoffset, data)
    if data ~= nil and data.duration ~= nil then
        inst.components.timer:StartTimer(BUFF_TIMER, data.duration)
    end
end
```

  也就是说：**“给同一个 buff prefab 不同的延续时间/强度”**，靠 `data` 传参，不需要派生多种 prefab。

- `buffer` 是“**施加者**”，主要用于 `OnExtended` 重新计算时区分来源。`abigail.lua` 的 `abigail_vex_shadow_debuff` 里：

```lua
local function buff_OnExtended(inst, target, followsymbol, followoffset, data, buffer)
    local duration = TUNING.ABIGAIL_VEX_DURATION
    if buffer and buffer:HasTag("gestalt") then
        duration = TUNING.ABIGAIL_VEX_DURATION * TUNING.SKILLS.WENDY.ABIGAIL_GESTALT_VEX_DURATION_MULT
    end
    inst.decaytimer = inst:DoTaskInTime(duration, function() inst.components.debuff:Stop() end)
end
```

  当 Abigail 处于格斯塔形态时，**她释放出来的 vex debuff 持续时间被加成**——这个加成靠把 Abigail 本体作为 `buffer` 传进 `AddDebuff` 实现。

#### 4.4 玩家专用：`SetFollowSymbol` + `ondebuffadded`

`player_common.lua` 第 2803–2807 行：

```lua
inst:AddComponent("pinnable")
inst:AddComponent("debuffable")
inst.components.debuffable:SetFollowSymbol("headbase", 0, -200, 0)
inst.components.debuffable.ondebuffadded = fns.OnDebuffAdded
inst.components.debuffable.ondebuffremoved = fns.OnDebuffRemoved
```

`SetFollowSymbol` 的作用：**如果你的 buff prefab 有可见 FX**（即带 `AddNetwork` + `AddAnimState`），就会自动以 `headbase` 为锚点挂在玩家头顶上方 200 像素的位置。

`ondebuffadded`/`ondebuffremoved` 配合 `_buffsymbol` 网络字段，使**客户端立刻能感知到玩家正在持有何种 buff**——例如灵药皮肤会让玩家头上的 buff 符号亮起：

```lua
fns.OnDebuffAdded = function(inst, name, debuff)
    if name == "elixir_buff" then
        fns.SetSymbol(inst, debuff.prefab)
    end
end
```

> **玩家 prefab 的 pristine 阶段（`SetPristine` 之前）就 `AddTag("debuffable")`**（`player_common.lua:2530-2531`），这是一种性能优化——客户端**不需要等组件加上去**就能判断“这是一个 debuffable 实体”，可以减少一次网络等待。新写自定义 mod 角色时也建议保留这两行。

#### 4.5 跨断线、跨重连、跨身体迁移

| 场景 | 关键代码 | 行为 |
| --- | --- | --- |
| 玩家下线 | `player_common.lua:1413` 调 `debuffable:RemoveOnDespawn()` | 没 `keepondespawn` 的 buff 被清掉；带标记的留下 |
| 服务器存档 | `Debuffable:OnSave` | 把所有 buff 实体的 `GetSaveRecord()` 序列化进 player 自己的 save data |
| 服务器读档 | `Debuffable:OnLoad` | 用 `SpawnSaveRecord` 还原 buff 实体并重新 Register |
| 角色复活成不同身体（变身/变骷髅/换皮肤） | `Debuffable:TransferComponent(newinst)` | 把整份账本搬到新身上 |

> **重要**：能持久化的前提是 `buff` 实体本身写了正确的 `OnSave/OnLoad`（用 `timer` 组件自动支持）。如果你写了一个没有 `timer` 也没有自定义 `OnSave` 的 buff 实体，**重启服务器后这条 buff 会立刻过期/失效**。

#### 4.6 与 UI / 玩家说话系统的整合

老的食物 buff 系统用一个统一事件来通知 UI：

```lua
target:PushEvent("foodbuffattached", { buff = "ANNOUNCE_ATTACH_BUFF_ATTACK", priority = 1 })
target:PushEvent("foodbuffdetached", { buff = "ANNOUNCE_DETACH_BUFF_ATTACK", priority = 1 })
```

这两个事件由玩家身上的 `wisecracker` 组件（`scripts/components/wisecracker.lua` 第 308–314 行）监听，它会根据 `data.buff`（STRINGS 索引）和 `data.priority` 决定让角色喊一句对应的台词。**自定义 buff 想要触发“吃下/失效”的语音播报，就必须 PushEvent 这两个事件**——不发的话角色不会出声，但 buff 本身仍然能正常工作。

#### 4.7 性能与边界

1. **不要在 `OnTick`/`DoPeriodicTask` 里做高成本操作**——buff 实体的回调可能每秒数次执行。
2. **不要把 buff 实体加入物理碰撞**（`AddPhysics`），它只是个逻辑实体。
3. **`Stop()` 必须在 buff 实体的回调里调**（`inst.components.debuff:Stop()`）；如果是外部要解除，请调 `target:RemoveDebuff(name)`。两者效果一致，但调用方不同。
4. 给**怪物**（非玩家）加 buff 时也成立：见 `dummytarget.lua`、`punchingbag.lua`，它们都直接 `AddComponent("debuffable")` 然后什么都不设置，因此能接收各种测试用 buff。
5. **死亡瞬间不会自动清 buff**——你必须自己在 buff prefab 里 `ListenForEvent("death", ...)` 来 `Stop()`，否则 buff 会附在尸体/幽灵上，可能引起异常。这是 `healthregenbuff`/`hungerregenbuff`/`foodbuffs` 都显式监听 death 的原因。

### 五、可复用模板代码

下面给三段“拿来就能跑”的参考代码，按从简单到复杂排列。**这是参考实现，不会自动写进项目，请你按需贴到 mod 里。**

#### 5.1 最简：一个“5 秒后回 20 血”的一次性 buff

```lua
-- mods/<你的mod>/scripts/prefabs/sample_healbuff.lua
local function OnTick(inst, target)
    if target.components.health and not target.components.health:IsDead() then
        target.components.health:DoDelta(2, nil, inst.prefab)
    else
        inst.components.debuff:Stop()
    end
end

local function OnAttached(inst, target)
    inst.entity:SetParent(target.entity)
    inst.Transform:SetPosition(0, 0, 0)
    inst.task = inst:DoPeriodicTask(0.5, OnTick, nil, target)
    inst:ListenForEvent("death", function() inst.components.debuff:Stop() end, target)
end

local function OnTimerDone(inst, data)
    if data.name == "expire" then
        inst.components.debuff:Stop()
    end
end

local function fn()
    local inst = CreateEntity()
    if not TheWorld.ismastersim then
        inst:DoTaskInTime(0, inst.Remove)
        return inst
    end
    inst.entity:AddTransform()
    inst.entity:Hide()
    inst.persists = false
    inst:AddTag("CLASSIFIED")

    inst:AddComponent("debuff")
    inst.components.debuff:SetAttachedFn(OnAttached)
    inst.components.debuff:SetDetachedFn(inst.Remove)

    inst:AddComponent("timer")
    inst.components.timer:StartTimer("expire", 5)
    inst:ListenForEvent("timerdone", OnTimerDone)

    return inst
end

return Prefab("sample_healbuff", fn)
```

挂载：`eater:AddDebuff("sample_healbuff", "sample_healbuff")`。

#### 5.2 中等：支持自定义时长 + 自定义攻击加成

```lua
-- 用法：weapon:AddDebuff("sample_atkbuff", "sample_atkbuff", { duration = 30, mult = 1.5 })
local function attack_attach(inst, target, _, _, data)
    if target.components.combat and data and data.mult then
        target.components.combat.externaldamagemultipliers:SetModifier(inst, data.mult)
    end
end
local function attack_detach(inst, target)
    if target.components.combat then
        target.components.combat.externaldamagemultipliers:RemoveModifier(inst)
    end
end
local function OnExtended(inst, target, _, _, data)
    inst.components.timer:StopTimer("expire")
    inst.components.timer:StartTimer("expire", (data and data.duration) or 10)
    attack_attach(inst, target, nil, nil, data)
end

local function fn()
    local inst = CreateEntity()
    if not TheWorld.ismastersim then
        inst:DoTaskInTime(0, inst.Remove)
        return inst
    end
    inst.entity:AddTransform()
    inst.entity:Hide()
    inst.persists = false
    inst:AddTag("CLASSIFIED")

    inst:AddComponent("debuff")
    inst.components.debuff:SetAttachedFn(function(inst, target, _, _, data)
        inst.entity:SetParent(target.entity)
        inst.Transform:SetPosition(0, 0, 0)
        attack_attach(inst, target, nil, nil, data)
        inst.components.timer:StartTimer("expire", (data and data.duration) or 10)
        inst:ListenForEvent("death", function() inst.components.debuff:Stop() end, target)
    end)
    inst.components.debuff:SetDetachedFn(function(inst, target)
        attack_detach(inst, target)
        inst:Remove()
    end)
    inst.components.debuff:SetExtendedFn(OnExtended)
    inst.components.debuff.keepondespawn = true

    inst:AddComponent("timer")
    inst:ListenForEvent("timerdone", function(_, data)
        if data.name == "expire" then inst.components.debuff:Stop() end
    end)

    return inst
end

return Prefab("sample_atkbuff", fn)
```

#### 5.3 进阶：玩家头上挂一个可见 FX，并通知 UI

```lua
-- 注意：这次需要 AddNetwork
local assets = { Asset("ANIM", "anim/sample_visualbuff.zip") }

local function buff_OnAttached(inst, target)
    inst.entity:SetParent(target.entity)

    target:PushEvent("foodbuffattached", { buff = "ANNOUNCE_ATTACH_BUFF_ATTACK", priority = 1 })

    inst:ListenForEvent("death", function() inst.components.debuff:Stop() end, target)
end

local function buff_OnDetached(inst, target)
    target:PushEvent("foodbuffdetached", { buff = "ANNOUNCE_DETACH_BUFF_ATTACK", priority = 1 })
    inst:Remove()
end

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    inst.AnimState:SetBank("sample_visualbuff")
    inst.AnimState:SetBuild("sample_visualbuff")
    inst.AnimState:PlayAnimation("idle", true)

    inst:AddTag("FX")
    inst:AddTag("NOCLICK")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst.persists = false

    inst:AddComponent("debuff")
    inst.components.debuff:SetAttachedFn(buff_OnAttached)
    inst.components.debuff:SetDetachedFn(buff_OnDetached)

    inst:AddComponent("timer")
    inst.components.timer:StartTimer("expire", 30)
    inst:ListenForEvent("timerdone", function(_, data)
        if data.name == "expire" then inst.components.debuff:Stop() end
    end)

    return inst
end

return Prefab("sample_visualbuff", fn, assets)
```

挂载时配合玩家身上的 `SetFollowSymbol("headbase", ...)`，FX 会自动跟随到头顶。

### 六、源码出处与延伸阅读

| 你想了解什么 | 看这个文件 |
| --- | --- |
| Debuff 组件本体（最短，先读它） | `scripts/components/debuff.lua` |
| Debuffable 组件本体（重点是 `RegisterDebuff` 函数） | `scripts/components/debuffable.lua` |
| 实体级便捷方法 | `scripts/entityscript.lua` 第 2099–2135 行 |
| 玩家挂载范例 | `scripts/prefabs/player_common.lua` 第 2530–2531、2803–2807、1413 行 |
| 最简 buff prefab 模板 | `scripts/prefabs/hungerregenbuff.lua` |
| 带 timer 自动到期的模板 | `scripts/prefabs/healthregenbuff.lua` |
| 多 buff 工厂模式（强烈推荐） | `scripts/prefabs/foodbuffs.lua` |
| 有 fx 的可见 buff | `scripts/prefabs/spawnprotectionbuff.lua` |
| 用 data 传时长的范例 | `scripts/prefabs/wx78_shadow_fuel_debuff.lua` |
| 用 buffer 区分施加者 | `scripts/prefabs/abigail.lua` 第 1051–1121 行 |
| 食物挂 buff 实战 | `scripts/preparedfoods.lua` 第 470–474、870–875 行 |
| `pre_buff_fn` 用法 | `scripts/prefabs/ghostly_elixirs.lua` 第 351–360 行 |

> **学习路径建议**：新手只读到 2.2 即可开始动手；进阶玩家从 3.1 通读到 3.3，再去对照 `foodbuffs.lua` 自己抄一个；老手直接看 4.3–4.7 的细节，关注**持久化、网络化、施加者区分**这三块差异化能力。

---


## 22.2 Buff 的持续时间、刷新与清除机制

> 涉及源码文件：
> - `scripts/components/timer.lua`（Timer 组件本体——buff 时长引擎）
> - `scripts/components/debuff.lua`、`scripts/components/debuffable.lua`（22.1 节已通读，本节聚焦时长相关分支）
> - `scripts/entityscript.lua` 第 2099–2135 行（`AddDebuff` 的 `skip_test` 与 `pre_buff_fn`）
> - `scripts/componentutil.lua` 第 7–20 行（`IsEntityDead` / `IsEntityDeadOrGhost`）
> - `scripts/prefabs/healthregenbuff.lua`（固定时长 + 重置型刷新 + 死亡自停）
> - `scripts/prefabs/foodbuffs.lua`（统一模板：StopTimer + StartTimer 刷新）
> - `scripts/prefabs/wx78_shadow_fuel_debuff.lua`（`data.duration` 自定义时长 + fallback）
> - `scripts/prefabs/yoth_buffs.lua`、`scripts/prefabs/nonslipgrit.lua`（取较大值的刷新策略）
> - `scripts/prefabs/wintersfeastbuff.lua`（叠加型时长 + 上限）
> - `scripts/prefabs/spawnprotectionbuff.lua`（无 timer 的纯事件/距离驱动型清除）
> - `scripts/prefabs/ghostly_elixirs.lua` 第 348–367 行（`pre_buff_fn` 强制替换）
> - `scripts/prefabs/player_common.lua` 第 1413 行（断线时调 `RemoveOnDespawn`）
> - `scripts/preparedfoods.lua` 第 470–474、870–875 行（食物挂 buff 的典型写法）

### 一、本节要解决的问题（所有读者）

22.1 节我们已经知道：

- buff 实体身上挂 `debuff` 组件；
- 目标身上挂 `debuffable` 组件；
- 用 `target:AddDebuff(name, prefab, ...)` 把 buff 挂上去。

但还剩下一堆"工程问题"没有解答：

1. **怎么让一个 buff 在 N 秒后自动到期？**
2. **N 秒还没到，同一个 buff 又被加一次时——是延长 N 秒？还是从头再算？还是直接替换？**
3. **如果玩家在 buff 期间死了 / 变鬼魂 / 离线 / 换身体，buff 会发生什么？**
4. **能不能让 buff 像比赛"中场暂停"一样停一下再继续？**
5. **怎么让 buff 在玩家"做某件事"（攻击、建造、离开出生点）时立刻消失？**

22.2 节就把这些问题的实现方案讲清楚。**核心是 `Timer` 组件**——几乎所有 buff 的"沙漏"都是它在转。

### 二、新手篇：4 个最常用的写法

#### 2.1 套路一：固定时长 + 自然到期

90% 的 buff 想做的事就一句话——"挂上 → N 秒后掉"。完整模板：

```lua
local function OnAttached(inst, target)
    inst.entity:SetParent(target.entity)
    inst.Transform:SetPosition(0, 0, 0) --in case of loading
end

local function OnTimerDone(inst, data)
    if data.name == "buffover" then
        inst.components.debuff:Stop()
    end
end

local function fn()
    local inst = CreateEntity()
    if not TheWorld.ismastersim then
        inst:DoTaskInTime(0, inst.Remove)
        return inst
    end
    inst.entity:AddTransform()
    inst.entity:Hide()
    inst.persists = false
    inst:AddTag("CLASSIFIED")

    inst:AddComponent("debuff")
    inst.components.debuff:SetAttachedFn(OnAttached)
    inst.components.debuff:SetDetachedFn(inst.Remove)

    inst:AddComponent("timer")
    inst.components.timer:StartTimer("buffover", 30)
    inst:ListenForEvent("timerdone", OnTimerDone)
    return inst
end
```

要点：

- **`timer` 组件是 buff 倒计时的标配**——服务器存档/读档的时候 Timer 会自动保留剩余时间（见 3.2 节）。
- **`"buffover"` 是官方约定的 timer 名**（`foodbuffs.lua` 第 134、164、207 行都用这个）。其他 buff 也可以自起，但建议沿用这个名字便于多 mod 协作。
- **timer 跑完后调的是 `inst.components.debuff:Stop()`，不是 `inst:Remove()`**。`Stop` 会触发完整的 OnDetached 流程并清账本；`Remove` 是粗暴销毁。

#### 2.2 套路二：玩家死亡/变鬼魂时也要把 buff 收掉

`healthregenbuff.lua` 第 11–18 行：

```lua
local function OnAttached(inst, target)
    inst.entity:SetParent(target.entity)
    inst.Transform:SetPosition(0, 0, 0) --in case of loading
    inst.task = inst:DoPeriodicTask(TUNING.JELLYBEAN_TICK_RATE, OnTick, nil, target)
    inst:ListenForEvent("death", function()
        inst.components.debuff:Stop()
    end, target)
end
```

注意 `ListenForEvent` 的**第三个参数是 `target`**——监听的事件源是被影响的实体，不是 buff 实体自身。

> 不写这一行的后果：玩家死掉后 buff 实体仍然挂在尸体身上，OnTick 还在执行——`healthregenbuff` 内部又加了一层防御（OnTick 中 `IsDead()` / `HasTag("playerghost")` 判断），但你自己写 buff 的时候很容易忘。**先把死亡监听挂上，再写任何 OnTick 逻辑**。

#### 2.3 套路三：同名 buff 再次添加 = "续期"

`Debuffable:AddDebuff`（`debuffable.lua` 第 95–108 行）的关键分支：

```lua
if self.debuffs[name] == nil then
    local ent = SpawnPrefab(prefab)
    if ent ~= nil then
        RegisterDebuff(self, name, ent, data, buffer)
    end
    return ent
else
    self.debuffs[name].inst.components.debuff:Extend(self.followsymbol, self.followoffset, data, buffer)
    return self.debuffs[name].inst
end
```

所以：

```lua
target:AddDebuff("myhealbuff", "myhealbuff") -- 第一次 → 走 OnAttached
target:AddDebuff("myhealbuff", "myhealbuff") -- 第二次 → 同一个 buff 实体，走 OnExtended
```

最常见的"续期"实现（取自 `foodbuffs.lua` 第 163–171 行）：

```lua
local function OnExtended(inst, target)
    inst.components.timer:StopTimer("buffover")
    inst.components.timer:StartTimer("buffover", duration)
    target:PushEvent("foodbuffattached", ATTACH_BUFF_DATA)
end

inst.components.debuff:SetExtendedFn(OnExtended)
```

注意：

- **必须先 `StopTimer` 再 `StartTimer`**。Timer 不允许同名 timer 同时存在——直接 `StartTimer` 会被 `timer.lua` 第 37–40 行拒绝并打印 warning。
- **OnExtended 里不要再 `SetParent`**——目标没变，已经挂着了。

#### 2.4 套路四：吃东西就回血（食物挂 buff 实战）

`preparedfoods.lua` 第 470–474 行就是入门示例：

```lua
prefabs = { "healthregenbuff" },
oneat_desc = STRINGS.UI.COOKBOOK.FOOD_EFFECTS_HEALTH_REGEN,
oneatenfn = function(inst, eater)
    eater:AddDebuff("healthregenbuff", "healthregenbuff")
end,
```

`prefabs = { "healthregenbuff" }` **必须写**，否则世界加载时不会预加载 buff prefab，第一次吃可能会卡顿一帧。

#### 2.5 新手最常踩的 5 个坑

1. **`OnAttached` 内反复 `StartTimer`**：不要既在 `fn()` 顶层又在 `OnAttached` 都启动 timer，否则会得到 `A timer with the name buffover already exists` 警告（`timer.lua` 第 37–40 行）。建议二选一：
   - 简单 buff 在 `fn()` 顶层一次 `StartTimer`；
   - 时长由 `data` 决定的 buff 在 `OnAttached` 里 `StartTimer`，`fn()` 顶层可以留一个 fallback（见进阶 4.1 节）。
2. **想停 timer 时不要 `:Cancel()` 内部 task**。直接调 `inst.components.timer:StopTimer("buffover")`，否则 Timer 自身状态不会清。
3. **`StopTimer` 不会触发 `timerdone` 事件**——只有自然到期才触发。手动停 timer 想做收尾工作，请自己在 `StopTimer` 之后调用对应的逻辑。
4. **`SetTimeLeft` 接受负数但内部夹到 0**（`timer.lua` 第 108、111 行），但**不会立刻触发到期**，要等下一帧的 task 触发。
5. **客户端调用 `AddDebuff` 既不报错也不生效**——buff 是服务器权威。请确保 `TheWorld.ismastersim` 时再调。

### 三、进阶篇：Timer 组件全解 + 三种刷新策略

> 这一节适合"会写一个简单 buff，想搞懂底层时长机制"的同学。

#### 3.1 Timer 完整接口（基于 `scripts/components/timer.lua`）

| 方法 | 签名 | 行为要点 |
| --- | --- | --- |
| `StartTimer(name, time, paused, initialtime_override)` | 启动倒计时 | 已存在同名 timer 时**打印 warning 并 return**，不覆盖（第 37–40 行） |
| `StopTimer(name)` | 提前停止 | **不触发 `timerdone`**（第 56–66 行） |
| `TimerExists(name)` | 是否存在 | 第 27–29 行 |
| `PauseTimer(name)` | 暂停 | 内部 Cancel 当前 task，记下 timeleft（第 72–82 行） |
| `ResumeTimer(name)` | 恢复 | 用 timeleft 重新 `DoTaskInTime`（第 84–93 行） |
| `IsPaused(name)` | 是否处于暂停态 | 第 68–70 行 |
| `GetTimeLeft(name)` | 剩余时间 | 暂停时返回缓存值（第 95–102 行） |
| `SetTimeLeft(name, time)` | 修改剩余 | 内部相当于 Pause + 改值 + Resume（第 104–114 行） |
| `GetTimeElapsed(name)` | 已逝时间 | `initial_time - GetTimeLeft`（第 116–120 行） |
| `OnSave / OnLoad` | 持久化 | 自动支持 paused/initial_time（第 122–142 行） |
| `LongUpdate(dt)` | 跳时（如世界事件） | 所有 timer 一起减去 dt（第 144–148 行） |
| `TransferComponent(newinst)` | 转移到新身体 | 第 150–157 行 |
| `OnRemoveFromEntity()` | 卸载组件时清理 | 第 6–12 行，会 cancel 所有 task |

到期事件统一形式：

```lua
inst:ListenForEvent("timerdone", function(inst, data)
    if data.name == "buffover" then
        inst.components.debuff:Stop()
    end
end)
```

`data.name` 是 timer 的键名。**一个 buff 实体可以挂多个 timer**（如一个 `"buffover"` 控总时长，一个 `"tickdown"` 控阶段切换），用 `name` 区分回调。

#### 3.2 Timer 的持久化怎么自动起作用

`timer.lua` 第 122–142 行：

```lua
function Timer:OnSave()
    local data = {}
    for k, v in pairs(self.timers) do
        data[k] = {
            timeleft = self:GetTimeLeft(k),
            paused = v.paused,
            initial_time = v.initial_time,
        }
    end
    return next(data) ~= nil and { timers = data } or nil
end

function Timer:OnLoad(data)
    if data.timers ~= nil then
        for k, v in pairs(data.timers) do
            self:StopTimer(k)
            self:StartTimer(k, v.timeleft, v.paused, v.initial_time)
        end
    end
end
```

但是，**buff 实体的 `persists = false`**——它不会单独存档。它是怎么穿越服务器重启的？

来看 `debuffable.lua` 第 126–137 行：

```lua
function Debuffable:OnSave()
    if next(self.debuffs) == nil then
        return
    end

    local data = {}
    for k, v in pairs(self.debuffs) do
        local saved--[[, refs]] = v.inst:GetSaveRecord()
        data[k] = saved
    end
    return { debuffs = data, add_component_if_missing = true }
end
```

`Debuffable:OnSave` 主动调用每个 buff 实体的 `GetSaveRecord()`——也就是说**buff 实体的存档数据是被宿主"打包带走"的**。

实际效果：

- 你的 buff prefab **只要带 `timer` 组件，且没有别的需要持久化的字段，存档/读档完全无感**。
- 如果除了 timer 还有自己定义的字段（比如吸血次数），那需要在 buff prefab 里写自己的 `OnSave/OnLoad`。

#### 3.3 三种刷新策略的代码对照

##### A. 重置式（最常见，foodbuffs 全家桶）

`foodbuffs.lua` 第 163–171 行：

```lua
local function OnExtended(inst, target)
    inst.components.timer:StopTimer("buffover")
    inst.components.timer:StartTimer("buffover", duration)

    target:PushEvent("foodbuffattached", ATTACH_BUFF_DATA)
    if onextendedfn ~= nil then
        onextendedfn(inst, target)
    end
end
```

**语义**：每次重新加 buff，时长都从头开始算。原本剩 1 秒，重新一加变 `duration`。

**适用**：食物、药水等"鼓励玩家持续吃"的设定。

##### B. 取较大值（避免被短 buff 截短）

`yoth_buffs.lua` 第 17–23 行（与 `nonslipgrit.lua` 第 199–205 行实质相同）：

```lua
local function OnExtendedBuff(inst, target, followsymbol, followoffset, data)
    local duration = data and data.duration or TUNING.YOTH_PRINCESS_SUMMON_COOLDOWN
    local time_remaining = inst.components.timer:GetTimeLeft("buffover")
    if time_remaining == nil or duration > time_remaining then
        inst.components.timer:SetTimeLeft("buffover", duration)
    end
end
```

**语义**：想加的时长**比剩余时间长**就拉到那么长，否则保持不变。

**适用**：CD 类 buff（一旦触发就要冷却一段时间，新触发不该让 CD 变短）；持续型陷阱（同一个陷阱多次触发，取最长持续）。

##### C. 叠加式（每次都加一段，可设上限）

`wintersfeastbuff.lua` 第 78–97 行：

```lua
local function AddEffectBonus(inst, num_feasters, num_foodtypes, num_totalfood)
    num_feasters, num_foodtypes = num_feasters or 1, num_foodtypes or 1

    local timeleft = inst.components.timer:GetTimeLeft("buffover")

    local score = ((num_feasters^0.3)*0.3* (num_foodtypes))  + ((num_totalfood-num_foodtypes)*0.2)
    local bonus = (score * TUNING.TOTAL_DAY_TIME/2)/TUNING.WINTERSFEASTBUFF.EATTIME
    if bonus > 0 then
        inst.components.timer:SetTimeLeft("buffover", timeleft + bonus )
    end

    inst.SoundEmitter:SetParameter("loop", "intensity", CalcIntensity(inst))
end
```

**语义**：每次"额外的好事"都把 buff 时间往后推。

**特别注意**：冬季盛宴 buff 没有用 `OnExtended`——它通过外部脚本调用 `inst.addeffectbonusfn`（第 123 行 `inst.addeffectbonusfn = AddEffectBonus`）来加时长。原因是冬季盛宴的"评分逻辑"远比"再加一次 buff"复杂得多。**所以当你的 buff 需要带参数累加时，给 buff 实体挂一个自定义函数指针、由外部脚本调用，比硬塞进 OnExtended 更清晰**。

它还有一个隐含的"上限"——`CalcIntensity` 函数（第 25–27 行）：

```lua
local function CalcIntensity(inst)
    return math.min(inst.components.timer:GetTimeLeft("buffover") / TUNING.WINTERSFEASTBUFF.MAXDURATION, 1)
end
```

视觉/音效强度按 `min(剩余时间 / MAXDURATION, 1)` 计算——这意味着即便 timer 自然能超过 MAXDURATION，玩家**感知上的"满档"**也封顶在 MAXDURATION。

#### 3.4 关于 `OnExtended` 的 5 个细节

1. **OnExtended 不会刷新 `inst.entity:SetParent`**——目标没换，依旧贴在同一目标上。
2. **`data` 和 `buffer` 在 OnExtended 中会被重新传**——所以"用 buffer 区分施加者"的逻辑（22.1 第 4.3 节）在 OnExtended 中同样适用。
3. **`debuffable.ondebuffadded` 在 OnExtended 时不会触发**（看 `debuffable.lua` 第 87–89 行：`ondebuffadded` 只在 `RegisterDebuff` 里调用，而 OnExtended 走的是第 104 行的另一分支）。如果你需要每次"续期"也通知 UI，请在 OnExtended 里自己 PushEvent。
4. **OnExtended 内可以再次调 `inst.entity:Hide()` / `inst:RemoveTag(...)`**，但通常没必要——它们在 OnAttached 时已经设好。
5. **没设 `SetExtendedFn` 的 buff，被重新 Add 时啥也不会发生**——这就是 `hungerregenbuff.lua` 的策略，它**故意不设**，因为吃饭 buff 的时长由调用方控制，buff 实体内部没有过期逻辑。

### 四、老手篇：边界场景与高阶技巧

#### 4.1 `data.duration` + fallback：一个 buff 支持多种时长

`wx78_shadow_fuel_debuff.lua` 完整结构示范了"自定义时长 + fallback"模式：

```lua
local BUFF_TIMER = "wx78_shadow_fuel_debuff"

local function buff_OnAttached(inst, target, followsymbol, followoffset, data)
    inst.entity:SetParent(target.entity)
    inst.Transform:SetPosition(0, 0, 0)

    target:PushEvent("foodbuffattached", ATTACH_BUFF_DATA)
    -- ...省略其它逻辑...

    if data ~= nil and data.duration ~= nil then
        if target.components.wx78_abilitycooldowns ~= nil then
            target.components.wx78_abilitycooldowns:RestartAbilityCooldown("shadow_energy", data.duration)
        end
        inst.components.timer:StopTimer(BUFF_TIMER)
        inst.components.timer:StartTimer(BUFF_TIMER, data.duration)
    end

    inst:ListenForEvent("death", function()
        inst.components.debuff:Stop()
    end, target)
end

local function fn()
    -- ...
    inst:AddComponent("timer")
    inst.components.timer:StartTimer(BUFF_TIMER, TUNING.SKILLS.WX78.SHADOWFUEL_DEBUFF_TIME) -- fall back
    inst:ListenForEvent("timerdone", buff_OnTimerDone)
    return inst
end
```

**两条 StartTimer 都要存在**：

- `fn()` 末尾的 `StartTimer(BUFF_TIMER, FALLBACK)` 保证**就算调用方忘了传 `data`，也有 20 秒的兜底时长**（`TUNING.SKILLS.WX78.SHADOWFUEL_DEBUFF_TIME = 20`，`tuning.lua` 第 7560 行）。
- `OnAttached` 里的 `if data.duration` 分支再次 `Stop + Start` 用调用方指定的时长覆盖。

调用方：

```lua
target:AddDebuff(
    "wx78_shadow_fuel_debuff",
    "wx78_shadow_fuel_debuff",
    { duration = 120 }
)
```

#### 4.2 强制替换：`pre_buff_fn` 在 Add 之前手动 Remove

`ghostly_elixirs.lua` 第 348–367 行：

```lua
local function DoApplyElixir(inst, giver, target)
    local buff_type = "elixir_buff"

    if inst.potion_tunings.super_elixir then
        buff_type = "super_elixir_buff"
    end

    local buff = target:AddDebuff(buff_type, inst.buff_prefab, nil, nil, function()
        local cur_buff = target:GetDebuff(buff_type)
        if cur_buff ~= nil and cur_buff.prefab ~= inst.buff_prefab then
            target:RemoveDebuff(buff_type)
        end
    end)
    ...
end
```

注意这里调用的是 `EntityScript:AddDebuff`（`entityscript.lua` 第 2115 行，**6 个参数**），不是 `Debuffable:AddDebuff`（**4 个参数**）。

```lua
function EntityScript:AddDebuff(name, prefab, data, skip_test, pre_buff_fn, buffer)
```

| 位置 | 参数 | 这里的用法 |
| --- | --- | --- |
| 1 | `name` | `"elixir_buff"` |
| 2 | `prefab` | `inst.buff_prefab`（不同药水不同） |
| 3 | `data` | nil |
| 4 | `skip_test` | nil（保持默认死亡/Enable 检查） |
| 5 | `pre_buff_fn` | 闭包：如果当前挂着的不是同一个 prefab 就先 Remove |
| 6 | `buffer` | nil |

执行流程：

1. `pre_buff_fn` 在 `AddDebuff` **真正 spawn buff prefab 之前**执行。
2. 如果当前已有同 name 但不同 prefab 的 buff，就调 `RemoveDebuff` → 走 OnDetached → 账本清空。
3. 然后 `AddDebuff` 内部走"新挂载"分支（因为 name 已经不在账本里了），spawn 新 prefab → 走 OnAttached。

**典型场景**：

- 一组"互斥"的同类 buff（5 种月相精灵灵药、3 种力量药剂等，只能挂一种）。
- "新版本"buff 强制覆盖"旧版本"buff，并触发完整的 OnAttached 而不是 OnExtended。

#### 4.3 条件触发清除：监听任意事件 + 距离检测

`spawnprotectionbuff.lua` 第 30–76 行的"重生保护"buff 是事件清除的教科书：

```lua
local function owner_stop_buff_fn(owner)
    if owner:IsValid() then
        owner:RemoveDebuff("spawnprotectionbuff")
    end
end

local function check_dist_from_spawnpt(inst, target)
    if not (target:GetDistanceSqToPoint(inst.spawn_pt) < TUNING.SPAWNPROTECTIONBUFF_SPAWN_DIST_SQ) then
        inst.check_dist_task:Cancel()
        inst.check_dist_task = nil

        inst.expire_task:Cancel()
        start_exipiring(inst)
    elseif TheWorld.state.isnight then
        inst.expire_task:Cancel()
        inst.expire_task = inst:DoTaskInTime(TUNING.SPAWNPROTECTIONBUFF_IDLE_DURATION, start_exipiring)
    end
end

local function buff_OnAttached(inst, target)
    inst.entity:SetParent(target.entity)
    inst.spawn_pt = target:GetPosition()
    inst.fx = SpawnFx(target)

    inst.expire_task   = inst:DoTaskInTime(TUNING.SPAWNPROTECTIONBUFF_IDLE_DURATION, start_exipiring)
    inst.check_dist_task = inst:DoPeriodicTask(0.25, check_dist_from_spawnpt, nil, target)

    inst:OnEnableProtectionFn(target, true)

    inst:ListenForEvent("death",            owner_stop_buff_fn, target)
    inst:ListenForEvent("doattack",         owner_stop_buff_fn, target)
    inst:ListenForEvent("onattackother",    owner_stop_buff_fn, target)
    inst:ListenForEvent("onmissother",      owner_stop_buff_fn, target)
    inst:ListenForEvent("onthrown",         owner_stop_buff_fn, target)
    inst:ListenForEvent("buildstructure",   owner_stop_buff_fn, target)
    inst:ListenForEvent("builditem",        owner_stop_buff_fn, target)
    inst:ListenForEvent("on_enter_might_gym", owner_stop_buff_fn, target)
end
```

设计要点（值得记下来）：

1. **完全没用 `timer` 组件**——用 `DoTaskInTime` 直接搞定。因为"重生保护"持续时间短到不需要持久化。
2. **监听挂在 buff 实体上，事件源是 target**（`ListenForEvent` 第三参）——buff 实体被移除时这些监听会自动解绑（`onremove` 时清理），不会泄露。
3. **清除调用的是 `target:RemoveDebuff("spawnprotectionbuff")` 而不是 `inst.components.debuff:Stop()`**。两者效果一致，但 RemoveDebuff 这种"由外部主动撕"的语义在事件回调里更直观。
4. **多重清除条件并行存在**：时间到、玩家攻击、玩家建造、离开出生点、天黑站着不动……任何一个先发生都会触发清除。
5. **没有 `OnExtended`**——重新触发不应该续命（防止玩家无脑刷重生保护）。

#### 4.4 暂停与恢复

Timer 天然支持 `PauseTimer / ResumeTimer`，但**官方所有名字带 "buff" 的 prefab 都没有直接使用它们**——`scripts/prefabs/*buff*.lua` 下 ripgrep 找不到任何匹配。

原因：饥荒的 buff 设计哲学是"挂上就在跑、撕下就消失"，不存在"暂停一会"的需求；真正用到 PauseTimer / ResumeTimer 的地方多是 boss 行为、世界事件、技能 CD 等场景。

但 mod 完全可以用。例子：实现一个"赛车 buff"——只有玩家移动时才扣 buff 时间：

```lua
local function OnAttached(inst, target)
    inst.entity:SetParent(target.entity)
    inst.Transform:SetPosition(0, 0, 0)

    inst:ListenForEvent("newstate", function(_, data)
        local sg = target.sg
        if sg == nil then return end

        local moving = sg:HasStateTag("moving") or sg:HasStateTag("running")
        if moving then
            if inst.components.timer:IsPaused("buffover") then
                inst.components.timer:ResumeTimer("buffover")
            end
        else
            if inst.components.timer:TimerExists("buffover") and not inst.components.timer:IsPaused("buffover") then
                inst.components.timer:PauseTimer("buffover")
            end
        end
    end, target)

    inst:ListenForEvent("death", function() inst.components.debuff:Stop() end, target)
end
```

> `Timer:OnSave` 第 128 行把 `paused = v.paused` 一起序列化了，所以**服务器重启后暂停态会被精确恢复**。

#### 4.5 玩家断线时的清除时序

`player_common.lua` 第 1413 行（玩家下线/迁移时执行）：

```lua
inst.components.debuffable:RemoveOnDespawn()
```

`Debuffable:RemoveOnDespawn`（`debuffable.lua` 第 39–49 行）：

```lua
function Debuffable:RemoveOnDespawn()
    local toremove = {}
    for k, v in pairs(self.debuffs) do
        if not (v.inst.components.debuff ~= nil and v.inst.components.debuff.keepondespawn) then
            table.insert(toremove, k)
        end
    end
    for i, v in ipairs(toremove) do
        self:RemoveDebuff(v)
    end
end
```

清除规则一张表：

| `keepondespawn` | 玩家断线/被迁出时 | 玩家重连后 |
| --- | --- | --- |
| `nil` / `false`（默认） | **立刻 RemoveDebuff** → 走 OnDetached | buff 已没 |
| `true` | 保留在玩家账本里 | buff 仍在，timer 状态延续 |

**典型选择**：

- 出生保护、临时增益（药剂、boss 战 buff、教程引导）→ `keepondespawn` 默认 `nil` / 不设。
- 食物 buff、皮肤永久 buff、技能 CD、宠物再召唤 CD（yoth_princesscooldown_buff）、wx78 暗影燃料 → `keepondespawn = true`。

直接看几个对比（截取关键行）：

```lua
-- healthregenbuff.lua 第 56 行（果冻豆回血——食物 buff）
inst.components.debuff.keepondespawn = true

-- foodbuffs.lua 第 204 行（buff_attack 等 7 种食物 buff）
inst.components.debuff.keepondespawn = true

-- wintersfeastbuff.lua 第 128 行
inst.components.debuff.keepondespawn = true

-- yoth_buffs.lua 第 51 行（princess 召唤 CD）
inst.components.debuff.keepondespawn = true

-- hungerregenbuff.lua（吃饭回饱食度）→ 没设，断线就掉
-- spawnprotectionbuff.lua（重生保护）→ 没设，断线就掉
```

> 注意 `hungerregenbuff` 故意没设 `keepondespawn`——因为"吃饭那一瞬间的回饱食度"应该立刻在线消耗完，下线一段时间再上线还有就不合理了。

#### 4.6 死亡、鬼魂、复活——三层防御

死亡相关的清理分三步看：

1. **死亡瞬间（玩家变成尸体 / 鬼魂）**：buff 自身在 OnAttached 里写 `ListenForEvent("death", ..., target)` → `Stop()`。
2. **OnTick 二次防御**：`healthregenbuff.lua` 第 1–9 行：

   ```lua
   local function OnTick(inst, target)
       if target.components.health ~= nil and
           not target.components.health:IsDead() and
           not target:HasTag("playerghost") then
           target.components.health:DoDelta(TUNING.JELLYBEAN_TICK_VALUE, nil, "jellybean")
       else
           inst.components.debuff:Stop()
       end
   end
   ```

   即便 `death` 监听某种原因没及时触发，下一个 tick 也会自停。
3. **`AddDebuff` 入口拒绝**：`entityscript.lua` 第 2115–2129 行：

   ```lua
   function EntityScript:AddDebuff(name, prefab, data, skip_test, pre_buff_fn, buffer)
       if self.components.debuffable == nil then
           self:AddComponent("debuffable")
       end

       if skip_test or (self:DebuffsEnabled() and not IsEntityDeadOrGhost(self)) then
           if pre_buff_fn then
               pre_buff_fn()
           end
           self.components.debuffable:AddDebuff(name, prefab, data, buffer)
           return true
       end

       return false
   end
   ```

   `IsEntityDeadOrGhost`（`componentutil.lua` 第 15–20 行）：

   ```lua
   function IsEntityDeadOrGhost(inst, require_health)
       if inst:HasTag("playerghost") then
           return true
       end
       return IsEntityDead(inst, require_health)
   end
   ```

   死人/鬼魂**默认不能加新 buff**。

`skip_test = true` 适合的少数场景：

- boss 死亡瞬间触发的"死亡爆炸"型 debuff，需要在 boss 已死时还能挂上。
- 剧情触发的"鬼眼视觉"——主动给鬼魂加 buff。
- 测试用例。

复活成新身体时——`Debuffable:TransferComponent`（`debuffable.lua` 第 152–162 行）：

```lua
function Debuffable:TransferComponent(newinst)
    local data = self:OnSave()
    if data then
        local newcomponent = newinst.components.debuffable
        if not newcomponent then
            newinst:AddComponent("debuffable")
            newcomponent = newinst.components.debuffable
        end
        newcomponent:OnLoad(data)
    end
end
```

它把整个账本通过 OnSave/OnLoad 流程"打包搬家"。**前提是 buff 自身的状态都能通过 `GetSaveRecord` 序列化**——所以**没有 timer 也没有自定义 OnSave 的 buff，搬家过去会立刻丢失状态**。

#### 4.7 用 `Enable(false)` 一键清空

`debuffable.lua` 第 28–37 行：

```lua
function Debuffable:Enable(enable)
    self.enable = enable
    if not enable then
        local k = next(self.debuffs)
        while k ~= nil do
            self:RemoveDebuff(k)
            k = next(self.debuffs)
        end
    end
end
```

`Enable(false)` 是**最暴力的清除手段**：

- 把 `self.enable` 置为 false → 后续 `AddDebuff` 直接被 `if self.enable then` 拒掉（第 96 行）。
- 立即把所有现有 buff 撕掉，**挨个走完整的 OnDetached 流程**。
- 同时通过 `onenable` 回调把目标身上的 `"debuffable"` tag 也撕掉（第 1–7 行）——可能影响其它依赖这个 tag 的系统。

**适用场景**：

- 玩家进入某些剧情副本，临时禁用所有 buff。
- 角色换身体（变骷髅、变水龙、变怪物）前的状态清空。
- 调试控制台一键 `c_select().components.debuffable:Enable(false)`。

#### 4.8 性能与边界

1. **不要在 OnTick 里反复 `GetTimeLeft`**：`timer.lua` 第 99 行每次都会调用 `GetTime()`。需要"剩余 X 秒以下"判断的，请挂一个**短一些的副 timer**（如 `"phase2_start"`），timerdone 时切换阶段。
2. **`SetTimeLeft` 不便宜**：内部走 `PauseTimer + ResumeTimer`（第 109–112 行），等于撤销 + 重建一次 task。**不要每帧 `SetTimeLeft -1`**——你需要的是 `DoPeriodicTask` 或者监听 `timerdone`。
3. **`LongUpdate` 是世界级时间跳跃接口**（用于"快进 N 个游戏日"），普通 buff 不要主动调。但要意识到：服务器在加载存档时可能会调用 `LongUpdate(dt)` 让 buff 在"玩家不在线的那段时间"也按比例消逝。
4. **`StartTimer` 同名会被拒**——要先 `StopTimer` 或者用 `TimerExists` 判断。`yoth_buffs.lua` 第 5–7 行就用了这个守卫：

   ```lua
   if not inst.components.timer:TimerExists("buffover") then
       inst.components.timer:StartTimer("buffover", duration)
   end
   ```

#### 4.9 调试技巧

控制台（按 `~` 打开，需要管理员权限）：

```lua
-- 给当前选中目标加一个测试 buff
c_select():AddDebuff("test", "healthregenbuff")

-- 查看当前 buff 列表
print(c_select().components.debuffable:GetDebugString())

-- 看 timer 详情
local buff = c_select().components.debuffable:GetDebuff("test")
print(buff.components.timer:GetDebugString())

-- 改剩余时间（剩 2 秒）
buff.components.timer:SetTimeLeft("regenover", 2)

-- 立即停掉
c_select():RemoveDebuff("test")

-- 一键清空全部 buff
c_select().components.debuffable:Enable(false)
c_select().components.debuffable:Enable(true) -- 别忘了恢复
```

调试面板（`Ctrl+Shift+F1`）勾上 Debuffable 也能可视化看到 buff 与剩余时间。

### 五、决策表（按需求查表）

| 你想做什么 | 选哪种策略 | 参考 prefab |
| --- | --- | --- |
| 固定时长，N 秒后自动到期 | `timer:StartTimer` + `timerdone` 监听 | `foodbuffs.lua` |
| 重新加同名 buff 时**重置**时长 | OnExtended: `StopTimer + StartTimer` | `foodbuffs.lua` |
| 重新加时**取较大值**（不缩短现有 CD） | OnExtended: 比较 + `SetTimeLeft` | `yoth_buffs.lua` / `nonslipgrit.lua` |
| 持续做好事可以**叠加**时长（带上限） | 外部脚本调 `SetTimeLeft(left + bonus)` + `CalcIntensity` 视觉封顶 | `wintersfeastbuff.lua` |
| 不同来源不同时长 | 用 `data.duration` + fallback `StartTimer` | `wx78_shadow_fuel_debuff.lua` |
| "新喝的药水替换旧药水" | `EntityScript:AddDebuff` 第 5 参 `pre_buff_fn` 里手动 Remove | `ghostly_elixirs.lua` |
| 玩家做了某事就立刻消失 | OnAttached 里 `ListenForEvent(<事件>, fn, target)` | `spawnprotectionbuff.lua` |
| 离开某个点/夜晚自动结束 | `DoPeriodicTask` 检查距离 + 状态 | `spawnprotectionbuff.lua` |
| 玩家死亡时 buff 也终止 | OnAttached 里 `ListenForEvent("death", ..., target)` | 几乎所有玩家 buff |
| 跨断线/重连保留 | `inst.components.debuff.keepondespawn = true` | `healthregenbuff.lua` / `foodbuffs.lua` |
| 跨身体迁移（复活换身） | 什么都不用做，`Debuffable:TransferComponent` 自动 | `player_common.lua` |
| 暂停一会儿再继续 | `PauseTimer` / `ResumeTimer`（mod 用） | 官方无现成例子 |
| 整批清空 buff | `debuffable:Enable(false)` 或循环 RemoveDebuff | `debuffable.lua` |
| 给鬼魂 / 尸体加 buff（少数场景） | `AddDebuff(..., skip_test=true)` | `entityscript.lua` 第 2115 行 |

### 六、可复用模板代码

下面三段"拿来就能跑"的参考代码，覆盖最常见的三种需求。**这是参考实现，不会自动写进项目，请你按需贴到 mod 里**。

#### 6.1 模板一：取较大值刷新 + 自定义时长 + 死亡自停

完整可跑模板。最适合"力量药剂、武器附魔"这类增益 buff。

```lua
-- mods/<你的mod>/scripts/prefabs/sample_atkbuff.lua
local BUFF_TIMER = "buffover"
local DEFAULT_DURATION = 30
local DEFAULT_MULT = 1.5

local function attach_attack(inst, target, data)
    if target.components.combat then
        local mult = (data and data.mult) or DEFAULT_MULT
        target.components.combat.externaldamagemultipliers:SetModifier(inst, mult)
    end
end

local function detach_attack(inst, target)
    if target.components.combat then
        target.components.combat.externaldamagemultipliers:RemoveModifier(inst)
    end
end

local function OnAttached(inst, target, _, _, data)
    inst.entity:SetParent(target.entity)
    inst.Transform:SetPosition(0, 0, 0)

    local duration = (data and data.duration) or DEFAULT_DURATION
    if inst.components.timer:TimerExists(BUFF_TIMER) then
        inst.components.timer:StopTimer(BUFF_TIMER)
    end
    inst.components.timer:StartTimer(BUFF_TIMER, duration)

    attach_attack(inst, target, data)

    inst:ListenForEvent("death", function()
        inst.components.debuff:Stop()
    end, target)
end

local function OnExtended(inst, target, _, _, data)
    local duration = (data and data.duration) or DEFAULT_DURATION
    local time_remaining = inst.components.timer:GetTimeLeft(BUFF_TIMER)
    if time_remaining == nil or duration > time_remaining then
        if inst.components.timer:TimerExists(BUFF_TIMER) then
            inst.components.timer:SetTimeLeft(BUFF_TIMER, duration)
        else
            inst.components.timer:StartTimer(BUFF_TIMER, duration)
        end
    end
    attach_attack(inst, target, data)
end

local function OnDetached(inst, target)
    detach_attack(inst, target)
    inst:Remove()
end

local function OnTimerDone(inst, data)
    if data.name == BUFF_TIMER then
        inst.components.debuff:Stop()
    end
end

local function fn()
    local inst = CreateEntity()
    if not TheWorld.ismastersim then
        inst:DoTaskInTime(0, inst.Remove)
        return inst
    end

    inst.entity:AddTransform()
    inst.entity:Hide()
    inst.persists = false
    inst:AddTag("CLASSIFIED")

    inst:AddComponent("debuff")
    inst.components.debuff:SetAttachedFn(OnAttached)
    inst.components.debuff:SetDetachedFn(OnDetached)
    inst.components.debuff:SetExtendedFn(OnExtended)
    inst.components.debuff.keepondespawn = true

    inst:AddComponent("timer")
    inst:ListenForEvent("timerdone", OnTimerDone)

    return inst
end

return Prefab("sample_atkbuff", fn)
```

调用方式：

```lua
-- 默认 30 秒 1.5 倍
target:AddDebuff("sample_atkbuff", "sample_atkbuff")
-- 60 秒 2 倍（如果当前剩余不足 60，就拉到 60；超过则不变）
target:AddDebuff("sample_atkbuff", "sample_atkbuff", { mult = 2.0, duration = 60 })
```

#### 6.2 模板二：互斥灵药（pre_buff_fn 强制替换）

适合"一组互斥的同类 buff，只能挂一种"。

```lua
-- 多种 buff prefab 共用同一个 name = "my_elixir_buff"
-- 但 prefab 名各不相同（my_red_potion_buff / my_blue_potion_buff / ...）

local function ApplyMyElixir(target, buff_prefab)
    target:AddDebuff("my_elixir_buff", buff_prefab, nil, nil, function()
        local cur = target:GetDebuff("my_elixir_buff")
        if cur ~= nil and cur.prefab ~= buff_prefab then
            target:RemoveDebuff("my_elixir_buff")
        end
    end)
end

-- 使用示例
ApplyMyElixir(player, "my_red_potion_buff")
ApplyMyElixir(player, "my_blue_potion_buff") -- 红的会被先撕掉，再挂蓝的
ApplyMyElixir(player, "my_blue_potion_buff") -- 同 prefab → 走 OnExtended 续期
```

注意：

- 每种 buff prefab 的实现可以差异很大（红的回血、蓝的回 san、紫的减伤），它们**不需要互相知道彼此**。
- "互斥"完全由 `pre_buff_fn` + 同 name 实现。

#### 6.3 模板三：行为打断 + 距离打断（重生保护类）

适合"一旦玩家主动行动/离开特定位置就立刻消失"的 buff。

```lua
local BUFF_NAME = "sample_safebuff"
local SAFE_RADIUS_SQ = 10 * 10

local function buff_off(target)
    if target:IsValid() then
        target:RemoveDebuff(BUFF_NAME)
    end
end

local function OnAttached(inst, target)
    inst.entity:SetParent(target.entity)
    inst.spawn_pt = target:GetPosition()

    inst:ListenForEvent("death",         buff_off, target)
    inst:ListenForEvent("doattack",      buff_off, target)
    inst:ListenForEvent("attacked",      buff_off, target)
    inst:ListenForEvent("buildstructure",buff_off, target)
    inst:ListenForEvent("builditem",     buff_off, target)

    inst.expire_task = inst:DoTaskInTime(60, function()
        buff_off(target)
    end)

    inst.check_dist_task = inst:DoPeriodicTask(0.25, function()
        if target:GetDistanceSqToPoint(inst.spawn_pt) > SAFE_RADIUS_SQ then
            buff_off(target)
        end
    end)

    target:AddTag("notarget")
end

local function OnDetached(inst, target)
    if target:IsValid() then
        target:RemoveTag("notarget")
    end
    if inst.expire_task then
        inst.expire_task:Cancel()
        inst.expire_task = nil
    end
    if inst.check_dist_task then
        inst.check_dist_task:Cancel()
        inst.check_dist_task = nil
    end
    inst:Remove()
end

local function fn()
    local inst = CreateEntity()
    if not TheWorld.ismastersim then
        inst:DoTaskInTime(0, inst.Remove)
        return inst
    end

    inst.entity:AddTransform()
    inst.entity:Hide()
    inst.persists = false
    inst:AddTag("CLASSIFIED")

    inst:AddComponent("debuff")
    inst.components.debuff:SetAttachedFn(OnAttached)
    inst.components.debuff:SetDetachedFn(OnDetached)

    return inst
end

return Prefab("sample_safebuff", fn)
```

注意：

- 这里**不用 timer 组件**，因为重生保护时间短、不需要持久化。
- 多重清除条件并行：60 秒超时、玩家攻击、被攻击、建造、距离超出——任一触发都会清除。
- `OnDetached` 里**记得 Cancel 所有 task**，否则 buff 没了 task 还在跑。

### 七、源码出处与延伸阅读

| 你想确认 | 看这个 |
| --- | --- |
| Timer 完整接口 | `scripts/components/timer.lua` |
| Timer 的持久化机制 | 同上第 122–142 行 |
| Debuff 的 4 个回调 Setter | `scripts/components/debuff.lua` 第 12–26 行 |
| AddDebuff 的"同名走 Extend"分支 | `scripts/components/debuffable.lua` 第 95–108 行 |
| `RemoveOnDespawn` 在玩家断线时被调 | `scripts/prefabs/player_common.lua` 第 1413 行 |
| `RemoveOnDespawn` 实现 | `scripts/components/debuffable.lua` 第 39–49 行 |
| `IsEntityDeadOrGhost` 定义 | `scripts/componentutil.lua` 第 15–20 行 |
| `skip_test` / `pre_buff_fn` 用法 | `scripts/entityscript.lua` 第 2115–2129 行 |
| 经典固定时长 + 重置刷新模板 | `scripts/prefabs/foodbuffs.lua` 第 134–215 行 |
| 取较大值刷新 | `scripts/prefabs/yoth_buffs.lua` 第 17–23 行 |
| 时间叠加 + 视觉封顶 | `scripts/prefabs/wintersfeastbuff.lua` 第 78–97 行 |
| 行为打断式清除 | `scripts/prefabs/spawnprotectionbuff.lua` 第 47–76 行 |
| 互斥替换式 | `scripts/prefabs/ghostly_elixirs.lua` 第 348–367 行 |
| `Enable` 一键清空 | `scripts/components/debuffable.lua` 第 28–37 行 |
| `data.duration` + fallback | `scripts/prefabs/wx78_shadow_fuel_debuff.lua` 第 11–32、84 行 |

> **学习路径建议**：新手抓住"`timer` + `ListenForEvent("death")`"这两件事，就能写出 80% 的合格 buff；进阶玩家熟悉三种刷新策略（重置 / 取较大值 / 叠加）后，几乎所有现有 buff 都能改造；老手则要关注 `keepondespawn`、`Enable`、`TransferComponent`、`skip_test` 在断线/复活/换身这些边界场景里的交互。


## 22.3 SpellCaster 组件——法术释放的通用框架

> 涉及源码文件：
> - `scripts/components/spellcaster.lua`（SpellCaster 组件本体，全文仅 203 行）
> - `scripts/actions.lua` 第 433 行（`CASTSPELL` 动作定义）、第 3388–3417 行（`CASTSPELL` 服务端执行函数 `fn` / `strfn`）
> - `scripts/componentactions.lua` 第 2149–2178、2407–2429、2878–2891 行（三种使用场景下"右键能否触发法术"的过滤器）
> - `scripts/stategraphs/SGwilson.lua` 第 1234–1246、15814–15903、15905–15941、15943–15977 行（玩家服务端：`castspell` / `quickcastspell` / `veryquickcastspell` 三个状态）
> - `scripts/stategraphs/SGwilson_client.lua` 第 563–575、3958–4032 行（同样三个状态的客户端预测版本）
> - `scripts/constants.lua` 第 1964–1971 行（`SPELLTYPES` 表）
> - `scripts/prefabs/staff.lua` 第 887–911（紫法杖 teleport）、第 913–952（黄法杖 createlight）、第 954–974（绿法杖 destroystructure）、第 1013–1062 行（蛋白石法杖）
> - `scripts/prefabs/reskin_tool.lua` 第 207–327、367–399 行（综合范例：`SetCanCastFn`、`veryquickcast`、`canuseondead`）
> - `scripts/prefabs/staff_tornado.lua` 第 71–112 行（`quickcast` + `canonlyuseoncombat` + `canonlyuseonworkable` 组合）
> - `scripts/prefabs/horrorfuel.lua`、`scripts/prefabs/purebrilliance.lua`、`scripts/prefabs/wortox_reviver.lua`（`SetSpellType` 配合权限标签的典型场景）
> - `scripts/prefabs/trident.lua`、`scripts/prefabs/gnarwail_horn.lua`、`scripts/prefabs/pig_coin.lua`、`scripts/prefabs/wurt_terraform_item.lua`（其他法术物品参考）

### 一、为什么需要这套组件？（所有读者）

回想一下我们在游戏里"右键紫色法杖然后点猪人"这一连串操作背后到底发生了什么：

1. **客户端**先要决定"右键当前指向的目标，到底能不能触发某个动作"——这是 `componentactions.lua` 里的"右键过滤器"在管。
2. **客户端**会把这个动作发到服务端去做权威判定（同时本地播放一段假动作，让人感觉"按一下马上有反应"）。
3. **服务端**收到动作后，要再确认"这个目标真的合法吗？现在被攻击者真的能用这个东西吗？"，确认通过才真正生效。
4. 法杖本身要播放"举起来念咒"的动画，玩家要被锁在动画里不能走动，念完之后才真正"咒语生效"——比如生成一个传送动画、销毁建筑、放下灯柱。
5. 最后还可能有"用完一次"的耗损、消耗理智、消耗耐久度等收尾工作。

如果每写一根法杖都要把这 5 件事重做一遍，整个 `prefabs/` 文件夹早就被淹没了。`spellcaster` 组件的作用就是**把"是什么样的法术物品"和"释放法术时具体要做什么"完全解耦**：

- 你只要选好"能不能对地面用？能不能对生物用？能不能放进背包里用？要不要慢慢念咒还是快速挥一下？"几个开关；
- 然后注册一个"咒语本体函数"（接到合法目标后究竟要执行什么逻辑）；
- 剩下的——加什么标签、动作菜单要不要出现、播什么动画、念多久、要不要锁玩家、念完了播什么音效——**全部由 `spellcaster`、`CASTSPELL` 动作、`SGwilson` 这三处协作完成**，你完全不必碰。

这一节，我们会从源码层面把这条流水线拆开看一遍。

> 一个值得记下来的概念分工：**`spellcaster` 组件只回答"法术属于哪一类、能不能现在释放"，它本身既不播动画、也不耗耐久。动画和锁人由 `SGwilson` 状态机管，耐久、理智、特效全部由你写的 `spellFn` 自己管。**

### 二、新手篇：5 分钟上手 SpellCaster

这一节用最少的代码做一根可玩的"右键地面就召唤一道闪电"的简易法杖。你不需要理解组件内部，只要按四步抄就够了。

#### 2.1 最小四步走

```lua
-- 1) 准备一个 spell 函数：被释放时做点什么
local function spellfn(staff, target, pos, caster)
    SpawnPrefab("lightning"):Transform():SetPosition(pos:Get())
    if staff.components.finiteuses then
        staff.components.finiteuses:Use(1)
    end
end

-- 2) 在你的 fn() 服务端段落里挂组件（注意必须是 master sim）
if not TheWorld.ismastersim then
    return inst
end

-- 3) 挂组件
inst:AddComponent("spellcaster")
inst.components.spellcaster:SetSpellFn(spellfn)

-- 4) 配置"能在哪儿用"
inst.components.spellcaster.canuseonpoint = true  -- 允许右键陆地点位
```

这就完成了——装备它，然后对着地面右键，就会有一个动作选项"使用 [物品]"，确认后 13 帧左右挥一下，53 帧时调用你的 `spellfn`，整段动画 69 帧后结束。

#### 2.2 必须知道的 4 个"开关"

`spellcaster` 暴露出来的字段非常多，但**新手 80% 的需求只用得到这 4 个**：

| 字段 | 类型 | 作用 |
| --- | --- | --- |
| `canuseonpoint` | bool | 允许右键**陆地坐标**释放（任何可走的地块） |
| `canuseonpoint_water` | bool | 允许右键**水面坐标**释放（海洋格） |
| `canuseontargets` | bool | 允许右键**某个实体**释放（一定要配合下面的 `canonlyuse*` 决定"哪些实体"） |
| `canusefrominventory` | bool | 允许**直接在背包里点击图标**释放（不需要装备到手） |

四个字段都可以**多选**，比如"既可以对地放又可以对人放"，但是要清楚：**只有 `canuseontargets = true` 时，下面那一堆 `canonlyuse*` 才有意义**，不然实体过滤完全不生效。

#### 2.3 选哪些目标？—— `canonlyuseon*` 5 兄弟

紧接着 `canuseontargets`，组件提供了 5 个互补的"白名单开关"：

| 字段 | 含义 | 官方代表物品 |
| --- | --- | --- |
| `canonlyuseonrecipes` | 只能对**有配方的建筑/物品**用（用来"拆建筑"） | 绿色解构法杖 `greenstaff` |
| `canonlyuseonlocomotors` | 只能对**会移动的实体**用（带 `locomotor` 组件，比如生物、玩家） | —— |
| `canonlyuseonlocomotorspvp` | 在 `canonlyuseonlocomotors` 基础上加一层"对人 PVP 检查"——和平模式下不能对其他玩家用 | 紫色传送法杖 `purplestaff`、猪人金币 `pig_coin`、惊吓燃料 `horrorfuel` |
| `canonlyuseonworkable` | 只能对**可被砍/挖/锤/凿**的目标用 | 龙卷风法杖 `staff_tornado` |
| `canonlyuseoncombat` | 只能对**能被自己作为战斗目标**的实体用（调用 `combat:CanTarget`） | 龙卷风法杖 `staff_tornado` |

> 五个白名单是**或**关系：满足其一即合法。`staff_tornado` 同时开了 `canonlyuseonworkable` 和 `canonlyuseoncombat`，意思是"既能对树/矿石用，也能对怪用"。

#### 2.4 一个对得起官方质量的小完整例子

```lua
local assets = { Asset("ANIM", "anim/staffs.zip") }
local prefabs = { "lightning" }

local function spellfn(staff, target, pos, caster)
    SpawnPrefab("lightning").Transform:SetPosition(pos:Get())
    if caster ~= nil and caster.components.sanity ~= nil then
        caster.components.sanity:DoDelta(-TUNING.SANITY_MED)
    end
    staff.components.finiteuses:Use(1)
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()
    MakeInventoryPhysics(inst)
    inst.AnimState:SetBank("staffs")
    inst.AnimState:SetBuild("staffs")
    inst.AnimState:PlayAnimation("yellowstaff")
    inst:AddTag("nopunch")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("equippable")
    inst.components.equippable.equipslot = EQUIPSLOTS.HANDS
    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(20)
    inst.components.finiteuses:SetUses(20)
    inst.components.finiteuses:SetOnFinished(inst.Remove)

    inst:AddComponent("spellcaster")
    inst.components.spellcaster:SetSpellFn(spellfn)
    inst.components.spellcaster.canuseonpoint = true   -- 对陆地点位生效

    return inst
end

return Prefab("mylightningstaff", fn, assets, prefabs)
```

读它一遍，你就掌握了 `spellcaster` 90% 的常见用法。

#### 2.5 新手最容易踩的 5 个坑

1. **不写 `canuseonpoint` / `canuseontargets` / `canusefrominventory` 中任何一个**——右键根本不会出现"使用"选项，你以为"组件挂错了"，其实是没告诉它"在哪儿能用"。
2. **`spellFn` 第一个参数不是 caster，是 staff 本身**。签名顺序是 `(inst, target, pos, doer)`，新手最常见的错误就是把第一个当玩家。源码见 `spellcaster.lua:134-142`。
3. **不要在客户端调 `CastSpell`**。整个流程都是服务端权威——客户端只决定"显示动作选项 / 播预测动画"，真正的 `CastSpell` 是 `actions.lua:3411` 在服务端调用的。
4. **耐久、理智、特效是 spellFn 自己的活**。组件不会替你扣 `finiteuses`，也不会扣理智。如果你看到一根官方法杖"用一下掉一点耐久"，那一定是它自己的 `spellFn` 写的（搜 `createlight`、`destroystructure` 印证）。
5. **改 `canuseontargets` 等开关时，标签会自动同步**。比如你写 `inst.components.spellcaster.canuseonpoint = true`，组件内部会自动给实体 `AddTag("castonpoint")`，**不要再手动加同名标签**，否则你 set 回 false 时反而清不掉。源码见 `spellcaster.lua:1-46`（`oncancast` 钩子）。

### 三、进阶篇：组件结构 + 动作流水线全解

> 这一节适合已经写过 prefab、知道 `componentactions`、`actions.lua`、`stategraph` 三件套各自负责什么的同学。我们用 4 个小节把 SpellCaster 整条流水线串起来：组件字段 → 标签机制 → `CASTSPELL` 动作 → 玩家状态机。

#### 3.1 SpellCaster 组件的字段全景

源码 `scripts/components/spellcaster.lua` 第 67–103 行：

```lua
local SpellCaster = Class(function(self, inst)
    self.inst = inst
    self.onspellcast = nil               -- 在 spell 之后再执行一个"后置回调"
    self.canusefrominventory = false     -- 是否可"在背包里直接点"
    self.canuseontargets = false         -- 是否可对实体释放
    self.canuseondead = false             -- 是否允许目标"已死"还能被释放
    self.canonlyuseonrecipes = false     -- 仅对有配方的对象（解构系列）
    self.canonlyuseonlocomotors = false  -- 仅对会移动的实体
    self.canonlyuseonlocomotorspvp = false -- 同上但加 PVP 限制
    self.canonlyuseonworkable = false    -- 仅对 CHOP/DIG/HAMMER/MINE 可工作目标
    self.canonlyuseoncombat = false      -- 仅对可作为战斗目标的对象
    self.canuseonpoint = false           -- 是否对陆地点位
    self.canuseonpoint_water = false     -- 是否对水面点位
    self.spell = nil                     -- 真正的"咒语本体" function
    self.quickcast = false                -- 启用"挥一下"型快速咏唱
    self.veryquickcast = false           -- 启用"瞬发"型极快咏唱
    self.spelltype = nil                 -- 法术类型字符串（权限分组）
    --self.can_cast_fn = nil             -- 自定义额外检查（注释掉是因为是可选）
end, nil, {
    spell = oncancast,
    canusefrominventory = oncancast,
    canuseontargets = oncancast,
    canonlyuseonrecipes = oncancast,
    canonlyuseonlocomotors = oncancast,
    canonlyuseonlocomotorspvp = oncancast,
    canonlyuseonworkable = oncancast,
    canonlyuseoncombat = oncancast,
    canuseonpoint = oncancast,
    canuseonpoint_water = oncancast,
    quickcast = onquickcast,
    veryquickcast = onveryquickcast,
    spelltype = onspelltype,
})
```

注意构造函数末尾的那张 setter watch 表——这是 EntityScript Class 系统的**字段写入回调**：每次你给这些字段赋新值时，相应的回调（`oncancast` / `onquickcast` / `onveryquickcast` / `onspelltype`）会**立即**被触发，把对应的标签同步到实体身上。

公共方法只有 5 个：

| 方法 | 签名 | 作用 |
| --- | --- | --- |
| `SetSpellFn(fn)` | `fn(inst, target, pos, doer)` | **必填**，定义"咒语真正做什么" |
| `SetOnSpellCastFn(fn)` | `fn(inst, target, pos, doer)` | 可选，在 spell 之后再额外跑一段（比如统一播音效） |
| `SetCanCastFn(fn)` | `fn(doer, target, pos, inst) → bool, reason` | 可选，**附加**自定义合法性判定 |
| `SetSpellType(t)` | `t : string \| nil` | 标记法术类型，配合 `_spelluser` 标签实现权限 |
| `CastSpell(target, pos, doer)` | —— | **由 `CASTSPELL` 动作调用**，不要手动调 |
| `CanCast(doer, target, pos)` | → bool, reason | **由 `CASTSPELL` 动作调用**，用来回答"现在能不能释放" |
| `OnRemoveFromEntity` | —— | 组件被卸载时清掉所有标签 |

`CastSpell` 的实现极简（`spellcaster.lua:134-142`）：

```lua
function SpellCaster:CastSpell(target, pos, doer)
    if self.spell ~= nil then
        self.spell(self.inst, target, pos, doer)
        if self.onspellcast ~= nil then
            self.onspellcast(self.inst, target, pos, doer)
        end
    end
end
```

不做任何"耐久 -1、理智 -X"——这些副作用全在你自己的 `spell` 函数里。

#### 3.2 标签机制：组件如何"告诉世界"自己能干嘛

`oncancast` 钩子（`spellcaster.lua:1-47`）是整套机制最巧妙的一段。每当上面 11 个开关变化时，它会自动把对应的标签同步进 / 出实体：

| 字段 | 同步的标签 |
| --- | --- |
| `canusefrominventory` | `castfrominventory` |
| `canuseontargets`（无任何 `canonlyuse*`）| `castontargets` |
| `canonlyuseonrecipes` | `castonrecipes` |
| `canonlyuseonlocomotors` | `castonlocomotors` |
| `canonlyuseonlocomotorspvp` | `castonlocomotorspvp`（且自动**互斥**地清掉 `castonlocomotors`） |
| `canonlyuseonworkable` | `castonworkable` |
| `canonlyuseoncombat` | `castoncombat` |
| `canuseonpoint` | `castonpoint` |
| `canuseonpoint_water` | `castonpointwater` |
| `quickcast` | `quickcast` |
| `veryquickcast` | `veryquickcast` |
| `spelltype = "wurt_lunar"` 等 | `wurt_lunar_spellcaster`（用 `_spellcaster` 后缀） |

这些标签**正是 `componentactions.lua` 用来判定"右键要不要弹出 CASTSPELL 选项"的依据**。换句话说：**`componentactions.lua` 根本不读 `spellcaster` 组件的内部字段，它只读 `inst:HasTag("xxx")`**——这是一种典型的"用标签做组件 → 客户端通信"的模式。客户端没有组件实例，只能读标签。

> 这也是为什么有些 prefab 会写 `inst:AddTag("veryquickcast")` 在 pristine state 里再加一次（`reskin_tool.lua:370`）——为了让**客户端在 SetPristine 之后**就已经拥有这个标签，否则联机刚连进来一两帧内可能客户端那边 `componentactions` 拿不到，会闪一下没动作。

#### 3.3 `CASTSPELL` 动作：从右键到 `CastSpell` 之间发生了什么

整条流水线如下（按时间先后顺序）：

**① 客户端：右键过滤器决定要不要列动作**

`componentactions.lua` 里 `spellcaster` 这个 component handler 出现了 3 次，对应玩家鼠标可能停留的 3 种场景：

- **POINT**（`componentactions.lua:2149-2178`）——右键点的是空地或水面。它会读 `castonpoint` / `castonpointwater` 标签判定。
- **EQUIPPED → target**（`componentactions.lua:2407-2429`）——右键点的是某个实体，并且当前法术物品**在手上**。它读 `castontargets` / `castonrecipes` / `castonlocomotors` / `castonworkable` / `castoncombat` 这一系列。注意它还多了一步 `not target:HasTag("nomagic")` 检查——任何带 `nomagic` 标签的实体永远拒绝被法术指向，这是给一些"不可对其用魔法"的特殊实体准备的逃生通道。
- **INVENTORY**（`componentactions.lua:2878-2891`）——右键的是背包里的图标本身（没有目标）。它只看 `castfrominventory`。

注意三段开头都有一段相同代码：

```lua
for k,v in pairs(SPELLTYPES) do
    if inst:HasTag(v.."_spellcaster") and not doer:HasTag(v.."_spelluser") then
        return
    end
end
```

意思是：**只要这件物品声明了 spelltype，但 doer 不持有对应的 `_spelluser` 标签，就直接 return**——动作菜单根本不会冒出来。这是 SpellType 的客户端短路。

**② 客户端 → 服务端：发送 BufferedAction**

玩家选中 `CASTSPELL` 动作后，`playercontroller` 会把它送给本地 stategraph，同时通过网络发到服务端。两边的 stategraph 都会按 `ActionHandler(ACTIONS.CASTSPELL, ...)` 分流：

```lua
ActionHandler(ACTIONS.CASTSPELL,
    function(inst, action)
        return action.invobject ~= nil
            and ((action.invobject:HasTag("gnarwail_horn") and "play_gnarwail_horn")
                or (action.invobject:HasTag("guitar") and "play_strum")
                or (action.invobject:HasTag("cointosscast") and "cointosscastspell")
                or (action.invobject:HasTag("crushitemcast") and "crushitemcast")
                or (action.invobject:HasTag("quickcast") and "quickcastspell")
                or (action.invobject:HasTag("veryquickcast") and "veryquickcastspell")
                or (action.invobject:HasTag("mermbuffcast") and "mermbuffcastspell"))
            or "castspell"
    end),
```

这个分流表是 SpellCaster 教程里**最经常被新手忽略**的一段。它的意思是：**法杖的标签决定播什么动画**：

| 标签 | 进入状态 | 动画特点 |
| --- | --- | --- |
| `quickcast` | `quickcastspell` | 21 帧动画，5 帧时 `PerformBufferedAction`，16 帧后解除 busy |
| `veryquickcast` | `veryquickcastspell` | 21 帧动画，9 帧时同步 `PerformBufferedAction` 和解 busy |
| 默认（无标签） | `castspell` | 69 帧"举杖咏唱"，53 帧时调用 spell，玩家在 13–69 帧之间 `playercontroller:Enable(false)` 被锁定 |
| `gnarwail_horn` | `play_gnarwail_horn` | 这就是为什么 `gnarwail_horn` 用 spellcaster 而不是 weapon，它要走自己的动画 |
| `cointosscast` | `cointosscastspell` | 抛硬币的特殊状态 |
| `mermbuffcast` | `mermbuffcastspell` | 沃特的 buff 法术 |

**③ 服务端：状态机 `PerformBufferedAction` → `actions.lua` 的 fn**

进入状态后，到了状态时间线指定的某个 TimeEvent，玩家调用 `inst:PerformBufferedAction()`，这一步真正进入 `actions.lua:3392-3417`：

```lua
ACTIONS.CASTSPELL.fn = function(act)
    local staff = act.invobject or act.doer.components.inventory:GetEquippedItem(EQUIPSLOTS.HANDS)
    local act_pos = act:GetActionPoint()
    if staff and staff.components.spellcaster then
        if ShouldItemMimicBeRevealedFor(staff, act.doer) then
            return false, "ITEMMIMIC"  -- 物品其实是怪物伪装
        end
        if staff:HasTag("crushitemcast") then
            -- crushitem 系列：骑乘或搬重物时不能用
            ...
        end
        local can_cast, cant_cast_reason = staff.components.spellcaster:CanCast(act.doer, act.target, act_pos)
        if can_cast then
            staff.components.spellcaster:CastSpell(act.target, act_pos, act.doer)
            return true
        else
            return can_cast, cant_cast_reason
        end
    end
end
```

注意两点：

- **服务端会再次调用 `CanCast` 做权威检查**。客户端 `componentactions` 只是"乐观假设"，服务端这一关才说了算——这能防止"客户端篡改"或"客户端和服务端状态不一致"的法术滥用。
- **act.invobject 可能为 nil**。如果是空的，会回退到当前装备的手持物——这就是为什么 `quickcast` / `veryquickcast` 即使 invobject 没被传也能工作。

**④ `CanCast` 的判定流程**（`spellcaster.lua:151-200`）

```lua
function SpellCaster:CanCast(doer, target, pos)
    if self.spell == nil then return false end  -- 没设置 spell：直接不行
    if self.spelltype ~= nil and not doer:HasTag(self.spelltype.."_spelluser") then
        return false                             -- SpellType 权限不通过
    end
    if target == nil then
        if pos == nil then
            return self.canusefrominventory      -- 背包模式：只看一个开关
        else
            -- 地面/水面模式
            local can_cast, cast_fail_reason = true, nil
            if self.can_cast_fn ~= nil then
                can_cast, cast_fail_reason = self.can_cast_fn(doer, nil, pos, self.inst)
            end
            if not can_cast then return can_cast, cast_fail_reason end
            if self.canuseonpoint then
                local px, py, pz = pos:Get()
                return TheWorld.Map:IsAboveGroundAtPoint(px, py, pz, self.canuseonpoint_water)
                    and not TheWorld.Map:IsGroundTargetBlocked(pos)
            elseif self.canuseonpoint_water then
                return TheWorld.Map:IsOceanAtPoint(pos:Get())
                    and not TheWorld.Map:IsGroundTargetBlocked(pos)
            else
                return false
            end
        end
    elseif target:IsInLimbo()
        or not target.entity:IsVisible()
        or (target.components.health ~= nil and target.components.health:IsDead() and not self.canuseondead)
        or (target.sg ~= nil and (...)) then
        return false                              -- 目标处在某种无效状态
    else
        return self.canuseontargets and (
            (self.canonlyuseonrecipes and AllRecipes[target.prefab] ~= nil and ...) or
            (target.components.locomotor ~= nil and (
                (self.canonlyuseonlocomotors and not self.canonlyuseonlocomotorspvp) or
                (self.canonlyuseonlocomotorspvp and (target == doer or TheNet:GetPVPEnabled() or not (target:HasTag("player") and doer:HasTag("player"))))
            )) or
            (self.canonlyuseonworkable and target.components.workable ~= nil and target.components.workable:CanBeWorked() and IsWorkAction(target.components.workable:GetWorkAction())) or
            (self.canonlyuseoncombat and doer.components.combat ~= nil and doer.components.combat:CanTarget(target)) or
            (self.can_cast_fn ~= nil and self.can_cast_fn(doer, target, pos, self.inst))
        )
    end
end
```

可以把它读成下面这张决策树：

```
spell == nil              → false
spelltype 不通过           → false
target == nil
  ├─ pos == nil           → 看 canusefrominventory
  └─ pos ≠ nil
      ├─ can_cast_fn 拒绝 → false（带 reason）
      ├─ canuseonpoint    → 地面是陆地且未被堵
      ├─ canuseonpoint_water → 地面是海洋且未被堵
      └─ 都没开           → false
target 处在不可被选状态     → false
其它                     → canuseontargets AND (recipes OR locomotor OR workable OR combat OR can_cast_fn)
```

特别注意 `target.sg:HasStateTag("nospellcasting")` 这一条——任何状态机带 `nospellcasting` 标签的目标都拒绝被指向。当前源码里只有 `SGmerm.lua` 的"刚刚起床（gettingup）/变身（transforming）"两个状态用了这个标签，意味着**沃特/沃斯托克对正在变身的鱼人不能下咒**。

**⑤ 状态机播完动画 → 状态结束**

进入 `castspell` 状态后流程如下（以最复杂的"标准咏唱"为例，源码 `SGwilson.lua:15814-15903`）：

```
onenter:
  playercontroller:Enable(false)             -- 锁玩家
  AnimState:PlayAnimation("staff_pre")
  AnimState:PushAnimation("staff", false)
  locomotor:Stop()
  生成 staffcastfx / staffcastfx_mount        -- 法杖光晕
  生成 staff_castinglight                    -- 短暂场地光
  如果 staff 有 aoetargeting，生成目标 fx
  记下要播什么音效（staff.castsound 优先）

timeline:
  13 帧:  SoundEmitter:PlaySound(castsound)   -- 咏唱音效
  53 帧:  PerformBufferedAction()             -- 真正调 actions.lua: CASTSPELL.fn → CastSpell
  69 帧:  RemoveStateTag("busy")
          playercontroller:Enable(true)       -- 解锁玩家

events:
  animqueueover: GoToState("idle")

onexit:
  playercontroller:Enable(true)               -- 保底解锁
  清除 stafffx / stafflight / targetfx
```

**这就是为什么 `staff.castsound` 字段（紫法杖里赋的 `dontstarve/common/staffteleport`）会有效——状态机会主动查 staff 上的这个字段。** 同样地，`staff.fxcolour`（紫法杖赋的紫色三元组）也被 `castspell` 状态在 `onenter` 时读取并塞给 `staffcastfx:SetUp(colour)`，让法杖光晕的颜色与法杖本身匹配。这是 SpellCaster 一个**隐性约定**：你只要在法杖 prefab 上挂这两个字段，状态机会自动用上。

#### 3.4 `quickcast` vs `veryquickcast` 是什么

简单对比一下三个状态的"时间窗"（FRAMES = 1/30 秒）：

| 状态 | 真正释放法术的帧 | 解 busy 的帧 | 状态机播放动画 | 锁玩家移动？ |
| --- | --- | --- | --- | --- |
| `castspell`（默认） | **第 53 帧**（≈1.77 s） | 第 69 帧（≈2.30 s）| `staff_pre + staff` | **是**（13–69 帧之间 `playercontroller:Enable(false)`） |
| `quickcastspell` | **第 5 帧**（≈0.17 s）| 第 16 帧 | `atk_pre + atk`（普通挥武器） | 否，但 busy 5–16 帧 |
| `veryquickcastspell` | **第 9 帧**（同一帧解 busy）| 第 9 帧 | `atk_pre + atk` | 否，几乎"瞬发" |

应用场景：

- 龙卷风法杖（`staff_tornado`）走 `quickcast`，因为它是"快速挥一下"的法术武器，不是"举杖念咒"。
- 时装工具（`reskin_tool`）走 `veryquickcast`，因为它仅仅是"换皮肤"，没必要锁玩家。
- 其他绝大多数有"我在念咒"感觉的法杖（紫法杖、绿法杖、黄法杖）都走默认 `castspell`。

> 一个隐藏陷阱：`quickcastspell` / `veryquickcastspell` **没有**像 `castspell` 那样去读 `staff.castsound`，而是直接读 `inst.SoundEmitter:PlaySound((staff ~= nil and staff.castsound) or "dontstarve/wilson/attack_weapon")`——也就是说，如果你的法杖**没设**`castsound`，会播默认的武器音。这点写新法杖时别忘了赋值。

### 四、老手篇：SpellType、链路插桩与边界场景

#### 4.1 SpellType：用一行代码做权限组

`SPELLTYPES` 在 `constants.lua:1964` 注册了 5 个固定常量：

```lua
SPELLTYPES = -- NOTES(JBK): Keep this table updated in export_accountitems.lua [EAITAB]
{
    WURT_SHADOW = "wurt_shadow",
    WURT_LUNAR = "wurt_lunar",
    SHADOW_SWAMP_BOMB = "shadow_swamp_bomb",
    LUNAR_SWAMP_BOMB = "lunar_swamp_bomb",
    WORTOX_REVIVER_LOCK = "wortox_reviver_lock", -- Inverted and stops allowing to cast.
}
```

机制非常简洁：

- 你对物品调用 `spellcaster:SetSpellType("wurt_shadow")`，组件会在物品上加 `wurt_shadow_spellcaster` 标签。
- 玩家身上必须有 `wurt_shadow_spelluser` 标签，物品才会出现动作选项 / 才能真正释放。
- 添加 / 移除"spelluser"标签的，通常是技能树或者 buff——`skilltree_wurt.lua:377` 就是当沃特点出"暗影法术"天赋时给玩家加 `wurt_shadow_spelluser`。

注意最后那一条 `WORTOX_REVIVER_LOCK = "wortox_reviver_lock", -- Inverted and stops allowing to cast.`——这是一个**反向锁**：它的用法是"当某个条件成立时把物品的 spelltype 设成它，玩家身上没人会有 `wortox_reviver_lock_spelluser` 标签，所以物品永远不能释放"。沃克斯的复活肉块在 `wortox_reviver.lua:63-68` 用这个机制实现"还没充能好的时候不能用"：

```lua
local function SetAllowConsumption(inst, allow)
    if inst.components.spellcaster then
        inst.components.spellcaster:SetSpellType(not allow and SPELLTYPES.WORTOX_REVIVER_LOCK or nil)
    end
end
```

为 mod 设计权限：

- **加新 SpellType**：覆写 `SPELLTYPES`（注意一定要在游戏加载早期，建议放 `modmain` 顶部）即可。但官方注释 `[EAITAB]` 标明它和 `export_accountitems.lua` 联动，所以更稳妥的做法是**反向锁**：把 `spelltype = "<我自己的常量>"`，给授权角色加 `<我自己的常量>_spelluser` 标签即可。
- **临时禁用法术**：临时 `inst.components.spellcaster:SetSpellType("locked_by_mod")` 实现"一次性禁用"。释放许可时 `SetSpellType(nil)` 恢复。

#### 4.2 `SetCanCastFn`：精细化的合法性判定

注意 `CanCast` 决策树里，`can_cast_fn` 出现在两个位置：

1. **target == nil 且 pos ≠ nil 的"对地施法"分支**——`can_cast_fn` 会**先于**地图合法性检查触发，是一道"前置否决"。
2. **target ≠ nil 的"对实体施法"分支**——`can_cast_fn` 是**白名单中的最后一条**（OR），用来扩展"标签都不满足但我想让它合法"的特例。

`reskin_tool.lua:284-327` 是一个**最经典**的 `can_cast_fn` 用法。它做了如下检查：

- target 上有 `reskin_tool_target_redirect` 字段时，重定向到真正目标。
- target 是其他玩家时拒绝（仅 owner 可换皮肤）。
- target 有 `reskin_tool_cannot_target_this` 时拒绝。
- 若 target 是当前玩家自身，检查他的胡须是不是可换皮的。
- 若用户拥有皮肤或 target 当前已应用皮肤可关闭，则允许；否则拒绝。

这段代码的关键洞察是：**`can_cast_fn` 不读组件的任何字段，可以完全替代 `canonlyuse*` 这一整套硬编码标签**。如果你写一个"只能对'插了树苗的农作物'用的肥料法术"，硬编码 5 个标签里没有适合的，那么直接 `SetCanCastFn` 比试图扩展硬编码更干净。

> 一个老手的小技巧：**`can_cast_fn` 的返回值可以是 `(false, "REASON_STRING")` 两值**。`actions.lua:3414` 会原样把 reason 透回去，让你能展示"为什么不能施法"的具体说话气泡。要在 `STRINGS.ACTIONFAIL.CASTSPELL.REASON_STRING` 里登记字符串。

#### 4.3 与 `aoetargeting` / `reticule` / `equippable` 的协同

SpellCaster 不和这些组件互斥，它们各管一段：

- **`reticule` 组件**：负责"右键地面时显示一个准星"。注意准星只是**预测**目标点，真正能不能施法仍由 `CanCast` 决定。`staff.lua` 里的紫色法杖没有 `reticule`，而黄色法杖有——这是因为黄色法杖目标点高度灵活，要给玩家视觉引导。
- **`aoetargeting` 组件**：声明"我是一个大范围攻击"。在 `castspell` 状态 `onenter` 里被读取，用来生成范围预警 fx（`SGwilson.lua:15838-15846`）。
- **`equippable`**：决定法杖能不能装备到手上、装上之后玩家走得快不快等——和 SpellCaster 完全正交。
- **`finiteuses` / `fueled`**：耐久和燃料完全由 spellFn 自己消耗。`spellcaster` 不会自动扣。
- **`combat`**：`canonlyuseoncombat` 间接借用了 doer 的 `combat:CanTarget`，但 SpellCaster 自己不带战斗逻辑。
- **`weapon`**：对于"既能打又能放法术"的物品（橘色法杖、龙卷风法杖、紫色法杖打）来说，`weapon` 和 `spellcaster` 同时挂——左键走 `weapon` 的攻击通路，右键走 `CASTSPELL` 通路，互不干扰。

`reskin_tool.lua:392-398` 是把 `equippable`、`spellcaster`、`fuel` 三件套挂一起的范式：

```lua
inst:AddComponent("spellcaster")
inst.components.spellcaster.canuseontargets = true
inst.components.spellcaster.canuseondead = true       -- 允许给"死掉的玩家"换皮
inst.components.spellcaster.veryquickcast = true       -- 走瞬发
inst.components.spellcaster.canusefrominventory = true -- 在背包里直接点也能换皮
inst.components.spellcaster:SetSpellFn(spellCB)
inst.components.spellcaster:SetCanCastFn(can_cast_fn)
```

注意它**同时**开启了 `canuseontargets` 和 `canusefrominventory`——意思是"你可以对一个实体右键换皮，也可以在背包里点击换自己胡须"。这是为什么 `canonlyuse*` 五个开关全部没有打开：`can_cast_fn` 接管了实体筛选。

#### 4.4 边界场景速查表

整理一份"特殊情况你应该这么做"清单：

| 你想要 | 写法 |
| --- | --- |
| 对已死目标也能施法（如复活、换皮）| `canuseondead = true`（`reskin_tool` / `wortox_reviver` 都开了） |
| 法杖在骑乘 / 搬重物时禁用 | 给物品 `AddTag("crushitemcast")`，`componentactions` 会自动忽略 |
| 让一群法杖按"是不是某个角色的专属"分组 | 配合 `SetSpellType(...)` + `_spelluser` 标签 |
| 对某种地形（非陆非海）严格控制 | 用 `SetCanCastFn` 自己判定 `TheWorld.Map:GetTileAtPoint(pos:Get())` |
| 法杖被丢/吃完后自动清理 | 组件自带 `OnRemoveFromEntity`，会清光所有标签 |
| 给同一根法杖临时改 spellFn（充能/未充能不同行为）| `inst.components.spellcaster:SetSpellFn(...)` 随时切（见 `wurt_terraform_item.lua:104-112`） |
| 想让 spellFn 抛出的额外副作用走统一日志 | 注册 `SetOnSpellCastFn`——它会在 spellFn 之后被无条件调用 |
| **千万不要**：手动 `AddTag("castontargets")` | 该标签是组件管理的；你手动加之后 set false 时组件不会清掉 |

#### 4.5 一个老手才会注意的"死字段"

`staff_tornado.lua:111` 有一行：

```lua
inst.components.spellcaster.castingstate = "castspell_tornado"
```

但是**搜遍整个 `scripts/`，没有一处读这个字段**。它不会让玩家进入 `castspell_tornado` 状态——龙卷风法杖之所以走"挥一下"是因为它带 `quickcast` 标签。

这一行在 1.99 版本之后就是死代码，留着的原因可能是历史保留。**不要在你自己的法杖里照抄它**。要自定义状态名，正确做法是：

1. 给法杖加一个独有标签，例如 `mytornadocast`。
2. 在 `SGwilson.lua` 的 `ActionHandler(ACTIONS.CASTSPELL, ...)` 表里加一行 `or (action.invobject:HasTag("mytornadocast") and "my_tornado_state")`（mod 里要用 `AddStategraphActionHandler` 注入）。
3. 自己定义一个 `my_tornado_state` 状态。

源码 `SGwilson.lua:1234-1246` 是分流表的所有合法标签——`mod` 里加新分支必须遵守这套约定。

### 五、老手实战：用源码验证一根新法杖的全链路

让我们用一根虚构的"奶酪法杖"（右键奶酪能让一个怪物"被臭晕"，并扣施法者饥饿）走一遍全链路验证。设计要求：

- 只能对生物用，不能对玩家（避免 PVP 问题）。
- 念咒时间走默认 `castspell`（看起来更像法术）。
- 念咒结束玩家饥饿 -10。
- 法杖只有 3 次使用机会。

```lua
local function cheesespell(staff, target, pos, caster)
    target.components.sleeper:AddSleepiness(5, 10, staff)
    if caster.components.hunger then
        caster.components.hunger:DoDelta(-10)
    end
    staff.components.finiteuses:Use(1)
end

local function fn()
    local inst = CreateEntity()
    -- ...省略基础设置...
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("equippable")
    inst.components.equippable.equipslot = EQUIPSLOTS.HANDS
    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(3)
    inst.components.finiteuses:SetUses(3)
    inst.components.finiteuses:SetOnFinished(inst.Remove)

    inst.fxcolour = { 1, 0.85, 0.2 }
    inst.castsound = "dontstarve/wilson/use_gemstaff"

    inst:AddComponent("spellcaster")
    inst.components.spellcaster:SetSpellFn(cheesespell)
    inst.components.spellcaster.canuseontargets = true
    inst.components.spellcaster.canonlyuseonlocomotorspvp = true  -- 用 pvp 兼容版避免误伤队友

    return inst
end
```

照下表逐条对应源码自检：

| 检查项 | 源码位置 | 我的法杖 |
| --- | --- | --- |
| 必须 `SetSpellFn` 否则 `CanCast` 返回 false | `spellcaster.lua:152` | ✅ 已设置 |
| 必须开 `canuseontargets` 否则实体白名单永远 false | `spellcaster.lua:189` | ✅ 开启 |
| 选 `canonlyuseonlocomotorspvp` 防 PVP 误伤 | `spellcaster.lua:193` | ✅ 选了 PVP 版 |
| 默认走 `castspell` 念咒，能读 `castsound` | `SGwilson.lua:15849` | ✅ 已赋 `inst.castsound` |
| 默认走 `castspell`，能读 `fxcolour` 染色光晕 | `SGwilson.lua:15828` | ✅ 已赋 `inst.fxcolour` |
| spellFn 第一个参数是 staff 不是 caster | `spellcaster.lua:136` | ✅ `cheesespell(staff, target, pos, caster)` |
| `componentactions` 会因 `castontargets` + `castonlocomotorspvp` 标签自动列出"使用 X"动作 | `componentactions.lua:2415-2422` | ✅ 标签由组件自动加 |
| 服务端 `CASTSPELL.fn` 二次 `CanCast` 校验 | `actions.lua:3409` | ✅ 自动 |

最后再用源码思考几个边界：

- **目标是不能行动的生物（无 `locomotor`）**：`spellcaster.lua:191` 的 `target.components.locomotor ~= nil` 直接断在这里——奶酪法杖不会对杂物生效，符合"对生物"的预期。
- **目标已死**：`spellcaster.lua:180` 拒绝（因为没开 `canuseondead`）。这一条要在策划层确认"死掉的鱼人能不能再臭晕"，如果允许，加 `canuseondead = true` 即可。
- **目标正在变身（`nospellcasting` 状态标签）**：`spellcaster.lua:185` 拒绝——符合"变身中无法施法"。
- **PVP 模式下队友 vs 敌方玩家**：`spellcaster.lua:193` 的 `canonlyuseonlocomotorspvp` 三段判定：`target == doer`（自己）→ 允许；`TheNet:GetPVPEnabled()` → 允许；`not (target:HasTag("player") and doer:HasTag("player"))` → 允许，否则拒。即"PVP 关闭时玩家间互相不许施"。

如果实测时有怪不被臭晕，**唯一会出问题的地方**是 spellFn——`target.components.sleeper` 不一定每只怪都有，需要先 `if target.components.sleeper then` 兜底。

### 六、本节回顾与学习路径

整套 SpellCaster 流水线可以归纳成一张图：

```
[客户端] componentactions 用标签筛 → 弹"使用 X"菜单
        ↓
[网络] BufferedAction.CASTSPELL 上行
        ↓
[服务端 + 客户端] SGwilson ActionHandler 分流到 castspell / quickcastspell / veryquickcastspell
        ↓
[服务端] 状态机 timeline 在固定帧 PerformBufferedAction
        ↓
[服务端] actions.lua CASTSPELL.fn 二次 CanCast 权威判定
        ↓
[服务端] spellcaster:CastSpell → 你写的 spellFn → onspellcast 后置
```

不同读者的学习侧重：

- **新手**：把 2.4 的最小例子抄一遍并改 4 个开关字段就能写出 80% 实用的小法杖；记住"客户端不调 CastSpell"和"耐久要自己扣"两条就足够。
- **进阶**：吃透"组件 → 标签 → componentactions → 状态机"这条单向同步链路，所有"为什么我加了组件但右键没动作"的疑问都能自答。
- **老手**：把 `SetCanCastFn` 和 `SetSpellType` 当作你的核心扩展点，所有"加 SPELLTYPES 常量"和"扩展 SGwilson 分流表"必须配合 mod 注入接口（`AddStategraphActionHandler` / `AddStategraphState`），不要直接编辑文件。同时记住"`castingstate` 是死字段"这种陷阱，看到不要照抄。

> **学习路径建议**：先用 2.4 的最小例子改出 3 根"对地、对生物、可在背包里直接用"的法杖练手；然后阅读 `staff.lua` 里紫/黄/绿三根法杖（覆盖三个最常见的开关组合）；最后翻 `reskin_tool.lua` 一整套以 `SetCanCastFn` 完全替代硬编码白名单的写法。下一节会带你进入 `spell` 组件——它是法术书 + 多法术管理的核心，和 `spellcaster` 是两条不同的法术线。


## 22.4 Spell 组件与 SpellBook——法术书与多法术管理

> 涉及源码文件：
> - `scripts/components/spell.lua`（Spell 组件本体，全文 147 行）
> - `scripts/components/spellbook.lua`（SpellBook 组件本体，全文 155 行）
> - `scripts/actions.lua` 第 630–632 行（`USESPELLBOOK` / `CLOSESPELLBOOK` / `CAST_SPELLBOOK` 三个动作定义）、第 5844–5942 行（三动作的 fn / strfn / pre_action_cb）
> - `scripts/componentactions.lua` 第 814–833、2867–2876 行（spellbook 的两个 component handler）
> - `scripts/stategraphs/SGwilson.lua` 第 1624–1629 行（`CAST_SPELLBOOK` ActionHandler）、第 9892–10250 行（`book` / `book2` 两个状态）
> - `scripts/screens/playerhud.lua` 第 1236–1320 行（`OpenSpellWheel` / `CloseSpellWheel` / `GetCurrentOpenSpellBook` 三个 HUD 接口）
> - `scripts/prefabs/wormlight.lua` 第 28–50、298–326 行（最干净的 `Spell` 组件用例）
> - `scripts/prefabs/waxwelljournal.lua` 第 244–340、615–700 行（最完整的 `SpellBook` + `aoespell` 多法术示例）
> - `scripts/prefabs/willow_ember.lua` 第 925–955 行、`scripts/prefabs/abigail_flower.lua` 第 311–331 行、`scripts/prefabs/winona_remote.lua` 第 660–693 行（其它三本"法术书"对比）
> - `scripts/prefabs/walter.lua` 第 274–320 行、`scripts/prefabs/wobycommon.lua` 第 510–543 行（特殊：挂在角色 / 宠物身上的 spellbook）

### 一、为什么把 Spell 和 SpellBook 放一起讲？（所有读者）

这两个组件名字都带 "spell"，但**承担完全不同的工作**——它们也和上一节的 SpellCaster 都不是同一回事。我先把三者拎清楚：

| 组件 | 挂在什么实体上？ | 负责什么？ | 类比 |
| --- | --- | --- | --- |
| `spellcaster`（22.3） | "释放法术的物品"（法杖、咒符、惊吓燃料）| **判定**能不能释放，**触发**一次性的咒语函数 | 法术 _发射器_ |
| `spell`（本节） | "被释放出来的效果实体"（如发光萤虫附身的隐形实体）| **维持**一段时间内每帧做点什么，到期自动结束 | 法术 _引信_ |
| `spellbook`（本节）| "一本可以列出多个法术的物品"（沃尔特的口令本、麦斯威尔的影书、薇洛的余烬）| **弹出**法术轮盘 UI，让玩家**选一个**法术，再走相应动作 | 法术 _菜单_ |

可以这么记忆：

```
打开一本 SpellBook → 选一个 spell → 进入 "book" 状态 → 执行该 spell 选项绑的 spellfn 或者 aoespell
                                                                                ↓ 有些 spell 用 spellcaster：触发 CASTSPELL 流水线（22.3）
                                                                                ↓ 有些 spell 自带"持续效果"：生成一个挂 spell 组件的隐形 entity
```

> 也就是说，"一根紫色法杖"只用 `spellcaster`；"一颗果冻豆产生的发光效果"只用 `spell`；"麦斯威尔的影书"用 `spellbook` 选择四种影子法术，并间接调用 `aoespell` 或 `spellcaster`。三者会经常组合，但**不存在依赖**——你可以单独用任何一个。

我们会先讲 `Spell`（小而清晰），再讲 `SpellBook`（接了一整套 UI / 动作链路）。

---

### 二、新手篇 · Spell 组件：5 分钟做一个"吃了发光 8 分钟"的效果

#### 2.1 Spell 组件的工作模型

`spell` 组件**只关心三件事**：

1. 选一个目标（`target`），把自己绑到它身上。
2. 跑一个时长（`duration`），每帧或每隔 `period` 秒执行一次 `fn`。
3. 时间到了自动 finish，可选择把自己 `Remove`。

它**不是**通过 debuffable 那种"挂账本"来追踪——而是依赖**一个独立的隐形 entity 抱着 spell 组件**，挂在目标身上做计时。等到时间到了，由它**自己**清理一切。

举个最直白的例子。看 `wormlight.lua:30-50`：

```lua
local function create_light(eater, lightprefab)
    if eater.wormlight ~= nil then
        -- 重复吃了：刷新计时
        if eater.wormlight.prefab == lightprefab then
            eater.wormlight.components.spell.lifetime = 0
            eater.wormlight.components.spell:ResumeSpell()
            return
        else
            -- 吃了不同等级的：先结束旧的
            eater.wormlight.components.spell:OnFinish()
        end
    end

    local light = SpawnPrefab(lightprefab)
    light.components.spell:SetTarget(eater)    -- 1) 绑目标
    if light:IsValid() then
        if light.components.spell.target == nil then
            light:Remove()
        else
            light.components.spell:StartSpell()  -- 2) 开始计时
        end
    end
end
```

注意流程：**吃到果子的不是 `light` 实体，而是 `eater`**——`light` 是新生成的、隐藏的、专门负责"持续发光"的隐形实体。它在 `light_ontarget`（`wormlight.lua:246-276`）里执行了几件事：

- 把自己 `Follower:FollowSymbol` 到目标，让发光特效跟着目标走；
- 在目标身上挂一个 `target.wormlight = inst`（这是个非组件的临时索引，方便后面查找）；
- 给目标绑死亡、变鬼、过载等几种事件，触发时直接 `inst.components.spell:OnFinish()`。

#### 2.2 最小四步走

```lua
local function fxprefab()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddFollower()
    inst:Hide()                    -- 不需要被看到
    inst:AddTag("FX")
    inst:AddTag("NOCLICK")
    -- 1) 不要网络化（详见 2.3 第 3 条）

    inst:AddComponent("spell")
    inst.components.spell.spellname = "myglow"             -- 2) 唯一名字
    inst.components.spell.duration  = 480                  -- 3) 8 分钟
    inst.components.spell.onstartfn  = function(inst) print("glow on") end
    inst.components.spell.onfinishfn = function(inst) print("glow off") end
    inst.components.spell.removeonfinish = true            -- 4) 到期自动 Remove

    inst.persists = false  -- 在没目标前不持久化（有了目标后由 ontargetfn 改）

    return inst
end

return Prefab("myglow", fxprefab)
```

然后在你的食物的 `oneatenfn` 里：

```lua
oneatenfn = function(inst, eater)
    local spell = SpawnPrefab("myglow")
    spell.components.spell:SetTarget(eater)
    spell.components.spell:StartSpell()
end
```

吃下去就会立刻打印 `glow on`，8 分钟后自动 `glow off` 并被回收。

#### 2.3 新手最常踩的 4 个坑

1. **必须先 `SetTarget` 再 `StartSpell`**。源码 `spell.lua:114-120`：

```lua
function Spell:StartSpell()
    if self.target == nil then
        return    -- 没目标直接退出
    end
    self.inst:StartUpdatingComponent(self)
    self:OnStart()
end
```

调用顺序反了，spell 根本不会被加进 update 循环。

2. **多次给同一个目标贴同名 spell 不会"刷新"，会创建两个独立的效果实体**。`spell` 组件**不像 `debuffable` 有账本机制**。你需要像 `wormlight.lua:31-39` 那样**手动**在目标身上记一个临时索引（`eater.wormlight`），下次再贴时主动判断重置。

3. **不要 `inst.entity:AddNetwork()`**。绝大多数 spell 效果实体是**非网络化**的隐形服务端实体——客户端不需要看到它本身，看到的只是它产生的特效（`fx`）。这能极大减少网络压力。

4. **`spellname` 字段是个"自旋锁"，别和目标已有标签撞名**。`spell.lua:139-144`：

```lua
function Spell:SetTarget(target)
    if not target:HasTag(self.spellname) then
        self.target = target
        self:OnTarget()
    end
end
```

如果你设 `spellname = "wormlight"` 而目标身上已经有 `wormlight` 标签，`SetTarget` 会**直接跳过**，target 字段都不会被赋值——这是用来防止"两个同名 spell 互相覆盖"的兜底。

### 三、进阶篇 · Spell 组件全字段拆解

#### 3.1 字段表

`spell.lua:1-22` 的构造函数：

| 字段 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `inst` | Entity | —— | 持有 spell 的隐形实体（自己）|
| `active` | bool | false | 是否已 `StartSpell` |
| `spellname` | string | `"spell"` | 唯一标识，会被 `SetTarget` 用来检测重复 |
| `onstartfn` | fn(inst) | nil | `OnStart` 时调用 |
| `onfinishfn` | fn(inst) | nil | `OnFinish` 时调用（无论是自然到期还是被强制结束）|
| `ontargetfn` | fn(inst, target) | nil | `SetTarget` 成功后调用 |
| `fn` | fn(inst, target) | nil | 每帧或每 `period` 秒被调用 |
| `resumefn` | fn(inst, timeleft) | nil | 从存档加载或主动调用 `ResumeSpell` 时调用 |
| `target` | Entity | nil | 当前贴的目标，**没 SetTarget 之前是 nil** |
| `duration` | number | 3 | 总时长（秒）|
| `lifetime` | number | 0 | 已经存活了多少秒 |
| `period` | number | nil | 多久跑一次 `fn`，nil = 每帧 |
| `timer` | number | nil | `period` 倒计时（内部用）|
| `removeonfinish` | bool | false | finish 时是否 `inst:Remove()` |
| `variables` | table | `{}` | 给你存自定义参数用的容器（不参与存档）|

#### 3.2 核心方法的实现细节

`OnUpdate(dt)`（`spell.lua:43-67`）：

```lua
function Spell:OnUpdate(dt)
    self.lifetime = self.lifetime + dt

    if self.timer ~= nil then
        self.timer = self.timer - dt
        if self.timer <= 0 then
            self.timer = nil
        end
    end

    if self.timer == nil then
        if self.fn ~= nil then
            self.fn(self.inst, self.target)
        end
        if self.period ~= nil then
            self.timer = self.period
        end
    end

    if self.lifetime >= self.duration then
        self:OnFinish()
    end
end
```

读法：

- 每帧 `lifetime += dt`。
- 如果设置了 `period`，那么进入 `timer` 倒计时，到 0 才触发一次 `fn`，触发后重置 `timer = period`。`fn` 第一次会在**进入 update 的当帧**触发（因为 timer 初始是 nil）。
- 如果没设 `period`，每帧都跑 `fn`。
- `lifetime >= duration` 后 `OnFinish`。注意这里**不会清掉 `timer`**——意味着如果你刚好在 `fn` 触发那一帧到期，会先跑一次 `fn` 再 finish。

`OnFinish`（`spell.lua:31-41`）：

```lua
function Spell:OnFinish()
    if self.onfinishfn ~= nil then
        self.onfinishfn(self.inst)
    end
    self.inst:StopUpdatingComponent(self)
    if self.removeonfinish then
        self.inst:Remove()
    end
end
```

注意它**不重置 `active`、`lifetime`**——这是为了让 `OnFinish` 是个**幂等**且**显式**的操作；多次调用第二次以后实际啥也不做（因为已经 StopUpdating）。

#### 3.3 ResumeSpell：存档读盘后怎么继续？

`spell.lua:75-128` 处理了"存档读盘"。关键设计：

```lua
function Spell:OnSave()
    local data = {}
    data.lifetime = self.lifetime
    data.timer = self.timer
    data.active = self.active
    if self.target ~= nil and not self.target:HasTag("player") then
        data.target = self.target.GUID
        return data, { self.target.GUID }
    end
    return data
end
```

**关键发现 1**：玩家目标**不会被存**到 spell 的 OnSave 里。原因是玩家自己是另一个独立的存档单位，且断线/上线/复活会换 GUID——所以官方选择把"玩家关联"放在玩家自己身上处理（比如 `wormlight.lua:269` 是 `inst.persists = false`，玩家断开重连时该效果实体就被销毁了）。

**关键发现 2**：返回值的第二个参数 `{ self.target.GUID }` 是"延迟引用"。`LoadPostPass` 在 `newents` 里查找它再绑回去（`spell.lua:98-112`）。

`ResumeSpell` 本质上是"我已经在跑，存档读出来之后接着跑"：

```lua
function Spell:ResumeSpell()
    if self.resumefn ~= nil then
        local timeleft = self.duration - self.lifetime
        self.resumefn(self.inst, timeleft)
        self.inst:StartUpdatingComponent(self)
    end
end
```

`wormlight.lua:166-168` 的 `light_resume` 用了它：

```lua
local function light_resume(inst, time)
    inst.fx:setprogress(1 - time / inst.components.spell.duration)
end
```

读法：把剩余时间换算成"光亮还剩多少比例"，让特效从那一刻开始播。

> 一个老手才会注意的小坑：**如果你不设 `resumefn`，断线重连的 spell 永远不会被加进 update 循环**——这就是个"哑 spell"了，时间不再流逝。**任何你想让它跨档继续走的 spell，必须给 `resumefn` 至少一个空函数**。

#### 3.4 与 22.1 的 buff 系统对比

| 对比项 | `debuff` / `debuffable`（22.1）| `spell` |
| --- | --- | --- |
| 谁挂在被影响目标上？ | `debuffable` 组件 | **没有**——只有 spell 实体跟着 target 走 |
| 谁挂在效果实体上？ | `debuff` 组件 | `spell` 组件 |
| 是否有"账本"（同名查找）？ | 有，`debuffable.debuffs[name]` | **无**，需要自己在 target 上挂临时索引 |
| 同名重复 Add 时？ | 触发 `OnExtended`（刷新 / 延长）| **无内置机制**，要自己判断 |
| 是否支持网络化（fx 可见）？ | debuff 实体常常网络化（如玩家的可见 buff icon）| 通常**不网络化**，靠 fx 子实体可见 |
| 用例 | 食物 buff、毒雾 debuff | 萤光蠕虫、临时跟踪型法术效果 |

**何时选 `spell` 而不是 `debuff`**：

- 效果的"主体"是一个有自己状态的实体（如萤光附身、跟随式光球），而不是一段附加属性。
- 你需要一个独立的实体来挂 `Follower`、`SoundEmitter`、`Light` 等组件。
- 你不需要"在目标的 buff 栏里看到这条"。

**何时选 `debuff`**：

- 效果只是一系列属性变化（速度、伤害、回血、消耗等）。
- 你希望它在断线重连后**自动**被 `debuffable:RestoreDebuff` 恢复。
- 你需要"同名再贴自动刷新计时"这种官方机制。

### 四、新手篇 · SpellBook 组件：3 分钟做一本"两个法术"的小书

#### 4.1 SpellBook 是什么

`SpellBook` 是**玩家在 HUD 上看到的法术轮盘 UI 的服务端 / 客户端控制器**。它本身**不释放任何法术**——它只负责：

1. 弹出 / 关闭轮盘 UI（通过 `OpenSpellWheel` / `CloseSpellWheel`）；
2. 记住玩家选中的是哪一个 spell（`SelectSpell` / `GetSelectedSpell`）；
3. 把"选 spell"这一步与"实际释放"解耦——选完之后，**根据 spell 配置**走不同动作（直接 `aoespell`、`spellcaster`，或者 spellbook 自己的 `spellfn`）。

#### 4.2 三步法术书

最简洁的"两个法术的书"骨架：

```lua
local SPELLS = {
    {
        label = "回血", atlas = "images/spell_icons.xml", normal = "heal.tex",
        onselect = function(inst)
            inst.components.spellbook:SetSpellName("回血")
            if TheWorld.ismastersim then
                inst.components.spellbook:SetSpellFn(function(book, user)
                    user.components.health:DoDelta(20)
                    return true
                end)
            end
        end,
        execute = function(inst)
            -- 让 SGwilson 进入 "book" 动画，状态机自动 PerformBufferedAction
            BufferedAction(ThePlayer, nil, ACTIONS.CAST_SPELLBOOK, inst):Do()
        end,
    },
    {
        label = "回理智", atlas = "images/spell_icons.xml", normal = "sanity.tex",
        onselect = function(inst)
            inst.components.spellbook:SetSpellName("回理智")
            if TheWorld.ismastersim then
                inst.components.spellbook:SetSpellFn(function(book, user)
                    user.components.sanity:DoDelta(20)
                    return true
                end)
            end
        end,
        execute = function(inst)
            BufferedAction(ThePlayer, nil, ACTIONS.CAST_SPELLBOOK, inst):Do()
        end,
    },
}

local function fn()
    -- ...省略基础设置...
    inst:AddTag("book")
    inst.entity:SetPristine()

    inst:AddComponent("spellbook")
    inst.components.spellbook:SetItems(SPELLS)
    inst.components.spellbook:SetRequiredTag("magician")  -- 持有者必须有该标签
    return inst
end
```

四个最常用 Setter：

| 字段 / Setter | 作用 |
| --- | --- |
| `SetItems(items)` | 必填，给一张 spell 选项表 |
| `SetRequiredTag(tag)` | 用户必须有这个标签才能用（如 `"shadowmagic"`、`"ember_master"`）|
| `SetRadius(r)` / `SetFocusRadius(r)` | 轮盘视觉半径 |
| `SetSpellFn(fn)` | 在 `CAST_SPELLBOOK.fn` 里被调用（无 aoespell / spellcaster 路径时）|

#### 4.3 spell 选项表（items）的字段

来自 `playerhud.lua:1240-1310` 与各家 spellbook 的实际配置，items 中每一项支持以下字段：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `label` | string | UI 上显示的法术名 |
| `atlas` / `normal` | string | 图标的 atlas 与 tex 名 |
| `widget_scale` / `hit_radius` | number | UI 尺寸微调 |
| `onselect(inst)` | fn | **客户端 + 服务端都跑**——切换到这个 spell 时被调用 |
| `execute(inst)` | fn | **只在客户端**——玩家"确认"该项时跑，比如发动作 |
| `ondown` | fn | （可选）按下时跑，控制器模式下用 |
| `onfocus` | fn | （可选）焦点切到该项时跑（高亮音效等）|
| `selected` | bool | （内部用）当前选中标记 |
| `default_focus` | bool | （可选）初始焦点 |

#### 4.4 注意"客户端 + 服务端"二元性

`onselect` 同时跑在**客户端和服务端**——因为玩家**任意一台机器**上选的法术，状态必须在两侧都同步：客户端要切 `aoetargeting` 准星、服务端要切 `aoespell` 的 spellFn。源码 `waxwelljournal.lua:255-258`：

```lua
if TheWorld.ismastersim then
    inst.components.aoetargeting:SetTargetFX("reticuleaoesummontarget_1d2")
    inst.components.aoespell:SetSpellFn(WorkerSpellFn)
    inst.components.spellbook:SetSpellFn(nil)
end
```

`if TheWorld.ismastersim then` 之外的代码是客户端 + 服务端都要跑的；之内的只在服务端跑。

#### 4.5 让玩家触发的三种"路径"

`waxwelljournal.lua` 一本影书里同时演示了**两种**释放路径：

**路径 A：通过 aoespell（范围法术）**

适用于"召唤影子工人""布下影子陷阱"这种需要在地面选点的法术。`onselect` 里设置 `aoespell:SetSpellFn(WorkerSpellFn)`，并把 `spellbook:SetSpellFn(nil)` 清空。玩家在选完法术后，`execute = StartAOETargeting` 把控制权交给 `playercontroller:StartAOETargetingUsing(inst)` 进入瞄准模式；接下来的"按确认键释放"会走 `CASTAOE` 动作流水线（详见 22.5）。

**路径 B：通过 spellbook:SpellFn（即时法术）**

适用于"召唤一顶大礼帽""读书提升精神"这种不需要选点的法术。`onselect` 里 `aoespell:SetSpellFn(nil)`、`spellbook:SetSpellFn(TopHatSpellFn)`，`execute` 直接发 `BufferedAction(ACTIONS.CAST_SPELLBOOK)`。源码 `waxwelljournal.lua:332-340`：

```lua
{
    label = STRINGS.SPELLS.SHADOW_TOPHAT,
    onselect = function(inst)
        inst.components.spellbook:SetSpellName(STRINGS.SPELLS.SHADOW_TOPHAT)
        inst.components.spellbook:SetSpellAction(nil)
        if TheWorld.ismastersim then
            inst.components.aoespell:SetSpellFn(nil)
            inst.components.spellbook:SetSpellFn(TopHatSpellFn)
        end
    end,
    ...
},
```

最后会走到 `actions.lua:5928-5942` 的 `CAST_SPELLBOOK.fn`：

```lua
ACTIONS.CAST_SPELLBOOK.fn = function(act)
    if act.invobject then
        if act.doer.components.inventory then
            act.doer.components.inventory:ReturnActiveActionItem(act.invobject)
        end
        if act.invobject.components.inventoryitem and
            act.invobject.components.inventoryitem:GetGrandOwner() == act.doer and
            act.invobject.components.spellbook and
            act.invobject.components.spellbook:CanBeUsedBy(act.doer) then
            return act.invobject.components.spellbook:CastSpell(act.doer)
        end
    elseif act.target == act.doer and act.target.components.spellbook and
        act.target.components.spellbook:CanBeUsedBy(act.doer) then
        return act.target.components.spellbook:CastSpell(act.doer)
    end
end
```

**路径 C：通过 `SetSpellAction(action)`**

适用于"我要走某个特定动作（比如 `COMMUNE_WITH_ABIGAIL`）而不是默认 `CAST_SPELLBOOK`"。abigail_flower 在每个具体的命令选项里调 `spellbook:SetSpellAction(SOME_ACTION)`，让玩家执行时不走 `CAST_SPELLBOOK` 而是走该自定义动作。状态机入口在 `SGwilson.lua:1624-1629`，可以看到针对 `abigail_flower` 还有特殊分支：

```lua
ActionHandler(ACTIONS.CAST_SPELLBOOK, function(inst, action)
    return action.invobject ~= nil
        and ((action.invobject:HasTag("abigail_flower")
            and ((action.invobject:HasTag("unsummoning_spell") and "unsummon_abigail")
                or "commune_with_abigail")))
        or "book"
end),
```

### 五、进阶篇 · SpellBook 全流水线

#### 5.1 SpellBook 组件字段表

`spellbook.lua:15-36` 的构造函数：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `tag` | string | `SetRequiredTag` 设置，用户必须 `HasTag(tag)` 才能用 |
| `items` | table | spell 选项数组 |
| `bgdata` | table | 轮盘背景动画数据（含 `build` / `bank` / `anim` / `widget_scale` / `loop`）|
| `radius` / `focus_radius` | number | 轮盘尺寸 |
| `spell_id` | number | 当前选中的 spell 在 items 中的索引 |
| `spellname` | string | 当前选中的 spell 名（用于显示）|
| `spellaction` | Action | （可选）用 `SetSpellAction` 替换默认 `CAST_SPELLBOOK` 动作 |
| `spellfn` | fn | 当走默认 `CAST_SPELLBOOK` 路径时调用 |
| `onopenfn` / `onclosefn` | fn | 轮盘开 / 关时的回调 |
| `canusefn(inst, user)` | fn | 自定义"能不能用"判定，与 `tag` AND 关系 |
| `shouldopenfn(inst, user)` | fn | "通过了 CanBeUsedBy 但要不要真打开" |
| `opensound` / `closesound` / `executesound` / `focussound` | string | 各时机播放的音效 |
| `closeonexecute` | bool | 默认 true，控制器模式按下后是否关掉 |

#### 5.2 完整流水线

```
玩家右键背包里的法术书
    ↓
componentactions.lua: spellbook(inst, doer, actions) 
    判断 doer.HUD:GetCurrentOpenSpellBook() 是否已是它
        是 → 弹 CLOSESPELLBOOK
        否 → CanBeUsedBy 通过 → 弹 USESPELLBOOK
    ↓
actions.lua: USESPELLBOOK.pre_action_cb
    spellbook:ShouldOpen(doer) → spellbook:OpenSpellBook(doer)
        → user.HUD:OpenSpellWheel(items, radius, focus_radius, bgdata)
        → inst:PushEvent("openspellwheel")     ← spellbook 监听到，调 onopenfn
    ↓
玩家在轮盘上选了一项
    HUD 转发 → invobject.components.spellbook:SelectSpell(id)
        → item.onselect(inst) [客户端 + 服务端]
    ↓
玩家"确认"
    item.execute(inst) [客户端]
        → 三种路径之一：
            A) BufferedAction(ACTIONS.CAST_SPELLBOOK)
            B) playercontroller:StartAOETargetingUsing(inst) → CASTAOE
            C) BufferedAction(SetSpellAction 指定的动作)
    ↓
SGwilson 进入对应状态（book / castspellmind / 自定义状态）
    PerformBufferedAction → 对应 fn → spellbook:CastSpell(user) 或 aoespell:CastSpell(...)
```

#### 5.3 `CanBeUsedBy` 的细节

`spellbook.lua:80-86`：

```lua
function SpellBook:CanBeUsedBy(user)
    return (self.tag == nil or user:HasTag(self.tag))
        and (self.canusefn == nil or self.canusefn(self.inst, user))
        and self.items ~= nil
        and #self.items > 0
        and (not self.inst.isplayer or self.inst == user)
end
```

特别注意最后一条 `(not self.inst.isplayer or self.inst == user)`：

- 如果 spellbook 挂在某物品上（`inst.isplayer == nil`），那任何拿着这物品的玩家都能用；
- **如果 spellbook 挂在玩家自己身上**（`walter.lua:274` 那种），那只有这个玩家自己能用——别的玩家点击不会弹轮盘。

这就是 `walter` 和 `woby` 把 spellbook 挂在 player 实体上的特殊场景：**口令本不是一个可拿可掉落的物品，而是角色自带的菜单**。

#### 5.4 三个回调钩子的触发时机

`SetOnOpenFn` / `SetOnCloseFn` 是通过事件挂上的，源码 `spellbook.lua:1-13` + `34-35`：

```lua
local function OnOpenSpellWheel(inst)
    local self = inst.components.spellbook
    if self.onopenfn ~= nil then
        self.onopenfn(inst)
    end
end
... 
inst:ListenForEvent("openspellwheel", OnOpenSpellWheel)
inst:ListenForEvent("closespellwheel", OnCloseSpellWheel)
```

事件 `openspellwheel` / `closespellwheel` 是从 HUD 推送的（`playerhud.lua:1305` / `1320` 处）。这意味着：**只要有任何代码 `inst:PushEvent("openspellwheel")`，spellbook 就以为轮盘打开了**——比如服务端 hook 收到 RPC 后也能模拟一次推送。

`SetCanUseFn` 与 `SetShouldOpenFn` 的区别值得多说一句：

- `canusefn`：决定"右键时要不要列出 USESPELLBOOK 动作"。它**直接**影响动作菜单。
- `shouldopenfn`：在 `pre_action_cb` 里，**已经确认要发动作了**，但在真打开 HUD 之前**再问一次**——典型用法是"这一帧正在播预览动画，不要打开打断"。`wobycommon.lua:262-264` 就用它检查 `woby_commands_classified:IsBusy()`。

#### 5.5 三个动作的分工

| 动作 | 触发场景 | 主要工作 |
| --- | --- | --- |
| `USESPELLBOOK` | 右键且当前没打开 | `pre_action_cb` 里 `spellbook:OpenSpellBook(doer)` |
| `CLOSESPELLBOOK` | 右键且当前正打开 | `pre_action_cb` 里 `HUD:CloseSpellWheel()` |
| `CAST_SPELLBOOK` | 玩家在轮盘选完 + 确认 | `fn` 里调 `spellbook:CastSpell(doer)` |

三者都标了 `priority = 2`（前两者）或默认（最后一个），而且 `mount_valid = true`——意味着骑乘状态下也能用书。

### 六、老手篇 · 三种"法术书"的实战对比

下面以四个官方实例对比，看 SpellBook 的灵活性：

| 法术书 prefab | `RequiredTag` | items 数量 | 释放路径 | 备注 |
| --- | --- | --- | --- | --- |
| `waxwelljournal`（影书）| `"shadowmagic"` | 5（影工/影守/影陷阱/影柱/影帽）| A + B 混用 | 经典"多路径混合"教学样本 |
| `willow_ember`（薇洛余烬）| `"ember_master"` | 多个（焚烧、闪现）| 主走 B（aoespell） | 余烬作为燃料，每法术消耗不同（看 `executesound`）|
| `abigail_flower`（艾比盖尔之花）| `"ghostlyfriend"` | 由技能树动态决定（`SetItems` 多次切换）| 主走 C（`SetSpellAction(ACTIONS.COMMUNE_WITH_ABIGAIL)`）| 唯一一个用 `SetSpellAction` 显式接管的 |
| `winona_remote`（薇诺娜遥控器）| `"portableengineer"` | 多个工程器 | A（`USEITEM` / 直接召唤）| `SetItems` 在动态刷新工程器时反复重置 |
| `walter` 的 `spellbook`（口令本）| `nil`（用 `SetCanUseFn`）| 动态 | B / C 混合 | 挂在玩家自己身上，借 `isplayer` 检查避免别人调用 |
| `woby` 的 `spellbook`（口令）| `nil`（用 `SetCanUseFn`）| 动态 | C | 挂在宠物自己身上，需要 `SetShouldOpenFn` 避免预览状态被打开 |

#### 6.1 一个"动态 items"的模式

不少官方 spellbook 会**不停地 SetItems**——比如沃尔特习得新口令时往 spellbook 加一项，技能解锁/卸下时移除一项。范式：

```lua
local function RefreshSpells(inst)
    local commands = {}
    if HasSkill(inst, "skill_a") then table.insert(commands, SPELL_A) end
    if HasSkill(inst, "skill_b") then table.insert(commands, SPELL_B) end
    inst.components.spellbook:SetItems(commands)
end

inst:ListenForEvent(TheWorld.ismastersim and "ms_skilltreeinitialized" or "skilltreeinitialized_client", RefreshSpells)
inst:ListenForEvent("onactivateskill_server", RefreshSpells)
inst:ListenForEvent("ondeactivateskill_server", RefreshSpells)
```

注意几个细节：

- 服务端事件名是 `ms_skilltreeinitialized` / `onactivateskill_server`，客户端是 `skilltreeinitialized_client` / `onactivateskill_client`。
- 客户端那一份也要刷，因为 items 列表参与了客户端 HUD 的 spell wheel 显示。
- `SetItems` 后**不会**自动重开轮盘——下次玩家自己再开就生效。

#### 6.2 与 22.6 SpellBookCooldowns 的关系

`SpellBook` 组件**不带冷却机制**——它只决定"能不能用"和"选了哪个"。冷却由独立组件 `SpellBookCooldowns` 维护。这就是为什么 22.6 单独成节：它本身是个**装饰器**，不修改 spellbook 内部，而是通过外部对 spell 选项的 `selectable` 状态做约束。详见 22.6。

#### 6.3 边界场景速查

| 场景 | 处理 |
| --- | --- |
| 切换法术后玩家立即被打断（如跳船、受伤） | 没问题——`onselect` 只改组件配置，没有锁玩家。下次玩家选时再触发 execute |
| spellbook 与 inventoryitem 同存 | 自动支持。`USESPELLBOOK.pre_action_cb` 会先 ReturnActiveItem |
| spellbook 挂在 player 上（如 walter）| `CanBeUsedBy` 中 `self.inst == user` 检查兜底，别的玩家点击无效 |
| 网络模式下断线重连 | SpellBook 自己不持久化，依赖 RequiredTag 和外部 items 的存档 |
| 法术书没燃料 (fueldepleted) | `componentactions.lua:2871` 显式排除 `fueldepleted` 标签，菜单不弹 |
| 切换 spell 时正在 AOE 瞄准 | `OpenSpellBook` 里会 `playercontroller:CancelAOETargeting()` 兜底 |

### 七、综合实战：用 Spell + SpellBook 做一个"双效魔法书"

**目标**：一本能让玩家二选一释放"持续 30 秒回血" 或 "瞬间回理智 50"的小法术书，挂在背包物品上。

```lua
-- 1) 持续效果实体（spell 组件）
local function RegenSpellFx()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddFollower()
    inst:Hide()
    inst:AddTag("FX")
    inst:AddTag("NOCLICK")

    inst:AddComponent("spell")
    inst.components.spell.spellname = "myregenspell"
    inst.components.spell.duration = 30
    inst.components.spell.period = 1  -- 每秒触发一次
    inst.components.spell.fn = function(inst, target)
        if target.components.health then
            target.components.health:DoDelta(2)
        end
    end
    inst.components.spell.removeonfinish = true
    inst.persists = false

    return inst
end

-- 1.5) 通用 execute：把"由背包发起施法"这一步交给 inventory_replica
-- 它会按当前 spellbook 已 SelectSpell 的索引 + spellaction，转 RPC 到服务端
local function DoSpellAction(inst)
    local inventory = ThePlayer.replica.inventory
    if inventory then
        inventory:CastSpellBookFromInv(inst)
    end
end

-- 2) 两个 spell 选项
local SPELLS = {
    {
        label = "回血", atlas = "images/spell_icons.xml", normal = "heal.tex",
        onselect = function(inst)
            inst.components.spellbook:SetSpellName("回血")
            inst.components.spellbook:SetSpellAction(nil)
            if TheWorld.ismastersim then
                inst.components.spellbook:SetSpellFn(function(book, user)
                    local fx = SpawnPrefab("myregenspell_fx")
                    fx.components.spell:SetTarget(user)
                    if fx.components.spell.target then
                        fx.components.spell:StartSpell()
                    else
                        fx:Remove()
                    end
                    return true
                end)
            end
        end,
        execute = DoSpellAction,
    },
    {
        label = "回理智", atlas = "images/spell_icons.xml", normal = "sanity.tex",
        onselect = function(inst)
            inst.components.spellbook:SetSpellName("回理智")
            inst.components.spellbook:SetSpellAction(nil)
            if TheWorld.ismastersim then
                inst.components.spellbook:SetSpellFn(function(book, user)
                    if user.components.sanity then
                        user.components.sanity:DoDelta(50)
                    end
                    return true
                end)
            end
        end,
        execute = DoSpellAction,
    },
}

-- 3) 主物品 prefab
local function bookfn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()
    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("book_maxwell")
    inst.AnimState:SetBuild("book_maxwell")
    inst.AnimState:PlayAnimation("idle")

    inst:AddTag("book")
    inst:AddTag("nopunch")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")

    inst:AddComponent("spellbook")
    inst.components.spellbook:SetItems(SPELLS)
    inst.components.spellbook.opensound  = "dontstarve/common/together/book_maxwell/use"
    inst.components.spellbook.closesound = "dontstarve/common/together/book_maxwell/close"

    return inst
end

return Prefab("mymagicbook", bookfn),
       Prefab("myregenspell_fx", RegenSpellFx)
```

照下表逐条对照源码自检：

| 检查项 | 源码位置 | 我的法术书 |
| --- | --- | --- |
| spellbook items 非空 | `spellbook.lua:84` | ✅ 2 项 |
| 没设 `tag` 时不限制持有者 | `spellbook.lua:81` | ✅ 所有人可用 |
| 不挂在玩家身上时 `isplayer == nil` | `spellbook.lua:85` | ✅ inst 不是 player |
| `onselect` 在两侧都跑、`SetSpellFn` 只在 ismastersim | `waxwelljournal.lua:255-258` | ✅ 一致 |
| 默认走 `CAST_SPELLBOOK` 而非自定义动作 | `actions.lua:5928` | ✅ `SetSpellAction(nil)` |
| 玩家用 `inventory:CastSpellBookFromInv(item)` 转 RPC | `walter.lua:50-55`、`waxwelljournal.lua:340-345` | ✅ `DoSpellAction` 复用 |
| 客户端 RPC 触达服务端 `playercontroller:RemoteCastSpellBookFromInv` | `playercontroller.lua:5625-5644` | ✅ 自动 |
| spell 组件需先 SetTarget 再 StartSpell | `spell.lua:114-120` | ✅ 顺序正确 |
| spell 实体不要 AddNetwork | `wormlight.lua:298-326` | ✅ 隐形非网络化 |

### 八、本节回顾与学习路径

把这一节的所有要点压缩成一张图：

```
[玩家] ─右键背包书→ componentactions.spellbook → USESPELLBOOK
            ↓                                       ↓
       CLOSESPELLBOOK ←─ 已开                  pre_action_cb
            ↓                                       ↓
   HUD:CloseSpellWheel              spellbook:OpenSpellBook(user)
                                                ↓
                                  HUD:OpenSpellWheel + PushEvent("openspellwheel")
                                                ↓
                       [玩家选项] item.onselect(inst) [两端运行]
                                                ↓
                       [玩家确认] item.execute(inst) [客户端]
                                                ↓
                              三条路径分流：CAST_SPELLBOOK / CASTAOE / 自定义动作
                                                ↓
                                            SGwilson 状态机
                                                ↓
                              spellbook:CastSpell(user) / aoespell.spell(...) 
                                                ↓
                          可选：SpawnPrefab + spell:SetTarget + spell:StartSpell
                                                ↓
                          每帧 / 每 period：spell.fn → 到期 OnFinish
```

学习侧重：

- **新手**：先把 `Spell` 用熟（它的字段最少、行为最直白），然后做出"自带轮盘的两选一小书"。
- **进阶**：理解"客户端 + 服务端二元性"，知道 `onselect` 跑两边、`SetSpellFn` 只跑服务端这种 split execution 模式；能完整说出"右键 → USESPELLBOOK → OpenSpellWheel → SelectSpell → execute → CAST_SPELLBOOK → CastSpell"链路。
- **老手**：能模仿沃尔特/沃比那种"挂在 player 上、用 `SetShouldOpenFn` 防止 UI 抖动"的模式；能在 `SetItems` 动态变化（技能树）时同步两端；能搭配 `aoespell` / `spellcaster` 形成混合释放路径；并清楚 `SpellBookCooldowns` 是独立的（22.6 详解）。

> **学习路径建议**：先按 2.2 写一个最小的 `spell` 效果做"吃完发光 8 分钟"；再按 4.2 写一本"两选一"的法术书，确认能打开轮盘 + 选中 + 执行；最后参考 `waxwelljournal.lua` 把 `aoespell` 加进来变成"路径 A + 路径 B 混合"的影书结构。下一节会带你深入 `AOESpell`——范围法术怎么定准星、怎么消耗子弹、怎么 `canrepeatcast`。


## 22.5 AOESpell——范围法术的实现

（待编写）

## 22.6 SpellBookCooldowns——法术冷却系统

（待编写）

## 22.7 实战：创建带有 Buff 效果的法杖 Mod

（待编写）
