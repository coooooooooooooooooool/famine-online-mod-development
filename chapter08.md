# 第8章 事件系统

## 8.1 PushEvent / ListenForEvent / RemoveEventCallback

### 本节导读

第 7 章我们把 Action 系统讲透了——**玩家输入 → 实体动作**这条链路完全打通。但**实体之间是怎么互相"通气"的**？比如：

- 玩家**死了**——成就系统、复活系统、世界 boss 进度全要立刻知道
- 树**被砍倒**——临近的猪人、克劳斯日记、地形系统都要响应
- 玩家**装备了一件新衣服**——温度组件、防御组件、外观组件**同时**要更新

如果让"死亡逻辑"自己直接调"成就 +1"、再调"复活监控"——业务**死死耦合**，往里塞一个新 mod 都得改原版代码。**事件系统**就是为了解决这种耦合：**生产者（Pusher）只负责"喊一声"，消费者（Listener）自己决定听不听、怎么响应**——典型的**观察者模式**。

这一章我们打开饥荒里**最贴近 mod 开发者**的两套通信机制：

> **新手**从 8.1.1-8.1.3 起步——三件套 API 怎么用、`source` 参数为什么重要；**进阶读者**继续看 8.1.4-8.1.6，深入"双向反查表"的存储结构、`PushEvent` vs `PushEventImmediate` vs `PushEventInTime`、`RemoveAllEventCallbacks` 在实体销毁时的自动清理；**老手**跳到 8.1.7-8.1.8，逐行拆 `EntityScript:PushEvent_Internal`、闭包/匿名函数解除问题、六个最容易踩的坑——尤其是**回调函数变量丢失导致永远拆不掉**这种典型翻车场景。

---

### 8.1.1 快速入门：从一次"砍树掉血"看事件是什么

#### 第一步：在游戏里观察一次完整的"事件链"

游戏里**用斧头砍一只猪人**——

**你看到的**：
1. 猪人头顶冒红色伤害数字
2. 猪人音效：哀嚎一声
3. 猪人转身扑向玩家
4. 多砍几次后猪人倒地
5. 同屏的另一只猪人**也**冲过来帮忙（"为兄弟报仇"）

**引擎在背后做了什么？**

每一步都对应一个**事件 PushEvent**：

```
[1] combat:GetAttacked() 调用
       ↓
[2] inst:PushEvent("attacked", {attacker=玩家, damage=10, ...})
       ↓
[3] 猪人自己在 prefab 里 ListenForEvent("attacked", OnAttacked)
    → 播放哀嚎音效、设置 target = 玩家
       ↓
[4] inst:PushEvent("healthdelta", {oldpercent=1, newpercent=0.7, ...})
    → 头顶冒伤害数字（DamageNumber 组件监听）
       ↓
[5] 健康降到 0 后 health:Kill()
       ↓
[6] inst:PushEvent("death", {cause=...})
       ↓ (世界级的)
    TheWorld:PushEvent("entity_death", {inst=猪人, cause=...})
       ↓
[7] 周围猪人监听 "entity_death" → 找凶手 → 攻击玩家
```

**整个链路里事件起的作用**：**把"发生了什么"广播出去**——具体由**谁在意、要怎么响应**全权交给 listener 决定。

> **核心结论**：饥荒里几乎所有"涉及多个实体、多个组件协同"的逻辑——**全部走事件**。事件**不是 Action 的替代**——Action 是"玩家意图 → 行为"的入口；事件是"行为发生 → 系统协同"的总线。

#### 第二步：看一眼真实源码——`combat.lua` 的 PushEvent

打开 `scripts/components/combat.lua` 第 686 行：

```686:686:scripts/components/combat.lua
		self.inst:PushEvent("attacked", { attacker = attacker, damage = damage, damageresolved = damageresolved, original_damage = original_damage, weapon = weapon, stimuli = stimuli, spdamage = spdamage, redirected = damageredirecttarget, noimpactsound = self.noimpactsound })
```

**就一行。** 一个 `PushEvent`，事件名 `"attacked"`，data 字典里塞了 9 个字段——攻击者、伤害值、武器、刺激源、是否重定向……**所有 listener 看到的都是同一份 data**。

再看 `health.lua` 的 `Kill` 函数：

```589:590:scripts/components/health.lua
        TheWorld:PushEvent("entity_death", { inst = self.inst, cause = cause, afflicter = afflicter, corpsing = is_corpsing })
        self.inst:PushEvent("death", { cause = cause, afflicter = afflicter, corpsing = is_corpsing })
```

**注意这两行的差异**：
- 第一行 `TheWorld:PushEvent("entity_death", ...)` —— **世界级事件**，所有监听 `TheWorld` 的 listener 都会收到（比如成就系统、boss 监控）
- 第二行 `self.inst:PushEvent("death", ...)` —— **实体级事件**，只有监听**这个实体**的 listener 才会收到（比如这个生物的 prefab、它的 stategraph、它的 brain）

**事件作用域**这一概念——8.2 节会专门讲。本节先聚焦**实体级事件**。

#### 第三步：再看一眼"听"的一面——`ListenForEvent` 实战

`scripts/prefabs/pigman.lua`（虚构的简化版，原版结构类似）：

```lua
-- 猪人 prefab 在 fn 里注册"听"
inst:ListenForEvent("attacked", function(inst, data)
    if inst.components.combat ~= nil and data.attacker ~= nil then
        inst.components.combat:SetTarget(data.attacker)
    end
end)
```

**两段代码合起来**——`combat.lua` 的 `PushEvent("attacked", ...)` 是发送方、`pigman.lua` 的 `ListenForEvent("attacked", ...)` 是接收方。**它们之间没有直接调用**——通过事件名 `"attacked"` 解耦。

> **新手到这里只需记住**：`PushEvent` 是"喊"，`ListenForEvent` 是"听"，`RemoveEventCallback` 是"不听了"。下面把三件套 API 速查列出来。

---

### 8.1.2 快速入门：三件套 API 速查

#### 第一步：API 签名

| API | 签名 | 用途 |
| --- | --- | --- |
| `inst:PushEvent(event, data)` | `(string, table?) -> nil` | 推送事件——**自己**作为发送源 |
| `inst:ListenForEvent(event, fn, source?)` | `(string, function, EntityScript?) -> nil` | **inst** 监听 **source** 实体的 event 事件，回调 fn |
| `inst:RemoveEventCallback(event, fn, source?)` | `(string, function, EntityScript?) -> nil` | 解除一次注册——**fn 必须是注册时同一个函数引用** |
| `inst:RemoveAllEventCallbacks()` | `() -> nil` | 把 inst 注册的所有监听全部解除——**不论事件名、不论 source** |
| `inst:PushEventImmediate(event, data)` | `(string, table?) -> nil` | 立刻同步推送给 SG（绕过 SG 的事件队列），8.1.5 节专讲 |
| `inst:PushEventInTime(time, eventname, data)` | `(number, string, table?) -> nil` | 延时 time 秒后推送 |

#### 第二步：5 行代码完整示范

```lua
-- 1) 自定义组件里推事件
function MyComponent:DoSomething()
    self.inst:PushEvent("mycustomevent", { who = self.inst, value = 42 })
end

-- 2) prefab 的 fn 里听事件
inst:ListenForEvent("mycustomevent", function(inst, data)
    print("received:", data.who, data.value)
end)
```

**就这么简单**。`event` 名字**自己定**——只要 push 和 listen 用同一个字符串就行（注意大小写敏感）。`data` 是个**任意 table**——你想塞什么塞什么，listener 收什么是什么。

#### 第三步：参数细节速查

**`event`（事件名）**
- 类型：**字符串**
- 大小写敏感：`"attacked"` ≠ `"Attacked"` —— **永远小写**是 Klei 约定（陷阱见 8.1.8 陷阱 1）
- 没有命名空间——**所有 mod 共享同一个事件名空间**——所以自定义事件**带上 mod 前缀**避免冲突，比如 `"mymod_dance"`

**`data`（事件载荷）**
- 类型：**任意 table**，可省略（默认 nil）
- listener 收到的是**同一引用**——**不要在 listener 里改 data 的字段**（除非你确认只有一个 listener，且原作者也允许）
- 嵌套结构 OK：`{ attacker = inst, damage = 10, weapon = act.invobject, ... }`

**`fn`（回调）**
- 签名：`function(inst, data)`
  - **第一个参数 `inst` 是 source**（事件源），**不是 listener self！**
  - 第二个参数 `data` 就是 push 时传的 table
- 这个签名**和"谁在监听"无关**——回调里要拿到 listener 实体得**用闭包捕获 self**（8.1.7 第二步详解）

**`source`（事件源）**
- 类型：`EntityScript`，可省略（默认 self）
- **省略 = "我监听我自己的事件"**——这是最常见的写法
- **传别的实体 = "我监听别人的事件"**——比如玩家 GUI 里 `ThePlayer:ListenForEvent("hungerdelta", ..., target_player)`

> **新手 MVP 检查点**：能写出"组件 PushEvent + prefab fn 里 ListenForEvent"两行——就达到 80% 场景的事件系统使用要求了。

---

### 8.1.3 快速入门：`ListenForEvent` 的 `source` 参数为什么重要

#### 第一步：先看两种写法的差别

**写法 A：监听自己**
```lua
-- 在 pigman.lua 里
inst:ListenForEvent("death", function(inst, data)
    print("我自己死了")
end)
```

`source` 省略 = `inst`（自己）—— **猪人监听自己的 death**。当 `inst:PushEvent("death", ...)` 发生时，回调被触发。

**写法 B：监听别人**
```lua
-- 在主玩家身上注册
ThePlayer:ListenForEvent("death", function(player, data)
    print("我看到一只生物死了：", player)
end, target_pigman)
```

`source = target_pigman` —— **玩家监听某只猪人的 death**。当**那只猪人** `inst:PushEvent("death", ...)` 时，回调被触发。**注意第一个参数是 source（猪人），不是 listener（玩家）**。

#### 第二步：为什么要分 inst 和 source？

看 `EntityScript:ListenForEvent` 的真实源码：

```1188:1203:scripts/entityscript.lua
function EntityScript:ListenForEvent(event, fn, source)
    --print ("Listen for event", self, event, source)
    source = source or self

    if not source.event_listeners then
        source.event_listeners = {}
    end

    AddListener(source.event_listeners, event, self, fn)

    if not self.event_listening then
        self.event_listening = {}
    end

    AddListener(self.event_listening, event, source, fn)
end
```

**两件事**：
1. 在 **source.event_listeners** 里登记"**self** 在听 **event** 这个事件"——push 时 source 拿这张表分发
2. 在 **self.event_listening** 里登记"**self** 在听 **source** 的 **event**"——self 销毁时反查这张表去清理

**这是双向链表结构**——为什么要这么做？**为了 `RemoveAllEventCallbacks` 能在实体销毁时正确清理跨实体监听**。8.1.4 节会画图详解。

#### 第三步：常见的 source 用法 ——"GUI 监听玩家组件"

最经典的场景是 **HUD widgets 监听玩家**：

```lua
-- scripts/widgets/healthbadge.lua（虚构的简化）
function HealthBadge:OnAttachToOwner(owner)
    self._owner = owner
    self.inst:ListenForEvent("healthdelta", function(player, data)
        self:UpdateBar(data.newpercent)
    end, owner)  -- ← 第三个参数：owner = ThePlayer
end
```

**这里 widget 不是 EntityScript** —— widget 通常是个 lua object，但 `self.inst` 是 widget 内嵌的轻量 entity。**重点是 source = owner**——widget 监听**玩家**的 healthdelta，玩家本身的 `health.lua` push 事件后，widget 立刻更新血条。

> **设计哲学**：监听跨实体事件时，把"听者"和"被听者"严格区分——**不是"我喊大家来听"**，而是"**我自己决定听谁**"。这种"反向订阅"是观察者模式的精髓。

#### 第四步：在控制台亲眼验证

```lua
-- 给猪人加一个临时 listener
local pig = c_select()
pig:ListenForEvent("death", function(inst, data)
    print("猪死了！原因:", data.cause)
end)

-- 强制让它死
pig.components.health:Kill()
-- 输出：猪死了！原因: nil
```

```lua
-- 跨实体监听
ThePlayer:ListenForEvent("death", function(source, data)
    print("我看到", source, "死了")
end, pig)

c_select():components.health:Kill()
-- 输出：我看到 pigman[xxx] 死了
```

> **新手到这里**：三件套用法已经掌握，下一节进阶部分把"为什么 listener 要存两份"和"事件作用域中的关键差异"讲透。

---

### 8.1.4 进阶：事件回调存储的"双向反查表"结构

#### 第一步：先把存储结构画出来

回到 8.1.3 第二步那段源码——`ListenForEvent` 干了**两件事**：往 `source.event_listeners` 里写一份、往 `self.event_listening` 里写一份。**两份表的结构如下**：

```
source.event_listeners = {
    [event_name] = {
        [listener_inst1] = { fn1, fn2 },  -- 同一个 listener 可以注册多个 fn
        [listener_inst2] = { fn3 },
    }
}

self.event_listening = {
    [event_name] = {
        [source_inst1] = { fn1 },         -- 同一个 listener 可以监听多个 source
        [source_inst2] = { fn3, fn4 },
    }
}
```

**举例**：玩家监听了**两只**猪人 A、B 的 `"death"`，**和自己的** `"hungerdelta"`：

```
ThePlayer.event_listening = {
    ["death"] = {
        [pigman_A] = { fn1 },
        [pigman_B] = { fn1 },
    },
    ["hungerdelta"] = {
        [ThePlayer] = { fn2 },
    },
}

pigman_A.event_listeners = {
    ["death"] = {
        [ThePlayer] = { fn1 },
    },
}
```

#### 第二步：为什么要双向？

回到 `EntityScript:RemoveAllEventCallbacks`：

```1232:1262:scripts/entityscript.lua
function EntityScript:RemoveAllEventCallbacks()
    --tell others that we are no longer listening for them
    if self.event_listening then
        for event, sources  in pairs(self.event_listening) do
            for source, fns in pairs(sources) do
                if source.event_listeners then
                    local listeners = source.event_listeners[event]
                    if listeners then
                        listeners[self] = nil
                    end
                end
            end
        end
        self.event_listening = nil
    end    

    --tell others who are listening to us to stop
    if self.event_listeners then
        for event, listeners in pairs(self.event_listeners) do
            for listener, fns in pairs(listeners) do
                if listener.event_listening then
                    local sources = listener.event_listening[event]
                    if sources then
                        sources[self] = nil
                    end
                end
            end
        end
        self.event_listeners = nil
    end
end
```

**两段循环对应两份表**：

- 第一段：**遍历自己听的所有 source**，去**对方的 event_listeners** 把自己删掉——避免 source 之后还往自己这里推事件，但自己已经销毁
- 第二段：**遍历所有听自己的 listener**，去**对方的 event_listening** 把自己删掉——避免 listener 销毁时反查 self.event_listeners 还能找到已销毁的 source

> **这是观察者模式 + GC 的经典实现**——双向链表使得**任意一方销毁时都能正确清理另一方的引用**，从根本上避免悬挂指针。

**实体销毁触发 `RemoveAllEventCallbacks` 的位置**——`EntityScript:OnRemoveEntity` 内部会调它（不展开）。**这就是为什么实体销毁时你不用手动 `RemoveEventCallback`**——8.1.6 节细讲。

#### 第三步：用控制台亲眼看双向表

```lua
-- 在 ThePlayer 身上注册一个监听
local pig = c_spawn("pigman")
local cb = function(inst, data) print("pig dead") end
ThePlayer:ListenForEvent("death", cb, pig)

-- 看 ThePlayer 这边的表
print(ThePlayer.event_listening["death"][pig])
-- => table 0xXXX (含 cb)

-- 看 pig 那边的表
print(pig.event_listeners["death"][ThePlayer])
-- => table 0xXXX (含 cb)

-- RemoveEventCallback 后两边都消失
ThePlayer:RemoveEventCallback("death", cb, pig)
print(ThePlayer.event_listening and ThePlayer.event_listening["death"])  -- nil
print(pig.event_listeners and pig.event_listeners["death"])               -- nil
```

#### 第四步：同一个 (event, listener, source) 三元组下的 fn 列表

注意 `event_listeners[event][listener]` 的值**是个数组 `{fn1, fn2, ...}`**——你可以**对同一个 source 的同一个 event 注册多次** ListenForEvent，每个 fn 会**全部触发**。

**这是 7.4.7 链式 SuccessAction/FailAction 在 EntityScript 这一级的同款机制**——典型的"多个独立模块都关心同一个事件"。

**RemoveByValue 的实现细节**：

```1205:1221:scripts/entityscript.lua
local function RemoveListener(t, event, inst, fn)
    if not t then return end

    local listeners = t[event]
    if not listeners then return end

    local listener_fns = listeners[inst]
    if listener_fns then
        RemoveByValue(listener_fns, fn)
        if next(listener_fns) == nil then
            listeners[inst] = nil
        end
    end
    if next(listeners) == nil then
        t[event] = nil
    end
end
```

**关键观察**：`RemoveByValue(listener_fns, fn)`——如果**两次注册了同一个 fn 引用**，`RemoveEventCallback` 一次只删**一个**。**重复注册要重复删除**。

> **典型陷阱**：在 prefab fn 里**两次** `ListenForEvent("death", OnDeath)`——一次"业务逻辑"、一次"音效"——结果删一次还剩一次。详见 8.1.8 陷阱 4。

---

### 8.1.5 进阶：`PushEvent` vs `PushEventImmediate` vs `PushEventInTime`

#### 第一步：三者源码对比

```1286:1323:scripts/entityscript.lua
function EntityScript:PushEvent_Internal(event, data, immediate)
    if self.event_listeners then
        local listeners = self.event_listeners[event]
        if listeners then
            --make a copy list of all callbacks first in case
            --listener tables become altered in some handlers
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

	if self.sg then
		if immediate then
			self.sg:HandleEvent(event, data)
		elseif self.sg:IsListeningForEvent(event) and SGManager:OnPushEvent(self.sg) then
			self.sg:PushEvent(event, data)
		end
    end

    if self.brain then
        self.brain:PushEvent(event, data)
    end
end

function EntityScript:PushEvent(event, data)
	self:PushEvent_Internal(event, data, false)
end

function EntityScript:PushEventImmediate(event, data)
	self:PushEvent_Internal(event, data, true)
end
```

**核心差别在 SG 层**：
- **普通 `PushEvent`**：SG 进入**事件队列**——这一帧或下一帧由 `SGManager` 调度处理
- **`PushEventImmediate`**：直接 `sg:HandleEvent`——**当前调用栈里立刻执行**——绕过队列

**而 `event_listeners` 那部分（普通 lua 监听）**：**两者都是立刻同步触发**——`for fn in tocall do fn(self, data) end`。

#### 第二步：拷贝列表的精妙细节

注意第 1292-1296 行：

```lua
local tocall = {}
for entity, fns in pairs(listeners) do
    for i, fn in ipairs(fns) do
        table.insert(tocall, fn)
    end
end
for i, fn in ipairs(tocall) do
    fn(self, data)
end
```

**为什么先拷贝？** 因为某个 fn 可能在执行过程中**注册新 listener** 或者 **`RemoveEventCallback`**——直接遍历 `listeners` 会破坏迭代器。**拷贝一份再遍历**——这是**安全迭代**的标准做法。

**它给开发者的承诺**：**你可以在 listener fn 里 `RemoveEventCallback` 自己**——不会崩。

#### 第三步：`PushEventImmediate` 什么时候用？

**95% 场景下用普通 `PushEvent`**——除非：

1. **当前帧内必须让 SG 立刻响应**：比如玩家死亡时 SG 必须**马上**切到 `death` state，不能等下一帧（否则可能多挨一记伤害）
2. **事件触发链很深、需要严格同步**：比如"钓鱼成功 → 鱼上钩 → 立刻播音效（SG）→ 立刻给物品"

**Klei 自己用 `PushEventImmediate` 的地方非常少**——`scripts/` 里全局 grep 大概十几处。**新手可以忽略**。

#### 第四步：`PushEventInTime` 实战

```1532:1540:scripts/entityscript.lua
function EntityScript:PushEventInTime(time, eventname, data)
    self.pendingtasks = self.pendingtasks or {}

    local event_function = function(inst)
        inst:PushEvent(eventname, data)
    end
    local periodic = scheduler:ExecuteInTime(time, event_function, self.GUID, self, data)
    self.pendingtasks[periodic] = true
    periodic.onfinish = task_finish
```

**用法**：

```lua
-- 5 秒后推送"我醒了"
inst:PushEventInTime(5, "myhibernate_wakeup", { reason = "noon" })
```

**实现要点**：
- 用 `scheduler:ExecuteInTime` 排队——**不阻塞当前调用栈**
- 注册到 `self.pendingtasks` ——实体销毁时**自动取消**（避免对已销毁实体推事件）
- 闭包捕获 inst —— **闭包**会让 inst 引用计数 +1，**短期延时**没问题，**长期持有**要小心 GC

**典型用例**：动画播完后再推业务事件、cooldown 结束后推一个"刷新 UI"事件、给敌人 0.5 秒反应时间再 push 一个 alert。

#### 第五步：四种 push 方式对照

| API | 同步性 | SG 处理时机 | 典型用例 |
| --- | --- | --- | --- |
| `PushEvent` | listener 立刻触发；SG 进队列 | 当前帧或下一帧 | 95% 默认场景 |
| `PushEventImmediate` | 完全同步——listener 和 SG 都立刻 | 当前调用栈 | 必须当帧 SG 切状态 |
| `PushEventInTime` | 异步（延时 + 走普通 push） | 延时后下一帧 | 动画后业务、cooldown 后刷新 |
| `TheWorld:PushEvent` | 同 PushEvent，但作用域是世界 | 世界 SG（如果有） | 8.2 节专讲 |

---

### 8.1.6 进阶：`RemoveAllEventCallbacks` 与实体销毁时的自动清理

#### 第一步：什么时候被自动调用？

`Entity:Remove()` → `EntityScript:OnRemoveEntity()` 内部会**链式清理所有外部引用**：

- `RemoveAllEventCallbacks()` —— 清事件
- `StopAllWatchingWorldStates()` —— 清 worldstate watchers
- `RemoveTags`、关组件、关 SG 等 —— 不一一展开

**结论**：**只要实体被 `inst:Remove()` 正常销毁，所有事件监听都会自动解开**——你不需要在 `OnRemoveEntity` 里再写 `RemoveEventCallback`。

#### 第二步：什么时候你**仍然需要**手动 `RemoveEventCallback`？

**3 个典型场景**：

1. **动态注册/解注册**：比如"装备某武器时监听玩家 healthdelta、卸下时取消监听"——**没销毁实体**，不能等 `RemoveAllEventCallbacks`，只能 `RemoveEventCallback`
2. **跨实体监听 + listener 长寿**：玩家**永远不死**，他监听过的猪人都死了——push 这边的清理由 source 销毁时反向走第二段循环（见 8.1.4 第二步）；但**listener 自己 event_listening 表里那条记录**也会被同步清理。**所以这种场景也不需要手动**——除非你要"提前停"
3. **临时监听 + 业务期满**：比如"装填 5 颗子弹后停止监听弹药消耗"——**这才是必须手动 RemoveEventCallback** 的场景

#### 第三步：实战 1—— 装备/卸下场景

```lua
local function OnPlayerHealthDelta(player, data)
    if data.newpercent < 0.3 then
        player.components.combat.damage_buff = 1.5
    end
end

local function OnEquipped(inst, data)
    -- 装上头盔时，监听玩家血量
    data.owner:ListenForEvent("healthdelta", OnPlayerHealthDelta)
end

local function OnUnequipped(inst, data)
    -- 卸下时，**必须**手动取消监听
    data.owner:RemoveEventCallback("healthdelta", OnPlayerHealthDelta)
end

-- 在 prefab fn 里
inst:ListenForEvent("equipped", OnEquipped)
inst:ListenForEvent("unequipped", OnUnequipped)
```

**关键点**：`OnPlayerHealthDelta` 是个**模块级 local 函数**——**注册和解注册引用同一个值**。如果写成匿名 function（每次创建新闭包），就解不掉。**这是 8.1.7 节闭包陷阱的反例**。

#### 第四步：实战 2—— 临时监听 + "解一次后再注册"

需求：玩家**第一次**喝水才推音效，之后不再推。

```lua
local function OnFirstDrink(inst, data)
    inst.SoundEmitter:PlaySound("dontstarve/firstdrink")
    -- 解除自己——只触发一次
    inst:RemoveEventCallback("drink", OnFirstDrink)
end

inst:ListenForEvent("drink", OnFirstDrink)
```

**核心技巧**：**注册一个具名函数 → 在 fn 内部解除自己**。这是"once 监听"的标准实现。

> **更优雅的封装**——可以写一个全局工具：

```lua
function EntityScript:ListenForEventOnce(event, fn, source)
    local wrapped
    wrapped = function(inst, data)
        fn(inst, data)
        self:RemoveEventCallback(event, wrapped, source)
    end
    self:ListenForEvent(event, wrapped, source)
end
```

> 但 Klei 原版**没有提供这个 API**——开源 mod 里看到这种封装是 mod 自己写的。

---

### 8.1.7 老手进阶：闭包陷阱与"匿名函数解除问题"

#### 第一步：先看一段错误代码

```lua
-- 错误示范！
inst:ListenForEvent("attacked", function(inst, data)
    print("被打了")
end)

-- ... 别的地方
inst:RemoveEventCallback("attacked", function(inst, data)
    print("被打了")
end)
-- 这一句 RemoveEventCallback 完全不生效！
```

**原因**：每个 `function() ... end` 字面量都创建**一个全新的 function 对象**——两次写出"看起来一样"的函数体，**Lua 视角下是两个不同的引用**。

`RemoveByValue` 用 `==` 比较——**只有同一个引用才算**。

#### 第二步：正确写法 1——具名函数

```lua
local function OnAttacked(inst, data)
    print("被打了")
end

inst:ListenForEvent("attacked", OnAttacked)
inst:RemoveEventCallback("attacked", OnAttacked)  -- 同一个引用，OK
```

**这是最常见的写法**——把回调定义成 prefab 文件顶层的 local function。

#### 第三步：正确写法 2——存到实体 / 自身上

```lua
-- 在某个组件里
function MyComp:WatchHealth()
    self._on_healthdelta = function(player, data)
        self:OnHealthChange(data)
    end
    self.inst:ListenForEvent("healthdelta", self._on_healthdelta)
end

function MyComp:UnwatchHealth()
    if self._on_healthdelta then
        self.inst:RemoveEventCallback("healthdelta", self._on_healthdelta)
        self._on_healthdelta = nil
    end
end
```

**精妙之处**：闭包把 `self` 捕获进去——回调内部能调 `self:OnHealthChange(data)`，**回调引用本身存在 `self._on_healthdelta`** —— 注册和解注册引用同一个值。

#### 第四步：用闭包但不能解除——绕开法

如果你**实在**只想用一次性匿名函数（比如"5 秒后印一句话就不管了"），用 `DoTaskInTime` 或 `PushEventInTime` 替代 ListenForEvent：

```lua
-- 不要用 ListenForEvent，因为没有解除引用
inst:DoTaskInTime(5, function(inst)
    print("5 秒过去了")
end)
```

**核心原则**：**ListenForEvent 必须能拿到回调引用，否则就用别的机制**。

#### 第五步：闭包导致的"实体引用泄漏"

**反例**：

```lua
-- 在 prefab.lua 里
inst:ListenForEvent("playerentered", function(world, data)
    if data.player == ThePlayer then
        inst.boss_target = ThePlayer  -- 闭包 + 字段都引用 ThePlayer
    end
end, TheWorld)
```

**问题**：这个 listener 注册到 `TheWorld.event_listeners` 里——**永远不会自动解除**（TheWorld 不会销毁）。如果 inst 销毁了，`RemoveAllEventCallbacks` 会解掉，**但闭包内捕获的 `inst` 已经悬挂**——回调被 push 时访问 `inst.boss_target` 可能崩溃。

**修复**：实体销毁时 `inst:RemoveAllEventCallbacks()` 会自动清掉对 TheWorld 的监听（因为双向表的第一段循环），所以**实际上是安全的**——但**前提是实体走正常 Remove 流程**。**自定义不走 Remove 的特殊场景下要警惕**。

> **设计经验**：跨"长寿实体（TheWorld、ThePlayer、ThePlayer.HUD）"的监听，永远把回调引用**显式存到自身字段**，便于必要时手动 RemoveEventCallback。

---

### 8.1.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：事件名大小写 / 拼写错误

**症状**：明明 `PushEvent("attacked")` 触发了，但 `ListenForEvent("Attacked", fn)` 永远没收到。  
**原因**：事件名**字符串字面量**——比较的是 `"attacked" == "Attacked"`，false。  
**修复**：**全部小写**——这是 Klei 约定。**自定义事件**用下划线分词：`"mymod_dance_finished"`。

#### 陷阱 2：用匿名 function 注册却想解除

**症状**：`RemoveEventCallback` 看似调用了，但 listener 仍然被触发。  
**原因**：见 8.1.7 第一步——**两次匿名函数引用不同**。  
**修复**：永远把 callback 定义成 **local function** 或存到 `self.xxx` 字段。

#### 陷阱 3：在 listener fn 里**修改 data 字段**导致下一个 listener 看错

**症状**：多个 mod 都监听 `"attacked"`，第一个把 `data.damage` 改成 0、后面所有 listener 都看不到伤害。  
**原因**：`data` 是同一引用——**任何 listener 修改都立即对其他 listener 可见**。  
**修复**：**只读**——如果你确实要修改伤害值，应该走"伤害修改 callback"机制（如 `combat.damagemodifier`），不要侵入事件 data。

> **想覆盖伤害**——用 `combat.damagedeflectfn` / `health.deltamodifierfn`——见 7 章对应组件。

#### 陷阱 4：重复 ListenForEvent

**症状**：每次玩家复活，监听数翻倍——5 次复活后一个 OnDeath 触发 5 次。  
**原因**：在 `OnPlayerSpawn` 之类的回调里**没判断是否已注册**就 ListenForEvent，每次都新增一份。  
**修复 1**：先调 `RemoveEventCallback`，再 `ListenForEvent`——保证最多一份。  
**修复 2**：把回调存到 `self._cb`，注册前判断 `if not self._cb then`。

#### 陷阱 5：在 SG state 的 onenter 里 ListenForEvent，但忘了在 onexit 里 RemoveEventCallback

**症状**：玩家进出 state 100 次后，listener 数量爆炸，性能下降。  
**原因**：state 是**多次进入**的——每次 onenter 都新增一份监听。  
**修复**：

```lua
State{
    name = "myattack",
    onenter = function(inst)
        inst._mycb = function(inst, data) ... end
        inst:ListenForEvent("attacked", inst._mycb)
    end,
    onexit = function(inst)
        if inst._mycb then
            inst:RemoveEventCallback("attacked", inst._mycb)
            inst._mycb = nil
        end
    end,
}
```

#### 陷阱 6：往**已销毁**的 source 推事件

**症状**：报错 "attempt to index nil value (field 'event_listeners')"  
**原因**：在异步回调里 push 一个早已销毁的实体——比如 `inst:DoTaskInTime(5, function() inst:PushEvent(...) end)`，5 秒前 inst 被销毁了。  
**修复**：**push 前用 `IsValid` 判**：

```lua
inst:DoTaskInTime(5, function(inst)
    if inst:IsValid() then
        inst:PushEvent("delayed", { ... })
    end
end)
```

> **更优雅**：用 `inst:PushEventInTime(5, "delayed", { ... })`——它内部走 `self.pendingtasks`，实体销毁时**自动取消**。

#### 设计经验三条

**经验 ①：事件名加 mod 前缀**

`"attacked"`、`"death"`、`"healthdelta"` 是 Klei 约定的全局事件。**自定义事件**永远加 mod 前缀：`"mymod_xxx"`——避免和其他 mod 撞名导致**两边都被错误触发**。

**经验 ②：把"业务"和"反应"分开放**

错误结构：

```lua
-- 业务直接侵入回调
inst:ListenForEvent("attacked", function(inst, data)
    inst.components.combat:SetTarget(data.attacker)
    inst.AnimState:PlayAnimation("hurt")
    inst.SoundEmitter:PlaySound("dontstarve/pig/grunt")
    if data.damage > 50 then
        inst.components.health:Kill()
    end
end)
```

正确结构：**业务交给组件，回调只做"决策分派"**：

```lua
local function OnAttacked(inst, data)
    if inst.components.combat then
        inst.components.combat:SetTarget(data.attacker)
    end
end

inst:ListenForEvent("attacked", OnAttacked)
```

剩下"播动画 + 音效"那部分应该**走 stategraph**——SG 监听 `"attacked"` 事件并切到 hit state。**事件回调里放业务、SG 里放表现**——和 7.7.9 设计经验 ① 是同一条思路：分层。

**经验 ③：跨实体监听用"长寿者监听短寿者"**

监听的语义是"我关心你"——**让短寿者推、长寿者听**：

- ✅ HUD widget 监听 ThePlayer
- ✅ ThePlayer 监听 boss
- ❌ boss 监听 ThePlayer（boss 死了，监听才会被自动清理，玩家还在玩——push 完全无效，徒增负担）

**反过来 source 销毁时也清理 listener，但效率不如直接结构化**——双向表的清理是 O(N) 全表扫描，频繁触发会拖慢销毁帧。

---

### 8.1.9 小结

**三件套 API 一句话总结**：**`PushEvent` 喊、`ListenForEvent` 听、`RemoveEventCallback` 不听**——靠**事件名字符串**解耦发送方和接收方。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 推送事件给自己 | `inst:PushEvent("myevent", { foo = 1 })` |
| 推送世界事件 | `TheWorld:PushEvent("myevent", { foo = 1 })` |
| 监听自己的事件 | `inst:ListenForEvent("myevent", fn)` |
| 监听别人的事件 | `inst:ListenForEvent("myevent", fn, other_inst)` |
| 立刻同步推（含 SG）| `inst:PushEventImmediate("myevent", data)` |
| 延时推 | `inst:PushEventInTime(5, "myevent", data)` |
| 解除一次注册 | `inst:RemoveEventCallback("myevent", fn, source)` |
| 解除全部 | `inst:RemoveAllEventCallbacks()` |

**6 个陷阱排雷顺序**

1. 事件名拼写 / 大小写 → 永远小写、永远字符串字面量
2. 匿名 function → 永远具名 / 存字段
3. listener 改 data 字段 → 只读
4. 重复 ListenForEvent → 注册前先 Remove
5. SG state 注册没解除 → onexit 配 onexit
6. 往已销毁 source 推 → 用 `IsValid` 或 `PushEventInTime`

**3 条设计经验**

- ① **加 mod 前缀**避免事件名冲突
- ② **业务 vs 表现分层**：lua callback 做业务、SG 做表现
- ③ **长寿者听短寿者**：HUD 听玩家、玩家听 boss

> **下一节预告**：8.2 节我们把"事件作用域"讲清楚——**实体级 `inst:PushEvent`** 和 **世界级 `TheWorld:PushEvent`** 的差异、`TheGlobalInstance` 这种"全局监听桥梁"、以及为什么有些事件**两边都要推**（健康例子：`health.lua` 的 `Kill` 同时推 `entity_death`（世界）和 `death`（自身））。读完 8.2，你将能清晰判断 mod 里**新加的事件应该推到哪一级**——这是 mod 体系架构的关键设计决策。

## 8.2 事件的作用域：实体事件 vs 世界事件

### 本节导读

8.1 节我们把 `PushEvent / ListenForEvent / RemoveEventCallback` 三件套讲透了——但**所有例子的 source 都是某个具体实体**（猪人、玩家、目标）。新手第一次写 mod 想做"**玩家死亡时全图猪人都跳一下**"或者"**白天到了所有花朵都开放**"——立刻就懵了：**我去监听谁的事件？**

答案是：**`TheWorld`**。饥荒里的事件分两个作用域：

> **实体事件**：发送方就是普通实体——`猪人:PushEvent("death")`、`玩家:PushEvent("hungerdelta")`。你**必须先拿到那个实体的引用**才能监听——8.1 讲的就是这一类。
> **世界事件**：发送方是 `TheWorld`（一个全局唯一的特殊实体）——`TheWorld:PushEvent("ms_playerjoined")`、`TheWorld:PushEvent("phasechanged", "night")`。**任何 mod、任何代码都能直接拿到 TheWorld 的引用**，所以这是事件系统的"广播频道"。

这一节我们把作用域讲清楚——**什么时候推到实体上、什么时候推到世界上**——这是 mod **架构层面**的关键决策。

> **新手**从 8.2.1-8.2.3 起步——理解 `TheWorld` 是什么、和 `TheGlobalInstance` 的差异、典型世界事件速查；**进阶读者**继续看 8.2.4-8.2.6，深入 `ms_` 前缀约定、`playerentered/playeractivated/ms_playerjoined` 这三个**长得很像但触发时机完全不同**的事件、以及 Klei 的"双推模式"（同一件事既推实体又推世界）；**老手**跳到 8.2.7-8.2.8，看 mod 自定义"全局事件总线"的设计模式，以及 6 个最容易踩的坑——尤其是**世界事件名冲突**和**客户端发不出 ms_ 事件**这两个翻车场景。

---

### 8.2.1 快速入门：从一次"玩家上线"看世界事件

#### 第一步：在游戏里观察一次完整的"玩家加入"事件链

启动一个联机游戏，**第二个玩家加入**——

**你（主机）看到的**：
1. 屏幕中央弹一行字："Wilson 加入了游戏"
2. 地图上出现一个新的玩家头像
3. 季节、天气、月相**对新玩家立刻生效**
4. 服务器开始为这个玩家计算饥饿、健康、理智 tick
5. 临近的怪物 spawner 知道"玩家+1"——刷怪概率上调

**引擎在背后做了什么？**

新玩家 prefab 创建后，`scripts/prefabs/player_common.lua` 在它的"激活"流程里**连续推了 4 个世界事件**：

```846:853:scripts/prefabs/player_common.lua
    -- "playerentered" is available on both server and client.
    -- - On clients, this is pushed whenever a player entity is added
    --   locally because it has come into range of your network view.
    -- - On servers, this message is identical to "ms_playerjoined", since
    --   players are always in network view range once they are connected.
    TheWorld:PushEvent("playerentered", inst)
    if TheWorld.ismastersim then
        TheWorld:PushEvent("ms_playerjoined", inst)
```

**整条链路里世界事件起的作用**：**让所有"关心新玩家"的系统不需要相互认识也能联动**——
- 季节系统监听 `ms_playerjoined` → 给新玩家立即同步当前季节
- birdspawner 监听 `ms_playerjoined` → 把新玩家加入 `_activeplayers` 列表
- bearger spawner 监听 `ms_playerjoined` → 给新玩家计算自己的"狗熊倒计时"

**典型例子**——`scripts/components/beargerspawner.lua`：

```438:442:scripts/components/beargerspawner.lua
self.inst:ListenForEvent("ms_playerjoined", OnPlayerJoined, TheWorld)
self.inst:ListenForEvent("ms_playerleft", OnPlayerLeft, TheWorld)
self.inst:ListenForEvent("seasontick", OnSeasonTick, TheWorld)
self.inst:ListenForEvent("beargerremoved", OnHasslerRemoved, TheWorld)
self.inst:ListenForEvent("beargerkilled", OnHasslerKilled, TheWorld)
```

**注意第 3 个参数 `TheWorld`** ——这是 8.1.3 讲的**跨实体监听**：beargerspawner 这个组件挂在 TheWorld 自己身上，但它监听的是 `TheWorld` 的事件（其实它就在监听**自己的宿主**）。**世界事件的标准注册写法就是这样**。

#### 第二步：`TheWorld` 是什么？

打开 `scripts/main.lua` 第 328 行：

```lua
TheWorld = nil
```

`TheWorld` 在游戏启动时是 nil——**直到世界 prefab 被实例化**。看 `scripts/prefabs/world.lua` 第 421 行：

```413:425:scripts/prefabs/world.lua
    local function fn()
        local inst = CreateEntity()

		if TheWorld ~= nil then
			print("You cannot spawn multiple worlds!")
			return nil
		end

        TheWorld = inst
        inst.net = nil
        inst.shard = nil

        inst.ismastersim = TheNet:GetIsMasterSimulation()
```

**关键观察**：
- **`TheWorld` 是一个 EntityScript**——和 prefab 里的 `inst` 一样有 `event_listeners` 字段
- **全局唯一**——代码里有显式断言"不能 spawn 两个 world"
- **既存在于服务端、也存在于客户端**——但 `TheWorld.ismastersim` 区分这两边
- **它本身就是一堆"世界级组件"的宿主**：season、worldstate、playerspawner、shard、weather、worldcharacterstate……

> **核心结论**：`TheWorld` 是个**特殊的"系统级实体"**——它没有动画、没有 transform、不在地图上有"位置"，但**它有完整的事件系统**。`TheWorld:PushEvent` 和 `inst:PushEvent` 用的是**同一份代码** —— `EntityScript:PushEvent_Internal`（见 8.1.5）。

#### 第三步：用控制台亲眼看 TheWorld 的事件结构

```lua
print(TheWorld)                        -- => world[xxxxx]
print(TheWorld:HasTag("CLASSIFIED"))   -- => true（世界实体不是普通实体）
print(TheWorld.ismastersim)            -- => true (主机) / false (客户端)
print(TheWorld.net)                    -- => 联机版的"网络分身"实体（详见 8.2.4）

-- 看一眼有多少 listener
local count = 0
for ev, _ in pairs(TheWorld.event_listeners or {}) do
    count = count + 1
end
print("registered events:", count)      -- => 通常 30~80
```

> **新手到这里只需记住**：**只要事件影响"全图所有实体"或"游戏整体状态"，就推到 `TheWorld` 上**——而**只影响"这一个实体的内部状态"**，就推到 `inst` 上。

---

### 8.2.2 快速入门：实体事件 vs 世界事件 vs TheGlobalInstance

#### 第一步：三种"全局实体"的对照

饥荒里有**三个**容易混淆的"全局对象"：

| 名字 | 类型 | 主要用途 | 是否能 push 事件 |
| --- | --- | --- | --- |
| `TheWorld` | EntityScript（世界 prefab 的 inst） | **游戏内**的所有"系统级"事件——天气、季节、玩家加入、boss 重生……跟着 world 一起销毁 | **是**——99% 世界事件走它 |
| `TheGlobalInstance` | EntityScript（无 prefab 的特殊实体） | **跨世界**的全局——主菜单时也存在；ShadowManager / RoadManager / EnvelopeManager 等子系统宿主 | **理论上能**，但 Klei 几乎不用它推业务事件 |
| `TheNet`、`TheSim`、`TheShard` 等 | 引擎绑定的 C++ 单例 | 网络、模拟、分片——**不是 EntityScript**，没有事件系统 | **不能**（直接用方法调用） |

> 看 `scripts/main.lua:441`：

```441:445:scripts/main.lua
    TheGlobalInstance = CreateEntity("TheGlobalInstance")
    TheGlobalInstance.entity:AddTransform()
    TheGlobalInstance.entity:SetCanSleep(false)
    TheGlobalInstance.persists = false
    TheGlobalInstance:AddTag("CLASSIFIED")
```

`TheGlobalInstance` 在**游戏进程启动时**就被创建——比 `TheWorld` 更早。但它的设计目的是"**给一些必须存在但不属于任何 world 的子系统提供宿主**"——比如阴影管理器、postprocess、roadmanager。**业务事件用 `TheWorld` 即可**——除非你的事件**真的要在主菜单/loading 阶段就触发**（罕见）。

#### 第二步：作用域决策矩阵

写 mod 时遇到事件，**怎么决定推到哪？**

| 影响范围 | 推到哪 | 例子 |
| --- | --- | --- |
| 一个实体内部 | `inst:PushEvent` | `health.lua` 推 `"healthdelta"`、`combat.lua` 推 `"attacked"` |
| 一对父子实体（如装备 ↔ 持有者） | 推到**关心方** | `inst:PushEvent("equipped", { owner = ... })` 推到装备自己上 |
| 全图所有玩家 | `TheWorld:PushEvent` | `"phasechanged"`、`"ms_playerjoined"`、`"cycleschanged"` |
| 主机端业务（仅服务端走） | `TheWorld:PushEvent("ms_xxx", ...)`（**ms_ 前缀**） | `"ms_setseason"`、`"ms_lightning"`、`"ms_newday"` |
| 客户端 UI 同步 | 通常 `inst:PushEvent` 到 `ThePlayer` 自己 | `"hungerdelta"` 给 hunger UI |
| 跨世界（主菜单/shard 切换） | `TheGlobalInstance:PushEvent` | 罕见，Klei 几乎不用 |

#### 第三步：不知道选哪？记住三个反问

新手判断时，问自己三个问题：

1. **这件事会不会影响多个实体？** 是 → 倾向世界事件
2. **listener 是否预先就能拿到 source 的引用？** 不能（比如 listener 是个还没出生的 prefab）→ 必须世界事件
3. **是不是仅服务端逻辑？** 是 → 用 `ms_` 前缀的世界事件（8.2.4 详解）

> **设计哲学**：实体事件**强耦合于实体身份**——push 和 listen 必须知道彼此；世界事件**通过 TheWorld 解耦**——push 方只对 TheWorld 喊一声、谁听谁决定。**和操作系统的"信号 vs 共享内存"很像**。

---

### 8.2.3 快速入门：典型场景速查

#### 场景 1：玩家加入服务器，给所有人发欢迎

```lua
-- 在 modmain.lua / 某 component 里
TheWorld:ListenForEvent("ms_playerjoined", function(world, player)
    if not player.HUD then return end
    player.HUD.controls:ShowToast("welcome!", 5)
end)

-- ms_playerjoined 只在服务端推（见 8.2.4）
-- 监听者通常是 component / prefab fn / mod 顶层
```

但更常见的写法是**让"关心的组件"自己挂载到 TheWorld 并 listen**——见 8.2.1 第一步 beargerspawner 的范例。

#### 场景 2：季节变化时让所有植物刷新

```lua
TheWorld:ListenForEvent("seasontick", function(world, data)
    -- data 包含 progress, season, elapseddaysinseason 等
    print("当前季节：", data.season, "进度：", data.progress)
end)
```

**`seasontick`** 是世界事件——由 `seasons` 组件每帧推一次（worldstate 变化时）。

#### 场景 3：监听任意实体死亡（不知道是哪只）

```lua
TheWorld:ListenForEvent("entity_death", function(world, data)
    if data.inst:HasTag("monster") then
        print("一只怪物死了：", data.inst.prefab)
    end
end)
```

注意 `entity_death` 的 data 里**有 `inst` 字段**——这是 8.1.6 讲的"双推模式"的产物——同一件事**既推到 inst 自己（"death"）又推到 TheWorld（"entity_death"）**——前者只有"凶手"和"原因"，后者额外把"是谁死了"塞进 data 让世界级监听者能识别。

#### 场景 4：白天/黄昏/夜晚切换

```lua
TheWorld:ListenForEvent("phasechanged", function(world, phase)
    print("现在是：", phase)  -- "day" / "dusk" / "night"
end)
```

**注意**：这里 push 时 data 是个**字符串而不是 table**——`TheWorld:PushEvent("phasechanged", "night")`。Klei 的代码里**事件 data 不一定是 table**——`PushEvent` 第二个参数是**任意值**，listener fn 收到什么取决于 push 时传什么。

> 这种"字符串 data"是早年 Klei 的代码遗留——新写的事件**统一推 table**比较好维护（见 8.2.8 设计经验 ②）。

#### 场景 5：自定义"地震"全局事件

```lua
-- 推送
function MyEarthquake:Trigger()
    TheWorld:PushEvent("mymod_earthquake", {
        magnitude = 5,
        epicenter = self.inst:GetPosition(),
    })
end

-- 监听（任何想响应的 mod 都可以）
TheWorld:ListenForEvent("mymod_earthquake", function(world, data)
    if data.magnitude > 3 then
        ShakeAllScreens(data.magnitude)
    end
end)
```

**关键**：自定义世界事件**永远加 mod 前缀**（`mymod_xxx`）——避免和 Klei 内置或别的 mod 冲突。

#### 场景 6：触发避难所自动生成（仅主机端）

```lua
-- 服务端
if TheWorld.ismastersim then
    TheWorld:PushEvent("ms_buildshelter", { x = ..., y = ..., z = ... })
end

-- ms_ 前缀表明"主机端事件"——8.2.4 节专讲
```

---

### 8.2.4 进阶：客户端 / 服务端的 `ms_` 前缀约定

#### 第一步：`ms_` 是什么？

观察 Klei 的代码：

| 事件名 | 推送时机 | 谁能监听 |
| --- | --- | --- |
| `playerentered` | 玩家 prefab **激活时**（主机和客户端都推） | 服务端 + 客户端 |
| `ms_playerjoined` | **仅主机端**——玩家成功连接时 | 仅服务端 |
| `playerexited` | 玩家 prefab **去激活时**（两端都推） | 服务端 + 客户端 |
| `ms_playerleft` | **仅主机端**——玩家断线时 | 仅服务端 |
| `entity_death` | health.Kill 里推（**仅主机端**） | 仅服务端 |
| `phasechanged` | clock 组件推（两端都同步推） | 两端都收 |

**`ms_` 前缀**是 Klei 内部约定——意为 **"master sim only"**——**只在主机端推**的事件。

回看 `player_common.lua` 第 851-853 行：

```851:853:scripts/prefabs/player_common.lua
    TheWorld:PushEvent("playerentered", inst)
    if TheWorld.ismastersim then
        TheWorld:PushEvent("ms_playerjoined", inst)
```

**模式很清晰**：
- 第一行 `playerentered` ——两端都推，**因为客户端也要知道"有玩家激活了，更新 UI"**
- 第二行 `ms_playerjoined` ——只主机端推，**因为业务逻辑（spawn 怪物、计算饥饿）只在主机端跑**

#### 第二步：`ms_` 事件的客户端监听者会怎样？

**客户端永远收不到 `ms_xxx` 事件**——因为客户端的 `TheWorld` 自己就**没人推 `ms_xxx`**。

**客户端代码里写 `TheWorld:ListenForEvent("ms_playerjoined", ...)` ≠ 报错**——只是**永远不会触发**。

**这是一个常见 bug 来源**：在 modmain 顶层（两端都跑）写监听 `ms_playerjoined`——主机端正常工作，**客户端这条 listener 注册了但永远空跑**。**通常没坏处**，但你以为有 listener 实际没反应——浪费调试时间。

**更安全的写法**：

```lua
if TheWorld.ismastersim then
    TheWorld:ListenForEvent("ms_playerjoined", OnPlayerJoined)
end
```

或者直接用 `playerentered` ——这个两端都推，客户端能收到。

#### 第三步：如何让自定义事件支持"仅主机端"语义？

**做法 1**：**主机端推 + ms_ 前缀**——和 Klei 一致：

```lua
if TheWorld.ismastersim then
    TheWorld:PushEvent("ms_mymod_someaction", data)
end
```

**做法 2**：**普通事件，但 listener 自己判 `TheWorld.ismastersim`**：

```lua
TheWorld:PushEvent("mymod_someaction", data)  -- 两端都推

TheWorld:ListenForEvent("mymod_someaction", function(world, data)
    if not TheWorld.ismastersim then return end
    -- 业务逻辑
end)
```

**两者都对**。**做法 1 更明确**（事件名一眼看出 server-only）；**做法 2 更灵活**（同一事件名，主机和客户端各做各的）。

> Klei 自己**普遍用做法 1**——`ms_xxx` 事件名占了世界事件总数的大约 30%。

#### 第四步：`TheWorld.net` —— 客户端唯一的"对应物"

```424:426:scripts/prefabs/world.lua
        TheWorld = inst
        inst.net = nil
        inst.shard = nil
```

`TheWorld.net` 是**世界的客户端复制实体**——它有自己的网络变量（`net_*`），允许主机端**主动通知**客户端某些状态。

**典型用例**：主机端事件触发了某个 net 变量更新 → 客户端这边的 `inst.net` 上的 net 变量监听被触发 → 客户端代码响应。

**这不是事件系统，是 networking 的另一条路径**——**8.5 节会拆开它和事件系统的协作关系**。

#### 第五步：用控制台验证 ms_ 前缀的两端差异

主机控制台：

```lua
TheWorld:ListenForEvent("ms_test_event", function(_, data) print("master got:", data) end)
TheWorld:PushEvent("ms_test_event", { foo = 1 })
-- => master got: table 0xXXX
```

客户端控制台（**联机模式**）：

```lua
TheWorld:ListenForEvent("ms_test_event", function(_, data) print("client got:", data) end)
TheWorld:PushEvent("ms_test_event", { foo = 1 })
-- => client got: table 0xXXX  （客户端自己 push、自己 listen，照样触发）
```

**注意**：客户端能**给自己的 TheWorld push 事件**——只是**主机的 push 不会自动同步过来**。**`ms_xxx` 不是个网络协议，只是个命名约定** —— **不会自动跨节点广播**。

---

### 8.2.5 进阶：`playerentered` / `playeractivated` / `ms_playerjoined` 三件套差异

#### 第一步：三个事件的精确定义

`scripts/prefabs/player_common.lua` 里**不同时机**推了 3 个事件：

```777:853:scripts/prefabs/player_common.lua
    TheWorld:PushEvent("playerdeactivated", inst)
    ...
    TheWorld:PushEvent("playeractivated", inst)
    ...
    TheWorld:PushEvent("playerentered", inst)
    if TheWorld.ismastersim then
        TheWorld:PushEvent("ms_playerjoined", inst)
```

**4 个事件的语义对比**：

| 事件 | 推送时机 | 推送侧 | 一个玩家的生涯里推几次 |
| --- | --- | --- | --- |
| `playerentered` | 玩家 prefab **激活进入**——客户端也会在玩家"进入视野"时推；主机就是连接时 | 两端都推 | 多次（每次进入视野） |
| `playerexited` | 玩家 prefab **去激活离开**——客户端是出视野，主机是断线 | 两端都推 | 多次 |
| `playeractivated` / `playerdeactivated` | 玩家**激活/去激活到状态变化**——比 entered/exited 更频繁 | 两端都推 | 很多次 |
| `ms_playerjoined` | 玩家**真正加入服务器**——只一次（每个 session） | 仅主机 | 1 次 |
| `ms_playerleft` | 玩家**永久离开服务器** | 仅主机 | 1 次 |
| `ms_playerspawn` | 玩家 spawn 时（包括复活） | 仅主机 | 多次（每次复活） |
| `ms_newplayerspawned` | **第一次** spawn（新玩家创建） | 仅主机 | 1 次 |
| `ms_newplayercharacterspawned` | 玩家选完角色第一次 spawn | 仅主机 | 1 次 |

#### 第二步：选错事件的典型 bug

**bug 1**：监听 `ms_playerjoined` 想给玩家加道具——结果**复活时不再触发**。  
**修复**：监听 `ms_playerspawn`（包括复活）。

**bug 2**：监听 `playerentered` 想给玩家初始化数据——结果**进入视野**时反复初始化。  
**修复**：用 `ms_playerjoined` ——保证一个 session 只一次。

**bug 3**：监听 `ms_newplayerspawned` 想给老玩家也加属性——结果**只新玩家有效，老玩家没有**。  
**修复**：用 `ms_playerspawn`（每次 spawn 都触发）或 `ms_playerjoined`（每次连接都触发）。

#### 第三步：实战 ——给所有玩家监听其饥饿值

```lua
-- 写在某个 component 里
local function OnPlayerHunger(player, data)
    if data.newpercent < 0.2 then
        -- 玩家很饿了，触发某个 mod 业务
    end
end

local function StartListening(player)
    player:ListenForEvent("hungerdelta", OnPlayerHunger)
end

-- 给现有玩家挂监听
for _, player in ipairs(AllPlayers) do
    StartListening(player)
end

-- 给后来加入的玩家也挂监听
TheWorld:ListenForEvent("ms_playerjoined", function(world, player)
    StartListening(player)
end)
```

**关键**：**两套并行**——遍历 `AllPlayers` 处理已存在的玩家，监听 `ms_playerjoined` 处理后来的玩家。**这是 mod 标准模式**——见 `beargerspawner.lua:434-438` 同款写法。

#### 第四步：玩家断线时主动清理

```lua
TheWorld:ListenForEvent("ms_playerleft", function(world, player)
    player:RemoveEventCallback("hungerdelta", OnPlayerHunger)
    -- 清理你给玩家挂的所有 listener / 任务 / 数据
end)
```

> **注意**：**通常 8.1.6 讲的"实体销毁自动 RemoveAllEventCallbacks"会处理这个**——但 player prefab 在断线时**未必立刻销毁**（可能等 reconnect 5 分钟）。**保险起见，自己手动清理**。

---

### 8.2.6 进阶：双推模式 —— 实体级 + 世界级（health.Kill 的设计）

#### 第一步：再看一次 `health.lua:589-590`

```589:590:scripts/components/health.lua
        TheWorld:PushEvent("entity_death", { inst = self.inst, cause = cause, afflicter = afflicter, corpsing = is_corpsing })
        self.inst:PushEvent("death", { cause = cause, afflicter = afflicter, corpsing = is_corpsing })
```

**同一件事推了两次**：
- `TheWorld:PushEvent("entity_death", { inst = self.inst, ... })` —— 世界级，**data 含 inst 字段** 让监听者知道是谁死了
- `self.inst:PushEvent("death", { ... })` —— 实体级，**不带 inst 字段**（监听者自然知道 source 就是死者）

**为什么要双推？**

#### 第二步：双推的 3 个原因

**原因 1：消费者结构不同**

- 监听**特定实体**的 listener（比如 prefab 自己的 OnDeath、它的 SG、它的 brain）—— 用实体事件 `"death"` 即可
- 监听**任意实体死亡**的 listener（比如成就系统、boss 监控、克劳斯日记）—— **必须**用世界事件 `"entity_death"`

**原因 2：避免"广播负担"**

如果只推世界事件 `"entity_death"`：
- 死一只 butterfly 也要广播——所有"成就 listener"被打扰一次
- 死一个 boss 也广播——同样的 listener 数量

如果只推实体事件 `"death"`：
- "成就 listener" 必须**给每一个还活着的实体注册一遍**——千万级别的实体——彻底崩溃

**双推 = 各取所需**：实体监听者只听自己需要的，世界监听者一站式接收所有。

**原因 3：data 结构不同**

实体事件的 source 隐含已知（`fn(inst, data)` 第一个参数就是 source）——data 里**不需要重复 inst**；世界事件的 source 是 TheWorld，监听者**必须从 data 里拿到**真正死的是谁——所以多一个 `inst = self.inst`。

#### 第三步：你的 mod 该不该双推？

**判断标准**：

> **存在多个 listener 类型——既有"特定实体监听"也有"全局监听"——就双推**。

**例 1**：mod 自定义"暴击"事件——猪人挨打时有几率触发暴击。
- 只有"被打的实体"关心暴击 → **只推 inst**：`inst:PushEvent("mymod_critical", { damage = ... })`
- 不需要双推

**例 2**：mod 自定义"宝藏被发现"事件——挖到宝藏后通知地图、成就、UI。
- "宝藏自身"想知道（播音效）+ 全局系统想知道（更新地图标记）→ **双推**：

```lua
inst:PushEvent("mymod_treasurefound", { finder = doer })
TheWorld:PushEvent("mymod_treasurefound", { inst = inst, finder = doer })
```

**例 3**：mod 自定义"季节切换到末世"——影响所有实体。
- 没有"特定实体"概念 → **只推 TheWorld**：

```lua
TheWorld:PushEvent("mymod_apocalypse", { stage = 1 })
```

#### 第四步：双推的命名约定

**Klei 风格**：
- 实体事件：动作名/状态名 → `"death"`、`"attacked"`、`"equipped"`
- 世界事件：`entity_动作名` 或 `ms_动作名` → `"entity_death"`、`"ms_playerjoined"`

**mod 风格建议**：
- 实体事件：`"mymod_xxx"`
- 世界事件：`"mymod_world_xxx"` 或 `"mymod_ms_xxx"`

**避免**：双推用同一个事件名 → 听 `inst:PushEvent("mymod_xxx")` 的 listener 和听 `TheWorld:PushEvent("mymod_xxx")` 的 listener 不会互相影响（事件名空间是按 source 分的——见 8.1.4 结构图），但**调试时容易混淆**。

---

### 8.2.7 老手进阶：自己定义"全局事件总线"的设计模式

#### 第一步：什么时候**不**用 TheWorld？

**95% 全局事件用 TheWorld 就够了**。但有 3 种边界情况：

1. **跨 shard 通信**（地表 ↔ 洞穴）：`TheWorld` 在两个 shard 各有一份——push 到 A shard 的 TheWorld 不会被 B shard 听到——需要**用 RPC 桥接**或 `ms_sendshardrpc`
2. **跨 session（保留到下一局）**：`TheWorld` 和它的事件**这一局结束就消失**——需要持久化你得用 `TheWorld:GetPersistData()` 钩子
3. **mod 间约定的"接口事件"**（多个 mod 都关心同一个标准事件）—— TheWorld 可用，但**多个 mod 用同一个事件名**会有命名冲突——**用一个全局桥实体**会更整洁

#### 第二步：自定义 "Bus" 实体

```lua
-- 在 modmain.lua 顶层
GLOBAL.MyModBus = nil

AddSimPostInit(function()
    if GLOBAL.MyModBus == nil then
        GLOBAL.MyModBus = GLOBAL.CreateEntity("MyModBus")
        GLOBAL.MyModBus.entity:AddTransform()
        GLOBAL.MyModBus.entity:SetCanSleep(false)
        GLOBAL.MyModBus.persists = false
        GLOBAL.MyModBus:AddTag("CLASSIFIED")
    end
end)
```

**好处**：
- 你的 mod 定义的事件**全部走自己的 Bus**——和 TheWorld 隔离
- 别的 mod 只要 `if MyModBus then MyModBus:ListenForEvent("xxx", ...) end` 就能对接
- 不污染 TheWorld 的事件名空间——避免和未来 Klei 内置事件撞名

**缺点**：
- 多写几行代码
- mod 卸载时记得 `MyModBus:Remove()`——否则下一局会有"上一局的 Bus 残骸"

#### 第三步：用 `BindGlobalCallable` 模式做"事件代理"

更高级的封装：

```lua
-- mybus.lua
local Bus = Class(function(self)
    self.inst = CreateEntity("MyBus")
    self.inst.entity:AddTransform()
    self.inst.persists = false
end)

function Bus:On(event, fn)
    self.inst:ListenForEvent(event, fn)
end

function Bus:Emit(event, data)
    self.inst:PushEvent(event, data)
end

function Bus:Off(event, fn)
    self.inst:RemoveEventCallback(event, fn)
end

return Bus
```

```lua
-- 使用
local MyBus = require("mybus")()
MyBus:On("treasurefound", function(_, data) ... end)
MyBus:Emit("treasurefound", { who = doer })
```

> **设计哲学**：这是把 EntityScript 的事件系统**包装成一个轻量级的 EventEmitter**——和 Node.js 的 EventEmitter 同款思路。**适合大型 mod 的内部事件**——业务模块之间解耦。

#### 第四步：跨 shard 事件桥（高阶）

如果你需要"地表事件触发洞穴的 mob 行为"——**TheWorld 不能跨 shard**。**用 ShardRPC**：

```lua
-- 主机端 ShardRPC 注册
AddShardModRPCHandler("mymod", "TreasureFoundCrossShard", function(shardid, data)
    TheWorld:PushEvent("mymod_treasurefound_xshard", { from_shard = shardid, data = data })
end)

-- A shard 触发
SendModRPCToShard(GetShardModRPC("mymod", "TreasureFoundCrossShard"), nil, data)
-- nil = 全部 shard
```

**两步**：本 shard `PushEvent` → ShardRPC 同步给其他 shard → 对方 shard 在 RPC handler 里 `PushEvent` 进自己的 TheWorld。

**这是 ShardRPC 的标准用法**—— 8.5 节会更详细讲。

---

### 8.2.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：客户端监听 `ms_xxx` 事件，永远不触发

**症状**：modmain 顶层写了 `TheWorld:ListenForEvent("ms_playerjoined", ...)`——客户端永远没反应。  
**原因**：`ms_xxx` 事件**只在主机端推**——客户端 TheWorld 上没人推。  
**修复**：

```lua
if TheWorld.ismastersim then
    TheWorld:ListenForEvent("ms_playerjoined", OnPlayerJoined)
end
```

或者改用 `playerentered`（两端都推，但触发时机不同——见 8.2.5 第二步）。

#### 陷阱 2：在 `modmain.lua` 顶层引用 `TheWorld`

**症状**：报 "attempt to index nil value (TheWorld)" —— mod 启动崩溃。  
**原因**：`modmain.lua` 在游戏**启动早期**加载——这时 `TheWorld == nil`。  
**修复**：用 `AddSimPostInit` / `AddPlayerPostInit` 等 PostInit 钩子——**在 TheWorld 已存在时才执行**：

```lua
AddSimPostInit(function()
    TheWorld:ListenForEvent("ms_playerjoined", ...)
end)
```

#### 陷阱 3：自定义世界事件没加 mod 前缀，撞了别的 mod

**症状**：mod A 推 `"buildshelter"`、mod B 也推 `"buildshelter"` —— 两边的 listener 都被对方触发，行为错乱。  
**原因**：世界事件名是**全局共享的字符串空间**——没有命名空间。  
**修复**：永远 `mymod_xxx`：

```lua
TheWorld:PushEvent("mymod_buildshelter", { ... })
```

#### 陷阱 4：以为 `TheWorld:PushEvent` 会跨 shard

**症状**：地表 push 一个事件，洞穴的 listener 等不到。  
**原因**：每个 shard 有**自己的 TheWorld** ——push 不会自动同步。  
**修复**：用 `SendModRPCToShard` + ShardRPC handler 显式桥接（见 8.2.7 第四步）。

#### 陷阱 5：在 listener 里改了 data 字段，其他 listener 看错

和 8.1.8 陷阱 3 同款。世界事件**多 listener 是常态**——比实体事件更容易撞——更要严守"data 只读"。

**典型反例**：

```lua
TheWorld:ListenForEvent("entity_death", function(_, data)
    data.cause = "by_my_mod"  -- 错！会污染下一个 listener 看到的 cause
end)
```

#### 陷阱 6：在世界事件 listener 里访问还没加载的组件

**症状**：listener 里 `world.components.season:GetSeason()` 报 nil。  
**原因**：世界事件可能在**世界加载早期**就触发——这时候季节组件还没装上。  
**修复**：判 nil：

```lua
TheWorld:ListenForEvent("phasechanged", function(world, phase)
    local season = world.components.season and world.components.season:GetSeason()
    if season == nil then return end  -- 还没准备好
    -- 业务
end)
```

或者**等更晚的事件再注册**：用 `worldgenerated` / `worldloaded` 之类的"准备好"事件做触发器。

#### 设计经验三条

**经验 ①：世界事件**永远**带 mod 前缀**

`"mymod_xxx"` 是底线。**两个 mod 都用 `"buildshelter"` 时撞名 → 互相把对方的事件监听了 → 谁也不知道为什么 mod 行为异常**。这是 mod 调试时**最难定位的 bug 类型**之一。

**经验 ②：data 用 table、字段语义清晰**

- ❌ `TheWorld:PushEvent("phasechanged", "night")` —— 字符串 data
- ✅ `TheWorld:PushEvent("mymod_phasechanged", { phase = "night", lastphase = "dusk" })`

**理由**：
1. **可扩展**——以后想加字段不破坏旧 listener
2. **自文档化**——`data.phase` 一眼明白比 `arg2` 强一万倍
3. **调试友好**——`print(data)` 能看到所有字段

**经验 ③：把"业务监听"集中到一个 component 里**

错误结构：modmain 顶层散布 30 行 `TheWorld:ListenForEvent` —— mod 维护时找不到从哪监听。

正确结构：写一个 `mymod_world_listener` component，挂到 `TheWorld` 上，所有世界事件监听都集中在它的 `OnInit` 里：

```lua
-- scripts/components/mymod_world_listener.lua
local MyModWorldListener = Class(function(self, inst)
    self.inst = inst
    inst:ListenForEvent("ms_playerjoined", function(_, p) self:OnPlayerJoined(p) end)
    inst:ListenForEvent("phasechanged",   function(_, p) self:OnPhase(p)        end)
    inst:ListenForEvent("entity_death",   function(_, d) self:OnEntityDeath(d)  end)
    -- ... 集中
end)

-- modmain.lua
AddPrefabPostInit("world", function(inst)
    if inst.ismastersim then
        inst:AddComponent("mymod_world_listener")
    end
end)
```

**好处**：
- mod 卸载时**一键** `inst:RemoveComponent("mymod_world_listener")`——所有 listener 自动清理（见 8.1.6 第一步）
- 业务方法在同一文件，跨方法调用方便

---

### 8.2.9 小结

**两种作用域一句话总结**：**实体事件**是"私聊"——你必须知道对方是谁；**世界事件**是"广播"——只要在 `TheWorld` 这个频道上喊一声，谁都能收。

**速查表**

| 想做的事 | 推到哪 |
| --- | --- |
| 通知"我自己"内部某状态变了 | `inst:PushEvent` |
| 通知"全图"某事件发生 | `TheWorld:PushEvent` |
| 仅主机端业务 | `TheWorld:PushEvent("ms_xxx")` |
| 既要单实体响应也要全局响应 | 双推（实体 + 世界） |
| 跨 shard | ShardRPC 桥接 |
| 大型 mod 内部解耦 | 自定义 Bus 实体 |

**6 个陷阱排雷顺序**

1. 客户端听 `ms_xxx` → 加 ismastersim 判
2. modmain 顶层访问 TheWorld → 用 PostInit
3. 自定义事件没加 mod 前缀 → 加！
4. 以为 PushEvent 跨 shard → ShardRPC
5. listener 改 data 字段 → 只读
6. 早期 listener 访问还没加载的组件 → 判 nil 或换更晚的事件

**3 条设计经验**

- ① **mod 前缀**——`mymod_xxx` 是底线
- ② **data 用 table**——可扩展、自文档化
- ③ **业务监听集中到 component**——便于一键清理

> **下一节预告**：8.3 节我们打开 **`WatchWorldState`** ——这是 Klei 给"季节、月相、白天/黑夜"等**枚举型世界状态**专门做的另一套监听 API——和事件系统**长得像但不一样**：worldstate **是值快照**，新 listener 注册时立刻收到当前值；事件**是瞬时触发**，新 listener 注册前的事件错过就错过。读完 8.3，你就知道**写 mod 时什么时候用 ListenForEvent("phasechanged")、什么时候用 WatchWorldState("phase")**——这是非常实用的判断点。

## 8.3 WatchWorldState——监听世界状态变化

### 本节导读

8.1 / 8.2 我们把"瞬时事件"那一面讲透了——`PushEvent` 喊一声、`ListenForEvent` 听一声。但有一类信息**不是瞬时的**——**它是"持续保持的状态"**：

- 现在是**白天 / 黄昏 / 夜晚**——这是一个**值**，不是一次"变化"
- 现在是**春夏秋冬**哪个季节——同样是一个值
- 正在**下雨 / 下雪 / 下酸雨吗**——值
- 第几天了？月相是哪个？玩家睡了多久？……

**用事件系统当然也能传递这些信息**——`phasechanged` / `seasontick` 都是事件——但**有一个致命问题**：**事件是"瞬时通知"，新订阅者错过了就拿不到**。

> 想象你写了一个 mod：背包里装了某物品就要在白天发光。物品被放进背包的瞬间——它**根本不知道现在是不是白天**——除非它去查一个"当前世界状态"。

为了解决这个问题，Klei 在事件系统之上**包了一层"世界状态系统"** —— `TheWorld.components.worldstate`——它做两件事：

1. **维护一份当前世界状态的快照**（季节、月相、天气……约 60 个变量），任何代码任何时候都能 `TheWorld.state.xxx` 读到当前值
2. **提供 `WatchWorldState` API**——只在某个状态**真正变化时**触发回调（不是每帧、不是每个 tick）

这一节把这套机制讲透——**它是事件系统在"持续状态"领域的特化**。

> **新手**从 8.3.1-8.3.3 起步——理解 `worldstate` 的存在意义、`WatchWorldState` 三件套、worldstate 全表速查；**进阶读者**继续看 8.3.4-8.3.6，深入 `start*/stop*` "切换变量"的精妙设计、worldstate 和事件系统的内部协作、`Component:WatchWorldState` vs `EntityScript:WatchWorldState` 的差异；**老手**跳到 8.3.7-8.3.8，看"注册时不会触发"这个新手必坑、初始化技巧，以及 6 个最常踩的陷阱。

---

### 8.3.1 快速入门：从一只"老化的胡子"看 worldstate 是什么

#### 第一步：在游戏里观察"胡子每天长长一点"

威尔逊**每过一天**胡子长一点——3、6、9 天后形态依次变化。**问题是**：威尔逊**怎么知道又过了一天**？

打开 `scripts/components/beard.lua` 第 73 行：

```73:73:scripts/components/beard.lua
            self:WatchWorldState("cycles", OnDayComplete)
```

**就一行！** "盯住 `TheWorld.state.cycles` 这个变量，**每次它变化时调一次 `OnDayComplete`**"。

`cycles` 是世界已经过去的天数——**当 clock 组件每过一天就 +1 时，所有 watch 它的实体的回调都被触发**——胡子组件就在自己的回调里 `length += 1`。

**整条链路**：

```
[1] clock 组件每天结束 PushEvent("cycleschanged", new_cycles)
       ↓
[2] worldstate 组件监听 "cycleschanged" → 调用 SetVariable("cycles", new_cycles)
       ↓
[3] SetVariable 检测到 self.data.cycles 变了
       ↓
[4] 遍历 _watchers["cycles"] 里所有 watcher → 调用每一个 fn
       ↓
[5] 胡子组件的 OnDayComplete 被调用 → 胡子 +1
```

> **核心结论**：**worldstate 是"事件系统的状态化封装"**——内部仍然是事件驱动，但**对外表现为"值的变化通知"**。**新人 mod 开发者 90% 场景应该用 WatchWorldState 而不是 ListenForEvent**——下面解释为什么。

#### 第二步：看 worldstate 组件的源码核心

打开 `scripts/components/worldstate.lua` 第 24-48 行：

```24:48:scripts/components/worldstate.lua
local function SetVariable(var, val, togglename)
    if self.data[var] ~= val and val ~= nil then
        self.data[var] = val

        local watchers = _watchers[var]
        if watchers ~= nil then
            for k, v in pairs(watchers) do
                for i, fn in ipairs(v) do
                    fn[1](fn[2], val)
                end
            end
        end

        if togglename then
            watchers = _watchers[(val and "start" or "stop")..togglename]
            if watchers ~= nil then
                for k, v in pairs(watchers) do
                    for i, fn in ipairs(v) do
                        fn[1](fn[2])
                    end
                end
            end
        end
    end
end
```

**两段关键代码**：

- **第 25 行**：`if self.data[var] ~= val and val ~= nil then` —— **核心去重**——只在**值真的变化**时才触发回调。这就是 worldstate 和 ListenForEvent 的**根本差异**：事件系统**每次 push 都触发**，worldstate **值不变就不触发**。
- **第 37-46 行**：`togglename` 这部分——给某些"布尔型"状态额外提供 `startxxx`/`stopxxx` 两个 watch 名——8.3.4 节专讲。

#### 第三步：再看一眼 worldstate 的存储结构

```12:18:scripts/components/worldstate.lua
assert(inst == TheWorld, "Invalid world")
self.inst = inst
self.data = {}

--Private
local _iscave = inst:HasTag("cave")
local _watchers = {}
```

**两份核心结构**：
- `self.data` ——**世界状态当前值的字典**——全局只有一份，**任何代码都能读**
- `_watchers` ——**watcher 字典**——按 `var → inst → fn 列表` 三级嵌套（和 8.1.4 的事件 listener 表结构很像）

**`world.lua:528`** 还把 `data` 表挂到 `inst.state` 上做快捷访问：

```528:528:scripts/prefabs/world.lua
        inst.state = inst.components.worldstate.data
```

**于是任何代码都能这样读**：

```lua
TheWorld.state.phase             -- "day" / "dusk" / "night"
TheWorld.state.season            -- "autumn" / "winter" / "spring" / "summer"
TheWorld.state.cycles            -- 已过去的天数
TheWorld.state.isfullmoon        -- true / false
TheWorld.state.temperature       -- 当前气温
TheWorld.state.israining         -- 是不是在下雨
```

**就这么简单**。**99% 想读"当前世界状态"的代码，都只需要 `TheWorld.state.xxx` 一行**。

---

### 8.3.2 快速入门：API 三件套

#### 第一步：API 签名

| API | 签名 | 用途 |
| --- | --- | --- |
| `inst:WatchWorldState(var, fn)` | `(string, function) -> nil` | 注册：当 `TheWorld.state[var]` 变化时调 `fn(inst, new_value)` |
| `inst:StopWatchingWorldState(var, fn)` | `(string, function) -> nil` | 解除：fn 必须是注册时同一引用 |
| `inst:StopAllWatchingWorldStates()` | `() -> nil` | 把 inst 注册的所有 watch 全部解除 |
| `TheWorld.state.xxx` | 直接索引 | 读当前值（任何时候） |

注意**没有 push API**——worldstate 的值由 worldstate 组件**自动从事件计算**——mod 通常不直接修改它。

> 想"自己定义新的 worldstate 变量"？参考 8.3.5 第二步——**通过推世界事件让 worldstate 自动同步**，不要直接改 `data` 字典。

#### 第二步：WatchWorldState 的回调签名

```lua
inst:WatchWorldState("phase", function(inst, new_phase)
    -- inst：第一个参数是 watcher 自己（不是 TheWorld！）
    -- new_phase：变化后的新值（"day" / "dusk" / "night"）
    print("phase changed to:", new_phase)
end)
```

**对比 ListenForEvent**：
- `ListenForEvent("phasechanged", fn)` 的 fn 收到 `(world, phase_string)` —— 第一个参数是 source（TheWorld）
- `WatchWorldState("phase", fn)` 的 fn 收到 `(inst, new_value)` —— 第一个参数是**watcher 自己**

> **这是新手最容易混的点**——签名乍看一样、第一个参数实际意义完全不同。

#### 第三步：5 个真实例子

**例 1：胡子组件——监听天数**

```73:73:scripts/components/beard.lua
            self:WatchWorldState("cycles", OnDayComplete)
```

**例 2：deerclops spawner ——监听季节**

```450:450:scripts/components/deerclopsspawner.lua
self:WatchWorldState("season", OnSeasonChange)
```

**例 3：klaussackspawner——只在冬天**

```215:215:scripts/components/klaussackspawner.lua
        self:WatchWorldState("iswinter", OnIsWinter)
```

**例 4：drying rack——只在下雨**

```103:104:scripts/components/dryingrack.lua
		self:WatchWorldState("israining", OnIsRaining)
		self:WatchWorldState("isacidraining", OnIsAcidRaining)
```

**例 5：lunarhailbuildup——只在落月雹**

```11:11:scripts/components/lunarhailbuildup.lua
    self:WatchWorldState("islunarhailing", self.OnIsLunarHailing)
```

> **观察规律**：**95% 的 watch 都用在 component 里**（用 `self:WatchWorldState`，不是 `self.inst:WatchWorldState`）—— 8.3.6 节解释为什么。

---

### 8.3.3 快速入门：worldstate 全表速查

#### 第一步：从源码生成完整变量列表

worldstate 的所有变量都集中在 `scripts/components/worldstate.lua` 第 188-271 行的"Initialization"块里——下面按主题分类：

**Clock（时钟相关）**

| 变量 | 类型 | 含义 |
| --- | --- | --- |
| `time` | number | 当前 phase 的累计时间（秒） |
| `timeinphase` | number | 当前 phase 进度（0-1） |
| `cycles` | number | 已过的天数（从 0 开始） |
| `phase` | string | "day" / "dusk" / "night" |
| `isday` | bool | phase == "day" |
| `isdusk` | bool | phase == "dusk" |
| `isnight` | bool | phase == "night" |
| `moonphase` | string | "full" / "new" / "quarter" / "threequarter" / "half" |
| `iswaxingmoon` | bool | 是上弦还是下弦 |
| `isfullmoon` | bool | 满月（且夜晚） |
| `isnewmoon` | bool | 新月（且夜晚） |

**Cave clock（洞穴时钟）**——洞穴版本，地表为默认值

| 变量 | 类型 | 含义 |
| --- | --- | --- |
| `cavephase` | string | 洞穴的"白天/黄昏/夜晚" |
| `iscaveday` / `iscavedusk` / `iscavenight` | bool | 洞穴版 |
| `cavemoonphase` | string | 洞穴月相 |
| `iscavefullmoon` / `iscavenewmoon` | bool | 洞穴满月/新月 |

**Nightmare clock（梦魇时钟，安魇遗迹）**

| 变量 | 类型 | 含义 |
| --- | --- | --- |
| `nightmarephase` | string | "calm" / "warn" / "wild" / "dawn" / "none" |
| `nightmaretime` / `nightmaretimeinphase` | number | 进度 |
| `isnightmarecalm` 等 | bool | 当前阶段判断 |

**Season（季节）**

| 变量 | 类型 | 含义 |
| --- | --- | --- |
| `season` | string | "autumn" / "winter" / "spring" / "summer" |
| `isautumn` / `iswinter` / `isspring` / `issummer` | bool | 当前季节判断 |
| `elapseddaysinseason` | number | 当前季节已过的天数 |
| `remainingdaysinseason` | number | 当前季节还剩的天数 |
| `seasonprogress` | number | 当前季节进度（0-1） |
| `springlength` / `summerlength` / `autumnlength` / `winterlength` | number | 各季节长度配置 |

**Weather（天气）**

| 变量 | 类型 | 含义 |
| --- | --- | --- |
| `temperature` | number | 当前气温 |
| `moisture` | number | 累计湿气 |
| `moistureceil` | number | 当前湿气阈值（爆雨阈） |
| `pop` | number | 概率因子 |
| `precipitationrate` | number | 降水率 |
| `precipitation` | string | "none" / "rain" / "snow" / "lunarhail" / "acidrain" |
| `israining` / `issnowing` / `islunarhailing` / `isacidraining` | bool | 当前降水类型 |
| `issnowcovered` | bool | 雪覆盖（冬天积雪） |
| `snowlevel` | number | 雪深度（0-1） |
| `lunarhaillevel` / `lunarhailrate` | number | 月雹等级/速率 |
| `wetness` | number | 玩家身上的湿度（实际由玩家自己同步） |
| `iswet` | bool | 玩家是否湿的 |

**特殊状态**

| 变量 | 类型 | 含义 |
| --- | --- | --- |
| `isalterawake` | bool | "另界觉醒"事件中（月暴风期间） |

#### 第二步：还有"切换变量" `start*/stop*`

注意 `worldstate.lua` 第 37-46 行的 `togglename` 处理——某些**布尔变量**会**额外**生成两个虚拟变量：

| `togglename` | 触发条件 |
| --- | --- |
| `startday` / `stopday` | `isday` 从 false→true / true→false |
| `startnight` / `stopnight` | 同理 |
| `startwinter` / `stopwinter` | `iswinter` 切换 |
| `startrain` / `stoprain` | `israining` 切换 |
| `startfullmoon` / `stopfullmoon` | 满月开始/结束 |
| ... | 详见源码 SetVariable 调用 |

**典型用法**——某 mod 想做"夜晚一开始就出现 buff"：

```lua
inst:WatchWorldState("startnight", function(inst)
    inst:AddTag("nightbuff")
end)
inst:WatchWorldState("stopnight", function(inst)
    inst:RemoveTag("nightbuff")
end)
```

**优势**：比 `WatchWorldState("isnight", function(inst, val) if val then ... else ... end end)` **更清晰**——分别注册"开始"和"结束"。

#### 第三步：用控制台 dump 全部状态

```lua
print(TheWorld.components.worldstate:Dump())
-- => 打印当前所有 60 多个变量的值
```

或者直接**遍历**：

```lua
for k, v in pairs(TheWorld.state) do
    print(k, v)
end
```

> **新手 MVP 检查点**：**写 mod 想读"当前是不是白天"——`if TheWorld.state.isday then`**。**想"白天到了的瞬间触发某事"——`inst:WatchWorldState("isday", fn)` 或 `inst:WatchWorldState("startday", fn)`**。两条规则就够 80% 场景了。

---

### 8.3.4 进阶：`start*/stop*` "切换变量"机制

#### 第一步：再看一次 SetVariable 的 togglename 逻辑

```24:48:scripts/components/worldstate.lua
local function SetVariable(var, val, togglename)
    if self.data[var] ~= val and val ~= nil then
        self.data[var] = val

        local watchers = _watchers[var]
        ...

        if togglename then
            watchers = _watchers[(val and "start" or "stop")..togglename]
            if watchers ~= nil then
                for k, v in pairs(watchers) do
                    for i, fn in ipairs(v) do
                        fn[1](fn[2])
                    end
                end
            end
        end
    end
end
```

**关键**：`(val and "start" or "stop") .. togglename`

- `val=true` → 触发 `_watchers["start"..togglename]`
- `val=false` → 触发 `_watchers["stop"..togglename]`

**举例**：`SetVariable("iswinter", true, "winter")`
- 主 watcher：`_watchers["iswinter"]` 全触发，参数 `(inst, true)`
- 切换 watcher：`_watchers["startwinter"]` 全触发，**没有第二个参数！**

#### 第二步：所有支持 toggle 的变量列表

来自 `worldstate.lua` 里所有 `SetVariable(...,..., togglename)` 调用：

| 主变量 | startxxx / stopxxx |
| --- | --- |
| `isday` | `startday` / `stopday` |
| `isdusk` | `startdusk` / `stopdusk` |
| `isnight` | `startnight` / `stopnight` |
| `isfullmoon` | `startfullmoon` / `stopfullmoon` |
| `isnewmoon` | `startnewmoon` / `stopnewmoon` |
| `iscaveday` / `iscavedusk` / `iscavenight` | 同款 cave 版 |
| `iscavefullmoon` / `iscavenewmoon` | 同款 cave 版 |
| `isnightmarecalm` 等 | `startnightmarecalm` 等 |
| `isspring` / `issummer` / `isautumn` / `iswinter` | `startspring` / `stopspring` 等 |
| `israining` / `issnowing` / `islunarhailing` / `isacidraining` | `startrain` / `stoprain` 等 |

**注意**：toggle 变量**不会出现在 `TheWorld.state` 里**——它们**只是 watcher 名字**。`TheWorld.state.startnight` **永远是 nil**。

#### 第三步：toggle 变量 vs 主变量——什么时候用哪个？

```lua
-- 写法 A：用主变量 + 判分支
inst:WatchWorldState("isnight", function(inst, isnight)
    if isnight then
        OnNightStart(inst)
    else
        OnNightEnd(inst)
    end
end)

-- 写法 B：用 toggle 变量
inst:WatchWorldState("startnight", OnNightStart)
inst:WatchWorldState("stopnight", OnNightEnd)
```

**A 写法**：一个 fn 处理两个边界——**fn 必须有分支**——共享上下文方便。  
**B 写法**：两个独立 fn——**职责单一**——但要注册两次。

> **设计原则**：**逻辑独立用 toggle，逻辑相互依赖（共享 local 变量等）用主变量**。

#### 第四步：不要 watch toggle 变量再去读 `state.startnight`

**错误代码**：

```lua
inst:WatchWorldState("startnight", function(inst)
    print("夜晚开始, 当前 startnight 状态:", TheWorld.state.startnight)
end)
-- 输出：夜晚开始, 当前 startnight 状态: nil
```

**`TheWorld.state.startnight` 永远是 nil** —— 它只是 watcher 名，不是真实状态。**要读"当前是不是夜晚"用 `TheWorld.state.isnight`**。

---

### 8.3.5 进阶：worldstate 与事件系统的协作关系

#### 第一步：worldstate 内部其实是个"事件汇总"

回到 `worldstate.lua` 第 212-271 行——它**自己注册了一堆 ListenForEvent**：

```212:271:scripts/components/worldstate.lua
inst:ListenForEvent("clocktick", OnClockTick)
inst:ListenForEvent("cycleschanged", OnCyclesChanged)
inst:ListenForEvent("phasechanged", _iscave and OnCavePhaseChanged or OnPhaseChanged)
inst:ListenForEvent("moonphasechanged2", _iscave and OnCaveMoonPhaseChanged2 or OnMoonPhaseChanged2)

inst:ListenForEvent("ms_stormchanged", OnAlterAwake)

...

inst:ListenForEvent("nightmareclocktick", OnNightmareClockTick)
inst:ListenForEvent("nightmarephasechanged", OnNightmarePhaseChanged)

...

inst:ListenForEvent("seasontick", OnSeasonTick)
inst:ListenForEvent("seasonlengthschanged", OnSeasonLengthsChanged)

...

inst:ListenForEvent("temperaturetick", OnTemperatureTick)
inst:ListenForEvent("weathertick", OnWeatherTick)
inst:ListenForEvent("moistureceilchanged", OnMoistureCeilChanged)
inst:ListenForEvent("precipitationchanged", OnPrecipitationChanged)
inst:ListenForEvent("snowcoveredchanged", OnSnowCoveredChanged)
inst:ListenForEvent("wetchanged", OnWetChanged)
```

**每个事件回调里都调 `SetVariable`** ——把"瞬时事件" sublimate 成"持久状态值"。

**架构层叠**：

```
clock/season/weather 等组件
    │
    │ PushEvent("phasechanged", "night")
    ▼
worldstate 组件（监听上面的事件）
    │
    │ SetVariable("phase", "night", "phase")
    ▼
_watchers 字典
    │
    │ 调用所有 watcher fn
    ▼
你的 component 的 OnIsNight 函数
```

#### 第二步：什么时候 push 事件，什么时候 watch state？

**判断标准**：

| 类型 | 例子 | 应该用 |
| --- | --- | --- |
| **状态值，可能持续保持** | "现在是不是夜晚"、"现在是不是冬天" | **WatchWorldState** |
| **瞬时事件，没有"持续值"概念** | "玩家死了"、"被攻击"、"完成砍树" | **ListenForEvent** |
| **状态值，但你只关心"变化的瞬间"** | "白天开始的瞬间播音效" | **WatchWorldState** + `start*` |
| **状态值，但你只关心"当前值"** | "我装备了，看现在是不是白天" | 直接读 `TheWorld.state.xxx` |

#### 第三步：自定义 worldstate 变量怎么做？

Klei 的设计是**封闭的**——`worldstate.lua` 是固定文件，**mod 不应该直接改**它的 `data` 字典。但你可以**通过 prefab postinit 间接添加变量**：

```lua
-- modmain.lua
AddPrefabPostInit("world", function(inst)
    -- 监听某个事件 → 写入新 worldstate 变量
    inst:ListenForEvent("mymod_apocalypse_phase_changed", function(world, data)
        if inst.components.worldstate then
            inst.state.mymod_apocalypsephase = data.phase
            -- 注意：直接改 data 字典 → 不会触发 _watchers
        end
    end)
end)
```

**问题**：直接改 `inst.state.xxx` **不会触发 watcher**——因为 worldstate 的内部 `SetVariable` 才管 watcher 触发。

**变通**：**调用** `inst.components.worldstate:AddWatcher` 和 `RemoveWatcher` 公开接口，**或者**——**最务实的做法**——**业务 mod 直接用事件系统**，不要去碰 worldstate 内部。**worldstate 系统是 Klei 内置的，扩展性差**。

> **设计经验**：**自定义状态用 `inst.xxx` 直接挂世界，提供 getter；变化时 push 自定义事件**——比破坏 worldstate 简单且不易出 bug。

#### 第四步：实战 ——为什么 dryingrack 同时用 watch 和 state

回到 `scripts/components/dryingrack.lua` 第 103-120 行：

```103:121:scripts/components/dryingrack.lua
		self:WatchWorldState("israining", OnIsRaining)
		self:WatchWorldState("isacidraining", OnIsAcidRaining)

		--V2C: use closures
		--     Don't use "local self = inst.components.dryingrack"
		--     because it might be wobyrack
		self._onrainimmunity = function()
			self:SetContainerRainImmunity(true)
			self:SetContainerIsInAcid(false)
			self:ResumeDrying()
		end
		self._onrainvulnerable = function()
			if not self:HasRainImmunity() then
				self:SetContainerRainImmunity(false)
				if self:IsExposedToRain() then
					self:SetContainerIsInAcid(TheWorld.state.isacidraining)
					self:PauseDrying()
```

**两种用法并存**：
- `self:WatchWorldState("israining", OnIsRaining)` —— 下雨**变化时**响应（开始/结束都触发）
- `self:SetContainerIsInAcid(TheWorld.state.isacidraining)` —— **当前是不是酸雨**——直接查 state

**这是教科书式的"watch + state"配合**：
- watch 处理"边界事件"
- state 处理"当下查询"

---

### 8.3.6 进阶：`Component:WatchWorldState` vs `EntityScript:WatchWorldState` 的差异

#### 第一步：两份不同的实现

**`EntityScript:WatchWorldState`**（在 `entityscript.lua:1264`）：

```1264:1267:scripts/entityscript.lua
function EntityScript:WatchWorldState(var, fn)
    EntityWatchWorldState(self, var, fn)
    TheWorld.components.worldstate:AddWatcher(var, self, fn, self)
end
```

**`Component:WatchWorldState`**（在 `entityscript.lua:42`）：

```42:45:scripts/entityscript.lua
local function ComponentWatchWorldState(self, var, fn)
    EntityWatchWorldState(self.inst, var, fn)
    TheWorld.components.worldstate:AddWatcher(var, self.inst, fn, self)
end
```

#### 第二步：差异对比

| 维度 | EntityScript:WatchWorldState | Component:WatchWorldState |
| --- | --- | --- |
| 调用形式 | `inst:WatchWorldState("var", fn)` | `self:WatchWorldState("var", fn)` |
| `AddWatcher` 第一个参数 | `self`（inst 自己） | `self.inst`（组件的宿主） |
| `AddWatcher` 第二个参数（target） | `self`（inst 自己） | `self`（组件实例） |
| **回调 fn 接收的第一个参数** | **inst** | **self（组件）** |

**最关键的差异在第二个参数 target**——回到 `worldstate.lua:281-295`：

```281:295:scripts/components/worldstate.lua
function self:AddWatcher(var, inst, fn, target)
    local watchers = _watchers[var]
    if watchers == nil then
        watchers = {}
        _watchers[var] = watchers
    end

    local watcherfns = watchers[inst]
    if watcherfns == nil then
        watcherfns = {}
        watchers[inst] = watcherfns
    end

    table.insert(watcherfns, { fn, target })
end
```

存进去的是 `{ fn, target }`——回到 SetVariable：`fn[1](fn[2], val)` —— **fn[2] 就是 target，传给 fn 当第一个参数**。

**于是回调 fn 的第一个参数**：
- `EntityScript:WatchWorldState` 注册时 → 第一个参数是 **inst**
- `Component:WatchWorldState` 注册时 → 第一个参数是 **组件实例 self**

#### 第三步：实战写法对比

**写法 A：在 component 里**

```lua
local MyComp = Class(function(self, inst)
    self.inst = inst
    self.value = 0
    self:WatchWorldState("cycles", self.OnDayChanged)
end)

function MyComp:OnDayChanged(cycles)
    -- self 是 MyComp 实例，cycles 是新值
    self.value = self.value + 1
end
```

**注意**：`self:WatchWorldState` ——没显式传 `self`。回调 fn 是 `self.OnDayChanged`，**回调被触发时 self 隐式自动传入**——和"普通 OOP 方法"调用一致。

**写法 B：在 prefab fn 里**

```lua
local function fn()
    local inst = CreateEntity()
    -- ...
    inst:WatchWorldState("cycles", function(inst, cycles)
        print("day", cycles, "for", inst)
    end)
    return inst
end
```

**这里第一个参数是 inst** —— 和 ListenForEvent 的语义一致。

#### 第四步：用错版本会怎样？

**错误**：在 component 里用 `self.inst:WatchWorldState`：

```lua
function MyComp:OnInit()
    -- 错！
    self.inst:WatchWorldState("cycles", function(inst, cycles)
        -- 这里 inst 是组件宿主，不是 self
        -- 你想访问 self.value 必须闭包捕获
    end)
end
```

**结果**：能跑——但**回调里没法直接 self.xxx**——必须闭包捕获 `local s = self`。**且实体销毁时清理可能不一致**（EntityScript 路径自动清理；Component 路径可能漏一份）。

> **设计经验**：**在 component 里永远用 `self:WatchWorldState`**——这是 Klei 的所有内置组件统一做法。在 prefab fn / mod 顶层用 `inst:WatchWorldState`。

#### 第五步：为什么这个差异重要？

回想 8.1.3 双向链表——RemoveAllEventCallbacks 在实体销毁时**自动清理双向引用**。但 worldstate 的 watcher 表存在**`worldstate.lua` 内部的 `_watchers`** —— **不是 `event_listeners`**。

幸运的是 `EntityWatchWorldState` 把 fn 也存在 `self.worldstatewatching` 里——`StopAllWatchingWorldStates`（实体销毁时自动调）会**反查这张表去 `worldstate:RemoveWatcher`**：

```1274:1284:scripts/entityscript.lua
function EntityScript:StopAllWatchingWorldStates()
    if not self.worldstatewatching then
        return
    end

    for var in pairs(self.worldstatewatching) do
        TheWorld.components.worldstate:RemoveWatcher(var, self)
    end

    self.worldstatewatching = nil
end
```

**注意**：`RemoveWatcher` 这里**不传 fn**——所以**会删掉这个 inst 注册的所有 fn**——批量清理。

**但**：`Component:WatchWorldState` 把 target 设为 `self`（组件），而不是 inst——**实体销毁清理时按 inst 删**——**target 字段没被检查** —— 全部删除是没问题的，但调试时可能看不清楚谁在听。

> **新手不必纠结这块细节**——只记"在 component 里用 `self:WatchWorldState`、其他场景用 `inst:WatchWorldState`"就够了。

---

### 8.3.7 老手进阶：注册时不会触发——初始化技巧

#### 第一步：经典坑——Watch 注册时不会立刻触发

**新手常见的错觉**：`WatchWorldState("isnight", fn)` 注册的瞬间——以为 fn 会被立即调一次（"告诉我现在是不是夜晚"）。

**实际**：**只在变化时**触发——如果你**夜晚的时候**才注册，**这个夜晚结束前不会触发任何回调**——你的实体永远以为"现在不是夜晚"。

回看 SetVariable：

```lua
if self.data[var] ~= val and val ~= nil then  -- 只在变化时
    self.data[var] = val
    -- ... 调 watcher
end
```

注册时**没人调 fn**——fn 第一次被调是**下一次值变化**。

#### 第二步：标准修复模式——注册后手动初始化

```lua
local function OnIsNight(inst, isnight)
    if isnight then
        inst:AddTag("nightbuff")
    else
        inst:RemoveTag("nightbuff")
    end
end

inst:WatchWorldState("isnight", OnIsNight)
-- 立刻按当前状态调一次
OnIsNight(inst, TheWorld.state.isnight)
```

**核心模式**：**注册 watch + 立刻按当前状态手动调一次**。**写 mod 时这一对是"绑在一起的"**。

#### 第三步：在 component 里的对应写法

```lua
function MyComp:Init()
    self:WatchWorldState("season", self.OnSeasonChange)
    -- 立刻初始化
    self:OnSeasonChange(TheWorld.state.season)
end

function MyComp:OnSeasonChange(season)
    if season == "winter" then
        self:EnableSnowGlow()
    else
        self:DisableSnowGlow()
    end
end
```

**等价写法**：

```lua
function MyComp:Init()
    self:WatchWorldState("season", function(self, season) self:OnSeasonChange(season) end)
    self:OnSeasonChange(TheWorld.state.season)  -- 直接调
end
```

#### 第四步：start*/stop* 的初始化技巧

如果用 `startxxx`/`stopxxx`——**初始化必须自己分支判**：

```lua
inst:WatchWorldState("startnight", OnNightStart)
inst:WatchWorldState("stopnight", OnNightEnd)

-- 立刻按当前状态调一次
if TheWorld.state.isnight then
    OnNightStart(inst)
else
    OnNightEnd(inst)
end
```

**否则**：白天注册后，要等到**下一次夜晚开始**才会触发 OnNightStart——白天期间 OnNightEnd **永远不会被调**——你的"白天默认状态"逻辑跑不到。

#### 第五步：异步初始化——如果 TheWorld 还没准备好

如果你的代码在 **mod 早期加载**（比如 modmain 顶层）就要 watch——`TheWorld` 可能还是 nil。**用 PostInit 钩子等到 TheWorld 准备好**：

```lua
AddPrefabPostInit("world", function(world)
    world:WatchWorldState("season", OnSeasonChange)
    OnSeasonChange(world, world.state.season)
end)
```

或者用 `AddSimPostInit`（更晚，全局确认完毕后）。

---

### 8.3.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：注册后等不到回调——以为 watch 注册就触发

**症状**：写"夜晚发光"——半小时内夜晚都没发光，调试半天发现是夜晚里注册的，**这个夜晚根本不会再触发"isnight=true"**。  
**修复**：见 8.3.7 第二步——注册后立刻按当前状态调一次。

#### 陷阱 2：在 component 里用 `self.inst:WatchWorldState` 而非 `self:WatchWorldState`

**症状**：回调里 `self.value` 报 nil（self 是 inst 不是 component）。  
**原因**：`EntityScript:WatchWorldState` 的回调第一个参数是 inst；`Component:WatchWorldState` 的回调第一个参数才是 component self。  
**修复**：在 component 里**永远用 `self:WatchWorldState`**。

#### 陷阱 3：注册的 fn 是匿名 function，没法 Stop

**症状**：mod 卸载或玩家退出时 `StopWatchingWorldState` 调用了但 fn 引用对不上——watcher 残留。  
**原因**：和 8.1.7 的事件系统问题一模一样——匿名 fn 每次创建都是不同的引用。  
**修复**：把 fn 存到 `self._cb_xxx`，注册和 stop 时用同一个引用。

#### 陷阱 4：把"切换变量"当成可读字段

**症状**：

```lua
if TheWorld.state.startnight then ... end
-- 永远是 nil ≠ true
```

**原因**：`startxxx` / `stopxxx` 只是 watcher 名，不是 worldstate.data 的字段。  
**修复**：读 state 用主变量 `isnight`；watch "瞬间触发"用 `startnight`。

#### 陷阱 5：直接修改 `TheWorld.state.xxx` 想触发 watcher

**症状**：mod 想强制让所有人"以为是夜晚"，写 `TheWorld.state.isnight = true` —— 别的代码用 `TheWorld.state.isnight` 看到 true，**但 watcher 没触发**。  
**原因**：watcher 触发**只在 SetVariable 内部**——直接改 data 字典绕过它。  
**修复**：要触发 watcher 必须**走 worldstate 的 source 事件**——比如 push `phasechanged` 给 TheWorld（但这会被 clock 组件覆盖）。**正确做法**：用自己的事件 + 自己的状态变量，不要篡改 worldstate。

#### 陷阱 6：在客户端 `TheWorld.state.xxx` 不准

**症状**：客户端读 `TheWorld.state.season` 是 "autumn"，主机端是 "winter"。  
**原因**：worldstate 的源事件部分是**主机端独占**——客户端 worldstate 组件可能依赖不同的同步路径——**不是所有变量都两端一致**。  
**修复**：判断 ismastersim 或者**通过网络变量同步**关键状态——用 `TheWorld.net` 或 `inst.replica.xxx`。**通常 `phase`、`isday`、`season` 在客户端是同步的**——但 `temperature`、`moisture` 等连续值**可能仅服务端有效**。**遇到具体问题时查 worldstate 源码确认**。

#### 设计经验三条

**经验 ①：永远"watch + 立刻初始化"成对写**

```lua
inst:WatchWorldState("isnight", OnIsNight)
OnIsNight(inst, TheWorld.state.isnight)  -- 永远跟一行
```

**经验 ②：能用 `TheWorld.state.xxx` 就别 watch**

如果你只关心"我现在做某事时世界是什么状态"——直接查 state，**不要 watch**。watch 是"持续订阅"，**只有当你需要"在变化的瞬间响应"**时才用。

```lua
-- 装备时检查当前是不是夜晚——不需要 watch
function MyComp:OnEquipped(owner)
    if TheWorld.state.isnight then
        self:Glow()
    end
end
```

**经验 ③：toggle 变量优于"主变量+分支"**

如果"开始/结束"逻辑**完全独立**（不共享 local 变量、不用同一个 cleanup），**优先 toggle**：

```lua
-- 推荐
inst:WatchWorldState("startwinter", OnWinterStart)
inst:WatchWorldState("stopwinter", OnWinterEnd)

-- 也对，但不那么清晰
inst:WatchWorldState("iswinter", function(inst, iswinter)
    if iswinter then OnWinterStart(inst) else OnWinterEnd(inst) end
end)
```

---

### 8.3.9 小结

**worldstate 一句话总结**：**`WatchWorldState` 是事件系统的"持久状态版"**——只在值真正变化时触发，注册时**不会**立刻触发；任何代码任何时候都能 `TheWorld.state.xxx` 读当前值。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 读当前状态 | `TheWorld.state.isnight` / `TheWorld.state.season` |
| 监听变化 | `inst:WatchWorldState("isnight", fn)` |
| 监听"开始"瞬间 | `inst:WatchWorldState("startnight", fn)` |
| 监听"结束"瞬间 | `inst:WatchWorldState("stopnight", fn)` |
| 解除一个 | `inst:StopWatchingWorldState("isnight", fn)` |
| 解除全部 | `inst:StopAllWatchingWorldStates()` |
| Component 内 | `self:WatchWorldState("xxx", self.OnX)` |

**6 个陷阱排雷顺序**

1. 注册不触发 → 立刻手动初始化
2. component 用错版本 → `self:WatchWorldState`
3. 匿名 fn 没法 stop → 具名/存字段
4. toggle 当字段读 → 用主变量 `isnight`
5. 直接改 state.xxx → 没用，必须 push 源事件
6. 客户端读不准 → 用网络变量同步

**3 条设计经验**

- ① **watch + 初始化成对**——永远在注册后立刻手动调一次
- ② **能用 state 别用 watch**——只在"瞬间响应"时才订阅
- ③ **toggle 优于主变量分支**——逻辑独立时更清晰

**worldstate 与事件系统对比**

| 维度 | ListenForEvent | WatchWorldState |
| --- | --- | --- |
| 触发条件 | 每次 PushEvent 都触发 | 仅当 data[var] 真实变化 |
| 注册时是否触发 | 否 | 否（同样需要手动初始化）|
| 数据 | 任意 data | 单个 val（变化后的新值） |
| 状态查询 | 没有"当前事件值"概念 | `TheWorld.state.xxx` 任意时刻可读 |
| 适用场景 | 瞬时通知、跨实体通信 | 持续状态、周期性变化 |
| 作用域 | 任意实体 | 仅 TheWorld（理论上） |

> **下一节预告**：8.4 节我们把"事件驱动 vs 帧更新"两种范式对比讲清楚——`OnUpdate` / `DoTaskInTime` / `DoPeriodicTask` / `EventHandler` —— **同一个需求用哪种实现性能最好、维护最易**？比如"每秒检查一次怪物距离"该用 OnUpdate 还是 DoPeriodicTask？"玩家死亡时立刻播音效"该用事件还是 OnUpdate 轮询？读完 8.4，你将能在 mod 写法上做出**架构层级的优化决策**——这是 Klei 内部组件被反复推敲的设计理念。

## 8.4 事件驱动 vs 帧更新——何时用哪个

### 本节导读

8.1 ~ 8.3 我们把"事件 / 状态"这条**响应式**线讲透了——**有事发生 → 触发回调**。但 mod 里**还有大量逻辑不是"被动响应"，而是"主动周期性执行"**：

- 玩家**每秒掉 1 点饥饿值**——没有"饥饿事件"在主动 push，就是要持续地减
- 攻击玩家的怪物**每帧检测距离**——决定路径/转向
- 季节倒计时**每帧累加 dt**
- 自动炊具的"15 秒后出锅"
- 暴风雪的"每 30 秒刷一次额外伤害"

这些**周期性 / 延时执行**的需求，饥荒提供了**至少 4 套不同的工具**——`OnUpdate` / `DoTaskInTime` / `DoPeriodicTask` / `DoStaticTaskInTime`。**它们语义有微妙差异、性能开销也不同**——选错了不是不能跑，而是会**额外 CPU 开销 / 暂停时机不对 / 实体销毁后还在跑**。

> **新手**从 8.4.1-8.4.3 起步——理解"事件 vs 更新"的本质区别、4 套 API 速查、什么时候用哪种；**进阶读者**继续看 8.4.4-8.4.6，深入 `scheduler` vs `staticScheduler` 的差异、`StartUpdatingComponent` 的实现细节、`DoPeriodicTask` / `OnUpdate` / `OnPostUpdate` 的适用边界；**老手**跳到 8.4.7-8.4.8，看"监听 vs 轮询"的真实性能对比、6 个最常踩的坑。

---

### 8.4.1 快速入门：从一只"饥饿"看更新机制

#### 第一步：在游戏里观察"持续掉饥饿值"

游戏里玩家**每秒掉一点饥饿**——这不是事件触发的（没有"每秒事件"在 push）——它是**饥饿组件主动按周期减**。

打开 `scripts/components/hunger.lua` 第 38 行：

```38:38:scripts/components/hunger.lua
    self.updatetask = self.inst:DoPeriodicTask(UPDATE_PERIOD, OnTaskTick, nil, self)
```

**`UPDATE_PERIOD = 1`**（第 4 行）—— **每 1 秒触发一次** OnTaskTick。

```18:20:scripts/components/hunger.lua
local function OnTaskTick(inst, self)
    self:DoDec(UPDATE_PERIOD)
end
```

**就这么简单**——周期性减 1 点饥饿值。

**关键观察**：
- **不是事件驱动**（没人 PushEvent("hungertick")）
- **不是帧更新**（不是每帧 60 次，是每秒 1 次）
- **是"周期性任务"**——`DoPeriodicTask`

而对比 **locomotor 移动组件**——它需要**每帧**重新计算路径、调整速度——用 `OnUpdate`：

```1424:1424:scripts/components/locomotor.lua
function LocoMotor:OnUpdate(dt, arrive_check_only)
```

Locomotor 在开始移动时调 `self.inst:StartUpdatingComponent(self)` 把自己注册到**每帧 update 列表**——引擎每帧调用所有组件的 `OnUpdate(dt)`。

#### 第二步：4 种"持续执行"机制对比

| 机制 | 频率 | 实现 | 典型场景 |
| --- | --- | --- | --- |
| `OnUpdate(dt)` + StartUpdatingComponent | **每帧**（约 30 FPS） | 组件注册到引擎全局 update 列表 | 移动、动画驱动的精确控制、需要 dt 的连续模拟 |
| `DoPeriodicTask(dt, fn)` | **每 dt 秒**（任意周期） | scheduler 排进任务队列 | 饥饿/口渴/精神/生长——**周期>=0.5 秒**的低频任务 |
| `DoTaskInTime(dt, fn)` | **延时 dt 秒后一次性** | scheduler 排进任务队列 | 死亡后 5 秒 fade out、动画后回调 |
| `DoStaticTaskInTime(dt, fn)` | 同上，但用 staticScheduler | staticScheduler 不受暂停影响 | 暂停菜单时仍要走的任务（罕见） |

#### 第三步：再看一次"事件 vs 更新"的核心区分

| 维度 | 事件驱动 | 更新驱动 |
| --- | --- | --- |
| 触发条件 | 外部 `PushEvent` 推送 | 时间到达（每帧 / 每周期 / 延时点） |
| 谁主动 | 推送者 | 自己 |
| 数据 | 事件 data | 自己内部状态 |
| 性能 | 0 push 0 触发；push 1 次触发 N 个 listener | **永远在跑**，不管有没有"事情" |
| 暂停 | 暂停时不 push 就不触发 | 周期任务可能仍在跑（取决于 scheduler） |
| 适用 | "X 发生了，我想响应" | "我要持续做 Y" |

> **核心原则**：**能用事件驱动的不要用更新轮询**——**事件 = 节能模式，更新 = 24 小时持续耗电**。但**有些场景必须更新驱动**（连续值、每帧响应）—— 不能强求事件。

---

### 8.4.2 快速入门：四种"持续执行"工具速查

#### 第一步：`OnUpdate(dt)` —— 每帧调用

**写法**：

```lua
local MyComp = Class(function(self, inst)
    self.inst = inst
    self.elapsed = 0
end)

function MyComp:Activate()
    self.inst:StartUpdatingComponent(self)
end

function MyComp:Deactivate()
    self.inst:StopUpdatingComponent(self)
end

function MyComp:OnUpdate(dt)
    self.elapsed = self.elapsed + dt
    -- 每帧业务
end
```

**两个钩子**：
- `StartUpdatingComponent(self)` —— 把组件**加入**全局每帧 update 列表
- `StopUpdatingComponent(self)` —— **移出**

**频率**：**每帧**（30 FPS = 每秒 30 次）

**dt**：上一帧到这一帧的时间间隔（秒，浮点）—— 通常 0.033 左右（30 FPS）

**使用场景**：
- 移动 / 动画 / 物理同步
- 任何"每帧都要做"的精确控制
- **不要**用来做"每秒做一次"——用 DoPeriodicTask

#### 第二步：`DoPeriodicTask(time, fn, initialdelay, ...)` —— 周期任务

**写法**：

```lua
self.task = self.inst:DoPeriodicTask(1, function(inst, ...)
    print("每秒触发一次")
end, nil, ...)

-- 取消
if self.task then
    self.task:Cancel()
    self.task = nil
end
```

**参数**：
- `time` —— 周期（秒）
- `fn` —— 回调，**签名 `function(inst, ...)`**——第一个参数是 inst
- `initialdelay` —— 第一次触发的初始延迟（nil 时同 time；传 0 立刻触发一次）
- `...` —— 任意附加参数，会传给 fn 的尾部参数

**返回值**：`Periodic` 对象——可调 `:Cancel()` 终止

**典型用例**：
- hunger / sanity / health 的"每秒衰减"
- "每 5 秒检查 boss 距离"
- "每 30 秒生成一只蜘蛛"

#### 第三步：`DoTaskInTime(time, fn, ...)` —— 延时一次性

**写法**：

```lua
self.inst:DoTaskInTime(3, function(inst)
    print("3 秒后才执行")
end)
```

**参数**：和 DoPeriodicTask 类似，但**只触发一次**。

**典型用例**：
- "动画播完 0.5 秒后销毁特效"
- "玩家死后 5 秒 fade out"
- "对话框 10 秒后自动关闭"

#### 第四步：`DoStaticTaskInTime(time, fn, ...)` —— 不受暂停影响

**写法**：

```lua
self.inst:DoStaticTaskInTime(2, function(inst)
    print("即使游戏暂停，2 秒后也会触发")
end)
```

**和 DoTaskInTime 的差异**：用的是 `staticScheduler` 而不是 `scheduler`——**`staticScheduler` 在游戏暂停时仍然推进**。

**典型用例**：
- 暂停菜单上的动画
- 不依赖游戏内时间的 UI tween
- **罕见**——95% 的延时任务用 `DoTaskInTime` 就够了

> 看 `entityscript.lua:1500-1510`：

```1500:1510:scripts/entityscript.lua
function EntityScript:DoStaticTaskInTime(time, fn, ...)
    --print ("DO TASK IN TIME", time, self)
    if not self.pendingtasks then
        self.pendingtasks = {}
    end

    local per = staticScheduler:ExecuteInTime(time, fn, self.GUID, self, ...)
    self.pendingtasks[per] = true
    per.onfinish = task_finish -- function() if self and self.pendingtasks then self.pendingtasks[per] = nil end end
    return per
end
```

#### 第五步：用控制台快速验证

```lua
-- DoTaskInTime 一次性
ThePlayer:DoTaskInTime(2, function(p) print("2s passed") end)

-- DoPeriodicTask 周期
local task = ThePlayer:DoPeriodicTask(1, function(p) print("tick") end)
-- 5 秒后取消
ThePlayer:DoTaskInTime(5, function() task:Cancel() end)
```

---

### 8.4.3 快速入门：什么时候用事件、什么时候用更新

#### 第一步：决策表

| 你想做的事 | 应该用 |
| --- | --- |
| "玩家被攻击时播音效" | **事件**（`ListenForEvent("attacked", fn)`） |
| "玩家每秒掉饥饿值" | **更新**（`DoPeriodicTask(1, ...)`） |
| "动作动画第 10 帧时执行业务" | **stategraph TimeEvent**（事件的特化，见 7.3.5） |
| "boss 死了后 5 秒生成奖励" | **延时**（`DoTaskInTime(5, ...)`） |
| "每帧检测玩家距离调整 AI" | **更新**（`OnUpdate(dt)`） |
| "白天到了的瞬间触发某事" | **worldstate**（`WatchWorldState("startday", fn)`） |
| "玩家每过 1 天获得 +1 经验" | **worldstate** + cycles（`WatchWorldState("cycles", fn)`） |
| "穿装备时持续 buff 攻击力" | **事件**（equipped/unequipped） + 标记 buff |
| "持续追踪鼠标位置画一条线" | **更新**（每帧 OnUpdate） |
| "30 秒倒计时 UI" | **DoPeriodicTask 0.1 秒** + 文本更新 |

#### 第二步：判断"必须用更新"的 3 个场景

**场景 1：连续值，需要 dt 累加**

`OnUpdate(dt)` 提供了精确的 dt——**适合连续模拟**：

```lua
function MyComp:OnUpdate(dt)
    self.charging = self.charging + dt
    if self.charging > 5 then
        self:Trigger()
    end
end
```

事件做不到这个——除非你用 `DoPeriodicTask(0.033)` 自己模拟每帧——**性能开销和 OnUpdate 一样**——直接用 OnUpdate 更对。

**场景 2：高频率响应（>10 Hz）**

```lua
-- 每 0.1 秒检测一次（10 Hz）—— 性能尚可
self.inst:DoPeriodicTask(0.1, CheckEnemies)

-- 每 0.05 秒检测（20 Hz）—— 还行
self.inst:DoPeriodicTask(0.05, CheckEnemies)

-- 每帧检测（30 Hz）—— 用 OnUpdate
self.inst:StartUpdatingComponent(self)
```

**经验值**：周期 < 0.05 秒考虑 OnUpdate，周期 >= 0.5 秒用 DoPeriodicTask，中间区间两种都行。

**场景 3：游戏循环主流程相关**

引擎层面的 `Camera:Update`、`PostProcessor:Update`、`Networking:Update` 都按引擎时序——这些用 `OnUpdate` 才能与帧严格对齐。

#### 第三步：判断"必须用事件"的 3 个场景

**场景 1：跨实体响应**

如果你的 component **关心别人的事**——必须用事件（推过来）或主动查询（每帧轮询），事件**性能远优于轮询**。

```lua
-- 推荐：事件驱动
inst:ListenForEvent("startfreezing", OnStartFreezing, ThePlayer)

-- 反推荐：每帧轮询
self.inst:StartUpdatingComponent(self)  -- ❌
function MyComp:OnUpdate(dt)
    if ThePlayer:HasTag("freezing") and not self._was_freezing then
        OnStartFreezing(ThePlayer)
        self._was_freezing = true
    end
end
```

**轮询的问题**：每帧 30 次空转——10 个 mod 这样写就 300 次空转——**绝对不要这样写**。

**场景 2：稀疏触发的业务**

"玩家死亡 / 装备替换 / 状态变化"——这些**几小时才触发一次**——用事件几乎零开销；用更新轮询则**永远在跑**。

**场景 3：状态变化通知**

worldstate / netvar 等"值变化"——已经有专门的"瞬时通知"机制（WatchWorldState、netvar dirty handler）——**直接用就行**。

---

### 8.4.4 进阶：`scheduler` 与 `staticScheduler` 的差异

#### 第一步：scheduler 是什么？

`scripts/scheduler.lua`（不展开源码）—— Klei 自己实现的 Lua 协程调度器，**和引擎主循环挂钩**：每帧引擎跑完核心流程后调一次 `scheduler:OnUpdate(dt)`，让队列里的延时任务/周期任务推进。

**两个独立 scheduler**：
- `scheduler`（默认）—— 受游戏暂停影响；游戏暂停时**不推进**任务时间
- `staticScheduler` —— 不受暂停影响；按真实时钟推进

**99% 业务用 scheduler**——只有 UI 动画/计时器等"暂停时也要继续"的场景才用 staticScheduler。

#### 第二步：DoTaskInTime / DoPeriodicTask 用的是哪个？

回到 `entityscript.lua:1512-1530`：

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
```

**默认 scheduler** ——暂停时停。

而 `DoStaticTaskInTime` 用 `staticScheduler:ExecuteInTime`。

#### 第三步：实体销毁时的自动清理

**关键设计**：每个延时/周期任务都注册到 `self.pendingtasks`：

```lua
self.pendingtasks = self.pendingtasks or {}
self.pendingtasks[periodic] = true
```

**实体销毁时**——`EntityScript:OnRemoveEntity`（不展开源码）会**遍历 `pendingtasks` 并 Cancel 所有任务**——避免对已销毁实体的回调。

> **新手不必手动 Cancel** ——只要实体走正常 `Remove` 流程，所有 `DoTaskInTime` 和 `DoPeriodicTask` 都会被自动清理。但**如果任务的回调里访问的是 inst 之外的对象**——可能仍然有问题（见 8.4.8 陷阱 5）。

#### 第四步：手动 Cancel 任务

```lua
local task = self.inst:DoPeriodicTask(1, OnTick)
-- ...

if task ~= nil then
    task:Cancel()
    task = nil
end
```

**`task:Cancel()`** ——立刻从 scheduler 队列里移除该任务。**注意**：Cancel 后**别再访问 task.xxx**——里面的字段可能已经被清理。

---

### 8.4.5 进阶：`StartUpdatingComponent` 的实现细节

#### 第一步：源码核心

```439:473:scripts/entityscript.lua
function EntityScript:StartUpdatingComponent(cmp, do_static_update)
    if not self:IsValid() then
        return
    end

    if not self.updatecomponents then
        self.updatecomponents = {}
        NewUpdatingEnts[self.GUID] = self
        num_updating_ents = num_updating_ents + 1
    end

    if do_static_update then
        if not self.updatestaticcomponents then
            self.updatestaticcomponents = {}
            NewStaticUpdatingEnts[self.GUID] = self
        end
    end

    if StopUpdatingComponents[cmp] == self then
        StopUpdatingComponents[cmp] = nil
    end

    local cmpname = nil
    for k,v in pairs(self.components) do
        if v == cmp then
            cmpname = k
            break
        end
    end
    self.updatecomponents[cmp] = cmpname or "component"

    if do_static_update then
        self.updatestaticcomponents[cmp] = cmpname or "component"
    end
end
```

**关键观察**：
- **`NewUpdatingEnts[self.GUID] = self`** —— 加入"待更新实体"全局表（下一帧合并到 `UpdatingEnts`）
- **`self.updatecomponents[cmp]` 存组件名** —— 用于调试时定位"是哪个组件在跑"
- **`StopUpdatingComponents[cmp] = nil`** —— 撤销之前可能存在的"延后停止"标记

#### 第二步：每帧引擎做的事

伪码：

```
for each ent in UpdatingEnts:
    if not ent:IsValid(): remove
    else:
        for each cmp in ent.updatecomponents:
            cmp:OnUpdate(dt)
```

**每帧** Klei 会遍历**所有注册了 update 的组件**——调它们的 `OnUpdate(dt)`。**注意**：**所有实体所有 update 组件都在主线程同步跑**——**不是分线程**。

**性能含义**：**update 组件越多，每帧越慢**。**饥荒峰值时全图可能有数千个 update 组件**——一个简单的 mod 加几百个就开始卡。

#### 第三步：Stop 用的是"延后清理"机制

```475:478:scripts/entityscript.lua
function EntityScript:StopUpdatingComponent(cmp)
    if self.updatecomponents or self.updatestaticcomponents then
        StopUpdatingComponents[cmp] = self
    end
end
```

**注意**：`StopUpdatingComponent` **不是立刻从 update 列表里删** —— 它**只是把 cmp 加入"待停止"全局表**。**真正的删除发生在下一帧的合并阶段**——见 `StopUpdatingComponent_Deferred`：

```481:502:scripts/entityscript.lua
function EntityScript:StopUpdatingComponent_Deferred(cmp)
    if self.updatecomponents then
        self.updatecomponents[cmp] = nil

        if IsTableEmpty(self.updatecomponents) then
            self.updatecomponents = nil
            UpdatingEnts[self.GUID] = nil
            NewUpdatingEnts[self.GUID] = nil
            num_updating_ents = num_updating_ents - 1
        end
    end
    ...
end
```

**为什么延后？** ——**避免在 update 循环中改 update 表**——经典的"迭代时修改容器"问题。

**对开发者的影响**：**调 `StopUpdatingComponent` 后，本帧 `OnUpdate` 仍可能再被调一次**——回调里要做"是否还应该跑"的判断：

```lua
function MyComp:OnUpdate(dt)
    if self.disabled then return end  -- 防御
    -- ...
end
```

#### 第四步：Wall update 是另一套

`StartWallUpdatingComponent` —— 用于"墙体物理"等需要在世界更新前跑的逻辑——**优先级高于普通 update**。**95% 的 mod 用普通 update 即可**。

---

### 8.4.6 进阶：DoPeriodicTask vs OnUpdate vs OnPostUpdate 适用场景

#### 第一步：3 种"周期性钩子"对比

| 钩子 | 频率 | 实现 | 性能开销 | 取消方式 |
| --- | --- | --- | --- | --- |
| `OnUpdate(dt)` + StartUpdatingComponent | 每帧 ~30 Hz | update 列表 | 全实体并行扫描——所有 update 组件累加 | StopUpdatingComponent |
| `DoPeriodicTask(time, fn)` | 任意周期 | scheduler | 每周期入队/出队——周期长则便宜 | task:Cancel() |
| `OnPostUpdate(dt)` + RegisterEntityForPostUpdate | 每帧 ~30 Hz（在 update 之后） | postupdate 列表 | 同 update | 同 update |

**`OnPostUpdate`** 适用于"必须在所有 update 完成后才能跑的逻辑"——比如**跨多个组件的同步**。**实战中很少用**——99% 用 `OnUpdate`。

#### 第二步：决策表

**用 OnUpdate 的场景**：
- 移动、动画、物理
- 需要 dt 的连续模拟（充能、累计）
- 周期 < 0.1 秒的高频任务

**用 DoPeriodicTask 的场景**：
- 周期 >= 0.5 秒的低频任务
- 饥饿/口渴/精神/生长等"游戏化"周期值
- 周期固定且不需要 dt 精度

**用 DoTaskInTime 的场景**：
- 一次性延时
- 状态变化后的"延后处理"
- 动画后清理特效

#### 第三步：实战 1 ——同样需求的不同写法

需求：**实体 5 秒后自动销毁**。

**写法 A：DoTaskInTime**（推荐）

```lua
inst:DoTaskInTime(5, function(inst)
    inst:Remove()
end)
```

**写法 B：OnUpdate 累计**

```lua
self.inst:StartUpdatingComponent(self)
self.elapsed = 0
function MyComp:OnUpdate(dt)
    self.elapsed = self.elapsed + dt
    if self.elapsed >= 5 then
        self.inst:Remove()
    end
end
```

**写法 A 远优**：
- 一行代码 vs 一整套
- 性能 ~0（scheduler 内部 priority queue）vs 每帧 30 Hz 空转
- 自动清理 vs 需要 StopUpdatingComponent

**结论**：**永远不要用 OnUpdate 做"延时一次性"逻辑**。

#### 第四步：实战 2 —— 周期任务的两种写法

需求：**每秒检查一次玩家温度**。

**写法 A：DoPeriodicTask 1 秒**（推荐）

```lua
self.task = inst:DoPeriodicTask(1, CheckTemperature)
```

**写法 B：OnUpdate 累计 1 秒**

```lua
self.elapsed = 0
function MyComp:OnUpdate(dt)
    self.elapsed = self.elapsed + dt
    if self.elapsed >= 1 then
        CheckTemperature(self.inst)
        self.elapsed = 0
    end
end
```

**写法 A 略优**：
- scheduler 队列 O(log n) 入出 vs 每帧 30 次空转
- 大量实体时 A 性能更好

**但**——**如果你已经在 OnUpdate 里跑别的代码（移动等），加一个 `if self.elapsed >= 1` 几乎零额外开销**。**没必要为了"风格统一"再开一个 PeriodicTask**。

---

### 8.4.7 老手进阶：性能优化—— 监听 vs 轮询的真实开销

#### 第一步：基准对比

设想：你的 mod 想做"玩家手里拿火把时点燃接近的可燃物"——**每帧检查一次**？

**写法 A：OnUpdate 每帧轮询**

```lua
function MyComp:OnUpdate(dt)
    if self.inst.components.inventory:Has("torch") then
        local x, y, z = self.inst.Transform:GetWorldPosition()
        local ents = TheSim:FindEntities(x, y, z, 2, {"flammable"})
        for _, ent in ipairs(ents) do
            ent.components.burnable:Ignite()
        end
    end
end
```

**问题**：
- **每帧调用 `FindEntities`** —— 内部 spatial query，N log N
- **大部分时间没人在火把附近** —— 99% 帧白忙
- **30 FPS × 多个玩家 × 多个 mod = 显著 CPU**

**写法 B：事件驱动**

```lua
-- 只在"靠近可燃物"事件触发时检查
inst:ListenForEvent("entered_proximity", function(inst, data)
    if data.target:HasTag("flammable") and inst.components.inventory:Has("torch") then
        data.target.components.burnable:Ignite()
    end
end)
```

**问题**：饥荒**没有 entered_proximity 事件**——这是假设。

**写法 C：DoPeriodicTask 0.5 秒**

```lua
self.task = self.inst:DoPeriodicTask(0.5, function(inst)
    if inst.components.inventory:Has("torch") then
        -- 同写法 A 的查询
    end
end)
```

**性能差异**：每帧 30 次 → 每秒 2 次 = **15 倍降低查询频率**。

**对玩家体验影响**：响应延迟从 30ms 增加到 500ms——但"点火"这种**人能接受半秒延迟**——**完全可以**。

> **设计经验**：**"不需要每帧响应的事，永远不要每帧执行"** ——这是 mod 性能优化的**第一原则**。

#### 第二步：何时**必须**用 OnUpdate

**真正必须 OnUpdate 的场景**：
- 移动 / 动画 / 物理（locomotor 是经典）
- 输入响应（mouse、keypress）
- 视觉特效（粒子、闪烁）

**几乎都是"用户能感知 30ms 延迟"的场景**——其他都可以用 PeriodicTask 0.1 ~ 1 秒替代。

#### 第三步：批量 OnUpdate 的优化模式

如果你**必须每帧**给所有实体跑一段逻辑——**集中到一个组件**而不是给每个实体加：

**反例**：每个 prefab 都自己 StartUpdatingComponent —— 一个 update 组件就只跑自己——**N 个实体 N 次 update 组件遍历**。

**正例**：一个全局 manager 组件挂到 TheWorld，**自己维护实体列表**——**1 次 update 调用，内部循环处理 N 个实体**。

**例**：`scripts/components/halloweenmoonmutationmanager.lua`、`scripts/components/birdspawner.lua`——这些"管理器"组件挂在 TheWorld 上，**OnUpdate 一次**遍历所有相关实体。

---

### 8.4.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：用 OnUpdate 做"延时一次性"

**症状**：性能下降；新手特别容易这么写。  
**原因**：见 8.4.6 第三步。  
**修复**：`DoTaskInTime(time, fn)` 完事。

#### 陷阱 2：DoTaskInTime 闭包持有外部对象引用

```lua
-- 危险！
inst:DoTaskInTime(60, function()
    target_pigman:PushEvent("hello")
end)
```

**症状**：60 秒后 target_pigman 已被销毁，`PushEvent` 报错。  
**原因**：闭包捕获的是**引用**——但销毁不通知闭包。  
**修复**：

```lua
inst:DoTaskInTime(60, function(inst)
    if target_pigman:IsValid() then
        target_pigman:PushEvent("hello")
    end
end)
```

或者把 target 存到 inst 上、闭包里查：

```lua
inst._target = target_pigman
inst:DoTaskInTime(60, function(inst)
    if inst._target and inst._target:IsValid() then
        inst._target:PushEvent("hello")
    end
end)
```

#### 陷阱 3：DoPeriodicTask 漏 Cancel

**症状**：组件被禁用后、PeriodicTask 仍在跑——继续修改"应该停止"的状态。  
**原因**：禁用组件**不等于销毁实体** —— 不会触发自动 Cancel。  
**修复**：组件 `Pause/Disable` 时主动 Cancel：

```lua
function MyComp:Disable()
    if self.task ~= nil then
        self.task:Cancel()
        self.task = nil
    end
end
```

参考 `hunger.lua:78-81`：

```78:81:scripts/components/hunger.lua
    if self.updatetask ~= nil then
        self.updatetask:Cancel()
        self.updatetask = nil
    end
```

#### 陷阱 4：StopUpdatingComponent 后本帧仍被调一次

**症状**：自定义组件在某条件触发时调 StopUpdatingComponent，但 OnUpdate 又跑了一次——影响业务。  
**原因**：见 8.4.5 第三步——"延后清理"机制。  
**修复**：OnUpdate 第一行加防御判断：

```lua
function MyComp:OnUpdate(dt)
    if self.stopped then return end
    -- ...
end
```

#### 陷阱 5：用 `DoStaticTaskInTime` 做暂停感知任务，但忘了"暂停时游戏内时间不流逝"

**症状**：mod 想"暂停时也走 5 秒"的 UI 提示——用了 `DoStaticTaskInTime`——结果 UI 是关了，但 mod 内的"游戏内业务"仍按为 5 秒已过——其实游戏内根本没流逝。  
**原因**：`staticScheduler` 走真实时钟，**和游戏内 GetTime() 不一致**。  
**修复**：明确区分"UI 时间"和"游戏内时间"——UI 用 staticScheduler，业务用 scheduler。

#### 陷阱 6：在 OnUpdate 里 PushEvent 给自己——递归触发

```lua
function MyComp:OnUpdate(dt)
    self.inst:PushEvent("mytick", { dt = dt })
end

inst:ListenForEvent("mytick", function(inst, data)
    -- 业务
end)
```

**症状**：每帧 PushEvent + ListenForEvent —— **看起来很统一**——但是**性能差**——比 listener fn 直接调用慢一倍。  
**修复**：**OnUpdate 里直接调业务函数**：

```lua
function MyComp:OnUpdate(dt)
    self:OnTick(dt)
end

function MyComp:OnTick(dt)
    -- 业务
end
```

#### 设计经验三条

**经验 ①：DoTaskInTime > DoPeriodicTask > OnUpdate 的"性能阶梯"**

按性能开销从低到高：
1. **DoTaskInTime / DoPeriodicTask（长周期）** —— 几乎免费
2. **DoPeriodicTask 0.1~1 秒** —— 轻量
3. **OnUpdate（每帧）** —— 最重——慎用

**写代码前先问**：能不能用更稀疏的频率？能就降频。

**经验 ②：批处理优于单兵作战**

不要给每个 prefab 加 `StartUpdatingComponent`——**集中到 TheWorld 上的 manager 组件**——一次 update 处理所有相关实体。

**经验 ③：实体生命周期对齐**

- 任务的回调访问的对象 → 永远先 `IsValid` 检查
- 组件被禁用时 → 主动 Cancel 任务、StopUpdatingComponent
- 实体销毁时 → 信任自动清理（但你的回调里仍要判 IsValid）

---

### 8.4.9 小结

**事件 vs 更新的一句话总结**：**事件 = 被动响应**——零开销但只在 push 时触发；**更新 = 主动周期**——持续跑但能精确控制频率。**两者互补，不互替**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 每帧执行 | `inst:StartUpdatingComponent(self)` + `function MyComp:OnUpdate(dt)` |
| 延时一次 | `inst:DoTaskInTime(5, fn)` |
| 周期任务 | `local t = inst:DoPeriodicTask(1, fn)` |
| 取消周期 | `t:Cancel()` |
| 暂停不停 | `inst:DoStaticTaskInTime(2, fn)` |
| 停 update | `inst:StopUpdatingComponent(self)` |
| 事件触发 | `inst:ListenForEvent("event", fn)` |
| 状态变化 | `inst:WatchWorldState("var", fn)` |

**6 个陷阱排雷顺序**

1. OnUpdate 做延时一次 → 用 DoTaskInTime
2. 闭包持有外部引用 → IsValid 判
3. DoPeriodicTask 漏 Cancel → 主动 Cancel
4. StopUpdatingComponent 当帧仍触发 → 防御判
5. StaticTask 时钟混淆 → 区分 UI/业务
6. OnUpdate 里 PushEvent 给自己 → 直接调函数

**3 条设计经验**

- ① **性能阶梯**——能稀疏不密集
- ② **批处理**——manager 组件集中处理
- ③ **生命周期对齐**——任务里 IsValid 判 + 禁用时 Cancel

**事件系统四件套对比一图**

```
                  瞬时    持续
              ┌─────────────────────────┐
   特定实体   │  ListenForEvent          │
              │   (跨实体只能用)         │
              ├─────────────────────────┤
   全局        │  TheWorld:ListenForEvent │ WatchWorldState
              │  TheWorld:PushEvent      │ TheWorld.state.xxx
              └─────────────────────────┘
   
   主动持续：StartUpdatingComponent / DoPeriodicTask
   主动延时：DoTaskInTime / DoStaticTaskInTime
   主动延时事件：PushEventInTime
```

> **下一节预告**：8.5 节是整章的**收尾速查**——把饥荒**最常用的 50+ 内置事件**按主题整理成索引（玩家生命周期 / 战斗 / 装备 / 库存 / 容器 / 工作 / 食物 / 季节天气 / 状态切换 / 网络）——每个事件附**推送时机**、**data 字段**、**典型监听场景**。读完 8.5，你的 mod 写到任何地方需要"现在 X 发生时让 Y 响应"——都能**秒查到对应事件名**——这是 mod 开发的**生产力工具书**。

## 8.5 常用事件大全（附 events.txt 索引）

### 本节导读

8.1 ~ 8.4 我们把事件系统**机制层面**讲透了——`PushEvent / ListenForEvent / RemoveEventCallback / WatchWorldState` / `OnUpdate` / `DoPeriodicTask`。**剩下最后一块**：**到底有哪些事件可用？分别什么时候推、data 里有什么字段、谁会监听？**

这一节是整章的**速查工具书**——把饥荒里**最常用的 80+ 内置事件**按主题分类整理，每个事件附：

- **推送时机**：什么时候被 push 出来
- **推送源**：实体 (`inst`) 还是世界 (`TheWorld`)
- **data 字段**：listener 收到什么
- **典型监听场景**：哪种 mod / 组件会监听它

> **新手**直接从 8.5.1 快速入门读到 8.5.6——熟悉"玩家生命周期 / 战斗 / 装备库存 / 工作采集 / 食物"这五大类事件就足以覆盖 80% mod 开发场景；**进阶读者**继续看 8.5.7-8.5.8，掌握世界级事件、状态切换事件，能写出"季节切换/天气响应/温度湿度"等系统级 mod；**老手**直接跳 8.5.9-8.5.10 看陷阱与速查表。

> **资源指引**：`scripts/events.txt` 是 Klei 维护的**事件文档原件**——但内容**不完整且年代较远**。**真正可信的来源**是 `scripts/components/` 下各组件的 `PushEvent` 调用——本节速查表就是直接从源码扫出来的。

---

### 8.5.1 快速入门：怎么读懂这本"事件大全"

#### 第一步：每条事件的"四件套"

下面 8.5.2 ~ 8.5.8 的每张表都按以下格式：

| 事件名 | 推送源 | 推送时机 | data 字段 / 推送源代码位置 |

**比如**这条：

| `attacked` | `inst` | `combat:GetAttacked` 时 | `{ attacker, damage, weapon, stimuli, ... }`（`combat.lua:686`） |

**怎么看**：
- **事件名 `"attacked"`** —— `ListenForEvent("attacked", fn)` 用这个字符串
- **推送源 `inst`** —— 推到**被攻击的实体**自己身上 —— 你监听**谁的** attacked 事件，由你决定（自己 / 跨实体）
- **推送时机** —— `combat.lua` 的 `GetAttacked` 函数被调时（**任何战斗伤害都会触发**——刀砍、火烧、自爆都算）
- **data 字段** —— listener 收到这些字段——具体语义见 7.x 战斗章节

#### 第二步：怎么用这本速查表

**最佳工作流**：
1. 写 mod 时遇到"X 发生时让 Y 响应"——**先来本节查 X 对应的事件名**
2. 找到事件名 → **看 data 字段** → **看推送源**
3. 写代码：

```lua
inst:ListenForEvent("X 的事件名", function(inst, data)
    -- 用 data.xxx 字段做业务
end, source)  -- source 由"推送源"决定
```

**找不到？** 对应的需求可能：
- 不是"事件触发型" → 可能要用 `WatchWorldState` 或 `OnUpdate` —— 见 8.3 / 8.4
- 是新引入的 → 8.5 大全只覆盖**通用且稳定**的事件——某些 DLC 专属或 boss 专属事件未列入

#### 第三步：事件名约定速查

| 前缀 | 含义 | 例子 |
| --- | --- | --- |
| `on*` | 状态变化或动作发生（被动） | `onequip` / `onpickup` / `onhitother` |
| `*delta` | 数值变化（连续值） | `healthdelta` / `hungerdelta` / `sanitydelta` |
| `*changed` | 离散值变化 | `phasechanged` / `precipitationchanged` |
| `*tick` | 周期触发（不是真"事件"是定时通知） | `clocktick` / `seasontick` / `weathertick` |
| `start*` / `stop*` | 状态边界 | `startfreezing` / `stoptarget` |
| `entity_*` | 世界级，data 含 inst | `entity_death` / `entity_droploot` |
| `ms_*` | master sim only（仅服务端） | `ms_playerjoined` / `ms_setseason` |

---

### 8.5.2 玩家生命周期事件

> **作用域**：大多推到 `TheWorld` 上（少数推到 player 本身）。**仅服务端 ms_xxx 的客户端听不到**——见 8.2.4。

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `playerentered` | `TheWorld` | 玩家 prefab 激活进入；客户端是"进入网络视野"，主机是"连接" | `inst`（player）（`player_common.lua:851`） |
| `playerexited` | `TheWorld` | 玩家 prefab 去激活离开 | `inst`（player）（`player_common.lua:1172`） |
| `playeractivated` / `playerdeactivated` | `TheWorld` | 玩家激活/去激活的细粒度通知 | `inst`（player）（`player_common.lua:777, 827`） |
| `ms_playerjoined` | `TheWorld`（仅主机） | 玩家**加入服务器**——一次/session | `inst`（player）（`player_common.lua:853`） |
| `ms_playerleft` | `TheWorld`（仅主机） | 玩家**离开服务器** | `inst`（player）（`player_common.lua:1174`） |
| `ms_playerspawn` | `TheWorld`（仅主机） | 玩家 spawn（包括复活） | `inst`（player）（`player_common.lua:2953`） |
| `ms_newplayerspawned` | `TheWorld`（仅主机） | **第一次** spawn | `inst`（player）（`player_common.lua:1479`） |
| `ms_newplayercharacterspawned` | `TheWorld`（仅主机） | 选完角色第一次 spawn | `{ player, mode }`（`playerspawner.lua:298`） |
| `ms_playerdespawnandmigrate` | `TheWorld`（仅主机） | 玩家穿越传送到另一 shard | `{ player, portalid, worldid, x, y, z }`（`SGwilson.lua:20009`） |
| `ms_playerdespawnanddelete` | `TheWorld`（仅主机） | 玩家从世界永久删除（自杀重置） | `inst`（player）（`multiplayer_portal.lua:394`） |

**典型监听场景**：

```lua
-- 给所有新加入的玩家挂监听
TheWorld:ListenForEvent("ms_playerjoined", function(world, player)
    player:ListenForEvent("hungerdelta", OnHungerChange)
end)

-- 玩家断线时清理
TheWorld:ListenForEvent("ms_playerleft", function(world, player)
    player:RemoveEventCallback("hungerdelta", OnHungerChange)
end)
```

**注意 `playerentered` 和 `ms_playerjoined` 的差别**——见 8.2.5 第二步。

---

### 8.5.3 战斗事件

> **作用域**：全部推到 `inst`（攻击者或被攻击者）。**双向推**——一次攻击同时触发"被攻击"和"打中"两套事件。

#### 攻击者收到的事件

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `doattack` | 攻击者 | `combat:DoAttack` 开始时（动画前） | `{ target }`（`combat.lua:808/819`） |
| `onattackother` | 攻击者 | 攻击命中目标后 | `{ target, weapon, projectile, stimuli }`（`combat.lua:1106`） |
| `onhitother` | 攻击者 | 攻击造成伤害后 | `{ target, damage, damageresolved, stimuli, spdamage, weapon, redirected }`（`combat.lua:693`） |
| `onmissother` | 攻击者 | 攻击未命中（target 已无效或闪避） | `{ target, weapon }`（`combat.lua:1088`） |
| `weapontooweak` | 攻击者 | 武器伤害太小 | （无 data）（`combat.lua:1101`） |
| `onareaattackother` | 攻击者 | AOE 攻击命中范围内任一目标 | `{ target, weapon, stimuli }`（`combat.lua:1259, 1271`） |
| `killed` | 攻击者 | 击杀目标 | `{ victim, attacker }`（`combat.lua:652`） |
| `murdered` | 攻击者 | 屠宰物品（杀死库存里的活物） | `{ victim, stackmult }`（`incinerator.lua:32` 等） |

#### 被攻击者收到的事件

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `attacked` | 被攻击者 | 任何战斗伤害到来 | `{ attacker, damage, damageresolved, original_damage, weapon, stimuli, spdamage, redirected, noimpactsound }`（`combat.lua:686`） |
| `blocked` | 被攻击者 | 伤害被装备（盾、护甲）吸收 | `{ attacker, damage, spdamage, original_damage }`（`combat.lua:700`） |
| `death` | 被攻击者 | 健康归零 | `{ cause, afflicter, corpsing }`（`health.lua:590`） |
| `entity_death` | `TheWorld`（仅主机） | 任意实体死亡 | `{ inst, cause, afflicter, corpsing }`（`health.lua:589`） |

#### 战斗目标管理

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `newcombattarget` | 攻击者 | 设新目标 | `{ target, oldtarget }`（`combat.lua:385`） |
| `losttarget` | 攻击者 | 目标失效（消失/死亡） | （无 data）（`combat.lua:336`） |
| `droppedtarget` | 攻击者 | 主动放弃目标 | `{ target }`（`combat.lua:371`） |
| `giveuptarget` | 攻击者 | 因脱战放弃 | `{ target }`（`combat.lua:526`） |

**典型监听场景**：

```lua
-- 给玩家做"攻击时震屏"
ThePlayer:ListenForEvent("onattackother", function(player, data)
    if data.target:HasTag("epic") then
        TheWorld:DoTaskInTime(0, ShakeCamera)
    end
end)

-- 给某 boss 做"反伤"
inst:ListenForEvent("attacked", function(inst, data)
    if data.attacker and data.attacker.components.health then
        data.attacker.components.health:DoDelta(-data.damage * 0.3)
    end
end)
```

---

### 8.5.4 装备 / 库存 / 容器事件

#### 装备相关（推到**装备物品自己**身上）

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `equipped` | 装备 | 被装备到任何 slot | `{ owner }`（`equippable.lua:102`） |
| `unequipped` | 装备 | 从任何 slot 取下 | `{ owner }`（`equippable.lua:122`） |
| `equipskinneditem` | 玩家 | 装备外观皮肤变化 | `name`（`skinner.lua:528`） |
| `unequipskinneditem` | 玩家 | 取下皮肤 | `clothing[type]`（`skinner.lua:522`） |

#### 库存相关（推到**持有者**身上）

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `equip` | 持有者 | 装备到 inventory.equipslots | `{ item, eslot, no_animation }`（`inventory.lua:1271`） |
| `unequip` | 持有者 | 从 equipslots 取下 | `{ item, eslot, slip }`（`inventory.lua:1135`） |
| `itemget` | 持有者 | 收到物品（存到 slot） | `{ slot, item, src_pos }`（`inventory.lua:1044, 937`） |
| `gotnewitem` | 持有者 | 第一次得到此 prefab（成就常用） | `{ item, slot, toactiveitem }`（`inventory.lua:1098, 1016`） |
| `itemlose` | 持有者 | 失去物品 | `{ slot, prev_item, activeitem }`（`inventory.lua:550, 1302, 1312`） |
| `dropitem` | 持有者 | 主动丢弃物品 | `{ item }`（`inventory.lua:740`） |
| `inventoryfull` | 持有者 | 库存满了想塞物品 | `{ item }`（`inventory.lua:1108`） |
| `newactiveitem` | 持有者 | activeitem 变了 | `{ item }`（`inventory.lua:1150`） |
| `setoverflow` | 持有者 | 装上/取下"溢出袋"（背包） | `{ overflow }`（`inventory.lua:1124, 1264`） |

#### 物品自身（推到**物品**自己身上）

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `onpickup` | 物品 | 被捡起 | `{ owner }`（`inventoryitem.lua:352`） |
| `onputininventory` | 物品 | 被放进库存 | `owner`（`inventoryitem.lua:257`） |
| `ondropped` | 物品 | 从库存丢出 | （无 data）（`inventoryitem.lua:289`） |
| `forgetinventoryitem` | `TheWorld` | 物品从所有 inventory 中移除 | `inst`（物品自己）（`inventoryitem.lua:392`） |

#### 容器（箱子、背包、灶台等）

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `onopen` | 容器 | 被打开 | `{ doer }`（`container.lua:588`） |
| `onclose` | 容器 | 被关闭 | `{ doer }`（`container.lua:651`） |
| `itemget` | 容器 | 容器收到物品 | `{ slot, item, src_pos }`（`container.lua:470, 479`） |
| `itemlose` | 容器 | 容器丢失物品 | `{ slot, prev_item }`（`container.lua:942`） |
| `dropitem` | 容器 | 容器内物品被丢出 | `{ item }`（`container.lua:136, 261`） |

#### 套接物品（宝石插槽等）

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `onsocketeditem` | 持有插槽实体 | 插入物品 | `{ item, doer }`（`socketholder.lua:307`） |
| `onunsocketeditem` | 同上 | 拔出物品 | `{ item }`（`socketholder.lua:345`） |

**典型监听场景**：

```lua
-- 给装备做"装上时给 buff"
inst:ListenForEvent("equipped", function(weapon, data)
    if data.owner and data.owner.components.health then
        data.owner.components.health:DoDelta(20)
    end
end)

-- 给玩家做"得到稀有道具时弹通知"
ThePlayer:ListenForEvent("gotnewitem", function(player, data)
    if data.item:HasTag("rare") then
        ShowToast(player, "稀有物品！")
    end
end)
```

---

### 8.5.5 工作 / 采摘 / 建造事件

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `worked` | 被工作的实体 | 任何 workable 动作（CHOP/MINE/HAMMER/DIG）的**每一次**触发 | `{ worker, workleft }`（`workable.lua:149`） |
| `workfinished` | 被工作的实体 | workleft 归零（树倒下/石头采完） | `{ worker }`（`workable.lua:165`） |
| `picked` | 植物 | pickable 被采摘（果子、花、苗） | `{ picker, loot, plant }`（`pickable.lua:575`） |
| `onbuilt` | **被建造的物品** | builder 成功制造一个建筑/物品 | `{ builder, pos }`（`builder.lua:820`） |
| `builditem` | 建造者 | 同一动作的建造者侧 | `{ item, recipe, skin, prototyper }`（`builder.lua:739`） |
| `buildstructure` | 建造者 | 仅建筑 prefab（不是物品） | `{ item, recipe, skin }`（`builder.lua:819`） |
| `unlockrecipe` | 建造者 | 解锁新配方 | `{ recipe }`（`builder.lua:139, 468`） |
| `techtreechange` | 建造者 | 科技树变化（靠近科技台） | `{ level }`（`builder.lua:414`） |
| `consumeingredients` | 建造者 | 制造时消耗材料 | `{ discounted }`（`builder.lua:587`） |
| `consumehealthcost` | 建造者 | 制造时扣血 | （无 data）（`builder.lua:568`） |
| `hungrybuild` | 建造者 | 饥饿不够建造 | （无 data）（`builder.lua:700`） |
| `refreshcrafting` | 建造者 | 制造菜单需要刷新 | （无 data）（`builder.lua:708`） |
| `makerecipe` | 建造者 | 调用 MakeRecipe 时 | `{ recipe }`（`builder.lua:628`） |
| `plantkilled` | `TheWorld` | 植物被烧毁/砍光 | `{ pos, doer, workaction }`（`burnable.lua:313`、`workable.lua:168`） |
| `entity_droploot` | `TheWorld` | 任何 lootdropper 掉落物品 | `{ inst }`（掉落者）（`lootdropper.lua:453`） |

**典型监听场景**：

```lua
-- 玩家砍树时随机掉一根额外原木
ThePlayer:ListenForEvent("onhitother", function(player, data)
    if data.target:HasTag("tree") and math.random() < 0.1 then
        SpawnPrefab("log").Transform:SetPosition(data.target.Transform:GetWorldPosition())
    end
end)

-- 监听"任何树倒下"做世界统计
TheWorld:ListenForEvent("plantkilled", function(world, data)
    print("a plant died at", data.pos)
end)
```

---

### 8.5.6 食物 / 进食事件

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `oneat` | 食用者（玩家/动物） | 吃下食物 | `{ food, feeder }`（`eater.lua:302`） |
| `oneaten` | 被吃的食物 | 食物被吃掉 | `{ eater }`（`edible.lua:210`） |
| `perishchange` | 食物 | 鲜度变化（每次 OnUpdate） | `{ percent }`（`perishable.lua:176, 185, 229`） |
| `forceperishchange` | 食物 | 强制刷新鲜度 | （无 data）（`perishable.lua:9, 16, 22`） |
| `perished` | 食物 | 完全腐败 | （无 data）（`perishable.lua:290`） |
| `hungerdelta` | 玩家 | 饥饿值变化 | `{ oldpercent, newpercent, overtime }`（`hunger.lua:109`） |

**典型监听场景**：

```lua
-- 食物腐败时告知玩家
inst:ListenForEvent("perished", function(food)
    if food.components.inventoryitem and food.components.inventoryitem.owner then
        food.components.inventoryitem.owner.components.talker:Say("一份食物腐败了")
    end
end)

-- 玩家进食时给 buff
ThePlayer:ListenForEvent("oneat", function(player, data)
    if data.food.prefab == "monstermeat" then
        player.components.sanity:DoDelta(-10)
    end
end)
```

---

### 8.5.7 季节 / 天气 / 时钟世界事件

> **作用域**：**全部推到 TheWorld**。但**真正用 mod 时**——直接 `WatchWorldState` 更优（见 8.3）。下面这些事件主要给"想拿到原始 data 的高级 mod"。

#### 时钟

| 事件名 | 时机 | data |
| --- | --- | --- |
| `clocktick` | 每帧（worldstate 监听这个去更新 time） | `{ time, timeinphase, ... }` |
| `cycleschanged` | 每天结束 cycles +1 | `cycles` |
| `phasechanged` | day/dusk/night 切换 | `phase` 字符串 |
| `moonphasechanged2` | 月相切换 | `{ moonphase, waxing }` |
| `nightmareclocktick` / `nightmarephasechanged` | 安魇遗迹梦魇时钟 | 同上 |

#### 季节

| 事件名 | 时机 | data |
| --- | --- | --- |
| `seasontick` | 每帧（worldstate 监听更新季节） | `{ season, progress, elapseddaysinseason, remainingdaysinseason }` |
| `seasonlengthschanged` | 季节长度配置改变 | `{ spring, summer, autumn, winter }` |

#### 天气

| 事件名 | 时机 | data |
| --- | --- | --- |
| `weathertick` | 每帧 | `{ moisture, pop, precipitationrate, snowlevel, lunarhaillevel, lunarhailrate, wetness }` |
| `temperaturetick` | 每帧 | `temperature` |
| `precipitationchanged` | 降水类型变化 | `preciptype`（"none" / "rain" / "snow" / "lunarhail" / "acidrain"） |
| `snowcoveredchanged` | 积雪显示状态变化 | `show` |
| `wetchanged` | 玩家"湿"状态变化 | `wet` |
| `moistureceilchanged` | 湿气阈值变化 | `moistureceil` |

#### 全局事件

| 事件名 | 时机 | data |
| --- | --- | --- |
| `entity_death` | 任意实体死亡（仅主机） | `{ inst, cause, afflicter, corpsing }`（`health.lua:589`） |
| `entity_droploot` | 任意 lootdropper 掉落 | `{ inst }` |
| `forgetinventoryitem` | 物品离开所有库存 | `inst` |
| `nutrientsvision` | 营养视觉开关 | `{ enabled }`（`playervision.lua:59`） |
| `continuefrompause` | 暂停菜单继续 | （无 data）（`playercontroller.lua:3167`） |
| `charliecutscene` | 查理事件开始/结束 | `bool`（`charliecutscene.lua:206, 227`） |
| `shadowrift_opened` | 暗影裂痕开启 | （无 data） |

#### ms_ 仅主机端的"控制类"事件

| 事件名 | 时机 | data | 含义 |
| --- | --- | --- | --- |
| `ms_setseason` | mod / 控制台触发 | `season` | 强制切换季节 |
| `ms_setmoonphase` | 同上 | `{ moonphase, iswaxing }` | 强制切换月相 |
| `ms_locknightmarephase` | 同上 | `phase` 或 nil | 锁/解锁梦魇阶段 |
| `ms_lockmoonphase` | 同上 | `{ lock }` | 锁定月相 |
| `ms_setclocksegs` | 同上 | `{ day, dusk, night }` | 改变白天/黄昏/夜晚长度 |
| `ms_forceprecipitation` | 同上 | `bool` | 强制开始/停止降水 |
| `ms_startlunarhail` | 同上 | （无 data） | 开始月雹 |
| `ms_lightning` / `ms_sendlightningstrike` | 同上 | 位置 | 召唤闪电 |
| `ms_miniquake` | 同上 | `{ rad, num, duration, target }` | 小型地震 |
| `ms_save` | mod / 控制台触发 | （无 data） | 强制存档 |
| `ms_stormchanged` | 同上 | `{ stormtype, setting }` | 风暴变化 |

**典型监听场景**：

```lua
-- 用事件去改变"季节配置"
TheWorld:PushEvent("ms_setseason", "winter")
TheWorld:PushEvent("ms_setclocksegs", { day = 4, dusk = 4, night = 8 })

-- 用 PrefabPostInit 给某 boss 做"看到 charliecutscene 开始"
AddPrefabPostInit("crabking", function(inst)
    if TheWorld.ismastersim then
        inst:ListenForEvent("charliecutscene", function(world, started)
            if started then inst:DoCowering() end
        end, TheWorld)
    end
end)
```

---

### 8.5.8 状态切换事件（Health / Sanity / Hunger / Temperature / Moisture / Burnable / Freezable / Sleeper）

#### 健康

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `healthdelta` | 实体 | 健康值变化 | `{ oldpercent, newpercent, overtime, cause, afflicter, amount }`（`health.lua:642`） |
| `pre_health_setval` | 实体 | 设新血量前 | `{ val, old_health }`（`health.lua:570`） |
| `minhealth` | 实体 | 触底（撑住一击） | `{ cause, afflicter }`（`health.lua:578`） |
| `invincibletoggle` | 实体 | 无敌状态切换 | `{ invincible }`（`health.lua:147`） |
| `startfiredamage` / `stopfiredamage` | 实体 | 开始/结束着火受伤 | `{ low }` / 无（`health.lua:206, 250`） |
| `firedamage` | 实体 | 火焰伤害 tick | （无）（`health.lua:230`） |
| `changefiredamage` | 实体 | 火焰强度变化 | `{ low }`（`health.lua:216, 235`） |
| `startlunarburn` / `stoplunarburn` | 实体 | 月炽伤害切换 | flags / 无（`health.lua:135, 276, 287, 303, 309, 311`） |

#### 精神

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `sanitydelta` | 实体 | 精神变化 | `{ oldpercent, newpercent, overtime, sanitymode }`（`sanity.lua:405`） |
| `gosane` / `goinsane` / `goenlightened` | 实体 | 精神切换到三种状态 | （无 data）（`sanity.lua:416, 423, 429`） |
| `sanitymodechanged` | 实体 | 精神计算模式变化 | `{ mode }`（`sanity.lua:179`） |
| `inducedinsanity` / `inducedlunacy` | 实体 | 强制疯狂/月狂模式 | `val`（`sanity.lua:350, 373`） |

#### 饥饿

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `hungerdelta` | 实体 | 饥饿变化 | `{ oldpercent, newpercent, overtime }`（`hunger.lua:109`） |

#### 温度

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `temperaturedelta` | 实体 | 温度变化 | `{ last, new, hasrate }`（`temperature.lua:175`） |
| `startfreezing` / `stopfreezing` | 实体 | 冻僵开始/结束 | （由 freezable 组件推） |
| `freeze` / `unfreeze` | 实体 | 完全冻结/解冻 | （由 freezable 组件推） |

#### 湿度

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `moisturedelta` | 实体 | 湿度变化 | `{ old, new }`（`moisture.lua:146, 156`） |
| `wetnesschange` | 物品 | 物品本身的湿/干切换（replica） | `iswet`（`inventoryitem_replica.lua:471`） |

#### 燃烧

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `onignite` | 实体 | 开始燃烧 | `{ source, doer }`（`burnable.lua:375`） |
| `onextinguish` | 实体 | 熄灭 | （无 data）（`burnable.lua:492`） |
| `onburnt` | 实体 | 烧毁 | （由 burnable.lua 推，行号见源码） |

#### 睡眠

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `gotosleep` | 实体 | 进入睡眠 | （无 data）（`sleeper.lua:300`） |
| `wakeup` | 实体 | 醒来 | （由 sleeper.lua 在 OnWake 推） |

#### 庇护 / 其他

| 事件名 | 源 | 时机 | data |
| --- | --- | --- | --- |
| `sheltered` | 实体 | 进入/离开庇护 | `{ sheltered, level }`（`sheltered.lua:47, 54`） |
| `feetslipped` | 实体 | 脚下打滑（雪/酸雨） | （无 data）（`slipperyfeet.lua:154`） |
| `acidsizzlingchange` | 物品 | 酸雨腐蚀切换 | `isacidsizzling`（`inventoryitem_replica.lua:482`） |

**典型监听场景**：

```lua
-- 玩家精神归零时触发特殊 buff
ThePlayer:ListenForEvent("goinsane", function(player)
    player.components.health:DoDelta(-20)
end)

-- 物品着火时让玩家说话
inst:ListenForEvent("onignite", function(item, data)
    if item.components.inventoryitem and item.components.inventoryitem.owner then
        item.components.inventoryitem.owner.components.talker:Say("烧起来啦！")
    end
end)
```

---

### 8.5.9 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：用了"已废弃"的事件名

**症状**：mod 监听某事件，但游戏更新后**该事件不再被推**。  
**原因**：Klei 可能重命名或删除事件——比如 `moonphasechanged` 被 `moonphasechanged2` 替换；老 mod 用旧名失效。  
**修复**：**写 mod 时直接读源码确认事件名**——不要用过时教程里的事件名。**本节速查表里的事件名都来自 2024+ 源码**。

#### 陷阱 2：以为 `attacked` 事件 data.attacker 一定有

**症状**：访问 `data.attacker` 报 nil。  
**原因**：自然伤害（火、酸、冰）不一定有"攻击者"——`combat:GetAttacked` 可能传 nil attacker。  
**修复**：

```lua
inst:ListenForEvent("attacked", function(inst, data)
    if data.attacker and data.attacker:IsValid() then
        -- 业务
    end
end)
```

#### 陷阱 3：监听 `equipped` 但没区分 owner 是不是 ThePlayer

**症状**：mod 想"装上某剑给当前玩家加 buff"，但**所有装备这把剑的玩家**都加了 buff。  
**原因**：`equipped` 推到**装备**身上，data.owner 是装备者—— 不一定是当前玩家。  
**修复**：

```lua
inst:ListenForEvent("equipped", function(weapon, data)
    if data.owner == ThePlayer then  -- 客户端
        AddBuff()
    end
end)
```

或者用 `AddPlayerPostInit` 给玩家挂监听，而不是给装备挂。

#### 陷阱 4：`onbuilt` 推到的是**新建造的物品**而不是 builder

**症状**：在 `onbuilt` 监听里 `inst.components.builder` 报 nil。  
**原因**：

```820:820:scripts/components/builder.lua
                prod:PushEvent("onbuilt", { builder = self.inst, pos = pt })
```

`prod` 是**新造出来的**物品，`builder` 是建造者—— **fn 第一个参数 inst = prod**，要拿建造者用 `data.builder`。  
**修复**：

```lua
AddPrefabPostInit("scienceprop1", function(inst)
    inst:ListenForEvent("onbuilt", function(prop, data)
        local builder = data.builder
        -- ...
    end)
end)
```

#### 陷阱 5：`worked` / `workfinished` 重复触发

**症状**：监听 `workfinished` 两次都触发——第一次合理、第二次报错。  
**原因**：某些 workable 实体（biofilter、矿石）会在动画结束、`Remove` 之前**重复进入 work 状态**。  
**修复**：判 `inst:IsValid()` 加自定义标记防重入：

```lua
inst:ListenForEvent("workfinished", function(inst, data)
    if inst._handled_workfinished then return end
    inst._handled_workfinished = true
    -- 业务
end)
```

#### 陷阱 6：监听 `oneat` 想拿食物 prefab，但其实推送时机是**进食结束**之前

**症状**：在 `oneat` 回调里 `data.food.prefab` 是有效的，但 `data.food.components` 已被 `Remove` 引起的清理动作干扰。  
**原因**：`oneat` 在 `eater:Eat` 内部推送——同一个调用栈里**接下来**会调 `food:Remove`。  
**修复**：**回调里只读 prefab、不要访问 components**：

```lua
ThePlayer:ListenForEvent("oneat", function(player, data)
    local prefab = data.food.prefab  -- ok
    local hp = data.food.components.health and data.food.components.health.currenthealth  -- 危险
end)
```

#### 设计经验三条

**经验 ①：永远先查源码再相信文档**

`scripts/events.txt` 是 Klei 的事件文档——**经常滞后于代码**。**写 mod 时**：
1. 用 `rg "PushEvent\(.events_name." scripts/components/` 确认事件名拼写
2. 看推送处的 data 字段
3. 看周边业务逻辑——确认推送时机是"事件之前还是之后"

**经验 ②：能用 `WatchWorldState` 就别用 `phasechanged` 之类的事件**

8.3.5 第二步讲过——**值变化场景永远 worldstate 优于事件**。比如：

- ❌ `TheWorld:ListenForEvent("phasechanged", fn)` —— 注册时不知道当前 phase
- ✅ `inst:WatchWorldState("phase", fn) + 立刻读 TheWorld.state.phase 初始化`

**经验 ③：所有 listener 第一行先做防御**

```lua
inst:ListenForEvent("attacked", function(inst, data)
    if not inst:IsValid() then return end
    if data == nil then return end
    -- 业务
end)
```

不写防御 → 异常路径下 listener 崩溃 → 整个事件链卡住——其他 listener 也跑不起来。**饥荒 listener 是同步串行触发**——一个崩了影响整条链。

---

### 8.5.10 小结

**整章收尾·事件大全**：从玩家生命周期 → 战斗 → 装备库存 → 工作建造 → 食物 → 季节天气 → 状态切换——**80+ 事件分门别类**。**写 mod 时按主题查表**——5 秒内定位到对应事件名 + data 字段 + 监听场景。

**速查跳转索引**

| 主题 | 章节 |
| --- | --- |
| 玩家生命周期 | 8.5.2 |
| 战斗 | 8.5.3 |
| 装备 / 库存 / 容器 | 8.5.4 |
| 工作 / 采摘 / 建造 | 8.5.5 |
| 食物 / 进食 / 腐败 | 8.5.6 |
| 季节 / 天气 / 时钟 | 8.5.7 |
| 状态切换（健康/精神/饥饿/温度/湿度/燃烧/睡眠） | 8.5.8 |
| 陷阱 + 设计经验 | 8.5.9 |

**6 个陷阱速查**

1. 用废弃事件名 → 读源码
2. data.attacker 可能为 nil → 加 IsValid 判
3. equipped owner 不是当前玩家 → 比较 ThePlayer
4. onbuilt inst 是物品不是 builder → 读 data.builder
5. workfinished 重复触发 → 加防重入标记
6. oneat 时 food 即将销毁 → 只读 prefab

**3 条设计经验**

- ① **源码 > 文档**：永远先 grep 源码确认事件名和 data
- ② **worldstate 优先**：值变化用 watch 不用事件
- ③ **listener 防御**：第一行先 IsValid + data ~= nil 判

> **整章总结**：第 8 章 **事件系统**到此完整收尾——
> - **8.1** PushEvent / ListenForEvent / RemoveEventCallback 三件套机制
> - **8.2** 实体事件 vs 世界事件：作用域决策
> - **8.3** WatchWorldState：状态化的事件
> - **8.4** 事件驱动 vs 帧更新：性能 vs 响应性
> - **8.5** 80+ 内置事件速查工具书
>
> 读完整章，你将能在 mod 写到任何"X 发生时让 Y 响应"的需求时——**秒选机制**（事件 / worldstate / OnUpdate）+ **秒查事件名** + **秒写 listener**——这是中级 mod 开发者必备的"事件直觉"。
>
> **下一章预告**：第 9 章 **网络系统** —— 我们将打开**联机版**的根本机制：**主机 / 客户端 / 网络变量 / RPC / replica / classified entity** —— 这是把"单机 mod"升级到"联机 mod 不报错"的关键章节。事件系统 + 网络系统**两手齐下**，你的 mod 就具备了应对任何 mod 场景的能力。