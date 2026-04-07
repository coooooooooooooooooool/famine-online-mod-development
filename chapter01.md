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

（待编写）

## 1.4 元表与面向对象（Class 系统的基础）

（待编写）

## 1.5 模块系统与 require

（待编写）

## 1.6 协程基础与 StartThread——饥荒中的"线程"

（待编写）

## 1.7 字符串模式匹配——饥荒代码中的 string.find / string.match

（待编写）

## 1.8 弱引用表（Weak Table）——避免内存泄漏的利器

（待编写）
