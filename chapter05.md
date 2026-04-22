# 第5章 实体与 Prefab 系统

## 5.1 什么是 Prefab？从"图纸"到"实例"

### 本节导读

Prefab 是饥荒联机版中最核心的概念之一——游戏世界中你看到的一切东西（树木、石头、怪物、武器、玩家角色）都是 Prefab 的实例。理解 Prefab 系统，是开发任何有实质内容的 Mod 的前提。

> **新手**可以从「什么是 Prefab、它和实体有什么关系」开始，建立直观理解；**进阶读者**可以深入 Prefab 类的定义、`SpawnPrefab` 的完整流程、以及 PostInit 钩子的注入时机；**老手**可以直接跳到引擎交互层，了解 `SpawnPrefabFromSim`、`CreateEntity`、`Ents` 全局表等底层机制。

---

### 5.1.1 快速入门：Prefab 是什么？

用一句话解释：**Prefab 是"图纸"，实体（Entity）是"成品"**。

打个比方：
- **Prefab** = 一张长矛的设计图纸——它定义了长矛长什么样、有什么属性、能做什么
- **实体** = 你背包里的那把长矛——它是按照图纸制造出来的具体物品

在代码中，这个"从图纸到成品"的过程就是 **`SpawnPrefab("spear")`**——传入图纸名称，返回一个具体的实体。

游戏世界中的每一棵树、每一块石头、每一只蜘蛛，都是由对应的 Prefab 图纸生成的实体。同一张图纸可以生成无数个实体——地图上可能有几百棵 `"evergreen"`（常青树），但它们都来自同一份 Prefab 定义。

---

### 5.1.2 快速入门：一个真实的 Prefab 文件

让我们看一个最简单的官方 Prefab——**木头**（`scripts/prefabs/log.lua`）：

```lua
-- scripts/prefabs/log.lua 完整代码

local assets =
{
    Asset("ANIM", "anim/log.zip"),
}

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("log")
    inst.AnimState:SetBuild("log")
    inst.AnimState:PlayAnimation("idle")

    inst:AddTag("log")
    inst.pickupsound = "wood"

    MakeInventoryFloatable(inst, "med", 0.1, 0.75)

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("tradable")
    inst:AddComponent("edible")
    inst.components.edible.foodtype = FOODTYPE.WOOD
    inst.components.edible.healthvalue = 0
    inst.components.edible.hungervalue = 0

    inst:AddComponent("fuel")
    inst.components.fuel.fuelvalue = TUNING.MED_FUEL

    MakeSmallBurnable(inst, TUNING.MED_BURNTIME)
    MakeSmallPropagator(inst)
    MakeHauntableLaunchAndIgnite(inst)

    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")
    inst:AddComponent("stackable")

    inst:AddComponent("repairer")
    inst.components.repairer.repairmaterial = MATERIALS.WOOD
    inst.components.repairer.healthrepairvalue = TUNING.REPAIR_LOGS_HEALTH

    return inst
end

return Prefab("log", fn, assets)
```

**这 60 行代码定义了游戏中"木头"的一切**。让我们拆解它的结构：

| 部分 | 代码 | 作用 |
|------|------|------|
| **资源声明** | `local assets = { Asset("ANIM", "anim/log.zip") }` | 告诉引擎需要加载哪些美术资源 |
| **工厂函数** | `local function fn() ... end` | 定义如何创建一个木头实体 |
| **Prefab 注册** | `return Prefab("log", fn, assets)` | 把图纸注册到系统中 |

工厂函数 `fn()` 是核心——每次 `SpawnPrefab("log")` 被调用时，引擎就会执行这个函数来创建一个新的木头实体。

> **新手记忆**：一个 Prefab 文件 = 资源列表 + 工厂函数 + Prefab 注册。文件最后 `return Prefab(名称, 工厂函数, 资源列表)` 是固定格式。

---

### 5.1.3 进阶：Prefab 类的定义

`Prefab` 是一个 Lua 类，定义在 `scripts/prefabs.lua` 中：

```lua
-- scripts/prefabs.lua 第 6-19 行
Prefab = Class( function(self, name, fn, assets, deps, force_path_search)
    self.name = string.sub(name, string.find(name, "[^/]*$"))
    self.desc = ""
    self.fn = fn
    self.assets = assets or {}
    self.deps = deps or {}
    self.force_path_search = force_path_search or false

    if PREFAB_SKINS[self.name] ~= nil then
        for _,prefab_skin in pairs(PREFAB_SKINS[self.name]) do
            table.insert( self.deps, prefab_skin )
        end
    end
end)
```

**构造函数参数**：

| 参数 | 类型 | 含义 |
|------|------|------|
| `name` | string | Prefab 名称（如 `"log"`、`"spear"`） |
| `fn` | function | 工厂函数——调用它会创建一个新实体 |
| `assets` | table | 需要加载的资源列表（动画、贴图、音效等） |
| `deps` | table | 依赖的其他 Prefab 名称（确保它们也被加载） |
| `force_path_search` | bool | 是否强制搜索资源路径 |

注意 `self.name` 的处理：`string.sub(name, string.find(name, "[^/]*$"))` 会去掉路径前缀——这意味着你写 `Prefab("prefabs/log", fn)` 和 `Prefab("log", fn)` 的效果是一样的。这是为了兼容旧版本中可能带路径前缀的 Prefab 名称。

**皮肤依赖自动注入**：如果 `PREFAB_SKINS` 表中有这个 Prefab 的皮肤条目，它们会被自动添加到 `deps` 中。这确保了当玩家使用皮肤时，皮肤资源已经被加载。

同文件还定义了 `Asset` 类：

```lua
-- scripts/prefabs.lua 第 25-29 行
Asset = Class( function(self, type, file, param)
    self.type = type
    self.file = file
    self.param = param
end)
```

`Asset` 用于声明资源条目，常见的 `type` 值包括：
- `"ANIM"` —— 动画文件（`.zip`）
- `"IMAGE"` —— 图片
- `"ATLAS"` —— 图集（`.xml`）
- `"SOUND"` —— 音效
- `"SHADER"` —— 着色器
- `"PKGREF"` —— 包引用

---

### 5.1.4 进阶：CreateEntity —— 实体的诞生

当工厂函数 `fn()` 的第一行写 `local inst = CreateEntity()` 时，到底发生了什么？

```lua
-- scripts/mainfunctions.lua 第 486-496 行
function CreateEntity(name)
    local ent = TheSim:CreateEntity()        -- 引擎创建底层实体对象
    local guid = ent:GetGUID()               -- 获取全局唯一 ID
    local scr = EntityScript(ent)            -- 用 Lua 包装类包裹
    if name ~= nil then
        scr.name = name
    end
    Ents[guid] = scr                         -- 注册到全局实体表
    NumEnts = NumEnts + 1                    -- 计数
    return scr                               -- 返回 Lua 实体对象
end
```

**三步走**：

1. **`TheSim:CreateEntity()`**：调用 C++ 引擎创建一个底层实体对象——它此时只是一个空壳，没有位置、没有外观、没有任何游戏逻辑
2. **`EntityScript(ent)`**：用 Lua 的 `EntityScript` 类包裹这个底层实体——从此你可以通过 Lua 代码操作它（添加组件、监听事件、设置标签等）
3. **`Ents[guid] = scr`**：把这个实体注册到 `Ents` 全局表中——游戏运行时，`Ents` 包含了世界上的所有实体

**`Ents` 全局表**是一个以 GUID 为键的字典。任何时候你想通过 GUID 获取实体，都可以用 `Ents[guid]`。控制台命令 `c_inst(guid)` 就是这么实现的。

**为什么需要 `EntityScript` 包装？** 因为引擎的底层实体（C++ 对象）只提供最基本的功能（位置、渲染、物理）。`EntityScript` 是 Lua 层的扩展——组件系统、事件系统、标签系统、状态机等都是在 `EntityScript` 上实现的。你在 Mod 中操作的 `inst` 就是一个 `EntityScript` 实例。

---

### 5.1.5 进阶：SpawnPrefab —— 从图纸到实体

当你在控制台输入 `c_spawn("log")` 或在代码中写 `SpawnPrefab("log")` 时，完整的执行链如下：

#### 第一步：Lua 入口

```lua
-- scripts/mainfunctions.lua 第 403-411 行
function SpawnPrefab(name, skin, skin_id, creator, skin_custom)
    name = string.sub(name, string.find(name, "[^/]*$"))
    if skin and not IsItemId(skin) then
        print("Unknown skin", skin)
        skin = nil
    end
    local guid = TheSim:SpawnPrefab(name, skin, skin_id, creator, skin_custom)
    return Ents[guid]
end
```

`SpawnPrefab` 做了两件事：清理名称、调用引擎 `TheSim:SpawnPrefab`。引擎内部会回调 Lua 的 `SpawnPrefabFromSim`。

#### 第二步：引擎回调 —— `SpawnPrefabFromSim`

```lua
-- scripts/mainfunctions.lua 第 347-397 行
function SpawnPrefabFromSim(name)
    name = string.sub(name, string.find(name, "[^/]*$"))
    name = string.lower(name)

    local prefab = Prefabs[name]          -- 从全局表查找图纸
    if prefab == nil then
        print( "Can't find prefab " .. tostring(name) )
        return -1
    end

    if prefab then
        local inst = prefab.fn(TheSim)    -- 执行工厂函数！创建实体！

        if inst ~= nil then
            inst:SetPrefabName(inst.prefab or name)

            -- 执行所有 Mod 注册的 PrefabPostInit
            local modfns = modprefabinitfns[inst.prefab or name]
            if modfns ~= nil then
                for k,mod in pairs(modfns) do
                    mod(inst)
                end
            end

            -- 执行所有 Mod 注册的 PrefabPostInitAny
            for k,prefabpostinitany in pairs(ModManager:GetPostInitFns("PrefabPostInitAny")) do
                prefabpostinitany(inst)
            end

            -- 通知世界有实体被生成了
            if TheWorld then
                TheWorld:PushEvent("entity_spawned", inst)
            end

            return inst.entity:GetGUID()
        end
    end
end
```

**这是 Prefab 系统最核心的函数**。它的执行顺序是：

1. **查找图纸**：在 `Prefabs` 全局表中用名称查找对应的 `Prefab` 对象
2. **执行工厂函数**：调用 `prefab.fn(TheSim)` —— 这就是你在 Prefab 文件中写的 `local function fn()` 那个函数。它会创建实体、添加组件、设置属性
3. **执行 Mod PostInit**：依次执行所有 Mod 通过 `AddPrefabPostInit` 注册的回调——**这就是 Mod 修改原版 Prefab 的时机**
4. **执行 PrefabPostInitAny**：执行所有 Mod 通过 `AddPrefabPostInitAny` 注册的回调——它会对**每一个**生成的实体执行
5. **触发事件**：通知世界有新实体被创建

**为什么 PostInit 的顺序很重要？** 因为工厂函数先执行，PostInit 后执行。PostInit 拿到的 `inst` 已经是一个完全初始化好的实体——你可以修改它的组件参数、添加新组件、删除标签等。但如果你需要替换工厂函数本身的行为（比如替换动画），PostInit 可能不够用——你可能需要更底层的方法（如 `AddClassPostConstruct`）。

---

### 5.1.6 进阶：Prefab 的注册流程

Prefab 文件在什么时候被加载？注册到 `Prefabs` 表中的？

#### 游戏启动时

游戏启动时，`scripts/gamelogic.lua` 会调用 `LoadPrefabFile` 来加载所有 Prefab 文件：

```lua
-- scripts/mainfunctions.lua 第 145-180 行
function LoadPrefabFile( filename, async_batch_validation, search_asset_first_path )
    local fn, r = loadfile(filename)        -- 编译 Lua 文件
    assert(fn, "Could not load file ".. filename)
    -- ...

    local ret = {fn()}                       -- 执行文件，获取返回值

    if ret then
        for i,val in ipairs(ret) do
            if type(val)=="table" and val.is_a and val:is_a(Prefab) then
                -- 只收集返回值中的 Prefab 实例
                RegisterSinglePrefab(val)    -- 注册到系统
                PREFABDEFINITIONS[val.name] = val
            end
        end
    end

    return ret
end
```

**执行流程**：

1. `loadfile(filename)` 编译 Prefab 的 Lua 文件
2. `fn()` 执行文件——文件最后一行 `return Prefab("log", fn, assets)` 创建并返回一个 `Prefab` 对象
3. 收集返回值中的 `Prefab` 实例，调用 `RegisterSinglePrefab` 注册

`RegisterPrefabsImpl`（第 103-117 行）是实际的注册逻辑：

```lua
function RegisterPrefabsImpl(prefab, resolve_fn)
    for i,asset in ipairs(prefab.assets) do
        if not ShouldIgnoreResolve(asset.file, asset.type) then
            resolve_fn(prefab, asset)      -- 解析资源路径
        end
    end

    modprefabinitfns[prefab.name] = ModManager:GetPostInitFns("PrefabPostInit", prefab.name)
    Prefabs[prefab.name] = prefab          -- 存入全局表

    TheSim:RegisterPrefab(prefab.name, prefab.assets, prefab.deps)  -- 通知引擎
end
```

**注意第二步**：`modprefabinitfns[prefab.name]` 在注册时就预先获取了所有 Mod 的 PostInit 函数列表——这意味着**Mod 的 PostInit 必须在 Prefab 注册之前注册**，否则不会生效。这就是为什么 `modmain.lua` 在 Prefab 注册流程之前执行。

#### Mod 的 Prefab 注册

Mod 的 Prefab 通过 `PrefabFiles` 列表声明（在 `modmain.lua` 中），然后在 `ModWrangler:RegisterPrefabs`（`scripts/mods.lua` 第 675-718 行）中被加载和注册。最终也是合并到同一个 `Prefabs` 全局表中。

---

### 5.1.7 老手进阶：Prefab 系统的全景图

把以上所有内容串起来，Prefab 的完整生命周期如下：

```
游戏启动
  │
  ├── 加载 Mod → 执行 modmain.lua → 注册 AddPrefabPostInit 等钩子
  │
  ├── LoadPrefabFile("prefabs/log.lua")
  │     ├── loadfile → 编译
  │     ├── fn() → 返回 Prefab("log", fn, assets)
  │     └── RegisterPrefabsImpl
  │           ├── 解析资源路径
  │           ├── 预取 Mod PostInit 列表
  │           ├── Prefabs["log"] = prefab
  │           └── TheSim:RegisterPrefab (通知引擎加载资源)
  │
  ├── 游戏进行中...
  │
  └── SpawnPrefab("log") 被调用
        ├── TheSim:SpawnPrefab → C++ 引擎
        ├── 回调 SpawnPrefabFromSim("log")
        │     ├── Prefabs["log"].fn() → 执行工厂函数
        │     │     ├── CreateEntity()
        │     │     ├── AddTransform/AnimState/Network
        │     │     ├── SetPristine() → 客户端到此为止
        │     │     └── AddComponent 等（服务端专属）
        │     │
        │     ├── 执行 Mod PrefabPostInit → Mod 修改实体
        │     ├── 执行 Mod PrefabPostInitAny
        │     └── PushEvent("entity_spawned")
        │
        └── return Ents[guid] → 返回实体给调用者
```

**关键全局表**：

| 全局表 | 键 | 值 | 用途 |
|--------|----|----|------|
| `Prefabs` | Prefab 名称（如 `"log"`） | Prefab 对象（含工厂函数） | 存储所有已注册的图纸 |
| `Ents` | GUID（数字） | EntityScript 实例 | 存储所有存活的实体 |
| `PREFABDEFINITIONS` | Prefab 名称 | Prefab 对象 | 存储通过 `LoadPrefabFile` 加载的 Prefab |

**检查 Prefab 是否存在**：

```lua
function PrefabExists(name)
    return Prefabs[name] ~= nil
end
```

---

### 5.1.8 小结

- **Prefab 是"图纸"，实体是"成品"**——`Prefab` 类定义了如何创建实体，`SpawnPrefab` 按图纸生成实体
- 一个 Prefab 文件由三部分组成：**资源声明**（`assets`）+ **工厂函数**（`fn`）+ **Prefab 注册**（`return Prefab(...)`) 
- **`CreateEntity()`** 在引擎中创建空壳实体，再由 `EntityScript` 包裹为 Lua 可操作的对象
- **`SpawnPrefab`** 的完整链路：Lua 入口 → C++ 引擎 → `SpawnPrefabFromSim` 回调 → 执行工厂函数 → 执行 Mod PostInit → 返回实体
- **`Prefabs` 全局表**存储所有已注册的图纸；**`Ents` 全局表**存储所有存活的实体
- **Mod 的 `AddPrefabPostInit`** 在 Prefab 注册时就预先绑定，在每次 `SpawnPrefab` 时自动执行

> **下一节预告**：5.2 节我们将详细分析 Prefab 文件的内部结构——工厂函数中各个部分的作用、`SetPristine` 的分界线意义、以及实体的完整生命周期（从创建到销毁）。

## 5.2 Prefab 文件的结构与生命周期

### 本节导读

在 5.1 节，我们已经知道 Prefab 是"图纸"、实体是"成品"。这一节我们把视角从"Prefab 是什么"切换到"一个 Prefab 文件到底长什么样、生成出来的实体会经历哪些阶段"。

> **新手**可以从「5.2.1 三大件」「5.2.2 工厂函数的五段式」开始——只要能看懂 `log.lua` 和 `spear.lua` 的结构，就能自己动手写一个最简单的物品；**进阶读者**可以深入 `create_common` 模式与多形态 Prefab、以及 `OnSave / OnLoad / OnEntitySleep` 等生命周期钩子；**老手**可以直接跳到「实体完整生命周期流程图」，了解从 `CreateEntity` 到 `Retire` 的完整时序，以及 `persists`、`OnEntityReplicated` 等容易被忽视的扩展点。

---

### 5.2.1 快速入门：Prefab 文件的"三大件"

打开 `scripts/prefabs/` 目录下任何一个 Prefab 文件（比如 `log.lua`、`spear.lua`、`spider.lua`），你都会发现它的"骨架"由**三大件**组成：

```lua
-- 第一件：资源列表
local assets =
{
    Asset("ANIM", "anim/log.zip"),
}

-- 第二件：工厂函数
local function fn()
    local inst = CreateEntity()
    -- ... 一堆初始化代码 ...
    return inst
end

-- 第三件：Prefab 注册（文件最后一行）
return Prefab("log", fn, assets)
```

**三大件各自的职责**：

| 组成 | 职责 | 何时被引擎用到 |
|------|------|----------------|
| `assets` | 告诉引擎"我需要加载这些美术/音效文件" | Prefab **注册**时（游戏启动阶段）交给 `TheSim:RegisterPrefab` |
| `fn`（工厂函数） | 告诉引擎"怎么从零创建一个实体" | 每次 `SpawnPrefab` 时被调用 |
| `return Prefab(...)` | 把"名称 + 工厂函数 + 资源"打包注册 | `LoadPrefabFile` 读到返回值后调用 `RegisterPrefabsImpl` |

为什么这三件都必不可少？你可以这样理解：

- 只有 `assets`，引擎知道该加载什么素材，但不知道怎么把素材组装成实体
- 只有 `fn`，引擎能创建实体，但运行到 `SetBuild("log")` 时会因为没加载 `anim/log.zip` 而报错
- 不 `return Prefab(...)`，前两件都写了也白写——`LoadPrefabFile` 根本拿不到这个 Prefab 对象（回顾 5.1.6 节，注册靠的就是 `fn()` 执行后的返回值）

> **新手记忆**：每写一个 Prefab，脑子里先默念一遍"资源、工厂、注册"——缺一个，游戏要么找不到素材，要么找不到图纸。

---

### 5.2.2 快速入门：工厂函数的"五段式"

工厂函数是 Prefab 文件里最长、最复杂、也最值得掌握的一段代码。好消息是：**官方几乎所有物品类 Prefab 的工厂函数都按同一个套路写**。我们把 `scripts/prefabs/spear.lua` 第 28-77 行拆成五段：

```lua
local function fn()
    -- ① 创建实体 + 加载引擎组件
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    -- ② 物理和视觉表现
    MakeInventoryPhysics(inst)
    inst.AnimState:SetBank("spear")
    inst.AnimState:SetBuild("swap_spear")
    inst.AnimState:PlayAnimation("idle")

    -- ③ 客户端也需要的标签和状态
    inst:AddTag("sharp")
    inst:AddTag("pointy")
    inst:AddTag("weapon")
    MakeInventoryFloatable(inst, "med", 0.05, {1.1, 0.5, 1.1}, true, -9)

    -- ④ 分界线：SetPristine
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    -- ⑤ 仅服务端（主机）需要的组件和逻辑
    inst:AddComponent("weapon")
    inst.components.weapon:SetDamage(TUNING.SPEAR_DAMAGE)
    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(TUNING.SPEAR_USES)
    inst.components.finiteuses:SetUses(TUNING.SPEAR_USES)
    inst.components.finiteuses:SetOnFinished(inst.Remove)
    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")
    inst:AddComponent("equippable")
    inst.components.equippable:SetOnEquip(onequip)
    inst.components.equippable:SetOnUnequip(onunequip)
    MakeHauntableLaunch(inst)

    return inst
end
```

**逐段拆解**：

#### 第①段：三件套引擎组件

```lua
inst.entity:AddTransform()   -- 位置/朝向/缩放
inst.entity:AddAnimState()   -- 动画状态（SetBank/SetBuild 等都靠它）
inst.entity:AddNetwork()     -- 网络同步（联机版几乎所有实体都要）
```

这三行几乎是所有联机 Prefab 的**开场标配**。`CreateEntity` 返回的只是一个空壳（回顾 5.1.4 节），必须把"位置"「渲染」「网络」三件套显式加上才能用。

不同类型的实体还会在此基础上再加其它引擎组件，例如：

- 生物类 → `AddSoundEmitter()`（音效）、`AddDynamicShadow()`（动态阴影）
- 光源类 → `AddLight()`
- 地图标记 → `AddMiniMapEntity()`

在 `scripts/prefabs/spider.lua` 第 575-579 行就能看到蜘蛛多加了 `SoundEmitter`、`DynamicShadow`。

#### 第②段：物理和视觉

```lua
MakeInventoryPhysics(inst)              -- 物理：可被拾起的小物品
inst.AnimState:SetBank("spear")         -- 动画"骨架"——描述动画逻辑
inst.AnimState:SetBuild("swap_spear")   -- 动画"皮肤"——描述动画外观
inst.AnimState:PlayAnimation("idle")    -- 播放哪一段动画
```

`MakeInventoryPhysics` 是 `scripts/standardcomponents.lua` 里的一个辅助函数（第 370-386 行）：

```13:25:scripts/standardcomponents.lua
function MakeInventoryPhysics(inst, mass, rad)
    mass = mass or 1
    rad = rad or .5
	local phys = inst.entity:AddPhysics()
	phys:SetMass(mass)
	phys:SetFriction(.1)
	phys:SetDamping(0)
	phys:SetRestitution(.5)
	phys:SetCollisionGroup(COLLISION.ITEMS)
	phys:SetCollisionMask(
		COLLISION.WORLD,
		COLLISION.OBSTACLES,
		COLLISION.SMALLOBSTACLES
	)
	phys:SetSphere(rad)
    return phys
end
```

一行 `MakeInventoryPhysics(inst)` 就替你把 `AddPhysics`、`SetMass`、`SetFriction`、`SetCollisionGroup` 等八行配置全写好了。`scripts/standardcomponents.lua` 里类似的"一键配置"函数还有几十个：`MakeCharacterPhysics`（生物物理）、`MakeProjectilePhysics`（弹射物）、`MakeSmallBurnable`（可小型燃烧）、`MakeInventoryFloatable`（掉进水里会漂）、`MakeHauntableLaunch`（被鬼附身后被弹飞）等。

> **开发逻辑**：官方把这些"固定配方"抽成辅助函数，是因为游戏中大量物品的参数都一样——长矛和木头都是**可拾起的小物品**，它们的物理参数几乎一致，没必要每个 Prefab 都重复写 8 行。你自己写 Mod 时也应该尽量直接调用这些 `MakeXxx` 函数，保持和官方的一致。

#### 第③段：标签 + 客户端共享状态

```lua
inst:AddTag("sharp")
inst:AddTag("pointy")
inst:AddTag("weapon")  -- 注释里明确写"added to pristine state for optimization"
MakeInventoryFloatable(inst, "med", 0.05, {1.1, 0.5, 1.1}, true, -9)
```

这里放的都是**客户端和服务端都需要知道的信息**。标签系统（Tag）我们在 5.5 节会专门讲，现在只要理解：

- 客户端拿到这个实体时，也要能判断"它是武器吗？"——所以标签要在 `SetPristine` 之前加
- `MakeInventoryFloatable` 涉及漂浮动画，客户端要播放动画，所以也在这里加

注意 `spear.lua` 第 44-45 行有一行很耐人寻味的注释：

```lua
--weapon (from weapon component) added to pristine state for optimization
inst:AddTag("weapon")
```

组件 `weapon` 本来会自动在客户端上添加 `"weapon"` 标签，但官方选择**手动提前**加到 pristine 阶段，这样客户端不用等到组件同步就能识别它是武器。这是一个非常典型的"用标签避免组件同步"的性能优化手法。

#### 第④段：SetPristine 分界线

```lua
inst.entity:SetPristine()
if not TheWorld.ismastersim then
    return inst
end
```

**这两行是整个工厂函数里最重要的两行**。它们把工厂函数切成了"客户端/服务端共同执行的部分"和"仅服务端执行的部分"。

- `inst.entity:SetPristine()`：告诉引擎"**我已经把客户端也需要的全部初始化完成了**"——此后对实体的所有修改，客户端默认不会感知到。
- `if not TheWorld.ismastersim then return inst end`：如果**当前不是主机**（比如你连进别人的服务器），就直接返回，后面的代码全部不执行。

为什么要这样分？因为：

1. **客户端和主机执行的代码必须一致地完成客户端初始化**，否则客户端看到的实体外观会错乱
2. **服务端组件（如 `health`、`combat`、`inventoryitem`）客户端根本用不上**——它们的数据靠 replica 系统同步过来。如果客户端也去 `AddComponent("weapon")`，就是白白浪费内存

这个分界线的深层原理我们会在 **5.4 节** 详细讲。现在你只需要记住一条口诀：**客户端要用的写在上面，只有主机要用的写在下面**。

#### 第⑤段：服务端专属组件和逻辑

```lua
inst:AddComponent("weapon")
inst.components.weapon:SetDamage(TUNING.SPEAR_DAMAGE)
-- ...省略...
inst:AddComponent("equippable")
inst.components.equippable:SetOnEquip(onequip)
inst.components.equippable:SetOnUnequip(onunequip)
MakeHauntableLaunch(inst)

return inst
```

这一段是 Prefab 的"业务逻辑"所在：

- `AddComponent("weapon")` + `SetDamage`——决定它是把武器、伤害多少
- `AddComponent("finiteuses")`——它能用多少次、用完之后干什么
- `AddComponent("inventoryitem")`——它可以被捡起、放进背包
- `AddComponent("equippable")`——它可以被装备（挂到左/右手或头上）
- `MakeHauntableLaunch(inst)`——它被鬼附身后会被弹飞

最后 `return inst` 返回实体——回到 `SpawnPrefabFromSim`（5.1.5 节），Mod 的 `PrefabPostInit` 就在这个 return 之后被引擎调用。

> **新手记忆**：五段式"**创建 → 物理视觉 → 标签 → SetPristine → 服务端组件**"。把 `log.lua`、`spear.lua`、`flint.lua`（燧石）这三个文件对照着读一遍，就会发现它们 80% 长得一样——这是"惯用法"而不是巧合。

---

### 5.2.3 进阶：create_common 模式与多形态 Prefab

很多生物/物品有**多个变种**——比如蜘蛛有普通蜘蛛、战士蜘蛛、洞穴蜘蛛、吐蜘蛛、月亮蜘蛛、治疗蜘蛛、水上蜘蛛等 8 种。它们 90% 的逻辑一样（都有血量、战斗、AI、睡眠），只有 10% 不同（血量/伤害数值、贴图、行为树）。

如果每种都复制粘贴一份工厂函数，维护起来就是地狱。官方的做法是**用一个 `create_common` 函数处理共用部分，再用若干 `create_xxx` 处理差异**，最后在文件末尾 `return` 多个 `Prefab`。

以 `scripts/prefabs/spider.lua` 为例：

```572:590:scripts/prefabs/spider.lua
local function create_common(bank, build, tag, common_init, extra_data)
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddDynamicShadow()
    inst.entity:AddNetwork()

    MakeCharacterPhysics(inst, 10, .5)

    inst.DynamicShadow:SetSize(1.5, .5)
    inst.Transform:SetFourFaced()

    inst:AddTag("cavedweller")
    inst:AddTag("monster")
    -- ... 更多通用标签 ...
```

`create_common` 接收 `bank / build / tag / common_init / extra_data` 五个参数，把它们"插"到共用流程的关键位置。差异部分由各个 `create_xxx` 函数在调用 `create_common` 后补写：

```751:776:scripts/prefabs/spider.lua
local function create_spider()
    local inst = create_common("spider", "spider_build")

    if not TheWorld.ismastersim then
        return inst
    end

    inst.lunar_mutation_chance = TUNING.SPIDER_PRERIFT_MUTATION_SPAWN_CHANCE
    inst.components.health:SetMaxHealth(TUNING.SPIDER_HEALTH)
    inst.components.combat:SetDefaultDamage(TUNING.SPIDER_DAMAGE)
    -- ... 普通蜘蛛特有参数 ...
    return inst
end
```

文件末尾的 `return` 语句返回 8 个 Prefab 对象：

```1082:1089:scripts/prefabs/spider.lua
return Prefab("spider", create_spider, assets, prefabs),
       Prefab("spider_warrior", create_warrior, warrior_assets, prefabs),
       Prefab("spider_hider", create_hider, hiderassets, prefabs),
       Prefab("spider_spitter", create_spitter, spitterassets, prefabs),
       Prefab("spider_dropper", create_dropper, dropperassets, prefabs),
       Prefab("spider_moon", create_moon, moon_assets, prefabs),
       Prefab("spider_healer", create_healer, healer_assets, prefabs),
       Prefab("spider_water", create_water, water_assets, prefabs)
```

**回顾 5.1.6 节** `LoadPrefabFile` 的逻辑：

```lua
local ret = {fn()}  -- 执行文件，获取所有返回值
for i, val in ipairs(ret) do
    if type(val) == "table" and val.is_a and val:is_a(Prefab) then
        RegisterSinglePrefab(val)  -- 每个 Prefab 都注册
    end
end
```

`ret` 是一个 table，Lua 的多返回值都会被收集进去——所以**一个文件里 `return` 多少个 `Prefab`，就会注册多少个**。

**`create_common` 里的 `common_init` 参数**是一个很优雅的扩展点：

```611:613:scripts/prefabs/spider.lua
    if common_init ~= nil then
        common_init(inst)
    end
```

它允许某些变种在"共用流程"的中途插入自己的初始化逻辑。比如月亮蜘蛛（`spider_moon`）需要在 pristine 阶段加一个 `"lunar_aligned"` 标签（客户端也要识别），就把这段逻辑包成一个 `spider_moon_common_init` 函数传进 `create_common`（见 `spider.lua` 第 903-908 行）。

> **开发逻辑**：`create_common + 多 Prefab return` 是饥荒联机版里"一个文件定义一族 Prefab"的标准写法。玩家角色（`wilson.lua` 等 18 个角色共用 `player_common.lua` 里的 `MakePlayerCharacter`）、家具、船只的逻辑都是这么组织的。你写 Mod 时如果要做一个"多形态同源 Prefab"，直接参考 `spider.lua` 就行。

---

### 5.2.4 进阶：实体的生命周期回调字段

一个实体从诞生到销毁，会经过若干"关键事件"。引擎在这些关键时刻会**查找 `inst` 身上是否挂了对应的回调字段**，有就调用。

这些回调字段**不是 `EntityScript` 的方法**（你在 `scripts/entityscript.lua` 里搜不到它们的 `function EntityScript:OnSave` 定义），而是**动态挂载的字段**——你在工厂函数里写 `inst.OnSave = OnSave`，引擎在该事件触发时就会调用它。

| 回调字段 | 何时触发 | 触发位置（源码） | 常见用途 |
|----------|---------|------------------|----------|
| `inst.OnSave(inst, data)` | 存档时 | `scripts/entityscript.lua` 第 1924-1932 行 | 把 `inst` 上的自定义字段写入 `data` |
| `inst.OnLoad(inst, data, newents)` | 读档时 | `scripts/entityscript.lua` 第 1986-1988 行 | 从 `data` 恢复自定义字段 |
| `inst.OnPreLoad(inst, data, newents)` | 读档**前**（组件 load 之前） | `scripts/entityscript.lua` 第 1963-1965 行 | 需要在组件恢复前调整实体的特殊场景 |
| `inst.OnLoadPostPass(inst, newents, savedata)` | 读档后**二次 pass** | `scripts/entityscript.lua` 第 1949-1951 行 | 处理实体间引用（比如"我的主人"是另一个实体） |
| `inst.OnEntitySleep(inst)` | 实体进入休眠（玩家离远了） | `scripts/mainfunctions.lua` 第 675-677 行 | 做清理、归巢、停止耗性能的任务 |
| `inst.OnEntityWake(inst)` | 实体从休眠唤醒 | `scripts/mainfunctions.lua` 第 704-706 行 | 恢复行为 |
| `inst.OnRemoveEntity(inst)` | 实体被销毁 | `scripts/entityscript.lua` 第 1767-1769 行 | 解除引用、通知其他系统 |
| `inst.OnEntityReplicated(inst)` | 客户端首次接收到该实体 | C++ 引擎在客户端触发 | 客户端上做一次性初始化（仅客户端代码） |
| `inst.OnBuiltFn(inst, builder)` | 实体"被玩家建造出来"（区别于 Spawn） | `scripts/entityscript.lua` 第 1693-1703 行 | 建筑的建成动画、成就统计 |

#### 示例：OnSave / OnLoad（保存自定义数据）

看 `scripts/prefabs/tree_rocks.lua`（石化树）第 655-686 行：

```lua
local function OnSave(inst, data)
    if inst:HasTag("burnt") or (inst.components.burnable ~= nil and inst.components.burnable:IsBurning()) then
        data.burnt = true
    end
    if inst:HasTag("boulder") then
        data.boulder = true
    end
    if inst.vine_loot then
        data.vine_loot = inst.vine_loot
    end
end

local function OnLoad(inst, data)
    if not data then
        return
    end
    if data.burnt and not inst:HasTag("burnt") then
        OnBurnt(inst, true)
    elseif data.boulder then
        -- ...
    end
end
```

挂到实体上：

```lua
inst.OnSave = OnSave
inst.OnLoad = OnLoad
```

**执行流程**（结合 `entityscript.lua` 第 1903-1937 行）：

1. 存档时，`GetPersistData` 先遍历 `inst.components`，让**每个组件**自己 `OnSave`（组件各存各的数据）
2. 然后才调用 `inst.OnSave(inst, data)`——让**工厂函数**在 `data` 上补写自己的东西（标签状态、临时变量等）

**为什么不把所有状态都交给组件保存？** 因为有些状态本质上不属于任何组件——比如 `tree_rocks` 是不是处于"boulder"（巨石）形态，这是 Prefab 级的外观状态，不归任何组件管。`OnSave` 就是给这种"散装状态"保存的地方。

#### 示例：OnEntitySleep（实体离玩家远的时候）

看 `scripts/prefabs/spider.lua` 第 282-286 行：

```lua
local function OnEntitySleep(inst)
    if TheWorld.state.iscaveday then
        DoReturn(inst)
    end
end
```

挂到实体上（第 624 行）：

```lua
inst.OnEntitySleep = OnEntitySleep
```

**为什么需要这个？** 饥荒联机版做了"离玩家远的实体进入休眠"的性能优化——实体会被"关掉"（`OnEntitySleep`），它的 StateGraph、Brain、Emitter 都会被停掉（见 `scripts/mainfunctions.lua` 第 679-695 行）。如果蜘蛛进入休眠时是白天，它应该自动回巢（不然玩家转一圈回来，白天蜘蛛还在外面晒太阳就很奇怪）——这个"睡前收拾一下"的逻辑就放在 `OnEntitySleep` 里。

#### 示例：OnRemoveEntity（销毁清理）

`EntityScript:Remove` 在 `scripts/entityscript.lua` 第 1705-1772 行，它会在最后调用 `inst.OnRemoveEntity`：

```1767:1770:scripts/entityscript.lua
    if self.OnRemoveEntity then
        self:OnRemoveEntity()
    end
    self.persists = false
    self.entity:Retire()
```

注意调用顺序：

1. 先解除父子、平台关系
2. `OnRemoveEntity(self.GUID)` 把 `Ents[guid]` 移除
3. 推 `"onremove"` 事件（给其他监听者一次机会）
4. 取消所有 listener / watcher / pending task
5. 遍历每个组件调用 `v:OnRemoveEntity()`
6. 递归 `Remove` 所有 child
7. **最后**才调 `self.OnRemoveEntity()`——这时整个实体已经"基本散伙"，可以做最终清理
8. `persists = false`（见下一节）+ `entity:Retire()`（底层引擎释放）

---

### 5.2.5 进阶：persists 字段与实体的"持久性"

你会注意到很多 FX、弹射物 Prefab 里有这样一行：

```lua
inst.persists = false
```

它是干什么的？

在 `EntityScript` 的构造函数里（`scripts/entityscript.lua` 第 222 行）：

```lua
self.persists = true
```

默认所有实体都 `persists = true`，意思是"**我要被存档保存下来**"。存档代码（`scripts/mainfunctions.lua` 第 1088 行）在遍历实体时会判断：

```1088:1088:scripts/mainfunctions.lua
        if v.persists and v.prefab ~= nil and v.Transform ~= nil and v.entity:GetParent() == nil and v:IsValid() then
```

只有同时满足：
- `persists == true`
- 有 prefab 名字
- 有位置
- 没有父实体（不是某个实体的子物件）
- 是有效的

才会被写进存档。

**什么情况下应该设 `persists = false`？**

看 `scripts/prefabs/wx78_drone_delivery.lua` 第 66 行，一个短暂出现的无人机 FX：

```lua
inst.persists = false
```

以及 `scripts/prefabs/boat_leak.lua` 第 236 行的船只漏水特效：

```lua
inst.persists = false
```

这些场景的共同点是：

- **临时性**：它们存在几秒到几分钟，然后自动消失
- **可重新生成**：玩家下次进游戏时，如果游戏状态还需要这个特效，会重新 SpawnPrefab
- **存了也没意义**：存档它们会徒增存档体积

相反，一把玩家丢在地上的长矛、一只游荡的蜘蛛，这些就必须 `persists = true`（默认值），不然下次登录就凭空消失了。

> **新手实战经验**：自己写 Mod 时，如果这个 Prefab 是**特效/弹射物/短暂生成物**，一律 `inst.persists = false`；如果是**物品/怪物/建筑**，保持默认即可。

---

### 5.2.6 老手进阶：实体的完整生命周期流程图

把前面所有回调串起来，一个实体从"存在"到"不存在"会经历以下时序（实心箭头代表引擎驱动，虚线代表可能发生）：

```
┌─────────────────────────────────────────────────────────────┐
│  A. 诞生阶段（每个实体都会经历一次）                           │
├─────────────────────────────────────────────────────────────┤
│  SpawnPrefab("log") 或 从存档反序列化                         │
│        ↓                                                      │
│  TheSim:SpawnPrefab → SpawnPrefabFromSim                     │
│        ↓                                                      │
│  prefab.fn()                                                  │
│    ├── CreateEntity() → Ents[guid] = inst                    │
│    ├── 第①②③段（客户端也要执行）                              │
│    ├── SetPristine()                                          │
│    └── 第⑤段（仅 master）                                     │
│        ↓                                                      │
│  执行 Mod 的 PrefabPostInit / PrefabPostInitAny              │
│        ↓                                                      │
│  PushEvent("entity_spawned") 到 TheWorld                     │
│        ↓                                                      │
│  (如果是从存档恢复) SetPersistData(data, newents)             │
│    ├── OnPreLoad(inst, data, newents)                        │
│    ├── 每个组件 cmp:OnLoad(v, newents)                       │
│    └── inst:OnLoad(data, newents)                            │
│        ↓                                                      │
│  (稍后) LoadPostPass                                          │
│    ├── 每个组件 cmp:LoadPostPass(newents, v)                 │
│    └── inst:OnLoadPostPass(newents, savedata)                │
│        ↓                                                      │
│  (如果是玩家建造的) EntityScript:OnBuilt(builder)             │
│    ├── 每个组件 cmp:OnBuilt(builder)                         │
│    └── inst:OnBuiltFn(builder)                               │
│        ↓                                                      │
│  (客户端特有) inst:OnEntityReplicated() ← C++ 引擎触发        │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  B. 存活阶段（可能反复进入 sleep / wake）                      │
├─────────────────────────────────────────────────────────────┤
│                                                                │
│  正常运行：Brain tick、SG update、事件监听、组件 OnUpdate     │
│        │                                                      │
│        ├→ 玩家走远          → OnEntitySleep                  │
│        │     ├── PushEvent("entitysleep")                    │
│        │     ├── inst:OnEntitySleep()                        │
│        │     ├── 停止 Brain、Hibernate SG                    │
│        │     └── 每个组件 cmp:OnEntitySleep()                │
│        │                                                      │
│        ├→ 玩家回来          → OnEntityWake                   │
│        │     ├── PushEvent("entitywake")                     │
│        │     ├── inst:OnEntityWake()                         │
│        │     ├── 重启 Brain、Wake SG                         │
│        │     └── 每个组件 cmp:OnEntityWake()                 │
│        │                                                      │
│        └→ 定期存档          → GetPersistData                 │
│              ├── 每个组件 cmp:OnSave()                       │
│              └── inst:OnSave(data)                           │
│                                                                │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  C. 销毁阶段                                                  │
├─────────────────────────────────────────────────────────────┤
│  inst:Remove() 被调用（主动销毁 / 切片卸载 / 玩家退出）        │
│        ↓                                                      │
│  if parent → parent:RemoveChild(self)                        │
│  if platform → platform:RemovePlatformFollower(self)         │
│        ↓                                                      │
│  OnRemoveEntity(self.GUID)                                    │
│    ├── Ents[guid] = nil                                       │
│    ├── BrainManager/SGManager:OnRemoveEntity                  │
│    └── NumEnts -= 1                                           │
│        ↓                                                      │
│  PushEvent("onremove")                                        │
│        ↓                                                      │
│  StopAllWatchingWorldStates / RemoveAllEventCallbacks         │
│        ↓                                                      │
│  每个组件 cmp:OnRemoveEntity()                                │
│        ↓                                                      │
│  每个 replica cmp:OnRemoveEntity()                            │
│        ↓                                                      │
│  递归 Remove 所有 child                                       │
│        ↓                                                      │
│  inst:OnRemoveEntity() ← 最后一次机会                         │
│        ↓                                                      │
│  persists = false; entity:Retire() ← 真正销毁                 │
└─────────────────────────────────────────────────────────────┘
```

**几个容易踩坑的细节**：

1. **`OnSave` 是反复触发的**——不仅在玩家主动保存时，世界自动存档也会触发。所以 `OnSave` 必须是**纯只读**的，不要在里面改 `inst` 的状态。

2. **`OnLoad` 的 `data` 可能是 `nil`**——如果一个实体是"新生成"而不是"从存档恢复"，`SetPersistData(nil, newents)` 也会被调用，但 `data` 是 `nil`。`tree_rocks.lua` 的 OnLoad 第一行就是 `if not data then return end`，这是必须的。

3. **`OnRemoveEntity` 在组件清理**之后**才调**——此时 `inst.components.health` 等都还存在（组件的 `OnRemoveEntity` 只是让组件自身做清理，并不从 `components` 表删除），但如果你访问 `inst.components.X.Y`，要注意 X 的内部字段可能已经置空。

4. **`OnEntitySleep` 期间 StateGraph 和 Brain 停了**——在 `OnEntitySleep` 里 `inst.sg:GoToState(...)` 是**没效果的**。所以 `spider.lua` 的 `OnEntitySleep` 只是调 `DoReturn(inst)`（逻辑上"回家"），不会播动画。

5. **`OnEntityReplicated` 只在客户端触发**——它是 C++ 引擎在接收到网络同步的实体时调用的 Lua 钩子。所以在 `classified` 类 Prefab（客户端专属状态同步实体）里看到它很常见。它发生在 `SetPristine` 之后，但此时 `inst.replica` 已经准备好了，是做"客户端初始化"的最佳位置。

---

### 5.2.7 老手进阶：工厂函数中容易被忽视的扩展点

除了回调字段，工厂函数里还有一些**常见但容易被新手忽略**的小约定，它们往往是 Mod 要 hook 原版时的关键抓手。

#### (1) `inst.scrapbook_deps`——图鉴依赖

在 `scripts/prefabs/spider.lua` 第 596 行：

```lua
inst.scrapbook_deps = {"silk","spidergland","monstermeat"}
```

这是**图鉴系统**（Scrapbook）的一个提示——当玩家查看蜘蛛的图鉴词条时，UI 会把这几个掉落物一起展示。Mod 里如果要让你的新生物出现在图鉴掉落列表里，需要设置这个字段。

#### (2) `inst.pickupsound`——拾起时的音效类型

在 `scripts/prefabs/log.lua` 第 21 行：

```lua
inst.pickupsound = "wood"
```

`inventoryitem` 组件在玩家捡起这个物品时，会根据这个字段在"木头/金属/布料/纸..."等一系列预设音效里选一个播放。比木头、燧石、布、纸各自"叮"一种声音。

#### (3) `inst.build`——约定俗成的外观记录

在 `scripts/prefabs/spider.lua` 第 745 行：

```lua
inst.build = build
```

`AnimState:SetBuild(build)` 已经告诉了引擎，但很多代码（比如 `SetHappyFace` 的 `OverrideSymbol(..., inst.build, ...)`）还需要在运行时知道"我当前是什么 build"。官方约定把 build 名也存一份到 `inst.build`。Mod 里做皮肤替换时也要遵守这个约定。

#### (4) `inst.incineratesound`、`inst.SoundPath`——公开的音效接口

```lua
inst.SoundPath = SoundPath
inst.incineratesound = SoundPath(inst, "die")
```

这是把音效路径"对外暴露"到 `inst` 上——其他组件（比如 `burnable` 组件在实体被烧毁时）会找 `inst.incineratesound` 来播放。不遵守这个约定，你的怪物被烧死就不会哼一声。

#### (5) `inst._braindisabled`、`inst.sleepstatepending`——引擎内部字段

这些**下划线开头**的字段是引擎/EntityScript 内部使用的，**Mod 不要动它们**。你会在源码里看到 `EntityScript:SetBrain`（`entityscript.lua` 第 1103-1116 行）设置了 `self._braindisabled`，这是为了和 `OnEntitySleep` 配合——睡眠时 Brain 停止但 `brainfn` 保留，唤醒时再重建。

#### (6) Mod 层的三个 PostInit 钩子

这些不是写在工厂函数里，而是**在 `modmain.lua` 里注册**，但它们会在实体生命周期的固定时刻被调用：

| 钩子 | 注册方式 | 触发位置 |
|------|---------|---------|
| `PrefabPostInit` | `AddPrefabPostInit("log", function(inst) ... end)` | 每次 `SpawnPrefabFromSim` 末尾（5.1.5 节流程第 3 步） |
| `PrefabPostInitAny` | `AddPrefabPostInitAny(function(inst) ... end)` | 每次 `SpawnPrefabFromSim` 末尾（流程第 4 步） |
| `ClassPostConstruct` | `AddClassPostConstruct("prefabs/log", fn)` | Prefab **文件加载时**，`modmain.lua` 阶段 |

三者的本质区别：

- `ClassPostConstruct` 改的是**文件级别的东西**——比如在 `prefabs/log.lua` 里追加一个 `local function NewHelperFn` 或修改 `assets` 列表
- `PrefabPostInit` 改的是**每个实例**——在 `log` 实体被 Spawn 时给它加组件、改参数
- `PrefabPostInitAny` 也是每个实例，但作用于**所有 Prefab**（比如"全世界所有实体都加个调试标签"）

大多数时候你想做的是 `AddPrefabPostInit`——它最安全、最常用。

---

### 5.2.8 小结

- **每个 Prefab 文件都由"三大件"组成**：`assets`（资源列表）+ `fn`（工厂函数）+ `return Prefab(name, fn, assets)`（注册）。缺一不可。
- **工厂函数遵循"五段式"**：①创建+引擎组件 → ②物理视觉 → ③标签+客户端共享 → ④`SetPristine` 分界 → ⑤服务端组件。读 `log.lua` 和 `spear.lua` 就是最好的入门。
- **`MakeXxx` 系列辅助函数**（`scripts/standardcomponents.lua`）把常用配置打包成一行调用。自己写 Mod 时也应该优先用它们，保持和官方一致。
- **多形态 Prefab 用 `create_common` 模式**：共用逻辑抽成 `create_common(…)`，差异部分由 `create_xxx` 补写，文件末尾 `return` 多个 `Prefab` 对象（如 `spider.lua` 的 8 种蜘蛛）。
- **实体生命周期靠一组回调字段驱动**：`OnSave / OnLoad / OnPreLoad / OnLoadPostPass / OnEntitySleep / OnEntityWake / OnRemoveEntity / OnEntityReplicated / OnBuiltFn`。它们不是 `EntityScript` 的方法，而是**挂在 `inst` 上的字段**——引擎遇到对应事件时会查找并调用。
- **`inst.persists`** 控制实体是否进存档。默认 `true`；临时性的 FX、弹射物应显式设 `false`。
- **完整生命周期 = A 诞生 → B 存活（可反复 sleep/wake）→ C 销毁**，每个阶段都有明确的钩子时序。
- **Mod 的三种 PostInit（`PrefabPostInit` / `PrefabPostInitAny` / `ClassPostConstruct`）**对应不同粒度的修改——最常用的是 `PrefabPostInit`。

> **下一节预告**：5.3 节会把视角从"单个 Prefab 的结构"拉到"引擎是怎么协助我们创建实体的"——我们会系统整理 `scripts/standardcomponents.lua` 里的 `MakeInventoryPhysics / MakeCharacterPhysics / MakePlayerCharacter / MakeSmallBurnable / MakeHauntableXxx / MakeInventoryFloatable` 等几十个辅助函数，看它们各自解决什么问题、什么时候该用哪个。

## 5.3 实体的创建流程：MakePlayerCharacter / MakeInventoryPhysics 等辅助函数

（待编写）

## 5.4 Pristine State——为什么 Prefab 有两段代码

（待编写）

## 5.5 Tags 系统——实体的"标签身份"

（待编写）

## 5.6 Asset 声明与资源加载机制（RegisterPrefabs 流程）

（待编写）

## 5.7 实战：创建一个全新的自定义物品

（待编写）
