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

（待编写）

## 6.3 Classified 实体——网络同步的桥梁

（待编写）

## 6.4 Timer 组件——组件级计时器的正确使用

（待编写）

## 6.5 SourceModifierList——多来源叠加的通用工具

（待编写）

## 6.6 如何阅读和理解一个组件的源码

（待编写）

## 6.7 实战：编写一个自定义组件（含 Replica）

（待编写）
