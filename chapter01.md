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

（待编写）

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
