# 第2章 饥荒的游戏架构

## 2.1 目录结构与文件组织

了解饥荒的文件组织方式，就像拿到了一栋大楼的平面图——你知道每个房间在哪、干什么用的，才能快速找到你想改的东西。本节从全局到局部，带你理清饥荒联机版 `scripts` 目录下几千个 Lua 文件的组织逻辑。

### 2.1.1 scripts 目录总览——一张全局地图

饥荒联机版的所有游戏逻辑都在 `scripts` 文件夹中，包含约 **3900+ 个 Lua 文件**，分布在 14 个子目录和约 200 多个根目录文件中：

```
scripts/
├── prefabs/          # 1544 个文件 —— 实体定义（"世界上的东西长什么样、能干什么"）
├── components/       # 792 个文件  —— 组件（可复用的行为模块）
├── map/              # 444 个文件  —— 世界生成与地图布局
├── widgets/          # 268 个文件  —— UI 小部件（血条、物品栏、按钮等）
├── stategraphs/      # 251 个文件  —— 状态图（动画状态机）
├── brains/           # 184 个文件  —— AI 大脑（决定生物做什么）
├── screens/          # 134 个文件  —— 全屏界面（主菜单、设置、地图等）
├── scenarios/        # 50 个文件   —— 场景脚本（特殊事件、关卡）
├── behaviours/       # 28 个文件   —— 行为树节点（AI 的基本动作单元）
├── util/             # 8 个文件    —— 工具函数库
├── cameras/          # 1 个文件    —— 摄像机控制
├── languages/        # 翻译与本地化
├── nis/              # 非交互式序列（过场动画脚本）
├── tools/            # 开发工具
└── *.lua             # ~217 个根文件 —— 全局定义、核心系统、配置
```

别被数量吓到——这些文件有清晰的分类逻辑。下面逐个讲解每个部分的作用。

### 2.1.2 prefabs——游戏世界中的一切"东西"

`prefabs` 是最大的目录，有 **1544 个文件**。**游戏中你能看到、捡起、攻击、建造的所有东西**——从一棵树到一把斧头、从一只蜘蛛到威尔逊本人——都是一个 prefab。

> **给新手的解释**："Prefab"是 "prefabricated（预制件）"的缩写，源自游戏引擎术语。你可以把它理解为一个"实体模板"——它定义了一个东西**是什么**（有哪些组件、什么外观、什么行为），但不关心它**在哪里**（位置由生成时决定）。

每个 prefab 文件通常遵循这样的结构：

```lua
-- 典型的 prefab 文件结构（以 axe.lua 为例）

-- 1. 资源声明
local assets = {
    Asset("ANIM", "anim/axe.zip"),
    Asset("ANIM", "anim/swap_axe.zip"),
}

-- 2. 局部辅助函数
local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_object", "swap_axe", "swap_axe")
end

-- 3. 构建函数（核心）
local function fn()
    local inst = CreateEntity()          -- 创建空实体
    -- 添加引擎级组件（Transform, AnimState 等）
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    -- 设置动画
    inst.AnimState:SetBank("axe")
    inst.AnimState:SetBuild("axe")
    inst.AnimState:PlayAnimation("idle")
    -- 添加 Lua 层组件
    inst:AddComponent("inventoryitem")   -- 可以被捡起
    inst:AddComponent("equippable")      -- 可以被装备
    inst:AddComponent("tool")            -- 是一个工具
    inst:AddComponent("weapon")          -- 可以当武器
    inst:AddComponent("finiteuses")      -- 有耐久度
    return inst
end

-- 4. 返回 Prefab 对象
return Prefab("axe", fn, assets)
```

一个文件可以定义多个 prefab。比如 `axe.lua` 同时定义了斧头、金斧头和月光石斧：

```lua
return Prefab("axe", normal, assets),
    Prefab("goldenaxe", golden, golden_assets),
    Prefab("moonglassaxe", moonglass, moonglass_assets)
```

所有 prefab 文件名在 `scripts/prefablist.lua` 中被登记（这是一个自动生成的列表），引擎启动时会遍历这个列表加载所有 prefab。

**prefab 文件的组织方式**：没有子目录，所有 1544 个文件都平铺在 `prefabs/` 下。文件名通常就是 prefab 名（如 `axe.lua` → prefab 名 `"axe"`），但也有一个文件包含多个 prefab 的情况。

**对 Mod 开发者来说**：你的自定义物品、生物、建筑都要以 prefab 文件的形式编写，放在你 Mod 的 `scripts/prefabs/` 目录下。

### 2.1.3 components——可插拔的行为模块

`components` 是第二大的目录，有 **792 个文件**。组件是饥荒最核心的架构思想之一：**不同的实体通过组合不同的组件来获得不同的能力**。

> **给新手的类比**：组件就像乐高积木。一把斧头 = 可捡起 + 可装备 + 工具 + 武器 + 有耐久。一棵树 = 可砍 + 可被点燃 + 有掉落物。同一个"可被点燃"的组件（`burnable`）被树、营火、角色等完全不同的实体共享。

常见的组件（你会频繁接触的）：

| 组件文件 | 功能 | 典型使用者 |
|---------|------|-----------|
| `health.lua` | 生命值 | 角色、生物、可被攻击的建筑 |
| `combat.lua` | 战斗 | 能攻击或被攻击的实体 |
| `inventory.lua` | 物品栏 | 玩家、猪人（能捡东西的生物）|
| `inventoryitem.lua` | 可被捡起 | 所有地上的物品 |
| `equippable.lua` | 可装备 | 武器、帽子、护甲 |
| `hunger.lua` | 饥饿值 | 玩家 |
| `sanity.lua` | 理智值 | 玩家 |
| `burnable.lua` | 可燃 | 树、建筑、物品 |
| `workable.lua` | 可被工具作用 | 树（可砍）、矿石（可挖）|
| `container.lua` | 容器 | 箱子、冰箱、背包 |
| `locomotor.lua` | 移动 | 能走动的生物 |
| `lootdropper.lua` | 掉落物 | 被杀死/破坏后掉东西的实体 |
| `stackable.lua` | 可堆叠 | 材料、食物 |

组件文件有两种命名模式：

- **服务端组件**：如 `health.lua`、`combat.lua` —— 只在服务端运行，掌管实际的游戏逻辑
- **客户端副本（replica）**：如 `health_replica.lua`、`combat_replica.lua` —— 在客户端运行，用于 UI 显示和预测

> **给老鸟的说明**：`_replica` 后缀的组件是饥荒 C/S 架构的产物。服务端组件拥有完整的数据和逻辑；客户端的 replica 组件只持有同步过来的"只读"数据快照，用于渲染 HUD 和做客户端预测（比如显示血条不需要等服务端回包）。这个设计在 2.6 节会详细展开。

### 2.1.4 stategraphs——动画状态机

`stategraphs` 有 **251 个文件**，文件名都以 `SG` 开头（如 `SGwilson.lua`、`SGpig.lua`）。状态图控制实体的**动画和行为状态之间的转换**。

> **给新手的解释**：想象一个角色的状态——站着、走路、攻击、吃东西、被打、死亡——每个状态对应一组动画。状态图定义了"什么时候从一个状态切换到另一个状态"以及"每个状态下做什么"。

```
idle（站着）──按下移动键──→ run（跑步）──松开移动键──→ idle
   │                            │
   ├──按攻击键──→ attack（攻击）──攻击结束──→ idle
   │
   └──受到伤害──→ hit（被打）──恢复──→ idle
```

`SGwilson.lua` 是最大的状态图文件（所有玩家角色共用），包含了几百个状态——砍树、挖矿、钓鱼、划船、骑牛、使用魔法书……玩家能做的每一个动作都有对应的状态。

### 2.1.5 brains 与 behaviours——AI 系统

`brains` 目录有 **184 个文件**，每个文件定义一种生物的 AI 决策逻辑。`behaviours` 有 **28 个文件**，定义了 AI 的基本行为节点。

> **给新手的类比**：如果 `stategraph` 是角色的"身体"（知道怎么做每个动作），那么 `brain` 就是角色的"大脑"（决定下一步做什么）。`behaviours` 是大脑的"基本本能"（走向目标、逃跑、跟随、巡逻等）。

```lua
-- brains/spiderbrain.lua 的简化逻辑：
-- 蜘蛛的 AI 大概是这样思考的：
-- 1. 如果附近有食物，去吃
-- 2. 如果主人（蜘蛛女王）叫我攻击，去攻击
-- 3. 如果天亮了，回巢穴
-- 4. 如果没事干，在巢穴附近晃悠
```

### 2.1.6 widgets 与 screens——用户界面

`widgets`（268 个文件）是 UI 的基础构件——血条、饥饿条、物品格子、按钮、文本框等。`screens`（134 个文件）是完整的界面——主菜单、设置界面、服务器列表、角色选择等。

widget 和 screen 的关系：screen 由多个 widget 组合而成，就像一个网页由多个 HTML 元素组成。

### 2.1.7 map——世界生成

`map` 目录有 **444 个文件**，负责饥荒最复杂的系统之一——程序化世界生成。包括：

- 地形瓦片定义（`terrain.lua`）
- 房间模板（`rooms/`）——定义每种区域里有什么（沼泽有蜘蛛网和触手，森林有树和浆果丛）
- 任务流（`tasks/`、`tasksets/`）——定义区域如何连接
- 自定义选项（`customize.lua`）——世界设置面板的选项
- 出生点（`startlocations/`）

### 2.1.8 根目录文件——游戏的"胶水"和骨架

`scripts` 根目录下有约 **217 个 Lua 文件**，它们是连接所有子系统的核心。按功能分类：

**启动与主循环**

| 文件 | 作用 |
|------|------|
| `main.lua` | 游戏入口，配置 `package.path`，加载基础模块 |
| `mainfunctions.lua` | 核心函数：`LoadPrefabFile`、`RegisterPrefabs`、`SpawnPrefab` 等 |
| `gamelogic.lua` | 游戏逻辑主控：加载世界、管理 prefab 注册、世界初始化 |
| `update.lua` | 每帧更新调度 |

**核心架构**

| 文件 | 作用 |
|------|------|
| `class.lua` | 面向对象系统（`Class` 函数）|
| `entityscript.lua` | 所有实体的基类（`EntityScript`）|
| `scheduler.lua` | 协程调度器和定时任务系统 |
| `stategraph.lua` | 状态图基类（`StateGraph`、`State`、`EventHandler`）|
| `behaviourtree.lua` | 行为树基类（`BehaviourNode`、`BT`）|
| `actions.lua` | 所有交互动作的定义（砍、挖、捡、吃……）|

**数据与配置**

| 文件 | 作用 |
|------|------|
| `tuning.lua` | 全局数值平衡表（攻击力、生命值、持续时间……）|
| `constants.lua` | 全局常量（方向、图层、碰撞组……）|
| `strings.lua` | 所有游戏内文本 |
| `recipes.lua` | 合成配方 |
| `containers.lua` | 容器配置（箱子几格、冰箱几格……）|
| `prefabs.lua` | `Prefab` 和 `Asset` 类定义 |
| `prefablist.lua` | 自动生成的 prefab 文件名列表 |
| `preparedfoods.lua` | 烹饪锅食谱 |

**网络与多人**

| 文件 | 作用 |
|------|------|
| `networking.lua` | 网络相关工具函数 |
| `networkclientrpc.lua` | 客户端远程调用 |
| `networkserverrpc.lua` | 服务端远程调用 |
| `shardnetworking.lua` | 多世界（地上/洞穴）通信 |

**工具库**

| 文件 | 作用 |
|------|------|
| `util.lua` | 通用工具函数（深拷贝、查找等）|
| `simutil.lua` | 模拟相关工具（查找附近实体等）|
| `mathutil.lua` | 数学工具（插值、角度等）|
| `stringutil.lua` | 字符串工具（`subfmt`、`TrimString` 等）|
| `debugtools.lua` | 调试工具 |

**角色台词**（大量 `speech_*.lua` 文件）

每个角色有独立的台词文件：`speech_wilson.lua`、`speech_willow.lua`、`speech_wendy.lua` ……定义了该角色检查每种物品时说的话。

### 2.1.9 Mod 的目录结构——你自己的代码怎么组织

了解了原版的结构后，来看 Mod 应该怎么组织。一个标准的 Mod 目录结构是：

```
mods/my_cool_mod/
├── modinfo.lua              # Mod 元信息（名称、描述、版本、配置选项）
├── modmain.lua              # Mod 入口（注册 PostInit、修改全局配置等）
├── modworldgenmain.lua      # 世界生成修改（可选）
├── scripts/
│   ├── prefabs/
│   │   └── my_item.lua      # 自定义实体
│   ├── components/
│   │   └── mycomponent.lua  # 自定义组件
│   ├── stategraphs/
│   │   └── SGmy_creature.lua # 自定义状态图
│   └── brains/
│       └── my_creature_brain.lua # 自定义 AI
├── anim/                    # 动画文件（.zip）
├── images/                  # 图片资源（物品图标等）
└── sound/                   # 音效文件（可选）
```

关键规则：
- **Mod 的 `scripts/` 会被加入 `require` 搜索路径**——你的 `scripts/components/mycomponent.lua` 可以被引擎自动找到
- **`modmain.lua` 运行在沙箱环境中**——访问全局变量需要 `GLOBAL.xxx`（见 1.5.7）
- **prefab 文件放在 `scripts/prefabs/` 下**，通过 `modmain.lua` 中的 `PrefabFiles = {"my_item"}` 注册
- **目录结构要和原版保持一致**——组件放 `components`、状态图放 `stategraphs`，这样引擎的自动加载机制才能正确工作

### 2.1.10 文件之间的调用关系——理解全局图景

最后，用一张关系图帮你理解这些文件如何协作：

```
游戏启动
  └─→ main.lua
       ├─→ class.lua（注册 Class 函数）
       ├─→ vector3.lua（注册 Vector3）
       ├─→ mainfunctions.lua（注册 SpawnPrefab 等核心函数）
       ├─→ scheduler.lua（注册调度器）
       ├─→ tuning.lua（注册 TUNING 全局表）
       ├─→ strings.lua（注册 STRINGS 全局表）
       └─→ mods.lua（加载所有 Mod）
            └─→ 每个 Mod 的 modmain.lua

加载世界
  └─→ gamelogic.lua
       ├─→ prefablist.lua（获取所有 prefab 文件名）
       └─→ 遍历 LoadPrefabFile("prefabs/" .. name)
            └─→ 每个 prefab 文件
                 ├─→ require("components/xxx")（按需加载组件）
                 ├─→ require("stategraphs/SGxxx")（加载状态图）
                 └─→ require("brains/xxxbrain")（加载 AI）

游戏运行中
  └─→ update.lua（每帧调用）
       ├─→ RunScheduler()（执行定时任务和协程）
       ├─→ SGManager:Update()（更新所有状态图）
       └─→ BrainManager:Update()（更新所有 AI）

用户交互
  └─→ actions.lua（定义动作）
       └─→ entityscript.lua（实体执行动作）
            └─→ components/（组件处理动作）
```

> **给初学者的建议**：刚开始不需要理解所有文件。做 Mod 时最常接触的是 `prefabs/`（定义新东西）、`components/`（定义新行为）和 `modmain.lua`（注册修改）。其他文件在你需要时再深入学习。

> **给老鸟的提示**：饥荒的架构是一种**数据驱动**的设计。`tuning.lua` 集中管理数值、`recipes.lua` 集中管理配方、`containers.lua` 集中管理容器布局——修改游戏不需要改动逻辑代码，只需要修改这些数据配置。这种设计极大地方便了 Mod 开发：很多时候你只需要改一行 `TUNING.xxx = yyy` 就能达到目的。

---

> **本节小结**
> - 饥荒的 `scripts` 目录约 3900+ 个 Lua 文件，按功能分为 14 个子目录
> - `prefabs/`（1544 文件）定义游戏中所有实体的模板——最大的目录
> - `components/`（792 文件）是可复用的行为模块，通过组合为实体赋予能力
> - `stategraphs/` 控制动画状态转换，`brains/` + `behaviours/` 控制 AI 决策
> - `widgets/` + `screens/` 构成用户界面，`map/` 管理世界生成
> - 根目录文件是全局定义和核心系统，包括启动流程、数据配置、网络通信和工具库
> - Mod 的目录结构应**镜像原版**（`scripts/prefabs/`、`scripts/components/` 等），方便引擎自动加载
> - 开发 Mod 最常接触的目录：`prefabs/`、`components/`、根目录的配置文件

## 2.2 游戏启动流程：main.lua → gamelogic.lua → mainfunctions.lua


理解游戏的启动流程，就像了解一台机器是怎么开机的——知道了每一步做了什么，你就知道你的 Mod 代码在什么时机被加载、能访问到哪些资源。这对于排查"为什么我的 Mod 报错了"或"为什么某个变量是 nil"至关重要。

### 2.2.1 启动的大局观——从 C++ 到 Lua

饥荒联机版是一个 C++ 引擎 + Lua 脚本的混合架构。C++ 引擎负责底层的渲染、物理、网络、音频等；Lua 负责几乎所有的游戏逻辑。启动流程可以简化为：

```
C++ 引擎启动
  │
  ├── 初始化渲染、物理、网络等底层系统
  ├── 暴露 C API 给 Lua（TheSim、TheNet、TheInput 等）
  │
  └── 加载并执行 scripts/main.lua  ← 这是 Lua 世界的入口
       │
       ├── 第一阶段：基础设施搭建
       ├── 第二阶段：Mod 加载与 Prefab 注册
       ├── 第三阶段：前端（主菜单）启动
       │
       └── 玩家点击"开始游戏"
            │
            └── gamelogic.lua 接管
                 ├── 加载/生成世界
                 ├── 注册 Prefab 到引擎
                 └── 进入游戏主循环（update.lua）
```

### 2.2.2 main.lua——Lua 世界的创世纪

`main.lua` 是饥荒 Lua 层的**绝对入口**——C++ 引擎启动后执行的第一个 Lua 文件。它的工作像是在盖楼之前打地基。

**第一步：配置模块搜索路径**

```lua
-- 来自 scripts/main.lua 第 1-2 行
package.path = "scripts\\?.lua;scriptlibs\\?.lua"
package.assetpath = {}
table.insert(package.assetpath, {path = ""})
```

这决定了 `require` 去哪里找文件。所有后续的 `require` 调用都依赖这个配置。

**第二步：安装自定义加载器**

```lua
-- 来自 scripts/main.lua（简化）
local loadfn = function(modulename)
    local modulepath = string.gsub(modulename, "[%.\\]", "/")
    for path in string.gmatch(package.path, "([^;]+)") do
        local filename = string.gsub(path, "%?", modulepath)
        local result = kleiloadlua(filename)
        if result then return result end
    end
end
table.insert(package.loaders, 2, loadfn)
```

饥荒用自己的 `kleiloadlua` 替代了 Lua 标准的文件加载方式（因为游戏资源可能在压缩包中）。

**第三步：按顺序加载核心模块**

这是 `main.lua` 最长的部分——大量的 `require` 调用，按照精心设计的顺序加载。顺序很重要，因为后面的模块可能依赖前面的：

```lua
-- 来自 scripts/main.lua —— 核心加载顺序

-- 第一批：基础工具
require("strict")          -- 严格模式：访问未声明的全局变量会报错
require("debugprint")      -- 调试输出
require("config")          -- 游戏配置
require("vector3")         -- Vector3 类（全局注册）
require("mainfunctions")   -- 核心函数（SpawnPrefab、LoadPrefabFile 等）

-- 第二批：核心系统
require("mods")            -- Mod 管理系统
require("tuning")          -- 全局数值表
require("strings")         -- 所有游戏文本
require("constants")       -- 全局常量
require("class")           -- Class 系统
require("util")            -- 工具函数

-- 第三批：游戏框架
require("actions")         -- 所有交互动作
require("scheduler")       -- 协程调度器
require("stategraph")      -- 状态图基类
require("behaviourtree")   -- 行为树基类
require("prefabs")         -- Prefab 和 Asset 类
require("entityscript")    -- EntityScript 基类

-- 第四批：数据与配置
require("recipes")         -- 合成配方
require("prefablist")      -- prefab 文件名列表
require("networking")      -- 网络工具
require("update")          -- 帧更新函数
```

> **给 Mod 开发者的重要提示**：这个加载顺序解释了为什么你在 `modmain.lua` 里可以使用 `Class`、`TUNING`、`STRINGS` 等——因为它们在 Mod 加载之前就已经被 `main.lua` 注册好了。

**第四步：声明全局变量**

`main.lua` 声明了大量的全局变量，这些是游戏运行时最重要的"共享状态"：

```lua
-- 来自 scripts/main.lua
Prefabs = {}             -- 所有已注册的 prefab
Ents = {}                -- 当前世界中的所有实体
AwakeEnts = {}           -- 所有醒着的（活跃的）实体
UpdatingEnts = {}        -- 需要每帧更新的实体

-- 全局单例对象（初始化为 nil，后续启动时赋值）
global("TheWorld")       -- 当前世界实体
global("ThePlayer")      -- 本地玩家实体
global("TheFrontEnd")    -- 前端 UI 管理器
global("TheCamera")      -- 摄像机
global("AllPlayers")     -- 所有玩家列表
AllPlayers = {}
```

**第五步：Mod 安全启动**

在加载完所有核心模块后，`main.lua` 最后调用 `ModSafeStartup()`：

```lua
-- 来自 scripts/main.lua
local function ModSafeStartup()
    -- 加载所有 Mod
    ModManager:LoadMods()

    -- 重新应用翻译（Mod 可能添加了新字符串）
    TranslateStringTable(STRINGS)

    -- 注册核心 prefab
    LoadPrefabFile("prefabs/global")
    LoadPrefabFile("prefabs/event_deps")

    -- 初始化各种全局数据系统
    TheRecipeBook = require("quagmire_recipebook")()
    TheCookbook = require("cookbookdata")()
    ThePlantRegistry = require("plantregistrydata")()
    TheSkillTree = require("skilltreedata")()

    -- 创建全局实体和管理器
    TheGlobalInstance = CreateEntity("TheGlobalInstance")
    TheCamera = FollowCamera()
    ShadowManager = TheGlobalInstance.entity:AddShadowManager()
    -- ...
end

-- Mod 的加载流程
if not MODS_ENABLED then
    ModSafeStartup()
else
    KnownModIndex:Load(function()
        KnownModIndex:BeginStartupSequence(function()
            ModSafeStartup()
        end)
    end)
end
```

这里有个重要的设计：Mod 的加载被包在 `ModSafeStartup` 中，如果 Mod 导致启动崩溃，下次可以禁用所有 Mod 重试。

### 2.2.3 Mod 的加载时机——你的代码何时执行

Mod 的加载发生在 `ModManager:LoadMods()` 中，时间线是这样的：

```
main.lua 开始执行
  ├── 加载 class.lua、tuning.lua 等核心模块
  ├── 声明全局变量
  │
  ├── ModSafeStartup()
  │     ├── ModManager:LoadMods()
  │     │     ├── 遍历所有启用的 Mod
  │     │     ├── 对每个 Mod：
  │     │     │     ├── 把 Mod 的 scripts/ 加入 package.path
  │     │     │     ├── 执行 modworldgenmain.lua（世界生成相关）
  │     │     │     └── 执行 modmain.lua ← 你的 Mod 代码在这里执行
  │     │     └── 注册所有 Mod 的 Prefab
  │     │
  │     ├── 注册核心 Prefab（global, event_deps）
  │     └── 初始化全局系统
  │
  └── 启动前端（主菜单）
```

这意味着：
- 你的 `modmain.lua` 执行时，`Class`、`TUNING`、`STRINGS`、`ACTIONS` 等已经可用
- 但 `TheWorld`、`ThePlayer` 还是 `nil`——因为世界还没有被创建
- 你在 `modmain.lua` 中注册的 `AddPrefabPostInit` 等回调，会在后续加载 prefab 时被调用

### 2.2.4 gamelogic.lua——从菜单到游戏世界

当玩家在主菜单点击"开始游戏"或"加入服务器"时，`gamelogic.lua` 接管控制权。它负责**加载或生成世界、注册所有 prefab、进入游戏**。

核心流程如下：

```lua
-- 来自 scripts/gamelogic.lua（简化后的核心逻辑）

-- 遍历 prefablist.lua 中的所有 prefab 文件名，逐个加载
for i, file in ipairs(PREFABFILES) do
    LoadPrefabFile("prefabs/" .. file)
end

-- 让 Mod 也注册它们的 prefab
ModManager:RegisterPrefabs()
```

`LoadPrefabFile` 在 `mainfunctions.lua` 中定义（1.5.5 已讲过），它加载 prefab 文件、执行构建函数、注册到引擎。

`gamelogic.lua` 还会区分两种场景：
- **前端（Frontend）**：主菜单、设置界面、服务器列表
- **后端（Backend）**：实际的游戏世界

```lua
-- 来自 scripts/gamelogic.lua
if Settings.current_asset_set == "FRONTEND" then
    -- 加载前端资源
    TheSim:LoadPrefabs(FRONTEND_PREFABS)
else
    -- 加载后端（游戏世界）资源
    TheSim:LoadPrefabs(BACKEND_PREFABS)
    TheSim:LoadPrefabs(RECIPE_PREFABS)
end
```

### 2.2.5 mainfunctions.lua——核心工具函数

`mainfunctions.lua` 并不是一个"启动阶段"，而是一个**工具库**，定义了启动和运行过程中反复使用的核心函数。它很早就被 `main.lua` require 了。

最重要的几个函数：

**`SpawnPrefab(name)`——生成实体**

```lua
-- 这是游戏中使用最频繁的函数之一
local axe = SpawnPrefab("axe")           -- 生成一把斧头
axe.Transform:SetPosition(10, 0, 20)     -- 放到坐标 (10, 0, 20)
```

**`LoadPrefabFile(filename)`——加载 prefab 文件**

```lua
-- 在启动时被大量调用
function LoadPrefabFile(filename)
    local fn = loadfile(filename)
    local ret = {fn()}   -- 执行文件，收集返回的 Prefab
    for i, val in ipairs(ret) do
        if val:is_a(Prefab) then
            RegisterSinglePrefab(val)
        end
    end
end
```

**`RegisterPrefabs(...)`——注册 prefab 到引擎**

把 Lua 层的 Prefab 对象注册到 C++ 引擎，这样引擎才能在需要时实例化它们。

### 2.2.6 update.lua——游戏的心跳

一旦进入游戏世界，`update.lua` 中定义的更新函数就开始被引擎**每帧调用**。饥荒的帧率是 30 FPS（即每秒 30 tick），每一帧做以下事情：

```lua
-- 来自 scripts/update.lua —— 主更新函数（简化）
function Update(dt)
    local tick = TheSim:GetTick()

    -- 1. 执行调度器（协程和定时任务）
    for i = last_tick_seen + 1, tick do
        RunScheduler(i)
    end

    -- 2. 更新所有需要每帧更新的组件
    for k, v in pairs(UpdatingEnts) do
        if v.updatecomponents then
            for cmp in pairs(v.updatecomponents) do
                if cmp.OnUpdate then
                    cmp:OnUpdate(dt)
                end
            end
        end
    end

    -- 3. 更新所有状态图
    for i = last_tick_seen + 1, tick do
        SGManager:Update(i)
    end

    -- 4. 更新所有 AI
    for i = last_tick_seen + 1, tick do
        BrainManager:Update(i)
    end

    last_tick_seen = tick
end
```

每一帧的执行顺序：

```
引擎帧循环（C++ 侧，30 FPS）
  │
  ├── WallUpdate(dt)          ← 总是执行（即使暂停时）
  │     ├── HandleRPCQueue()  ← 处理网络远程调用
  │     └── 前端 UI 更新
  │
  └── Update(dt)              ← 仅在游戏未暂停时执行
        ├── RunScheduler()    ← 协程和定时任务
        ├── 组件 OnUpdate     ← 所有活跃组件的帧更新
        ├── SGManager:Update  ← 状态图更新
        └── BrainManager:Update ← AI 更新
```

还有一个 `WallUpdate`，它在**服务器暂停时也会运行**（用于处理网络包和 UI），而 `Update` 只在未暂停时运行。

> **给新手的提示**：你在做 Mod 时，大部分时间不需要关心更新循环的细节。组件的 `OnUpdate` 会在你调用 `inst:StartUpdatingComponent(self)` 后自动被加入循环，调用 `inst:StopUpdatingComponent(self)` 后自动移除。

### 2.2.7 从头到尾的完整时间线

把所有环节串起来，这是饥荒从启动到进入游戏的完整时间线：

```
1.【引擎启动】C++ 引擎初始化

2.【main.lua】Lua 世界初始化
   ├── 配置路径和加载器
   ├── 加载核心模块（class, vector3, entityscript, scheduler...）
   ├── 加载数据（tuning, strings, recipes, constants...）
   ├── 声明全局变量（Ents, Prefabs, TheWorld=nil, ThePlayer=nil...）
   ├── 加载 Mod（ModManager:LoadMods → 执行每个 modmain.lua）
   ├── 注册核心 Prefab
   └── 启动前端（显示主菜单）

3.【主菜单阶段】玩家在 UI 中操作
   ├── gamelogic.lua 加载前端资源
   └── 玩家选择"创建世界"或"加入服务器"

4.【gamelogic.lua】加载游戏世界
   ├── 加载所有 prefab 文件（遍历 PREFABFILES）
   ├── Mod 的 prefab 也在此注册
   ├── 加载或生成世界地图
   ├── 创建 TheWorld 实体
   └── 生成玩家角色（ThePlayer）

5.【游戏运行中】update.lua 每帧驱动
   ├── 调度器：协程、定时任务
   ├── 组件更新：每帧逻辑
   ├── 状态图：动画状态机
   └── AI：行为树决策

6.【离开游戏】返回主菜单或退出
   ├── 保存世界数据
   ├── 清理实体和资源
   └── 回到步骤 3 或关闭
```

### 2.2.8 对 Mod 开发者的实际意义

理解启动流程后，你能回答以下常见问题：

**Q：为什么在 modmain.lua 里访问 TheWorld 是 nil？**
A：因为 modmain.lua 在 main.lua 的 `ModSafeStartup()` 中执行，此时世界还没有创建。你需要在 `AddPrefabPostInit` 或 `AddGamePostInit` 等回调中才能访问 TheWorld。

**Q：为什么我修改了 TUNING 的值但没生效？**
A：检查你的修改时机。如果某个组件在初始化时读取了 TUNING 值并缓存了，你在之后修改 TUNING 不会影响已缓存的值。解决方案：使用 `AddPrefabPostInit` 在 prefab 创建后直接修改组件的值。

**Q：我的 Mod 加了一个新 prefab，但游戏里找不到？**
A：确认你在 modmain.lua 中声明了 `PrefabFiles = {"my_prefab_name"}`，并且文件在 `scripts/prefabs/my_prefab_name.lua`。`PrefabFiles` 列表中的名字不需要路径前缀和 `.lua` 后缀。

**Q：多个 Mod 的加载顺序是什么？**
A：Mod 按照 `modinfo.lua` 中的 `priority` 值排序加载。priority 越高越先加载。如果你的 Mod 需要在另一个 Mod 之后运行，设置更低的 priority。

---

> **本节小结**
> - 饥荒是 C++ 引擎 + Lua 脚本的混合架构，所有游戏逻辑在 Lua 中
> - `main.lua` 是 Lua 的入口，按精确顺序加载核心模块、声明全局变量、加载 Mod
> - Mod 的 `modmain.lua` 在 `main.lua` 的 `ModSafeStartup` 阶段执行，此时核心模块可用但世界未创建
> - `gamelogic.lua` 在玩家进入游戏时接管，加载所有 prefab 并创建世界
> - `mainfunctions.lua` 提供核心工具函数（`SpawnPrefab`、`LoadPrefabFile` 等）
> - `update.lua` 定义了每帧更新函数，驱动调度器、组件、状态图和 AI
> - 了解时间线能帮助你正确使用 Mod API——知道什么时候哪些东西可用

## 2.3 class.lua——饥荒的面向对象系统实现

（待编写）

## 2.4 实体-组件-系统（ECS）架构概述

（待编写）

## 2.5 EntityScript——所有实体的基类（entityscript.lua 核心 API）

（待编写）

## 2.6 客户端/服务端（C/S）架构——为什么有两份代码

（待编写）

## 2.7 游戏循环：帧更新、定时任务与事件驱动

（待编写）

## 2.8 scheduler.lua——任务调度器与协程管理的核心

（待编写）

## 2.9 Shard 系统——多世界（地上/洞穴）通信

（待编写）

## 2.10 核心全局对象：TheWorld、ThePlayer、TheSim、TheInput、TheNet

（待编写）

## 2.11 定时任务详解：DoTaskInTime、DoPeriodicTask、Timer 组件

（待编写）

## 2.12 STRINGS 系统——游戏文本的组织结构与访问方式

（待编写）

## 2.13 数据驱动设计模式——tuning.lua / recipes.lua / containers.lua 的设计哲学

（待编写）

## 2.14 游戏模式：Survival / Endless / Wilderness / Lavaarena / Forge 的差异

（待编写）
