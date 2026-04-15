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

（待编写）

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
