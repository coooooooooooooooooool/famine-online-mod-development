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

在 1.4 节中，我们从 Lua 语言的角度学习了元表和 `Class` 函数的基本原理。这一节换个视角——从**游戏架构设计**的角度，深入分析 `class.lua` 为什么这样设计、它在整个游戏系统中扮演什么角色，以及做 Mod 时如何充分利用这个系统。

### 2.3.1 为什么饥荒需要自己的 Class 系统

Lua 本身不提供面向对象语法。饥荒团队（Klei）面临一个选择：要么用纯函数式/过程式编程，要么自己造一套面向对象系统。他们选择了后者，原因很实际：

1. **游戏有大量相似但又有差异的实体**——所有武器都有攻击力和耐久，但具体数值不同。用类继承来共享通用逻辑、在子类中定制差异，比复制粘贴代码高效得多。

2. **组件需要统一的接口**——几百个组件都要有 `OnSave`、`OnLoad`、`GetDebugString` 等方法，需要一个一致的"类"结构来约束。

3. **状态图、行为树等系统有天然的继承层次**——`DecoratorNode` 继承 `BehaviourNode`，`Recipe2` 继承 `Recipe`，这些用 Class 系统实现最自然。

### 2.3.2 class.lua 的完整源码解读

`class.lua` 只有约 260 行代码，但支撑了整个游戏。让我们逐段剖析它的完整实现（1.4 中只看了简化版，这里看完整版）。

**ClassRegistry——热重载的基础设施**

```lua
-- 来自 scripts/class.lua 第 6 行
ClassRegistry = {}
```

每个通过 `Class` 创建的类都会被注册到 `ClassRegistry` 中，记录它从基类继承了哪些方法。这是为**热重载（hot reload）**准备的——开发者在运行时修改代码后，引擎能知道哪些方法是继承来的、哪些是类自己定义的，从而正确地更新。

```lua
-- 来自 scripts/class.lua 第 205 行
ClassRegistry[c] = c_inherited
```

`c_inherited` 记录的是从基类浅拷贝过来的方法。当热重载时，系统对比 `c` 的当前方法和 `c_inherited` 中的版本，就能分辨哪些方法是子类**重写**的（和 inherited 不同）、哪些是直接继承的（和 inherited 相同）。

> **给老鸟的提示**：热重载逻辑在 `reload.lua` 中实现。它利用 `ClassRegistry` 来做"猴子补丁"——重新加载一个模块后，把新版本的方法覆盖到旧的类原型表上。这样运行中的实例不需要重新创建就能获得新的方法实现。这是一个纯开发期的功能，生产环境中不使用。

**props 系统——__index 和 __newindex 的分支**

```lua
-- 来自 scripts/class.lua 第 102-109 行
if props ~= nil then
    c.__index = __index        -- 自定义的 __index，拦截读取
    c.__newindex = __newindex   -- 自定义的 __newindex，拦截赋值
else
    c.__index = c              -- 标准模式：实例找不到方法就去类原型找
end
```

这是一个关键的分支：

- **没有 props**（绝大多数情况）：`__index = c`，标准的原型查找，性能最好
- **有 props**（如 health 组件）：使用自定义的 `__index` 和 `__newindex`，每次读写都经过拦截函数

为什么有 props 时要改变 `__index`？因为 props 系统把被监听的属性存储在一个隐藏的 `_` 表中（而不是直接存在实例上），`__index` 需要先去 `_` 表里找属性值：

```lua
-- 来自 scripts/class.lua 第 28-34 行
local function __index(t, k)
    local p = rawget(t, "_")[k]    -- 先看 _ 表里有没有这个属性
    if p ~= nil then
        return p[1]                 -- 有就返回属性值（p[1] 是值，p[2] 是回调）
    end
    return getmetatable(t)[k]       -- 没有就走正常的原型链查找
end
```

这个设计在性能和功能之间做了取舍：大多数类不需要属性监听，走最快的路径；少数需要网络同步的核心组件才使用 props。

### 2.3.3 is_a、is_class、is_instance——三种类型检查

`class.lua` 为每个类提供了三个类型检查工具：

```lua
-- 来自 scripts/class.lua 第 197-202 行
c.is_a = _is_a               -- 实例方法：obj:is_a(SomeClass)
c.is_class = _is_class         -- 判断自身是类还是实例
c.is_instance = function(obj)  -- 静态方法：SomeClass.is_instance(obj)
    return type(obj) == "table" and _is_a(obj, c)
end
```

它们的用法和区别：

```lua
local h = Health(inst)

-- is_a：检查一个对象是否是某个类（或其子类）的实例
h:is_a(Health)         -- true
h:is_a(Combat)         -- false

-- is_instance：从类的角度检查（不需要对象引用）
Health.is_instance(h)  -- true
Combat.is_instance(h)  -- false

-- is_class：判断是类本身还是实例
Health:is_class()      -- true（Health 是类定义，有 is_instance 方法）
h:is_class()           -- false（h 是实例，没有 is_instance 方法）
```

`is_instance` 的实际用途——在不确定对象类型时安全地判断：

```lua
-- 来自 scripts/mainfunctions.lua —— 检查 LoadPrefabFile 的返回值
for i, val in ipairs(ret) do
    if type(val) == "table" and val.is_a and val:is_a(Prefab) then
        RegisterSinglePrefab(val)
    end
end
```

这里先检查 `val` 是不是 table，再检查有没有 `is_a` 方法，最后才调用——层层防御，避免在非 Class 对象上调用 `is_a` 导致报错。

### 2.3.4 makereadonly 和 addsetter——运行时属性控制

`class.lua` 还提供了两个强大但较少使用的工具：

**`makereadonly(t, k)`——把属性变为只读**

```lua
-- 来自 scripts/class.lua 第 51-61 行
function makereadonly(t, k)
    local _ = rawget(t, "_")
    assert(_ ~= nil, "Class does not support read only properties")
    local p = _[k]
    if p == nil then
        _[k] = { t[k], onreadonly }   -- 把当前值存入 _ 表，设回调为 onreadonly
        rawset(t, k, nil)              -- 从实例上删除原始属性
    else
        p[2] = onreadonly              -- 替换回调为只读检查
    end
end
```

`onreadonly` 回调在赋值时断言新值必须等于旧值——如果不等就报错：

```lua
local function onreadonly(t, v, old)
    assert(v == old, "Cannot change read only property")
end
```

**`addsetter(t, k, fn)`——给属性添加自定义 setter**

```lua
function addsetter(t, k, fn)
    local _ = rawget(t, "_")
    local p = _[k]
    if p == nil then
        _[k] = { t[k], fn }
        rawset(t, k, nil)
    else
        p[2] = fn
    end
end
```

这允许你在运行时**动态地**给一个已创建的对象添加属性监听器。而 props 参数是在类定义时静态设置的。

**`removesetter(t, k)`——移除属性监听器**

```lua
function removesetter(t, k)
    local _ = rawget(t, "_")
    if _ ~= nil and _[k] ~= nil then
        rawset(t, k, _[k][1])   -- 把值从 _ 表移回实例上
        _[k] = nil               -- 删除监听
    end
end
```

> **给 Mod 开发者的提示**：`addsetter` 和 `removesetter` 可以用于在 Mod 中给已有组件的属性添加自定义监听逻辑。但要注意，只有使用了 props 参数创建的类才支持这些操作（因为需要 `_` 表的存在）。

### 2.3.5 Class 系统在游戏中的层次结构

整个饥荒的代码可以看做一个类层次结构树：

```
Class (class.lua)
├── EntityScript (entityscript.lua)          ← 所有实体的基类
├── Vector3 (vector3.lua)                    ← 三维向量
├── Prefab (prefabs.lua)                     ← Prefab 定义
├── Brain (brain.lua)                        ← AI 大脑基类
│   ├── AbigailBrain                         ← 阿比盖尔的 AI
│   ├── SpiderBrain                          ← 蜘蛛的 AI
│   └── ...
├── BehaviourNode (behaviourtree.lua)        ← 行为树节点基类
│   ├── DecoratorNode                        ← 装饰器
│   ├── ConditionNode                        ← 条件节点
│   ├── SequenceNode                         ← 序列节点
│   └── ...
├── StateGraph (stategraph.lua)              ← 状态图定义
├── StateGraphInstance                       ← 状态图运行实例
├── State / EventHandler / ActionHandler     ← 状态图组件
│
├── 组件（无继承，独立定义）
│   ├── Health                               ← 带 props
│   ├── Combat                               ← 无 props
│   ├── Inventory                            ← 无 props
│   └── ...（约 400+ 个）
│
├── Widget (widgets/widget.lua)              ← UI 基类
│   ├── Text                                 ← 文本
│   ├── Image                                ← 图片
│   ├── Button                               ← 按钮
│   └── ...
├── Screen (screens/screen.lua)              ← 界面基类
│   ├── MainScreen                           ← 主菜单
│   ├── PlayerHud                            ← 游戏 HUD
│   └── ...
│
└── 工具类
    ├── BT (behaviourtree.lua)               ← 行为树容器
    ├── Scheduler / Task / Periodic          ← 调度系统
    ├── Recipe / Recipe2                     ← 合成配方（Recipe2 继承 Recipe）
    └── ...
```

注意一个有趣的事实：**绝大多数组件之间没有继承关系**。`Health` 不继承自某个"BaseComponent"，`Combat` 也不继承自 `Health`。它们都是独立的 `Class(function(self, inst) ... end)` 定义。

这是**组合优于继承**的设计哲学——不是通过继承树来复用代码，而是通过给实体**组合不同的组件**来构建功能。一个实体有 `Health`（能受伤）+ `Combat`（能打架）+ `Locomotor`（能移动）= 一个可战斗的生物。

少数例外是确实需要继承的场景：
- `Recipe2` 继承 `Recipe`——在原版配方基础上扩展新功能
- `AOEWeapon_Leap` 继承 `AOEWeapon_Base`——范围武器的变体
- `WobyRack` 继承 `DryingRack`——沃比的移动晾肉架

### 2.3.6 MetaClass——另一种 Class 实现

饥荒中还有一个 `metaclass.lua`，它提供了另一种面向对象的实现方式——基于 `newproxy` 的 userdata 对象：

```lua
-- 来自 scripts/metaclass.lua
mt.__call = function(class_tbl, ...)
    local obj = newproxy(true)        -- 创建 userdata（不是 table！）
    local objmt = getmetatable(obj)
    metatable_refs[obj] = objmt       -- 防止 GC 顺序问题
    objmt._ = {}                      -- 属性存储
    objmt.c = c                       -- 类引用
    for k, v in pairs(metafunctions) do
        objmt[k] = v                  -- 安装所有元方法
    end
    if c._ctor then
        c._ctor(obj, ...)
    end
    return obj
end
```

`MetaClass` 和 `Class` 的核心区别：

| 特性 | `Class` | `MetaClass` |
|------|---------|-------------|
| 实例类型 | table | userdata |
| 属性存储 | 直接在 table 上 | 存在 metatable._ 中 |
| 直接访问属性 | `rawget(obj, key)` | 不可能（userdata 不支持）|
| 内存占用 | 较大 | 较小（userdata 比 table 轻量）|
| 用途 | 几乎所有游戏对象 | 特殊的轻量对象 |

`MetaClass` 创建的对象是 userdata，外部代码无法绕过元方法直接读写属性。这提供了更好的封装性，但代价是灵活性降低。

> **给 Mod 开发者的提示**：你在 Mod 中基本不会用到 `MetaClass`。如果你遇到一个对象 `type(obj)` 返回 `"userdata"` 但它表现得像 table（有方法可以调用），那它可能就是 `MetaClass` 创建的。这时你不能用 `rawget` 直接读它的属性。

### 2.3.7 热重载系统——开发者的利器

`class.lua` 配合 `reload.lua`，实现了一个开发期的**热重载**系统。当你修改了一个 Lua 文件并保存后，不需要重启游戏就能看到效果。

热重载的原理：

```lua
-- 来自 scripts/reload.lua（简化后的核心逻辑）
function hotswap(modname)
    local ok, oldmod = pcall(require, modname)  -- 获取旧模块

    package.loaded[modname] = nil                -- 清除缓存
    local newmod = require(modname)              -- 重新加载

    -- 把新版本的方法更新到旧版本的表上
    if type(oldmod) == "table" then
        update(oldmod, newmod)
    end
end
```

关键在于 `update` 函数——它把新模块的方法逐个复制到旧模块的表上。由于所有实例都通过 `__index` 指向类原型表，更新类原型表就等于更新了所有实例的方法。

`ClassRegistry` 的 `c_inherited` 在这里派上了用场：当基类被热重载时，系统需要把新方法同步到所有子类。通过对比 `c_inherited`，系统知道子类的哪些方法是从基类继承的（需要更新），哪些是子类自己定义的（不应被覆盖）。

### 2.3.8 Mod 开发中的 Class 实战模式

**模式一：定义新组件（最常见）**

```lua
-- scripts/components/mycomponent.lua
local MyComponent = Class(function(self, inst)
    self.inst = inst
    self.power = 0
end)

function MyComponent:SetPower(val)
    self.power = val
    self.inst:PushEvent("powerchanged", {power = val})
end

function MyComponent:OnSave()
    return {power = self.power}
end

function MyComponent:OnLoad(data)
    if data then self.power = data.power or 0 end
end

return MyComponent
```

**模式二：扩展已有类（通过 PostInit）**

如果你想给已有的组件添加新方法，不需要继承——直接通过 Mod API 注入：

```lua
-- modmain.lua
AddComponentPostInit("combat", function(self)
    local old_GetAttacked = self.GetAttacked
    function self:GetAttacked(attacker, damage, ...)
        -- 在原有逻辑前做点什么
        print(self.inst.prefab .. " is being attacked!")
        -- 调用原版逻辑
        return old_GetAttacked(self, attacker, damage, ...)
    end
end)
```

这是饥荒 Mod 最常见的模式——**包装（wrapping）**原版方法。先保存原方法的引用，再定义新方法，在新方法中调用原方法。

**模式三：创建带继承的类**

```lua
-- scripts/components/my_special_weapon.lua
local AOEWeapon_Base = require("components/aoeweapon_base")

local MySpecialWeapon = Class(AOEWeapon_Base, function(self, inst)
    AOEWeapon_Base._ctor(self, inst)  -- 必须手动调用父类构造函数
    self.special_damage = 50
end)

function MySpecialWeapon:DoSpecialAttack(target)
    -- 先调用基类的方法
    self:DoAOEAttack(target)
    -- 再做额外的特殊处理
    target.components.health:DoDelta(-self.special_damage)
end

return MySpecialWeapon
```

---

> **本节小结**
> - `class.lua` 是饥荒全部面向对象代码的基础，约 260 行代码支撑了 3900+ 个文件
> - `ClassRegistry` 记录每个类的继承信息，为热重载提供支持
> - props 系统通过自定义 `__index`/`__newindex` 实现属性监听，只有需要网络同步的组件使用
> - `makereadonly`/`addsetter`/`removesetter` 提供运行时的属性控制能力
> - 饥荒的类层次以"组合优于继承"为设计哲学——组件独立定义，通过组合赋予实体能力
> - `MetaClass` 是另一种基于 userdata 的轻量实现，用于特殊场景
> - Mod 开发中最常用三种模式：定义新组件、包装已有方法、创建带继承的类

## 2.4 实体-组件-系统（ECS）架构概述

如果说 `class.lua` 是饥荒的"语法"，那么 **实体-组件** 架构就是饥荒的"语法结构"——它决定了游戏世界中每一样东西是如何被构造和组织的。理解这个架构，你就能理解为什么饥荒的代码长那个样子、为什么 Mod 的 API 是那样设计的。

### 2.4.1 什么是实体-组件架构

> **给新手的类比**：想象你在组装一台电脑。电脑本身就是一个"实体"（Entity）——一个空壳子。你往里面装 CPU（计算能力）、内存（临时存储）、硬盘（永久存储）、显卡（图形处理）。每个硬件就是一个"组件"（Component）。同一个机箱可以装不同的配置——游戏主机装好显卡，办公电脑可能不需要。

在饥荒中：
- **实体（Entity）**：游戏世界中的"一样东西"——一棵树、一块石头、一把斧头、一只蜘蛛、威尔逊本人，甚至世界本身
- **组件（Component）**：赋予实体特定能力的模块——生命值、战斗、移动、可被捡起、可燃烧等

一把斧头的组成：

```lua
-- 实体（空壳）
local inst = CreateEntity()

-- 添加组件（赋予能力）
inst:AddComponent("inventoryitem")   -- 能力：可以被捡起放进物品栏
inst:AddComponent("equippable")      -- 能力：可以被装备到手上
inst:AddComponent("tool")            -- 能力：可以用来砍/挖/锤等
inst:AddComponent("weapon")          -- 能力：可以用来攻击造成伤害
inst:AddComponent("finiteuses")      -- 能力：有耐久度，用完会坏
inst:AddComponent("inspectable")     -- 能力：可以被检查（右键查看描述）
```

一棵常青树的组成：

```lua
local inst = CreateEntity()
inst:AddComponent("inspectable")     -- 可以被检查
inst:AddComponent("workable")        -- 可以被工具作用（砍）
inst:AddComponent("lootdropper")     -- 被砍倒后掉落物品（原木）
inst:AddComponent("burnable")        -- 可以被点燃
inst:AddComponent("propagator")      -- 火焰可以蔓延到附近
inst:AddComponent("growable")        -- 会生长（小树 → 中树 → 大树）
```

注意到关键的区别：斧头有 `weapon`（能攻击）和 `finiteuses`（有耐久），树没有；树有 `growable`（会生长）和 `workable`（可被砍），斧头没有。**通过组合不同的组件，同一套代码可以创造出千变万化的实体**。

### 2.4.2 为什么用组件而不用继承

你可能会想：为什么不定义一个 `Tool` 类继承自 `Item`，`Weapon` 也继承自 `Item`？

问题是：一把**斧头**既是工具又是武器。如果你用继承，Lua 不支持多重继承（一个类只能有一个 `_base`）。即使语言支持，多重继承也会带来"菱形继承"等一系列问题。

```
继承方式（问题多多）：         组件方式（饥荒的选择）：

    Item                        Entity
   /    \                    + inventoryitem
  Tool  Weapon                + tool
   \    /                    + weapon
    ???（斧头该继承谁？）       + finiteuses
                              → 斧头！
```

组件系统的优势：
1. **灵活组合**——任何组件都可以自由搭配，不受继承层次限制
2. **独立修改**——修改 `weapon` 组件不影响 `tool` 组件
3. **运行时增删**——游戏运行中可以动态添加或移除组件（比如角色获得/失去某个能力）
4. **Mod 友好**——Mod 开发者可以用 `AddComponent` 给已有实体加新能力

### 2.4.3 饥荒的"两层"实体

饥荒的实体有两个层次——**C++ 层**（引擎实体）和 **Lua 层**（`EntityScript`）：

```lua
-- 创建实体时，两层同时建立
local inst = CreateEntity()

-- C++ 层的引擎组件（由 entity 对象管理）
inst.entity:AddTransform()       -- 位置/旋转/缩放
inst.entity:AddAnimState()       -- 动画状态
inst.entity:AddSoundEmitter()    -- 声音发射器
inst.entity:AddNetwork()         -- 网络同步
inst.entity:AddPhysics()         -- 物理碰撞

-- Lua 层的游戏组件（由 EntityScript 管理）
inst:AddComponent("health")      -- 生命值
inst:AddComponent("combat")      -- 战斗
inst:AddComponent("locomotor")   -- 移动
```

**C++ 层组件**负责底层的渲染、物理和网络，它们的数据和逻辑在 C++ 引擎中运行，性能极高。

**Lua 层组件**负责游戏逻辑——血量扣多少、攻击力是多少、移动速度快不快。这些是我们在 Mod 开发中主要打交道的部分。

C++ 层的组件通过 `inst.entity:AddXxx()` 添加，之后通过 `inst.Transform`、`inst.AnimState`、`inst.SoundEmitter` 等访问。Lua 层的组件通过 `inst:AddComponent("xxx")` 添加，之后通过 `inst.components.xxx` 访问。

### 2.4.4 AddComponent 的背后——组件如何被加载和挂载

当你调用 `inst:AddComponent("health")` 时，饥荒内部做了这些事：

```lua
-- 来自 scripts/entityscript.lua（简化版）
function EntityScript:AddComponent(name)
    -- 1. 检查是否已有同名组件
    if self.lower_components_shadow[string.lower(name)] ~= nil then
        return self.components[name]   -- 已经有了，直接返回
    end

    -- 2. 加载组件类（require("components/" .. name)）
    local cmp = LoadComponent(name)

    -- 3. 注册 replica 组件（如果有的话）
    self:ReplicateComponent(name)

    -- 4. 实例化组件，传入 self（实体）
    local loadedcmp = cmp(self)        -- 等价于 Health(self)

    -- 5. 存储到 components 表
    self.components[name] = loadedcmp

    -- 6. 执行 Mod 的 ComponentPostInit
    local postinitfns = ModManager:GetPostInitFns("ComponentPostInit", name)
    for _, fn in ipairs(postinitfns) do
        fn(loadedcmp, self)            -- 让 Mod 有机会修改这个组件
    end

    -- 7. 注册组件的可用动作
    self:RegisterComponentActions(name)

    return loadedcmp
end
```

注意第 6 步——这就是为什么 `AddComponentPostInit` 有效。每次一个组件被添加到任何实体上时，你注册的回调都会被调用。

### 2.4.5 组件之间如何通信——事件系统

组件是独立的模块，但它们经常需要相互配合。比如角色的 `health` 组件生命值归零时，需要通知 `combat` 停止战斗、`locomotor` 停止移动、`inventory` 掉落物品。

饥荒用**事件系统**来实现这种组件间通信：

```lua
-- 发送事件（"我死了！"）
self.inst:PushEvent("death", {cause = attacker})

-- 监听事件（"如果他死了，我要做点什么"）
self.inst:ListenForEvent("death", function(inst, data)
    inst.components.inventory:DropEverything()
end)
```

事件系统的实现原理：

```lua
-- 来自 scripts/entityscript.lua
function EntityScript:PushEvent_Internal(event, data)
    if self.event_listeners then
        local listeners = self.event_listeners[event]
        if listeners then
            -- 复制一份监听者列表再遍历（防止遍历过程中列表被修改）
            local tocall = {}
            for entity, fns in pairs(listeners) do
                for i, fn in ipairs(fns) do
                    table.insert(tocall, fn)
                end
            end
            for i, fn in ipairs(tocall) do
                fn(self, data)
            end
        end
    end

    -- 事件也会传递给状态图
    if self.sg then
        self.sg:PushEvent(event, data)
    end
end
```

**重要的设计选择**：事件在传递给 Lua 监听器后，还会传递给状态图（`self.sg`）。这让状态图可以响应任何组件触发的事件——比如 `health` 组件推送 `"attacked"` 事件后，状态图可以切换到"被打"的动画状态。

**你也可以监听另一个实体的事件**：

```lua
-- 在 inst 上监听 other_entity 的事件
inst:ListenForEvent("death", function(other, data)
    print(other.prefab .. " died!")
end, other_entity)   -- 第三个参数指定事件源
```

### 2.4.6 Tag 系统——轻量级的"标签"

除了组件和事件，饥荒还有一个**Tag 系统**——给实体打上标签，用于快速查询。

```lua
-- 添加标签
inst:AddTag("weapon")
inst:AddTag("sharp")

-- 检查标签
if inst:HasTag("weapon") then ... end
if inst:HasTag("sharp") then ... end

-- 移除标签
inst:RemoveTag("weapon")
```

Tag 是在 C++ 引擎层实现的，查询速度极快。组件通常在初始化时给实体添加 tag，让其他系统可以快速判断一个实体有什么能力，**而不需要检查它有哪些组件**。

为什么不直接检查组件？因为组件检查需要：

```lua
-- 慢：需要访问 Lua table
if inst.components.weapon ~= nil then

-- 快：tag 查询在 C++ 层完成
if inst:HasTag("weapon") then
```

更重要的是，在**客户端**上，你无法访问服务端的组件（组件只存在于服务端），但你可以检查 tag（tag 会自动同步到客户端）。这就是 tag 在 C/S 架构中的关键作用。

### 2.4.7 组件的生命周期

一个组件从创建到销毁，经历这些阶段：

```
1. 加载       LoadComponent(name) → require("components/name")
2. 实例化     cmp(inst) → 执行构造函数，初始化字段
3. PostInit   Mod 的 ComponentPostInit 回调
4. 运行       处理事件、执行方法、可选的 OnUpdate
5. 存档       OnSave() → 序列化数据
6. 读档       OnLoad(data) → 恢复数据
7. 移除       OnRemoveFromEntity() → 清理资源
```

**OnSave / OnLoad** 是饥荒存档系统的核心——你的组件需要把所有"重要状态"保存下来：

```lua
function MyComponent:OnSave()
    return {
        level = self.level,
        experience = self.experience,
        is_active = self.is_active,
    }
end

function MyComponent:OnLoad(data)
    if data then
        self.level = data.level or 1
        self.experience = data.experience or 0
        self.is_active = data.is_active or false
    end
end
```

**OnRemoveFromEntity** 在组件被动态移除时调用，用于清理资源（取消定时器、移除事件监听等）。

### 2.4.8 实体的查找——FindEntities

游戏运行中经常需要查找附近的实体（比如"找到周围 10 格内的所有可攻击生物"）。饥荒提供了 C++ 层的高效查找函数：

```lua
-- 查找指定范围内的实体
local x, y, z = inst.Transform:GetWorldPosition()
local entities = TheSim:FindEntities(x, y, z, 10,    -- 半径 10
    {"_combat"},        -- 必须有的 tag（must_tags）
    {"player", "wall"}, -- 不能有的 tag（cant_tags）
    {"monster", "pig"}  -- 至少有一个的 tag（oneof_tags）
)

for _, ent in ipairs(entities) do
    -- 对每个找到的实体做处理
end
```

注意 `FindEntities` 使用的是 **tag** 而不是组件名。这就是为什么每个组件在初始化时通常会给实体添加对应的 tag——让 `FindEntities` 能快速筛选。

### 2.4.9 用组件思维设计你的 Mod

理解了 ECS 架构后，来看如何应用到 Mod 开发中。

**思考方式**：不要想"我要做一个什么东西"，而是想"我要给这个东西什么能力"。

例如：你想做一个"魔法宝石"——
- 可以被捡起 → `inventoryitem` 组件
- 可以被装备 → `equippable` 组件
- 提供光环效果 → 自定义组件或 `inst:DoPeriodicTask` 检测周围
- 有耐久 → `finiteuses` 组件
- 用完后消失 → 在 finiteuses 用完的回调中 `inst:Remove()`

**Mod 中操作组件的常用方法**：

```lua
-- 给已有 prefab 添加新组件
AddPrefabPostInit("axe", function(inst)
    inst:AddComponent("waterproofer")  -- 让斧头防水
    inst.components.waterproofer:SetEffectiveness(0.5)
end)

-- 修改已有组件的参数
AddPrefabPostInit("wilson", function(inst)
    if inst.components.health then
        inst.components.health:SetMaxHealth(200)  -- 威尔逊血量改为 200
    end
end)

-- 动态添加/移除组件
inst:ListenForEvent("equip", function(inst, data)
    inst:AddComponent("nightvision")  -- 装备时获得夜视
end)
inst:ListenForEvent("unequip", function(inst, data)
    inst:RemoveComponent("nightvision")  -- 卸下时失去夜视
end)
```

---

> **本节小结**
> - 饥荒采用实体-组件架构：实体是空壳，组件赋予能力，通过组合构建功能
> - 组件系统的优势：灵活组合、独立修改、运行时增删、Mod 友好
> - 实体有两层：C++ 层（Transform、AnimState 等）和 Lua 层（health、combat 等）
> - `AddComponent` 内部：加载组件类 → 实例化 → 存入 components 表 → 执行 PostInit
> - 组件间通过**事件系统**（`PushEvent` / `ListenForEvent`）通信
> - **Tag 系统**是轻量级标签，在 C++ 层实现，查询快、可跨客户端/服务端使用
> - 组件生命周期：实例化 → 运行 → OnSave/OnLoad（存档）→ OnRemoveFromEntity（清理）
> - `FindEntities` 基于 tag 查找，是游戏中查找附近实体的标准方式

## 2.5 EntityScript——所有实体的基类（entityscript.lua 核心 API）

`EntityScript` 是饥荒联机版中**所有 Lua 实体对象的基类**。当你在 Mod 或 Prefab 代码中拿到一个 `inst`（不管它是角色、物品还是世界），它就是一个 `EntityScript` 的实例。本节将系统梳理它的核心 API，按功能分类，方便你在开发中速查。

### 2.5.1 EntityScript 的构造——一个实体是怎么诞生的

```lua
-- scripts/entityscript.lua
EntityScript = Class(function(self, entity)
    self.entity = entity              -- C++ 引擎实体对象
    self.components = {}              -- Lua 组件容器
    self.lower_components_shadow = {} -- 防重复添加（大小写无关）
    self.GUID = entity:GetGUID()      -- 全局唯一标识符
    self.spawntime = GetTime()        -- 出生时刻
    self.persists = true              -- 是否需要存档
    self.inlimbo = false              -- 是否在 limbo（不可见）状态
    self.name = nil                   -- 显示名称

    self.event_listeners = nil        -- 别人在"我"身上监听的事件
    self.event_listening = nil        -- "我"在别人身上监听的事件
    self.pendingtasks = nil           -- 待执行的定时任务
    self.children = nil               -- 子实体列表

    -- Replica 组件容器（客户端可访问的组件镜像）
    self.replica = { _ = {}, inst = self }
    setmetatable(self.replica, replica_mt)
end)
```

每个字段的设计都有特定目的：
- `entity`：连接 C++ 引擎层，通过它访问 `Transform`、`AnimState`、`Physics` 等底层组件
- `GUID`：引擎分配的唯一 ID，用于查找实体（`Ents[GUID]`）和绑定定时任务
- `persists`：设为 `false` 的实体不会被存档（临时特效、粒子等）
- `replica`：带自定义 `__index` 的 table，让 `inst.replica.xxx` 能自动解析 replica 组件

### 2.5.2 组件管理 API

#### AddComponent(name) / RemoveComponent(name)

```lua
-- 添加组件
local cmp = inst:AddComponent("health")

-- 内部流程：
-- 1. 检查是否已存在（大小写无关）→ 已有则直接返回
-- 2. require("components/" .. name) 加载组件类
-- 3. 实例化：cmp(self) → 传入实体作为 inst
-- 4. 存入 self.components[name]
-- 5. 调用所有 Mod 的 ComponentPostInit
-- 6. 注册该组件提供的 Action
```

```lua
-- 移除组件
inst:RemoveComponent("health")

-- 内部流程：
-- 1. 停止该组件的更新（如果有 OnUpdate）
-- 2. 从 self.components 中移除
-- 3. 调用 cmp:OnRemoveFromEntity()（清理回调）
-- 4. 清理 Replica 组件
-- 5. 注销该组件的 Action
```

**Mod 实践要点**：

```lua
-- 安全地访问组件（组件可能不存在）
if inst.components.health then
    inst.components.health:SetMaxHealth(200)
end

-- 添加组件前不需要检查（重复添加会直接返回已有组件）
inst:AddComponent("inventoryitem")

-- RemoveComponent 后，组件引用立即失效
inst:RemoveComponent("combat")
-- inst.components.combat 现在是 nil
```

#### StartUpdatingComponent(cmp) / StopUpdatingComponent(cmp)

并非所有组件都需要每帧执行逻辑。只有调用了 `StartUpdatingComponent` 的组件，其 `OnUpdate(dt)` 方法才会在每个游戏帧被调用。

```lua
-- 组件内部典型用法
function MyComponent:TurnOn()
    self.active = true
    self.inst:StartUpdatingComponent(self)  -- 开始每帧更新
end

function MyComponent:TurnOff()
    self.active = false
    self.inst:StopUpdatingComponent(self)   -- 停止每帧更新
end

function MyComponent:OnUpdate(dt)
    -- 每帧执行的逻辑
end
```

源码中，`StartUpdatingComponent` 会把实体注册到全局的 `UpdatingEnts` 表中，然后 `update.lua` 中的主循环每帧遍历该表，调用每个组件的 `OnUpdate`：

```lua
-- entityscript.lua 简化版
function EntityScript:StartUpdatingComponent(cmp)
    if not self.updatecomponents then
        self.updatecomponents = {}
        NewUpdatingEnts[self.GUID] = self     -- 加入全局更新列表
        num_updating_ents = num_updating_ents + 1
    end
    self.updatecomponents[cmp] = cmpname      -- 记录需要更新的组件
end
```

饥荒还有 **WallUpdate**（墙上时钟更新，不受暂停影响）和 **StaticUpdate**（不受游戏时间缩放影响），分别对应 `StartWallUpdatingComponent` 和 `DoStaticPeriodicTask`。

### 2.5.3 定时任务 API——DoTaskInTime 家族

这是 Mod 开发中最常用的 API 之一。EntityScript 提供了四个定时任务方法：

| 方法 | 说明 | 受时间暂停影响 | 执行次数 |
|---|---|---|---|
| `DoTaskInTime(time, fn, ...)` | 延迟执行 | 是 | 一次 |
| `DoPeriodicTask(time, fn, delay, ...)` | 周期执行 | 是 | 无限 |
| `DoStaticTaskInTime(time, fn, ...)` | 延迟执行 | 否 | 一次 |
| `DoStaticPeriodicTask(time, fn, delay, ...)` | 周期执行 | 否 | 无限 |

```lua
-- 3 秒后执行一次
inst:DoTaskInTime(3, function(inst)
    print(inst.prefab .. " 3 seconds have passed!")
end)

-- 每 5 秒执行一次（第 3 个参数是首次延迟，nil = 立即开始第一次计时）
local task = inst:DoPeriodicTask(5, function(inst)
    print(inst.prefab .. " periodic tick!")
end)

-- 取消某个特定任务
task:Cancel()

-- 取消该实体的所有任务
inst:CancelAllPendingTasks()
```

**Static vs 普通**的区别：普通任务使用游戏内时间（受暂停、时间缩放影响），Static 任务使用真实墙钟时间。UI 相关的定时器通常用 Static 版本。

**源码实现细节**——所有任务都注册到 `self.pendingtasks` 表中：

```lua
function EntityScript:DoTaskInTime(time, fn, ...)
    local periodic = scheduler:ExecuteInTime(time, fn, self.GUID, self, ...)
    self.pendingtasks = self.pendingtasks or {}
    self.pendingtasks[periodic] = true
    periodic.onfinish = task_finish   -- 完成后自动从 pendingtasks 移除
    return periodic
end
```

这意味着当实体被 `Remove()` 时，`CancelAllPendingTasks()` 会自动取消所有未完成的定时任务——不会出现"实体已删除但定时器还在跑"的问题。

**还有一个便利方法** `PushEventInTime`：

```lua
-- 3 秒后推送一个事件（相当于 DoTaskInTime + PushEvent 的合体）
inst:PushEventInTime(3, "timer_done", {source = "magic"})
```

### 2.5.4 事件系统 API

#### ListenForEvent(event, fn, source)

```lua
-- 监听自己身上的事件
inst:ListenForEvent("death", function(inst, data)
    print(inst.prefab .. " died!")
end)

-- 监听别的实体的事件（第三个参数指定事件源）
inst:ListenForEvent("attacked", function(target, data)
    print(target.prefab .. " was attacked!")
end, some_target)
```

事件系统使用**双向注册**——源实体记录"谁在监听我"（`event_listeners`），监听者记录"我在监听谁"（`event_listening`）。这样做是为了：
1. **推送事件时**：遍历源实体的 `event_listeners` 找到所有回调
2. **清理时**：遍历监听者的 `event_listening` 取消所有监听（防内存泄漏）

#### RemoveEventCallback(event, fn, source)

```lua
-- 取消监听（必须传入完全相同的 fn 引用）
local function on_death(inst, data) ... end
inst:ListenForEvent("death", on_death)
inst:RemoveEventCallback("death", on_death)
```

注意：因为需要传入**同一个函数引用**来取消，所以如果你需要取消监听，就不能使用匿名函数——必须把函数保存到一个变量中。

#### PushEvent(event, data) / PushEventImmediate(event, data)

```lua
-- 推送事件
inst:PushEvent("attacked", {attacker = attacker, damage = damage})

-- data 可以是任意 table，约定俗成传递相关信息
```

`PushEvent` 和 `PushEventImmediate` 的区别在于状态图的处理方式：
- `PushEvent`：状态图在当前帧末尾处理事件（经过 `SGManager` 调度）
- `PushEventImmediate`：状态图立即处理事件（`self.sg:HandleEvent` 直接调用）

大多数情况下使用 `PushEvent` 即可。`PushEventImmediate` 主要用于需要状态图**立刻**响应的场景。

### 2.5.5 位置与距离 API

```lua
-- 获取世界坐标
local pos = inst:GetPosition()               -- 返回 Point 对象
local x, y, z = inst.Transform:GetWorldPosition() -- 返回三个数字（更常用）

-- 获取朝向角度
local rot = inst:GetRotation()

-- 计算到某点的角度
local angle = inst:GetAngleToPoint(x, y, z)

-- 面朝某个点
inst:ForceFacePoint(x, y, z)   -- 无条件转向
inst:FacePoint(x, y, z)        -- 如果状态图在 "busy" 就不转

-- 距离计算（返回距离的平方，避免开方运算）
local dist_sq = inst:GetDistanceSqToInst(other)
local dist_sq = inst:GetDistanceSqToPoint(x, y, z)

-- 判断是否在某距离内（内部用 dist² < range² 比较，高效）
if inst:IsNear(other, 10) then
    print("within 10 units!")
end

-- 附近是否有玩家
if inst:IsNearPlayer(20, true) then  -- 20 格内，且玩家活着
    print("a living player is nearby")
end
```

**为什么用距离平方**？因为 `sqrt()` 是昂贵的运算。如果你只是比较"是否在范围内"，可以直接比较平方值：

```lua
-- 慢（需要开方）
if math.sqrt(dist_sq) < 10 then

-- 快（比较平方）
if dist_sq < 10 * 10 then
```

`IsNear` 方法内部就是这么做的：

```lua
function EntityScript:IsNear(otherinst, dist)
    return otherinst ~= nil and self:GetDistanceSqToInst(otherinst) < dist * dist
end
```

### 2.5.6 可见性与场景管理

#### Hide() / Show()

```lua
inst:Hide()   -- 隐藏实体（不可见，但仍在场景中）
inst:Show()   -- 重新显示
```

#### RemoveFromScene() / ReturnToScene()

这对方法比 `Hide/Show` 要"重"得多——它会把实体放入 **Limbo** 状态：

```lua
-- RemoveFromScene 做了这些事：
-- 1. 添加 "INLIMBO" tag
-- 2. 隐藏实体
-- 3. 停止 AI 脑（Brain）
-- 4. 停止状态图
-- 5. 关闭物理碰撞
-- 6. 暂停动画
-- 7. 关闭灯光、阴影、小地图图标
-- 8. 推送 "enterlimbo" 事件

inst:RemoveFromScene()

-- ReturnToScene 是完全相反的操作：
-- 恢复所有被停止的系统，推送 "exitlimbo" 事件
inst:ReturnToScene()
```

**典型应用场景**：物品被捡起放入物品栏时，调用 `RemoveFromScene()` 使其从地面消失；丢弃物品时调用 `ReturnToScene()` 使其重新出现。

### 2.5.7 状态图与 AI 脑

#### SetStateGraph(name)

```lua
-- 设置状态图（角色/生物的动画状态机）
inst:SetStateGraph("SGwilson")

-- 内部流程：
-- 1. 如果已有状态图，先从 SGManager 移除
-- 2. 加载状态图定义：require("stategraphs/" .. name)
-- 3. 创建 StateGraphInstance 并注册到 SGManager
-- 4. 进入默认状态
```

#### SetBrain(brainfn)

```lua
-- 设置 AI 脑（生物的行为决策）
inst:SetBrain(require("brains/spiderbrain"))

-- brainfn 是一个函数，调用后返回 Brain 实例
-- Brain 内部使用行为树（BehaviourTree）来做决策
```

Brain 有智能的启用/禁用机制：
- 实体进入睡眠状态（`IsAsleep`，即离玩家太远）→ Brain 被停用（节省性能）
- 实体唤醒 → Brain 重新创建并启动
- `StopBrain(reason)` / `RestartBrain(reason)` 支持多重原因（reason-based），只有所有原因都解除后 Brain 才会重启

```lua
-- 因为"frozen"而停止 AI
inst:StopBrain("frozen")

-- 因为"sleeping"也停止 AI
inst:StopBrain("sleeping")

-- 只解除"frozen"原因——AI 仍然不会启动（还有"sleeping"）
inst:RestartBrain("frozen")

-- 解除"sleeping"——所有原因都解除了，AI 重启
inst:RestartBrain("sleeping")
```

### 2.5.8 线程管理

```lua
-- 启动一个绑定到实体的线程
inst:StartThread(function()
    while true do
        print("doing something...")
        Sleep(1)
    end
end)

-- 杀死该实体的所有线程
inst:KillTasks()
```

`StartThread` 内部使用实体的 GUID 作为线程 ID，这样当实体被 `Remove()` 时，可以通过 `KillThreadsWithID(self.GUID)` 批量终止所有相关线程。

### 2.5.9 世界状态监听

```lua
-- 监听世界状态变化
inst:WatchWorldState("isday", function(inst, isday)
    if isday then
        print("It's daytime!")
    end
end)

-- 停止监听
inst:StopWatchingWorldState("isday", fn)

-- 停止所有世界状态监听
inst:StopAllWatchingWorldStates()
```

常用的世界状态变量包括：`isday`、`isdusk`、`isnight`、`season`、`temperature`、`moisture`、`cycles`（天数）等。这些由 `TheWorld.components.worldstate` 管理。

### 2.5.10 存档系统 API

#### GetPersistData() / SetPersistData(data, newents)

```lua
-- 存档时调用（自动遍历所有组件的 OnSave）
function EntityScript:GetPersistData()
    local data = {}
    for k, v in pairs(self.components) do
        if v.OnSave then
            local t, refs = v:OnSave()
            if type(t) == "table" and not IsTableEmpty(t) then
                data[k] = t
            end
        end
    end
    -- 还会调用实体自身的 OnSave
    if self.OnSave then
        self.OnSave(self, data)
    end
    return data, references
end
```

读档时调用链：
1. `SetPersistData(data, newents)` → 遍历所有组件的 `OnLoad(data[k])`
2. `LoadPostPass(newents, savedata)` → 遍历所有组件的 `LoadPostPass`

在你的 Prefab 中使用：

```lua
local function fn()
    local inst = CreateEntity()
    -- ...

    -- 自定义存档逻辑
    inst.OnSave = function(inst, data)
        data.my_custom_value = inst.my_value
    end

    inst.OnLoad = function(inst, data)
        if data then
            inst.my_value = data.my_custom_value
        end
    end

    return inst
end
```

#### LongUpdate(dt)

当玩家长时间离线或世界被暂停后恢复时，饥荒会调用 `LongUpdate` 来追赶错过的时间：

```lua
function EntityScript:LongUpdate(dt)
    -- 调用实体自身的 LongUpdate
    if self.OnLongUpdate ~= nil then
        self:OnLongUpdate(dt)
    end
    -- 调用所有组件的 LongUpdate
    for k, v in pairs(self.components) do
        if v.LongUpdate ~= nil then
            v:LongUpdate(dt)
        end
    end
end
```

### 2.5.11 实体的销毁——Remove()

`Remove()` 是实体的"死刑"，执行一系列彻底的清理操作：

```lua
function EntityScript:Remove()
    -- 1. 从父实体/平台上脱离
    if self.parent then self.parent:RemoveChild(self) end
    if self.platform then self.platform:RemovePlatformFollower(self) end

    -- 2. 从全局实体表中注销
    OnRemoveEntity(self.GUID)

    -- 3. 推送 "onremove" 事件（最后的通知机会）
    self:PushEvent("onremove")

    -- 4. 清理所有监听和任务
    self:StopAllWatchingWorldStates()
    self:RemoveAllEventCallbacks()
    self:CancelAllPendingTasks()

    -- 5. 通知所有组件和 replica 清理
    for k, v in pairs(self.components) do
        if v.OnRemoveEntity then v:OnRemoveEntity() end
    end

    -- 6. 从更新列表中移除
    -- 7. 递归移除所有子实体
    -- 8. 调用实体自身的 OnRemoveEntity
    -- 9. 标记不存档，交还给引擎
    self.persists = false
    self.entity:Retire()
end
```

**注意**：`Remove()` 是**不可逆**的。调用后实体永久失效。使用 `IsValid()` 检查实体是否仍然有效：

```lua
if inst:IsValid() then
    -- 实体还活着，可以安全操作
end
```

### 2.5.12 Replica 系统——客户端的镜像

在联机版中，组件只存在于服务端（`mastersim`）。客户端需要访问某些组件信息时，使用 **Replica** 组件：

```lua
-- 服务端：直接访问组件
if TheWorld.ismastersim then
    inst.components.health:SetMaxHealth(200)
end

-- 客户端：通过 replica 访问
local health_replica = inst.replica.health
if health_replica then
    local max_hp = health_replica:Max()
end
```

Replica 的内部机制依赖一个自定义元表：

```lua
local replica_mt = {
    __index = function(t, k)
        return rawget(t, "inst"):ValidateReplicaComponent(k, rawget(t, "_")[k])
    end,
}
```

访问 `inst.replica.health` 时，实际上是通过 `__index` 元方法去 `_` 子表中查找 replica 组件实例。`ValidateReplicaComponent` 会额外做合法性校验。

### 2.5.13 地形与平台 API

```lua
-- 判断是否在有效地面上（不包括船）
inst:IsOnValidGround()

-- 判断是否在可通过的点上（更通用）
inst:IsOnPassablePoint(include_water, floating_platforms_are_not_passable)

-- 判断是否在海洋上
inst:IsOnOcean(allow_boats)

-- 获取当前所在平台（船）
local platform = inst:GetCurrentPlatform()

-- 获取脚下的地形类型
local tile_type, tile_info = inst:GetCurrentTileType()

-- 如果掉到无效位置，自动移回陆地
inst:PutBackOnGround(radius)  -- 在 radius 范围内找最近的合法点
```

### 2.5.14 其他实用 API

#### Debuff（增减益效果）

```lua
-- 添加 debuff（如果没有 debuffable 组件会自动添加）
inst:AddDebuff("mymod_buff", "mymod_buff_prefab", {duration = 10})

-- 检查是否有某个 debuff
if inst:HasDebuff("mymod_buff") then

-- 移除 debuff
inst:RemoveDebuff("mymod_buff")
```

#### 显示名称

```lua
-- 获取显示名称（带形容词，如"潮湿的斧头"）
local name = inst:GetDisplayName()

-- 获取基础名称（不带形容词）
local name = inst:GetBasicDisplayName()
```

#### 子实体管理

```lua
-- 生成子实体（随父实体一起存档/删除）
local child = inst:SpawnChild("fireflies")

-- 手动添加/移除子实体
inst:AddChild(child)
inst:RemoveChild(child)
```

---

> **本节小结——EntityScript 核心 API 速查**
>
> | 类别 | 核心方法 |
> |---|---|
> | 组件管理 | `AddComponent`, `RemoveComponent`, `StartUpdatingComponent`, `StopUpdatingComponent` |
> | 定时任务 | `DoTaskInTime`, `DoPeriodicTask`, `DoStaticTaskInTime`, `DoStaticPeriodicTask`, `CancelAllPendingTasks` |
> | 事件系统 | `ListenForEvent`, `RemoveEventCallback`, `PushEvent`, `PushEventImmediate` |
> | 位置距离 | `GetPosition`, `GetDistanceSqToInst`, `IsNear`, `ForceFacePoint`, `IsNearPlayer` |
> | 场景管理 | `Hide/Show`, `RemoveFromScene/ReturnToScene`, `Remove`, `IsValid` |
> | 状态图/AI | `SetStateGraph`, `ClearStateGraph`, `SetBrain`, `StopBrain/RestartBrain` |
> | 存档系统 | `OnSave/OnLoad`（自定义）, `GetPersistData/SetPersistData`（框架调用）, `LongUpdate` |
> | 世界状态 | `WatchWorldState`, `StopWatchingWorldState` |
> | Tag | `AddTag`, `RemoveTag`, `HasTag`, `HasTags`, `HasOneOfTags` |
> | 地形平台 | `IsOnValidGround`, `IsOnPassablePoint`, `IsOnOcean`, `GetCurrentPlatform` |
> | Replica | `inst.replica.xxx`——客户端访问服务端组件信息的桥梁 |

## 2.6 客户端/服务端（C/S）架构——为什么有两份代码

饥荒联机版和单机版最本质的区别不是"能联机"这么简单——它从底层改变了整个游戏的运行方式。理解 C/S 架构，是从"能写 Mod"到"能写**稳定**的 Mod"的分水岭。

### 2.6.1 单机版 vs 联机版——到底改了什么

> **给新手的类比**：想象一个棋盘游戏。
> - **单机版**就像你一个人下棋——棋子在哪、轮到谁、规则判断，全都在你面前。
> - **联机版**就像网络对弈——只有**裁判**（服务端）知道完整的棋局状态。每个玩家（客户端）只能看到自己被允许看到的部分，所有操作都要发给裁判，由裁判判定后再通知各玩家结果。

在技术上：
- **服务端（Server / Master Simulation）**：运行所有游戏逻辑——战斗计算、血量变化、物品掉落、AI 决策、世界状态。它是游戏的"唯一真相"。
- **客户端（Client）**：负责渲染画面、播放动画和声音、显示 UI、处理玩家输入。它不运行大部分游戏逻辑，只展示服务端告诉它的结果。

```
┌───────────────────────────────┐
│         服务端 (Server)        │
│  ● 所有 components            │
│  ● 所有游戏逻辑               │
│  ● AI (Brain + BehaviourTree) │
│  ● 状态图 (StateGraph)        │
│  ● 存档 (Save/Load)           │
│  ● 权威判定                   │
└─────────────┬─────────────────┘
              │ 网络同步（net variables）
              ▼
┌───────────────────────────────┐
│         客户端 (Client)        │
│  ● 渲染 (AnimState)           │
│  ● UI 界面 (Widgets/Screens)  │
│  ● 声音 (SoundEmitter)        │
│  ● replica 组件（只读镜像）    │
│  ● 输入处理 → 发送给服务端     │
└───────────────────────────────┘
```

### 2.6.2 TheWorld.ismastersim——一行代码分两个世界

判断当前代码运行在服务端还是客户端，只需要一个变量：

```lua
if TheWorld.ismastersim then
    -- 这里的代码只在服务端运行
else
    -- 这里的代码只在客户端运行
end
```

在 Prefab 构造函数中，有一个**标准模式**贯穿几乎所有 prefab 文件：

```lua
-- scripts/prefabs/axe.lua（简化版）
local function fn()
    local inst = CreateEntity()

    -- ===== 公共部分：客户端和服务端都要执行 =====
    inst.entity:AddTransform()       -- 位置（两端都需要）
    inst.entity:AddAnimState()       -- 动画（两端都需要渲染）
    inst.entity:AddNetwork()         -- 标记为网络同步实体

    MakeInventoryPhysics(inst)       -- 物理设置
    inst.AnimState:SetBank("axe")
    inst.AnimState:SetBuild("axe")
    inst.AnimState:PlayAnimation("idle")

    inst:AddTag("sharp")             -- Tag（两端都可查询）

    inst.entity:SetPristine()        -- 标记"公共初始化完成"

    -- ===== 分界线 =====
    if not TheWorld.ismastersim then
        return inst   -- 客户端到此为止，直接返回
    end

    -- ===== 服务端专属：添加组件和游戏逻辑 =====
    inst:AddComponent("inventoryitem")
    inst:AddComponent("tool")
    inst:AddComponent("weapon")
    inst:AddComponent("finiteuses")

    inst.components.weapon:SetDamage(27)
    inst.components.finiteuses:SetMaxUses(100)

    return inst
end
```

**三段式结构**：
1. **公共部分**——`CreateEntity()` 到 `SetPristine()`：设置视觉表现（动画、物理、Tag），两端都执行
2. **分界线**——`if not TheWorld.ismastersim then return inst end`：客户端在这里离开
3. **服务端部分**——`AddComponent` 和所有游戏逻辑：只在服务端执行

### 2.6.3 为什么客户端不能有组件

你可能会问：为什么不让客户端也运行完整的组件？

1. **防作弊**——如果客户端自己计算伤害和血量，作弊者可以直接修改这些值
2. **一致性**——多个客户端各自计算的结果可能不同（浮点精度、时序差异等），只有一个权威来源才能保证所有人看到的是同一个游戏
3. **带宽**——同步所有组件的完整状态太昂贵。只同步必要的"展示信息"就够了

但客户端确实需要一些信息来渲染 UI——比如显示血条就需要知道当前血量。怎么办？这就是 Replica 和 Classified 的工作。

### 2.6.4 网络变量（Net Variables）——数据同步的基础管道

饥荒使用 **net_xxx** 类型的特殊变量来在服务端和客户端之间同步数据：

```lua
-- 常见的 net 变量类型
net_bool(guid, name, dirty_event)         -- 布尔值
net_byte(guid, name, dirty_event)         -- 0~255
net_tinybyte(guid, name, dirty_event)     -- 更小范围的整数
net_smallbyte(guid, name, dirty_event)    -- 小字节
net_ushortint(guid, name, dirty_event)    -- 无符号短整数
net_shortint(guid, name, dirty_event)     -- 有符号短整数
net_uint(guid, name, dirty_event)         -- 无符号整数
net_int(guid, name, dirty_event)          -- 有符号整数
net_float(guid, name, dirty_event)        -- 浮点数
net_string(guid, name, dirty_event)       -- 字符串
net_hash(guid, name, dirty_event)         -- 哈希值
net_entity(guid, name, dirty_event)       -- 实体引用
net_event(guid, name)                     -- 一次性事件触发
net_bytearray(guid, name, dirty_event)    -- 字节数组
net_smallbytearray(guid, name, dirty_event) -- 小字节数组
```

三个参数的含义：
- `guid`：属于哪个实体
- `name`：变量名（全局唯一标识）
- `dirty_event`：变量值改变时触发的事件名（可选）

使用方式：

```lua
-- 创建
self.myvalue = net_byte(inst.GUID, "mymod.myvalue", "myvaluedirty")

-- 服务端写入
self.myvalue:set(42)

-- 客户端读取
local val = self.myvalue:value()

-- 客户端监听变化（通过 dirty event）
inst:ListenForEvent("myvaluedirty", function(inst)
    local new_val = inst.myvalue:value()
    -- 更新 UI 等
end)
```

来看一个真实例子——`player_classified.lua` 中的生命值同步：

```lua
-- scripts/prefabs/player_classified.lua
-- 服务端创建网络变量
inst.currenthealth = net_ushortint(inst.GUID, "health.currenthealth", "healthdirty")
inst.maxhealth = net_ushortint(inst.GUID, "health.maxhealth", "healthdirty")
inst.healthpenalty = net_byte(inst.GUID, "health.penalty", "healthdirty")
inst.istakingfiredamage = net_bool(inst.GUID, "health.takingfiredamage", "istakingfiredamagedirty")
```

### 2.6.5 Classified 实体——网络数据的"信使"

你注意到上面的 net 变量定义在 `player_classified` 而不是玩家实体本身。为什么要多一个实体？

**Classified** 是饥荒设计的一种特殊实体，专门作为服务端和客户端之间的数据中转站。它：
- 是一个**隐藏的子实体**（`Hide()` + `AddTag("CLASSIFIED")`），玩家看不到
- 身上挂载了大量 `net_xxx` 变量
- 服务端往里面写数据，客户端从里面读数据

创建模式：

```lua
-- scripts/prefabs/inventoryitem_classified.lua（简化版）
local function fn()
    local inst = CreateEntity()

    inst.entity:AddNetwork()
    inst.entity:Hide()
    inst:AddTag("CLASSIFIED")

    -- 声明网络变量
    inst.image = net_hash(inst.GUID, "inventoryitem.image", "imagedirty")
    inst.atlas = net_hash(inst.GUID, "inventoryitem.atlas", "imagedirty")
    inst.canbepickedup = net_bool(inst.GUID, "inventoryitem.canbepickedup")

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        -- 客户端：通过 OnEntityReplicated 把自己挂到父实体上
        inst.OnEntityReplicated = function(inst)
            inst._parent = inst.entity:GetParent()
            inst._parent:TryAttachClassifiedToReplicaComponent(inst, "inventoryitem")
        end
        return inst
    end

    -- 服务端：提供设置数据的方法
    return inst
end
```

**为什么不直接在实体上挂 net 变量**？因为：
1. 一个实体的 net 变量数量有上限
2. 不同系统的数据（生命值、饥饿值、物品栏）可以分到不同的 classified 实体上，模块化管理
3. Classified 可以只发给相关玩家，减少不必要的网络流量

### 2.6.6 Replica 组件——统一的访问接口

客户端上没有 `inst.components.health`，但有 `inst.replica.health`。Replica 组件是一层**适配器**，提供统一的 API，内部根据运行环境自动选择数据来源：

```lua
-- scripts/components/health_replica.lua
function Health:GetPercent()
    if self.inst.components.health ~= nil then
        -- 服务端：直接读组件的真实值
        return self.inst.components.health:GetPercent()
    elseif self.classified ~= nil then
        -- 客户端：从 classified 的 net 变量读
        return self.classified.currenthealth:value()
             / self.classified.maxhealth:value()
    else
        -- 兜底：返回默认值
        return 1
    end
end
```

**同一个方法，两套实现路径**——在服务端走组件，在客户端走 classified。调用者不需要关心自己在哪端：

```lua
-- 这段代码在客户端和服务端都能正确工作
local health_percent = inst.replica.health:GetPercent()
```

#### Replica 的注册机制

哪些组件有 Replica？由 `entityreplica.lua` 中的白名单控制：

```lua
-- scripts/entityreplica.lua
local REPLICATABLE_COMPONENTS = {
    builder = true,
    combat = true,
    container = true,
    equippable = true,
    health = true,
    hunger = true,
    inventory = true,
    inventoryitem = true,
    sanity = true,
    stackable = true,
    -- ... 等等
}
```

当 `AddComponent("health")` 被调用时，框架自动检查白名单，如果命中就加载对应的 `health_replica.lua`：

```lua
-- scripts/entityreplica.lua
function EntityScript:ReplicateComponent(name)
    if not REPLICATABLE_COMPONENTS[name] then
        return  -- 不在白名单中，跳过
    end

    if TheWorld.ismastersim then
        self:AddTag("_" .. name)  -- 服务端加 tag，同步给客户端
    end

    -- 加载并实例化 replica
    local cmp = require("components/" .. name .. "_replica")
    rawset(self.replica._, name, cmp(self))
end
```

**Tag 的巧妙用法**：服务端给实体加 `"_health"` tag，这个 tag 同步到客户端后，客户端的 `ReplicateEntity()` 看到 tag 就知道要创建对应的 replica 组件。

**Mod 开发者**可以注册自己的 replica：

```lua
-- 在 modmain.lua 中
AddReplicableComponent("mymod_component")
```

### 2.6.7 完整的数据同步链路

把前面的概念串起来，一个完整的数据同步链路是这样的（以生命值为例）：

```
服务端 Health 组件                          客户端
  │                                         │
  │  health.currenthealth = 80              │
  │       │                                 │
  │       ▼                                 │
  │  player_classified                      │
  │  .currenthealth:set(80)                 │
  │       │                                 │
  │       │  ~~~~ 网络传输 ~~~~              │
  │       │                                 │
  │       ▼                                 │
  │  player_classified                      │
  │  .currenthealth:value() → 80            │
  │       │                                 │
  │       ▼                                 │
  │  health_replica                         │
  │  :GetPercent() → 从 classified 读取     │
  │       │                                 │
  │       ▼                                 │
  │  UI 血条更新                            │
```

### 2.6.8 AddNetwork 与 SetPristine——Prefab 的"网络化仪式"

在 Prefab 构造函数中，有两个关键调用：

**`inst.entity:AddNetwork()`**——把实体标记为"需要网络同步"。调用后：
- 引擎会把这个实体复制到所有客户端
- 可以在这个实体上使用 `net_xxx` 变量
- 会自动创建 `actionreplica`（用于同步动作组件信息）

**`inst.entity:SetPristine()`**——标记"公共初始化完成"。这个调用：
- 告诉引擎"到此为止的所有设置是两端共享的初始状态"
- 引擎用它来做初始化优化和预测

这两个调用的顺序是固定的，构成了 Prefab 的"网络化仪式"：

```lua
local inst = CreateEntity()
inst.entity:AddTransform()
inst.entity:AddAnimState()
inst.entity:AddNetwork()        -- 第一步：注册网络

-- ... 公共设置（动画、tag、物理等）

inst.entity:SetPristine()       -- 第二步：标记完成

if not TheWorld.ismastersim then
    return inst                 -- 客户端离场
end

-- ... 服务端专属逻辑
```

### 2.6.9 Mod 开发中的 C/S 注意事项

#### 最常见的错误：在客户端访问组件

```lua
-- 错误！在客户端上 inst.components.health 是 nil
AddPrefabPostInit("wilson", function(inst)
    print(inst.components.health:GetPercent())  -- 客户端崩溃！
end)

-- 正确做法
AddPrefabPostInit("wilson", function(inst)
    if not TheWorld.ismastersim then return end  -- 客户端跳过
    print(inst.components.health:GetPercent())
end)

-- 如果两端都需要获取血量
AddPrefabPostInit("wilson", function(inst)
    if TheWorld.ismastersim then
        -- 服务端：通过组件
        print(inst.components.health:GetPercent())
    else
        -- 客户端：通过 replica
        print(inst.replica.health:GetPercent())
    end
end)
```

#### Tag 是两端共享的

```lua
-- Tag 在客户端和服务端都可以查询
if inst:HasTag("weapon") then
    -- 两端都能执行到这里
end

-- 但 Tag 只应该在服务端修改
if TheWorld.ismastersim then
    inst:AddTag("mymod_tag")
end
```

#### 事件也分客户端和服务端

```lua
-- 服务端事件：由组件推送
inst:ListenForEvent("death", fn)         -- 只在服务端收到

-- 客户端事件：由 net variable dirty 触发
inst:ListenForEvent("healthdirty", fn)   -- 在客户端收到

-- 两端都有的事件（比如 "onremove"）
inst:ListenForEvent("onremove", fn)      -- 两端都会收到
```

#### 何时需要自己做网络同步

如果你的 Mod 需要从服务端向客户端传递自定义数据，需要：

```lua
-- 方法一：用 Tag（简单的开关状态）
if TheWorld.ismastersim then
    inst:AddTag("mymod_active")    -- 自动同步到客户端
end

-- 方法二：用 net variable（数值/字符串数据）
-- 在公共部分（SetPristine 之前）创建
inst.mymod_level = net_byte(inst.GUID, "mymod.level", "mymod_leveldirty")

-- 服务端写入
if TheWorld.ismastersim then
    inst.mymod_level:set(5)
end

-- 客户端读取/监听
if not TheWorld.ismastersim then
    inst:ListenForEvent("mymod_leveldirty", function(inst)
        local level = inst.mymod_level:value()
        -- 更新显示
    end)
end
```

#### 快速判断一个 API 在哪端可用

| 可用性 | API |
|---|---|
| **两端** | `inst:HasTag()`, `inst.Transform`, `inst.AnimState`, `inst.replica.xxx`, `inst:GetPosition()`, `inst:IsValid()` |
| **仅服务端** | `inst.components.xxx`, `inst:AddComponent()`, `inst:PushEvent("xxx")` (组件事件), `inst.sg` (状态图) |
| **仅客户端** | UI 操作 (`TheFrontEnd`, Widgets), 输入处理, `net_xxx:value()` dirty 事件 |

---

> **本节小结**
> - 饥荒联机版采用严格的 C/S 架构：服务端是"唯一真相"，客户端只负责展示
> - `TheWorld.ismastersim` 区分当前代码运行在服务端还是客户端
> - Prefab 使用"三段式"构造：公共部分 → `SetPristine` → 服务端专属
> - **net_xxx 变量**是服务端和客户端之间的数据通道，服务端 `set()`，客户端 `value()`
> - **Classified 实体**是隐藏的子实体，集中存放 net 变量，作为数据中转站
> - **Replica 组件**提供统一接口，自动根据运行环境选择组件或 classified 作为数据来源
> - Tag 两端共享、组件仅服务端、net dirty 事件主要在客户端监听
> - Mod 开发中最常见的错误是在客户端访问 `inst.components.xxx`——这在客户端是 `nil`

## 2.7 游戏循环：帧更新、定时任务与事件驱动

游戏本质上是一个永不停歇的**循环**——每秒执行固定次数，每次检查输入、更新逻辑、渲染画面。理解游戏循环，就理解了饥荒中"什么时候执行什么代码"的根本规律。

### 2.7.1 游戏循环是什么

> **给新手的类比**：想象一个钟表匠制作的机械时钟。
> - 发条每秒转动 30 下（**Tick**），每转一下，所有齿轮都跟着动一格
> - 时针齿轮（**组件更新**）每秒动 30 次，检查时间该不该变
> - 报时齿轮（**定时任务**）平时不动，到了整点才响
> - 闹钟齿轮（**事件**）被设定了特定条件，满足了就触发

在饥荒中：
- **帧（Frame）**：游戏以 **30 帧/秒** 运行，每帧间隔约 0.033 秒
- **Tick**：引擎维护的帧计数器，每帧 +1
- **帧更新（Update）**：每帧执行一次的代码
- **定时任务（Task）**：到指定时间后执行的代码
- **事件（Event）**：条件满足时立即触发的代码

```lua
-- constants.lua
FRAMES = 1/30   -- 一帧的时间，约 0.0333 秒
```

### 2.7.2 三种驱动模式的对比

饥荒中的代码有三种运行方式：

| 驱动模式 | 触发条件 | 典型场景 | API |
|---|---|---|---|
| **帧更新** | 每帧自动执行 | 位移插值、动画同步、持续检测 | `OnUpdate(dt)` |
| **定时任务** | 到达指定时间 | 延迟执行、周期检查 | `DoTaskInTime`, `DoPeriodicTask` |
| **事件驱动** | 特定条件触发 | 受伤响应、死亡处理、拾取 | `ListenForEvent`, `PushEvent` |

它们各有优缺点：

```lua
-- 帧更新：每帧都跑，适合平滑的持续行为
-- 缺点：性能开销大，能不用就不用
function Locomotor:OnUpdate(dt)
    -- 每帧更新角色位置
    self:RunForward(dt)
end

-- 定时任务：到时间才跑，适合间隔性行为
-- 缺点：时间精度受帧率限制
inst:DoPeriodicTask(5, function(inst)
    -- 每 5 秒检查一次周围
    local nearby = TheSim:FindEntities(...)
end)

-- 事件驱动：条件满足才跑，完全不浪费性能
-- 缺点：依赖事件被正确推送
inst:ListenForEvent("attacked", function(inst, data)
    -- 被攻击时才执行
    inst.components.combat:SetTarget(data.attacker)
end)
```

**性能优先级**：事件驱动 > 定时任务 > 帧更新。能用事件就用事件，能用定时任务就不用帧更新。

### 2.7.3 Update 函数——游戏主循环的心脏

饥荒的主循环定义在 `scripts/update.lua` 中。C++ 引擎每帧调用 `Update(dt)`，它是整个 Lua 层面的驱动核心：

```lua
-- scripts/update.lua（简化注释版）
function Update(dt)
    HandleClassInstanceTracking()    -- 类实例跟踪（调试用）

    local tick = TheSim:GetTick()

    -- 第 1 步：运行调度器（执行到期的定时任务和协程）
    for i = last_tick_seen + 1, tick do
        RunScheduler(i)
    end

    -- 第 2 步：更新全局静态组件
    for k, v in pairs(StaticComponentUpdates) do
        v(dt)
    end

    -- 第 3 步：更新所有注册了 OnUpdate 的组件
    for k, v in pairs(UpdatingEnts) do
        if v.updatecomponents then
            for cmp in pairs(v.updatecomponents) do
                if cmp.OnUpdate and not StopUpdatingComponents[cmp] then
                    cmp:OnUpdate(dt)
                end
            end
        end
    end

    -- 处理新注册和取消注册的组件
    -- （本帧新注册的下帧才开始更新，防止迭代异常）

    -- 第 4 步：更新状态图
    for i = last_tick_seen + 1, tick do
        SGManager:Update(i)      -- 状态图更新
        BrainManager:Update(i)   -- AI 脑更新
    end

    last_tick_seen = tick
end
```

执行顺序非常重要：

```
每帧执行顺序：
┌──────────────────────────────────────┐
│ 1. RunScheduler                       │  ← 定时任务、协程
│ 2. StaticComponentUpdates             │  ← 全局静态更新
│ 3. UpdatingEnts → cmp:OnUpdate(dt)    │  ← 组件帧更新
│ 4. SGManager:Update                   │  ← 状态图
│ 5. BrainManager:Update                │  ← AI 脑
└──────────────────────────────────────┘
```

这意味着：**定时任务先于组件更新，组件更新先于状态图，状态图先于 AI**。

### 2.7.4 WallUpdate——不受暂停影响的更新

除了 `Update`，还有 `WallUpdate`——它使用**墙钟时间**（真实时间），即使游戏暂停也会运行：

```lua
-- scripts/update.lua
function WallUpdate(dt)
    -- 即使游戏暂停也执行的内容：
    HandleRPCQueue()           -- RPC 消息队列
    HandleUserCmdQueue()       -- 用户命令

    -- WallUpdate 组件
    for k, v in pairs(WallUpdatingEnts) do
        for cmp in pairs(v.wallupdatecomponents) do
            if cmp.OnWallUpdate then
                cmp:OnWallUpdate(dt)
            end
        end
    end

    TheMixer:Update(dt)        -- 音频混合
    TheCamera:Update(dt)       -- 相机
    TheInput:OnUpdate()        -- 输入
    TheFrontEnd:Update(dt)     -- UI 前端
end
```

**WallUpdate 的典型用户**：UI 动画、输入处理、音频系统——这些即使游戏暂停也需要响应。

### 2.7.5 StaticUpdate——不受时间缩放影响的更新

还有一个 `StaticUpdate`，它不受游戏时间缩放影响（比如游戏加速），也会在暂停时处理特定逻辑：

```lua
-- scripts/update.lua
function StaticUpdate(dt)
    -- 运行 static 调度器
    for i = last_static_tick_seen + 1, static_tick do
        RunStaticScheduler(i)
    end

    -- 暂停时仍更新 static 组件
    if TheNet:IsServerPaused() then
        for k, v in pairs(StaticUpdatingEnts) do
            for cmp in pairs(v.updatecomponents) do
                if cmp.OnStaticUpdate then
                    cmp:OnStaticUpdate(0)
                end
            end
        end
        -- 暂停时仍处理状态图事件
        SGManager:UpdateEvents()
    end
end
```

### 2.7.6 帧更新的详细机制

#### 组件如何注册帧更新

不是所有组件都有帧更新——只有主动注册了的才会每帧执行：

```lua
-- 在组件内部
function MyComponent:StartUpdating()
    self.inst:StartUpdatingComponent(self)
end

function MyComponent:StopUpdating()
    self.inst:StopUpdatingComponent(self)
end

function MyComponent:OnUpdate(dt)
    -- 这个方法在 StartUpdatingComponent 之后，每帧调用
    -- dt 是帧间隔（约 0.033 秒）
end
```

底层实现是把实体和组件注册到全局的 `UpdatingEnts` 表中。`Update` 函数每帧遍历这个表。

#### NewUpdatingEnts——延迟注册

注意源码中有一个细节：

```lua
-- 本帧新注册的组件不会立即被遍历
NewUpdatingEnts[self.GUID] = self     -- 暂存

-- 遍历完毕后才合并
if next(NewUpdatingEnts) ~= nil then
    for k, v in pairs(NewUpdatingEnts) do
        UpdatingEnts[k] = v           -- 下帧才生效
    end
    NewUpdatingEnts = {}
end
```

这是为了**安全**——如果在遍历 `UpdatingEnts` 的过程中直接添加新元素，会导致不可预测的迭代行为。

#### StopUpdatingComponents——延迟注销

同样，停止更新也不是立即生效的：

```lua
function EntityScript:StopUpdatingComponent(cmp)
    if self.updatecomponents or self.updatestaticcomponents then
        StopUpdatingComponents[cmp] = self   -- 标记，稍后处理
    end
end
```

在 `Update` 的遍历中，被标记的组件会被跳过（`not StopUpdatingComponents[cmp]`），遍历结束后才真正移除。

### 2.7.7 定时任务的执行时机

`DoTaskInTime` 和 `DoPeriodicTask` 本质上是向 Scheduler 注册一个协程，在指定的 tick 唤醒：

```lua
-- 当你写
inst:DoTaskInTime(1, my_function)

-- 等价于：
-- "在第 current_tick + 30 个 tick 时（1 秒后），唤醒一个协程来执行 my_function"
```

这些任务在 `Update` 的第一步 `RunScheduler(i)` 中被处理——**先于组件更新**。

```lua
function RunScheduler(tick)
    scheduler:OnTick(tick)   -- 唤醒到期的协程
    scheduler:Run()          -- 执行唤醒的协程
end
```

这意味着如果你在一个 `DoTaskInTime` 回调中修改了某个组件的数据，同一帧内该组件的 `OnUpdate` 会看到修改后的值。

### 2.7.8 事件的执行时机

事件（`PushEvent`）是**同步**执行的——调用 `PushEvent` 时，所有监听者的回调**立即**被调用，然后代码才继续往下走：

```lua
-- PushEvent 是同步的
print("before")
inst:PushEvent("myevent", {value = 42})
print("after")  -- 所有 myevent 的监听器都执行完了才到这里
```

但有一个特殊情况：状态图对事件的响应是**延迟**的。`PushEvent_Internal` 中状态图不是直接处理事件，而是放入事件队列，等 `SGManager:Update` 时才处理：

```lua
function EntityScript:PushEvent_Internal(event, data, immediate)
    -- Lua 监听器：立即执行
    if self.event_listeners then
        for entity, fns in pairs(listeners) do
            for i, fn in ipairs(fns) do fn(self, data) end
        end
    end

    -- 状态图：放入队列（除非 immediate = true）
    if self.sg then
        if immediate then
            self.sg:HandleEvent(event, data)        -- 立即
        elseif self.sg:IsListeningForEvent(event) then
            self.sg:PushEvent(event, data)          -- 排队
        end
    end
end
```

### 2.7.9 LongUpdate——追赶错过的时间

当发生时间跳跃时（跳过夜晚、从洞穴返回、服务器长时间暂停后恢复），游戏调用 `LongUpdate(dt)` 让所有实体"补课"：

```lua
-- scripts/update.lua
function LongUpdate(dt, ignore_player)
    -- 全局静态组件的 LongUpdate
    for k, v in pairs(StaticComponentLongUpdates) do
        v(dt)
    end

    -- 所有实体的 LongUpdate
    for k, v in pairs(Ents) do
        v:LongUpdate(dt)
    end
end
```

`ignore_player` 为 `true` 时，会跳过玩家及其物品，只让世界"老化"（典型场景：进洞穴后地上世界的植物需要继续生长，但玩家的物品不应该变化）。

在组件中实现 `LongUpdate`：

```lua
function Growable:LongUpdate(dt)
    -- 在 dt 秒内，跳过的时间里应该生长了多少
    if self.stage and self.growtime then
        local stages_to_advance = math.floor(dt / self.growtime)
        for i = 1, stages_to_advance do
            self:DoGrowth()
        end
    end
end
```

### 2.7.10 PostUpdate——帧末尾的清理

在 `Update` 之后，引擎还会调用 `PostUpdate`：

```lua
function PostUpdate(dt)
    EmitterManager:PostUpdate()    -- 粒子发射器
    UpdateLooper_PostUpdate()      -- PostUpdate 回调
end
```

`UpdateLooper` 组件提供了一种比 `DoPeriodicTask(0)` 更精确的每帧回调机制（因为 `DoPeriodicTask(0)` 不保证严格每帧触发一次）。

### 2.7.11 用一张图理清完整的帧执行顺序

```
C++ 引擎每帧调用：

┌─── WallUpdate(dt) ─────────────────────────┐
│  RPC 队列处理                               │
│  WallUpdate 组件（不受暂停影响）             │
│  音频、相机、输入、UI 更新                   │
└─────────────────────────────────────────────┘
           ↓
┌─── StaticUpdate(dt) ───────────────────────┐
│  RunStaticScheduler（Static 定时任务）       │
│  暂停时的 Static 组件更新                    │
│  暂停时的状态图事件                          │
└─────────────────────────────────────────────┘
           ↓ （仅在未暂停时）
┌─── Update(dt) ─────────────────────────────┐
│  ① RunScheduler → 定时任务 / 协程           │
│  ② StaticComponentUpdates                   │
│  ③ UpdatingEnts → cmp:OnUpdate(dt)          │
│  ④ SGManager:Update → 状态图               │
│  ⑤ BrainManager:Update → AI 脑            │
└─────────────────────────────────────────────┘
           ↓
┌─── PostUpdate(dt) ─────────────────────────┐
│  粒子系统                                   │
│  PostUpdate 回调                            │
└─────────────────────────────────────────────┘
           ↓
┌─── C++ 引擎 ───────────────────────────────┐
│  物理模拟                                   │
│  渲染                                       │
│  网络同步                                   │
└─────────────────────────────────────────────┘
```

### 2.7.12 Mod 开发的实践指导

#### 选择合适的驱动模式

```lua
-- 场景 1：角色被攻击时反击 → 事件驱动（最高效）
inst:ListenForEvent("attacked", function(inst, data)
    inst.components.combat:SetTarget(data.attacker)
end)

-- 场景 2：每 10 秒恢复 1 点生命 → 定时任务
inst:DoPeriodicTask(10, function(inst)
    if inst.components.health then
        inst.components.health:DoDelta(1)
    end
end)

-- 场景 3：角色头上的光环需要平滑移动 → 帧更新
function AuraComponent:OnUpdate(dt)
    local x, y, z = self.inst.Transform:GetWorldPosition()
    self.aura_fx.Transform:SetPosition(x, y + 2, z)
end

-- 场景 4：需要在暂停时也能操作的 UI → WallUpdate
function UIWidget:OnWallUpdate(dt)
    self:UpdateAnimation(dt)
end
```

#### 注意帧更新的性能开销

```lua
-- 不要这样做——大量实体每帧更新会严重拖慢性能
-- 100 个实体各有 3 个帧更新组件 = 每帧 300 次 OnUpdate 调用

-- 更好的做法：用定时任务替代不需要精确帧同步的逻辑
inst:DoPeriodicTask(0.5, CheckNearby)    -- 每 0.5 秒检查一次就够了

-- 或者：只在需要时开启帧更新
function MyComponent:Activate()
    self.active = true
    self.inst:StartUpdatingComponent(self)
end

function MyComponent:Deactivate()
    self.active = false
    self.inst:StopUpdatingComponent(self)   -- 不需要时关掉
end
```

#### dt 参数的含义

```lua
function MyComponent:OnUpdate(dt)
    -- dt 是自上一帧以来的时间（秒），通常约 0.033
    -- 使用 dt 而不是硬编码值，让逻辑在任何帧率下都正确

    -- 正确：
    self.timer = self.timer - dt
    self.position = self.position + self.speed * dt

    -- 错误：
    self.timer = self.timer - 0.033    -- 假设帧率固定是危险的
end
```

---

> **本节小结**
> - 饥荒以 30 帧/秒运行，每帧依次执行：WallUpdate → StaticUpdate → Update → PostUpdate
> - **三种驱动模式**：帧更新（每帧）、定时任务（到时间）、事件驱动（条件触发），性能优先级依次升高
> - `Update` 内部顺序：调度器 → 静态组件 → 组件 OnUpdate → 状态图 → AI 脑
> - `WallUpdate` 不受暂停影响，主要服务 UI、输入、音频
> - `StaticUpdate` 不受时间缩放影响，处理 static 调度器和暂停时的状态图事件
> - `LongUpdate` 用于时间跳跃后"补课"（跳夜、进出洞穴）
> - 组件帧更新需要主动注册（`StartUpdatingComponent`），注册和注销都是延迟生效
> - 事件是同步执行的（除了状态图的事件响应是延迟到 SGManager:Update）
> - Mod 中应优先使用事件驱动，其次定时任务，最后帧更新

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
