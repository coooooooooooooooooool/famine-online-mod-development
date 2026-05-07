# 第11章 StateGraph 状态机

## 11.1 State、EventHandler、TimelineEvent 的结构

### 本节导读

第 7 章 7.3 节我们把 StateGraph **作为 Action 系统的下游**讲过——`actionhandlers` 怎么把动作映射到 state、`dolongaction` 是什么。但**那只是 SG 系统的一面**——SG 还是**整个角色行为表现层的核心**：每只生物、每个玩家、每个有动画的实体——**几乎全部**通过 SG 驱动动画 / 音效 / tag / 业务回调。

第 11 章我们**把 SG 系统单独拿出来从底层讲透**——本节先看**最小三件套**：

1. **State**——一个独立"动作状态"（如 idle / chop / death）
2. **EventHandler**——状态收到事件时的反应函数
3. **TimeEvent / TimelineEvent**——动画播放过程中"按帧触发"的回调

读完 11.1，你会对 `inst.sg.currentstate` 是什么、它的 `events` 表为什么这么设计、`timeline` 是怎么调度的——**有底层级别的把握**。这是后面 11.2-11.6 各种"高阶模式"的基础。

> **新手**从 11.1.1-11.1.3 起步——理解 SG 三件套结构、State 类 7 大字段、EventHandler/TimeEvent 各自定义；**进阶读者**继续看 11.1.4-11.1.6，深入 onenter/onexit/onupdate/ontimeout 生命周期、events 派发机制、timeline 排序与时间精度；**老手**跳到 11.1.7-11.1.8，看 StateGraph 整体注册 + mod 扩展机制 + 6 个最容易踩的陷阱。

---

### 11.1.1 快速入门：从一段砍树动画看 SG 是什么

#### 第一步：观察玩家砍树时的状态序列

游戏里玩家右键一棵树——

```
[T=0]    玩家 idle
         ↓
[T=0.1]  walk to tree（locomote）
         ↓
[T=1.5]  到达 → chop_pre（举斧头预备）
         ↓
[T=1.6]  chop_loop（砍击中）—— 第 9 帧砍树音效，第 14 帧伤害结算，第 24 帧动画结束
         ↓
[T=2.0]  树倒下 → 进入 chop_pst（收尾动画）
         ↓
[T=2.3]  回到 idle
```

**这一系列状态切换都是 SG 在调度**——每个 state 有自己的：
- 动画：`PlayAnimation("chop_pre")`
- tag：`{ "doing", "busy" }`
- 时长：`SetTimeout(...)`
- 业务回调（接触帧）：`PerformBufferedAction`

#### 第二步：State 不是"普通的 lua table"

State 是个 Class：

```213:262:scripts/stategraph.lua
State = Class(
    function(self, args)
        local info = debug.getinfo(3, "Sl")
        self.defline = string.format("%s:%d", info.short_src, info.currentline)

        assert(args.name, "State needs name")
        self.name = args.name
        self.onenter = args.onenter
        self.onexit = args.onexit
        self.onupdate = args.onupdate
        self.ontimeout = args.ontimeout

        self.tags = {}
        if args.tags then
            for k, v in ipairs(args.tags) do
                self.tags[v] = true
            end
        end

		--#V2C #client_prediction
		if args.server_states ~= nil then
			--client player only
			self.server_states = {}
			for _, v in ipairs(args.server_states) do
				self.server_states[hash(v)] = true
			end
			self.forward_server_states = args.forward_server_states
		else
			--server player only
			self.no_predict_fastforward = args.no_predict_fastforward
		end

        self.events = {}
        if args.events ~= nil then
            for k,v in pairs(args.events) do
                assert(v:is_a(EventHandler), "non-EventHandler in event list")
                self.events[v.name] = v
            end
        end

        self.timeline = {}
        if args.timeline ~= nil then
            for k,v in ipairs(args.timeline) do
                assert(v:is_a(TimeEvent), "non-TimeEvent in timeline")
                table.insert(self.timeline, v)
            end
        end

		table.sort(self.timeline, Chronological)
	end)
```

**关键观察**：
- 构造函数接收一个 `args` table（即 `State{name=..., onenter=..., ...}` 的写法）
- **强制断言**：`name` 必填、`events` 必须是 EventHandler、`timeline` 必须是 TimeEvent —— 类型安全
- **`self.tags` 是 set 而不是 array**——便于 O(1) 查询
- **`self.events` 按事件名索引**——派发时 O(1)
- **`self.timeline` 排序后存** —— 帧顺序播放
- **`args.server_states` / `forward_server_states`** —— 客户端预测专用（11.1 暂不展开）

#### 第三步：典型 state 长什么样

```lua
State{
    name = "chop_loop",
    tags = { "doing", "busy" },
    
    onenter = function(inst)
        inst.AnimState:PlayAnimation("chop_loop")
    end,
    
    timeline = {
        TimeEvent(9 * FRAMES, function(inst)
            inst.SoundEmitter:PlaySound("dontstarve/wilson/use_axe_tree")
        end),
        TimeEvent(14 * FRAMES, function(inst)
            inst:PerformBufferedAction()  -- 业务接触帧
        end),
    },
    
    events = {
        EventHandler("animover", function(inst)
            if not inst:PerformBufferedAction() then
                inst.sg:GoToState("chop_pst")
            else
                inst.sg:GoToState("chop_loop")
            end
        end),
    },
    
    onexit = function(inst)
        -- 退出时清理
    end,
}
```

**5 部分**：
- **name**：状态名（"chop_loop"）—— 唯一标识
- **tags**：状态标签（"doing"、"busy"）—— 影响其他系统的"能不能做某事"判断
- **onenter**：进入时执行（播动画）
- **timeline**：基于动画帧的回调（音效、业务）
- **events**：状态期间监听的事件

> **核心结论**：**State 是"一段角色行为"的完整定义**——动画、声效、tag、事件响应**全部封装在一个 table 里**。SG 的核心功能就是**根据当前 state 调度这些字段**。

---

### 11.1.2 快速入门：State 类的 7 大字段

#### 第一步：完整字段表

| 字段 | 类型 | 必填 | 用途 |
| --- | --- | --- | --- |
| `name` | string | ✓ | 状态名（唯一） |
| `tags` | array of string | ✗ | 状态标签（用于 HasStateTag 查询） |
| `onenter` | function(inst, ...) | ✗ | 进入状态时调用 |
| `onexit` | function(inst) | ✗ | 离开状态时调用 |
| `onupdate` | function(inst, dt) | ✗ | 每帧调用（罕见） |
| `ontimeout` | function(inst) | ✗ | SetTimeout 计时结束时调用 |
| `events` | array of EventHandler | ✗ | 状态期间监听的事件 |
| `timeline` | array of TimeEvent | ✗ | 基于时间的回调 |

#### 第二步：3 个客户端预测字段（高阶）

| 字段 | 用途 |
| --- | --- |
| `server_states` | 客户端版 state，列出对应的服务端 state（用于预测对齐）|
| `forward_server_states` | 服务端切到这些 state 时，客户端继续保持当前 state |
| `no_predict_fastforward` | 服务端 state 不被客户端预测加速 |

> 这些字段在 SGwilson_client.lua 大量使用——客户端 state **明确声明**它对应哪些服务端 state——避免 9.5 节讲过的"双 SG 不一致"问题。

#### 第三步：实例化 State 的标准写法

```lua
local s = State{
    name = "myaction",
    tags = { "busy", "doing" },
    onenter = function(inst, arg)
        inst.AnimState:PlayAnimation("myanim")
        if arg then
            inst.sg.statemem.value = arg
        end
    end,
    onexit = function(inst)
        inst.AnimState:Stop()
    end,
    timeline = {
        TimeEvent(10 * FRAMES, function(inst)
            inst.SoundEmitter:PlaySound("...")
        end),
    },
    events = {
        EventHandler("animqueueover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
}
```

**注意几个细节**：

- **onenter 接收 `...` 多余参数** —— `inst.sg:GoToState("myaction", arg)` 的 arg 会作为第二个参数
- **`inst.sg.statemem`** —— 状态期间的"临时存储"——`onexit` 时自动清空
- **`inst.sg.mem`** —— **持久存储**——跨 state 保留

---

### 11.1.3 快速入门：EventHandler / TimeEvent / FrameEvent / SoundFrameEvent

#### EventHandler

```173:181:scripts/stategraph.lua
EventHandler = Class(
    function(self, name, fn)
        local info = debug.getinfo(3, "Sl")
        self.defline = string.format("%s:%d", info.short_src, info.currentline)
        assert (type(name) == "string")
        assert (type(fn) == "function")
        self.name = string.lower(name)
        self.fn = fn
    end)
```

**用法**：

```lua
EventHandler("attacked", function(inst, data)
    inst.sg:GoToState("hit")
end)
```

**两个参数**：
- **name** —— 事件名字符串（强制小写）
- **fn** —— 回调函数，签名 `(inst, data)`

**注意**：name 会被 `string.lower(name)` —— **大小写不敏感** —— 但事件 PushEvent 时是大小写敏感的——所以推荐**永远全小写**。

#### TimeEvent

```183:191:scripts/stategraph.lua
TimeEvent = Class(
    function(self, time, fn)
        local info = debug.getinfo(3, "Sl")
        self.defline = string.format("%s:%d", info.short_src, info.currentline)
        assert (type(time) == "number")
        assert (type(fn) == "function")
        self.time = time
        self.fn = fn
    end)
```

**用法**：

```lua
TimeEvent(0.5, function(inst)
    inst:DoSomething()
end)
```

**time 是秒数**——0.5 表示进入状态后 0.5 秒触发。

#### FrameEvent —— TimeEvent 的语法糖

```193:195:scripts/stategraph.lua
function FrameEvent(frame, fn)
	return TimeEvent(frame * FRAMES, fn)
end
```

**`FRAMES = 1/30`**（约 0.0333）——所以 `FrameEvent(15, fn)` 等于 `TimeEvent(0.5, fn)` 等于"第 15 帧"。

**为什么用 frame 而不是 time？** —— 动画文件是按帧打的——**精确对齐到帧**比"猜秒数"更准确。

#### SoundFrameEvent —— 音效专用

```203:207:scripts/stategraph.lua
function SoundFrameEvent(frame, sound_event)
    return TimeEvent(frame * FRAMES, function(inst)
        inst.SoundEmitter:PlaySound(sound_event)
    end)
end
```

**用法**：

```lua
SoundFrameEvent(9, "dontstarve/wilson/use_axe_tree")
```

等价于：

```lua
TimeEvent(9 * FRAMES, function(inst)
    inst.SoundEmitter:PlaySound("dontstarve/wilson/use_axe_tree")
end)
```

**专门为播音效优化**——简洁。

#### SoundTimeEvent —— 同款用 time 而非 frame

```197:201:scripts/stategraph.lua
function SoundTimeEvent(time, sound_event)
    return TimeEvent(time, function(inst)
        inst.SoundEmitter:PlaySound(sound_event)
    end)
end
```

---

### 11.1.4 进阶：onenter / onexit / onupdate / ontimeout 生命周期

#### 第一步：4 个生命周期钩子

| 钩子 | 触发时机 | 典型用途 |
| --- | --- | --- |
| `onenter(inst, ...)` | 进入状态时**一次** | 播动画、设 tag、初始化 statemem |
| `onexit(inst)` | 离开状态时**一次** | 清理 listener、停止音效 |
| `onupdate(inst, dt)` | **每帧**调用 | 持续物理 / 持续判定（罕见）|
| `ontimeout(inst)` | `SetTimeout` 计时结束时 | 长动作的"时间到"业务 |

#### 第二步：典型 onenter 模式

```lua
onenter = function(inst, arg)
    -- 1. 停止移动
    inst.components.locomotor:Stop()
    
    -- 2. 播动画
    inst.AnimState:PlayAnimation("myanim")
    
    -- 3. 初始化 statemem
    inst.sg.statemem.value = 0
    
    -- 4. 处理参数
    if arg and arg.target then
        inst.sg.statemem.target = arg.target
    end
    
    -- 5. 如果是限时状态，设 timeout
    inst.sg:SetTimeout(2)
end
```

**注意 `arg`** —— 来自 `GoToState("myaction", arg)` 第二个参数——可以是任意类型。

#### 第三步：典型 onexit 模式

```lua
onexit = function(inst)
    -- 1. 停止音效
    inst.SoundEmitter:KillSound("loop_sound")
    
    -- 2. 清理添加的 tag
    if inst.sg.statemem.added_tag then
        inst:RemoveTag("custom_tag")
    end
    
    -- 3. 解除 listener
    if inst.sg.statemem.listener then
        inst:RemoveEventCallback("...", inst.sg.statemem.listener)
    end
    
    -- 4. 处理 BufferedAction（重要！）
    if inst.bufferedaction == inst.sg.statemem.action then
        inst:ClearBufferedAction()
    end
end
```

**onexit 的两大职责**：
- 清理 **state 期间添加的副作用**（tag、listener、音效）
- 处理 **未完成的 BufferedAction**（避免悬挂）

#### 第四步：onupdate 慎用

```lua
onupdate = function(inst, dt)
    inst.sg.statemem.elapsed = inst.sg.statemem.elapsed + dt
    if inst.sg.statemem.elapsed > 1 then
        -- ...
    end
end
```

**问题**：onupdate **每帧**调用——性能开销大。**99% 场景用 timeline 替代**——timeline 是"事件驱动"，onupdate 是"轮询"。

**仅在以下场景用 onupdate**：
- 持续物理同步（移动、加速度）
- 实时输入响应（玩家按住按钮）
- 不能用 TimeEvent 表达的连续逻辑

#### 第五步：ontimeout 与 SetTimeout

```lua
onenter = function(inst)
    inst.sg:SetTimeout(2)  -- 2 秒后触发 ontimeout
    -- ...
end,

ontimeout = function(inst)
    inst.sg:GoToState("idle")
end,
```

**等价于** `TimeEvent(2, function(inst) inst.sg:GoToState("idle") end)`——但语义更清楚（"这个状态有一个总时长"）。

> **看 7.3.4 的 dolongaction state** —— 它就是用 ontimeout 实现"长动作时长"——典型范例。

---

### 11.1.5 进阶：events 派发与 HandleEvent

#### 第一步：HandleEvent 的核心逻辑

```264:272:scripts/stategraph.lua
function State:HandleEvent(sg, eventname, data)
	if type(data) ~= "table" or data.state == nil or data.state == self.name then
        local handler = self.events[eventname]
        if handler ~= nil then
            return handler.fn(sg.inst, data)
        end
    end
    return false
end
```

**逻辑**：
1. **如果 data 不是 table 或 data.state == nil 或 data.state == 当前状态名** → 走下面
2. **从 self.events 找对应 handler**
3. **找到 → 调 fn(inst, data)**
4. **没找到 → 返回 false**

#### 第二步：data.state 字段的精妙

注意**第 1 行的判断**：

```lua
if type(data) ~= "table" or data.state == nil or data.state == self.name then
```

**意思**：如果 data 是 table 且**指定了 `state` 字段**——只有当前状态名匹配时才处理。

**典型用法**：定时事件**只在某 state 处理**

```lua
-- 推送
inst:PushEvent("delayedaction", { state = "myaction", arg = 42 })

-- 业务方监听
events = {
    EventHandler("delayedaction", function(inst, data)
        -- 只有当前在 myaction 状态时才会到这里
    end),
},
```

**避免**：玩家先 push 了事件、SG 切到别的 state、事件被错误处理。

#### 第三步：state 级 events vs SG 级 events

State 有自己的 `self.events` —— **状态特定**响应。

但 StateGraph 整体也有 `self.events` —— **任何 state 都触发**：

```lua
-- 整个 SG 都监听
local sg_events = {
    EventHandler("attacked", function(inst, data)
        inst.sg:GoToState("hit")
    end),
}

local SG_xxx = StateGraph("xxx", states_table, sg_events, "idle", actionhandlers)
```

**优先级**：state 级 events 先匹配——**没有匹配的事件才回落到 SG 级**。

#### 第四步：常见 event 名

| event 名 | 来源 | 含义 |
| --- | --- | --- |
| `animover` | AnimState 推送 | 动画 PlayAnimation 完成 |
| `animqueueover` | AnimState 推送 | 整个动画队列完成（包括 PushAnimation 的）|
| `attacked` | combat 推送 | 受到攻击 |
| `death` | health 推送 | 健康归零 |
| `locomote` | playercontroller 推送 | 玩家想移动 |
| `doaction` | playercontroller 推送 | 玩家请求执行某动作 |
| `gohome` | brain 推送 | NPC 接到回家命令 |

> 11.4 节会专门讲 commonstates 里这些事件的标准处理模式。

---

### 11.1.6 进阶：timeline 排序与时间精度

#### 第一步：timeline 是排序的

```lua
table.sort(self.timeline, Chronological)
```

```lua
local function Chronological(a, b)
	return a.time < b.time
end
```

**State 构造时**——timeline 自动按 time 升序排序。**对开发者影响**：你不用关心 TimeEvent 的写入顺序——SG 内部按时间触发。

#### 第二步：timeline 的运行机制

`StateGraphInstance.UpdateState` 大概逻辑（不展开源码）：

```
每帧：
  累加 timeinstate
  对每个 timeline event：
    如果 event.time <= timeinstate 且未触发过：
      调 event.fn(inst)
      标记为已触发
```

**意思**：timeline event 是**单次触发**——同一帧内**多个 event 全部触发**——都按顺序调用。

#### 第三步：FRAMES 常量

```lua
FRAMES = 1/30  -- 1 帧 = 1/30 秒
```

**饥荒动画运行在 30 FPS**——所以"第 N 帧" = `N * FRAMES` 秒。

**精度问题**：
- 你写 `TimeEvent(9 * FRAMES, fn)` —— 0.3 秒
- 实际触发时间 ≈ 0.3 秒——**但有 1 帧的误差**（最多 33ms）
- 因为 timeline 检查是每帧一次——**不是连续的**

**对开发者影响**：
- 不要假定 timeline 精确到毫秒
- 要"完全精确" → 用 stategraph 之外的高精度调度（少见）

#### 第四步：timeline 的常见模式

**模式 A：音效在动画的"接触帧"**

```lua
timeline = {
    SoundFrameEvent(9, "dontstarve/wilson/use_axe_tree"),
    TimeEvent(14 * FRAMES, function(inst) inst:PerformBufferedAction() end),
}
```

**模式 B：移除 busy tag**

```lua
timeline = {
    TimeEvent(10 * FRAMES, function(inst)
        inst.sg:RemoveStateTag("busy")
    end),
}
```

**模式 C：循环动画的"触发点"**

```lua
events = {
    EventHandler("animover", function(inst)
        if inst:PerformBufferedAction() then
            inst.sg:GoToState("chop_loop")  -- 继续砍
        else
            inst.sg:GoToState("chop_pst")    -- 收尾
        end
    end),
}
```

---

### 11.1.7 老手进阶：StateGraph 整体注册与 mod 扩展

#### 第一步：StateGraph 类的构造

```274:325:scripts/stategraph.lua
StateGraph = Class( function(self, name, states, events, defaultstate, actionhandlers)
    assert(name and type(name) == "string", "You must specify a name for this stategraph")
    local info = debug.getinfo(3, "Sl")
    self.defline = string.format("%s:%d", info.short_src, info.currentline)
    self.name = name
    self.defaultstate = defaultstate

    --reindex the tables
    self.actionhandlers = {}
    if actionhandlers then
        for k,v in pairs(actionhandlers) do
            assert( v:is_a(ActionHandler),"Non-action handler added in actionhandler table!")
            self.actionhandlers[v.action] = v
        end
    end
	for k,modhandlers in pairs(ModManager:GetPostInitData("StategraphActionHandler", self.name)) do
		for i,v in ipairs(modhandlers) do
			assert( v:is_a(ActionHandler),"Non-action handler added in mod actionhandler table!")
			self.actionhandlers[v.action] = v
		end
	end
    ...
```

**关键观察**：
- **5 个参数**：name, states, events, defaultstate, actionhandlers
- **3 张表 reindex**：actionhandlers 按 action 索引、events 按事件名索引、states 按 state 名索引
- **mod 扩展点**：`ModManager:GetPostInitData("Stategraph...", name)` —— 把 mod 注册的 handler/event/state 合并进来

#### 第二步：标准 SG 文件结构

```lua
-- scripts/stategraphs/SGmycharacter.lua

require("stategraphs/commonstates")

local actionhandlers =
{
    ActionHandler(ACTIONS.CHOP, "chop_pre"),
    ActionHandler(ACTIONS.MINE, "mine"),
    -- ...
}

local events =
{
    EventHandler("attacked", function(inst, data)
        inst.sg:GoToState("hit")
    end),
    EventHandler("death", function(inst)
        inst.sg:GoToState("death")
    end),
}

local states =
{
    State{ name = "idle", ... },
    State{ name = "chop_pre", ... },
    State{ name = "chop_loop", ... },
    State{ name = "chop_pst", ... },
    -- ...
}

return StateGraph("mycharacter", states, events, "idle", actionhandlers)
```

**6 段**：
1. require commonstates（如果用通用状态）
2. 定义 actionhandlers
3. 定义 events
4. 定义 states
5. **return StateGraph(name, states, events, defaultstate, actionhandlers)**

#### 第三步：mod 给现有 SG 加内容

```lua
-- modmain.lua

-- 加 ActionHandler
AddStategraphActionHandler("wilson", ActionHandler(ACTIONS.MYACTION, "myaction"))

-- 加 EventHandler
AddStategraphEvent("wilson", EventHandler("mymod_event", function(inst, data)
    inst.sg:GoToState("special")
end))

-- 加 State
AddStategraphState("wilson", State{
    name = "myaction",
    -- ...
})

-- PostInit 整个 SG
AddStategraphPostInit("wilson", function(sg)
    -- 改 SG 的某些属性
    sg.actionhandlers[ACTIONS.CHOP].deststate = function(inst, action)
        return "myspecial_chop"
    end
end)
```

#### 第四步：mod 扩展机制源码

```289:294:scripts/stategraph.lua
	for k,modhandlers in pairs(ModManager:GetPostInitData("StategraphActionHandler", self.name)) do
		for i,v in ipairs(modhandlers) do
			assert( v:is_a(ActionHandler),"Non-action handler added in mod actionhandler table!")
			self.actionhandlers[v.action] = v
		end
	end
```

`ModManager:GetPostInitData(name, sgname)` —— 拿到所有 mod 注册的 handler——合并到 SG。**同 action 的覆盖**：后注册的覆盖先注册的——**多个 mod 修改同一 action 时只剩最后一个**。

---

### 11.1.8 老手进阶：六个常见陷阱

#### 陷阱 1：onenter 直接调 GoToState

```lua
-- ❌
onenter = function(inst)
    if some_condition then
        inst.sg:GoToState("idle")  -- 不能在 onenter 里 GoToState！
    end
end
```

**原因**：SG 内部正在切到这个 state——onenter 又切——**双切互相干扰**——不可预测行为。  
**修复**：用 `inst:DoTaskInTime(0, ...)` 推迟到下一帧：

```lua
onenter = function(inst)
    if some_condition then
        inst:DoTaskInTime(0, function(inst)
            inst.sg:GoToState("idle")
        end)
    end
end
```

或者**判断在 events 里**——比如 EventHandler("animover", ...)。

#### 陷阱 2：timeline 改变 statemem 但 onenter 没初始化

```lua
-- ❌
timeline = {
    TimeEvent(0.5, function(inst)
        inst.sg.statemem.counter = inst.sg.statemem.counter + 1  -- nil 错
    end),
},
```

**修复**：onenter 永远初始化 statemem：

```lua
onenter = function(inst)
    inst.sg.statemem.counter = 0
end,
```

#### 陷阱 3：onexit 没清理 BufferedAction

**症状**：玩家中途取消砍树——但 BufferedAction 没清——后续动作行为异常。  
**修复**：onexit 标准清理模式：

```lua
onexit = function(inst)
    if inst.bufferedaction == inst.sg.statemem.action then
        inst:ClearBufferedAction()
    end
end,
```

#### 陷阱 4：events 中的事件名没小写

```lua
-- ❌
events = {
    EventHandler("Attacked", fn),  -- name 会被强制小写为 "attacked"
}
```

**实际**：EventHandler 内部 `string.lower(name)` ——所以你写大写也能工作——但**调试时混乱**。**永远小写写法**。

#### 陷阱 5：timeline event 时间不对齐到帧

```lua
-- ❌ 不推荐
TimeEvent(0.317, function(inst) ... end)
```

**问题**：动画帧是 30 FPS——0.317 秒不对齐到任何帧——可能滞后或提前。  
**修复**：用 `FRAMES`：

```lua
TimeEvent(10 * FRAMES, function(inst) ... end)  -- 第 10 帧
```

#### 陷阱 6：state 里访问 inst.bufferedaction.target 时已 invalid

**症状**：玩家在长动作过程中——目标被销毁——SG 的 ontimeout 调 PerformBufferedAction —— `act.target` 是 invalid entity。  
**修复**：永远 IsValid 判：

```lua
ontimeout = function(inst)
    if inst.bufferedaction and inst.bufferedaction.target and inst.bufferedaction.target:IsValid() then
        inst:PerformBufferedAction()
    else
        inst.sg:GoToState("idle")
    end
end
```

#### 设计经验三条

**经验 ①：state 单一职责**

每个 state **只做一件事**：
- ❌ 一个 state 既播 chop 动画又播 mine 动画又播 dig 动画
- ✓ 三个 state 分别叫 chop / mine / dig——actionhandler 分别路由

**经验 ②：timeline 用 FrameEvent 而不是 TimeEvent**

```lua
-- 推荐
FrameEvent(15, fn)  -- 第 15 帧

-- 不推荐
TimeEvent(15 * FRAMES, fn)  -- 等价但更冗长
TimeEvent(0.5, fn)  -- 不对齐到帧
```

**经验 ③：onenter 永远先停 locomotor**

```lua
onenter = function(inst)
    inst.components.locomotor:Stop()  -- 第一行
    inst.AnimState:PlayAnimation("myanim")
    -- ...
end,
```

**否则**：玩家可能在状态期间继续滑动——动画和位置不一致。

---

### 11.1.9 小结

**State 三件套一句话总结**：**State 封装"一段角色行为"（动画 + tag + 生命周期 + events + timeline）；EventHandler 是事件→响应映射；TimeEvent/FrameEvent 是动画帧→业务回调映射**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 定义状态 | `State{ name="x", onenter=fn, ... }` |
| 进入时调 | `onenter = function(inst, ...) end` |
| 退出时调 | `onexit = function(inst) end` |
| 监听事件 | `events = { EventHandler("name", fn) }` |
| 帧触发回调 | `timeline = { FrameEvent(15, fn) }` |
| 帧触发音效 | `timeline = { SoundFrameEvent(9, "...") }` |
| 限时切换 | `inst.sg:SetTimeout(2)` + `ontimeout = fn` |
| 切换 state | `inst.sg:GoToState("name", arg)` |
| 临时存储 | `inst.sg.statemem.x = ...` |
| 持久存储 | `inst.sg.mem.x = ...` |

**6 个陷阱排雷顺序**

1. onenter 直接 GoToState → DoTaskInTime(0)
2. statemem 没初始化 → onenter 第一行
3. onexit 没清 BufferedAction → 标准清理模式
4. event 名大写 → 永远小写
5. TimeEvent 用秒不用 FRAMES → 用 FrameEvent
6. PerformBufferedAction 没判 IsValid → 永远判

**3 条设计经验**

- ① **state 单一职责**
- ② **timeline 用 FrameEvent**
- ③ **onenter 第一行 stop locomotor**

> **下一节预告**：11.2 节我们将深入 **StateTag 与状态属性**——`HasStateTag` / `AddStateTag` / `RemoveStateTag` 的内部机制 + Klei 内置的 30+ tag 含义详解。读完 11.2，你将能精准判断"角色当前能不能做某事"——这是 SG 系统中最常用、也最容易误用的功能。

## 11.2 StateTag 与状态属性

### 本节导读

11.1 节我们看到 State 有个字段叫 `tags`：

```lua
State{
    name = "chop_loop",
    tags = { "doing", "busy" },
    ...
}
```

这些"tag"是什么？为什么 SG 系统专门为它们设计了一套查询 API（`HasStateTag` / `AddStateTag` / `RemoveStateTag`）？

**答案**：StateTag 是**饥荒的"瞬时状态身份"**——告诉游戏其他系统"**这个角色当前能不能做某事**"：

- 玩家在 `idle` 状态 → 能接受新输入
- 玩家在 `chop` 状态 → tag 含 "busy" → 不能再砍别的树（动作互斥）
- 玩家在 `attack` 状态 → tag 含 "attack" → 防御组件可以判断"我在主动攻击"
- 玩家在 `death` 状态 → tag 含 "dead" → 怪物 AI 不再追杀

**这种"tag"机制比直接看 state name 优越**——因为：
- state name 有几百个（chop_pre / chop_loop / chop_pst / mine / dig / build / ...）
- tag 只有 ~30 个（大类）
- **判断"角色是不是 busy"** 用 tag 直接 O(1)；用 state name 列表要枚举数十个

这一节我们把 SG 的 tag 系统讲透——从机制到内置 tag 含义到 mod 自定义模式。

> **新手**从 11.2.1-11.2.3 起步——理解 StateTag 存在意义、3 个核心 API、`SGTagsToEntTags` 双向同步机制；**进阶读者**继续看 11.2.4-11.2.6，深入 30+ 内置 tag 含义、SG 切换时的"擦除-重建"、复合查询 API；**老手**跳到 11.2.7-11.2.8，看 mod 自定义 StateTag 实战 + 6 个最容易踩的坑。

---

### 11.2.1 快速入门：从一次"按攻击键"看 StateTag 是什么

#### 第一步：玩家攻击的"互斥逻辑"

游戏里玩家**按了一次攻击键**——SG 切到 `attack` state——**这时如果玩家又按攻击键**——会发生什么？

**期望行为**：
- 当前的 attack 动画**不能被打断**——直到完成
- 第二次按键被**忽略**（或者排队）

**怎么实现这个判断？**

PlayerController 的处理大致是：

```lua
function PlayerController:DoAttack(...)
    if self.inst.sg:HasStateTag("attack") then
        return  -- 当前在攻击中，忽略这次输入
    end
    -- 否则正常处理
end
```

**`HasStateTag("attack")`** —— **O(1) 查询**——比检查 "currentstate name 是否在某 list 里"快得多。

#### 第二步：tag 不只用于"互斥"

| tag | 含义 | 谁查询 |
| --- | --- | --- |
| `busy` | 角色正忙（砍/挖/建造） | playercontroller 拒绝新输入 |
| `attack` | 角色正在攻击 | combat 拒绝再次攻击 |
| `moving` | 角色正在移动 | 各种系统判断"角色在动" |
| `idle` | 角色空闲 | 各种系统判断"可以干新活" |
| `nopredict` | 客户端不预测此状态 | playercontroller 客户端 |
| `sleeping` | 角色在睡觉 | brain 不打架、health 缓恢复 |
| `dead` | 角色已死 | health/combat 拒绝继续操作 |

**几乎所有"角色当前能不能做某事"的判断都靠 tag**。

#### 第三步：tag 是"瞬时"的

**对比 entity 的普通 tag（`inst:AddTag`/`HasTag`）**：

| 维度 | entity tag | state tag |
| --- | --- | --- |
| 持久性 | 直到显式 RemoveTag | 仅当前 state |
| 设置方式 | `inst:AddTag("hostile")` | State 的 `tags` 字段或 `sg:AddStateTag` |
| 离开状态时 | 保留 | **自动清空** |
| 用途 | 实体身份（怪物/物品/玩家） | 行为状态（busy/attack/...）|

**关键**：state tag **进入 state 时建立、离开 state 时清空**——它是"瞬时身份"。

---

### 11.2.2 快速入门：3 个核心 API

#### 第一步：API 速查

```602:618:scripts/stategraph.lua
function StateGraphInstance:AddStateTag(tag)
    self.tags[tag] = true
    if SGTagsToEntTags[tag] and (TheWorld.ismastersim or self.inst.Network == nil) then
        self.inst:AddTag(tag)
    end
end

function StateGraphInstance:RemoveStateTag(tag)
    self.tags[tag] = nil
    if SGTagsToEntTags[tag] and (TheWorld.ismastersim or self.inst.Network == nil) then
        self.inst:RemoveTag(tag)
    end
end

function StateGraphInstance:HasStateTag(tag)
	return self.tags[tag] == true
end
```

| API | 用途 |
| --- | --- |
| `inst.sg:HasStateTag(tag)` | 查询当前 state 是否有该 tag |
| `inst.sg:AddStateTag(tag)` | 当前 state 加 tag |
| `inst.sg:RemoveStateTag(tag)` | 当前 state 删 tag |
| `inst.sg:HasAnyStateTag(...)` | 任一 tag 存在则返回 true（11.2.6 节）|
| `inst.sg:HasAllStateTags(...)` | 所有 tag 都存在则返回 true |

#### 第二步：3 种"加 tag"路径

**路径 A：在 State 定义里声明（最常见）**

```lua
State{
    name = "chop",
    tags = { "doing", "busy" },  -- 进入时自动加这两个 tag
    ...
}
```

**路径 B：onenter 里动态加**

```lua
State{
    name = "myaction",
    tags = { "doing" },
    onenter = function(inst, arg)
        if arg and arg.is_special then
            inst.sg:AddStateTag("special")  -- 条件加 tag
        end
    end,
}
```

**路径 C：timeline 里某帧后加 / 删**

```lua
timeline = {
    TimeEvent(10 * FRAMES, function(inst)
        inst.sg:RemoveStateTag("busy")  -- 第 10 帧后不再 busy（可被打断）
    end),
}
```

**路径 C 是非常常见的模式**——**典型应用**：动作的"前 1/3 是 busy 不能打断 / 后 2/3 可被打断"。

#### 第三步：HasStateTag 的 3 个使用场景

**场景 1：组件判断状态**

```lua
function Combat:DoAttack(target)
    if self.inst.sg:HasStateTag("busy") then
        return  -- 正忙，不能攻击
    end
    if self.inst.sg:HasStateTag("attack") then
        return  -- 已经在攻击
    end
    -- 正常攻击
end
```

**场景 2：state 内部条件分支**

```lua
events = {
    EventHandler("attacked", function(inst, data)
        if not inst.sg:HasStateTag("invisible") then  -- 隐身状态被攻击不切到 hit
            inst.sg:GoToState("hit")
        end
    end),
}
```

**场景 3：mod 业务**

```lua
inst:DoPeriodicTask(1, function(inst)
    if inst.sg:HasStateTag("sleeping") then
        inst.components.health:DoDelta(2)  -- 睡眠回血
    end
end)
```

---

### 11.2.3 快速入门：`SGTagsToEntTags` 双向同步机制

#### 第一步：源码 SGTagsToEntTags 表

```501:523:scripts/stategraph.lua
local SGTagsToEntTags =
{
    ["attack"] = true,
    ["autopredict"] = true,
    ["busy"] = true,
    ["dirt"] = true,
    ["doing"] = true,
    ["fishing"] = true,
    ["flight"] = true,
    ["hiding"] = true,
    ["idle"] = true,
    ["invisible"] = true,
    ["lure"] = true,
    ["moving"] = true,
    ["nibble"] = true,
    ["noattack"] = true,
    ["nopredict"] = true,
    ["pausepredict"] = true,
    ["sleeping"] = true,
    ["working"] = true,
    ["boathopping"] = true,
	["shouldautopausecontrollerinventory"] = true,
}
```

**这 21 个 tag 是"白名单"** —— SG 切换时**自动同步到 entity tag**。

#### 第二步：双向同步是什么意思？

回看 `AddStateTag`：

```lua
function StateGraphInstance:AddStateTag(tag)
    self.tags[tag] = true
    if SGTagsToEntTags[tag] and (TheWorld.ismastersim or self.inst.Network == nil) then
        self.inst:AddTag(tag)  -- ★ 同步到 entity tag
    end
end
```

**机制**：
- 加 state tag "busy" → **同时**也调 `inst:AddTag("busy")`
- 删除时同样反向同步

**目的**：让"不在乎 SG 内部的代码"也能查到这个 tag——比如：

```lua
-- 在某 prefab 的某个事件 handler 里
if inst:HasTag("busy") then
    -- 玩家正忙
end
```

**不需要写 `inst.sg:HasStateTag("busy")`**——因为已经同步成 entity tag——`HasTag` 也能查到。

#### 第三步：哪些 tag 同步、哪些不同步？

**只有 SGTagsToEntTags 里列出的 21 个 tag 才会同步**——其他 tag 不会。

**举例**：
- ✓ `tags = { "busy", "doing" }` —— 同步
- ✗ `tags = { "myspecialtag" }` —— 不同步（mod 自定义 tag 不在白名单）

**为什么不全同步？** —— 性能 + 命名空间隔离——只让"系统级"tag 跨 SG/entity——避免污染 entity tag 系统。

#### 第四步：mod 自定义 tag 该怎么办？

**两个选择**：

**选择 A**：用 SG 内部 tag（`HasStateTag` 查询）

```lua
State{ name = "myaction", tags = { "myspecialtag" } }

-- 业务方
if inst.sg:HasStateTag("myspecialtag") then
    -- ...
end
```

**选择 B**：手动同步——onenter 加 entity tag、onexit 删

```lua
State{
    name = "myaction",
    tags = { "doing" },
    onenter = function(inst)
        inst:AddTag("myspecialtag")
    end,
    onexit = function(inst)
        inst:RemoveTag("myspecialtag")
    end,
}
```

**推荐 A**——除非你**真的需要**别的代码用 `inst:HasTag` 查询。

#### 第五步：客户端 vs 服务端的同步

```lua
if SGTagsToEntTags[tag] and (TheWorld.ismastersim or self.inst.Network == nil) then
    self.inst:AddTag(tag)
end
```

**判断**：仅在**主机端**或**没有 Network 组件**（即非联机实体）时才同步到 entity tag。

**原因**：联机模式下，entity tag 由网络同步 —— 客户端**不应该自己改 tag**——会和服务端不一致。

**对开发者影响**：客户端 SG 的 AddStateTag 仅记在 self.tags 里、**不**改 entity——HasStateTag 仍然有效——但 HasTag（不带 sg 前缀）拿不到。

---

### 11.2.4 进阶：常见 30+ 内置 tag 含义详解

#### 第一步：核心行为状态 tag

| tag | 含义 | 查询频次 |
| --- | --- | --- |
| `idle` | 空闲——准备接受新输入 | 极高 |
| `busy` | 忙——拒绝新输入 | 极高 |
| `doing` | 正在执行长动作 | 高 |
| `moving` | 正在移动 | 高 |
| `working` | 正在工作（砍/挖/敲）| 中 |
| `attack` | 正在攻击 | 高 |
| `noattack` | 在此状态下被攻击不算（如鬼魂）| 低 |

#### 第二步：动作类型 tag

| tag | 含义 |
| --- | --- |
| `chopping` | 砍树 |
| `mining` | 采矿 |
| `digging` | 挖掘 |
| `hammering` | 锤击 |
| `casting` | 施法 |
| `casting_long_action` | 长施法 |
| `eating` | 进食 |
| `cooking` | 烹饪 |
| `building` | 建造 |
| `talking` | 说话 |
| `fishing` | 钓鱼 |
| `boating` / `sailing` | 在船上 |

#### 第三步：身体姿态 tag

| tag | 含义 |
| --- | --- |
| `dead` | 已死亡 |
| `sleeping` | 睡眠中 |
| `notalking` | 不能说话（哑）|
| `nodangle` | 不能挂载 |
| `nointerrupt` | 不能被打断 |
| `frozen` | 被冻结 |
| `flight` | 飞行 |
| `hiding` | 隐藏 |
| `invisible` | 隐身 |

#### 第四步：客户端预测 tag

| tag | 含义 |
| --- | --- |
| `nopredict` | 客户端不预测此状态 |
| `autopredict` | 客户端自动预测同步 |
| `pausepredict` | 暂停预测 |
| `keep_pocket_rummage` | 保持口袋翻找状态 |

#### 第五步：特殊 tag

| tag | 含义 |
| --- | --- |
| `slowaction` | 慢速动作（影响动画播放速度）|
| `nomorph` | 不能变形 |
| `nostatue` | 不能变石像 |
| `lure` | 钓鱼诱饵中 |
| `nibble` | 钓鱼咬钩 |
| `dirt` | 脚下溅起尘土 |
| `playercontroller` | 玩家控制器活跃 |
| `boathopping` | 船间跳跃中 |

> 完整列表可以 grep `:AddStateTag(` 在 SGwilson.lua 里查所有用到的 tag。

---

### 11.2.5 进阶：tag 在 SG 切换时的"擦除-重建"

#### 第一步：GoToState 内部源码

```529:600:scripts/stategraph.lua
function StateGraphInstance:GoToState(statename, params)
    local state = self.sg.states[statename]
    ...
    if self.currentstate ~= nil and self.currentstate.onexit ~= nil then
        self.currentstate.onexit(self.inst, statename)
    end
    ...
    self.statemem = {}
	self.lasttags = self.tags
    self.tags = {}                       -- ★ 清空 tags
    if state.tags ~= nil then
        for k, v in pairs(state.tags) do
            self.tags[k] = true            -- ★ 重建 tags
        end
    end
    if TheWorld.ismastersim or self.inst.Network == nil then
        for k, v in pairs(SGTagsToEntTags) do
            if self.tags[k] then
                self.inst:AddTag(k)
            else
                self.inst:RemoveTag(k)     -- ★ 同步 entity tag
            end
        end
    end
    ...
```

**关键步骤**：
1. **调 onexit**（旧 state）
2. **保留 lasttags = 旧 self.tags**（用于过渡）
3. **self.tags = {}** —— 清空
4. **重建** state.tags 里声明的 tag
5. **同步到 entity tag**：遍历 SGTagsToEntTags 白名单，按需加/删
6. 最后调 onenter（新 state）

#### 第二步：动态修改的 tag 怎么办？

```lua
-- v1：state 定义只声明初始 tag
State{
    name = "myaction",
    tags = { "doing", "busy" },
    
    onenter = function(inst)
        inst.sg:AddStateTag("special")  -- 动态加的——但只在当前 state 期间有效
    end,
}
```

**注意**：**动态加的 "special" tag** —— **下次 GoToState 时会被清空**——因为 self.tags = {} 清空了。**符合预期**。

#### 第三步：lasttags 的作用

```lua
self.lasttags = self.tags  -- 切换前保存
self.tags = {}              -- 重建
```

**`lasttags`** 是"旧 state 的 tag 集合"——**在 onenter 期间可访问**——用于"基于旧状态"做条件判断：

```lua
onenter = function(inst)
    if inst.sg.lasttags and inst.sg.lasttags["sleeping"] then
        -- 我刚从睡眠醒来
        inst.AnimState:PlayAnimation("wake")
    else
        inst.AnimState:PlayAnimation("idle")
    end
end,
```

**onenter 结束后** —— `self.lasttags = nil`（最后一行 `self.lasttags = nil` 见 11.2 第一步源码末尾）。

#### 第四步：tag 的"瞬时性"完整图

```
[T-3] state A 进入：tags = {a, b}（来自 state 定义）
[T-2] 运行中加 "x"：tags = {a, b, x}
[T-1] 运行中加 "y"：tags = {a, b, x, y}
[T-0] GoToState("B")：
       onexit(A)
       lasttags = {a, b, x, y}
       tags = {} → {c, d}（来自 state B 定义）
       同步 entity tag
       onenter(B) (lasttags 仍可用)
       lasttags = nil
[T+1] state B 中：tags = {c, d}
```

**关键：动态加的 x、y 没有"持久跨 state"——每个 state 的 tag 是独立的快照**。

---

### 11.2.6 进阶：复合查询 `HasAnyStateTag` / `HasAllStateTags`

#### 第一步：源码

```616:660:scripts/stategraph.lua
function StateGraphInstance:HasStateTag(tag)
	return self.tags[tag] == true
end

function StateGraphInstance:HasAnyStateTag(...)
	local tags = select(1, ...)
	if type(tags) == "table" then
		for i, v in ipairs(tags) do
			if self.tags[v] then
                ...
```

#### 第二步：复合查询的两种模式

**模式 1**：可变参数

```lua
if inst.sg:HasAnyStateTag("busy", "attack", "casting") then
    return  -- 任一 true 就拒绝
end
```

**模式 2**：传 array

```lua
local interrupt_tags = { "busy", "attack", "casting" }
if inst.sg:HasAnyStateTag(interrupt_tags) then
    return
end
```

**HasAllStateTags 同理**——所有 tag 都存在才返回 true。

#### 第三步：典型用法

**场景**："角色能不能开始一段新动作？"

```lua
local function CanStartAction(inst)
    return not inst.sg:HasAnyStateTag("busy", "attack", "casting", "dead", "frozen")
end
```

**场景**："角色当前是不是在所有"工作"类状态？"

```lua
if inst.sg:HasAnyStateTag("chopping", "mining", "digging", "hammering") then
    inst.components.workmaster:GiveExperience(0.1)
end
```

**场景**："玩家在睡觉且没受攻击？"

```lua
if inst.sg:HasAllStateTags("sleeping", "noattack") then
    inst.components.health:DoDelta(5)  -- 安全睡眠回血
end
```

#### 第四步：性能考虑

`HasStateTag` 是 **O(1)**（table[key] 查询）。

`HasAnyStateTag(...)` 是 **O(N)**（N = 传入的 tag 数）—— 通常 N <= 5——仍然很快。

**性能不是问题**——但**频繁调用**（如每帧）需要警惕——优先用单 tag 快速判断。

---

### 11.2.7 老手进阶：mod 自定义 StateTag 实战

#### 第一步：场景——mod 加"魔法附身"机制

mod 想做：玩家施放某法术 → 进入"附身状态" → 30 秒内攻击有 50% 暴击 + 移动速度 +20% + 不能再施放法术。

**用 SG tag 表达这个状态**——加一个自定义 tag `"possessed"`。

#### 第二步：定义 mod state

```lua
-- mymod/scripts/stategraphs/SGactions_possess.lua
local possess_states = {
    State{
        name = "possess_action",
        tags = { "doing", "casting" },
        
        onenter = function(inst)
            inst.AnimState:PlayAnimation("research_pre")
            inst.AnimState:PushAnimation("research_loop", false)
            inst.AnimState:PushAnimation("research_pst", false)
            -- 进入施法 state
        end,
        
        timeline = {
            TimeEvent(30 * FRAMES, function(inst)
                inst:PerformBufferedAction()  -- 触发 ACTIONS.MYPOSSESS
            end),
        },
        
        events = {
            EventHandler("animqueueover", function(inst)
                inst.sg:GoToState("idle")
            end),
        },
    },
}

return possess_states
```

#### 第三步：用 entity tag + 定时器实现"持续 buff"

由于 30 秒 buff **跨多个 state**——不能用 SG state tag（state 切换会清掉）。**用 entity tag**：

```lua
-- mymod/scripts/components/possessbuff.lua
local PossessBuff = Class(function(self, inst)
    self.inst = inst
    self.duration = 30
end)

function PossessBuff:Apply()
    self.inst:AddTag("possessed")  -- entity tag，持久
    self.inst.components.combat:SetDamageMultModifier("possessed", 2)  -- 暴击
    self.inst.components.locomotor:SetExternalSpeedMultiplier(self.inst, "possessed", 1.2)
    
    -- 30 秒后自动移除
    self.task = self.inst:DoTaskInTime(self.duration, function()
        self:Remove()
    end)
end

function PossessBuff:Remove()
    self.inst:RemoveTag("possessed")
    self.inst.components.combat:RemoveDamageMultModifier("possessed")
    self.inst.components.locomotor:RemoveExternalSpeedMultiplier(self.inst, "possessed")
    if self.task then self.task:Cancel(); self.task = nil end
end

return PossessBuff
```

#### 第四步：业务方查询

```lua
-- 战斗组件里
if inst:HasTag("possessed") then
    damage = damage * 2  -- 暴击
end

-- 玩家想再施法时
local function CanCastSpell(inst)
    return not inst:HasTag("possessed")
end
```

**注意**：用 **entity tag** `inst:HasTag("possessed")`——不是 state tag——因为 buff 是持续 30 秒的，跨多个 state。

#### 第五步：什么时候用 state tag、什么时候用 entity tag？

| 情况 | 用什么 |
| --- | --- |
| 当前一段动作的属性（busy/attack）| state tag |
| 角色身上的持续 buff（30 秒持续）| entity tag |
| 角色的永久身份（hostile/player/monster）| entity tag |
| 当前姿态的临时标记（隐身/飞行）| state tag |

---

### 11.2.8 老手进阶：六个常见陷阱

#### 陷阱 1：用 state tag 表达"持续 buff"

**症状**：mod 给玩家加"持续 30 秒攻击力 buff"——用了 SG state tag——但玩家随便 GoToState（移动、攻击）—— state 切换 → tag 被清空 → buff 失效。  
**修复**：跨 state 的状态用 entity tag，**不要**用 state tag。

#### 陷阱 2：onenter 里用 lasttags 但忘了 nil 判

```lua
-- ❌
onenter = function(inst)
    if inst.sg.lasttags["sleeping"] then  -- nil 索引
        ...
    end
end
```

**原因**：第一次进入某 state 时（比如玩家刚 spawn）—— lasttags 可能是 nil。  
**修复**：永远先判：

```lua
if inst.sg.lasttags and inst.sg.lasttags["sleeping"] then
    ...
end
```

#### 陷阱 3：动态加的 tag 期望"持久"

```lua
-- ❌
onenter = function(inst)
    inst.sg:AddStateTag("special")
end,

-- 期望切到下一个 state 后还有 "special" → 错！
```

**原因**：GoToState 自动清空 tags。  
**修复**：要持久就用 entity tag，或者**所有相关 state 的定义里都声明 "special"**。

#### 陷阱 4：mod 自定义 tag 期望被同步到 entity

**症状**：mod 在 SG state 用 `tags = { "myspecialtag" }`——业务方 `inst:HasTag("myspecialtag")` 查不到。  
**原因**：myspecialtag 不在 SGTagsToEntTags 白名单里——不会同步。  
**修复**：要用 `inst.sg:HasStateTag("myspecialtag")`，或者 onenter/onexit 手动同步：

```lua
onenter = function(inst) inst:AddTag("myspecialtag") end,
onexit = function(inst) inst:RemoveTag("myspecialtag") end,
```

#### 陷阱 5：HasStateTag 在 SG 不存在时 panic

**症状**：组件代码 `inst.sg:HasStateTag("busy")` —— 报 nil 索引。  
**原因**：实体没有 SG —— inst.sg 是 nil。  
**修复**：

```lua
if inst.sg and inst.sg:HasStateTag("busy") then
    ...
end
```

或者用 entity tag —— `inst:HasTag("busy")` —— 会自动从 SG 同步过来——**但仅 SGTagsToEntTags 白名单内的 tag**才同步。

#### 陷阱 6：tag 大小写不一致

```lua
tags = { "Busy" }  -- ❌ 大写
inst.sg:HasStateTag("busy")  -- 查不到
```

**原因**：state tag 是字符串字面量——大小写敏感——**不像 EventHandler 会自动小写**。  
**修复**：永远小写。

#### 设计经验三条

**经验 ①：state tag 命名 lower_snake_case**

```lua
tags = { "casting_spell", "long_action" }  -- ✓
tags = { "CastingSpell" }                   -- ✗
```

**经验 ②：mod 自定义 tag 加前缀**

```lua
tags = { "doing", "mymod_special" }  -- 避免和别的 mod 冲突
```

**经验 ③：判断"能不能做某事"用 HasAnyStateTag**

```lua
-- ✓
if inst.sg:HasAnyStateTag("busy", "attack", "dead") then
    return
end

-- ✗ 三个独立判断（冗长）
if inst.sg:HasStateTag("busy") or inst.sg:HasStateTag("attack") or inst.sg:HasStateTag("dead") then
    return
end
```

---

### 11.2.9 小结

**StateTag 一句话总结**：**state tag 是"角色当前能不能做某事"的瞬时标记——SGTagsToEntTags 白名单内的 21 个 tag 自动同步到 entity tag——其他靠 HasStateTag 查询**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 声明 state tag | `State{ tags = {"busy", "doing"} }` |
| 动态加 | `inst.sg:AddStateTag("x")` |
| 动态删 | `inst.sg:RemoveStateTag("x")` |
| 查询单个 | `inst.sg:HasStateTag("busy")` |
| 查询任一 | `inst.sg:HasAnyStateTag("busy", "attack")` |
| 查询全部 | `inst.sg:HasAllStateTags("doing", "casting")` |
| 跨 state 持续 | 用 entity tag `inst:AddTag` / `inst:HasTag` |

**21 个白名单 tag**：`attack` / `autopredict` / `busy` / `dirt` / `doing` / `fishing` / `flight` / `hiding` / `idle` / `invisible` / `lure` / `moving` / `nibble` / `noattack` / `nopredict` / `pausepredict` / `sleeping` / `working` / `boathopping` / `shouldautopausecontrollerinventory`

**6 个陷阱排雷顺序**

1. 用 state tag 表达持续 buff → 用 entity tag
2. lasttags 没判 nil → 永远先判
3. 动态加 tag 期望持久 → state 定义里声明
4. mod tag 期望自动同步 → 手动 AddTag / 加白名单是不可能的
5. HasStateTag 没判 inst.sg nil → 永远先判
6. tag 大小写 → 永远小写

**3 条设计经验**

- ① **state tag lower_snake_case 命名**
- ② **mod tag 加前缀**
- ③ **复合判断用 HasAnyStateTag**

> **下一节预告**：11.3 节我们将深入 **动画播放与关键帧回调** —— `PlayAnimation` / `PushAnimation` / `animover` / `animqueueover` 等核心动画 API + AnimState 与 SG 的协作模式。读完 11.3，你将能精确控制 mod 角色的动画时序——这是把"动画素材"变成"真实玩法"的关键。

## 11.3 动画播放与关键帧回调

### 本节导读

11.1 / 11.2 我们把 SG 的"逻辑层"讲透了——State / EventHandler / TimeEvent / StateTag。但 SG 的另一半是**视觉层**——**动画播放**。每个 state 几乎都要做一件事：

```lua
inst.AnimState:PlayAnimation("chop_loop")
```

**这一行做了什么？** 内部怎么调度动画帧？什么时候触发 `animover` 事件？怎么连续播多段动画？怎么暂停/加速？怎么换装（替换装备贴图 symbol）？

**这是动画驱动业务的核心机制**——也是 mod 添加自定义角色 / 自定义动作时**最容易翻车**的环节。

这一节我们把动画系统讲透——从最基础的 PlayAnimation 到高阶的 symbol 替换、动画速率、关键帧回调。

> **新手**从 11.3.1-11.3.3 起步——理解 AnimState 是什么、PlayAnimation/PushAnimation 区别、animover/animqueueover 事件触发时机；**进阶读者**继续看 11.3.4-11.3.6，深入动画时序 API（IsCurrentAnimation / GetCurrentAnimation / SetTime / AnimDone）、symbol 替换与外观叠加、动画速率与暂停；**老手**跳到 11.3.7-11.3.8，看动画驱动业务的 3 种模式 + 6 个最容易踩的坑。

---

### 11.3.1 快速入门：从一段砍树动画看 AnimState 是什么

#### 第一步：观察玩家砍树时的"动画 + 业务"分离

砍树 state（11.1.1 节看过的简化版）：

```lua
State{
    name = "chop_loop",
    tags = { "doing", "busy" },
    
    onenter = function(inst)
        inst.AnimState:PlayAnimation("chop_loop")  -- ★ 动画
    end,
    
    timeline = {
        TimeEvent(9 * FRAMES, function(inst)
            inst.SoundEmitter:PlaySound("...")  -- 第 9 帧音效
        end),
        TimeEvent(14 * FRAMES, function(inst)
            inst:PerformBufferedAction()           -- 第 14 帧业务
        end),
    },
    
    events = {
        EventHandler("animover", function(inst)    -- 动画结束事件
            inst.sg:GoToState("idle")
        end),
    },
}
```

**4 处都和动画相关**：
- **`PlayAnimation("chop_loop")`** —— 让 AnimState 开始播放
- **`TimeEvent(9 * FRAMES, ...)`** —— 第 9 帧触发回调（基于 SG 计时）
- **`TimeEvent(14 * FRAMES, ...)`** —— 第 14 帧触发回调
- **`EventHandler("animover", ...)`** —— **动画播放完成后**触发

**SG 的核心职责**：把"动画时间轴"和"游戏逻辑"对齐——按帧触发音效、按帧触发业务、动画完成后切下一个 state。

#### 第二步：AnimState 是什么？

`AnimState` 是 entity 的一个引擎组件——和 `Transform`、`Physics`、`SoundEmitter` 同级——由 C++ 引擎实现，**lua 层只是调它的方法**：

```lua
inst.entity:AddAnimState()  -- prefab fn 里加
inst.AnimState:SetBank("wilson")
inst.AnimState:SetBuild("wilson")
inst.AnimState:PlayAnimation("idle")
```

**3 个核心概念**：
- **Bank**：动画"骨架"——决定有哪些骨骼/symbol（同 prefab 通常用同一个 bank）
- **Build**：动画"贴图"——决定每个 symbol 的具体图案（不同皮肤换不同 build）
- **Animation**：动画"动作"——决定骨骼如何运动（idle / walk / chop_loop / ...）

**等价类比**：bank=骨骼模型、build=皮肤、animation=动作。

#### 第三步：动画播放的"流水线"

```
[T=0]   AnimState:PlayAnimation("chop_loop")
        ↓
[引擎]  内部加载 chop_loop 动画的关键帧数据
        每帧渲染：根据 build 计算贴图 + 根据 animation 计算位置
        ↓
[T=播完] 内部触发"动画完成"
        ↓
[引擎]  PushEvent("animover") 给 entity
        ↓
[lua]   SG 的 events["animover"] handler 触发
        ↓
[lua]   handler 里通常 GoToState("idle")
```

**注意**：**`animover` 是引擎自动 push 的事件**——不是开发者手动触发——SG 收到后切到下个 state。

#### 第四步：动画时长 vs SG 时长

**动画**有自己的时长（frames * 1/30 秒）。**SG state** 也有自己的时长（`SetTimeout`）。**两者是独立的**。

- 动画播完 → push `animover` → state 通过 events 处理
- state 时长到 → 触发 `ontimeout` → 通过 ontimeout 处理

**典型场景**：state 用 ontimeout（限时 1 秒），不依赖动画 → 动画 0.7 秒就完成 → 后 0.3 秒动画停止但 state 没切。**修复**：要么用 PushAnimation 让动画循环，要么把 state 时长改成动画时长。

---

### 11.3.2 快速入门：PlayAnimation / PushAnimation 的基础

#### 第一步：PlayAnimation —— 立刻播放

```lua
inst.AnimState:PlayAnimation("chop_loop")
inst.AnimState:PlayAnimation("chop_loop", true)  -- 第二个参数 true = loop
```

**作用**：**清空动画队列**——立刻开始播放新动画。

**第二个参数 `loop`**：true = 循环；false（默认）= 播一次。

#### 第二步：PushAnimation —— 加入队列

```lua
inst.AnimState:PlayAnimation("chop_pre")
inst.AnimState:PushAnimation("chop_loop", true)
```

**作用**：把动画**追加到队列**——播完前一个之后播这个。

**典型用法**：3 段拼接

```lua
inst.AnimState:PlayAnimation("chop_pre")            -- 起手
inst.AnimState:PushAnimation("chop_loop", false)    -- 主体
inst.AnimState:PushAnimation("chop_pst", false)     -- 收尾
```

**3 段播完后**触发 `animqueueover` 事件。

#### 第三步：PlayAnimation vs PushAnimation 的差异

| API | 行为 |
| --- | --- |
| `PlayAnimation` | **重置队列**——已加入的 PushAnimation 全清掉 |
| `PushAnimation` | **加入队列尾**——前面的会按顺序播放 |

**典型陷阱**：

```lua
inst.AnimState:PushAnimation("idle", true)
inst.AnimState:PushAnimation("chop_pre", false)
-- 期望：先播 idle 一段，再播 chop_pre
-- 实际：第一个 PushAnimation 没有"立刻播"——它在等队列前面的项
-- 但队列里没有项 → 卡住直到下一个 PlayAnimation
```

**修复**：第一项用 PlayAnimation：

```lua
inst.AnimState:PlayAnimation("idle", true)
inst.AnimState:PushAnimation("chop_pre", false)
```

#### 第四步：常用动画名约定

Klei 动画文件命名规范：

```
walk_pre / walk_loop / walk_pst
attack
attack_pre / attack_loop / attack_pst
chop_pre / chop_loop / chop_pst
mine_pre / mine_loop / mine_pst
build_pre / build_loop / build_pst
death
hit
idle
```

**3 段模式**（`_pre` / `_loop` / `_pst`）—— 起手、主体（可循环）、收尾。

---

### 11.3.3 快速入门：animover / animqueueover 事件

#### animover —— 当前动画播放完

```lua
events = {
    EventHandler("animover", function(inst)
        inst.sg:GoToState("idle")
    end),
}
```

**触发时机**：当前 PlayAnimation 设置的动画播放完一遍。

**对 PushAnimation 的行为**：
- `PlayAnimation("a")` + `PushAnimation("b")` —— a 播完触发 1 次 animover、然后开始播 b、b 播完再触发 1 次 animover

**所以**：**每段动画播完都触发 1 次 animover**——`animover` 事件可能在一个 state 期间触发多次。

#### animqueueover —— 整个动画队列播完

```lua
events = {
    EventHandler("animqueueover", function(inst)
        if inst.AnimState:AnimDone() then
            inst.sg:GoToState("idle")
        end
    end),
}
```

**触发时机**：**动画队列**全部播完后——整个流水线结束。

**对比**：

```
PlayAnimation("a"); PushAnimation("b"); PushAnimation("c")
  ↓
animover (a 播完)
  ↓ b 开始播
animover (b 播完)
  ↓ c 开始播
animover (c 播完)
  ↓
animqueueover (整个队列播完)
```

#### 用哪个？

**用 animqueueover 的场景**：3 段拼接动画——只关心"整段播完"

```lua
onenter = function(inst)
    inst.AnimState:PlayAnimation("attack_pre")
    inst.AnimState:PushAnimation("attack_loop", false)
    inst.AnimState:PushAnimation("attack_pst", false)
end,

events = {
    EventHandler("animqueueover", function(inst)
        inst.sg:GoToState("idle")  -- 整套打完才回 idle
    end),
}
```

**用 animover 的场景**：单动画或者有循环的复杂逻辑

```lua
onenter = function(inst)
    inst.AnimState:PlayAnimation("idle_loop", true)  -- 循环
end,

events = {
    EventHandler("animover", function(inst)
        if some_condition then
            inst.sg:GoToState("xxx")
        end
        -- 否则继续循环
    end),
}
```

#### AnimDone 检查

```lua
events = {
    EventHandler("animqueueover", function(inst)
        if inst.AnimState:AnimDone() then  -- 防御性检查
            inst.sg:GoToState("idle")
        end
    end),
}
```

**`AnimDone()`** —— 引擎层级判断"当前动画/队列是否真的完成"——`animqueueover` 事件触发时大概率 true，但**有些边界情况**（比如同帧多次 PlayAnimation）会有偏差——加上判断更稳健。

> 看 11.3.1 第一步真实代码 `SGwilson.lua:2588-2592` —— Klei 自己也这样写：

```2588:2592:scripts/stategraphs/SGwilson.lua
            EventHandler("animover", function(inst)
                if inst.AnimState:AnimDone() then
                    inst.sg:GoToState("idle")
                end
            end),
```

---

### 11.3.4 进阶：动画时序的常用 API

#### IsCurrentAnimation —— 当前播的是不是某动画

```lua
if inst.AnimState:IsCurrentAnimation("idle_loop") then
    -- ...
end
```

**典型用法**：避免重复 PlayAnimation 同一个动画

```lua
onenter = function(inst)
    if not inst.AnimState:IsCurrentAnimation("idle") then
        inst.AnimState:PlayAnimation("idle", true)
    end
end,
```

#### GetCurrentAnimation —— 获取当前动画名

```lua
local anim = inst.AnimState:GetCurrentAnimation()
print(anim)  -- "idle_loop"
```

**注意**：这个返回**当前正在播的**——不是队列里所有的。

#### SetTime —— 跳到动画的指定时间

```lua
inst.AnimState:SetTime(1.5)  -- 跳到第 1.5 秒
```

**用法**：玩家随机化某些动画的"起始时间"——避免所有同 prefab 同步播放（视觉单调）。

#### AnimDone —— 当前动画/队列是否结束

```lua
if inst.AnimState:AnimDone() then
    -- 动画播完了
end
```

#### SetFrame —— 跳到指定帧

```lua
inst.AnimState:SetFrame(15)  -- 跳到第 15 帧
```

#### GetCurrentAnimationLength / GetCurrentAnimationNumFrames

```lua
local seconds = inst.AnimState:GetCurrentAnimationLength()  -- 总时长（秒）
local frames = inst.AnimState:GetCurrentAnimationNumFrames()  -- 总帧数
```

**用法**：动态计算 SG state 时长

```lua
onenter = function(inst)
    inst.AnimState:PlayAnimation("varying_anim")
    inst.sg:SetTimeout(inst.AnimState:GetCurrentAnimationLength())
end,
```

---

### 11.3.5 进阶：Symbol 替换与外观叠加

#### 第一步：Symbol 是什么？

动画 build 由多个 **symbol** 组成：

- `head` —— 头部
- `body` —— 身体
- `arm_upper` / `arm_lower_l` / `arm_lower_r` —— 手臂
- `swap_object` —— 装备物品的"挂载点"
- `hat` —— 帽子
- `face` / `face_shock` / `face_neutral` —— 表情

**每个 symbol 在 build 文件里有对应的贴图**——动画播放时按 symbol 换贴图组合渲染。

#### 第二步：OverrideSymbol —— 用别的 build 替换某 symbol

```lua
inst.AnimState:OverrideSymbol("swap_object", "swap_axe", "axe")
-- 把当前 build 的 swap_object symbol 替换为 swap_axe build 的 axe symbol
```

**3 个参数**：
- 当前 build 中要替换的 symbol 名
- 要从哪个 build 取
- 那个 build 中的 symbol 名

**典型用法**：装备武器时——替换玩家手里的"挂载点"贴图：

```lua
-- 装上斧头
inst.AnimState:OverrideSymbol("swap_object", "swap_axe", "axe")

-- 卸下斧头
inst.AnimState:ClearOverrideSymbol("swap_object")
```

#### 第三步：HideSymbol / ShowSymbol

```lua
inst.AnimState:HideSymbol("hat")
inst.AnimState:ShowSymbol("hat")
```

**用法**：玩家变身时隐藏头发；穿头盔时隐藏头发。

#### 第四步：Override / Hide 的清理

**重要**：override / hide 操作**会持续到清除**——不会因为 PlayAnimation 重置。**必须主动 ClearOverride / ShowSymbol** 还原。

**常见 mod 陷阱**：临时换 symbol 后没清——下一次播别的动画时 symbol 还是旧值。

#### 第五步：SetMultColour / SetAddColour —— 颜色叠加

```lua
inst.AnimState:SetMultColour(1, 0, 0, 1)  -- 红色叠加（乘）
inst.AnimState:SetAddColour(0.5, 0, 0, 1)  -- 红光（加）
```

**典型用法**：着火时角色变红、被冻时变蓝。

---

### 11.3.6 进阶：动画速率与暂停

#### SetDeltaTimeMultiplier —— 改变动画速度

```lua
inst.AnimState:SetDeltaTimeMultiplier(2)   -- 2 倍速
inst.AnimState:SetDeltaTimeMultiplier(0.5) -- 半速
inst.AnimState:SetDeltaTimeMultiplier(1)   -- 还原
```

**用法**：施法卡顿（慢速）、急速 buff（加速）。

#### Pause / Resume —— 暂停/继续

```lua
inst.AnimState:Pause()
inst.AnimState:Resume()
```

**用法**：玩家被冻结时暂停动画。

#### 多个 multiplier 互相影响

如果同时有"buff +20%"和"debuff -30%"——**SetDeltaTimeMultiplier 是覆盖式**——后调的覆盖前调的。

**协调方式**：用 mod 管理器或共享 modifier 列表：

```lua
local function UpdateAnimSpeed(inst)
    local final = 1
    for _, m in pairs(inst._anim_modifiers) do
        final = final * m
    end
    inst.AnimState:SetDeltaTimeMultiplier(final)
end
```

---

### 11.3.7 老手进阶：动画驱动业务的 3 种模式

#### 模式 1：fixed timeline ——精确帧

```lua
State{
    name = "chop",
    
    onenter = function(inst)
        inst.AnimState:PlayAnimation("chop")
    end,
    
    timeline = {
        FrameEvent(9, function(inst)  -- 第 9 帧
            inst.SoundEmitter:PlaySound("axe_sound")
        end),
        FrameEvent(14, function(inst)
            inst:PerformBufferedAction()
        end),
    },
    
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
}
```

**适用**：动作的"接触帧"已知——**只有这个帧才该触发音效/伤害**。

**优点**：精确——和动画严格对齐。  
**缺点**：动画文件改了帧数 → timeline 也得改。

#### 模式 2：phase animations ——多段拼接

```lua
State{
    name = "attack",
    
    onenter = function(inst)
        inst.AnimState:PlayAnimation("attack_pre")
        inst.AnimState:PushAnimation("attack_loop", false)
        inst.AnimState:PushAnimation("attack_pst", false)
    end,
    
    events = {
        EventHandler("animover", function(inst)
            if inst.AnimState:IsCurrentAnimation("attack_loop") then
                inst:PerformBufferedAction()  -- 主体动画结束 = 接触帧
            end
        end),
        EventHandler("animqueueover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
}
```

**适用**：3 段拼接——业务在主体动画结束时触发。

**优点**：动画文件可独立调整——业务挂钩是"动画段"而非"具体帧"。  
**缺点**：拼接逻辑复杂、需要多个动画文件。

#### 模式 3：anim-driven loop ——动画循环 + state 内部计数

```lua
State{
    name = "long_chop",
    
    onenter = function(inst)
        inst.sg.statemem.hits = 0
        inst.AnimState:PlayAnimation("chop_loop", true)  -- 循环
    end,
    
    timeline = {
        FrameEvent(14, function(inst)
            inst:PerformBufferedAction()
            inst.sg.statemem.hits = inst.sg.statemem.hits + 1
            if inst.sg.statemem.hits >= 5 then
                inst.sg:GoToState("idle")  -- 5 次后退出
            end
        end),
    },
}
```

**适用**：连续多次动作——每次的接触帧都触发业务——直到达成条件。

**优点**：业务逻辑灵活、动画无限循环。  
**缺点**：循环内部状态管理复杂——容易遗漏退出条件。

---

### 11.3.8 老手进阶：六个常见陷阱

#### 陷阱 1：onenter 里 PlayAnimation 没考虑当前已经在播

```lua
-- ❌
onenter = function(inst)
    inst.AnimState:PlayAnimation("idle", true)  -- 每次 GoToState idle 都重新播
end,
```

**问题**：玩家进出 idle 几十次，每次 PlayAnimation 都从第 0 帧开始——视觉上"卡顿"。  
**修复**：

```lua
onenter = function(inst)
    if not inst.AnimState:IsCurrentAnimation("idle") then
        inst.AnimState:PlayAnimation("idle", true)
    end
end,
```

#### 陷阱 2：用 SetTime 跳帧但忘了 timeline 已触发过

```lua
onenter = function(inst)
    inst.AnimState:PlayAnimation("chop")
    inst.AnimState:SetTime(0.4)  -- 跳到第 0.4 秒
end,

timeline = {
    FrameEvent(5, fn),  -- 第 5 帧（约 0.167 秒），已被跳过——但 SG timeline 系统**不知道**——可能不会触发
}
```

**修复**：跳帧时要重置 timeline 状态——或者用 onenter 直接调 fn：

```lua
onenter = function(inst)
    inst.AnimState:PlayAnimation("chop")
    inst.AnimState:SetTime(0.4)
    -- 第 5 帧的业务直接调
    inst.SoundEmitter:PlaySound("preemptive_sound")
end,
```

#### 陷阱 3：override symbol 没在 onexit 清

**症状**：玩家从 attack state 出来后——手里仍然显示"挥剑"贴图（应该回到 idle 的拿剑姿态）。  
**修复**：onexit 清除：

```lua
onexit = function(inst)
    inst.AnimState:ClearOverrideSymbol("swap_object")
end,
```

#### 陷阱 4：animover 在 PushAnimation 队列中触发多次

**症状**：

```lua
events = {
    EventHandler("animover", function(inst)
        inst.sg:GoToState("idle")  -- 第一段播完就切了——后面两段没播
    end),
}
```

**修复**：用 `animqueueover`：

```lua
events = {
    EventHandler("animqueueover", function(inst)
        if inst.AnimState:AnimDone() then
            inst.sg:GoToState("idle")
        end
    end),
}
```

#### 陷阱 5：动画文件不存在导致 PlayAnimation 失败

**症状**：mod 自定义的 prefab——`PlayAnimation("myanim")` —— 但 build 文件里没有 myanim 动画——**静默失败**——动画卡住。  
**修复**：开发时 grep build 文件确认动画名——或者用 `IsValid()` 等防御调用。

#### 陷阱 6：SetDeltaTimeMultiplier 没还原

**症状**：mod 给某 state 改了动画速度——下一个 state 也是这个速度——**全局影响**。  
**修复**：

```lua
onenter = function(inst)
    inst.AnimState:SetDeltaTimeMultiplier(2)
end,
onexit = function(inst)
    inst.AnimState:SetDeltaTimeMultiplier(1)  -- 还原
end,
```

#### 设计经验三条

**经验 ①：动画名 lowercase + snake_case**

`chop_pre` / `attack_loop` —— Klei 约定。

**经验 ②：3 段拼接是黄金模式**

复杂动作几乎都用 `_pre / _loop / _pst` 三段——好维护、好动画师协作。

**经验 ③：onenter 第一句永远是 stop locomotor**

```lua
onenter = function(inst)
    inst.components.locomotor:Stop()  -- 永远第一行
    inst.AnimState:PlayAnimation("...")
end,
```

避免动画播放时玩家还在滑动。

---

### 11.3.9 小结

**动画系统一句话总结**：**AnimState 是动画引擎组件，PlayAnimation/PushAnimation 控制队列，timeline + animover/animqueueover 把动画帧和业务逻辑对齐**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 播单段 | `inst.AnimState:PlayAnimation("anim")` |
| 循环 | `inst.AnimState:PlayAnimation("anim", true)` |
| 队列 | `inst.AnimState:PushAnimation("next", false)` |
| 监听结束 | `EventHandler("animqueueover", fn)` |
| 替换 symbol | `inst.AnimState:OverrideSymbol("swap_object", "swap_axe", "axe")` |
| 清除替换 | `inst.AnimState:ClearOverrideSymbol("swap_object")` |
| 加速 | `inst.AnimState:SetDeltaTimeMultiplier(2)` |
| 暂停 | `inst.AnimState:Pause()` |
| 跳帧 | `inst.AnimState:SetTime(0.5)` 或 `SetFrame(15)` |
| 查询当前 | `inst.AnimState:IsCurrentAnimation("anim")` |
| 查询长度 | `inst.AnimState:GetCurrentAnimationLength()` |

**6 个陷阱排雷顺序**

1. 重复 PlayAnimation → IsCurrentAnimation 判
2. SetTime 跳过 timeline → onenter 直接调 fn
3. override 没清 → onexit 清
4. animover 多次触发 → 用 animqueueover
5. 动画名拼写错 → grep build 确认
6. SetDeltaTimeMultiplier 没还原 → onexit 改回 1

**3 条设计经验**

- ① **动画名 lowercase + snake_case**
- ② **3 段拼接黄金模式**
- ③ **onenter 第一行 stop locomotor**

> **下一节预告**：11.4 节我们将打开 **commonstates.lua —— 共享状态库** —— Klei 把"吃 / 睡 / 战斗 / 砍树 / 挖矿"等几十个通用状态做成可复用的工具函数。读完 11.4，你将知道 mod 怎么**直接复用 Klei 的成熟 state**——避免重复造轮子。

## 11.4 commonstates.lua——共享状态库详解（吃、睡、战斗、砍树等）

### 本节导读

11.1 ~ 11.3 我们看到怎么写一个 State——但**实际开发**中你会发现**90% 的怪物 / NPC 都需要相同的基础状态**：

- `idle`（站着/走来走去）
- `walk_pre / walk / walk_pst`（走路 3 段）
- `run_pre / run / run_pst`（跑步 3 段）
- `attack`（攻击）
- `hit`（被打）
- `death`（死亡）
- `sleep / sleeping / wakeup`（睡眠周期）
- `frozen / hit_frozen / thaw`（冻结）

**饥荒里几百种生物**——如果每只都从零写这些 state——会有数万行重复代码。**Klei 的解决方案是 `commonstates.lua`** —— 把这些通用 state **参数化为函数**——任何 SG 都可以**一行调用复用**：

```lua
CommonStates.AddIdle(states)
CommonStates.AddSimpleWalkStates(states, "walk")
CommonStates.AddCombatStates(states)
CommonStates.AddSleepStates(states)
```

**就这 4 行**——你的怪物 SG 已经具备 12+ 个 state——**完全免费**。

这一节我们打开 `commonstates.lua`——3000 多行的通用状态库——讲清楚 mod 怎么"复用 + 定制" Klei 的成熟 state。

> **新手**从 11.4.1-11.4.3 起步——理解 commonstates 存在意义、CommonHandlers vs CommonStates 两大类、常用 Handlers 速查；**进阶读者**继续看 11.4.4-11.4.6，深入 CommonStates.AddXxx 状态生成函数（idle/locomote/combat/sleep/death）、参数化设计、mod 自定义生物完整模板；**老手**跳到 11.4.7-11.4.8，看定制化用法 + 6 个最容易踩的陷阱。

---

### 11.4.1 快速入门：从一个简单 SG 看 commonstates 的存在意义

#### 第一步：简单生物 SG 的"骨架"

打开任意一个简单生物的 SG（比如 `scripts/stategraphs/SGmonster.lua` 类似的简化结构）：

```lua
require("stategraphs/commonstates")

local actionhandlers = {
    -- ...
}

local events = {
    CommonHandlers.OnLocomote(false, true),  -- 一行替代 walk 事件处理
    CommonHandlers.OnSleep(),                -- 一行替代 gotosleep 事件处理
    CommonHandlers.OnFreeze(),               -- 一行替代 freeze 事件处理
    CommonHandlers.OnAttacked(),             -- 一行替代 attacked 事件处理
    CommonHandlers.OnDeath(),                -- 一行替代 death 事件处理
}

local states = {
    -- 自定义 state 写在这里
}

CommonStates.AddIdle(states)
CommonStates.AddSimpleWalkStates(states, "walk")
CommonStates.AddSleepStates(states)
CommonStates.AddFrozenStates(states)
CommonStates.AddCombatStates(states)
CommonStates.AddDeathState(states)

return StateGraph("monster", states, events, "idle", actionhandlers)
```

**就这一段**——一只完整生物的 SG 就**已经能用了**：能 idle、能走路、能睡觉、能被冻、能战斗、能死。**总代码量 < 30 行**。

#### 第二步：commonstates 的设计哲学

`commonstates.lua` 不是"具体 state 列表"——它是**生成 state 的工厂函数集合**：

```lua
CommonStates.AddIdle = function(states, funny_idle_state, anim_override, timeline)
    table.insert(states, State{
        name = "idle",
        ...
    })
end
```

**`AddIdle(states)` 做的事**：把一个**已经写好的 idle State** 塞进 states table。

**好处**：
- mod 不需要重新发明轮子
- 所有 mod 共享 idle 行为——一致性强
- 修改 idle 的"通用部分"只改 commonstates 一处

#### 第三步：参数化定制

但**不同生物的 idle 又不完全一样**——比如：

- 普通生物：站着不动
- 兔子：偶尔抖一下
- 火鸡：摇头

`commonstates` 的函数**参数化**——接受 mod 的"特化"参数：

```lua
CommonStates.AddIdle(states, "funny_idle_state_name")  -- 自定义"偶尔切到 funny 状态"
CommonStates.AddIdle(states, nil, "custom_anim")        -- 自定义动画名
```

**这样**：基础逻辑复用、特化部分可改。

#### 第四步：CommonHandlers 同款思路

```24:46:scripts/stategraphs/commonstates.lua
local function onstep(inst)
    if inst.SoundEmitter ~= nil then
        inst.SoundEmitter:PlaySound("dontstarve/movement/run_dirt")
        --inst.SoundEmitter:PlaySound("dontstarve/movement/walk_dirt")
    end
end

CommonHandlers.OnStep = function()
    return EventHandler("step", onstep)
end

--------------------------------------------------------------------------
local function onsleep(inst)
	if not (inst.components.health and inst.components.health:IsDead() or inst.sg:HasStateTag("electrocute")) then
        local fallingreason = inst.components.drownable and inst.components.drownable:GetFallingReason() or nil
        if fallingreason ~= nil and inst.sg:HasStateTag("jumping") then
            if fallingreason == FALLINGREASON.OCEAN then
                inst.sg:GoToState("sink")
            elseif fallingreason == FALLINGREASON.VOID then
                inst.sg:GoToState("abyss_fall")
            end
		else
		    inst.sg:GoToState(inst.sg:HasStateTag("sleeping") and "sleeping" or "sleep")
		end
    end
end

CommonHandlers.OnSleep = function()
    return EventHandler("gotosleep", onsleep)
end
```

**机制**：每个 `CommonHandlers.OnXxx` **是个工厂函数**——返回一个 `EventHandler` —— 直接塞进 SG 的 events 表。

> **核心结论**：**commonstates = state 生成 + handler 生成的函数库**。复用度极高、定制度也保留——是 Klei 设计精妙的工程实践。

---

### 11.4.2 快速入门：CommonHandlers vs CommonStates 两大类

#### 第一步：API 全景

| 类别 | 用途 | 函数命名 |
| --- | --- | --- |
| `CommonHandlers.OnXxx()` | 返回一个 EventHandler | OnStep / OnSleep / OnFreeze / OnAttacked / OnAttack / OnDeath / OnLocomote / ... |
| `CommonStates.AddXxx(states, ...)` | 把 state(s) 塞进 states 表 | AddIdle / AddSimpleState / AddRunStates / AddWalkStates / AddSleepStates / AddCombatStates / AddDeathState / AddFrozenStates / AddElectrocuteStates / AddHopStates / AddRowStates / ... |

#### 第二步：使用模式

**CommonHandlers**：

```lua
local events = {
    CommonHandlers.OnSleep(),      -- 直接塞进 events 表
    CommonHandlers.OnFreeze(),
    CommonHandlers.OnDeath(),
    CommonHandlers.OnLocomote(false, true),  -- 带参数
}
```

**CommonStates**：

```lua
local states = {}

CommonStates.AddIdle(states)              -- 改原 table（无返回）
CommonStates.AddSimpleWalkStates(states, "walk")
CommonStates.AddCombatStates(states)
CommonStates.AddDeathState(states)
```

**关键差异**：
- CommonHandlers 是 **返回值**（EventHandler 对象）
- CommonStates 是 **副作用**（改 states 表）

#### 第三步：参数对照速查

| CommonStates 函数 | 关键参数 |
| --- | --- |
| `AddIdle(states, funny_idle_state, anim_override, timeline)` | funny 状态名 / 动画名 / timeline 自定义 |
| `AddSimpleState(states, name, anim, tags, finishstate, timeline, fns)` | 通用单 state 生成 |
| `AddSimpleActionState(states, name, anim, time, tags, finishstate, timeline, fns)` | 带计时的动作 state |
| `AddRunStates(states, timelines, anims, softstop, delaystart, fns)` | 完整跑步状态组 |
| `AddSimpleRunStates(states, anim, timelines)` | 简化版跑步 |
| `AddWalkStates(states, timelines, anims, softstop, delaystart, fns)` | 完整走路状态组 |
| `AddSimpleWalkStates(states, anim, timelines)` | 简化版走路 |
| `AddSleepStates(states, timelines, fns)` | 睡眠状态组（sleep / sleeping / wake） |
| `AddCombatStates(states, timelines, anims, fns, data)` | 战斗状态组（attack / hit / death）|
| `AddHitState(states, timeline, anim)` | 仅 hit 状态 |
| `AddDeathState(states, timeline, anim, fns, data)` | 仅 death 状态 |
| `AddFrozenStates(states, onoverridesymbols, onclearsymbols)` | 冻结状态组 |
| `AddElectrocuteStates(states, timelines, anims, fns)` | 电击状态组 |
| `AddHopStates(states, wait_for_pre, anims, timelines, ...)` | 跳跃（船边）状态组 |
| `AddRowStates(states, is_client)` | 划船状态组 |

---

### 11.4.3 快速入门：常用 CommonHandlers 速查

#### CommonHandlers.OnLocomote(can_run, can_walk)

```lua
events = { CommonHandlers.OnLocomote(false, true) }
```

**作用**：自动处理 `locomote` 事件——根据 can_run / can_walk 切到对应 state（runspeed / walk）。

**参数**：
- `can_run` —— 是否能跑
- `can_walk` —— 是否能走

**典型组合**：
- `(false, true)` —— 只能走（普通 NPC）
- `(true, true)` —— 能跑也能走（玩家、紧急生物）
- `(true, false)` —— 只跑不走（追击型怪物）

#### CommonHandlers.OnSleep()

```lua
events = { CommonHandlers.OnSleep() }
```

**作用**：处理 `gotosleep` 事件——切到 `sleep` 或 `sleeping` state。

**前提**：SG 里必须有 sleep 相关 state——通常 `CommonStates.AddSleepStates(states)` 自动加。

#### CommonHandlers.OnFreeze() / OnFreezeEx()

```lua
events = { CommonHandlers.OnFreeze() }
```

**作用**：处理 `freeze` 事件——切到 `frozen` state。

**OnFreeze 与 OnFreezeEx 的区别**：
- `OnFreeze` —— 检查 health 组件（无 health 不冻）
- `OnFreezeEx` —— 不检查 health（适用于无 health 的 entity）

#### CommonHandlers.OnAttacked(hitreact_cooldown, max_hitreacts, skip_cooldown_fn)

```lua
events = { CommonHandlers.OnAttacked() }                  -- 默认
events = { CommonHandlers.OnAttacked(0.5, 3) }             -- 自定义
```

**作用**：处理 `attacked` 事件——切到 `hit` state（通常）+ 处理 hitreact 冷却。

**3 参数**：
- `hitreact_cooldown` —— hit 反应的冷却时间（避免连击锁死）
- `max_hitreacts` —— 进入冷却前能 hit 几次
- `skip_cooldown_fn` —— 自定义"是否能 hit"判定

#### CommonHandlers.OnAttack()

```lua
events = { CommonHandlers.OnAttack() }
```

**作用**：处理 `doattack` 事件——切到 `attack` state。

#### CommonHandlers.OnDeath()

```lua
events = { CommonHandlers.OnDeath() }
```

**作用**：处理 `death` 事件——切到 `death` state。

#### CommonHandlers.OnHop()

```lua
events = { CommonHandlers.OnHop() }
```

**作用**：处理 `onhop` 事件——切到 `hop_pre` state（船边跳跃）。

#### CommonHandlers.OnElectrocute()

```lua
events = { CommonHandlers.OnElectrocute() }
```

**作用**：处理 `electrocute` 事件——切到 `electrocute` state。

#### CommonHandlers.OnFossilize()

```lua
events = { CommonHandlers.OnFossilize() }
```

**作用**：处理 `fossilize` 事件——切到 `fossilized` state。

---

### 11.4.4 进阶：CommonStates.AddXxx 详解

#### CommonStates.AddIdle(states, funny_idle_state, anim_override, timeline)

```408:455:scripts/stategraphs/commonstates.lua
CommonStates.AddIdle = function(states, funny_idle_state, anim_override, timeline)
```

**生成的 state**：`idle`

**核心行为**：
- onenter：播 idle 动画（默认 "idle_loop"）
- 偶尔随机切到 funny_idle_state（如果指定）

**参数详解**：
- `funny_idle_state` —— mod 自定义"偶尔切到这个 state"——通常是 "idle_funny" 之类
- `anim_override` —— 自定义动画名（默认 "idle_loop"）
- `timeline` —— 自定义 TimeEvent 列表

**实例**：

```lua
-- 普通 idle
CommonStates.AddIdle(states)

-- 偶尔切到 funny
CommonStates.AddIdle(states, "funny_idle")

-- 自定义动画
CommonStates.AddIdle(states, nil, "stand_loop")

-- 自定义 timeline（idle 期间也有事件）
CommonStates.AddIdle(states, nil, nil, {
    TimeEvent(2, function(inst) print("idle 2 seconds") end),
})
```

#### CommonStates.AddSimpleWalkStates / AddSimpleRunStates

```lua
CommonStates.AddSimpleWalkStates(states, "walk")
CommonStates.AddSimpleRunStates(states, "run")
```

**生成的 state**（每组 3 个）：
- `walk_start` / `walk_loop` / `walk_stop` (或 startwalk / walk / stopwalk)
- `run_start` / `run_loop` / `run_stop` (或 startrun / run / stoprun)

**作用**：完整 3 段移动动画+ 业务（移动事件、玩家朝向）。

#### CommonStates.AddCombatStates

```1385:1470:scripts/stategraphs/commonstates.lua
CommonStates.AddCombatStates = function(states, timelines, anims, fns, data)
```

**生成的 state**（默认 3 个）：
- `attack` —— 攻击
- `hit` —— 被打
- `death` —— 死亡（如果 `data.no_death = false`）

**参数详解**：
- `timelines` —— `{attack = {...}, hit = {...}, death = {...}}` 各 state 的 timeline 自定义
- `anims` —— `{attack = "atk", hit = "hit", death = "die"}` 自定义动画名
- `fns` —— `{onattack = function(inst) end, ...}` 自定义函数
- `data` —— `{no_death = true}` 等额外配置

**实例**：

```lua
CommonStates.AddCombatStates(states,
    -- timelines
    {
        attack = {
            TimeEvent(15 * FRAMES, function(inst) inst.components.combat:DoAttack() end),
        },
    },
    -- anims
    {
        attack = "atk_special",
    },
    -- fns
    {
        onhit = function(inst)
            inst.SoundEmitter:PlaySound("dontstarve/creature/hurt")
        end,
    }
)
```

#### CommonStates.AddSleepStates

```1199:1338:scripts/stategraphs/commonstates.lua
CommonStates.AddSleepStates = function(states, timelines, fns)
```

**生成的 state**：
- `sleep` —— 入睡前
- `sleeping` —— 睡着循环
- `wake` —— 醒来

**典型用法**：

```lua
CommonStates.AddSleepStates(states, {
    starttimeline = {
        SoundFrameEvent(8, "dontstarve/creature/snore"),
    },
})
```

#### CommonStates.AddFrozenStates

```1339:1384:scripts/stategraphs/commonstates.lua
CommonStates.AddFrozenStates = function(states, onoverridesymbols, onclearsymbols)
```

**生成的 state**：
- `frozen` —— 冻结
- `hit_frozen` —— 冻结时受击
- `thaw` —— 解冻过程

#### CommonStates.AddDeathState

```1595:1651:scripts/stategraphs/commonstates.lua
CommonStates.AddDeathState = function(states, timeline, anim, fns, data)
```

**生成的 state**：`death`

**死亡动画的关键 timeline 通常**：
- 第 X 帧：播死亡音效
- 死亡动画结束：触发 lootdrop / corpse spawn

---

### 11.4.5 进阶：commonstates 的"参数化"设计

#### 第一步：函数签名的"展开"

观察 CommonStates.AddCombatStates 的签名：

```lua
CommonStates.AddCombatStates(states, timelines, anims, fns, data)
```

**5 个参数**：
- `states` —— 要修改的 state table（必需）
- `timelines` —— 子 state 的 timeline 覆盖（可选）
- `anims` —— 子 state 的动画名覆盖（可选）
- `fns` —— 自定义钩子函数（可选）
- `data` —— 额外配置（可选）

**好处**：
- mod 只传入"需要定制"的部分——其他默认
- 函数内部按 `or 默认值` 处理 nil 参数——优雅

#### 第二步：典型 fns 钩子

```lua
CommonStates.AddCombatStates(states, nil, nil, {
    onhit = function(inst)
        -- 被打时的额外业务
    end,
    onattack = function(inst)
        -- 攻击时的额外业务
    end,
    ondeath = function(inst)
        -- 死亡时的额外业务
    end,
})
```

**`onhit` 等钩子**——commonstates 内部调用它，但**不影响默认行为**——纯加法。

#### 第三步：anims 覆盖

```lua
CommonStates.AddCombatStates(states, nil, {
    attack = "atk_2",     -- 默认 "atk"
    hit = "hit_special",   -- 默认 "hit"
    death = "die_dramatic", -- 默认 "death"
})
```

**用途**：mod 自定义动画文件名时。

#### 第四步：timelines 覆盖

```lua
CommonStates.AddCombatStates(states, {
    attack = {
        TimeEvent(15 * FRAMES, function(inst)
            inst.components.combat:DoAttack()
        end),
    },
}, nil, nil, nil)
```

**注意**：传入的 timeline **追加到**默认 timeline 后面——不是替换。

---

### 11.4.6 进阶：mod 自定义生物的 commonstates 标准模板

#### 第一步：完整 SG 模板

```lua
-- mymod/scripts/stategraphs/SGmymonster.lua

require("stategraphs/commonstates")

local actionhandlers = {
    -- 自定义 action 在这
}

local events = {
    -- CommonHandlers
    CommonHandlers.OnLocomote(false, true),
    CommonHandlers.OnSleep(),
    CommonHandlers.OnFreeze(),
    CommonHandlers.OnAttacked(),
    CommonHandlers.OnAttack(),
    CommonHandlers.OnDeath(),
    
    -- 自定义事件
    EventHandler("doaction", function(inst, action)
        -- ...
    end),
}

local states = {
    -- 自定义 state 在这（如果有特殊状态）
    State{
        name = "spawn_in",
        tags = { "busy", "noattack" },
        onenter = function(inst)
            inst.AnimState:PlayAnimation("spawn")
        end,
        events = {
            EventHandler("animover", function(inst)
                inst.sg:GoToState("idle")
            end),
        },
    },
}

-- 复用 commonstates
CommonStates.AddIdle(states)
CommonStates.AddSimpleWalkStates(states, "walk")
CommonStates.AddSimpleRunStates(states, "run")
CommonStates.AddSleepStates(states)
CommonStates.AddFrozenStates(states)
CommonStates.AddCombatStates(states, {
    attack = {
        TimeEvent(15 * FRAMES, function(inst)
            inst.components.combat:DoAttack()
        end),
    },
})
CommonStates.AddDeathState(states)

return StateGraph("mymonster", states, events, "spawn_in", actionhandlers)
```

**总代码量**：约 50 行——具备完整生物的所有 state（idle / walk / run / sleep / frozen / attack / hit / death / spawn_in）—— **9+ 个 state**。

#### 第二步：在 prefab 里挂 SG

```lua
-- mymod/scripts/prefabs/mymonster.lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()
    
    inst.AnimState:SetBank("mymonster")
    inst.AnimState:SetBuild("mymonster")
    inst.AnimState:PlayAnimation("idle_loop", true)
    
    inst:AddTag("monster")
    inst:AddTag("hostile")
    
    inst.entity:SetPristine()
    
    if not TheWorld.ismastersim then
        return inst
    end
    
    inst:AddComponent("locomotor")
    inst.components.locomotor.walkspeed = 4
    inst.components.locomotor.runspeed = 7
    
    inst:AddComponent("health")
    inst.components.health:SetMaxHealth(150)
    
    inst:AddComponent("combat")
    inst.components.combat:SetAttackPeriod(2)
    inst.components.combat:SetRange(2)
    inst.components.combat:SetDefaultDamage(20)
    
    inst:AddComponent("freezable")
    inst:AddComponent("sleeper")
    inst:AddComponent("inspectable")
    
    inst:SetStateGraph("SGmymonster")  -- ★ 挂 SG
    
    return inst
end
```

#### 第三步：测试

```lua
-- 控制台
local m = c_spawn("mymonster")

-- 测各 state
m.sg:GoToState("walk")
m.sg:GoToState("attack")
m.components.health:DoDelta(-50)  -- 触发 hit
m.components.health:Kill()        -- 触发 death
```

---

### 11.4.7 老手进阶：定制化 commonstates 用法

#### 模式 1：替换默认动画

mod 自定义生物的动画名和 commonstates 默认不一致——用 anims 参数：

```lua
CommonStates.AddCombatStates(states, nil, {
    attack = "fancy_atk",
    hit = "fancy_hit",
    death = "fancy_die",
})
```

#### 模式 2：在默认 timeline 上加事件

```lua
CommonStates.AddCombatStates(states, {
    attack = {
        TimeEvent(0, function(inst)
            -- 攻击开始时的特效
            SpawnPrefab("attack_fx").Transform:SetPosition(inst.Transform:GetWorldPosition())
        end),
        TimeEvent(15 * FRAMES, function(inst)
            inst.components.combat:DoAttack()
        end),
        TimeEvent(20 * FRAMES, function(inst)
            -- 攻击后的特效
            inst.SoundEmitter:PlaySound("dontstarve/creature/scream")
        end),
    },
})
```

#### 模式 3：用 fns 加额外业务

```lua
CommonStates.AddCombatStates(states, nil, nil, {
    ondeath = function(inst)
        -- 死亡时额外业务
        inst.components.lootdropper:DropLoot()
        TheWorld:PushEvent("mymod_monster_killed", { inst = inst })
    end,
})
```

#### 模式 4：完全替换某 state

如果 commonstates 默认行为完全不符——可以**先调 commonstates 加默认 state，再用 mod 自己的 state 覆盖**：

```lua
CommonStates.AddCombatStates(states)

-- 覆盖 hit state
states[#states + 1] = State{
    name = "hit",
    tags = { "busy", "noattack", "custom" },
    onenter = function(inst)
        -- 完全自定义逻辑
    end,
    -- ...
}

-- 这样 SG 的 states["hit"] 就是 mod 的版本（最后塞的覆盖前面的）
```

不过通常**用 fns 钩子已经够灵活**——直接覆盖 state 应该是最后选择。

---

### 11.4.8 老手进阶：六个常见陷阱

#### 陷阱 1：忘记 require commonstates

```lua
-- ❌ SG 文件顶部没 require
local actionhandlers = {}

CommonHandlers.OnSleep()  -- nil 索引——CommonHandlers 是 nil
```

**修复**：永远第一行：

```lua
require("stategraphs/commonstates")
```

#### 陷阱 2：CommonHandlers 调用没加 ()

```lua
events = {
    CommonHandlers.OnSleep,  -- ❌ 这是 function 引用，不是 EventHandler
}
```

**修复**：

```lua
events = {
    CommonHandlers.OnSleep(),  -- ✓ 调用 () 才返回 EventHandler 对象
}
```

#### 陷阱 3：CommonStates 在自定义 state 之前调

```lua
-- ❌
CommonStates.AddIdle(states)
states[#states + 1] = State{ name = "idle", ... }  -- 覆盖了 commonstates 的
```

**症状**：自定义 idle 没生效——commonstates 的覆盖了。  
**原因**：`states` 是按 `state.name` 索引的——重名的覆盖（后塞的不一定胜出，看 SG 内部 `for k,v in pairs(states)` 的处理）。  
**修复**：要么不调 AddIdle、要么自定义 state 用别的 name。

#### 陷阱 4：参数顺序错

```lua
-- ❌
CommonStates.AddCombatStates(states, anims, timelines)  -- 参数顺序错了
```

**修复**：参数 4 个——记不住就写注释或用 nil 占位：

```lua
CommonStates.AddCombatStates(states, --[[timelines]] nil, --[[anims]] {attack="atk2"}, --[[fns]] nil)
```

#### 陷阱 5：fns 钩子签名错

```lua
-- ❌
fns = {
    ondeath = function() print("died") end,  -- 没接 inst
}
```

**修复**：永远 `function(inst, ...)`：

```lua
fns = {
    ondeath = function(inst) print(inst, "died") end,
}
```

#### 陷阱 6：忘记给生物加必备组件

**症状**：commonstates 的 sleep state 报错——`inst.components.sleeper` 是 nil。  
**原因**：SG 用了 sleep states 但 prefab 没加 sleeper 组件。  
**修复**：

```lua
-- prefab fn 服务端段
inst:AddComponent("sleeper")  -- ★
inst:AddComponent("freezable")
inst:AddComponent("combat")
inst:AddComponent("locomotor")
inst:AddComponent("health")
```

#### 设计经验三条

**经验 ①：先用 commonstates、再写自定义**

90% mod 生物完全不需要写自定义 state——`commonstates` 的 idle / walk / sleep / combat / death 已经够用。**先调 CommonStates.AddXxx 全套**——再发现真不够用时再加自定义。

**经验 ②：fns 钩子优先于覆盖 state**

要"加额外业务"——用 fns 钩子；要"完全替换行为"——再考虑覆盖 state。fns 的好处是**保留 commonstates 的兼容更新**——Klei 改 commonstates 时你不用改。

**经验 ③：测试每种 state 都跑通**

用控制台触发：

```lua
c_select().sg:GoToState("idle")
c_select().sg:GoToState("walk_loop")
c_select().sg:GoToState("attack")
c_select().sg:GoToState("hit")
c_select().sg:GoToState("death")
c_select().sg:GoToState("frozen")
c_select().sg:GoToState("sleep")
```

确保每个 state 都正常播完——没有报错——然后再放进游戏测试。

---

### 11.4.9 小结

**commonstates 一句话总结**：**Klei 把通用 state 参数化为函数库——CommonHandlers 返回 EventHandler、CommonStates 修改 states 表——mod 一行调用复用 + 参数定制**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 加 idle | `CommonStates.AddIdle(states)` |
| 加走路 | `CommonStates.AddSimpleWalkStates(states, "walk")` |
| 加跑步 | `CommonStates.AddSimpleRunStates(states, "run")` |
| 加睡眠 | `CommonStates.AddSleepStates(states)` |
| 加冻结 | `CommonStates.AddFrozenStates(states)` |
| 加战斗（attack/hit/death） | `CommonStates.AddCombatStates(states, timelines, anims, fns)` |
| 仅加死亡 | `CommonStates.AddDeathState(states, timeline, anim, fns)` |
| 监听 locomote | `CommonHandlers.OnLocomote(can_run, can_walk)` |
| 监听 attacked | `CommonHandlers.OnAttacked()` |
| 监听 freeze | `CommonHandlers.OnFreeze()` |
| 监听 sleep | `CommonHandlers.OnSleep()` |
| 监听 death | `CommonHandlers.OnDeath()` |

**6 个陷阱排雷顺序**

1. 没 require commonstates → 第一行 require
2. Handler 没加 () → 永远调用
3. CommonStates 之后写覆盖 → 顺序或换 name
4. 参数顺序 → 用 nil 占位 + 注释
5. fns 钩子签名 → `function(inst, ...)`
6. 忘加组件 → sleeper / freezable / combat / locomotor

**3 条设计经验**

- ① **先用 commonstates、再写自定义**
- ② **fns 钩子优先于覆盖 state**
- ③ **每种 state 都跑通**

> **下一节预告**：11.5 节我们将打开 **SGwilson / SGwilson_client** —— 玩家专用 SG —— 包含约 600 个 state、19000 行代码 —— 是饥荒里最复杂的 SG。读完 11.5，你将对玩家所有动作的实现有底层认知 —— mod 修改玩家行为时**精准下刀**。

## 11.5 通用 StateGraph（SGwilson / SGwilson_client）的核心状态分析

### 本节导读

11.4 节我们看了**通用怪物 SG** 怎么用 commonstates 复用——通常 50 行就够。但**玩家 SG（SGwilson）完全是另一个量级**：

- 文件大小：**19000+ 行**
- state 数量：**330+ 个**
- 涵盖：所有玩家动作（移动 / 战斗 / 砍树 / 挖矿 / 建造 / 烹饪 / 钓鱼 / 滑冰 / 划船 / 骑牛 / 骑猪 / 月亮变身 / 海狸变身 / ...）

**为什么这么大？** 因为玩家 SG 包含**所有 mod 都会接触的玩家行为**——任何"添加新动作"、"修改原版动作"、"自定义动画"的 mod —— **都要改 SGwilson**。

本节我们**不可能逐个讲 330 个 state**——而是给你一份**鸟瞰式地图**：

- SGwilson 的 state 怎么分类？
- 每类 state 的入口和共性是什么？
- mod 想改某个动作时——**应该看哪几个 state**？
- SGwilson 和 SGwilson_client 的差异具体在哪？
- 改 SG 的 3 种方式（AddStategraphState / 自定义 PostInit / 替换 deststate）各自适合什么场景？

读完 11.5——你将能**在 19000 行代码里精准定位**任何想改的 state——**不再迷路在 SGwilson 的海洋**。

> **新手**从 11.5.1-11.5.3 起步——理解 SGwilson 体量、state 主题分类、核心 state 解读；**进阶读者**继续看 11.5.4-11.5.6，深入玩家专属机制（bufferedaction / 自动装备 / talk）、SGwilson_client 的镜像设计、mod 修改的 3 种路径；**老手**跳到 11.5.7-11.5.8，看 mod 修改 SGwilson 的实战案例 + 6 个最容易踩的坑。

---

### 11.5.1 快速入门：从一个数字看 SGwilson 的体量

#### 第一步：数字震撼

```
ls scripts/stategraphs/SGwilson.lua
-rw-r--r-- 19000+ lines

grep -c "^    State{" scripts/stategraphs/SGwilson.lua
330  -- state 总数

grep -c "^    State{" scripts/stategraphs/SGwilson_client.lua
~140  -- 客户端版 state 数
```

**对比**：典型怪物 SG 才**~30 个 state**——SGwilson 是**~10 倍以上**。

#### 第二步：为什么这么大？

玩家是游戏里**最复杂的实体**——

- **大量装备类动作**：装斧头有 chop、装矛有 attack_jab、装钓竿有 fishing_pre / fishing_loop / fishing_pst...
- **多种交互场景**：吃东西（eat）、给东西（give）、打开容器（rummage）、读书（read）、施法（castspell）...
- **多种状态变化**：变成鬼魂（ghost）、变成海狸（beaver）、变成幽灵（ghost）、骑生物（mounted）、坐在椅子上（sitting）...
- **特殊动作**：跳船（hop）、划船（row）、滑冰（skating）、攀爬（climbing）...

**约 330 个 state** ≈ 玩家**所有可能的动作 / 姿态**的并集。

#### 第三步：mod 修改的常见痛点

- "我想加个新动作怎么办？" —— 11.5.6 节解答
- "我想修改 chop 动作的伤害" —— 改 chop state 的 timeline
- "我想给玩家加新表情" —— emote 系统（11.5.4）
- "我想让玩家在某状态下不能被攻击" —— 加 noattack tag

**所有这些**都要先**找到对应 state**——从 19000 行里捞——本节给你"分类地图"。

---

### 11.5.2 快速入门：SGwilson 的 state 主题分类

#### 12 大类（约 330 个 state 的分类）

| 类别 | 代表 state | 数量 | 关键文件位置 |
| --- | --- | --- | --- |
| **基础姿态** | idle / death / wakeup / hit | ~10 | 4068, 3569, 2556, 12524 |
| **移动** | walk / run / migrate / boating | ~30 | 11267, 18972 |
| **战斗** | attack / blowdart_attack / catapult_attack / ... | ~40 | 10641 |
| **工作** | chop / chop_start / mine / mine_start / dig / hammer | ~30 | 4880, 4908, 5052 |
| **建造** | build / construct / decorate | ~15 | |
| **施法** | castspell / channel / ... | ~25 | |
| **食用** | eat / opengift / drink / ... | ~10 | |
| **物品交互** | pickup / give / rummage / item_in / item_out | ~20 | 12115, 12142 |
| **变身** | beaver / werewolf / ghost / mooncurse | ~30 | |
| **骑乘** | mounted / mountedidle / dismount | ~15 | |
| **特殊场景** | row / hop / skating / sing / dancing | ~25 | |
| **emote / 社交** | emote / talk / wave / dance / sit | ~30 | 18374 |

**有了这张表**——mod 想"改 chop 动画" → 直接看 4908 附近——**不用从头读**。

#### 各分类的"入口 state"

| 类别 | 入口 state（actionhandler 切到这里）|
| --- | --- |
| 砍树 | `chop_start` |
| 采矿 | `mine_start` |
| 挖掘 | `dig_start` |
| 建造 | `build` |
| 烹饪 | `dolongaction` |
| 攻击 | `attack` |
| 施法 | `castspell` |
| 喂食 | `give` |
| 检查 | `dolongaction` 或 `inspect` |

#### state 命名规律

- **`xxx_pre / xxx / xxx_pst`** —— 三段动画（起手、主体、收尾）
- **`xxx_start`** —— 进入复杂动作的"前置"state
- **`do_xxx`** —— 业务执行型 state（dolongaction / dotalk）
- **`mounted_xxx`** —— 骑乘版本的对应 state
- **`item_in / item_out`** —— 装备拾取/放下时的轻量 state

---

### 11.5.3 快速入门：核心 state 解读

#### idle —— 玩家的"枢纽 state"

`SGwilson.lua:4068`：

```4068:4120:scripts/stategraphs/SGwilson.lua
    State{
        name = "idle",
        tags = { "idle", "canrotate" },

        onenter = function(inst, pushanim)
			if inst.sg.lasttags and not inst.sg.lasttags["busy"] then
				inst.components.locomotor:StopMoving()
			else
				inst.components.locomotor:Stop()
				inst.components.locomotor:Clear()
			end
			inst:ClearBufferedAction()

            local drownable = inst.components.drownable
            if drownable then
                local fallingreason = drownable:GetFallingReason()
                if fallingreason == FALLINGREASON.OCEAN then
                    inst.sg:GoToState("sink_fast")
                    return
                elseif fallingreason == FALLINGREASON.VOID then
                    inst.sg:GoToState("abyss_fall")
                    return
                end
            end

            inst.sg.statemem.ignoresandstorm = true

            if inst.components.rider:IsRiding() then
                inst.sg:GoToState("mounted_idle", pushanim)
                return
            end

            local equippedArmor = inst.components.inventory:GetEquippedItem(EQUIPSLOTS.BODY)
            if equippedArmor ~= nil and equippedArmor:HasTag("band") then
                inst.sg:GoToState("enter_onemanband", pushanim)
                return
            end
            ...
```

**关键观察**：
- **`tags = { "idle", "canrotate" }`** —— "idle" 标签让所有"准备接受输入"的检查通过；"canrotate" 允许旋转
- **onenter 大量分支** —— idle 是"动作完成后的默认归宿"——但根据玩家当前状态（骑乘 / 游泳 / 装备 onemanband / ...）切到对应的"特化 idle"
- **`ClearBufferedAction()`** —— 进入 idle 时清空 BufferedAction

**改 idle 的典型 mod 场景**：
- 给玩家加"周期性自动回血"——但 idle 期间才生效
- 修改 funny_idle 的随机概率

#### chop_start / chop —— 砍树的两段

`SGwilson.lua:4880` 和 `4908`：

```4879:4905:scripts/stategraphs/SGwilson.lua
    State{
        name = "chop_start",
        tags = { "prechop", "working" },

        onenter = function(inst)
            inst.components.locomotor:Stop()
            inst.AnimState:PlayAnimation(inst:HasTag("woodcutter") and "woodie_chop_pre" or "chop_pre")
			inst:AddTag("prechop")
        end,
        ...
```

**chop_start**：起手动画 chop_pre。

```4907:4956:scripts/stategraphs/SGwilson.lua
    State{
        name = "chop",
        tags = { "prechop", "chopping", "working" },

        onenter = function(inst)
            inst.sg.statemem.action = inst:GetBufferedAction()
            inst.sg.statemem.iswoodcutter = inst:HasTag("woodcutter")
            inst.AnimState:PlayAnimation(inst.sg.statemem.iswoodcutter and "woodie_chop_loop" or "chop_loop")
			inst:AddTag("prechop")
        end,

        timeline =
        {
            ----------------------------------------------
            --Woodcutter chop

            TimeEvent(2 * FRAMES, function(inst)
                if inst.sg.statemem.iswoodcutter then
                    inst:PerformBufferedAction()
                end
            end),
            ...
```

**chop**：循环砍树 + timeline 触发业务（PerformBufferedAction）。

**关键观察**：
- **iswoodcutter 分支** —— Woodie（伍迪）专用动画 + 节奏
- **timeline 第 2 帧 vs 第 X 帧** —— 不同身份的"接触帧"不同
- **支持"按住按钮自动连砍"** —— timeline 第 10 帧检查输入按住状态

#### attack —— 战斗主 state

```10641:...:scripts/stategraphs/SGwilson.lua
    State{
        name = "attack",
        tags = { "attack", "notalking", "abouttoattack", "autopredict" },
        ...
```

**关键 tag**：
- `"attack"` —— 互斥锁（同一时间不能再攻击）
- `"abouttoattack"` —— 玩家"准备出招"——某些目标可以闪避
- `"autopredict"` —— 客户端自动预测同步
- `"notalking"` —— 攻击时不能说话

#### hit —— 受击

`SGwilson.lua:12524`——玩家被打的标准 state——`onenter` 播 hit 动画 + 短暂 noattack 保护。

#### death —— 死亡

`SGwilson.lua:3569`——核心死亡 state——onenter 播死亡动画 + spawn 鬼魂或 corpse + 触发 ms_playerdied 等。

---

### 11.5.4 进阶：玩家专属机制

#### 机制 1：BufferedAction 的"动作流水线"

玩家 SG 几乎所有动作 state 都通过 BufferedAction 工作：

```
[1] 玩家点击物品 → PlayerActionPicker 创建 BufferedAction
[2] BufferedAction 包含 (action, target, invobject, doer)
[3] sg:GoToState("xxx_state", buffaction)
[4] state 的 timeline 在某帧调 inst:PerformBufferedAction()
[5] PerformBufferedAction 执行 ACTIONS.XXX.fn(act)
```

**几乎所有 state 的 onexit 都包含**：

```lua
onexit = function(inst)
    if inst.bufferedaction == inst.sg.statemem.action then
        inst:ClearBufferedAction()
    end
end,
```

**避免动作中途切换 state 时 buffered action 悬挂**。

#### 机制 2：自动装备

玩家执行某动作前——SGwilson **自动**给玩家装备需要的工具：

- chop 之前自动装斧头
- mine 之前自动装镐
- dig 之前自动装铲

这是通过 `actionhandler` 的 deststate 函数实现：

```lua
ActionHandler(ACTIONS.CHOP, function(inst, action)
    -- 检查手里有没有斧头
    -- 没有 → 切到 "doequip" state 自动装备
    -- 有 → 切到 "chop_start"
end)
```

**修改这个机制 mod 慎重**——影响所有玩家所有动作。

#### 机制 3：talk（说话动画）

玩家 talk 是个特殊 state——

```lua
State{
    name = "talk",
    tags = { "idle", "talking" },
    onenter = function(inst, noanim)
        inst.AnimState:PlayAnimation(noanim and "idle_loop" or "dial_loop", true)
    end,
    -- ...
}
```

**"idle" tag 也在这里**——说话时仍能接受新输入——不会卡住玩家。

#### 机制 4：变身 state 系列

- **beaver_xxx** —— Woodie 海狸形态（约 20 个 state）
- **werewolf_xxx** —— Webber/Walani 等的变身形态
- **ghost_xxx** —— 鬼魂形态
- **mooncurse_xxx** —— 月亮诅咒形态

每种"变身"都有自己的完整 state 集合——**等于又写一份玩家 SG**。

---

### 11.5.5 进阶：SGwilson_client 是 SGwilson 的"轻量镜像"

#### 第一步：SGwilson_client 的本质

回到 9.5 / 11.3 提到的**双 SG 设计**：

- **SGwilson** —— 服务端权威 SG —— **完整业务逻辑**
- **SGwilson_client** —— 客户端预测 SG —— **只播动画 + 客户端可见 tag** —— **不调 PerformBufferedAction**

#### 第二步：典型差异

服务端版（SGwilson.lua）：

```lua
State{
    name = "dolongaction",
    tags = { "doing", "busy", "nodangle", "keep_pocket_rummage" },
    
    onenter = function(inst, timeout)
        inst.AnimState:PlayAnimation("build_pre")
        ...
    end,
    
    ontimeout = function(inst)
        ...
        inst:PerformBufferedAction()  -- ★ 服务端：执行业务
    end,
}
```

客户端版（SGwilson_client.lua）：

```lua
State{
    name = "dolongaction",
    tags = { "doing", "busy", "nodangle" },
    
    onenter = function(inst, timeout)
        inst.AnimState:PlayAnimation("build_pre")
        ...
    end,
    
    -- ★ 客户端：没有 ontimeout 调 PerformBufferedAction
    -- 等服务端权威 NetVar 同步过来切回 idle
}
```

#### 第三步：SGwilson_client 数量更少（约 140 个）

不是所有 SGwilson 的 state 都需要客户端版——**只有**：
- 长动作（dolongaction / dotalk / ...）
- 走路（walk / run）
- 战斗（attack）
- 部分施法

**短动作 / 瞬时动作**（item_in / item_out / hit / death）—— 客户端不需要预测——**直接走服务端 NetVar 同步**。

#### 第四步：mod 给 wilson_client 加 state 的标准模式

```lua
-- modmain.lua
local custom_state = State{
    name = "myaction",
    tags = { "doing", "busy" },
    onenter = function(inst)
        inst.components.locomotor:Stop()
        inst.AnimState:PlayAnimation("myanim")
    end,
    -- 客户端版：没有 PerformBufferedAction
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
}

AddStategraphState("wilson", custom_state_server)        -- 服务端版
AddStategraphState("wilson_client", custom_state_client) -- 客户端版（去 PerformBufferedAction）
```

---

### 11.5.6 进阶：mod 修改 SGwilson 的 3 种路径

#### 路径 1：AddStategraphState —— 加新 state

```lua
-- modmain.lua
AddStategraphState("wilson", State{
    name = "mymod_dance",
    tags = { "idle", "talking" },
    onenter = function(inst)
        inst.AnimState:PlayAnimation("emote_dance")
    end,
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
})
```

**适用**：mod 加新动作——以前 SGwilson 没有这个 state。

#### 路径 2：AddStategraphActionHandler —— 让某 action 切到这个 state

```lua
AddStategraphActionHandler("wilson", ActionHandler(ACTIONS.MYACTION, "mymod_dance"))
AddStategraphActionHandler("wilson_client", ActionHandler(ACTIONS.MYACTION, "mymod_dance"))
```

**适用**：和路径 1 配合——给新 action 路由到新 state。

#### 路径 3：AddStategraphPostInit —— 修改现有 state

```lua
AddStategraphPostInit("wilson", function(sg)
    -- 修改 idle state 的 onenter
    local old_idle = sg.states.idle
    local old_onenter = old_idle.onenter
    old_idle.onenter = function(inst, pushanim)
        old_onenter(inst, pushanim)  -- 调原版
        -- mod 加自己的逻辑
        if inst:HasTag("blessed") then
            inst.components.health:DoDelta(1)  -- 站着回血
        end
    end
end)
```

**适用**：mod 想"在原版 state 上加额外逻辑"——不替换、只追加。

#### 模式选择决策表

| 需求 | 路径 |
| --- | --- |
| 加全新 state | AddStategraphState |
| 让某 action 用新 state | AddStategraphActionHandler |
| 给现有 state 加额外逻辑 | AddStategraphPostInit + 包装原 onenter |
| 完全替换某 state | AddStategraphState（同名覆盖） |
| 改某 state 的 timeline | AddStategraphPostInit + 改 sg.states.xxx.timeline |

---

### 11.5.7 老手进阶：mod 修改 SGwilson 实战案例

#### 场景 1：让玩家在某 state 期间获得 buff

```lua
AddStategraphPostInit("wilson", function(sg)
    local chop_state = sg.states.chop
    local old_onenter = chop_state.onenter
    chop_state.onenter = function(inst, ...)
        old_onenter(inst, ...)
        if inst:HasTag("woodfriend") then
            inst.components.locomotor:SetExternalSpeedMultiplier(inst, "woodfriend", 1.2)
        end
    end
    
    local old_onexit = chop_state.onexit
    chop_state.onexit = function(inst, ...)
        if old_onexit then old_onexit(inst, ...) end
        inst.components.locomotor:RemoveExternalSpeedMultiplier(inst, "woodfriend")
    end
end)
```

**效果**：带 "woodfriend" tag 的玩家砍树时移速 +20%。

#### 场景 2：在 chop 的 timeline 里加额外特效

```lua
AddStategraphPostInit("wilson", function(sg)
    local chop_state = sg.states.chop
    if chop_state and chop_state.timeline then
        table.insert(chop_state.timeline, TimeEvent(8 * FRAMES, function(inst)
            if inst:HasTag("woodfriend") then
                SpawnPrefab("leaf_fx").Transform:SetPosition(inst.Transform:GetWorldPosition())
            end
        end))
        
        -- 重新排序 timeline
        table.sort(chop_state.timeline, function(a, b) return a.time < b.time end)
    end
end)
```

#### 场景 3：替换某 state（罕用，慎重）

```lua
AddStategraphState("wilson", State{
    name = "death",  -- 同名 → 覆盖
    tags = { "busy", "dead", "noattack", "nopredict", "nomorph" },
    onenter = function(inst)
        -- 完全自定义死亡逻辑
        inst.AnimState:PlayAnimation("death")
        -- 可能触发 mod 自己的"复活机制"
    end,
})
```

**注意**：替换原 state 时——**老逻辑全部丢失**——mod 必须负责所有原本的副作用。**通常不推荐——优先用 PostInit 包装**。

---

### 11.5.8 老手进阶：六个常见陷阱

#### 陷阱 1：改 wilson 没改 wilson_client

**症状**：mod 加新 state、加 ActionHandler——主机端测试 OK——联机时客户端动作卡住。  
**原因**：客户端 SG 没注册——预测路径走不通。  
**修复**：永远成对：`AddStategraphState("wilson", ...)` + `AddStategraphState("wilson_client", ...)`。

#### 陷阱 2：PostInit 时机不对

**症状**：

```lua
AddStategraphPostInit("wilson", function(sg)
    local idle = sg.states.idle
    -- idle 是 nil
end)
```

**原因**：PostInit 在 SG 加载完后调用——但有些 mod 加载顺序导致 sg.states 还没构建完。  
**修复**：判 nil：

```lua
if sg.states and sg.states.idle then
    -- ...
end
```

#### 陷阱 3：包装 onenter 没接全部参数

```lua
-- ❌
old_idle.onenter = function(inst)
    old_onenter(inst)  -- 漏了 pushanim
end
```

**修复**：用 `...`：

```lua
old_idle.onenter = function(inst, ...)
    old_onenter(inst, ...)
end
```

#### 陷阱 4：覆盖某 state 没考虑装备分支

**症状**：mod 替换了 idle state——玩家骑牛时——idle 没切到 mounted_idle——卡住。  
**原因**：原 idle 有大量"骑乘 / 武装 / 流血"等分支——mod 替换时丢失了。  
**修复**：用 PostInit 包装而非完全替换。

#### 陷阱 5：timeline 修改没排序

**症状**：mod 在 chop state timeline 末尾加了一个 `TimeEvent(5 * FRAMES, ...)`——但前面已经有第 10 帧的事件——`TimeEvent(5 * FRAMES, ...)` 没触发。  
**原因**：timeline 是按时间顺序遍历——后加的早时间项被跳过。  
**修复**：

```lua
table.insert(chop_state.timeline, ...)
table.sort(chop_state.timeline, function(a, b) return a.time < b.time end)
```

#### 陷阱 6：直接改 sg.states 但 mod 顺序乱

**症状**：mod A 改了 idle、mod B 也改了 idle——A 改的没了。  
**原因**：B 的 PostInit 在 A 之后跑——B 的修改覆盖 A 的。  
**修复**：mod 之间通常无法保证顺序——**用包装 fn**而非直接赋值——这样多个 mod 包装可以"层层生效"：

```lua
AddStategraphPostInit("wilson", function(sg)
    local old_onenter = sg.states.idle.onenter
    sg.states.idle.onenter = function(inst, ...)
        old_onenter(inst, ...)
        -- mod 自己的逻辑
    end
end)
```

#### 设计经验三条

**经验 ①：永远用 PostInit + 包装、避免完全替换**

```lua
-- ✓ 推荐
AddStategraphPostInit("wilson", function(sg)
    local old_fn = sg.states.idle.onenter
    sg.states.idle.onenter = function(inst, ...)
        old_fn(inst, ...)
        -- 加新逻辑
    end
end)

-- ✗ 避免
AddStategraphState("wilson", State{ name = "idle", ... })  -- 完全替换原 idle
```

**经验 ②：state 命名加 mod 前缀**

```lua
AddStategraphState("wilson", State{
    name = "mymod_dance",  -- 永远加前缀
    ...
})
```

避免和别的 mod 撞名。

**经验 ③：测 4 种部署模式**

mod 改 SG 的影响特别广——必须测：单机、主机端联机、客户端联机、dedicated。**每种都要跑通**。

---

### 11.5.9 小结

**SGwilson 一句话总结**：**19000+ 行、330+ state 的玩家 SG——按 12 大主题分类——SGwilson_client 是其轻量镜像（约 140 state）——mod 改 SG 用 AddStategraphState/ActionHandler/PostInit 三件套**。

**速查表**

| 想找某 state | 入口 |
| --- | --- |
| idle | `SGwilson.lua:4068` |
| chop | `SGwilson.lua:4908` |
| chop_start | `SGwilson.lua:4880` |
| mine | `SGwilson.lua:5052` |
| attack | `SGwilson.lua:10641` |
| run | `SGwilson.lua:11267` |
| hit | `SGwilson.lua:12524` |
| death | `SGwilson.lua:3569` |
| wakeup | `SGwilson.lua:2556` |
| dolongaction | `SGwilson.lua:8217` |

**3 条修改路径**

| 需求 | API |
| --- | --- |
| 加新 state | `AddStategraphState(sg, State{...})` |
| Action 路由 | `AddStategraphActionHandler(sg, ActionHandler(ACTIONS.X, "state"))` |
| 改现有 state | `AddStategraphPostInit(sg, function(sg) ... end)` + 包装 onenter |

**6 个陷阱排雷顺序**

1. 改 wilson 没改 wilson_client → 永远成对
2. PostInit 时机 → 判 nil
3. 包装 onenter 漏参数 → 用 `...`
4. 完全替换 state → 用 PostInit 包装
5. timeline 没排序 → table.sort
6. mod 顺序乱 → 包装 fn 层层生效

**3 条设计经验**

- ① **PostInit + 包装** 优于完全替换
- ② **state 名加 mod 前缀**
- ③ **测 4 种部署模式**

> **下一节预告**：11.6 节是整章收尾——**实战：为自定义角色添加新动作状态** —— 把 11.1 ~ 11.5 学到的所有 SG 知识串起来——**完整搭建一个自定义角色的 mod**，包含自定义 SG / 自定义动作 / 自定义动画。读完 11.6，你将真正具备**独立设计自定义角色 mod 的能力**——这是把 SG 知识"落地"到玩法的最终一步。

## 11.6 实战：为自定义角色添加新动作状态

### 本节导读

11.1 ~ 11.5 我们把 SG 系统**从下到上拆透**了：State 三件套（11.1）、StateTag（11.2）、动画驱动（11.3）、commonstates（11.4）、SGwilson（11.5）。但**任何"原理章"都需要一个落地章把所有零件串起来**——本节就是 11 章的"实战收尾"。

我们以一个**完整可运行的需求**为主线：

> **「打坐回血」（meditate）**——玩家右键自己（或按一个 mod 自定义键），盘腿坐下、播放打坐动画 3 秒、每秒回 5 点生命；期间被攻击、被冻、操作位移键都要中断；中断后扣除 buffer 但不掉血；联机预测要顺。

围绕这一个动作——我们**从最简的"骨架版"一路升级到生产级"全联机版 + 中断保护版"**——这正好是 11 章所有知识点的"使用场景集合"：

- 11.6.1 ~ 11.6.3 **快速入门**——给 wilson 加新 state、Action、ActionHandler，最小可跑
- 11.6.4 ~ 11.6.6 **进阶**——客户端 SG 镜像、三段式动画 + timeline + tag 综合运用、为自定义角色 prefab 写**独立 SG 文件**
- 11.6.7 ~ 11.6.8 **老手进阶**——边界保护（中断 / 死亡 / 取消 / 装备切换）、6 个常见陷阱
- 11.6.9 **小结**——从"动作设计"到"上线"的完整 checklist

> **新手**从 11.6.1 起步，跟着代码块敲一遍，能跑通就行；**进阶读者**继续看 11.6.4 ~ 11.6.6，理解联机镜像和动画分段；**老手**直接跳到 11.6.7 ~ 11.6.8，看边界保护和踩坑总结。

---

### 11.6.1 快速入门：实战目标——「打坐回血」需求拆解

#### 第一步：从需求到状态机草图

我们先不写代码——先**画出状态机**：

```
[idle]
  │ (玩家发起 meditate)
  ▼
[meditate_pre]   ← 0.5s 盘腿动作
  │ (animover)
  ▼
[meditate_loop] ← 2.0s 打坐 + 每秒回 5 血
  │ (3s 计时到 / 被攻击 / 移动 / 死亡)
  ▼
[meditate_pst]  ← 0.4s 起身
  │ (animover)
  ▼
[idle]
```

**3 个 state**——和 11.4 节的 commonstates 套路一致（pre / loop / pst）：

- `meditate_pre`——预备动作，不可被打断的"过渡"
- `meditate_loop`——核心 buff state，每秒回血、计时到 3s 切到 pst
- `meditate_pst`——起身收尾，回 idle

#### 第二步：从需求到字段表

把需求**逐项映射到 State 的 7 大字段**（11.1.2 学过）：

| 需求 | 落到哪 |
| --- | --- |
| 盘腿动画 | `onenter`：`PlayAnimation("meditate_pre")` |
| 不能再做别的事 | `tags = {"busy", "doing"}` |
| 起身音效 | `timeline`：`TimeEvent(5*FRAMES, ...PlaySound...)` |
| 每秒回 5 血 | `onupdate` 里累加 dt，到 1s 触发 `health:DoDelta(5)` |
| 3s 计时到 → pst | `onenter` 里 `inst.sg:SetTimeout(3)` + `ontimeout` 切 pst |
| 被攻击中断 | `events = { EventHandler("attacked", ...) }` |
| 移动中断 | `events = { EventHandler("locomote", ...) }` |
| 死亡中断 | `events = { EventHandler("death", ...) }` |
| 退出时清场 | `onexit`：`StopSound`、`StateTag` 收尾 |

> **关键观察**：**"动作 buff" 是 SG 表达力最强的形式之一**——你写"3 个 state + 7 个字段"——就把"盘腿 / 回血 / 中断 / 起身"全部表达完毕。**这是 SG 相对于"用 component 自己写定时器"最大的优势——表达力强、生命周期清晰**。

#### 第三步：需要的工程文件

| 文件 | 内容 |
| --- | --- |
| `modmain.lua` | 注册 Action、ActionHandler、State |
| `scripts/main/sg_meditate.lua`（建议拆分） | 3 个 state 的实现 |
| `scripts/actions/meditate_action.lua` | Action 定义 |
| `anim/meditate.zip` | 三段式动画（mod 作者准备 .scml + Spriter） |

> 进阶版（11.6.6）我们会把它做成**自定义角色 prefab + 独立 SG**——但 11.6.1 ~ 11.6.5 先以"给 wilson 加 state"为主线——更通用、入门曲线更平。

---

### 11.6.2 快速入门：最小可跑版本——只加一个 state

#### 第一步：先不要 Action，直接 GoToState 跑通骨架

很多新手第一次写 SG 卡在"Action 怎么定义"——其实**先抛开 Action**，用最直接的方式触发 state：**按一个键，调一行 `inst.sg:GoToState("meditate_loop")`**——这样我们能**专心调 state 内部逻辑**。

```lua
-- modmain.lua（最简骨架版）
local State = GLOBAL.State
local TimeEvent = GLOBAL.TimeEvent
local EventHandler = GLOBAL.EventHandler
local FRAMES = GLOBAL.FRAMES

AddStategraphState("wilson", State{
    name = "meditate_loop",
    tags = { "busy", "doing" },

    onenter = function(inst)
        inst.components.locomotor:Stop()
        inst.AnimState:PlayAnimation("research", true)  -- 借用研究台动画
        inst.sg.statemem.heal_t = 0
        inst.sg:SetTimeout(3)
    end,

    onupdate = function(inst, dt)
        inst.sg.statemem.heal_t = inst.sg.statemem.heal_t + dt
        if inst.sg.statemem.heal_t >= 1 then
            inst.sg.statemem.heal_t = inst.sg.statemem.heal_t - 1
            inst.components.health:DoDelta(5)
        end
    end,

    ontimeout = function(inst)
        inst.sg:GoToState("idle")
    end,

    events = {
        EventHandler("attacked", function(inst)
            inst.sg:GoToState("hit")
        end),
    },

    onexit = function(inst)
        inst.AnimState:Stop()
    end,
})
```

#### 第二步：用 console / mod hotkey 触发

最快的"先跑通"——用 `~` 控制台直接发：

```lua
-- 控制台输入
ThePlayer.sg:GoToState("meditate_loop")
```

**应该看到**：人物站住、播研究台动画、生命每秒 +5、3 秒后回 idle、期间被打会切到 hit。**到这里 state 内部逻辑已经验证完毕**——和"怎么触发它"完全解耦。

#### 第三步：观察并理解每个字段做了什么

逐字段解读这个最小版本：

- **`tags = { "busy", "doing" }`**——告诉游戏"我在忙"——其他系统会因此**禁止再叠 action**（11.2.4 学过 `busy` 是核心动作 tag）
- **`onenter` 里 `locomotor:Stop()`**——SG 要负责让玩家**停下来**——locomotor 是移动核心 component（参考 9.3）
- **`PlayAnimation("research", true)` 第二参数 = loop**——动画自动循环（11.3.2 学过）
- **`inst.sg.statemem.heal_t`**——`statemem` 是 state 私有变量——切 state 时自动清空（11.1.4 学过 statemem）
- **`SetTimeout(3)` + `ontimeout`**——3 秒后触发 `ontimeout` 切回 idle（11.1.4 学过）
- **`onupdate(dt)`**——每帧调一次——这里用于"累加 dt 到 1s 触发回血"——这是 onupdate 的**正确用法**（11.1.4 提示过 onupdate 慎用，但"累加计数器"是合法用例）
- **`events.attacked`**——被攻击事件触发——切到 hit state（hit 是 wilson 内置的"被打"state）
- **`onexit` 里 `AnimState:Stop()`**——退出 state 时停掉循环动画——避免动画"溢出"到下一个 state

> **核心结论**：**11.1 学的"7 字段 + statemem + SetTimeout + events"——在 28 行代码里全部用到**——这就是 SG 的表达密度。

#### 第四步：跑通后立即拆分文件

代码堆 modmain 很快就乱——拆出来：

```lua
-- scripts/sg_meditate.lua
local State = GLOBAL.State
local TimeEvent = GLOBAL.TimeEvent
local EventHandler = GLOBAL.EventHandler
local FRAMES = GLOBAL.FRAMES

local function MakeMeditateLoop()
    return State{
        name = "meditate_loop",
        -- ... 略 ...
    }
end

return {
    meditate_loop = MakeMeditateLoop(),
}
```

```lua
-- modmain.lua
modimport("scripts/sg_meditate.lua")
-- 或：
local sg_states = require("sg_meditate")
AddStategraphState("wilson", sg_states.meditate_loop)
```

> 这一步看似多余——但当 state 数量从 1 升到 3 升到 10 时——**拆分文件是把 mod 维护住的关键**。

---

### 11.6.3 快速入门：升级——用 Action 触发，进入"产品级"接入

#### 第一步：为什么不能一直靠按键

11.6.2 我们用控制台 / 一个键直接 `GoToState`——这种"硬编码触发"有 3 个问题：

1. **绕过了 BufferedAction**——玩家"右键自己 → 选打坐"这种 UI 流程没法用
2. **客户端预测难写**——`GoToState` 不会自动同步给服务端
3. **不通用**——别的 mod、别的角色没法复用你的 state——只有"按键"能进

**正解**：把 meditate 包装成一个 **Action**——用 `inst.components.playercontroller:DoAction` 触发——经过 BufferedAction → ActionHandler → State 的标准链路（参考 7 章 Action 系统）。

#### 第二步：定义 Action

新增 `scripts/actions/action_meditate.lua`：

```lua
local Action = GLOBAL.Action
local ACTIONS = GLOBAL.ACTIONS
local AddAction = GLOBAL.AddAction or _G.AddAction  -- 不同环境取法

local MEDITATE = Action({
    priority = 0,
    rmb = true,
    distance = 1,
})
MEDITATE.id = "MEDITATE"
MEDITATE.str = "打坐"
MEDITATE.fn = function(act)
    if act.doer ~= nil and act.doer.components.health ~= nil
        and not act.doer.components.health:IsDead() then
        return true  -- 返回 true 让 SG 真正进入 meditate state
    end
    return false
end

AddAction("MEDITATE", "打坐", MEDITATE.fn)
STRINGS.ACTIONS.MEDITATE = "打坐"
```

> 关于 `AddAction` / `Action` 的源码——7 章 7.2 节讲过——这里**关注两个字段**：
> - `rmb = true`——右键触发
> - `distance = 1`——必须靠近目标 1 距离内（自己点自己肯定满足）
> - **`fn`**——这是 Action 真正"生效"的回调——通常用于实例化效果（这里我们让 state 来做"回血"，所以 fn 只做合法性判断）

#### 第三步：把 Action 路由到 state——ActionHandler

```lua
-- modmain.lua
local ActionHandler = GLOBAL.ActionHandler
local ACTIONS = GLOBAL.ACTIONS

AddStategraphActionHandler("wilson", ActionHandler(ACTIONS.MEDITATE, "meditate_pre"))
AddStategraphActionHandler("wilson_client", ActionHandler(ACTIONS.MEDITATE, "meditate_pre"))
```

**关键点**：**两个 SG 都要注册**——服务端 `wilson` 和客户端 `wilson_client`——否则联机预测不通（11.5.8 陷阱 1 强调过）。

> **路由含义**：当玩家发起 ACTIONS.MEDITATE → 玩家组件层处理 BufferedAction → ActionHandler 找到对应 state 名（"meditate_pre"）→ SG 切到该 state。
> **这条路径就是 SG 和 Action 系统的接口**——理解这条路径就理解 90% 的玩家动作 mod。

#### 第四步：让玩家能"右键自己"触发——ComponentAction

光有 Action 还不够——玩家在 UI 上**怎么调起这个 Action**？答案是 **ComponentAction**——在 7 章 7.6 节讲过：

```lua
-- modmain.lua
AddComponentAction("CHARACTER", "health", function(inst, doer, actions, right)
    if right and inst == doer
       and inst.components.health ~= nil
       and not inst.components.health:IsDead() then
        table.insert(actions, ACTIONS.MEDITATE)
    end
end)
```

**这段做了什么**：
- **`AddComponentAction`**——给 health 组件挂一个"动作生成器"
- **`"CHARACTER"`**——动作类型——表示"对角色的动作"
- **`right`**——只在右键菜单出现
- **`inst == doer`**——只有"对自己"才出现（不能给别人打坐）

> 现在玩家右键自己——会看到"打坐"选项——点击后 → 走 BufferedAction → ActionHandler → SG 切到 `meditate_pre` → state 接管。

#### 第五步：补齐 meditate_pre / meditate_pst

现在我们补出三段式：

```lua
AddStategraphState("wilson", State{
    name = "meditate_pre",
    tags = { "busy", "doing" },

    onenter = function(inst)
        inst.components.locomotor:Stop()
        inst.AnimState:PlayAnimation("research_pre")  -- 借用研究台预备动画
    end,

    events = {
        EventHandler("animover", function(inst)
            if inst.AnimState:AnimDone() then
                inst.sg:GoToState("meditate_loop")
            end
        end),
    },
})

AddStategraphState("wilson", State{
    name = "meditate_pst",
    tags = { "idle" },  -- 退出阶段不再 busy

    onenter = function(inst)
        inst.AnimState:PlayAnimation("research_pst")
    end,

    events = {
        EventHandler("animover", function(inst)
            if inst.AnimState:AnimDone() then
                inst.sg:GoToState("idle")
            end
        end),
    },
})

-- meditate_loop 改为 ontimeout 切到 meditate_pst
AddStategraphState("wilson", State{
    name = "meditate_loop",
    tags = { "busy", "doing" },

    onenter = function(inst)
        inst.AnimState:PlayAnimation("research", true)
        inst.sg.statemem.heal_t = 0
        inst.sg:SetTimeout(3)
    end,

    onupdate = function(inst, dt)
        inst.sg.statemem.heal_t = inst.sg.statemem.heal_t + dt
        if inst.sg.statemem.heal_t >= 1 then
            inst.sg.statemem.heal_t = inst.sg.statemem.heal_t - 1
            inst.components.health:DoDelta(5)
        end
    end,

    ontimeout = function(inst)
        inst.sg:GoToState("meditate_pst")
    end,

    events = {
        EventHandler("attacked", function(inst)
            inst.sg:GoToState("hit")  -- 中断
        end),
    },

    onexit = function(inst)
        inst.AnimState:Stop()
    end,
})
```

> **3 个 state 串成"管道"——pre → loop → pst → idle**——这正是 11.4.1 讲的 commonstates 标准模板。**新手到这里——已经写完一个"产品级"的 mod 动作的 60%**。

---

### 11.6.4 进阶：联机版必备——补齐 wilson_client 客户端镜像

#### 第一步：为什么"在主机能用"远远不够

11.6.3 跑通后——你**单机**测——完美。**主机+本地客户端**测——还行（其实有微卡顿）。**dedicated 服务器 + 远程客户端**测——客户端**右键打坐 → 卡住 0.3 秒 → 才开始播动画**。

**原因**：客户端 SG 没注册 meditate state——客户端 ActionHandler 找不到 → BufferedAction **无法本地预测** → 必须等服务端确认（200ms+ 延迟）→ 玩家感受**卡顿**。

> 11.5.5 学过——`SGwilson_client` 是 SGwilson 的"轻量镜像"——专管**客户端预测**。所有玩家可触发的 action——必须**两边都有 state**。

#### 第二步：客户端 state 的标准模板

参考神话未加密 mod 中 `useyjp` 的客户端镜像（项目 `mods/联机版mod/神话未加密/main/sg.lua:162` 起）：

```lua
AddStategraphState("wilson_client", State{
    name = "meditate_pre",
    tags = { "busy", "doing" },

    onenter = function(inst)
        inst.components.locomotor:Stop()
        inst.AnimState:PlayAnimation("research_pre")

        local buffaction = inst:GetBufferedAction()
        if buffaction ~= nil then
            inst:PerformPreviewBufferedAction()  -- ★ 客户端"先演一遍"
        end
        inst.sg:SetTimeout(2)  -- 客户端兜底超时
    end,

    onupdate = function(inst)
        if inst:HasTag("doing") then
            if inst.entity:FlattenMovementPrediction() then
                inst.sg:GoToState("idle", "noanim")  -- ★ 服务端确认到达 → 切 idle
            end
        elseif inst.bufferedaction == nil then
            inst.sg:GoToState("idle")
        end
    end,

    ontimeout = function(inst)
        inst:ClearBufferedAction()
        inst.sg:GoToState("idle")
    end,
})
```

**4 个客户端独有的关键点**：

1. **`PerformPreviewBufferedAction()`**——客户端"先演一遍"——让玩家立刻看到反馈，不等服务端
2. **`SetTimeout(2)`**——超时兜底——服务端 2 秒还不确认就退出
3. **`FlattenMovementPrediction()`**——预测对齐成功——这时让客户端"贴回"服务端 idle
4. **`GoToState("idle", "noanim")`**——`"noanim"` 参数告诉 idle state 不要重播动画——避免预测对齐时画面闪烁

> **核心结论**：**客户端 state ≠ 把服务端 state 复制一份**——它是"预测层 + 服务端权威结果对齐"的特殊模式——`PerformPreviewBufferedAction` + `FlattenMovementPrediction` 是它的 DNA。

#### 第三步：客户端 _loop 和 _pst 通常可以"省略"

**反直觉**：客户端只需要镜像 **进入入口的那一个 state（meditate_pre）**——**loop 和 pst 通常不需要客户端镜像**。

**为什么**：当 `FlattenMovementPrediction` 检测到服务端进入 meditate_loop——客户端**直接进入 idle "noanim" 模式**——但因为服务端权威 SG 在播动画 + 改 stategraph_tags——**客户端通过 RPC 看到正确的动画 / tag**——视觉上**完全无差**。

> **这是 SGwilson_client 的设计哲学**——客户端只关心"如何快速触发预测 + 如何对齐"——业务逻辑全部由服务端跑。**所以客户端 SG 通常只有 100~150 个 state——而服务端有 330+**。

#### 第四步：sourcse_states / forward_server_states

11.1.2 提到客户端 state 有 3 个特殊字段——这里看真实例子：

```lua
AddStategraphState("wilson_client", State{
    name = "meditate_pre",
    tags = { "busy", "doing" },

    server_states = { "meditate_pre", "meditate_loop", "meditate_pst" },
    forward_server_states = { "meditate_loop", "meditate_pst" },

    onenter = function(inst) ... end,
})
```

**含义**：
- **`server_states`**——服务端这些 state 都"对应"客户端的 meditate_pre——客户端预测路径里**只看到这 3 个 server state 算"匹配"**——不会无意义切到 idle
- **`forward_server_states`**——服务端切到 loop / pst 时——**客户端继续保持当前 state**，不被"反向同步"打断

> **效果**：服务端从 pre → loop → pst → idle 整个 4 秒过程——**客户端只切了 1 次 state（pre → idle "noanim"）**——网络抖动也不会闪。

#### 第五步：测 4 种部署模式

**永远在 mod 上线前测这 4 种**：

| 部署 | 测什么 |
| --- | --- |
| 单机 | state 内部逻辑、动画、回血 |
| 主机 + 本地客户端 | 客户端镜像生效、本地预测顺畅 |
| 主机 + 远程客户端（>50ms 延迟） | 预测对齐、网络抖动 |
| Dedicated + 远程客户端 | 完整生产环境 |

**典型 bug**：dedicated 上 onenter 调了 `inst.HUD:OpenScreen(...)`——HUD 在服务端是 nil → 服务端崩溃。**这种 bug 单机永远测不出来**。

---

### 11.6.5 进阶：综合运用——三段式动画 + timeline + tag + state mem

#### 第一步：把 state mem、timeline、tag 全部用上

11.6.3 的 meditate_loop 用了 `statemem` 和 `events`——但**只用到 SG 表达力的 30%**。生产级版本会综合用：

```lua
AddStategraphState("wilson", State{
    name = "meditate_loop",
    tags = { "busy", "doing", "meditating" },  -- ★ 自定义 tag

    onenter = function(inst, was_interrupted)
        inst.AnimState:PlayAnimation("research", true)
        inst.sg.statemem.heal_t = 0
        inst.sg.statemem.tick_count = 0
        inst.sg:SetTimeout(3)
        inst.SoundEmitter:PlaySoundWithParams("dontstarve/wilson/meditate_loop",
                                              {volume=0.6}, "meditate_sound")
    end,

    onupdate = function(inst, dt)
        inst.sg.statemem.heal_t = inst.sg.statemem.heal_t + dt
        if inst.sg.statemem.heal_t >= 1 then
            inst.sg.statemem.heal_t = inst.sg.statemem.heal_t - 1
            inst.sg.statemem.tick_count = inst.sg.statemem.tick_count + 1
            inst.components.health:DoDelta(5)
            -- 第 3 跳触发"圆满"特效
            if inst.sg.statemem.tick_count == 3 then
                local fx = SpawnPrefab("statue_transition_2")
                if fx then
                    fx.Transform:SetPosition(inst.Transform:GetWorldPosition())
                end
            end
        end
    end,

    timeline = {
        TimeEvent(15 * FRAMES, function(inst)
            -- 0.5s 之后允许移动键中断（pre 阶段不允许中断）
            inst.sg:RemoveStateTag("nointerrupt")
        end),
    },

    ontimeout = function(inst)
        inst.sg:GoToState("meditate_pst")
    end,

    events = {
        EventHandler("attacked", function(inst)
            inst.sg:GoToState("hit")
        end),
        EventHandler("locomote", function(inst)
            -- 玩家按移动键 → 立即起身
            if not inst.sg:HasStateTag("nointerrupt") then
                inst.sg:GoToState("meditate_pst")
            end
        end),
        EventHandler("death", function(inst)
            inst.sg:GoToState("death")
        end),
    },

    onexit = function(inst)
        inst.AnimState:Stop()
        inst.SoundEmitter:KillSound("meditate_sound")  -- ★ 关循环音效
    end,
})
```

#### 第二步：逐步解读"加了什么"

**新增 1：自定义 tag `"meditating"`**

```lua
tags = { "busy", "doing", "meditating" },
```

外部业务可以查这个 tag——例如 mod 加成"打坐时吸血鬼不变身"：

```lua
AddPrefabPostInit("vampire", function(inst)
    inst:WatchWorldState("phase", function(inst, phase)
        if phase == "night" and not inst.sg:HasStateTag("meditating") then
            inst.sg:GoToState("vampire_transform")
        end
    end)
end)
```

> **11.2.7 学过——mod 自定义 tag 必须加 mod 前缀**——这里如果上线建议改成 `"mymod_meditating"`。

**新增 2：`onenter(inst, was_interrupted)`**

接收第二参数——表示"是否从中断里回来"——可以做"二段式打坐：被打断后再次进入跳过 pre"等高级模式。

**新增 3：`statemem.tick_count` 计数器**

每秒回血累加 → 第 3 跳触发圆满特效——这是**用 statemem 实现"state 内进度"**的标准模式。

**新增 4：`PlaySoundWithParams + KillSound`**

循环音效——`onenter` 开 → `onexit` 关——必须配对（11.6.8 陷阱 5 详谈）。

**新增 5：`timeline` 动态去 tag**

```lua
TimeEvent(15 * FRAMES, function(inst)
    inst.sg:RemoveStateTag("nointerrupt")
end),
```

**前 0.5 秒不允许中断**——保护盘腿动画完整性——0.5 秒后允许玩家用移动键起身。
> 11.2.5 学过——动态加 / 去 tag 是**当前 state 期间生效**——切 state 自动重置——所以**安全**。

**新增 6：`locomote` 事件**

```lua
EventHandler("locomote", function(inst)
    if not inst.sg:HasStateTag("nointerrupt") then
        inst.sg:GoToState("meditate_pst")
    end
end),
```

玩家按 W/A/S/D → SG 收到 `locomote` 事件 → 我们手动判断"是否允许中断"——这是 **SG 控制玩家移动**的核心模式。

**新增 7：death 中断**

死亡是"最高优先级"中断——这条**必加**——否则玩家在 meditate_loop 期间死亡，会卡死直到 timeout。

#### 第三步：onenter 之前先加个"前置检查"

实际上面代码还可以更稳——meditate_pre 入口前应该**做 health / sanity 检查**：

```lua
AddStategraphState("wilson", State{
    name = "meditate_pre",
    tags = { "busy", "doing", "nointerrupt" },  -- ★ pre 阶段加 nointerrupt

    onenter = function(inst)
        inst.components.locomotor:Stop()

        -- ★ 满血就不能打坐——节省时间
        if inst.components.health:GetPercent() >= 1 then
            inst.sg:GoToState("idle")
            return
        end
        -- ★ 在水里 / 骑乘时禁止
        if inst.components.rider and inst.components.rider:IsRiding() then
            inst.sg:GoToState("idle")
            return
        end

        inst.AnimState:PlayAnimation("research_pre")
    end,

    events = {
        EventHandler("animover", function(inst)
            if inst.AnimState:AnimDone() then
                inst.sg:GoToState("meditate_loop")
            end
        end),
    },
})
```

> **关键**：**入口检查放 onenter——而不是 ActionHandler 的 fn**——因为 fn 在 client 是 nil（fn 在 server 跑）——客户端预测会"假进入" → 服务端拒绝 → 客户端被强行回退——画面闪。**onenter 检查是双端都跑的最稳路径**。

#### 第四步：动画事件的精细化

如果你不想用 `research`——而是有自己的 .scml——典型动画文件结构：

```
anim/meditate.zip
├── build.bin
├── atlas-0.tex
├── meditate_pre.json   ← 0.5s
├── meditate_loop.json  ← 1.0s（循环）
└── meditate_pst.json   ← 0.4s
```

**对应 state 写法**：

```lua
onenter = function(inst)
    inst.AnimState:SetBank("meditate")  -- 如果是新 bank
    inst.AnimState:SetBuild("meditate")
    inst.AnimState:PlayAnimation("meditate_pre")
end,
```

**或者（推荐）使用 OverrideSymbol**——保留人物 build——只覆盖盘腿动作的 symbol：

```lua
onenter = function(inst)
    inst.AnimState:OverrideSymbol("legs", "meditate", "legs_crossed")
    inst.AnimState:PlayAnimation("research_pre")  -- 借用研究台动画框架
end,

onexit = function(inst)
    inst.AnimState:ClearOverrideSymbol("legs")  -- ★ 退出必须清掉
end,
```

> 11.3.5 学过 OverrideSymbol——这里的"借框架 + 换 symbol"是**最快的自定义动作做法**——比从头画一套完整动画**省 90% 工作量**。

---

### 11.6.6 老手进阶：自定义角色 prefab + 独立 SG 文件

#### 第一步：什么时候需要"独立 SG"

11.6.1 ~ 11.6.5 我们一直在**给 wilson 加 state**——这是 mod 加新动作的**主流路径**。但有 3 种场景需要**独立 SG**：

1. **完全自定义角色（非 wilson）**——例如新建一个"小狼"prefab——不能继承 wilson 的 SG（动画都不一样）
2. **给生物加复杂状态机**——比如 mod 自创"魔法宠物 myth_pet"——需要 idle / follow / attack / dance 多种状态
3. **完全独立的 NPC**——参考"登仙"mod 的 `SGxd_zyg_npc.lua`、`SGxd_xianhe.lua`——它们都是从零建的 SG

**通用模式**：**新 prefab → 新 SG 文件 → 在 prefab fn 里 `inst:SetStateGraph("SGxxx")` 绑定**。

#### 第二步：完整工程结构

```
my_meditator_mod/
├── modmain.lua
├── modinfo.lua
├── scripts/
│   ├── prefabs/
│   │   └── meditator.lua       ← 新 prefab
│   └── stategraphs/
│       └── SGmeditator.lua     ← 独立 SG
└── anim/
    └── meditator.zip
```

**关键**：**独立 SG 文件命名 `SGxxx.lua`——必须放在 `scripts/stategraphs/` 目录**——这是引擎硬编码的查找路径。

#### 第三步：SGmeditator.lua 完整骨架

参考 `mods/联机版mod/登仙/scripts/stategraphs/SGxd_xianhe.lua`、`SGxd_zyg_npc.lua` 的标准结构：

```lua
-- scripts/stategraphs/SGmeditator.lua
require("stategraphs/commonstates")  -- ★ 引入通用 state 库

local actionhandlers = {
    ActionHandler(ACTIONS.MEDITATE, "meditate_pre"),
}

local events = {
    -- SG 级 events（所有 state 都监听）
    CommonHandlers.OnDeath(),
    CommonHandlers.OnSleep(),
    CommonHandlers.OnAttacked(),
    CommonHandlers.OnAttack(),
    CommonHandlers.OnLocomote(true, true),   -- canrun, canwalk
    CommonHandlers.OnFreeze(),
    EventHandler("doaction", function(inst, data)
        -- 自定义 doaction 路由
    end),
}

local states = {
    State{
        name = "idle",
        tags = { "idle", "canrotate" },

        onenter = function(inst, pushanim)
            inst.components.locomotor:StopMoving()
            if pushanim then
                inst.AnimState:PushAnimation("idle_loop", true)
            else
                inst.AnimState:PlayAnimation("idle_loop", true)
            end
        end,
    },

    State{
        name = "meditate_pre",
        tags = { "busy", "doing" },

        onenter = function(inst)
            inst.components.locomotor:Stop()
            inst.AnimState:PlayAnimation("meditate_pre")
        end,

        events = {
            EventHandler("animover", function(inst)
                if inst.AnimState:AnimDone() then
                    inst.sg:GoToState("meditate_loop")
                end
            end),
        },
    },

    -- meditate_loop / meditate_pst 略 ...
}

-- ★ 用 CommonStates 自动加 hit / death / sleep / freeze
CommonStates.AddSimpleState(states, "hit", "hit_anim")
CommonStates.AddSimpleActionState(states, "scared", "scared", 10*FRAMES, {"busy"})
CommonStates.AddDeathState(states)
CommonStates.AddSleepStates(states, {
    starttimeline = nil,
    waketimeline = {
        TimeEvent(20*FRAMES, function(inst)
            inst.SoundEmitter:PlaySound("dontstarve/wilson/wakeup")
        end),
    },
})
CommonStates.AddCombatStates(states, {})
CommonStates.AddFrozenStates(states)
CommonStates.AddWalkStates(states, {
    starttimeline = nil,
    walktimeline = nil,
})
CommonStates.AddRunStates(states, {
    starttimeline = nil,
    runtimeline = {
        TimeEvent(0*FRAMES, function(inst)
            inst.SoundEmitter:PlaySound("dontstarve/creatures/run")
        end),
    },
})
CommonStates.AddIdle(states)

return StateGraph("meditator", states, events, "idle", actionhandlers)
```

**4 个关键点**：

1. **`require("stategraphs/commonstates")`**——引入 commonstates（11.4 节）
2. **`CommonHandlers.OnXxx()` 一次注册 5 + 个常用事件**——节省 80 行
3. **`CommonStates.AddXxxStates`**——把 hit / death / sleep / walk / run 等"通用 state"自动加好——节省 200 + 行
4. **`StateGraph("meditator", states, events, "idle", actionhandlers)`**——StateGraph 构造函数 4 参数——名字、states、SG 级 events、初始 state、actionhandlers

#### 第四步：在 prefab 里绑定 SG

```lua
-- scripts/prefabs/meditator.lua
local assets = {
    Asset("ANIM", "anim/meditator.zip"),
}

local prefabs = {}

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    inst.entity:AddSoundEmitter()

    MakeCharacterPhysics(inst, 50, 0.5)

    inst.AnimState:SetBank("meditator")
    inst.AnimState:SetBuild("meditator")
    inst.AnimState:PlayAnimation("idle_loop", true)

    inst:AddTag("character")
    inst:AddTag("scarytoprey")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    -- ★ component 配置
    inst:AddComponent("locomotor")
    inst.components.locomotor.runspeed = 4
    inst.components.locomotor.walkspeed = 2

    inst:AddComponent("health")
    inst.components.health:SetMaxHealth(150)

    inst:AddComponent("combat")
    inst.components.combat:SetDefaultDamage(10)

    inst:AddComponent("locomotor")  -- meditator 也是 character——locomotor 必须有
    inst:AddComponent("inspectable")

    -- ★ 关键：绑定独立 SG
    inst:SetStateGraph("SGmeditator")
    inst:SetBrain(require("brains/meditatorbrain"))

    return inst
end

return Prefab("meditator", fn, assets, prefabs)
```

#### 第五步：`SetStateGraph` 内部做了什么

源码 `scripts/entityscript.lua`：

```lua
function EntityScript:SetStateGraph(name)
    if self.sg ~= nil then
        SGManager:RemoveInstance(self.sg)
    end
    local sg = LoadStateGraph(name)
    if sg ~= nil then
        self.sg = StateGraphInstance(sg, self)
        SGManager:AddInstance(self.sg)
        self.sg:GoToState(sg.defaultstate)
    end
    return self.sg
end
```

**做了 3 件事**：

1. **清理旧 SG**——同一个 entity 切 SG 时不会泄露
2. **`LoadStateGraph(name)`**——从 `scripts/stategraphs/` 加载 SG 类
3. **`GoToState(sg.defaultstate)`**——立即进入默认 state（我们传的 `"idle"`）

> **这就是为什么 prefab fn 里调一次 `SetStateGraph` 就够了**——后续 SG 自动跑、`inst.sg.currentstate` 自动维护。

#### 第六步：modmain 注册 prefab

```lua
-- modmain.lua
PrefabFiles = {
    "meditator",
}

Assets = {
    Asset("ANIM", "anim/meditator.zip"),
}

modimport("scripts/actions/action_meditate.lua")  -- 如果 meditator 也用 MEDITATE action
```

**测试**：控制台 `c_spawn("meditator")` → 应该看到 meditator entity 出现 → 走过去用 wilson 右键它（如果做了 component_action）。

---

### 11.6.7 老手进阶：边界保护——中断、死亡、装备切换、buff 清理

#### 第一步：buff 类 state 的"五道安全网"

任何"持续给玩家加效果"的 state——必须考虑 5 种"突发情况"——否则玩家会卡死、状态泄露、buff 不消失。这五道网是：

1. **被攻击中断**（attacked event）
2. **死亡中断**（death event）
3. **装备切换中断**（equipped / unequipped event）
4. **被冰冻中断**（freeze event）
5. **被踩踏中断**（stunned / knockback event）

漏一道——上线后必收到玩家"打坐永远不结束"或"打坐时穿盔甲，装备图标不见"等 bug 反馈。

#### 第二步：完整的 events 表

```lua
events = {
    EventHandler("attacked", function(inst)
        inst.sg:GoToState("hit")
    end),
    EventHandler("death", function(inst)
        inst.sg:GoToState("death")
    end),
    EventHandler("locomote", function(inst)
        if not inst.sg:HasStateTag("nointerrupt") then
            inst.sg:GoToState("meditate_pst")
        end
    end),
    EventHandler("equip", function(inst, data)
        -- 装备时打断（如玩家中途换斧子）
        if data.eslot == EQUIPSLOTS.HANDS then
            inst.sg:GoToState("meditate_pst")
        end
    end),
    EventHandler("freeze", function(inst)
        inst.sg:GoToState("frozen")
    end),
    EventHandler("knockback", function(inst, data)
        inst.sg:GoToState("knockback")
    end),
    EventHandler("ondrowned", function(inst)
        inst.sg:GoToState("sink_fast")
    end),
}
```

> **观察**：**生产级 SG state 的 events 通常 6~10 条**——这是"防御性 state 设计"的一部分。

#### 第三步：onexit 必须是"清场出口"

**onexit 的核心使命**：**确保 state 期间所做的一切外部副作用——全部还原**。否则下个 state 接手时——副作用泄露——状态机彻底乱。

```lua
onexit = function(inst)
    -- ① 停动画
    inst.AnimState:Stop()

    -- ② 关音效
    inst.SoundEmitter:KillSound("meditate_sound")

    -- ③ 清 OverrideSymbol
    inst.AnimState:ClearOverrideSymbol("legs")

    -- ④ 清自定义 component buff
    if inst.components.tempbuff then
        inst.components.tempbuff:Remove("meditating_regen")
    end

    -- ⑤ 清未消费的 BufferedAction
    if inst.bufferedaction == inst.sg.statemem.action then
        inst:ClearBufferedAction()
    end

    -- ⑥ 解锁可能在 onenter 锁定的玩家控制
    if inst.components.playercontroller and inst.sg.statemem.locked_pc then
        inst.components.playercontroller:Enable(true)
    end
end,
```

> **6 项都很常见**——少一项都可能成线上 bug。**老手习惯写完 onenter → 立刻写 onexit 对称清理**。

#### 第四步：装备切换的特殊处理

**问题**：玩家打坐中——别的玩家给他扔个装备 → inventory 自动 equip → 你的 state 没监听 equip 事件 → 装备成功穿上但**手部 ARM_carry symbol 没显示**——因为 onenter 时 inst 没有装备——根据 11.3.5 没调用 `AnimState:Show("ARM_carry")`——结果手不对。

**修复 1：onenter 监听 inventory 事件**

```lua
onenter = function(inst)
    inst.AnimState:PlayAnimation("research", true)
    if inst.components.inventory:GetEquippedItem(EQUIPSLOTS.HANDS) ~= nil then
        inst.AnimState:Show("ARM_carry")
        inst.AnimState:Hide("ARM_normal")
    else
        inst.AnimState:Hide("ARM_carry")
        inst.AnimState:Show("ARM_normal")
    end
    -- 监听中途变化
    inst:ListenForEvent("equip", inst.sg.statemem.OnEquipChange)
    inst:ListenForEvent("unequip", inst.sg.statemem.OnEquipChange)
    inst.sg.statemem.OnEquipChange = function(inst)
        -- 简单粗暴：触发 equip → 直接进 pst 收尾
        inst.sg:GoToState("meditate_pst")
    end
end,
```

**修复 2（更稳）：直接在 events 加 equip / unequip handler——不必 ListenForEvent**

```lua
events = {
    -- ... 略 ...
    EventHandler("equip", function(inst, data)
        if data.eslot == EQUIPSLOTS.HANDS then
            inst.sg:GoToState("meditate_pst")
        end
    end),
    EventHandler("unequip", function(inst, data)
        if data.eslot == EQUIPSLOTS.HANDS then
            inst.sg:GoToState("meditate_pst")
        end
    end),
}
```

> **为什么 SG events 比 ListenForEvent 好**：events 表是 state 私有——`onexit` 时**自动解绑**——不会泄露。`ListenForEvent` 必须手动 `RemoveEventCallback`——容易漏。

#### 第五步：死亡的"特殊位置"

`death` event 几乎所有 state 都要监听——但**有一条例外**：**`death` state 自己**不能监听 `death` event——会无限递归。

**正确做法**：把 `death` 的处理放到**SG 级 events**：

```lua
-- StateGraph 构造时的 events
local events = {
    EventHandler("death", function(inst)
        if not inst.sg:HasStateTag("dead") then  -- ★ 已经在 death state 时不再触发
            inst.sg:GoToState("death")
        end
    end),
    -- ... 其他 SG 级 events
}

return StateGraph("meditator", states, events, "idle", actionhandlers)
```

> **SG 级 events 是兜底——所有 state 共享**——这避免了"每个 state 都得复制 death handler"。

#### 第六步：buff state 的"心跳保险"

**问题**：万一 timeout 设置失败、ontimeout 抛异常——state 卡 forever——玩家 stuck。

**保险 1：onenter 限制最长持续时间**

```lua
onenter = function(inst)
    inst.sg.statemem.start_time = GetTime()
    -- ... 略 ...
end,

onupdate = function(inst, dt)
    -- ... 回血逻辑 ...

    -- ★ 心跳保险：超过 5s 强制退出（理论 3s + 1s 缓冲）
    if GetTime() - inst.sg.statemem.start_time > 5 then
        inst.sg:GoToState("meditate_pst")
    end
end,
```

**保险 2：不信任 SetTimeout——用 GetTime() 自己算**

`SetTimeout` 在罕见情况下（比如 `Pause`、网络抖动）可能不触发——上面保险 1 是**绕过 SetTimeout 的二次校验**。

> **生产级 buff state 通常都有"双时间源"**——SetTimeout（首选）+ statemem.start_time（兜底）。

---

### 11.6.8 老手进阶：六个常见陷阱

#### 陷阱 1：onenter 里 `if x then GoToState("y") end` 没 return

**症状**：

```lua
onenter = function(inst)
    if inst.components.health:IsDead() then
        inst.sg:GoToState("death")
    end
    inst.AnimState:PlayAnimation("meditate_pre")  -- ★ 死了还播动画
end,
```

**原因**：`GoToState` **不会立即终止函数**——它**注册一个"切 state 的 pending 操作"**——onenter 继续执行——动画照样播——下一帧才切走。

**修复**：

```lua
if inst.components.health:IsDead() then
    inst.sg:GoToState("death")
    return  -- ★ 必须 return
end
```

**11.1.8 已经讲过——但出现频率太高，再次强调**。

---

#### 陷阱 2：onexit 漏关循环音效

**症状**：玩家打坐 → 中断 → 打坐音效**继续播放**——直到下次切到其他 state 重新覆盖。

**原因**：`onexit` 没调 `KillSound`——循环音效一旦启动**永远在播**。

**修复**：

```lua
onenter = function(inst)
    inst.SoundEmitter:PlaySoundWithParams("...", {volume=0.6}, "meditate_sound")
end,

onexit = function(inst)
    inst.SoundEmitter:KillSound("meditate_sound")  -- ★ 必加
end,
```

**关键**：**任何 `PlaySound + 给 name`（持续音效）的 onenter——必须有对应 `KillSound(name)` 的 onexit**——这是 SG 编写**铁律**。

---

#### 陷阱 3：客户端 state 调了 server-only API

**症状**：dedicated 服务器单跑没事——客户端连进来——左下角 mod 报错红字"attempt to index nil (components.health)"。

**原因**：客户端 SG 的 onenter 写了：

```lua
inst.components.health:DoDelta(5)  -- ★ 客户端 health 是 nil
```

——客户端的 components 表大部分**只在 server 上初始化**（health / inventory / hunger 等核心）——客户端 SG 不能直接调。

**修复**：

```lua
-- 客户端 SG 只做"动画 + 预测"——业务逻辑全部在服务端 state
AddStategraphState("wilson_client", State{
    name = "meditate_pre",
    onenter = function(inst)
        inst.AnimState:PlayAnimation("research_pre")
        if inst:GetBufferedAction() ~= nil then
            inst:PerformPreviewBufferedAction()
        end
    end,
    -- ★ 不要在这里调 components.health
})
```

**口诀**：**"客户端 state 写动画——服务端 state 写业务"**——这是 11.5.5 SGwilson_client 的核心设计哲学。

---

#### 陷阱 4：自定义 SG 没引 commonstates

**症状**：mod 写了独立 SG——meditator 死了——尸体不消失、没有死亡动画——只是站着。

**原因**：

```lua
-- ★ 没引
-- require("stategraphs/commonstates")

local states = {
    State{ name = "idle", ... },
    State{ name = "meditate_pre", ... },
    -- ... 自己写的
}
-- ★ 没加
-- CommonStates.AddDeathState(states)

return StateGraph("meditator", states, events, "idle", actionhandlers)
```

——meditator 的 SG 里**根本没有 `death` state**——`GoToState("death")` 找不到 → fallback 到 idle → 玩家看到尸体站着。

**修复**：**任何独立 SG 至少加 4 项 CommonStates**：

```lua
require("stategraphs/commonstates")

CommonStates.AddDeathState(states)
CommonStates.AddSleepStates(states)
CommonStates.AddCombatStates(states, {})
CommonStates.AddFrozenStates(states)
CommonStates.AddWalkStates(states, {})
CommonStates.AddRunStates(states, {})
CommonStates.AddIdle(states)
```

**最少 7 个 CommonStates**——这样 death / sleep / hit / frozen / walk / run / idle 都有保底——其他特色 state 自己写。

---

#### 陷阱 5：自定义 tag 没加 mod 前缀

**症状**：你的 meditate state 加了 `"meditating"` tag——mod A 也叫 `"meditating"`——两个 mod 装一起——业务方 `HasStateTag("meditating")` 判断混乱。

**修复**：

```lua
tags = { "busy", "doing", "mymod_meditating" },  -- ★ mod 前缀
```

11.2.7 学过——这条**永远适用**——不止 meditate——**所有自定义 tag 都加前缀**。

---

#### 陷阱 6：state 名拼写错误——SG 静默切 idle

**症状**：玩家发起 MEDITATE → 一瞬间切到 idle（动画都没播）。

**原因**：

```lua
-- modmain.lua
AddStategraphActionHandler("wilson", ActionHandler(ACTIONS.MEDITATE, "mediate_pre"))  -- ★ "mediate" 漏了一个 t
```

——SG 找不到 state → 走 fallback → idle。**没有任何报错**——SG 默认对"找不到的 state"静默处理（参考 `scripts/stategraph.lua` 的 GoToState 源码）。

**修复**：上线前**人工对照 state name vs ActionHandler 的 state name**——或写一个 mod 自检：

```lua
AddStategraphPostInit("wilson", function(sg)
    assert(sg.states["meditate_pre"], "meditate_pre state not found!")
    assert(sg.states["meditate_loop"], "meditate_loop state not found!")
    assert(sg.states["meditate_pst"], "meditate_pst state not found!")
end)
```

> **断言检查 = 把"运行时静默 bug"变成"启动时报错"**——上线 mod 推荐手段。

---

#### 设计经验三条

**经验 ①：从最小骨架做起，逐层加复杂度**

```
step 1：onenter + AnimState + GoToState idle    （最简）
step 2：+ events.attacked、events.death           （中断）
step 3：+ timeline + statemem                     （行为细节）
step 4：+ wilson_client 镜像                      （联机）
step 5：+ ActionHandler + ComponentAction         （UI）
step 6：+ onexit 全清理 + 边界保护                （上线）
```

每一步**单独验证**——debug 时定位到出错的层。

**经验 ②：永远写 onexit，永远清干净**

> **"任何外部副作用——onenter 起 → onexit 灭"**——这是 SG 编写的"GC 铁律"。

**经验 ③：4 种部署模式都过一遍才上线**

单机 / 主机 + 本地 / 主机 + 远程（>50ms）/ Dedicated + 远程——**都要跑通**。

---

### 11.6.9 小结

#### 一句话总结

**给玩家加新 state 是 mod 的核心能力——把 11.1-11.5 学到的"State 三件套 + StateTag + AnimState + commonstates + SGwilson"全部串起来——3 个 state（pre / loop / pst）+ Action + ActionHandler + 客户端镜像 + onexit 清理 + 边界保护，就能做出生产级动作 mod**。

#### 「打坐回血」实战 6 步落地 checklist

| 步骤 | 内容 | 检查点 |
| --- | --- | --- |
| ① 设计阶段 | 画状态机草图、列字段映射表 | 3 个 state（pre/loop/pst）、7 个字段 |
| ② 骨架版 | 一个 state + 控制台触发 | 动画 + 回血 + 计时切走 |
| ③ Action 接入 | Action + ActionHandler + ComponentAction | 右键菜单出现、能触发 |
| ④ 客户端镜像 | 同步注册到 wilson_client | 远程客户端无卡顿 |
| ⑤ 综合细节 | timeline / 自定义 tag / OverrideSymbol | 动画分阶、tag 不冲突、装备 ARM 正确 |
| ⑥ 边界 + 上线 | events 6+ 条 / onexit 6 项清理 / 4 种部署测试 | 中断 / 死亡 / 装备 / 冰冻 / 沉海全部 OK |

#### 工程文件 5 件套

| 文件 | 作用 |
| --- | --- |
| `modmain.lua` | 注册 Assets、Action、ActionHandler、State |
| `scripts/actions/action_meditate.lua` | Action 定义（fn / rmb / distance） |
| `scripts/sg_meditate.lua`（或 `scripts/stategraphs/SGmeditator.lua`） | State 实现 |
| `scripts/prefabs/meditator.lua`（独立角色时） | prefab fn + `SetStateGraph` |
| `anim/meditate.zip` | 三段式动画资源 |

#### 给 wilson 加 vs 独立 SG 的选择

| 场景 | 路径 |
| --- | --- |
| 给玩家加新动作（任何角色都能用） | `AddStategraphState("wilson", ...)` + `wilson_client` 镜像 |
| 给某 mod 角色专属动作 | `AddStategraphPostInit` 包装 + 用 tag 区分 |
| 完全自定义角色 / 生物 / NPC | 独立 SG 文件 + `inst:SetStateGraph("SGxxx")` |

#### 6 个陷阱排雷顺序

1. `GoToState` 后没 return → onenter 后续逻辑还在跑 → 动画错乱
2. onexit 漏关循环音效 → 音效永远在播
3. 客户端 state 调 server-only API → 客户端报错
4. 独立 SG 没引 commonstates → death/sleep/hit 全失效
5. 自定义 tag 没加 mod 前缀 → mod 间撞名
6. state 名拼写错误 → SG 静默切 idle，无报错

#### 3 条设计经验

- **① 从最小骨架做起**——每一层单独验证
- **② onenter 起 → onexit 灭**——任何外部副作用都对称清理
- **③ 4 种部署模式跑通才上线**——单机 / 主机本地 / 主机远程 / dedicated

#### 整章收尾——你现在掌握了什么

读完整个 11 章——你已经具备了：

1. **理解 SG 系统结构**（11.1）：State / EventHandler / TimeEvent / FrameEvent / SoundFrameEvent 的 7 大字段、生命周期、events 派发、timeline 排序
2. **驾驭 StateTag**（11.2）：3 个核心 API、`SGTagsToEntTags` 双向同步、30+ 内置 tag、复合查询
3. **联动 AnimState**（11.3）：PlayAnimation / PushAnimation、animover / animqueueover、Symbol 替换、动画速率
4. **复用 commonstates**（11.4）：CommonHandlers vs CommonStates、参数化设计、生物 mod 标准模板
5. **修改 SGwilson**（11.5）：330+ state 速查、3 种修改路径、wilson + wilson_client 成对原则
6. **完整动作 mod 实战**（11.6）：从需求拆解到生产上线——9 个子小节落地
