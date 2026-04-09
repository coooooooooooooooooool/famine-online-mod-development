# 第1章 Lua 语言速成

## 1.1 变量与数据类型

Lua 是一门轻量级的脚本语言，饥荒联机版的整个游戏逻辑层都是用 Lua 写的。在开始 Mod 开发之前，你需要掌握 Lua 的基本数据类型——它们是你与游戏交互的基本工具。

### 1.1.1 Lua 的八种基本类型

Lua 一共有八种基本数据类型：

| 类型 | 说明 | 饥荒中的典型用途 |
|------|------|------------------|
| `nil` | 空值，表示"不存在" | 检测组件/对象是否存在 |
| `boolean` | 布尔值，`true` 或 `false` | 开关、条件判断 |
| `number` | 数字（整数和浮点数共用） | 血量、伤害、时间等数值 |
| `string` | 字符串 | prefab 名、tag 名、动画名 |
| `table` | 表（万能容器） | 配置、列表、对象——饥荒中无处不在 |
| `function` | 函数 | 回调、事件监听 |
| `userdata` | C 层暴露的对象 | 引擎底层对象（Entity 等） |
| `thread` | 协程 | 饥荒中的 `StartThread` |

在饥荒 Mod 开发中，你最常打交道的是前六种。可以使用 `type()` 函数来查看一个值的类型：

```lua
print(type(nil))        -- nil
print(type(true))       -- boolean
print(type(42))         -- number
print(type("axe"))      -- string
print(type({}))         -- table
print(type(print))      -- function
```

### 1.1.2 nil——"不存在"的哲学

在 Lua 中，`nil` 代表"没有值"或"不存在"。任何未赋值的变量默认就是 `nil`。

这在饥荒代码里特别重要，因为**判断一个东西存不存在**是最频繁的操作之一。比如：

```lua
-- 检查一个实体是否有某个组件
if inst.components.health ~= nil then
    -- 有生命值组件，说明这个东西可以被打
    inst.components.health:DoDelta(-10)
end
```

为什么要这样检查？因为饥荒里不是所有实体都有相同的组件——一块石头没有 `health` 组件（不可攻击），但猪人有。如果你不检查就直接访问，游戏就会报错崩溃。

还有一个常见用法——**把变量设为 `nil` 来"删除"它**：

```lua
local task = inst:DoTaskInTime(5, some_function)
-- 之后想取消这个定时任务
if task ~= nil then
    task:Cancel()
    task = nil  -- 清除引用，让 Lua 可以回收内存
end
```

> **小贴士**：在 Lua 中，`nil` 和 `false` 都被视为"假"，其他所有值（包括 `0` 和空字符串 `""`）都是"真"。这和很多语言不一样！

```lua
if 0 then print("0 是真的！") end           -- 会打印，因为 0 在 Lua 中是真
if "" then print("空字符串也是真的！") end    -- 会打印
if nil then print("假") end            -- 不会打印
if false then print("假") end          -- 不会打印
```

### 1.1.3 boolean——开关与条件

`boolean` 类型只有两个值：`true` 和 `false`。在饥荒中，它常用来表示各种开关状态。

来看一个饥荒源码中的实际例子——孵蛋组件（`hatchable`）：

```lua
-- 来自 scripts/components/hatchable.lua
self.heater_prefs = {
    day = false,     -- 白天不需要加热
    dusk = nil,      -- 黄昏无所谓（nil 表示不关心）
    night = true,    -- 夜晚需要加热
}
```

注意这里 `false` 和 `nil` 的区别：`false` 明确表示"不需要"，而 `nil` 表示"不关心/没有设定"。这种语义上的区分在饥荒代码里非常常见。

### 1.1.4 number——饥荒的数值世界

Lua 5.1（饥荒使用的版本）中，所有数字都是双精度浮点数，没有整数和浮点数的区分。

饥荒中最集中的数值定义在 `tuning.lua` 文件里，这个文件掌控着整个游戏的数值平衡：

```lua
-- 来自 scripts/tuning.lua
local seg_time = 30               -- 一个时间片段 = 30秒
local total_day_time = seg_time * 16   -- 一整天 = 480秒（8分钟）

local day_segs = 10    -- 白天占 10 个片段
local dusk_segs = 4    -- 黄昏占 4 个片段
local night_segs = 2   -- 夜晚占 2 个片段

local wilson_attack = 34   -- 威尔逊的基础攻击力
local wilson_health = 150  -- 威尔逊的生命上限
local wilson_hunger = 150  -- 威尔逊的饥饿上限
local wilson_sanity = 200  -- 威尔逊的理智上限
```

这些数值定义了饥荒的核心体验。做 Mod 的时候，你经常需要引用或修改这些数值，比如：

```lua
-- 在你的 Mod 中修改斧头的耐久度
TUNING.AXE_USES = 200  -- 原版是 100 次
```

数字还支持科学计数法和一些常用运算：

```lua
local big_number = 1e6       -- 1000000
local remainder = 10 % 3     -- 1（取模运算）
local power = 2 ^ 10         -- 1024（幂运算）

-- 饥荒中常见的随机数用法
local random_delay = 5 + math.random() * 10  -- 5 到 15 之间的随机数
```

### 1.1.5 string——名字就是一切

字符串在饥荒中的地位极其重要，因为饥荒的底层用字符串来标识几乎所有东西：prefab 名、tag 名、动画名、事件名等。

```lua
-- prefab 名是字符串
local axe = SpawnPrefab("axe")

-- tag 也是字符串
inst:AddTag("sharp")
inst:AddTag("weapon")

-- 动画名也是字符串
inst.AnimState:SetBank("axe")
inst.AnimState:SetBuild("axe")
inst.AnimState:PlayAnimation("idle")

-- 事件名还是字符串
inst:PushEvent("equipskinneditem", inst:GetSkinName())
```

Lua 中字符串可以用双引号 `"..."` 或单引号 `'...'`，效果完全一样：

```lua
local name1 = "wilson"
local name2 = 'wilson'
-- name1 和 name2 完全相同
```

多行字符串可以用双方括号 `[[...]]`：

```lua
local long_text = [[
这是第一行
这是第二行
可以随便写，不用转义
]]
```

**常用的字符串操作**：

```lua
-- 拼接用 ..
local full_name = "swap_" .. "axe"        -- "swap_axe"

-- 获取长度用 #
local len = #"hello"                       -- 5

-- 格式化用 string.format
local msg = string.format("血量: %d/%d", 80, 150)  -- "血量: 80/150"

-- 饥荒源码中的实际用法（来自 hatchable.lua）
local debug_str = string.format(
    "state: %s, progress: %2.2f/%2.2f",
    self.state, self.progress, self.cracktime
)
```

### 1.1.6 变量的作用域——local 的重要性

Lua 中变量分为**全局变量**和**局部变量**：

```lua
-- 全局变量：直接赋值，不加 local
my_global = "所有地方都能访问"

-- 局部变量：加 local 关键字
local my_local = "只在当前作用域内有效"
```

**在饥荒 Mod 开发中，几乎永远应该使用 `local`**。原因很现实：

1. **避免和其他 Mod 冲突**——全局变量是所有 Mod 共享的，如果两个 Mod 都定义了同名的全局变量，后加载的会覆盖先加载的
2. **性能更好**——Lua 访问局部变量比全局变量快得多
3. **不会污染命名空间**——代码更干净

来看一个饥荒源码中的典型例子（来自 `prefabs/axe.lua`）：

```lua
-- 资源列表：局部变量，只在这个文件里用
local assets = {
    Asset("ANIM", "anim/axe.zip"),
    Asset("ANIM", "anim/swap_axe.zip"),
}

-- 函数也定义为局部的
local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_object", "swap_axe", "swap_axe")
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end
```

但有些东西必须是全局的，比如 `TUNING` 表——因为所有文件都需要访问它：

```lua
-- tuning.lua 中
TUNING = {}   -- 全局变量，故意不加 local
```

**作用域的规则**是：`local` 变量在其声明的**代码块**（`do...end`、函数体、`if...then...end` 等）内有效：

```lua
local a = 1

if true then
    local b = 2      -- b 只在 if 块内有效
    print(a, b)       -- 1, 2
end

print(a)              -- 1
print(b)              -- nil，b 已经超出作用域了
```

### 1.1.7 多重赋值与交换

Lua 支持一次给多个变量赋值：

```lua
local x, y, z = 1, 2, 3

-- 多余的值会被丢弃
local a, b = 1, 2, 3    -- 3 被丢弃

-- 不够的变量会得到 nil
local c, d, e = 1, 2    -- e 是 nil
```

这个特性有个很实用的用法——**交换两个变量的值**：

```lua
local a, b = 1, 2
a, b = b, a    -- 现在 a=2, b=1，不需要临时变量
```

在饥荒中，函数经常返回多个值：

```lua
-- Transform:GetWorldPosition() 返回 x, y, z 三个坐标
local x, y, z = inst.Transform:GetWorldPosition()
```

### 1.1.8 类型转换

Lua 会在某些情况下自动转换类型：

```lua
-- 数字和字符串的自动转换
print("100" + 1)          -- 101（字符串 "100" 被自动转为数字）
print(100 .. " points")   -- "100 points"（数字被自动转为字符串进行拼接）
```

不过依赖自动转换是不好的习惯，推荐显式转换：

```lua
-- 显式转换
local num = tonumber("42")     -- 字符串转数字
local str = tostring(42)       -- 数字转字符串
```

---

> **本节小结**
> - Lua 有八种基本类型，Mod 开发中最常用的是 `nil`、`boolean`、`number`、`string`、`table` 和 `function`
> - `nil` 既表示"不存在"，也用于清除引用；在条件判断中，只有 `nil` 和 `false` 为假
> - 数值全部是浮点数，游戏的核心数值集中在 `tuning.lua`
> - 字符串是饥荒的"万能标识符"——prefab、tag、动画、事件都靠它
> - 始终使用 `local` 声明变量，避免全局污染





## 1.2 Table——Lua 的万能数据结构

Table（表）是 Lua 中**唯一**的复合数据结构。数组、字典、列表、集合、对象……在 Lua 里全部用 table 实现。可以毫不夸张地说，**如果你理解了 table，你就理解了半个饥荒**。

### 1.2.1 创建 table

用花括号 `{}` 即可创建一个 table：

```lua
local empty = {}               -- 空表
local list = {1, 2, 3}        -- 像数组的表
local dict = {name = "axe", damage = 27}  -- 像字典的表
```

### 1.2.2 table 当数组用

当 table 的键是**连续的整数**（从 1 开始）时，它就像数组一样：

```lua
-- 来自 prefabs/axe.lua —— 一个典型的资源列表
local assets = {
    Asset("ANIM", "anim/axe.zip"),
    Asset("ANIM", "anim/swap_axe.zip"),
}
```

这里 `assets[1]` 是第一个 Asset，`assets[2]` 是第二个。注意 **Lua 的数组下标从 1 开始**，不是 0！

**常用操作**：

```lua
local items = {"log", "rocks", "cutgrass"}

-- 获取长度
print(#items)                  -- 3

-- 访问元素
print(items[1])                -- "log"
print(items[3])                -- "cutgrass"

-- 添加元素到末尾
table.insert(items, "twigs")   -- items 现在有 4 个元素

-- 删除指定位置的元素
table.remove(items, 2)         -- 删除 "rocks"，后面的元素自动前移

-- 遍历数组——用 ipairs
for i, item in ipairs(items) do
    print(i, item)
end
```

来看一个饥荒中用数组存储食谱卡牌素材的例子：

```lua
-- 来自 preparedfoods.lua
card_def = {
    ingredients = {{"butterflywings", 1}, {"carrot", 2}, {"berries", 1}}
},
```

这是一个"数组套数组"的嵌套结构——外层数组的每个元素是一个内层数组 `{材料名, 数量}`。

### 1.2.3 table 当字典（键值对）用

当 table 的键是字符串时，它就像字典/哈希表：

```lua
-- 来自 preparedfoods.lua —— 蝴蝶松饼的食谱
butterflymuffin = {
    foodtype = FOODTYPE.VEGGIE,       -- 食物类型：蔬菜
    health = TUNING.HEALING_MED,      -- 回血量
    hunger = TUNING.CALORIES_LARGE,   -- 回饱食度
    perishtime = TUNING.PERISH_SLOW,  -- 保质期
    sanity = TUNING.SANITY_TINY,      -- 回理智
    cooktime = 2,                      -- 烹饪时间
    priority = 1,                      -- 优先级
}
```

这就是饥荒食谱系统的核心结构——每道菜就是一个 table，里面用键值对描述所有属性。

**两种访问方式**：

```lua
local food = {name = "meatballs", hunger = 62.5}

-- 方式一：点号访问（键是合法标识符时用）
print(food.name)          -- "meatballs"
print(food.hunger)        -- 62.5

-- 方式二：方括号访问（更通用，键可以是任意值）
print(food["name"])       -- "meatballs"

-- 方括号的好处：键可以是变量
local key = "hunger"
print(food[key])          -- 62.5
```

**遍历字典——用 `pairs`**：

```lua
-- 来自 spicedfoods.lua
local SPICES = {
    SPICE_GARLIC = { oneatenfn = oneaten_garlic },
    SPICE_SUGAR  = { oneatenfn = oneaten_sugar },
    SPICE_CHILI  = { oneatenfn = oneaten_chili },
    SPICE_SALT   = {},
}

-- pairs 遍历所有键值对（顺序不确定）
for spicename, spicedata in pairs(SPICES) do
    print(spicename)  -- "SPICE_GARLIC", "SPICE_SUGAR" 等
end
```

> **ipairs vs pairs**：`ipairs` 只遍历连续整数键（数组部分），遇到 `nil` 就停；`pairs` 遍历所有键值对。选错了会漏掉数据。

### 1.2.4 table 当集合（Set）用

在饥荒中，经常需要判断某个值是否在一个集合里。比如"这把斧头对哪些类型的敌人有加成？"Lua 没有内置 Set，但可以用 table 的键来模拟：

```lua
-- 来自 prefabs/axe.lua —— 月光石斧对这些 tag 的敌人有额外磨损
target:HasAnyTag("shadow", "shadowminion", "shadowchesspiece", "stalker", "stalkerminion", "shadowthrall")
```

如果你自己要做集合判断，最高效的方式是：

```lua
-- 把值作为键，值设为 true
local SHADOW_TAGS = {
    shadow = true,
    shadowminion = true,
    shadowchesspiece = true,
    stalker = true,
}

-- 查找是 O(1) 的，非常快
if SHADOW_TAGS[target_tag] then
    print("这是暗影生物！")
end
```

对比"把值放在数组里然后遍历查找"的方式（O(n)），这种写法在性能上好得多。饥荒源码中到处都在用这种模式：

```lua
-- 来自 recipe.lua —— 把 CHARACTER_INGREDIENT 的所有值变成集合
local is_character_ingredient = {}
for k, v in pairs(CHARACTER_INGREDIENT) do
    is_character_ingredient[v] = true
end

-- 之后就可以 O(1) 查找了
return is_character_ingredient[ingredienttype] == true
```

### 1.2.5 混合使用——数组 + 字典

table 可以同时拥有数组部分和字典部分，饥荒里经常这么做：

```lua
-- 浮水效果参数：前面是数组部分，后面是字典部分
local swap_data = {sym_build = "swap_glassaxe", bank = "glassaxe"}
-- 有些场景下可以把位置参数和命名参数混用
local floater = {"small", 0.05, {1.2, 0.75, 1.2}}
```

再看一个更典型的例子——漂浮参数：

```lua
-- 来自 preparedfoods.lua
floater = {"med", nil, 0.55},
-- floater[1] = "med"（大小）
-- floater[2] = nil（偏移，使用默认值）
-- floater[3] = 0.55（缩放）
```

### 1.2.6 嵌套 table——饥荒的数据骨架

饥荒的复杂配置几乎都靠嵌套 table 来组织。来看一个幻觉系统的配置（来自 `hallucinations.lua`）：

```lua
local HALLUCINATION_TYPES = {
    creepyeyes = {
        interval = 5,         -- 每 5 秒触发一次
        variance = 2.5,       -- 随机浮动 ±2.5 秒
        initial_variance = 20, -- 首次触发的随机延迟
        nightonly = true,      -- 只在夜晚出现
    },
    shadowwatcher = {
        interval = 30,
        variance = 15,
        initial_variance = 10,
        nightonly = true,
    },
    shadowskittish = {
        interval = 10,
        variance = 5,
        initial_variance = 20,
        nightonly = false,     -- 白天也会出现
    },
}
```

这是一个"字典套字典"的结构：外层键是幻觉类型名，内层键是各种参数。通过 `HALLUCINATION_TYPES.creepyeyes.interval` 就能拿到对应的数值。

### 1.2.7 常用的 table 操作函数

```lua
local t = {3, 1, 4, 1, 5, 9}

-- 插入
table.insert(t, 2)           -- 末尾追加 2
table.insert(t, 1, 0)        -- 在位置 1 插入 0

-- 删除
table.remove(t, 1)           -- 删除位置 1 的元素并返回它
table.remove(t)              -- 删除并返回最后一个元素

-- 排序
table.sort(t)                -- 升序排列
table.sort(t, function(a, b) return a > b end)  -- 降序

-- 拼接成字符串
local s = table.concat({"a", "b", "c"}, ", ")  -- "a, b, c"
```

饥荒还扩展了一些好用的 table 函数：

```lua
-- 检查值是否在数组中
table.contains(self.items, itemname)

-- 浅拷贝
local newdata = shallowcopy(olddata)
```

### 1.2.8 table 是引用类型

这一点非常重要，很多初学者会在这里踩坑：

```lua
local a = {hp = 100}
local b = a          -- b 和 a 指向同一个 table！

b.hp = 50
print(a.hp)          -- 50！a 也被改了！
```

`a` 和 `b` 并不是两份独立的数据，而是两个指向**同一个 table** 的引用。修改 `b` 就等于修改 `a`。

如果你需要一份独立的拷贝，要用 `shallowcopy`（饥荒提供的工具函数）：

```lua
local a = {hp = 100}
local b = shallowcopy(a)  -- b 是一份新的独立拷贝
b.hp = 50
print(a.hp)                -- 100，a 没有被影响
```

这也是为什么饥荒在生成调味食谱时会先拷贝一份原始数据：

```lua
-- 来自 spicedfoods.lua
local newdata = shallowcopy(fooddata)  -- 拷贝一份再修改，不影响原始食谱
newdata.cooktime = .12
newdata.spice = spicenameupper
```

### 1.2.9 table 作为函数参数

因为 table 是引用类型，传给函数时传的是引用，函数内部可以直接修改原始 table：

```lua
local function heal(entity_data, amount)
    entity_data.hp = entity_data.hp + amount  -- 直接修改了传入的 table
end

local player = {hp = 80, max_hp = 150}
heal(player, 20)
print(player.hp)  -- 100
```

这在饥荒的组件系统中随处可见——组件方法通过 `self`（也是一个 table）直接修改自身的数据。

---

> **本节小结**
> - Table 是 Lua 唯一的复合数据结构，身兼数组、字典、集合、对象等多重身份
> - 数组下标从 1 开始；遍历数组用 `ipairs`，遍历字典用 `pairs`
> - 用 `table[key] = true` 模式可以高效地模拟集合
> - 嵌套 table 是饥荒存储配置数据的标准方式（食谱、幻觉类型、漂浮物等）
> - Table 是引用类型，赋值只是复制引用，需要独立副本时要用 `shallowcopy`
> - 饥荒扩展了 `table.contains`、`shallowcopy` 等实用工具函数



## 1.3 函数与闭包

函数是 Lua 中的"一等公民"——它可以像数字、字符串一样被赋值给变量、存进 table、作为参数传递。在饥荒中，函数是连接各个系统的"胶水"：回调函数、事件监听、定时任务……到处都是函数。

### 1.3.1 函数的基本定义

Lua 中定义函数有两种等价的写法：

```lua
-- 写法一：常规写法
local function onhammered(inst, worker)
    inst:Remove()
end

-- 写法二：把匿名函数赋给变量
local onhammered = function(inst, worker)
    inst:Remove()
end
```

两种写法几乎等价，但有个微妙的区别：写法一的函数可以在函数体内递归调用自己，写法二的不行（因为赋值还没完成时函数体已经在执行了）。在饥荒代码中，两种写法都很常见。

### 1.3.2 参数与返回值

Lua 函数的参数和返回值都非常灵活：

```lua
-- 多参数
local function damage(target, amount, attacker)
    target.components.health:DoDelta(-amount)
end

-- 多返回值
local function get_position(inst)
    return inst.Transform:GetWorldPosition()  -- 返回 x, y, z 三个值
end

local x, y, z = get_position(some_entity)
```

**参数个数不匹配时**不会报错——多余的参数被丢弃，缺少的参数变成 `nil`：

```lua
local function greet(name, title)
    print(title, name)
end

greet("Wilson")              -- nil  Wilson（title 没传，是 nil）
greet("Wilson", "Dr.", "!")  -- Dr.  Wilson（"!" 被丢弃）
```

这个特性在饥荒中被广泛利用来做**可选参数**：

```lua
-- 来自 campfire.lua —— delay 参数是可选的
local function RepeatHallucination(hallucination, delay)
    hallucination.task = inst:DoTaskInTime(
        delay or (hallucination.params.interval + hallucination.params.variance * math.random()),
        hallucination.params.spawnfn,
        hallucination
    )
end
```

`delay or (...)` 的意思是：如果传了 `delay` 就用它，没传（`nil`）就用后面计算出来的默认值。

### 1.3.3 函数作为参数——回调模式

在饥荒中，最核心的编程模式之一就是**把函数作为参数传递给另一个函数**（回调）。来看营火（`campfire.lua`）的代码：

```lua
-- 定义回调函数
local function ontakefuel(inst)
    inst.SoundEmitter:PlaySound("dontstarve/common/fireAddFuel")
end

local function onupdatefueled(inst)
    if inst.components.burnable ~= nil and inst.components.fueled ~= nil then
        updatefuelrate(inst)
        inst.components.burnable:SetFXLevel(
            inst.components.fueled:GetCurrentSection(),
            inst.components.fueled:GetSectionPercent()
        )
    end
end

local function onfuelchange(newsection, oldsection, inst)
    if newsection <= 0 then
        inst.components.burnable:Extinguish()
        -- ...
    end
end

-- 把回调函数注册到燃料组件
inst.components.fueled:SetTakeFuelFn(ontakefuel)       -- 添加燃料时调用
inst.components.fueled:SetUpdateFn(onupdatefueled)     -- 每帧更新时调用
inst.components.fueled:SetSectionCallback(onfuelchange) -- 燃料档位变化时调用
```

这就是**事件驱动编程**的核心思路：你不直接调用这些函数，而是把它们"注册"到系统里，当特定事情发生时，系统会替你调用它们。开发者之所以这么设计，是因为营火的逻辑（加燃料、燃烧更新、熄灭）不需要每帧主动检查，只需要在对应事件发生时做出反应就够了。

### 1.3.4 匿名函数——一次性的回调

当回调函数很短、只用一次时，可以直接写匿名函数，不用单独取名：

```lua
-- 来自 hatchable.lua —— 延迟一段时间后重置状态
self.inst:DoTaskInTime(time, function()
    self.delay = false
end)
```

`DoTaskInTime` 接收两个参数：延迟时间和一个函数。这里的 `function() self.delay = false end` 就是匿名函数，它在 `time` 秒后被调用。

饥荒中匿名函数的典型使用场景：

```lua
-- 事件监听
inst:ListenForEvent("onextinguish", function(inst)
    if inst.components.fueled ~= nil then
        inst.components.fueled:InitializeFuelLevel(0)
    end
end)

-- 排序
table.sort(entities, function(a, b)
    return a.components.health:GetPercent() < b.components.health:GetPercent()
end)

-- 食谱的测试函数
test = function(cooker, names, tags)
    return names.froglegs and tags.veggie and tags.veggie >= 0.5
end,
```

### 1.3.5 闭包——函数"记住"了外部变量

闭包是 Lua 中一个强大的概念。简单说：**当一个函数引用了它外部的局部变量时，这个函数就是一个闭包，它会"记住"那些变量**。

先看一个简单的例子：

```lua
local function make_counter()
    local count = 0     -- 外部的局部变量
    return function()   -- 返回一个内部函数
        count = count + 1
        return count
    end
end

local counter = make_counter()
print(counter())  -- 1
print(counter())  -- 2
print(counter())  -- 3
```

`counter` 函数"记住"了 `count` 变量，每次调用都会在上一次的基础上加 1。即使 `make_counter` 已经执行完毕，`count` 也不会被销毁，因为闭包还在引用它。

### 1.3.6 闭包在饥荒中的实际应用

饥荒大量使用闭包。来看礼物接收器组件（`giftreceiver.lua`）中的例子：

```lua
local GiftReceiver = Class(function(self, inst)
    self.inst = inst
    self.giftcount = 0
    self.giftmachine = nil

    -- 这里创建了一个闭包，它"记住"了 self
    self.onclosepopup = function(doer, data)
        if data.popup == POPUPS.GIFTITEM then
            self:OnStopOpenGift(data.args[1])
        end
    end
    inst:ListenForEvent("ms_closepopup", self.onclosepopup)
end)
```

这里的匿名函数引用了外部的 `self`，形成了闭包。为什么要这么做？因为事件回调函数的签名是固定的（`doer, data`），没有传 `self` 的参数位。通过闭包，函数就能"记住"创建时的 `self`，从而访问到组件实例。

再来看青蛙雨组件（`frograin.lua`）中更典型的闭包用法：

```lua
return Class(function(self, inst)
    -- 这些 local 变量被下面的函数共享
    local _activeplayers = {}
    local _frogs = {}
    local _frogcap = 0
    local _updating = false

    -- 这个函数引用了上面的 local 变量，形成闭包
    local function GetSpawnPoint(pt)
        local function TestSpawnPoint(offset)
            local spawnpoint = pt + offset
            return _map:IsAboveGroundAtPoint(spawnpoint:Get())
            -- _map 也是上面定义的闭包变量
        end
        -- ...
    end
end)
```

整个组件的私有变量都是通过闭包实现的——`_activeplayers`、`_frogs` 等变量对外部不可见，只有同一个闭包作用域内的函数才能访问。这是 Lua 实现**私有成员**的标准手法。

### 1.3.7 可变参数

Lua 用 `...` 表示可变参数：

```lua
local function myprint(...)
    local args = {...}  -- 把可变参数收集到一个 table 里
    for i, v in ipairs(args) do
        print(v)
    end
end

myprint("a", "b", "c")  -- 打印 a、b、c
```

在饥荒中，`SpawnPrefab` 生成实体后经常需要链式设置，可变参数在这类工具函数中很有用。

### 1.3.8 函数存在 table 里——方法的实现

在 Lua 中，把函数放进 table 就可以模拟"对象方法"：

```lua
local dog = {}
dog.name = "Woby"

dog.bark = function(self)
    print(self.name .. " says: Woof!")
end

dog.bark(dog)   -- Woby says: Woof!
dog:bark()      -- 等价写法，用冒号自动把 dog 作为第一个参数 self 传入
```

**冒号语法 `:` 是 Lua 的语法糖**：
- `dog:bark()` 等价于 `dog.bark(dog)`
- 定义时 `function dog:bark()` 等价于 `function dog.bark(self)`

饥荒中到处都在用冒号语法：

```lua
-- 这两行是等价的
inst.components.health:DoDelta(-10)
inst.components.health.DoDelta(inst.components.health, -10)
```

---

> **本节小结**
> - 函数是 Lua 的一等公民，可以赋值、传参、存入 table
> - 回调模式是饥荒编程的核心——把函数注册到系统中，由事件触发调用
> - 匿名函数适合短小的一次性回调
> - 闭包让函数"记住"外部变量，饥荒用它实现私有成员和事件监听
> - 冒号语法 `:` 是方法调用的语法糖，自动传递 `self`

## 1.4 元表与面向对象（Class 系统的基础）

Lua 本身没有"类"的概念，没有 `class` 关键字，也没有继承语法。但 Lua 提供了一个极其灵活的机制——**元表（metatable）**，让你可以自己"造"出面向对象系统。饥荒的整个游戏架构——实体、组件、状态图、行为树——全部建立在用元表实现的 `Class` 系统之上。

### 1.4.1 什么是元表——给 table 装上"超能力"

我们知道 table 是 Lua 唯一的复合数据结构。默认情况下，table 只能做最基本的操作：存取键值、遍历。但如果你给一个 table 设置了**元表**，就可以改变它的行为——比如让两个 table 相加、让 table 像函数一样被调用、或者在访问不存在的键时自动去别处查找。

```lua
-- 创建一个普通 table
local t = {name = "axe"}

-- 创建一个元表
local mt = {}

-- 把 mt 设为 t 的元表
setmetatable(t, mt)

-- 获取 t 的元表
print(getmetatable(t) == mt)  -- true
```

元表本身也是一个普通的 table，但它里面可以放一些以双下划线 `__` 开头的特殊键，叫做**元方法（metamethod）**。Lua 在执行某些操作时会去元表里查找对应的元方法，如果找到了，就用它来处理。

### 1.4.2 最重要的元方法——__index

`__index` 是饥荒中使用最频繁的元方法，也是整个 Class 系统的基石。它的作用是：**当你访问 table 中不存在的键时，Lua 会去元表的 `__index` 里查找**。

先看一个简单的例子理解原理：

```lua
local defaults = {
    health = 100,
    damage = 10,
    speed = 5,
}

local mt = { __index = defaults }

local warrior = setmetatable({damage = 25}, mt)

print(warrior.damage)   -- 25（warrior 自己有这个键，直接返回）
print(warrior.health)   -- 100（warrior 没有，去 defaults 里找，找到了）
print(warrior.speed)    -- 5（同上）
print(warrior.name)     -- nil（warrior 没有，defaults 也没有）
```

`warrior` 本身只定义了 `damage = 25`，但通过 `__index`，它可以"继承"到 `defaults` 里的 `health` 和 `speed`。这就是 Lua 实现继承的核心原理——**原型链**。

`__index` 也可以是一个**函数**。饥荒的 `replica` 系统就是这么做的：

```lua
-- 来自 scripts/entityscript.lua
local replica_mt =
{
    __index = function(t, k)
        return rawget(t, "inst"):ValidateReplicaComponent(k, rawget(t, "_")[k])
    end,
}
```

当你访问 `inst.replica.health` 时，Lua 发现 `replica` 表里没有 `health` 这个键，于是调用 `__index` 函数。函数接收两个参数：`t`（被访问的表）和 `k`（被访问的键名）。这里它会先校验组件的合法性再返回。

为什么要用函数而不是直接用另一个 table？因为 replica 组件是动态注册的，而且需要做额外的校验逻辑，简单的"去另一个 table 查找"满足不了需求。

### 1.4.3 __newindex——拦截赋值操作

`__newindex` 在你**给 table 中不存在的键赋值**时触发。饥荒用它来实现**属性监听器**——当某个属性的值发生变化时，自动同步到网络（客户端/服务端同步）。

来看一个让你惊叹的实际用例——`health` 组件的定义：

```lua
-- 来自 scripts/components/health.lua
local Health = Class(function(self, inst)
    self.inst = inst
    self.maxhealth = 100
    self.minhealth = 0
    self.currenthealth = self.maxhealth
    self.invincible = false
    -- ...
end,
nil,   -- 没有基类
{      -- 第三个参数：属性监听器（props）
    maxhealth = onmaxhealth,
    currenthealth = oncurrenthealth,
    takingfiredamage = ontakingfiredamage,
    penalty = onpenalty,
    canmurder = oncanmurder,
    canheal = oncanheal,
    invincible = oninvincible,
})
```

看到那个第三个参数了吗？它是一个键值对表：键是属性名，值是一个回调函数。当你执行 `self.currenthealth = 50` 时，`__newindex` 元方法会拦截这个赋值，除了更新值之外，还会自动调用 `oncurrenthealth` 回调：

```lua
-- 来自 scripts/components/health.lua
local function oncurrenthealth(self, currenthealth)
    self.inst.replica.health:SetCurrent(currenthealth)    -- 同步到客户端
    self.inst.replica.health:SetIsDead(currenthealth <= 0) -- 同步死亡状态
    -- ...
end
```

这样开发者只需要写 `self.currenthealth = 50` 这样自然的赋值语句，网络同步就自动完成了，不需要每次都手动调用同步函数。这就是元表的威力——**让复杂的逻辑隐藏在简单的语法背后**。

底层的实现在 `class.lua` 中：

```lua
-- 来自 scripts/class.lua
local function __newindex(t, k, v)
    local p = rawget(t, "_")[k]  -- 查找是否有属性监听器
    if p == nil then
        rawset(t, k, v)          -- 没有监听器，正常赋值
    else
        local old = p[1]         -- 取出旧值
        p[1] = v                 -- 存入新值
        p[2](t, v, old)          -- 调用回调函数，传入 self, 新值, 旧值
    end
end
```

> **给新手的提示**：`rawget` 和 `rawset` 是 Lua 的原始操作，它们绕过元表直接读写 table。如果在 `__index` 或 `__newindex` 里又触发了元方法，就会无限递归，所以必须用 `rawget` / `rawset`。

### 1.4.4 其他常用元方法——让 table 像数字一样运算

除了 `__index` 和 `__newindex`，Lua 还提供了许多元方法。饥荒的 `Vector3` 类是展示这些元方法的最佳范例——它让三维坐标向量可以直接用 `+`、`-`、`*` 进行运算：

```lua
-- 来自 scripts/vector3.lua
Vector3 = Class(function(self, x, y, z)
    self.x, self.y, self.z = x or 0, y or 0, z or 0
end)

-- __add：让两个 Vector3 可以用 + 号相加
function Vector3:__add(rhs)
    return Vector3(self.x + rhs.x, self.y + rhs.y, self.z + rhs.z)
end

-- __sub：减法
function Vector3:__sub(rhs)
    return Vector3(self.x - rhs.x, self.y - rhs.y, self.z - rhs.z)
end

-- __mul：标量乘法
function Vector3:__mul(rhs)
    return Vector3(self.x * rhs, self.y * rhs, self.z * rhs)
end

-- __unm：取负号
function Vector3:__unm()
    return Vector3(-self.x, -self.y, -self.z)
end

-- __eq：判断两个向量是否相等
function Vector3:__eq(rhs)
    return self.x == rhs.x and self.y == rhs.y and self.z == rhs.z
end

-- __tostring：print 时的输出格式
function Vector3:__tostring()
    return string.format("(%2.2f, %2.2f, %2.2f)", self.x, self.y, self.z)
end
```

有了这些元方法，你就可以像操作数字一样操作向量：

```lua
local pos1 = Vector3(10, 0, 20)
local pos2 = Vector3(5, 0, 8)

local sum = pos1 + pos2        -- Vector3(15, 0, 28)
local diff = pos1 - pos2       -- Vector3(5, 0, 12)
local scaled = pos1 * 2        -- Vector3(20, 0, 40)
local neg = -pos1              -- Vector3(-10, 0, -20)

print(pos1)                    -- (10.00, 0.00, 20.00)
print(pos1 == pos2)            -- false
```

这在游戏开发中极其实用——你需要大量做坐标运算（求距离、求方向、偏移位置等），元方法让代码简洁得多。

以下是 Lua 中常用元方法的完整列表：

| 元方法 | 触发时机 | 饥荒中的典型用途 |
|--------|---------|-----------------|
| `__index` | 访问不存在的键 | 实现继承（Class 的核心）|
| `__newindex` | 给不存在的键赋值 | 属性监听器（网络同步）|
| `__tostring` | `tostring()` 或 `print()` | 调试输出（EntityScript、Vector3）|
| `__add` | `+` 运算 | 向量加法 |
| `__sub` | `-` 运算（二元）| 向量减法 |
| `__mul` | `*` 运算 | 向量缩放 |
| `__div` | `/` 运算 | 向量除法 |
| `__unm` | `-` 运算（一元取负）| 向量取反 |
| `__eq` | `==` 比较 | 向量/位置比较 |
| `__call` | 像函数一样调用 table | `Class()` 创建实例 |
| `__len` | `#` 取长度 | 自定义长度 |

### 1.4.5 用元表实现"继承"——从零理解原理

在深入饥荒的 Class 系统之前，让我们先从零手写一个最简单的面向对象系统，理解元表是怎么实现继承的：

```lua
-- 第一步：定义一个"类"
local Animal = {}
Animal.__index = Animal    -- 关键：让实例在找不到方法时，去 Animal 里找

function Animal.new(name, sound)
    local self = {}
    setmetatable(self, Animal)  -- 将 Animal 设为实例的元表
    self.name = name
    self.sound = sound
    return self
end

function Animal:speak()
    print(self.name .. " says " .. self.sound)
end

-- 第二步：创建实例
local dog = Animal.new("Dog", "Woof")
dog:speak()  -- Dog says Woof
```

这里的关键是 `Animal.__index = Animal`。当你调用 `dog:speak()` 时：
1. Lua 在 `dog` 表里找 `speak`——没找到
2. Lua 查看 `dog` 的元表（`Animal`），发现 `__index` 指向 `Animal` 自身
3. Lua 在 `Animal` 里找到了 `speak`，调用它

这就是原型继承——实例 `dog` 通过元表"继承"了 `Animal` 的所有方法。

**继承的实现**——让一个类继承另一个类：

```lua
-- 第三步：定义子类
local Dog = {}
for k, v in pairs(Animal) do  -- 把 Animal 的方法复制到 Dog
    Dog[k] = v
end
Dog.__index = Dog
Dog._base = Animal  -- 记录父类

function Dog.new(name)
    local self = Animal.new(name, "Woof")  -- 调用父类构造函数
    setmetatable(self, Dog)                -- 改为用 Dog 作元表
    return self
end

function Dog:fetch(item)
    print(self.name .. " fetches " .. item)
end

local rex = Dog.new("Rex")
rex:speak()          -- Rex says Woof（继承自 Animal）
rex:fetch("stick")   -- Rex fetches stick（Dog 自己的方法）
```

是不是觉得每次手写这些很麻烦？这就是为什么饥荒封装了 `Class` 函数。

### 1.4.6 饥荒的 Class 函数——剥开源码看本质

现在让我们看饥荒的 `Class` 函数是怎么实现的。核心代码在 `scripts/class.lua` 中：

```lua
-- 来自 scripts/class.lua（简化版，突出核心逻辑）
function Class(base, _ctor, props)
    local c = {}    -- 新类的原型表
    
    -- 如果只传了一个函数，说明没有基类
    if not _ctor and type(base) == 'function' then
        _ctor = base
        base = nil
    -- 如果传了基类（一个 table），把基类的方法复制过来
    elseif type(base) == 'table' then
        for i, v in pairs(base) do
            c[i] = v             -- 浅拷贝基类的所有方法
        end
        c._base = base           -- 记住基类，用于 is_a 查找
    end

    -- 实例在找不到方法时，去类的原型表 c 里找
    c.__index = c

    -- 创建一个元表 mt，让 c 可以被"调用"（像函数一样）
    local mt = {}
    mt.__call = function(class_tbl, ...)
        local obj = {}               -- 创建新实例
        setmetatable(obj, c)         -- 设置元表为类原型
        if c._ctor then
            c._ctor(obj, ...)        -- 调用构造函数
        end
        return obj
    end

    c._ctor = _ctor                  -- 保存构造函数
    c.is_a = _is_a                   -- 类型检查方法
    
    setmetatable(c, mt)              -- 让类本身可以被"调用"
    return c
end
```

这段代码做了四件事：

1. **创建原型表 `c`**：这是类的"模板"，所有方法都定义在这里
2. **复制基类方法**：如果有基类，把基类的方法全部浅拷贝到 `c`，实现继承
3. **设置 `__index`**：让实例找不到方法时去 `c` 里查找
4. **设置 `__call`**：让类可以像函数一样被调用来创建实例

当你写 `Health(inst)` 时，实际上是在调用 `mt.__call`，它创建一个新 table、设置元表、调用构造函数，然后返回这个新实例。

### 1.4.7 Class 的三种使用模式

理解了原理后，来看饥荒中 `Class` 的三种实际使用方式。

**模式一：无基类——最常见的组件定义**

绝大多数组件都使用这种模式，只传一个构造函数：

```lua
-- 来自 scripts/components/combat.lua
local Combat = Class(function(self, inst)
    self.inst = inst
    self.target = nil
    self.defaultdamage = 0
    self.damagemultiplier = 1
    -- ...
end)

-- 方法定义在类上（实例通过 __index 找到）
function Combat:SetDefaultDamage(damage)
    self.defaultdamage = damage
end

function Combat:GetDamage()
    return self.defaultdamage * self.damagemultiplier
end
```

这里 `Class(function(self, inst) ... end)` 只传了一个参数（函数），没有基类。`Class` 内部检测到第一个参数是函数，就把它当做构造函数。

**模式二：带基类——继承现有类**

当你需要扩展一个已有的类时，传两个参数：

```lua
-- 来自 scripts/components/aoeweapon_leap.lua
local AOEWeapon_Base = require("components/aoeweapon_base")

local AOEWeapon_Leap = Class(AOEWeapon_Base, function(self, inst)
    AOEWeapon_Base._ctor(self, inst)   -- 调用父类构造函数
    self.aoeradius = 4
    self.physicspadding = 3
    inst:AddTag("aoeweapon_leap")
end)
```

注意这里**必须手动调用 `AOEWeapon_Base._ctor(self, inst)`**——饥荒的 Class 系统不会自动调用父类构造函数。这和 Java/Python 等语言不同，需要特别注意。

行为树中更能看到继承链的威力：

```lua
-- 来自 scripts/behaviourtree.lua
-- 基类：所有行为树节点的父类
BehaviourNode = Class(function(self, name, children)
    self.name = name or ""
    self.children = children
    self.status = READY
end)

-- 子类：装饰器节点，继承自 BehaviourNode
DecoratorNode = Class(BehaviourNode, function(self, name, child)
    BehaviourNode._ctor(self, name or "Decorator", {child})
end)

-- 子类：条件节点，也继承自 BehaviourNode
ConditionNode = Class(BehaviourNode, function(self, fn, name)
    BehaviourNode._ctor(self, name or "Condition")
    self.fn = fn
end)
```

`DecoratorNode` 和 `ConditionNode` 都继承了 `BehaviourNode` 的所有方法（如 `Visit`、`Reset` 等），同时可以添加或重写自己的方法。

**模式三：带属性监听器（props）——联网组件**

这是最高级的用法，传三个参数：

```lua
-- 来自 scripts/components/health.lua
local Health = Class(function(self, inst)
    self.inst = inst
    self.maxhealth = 100
    self.currenthealth = self.maxhealth
    -- ...
end,
nil,     -- 第二个参数 nil 表示没有基类
{        -- 第三个参数：属性监听器
    maxhealth = onmaxhealth,
    currenthealth = oncurrenthealth,
    takingfiredamage = ontakingfiredamage,
    penalty = onpenalty,
})
```

有了属性监听器后，每次 `self.currenthealth = 50` 都会自动触发 `oncurrenthealth` 回调。这个机制主要用于网络同步——服务端修改一个值时，需要自动把变化通知到客户端。

> **给初学者的提示**：如果你只是做简单的 Mod，基本只用到模式一（无基类）。模式二在你需要扩展已有组件时用到。模式三几乎只在引擎核心代码里出现，做 Mod 时很少需要自己用。

### 1.4.8 is_a——类型检查

`Class` 系统提供了 `is_a` 方法来检查一个对象是否属于某个类（或其子类）。这类似于 Java 中的 `instanceof` 或 Python 中的 `isinstance`。

```lua
-- 来自 scripts/stategraph.lua —— 检查传入的是否真的是 ActionHandler
for k, v in pairs(actionhandlers) do
    assert(v:is_a(ActionHandler), "Non-action handler added in actionhandler table!")
    self.actionhandlers[v.action] = v
end
```

`is_a` 的实现其实很简单——沿着元表链往上查找：

```lua
-- 来自 scripts/class.lua
local function _is_a(self, klass)
    local m = getmetatable(self)
    while m do
        if m == klass then return true end
        m = m._base                         -- 沿继承链向上查找
    end
    return false
end
```

它会从当前对象的元表开始，一层层往上查找 `_base`（父类），直到找到匹配的类或到达继承链顶端。

在饥荒的行为树中经常用 `is_a` 做类型区分：

```lua
-- 来自 scripts/behaviourtree.lua —— 只有非 ConditionNode 才需要睡眠计时
if self.status == RUNNING and not self.children and not self:is_a(ConditionNode) then
    -- ...
end
```

在动作选择器中区分目标是实体还是坐标：

```lua
-- 来自 scripts/components/playeractionpicker.lua
if target:is_a(EntityScript) then
    -- 目标是一个实体（比如一棵树、一个怪物）
elseif target:is_a(Vector3) then
    -- 目标是一个坐标点（比如地面某个位置）
end
```

### 1.4.9 __tostring——调试你的类

定义 `__tostring` 元方法后，`print()` 和 `tostring()` 会使用你定义的格式输出对象，而不是默认的 `table: 0x...`。这对调试非常有帮助。

```lua
-- 来自 scripts/entityscript.lua
function EntityScript:__tostring()
    return string.format("%d - %s%s", self.GUID, self.prefab or "", self.inlimbo and "(LIMBO)" or "")
end
-- 打印一个实体时会显示：1234 - axe 或 5678 - wilson(LIMBO)

-- 来自 scripts/stategraph.lua
function StateGraph:__tostring()
    return "Stategraph : " .. self.name
end
-- 打印状态图时会显示：Stategraph : wilson
```

在你自己的 Mod 中，也建议给自定义类加上 `__tostring`，方便调试时快速看出对象的内容。

### 1.4.10 __call——让 table 像函数一样调用

`__call` 让你可以对一个 table 使用函数调用的语法 `()`。饥荒的 `Class` 函数正是用它来实现"调用类名创建实例"的效果：

```lua
-- class.lua 中给每个类设置了 __call
mt.__call = function(class_tbl, ...)
    local obj = {}
    setmetatable(obj, c)
    if c._ctor then
        c._ctor(obj, ...)
    end
    return obj
end
```

所以当你写 `Health(inst)` 时：
1. `Health` 本身是一个 table（类原型）
2. Lua 发现你在对一个 table 使用 `()`，于是查找元表的 `__call`
3. `__call` 创建一个新的空 table，设置元表，调用构造函数
4. 返回新创建的实例

这就是为什么 `Vector3(10, 0, 20)` 看起来像在调用构造函数，实际上是元表在幕后工作。

### 1.4.11 元表的查找链——理解方法调用的完整过程

把前面的知识串起来，看看当你在饥荒中写 `inst.components.health:DoDelta(-10)` 时，Lua 到底做了什么：

```
1. inst.components         → 在 inst 表中找到 components 表
2. .health                 → 在 components 表中找到 health 实例
3. :DoDelta(-10)           → 等价于 health.DoDelta(health, -10)
   3a. 在 health 实例中找 DoDelta → 没找到
   3b. 查看 health 的元表（Health 类原型）
   3c. 在 Health 类原型中找到 DoDelta → 调用它
```

如果 `Health` 继承了某个基类，而 `DoDelta` 定义在基类中，查找会继续：

```
   3c. 在 Health 类原型中找 DoDelta → 没找到
   3d. 查看 Health 原型的 __index → 指向自身，但通过 _base 可追溯
       （实际上，Class 在创建时已经把基类的方法拷贝到了 c 中，
        所以通常不需要多级查找）
```

> **给老鸟的提示**：饥荒的 `Class` 系统在继承时使用的是**浅拷贝**而非原型链查找。也就是说，`Class(Base, ctor)` 会在创建时把 `Base` 的所有字段拷贝到新类的原型表中。这意味着：
> - 查找速度更快（不需要遍历链）
> - 但如果你后来往 `Base` 中添加了新方法，已经创建的子类不会自动获得它
> - 这和 JavaScript 的原型链不同，更接近"混入（mixin）"模式

### 1.4.12 实战：用 Class 写一个简单组件

把这一节学到的知识汇总，写一个完整的自定义组件：

```lua
-- 文件：scripts/components/mycomponent.lua

local MyComponent = Class(function(self, inst)
    self.inst = inst
    self.value = 0
    self.max_value = 100
end)

function MyComponent:SetValue(val)
    self.value = math.clamp(val, 0, self.max_value)
    self.inst:PushEvent("myvaluechanged", { value = self.value })
end

function MyComponent:GetPercent()
    return self.value / self.max_value
end

function MyComponent:DoDelta(delta)
    self:SetValue(self.value + delta)
end

function MyComponent:OnSave()
    return { value = self.value }
end

function MyComponent:OnLoad(data)
    if data and data.value then
        self:SetValue(data.value)
    end
end

function MyComponent:GetDebugString()
    return string.format("value: %2.2f / %2.2f (%2.0f%%)",
        self.value, self.max_value, self:GetPercent() * 100)
end

return MyComponent
```

这个组件遵循了饥荒组件的标准模式：
- 用 `Class(function(self, inst) ... end)` 定义
- 构造函数初始化所有字段
- 方法用冒号语法定义（`function MyComponent:Method()`）
- 提供 `OnSave` / `OnLoad` 实现存档
- 提供 `GetDebugString` 方便调试
- 最后 `return MyComponent` 导出

---

> **本节小结**
> - 元表通过特殊的元方法（`__index`、`__newindex`、`__call` 等）赋予 table 超越普通操作的能力
> - `__index` 是实现继承的核心——实例找不到方法时，沿着元表链去类原型中查找
> - `__newindex` 用于拦截赋值，饥荒用它实现属性监听和网络同步
> - 饥荒的 `Class` 函数封装了元表操作，提供三种模式：无基类、有基类（继承）、带属性监听器
> - `is_a` 沿继承链检查类型，类似其他语言的 `instanceof`
> - `Vector3` 是元方法运算符重载的完美范例——让向量可以像数字一样 `+`、`-`、`*`
> - 做 Mod 时最常用的是"无基类"模式，掌握 `Class(function(self, inst) ... end)` 即可应对大部分场景

## 1.5 模块系统与 require

（待编写）

## 1.6 协程基础与 StartThread——饥荒中的"线程"

（待编写）

## 1.7 字符串模式匹配——饥荒代码中的 string.find / string.match

（待编写）

## 1.8 弱引用表（Weak Table）——避免内存泄漏的利器

（待编写）
