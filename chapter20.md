# 第20章 食物系统

## 20.1 Edible 组件——食物的属性定义（FOODTYPE / FOODGROUP）

> 本节面向三类读者编写，章节中会以「【新手】」「【进阶】」「【老手】」分别标注关键信息，方便不同层次的读者各取所需。
>
> - **【新手】**：第一次接触食物开发，目标是能照着模板做出一个"能吃、能回血回饱食"的物品。
> - **【进阶】**：已经做过几个食物，希望理解 `FOODTYPE` / `FOODGROUP` 与标签系统、腐烂系统、香料、温度的协作。
> - **【老手】**：希望读懂 `edible.lua` 每一行细节，写自定义吃法、自定义伤害模型、或为 NPC / 怪物配置专门 Diet。
>
> 本节涉及的源码：
> - `scripts/components/edible.lua`（核心，本节主角）
> - `scripts/components/eater.lua`（搭档，但下一节才系统讲，本节只摘几段说明协作）
> - `scripts/constants.lua` 第 2010 行起：`FOODTYPE` 表
> - `scripts/constants.lua` 第 2033 行起：`FOODGROUP` 表
> - `scripts/tuning.lua` 第 2073-2077 行：`STALE_FOOD_*` / `SPOILED_FOOD_*` 默认值
> - `scripts/tuning.lua` 第 4593 行起：`SPICE_MULTIPLIERS`
>
> 教学示例 prefab：
> - `scripts/prefabs/honey.lua`：最简单的「能吃、会坏、会自动叠堆」食物
> - `scripts/prefabs/meats.lua`：主食物类型 + 次食物类型（怪物肉的 `MONSTER`）
> - `scripts/prefabs/inv_rocks_ice.lua`：温度类食物（冰）
> - `scripts/prefabs/veggies.lua`：`VEGGIE` + 次类型 `BERRY` 的配置写法
> - `scripts/prefabs/tallbirdegg.lua`：`SetOnEatenFn` 的用法
> - `mods/联机版mod/万物书/scripts/prefabs/04_tbat_foods/01_hedgehog_cactus_meat.lua`：mod 中的入门级食物范例

---

### 一、为什么需要 Edible 组件？（新手必读）

【新手】饥荒里"能不能吃"这个概念，看似简单，背后其实牵涉到 5 个独立问题：

1. **它能不能进玩家的嘴？**（即"这是食物"——这就是 `Edible` 组件回答的）
2. **谁能吃它？**（皮蛋白人 Wickerbottom、皮蛋 Webber、皮蛋 Wigfrid 各有忌口——这是 `Eater` 组件回答的，靠 `FOODTYPE` / `FOODGROUP` 沟通）
3. **吃下去给多少血、饱食、精神？**（`Edible` 组件回答）
4. **它现在新鲜不新鲜？**（`Perishable` 组件回答，但 `Edible` 会去读取它的状态）
5. **吃下去之后还要发生什么特殊效果？**（自定义回调，例如冰激凌降体温、毒蘑菇 daze、神秘果回 sanity）

所以一句话：**`Edible` 是「食物身份证 + 营养标签」，它把"这是食物，营养是 ABC，类型是 XYZ"挂到 prefab 身上**。剩下的腐烂、吃下去的状态变化由 `Perishable` 和 `Eater` 协作完成。

【进阶】更精确地说：`Edible` 是一个**服务端组件**（只在 `TheWorld.ismastersim` 中 `AddComponent("edible")`），它不做任何网络同步，所有计算都发生在主机。它只对外暴露两个关键面：

- 静态面：foodtype / secondaryfoodtype 通过给 entity `AddTag("edible_<type>")` 暴露给客户端，使得 `Eater:CanEat` 这种判断在客户端也能预判 UI（例如能不能拖到嘴上）。
- 动态面：`GetHealth(eater)` / `GetHunger(eater)` / `GetSanity(eater)` 在每次进食时被 `Eater:Eat` 调用，根据 eater 本身、香料、腐烂程度、亲和度共同算出最终增益。

【老手】Edible 不直接修改任何统计值（健康/饱食/精神），它只**计算**值并返回。真正调用 `health:DoDelta` / `hunger:DoDelta` / `sanity:DoDelta` 的是 `Eater:Eat`。这是一个清晰的"数据-动作"分离，方便扩展香料、亲和、`custom_stats_mod_fn` 等。

---

### 二、源码总览：`edible.lua` 的骨架

```1:54:scripts/components/edible.lua
local function oncheckbadfood(self)
    if self.healthvalue < 0 or (self.sanityvalue ~= nil and self.sanityvalue < 0) then
        self.inst:AddTag("badfood")
    else
        self.inst:RemoveTag("badfood")
    end
end

local function onfoodtype(self, new_foodtype, old_foodtype)
    if old_foodtype ~= nil then
        self.inst:RemoveTag("edible_"..old_foodtype)
    end
    if new_foodtype ~= nil then
		assert(self.foodtype ~= self.secondaryfoodtype, "Edible component: The main and secondary food types cannot be set to the same value.")
        self.inst:AddTag("edible_"..new_foodtype)
    end
end

local Edible = Class(function(self, inst)
    self.inst = inst
    self.healthvalue = 10
    self.hungervalue = 10
    self.sanityvalue = 0
    self.foodtype = FOODTYPE.GENERIC
    self.secondaryfoodtype = nil
    self.oneaten = nil
    self.degrades_with_spoilage = true
    self.gethealthfn = nil
    self.getsanityfn = nil
    --self.handleremovefn = nil

    self.temperaturedelta = 0
    self.temperatureduration = 0

    --chill is a percentage [0, 1] of .temperatureduration
    --don't change this from 0 unless .temperaturedelta > 0
    self.chill = 0
    --self.nochill = false

    self.stale_hunger = TUNING.STALE_FOOD_HUNGER
    self.stale_health = TUNING.STALE_FOOD_HEALTH

    self.spoiled_hunger = TUNING.SPOILED_FOOD_HUNGER
    self.spoiled_health = TUNING.SPOILED_FOOD_HEALTH

    self.spice = nil
end,
nil,
{
    healthvalue = oncheckbadfood,
    sanityvalue = oncheckbadfood,
    foodtype = onfoodtype,
    secondaryfoodtype = onfoodtype,
})
```

【新手】只看构造函数：默认值就是 health=10、hunger=10、sanity=0、foodtype=GENERIC。所以你只 `AddComponent("edible")` 不改任何字段，物品就已经能给玩家加 10 血和 10 饱食了。

【进阶】注意 `Class` 的第 4 个参数是 **setter 表**（`onfoodtype`、`oncheckbadfood`）。在 Klei 自家的 `class.lua` 里，当你给一个被注册过 setter 的字段赋值时（例如 `inst.components.edible.foodtype = FOODTYPE.MEAT`），会触发对应函数。也就是说：**每次改 foodtype/secondaryfoodtype，旧标签会被移除、新标签会被加上**；**每次改 health/sanity，会重新判断要不要打 badfood 标签**。

【老手】这种"setter 钩子"机制让 `Edible` 的所有标签维护都是自动的。你**绝不需要**手动 `inst:AddTag("edible_meat")` —— 这是 mod 里常见的坏味道。

#### setter 触发的两个动作

- `oncheckbadfood`：判定 `healthvalue < 0` 或 `sanityvalue < 0`，是则 `AddTag("badfood")`，否则移除。`badfood` 标签被很多 AI 用来判断"我不要捡这个"。
- `onfoodtype`：先去除旧的 `edible_<old>`，再加上 `edible_<new>`，同时断言主类型与次类型不能相同。

【进阶】源码里有个细节：`onfoodtype` 同时被 `foodtype` 和 `secondaryfoodtype` 复用，但 assert 只在 new_foodtype 非空时才执行。这意味着 **赋值时序很重要** —— 如果你想从一个主肉、次怪物的物品改成主怪物、次肉，正确做法是先把 `secondaryfoodtype` 置 nil，再改 `foodtype`，最后再设 `secondaryfoodtype`。否则中间会出现"主=次"瞬时状态，触发 assert 崩服。

---

### 三、字段逐项解析

下面把构造函数里和 `OnEaten` 里出现的字段全部讲清楚。

| 字段名 | 类型 | 默认值 | 含义 |
| --- | --- | --- | --- |
| `inst` | Entity | 自动赋值 | 宿主实体 |
| `healthvalue` | number | 10 | 基础健康增量（负数代表掉血） |
| `hungervalue` | number | 10 | 基础饱食增量 |
| `sanityvalue` | number | 0 | 基础精神增量 |
| `foodtype` | string | `FOODTYPE.GENERIC` | 主食物类型 |
| `secondaryfoodtype` | string\|nil | nil | 次食物类型，用来让一份食物挂上两个 `edible_*` 标签 |
| `oneaten` | function\|nil | nil | 自定义吃后回调（fn(inst, eater)） |
| `degrades_with_spoilage` | bool | true | 该食物是否因腐烂衰减营养 |
| `gethealthfn` | function\|nil | nil | 健康值动态计算函数（fn(inst, eater)），返回值会替代 `healthvalue` |
| `getsanityfn` | function\|nil | nil | 精神值动态计算函数（同上） |
| `handleremovefn` | function\|nil | nil | 自定义"吃完后怎么处理这个 prefab" |
| `overridestackmultiplierfn` | function\|nil | nil | 自定义吃整堆的倍率 |
| `temperaturedelta` | number | 0 | 进食腹中温度变化幅度（正=加热，负=制冷） |
| `temperatureduration` | number | 0 | 进食腹中温度变化持续时间 |
| `chill` | number\[0,1\] | 0 | 温度衰减百分比（被冰之类的逻辑使用） |
| `nochill` | bool\|nil | nil | true 时禁用 chill 衰减 |
| `stale_hunger` | number | `TUNING.STALE_FOOD_HUNGER` (0.667) | 食物变陈旧后饱食倍率 |
| `stale_health` | number | `TUNING.STALE_FOOD_HEALTH` (0.333) | 食物变陈旧后健康倍率 |
| `spoiled_hunger` | number | `TUNING.SPOILED_FOOD_HUNGER` (0.5) | 食物腐烂后饱食倍率 |
| `spoiled_health` | number | `TUNING.SPOILED_FOOD_HEALTH` (0) | 食物腐烂后健康倍率 |
| `spice` | string\|nil | nil | 香料标识，对应 `TUNING.SPICE_MULTIPLIERS` 的 key |
| `spoiledfood` | bool\|nil | nil | 内部标记：这就是一份"已腐烂食物"（如 `spoiled_food` prefab） |
| `ismeat` | bool\|nil | nil | 旧字段，仅用于某些遗留 AI 判断（如老式肉架），新代码请用 `foodtype == FOODTYPE.MEAT` |

【新手】最常用的 4 个字段是：`foodtype`、`healthvalue`、`hungervalue`、`sanityvalue`。其他默认即可。

【进阶】注意：

- `gethealthfn` 与 `healthvalue` **不冲突**，`gethealthfn` 优先；但 `healthvalue` 仍会被 `oncheckbadfood` 用来打 `badfood` 标签。所以如果你写的是负数动态健康但 `healthvalue` 设了正数，物品**不会**有 `badfood` 标签，AI 可能错误捡食。
- 同理 `getsanityfn` 也有同样的"动态/静态分离"问题。

【老手】`degrades_with_spoilage = false` 的食物（如冰、香料 prepared foods 等）**完全跳过腐烂衰减**，但它本身的 `perishable` 组件依然可以正常计时——只是吃下去时不打折。

---

### 四、`FOODTYPE` 常量表（出自 `scripts/constants.lua`）

```2010:2031:scripts/constants.lua
FOODTYPE =
{
    GENERIC = "GENERIC",
    MEAT = "MEAT",
    VEGGIE = "VEGGIE",
    ELEMENTAL = "ELEMENTAL",
    GEARS = "GEARS",
    HORRIBLE = "HORRIBLE",
    INSECT = "INSECT",
    SEEDS = "SEEDS",
    BERRY = "BERRY", --hack for smallbird; berries are actually part of veggie
    RAW = "RAW", -- things which some animals can eat off the ground, but players need to cook
    BURNT = "BURNT", --For lavae.
    NITRE = "NITRE", -- For acidbats; they are part of elemental.
    ROUGHAGE = "ROUGHAGE",
	WOOD = "WOOD",
    GOODIES = "GOODIES",
    MONSTER = "MONSTER", -- Added in for woby, uses the secondary foodype originally added for the berries
    LUNAR_SHARDS = "LUNAR_SHARDS", -- For rift birds, yummy glass
    CORPSE = "CORPSE", -- For rift buzzards potentially
    MIASMA = "MIASMA", -- For the centipede thrall
}
```

下面一一解释，并标注哪些"角色"会吃它们（来自 `eater.lua`/各角色 prefab）：

| FOODTYPE | 直译 | 适用对象 | 典型食物 |
| --- | --- | --- | --- |
| `GENERIC` | 通用 | 任何 OMNI、VEGETARIAN Diet 都能吃 | 蜂蜜、料理、果酱 |
| `MEAT` | 肉类 | OMNI、肉食者、Bearger、Moose | 大肉、小肉、烤肉 |
| `VEGGIE` | 蔬果 | OMNI、VEGETARIAN、Bearger、Moose | 胡萝卜、玉米、berries |
| `ELEMENTAL` | 元素 | 火龙、岩石龙等专属 Diet | 紫宝石、燧石（特殊配置） |
| `GEARS` | 齿轮 | WX-78 专属 Diet（`SetCanEatGears`） | gears、battery 等 |
| `HORRIBLE` | 怪味 | 强胃 Diet（`SetCanEatHorrible`） | 一般 mod 用来做剧毒食物 |
| `INSECT` | 昆虫 | OMNI 包含、蜘蛛、鸟类 | 蜜蜂、蝴蝶 |
| `SEEDS` | 种子 | 鸟、Moose、OMNI | 各种种子、培育种子 |
| `BERRY` | 浆果 | 注释明确：smallbird 专用 hack | berries（berries 主类型是 VEGGIE，次类型才是 BERRY） |
| `RAW` | 生 | 部分动物可吃地上的生食 | 生肉给狗 / 蜘蛛吃 |
| `BURNT` | 烧焦 | Lavae 专属 | 烧焦的物体 |
| `NITRE` | 硝石 | Acidbats，归 ELEMENTAL（`SetCanEatNitre`） | 硝石 |
| `ROUGHAGE` | 粗食 | 牛、河马 | 干草、植物纤维 |
| `WOOD` | 木材 | 海狸 Woodie | 树枝、原木 |
| `GOODIES` | 零食 | OMNI、VEGETARIAN 包含 | 糖果、玉米爆米花 |
| `MONSTER` | 怪物 | Woby（Walter 狗）次类型；玩家通常不吃 | 怪物肉、durian、moonshroom 等 |
| `LUNAR_SHARDS` | 月碎 | rift bird | 月亮玻璃 |
| `CORPSE` | 尸体 | rift buzzards（潜在） | 尸体 |
| `MIASMA` | 瘴气 | centipede thrall | rift 怪物 |

【新手】99% 的情况你只用：`GENERIC` / `MEAT` / `VEGGIE` / `SEEDS` / `GOODIES`，五个就够了。

【进阶】`MONSTER` 是 Klei 的一个机制：它**不是主食物类型**，而是和"肉"或者"蔬菜"叠加在次食物类型上。例如怪物肉的主类型是 MEAT、次类型是 MONSTER；榴莲（durian）的主类型是 VEGGIE、次类型是 MONSTER。这样设计的好处是：让大多数 Diet（OMNI/MEAT/VEGGIE）都能吃，但 Woby（专门吃 MONSTER 的）也单独识别。

【老手】`BERRY` 字段在注释里写着 `hack for smallbird`。具体原因是：smallbird 只爱吃浆果，但浆果在游戏里属于蔬菜（VEGGIE）。Klei 不想给 smallbird 一个 VEGGIE Diet（那它就会吃胡萝卜），于是给浆果加了一个次类型 BERRY，再给 smallbird 一个"只吃 BERRY"的 Diet。这就是 secondary foodtype 设计的源头。

---

### 五、`FOODGROUP` 常量表

```2033:2093:scripts/constants.lua
FOODGROUP =
{
    OMNI =
    {
        name = "OMNI",
        types =
        {
            FOODTYPE.MEAT,
            FOODTYPE.VEGGIE,
            FOODTYPE.INSECT,
            FOODTYPE.SEEDS,
            FOODTYPE.GENERIC,
            FOODTYPE.GOODIES,
        },
    },

    BERRIES_AND_SEEDS =
    {
        name = "BERRIES_AND_SEEDS",
        types =
        {
            FOODTYPE.SEEDS,
            FOODTYPE.BERRY,
        },
    },

    BEARGER =
    {
        name = "BEARGER",
        types =
        {
            FOODTYPE.MEAT,
            FOODTYPE.VEGGIE,
            FOODTYPE.BERRY,
            FOODTYPE.GENERIC,
        },
    },

    MOOSE =
    {
        name = "MOOSE",
        types =
        {
            FOODTYPE.MEAT,
            FOODTYPE.VEGGIE,
            FOODTYPE.SEEDS,
        },
    },

    VEGETARIAN =
    {
        name = "VEGETARIAN",
        types =
        {
            FOODTYPE.VEGGIE,
            FOODTYPE.SEEDS,
            FOODTYPE.GENERIC,
            FOODTYPE.GOODIES,
        },
    },
}
```

`FOODGROUP` 是**给 `Eater` 用的**，不是给 `Edible` 用的（注意区分！）。你**永远不会**给一个食物的 `edible.foodtype` 赋 `FOODGROUP.OMNI`，那是错误用法；你只会给 `eater.caneat` 赋 `FOODGROUP.OMNI`。

| 组名 | 包含类型 | 用途 |
| --- | --- | --- |
| `OMNI` | MEAT/VEGGIE/INSECT/SEEDS/GENERIC/GOODIES | **所有玩家**默认 Diet；大多数生物 |
| `BERRIES_AND_SEEDS` | SEEDS/BERRY | smallbird、鸟类 |
| `BEARGER` | MEAT/VEGGIE/BERRY/GENERIC | 大熊獾 |
| `MOOSE` | MEAT/VEGGIE/SEEDS | 麋鹿驼 |
| `VEGETARIAN` | VEGGIE/SEEDS/GENERIC/GOODIES | Wormwood、素食生物 |

【进阶】每个 FOODGROUP 是一个 `{name=..., types={...}}` 结构。它和直接传 `FOODTYPE.XXX` 字符串可以混着用。例如 Wigfrid 的 Diet 是 `{ FOODTYPE.MEAT, FOODTYPE.MONSTER }`，没用 FOODGROUP；而玩家默认 Diet 用的就是 `{ FOODGROUP.OMNI }`。

【老手】源码里 `Eater:GetEdibleTags` 把 FOODGROUP 展平：

```208:228:scripts/components/eater.lua
function Eater:GetEdibleTags()
    if self.cacheedibletags then
        return self.cacheedibletags
    end

    local tags = {}
    for i, v in ipairs(self.caneat) do
        if type(v) == "table" then
            for i2, v2 in ipairs(v.types) do
                table.insert(tags, "edible_"..v2)
            end
        else
            table.insert(tags, "edible_"..v)
        end
    end

    if self.cacheedibletags ~= false then
        self.cacheedibletags = tags
    end
    return tags
end
```

这段说明：**FOODGROUP 最终也会被翻译成一堆 `edible_<type>` 标签**。这就是 Edible 与 Eater 的"协议"——双方完全通过 `edible_<FOODTYPE>` 标签做匹配，不直接共享 lua 表。

---

### 六、Edible 的核心方法

#### 6.1 `Edible:GetHealth(eater)`：算出最终健康增量

```121:143:scripts/components/edible.lua
function Edible:GetHealth(eater)
    local multiplier = 1
    local healthvalue = self.gethealthfn ~= nil and self.gethealthfn(self.inst, eater) or self.healthvalue
    local spice_source = self.spice
    local ignore_spoilage = not self.degrades_with_spoilage or healthvalue < 0 or (eater ~= nil and eater.components.eater ~= nil and eater.components.eater.ignoresspoilage)

    if eater ~= nil and eater.components.eater ~= nil and eater.components.eater:CanProcessSpoiledItem(self.inst) then
        healthvalue = math.max(0, healthvalue)
    elseif not ignore_spoilage and self.inst.components.perishable ~= nil then
        if self.inst.components.perishable:IsStale() then
            multiplier = eater ~= nil and eater.components.eater ~= nil and eater.components.eater.stale_health or self.stale_health
        elseif self.inst.components.perishable:IsSpoiled() then
            multiplier = eater ~= nil and eater.components.eater ~= nil and eater.components.eater.spoiled_health or self.spoiled_health
            spice_source = nil
        end
    end

    if spice_source and TUNING.SPICE_MULTIPLIERS[spice_source] and TUNING.SPICE_MULTIPLIERS[spice_source].HEALTH then
        multiplier = multiplier + TUNING.SPICE_MULTIPLIERS[spice_source].HEALTH
    end

    return multiplier * healthvalue
end
```

【函数解析】

- **作用**：根据食物本身、腐烂状态、香料、吃食者的特性，计算最终回血量。
- **参数**：
  - `eater`（Entity，可选，无默认值）：吃这份食物的实体。注意可以传 nil（例如 hover 在 UI 上预览算值时）。
- **返回值**：
  - `(number)`：最终健康增量，可正可负。
- **实现流程**：
  1. `multiplier = 1`，先算基础值 `healthvalue`：如果有 `gethealthfn`，则调用 `gethealthfn(inst, eater)`，否则用 `self.healthvalue`。
  2. `spice_source = self.spice`（先记下来，腐烂时可能会清掉）。
  3. 判断是否跳过腐烂衰减：当 `degrades_with_spoilage == false`，**或** healthvalue 已经是负数（毒食物腐烂了不会变更毒），**或** eater 有 `ignoresspoilage` 标记（如鸟类、蜘蛛），都跳过腐烂衰减。
  4. 若 eater 是 `spoiledprocessor`（例如 Wickerbottom 的早期版本），把腐烂食物变废为宝（`healthvalue = max(0, healthvalue)`），直接清零所有负健康。
  5. 否则若 `perishable` 是 `IsStale()`，乘 `eater.stale_health` 或 `self.stale_health`（默认 0.333）。
  6. 否则若 `perishable` 是 `IsSpoiled()`，乘 `eater.spoiled_health` 或 `self.spoiled_health`（默认 0），同时清掉 `spice_source` —— **腐烂后香料没效果**。
  7. 最后加香料加成：`multiplier += TUNING.SPICE_MULTIPLIERS[spice].HEALTH`（如果有）。
  8. 返回 `multiplier * healthvalue`。

【涉及的标签】（仅本函数中出现）

- `ignoresspoilage` —— 见 6.4。
- `spoiledprocessor` / `allspoiledprocessor` —— 见 6.4。

#### 6.2 `Edible:GetHunger(eater)` 与 `Edible:GetSanity(eater)`

```96:119:scripts/components/edible.lua
function Edible:GetHunger(eater)
    local hungervalue = self.hungervalue
    local multiplier = 1
    local ignore_spoilage = not self.degrades_with_spoilage or hungervalue < 0 or (eater ~= nil and eater.components.eater ~= nil and eater.components.eater.ignoresspoilage)

    if eater ~= nil and eater.components.eater ~= nil and eater.components.eater:CanProcessSpoiledItem(self.inst) then
        hungervalue = math.max(0, hungervalue)
    elseif not ignore_spoilage and self.inst.components.perishable ~= nil then
        if self.inst.components.perishable:IsStale() then
            multiplier = eater ~= nil and eater.components.eater ~= nil and eater.components.eater.stale_hunger or self.stale_hunger
        elseif self.inst.components.perishable:IsSpoiled() then
            multiplier = eater ~= nil and eater.components.eater ~= nil and eater.components.eater.spoiled_hunger or self.spoiled_hunger
        end
    end

    if eater ~= nil and eater.components.foodaffinity ~= nil then
        local affinity_bonus = eater.components.foodaffinity:GetAffinity(self.inst)
        if affinity_bonus ~= nil then
            multiplier = multiplier * affinity_bonus
        end
    end

    return hungervalue * multiplier
end
```

【进阶】`GetHunger` 与 `GetHealth` 的**重要差别**：

1. `GetHunger` **不支持 `gethungerfn`**（没有 `SetGetHungerFn` 方法）。如果你想做"动态饱食食物"，得在 Eater 那边用 `custom_stats_mod_fn`。
2. `GetHunger` **会乘 foodaffinity 加成**（Warly 吃自己第一次做的菜 +33%）；但 `GetHealth` / `GetSanity` 不乘。
3. `GetHunger` **不受 spice 影响**。

```72:94:scripts/components/edible.lua
function Edible:GetSanity(eater)
    local sanityvalue = self.getsanityfn ~= nil and self.getsanityfn(self.inst, eater) or self.sanityvalue
    local ignore_spoilage = not self.degrades_with_spoilage or sanityvalue < 0 or (eater ~= nil and eater.components.eater ~= nil and eater.components.eater.ignoresspoilage)

    if eater ~= nil and eater.components.eater ~= nil and eater.components.eater:CanProcessSpoiledItem(self.inst) then
        sanityvalue = math.max(0, sanityvalue)
    elseif not ignore_spoilage and self.inst.components.perishable ~= nil then
        if self.inst.components.perishable:IsStale() then
            if sanityvalue > 0 then
                return 0
            end
        elseif self.inst.components.perishable:IsSpoiled() then
            return -TUNING.SANITY_SMALL
        end
    end

    local multiplier = 1
    if self.spice and TUNING.SPICE_MULTIPLIERS[self.spice] and TUNING.SPICE_MULTIPLIERS[self.spice].SANITY then
        multiplier = multiplier + TUNING.SPICE_MULTIPLIERS[self.spice].SANITY
    end

    return sanityvalue * multiplier
end
```

【老手】`GetSanity` 的腐烂规则**与健康/饱食不同**：

- Stale（陈旧）：只要 sanityvalue > 0，直接返回 0（不是乘倍率，是清零）。
- Spoiled（腐烂）：**统一返回 `-TUNING.SANITY_SMALL`**，无论你原本回多少 sanity。
- 香料同样作用（HEALTH 字段是健康加成，SANITY 字段是精神加成）。

#### 6.3 `Edible:OnEaten(eater)`：吃下后的统一回调

```183:214:scripts/components/edible.lua
function Edible:OnEaten(eater)
    if self.oneaten ~= nil then
        self.oneaten(self.inst, eater)
    end

    local delta_multiplier = 1
    local duration_multiplier = 1

    if self.spice and TUNING.SPICE_MULTIPLIERS[self.spice] then
        if TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDELTA then
            delta_multiplier = delta_multiplier + TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDELTA
        end

        if TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDURATION then
            duration_multiplier = duration_multiplier + TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDURATION
        end
    end

    -- Food is an implicit heater/cooler if it has temperature
    if self.temperaturedelta ~= 0 and
        self.temperatureduration ~= 0 and
        self.chill < 1 and
        eater ~= nil and
        eater.components.temperature ~= nil then
        eater.components.temperature:SetTemperatureInBelly(self.temperaturedelta * (1 - self.chill) * delta_multiplier, self.temperatureduration * duration_multiplier)
    end

    self.inst:PushEvent("oneaten", { eater = eater })
    if self.inst.eatensound ~= nil and eater.SoundEmitter ~= nil then
        eater.SoundEmitter:PlaySound(self.inst.eatensound)
    end
end
```

【函数解析】

- **作用**：在 `Eater:Eat` 算完三大属性、扣完 stack 之前调用，处理"附加效果"：自定义回调、肚中温度变化、`oneaten` 事件、咀嚼音效。
- **参数**：
  - `eater`（Entity）：吃这份食物的实体。
- **返回值**：无。
- **实现流程**：
  1. 若注册过 `self.oneaten`，调用 `self.oneaten(self.inst, eater)`。
  2. 计算 spice 对温度的加成（`TEMPERATUREDELTA` / `TEMPERATUREDURATION`）。
  3. 若 `temperaturedelta != 0`、`temperatureduration != 0`、`chill < 1`、eater 有 `temperature` 组件，调用 `eater.components.temperature:SetTemperatureInBelly(temperaturedelta * (1-chill) * 加成, temperatureduration * 加成)`。
  4. 推送 `oneaten` 事件到 self.inst，data = `{ eater = eater }`。
  5. 若 self.inst 有 `eatensound`（Lua 字段，不是组件），且 eater 有 `SoundEmitter`，播放音效。

【涉及的事件】（本函数推送的）

- **事件名**：`oneaten`
- **含义**：这份食物被吃了。
- **推送时机**：在 `Edible:OnEaten` 的尾部，**晚于** `Eater` 在玩家身上的 `oneat`，**早于** Edible 的物理移除。
- **推送数据**：
  - `eater`（Entity）：吃下这份食物的实体。
- **推送对象**：被吃的食物本身（`self.inst`）。

注意：`oneaten` 是 push 在**食物 prefab** 上的；玩家身上 push 的叫 `oneat`（在 `Eater:Eat` 里推送）。这是两个事件，区分清楚。

#### 6.4 `Edible:HandleEatRemove(eatwholestack)`

```216:226:scripts/components/edible.lua
function Edible:HandleEatRemove(eatwholestack) -- Called from eater.lua, internal, don't touch elsewhere!
    if self.inst:IsValid() then --might get removed in OnEaten...
        if self.handleremovefn ~= nil then
            self.handleremovefn(self.inst, eatwholestack)
        elseif not eatwholestack and self.inst.components.stackable ~= nil then
            self.inst.components.stackable:Get():Remove()
        else
            self.inst:Remove()
        end
    end
end
```

【函数解析】

- **作用**：处理"吃完后这份食物怎么办"——通常是从堆叠里减 1 个，或者整堆消失。
- **参数**：
  - `eatwholestack`（bool）：是否吃光整堆。来自 `eater.eatwholestack` 字段（如蛤蜊精吃整堆贝壳）。
- **返回值**：无。
- **实现流程**：
  1. 先检查 self.inst 是否还有效（因为 `OnEaten` 里 oneaten 回调可能已经 `Remove()` 了）。
  2. 若注册过 `handleremovefn`，调用 `handleremovefn(inst, eatwholestack)`，例如做"吃完变成 spoiled_food"。
  3. 否则若 `eatwholestack == false` 且有 stackable，`stackable:Get():Remove()` 取一个出来 remove。
  4. 否则直接 `inst:Remove()`。

#### 6.5 各 Setter 一览

| 方法 | 作用 |
| --- | --- |
| `SetOnEatenFn(fn)` | 注册 `oneaten` 回调，`fn(inst, eater)` |
| `SetHandleRemoveFn(fn)` | 注册自定义"吃完后处理"，`fn(inst, eatwholestack)` |
| `SetGetHealthFn(fn)` | 健康值由函数 `fn(inst, eater)` 动态决定 |
| `SetGetSanityFn(fn)` | 精神值由函数 `fn(inst, eater)` 动态决定 |
| `SetOverrideStackMultiplierFn(fn)` | 自定义堆叠倍率（替代 `stackable:StackSize()`） |
| `SetForceSpoiledFood(spoiled)` | 强制标记 / 取消 "spoiledfood" 标签 |

#### 6.6 温度相关：`AddChill` / `DiluteChill`

```238:249:scripts/components/edible.lua
function Edible:AddChill(delta)
    if self.temperaturedelta > 0 and not self.nochill then
        self.chill = math.clamp(self.chill + delta / self.temperatureduration, 0, 1)
    end
end

function Edible:DiluteChill(item, count)
    if self.temperaturedelta > 0 and not self.nochill and self.inst.components.stackable ~= nil and item.components.edible ~= nil then
        local stacksize = self.inst.components.stackable.stacksize
        self.chill = (stacksize * self.chill + count * item.components.edible.chill) / (stacksize + count)
    end
end
```

【老手】这两个函数描绘了"热食在冰箱里会变冷"的具体机制：

- `AddChill(delta)`：把 `chill` 增加 `delta / temperatureduration`，限制在 \[0, 1\]。仅对温度增益正向的食物生效（`temperaturedelta > 0`），且需要不是 `nochill`。
- `DiluteChill(item, count)`：把两份食物合并堆叠时，按比例稀释 chill。

【进阶】可以观察到只有"加热食物"（`temperaturedelta > 0`，比如热椒料理）才会 chill；冷食物（`temperaturedelta < 0`，比如冰）不会被"加热"。这是设计取舍——冷食物在物理意义上不会随时间变热，至少在游戏里没建模。

#### 6.7 `OnSave` / `OnLoad`

```251:259:scripts/components/edible.lua
function Edible:OnSave()
    return self.chill > 0 and { chill = self.chill } or nil
end

function Edible:OnLoad(data)
    if data.chill ~= nil and self.temperaturedelta > 0 and not self.nochill then
        self.chill = math.clamp(data.chill, 0, 1)
    end
end
```

【老手】只有 `chill > 0` 才存盘；这是因为其他所有字段都在 prefab 注册时静态写入，存盘没意义。chill 是运行时变量，需要保存。

#### 6.8 `OnRemoveFromEntity`

```56:66:scripts/components/edible.lua
function Edible:OnRemoveFromEntity()
    self.inst:RemoveTag("badfood")
    if self.foodtype ~= nil then
        self.inst:RemoveTag("edible_"..self.foodtype)
    end
    if self.secondaryfoodtype ~= nil then
        self.inst:RemoveTag("edible_"..self.secondaryfoodtype)
    end

    self.inst:RemoveTag("edible_"..FOODTYPE.BERRY)
end
```

【老手】最后那行 `inst:RemoveTag("edible_"..FOODTYPE.BERRY)` 显得很神秘——它**额外**清掉 BERRY 标签，即使从未设过。这是历史遗留的兼容代码：早期 berries 是用 secondary foodtype 直接 hack 出 BERRY 的，现在的 secondaryfoodtype 是后来加的，但移除时为了兼容老存档，统一兜底清一次。这是 Klei 自己留下的"老人加固"代码，**新代码不要模仿**。

---

### 七、Edible 与 Eater 的协作流程（精简版，详细见 20.2）

【进阶】玩家吃一份食物的完整链路（仅展示与 Edible 相关部分）：

1. 玩家执行 `ACTIONS.EAT`，目标是某个食物。
2. 进入 `Eater:Eat(food, feeder)`。
3. 调用 `self:PrefersToEat(food)`，内部用 `TestFood(food, self.preferseating)` —— 检查食物身上是否有 `edible_<某 FOODTYPE>` 标签且这个 FOODTYPE 在 Diet 列表里。
4. 通过的话，分别调用：
   - `food.components.edible:GetHealth(eater)`
   - `food.components.edible:GetHunger(eater)`
   - `food.components.edible:GetSanity(eater)`
5. 应用 `eater.healthabsorption` / `hungerabsorption` / `sanityabsorption` 系数。
6. 应用 `foodmemory:GetFoodMultiplier(prefab)`（吃腻打折）。
7. （可选）调用 `eater.custom_stats_mod_fn` 做最后一层修正。
8. 调用 `health:DoDelta` / `hunger:DoDelta` / `sanity:DoDelta` 真正应用变化。
9. PushEvent `oneat`（在玩家上）。
10. 调用 `eater.oneatfn(inst, food, feeder)`。
11. 调用 `food.components.edible:OnEaten(eater)`（这一步内会 push 食物上的 `oneaten` 事件）。
12. 调用 `food.components.edible:HandleEatRemove(eatwholestack)` 处理移除。
13. 更新 `lasteattime`、加入 `foodmemory`。

【老手】这个链路里，Edible 是"被使用方"，Eater 是"使用方"。所以 Edible 的字段大多是"对外暴露的可配置点"。

---

### 八、新手实战：定义一个能吃的食物

#### 8.1 最小可行食物（参考 `honey.lua` 简化）

```lua
local assets =
{
    Asset("ANIM", "anim/honey.zip"),
}

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)
    inst.AnimState:SetBuild("honey")
    inst.AnimState:SetBank("honey")
    inst.AnimState:PlayAnimation("idle")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")
    inst:AddComponent("stackable")
    inst.components.stackable.maxsize = 40

    inst:AddComponent("edible")
    inst.components.edible.foodtype = FOODTYPE.GENERIC
    inst.components.edible.healthvalue = 5
    inst.components.edible.hungervalue = 12.5
    inst.components.edible.sanityvalue = 0

    return inst
end

return Prefab("my_honey", fn, assets)
```

【新手】注意要点：

- `AddComponent("edible")` 必须在 `if not TheWorld.ismastersim then return inst end` **之后**，因为 Edible 是服务端组件。
- 你不必显式 `AddTag("edible_GENERIC")`——构造函数里已经默认 foodtype=GENERIC，setter 会自动打标签。
- `stackable` 是配套的，没有它吃光后整堆消失；有 stackable，则一次只消耗一个。

#### 8.2 让食物会腐烂

```lua
inst:AddComponent("perishable")
inst.components.perishable:SetPerishTime(TUNING.PERISH_MED)  -- 半个游戏年
inst.components.perishable:StartPerishing()
inst.components.perishable.onperishreplacement = "spoiled_food"
```

【新手】只要食物有 `perishable`，`Edible:GetHealth/GetHunger/GetSanity` 就会自动按 `stale_*` 和 `spoiled_*` 系数衰减。你不需要写额外代码。

#### 8.3 mod 里完整的入门例子（万物书的 `tbat_food_hedgehog_cactus_meat`）

```56:69:mods/联机版mod/万物书/scripts/prefabs/04_tbat_foods/01_hedgehog_cactus_meat.lua
        --- 可食用
            inst:AddComponent("edible")
            inst.components.edible.ismeat = true
            inst.components.edible.foodtype = FOODTYPE.MEAT
            inst.components.edible.healthvalue = 5
            inst.components.edible.hungervalue = 5
            inst.components.edible.sanityvalue = 5
        -----------------------------------------------------------
        --- 腐烂
            inst:AddComponent("perishable")
            inst.components.perishable:StartPerishing()
            inst.components.perishable.onperishreplacement = "spoiled_food"
            inst.components.perishable:SetPerishTime(TBAT.PARAM.ONE_DAY*10)
```

【进阶】注意 `ismeat = true` —— 这是个**老字段**。Klei 自己在 `scripts/prefabs/oceanfish.lua:518` 留了一句注释：

> `--edible.ismeat doesn't appear to actually be used anywhere, might not be necessary.`

也就是说官方虽然在大量肉类 prefab（meats / fish / wobster / wereitems / woby_treat / trunk / pondfish 等）里都会设它，但**当前版本所有"是不是肉"的判断已经迁移到 `foodtype == FOODTYPE.MEAT`**（例如 `meatrack2.lua` 第 228 行）。`ismeat` 是历史遗留兼容字段，新代码可以省略它，但跟随官方风格保留也无害。

---

### 九、进阶要点

#### 9.1 主类型 + 次类型：怎么让 mod 食物同时属于"肉"和"怪物"

参考 `meats.lua` 怪物肉的写法：

```271:283:scripts/prefabs/meats.lua
local function monster()
    --selfstacker (from selfstacker component) added to pristine state for optimization
    local inst = common("monstermeat", "meat_monster", "idle", { "monstermeat", "selfstacker" }, { product = "monstermeat_dried", time = TUNING.DRY_FAST }, { product = "cookedmonstermeat" })

    if not TheWorld.ismastersim then
        return inst
    end

    inst.components.edible.secondaryfoodtype = FOODTYPE.MONSTER
    inst.components.edible.healthvalue = -TUNING.HEALING_MED
    inst.components.edible.hungervalue = TUNING.CALORIES_MEDSMALL
    inst.components.edible.sanityvalue = -TUNING.SANITY_MED
```

【进阶】`common` 函数里已经把 `foodtype = FOODTYPE.MEAT` 设过了，这里再追加 `secondaryfoodtype = FOODTYPE.MONSTER`。结果是怪物肉同时挂着 `edible_MEAT` 和 `edible_MONSTER` 两个标签。

【老手】这样设计后，假设你做个 mod 角色"专门吃怪物食物"，给它的 Eater 配 `{ FOODTYPE.MONSTER }` Diet 就能让它吃怪物肉 + durian + monstermeat 等所有怪物类食物。

#### 9.2 香料（spice）

【进阶】Klei 在 `tuning.lua` 里定义了 `SPICE_MULTIPLIERS`，但官方目前只用了 `SPICE_SALT = { HEALTH = 0.25 }`，其他都是预留接口。完整支持的字段：

```lua
SPICE_MULTIPLIERS[spice_key] = {
    HEALTH             = 0.25,  -- 健康加成（加性叠到 multiplier）
    SANITY             = 0.10,  -- 精神加成
    TEMPERATUREDELTA   = 0.20,  -- 体温变化加成
    TEMPERATUREDURATION= 0.20,  -- 体温持续加成
}
```

【老手】使用方式：在烹饪完后把 `edible.spice` 设为 spice_key 即可。Edible 的 `GetHealth` / `GetSanity` / `OnEaten` 都会自动读取。

#### 9.3 动态健康：`SetGetHealthFn` 实战

【进阶】鸵鸟蛋的"裂蛋"在被吃时会让玩家说一句台词，但不是动态健康——它用的是 `SetOnEatenFn`：

```146:150:scripts/prefabs/tallbirdegg.lua
local function OnEaten(inst, eater)
    if eater.components.talker ~= nil then
        eater.components.talker:Say( GetString(eater, "EAT_FOOD", "TALLBIRDEGG_CRACKED") )
    end
end
```

【老手】如果你要做"健康根据玩家当前生命百分比变化"的食物（例如剩血少时回血多）：

```lua
local function GetHealthFn(inst, eater)
    if eater and eater.components.health then
        local hp_percent = eater.components.health:GetPercent()
        if hp_percent < 0.3 then
            return TUNING.HEALING_LARGE   -- 残血回大血
        else
            return TUNING.HEALING_SMALL   -- 普通回小血
        end
    end
    return TUNING.HEALING_TINY
end

inst.components.edible:SetGetHealthFn(GetHealthFn)
```

**注意**：因为 `SetGetHealthFn` 注册的函数返回值会被 `GetHealth` 当成 healthvalue 用，**所以原本的 `healthvalue` 不会再起作用了**——但 `badfood` 标签的判定还是基于 `healthvalue` 字段。要保证标签正确，记得手动 `inst.components.edible.healthvalue = 一个有代表性的值`。

#### 9.4 温度食物：冰

```86:92:scripts/prefabs/inv_rocks_ice.lua
    inst:AddComponent("edible")
    inst.components.edible.foodtype = FOODTYPE.GENERIC
    inst.components.edible.healthvalue = TUNING.HEALING_TINY/2
    inst.components.edible.hungervalue = TUNING.CALORIES_TINY/4
    inst.components.edible.degrades_with_spoilage = false
    inst.components.edible.temperaturedelta = TUNING.COLD_FOOD_BONUS_TEMP
    inst.components.edible.temperatureduration = TUNING.FOOD_TEMP_BRIEF * 1.5
```

【进阶】五行配置就是完整冰食物：

- `foodtype = GENERIC`：所有杂食角色都能吃。
- `degrades_with_spoilage = false`：冰即使在 perishable 即将融化时也不"陈旧/腐烂"打折，吃下去就是吃下去。
- `temperaturedelta < 0`（COLD_FOOD_BONUS_TEMP 是负值）：在 eater 肚子里制冷。
- `temperatureduration`：制冷持续时间。

【老手】温度食物有个看不见的"chill 衰减"：热食物（temperaturedelta > 0）放久了会被环境降温，吃下去效果减弱；冷食物因为代码里 `if self.temperaturedelta > 0` 的限制不会被加热。这是模拟"热菜会凉、凉菜不会自己变热"的设计。

#### 9.5 `foodaffinity`（食物亲和）

【进阶】Edible **仅在 `GetHunger` 中**查询 `eater.components.foodaffinity:GetAffinity(self.inst)`，所以**饥饿值会被亲和加成**，但其他属性不会。

`foodaffinity` 是一个通用组件（`scripts/components/foodaffinity.lua`），不是 Warly 专有。它支持三种维度的亲和：

- `AddPrefabAffinity(prefab, bonus)`：对某个 prefab 的整体倍率。
- `AddFoodtypeAffinity(foodtype, bonus)`：对某个 FOODTYPE 的倍率。
- `AddTagAffinity(tag, bonus)`：对带某标签的食物的倍率。

【老手】具体倍率来自角色 prefab 的注册，例如：

- WX-78 注册了 `butterflymuffin` → `TUNING.AFFINITY_15_CALORIES_LARGE`（= 1.4，即 +40% 饱食）。
- Walter 注册了 `trailmix` → `TUNING.AFFINITY_15_CALORIES_SMALL`（= 2.2，即 +120% 饱食）。
- Wormwood（Merm 也有类似机制）注册了 `kelp` / `durian` 等"原本扣血"食物为 `1`（恰好抵消负面）。

亲和倍率值预设在 `scripts/tuning.lua:2052`：

```2052:2056:scripts/tuning.lua
		AFFINITY_15_CALORIES_SMALL = 2.2,
		AFFINITY_15_CALORIES_MED = 1.6,
		AFFINITY_15_CALORIES_LARGE = 1.4,
		AFFINITY_15_CALORIES_HUGE = 1.2,
		AFFINITY_15_CALORIES_SUPERHUGE = 1.1,
```

设计思路是：饱食上限大的食物，亲和加成的倍率反而越接近 1，避免造成"一个馅饼能补满"的失衡。注册时需要根据食物本身的饱食量级匹配对应常量。

#### 9.6 `foodmemory`（吃腻系统）

【进阶】不在 Edible 里实现，而在 `Eater:Eat` 里查询：

```236:237:scripts/components/eater.lua
        local stack_mult = self.eatwholestack and food.components.edible:GetStackMultiplier() or 1
        local base_mult = self.inst.components.foodmemory ~= nil and self.inst.components.foodmemory:GetFoodMultiplier(food.prefab) or 1
```

`base_mult` 是 prefab 维度的吃腻倍率，0~1 之间。重复吃同样的 prefab 越多，倍率越低。这就是 Warly "吃腻"系统。

---

### 十、老手深挖

#### 10.1 标签的"反向工程"

【老手】Edible 暴露给世界的接口只有 **标签**：

| 标签 | 何时打 | 何时移除 | 含义 |
| --- | --- | --- | --- |
| `edible_<FOODTYPE>` | foodtype 改变时 | foodtype 改变 / 组件被移除 | 这份食物属于该类型 |
| `badfood` | health 或 sanity < 0 | 不再 < 0 / 组件被移除 | 这份食物有副作用 |
| `spoiledfood` | `SetForceSpoiledFood(true)` 调用 | 同上 false | 这份本身就是"已腐烂食物"prefab |

【老手】**所有的 CanEat/PrefersToEat 判定都是基于标签**，不是基于读 `food.components.edible.foodtype`。这给你一个 dirty trick：你可以**在不挂 Edible 组件的物品上手动 AddTag("edible_MEAT")**，让某些 AI 把它当肉认（但玩家吃它就会崩，因为 `Eater:Eat` 里会读 `food.components.edible`）。

#### 10.2 `cacheedibletags` 性能优化

【老手】`Eater:GetEdibleTags` 缓存了一份"我能吃的所有 edible_xxx 标签"，避免每帧 FindEntities 时反复展平 FOODGROUP。如果你动态改 Diet（例如某些 buff），记得：

```lua
inst.components.eater.cacheedibletags = false
```

清空缓存，下次会重建。

#### 10.3 `nochill = true` 的妙用

【老手】例如香料火药、辣椒奶昔这些"主动加热"的食物，吃下去你想保证一定燃烧、不希望放仓库后效果衰减，就设 `nochill = true`。这样 `AddChill` / `DiluteChill` 都会被跳过，但温度增益依然在 `OnEaten` 里生效（因为 `OnEaten` 检查 `chill < 1`，初始 0，不衰减时永远 0）。

#### 10.4 `eatensound`：奇怪的非组件字段

```210:213:scripts/components/edible.lua
    self.inst:PushEvent("oneaten", { eater = eater })
    if self.inst.eatensound ~= nil and eater.SoundEmitter ~= nil then
        eater.SoundEmitter:PlaySound(self.inst.eatensound)
    end
```

【老手】`inst.eatensound` 是直接挂在 entity 上的 Lua 字段（**不是** Edible 组件字段、**不是** AnimState 状态）。在 prefab fn 里这样写即可：

```lua
inst.eatensound = "dontstarve/wilson/eat_meat"
```

它会让吃下这份食物的人发出咀嚼声。这是个零开销机制，比专门做个组件简洁。

#### 10.5 `OnRemoveFromEntity` 的 BERRY 兜底

前面 6.8 已经讲过，这里强调一次：**移除 Edible 时会无脑清一次 `edible_BERRY`**，这是兼容老存档的兜底。你看到时不要疑惑。

#### 10.6 `oncheckbadfood` 不考虑动态值

```1:7:scripts/components/edible.lua
local function oncheckbadfood(self)
    if self.healthvalue < 0 or (self.sanityvalue ~= nil and self.sanityvalue < 0) then
        self.inst:AddTag("badfood")
    else
        self.inst:RemoveTag("badfood")
    end
end
```

【老手】注意它只看静态的 `healthvalue` / `sanityvalue` 字段，**不调用 `gethealthfn` / `getsanityfn`**。所以：

- 如果你的食物用 `gethealthfn` 返回的值可能为负，但 `healthvalue` 默认是正数，那它**不会**被标记为 `badfood`。
- 解决办法：手动 `inst.components.edible.healthvalue = -1`（不需要准确，只要符号对即可，因为真正的伤害还是来自 `gethealthfn`）。

---

### 十一、本节涉及的标签与事件（速查）

#### 标签清单（出自 `edible.lua` 与协作组件）

| 标签 | 含义 | 谁打 / 谁读 |
| --- | --- | --- |
| `edible_<FOODTYPE>` | 这份食物属于该 FOODTYPE | Edible:setter `onfoodtype` 打 / Eater:TestFood 读 |
| `badfood` | 健康或精神为负 | Edible:setter `oncheckbadfood` 打 / 多个 AI 读 |
| `spoiledfood` | 这份就是腐烂物（如 `spoiled_food` prefab） | Edible:SetForceSpoiledFood 打 / 灯泡蘑菇等读 |
| `<FOODTYPE>_eater` | 该实体能吃该 FOODTYPE 类型 | Eater:setter `oncaneat` 打 / 食物 AI 读 |
| `<FOODGROUP_NAME>_eater` | 该实体能吃该 FOODGROUP | Eater:setter `oncaneat` 打 |
| `ignoresspoilage` | 该实体吃东西无视腐烂 | Eater:SetIgnoresSpoilage 打 / Edible:Get* 读 |
| `spoiledprocessor` | 该实体能把腐烂食物变成 0 伤害 | Eater:SetSpoiledProcessor 打 / Edible:Get* 读 |
| `allspoiledprocessor` | 增强版 spoiledprocessor，连接 perishable | 同上 |
| `nospoiledfood` | 该实体拒绝腐烂食物 | Eater:SetRefusesSpoiledFood 打 |
| `strongstomach` | 强胃，对怪物肉无副作用 | Eater:SetStrongStomach 打 / Eater:DoFoodEffects 读 |
| `eatsrawmeat` | 能吃生肉无副作用 | Eater:SetCanEatRawMeat 打 |

#### 事件清单（本节涉及的）

**1. `oneaten`**

- **含义**：食物 prefab 被吃下。
- **推送时机**：`Edible:OnEaten` 中段，已应用温度但还未 `HandleEatRemove`。
- **推送数据**：
  - `eater`（Entity）：吃这份食物的实体。
- **推送对象**：食物本身（`food.inst`）。

**2. `oneat`**（属于下节 Eater 的内容，但与 oneaten 易混，列出对照）

- **含义**：实体（如玩家）吃了某份食物。
- **推送时机**：`Eater:Eat` 中段，扣完属性后、调用 `Edible:OnEaten` 之前。
- **推送数据**：
  - `food`（Entity）：吃下的食物。
  - `feeder`（Entity）：投喂者（自己吃自己时 feeder == self.inst）。
- **推送对象**：吃食物的实体（`eater.inst`）。

**3. `feedincontainer`**

- **含义**：通过容器（如自己的背包）喂食给别人。
- **推送时机**：feeder ≠ self.inst，且物品的 grandowner 是 feeder 或者 feeder 打开了 grandowner 的容器。
- **推送数据**：无。
- **推送对象**：feeder。

**4. `feedmount`**

- **含义**：给坐骑喂食。
- **推送时机**：feeder ≠ self.inst，且 eater 是 rideable 且 feeder 是骑手。
- **推送数据**：
  - `food`（Entity）。
  - `eater`（Entity）：坐骑。
- **推送对象**：feeder。

---

### 十二、常见坑与最佳实践（Checklist）

【新手】

- [ ] `AddComponent("edible")` 一定要在 `if not TheWorld.ismastersim` **之后**。
- [ ] 别忘了配 `stackable`，否则吃一次整堆消失。
- [ ] foodtype 不设 = GENERIC，多数情况够用；想做"只有特定角色能吃"再考虑改。

【进阶】

- [ ] 同时设主+次类型时，确保它们不相等（assert 会崩服）。
- [ ] 食物 mod 给负健康/负精神时，记得 `healthvalue` / `sanityvalue` 字段也要写负值（保证 `badfood` 标签正确）。
- [ ] 用 `gethealthfn` 时，记得手动维护 `healthvalue` 的代表符号。
- [ ] 食物有 spice 时，注意香料 multiplier 是**加性**叠加到基础 1 上，不是乘性。

【老手】

- [ ] 想做"夹生肉、外面腐里面新鲜"这种伪 perishable 食物，`degrades_with_spoilage = false` 比手动魔改腐烂更稳。
- [ ] 食物在容器里被 `Eater:Eat`，会推 `feedincontainer` —— 这是 Wormwood 给坐骑/盆栽喂养的关键触发点，做新角色/新坐骑时记得监听。
- [ ] 缓存 `cacheedibletags` 在动态 Diet（如 buff、装备）变更时记得失效。
- [ ] 不要乱写 `inst:AddTag("edible_xxx")`——所有 edible_xxx 必须通过 `edible.foodtype` 字段统一管控，否则会出现 OnRemoveFromEntity 清不掉的脏标签。

---

至此 20.1 Edible 完结。下一节进入 Eater 组件，会把"吃下去后发生的事"展开讲透。


## 20.2 Eater 组件——谁能吃什么（Diet 系统与 CanEat 判定）

> 本节面向三类读者编写，章节中会以「【新手】」「【进阶】」「【老手】」分别标注关键信息，方便不同层次的读者各取所需。
>
> - **【新手】**：第一次给 mod 角色 / 生物挂"嘴"，目标是能正确选择 Diet、吸收倍率，让目标"能吃东西、能加饱食"。
> - **【进阶】**：理解 `caneat` 和 `preferseating` 的真正区别、`Warly` 的"只吃料理"是怎么做的、`Woby` 的"吃 treat 三倍饱食"是怎么写的。
> - **【老手】**：读懂 `Eater:Eat` 全链路，写自定义的 `oneatfn` / `custom_stats_mod_fn` / 喂食判定，做"吃完整堆"、"吃 buff"、"挑食"、"无视腐烂"等复杂行为。
>
> 本节涉及的源码：
> - `scripts/components/eater.lua`（核心，本节主角，全文 371 行）
> - `scripts/components/edible.lua`（上一节，本节会引用 `GetHealth`/`GetHunger`/`GetSanity`）
> - `scripts/components/foodaffinity.lua`（食物亲和，前置）
> - `scripts/components/foodmemory.lua`（吃腻，Warly 专用）
> - `scripts/actions.lua` 第 748-771 行：`ACTIONS.EAT.fn`
> - `scripts/constants.lua` 第 2010-2093 行：`FOODTYPE` / `FOODGROUP`
>
> 教学示例 prefab：
> - `scripts/prefabs/wathgrithr.lua`（Wigfrid）：`caneat` ≠ `preferseating` 的经典案例
> - `scripts/prefabs/warly.lua`：`SetPrefersEatingTag` 的"只吃料理"机制
> - `scripts/prefabs/wormwood.lua`：`SetAbsorptionModifiers(0, 1, 1)` 的素食心机
> - `scripts/prefabs/wx78.lua` / `scripts/prefabs/wx78_possessedbody.lua`：`SetIgnoresSpoilage` + `SetCanEatGears` + `SetOnEatFn`
> - `scripts/prefabs/wobysmall.lua` / `scripts/prefabs/wobybig.lua`：`custom_stats_mod_fn` 的实战
> - `scripts/prefabs/bearger.lua`：`eatwholestack` 的整堆吞咽
> - `scripts/prefabs/spider.lua`：Diet + 强胃 + 吃生肉
> - `scripts/prefabs/hats.lua` 第 2344-2369 行（眼面具帽 eyemask）：`SetAbsorptionModifiers` + 自定义 `oneatfn` 修甲
> - `mods/联机版mod/勋章/scripts/medal_defs/medal_buff_defs.lua` 第 220-273 行：Diet 备份与还原（buff 全食物）

---

### 一、为什么需要 Eater 组件？（新手必读）

【新手】上一节 `Edible` 解决了"这东西是不是食物"，本节 `Eater` 解决"这家伙能不能、想不想吃这份食物"。两个组件配合，才能完成"玩家吃苹果"这件事：

- **谁是吃方**：玩家、生物、宠物、坐骑——只要挂了 `Eater` 组件，就有"嘴"。
- **吃方的喜好**：能吃哪些 `FOODTYPE`？只想吃哪些？吃下去是否吸收？吃下去给什么副作用？——全部 `Eater` 字段控制。
- **吃方的副作用**：玩家吃怪物肉掉精神、吃硝石变 buff、吃药水有词条——全部 `oneatfn` / `custom_stats_mod_fn` / 事件控制。

【进阶】更精确：`Eater` 是一个**服务端组件**（玩家在 `player_common.lua:2783` 处 `AddComponent("eater")`，且是 `if TheWorld.ismastersim` 内），它做四件事：

1. **能吃判定**：暴露 `Eater:CanEat(food)`、`Eater:PrefersToEat(food)`，让外部（行动、UI、AI）问"能不能吃"。
2. **打类型标签**：在 entity 上打 `<FOODTYPE>_eater` / `<FOODGROUP_NAME>_eater` 标签，让食物侧（`Edible`、`badfood` AI 等）反向查询"这个生物能吃我吗"。
3. **执行进食**：`Eater:Eat(food, feeder)` 是真正吃下去的入口，它去问 `Edible:GetHealth/Hunger/Sanity`、应用吸收倍率、推事件、回调、移除食物。
4. **记录最后进食时间**：`lasteattime`，供饥饿 / 行为树（brain）判断"该不该再吃"。

【老手】`Eater` 不直接修改任何属性，**真正调 `DoDelta` 的就在 `Eater:Eat` 内部**——这点和 Edible 完全不同：Edible 只算，不动；Eater 算完之后**自己**动手。也就是说，`Eater:Eat` 是这个系统的总控函数，剩下所有东西都是它的下游 / 参数。

---

### 二、源码总览：`eater.lua` 的骨架

```18:46:scripts/components/eater.lua
local Eater = Class(function(self, inst)
    self.inst = inst
    self.eater = false
    self.strongstomach = false
    self.preferseating = { FOODGROUP.OMNI }
    --self.perferseatingtags = nil
    self.caneat = { FOODGROUP.OMNI }
    self.oneatfn = nil
    self.lasteattime = nil
    self.ignoresspoilage = false
    self.eatwholestack = false
--[[
    --can be overridden by prefabs
    self.stale_hunger = nil
    self.stale_health = nil
    self.spoiled_hunger = nil
    self.spoiled_health = nil
]]
    self.healthabsorption = 1
    self.hungerabsorption = 1
    self.sanityabsorption = 1

    --set to false to disable cached tags
    --self.cacheedibletags = nil
end,
nil,
{
    caneat = oncaneat,
})
```

【新手】只看构造函数：默认 `caneat = preferseating = { FOODGROUP.OMNI }`，吸收倍率全 1。所以**只要 `AddComponent("eater")` 不动任何字段，得到的就是一个"标准杂食玩家"**。

【进阶】注意第 3、4 个 `Class` 参数：`nil` 是 onsetparent，setter 表里只有 `caneat = oncaneat`。也就是说**只有改 `caneat` 才会触发自动打标签**，**改 `preferseating` 不会触发**。这是 Klei 的设计：preferseating 是软偏好，不影响标签；caneat 才是硬资格，影响标签匹配。

【老手】特别留意源码注释里的拼写错误：`--self.perferseatingtags = nil`，实际字段叫 `preferseatingtags`（少打了一个 e）。源码注释只是给作者自己看的提醒，不要直接复制粘贴。

#### setter 触发的关键动作

```1:16:scripts/components/eater.lua
local function clearcaneat(self, caneat)
    for i, v in ipairs(caneat) do
        self.inst:RemoveTag((type(v) == "table" and v.name or v).."_eater")
    end
end

local function oncaneat(self, caneat, old_caneat)
    if old_caneat ~= nil then
        clearcaneat(self, old_caneat)
    end
    if caneat ~= nil then
        for i, v in ipairs(caneat) do
            self.inst:AddTag((type(v) == "table" and v.name or v).."_eater")
        end
    end
end
```

【进阶】`oncaneat` 是当 `caneat` 被赋值时触发的 setter。它会：

1. 先 `clearcaneat`，把旧 `caneat` 对应的所有 `<X>_eater` 标签清掉；
2. 再为新 `caneat` 的每一项打上 `<X>_eater` 标签。
3. 兼容两种写法：直接的 FOODTYPE 字符串（如 `"MEAT"`）打 `"MEAT_eater"`；FOODGROUP 表（带 `.name` 字段）打 `"OMNI_eater"`、`"BEARGER_eater"` 等。

【老手】这意味着**修改 `eater.caneat` 是有副作用的**，会增删大量标签。如果你正在做"全食物 buff"或者"挑食 debuff"，记得**用 `SetDiet`**（它内部直接赋值 `self.caneat`，触发 setter），不要"手动 `table.insert`"——后者绕过 setter 不会自动打标签。

---

### 三、字段逐项解析

| 字段名 | 类型 | 默认值 | 含义 |
| --- | --- | --- | --- |
| `inst` | Entity | 自动赋值 | 宿主实体 |
| `eater` | bool | false | 历史遗留字段，已基本不再使用，无明确语义 |
| `strongstomach` | bool | false | 强胃：吃怪物肉无副作用 |
| `eatsrawmeat` | bool\|nil | nil | 能吃生肉无副作用 |
| `preferseating` | table | `{ FOODGROUP.OMNI }` | **偏好** Diet，决定 `PrefersToEat` 是否通过 |
| `caneat` | table | `{ FOODGROUP.OMNI }` | **资格** Diet，决定 `CanEat` 是否通过、自动打 `<X>_eater` 标签 |
| `preferseatingtags` | table\|nil | nil | 偏好标签清单：必须先满足才能进 preferseating 判定（Warly 用） |
| `oneatfn` | function\|nil | nil | 吃下后回调 `fn(inst, food, feeder)` |
| `custom_stats_mod_fn` | function\|nil | nil | 进食增量最后一层修正 `fn(inst, h, hg, s, food, feeder)` |
| `lasteattime` | number\|nil | nil | 最后一次吃下去的 `GetTime()` |
| `ignoresspoilage` | bool | false | 吃东西无视腐烂衰减 |
| `eatwholestack` | bool | false | 吃下去整堆消耗（且按堆数倍叠加属性） |
| `healthabsorption` | number | 1 | 健康吸收倍率 |
| `hungerabsorption` | number | 1 | 饱食吸收倍率 |
| `sanityabsorption` | number | 1 | 精神吸收倍率 |
| `stale_hunger` | number\|nil | nil | 自定义"陈旧饱食倍率"，覆盖 Edible 的 stale_hunger（默认 0.667） |
| `stale_health` | number\|nil | nil | 自定义"陈旧健康倍率" |
| `spoiled_hunger` | number\|nil | nil | 自定义"腐烂饱食倍率" |
| `spoiled_health` | number\|nil | nil | 自定义"腐烂健康倍率" |
| `nospoiledfood` | bool\|nil | nil | 拒绝腐烂食物（`PrefersToEat` 直接返回 false） |
| `spoiledprocessor` | bool\|nil | nil | 能把腐烂食物变 0 伤害（不能让属性负） |
| `allspoiledprocessor` | bool\|nil | nil | 增强版 spoiledprocessor，连"perishable 已腐烂"也能处理 |
| `cacheedibletags` | table\|false\|nil | nil | `GetEdibleTags` 缓存；设 false 禁用缓存 |

【新手】最常用的 5 个字段：`caneat`、`preferseating`（用 `SetDiet` 设）、`healthabsorption`、`hungerabsorption`、`sanityabsorption`（用 `SetAbsorptionModifiers` 设）。其他默认即可。

【进阶】Klei 自己在 `Eater` 注释里写过：`eater = false` 这个字段在整个 codebase 没人读，**几乎可以确定是死字段**——但保留是为了存档兼容。新代码不要依赖它。

【老手】`stale_hunger`/`stale_health`/`spoiled_hunger`/`spoiled_health` 这四个字段是 nil 时，`Edible:GetHealth`/`GetHunger` 会回落到 `food.components.edible.stale_*`。看上节 6.1 / 6.2 的实现：

```scripts/components/edible.lua (摘自上节)
multiplier = eater ~= nil and eater.components.eater ~= nil
    and eater.components.eater.stale_health or self.stale_health
```

也就是 **Eater 上覆写值优先于食物自己的默认值**。这给你一个 trick：可以做"某 buff 让玩家陈旧吃也满血"——直接 `eater.stale_health = 1`。

---

### 四、`caneat` 与 `preferseating`：核心 Diet 字段的差别

【进阶】这是新人最容易混淆的点。源码注释把这件事讲得很清楚：

```232:235:scripts/components/eater.lua
    -- This used to be CanEat. The reason for two checks is to that special diet characters (e.g.
    -- wigfrid) can TRY to eat all foods (they get the actions for it) but upon actually put it in
    -- their mouth, they bail and "spit it out" so to speak.
    if self:PrefersToEat(food) then
```

**翻译**：原本游戏只有 `CanEat` 一个判定，但需要做 Wigfrid 这种角色"看上去想吃，但凑近嘴巴又吐了"的效果，于是分裂成两个：

| 概念 | 字段 | 由谁用 |
| --- | --- | --- |
| 资格 (CanEat) | `caneat` | 行动菜单是否显示"吃"；自动打 `<X>_eater` 标签；外部 AI 查询 |
| 偏好 (PrefersToEat) | `preferseating` | `Eater:Eat` 真正吃下去时检查；不通过则原地什么都不发生 |

【进阶】**总不变量**：`preferseating ⊆ caneat`。Wigfrid 的配置是经典：

```167:169:scripts/prefabs/wathgrithr.lua
    if inst.components.eater ~= nil then
        inst.components.eater:SetDiet({ FOODGROUP.OMNI }, { FOODTYPE.MEAT, FOODTYPE.GOODIES })
    end
```

- `caneat = FOODGROUP.OMNI`（看上去能吃所有东西，UI 上"吃"选项可点）
- `preferseating = { FOODTYPE.MEAT, FOODTYPE.GOODIES }`（实际只吞肉和零食，其他原样吐回）

【老手】这种"行动可触发但真正调用拒绝"是非常优雅的设计：

- 玩家点了"吃苹果"，`ACTIONS.EAT` 进入 `eater:Eat`；
- `PrefersToEat(苹果)` 用 `preferseating` 判，发现苹果是 VEGGIE，不在 `{MEAT, GOODIES}` 列表中，返回 false；
- `Eat` 函数直接 return（没有 return 值，Lua 中等价 nil = false）；
- 行动结果：玩家什么也没发生，苹果还在手里。

【老手】**坑**：因为 `caneat` 是 OMNI、UI 上"吃"按钮可点，玩家会觉得"我点了为什么没反应"。Klei 用台词系统补救——`speech_wathgrithr.lua` 有大量"我才不吃这种植物饲料"的台词通过状态机触发，玩家就明白了。如果你 mod 一个挑食角色，**记得加台词反馈**。

---

### 五、Eater 的核心方法

#### 5.1 `Eater:SetDiet(caneat, preferseating)`

```52:55:scripts/components/eater.lua
function Eater:SetDiet(caneat, preferseating)
    self.caneat = caneat
    self.preferseating = preferseating or caneat
end
```

【函数解析】

- **作用**：一次性设置吃方的两个 Diet 列表。
- **参数**：
  - `caneat`（table，必填，无默认值）：资格 Diet 列表。元素可以是 FOODTYPE 字符串（如 `FOODTYPE.MEAT`）或 FOODGROUP 表（如 `FOODGROUP.OMNI`），二者可混合。
  - `preferseating`（table\|nil，可选，默认 nil）：偏好 Diet 列表。**为 nil 时复用 `caneat`**，二者指向同一个表。
- **返回值**：无。
- **实现流程**：
  1. 直接给 `self.caneat` 赋值，触发 setter `oncaneat`：清除旧标签、打新标签。
  2. 给 `self.preferseating` 赋值，没有 setter，仅记录。

【新手】**这是 99% 情况你设置 Diet 的唯一接口**。模板：

```lua
inst.components.eater:SetDiet({ FOODTYPE.VEGGIE }, { FOODTYPE.VEGGIE })  -- 兔人、麋鹿驼
inst.components.eater:SetDiet({ FOODTYPE.MEAT }, { FOODTYPE.MEAT })       -- 蜘蛛、狗
inst.components.eater:SetDiet({ FOODGROUP.OMNI }, { FOODGROUP.OMNI })     -- 玩家、隐士螃蟹
```

【进阶】`preferseating or caneat` 的二选一是为了简写：如果你只想做"看上去能吃啥就真吃啥"的普通生物，第二参传 nil 即可，省一行。

【老手】两个表指向同一个引用时，**任何对一个的 `table.insert` 都会同时改另一个**。`SetCanEatHorrible` 等内置函数正是利用这一点：

```85:89:scripts/components/eater.lua
function Eater:SetCanEatHorrible()
    table.insert(self.preferseating, FOODTYPE.HORRIBLE)
    table.insert(self.caneat, FOODTYPE.HORRIBLE)
    self.inst:AddTag(FOODTYPE.HORRIBLE.."_eater")
end
```

源码两个表分别插了一次 —— 但如果你 `SetDiet({...}, nil)` 这种简写之后又用 `SetCanEatHorrible`，**`HORRIBLE` 会被插两次到同一个表里**。这是一个真实的源码隐患，但因为 `TestFood` 只看"标签是否存在"不在乎重复，所以表现上没问题。**强迫症 mod 作者要注意，写新代码时显式传两个独立表**。

#### 5.2 `Eater:SetAbsorptionModifiers(health, hunger, sanity)`

```57:61:scripts/components/eater.lua
function Eater:SetAbsorptionModifiers(health, hunger, sanity)
    self.healthabsorption = health
    self.hungerabsorption = hunger
    self.sanityabsorption = sanity
end
```

【函数解析】

- **作用**：设置三项属性的吸收倍率。
- **参数**：
  - `health`（number）：健康倍率。0 = 完全不吸收（吃药都不回血），1 = 标准，2 = 翻倍。
  - `hunger`（number）：饱食倍率。
  - `sanity`（number）：精神倍率。
- **返回值**：无。
- **实现流程**：依次赋三个字段，无 setter。

【新手】典型用例：

| 角色 / 生物 | 配置 | 解读 |
| --- | --- | --- |
| Wormwood | `(0, 1, 1)` | 食物不回血（必须通过种植 / 肥料） |
| 河马 beefalo | `(4, 1, 1)` | 喂食回血效果 ×4，调教用 |
| Wortox | `(TUNING.WORTOX_FOOD_MULT, ...)` ≈ 0.5 三项全部 | 食物效果减半（靠灵魂吃饭） |
| 大眼罩 eyemask | `(4.0, 1.75, 0)` | 健康强吸收（修甲多），精神 0（不掉 sanity） |

```797:800:scripts/prefabs/wormwood.lua
    if inst.components.eater then
        --No health from food
        inst.components.eater:SetAbsorptionModifiers(0, 1, 1)
    end
```

【进阶】注意倍率应用顺序（详见 6 节）：

```245:245:scripts/components/eater.lua
            health_delta = food.components.edible:GetHealth(self.inst) * base_mult * self.healthabsorption
```

即 `健康增量 = Edible.GetHealth(eater) × foodmemory倍率 × 吸收倍率`。三个倍率**乘性叠加**。如果 healthabsorption=0，无论 GetHealth 多大，最终都是 0。

【老手】**Wormwood 的负值食物会怎么样？**例如吃榴莲 (`healthvalue = -10`)，因为 `0 * -10 = 0`，**Wormwood 吃榴莲完全不掉血**。但他**仍然会掉 sanity**（榴莲 sanityvalue 也是负的，sanityabsorption=1）。这是设计上的取舍：植物不流血，但精神可以受打击。

#### 5.3 `SetCanEat<XXX>`：扩展 Diet 的快捷方式

```85:119:scripts/components/eater.lua
function Eater:SetCanEatHorrible()
    table.insert(self.preferseating, FOODTYPE.HORRIBLE)
    table.insert(self.caneat, FOODTYPE.HORRIBLE)
    self.inst:AddTag(FOODTYPE.HORRIBLE.."_eater")
end

function Eater:SetCanEatGears()
    table.insert(self.preferseating, FOODTYPE.GEARS)
    table.insert(self.caneat, FOODTYPE.GEARS)
    self.inst:AddTag(FOODTYPE.GEARS.."_eater")
end

function Eater:SetCanEatNitre(can_eat)
    local tag = FOODTYPE.NITRE .. "_eater"
    local hastag = self.inst:HasTag(tag)
    if can_eat then
        if not hastag then
            table.insert(self.preferseating, FOODTYPE.NITRE)
            table.insert(self.caneat, FOODTYPE.NITRE)
            self.inst:AddTag(tag)
        end
    else
        if hastag then
            table.removearrayvalue(self.preferseating, FOODTYPE.NITRE)
            table.removearrayvalue(self.caneat, FOODTYPE.NITRE)
            self.inst:RemoveTag(tag)
        end
    end
end

function Eater:SetCanEatRaw()
    table.insert(self.preferseating, FOODTYPE.RAW)
    table.insert(self.caneat, FOODTYPE.RAW)
    self.inst:AddTag(FOODTYPE.RAW.."_eater")
end
```

【函数解析】这是四个"在原有 Diet 上追加一个 FOODTYPE"的便捷方法：

| 方法 | 作用 | 是否带参数 | 典型用户 |
| --- | --- | --- | --- |
| `SetCanEatHorrible()` | 追加 `HORRIBLE` | 不带参数（无 toggle） | 蜘蛛、狗、隐士螃蟹 |
| `SetCanEatGears()` | 追加 `GEARS` | 不带参数 | WX-78 |
| `SetCanEatNitre(can_eat)` | 追加 / 移除 `NITRE` | 带 bool | Acidbats 翻转后 |
| `SetCanEatRaw()` | 追加 `RAW` | 不带参数 | 兔人、隐士螃蟹、跳跳蛇 perd |

【进阶】注意**只有 `SetCanEatNitre` 是双向 toggle**，其他三个**只能加不能减**。如果你需要"先加再移除"，得自己 `table.removearrayvalue(self.preferseating, FOODTYPE.HORRIBLE)` 并 `RemoveTag`。

【老手】源码里 `Eater:SetCanEatHorrible()` 不接受参数，但你在 `scripts/prefabs/hats.lua:2360` 会看到 `inst.components.eater:SetCanEatHorrible(true)` 这种写法——Klei 自己也写错过。Lua 允许多余参数被丢弃，所以**不报错**，但属于源码内的"潜在历史 bug 痕迹"。新代码请按定义无参调用：`eater:SetCanEatHorrible()`。

【老手】注意一个 mod 大坑：如果 `caneat` 和 `preferseating` 是同一个引用（用 `SetDiet({...}, nil)` 设置），调用 `SetCanEatHorrible` 之后会**同时插两次** `HORRIBLE`。表里有重复元素时 `TestFood` 还是只比一次，无功能影响，但**清空 / 序列化时会重复**。最佳实践：显式 `SetDiet(caneatT, preferseatingT)` 用两个独立表。

#### 5.4 `SetPrefersEatingTag(tag)`：Warly 的"只吃料理"

```121:127:scripts/components/eater.lua
function Eater:SetPrefersEatingTag(tag)
    if self.preferseatingtags == nil then
        self.preferseatingtags = { tag }
    else
        table.insert(self.preferseatingtags, tag)
    end
end
```

【函数解析】

- **作用**：注册一个或多个偏好标签。**只有食物带其中任意一个标签**，`PrefersToEat` 才会进入 `TestFood` 的下一步判定。
- **参数**：
  - `tag`（string）：食物侧应有的标签名（如 `"preparedfood"`、`"pre-preparedfood"`）。
- **返回值**：无。
- **实现流程**：第一次调用时初始化 `preferseatingtags = {tag}`，之后 `table.insert` 追加。

【进阶】Warly 的设置就是经典：

```35:38:scripts/prefabs/warly.lua
    if inst.components.eater ~= nil then
        inst.components.eater:SetPrefersEatingTag("preparedfood")
        inst.components.eater:SetPrefersEatingTag("pre-preparedfood")
    end
```

含义：

- Warly 的 `caneat` 还是 OMNI（行动菜单上"吃"任何食物可点击）；
- 但 `preferseatingtags = {"preparedfood", "pre-preparedfood"}`，意思是"只接受已经烹饪 / 半成品菜"；
- 玩家点"吃 一个生胡萝卜"——`PrefersToEat` 看见胡萝卜没有 `preparedfood` 标签，返回 false，Warly 不吃。

【老手】看 `PrefersToEat` 的源码细节：

```336:350:scripts/components/eater.lua
function Eater:PrefersToEat(food)
    if food.prefab == "winter_food4" and self.inst:HasTag("player") then
        --V2C: fruitcake hack. see how long this code stays untouched - _-"
        return false
    elseif self.nospoiledfood and (food.components.perishable and food.components.perishable:IsSpoiled()) then
        return false
    elseif self.preferseatingtags ~= nil then
        --V2C: now it has the warly hack for only eating prepared foods ;-D
        local preferred = food:HasAnyTag(self.preferseatingtags)
        if not preferred then
            return false
        end
    end
    return self:TestFood(food, self.preferseating)
end
```

`preferseatingtags` 是**首要门槛**，先通过它，再走 `TestFood(food, preferseating)`。这是个"AND"逻辑：标签必须有 **且** FOODTYPE 必须在 preferseating 里。

【老手】源码里两个"V2C 注释"（V2C 是 Klei 程序员 Vincent）是 Klei 自己留下的 hack 标记。`winter_food4`（水果蛋糕）的 hack 是"任何玩家都拒绝水果蛋糕"，但 NPC 还能吃（自动留给给坐骑 / 宠物喂）。

#### 5.5 `SetStrongStomach` / `SetCanEatRawMeat` / `SetIgnoresSpoilage`

```129:157:scripts/components/eater.lua
function Eater:SetStrongStomach(is_strong)
    if is_strong then
        self.inst:AddTag("strongstomach")
        self.strongstomach = true
    elseif self.strongstomach then
        self.inst:RemoveTag("strongstomach")
        self.strongstomach = false
    end
end

function Eater:SetCanEatRawMeat(can_eat)
    if can_eat then
        self.inst:AddTag("eatsrawmeat")
        self.eatsrawmeat = true
    elseif self.eatsrawmeat then
        self.inst:RemoveTag("eatsrawmeat")
        self.eatsrawmeat = false
    end
end

function Eater:SetIgnoresSpoilage(ignores)
    if ignores then
        self.inst:AddTag("ignoresspoilage")
        self.ignoresspoilage = true
    elseif self.ignoresspoilage then
        self.inst:RemoveTag("ignoresspoilage")
        self.ignoresspoilage = false
    end
end
```

【函数解析】这三个方法都是 **bool 开关 + 标签同步**：

| 方法 | 作用 | 标签 | 影响哪些判定 |
| --- | --- | --- | --- |
| `SetStrongStomach(b)` | 强胃 | `strongstomach` | `DoFoodEffects` 检查；吃 `monstermeat` 标签的食物时不应用 health/sanity 负面 |
| `SetCanEatRawMeat(b)` | 吃生肉 | `eatsrawmeat` | 同上，但检查 `rawmeat` 标签 |
| `SetIgnoresSpoilage(b)` | 无视腐烂 | `ignoresspoilage` | `Edible:GetHealth/Hunger/Sanity` 直接跳过 stale/spoiled 倍率 |

【进阶】注意 `SetStrongStomach` 是**bool**，可以 toggle off；但**只有当前为 true 才会执行 elseif 分支**。如果当前是 false 你又传 false，什么也不会发生（这是为了避免重复 RemoveTag 警告）。

【老手】`strongstomach` / `eatsrawmeat` / `foodaffinity` 三个抵消负面的机制共同实现 `DoFoodEffects`：

```202:206:scripts/components/eater.lua
function Eater:DoFoodEffects(food)
    return not ((self.strongstomach and food:HasTag("monstermeat")) or
                (self.eatsrawmeat and food:HasTag("rawmeat")) or
                (self.inst.components.foodaffinity and self.inst.components.foodaffinity:HasPrefabAffinity(food)))
end
```

**`DoFoodEffects(food)` 返回 false 时**，`Eat` 函数会跳过负值食物的健康 / 精神扣减：

```243:267:scripts/components/eater.lua
        if self.inst.components.health ~= nil and
            (food.components.edible.healthvalue >= 0 or self:DoFoodEffects(food)) then
            ...
        end
        ...
        if self.inst.components.sanity ~= nil and
            (food.components.edible.sanityvalue >= 0 or self:DoFoodEffects(food)) then
            ...
        end
```

也就是说：

- `healthvalue >= 0` —— 正常吸收（要回血就回血）；
- `DoFoodEffects(food) == true` —— 正常吸收（包括扣血）；
- `DoFoodEffects(food) == false` —— **跳过 health_delta 计算**，等于 0 不扣血。

【老手】**反推**：你只需要给食物加 `monstermeat` 标签 + 给吃方加 `strongstomach`，就能让"怪物肉对它无害"。但**饱食仍然吸收**（hunger 没有 DoFoodEffects 检查），这是设计取舍——强胃 ≠ 不吃。

#### 5.6 `SetRefusesSpoiledFood` / `SetSpoiledProcessor`

```159:186:scripts/components/eater.lua
function Eater:SetRefusesSpoiledFood(refuses)
    if refuses then
        self.inst:AddTag("nospoiledfood")
        self.nospoiledfood = true
    elseif self.nospoiledfood then
        self.inst:RemoveTag("nospoiledfood")
        self.nospoiledfood = false
    end
end

function Eater:SetSpoiledProcessor(processor, allspoiled)
    if processor then
        self.inst:AddTag("spoiledprocessor")
        self.spoiledprocessor = true
        if allspoiled then -- perishables in spoiled state, instead of just rot.
            self.inst:AddTag("allspoiledprocessor")
            self.allspoiledprocessor = true
        else
            self.inst:RemoveTag("allspoiledprocessor")
            self.allspoiledprocessor = false
        end
    elseif self.spoiledprocessor then
        self.inst:RemoveTag("spoiledprocessor")
        self.spoiledprocessor = false
        self.inst:RemoveTag("allspoiledprocessor")
        self.allspoiledprocessor = false
    end
end

function Eater:IsSpoiledProcessor()
    return self.spoiledprocessor
end

function Eater:CanProcessSpoiledItem(food)
    return self:IsSpoiledProcessor() and
        (food.components.edible:IsSpoiledFood() or
        (self.allspoiledprocessor and food.components.perishable and food.components.perishable:IsSpoiled()))
end
```

【函数解析】两个反向接口：

| 方法 | 作用 |
| --- | --- |
| `SetRefusesSpoiledFood(b)` | 该实体**拒绝**腐烂食物：`PrefersToEat` 直接返 false（看到腐烂食物完全不吃） |
| `SetSpoiledProcessor(p, a)` | 该实体**能净化**腐烂食物：吃下去时 health/hunger/sanity 直接取 `max(0, 原值)`，负值变 0 |

`SetSpoiledProcessor` 的第二参数 `allspoiled`：

- false / nil：只能处理 `spoiledfood`（即 `spoiled_food` 那种已腐烂物品 prefab）；
- true：连带处理 `perishable:IsSpoiled()` 状态的所有食物（陈旧到腐烂的胡萝卜、玉米等）。

【进阶】`SetSpoiledProcessor` 在官方 prefab 中**没有被调用**（搜索过整个 scripts 文件夹）—— 它是 Klei 为未来角色或 mod 预留的接口。如果你做一个"垃圾桶大叔"或"腐尸生物"角色，可以用：

```lua
inst.components.eater:SetSpoiledProcessor(true, true)
```

这之后该角色吃腐烂胡萝卜 / 腐烂肉，**不会负 health，不会掉 sanity，但仍按 spoiled_health/spoiled_hunger 倍率**（默认 0/0.5）应用衰减。

【老手】注意 `CanProcessSpoiledItem` 在上一节 `Edible:GetHealth/Hunger` 里被调用：

```scripts/components/edible.lua (上节 6.1 / 6.2)
    if eater ~= nil and eater.components.eater ~= nil and eater.components.eater:CanProcessSpoiledItem(self.inst) then
        healthvalue = math.max(0, healthvalue)
    elseif not ignore_spoilage and self.inst.components.perishable ~= nil then
        ...
    end
```

注意 `if / elseif` 结构 —— 一旦 `CanProcessSpoiledItem` 返回 true，**整个腐烂衰减分支被跳过**。

【老手】**`spoiledprocessor` 还会影响 ACTIONS.EAT 的菜单文案**：

```748:758:scripts/actions.lua
ACTIONS.EAT.strfn = function(act)
    if act.invobject ~= nil then
        return (act.doer ~= nil and
                (act.doer:HasTag("spoiledprocessor") and act.invobject:HasTag("spoiledfood"))
                or (act.doer:HasTag("allspoiledprocessor") and act.invobject:HasTag("spoiled"))) and "PROCESS"
            or act.invobject:HasTag("fooddrink") and "DRINK"
            or nil
    end
    return nil
end
```

如果玩家是 spoiledprocessor，吃 spoiled_food 时菜单会变成"处理"（PROCESS）而不是"吃"（EAT）。**这是非常少见的特殊菜单 trick**——一般 mod 用不上，但 STRINGS 里得对应配 `ACTIONS.EAT.PROCESS = "处理"` 才不报错。

#### 5.7 `SetOnEatFn(fn)` 和 `oneatfn`

```198:200:scripts/components/eater.lua
function Eater:SetOnEatFn(fn)
    self.oneatfn = fn
end
```

【函数解析】

- **作用**：注册"吃下后回调"。
- **参数**：
  - `fn`（function）：签名 `fn(inst, food, feeder)`，其中 inst = 吃方实体，food = 食物，feeder = 投喂者（自己吃自己时 feeder == inst）。
- **返回值**：无。

【进阶】这个回调在 `Eater:Eat` 中**晚于 `oneat` 事件、早于 `Edible:OnEaten`**：

```302:307:scripts/components/eater.lua
        self.inst:PushEvent("oneat", { food = food, feeder = feeder })
        if self.oneatfn ~= nil then
            self.oneatfn(self.inst, food, feeder)
        end

        food.components.edible:OnEaten(self.inst)
```

【老手】实战示例：

**Woby 吃 treat 推事件**（让主人开心）：

```657:661:scripts/prefabs/wobysmall.lua
local function OnEat(inst, food, feeder)
	if food:HasTag("pet_treat") then
		feeder:PushEvent("treatwoby", inst)
	end
end
```

**眼面具帽 eyemask 吃食物时修甲**（健康 + 饱食的绝对值之和转换为护甲恢复）：

```2331:2342:scripts/prefabs/hats.lua
	local function eyemask_oneatfn(inst, food)
		local health = math.abs(food.components.edible:GetHealth(inst)) * inst.components.eater.healthabsorption
		local hunger = math.abs(food.components.edible:GetHunger(inst)) * inst.components.eater.hungerabsorption
		inst.components.armor:Repair(health + hunger)

		if not inst.inlimbo then
			inst.AnimState:PlayAnimation("eat")
			inst.AnimState:PushAnimation("anim", true)

			inst.SoundEmitter:PlaySound("terraria1/eyemask/eat")
		end
	end
```

【老手】两个易踩坑点：

1. **`oneatfn` 是同步执行的**。如果你在里面 `inst:Remove()` 把吃方杀死，后面 `Edible:OnEaten` / `HandleEatRemove` 会跑在 invalid entity 上 ——`HandleEatRemove` 内部有 `inst:IsValid()` 检查保护食物，但 eater 自身没保护。**杀人放在事件 listener 或 DoTaskInTime 里**。
2. **`feeder` 可能为 nil 吗？** 看源码：

```230:231:scripts/components/eater.lua
function Eater:Eat(food, feeder)
    feeder = feeder or self.inst
```

不会，永远至少有自己。但**注意当 ACTIONS.EAT 通过 invobject 触发时，feeder == doer == eater 是同一个**；当 ACTIONS.FEED / FEEDPLAYER 触发时，feeder ≠ eater。

#### 5.8 `Eater:GetEdibleTags()` 与 `cacheedibletags`

```208:228:scripts/components/eater.lua
function Eater:GetEdibleTags()
    if self.cacheedibletags then
        return self.cacheedibletags
    end

    local tags = {}
    for i, v in ipairs(self.caneat) do
        if type(v) == "table" then
            for i2, v2 in ipairs(v.types) do
                table.insert(tags, "edible_"..v2)
            end
        else
            table.insert(tags, "edible_"..v)
        end
    end

    if self.cacheedibletags ~= false then
        self.cacheedibletags = tags
    end
    return tags
end
```

【函数解析】

- **作用**：返回吃方"能找的所有食物标签"集合（用于 `FindEntities` 等 AI 查询）。
- **参数**：无。
- **返回值**：table（如 `{"edible_MEAT", "edible_VEGGIE", "edible_INSECT", ...}`）。
- **实现流程**：
  1. 缓存命中则直接返回。
  2. 遍历 `caneat`，把每项展平：FOODGROUP 的话取 `.types` 列表全展开，FOODTYPE 字符串直接前缀 `"edible_"`。
  3. 写入缓存（除非 `cacheedibletags == false` 主动禁用）。

【进阶】**何时禁用缓存？** 当你做的是 buff / debuff / 装备引起的动态 Diet 切换（如勋章 mod 那个全食物 buff），就**必须**在 SetDiet 之后让缓存失效：

```lua
inst.components.eater.cacheedibletags = nil  -- 触发下次重建
-- 或
inst.components.eater.cacheedibletags = false -- 永久禁用缓存
```

【老手】注意第 17 行的奇怪三态：

- `nil`：未初始化，下次 `GetEdibleTags()` 时计算且写入缓存。
- `table`：缓存命中，直接返。
- `false`：永久禁用，每次都重新计算。

这是 Klei 用同一字段表示三种状态的紧凑写法。如果你写 mod 不熟悉这点，常见错误是 `cacheedibletags = {}` 想"清空"——会被当成"已缓存空表"，比 nil 还糟糕。

#### 5.9 其他辅助方法

```63:69:scripts/components/eater.lua
function Eater:TimeSinceLastEating()
    return self.lasteattime ~= nil and GetTime() - self.lasteattime or nil
end

function Eater:HasBeen(time)
    return self.lasteattime == nil or self:TimeSinceLastEating() >= time
end
```

- `TimeSinceLastEating()`：从最后一次吃东西到现在经过的秒数，没吃过返回 nil。
- `HasBeen(time)`：自从上次吃，是否已经过去了至少 `time` 秒。**没吃过也返 true**（这点反直觉，但符合"超久没吃了"的语义）。

【老手】SG (stategraph) 里有时会**手动写 `lasteattime`**，例如鱿鱼咀嚼掉食物：

```831:831:scripts/stategraphs/SGsquid.lua
inst.components.eater.lasteattime = GetTime()
```

或者鳄鱼 cookiecutter 钻取食物：

```625:625:scripts/stategraphs/SGcookiecutter.lua
inst.components.eater.lasteattime = GetTime()
```

这是当生物用"非 EAT 动作"消耗食物时，**手动同步进食时间**给行为树用。意思是"我刚啃过，先别马上又找食物"。Mod 里如果你做"特殊吃法"，记得做这种同步。

#### 5.10 `OnSave` / `OnLoad`

```71:83:scripts/components/eater.lua
function Eater:OnSave()
    return self.lasteattime ~= nil
        and {
                time_since_eat = self:TimeSinceLastEating(),
            }
        or nil
end

function Eater:OnLoad(data)
    if data.time_since_eat then
        self.lasteattime = GetTime() - data.time_since_eat
    end
end
```

【老手】**只存 `lasteattime`**，而且是用"距离当前的相对时间"存（避免重新加载游戏后绝对时间错位）。其他所有字段（`caneat`、`preferseating`、`strongstomach`、各种 absorption）都**不存盘**，它们的值由 prefab `master_postinit` 重新设置。

**坑**：如果你的 mod 做"动态 Diet"（buff 修改了 caneat），存盘 → 重新加载后，**Diet 恢复成 prefab 默认值**。要让 Diet 持久化，得自己在 entity 的 OnSave/OnLoad 里序列化。

---

### 六、核心吃食流程：`Eater:Eat` 全链路

这是本组件最重要的函数，**所有"玩家 / 生物吃东西"的路径都经过它**。

```230:318:scripts/components/eater.lua
function Eater:Eat(food, feeder)
    feeder = feeder or self.inst
    -- This used to be CanEat. The reason for two checks is to that special diet characters (e.g.
    -- wigfrid) can TRY to eat all foods (they get the actions for it) but upon actually put it in
    -- their mouth, they bail and "spit it out" so to speak.
    if self:PrefersToEat(food) then
        local stack_mult = self.eatwholestack and food.components.edible:GetStackMultiplier() or 1
        local base_mult = self.inst.components.foodmemory ~= nil and self.inst.components.foodmemory:GetFoodMultiplier(food.prefab) or 1

		local health_delta = 0
		local hunger_delta = 0
		local sanity_delta = 0

        if self.inst.components.health ~= nil and
            (food.components.edible.healthvalue >= 0 or self:DoFoodEffects(food)) then
            health_delta = food.components.edible:GetHealth(self.inst) * base_mult * self.healthabsorption
        end

        if self.inst.components.hunger ~= nil then
            hunger_delta = food.components.edible:GetHunger(self.inst) * base_mult * self.hungerabsorption
        end

        if self.inst.components.sanity ~= nil and
            (food.components.edible.sanityvalue >= 0 or self:DoFoodEffects(food)) then
            sanity_delta = food.components.edible:GetSanity(self.inst) * base_mult * self.sanityabsorption
        end

		if self.custom_stats_mod_fn ~= nil then
			health_delta, hunger_delta, sanity_delta = self.custom_stats_mod_fn(self.inst, health_delta, hunger_delta, sanity_delta, food, feeder)
		end

        if health_delta ~= 0 then
            self.inst.components.health:DoDelta(health_delta * stack_mult, nil, food.prefab)
        end
        if hunger_delta ~= 0 then
            self.inst.components.hunger:DoDelta(hunger_delta * stack_mult)
        end
        if sanity_delta ~= 0 then
            self.inst.components.sanity:DoDelta(sanity_delta * stack_mult)
        end

		if feeder ~= self.inst then
			if self.inst.components.inventoryitem then
				local owner = self.inst.components.inventoryitem:GetGrandOwner()
				if owner and (owner == feeder or (owner.components.container and owner.components.container:IsOpenedBy(feeder))) then
					feeder:PushEvent("feedincontainer")
				end
			end
			if self.inst.components.rideable and feeder == self.inst.components.rideable:GetRider() then
				feeder:PushEvent("feedmount", { food = food, eater = self.inst })
			end
		end

        self.inst:PushEvent("oneat", { food = food, feeder = feeder })
        if self.oneatfn ~= nil then
            self.oneatfn(self.inst, food, feeder)
        end

        food.components.edible:OnEaten(self.inst)
        food.components.edible:HandleEatRemove(self.eatwholestack)

        self.lasteattime = GetTime()

        if self.inst.components.foodmemory ~= nil and not food:HasTag("potion") then
            self.inst.components.foodmemory:RememberFood(food.prefab)
        end

        return true
    end
end
```

【函数解析】

- **作用**：让 `self.inst` 吃下 `food` 这份食物，处理所有属性变更、事件、移除等。
- **参数**：
  - `food`（Entity，必填）：要吃的食物，必须挂 `Edible` 组件。
  - `feeder`（Entity\|nil，可选，默认 `self.inst`）：投喂者；自己吃自己时与 self.inst 相同；他人喂食时 feeder ≠ self.inst。
- **返回值**：
  - `true`：成功吃下并应用了所有效果。
  - `nil`（即 false 等价）：`PrefersToEat` 不通过，未吃。
- **实现流程**（13 步）：

| 步 | 动作 | 解读 |
| --- | --- | --- |
| 1 | `feeder = feeder or self.inst` | 默认自己投喂 |
| 2 | `PrefersToEat(food)` 守门 | 偏好不过则 return（不打 oneat 也不消耗食物） |
| 3 | 算 `stack_mult` | 如果 `eatwholestack`，整堆倍数；否则 1 |
| 4 | 算 `base_mult` | `foodmemory` 的吃腻倍率（无 foodmemory = 1） |
| 5 | 算 `health_delta` | `Edible:GetHealth(inst) * base_mult * healthabsorption`；若是负值且 `DoFoodEffects=false`，强制为 0 |
| 6 | 算 `hunger_delta` | 不需要 DoFoodEffects 检查，永远应用 |
| 7 | 算 `sanity_delta` | 同 health，负值且 `DoFoodEffects=false` 强制为 0 |
| 8 | `custom_stats_mod_fn` | 最后一层 mod 修正，可以把 delta 改成任何值 |
| 9 | DoDelta 三件套 | 真正应用到 health/hunger/sanity 组件，乘 stack_mult |
| 10 | 推容器 / 坐骑事件 | feeder ≠ self.inst 时推 `feedincontainer` / `feedmount` |
| 11 | 推 `oneat` 事件 + 调 `oneatfn` | 这是 Eater 侧的"吃完"通知 |
| 12 | 调 `Edible:OnEaten` + `HandleEatRemove` | 这是 Edible 侧的"被吃"通知 + 移除/扣堆 |
| 13 | 记 `lasteattime` + foodmemory:RememberFood | 持久化进食记录 |

【新手】**总结一图**：

```
[食物 food]  ←  Edible.GetHealth / GetHunger / GetSanity
       │
       ▼
[Eater.Eat]
       │
       ├── PrefersToEat?  →  否：return nil（食物没动）
       │
       ├── base_mult (foodmemory) × absorption × stack_mult
       │
       ├── custom_stats_mod_fn?
       │
       ├── health/hunger/sanity 真正变化（DoDelta）
       │
       ├── PushEvent "oneat" + oneatfn
       │
       ├── Edible.OnEaten（push "oneaten" + 温度 + 音效）
       │
       └── Edible.HandleEatRemove（消耗 / Remove）
```

【进阶】注意倍率应用的**顺序**：

1. `Edible:GetHealth(eater)` 内部已经乘了**腐烂倍率 + 香料倍率**。
2. `base_mult` 是 foodmemory 倍率，乘在外面。
3. `healthabsorption` 是 Eater 自身倍率，又乘在外面。
4. `custom_stats_mod_fn` 是最后一刀，可改可不改。
5. **最后 `* stack_mult` 才传给 `DoDelta`**。

也就是说，全部倍率合起来：

```
最终健康变化 = stack_mult × custom_stats_mod_fn(base_mult × absorption × spice_mult × spoil_mult × healthvalue)
```

【老手】**foodmemory 不进 stack_mult**：这是反直觉的细节。`foodmemory:GetFoodMultiplier` 是 base_mult，吃整堆时 stack_mult 是堆数（如吃 20 个）但 base_mult 仍只算这一种 prefab 的吃腻系数。所以**吃腻 + 整堆吃**不会"吃腻系数乘 20 次再叠"，仍只乘 1 次。这是合理的，否则同一种食物堆叠后吃腻效果会爆。

【老手】注意 `food.components.edible.healthvalue >= 0 or self:DoFoodEffects(food)` 的逻辑：

- 食物本身正向（healthvalue ≥ 0）：直接吸收，无需 DoFoodEffects。
- 食物本身负向（healthvalue < 0）：必须 DoFoodEffects==true 才吸收负值；如果 false（强胃、生肉党、亲和），跳过 → health_delta = 0。

**这意味着**：负值食物 + 强胃 = 不扣血，**但也不会回血**（healthvalue 是负的，max(0, neg) 不会发生，因为只是被 if 跳过）。如果你想让"强胃吃怪物肉还能回血"，得在 `custom_stats_mod_fn` 里手动修正。

---

### 七、`TestFood` / `PrefersToEat` / `CanEat`：三个匹配函数

```320:354:scripts/components/eater.lua
function Eater:TestFood(food, testvalues)
    if food ~= nil and food.components.edible ~= nil then
        for i, v in ipairs(testvalues) do
            if type(v) == "table" then
                for i2, v2 in ipairs(v.types) do
                    if food:HasTag("edible_"..v2) then
                        return true
                    end
                end
            elseif food:HasTag("edible_"..v) then
                return true
            end
        end
    end
end

function Eater:PrefersToEat(food)
    if food.prefab == "winter_food4" and self.inst:HasTag("player") then
        --V2C: fruitcake hack. see how long this code stays untouched - _-"
        return false
    elseif self.nospoiledfood and (food.components.perishable and food.components.perishable:IsSpoiled()) then
        return false
    elseif self.preferseatingtags ~= nil then
        --V2C: now it has the warly hack for only eating prepared foods ;-D
        local preferred = food:HasAnyTag(self.preferseatingtags)
        if not preferred then
            return false
        end
    end
    return self:TestFood(food, self.preferseating)
end

function Eater:CanEat(food)
    return self:TestFood(food, self.caneat)
end
```

【函数解析】

- **`TestFood(food, testvalues)`**：通用匹配函数。
  - 要求 food 必须挂 Edible 组件（保护，避免对非食物 panic）；
  - 遍历 testvalues 列表，每项可以是 FOODTYPE 字符串或 FOODGROUP 表；
  - **匹配方式是查食物身上的 `edible_<X>` 标签**，不是直接读 `edible.foodtype` 字段。
- **`PrefersToEat(food)`**：在 TestFood 之前加三道前置门槛：
  1. 水果蛋糕 hack；
  2. 拒绝腐烂食物；
  3. preferseatingtags 标签门槛（Warly 用）。
- **`CanEat(food)`**：纯 TestFood + caneat 列表，不带任何前置。

【进阶】**典型应用区别**：

| 场景 | 用 `CanEat` 还是 `PrefersToEat`？ |
| --- | --- |
| ACTIONS.EAT 是否可选 | `CanEat`（资格） |
| brain 找食物的 FindEntities | `GetEdibleTags`（基于 caneat） |
| 拖食物进玩家嘴上能否高亮 | `CanEat` |
| 真正进入嘴里那一刻是否吐出 | `PrefersToEat`（偏好） |
| trader 给主人是否接受这份礼物 | `CanEat` 多用 |

【老手】**注意 `TestFood` 没有 return 兜底**：如果 testvalues 是空表，循环里没 match 到 true，函数会**返回 nil**（Lua 默认无 return）。在 if 里被当 false，等价不能吃。**所以 `caneat = {}` 是合法的——意思是"这家伙啥都不吃"**。但要小心**别赋成 `caneat = nil`**——会触发 `oncaneat(self, nil, ...)`，第二参为 nil 仍走 RemoveTag 但 first if 条件 `caneat ~= nil` 不进，结果旧标签清不掉，**这是写 mod 时的真坑**。

【老手】**Warly 偏好链流程**（举例：玩家点"吃 一份蝴蝶汁"）：

1. ACTION.EAT 触发，进入 `eater:Eat(蝴蝶汁, walter)`；
2. `PrefersToEat(蝴蝶汁)` 开始：
   - `prefab != "winter_food4"` ✓；
   - `nospoiledfood = false` ✓；
   - `preferseatingtags = {"preparedfood", "pre-preparedfood"}`，蝴蝶汁有 "preparedfood" 标签 → `HasAnyTag` 返回 true ✓；
   - 进入 `TestFood(蝴蝶汁, preferseating)`，preferseating = OMNI，蝴蝶汁有 `edible_GENERIC` 标签 ✓；
3. 通过，正常吃。

如果换成"生胡萝卜"：

1. 同样进 PrefersToEat；
2. 蝴蝶汁标签步检测 carrot 没有 "preparedfood"，**直接 return false**，Warly 拒绝。

---

### 八、新手实战：给 mod 角色 / 生物配置 Diet

#### 8.1 普通杂食角色（默认）

```lua
local function master_postinit(inst)
    -- 玩家 prefab 已经自动 AddComponent("eater") 并设 OMNI，无需再写。
    -- 改吸收倍率即可：
    inst.components.eater:SetAbsorptionModifiers(1, 1, 1)  -- 标准（默认就是这样）
end
```

#### 8.2 肉食生物（如 mod 狗）

```lua
local function fn()
    local inst = CreateEntity()
    ...
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("eater")
    inst.components.eater:SetDiet({ FOODTYPE.MEAT }, { FOODTYPE.MEAT })
    inst.components.eater:SetStrongStomach(true)     -- 吃怪物肉无负面
    inst.components.eater:SetCanEatRawMeat(true)      -- 吃生肉无负面

    return inst
end
```

#### 8.3 素食角色（如 mod 植物人）

```lua
local function master_postinit(inst)
    if inst.components.eater ~= nil then
        inst.components.eater:SetDiet(
            { FOODGROUP.VEGETARIAN },               -- 资格 = VEGETARIAN（VEGGIE/SEEDS/GENERIC/GOODIES）
            { FOODGROUP.VEGETARIAN }                -- 偏好同上
        )
        inst.components.eater:SetAbsorptionModifiers(0, 1, 1)  -- 食物不回血
    end
end
```

#### 8.4 wild 杂食 + 强胃 + 整堆吃（参考 bearger）

```569:570:scripts/prefabs/bearger.lua
	inst.components.eater:SetDiet({ FOODGROUP.BEARGER }, { FOODGROUP.BEARGER })
	inst.components.eater.eatwholestack = true
```

【新手】注意 `eatwholestack = true` 是**字段直接赋值**，不需要 setter。它告诉 `Eat` 函数"吃下去整堆消耗、属性按堆数倍叠"。

#### 8.5 挑食角色（参考 Wigfrid）

```167:172:scripts/prefabs/wathgrithr.lua
    if inst.components.eater ~= nil then
        inst.components.eater:SetDiet({ FOODGROUP.OMNI }, { FOODTYPE.MEAT, FOODTYPE.GOODIES })
    end

    inst.components.foodaffinity:AddPrefabAffinity("turkeydinner", TUNING.AFFINITY_15_CALORIES_HUGE )
```

【新手】这种"资格 ≠ 偏好"的写法关键好处：**UI 上玩家以为能吃，行为是吐出来**。配上 `wisecracker` 角色台词，玩家就会自然学会"哦原来这角色不吃菜"。

#### 8.6 万物书 mod 中的入门示例（松鼠 prefab）

```254:255:mods/联机版mod/万物书/scripts/prefabs/11_tbat_animals/03_maple_squirrel.lua
			-- inst.components.eater:SetDiet({ FOODTYPE.SEEDS }, { FOODTYPE.SEEDS })
			inst.components.eater:SetDiet({ FOODTYPE.SEEDS,FOODTYPE.VEGGIE,FOODTYPE.GOODIES })
```

【新手】这里 `SetDiet` 只传一个参数，第二个 nil，于是 `preferseating = caneat`——常见的"看到啥能吃就吃啥"配置。等价于：

```lua
inst.components.eater:SetDiet(
    { FOODTYPE.SEEDS, FOODTYPE.VEGGIE, FOODTYPE.GOODIES },
    { FOODTYPE.SEEDS, FOODTYPE.VEGGIE, FOODTYPE.GOODIES }
)
```

---

### 九、进阶要点

#### 9.1 `custom_stats_mod_fn`：最后一刀

【进阶】这是除了 oneatfn 之外，mod 修改吃食效果的**最大杠杆**。Klei 用它做了 Woby 的"吃 treat 三倍饱食"：

```516:521:scripts/prefabs/wobysmall.lua
local function CustomFoodStatsMod(inst, health_delta, hunger_delta, sanity_delta, food, feeder)
	if food and food.prefab == "woby_treat" and hunger_delta and hunger_delta > 0 then
		hunger_delta = hunger_delta * 3
	end
	return health_delta, hunger_delta, sanity_delta
end
```

```761:762:scripts/prefabs/wobysmall.lua
    inst.components.eater:SetDiet({ FOODTYPE.MONSTER }, { FOODTYPE.MONSTER })
	inst.components.eater.custom_stats_mod_fn = CustomFoodStatsMod
```

【函数解析】`custom_stats_mod_fn` 的签名：

```
fn(inst, health_delta, hunger_delta, sanity_delta, food, feeder)
  → 返回 (health_delta, hunger_delta, sanity_delta)
```

注意必须**返回三元组**，否则后续 if 检查中 nil != 0 会进入 DoDelta 调用 `DoDelta(nil)` 报错。

【老手】实战例子：mod 一个"残血回多血"角色：

```lua
local function CustomStatsModFn(inst, h, hg, s, food, feeder)
    if inst.components.health and inst.components.health:GetPercent() < 0.25 then
        h = h * 2
    end
    return h, hg, s
end

inst.components.eater.custom_stats_mod_fn = CustomStatsModFn
```

【老手】**和 `SetGetHealthFn` 的区别**：

| 接口 | 注入位置 | 看哪些信息 |
| --- | --- | --- |
| `Edible:SetGetHealthFn` | 食物 prefab 上 | 食物自身、eater |
| `Eater.custom_stats_mod_fn` | eater 上 | h/hg/s 增量、food、feeder |

如果你的逻辑是"我（角色）特别吃这种东西"，写 `custom_stats_mod_fn`；如果是"这种食物对所有人都有特殊算法"，写 `SetGetHealthFn`。

#### 9.2 `foodaffinity`（食物亲和）

【进阶】这是一个**独立组件**（`scripts/components/foodaffinity.lua`），玩家在 `player_common.lua:2785` 处也 AddComponent("foodaffinity")。它提供三种亲和：

- `AddPrefabAffinity(prefab, bonus)`：对某 prefab 的整体饱食倍率（上一节 Edible:GetHunger 中查询）；
- `AddFoodtypeAffinity(foodtype, bonus)`：对某 FOODTYPE 的倍率；
- `AddTagAffinity(tag, bonus)`：对带标签的食物倍率。

**实战**：Walter 吃 trailmix 多 ×2.2 饱食：

```1191:1191:scripts/prefabs/walter.lua
    inst.components.foodaffinity:AddPrefabAffinity("trailmix", TUNING.AFFINITY_15_CALORIES_SMALL)
```

Wigfrid 吃火鸡晚餐多倍：

```171:171:scripts/prefabs/wathgrithr.lua
    inst.components.foodaffinity:AddPrefabAffinity("turkeydinner", TUNING.AFFINITY_15_CALORIES_HUGE )
```

【进阶】**foodaffinity 还有一个隐藏威力**：它出现在 `Eater:DoFoodEffects(food)` 的判断里：

```202:206:scripts/components/eater.lua
function Eater:DoFoodEffects(food)
    return not ((self.strongstomach and food:HasTag("monstermeat")) or
                (self.eatsrawmeat and food:HasTag("rawmeat")) or
                (self.inst.components.foodaffinity and self.inst.components.foodaffinity:HasPrefabAffinity(food)))
end
```

也就是说：**只要 eater 对这份 prefab 有亲和，DoFoodEffects 返回 false → 负值食物的扣血、扣 sanity 全部跳过**。

这就是 Wormwood 吃榴莲（durian）不掉血的真正机制——`foodaffinity:AddPrefabAffinity("durian", 1)`，亲和系数 1 等于"不变倍率但触发 affinity 检测"，避开 DoFoodEffects 的负面。

#### 9.3 `foodmemory`（吃腻）

【进阶】Warly 专属组件（其他玩家不挂），通过 `master_postinit` 注册：

```40:42:scripts/prefabs/warly.lua
    inst:AddComponent("foodmemory")
    inst.components.foodmemory:SetDuration(TUNING.WARLY_SAME_OLD_COOLDOWN)
    inst.components.foodmemory:SetMultipliers(TUNING.WARLY_SAME_OLD_MULTIPLIERS)
```

【老手】`Eater:Eat` 在第 4 步查询 `foodmemory:GetFoodMultiplier(food.prefab)`，得到 base_mult。这个倍率随"最近吃过同 prefab 次数"递减（典型：5 次后衰减到 0.1，吃饱食几乎为零）。然后吃完后调用 `RememberFood(food.prefab)`，下次记录 + 1。

【老手】**`Eat` 中 `not food:HasTag("potion")` 守门**——药水（药水也用 Edible 实现）不进入吃腻系统。如果你 mod 一个"药水"prefab，要让 Warly 喝了不计入吃腻，**给它打 `potion` 标签**：

```lua
inst:AddTag("potion")
```

#### 9.4 `eatwholestack`：整堆吞咽

【进阶】两个关键变化：

1. 整堆消耗：`Edible:HandleEatRemove(eatwholestack)` 内部 `inst:Remove()` 而非 `stackable:Get():Remove()`。
2. 属性按堆数倍乘：`stack_mult = self.eatwholestack and food.components.edible:GetStackMultiplier() or 1`。

`GetStackMultiplier()` 的实现（见 edible.lua 上节，本节略）一般返回 `stackable.stacksize`，但可以被 `SetOverrideStackMultiplierFn` 覆盖。

【老手】bearger 配置 `eatwholestack = true` 意味着它一口气吃完一堆 hambat、吃完一堆肉，所以**狂暴期 bearger 的伤害和饱食都按堆叠数倍翻**——这就是为什么不能堆叠物品扔在 bearger 路径上。

【老手】moose / mossling 也设了 `eatwholestack = true`，但 mod 里的"小型生物"不要轻易设——会导致 player 喂食一次清空一堆，行动菜单上看不出来。

#### 9.5 `lasteattime` 手动同步

【老手】SG / brain / 自定义动作有时绕过 `Eater:Eat`（比如鱿鱼咀嚼掉食物的状态机），这时**记得手动同步**：

```831:831:scripts/stategraphs/SGsquid.lua
inst.components.eater.lasteattime = GetTime()
```

否则行为树查询 `eater:HasBeen(60)` 时永远 true，AI 会"刚吃就找食物"。

---

### 十、老手深挖

#### 10.1 喂食的复杂事件链

【老手】当 `feeder ≠ self.inst`（即他人喂食），`Eat` 会**额外**推送 0-2 个事件：

```290:300:scripts/components/eater.lua
		if feeder ~= self.inst then
			if self.inst.components.inventoryitem then
				local owner = self.inst.components.inventoryitem:GetGrandOwner()
				if owner and (owner == feeder or (owner.components.container and owner.components.container:IsOpenedBy(feeder))) then
					feeder:PushEvent("feedincontainer")
				end
			end
			if self.inst.components.rideable and feeder == self.inst.components.rideable:GetRider() then
				feeder:PushEvent("feedmount", { food = food, eater = self.inst })
			end
		end
```

| 事件 | 推送条件 | 推送对象 | 数据 |
| --- | --- | --- | --- |
| `feedincontainer` | eater 有 inventoryitem，grandowner == feeder（或 feeder 打开了 grandowner 的容器） | feeder | 无 |
| `feedmount` | eater 是 rideable，feeder 是当前 rider | feeder | `{food, eater}` |

【老手】**触发场景举例**：

- Wormwood 把 berry 喂给背包里的 mandrake → mandrake 有 inventoryitem，grandowner 是 Wormwood，feeder 也是 Wormwood → 推 `feedincontainer`。
- 玩家给坐在身上的 beefalo 喂干草 → beefalo 有 rideable + rider 是玩家 → 推 `feedmount`。

这两个事件是各种坐骑 / 盆栽 mod 的关键 hook 点。如果你的 mod 角色要做"喂坐骑加亲密度"，监听 feeder 的 `feedmount`，data.eater 就是坐骑、data.food 就是吃下的。

#### 10.2 喂食意图判定：`IsTryingToFeedMe`

```356:369:scripts/components/eater.lua
function Eater:IsTryingToFeedMe(inst)
	local target
    local act = inst:GetBufferedAction()
	if act then
		target = act.target
		act = act.action
	elseif inst.components.playercontroller then
		act, target = inst.components.playercontroller:GetRemoteInteraction()
	end
	return target == self.inst
		and (	act == ACTIONS.FEED or
				act == ACTIONS.FEEDPLAYER
			)
end
```

【函数解析】

- **作用**：判断 `inst` 是不是正在尝试喂 self.inst 食物。
- **参数**：
  - `inst`（Entity）：可能是潜在喂食者。
- **返回值**：
  - bool：是否正在 buffer 一个 FEED / FEEDPLAYER 动作，且目标是自己。
- **实现流程**：
  1. 先取 inst 的当前 buffered action；
  2. 取不到的话从 playercontroller 取 remote interaction（网络同步的远端动作）；
  3. 判断 target == self.inst 且 act 是 FEED 或 FEEDPLAYER。

【老手】**用法**：trader 接受礼物前先判断"是不是真的在喂我"，避免接到误丢的食物。生物 brain 的"接受喂食"分支会用它判断。Mod 写"按住喂"功能时也用这个。

#### 10.3 `OnRemoveFromEntity` 的清理

```48:50:scripts/components/eater.lua
function Eater:OnRemoveFromEntity()
    clearcaneat(self, self.caneat)
end
```

【老手】Eater 被移除时只清 `<X>_eater` 标签，**不清** `strongstomach` / `ignoresspoilage` / `eatsrawmeat` / `nospoiledfood` / `spoiledprocessor` 等标签——它们由各 Setter 在 set false 时清。

这是个**疑似遗漏**：如果你 mod 一个"动态可移除 Eater 组件"的物品，Eater 走 OnRemoveFromEntity 后这些遗留标签会卡在 entity 上。**正确做法是在自己 mod 里手动 RemoveComponent 之前先 SetStrongStomach(false)、SetIgnoresSpoilage(false)** 等等。或者把 mod 改成不移除 Eater，只改 caneat。

#### 10.4 喂"食物到坐骑"vs"食物到玩家自己"的行为差

【老手】很有趣的细节：玩家不能像 Wigfrid 那样吃 monstermeat（preferseating 排除），但可以**喂给** beefalo 坐骑。这是因为 `eater:Eat` 是被 ACTIONS.FEED / FEEDPLAYER 路由触发，**target 是坐骑而非玩家**，调用的是 `beefalo.components.eater:Eat(monstermeat, wigfrid_player)`。坐骑的 preferseating 包含 VEGGIE/ROUGHAGE 但不包含 MEAT，**所以坐骑也拒绝**，但拒绝的是坐骑、不是玩家。

`feedmount` 事件在玩家身上推送，玩家的 sg 走 `feed_mount` 状态做投喂动画，但不会真的"喂下去"——因为坐骑的 eater 内部 PrefersToEat 没过。这是分两层的安全设计。

#### 10.5 `cacheedibletags` 的失效时机

【老手】重申：**改 SetDiet 之后**或**改 SetCanEatXXX 之后**，缓存自动失效吗？**答案：不会**。

源码：

```46:46:scripts/components/eater.lua
{
    caneat = oncaneat,
})
```

`caneat` 的 setter 只清 / 加标签，**没有清 cacheedibletags**。所以你赋了新 caneat，下一次 `GetEdibleTags()` 还是返回旧缓存。

**正确做法**：所有 SetDiet / SetCanEatHorrible 等修改 caneat 的操作后，**手动**：

```lua
inst.components.eater.cacheedibletags = nil
```

或者**永久禁用**：

```lua
inst.components.eater.cacheedibletags = false
```

这是 Klei 的设计 bug——以"在 prefab 初始化期间设置一次后不变"为前提。Mod 做动态 buff 时必须自己维护失效。

#### 10.6 玩家身上的 `oneat` 事件用于诸多 mod 拓展

【老手】很多 mod 用 `oneat` 实现：

- 吃肉减 sanity：listen oneat，data.food:HasTag("meat") 时手动扣 sanity；
- 吃饭加 buff：listen oneat，data.food.prefab == "xxx" 时挂 buff；
- 食物激活技能：吃下特定食物触发特殊事件。

**示例**（mod 做"吃完红椒进入燃烧 buff"）：

```lua
inst:ListenForEvent("oneat", function(inst, data)
    if data.food and data.food.prefab == "pepper" then
        inst:AddDebuff("burning_buff", "spicebuff_burning")
    end
end)
```

【老手】注意 `oneat` 推送时机：**所有 delta 已经应用、food 还在内存、食物还没被 Remove**。所以你可以读 food.components.edible、读 food.prefab，但不能依赖 food 在事件结束后还存在（HandleEatRemove 会跟在 oneat 后面立刻执行）。

---

### 十一、本节涉及的标签与事件（速查）

#### 标签清单（Eater 直接打 / 读的）

| 标签 | 含义 | 谁打 / 谁读 |
| --- | --- | --- |
| `<FOODTYPE>_eater` | 该实体能吃该 FOODTYPE | Eater:setter `oncaneat` 打 / 食物 AI 读 |
| `<FOODGROUP_NAME>_eater` | 该实体能吃该 FOODGROUP | Eater:setter `oncaneat` 打 / 同上 |
| `strongstomach` | 强胃，怪物肉无副作用 | Eater:SetStrongStomach 打 / Eater:DoFoodEffects 读 |
| `eatsrawmeat` | 能吃生肉无副作用 | Eater:SetCanEatRawMeat 打 / Eater:DoFoodEffects 读 |
| `ignoresspoilage` | 无视腐烂衰减 | Eater:SetIgnoresSpoilage 打 / Edible:Get* 读 |
| `nospoiledfood` | 拒绝腐烂食物 | Eater:SetRefusesSpoiledFood 打 / Eater:PrefersToEat 读 |
| `spoiledprocessor` | 能净化 spoiled_food 食物 | Eater:SetSpoiledProcessor 打 / Edible:Get*、ACTIONS.EAT.strfn 读 |
| `allspoiledprocessor` | 增强版 spoiledprocessor | 同上 |

#### Eater 之外但与本节强相关的标签

| 标签 | 含义 | 来源 |
| --- | --- | --- |
| `edible_<FOODTYPE>` | 这份食物属于该 FOODTYPE | Edible setter（见上节） |
| `badfood` | 该食物有副作用 | Edible setter |
| `spoiledfood` | 这份本身就是腐烂物 | Edible:SetForceSpoiledFood |
| `monstermeat` | 该食物是"怪物肉"类 | 各怪物肉 prefab AddTag |
| `rawmeat` | 该食物是生肉 | 生肉 prefab AddTag |
| `preparedfood` | 已完成烹饪的料理 | preparedfoods.lua 的注册过程 |
| `pre-preparedfood` | 半成品料理（Warly 专属） | preparedfoods.lua |
| `pet_treat` | 宠物专属零食 | 各 treat 食物 prefab AddTag |
| `potion` | 药水（吃下不进入 foodmemory） | 各药水 prefab AddTag |

#### 事件清单

**1. `oneat`**

- **含义**：实体（如玩家）吃了一份食物。
- **推送时机**：`Eater:Eat` 中段，**delta 已应用，food 还未被 OnEaten / Remove**。
- **推送数据**：
  - `food`（Entity）：吃下的食物。
  - `feeder`（Entity）：投喂者（自己吃自己时 feeder == self.inst）。
- **推送对象**：吃食物的实体（`eater.inst`）。

**2. `feedincontainer`**

- **含义**：feeder 通过容器（如自己的背包、自己打开的某容器）把食物喂给了 self.inst。
- **推送时机**：`Eater:Eat` 中段，feeder ≠ self.inst 且 eater 是 inventoryitem 且 grandowner 是 feeder（或 feeder 打开了 grandowner 容器）。
- **推送数据**：无。
- **推送对象**：feeder。

**3. `feedmount`**

- **含义**：feeder 喂食给自己的坐骑。
- **推送时机**：`Eater:Eat` 中段，feeder ≠ self.inst 且 eater 是 rideable 且 feeder 是当前 rider。
- **推送数据**：
  - `food`（Entity）：投喂的食物。
  - `eater`（Entity）：坐骑（即 self.inst）。
- **推送对象**：feeder（rider）。

**4. `oneaten`**（属于上节 Edible 内容，列出对照）

- **含义**：食物 prefab 被吃下了。
- **推送时机**：`Edible:OnEaten` 中段，**晚于 Eater 的 `oneat`、早于 HandleEatRemove**。
- **推送数据**：`{ eater = ... }`
- **推送对象**：食物自身。

---

### 十二、常见坑与最佳实践（Checklist）

【新手】

- [ ] 玩家（player）已经自动有 eater，**不要再 AddComponent("eater")**——会触发警告。
- [ ] `AddComponent("eater")` 一定要在 `if not TheWorld.ismastersim` **之后**（服务端组件）。
- [ ] `SetDiet` 只传一个参数时，preferseating 等于 caneat —— 适合普通生物。
- [ ] 想做"挑食角色"，第二参数传**真子集**。

【进阶】

- [ ] `caneat` 改了之后 **`cacheedibletags` 不会自动清**——动态 Diet 时手动 `cacheedibletags = nil`。
- [ ] `SetCanEatHorrible` 等四个函数会同时 push 到 caneat + preferseating，**两个表共引用时插两次**——尽量显式两个独立表。
- [ ] 角色挑食时记得加台词/UI 反馈，否则玩家迷惑"为啥点了没反应"。
- [ ] 想做"吃菜不掉血但掉精神"：`SetAbsorptionModifiers(0, 1, 1)` + 食物维持负 sanity。
- [ ] 注意 `Eater:Eat` 中 `food:HasTag("potion")` 的守门——药水不进入 foodmemory。

【老手】

- [ ] mod 不要绕过 SetDiet 手动 `table.insert(caneat)` ——不会自动打 `<X>_eater` 标签。
- [ ] 改字段直接赋值（如 `eater.healthabsorption = 2`）是合法的，但 `caneat` 字段赋值会触发 setter；preferseating 不会。
- [ ] `lasteattime` 由非 Eat 路径消耗食物时，记得手动同步。
- [ ] `feedincontainer` / `feedmount` 是 0/1/2 阶段事件链的最早提示——mod 不要在 oneat 监听里去判 "是不是喂"，太晚了。
- [ ] `IsTryingToFeedMe(inst)` 是 trader 类组件的标准前置判定，写新交易组件时优先复用。
- [ ] `oneat` 内禁止 `food:Remove()`——`HandleEatRemove` 后面要用 food，Klei 自己有 `inst:IsValid()` 兜底，但你别给自己挖坑。
- [ ] 想做"换 Diet"的 buff（如勋章 mod 的 `buff_medal_eat`），参考第三方实现：先备份原 caneat / preferseating 数组（注意 FOODGROUP 用 .name 序列化），再 SetDiet 到目标值，detach 时反序列化恢复。

---

至此 20.2 Eater 完结。下一节会把 Eater 与 Edible 推送的进食事件链（`oneat` / `oneaten` / `feedincontainer` / `feedmount`）以及它们在游戏中的具体监听者一一拆解，并讨论与状态机的对接细节。


## 20.3 进食回调链：oneatfn / oneaten / onfinisheating

> 本节面向三类读者编写，章节中会以「【新手】」「【进阶】」「【老手】」分别标注关键信息，方便不同层次的读者各取所需。
>
> - **【新手】**：已经会做"能吃的食物"和"会吃的生物"，希望知道"吃下后还能挂哪些副作用"，例如加 buff、推台词、出粪便。
> - **【进阶】**：理解一份食物从被选中、判定、吃下到状态机收尾的整条事件链；能区分 `oneat` / `oneaten` 这两个名字几乎一样、其实完全不同的事件。
> - **【老手】**：懂得在状态机层做"吃完后进入特殊动画"、"拒绝吃时打嘴炮"、"喂坐骑时推 mount_eat"，并能修复 mod 在事件链各阶段的常见隐患。
>
> 本节涉及的源码：
> - `scripts/components/eater.lua`（`oneatfn` 字段，`SetOnEatFn`，`oneat` / `feedincontainer` / `feedmount` 事件推送）
> - `scripts/components/edible.lua`（`oneaten` 字段，`SetOnEatenFn`，`oneaten` 事件推送）
> - `scripts/components/bait.lua`（`oneaten` 事件监听器经典样例）
> - `scripts/components/wisecracker.lua`（`oneat` 事件监听器经典样例）
> - `scripts/components/wereeater.lua`（`oneat` 事件监听器 + 状态触发）
> - `scripts/components/beefalometrics.lua`（`oneat` 事件用于统计）
> - `scripts/actions.lua` 第 760-771 行（`ACTIONS.EAT.fn`）
> - `scripts/actions.lua` 第 2362-2399 行（`ACTIONS.FEEDPLAYER.fn`）
> - `scripts/actions.lua` 第 3705-3735 行（`ACTIONS.FEED.fn`）
> - `scripts/stategraphs/SGwilson.lua` 第 1159-1194 行（`ACTIONS.EAT` ActionHandler）
> - `scripts/stategraphs/SGwilson.lua` 第 2276-2307 行（`wonteatfood` / `feedmount` 全局 EventHandler）
> - `scripts/stategraphs/SGwilson.lua` 第 6276-6534 行（`eat` / `quickeat` / `refuseeat` 三个状态）
> - `scripts/prefabs/player_common.lua` 第 534-538、690 行（玩家的 `oneat` / `wonteatfood` 监听）
> - `scripts/prefabs/player_common_extensions.lua` 第 918-923 行（玩家吃完料理写入 cookbook）
> - `scripts/wx78_moduledefs.lua` 第 1456-1466 行（消化模块用 `oneat` 监听 + `queue_post_eat_state` 触发"烤吐"状态）
>
> 教学示例 prefab：
> - `scripts/prefabs/preparedfoods.lua`（料理统一在 data 表的 `oneatenfn` 字段挂回调）
> - `scripts/spicedfoods.lua`（香料把额外 `oneatenfn` 与原回调串联）
> - `scripts/prefabs/firenettles.lua`（最简单"吃下加 debuff"的 `oneaten`）
> - `scripts/prefabs/mandrake_inactive.lua`（吃下让范围内 sleep）
> - `scripts/prefabs/moon_mushroom.lua`（吃月蘑菇昏睡）
> - `scripts/prefabs/tallbirdegg.lua`（裂蛋让玩家说台词的 `oneaten`）
> - `scripts/prefabs/ipecacsyrup.lua`（药水触发玩家拉屎状态机）
> - `scripts/prefabs/wormlight.lua`（吃下挂"萤光"灯）
> - `scripts/prefabs/wx78_common.lua` 第 978-991 行（WX-78 吃齿轮 / 充电）
> - `scripts/prefabs/wobysmall.lua` 第 657-661 行（Woby 吃 treat 推 `treatwoby`）
> - `scripts/prefabs/spider.lua` 第 366-370 行（蜘蛛吃突变剂触发突变）
> - `scripts/prefabs/pigman.lua` 第 128-139 行（猪人吃蔬菜拉屎、吃怪物肉触发狼人化）
> - `scripts/prefabs/beefalo.lua` 第 746-759 行（牛人喂食驯化系统）
> - `scripts/prefabs/monkey.lua` 第 55-64 行（猴子吃东西拉屎）
> - `scripts/prefabs/hats.lua` 第 2331-2342 行（眼面具帽吃食物修甲）
> - `scripts/prefabs/cookiecutter.lua` 第 98-100 行（饼干切割鱼咬下音效）

---

### 一、为什么会有"进食回调链"？（新手必读）

【新手】上两节我们看到 `Edible:OnEaten` 与 `Eater:Eat` 都在内部"额外做一些事情"——比如玩家吃裂蛋会说台词、Wormwood 吃榴莲会推坐骑事件。这些"额外做的事情"被故意拆开成了**三层独立扩展点**，分别挂在**三个不同的位置**：

| 层次 | 字段 / 事件 | 谁挂 | 何时触发 | 经典语义 |
| --- | --- | --- | --- | --- |
| Eater 字段 | `eater.oneatfn` | `Eater:SetOnEatFn(fn)` | 玩家 / 生物吃下东西后 | 吃方"刚吞下"的本地反应 |
| Eater 事件 | `oneat` | `inst:ListenForEvent("oneat", ...)` | 同上，紧跟 `oneatfn` | 多个监听者"同时关心吞下这件事" |
| Edible 字段 | `edible.oneaten` | `Edible:SetOnEatenFn(fn)` | 食物被吃下后 | 食物"被吞掉"前的最后一击（自己即将被销毁） |
| Edible 事件 | `oneaten` | `food:ListenForEvent("oneaten", ...)` | 同上，紧跟 `oneaten` 字段调用 | 第三方对"这份食物被吃"感兴趣（如鱼饵、礼仪宝箱） |
| 状态机事件 | `animqueueover` / `queue_post_eat_state` | `stategraph` 内部 | 吃食物动画播完 / 收到指令时 | 决定下一个动画状态 |

【新手】**这一节标题里的 `onfinisheating` 是个总括概念**，不是源码里的事件名。它指的是**状态机层面"吃完动画的收尾"**——具体由 `animqueueover` 事件触发回到 `idle`，或被 `queue_post_eat_state` 重定向到其他状态（如 `wx_bake`、`mount_eat`）。

把这三层放在一起讲，是因为它们**在时间线上接力发生**，常被新手误以为是同一回事：

```
玩家点了"吃"  ──┐
                ▼
        ACTIONS.EAT.fn ──→ eater:Eat(food)
                              │
                              ├── ① 算 health/hunger/sanity delta
                              ├── ② 应用 DoDelta 三件套
                              ├── ③ 推喂食事件 feedincontainer / feedmount
                              ├── ④ PushEvent "oneat" ←─── 多个监听者抢着接
                              ├── ⑤ 调 self.oneatfn(inst, food, feeder)
                              ├── ⑥ Edible:OnEaten(eater)
                              │        ├── ⓐ 调 edible.oneaten(food, eater)
                              │        ├── ⓑ 应用温度
                              │        ├── ⓒ PushEvent "oneaten"   ←─── 鱼饵等监听
                              │        └── ⓓ 播 eatensound
                              ├── ⑦ Edible:HandleEatRemove(整堆?)
                              └── ⑧ 记 lasteattime / foodmemory
                            
                            ▽ 与此并行
                状态机 SGwilson "eat" / "quickeat":
                  frame 28 / 12 ──→ 上面 ④~⑧ 在此刻发生
                  ⑨ 收 queue_post_eat_state (可选) ──→ statemem 记下
                  ⑩ animqueueover ──→ 进 idle 或 queued_post_eat_state
```

【进阶】上一节末尾我们粗略列过这条链路，但当时只画到 `Eater:Eat` 的 13 步。本节把第 4、5、6 步**展开**，并把第 9、10 步——也就是状态机这一侧——补全。**事件链的真正复杂度不在 Lua 层面，而在"哪个钩子的副作用会被哪个监听者撞到"**。这是 mod 开发里最容易踩坑的地方。

【老手】所有这些钩子全部是**同步执行**的（除非你显式 `DoTaskInTime`）。也就是说，如果 ⑤ `oneatfn` 里 `inst:Remove()` 了，⑥ 还会用 invalid 的 inst 去取 `food.components.edible`——`HandleEatRemove` 内部对食物有 `IsValid()` 保护，但 eater 自己**没保护**。本节最后会反复强调这一类时序坑。

---

### 二、源码总览：完整事件链一图流

把 `Eater:Eat`、`Edible:OnEaten` 和玩家状态机三处的关键片段并排放，就是这节的"全家福"：

```230:318:scripts/components/eater.lua
function Eater:Eat(food, feeder)
    feeder = feeder or self.inst
    if self:PrefersToEat(food) then
        -- ...（health/hunger/sanity 计算与 DoDelta，详见 20.2）

		if feeder ~= self.inst then
			if self.inst.components.inventoryitem then
				local owner = self.inst.components.inventoryitem:GetGrandOwner()
				if owner and (owner == feeder or (owner.components.container and owner.components.container:IsOpenedBy(feeder))) then
					feeder:PushEvent("feedincontainer")
				end
			end
			if self.inst.components.rideable and feeder == self.inst.components.rideable:GetRider() then
				feeder:PushEvent("feedmount", { food = food, eater = self.inst })
			end
		end

        self.inst:PushEvent("oneat", { food = food, feeder = feeder })
        if self.oneatfn ~= nil then
            self.oneatfn(self.inst, food, feeder)
        end

        food.components.edible:OnEaten(self.inst)
        food.components.edible:HandleEatRemove(self.eatwholestack)

        self.lasteattime = GetTime()

        if self.inst.components.foodmemory ~= nil and not food:HasTag("potion") then
            self.inst.components.foodmemory:RememberFood(food.prefab)
        end

        return true
    end
end
```

```183:214:scripts/components/edible.lua
function Edible:OnEaten(eater)
    if self.oneaten ~= nil then
        self.oneaten(self.inst, eater)
    end

    local delta_multiplier = 1
    local duration_multiplier = 1

    if self.spice and TUNING.SPICE_MULTIPLIERS[self.spice] then
        if TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDELTA then
            delta_multiplier = delta_multiplier + TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDELTA
        end

        if TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDURATION then
            duration_multiplier = duration_multiplier + TUNING.SPICE_MULTIPLIERS[self.spice].TEMPERATUREDURATION
        end
    end

    -- Food is an implicit heater/cooler if it has temperature
    if self.temperaturedelta ~= 0 and
        self.temperatureduration ~= 0 and
        self.chill < 1 and
        eater ~= nil and
        eater.components.temperature ~= nil then
        eater.components.temperature:SetTemperatureInBelly(self.temperaturedelta * (1 - self.chill) * delta_multiplier, self.temperatureduration * duration_multiplier)
    end

    self.inst:PushEvent("oneaten", { eater = eater })
    if self.inst.eatensound ~= nil and eater.SoundEmitter ~= nil then
        eater.SoundEmitter:PlaySound(self.inst.eatensound)
    end
end
```

【进阶】**关键观察**：

1. `Eater:Eat` 内部**先**推 `oneat` 事件，**后**调 `oneatfn` 字段。这是反直觉的——一般我们会以为"字段优先级高，事件兜底"，这里**事件先到**。
2. `Edible:OnEaten` 内部**先**调 `oneaten` 字段，**后**推 `oneaten` 事件。**字段优先**。两个组件的顺序相反，记不清的时候这样推：**带 fn 的字段总是单一注册者，"老大"。事件是任意多监听者的广播。眼罩帽吃食物修甲（字段）一定先于 wisecracker 评论（事件），是因为这是 Eater 侧；料理加 buff（字段）也一定先于鱼饵失踪事件（事件），但因为这是 Edible 侧——食物注入的逻辑一般比"周边监听"重要，所以字段在前合理**。
3. **Eater 的两个钩子结束后才进 Edible**。这意味着如果 `eater.oneatfn` 把吃方 `inst:Remove()` 了，`food.components.edible:OnEaten(self.inst)` 拿到的就是一个已 Remove 的 eater。
4. `HandleEatRemove` 在所有"内容钩子"之后才执行，所以你**可以**在 `oneaten` 字段或事件里继续访问 food（如 `food.prefab` 还有效），但**不能**假设 food 在 oneaten 事件之后还活着——它马上就被 `Get():Remove()` 或 `Remove()` 了。

【老手】上面四条共同决定了一个简单原则：**"凡事都假设是同步链上的下一个调用，前后兄弟之间共享同一帧 entity 状态"**。任何想破坏这个假设的 mod（比如要 `DelayBy(1, ...)` 再删 food），都需要用 `DoTaskInTime` 显式异步化，绝不能直接 `inst:Remove()`。

---

### 三、Eater 侧：`oneatfn` 字段与 `oneat` 事件

#### 3.1 `Eater:SetOnEatFn` 与 `oneatfn` 字段

```198:200:scripts/components/eater.lua
function Eater:SetOnEatFn(fn)
    self.oneatfn = fn
end
```

【函数解析】

- **作用**：注册一个"自己吞下东西后立刻执行"的回调，挂在 Eater 上。
- **参数**：
  - `fn`（function，必填）：签名 `fn(inst, food, feeder)`。
    - `inst`（Entity）：吃方实体本身（== `self.inst`）。
    - `food`（Entity）：吃下的食物。
    - `feeder`（Entity）：投喂者（自己吃自己时 == inst）。
- **返回值**：无。
- **实现流程**：仅赋值 `self.oneatfn = fn`，无副作用。

【进阶】**只有一个 `oneatfn`**。后注册者会**覆盖**前一个：

```lua
inst.components.eater:SetOnEatFn(fnA)
inst.components.eater:SetOnEatFn(fnB)  -- 现在只有 fnB 在生效，fnA 被覆盖
```

如果想"两个回调都执行"，**得自己包装**：

```lua
local oldfn = inst.components.eater.oneatfn
inst.components.eater:SetOnEatFn(function(inst, food, feeder)
    if oldfn ~= nil then oldfn(inst, food, feeder) end
    -- 你的新逻辑
end)
```

【老手】**注意 `oneatfn` 不存盘**——你必须在 `master_postinit` / 角色 prefab 的 `fn` 里重新注册，每次进游戏重新挂。这点和"事件监听需要重挂"的逻辑一致：函数引用不能序列化。

#### 3.2 经典 `oneatfn` 实战

下面给出 7 个不同复杂度的实战例子。

##### 3.2.1 最简单：吃东西放屁（猴子）

```55:64:scripts/prefabs/monkey.lua
local function oneat(inst)
    --Monkey ate some food. Give him some poop!
    if inst.components.inventory ~= nil then
        local maxpoop = 3
        local poopstack = inst.components.inventory:FindItem(IsPoop)
        if not poopstack or poopstack.components.stackable.stacksize < maxpoop then
            inst.components.inventory:GiveItem(SpawnPrefab("poop"))
        end
    end
end
```

【新手】注意：这里函数签名只声明了 `inst`，没接 food / feeder。Lua 允许"少接参数"，多余的参数被丢弃。**这是合法风格**，但**不要把它当成默认值**——如果你需要 food，必须显式声明 `function oneat(inst, food, feeder)`。

##### 3.2.2 双分支：分类型反应（猪人）

```128:139:scripts/prefabs/pigman.lua
local function OnEat(inst, food)
    if food.components.edible ~= nil then
        if food.components.edible.foodtype == FOODTYPE.VEGGIE then
            SpawnPrefab("poop").Transform:SetPosition(inst.Transform:GetWorldPosition())
        elseif food.components.edible.foodtype == FOODTYPE.MEAT and
            inst.components.werebeast ~= nil and
            not inst.components.werebeast:IsInWereState() and
            food.components.edible:GetHealth(inst) < 0 then
            inst.components.werebeast:TriggerDelta(1)
        end
    end
end
```

【进阶】**注意 `food.components.edible:GetHealth(inst)` 的取值是动态计算的**——它会算腐烂、香料、亲和等所有 multiplier。这里用 `< 0` 判断"是否吃下了对自己有害的肉"，正好等同于"是否吃了怪物肉/腐肉"。是个比 `HasTag("monstermeat")` 更通用的写法。

【老手】**关键的二次判定 `not IsInWereState()`**：werebeast 的狼变形状态机里，狼形态也能 `Eater:Eat`，但**不应再次触发狼人化**（否则会无限叠）。这是 stategraph + 组件协作时常见的"重入保护"。

##### 3.2.3 Beefalo：完整的喂食驯化逻辑

```746:759:scripts/prefabs/beefalo.lua
local function OnEat(inst, food, feeder)
    local full = inst.components.hunger:GetPercent() >= 1
    if not full then
        inst.components.domesticatable:DeltaObedience(TUNING.BEEFALO_DOMESTICATION_FEED_OBEDIENCE)

        inst.components.domesticatable:TryBecomeDomesticated()
    else
        inst.components.domesticatable:DeltaObedience(TUNING.BEEFALO_DOMESTICATION_OVERFEED_OBEDIENCE)
        inst.components.domesticatable:DeltaDomestication(TUNING.BEEFALO_DOMESTICATION_OVERFEED_DOMESTICATION, feeder)
        inst.components.domesticatable:DeltaTendency(TENDENCY.PUDGY, TUNING.BEEFALO_PUDGY_OVERFEED)
    end
    inst:PushEvent("eat", { full = full, food = food })
    inst.components.knownlocations:RememberLocation("loiteranchor", inst:GetPosition())
end
```

【进阶】这是同时使用 3 个参数的标准案例：

- `inst`：牛自己，去问自己饱不饱、改驯化度；
- `food`：吃下了什么（被 push 给 `eat` 事件用，brain 用 food.prefab 评估"喜爱程度"）；
- `feeder`：投喂者（驯化度增长归功给他，让 Wormwood 喂奶牛 vs Wolfgang 喂区分开来）。

【老手】注意末尾 `PushEvent("eat", ...)` —— 这是 beefalo 自定义的二级事件，**不要和系统级 `oneat` 事件混淆**。Klei 这里再推一个"eat" 事件，是为了让 beefalo 的 brain（`beefalobrain.lua`）和 stategraph（`SGBeefalo.lua` 注释掉了 ActionHandler）能监听到。本质上是**多此一举**——直接监听 `oneat` 也可以——但保留语义"我牛刚才吃了"。Mod 写仿生物时**不需要**学这个，直接用 `oneat`。

##### 3.2.4 WX-78：齿轮计数 + 充电

```978:991:scripts/prefabs/wx78_common.lua
local function OnEat(inst, food)
    local edible = food.components.edible
    if edible ~= nil then
        if edible.foodtype == FOODTYPE.GEARS then
            inst._gears_eaten = inst._gears_eaten + 1
            inst.SoundEmitter:PlaySound("dontstarve/characters/wx78/levelup")
        end

        local charge_amount = edible.chargevalue
        if charge_amount ~= nil and charge_amount ~= 0 then
            inst.components.upgrademoduleowner:DoDeltaCharge(charge_amount)
        end
    end
end
```

【进阶】WX-78 的 `OnEat` 同时处理两个机制：

- **齿轮统计**：吃一个齿轮 `_gears_eaten + 1`，死时会按这个数量回吐齿轮（`DropEatenGears`）。
- **充电**：食物可以挂一个**自定义字段** `edible.chargevalue`（不是 Klei 标准 Edible 字段，是 WX 加的扩展）。Edible 不知道这个字段，但 WX 的 oneatfn 主动去读，吃下后 `upgrademoduleowner:DoDeltaCharge`。

【老手】**这就是 mod 添加角色特有属性的标准做法**：往 Edible 上加一个 mod 私有字段（如 `chargevalue`），不去碰 Edible 本身，吃下时由 mod 角色的 `oneatfn` 主动读取。**不要直接修改 Edible:GetHealth / GetHunger / GetSanity**——那是 Klei 设计的标准接口。

##### 3.2.5 Walter：吃特定食物启动剧情

```557:561:scripts/prefabs/walter.lua
local function oneat(inst, food)
	if food ~= nil and food:IsValid() and (food.prefab == "glommerfuel" or food:HasTag("tallbirdegg")) then
        inst:ListenForEvent("animqueueover", startsong)
	end
end
```

【进阶】这是 `oneatfn` 与 **状态机** 配合的范例：

- `Eater:Eat` 在状态机 `eat` 的 frame 28 调用，触发 `oneatfn`；
- `oneatfn` 监听 `animqueueover` 事件，即"`PushAnimation("eat", false)` 播完"那一刻；
- 那时玩家 `eat` state 还没 GoToState 到 idle，所以 `startsong` 在 idle 之前抢先执行（具体逻辑 startsong 可能播台词或开启唱歌动画）。

这是用 oneatfn 触发"状态机次级事件"的经典写法。**注意一定要 `inst:RemoveEventCallback("animqueueover", startsong)` 防止泄漏**（看 `startsong` 内部应该会自己 remove）。

##### 3.2.6 Woby（小）：吃 treat 推 `treatwoby` 让主人开心

```657:661:scripts/prefabs/wobysmall.lua
local function OnEat(inst, food, feeder)
	if food:HasTag("pet_treat") then
		feeder:PushEvent("treatwoby", inst)
	end
end
```

【进阶】这里 `feeder` 是 Walter（投喂者），`inst` 是 Woby（被喂者）。`PushEvent("treatwoby", inst)` 在 feeder 上，data 就是 Woby 本身。

【老手】Walter 的 `Wisecracker` 组件就是这条事件的下游：

```419:419:scripts/components/wisecracker.lua
		inst:ListenForEvent("treatwoby",		function(inst, woby) talktowoby(inst, woby, "ANNOUNCE_WOBY_PRAISE",	4,	6,	0) end)
```

【老手】**这就是事件链的扩展性魅力**——Woby 的 oneatfn 只关心"我吃到 treat 了"，至于这个事件后面接谁，由 Walter 的 wisecracker 决定。**Woby 不需要知道 Walter 的存在**，wisecracker 不需要知道 Woby 怎么吃饭，**双方通过事件名 `treatwoby` 解耦**。这种"事件即接口"的设计是 DS 整个 codebase 的核心范式。

##### 3.2.7 眼面具帽：吃食物按 health+hunger 修甲

```2331:2342:scripts/prefabs/hats.lua
	local function eyemask_oneatfn(inst, food)
		local health = math.abs(food.components.edible:GetHealth(inst)) * inst.components.eater.healthabsorption
		local hunger = math.abs(food.components.edible:GetHunger(inst)) * inst.components.eater.hungerabsorption
		inst.components.armor:Repair(health + hunger)

		if not inst.inlimbo then
			inst.AnimState:PlayAnimation("eat")
			inst.AnimState:PushAnimation("anim", true)

			inst.SoundEmitter:PlaySound("terraria1/eyemask/eat")
		end
	end
```

【进阶】这是**用 oneatfn 重新利用 Edible 的算值能力**——眼面具帽不在乎"血量加多少"（它没有 health 组件），但它**借用 `Edible:GetHealth` 当作"营养度"计算单位**，把 abs 后乘 absorption，得到修甲量。`Eater:Eat` 这边因为 health/sanity 都被 absorption 0 吸收过滤掉了，没真扣血扣 sanity，只剩 hunger 进了 hunger 组件。

【老手】**注意 `inlimbo` 守门**：被吃下后这一刻 hat 自己已经从某处被取走（戴在头上），但还没进入 limbo。播动画前要确认"我现在不在虚空"才能播。这是 Klei 自家的 limbo 防御性写法，mod 可以借鉴。

#### 3.3 `oneat` 事件（玩家身上推送）

```302:303:scripts/components/eater.lua
        self.inst:PushEvent("oneat", { food = food, feeder = feeder })
```

【事件解析】

- **事件名**：`oneat`
- **含义**：该实体（eater）吃下了一份食物。
- **推送时机**：`Eater:Eat` 内部，**delta 已应用、food 还未被 OnEaten / Remove**。
- **推送数据**：
  - `food`（Entity）：吃下的食物，还有效。
  - `feeder`（Entity）：投喂者；自己吃自己时 == eater。
- **推送对象**：吃食物的实体（玩家 / 生物 / 坐骑）。

【进阶】这是**整条链路里最容易被 mod 监听的事件**。原因：

- **不需要改源码**：用 `inst:ListenForEvent("oneat", fn)` 就行。
- **多 mod 共存**：N 个 mod 同时监听 oneat 不会互相覆盖（不像 `oneatfn` 字段会被覆盖）。
- **能拿到 food + feeder**：`data.food`、`data.feeder` 全有。

下面是源码里 `oneat` 事件的所有监听者（搜索 `ListenForEvent("oneat"`）：

| 文件 | 用途 |
| --- | --- |
| `prefabs/player_common.lua:690` | 玩家吃料理 → 注册到 cookbook 食谱书 |
| `components/wisecracker.lua:8` | 各类台词（吃到腐烂、吃到痛苦食物、吃同样食物吃腻、吃陈旧、masterchef 评价） |
| `prefabs/merm.lua:1363` | merm 的 OnEat 处理 |
| `prefabs/spider.lua:727` | 蜘蛛吃到 spidermutator 突变剂时变种 |
| `prefabs/dustmoth.lua:276` | 飞蛾的 OnEat |
| `prefabs/carrat.lua:745` | YOTC（鼠年活动）carrat 教练评价 |
| `components/crittertraits.lua:120` | 小可爱（critter）评估 wellfed、playful 等特质 |
| `components/wereeater.lua:12` | 玩家狼变形系统：吃 monstermeat 累计计数 |
| `components/beefalometrics.lua:129` | 牛年活动评估 |
| `wx78_moduledefs.lua:1482` | WX-78 消化模块：吃腐烂物累计触发 `wx_bake` 状态 |

【老手】**`oneat` 的最强典型应用**：`scripts/components/wereeater.lua`：

```1:13:scripts/components/wereeater.lua
local function OnEat(inst, data)
    inst.components.wereeater:EatMosterFood(data)
end

local WereEater = Class(function(self, inst)
    self.inst = inst
    self.duration = TUNING.TOTAL_DAY_TIME / 2
    self.monster_count = 0
    self.forget_task = nil
    self.forcetransformfn = nil

    inst:ListenForEvent("oneat", OnEat)
end)
```

注意它**直接在 `Class` 构造里就 ListenForEvent**——这是 Klei 内置组件的标准做法。`OnRemoveFromEntity` 里同步反注册（`RemoveEventCallback`）以防泄漏。**Mod 写组件时要照搬这个模式**，否则组件被换走后事件回调还在跑。

#### 3.4 `oneatfn` vs `oneat` 事件：什么时候用哪个？

| 维度 | `oneatfn` 字段 | `oneat` 事件 |
| --- | --- | --- |
| 注册方式 | `eater:SetOnEatFn(fn)` | `inst:ListenForEvent("oneat", fn)` |
| 是否覆盖 | **会覆盖**（只有一个） | 不会（任意多个监听者） |
| 推送顺序 | **晚于事件 1 微秒** | **早于字段** |
| 调用签名 | `fn(inst, food, feeder)` | `fn(inst, data)`，`data = {food, feeder}` |
| 是否随存档 | **不随**，必须 prefab fn 重新注册 | 不随，必须 ListenForEvent 重新注册 |
| 推荐用途 | **角色 / 生物**自己的"核心吃食反应" | **额外** mod、**额外** 组件、**额外** 评论 |

【进阶】**判断指南**：

- 你在写**生物 / 角色 prefab** 的"我自己吃了东西要 X"——用 `oneatfn`。
- 你在写**第三方 mod 钩子**、**外挂组件**、**Buff 系统** —— 用 `oneat` 事件。
- 你**两个都想用**——`oneatfn` 先 SetOnEatFn 装"主反应"，再 `ListenForEvent("oneat", fn)` 加"次反应"，二者**完全独立**，互不干扰。

【老手】**有些 Klei 自家 prefab 两个都用了**——例如 carrat、merm、dustmoth、smallbird 都同时挂了 `oneatfn` 和 `oneat` 监听器。同时用没问题，因为它们处理的是不同语义层面。

---

### 四、Edible 侧：`oneaten` 字段与 `oneaten` 事件

#### 4.1 `Edible:SetOnEatenFn` 与 `oneaten` 字段

```149:151:scripts/components/edible.lua
function Edible:SetOnEatenFn(fn)
    self.oneaten = fn
end
```

【函数解析】

- **作用**：注册"食物被吞下时执行"的回调，挂在食物 Edible 上。
- **参数**：
  - `fn`（function，必填）：签名 `fn(inst, eater)`。
    - `inst`（Entity）：食物自身。
    - `eater`（Entity）：吃下这份食物的实体。
- **返回值**：无。
- **实现流程**：仅赋值 `self.oneaten = fn`。

【新手】**这是给食物 prefab 挂"吃下去发生什么"的标准入口**。`firenettles.lua` 是最简单的例子：

```13:17:scripts/prefabs/firenettles.lua
local function oneaten(inst, eater)
	if not eater:HasTag("plantkin") then
        eater:AddDebuff("firenettle_toxin", "firenettle_toxin")
	end
end
```

吃下火荨麻 → 给吃方加"火毒"debuff。`plantkin` 标签（Wormwood）免疫。**5 行代码就够了**。

#### 4.2 `oneaten` 在料理系统中的统一注入

【进阶】Klei 把**所有料理 / 香料料理**的吃下回调都集中放在 `preparedfoods.lua` 的 data 表里：

```113:124:scripts/prefabs/preparedfoods.lua
        inst:AddComponent("edible")
        inst.components.edible.healthvalue = data.health
        inst.components.edible.hungervalue = data.hunger
        inst.components.edible.foodtype = data.foodtype or FOODTYPE.GENERIC
        inst.components.edible.secondaryfoodtype = data.secondaryfoodtype or nil
        inst.components.edible.sanityvalue = data.sanity or 0
        inst.components.edible.temperaturedelta = data.temperature or 0
        inst.components.edible.temperatureduration = data.temperatureduration or 0
        inst.components.edible.nochill = data.nochill or nil
        inst.components.edible.spice = data.spice
        inst.components.edible:SetOnEatenFn(data.oneatenfn)
        inst.components.edible.chargevalue = data.chargevalue or nil -- Wx-78
```

【进阶】**所以全部 100+ 种料理共享同一个 prefab fn**，仅靠 data 表的 `oneatenfn` 字段挂自定义回调。具体一份料理的回调长这样：

```460:477:scripts/prefabs/preparedfoods.lua
		health = TUNING.JELLYBEAN_TICK_VALUE,
		hunger = 0,
		perishtime = nil, -- not perishable
		sanity = TUNING.SANITY_TINY,
		cooktime = 2.5,
        potlevel = "low",
		tags = {"honeyed"},
		stacksize = 3,
        prefabs = { "healthregenbuff" },
		oneat_desc = STRINGS.UI.COOKBOOK.FOOD_EFFECTS_HEALTH_REGEN,
        oneatenfn = function(inst, eater)
			eater:AddDebuff("healthregenbuff", "healthregenbuff")
        end,
        floater = {"small", nil, 0.85},
		scrapbook_healthvalue = 122, -- First tick + total ticks
	},
```

【老手】jellybean（豆糖）的 `oneatenfn` 就一行 `AddDebuff("healthregenbuff", "healthregenbuff")`，挂一个回血 buff。

【老手】**自定义 mod 料理时，按 preparedfoods 的数据格式**：

```lua
mymod_dish =
{
    test = function(cooker, names, tags) return ... end,
    priority = 50,
    health = TUNING.HEALING_MED,
    hunger = TUNING.CALORIES_LARGE,
    sanity = TUNING.SANITY_MED,
    perishtime = TUNING.PERISH_MED,
    cooktime = 1,
    prefabs = { "my_buff" },
    oneat_desc = "My custom buff",
    oneatenfn = function(inst, eater)
        eater:AddDebuff("my_buff", "my_buff")
    end,
}
```

只要把这条 data 通过 `AddCookerRecipe("cookpot", "mymod_dish", ...)` 注册到锅里，Klei 的 preparedfoods fn 会自动给食物 prefab 挂上 oneatenfn——**你完全不用碰 Edible**。

#### 4.3 `oneaten` 事件（食物身上推送）

```210:210:scripts/components/edible.lua
    self.inst:PushEvent("oneaten", { eater = eater })
```

【事件解析】

- **事件名**：`oneaten`
- **含义**：这份食物 prefab 已被 eater 吃下。
- **推送时机**：`Edible:OnEaten` 中段 —— **`oneaten` 字段已执行、温度已应用、还未 `HandleEatRemove`**。
- **推送数据**：
  - `eater`（Entity）：吃下这份食物的实体。
- **推送对象**：食物自身（`food.inst`）。

【进阶】这是**所有源码里只有少数监听者关心的事件**。完整列表：

| 文件 | 用途 |
| --- | --- |
| `components/bait.lua:27` | 鱼饵：被吃时通知陷阱"鱼饵被取走了" |
| `brains/cozy_bunnymanbrain.lua:541` | YOTR（兔年活动）兔人 brain 监视玩家是否吃下了献祭品 |

【新手】**事件这么少，是不是没人用？** 不是。它**专门给食物本身的"周边系统"用**——比如鱼饵本来挂在陷阱上、被偷或被吃要触发陷阱内部清理。Klei 在 `bait.lua` 里这样写：

```7:11:scripts/components/bait.lua
local function OnEaten(inst, data)
    if inst.components.bait.trap ~= nil then
        inst.components.bait.trap:BaitTaken(data.eater)
    end
end
```

【老手】**为什么不直接在 `oneaten` 字段里调 BaitTaken？** 因为 bait 是一个**通用组件**，**任何**食物 prefab（如 berries）都可以挂 `bait`。如果它要去抢占 Edible 的 `oneaten` 字段，那 berries 自己的 `oneaten` 就被覆盖了——Klei 不知道你 mod 的某个东西会不会在 berries 上塞 oneaten。**事件不抢占任何字段，是最干净的多组件协作方式**。

#### 4.4 `oneaten` 字段 vs `oneaten` 事件：什么时候用哪个？

| 维度 | `oneaten` 字段 | `oneaten` 事件 |
| --- | --- | --- |
| 注册方式 | `edible:SetOnEatenFn(fn)` | `food:ListenForEvent("oneaten", fn)` |
| 是否覆盖 | **会覆盖**（只有一个） | 不会 |
| 推送顺序 | **早于事件** | 晚 1 微秒 |
| 调用签名 | `fn(food, eater)` | `fn(food, data)`，`data = {eater}` |
| 推荐用途 | **食物核心效果**（buff、debuff、状态变化） | 食物**外挂组件**（鱼饵、礼仪、追踪） |

【进阶】**最常见用法**：

- 你写**单个食物 prefab** 的"吃下加什么 buff"——用 `oneaten` 字段。
- 你写**通用组件**（如自定义鱼饵）—— 用 `oneaten` 事件。
- 你写**料理** —— 在 `preparedfoods.lua` 的 data 表加 `oneatenfn`，由 Klei 帮你 `SetOnEatenFn`。
- 你写**香料** —— 在 `spicedfoods.lua` 的 SPICES 表加 `oneatenfn`，Klei 会把它和原食物 `oneatenfn` **串联**。

#### 4.5 香料系统的链式 `oneatenfn`

【进阶】`spicedfoods.lua` 展示了"两个 oneatenfn 串联"的标准做法：

```65:75:scripts/spicedfoods.lua
            if spicedata.oneatenfn ~= nil then
                if newdata.oneatenfn ~= nil then
                    local oneatenfn_old = newdata.oneatenfn
                    newdata.oneatenfn = function(inst, eater)
                        spicedata.oneatenfn(inst, eater)
                        oneatenfn_old(inst, eater)
                    end
                else
                    newdata.oneatenfn = spicedata.oneatenfn
                end
            end
```

【老手】这就是上面 3.1 节给出的"包装旧 fn"标准模板。Klei 这边把它写成静态生成器：原料理的 oneatenfn（如 jellybean 的回血 buff）+ 香料的 oneatenfn（如大蒜的攻击 buff）会**两个都跑**，顺序是"先香料再原料理"。

【老手】**你要在 mod 里"给所有现存食物挂一个 oneatenfn"**，比如做"全食物加 sanity"buff，可以学这种链式写法：

```lua
-- 假设 some_food 已经有 SetOnEatenFn 注册过
local edible = some_food.components.edible
local old_oneaten = edible.oneaten
edible:SetOnEatenFn(function(inst, eater)
    if old_oneaten then old_oneaten(inst, eater) end
    eater.components.sanity:DoDelta(5)  -- 你的新效果
end)
```

但**注意**：这只对当前 instance 生效。要对所有 instance 生效，得 hook `Edible:OnEaten` 或在 prefab 创建后用 modutil 的 `AddPrefabPostInit`。

---

### 五、状态机侧：`onfinisheating` 含义与拆解

【新手】**严格来说，源码里没有名叫 `onfinisheating` 的事件**。但"吃完动作的收尾"是真实存在的——它由三件事组合而成：

1. **`animqueueover` 事件**：玩家 `eat` / `quickeat` 状态机里推 `eat_pre` + `eat` 两段动画，等动画队列播完触发。
2. **`queue_post_eat_state` 事件**：外部告诉状态机"吃完后跳到指定状态，而不是 idle"。
3. **`refuseeat` 状态**：当 `wonteatfood` 事件触发时进入的"摇头吐出来"状态。

把这三件事拼起来，就是"吃完动画的全套收尾流程"。

#### 5.1 玩家 SGwilson 的 `eat` 状态全流程

```6275:6388:scripts/stategraphs/SGwilson.lua
    State{
        name = "eat",
		tags = { "busy", "nodangle", "keep_pocket_rummage" },

        onenter = function(inst, foodinfo)
            inst.components.locomotor:Stop()

            local feed = foodinfo and foodinfo.feed
            if feed ~= nil then
                inst.components.locomotor:Clear()
                inst:ClearBufferedAction()
                inst.sg.statemem.feed = foodinfo.feed
                inst.sg.statemem.feeder = foodinfo.feeder
				inst.sg.statemem.feedwasactiveitem = foodinfo.active
                inst.sg:AddStateTag("pausepredict")
                if inst.components.playercontroller ~= nil then
                    inst.components.playercontroller:RemotePausePrediction()
                end
            elseif inst:GetBufferedAction() then
                feed = inst:GetBufferedAction().invobject
            end
            -- ... 此处省略 soulfx、heavy_eat 等分支 ...

            inst.AnimState:PlayAnimation("eat_pre")
            inst.AnimState:PushAnimation("eat", false)

            inst.components.hunger:Pause()
        end,

        timeline =
        {
			FrameEvent(6, DoEatSound),
            TimeEvent(28 * FRAMES, function(inst)
                if inst.sg.statemem.feed == nil then
                    inst:PerformBufferedAction()
                elseif inst.sg.statemem.feed.components.soul == nil then
                    inst.components.eater:Eat(inst.sg.statemem.feed, inst.sg.statemem.feeder)
                elseif inst.components.souleater ~= nil then
                    inst.components.souleater:EatSoul(inst.sg.statemem.feed)
                end
				--NOTE: "queue_post_eat_state" can be triggered immediately from the eat action
            end),

            TimeEvent(30 * FRAMES, function(inst)
				if inst.sg.statemem.queued_post_eat_state == nil then
					inst.sg:RemoveStateTag("busy")
					inst.sg:RemoveStateTag("pausepredict")
				end
            end),
			FrameEvent(52, function(inst)
				if inst.sg.statemem.queued_post_eat_state ~= nil then
					inst.sg:GoToState(inst.sg.statemem.queued_post_eat_state)
				end
			end),
            TimeEvent(70 * FRAMES, function(inst)
				if inst.sg.statemem.doeatingsfx then
					inst.sg.statemem.doeatingsfx = nil
					inst.SoundEmitter:KillSound("eating")
				end
            end),
			FrameEvent(94, TryResumePocketRummage),
        },

        events =
        {
			EventHandler("queue_post_eat_state", function(inst, data)
				--NOTE: this event can trigger instantly instead of buffered
				if data ~= nil then
					inst.sg.statemem.queued_post_eat_state = data.post_eat_state
					if data.nointerrupt then
						inst.sg:AddStateTag("nointerrupt")
					end
				end
			end),
            EventHandler("animqueueover", function(inst)
                if inst.AnimState:AnimDone() then
					inst.sg:GoToState(inst.sg.statemem.queued_post_eat_state or "idle")
                end
            end),
        },
        -- ... onexit 省略 ...
    },
```

【进阶】**关键节点（按帧）**：

| frame | 行为 |
| --- | --- |
| 0 (onenter) | 暂停 hunger，播 eat_pre + eat 动画 |
| 6 | 播咀嚼音效（`DoEatSound`） |
| 28 | **核心**：调 `eater:Eat(feed, feeder)`——20.2 那条 13 步链路全部在这一帧执行 |
| 30 | 如果没有 `queue_post_eat_state` 排队，移除 busy 标签，允许玩家行动 |
| 52 | 如果 `queue_post_eat_state` 已排队，直接 GoToState 跳转 |
| 70 | 杀掉咀嚼音 |
| 94 | 尝试恢复"翻包"动作 |
| animqueueover | 动画播完，GoToState 到 `idle` 或 `queued_post_eat_state` |

【老手】**`queue_post_eat_state` 的两条路径**：

- **早到**：在 frame 28 的 `eater:Eat` 内部由某些 oneatfn / oneat 监听器立刻推（`PushEventImmediate`），statemem 已记下，frame 30 检测到不为 nil **跳过移除 busy 标签**，等 frame 52 直接跳转目标状态。
- **晚到**：动画播完前外部推送，先存 statemem，等 `animqueueover` 时 GoToState 到目标状态。

frame 28 的注释 `--NOTE: "queue_post_eat_state" can be triggered immediately from the eat action` 说明了这种"在事件链中段插队"是预期行为。

#### 5.2 `queue_post_eat_state`：吃完后跳转指定状态

```6359:6367:scripts/stategraphs/SGwilson.lua
			EventHandler("queue_post_eat_state", function(inst, data)
				--NOTE: this event can trigger instantly instead of buffered
				if data ~= nil then
					inst.sg.statemem.queued_post_eat_state = data.post_eat_state
					if data.nointerrupt then
						inst.sg:AddStateTag("nointerrupt")
					end
				end
			end),
```

【事件解析】

- **事件名**：`queue_post_eat_state`
- **含义**：告诉状态机"等吃完动作播完后，跳到指定的状态而不是 idle"。
- **推送时机**：通常在 `oneat` 监听里 PushEventImmediate（必须 immediate，不然事件会被 buffer 到下一帧，错过 statemem 记录窗口）。
- **推送数据**：
  - `post_eat_state`（string）：目标状态名（如 `"wx_bake"`、`"refuseeat"`）。
  - `nointerrupt`（bool，可选）：是否锁定状态机不允许 hit interrupt。
- **推送对象**：玩家自己（eater）。

【进阶】**WX-78 消化模块的标准用法**：

```1456:1466:scripts/wx78_moduledefs.lua
local function digestion_OnEaten(wx, data)
    if data ~= nil
        and data.food ~= nil
        and wx.components.eater:CanProcessSpoiledItem(data.food) then
        wx._num_spoiledfood_eaten = (wx._num_spoiledfood_eaten or 0) + 1
        if wx._num_spoiledfood_eaten >= (TUNING.WX78_DIGESTION_SPOILED_NEEDED - (wx._digestion_modules - 1)) then
            wx._num_spoiledfood_eaten = 0
            wx:PushEventImmediate("queue_post_eat_state", { post_eat_state = "wx_bake" })
        end
    end
end
```

【老手】**`PushEventImmediate` vs `PushEvent`**：

- `PushEvent`：事件被 buffer，**下一帧**才被监听器处理。
- `PushEventImmediate`：事件**立刻**同步分发给所有监听器。

【老手】**这里为什么必须 `PushEventImmediate`？** 因为 frame 28 的事件链里：

1. `Eater:Eat` 推 `oneat` 事件（这一行）；
2. WX-78 的 `digestion_OnEaten` 立刻收到（同一帧）；
3. 它推 `queue_post_eat_state`；
4. **如果用普通 `PushEvent`**，事件被 buffer 到下一帧，frame 28 检查 statemem 时 `queued_post_eat_state` 还是 nil，结果**frame 30 移除 busy，frame 52 不跳转**——WX 该烤吐却没烤。
5. **用 `PushEventImmediate`**，状态机的 EventHandler 立刻执行，statemem 记上 `"wx_bake"`，frame 30 检测到不为 nil 跳过解锁，frame 52 跳转到 wx_bake。✓

【老手】**这是 mod 写"吃完后特殊状态"的标准模板**：

```lua
inst:ListenForEvent("oneat", function(inst, data)
    if data.food and data.food.prefab == "my_special_food" then
        inst:PushEventImmediate("queue_post_eat_state", {
            post_eat_state = "my_special_post_eat_state",
            nointerrupt = true,
        })
    end
end)
```

注意要在状态机里**先定义** `my_special_post_eat_state` 状态，否则 GoToState 会 crash。

#### 5.3 `wonteatfood` 事件与 `refuseeat` 状态

```1167:1180:scripts/stategraphs/SGwilson.lua
            elseif obj.components.edible ~= nil then
                if not inst.components.eater:PrefersToEat(obj) then
                    inst:PushEvent("wonteatfood", { food = obj })
                    return
                end
            elseif obj.components.soul ~= nil then
                if inst.components.souleater == nil then
                    inst:PushEvent("wonteatfood", { food = obj })
                    return
                end
            else
                return
            end
```

【事件解析】

- **事件名**：`wonteatfood`
- **含义**：尝试吃某食物，但 PrefersToEat 不通过（角色挑食、food 不是它能接受的）。
- **推送时机**：
  - 路径 A：`ACTIONS.EAT` 的 ActionHandler 调度时（玩家自己点"吃"按钮）。
  - 路径 B：`ACTIONS.FEEDPLAYER.fn` 内部（喂队友时）。
- **推送数据**：
  - `food`（Entity）：那份被拒绝的食物。
- **推送对象**：吃方（玩家、生物）。

```2276:2281:scripts/stategraphs/SGwilson.lua
    EventHandler("wonteatfood",
        function(inst)
			if inst.components.health and not inst.components.health:IsDead() and not inst.sg:HasStateTag("floating") then
                inst.sg:GoToState("refuseeat")
            end
        end),
```

```6487:6534:scripts/stategraphs/SGwilson.lua
    State{
        name = "refuseeat",
		tags = { "busy", "pausepredict", "keep_pocket_rummage" },

        onenter = function(inst)
            inst.components.locomotor:Stop()
            inst.components.locomotor:Clear()
            inst:ClearBufferedAction()

            if inst.components.rider:IsRiding() then
                DoTalkSound(inst)
                inst.AnimState:PlayAnimation("dial_loop")
            else
                DoTalkSound(inst)
                inst.AnimState:PlayAnimation(inst.components.inventory:IsHeavyLifting() and "heavy_refuseeat" or "refuseeat")
				inst.sg:SetTimeout(60 * FRAMES)
            end
            -- ...
```

【进阶】**`refuseeat` 状态做的事**：

- 播 `refuseeat` 动画（摇头吐出来）。
- `DoTalkSound`，触发台词系统。
- 设置 timeout 60 帧，避免动画卡死。

【老手】**玩家身上还有一个并行监听**：

```534:538:scripts/prefabs/player_common.lua
local function OnWontEatFood(inst, data)
    if inst.components.talker ~= nil then
        inst.components.talker:Say(GetString(inst, "ANNOUNCE_EAT", "YUCKY"))
    end
end
```

```676:676:scripts/prefabs/player_common.lua
    inst:ListenForEvent("wonteatfood", OnWontEatFood)
```

【老手】**所以一份食物被拒绝时，会同时发生两件事**：

1. SGwilson 全局 EventHandler 收到 `wonteatfood` → GoToState `refuseeat`（动画 + sound）；
2. `player_common.lua` 的监听器收到同一事件 → `talker:Say("ANNOUNCE_EAT.YUCKY")` 喷台词。

**两者完全独立**，互不干扰——这就是事件系统多监听者的妙处。

【老手】**Mod 角色"挑食有台词"的实现**：你**不用**自己手写 OnWontEatFood，**只要**在 STRINGS 表里给你的角色覆写 `ANNOUNCE_EAT.YUCKY` 即可：

```lua
STRINGS.CHARACTERS.MYMOD.ANNOUNCE_EAT.YUCKY = "我才不吃这种东西！"
```

#### 5.4 `feedmount` 事件与 `mount_eat` 状态

```2300:2307:scripts/stategraphs/SGwilson.lua
	EventHandler("feedmount",
		function(inst, data)
			if not (inst.sg:HasStateTag("busy") or inst.components.health:IsDead()) and
				data and data.eater and data.eater == inst.components.rider:GetMount()
			then
				inst.sg:GoToState("mount_eat")
			end
		end),
```

【进阶】这是**喂坐骑**的完整链：

1. 玩家骑在 beefalo 上，把胡萝卜拖给 beefalo（FEED action）；
2. `ACTIONS.FEED.fn` 调用 `beefalo.components.eater:Eat(carrot, walter)`；
3. beefalo 的 Eater:Eat 内部因为 `feeder ≠ self.inst`，推 `feedmount` 给 feeder（即 walter）；
4. walter 的 SGwilson 收到 `feedmount`，GoToState 到 `mount_eat`（喂食动画，由玩家这边播）；
5. beefalo 自己进入 `eat_loop` 状态（咀嚼动画）。

**注意**：玩家身上播 mount_eat、坐骑身上播 eat_loop——两个动画并行。这是"双方都展示在吃"的视觉设计。

【老手】**`mount_eat` 状态完整看 SGwilson 的搜索结果**——在玩家骑坐骑时一个独立的动画，模拟玩家俯身喂食的动作。Mod 做坐骑生物时，**只要** beefalo 的 eater 配置正确，玩家这边的喂食动画**自动**有；只有 sg 不接 `feedmount` 才会卡。

---

### 六、`feedincontainer` 与 `feedmount` 事件详解

```290:300:scripts/components/eater.lua
		if feeder ~= self.inst then
			if self.inst.components.inventoryitem then
				local owner = self.inst.components.inventoryitem:GetGrandOwner()
				if owner and (owner == feeder or (owner.components.container and owner.components.container:IsOpenedBy(feeder))) then
					feeder:PushEvent("feedincontainer")
				end
			end
			if self.inst.components.rideable and feeder == self.inst.components.rideable:GetRider() then
				feeder:PushEvent("feedmount", { food = food, eater = self.inst })
			end
		end
```

【进阶】这两个事件**只在他人投喂时推送**（`feeder ~= self.inst`），自己吃自己不会触发。

#### 6.1 `feedincontainer`

【事件解析】

- **事件名**：`feedincontainer`
- **含义**：feeder 通过"自己的背包"或"自己打开的容器"喂给了 self.inst 食物。
- **推送时机**：feeder ≠ self.inst，eater 是 inventoryitem 且 grandowner == feeder（或 grandowner.container 被 feeder 打开）。
- **推送数据**：无。
- **推送对象**：feeder（投喂者）。

【进阶】**主要应用场景**：

```1174:1174:scripts/prefabs/player_classified.lua
    inst:ListenForEvent("feedincontainer", OnFeedInContainer, inst._parent)
```

`player_classified.lua` 监听这个事件，用来在客户端弹动画反馈（"袋子里的东西被吃了"）。但 mod 一般用不到这层。

【老手】**坐骑 / 盆栽 / mandrake 是典型的 inventory eater**——他们被 Wormwood 喂背包里的莓果时，feeder 是 Wormwood、self.inst 是 mandrake、grandowner 是 Wormwood。**于是 feedincontainer 在 Wormwood 身上推送**，Wormwood 自己监听 feedincontainer 就能知道"我刚通过容器喂了我背包里的东西"。

#### 6.2 `feedmount`

【事件解析】

- **事件名**：`feedmount`
- **含义**：feeder 把食物喂给了自己骑着的坐骑。
- **推送时机**：feeder ≠ self.inst，eater 是 rideable 且 feeder 是当前 rider。
- **推送数据**：
  - `food`（Entity）：吃下的食物。
  - `eater`（Entity）：被喂的坐骑（即 self.inst）。
- **推送对象**：feeder（rider）。

【进阶】玩家骑 beefalo 时 `feedmount` 触发 `mount_eat` 状态——见 5.4。**这是 SGwilson 唯一直接监听的 feed 事件**。

【老手】**Mod 角色的坐骑亲密度系统**：监听 feeder 上的 feedmount，data.eater 拿到坐骑实体，data.food 拿到食物，做加亲密度 / 改坐骑心情等：

```lua
inst:ListenForEvent("feedmount", function(walter, data)
    if data.eater and data.food then
        -- data.eater 是坐骑，data.food 是食物
        if data.food.prefab == "my_special_treat" then
            data.eater:AddDebuff("happybuff", "happybuff")
        end
    end
end)
```

---

### 七、典型用例集锦（按链上位置归类）

#### 7.1 在 `oneat` 监听器里推 `queue_post_eat_state`（最复杂）

- 见 5.2 的 WX-78 消化模块。
- **关键**：用 `PushEventImmediate`，不要用 `PushEvent`。

#### 7.2 在 `oneatfn` 里推自定义事件（让其他系统知道）

- Woby 推 `treatwoby`（见 3.2.6）。
- beefalo 推 `eat`（见 3.2.3）。

#### 7.3 在 `oneatfn` 里改驯化 / 心情度

- beefalo 改 obedience / domestication / tendency。
- 注意区分 `full` 与 `not full` 的两条分支。

#### 7.4 在 `oneatenfn` 里给吃方加 buff（最常见料理写法）

```lua
oneatenfn = function(inst, eater)
    eater:AddDebuff("my_buff", "my_buff")
end
```

- jellybean（豆糖）、sweettea（甜茶）、moqueca、buff_playerabsorption（大蒜料理）全都是这个模板。

#### 7.5 在 `oneatenfn` 里给周围生物群发状态（mandrake）

```47:52:scripts/prefabs/mandrake_inactive.lua
local function oneaten_raw(inst, eater)
    eater.SoundEmitter:PlaySound("dontstarve/creatures/mandrake/death")
    eater:DoTaskInTime(0.5, function()
        doareasleep(eater, TUNING.MANDRAKE_SLEEP_RANGE, TUNING.MANDRAKE_SLEEP_TIME)
    end)
end
```

【老手】注意 `DoTaskInTime(0.5, ...)` —— **延迟 0.5 秒再范围睡眠**。这是有原因的：

- 此刻 `Edible:OnEaten` 还没结束，立刻 doareasleep 会让吃方自己也立刻 sleep；
- 但 sleep 状态机会打断当前 eat 状态（busy 被 sleep 状态打破）；
- 结果你看到的可能是"咬一口然后倒地睡着，但吃了没"——错误。
- 延迟 0.5 秒后，eat 动画已经基本结束，进入 idle 再 sleep，看起来就是"吃完再倒地"。

这是 oneatenfn 必须懂的**时序保护技巧**。

#### 7.6 在 `oneatenfn` 里触发吃方"特殊状态"（药水）

```13:17:scripts/prefabs/ipecacsyrup.lua
local function syrup_OnEaten(inst, eater)
    if eater.sg ~= nil and eater.sg:HasState("ipecacpoop") then
        eater:AddDebuff("ipecacsyrup_buff", "ipecacsyrup_buff")
    end
end
```

【进阶】检查 `sg:HasState("ipecacpoop")` 是为了"只对有这个状态机的角色生效"——某些 mod 角色可能没有这个特殊状态。给 buff 后，**buff 内部自己**会推 `queue_post_eat_state` 让玩家进入 ipecacpoop 状态。

#### 7.7 在 `oneatenfn` 里给吃方挂"光环"（萤光蠕虫）

```52:54:scripts/prefabs/wormlight.lua
local function item_oneaten(inst, eater)
    create_light(eater, "wormlight_light")
end
```

`create_light` 会给 eater 加一个挂载灯光实体，持续若干秒。**注意**：因为 oneatenfn 早于 HandleEatRemove，**food 还在场景里**——但你**不能**在 eater 身上挂 food 做光源，因为 food 马上被 Remove。所以这里 `create_light` 内部 spawn 一个独立的灯 prefab 挂在 eater 上，**与 food 解耦**。

#### 7.8 在 `oneatenfn` 里让 eater 说话（裂蛋）

```146:150:scripts/prefabs/tallbirdegg.lua
local function OnEaten(inst, eater)
    if eater.components.talker ~= nil then
        eater.components.talker:Say( GetString(eater, "EAT_FOOD", "TALLBIRDEGG_CRACKED") )
    end
end
```

【新手】**最简短的"吃后台词"模板**。`GetString(eater, "EAT_FOOD", "TALLBIRDEGG_CRACKED")` 会查角色的 `STRINGS.CHARACTERS.<NAME>.EAT_FOOD.TALLBIRDEGG_CRACKED`，未覆写时回落到 `WILSON`（默认）。

#### 7.9 `oneat` 监听用于评论（wisecracker 台词）

```8:56:scripts/components/wisecracker.lua
    inst:ListenForEvent("oneat",
        function(inst, data)
            if data.food ~= nil and data.food.components.edible ~= nil then
                if data.food.prefab == "spoiled_food" then
                    inst.components.talker:Say(GetString(inst, "ANNOUNCE_EAT", "SPOILED"))
                elseif data.food.components.edible:GetHealth(inst) < 0 and
                    data.food.components.edible:GetSanity(inst) <= 0 and
                    not (inst.components.eater ~= nil and (
                            inst.components.eater.strongstomach and
                            data.food:HasTag("monstermeat") or
                            inst.components.eater.healthabsorption == 0
                        )) and not (inst.components.foodaffinity and inst.components.foodaffinity:HasPrefabAffinity(data.food)) then

                    inst.components.talker:Say(GetString(inst, "ANNOUNCE_EAT", "PAINFUL"))

                elseif data.food.components.perishable ~= nil then
                    if data.food.components.perishable:IsFresh() then
                        local ismasterchef = inst:HasTag("masterchef")
                        if ismasterchef and data.food.prefab == "wetgoop" then
                            inst.components.talker:Say(GetString(inst, "ANNOUNCE_EAT", "PAINFUL"))
                        else
                            local count = inst.components.foodmemory ~= nil and inst.components.foodmemory:GetMemoryCount(data.food.prefab) or 0
                            if count > 0 then
                                inst.components.talker:Say(GetString(inst, "ANNOUNCE_EAT", "SAME_OLD_"..tostring(math.min(5, count))))
                            elseif ismasterchef then
                                ...
```

【进阶】**6 类台词分流**：
1. spoiled_food → SPOILED；
2. 血/精神双负且非强胃非素食 → PAINFUL；
3. fresh 食物 + 吃腻 → SAME_OLD_1..5；
4. fresh + masterchef 评价（TASTY/PREPARED/RAW/DRIED/COOKED）；
5. stale → STALE；
6. spoiled（perishable 视角）→ SPOILED。

【老手】**注意 PAINFUL 的反向守门**：

```13:19:scripts/components/wisecracker.lua
                elseif data.food.components.edible:GetHealth(inst) < 0 and
                    data.food.components.edible:GetSanity(inst) <= 0 and
                    not (inst.components.eater ~= nil and (
                            inst.components.eater.strongstomach and
                            data.food:HasTag("monstermeat") or
                            inst.components.eater.healthabsorption == 0
                        )) and not (inst.components.foodaffinity and inst.components.foodaffinity:HasPrefabAffinity(data.food)) then
```

`GetHealth/GetSanity` 都为负 + 不是强胃 + 不是素食 0 吸收 + 没亲和——多重否定让 Wormwood 吃榴莲不说"好痛"，pigman 吃 monstermeat 不说"好痛"。**这就是事件监听器为何要懂 Eater/Edible/Foodaffinity 三个组件的协作**。

#### 7.10 `oneat` 监听用于游戏机制（wereeater 狼变形）

见 3.3 后半节。

#### 7.11 `oneat` 监听用于统计（beefalometrics、crittertraits、carrat 教练）

【进阶】这些都是"周边系统"，本身不影响吃饭，只记录数据：
- `beefalometrics.lua`：统计被喂食次数，影响牛年活动结算。
- `crittertraits.lua`：吃东西 +1 wellfed，影响 critter 的成长性格（懒、贪吃等）。
- `carrat.lua`：YOTC carrat 训练评估。

#### 7.12 `oneat` 监听用于学习食谱（玩家）

```918:923:scripts/prefabs/player_common_extensions.lua
local function OnEat(inst, data)
	local product = (data ~= nil and data.food ~= nil and data.food:HasTag("preparedfood")) and (data.food.food_basename or data.food.prefab) or nil
	if product ~= nil then
		OnLearnCookbookStats(inst, product)
	end
end
```

【老手】**`food_basename`** 是 `preparedfoods.lua:111` 设的，香料料理的 basename 是不带香料后缀的原料理名（如 `kabobs`），保证你吃 `kabobs_garlic` 也能开 `kabobs` 食谱条目。这是个 mod 做"自定义料理 + 想被记入食谱"必须懂的字段。

#### 7.13 `oneaten` 监听用于鱼饵触发（bait）

见 4.3。

---

### 八、新手实战：给食物添加"吃下后"效果

#### 8.1 最简：吃下加 buff

```lua
local function oneaten(inst, eater)
    if eater.components.health ~= nil then
        eater:AddDebuff("my_food_buff", "my_food_buff")
    end
end

-- 在你的食物 prefab fn 里：
inst.components.edible:SetOnEatenFn(oneaten)
```

#### 8.2 简：吃下让吃方说台词

```lua
local function oneaten(inst, eater)
    if eater.components.talker ~= nil then
        eater.components.talker:Say(GetString(eater, "EAT_FOOD", "MY_SPECIAL_FOOD"))
    end
end

-- STRINGS 表里给每个可能吃的角色补 EAT_FOOD.MY_SPECIAL_FOOD
STRINGS.CHARACTERS.WILSON.EAT_FOOD.MY_SPECIAL_FOOD = "怪味道..."
STRINGS.CHARACTERS.WICKERBOTTOM.EAT_FOOD.MY_SPECIAL_FOOD = "嗯，确实独特。"
```

#### 8.3 中：吃下治疗周围

```lua
local function oneaten(inst, eater)
    local x, y, z = eater.Transform:GetWorldPosition()
    for _, v in ipairs(TheSim:FindEntities(x, y, z, 6, {"player"})) do
        if v.components.health ~= nil and not v.components.health:IsDead() then
            v.components.health:DoDelta(10)
        end
    end
end

inst.components.edible:SetOnEatenFn(oneaten)
```

【新手】**注意半径要合理**。`FindEntities` 用 must_tags 时把 player tag 当唯一过滤，**不要忘了排除自己**——如果想"群体不包括自己"，加 `if v ~= eater` 检查。

#### 8.4 中：mod 角色吃东西放屁

```lua
local function oneat(inst, food, feeder)
    if food.components.edible.foodtype == FOODTYPE.VEGGIE then
        SpawnPrefab("poop").Transform:SetPosition(inst.Transform:GetWorldPosition())
    end
end

-- master_postinit：
inst.components.eater:SetOnEatFn(oneat)
```

【新手】注意 `oneat` 在 `master_postinit` 里设——它是吃方的字段，应该在吃方 prefab 注册时挂。

#### 8.5 进：吃完后跳到自定义动画状态

```lua
-- mod 角色的 sg 文件里：
State{
    name = "my_post_eat",
    tags = { "busy", "nointerrupt" },
    onenter = function(inst)
        inst.AnimState:PlayAnimation("emote_loop")
        inst.SoundEmitter:PlaySound("dontstarve/wilson/emote_taunt")
    end,
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
}

-- 然后在角色 master_postinit：
inst:ListenForEvent("oneat", function(inst, data)
    if data.food and data.food.prefab == "my_special_food" then
        inst:PushEventImmediate("queue_post_eat_state", {
            post_eat_state = "my_post_eat",
            nointerrupt = true,
        })
    end
end)
```

【新手】**这是最复杂的实战**，但只要按这套模板写就稳。注意：

- `PushEventImmediate` 必不可少；
- `my_post_eat` 状态必须用 `AddStategraphState` 注入到 SGwilson；
- 不能太长，否则玩家"卡住吃饭"会反感。

---

### 九、进阶要点

#### 9.1 时序图复盘

```
T0  ACTIONS.EAT.fn() called
T1    eater:Eat(food, feeder)
T2      PrefersToEat ✓
T3      calc delta
T4      DoDelta (health/hunger/sanity)
T5      [feeder ≠ self.inst]
T6         PushEvent "feedincontainer" → feeder
T7         PushEvent "feedmount" → feeder
T8      PushEvent "oneat" → self.inst        ←─ wisecracker / wereeater / beefalometrics / digestion / 玩家 cookbook
T9      oneatfn(self, food, feeder)           ←─ angle / pigman / beefalo / eyemask / wx78 / Walter / woby
T10     food.components.edible:OnEaten(self)
T11       oneaten(food, self)                 ←─料理 buff / mandrake sleep / firenettle 毒 / wormlight
T12       temperature:SetTemperatureInBelly
T13       PushEvent "oneaten" → food          ←─ bait
T14       eatensound 播放
T15     food.components.edible:HandleEatRemove(eatwholestack)
T16       food 被 Remove / 减一堆
T17     lasteattime = GetTime()
T18     foodmemory:RememberFood(food.prefab)
T19   状态机 frame 30 解锁 busy（如果没排 queue_post_eat_state）
T20   状态机 frame 52 跳转 queued_post_eat_state（如果有）
T21   状态机 animqueueover 跳转 idle
```

【进阶】**注意 T8 ~ T18 在一帧内执行完**。T19 / T20 / T21 是后续帧的状态机进度。**整条吃饭流程在用户看来是 0.5~2 秒，但游戏逻辑上的全部副作用都集中在 T8 那一刻**。

#### 9.2 `PushEvent` vs `PushEventImmediate`

| 项 | `PushEvent` | `PushEventImmediate` |
| --- | --- | --- |
| 调用栈 | 事件被压入 buffer 队列 | 立刻同步遍历监听器 |
| 何时触发 | **下一帧**的事件分发循环 | **当下行**（同步调用） |
| 监听器返回 | 永不阻塞调用者 | 监听器的副作用立刻被看到 |
| 适合场景 | 一般事件、跨帧信号 | 必须立刻处理的同步信号（如 queue_post_eat_state、attackbutton） |

【老手】**滥用 `PushEventImmediate` 会让事件链变成"伪函数调用"** —— 监听器很容易意外修改外部状态，造成调试困难。**只在必须立刻处理时用**。`Eater:Eat` 内部用普通 `PushEvent`，是因为 oneat 的处理可以拖到下一帧也没问题（玩家不会注意到）；但 `queue_post_eat_state` 必须立即到达 statemem，否则错过窗口。

#### 9.3 `oneat` 监听器抢着改 `eater` 状态？

【老手】**oneat 是同步事件**，**多监听器顺序执行**。如果两个监听器都改 `inst.components.health:DoDelta`，**两个都生效**。如果都做 `inst:Remove()`——第二个会拿到一个无效 inst，理论上 `DoDelta` 抛 nil 错。但事件分发器有保护：监听器内部出错被 catch，不影响后续监听器。

【老手】**Mod 编写规则**：

- **不要**在 `oneat` 监听里 `inst:Remove()` —— 即使 catch 了，状态机后续帧操作 invalid inst 还是 crash。
- **不要**在 `oneat` 监听里 `RemoveComponent("eater")`，否则后续 `Edible:OnEaten` 会失败。
- **可以**在 `oneat` 监听里挂 buff、改属性、推自定义事件、`DoTaskInTime` 异步处理。

#### 9.4 食物的 `oneatensound`：与 `eatensound` 区分

【老手】`eatensound` 是**挂在 entity 上的 Lua 字段**（不是 Edible 组件字段、不是事件）：

```210:213:scripts/components/edible.lua
    self.inst:PushEvent("oneaten", { eater = eater })
    if self.inst.eatensound ~= nil and eater.SoundEmitter ~= nil then
        eater.SoundEmitter:PlaySound(self.inst.eatensound)
    end
```

【老手】**用法**：

```lua
inst.eatensound = "dontstarve/wilson/eat_meat"
```

注意这是 `inst.eatensound`，**不是** `inst.components.edible.eatensound`。这是 Klei 把"咀嚼声"放在 entity 顶层，避免每次开 Edible 都要查组件字段。Mod 注意写法：

```lua
inst.eatensound = "yourmod/sound/eat"
```

而不是误写成：

```lua
inst.components.edible.eatensound = "..."  -- 不生效！
```

#### 9.5 `oneatfn` 早注册 vs `SetOnEatFn` 后期切换

【老手】角色 prefab 默认 `master_postinit` 里 `SetOnEatFn(默认 OnEat)`，但**装备 / Buff / 技能树**可以**动态切换**：

```lua
-- 装备某眼罩时：
local prev_oneat = inst.components.eater.oneatfn
inst.components.eater:SetOnEatFn(function(inst, food, feeder)
    if prev_oneat then prev_oneat(inst, food, feeder) end
    -- 装备特殊效果
end)
inst.eyemask_prev_oneatfn = prev_oneat   -- 备份

-- 卸下装备时还原：
inst.components.eater:SetOnEatFn(inst.eyemask_prev_oneatfn)
inst.eyemask_prev_oneatfn = nil
```

【老手】**注意持有 reference 的方式**：不能直接 `inst.components.eater.oneatfn = prev_oneat`——这样跳过了 setter（虽然 oneatfn 没 setter，可读性差），写成 `SetOnEatFn(prev_oneat)` 更清晰。

---

### 十、老手深挖

#### 10.1 `oneatfn` 的隐式 contract：必须不能让 eater invalidate

【老手】回顾时序图，T9 `oneatfn` 之后 T10 `Edible:OnEaten(self)`。如果 T9 把 `self` 弄无效，T10 取 `self.components.something` 就抛错。

**`Edible:OnEaten` 内部对 `eater` 的访问**：

```201:213:scripts/components/edible.lua
    -- Food is an implicit heater/cooler if it has temperature
    if self.temperaturedelta ~= 0 and
        self.temperatureduration ~= 0 and
        self.chill < 1 and
        eater ~= nil and
        eater.components.temperature ~= nil then
        eater.components.temperature:SetTemperatureInBelly(self.temperaturedelta * (1 - self.chill) * delta_multiplier, self.temperatureduration * duration_multiplier)
    end

    self.inst:PushEvent("oneaten", { eater = eater })
    if self.inst.eatensound ~= nil and eater.SoundEmitter ~= nil then
        eater.SoundEmitter:PlaySound(self.inst.eatensound)
    end
```

它**对 eater 做了 3 次访问**：

1. `eater.components.temperature`；
2. `oneaten` 事件 data 含 eater；
3. `eater.SoundEmitter`。

`eater ~= nil` 检查只挡 `nil`，**不挡 invalid entity**。如果 oneatfn 把 eater Remove 了，eater ~= nil 但 `eater.components` 是 nil（Lua 行为：被 Remove 的 entity 仍是一个 Lua table，但 components 索引返 nil）。**实际效果是各种 nil 索引错误**。

**结论**：**永远不要在 oneatfn 里 Remove eater**。

#### 10.2 `oneaten` 的隐式 contract：可以 Remove food 但要注意 IsValid

【老手】T11 `oneaten(food, eater)` 之后 T12-T16 各种操作都依赖 food。但 `Edible:HandleEatRemove` 已经做了保护：

```216:226:scripts/components/edible.lua
function Edible:HandleEatRemove(eatwholestack) -- Called from eater.lua, internal, don't touch elsewhere!
    if self.inst:IsValid() then --might get removed in OnEaten...
        if self.handleremovefn ~= nil then
            self.handleremovefn(self.inst, eatwholestack)
        elseif not eatwholestack and self.inst.components.stackable ~= nil then
            self.inst.components.stackable:Get():Remove()
        else
            self.inst:Remove()
        end
    end
end
```

注释 `--might get removed in OnEaten...` 明确告诉我们：**Klei 预期 oneaten 字段可能会 Remove food**。这是合法用法，但 **T12 `SetTemperatureInBelly`、T13 推事件、T14 `eatensound` 之前 food 还活着**。

【老手】**所以 mod 在 `oneaten` 字段里 Remove food 的位置很关键**：

- 调用 `eater.components.temperature:SetTemperatureInBelly(food.temperaturedelta, ...)` ——会失败因为 food 已经 invalid，但反正 `Edible:OnEaten` 已经传 self.inst 进去了，访问 self.inst 还能算（self 是 Edible 的，self.inst 是 food，被 Remove 后部分访问失效）。
- 推 `oneaten` 事件 → 监听器（如 bait）拿到的 `data.eater` 还有效，但 `data.eater` 可能不能再访问 food 字段。
- `inst.eatensound` 是 entity 字段，被 Remove 后还能读但播 sound 也无意义。

**安全做法**：在 oneaten 字段里 **不要 Remove food**，让 `HandleEatRemove` 自动处理。如果一定要 Remove（如吃完变 spoiled_food），用 `SetHandleRemoveFn` 改写 HandleEatRemove，**而不是** 直接 `food:Remove()`。

#### 10.3 跨网络同步的隐患

【老手】所有 Eater / Edible 的事件、回调都**只在服务端**触发——客户端的 oneat 事件**不存在**。所以 mod 如果在 `oneat` 监听里改 client 的 UI、播 client 特有音效，**得通过 player_classified 转发**。

【老手】查 `scripts/prefabs/player_classified.lua`：

```1174:1174:scripts/prefabs/player_classified.lua
    inst:ListenForEvent("feedincontainer", OnFeedInContainer, inst._parent)
```

`feedincontainer` 是少数 Klei 给 classified 转发的进食事件。如果 mod 要"客户端动画反馈"，需要：

1. 服务端 `oneat` 推一个 net_event（如 `RPC`），
2. 客户端监听 player_classified 的对应 net_event，
3. 客户端这边播 UI 动画。

直接监听客户端的 `oneat`——无效。

#### 10.4 多 mod 链式 `oneatenfn` 的脏话题

【老手】两个 mod 都 hook 同一份食物的 `oneatenfn`：

**Mod A**（先加载）：

```lua
local oldfn = inst.components.edible.oneaten
inst.components.edible:SetOnEatenFn(function(inst, eater)
    if oldfn then oldfn(inst, eater) end
    -- A 的效果
end)
```

**Mod B**（后加载）：

```lua
local oldfn = inst.components.edible.oneaten
inst.components.edible:SetOnEatenFn(function(inst, eater)
    if oldfn then oldfn(inst, eater) end
    -- B 的效果
end)
```

链式执行顺序：B 先跑 → 调用 oldfn（= A 的包装）→ A 跑 → 调用 oldfn（= 原始）→ 原始跑。**所以是 A→B 顺序**（注册顺序的反向）。

【老手】**问题**：如果第二个 mod 写错成：

```lua
inst.components.edible:SetOnEatenFn(function(inst, eater)
    -- B 的效果，没调 oldfn
end)
```

那 Mod A 的效果就丢了，**且无任何报错**。这是 mod 兼容性的常见坑。**最佳实践是 mod 用事件 ListenForEvent("oneaten")**——多个监听器不互相覆盖。

#### 10.5 `oneat` 监听里调用 `eater:Eat` 的递归地狱

【老手】**禁止**：

```lua
inst:ListenForEvent("oneat", function(inst, data)
    -- 错误！会再次进入 eater:Eat，无限递归
    inst.components.eater:Eat(some_other_food)
end)
```

**正确**（异步化）：

```lua
inst:ListenForEvent("oneat", function(inst, data)
    inst:DoTaskInTime(0.5, function()
        inst.components.eater:Eat(some_other_food)
    end)
end)
```

【老手】也禁止在 oneat 监听里直接 `food.components.edible:OnEaten`——同样无限递归。

---

### 十一、本节涉及的标签与事件（速查）

#### 标签清单（本节新涉及，未在 20.1/20.2 列出的）

| 标签 | 含义 | 谁打 / 谁读 |
| --- | --- | --- |
| `preparedfood` | 这是一份已烹饪料理 | preparedfoods fn 自动打 / Warly preferseatingtags 读、player_common_extensions OnEat 读 |
| `pre-preparedfood` | 这是一份半成品料理（Warly 专用） | 同上 / Warly preferseatingtags 读 |
| `fooddrink` | 这是饮料类（喝而不是嚼） | hermitcrabtea 等 prefab / ACTIONS.EAT.strfn 读、SGwilson `eat` state 内分流读 |
| `sloweat` | 强制走 `eat`（长动画）状态 | mod 食物可以打 / SGwilson ACTIONS.EAT handler 读 |
| `quickeat` | 强制走 `quickeat`（短动画）状态 | 同上 |
| `pet_treat` | 是宠物专属零食 | treat 类 prefab 自动打 / Woby OnEat 读、宠物 brain 读 |
| `monstermeat` | 这是怪物肉类食物 | meats/monstermeat 等 prefab 自动打 / wereeater / wisecracker / DoFoodEffects 读 |
| `wereitem` | 强制狼变形食物 | hounded_were 等 / wereeater EatMosterFood 读 |

#### 事件清单（本节核心）

**1. `oneat`**

- **含义**：实体吃下了一份食物。
- **推送时机**：`Eater:Eat` 中段，**delta 已应用、food 仍有效、Edible:OnEaten 还未调用**。
- **推送数据**：
  - `food`（Entity）：吃下的食物。
  - `feeder`（Entity）：投喂者（自己吃自己时 == eater）。
- **推送对象**：吃方（eater）。
- **典型监听者**：玩家 cookbook、wisecracker、wereeater、beefalometrics、crittertraits、各 NPC 的 OnEat。

**2. `oneaten`**

- **含义**：食物 prefab 被吃下了。
- **推送时机**：`Edible:OnEaten` 中段，**oneaten 字段已执行、温度已应用、HandleEatRemove 还未调用**。
- **推送数据**：
  - `eater`（Entity）：吃下这份食物的实体。
- **推送对象**：食物本身。
- **典型监听者**：bait 鱼饵、YOTR 兔人 brain。

**3. `feedincontainer`**

- **含义**：feeder 通过容器把食物喂给了 self.inst。
- **推送时机**：`Eater:Eat` 中段，feeder ≠ self.inst 且 eater 是 inventoryitem 且 grandowner == feeder（或 grandowner.container 被 feeder 打开）。
- **推送数据**：无。
- **推送对象**：feeder。
- **典型监听者**：player_classified 用于客户端反馈。

**4. `feedmount`**

- **含义**：feeder 把食物喂给了坐骑。
- **推送时机**：`Eater:Eat` 中段，feeder ≠ self.inst 且 eater 是 rideable 且 feeder 是当前 rider。
- **推送数据**：
  - `food`（Entity）：投喂的食物。
  - `eater`（Entity）：坐骑（即 self.inst）。
- **推送对象**：feeder（rider）。
- **典型监听者**：SGwilson 全局 EventHandler → 进入 mount_eat 状态。

**5. `wonteatfood`**

- **含义**：尝试吃但 PrefersToEat 不通过。
- **推送时机**：`ACTIONS.EAT` 在 SGwilson ActionHandler 中检测到偏好不过；`ACTIONS.FEEDPLAYER.fn` 内部检测到偏好不过。
- **推送数据**：
  - `food`（Entity）：被拒绝的食物。
- **推送对象**：吃方。
- **典型监听者**：SGwilson 全局 EventHandler → 进入 refuseeat 状态；player_common OnWontEatFood → 喷 YUCKY 台词。

**6. `queue_post_eat_state`**

- **含义**：告诉吃方状态机"吃完后跳到指定状态"。
- **推送时机**：通常在 `oneat` 或 `oneatfn` 链上立刻 PushEventImmediate。
- **推送数据**：
  - `post_eat_state`（string）：目标状态名。
  - `nointerrupt`（bool，可选）：是否锁定不允许打断。
- **推送对象**：吃方（玩家）。
- **典型监听者**：SGwilson 的 eat / quickeat 状态内部 EventHandler。

**7. `animqueueover`**（状态机层，非 Eater/Edible 推送）

- **含义**：当前动画队列播放完毕。
- **推送时机**：AnimState 内部，每次 PushAnimation 队列耗尽时。
- **推送数据**：无。
- **推送对象**：动画所属 entity。
- **典型监听者**：SGwilson 的 eat / quickeat 状态用它收尾跳转 idle 或 queued_post_eat_state。

#### 玩家自定义二级事件（被 `oneat` / `oneatfn` 间接触发）

**8. `treatwoby`**

- **含义**：玩家投喂了 pet_treat 给 Woby。
- **推送时机**：Woby 的 oneatfn 内部，`feeder:PushEvent("treatwoby", inst)`。
- **推送数据**：`woby`（即 Woby 实体）。
- **推送对象**：feeder（Walter）。
- **典型监听者**：Walter 的 wisecracker → ANNOUNCE_WOBY_PRAISE 台词。

**9. `eat`**（beefalo 专属）

- **含义**：beefalo 吃了东西。
- **推送时机**：beefalo OnEat 内部主动推。
- **推送数据**：`full`（bool）、`food`（Entity）。
- **推送对象**：beefalo 自己。
- **典型监听者**：beefalo brain / SGBeefalo 内部某些分支。

---

### 十二、常见坑与最佳实践（Checklist）

【新手】

- [ ] `SetOnEatFn` 后注册者**覆盖**先注册者；用之前先备份 `eater.oneatfn`。
- [ ] `SetOnEatenFn` 后注册者**覆盖**先注册者；同上。
- [ ] `oneatfn` 在 `oneat` 事件**之后**才被调用，反直觉但要记住。
- [ ] 自定义料理用 `preparedfoods.lua` 的 `oneatenfn` 字段挂回调，不要碰 Edible 源码。
- [ ] 食物 prefab 的咀嚼声用 `inst.eatensound = "..."`，**不是** `inst.components.edible.eatensound`。

【进阶】

- [ ] `oneat` 事件**多监听不互相覆盖**，是 mod 的首选 hook 点。
- [ ] `oneaten` 事件少有人监听，但是给"食物外挂组件"（如鱼饵）的天然位置。
- [ ] 在 `oneatfn` / `oneat` / `oneatenfn` / `oneaten` 里**不要** Remove eater / food，让 Klei 的兜底处理（IsValid 检查）来做。
- [ ] 想做"吃完进入特殊状态"，**必须** `PushEventImmediate("queue_post_eat_state", ...)`，普通 PushEvent 会错过 statemem 记录窗口。
- [ ] 监听 `wonteatfood` 在状态机里走 `refuseeat`，自动有"摇头吐"动画。
- [ ] 喂坐骑系统：feeder 收 `feedmount`，eater 收 `oneat`——双方监听点不同。

【老手】

- [ ] 跨网络场景下 `oneat` / `oneaten` 只在服务端触发，客户端 UI 需经 player_classified 网络转发。
- [ ] 多 mod 链式 hook `oneatfn` 的反向执行顺序（B → A → 原始）——用事件 ListenForEvent 才能避免覆盖冲突。
- [ ] `Edible:OnEaten` 内的事件分发器 catch 监听器异常，**单个监听器报错不会中断后续监听器**。但 mod 应避免错误传播，记得 `assert(inst:IsValid())` 自防御。
- [ ] `Edible:HandleEatRemove` 注释明确 `--might get removed in OnEaten...`——Klei 预期 oneaten 字段会 Remove food，但你应该用 `SetHandleRemoveFn` 来定义"吃完变 X 物品"，而不是直接 `food:Remove()`。
- [ ] `oneat` 监听里调用 `eater:Eat` 会**无限递归**，必须 `DoTaskInTime` 异步化。
- [ ] 同一帧内 `T8` `oneat` 事件、`T9` `oneatfn`、`T11` `oneaten`、`T13` `oneaten` 事件**全部同步执行**，整条进食流程在玩家看来 0.5~2 秒，但代码上的所有副作用集中在 1 帧。

---

至此 20.3 完结。下一节进入 Perishable 组件，会把"食物如何随时间腐烂、与 Edible 的耦合细节、各种 ignoresspoilage / allspoiledprocessor 的判定路径"全部拆解。


## 20.4 Perishable 组件——食物腐烂与保鲜

（待编写）

## 20.5 PreparedFoods——烹饪系统的数据驱动设计

（待编写）

## 20.6 烹饪算法详解：cooking.lua 的配方匹配、优先级排序与 test 判定链

（待编写）

## 20.7 Cookable / Dryer / Stewer——各种烹饪方式的实现

（待编写）

## 20.8 Warly 专属料理系统——独家配方与食物记忆

（待编写）

## 20.9 实战：添加自定义食物与烹饪配方

（待编写）
