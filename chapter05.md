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

### 本节导读

在前面四节里，你已经见过 tag 无数次——`inst:AddTag("weapon")`、`inst:HasTag("monster")`、`TheSim:FindEntities(..., nil, SPIDER_IGNORE_TAGS, SPIDER_TAGS)`、Pristine state 偷偷夹带的 `_health` 标签……

Tag（标签）几乎是**饥荒联机版中出现频率最高的机制**，超过组件（component）、超过事件（event）。它看起来"简单得没什么内容"——不就是一串字符串嘛——但实际上它承担了整个游戏的**身份识别、网络同步、世界搜索**这三大职责。

> **新手**只需要 5.5.1 和 5.5.2：理解 tag 是什么，为什么要用它、什么时候用它；**进阶读者**应该吃透 5.5.3-5.5.5：tag 的 API 全貌、五类典型标签、以及 tag 和 `TheSim:FindEntities` 的组合套路；**老手**直接跳到 5.5.6-5.5.7：tag 的设计品味、Mod 里如何和 tag 系统融洽相处、以及一些进阶手法。

---

### 5.5.1 快速入门：Tag 是什么？

**Tag 就是一串字符串**。你可以把它理解成**贴在实体身上的"身份标签纸"**：

- 一把长矛身上贴着 `"weapon"`、`"sharp"`、`"pointy"` 三张标签（见 `scripts/prefabs/spear.lua` 第 41-45 行）
- 一只蜘蛛身上贴着 `"cavedweller"`、`"monster"`、`"hostile"`、`"scarytoprey"`、`"canbetrapped"`、`"smallcreature"`、`"spider"`、`"drop_inventory_onpickup"`、`"drop_inventory_onmurder"` 九张（见 `scripts/prefabs/spider.lua` 第 586-594 行）
- 一个玩家身上贴着 `"player"`、`"scarytoprey"`、`"character"`、`"lightningtarget"`、`"_health"`、`"_hunger"`、`"_sanity"` 等二十来张

**三个基础 API**（`scripts/entityscript.lua` 第 556-574 行）：

```lua
function EntityScript:AddTag(tag)
    self.entity:AddTag(tag)
end

function EntityScript:RemoveTag(tag)
    self.entity:RemoveTag(tag)
end

function EntityScript:HasTag(tag)
    return self.entity:HasTag(tag)
end
```

可以看到——Lua 层的 `AddTag / RemoveTag / HasTag` 都是**一行转发**给 C++ 层的 `self.entity:AddTag(...)`。**真正的 tag 存储和查找都在 C++ 引擎里**，Lua 只是接口。所以它极其**轻量、高效**：查一个实体有没有 `"monster"` 标签，底层就是一次哈希集合查找。

**实战示例**：

```lua
-- 加标签
inst:AddTag("weapon")
inst:AddTag("sharp")

-- 查标签
if inst:HasTag("monster") then
    -- 这是个怪物
end

-- 条件式加/删
inst:AddOrRemoveTag("isdead", health_is_zero)

-- 删除标签
inst:RemoveTag("invisible")
```

除了这三个基础 API，`EntityScript` 还提供了两个复合查询（`scripts/entityscript.lua` 第 576-594 行）：

```lua
function EntityScript:HasTags(...)        -- 等价于 HasAllTags：必须同时具备所有标签
function EntityScript:HasOneOfTags(...)   -- 等价于 HasAnyTag：只要有其中任意一个
```

用法：

```lua
-- 同时是怪物 + 能被攻击
if inst:HasTags("monster", "_combat") then ... end

-- 蜘蛛或者蜘蛛战士
if inst:HasOneOfTags("spider", "spider_warrior") then ... end

-- 列表形式（tags 是一个 table 时）
local HOSTILE_TAGS = {"monster", "hostile", "epic"}
if inst:HasOneOfTags(HOSTILE_TAGS) then ... end
```

---

### 5.5.2 快速入门：为什么不用组件判断，要用 tag 判断？

你可能会想：判断"这是不是武器"，最直接的写法不应该是 `inst.components.weapon ~= nil` 吗？为什么官方偏要先 `inst:AddTag("weapon")` 再用 `inst:HasTag("weapon")` 去查？

**两个根本原因**：

#### 原因 1：客户端没有 `inst.components.xxx`

回顾 5.4 节的核心知识——`components` 只在主机上存在。当客户端想判断"地上这个东西能不能当武器用"时，它**根本访问不到 `inst.components.weapon`**。

但 tag 是随 pristine state 一起同步到客户端的——客户端一收到这个实体就能立刻查 `inst:HasTag("weapon")`。所以 `spear.lua` 第 44-45 行才会有那句注释：

```lua
--weapon (from weapon component) added to pristine state for optimization
inst:AddTag("weapon")
```

**翻译一下**：`weapon` 组件本来会在自身构造时给实体加 `"weapon"` 标签，但那得等客户端收到组件数据才能加（而 `weapon` 又不在 replica 白名单）。为了让客户端提前识别，官方在 pristine 阶段就手动贴上了这张标签——**一次标签同步代替一整个组件的同步**。

#### 原因 2：tag 是"最轻量的身份牌"

在饥荒的引擎里，每个实体的 tag 集合是一个**紧凑的哈希表**——单次查询几乎零开销。而 `inst.components.weapon` 是一个 Lua table，检查 `~= nil` 要走 Lua 的 `__index` 元表查找，成本高几十倍。

更重要的是 **`TheSim:FindEntities` 的过滤是在 C++ 里用 tag 集合做的——不需要进入 Lua**。这是一次关键的性能优化：

```lua
-- 在方圆 15 格内找所有"怪物 + 可被攻击 + 不是虚化态"的实体
local ents = TheSim:FindEntities(x, y, z, 15,
    { "monster", "_combat" },           -- must（必须都有）
    { "INLIMBO", "invisible" },         -- cant（不能有）
    { "hostile", "scarytoprey" })       -- mustoneof（至少有一个）
```

如果用组件判断，得先把范围内所有实体拉到 Lua，再一个个 `if inst.components.xxx ~= nil`——性能差距是数量级的。

> **新手记忆**：**"需要两端都识别的信息 = tag；只有主机算的数据 = component"**。比如"是不是武器"用 tag；"武器剩多少耐久"用 component（并通过 replica 同步到客户端）。

---

### 5.5.3 进阶：Tag 的 API 速查

| API | 返回 | 作用 |
|------|------|------|
| `inst:AddTag(tag)` | — | 加一张标签 |
| `inst:RemoveTag(tag)` | — | 删一张标签 |
| `inst:AddOrRemoveTag(tag, cond)` | — | 根据 `cond` 加或删 |
| `inst:HasTag(tag)` | `bool` | 有没有这张标签 |
| `inst:HasTags(t1, t2, ...)` | `bool` | 是否**同时**具备（= `HasAllTags`） |
| `inst:HasOneOfTags(t1, t2, ...)` | `bool` | 是否具备**其中之一**（= `HasAnyTag`） |

**搜索 API**（定义于 `scripts/simutil.lua` 和 C++ 引擎）：

| API | 作用 |
|------|------|
| `TheSim:FindEntities(x, y, z, radius, must, cant, oneof)` | **核心搜索**：在指定范围找符合 tag 条件的实体（基于 C++ 实现） |
| `FindEntity(inst, radius, fn, must, cant, oneof)` | 以 `inst` 为中心搜一个实体，附加 Lua 判定 `fn` |
| `FindClosestEntity(inst, radius, ignoreheight, must, cant, oneof, fn)` | 同上但返回最近的那个 |
| `GetClosestInstWithTag(tags, inst, radius)` | 简写版：只查一个 tag（或 tag 列表），不需要 cant/oneof |

`TheSim:FindEntities` 的三个 tag 参数是**组合关系**：

- `must`（必须）：实体必须**同时具备这些 tag 中的所有**。空 / `nil` 表示不要求。
- `cant`（不能）：实体**不能具备这些 tag 中的任何一个**。
- `oneof`（至少其一）：实体必须**具备这些 tag 中的至少一个**。

看 `scripts/prefabs/spider.lua` 第 102-108 行的实战例子：

```lua
local SPIDER_TAGS = { "spider" }
local SPIDER_IGNORE_TAGS = { "FX", "NOCLICK", "DECOR", "INLIMBO", "creaturecorpse" }

local function GetOtherSpiders(inst, radius, tags)
    tags = tags or SPIDER_TAGS
    local x, y, z = inst.Transform:GetWorldPosition()

    local spiders = TheSim:FindEntities(x, y, z, radius, nil, SPIDER_IGNORE_TAGS, tags)
    -- ...
end
```

这段代码的意思是：

- **must**：`nil`，没有强制要求
- **cant**：不能是 `"FX"`（特效）、`"NOCLICK"`（不可点击）、`"DECOR"`（装饰）、`"INLIMBO"`（在背包里）、`"creaturecorpse"`（尸体）——这些都是"假"的实体，过滤掉
- **oneof**：必须有 `"spider"` 标签之一（默认参数）

一行 `TheSim:FindEntities` 就完成了**"排除掉所有不是真实蜘蛛的垃圾实体"**这件事。如果用纯 Lua 遍历所有实体再过滤，开销会高两个数量级。

---

### 5.5.4 进阶：五类典型标签

看了几百个官方 prefab 的 tag 使用后，可以把**所有 tag 分成五类**——每类有不同的命名约定和作用。

#### ① 身份类别（小写字母名词）

这类 tag 描述"**这是什么**"，是最自然、最多的一类。它们都是小写英文名词，通常没有前缀。

| Tag | 含义 |
|------|------|
| `"player"` / `"character"` | 玩家 / 角色 |
| `"monster"` / `"hostile"` / `"epic"` | 怪物 / 敌对 / 巨怪 |
| `"spider"` / `"pig"` / `"rabbit"` | 具体物种 |
| `"animal"` / `"bird"` / `"insect"` | 大类 |
| `"structure"` / `"plant"` / `"tree"` | 建筑 / 植物 / 树 |
| `"fire"` / `"light"` / `"heat"` | 火 / 光源 / 热源 |
| `"weapon"` / `"tool"` / `"armor"` | 武器 / 工具 / 盔甲 |
| `"food"` / `"meat"` / `"veggie"` / `"raw"` | 食物类 |
| `"scarytoprey"` / `"canbetrapped"` | 行为类别 |

**使用场景**：

```lua
-- 战斗系统：只打怪物
if target:HasTag("monster") then ... end

-- 猪人：怕蜘蛛
if threat:HasTag("spider") then ... end

-- 猫头鹰：晚上对玩家恶意
if target:HasTag("player") and TheWorld.state.isnight then ... end
```

#### ② 下划线前缀标签（`_health`、`_combat`...）—— Replica 信号

这类以**单下划线**开头。它们是 **replica 系统的组件标记**——上一节详细讲过（见 5.4.4 节），每个可 replicated 的组件在实体上对应一个 `_组件名` 标签。

| Tag | 对应组件 | 含义 |
|------|---------|------|
| `"_health"` | `health` | 这个实体有血量 |
| `"_combat"` | `combat` | 能参与战斗 |
| `"_inventory"` | `inventory` | 能装东西 |
| `"_inventoryitem"` | `inventoryitem` | 是个可捡起的物品 |
| `"_equippable"` | `equippable` | 可装备 |
| `"_hunger"` / `"_sanity"` | `hunger` / `sanity` | 有饥饿/理智 |
| `"_builder"` | `builder` | 能制造 |
| `"_stackable"` | `stackable` | 可堆叠 |
| `"_container"` | `container` | 是容器 |

客户端常常用这些查组件是否"能用"：

```lua
-- 客户端判断 target 能不能战斗
if target:HasTag("_combat") and not target:HasTag("notarget") then
    -- 能攻击
end
```

**注意**：`"__健康"`（**双下划线**）是"曾经有但被移除"的标记——当你主动 `UnreplicateComponent("health")` 时，引擎会把 `_health` 换成 `__health`，以免客户端错误地认为实体从来没有过 health（见 `scripts/entityreplica.lua` 第 39-44 行）。

#### ③ 大写特殊状态（`INLIMBO`、`FX`、`NOCLICK`）—— 引擎级状态

这类全大写，通常是**引擎级的行为开关**——被引擎或核心系统特殊处理。

| Tag | 含义 | 触发时机 |
|------|------|---------|
| `"INLIMBO"` | 实体"不在场景中" | 进入背包/容器时自动加（`scripts/entityscript.lua` 第 349 行 `RemoveFromScene`） |
| `"CLASSIFIED"` | 只用于网络同步的辅助实体 | 如 `player_classified`、`inventoryitem_classified` |
| `"FX"` | 纯视觉特效 | 由开发者显式添加 |
| `"NOCLICK"` | 玩家鼠标点不到 | 特效、弹射物、隐藏物 |
| `"NOBLOCK"` | 不阻挡视线/选择 | 辅助识别 |
| `"DECOR"` | 纯装饰元素 | 地形装饰 |
| `"NOTARGET"` / `"notarget"` | 不被锁定为目标 | 战斗临时失效时 |
| `"invisible"` | 看不见 | 隐身效果 |
| `"playerghost"` | 玩家鬼魂形态 | 死亡后 |

**关键例子**——`scripts/entityscript.lua` 第 348-376 行的 `RemoveFromScene`：

```lua
function EntityScript:RemoveFromScene()
    self.entity:AddTag("INLIMBO")       -- 加 INLIMBO 标签
    self.entity:SetInLimbo(not self.forcedoutoflimbo)
    self.inlimbo = true
    self.entity:Hide()                  -- 不再渲染
    self:_DisableBrain_Internal()       -- 禁用 AI
    if self.sg then self.sg:Stop() end  -- 状态机停止
    -- ... 物理、光影、动画、小地图全部暂停 ...
    self:PushEvent("enterlimbo")
end
```

**一把被放进背包的长矛**：它进入 limbo 后，`"INLIMBO"` 标签一加，世界上所有 `TheSim:FindEntities` 的 `cant` 参数如果包含 `"INLIMBO"`，就会忽略它。所以蜘蛛不会去"攻击"玩家背包里的长矛——物品进入背包，"从世界上消失"。

**同理**，`NOCLICK` 让特效和弹射物不挡玩家鼠标；`FX` 让特效不被战斗系统扫描到；`CLASSIFIED` 防止网络辅助实体被误选中——这些都是引擎级的"开关"。

#### ④ 能力标签（`sharp`、`deployable`、`stackable`）—— "会什么"

这类描述实体"**会做什么**"——往往被"别的系统"用来判断交互资格。

| Tag | 含义 | 谁来查 |
|------|------|--------|
| `"sharp"` / `"pointy"` | 锋利、尖锐 | 未来可能的耐久互动、皮肤特效 |
| `"deployable"` | 可以被"放置"到地上 | `deployable` 组件的 placer |
| `"tile_deploy"` | 基于地块的放置 | 农田、地毯 |
| `"heavy"` | 重物（需双手扛） | `heavyobstaclephysics` |
| `"fridge"` / `"cooker"` / `"stewer"` | 冰箱 / 烹饪器 / 炖锅 | 放入食物时检查 |
| `"burnable"` / `"freezable"` | 能被烧 / 能被冻 | 火焰 / 冰冻手杖 |
| `"pickable"` / `"workable"` | 能被采摘 / 能被采集 | `pickable` / `workable` 组件 |
| `"flying"` | 飞行生物 | 攻击 / 视觉 |
| `"hasmagic"` | 有魔法属性 | 某些装备 / 仪式 |

这类 tag 是 **Mod 扩展最活跃的领域**——如果你的 Mod 要判断"这把武器是不是有毒"，你可以定义一个 `"poisonweapon"` 标签；然后在 `PrefabPostInit` 里给原版武器动态加上这个标签，再在你的"中毒"组件里 `HasTag("poisonweapon")` 判断。

#### ⑤ 状态标签（`burnt`、`frozen`、`isdead`）—— "当前处于什么状态"

这类描述**当前状态**，会随运行时动态变化。

| Tag | 含义 | 变化时机 |
|------|------|---------|
| `"burnt"` | 已烧成灰 | `burnable` 组件触发 |
| `"frozen"` | 被冰冻 | `freezable` 组件触发 |
| `"isdead"` | 死了 | `health` 组件（见 `health_replica.lua` 第 130-140 行） |
| `"hungry"` / `"crazy"` / `"hassleep"` | 状态机暂时状态 | StateGraph 添加 |
| `"wet"` / `"moist"` | 被雨淋湿 | 环境系统 |
| `"fire"` | 正在燃烧 | `burnable` |
| `"sleeping"` | 正在睡觉 | `sleeper` 组件 |
| `"attack"` | 正在攻击 | StateGraph |
| `"invisible"` | 当前隐身 | 隐身效果 |

**`health_replica.lua` 第 130-140 行的经典例子**：

```lua
function Health:SetIsDead(isdead)
    if isdead then
        self.inst:AddTag("isdead")
    else
        self.inst:RemoveTag("isdead")
    end
end

function Health:IsDead()
    return self.inst:HasTag("isdead")
end
```

**为什么连"是否死亡"都用 tag 表示？** 因为客户端也需要知道这个状态（画死亡 UI、停止互动提示等）——而 tag 天然会同步到客户端，无需单独的 net 变量。这是饥荒联机版**把状态用 tag 承载**的典型优化手法。

---

### 5.5.5 进阶：tag 在世界搜索中的核心作用

Tag 最强大的用处就是跟 `TheSim:FindEntities` 配合——**在"每帧扫整个世界"这种性能敏感的场景下尤其关键**。

#### 经典查找模式

**(1) "找到附近的怪物，排除假的"**（`scripts/prefabs/spider.lua` 第 208-223 行）：

```lua
local TARGET_MUST_TAGS = { "_combat", "character" }
local TARGET_CANT_TAGS = { "spiderwhisperer", "spiderdisguise", "INLIMBO" }

local function FindTarget(inst, radius)
    if not inst.no_targeting then
        return FindEntity(
            inst,
            SpringCombatMod(radius),
            function(guy)
                return (not inst.bedazzled and (guy.isplayer or not guy:HasTag("monster")))
                    and inst.components.combat:CanTarget(guy)
                    and not IsSpiderAlly(inst, guy)
            end,
            TARGET_MUST_TAGS,
            TARGET_CANT_TAGS
        )
    end
end
```

搜索条件：

- **must**：`_combat`（能战斗）+ `character`（是角色/生物）
- **cant**：`spiderwhisperer`（蛛语者，蜘蛛不打）、`spiderdisguise`（蛛蛹伪装）、`INLIMBO`（在背包里）
- **Lua 回调**：再加更精细的判断（bedazzled、是玩家但不是怪物、不是盟友）

注意 `TARGET_MUST_TAGS` 和 `TARGET_CANT_TAGS` 都是**模块级常量**——不在函数里临时构造 table，避免每次调用都 GC。

**(2) "找到特定 tag 的所有实体"**（`GetClosestInstWithTag`）：

```lua
local SPIDERDEN_TAGS = {"spiderden"}
local function SummonFriends(inst, attacker)
    local den = GetClosestInstWithTag(SPIDERDEN_TAGS, inst, 30)
    if den ~= nil and den.components.combat ~= nil then
        den.components.combat.onhitfn(den, attacker)
    end
end
```

一行找到最近的蜘蛛巢穴，让它呼唤援军。

#### 两阶段筛选：tag 粗筛 + Lua 精筛

一个非常重要的性能模式——**"先用 tag 在 C++ 里粗筛，再用 Lua 回调精筛"**。Lua 回调（上面示例里的 `function(guy) ... end`）**只会在 tag 粗筛通过的实体上执行**。所以：

- 先 tag 过滤："不要 FX、NOCLICK、INLIMBO、尸体"——一下砍掉 90% 的无关实体
- 再 Lua 过滤："必须是正在移动的、必须没被沙暴挡住、必须在视野内"——少量判断，不卡帧

这个模式贯穿全游戏：蜘蛛 AI、奶牛寻找玩家、风滚草搜集掉落物、食人花锁定目标……全都是这个套路。**Mod 作者自己写 AI 时也应该照搬**——**尽量把判断压到 tag 上，实在压不下去才走 Lua 回调**。

---

### 5.5.6 老手进阶：Tag 的设计原则

#### 命名约定速查

| 前缀/形式 | 代表 | 例子 |
|----------|------|------|
| 全小写英文名词 | 身份类别、能力 | `monster`、`weapon`、`sharp` |
| 单下划线开头 | Replica 组件标记 | `_health`、`_combat` |
| 双下划线开头 | 已移除的 replica 标记 | `__health` |
| 全大写 | 引擎级特殊状态 | `INLIMBO`、`FX`、`NOCLICK` |
| 小写名词 + 形容词 | 能力 / 可交互属性 | `pickable`、`deployable` |
| 描述当前状态 | 运行时状态 | `burnt`、`isdead`、`wet` |

#### 命名建议（Mod 开发者）

1. **给 Mod 的自定义 tag 加**一个**不容易撞车**的前缀——比如你 Mod 叫 `arcane_scrolls`，tag 就用 `arcane_weapon` 而不是直接 `weapon`。否则和原版 / 其他 Mod 冲突时极难排查。

2. **描述"是什么"用小写名词，描述"状态/能力"用小写形容词**——保持和官方一致的可读性。

3. **不要用下划线开头的 tag 名**（除非你**确实**在实现 replica）——避免被 replica 系统当作组件标记处理。

4. **不要发明大写的 tag 名**——大写 tag 都是引擎级语义，乱加可能触发未知行为。

#### 为什么不建议在运行时"批量添加"/"批量删除"tag？

Tag 的添加/删除会**触发网络同步**——每一次 `AddTag` 都可能产生一个"标签变化"的网络包（实际上引擎会做一些合并）。如果你每帧都 `AddTag / RemoveTag` 切换一个状态，就是在**白白占用网络带宽**。

**正确做法**：

- 能用 `inst.persistent_flag = true` 这种纯 Lua 字段搞定的，不要用 tag
- 只有**需要客户端也知道**或**需要在 `FindEntities` 里被搜到**的情况，才用 tag

#### Pristine 之前加 tag VS 之后加 tag

- **Pristine 之前加**：会被存入 pristine state，**随实体首次同步发给所有新客户端**——"开局就有"
- **Pristine 之后加**：会走"标签变更同步"通道——**稍后才到客户端**

对"**开局就需要**"的身份类别标签（`"monster"`、`"weapon"`、`"spider"`），**一律在 Pristine 之前加**。对"**运行时状态**"标签（`"isdead"`、`"burnt"`、`"frozen"`），在 Pristine 之后由业务逻辑动态加。

这就是为什么 `scripts/prefabs/player_common.lua` 第 2485-2492 行会把 `_health / _hunger / _sanity` 偷偷塞到 pristine state——玩家角色生成的**那一瞬间**，其他实体（比如猪人）如果要扫描附近的玩家，已经能查到 `_combat` 标签。如果走 replica 标签同步，会延迟几帧。

---

### 5.5.7 老手进阶：Mod 里的高级 tag 技巧

#### 技巧 1：用 `AddPrefabPostInit` 给原版 prefab 加标签

想让所有蜘蛛都被识别为"某种特殊群体"？

```lua
-- modmain.lua
AddPrefabPostInit("spider", function(inst)
    inst:AddTag("my_mod_marked_monster")
end)
AddPrefabPostInit("spider_warrior", function(inst)
    inst:AddTag("my_mod_marked_monster")
end)
```

然后你的新武器在击杀时只对带这个 tag 的生物产生效果：

```lua
local function onattacked(inst, data)
    if data.attacker and data.attacker:HasTag("my_mod_marked_monster") then
        -- 特殊效果
    end
end
```

**关键点**：因为这个 tag 是在 `AddPrefabPostInit` 里加的——**它必然跑在主机上**，且**也会跑在客户端上**（因为 `AddPrefabPostInit` 对客户端的 Prefab 也生效）。所以这张 tag 客户端也能识别。但**具体它是走 pristine 还是 replica 同步**取决于你加的时机：

- 如果 `AddPrefabPostInit` 回调在 `SetPristine` 之前运行——跟着 pristine 走
- 如果在之后——走 replica

`AddPrefabPostInit` 的时机在 5.1.5 节讨论过：**它在工厂函数返回之后**被调用——这意味着 `SetPristine` 已经触发了。所以按原理，这里加的 tag 会走 replica 同步。

不过实际上饥荒引擎做了优化——在 Pristine state 还未被最终固化前，`AddPrefabPostInit` 的修改可能也会被合并进去（具体行为和版本有关）。**保险的做法**是：如果你确定这个 tag 是"原生身份类"，应该用 `AddPrefabPostInitAny` 在更早的时机加。

#### 技巧 2：用 tag 代替自己写 replica

前面说过"自定义组件要客户端也能访问，需要写 replica"——但**如果你只是想让客户端知道"有还是没有"这种二元状态**，直接用 tag 就够了：

```lua
-- 服务端业务逻辑
function MyComponent:SetPoisoned(poisoned)
    self.poisoned = poisoned
    self.inst:AddOrRemoveTag("mymod_poisoned", poisoned)  -- ← tag 自动同步到客户端
end

-- 客户端查询
if inst:HasTag("mymod_poisoned") then
    -- 显示特效
end
```

**一行代码解决网络同步**，不用写 net 变量、不用写 replica 类、不用搞事件监听。适用于"非此即彼"的状态（有毒/无毒、点燃/未点燃、被拘束/未被拘束）。

#### 技巧 3：tag 是 FSM 的辅助条件

很多 prefab 的状态机（StateGraph）会根据 tag 决定转移——比如生物"看到有 `fire` 标签的实体就逃跑"。这和传统 FSM 的"带条件转移"完全一致，只是条件是 tag。

#### 技巧 4：tag 作为"懒加载的组件缓存"

官方玩法：**一些组件在加载时会给实体加一个"能力 tag"，这样其他系统不需要去访问 `inst.components.xxx`（也可能不在客户端）就能判断**。比如 `eater` 组件会给食物实体加 `"eater"` 之类的标签。

Mod 里可以照做：你的 `MyCustomComponent` 构造时加一个 `"mycomponent_ready"` 标签，其他 Mod 想知道是否安装，不用 `inst.components.mycomponent ~= nil`，直接 `HasTag` 即可。

#### 技巧 5：tag 永远先于组件加载完成

**Pristine state 里的 tag 在客户端上生效时机比 replica 组件要早**——这就是为什么在 `OnEntityReplicated` 里用 tag 做初始化决策比用 replica 对象更可靠：

```lua
function inst.OnEntityReplicated(inst)
    -- replica 此时已经就位
    if inst:HasTag("_combat") then
        -- 准备战斗相关 UI
    end
    -- 但如果用 inst.replica.combat 直接访问...
    -- 有些情况 classified 连接可能还没建立，.replica.combat 可能是 nil
end
```

---

### 5.5.8 小结

- **Tag = 贴在实体上的字符串标签纸**。三个基础 API：`AddTag / HasTag / RemoveTag`，加复合的 `HasTags`（所有）和 `HasOneOfTags`（任一）。底层是 C++ 哈希集合，单次查询几乎零开销。
- **为什么 tag 比组件判断更常用**：(1) 客户端没有 `inst.components.xxx`，只有 tag；(2) 每个 tag 极其轻量，查询超快；(3) `TheSim:FindEntities` 原生支持三个 tag 参数筛选，在 C++ 层一次过滤，性能甩 Lua 几十条街。
- **五类典型标签**：①身份类别（小写名词：monster、spider、weapon）②下划线组件标记（`_health / _combat`）③大写引擎级状态（`INLIMBO / FX / NOCLICK / CLASSIFIED`）④能力标签（sharp、deployable、fridge）⑤运行时状态（isdead、burnt、frozen）。
- **`TheSim:FindEntities` 的黄金模式**：tag 粗筛（C++ 里快速过滤）+ Lua 回调精筛（细节判断）。`must + cant + oneof` 三参数组合+ 常量 table 复用 = 官方标准用法。
- **Tag 设计原则**：Mod 自定义 tag 加前缀避免撞车；小写名词表身份、大写保留给引擎、下划线保留给 replica；运行时尽量少频繁切换 tag，因为会产生网络同步。
- **Pristine 前加 vs. 后加**：身份类 / 预知类标签放 pristine 前（随实体首次到达），运行时状态标签放 pristine 后（动态变化）。`player_common.lua` 里 "Sneak these into pristine state" 就是这种优化。
- **Mod 里的五个高级技巧**：(1) `AddPrefabPostInit` 动态加 tag 给原版 prefab；(2) 用 tag 代替简单的 replica 同步；(3) tag 作为 FSM 转移条件；(4) 组件加载时加能力 tag 供其他系统探测；(5) 在 `OnEntityReplicated` 里用 tag 做决策比直接访问 replica 对象更稳。

> **下一节预告**：5.6 节我们会把视角转到资源系统——`Asset` 声明到底是什么？`anim/log.zip` 这种文件怎么被加载？`RegisterPrefabsImpl` 内部的 `resolve_fn` 在做什么？Mod 怎么正确地声明自己的资源，避免"找不到素材"或"资源冲突"？ 我们会用官方的 `log.lua` 配合 `mods.lua` 里的 Mod 资源挂载流程走一遍。

## 5.6 Asset 声明与资源加载机制（RegisterPrefabs 流程）

### 本节导读

前面五节我们反复见到 Prefab 文件顶部的 `local assets = { Asset("ANIM", "anim/log.zip"), ... }`——你知道它"告诉引擎要加载什么美术资源"，但具体：

- `Asset` 对象到底存了什么？
- `"anim/log.zip"` 这个相对路径是怎么被引擎找到真实文件的？
- 游戏启动时"加载 Prefab"和"加载资源"是同一件事吗？
- 专用服务器为什么不下载音效？
- Mod 的 `modmain.lua` 里的 `Assets = { ... }` 和 Prefab 文件里的 `assets` 什么关系？

这一节把这些问题一次讲透。

> **新手**看 5.6.1-5.6.3 就够：认识 `Asset` 每种类型的用途、Prefab 文件里怎么写；**进阶读者**要理解从 `LoadPrefabFile` → `RegisterPrefabsImpl` → `resolvefilepath` 的完整注册流程，以及 `package.assetpath` 搜索路径的作用；**老手**可以重点看 5.6.7 和 5.6.8：Mod 的 `PrefabFiles / Assets` 是怎么挂到 `"MOD_xxx"` 虚拟 Prefab 上的，以及避免"找不到资源"、"资源冲突"、"专用服务器报错"的实战经验。

---

### 5.6.1 快速入门：Asset 是什么？

**一句话：`Asset` 是一个描述"美术资源依赖"的小对象。**

回顾 5.1 节，`Asset` 类的定义只有 4 行（`scripts/prefabs.lua` 第 25-29 行）：

```lua
Asset = Class( function(self, type, file, param)
    self.type = type
    self.file = file
    self.param = param
end)
```

它只存三个字段：

| 字段 | 含义 | 例子 |
|------|------|------|
| `type` | 资源类型（大写字符串） | `"ANIM"`、`"IMAGE"`、`"ATLAS"`、`"SOUND"` |
| `file` | 资源文件的相对路径 | `"anim/log.zip"`、`"images/wilson.tex"` |
| `param` | 某些类型需要的额外参数（多数不用） | 可选，默认 `nil` |

Prefab 文件顶部的 `local assets = { ... }` 就是一个 `Asset` 数组。举几个真实例子：

```lua
-- scripts/prefabs/log.lua（木头）
local assets =
{
    Asset("ANIM", "anim/log.zip"),           -- 一个动画文件
}

-- scripts/prefabs/spear.lua（长矛）
local assets =
{
    Asset("ANIM", "anim/spear.zip"),          -- 地上形态的动画
    Asset("ANIM", "anim/swap_spear.zip"),     -- 手握形态的动画
}
```

玩家角色的 assets 能到几十上百条（`scripts/prefabs/player_common.lua` 第 1906-2088 行有 180 多个 `Asset`），因为玩家动画非常复杂——走、跑、攻击、砍树、挖矿、炼药、吃饭…… 每种行为一个 `.zip` 文件。

**assets 和工厂函数的关系**：

- `assets` 是**告诉引擎"请把这些文件加载到内存里，我等下要用"**
- 工厂函数里写 `inst.AnimState:SetBank("log")` / `SetBuild("log")` 时用的"log"就是从 `anim/log.zip` 里的构建项（build）名

没在 assets 里声明过的资源，`SetBuild("xxx")` 会直接失败（客户端表现：白色方块 / 不可见）。

---

### 5.6.2 快速入门：常见的 Asset 类型

按使用频率从高到低：

| 类型 | 文件后缀 | 含义 | 典型用途 |
|------|---------|------|---------|
| `"ANIM"` | `.zip` | 动画资源（Spriter 导出的动画 + 贴图的打包文件） | 所有实体的模型、动画 |
| `"IMAGE"` | `.tex` | 单张贴图 | UI 图标、小地图图标 |
| `"ATLAS"` | `.xml` | **图集**（描述一张大贴图里各个小图的位置） | UI 按钮、物品栏 slot |
| `"SOUND"` | `.fsb` | 音效包（FMOD Sound Bank） | 怪物叫声、技能音效 |
| `"SOUNDPACKAGE"` | `.fev` | FMOD 声音工程描述 | 声音包的入口元数据 |
| `"SHADER"` | `.ksh` | Klei 自定义着色器 | 阴影、发光、水面效果 |
| `"PKGREF"` | 同多种 | "**引用而不加载**"——告诉引擎"这个文件存在，可能会被远端/皮肤系统用到" | 皮肤动画、事件主题 |
| `"SCRIPT"` | `.lua` | 声明依赖的 Lua 文件 | 共享代码（如 `prefabs/wx78_common.lua`） |
| `"INV_IMAGE"` | 名字（不带路径） | 背包里的物品图标 | 物品展示 |
| `"MINIMAP_IMAGE"` | 名字（不带路径） | 小地图上的图标 | 建筑、特殊地点 |

**几点常见疑惑**：

#### Q: `ATLAS` 和 `IMAGE` 为什么常常成对出现？

很多 UI 贴图是用"大图集"格式导出的——一张 `images/buttons.tex`（大贴图）里打包了几十个小按钮，每个按钮在大图中的位置由 `images/buttons.xml` 描述。所以你会在 Mod 里看到：

```lua
Asset("ATLAS", "images/medal_buff_ui.xml"),
Asset("IMAGE", "images/medal_buff_ui.tex"),
```

**两者缺一**，UI 就会显示错乱。

#### Q: `PKGREF` 和 `ANIM` 有什么区别？

`ANIM` 是"**加载进内存、立刻可用**"；`PKGREF` 是"**只做引用登记，真正加载交给别的系统**"。典型场景是**皮肤系统**——皮肤文件 `.zip` 特别多、每个玩家只会用到其中几个，全部预加载就是浪费。所以皮肤用 `PKGREF` 登记依赖，只有玩家真正选择某个皮肤时才动态加载。

看 `scripts/prefabs/event_deps.lua` 第 74-103 行，整页都是 `PKGREF`——这些是**节日特供菜单动画**，每年只用几次，完全没必要加进内存。

#### Q: `SCRIPT` 类型是什么用？

`SCRIPT` 声明 Lua 源代码依赖。看 `scripts/prefabs/wx78_backupbody.lua` 第 5 行：

```lua
Asset("SCRIPT", "scripts/prefabs/wx78_common.lua"),
```

这告诉打包系统"这个 Prefab 依赖 `wx78_common.lua`"——联机版客户端首次连入服务器时，如果本地没这个 Lua 脚本，会自动下载。对 Mod 来说这在 `modmain.lua` 定义的 `assets` 列表里是很重要的——保证每个客户端都能拿到必须的代码文件。

---

### 5.6.3 进阶：Asset 数据如何挂到 Prefab 上

回顾 5.1 节的 `Prefab` 类（`scripts/prefabs.lua` 第 6-19 行）：

```lua
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

Prefab 的构造函数只是**把 assets 数组原样存到 `self.assets`**。注意最后一块关于 `PREFAB_SKINS` 的代码——如果这个 Prefab 有皮肤，皮肤 Prefab 名会被**自动塞进 `self.deps`**。

**`assets` 和 `deps` 的区别**：

- `assets`：**这个 Prefab 要用的具体资源文件**（`.zip / .tex / .xml`）
- `deps`：**这个 Prefab 依赖的其他 Prefab 的名字**（字符串）

比如 `spider.lua` 第 69-84 行：

```lua
local prefabs =
{
    "spidergland",
    "monstermeat",
    "silk",
    "spider_web_spit",
    "spider_web_spit_acidinfused",
    "moonspider_spike",
    -- ...
}
```

这是蜘蛛的 `deps`——蜘蛛打死会掉"蜘蛛腺"、"怪物肉"、"蜘蛛丝"，生成"蜘蛛网弹射物"。这些都是**其他 Prefab 的名字**。引擎在加载 `spider` Prefab 时，会自动把这些 Prefab 也标记为"需要加载"，避免运行时 `SpawnPrefab("spidergland")` 出现"资源没加载"。

文件末尾传给 `Prefab(...)`：

```lua
return Prefab("spider", create_spider, assets, prefabs)
```

`prefabs`（deps）是第 4 个参数。

---

### 5.6.4 进阶：资源加载的完整流程

游戏启动时，每个 Prefab 文件经过一条"注册 → 解析 → 加载"的流水线。我们把它拆开看。

#### 第一步：`LoadPrefabFile` 读取 Prefab 文件

回顾 5.1.6 节——`scripts/mainfunctions.lua` 第 145-180 行：

```lua
function LoadPrefabFile( filename, async_batch_validation, search_asset_first_path )
    local fn, r = loadfile(filename)
    assert(fn, "Could not load file ".. filename)
    -- ...
    local ret = {fn()}         -- 执行文件，拿到 return 的 Prefab 对象们

    if ret then
        for i,val in ipairs(ret) do
            if type(val)=="table" and val.is_a and val:is_a(Prefab) then
                val.search_asset_first_path = search_asset_first_path
                if async_batch_validation then
                    RegisterPrefabsImpl(val, VerifyPrefabAssetExistsAsync)
                else
                    RegisterSinglePrefab(val)
                end
                PREFABDEFINITIONS[val.name] = val
            end
        end
    end
    return ret
end
```

关键动作：

1. `loadfile` + `fn()`：执行 Prefab 文件，拿到所有 `Prefab` 对象
2. 给每个 Prefab 打上 `search_asset_first_path`（**Mod 的 Prefab 会有这个字段——后面详细讲**）
3. 调用 `RegisterSinglePrefab` 注册

#### 第二步：`RegisterSinglePrefab` / `RegisterPrefabsImpl`

`scripts/mainfunctions.lua` 第 103-117 行：

```lua
function RegisterPrefabsImpl(prefab, resolve_fn)
    for i,asset in ipairs(prefab.assets) do
        if not ShouldIgnoreResolve(asset.file, asset.type) then
            resolve_fn(prefab, asset)
        end
    end

    modprefabinitfns[prefab.name] = ModManager:GetPostInitFns("PrefabPostInit", prefab.name)
    Prefabs[prefab.name] = prefab

    TheSim:RegisterPrefab(prefab.name, prefab.assets, prefab.deps)
end
```

四件事：

1. **遍历 assets**，对每个资源调用 `resolve_fn`——**这是把相对路径解析成绝对路径的地方**
2. **预取 Mod PostInit**：把所有 Mod 注册的 `PrefabPostInit` 函数先收集进 `modprefabinitfns`——这样每次 `SpawnPrefab` 时可以直接索引，不用再查（回顾 5.1.5 节的 Spawn 流程）
3. **登记到全局 `Prefabs` 表**（`Prefabs[prefab.name] = prefab`）
4. **通知 C++ 引擎**：`TheSim:RegisterPrefab(...)` 告诉底层"**从现在起，这个 name 对应的资源清单是这些**"

#### 第三步：`resolve_fn` —— 路径解析

实际的 `resolve_fn` 是 `RegisterPrefabsResolveAssets`（同文件第 119-125 行）：

```lua
local function RegisterPrefabsResolveAssets(prefab, asset)
    local resolvedpath = resolvefilepath(asset.file, prefab.force_path_search, prefab.search_asset_first_path)
    assert(resolvedpath, "Could not find "..asset.file.." required by "..prefab.name)
    TheSim:OnAssetPathResolve(asset.file, resolvedpath)
    asset.file = resolvedpath  -- ← 把解析后的绝对路径写回 asset
end
```

关键：

- `resolvefilepath` 把相对路径（`"anim/log.zip"`）变成绝对路径（如 `"C:/.../data/anim/log.zip"`）
- 找不到会直接 `assert` 崩溃——**这就是游戏启动时最常见的崩溃："Could not find anim/xxx.zip required by yyy"**
- 解析后的绝对路径**原地写回 `asset.file`**，以后不再需要解析

#### 第四步：`TheSim:RegisterPrefab` —— 通知 C++ 引擎

这是 C++ 接口（Lua 层看不到实现）。它做的事大致是：

- 把 Prefab 名字、资源清单（现在都是绝对路径）、依赖列表都记录到引擎侧的 Prefab 注册表
- 此时**资源还没真正加载进内存**——只是"登记了"
- 真正的加载由 `TheSim:LoadPrefabs({"log", "spear", ...})` 触发（比如世界初始化时）

#### 整体流程图

```
    scripts/prefabs/log.lua
            ↓
    LoadPrefabFile("scripts/prefabs/log.lua")
            ↓
    loadfile → fn() → return Prefab("log", fn, assets, deps)
            ↓
    RegisterSinglePrefab(prefab)
            ↓
    RegisterPrefabsImpl(prefab, RegisterPrefabsResolveAssets)
      ├─ for each asset：
      │   ├─ ShouldIgnoreResolve？（专用服务器跳过音效等）
      │   └─ resolvefilepath → 绝对路径
      │       ├─ 找不到 → assert 崩溃
      │       └─ 写回 asset.file
      ├─ Prefabs["log"] = prefab
      └─ TheSim:RegisterPrefab("log", assets, deps)
            ↓
    游戏运行中，需要时：
    TheSim:LoadPrefabs({"log"}) → C++ 真正把资源加载进内存
            ↓
    SpawnPrefab("log") → 实体被造出来，引用已加载的资源
```

---

### 5.6.5 进阶：资源路径解析机制

`resolvefilepath` 是**把相对路径 → 绝对路径**的核心函数（`scripts/util.lua` 第 636-643 行）：

```lua
function resolvefilepath(filepath, force_path_search, search_first_path)
    if memoizedFilePaths[filepath] then
        return memoizedFilePaths[filepath]
    end
    local resolved = resolvefilepath_internal(filepath, force_path_search, search_first_path)
    assert(resolved ~= nil, "Could not find an asset matching "..filepath.." in any of the search paths.")
    return resolved
end
```

**两个关键优化 + 一个失败机制**：

1. **`memoizedFilePaths` 缓存**：每个解析结果都被缓存，第二次同样请求直接返回（无论解析成功还是失败）
2. **失败即崩溃**（`assert`）：找不到直接游戏 crash，**避免运行时出现"谜之白方块"**
3. 对应的"软性"版本 `resolvefilepath_soft`（第 629-634 行）找不到时返回 `nil`——适用于"这个资源可能不存在，失败也没关系"的场景

#### 内部的 `softresolvefilepath_internal`

`scripts/util.lua` 第 585-620 行：

```lua
local function softresolvefilepath_internal(filepath, force_path_search, search_first_path)
    force_path_search = force_path_search or false

    if IsConsole() and not force_path_search then
        return filepath -- it's already absolute, so just send it back
    end

    --on PC platforms, search all the possible paths

    --mod folders don't have "data" in them, so we strip that off if necessary. It will
    --be added back on as one of the search paths.
    filepath = string.gsub(filepath, "^/", "")

    --sometimes from context we can know the most likely path for an asset, this can result in less time spent searching the tons of mod search paths.
    if search_first_path then
        local filename = search_first_path..filepath
        if kleifileexists(filename) then
            return filename
        end
    end

    local searchpaths = package.assetpath
    for i, pathdata in ipairs_reverse(searchpaths) do
        local filename = string.gsub(pathdata.path..filepath, "\\", "/")
        if kleifileexists(filename, pathdata.manifest, filepath) then
            return filename
        end
    end

    --as a last resort see if the file is an already correct path (incase this asset has already been processed)
    if kleifileexists(filepath) then
        return filepath
    end

    return nil
end
```

**搜索逻辑**（自上而下）：

1. **主机平台直接用绝对路径**（针对 PS4 / Xbox 等平台）
2. **优先搜索 `search_first_path`**——这是 Mod 的根目录路径。对 Mod 的资源来说，**直接检查 `MODS_ROOT + modname + /anim/xxx.zip` 是最快的**，能命中 99% 的情况
3. **按 `package.assetpath` 列表反向搜索**——`assetpath` 是引擎维护的"全局搜索路径"列表，包含主程序 data 目录、所有已启用 Mod 的根目录等
4. **最后再尝试原路径作为绝对路径**（兜底，应对已解析过的情况）

**`package.assetpath` 是什么？**
它是 C++ 引擎在启动时按优先级从高到低填入的搜索路径列表：

- Mod1 的根目录
- Mod2 的根目录
- ...
- 游戏主程序的 `data/` 目录

`ipairs_reverse` 从后往前遍历意味着**主程序路径优先命中**——如果一个 Mod 和主程序都叫 `anim/log.zip`，主程序版本会被优先使用。（有的 Mod 会利用这个规则做**替换式覆盖**，例如把自己的 `anim/log.zip` 放在 `data/` 里让原版资源被代替……但这是不推荐的侵入性做法。）

---

### 5.6.6 进阶：专用服务器的资源忽略

有个细节在 `RegisterPrefabsImpl` 里一闪而过——`ShouldIgnoreResolve(asset.file, asset.type)`。看 `scripts/mainfunctions.lua` 第 69-98 行：

```lua
function ShouldIgnoreResolve( filename, assettype )
    if assettype == "INV_IMAGE" then
        return true
    end
    if assettype == "MINIMAP_IMAGE" then
        return true
    end
    if filename:find(".dyn") and assettype == "PKGREF" then
        return true
    end

    if TheNet:IsDedicated() then
        if assettype == "SOUNDPACKAGE" then
            return true
        end
        if assettype == "SOUND" then
            return true
        end
        if filename:find(".ogv") then
            return true
        end
        if filename:find(".fev") and assettype == "PKGREF" then
            return true
        end
        if filename:find("fsb") then
            return true
        end
    end
    return false
end
```

**两组规则**：

#### 规则 1：所有平台都跳过的类型

- **`INV_IMAGE`、`MINIMAP_IMAGE`**：这两种类型的"路径"其实不是文件路径，而是 UI 动画内部的**贴图名**——不需要真实路径解析
- **`.dyn` 的 `PKGREF`**：动态皮肤文件，运行时按需加载

#### 规则 2：专用服务器跳过的类型

`TheNet:IsDedicated()` 为真时（专用服务器），**一律跳过**：

- `SOUNDPACKAGE` / `SOUND` / `.fsb` / `.fev`（`PKGREF`）：服务器不需要音效
- `.ogv`：视频文件（片头过场动画），服务器不需要

这是**极其重要的优化**——专用服务器通常跑在 Linux VPS 上，没声卡、没显卡，下载音效视频就是纯浪费。这段代码让专用服务器启动速度快了一大截。

> **Mod 作者启示**：你的 Mod 的 `Assets` 列表在专用服务器上会被过滤——如果你的 Mod 强依赖某些音效/视频（不太可能，但假设），要意识到专服拿不到。这也是为什么 Mod 纯**客户端 UI 增强**（如中文字体、键位提示）常会标 `client_only_mod = true`，让专服根本不参与这个 Mod。

---

### 5.6.7 老手进阶：Mod 的 `PrefabFiles` 与 `Assets`

现在我们转到 Mod 开发者真正关心的部分：**Mod 里声明的资源到底是怎么挂到游戏里的？**

在 `modmain.lua` 里你会定义两个全局表：

```lua
-- 要加载的 Prefab 文件名列表（不带 prefabs/ 前缀）
PrefabFiles = {
    "scroll_lightning",
    "scroll_heal",
    "scroll_summon",
}

-- Mod 共用的资源列表（不属于某个具体 Prefab 的资源）
Assets = {
    Asset("ATLAS", "images/my_ui.xml"),
    Asset("IMAGE", "images/my_ui.tex"),
    Asset("ANIM", "anim/my_common_fx.zip"),
    Asset("SOUND", "sound/my_sounds.fsb"),
}
```

它们各自会被怎样处理？

#### `PrefabFiles` 的加载

看 `scripts/mods.lua` 第 676-703 行的 `ModWrangler:RegisterPrefabs`：

```lua
function ModWrangler:RegisterPrefabs()
    if not MODS_ENABLED then return end

    for i,modname in ipairs(self.enabledmods) do
        local mod = self:GetMod(modname)

        mod.LoadPrefabFile = LoadPrefabFile
        mod.RegisterPrefabs = RegisterPrefabs
        mod.Prefabs = {}

        print("Mod: "..ModInfoname(modname), "Registering prefabs")

        if mod.PrefabFiles then
            for _, prefab_path in ipairs(mod.PrefabFiles) do
                print("Mod: "..ModInfoname(modname), "  Registering prefab file: prefabs/"..prefab_path)
                local ret = runmodfn( mod.LoadPrefabFile, mod, "LoadPrefabFile" )(
                    "prefabs/"..prefab_path,
                    nil,
                    MODS_ROOT..modname.."/"  -- ← 传入 search_first_path
                )
                if ret then
                    for _, prefab in ipairs(ret) do
                        print("Mod: "..ModInfoname(modname), "    "..prefab.name)
                        mod.Prefabs[prefab.name] = prefab
                    end
                end
            end
        end

        local prefabnames = {}
        for name, prefab in pairs(mod.Prefabs) do
            table.insert(prefabnames, name)
            Prefabs[name] = prefab -- copy the prefabs back into the main environment
        end
```

流程：

1. 遍历 `mod.PrefabFiles` 里的每个名字 `name`
2. 调用 `LoadPrefabFile("prefabs/" .. name, nil, MODS_ROOT..modname.."/")`——**关键点：第三个参数 `search_first_path` 传入了 Mod 的根目录**
3. 每个 Prefab 被注册后，把它从 Mod 的沙箱环境复制回全局 `Prefabs` 表

**第三个参数的意义**——回顾 5.6.5 节的 `softresolvefilepath_internal`：当解析资源路径时，**先用 `search_first_path + filepath` 组合尝试**。对 Mod 来说就是"**先在本 Mod 目录里找**"——因此你的 Mod 的 `anim/my_scroll.zip` 真实路径是 `MODS_ROOT + modname + /anim/my_scroll.zip`，这条规则让它第一时间命中。

#### `Assets` 的加载

同一个函数后半段（`scripts/mods.lua` 第 711-719 行）：

```lua
print("Mod: "..ModInfoname(modname), "  Registering default mod prefab")

local pref = Prefab("MOD_"..modname, nil, mod.Assets, prefabnames, true)
pref.search_asset_first_path = MODS_ROOT..modname.."/"
RegisterSinglePrefab(pref)

TheSim:LoadPrefabs({pref.name})
table.insert(self.loadedprefabs, pref.name)
```

**这是最巧妙的设计**：

- 创建一个**虚拟 Prefab**叫 `"MOD_<modname>"`（比如 `"MOD_medal"`）
- 它的**工厂函数是 `nil`**（永远不会被 `SpawnPrefab`）
- 它的 **`assets` 字段填入 `mod.Assets`**——你在 `modmain.lua` 里写的 `Assets` 全表
- 它的 **`deps` 字段填入 Mod 的所有 Prefab 名字**——相当于说"只要这个 MOD_xxx 加载了，它的所有子 Prefab 也要加载"
- `force_path_search = true`（第 5 个参数）——强制搜索，即使主机平台也要走完整路径解析流程
- `search_asset_first_path = MODS_ROOT..modname.."/"`——资源优先在 Mod 目录找
- 立刻 `TheSim:LoadPrefabs({"MOD_medal"})`——**游戏启动时立刻加载这个虚拟 Prefab，它会触发所有依赖 Prefab 和所有全局 Assets 的加载**

**效果**：`modmain.lua` 里 `Assets` 声明的资源**在 Mod 启动时一次性全部加载**，不随某个具体 Prefab。这样多个 Prefab 共用的资源（UI 图集、通用特效）不需要在每个 Prefab 文件里重复声明。

---

### 5.6.8 老手进阶：Mod 资源声明的常见坑

在 Mod 开发中和资源系统打交道是非常容易踩坑的场景。下面是八个最常见的问题和解决方案。

#### 坑 1：路径写错大小写

Mod 的资源路径在 Windows 上大小写不敏感，但 Linux（专用服务器的常见环境）上**大小写敏感**。常见问题：

```lua
Asset("ANIM", "anim/MyScroll.zip"),   -- ← Windows 测试没问题
-- 真实文件名可能是 anim/myscroll.zip，Linux 专服会崩
```

**一律用小写路径**。文件名也尽量都小写。

#### 坑 2：`ATLAS` 和 `IMAGE` 只声明了一个

```lua
Asset("ATLAS", "images/my_ui.xml"),   -- 只声明了 xml
-- 忘记声明 images/my_ui.tex
```

结果：游戏启动不报错，但 UI 显示空白。**必须成对声明**。

#### 坑 3：在具体 Prefab 的 `assets` 里放了全局 UI 资源

```lua
-- prefabs/my_scroll.lua
local assets =
{
    Asset("ANIM", "anim/my_scroll.zip"),
    Asset("ATLAS", "images/my_ui.xml"),   -- ← 不推荐：UI 资源放这
    Asset("IMAGE", "images/my_ui.tex"),
}
```

问题：UI 资源加载时机变得不稳定，且如果同 Mod 有多个 Prefab 都用这个 UI，会**重复加载警告**。

**正确做法**：全局 UI 资源一律放到 `modmain.lua` 的 `Assets = { ... }` 里。具体 Prefab 的 `assets` 只放那个 Prefab 独有的资源（自己的 `anim/xxx.zip`、自己的音效）。

#### 坑 4：Mod 里的资源路径带前缀 `mods/xxx/`

```lua
Asset("ANIM", "mods/my_scroll_mod/anim/my_scroll.zip"),   -- ← 错
```

Mod 里的路径都**相对于 Mod 根目录**（因为 `search_first_path = MODS_ROOT..modname.."/"`）。正确写法：

```lua
Asset("ANIM", "anim/my_scroll.zip"),   -- ← 对
```

#### 坑 5：忘了在 `PrefabFiles` 里登记新 Prefab

你写了 `mods/my_mod/scripts/prefabs/my_scroll.lua`，但 `modmain.lua` 里的 `PrefabFiles` 忘了加：

```lua
PrefabFiles = {
    "other_scroll",
    -- "my_scroll",   ← 忘了加
}
```

结果：游戏启动不报错，但 `SpawnPrefab("my_scroll")` 时报 "Can't find prefab my_scroll"。**所有新 Prefab 必须在 `PrefabFiles` 里登记**。

#### 坑 6：打算让所有客户端也用的 Mod 没设 `all_clients_require_mod`

`modinfo.lua` 里有几个重要开关：

```lua
-- modinfo.lua
client_only_mod = false            -- 是否仅客户端使用（不影响游戏逻辑）
all_clients_require_mod = true     -- 所有客户端是否必须装这个 Mod
dst_compatible = true              -- 是否兼容联机版
```

- **`client_only_mod = true`**：像"中文字体增强"、"UI 改动"、"屏幕上显示 FPS"这种**不改游戏逻辑**的 Mod——主机和客户端独立启用，客户端的 Mod 不会上传到服务器
- **`all_clients_require_mod = true`**：主机启用这个 Mod 时，客户端**必须也装同版本**才能连进来——用于**改了游戏逻辑/添加了新 Prefab**的 Mod。联机时主机会自动推送 Lua 脚本到客户端，但 Mod 的 Assets（美术资源）**客户端必须自己本地有**
- **`dst_compatible = true`**：标记兼容联机版（单机版和联机版数据结构不同）

**常见问题**：你做了一个"加 5 种魔法卷轴"的 Mod，`all_clients_require_mod` 忘了设——其他玩家进服之后能看到长矛，但看不到卷轴（因为卷轴是新 Prefab，客户端本地没有资源）。

#### 坑 7：Mod 的贴图/动画更新了，但客户端没重启

游戏客户端对 `memoizedFilePaths`（第一次解析成功的路径）会缓存。如果你在游戏运行中改了 Mod 的资源文件，**客户端可能还在用内存里的旧版本**。通用做法是——**客户端 + 主机都重启游戏**，别指望"reload Mod"能彻底刷新。

#### 坑 8：Mod 的 `Assets` 列表过大影响启动

每个 `Asset` 都要走 `resolvefilepath` 一次。如果你的 Mod 有几百个资源，启动会显著变慢。对策：

- 只声明**真正会用到**的资源
- 对**偶尔用到**的资源用 `PKGREF`——延迟加载
- 对**皮肤系统**用 `PREFAB_SKINS` 机制（回顾 5.6.3 节的 `Prefab` 构造函数最后那段——皮肤会被自动注入到 `deps`）

---

### 5.6.9 小结

- **`Asset` 是一个三字段小对象**（type / file / param），声明在 Prefab 顶部或 `modmain.lua` 的 `Assets` 里。它告诉引擎"我用到这些资源文件"。
- **常见 Asset 类型**：`ANIM`（动画）、`IMAGE`（贴图）、`ATLAS`（图集，配 IMAGE）、`SOUND`（音效）、`SHADER`（着色器）、`PKGREF`（仅引用、延迟加载）、`SCRIPT`（Lua 脚本依赖）、`INV_IMAGE` / `MINIMAP_IMAGE`（UI 用的贴图名）。
- **`Prefab` 的 `assets` 字段**存资源列表，**`deps` 字段**存依赖的其他 Prefab 名字。皮肤 Prefab 会被自动追加到 `deps`。
- **资源注册流程**：`LoadPrefabFile` → `RegisterSinglePrefab` → `RegisterPrefabsImpl` → `resolve_fn`（`RegisterPrefabsResolveAssets`）→ `resolvefilepath` 解析相对路径为绝对路径 → `TheSim:RegisterPrefab` 通知 C++ 引擎。
- **`resolvefilepath` 搜索策略**：先 `search_first_path`（Mod 根目录），再 `package.assetpath` 反向遍历（主程序 data 优先于 Mod），最后兜底原路径。找不到直接 `assert` 崩溃。
- **`ShouldIgnoreResolve`** 在两种情况跳过资源：(1) `INV_IMAGE` / `MINIMAP_IMAGE` 类型（它们不是真实路径）；(2) 专用服务器跳过所有音效、视频、fev 包——大幅加快专服启动速度。
- **Mod 的 `PrefabFiles`** 会被逐个 `LoadPrefabFile`，传入 `search_first_path = MODS_ROOT + modname + "/"`，保证 Mod 资源优先从自己目录找。
- **Mod 的 `Assets`** 被打包进一个虚拟 Prefab `"MOD_<modname>"`（没有工厂函数，`deps` 包含 Mod 所有 Prefab 名），这个虚拟 Prefab 游戏启动时立刻加载，触发所有全局资源一次性到位。
- **八个实战坑**：路径大小写、ATLAS + IMAGE 必须成对、全局 UI 资源不要放到具体 Prefab、Mod 路径不要带前缀、新 Prefab 必须登记到 PrefabFiles、`modinfo.lua` 的 `all_clients_require_mod` / `client_only_mod` 别忘设、客户端缓存问题要重启、Assets 列表太大影响启动速度。

> **下一节预告**：前六节我们建起了 Prefab 系统的完整知识体系——从"图纸是什么"到"Mod 资源挂载"。5.7 节是本章的**实战总结**——我们从零开始写一个完整的自定义物品 Mod：一把叫"**灵能短剑**"的武器。会涵盖 Prefab 文件结构、工厂函数五段式、tag 选用、MakeXxx 辅助函数、SetPristine 两段式、replica 考虑、Mod 的 `PrefabFiles / Assets` 声明——**把前六节所有知识点过一遍**。

## 5.7 实战：创建一个全新的自定义物品

### 本节导读

前面六节我们建立了 Prefab 系统的完整知识体系——从「图纸是什么」「工厂函数五段式」「Pristine 分界线」到「Tag 身份识别」「资源加载流程」。这一节是**整章的实战总结**：我们从零开始做一把叫「**灵能短剑**」（`psionic_shortsword`）的武器 Mod，把前六节所有的知识点串起来。

> **新手**读完 5.7.1-5.7.3 就能做出一把"看得见、捡得起、能堆叠"的物品，进游戏调个 `c_give("psionic_shortsword")` 就有成就感；**进阶读者**继续看 5.7.4-5.7.6，会给它加上**武器伤害、有限耐久、物品栏图标、合成配方**——它就是一把真正可以玩的武器了；**老手**可以跳到 5.7.7-5.7.8，了解如何用 `modinfo.configuration_options` 让玩家自己调伤害，以及用 `AddPrefabPostInit` **不创建新 Prefab 也能魔改原版长矛**。最后 5.7.9 列出所有我在写这个教程时亲手踩过的坑，供你速查。

**本节选定的物品设计**：

| 属性 | 数值 | 说明 |
|------|------|------|
| 名字 | 灵能短剑 | 英文 `psionic_shortsword` |
| 基础伤害 | 38 | 比长矛（34）略高 |
| 耐久次数 | 150 | 和长矛相同 |
| 特殊效果 | 命中时消耗施法者 1 点理智 | 展示多组件配合 |
| 合成材料 | 3 金块 + 1 噩梦燃料 + 2 木头 | 在炼金引擎处合成 |

我们用**灵能短剑**做例子，因为它涵盖的知识点正好**完整覆盖前六节**：

- `SetBank / SetBuild`（5.1、5.2）——如何复用长矛的动画让它有可见外观
- 工厂函数五段式（5.2.2）——一步步搭骨架
- `SetPristine` 分界（5.4）——哪些代码在客户端执行
- tag 提前声明（5.5.4）——`"weapon"`、`"sharp"` 如何做性能优化
- `Assets` 和 `PrefabFiles`（5.6）——Mod 资源声明的正确姿势

---

### 5.7.1 快速入门：Mod 目录骨架与 `modinfo.lua`

打开你本地的 Don't Starve Together 的 `mods/` 目录（通常在 Steam 安装目录下），新建一个文件夹 `psionic_shortsword_mod/`。这个文件夹就是我们整个 Mod 的"工作区"。

#### 第一步：最小目录结构

对一个**物品类 Mod**来说，最小的目录结构是：

```
psionic_shortsword_mod/
├── modinfo.lua          ← Mod 的元信息（名字、作者、版本、兼容性）
├── modmain.lua          ← Mod 的入口文件（加载 Prefab、注册资源）
├── modicon.tex          ← Mod 在游戏列表里的 128x128 图标
├── modicon.xml          ← 图标的图集描述
├── scripts/
│   └── prefabs/
│       └── psionic_shortsword.lua   ← 我们的物品定义
├── anim/                ← 动画资源（.zip）
└── images/
    └── inventoryimages/
        ├── psionic_shortsword.tex   ← 64x64 物品栏图标
        └── psionic_shortsword.xml   ← 对应的图集
```

**每个文件的职责**：

| 文件/目录 | 职责 | 什么时候必须 |
|-----------|------|--------------|
| `modinfo.lua` | 描述 Mod 本身（不含游戏逻辑） | **必须**——游戏靠它识别 Mod |
| `modmain.lua` | Mod 的引导脚本——注册资源、加载 Prefab、设置 STRINGS | **必须**——游戏启动时执行它 |
| `modicon.tex/xml` | Mod 列表里的小图标 | **必须**（否则 Mod 列表里显示红框） |
| `scripts/prefabs/*.lua` | 具体的 Prefab 定义 | 想添加新物品就**必须** |
| `anim/*.zip` | 动画资源 | 有自定义动画时**必须** |
| `images/inventoryimages/*` | 物品栏 64x64 图标 | 有新物品时**必须**（否则图标是紫底红叹号） |

> **开发逻辑**：Klei 强制要求这几个文件，是因为游戏在"扫描 Mod 列表"阶段就要读 `modinfo.lua`，此时根本没执行 `modmain.lua`——两个文件解耦是为了**让玩家在启用 Mod 之前就能看到它的信息**（名字、作者、配置选项）。

#### 第二步：写 `modinfo.lua`

对照 `mods/联机版mod/renwu/modinfo.lua` 的风格，我们的版本如下：

```lua
name = "灵能短剑"
description = "来自《饥荒联机版 Mod 开发白皮书》第 5 章实战教程的示例物品。"
author = "你的名字"
version = "1.0"

forumthread = ""
api_version = 10               -- 联机版 API 版本，填 10 即可
dst_compatible = true          -- 兼容联机版（DST）
dont_starve_compatible = false -- 不兼容单机版

icon_atlas = "modicon.xml"
icon = "modicon.tex"

-- 联机关键开关
all_clients_require_mod = true  -- 客机必须装这个 Mod 才能连进来
client_only_mod = false         -- 不是纯客户端 Mod（它改了游戏逻辑）
server_only_mod = false         -- 也不是纯服务端 Mod

-- 先留空，5.7.7 节再加配置项
configuration_options = {}
```

**关键字段解释**：

| 字段 | 我们的值 | 说明 |
|------|---------|------|
| `api_version` | `10` | 联机版 API，固定 `10`（**不要**填成 `6` 那是单机版） |
| `dst_compatible` | `true` | 联机版兼容 |
| `dont_starve_compatible` | `false` | 单机版不兼容（联机和单机数据结构有差异） |
| `all_clients_require_mod` | `true` | 因为我们**加了新 Prefab**——客户端本地必须有资源 |
| `client_only_mod` | `false` | 我们改游戏逻辑（加物品），不是纯 UI Mod |

> **为什么 `all_clients_require_mod = true`？** 回顾 5.6.8 节的坑 6：主机启用这个 Mod 时，Lua 脚本会自动同步给客户端，但客户端本地**必须自己有资源**（`.zip`、`.tex`、`.xml`），否则他们看到的"灵能短剑"会是紫底红叹号。

#### 第三步：写一个最小的 `modmain.lua`（暂时留空）

```lua
-- modmain.lua 最小版
-- 让 env（Mod 沙箱环境）能直接访问全局变量
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL.rawget(GLOBAL, k)
    end
})

-- 先声明一个空的 PrefabFiles，5.7.3 节我们填它
PrefabFiles = {}
Assets = {}
```

**那段 `setmetatable(env, ...)` 做什么？** Mod 的脚本是在一个**沙箱环境 `env`** 里跑的——直接写 `TheWorld` 访问不到，要写 `GLOBAL.TheWorld`。这段代码做了一个"**穿透**"：在沙箱里找不到的名字，自动去 `GLOBAL` 里找——相当于以后你写 `TheWorld.ismastersim` 就能直接生效，不用每次都写 `GLOBAL.TheWorld.ismastersim`。

这是 Mod 圈子的惯用法，几乎每个 Mod 的 `modmain.lua` 开头都有这段（见 `renwu/modmain.lua` 第 2-6 行）。

> **新手记忆**：三个文件画个心智图——`modinfo.lua` 是 Mod 的"身份证"，`modmain.lua` 是 Mod 的"主函数"，`scripts/prefabs/*.lua` 是 Mod 里具体的"图纸"。我们到这里身份证写好了，主函数留了个空壳，接下来该画图纸了。

---

### 5.7.2 快速入门：最小可运行的 Prefab —— `psionic_shortsword.lua`

现在进入最重要的一步：写 Prefab 文件。我们**暂时不考虑美术资源**——先让它借用长矛的动画跑起来，看到"物品确实在世界里"，再一步步加功能。

#### 第一步：完整骨架（五段式回顾）

创建 `scripts/prefabs/psionic_shortsword.lua`：

```lua
-- psionic_shortsword.lua —— 第一版，最小可运行
local assets =
{
    -- 暂时借用长矛的动画，所以这里不声明自己的 anim
    -- 等 5.7.5 我们加自定义图标时再补
}

local function fn()
    -- ① 创建实体 + 引擎组件
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    -- ② 物理 + 视觉（借用长矛的动画）
    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("spear")
    inst.AnimState:SetBuild("swap_spear")
    inst.AnimState:PlayAnimation("idle")

    -- ③ 客户端也要识别的标签
    inst:AddTag("sharp")
    inst:AddTag("pointy")
    inst:AddTag("weapon")   -- 从 weapon 组件提前加到 pristine，优化性能

    MakeInventoryFloatable(inst, "med", 0.05, 0.75)

    -- ④ SetPristine 分界
    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst
    end

    -- ⑤ 服务端组件
    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")

    MakeHauntableLaunch(inst)

    return inst
end

return Prefab("psionic_shortsword", fn, assets)
```

这 30 行代码已经具备一个可生成、可拾起、可堆叠、被鬼可附身的完整物品了——**它和 `log.lua`、`flint.lua` 的结构几乎一模一样**（回顾 5.1.2 和 5.2.2 节）。我们重点关注两处选型：

#### 第二步：为什么 `SetBank("spear") / SetBuild("swap_spear")`？

这是本节最关键的**偷懒技巧**——**让新物品复用原版的动画资源**。

看一下回忆 5.2.2 节里对 `AnimState` 两个函数的解释：

- `SetBank("spear")` —— 告诉引擎用 `spear` 这个**动画骨架**（包含所有动画片段的骨骼定义）
- `SetBuild("swap_spear")` —— 告诉引擎用 `swap_spear` 这个**外观皮肤**（具体贴图）

这两个名字来自原版 `scripts/prefabs/spear.lua` 第 37-38 行：

```lua
inst.AnimState:SetBank("spear")
inst.AnimState:SetBuild("swap_spear")
```

**为什么能复用？** 因为联机版的动画系统里，**只要 bank 和 build 的名字在游戏里已经注册过**（长矛已经加载了 `anim/spear.zip` 和 `anim/swap_spear.zip`），任何 Prefab 都可以直接 `SetBank/SetBuild` 到这些资源上。**我们不需要在 `assets` 里重新声明**——因为引擎已经在加载长矛时加载过这些 zip 了。

> **开发逻辑**：原版的 `anim/spear.zip` 在游戏启动时就已加载（因为原版长矛是玩家常合成的物品），我们的新物品直接复用它等于"白嫖"。**坏处**：玩家看到的"灵能短剑"长得和普通长矛一模一样；**好处**：你可以先不用学动画制作，专注于代码逻辑。后面 5.7.5 节会讲怎么换自定义图标。

#### 第三步：为什么提前加 `"weapon"` 标签？

对比 `spear.lua` 第 44-45 行的这条注释：

```lua
--weapon (from weapon component) added to pristine state for optimization
inst:AddTag("weapon")
```

**正常情况**下，你在服务端 `inst:AddComponent("weapon")` 时，组件会**自动**给实体加一个 `"weapon"` 标签——但这个标签是在 replica 同步后客户端才能看到，有几帧延迟。

Klei 的做法是**手动在 pristine 阶段提前加这个标签**，让客户端一拿到实体就能通过 `inst:HasTag("weapon")` 识别它是武器——这样 UI 渲染、动作判断、动画播放都不用等组件同步。这是 5.5.4 节讲的"用 tag 避免组件同步"的典型案例。

> **新手记忆**：**给武器类 Prefab 提前加 `"weapon"` 标签，给光源类提前加 `"lighter"` 标签，给投掷物提前加 `"projectile"` 标签**——这是 Klei 推荐的标准写法，源码里到处都是这种注释。

#### 第四步：验证"能跑"

此时如果我们把 Mod 完整装好（再等 5.7.3 节讲注册），游戏启动控制台输入：

```
c_spawn("psionic_shortsword")
```

一个**看起来和长矛一模一样的物品**会出现在你脚下。你可以：

- **走过去** → 提示「拾起」
- **E 键捡起** → 它进入你的背包
- **右上角背包槽**显示一把长矛图标（因为我们借用了长矛的 UI）

这时虽然它"看起来像长矛"，但它在**代码层面已经是一个独立的 Prefab**——`c_find("psionic_shortsword")` 能找到它、`inst.prefab == "psionic_shortsword"` 会返回 `true`。

**但它还不能当武器使**——右键砸蜘蛛没有伤害，因为我们还没加 `weapon` 组件。这就是下一节（5.7.4）的事了。

---

### 5.7.3 快速入门：在 `modmain.lua` 里"登记"你的 Prefab

光写了 `psionic_shortsword.lua` 游戏是**认不出它**的——必须在 `modmain.lua` 里登记才能让引擎加载。回顾 5.6.7 节我们讲过的 `PrefabFiles` 和 `Assets` 机制，这节把它落到实处。

#### 第一步：在 `PrefabFiles` 里加文件名

打开 `modmain.lua`，把 `PrefabFiles` 改成：

```lua
PrefabFiles = {
    "psionic_shortsword",   -- 对应 scripts/prefabs/psionic_shortsword.lua
}
```

**注意三件事**：

1. **不带 `.lua` 后缀**——引擎会自动补
2. **不带 `prefabs/` 前缀**——引擎会自动加（回顾 5.6.7 节里 `"prefabs/"..prefab_path`）
3. **文件名全小写**——Linux 专服大小写敏感（5.6.8 节坑 1）

这一行会让 `ModWrangler:RegisterPrefabs` 在游戏启动时做这些事（回顾 5.6.7 节的源码）：

```
① LoadPrefabFile("prefabs/psionic_shortsword", nil, MODS_ROOT.."psionic_shortsword_mod/")
② 执行 psionic_shortsword.lua → 拿到 return 的 Prefab 对象
③ 注册到 Prefabs["psionic_shortsword"] = prefab
④ 这把"灵能短剑"就能被 SpawnPrefab 找到了
```

#### 第二步：`Assets = {}` 暂时留空

因为这一版我们**完全复用原版的动画资源**，没有自己的 `.zip`、`.tex`、`.xml`——所以 `Assets` 可以留空：

```lua
Assets = {}
```

等 5.7.5 节加物品栏图标时，这里会填几行：

```lua
-- 预告：5.7.5 节我们会这样补
Assets = {
    Asset("ATLAS", "images/inventoryimages/psionic_shortsword.xml"),
    Asset("IMAGE", "images/inventoryimages/psionic_shortsword.tex"),
    Asset("ATLAS_BUILD", "images/inventoryimages/psionic_shortsword.xml", 256),
}
```

#### 第三步：完整的第一版 `modmain.lua`

```lua
-- modmain.lua —— 第一版，最小可运行
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL.rawget(GLOBAL, k)
    end
})

PrefabFiles = {
    "psionic_shortsword",
}

Assets = {}
```

**6 行代码，加上 30 行 Prefab**——一个新物品 Mod 就成型了。这个 Mod 装进游戏（放到 `mods/psionic_shortsword_mod/`）+ 主菜单启用，进存档后控制台输入 `c_spawn("psionic_shortsword")`，你就能得到世界上第一把属于你的"灵能短剑"（虽然它还不能打架）。

#### 第四步：理解引擎背后发生了什么

把 5.6 节学到的流程再走一遍——**你启用这个 Mod 后，游戏启动做了哪些事**：

```
游戏启动
  │
  ├── 扫描 mods/psionic_shortsword_mod/modinfo.lua
  │     ├── 读到 name = "灵能短剑" 显示在 Mod 列表
  │     └── all_clients_require_mod = true 记录为"硬依赖"
  │
  ├── 玩家启用 Mod，加载存档
  │
  ├── ModManager:LoadMods
  │     ├── 执行 modmain.lua（在沙箱 env 里）
  │     │     ├── setmetatable(env, ...)  让 env 能穿透访问全局
  │     │     ├── PrefabFiles = {"psionic_shortsword"}
  │     │     └── Assets = {}
  │     │
  │     └── 此时 mod.PrefabFiles 和 mod.Assets 已被 Wrangler 记录
  │
  ├── ModWrangler:RegisterPrefabs
  │     ├── 对每个 PrefabFiles 名字调用 LoadPrefabFile
  │     │     └── 执行 scripts/prefabs/psionic_shortsword.lua
  │     │           ├── 返回 Prefab("psionic_shortsword", fn, {})
  │     │           └── 注册到 Prefabs["psionic_shortsword"] = prefab
  │     │
  │     └── 创建虚拟 Prefab "MOD_psionic_shortsword_mod"
  │           └── TheSim:LoadPrefabs 触发所有资源加载（本例因 Assets 为空，空转）
  │
  └── 游戏进行中
        └── c_spawn("psionic_shortsword") 可以成功执行
              └── 回顾 5.1.5 节的 SpawnPrefabFromSim 全流程
```

**你写的 6 行 modmain + 30 行 Prefab**，触发了上面一整条引擎内部的机制——这就是 Mod 系统的魅力：**引擎把你的文件当作一等公民**，和原版 Prefab 一视同仁。

> **新手记忆到这里**：写 Mod 的**最小闭环**是四步——建目录、写 modinfo、写 Prefab、在 modmain 的 PrefabFiles 登记。**缺一不可**。接下来我们进入进阶部分，给这把"看得见的假长矛"加上真正的战斗属性。

---

### 5.7.4 进阶：让它"真正能用"—— 组件叠加

新手版的灵能短剑缺少两件事：**不能打人、不能装备**。这一节我们按**组件叠加**的思路把它补齐，同时展示"给命中目标扣施法者 sanity"这种**多组件配合**的效果。

#### 第一步：装备逻辑 —— `equippable` + onequip/onunequip

长矛是怎么"被拿在手里"的？看 `scripts/prefabs/spear.lua` 第 7-26 行：

```7:26:scripts/prefabs/spear.lua
local function onequip(inst, owner)
    local skin_build = inst:GetSkinBuild()
    if skin_build ~= nil then
        owner:PushEvent("equipskinneditem", inst:GetSkinName())
        owner.AnimState:OverrideItemSkinSymbol("swap_object", skin_build, "swap_spear", inst.GUID, "swap_spear")
    else
        owner.AnimState:OverrideSymbol("swap_object", "swap_spear", "swap_spear")
    end
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end

local function onunequip(inst, owner)
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
    local skin_build = inst:GetSkinBuild()
    if skin_build ~= nil then
        owner:PushEvent("unequipskinneditem", inst:GetSkinName())
    end
end
```

**这段代码在做什么？** 你可以把玩家的动画想象成一套"**预置骨架 + 可替换贴图**"——玩家手里有一个叫 `"swap_object"` 的挂载点，它的贴图会随着你装备不同的物品而切换。

- **装备时**：把 `"swap_object"` 的贴图**替换成 `"swap_spear"`**，再显示 `"ARM_carry"` 骨骼（握着东西的手臂）、隐藏 `"ARM_normal"` 骨骼（空手）
- **卸下时**：反过来——切回空手动画

中间那段 `if skin_build ~= nil` 处理的是**皮肤**：皮肤版的长矛外观不一样，要用 `OverrideItemSkinSymbol` 的多层覆盖。我们现在不做皮肤，直接抄外层逻辑就够了。

#### 第二步：战斗逻辑 —— `weapon` + `finiteuses` + 命中回调

我们想让它**每次命中消耗 1 点施法者理智**——这要求在 `weapon:SetOnAttack` 里写一个回调：

```lua
local function onattack(weapon, attacker, target)
    -- target 可能在伤害结算阶段被杀死或删除
    if attacker ~= nil and attacker:IsValid() and attacker.components.sanity ~= nil then
        attacker.components.sanity:DoDelta(-TUNING.PSIONIC_SHORTSWORD_SANITY_COST)
    end
end
```

**几个容易踩坑的点**：

1. **判断 `attacker:IsValid()`**——攻击者可能在命中瞬间死了（被怪反杀），必须先判空
2. **判断 `attacker.components.sanity ~= nil`**——不是所有实体都有 sanity 组件（比如幽灵、机器人 WX-78），怕崩溃
3. **用 `TUNING` 常量而不是硬编码 `-1`**——方便后面 5.7.7 节做配置项

在 `modmain.lua` 顶部先加一条 TUNING：

```lua
TUNING.PSIONIC_SHORTSWORD_DAMAGE = 38
TUNING.PSIONIC_SHORTSWORD_USES = 150
TUNING.PSIONIC_SHORTSWORD_SANITY_COST = 1
```

#### 第三步：完整的第二版 Prefab

整合装备 + 战斗 + 耐久，现在的 `psionic_shortsword.lua` 完整版：

```lua
-- psionic_shortsword.lua —— 第二版：可装备 + 可战斗 + 消耗理智
local assets =
{
}

local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_object", "swap_spear", "swap_spear")
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end

local function onunequip(inst, owner)
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
end

local function onattack(weapon, attacker, target)
    if attacker ~= nil and attacker:IsValid() and attacker.components.sanity ~= nil then
        attacker.components.sanity:DoDelta(-TUNING.PSIONIC_SHORTSWORD_SANITY_COST)
    end
end

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("spear")
    inst.AnimState:SetBuild("swap_spear")
    inst.AnimState:PlayAnimation("idle")

    inst:AddTag("sharp")
    inst:AddTag("pointy")
    inst:AddTag("weapon")

    MakeInventoryFloatable(inst, "med", 0.05, 0.75)

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("weapon")
    inst.components.weapon:SetDamage(TUNING.PSIONIC_SHORTSWORD_DAMAGE)
    inst.components.weapon:SetOnAttack(onattack)

    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(TUNING.PSIONIC_SHORTSWORD_USES)
    inst.components.finiteuses:SetUses(TUNING.PSIONIC_SHORTSWORD_USES)
    inst.components.finiteuses:SetOnFinished(inst.Remove)

    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")

    inst:AddComponent("equippable")
    inst.components.equippable:SetOnEquip(onequip)
    inst.components.equippable:SetOnUnequip(onunequip)

    MakeHauntableLaunch(inst)

    return inst
end

return Prefab("psionic_shortsword", fn, assets)
```

**对照 `spear.lua` 完整版**（第 28-77 行），会发现我们的代码几乎是它的一个**精简分支**——只多了 `SetOnAttack(onattack)` 一行和我们自己的 TUNING 常量。这就是**组件系统的威力**：**组合已有组件** = 造出完全新的物品，不需要写底层代码。

#### 第四步：为什么 `finiteuses.SetOnFinished(inst.Remove)`

对照 `spear.lua` 第 64 行：

```lua
inst.components.finiteuses:SetOnFinished(inst.Remove)
```

`finiteuses` 组件维护"剩余使用次数"——每次攻击后自动 -1（其实是 `combat` 组件触发 `weapon:OnAttack` 同时走了 finiteuses 的扣减流程，引擎内部打通）。当归零时会调用 `OnFinished` 回调。

这里 `inst.Remove` 是个**方法引用**——等价于 `function() inst:Remove() end`。耐久归零 → 物品消失。

> **开发逻辑**：Klei 不把"耐久归零自动 Remove"做成组件默认行为，是因为不同物品的"用完"行为不一样：
> - **武器类**：消失（`inst.Remove`）
> - **背包**：变成空背包（不消失）
> - **火把**：有专属销毁动画（`torch.lua` 第 262 行的 `ErodeAway(inst)`）
>
> 所以 `SetOnFinished` 让你**自己决定**用完后怎么处理。

#### 第五步：测试"真的能打"

此时你把 Mod 重装、进游戏，控制台：

```
c_give("psionic_shortsword")    -- 直接给到背包
c_spawn("pigman")               -- 再召个猪人当沙包
```

装备短剑、右键猪人——每次命中：

- 猪人掉 38 点血（比长矛 34 点高）
- 你自己掉 1 点 sanity（右上角数字变绿下跳）
- 用了 150 次后剑消失

**到这里它是一把完整可玩的武器了。** 下一节我们让它"看起来像自己"（物品栏图标），而不是借用长矛的外观。

---

### 5.7.5 进阶：STRINGS 与物品栏图标 —— 让玩家"看得见、叫得出"

现在这把短剑有两个"没做的事"：

- **物品栏图标是长矛**（因为我们复用了 `swap_spear` 的 build）
- **检查物品时名字叫 `MISSING NAME`，描述叫 `MISSING DESCRIPTION`**

这一节我们把它们补齐。

#### 第一步：STRINGS 文本注册

在 `modmain.lua` 里加上：

```lua
-- 在 modmain.lua 任意位置
STRINGS.NAMES.PSIONIC_SHORTSWORD = "灵能短剑"
STRINGS.CHARACTERS.GENERIC.DESCRIBE.PSIONIC_SHORTSWORD = "锋利的刃上流淌着紫色的光。"
STRINGS.RECIPE_DESC.PSIONIC_SHORTSWORD = "以理智为代价的锋利。"
```

**三条 STRINGS 各自的用途**：

| STRINGS 路径 | 在哪里显示 |
|--------------|------------|
| `STRINGS.NAMES.<PREFAB_UPPER>` | 鼠标悬停、"掉落物"提示等 |
| `STRINGS.CHARACTERS.GENERIC.DESCRIBE.<PREFAB_UPPER>` | 检查时（按 T 或右键 -> 查看）角色说的话 |
| `STRINGS.RECIPE_DESC.<PREFAB_UPPER>` | 合成菜单里鼠标悬停时的物品描述 |

**关键约定**：**键名必须用 Prefab 名的大写形式**——`psionic_shortsword` → `PSIONIC_SHORTSWORD`。这是 Klei 在 `scripts/strings.lua` 里的固定约定，查找时会 `string.upper(prefab_name)`。

参考 `renwu/modmain.lua` 第 208-210 行的批量写法：

```208:210:mods/联机版mod/renwu/modmain.lua
        STRINGS.NAMES[string.upper(v.prefabname)] = v.name_cn
        STRINGS.CHARACTERS.GENERIC.DESCRIBE[string.upper(v.prefabname)] = v.describe
        STRINGS.RECIPE_DESC[string.upper(v.prefabname)] = v.desc
```

#### 第二步：角色专属检查文字（可选）

如果你想让 WX-78 说"这玩意居然能吸灵能 01010"，可以加：

```lua
STRINGS.CHARACTERS.WX78.DESCRIBE.PSIONIC_SHORTSWORD = "灵能短剑？是某种生物兼容的武器。"
STRINGS.CHARACTERS.WENDY.DESCRIBE.PSIONIC_SHORTSWORD = "它吸食的是你我的灵魂…"
```

引擎在 `scripts/entityscript.lua` 的 `Describe` 函数会**按角色名找不到就 fallback 到 `GENERIC`**——所以只写 `GENERIC` 也没问题，其他角色都会用那句。

#### 第三步：物品栏图标（`images/inventoryimages/*`）

物品栏图标是 **64x64 的 png 打包出来的 `tex` + `xml`**。制作方法超出代码范围（要用 Klei 的 `TEXTools` 或 `ktools` 打包），但**注册流程是代码层的**，我们重点讲这部分。

假设你已经做好了 `images/inventoryimages/psionic_shortsword.tex` 和 `psionic_shortsword.xml` 放进 Mod 目录。在 `modmain.lua` 里注册：

```lua
-- modmain.lua —— 物品栏图标注册
local imageassets =
{
    "psionic_shortsword",
}

for k, v in pairs(imageassets) do
    table.insert(Assets, Asset("ATLAS", "images/inventoryimages/"..v..".xml"))
    table.insert(Assets, Asset("IMAGE", "images/inventoryimages/"..v..".tex"))
    table.insert(Assets, Asset("ATLAS_BUILD", "images/inventoryimages/"..v..".xml", 256))
    RegisterInventoryItemAtlas("images/inventoryimages/"..v..".xml", v..".tex")
end
```

**这段代码的四步分别做什么？**

| 代码行 | 作用 |
|--------|------|
| `Asset("ATLAS", ...)` | 声明图集 XML（描述 tex 里每个子图的坐标） |
| `Asset("IMAGE", ...)` | 声明贴图 TEX 本身 |
| `Asset("ATLAS_BUILD", xml, 256)` | 把图集也作为**动画素材**注册——让引擎把它当成一个 256x256 的 build，可以被 `SetBuild` 引用 |
| `RegisterInventoryItemAtlas(xml, tex)` | 在全局 `inventoryItemAtlasLookup` 表里登记——让其他 Mod/系统知道"这个 tex 的图集在哪" |

这是 `renwu/modmain.lua` 第 110-117 行的标准写法，**几乎所有物品 Mod 都照抄**。

#### 第四步：告诉 `inventoryitem` 组件用哪张图

光注册了图集还不够——还要告诉 Prefab 的 `inventoryitem` 组件**"物品栏里显示的就是这张图"**。看 `haze_paperclip.lua` 第 30-31 行：

```30:31:mods/联机版mod/renwu/scripts/prefabs/haze_paperclip.lua
    inst.components.inventoryitem.imagename = 'haze_paperclip'
    inst.components.inventoryitem.atlasname = 'images/inventoryimages/haze_paperclip.xml'
```

我们在 `psionic_shortsword.lua` 的服务端段落里加上同样两行：

```lua
-- 在 inst:AddComponent("inventoryitem") 下面加
inst.components.inventoryitem.imagename = "psionic_shortsword"
inst.components.inventoryitem.atlasname = "images/inventoryimages/psionic_shortsword.xml"
```

引擎会根据这两个字段**自动**选图——背包格、物品掉在地上的悬浮图标、合成菜单里的小图都是这套机制。

**如果不写这两行会发生什么？** 引擎会按默认规则找 `images/inventoryimages/<prefab>.xml`——**只要你的文件名和 Prefab 名一致**（我们这里就是一致的），其实可以不写。但**显式写上**的好处是清晰、支持重命名、便于跨 Mod 引用。

#### 第五步：完整的 `modmain.lua`（第三版）

```lua
-- modmain.lua —— 第三版：加上 STRINGS 和物品栏图标
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL.rawget(GLOBAL, k)
    end
})

TUNING.PSIONIC_SHORTSWORD_DAMAGE = 38
TUNING.PSIONIC_SHORTSWORD_USES = 150
TUNING.PSIONIC_SHORTSWORD_SANITY_COST = 1

PrefabFiles = {
    "psionic_shortsword",
}

Assets = {}

local imageassets =
{
    "psionic_shortsword",
}

for k, v in pairs(imageassets) do
    table.insert(Assets, Asset("ATLAS", "images/inventoryimages/"..v..".xml"))
    table.insert(Assets, Asset("IMAGE", "images/inventoryimages/"..v..".tex"))
    table.insert(Assets, Asset("ATLAS_BUILD", "images/inventoryimages/"..v..".xml", 256))
    RegisterInventoryItemAtlas("images/inventoryimages/"..v..".xml", v..".tex")
end

STRINGS.NAMES.PSIONIC_SHORTSWORD = "灵能短剑"
STRINGS.CHARACTERS.GENERIC.DESCRIBE.PSIONIC_SHORTSWORD = "锋利的刃上流淌着紫色的光。"
STRINGS.RECIPE_DESC.PSIONIC_SHORTSWORD = "以理智为代价的锋利。"
```

到这里，游戏里这把短剑：

- 背包槽显示**你自己画的图标**
- 鼠标悬停显示**灵能短剑**
- 按 T 检查时，角色说出**"锋利的刃上流淌着紫色的光。"**

**它看起来完全像一把正式的武器了。** 但目前还要靠 `c_give` 作弊给自己——下一节我们加上合成配方，让玩家能正常合成它。

---

### 5.7.6 进阶：添加合成配方 `AddRecipe2`

有了物品、有了名字和图标，最后一步是**让玩家能合成它**。这一节讲 `AddRecipe2` 函数怎么用。

#### 第一步：`AddRecipe2` 的签名

看 `scripts/modutil.lua` 第 732 行：

```732:756:scripts/modutil.lua
	env.AddRecipe2 = function(name, ingredients, tech, config, filters)
		initprint("AddRecipe2", name)
		require("recipe")
		mod_protect_Recipe = false
		local rec = Recipe2(name, ingredients, tech, config)

		if not rec.is_deconstruction_recipe then
			if config ~= nil and config.nounlock then
				env.AddRecipeToFilter(name, CRAFTING_FILTERS.CRAFTING_STATION.name)
			else
				env.AddRecipeToFilter(name, CRAFTING_FILTERS.MODS.name)
			end

			if filters ~= nil then
				for _, filter_name in ipairs(filters) do
					env.AddRecipeToFilter(name, filter_name)
				end
			end
		end


		mod_protect_Recipe = true
		rec:SetModRPCID()
		return rec
	end
```

它接收 **5 个参数**：

| 参数 | 类型 | 含义 |
|------|------|------|
| `name` | string | 合成出来的 Prefab 名（`"psionic_shortsword"`） |
| `ingredients` | table | `Ingredient` 数组（材料列表） |
| `tech` | `TECH.XXX` | 需要的科技等级（如 `TECH.SCIENCE_ONE`） |
| `config` | table | 可选配置（图标、自定义图集、过滤等） |
| `filters` | table | 可选，额外加入的分类过滤器（如 `"WEAPONS"`） |

返回一个 `Recipe2` 对象（你通常不用它，除非要做后续特殊处理）。

#### 第二步：定义材料列表（`Ingredient`）

`Ingredient` 是什么？看 `scripts/recipe.lua` 第 5 行：

```lua
Ingredient = Class(function(self, ingredienttype, amount, atlas, deconstruct, imageoverride)
    -- ...
end)
```

最常用的就 **前 3 个参数**：

- `ingredienttype`（string）——材料 Prefab 名
- `amount`（number）——需要几个
- `atlas`（string，可选）——如果材料是 Mod 物品，要指定图集路径（否则合成菜单里材料图标显示不出来）

**本节的配方**：3 金块 + 1 噩梦燃料 + 2 木头。代码：

```lua
local ingredients =
{
    Ingredient("goldnugget", 3),
    Ingredient("nightmarefuel", 1),
    Ingredient("log", 2),
}
```

#### 第三步：选择科技等级（`TECH`）

`TECH` 是一张表，常见值：

| TECH | 需要的工作台 |
|------|--------------|
| `TECH.NONE` | 无 —— 随时可合成 |
| `TECH.SCIENCE_ONE` | 科学机器附近 |
| `TECH.SCIENCE_TWO` | 炼金引擎附近 |
| `TECH.MAGIC_TWO` | 暗影操纵者附近 |
| `TECH.MAGIC_THREE` | 棱镜 / 潜影头骨附近 |
| `TECH.ANCIENT_TWO` | 远古伪科学基地 |

这些值定义在 `scripts/techtree.lua`。我们的灵能短剑涉及"噩梦燃料"，很自然应该在**炼金引擎**（`SCIENCE_TWO`）合成：

```lua
local tech = TECH.SCIENCE_TWO
```

#### 第四步：config 配置（图标、过滤器）

```lua
local config =
{
    atlas = "images/inventoryimages/psionic_shortsword.xml",
    image = "psionic_shortsword.tex",
    -- nounlock = false,      -- 默认 false，需要科学机器试玩后解锁
    -- numtogive = 1,         -- 默认 1，合成一次得多少个
    -- min_spacing = nil,     -- 建筑物用：最小间距
}
```

`atlas` 和 `image` 告诉合成菜单用哪张图作为配方封面——和我们 5.7.5 节注册的图集一致。

#### 第五步：`filters` 参数——归到哪个分类

`filters` 决定这把剑在合成菜单里**出现在哪个分类标签下**。常见值（见 `scripts/crafting_filters.lua`）：

| filter | 分类 |
|--------|------|
| `"TOOLS"` | 工具 |
| `"WEAPONS"` | 武器 |
| `"LIGHT"` | 光源 |
| `"SURVIVAL"` | 生存 |
| `"MAGIC"` | 魔法 |
| `"MODS"` | Mods（默认总会加入这个） |

回头看 `AddRecipe2` 源码第 739-749 行：

- **默认总会加到 `CRAFTING_FILTERS.MODS`**（你不指定也会有）
- `filters` 参数是**额外**加到其他分类

既然灵能短剑是魔法武器，我们想让它同时出现在"武器"和"魔法"分类下：

```lua
local filters = { "WEAPONS", "MAGIC" }
```

#### 第六步：完整的合成注册代码

在 `modmain.lua` 最后加上：

```lua
AddRecipe2(
    "psionic_shortsword",
    {
        Ingredient("goldnugget", 3),
        Ingredient("nightmarefuel", 1),
        Ingredient("log", 2),
    },
    TECH.SCIENCE_TWO,
    {
        atlas = "images/inventoryimages/psionic_shortsword.xml",
        image = "psionic_shortsword.tex",
    },
    { "WEAPONS", "MAGIC" }
)
```

重装 Mod 进游戏，打开合成菜单 → "武器" 分类下 → 就能看到你的灵能短剑（如果当前没站在炼金引擎旁会显示**灰色 + 提示"需要炼金引擎"**）。材料齐全后点击 → 按住鼠标不放 → 合成成功。

> **开发逻辑**：为什么要做分类？因为联机版有上百个可合成物品，全堆一起玩家根本找不到。分类过滤器（5.6 Uncompressed 更新后的大改）让玩家可以**按"武器"/"工具"/"光源"**等聚合浏览。作为 Mod 开发者，**一定要把配方加到合适的分类**——否则玩家得翻到 MODS 分类最底下才能找到，体验很差。

#### 第七步：配方解锁——`TECH.NONE` vs 科技栏

默认情况下，玩家要**先合成过一次**这个物品才能"解锁"它——下次合成时配方会自动出现。如果你想让配方**默认就显示出来**（不需要手动点合成按钮），可以设：

```lua
local config =
{
    atlas = "...",
    image = "...",
    nounlock = true,   -- 默认解锁，不需要首次合成触发
}
```

这在**主打"无限次合成"的 Mod**（比如书籍类、特殊装备类）里常用。普通材料类物品**建议保持默认 `nounlock = false`**，让玩家自己发现新内容也是游戏乐趣的一部分。

---

### 5.7.7 老手进阶：`modinfo.configuration_options` 与 TUNING 联动

前面我们把伤害、耐久、理智消耗都写进了 `TUNING` 常量。但每个玩家的偏好不同——有人想要"38 伤害 + 1 sanity"，有人想要"50 伤害 + 3 sanity"。**怎么让玩家在 Mod 页面自己调？** 这就需要 `modinfo.configuration_options`。

#### 第一步：`configuration_options` 的格式

看 `renwu/modinfo.lua` 第 19-109 行就是完整样板：

```19:40:mods/联机版mod/renwu/modinfo.lua
configuration_options = {
    -----------------------------------
    -- {
    --     name = "Obrsword_navoice",--自定义选项名称,用于后续GetModConfigData设置
    --     label = "普通攻击音效",--选项名称，用于mod设置中看见的名称
    --     hover = "是否开启普通攻击的集中音效。",--选项标题下对应的描述
    --     options =
    --     {
    --         -- {description = "自动", data = "auto"},
    --         { 
    --             description = "开启(默认)",--选项名称
    --             hover = "是否开启普通攻击的集中音效。",--选项标题下对应的描述
    --             data = true --选项的值,用于后续GetModConfigData拿到的值
    --         },
```

每个选项是一个表，有 5 个关键字段：

| 字段 | 类型 | 作用 |
|------|------|------|
| `name` | string | **代码里**用来读取配置的键（`GetModConfigData("name")`） |
| `label` | string | **Mod 配置界面**显示的选项标题 |
| `hover` | string | 鼠标悬停时显示的说明 |
| `options` | table | 可选值的数组——每个元素有 `description`（界面文字）+ `data`（实际值） |
| `default` | any | 默认选中哪个 `data` 值 |

#### 第二步：给灵能短剑加三个配置项

打开 `modinfo.lua`，把原来的 `configuration_options = {}` 改成：

```lua
configuration_options = {
    {
        name = "PSIONIC_SHORTSWORD_DAMAGE",
        label = "基础伤害",
        hover = "灵能短剑每次攻击的基础伤害。",
        options = {
            { description = "低（28）",       data = 28 },
            { description = "中（38，默认）", data = 38 },
            { description = "高（50）",       data = 50 },
            { description = "极高（68）",     data = 68 },
        },
        default = 38,
    },
    {
        name = "PSIONIC_SHORTSWORD_USES",
        label = "耐久次数",
        hover = "灵能短剑可以攻击多少次之后损毁。",
        options = {
            { description = "短（100）",       data = 100 },
            { description = "标准（150，默认）", data = 150 },
            { description = "长（250）",       data = 250 },
            { description = "无限（9999）",    data = 9999 },
        },
        default = 150,
    },
    {
        name = "PSIONIC_SHORTSWORD_SANITY_COST",
        label = "命中理智消耗",
        hover = "每次命中时，施法者扣多少理智。",
        options = {
            { description = "不消耗（0）",   data = 0 },
            { description = "1（默认）",     data = 1 },
            { description = "3",             data = 3 },
            { description = "5（高强度）",   data = 5 },
        },
        default = 1,
    },
}
```

#### 第三步：在 `modmain.lua` 里读配置并覆盖 TUNING

现在你 **不能再硬编码** `TUNING.PSIONIC_SHORTSWORD_DAMAGE = 38` 了——应该改成：

```lua
-- modmain.lua —— 读取配置项
TUNING.PSIONIC_SHORTSWORD_DAMAGE = GetModConfigData("PSIONIC_SHORTSWORD_DAMAGE") or 38
TUNING.PSIONIC_SHORTSWORD_USES = GetModConfigData("PSIONIC_SHORTSWORD_USES") or 150
TUNING.PSIONIC_SHORTSWORD_SANITY_COST = GetModConfigData("PSIONIC_SHORTSWORD_SANITY_COST") or 1
```

**为什么要 `or 38` 兜底？** `GetModConfigData` 在**老存档**或**配置项被删除**的场景下可能返回 `nil`。用 `or 默认值` 兜底是一种防御性写法，让 Mod 在各种情况下都不会崩溃。

#### 第四步：配置项生效的流程

```
玩家点击 Mod 管理 → 打开"配置 Mod" → 选了"高伤害"
  │
  ├── 游戏把选项写到 modconfiguration_<modname>.lua
  │
  ├── 玩家点击"启用并重启游戏"
  │
  ├── 游戏重启、加载 Mod
  │     ├── 读 modinfo.lua → 加载 configuration_options 的当前值
  │     ├── 执行 modmain.lua
  │     │     └── GetModConfigData("PSIONIC_SHORTSWORD_DAMAGE") 返回 50
  │     │     └── TUNING.PSIONIC_SHORTSWORD_DAMAGE = 50
  │     │
  │     └── 加载 scripts/prefabs/psionic_shortsword.lua
  │           └── 执行工厂函数时 weapon:SetDamage(TUNING.PSIONIC_SHORTSWORD_DAMAGE)
  │               → 此时拿到的是 50！
  │
  └── 玩家造出的每把短剑都是 50 伤害
```

**关键点**：`TUNING` 的修改必须**在 Prefab 工厂函数被执行之前**完成。`modmain.lua` 比 Prefab 文件早执行，所以在 `modmain.lua` 里改 `TUNING` 是**时机安全**的。如果你在某个运行时事件里改 `TUNING`（比如玩家切换了武器），**已经创建的物品不会生效**——它已经被 `SetDamage` 固定了伤害值。

#### 第五步：运行时改数值的办法

万一你真的要在游戏中途动态调整伤害（比如"每夜晚自动变强"），应该**直接改组件，不改 TUNING**：

```lua
-- 假设要让它在夜里伤害 +10
inst:WatchWorldState("phase", function(inst, phase)
    if phase == "night" then
        inst.components.weapon:SetDamage(TUNING.PSIONIC_SHORTSWORD_DAMAGE + 10)
    else
        inst.components.weapon:SetDamage(TUNING.PSIONIC_SHORTSWORD_DAMAGE)
    end
end)
```

`WatchWorldState` 是监听世界状态变化的接口，`phase` 就是"day / dusk / night"。这是一个**组件即时修改**的写法，比改 TUNING 可靠得多。

> **开发逻辑**：Mod 配置项有两层设计——**静态数值调参**（`GetModConfigData` + 改 `TUNING`）用于玩家在启动前定制；**动态运行时调整**（直接改组件）用于游戏进行中的机制。两者各有适用场景，不要混用。

---

### 5.7.8 老手进阶：`AddPrefabPostInit` —— 不创建新 Prefab 也能改原版

到这里灵能短剑已经是完整的 Mod 了。但是 Mod 开发还有另一条路线：**魔改原版 Prefab**。

假设你**不想造一把新武器**，而是想把**原版长矛**的伤害从 34 改成 50——怎么办？这就是 `AddPrefabPostInit` 的用武之地。

#### 第一步：`AddPrefabPostInit` 的定义

看 `scripts/modutil.lua` 第 592-599 行：

```592:599:scripts/modutil.lua
	env.postinitfns.PrefabPostInit = {}
	env.AddPrefabPostInit = function(prefab, fn)
		initprint("AddPrefabPostInit", prefab)
		if env.postinitfns.PrefabPostInit[prefab] == nil then
			env.postinitfns.PrefabPostInit[prefab] = {}
		end
		table.insert(env.postinitfns.PrefabPostInit[prefab], fn)
	end
```

它做的事情很简单：把你传进来的 `fn` 加到 `postinitfns.PrefabPostInit[prefab]` 表里——后续会被 `RegisterPrefabsImpl`（5.1.6 节）收集到 `modprefabinitfns[prefab]`，在每次 `SpawnPrefab` 后自动执行（5.1.5 节的 `SpawnPrefabFromSim`）。

**简单说**：你注册的回调会**在每次 `SpawnPrefab("spear")` 的工厂函数执行完之后**被调用，参数是刚创建好的 `inst`。

#### 第二步：一个简单的例子——让长矛伤害翻倍

在 `modmain.lua` 里加一行：

```lua
AddPrefabPostInit("spear", function(inst)
    if not TheWorld.ismastersim then
        return
    end

    if inst.components.weapon ~= nil then
        inst.components.weapon:SetDamage(TUNING.SPEAR_DAMAGE * 2)
    end
end)
```

**三个要点**：

1. **回调会在客户端和服务端都执行一次**——因为 `SpawnPrefabFromSim` 是两端各走一次。所以里面要**自己判断 `ismastersim`**，组件类的修改只在服务端做
2. **判断 `inst.components.weapon ~= nil`**——防御性：万一 Klei 哪天改了 spear 的组件结构，组件不存在时你的修改会崩，不如跳过
3. **直接改组件**——不要再改 `TUNING.SPEAR_DAMAGE`，因为原版长矛工厂函数已经用 34 初始化过一次了

#### 第三步：更常见的场景——加新组件、加标签、监听事件

PostInit 最常用的场景是**给原版 Prefab 加东西**。看几个实战案例：

**案例 A：给所有长矛加一个"命中后 30% 概率触发流血"效果**

```lua
AddPrefabPostInit("spear", function(inst)
    if not TheWorld.ismastersim then return end

    if inst.components.weapon ~= nil then
        local old_onattack = inst.components.weapon.onattack  -- 保留原版回调（如有）
        inst.components.weapon:SetOnAttack(function(weapon, attacker, target)
            if old_onattack ~= nil then
                old_onattack(weapon, attacker, target)
            end

            if target ~= nil and target:IsValid() and math.random() < 0.3 then
                if target.components.debuffable ~= nil then
                    target.components.debuffable:AddDebuff("bleed_debuff", "bleed_debuff")
                end
            end
        end)
    end
end)
```

**这里的关键技巧**：**保留原版回调**（`old_onattack`），再**在它之后追加**你的新逻辑。这样你不会破坏原版行为，又能叠加新效果——这是 Mod 兼容性最佳实践。

**案例 B：给所有蜘蛛加一个"看到玩家就自动标记 mark 2 秒"**

```lua
AddPrefabPostInit("spider", function(inst)
    if not TheWorld.ismastersim then return end

    inst:ListenForEvent("newcombattarget", function(inst, data)
        if data.target ~= nil and data.target:HasTag("player") then
            inst:AddTag("alerted_by_mod")
            inst:DoTaskInTime(2, function() inst:RemoveTag("alerted_by_mod") end)
        end
    end)
end)
```

**案例 C：给所有合成出来的物品加一个标记（用 `AddPrefabPostInitAny`）**

```lua
AddPrefabPostInitAny(function(inst)
    if inst.prefab ~= nil and inst:HasTag("weapon") then
        inst:AddTag("my_mod_weapon")
    end
end)
```

`AddPrefabPostInitAny`（回顾 5.1.5 节的源码）会在**每一个**被生成的 Prefab 上都执行一次——慎用，性能影响大。但某些场景（如"所有武器都加上某个 buff"）只能用它。

#### 第四步：PostInit 的时机回顾

回顾 5.1.5 节 `SpawnPrefabFromSim` 的执行顺序：

```
1. prefab.fn(TheSim)           ← 执行原版工厂函数（长矛自己的 fn）
2. inst:SetPrefabName(...)
3. modprefabinitfns[prefab]    ← 执行所有 Mod 的 PrefabPostInit
4. ModManager:GetPostInitFns("PrefabPostInitAny")  ← 执行所有 PrefabPostInitAny
5. TheWorld:PushEvent("entity_spawned", inst)
```

所以 PostInit 拿到的 `inst` 是**已经完全初始化的原版实体**——所有组件、标签都到位了。你的修改会在它之上叠加。

**容易犯的错**：有 Mod 开发者在 PostInit 里写 `inst:AddComponent("xxx")` 不检查 `inst.components.xxx` 是否已存在——如果原版已经有这个组件，再加一次会**覆盖**原版的配置。正确写法：

```lua
AddPrefabPostInit("spear", function(inst)
    if not TheWorld.ismastersim then return end

    -- 错误：直接 AddComponent 会覆盖原版
    -- inst:AddComponent("waterproofer")
    -- inst.components.waterproofer:SetEffectiveness(1)

    -- 正确：先判断
    if inst.components.waterproofer == nil then
        inst:AddComponent("waterproofer")
        inst.components.waterproofer:SetEffectiveness(1)
    end
end)
```

#### 第五步：什么时候用 PostInit，什么时候创建新 Prefab？

这是老手最常纠结的问题。**经验法则**：

| 场景 | 推荐做法 |
|------|---------|
| 调整原版物品的**数值**（伤害、耐久、速度） | `AddPrefabPostInit` + 改组件 |
| 给原版物品**加新功能**（新组件、新事件监听） | `AddPrefabPostInit` |
| 造一个**视觉/行为明显不同**的新物品 | 新建 Prefab |
| "魔改风"——想把长矛变成完全不同的东西 | 新建 Prefab（保留原版 spear 不变，玩家可自由选择） |
| 扩展**所有**同类物品（如"所有武器 +10% 伤害"） | `AddPrefabPostInitAny` + `inst:HasTag("weapon")` 过滤 |

> **开发逻辑**：PostInit 是一种"**非侵入式**"改造——你不破坏原版代码，只在其之上挂载修改。这让你的 Mod 和其他 Mod 更容易兼容（因为大家都在同一 Prefab 上叠加，只要不互相冲突就共存）。**新建 Prefab** 则是"**平行世界**"——你的新物品和原版独立，不干扰原版玩家。两条路线各有优势，好的 Mod 通常两种都用：**核心新内容用新 Prefab，配套的微调用 PostInit**。

---

### 5.7.9 常见坑与排查清单

Mod 开发最痛苦的不是写代码，而是"**为什么进游戏什么都看不到 / 控制台没报错却不工作**"。这一节汇总**我在写这个教程时亲手踩过的 10 个坑**，供你速查。

#### 坑 1：`c_spawn("psionic_shortsword")` 返回 `Can't find prefab`

**原因**：`modmain.lua` 的 `PrefabFiles` 里没加文件名。

**排查**：

```lua
-- modmain.lua 里必须有：
PrefabFiles = {
    "psionic_shortsword",   -- ← 必须和 scripts/prefabs/xxx.lua 的文件名一致
}
```

**相关章节**：5.6.8 坑 5。

---

#### 坑 2：物品栏图标是紫底红叹号

**原因**：三种情况之一——

1. 忘了写 `Asset("ATLAS", ...)` 和 `Asset("IMAGE", ...)` 中的任意一个
2. 文件名大小写不对（Linux 敏感）
3. `RegisterInventoryItemAtlas` 没调用

**排查**：对照 5.7.5 节第三步的完整四行——**四行必须全写**。

---

#### 坑 3：名字显示 `MISSING NAME` / `MISSING DESCRIPTION`

**原因**：`STRINGS.NAMES.<PREFAB_UPPER>` 没写，或键名大小写错。

**排查**：键名必须**全大写**——`"psionic_shortsword"` → `"PSIONIC_SHORTSWORD"`。如果你写成 `STRINGS.NAMES.psionic_shortsword` 不会报错但也不会生效。

---

#### 坑 4：装备上之后玩家手里什么都没有

**原因**：`onequip` 里的 `OverrideSymbol` 参数不对。

**排查**：这三个参数必须对齐：

```lua
owner.AnimState:OverrideSymbol("swap_object", "swap_spear", "swap_spear")
--                              ↑              ↑             ↑
--                              挂载点          build 名       symbol 名
```

- 第一个是**挂载点**（`swap_object` 是 Klei 约定的手持物挂载点，几乎所有武器都用它）
- 第二个是**被替换用的 build 名**——这个 build 必须已加载（借用 `swap_spear` 时它是原版自带加载的）
- 第三个是**build 内的 symbol**（动画 zip 里的图层名，通常和 build 同名）

---

#### 坑 5：服务端一切正常，客户端看到的物品不对

**原因**：把应该放在 pristine 阶段的东西写到了服务端段落，或反过来。

**排查**：回顾 5.4 节——**客户端要识别的所有东西**（外观、tag）必须在 `SetPristine()` 之前；**纯逻辑**（组件、伤害数值）在 `SetPristine()` 之后。

常见错误：

```lua
-- ❌ 错：客户端看不到这个 tag
if not TheWorld.ismastersim then
    return inst
end
inst:AddTag("weapon")   -- ← 客户端不知道这是武器！

-- ✅ 对
inst:AddTag("weapon")
inst.entity:SetPristine()
if not TheWorld.ismastersim then
    return inst
end
```

---

#### 坑 6：合成菜单里看不到配方

**原因**：三种可能——

1. `AddRecipe2` 漏写或写错 Prefab 名
2. 当前不在要求的科技工作台附近（需要炼金引擎 `TECH.SCIENCE_TWO`）
3. 配方还没解锁，要先在有科学机器时手动触发一次

**排查**：开启控制台模式 `c_unlockrecipe("psionic_shortsword")` 强制解锁，如果还看不到，就是 `AddRecipe2` 没注册成功——看游戏日志（`client_log.txt`）找 `AddRecipe2 psionic_shortsword` 这行。

---

#### 坑 7：材料图标显示不出（Mod 物品作为材料时）

**原因**：`Ingredient` 的第三个参数 `atlas` 没写。

**排查**：

```lua
-- ❌ 错：如果 my_mod_item 是 Mod 物品，图标显示不出
Ingredient("my_mod_item", 1)

-- ✅ 对：显式指定图集
Ingredient("my_mod_item", 1, "images/inventoryimages/my_mod_item.xml")
```

原版物品（`"goldnugget"`、`"log"`）不需要这个参数——引擎有内建查找表。

---

#### 坑 8：`onattack` 回调里 `target` 为 nil 导致崩溃

**原因**：目标可能在命中伤害结算过程中被销毁（例如低血量的怪被秒杀）。

**排查**：**永远先判断** `target ~= nil and target:IsValid()`：

```lua
local function onattack(weapon, attacker, target)
    if attacker ~= nil and attacker:IsValid()
        and target ~= nil and target:IsValid()
        and attacker.components.sanity ~= nil then
        attacker.components.sanity:DoDelta(-TUNING.PSIONIC_SHORTSWORD_SANITY_COST)
    end
end
```

看 `torch.lua` 第 207 行官方的类似写法：`if target ~= nil and target:IsValid() and target.components.burnable ~= nil and ...`。

---

#### 坑 9：配置项改了但游戏里没变

**原因**：两种情况——

1. 只改了 `modinfo.lua` 的 `default` 但没重新启动
2. 在 `modmain.lua` 里忘了把硬编码改成 `GetModConfigData`

**排查**：对照 5.7.7 节第三步——**所有 TUNING 赋值都要走 `GetModConfigData`**，而且 Mod 配置改动**必须重启游戏才能生效**。

---

#### 坑 10：联机时主机能合成短剑，客户端合成不出

**原因**：`modinfo.lua` 的 `all_clients_require_mod` 没设成 `true`。

**排查**：

```lua
-- modinfo.lua
all_clients_require_mod = true   -- 必须！加了新 Prefab 的 Mod 都要设
```

如果没设，主机启用 Mod 时**允许客户端没装这个 Mod 就连进来**，然后客户端本地没有 `anim/`、`images/` 的文件，一合成就崩溃。

---

#### 快速排查流程图

遇到问题先按这个顺序查：

```
Mod 在菜单里找不到
  ├─> 检查 modinfo.lua 的 name/version/api_version（必须 10）
  └─> 文件夹名是否不含中文/空格

c_spawn 报 Can't find prefab
  └─> PrefabFiles 有没有加这个名字

图标紫红色
  ├─> Asset 的 ATLAS + IMAGE 是否成对
  ├─> 文件是否真实存在（路径 + 大小写）
  └─> RegisterInventoryItemAtlas 是否调用

装备外观不对
  ├─> OverrideSymbol 的三个参数（swap_object + build + symbol）
  └─> 对应的 .zip 是否被加载（原版自带 or 自己声明 Asset）

合成菜单找不到
  ├─> AddRecipe2 是否正确执行（查日志）
  ├─> TECH 等级对不对
  └─> filters 分类是否合适

联机客户端看不到
  └─> all_clients_require_mod = true 必须设
```

---

### 5.7.10 小结

- **Mod 最小闭环**：目录 / `modinfo.lua` / `modmain.lua` / `scripts/prefabs/xxx.lua`——四件俱全，缺一不可。
- **Prefab 复用动画是新手的第一步**——`SetBank("spear") / SetBuild("swap_spear")` 让你在**不做美术**的情况下也能看到物品，把精力集中到代码逻辑上。
- **组件叠加 = 物品行为**：`weapon + finiteuses + equippable + inventoryitem + inspectable` 是武器类的标配五件套，和 `spear.lua` 完全一致。特殊效果（如命中扣 sanity）靠 `SetOnAttack` 回调实现。
- **STRINGS 的键名规则**：`NAMES.<PREFAB_UPPER>`、`CHARACTERS.GENERIC.DESCRIBE.<PREFAB_UPPER>`、`RECIPE_DESC.<PREFAB_UPPER>`——**全大写**是约定。
- **物品栏图标注册四件套**：`ATLAS` + `IMAGE` + `ATLAS_BUILD` + `RegisterInventoryItemAtlas`，再加上 `inventoryitem.imagename/atlasname` 告诉组件用哪张图。
- **`AddRecipe2` 参数 5 件套**：prefab 名、`Ingredient` 数组、`TECH` 科技等级、`config`（atlas/image 等）、`filters`（武器/魔法等分类）。合理的分类能大幅提升玩家体验。
- **`modinfo.configuration_options`** + `GetModConfigData` + 修改 `TUNING` 是**让玩家定制数值**的标准链路；`TUNING` 必须在 Prefab 工厂函数执行**之前**改，否则不会生效。
- **`AddPrefabPostInit`** 是**不创建新 Prefab 也能魔改原版**的方法——它会在每次 `SpawnPrefab` 的原工厂函数之后被调用。**写 PostInit 时三要素**：判断 `ismastersim`、判断组件存在、保留原版回调再追加。
- **PostInit vs 新 Prefab**：微调用 PostInit（非侵入），新机制用新 Prefab（独立）——好 Mod 通常两种都用。
- **十大排查坑**：PrefabFiles 漏登记、ATLAS+IMAGE 没成对、STRINGS 键名大小写、onequip 的 symbol 参数、pristine 分界误用、合成菜单配方不显示、材料图集缺失、`onattack` 目标 nil、配置项没走 `GetModConfigData`、`all_clients_require_mod` 没设。

> **本章收尾**：从 5.1 的"Prefab 是图纸，实体是成品"，到 5.7 的"一把完整可玩的灵能短剑"——你已经掌握了饥荒联机版中**所有"可创造的东西"**的底层机制。Chapter 6 我们将离开 Prefab 视角，进入**组件系统（Component System）**——"组件是什么、怎么自定义一个组件、replica 到底怎么工作"——那是 Mod 开发进阶的下一大关。