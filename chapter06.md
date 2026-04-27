# 第6章 组件系统详解

## 6.1 组件的生命周期：Add → OnLoad → OnEntitySleep/Wake → OnRemoveFromEntity

### 本节导读

第 5 章我们把 **Prefab**（图纸）和 **实体**（成品）学透了。你可能已经注意到：同样是 `inst:AddComponent("weapon")`，写下去这一行字之后，物品就**自动会打伤害、自动消耗耐久、自动在存档里保存剩余使用次数**——这些**"自动"**的背后就是**组件的生命周期**。

这一节我们把组件从"出生到死亡"的完整链路拆开讲清楚——**它是在什么时候被加载的？怎么和实体绑定？存档时谁来保存它？玩家走远了组件会干嘛？最后销毁时又会做什么？**

> **新手**从 6.1.1-6.1.3 开始，只要理解「组件是一种可以"挂"到实体上的能力模块」「`AddComponent` 会触发一系列注册动作」「五个生命周期钩子分别在什么时机被调用」，就能看懂任何组件的骨架；**进阶读者**从 6.1.4-6.1.6 深入 `OnSave / OnLoad / OnEntitySleep / OnRemoveFromEntity` 的实际用法，结合 `health` / `perishable` / `combat` / `inventoryitemmoisture` 等官方组件的源码理解每个钩子的设计初衷；**老手**可以跳到 6.1.7 看完整生命周期流程图，以及 6.1.8 的 `AddComponentPostInit` 机制和 5 个常见陷阱——那是写自定义组件时最容易栽跟头的地方。

---

### 6.1.1 快速入门：组件是什么？—— 从"长矛有伤害"说起

先看一个直观场景。第 5 章里我们的灵能短剑（`psionic_shortsword.lua`）里有这两行：

```lua
inst:AddComponent("weapon")
inst.components.weapon:SetDamage(TUNING.PSIONIC_SHORTSWORD_DAMAGE)
```

**第一行**：往实体上挂了一个 `weapon`（武器）组件。**第二行**：设置这个组件的伤害值为 38。

神奇的是——你没有写任何「攻击判定」「伤害结算」「耐久扣减」「挥砍动画触发」的代码，但灵能短剑在游戏里就是**能正常当武器用**。这就是**组件**的威力。

**组件的本质**：一个封装好的**"能力模块"**，挂到实体上后，实体就拥有了这个能力。

可以用一个比喻：
- 实体 = 一辆车的**车架**（光秃秃、什么都干不了）
- 组件 = 车架上装的**一个个零件**（发动机、车轮、座椅、音响）
- 每装一个零件，车就多一项能力（能跑、有座、能听歌）

饥荒联机版的 `scripts/components/` 目录下**大约有 350 个组件文件**——每一个 `.lua` 文件就是一个能力模块。常见的：

| 组件 | 作用 |
|------|------|
| `health` | 血量、死亡、受伤 |
| `combat` | 战斗行为、目标选择、伤害结算 |
| `weapon` | 作为武器的属性（伤害、攻击范围、射程） |
| `inventoryitem` | 可以被拾起、放进背包 |
| `stackable` | 可以堆叠 |
| `equippable` | 可以被装备 |
| `burnable` | 可以被点燃 |
| `fueled` | 可以消耗燃料（火把、营火） |
| `perishable` | 会变质（食物） |
| `edible` | 可以被吃 |
| `inspectable` | 可以被检查 |

每个组件都是一个 **Lua Class**，有自己的状态字段（如 `currenthealth`）和方法（如 `DoDelta`）。当你 `AddComponent("health")` 后，这个 Class 的一个**实例**被创建出来，挂在 `inst.components.health`。

看 `health.lua` 第 62 行的类定义：

```62:80:scripts/components/health.lua
local Health = Class(function(self, inst)
    self.inst = inst
    self.maxhealth = 100
    self.minhealth = 0
    self.currenthealth = self.maxhealth
    self.invincible = false

    --V2C: Not used at all, but who knows about MODs?
    --     Save memory instead by making nil default to true
    --self.vulnerabletoheatdamage = true
    -----

    self.takingfiredamage = false
    self.takingfiredamagetime = 0
    --self.takingfiredamagelow = nil
	--self.lunarburns = nil
	--self.lunarburnflags = nil
	--self.lastlunarburnpulsetick = nil
    self.fire_damage_scale = 1
```

`Health` 类的**构造函数**接收一个参数 `inst`——也就是"这个组件将要挂到哪个实体上"。构造函数里把默认值（最大血量 100、当前血量 100、不无敌等）初始化好——这就是组件被创建时的**初始状态**。

> **新手记忆**：**组件就是一个 Class，`AddComponent` 就是 new 一个这个 Class 的实例并挂到实体上**。实体是容器，组件是内容——两者合起来才是完整的游戏对象。

---

### 6.1.2 快速入门：`AddComponent` 到底做了什么？

当你写 `inst:AddComponent("weapon")` 时，引擎在幕后做了一整套流程。看 `scripts/entityscript.lua` 第 610-646 行的完整代码：

```610:646:scripts/entityscript.lua
function EntityScript:AddComponent(name)
    local lower_name = string.lower(name)
    if self.lower_components_shadow[lower_name] ~= nil then
        print("component "..name.." already exists on entity "..tostring(self).."!"..debugstack_oneline(3))
		local existingcmp = self.components[lower_name]
		if existingcmp == nil then
			for k, v in pairs(self.components) do
				if string.lower(k) == lower_name then
					existingcmp = v
					break
				end
			end
		end
		return existingcmp
    end

    local cmp = LoadComponent(name)
	if not cmp then
	    error("component ".. name .. " does not exist!")
	end

    self:ReplicateComponent(name)

    local loadedcmp = cmp(self)
    self.components[name] = loadedcmp
    self.lower_components_shadow[lower_name] = true

    local postinitfns = ModManager:GetPostInitFns("ComponentPostInit", name)

    for _, fn in ipairs(postinitfns) do
        fn(loadedcmp, self)
    end

    self:RegisterComponentActions(name)

	return loadedcmp
end
```

这段代码看起来长，其实做的事可以拆成**六步**：

#### 第一步：重复检查

```lua
if self.lower_components_shadow[lower_name] ~= nil then
    print("component "..name.." already exists on entity "..tostring(self).."!"
```

**同一个实体不允许重复加同一个组件**。如果你不小心写了两次 `inst:AddComponent("health")`——第二次会**直接返回已存在的那个组件**，并在控制台打印警告。

> **开发逻辑**：Klei 选择不报错崩溃、而是返回已有实例，是为了**防御 Mod 之间互相覆盖**。如果 A Mod 加了 health，B Mod 的 PostInit 又想加一次，这段代码让 B Mod 拿到 A 已加的那个实例，两个 Mod 可以继续协作。

#### 第二步：加载组件 Class

```lua
local cmp = LoadComponent(name)
```

对应 `entityscript.lua` 第 52-60 行：

```52:60:scripts/entityscript.lua
local function LoadComponent(name)
    if Components[name] == nil then
        Components[name] = require("components/"..name)
        assert(Components[name], "could not load component "..name)
        Components[name].WatchWorldState = ComponentWatchWorldState
        Components[name].StopWatchingWorldState = ComponentStopWatchingWorldState
    end
    return Components[name]
end
```

**三件事**：

1. **延迟加载 + 缓存**：第一次用 `weapon` 组件时才 `require("components/weapon")`，加载后缓存到 `Components["weapon"]` 里。后续再加就不用重复加载了
2. **全局容器 `Components`**：所有被用到的组件 Class 都放在这里
3. **注入 `WatchWorldState` 方法**：这两个是引擎给每个组件"免费"提供的全局能力——让组件可以监听世界状态（天气、季节、日夜）

这就是为什么你**第一次 `SpawnPrefab("flower")` 时可能感觉游戏卡了一下**——它在第一次运行 `flower.lua` 时触发了一连串组件的加载。

#### 第三步：创建 Replica（客户端影子）

```lua
self:ReplicateComponent(name)
```

**这行代码是联机版的灵魂**。我们在 5.4 节讲过 replica 系统——**服务端加了哪些需要同步的组件，客户端就要有对应的"影子"来接收同步数据**。

`ReplicateComponent` 会查一张表（定义在 `entityreplica.lua`），如果这个组件在"需要同步"的列表里（如 `health`、`combat`、`inventoryitem`），就在客户端的 `inst.replica` 下挂一个对应的简化版组件。不在列表里的（如 `fueled`、`perishable`）就跳过。

> **开发逻辑**：不是每个组件都要同步——只有**客户端需要显示/决策的**才需要。`fueled`（火把剩多少燃料）客户端能看到 section 变化就够了，不用知道精确的剩余秒数，所以 `fueled` 不需要 replica。

**这一步的后果**：**`AddComponent` 一行代码就已经完成了联机同步的底层准备**，你不用自己写任何网络代码。

#### 第四步：真正实例化

```lua
local loadedcmp = cmp(self)
self.components[name] = loadedcmp
self.lower_components_shadow[lower_name] = true
```

- **`cmp(self)`**：调用 Class 的构造函数——把**自己**（实体 `inst`）作为参数传进去。这就是为什么 `Health` 的 `self.inst = inst`，组件始终能通过 `self.inst` 找到挂载它的实体
- **`self.components[name] = loadedcmp`**：把新创建的组件实例**挂到实体的 `components` 表上**——这就是为什么你能用 `inst.components.weapon` 访问它
- **`lower_components_shadow`**：一个小写索引的镜像表，用来在重复检测时跳过大小写问题

#### 第五步：执行 Mod 的 `ComponentPostInit`

```lua
local postinitfns = ModManager:GetPostInitFns("ComponentPostInit", name)

for _, fn in ipairs(postinitfns) do
    fn(loadedcmp, self)
end
```

这是 **Mod 魔改组件的入口**——在组件实例创建好之后，所有 Mod 注册过 `AddComponentPostInit("weapon", fn)` 的回调都会被调用，每个回调拿到刚创建的组件实例和实体。

我们会在 6.1.8 节详细讲这个机制。

#### 第六步：注册组件动作

```lua
self:RegisterComponentActions(name)
```

这和 **"右键点击物品弹出的菜单"** 相关——比如右键点草叉会出现"装备"选项，右键点食物会出现"吃"选项。每个组件可以通过 `AddComponentAction` 注册它想提供的动作。

`AddComponent` 时会触发一次注册，`RemoveComponent` 时会注销——保证动作菜单始终和组件状态一致。

#### 小结：一行 `AddComponent` 的完整效果

```
inst:AddComponent("weapon")
  │
  ├── 检查重复
  ├── LoadComponent("weapon")           ← require("components/weapon"), 缓存
  ├── ReplicateComponent("weapon")      ← 客户端创建 inst.replica.weapon
  ├── Weapon(inst)                      ← 调用构造函数，设置默认值
  ├── inst.components.weapon = <instance>
  ├── 执行 Mod 的 AddComponentPostInit  ← 可能被 Mod 改参数/加行为
  └── RegisterComponentActions          ← 加入右键菜单
```

> **新手记忆**：**`AddComponent` 不只是"添加一个字段"**——它完成了**加载、同步、实例化、Mod 注入、动作注册**五件事。这就是饥荒组件系统最强大的地方：你写一行代码，引擎替你做十件事。

---

### 6.1.3 快速入门：五个最核心的生命周期钩子

一个组件从被添加到实体上，到实体销毁这段时间里，会在几个**关键时刻**被引擎"叫回来"做事。这些时刻就是**生命周期钩子**（lifecycle hook）。

饥荒组件系统里**最常用的五个**钩子：

| 钩子 | 何时触发 | 常见用途 |
|------|---------|---------|
| `构造函数` | `AddComponent` 时 | 初始化字段 |
| `OnSave(self)` | 存档时 | 返回需要保存的数据表 |
| `OnLoad(self, data)` | 读档时 | 从数据表恢复状态 |
| `OnEntitySleep(self)` | 玩家离远了，实体休眠 | 停止耗性能的任务、解除引用 |
| `OnEntityWake(self)` | 玩家回来，实体唤醒 | 恢复任务、重建状态 |
| `OnRemoveFromEntity(self)` | 组件被移除（不是实体销毁） | 清理标签、取消 task、解除事件 |

**注意区分两个"移除"**：

- **`OnRemoveFromEntity`**：组件级事件，**只是把这个组件从实体上拆下来**（比如 `inst:RemoveComponent("burnable")` 不烧了，实体还在）
- **`OnRemoveEntity`**：实体级事件，**整个实体要销毁了**——组件也顺带销毁。它在 `entityscript.lua` 第 1705-1772 行的 `Remove` 里被调用（回顾 5.2.4 节）

这两个钩子在不同的生命周期阶段触发，用途也不一样。本节我们主要讲前者（`OnRemoveFromEntity`），后者在 5.2 讲过不再赘述。

#### 一个最简单的"全家福"例子

结合 `health.lua` 的几个钩子，把它们在一个组件里的样子看一遍：

```lua
local Health = Class(function(self, inst)
    -- 构造函数：设置默认值
    self.inst = inst
    self.maxhealth = 100
    self.currenthealth = 100
end)

function Health:OnSave()
    return {
        health = self.currenthealth,
        maxhealth = self.save_maxhealth and self.maxhealth or nil,
    }
end

function Health:OnLoad(data)
    if data.maxhealth ~= nil then
        self.maxhealth = data.maxhealth
    end
    if data.health ~= nil then
        self:SetVal(data.health, "file_load")
    end
end

-- health 组件没有 OnEntitySleep/Wake（血量不需要根据休眠做事）

function Health:OnRemoveFromEntity()
    self:StopRegen()  -- 清理：停止回血任务
    -- ...其他清理...
end
```

每个函数的职责都**清晰且单一**：

- 构造函数：**"我是谁，我的初始状态是什么"**
- `OnSave`：**"我要保存什么东西到存档里"**
- `OnLoad`：**"给我存档数据，我恢复自己"**
- `OnRemoveFromEntity`：**"我要被拆下来了，赶紧收拾烂摊子"**

> **新手记忆**：**写一个新组件时，先把这几个钩子的空架子摆上，再往里填内容**——这是老开发者的习惯。不写也不会崩（引擎会检查 `if v.OnSave then`），但一写就能享受到存档/休眠/清理的"免费能力"。

---

### 6.1.4 进阶：`OnSave` 与 `OnLoad` —— 存档的双向桥梁

饥荒联机版的存档是**每组件各存各的**——不是引擎"全盘序列化"整个实体。这让存档体积更小，也让 Mod 加的新组件能无缝参与存档。

#### 第一步：`OnSave` 的调用时机

看 `entityscript.lua` 第 1903-1937 行的 `GetPersistData`：

```1903:1937:scripts/entityscript.lua
function EntityScript:GetPersistData()
    local references = {}
    local data = {}
    for k, v in pairs(self.components) do
        if v.OnSave then
            local t, refs = v:OnSave()
            if type(t) == "table" and not IsTableEmpty(t) then
                data[k] = t
				if t.add_component_if_missing then
					data.add_component_if_missing = true
				end
            end

            if refs then
                for k1, v1 in pairs(refs) do
                    table.insert(references, v1)
                end
            end
        end
    end

    if self.OnSave then
        local refs = self.OnSave(self, data)

        if refs then
            for k, v in pairs(refs) do
                table.insert(references, v)
            end
        end
    end

    if not IsTableEmpty(data) or not IsTableEmpty(references) then
        return data, references
    end
end
```

**执行顺序**：

1. 遍历 `self.components` 里的**每一个组件**
2. 如果这个组件有 `OnSave` 方法，**就调用它**
3. 组件 `OnSave` 返回一张**表**，里面是这个组件要保存的数据——引擎把它存到 `data[组件名]` 下
4. **所有组件都 OnSave 完**之后，再调用实体级的 `self.OnSave(self, data)`——让 Prefab 能追加自己的散装数据（5.2.4 节讲过）

#### 第二步：`OnSave` 的常见写法

看三个典型组件的 `OnSave`：

**① `health.lua`（第 154-161 行）**：

```154:161:scripts/components/health.lua
function Health:OnSave()
    return
    {
        health = self.currenthealth,
        penalty = self.penalty > 0 and self.penalty or nil,
		maxhealth = self.save_maxhealth and self.maxhealth or nil
    }
end
```

**只保存必须保存的东西**——`currenthealth` 必存，`penalty`（死亡惩罚）只在 > 0 时才存，`maxhealth` 只在特殊标记时存。

**② `fueled.lua`（第 133-137 行）**：

```133:137:scripts/components/fueled.lua
function Fueled:OnSave()
    if self.currentfuel ~= self.maxfuel then
        return {fuel = self.currentfuel}
    end
end
```

**只在非默认状态下才保存**——如果燃料还是满的（`currentfuel == maxfuel`），**返回 nil** 什么都不存。这样空火把的数据根本不进存档，节省空间。

**③ `perishable.lua`（第 331-337 行）**：

```331:337:scripts/components/perishable.lua
function Perishable:OnSave()
    return
    {
        paused = self.updatetask == nil or nil,
        time = self.perishremainingtime,
    }
end
```

**保存状态 + 剩余时间**——`paused` 标记这个东西是不是停止了腐烂进度，`time` 是还剩多久腐烂。

#### 第三步：`OnSave` 的几个"潜规则"

从源码里能提炼出几条**经验法则**：

1. **`OnSave` 返回 nil → 不存**：引擎判断 `type(t) == "table" and not IsTableEmpty(t)`——返回 nil 或空表都会被跳过
2. **只存"必要的"字段**：不要整个 `self` dump——会让存档膨胀
3. **默认值不存**：如 fueled 的"满燃料"不存——读档时组件用默认值即可
4. **不存可重新计算的字段**：比如 `self.updatetask`（定时任务）——读档时 `OnLoad` 里根据当前状态重启就行
5. **不存函数引用**：Lua 函数不能序列化，存了也读不回来

> **开发逻辑**：为什么 Klei 这么抠门？因为一个饥荒世界存档经常包含**几千个实体**——每个实体几十 KB 的话，存档文件立刻就上百 MB 了。Klei 的设计哲学是"**可推断的不保存，可默认的不保存**"——让存档尽量精简。你写自定义组件时**一定要遵守这个原则**。

#### 第四步：`OnLoad` 的调用时机

`OnLoad` 在 `SetPersistData` 里被调用。看 `entityscript.lua` 第 1954-1984 行：

```1954:1984:scripts/entityscript.lua
function EntityScript:SetPersistData(data, newents)
	if data and data.add_component_if_missing then
		for k, v in pairs(data) do
			if self.components[k] == nil and type(v) == "table" and v.add_component_if_missing then
				self:AddComponent(k)
			end
		end
	end

    if self.OnPreLoad ~= nil then
        self:OnPreLoad(data, newents)
    end

    if data ~= nil then
        for k, v in pairs(data) do
            local cmp = self.components[k]

			--V2C: -backward compatibility for add_component_if_missing
			--     -the flag has since been added to the base data table as well so that we
			--      can add the missing components before preload
            if cmp == nil and type(v) == "table" and v.add_component_if_missing then
				self:AddComponent(k)
                cmp = self.components[k]
            end
			------

            if cmp ~= nil and cmp.OnLoad ~= nil then
                cmp:OnLoad(v, newents)
            end
        end
    end
```

**五步流程**：

1. **`add_component_if_missing` 检查**：如果存档数据里标记"这个组件存档时有，但实体工厂函数没 Add"——**引擎会自动补加这个组件**
2. **`OnPreLoad` 先执行**：实体级的钩子，可以在组件 `OnLoad` 之前做调整
3. **遍历存档数据**：对每个 `data[组件名]`，找到实体上的对应组件
4. **兜底补加**：如果组件不存在但数据里有 `add_component_if_missing` 标记，再补加一次
5. **调用组件的 `OnLoad(v, newents)`**：把存档数据喂给组件

**注意参数 `newents`**：它是一张表——新生成的实体 GUID 映射到实体对象。用来解决"**实体之间的交叉引用**"——比如一个容器实体要引用它里面的物品实体，读档时需要通过 `newents` 找到那些物品。

#### 第五步：`OnLoad` 的常见写法

看 `perishable.lua` 第 339-351 行：

```339:351:scripts/components/perishable.lua
function Perishable:OnLoad(data)
    if data ~= nil then
		if data.time ~= nil then
	        self.perishremainingtime = data.time
		end

        if data.paused then
            self:StopPerishing()
		elseif data.time ~= nil then
            self:StartPerishing()
        end
    end
end
```

**关键写法**：

- **先判断 `data ~= nil`**——存档可能没这个组件的数据
- **逐个字段判断 nil**——你可能在新版 Mod 里加了字段，老存档里没有，直接用会崩
- **对应动作要重启**——如 `StartPerishing` 重启计时任务（你在 `OnSave` 里没存 task，现在根据状态恢复它）

**对比 `health.lua` 的 `OnLoad`（第 163-190 行）**，你会看到类似的防御式写法：先 `if data.maxhealth ~= nil`，再 `if data.health ~= nil`，最后才是各种条件分支。

> **开发逻辑**：`OnLoad` 的防御式检查是**为了向前兼容**。老存档可能没有新字段，新版 Mod 可能加了字段——每个字段都要 nil 检查，才能让**老存档升级到新版 Mod 时不崩溃**。这是一种非常重要的 Mod 开发思维。

---

### 6.1.5 进阶：`OnEntitySleep` 与 `OnEntityWake` —— 性能优化的关键

#### 第一步：什么是"实体休眠"？

饥荒联机版做了一个重要的优化——**离玩家很远的实体会"冻结"**，减少 CPU 占用。这就是**实体休眠**（Entity Sleep）机制。

想象你在地图左下角，而地图右上角有 100 只小蜘蛛在它们的巢穴外闲逛——如果每一只都在每一帧计算 AI、重选目标、更新动画状态，游戏会卡到飞起。Klei 的方案是：

- 玩家周围**一定距离内**的实体——**正常运行**
- 距离外的实体——**进入休眠**（stop update、stop brain、stop stategraph）

看 `mainfunctions.lua` 第 670-697 行的 `OnEntitySleep` 函数：

```670:696:scripts/mainfunctions.lua
function OnEntitySleep(guid)
    local inst = Ents[guid]
    if inst then
        inst:PushEvent("entitysleep")

        if inst.OnEntitySleep then
            inst:OnEntitySleep()
        end

		inst:_DisableBrain_Internal()

        if inst.sg then
            SGManager:Hibernate(inst.sg)
        end

		inst.sleepstatepending = nil


        for k,v in pairs(inst.components) do
            if v.OnEntitySleep then
                v:OnEntitySleep()
            end
        end
    end
end
```

**执行顺序**：

1. `PushEvent("entitysleep")`——触发事件（有事件监听者可以响应）
2. 实体级的 `inst:OnEntitySleep()` 先执行（5.2.4 节讲过）
3. **关停 Brain**（AI 不再决策）
4. **StateGraph 休眠**（动画状态机停在当前状态）
5. **对每个组件调用 `OnEntitySleep()`**——组件自己决定要不要清理些什么

对应的 `OnEntityWake` 在第 699-731 行，几乎是反向流程。

#### 第二步：组件应该在 `OnEntitySleep` 里干什么？

核心原则：**取消所有"耗 CPU 的定时任务"和"容易过期的引用"**。

看 `combat.lua` 第 284-289 行：

```284:289:scripts/components/combat.lua
function Combat:OnEntitySleep()
    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end
end
```

**`combat` 组件**维护一个"每隔几秒重新选目标"的定时任务 `retargettask`。休眠时**取消这个任务**——离玩家远的蜘蛛没人打架，没必要再持续找目标。

#### 第三步：对应的 `OnEntityWake` 要恢复这些任务

看同文件第 291-301 行：

```291:301:scripts/components/combat.lua
function Combat:OnEntityWake()
    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end

    if self.retargetperiod ~= nil then
        self.retargettask = self.inst:DoPeriodicTask(self.retargetperiod, dotryretarget, self.retargetperiod*math.random(), self)
    end

    if self.target ~= nil and self.keeptargetfn ~= nil then
```

**唤醒时重新启动 retargettask**——注意第一行 `Cancel` 是防御性的（万一 `OnEntitySleep` 没清理干净），然后根据 `retargetperiod` 重启任务。

#### 第四步：更复杂的例子——`inventoryitemmoisture` 的时间补偿

看 `inventoryitemmoisture.lua` 第 91-113 行：

```91:113:scripts/components/inventoryitemmoisture.lua
function InventoryItemMoisture:OnEntitySleep()
	if self.moistureupdatetask then
		self.moistureupdatetask:Cancel()
		self.moistureupdatetask = nil
	end

	self._entitysleeptime = GetTime()
end

function InventoryItemMoisture:OnEntityWake()
	local updated
	if self._entitysleeptime then
		local time_slept = GetTime() - self._entitysleeptime
		if time_slept > 0 then
			updated = self:UpdateMoisture(time_slept)
		end
		self._entitysleeptime = nil
	end
	if self.moistureupdatetask == nil then
		self.moistureupdatetask = self.inst:DoPeriodicTask(updated and UPDATE_TIME or SLOW_UPDATE_TIME, DoUpdate, math.random() * UPDATE_TIME)
```

**这是一个非常典型的"休眠时间补偿"模式**：

- **`OnEntitySleep`**：记录下**休眠开始时间**（`self._entitysleeptime = GetTime()`）
- **`OnEntityWake`**：计算**休眠时长**（`GetTime() - _entitysleeptime`），然后一次性 `UpdateMoisture(time_slept)`——**把这段时间里"应该发生的变化"补上**

**为什么要这样？** 地上的一把湿草——休眠时没人更新它，但按"真实物理"它应该在慢慢变干。醒来后一次性补偿这段时间的变化，让玩家的体验不受休眠影响。

> **开发逻辑**：这种"**离散补偿**"是游戏开发的经典技巧——真实时间流逝可能是连续的，但出于性能考虑我们可以"跳过中间过程，直接更新最终状态"。**你写自定义组件时如果有时间相关的状态（腐烂、成长、充能），都应该考虑用这种模式**。

#### 第五步：哪些情况下组件不需要 `OnEntitySleep`？

看源码你会发现 **`health` 组件根本没有 `OnEntitySleep` / `OnEntityWake`**。为什么？

因为 `health` 是**被动的**——血量只在"被攻击"「被治疗」「回血回调」等**事件触发**时变化，组件自己没有定时任务在跑。休眠或唤醒对它的状态没影响——所以不需要这两个钩子。

**判断原则**：

- **有定时任务 / 心跳回调的组件** → **需要** `OnEntitySleep/Wake`
- **有"时间流逝自动变化"状态的组件** → **需要**（而且可能需要补偿）
- **纯被动、事件驱动的组件** → **不需要**

> **新手记忆**：**先判断你的组件有没有"在不断运行的定时任务"**——有就加 `OnEntitySleep/Wake`，没有就不加。盲目加反而可能带来 bug。

---

### 6.1.6 进阶：`OnRemoveFromEntity` —— 组件的"离场清理"

#### 第一步：什么时候组件会被"单独移除"？

最常见的场景是 `inst:RemoveComponent("xxx")`——**组件被拆下来，但实体还活着**。例如：

- 一把可燃的武器被施了"防火"buff → `RemoveComponent("burnable")`
- 玩家解除装备 → 某些装备特殊组件被临时移除
- 实体形态切换 → 旧形态的组件换成新形态的

这时候组件要"整理行囊"——停任务、解监听、清除 tag。看 `entityscript.lua` 第 648-663 行：

```648:663:scripts/entityscript.lua
function EntityScript:RemoveComponent(name)
    local cmp = self.components[name]
    if cmp then
        self:StopUpdatingComponent(cmp)
        self:StopWallUpdatingComponent(cmp)
        self.components[name] = nil
        self.lower_components_shadow[string.lower(name)] = nil

        if cmp.OnRemoveFromEntity then
            cmp:OnRemoveFromEntity()
        end

        self:UnreplicateComponent(name)
        self:UnregisterComponentActions(name)
    end
end
```

**执行顺序**：

1. 停止引擎层的 update / wallupdate（再也不会被 tick）
2. 从 `components` 表里删除
3. **调用 `OnRemoveFromEntity`**——组件自己清理
4. 取消 replica（客户端影子也删除）
5. 注销组件动作（右键菜单里这项消失）

#### 第二步：`OnRemoveFromEntity` 要清理什么？

看 `health.lua` 第 116-136 行：

```116:136:scripts/components/health.lua
function Health:OnRemoveFromEntity()
    self:StopRegen()

    if self.regensources ~= nil then
        for source, _ in pairs(self.regensources) do
            self:RemoveRegenSource(source)
        end
    end

	if self.lunarburns then
		for source in pairs(self.lunarburns) do
			if EntityScript.is_instance(source) then
				self.inst:RemoveEventCallback("onremove", self._onremovelunarburn, source)
			end
		end
		self.lunarburns = nil
		self.lunarburnflags = nil
		self.lastlunarburnpulsetick = nil
		self._onremovelunarburn = nil
		self.inst:PushEvent("stoplunarburn")
	end
```

**做了什么**：

- **`StopRegen()`**：停止回血的定时任务
- **`RemoveRegenSource`**：每个"回血源"（buff）都要解除
- **`RemoveEventCallback`**：解除所有事件监听——`lunarburns` 里的每个源都绑定了"onremove"事件，现在要解绑
- **清空字段**：`self.lunarburns = nil`、`self._onremovelunarburn = nil` 等——帮 GC 回收

#### 第三步：更简洁的例子——`burnable` 和 `fueled`

`burnable.lua` 第 593-604 行：

```593:604:scripts/components/burnable.lua
function Burnable:OnRemoveFromEntity()
    --self:StopSmoldering()
    --Extinguish() already calls StopSmoldering()
    self:Extinguish()
    if self.task ~= nil then
        self.task:Cancel()
        self.task = nil
    end
    self.inst:RemoveTag("canlight")
    self.inst:RemoveTag("nolight")
    self.inst:RemoveTag("burnableignorefuel")
end
```

**做了三件事**：

1. **`Extinguish`**——如果正在燃烧就先扑灭（避免"一个不会燃烧的物品还在产生火焰特效"）
2. **`self.task:Cancel`**——取消所有定时任务
3. **移除它之前加的 tag**——`"canlight"`、`"nolight"` 等是这个组件自己加的，离场时要清理掉

`fueled.lua` 第 116-125 行也是类似逻辑——**移除自己加的 tag**。

#### 第四步：为什么一定要清理 tag？

**因为 tag 是"公共资产"**。回顾 5.5 节，tag 是存在实体上的——**不是组件私有**。如果 `burnable` 加了 `"canlight"` tag，但组件被移除后 tag 没清理，其他模块（如"找可点燃目标"的搜索代码）会**误把这个已经不能点燃的实体当成可点燃的目标**。

**这是新手最容易踩的坑**：给组件加了 tag 但忘记在 `OnRemoveFromEntity` 里清理——导致"幽灵标签"污染游戏逻辑。

> **开发逻辑**：`OnRemoveFromEntity` 的本质是**"把你加的所有东西都拆回来"**——加了 tag 就移除 tag、加了监听就解除监听、开了定时任务就取消任务、持有了引用就置 nil。这是一种"**谁污染谁治理**"的编程习惯，让组件可以安全地"插拔"。

#### 第五步：实体销毁时组件会经过 `OnRemoveFromEntity` 吗？

**会**。看 `entityscript.lua` 的 `Remove` 函数，在实体销毁流程里会遍历所有组件并调用：

```lua
for k,v in pairs(self.components) do
    self:RemoveComponent(k)  -- 也会触发 OnRemoveFromEntity
end
```

这也是**为什么你在 `OnRemoveFromEntity` 里不用特意区分"是单独移除还是随实体销毁"**——两种情况都会走这个钩子，做同样的清理就行。

**唯一要小心的**：在 `OnRemoveFromEntity` 里**不要访问其他组件**——因为实体销毁时，其他组件可能已经被销毁了（顺序不确定）。如果必须访问，要先判断 `self.inst.components.xxx ~= nil`。

---

### 6.1.7 老手进阶：组件完整生命周期流程图

把前面所有钩子串起来，一个组件从"挂到实体上"到"离开实体"的完整时序如下：

```
┌─────────────────────────────────────────────────────────────┐
│  A. 诞生阶段                                                  │
├─────────────────────────────────────────────────────────────┤
│  inst:AddComponent("weapon")                                  │
│        ↓                                                      │
│  LoadComponent("weapon")                                      │
│    ├── require("components/weapon")                           │
│    └── Components["weapon"] = 缓存                            │
│        ↓                                                      │
│  ReplicateComponent("weapon")                                 │
│    └── 客户端创建 inst.replica.weapon                         │
│        ↓                                                      │
│  Weapon(self.inst) 构造函数                                   │
│    └── 初始化字段、默认值                                     │
│        ↓                                                      │
│  self.components.weapon = 实例                                │
│        ↓                                                      │
│  执行 Mod AddComponentPostInit("weapon", fn) 注册的回调       │
│        ↓                                                      │
│  RegisterComponentActions("weapon")                           │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  B. 存活阶段（可能反复进入 sleep / wake，可能存档/读档）        │
├─────────────────────────────────────────────────────────────┤
│                                                                │
│  正常运行（组件方法被各种事件调用）                            │
│        │                                                      │
│        ├→ 存档 → GetPersistData                              │
│        │     └── cmp:OnSave() → 返回数据表                   │
│        │                                                      │
│        ├→ 读档 → SetPersistData(data)                        │
│        │     ├── (可能) add_component_if_missing 补加组件    │
│        │     └── cmp:OnLoad(data) ← 恢复状态                 │
│        │                                                      │
│        ├→ 玩家走远 → OnEntitySleep                           │
│        │     ├── PushEvent("entitysleep")                    │
│        │     ├── inst:OnEntitySleep()                        │
│        │     ├── 禁用 Brain / SGManager:Hibernate            │
│        │     └── cmp:OnEntitySleep()                         │
│        │                                                      │
│        ├→ 玩家回来 → OnEntityWake                            │
│        │     ├── PushEvent("entitywake")                     │
│        │     ├── inst:OnEntityWake()                         │
│        │     ├── 启用 Brain / SGManager:Wake                 │
│        │     └── cmp:OnEntityWake()                          │
│        │                                                      │
│        └→ (可能) Mod 通过 AddComponentPostInit 后续修改       │
│                                                                │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  C. 离场阶段                                                  │
├─────────────────────────────────────────────────────────────┤
│  inst:RemoveComponent("weapon")  或  inst:Remove()            │
│        ↓                                                      │
│  StopUpdatingComponent / StopWallUpdatingComponent            │
│        ↓                                                      │
│  self.components.weapon = nil                                 │
│        ↓                                                      │
│  cmp:OnRemoveFromEntity()  ← 你的清理逻辑                    │
│    └── 停任务、解监听、移除 tag、nil 引用                     │
│        ↓                                                      │
│  UnreplicateComponent("weapon")                               │
│    └── 客户端的 inst.replica.weapon 也删除                   │
│        ↓                                                      │
│  UnregisterComponentActions("weapon")                         │
└─────────────────────────────────────────────────────────────┘
```

**关键时序洞察**：

1. **`AddComponent` 先 Replicate 再 new**：客户端的影子在**服务端实例化之前**就已经创建——保证 client/server 在 `PushEvent` 时同步
2. **`OnLoad` 可能触发 `AddComponent`**：存档里标记了但工厂函数没加的组件，会被自动补加（向前兼容机制）
3. **Sleep/Wake 是对称的**：你在 sleep 里做的事，通常在 wake 里要做反向操作——源码里 `combat`、`inventoryitemmoisture` 都是这种对称式写法
4. **销毁时组件先清理，实体再消失**：这保证你在 `OnRemoveFromEntity` 里引用 `self.inst` 始终有效

---

### 6.1.8 老手进阶：`AddComponentPostInit` 与五个常见陷阱

#### 第一步：`AddComponentPostInit` 的定义

看 `modutil.lua` 第 566-573 行：

```566:573:scripts/modutil.lua
	env.postinitfns.ComponentPostInit = {}
	env.AddComponentPostInit = function(component, fn)
		initprint("AddComponentPostInit", component)
		if env.postinitfns.ComponentPostInit[component] == nil then
			env.postinitfns.ComponentPostInit[component] = {}
		end
		table.insert(env.postinitfns.ComponentPostInit[component], fn)
	end
```

**做的事**：把你的回调存到 `env.postinitfns.ComponentPostInit[component]` 里——每次**任何实体** `AddComponent(component)` 时，这个回调会在构造函数之后执行。

#### 第二步：典型用法——"给所有血量组件加 'god mode' 开关"

```lua
-- modmain.lua
AddComponentPostInit("health", function(cmp, inst)
    if inst:HasTag("player") then
        cmp.__ModGodMode = false
        local old_DoDelta = cmp.DoDelta
        cmp.DoDelta = function(self, amount, overtime, cause, ...)
            if self.__ModGodMode and amount < 0 then
                return 0  -- 屏蔽所有扣血
            end
            return old_DoDelta(self, amount, overtime, cause, ...)
        end
    end
end)
```

**流程**：

1. 任何实体 `AddComponent("health")` 时，这个函数被调用，参数是新建的 `cmp` 和 `inst`
2. 我们只改动玩家的 health 组件（`HasTag("player")`）
3. 保留原版 `DoDelta` 方法（`old_DoDelta = cmp.DoDelta`），然后覆盖它
4. 新版 `DoDelta` 先检查 god mode 标记，不屏蔽的话再调用原版

**这种"**保留原版 + 包装**"的手法叫 method overriding**——在 Mod 开发中极其常见。

#### 第三步：五个常见陷阱

写 `AddComponentPostInit` 的坑非常多。这里列出我和其他 Mod 开发者最常踩的 5 个：

**陷阱 1：忘记判断 ismastersim**

```lua
-- ❌ 错：客户端也会执行
AddComponentPostInit("combat", function(cmp, inst)
    cmp:SetDefaultDamage(100)   -- 客户端不会真正执行，但仍然浪费 CPU
end)

-- ✅ 对：先判断
AddComponentPostInit("combat", function(cmp, inst)
    if not TheWorld.ismastersim then return end
    cmp:SetDefaultDamage(100)
end)
```

**原因**：`AddComponent` 在客户端和服务端都会被执行（尽管客户端不 create 某些组件）。服务端专属的修改要加 ismastersim 判断，避免客户端浪费。

**陷阱 2：PostInit 里做耗时操作**

```lua
-- ❌ 错：每个带 inventoryitem 的实体生成都会 scan 一次数据库
AddComponentPostInit("inventoryitem", function(cmp, inst)
    local db = LoadBigDatabase()   -- 耗时几十毫秒
    -- ...
end)
```

**原因**：`AddComponent` 被调用的频次**很高**——游戏里成千上万个实体都会 AddComponent("inventoryitem")。如果你在 PostInit 里做耗时操作，每次 `SpawnPrefab` 都会卡一下。

**解决**：**耗时操作提前到 `modmain.lua` 顶部做一次**，PostInit 里只读结果。

**陷阱 3：PostInit 改了组件字段但未触发 onset**

```lua
-- ❌ 错：仅改字段不触发同步
AddComponentPostInit("health", function(cmp, inst)
    cmp.maxhealth = 200   -- 客户端 replica 不会同步！
end)

-- ✅ 对：用组件提供的 setter
AddComponentPostInit("health", function(cmp, inst)
    cmp:SetMaxHealth(200)   -- setter 内部会同步到 replica
end)
```

**原因**：`health.lua` 的 `maxhealth` 字段用 `Class` 的 onset 机制（回顾 `onmaxhealth` 回调）——**直接赋值不会触发回调**，replica 不会同步。必须用 `SetMaxHealth` 或 `self.maxhealth` 才能触发。

看 `health.lua` 第 9-16 行的 onset：

```9:16:scripts/components/health.lua
local function onmaxhealth(self, maxhealth)
    self.inst.replica.health:SetMax(maxhealth)
    local repairable = self.inst.components.repairable
    if repairable then
        repairable:SetHealthRepairable((self.currenthealth or maxhealth) < maxhealth)
    end
    onpercent(self)
end
```

`onmaxhealth` 里调用了 `replica.health:SetMax`——这才是同步到客户端的关键。

**陷阱 4：覆盖方法时没保留原版**

```lua
-- ❌ 错：直接覆盖，原版逻辑全丢
AddComponentPostInit("weapon", function(cmp, inst)
    cmp.SetOnAttack = function(self, fn)
        -- ...我的新逻辑
    end
end)

-- ✅ 对：保留原版再追加
AddComponentPostInit("weapon", function(cmp, inst)
    local old_SetOnAttack = cmp.SetOnAttack
    cmp.SetOnAttack = function(self, fn)
        old_SetOnAttack(self, fn)
        -- ...我的追加逻辑
    end
end)
```

**原因**：直接覆盖会破坏原版行为，而且**其他 Mod 的 PostInit 如果在你之前**，他们的修改也会被你覆盖。**始终保留并调用原版**是 Mod 兼容性的基本要求。

**陷阱 5：PostInit 里添加的字段没进 OnSave**

```lua
-- ❌ 错：加的字段玩家存档后丢失
AddComponentPostInit("weapon", function(cmp, inst)
    cmp.__ModUpgradeLevel = 0   -- 这个字段重启游戏后变回 0
end)

-- ✅ 对：hook OnSave/OnLoad
AddComponentPostInit("weapon", function(cmp, inst)
    cmp.__ModUpgradeLevel = cmp.__ModUpgradeLevel or 0

    local old_OnSave = cmp.OnSave
    cmp.OnSave = function(self)
        local data = old_OnSave and old_OnSave(self) or {}
        data.__ModUpgradeLevel = self.__ModUpgradeLevel
        return data
    end

    local old_OnLoad = cmp.OnLoad
    cmp.OnLoad = function(self, data)
        if old_OnLoad then old_OnLoad(self, data) end
        if data and data.__ModUpgradeLevel then
            self.__ModUpgradeLevel = data.__ModUpgradeLevel
        end
    end
end)
```

**原因**：PostInit 里加的字段**不会自动进存档**——引擎的 `GetPersistData` 只调用组件的 `OnSave` 方法。你要让字段持久化，必须 hook 原版的 `OnSave/OnLoad`，在返回表里加自己的字段。

#### 第四步：什么时候用 PostInit，什么时候自己写新组件？

| 场景 | 推荐 |
|------|------|
| 调整现有组件的**默认值** | PostInit |
| 给现有组件**加方法 / 改方法** | PostInit（保留原版） |
| 给现有组件**加存档字段** | PostInit（hook OnSave/OnLoad） |
| 实现**全新的能力** | 自己写组件（6.7 节会讲） |
| 跨多个原版组件的**综合行为** | 自己写组件（调度其他组件） |

> **开发逻辑**：PostInit 是"**外挂式改造**"——你不动原版文件一个字，只在旁边加钩子。优点是**兼容性极好**（原版更新了只要结构不变就继续生效）；缺点是**钩子多了代码难读**——一个方法可能被 5 个 Mod 层层包裹。好的 Mod 会尽量**用最小的 PostInit 解决问题**，必要时才自己写新组件。

---

### 6.1.9 小结

- **组件是挂到实体上的 Lua Class 实例**——`scripts/components/*.lua` 各自定义一种"能力"。`Health`、`Weapon`、`Inventoryitem` 等约 350 个组件构成了游戏的全部行为基础。
- **`AddComponent` 不只是"添加字段"**：它经过 `LoadComponent`（延迟加载 + 缓存）→ `ReplicateComponent`（客户端影子）→ 构造函数 → Mod `ComponentPostInit` → `RegisterComponentActions` 五步完整流程——你写一行，引擎替你做十件事。
- **五个核心生命周期钩子**：构造函数（初始化）、`OnSave`（存档）、`OnLoad`（读档）、`OnEntitySleep/Wake`（休眠优化）、`OnRemoveFromEntity`（离场清理）。
- **`OnSave` 的铁律**：**只存必要的、只存非默认的、不存可推断的、不存函数**——这是 Klei 的存档精简哲学，也是你写 Mod 组件时必须遵守的规矩。
- **`OnLoad` 的铁律**：**所有字段都要 nil 检查**——为了兼容老存档升级到新版 Mod，每个字段都要防御式判断。
- **`OnEntitySleep/Wake` 用于性能优化**：取消定时任务、解除耗时引用、记录休眠时间。**醒来时做离散补偿**（如 `inventoryitemmoisture` 计算 `time_slept` 一次性 `UpdateMoisture`）是老手的标准做法。
- **`OnRemoveFromEntity` 的核心是"谁污染谁治理"**：**把组件加的所有 tag / 事件监听 / 定时任务 / 引用都拆干净**——幽灵标签是最常见的 bug 源。
- **`AddComponentPostInit`** 在每个实体 `AddComponent` 时都会执行——是 Mod 修改现有组件的入口。写 PostInit 的五个陷阱：不判 ismastersim、做耗时操作、改字段不触发 onset、覆盖不保留原版、加字段不 hook OnSave/OnLoad。
- **组件完整生命周期**（诞生→存活→离场）和第 5.2 节的**实体生命周期**是嵌套关系——**组件是实体的"细胞"，实体是组件的"容器"**，理解了两者的生命周期就能理解饥荒联机版的整个运行时架构。

> **下一节预告**：6.2 节我们深入 **Replica 组件**——`inst.replica.health`、`inst.replica.combat` 到底是什么？服务端 `health.maxhealth = 200` 是怎么"神奇地"让客户端也知道的？net 变量和 Classified 实体又如何串起来？

---

## 6.2 Replica 组件——客户端的"影子"

### 本节导读

6.1 节我们讲过：`AddComponent("health")` 里的第三步是 **`ReplicateComponent("health")`**——"客户端创建一个叫 `inst.replica.health` 的影子"。那一节一笔带过没深入讲。这一节我们把这个"**影子**"彻底拆开——**它是什么、怎么被创建、怎么和服务端组件同步、你写 Mod 时要注意什么**。

> **新手**从 6.2.1-6.2.3 入手，理解「客户端看不到服务端的真组件，只能通过 replica 拿数据」这个核心命题，能看懂官方任何一个 `*_replica.lua` 文件的结构；**进阶读者**从 6.2.4 开始深入，学习 `REPLICATABLE_COMPONENTS` 注册表、tag 标记法（`_health` vs `__health`）、以及 replica 里三种不同的数据源（net 变量、直接 tag、查 classified）；**老手**可以跳到 6.2.7-6.2.8，了解 replica 的完整生命周期、`AddReplicableComponent` 给自定义组件加 replica 的方法，以及写 replica 时最容易踩的 6 个陷阱。

**前置知识**：本节假设你**已经读过 5.4 节**（Pristine State）对"两段代码为什么各执行一次"和 replica 整体机制的讲解。6.2 节**不重复 5.4**——我们从**组件视角**重新切入，关注 replica 组件文件的骨架、API 使用模式、和写 Mod 时的实战。

---

### 6.2.1 快速入门：为什么需要 Replica？

#### 第一步：服务端和客户端看到的是"同一个实体"吗？

先想一个看似简单的问题——**玩家 A 的客户端怎么知道"蜘蛛怪 X 还剩 50 血"？**

在单机游戏里，这不是问题：血量就存在某个变量里，UI 直接读。

**但在联机版里情况完全不同**：

- **服务端（主机）**：蜘蛛的 `health` 组件是真实的——`inst.components.health.currenthealth = 50`
- **客户端**（连进来的其他玩家）：**根本没有 `inst.components.health`！**

这不是 bug——是饥荒联机版的故意设计。**客户端实体 `inst.components` 表里几乎是空的**（只有极少数客户端也需要独立运行的组件）。因为：

1. **网络带宽限制**：如果每个组件的所有字段都实时同步，几百只蜘蛛 × 几十个组件 × 几个字段 = 服务器上行带宽爆炸
2. **权威性要求**：只有服务端的数据才是"真"——客户端随便改数字没意义（也会被系统检测为作弊）
3. **客户端不需要全部信息**：客户端只要**能显示、能做预判**就够了——蜘蛛的 AI 决策、伤害计算、目标搜索都在服务端跑

但客户端又必须能看到"这只蜘蛛还剩 50 血"——因为：
- 要显示血量 debug UI（控制台 `c_sel()` 检查）
- 要判断"蜘蛛死了没，要不要播放死亡动画"
- 要给玩家显示"这个敌人血量低了"的视觉提示

**Klei 的解决方案**：**在客户端创建一个"影子组件"——只有必要字段，通过网络同步**。这就是 **Replica**（复刻、副本）。

#### 第二步：Replica 的工作模型

```
┌──────────────────────────────────┐           ┌──────────────────────────────────┐
│  服务端（主机）                    │           │  客户端（其他玩家）                 │
├──────────────────────────────────┤           ├──────────────────────────────────┤
│  inst.components.health          │           │  inst.components.health = nil    │
│    .maxhealth = 200              │           │                                   │
│    .currenthealth = 50           │           │  inst.replica.health             │
│    .regenrate = ...              │──────────▶│    .classified.maxhealth:value() │
│    .invincible = false           │  同步     │      → 200                        │
│    .lunarburns = {...}           │           │    .classified.currenthealth     │
│    ...（几十个字段）                │           │      → 50                         │
│                                  │           │    （只读接口）                      │
└──────────────────────────────────┘           └──────────────────────────────────┘
```

关键差异：

- **服务端**：`inst.components.health` 是一个**完整的** `Health` 实例，几十个字段、几十个方法、各种回调
- **客户端**：`inst.components.health` 是 `nil`！但 `inst.replica.health` 是一个**精简的** `Health` 实例——只有几个必要方法（`GetCurrent`、`GetPercent`、`IsDead` 等），字段通过 classified（联网对象）拿
- 服务端某字段变了 → 通过 net 变量自动同步 → 客户端的 `classified` 拿到新值 → `replica` 方法读 classified 就是最新值

#### 第三步：一个直观的 Mod 示例

假设你想做一个"**看某个怪物还剩多少血**"的 UI（类似其他 MMO 的敌人血条）——Mod 代码必须在**客户端**运行。但你在客户端写：

```lua
-- ❌ 错！客户端没有 components.health
local hp = inst.components.health.currenthealth
```

**崩溃**——因为客户端 `inst.components.health == nil`。

正确写法是**通过 replica 访问**：

```lua
-- ✅ 对
local hp = inst.replica.health:GetCurrent()
local pct = inst.replica.health:GetPercent()
```

同一个方法、同一个调用——但**底层在服务端读组件字段，在客户端读 classified 同步值**。这个"**无感切换**"就是 replica 最大的价值。

> **新手记忆**：**客户端要"看到"服务端组件的数据，只能通过 `inst.replica.<组件名>`**。这是饥荒联机版的铁律，违反它就是崩溃。

---

### 6.2.2 快速入门：`inst.replica.health` 到底是什么？

先看**引擎给实体预置的 `replica` 容器**（`scripts/entityscript.lua` 第 208-246 行）：

```208:246:scripts/entityscript.lua
local replica_mt =
{
	__index = function(t, k)
		return rawget(t, "inst"):ValidateReplicaComponent(k, rawget(t, "_")[k])
	end,
}

EntityScript = Class(function(self, entity)
    self.entity = entity
    self.components = {}
    self.lower_components_shadow = {}
    self.GUID = entity:GetGUID()
    self.spawntime = GetTime()
```

注意第 208-213 行的 `replica_mt`——这是 `inst.replica` 这张表的 **metatable**。当你写 `inst.replica.health` 时，实际发生的是：

1. **表里直接找**——初始是空的，找不到
2. **触发 `__index`**——调用 `ValidateReplicaComponent("health", rawget(t, "_")["health"])`
3. **检查 `_health` tag**——`ValidateReplicaComponent` 实现在 `entityreplica.lua` 第 30-32 行：

```30:32:scripts/entityreplica.lua
function EntityScript:ValidateReplicaComponent(name, cmp)
    return self:HasTag("_"..name) and cmp or nil
end
```

**翻译**：如果实体有 `"_health"` tag，就返回真正的 replica 组件；否则返回 `nil`。

**为什么要检查 `_health` tag？** 因为 replica 实例虽然在 `inst.replica._` 表里挂着，但只有"**当前活跃**"的 replica 才应该被访问——如果你在 `RemoveComponent("health")` 之后继续访问 `inst.replica.health`，会得到 `nil`（因为 `_health` tag 被移除了）。这是一种**"安全阀"**机制。

#### `inst.replica._` 是什么？

是**真正存放 replica 实例的"小仓库"**。第 244 行：

```lua
self.replica = { _ = {}, inst = self }
```

当 `ReplicateComponent("health")` 执行时（6.1.2 节讲过），最后一行：

```lua
rawset(self.replica._, name, cmp(self))
```

**把 replica 实例放到 `inst.replica._.health`**。外界访问 `inst.replica.health` 时，经过 metatable 里的 `rawget(t, "_")[k]` 再走 `_health` tag 检查——两层间接，双保险。

> **开发逻辑**：为什么不直接 `inst.replica.health = cmp(self)`？因为 Klei 要保证**只有合法的 replica 才能被访问**——直接赋值让检查失去意义。metatable + 下划线约定是 Lua 里实现"**有条件的字段访问**"最干净的写法。

#### 看一个 replica 文件的完整开头

`scripts/components/health_replica.lua` 第 1-9 行：

```1:9:scripts/components/health_replica.lua
local Health = Class(function(self, inst)
    self.inst = inst

    if TheWorld.ismastersim then
        self.classified = inst.player_classified
    elseif self.classified == nil and inst.player_classified ~= nil then
        self:AttachClassified(inst.player_classified)
    end
end)
```

**解读**：

- **`Class(function(self, inst) ... end)`**——和主组件一样，是一个 Lua Class
- **构造函数参数 `inst`**——将要挂到哪个实体上
- **`if TheWorld.ismastersim then`**——**服务端走这支**：直接把 `player_classified` 引用过来（因为服务端两个都在）
- **`else` 支**——**客户端走这支**：调用 `AttachClassified`，挂载联网对象并监听"onremove"事件

**这就是 replica 的核心矛盾——服务端和客户端各走一支**。同一个 replica 文件在两端都会被执行，但做的事很不一样：服务端只是**引用**真组件，客户端才真正**建立网络接收通道**。

---

### 6.2.3 快速入门：Replica 组件的骨架

刚才我们看了 `health_replica.lua` 的构造函数。现在看它的**方法**是如何实现"服务端和客户端**不同路径**"的。

#### 经典三段式：`GetCurrent`

```100:108:scripts/components/health_replica.lua
function Health:GetCurrent()
    if self.inst.components.health ~= nil then
        return self.inst.components.health.currenthealth
    elseif self.classified ~= nil then
        return self.classified.currenthealth:value()
    else
        return 100
    end
end
```

**完美的三段式**：

```
① 服务端：有真组件 → 直接读真组件字段
② 客户端：有 classified → 读 net 变量的值
③ 极端兜底：什么都没有 → 返回默认值（100）
```

为什么要三段式？**因为 replica 方法可能在以下任意场景下被调用**：

| 场景 | `components.health` | `classified` | 命中分支 |
|------|---------------------|--------------|---------|
| 服务端查询任意实体 | 存在 | 存在（同一引用） | ① 直接读真组件 |
| 客户端查询玩家实体 | `nil` | 存在 | ② 读 classified |
| 客户端查询其他实体 | `nil` | `nil`（未建立） | ③ 默认值 |

**注意**：**第一支 `self.inst.components.health ~= nil` 不是多余的**——即便 `classified` 存在，服务端也应该优先读真组件（更及时、无网络延迟）。

#### 另一种常见模式：直接读 tag

看 `health_replica.lua` 第 130-140 行：

```130:140:scripts/components/health_replica.lua
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

**这里没有走 classified**——用 **tag 直接存状态**。为什么？因为：

- **tag 的同步成本几乎为零**：它在实体创建时通过 pristine state 同步，后续可实时同步（tag 变化会通过实体的网络消息系统推送）
- **语义简单**：`isdead` 是布尔量，用 tag 比开一个 net_bool 更轻量
- **查询快**：`HasTag` 是内置 C++ 实现，比访问 `classified.xxx:value()` 还快

> **开发逻辑**：**能用 tag 表达的布尔状态，就不要开 net 变量**。Klei 在源码里大量用 tag 做这种"轻量状态同步"——`"isdead"`、`"cannotheal"`、`"cannotmurder"`、`"sleeping"`、`"incombat"` 等。

#### SetXxx 方法的写法

看 `health_replica.lua` 第 51-61 行：

```51:61:scripts/components/health_replica.lua
function Health:SetCurrent(current)
    if self.classified ~= nil then
        self.classified:SetValue("currenthealth", current)
    end
end

function Health:SetMax(max)
    if self.classified ~= nil then
        self.classified:SetValue("maxhealth", max)
    end
end
```

**SetXxx 只在服务端被调用**——因为同步是单向的（服务端 → 客户端）。客户端调用 `SetCurrent(50)` 不会改变服务端的值（事实上引擎会 assert 阻止这种操作）。

**实际流程**：

```
服务端：
  inst.components.health.currenthealth = 50  （直接改字段）
    ↓
  Class 的 onset 触发 oncurrenthealth 回调（health.lua 第 18-26 行）
    ↓
  oncurrenthealth 里调用 self.inst.replica.health:SetCurrent(50)
    ↓
  replica 的 SetCurrent 调用 self.classified:SetValue("currenthealth", 50)
    ↓
  classified 更新 net 变量 → 网络包发出 → 客户端收到 → classified 的 net 变量更新
    ↓
  客户端调用 inst.replica.health:GetCurrent() 时，读到最新值 50
```

**整个链路**：写一个字段 `health.currenthealth = 50` → 自动触发几跳 → 客户端同步——**你不用写一行网络代码**。这就是 replica 系统最干净的地方。

#### 用 `inst.player_classified` vs 独立 classified

`health_replica.lua` 用的是 **`inst.player_classified`**（玩家专属共享 classified），因为 health 只在玩家身上存在——Klei 把所有玩家相关的 replica 数据都挤进一个 `player_classified` 实体，节省网络实体数量。

而 `inventoryitem_replica.lua`（第 13-14 行）为**每个物品** SpawnPrefab 了一个独立的 classified：

```13:14:scripts/components/inventoryitem_replica.lua
        self.classified = SpawnPrefab("inventoryitem_classified")
        self.classified.entity:SetParent(inst.entity)
```

**差异的原因**：玩家数量少（最多 6-8 个），可以共用 classified；但物品可能有几千个，所以每个物品独立 classified（这些 classified 实体也会加到 `Ents` 全局表里）。

6.3 节我们会详细讲 Classified 实体的设计哲学，这里只要记住：**replica 背后的 net 数据存在某个 classified 实体上**——可能是玩家共享的，也可能是物品独有的。

---

### 6.2.4 进阶：`REPLICATABLE_COMPONENTS` 注册表

上一节你可能注意到一个反直觉的事实：**不是所有组件都会创建 replica**。只有被"登记"在一张白名单里的组件，才会在 `AddComponent` 时触发 `ReplicateComponent` 真正做事情。

这张白名单就在 `entityreplica.lua` 第 5-26 行：

```5:26:scripts/entityreplica.lua
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

**官方原版一共只有 20 个"可复刻"组件**。绝大多数组件（`fueled`、`perishable`、`workable`、`weapon` 等）**没有 replica**——因为客户端根本不需要知道它们的精细状态。

#### `ReplicateComponent` 的过滤逻辑

看 `entityreplica.lua` 第 34-60 行：

```34:60:scripts/entityreplica.lua
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

**第一行就是白名单检查**：`if not REPLICATABLE_COMPONENTS[name] then return end`——**不在白名单里的组件，这个函数啥都不干直接返回**。

这就是为什么你在 `weapon.lua` 里写 `SetDamage` 不需要考虑客户端同步——**weapon 压根没有 replica**，客户端只能通过其他组件间接感知（比如 `inventoryitem_replica` 里记录了 attackrange）。

#### 为什么不全部都做 replica？

你可能想："干脆所有组件都做 replica 不就一劳永逸吗？" **不行**，原因有三：

1. **网络负担**：每个 replica 要同步几个字段——350 个组件 × 几个字段 × 几千个实体 = 带宽灾难
2. **大部分组件不需要客户端感知**：`weapon.damage` 客户端知道没意义（伤害在服务端结算）
3. **设计约束**：强制开发者思考"**哪些状态客户端真的需要**"，避免滥用同步

**判断哪些组件需要 replica 的经验法则**：

| 组件需要 replica 当且仅当 | 举例 |
|---------------------------|------|
| 客户端需要它的**数据来渲染 UI** | `health` 显示血量、`sanity` 显示理智 |
| 客户端需要它的**数据做预测判断** | `inventoryitem` 判断能否拾起 |
| 客户端需要它来**识别实体能力** | `combat` 判断"这个东西能不能打我" |
| **右键菜单要用它**判断可用动作 | `workable`（可采集？）实际是通过 tag 实现 |

**绝大多数"纯服务端计算"的组件不需要 replica**：`fueled`（火把燃料）、`perishable`（腐烂）、`burnable`（燃烧）、`growable`（生长）等——客户端即便想知道也知道不了，也没必要知道。

#### Mod 如何往白名单里加组件？

`entityreplica.lua` 最后一行（第 98-100 行）：

```98:100:scripts/entityreplica.lua
-- Mod access
function AddReplicableComponent(name)
    REPLICATABLE_COMPONENTS[name] = true
end
```

这是 **Klei 留给 Mod 的扩展入口**——如果你写了一个自定义组件 `myweapon`，想让客户端也能感知到它的数据：

```lua
-- modmain.lua
AddReplicableComponent("myweapon")
```

然后在 `scripts/components/` 下放一个 `myweapon_replica.lua`——这样每次 `AddComponent("myweapon")` 都会自动 ReplicateComponent，客户端就会有 `inst.replica.myweapon`。

我们在 6.7 节会完整实战一次这个流程。

---

### 6.2.5 进阶：tag 标记机制（`_health` vs `__health`）

`ReplicateComponent` 第 39-45 行有一段容易被忽视的代码：

```lua
if TheWorld.ismastersim then
    self:AddTag("_"..name)
    if self:HasTag("__"..name) then
        self:RemoveTag("__"..name)
        return
    end
end
```

**这段代码里两个 tag**：`_health`（单下划线）和 `__health`（双下划线）。它们是干什么的？

#### 第一步：`_health` tag 的作用

**`_health` tag 表示"**该实体的 health replica 当前是活跃的**"。

前面 6.2.2 节的 `ValidateReplicaComponent` 会检查这个 tag——没有这个 tag 的话，`inst.replica.health` 访问会返回 `nil`，就像这个 replica 不存在一样。

**谁会加这个 tag？**
- 服务端 `AddComponent("health")` 时自动加
- 通过 pristine state **自动同步到客户端**——客户端看到这个 tag 就知道"这个实体有 health replica"

**为什么客户端也需要知道这个 tag？** 因为客户端要**判断一个实体有没有某能力**。比如客户端 UI 代码：

```lua
if target.replica.health ~= nil then
    -- 显示目标血条
end
```

如果没有 `_health` tag，这个判断就永远是 `nil`——正确。

#### 第二步：`__health` tag 是什么？

**`__health`（双下划线）表示"**服务端曾经有 health 组件但现在没了**"——即"**被移除的**"状态。

看 `UnreplicateComponent` 第 62-67 行：

```62:67:scripts/entityreplica.lua
function EntityScript:UnreplicateComponent(name)
    if rawget(self.replica, "_")[name] ~= nil and TheWorld.ismastersim then
        self:RemoveTag("_"..name)
        self:AddTag("__"..name)
    end
end
```

**逻辑**：移除 `_health`，加上 `__health`——这是一种"**墓碑标记**"：告诉世界"这个实体**曾经**有 health，但现在没了"。

#### 第三步：`__health` 为什么要存在？

看起来很奇怪：既然都没了，为什么不干脆删除 tag？答案在 `ReplicateComponent` 第 41-44 行：

```lua
if TheWorld.ismastersim then
    self:AddTag("_"..name)
    if self:HasTag("__"..name) then
        self:RemoveTag("__"..name)
        return  -- ← 关键：碰到墓碑就 return，不再真正创建 replica
    end
end
```

**场景**：玩家实体**先被添加 health → 然后移除 health → 又再次添加 health**。

- **第一次 Add**：加 `_health` tag，创建 replica
- **Remove**：移除 `_health`，加 `__health`（墓碑）
- **第二次 Add**：加 `_health`（重新标记活跃），检测到 `__health` 存在 → 移除墓碑 → **直接 return，不再创建 replica 实例**！

**为什么不再创建？** 因为 replica 实例**已经存在于 `rawget(self.replica._, name)` 里**（没被删除过）——直接复用它即可。节省内存、避免对象重建。

#### 第四步：`PrereplicateComponent` 的用途

还有一个特殊函数 `PrereplicateComponent`（第 69-72 行）：

```69:72:scripts/entityreplica.lua
function EntityScript:PrereplicateComponent(name)
    self:ReplicateComponent(name)
    self:UnreplicateComponent(name)
end
```

**看起来很怪**——加了又立刻删？作用是**只留下墓碑 `__health` tag 但不创建 replica**。

这在**实体被延迟初始化**的场景下有用：比如某些 Prefab 在 pristine 阶段**不创建 health 组件**，但希望客户端知道"**如果后续服务端加了 health，这是合法的**"——调用 `PrereplicateComponent("health")`，就会留下 `__health` 墓碑，未来正式添加时就能走"**复用已有 replica**"的路径。

这属于比较少见的底层机制，**普通 Mod 用不到** `PrereplicateComponent`——90% 的场景下 `AddComponent` 自动 `ReplicateComponent` 就够了。

#### 第五步：tag 状态转换图

把三种状态的转换画出来：

```
初始状态：实体刚创建
  │
  │  ReplicateComponent("health")
  ▼
┌────────────────┐
│  +_health tag  │  ← "活跃中"
│  replica 实例存在 │
└────────────────┘
  │
  │  UnreplicateComponent
  ▼
┌────────────────┐
│  -_health tag  │
│  +__health tag │  ← "已移除（墓碑）"
│  replica 实例仍存在（复用）│
└────────────────┘
  │
  │  再次 ReplicateComponent
  ▼
┌────────────────┐
│  +_health tag  │
│  -__health tag │  ← "重新活跃"（复用旧实例）
└────────────────┘
```

**这是 Klei 用 tag 做"状态机"的经典案例**——两个简单的 tag 组合出 4 种状态（都没 / 只有 _ / 只有 __ / 都没），覆盖所有可能的生命周期阶段。

> **开发逻辑**：为什么用 tag 而不是用组件内部的 bool 字段？因为 **tag 会自动同步到客户端**——服务端的 bool 字段客户端看不到。tag 是跨端的"状态广告板"，是饥荒联机版最高效的跨端状态传递方式。

---

### 6.2.6 进阶：Replica 里的三种数据来源

同一个 replica 文件里，你会看到字段的读取路径有**三种不同来源**。理解它们的区别是写好 replica 的关键。

#### 来源 ①：服务端真组件字段

```100:108:scripts/components/health_replica.lua
function Health:GetCurrent()
    if self.inst.components.health ~= nil then
        return self.inst.components.health.currenthealth
    elseif self.classified ~= nil then
        return self.classified.currenthealth:value()
    else
        return 100
    end
end
```

**第一行分支 `self.inst.components.health`**——直接读真组件字段。

**特点**：
- **只能在服务端生效**（客户端 `inst.components.health == nil`）
- **实时、准确**——没有网络延迟
- **不占任何网络带宽**

#### 来源 ②：classified 的 net 变量

```lua
return self.classified.currenthealth:value()
```

**`classified` 是一个独立的联网实体**（或玩家共享的 `player_classified`），上面挂了一堆 `net_bool` / `net_uint` / `net_hash` 等 net 变量。

**特点**：
- **客户端专用**
- **有延迟**（网络包过来才更新）
- **只读**——客户端不能 set，只能 read value

#### 来源 ③：tag 直接存状态

```130:140:scripts/components/health_replica.lua
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

**`AddTag / HasTag` 自动跨端同步**。

**特点**：
- **最轻量**——tag 是 C++ 实现的字符串哈希
- **只适合布尔状态**
- **服务端 AddTag 后客户端立即能 HasTag**

#### 三种来源的选择决策树

```
问：这个字段客户端需不需要知道？
├─ 不需要 → 不写 replica 方法，放在主组件里就够
└─ 需要 → 下一步
   │
   问：这是一个布尔量吗？
   ├─ 是 → 用 tag（来源 ③）
   └─ 不是 → 下一步
      │
      问：数值范围小（能塞进 8/16/32 位）吗？
      ├─ 是 → net_byte / net_smallbyte / net_ushortint（来源 ②）
      └─ 不是 → net_uint / net_string（来源 ②）
```

#### 几个实战示例

**① "是否无敌"**（布尔量）→ **用 tag**：

```lua
-- health.lua 第 52-60 行
local function oninvincible(self, invincible)
    if CHEATS_ENABLED then
        if invincible then
            self.inst:AddTag("invincible")
        else
            self.inst:RemoveTag("invincible")
        end
    end
end
```

**② "当前血量"**（0-999 的整数）→ **net 变量**：

```lua
-- player_classified.lua 里
inst.currenthealth = net_ushortint(inst.GUID, "health.currenthealth", "currenthealthdirty")
```

**③ "最大血量"**（0-999）→ 同样 net 变量。

**④ "血量百分比"**（0-1 浮点数）→ **不同步**！客户端用 currenthealth / maxhealth **现算**：

```lua
-- health_replica.lua 第 90-98 行
function Health:GetPercent()
    if self.inst.components.health ~= nil then
        return self.inst.components.health:GetPercent()
    elseif self.classified ~= nil then
        return self.classified.currenthealth:value() / self.classified.maxhealth:value()
    else
        return 1
    end
end
```

**这是典型的"派生字段不同步"优化**——百分比是由 current/max 推导的，同步它是浪费。

> **开发逻辑**：**能不同步就不同步，能派生就不存储**。写 replica 时每加一个 net 变量，你就是在"掏玩家的网络带宽"——一定要问自己"这个数据客户端真的需要吗？能不能从已有数据推出来？"

---

### 6.2.7 老手进阶：Replica 的完整生命周期

Replica 组件也有生命周期——虽然它的钩子没有主组件那么多（`OnSave/OnLoad/OnEntitySleep/Wake` 都不适用），但诞生、存活、销毁三阶段依然清晰。

#### A. 诞生阶段：服务端 + 客户端各走一次

```
【服务端】
AddComponent("health")
  ├── ReplicateComponent("health")
  │     ├── REPLICATABLE_COMPONENTS["health"] == true  ✓
  │     ├── AddTag("_health")                    ← 告诉客户端"我有 health replica"
  │     ├── require("components/health_replica")  （第一次才加载）
  │     ├── rawset(self.replica._, "health", Health(inst))  ← 创建实例
  │     └── 构造函数里 self.classified = inst.player_classified
  │
  └── 继续原版 AddComponent 流程（构造真 Health、AttributePostInit 等）

【客户端】
实体反序列化完成
  ├── 所有 tag 已同步（包括 _health）
  │
  └── ReplicateEntity()（entityreplica.lua 第 75-85 行）
        └── 遍历 REPLICATABLE_COMPONENTS，发现 _health tag 存在
              └── ReplicateComponent("health")
                    ├── require("components/health_replica")
                    ├── 创建 Health(inst) 实例
                    └── 构造函数里走 else 支：
                          if inst.player_classified ~= nil then
                              self:AttachClassified(inst.player_classified)
```

**关键点**：客户端的 replica **不是**在 `AddComponent` 时创建的——是在**实体反序列化完成**后，通过 `ReplicateEntity` 根据 `_<name>` tag **批量创建**的。

看 `entityreplica.lua` 第 75-85 行：

```75:85:scripts/entityreplica.lua
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

**末尾的 `OnEntityReplicated` 钩子**——5.2.4 节介绍过。这是**客户端专属的"实体刚同步完"事件**，非常适合做客户端的一次性初始化。

#### B. 存活阶段：数据同步透明进行

- 服务端改 `health.currenthealth` → onset 回调触发 → `replica.health:SetCurrent(current)` → `classified:SetValue` → net 变量脏标记 → 下一帧同步
- 客户端 net 变量脏标记 dispatch → listener 处理 → UI 重绘

**这一阶段 replica 实例本身几乎不做事**——它只是一个"门面"（façade pattern），所有工作都由引擎的 net 变量系统自动完成。

#### C. 销毁阶段：replica 随实体一起销毁

看 `entityscript.lua` 第 1729-1733 行：

```1729:1733:scripts/entityscript.lua
    for k, v in pairs(rawget(self.replica, "_")) do
        if v and type(v) == "table" and v.OnRemoveEntity then
            v:OnRemoveEntity()
        end
    end
```

**实体销毁时遍历所有 replica，如果它们有 `OnRemoveEntity` 方法就调用**——让 replica 有机会清理自己的 classified。

看 `inventoryitem_replica.lua` 第 64-69 行：

```64:69:scripts/components/inventoryitem_replica.lua
function InventoryItem:OnRemoveEntity()
	if self.classified and TheWorld.ismastersim then
		self.classified:Remove()
		self.classified = nil
	end
end
```

**关键**：**只在服务端清理 classified**——因为 classified 本身是联网实体，服务端销毁它后客户端会自动收到 `onremove` 事件。

#### D. 对比：health_replica 为什么没有 OnRemoveEntity？

看 `health_replica.lua` 第 13-25 行的**注释掉**的代码：

```13:25:scripts/components/health_replica.lua
--V2C: OnRemoveFromEntity not supported
--[[function Health:OnRemoveFromEntity()
    if self.classified ~= nil then
        if TheWorld.ismastersim then
            self.classified = nil
        else
            self.inst:RemoveEventCallback("onremove", self.ondetachclassified, self.classified)
            self:DetachClassified()
        end
    end
end

Health.OnRemoveEntity = Health.OnRemoveFromEntity]]
```

**为什么注释掉？** 因为 `health_replica` 用的是 `inst.player_classified`——**整个玩家共享的**，health replica 本身不拥有这个 classified。如果 health replica 移除时把 player_classified 清掉，会影响玩家实体上的其他 replica（sanity、hunger、inventory 等）。

**老版本**可能支持过，但 V2C（Klei 的一位核心开发）在 `// V2C:` 注释里明确说**不再支持**——让 player_classified 随玩家实体整体管理，replica 不干涉。

---

### 6.2.8 老手进阶：`AddReplicableComponent` 与自定义 Replica 的六个陷阱

#### 第一步：自定义 Replica 的基本流程

假设你自己写了一个 `myability` 组件（主组件），想让客户端也能读它的"当前魔力值 / 最大魔力值"。步骤：

1. **声明为可复刻**：`modmain.lua` 里加 `AddReplicableComponent("myability")`
2. **写 replica 文件**：创建 `scripts/components/myability_replica.lua`
3. **在主组件里写 onset 回调**：让字段变化自动触发 replica 同步
4. **决定 classified 方案**：是用独立 classified、还是共享（如 `player_classified`）

下面是一个最小范例的四件套：

**① `modmain.lua`**：

```lua
AddReplicableComponent("myability")
```

**② `scripts/components/myability.lua`（主组件）**：

```lua
local function onmanamax(self, manamax)
    self.inst.replica.myability:SetManaMax(manamax)
end

local function oncurrentmana(self, currentmana)
    self.inst.replica.myability:SetCurrentMana(currentmana)
end

local MyAbility = Class(function(self, inst)
    self.inst = inst
    self.manamax = 100
    self.currentmana = 100
end,
nil,
{
    manamax = onmanamax,
    currentmana = oncurrentmana,
})

return MyAbility
```

注意 **Class 的第三个参数 `{manamax = onmanamax, ...}`**——这是 Lua Class 的 **onset 机制**：设置字段时自动调用回调（不是赋值就执行，而是 `__newindex` 里识别并调用）。

**③ `scripts/components/myability_replica.lua`**：

```lua
local MyAbility = Class(function(self, inst)
    self.inst = inst

    self._manamax = net_ushortint(inst.GUID, "myability._manamax", "manamaxdirty")
    self._currentmana = net_ushortint(inst.GUID, "myability._currentmana", "currentmanadirty")
end)

function MyAbility:SetManaMax(manamax)
    self._manamax:set(manamax)
end

function MyAbility:SetCurrentMana(currentmana)
    self._currentmana:set(currentmana)
end

function MyAbility:GetManaMax()
    if self.inst.components.myability ~= nil then
        return self.inst.components.myability.manamax
    else
        return self._manamax:value()
    end
end

function MyAbility:GetCurrentMana()
    if self.inst.components.myability ~= nil then
        return self.inst.components.myability.currentmana
    else
        return self._currentmana:value()
    end
end

return MyAbility
```

**注意**：这里用的是**直接挂 net 变量**（不是通过 classified 实体）。对于简单的数据、不需要玩家共享的场景，这种写法最简单。复杂场景（如物品的 `inventoryitem_replica`）才需要独立 classified 实体。

**④ 实体在服务端使用**：

```lua
inst:AddComponent("myability")
inst.components.myability.manamax = 200  -- 触发 onmanamax → replica:SetManaMax → net 变量同步
```

#### 第二步：六个常见陷阱

**陷阱 1：忘了 `AddReplicableComponent`**

```lua
-- ❌ 错：不加这行，ReplicateComponent 在白名单检查时直接 return
-- AddReplicableComponent("myability")

inst:AddComponent("myability")
-- 客户端 inst.replica.myability == nil
```

**症状**：客户端 `inst.replica.myability` 永远是 nil，即便 replica 文件写得再好。

---

**陷阱 2：主组件字段改了但没触发 replica 同步**

```lua
-- ❌ 错：直接赋值，没有 onset 回调
inst.components.myability.currentmana = 50

-- ✅ 对：在 Class 里声明 onset
local MyAbility = Class(..., nil, {currentmana = oncurrentmana})
inst.components.myability.currentmana = 50  -- 这次会触发 oncurrentmana
```

**原理**：Lua Class 的 onset 是通过 `__newindex` 实现的——必须在 Class 定义时声明，后续字段赋值才会触发。

---

**陷阱 3：客户端代码用 `components` 而不是 `replica`**

```lua
-- ❌ 错：客户端代码里这样写
local mana = inst.components.myability.currentmana   -- 客户端这里是 nil

-- ✅ 对：通过 replica
local mana = inst.replica.myability:GetCurrentMana()
```

记住 6.2.1 节的铁律——**客户端要读数据一律走 replica**。

---

**陷阱 4：net 变量名字冲突**

```lua
-- myability_replica.lua
self._mana = net_ushortint(inst.GUID, "myability.mana", ...)

-- 某个老的 component_replica.lua 也用了同样的名字
self._mana = net_ushortint(inst.GUID, "myability.mana", ...)  -- 崩溃！
```

**net 变量的第二个参数是字符串名字——必须全实体唯一**。Klei 的约定是 `<组件名>.<字段名>`——你的 Mod 也应该这么做（前面加上 Mod 前缀更稳妥，如 `"mymod.myability._mana"`）。

---

**陷阱 5：在 replica 里尝试 set 服务端独有字段**

```lua
-- replica 里
function MyAbility:DoSomething()
    self._currentmana:set(100)   -- 只在服务端有效——客户端 set 会崩或被忽略
end
```

**net 变量的 `:set`** 只在服务端生效——客户端调用会报错或静默失败。**正确的做法**：客户端通过 RPC（Remote Procedure Call）向服务端发请求，由服务端 set。

---

**陷阱 6：数据更新太频繁**

```lua
-- 主组件里
function MyAbility:OnUpdate(dt)
    self.currentmana = self.currentmana + dt * 1.0  -- 每帧都触发一次同步！
end
```

**每帧更新 net 变量 = 带宽灾难**。正确的做法：

```lua
-- 只在跨越整数阈值时才同步
function MyAbility:OnUpdate(dt)
    local old = math.floor(self.currentmana)
    self.currentmana = self.currentmana + dt * 1.0
    if math.floor(self.currentmana) ~= old then
        -- 只在整数值改变时触发（onset 里再 set net 变量）
    end
end
```

官方 `health.lua` 对 `currenthealth` 的处理就是**只在值实际变化时才赋值**——而不是连续的浮点数更新。这避免了每帧发网络包。

#### 第三步：什么时候不用 classified 实体？

这是最常见的架构决策。判断标准：

| 场景 | 选择 |
|------|------|
| 所有玩家都能看到的**少量字段**（< 5） | 直接 net 变量（挂在实体本身） |
| 只有自己（owner）能看到的字段（如背包内容） | **独立 classified 实体**（SetParent 到 owner，用 `Classified` 的网络过滤） |
| 数据量大且玩家共享 | 挂在 `player_classified` 上（看 `player_classified.lua`） |
| 物品通用数据（可拾起、可堆叠、可装备等） | 独立 classified（见 `inventoryitem_classified.lua`） |

这部分我们会在 6.3 节 Classified 专题详细讲。

---

### 6.2.9 小结

- **Replica 是"客户端看不到服务端真组件时的唯一入口"**——客户端的 `inst.components` 几乎空，必须通过 `inst.replica.<组件名>` 访问数据。
- **`inst.replica` 的 metatable 机制**：访问 `inst.replica.health` 会触发 `ValidateReplicaComponent`，检查 `_health` tag 是否存在——确保只有"活跃的" replica 才能被访问。
- **`*_replica.lua` 文件骨架**：Class 构造函数里分服务端/客户端两路（`if TheWorld.ismastersim`），服务端引用真组件，客户端挂 classified 并监听 onremove。
- **Replica 方法三段式**：① 服务端有真组件直接读 → ② 客户端有 classified 读 net 变量 → ③ 兜底默认值。每个 Getter 方法都应该遵循。
- **`REPLICATABLE_COMPONENTS` 白名单**：原版 20 个组件，Mod 通过 `AddReplicableComponent(name)` 加入——不在白名单里的组件，`ReplicateComponent` 直接 return 不做事。
- **`_health` vs `__health` tag 双标机制**：前者是"活跃"，后者是"墓碑"。服务端通过这两个 tag 广播 replica 的生命周期状态，客户端据此决定是否创建 replica。
- **三种数据来源**：① 服务端真组件字段（服务端专用）、② classified 的 net 变量（跨端同步）、③ tag（布尔量的最轻同步）。能用 tag 的别开 net，能派生的别存储。
- **Replica 生命周期**：服务端 AddComponent 时同步创建；客户端在实体反序列化后通过 `ReplicateEntity` 批量创建（根据 `_<name>` tag）；销毁时通过 `OnRemoveEntity` 清理 classified。
- **自定义 Replica 六陷阱**：没 `AddReplicableComponent`、主组件字段没 onset、客户端误用 `components`、net 变量名冲突、客户端 set、更新频率太高——每个都是自己写组件时的血的教训。

> **下一节预告**：6.3 节我们进入 **Classified 实体**——它本质是一个"**只为承载 net 变量而存在的"隐形联网实体**。`player_classified` 为什么能承载 50+ 字段？物品 classified 是怎么做"只给 owner 看"的可见性过滤？Classified 如何成为 replica 背后的真正"同步引擎"——一切答案在下一节。

## 6.3 Classified 实体——网络同步的桥梁

### 本节导读

6.2 节我们看到 `health_replica.lua` 里反复出现一个字段叫 `self.classified`——这个神秘的"联网对象"到底是什么？为什么客户端要通过它才能读到服务端的数据？本节我们把 **Classified 实体**从里到外拆开——**它是什么、为什么要用独立实体来承载 net 变量、父子关系怎么设计、可见性怎么过滤、自己写一个要注意什么**。

> **新手**从 6.3.1-6.3.3 入手，理解「classified 就是一个专门承载 net 变量的隐形实体」，看懂 `player_classified.lua` 和 `inventoryitem_classified.lua` 的基本骨架，记住 `CLASSIFIED` tag 的意义；**进阶读者**从 6.3.4 开始深入 CLASSIFIED tag 的网络可见性过滤、`SetParent` 父子关系、以及 `Serialize/Deserialize` 双向模式（如 `SerializePercentUsed` / `DeserializePercentUsed`）；**老手**跳到 6.3.7-6.3.8 看 `player_classified` 1600 多行的整体架构——为什么 50+ 个字段还能这么组织、六个自己写 classified 时最容易栽的坑。

**前置**：建议先读过 6.2 节的 Replica 机制，本节假设你已经知道 replica 里 `self.classified.xxx:value()` 是在干什么。

---

### 6.3.1 快速入门：Classified 到底是什么？

**一句话定义**：**Classified 是一个"没有身体、只为承载 net 变量而存在的隐形联网实体"**。

这听起来抽象，我们从"**为什么非要用一个独立实体来承载 net 变量**"这个问题切入。

#### 第一个问题：为什么不把 net 变量直接挂在主实体上？

事实上**可以**。看 `inventoryitem_replica.lua` 第 7-10 行：

```7:10:scripts/components/inventoryitem_replica.lua
    self._cannotbepickedup = net_bool(inst.GUID, "inventoryitem._cannotbepickedup")
    self._iswet = net_bool(inst.GUID, "inventoryitem._iswet", "iswetdirty")
    self._isacidsizzling = net_bool(inst.GUID, "inventoryitem._isacidsizzling", "isacidsizzlingdirty")
    self._grabbableoverridetag = net_hash(inst.GUID, "inventoryitem._grabbableoverridetag")
```

这四个 net 变量用的都是 `inst.GUID`——**挂在物品自己身上**的。对于简单的字段，这种"**直接挂实体**"的方式就够用。

**但当字段很多、或需要单独管理生命周期时，问题就来了**。想象 `player_classified` 上挂着：

- health 相关：currenthealth、maxhealth、healthpenalty、istakingfiredamage、lunarburnflags... 10+ 个
- hunger 相关：currenthunger、maxhunger、ishungerpulseup... 5+ 个
- sanity 相关：currentsanity、maxsanity、sanitypenalty... 10+ 个
- 临时状态：pausepredictionframes、oldagerrate、bathingpoolcamera... 20+ 个
- 其他：inspiration、wereness、skilltree、moonstorm... 几十个

**50+ 个 net 变量**全塞到玩家实体上，会带来几个严重问题：

1. **字段命名冲突**：`inst.maxhealth` 和 `inst.components.health.maxhealth` 名字相同，访问时要加前缀容易搞错
2. **可见性控制困难**：如果某些字段"**只给 owner（主人）看**"（如玩家背包明细），混在一起难做过滤
3. **生命周期耦合**：主实体销毁、复活、传送时，这些字段全混在一起，要分别处理很麻烦

#### 第二个问题：用独立实体承载有什么好处？

把所有 net 变量搬到一个**专属的独立实体**上（就是 classified），**立即解决三个问题**：

1. **命名空间独立**：`inst.classified.maxhealth` 明确指明是网络同步值，不和 `inst.components.health.maxhealth` 混淆
2. **可见性一键控制**：整个 classified 实体打上一个可见性标记（通过 `Network` 系统），所有字段跟着走
3. **独立销毁**：classified 可以单独 `Remove`——主实体还活着，但 classified 清零 → 相当于"**断开所有网络同步**"

这就是 Classified 的设计初衷——**用"**子实体**"的方式管理一组相关的 net 变量**。

#### 第三步：一个生动的类比

可以把 Classified 想象成**"网络快递包裹"**：

- **主实体**（玩家 / 物品） = 大客户，**要寄东西**
- **Classified** = 快递包裹，**装着要寄的内容**（net 变量）
- **父子关系（SetParent）** = 包裹贴着客户的地址标签
- **`CLASSIFIED` tag** = "这是包裹不是客户本人"的识别标志
- **可见性过滤** = "这个包裹寄给谁"（所有人 vs 只给 owner）

**所以 classified 本身不是游戏对象**——它是"**服务于主实体的网络载体**"。

#### 第四步：两种典型的 Classified

饥荒联机版里最常见的两种 classified：

| 类型 | 作用 | 字段数 | 父实体 | 可见性 |
|------|------|-------|--------|--------|
| `player_classified` | 玩家专属 | 50+ | player 实体 | 所有人可见 |
| `inventoryitem_classified` | 物品专属 | 20+ | 物品实体 | 所有人可见 |
| `inventory_classified` | 背包内容 | 复杂 | player 实体 | **只给 owner 可见** |
| `wx78_classified` | WX-78 特有数据 | 几个 | WX-78 实体 | 所有人可见 |

**`inventory_classified` 是"只给 owner 看"的典型例子**——其他玩家不能看你背包里有什么，这是设计上的需求，classified 系统提供了原生支持。

> **新手记忆**：**所有带 `_classified` 后缀的 Prefab，都是一个专门承载 net 变量的隐形实体，用来为某个主实体做网络同步**。读到 `SpawnPrefab("xxx_classified")` 不要困惑——就是在"**给主实体配一个快递包裹**"。

---

### 6.3.2 快速入门：一个 Classified 文件的骨架

现在我们打开 `scripts/prefabs/inventoryitem_classified.lua`——这是一个 **224 行的完整 classified 样板**，官方写得很规整，非常适合作为学习范本。

#### 第一步：工厂函数的最小骨架

看第 142-162 行的开头：

```142:162:scripts/prefabs/inventoryitem_classified.lua
local function fn()
    local inst = CreateEntity()

    if TheWorld.ismastersim then
        inst.entity:AddTransform() --So we can follow parent's sleep state
    end
    inst.entity:AddNetwork()
    inst.entity:Hide()
    inst:AddTag("CLASSIFIED")

    inst.image = net_hash(inst.GUID, "inventoryitem.image", "imagedirty")
    inst.atlas = net_hash(inst.GUID, "inventoryitem.atlas", "imagedirty")
    inst.cangoincontainer = net_bool(inst.GUID, "inventoryitem.cangoincontainer")
    inst.canonlygoinpocket = net_bool(inst.GUID, "inventoryitem.canonlygoinpocket")
    inst.canonlygoinpocketorpocketcontainers = net_bool(inst.GUID, "inventoryitem.canonlygoinpocketorpocketcontainers")
    inst.src_pos =
    {
        isvalid = net_bool(inst.GUID, "inventoryitem.src_pos.isvalid"),
        x = net_float(inst.GUID, "inventoryitem.src_pos.x"),
        z = net_float(inst.GUID, "inventoryitem.src_pos.z"),
    }
```

**五个关键点**：

**① `CreateEntity()`**：创建底层实体——classified 本质上还是一个实体，只是"看不见"。

**② `AddTransform()` 只在服务端**：因为 classified 要"**跟着父实体一起休眠**"（`So we can follow parent's sleep state`）。Transform 组件让它有位置，从而能触发 sleep/wake 机制。

**③ `AddNetwork()` 两端都要**：这是 classified 的核心——它是联网实体，必须有网络组件。

**④ `Hide()` 隐身**：classified 不需要渲染，直接隐藏。这是一个引擎级优化——不占用渲染资源。

**⑤ `AddTag("CLASSIFIED")` 标识**：打上 `"CLASSIFIED"` tag，告诉其他系统"**我是一个 classified，不是普通实体**"——在实体搜索、目标选择、UI 查找时会被特殊处理。

#### 第二步：声明 net 变量

紧接着的几十行都是在声明 net 变量：

```lua
inst.image = net_hash(inst.GUID, "inventoryitem.image", "imagedirty")
inst.atlas = net_hash(inst.GUID, "inventoryitem.atlas", "imagedirty")
inst.cangoincontainer = net_bool(inst.GUID, "inventoryitem.cangoincontainer")
...
inst.percentused = net_byte(inst.GUID, "inventoryitem.percentused", "percentuseddirty")
inst.perish = net_smallbyte(inst.GUID, "inventoryitem.perish", "perishdirty")
...
```

**每个 net 变量的三个参数**：

1. **`inst.GUID`**：属于哪个实体
2. **名字字符串**：引擎内部用来识别这个变量的唯一 ID
3. **dirty 事件名**（可选）：当这个变量在客户端发生变化时，在实体上 push 一个对应的事件

我们在 6.3.3 节详细讲各种 net 类型。

#### 第三步：设置默认值

看第 177-195 行：

```177:195:scripts/prefabs/inventoryitem_classified.lua
    inst.image:set(0)
    inst.atlas:set(0)
    inst.cangoincontainer:set(true)
    inst.canonlygoinpocket:set(false)
    inst.canonlygoinpocketorpocketcontainers:set(false)
    inst.src_pos.isvalid:set(false)
    inst.percentused:set(255)
    inst.perish:set(63)
    inst.recharge:set(255)
    inst.rechargetime:set(-2)
    inst.deploymode:set(DEPLOYMODE.NONE)
    inst.deployspacing:set(DEPLOYSPACING.DEFAULT)
    inst.deployrestrictedtag:set(0)
    inst.usegridplacer:set(false)
    inst.attackrange:set(-99)
    inst.walkspeedmult:set(1)
    inst.equiprestrictedtag:set(0)
    inst.moisture:set(0)
	inst.islockedinslot:set(false)
```

**每个 net 变量都显式 `set` 一次**——不是赋值，是**告诉网络层"这个变量有初值，要同步给客户端"**。

**留意两个特殊的默认值**：

- **`percentused:set(255)`**：255 是"**无值**"标记（百分比 0-100，254/255 留作特殊含义）
- **`perish:set(63)`**：63 也是"**无腐烂**"标记

这是 net 变量的常见技巧——**用取值范围外的数值表示"未设置"**，避免开额外 bool 字段。

#### 第四步：Pristine 分界 + 注册服务端/客户端方法

看第 197-221 行：

```197:221:scripts/prefabs/inventoryitem_classified.lua
    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        inst.DeserializePercentUsed = DeserializePercentUsed
        inst.DeserializePerish = DeserializePerish
        inst.DeserializeRecharge = DeserializeRecharge
        inst.DeserializeRechargeTime = DeserializeRechargeTime
        inst.OnEntityReplicated = OnEntityReplicated

        --inst._rechargetask = nil

        --Delay net listeners until after initial values are deserialized
        inst:DoStaticTaskInTime(0, RegisterNetListeners)
        return inst
    end

    inst.persists = false

    inst.SerializePercentUsed = SerializePercentUsed
    inst.SerializePerish = SerializePerish
    inst.ForcePerishDirty = ForcePerishDirty
    inst.SerializeRecharge = SerializeRecharge
    inst.SerializeRechargeTime = SerializeRechargeTime

    return inst
end

return Prefab("inventoryitem_classified", fn)
```

**熟悉的 Pristine 分界**（回顾 5.4 节）——但用法略有不同：

- **客户端分支**：挂上 `DeserializeXxx`（反序列化）方法 + `OnEntityReplicated`（实体反序列化完成回调）+ 延迟注册 dirty 事件监听器
- **服务端分支**：`persists = false`（classified **不进存档**——每次游戏启动重新创建）+ 挂上 `SerializeXxx`（序列化）方法

**注意 `persists = false`**：5.2.5 节讲过"持久性"。classified 不应该被存档——它的数据来自主组件的 `OnLoad`，读档后主组件会重新 push 到 classified。

#### 第五步：整体结构图

```
┌─────────────────────────────────────────────────────────┐
│  inventoryitem_classified.lua                           │
├─────────────────────────────────────────────────────────┤
│  SerializeXxx 函数们（服务端 → classified）               │
│    local function SerializePercentUsed(inst, percent)   │
│      ...                                                 │
│                                                          │
│  DeserializeXxx 函数们（classified dirty → 主实体）       │
│    local function DeserializePercentUsed(inst)          │
│      ...                                                 │
│                                                          │
│  OnEntityReplicated  ← 客户端初始化时找到父实体           │
│                                                          │
│  RegisterNetListeners  ← 注册 dirty 事件监听             │
│                                                          │
│  local function fn()                                    │
│    ① CreateEntity + AddTransform + AddNetwork + Hide   │
│    ② AddTag("CLASSIFIED")                              │
│    ③ 声明 net 变量                                       │
│    ④ 设置默认值                                          │
│    ⑤ SetPristine                                         │
│    ⑥ 客户端：挂 DeserializeXxx / 服务端：挂 SerializeXxx│
│  end                                                    │
│                                                          │
│  return Prefab("inventoryitem_classified", fn)          │
└─────────────────────────────────────────────────────────┘
```

**这就是一个 classified 文件的完整结构**。你自己写一个 classified 时，照这个骨架填就行。

---

### 6.3.3 快速入门：net 变量的基础类型

Classified 的核心就是 net 变量。这一节快速过一遍常用类型。

#### 所有 net 变量的共同特征

```lua
inst.xxx = net_<type>(inst.GUID, "标识符字符串", "脏事件名")
```

**三个参数**：

1. **GUID**：属于哪个实体
2. **标识符**：全实体唯一的字符串名，推荐格式 `"组件名.字段名"`
3. **脏事件名**（可选）：当客户端收到新值时触发的事件名

**所有 net 变量的统一 API**：

- **`netvar:set(v)`**——服务端调用，设置值并标记为脏（触发同步）
- **`netvar:set_local(v)`**——任何端都能调用，仅本地改值不触发同步（优化用）
- **`netvar:value()`**——读取当前值

#### 常用 net 类型速查表

参考 `player_classified.lua` 第 1341-1370 行和 `inventoryitem_classified.lua` 的字段声明：

| 类型 | 值域 | 占用 | 使用场景 |
|------|-----|------|----------|
| `net_bool` | `true/false` | 1 位 | 布尔开关（但更优先用 tag） |
| `net_tinybyte` | 0-7（3 位）| 3 位 | 超小整数（枚举 < 8） |
| `net_smallbyte` | 0-63（6 位）| 6 位 | 小整数（6 位足够） |
| `net_byte` | 0-255 | 8 位 | 百分比 0-100、通用小整数 |
| `net_shortint` | -32768 ~ 32767 | 16 位 | 中等带符号整数 |
| `net_ushortint` | 0-65535 | 16 位 | 血量、饥饿、理智等 0-9999 范围值 |
| `net_int` | -2^31 ~ 2^31-1 | 32 位 | 大整数 |
| `net_uint` | 0 ~ 2^32-1 | 32 位 | 大无符号整数（GUID 引用） |
| `net_float` | IEEE 754 float | 32 位 | 位置、时间戳 |
| `net_hash` | 字符串哈希 | 32 位 | 字符串的哈希（比 net_string 省） |
| `net_string` | 任意字符串 | 可变 | 实际字符串 |
| `net_event` | 无数据 | 0 位（仅触发）| 纯事件信号 |
| `net_entity` | GUID | 32 位 | 其他实体的引用 |

#### 几个实战用法

**① 血量（0-9999）**：用 `net_ushortint`

```1341:1342:scripts/prefabs/player_classified.lua
    inst.currenthealth = net_ushortint(inst.GUID, "health.currenthealth", "healthdirty")
    inst.maxhealth = net_ushortint(inst.GUID, "health.maxhealth", "healthdirty")
```

**② 布尔开关**：优先用 tag，其次用 `net_bool`

```1344:1345:scripts/prefabs/player_classified.lua
    inst.istakingfiredamage = net_bool(inst.GUID, "health.takingfiredamage", "istakingfiredamagedirty")
    inst.istakingfiredamagelow = net_bool(inst.GUID, "health.takingfiredamagelow", "istakingfiredamagelowdirty")
```

**③ 字符串身份标识**：用 `net_hash`

```lua
-- inventoryitem_classified.lua 第 152-153 行
inst.image = net_hash(inst.GUID, "inventoryitem.image", "imagedirty")
inst.atlas = net_hash(inst.GUID, "inventoryitem.atlas", "imagedirty")
```

**为什么用 `net_hash` 而不是 `net_string`？** 因为 image 和 atlas 的值其实是**固定集合里挑选**（就那几百种可能的贴图名字）——用哈希传 4 字节，客户端再反哈希查表，比直接传字符串省带宽。

**④ 百分比**（0-100）：用 `net_byte`

```163:165:scripts/prefabs/inventoryitem_classified.lua
    inst.percentused = net_byte(inst.GUID, "inventoryitem.percentused", "percentuseddirty")
    inst.perish = net_smallbyte(inst.GUID, "inventoryitem.perish", "perishdirty")
    inst.recharge = net_byte(inst.GUID, "inventoryitem.recharge", "rechargedirty")
```

**⑤ 事件信号**：用 `net_event`

```lua
-- 纯粹触发事件，不携带数据（上次我们提到的"force dirty"场景）
inst.ispulseup = net_event(inst.GUID, "ispulseupdirty")
```

#### 脏事件（dirty event）的作用

回到第二个参数——**dirty 事件名**。看一个完整链路：

```
服务端：
inst.classified.percentused:set(50)   ← set 会触发同步
    ↓
网络包发送
    ↓
客户端：
inst.classified.percentused 收到 50
    ↓
触发事件 "percentuseddirty" 在 inst.classified 上
    ↓
RegisterNetListeners 里监听过这个事件 →
    ↓
DeserializePercentUsed(inst.classified) 被调用
    ↓
DeserializePercentUsed 在主实体上 push "percentusedchange" 事件
    ↓
主实体的监听器（如 UI、动画）响应
```

**这种两层事件的设计**——

1. classified 收到 dirty 就触发 classified 上的事件
2. classified 的处理函数（Deserialize）再把事件 push 到**主实体**上

**目的**：**让 classified 和主实体之间解耦**。UI 代码监听主实体的"percentusedchange"事件就行，不用关心背后有 classified 存在。

---

### 6.3.4 进阶：`CLASSIFIED` tag 与网络可见性过滤

#### 第一步：为什么需要 `CLASSIFIED` tag？

如果 classified 和普通实体完全一样，就会引发几个问题：

1. **玩家按 Tab 键打开实体调试界面，能看到一堆 classified 实体——干扰** UI 可读性
2. **AI 搜索附近实体（`FindEntities`）时，会把 classified 也当成目标**——但它没有血、没有 combat，搜到没意义
3. **鼠标点击会选中 classified**——破坏游戏体验

**`CLASSIFIED` tag 就是给这些系统一个"**请忽略我**"的信号**。

看 `consolecommands.lua` 第 1221-1224 行（用于调试选择实体的代码）：

```1221:1224:scripts/consolecommands.lua
			and ent.Network ~= nil
			and not ent:HasTag("CLASSIFIED")
			and not ent:HasTag("INLIMBO")
			then
```

**筛选条件里显式排除 `CLASSIFIED` tag**——调试选择实体时跳过 classified。

#### 第二步：打 tag 的标准姿势

所有 classified 的工厂函数**开头都会打 `CLASSIFIED` tag**：

```lua
-- player_classified.lua 第 1336-1337 行
inst.entity:Hide()
inst:AddTag("CLASSIFIED")

-- inventoryitem_classified.lua 第 149-150 行
inst.entity:Hide()
inst:AddTag("CLASSIFIED")
```

**顺序重要**：**先 `Hide` 再 `AddTag`**——虽然两者独立，但这个顺序是 Klei 的约定（符合"先解决视觉，再解决标识"的设计顺序）。

#### 第三步：网络可见性过滤（只给 owner 看）

除了 `CLASSIFIED` tag 之外，classified 还有一个更强大的机制——**网络可见性过滤**。

想象一个场景：**玩家 A 的背包里有什么，应该只有玩家 A 的客户端知道**——其他玩家的客户端不需要、也不应该收到这些数据。

**这是通过 `Network` 组件的 `SetClassifiedTarget` 方法实现的**。看一下 `inventory_classified.lua`（玩家背包的 classified）的创建：

```lua
-- scripts/prefabs/inventory_classified.lua 的工厂函数里（简化示意）
inst.entity:AddNetwork()
inst.Network:SetClassifiedTarget(owner_player_entity)
```

调用 `SetClassifiedTarget(player)` 后，**这个 classified 实体的网络数据只会发给指定的 player**，其他客户端根本收不到。

**对比两种可见性**：

| 可见性类型 | 做法 | 结果 |
|-----------|------|------|
| **所有人可见** | 普通 AddNetwork，不 SetClassifiedTarget | 所有客户端都能收到 net 变量更新 |
| **只给 owner 可见** | `inst.Network:SetClassifiedTarget(owner)` | 只有 owner 客户端能收到，其他客户端字段永远是默认值 |

#### 第四步：实际应用

**例子 1：玩家血量**——所有人可见（`player_classified`）

玩家的血量其他玩家也要看到（显示玩家列表的血条、判断目标血量低）——所以 `player_classified` **不**设 ClassifiedTarget，所有客户端都能看到。

**例子 2：背包内容**——只给 owner 可见（`inventory_classified`）

背包里具体有什么物品、什么槽位、每个堆叠多少——**只有玩家自己**的客户端需要知道。所以 `inventory_classified` 在创建时**设置 ClassifiedTarget 为 owner**，其他玩家收不到这些数据——既保护隐私（多人游戏里不能看对方背包）、又节省带宽。

**例子 3：物品通用数据**——所有人可见（`inventoryitem_classified`）

物品的耐久、腐烂度这些——其他玩家捡起来用时也要知道。所以 `inventoryitem_classified` 也是所有人可见。

> **开发逻辑**：可见性设计是**隐私 + 带宽**的双重考量——看不到的数据根本不发送，既保护玩家、又省带宽。Klei 通过 classified + SetClassifiedTarget 把这种需求做成了语言原语，你写 Mod 时只要按这个模式组织数据就行。

---

### 6.3.5 进阶：父子关系——`SetParent` 让 classified 跟随 owner

Classified 和主实体之间通过 **Entity 层的父子关系**建立绑定——这是最重要的一层组织关系。

#### 第一步：父子关系是怎么建立的？

看 `inventoryitem_replica.lua` 第 13-14 行：

```13:14:scripts/components/inventoryitem_replica.lua
        self.classified = SpawnPrefab("inventoryitem_classified")
        self.classified.entity:SetParent(inst.entity)
```

**两行代码**：

1. **SpawnPrefab** 创建 classified 实体
2. **`SetParent(inst.entity)`** 把 classified 的父亲设为主实体

**父子关系做了什么？**

- **位置跟随**：子实体自动跟着父实体移动（这就是为什么 classified 要 `AddTransform`——才能被父实体的 Transform 带着跑）
- **休眠同步**：父实体休眠时子实体也休眠——减少网络同步（6.2 节讲过 sleep 优化）
- **销毁级联**：父实体销毁时子实体也自动销毁——省得手动清理
- **场景跨越共同**：传送、进入洞穴等场景变换，父子一起走

#### 第二步：客户端如何找到父实体？

服务端创建 classified 时设置了 `SetParent`——这个关系会通过网络同步给客户端。客户端在 `OnEntityReplicated` 钩子里查询父实体。

看 `inventoryitem_classified.lua` 第 7-15 行：

```7:15:scripts/prefabs/inventoryitem_classified.lua
local function OnEntityReplicated(inst)
    inst._parent = inst.entity:GetParent()
    if inst._parent == nil then
        print("Unable to initialize classified data for inventory item")
	elseif not inst._parent:TryAttachClassifiedToReplicaComponent(inst, "inventoryitem") then
        inst._parent.inventoryitem_classified = inst
        inst.OnRemoveEntity = OnRemoveEntity
    end
end
```

**执行流程**：

1. **`inst.entity:GetParent()`**：获取父实体——**这个父实体是服务端通过 `SetParent` 设定的**，客户端通过网络同步拿到
2. **`TryAttachClassifiedToReplicaComponent(inst, "inventoryitem")`**：尝试把 classified 挂给主实体的 `inst.replica.inventoryitem`——如果成功（主实体已经有对应 replica），就挂上
3. **否则**：`inst._parent.inventoryitem_classified = inst`——直接挂到主实体的字段上，等主实体创建 replica 时通过 `inst.inventoryitem_classified` 拿到它

**`TryAttachClassifiedToReplicaComponent`**（在 `entityreplica.lua` 第 88-95 行）：

```88:95:scripts/entityreplica.lua
function EntityScript:TryAttachClassifiedToReplicaComponent(classified, name)
	local cmp = rawget(self.replica, "_")[name]
	if cmp then
		cmp:AttachClassified(classified)
		return true
	end
	return false
end
```

**注意时序问题**：在客户端，**classified 和 replica 的创建顺序是不确定的**——有可能 replica 先创建（找不到 classified，等着），也可能 classified 先创建（找不到 replica，存在主实体字段上等着）。这段代码处理两种时序：

- **replica 先到 classified 后到**：classified 的 `OnEntityReplicated` 成功调用 `AttachClassified`
- **classified 先到 replica 后到**：classified 存到 `inst.inventoryitem_classified`，replica 的构造函数里去读（回顾 6.2.2 节的 replica 构造代码）

这是 Klei 对"分布式时序不确定性"的鲁棒处理。

#### 第三步：销毁时的级联

看 `inventoryitem_classified.lua` 第 1-5 行：

```1:5:scripts/prefabs/inventoryitem_classified.lua
local function OnRemoveEntity(inst)
    if inst._parent ~= nil then
        inst._parent.inventoryitem_classified = nil
    end
end
```

**classified 被销毁时**：清理 `inst._parent.inventoryitem_classified = nil`——把主实体上对这个 classified 的引用置空，避免悬挂引用（主实体如果继续活着，它不该持有一个已销毁的 classified）。

#### 第四步：父子关系的两种用法

**用法 1**：物品独立 classified（每个物品一个）

```
物品实体（刀）
  ├── inventoryitem_classified（子）
        ├── percentused
        ├── perish
        └── ...
```

**特点**：每个物品一个 classified——游戏里 1000 件物品就有 1000 个 classified。虽然多，但每个都很小，总成本可控。

**用法 2**：玩家共享 classified（所有相关组件共用一个）

```
玩家实体（Wilson）
  ├── player_classified（子）
        ├── health.currenthealth
        ├── health.maxhealth
        ├── hunger.currenthunger
        ├── hunger.maxhunger
        ├── sanity.currentsanity
        └── ...（50+ 字段）
```

**特点**：所有玩家相关组件都共用这一个 classified——节省网络实体数（不像物品那样每个组件一个）。

> **开发逻辑**：**物品量多、但每个简单** → 一物一 classified。**玩家量少、但数据复杂** → 多组件共享 classified。这是针对两种不同数据规模的最优解。你写 Mod 时要根据自己的场景选型——比如"为每个怪物加一个专属 classified"vs"为所有怪物共用一个 boss_classified"——各有优劣。

---

### 6.3.6 进阶：Serialize / Deserialize 模式

Classified 和主实体之间的数据流是**双向的**——服务端把主组件的状态变化"**序列化**"进 classified，客户端从 classified "**反序列化**"出状态并通知主实体。

#### 第一步：Serialize（服务端 → classified）

看 `inventoryitem_classified.lua` 第 23-25 行：

```23:25:scripts/prefabs/inventoryitem_classified.lua
local function SerializePercentUsed(inst, percent)
    inst.percentused:set((percent == nil and 255) or (percent <= 0 and 0) or math.clamp(math.floor(percent * 100 + .5), 1, 100))
end
```

**做的事**：接收一个 `percent`（0-1 浮点数），转换成 0-100 的字节值放到 net 变量。

**几个技巧**：

- **`nil → 255`**：percent 是 nil 表示"无意义"——用 255 这个特殊值标记
- **`<=0 → 0`**：clamp 下限
- **`math.floor(percent * 100 + .5)`**：乘 100 后四舍五入（`+ .5` 再 floor 是 Lua 传统的四舍五入写法）
- **`math.clamp(x, 1, 100)`**：clamp 到 1-100（避免和 0/255 的特殊值冲突）

**调用时机**：主组件的字段变化时——通常在 replica 的 SetXxx 里调用。看 `inventoryitem_replica.lua` 第 16 行：

```lua
inst:ListenForEvent("percentusedchange", function(inst, data) self.classified:SerializePercentUsed(data.percent) end)
```

**整个链路**：

```
服务端：主组件 finiteuses 的耐久变化
  ↓
finiteuses 组件 push "percentusedchange" 事件，data = {percent = 0.5}
  ↓
inventoryitem_replica 监听器触发 → self.classified:SerializePercentUsed(0.5)
  ↓
Serialize 函数里：inst.percentused:set(50)（把 0.5 转成 50）
  ↓
net 变量标记脏，下一帧同步到客户端
```

#### 第二步：Deserialize（classified dirty → 主实体）

客户端收到 net 变量更新后的流程。看 `inventoryitem_classified.lua` 第 27-31 行：

```27:31:scripts/prefabs/inventoryitem_classified.lua
local function DeserializePercentUsed(inst)
    if inst.percentused:value() ~= 255 and inst._parent ~= nil then
        inst._parent:PushEvent("percentusedchange", { percent = inst.percentused:value() / 100 })
    end
end
```

**做的事**：读 net 变量（已经是最新值），转换回 0-1 浮点数，在**主实体上 push 事件**。

**关键两句**：

- **`inst.percentused:value() ~= 255`**：跳过"无意义"值（255 是 Serialize 里用的特殊标记，客户端不处理）
- **`inst._parent:PushEvent(...)`**：把事件 push 到**主实体**上——这样主实体的 UI、动画系统能响应

**调用时机**：通过 dirty 事件监听器触发。看 `RegisterNetListeners`（第 131-140 行）：

```131:140:scripts/prefabs/inventoryitem_classified.lua
local function RegisterNetListeners(inst)
    inst:ListenForEvent("imagedirty", OnImageDirty)
    inst:ListenForEvent("percentuseddirty", DeserializePercentUsed)
    inst:ListenForEvent("perishdirty", DeserializePerish)
    inst:ListenForEvent("rechargedirty", DeserializeRecharge)
    inst:ListenForEvent("rechargetimedirty", DeserializeRechargeTime)
	inst:ListenForEvent("inventoryitem_stacksizedirty", OnStackSizeDirty, inst._parent)
    inst:ListenForEvent("iswetdirty", OnIsWetDirty, inst._parent)
    inst:ListenForEvent("isacidsizzlingdirty", OnIsAcidSizzlingDirty, inst._parent)
end
```

**`"percentuseddirty"`** 是 net 变量声明时的第三个参数——客户端收到 dirty 就会触发这个事件。

#### 第三步：为什么要 Serialize/Deserialize？——压缩与解压

本质上，Serialize/Deserialize 是**浮点数 ↔ 整数（字节）**的转换：

```
0.0 - 1.0 浮点数
   ↓ Serialize：× 100 → 整数
0 - 100 整数
   ↓ 网络传输（8 bits）
0 - 100 整数
   ↓ Deserialize：÷ 100 → 浮点数
0.0 - 1.0 浮点数
```

**为什么不直接传 float？** 因为 float 是 32 bits，byte 是 8 bits——对于 0-1 百分比，byte 精度完全足够（1%），**省 75% 带宽**。

这就是 classified 设计里的"**精度交换**"模式——**用合适的整数类型代替浮点数**，牺牲一些精度换带宽。

#### 第四步：`ForcePerishDirty` —— 强制触发同步

`inventoryitem_classified.lua` 第 37-41 行有一个特殊函数：

```37:41:scripts/prefabs/inventoryitem_classified.lua
--V2C: used to force color refresh when spoilage changes around 50%/20%
local function ForcePerishDirty(inst)
    inst.perish:set_local(inst.perish:value())
    inst.perish:set(inst.perish:value())
end
```

**做的事**：**先 `set_local` 改成另一个值，再 `set` 改回原值**——强制网络标记脏，触发客户端的 dirty 事件。

**为什么要这样？** 因为 net 变量的 `:set` 只在**值真正改变时**才标记脏——如果服务端反复 set 同一个值，客户端什么事件都不会触发。

但**有些场景需要"无值变化的强制刷新"**——比如当腐烂度刚好跨越了 50%/20% 这种视觉阈值时，需要重新计算颜色（`V2C:` 注释说明了这点）。这时就用 `set_local + set` 这个"假动作"强制触发。

> **新手记忆**：**Serialize 是"打包浮点数"，Deserialize 是"拆包还原浮点数"**。能用整数表达的就别用浮点数，这是饥荒联机版省带宽的核心思路。

---

### 6.3.7 老手进阶：`player_classified` 的整体架构

`scripts/prefabs/player_classified.lua` 是官方最大的 classified 文件——**1669 行**，挂了 **50+ 个 net 变量**。理解它的组织哲学对写自定义 classified 是最大的营养。

#### 第一步：文件组织

打开文件看结构：

```
player_classified.lua
├── 第 1-10 行：依赖、局部变量
├── 第 11-20 行：辅助函数 SetValue / SetDirty / PushPausePredictionFrames
├── 第 27-93 行：各组件事件回调（OnHealthDelta / OnHungerDelta 等）
├── 第 94-1000 行：业务逻辑函数（各种 SerializeXxx / DeserializeXxx / OnXxxDirty）
├── 第 1000-1300 行：RegisterNetListeners、AttachClassified 等辅助设施
├── 第 1330-1669 行：主工厂函数 fn()
│     ├── CreateEntity + AddTransform + AddNetwork + Hide
│     ├── AddTag("CLASSIFIED")
│     ├── 声明 50+ 个 net 变量（分组：Health / Hunger / Sanity / ...）
│     ├── 设置默认值
│     ├── SetPristine
│     ├── 客户端/服务端分支
│     └── return inst
└── return Prefab("player_classified", fn)
```

**规律**：按"**业务逻辑 → 注册与辅助 → 实体工厂**"的顺序组织。业务逻辑永远在最上，主函数永远在最下。

#### 第二步：字段分组

看 `player_classified.lua` 第 1341-1370 行的字段声明节选：

```1341:1370:scripts/prefabs/player_classified.lua
    inst.currenthealth = net_ushortint(inst.GUID, "health.currenthealth", "healthdirty")
    inst.maxhealth = net_ushortint(inst.GUID, "health.maxhealth", "healthdirty")
    inst.healthpenalty = net_byte(inst.GUID, "health.penalty", "healthdirty")
    inst.istakingfiredamage = net_bool(inst.GUID, "health.takingfiredamage", "istakingfiredamagedirty")
    inst.istakingfiredamagelow = net_bool(inst.GUID, "health.takingfiredamagelow", "istakingfiredamagelowdirty")
    inst.issleephealing = net_bool(inst.GUID, "health.healthsleep")
    inst.ishealthpulseup = net_bool(inst.GUID, "health.dodeltaovertime(up)", "healthdirty")
    inst.ishealthpulsedown = net_bool(inst.GUID, "health.dodeltaovertime(down)", "healthdirty")
	inst.lunarburnflags = net_tinybyte(inst.GUID, "health.lunarburnflags", "lunarburnflagsdirty")
```

**命名约定**：

- **字段名就是字面意思**：`currenthealth`、`maxhealth`——和 `inst.components.health` 里的字段同名
- **标识字符串前缀**：**`health.xxx`**——表明这是 health 相关的字段
- **dirty 事件名复用**：多个字段共享同一个 dirty 事件名（如 `"healthdirty"`）——节省监听器数量

**为什么多个字段共享 dirty 事件？** 因为：

- **UI 刷新一般是整体的**：血条显示会同时用 currenthealth 和 maxhealth——一个触发就重绘整个血条
- **节省回调数量**：3 个字段共用 1 个监听器，比 3 个字段 3 个监听器要轻

#### 第三步：`player_classified` 的可见性——"所有人可见"

注意 `player_classified.lua` 里**没有调用 `SetClassifiedTarget`**——所以它是**所有客户端都能看到的**。

**原因**：玩家的 HUD 信息（血量、饥饿、理智等）不只自己看——其他玩家也要看：

- 查看玩家列表
- 判断队友状态
- 做"帮队友续命"的判断

**对比**：`inventory_classified.lua` 是 **只给 owner 看的**——所以要 SetClassifiedTarget。

#### 第四步：`player_classified` 的生命周期

**创建时机**：玩家实体初始化时（在 `player_common.lua` 里），SpawnPrefab 一个 `player_classified` 并设为 player 的子。

**销毁时机**：玩家实体销毁时（退出游戏、换角色）自动级联销毁——因为是子实体。

**持久化**：**`persists = false`**——不进存档。下次游戏启动时重新创建，数据由主组件的 OnLoad 重新填进去。

#### 第五步：如何追踪一个字段的完整链路？

以"玩家 A 被打了一拳，血量从 100 变成 50，玩家 B 的客户端是怎么看到的"为例——这是理解整套机制的黄金路径：

```
服务端（玩家 A 所在服务端）：
1. combat 组件调用 health.DoDelta(-50)
2. health.lua 的 SetVal 设置 self.currenthealth = 50
3. Class 的 onset 触发 oncurrenthealth(self, 50)
4. oncurrenthealth（health.lua 第 18 行）调用 self.inst.replica.health:SetCurrent(50)
5. health_replica.SetCurrent 调用 self.classified:SetValue("currenthealth", 50)
6. player_classified 的 SetValue（第 11-14 行）调用 inst.currenthealth:set(50)
7. net 变量标记脏

网络层：
8. 下一个 tick 发出更新包

玩家 B 的客户端：
9. 收到 net 变量更新，inst.currenthealth 值变为 50
10. 触发 "healthdirty" 事件
11. player_classified 的 RegisterNetListeners 里监听的处理函数触发
12. 处理函数在 player 实体上 PushEvent（比如 "healthchange"）
13. HUD 监听器响应，重绘血条
```

**13 步链路，每一步都是独立可替换的模块**——这就是组件化设计的威力。你可以在任何一步插入自己的代码（加一个 buff 监听、改 UI 显示、拦截 SetValue 等）。

---

### 6.3.8 老手进阶：自定义 classified 的六个陷阱

假设你写了一个"**灵能**"组件 `myability`（6.2.8 节示范过 replica），要配套一个 `myability_classified`——这里是你很容易栽的六个坑。

#### 陷阱 1：忘记在 Prefab 里注册 classified

```lua
-- ❌ 错：只写了 _classified.lua 文件，但没注册到 PrefabFiles
PrefabFiles = {
    "myability_mod",
    -- "myability_classified",   ← 忘了加
}
```

**症状**：`SpawnPrefab("myability_classified")` 报错 `Can't find prefab`。

**解决**：**所有 classified 的 lua 文件都必须在 modmain 的 `PrefabFiles` 里登记**。

---

#### 陷阱 2：SetParent 忘了或写反

```lua
-- ❌ 错：没 SetParent，classified 成了"孤儿"
local classified = SpawnPrefab("myability_classified")
-- 漏了 classified.entity:SetParent(inst.entity)

-- ❌ 错：写反了
inst.entity:SetParent(classified.entity)   -- 主实体变成 classified 的儿子？！
```

**症状**：
- 没 SetParent：classified 不跟随主实体休眠/移动/销毁——内存泄漏
- 写反：主实体会被当作 classified 的子，各种场景切换会崩

**解决**：**始终是 `classified.entity:SetParent(mainentity.entity)`**——classified 是**子**，主实体是**父**。

---

#### 陷阱 3：客户端代码访问 classified 但没判空

```lua
-- ❌ 错：时序不可控
function MyAbility:GetMana()
    return self.classified.currentmana:value()   -- 崩：self.classified 可能 nil
end

-- ✅ 对：防御式
function MyAbility:GetMana()
    if self.classified ~= nil then
        return self.classified.currentmana:value()
    else
        return 0  -- 默认值
    end
end
```

**原因**：6.3.5 节讲过，classified 在客户端的 attach 是异步的——有窗口期 `self.classified == nil`。客户端代码必须**所有地方都防御式判空**。

---

#### 陷阱 4：net 变量名全局冲突

```lua
-- myability_classified.lua
inst.currentmana = net_ushortint(inst.GUID, "health.currenthealth", "dirtyhealth")  -- ❌ 用了 health 的名字！
```

**net 变量的标识符是在 GUID 维度唯一的**——但如果你复制别人代码时没改名字，有概率撞名导致引擎错误覆盖。

**解决**：**名字永远带上 Mod 前缀**：

```lua
inst.currentmana = net_ushortint(inst.GUID, "mymod.myability.currentmana", "currentmanadirty")
```

---

#### 陷阱 5：SerializeXxx 里用了浮点数

```lua
-- ❌ 错：精度浪费
local function SerializeMana(inst, mana)
    inst.currentmana:set(mana)   -- mana 是 0-100 浮点数，但 net 是 ushortint
end

-- ✅ 对：四舍五入
local function SerializeMana(inst, mana)
    inst.currentmana:set(math.floor(mana + .5))
end
```

**原因**：net_ushortint 的 `:set` 如果收到非整数参数，有些引擎实现会 `math.floor`（丢弃小数）、有些会 `math.ceil`、有些直接报错——不可靠。**显式 `math.floor(x + .5)` 做四舍五入**是最稳的。

---

#### 陷阱 6：Deserialize 在客户端改服务端字段

```lua
-- ❌ 错：客户端不能写服务端组件
local function DeserializeMana(inst)
    if inst._parent and inst._parent.components.myability then
        inst._parent.components.myability.currentmana = inst.currentmana:value()   -- 客户端这里 components.myability == nil
    end
end

-- ✅ 对：通过事件通知，让主组件或 replica 处理
local function DeserializeMana(inst)
    if inst._parent ~= nil then
        inst._parent:PushEvent("manachange", { mana = inst.currentmana:value() })
    end
end
```

**原因**：客户端根本没有 `components.myability`（客户端 components 几乎空）——直接改会崩。**客户端和 classified 通信的唯一合法方式是 `PushEvent`**，让监听事件的代码（通常是 UI）处理。

#### 第七步：自定义 classified 的最小骨架

结合所有经验，一个 **Mod 自定义 classified 的最小可用模板**：

```lua
-- scripts/prefabs/myability_classified.lua

local function SerializeMana(inst, mana)
    inst.currentmana:set(math.floor(mana + .5))
end

local function DeserializeMana(inst)
    if inst._parent ~= nil then
        inst._parent:PushEvent("manachange", { mana = inst.currentmana:value() })
    end
end

local function RegisterNetListeners(inst)
    inst:ListenForEvent("currentmanadirty", DeserializeMana)
end

local function OnRemoveEntity(inst)
    if inst._parent ~= nil then
        inst._parent.myability_classified = nil
    end
end

local function OnEntityReplicated(inst)
    inst._parent = inst.entity:GetParent()
    if inst._parent == nil then
        print("Unable to initialize myability classified")
    elseif not inst._parent:TryAttachClassifiedToReplicaComponent(inst, "myability") then
        inst._parent.myability_classified = inst
        inst.OnRemoveEntity = OnRemoveEntity
    end
end

local function fn()
    local inst = CreateEntity()

    if TheWorld.ismastersim then
        inst.entity:AddTransform()
    end
    inst.entity:AddNetwork()
    inst.entity:Hide()
    inst:AddTag("CLASSIFIED")

    inst.currentmana = net_ushortint(inst.GUID, "mymod.myability.currentmana", "currentmanadirty")
    inst.maxmana = net_ushortint(inst.GUID, "mymod.myability.maxmana", "maxmanadirty")

    inst.currentmana:set(100)
    inst.maxmana:set(100)

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        inst.OnEntityReplicated = OnEntityReplicated
        inst:DoStaticTaskInTime(0, RegisterNetListeners)
        return inst
    end

    inst.persists = false
    inst.SerializeMana = SerializeMana

    return inst
end

return Prefab("myability_classified", fn)
```

**可以直接复制使用**——把"myability" 改成你的组件名，把字段改成你的字段。

---

### 6.3.9 小结

- **Classified 是"一个专门承载 net 变量的隐形联网实体"**——它存在的意义是为主实体的某类数据提供**独立的网络载体**，解决"字段混杂、可见性难控、生命周期耦合"三大问题。
- **Classified 实体的标志性特征**：`AddNetwork + Hide + AddTag("CLASSIFIED") + SetParent` 四件套，外加 `persists = false`（不入存档）。
- **`CLASSIFIED` tag 是"请忽略我"信号**——`FindEntities`、调试 UI、实体选择等系统都会跳过 classified。
- **可见性过滤**：默认所有人可见；`inst.Network:SetClassifiedTarget(owner)` 让 classified **只给指定玩家可见**——用于背包这类隐私数据。
- **net 变量类型速查**：`net_bool`（1 bit）、`net_tinybyte`（3 bits）、`net_smallbyte`（6 bits）、`net_byte`（8 bits）、`net_ushortint`（16 bits）、`net_float`（32 bits）、`net_hash`（字符串哈希）、`net_string`（真字符串）、`net_event`（纯事件）。**能小就小**。
- **脏事件（dirty event）机制**：net 变量声明时的第三个参数是事件名——客户端收到更新时自动触发这个事件，供 `RegisterNetListeners` 监听。多字段共享同一事件是常见优化。
- **父子关系的三个好处**：位置跟随、休眠同步、销毁级联——这是用 `classified.entity:SetParent(mainentity.entity)` 一行代码换来的。
- **客户端挂载时序不确定**：`OnEntityReplicated` 通过 `TryAttachClassifiedToReplicaComponent` 处理"replica 先到 vs classified 先到"两种情况——是 Klei 对分布式时序的鲁棒处理。
- **Serialize/Deserialize 模式**：服务端 Serialize（浮点 → 整数 → net 变量），客户端 Deserialize（net 变量 → 整数 → 浮点 → PushEvent 到主实体）。**用精度换带宽**是核心设计。
- **`player_classified` 的规模**：1669 行、50+ 字段、所有人可见——但仍保持清晰的"按组件分组、命名统一、多字段共享 dirty 事件"的组织方式。
- **自定义 classified 六陷阱**：忘了登记到 PrefabFiles、SetParent 写反/遗漏、客户端不判空崩溃、net 变量名字撞名、Serialize 用浮点、Deserialize 客户端写服务端字段——每个都是实际写时会栽的跟头。

> **下一节预告**：6.4 节我们讲 **Timer 组件**——"如何优雅地管理组件级的多个计时任务"。它和 `inst:DoPeriodicTask` 的手动计时有什么区别？多个 Mod 往同一个组件加计时任务时会不会互相干扰？答案在下一节。

## 6.4 Timer 组件——组件级计时器的正确使用

### 本节导读

前面两节（6.2 Replica / 6.3 Classified）我们讲了**联机同步的底层机制**——那是相对复杂的话题。本节换一个简单但极实用的主题：**Timer 组件**。

几乎每一个带"**定时行为**"的 Prefab 都会用到 Timer——比如魏斯（Walter）的 Woby 攻击冷却、WX-78 的充电再生节奏、Tillweed Salve 的 buff 持续时间、Winona 电池的过载计时——都靠它。但 Timer 也是**最常被误用**的组件之一：很多 Mod 用 `DoPeriodicTask` 硬写计时，结果存档读档后 buff 消失、控制台 `c_reset` 后定时器还在鬼祟运行、多个 Mod 的定时器互相覆盖——这些问题 99% 都是"**没用 Timer 组件**"造成的。

> **新手**从 6.4.1-6.4.3 入手，学会 `StartTimer / StopTimer / TimerExists / timerdone 事件`——只要会这五个就能做大多数"倒计时 buff / 冷却时间"功能；**进阶读者**从 6.4.4-6.4.6 深入 Pause/Resume/LongUpdate 高级操作、OnSave/OnLoad 自动存档机制、以及 Timer 和原生 `DoTaskInTime` 的区别；**老手**跳到 6.4.7-6.4.8 学习真实 Prefab（Walter、WX-78、Winona 电池、Wortox）的 Timer 使用模式，以及新手/进阶都容易踩的五个陷阱。

**前置**：本节假设你已经读过 6.1 节（组件生命周期）和 5.2.4 节（实体 OnSave/OnLoad），对组件的存档机制有基本了解。

---

### 6.4.1 快速入门：为什么需要 Timer 组件？

#### 场景 1：一个 buff 要持续 60 秒，然后自动移除

**不用 Timer 组件的写法（原始）**：

```lua
-- buff 附加上来时
inst:DoTaskInTime(60, function()
    inst:RemoveDebuff("speed_buff")
end)
```

**这有什么问题？**

1. **无法查询"还剩多久"**：UI 想在屏幕上显示"30 秒后结束"——拿不到剩余时间
2. **无法暂停**：游戏菜单暂停、玩家被冻结——task 还在倒计时
3. **无法存档**：玩家退出游戏再进来，这个 task 消失了，buff 永远不会结束（变成永久 buff）
4. **无法手动中止**：玩家喝了"净化药水"想立刻消除所有 buff——找不到这个 task 怎么取消

**用 Timer 组件的写法**：

```lua
-- buff 附加时
inst.components.timer:StartTimer("speed_buff_duration", 60)

-- 监听 timerdone
inst:ListenForEvent("timerdone", function(inst, data)
    if data.name == "speed_buff_duration" then
        inst:RemoveDebuff("speed_buff")
    end
end)
```

**立即获得四大好处**：

1. **查询剩余时间**：`inst.components.timer:GetTimeLeft("speed_buff_duration")`
2. **暂停/恢复**：`PauseTimer / ResumeTimer`
3. **自动存档**：存档时 Timer 组件的 OnSave 会把"剩余 32.5 秒"写进存档，读档恢复
4. **手动中止**：`StopTimer("speed_buff_duration")` 一行搞定

#### 场景 2：一个怪物每隔 5 秒重新选择目标

**不用 Timer 的写法**：

```lua
inst.retargettask = inst:DoPeriodicTask(5, function()
    inst.components.combat:TryRetarget()
end)
```

这在普通怪物身上**确实更简单、更合适**——我们**并不想**让这个任务能暂停/恢复/存档——它是纯粹的运行时行为，跟随实体生死即可。

**所以 Timer 不是"万能替代"**——它是**对"需要存档/可查询/可暂停"的倒计时类场景**的最佳方案。

#### 第三步：两种任务的选型

| 场景 | 推荐 |
|------|------|
| buff / debuff 持续时间 | **Timer** |
| 冷却时间（技能、武器） | **Timer**（可以查 UI 进度） |
| 物品耐久归零倒计时 | **Timer** |
| 延迟执行某动作（触发后 3 秒生效） | **Timer** 或 `DoTaskInTime`（看是否需要存档） |
| 心跳任务（每帧/每秒检查某条件） | `DoPeriodicTask` |
| 实体临时高亮（0.5 秒后消失） | `DoTaskInTime`（不需要存档） |
| 预判攻击（用 `DoTaskInTime` 立即触发） | `DoTaskInTime` |

**核心判别标准**：

- **需要 UI 显示进度 / 可查询剩余时间** → Timer
- **需要存档恢复** → Timer
- **可能需要暂停** → Timer
- **纯粹的瞬态任务（随实体销毁就没了）** → 原生 task

> **新手记忆**：**所有"玩家眼里感知到的倒计时"都用 Timer**。玩家看不到的内部状态更新用原生 task。

---

### 6.4.2 快速入门：Timer 的基本 API

打开 `scripts/components/timer.lua`——它只有 **159 行**，是饥荒最精简的组件之一。我们一起过 Timer 的核心 API。

#### Timer 组件的状态存储

看第 1-4 行：

```1:4:scripts/components/timer.lua
local Timer = Class(function(self, inst)
    self.inst = inst
    self.timers = {}
end)
```

**只有一个字段 `self.timers`**——这是一张表，以**计时器名称**为键，存储所有活跃的计时器。每个计时器的数据结构是：

```lua
self.timers["speed_buff"] = {
    timer = <scheduler_task>,    -- 底层的 DoTaskInTime 任务引用
    timeleft = 30.5,              -- 剩余时间（用于暂停恢复）
    end_time = 12345.5,           -- 绝对结束时间戳
    initial_time = 60,            -- 最初设置的总时长
    paused = false,               -- 是否暂停中
}
```

**注意 `self.timers` 是以字符串名字为 key**——这就是 Timer 组件的"**命名复用**"机制：同一个组件上可以有多个计时器，只要名字不同。

#### API 1：`StartTimer` —— 启动一个计时器

看第 36-54 行：

```36:54:scripts/components/timer.lua
function Timer:StartTimer(name, time, paused, initialtime_override)
    if self:TimerExists(name) then
        print("A timer with the name ", name, " already exists on ", self.inst, "!")
        return
    end

    self.timers[name] =
    {
        timer = self.inst:DoTaskInTime(time, OnTimerDone, self, name),
        timeleft = time,
        end_time = GetTime() + time,
        initial_time = initialtime_override or time,
        paused = false,
    }

    if paused then
        self:PauseTimer(name)
    end
end
```

**签名**：`StartTimer(name, time, paused, initialtime_override)`

| 参数 | 类型 | 含义 |
|------|------|------|
| `name` | string | 计时器的唯一名称（组件范围内） |
| `time` | number | 剩余时长（秒） |
| `paused` | bool（可选）| 是否立即进入暂停状态 |
| `initialtime_override` | number（可选）| 覆盖 initial_time（总时长，用于 GetTimeElapsed 计算） |

**注意第一行的防御**：`if self:TimerExists(name)`——**如果同名计时器已存在，直接返回 + 打印警告**。你**不能简单地"重启"一个计时器**——要先 `StopTimer` 再 `StartTimer`。

**使用示例**（来自 `scripts/prefabs/tillweedsalve.lua` 第 83-84 行）：

```83:84:scripts/prefabs/tillweedsalve.lua
    inst.components.timer:StopTimer("regenover")
    inst.components.timer:StartTimer("regenover", BUFF_DURATION)
```

**这就是"刷新 buff 持续时间"的标准写法**——先停止再启动。

#### API 2：`StopTimer` —— 取消一个计时器

看第 56-66 行：

```56:66:scripts/components/timer.lua
function Timer:StopTimer(name)
    if not self:TimerExists(name) then
        return
    end

    if self.timers[name].timer ~= nil then
        self.timers[name].timer:Cancel()
        self.timers[name].timer = nil
    end
    self.timers[name] = nil
end
```

**做的事**：
1. 检查计时器存不存在（不存在就静默返回——不报错）
2. 取消底层的 scheduler task
3. 从 `self.timers` 表里删除这个计时器

**注意**：**`StopTimer` 不会触发 `timerdone` 事件**——它是"手动中止"，而不是"正常结束"。

#### API 3：`TimerExists` —— 检查存在

看第 27-29 行：

```27:29:scripts/components/timer.lua
function Timer:TimerExists(name)
    return self.timers[name] ~= nil
end
```

**一行——但极常用**。几乎每个使用 Timer 的代码都有 `TimerExists` 判断：

- **启动前检查**：`if not inst.components.timer:TimerExists("xxx") then StartTimer(...) end`——避免重复启动
- **触发前检查**：冷却时间还没结束就不让技能释放——`if not inst.components.timer:TimerExists("cooldown") then DoSkill() end`

#### API 4：`GetTimeLeft / GetTimeElapsed` —— 查询时间

```95:102:scripts/components/timer.lua
function Timer:GetTimeLeft(name)
    if not self:TimerExists(name) then
        return
    elseif not self:IsPaused(name) then
        self.timers[name].timeleft = self.timers[name].end_time - GetTime()
    end
    return self.timers[name].timeleft
end
```

**`GetTimeLeft`** 返回剩余秒数——**注意它会懒更新**：只有没暂停的情况下才根据 `end_time - GetTime()` 重新计算 `timeleft`。如果暂停，直接返回之前保存的值（凝固的剩余时间）。

```116:120:scripts/components/timer.lua
function Timer:GetTimeElapsed(name)
    return self:TimerExists(name)
        and (self.timers[name].initial_time or 0) - self:GetTimeLeft(name)
        or nil
end
```

**`GetTimeElapsed`** 返回已过去的秒数——用 `initial_time - GetTimeLeft` 算出来。

**使用场景**：UI 要画一个进度条：

```lua
local percent = inst.components.timer:GetTimeElapsed("cooldown") / <总冷却时间>
-- 画进度条 percent 从 0 到 1
```

---

### 6.4.3 快速入门：`timerdone` 事件与 OnTimerDone 回调

Timer 组件**不直接接受回调函数**——它通过**事件**机制通知你。

#### 核心机制：`OnTimerDone` 内部回调

看第 31-34 行：

```31:34:scripts/components/timer.lua
local function OnTimerDone(inst, self, name)
    self:StopTimer(name)
    inst:PushEvent("timerdone", { name = name })
end
```

**两件事**：
1. **清理自己**：调用 `StopTimer` 从 `self.timers` 里移除
2. **触发 `timerdone` 事件**：data 是 `{ name = "xxx" }`

#### 典型的监听模式

**Pattern 1：统一分发 OnTimerDone 回调**

看 `scripts/prefabs/walter.lua` 第 625-629 行：

```625:629:scripts/prefabs/walter.lua
local function OnTimerDone(inst, data)
	if data and data.name == "wobybuck" then
		inst._wobybuck_damage = 0
	end
end
```

然后在工厂函数里：

```lua
inst:ListenForEvent("timerdone", OnTimerDone)
```

**这是最常见的模式**——**一个 `OnTimerDone` 函数 + 内部 `if data.name == "xxx" then`** 分发。组件上有几个不同名字的计时器，就有几个分支。

看 `scripts/prefabs/winona_battery_high.lua` 第 763-773 行的多分支：

```763:773:scripts/prefabs/winona_battery_high.lua
local function OnTimerDone(inst, data)
	if data then
		if data.name == "shardloaddelay" then
			inst.components.timer:ResumeTimer("shardload")
			StartUpdatingShardLoad(inst)
		elseif data.name == "shardload" then
			StopUpdatingShardLoad(inst)
			OnUpdateShardLoad(inst)
		elseif data.name == "overloaded" then
			SetOverloaded(inst, false)
			inst.components.timer:StartTimer("shardload", CalcOverloadThreshold(inst))
```

**三个分支对应三个不同的计时器**——这正是 Timer 组件的优势：**一个监听器管多个计时器**。

#### Pattern 2：在 Prefab 定义时用 `OnTimerDoneFn` 字段

看 `scripts/prefabs/weed_defs.lua` 第 143-148 行：

```143:148:scripts/prefabs/weed_defs.lua
WEED_DEFS.weed_tillweed.OnTimerDoneFn = function(inst, data)
	if data.name == "make_debris" then
		local x, y, z = inst.Transform:GetWorldPosition()
		local tilling_dist = inst.weed_def.spread.ground_dist
		local debris = TheSim:FindEntities(x, y, z, tilling_dist, DEBRIS_OBJECTS_ONEOF_TAGS)
		if #debris < TUNING.WEED_TILLWEED_MAX_DEBRIS then
```

有些"**带 OnXxxFn 字段**"的 Prefab 定义方式是另一种抽象——把回调存在字段里供通用处理器调用。你可以把它看作 `OnTimerDone` 的变种。

#### 完整的 buff 示例

把前面学到的串起来，写一个完整的"速度 buff"——持续 60 秒：

```lua
-- 在 Prefab 的工厂函数末尾
inst:AddComponent("timer")

local function OnTimerDone(inst, data)
    if data.name == "speed_buff" then
        inst.components.locomotor:RemoveExternalSpeedMultiplier(inst, "speed_buff_src")
        inst:PushEvent("buff_expired", { type = "speed_buff" })
    end
end

inst:ListenForEvent("timerdone", OnTimerDone)

-- 外部代码调用（比如喝了加速药水）
local function ApplySpeedBuff(inst)
    inst.components.locomotor:SetExternalSpeedMultiplier(inst, "speed_buff_src", 1.5)

    if inst.components.timer:TimerExists("speed_buff") then
        inst.components.timer:StopTimer("speed_buff")
    end
    inst.components.timer:StartTimer("speed_buff", 60)
end
```

**这 12 行代码完整实现了一个"60 秒速度 buff、可叠加刷新、存档恢复"的机制**——如果不用 Timer 组件，你要自己处理存档、查询、暂停等所有细节，代码量至少 3 倍。

---

### 6.4.4 进阶：Pause / Resume / LongUpdate / SetTimeLeft

除了 Start / Stop / GetTimeLeft，Timer 还有几个"**进阶 API**"——它们在特殊场景下非常关键。

#### API：`PauseTimer / ResumeTimer`

看第 72-93 行：

```72:93:scripts/components/timer.lua
function Timer:PauseTimer(name)
    if not self:TimerExists(name) or self:IsPaused(name) then
        return
    end

    self:GetTimeLeft(name)

    self.timers[name].paused = true
    self.timers[name].timer:Cancel()
    self.timers[name].timer = nil
end

function Timer:ResumeTimer(name)
    if not self:IsPaused(name) then
        return
    end

    self.timers[name].paused = false
    self.timers[name].timer = self.inst:DoTaskInTime(self.timers[name].timeleft, OnTimerDone, self, name)
    self.timers[name].end_time = GetTime() + self.timers[name].timeleft
	return true
end
```

**Pause 的技巧**：

- `self:GetTimeLeft(name)` 一行——这是**把当前剩余时间"**锁定**"进 `timeleft` 字段**（回顾 `GetTimeLeft` 的实现，它会先更新 `timeleft = end_time - GetTime()`）
- 然后 `Cancel` 底层 task
- 设置 `paused = true`

**Resume 则是反过来**：根据当前的 `timeleft` 新建一个 task，重置 `end_time`。

**典型使用场景**——`scripts/prefabs/skilltree_wortox.lua` 第 39-44 行：

```39:44:scripts/prefabs/skilltree_wortox.lua
local function OnDeath(inst, data)
    if inst.components.timer:TimerExists("wortox_panflute_playing") then
        inst.components.timer:PauseTimer("wortox_panflute_playing")
    end
end
```

**Wortox 死亡时**暂停"**万花筒笛子**"的计时器——这样他复活后可以 `ResumeTimer` 继续倒计时，而不是从头开始或直接消失。

这就是 Timer 相比于原生 `DoTaskInTime` 的最大优势——**"**死亡不会丢失状态**"**。

#### API：`LongUpdate` —— 离散时间补偿

看第 144-148 行：

```144:148:scripts/components/timer.lua
function Timer:LongUpdate(dt)
    for k, v in pairs(self.timers) do
        self:SetTimeLeft(k, self:GetTimeLeft(k) - dt)
    end
end
```

**`LongUpdate(dt)`**——把所有计时器快进 `dt` 秒。

**什么时候被调用？** 主要是**离散时间补偿**场景：

- **玩家睡觉**：`sleep_buff.lua` 调用所有组件的 `LongUpdate(8 * 60)` 让时间"快进 8 分钟"
- **场景过渡**：从地面到洞穴、从主世界到远古洞穴——可能需要快进时间
- **世界重置**：`TheWorld:LongUpdate(3600)` 把世界快进 1 小时

**使用这个 API 的是引擎 / 世界系统**——你通常不直接调用 `LongUpdate`。但你要知道**如果你的 Timer 在睡觉快进时需要也跟着快进，Timer 组件已经自动处理了**。

#### API：`SetTimeLeft` —— 手动改剩余时间

看第 104-114 行：

```104:114:scripts/components/timer.lua
function Timer:SetTimeLeft(name, time)
    if not self:TimerExists(name) then
        return
    elseif self:IsPaused(name) then
        self.timers[name].timeleft = math.max(0, time)
    else
        self:PauseTimer(name)
        self.timers[name].timeleft = math.max(0, time)
        self:ResumeTimer(name)
    end
end
```

**两种情况**：

- **暂停中**：直接改 `timeleft`
- **运行中**：先 Pause（锁定当前时间）→ 改 `timeleft` → Resume（新时间重新开始倒计时）

**`math.max(0, time)`** 保护——防止你传负数把计时器搞崩。

**使用场景**：**玩家喝了"时间加速药剂"** → `SetTimeLeft("some_timer", current - 30)` 减 30 秒——立即触发剩余时间变化。

#### API：`TransferComponent` —— 跨实体转移

看第 150-157 行：

```150:157:scripts/components/timer.lua
function Timer:TransferComponent(newinst)
    local newcomponent = newinst.components.timer

    for k, v in pairs(self.timers) do
        newcomponent:StartTimer(k, self:GetTimeLeft(k), v.paused, v.initial_time)
    end

end
```

**`TransferComponent`** 是一个 Klei 约定俗成的组件接口——在"**实体形态切换**"时把旧实体的状态搬到新实体。比如 Woodie 变身 Beaver 时会用到，把 `speed_buff` 的剩余时间原封不动搬到新形态上。

---

### 6.4.5 进阶：Timer 的 OnSave / OnLoad —— 自动存档恢复

这是 Timer 相比于原生 task **最大的价值**——**自动存档和恢复**。

看第 122-142 行：

```122:142:scripts/components/timer.lua
function Timer:OnSave()
    local data = {}
    for k, v in pairs(self.timers) do
        data[k] =
        {
            timeleft = self:GetTimeLeft(k),
            paused = v.paused,
            initial_time = v.initial_time,
        }
    end
    return next(data) ~= nil and { timers = data } or nil
end

function Timer:OnLoad(data)
    if data.timers ~= nil then
        for k, v in pairs(data.timers) do
            self:StopTimer(k)
            self:StartTimer(k, v.timeleft, v.paused, v.initial_time)
        end
    end
end
```

#### OnSave 的三个技巧

**技巧 1：`GetTimeLeft` 保证时间准确**

```lua
timeleft = self:GetTimeLeft(k),
```

不是存 `v.timeleft`（那可能是很久以前的值），而是**用 `GetTimeLeft(k)` 重新计算**——保证存档写入的时候是最新的剩余秒数。

**技巧 2：`next(data) ~= nil and { timers = data } or nil`**

**只有 data 非空才返回**——没有活跃计时器就返回 nil，不写进存档。回顾 6.1.4 节的 OnSave 铁律——**"只存必要的"**。

**技巧 3：只存状态、不存 task 引用**

`v.timer`（scheduler task）**不存**——存了也没用（task 是运行时对象）。读档时由 `OnLoad` 里的 `StartTimer` 重新创建 task。

#### OnLoad 的关键写法

```lua
self:StopTimer(k)   -- 先清理可能的残留
self:StartTimer(k, v.timeleft, v.paused, v.initial_time)
```

**`StopTimer(k)` 的防御**：读档时实体可能刚刚经过一遍工厂函数（也可能已经启动了默认计时器），先 StopTimer 确保干净状态再 StartTimer——**幂等安全**。

#### 完整的存档生命周期

**场景**：玩家拿着一个"3 分钟后自毁"的道具——存档-退出-重进看一下：

```
玩家拿起道具
  ↓
StartTimer("self_destruct", 180)
  ↓
玩家玩了 90 秒（剩余 90 秒）
  ↓
玩家保存存档
  ├── 实体的 GetPersistData 遍历所有组件
  ├── timer:OnSave() 被调用
  └── 返回 { timers = { self_destruct = { timeleft = 90, paused = false, initial_time = 180 } } }
     ↓
  写入存档文件

玩家退出游戏、重启游戏、加载存档
  ↓
实体被重建（SpawnPrefabFromSim）
  ├── timer 组件被 AddComponent（工厂函数里）
  │     └── self.timers = {}（空）
  │
  └── SetPersistData 被调用
        └── timer:OnLoad(data) 被调用
              ├── StopTimer("self_destruct")（防御清理，没有也没事）
              └── StartTimer("self_destruct", 90, false, 180)
                    ├── timeleft = 90 ← 从存档恢复的
                    ├── initial_time = 180 ← 从存档恢复的
                    └── 新建 scheduler task
  ↓
玩家继续游戏，90 秒后自毁
```

**整条链路**——你一行存档/读档代码都没写，全靠 Timer 组件的 OnSave/OnLoad 帮你搞定。

> **开发逻辑**：**Timer 组件的 OnSave/OnLoad 是"**能力即免费**"的典型体现**——只要你用 Timer 而不是原生 task，就**自动获得**存档恢复能力，哪怕你根本不懂它怎么实现的。**这就是组件系统的美**——用现成的组件而不是自己造轮子。

---

### 6.4.6 进阶：Timer 与原生任务 API 的对比

上面一直说 Timer 比 `DoTaskInTime` 好——但更准确的说法是**两者适用场景不同**。这节做一个对比。

#### 原生任务：`DoTaskInTime` 和 `DoPeriodicTask`

看 `entityscript.lua` 第 1512-1530 行：

```1512:1530:scripts/entityscript.lua
function EntityScript:DoPeriodicTask(time, fn, initialdelay, ...)
    --print ("DO PERIODIC", time, self)
    local periodic = scheduler:ExecutePeriodic(time, fn, nil, initialdelay, self.GUID, self, ...)

    self.pendingtasks = self.pendingtasks or {}
    self.pendingtasks[periodic] = true
    periodic.onfinish = task_finish --function() if self.pendingtasks then self.pendingtasks[per] = nil end end
    return periodic
end

function EntityScript:DoTaskInTime(time, fn, ...)
    --print ("DO TASK IN TIME", time, self)
    local periodic = scheduler:ExecuteInTime(time, fn, self.GUID, self, ...)

    self.pendingtasks = self.pendingtasks or {}
    self.pendingtasks[periodic] = true
    periodic.onfinish = task_finish -- function() if self and self.pendingtasks then self.pendingtasks[per] = nil end end
    return periodic
end
```

**两个 API**：

- **`DoTaskInTime(time, fn, ...)`**：`time` 秒后**执行一次** `fn`
- **`DoPeriodicTask(time, fn, initialdelay, ...)`**：每隔 `time` 秒**周期执行** `fn`

**共同点**：

- 都挂在 `self.pendingtasks` 上——实体销毁时自动取消（避免"僵尸任务"）
- 都返回一个 `periodic` 对象，可以 `:Cancel()` 手动取消

#### 特性对比表

| 特性 | Timer 组件 | `DoTaskInTime` | `DoPeriodicTask` |
|------|-----------|----------------|-------------------|
| 一次性 or 周期性 | 一次性 | 一次性 | 周期性 |
| 有名字可查询 | ✅ 有（字符串 name） | ❌ 返回 task 对象 | ❌ 返回 task 对象 |
| 查询剩余时间 | ✅ `GetTimeLeft(name)` | ❌ 没有 API | ❌ 没有 API |
| 暂停/恢复 | ✅ `Pause/Resume` | ❌ 只能取消 | ❌ 只能取消 |
| 自动存档恢复 | ✅ Timer:OnSave/OnLoad | ❌ 需要自己做 | ❌ 需要自己做 |
| 自动 LongUpdate 支持 | ✅ 内置 | ❌ 无 | ❌ 无 |
| 触发回调 | 间接（通过 `timerdone` 事件） | 直接（闭包） | 直接（闭包） |
| 实体销毁时自动清理 | ✅ 通过 OnRemoveFromEntity | ✅ 通过 pendingtasks | ✅ 通过 pendingtasks |
| 开销 | 中等（Class + 事件） | 低（只是 scheduler） | 低（只是 scheduler） |

#### 什么时候用什么？

**用 Timer 组件**：

- **玩家可见的倒计时 / buff 持续时间**——永远用 Timer
- **技能冷却时间**——永远用 Timer
- **物品的"xx 秒后生效/失效"**——永远用 Timer

**用 `DoTaskInTime`**：

- **纯内部机制，不需要存档**（如"5 帧后播放下一个动画"）
- **仅一次性、不需要暂停**（如"拾起物品时的短暂高亮特效"）
- **回调逻辑非常简单、只有一处用**——写个 Timer 反而啰嗦

**用 `DoPeriodicTask`**：

- **周期性检查**（如"每秒检查一次是否在水里"）
- **心跳更新**（如 StateGraph 的 tick）
- **AI 周期性 retarget**

#### 对比示例

**场景**：玩家每受伤一次就显示一个红色受伤特效，1 秒后消失。

**用 Timer（过度工程）**：

```lua
inst:ListenForEvent("attacked", function(inst)
    if inst.components.timer:TimerExists("damage_flash") then
        inst.components.timer:StopTimer("damage_flash")
    end
    inst.components.timer:StartTimer("damage_flash", 1)
    inst:AddTag("damage_flashing")
end)

inst:ListenForEvent("timerdone", function(inst, data)
    if data.name == "damage_flash" then
        inst:RemoveTag("damage_flashing")
    end
end)
```

**用 `DoTaskInTime`（合适）**：

```lua
inst:ListenForEvent("attacked", function(inst)
    if inst._damage_flash_task then
        inst._damage_flash_task:Cancel()
    end
    inst:AddTag("damage_flashing")
    inst._damage_flash_task = inst:DoTaskInTime(1, function()
        inst:RemoveTag("damage_flashing")
        inst._damage_flash_task = nil
    end)
end)
```

**受伤特效不需要"**玩家查询"、存档恢复、暂停"**——纯瞬态行为**，用 DoTaskInTime 更简洁。

> **开发逻辑**：**Timer 不是"万能方案"**——它有固定成本（组件本身、事件系统）。对于瞬态、不可查询、不存档的任务，**原生 task 更轻量更合适**。写 Mod 时先问自己"**玩家能不能感知这个倒计时？需不需要它存档？**"——能，用 Timer；不能，用 DoTaskInTime。

---

### 6.4.7 老手进阶：Timer 的实战设计模式

这一节我们看真实 Prefab 中 Timer 的**典型使用模式**——这些模式是饥荒联机版的"**标准答案**"，值得直接套用。

#### 模式 1：多计时器统一分发（WX-78 模式）

`scripts/prefabs/wx78.lua` 在玩家身上维护了**至少 5 个**不同的计时器：

```lua
-- wx78.lua 片段
inst.components.timer:StartTimer(CHARGEREGEN_TIMERNAME, inst:GetChargeRegenTime())  -- 充电再生
inst.components.timer:StartTimer(MOISTURETRACK_TIMERNAME, TUNING.WX78_MOISTUREUPDATERATE*FRAMES)  -- 湿度追踪
inst.components.timer:StartTimer(HUNGERDRAIN_TIMERNAME, TUNING.WX78_HUNGRYCHARGEDRAIN_TICKTIME)  -- 饥饿时的能量流失
```

对应的 `OnTimerFinished` 函数里**统一分发**：

```lua
local function OnTimerFinished(inst, data)
    if data.name == CHARGEREGEN_TIMERNAME then
        DoChargeRegen(inst)
    elseif data.name == MOISTURETRACK_TIMERNAME then
        DoMoistureTrack(inst)
    elseif data.name == HUNGERDRAIN_TIMERNAME then
        DoHungerDrain(inst)
    -- ...
    end
end

inst:ListenForEvent("timerdone", OnTimerFinished)
```

**模式特征**：

- **使用 `local` 常量**（`CHARGEREGEN_TIMERNAME = "chargeregen_timer"`）——避免字符串手写拼错
- **一个 `OnTimerFinished` 函数 + 多个 elseif 分支**——集中管理
- **触发后很可能再次 StartTimer**——形成"**循环心跳**"（比如 chargeregen 每次触发后自动续命）

#### 模式 2：互斥计时器（Winona 电池模式）

`scripts/prefabs/winona_battery_high.lua` 第 763-779 行：

```lua
local function OnTimerDone(inst, data)
    if data then
        if data.name == "shardloaddelay" then
            inst.components.timer:ResumeTimer("shardload")
            StartUpdatingShardLoad(inst)
        elseif data.name == "shardload" then
            StopUpdatingShardLoad(inst)
            OnUpdateShardLoad(inst)
        elseif data.name == "overloaded" then
            SetOverloaded(inst, false)
            inst.components.timer:StartTimer("shardload", CalcOverloadThreshold(inst))
        end
    end
end
```

**模式特征**：

- **计时器之间相互联动**：`shardloaddelay` 结束 → Resume `shardload`
- **状态机式**：一个状态结束必然触发下一个状态——像多米诺骨牌
- **充分利用 `PauseTimer` + `ResumeTimer`**：让复杂的电池状态机不需要额外字段

这种模式适合"**多阶段倒计时机制**"——如电池充电→过载→冷却→重新充电的循环。

#### 模式 3：刷新式 buff（Tillweed Salve 模式）

`scripts/prefabs/tillweedsalve.lua` 第 76-84 行：

```76:84:scripts/prefabs/tillweedsalve.lua
local function OnTimerDone(inst, data)
    if data.name == "regenover" then
        inst.components.debuff:Stop()
    end
end

local function OnExtended(inst, target)
    inst.components.timer:StopTimer("regenover")
    inst.components.timer:StartTimer("regenover", BUFF_DURATION)
```

**模式特征**：

- **先 `StopTimer` 再 `StartTimer`**——经典"**刷新 buff 持续时间**"
- **命名清晰**：`"regenover"` = "再生结束"，语义直接
- **触发 `timerdone` 时调 `debuff:Stop()`**——让 debuff 组件自己处理剩余清理

这是**任何"可叠加刷新的 buff"的标准写法**——药水类、食物 buff、魔法 buff 都用这套。

#### 模式 4：短期冷却（WX 扫描仪模式）

`scripts/prefabs/wx78_scanner.lua` 第 242-244 行：

```lua
owner.components.talker:Say(GetString(owner,"ANNOUNCE_WX_SCANNER_NEW_FOUND"))
owner.components.timer:StartTimer("ANNOUNCE_WX_SCANNER_NEW_FOUND", 15)
```

配合**触发前检查**：

```lua
if not owner.components.timer:TimerExists("ANNOUNCE_WX_SCANNER_NEW_FOUND") then
    -- 说话 + 开冷却
end
```

**模式特征**：

- **冷却时间用 Timer**——15 秒内不重复触发
- **计时器名字就是事件名**——命名达到最高自洽（一看就知道这是"这个事件的冷却"）
- **只开启、不需要 timerdone 回调**——只是个"空转锁"

这对应"**防刷屏**"、"**技能 CD**"等场景——你要的不是计时器到期后做什么，而是"**在这段时间内不让某事件触发**"。

#### 模式 5：子实体跟随父实体计时（lunarthrall）

`scripts/prefabs/lunarthrall_plant_gestalt.lua` 第 8-10 行：

```8:10:scripts/prefabs/lunarthrall_plant_gestalt.lua
local function Spawn(inst)
    inst.components.timer:StartTimer("justspawned",15)
    inst.Transform:SetRotation(math.random()*360)
```

**模式特征**：

- **初始化时立刻启动计时器**——表示"刚生成状态持续 15 秒"
- **配合 `TimerExists("justspawned")` 判断**——"我还在刚生成状态吗？"
- **不一定需要 timerdone 回调**——计时器只作为"**状态标记**"

这是**"状态持续时间"的最轻量实现**——甚至不用布尔字段，直接靠 Timer 的 `TimerExists` 来判断。

#### 六种核心命名习惯

从源码里总结出的命名风格：

| 模式 | 命名约定 | 示例 |
|------|----------|------|
| buff/debuff 持续时间 | `<buff名>` or `<buff名>_duration` | `"speed_buff"`, `"regenover"` |
| 技能冷却 | `<skill名>_cd` or `<skill名>` | `"skill_cooldown"`, `"panflute_playing"` |
| 状态持续 | `<状态名>` | `"justspawned"`, `"overloaded"` |
| 事件防抖 | `<事件名>` | `"ANNOUNCE_WX_SCANNER_NEW_FOUND"` |
| 心跳节奏 | `<系统名>_ticktime` | `"heatsteam_ticktime"`, `"moisturetrack_timer"` |
| 延迟触发 | `<动作名>_delay` | `"shardloaddelay"` |

**你自己写 Mod 时**——建议**定义 `local` 常量**，避免手写字符串出错：

```lua
local SPEED_BUFF_TIMERNAME = "mymod_speed_buff"
local COOLDOWN_TIMERNAME = "mymod_skill_cooldown"

inst.components.timer:StartTimer(SPEED_BUFF_TIMERNAME, 60)
```

---

### 6.4.8 老手进阶：Timer 使用的五个陷阱

#### 陷阱 1：同名计时器重复启动

```lua
-- ❌ 错：StartTimer 第二次静默失败，警告到控制台但代码继续运行
inst.components.timer:StartTimer("buff", 60)
inst.components.timer:StartTimer("buff", 30)   -- 警告：计时器已存在！
```

**症状**：控制台打印警告，**第二次 StartTimer 什么都不做**——计时器还是 60 秒，而不是你期望的"刷新成 30 秒"。

**解决**：**先 Stop 再 Start**：

```lua
-- ✅ 对
if inst.components.timer:TimerExists("buff") then
    inst.components.timer:StopTimer("buff")
end
inst.components.timer:StartTimer("buff", 30)
```

---

#### 陷阱 2：在 timerdone 回调里再次 StartTimer 导致栈溢出

```lua
-- ❌ 错：看起来没问题
inst:ListenForEvent("timerdone", function(inst, data)
    if data.name == "tick" then
        inst.components.timer:StartTimer("tick", 0)   -- 0 秒立即触发 → 无限递归
    end
end)
```

**症状**：爆栈 / 无限循环 / 游戏卡死。

**原因**：**`StartTimer(name, 0)` 会立即触发 `OnTimerDone` → 再 StartTimer → 再立即触发 → 死循环**。

**解决**：**心跳类任务用 DoPeriodicTask，不用 Timer**。Timer 不适合高频重复任务。

---

#### 陷阱 3：名字冲突（多个 Mod 用同一个名字）

```lua
-- ModA 里
inst.components.timer:StartTimer("cooldown", 30)

-- ModB 里
inst.components.timer:StartTimer("cooldown", 60)   -- 静默失败
```

**症状**：两个 Mod 互相干扰，有时一个工作有时另一个工作。

**解决**：**计时器名字永远加 Mod 前缀**：

```lua
-- ModA
inst.components.timer:StartTimer("mymod_cooldown", 30)
```

这和 6.3.8 陷阱 4（net 变量名冲突）是同一个道理——**命名空间隔离是 Mod 兼容性的基础**。

---

#### 陷阱 4：`OnTimerDone` 没判断 `data.name`

```lua
-- ❌ 错：任何计时器结束都会执行这段
inst:ListenForEvent("timerdone", function(inst, data)
    inst.components.debuff:Stop()    -- 任何 timer 结束都停 debuff
end)

-- ✅ 对：严格判断
inst:ListenForEvent("timerdone", function(inst, data)
    if data.name == "debuff_duration" then
        inst.components.debuff:Stop()
    end
end)
```

**原因**：Timer 组件上**可能有多个不同计时器**，所有 `timerdone` 都通过同一个事件通知——你必须**严格判断 `data.name`**。

---

#### 陷阱 5：存档恢复后 `initial_time` 丢失

假设你要画 UI 进度条：`percent = TimeElapsed / initial_time`。一个玩家进游戏 → 拿起道具 → 存档 → 重启 → 继续玩——UI 进度条**可能显示错误**。

**原因**：`OnSave` 里存了 `initial_time`，`OnLoad` 里用 `StartTimer(k, v.timeleft, v.paused, v.initial_time)` 恢复——**第四个参数 `initialtime_override`**。

**看 `StartTimer` 的签名**（第 36 行）：

```lua
function Timer:StartTimer(name, time, paused, initialtime_override)
```

**第二个参数 `time` 是"剩余时间"**，第四个参数 `initial_time_override` 是"总时长"——两个是**独立的**。

**如果你自己调 `StartTimer("buff", 60)`**（没传第四参数），`initial_time` 会自动设为 60（和 time 相同）。但读档时 `OnLoad` 会正确传入原来的 `initial_time`。

**这个陷阱的细节**：如果你做 UI 进度条，要**永远用 `GetTimeElapsed` 和 `initial_time`**，而不是自己算——这样存档恢复也会正确。

```lua
-- ❌ 错：硬编码 60 作为总时长
local percent = inst.components.timer:GetTimeElapsed("buff") / 60

-- ✅ 对：从 Timer 组件拿（或自己保存 initial_time）
local timedata = inst.components.timer.timers["buff"]
local percent = (timedata.initial_time - inst.components.timer:GetTimeLeft("buff")) / timedata.initial_time
```

或者**更优雅地暴露一个 helper**：

```lua
function Timer:GetPercent(name)
    if not self:TimerExists(name) then return nil end
    local t = self.timers[name]
    return (t.initial_time - self:GetTimeLeft(name)) / t.initial_time
end
```

你可以通过 `AddComponentPostInit("timer", function(cmp) cmp.GetPercent = ... end)` 给官方 Timer 组件**加一个方法**——这是 6.1.8 讲过的 PostInit 技巧。

---

### 6.4.9 小结

- **Timer 组件是"命名化的、可查询的、可暂停的、可自动存档的计时器"**——比原生 `DoTaskInTime` 多出四个能力，但成本略高。
- **核心 API 五件套**：`StartTimer(name, time, paused?, initial_time_override?)` / `StopTimer(name)` / `TimerExists(name)` / `GetTimeLeft(name)` / `GetTimeElapsed(name)`——99% 的场景用这五个就够。
- **通过 `timerdone` 事件触发回调**：`ListenForEvent("timerdone", fn)` + `if data.name == "xxx"` 分发——一个监听器可以处理组件上任意多个计时器。
- **进阶 API**：`PauseTimer / ResumeTimer`（玩家死亡时冻结 buff 进度）、`LongUpdate(dt)`（离散时间补偿）、`SetTimeLeft`（动态改时间）、`TransferComponent`（实体形态切换）。
- **自动存档恢复**：`Timer:OnSave` 把 `{timeleft, paused, initial_time}` 写进存档；`Timer:OnLoad` 通过 StartTimer 恢复所有计时器——**用 Timer 就等于自动获得存档能力**。
- **Timer vs 原生 task**：玩家可感知 / 需存档 / 需要暂停 → Timer；纯瞬态 / 高频心跳 / 不存档 → `DoTaskInTime` / `DoPeriodicTask`。两者**适用场景不同，不是替代关系**。
- **五个实战设计模式**：多计时器统一分发（WX-78）、互斥计时器（Winona 电池）、刷新式 buff（Tillweed Salve）、短期冷却（WX 扫描仪）、状态持续标记（lunarthrall）——这些是源码里的"**标准答案**"，值得直接套用。
- **命名习惯**：buff 用 `<buff名>_duration`、冷却用 `<skill>_cd`、状态用 `<状态名>`、防抖用 `<事件名>`、心跳用 `<系统>_ticktime`——**加 Mod 前缀防冲突**。
- **五个陷阱**：同名重复启动（静默失败）、在 `timerdone` 里 StartTimer(0) 导致递归、Mod 间名字冲突、`OnTimerDone` 没判 `data.name`、UI 进度条硬编码总时长——每一个都有对应的防御写法。

> **下一节预告**：6.5 节我们讲 **SourceModifierList**——"多来源叠加"的通用工具。当玩家同时有"加速药水 +20%"、"靴子 +10%"、"马鞍 +30%"时，最终速度怎么算？如果有一个源消失了怎么清理？这个看似简单的问题有一个**非常优雅的官方解法**。

## 6.5 SourceModifierList——多来源叠加的通用工具

### 本节导读

这一节我们讲一个饥荒联机版里非常**优雅却被低估**的工具——`SourceModifierList`。它只有 **182 行**，但几乎每一个涉及"**数值受多个来源影响**"的组件都用到它：饥饿恢复速度、理智流失速度、战斗伤害倍率、电池耗能速率、buff/debuff 叠加⋯⋯

**它解决的核心问题是**：当玩家同时受到**多个独立来源**的 buff/debuff 影响时，最终值怎么算？如果有一个来源消失了怎么自动清理？Mod 之间怎么不打架？这些看似琐碎的问题如果不用抽象工具，**每个组件都要重写一遍**——而 `SourceModifierList` 就是 Klei 提炼出的**"**通用多源修正器**"**。

> **新手**从 6.5.1-6.5.3 入手，只要理解"**多源叠加问题**"、记住 `SetModifier / RemoveModifier / Get` 三件套 + 三种叠加函数（multiply/additive/boolean），就能在 Mod 里做 buff 系统；**进阶读者**从 6.5.4-6.5.6 深入 source 参数的双重语义（实体或字符串）、key 的分层机制、以及自动清理（`onremove` 事件）的妙处；**老手**跳到 6.5.7-6.5.8 看真实组件（`fueled` / `sanity` / `inventory` / `wx78_shield`）的实战应用，以及五个最容易栽的坑。

**前置**：本节假设你已经读过 6.1 节（组件生命周期）和 6.4 节（Timer 组件）——`SourceModifierList` 和 Timer 是"一对老搭档"，配合最多。

---

### 6.5.1 快速入门：多来源叠加问题是什么？

先看一个具体场景。玩家 Wilson 在某个时刻**同时受到以下影响**：

- **鞋子（步行鞋）** → 移动速度 ×1.2
- **喝了加速药水** → 移动速度 ×1.5（持续 60 秒）
- **骑了一只牛** → 移动速度 ×2.0
- **踩到蜘蛛网** → 移动速度 ×0.5
- **Wilson 有大胡子 buff** → 移动速度 ×1.1

**最终玩家的移动速度应该是多少？**

按 Klei 的设计逻辑，答案是：**`基础速度 × 1.2 × 1.5 × 2.0 × 0.5 × 1.1`**——每个来源按乘法叠加。

#### 这个问题的"**坏写法**"

没有工具时，程序员常见的错误写法：

```lua
-- ❌ 错：一堆 if 硬编码
function CalculateSpeed(inst)
    local speed = BASE_SPEED
    if inst:HasTag("boots") then speed = speed * 1.2 end
    if inst.components.sanity:IsInsane() then speed = speed * 0.8 end
    if inst._has_speed_potion then speed = speed * 1.5 end
    if inst.components.rider:IsRiding() then speed = speed * 2.0 end
    -- ... 几十行 if ...
    return speed
end
```

**这种写法的问题**：

1. **难以扩展**：想加一个新 buff，要改这个函数
2. **难以清理**：药水 buff 到期了，要手动把 `_has_speed_potion` 设为 false
3. **难以查询**：UI 想显示"当前被几个 buff 影响"——拿不到列表
4. **Mod 冲突**：Mod A 和 Mod B 都想加速，谁先谁后？
5. **状态溢出**：临时实体（加速药水）消失后忘了清理，`_has_speed_potion` 永远是 true

#### Klei 的解法：`SourceModifierList`

**核心思路**：每个组件维护一个 `SourceModifierList` 实例——**所有外部 buff/debuff 只需要调用 `SetModifier / RemoveModifier`**，内部自动：

- **叠加计算**（按指定函数：乘法 / 加法 / 布尔或）
- **自动清理**（来源实体被销毁时，自动移除它的 modifier）
- **精确查询**（"当前有哪些来源？"、"最终值是多少？"）
- **命名隔离**（不同来源用不同的 key，互不干扰）

看最简单的用法——`scripts/components/fueled.lua` 第 87-90 行：

```87:90:scripts/components/fueled.lua
    self.maxfuel = 0
    self.currentfuel = 0
    self.rate = 1
	self.rate_modifiers = SourceModifierList(self.inst)
```

**一行 `SourceModifierList(self.inst)` 就搞定了一个完整的"多源修正系统"**——之后任何外部代码调 `self.rate_modifiers:SetModifier(source, multiplier, key)` 就能影响 `rate`，`SourceModifierList` 自己管理所有清理工作。

#### 另一个实战使用

看 `scripts/prefabs/winona_battery_high.lua` 第 42-46 行：

```42:46:scripts/prefabs/winona_battery_high.lua
	if mult ~= 1 then
		inst.components.fueled.rate_modifiers:SetModifier(inst, mult, "efficiency")
	else
		inst.components.fueled.rate_modifiers:RemoveModifier(inst, "efficiency")
	end
```

**三行代码实现了"如果倍率不是 1 就设置，否则移除"**——省了几十行 if 判断。这就是 `SourceModifierList` 的简洁之美。

> **新手记忆**：**"多个来源同时影响一个数值"——用 `SourceModifierList`**。不要自己写 if 分支，也不要用布尔字段记录状态。Klei 把这个问题抽象得很干净，直接用就行。

---

### 6.5.2 快速入门：SourceModifierList 的核心 API

打开 `scripts/util/sourcemodifierlist.lua`——只有 **182 行**，超级简洁。我们一步步拆。

#### 构造函数

看第 8-23 行：

```8:23:scripts/util/sourcemodifierlist.lua
SourceModifierList = Class(function(self, inst, base_value, fn, dirtycb)
    self.inst = inst

    -- Private members
    self._modifiers = {}
    if base_value ~= nil then
        self._modifier = base_value
        self._base = base_value
    else
        self._modifier = 1
        self._base = 1
    end

    self._fn = fn or SourceModifierList.multiply
	self._dirtycb = dirtycb
end)
```

**四个参数**（前两个必填，后两个可选）：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `inst` | 必填 | 归属于哪个实体（用于事件监听） |
| `base_value` | `1` | 基础值（没有任何 modifier 时的值） |
| `fn` | `multiply` | 叠加函数（怎么把多个 modifier 组合） |
| `dirtycb` | `nil` | 值变化时的回调（`fn(inst, new_value)`） |

**三种常用的创建方式**：

```lua
-- ① 最简单：乘法叠加，基础值 1（对应"倍率"场景）
self.rate_modifiers = SourceModifierList(self.inst)

-- ② 加法叠加，基础值 0（对应"加值"场景）
self.externalmodifiers = SourceModifierList(self.inst, 0, SourceModifierList.additive)

-- ③ 布尔或，基础值 false（对应"只要有一个来源就 true"场景）
self.neg_aura_immune_sources = SourceModifierList(inst, false, SourceModifierList.boolean)
```

这三种你在 `scripts/components/sanity.lua` 第 78-99 行能看到全部用到——理智组件本身就用了 **7 个** `SourceModifierList` 实例处理各种修正。

#### API 1：`Get` —— 获取当前值

```38:40:scripts/util/sourcemodifierlist.lua
function SourceModifierList:Get()
	return self._modifier
end
```

**最常用的 API**——返回所有来源叠加后的最终值。

**使用场景**：

```lua
-- 实际计算某个受 modifier 影响的值时
local final_rate = self.rate * self.rate_modifiers:Get()
```

---

#### API 2：`SetModifier` —— 设置/更新 modifier

```64:99:scripts/util/sourcemodifierlist.lua
function SourceModifierList:SetModifier(source, m, key)
	if source == nil then
		return
	end

    if key == nil then
        key = "key"
    end

    if m == nil or m == self._base then
        self:RemoveModifier(source, key)
        return
    end

    local src_params = self._modifiers[source]
    if src_params == nil then
        self._modifiers[source] = {
            modifiers = { [key] = m },
        }

        -- If the source is an object, then add a onremove event listener to cleanup if source is removed from the game
		if EntityScript.is_instance(source) then
            self._modifiers[source].onremove = function(source)
                self._modifiers[source] = nil
				self:RecalculateModifier()
            end

            self.inst:ListenForEvent("onremove", self._modifiers[source].onremove, source)
        end

		self:RecalculateModifier()
    elseif src_params.modifiers[key] ~= m then
        src_params.modifiers[key] = m
		self:RecalculateModifier()
    end
end
```

**签名**：`SetModifier(source, m, key)`

| 参数 | 类型 | 含义 |
|------|------|------|
| `source` | Entity 或 string | 这个 modifier 的"来源"——可以是一个实体，也可以是一个字符串名 |
| `m` | number/bool | 这个 modifier 的值 |
| `key` | string（可选）| 同一来源可以有多个 modifier，用 key 区分 |

**内部做了 5 件事**：

1. **防御**：`source == nil` 直接返回
2. **默认 key**：没传 key 就用 `"key"` 作为默认值
3. **如果 m 等于 base_value 就调 `RemoveModifier`**——相当于 "setter 返回默认值就自动移除"（这是一个贴心的优化）
4. **如果 source 是实体，自动注册 onremove 监听器**——实体销毁时 modifier 自动移除
5. **调 `RecalculateModifier`**——重新计算 `_modifier`

**实战使用**（`winona_battery_low.lua` 第 283-284 行）：

```lua
inst.components.fueled.rate_modifiers:SetModifier(inst, CalcFuelRateRescale(inst), "rescale")
```

**同一个组件可以有多个 key**——`"rescale"` / `"efficiency"` / `"shard"` 各代表一个独立的 modifier。

---

#### API 3：`RemoveModifier` —— 移除 modifier

```103:122:scripts/util/sourcemodifierlist.lua
function SourceModifierList:RemoveModifier(source, key)
    local src_params = self._modifiers[source]
    if src_params == nil then
        return
    elseif key ~= nil then
        src_params.modifiers[key] = nil
        if next(src_params.modifiers) ~= nil then
            --this source still has other keys
			self:RecalculateModifier()
            return
        end
    end

    --remove the entire source
    if src_params.onremove ~= nil then
        self.inst:RemoveEventCallback("onremove", src_params.onremove, source)
    end
    self._modifiers[source] = nil
	self:RecalculateModifier()
end
```

**两种粒度**：

- **`RemoveModifier(source)`**（不传 key）——移除这个 source 的**所有** modifier
- **`RemoveModifier(source, key)`**——只移除这个 source 下的某一个 key 的 modifier，其他 key 保留

**智能清理**：如果 source 下所有 key 都被移除了（`next(src_params.modifiers) == nil`），就会**解除 onremove 事件监听**、并移除整个 source——避免内存泄漏。

---

#### API 4：`IsEmpty / HasAnyModifiers / HasModifier` —— 存在性查询

```42:44:scripts/util/sourcemodifierlist.lua
function SourceModifierList:IsEmpty()
	return next(self._modifiers) == nil
end
```

```177:179:scripts/util/sourcemodifierlist.lua
function SourceModifierList:HasAnyModifiers()
	return next(self._modifiers) ~= nil
end
```

```169:175:scripts/util/sourcemodifierlist.lua
function SourceModifierList:HasModifier(source, key)
    local src_params = self._modifiers[source]
    if src_params == nil then
        return false
    end
    return src_params.modifiers[key] ~= nil
end
```

**使用场景**：

```lua
-- 判断有没有任何 speed buff
if self.externalmodifiers:HasAnyModifiers() then
    -- 显示"buff 中"的 UI 图标
end

-- 判断特定来源是否有 modifier
if self.externalmodifiers:HasModifier(inst, "potion") then
    -- 这个实体已经喝过药水了
end
```

---

#### API 5：`Reset` —— 一次性清空

```124:135:scripts/util/sourcemodifierlist.lua
function SourceModifierList:Reset()
	for source, src_params in pairs(self._modifiers) do
		if src_params.onremove then
			self.inst:RemoveEventCallback("onremove", src_params.onremove, source)
		end
		self._modifiers[source] = nil
	end
	self._modifier = self._base
	if self._dirtycb then
		self._dirtycb(self.inst, self._modifier)
	end
end
```

**清空所有 modifier + 解除所有事件监听器 + 重置为 base_value + 触发 dirtycb**。

**使用场景**：实体死亡时清空所有 buff、角色切换形态时重置属性等。

---

### 6.5.3 快速入门：三种叠加函数

`SourceModifierList` 提供了三种内置叠加函数（定义在第 25-35 行）：

```25:35:scripts/util/sourcemodifierlist.lua
SourceModifierList.multiply = function(a, b)
	return a * b
end

SourceModifierList.additive = function(a, b)
	return a + b
end

SourceModifierList.boolean = function(a, b)
    return a or b
end
```

#### `multiply`（默认）

**典型场景**：倍率类数值——**速度倍率**、**伤害倍率**、**能量消耗倍率**等。

**公式**：`final = base × mod1 × mod2 × ... × modN`

**举例**：Winona 电池的能量消耗倍率（`fueled.rate_modifiers`）：

- **基础**：1.0（正常消耗）
- **效率模块加成**：0.8（消耗更慢）
- **过载惩罚**：1.5（消耗更快）
- **最终**：1.0 × 0.8 × 1.5 = 1.2

**创建方式**：`SourceModifierList(inst)` —— **不传第三个参数，默认 multiply**。

#### `additive`

**典型场景**：加值类数值——**外部理智回复速度**、**攻击伤害加成**等。

**公式**：`final = base + mod1 + mod2 + ... + modN`

**举例**：Wilson 外部理智恢复速度（`sanity.externalmodifiers`）：

- **基础**：0（没有任何外部恢复）
- **帅气发型 buff**：+1（每秒 +1 理智）
- **读了某本书**：+0.5（每秒 +0.5 理智）
- **最终**：0 + 1 + 0.5 = 1.5

**创建方式**：`SourceModifierList(inst, 0, SourceModifierList.additive)`——**基础值 0**（加法中的中性元素）。

看 `scripts/components/sanity.lua` 第 87 行：

```87:87:scripts/components/sanity.lua
	self.externalmodifiers = SourceModifierList(self.inst, 0, SourceModifierList.additive)
```

#### `boolean`

**典型场景**：**"只要有一个来源说 true 就 true"**——免疫某种效果、启用某个系统等。

**公式**：`final = mod1 or mod2 or ... or modN`

**举例**：Wendy 对鬼魂的理智光环免疫（`sanity.player_ghost_immune_sources`）：

- **基础**：false（默认不免疫）
- **Wendy 技能树解锁免疫**：true（她免疫）
- **其他玩家**：false（保持）
- **Wendy 最终**：true（免疫）
- **其他玩家最终**：false（受影响）

**创建方式**：`SourceModifierList(inst, false, SourceModifierList.boolean)`——**基础值 false**（or 的中性元素）。

看 `scripts/components/sanity.lua` 第 79 行：

```79:79:scripts/components/sanity.lua
	self._lunacy_sources = SourceModifierList(inst, false, SourceModifierList.boolean)
```

#### 自定义叠加函数

你也可以传入**任意自定义函数**。比如想要"**取最大值**"：

```lua
local function MaxOf(a, b)
    return math.max(a, b)
end

self.maxtaken_modifiers = SourceModifierList(inst, 0, MaxOf)
```

**典型场景**："**只取最严重的 debuff，不叠加**"——比如多种减速效果中取最强的。

#### 三者对照表

| 叠加类型 | 基础值 | 中性元素 | 典型场景 |
|---------|--------|----------|---------|
| multiply | 1 | 1（乘 1 不变）| 速度/伤害倍率 |
| additive | 0 | 0（加 0 不变）| 加值类回复/伤害 |
| boolean | false | false（or false 不变）| 免疫/启用标记 |
| max | 0 or -∞ | -∞ | 取最强效果 |
| min | ∞ | ∞ | 取最弱效果 |

> **新手记忆**：**先想"**基础状态是什么**"——倍率 1.0、加值 0、免疫 false——再用对应的叠加函数**。选错叠加函数会导致数值灾难（乘法里 base 设 0 → 永远 0）。

---

### 6.5.4 进阶：source 参数的设计——实体引用 vs 字符串标识

`SetModifier` 的第一个参数 `source` 是整个系统的灵魂——它接受两种类型：**实体（EntityScript 实例）** 或 **字符串/任意值**。

#### 实体作为 source：自动清理

看源码第 85-92 行的关键逻辑：

```85:92:scripts/util/sourcemodifierlist.lua
		if EntityScript.is_instance(source) then
            self._modifiers[source].onremove = function(source)
                self._modifiers[source] = nil
				self:RecalculateModifier()
            end

            self.inst:ListenForEvent("onremove", self._modifiers[source].onremove, source)
        end
```

**如果 source 是实体**，会自动：

1. **创建 onremove 回调**——实体被销毁时执行
2. **在 source 实体上 `ListenForEvent("onremove")`**——监听源实体的消失事件
3. 销毁时自动从 `_modifiers` 中清理对应条目

**使用场景**：

```lua
-- 喝了加速药水：药水实体是 source
inst.components.locomotor.externalmodifiers:SetModifier(potion_entity, 1.5)

-- 玩家丢下药水（药水实体销毁）
-- → SourceModifierList 自动收到 potion_entity 的 onremove 事件
-- → 自动 RemoveModifier
-- → 玩家的加速效果消失
```

**精髓**：**你不用自己写清理代码**——Klei 已经帮你捆绑了"源实体生命周期"和"modifier 生命周期"。

#### 字符串/任意值作为 source：手动管理

看 `scripts/prefabs/winona_battery_high.lua` 第 42-46 行：

```lua
if mult ~= 1 then
    inst.components.fueled.rate_modifiers:SetModifier(inst, mult, "efficiency")
else
    inst.components.fueled.rate_modifiers:RemoveModifier(inst, "efficiency")
end
```

**注意这里 `source` 传的是 `inst` 本身**——电池自己作为 source，key 用 `"efficiency"` 区分。

**为什么不用字符串？** 看另一个例子（`sanity.lua` 第 224 行）：

```lua
sources:SetModifier(source or self.inst, true)
```

**可以用字符串或实体**——只要是可以做 table key 的值都行。选哪个看场景：

| 选择 | 适用场景 | 好处 |
|------|---------|------|
| **传实体** | 这个实体"拥有"这个 modifier（如药水、buff 附着物、装备） | 实体销毁时自动清理 |
| **传实体自己** | 组件管理自己的 modifier | 方便，固定 source |
| **传字符串** | 全局/世界级 modifier | 语义明确，方便调试 |

#### 混合使用示例

一个组件可能同时有**几种不同 source 来源**：

```lua
-- sanity.lua 的 externalmodifiers 可能同时有：
sanity.externalmodifiers:SetModifier(hat_entity, 0.5)             -- 帽子（实体 source，戴上帽子时加）
sanity.externalmodifiers:SetModifier("codex_umbra", -0.2)          -- 暗影操纵书（字符串 source，全局规则）
sanity.externalmodifiers:SetModifier(self.inst, 1.0, "insanity")   -- 组件自己（实体 source + key）
```

**三种 source 混合**——每种都有各自的 key 空间。SourceModifierList 内部用一个 table 存所有 source（table key 可以是任意值），都能识别。

#### 实体 source 的技巧：替代手动 `ListenForEvent`

**对比两种写法**——同样效果，代码量差距巨大：

**手写**（locomotor.lua 老代码的做法）：

```lua
-- 注册
inst:ListenForEvent("onremove", function()
    self._multiplier_by_potion = nil
    self:RecalculateSpeed()
end, potion)

-- 清理（记得对称）
inst:RemoveEventCallback("onremove", <previous_fn>, potion)
```

**用 SourceModifierList**：

```lua
self._speed_modifiers:SetModifier(potion, 1.5)
-- 完。销毁自动清理。
```

**这就是抽象的威力**——8 行变 1 行。

---

### 6.5.5 进阶：key 参数 —— 同一来源多重 modifier

`SetModifier` 的第三个参数 `key` 有什么用？

#### 场景：同一来源多种效果

想象 **Wigfrid 唱了一首"冲锋歌"**，对自己同时施加：

- **速度加成** ×1.2
- **伤害加成** ×1.5（如果这两个都挂在同一个"buff 管理器"上）

Wigfrid 自己是 source，但要区分两种不同的 modifier。用 key：

```lua
inst.components.speed_modifiers:SetModifier(inst, 1.2, "song_haste")
inst.components.damage_modifiers:SetModifier(inst, 1.5, "song_attack")
```

#### 同一 source 多个 key 的数据结构

看源码第 80-82 行：

```80:82:scripts/util/sourcemodifierlist.lua
        self._modifiers[source] = {
            modifiers = { [key] = m },
        }
```

**内部结构**：

```lua
self._modifiers = {
    [source1] = {
        modifiers = {
            key1 = modifier1,
            key2 = modifier2,
            ...
        },
        onremove = <function>,  -- 仅当 source1 是实体时存在
    },
    [source2] = ...,
}
```

**两层索引**：先按 source 分组，再按 key 分支。

#### RecalculateModifier 如何遍历

看第 47-58 行：

```47:58:scripts/util/sourcemodifierlist.lua
function SourceModifierList:RecalculateModifier()
	local m = self._base
	for source, src_params in pairs(self._modifiers) do
        for k, v in pairs(src_params.modifiers) do
			m = self._fn(m, v)
        end
    end
	self._modifier = m
	if self._dirtycb then
		self._dirtycb(self.inst, m)
	end
end
```

**双层 for**——外层遍历 source，内层遍历每个 source 的所有 key——**每个 key 的值都参与叠加**。

所以如果 Wigfrid 上有：
- source = Wigfrid, key = "song_haste" = 1.2
- source = Wigfrid, key = "song_attack" = 1.5

那 `Get()` 返回 `base × 1.2 × 1.5 = 1.8`（如果这是一个 multiply 列表）。

#### key 的命名约定

| 场景 | key 命名示例 |
|------|--------------|
| buff 类 | `"speed_boost"`, `"attack_buff"`, `"defense_up"` |
| 装备类 | `"boots"`, `"amulet"`, `"helmet"` |
| 技能树 | `"wilson_torch_3"`, `"wx78_lunar_affinity"` |
| 状态类 | `"insanity"`, `"wet"`, `"frozen"` |
| 组件子系统 | `"rescale"`, `"efficiency"`, `"shard"` |

看 `winona_battery_high.lua` 用了 `"efficiency"`, `"shard"`, `"rescale"` 三个 key——**每个代表一个独立的影响源**。

#### 移除特定 key

**`RemoveModifier(source, key)`**（key 不为 nil）——只移除这个 key，其他 key 保留。

```lua
-- 移除冲锋歌的速度加成，但保留攻击加成
inst.components.speed_modifiers:RemoveModifier(inst, "song_haste")
-- "song_attack" 仍然生效
```

#### 不传 key：只有一个 modifier 的简写

```lua
-- 等价于 SetModifier(source, m, "key")
self.externalmodifiers:SetModifier(source, 1.5)
```

看源码第 69-71 行：

```lua
if key == nil then
    key = "key"
end
```

**默认 key 是字符串 `"key"`**——当同一 source 只有一个 modifier 时不用 key。

> **开发逻辑**：**key 参数让"同一来源的多重效果"变得优雅**。没有 key 时你需要为每个效果创建一个新 source（可能是假实体）——有了 key 可以**共享同一 source、分组管理**。这是少见的"**语义更重要**"设计——key 让代码可读性大幅提升。

---

### 6.5.6 进阶：自动清理机制（`onremove` event）

回到 `SetModifier` 第 85-92 行：

```lua
if EntityScript.is_instance(source) then
    self._modifiers[source].onremove = function(source)
        self._modifiers[source] = nil
        self:RecalculateModifier()
    end

    self.inst:ListenForEvent("onremove", self._modifiers[source].onremove, source)
end
```

这是 `SourceModifierList` 最精妙的设计——**自动清理**。我们把它的内部细节拆开讲。

#### 为什么要自动清理？

考虑一个场景：

```
t=0   玩家捡起加速药水 potion_A
t=0   speed_modifiers:SetModifier(potion_A, 1.5)
      现在玩家速度 ×1.5
t=10  玩家喝了药水（potion_A 实体消失）
      但 speed_modifiers._modifiers[potion_A] 还在！
      玩家速度永远 ×1.5
```

**这是"**幽灵 modifier**"问题**——source 实体没了，但 modifier 还挂着，永远生效。

#### Klei 的方案：绑定实体生命周期

**`ListenForEvent("onremove", fn, source)`**——在 source 实体上监听 onremove 事件。当 source 被销毁时：

1. source 的 `Remove` 函数被调用（5.2 节讲过）
2. 内部 `PushEvent("onremove")` 触发
3. **监听者（自动注册的 fn）被调用**
4. fn 里：`self._modifiers[source] = nil` + `RecalculateModifier()`

这样 modifier 的生命周期就**严格绑定**在 source 实体的生命周期上——**source 没了，modifier 也自动没**。

#### ListenForEvent 的三参数形式

注意 `ListenForEvent` 的用法：

```lua
self.inst:ListenForEvent("onremove", fn, source)
--                       [事件名]   [回调] [事件发射者]
```

**第三个参数 `source`** 告诉引擎"**监听的不是 self.inst 的事件，而是 source 的事件**"——这是 EntityScript 的事件系统标准用法，见 `entityscript.lua` 里的 `ListenForEvent` 实现。

**注意**：回调函数挂在 `self.inst` 上（在 `self.inst.event_listeners` 里），但绑定的是 source 的事件——**source 销毁时**会触发。

#### 对称地解除监听：`RemoveModifier`

看源码第 117-119 行：

```lua
if src_params.onremove ~= nil then
    self.inst:RemoveEventCallback("onremove", src_params.onremove, source)
end
```

**手动调 `RemoveModifier` 时会对称解除事件监听**——避免残留的回调在 source 实体上累积。

#### 为什么字符串 source 不自动清理？

回看第 85 行：

```lua
if EntityScript.is_instance(source) then
```

**只有 `source` 是实体时才注册 onremove**——字符串 source 没有"生命周期"概念，自然不能自动清理。

**用字符串 source 时，你必须手动调 RemoveModifier**——这和传实体是两种管理哲学，要注意区分。

#### 一个完整的自动清理示例

```lua
-- 制作一个"给玩家上 buff 的 Prefab"
local buff_inst = SpawnPrefab("speed_boost_buff")  -- 假设这是一个可见的 buff 实体
player.components.locomotor.externalmodifiers:SetModifier(buff_inst, 1.3)

-- 过了 30 秒
buff_inst:Remove()  -- 销毁 buff 实体

-- 下一帧：SourceModifierList 收到 buff_inst 的 onremove 事件
-- → 自动清理 player.components.locomotor.externalmodifiers 里的对应 modifier
-- → player 的速度回到原来的 1.0
```

**玩家的代码一行清理都没写**——全是 `SourceModifierList` 自动处理。这对 Mod 开发者意味着：**只要你用实体作为 source，就不用担心"忘了清理 buff"的 bug**。

---

### 6.5.7 老手进阶：SourceModifierList 的实战应用

这一节看真实组件怎么用 `SourceModifierList`——每个案例都体现了一种典型模式。

#### 案例 1：fueled 组件的单 SML

看 `scripts/components/fueled.lua` 第 87-90 行：

```87:90:scripts/components/fueled.lua
    self.maxfuel = 0
    self.currentfuel = 0
    self.rate = 1
	self.rate_modifiers = SourceModifierList(self.inst)
```

**一个组件就一个 SML**——默认 multiply，基础值 1.0。

**使用方**（`winona_battery_high.lua` 第 42-46 行）：

```lua
inst.components.fueled.rate_modifiers:SetModifier(inst, mult, "efficiency")
inst.components.fueled.rate_modifiers:SetModifier(inst, 0, "shard")
inst.components.fueled.rate_modifiers:SetModifier(inst, CalcFuelRateRescale(inst), "rescale")
```

**同一个 source（`inst`）** + **三个不同 key** —— 这是电池的三个独立效率调整源。

**关键领悟**：电池用 `inst` 自己作为 source 是一种简化——它的所有 modifier 都跟电池本身"同生共死"，不需要复杂的来源管理。

#### 案例 2：sanity 组件的多 SML

看 `scripts/components/sanity.lua` 第 78-116 行——**理智组件有 7 个 SourceModifierList**：

```lua
self._lunacy_sources = SourceModifierList(inst, false, SourceModifierList.boolean)      -- 月相模式
self.externalmodifiers = SourceModifierList(self.inst, 0, SourceModifierList.additive)  -- 外部回复速度
self.neg_aura_modifiers = SourceModifierList(self.inst)                                  -- 负光环倍率
self.neg_aura_immune_sources = SourceModifierList(inst, false, SourceModifierList.boolean)  -- 负光环免疫
self.sanity_aura_immune_sources = SourceModifierList(inst, false, SourceModifierList.boolean)  -- 全光环免疫
self.player_ghost_immune_sources = SourceModifierList(inst, false, SourceModifierList.boolean)  -- 玩家鬼魂免疫
self.light_drain_immune_sources = SourceModifierList(inst, false, SourceModifierList.boolean)  -- 黑暗流失免疫
```

**模式特征**：

- **一个组件可以有任意多个 SML**——每个管理一个独立维度
- **名字统一规范**：`_sources`（boolean 模式）、`_modifiers`（数值模式）
- **boolean 模式大量使用**——各种"免疫"、"模式启用"状态

#### 案例 3：slipperyfeet 的布尔 SML

看 `scripts/components/slipperyfeet.lua` 第 54-55 行：

```54:55:scripts/components/slipperyfeet.lua
	self.inst = inst
	self._sources = SourceModifierList(inst, false, SourceModifierList.boolean)
```

**经典的布尔 SML**——**"只要有一个来源说'踩冰上了'就启用滑倒逻辑"**。一旦所有 source 都移除（`IsEmpty`），就关闭更新任务。

#### 案例 4：efficientuser 的"SML 嵌套"

看 `scripts/components/efficientuser.lua` 第 18-22 行：

```18:22:scripts/components/efficientuser.lua
    if not self.actions[action] then
        self.actions[action] = SourceModifierList(self.inst)
    end

    self.actions[action]:SetModifier(source, multiplier)
```

**每个 action 一个 SML**——这是"**二维 modifier**"的经典模式：

```
self.actions = {
    [ACTIONS.CHOP] = SourceModifierList(...),
    [ACTIONS.MINE] = SourceModifierList(...),
    [ACTIONS.DIG]  = SourceModifierList(...),
    ...
}
```

**使用场景**：不同动作的"消耗减免"分别管理——伐木减免 ×0.8、挖矿减免 ×0.9 ⋯⋯每个动作独立计算。

#### 案例 5：wx78_shield 的 additive SML

看 `scripts/components/wx78_shield.lua` 第 38-39 行：

```38:39:scripts/components/wx78_shield.lua
    self.canshieldcharge = false
    self.chargegenerationsources = SourceModifierList(inst, 0, SourceModifierList.additive)
```

**additive 模式 + 基础值 0**——充能速率的加法叠加。

- 基础：0（没有任何来源时不充能）
- 月盾碎片加成：+0.5/秒
- 技能树解锁加成：+0.3/秒
- 最终：0 + 0.5 + 0.3 = 0.8/秒

这是典型的"**加速度叠加**"场景——默认 0、每个来源加一部分、最终总和。

#### 案例 6：locomotor 的**手写实现**（反面教材）

看 `scripts/components/locomotor.lua` 第 490-530 行——它**手写了 `SetExternalSpeedMultiplier / RemoveExternalSpeedMultiplier`**：

```lua
function LocoMotor:SetExternalSpeedMultiplier(source, key, m)
    -- 40 行的手写逻辑，和 SourceModifierList 做同样的事
end

function LocoMotor:RemoveExternalSpeedMultiplier(source, key)
    -- 20 行手写移除
end
```

**为什么不用 SourceModifierList？** 一种可能是**早期代码**（locomotor 是饥荒最早的组件之一，SourceModifierList 可能后出现）。

**但这是"反面教材"**——手写的代码和 SourceModifierList **做一样的事，但**：

- **不能复用**（两套实现）
- **容易出错**（手写 onremove 监听要小心）
- **难以 Mod**（其他组件用 SML，locomotor 要自己写配套代码）

**如果你写自定义组件，强烈建议直接用 SourceModifierList**——不要"重新发明轮子"。

> **开发逻辑**：**`SourceModifierList` 的通用性是它最大的价值**——任何涉及"多来源数值叠加"的场景都能用同一套 API。Klei 在新组件（如 `wx78_shield`、`fueled`）里全部使用它；老代码（如 `locomotor`）还保留着手写实现——这是**正在进行的"**统一重构**"**的遗迹。

---

### 6.5.8 老手进阶：SourceModifierList 的五个陷阱

#### 陷阱 1：base_value 选错导致数值灾难

```lua
-- ❌ 错：乘法列表用 0 作基础值
self.damage_modifiers = SourceModifierList(inst, 0, SourceModifierList.multiply)

-- 然后 SetModifier(source, 2) → Get() 返回 0 × 2 = 0！永远是 0
```

**原因**：乘法的中性元素是 **1 不是 0**——0 是乘法的"**零吸收元素**"。

**解决**：
- multiply → base = 1
- additive → base = 0
- boolean → base = false

---

#### 陷阱 2：忘了 RecalculateModifier 同步到组件字段

`Get()` 返回的是缓存值——SourceModifierList 在 `SetModifier / RemoveModifier` 时**自动**调用 `RecalculateModifier`。但如果你**直接**改了 `self._modifiers`（不推荐！），就要手动调用：

```lua
-- ❌ 极其危险的写法
self._modifiers[some_source] = {modifiers = {key = 1.5}}   -- 绕过了 API
-- Get() 返回的值是旧的！
```

**解决**：**永远用 `SetModifier` / `RemoveModifier` 公开 API**，别碰 `_modifiers` 私有字段。

---

#### 陷阱 3：dirtycb 回调里再次 SetModifier 导致递归

```lua
-- ❌ 错：dirtycb 里改 modifier
local function OnDirty(inst, value)
    inst.components.health.regen_modifiers:SetModifier(inst, value * 0.5, "derived")
    --                                   ↑ 这又会触发 RecalculateModifier → dirtycb → 递归
end

self.regen_modifiers = SourceModifierList(inst, 1, nil, OnDirty)
```

**症状**：爆栈。

**解决**：**dirtycb 里不要改同一个 SML**——要改就改别的组件。

---

#### 陷阱 4：实体 source 复用同一个引用

```lua
-- ❌ 错：两次 Set 不同 modifier 但 source 相同
inst.components.speed_mods:SetModifier(player, 1.5)  -- (没 key，默认 "key")
inst.components.speed_mods:SetModifier(player, 2.0)  -- 覆盖第一次！
-- 最终玩家速度 ×2.0（不是 ×1.5 × ×2.0）
```

**原因**：不传 key 时默认都是 `"key"`——两次 SetModifier 是**覆盖关系**，不是叠加。

**解决**：**同一 source 要叠加多个 modifier 必须用不同 key**：

```lua
inst.components.speed_mods:SetModifier(player, 1.5, "potion")
inst.components.speed_mods:SetModifier(player, 2.0, "buff")
-- 最终 1.0 × 1.5 × 2.0 = 3.0
```

---

#### 陷阱 5：字符串 source 没手动清理

```lua
-- ❌ 错：用字符串作为 source 后忘了清理
inst.components.speed_mods:SetModifier("temp_buff", 1.5)
-- 几分钟后 buff 应该结束，但没人调 RemoveModifier！
-- 玩家速度永远 ×1.5
```

**原因**：字符串 source 没有 "onremove" 事件——没有自动清理。

**解决**：

- **能用实体就用实体**（自动清理）
- **必须用字符串时**，配合 Timer 组件：

```lua
-- 用 Timer 管理字符串 source 的生命周期
inst.components.speed_mods:SetModifier("temp_buff", 1.5)
inst.components.timer:StartTimer("clear_temp_buff", 60)

inst:ListenForEvent("timerdone", function(inst, data)
    if data.name == "clear_temp_buff" then
        inst.components.speed_mods:RemoveModifier("temp_buff")
    end
end)
```

**这是 Timer + SourceModifierList 的经典配合**——一个管生命周期，一个管数值叠加。

#### 总结：何时用 SML 何时不用

**该用 SML**：

- 数值会被**多个独立来源**影响
- buff / debuff 的叠加
- 组件有"多种修正源"（如电池的 efficiency / shard / rescale）
- 多个免疫来源（boolean 模式）

**不该用 SML**：

- 数值只有一个来源（过度设计）
- 需要"有序"叠加（SML 内部遍历是无序的）
- 需要"非线性"计算（如 2 倍和 1.5 倍不一样结果——那是另一种逻辑）
- 只是"开关"状态（直接用 bool 字段或 tag）

---

### 6.5.9 小结

- **`SourceModifierList` 是"多来源叠加"的通用工具**——解决 buff/debuff 从多个独立来源影响同一数值时，如何叠加、如何清理、如何查询的三大问题。
- **构造函数 4 个参数**：`inst`（归属实体）、`base_value`（基础值）、`fn`（叠加函数）、`dirtycb`（值变化回调）——选错 `base_value + fn` 组合会导致数值灾难。
- **三大叠加函数**：`multiply`（默认，base=1，倍率）、`additive`（base=0，加值）、`boolean`（base=false，或运算）——还可以传自定义函数（如 max/min）。
- **核心 API 五件套**：`SetModifier(source, m, key?)` / `RemoveModifier(source, key?)` / `Get()` / `HasModifier(source, key)` / `Reset()`——99% 场景用这五个就够。
- **source 参数的双重语义**：**实体 source** 自动绑定生命周期（`onremove` 事件自动清理）；**字符串 source** 需要手动管理（配合 Timer 组件使用）。**能用实体就用实体**。
- **key 参数实现"同一来源多重效果"**：一个 source 下可以有多个独立 key，互不干扰——命名建议用 `"speed_boost"` / `"attack_buff"` 等语义化名字。
- **自动清理机制**：实体 source 下，SML 在 `SetModifier` 时自动 `ListenForEvent("onremove")`，source 销毁时自动移除——**0 行清理代码**。
- **六种实战应用**：fueled 单 SML（简单倍率）、sanity 多 SML（多维度）、slipperyfeet boolean（"任一生效"）、efficientuser 嵌套（每动作独立）、wx78_shield additive（加速度）、locomotor 手写（反面教材）。
- **五大陷阱**：base_value 和 fn 不匹配、改私有字段绕过 API、dirtycb 里递归 SetModifier、同一 source 不传 key 导致覆盖、字符串 source 忘了清理。
- **Timer + SML 是黄金组合**：Timer 管生命周期，SML 管数值叠加——**所有 buff 系统都可以用这个组合实现**。

> **下一节预告**：6.6 节我们讲 **如何阅读和理解一个组件的源码**——这是一个"**方法论**"章节，不再讲具体工具，而是教你"**看到一个陌生组件，怎么快速读懂它**"。Klei 的 350 个组件各有规律，掌握阅读技巧后你能独立理解任意组件的内部机制。


## 6.6 如何阅读和理解一个组件的源码

### 本节导读

前面五节（6.1-6.5）我们讲了**组件系统的核心机制**——生命周期、Replica、Classified、Timer、SourceModifierList。这些是"**地基知识**"。但饥荒联机版有 **350+ 个组件**——你不可能一个一个学完。

真正的高效路径是：**学会自己读组件源码**。遇到新组件不慌、改 Mod 不懵、调 bug 有思路——这是一种可迁移的能力。本节不再讲某一个具体组件，而是讲**"**阅读任何组件源码**"的通用方法论**。

> **新手**从 6.6.1-6.6.3 入手，学会"**读任何组件的五步标准流程**"、以及如何识别"**三个入口**"（Class 构造函数、公开方法、生命周期钩子）——掌握这些后，**再陌生的组件也能在 10 分钟内看懂骨架**；**进阶读者**从 6.6.4-6.6.6 深入"**反向追踪使用者**"、理解"**组件间的隐式依赖**"、以及用 `print` / `c_sel` 做**实时调试和探索**的技巧；**老手**可以跳到 6.6.7，我们拿 `perishable`（食物腐烂组件）作为**完整演示案例**——一步步走完整个阅读流程，最后 6.6.8 列出读 Klei 源码的五条黄金经验。

**前置**：假设你已经读过 6.1-6.5 节，对组件系统有整体认识。

---

### 6.6.1 快速入门：为什么要学会读组件源码？

写 Mod 的人迟早要面对一个问题——**官方文档很少，几乎所有"怎么实现 X 功能"都要从源码里挖**。

#### 场景 1：想写个"会漏气的气球"

**问题**：你要做一个能膨胀、会随时间漏气、漏光就爆炸的气球 Prefab。

**不会读源码的做法**：从零写一套"计时 + 状态机 + 爆炸效果"——可能要几百行代码。

**会读源码的做法**：
1. **意识到"漏气类似于燃料消耗"** → `fueled` 组件已经实现了"随时间消耗 + 到 0 时触发回调"
2. **`rg "AddComponent\(\"fueled\"\)"`** 找几个用它的 Prefab（如 `torch.lua`）
3. **10 分钟读懂** `fueled` 组件
4. **你的气球 = fueled + 自定义 onfinished 回调** —— 30 行代码搞定

**这就是学会读组件源码的第一个收益——"**偷懒**"**：不重复造轮子，直接用现成的能力。

#### 场景 2：想修改原版"长矛受雨天腐蚀更快"

**问题**：想让长矛在雨天耐久消耗更快。

**不会读源码的做法**：硬写一段每帧检查是否下雨的代码，然后手动扣耐久——很容易和其他 Mod 冲突。

**会读源码的做法**：
1. **意识到"耐久=finiteuses 组件"** → 读 `components/finiteuses.lua`
2. **看到它有 SourceModifierList 可以改消耗速率**（或用 `AddComponentPostInit` hook）
3. **一个 `WatchWorldState("israining", fn)` + 一个 SetModifier** —— 10 行代码、无冲突

**学会读源码的第二个收益——"**精准修改**"**：不蛮力加代码，用组件提供的正确扩展点。

#### 场景 3：调试一个 Bug

**问题**：玩家反馈"我的 Mod 自定义食物，放进冰箱后腐烂速度是正常的——但放进背包反而更快？"

**不会读源码的做法**：猜测、调参、盲改——可能改出新 bug。

**会读源码的做法**：
1. **腐烂逻辑在 `perishable.lua`** → 直接读 `Update` 函数（第 60-154 行）
2. **搜索 "fridge"、"modifier"** → 发现 `owner:HasTag("fridge")` 决定了冰箱 buff
3. **发现默认背包里的 modifier = 1**（没被 fridge tag 触发），而 "地上" 用的是 `TUNING.PERISH_GROUND_MULT`（0.5，更慢）
4. **原来"放地上反而更慢"是设计意图**——地上腐烂×0.5，背包里腐烂×1.0。这是 Klei 故意的！

**学会读源码的第三个收益——"**理解设计意图**"**：不被表面行为迷惑，看清内在设计。

#### 这三个场景的共同模式

| 能力 | 收益 |
|------|------|
| **知道有哪些组件可用** | 不重复造轮子 |
| **读懂组件的接口和扩展点** | 精准、无冲突的修改 |
| **理解组件的设计意图** | 真正"懂游戏"，能做出符合 Klei 风格的 Mod |

> **新手记忆**：**饥荒联机版的"文档"就在源码里**——你不看源码就等于蒙眼写代码。但好消息是：**Klei 的源码风格很规整，读起来比想象中容易**。

---

### 6.6.2 快速入门：读组件的五步法

面对一个陌生的组件文件，**按这个五步走一遍，90% 的组件都能在 15 分钟内搞懂**。

#### 第一步：扫一眼行数，判断复杂度

```bash
# 在项目根目录用 ripgrep 或直接看文件行数
wc -l scripts/components/perishable.lua
# 输出：354
```

| 行数 | 复杂度 | 典型例子 |
|------|--------|---------|
| < 100 | 简单 | `bait.lua`, `snowmandecor.lua` |
| 100-400 | 中等 | `perishable.lua`（354）, `fueled.lua`（300+） |
| 400-1000 | 复杂 | `health.lua`（800+）, `sanity.lua`（600+） |
| 1000+ | 核心 | `inventory.lua`（1400+）, `combat.lua`（1300+） |

**策略**：
- **< 100 行**：直接通读
- **100-400 行**：按下面五步法
- **400+ 行**：先读构造函数和公开方法签名，挑感兴趣的函数深入

#### 第二步：读第一段（Class 构造函数）

**构造函数告诉你："这个组件管理什么状态"**。

看 `perishable.lua` 第 29-48 行：

```29:48:scripts/components/perishable.lua
local Perishable = Class(function(self, inst)
    self.inst = inst
    self.perishfn = nil
    --self.perishtime = nil

    self.frozenfiremult = false

    self.targettime = nil
    --self.perishremainingtime = nil
    self.updatetask = nil
    self.dt = nil
    self.onperishreplacement = nil

    self.localPerishMultiplyer = 1
end,
nil,
{
    perishtime = onpercent,
    perishremainingtime = onpercent,
})
```

**从中得到的信息**：

| 字段 | 能推断 |
|------|--------|
| `self.perishfn` | 有**回调函数**机制（谁设置它的？`SetOnPerishFn`） |
| `self.frozenfiremult` | 有"**冷冻+火**"特殊逻辑——值得关注 |
| `self.perishremainingtime`（注释掉）| 用 nil 表示"**未开始计时**" |
| `self.updatetask` | 有**定时任务**——挂在组件上 |
| `self.localPerishMultiplyer` | 有**本地倍率**机制（拼写是个 typo 但原版就这么写） |

**最后的第三个参数 `{perishtime = onpercent, perishremainingtime = onpercent}`**——这是 **onset 回调**（6.1.8 讲过）：每次给 `perishtime` 或 `perishremainingtime` 赋值时，自动调用 `onpercent` 函数更新相关 tag。

**光读构造函数**，你已经知道 `perishable` 管 "**腐烂时间 + 状态标签 + 回调 + 倍率**" 四件事。

#### 第三步：列出所有公开方法（API 导览）

用 `rg "^function Perishable:"` 或直接浏览一遍：

```
function Perishable:OnRemoveFromEntity()      -- 清理
function Perishable:OnRemoveEntity()           -- 清理
function Perishable:IsFresh()                  -- 查询
function Perishable:IsStale()                  -- 查询
function Perishable:IsSpoiled()                -- 查询
function Perishable:Dilute(number, timeleft)   -- 堆叠时的时间平均
function Perishable:AddTime(time)              -- 延长腐烂时间
function Perishable:SetPerishTime(time)        -- 初始化设定
function Perishable:SetLocalMultiplier(newMult) -- 倍率
function Perishable:GetLocalMultiplier()        -- 查询
function Perishable:SetNewMaxPerishTime(newtime) -- 调整总时长
function Perishable:SetOnPerishFn(fn)          -- 设置回调
function Perishable:GetPercent()               -- 核心查询
function Perishable:SetPercent(percent)        -- 手动设置
function Perishable:ReducePercent(amount)      -- 扣除百分比
function Perishable:GetDebugString()           -- 调试输出
function Perishable:LongUpdate(dt)             -- 时间快进
function Perishable:StartPerishing()           -- 启动
function Perishable:IsPerishing()              -- 查询
function Perishable:Perish()                   -- 执行腐烂
function Perishable:StopPerishing()            -- 停止
function Perishable:OnSave()                   -- 存档
function Perishable:OnLoad(data)               -- 读档
```

**按语义分组**：

- **生命周期**：`StartPerishing` / `StopPerishing` / `IsPerishing` / `Perish`
- **时间操作**：`SetPerishTime` / `AddTime` / `SetNewMaxPerishTime` / `Dilute`
- **百分比操作**：`GetPercent` / `SetPercent` / `ReducePercent`
- **状态查询**：`IsFresh` / `IsStale` / `IsSpoiled`
- **倍率**：`SetLocalMultiplier` / `GetLocalMultiplier`
- **回调注册**：`SetOnPerishFn`
- **系统对接**：`OnSave` / `OnLoad` / `LongUpdate` / `OnRemoveFromEntity`

**光看方法名**，你已经基本知道这个组件"**能干什么**"——细节等要用的时候再查。

#### 第四步：读核心函数（通常是 Update / 主流程）

大多数组件的"**魂**"在一两个核心函数里。对 `perishable`，就是 `Update` 函数（第 60-154 行）——每次 tick 执行的腐烂计算。

**读核心函数的技巧**：**从顶部往下，边读边问"这一行在算什么？"**

看 `perishable.lua` 第 78-97 行：

```lua
if owner then
    if owner.components.preserver ~= nil then
        modifier = owner.components.preserver:GetPerishRateMultiplier(inst) or modifier
    elseif owner:HasTag("fridge") then
        if inst:HasTag("frozen") and not owner:HasTag("nocool") and not owner:HasTag("lowcool") then
            modifier = TUNING.PERISH_COLD_FROZEN_MULT
        else
            modifier = TUNING.PERISH_FRIDGE_MULT
        end
    elseif owner:HasTag("foodpreserver") then
        modifier = TUNING.PERISH_FOOD_PRESERVER_MULT
    elseif owner:HasTag("cage") and inst:HasTag("small_livestock") then
        modifier = TUNING.PERISH_CAGE_MULT
    end

    if owner:HasTag("spoiler") then
        modifier = modifier * TUNING.PERISH_GROUND_MULT
    end
else
    modifier = TUNING.PERISH_GROUND_MULT
    ...
end
```

**一眼看出设计意图**：**"物品在哪里决定腐烂倍率"**——冰箱、食物保鲜箱、笼子、地上、`preserver` 组件等。这就是 Klei 的"**环境感知腐烂系统**"。

#### 第五步：读生命周期钩子（OnSave / OnLoad / OnRemoveFromEntity）

这些钩子揭示了组件的"**状态持久化**"边界——哪些数据必存、哪些不存、读档时如何重建。

看 `perishable.lua` 第 331-351 行：

```331:351:scripts/components/perishable.lua
function Perishable:OnSave()
    return
    {
        paused = self.updatetask == nil or nil,
        time = self.perishremainingtime,
    }
end

function Perishable:OnLoad(data)
    if data ~= nil then
		if data.time ~= nil then
	        self.perishremainingtime = data.time
		end

        if data.paused then
            self:StopPerishing()
		elseif data.time ~= nil then
            self:StartPerishing()
        end
    end
end
```

**只保存两个字段**：`paused`（是否暂停中）和 `time`（剩余时间）。**不保存 `perishtime`**（假设 Prefab 工厂函数每次都会设定它）、**不保存 `updatetask`**（运行时对象重建就行）。

**读档逻辑**：有 `time` 就恢复它，根据 `paused` 决定是 `StopPerishing` 还是 `StartPerishing`——简单清晰。

#### 五步法总结

| 步骤 | 目的 | 耗时 |
|------|------|------|
| ① 扫行数 | 判断复杂度 | 1 秒 |
| ② 读构造函数 | 知道"管什么状态" | 1-2 分钟 |
| ③ 列公开方法 | 知道"能干什么" | 2-3 分钟 |
| ④ 读核心函数 | 知道"怎么工作" | 5-10 分钟 |
| ⑤ 读生命周期钩子 | 知道"持久化边界" | 1-2 分钟 |
| **总计** | **从 0 到看懂骨架** | **10-15 分钟** |

**这个流程对 90% 的组件都适用**——核心组件（`inventory`、`combat`）可能要 30-60 分钟，但方法不变。

---

### 6.6.3 快速入门：识别"三个入口"

除了五步法，我们还要学会识别**组件的"三个入口"**——这是快速定位关键代码的技巧。

#### 入口 ①：Class 构造函数

**位置**：组件文件的中上部，形如 `local XXX = Class(function(self, inst) ... end)`。

**告诉你**：
- **组件的状态字段清单**（所有 `self.xxx = ...`）
- **默认值**（推断使用模式）
- **onset 回调**（Class 的第三个参数，触发网络同步）

**读完构造函数，你就有了"组件的状态模型图"**。

#### 入口 ②：公开方法（`function XXX:YYY()`）

**位置**：构造函数下面的一连串 `function Perishable:Xxx(...)`。

**告诉你**：
- **组件对外提供什么能力**
- **有哪些事件回调可以注册**（`SetOnXxxFn`）
- **有哪些查询方法**（`IsXxx` / `GetXxx`）

**Klei 的命名约定**：

| 前缀 | 含义 |
|------|------|
| `Set` | 修改状态 |
| `Get` | 查询状态 |
| `Is` | 布尔查询 |
| `Has` | 布尔查询（存在性） |
| `Start / Stop` | 状态机控制 |
| `Can` | 权限/能力查询 |
| `SetOnXxxFn` | 注册回调 |
| `Do` | 动作触发 |
| `OnXxx` | 事件钩子 |

**看名字猜作用**——90% 准确。

#### 入口 ③：生命周期钩子和事件

**位置**：组件文件各处，但是有固定命名：

| 钩子 | 作用 |
|------|------|
| `Class(function(self, inst) end)` | 构造函数 |
| `function XXX:OnRemoveFromEntity()` | 组件被移除（6.1.6 节） |
| `function XXX:OnRemoveEntity()` | 实体被销毁 |
| `function XXX:OnEntitySleep()` / `OnEntityWake()` | 休眠 / 唤醒（6.1.5 节） |
| `function XXX:OnSave()` / `OnLoad(data)` | 存档 / 读档 |
| `function XXX:LongUpdate(dt)` | 离散时间补偿（玩家睡觉等） |
| `function XXX:TransferComponent(newinst)` | 实体形态切换 |

**事件监听**：通过 `inst:ListenForEvent("xxx", fn)` 或 `inst:PushEvent("xxx", data)` 可以找到组件会响应 / 触发的事件。

#### 三个入口的搜索命令（ripgrep 实战）

在终端里快速定位：

```bash
# ① 找构造函数
rg "^local Perishable = Class" scripts/components/perishable.lua

# ② 列所有公开方法
rg "^function Perishable:" scripts/components/perishable.lua

# ③ 列所有生命周期钩子
rg "^function Perishable:(On|Long|Transfer)" scripts/components/perishable.lua
```

**三个命令下去**，就掌握了组件的全部入口。

#### 对比一：简单组件 `bait`

看 `scripts/components/bait.lua`——极短（才 27 行左右）：

```lua
local Bait = Class(function(self, inst)
    self.inst = inst
end)
```

**构造函数空**——没有状态字段，只是一个"标记"。

**然后一两个方法**——比如 `SetTrap`、`GetTrap`。

**简单组件 = "空 Class + 几个方法"**——很多时候只是为了挂一个标签给其他组件识别用。

#### 对比二：核心组件 `health`

`scripts/components/health.lua` 有 800+ 行，构造函数：

```lua
local Health = Class(function(self, inst)
    self.inst = inst
    self.maxhealth = 100
    self.minhealth = 0
    self.currenthealth = self.maxhealth
    self.invincible = false
    self.takingfiredamage = false
    self.takingfiredamagetime = 0
    self.fire_damage_scale = 1
    self.externalabsorbmodifiers = SourceModifierList(self.inst, 0, SourceModifierList.additive)
    self.absorb = 0
    self.playerabsorb = 0
    self.canmurder = true
    self.canheal = true
    -- ...
end)
```

**20+ 个字段**——覆盖"血量、燃烧、吸收、伤害缩放、复活、禁止治疗/屠杀、月伤、充能区间等"——每个字段对应一整套方法和逻辑。

**复杂组件 = "状态字段越多，方法越多"**——一般看名字就知道职责。

> **新手记忆**：**复杂度和功能数是成正比的**——300 行组件约 20 个方法，900 行组件约 50 个方法。**行数是复杂度的直接指标**。

---

### 6.6.4 进阶：反向追踪 —— 找到"谁用了这个组件"

**光读组件自己的代码不够**——要理解它"怎么被用"。

#### 反向追踪的两种工具

**工具 1：`rg "components\.<组件名>"`**

找所有**使用者**（调用这个组件方法的代码）：

```bash
# 找谁在用 perishable 组件
rg "components\.perishable" scripts/

# 示例输出（精简版）：
# scripts/components/edible.lua:52:    local perish = self.inst.components.perishable
# scripts/prefabs/spoiledfood.lua:18:  if inst.components.perishable ~= nil then
# scripts/components/stewer.lua:139:   local perish = items[i].components.perishable
```

**工具 2：`rg "AddComponent\(\"<组件名>\"\)"`**

找所有**添加者**（哪些 Prefab 给实体加了这个组件）：

```bash
# 找哪些 Prefab 加了 perishable
rg 'AddComponent\("perishable"\)' scripts/prefabs/

# 示例输出：
# scripts/prefabs/berries.lua:83:     inst:AddComponent("perishable")
# scripts/prefabs/meat.lua:55:        inst:AddComponent("perishable")
# scripts/prefabs/preparedfoods.lua:67: inst:AddComponent("perishable")
```

#### 反向追踪的三个问题

对任何组件，追踪以下三个问题：

**Q1：谁加了这个组件？**

回答你"**这个组件挂在哪些实体上**"——知道它的"**受众**"。

```bash
rg 'AddComponent\("perishable"\)' scripts/prefabs/ | wc -l
# 输出：~80 个 Prefab 使用了 perishable
```

80 个——主要是食物类实体（莓果、肉、料理、蘑菇等）。**这说明 perishable 是一个"窄而深"的组件**——专门给食物用，但食物里几乎必备。

**Q2：谁调用了这个组件的方法？**

回答你"**组件的哪些方法最常用**"——从使用频率看哪些是核心接口。

```bash
# 找 perishable 被调用的所有方法
rg "components\.perishable:" scripts/ | awk -F'perishable:' '{print $2}' | awk -F'(' '{print $1}' | sort | uniq -c | sort -rn
```

**假设输出**：

```
45 GetPercent
32 SetPercent
28 IsFresh
22 StartPerishing
18 IsStale
15 IsSpoiled
10 SetPerishTime
...
```

**解读**：
- `GetPercent` 被调用 45 次——**最常用的查询**
- `SetPercent` / `StartPerishing` / `IsFresh` / `IsStale` / `IsSpoiled` 也是高频
- 说明这个组件的**"主用法"是获取/设置百分比 + 查询新鲜度状态**

**Q3：这个组件响应哪些事件？它发出哪些事件？**

```bash
# 它响应的事件（ListenForEvent）
rg 'ListenForEvent\("[^"]+"' scripts/components/perishable.lua

# 它发出的事件（PushEvent）
rg 'PushEvent\("[^"]+"' scripts/components/perishable.lua
```

对 `perishable`：

```bash
# PushEvent 找到的：
# "forceperishchange"  - tag 变化时
# "perishchange"       - 百分比变化时
# "perished"           - 完全腐烂时
```

**这三个事件就是组件的"对外广播"**——其他组件或 Prefab 可以监听这些事件做响应（UI 刷新、音效、连锁反应）。

#### 实战案例：追踪 `perishable` 的完整使用链

```
1. 追踪 AddComponent
   → berries.lua / meat.lua / preparedfoods.lua 等 80 个食物 Prefab
   → 几乎所有"会变质的东西"

2. 追踪调用
   → edible.lua（被吃掉时检查新鲜度，影响效果）
   → stewer.lua（煮饭时影响结果）
   → inventoryitem_replica（UI 显示腐烂度 icon）

3. 追踪事件
   → UI 监听 "perishchange" 重绘 icon
   → edible 组件监听 "perished" 自我销毁或变 spoiledfood
```

**整个"食物腐烂生态"的图景一下就清晰了**——perishable 不是孤立的，它和 edible、stewer、inventoryitem 深度联动。

#### 在 Cursor 里的快捷玩法

作为 Cursor 用户，可以用 **SemanticSearch 或 Grep 快速完成这些搜索**——比终端 rg 还方便。

> **开发逻辑**：**组件不是孤岛**——它的价值在于"**和谁配合**"。反向追踪帮你看清组件在整个游戏系统中的"社交网络"——这是写优秀 Mod 的必备能力。

---

### 6.6.5 进阶：理解组件间的隐式依赖

很多组件"**看似独立**"，实际上**隐式依赖其他组件**才能工作。这种依赖在源码里有明显的痕迹。

#### 依赖痕迹 1：`self.inst.components.xxx` 引用

看 `perishable.lua` 第 68-71 行：

```lua
local owner = inst.components.inventoryitem and inst.components.inventoryitem.owner or nil
if not owner and inst.components.occupier then
    owner = inst.components.occupier:GetOwner()
end
```

**关键词**：`inst.components.inventoryitem`、`inst.components.occupier`——**perishable 依赖这两个组件**。

**说明**：perishable 的完整工作需要 `inventoryitem`（可以被拿起）或 `occupier`（占位符——用于蜂巢等）。如果你给一个没有 `inventoryitem` 的实体加 perishable，**owner 永远是 nil**，行为会不完整。

#### 依赖痕迹 2：`self.inst:HasTag` 检查

同文件第 82 行：

```lua
if inst:HasTag("frozen") and not owner:HasTag("nocool") and not owner:HasTag("lowcool") then
```

**关键词**：`HasTag("frozen")` / `HasTag("nocool")` / `HasTag("lowcool")`——**perishable 依赖这些 tag**。

**说明**：如果你的实体要用到"frozen"加成，必须先给实体加 `"frozen"` tag（通常是 `freezable` 组件加上去的）。**Tag 是组件间的"通信协议"**（回顾 5.5 节）。

#### 依赖痕迹 3：`PushEvent` 触发

同文件第 176 行：

```lua
self.inst:PushEvent("perishchange", {percent = self:GetPercent()})
```

**说明**：perishable 发出 `"perishchange"` 事件——**谁在监听这个事件**？`rg 'ListenForEvent\("perishchange"'`：

```bash
# 结果：
# scripts/components/inventoryitem_replica.lua: inst:ListenForEvent("perishchange", ...)
# scripts/components/edible.lua: 可能也监听（用于显示 UI）
```

**这意味着**：perishable 的 perishchange 事件被 `inventoryitem_replica` 消费——所以**perishable 的网络同步依赖于 inventoryitem_replica**。如果你给一个没有 inventoryitem 的实体加 perishable，UI 显示会有问题。

#### 依赖痕迹 4：TUNING 常量

同文件第 83 行：

```lua
modifier = TUNING.PERISH_COLD_FROZEN_MULT
```

**关键词**：`TUNING.PERISH_XXX_MULT`——**perishable 依赖 TUNING 表的配置**。

**说明**：所有腐烂倍率都在 `scripts/tuning.lua` 里定义。如果你想"**整体调整腐烂节奏**"，改 TUNING 常量一次改所有——不用改组件本身。

#### 依赖梳理表

对一个中等复杂度组件（perishable），整理的依赖关系：

| 依赖 | 具体项 | 关键性 |
|------|--------|--------|
| 组件（硬依赖）| `inventoryitem` 或 `occupier` | 高——没它 owner 相关逻辑失效 |
| 组件（软依赖）| `stackable`（仅 Dilute 用）| 低 |
| Tag | `"frozen"`, `"fresh"`, `"stale"`, `"spoiled"` | 高——tag 是状态广播 |
| 容器 Tag | `"fridge"`, `"foodpreserver"`, `"cage"`, `"spoiler"` | 高——改变倍率 |
| 事件 | `"perishchange"` / `"perished"` / `"forceperishchange"` | 高——UI 同步 |
| TUNING 常量 | `PERISH_GROUND_MULT` / `PERISH_FRIDGE_MULT` 等 | 高——行为配置 |
| World 状态 | `TheWorld.state.temperature` / `israining` | 中——环境感知 |

**这张表就是 perishable 的"**生态依赖图**"**——写 Mod 扩展/改造 perishable 时，必须了解这些依赖点。

#### Mod 实战建议

根据依赖梳理得到几个 Mod 开发经验：

1. **加 perishable 前先加 inventoryitem**（或 occupier）
2. **想改腐烂倍率，**先改 TUNING**（用 modmain 的 `TUNING.PERISH_GROUND_MULT = 2.0`），而不是 hook Update 函数
3. **自己的"**类冰箱容器**"**—— 加上 `"fridge"` tag 就能自动让里面食物减慢腐烂
4. **自己的"**加速腐烂设备**"**—— 加上 `"spoiler"` tag 就能让里面食物腐烂更快

**这些都是基于对"组件依赖 + 扩展点"的理解**——没读组件源码根本想不到。

---

### 6.6.6 进阶：用 `print` / `c_sel` 实时探索

光读代码是"**静态理解**"——要真正搞懂一个组件，还要**运行时看它的实际行为**。饥荒联机版的控制台给了我们极强的实时探索工具。

#### 工具 1：`c_sel` —— 选中当前鼠标指向的实体

打开控制台（默认 **`~` 键**），输入：

```
c_sel()
```

鼠标指向一个实体时它会被"选中"——后续用 `c_sel()` 引用。**这是所有调试的起点**。

如果已经选中，再次 `c_sel()` 返回这个实体对象。

#### 工具 2：检查组件字段

```lua
c_sel().components.perishable.perishremainingtime
-- 输出：280.5（秒）

c_sel().components.perishable.perishtime
-- 输出：960

c_sel().components.perishable:GetPercent()
-- 输出：0.29
```

**一秒钟看清当前腐烂状态**——比读代码更直观。

#### 工具 3：动态修改

```lua
c_sel().components.perishable:SetPercent(0.5)
-- 腐烂度立刻变成 50%

c_sel().components.perishable:Perish()
-- 立刻让它完全腐烂（触发 perishfn 回调）
```

**这是调试 Prefab 的最快方式**——把组件"**拨到特定状态**"看后续行为。

#### 工具 4：给组件加 print 监听

**想实时看某个事件**：

```lua
-- 在控制台粘贴
local inst = c_sel()
inst:ListenForEvent("perishchange", function(_, data)
    print("[DEBUG] perishchange", data.percent)
end)
```

**之后每次腐烂度变化都会 print**——放桌子上看几分钟就知道 tick 频率和变化幅度。

#### 工具 5：临时 hook 方法

```lua
-- 临时 hook GetPercent，看谁在调用
local inst = c_sel()
local old_GetPercent = inst.components.perishable.GetPercent
inst.components.perishable.GetPercent = function(self)
    print("[DEBUG] GetPercent called", debug.traceback())
    return old_GetPercent(self)
end
```

**你能看到调用堆栈**——一目了然地看"**谁在调用 GetPercent**"。调完 5 分钟后记得恢复或重启游戏。

#### 工具 6：`GetDebugString` —— 组件自备的调试输出

很多组件有 `GetDebugString()` 方法。看 `perishable.lua` 第 242-251 行：

```242:251:scripts/components/perishable.lua
function Perishable:GetDebugString()
	if self.perishremainingtime and  self.perishremainingtime > 0 then
        return string.format("%s %2.2fs %s",
            self.updatetask and "Perishing" or "Paused",
            self.perishremainingtime,
            self.frozenfiremult and "frozenfiremult" or "")
	else
		return "perished"
	end
end
```

按 **Ctrl + Tab**（默认键位，可能不同）打开实体调试面板——它会自动调用 `GetDebugString` 显示信息，**一次看完所有关键字段**。

**`GetDebugString` 是 Klei 留给开发者的"**仪表盘**"**——大多数核心组件都有。

#### 工具 7：`rg` + `print` 组合找调用链

**问题**："为什么我的食物刚放进背包就腐烂了？"

**侦探式调试**：

```
1. c_sel() 选中食物
2. c_sel().components.perishable.perishremainingtime
   → 发现只剩 5 秒
3. 在控制台 hook SetPercent：
   inst.components.perishable.SetPercent = function(self, p)
       print("[DEBUG] SetPercent", p, debug.traceback())
       -- 继续原逻辑
   end
4. 放进背包 → 观察堆栈
5. 发现是某个 Mod 调用了 SetPercent(0.01) → 找到元凶
```

**这种"**运行时侦探**"能力非常强**——源码读千遍不如 print 一次。

#### 工具 8：临时打开 godmode / 切时间观察

```
c_godmode()        -- 开启无敌
c_speedmult(5)     -- 5 倍速度
LongUpdate(3600)   -- 时间快进 1 小时
```

**快进时间能快速看到"**长期演变**"**——比如腐烂、成长、日夜循环。

#### 综合调试流程

组合起来，一个"**调试陌生组件**"的完整流程：

```
①  读源码（6.6.2 五步法）——了解基本结构
②  反向追踪（6.6.4）——了解使用场景
③  游戏中 c_spawn 一个相关实体
④  c_sel() 选中
⑤  敲几个 inst.components.xxx 查询
⑥  按 Ctrl + Tab 看 GetDebugString
⑦  临时 hook 关键方法 + print
⑧  操作触发场景（吃、烧、砍、冻）
⑨  对照 print 输出和源码逻辑
⑩  搞清楚 → 写 Mod
```

**整个过程 20-30 分钟**——比盲写 Mod 调半天 bug 快得多。

> **开发逻辑**：**饥荒的控制台 + 组件系统是"**可探索游戏世界**"的极佳组合**——你可以像实验室一样操作游戏对象，实时观察结果。这种**探索式学习**比"读书式学习"快 10 倍。学会用它，你就从"Mod 新手"变成"Mod 侦探"。

---

### 6.6.7 老手进阶：完整案例 —— 逐行读懂 `perishable.lua`

现在我们把前面所有技巧**综合运用**，完整地"**走一遍**"`perishable.lua` 的阅读过程——就像真实遇到这个陌生组件时一样。

#### Step 1：情境 & 扫行数

**情境**：你想写一个 Mod "**夏天腐烂更快的食物**"，怀疑要 hook `perishable`。

**扫行数**：

```bash
wc -l scripts/components/perishable.lua
# 354
```

**预估 10-15 分钟读完**——中等复杂度。

#### Step 2：读构造函数（1 分钟）

看第 29-48 行（已在 6.6.2 第二步贴过）——**8 个字段 + 2 个 onset**。推断这是"**有定时任务 + 管百分比 + 有回调**"的组件。

#### Step 3：列所有公开方法（2 分钟）

已经在 6.6.2 第三步整理过——**23 个方法**，分为 7 组。其中核心是：

- `StartPerishing / StopPerishing / Perish` —— 生命周期
- `GetPercent / SetPercent` —— 主查询/设置
- `OnSave / OnLoad / LongUpdate` —— 存档/离散时间

#### Step 4：跳到 `Update`（核心函数）（8 分钟）

第 60-154 行。按块阅读：

**块 1（60-67 行）**：初始化

```lua
local function Update(inst, dt)
	local self = inst.components.perishable
    if self ~= nil then
		dt = self.start_dt or dt or (10 + math.random()*FRAMES*8)
		self.start_dt = nil

        local additional_decay = 0
		local modifier = 1
```

- `dt` 的三级 fallback——`start_dt`（启动时延迟）、参数传入、默认随机
- `modifier` 初始 1（乘法中性元素）
- `additional_decay` 额外衰减量（用于酸雨）

**块 2（68-76 行）**：获取 owner 和环境

```lua
local owner = inst.components.inventoryitem and inst.components.inventoryitem.owner or nil
if not owner and inst.components.occupier then
    owner = inst.components.occupier:GetOwner()
end

local pos = owner ~= nil and owner:GetPosition() or self.inst:GetPosition()
local inside_pocket_container = owner ~= nil and owner:HasTag("pocketdimension_container")

local ambient_temperature = inside_pocket_container and TheWorld.state.temperature or GetTemperatureAtXZ(pos.x, pos.z)
```

**关键发现**：
- owner 可能来自 `inventoryitem.owner` 或 `occupier` 组件——**依赖这两个组件之一**
- 温度来自 **世界坐标处的温度**（不是全局温度）——**地点相关**
- 口袋容器特殊处理——**用世界整体温度**

**块 3（78-105 行）**：容器倍率（已在 6.6.2 贴过）

- `preserver` 组件 > `"fridge"` tag > `"foodpreserver"` tag > `"cage"` tag
- 无 owner 时用 `GROUND_MULT`
- 酸雨时额外衰减

**块 4（107-130 行）**：各种环境因素

```lua
if inst:GetIsWet() and not self.ignorewentness then
    modifier = modifier * TUNING.PERISH_WET_MULT
end

if ambient_temperature < 0 then
    if inst:HasTag("frozen") and not self.frozenfiremult then
        modifier = TUNING.PERISH_COLD_FROZEN_MULT
    else
        modifier = modifier * TUNING.PERISH_WINTER_MULT
    end
end
-- ... summer mult / frozen fire mult ...

modifier = modifier * self.localPerishMultiplyer
modifier = modifier * TUNING.PERISH_GLOBAL_MULT
```

**关键发现**：
- 湿的加速腐烂
- 低温减速 / 冷冻+火 的特殊处理
- 高温加速
- **本地倍率 + 全局倍率最后乘进来**

**块 5（131-137 行）**：更新 remainingtime

```lua
local old_val = self.perishremainingtime
if self.perishremainingtime then
    self.perishremainingtime = math.min(self.perishtime, self.perishremainingtime - dt * modifier - additional_decay)
    if math.floor(old_val*100) ~= math.floor(self.perishremainingtime*100) then
        inst:PushEvent("perishchange", {percent = self:GetPercent()})
    end
end
```

**核心一步**——扣除 `dt * modifier + additional_decay`，限制在 `[0, perishtime]` 范围。`math.floor(x*100)` 是**百分位级粒度**——只在百分比变化时才触发事件（节省带宽）。

**块 6（139-147 行）**：食物降温

```lua
if inst.components.edible ~= nil and inst.components.edible.temperaturedelta ~= nil and ... then
    if owner ~= nil and owner:HasTag("fridge") then
        inst.components.edible:AddChill(1)
    elseif ambient_temperature < TUNING.OVERHEAT_TEMP - 5 then
        inst.components.edible:AddChill(.25)
    end
end
```

**不是腐烂相关的事**——**hot food 在 Perishable 里处理冷却**！这是**"**和 edible 组件协作**"**的典型表现。

**块 7（149-153 行）**：触发腐烂

```lua
if self.perishremainingtime and self.perishremainingtime <= 0 then
    self:Perish()
end
```

**结束条件**——降到 0 时调 `Perish()` 结束游戏。

#### Step 5：读 OnSave / OnLoad（2 分钟）

已在 6.6.2 第五步分析过——**只存 `time` 和 `paused`**，简洁清晰。

#### Step 6：回到"夏天腐烂更快"的 Mod 设计

现在你已经完整理解了 `perishable`，可以设计 Mod 了：

**思路 1**：Mod 直接提高 `TUNING.PERISH_SUMMER_MULT`

```lua
-- modmain.lua
TUNING.PERISH_SUMMER_MULT = TUNING.PERISH_SUMMER_MULT * 1.5  -- 夏天腐烂 1.5 倍更快
```

**优点**：一行代码、不破坏任何逻辑。

**缺点**：影响所有物品（不能针对特定 Prefab）。

**思路 2**：hook 特定 Prefab 的 perishable 组件

```lua
AddPrefabPostInit("meat", function(inst)
    if not TheWorld.ismastersim then return end

    if inst.components.perishable then
        local old_Update = inst.components.perishable.Update  -- 啊，Update 是 local function，拿不到
        -- ...
    end
end)
```

**发现问题**：`Update` 是 **`local function`**——外部拿不到！只能 hook 组件的公开方法。

**思路 3**：用 `SetLocalMultiplier`

查方法表：**`SetLocalMultiplier(newMult)` 是公开方法！**——可以通过它乘一个额外倍率：

```lua
AddPrefabPostInit("meat", function(inst)
    if not TheWorld.ismastersim then return end

    if inst.components.perishable then
        inst:WatchWorldState("season", function(inst, season)
            if season == "summer" then
                inst.components.perishable:SetLocalMultiplier(2.0)
            else
                inst.components.perishable:SetLocalMultiplier(1.0)
            end
        end)
    end
end)
```

**完美**——10 行代码实现"夏天腐烂 2 倍"效果，只影响 meat。

#### Step 7：总结流程

通过 6 步阅读 + 1 步 Mod 设计，**30 分钟内**：

- 看懂了 `perishable.lua` 的完整设计
- 发现了 `localPerishMultiplyer` 这个贴心扩展点
- 写出了一个无冲突、无副作用的 Mod

**这就是"**学会读源码**"的实战价值**——不是为了读而读，而是为了更好地 Mod。

---

### 6.6.8 老手进阶：阅读 Klei 源码的五条黄金经验

经过 6.1-6.6 这一大圈学习，总结出读 Klei 源码的"**元经验**"——这些对读任何 Lua 游戏代码都适用。

#### 经验 1：**命名规律是阅读的"路标"**

Klei 的命名**非常规整**，掌握几条规则能大幅加速阅读：

| 命名 | 含义 |
|------|------|
| `XxxYyyFn` | 回调函数字段 |
| `SetOnXxxYyy` | 设置回调 |
| `_privateXxx` | 私有字段（下划线前缀） |
| `XxxYyy_replica` | replica 文件 |
| `Xxx_classified` | classified Prefab |
| `MakeXxx` | 辅助函数（5.3 节的 `MakeInventoryPhysics` 等）|
| `TUNING.XXX_YYY` | 配置常量 |

**看到名字就知道角色**——不用每次看实现。

#### 经验 2：**注释是金子，别跳过**

Klei 代码里的注释分三种，都值得关注：

**① 普通 `--` 注释**：一般是逻辑说明，直接读。

**② `--V2C:` 注释**：**V2C 是 Klei 一位核心开发的签名**——V2C 留下的注释通常是"**关键设计决策**"或"**坑的说明**"。

看 `inventoryitem_classified.lua` 第 37-41 行：

```lua
--V2C: used to force color refresh when spoilage changes around 50%/20%
local function ForcePerishDirty(inst)
    inst.perish:set_local(inst.perish:value())
    inst.perish:set(inst.perish:value())
end
```

这种看似奇怪的"自己 set 自己"代码，V2C 解释了原因——**不看注释永远搞不懂**。

**③ 注释掉的代码块（`--[[ ... ]]`）**：可能是**历史版本、废弃 API、或特殊情况的说明**——值得一读，因为它反映了代码演化。

#### 经验 3：**TUNING 是"配置大全"**

`scripts/tuning.lua` 有 **8000+ 行**——包含游戏几乎所有数值。

**技巧**：遇到 `TUNING.XXX` 常量就去 `tuning.lua` 里搜 —— 能看到**原始设计数值**和**设计者意图**（通过常量的分组可以看出）。

**应用**：
- 想调整 Mod 数值：优先改 TUNING（全局一致性）
- 想知道"这个数值为什么是 60"：搜 TUNING 可能有注释

#### 经验 4：**`require "debugtools"` 是宝藏**

游戏内控制台里 `c_xxx` 命令大多定义在 `scripts/debugcommands.lua` 里。**读它**可以发现**隐藏的调试能力**：

```bash
rg "^function c_" scripts/debugcommands.lua
```

**精选输出**：

- `c_spawn("prefab")` —— 生成实体
- `c_give("prefab")` —— 直接放进背包
- `c_sel()` —— 选中
- `c_goto(x, y, z)` —— 传送
- `c_godmode()` —— 无敌
- `c_speedmult(x)` —— 速度倍率
- `c_heal()` / `c_setminimap(true)` —— 各种开挂命令

**为 Mod 开发者提供的无尽便利**——你甚至可以自己用 `AddGlobal` 加自己的 `c_xxx` 命令。

#### 经验 5：**注意"两种 Klei 风格"**

Klei 不同时期的代码风格不同，能从中推断时期：

| 风格特征 | 时期 | 示例 |
|----------|------|------|
| 大量 `_private` 字段 + metatable | 新 | `health_replica` |
| 直接 `self.xxx = net_bool(...)` | 新 | `inventoryitem_classified` |
| 手写 `_externalspeedmultipliers` 表 | 老 | `locomotor` |
| 用 `SourceModifierList` | 新 | `fueled`, `sanity` |
| 用 `SetFn` 风格注册回调 | 新 | 几乎所有新组件 |
| 硬编码 if-else | 老 | 部分 Prefab 文件 |

**读代码时留意这些信号**——新风格代码更规整、更容易模仿；老风格有历史包袱，模仿前想想"**是不是有新的方式可以用**"。

**具体**：想做"多来源叠加"时，**不要模仿 locomotor 的手写方式**——用 SourceModifierList。

#### 额外经验：善用 Cursor / AI 辅助

最后一条——既然你在用 Cursor，**充分利用 AI 的帮助**：

- 遇到看不懂的 50 行代码：**让 AI 总结**
- 想找某个功能在哪实现：**用 SemanticSearch 问**
- 想对比新老实现：**让 AI 列出差异**

**但记住**：AI 可能出错。对于**关键细节**要**自己对源码验证**一遍——这是"信任但验证"的态度。

---

### 6.6.9 小结

- **学会读组件源码是写好 Mod 的核心能力**——三大收益：不重复造轮子、精准无冲突修改、理解 Klei 设计意图。
- **五步读法**：① 扫行数（判断复杂度）→ ② 读构造函数（理解状态模型）→ ③ 列公开方法（了解能力）→ ④ 读核心函数（掌握主流程）→ ⑤ 读生命周期钩子（存档边界）。**中等组件 15 分钟搞定**。
- **组件的"三个入口"**：Class 构造函数（状态模型）、公开方法（对外能力）、生命周期钩子（系统集成）——用 `rg "^function Xxx:"` 可以快速列出所有方法。
- **命名前缀速查**：`Set/Get/Is/Has/Can` 是查询/修改、`SetOnXxxFn` 是注册回调、`Start/Stop` 是状态机——看名字猜功能 90% 准确。
- **反向追踪三问**：谁加了这个组件（`rg AddComponent`）、谁用了它的方法（`rg components.xxx`）、它响应/发出哪些事件（`rg ListenForEvent/PushEvent`）——画出组件的"**社交网络**"。
- **组件间的隐式依赖**：通过 `self.inst.components.xxx` 引用、`HasTag` 检查、`PushEvent` 事件、`TUNING.XXX` 常量——这四类痕迹揭示了组件的依赖图。
- **实时探索工具**：`c_sel()` 选中实体、`c_sel().components.xxx` 查字段、`Ctrl+Tab` 看 `GetDebugString`、临时 hook 方法加 `print`——**"**运行时侦探**"比静态读代码快 10 倍**。
- **完整案例（perishable）演示**：30 分钟从"听说过 perishable"到"写出夏天加速腐烂的 Mod"——读+Mod 设计一气呵成。
- **五条黄金经验**：命名规律是路标、V2C 注释是金子、TUNING 是配置大全、`debugcommands.lua` 藏宝、区分新老 Klei 风格——加上用 AI 加速但要验证。

> **下一节预告**：最后一节 6.7——**实战**：从零开始写一个**带 Replica 的自定义组件**。我们会把 6.1-6.6 所有知识点串起来——组件生命周期、Replica、Classified、net 变量、SourceModifierList、Timer⋯⋯写一个可用于真实 Mod 的"灵能"组件（对应 5.7 节的"灵能短剑"）。


## 6.7 实战：编写一个自定义组件（含 Replica）

### 本节导读

前 6 节我们一直在**读**——读 `health.lua`、`fueled.lua`、`perishable.lua`，读 `entityreplica.lua`、`entityscript.lua`，读 `SourceModifierList`，读完一遍组件系统的"教科书"。这一节我们换个姿势——**自己动手写**。

我们要做的，是**把第 5.7 节的故事延伸下去**：

> 5.7 节我们做了一把"**灵能短剑**"——攻击时消耗玩家理智作为施法代价。但"灵能"这种东西本身**只是被吸食的对象**，它从哪里来、能否储存、能否反过来释放？这一节我们把"灵能"**独立成一个真正的组件 `psionic`**，并给它配一颗可以拾起、自动充能、充满后释放净化效果的**灵能晶簇**（`psionic_crystal`）。

这个例子之所以选它，是因为它能**完整覆盖 6.1-6.6 的全部知识点**：

| 6.1-6.6 学到的知识 | 在 6.7 中的对应实现 |
|--------------------|-------------------|
| 6.1 组件生命周期五钩子 | `psionic` 完整实现 OnSave/OnLoad/OnEntitySleep/OnEntityWake/OnRemoveFromEntity |
| 6.1 onset 同步机制 | 用 onset 在 `current/max` 变化时自动 PushEvent + 同步给 replica |
| 6.2 Replica 客户端影子 | 写一份 `psionic_replica.lua`，用 net_smallbyte 同步给客户端 |
| 6.3 net 变量 / 父子结构 | 选用**轻型 net_xxx 方案**（不依赖 classified 实体），并解释何时该用 classified |
| 6.4 Timer 与定时任务 | 用 `DoPeriodicTask` 实现自动充能；附带"何时改用 timer 组件"的判断 |
| 6.5 SourceModifierList | 充能率支持多来源叠加（"附近有花/有恶魔花/装备 buff"等都能影响速率） |
| 6.6 阅读组件源码的方法 | 组件写完后用 6.6 的"五步读法"自己再读一遍，确认结构清晰 |

> **新手**从 6.7.1-6.7.3 起步——理解项目目标、建好文件骨架、写出最小可运行版本，进游戏 `c_spawn("psionic_crystal")` 就能看到一颗会自动发光的水晶；**进阶读者**继续到 6.7.4-6.7.6，把存档恢复、休眠优化、Replica 同步都接好——它就是一颗"工业级"的水晶了；**老手**跳到 6.7.7-6.7.8，看 `SourceModifierList` 的实战用法、调试技巧、五个常见陷阱，以及如何把这个组件**反过来挂到玩家身上**变成"法力值条"。最后 6.7.9 把 6.1-6.6 所有知识点和这一节的代码做对照表，整章闭环。

> **重要前置说明**：本节涉及的资源（`.tex`/`.xml`/`.zip`）我们**全部复用游戏里已有的紫宝石**（`purplegem`），不需要你做任何美术工作——这样你可以专注在 Lua 代码上。如果你想换成自己的图标，参考 5.7.5 节的步骤即可。

---

### 6.7.1 快速入门：项目目标 —— 一颗能自动充能的"灵能晶簇"

#### 第一步：先讲清楚要做什么

我们要做的东西，从玩家视角看是这样的：

| 维度 | 设定 |
|------|------|
| **名字** | 灵能晶簇（`psionic_crystal`） |
| **外观** | 借用游戏里紫宝石（`purplegem`）的模型和图标 |
| **行为** | 放在地上时，它会**自动以每秒 1 点的速度充能**，最多充到 100 |
| **互动** | 充满后**自动触发"灵能净化"**——使用方式见下文 |
| **可交互** | 玩家可以拾起放进背包、丢出去、在地上扔下，统统支持 |
| **存档** | 退出游戏再进——剩余灵能值**保持不变** |
| **休眠** | 玩家走远——晶簇停止 tick；走回来——按真实时间补充刚才"应充上的"灵能 |
| **多源叠加** | 附近有**罂粟花**时充能 ×1.5、附近有**恶魔花**时充能 ×0.5（演示 `SourceModifierList`） |
| **客户端可见** | 客户端可以通过 `inst.replica.psionic:GetPercent()` 读到当前百分比（用于 UI 显示） |

实现这些功能，我们需要**一个新组件 `psionic` + 一个新 Prefab `psionic_crystal`**。

#### 第二步：把功能拆成"组件能力"和"Prefab 配置"两层

回顾 5.1 节我们讲过的"**Prefab 是图纸、组件是能力模块**"——这次我们要清晰地把功能分到两层：

**`psionic` 组件（能力层）** —— 不知道自己挂在什么实体上，只负责"灵能值的管理"：

```
[职责清单]
- 维护当前灵能值 / 最大灵能值
- 自动充能（每秒按 rate 增加）
- 充能率支持多来源叠加（rate_modifiers）
- 满了 → PushEvent("psionic_full")
- 空了 → PushEvent("psionic_empty")
- 数值变化 → PushEvent("psionicchange") 通知所有监听者
- 存档/读档自动恢复
- 休眠时停 tick、唤醒时按时间补偿
- 提供 Burst() 方法：消耗灵能释放范围效果
```

**`psionic_crystal` Prefab（具体物体层）** —— 决定"要什么样的实体":

```
[职责清单]
- 一个可拾起的物品（inventoryitem）
- 模型来自 purplegem
- 挂上 psionic 组件，最大值 100、自动充能 rate=1
- 监听 "psionic_full" 事件，触发时调用组件的 Burst() 释放净化效果
- 一些细节：浮在水面、可被影投掷、可被检视
```

> **开发逻辑**：**组件不应该知道自己挂在哪个实体上**——这是组件设计的核心原则。比如 `health` 组件不会写"如果我挂的是猪人，做 X；如果我挂的是兔人，做 Y"。**所有"实体特化的逻辑"都应该放在 Prefab 文件里**——通过监听组件抛出的事件、或者覆盖组件的回调函数来定制。这样一个组件可以被**多种 Prefab 复用**，6.7.8 节我们会把 `psionic` 反过来挂到玩家身上演示这一点。

#### 第三步：跟"灵能短剑"的故事如何衔接？

5.7 节里我们写过这一段：

```5.7.4(伪行号):psionic_shortsword.lua
local function onattack(inst, attacker, target)
    if attacker.components.sanity then
        attacker.components.sanity:DoDelta(-TUNING.PSIONIC_SHORTSWORD_SANITY_COST)
    end
end
```

——攻击时**消耗玩家理智**作为施法代价。

6.7 节我们做出 `psionic` 组件后，可以**自然延伸**这个故事：

> **设想**：在 6.7.8 节末尾我们会展示——给玩家身上也挂一个 `psionic` 组件后，灵能短剑的 `onattack` 可以改成"**优先消耗灵能值，灵能不够时才扣理智**"。这样灵能晶簇就成了战士的"魔力电池"——你打一只蜘蛛吸不到灵能，但你**抱着一颗灵能晶簇出门**，就有 100 点灵能预算可以挥霍。

虽然这个延伸不是 6.7 主线，但**整个故事是连贯的**——5.7 是"消耗"、6.7 是"储存"，下一章我们会进一步讲"事件系统"，让"消耗"和"储存"通过事件无缝对接。

> **新手记忆**：**先想清楚"组件"和"Prefab"的边界**——组件解决"通用的能力"，Prefab 解决"具体的物体"。能力不绑定实体、实体可以叠加多种能力。这是饥荒组件系统设计的核心原则，也是 ECS（Entity-Component-System）思想的精髓。

---

### 6.7.2 快速入门：搭好骨架 —— 五个文件各管什么

#### 第一步：建立 Mod 目录结构

回顾 5.7.1 节，我们已经知道一个完整 Mod 的目录长什么样。这次我们建一个新的：

```
psionic_crystal_mod/
├── modinfo.lua                         ← Mod 元信息
├── modmain.lua                         ← Mod 启动脚本
└── scripts/
    ├── components/
    │   ├── psionic.lua                 ← 我们的组件（核心）
    │   └── psionic_replica.lua         ← Replica（客户端影子）
    └── prefabs/
        └── psionic_crystal.lua         ← Prefab（图纸）
```

**五个文件**，每个职责清晰：

| 文件 | 在 6.x 节里学到的对应概念 | 职责 |
|------|--------------------------|------|
| `modinfo.lua` | 5.6.6 | Mod 名称/版本/兼容性声明，告诉游戏"这是个 Mod" |
| `modmain.lua` | 5.6.7 / 6.2.4 | 注册 Prefab、注册 ReplicableComponent、调整 TUNING |
| `psionic.lua` | 6.1（生命周期）+ 6.5（多源叠加） | 组件主体：状态、行为、生命周期 |
| `psionic_replica.lua` | 6.2（Replica） | 客户端影子：读取同步过来的数据 |
| `psionic_crystal.lua` | 5.2-5.7（Prefab 全套） | 装配体：把组件 + 标签 + 资源拼成具体物品 |

#### 第二步：先把空骨架建起来

很多新手喜欢"一口气写完一个文件再写下一个"——我建议反过来——**先把所有文件的"空骨架"建起来**，让目录结构完整，再逐个填血肉。这样你能**早早地看到 Mod 在游戏里被识别**（虽然功能为零），调试节奏更稳。

**`modinfo.lua`**：

```lua
name = "灵能晶簇"
description = "一颗能自动累积并释放灵能的水晶。教程示例 Mod。"
author = "你的名字"
version = "1.0.0"

api_version = 10

dst_compatible = true
dont_starve_compatible = false
reign_of_giants_compatible = false

all_clients_require_mod = true

icon_atlas = "modicon.xml"
icon = "modicon.tex"

server_filter_tags = {"灵能", "教程"}

configuration_options = {
    {
        name = "MAX_PSIONIC",
        label = "最大灵能值",
        options = {
            {description = "50",  data = 50},
            {description = "100", data = 100},
            {description = "200", data = 200},
        },
        default = 100,
    },
}
```

> **复习要点**：`all_clients_require_mod = true` 表示"客户端必须本地有这个 Mod"——因为我们注册了**新组件 `psionic`**（5.6.8 节坑 6 讲过：注册新组件、新 Prefab 都必须勾选这个）。`api_version = 10` 是 DST 联机版的固定值（5.6.6 节说过）。

**`modmain.lua`**：

```lua
PrefabFiles = {
    "psionic_crystal",
}

Assets = {
}

GLOBAL.TUNING.PSIONIC = {
    DEFAULT_MAX  = GetModConfigData("MAX_PSIONIC") or 100,
    DEFAULT_RATE = 1,                  -- 每秒 1 点
    PERIOD       = 1,                  -- 每秒 tick 一次
    BURST_RANGE  = 8,                  -- Burst 影响范围
    BURST_SANITY = 15,                 -- Burst 给附近玩家恢复的理智
}

AddReplicableComponent("psionic")
```

> **复习要点**：
>
> - `PrefabFiles` 让游戏在启动时加载 `scripts/prefabs/psionic_crystal.lua`（5.6.7 节）
> - `GetModConfigData` 从 `modinfo.lua` 的 `configuration_options` 读配置（5.7.7 节）
> - `AddReplicableComponent("psionic")` 把 `psionic` 加进 `REPLICATABLE_COMPONENTS` 白名单（6.2.4 节）——这一行**一旦遗漏**，客户端就不会有 `inst.replica.psionic`，UI 全部失效

**`scripts/components/psionic.lua`**：

```lua
local Psionic = Class(function(self, inst)
    self.inst = inst
end)

return Psionic
```

**`scripts/components/psionic_replica.lua`**：

```lua
local Psionic = Class(function(self, inst)
    self.inst = inst
end)

return Psionic
```

**`scripts/prefabs/psionic_crystal.lua`**：

```lua
local assets = {
    Asset("ANIM", "anim/purple_gem.zip"),
    Asset("ATLAS", "images/inventoryimages/purplegem.xml"),
}

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    inst.AnimState:SetBank("purple_gem")
    inst.AnimState:SetBuild("purple_gem")
    inst.AnimState:PlayAnimation("idle")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")
    inst.components.inventoryitem.atlasname = "images/inventoryimages/purplegem.xml"

    return inst
end

return Prefab("psionic_crystal", fn, assets)
```

#### 第三步：第一次进游戏验证骨架

把这五个文件丢进 `mods/psionic_crystal_mod/` 文件夹（根据你的 DST 安装位置），主菜单 → Mods → 启用我们的 Mod → 进存档 → 控制台：

```lua
c_give("psionic_crystal")
```

**预期结果**：

- ✅ 你拿到一颗紫色宝石，可以放进背包
- ✅ 控制台**不报错**
- ⚠️ 它什么"灵能"功能都没有——因为 `psionic` 组件还没真的添加到这个 Prefab 上

> **新手记忆**：**先让骨架跑起来，再加血肉**——这是写 Mod 的黄金法则。如果你直接写完整代码再启动，碰到错误你完全不知道是 modinfo 问题、还是 prefab 问题、还是 component 问题。**先空骨架 → 再每加一段就回游戏验证**——这样错误定位永远在 5 行代码之内。

#### 第四步：进阶提示——`AddReplicableComponent` 的两个时机

注意 `modmain.lua` 里的 `AddReplicableComponent("psionic")`——它**必须在游戏加载组件之前调用**。看 `entityreplica.lua` 第 5-26 行的白名单：

```5:26:scripts/entityreplica.lua
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

我们 `AddReplicableComponent("psionic")` 就是把 `psionic = true` 加进这张表。**`modmain.lua` 是顶层执行的脚本**，所有 `Add*` 系列接口都在这里调用一次就行。

> **老手提示**：如果你看到论坛上有人问"我的自定义组件 `inst.replica.xxx` 一直是 `nil`"——99% 是忘了 `AddReplicableComponent`，1% 是 `_replica.lua` 文件名拼错（必须是 `xxx_replica.lua`，看 6.2.7 节）。

---

### 6.7.3 快速入门：组件主体 v1 —— 字段、Setter、DoDelta、自动充能

骨架已经跑起来了。现在我们让 `psionic` 组件**真正能用**——v1 版本只解决三件事：

1. **维护当前/最大灵能值**
2. **每秒自动充能**
3. **数值变化时抛出事件**（让监听者能反应）

存档、同步、休眠这些"工业级"细节我们留到 6.7.4 之后再加。

#### 第一步：先想清楚字段和"事件"

写组件之前，先问自己三个问题：

1. **我有哪些状态字段？** —— `current`（当前值）、`max`（最大值）、`rate`（每秒充能速率）、`task`（定时任务的引用，用来取消）
2. **谁会改这些字段？** —— 只通过 `SetMax` / `SetCurrent` / `DoDelta` 改，**不允许外部直接 `cmp.current = 99`**（避免漏掉同步、漏掉事件）
3. **变化时谁需要知道？** —— Prefab 的代码会监听 `psionic_full` 事件触发 Burst、UI 会监听 `psionicchange` 事件刷新百分比、Replica 会通过 onset 拿到通知

**这就是组件的"对外契约"**——字段是私有的，方法和事件是公开的。这跟 `health.lua`、`fueled.lua` 的设计哲学完全一致。

#### 第二步：完整代码（v1 版）

替换 `scripts/components/psionic.lua`：

```lua
local function oncurrent(self, current)
    if current >= self.max then
        self.inst:AddTag("psionic_full")
    else
        self.inst:RemoveTag("psionic_full")
    end
    if current <= 0 then
        self.inst:AddTag("psionic_empty")
    else
        self.inst:RemoveTag("psionic_empty")
    end
end

local function onmax(self, max)
    if self.current > max then
        self.current = max
    end
    oncurrent(self, self.current)
end

local Psionic = Class(function(self, inst)
    self.inst = inst

    self.max = TUNING.PSIONIC.DEFAULT_MAX
    self.current = 0
    self.rate = TUNING.PSIONIC.DEFAULT_RATE
    self.period = TUNING.PSIONIC.PERIOD

    self.task = nil
end,
nil,
{
    current = oncurrent,
    max = onmax,
})

function Psionic:SetMax(max)
    self.max = max
end

function Psionic:SetCurrent(current)
    self.current = math.clamp(current, 0, self.max)
end

function Psionic:SetRate(rate)
    self.rate = rate
end

function Psionic:GetPercent()
    return self.max > 0 and self.current / self.max or 0
end

function Psionic:GetCurrent() return self.current end
function Psionic:GetMax()     return self.max end
function Psionic:IsFull()     return self.current >= self.max end
function Psionic:IsEmpty()    return self.current <= 0 end

function Psionic:DoDelta(amount, source)
    local oldcurrent = self.current
    self.current = math.clamp(self.current + amount, 0, self.max)
    local delta = self.current - oldcurrent

    if delta == 0 then
        return 0
    end

    self.inst:PushEvent("psionicchange", {
        current = self.current,
        max     = self.max,
        delta   = delta,
        percent = self:GetPercent(),
        source  = source,
    })

    if self.current >= self.max and oldcurrent < self.max then
        self.inst:PushEvent("psionic_full")
    elseif self.current <= 0 and oldcurrent > 0 then
        self.inst:PushEvent("psionic_empty")
    end

    return delta
end

local function OnTick(inst, self)
    self:DoDelta(self.rate * self.period)
end

function Psionic:StartCharging()
    if self.task == nil then
        self.task = self.inst:DoPeriodicTask(self.period, OnTick, nil, self)
    end
end

function Psionic:StopCharging()
    if self.task ~= nil then
        self.task:Cancel()
        self.task = nil
    end
end

function Psionic:Burst()
    if self.current <= 0 then
        return false
    end
    local x, y, z = self.inst.Transform:GetWorldPosition()
    local players = TheSim:FindEntities(x, y, z, TUNING.PSIONIC.BURST_RANGE, {"player"})
    for _, p in ipairs(players) do
        if p.components.sanity then
            p.components.sanity:DoDelta(TUNING.PSIONIC.BURST_SANITY)
        end
    end
    self.inst:PushEvent("psionic_burst", {
        amount = self.current,
        targets = players,
    })
    self:SetCurrent(0)
    return true
end

function Psionic:GetDebugString()
    return string.format("%2.1f / %2.1f (rate=%2.2f/s, %s)",
        self.current, self.max, self.rate, self.task and "ON" or "OFF")
end

return Psionic
```

#### 第三步：逐段拆解——为什么这么写？

**(1) `oncurrent` / `onmax` 这两个 onset 回调**

```lua
{
    current = oncurrent,
    max = onmax,
}
```

回顾 6.1.1 节，`Class` 的第三个参数是 **onset 回调表**——任何对 `self.current` / `self.max` 的赋值都会触发对应的回调。

我们用它做了**两件事**：

- **`oncurrent`**：根据当前值自动加/移除 `"psionic_full"` 和 `"psionic_empty"` 这两个 tag。这样**外部代码可以用 `inst:HasTag("psionic_full")` 来判断状态**——而无需 `inst.components.psionic.current >= max`，**客户端也能用**（tag 是网络同步的，回顾 5.5 节）。
- **`onmax`**：当最大值被改小时，自动把当前值 clamp 下来——避免出现"current > max"的非法状态。

> **开发逻辑**：**onset 回调让"状态"和"派生表现"自动同步**。如果你不用 onset，就要在每个 `SetCurrent` / `DoDelta` 后手动加 `if self.current >= self.max then ... end` ——又啰嗦又容易漏。**这是 Klei 引擎给组件作者的免费工具**，应该用起来。

**(2) `SetCurrent` 用 `math.clamp` 而不是直接赋值**

```lua
function Psionic:SetCurrent(current)
    self.current = math.clamp(current, 0, self.max)
end
```

**永远不要相信调用方传的值是合理的**。如果有人写 `cmp:SetCurrent(99999)` 或 `cmp:SetCurrent(-50)`——`math.clamp` 帮你把它压在合法范围。这是**防御式编程**的体现。

**(3) `DoDelta` 是**修改值的"唯一正门"**

```lua
function Psionic:DoDelta(amount, source)
    local oldcurrent = self.current
    self.current = math.clamp(self.current + amount, 0, self.max)
    local delta = self.current - oldcurrent
    ...
end
```

注意 `delta` 是**实际发生的变化量**——可能小于 `amount`（因为 clamp 截断）。返回这个真实 delta，让调用方知道"我请求加 50，但只加了 30，因为已经满了"。这是 `health.lua` 的 `DoDelta` 同款写法，回顾 6.6.7 节。

**为什么 `PushEvent` 要带 `source`？** —— 让监听者能区分变化来源。比如 UI 可以做"被攻击时屏幕红屏（source 是攻击者）、自然回血时不红屏（source 是 nil）"。

**(4) 三个事件的语义**

```lua
"psionicchange"   -- 任何变化都抛（含 +1、-1）
"psionic_full"    -- 从"未满"到"满了"瞬间抛（不会重复抛）
"psionic_empty"   -- 从"非空"到"空了"瞬间抛（不会重复抛）
```

**`_full` 和 `_empty` 是"边界事件"**——只在跨越边界时抛一次。这避免了"满了之后每 tick 都抛 full 事件"的灾难。看 `health.lua` 的 `death` 事件、`fueled.lua` 的 `onfueldsectionchanged` 事件——都是这种"边界触发"模式。

**(5) `OnTick` 写在外面而不是闭包**

```lua
local function OnTick(inst, self)
    self:DoDelta(self.rate * self.period)
end

function Psionic:StartCharging()
    self.task = self.inst:DoPeriodicTask(self.period, OnTick, nil, self)
end
```

**注意 `OnTick` 是模块级的局部函数**，而不是写成 `self.inst:DoPeriodicTask(self.period, function() self:DoDelta(...) end)`。

为什么？**因为闭包每次 `StartCharging` 都创建一个新函数对象**——内存占用 + GC 压力。**模块级函数只创建一次，被所有实例共享**——这是 Klei 在 `fueled.lua` 第 265-267 行用的同款手法：

```265:267:scripts/components/fueled.lua
local function OnDoUpdate(inst, self, period)
    self:DoUpdate(period)
end
```

> **新手记忆**：**`DoPeriodicTask` 的回调要尽量用模块级函数**——把 `self` 作为参数传进去（`DoPeriodicTask` 第 4 个参数后是任意 user data）。这是写性能敏感组件时的好习惯。

**(6) `Burst` 是业务方法的示范**

```lua
function Psionic:Burst()
    ...
    local players = TheSim:FindEntities(x, y, z, TUNING.PSIONIC.BURST_RANGE, {"player"})
    ...
end
```

`TheSim:FindEntities(x, y, z, radius, must_tags, cant_tags, one_of_tags)` 是 6.5.7 节我们提过的"**范围查找**"API——按位置和标签过滤实体，**性能优于自己遍历 `Ents`**。这里用 `{"player"}` 表示"只找带 player tag 的实体"。

`Burst` 设计成**返回 bool**：成功返回 `true`、灵能不足返回 `false`——让外部调用方知道"是否真的释放了"。这是 `health.lua` 的 `Kill`、`fueled.lua` 的 `TakeFuelItem` 同款返回值习惯。

#### 第四步：把组件挂到 Prefab 上

修改 `scripts/prefabs/psionic_crystal.lua`，在 `inst.entity:SetPristine()` **之后**的服务端段加上：

```lua
inst:AddComponent("psionic")
inst.components.psionic:StartCharging()

inst:ListenForEvent("psionic_full", function(inst)
    inst.components.psionic:Burst()
end)
```

**两段代码各做什么**：

- **`AddComponent("psionic")`**：实例化组件、自动 ReplicateComponent（因为 modmain 已经 `AddReplicableComponent`）、执行所有 `AddComponentPostInit` 注册的回调
- **`StartCharging`**：启动每秒 +1 的定时任务
- **`ListenForEvent("psionic_full", ...)`**：把"满了就 Burst"这个**特化逻辑放在 Prefab 层**——组件本身不知道"满了之后该做什么"，由 Prefab 决定

> **复习 5.4 节的 Pristine 分界线**：`AddComponent` **必须在 `SetPristine` 之后**——因为只有服务端需要持有真实的组件实例，客户端只需要 `replica` 影子。`ListenForEvent` 同理。

#### 第五步：进游戏验证 v1

存档 → `c_give("psionic_crystal")` → 把它扔到地上：

```lua
c_sel().components.psionic:GetDebugString()
-- 输出: 0.0 / 100.0 (rate=1.00/s, ON)

-- 等 5 秒
c_sel().components.psionic:GetDebugString()
-- 输出: 5.0 / 100.0 (rate=1.00/s, ON)

-- 等 100 秒（或者直接快进）
-- 屏幕上：附近玩家恢复 +15 理智
-- 控制台：current 回到 0，重新开始充能

-- 验证 tag
c_sel():HasTag("psionic_full")  -- 在某一刻会 true
c_sel():HasTag("psionic_empty") -- 一直 true 直到第一次 +1
```

✅ **如果你看到这些输出，恭喜——你的第一个自定义组件成功跑起来了！**

#### 第六步：v1 版本的"未完成清单"

但 v1 还有 4 个问题（这正是后面 5 节要解决的）：

| 问题 | 后果 | 解决章节 |
|------|------|---------|
| ❌ 没存档 | 退出再进游戏，灵能值清零 | 6.7.4 |
| ❌ 没休眠 | 玩家走远，task 还在跑（浪费 CPU） | 6.7.5 |
| ❌ 客户端看不到 current/max | UI 没法显示百分比 | 6.7.6 |
| ❌ 充能率没法叠加 | 想做"附近有花充能更快"做不到 | 6.7.7 |

> **新手记忆**：**先做能跑的最小版本，再迭代加功能**——这种"**Walking Skeleton**"开发模式让你**任何时刻代码都能运行**，调试和阅读都更轻松。Klei 的源码很多组件也是这样从最简单版本演化来的，看 `health.lua` 的注释里能找到 V2C（一个 Klei 程序员）几年里加的几十处 bug fix 和功能扩展。

---

### 6.7.4 进阶：让灵能"活下去" —— OnSave / OnLoad

#### 第一步：明确"该存什么、不该存什么"

回顾 6.1.4 节我们总结的 5 条潜规则：

1. 返回 nil 不存
2. 只存"必要的"字段
3. 默认值不存
4. 不存可重新计算的字段
5. 不存函数引用

套用到我们的 `psionic`：

| 字段 | 存？ | 理由 |
|------|-----|------|
| `self.current` | ✅ | 必须保存，否则游戏重启就清零 |
| `self.max` | 看情况 | **大多数时候不存**——Prefab 会重新初始化为 100；但如果有人通过 `cmp:SetMax(200)` 改过最大值，就要存 |
| `self.rate` | ❌ | Prefab 会重新初始化；如果想运行时改变可存（少见） |
| `self.task` | ❌ | 函数引用不能存；OnLoad 时根据 current 状态重启 |
| `self.period` | ❌ | 配置常量，不变 |

#### 第二步：完整代码

在 `psionic.lua` 里加上这两个方法（放在 `DoDelta` 之后）：

```lua
function Psionic:OnSave()
    if self.current <= 0 and self.max == TUNING.PSIONIC.DEFAULT_MAX then
        return nil
    end
    return {
        current = self.current > 0 and self.current or nil,
        max = self.max ~= TUNING.PSIONIC.DEFAULT_MAX and self.max or nil,
        add_component_if_missing = true,
    }
end

function Psionic:OnLoad(data)
    if data == nil then
        return
    end

    if data.max ~= nil then
        self.max = data.max
    end

    if data.current ~= nil then
        self:SetCurrent(data.current)
    end
end
```

#### 第三步：逐句拆解——三个有意思的细节

**(1) "全默认状态时返回 nil"**

```lua
if self.current <= 0 and self.max == TUNING.PSIONIC.DEFAULT_MAX then
    return nil
end
```

如果当前值是 0、最大值是默认 100——**完全可以由 Prefab 重新初始化**，不需要存档。回顾 6.1.4 第 5 条：

> 默认值不存。如 `fueled.lua` 第 133-137 行的 `OnSave` 在燃料满时直接返回 nil。

**好处**：节省存档体积。一颗刚生成的灵能晶簇，存档里**完全没有它的 psionic 数据**。

**(2) `max` 字段的"按需保存"**

```lua
max = self.max ~= TUNING.PSIONIC.DEFAULT_MAX and self.max or nil,
```

只在 `max` **被改过**（不等于默认值）时才保存。这是 Lua 里的"条件赋值"惯用法——`A and B or C` 等效于 `A ? B : C`。

> **进阶提示**：这种写法在 `health.lua` 第 154-161 行可以看到：
>
> ```154:161:scripts/components/health.lua
> function Health:OnSave()
>     return
>     {
>         health = self.currenthealth,
>         penalty = self.penalty > 0 and self.penalty or nil,
> 		maxhealth = self.save_maxhealth and self.maxhealth or nil
>     }
> end
> ```
>
> Klei 用 `penalty > 0 and self.penalty or nil` 让 `penalty == 0` 时不存。我们用相同的手法。

**(3) `add_component_if_missing = true` 的妙用**

这是回顾 6.1.4 第四步里讲过的**特殊标记**：

```lua
add_component_if_missing = true,
```

**作用**：如果某天你**改了 Prefab、把 `inst:AddComponent("psionic")` 删了**——但**老存档里有这颗晶簇带着 psionic 数据**——引擎会**自动给它补上 psionic 组件**，然后 OnLoad 数据。

回顾 `entityscript.lua` 第 1954-1984 行的 `SetPersistData` 流程，那段代码会检测这个标记。

**什么时候用？** —— 当你想**保证存档兼容性**时。比如老版本 Mod 没有 `psionic` 组件、新版本加了——玩家升级 Mod 后，老存档里没有 `psionic` 数据，所以这个标记是**新版本对老版本的兼容**；反过来，如果你哪天要拆掉这个组件，**新版本不再 AddComponent**，但老存档里仍有 psionic 数据——这个标记保证组件被自动补上、数据被读回来。

> **老手提示**：这个标记**不需要每个组件都加**——只对那些"**绝对不能丢数据**"的组件加。比如 `health` 没有这个标记（因为生物总会再 `AddComponent("health")`）；但**自定义组件**最好加上，避免 Mod 升级造成的数据丢失。

**(4) `OnLoad` 的防御式 nil 检查**

```lua
if data == nil then
    return
end

if data.max ~= nil then
    self.max = data.max
end

if data.current ~= nil then
    self:SetCurrent(data.current)
end
```

**每一个字段都要 nil 检查**。看 `perishable.lua` 第 339-351 行的同款写法（6.1.4 节讲过）。原因有二：

- **新版本 Mod 加了字段**：老存档里没有这个字段，直接 `data.newfield` 是 `nil`
- **被截断的存档**：极少数情况下存档可能不完整，nil 检查避免崩溃

**注意 `current` 用 `SetCurrent`，而不是 `self.current = data.current`**：
- `SetCurrent` 会做 clamp（防止存档里有非法值）
- `SetCurrent` 会触发 `oncurrent` onset 回调（自动加 tag）

**这是组件作者的"自我保护"——尽量走自己设计的"正门"，避免直接改字段**。

#### 第四步：注意 `OnLoad` 之后要不要 StartCharging？

你可能会问：**`OnLoad` 完之后我们应该 `StartCharging` 吗？**

答案是**不需要**——因为 `psionic_crystal.lua` 的工厂函数里**已经调用了** `StartCharging`：

```lua
inst:AddComponent("psionic")
inst.components.psionic:StartCharging()    -- 工厂函数里已经启动了任务
```

实体生成的流程是：

```
SpawnPrefab("psionic_crystal")
    ├── 执行 fn() 工厂函数
    │     ├── AddComponent("psionic")
    │     └── StartCharging   ← 任务已启动
    └── 引擎 SetPersistData(data)
          └── psionic:OnLoad(data)   ← 此时再恢复 current 值
```

**任务在 OnLoad 之前就已经启动了**——OnLoad 只负责把"血量"补回来。这是一种**关注点分离**：定时任务由 Prefab 启动、数据由组件自管。

> **陷阱预警**：**不要在 `OnLoad` 里 `StartCharging`**——会和 Prefab 的工厂函数重复启动（虽然 `StartCharging` 内部有 `if self.task == nil` 防御，但语义上不清晰）。**让 Prefab 控制启动/停止时机，让组件管理数据状态**。

#### 第五步：进游戏验证存档

```
1. 进游戏，c_give("psionic_crystal")，等到 current 大约 30
2. 控制台：c_sel().components.psionic:GetCurrent() → 30
3. 退出游戏（让世界存档）
4. 重新进游戏
5. c_sel().components.psionic:GetCurrent() → 30  ✅ 数据保住了
```

✅ **验证通过**——OnSave/OnLoad 接好了。

> **新手记忆**：**只要你的组件有"运行时可变"的状态字段，就要写 OnSave/OnLoad**。规则三句话：① 默认值不存、② nil 检查防御、③ 用 setter 而不是直接赋值。

---

### 6.7.5 进阶：性能与时间补偿 —— OnEntitySleep / OnEntityWake

#### 第一步：先回顾 6.1.5 的判断标准

回顾 6.1.5 节末尾我们总结的判断原则：

> - **有定时任务 / 心跳回调的组件** → **需要** `OnEntitySleep/Wake`
> - **有"时间流逝自动变化"状态的组件** → **需要**（而且可能需要补偿）
> - **纯被动、事件驱动的组件** → **不需要**

我们的 `psionic` **同时满足前两条**——它有一个 `task` 在每秒 tick，还有"时间一过 current 就增长"的隐含状态。所以**必须实现 sleep/wake**，而且要带**时间补偿**。

#### 第二步：完整代码

在 `psionic.lua` 的 `StopCharging` 之后加上：

```lua
function Psionic:OnEntitySleep()
    self:StopCharging()
    self._sleeptime = GetTime()
end

function Psionic:OnEntityWake()
    if self._sleeptime ~= nil then
        local elapsed = GetTime() - self._sleeptime
        self._sleeptime = nil

        if elapsed > 0 then
            self:LongUpdate(elapsed)
        end
    end

    if self.task == nil and not self:IsFull() then
        self:StartCharging()
    end
end

function Psionic:LongUpdate(dt)
    if self.rate > 0 and dt > 0 then
        self:DoDelta(self.rate * dt)
    end
end
```

并且在 `OnRemoveFromEntity`（如果还没有就加上）里加清理：

```lua
function Psionic:OnRemoveFromEntity()
    self:StopCharging()
    self.inst:RemoveTag("psionic_full")
    self.inst:RemoveTag("psionic_empty")
    self._sleeptime = nil
end
```

#### 第三步：逐句拆解

**(1) `OnEntitySleep`：停 task + 记时间戳**

```lua
self:StopCharging()
self._sleeptime = GetTime()
```

两件事：

- **停 task**：玩家走远了，没必要继续 tick——这是性能优化的核心。看 `combat.lua` 第 284-289 行的同款写法（6.1.5 节讲过）。
- **记时间戳**：用 `GetTime()` 拿到当前游戏时间（不是真实时间！），存到一个**带下划线前缀的临时字段** `self._sleeptime`——下划线前缀是 Klei 的命名约定，表示"内部状态、不要从外部访问"。

> **进阶提示**：`GetTime()` 返回的是**游戏内累计时间（秒）**——不受暂停影响、不受真实时间影响。看 `inventoryitemmoisture.lua` 第 91-113 行的同款写法（6.1.5 节讲过）。

**(2) `OnEntityWake`：补偿 + 重启**

```lua
if self._sleeptime ~= nil then
    local elapsed = GetTime() - self._sleeptime
    self._sleeptime = nil
    if elapsed > 0 then
        self:LongUpdate(elapsed)
    end
end
```

**核心思路**：

- 算出休眠时长 `elapsed`
- 调 `LongUpdate(elapsed)`——一次性把"应该发生的变化"补上
- **清空 `_sleeptime`**（避免下次 wake 时被错误使用）

**为什么要单独抽 `LongUpdate` 方法？** —— Klei 的命名约定：**`LongUpdate(dt)` 是"一次性追赶时间"的标准接口**。除了 sleep/wake，**控制台命令 `c_longupdate(86400)` 也会调用每个组件的 `LongUpdate`**——比如让玩家"快进 24 小时"看看食物变质、植物生长效果。

看 `fueled.lua` 第 351 行的写法：

```351:351:scripts/components/fueled.lua
Fueled.LongUpdate = Fueled.DoUpdate
```

——**直接把 `LongUpdate` 别名为 `DoUpdate`**！为什么我们不这么写？因为我们的 `DoUpdate` 是模块级局部函数 `OnTick`，不是 method，没法直接别名。所以单独写一个 `LongUpdate(self, dt)` 更清晰。

**(3) `OnEntityWake` 末尾的"重启 task"**

```lua
if self.task == nil and not self:IsFull() then
    self:StartCharging()
end
```

**两个判断**：

- `self.task == nil` —— 防御式：万一 sleep 时 stop 失败、或者 sleep 时 task 已经 nil
- `not self:IsFull()` —— 满了就别重启了（`StartCharging` 不会自动检查满状态）

> **设计哲学**：**OnEntityWake 应该让组件"恢复到正常运行状态"**——但"正常运行"的定义可能依赖当前数据。如果 `LongUpdate` 已经把 current 充到满了，task 就不该再启动（不过即使启动了，下一个 tick 看到 IsFull 也不会真的 +1，因为 DoDelta 有 clamp）。**多一层判断让代码意图更明确**。

**(4) `OnRemoveFromEntity` 的清理三件套**

```lua
self:StopCharging()
self.inst:RemoveTag("psionic_full")
self.inst:RemoveTag("psionic_empty")
self._sleeptime = nil
```

回顾 6.1.6 节我们说过 `OnRemoveFromEntity` 的核心原则：**"谁污染谁治理"**——加了 task 就停 task、加了 tag 就移 tag、加了字段就置 nil。

注意 **`psionic_full` / `psionic_empty` 这两个 tag 是 onset 回调加上去的**，外部代码可能还在用 `inst:HasTag("psionic_full")` 做判断——如果不清理，就成了"幽灵 tag"（6.1.6 节讲过这个坑）。

#### 第四步：进游戏验证休眠补偿

最难验证的就是这个——因为你需要让实体真正进入"sleep"状态。在 DST 里实体距离玩家**一定距离**之外才会休眠（默认大约 35 个 tile）。

**测试步骤**：

```
1. c_give("psionic_crystal")，扔到地上
2. 控制台：c_sel().components.psionic:GetCurrent() → 假设为 5
3. 用 c_setseason("autumn") + c_setphase("day") 减少干扰
4. 玩家走远到 50 tile 之外，等 60 秒
5. 走回来，立刻：c_sel().components.psionic:GetCurrent()
   → 应该是 5 + 60 = 65 ✅（如果没补偿就还是 5+1=6）
```

**或者用 c_longupdate 测试 LongUpdate**：

```lua
-- 不需要走远，直接快进
c_longupdate(60)
-- 控制台输出会调用所有组件的 LongUpdate，包括我们的
-- 验证：c_sel().components.psionic:GetCurrent() 应该 +60
```

#### 第五步：性能数字感

**没做休眠时**：每颗灵能晶簇每秒 tick 一次，地图上有 50 颗 = 每秒 50 次回调
**做了休眠后**：玩家附近大约 5-10 颗在 tick，远处 40 颗都暂停 = 每秒 5-10 次回调

**虽然单次 tick 很轻**（只是 `current += rate`），但游戏里**这种"小定时任务"加起来非常可观**——尤其是 perishable、fueled、growable、grower、burnable 等组件，每个实体可能挂 3-5 个有 task 的组件。**做休眠优化能把"远处实体"的 CPU 占用降到接近零**。

> **新手记忆**：**只要你写了 `DoPeriodicTask`，就要写 `OnEntitySleep` 取消它**——这是组件作者的基本职业道德。哪怕你的组件再轻量，没人能保证它不会被"刷怪 mod"加到几百个实体上。

---

### 6.7.6 进阶：写一个轻型 Replica —— 让客户端也能看见灵能

#### 第一步：明确"客户端为什么需要看到"

如果只有服务端能读 `current` / `max` —— 谁来画 UI？

- **HUD 上的灵能进度条**：客户端绘制
- **拖拽到背包时鼠标提示"剩余 30/100 灵能"**：客户端渲染
- **第三方 UI Mod 想读取这个值显示**：客户端访问

回顾 6.2.1 节的核心结论：

> **服务端持有"真"组件、客户端持有"影子"replica，引擎自动同步两边的状态**。

所以我们要写 `psionic_replica.lua`——客户端版本的 `psionic`。

#### 第二步：选哪种 Replica 方案？

回顾 6.2 节我们看过两种方案：

| 方案 | 代价 | 适用场景 |
|------|------|---------|
| **A. classified 实体方案** | 多写 `psionic_classified.lua`、SpawnPrefab 一个隐形实体跟随主实体 | 字段多、需要 RPC 调用、玩家专属（如 health/sanity/inventory） |
| **B. 轻型 net_xxx 方案** | 在 `_replica.lua` 里直接 `net_smallbyte(inst.GUID, ...)` 挂到 inst 上 | 字段少、单向同步、非玩家实体（如 inventoryitem 的小字段） |

我们的 `psionic` **只有 3 个字段需要同步**（current / max / rate），而且**主要是单向**（服务端 → 客户端，客户端不需要主动设置）——**B 方案最合适**。

> **进阶提示**：B 方案的代价是"**没法做太多优化**"——每个 net 变量在网络层都是独立的字段。如果你有 30 个字段要同步，每个都用 net_xxx 会让网络包变大。这时候就该用 classified 把它们打包成一个实体。`psionic` 才 3 个字段，B 完全够用。

#### 第三步：先了解 `net_smallbyte` 等 net 变量

回顾 6.3.3 节的 net 变量基础类型：

| 类型 | 容量 | 适用 |
|------|------|------|
| `net_bool` | 1 bit | 二值开关 |
| `net_smallbyte` | 6 bits = 0-63 | 小整数 |
| `net_byte` | 8 bits = 0-255 | 中等整数 |
| `net_ushortint` | 16 bits = 0-65535 | 大整数 |
| `net_int` | 32 bits = -2^31 ~ 2^31-1 | 通用整数 |
| `net_float` | 32 bits | 浮点 |
| `net_string` | 变长 | 字符串 |
| `net_hash` | 32 bits | 字符串哈希（节省带宽） |

我们的灵能值范围 0-100（默认）—— `net_byte` 完全够（如果未来想支持上千就换 `net_ushortint`）。`rate` 是浮点，但我们可以**乘 100 转成整数**传递（节省带宽，6.3.3 节讲过这个 trick）。

> **设计权衡**：**用 `net_byte` 还是 `net_ushortint`？** —— 取决于你预期的最大值。如果你知道未来某个 Mod 会改 `cmp:SetMax(500)`，就用 `net_ushortint`。这里我们用 `net_byte`，假设没人会用超过 255。**实际工程里，宁可多用 8 bit 也别反复改类型**——网络变量类型一旦发布、玩家存档建立后，再改类型可能导致兼容性灾难。

#### 第四步：完整代码

替换 `scripts/components/psionic_replica.lua`：

```lua
local Psionic = Class(function(self, inst)
    self.inst = inst

    self._current = net_ushortint(inst.GUID, "psionic._current", "psionicchangedirty")
    self._max     = net_ushortint(inst.GUID, "psionic._max",     "psionicchangedirty")

    if not TheWorld.ismastersim then
        self._sub = inst:ListenForEvent("psionicchangedirty", function(inst)
            self.inst:PushEvent("psionicchange", {
                current = self._current:value(),
                max     = self._max:value(),
                percent = self._max:value() > 0 and self._current:value() / self._max:value() or 0,
            })
        end)
    end
end)

function Psionic:SetCurrent(current)
    self._current:set(math.floor(current + 0.5))
end

function Psionic:SetMax(max)
    self._max:set(math.floor(max + 0.5))
end

function Psionic:GetCurrent()
    if self.inst.components.psionic ~= nil then
        return self.inst.components.psionic.current
    else
        return self._current:value()
    end
end

function Psionic:GetMax()
    if self.inst.components.psionic ~= nil then
        return self.inst.components.psionic.max
    else
        return self._max:value()
    end
end

function Psionic:GetPercent()
    local max = self:GetMax()
    return max > 0 and self:GetCurrent() / max or 0
end

function Psionic:IsFull()  return self:GetCurrent() >= self:GetMax() end
function Psionic:IsEmpty() return self:GetCurrent() <= 0 end

return Psionic
```

#### 第五步：逐段拆解

**(1) net 变量声明位置**

```lua
self._current = net_ushortint(inst.GUID, "psionic._current", "psionicchangedirty")
self._max     = net_ushortint(inst.GUID, "psionic._max",     "psionicchangedirty")
```

**注意 net 变量必须在两端都声明**——也就是 **Replica 的构造函数里两端都跑**。这就是回顾 6.3.6 节为什么 net 变量要写在 pristine 之前的原因。

**三个参数**：

- **inst.GUID**：归属哪个实体
- **`"psionic._current"`**：唯一变量名（用 `组件名._字段名` 命名约定避免冲突）
- **`"psionicchangedirty"`**：dirty event 名——值变化时 PushEvent 这个事件

**两个变量共用一个 dirty event** 是惯用法——只要任何一个变了 UI 就会刷新。如果分开监听就需要两个回调，反而更麻烦。

**(2) 客户端监听 dirty event 转抛业务事件**

```lua
if not TheWorld.ismastersim then
    self._sub = inst:ListenForEvent("psionicchangedirty", function(inst)
        self.inst:PushEvent("psionicchange", { ... })
    end)
end
```

**只在客户端监听** —— 因为服务端的 `psionic` 组件已经在 `DoDelta` 里 PushEvent 过 `psionicchange`。客户端没有真组件，需要在收到 net 变量更新时**模拟一下**这个事件。

**这样 UI 代码就可以这么写**（不区分 client/server）：

```lua
inst:ListenForEvent("psionicchange", function(inst, data)
    local pct = data.percent
    badge:SetPercent(pct)
end)
```

——**两端都能用同一份监听代码**！这是 Replica 系统的精髓。

**(3) `Set` 方法只在服务端调用**

```lua
function Psionic:SetCurrent(current)
    self._current:set(math.floor(current + 0.5))
end
```

`net_xxx:set(value)` 是写入 net 变量。**只有服务端有权限改**——客户端调用会被引擎忽略（甚至报警告）。

**`math.floor(current + 0.5)` 是四舍五入**——因为 `net_ushortint` 只能存整数，但 `current` 在服务端可能是 65.7 这种浮点。如果直接 `set(65.7)`，引擎会做 `math.floor` 截断成 65。**主动四舍五入更精确**。

**(4) `Get` 方法的"双向兼容"**

```lua
function Psionic:GetCurrent()
    if self.inst.components.psionic ~= nil then
        return self.inst.components.psionic.current
    else
        return self._current:value()
    end
end
```

**这是 Replica 的标准模式**：

- 服务端：`inst.components.psionic` 存在 → 直接读真组件，**精确到浮点**
- 客户端：没有真组件 → 读 net 变量，**取整数版本**

看 `health_replica.lua` 第 90-98 行的 `GetPercent` 是同款写法（6.2.6 节讲过）。

**为什么服务端不直接读 net 变量？** —— 网络变量经过四舍五入有精度损失（`current = 65.7` 在 net 里是 66）。服务端有真值就用真值，更精确。

#### 第六步：让服务端组件主动同步给 Replica

光有 `_replica.lua` 还不够——服务端的 `psionic` 组件**得主动调用 Replica 的 setter**，否则 net 变量永远是 0。

修改 `psionic.lua` 的 `oncurrent` 和 `onmax`：

```lua
local function oncurrent(self, current)
    if current >= self.max then
        self.inst:AddTag("psionic_full")
    else
        self.inst:RemoveTag("psionic_full")
    end
    if current <= 0 then
        self.inst:AddTag("psionic_empty")
    else
        self.inst:RemoveTag("psionic_empty")
    end

    if self.inst.replica.psionic ~= nil then
        self.inst.replica.psionic:SetCurrent(current)
    end
end

local function onmax(self, max)
    if self.current > max then
        self.current = max
    end
    oncurrent(self, self.current)

    if self.inst.replica.psionic ~= nil then
        self.inst.replica.psionic:SetMax(max)
    end
end
```

**关键改动**：onset 回调最后调用 `replica:SetCurrent` / `:SetMax` —— **每次字段变化都同步给客户端**。

**为什么用 onset 而不是在 `DoDelta` 末尾？** —— 因为：

- `DoDelta` 不是改 current 的唯一入口（`SetCurrent`、`OnLoad` 也会改）
- onset 是**所有路径的汇合点** —— 写一次永远不漏

> **进阶提示**：看 `health.lua` 第 9-16 行的 `onmaxhealth`：
>
> ```9:16:scripts/components/health.lua
> local function onmaxhealth(self, maxhealth)
>     self.inst.replica.health:SetMax(maxhealth)
>     ...
> end
> ```
>
> Klei 用的就是这种 onset 同步模式——和我们一样。

#### 第七步：还需要在 Prefab 里"拉响"初始同步吗？

**通常不用**。因为 `oncurrent` / `onmax` 在**构造函数 `self.max = ...` 时就会触发**——构造函数里 `self.max = TUNING.PSIONIC.DEFAULT_MAX` 这行赋值会立刻调 `onmax`，进而调 `replica:SetMax(100)`——所以客户端从一开始就拿到 100。

**但有一种情况要注意**：如果 `inst.replica.psionic` **在构造函数执行时还没创建好**——onset 里的 `if self.inst.replica.psionic ~= nil` 检查会跳过。这时候同步就没发生，客户端拿到的是 net 变量默认值 0。

**解决方案**：在 Prefab 工厂函数里 `AddComponent("psionic")` 之后，手动 sync 一次：

```lua
inst:AddComponent("psionic")
inst.components.psionic:StartCharging()

if inst.replica.psionic ~= nil then
    inst.replica.psionic:SetMax(inst.components.psionic.max)
    inst.replica.psionic:SetCurrent(inst.components.psionic.current)
end
```

**实际上**，回看 `entityscript.lua` 第 624-633 行的 `AddComponent`：

```624:633:scripts/entityscript.lua
self:ReplicateComponent(name)

local loadedcmp = cmp(self)
self.components[name] = loadedcmp
self.lower_components_shadow[lower_name] = true
```

**`ReplicateComponent` 在 `cmp(self)` **之前**调用**——也就是说**先创建 Replica，再创建真组件**。所以构造函数里 `self.inst.replica.psionic` 已经存在，onset 同步是有效的。

**这两段代码（手动 sync）虽然不是必须，但是"防御式"写法**——避免某些边角情况下的同步漏。建议加上。

#### 第八步：进游戏验证 Replica

```
1. 在专用服务器上测试（或者主机模式）
2. 客户端进游戏
3. 等晶簇充能到 50 左右
4. 客户端控制台：c_sel().replica.psionic:GetPercent()
   → 应该返回 0.5 ✅
5. 客户端：c_sel().components.psionic
   → 应该是 nil（客户端没有真组件）✅
6. 客户端再换一个：c_sel().replica.psionic.GetCurrent
   → 应该是函数（不是 nil） ✅
```

> **新手记忆**：**Replica 的核心心法**：服务端有真值，replica 是影子。**onset → replica:SetXxx** 是 Klei 的标准同步模式。所有读字段的代码都通过 `inst.replica.psionic:GetXxx()`，**不要直接访问 `replica.psionic._current:value()`**——这违反了封装。

---

### 6.7.7 老手进阶：充能率多来源叠加 —— SourceModifierList 实战

#### 第一步：先想清楚为什么需要"多来源"

我们当前的 `rate` 是一个**裸字段**：

```lua
self.rate = 1
```

**问题**来了：

- 如果想做"**附近有罂粟花，充能率 ×1.5**"——你要 `cmp:SetRate(1.5)`
- 然后玩家又**穿了一件法袍 ×1.2**——你要 `cmp:SetRate(1.5 * 1.2 = 1.8)`
- 然后罂粟花被踩死了——你要 `cmp:SetRate(1.0 * 1.2 = 1.2)`
- ......

**每次叠加 / 解除一个 buff，都要重新算总 rate**。代码到处是 `1.5 * 1.2 / 1.5 * 0.8`，一个 bug 就把基准值搞乱。

这就是回顾 6.5.1 节我们讲的"**多来源叠加问题**"——不应该让外部调用方算总和，**应该让组件自己管理一张"修正源表"**。

回顾 6.5 节的 `SourceModifierList` API：

```lua
self.rate_modifiers = SourceModifierList(self.inst)
self.rate_modifiers:SetModifier(source, multiplier, key)
self.rate_modifiers:RemoveModifier(source, key)
local total = self.rate_modifiers:Get()  -- 默认乘法叠加
```

**关键好处**：

- 每个 source 独立——一个 source 来了 SetModifier、走了 RemoveModifier，**不需要知道其他 source 的存在**
- 如果 source 是**实体引用**，实体被销毁时自动从 modifier 列表里移除（自动清理）
- 一个 source 可以挂多个 key 的 modifier（比如同一个法袍既给 +1.5 充能率又给 +20 灵能上限）

#### 第二步：改造 `psionic.lua` 加上 modifier 支持

在 `psionic.lua` 顶部加 `require`：

```lua
local SourceModifierList = require("util/sourcemodifierlist")
```

修改构造函数：

```lua
local Psionic = Class(function(self, inst)
    self.inst = inst

    self.max = TUNING.PSIONIC.DEFAULT_MAX
    self.current = 0
    self.rate = TUNING.PSIONIC.DEFAULT_RATE
    self.rate_modifiers = SourceModifierList(self.inst, 1, SourceModifierList.multiply)
    self.period = TUNING.PSIONIC.PERIOD

    self.task = nil
end,
nil,
{
    current = oncurrent,
    max = onmax,
})
```

**注意构造函数第二、三参数**：

- 第一个 `1` —— 默认值。`SourceModifierList:Get()` 在没有任何 source 时返回 1（中性的乘法元素）。如果是加法叠加就传 `0`。
- 第二个 `SourceModifierList.multiply` —— 叠加方式。**乘法叠加**：所有 source 的值连乘起来。

回顾 6.5.3 节：

| 叠加方式 | 用途 | 默认值 |
|---------|------|------|
| `SourceModifierList.multiply` | 乘法（速率倍率、伤害倍率） | 1 |
| `SourceModifierList.additive` | 加法（生命上限加成、属性加成） | 0 |
| `SourceModifierList.highest` | 取最大（多源伤害减免，取最强的） | 0 |

我们的 `rate` 用乘法叠加：`实际 rate = 基础 rate × 所有 modifier 之积`。

**修改 `OnTick` 用 modifier 之后的实际值**：

```lua
local function OnTick(inst, self)
    self:DoDelta(self.rate * self.rate_modifiers:Get() * self.period)
end
```

**修改 `LongUpdate` 同步**：

```lua
function Psionic:LongUpdate(dt)
    local effective_rate = self.rate * self.rate_modifiers:Get()
    if effective_rate ~= 0 and dt > 0 then
        self:DoDelta(effective_rate * dt)
    end
end
```

**新增两个对外接口**：

```lua
function Psionic:AddRateModifier(source, mult, key)
    self.rate_modifiers:SetModifier(source, mult, key)
end

function Psionic:RemoveRateModifier(source, key)
    self.rate_modifiers:RemoveModifier(source, key)
end
```

**改进 `GetDebugString` 显示更多信息**：

```lua
function Psionic:GetDebugString()
    return string.format("%2.1f / %2.1f (rate=%2.2f x%2.2f = %2.2f/s, %s)",
        self.current, self.max,
        self.rate, self.rate_modifiers:Get(), self.rate * self.rate_modifiers:Get(),
        self.task and "ON" or "OFF")
end
```

#### 第三步：在 Prefab 里"侦测周围花卉"自动加 modifier

修改 `psionic_crystal.lua` 的工厂函数。先在文件顶部加几个常量：

```lua
local SCAN_RANGE = 6
local SCAN_PERIOD = 3
local POPPY_BONUS = 1.5
local FLOWER_EVIL_PENALTY = 0.5
```

然后增加一个扫描函数（写在 `fn` 之外）：

```lua
local function ScanFlowers(inst)
    local x, y, z = inst.Transform:GetWorldPosition()
    local poppy = TheSim:FindEntities(x, y, z, SCAN_RANGE, {"flower"}, {"flower_evil", "INLIMBO"})
    local evil  = TheSim:FindEntities(x, y, z, SCAN_RANGE, {"flower_evil"}, {"INLIMBO"})

    local cmp = inst.components.psionic
    if cmp == nil then return end

    if #poppy > 0 then
        cmp:AddRateModifier(inst, POPPY_BONUS, "poppy_aura")
    else
        cmp:RemoveRateModifier(inst, "poppy_aura")
    end

    if #evil > 0 then
        cmp:AddRateModifier(inst, FLOWER_EVIL_PENALTY, "evil_aura")
    else
        cmp:RemoveRateModifier(inst, "evil_aura")
    end
end
```

最后在工厂函数的服务端段加上启动扫描：

```lua
inst:AddComponent("psionic")
inst.components.psionic:StartCharging()

inst:DoPeriodicTask(SCAN_PERIOD, ScanFlowers, math.random() * SCAN_PERIOD)

inst:ListenForEvent("psionic_full", function(inst)
    inst.components.psionic:Burst()
end)
```

#### 第四步：逐段拆解

**(1) 为什么 source 用 `inst`（晶簇自己）而不是花？**

这里有个**重要设计决策**——`AddRateModifier(inst, ...)` 用**晶簇自己**作为 source，而不是"附近的某朵花"。

**对比两种思路**：

| 思路 | 优点 | 缺点 |
|------|------|------|
| **A. 把每朵花作为 source** | 自动清理（花死了 modifier 自动移除） | 多朵花就有多个 modifier，叠加效果失控（5 朵花 ×1.5^5 = 7.6 倍） |
| **B. 用晶簇自己作为 source，用 key 区分类型** | 同类型只有一个 modifier，可控 | 需要主动定时扫描、主动 Remove |

**我们用的是 B**——更可控。如果想做"花越多倍率越高"，可以在 ScanFlowers 里：

```lua
cmp:AddRateModifier(inst, 1 + 0.1 * math.min(#poppy, 5), "poppy_aura")
```

——**最多 5 朵生效，每多一朵 +10%**。这种"上限保护"防止失控。

> **进阶提示**：回顾 6.5.4 节我们讨论 source 的设计，**实体引用 vs 字符串标识**——这里我们其实可以用字符串 source（`"poppy"`、`"evil"`）：
>
> ```lua
> cmp:AddRateModifier("poppy", POPPY_BONUS)
> cmp:AddRateModifier("evil", FLOWER_EVIL_PENALTY)
> ```
>
> 但**字符串 source 没有 onremove 自动清理**——晶簇被销毁时，modifier 会驻留（虽然组件也会被销毁所以无所谓，但语义上不清晰）。**用 `inst` 自己作为 source**：晶簇被销毁时 `OnRemove` 自动触发 modifier 清理（看 `SourceModifierList` 第 38-43 行的实现）。

**(2) 为什么 `DoPeriodicTask` 要 randomize 起始延迟？**

```lua
inst:DoPeriodicTask(SCAN_PERIOD, ScanFlowers, math.random() * SCAN_PERIOD)
```

**第 3 个参数是首次延迟**。如果不 random，所有晶簇都会在**同一帧**做扫描——CPU 出现规律性峰值。`math.random() * SCAN_PERIOD` 让每颗晶簇错开起跑时间，**CPU 负载平摊**。

看 `inventoryitemmoisture.lua` 第 110 行的同款手法：

```110:110:scripts/components/inventoryitemmoisture.lua
self.moistureupdatetask = self.inst:DoPeriodicTask(updated and UPDATE_TIME or SLOW_UPDATE_TIME, DoUpdate, math.random() * UPDATE_TIME)
```

**这是性能优化的"经典手法"** —— 哪怕你的回调每次只跑 0.01ms，**100 个实体同时跑**也是 1ms 的瞬时峰值。错开起跑就能完全消除这种瞬时峰值。

**(3) `FindEntities` 的过滤参数**

```lua
TheSim:FindEntities(x, y, z, SCAN_RANGE, {"flower"}, {"flower_evil", "INLIMBO"})
```

**5 个参数**：

- `x, y, z`：搜索中心
- `SCAN_RANGE`：半径
- `{"flower"}`：**must_have_one** —— 必须**至少有一个**这些 tag
- `{"flower_evil", "INLIMBO"}`：**cant_have** —— **任何一个**这些 tag 都排除

**INLIMBO 是关键**——回顾 5.5 节，`INLIMBO` 表示"实体在 limbo 状态"（被装进容器、被装进背包后）。**搜索时必须排除 limbo**——否则你会把附近某个玩家背包里的花也当成"周围"。

**两个 tag 互斥**——`flower` 和 `flower_evil` 是分开的两类 tag，所以"普通花"用 `{"flower"}` + 排除 `{"flower_evil"}`；"恶魔花"用 `{"flower_evil"}`。

**(4) Modifier 用 `key` 区分类型**

```lua
cmp:AddRateModifier(inst, POPPY_BONUS, "poppy_aura")
cmp:AddRateModifier(inst, FLOWER_EVIL_PENALTY, "evil_aura")
```

**同一个 source（晶簇本身）下挂多个 key**——这是 6.5.5 节讲的"key 多重 modifier"机制。每个 key 是独立的 modifier，可以独立 Remove。

**最终总倍率 = `1 × poppy_aura × evil_aura`**——比如同时附近有罂粟花和恶魔花：`1 × 1.5 × 0.5 = 0.75`。

#### 第五步：进游戏验证多源叠加

```
1. c_give("psionic_crystal")，扔到地上
2. c_sel().components.psionic:GetDebugString()
   → "5.0 / 100.0 (rate=1.00 x1.00 = 1.00/s, ON)"
3. c_spawn("flower")，让花落在晶簇 6 tile 范围内，等 3 秒
4. c_sel().components.psionic:GetDebugString()
   → "8.0 / 100.0 (rate=1.00 x1.50 = 1.50/s, ON)"  ✅
5. c_spawn("flower_evil") 同样附近
6. c_sel().components.psionic:GetDebugString()
   → "10.0 / 100.0 (rate=1.00 x0.75 = 0.75/s, ON)"  ✅ (1.5 * 0.5)
7. 把恶魔花 c_select()/Pick 移走，等 3 秒
8. → "12.0 / 100.0 (rate=1.00 x1.50 = 1.50/s, ON)"  ✅
```

> **老手记忆**：**SourceModifierList 是组件作者的瑞士军刀**——但凡你的字段会被多个 buff 同时改，就用它。`health.regen_modifiers`、`fueled.rate_modifiers`、`combat.damagemodifiers`、`hunger.hungerrate_modifiers`——Klei 的核心组件几乎全部用了它。**学会它，你的组件就有了"工业级"的可扩展性**。

---

### 6.7.8 老手进阶：调试技巧、五个常见陷阱与扩展方向

#### 第一步：实战调试技巧

**(1) 用 `GetDebugString` 实时看状态**

我们已经在组件里写了 `GetDebugString`。在游戏里：

```lua
c_select(c_findnext("psionic_crystal"))   -- 选中最近的晶簇
-- 然后在游戏里按 Ctrl+Tab，左上角会持续显示 GetDebugString 的输出
```

**这是 Klei 内置的 "DebugEntity" 功能**——只要你的组件实现了 `GetDebugString`，按 Ctrl+Tab 就能看到。`fueled`、`health`、`combat` 都实现了，看着实时变化的数字调试 BUG **比 print 强 10 倍**。

**(2) 用 c_select + 字段访问探索状态**

```lua
local c = c_select(c_findnext("psionic_crystal"))

-- 看完整字段
for k, v in pairs(c.components.psionic) do
    print(k, "=", type(v) == "table" and "<table>" or tostring(v))
end

-- 看 rate_modifiers 内部
for src, kv in pairs(c.components.psionic.rate_modifiers.modifiers) do
    print("source", src)
    if type(kv) == "table" then
        for k, v in pairs(kv) do print("  ", k, "=", v) end
    end
end
```

**(3) 用临时 hook 加 print 抓事件流**

```lua
local p = c_sel().components.psionic
local old_DoDelta = p.DoDelta
p.DoDelta = function(self, amount, source)
    print(string.format("[psionic] DoDelta(%s, %s) src=%s",
        tostring(amount), tostring(self.current), tostring(source)))
    return old_DoDelta(self, amount, source)
end
```

——**回顾 6.6.6 节的实时探索手法**。打完 print 想恢复就 `p.DoDelta = old_DoDelta`，或者直接退游戏（变量是临时 hook）。

**(4) 看完整生命周期**

把所有钩子 hook 一遍：

```lua
local p = c_sel().components.psionic
for _, name in ipairs({"OnSave", "OnLoad", "OnEntitySleep", "OnEntityWake", "OnRemoveFromEntity"}) do
    local old = p[name]
    p[name] = function(self, ...)
        print("[psionic] "..name.." called")
        if old then return old(self, ...) end
    end
end

-- 然后你 c_save() / 走远 / 走近 / c_remove()，控制台会打印每个钩子的触发时机
```

**这是验证 6.7.5 节休眠补偿是否真的工作的最快方法**。

#### 第二步：五个最容易踩的陷阱

**陷阱 1：`AddReplicableComponent` 写错位置**

```lua
-- ❌ 错：写在某个事件回调里
AddPrefabPostInit("world", function(inst)
    AddReplicableComponent("psionic")
end)

-- ✅ 对：直接写在 modmain.lua 顶层
AddReplicableComponent("psionic")
```

**原因**：`AddReplicableComponent` 改的是全局 `REPLICATABLE_COMPONENTS` 表——这张表在**实体被首次创建时**就会被读取（看 `entityreplica.lua` 第 35-37 行的 `if not REPLICATABLE_COMPONENTS[name] then return end`）。如果你写在 `AddPrefabPostInit` 里，注册时机太晚，**第一个被生成的灵能晶簇 replica 失败，后面的才行**——这种"半死不活"的 bug 极难调试。

**陷阱 2：构造函数赋值触发 onset，但 replica 还没建好**

```lua
-- ❌ 错：构造函数里直接用 onset 同步 replica
local function oncurrent(self, current)
    self.inst.replica.psionic:SetCurrent(current)   -- 可能 nil!
end

-- ✅ 对：加 nil 检查
local function oncurrent(self, current)
    if self.inst.replica.psionic ~= nil then
        self.inst.replica.psionic:SetCurrent(current)
    end
end
```

**原因**：虽然 `entityscript.lua` 第 624 行的 `ReplicateComponent` 在 `cmp(self)` 之前调用——但**理论上 `AddReplicableComponent` 没生效时 `ReplicateComponent` 会跳过**，这时 `inst.replica.psionic == nil`。**onset 永远先 nil 检查**。

**陷阱 3：`DoPeriodicTask` 在 `OnEntitySleep` 时没取消**

```lua
-- ❌ 错：OnEntitySleep 没取消花卉扫描任务
function Psionic:OnEntitySleep()
    self:StopCharging()
    self._sleeptime = GetTime()
    -- 忘了取消 ScanFlowers task!
end
```

**问题**：`ScanFlowers` 的 task 是在 **prefab 工厂函数**里启动的（`inst:DoPeriodicTask(SCAN_PERIOD, ScanFlowers, ...)`），它**不在 psionic 组件里**——组件的 `OnEntitySleep` 取消不了。

**正解**：要么把扫描任务也搬进组件、用 `OnEntitySleep/Wake` 管理；要么在**实体级**（Prefab）的 `OnEntitySleep` 里取消。

**Prefab 级的休眠钩子**写法：

```lua
local function ScanFlowers(inst) ... end

local function StartScanning(inst)
    if inst._scantask == nil then
        inst._scantask = inst:DoPeriodicTask(SCAN_PERIOD, ScanFlowers, math.random() * SCAN_PERIOD)
    end
end

local function StopScanning(inst)
    if inst._scantask ~= nil then
        inst._scantask:Cancel()
        inst._scantask = nil
    end
end

local function OnEntitySleep(inst)
    StopScanning(inst)
end

local function OnEntityWake(inst)
    StartScanning(inst)
    ScanFlowers(inst)   -- 醒来立刻扫一次（更新一下倍率）
end

-- 工厂函数里：
inst.OnEntitySleep = OnEntitySleep
inst.OnEntityWake  = OnEntityWake
StartScanning(inst)
```

**回顾 5.2.4 节**——`OnEntitySleep` 也是实体级钩子，可以挂在 inst 上而不是组件上。**Prefab 启动的 task 由 Prefab 自己负责清理**——这是责任划分的原则。

**陷阱 4：`OnSave` 里把 task 也存了**

```lua
-- ❌ 错：连 task 引用都存了（会崩）
function Psionic:OnSave()
    return {
        current = self.current,
        task = self.task,    -- task 是函数引用，serialize 会爆炸
    }
end
```

**原因**：Lua 的 task 对象包含 C++ 函数指针，**无法序列化**。引擎在写 JSON 时会要么忽略它（如果是函数），要么报错。**OnSave 只存"纯数据"**——number、string、bool、table 嵌套，仅此而已。

**陷阱 5：客户端调用 `replica:SetCurrent`**

```lua
-- ❌ 错：客户端代码（如 UI）调用 SetCurrent 想要"改充能值"
inst.replica.psionic:SetCurrent(50)
```

**原因**：`net_xxx:set` 在客户端调用是**没用的**——网络变量永远是"服务端写、客户端读"。客户端想改服务端状态必须用 **RPC**（章节 7.5 会讲）。

**正确做法**：客户端只用 Replica 来**读取**（`GetCurrent` / `GetPercent`），写操作通过 RPC 发到服务端、由服务端调 `inst.components.psionic:SetCurrent(50)`。

> **新手记忆**：**Replica 是"只读的影子"**——除非你写 classified + Serialize/Deserialize 模式（6.3.6 节），否则客户端永远不应该 `:set` net 变量。

#### 第三步：扩展方向

写完了一个"工业级"的 `psionic` 组件——你可以做哪些更好玩的事？

**(1) 把 `psionic` 挂到玩家身上**

在 `modmain.lua` 加：

```lua
AddPlayerPostInit(function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end
    inst:AddComponent("psionic")
    inst.components.psionic:SetMax(100)
    inst.components.psionic:SetRate(0)   -- 玩家不自动充能
end)
```

——**所有玩家都自带 100 灵能**！默认 `rate=0` 不会自动充能，但可以通过别的方式获取（吃灵能水晶？击杀阴影？）。

**(2) 让灵能短剑优先消耗灵能**

修改 5.7 节的 `psionic_shortsword.lua` 的 `onattack`：

```lua
local function onattack(inst, attacker, target)
    local p = attacker.components.psionic
    if p ~= nil and p:GetCurrent() >= TUNING.PSIONIC_SHORTSWORD_SANITY_COST then
        p:DoDelta(-TUNING.PSIONIC_SHORTSWORD_SANITY_COST)
    elseif attacker.components.sanity ~= nil then
        attacker.components.sanity:DoDelta(-TUNING.PSIONIC_SHORTSWORD_SANITY_COST)
    end
end
```

**逻辑**：有灵能优先扣灵能、没灵能才扣理智。**故事完美闭环**——5.7 + 6.7 = 一个完整的"灵能武器系统"。

**(3) 写一个 HUD 灵能进度条**

写一个 `widgets/psionicbadge.lua`，参考游戏自带的 `widgets/sanitybadge.lua`。在 PlayerHUD 里 `AddChild` 这个 widget，每帧通过 `inst.replica.psionic:GetPercent()` 更新进度。

**这是 7.6 节会讲的内容**——HUD/UI 系统。

**(4) 把"满了 Burst"改成"玩家可以主动激活"**

加一个 RPC：

```lua
-- modmain.lua
AddModRPCHandler("psionic", "burst", function(player, target_guid)
    local target = Ents[target_guid]
    if target ~= nil and target.components.psionic ~= nil then
        target.components.psionic:Burst()
    end
end)

-- 客户端 widget 上某个按钮：
SendModRPCToServer(MOD_RPC.psionic.burst, target.GUID)
```

**这就把"自动 Burst"变成"玩家自主控制"**——更有 gameplay 价值。

**(5) 多个晶簇互相加成**

让一颗晶簇靠近另一颗时，互相提供 `+0.5` 充能率——做出"晶簇阵列"效果：

```lua
local function ScanCrystals(inst)
    local x, y, z = inst.Transform:GetWorldPosition()
    local crystals = TheSim:FindEntities(x, y, z, 4, {"psionic_crystal"}, nil, nil)
    -- 排除自己
    for _, c in ipairs(crystals) do
        if c ~= inst and c.components.psionic ~= nil then
            inst.components.psionic:AddRateModifier(c, 1.5, "crystal_array")
        end
    end
end
```

**这能引导玩家做"基地建造"——把多颗晶簇排在一起做成"灵能矩阵"**。

> **老手记忆**：**一个组件的好坏，不在于它写得多复杂，而在于"它能跟其他组件、其他实体如何组合"**。`psionic` 单独看只是"管理一个数值"，但**配合 SourceModifierList、配合 inventoryitem、配合 RPC、配合 widget**——就能玩出无数花样。**这就是 ECS 思想的力量**。

---

### 6.7.9 小结

**实战收官**——这一节我们从一个空目录开始，做出了一个**真正可用的、覆盖 6.1-6.6 全部知识点的自定义组件**。

#### 第一步：6.1-6.6 的知识点 → 6.7 的代码 对照表

写完这一节，我们建议你回头**逐行看自己写的 `psionic.lua` / `psionic_replica.lua` / `psionic_crystal.lua`**，把它和前 6 节的知识点对照一遍：

| 章节 | 学到的概念 | 在 `psionic` 系列里的对应实现 |
|------|----------|---------------------------|
| **6.1.1-6.1.2** | 组件是 Class、`AddComponent` 的六步流程 | `local Psionic = Class(...)`、Prefab 里 `inst:AddComponent("psionic")` |
| **6.1.3** | 五个核心生命周期钩子 | OnSave / OnLoad / OnEntitySleep / OnEntityWake / OnRemoveFromEntity 全都写了 |
| **6.1.4** | OnSave 三铁律：默认值不存、nil 检查、不存可推断字段 | `OnSave` 用 `current > 0 and self.current or nil`、`OnLoad` 每个字段 nil 检查 |
| **6.1.4** | `add_component_if_missing` 兼容机制 | OnSave 返回的 table 里加了这个标记 |
| **6.1.5** | OnEntitySleep 取消 task、记时间戳；OnEntityWake 补偿 | `self._sleeptime = GetTime()` + `LongUpdate(elapsed)` |
| **6.1.6** | OnRemoveFromEntity 三件套：停 task、移 tag、置 nil | 完整实现了三件套 |
| **6.1.8** | onset 回调机制 | `oncurrent` / `onmax` 自动加移 tag、自动同步 replica |
| **6.2.3** | Replica 骨架 | `psionic_replica.lua` 标准结构 |
| **6.2.4** | `AddReplicableComponent` | `modmain.lua` 顶层调用 |
| **6.2.5** | `_health` 这种 tag 标记 | `ReplicateComponent` 自动加 `_psionic` tag |
| **6.2.6** | Replica 三种数据来源 | 这里只用了 net_xxx（最轻量） |
| **6.3.3** | net 变量类型选择（small/byte/short/int） | 用 `net_ushortint` 平衡精度和带宽 |
| **6.3.6** | net 变量+dirty event 模式 | `psionicchangedirty` event 把 net 变化转抛业务事件 |
| **6.4.6** | Timer 组件 vs 原生 task | 这里用 `DoPeriodicTask`（简单场景）；如果 rate 经常被 Pause/Resume 应该改用 timer 组件 |
| **6.5.2-6.5.3** | SourceModifierList API | `self.rate_modifiers` 用乘法叠加 |
| **6.5.4** | source 实体 vs 字符串选择 | 用 `inst` 自己作为 source（保证 modifier 能被自动清理） |
| **6.5.5** | key 区分多重 modifier | `"poppy_aura"` / `"evil_aura"` 两个 key 共享一个 source |
| **6.6.1-6.6.3** | 读组件五步法 | 现在反过来——你写完了组件，可以让别人用五步法读懂它 |
| **6.6.4** | 反向追踪（`rg AddComponent`） | 写完后用 `rg "components.psionic"` 看自己组件被哪些代码用了 |

——**整章 6.1 到 6.6 的知识点，在 6.7 几乎全部用了一次**。这就是为什么实战是最好的复习。

#### 第二步：组件作者的"上线检查清单"

写完一个组件，**上线前**用这张清单逐项打勾：

```
□ 1. Class 构造函数：所有字段都有默认值（不留 nil 字段）
□ 2. onset 回调：能影响 tag/replica/外部状态的字段都用 onset 同步
□ 3. Setter 方法：所有"对外修改"通过 SetXxx 进行，不允许直接赋值
□ 4. DoDelta 模式：变化量类型的字段用 DoDelta 而不是 SetCurrent，方便监听
□ 5. 事件设计：状态变化抛事件、边界变化（满/空/死）单独抛
□ 6. OnSave：默认值不存、返回 nil 不存、加 add_component_if_missing
□ 7. OnLoad：每个字段 nil 检查、用 setter 而不是直接赋值
□ 8. OnEntitySleep：所有 DoPeriodicTask 都取消、所有 ListenForEvent 视情况解除
□ 9. OnEntityWake：根据当前状态决定是否重启 task；如有时间补偿调 LongUpdate(elapsed)
□ 10. LongUpdate：实现，让 c_longupdate 也能正确推进
□ 11. OnRemoveFromEntity：清理"组件加上去的所有东西"——task / tag / 监听 / 引用
□ 12. SourceModifierList：凡是会被多 buff 同时改的字段，用 modifier 而不是裸字段
□ 13. modmain.lua AddReplicableComponent: 写在顶层，不写在回调里
□ 14. _replica.lua：net 变量两端都声明、Get 优先读真组件、Set 只服务端
□ 15. GetDebugString：实现，方便 Ctrl+Tab 实时调试
```

**符合 80% 以上**——你的组件就是"工业级"的，能放心用在自己的 Mod、甚至上传到创意工坊。

#### 第三步：实战经验六条

最后是我（以及很多 Mod 老手）从无数次踩坑中提炼的经验：

1. **先做 Walking Skeleton 再加血肉**——5 个空文件先跑起来，再一点点填代码。任何时刻都让你的代码处于"可运行"状态，bug 永远在 5 行之内。
2. **读官方组件比读教程更有效**——本节的所有写法都对照了 `health.lua` / `fueled.lua` / `perishable.lua` / `inventoryitemmoisture.lua`。Klei 的源码就是最好的教科书。
3. **遇到不懂就 Ctrl+Tab + c_sel + print**——回顾 6.6.6 节的"运行时侦探"手法，比静态读代码快 10 倍。
4. **永远走"正门"**——SetXxx / DoDelta / AddRateModifier 这些是组件的"接口"。**自己写的代码也要走正门**，避免漏掉同步、漏掉事件。
5. **思考"几年后的自己"和"未来的协作者"**——你的组件可能被另一个 Mod 拓展、被 PostInit 修改、被 RPC 调用。每一行代码都问自己：**"如果别人改了这里，会不会引入 bug？"** 防御式编程不是啰嗦，是负责任。
6. **不是所有组件都需要 Replica**——只有客户端真正需要"看到"这个数据时才同步。比如 `fueled` 没有 replica（只通过 section 间接同步），是因为客户端不需要精确的剩余秒数。**节省网络带宽就是节省玩家延迟**。

#### 第四步：本章总结

**第 6 章的主线**：

- **6.1 拆开了组件的生命周期**——从 `AddComponent` 的六步流程到五个核心钩子，让你看懂任何组件的骨架。
- **6.2 揭开了 Replica 的面纱**——客户端"影子组件"是怎么和服务端真组件协作的，让多人游戏的状态同步成为可能。
- **6.3 深入了 Classified 实体**——`player_classified` 这种"超级桥梁"是怎么把上百个字段打包同步的。
- **6.4 拆解了 Timer 组件**——比 `DoPeriodicTask` 更"持久化"的计时管理。
- **6.5 引入了 SourceModifierList**——让"多 buff 叠加"这种业界难题变得简单。
- **6.6 教了你如何阅读源码**——五步法、反向追踪、运行时探索，让你成为组件世界的"侦探"。
- **6.7 让你亲手做了一个组件**——把前面所有知识点串起来，做出一颗会自动充能、能存档、会休眠、有客户端 UI 数据、能被多源 buff 影响、能释放范围效果的"灵能晶簇"。

**完成第 6 章后，你已经是"半个组件作者"了**。你能做到：

- ✅ 看懂任何官方组件的源码（哪怕是 1000+ 行的复杂组件）
- ✅ 写出符合 Klei 风格的自定义组件（其他 Mod 作者会觉得你写的代码"很 Klei"）
- ✅ 处理多人同步问题（不再有"客户端为什么看不到"的困惑）
- ✅ 用 `c_sel` / `Ctrl+Tab` / `print hook` 在游戏里实时调试

> **下一章预告**：第 7 章我们将进入"**事件、StateGraph、Brain**"的世界——组件之间是怎么"对话"的？为什么巨牛会自己求偶、会自己睡觉、会自己挑食？怎么让 NPC 看到玩家就跑？这些"行为决策"的代码 Klei 是用一套**事件驱动 + 状态机 + AI 决策树**的三层架构实现的——而我们会把它一层层剥开来看。

> **再下一章预告**（第 8 章）：写完了行为，还需要给行为加"特效"——粒子系统、动画、声音、屏幕震动……第 8 章是"**视听层 (Audio-Visual Layer)**"的全套——让你的 Mod 不光"能玩"，还要"好看、好听、有手感"。
