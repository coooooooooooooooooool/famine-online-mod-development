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

### 本节导读

5.2 节我们拆开了一个 Prefab 文件的结构，反复提到"`MakeInventoryPhysics`、`MakeSmallBurnable`、`MakeInventoryFloatable` 这些 `MakeXxx` 函数把一整段配置打包成了一行调用"。这一节就聚焦它们——**饥荒联机版官方有一套系统性的"辅助函数"**，绝大部分集中在 `scripts/standardcomponents.lua`（约 1900 行）和 `scripts/prefabs/player_common.lua` 里的 `MakePlayerCharacter`。

读懂这些辅助函数，等于**读懂了"做一个新物品/新怪物/新角色"的八成套路**。

> **新手**只需要记住"见到 `MakeXxx` 就知道是在'一键套用官方配方'"，掌握最常用的三五个就够写 Mod；**进阶读者**应该顺着本节的分类表去 `standardcomponents.lua` 里对照读一遍，看每个辅助函数内部到底做了什么，这样 Mod 中需要"略微改一改"时你能手动展开；**老手**可以直接跳到 `MakePlayerCharacter` 部分，理解官方为什么要用"高阶工厂 + common_postinit / master_postinit 回调"这种设计模式，并学会把它应用到自己的大型 Mod 里。

---

### 5.3.1 快速入门：为什么需要辅助函数？

回忆 5.2 节的例子——`scripts/prefabs/log.lua`（木头）第 13 行只写了一行：

```lua
MakeInventoryPhysics(inst)
```

如果不借助辅助函数，你就得手写一堆等价的底层调用：

```lua
-- 手动展开 MakeInventoryPhysics 的等价代码
local phys = inst.entity:AddPhysics()
phys:SetMass(1)
phys:SetFriction(.1)
phys:SetDamping(0)
phys:SetRestitution(.5)
phys:SetCollisionGroup(COLLISION.ITEMS)
phys:SetCollisionMask(
    COLLISION.WORLD,
    COLLISION.OBSTACLES,
    COLLISION.SMALLOBSTACLES
)
phys:SetSphere(.5)
```

这就是 `MakeInventoryPhysics` 内部的全部内容（见 `scripts/standardcomponents.lua` 第 370-386 行）。**它做的事就三件**：

1. 给实体挂上物理组件 `AddPhysics()`
2. 设置质量、摩擦力、阻尼、弹性等物理参数
3. 配好碰撞组与碰撞掩码（决定它能撞到什么、被什么撞）

一个物品写一遍可以接受；**游戏里几百件物品都要写一遍，就是灾难**——不仅啰嗦，还容易不同物品参数不一致。把"可拾起物品应有的物理配置"抽成一个函数叫 `MakeInventoryPhysics`，大家都调它，再也没人写错参数。

> **新手理解**：`MakeXxx` 是**一组"行业术语"**——"我的物品是可以被捡起来的小物品" → `MakeInventoryPhysics`。官方已经把"可被捡起来的小物品"应该有的物理参数调好了，你直接用这个名字喊出需求就行。

---

### 5.3.2 快速入门：三个最常用的物理辅助函数

写 Mod 遇到的第一个问题永远是"这东西的物理应该怎么配？"——下面这三个函数覆盖了 90% 的场景：

| 函数 | 适用对象 | 典型例子 |
|------|---------|---------|
| `MakeInventoryPhysics(inst, mass, rad)` | **可拾起的小物品** | 木头、燧石、长矛、食物 |
| `MakeCharacterPhysics(inst, mass, rad)` | **活的生物** | 蜘蛛、猪人、玩家角色 |
| `MakeObstaclePhysics(inst, rad, height)` | **静态建筑/障碍物** | 科学机器、箱子、墙 |

它们的核心差异是**碰撞组**：

- `MakeInventoryPhysics` 属于 `COLLISION.ITEMS`——能和世界、障碍物碰撞，不会卡住生物
- `MakeCharacterPhysics` 属于 `COLLISION.CHARACTERS`——生物之间会互相推挤，能走路
- `MakeObstaclePhysics` 属于 `COLLISION.OBSTACLES`，**质量为 0**——不会被推动，是"墙"

看 `scripts/standardcomponents.lua` 第 402-417 行 `MakeCharacterPhysics`：

```lua
function MakeCharacterPhysics(inst, mass, rad)
    local phys = inst.entity:AddPhysics()
    phys:SetMass(mass)
    phys:SetFriction(0)
    phys:SetDamping(5)
    phys:SetCollisionGroup(COLLISION.CHARACTERS)
    phys:SetCollisionMask(
        COLLISION.WORLD,
        COLLISION.OBSTACLES,
        COLLISION.SMALLOBSTACLES,
        COLLISION.CHARACTERS,
        COLLISION.GIANTS
    )
    phys:SetCapsule(rad, 1)
    return phys
end
```

注意三个细节：
1. **没有 `mass = mass or 1` 这样的默认值**——生物必须显式传质量（蜘蛛是 10、玩家是 75）。 
2. **用 `SetCapsule` 而不是 `SetSphere`**——生物是竖着的胶囊形状，物品是球形。
3. **碰撞掩码包含 `COLLISION.CHARACTERS` 本身**——所以生物之间会互相挡。

**实战经验**：

- 做一个"可以扔在地上的新物品"？→ `MakeInventoryPhysics(inst)`
- 做一只新的小怪物？→ `MakeCharacterPhysics(inst, 10, .5)`（参考蜘蛛）
- 做一个"玩家可以靠到但不能穿过"的新建筑？→ `MakeObstaclePhysics(inst, 1)`（半径 1）
- 做一个"可以被玩家'扛起来'搬动"的大型建筑？→ `MakeHeavyObstaclePhysics(inst, 1)`，再配合 `heavyobstaclephysics` 组件

---

### 5.3.3 进阶：辅助函数全家福（按功能分类）

`scripts/standardcomponents.lua` 里的 `MakeXxx` 函数有七十多个。直接按"我要做什么"分类整理如下——**这张表就是你写 Prefab 时的查找索引**。

#### ① 物理类（对应 `AddPhysics`）

| 函数 | 适用 | 典型 prefab |
|------|------|-------------|
| `MakeInventoryPhysics(inst, mass, rad)` | 可拾起的小物品 | `log.lua`（第 13 行） |
| `MakeProjectilePhysics(inst, mass, rad)` | 弹射物（只碰地面，不挡生物） | `spider_web_spit` 等 |
| `MakeCharacterPhysics(inst, mass, rad)` | 普通生物 | `spider.lua`（第 581 行） |
| `MakeFlyingCharacterPhysics(inst, mass, rad)` | 飞行生物 | `crow.lua`、`bee.lua` |
| `MakeGiantCharacterPhysics(inst, mass, rad)` | 巨型生物 | `deerclops.lua`、`beequeen.lua` |
| `MakeGhostPhysics(inst, mass, rad)` | 幽灵 | `ghost.lua` |
| `MakeObstaclePhysics(inst, rad, height)` | 静态大型障碍 | 科学机器、墙 |
| `MakeWaterObstaclePhysics(inst, rad, height, restitution)` | 水上障碍（挡船） | 灯塔、浮标 |
| `MakeSmallObstaclePhysics(inst, rad, height)` | 小型障碍（不挡 giant） | 花盆、矮栅栏 |
| `MakeHeavyObstaclePhysics(inst, rad, height)` | 可"扛起"的重型建筑 | 脚架、雕像 |
| `MakePondPhysics(inst, rad, height)` | 池塘类型（玩家不能进） | `pond.lua` |

源码入口在 `scripts/standardcomponents.lua` 第 370-769 行。**这些函数的区别就在 `SetCollisionGroup` 和 `SetCollisionMask` 的不同组合**——不同碰撞组决定"谁能撞到谁"。

> **开发逻辑**：饥荒的碰撞体系是分层的（`COLLISION.WORLD / OBSTACLES / SMALLOBSTACLES / CHARACTERS / GIANTS / ITEMS / FLYERS / LIMITS`）。一个 giant（比如巨鹿）踩小障碍物（花朵）是踩扁的，但撞到大障碍物（科学机器）会被挡——这种"体型感"就是靠这些辅助函数的碰撞组合表达的。

#### ② 燃烧与传播类（对应 `burnable` + `propagator` 组件）

| 函数 | 效果 | 典型 prefab |
|------|------|-------------|
| `MakeSmallBurnable(inst, time, offset, structure, sym)` | 小型可燃烧物（10 秒） | 木头、草 |
| `MakeMediumBurnable(inst, time, offset, structure, sym)` | 中型可燃烧物（20 秒） | 箱子、帐篷 |
| `MakeLargeBurnable(inst, time, offset, structure, sym)` | 大型可燃烧物（30 秒） | 树、篝火 |
| `MakeSmallBurnableCharacter(inst, sym, offset)` | 小型生物（可被烧但不传播热源） | 兔子、仓鼠 |
| `MakeMediumBurnableCharacter(inst, sym, offset)` | 中型生物 | 蜘蛛、猪人 |
| `MakeLargeBurnableCharacter(inst, sym, offset, scale)` | 大型生物 | 牛、独眼巨鹿 |
| `MakeSmallPropagator(inst)` | 小型火势传播源 | 木头、草 |
| `MakeMediumPropagator(inst)` | 中型火势传播源 | 树、箱子 |
| `MakeLargePropagator(inst)` | 大型火势传播源 | 篝火、锅 |

**`Burnable` vs `Propagator` 的区别**：`Burnable` 是"我自己会烧"，`Propagator` 是"我旁边有火就会被引燃 / 我烧起来会点燃旁边"。**二者通常配对使用**。

看 `scripts/prefabs/log.lua` 第 41-42 行：

```lua
MakeSmallBurnable(inst, TUNING.MED_BURNTIME)
MakeSmallPropagator(inst)
```

这两行合起来的效果是：木头会被旁边的火点燃（propagator），被点燃后会自己烧 `MED_BURNTIME` 秒（burnable），烧的时候旁边的易燃物也会被点燃（propagator 反向传播）。

`xxxBurnableCharacter` 系列专门针对**生物**：它只加 burnable（生物不加 propagator，因为生物跑来跑去不会"引燃"空气），且不会产生炭化尸体。看 `scripts/standardcomponents.lua` 第 244-255 行：

```lua
function MakeSmallBurnableCharacter(inst, sym, offset)
    local burnable = inst:AddComponent("burnable")
    burnable:SetFXLevel(1)
    burnable:SetBurnTime(6)
    burnable.canlight = false  -- 生物身上不能被手动点火（不能拿火把戳）
    burnable:AddBurnFX(burnfx.character, offset or ..., sym)

    local propagator = MakeSmallPropagator(inst)
    propagator.acceptsheat = false  -- 生物不会被旁边的火自动引燃

    return burnable, propagator
end
```

注意 `propagator.acceptsheat = false`——生物不会"因为旁边着火就自燃"，必须被直接攻击（比如被火枪射中）。

#### ③ 冰冻类（对应 `freezable` 组件）

| 函数 | 抗冻等级 | 典型 prefab |
|------|----------|-------------|
| `MakeTinyFreezableCharacter` | 1（最容易冻） | 小蜘蛛、小雀 |
| `MakeSmallFreezableCharacter` | 2 | 中等怪物 |
| `MakeMediumFreezableCharacter` | 3（`SetResistance(2)`） | 蜘蛛 |
| `MakeLargeFreezableCharacter` | 4（`SetResistance(3)`） | 猪人、高血怪 |
| `MakeHugeFreezableCharacter` | 5（`SetResistance(4)`） | boss 级 |

参数 `SetResistance(n)` 的意义：**需要被冰冻攻击击中 n+1 次才会被冻住**。Boss 级怪物往往要打 5 下冰冻手杖才能冰住。

#### ④ 鬼附身类（对应 `hauntable` 组件）

这是饥荒联机版独有的机制——玩家死亡后变成鬼，可以"附身"物品触发特殊效果（让长矛被弹飞、让书烧掉、让种子变成作物等）。`scripts/standardcomponents.lua` 第 987-1454 行定义了近 20 种 `MakeHauntableXxx`。

| 函数 | 效果 | 典型 |
|------|------|------|
| `MakeHauntable(inst)` | 基础：仅增加 cooldown（什么都不做） | |
| `MakeHauntableLaunch(inst)` | 被附身时飞出去 | `spear.lua` 第 74 行 |
| `MakeHauntableLaunchAndSmash(inst)` | 飞出去 + 落地碎裂 | 石头 |
| `MakeHauntableLaunchAndIgnite(inst)` | 飞出去 + 点燃 | `log.lua` 第 44 行 |
| `MakeHauntableWork(inst)` | 触发一次砍伐/挖掘（自己对自己 work） | 树、矿石 |
| `MakeHauntableIgnite(inst)` | 被附身后原地燃烧 | 易燃物 |
| `MakeHauntableFreeze(inst)` | 被附身后冻住 | 水 |
| `MakeHauntableChangePrefab(inst, newprefab)` | 变成另一种物品 | 种子 → 作物 |
| `MakeHauntablePerish(inst)` | 加速腐烂 | 食物 |
| `MakeHauntablePanic(inst)` | 让生物恐慌逃跑 | 中性生物 |
| `MakeHauntablePlayAnim(inst, ...)` | 播放动画 | 书、摆件 |
| `MakeHauntableGoToState(inst, state)` | 让实体切到指定 StateGraph 状态 | 需要 SG 的生物 |
| `MakeHauntableDropFirstItem(inst)` | 掉落第一件背包物品 | 容器 |

看 `scripts/standardcomponents.lua` 第 993-1009 行：

```lua
function MakeHauntableLaunch(inst, chance, speed, cooldown, haunt_value)
    if not inst.components.hauntable then inst:AddComponent("hauntable") end
    inst.components.hauntable.cooldown = cooldown or TUNING.HAUNT_COOLDOWN_SMALL
    inst.components.hauntable:SetOnHauntFn(function(inst, haunter)
        chance = chance or TUNING.HAUNT_CHANCE_ALWAYS
        if math.random() <= chance then
            Launch(inst, haunter, speed or TUNING.LAUNCH_SPEED_SMALL)
            inst.components.hauntable.hauntvalue = haunt_value or TUNING.HAUNT_TINY
            -- ...
            return true
        end
        return false
    end)
end
```

逻辑很简单：**加组件 → 配置 cooldown → 给组件注入一个 `OnHauntFn` 回调**，回调里按概率触发"弹飞"效果。其他 `MakeHauntableXxx` 都是换一个回调函数而已。

#### ⑤ 漂浮类（对应 `floater` 组件）

```lua
MakeInventoryFloatable(inst, size, offset, scale, swap_bank, float_index, swap_data)
```

**实现**（`scripts/standardcomponents.lua` 第 1700-1719 行）：

```lua
function MakeInventoryFloatable(inst, size, offset, scale, swap_bank, float_index, swap_data)
    local floater = inst:AddComponent("floater")
    floater:SetSize(size or "small")
    if offset then floater:SetVerticalOffset(offset) end
    if scale then floater:SetScale(scale) end
    if swap_bank then
        floater:SetBankSwapOnFloat(swap_bank, float_index, swap_data)
    elseif swap_data then
        floater:SetSwapData(swap_data)
    end
    return floater
end
```

这个是**可拾起物品掉进水里能漂浮**的机制——饥荒联机版新增海洋玩法后，所有能进水的物品都要加它（不然会直接沉底消失）。`log.lua` 第 23 行：

```lua
MakeInventoryFloatable(inst, "med", 0.1, 0.75)
```

`size`（`"small" / "med" / "large"`）影响漂浮幅度；后面几个可选参数是"漂在水里时切换动画库"的设定。

#### ⑥ 雪覆盖类（对应 `SnowCovered` 标签 + `lunarhailbuildup` 组件）

| 函数 | 效果 |
|------|------|
| `MakeSnowCoveredPristine(inst)` | 客户端视觉："冬天下雪时会积雪"的动画符号 |
| `MakeSnowCovered(inst)` | 等价于 `Pristine` + 监听世界雪状态 + 月球陨冰堆积（联机新机制） |
| `MakeNoGrowInWinter(inst)` | 冬天不再长出浆果/作物 |

#### ⑦ 宠物化/腐烂类（对应 `perishable` + `eater` + `inventoryitem` 组件）

| 函数 | 效果 |
|------|------|
| `MakeSmallPerishableCreaturePristine(inst)` | 客户端预告：可以被放进背包当"生鲜"保存 |
| `MakeSmallPerishableCreature(inst, starvetime, oninventory, ondropped)` | 完整配置：会饿死的小生物 |
| `MakeFeedableSmallLivestockPristine(inst)` | 在 Pristine 版基础上加 `"small_livestock"` 标签 |
| `MakeFeedableSmallLivestock(inst, starvetime, ...)` | 完整配置：可以被喂食、放进笼子的小生物（比如蝴蝶、青蛙、小猪） |

看 `scripts/standardcomponents.lua` 第 958-973 行：

```lua
function MakeFeedableSmallLivestockPristine(inst)
    MakeSmallPerishableCreaturePristine(inst)
    inst:AddTag("small_livestock")
end

function MakeFeedableSmallLivestock(inst, starvetime, oninventory, ondropped)
    MakeFeedableSmallLivestockPristine(inst)  -- 套用 Pristine 版

    if inst.components.eater == nil then
        inst:AddComponent("eater")
    end
    inst.components.eater:SetOnEatFn(oneat)
    MakeSmallPerishableCreature(inst, starvetime, oninventory, ondropped)
end
```

注意这里的嵌套：**非 Pristine 版会先调用 Pristine 版**，再加上组件。这揭示了 5.3.5 节要讲的重要模式。

#### ⑧ 其他实用辅助函数

| 函数 | 用途 |
|------|------|
| `MakeForgeRepairable(inst, material, onbroken, onrepaired)` | 武器/装备可以用特定材料（布鲁 / 月石）在锤子工作站修复 |
| `MakeWaxablePlant(inst)` | 植物可以被蜂蜡永久保鲜 |
| `MakeDeployableFertilizerPristine` + `MakeDeployableFertilizer` | 可以被"放置"在土地上的肥料（而不是扔进桶里） |
| `MakeCraftingMaterialRecycler(inst, data)` | 可以被"拆解"还原成合成材料 |
| `MakeHermitCrabAreaListener(inst, callbackfn)` | 和月岛隐士老奶奶任务挂钩（某物被破坏时让她不高兴） |
| `MakeCollidesWithElectricField(inst)` | 会被电围栏电到 |

---

### 5.3.4 进阶：辅助函数内部到底做了什么

辅助函数通常做以下几件事的组合：

| 步骤 | 内容 | 例子 |
|------|------|------|
| ① | 加底层引擎组件 | `inst.entity:AddPhysics()` |
| ② | 加 Lua 组件 | `inst:AddComponent("burnable")` |
| ③ | 配参数 | `burnable:SetBurnTime(10)` |
| ④ | 加标签（优化或功能） | `inst:AddTag("small_livestock")` |
| ⑤ | 注入回调 | `burnable:SetOnBurntFn(DefaultBurntFn)` |
| ⑥ | 监听事件/世界状态 | `inst:ListenForEvent(...)`、`inst:WatchWorldState(...)` |

并不是每个辅助函数都 6 步全做，但这 6 步是共同词汇表。**理解这一点很重要**——当你 Mod 中想"略微改一改"一个 `MakeXxx` 函数的行为时，你知道该去看它内部的哪几步。

**举个具体场景**：你做了一把 Mod 武器，想让它"被附身时弹飞，**但只有 30% 概率**弹飞"。展开 `MakeHauntableLaunch` 的源码（5.3.3 节 ④ 的代码块），你会看到：

- 第 2 步：加 `hauntable` 组件
- 第 3 步：配 `cooldown`
- 第 5 步：注入 `OnHauntFn`（里面 `chance = chance or TUNING.HAUNT_CHANCE_ALWAYS`）

于是你直接调用 `MakeHauntableLaunch(inst, 0.3)` 就行——第一个参数就是 `chance`，它会覆盖默认的"100% 概率"。

**再来一个场景**：你想让你的新怪物"被烧 20 秒才倒下"（而不是 `MakeMediumBurnableCharacter` 默认的 8 秒）。展开源码发现 `MakeMediumBurnableCharacter` 没有暴露 `time` 参数（写死成 8 秒）。这时你有两个选择：

1. **手动展开**：不调用辅助函数，自己写那 5 行代码，把 `SetBurnTime(8)` 改成 `SetBurnTime(20)`。
2. **先调用再修改**：`MakeMediumBurnableCharacter(inst, "body")` 后立刻 `inst.components.burnable:SetBurnTime(20)`——因为辅助函数只是把组件配好了，后续你完全可以继续修改参数。

第二种更简洁。看 `scripts/prefabs/player_common.lua` 第 2644-2646 行就是这么做的：

```lua
MakeMediumBurnableCharacter(inst, "torso")
inst.components.burnable:SetBurnTime(TUNING.PLAYER_BURN_TIME)
inst.components.burnable.nocharring = true
```

官方自己也是"套用辅助函数，再按需覆盖"。

---

### 5.3.5 进阶：Pristine 版本与非 Pristine 版本

你可能注意到一些辅助函数**成对出现**：

- `MakeSnowCoveredPristine` ↔ `MakeSnowCovered`
- `MakeSmallPerishableCreaturePristine` ↔ `MakeSmallPerishableCreature`
- `MakeFeedableSmallLivestockPristine` ↔ `MakeFeedableSmallLivestock`
- `MakeDeployableFertilizerPristine` ↔ `MakeDeployableFertilizer`

**区别是什么？** 回忆 5.2.2 节的"五段式"——工厂函数被 `SetPristine()` 切成两半：

- 上半段：**客户端和服务端都执行**（只能放客户端也需要的东西——引擎组件、动画、标签）
- 下半段：**仅服务端执行**（Lua 组件、业务逻辑）

**Pristine 版本 = 只做上半段应该做的事**（加标签、加动画 override、改 AnimState），不加 Lua 组件。
**非 Pristine 版本 = 先调 Pristine 版本，再补加 Lua 组件和逻辑**。

对照 `MakeFeedableSmallLivestock` 源码（前面已列出）就能看到：

```lua
function MakeFeedableSmallLivestock(inst, starvetime, oninventory, ondropped)
    MakeFeedableSmallLivestockPristine(inst)  -- 先做客户端也要的
    -- 然后才加 eater 组件（服务端才用）
    if inst.components.eater == nil then
        inst:AddComponent("eater")
    end
    -- ...
end
```

**使用时的正确姿势**：

```lua
local function fn()
    local inst = CreateEntity()
    -- ... 三件套组件 ...
    MakeCharacterPhysics(inst, 1, .5)
    
    MakeFeedableSmallLivestockPristine(inst)  -- ← Pristine 前调 Pristine 版
    
    -- ... 其他客户端共享内容 ...
    
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end
    
    -- 仅服务端
    MakeFeedableSmallLivestock(inst, 30 * 60)  -- ← Pristine 后调非 Pristine 版？
    -- 但这会重复加标签、重复调 Pristine 版本里的东西...
end
```

**实际上 `MakeFeedableSmallLivestock` 已经会检查并跳过重复工作**（内部调用的 `MakeFeedableSmallLivestockPristine` 里只加标签和符号 override，这些是幂等的——重复加标签不会出错）。所以**两种写法都能用**：

- **推荐写法**：只在服务端写一次 `MakeFeedableSmallLivestock(inst, 30 * 60)`（它内部会补上 Pristine 部分）
- **严格写法**：`MakeXxxPristine` 写在 Pristine 前，`MakeXxx` 写在 Pristine 后（这样客户端也能提前看到必要的标签/符号，减少 replica 带宽）

官方在对性能敏感的 prefab（如 `player_common.lua`）里会用严格写法，普通生物 prefab 就懒一点只写一次。

> **新手记忆**：对带 Pristine 后缀的函数**不要慌**——它和对应的非 Pristine 版是亲兄弟，区别是"只处理客户端也要看到的那部分"。你大多数时候只需要用非 Pristine 版，写在 `SetPristine` 之后即可。

---

### 5.3.6 老手进阶：`MakePlayerCharacter` —— 真正的高阶工厂

`scripts/standardcomponents.lua` 里的 `MakeXxx` 都是"**向实体追加一个功能**"。而 `scripts/prefabs/player_common.lua` 里的 `MakePlayerCharacter` 是**完全不同量级**的东西——它一次性产出一个完整的玩家角色 Prefab。

**函数签名**（`scripts/prefabs/player_common.lua` 第 1905 行）：

```lua
local function MakePlayerCharacter(name, customprefabs, customassets, common_postinit, master_postinit, starting_inventory)
```

六个参数的含义：

| 参数 | 含义 | 是否必填 |
|------|------|---------|
| `name` | 角色 prefab 名（`"wilson"`、`"wolfgang"`） | 必填 |
| `customprefabs` | 该角色额外依赖的 prefab 列表（技能树、特效） | 可选 |
| `customassets` | 该角色额外的美术资源（角色专属贴图/动画） | 可选 |
| `common_postinit` | 在 `SetPristine` **之前**被调用的回调 | 可选 |
| `master_postinit` | 在 `SetPristine` **之后**（仅服务端）被调用的回调 | 可选 |
| `starting_inventory` | 初始背包（已弃用，改用 `inst.starting_inventory`） | 已弃用 |

返回值是一个 `Prefab` 对象——所以用法就是"定义两个回调，扔给 `MakePlayerCharacter`"：

看 `scripts/prefabs/wolfgang.lua` 最后一行（第 785 行）：

```lua
return MakePlayerCharacter("wolfgang", prefabs, assets, common_postinit, master_postinit),
       -- ... 其他角色形态 prefab ...
```

`MakePlayerCharacter` 的内部会生成一个长达 1000 多行的工厂函数（`scripts/prefabs/player_common.lua` 第 2300 行开始），里面**加了大约 60 个组件、处理了 20+ 个事件、写入了所有玩家共有的技能树 / 皮肤 / 装备 / 背包等等机制**。任何一个新角色如果从零开始写，就要把这 1000 行复制粘贴一遍——灾难。

所以官方的办法是：**把 1000 行的工厂函数写在 `player_common.lua`，在其中两个关键位置预留回调**：

#### `common_postinit` —— 客户端也要执行的定制

它在工厂函数的 `SetPristine` **之前**被调用（`player_common.lua` 第 2471-2473 行）：

```lua
if common_postinit ~= nil then
    common_postinit(inst)
end

-- ...几行之后...
inst.entity:SetPristine()
```

这个位置放**客户端也需要知道的东西**——比如 `wolfgang.lua` 第 595-638 行的 `common_postinit` 主要做：

```lua
local function common_postinit(inst)
    inst:AddTag("strongman")                -- 客户端需要识别"这是强壮的沃尔夫冈"
    inst:AddTag("mightiness_normal")         -- 客户端 replica 相关的标签
    -- ...
    inst.GetMightiness = GetMightiness       -- 把访问方法挂到 inst 上（客户端也要用）
    inst.bell_percent = 0                    -- 客户端钟摆 UI 状态
    -- ...
end
```

#### `master_postinit` —— 仅服务端的业务逻辑

它在工厂函数的主流程末尾（`player_common.lua` 第 2860-2862 行）：

```lua
if master_postinit ~= nil then
    master_postinit(inst)
end
```

这里放**真正的角色特色逻辑**。`wolfgang.lua` 第 640 行的 `master_postinit` 内容：

```lua
local function master_postinit(inst)
    inst.starting_inventory = start_inv[...] or start_inv.default
    inst.components.hunger:SetMax(TUNING.WOLFGANG_HUNGER)     -- 改饥饿值上限
    inst.components.foodaffinity:AddPrefabAffinity("potato_cooked", ...)  -- 吃土豆增益

    inst.components.health:SetMaxHealth(TUNING.WOLFGANG_HEALTH_NORMAL)
    inst.components.sanity:SetMax(TUNING.WOLFGANG_SANITY)

    inst:AddComponent("mightiness")           -- 强壮值（沃尔夫冈独有）
    inst:AddComponent("dumbbelllifter")       -- 举哑铃
    inst:AddComponent("strongman")            -- 大力士
    inst:AddComponent("expertsailor")         -- 船夫
    inst:AddComponent("coach")                -- 教练
    -- ...
end
```

**为什么要分两个回调？**

1. **`common_postinit` 的东西要被客户端复制**——所以它必须在 `SetPristine` 之前。
2. **`master_postinit` 涉及服务端才有的 Lua 组件**（`mightiness`、`coach` 等）——它必须在"仅服务端"区段内。
3. **中间的那 1500 行共用代码**对所有角色都一样，不需要你关心。

这种设计模式叫**"模板方法 + 钩子函数"**——父级框架定义主流程和骨架，关键节点预留钩子，子级只填写钩子。OOP 语言里类似的东西叫 `TemplateMethod`；Rails、Django 等 Web 框架里叫 `before_save` / `after_save`。**老手看懂这一点就知道官方的架构品味不差**。

---

### 5.3.7 老手进阶：自己设计"族函数"

`MakePlayerCharacter` 的经验可以照搬到你自己的 Mod 里。举个例子：假设你的 Mod 要加 5 种"魔法卷轴"，它们都：

- 可以被手握（都有 `equippable` 组件）
- 使用时播放一段咏唱动画（都共享一段 `OnEquip` / `OnUnequip` 逻辑）
- 使用次数用完就消失（都有 `finiteuses`）
- **但**每个卷轴有不同的魔法效果（一个召唤怪、一个治疗、一个闪电...）

**不用族函数的写法**：复制粘贴 5 份类似的 Prefab 代码，每份都有 60 行公共代码 + 10 行独有逻辑。

**用族函数的写法**（模仿 `MakePlayerCharacter`）：

```lua
-- mods/yourmod/scripts/prefabs/scroll_common.lua
local function MakeScroll(name, assets, prefabs, common_postinit, master_postinit)
    local function fn()
        local inst = CreateEntity()
        inst.entity:AddTransform()
        inst.entity:AddAnimState()
        inst.entity:AddNetwork()

        MakeInventoryPhysics(inst)
        inst.AnimState:SetBank("scroll")
        inst.AnimState:SetBuild("scroll")
        inst.AnimState:PlayAnimation("idle")

        inst:AddTag("scroll")  -- 公共标签
        MakeInventoryFloatable(inst, "small")

        if common_postinit then
            common_postinit(inst)   -- 第一个钩子：客户端也要的差异
        end

        inst.entity:SetPristine()
        if not TheWorld.ismastersim then
            return inst
        end

        inst:AddComponent("inspectable")
        inst:AddComponent("inventoryitem")
        inst:AddComponent("equippable")
        inst.components.equippable:SetOnEquip(default_on_equip)
        inst.components.equippable:SetOnUnequip(default_on_unequip)
        inst:AddComponent("finiteuses")
        inst.components.finiteuses:SetOnFinished(inst.Remove)
        MakeHauntableLaunch(inst)

        if master_postinit then
            master_postinit(inst)   -- 第二个钩子：服务端的差异逻辑
        end

        return inst
    end

    return Prefab(name, fn, assets, prefabs)
end

return MakeScroll
```

每个具体卷轴只需要：

```lua
-- mods/yourmod/scripts/prefabs/scroll_lightning.lua
local MakeScroll = require("prefabs/scroll_common")

local function master_postinit(inst)
    inst.components.finiteuses:SetMaxUses(3)
    inst.components.equippable:SetOnEquip(function(inst, owner)
        -- 闪电卷轴独有的装备逻辑
    end)
end

return MakeScroll("scroll_lightning", assets, prefabs, nil, master_postinit)
```

5 个卷轴 = 1 个 `MakeScroll`（60 行）+ 5 个具体文件（每个 10-20 行）。代码量从 5×70=350 行降到 60+5×15=135 行，并且**只要改一次 `MakeScroll` 就能同步影响所有卷轴**。

---

### 5.3.8 小结

- **辅助函数 = 官方的"配方手册"**。写 Prefab 时遇到"要配物理 / 燃烧 / 冰冻 / 鬼附身 / 漂浮 / 雪覆盖 / 可宠养"这类需求，**第一反应应该是查 `scripts/standardcomponents.lua` 有没有对应的 `MakeXxx`**。
- **最常用的三个物理辅助函数**：`MakeInventoryPhysics`（小物品）、`MakeCharacterPhysics`（生物）、`MakeObstaclePhysics`（建筑/障碍）。
- **燃烧类和冰冻类**按体型分级（Small/Medium/Large/Huge），每级抗性不同。生物专用 `xxxCharacter` 版本。
- **鬼附身类**近 20 个 `MakeHauntableXxx`——它们本质上都是"加 `hauntable` 组件 + 注入不同的 `OnHauntFn` 回调"。
- **辅助函数内部 = 底层组件 + 参数 + 标签 + 回调 + 监听器**的组合；展开源码能让你知道"该改哪一步"。
- **Pristine / 非 Pristine 成对的函数**：Pristine 版只放上半段（客户端也要的），非 Pristine 版包含 Pristine 版并追加下半段（服务端的）。大多数时候你只用非 Pristine 版就够了。
- **`MakePlayerCharacter` 是"高阶工厂"**——它不是"追加功能"，而是"生成一整个 Prefab"。用 **`common_postinit`（客户端可见的差异）** + **`master_postinit`（仅服务端的差异）** 两个钩子把 1000 多行的共同初始化复用到所有 18 个角色。
- **自己设计族函数**就照搬 `MakePlayerCharacter` 的模式：主体放在 `xxx_common.lua`，在 `SetPristine` 前后各预留一个钩子，每个具体 Prefab 文件只写差异部分。

> **下一节预告**：5.4 节会聚焦**本章最核心也最容易踩坑的机制**——`SetPristine` 分界线背后的"客户端/服务端双执行"究竟意味着什么？为什么一段 Prefab 文件代码会被**同一台电脑执行两次**？replica 是什么？我们会跟着一个实体在主机和客户端的完整生命流程走一遍，彻底理解"两段式"结构的来龙去脉。


## 5.4 Pristine State——为什么 Prefab 有两段代码

### 本节导读

5.2 节我们提到工厂函数的"五段式"——`SetPristine()` + `if not TheWorld.ismastersim then return inst end` 这两行把工厂函数切成了"前半段（主客都执行）"和"后半段（只有主机执行）"。5.3 节我们见过 `MakePlayerCharacter` 用 `common_postinit / master_postinit` 两个钩子分别插入这两段。

但始终有一个核心问题没被真正解释：**同一段 Prefab 文件的代码，到底是被"谁"、"在什么情况下"执行了几次？为什么非得分成两段？**

这一节就彻底讲清楚。

> **新手**看 5.4.1 和 5.4.2 就够——先记住"主机/客户端都会跑一遍这段代码"这个事实，以及"上半段两边都跑、下半段只有主机跑"的写法；**进阶读者**要深入理解 `pristine`（原始快照）的语义、replica 系统的工作方式、以及 `net_xxx` 变量怎么把服务端改动推到客户端；**老手**可以跟着"一把长矛的完整旅程"把每一步抽丝剥茧走一遍，并看懂"写 Mod 时最容易踩的 5 个坑"。

---

### 5.4.1 快速入门：同一段代码为什么被执行两次？

**饥荒联机版是客户端/服务端架构的游戏**。任何联机游戏都有两类参与者：

- **主机（host）**：真实跑游戏世界的一台机器（开联机时你自己的电脑，或者饥荒专用服务器）
- **客户端（client）**：其他玩家的电脑，只是"看"这个世界（实际只跑一部分逻辑，大多数事情都由主机决定）

问题来了：**Prefab 文件（比如 `scripts/prefabs/spear.lua`）在主机和客户端上都会被加载、执行**。原因很简单——客户端也要能**显示**一把长矛（贴图、动画、物理），这些逻辑不跑一遍，客户端拿什么渲染？

所以，你在 `spear.lua` 里写的工厂函数 `fn`，在**主机上生成长矛时会被调用一次**，同时**每一个连进来的客户端上也会被调用一次**。这就是"一段代码被执行多次"的真相。

**`TheWorld.ismastersim` 就是用来区分身份的**。打开 `scripts/prefabs/world.lua` 第 425-426 行：

```lua
inst.ismastersim = TheNet:GetIsMasterSimulation()
inst.ismastershard = inst.ismastersim and not TheShard:IsSecondary()
```

`TheNet:GetIsMasterSimulation()` 是 C++ 引擎的接口——**主机返回 `true`，客户端返回 `false`**。所以 `TheWorld.ismastersim` 在主机上是 `true`，在客户端上是 `false`。这个值在游戏启动时被设置，整个游戏生命周期都不会变。

于是你在工厂函数中写：

```lua
if not TheWorld.ismastersim then
    return inst
end
-- 这下面的代码只有主机会执行
```

就等同于说："如果我不是主机，到此为止；下面的代码只有主机才往下走。"

---

### 5.4.2 快速入门：把工厂函数拆成两段的意义

既然主机和客户端都会执行工厂函数，为什么不让两边都跑同样的代码（反正都会读到 `if not TheWorld.ismastersim`）？为什么要把代码有意识地分成两段？

两个核心原因：

#### 原因 1：Lua 组件（如 `health`、`weapon`）只属于主机

游戏世界的**真实状态**只存在于主机——一把长矛的剩余耐久、一只蜘蛛的血量、一个玩家背包里有什么，这些都是主机算的，客户端只是被动接收。因此：

- `inst:AddComponent("health")` 创建的 `health` 组件，**只在主机上有意义**
- 客户端根本不应该也不需要创建这个组件

如果你在客户端也 `AddComponent("health")`，不仅是浪费内存——**客户端上那个"health 组件"是完全隔离的、不会和主机同步的假组件**。玩家砍一下长矛，主机上 `health:DoDelta(-10)`，客户端那个 `health.currenthealth` 根本不变，UI 显示永远是错的。

所以规则很明确：**凡是 `AddComponent` 开头的业务逻辑（除了少数例外），都应该放在 `SetPristine` 分界线下面**。

#### 原因 2：客户端也需要"最基本的实体样貌"

但反过来，客户端也不能完全一无所知——它至少要能：
- 正确显示长矛的模型和动画（`AnimState:SetBank / SetBuild / PlayAnimation`）
- 知道这是把"武器"（`inst:AddTag("weapon")`）以便玩家鼠标悬停提示
- 知道它是个可以漂在水上的物品（`MakeInventoryFloatable`）
- 配置好基本的物理，这样玩家看到它"咣当"掉在地上

这些**视觉和判定信息**必须两边都执行——这就是分界线**上半段**要做的事。

#### 分界线本身：`SetPristine` 的作用

`inst.entity:SetPristine()` 是一个 C++ 引擎提供的接口。它做的事大致可以理解为：

> "当前为止我给你看到的状态（tag、AnimState 配置、Physics 配置、引擎组件数据），请引擎把这份快照**原样保存**——以后任何新客户端连进来，只要用这份快照就能重建实体。之后我做的任何修改不再自动同步。"

这份"pristine 快照"会随着实体第一次被同步给客户端一起发过去。客户端拿到这份快照后跑一遍工厂函数，跑到 `SetPristine` 时和主机达成同一个起点。此后客户端再运行到 `if not TheWorld.ismastersim then return inst end` 就退出——主机还要继续往下加各种 Lua 组件，客户端到此打住。

所以两段代码的写作规则：

| 段落 | 放什么 | 为什么 |
|------|--------|--------|
| **上半段**（`SetPristine` 之前） | `AddTransform / AddAnimState / AddNetwork`、动画 `SetBank/SetBuild`、物理 `MakeXxxPhysics`、`AddTag`、`MakeInventoryFloatable`、事件监听（客户端也要监听的）等 | 客户端需要用来**正确显示和判定**这个实体 |
| **下半段**（`SetPristine` 之后 & `if not TheWorld.ismastersim` 过滤后） | `AddComponent` 各种业务组件、设置 `SetDamage/SetMaxUses`、`SetStateGraph`、`SetBrain`、`ListenForEvent`（仅服务端的）等 | 只在主机上跑的真实游戏逻辑 |

> **新手记忆**：**"看得见的放上半段、算得出来的放下半段"**。看得见 = 模型、动画、物理、可点击性、标签；算得出来 = 血量、伤害、AI、玩家交互后果。

---

### 5.4.3 进阶：pristine 的字面意义

"pristine" 在英文里意思是 **"原始的、未受污染的"**，也可以译成"纯净初始状态"。用到这里非常贴切：

- 实体被 `CreateEntity` 刚出生时，是"空壳"
- 你在工厂函数的**上半段**往它身上加东西，它还处于"**正在被构造**"的状态
- `SetPristine()` 一调，"构造完成"——此时的状态是这个实体最**干净、原始的初始面貌**
- 之后主机再往里加 Lua 组件、改状态，就进入"**运行时状态**"，不再属于"原始"

为什么要区分"原始状态"和"运行时状态"？因为**网络同步效率**：

- 原始状态对**所有**客户端都一样——无论哪个客户端，连进来时看到的新生成长矛都是同一个样子。所以服务端只需要序列化一份 pristine state 广播出去，不需要每个客户端单独算。
- 运行时状态是**不断变化**的（耐久从 100 降到 99 再降到 98 …）。不同时刻连进来的客户端看到的值不一样，需要通过 replica 系统持续同步。

**两份状态对应两种同步机制**：

| 状态 | 同步机制 | 时机 |
|------|----------|------|
| Pristine state | 打包一次随实体广播（`SetPristine` 冻结） | 实体**首次被同步给客户端**时 |
| 运行时状态 | 通过 replica 的 `net_xxx` 变量增量推送 | 状态变化时，实时（或下一次网络 tick） |

这就是为什么工厂函数必须"分两段写"——**两段对应两种同步渠道**。上半段写错，客户端看到的初始长矛就是错的；下半段写错（比如塞到上半段），客户端会为"根本用不到的 Lua 组件"白白初始化一堆垃圾。

---

### 5.4.4 进阶：replica 系统 —— 客户端怎么"复刻"组件

你会发现一个矛盾的现象：

- 客户端**没有** `inst.components.health`（因为 `AddComponent("health")` 只在主机执行）
- 但客户端的 UI 上**能正确显示血条**

中间缺的就是 **replica（复制品）** 系统。打开 `scripts/entityreplica.lua` 第 5-26 行：

```lua
local REPLICATABLE_COMPONENTS =
{
    builder = true,
    combat = true,
    container = true,
    constructionsite = true,
    equippable = true,
    fishingrod = true,
    follower = true,
    health = true,
    hunger = true,
    inventory = true,
    inventoryitem = true,
    moisture = true,
    named = true,
    oceanfishingrod = true,
    rider = true,
    sanity = true,
    sheltered = true,
    stackable = true,
    writeable = true,
}
```

**这张白名单里的组件有 18 个**——它们是"两边都需要感知"的组件。每个白名单组件都必须有两个对应文件：

- `scripts/components/xxx.lua`——**服务端组件**（有真正的数据，只在主机跑）
- `scripts/components/xxx_replica.lua`——**客户端复制品**（通过 `net_xxx` 变量镜像服务端状态）

以 `health` 为例：

- `scripts/components/health.lua` 存放 `currenthealth`、`maxhealth` 等字段，`DoDelta(-10)` 这种方法
- `scripts/components/health_replica.lua` 存放 `self.classified.currenthealth`（一个 `net_xxx` 变量），只提供 `GetPercent()` / `GetCurrent()` / `IsDead()` 之类的**只读**查询方法

客户端 UI 要画血条时，会查 `inst.replica.health:GetPercent()`——它实际读的是 replica 里的 net 变量的 `:value()`，而 net 变量的值是从服务端自动同步过来的。

#### 自动 replicate 的流程

看 `scripts/entityreplica.lua` 第 34-60 行 `ReplicateComponent`：

```lua
function EntityScript:ReplicateComponent(name)
    if not REPLICATABLE_COMPONENTS[name] then
        return
    end

    if TheWorld.ismastersim then
        self:AddTag("_"..name)
        if self:HasTag("__"..name) then
            self:RemoveTag("__"..name)
            return
        end
    end

    if rawget(self.replica, "_")[name] ~= nil then
        print("replica "..name.." already exists! "..debugstack_oneline(3))
    end

    local filename = name.."_replica"
    local cmp = Replicas[filename]
    if cmp == nil then
        cmp = require("components/"..filename)
        Replicas[filename] = cmp
    end
    assert(cmp ~= nil, "replica "..name.." does not exist!")

    rawset(self.replica._, name, cmp(self))
end
```

重点逻辑：

1. 白名单拦截——不在 18 个组件里的就直接 return，**不会被 replicate**
2. 如果是主机，**给实体加一个 `_xxx` 标签**（如 `_health`）——这个标签是"这个实体挂着 health 组件"的信号
3. `require("components/xxx_replica")` 加载对应的 replica 类
4. 用 `cmp(self)` 创建 replica 实例，挂到 `inst.replica._[name]`

而这一切的触发，在 `EntityScript:AddComponent`（`scripts/entityscript.lua` 第 610-646 行）里：

```lua
function EntityScript:AddComponent(name)
    -- ...检查是否已存在...
    local cmp = LoadComponent(name)
    -- ...
    self:ReplicateComponent(name)  -- ← 每次 AddComponent，顺手 ReplicateComponent
    local loadedcmp = cmp(self)
    self.components[name] = loadedcmp
    -- ...
end
```

**关键点**：**主机每次 `AddComponent("health")`，`ReplicateComponent("health")` 都会自动跟着跑**，顺手把 `_health` 标签加到实体身上。这个标签会被同步到所有客户端。

#### 客户端的 ReplicateEntity

客户端首次接收到实体的 pristine state 后，引擎会调用 `scripts/entityreplica.lua` 第 75-85 行：

```lua
--Triggered on clients immediately after initial deserialization of tags from construction
function EntityScript:ReplicateEntity()
    for k, v in pairs(REPLICATABLE_COMPONENTS) do
        if v and (self:HasTag("_"..k) or self:HasTag("__"..k)) then
            self:ReplicateComponent(k)
        end
    end

    if self.OnEntityReplicated ~= nil then
        self:OnEntityReplicated()
    end
end
```

客户端做的事：

1. 遍历 18 个白名单组件
2. 如果实体有 `_health` 标签（主机上那边加的，随 pristine state 一起同步过来），就 `ReplicateComponent("health")`——**在客户端挂一个 `health_replica`**
3. 所有 replica 挂好后，回调 `inst.OnEntityReplicated`（回顾 5.2.4 节的生命周期回调）

**这条链路完美闭环**：

```
主机 AddComponent("health")
  → ReplicateComponent("health")
    → AddTag("_health")
    → 主机的 inst.replica.health 实例化（以 classified 为连接点）

↓ 网络同步 ↓

客户端接到 pristine state（带 _health 标签）
  → ReplicateEntity()
    → 检测到 _health 标签
    → ReplicateComponent("health")
      → 客户端的 inst.replica.health 实例化
      → health_replica 构造函数里订阅 net 变量
```

---

### 5.4.5 进阶：net 变量 —— 状态变化的桥梁

光有 replica 类不够，还缺最后一公里：**主机上 health 从 100 变成 90 时，客户端的 `health_replica` 怎么知道？**

答案是 **net 变量**。打开 `scripts/components/stackable_replica.lua` 第 24-37 行（一个精简的 replica 示例）：

```lua
local Stackable = Class(function(self, inst)
    self.inst = inst

    self._stacksize = net_smallbyte(inst.GUID, "stackable._stacksize", "stacksizedirty")
    self._stacksizeupper = net_smallbyte(inst.GUID, "stackable._stacksizeupper", "stacksizedirty")
    self._ignoremaxsize = net_bool(inst.GUID, "stackable._ignoremaxsize")
    self._maxsize = net_tinybyte(inst.GUID, "stackable._maxsize")

    if not TheWorld.ismastersim then
        inst:ListenForEvent("stacksizedirty", OnStackSizeDirty)
    end
end)
```

这里声明了四个 **net 变量**。`net_smallbyte / net_bool / net_tinybyte` 都是 C++ 引擎暴露的 Lua 函数——它们创建的"变量"不是普通 Lua 值，而是**一个会自动在主机和客户端之间同步的网络变量**。

**net 变量常见类型**：

| 类型 | 数据范围 | 典型用途 |
|------|----------|----------|
| `net_bool` | 1 bit | 状态开关（是否着火、是否冻住） |
| `net_tinybyte` | 4 bit (0-15) | 很小的枚举 |
| `net_smallbyte` | 6 bit (0-63) | 小型计数（stack size 低位） |
| `net_byte` | 8 bit (0-255) | 一般计数 |
| `net_shortint` | 16 bit | 百分比、血量等 |
| `net_uint` | 32 bit | 长整型 |
| `net_float` | 浮点 | 连续值（温度、朝向） |
| `net_string` | 字符串 | 名字、消息 |
| `net_entity` | 另一个实体的 GUID | 引用另一个实体 |
| `net_hash` | 哈希值 | prefab 名、事件名 |
| `net_event` | 仅事件信号，无值 | 触发客户端事件 |

**构造参数三件套**：

```lua
net_xxx(inst.GUID, "唯一标识符", "变化时触发的事件名")
```

- `inst.GUID`：归属的实体
- `"唯一标识符"`：在这个实体内的唯一名字（用于 C++ 引擎识别）
- `"xxxdirty"`：**当值改变时，引擎会自动在该实体上 `PushEvent("xxxdirty")`**——客户端只需要 `ListenForEvent("xxxdirty", ...)` 就能知道"状态变了，该刷新 UI 了"

#### 服务端写、客户端读

以 `stackable` 为例看完整的数据流：

**服务端**（`scripts/components/stackable.lua` 第 3-9 行）：

```lua
local function onstacksize(self, stacksize)
    self.inst.replica.stackable:SetStackSize(stacksize)
    -- ...
end
```

当真实的 `self.stacksize` 改变时（比如 `stackable:SetStackSize(5)`），自动调用 `replica.stackable:SetStackSize(5)`。后者内部（`stackable_replica.lua` 第 47-64 行）：

```lua
function Stackable:SetStackSize(stacksize)
    stacksize = stacksize - 1
    if stacksize <= 63 then
        self._stacksizeupper:set(0)
        self._stacksize:set(stacksize)
    -- ...
    end
end
```

`:set(value)` 是**写入 net 变量的专用方法**。一旦写入，C++ 引擎就会把这次变化打包发给所有客户端。

**客户端**（`stackable_replica.lua` 第 11-22 行）：

```lua
local function OnStackSizeDirty(inst)
    local self = inst.replica.stackable
    -- ...
    self:ClearPreviewStackSize()
    inst:PushEvent("inventoryitem_stacksizedirty")
end

-- 构造函数里注册监听
if not TheWorld.ismastersim then
    inst:ListenForEvent("stacksizedirty", OnStackSizeDirty)
end
```

客户端监听 `"stacksizedirty"` 事件——net 变量一变，引擎自动 push 这个事件，客户端跟着做必要的响应（如清理预览值、通知 inventoryitem replica 更新）。

读取值也有专用方法 `:value()`——如 `stackable_replica.lua` 第 106-108 行：

```lua
function Stackable:StackSize()
    return self:GetPreviewStackSize() or (self._stacksizeupper:value() * 64 + self._stacksize:value() + 1)
end
```

客户端 UI 问"这堆东西有多少个？"，最终走到 net 变量的 `:value()`。

---

### 5.4.6 老手进阶：一把 `spear` 的完整旅程

把前面所有内容串起来，跟着一把长矛走一遍完整流程：

```
═════════════════════════════════════════════════════════════════════════
Phase 1: 主机生成长矛
═════════════════════════════════════════════════════════════════════════

玩家按下合成按钮 / 控制台 c_spawn("spear")
         ↓
    SpawnPrefab("spear")    [scripts/mainfunctions.lua: 403]
         ↓
    TheSim:SpawnPrefab → C++
         ↓ C++ 回调
    SpawnPrefabFromSim("spear")    [scripts/mainfunctions.lua: 347]
         ↓
    prefab.fn(TheSim)   ← 这就是 scripts/prefabs/spear.lua 的 fn
         ↓
    ┌─ 工厂函数 fn 的执行（主机版）────────────────────────────────┐
    │   inst = CreateEntity()                                        │
    │   inst.entity:AddTransform() / AddAnimState() / AddNetwork()   │ 上半段
    │   MakeInventoryPhysics(inst)                                   │ 主机也跑
    │   AnimState:SetBank("spear") / SetBuild("swap_spear")          │
    │   inst:AddTag("sharp") / "pointy" / "weapon"                   │
    │   MakeInventoryFloatable(inst, "med", ...)                     │
    │                                                                 │
    │   inst.entity:SetPristine()   ← ★ 引擎此时保存 pristine 快照  │
    │   if not TheWorld.ismastersim then return inst end  ←  false   │
    │     ↓ 主机继续                                                  │
    │   inst:AddComponent("weapon")   → ReplicateComponent("weapon")  │
    │     ↓ 但 weapon 不在白名单                                       │
    │     → 无操作                                                     │
    │   inst:AddComponent("finiteuses")  → 同上，无 replica           │ 下半段
    │   inst:AddComponent("inventoryitem") → 白名单！                  │ 仅主机
    │     → AddTag("_inventoryitem")                                  │
    │     → 主机创建 inventoryitem_replica 实例                       │
    │   inst:AddComponent("equippable") → 白名单！                    │
    │     → AddTag("_equippable")                                     │
    │     → 创建 equippable_replica                                   │
    │   MakeHauntableLaunch(inst) → AddComponent("hauntable")         │
    │     → hauntable 不在白名单，无 replica                          │
    └─────────────────────────────────────────────────────────────────┘
         ↓
    Mod 的 AddPrefabPostInit("spear", fn) 被执行（5.1.5 节）
         ↓
    TheWorld:PushEvent("entity_spawned", inst)
         ↓
    引擎把「pristine 快照 + 标签列表」打包成网络包

═════════════════════════════════════════════════════════════════════════
Phase 2: 网络同步到客户端
═════════════════════════════════════════════════════════════════════════

主机把数据通过网络发给所有客户端
         ↓
    客户端 C++ 引擎收到实体创建包
         ↓
    在客户端也调用 SpawnPrefabFromSim("spear") ← 注意是同一段代码
         ↓
    ┌─ 工厂函数 fn 的执行（客户端版）──────────────────────────────┐
    │   inst = CreateEntity()                                        │
    │   inst.entity:AddTransform() / AddAnimState() / AddNetwork()   │ 上半段
    │   MakeInventoryPhysics(inst)                                   │ 客户端也跑
    │   AnimState:SetBank / SetBuild                                 │ （和主机一致）
    │   inst:AddTag("sharp") / "pointy" / "weapon"                   │
    │   MakeInventoryFloatable(inst, ...)                            │
    │                                                                 │
    │   inst.entity:SetPristine()   ← 客户端达到与主机同步的起点     │
    │   if not TheWorld.ismastersim then return inst end  ←  true    │
    │     ↓ 客户端到此停止                                            │
    │   return inst                                                  │
    └─────────────────────────────────────────────────────────────────┘
         ↓
    此时主机发来的 pristine state 被反序列化——包括标签 _inventoryitem、_equippable
         ↓
    引擎触发 ReplicateEntity()    [scripts/entityreplica.lua: 75]
         ↓
    遍历 REPLICATABLE_COMPONENTS 白名单
         ↓
    检测到 _inventoryitem 标签 → ReplicateComponent("inventoryitem")
         → 客户端创建 inventoryitem_replica
         → 其构造函数内部声明 net 变量订阅服务端状态
    检测到 _equippable 标签 → ReplicateComponent("equippable")
         → 客户端创建 equippable_replica
         ↓
    如果 inst.OnEntityReplicated 存在，此时被调用
         ↓
    客户端的长矛"完全就位"——玩家看得见、能点击

═════════════════════════════════════════════════════════════════════════
Phase 3: 运行时状态变化
═════════════════════════════════════════════════════════════════════════

玩家在主机用长矛攻击 → 消耗 1 次耐久
         ↓
    主机的 finiteuses:Use() 内部触发 replica 更新
         ↓
    net_xxx:set(new_value)   ← 写入 net 变量
         ↓
    C++ 引擎打包变化广播给客户端
         ↓
    客户端 net 变量自动更新 → 触发 "xxxdirty" 事件
         ↓
    客户端 UI 监听者刷新（比如耐久条）
```

**有了这张图，你就能解释所有"为什么"了**：

- **为什么工厂函数要分两段写？** 因为主客双执行，而 Lua 组件只属于主机。
- **为什么 `_health / _inventoryitem` 这种下划线标签是标准约定？** 因为它们是 replica 系统的"组件标记"，靠 pristine state 同步。
- **为什么一些 prefab 有 `"weapon (from weapon component) added to pristine state for optimization"` 这种注释？** 看 `scripts/prefabs/spear.lua` 第 44-45 行——`weapon` 组件本来要加 `"weapon"` 标签，但 `weapon` 不在白名单，默认不会把 `"weapon"` 标签同步到客户端。但客户端又需要这个标签识别"这是武器"。所以官方**手动提前在 pristine 阶段加了 `"weapon"` 标签**，跟着 pristine state 一起同步，省了单独走 replica 的开销。类似的还有 `player_common.lua` 第 2485-2492 行手动加的 `_health / _hunger / _sanity / _builder / _combat / _moisture / _sheltered / _rider` 标签：

```lua
--Sneak these into pristine state for optimization
inst:AddTag("_health")
inst:AddTag("_hunger")
inst:AddTag("_sanity")
inst:AddTag("_builder")
inst:AddTag("_combat")
inst:AddTag("_moisture")
inst:AddTag("_sheltered")
inst:AddTag("_rider")
```

注释里写得很清楚：**"Sneak these into pristine state for optimization"（为了优化，把这些偷偷塞进 pristine 状态里）**。效果是玩家客户端一接到实体就知道要准备 `health / hunger / sanity` 这些 replica——不用等 `ReplicateComponent` 在实体到达后再加 tag 再同步。这是关键的"玩家角色预热"性能优化。

---

### 5.4.7 老手进阶：Mod 写 Pristine/Replica 时的陷阱

理解了原理，我们来看 Mod 开发中**最容易踩的 5 个坑**：

#### 陷阱 1：在客户端访问 `inst.components.xxx`

```lua
-- 错误示例
local function OnClientUpdate(inst)
    local hp = inst.components.health.currenthealth  -- ← 客户端上 components.health 不存在！
end
```

**客户端没有 `inst.components.health`**——它只有 `inst.replica.health`。正确写法：

```lua
local function OnClientUpdate(inst)
    -- 两边通用的安全写法
    local hp = inst.replica.health:GetCurrent()       -- 同时支持主客
    -- 或：
    local current = (inst.components.health and inst.components.health.currenthealth)
                    or (inst.replica.health and inst.replica.health:GetCurrent())
                    or 0
end
```

注意 `health_replica.lua` 第 100-108 行的 `GetCurrent` 已经内部做了双路兼容：

```lua
function Health:GetCurrent()
    if self.inst.components.health ~= nil then
        return self.inst.components.health.currenthealth    -- 主机走这
    elseif self.classified ~= nil then
        return self.classified.currenthealth:value()        -- 客户端走这
    else
        return 100
    end
end
```

所以**调用 `inst.replica.health:GetCurrent()` 在主客两边都能工作**——它是统一的查询接口。这就是为什么很多 Mod 作者推荐"能用 replica 就用 replica"。

#### 陷阱 2：在 `SetPristine` 之前 `AddComponent`

```lua
-- 反例：随意乱写的顺序
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddNetwork()
    
    inst:AddComponent("health")    -- ← 在 SetPristine 之前就加了主机组件？
    inst.components.health:SetMaxHealth(100)
    
    inst.entity:SetPristine()
    -- ...
end
```

两个问题：

1. **客户端也会执行 `AddComponent("health")`**——但客户端根本不需要主机版的 `health` 组件，纯浪费。
2. `ReplicateComponent("health")` 会在两边都跑一次，客户端凭空多了个 replica，但它的 classified 连接可能错乱。

正确顺序：`SetPristine` → `if not ismastersim then return` → 之后才 `AddComponent`。

> **例外**：官方偶尔会故意在 pristine 阶段加 Lua 组件——比如 `spear.lua` 第 49 行前就加了 `MakeInventoryFloatable`（内部 `AddComponent("floater")`）。这是因为 `floater` 是**客户端也需要看的视觉组件**——它靠覆盖动画来表现"在水上漂"，客户端必须能算出视觉。**规则不是绝对的，而是"是否客户端需要"**——需要就加在 pristine 前，不需要就加在后面。

#### 陷阱 3：自定义组件想让客户端也能访问

Mod 里常见场景：你加了自己的组件 `mymod_mightiness`，想让客户端的 UI 显示它的值。默认情况下这个组件**不会**被 replicate（不在白名单）。怎么做？

**方案 A（推荐）：注册成可 replicated 组件 + 提供 replica 类**

```lua
-- 在 modmain.lua 里
AddReplicableComponent("mymod_mightiness")
```

然后在 `scripts/components/mymod_mightiness_replica.lua` 里写对应的 replica 类，结构照抄 `health_replica.lua`。

`AddReplicableComponent` 在 `scripts/entityreplica.lua` 第 98-100 行定义：

```lua
function AddReplicableComponent(name)
    REPLICATABLE_COMPONENTS[name] = true
end
```

就是把你的组件名加到白名单里。之后 `AddComponent("mymod_mightiness")` 会自动 `ReplicateComponent`。

**方案 B（简单需求）：用 `net_xxx` 变量挂到 `inst` 上直接同步**

不想写完整的 replica 类？只有一两个字段要同步？直接 `net_xxx`：

```lua
-- 在 pristine 前（两边都跑）
inst._mightiness = net_shortint(inst.GUID, "mymod._mightiness", "mightinessdirty")

-- 在 SetPristine 之后（仅主机）
inst._mightiness:set(100)    -- 服务端写
```

客户端 `inst._mightiness:value()` 读即可。看 `wolfgang.lua` 的 `_isavatar = net_bool(...)` 就是这种写法。适合"单值"场景。

#### 陷阱 4：在 pristine 之前用 `inst.components.xxx`

有些 Mod 作者想在 pristine 之前就配置组件参数：

```lua
-- 反例
inst:AddComponent("floater")
inst.components.floater:SetSize("med")   -- ← 在 pristine 之前就取用组件
inst.entity:SetPristine()
```

问题是**客户端也会跑这行**——客户端虽然也加了 `floater` 组件（前面说过 floater 是例外），但如果这个组件只存在服务端（比如 `health`），`inst.components.health` 在客户端是 `nil`，`:SetMaxHealth` 会抛 "attempt to index a nil value"。

保险写法：**把需要访问 `components` 的调用一律放到 pristine 之后、`ismastersim` 判断之内**。`MakeXxx` 辅助函数除外——它们内部自己判断过了。

#### 陷阱 5：以为 `inst.OnEntityReplicated` 是服务端回调

```lua
-- 反例
inst.OnEntityReplicated = function(inst)
    inst.components.weapon:SetDamage(100)  -- ← components.weapon 在客户端是 nil！
end
```

回顾 5.2.4 节——`OnEntityReplicated` **只在客户端触发**。它的作用是"客户端在 replica 挂好之后做最终初始化"。这里如果要访问，应该用 `inst.replica.xxx`。

---

### 5.4.8 小结

- **联机版的 Prefab 文件在主机和客户端上都会被加载、执行**——这就是"两段式"写法的根本原因。
- **`TheWorld.ismastersim`** 来自 `TheNet:GetIsMasterSimulation()`：主机上 `true`，客户端上 `false`。`if not TheWorld.ismastersim then return inst end` 是两段的边界。
- **`inst.entity:SetPristine()`** 告诉引擎"当前状态是这个实体的初始快照，保存起来发给后续加入的客户端"。此后主机的修改走 replica 系统增量同步。
- **Pristine state 同步** = 打包快照一次性发送；**运行时状态同步** = net 变量增量推送。两种机制对应工厂函数的两段代码。
- **Replica 系统** = 白名单（`REPLICATABLE_COMPONENTS` 共 18 个）+ `components/xxx_replica.lua` 镜像类 + `_xxx` 标签作为信号。主机 `AddComponent` 时自动 `ReplicateComponent`，客户端通过标签识别并创建 replica。
- **net 变量**（`net_bool / net_byte / net_string / net_entity / net_event` 等）是主客之间自动同步的"桥梁变量"。`:set(v)` 写、`:value()` 读、变化时 push `xxxdirty` 事件。
- **跟着一把长矛走一遍完整流程**：主机生成 → pristine 打包 → 广播 → 客户端执行同一段代码到 SetPristine → 客户端 `ReplicateEntity` 按标签补齐 replica → 运行时状态变化通过 net 变量推送。
- **写 Mod 的 5 个陷阱**：(1) 客户端用 `inst.components.xxx`（应该用 `inst.replica.xxx`）；(2) 在 pristine 前 `AddComponent`（浪费 + 错乱）；(3) 自定义组件不 replicate（用 `AddReplicableComponent` 或直接 `net_xxx`）；(4) 在 pristine 前取用 `components`（客户端可能是 nil）；(5) 把 `OnEntityReplicated` 当成服务端回调（它只在客户端）。
- **`sneak these into pristine state for optimization`** 是官方代码里反复出现的注释——**预先加标签到 pristine 状态**，可以避免标签走 replica 造成的延迟。`player_common.lua` 第 2485-2492 行的 `_health / _hunger / _sanity` 等标签就是这种优化。

> **下一节预告**：5.5 节我们专门讲一下贯穿这一章反复出现的 **Tags 系统** —— 实体的"标签身份"。为什么在联机版里 tag 是比 component 更核心的机制？客户端靠 tag 判断战斗 / 交互资格是怎么工作的？`HasTag / AddTag` 和 `"_combat" / "playerghost" / "_burnable"` 这些下划线开头/普通命名的标签各自扮演什么角色？


## 5.5 Tags 系统——实体的"标签身份"

（待编写）

## 5.6 Asset 声明与资源加载机制（RegisterPrefabs 流程）

（待编写）

## 5.7 实战：创建一个全新的自定义物品

（待编写）
