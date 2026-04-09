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

（待编写）

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
