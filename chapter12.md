# 第12章 Brain 与 AI 系统

## 12.1 行为树（BehaviorTree）基础概念

### 本节导读

第 7 章讲完了 **Action 系统**——从玩家输入到动作执行的完整链路。但是有一个角色群体我们一直没关注：**NPC 和怪物**。猪人知道白天在外面闲逛、夜晚跑回家、看到食物去吃、被打了就追过来反击——这些"**智能行为**"不是靠玩家点鼠标驱动的，是靠**代码自动决策**的。

驱动它们做决策的核心框架，就是本章的主角—— **行为树（Behaviour Tree，简称 BT）**。

本节我们从零开始理解行为树：

> **新手**从 12.1.1-12.1.3 起步——在游戏里观察一只猪人的行为、理解"行为树是一棵什么样的树"、认识四种基本状态；**进阶读者**继续看 12.1.4-12.1.6，深入 `behaviourtree.lua` 的所有节点类型、每种节点的 `Visit()` 执行逻辑、BT 类的更新循环和休眠机制；**老手**跳到 12.1.7-12.1.8，逐行拆解 `BrainWrangler` 调度器、理解 Brain 基类的完整生命周期、六个最容易踩的坑。

---

### 12.1.1 快速入门：从一只猪人的一天看行为树是什么

#### 第一步：在游戏里观察一只猪人

打开饥荒联机版，找到一个猪人村落，**什么都不做**，就远远看着一只猪人——

**白天你看到的**：

1. 猪人在猪舍附近闲逛
2. 地上有肉时，猪人走过去把肉吃掉
3. 你走近时，猪人转头看你
4. 你给猪人肉，猪人开始跟着你
5. 你砍树，猪人也跑去帮你砍
6. 蜘蛛来了，猪人冲过去打架

**夜晚你看到的**：

7. 天黑了，猪人跑回家钻进猪舍
8. 如果回不了家，猪人到处找光源
9. 找不到光就到处乱跑（恐慌）

**关键问题**：猪人是怎么"决定"现在该做什么的？

它同时面临这么多可能的行为——闲逛、吃东西、跟随玩家、砍树、打架、回家、找光、恐慌——**谁来决定当前帧执行哪一个？** 用最朴素的 `if-elseif-else` 链来写能行吗？

答案是：**理论上能，但维护起来是噩梦。** 行为之间有优先级、有互斥、有条件切换、有持续执行、有中断恢复……Klei 选择了业界成熟的解决方案—— **行为树**。

#### 第二步：行为树的核心思想——用一棵树来做决策

你可以把行为树想象成一棵**倒过来的树**（根在上面、叶子在下面）：

```
                    PriorityNode (根：按优先级选一个)
                   /          |           \
              恐慌(最高优)   战斗(次高)    日常(最低)
                                         /    |     \
                                      吃东西  跟随  闲逛
```

**每一帧**（准确说是每个 Brain 更新周期），引擎从根节点开始，**自顶向下遍历这棵树**：

1. 先问"恐慌"分支：需要恐慌吗？——不需要→ 跳过
2. 再问"战斗"分支：有敌人要打吗？——没有→ 跳过
3. 最后问"日常"分支：吃东西？跟随？闲逛？——选一个执行

**核心规则**：

- **左边的分支优先级更高**——着火了恐慌 > 有敌人打架 > 日常行为
- **每个节点返回三种状态之一**：`SUCCESS`（成功）、`FAILED`（失败）、`RUNNING`（执行中）
- **一旦某个分支返回 RUNNING，本帧就"锁定"在这个分支上**——下一帧更新时，如果没有更高优先级的事件打断，就继续执行这个分支

这就是行为树的精髓：**用树结构组织决策，用优先级处理冲突，用状态码传递执行结果**。

#### 第三步：用控制台看一只猪人的行为树

在游戏里对着一只猪人，按 `~` 打开控制台：

```lua
local pig = c_findnext("pigman")
print(pig.brain)
```

你会看到类似这样的输出：

```
--brain--
sleep time: 0.50
Priority - RUNNING <SUCCESS> (execute 12, eval in 0.21)
   >Parallel - READY <READY> (...)
   >   >Condition - READY <READY> (...)
   >   >...
   >Priority - RUNNING <SUCCESS> (execute 2, eval in 0.33)
   >   >Wander - RUNNING <RUNNING> (walk for 1.23, ...)
```

**这就是猪人当前行为树的实时快照**——你能看到每个节点的名字、当前状态、上一次结果。`RUNNING` 标记的就是当前正在执行的分支。

> **新手记忆**：**行为树是一棵用来做"现在该做什么"决策的树**。每一帧从根到叶遍历，高优先级分支先检查。每个节点有 SUCCESS / FAILED / RUNNING 三种状态。这个框架让 NPC 的 AI 逻辑变得**模块化、可组合、可调试**。

---

### 12.1.2 快速入门：四种核心状态

#### 第一步：行为树节点的四种状态

在 `scripts/behaviourtree.lua` 的开头（第 4-7 行），定义了四个全局常量：

```4:7:scripts/behaviourtree.lua
SUCCESS = "SUCCESS"
FAILED = "FAILED"
READY = "READY"
RUNNING = "RUNNING"
```

这四个字符串就是行为树中所有节点的**状态码**：

| 状态 | 含义 | 类比 |
|------|------|------|
| `READY` | 就绪——还没被执行过 | 员工刚上班，等待分配任务 |
| `RUNNING` | 执行中——这一帧还没完成 | 员工正在处理一个多帧的长任务（如走路、砍树） |
| `SUCCESS` | 成功——这一帧完成了 | 员工任务完成，向上报告"搞定了" |
| `FAILED` | 失败——这一帧无法执行 | 员工发现做不了（目标不存在、条件不满足），向上报告"做不了" |

#### 第二步：状态是怎么流转的？

每个行为树节点都有一个 `status` 字段。在每一帧的更新循环中：

1. **Visit 阶段**：从根节点开始调用 `Visit()`，每个节点根据自己的逻辑设置 `self.status`
2. **SaveStatus 阶段**：每个节点把当前 status 存到 `lastresult`（用于调试显示）
3. **Step 阶段**：不再 RUNNING 的节点被 `Reset()` 回 READY 状态

这三步对应 `BT:Update()` 的源码（第 20-27 行）：

```20:27:scripts/behaviourtree.lua
function BT:Update()

    self.root:Visit()
    self.root:SaveStatus()
    self.root:Step()

    self.forceupdate = false
end
```

**关键理解**：

- `Visit()` 是"做决策、执行行为"——核心逻辑都在这里
- `SaveStatus()` 是"记录本帧结果"——给下一帧和调试用
- `Step()` 是"清理现场"——非 RUNNING 的节点回到 READY，等待下一帧重新评估

#### 第三步：RUNNING 为什么特殊？

在四种状态里，`RUNNING` 是最关键的。它表示"**这个行为还没做完，下一帧还要继续**"。

举个例子：猪人要走到一个位置吃食物。第一帧开始走路→ 返回 `RUNNING`。第二帧还在走→ 还是 `RUNNING`。第三帧到了、开始吃→ 还是 `RUNNING`。第四帧吃完了→ 返回 `SUCCESS`。

**RUNNING 的作用是"锁定当前行为"**——只要一个分支返回 RUNNING，父节点就会记住"这个分支正在执行"，下一帧优先继续执行它，而不是每帧都从头遍历所有分支。

> **新手记忆**：**READY → Visit 时执行 → 返回 SUCCESS / FAILED / RUNNING。RUNNING 意味着"还在做，别打断我"。** 整个行为树通过这四种状态的流转来控制 NPC 的行为。

---

### 12.1.3 快速入门：BehaviourNode 基类——所有节点的"祖先"

#### 第一步：定位源码

`BehaviourNode` 是所有行为树节点的基类，定义在 `scripts/behaviourtree.lua` 第 54 行：

```54:70:scripts/behaviourtree.lua
BehaviourNode = Class(function (self, name, children)
    self.name = name or ""
    self.children = children
    self.status = READY
    self.lastresult = READY
    self.nextupdatetick = 0

    --jcheng: this is for imgui to have an id to use
    self.id = NODE_COUNT
    NODE_COUNT = NODE_COUNT + 1

    if children then
        for i,k in pairs(children) do
            k.parent = self
        end
    end
end)
```

**核心字段**：

| 字段 | 类型 | 含义 |
|------|------|------|
| `name` | string | 节点名称，用于调试显示 |
| `children` | table/nil | 子节点数组。叶子节点没有 children |
| `status` | string | 当前状态（READY/RUNNING/SUCCESS/FAILED） |
| `lastresult` | string | 上一帧的结果（用于调试和 PriorityNode 决策） |
| `parent` | BehaviourNode | 父节点引用（在构造函数里自动设置） |
| `id` | number | 全局唯一 ID（imgui 调试用） |

#### 第二步：六个核心方法

| 方法 | 作用 |
|------|------|
| `Visit()` | **最核心的方法**。每帧被调用，设置 self.status。基类默认实现直接设为 FAILED |
| `SaveStatus()` | 把 status 存到 lastresult，递归子节点 |
| `Step()` | 非 RUNNING 的节点调用 Reset()；RUNNING 的递归子节点 |
| `Reset()` | 把 status 设回 READY，递归子节点 |
| `Stop()` | 停止时清理资源（调用 OnStop 回调），递归子节点 |
| `Sleep(t)` | 设置 nextupdatetime，告诉调度器"t 秒后再叫我" |

#### 第三步：树结构的递归特性

行为树的每一个节点——无论是"组合节点"（有 children）还是"叶子节点"（没有 children）——都继承自 `BehaviourNode`。这意味着**所有方法都是递归的**：

- 调用根节点的 `Visit()`，根节点会调用子节点的 `Visit()`，子节点再调用孙节点的 `Visit()`……
- 调用根节点的 `Reset()`，所有后代节点全部被 Reset

这就是行为树"**一棵树管全局**"的基础。

> **新手记忆**：`BehaviourNode` 是所有行为树节点的"祖先类"。它定义了 name、children、status 三个核心字段和 Visit / Reset / Step / Sleep 四个核心方法。**理解了 BehaviourNode，就理解了行为树的骨架。**

---

### 12.1.4 进阶：组合节点（Composite Nodes）—— 行为树的"骨架"

组合节点有 children（子节点数组），它们的职责是**控制子节点的执行顺序和逻辑**。饥荒实现了 5 种组合节点：

#### 第一种：`SequenceNode`——"全部成功才算成功"

源码位于 `scripts/behaviourtree.lua` 第 304-336 行。

**执行逻辑**：从左到右依次 `Visit()` 每个子节点——

- 如果某个子节点返回 `FAILED`，SequenceNode 立即返回 `FAILED`（短路）
- 如果某个子节点返回 `RUNNING`，SequenceNode 返回 `RUNNING`，下一帧从该子节点继续
- 全部子节点 `SUCCESS`，SequenceNode 返回 `SUCCESS`

**类比**：去商店买东西的步骤——① 出门 → ② 走到商店 → ③ 选东西 → ④ 付钱 → ⑤ 回家。任何一步失败整个流程就失败。

**实际使用举例**——小鸟的"饥饿时吃东西"行为（来自 `smallbirdbrain.lua`）：

```93:103:scripts/brains/smallbirdbrain.lua
        SequenceNode{
            ConditionNode(function() return IsStarving(self.inst) and CanSeeFood(self.inst) end, "SeesFoodToEat"),
            ParallelNodeAny {
                WaitNode(math.random()*.5),
                PriorityNode {
                    StandStill(self.inst, ShouldStandStill),
                    Follow(self.inst, function() return GetLeader(self.inst) end, MIN_FOLLOW_DIST, TARGET_FOLLOW_DIST, MAX_FOLLOW_DIST),
                },
            },
            DoAction(self.inst, function() return FindFoodAction(self.inst) end),
        },
```

先检查条件（饥饿且看到食物），再等一小会儿，最后执行吃东西的动作。只有前一步成功，才会执行下一步。

#### 第二种：`SelectorNode`——"有一个成功就算成功"

源码位于第 340-375 行。

**执行逻辑**：从左到右依次 `Visit()` 每个子节点——

- 如果某个子节点返回 `SUCCESS`，SelectorNode 立即返回 `SUCCESS`
- 如果某个子节点返回 `RUNNING`，SelectorNode 返回 `RUNNING`
- 全部 `FAILED`，SelectorNode 返回 `FAILED`

**类比**：找饭吃——① 看看冰箱有没有剩饭 → ② 看看外卖有没有到 → ③ 自己做。找到一个能吃的就行。

> 注意：饥荒源码中虽然定义了 `SelectorNode`，但在实际的 Brain 文件中**几乎不直接使用**，因为 `PriorityNode` 功能更强大（下面会讲）。

#### 第三种：`PriorityNode`——"带定时重评的选择器"（最核心！）

源码位于第 533-638 行。这是饥荒行为树中**使用最频繁**的节点，几乎每个 Brain 的根节点都是 PriorityNode。

**与 SelectorNode 的区别**：SelectorNode 在子节点 RUNNING 时，下一帧直接继续那个节点。而 PriorityNode 有一个 `period` 参数（默认 1 秒），**每隔 period 秒会从头重新评估所有子节点**。这意味着——

- 猪人正在闲逛（低优先级行为，RUNNING 中）
- 每 0.5 秒，PriorityNode 会重新检查：有没有更高优先级的事情要做？
- 如果突然来了一只蜘蛛，"战斗"分支返回 RUNNING → **中断闲逛，切换到战斗**

**构造函数**：

```lua
PriorityNode(children, period, noscatter)
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `children` | table | 子节点数组，按优先级从高到低排列 |
| `period` | number | 重评估间隔（秒），默认 1。值越小响应越灵敏，但 CPU 开销越大 |
| `noscatter` | boolean | 是否取消首次执行的随机偏移。默认 false，即首次执行时随机延迟一段时间，避免所有 NPC 同一帧扎堆更新 |

**实际使用举例**——查看猪人 Brain 的根节点（来自 `pigbrain.lua`）：

```382:418:scripts/brains/pigbrain.lua
    local root =
        PriorityNode(
        {
            BrainCommon.PanicWhenScared(self.inst, .25, "PIG_TALK_PANICBOSS"),
            -- ...恐慌、着火等最高优先级行为...
            ChattyNode(self.inst, "PIG_TALK_FIGHT",
                WhileNode(..., ChaseAndAttack(...))),
            -- ...战斗行为...
            in_contest,
            watch_game,
            day,     -- 白天行为（子树）
            night,   -- 夜晚行为（子树）
        }, .5)

    self.bt = BT(self.inst, root)
```

根节点 PriorityNode 的 `period` 设为 0.5 秒——每半秒重新评估一次，确保高优先级行为（恐慌、战斗）能及时抢占低优先级行为（日常活动）。

#### 第四种：`ParallelNode`——"同时执行所有子节点"

源码位于第 644-695 行。

**执行逻辑**：每帧 `Visit()` 所有子节点——

- 任何一个子节点 `FAILED`，ParallelNode 立即 `FAILED`
- 全部 `SUCCESS`，ParallelNode `SUCCESS`
- 否则 `RUNNING`

**变体 `ParallelNodeAny`**：只要有一个子节点 `SUCCESS`，整体就 `SUCCESS`。

**最重要的用途**：配合 `ConditionNode` 实现 **WhileNode**。

#### 第五种：`RandomNode`——"随机选一个执行"

源码位于第 484-529 行。

**执行逻辑**：随机选择一个子节点 Visit()。如果 FAILED 就尝试下一个（环形），全部 FAILED 则 RandomNode FAILED。

---

### 12.1.5 进阶：叶子节点（Leaf Nodes）—— 行为树的"执行器"

叶子节点没有 children，它们负责**做具体的事情或检查条件**。饥荒内置了以下叶子节点：

#### `ConditionNode`——条件检查

```202:213:scripts/behaviourtree.lua
ConditionNode = Class(BehaviourNode, function(self, fn, name)
    BehaviourNode._ctor(self, name or "Condition")
    self.fn = fn
end)

function ConditionNode:Visit()
    if self.fn() then
        self.status = SUCCESS
    else
        self.status = FAILED
    end
end
```

**逻辑极其简单**：调用 `fn()`，返回 true 则 SUCCESS，否则 FAILED。**永远不会返回 RUNNING**——条件检查是瞬时的。

**使用场景**：作为 SequenceNode 的第一个子节点，充当"门卫"：

```lua
SequenceNode{
    ConditionNode(function() return IsHungry(inst) end, "IsHungry"),
    DoAction(inst, FindFoodAction),
}
```

只有 IsHungry 返回 true，才会继续执行 DoAction。

#### `ConditionWaitNode`——等待条件满足

与 ConditionNode 类似，但条件不满足时返回 **RUNNING** 而非 FAILED。意思是"条件还没满足，我等着"。

#### `ActionNode`——立即执行动作

```259:267:scripts/behaviourtree.lua
ActionNode = Class(BehaviourNode, function(self, action, name)
    BehaviourNode._ctor(self, name or "ActionNode")
    self.action = action
end)

function ActionNode:Visit()
    self.action()
    self.status = SUCCESS
end
```

调用 `action()` 函数然后立即返回 SUCCESS。注意这个 ActionNode 和饥荒的 Action 系统无关——它只是一个"执行一个函数"的通用节点。

#### `WaitNode`——等待一段时间

```273:299:scripts/behaviourtree.lua
WaitNode = Class(BehaviourNode, function(self, time)
    BehaviourNode._ctor(self, "Wait")
    self.wait_time = time
end)

function WaitNode:Visit()
    local current_time = GetTime()

    if self.status ~= RUNNING then
        self.wake_time = current_time + FunctionOrValue(self.wait_time)
        self.status = RUNNING
    end

    if self.status == RUNNING then
        if current_time >= self.wake_time then
            self.status = SUCCESS
        else
            self:Sleep(current_time - self.wake_time)
        end
    end

end
```

第一次 Visit 时，记录唤醒时间（当前时间 + 等待时长），状态设为 RUNNING。后续 Visit 时检查是否到了唤醒时间，到了就 SUCCESS。`wait_time` 可以是数字或函数。

---

### 12.1.6 进阶：装饰器与便捷函数——WhileNode、IfNode、EventNode

#### `DecoratorNode`——装饰器基类

装饰器只有**一个子节点**，它的作用是**修改子节点的行为或结果**。

饥荒提供了三个装饰器：

| 装饰器 | 作用 |
|--------|------|
| `NotDecorator` | 反转子节点结果：SUCCESS→FAILED，FAILED→SUCCESS |
| `FailIfRunningDecorator` | 子节点 RUNNING 时返回 FAILED（把持续行为变成瞬时失败） |
| `FailIfSuccessDecorator` | 子节点 SUCCESS 时返回 FAILED（让 PriorityNode 跳到下一个分支） |

#### `WhileNode`——"只要条件满足就持续执行"

```771:777:scripts/behaviourtree.lua
function WhileNode(cond, name, node)
    return ParallelNode
        {
            ConditionNode(cond, name),
            node
        }
end
```

**WhileNode 不是一个类，是一个工厂函数**！它返回的是一个 `ParallelNode`，包含两个子节点：

1. `ConditionNode(cond)` —— 条件检查
2. `node` —— 实际要执行的行为

由于 ParallelNode **每帧都重新 Visit 所有子节点**，而 ConditionNode 会在 Step 阶段被 Reset 回 READY（因为它每帧都 SUCCESS），所以**条件会被每帧重新检查**。一旦条件变为 false → ConditionNode FAILED → ParallelNode 立即 FAILED → 行为被中断。

**这是饥荒行为树中最常见的模式之一**——几乎每个 Brain 都大量使用 WhileNode：

```333:356:scripts/brains/pigbrain.lua
    local day = WhileNode( function() return TheWorld.state.isday end, "IsDay",
        PriorityNode{
            ChattyNode(self.inst, "PIG_TALK_FIND_MEAT",
                DoAction(self.inst, FindFoodAction )),
            -- ...其他白天行为...
            Wander(self.inst, GetNoLeaderHomePos, MAX_WANDER_DIST)
        }, .5)
```

"只要是白天，就执行白天行为子树"——天一黑，WhileNode 条件变 false，整个子树立即中断。

#### `IfNode`——"条件满足时执行一次"

```783:789:scripts/behaviourtree.lua
function IfNode(cond, name, node)
    return SequenceNode
        {
            ConditionNode(cond, name),
            node
        }
end
```

与 WhileNode 的区别：IfNode 用的是 SequenceNode，条件只在 **进入时检查一次**。一旦子行为开始 RUNNING，即使条件变 false 也不会中断——要等行为自然结束（SUCCESS 或 FAILED）后才会重新检查。

#### `EventNode`——"事件驱动的行为树节点"

```702:765:scripts/behaviourtree.lua
EventNode = Class(BehaviourNode, function(self, inst, event, child, priority)
    BehaviourNode._ctor(self, "Event("..event..")", {child})
    self.inst = inst
    self.event = event
    self.priority = priority or 0

    self.eventfn = function(inst, data) self:OnEvent(data) end
    self.inst:ListenForEvent(self.event, self.eventfn)
end)
```

EventNode 在构造时就 `ListenForEvent`，监听指定事件。当事件触发时：

1. 设置 `self.triggered = true`
2. 调用 `self.inst.brain:ForceUpdate()` 强制唤醒 Brain
3. 清空所有祖先 PriorityNode 的 `lasttime`，让它们立即重评估

这让行为树可以**对游戏事件做出即时响应**，而不必等待下一个 period 周期。

#### `LatchNode`——"冷却锁"

LatchNode 有一个 `latchduration` 参数。每次执行后会进入冷却期，冷却期间直接返回 FAILED。用于防止行为被反复触发。

---

### 12.1.7 老手进阶：BT 类与 BrainWrangler 调度器——行为树的"引擎"

#### 第一步：BT 类——行为树的运行时容器

```12:47:scripts/behaviourtree.lua
BT = Class(function(self, inst, root)
    self.inst = inst
    self.root = root
end)

function BT:ForceUpdate()
    self.forceupdate = true
end
function BT:Update()

    self.root:Visit()
    self.root:SaveStatus()
    self.root:Step()

    self.forceupdate = false
end

function BT:Reset()
    self.root:Reset()
end

function BT:Stop()
    self.root:Stop()
end

function BT:GetSleepTime()
    if self.forceupdate then
        return 0
    end

    return self.root:GetTreeSleepTime()
end
```

BT 类非常简单：持有实体引用和根节点，提供 Update / Reset / Stop / ForceUpdate / GetSleepTime 方法。

**关键方法 `GetSleepTime()`**：计算整棵树中 RUNNING 节点的最小休眠时间。如果有 ForceUpdate 标记，返回 0（立即更新）。

#### 第二步：Brain 基类——Brain 文件的基础设施

每个 Brain 文件（如 `PigBrain`、`SpiderBrain`）都继承自 `Brain` 基类（定义在 `scripts/brain.lua` 第 159-285 行）：

```159:168:scripts/brain.lua
Brain = Class(function(self)
    self.inst = nil
    self.currentbehaviour = nil
    self.behaviourqueue = {}
    self.events = {}
    self.thinkperiod = nil
    self.lastthinktime = nil
    self.paused = false
    self.stopped = true
end)
```

**生命周期**：

1. **`_Start_Internal()`**：调用子类的 `OnStart()`（在这里构建行为树），然后注册到 BrainManager，最后执行 Mod 的 postinitfns
2. **`OnUpdate()`**：每帧被调度器调用，先调用子类的 `DoUpdate()`（如果有），再调用 `self.bt:Update()` 更新行为树
3. **`_Stop_Internal()`**：调用子类的 `OnStop()`，停止行为树，从 BrainManager 中移除
4. **`Pause()` / `Resume()`**：暂停/恢复 Brain，暂停时从 BrainManager 移除、恢复时重新添加

#### 第三步：BrainWrangler（BrainManager）——全局调度器

`BrainWrangler` 是所有 Brain 实例的"总管"，全局唯一实例为 `BrainManager`。

它维护三个"池子"：

| 池子 | 含义 |
|------|------|
| `updaters` | 需要被更新的 Brain（每帧被调用 OnUpdate） |
| `tickwaiters[tick]` | 在特定 tick 才需要醒来的 Brain（休眠中） |
| `hibernaters` | 冬眠中的 Brain（所有行为都不活跃） |

**调度循环**（`BrainWrangler:Update(current_tick)`）：

1. 把本 tick 到期的 tickwaiters 移到 updaters
2. 遍历 updaters，对每个 Brain 调用 `OnUpdate()`
3. 根据 `GetSleepTime()` 决定：
   - 返回值 > 一帧时间 → 移到 tickwaiters（休眠到那个 tick）
   - 返回值 ≤ 一帧时间 → 留在 updaters（下帧继续更新）
   - 返回 nil → 移到 hibernaters（冬眠，直到被 ForceUpdate 唤醒）

**这就是行为树的"省电模式"**：不是每帧都更新所有 NPC 的 AI，而是根据休眠时间按需唤醒。当一个 NPC 在 Wander 状态等待 3 秒时，它的 Brain 会被放入 tickwaiters，3 秒后才醒来——大幅降低 CPU 开销。

#### 第四步：用控制台验证调度状态

```lua
-- 看当前有多少 Brain 在活跃更新
local count = 0
for _ in pairs(BrainManager.updaters) do count = count + 1 end
print("Active brains:", count)

-- 看有多少在冬眠
count = 0
for _ in pairs(BrainManager.hibernaters) do count = count + 1 end
print("Hibernating brains:", count)
```

---

### 12.1.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：PriorityNode 的 period 设得太小

```lua
-- 错误：period 设为 0，每帧都从头评估
local root = PriorityNode({...}, 0)
```

**问题**：period=0 意味着每帧都重新评估所有子节点。如果子节点数量多或条件检查复杂，会严重影响性能。

**正确做法**：根据 NPC 的响应需求选择合适的 period。普通 NPC 用 0.5-1 秒，Boss 战时可以用 0.25 秒。

```lua
-- 正确：0.5 秒评估一次，够灵敏也不浪费
local root = PriorityNode({...}, 0.5)
```

#### 陷阱 2：WhileNode 和 IfNode 混淆

```lua
-- 预期：只要有敌人就追击
-- 错误：用了 IfNode，进入追击后即使敌人消失也不会中断
IfNode(function() return HasEnemy(inst) end, "HasEnemy",
    ChaseAndAttack(inst))
```

**问题**：IfNode 是 SequenceNode 的糖衣，条件只在进入时检查一次。应该用 WhileNode。

```lua
-- 正确：WhileNode 每帧检查条件，敌人消失时立即中断追击
WhileNode(function() return HasEnemy(inst) end, "HasEnemy",
    ChaseAndAttack(inst))
```

**选择规则**：
- 条件可能中途变化、需要即时中断 → 用 `WhileNode`
- 条件只在"开始时"有意义、一旦开始就要做完 → 用 `IfNode`

#### 陷阱 3：忘记在 OnStart 里设置 self.bt

```lua
function MyBrain:OnStart()
    local root = PriorityNode({...}, 1)
    -- 忘了这一行！
    -- self.bt = BT(self.inst, root)
end
```

**后果**：Brain 启动了但没有行为树，`Brain:OnUpdate()` 里 `self.bt` 是 nil，NPC 变成呆子。

**正确做法**：`OnStart()` 的最后一行必须是 `self.bt = BT(self.inst, root)`。

#### 陷阱 4：叶子节点的 Visit() 没正确设置 status

```lua
-- 错误：自定义叶子节点忘了设 status
MyNode = Class(BehaviourNode, function(self, inst)
    BehaviourNode._ctor(self, "MyNode")
    self.inst = inst
end)

function MyNode:Visit()
    -- 做了一些事情...
    DoSomething(self.inst)
    -- 但忘了设 self.status！
end
```

**后果**：`BehaviourNode:Visit()` 基类实现会把 status 设为 FAILED。如果你覆写了 Visit() 但忘记设 status，行为会出现不可预测的结果。

**正确做法**：覆写的 `Visit()` 必须在所有代码路径上都设置 `self.status`。

#### 陷阱 5：在 Brain 里直接访问其他实体的 components（多人游戏陷阱）

```lua
-- 潜在问题：直接访问 target 的 components
local function ShouldChase(inst)
    local target = inst.components.combat.target
    return target and target.components.health:GetPercent() < 0.5
end
```

**问题**：在联机版中，Brain 运行在服务端。如果 target 是另一个玩家，`target.components.health` 可能为 nil（客户端实体没有完整 components）。但由于 Brain 本身就只在服务端运行，这个问题通常不存在——**除非你的代码跑在了客户端**。

**安全做法**：加 nil 检查，或确认 Brain 只在服务端执行。

#### 陷阱 6：EventNode 事件监听没有被正确清理

EventNode 在构造时 `ListenForEvent`，在 `OnStop` 时 `RemoveEventCallback`。如果你手动 Stop 了行为树但又重新构建了一棵新树（比如在 OnStart 里重新创建），旧树的 EventNode 可能没有正确 Stop，导致**事件监听泄漏**。

**正确做法**：确保旧的行为树被完整 Stop 后再创建新树。Brain 基类的 `_Stop_Internal()` 会调用 `self.bt:Stop()`，递归停止所有节点。

#### 设计经验三条

**经验一：行为优先级的"金字塔"原则**

在 PriorityNode 中，子节点从左到右优先级递减。推荐的"金字塔"是：

```
最高优先级：生存（着火恐慌、被困逃脱）
     ↓
次高优先级：战斗（追击攻击、被打反击）
     ↓
中等优先级：社交（跟随、交易、面对玩家）
     ↓
较低优先级：需求（吃东西、回家）
     ↓
最低优先级：日常（闲逛、发呆）
```

所有原版 Brain 都遵循这个模式。以 ChesterBrain 为例——Chester 没有战斗能力，所以只有恐慌 > 跟随 > 面向主人 > 闲逛。

```29:40:scripts/brains/chesterbrain.lua
function ChesterBrain:OnStart()
    local root =
    PriorityNode({
        BrainCommon.PanicTrigger(self.inst),
        BrainCommon.ElectricFencePanicTrigger(self.inst),
        Follow(self.inst, function() return GetLeader(self.inst) end, MIN_FOLLOW_DIST, TARGET_FOLLOW_DIST, MAX_FOLLOW_DIST),
        FaceEntity(self.inst, GetFaceTargetFn, KeepFaceTargetFn),
        Wander(self.inst, function() return self.inst.components.knownlocations:GetLocation("home") end, MAX_WANDER_DIST),

    }, .25)
    self.bt = BT(self.inst, root)
end
```

**经验二：善用 behaviours 模块**

饥荒在 `scripts/behaviours/` 下提供了 29 个预制行为节点，覆盖了绝大多数常见场景：

| 行为模块 | 功能 |
|----------|------|
| `ChaseAndAttack` | 追击并攻击目标 |
| `RunAway` | 从威胁逃跑 |
| `Follow` | 跟随目标（三段距离：最小/目标/最大） |
| `Wander` | 在区域内闲逛 |
| `DoAction` | 执行一个 BufferedAction（万能接口） |
| `Panic` | 恐慌逃跑（随机方向） |
| `FaceEntity` | 面向目标 |
| `Leash` | 栓绳——离家太远时拉回来 |
| `FindLight` | 找光源（夜晚用） |
| `StandStill` | 站着不动 |
| `AttackWall` | 攻击挡路的墙 |

**写 Mod 时应优先使用这些现成模块**，而不是从零写叶子节点。`DoAction` 尤其万能——只要你能构造一个 `BufferedAction`，就能让 NPC 执行任何动作。

**经验三：行为树 vs 状态机**

有些开发者会困惑：饥荒既有行为树（Brain），又有状态机（StateGraph），它们的关系是什么？

- **行为树**负责"**决定做什么**"——当前应该追击、逃跑还是闲逛？
- **状态机**负责"**怎么做**"——追击时播放什么动画、动画到哪一帧执行攻击？

它们是**互补而非替代**的关系。典型流程是：

```
行为树决策 → 推送 BufferedAction → StateGraph 切换到对应 State → 播放动画 → 执行动作
```

行为树是**策略层**，StateGraph 是**表现层**。

---

### 12.1.9 小结

| 概念 | 一句话总结 |
|------|-----------|
| 行为树 | 一棵用来做"现在该做什么"决策的树，自顶向下遍历 |
| 四种状态 | READY / RUNNING / SUCCESS / FAILED，状态码驱动决策流 |
| BehaviourNode | 所有节点的基类，定义 Visit / Reset / Step / Sleep |
| PriorityNode | 最常用的组合节点，带定时重评估的优先级选择器 |
| SequenceNode | "全部成功才成功"，用于多步顺序流程 |
| ParallelNode | "同时执行"，常与 ConditionNode 配合实现 WhileNode |
| WhileNode | 便捷函数，"条件满足时持续执行"（每帧检查条件） |
| IfNode | 便捷函数，"条件满足时执行一次"（进入后不再检查条件） |
| EventNode | 事件驱动节点，监听事件并即时触发行为切换 |
| BT 类 | 行为树运行时容器，Update 驱动 Visit→SaveStatus→Step 循环 |
| Brain 基类 | Brain 文件的基础设施，OnStart 构建树、OnUpdate 驱动更新 |
| BrainWrangler | 全局调度器，管理所有 Brain 的更新/休眠/冬眠 |
| behaviours 模块 | 29 个预制叶子行为节点，覆盖追击/逃跑/跟随/闲逛等场景 |

**下一节**（12.2）我们将详细讲解每种 BrainNode 的实现细节、参数和使用场景。

## 12.2 BrainNode 类型：SequenceNode、PriorityNode、WhileNode、ConditionNode 等

### 本节导读

12.1 我们了解了行为树的整体概念和四种状态。本节深入 `scripts/behaviourtree.lua` 中**每一个节点类型**的实现细节——它的构造参数、Visit() 逻辑、使用场景、实际代码示例。同时覆盖 `scripts/behaviours/` 下的 **29 个预制行为节点**。

> **新手**从 12.2.1-12.2.3 起步——详细理解三种使用频率最高的节点 PriorityNode、SequenceNode、WhileNode/IfNode，每种配有游戏内的实际例子；**进阶读者**继续看 12.2.4-12.2.6，覆盖 ParallelNode、EventNode、装饰器节点、LoopNode、LatchNode 等"特种节点"；**老手**跳到 12.2.7-12.2.8，逐个拆解 29 个 behaviours 模块的参数和实现，六个常见陷阱与设计经验。

---

### 12.2.1 快速入门：PriorityNode——行为树的"主心骨"

#### 第一步：为什么先讲 PriorityNode

在饥荒的 255+ 个 Brain 文件中，**几乎每个 Brain 的根节点都是 PriorityNode**。它是行为树的"骨架中的骨架"——理解了 PriorityNode，就理解了 NPC 如何做出"先做什么、后做什么"的决策。

#### 第二步：构造函数

```533:539:scripts/behaviourtree.lua
PriorityNode = Class(BehaviourNode, function(self, children, period, noscatter)
    BehaviourNode._ctor(self, "Priority", children)
    self.period = period or 1
    if not noscatter then
        self.lasttime = self.period * 0.5 + (self.period * math.random())
    end
end)
```

| 参数 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `children` | table | 必填 | 子节点数组，按优先级**从高到低**排列（左边优先） |
| `period` | number | 1 | 重评估间隔（秒）。每隔 period 秒从头检查是否有更高优先级的行为需要抢占 |
| `noscatter` | boolean | false | 是否禁用首次更新的随机偏移。默认 false，即首次更新会随机延迟 `period*0.5 ~ period*1.5` 秒，防止大量 NPC 同帧更新导致卡顿 |

#### 第三步：Visit() 逻辑详解

PriorityNode 的 Visit() 有两条路径（第 582-638 行）：

**路径 A：重评估（do_eval = true）**

当前时间超过 `lasttime + period` 时触发。从第一个子节点开始依次 Visit：
1. 如果子节点 SUCCESS 或 RUNNING → 选中该子节点，记录 `self.idx`
2. 已经找到选中节点后，后续子节点全部 Reset
3. 如果没有任何子节点成功 → 整体 FAILED

**路径 B：继续当前（do_eval = false）**

未到重评估时间。直接 Visit 当前 `self.idx` 指向的子节点：
- 子节点 RUNNING → 整体继续 RUNNING
- 子节点 SUCCESS 或 FAILED → 清空 lasttime，下一帧触发重评估

**关键理解**：PriorityNode 不是每帧都从头遍历所有子节点的！它有一个"冷却期"（period），在冷却期内只关注当前正在 RUNNING 的子节点。只有到了重评估时刻，才会从头检查高优先级分支是否需要抢占。

#### 第四步：实例分析——蜜蜂的 PriorityNode

```57:84:scripts/brains/beebrain.lua
function BeeBrain:OnStart()
    local root = PriorityNode(
    {
        BrainCommon.PanicTrigger(self.inst),                              -- 优先级 1：恐慌
        BrainCommon.ElectricFencePanicTrigger(self.inst),                 -- 优先级 2：电击

        WhileNode(function() return ... end, "AttackMomentarily",         -- 优先级 3：攻击
            ChaseAndAttack(self.inst, ...)),
        WhileNode(function() return ... end, "Dodge",                     -- 优先级 4：躲闪
            RunAway(self.inst, ...)),

        WhileNode(function() return IsHomeOnFire(...) end, ..., Panic()), -- 优先级 5：家着火
        IfNode(function() return not TheWorld.state.iscaveday ... end,    -- 优先级 6：夜晚回家
            DoAction(self.inst, function() return beecommon.GoHomeAction(self.inst) end, ...)),
        IfNode(function() return ...HasCollectedEnough() end, ...),       -- 优先级 7：花粉满了回家
        IfNode(function() return TheWorld.state.iswinter end, ...),       -- 优先级 8：冬天回家

        IfNode(function() return FindBeeBeacon(self) ~= nil end, ...),    -- 优先级 9：蜂引
        FindFlower(self.inst),                                            -- 优先级 10：采花
        Wander(self.inst, ...),                                           -- 优先级 11：闲逛
    }, 1)

    self.bt = BT(self.inst, root)
end
```

蜜蜂的根 PriorityNode 有 11 个子节点，period 为 1 秒。优先级从高到低清晰可见：恐慌 > 攻击/躲闪 > 回家 > 采花 > 闲逛。每 1 秒重评估一次，确保高优先级行为能及时抢占。

#### 第五步：period 参数的选择策略

| 场景 | 推荐 period | 理由 |
|------|-------------|------|
| Boss / 高威胁怪物 | 0.1-0.25 | 需要极快的反应速度 |
| 普通战斗型 NPC | 0.5 | 响应灵敏，性能合理 |
| 和平型 NPC（猪人日常） | 0.5-1.0 | 不需要太快反应 |
| 宠物/跟随者 | 0.25 | 需要紧跟主人的移动 |
| 背景生物（蝴蝶、兔子） | 1.0 | 行为简单，不需要频繁决策 |

---

### 12.2.2 快速入门：SequenceNode——"先后顺序做事"

#### 第一步：核心逻辑

```304:336:scripts/behaviourtree.lua
SequenceNode = Class(BehaviourNode, function(self, children)
    BehaviourNode._ctor(self, "Sequence", children)
    self.idx = 1
end)

function SequenceNode:Visit()
    if self.status ~= RUNNING then
        self.idx = 1
    end

    while self.idx <= #self.children do
        local child = self.children[self.idx]
        child:Visit()
        if child.status == RUNNING or child.status == FAILED then
            self.status = child.status
            return
        end

        self.idx = self.idx + 1
    end

    self.status = SUCCESS
end
```

**一句话总结**：从左到右依次执行子节点——任何一个 FAILED 整体就 FAILED，任何一个 RUNNING 就暂停在那里等下一帧，全部 SUCCESS 才 SUCCESS。

**参数极简**：只有 `children`，没有 period。

#### 第二步：典型使用场景——"先检查条件，再执行动作"

SequenceNode 最经典的用法是和 ConditionNode 搭配：

```lua
SequenceNode{
    ConditionNode(function() return inst.components.hunger:GetPercent() < 0.3 end, "IsHungry"),
    DoAction(inst, FindFoodAction, "EatFood"),
}
```

意思是：先检查是否饥饿，如果是才去找东西吃。ConditionNode 返回 FAILED 时，整个 Sequence 立即 FAILED，DoAction 不会被执行。

#### 第三步：SequenceNode 的"记忆"特性

注意 SequenceNode 有 `self.idx` 字段。如果第二个子节点返回 RUNNING，下一帧 Visit 时会**从 idx=2 继续**，不会重新检查第一个子节点。

**这与 IfNode 的行为一致**——IfNode 就是 SequenceNode 的语法糖：

```783:789:scripts/behaviourtree.lua
function IfNode(cond, name, node)
    return SequenceNode
        {
            ConditionNode(cond, name),
            node
        }
end
```

一旦条件通过、行为开始 RUNNING，即使条件后来变了，行为也不会被中断——要等行为自然结束。

#### 第四步：实际案例——小鸟的多步进食行为

```93:103:scripts/brains/smallbirdbrain.lua
        SequenceNode{
            ConditionNode(function() return IsStarving(self.inst) and CanSeeFood(self.inst) end, "SeesFoodToEat"),
            ParallelNodeAny {
                WaitNode(math.random()*.5),
                PriorityNode {
                    StandStill(self.inst, ShouldStandStill),
                    Follow(self.inst, function() return GetLeader(self.inst) end, MIN_FOLLOW_DIST, TARGET_FOLLOW_DIST, MAX_FOLLOW_DIST),
                },
            },
            DoAction(self.inst, function() return FindFoodAction(self.inst) end),
        },
```

三步流程：① 检查条件（饥饿且看到食物）→ ② 等一小会儿同时跟随主人 → ③ 执行吃东西动作。任何一步失败都中断整个流程。

---

### 12.2.3 快速入门：WhileNode 与 IfNode——条件控制的"两兄弟"

#### 第一步：WhileNode——"持续检查条件"

```771:777:scripts/behaviourtree.lua
function WhileNode(cond, name, node)
    return ParallelNode
        {
            ConditionNode(cond, name),
            node
        }
end
```

WhileNode 是一个**工厂函数**，返回 ParallelNode{ConditionNode, node}。

**核心机制**：ParallelNode 每帧 Visit 所有子节点。ConditionNode 在 Step 阶段会被 Reset 回 READY（因为它不是 RUNNING），所以**每帧都重新检查条件**。一旦条件变 false：

1. ConditionNode FAILED
2. ParallelNode 规则：任何子节点 FAILED → 整体 FAILED
3. 行为被立即中断

**使用场景**：条件可能在行为执行过程中变化，需要及时中断。

最常见的例子——"只要是白天就执行白天行为"：

```333:334:scripts/brains/pigbrain.lua
    local day = WhileNode( function() return TheWorld.state.isday end, "IsDay",
        PriorityNode{
```

天一黑，WhileNode 条件变 false，整个白天子树立即中断。

#### 第二步：IfNode——"只检查一次条件"

```783:789:scripts/behaviourtree.lua
function IfNode(cond, name, node)
    return SequenceNode
        {
            ConditionNode(cond, name),
            node
        }
end
```

IfNode 是 SequenceNode{ConditionNode, node} 的语法糖。

**核心区别**：SequenceNode 有 idx 记忆。一旦 ConditionNode SUCCESS → idx 移到第二个子节点 → 之后的帧**不会再检查条件**，直到行为完成（SUCCESS 或 FAILED）后才重置 idx。

**使用场景**：条件只在"开始时"有意义，一旦开始就要做完。

最常见的例子——"夜晚回家"：

```70:71:scripts/brains/beebrain.lua
        IfNode(function() return not TheWorld.state.iscaveday or not self.inst:IsInLight() end, "IsNight",
            DoAction(self.inst, function() return beecommon.GoHomeAction(self.inst) end, "go home", true )),
```

蜜蜂一旦开始回家，即使突然日出也不会中断回家动作——会走完全程。这比 WhileNode 更合理，因为回家动作被中途打断会导致蜜蜂卡在半路。

#### 第三步：IfThenDoWhileNode——"先 If 启动，然后 While 维持"

```792:798:scripts/behaviourtree.lua
function IfThenDoWhileNode(ifcond, whilecond, name, node)
    return ParallelNode
        {
            MultiConditionNode(ifcond, whilecond, name),
            node
        }
end
```

这是一个更精细的组合：
- 第一次检查用 `ifcond`（启动条件，可能更严格）
- 后续检查用 `whilecond`（维持条件，可能更宽松）

**实际使用**——猪人帮忙砍树（来自 `braincommon.lua` 的 `NodeAssistLeaderDoAction`）：

```440:441:scripts/brains/braincommon.lua
    return IfThenDoWhileNode(ifnode, whilenode, action, looper)
```

启动条件是"玩家正在砍树"（需要看到玩家的砍树动画），维持条件是"玩家在附近"（更宽松，不需要玩家还在砍）。这让猪人"看到你砍树就开始帮忙，你走开了才停"。

#### 第四步：三者对比表

| 节点 | 条件检查频率 | 行为被打断条件 | 适用场景 |
|------|-------------|---------------|----------|
| `WhileNode` | **每帧** | 条件变 false 立即打断 | 条件随时可能变化（战斗、白天/黑夜） |
| `IfNode` | **只在进入时** | 行为自然结束才重新检查 | "一旦开始就做完"的动作（回家、吃东西） |
| `IfThenDoWhileNode` | 启动用 If，维持用 While | While 条件变 false 打断 | 启动条件严格、维持条件宽松 |

---

### 12.2.4 进阶：ConditionNode 与 MultiConditionNode——行为树的"门卫"

#### ConditionNode

```202:213:scripts/behaviourtree.lua
ConditionNode = Class(BehaviourNode, function(self, fn, name)
    BehaviourNode._ctor(self, name or "Condition")
    self.fn = fn
end)

function ConditionNode:Visit()
    if self.fn() then
        self.status = SUCCESS
    else
        self.status = FAILED
    end
end
```

**构造参数**：

| 参数 | 类型 | 含义 |
|------|------|------|
| `fn` | function | 条件检查函数，返回 true/false |
| `name` | string(可选) | 节点名称，用于调试 |

**特性**：永远不返回 RUNNING——它是瞬时的，一帧内完成判断。

**重要细节**：ConditionNode 在 `BehaviourNode:GetSleepTime()` 中被特殊处理（第 103 行）：

```103:103:scripts/behaviourtree.lua
    if self.status == RUNNING and not self.children and not self:is_a(ConditionNode) then
```

即使 ConditionNode 状态是 RUNNING（理论上不会，但防御性编程），也不会影响整棵树的休眠时间计算。

#### MultiConditionNode

```218:236:scripts/behaviourtree.lua
MultiConditionNode = Class(BehaviourNode, function(self, start, continue, name)
    BehaviourNode._ctor(self, name or "Condition")
    self.start = start
    self.continue = continue
end)

function MultiConditionNode:Visit()
    if not self.running then
        self.running = self.start()
    else
        self.running = self.continue()
    end

    if self.running then
        self.status = SUCCESS
    else
        self.status = FAILED
    end
end
```

**两个条件函数**：第一次 Visit 调用 `start()`，后续 Visit 调用 `continue()`。这是 `IfThenDoWhileNode` 的底层实现。

#### ConditionWaitNode

```242:253:scripts/behaviourtree.lua
ConditionWaitNode = Class(BehaviourNode, function(self, fn, name)
    BehaviourNode._ctor(self, name or "Wait")
    self.fn = fn
end)

function ConditionWaitNode:Visit()
    if self.fn() then
        self.status = SUCCESS
    else
        self.status = RUNNING
    end
end
```

与 ConditionNode 的区别：条件不满足时返回 **RUNNING** 而非 FAILED。意味着"条件还没满足，我在等它"。

---

### 12.2.5 进阶：ParallelNode——"同时做多件事"

```644:695:scripts/behaviourtree.lua
ParallelNode = Class(BehaviourNode, function(self, children, name)
    BehaviourNode._ctor(self, name or "Parallel", children)
end)

function ParallelNode:Visit()
    local done = true
    local any_done = false
    for idx, child in ipairs(self.children) do

        if child:is_a(ConditionNode) then
            child:Reset()
        end

        if child.status ~= SUCCESS then
            child:Visit()
            if child.status == FAILED then
                self.status = FAILED
                return
            end
        end

        if child.status == RUNNING then
            done = false
        else
            any_done = true
        end
    end

    if done or (self.stoponanycomplete and any_done) then
        self.status = SUCCESS
    else
        self.status = RUNNING
    end
end
```

**执行规则**：

1. 每帧 Visit 所有子节点
2. **ConditionNode 每帧先 Reset 再 Visit**——这是 WhileNode 能"每帧检查条件"的关键！
3. 已经 SUCCESS 的子节点不再 Visit
4. 任何子节点 FAILED → 整体 FAILED
5. 全部 SUCCESS → 整体 SUCCESS
6. 否则 → RUNNING

**`ParallelNodeAny` 变体**：

```692:695:scripts/behaviourtree.lua
ParallelNodeAny = Class(ParallelNode, function(self, children)
    ParallelNode._ctor(self, children, "Parallel(Any)")
    self.stoponanycomplete = true
end)
```

设置了 `stoponanycomplete = true`，任何一个子节点 SUCCESS 整体就 SUCCESS。

**Step 方法的特殊处理**：

```649:659:scripts/behaviourtree.lua
function ParallelNode:Step()
    if self.status ~= RUNNING then
        self:Reset()
    elseif self.children then
        for k, v in ipairs(self.children) do
            if v.status == SUCCESS and v:is_a(ConditionNode) then
                v:Reset()
            end
        end
    end
end
```

在 Step 阶段，ParallelNode 会把所有已 SUCCESS 的 ConditionNode Reset 回 READY——确保下一帧条件被重新检查。这是 WhileNode 机制的另一半。

---

### 12.2.6 进阶：EventNode、装饰器、LoopNode、LatchNode——"特种节点"

#### EventNode——事件驱动的响应式节点

```702:765:scripts/behaviourtree.lua
EventNode = Class(BehaviourNode, function(self, inst, event, child, priority)
    BehaviourNode._ctor(self, "Event("..event..")", {child})
    self.inst = inst
    self.event = event
    self.priority = priority or 0

    self.eventfn = function(inst, data) self:OnEvent(data) end
    self.inst:ListenForEvent(self.event, self.eventfn)
end)
```

**构造参数**：

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | 监听事件的实体 |
| `event` | string | 事件名 |
| `child` | BehaviourNode | 事件触发后要执行的行为 |
| `priority` | number(可选) | 事件优先级，默认 0。当 PriorityNode 中有多个 EventNode 时，高 priority 的可以抢占低 priority 的 |

**OnEvent 回调**（第 721-737 行）：

```721:737:scripts/behaviourtree.lua
function EventNode:OnEvent(data)
    if self.status == RUNNING then
        self.children[1]:Reset()
    end
    self.triggered = true
    self.data = data

    if self.inst.brain then
        self.inst.brain:ForceUpdate()
    end

    self:DoToParents(function(node) if node:is_a(PriorityNode) then node.lasttime = nil end end)
end
```

事件触发时做三件事：
1. 如果子节点正在 RUNNING，先 Reset 它
2. 设置 `triggered = true`，存储事件数据
3. **强制唤醒 Brain** 并**清空所有祖先 PriorityNode 的 lasttime**——让 PriorityNode 立即进入重评估模式

**Visit 逻辑**：

```749:765:scripts/behaviourtree.lua
function EventNode:Visit()
    if self.status == READY and self.triggered then
        self.status = RUNNING
    end

    if self.status == RUNNING then
        if self.children and #self.children == 1 then
            local child = self.children[1]
            child:Visit()
            self.status = child.status
        else
            self.status = FAILED
        end
    end
end
```

只有 `triggered = true` 时才会进入 RUNNING。未触发时始终停留在 READY，对 PriorityNode 来说就是"不存在"。

**使用场景**：braincommon.lua 中的 `PanicWhenScared` 内部使用了 EventNode 监听 "epicscare" 事件：

```127:129:scripts/brains/braincommon.lua
    local function onepicscarefn(inst, data)
        scareendtime = math.max(scareendtime, data.duration + GetTime() + math.random())
    end
    inst:ListenForEvent("epicscare", onepicscarefn)
```

#### 三个装饰器（Decorator）

装饰器只有一个子节点，用于**修改子节点的返回结果**。

**NotDecorator**——反转结果：

```378:392:scripts/behaviourtree.lua
NotDecorator = Class(DecoratorNode, function(self, child)
    DecoratorNode._ctor(self, "Not", child)
end)

function NotDecorator:Visit()
    local child = self.children[1]
    child:Visit()
    if child.status == SUCCESS then
        self.status = FAILED
    elseif child.status == FAILED then
        self.status = SUCCESS
    else
        self.status = child.status
    end
end
```

SUCCESS↔FAILED 互换，RUNNING 保持不变。

**FailIfRunningDecorator**——把 RUNNING 变成 FAILED：

```395:407:scripts/behaviourtree.lua
FailIfRunningDecorator = Class(DecoratorNode, function(self, child)
    DecoratorNode._ctor(self, "FailIfRunning", child)
end)

function FailIfRunningDecorator:Visit()
    local child = self.children[1]
    child:Visit()
    if child.status == RUNNING then
        self.status = FAILED
    else
        self.status = child.status
    end
end
```

用途：把一个持续行为变成"要么立即完成，要么放弃"。

**FailIfSuccessDecorator**——把 SUCCESS 变成 FAILED：

```411:423:scripts/behaviourtree.lua
FailIfSuccessDecorator = Class(DecoratorNode, function(self, child)
    DecoratorNode._ctor(self, "FailIfSuccess", child)
end)

function FailIfSuccessDecorator:Visit()
    local child = self.children[1]
    child:Visit()
    if child.status == SUCCESS then
        self.status = FAILED
    else
        self.status = child.status
    end
end
```

用途：让 PriorityNode 在子节点成功后也跳到下一个分支。实际案例——WX-78 靠近可附身底座后继续下一步：

```649:649:scripts/brains/braincommon.lua
                FailIfSuccessDecorator(Leash(self.inst, GetPossessableChassisPos, POSSESS_DIST, POSSESS_DIST_INNER, true)),
```

Leash 成功（到达目标位置）时返回 SUCCESS，但 FailIfSuccessDecorator 把它变成 FAILED，让 PriorityNode 继续检查下一个子节点（执行附身动作）。

#### LoopNode——"重复执行直到失败或达到上限"

```427:480:scripts/behaviourtree.lua
LoopNode = Class(BehaviourNode, function(self, children, maxreps)
    BehaviourNode._ctor(self, "Sequence", children)
    self.idx = 1
    self.maxreps = maxreps
    self.rep = 0
end)
```

**行为类似 SequenceNode**，但全部子节点 SUCCESS 后不返回 SUCCESS，而是**重置所有子节点、从头再来**。

| 参数 | 类型 | 含义 |
|------|------|------|
| `children` | table | 子节点数组 |
| `maxreps` | number/nil | 最大重复次数。nil 或不传则无限循环 |

**实际使用**——猪人帮忙砍树的循环（`braincommon.lua`）：

```435:437:scripts/brains/braincommon.lua
    if parameters.chatterstring then
        looper = LoopNode{ConditionNode(whilenode), ChattyNode(self.inst, parameters.chatterstring, DoAction(self.inst, findnode, "DoAction_Chatty", parameters.shouldrun, 3))}
    else
        looper = LoopNode{ConditionNode(whilenode), DoAction(self.inst, findnode, "DoAction_NoChatty", parameters.shouldrun, 3)}
    end
```

猪人反复执行"检查条件 → 砍树"的循环，直到条件不满足（ConditionNode FAILED → LoopNode FAILED）。

#### RandomNode——"随机选一个执行"

```484:529:scripts/behaviourtree.lua
RandomNode = Class(BehaviourNode, function(self, children)
    BehaviourNode._ctor(self, "Random", children)
end)
```

第一次 Visit 时随机选一个子节点。如果选中的 FAILED，就按顺序尝试下一个（环形遍历），全部 FAILED 则整体 FAILED。

#### LatchNode——"冷却锁"

```801:827:scripts/behaviourtree.lua
LatchNode = Class(BehaviourNode, function(self, inst, latchduration, child)
    BehaviourNode._ctor(self, "Latch ("..tostring(latchduration)..")", {child})
    self.inst = inst
    self.latchduration = latchduration
    self.currentlatchduration = 0
    self.lastlatchtime = -math.huge
end)
```

**参数**：

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | 实体引用 |
| `latchduration` | number/function | 冷却时间（秒）。可以是函数 |
| `child` | BehaviourNode | 子行为 |

每次执行后进入冷却期。冷却期内 Visit 直接返回 FAILED。

---

### 12.2.7 老手进阶：29 个 behaviours 模块——预制行为节点全解析

`scripts/behaviours/` 下有 29 个文件，每个文件定义一个继承自 `BehaviourNode` 的叶子行为。它们是**写 Brain 时最常用的积木块**。

#### 第一类：移动行为（8 个）

**① `ChaseAndAttack`——追击并攻击**

```1:1:scripts/behaviours/chaseandattack.lua
ChaseAndAttack = Class(BehaviourNode, function(self, inst, max_chase_time, give_up_dist, max_attacks, findnewtargetfn, walk, distance_from_ocean_target)
```

| 参数 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `inst` | EntityScript | 必填 | 执行追击的实体 |
| `max_chase_time` | number | nil | 最大追击时间（秒），超时放弃 |
| `give_up_dist` | number | nil | 最大追击距离，超距放弃 |
| `max_attacks` | number | nil | 最大攻击次数，达到后 SUCCESS |
| `findnewtargetfn` | function | nil | 当前无目标时，用此函数查找新目标 |
| `walk` | boolean | nil | 是否步行而非跑步追击 |
| `distance_from_ocean_target` | number/function | nil | 面对海上目标时保持的距离 |

Visit 逻辑：READY 时验证目标→ RUNNING 时持续追击、尝试攻击 → 目标死亡 SUCCESS、超时/超距/目标消失 FAILED。内部每 0.125 秒 Sleep 一次。

**② `RunAway`——逃跑**

```4:4:scripts/behaviours/runaway.lua
RunAway = Class(BehaviourNode, function(self, inst, hunterparams, see_dist, safe_dist, fn, runhome, fix_overhang, walk_instead, safe_point_fn)
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | 逃跑的实体 |
| `hunterparams` | string/table/function | 猎人的定义。string=标签名；table={tags,notags,fn,getfn}；function=过滤函数 |
| `see_dist` | number | 发现猎人的距离 |
| `safe_dist` | number | 安全距离（逃到这么远就停） |
| `fn` | function(可选) | 额外的判断函数，返回 false 则不逃 |
| `runhome` | boolean(可选) | 是否跑回家而非随机方向 |

**③ `Follow`——跟随**

```1:1:scripts/behaviours/follow.lua
Follow = Class(BehaviourNode, function(self, inst, target, min_dist, target_dist, max_dist, canrun, alwayseval, inlimbo_invalid)
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | 跟随者 |
| `target` | EntityScript/function | 跟随目标（可以是函数） |
| `min_dist` | number/function | 最小距离（太近就后退） |
| `target_dist` | number/function | 目标距离（到达此距离算 SUCCESS） |
| `max_dist` | number/function | 最大距离（超过此距离开始跑向目标） |
| `canrun` | boolean | 能否跑步（默认 true） |

三段距离设计：距离 < min_dist → 后退；min_dist ~ max_dist → 不动(FAILED)；> max_dist → 靠近。

**④ `Wander`——闲逛**

```8:8:scripts/behaviours/wander.lua
Wander = Class(BehaviourNode, function(self, inst, homelocation, max_dist, times, getdirectionFn, setdirectionFn, checkpointFn, data)
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | 闲逛的实体 |
| `homelocation` | Vector3/function | 家的位置，超出 max_dist 会走回来 |
| `max_dist` | number/function | 最大闲逛距离 |
| `times` | table(可选) | {minwalktime, randwalktime, minwaittime, randwaittime} |
| `data` | table(可选) | {wander_dist, should_run, ignore_walls, ...} |

**⑤ `Leash`——栓绳**

```1:1:scripts/behaviours/leash.lua
Leash = Class(BehaviourNode, function(self, inst, homelocation, max_dist, inner_return_dist, running)
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `homelocation` | Vector3/function | 锚点位置 |
| `max_dist` | number | 超出此距离就开始走回 |
| `inner_return_dist` | number | 走到此距离就停（SUCCESS） |
| `running` | boolean | 是否跑回 |

与 Wander 的区别：Leash 只做"走回家"这一件事，不闲逛。在 PriorityNode 中 Leash 通常放在 Wander 前面。

**⑥ `Panic`——恐慌乱跑**

```1:5:scripts/behaviours/panic.lua
Panic = Class(BehaviourNode, function(self, inst)
    BehaviourNode._ctor(self, "Panic")
    self.inst = inst
    self.waittime = 0
end)
```

每 0.25-0.5 秒随机选一个方向跑。始终返回 RUNNING——需要外层 WhileNode 来控制何时停止恐慌。

**⑦ `Approach`——接近**

简单的走向目标位置。

**⑧ `FindLight`——找光源**

```4:4:scripts/behaviours/findlight.lua
FindLight = Class(BehaviourNode, function(self, inst, see_dist, safe_dist)
```

寻找附近有 "lightsource" 标签的实体并走过去。每 5 秒重新搜索一次。

#### 第二类：战斗行为（4 个）

| 行为 | 功能 |
|------|------|
| `StandAndAttack` | 站桩攻击（不追击） |
| `AttackWall` | 攻击挡路的墙 |
| `UseShield` | 使用护盾防御 |
| `ChaseAndAttackAndAvoid` | 追击攻击同时躲避某些实体 |

#### 第三类：社交行为（2 个）

**`FaceEntity`——面向目标**

```1:1:scripts/behaviours/faceentity.lua
FaceEntity = Class(BehaviourNode, function(self, inst, getfn, keepfn, timeout, customalert)
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `getfn` | function(inst) | 获取面向目标的函数 |
| `keepfn` | function(inst, target) | 是否继续面向的判断函数 |
| `timeout` | number(可选) | 超时后 SUCCESS |
| `customalert` | string(可选) | 自定义警戒状态名 |

开始时停止移动，每 0.5 秒检查一次是否继续面向目标。

**`ChattyNode`——说话包装器**

```1:1:scripts/behaviours/chattynode.lua
ChattyNode = Class(BehaviourNode, function(self, inst, chatlines, child, delay, rand_delay, enter_delay, enter_delay_rand)
```

ChattyNode 是一个**装饰器风格的节点**——它包裹一个子节点，在子节点 RUNNING 时定时让实体"说话"。

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | 说话的实体 |
| `chatlines` | string/table/function | 对话内容（STRINGS 表的键名/数组/函数） |
| `child` | BehaviourNode | 被包裹的行为 |
| `delay` | number(可选) | 两次说话的最小间隔（默认 10 秒） |
| `rand_delay` | number(可选) | 随机额外间隔（默认 10 秒） |

#### 第四类：动作行为（3 个）

**`DoAction`——万能动作执行器**

```1:1:scripts/behaviours/doaction.lua
DoAction = Class(BehaviourNode, function(self, inst, getactionfn, name, run, timeout)
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | 执行动作的实体 |
| `getactionfn` | function(inst) | 返回 BufferedAction 的函数。返回 nil 则 FAILED |
| `name` | string(可选) | 节点名 |
| `run` | boolean/function(可选) | 是否跑向目标 |
| `timeout` | number(可选) | 超时秒数 |

DoAction 是最常用的叶子节点——只要你能构造一个 BufferedAction，就能让 NPC 执行任何动作。

Visit 逻辑：
1. READY 时调用 `getactionfn(inst)` 获取 BufferedAction
2. 如果获取到→ 注册成功/失败回调→ `locomotor:PushAction(action)` → RUNNING
3. RUNNING 时检查超时和 action 有效性
4. 收到回调→ SUCCESS 或 FAILED

**`FindFlower`——找花采蜜**

蜜蜂专用的行为节点。

**`FindFarmPlant`——找农作物**

农场相关的行为节点。

#### 第五类：其他行为（4 个）

| 行为 | 功能 |
|------|------|
| `StandStill` | 站着不动（有启动和维持条件） |
| `MinPeriod` | 最小间隔控制 |
| `ControlMinions` | 控制仆从 |
| `AvoidLight` | 躲避光源 |

---

### 12.2.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：DoAction 的 getactionfn 返回了"旧"的 BufferedAction

```lua
-- 错误：在 Brain 构造时就创建了 BufferedAction，之后一直复用
local eat_action = BufferedAction(inst, food, ACTIONS.EAT)
local function GetEatAction(inst) return eat_action end

DoAction(inst, GetEatAction)
```

**问题**：BufferedAction 是一次性的（7.4.8 陷阱 3）。一旦被执行过（Succeed 或 Fail），它就失效了。每次 DoAction 需要的是一个**全新的** BufferedAction。

**正确做法**：

```lua
local function GetEatAction(inst)
    local food = FindEntity(inst, 10, ...)
    return food ~= nil and BufferedAction(inst, food, ACTIONS.EAT) or nil
end

DoAction(inst, GetEatAction)
```

#### 陷阱 2：WhileNode 条件函数里做了昂贵操作

```lua
-- 错误：WhileNode 的条件每帧都执行，FindEntity 每帧搜索一次
WhileNode(function()
    return FindEntity(inst, 30, nil, {"monster"}) ~= nil
end, "HasMonster", RunAway(inst, "monster", 10, 15))
```

**问题**：WhileNode 的条件**每帧都被检查**（因为 ParallelNode 每帧都 Visit 所有子节点）。如果条件函数里做了 FindEntity 这样的搜索操作，每帧都搜索一次会严重影响性能。

**正确做法**：用一个缓存变量，定期更新：

```lua
local cached_result = false
local last_check_time = 0

WhileNode(function()
    local t = GetTime()
    if t - last_check_time > 1 then  -- 每秒检查一次
        cached_result = FindEntity(inst, 30, nil, {"monster"}) ~= nil
        last_check_time = t
    end
    return cached_result
end, "HasMonster", RunAway(inst, "monster", 10, 15))
```

或者用 EventNode 监听事件替代轮询。

#### 陷阱 3：PriorityNode 子节点顺序错误——低优先级行为永远得不到执行

```lua
-- 错误：Wander 放在了 Follow 前面
local root = PriorityNode({
    Wander(inst, ...),     -- 总是 RUNNING
    Follow(inst, ...),     -- 永远不会被执行！
}, 1)
```

**问题**：Wander 几乎总是返回 RUNNING（它一直在走来走去或等待）。PriorityNode 在非重评估帧直接继续 RUNNING 的子节点，不检查后面的。在重评估帧，从头遍历时 Wander 又返回 RUNNING，Follow 还是被跳过。

**正确做法**：优先级高的放前面。Follow 应在 Wander 之前：

```lua
local root = PriorityNode({
    Follow(inst, ...),     -- 有目标时跟随
    Wander(inst, ...),     -- 无事可做时闲逛
}, 1)
```

#### 陷阱 4：在 ConditionNode 的 fn 里修改了游戏状态

```lua
-- 错误：条件检查里产生了副作用
ConditionNode(function()
    inst.components.combat:SetTarget(FindEntity(inst, 10, nil, {"monster"}))
    return inst.components.combat.target ~= nil
end, "HasTarget")
```

**问题**：ConditionNode 可能被多次执行（PriorityNode 重评估时、ParallelNode 每帧都执行）。如果条件函数有副作用，会导致不可预测的行为。

**正确做法**：条件函数应该是**纯查询**，不修改状态：

```lua
ConditionNode(function()
    return inst.components.combat.target ~= nil
end, "HasTarget")
```

目标的设置应该交给其他逻辑（如 combat 组件的 retarget 回调）。

#### 陷阱 5：忘记 Sleep 导致 Brain 频繁更新

```lua
-- 错误：自定义叶子节点在 RUNNING 时没有调用 Sleep
function MyNode:Visit()
    if self.status ~= RUNNING then
        self.status = RUNNING
    end
    -- 做了一些检查...
    -- 忘了 self:Sleep(0.5)！
end
```

**问题**：没有 Sleep 的 RUNNING 节点会让 `GetTreeSleepTime()` 返回 0，BrainWrangler 每帧都更新这个 Brain，浪费性能。

**正确做法**：RUNNING 的叶子节点必须调用 `self:Sleep(t)` 指定下次检查的间隔。看原版行为节点：

```182:182:scripts/behaviours/runaway.lua
            self:Sleep(.25)
```

```155:155:scripts/behaviours/follow.lua
        self:Sleep(.25)
```

```54:54:scripts/behaviours/faceentity.lua
        self:Sleep(.5)
```

#### 陷阱 6：在行为树节点的构造函数里做了昂贵操作

```lua
-- 错误：在 OnStart 里的节点构造时就执行搜索
function MyBrain:OnStart()
    local target = FindEntity(self.inst, 30, ...)  -- OnStart 时搜索
    local root = PriorityNode({
        Follow(self.inst, function() return target end, ...),  -- target 被固定住了！
    }, 1)
    self.bt = BT(self.inst, root)
end
```

**问题**：`target` 在 OnStart 时就确定了，之后不会更新。Brain 的 OnStart 只调用一次！

**正确做法**：动态逻辑放在函数里，让每次 Visit 都能获取最新状态：

```lua
function MyBrain:OnStart()
    local root = PriorityNode({
        Follow(self.inst, function() return FindEntity(self.inst, 30, ...) end, ...),
    }, 1)
    self.bt = BT(self.inst, root)
end
```

#### 设计经验三条

**经验一：优先使用 DoAction 而不是自定义叶子节点**

大多数 NPC 行为可以归结为"走到某处+执行某个动作"——这正是 `DoAction` 做的事。只要你能构造一个 `BufferedAction`，就不需要写自定义叶子节点。

```lua
-- 几乎所有"找东西→做事"的行为都能用 DoAction
DoAction(inst, function()
    local target = FindEntity(inst, 10, ...)
    return target and BufferedAction(inst, target, ACTIONS.PICKUP) or nil
end, "PickUp", true)
```

**经验二：用 ChattyNode 包装行为让 NPC 更有"性格"**

ChattyNode 不改变行为逻辑，只在行为执行时让 NPC 说话。原版猪人大量使用这个模式：

```396:398:scripts/brains/pigbrain.lua
            ChattyNode(self.inst, "PIG_TALK_FIGHT",
                WhileNode(function() return ... end, "AttackMomentarily",
                    ChaseAndAttack(self.inst, MAX_CHASE_TIME, MAX_CHASE_DIST))),
```

猪人打架时会喊"PIG_TALK_FIGHT"对应的台词——只需一层包装就让 AI 更生动。

**经验三：行为树的"乐高积木"原则**

饥荒的行为树设计遵循**组合优于继承**的原则。复杂行为不是通过写一个巨大的自定义节点实现的，而是通过**组合小节点**搭建的：

```lua
-- "如果饥饿且看到食物，等一会儿，然后吃"——全部由现成节点组合
SequenceNode{
    ConditionNode(function() return IsHungry(inst) end),
    WaitNode(0.5),
    DoAction(inst, FindFoodAction),
}
```

每个节点只做一件小事，通过 PriorityNode / SequenceNode / WhileNode 等组合起来，就能表达复杂的 AI 行为。**当你发现需要写超过 50 行的自定义叶子节点时，先想想能不能用现有节点组合解决。**

---

### 12.2.9 小结

| 节点类型 | 类别 | 一句话总结 |
|----------|------|-----------|
| **PriorityNode** | 组合节点 | 带定时重评估的优先级选择器——行为树的"主心骨" |
| **SequenceNode** | 组合节点 | "全部成功才成功"——多步顺序流程 |
| **ParallelNode** | 组合节点 | "同时执行"——WhileNode 的底层实现 |
| **RandomNode** | 组合节点 | "随机选一个"——增加行为多样性 |
| **LoopNode** | 组合节点 | "反复执行"——直到失败或达到上限 |
| **ConditionNode** | 叶子节点 | 瞬时条件检查——行为树的"门卫" |
| **ConditionWaitNode** | 叶子节点 | 等待条件满足——条件不满足时 RUNNING |
| **ActionNode** | 叶子节点 | 立即执行一个函数——最简单的叶子 |
| **WaitNode** | 叶子节点 | 等待一段时间——延时器 |
| **EventNode** | 特殊节点 | 事件驱动——监听事件后激活子行为 |
| **LatchNode** | 特殊节点 | 冷却锁——执行后进入冷却期 |
| **WhileNode** | 便捷函数 | "条件满足时持续执行"——每帧检查条件 |
| **IfNode** | 便捷函数 | "条件满足时执行一次"——进入后不再检查 |
| **IfThenDoWhileNode** | 便捷函数 | "If 启动、While 维持"——精细条件控制 |
| **NotDecorator** | 装饰器 | 反转结果 |
| **FailIfRunningDecorator** | 装饰器 | RUNNING→FAILED |
| **FailIfSuccessDecorator** | 装饰器 | SUCCESS→FAILED |
| **DoAction** | behaviours | 万能动作执行器——最常用的叶子行为 |
| **ChaseAndAttack** | behaviours | 追击并攻击 |
| **RunAway** | behaviours | 逃跑 |
| **Follow** | behaviours | 三段距离跟随 |
| **Wander** | behaviours | 闲逛 |
| **Leash** | behaviours | 栓绳回家 |
| **Panic** | behaviours | 恐慌乱跑 |
| **FaceEntity** | behaviours | 面向目标 |
| **ChattyNode** | behaviours | 说话包装器 |

**下一节**（12.3）我们将详细讲解 `braincommon.lua` 中的通用 AI 行为模块——包括 PanicTrigger、PanicWhenScared、NodeAssistLeaderDoAction 等 Klei 提供的"高级行为积木"。

---

## 12.3 braincommon.lua——通用 AI 行为模块

### 本节导读

12.1-12.2 我们讲完了行为树的**底层节点类型**和 **29 个 behaviours 积木块**。但如果你翻开任意一个原版 Brain 文件，会发现几乎每个 Brain 的开头几行都有：

```lua
local BrainCommon = require "brains/braincommon"
```

`BrainCommon` 是 Klei 抽取出的**通用 AI 行为模块**——它把"着火恐慌"、"被吓恐慌"、"电击躲避"、"帮主人干活"、"帮主人捡东西"、"靠近盐舔石"这些在几十个 Brain 中**重复出现**的行为模式封装成了即插即用的函数。

> **新手**从 12.3.1-12.3.3 起步——了解 BrainCommon 提供了什么、怎么在 Brain 里使用、最常见的三个函数 PanicTrigger / ElectricFencePanicTrigger / PanicWhenScared 的用法；**进阶读者**继续看 12.3.4-12.3.5，深入 NodeAssistLeaderDoAction（帮主人干活）和 NodeAssistLeaderPickUps（帮主人捡东西）的完整实现和参数；**老手**跳到 12.3.6-12.3.7，了解 AnchorToSaltlick（盐舔石锚定）、PossessChassisNode（附身底座）等高级节点，以及六个常见陷阱与设计经验。

---

### 12.3.1 快速入门：BrainCommon 是什么——为什么需要它

#### 第一步：先看一个问题

打开猪人的 Brain（`pigbrain.lua`）和蜜蜂的 Brain（`beebrain.lua`），你会发现它们都有这样的代码：

**猪人**：
```lua
BrainCommon.PanicWhenScared(self.inst, .25, "PIG_TALK_PANICBOSS"),
WhileNode(function() return self.inst.components.hauntable and self.inst.components.hauntable.panic end, "PanicHaunted",
    ChattyNode(self.inst, "PIG_TALK_PANICHAUNT", Panic(self.inst))),
WhileNode(function() return self.inst.components.health.takingfiredamage end, "OnFire",
    ChattyNode(self.inst, "PIG_TALK_PANICFIRE", Panic(self.inst))),
```

**蜜蜂**：
```lua
BrainCommon.PanicTrigger(self.inst),
BrainCommon.ElectricFencePanicTrigger(self.inst),
```

**切斯特**：
```lua
BrainCommon.PanicTrigger(self.inst),
BrainCommon.ElectricFencePanicTrigger(self.inst),
```

**蜘蛛**：
```lua
BrainCommon.PanicWhenScared(self.inst, .3),
BrainCommon.PanicTrigger(self.inst),
BrainCommon.ElectricFencePanicTrigger(self.inst),
```

看到规律了吗？**"着火恐慌"和"电击恐慌"几乎出现在每一个 Brain 中。** 如果每个 Brain 都自己写一遍 WhileNode + 条件判断 + Panic，就会有大量重复代码。

BrainCommon 就是为了消除这种重复而诞生的。

#### 第二步：BrainCommon 的总览

`scripts/brains/braincommon.lua` 是一个 Lua 模块，返回一个 table `BrainCommon`，提供以下公开函数：

| 函数名 | 用途 | 使用频率 |
|--------|------|----------|
| `PanicTrigger(inst)` | 着火/被鬼吓时恐慌 | ★★★★★ 几乎每个 Brain |
| `ElectricFencePanicTrigger(inst)` | 被电击时躲避 | ★★★★★ 几乎每个 Brain |
| `PanicWhenScared(inst, chance, chatty)` | 被 Boss 吓到时恐慌 | ★★★★ 战斗型 NPC |
| `IpecacsyrupPanicTrigger(inst)` | 被催吐药影响时恐慌 | ★★★ 部分 NPC |
| `PanicTriggerShadowCreature(inst)` | 暗影生物特殊恐慌 | ★★ 暗影生物 |
| `NodeAssistLeaderDoAction(self, params)` | 帮主人干活（砍/挖/铲） | ★★★★ 跟随者 |
| `NodeAssistLeaderPickUps(self, params)` | 帮主人捡东西 | ★★★ 跟随者 |
| `AnchorToSaltlick(inst)` | 靠近盐舔石闲逛 | ★★ 牲畜 |
| `ShouldSeekSalt(inst)` | 判断是否需要盐 | ★★ 牲畜 |
| `PossessChassisNode(self, rate)` | WX-78 附身底座 | ★ 特殊 |
| `ShouldTriggerPanic(inst)` | 通用恐慌条件判断 | 内部使用 |
| `ShouldAvoidElectricFence(inst)` | 电击条件判断 | 内部使用 |

#### 第三步：在 Brain 中使用 BrainCommon

使用极其简单——在 PriorityNode 的 children 数组**最前面**放入 BrainCommon 的节点：

```lua
local BrainCommon = require "brains/braincommon"

function MyBrain:OnStart()
    local root = PriorityNode({
        -- 最高优先级：BrainCommon 提供的恐慌节点
        BrainCommon.PanicTrigger(self.inst),
        BrainCommon.ElectricFencePanicTrigger(self.inst),
        
        -- 你的自定义行为...
        ChaseAndAttack(self.inst, 10),
        Wander(self.inst, nil, 20),
    }, 1)
    self.bt = BT(self.inst, root)
end
```

**为什么要放最前面？** 因为 PriorityNode 从左到右按优先级处理。恐慌应该是最高优先级的行为——NPC 着火了，什么都不做，先跑。

> **新手记忆**：`BrainCommon` 是"行为树的标准前缀"。写新 Brain 时，**第一步**就是在 PriorityNode 最前面加上 `PanicTrigger` 和 `ElectricFencePanicTrigger`。这是行业惯例。

---

### 12.3.2 快速入门：PanicTrigger——"着火/被鬼吓就恐慌"

#### 第一步：源码

```86:94:scripts/brains/braincommon.lua
local function ShouldTriggerPanic(inst)
    return (inst.components.health and (inst.components.health.takingfiredamage or inst.components.health:GetLunarBurnFlags() ~= 0))
        or (inst.components.hauntable ~= nil and inst.components.hauntable.panic)
end

BrainCommon.ShouldTriggerPanic = ShouldTriggerPanic
BrainCommon.PanicTrigger = function(inst)
    return WhileNode(function() return ShouldTriggerPanic(inst) end, "PanicTrigger", Panic(inst))
end
```

#### 第二步：逻辑拆解

`PanicTrigger` 返回一个 `WhileNode`：
- **条件**：`ShouldTriggerPanic(inst)` —— 正在受到火焰伤害、月亮火伤害、或被鬼吓到处于 panic 状态
- **行为**：`Panic(inst)` —— 随机方向乱跑

**WhileNode 保证**：只要条件满足就持续恐慌，条件消失（火灭了/鬼走了）立即恢复正常。

#### 第三步：触发条件详解

| 条件 | 来源 | 含义 |
|------|------|------|
| `health.takingfiredamage` | health 组件 | 正在受到火焰伤害（被点燃） |
| `health:GetLunarBurnFlags() ~= 0` | health 组件 | 正在受到月亮火伤害 |
| `hauntable.panic` | hauntable 组件 | 被鬼吓到（玩家幽灵恐吓） |

---

### 12.3.3 快速入门：ElectricFencePanicTrigger 与 PanicWhenScared

#### ElectricFencePanicTrigger——"被电击时躲避"

```98:106:scripts/brains/braincommon.lua
require("behaviours/avoidelectricfence")
local function ShouldAvoidElectricFence(inst)
    return inst.panic_electric_field ~= nil
end

BrainCommon.ShouldAvoidElectricFence = ShouldAvoidElectricFence
BrainCommon.ElectricFencePanicTrigger = function(inst)
    return WhileNode(function() return ShouldAvoidElectricFence(inst) end, "ElectricShock", AvoidElectricFence(inst))
end
```

**与 PanicTrigger 的区别**：行为不是随机乱跑（Panic），而是 `AvoidElectricFence`——有方向地远离电场。

**触发条件**：`inst.panic_electric_field ~= nil` —— 实体身上有电场引用（被围墙感应灯/WX-78 超载等电击时设置）。

#### PanicWhenScared——"被 Boss 吓到时恐慌"

这是最复杂的恐慌节点。源码位于第 125-179 行：

```125:179:scripts/brains/braincommon.lua
local function PanicWhenScared(inst, loseloyaltychance, chatty)
    local scareendtime = 0
    local function onepicscarefn(inst, data)
        scareendtime = math.max(scareendtime, data.duration + GetTime() + math.random())
    end
    inst:ListenForEvent("epicscare", onepicscarefn)

    local panicscarednode = Panic(inst)

    if chatty ~= nil then
        panicscarednode = ChattyNode(inst, chatty, panicscarednode)
    end

    if loseloyaltychance ~= nil and loseloyaltychance > 0 then
        panicscarednode = ParallelNode{
            panicscarednode,
            LoopNode({
                WaitNode(3),
                ActionNode(function()
                    local leader = inst.components.follower ~= nil and inst.components.follower:GetLeader() or nil
                    if leader ~= nil and
                        inst.components.follower:GetLoyaltyPercent() > 0 and
                        TryLuckRoll(leader, loseloyaltychance, LuckFormulas.LoseFollowerOnPanic) then
                        inst.components.follower:SetLeader(nil)
                    end
                end),
            }),
        }
    end

    local scared = false
    panicscarednode = WhileNode(
        function()
            if (GetTime() < scareendtime) ~= scared then
                if inst.components.combat ~= nil then
                    inst.components.combat:SetTarget(nil)
                end
                scared = not scared
            end
            return scared
        end,
        "PanicScared",
        panicscarednode
    )

    local _OnStop = panicscarednode.OnStop
    panicscarednode.OnStop = function()
        inst:RemoveEventCallback("epicscare", onepicscarefn)
        if _OnStop ~= nil then
            _OnStop(panicscarednode)
        end
    end

    return panicscarednode
end
```

**参数**：

| 参数 | 类型 | 含义 |
|------|------|------|
| `inst` | EntityScript | NPC 实体 |
| `loseloyaltychance` | number(可选) | 每 3 秒有 X 概率失去忠诚（取消跟随）。nil 或 0 则不会 |
| `chatty` | string(可选) | 恐慌时说的话（STRINGS 键名） |

**工作原理**：

1. 监听 `"epicscare"` 事件（Boss 出现时推送），记录恐慌结束时间
2. 基础行为是 `Panic(inst)` 恐慌乱跑
3. 如果有 `chatty`，用 ChattyNode 包装（边跑边喊）
4. 如果有 `loseloyaltychance`，用 ParallelNode 并行执行"恐慌"+"每 3 秒判断是否叛逃"
5. 最外层用 WhileNode 包装，条件是"恐慌时间未到"
6. 恐慌开始/结束时清空战斗目标

**使用示例**——猪人被 Boss 吓到（25% 概率叛逃，会喊话）：

```385:385:scripts/brains/pigbrain.lua
            BrainCommon.PanicWhenScared(self.inst, .25, "PIG_TALK_PANICBOSS"),
```

蜘蛛被 Boss 吓到（30% 概率叛逃，不喊话）：

```137:137:scripts/brains/spiderbrain.lua
        BrainCommon.PanicWhenScared(self.inst, .3),
```

#### IpecacsyrupPanicTrigger——"催吐药恐慌"

```185:192:scripts/brains/braincommon.lua
local function IsUnderIpecacsyrupEffect(inst)
    return inst:HasDebuff("ipecacsyrup_buff")
end

BrainCommon.IpecacsyrupPanicTrigger = function(inst)
    return WhileNode(function() return BrainCommon.IsUnderIpecacsyrupEffect(inst) end, "IpecacsyrupPanicTrigger", Panic(inst))
end
```

当 NPC 身上有 "ipecacsyrup_buff" debuff 时恐慌。这是新版本加入的催吐药效果。

---

### 12.3.4 进阶：NodeAssistLeaderDoAction——"帮主人干活"

这是 BrainCommon 中最精巧的模块——让跟随者（如猪人、鱼人）在看到主人砍树/挖矿/铲地时**主动帮忙**。

#### 第一步：接口

```413:441:scripts/brains/braincommon.lua
local function NodeAssistLeaderDoAction(self, parameters)
    local action = parameters.action
    local defaults = AssistLeaderDefaults[action]

    local starter = parameters.starter or defaults.Starter
    local keepgoing = parameters.keepgoing or defaults.KeepGoing
    local finder = parameters.finder or defaults.FindNew

    local keepgoing_leaderdist = parameters.keepgoing_leaderdist or TUNING.FOLLOWER_HELP_LEADERDIST
    local finder_finddist = parameters.finder_finddist or TUNING.FOLLOWER_HELP_FINDDIST

    -- ...构建行为树节点...
    return IfThenDoWhileNode(ifnode, whilenode, action, looper)
end
```

**parameters 表的字段**：

| 字段 | 类型 | 必填 | 含义 |
|------|------|------|------|
| `action` | string | 是 | 动作类型："CHOP" / "MINE" / "DIG" / "TILL" |
| `starter` | function | 否 | 自定义启动条件（覆盖默认） |
| `keepgoing` | function | 否 | 自定义维持条件（覆盖默认） |
| `finder` | function | 否 | 自定义目标查找函数（覆盖默认） |
| `keepgoing_leaderdist` | number | 否 | 主人在多远内继续帮忙（默认 TUNING.FOLLOWER_HELP_LEADERDIST） |
| `finder_finddist` | number | 否 | 搜索目标的距离（默认 TUNING.FOLLOWER_HELP_FINDDIST） |
| `chatterstring` | string | 否 | 干活时说的话（STRINGS 键名） |
| `shouldrun` | boolean | 否 | 是否跑向目标 |

#### 第二步：四种动作的默认行为

`AssistLeaderDefaults` 表（第 287-407 行）为四种动作定义了默认的 Starter / KeepGoing / FindNew：

**CHOP（砍树）**：

| 回调 | 逻辑 |
|------|------|
| `Starter` | 主人正在"chopping"或"spinning"状态，或附近有怪物落叶松 |
| `KeepGoing` | 主人在 leaderdist 范围内，或附近有怪物落叶松 |
| `FindNew` | FindEntity 搜索 CHOP_workable 标签的目标 → 返回 BufferedAction(inst, target, ACTIONS.CHOP) |

**MINE（挖矿）**：

| 回调 | 逻辑 |
|------|------|
| `Starter` | 主人正在"mining"状态 |
| `KeepGoing` | 主人在 leaderdist 范围内 |
| `FindNew` | FindEntity 搜索 MINE_workable 标签 → BufferedAction(inst, target, ACTIONS.MINE) |

**DIG（掘地）** 和 **TILL（翻土）** 类似。

#### 第三步：节点构成

`NodeAssistLeaderDoAction` 返回的行为树结构是：

```
IfThenDoWhileNode(启动条件, 维持条件, 名字,
    LoopNode{
        ConditionNode(维持条件),
        [ChattyNode(说话,]
            DoAction(查找目标并执行)
        [)]
    }
)
```

拆解：
1. **IfThenDoWhileNode** —— 第一次用 Starter 检查"主人是否在砍树"，后续用 KeepGoing 检查"主人是否还在附近"
2. **LoopNode** —— 反复执行"检查条件 → 找目标 → 执行动作"
3. **DoAction** —— 找到最近的可砍树 → 创建 BufferedAction → locomotor:PushAction

#### 第四步：实际使用——猪人帮砍树

```337:342:scripts/brains/pigbrain.lua
            BrainCommon.NodeAssistLeaderDoAction(self, {
                action = "CHOP", -- Required.
                finder_finddist = SEE_TREE_DIST,
                keepgoing_leaderdist = KEEP_CHOPPING_DIST,
                chatterstring = "PIG_TALK_HELP_CHOP_WOOD",
            }),
```

猪人帮砍树时：搜索距离 15，主人在 10 格内继续砍，边砍边说"PIG_TALK_HELP_CHOP_WOOD"。

---

### 12.3.5 进阶：NodeAssistLeaderPickUps——"帮主人捡东西"

#### 第一步：接口

```565:613:scripts/brains/braincommon.lua
local function NodeAssistLeaderPickUps(self, parameters)
```

**parameters 表的字段**：

| 字段 | 类型 | 必填 | 含义 |
|------|------|------|------|
| `cond` | function | 否 | 是否应该捡东西的条件（默认永远 true） |
| `range` | number | 否 | 搜索范围（以主人为中心） |
| `range_local` | number | 否 | 额外的搜索范围（以 NPC 自身位置为中心） |
| `furthestfirst` | boolean | 否 | 是否先捡最远的 |
| `positionoverride` | Vector3/function | 否 | 覆盖搜索中心位置 |
| `ignorethese` | table | 否 | 忽略已尝试过的物品（带 5 秒冷却） |
| `wholestacks` | boolean | 否 | 是否只捡同类物品整堆 |
| `allowpickables` | boolean | 否 | 是否允许拾取可采摘物 |
| `give_cond` | function | 否 | 给主人东西的额外条件 |
| `give_range` | number | 否 | 给东西的距离限制 |
| `custom_pickup_filter` | function | 否 | 自定义物品过滤器 |
| `itemoverridefn` | function | 否 | 自定义物品获取函数 |

#### 第二步：内部流程

返回的行为树结构是：

```
PriorityNode({
    WhileNode(cond, "KeepPickup",
        DoAction(CustomPickUpAction)),
    WhileNode(give_cond, "Should Bring To Leader",
        PriorityNode({
            DoAction(GiveAction),
            DoAction(DropAction),
        })),
})
```

两个优先级分支：
1. **捡东西**：条件满足时，找到最近的可捡物品 → PICKUP / CHECKTRAP / PICK
2. **给主人**：背包里有东西时，走到主人身边 → GIVEALLTOPLAYER / GIVE / STORE。如果给不了就 DROP

#### 第三步：关键内部函数

**PickUpAction**（第 462-518 行）：

1. 先把 activeitem（手持物）丢掉
2. 如果 wholestacks 模式，只找和背包里同类的物品
3. 调用 `FindPickupableItem` 在 range 范围内搜索
4. 根据物品类型选择动作：trap → CHECKTRAP，pickable → PICK，其他 → PICKUP

**GiveAction**（第 520-550 行）：

1. 找到背包里的第一个物品
2. 检查主人能否接收
3. 根据主人类型选择动作：玩家 → GIVEALLTOPLAYER，NPC → GIVE，容器 → STORE

**IgnoreThis 机制**（第 452-460 行）：

捡取失败的物品会被加入 ignorethese 表，5 秒冷却后自动移除。防止 NPC 反复尝试捡同一个捡不到的物品。

---

### 12.3.6 老手进阶：AnchorToSaltlick 与 PossessChassisNode

#### AnchorToSaltlick——"靠近盐舔石闲逛"

```51:79:scripts/brains/braincommon.lua
local function AnchorToSaltlick(inst)
    local node = WhileNode(
        function()
            return FindSaltlick(inst)
        end,
        "Stay Near Salt",
        Wander(inst,
            function()
                return inst._brainsaltlick ~= nil
                    and inst._brainsaltlick:IsValid()
                    and inst._brainsaltlick:GetPosition()
                    or inst:GetPosition()
            end,
            WanderFromSaltlickDistFn)
    )

    local _OnStop = node.OnStop
    node.OnStop = function()
        if inst._brainsaltlick ~= nil then
            inst:RemoveEventCallback("saltlick_placed", OnSaltlickPlaced)
            inst._brainsaltlick = nil
        end
        if _OnStop ~= nil then
            _OnStop(node)
        end
    end

    return node
end
```

**工作原理**：

1. **FindSaltlick**（第 17-35 行）：搜索 TUNING.SALTLICK_CHECK_DIST 范围内的盐舔石，结果缓存在 `inst._brainsaltlick`
2. **WhileNode 条件**：附近有可用的盐舔石
3. **行为**：以盐舔石为中心进行 Wander，距离根据"距离需要舔盐的时间"动态调整——越急越靠近
4. **OnStop 清理**：节点停止时移除事件监听、清空缓存

**动态距离函数 WanderFromSaltlickDistFn**（第 37-43 行）：

```37:43:scripts/brains/braincommon.lua
local function WanderFromSaltlickDistFn(inst)
    local t = inst.components.timer ~= nil and (inst.components.timer:GetTimeLeft("salt") or 0) or nil
    return t ~= nil
        and t < TIME_TO_SEEK_SALT
        and Remap(math.max(TIME_TO_SEEK_SALT * .5, t), TIME_TO_SEEK_SALT * .5, TIME_TO_SEEK_SALT, TUNING.SALTLICK_USE_DIST * .75, TUNING.SALTLICK_CHECK_DIST * .75)
        or TUNING.SALTLICK_CHECK_DIST * .75
end
```

距离需要舔盐的时间越短（timer "salt" 剩余越少），闲逛范围越小（越靠近盐舔石）。超过 16 秒则使用较大的默认闲逛范围。

**配套函数 ShouldSeekSalt**：判断是否需要盐（timer 剩余 < 16 秒且附近有盐舔石）。

#### PossessChassisNode——"WX-78 附身底座"

```646:658:scripts/brains/braincommon.lua
local function PossessChassis(self, update_rate)
    return IfNode(function() return SelectPossessableChassis(self) end, "possess chassis",
            PriorityNode({
                FailIfSuccessDecorator(Leash(self.inst, GetPossessableChassisPos, POSSESS_DIST, POSSESS_DIST_INNER, true)),
                IfNode(function() return CheckPossessableChassis(self) end, "possess",
                    ActionNode(function() self.inst:PushEventImmediate("possess_chassis", { target = self.possessable_chassis }) end)),
                FaceEntity(self.inst,
                    function() return self.possessable_chassis end,
                    function() return CheckPossessableChassis(self) end),
            }, update_rate))
end
```

三步流程：
1. **Leash** 走到底座附近（FailIfSuccessDecorator 包装，到达后让出执行权）
2. **ActionNode** 推送 "possess_chassis" 事件（实际附身逻辑）
3. **FaceEntity** 如果上面两步都失败，至少面向底座

这是一个展示**装饰器实战用法**的好例子：`FailIfSuccessDecorator(Leash(...))` —— Leash 成功（到达目标）时，如果不加装饰器，PriorityNode 会认为任务完成。加了 FailIfSuccessDecorator 后，PriorityNode 会继续检查下一个子节点（执行附身），实现了"走到 → 然后附身"的效果。

---

### 12.3.7 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：PanicTrigger 放错位置——不在最前面

```lua
-- 错误：PanicTrigger 不是第一个
local root = PriorityNode({
    ChaseAndAttack(self.inst, 10),
    BrainCommon.PanicTrigger(self.inst),  -- 放在战斗后面！
    Wander(self.inst),
}, 1)
```

**问题**：NPC 着火时，如果正在战斗（ChaseAndAttack RUNNING），PriorityNode 只有在重评估时才会检查 PanicTrigger。但即使重评估，ChaseAndAttack 排在前面、也返回了 RUNNING，PanicTrigger 永远不会被选中。

**正确做法**：恐慌节点永远放在 PriorityNode 的最前面。

#### 陷阱 2：同时使用 PanicTrigger 和手写的着火恐慌

```lua
-- 错误：重复了！
BrainCommon.PanicTrigger(self.inst),
WhileNode(function() return self.inst.components.health.takingfiredamage end, "OnFire",
    Panic(self.inst)),
```

**问题**：PanicTrigger 已经包含了着火判断。手写的 WhileNode 是冗余的，而且 PriorityNode 会根据位置选一个执行，另一个被跳过。

**正确做法**：只用 PanicTrigger，或者如果需要自定义着火行为（如加 ChattyNode），就不用 PanicTrigger，完全自己写。猪人就是这种模式——它没有用 PanicTrigger，而是自己写了带 ChattyNode 的版本。

#### 陷阱 3：NodeAssistLeaderDoAction 的 action 参数拼写错误

```lua
-- 错误：action 字符串写错了
BrainCommon.NodeAssistLeaderDoAction(self, {
    action = "Chop",  -- 应该是 "CHOP"（全大写）
})
```

**问题**：`AssistLeaderDefaults` 表的 key 是 "CHOP"、"MINE"、"DIG"、"TILL"，全大写。如果拼写不匹配，`defaults` 为 nil，后续代码会崩溃。

#### 陷阱 4：PanicWhenScared 的清理问题——OnStop 被覆盖

```lua
-- 有风险：在 PanicWhenScared 返回的节点上覆写 OnStop
local scarednode = BrainCommon.PanicWhenScared(self.inst, .25)
scarednode.OnStop = function()
    -- 自定义清理，但忘了调用原始的 OnStop！
    DoMyCleanup()
end
```

**问题**：PanicWhenScared 内部覆写了 OnStop 来 RemoveEventCallback。如果你再覆写 OnStop 而不调用原始版本，事件监听会泄漏。

**正确做法**：

```lua
local scarednode = BrainCommon.PanicWhenScared(self.inst, .25)
local _OrigOnStop = scarednode.OnStop
scarednode.OnStop = function()
    DoMyCleanup()
    if _OrigOnStop then _OrigOnStop(scarednode) end
end
```

这正是 Klei 自己在 BrainCommon 内部用的模式（第 170-176 行）。

#### 陷阱 5：NodeAssistLeaderPickUps 忘记传 ignorethese

```lua
-- 潜在问题：没传 ignorethese
BrainCommon.NodeAssistLeaderPickUps(self, {
    range = 10,
})
```

**问题**：如果 NPC 附近有一个无法捡起的物品（比如太重或被挡住），它会**每帧都尝试捡**，反复失败又反复尝试。

**正确做法**：传入一个 ignorethese table：

```lua
local _ignorethese = {}
BrainCommon.NodeAssistLeaderPickUps(self, {
    range = 10,
    ignorethese = _ignorethese,
})
```

ignorethese 会自动记录失败的物品并冷却 5 秒。

#### 陷阱 6：Mod 中自定义 AssistLeaderDefaults 但没注册

如果你想让跟随者帮忙做自定义动作（如自定义的 ACTIONS.MYACTION），需要向 `BrainCommon.AssistLeaderDefaults` 表添加条目：

```lua
-- 在 modmain.lua 中
local BrainCommon = require "brains/braincommon"
BrainCommon.AssistLeaderDefaults.MYACTION = {
    Starter = function(inst, leaderdist, finddist)
        local leader = inst.components.follower and inst.components.follower:GetLeader()
        return leader ~= nil and leader.sg:HasStateTag("myaction_tag")
    end,
    KeepGoing = function(inst, leaderdist, finddist)
        local leader = inst.components.follower and inst.components.follower:GetLeader()
        return leader ~= nil and inst:IsNear(leader, leaderdist)
    end,
    FindNew = function(inst, leaderdist, finddist)
        local target = FindEntity(inst, finddist, nil, {"MYACTION_workable"})
        return target ~= nil and BufferedAction(inst, target, ACTIONS.MYACTION) or nil
    end,
}
```

Klei 专门暴露了 `BrainCommon.AssistLeaderDefaults` 供 Mod 使用（第 409 行的注释 `-- Mod support access.`）。

#### 设计经验三条

**经验一：BrainCommon 的"标准前缀"模板**

写任何新 Brain 时，建议使用以下标准前缀：

```lua
local BrainCommon = require "brains/braincommon"

function MyBrain:OnStart()
    local root = PriorityNode({
        -- === 标准前缀（按需选用）===
        BrainCommon.PanicWhenScared(self.inst, 0.25),  -- 如果是战斗型 NPC
        BrainCommon.PanicTrigger(self.inst),
        BrainCommon.ElectricFencePanicTrigger(self.inst),
        BrainCommon.IpecacsyrupPanicTrigger(self.inst), -- 如果可被催吐药影响
        
        -- === 你的自定义行为 ===
        -- ...
    }, period)
    self.bt = BT(self.inst, root)
end
```

**经验二：善用 NodeAssistLeaderDoAction 的可扩展性**

`NodeAssistLeaderDoAction` 的每个回调都可以被 parameters 覆盖。如果默认行为不满足需求，不要重写整个函数——只覆盖需要改的回调：

```lua
BrainCommon.NodeAssistLeaderDoAction(self, {
    action = "CHOP",
    -- 自定义启动条件：只有主人特定指令时才帮忙
    starter = function(inst, leaderdist, finddist)
        return inst.should_help_chop == true
    end,
    -- 其他回调使用默认值
})
```

**经验三：学习 BrainCommon 的"OnStop 链"模式**

BrainCommon 在需要清理资源（RemoveEventCallback）时，使用了一个优雅的模式：

```lua
local _OnStop = node.OnStop
node.OnStop = function()
    -- 自己的清理
    inst:RemoveEventCallback("myevent", myfn)
    -- 调用原始 OnStop
    if _OnStop ~= nil then _OnStop(node) end
end
```

这种"保存旧回调→包装新回调→在新回调里调用旧回调"的链式模式，在饥荒 Mod 开发中非常常见。它保证了多层包装不会互相覆盖。

---

### 12.3.8 小结

| 函数 | 返回节点类型 | 一句话总结 |
|------|-------------|-----------|
| `PanicTrigger` | WhileNode | 着火/被鬼吓→恐慌乱跑 |
| `ElectricFencePanicTrigger` | WhileNode | 被电击→躲避电场 |
| `PanicWhenScared` | WhileNode(嵌套) | 被 Boss 吓→恐慌+可能叛逃+可喊话 |
| `IpecacsyrupPanicTrigger` | WhileNode | 催吐药→恐慌 |
| `PanicTriggerShadowCreature` | WhileNode | 暗影生物特殊恐慌 |
| `NodeAssistLeaderDoAction` | IfThenDoWhileNode | 帮主人砍/挖/铲/翻 |
| `NodeAssistLeaderPickUps` | PriorityNode | 帮主人捡东西+给主人 |
| `AnchorToSaltlick` | WhileNode | 靠近盐舔石闲逛 |
| `ShouldSeekSalt` | function | 判断是否需要盐 |
| `PossessChassisNode` | IfNode | WX-78 附身底座 |

**核心原则**：BrainCommon 不是"可选的工具"，它是"**行为树的标准库**"。写新 Brain 时先加 BrainCommon 前缀，再写自定义行为。这样你的 NPC 就自动拥有了正确的恐慌、电击、Boss 恐惧响应——这些都是玩家期望每个 NPC 都有的基本行为。

**下一节**（12.4）我们将讲解 BufferedAction 如何在 Brain 中被使用——即 AI 系统与 Action 系统的交汇点。

---

## 12.4 BufferedAction——动作排队系统

### 本节导读

第 7 章（7.4 节）我们从**玩家视角**讲完了 BufferedAction 的完整生命周期——从创建到执行到销毁。本节换一个视角：**NPC/AI 视角**。当行为树决策"这只猪人现在应该去吃地上那块肉"时，代码层面到底发生了什么？BufferedAction 如何从 Brain 的决策层，穿过 locomotor 的移动层，到达 StateGraph 的执行层？

> **新手**从 12.4.1-12.4.3 起步——理解"Brain 决策 → BufferedAction → NPC 执行"的完整链路、DoAction 行为节点的工作原理、一个实际的"猪人吃肉"全流程追踪；**进阶读者**继续看 12.4.4-12.4.5，深入 `LocoMotor:PushAction` 的分支逻辑、NPC 的 StateGraph 如何接收和执行 BufferedAction；**老手**跳到 12.4.6-12.4.7，了解动作队列与中断机制、AI 动作与玩家动作的差异，六个常见陷阱与设计经验。

---

### 12.4.1 快速入门：Brain 是怎么让 NPC "做事"的

#### 第一步：回顾两个系统

到目前为止你已经学过两个独立的系统：

- **第 7 章的 Action 系统**：定义了"什么动作存在"（ACTIONS 表）、"动作怎么执行"（fn 回调）、"动作的运行时载体"（BufferedAction）
- **第 12 章的 Brain/AI 系统**：定义了"NPC 怎么决策"（行为树）、"什么时候做什么"（PriorityNode + 条件节点）

**问题**：这两个系统怎么连接？Brain 做出"吃东西"的决策后，怎么变成实际的"走到肉前面→播放吃东西动画→调用 ACTIONS.EAT.fn"？

**答案**：**BufferedAction 是两个系统的桥梁。** Brain 的叶子行为节点（如 DoAction）创建 BufferedAction，交给 locomotor 组件去执行。

#### 第二步：完整链路——从决策到执行

```
                    Brain 决策层
                        │
               行为树 Visit() 遍历
                        │
            ┌───────────┼───────────┐
            │           │           │
      ChaseAndAttack  DoAction   Follow    ...其他行为节点
            │           │           │
            │    getactionfn(inst)  │
            │     返回 BufferedAction│
            │           │           │
            └───────────┼───────────┘
                        │
              locomotor:PushAction(ba)
                        │
         ┌──────────────┼──────────────┐
         │              │              │
     需要走过去？    即时动作？      需要面向？
         │              │              │
    GoToEntity/    PushBuffered     FacePoint
    GoToPoint       Action          +PushBA
         │              │              │
    locomotor 驱动   StateGraph       直接执行
    走到目标附近     StartAction
         │              │
    到达 → PushBA  ActionHandler
         │         匹配目标 State
         │              │
         └──────────────┘
                  │
          sg:PerformBufferedAction
                  │
          bufferedaction:Do()
                  │
          ACTIONS.EAT.fn(act)
                  │
           实际业务逻辑
```

#### 第三步：关键角色介绍

| 角色 | 文件 | 职责 |
|------|------|------|
| **DoAction 行为节点** | `behaviours/doaction.lua` | 创建 BufferedAction，交给 locomotor |
| **LocoMotor:PushAction** | `components/locomotor.lua` | 决定是否需要走路、面向，然后提交给 entity |
| **EntityScript:PushBufferedAction** | `entityscript.lua` | 把 BufferedAction 交给 StateGraph |
| **StateGraph:StartAction** | `stategraph.lua` | 找到匹配的 ActionHandler，切换 State |
| **EntityScript:PerformBufferedAction** | `entityscript.lua` | 在动画的"接触帧"调用 bufferedaction:Do() |

> **新手记忆**：**DoAction 创建"快递包"（BufferedAction）→ locomotor 送快递（走过去）→ StateGraph 签收（播放动画）→ PerformBufferedAction 拆包（执行 fn）。** 这就是 Brain 让 NPC "做事"的完整流程。

---

### 12.4.2 快速入门：DoAction 行为节点——Brain 到 BufferedAction 的入口

#### 第一步：回顾 DoAction 源码

```1:58:scripts/behaviours/doaction.lua
DoAction = Class(BehaviourNode, function(self, inst, getactionfn, name, run, timeout)
    BehaviourNode._ctor(self, name or "DoAction")
    self.inst = inst
    self.shouldrun = run
    self.action = nil
    self.getactionfn = getactionfn
    self.time = nil
    self.timeout = timeout
end)

function DoAction:Visit()

    if self.status == READY then
        local action = self.getactionfn(self.inst)
        self.action = action
        self.pendingstatus = nil

        if action then
            action:AddFailAction(function()
                if action == self.action and self.pendingstatus == nil then
                    self:OnFail()
                end
            end)
            action:AddSuccessAction(function()
                if action == self.action then
                    self:OnSucceed()
                end
            end)
            self.inst.components.locomotor:PushAction(action, FunctionOrValue(self.shouldrun))
            self.time = GetTime()
            self.status = RUNNING
        else
            self.status = FAILED
        end
    end

    if self.status == RUNNING then
        if self.timeout and (GetTime() - self.time > self.timeout) then
            self.status = FAILED
        end

        if self.pendingstatus then
            self.status = self.pendingstatus
        elseif not self.action:IsValid() then
            self.status = FAILED
        end
    end

end
```

#### 第二步：执行流程分解

**READY → 第一次 Visit**：

1. 调用 `getactionfn(inst)` 获取 BufferedAction
2. 如果返回 nil → FAILED（没有可执行的动作）
3. 如果返回 BufferedAction：
   - 注册 **FailAction 回调**：动作失败时设置 pendingstatus = FAILED
   - 注册 **SuccessAction 回调**：动作成功时设置 pendingstatus = SUCCESS
   - 调用 `locomotor:PushAction(action, run)` → **把 BufferedAction 交给 locomotor**
   - 状态变为 RUNNING

**RUNNING → 后续 Visit**：

4. 检查超时
5. 检查 pendingstatus（来自回调）→ 如果有，设置为最终状态
6. 检查 action:IsValid() → 如果失效，FAILED

**关键设计**：DoAction 本身不做任何"走路"或"执行动作"的逻辑——它只负责**创建 BufferedAction 并交给 locomotor**，然后**等待回调通知结果**。所有实际工作由 locomotor → entity → StateGraph 完成。

#### 第三步：getactionfn 的编写规范

getactionfn 是 DoAction 的"灵魂"。它必须满足：

1. **接收 inst 参数**，返回 BufferedAction 或 nil
2. **每次调用都返回新的 BufferedAction**（不能复用旧的）
3. **不应有副作用**（不修改游戏状态）
4. **高频调用安全**（可能被 Reset 后反复调用）

典型写法：

```lua
local function FindFoodAction(inst)
    -- 1. 搜索目标
    local target = FindEntity(inst, 10, function(item)
        return inst.components.eater:CanEat(item) and item:IsOnValidGround()
    end)
    
    -- 2. 没找到→返回 nil
    if target == nil then return nil end
    
    -- 3. 找到→返回新的 BufferedAction
    return BufferedAction(inst, target, ACTIONS.EAT)
end

-- 在 Brain 中使用
DoAction(self.inst, FindFoodAction, "EatFood", true)
```

---

### 12.4.3 快速入门：追踪一次"猪人吃肉"的完整流程

让我们用猪人的 Brain 追踪一次完整的"看到肉→走过去→吃掉"流程：

#### 第一帧：行为树决策

1. BrainManager 调用 `PigBrain:OnUpdate()`
2. BT:Update() → root:Visit()
3. PriorityNode 从头评估：恐慌？否。战斗？否。白天日常行为？是。
4. 进入白天子树：`FindFoodAction` 找到了地上一块肉
5. DoAction 节点创建 `BufferedAction(pigman, meat, ACTIONS.EAT)`
6. DoAction 调用 `pigman.components.locomotor:PushAction(ba, true)`

#### 第二步：locomotor 接手

7. `LocoMotor:PushAction` 检查 BufferedAction（第 882-980 行）：
   - 调用 `bufferedaction:TestForStart()` —— 检查动作是否可以开始
   - 肉在远处，有 target，不是 instant → 走 `GoToEntity(meat, ba, true)` 路径
   - locomotor 设置 dest = 肉的位置，开始移动

#### 第三~N 帧：走向目标

8. locomotor 每帧驱动猪人走向肉
9. DoAction 节点状态保持 RUNNING
10. 行为树每次 Visit 时，DoAction 检查：超时？否。action 有效？是。继续等。

#### 到达目标帧：StateGraph 接手

11. locomotor 检测到到达目标范围
12. 调用 `EntityScript:PushBufferedAction(ba)`（第 1609-1652 行）：
    - 旧的 bufferedaction Fail 并清空
    - TestForStart() 再次验证
    - 不是 WALKTO，不是 instant → 设置 `self.bufferedaction = ba`
    - 调用 `self.sg:StartAction(ba)` → StateGraph 查找对应的 ActionHandler

13. StateGraph 的 actionhandlers 表中有 EAT → 切换到 "eat" 状态
14. "eat" 状态的 timeline 在某一帧调用 `inst:PerformBufferedAction()`

#### 执行帧：动作回调

15. `EntityScript:PerformBufferedAction()`（第 1654 行起）：
    - 面向目标
    - 推送 "performaction" 事件
    - 调用 `bufferedaction:Do()`

16. `BufferedAction:Do()` 内部：
    - IsValid 检查（6 项）
    - 调用 `ACTIONS.EAT.fn(act)` → 实际执行吃东西逻辑
    - 成功→ `bufferedaction:Succeed()` → 触发 SuccessAction 回调

#### 回调帧：DoAction 完成

17. DoAction 之前注册的 SuccessAction 回调被触发
18. `self.pendingstatus = SUCCESS`
19. 下一次 Visit 时，DoAction 检测到 pendingstatus → 设置 `self.status = SUCCESS`
20. SequenceNode/PriorityNode 收到 SUCCESS → 决定下一步行为

**整个流程耗时可能几十帧**，但代码结构清晰：Brain 只负责决策，BufferedAction 负责承载，locomotor 负责移动，StateGraph 负责动画和执行。

---

### 12.4.4 进阶：LocoMotor:PushAction 的分支逻辑

`LocoMotor:PushAction`（第 882-980 行）是 Brain 系统通向 Action 系统的"大门"。它根据 BufferedAction 的属性决定不同的处理路径：

#### 第一步：通用前置处理（882-903 行）

```882:903:scripts/components/locomotor.lua
function LocoMotor:PushAction(bufferedaction, run, try_instant)
    if bufferedaction == nil then
        return
    elseif self.inst.components.playercontroller ~= nil then
        self.inst.components.playercontroller:OnRemoteBufferedAction()
    end

    if bufferedaction.action.pre_action_cb ~= nil then
        bufferedaction.action.pre_action_cb(bufferedaction)
    end

    self.throttle = 1
    local success, reason = bufferedaction:TestForStart()
    if not success then
        self.inst:PushEvent("actionfailed", { action = bufferedaction, reason = reason })
        return
    end

    self:Clear()
```

1. 如果有 playercontroller → 通知远程操作（NPC 没有，跳过）
2. 如果 Action 有 `pre_action_cb` → 执行前置回调
3. **TestForStart** 验证动作是否可执行 → 失败则推送 "actionfailed" 事件并返回
4. Clear 清空当前移动状态

#### 第二步：六大分支路径

| 条件 | 路径 | 含义 |
|------|------|------|
| action == WALKTO | GoToEntity/GoToPoint | 纯走路（到达即成功） |
| action == LOOKAT（玩家） | 特殊处理 | 检视动作，可能不需要走 |
| bufferedaction.forced | GoToPoint（带覆盖） | 远程指定动作 |
| action.instant 或 action.do_not_locomote | PushBufferedAction | 即时动作，不需要走路 |
| 有 target | GoToEntity | 走到目标实体附近 |
| 有 action_pos | GoToPoint | 走到目标位置 |
| 无 target 无 pos | PushBufferedAction | 原地执行 |

**对于 NPC 来说**，最常走的路径是：
- **有 target** → `GoToEntity(target, ba, run)` → locomotor 驱动 NPC 走到目标附近
- **action.instant** → 直接 `PushBufferedAction` → StateGraph 立即处理

#### 第三步：GoToEntity 之后发生什么

`GoToEntity`（第 982 行）设置目的地，让 locomotor 每帧驱动实体移动。当到达目标附近（在 action 的 distance 范围内）时，locomotor 调用 `inst:PushBufferedAction(ba)`，把控制权交给 StateGraph。

---

### 12.4.5 进阶：EntityScript:PushBufferedAction——交接给 StateGraph

```1609:1652:scripts/entityscript.lua
function EntityScript:PushBufferedAction(bufferedaction)
    if bufferedaction ~= nil and
        self.bufferedaction ~= nil and
        bufferedaction.target == self.bufferedaction.target and
        bufferedaction.action == self.bufferedaction.action and
        bufferedaction.invobject == self.bufferedaction.invobject and
        not (self.sg ~= nil and self.sg:HasStateTag("idle")) then
        return
    end

    if self.bufferedaction ~= nil then
        self.bufferedaction:Fail()
        self.bufferedaction = nil
    end

    local success, reason = bufferedaction:TestForStart()
    if not success then
        self:PushEvent("actionfailed", { action = bufferedaction, reason = reason })
        return
    end

    if bufferedaction.action == ACTIONS.WALKTO then
        self:PushEvent("performaction", { action = bufferedaction })
        bufferedaction:Succeed()
        self.bufferedaction = nil
    elseif bufferedaction.action.instant or bufferedaction.options.instant then
        -- 即时动作：面向目标 → 直接 Do()
        if bufferedaction.target ~= nil and ... then
            self:FacePoint(...)
        end
        self:PushEvent("performaction", { action = bufferedaction })
        bufferedaction:Do()
        self.bufferedaction = nil
    else
        -- 需要 StateGraph 处理
        self.bufferedaction = bufferedaction
        if self.sg == nil then
            self:PushEvent("startaction", { action = bufferedaction })
        elseif not self.sg:StartAction(bufferedaction) then
            self:PushEvent("performaction", { action = bufferedaction })
            self.bufferedaction:Fail()
            self.bufferedaction = nil
        end
    end
end
```

**三大路径**：

1. **WALKTO**：纯走路动作 → 直接 Succeed（locomotor 已经完成了走路）
2. **instant**：即时动作 → 直接调用 `Do()` 执行
3. **需要 SG**：最常见路径 → `sg:StartAction(ba)` → StateGraph 找到 ActionHandler → 切换到对应 State → 播放动画 → 在接触帧调用 `PerformBufferedAction()`

**重复动作去重**（第 1610-1617 行）：如果新旧 BufferedAction 的 target、action、invobject 都相同，且 NPC 不在 idle 状态，就跳过——防止行为树反复创建相同的 BufferedAction 导致动作重启。

---

### 12.4.6 老手进阶：AI 动作与玩家动作的差异

#### 第一步：AI 不需要客户端预测

玩家的动作涉及客户端预测（PreviewBufferedAction）和远程同步（RemoteBufferedAction）。NPC 的动作完全在**服务端**执行，不需要这些机制。在 `PushAction` 中：

```885:887:scripts/components/locomotor.lua
    elseif self.inst.components.playercontroller ~= nil then
        self.inst.components.playercontroller:OnRemoteBufferedAction()
    end
```

NPC 没有 playercontroller，这段代码被跳过。

#### 第二步：AI 的 BufferedAction 来源不同

| | 玩家 | NPC |
|---|---|---|
| **创建者** | PlayerActionPicker + DoAction | Brain 的 getactionfn |
| **触发** | 鼠标点击/按键 | 行为树 Visit |
| **路径** | PlayerController → LocoMotor | DoAction → LocoMotor |
| **网络同步** | 客户端→服务端 RPC | 不需要(纯服务端) |
| **客户端预测** | 有 | 无 |

#### 第三步：AI 动作的中断机制

NPC 的当前动作会在以下情况被中断：

1. **PriorityNode 重评估**：更高优先级的行为抢占 → 当前 DoAction 被 Reset → DoAction 不再检查 action 状态（但已提交的 BufferedAction 可能还在执行）
2. **locomotor:Clear()**：新的 PushAction 调用会先 Clear 旧动作 → 旧 BufferedAction 被 Fail
3. **EntityScript:PushBufferedAction 的旧动作处理**：新 BufferedAction 提交时，旧的被 Fail

**关键理解**：当行为树切换行为时，旧的 BufferedAction **不是被"取消"的，而是被"失败"（Fail）的**。这会触发 FailAction 回调，DoAction 的 pendingstatus 变为 FAILED。

---

### 12.4.7 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：getactionfn 里忘记检查 locomotor

```lua
-- 错误：NPC 可能没有 locomotor
local function MyAction(inst)
    return BufferedAction(inst, target, ACTIONS.MYACTION)
end
DoAction(inst, MyAction)
```

**问题**：DoAction 的 Visit 里直接调用 `self.inst.components.locomotor:PushAction`。如果实体没有 locomotor 组件，直接崩溃。

**正确做法**：确保使用 DoAction 的实体一定有 locomotor。这通常不是问题（所有有 Brain 的实体都应该有 locomotor），但自定义 prefab 时要注意。

#### 陷阱 2：DoAction 的 getactionfn 执行了昂贵搜索但被频繁 Reset

```lua
-- 潜在性能问题
DoAction(inst, function()
    -- FindEntity 搜索整个世界
    local target = FindEntity(inst, 100, function(item)
        return HeavyValidation(item)
    end)
    return target and BufferedAction(inst, target, ACTIONS.PICKUP) or nil
end)
```

**问题**：如果这个 DoAction 在 PriorityNode 中，每次重评估时都可能先 Reset 再重新 Visit → getactionfn 被反复调用。如果搜索范围大、验证复杂，会严重影响性能。

**正确做法**：在 getactionfn 里加缓存：

```lua
local cached_target = nil
local last_search_time = 0
local function MyAction(inst)
    if GetTime() - last_search_time > 2 then
        cached_target = FindEntity(inst, 100, HeavyValidation)
        last_search_time = GetTime()
    end
    return cached_target and cached_target:IsValid()
        and BufferedAction(inst, cached_target, ACTIONS.PICKUP)
        or nil
end
```

#### 陷阱 3：BufferedAction 的 target 在执行前消失

```lua
local function EatAction(inst)
    local food = FindEntity(inst, 10, ...)
    return food and BufferedAction(inst, food, ACTIONS.EAT) or nil
end
```

**问题**：在 DoAction 创建 BufferedAction 和 locomotor 实际走到目标之间，目标可能被其他 NPC 先吃了、被玩家捡了、或者消失了。

**处理**：这种情况下 `bufferedaction:IsValid()` 会返回 false（它检查 target 是否有效）。DoAction 在 RUNNING 时每帧检查 `self.action:IsValid()`，发现无效就 FAILED。整个流程是安全的。

**但要注意**：NPC 会先走到目标位置，发现目标没了才停下。如果你希望 NPC 能更早发现目标消失并停止走路，需要在行为树上层加 WhileNode 条件。

#### 陷阱 4：在同一帧创建多个 BufferedAction 给同一个 NPC

```lua
-- 错误：两个 DoAction 在同一帧给同一个 NPC
PriorityNode({
    DoAction(inst, GetEatAction),     -- 创建 BA1
    DoAction(inst, GetPickupAction),  -- 创建 BA2
}, 0)  -- period=0 意味着每帧都从头评估
```

**问题**：每帧 PriorityNode 从头评估时，如果 GetEatAction 返回了 BA1 → DoAction1 PushAction → locomotor 开始走向食物。但 PriorityNode 还在继续评估 → GetPickupAction 也返回了 BA2...

**实际不会出问题**：因为 PriorityNode 找到第一个 SUCCESS/RUNNING 的子节点就停止了。第二个 DoAction 不会被 Visit。但如果两个 DoAction 在 ParallelNode 中，就会出问题——locomotor:PushAction 会 Clear 旧动作。

#### 陷阱 5：忘记 DoAction 的 timeout 参数

```lua
-- 潜在问题：没设 timeout
DoAction(inst, function()
    return BufferedAction(inst, faraway_target, ACTIONS.PICKUP)
end)
```

**问题**：如果目标很远，NPC 可能走很久。如果目标在走路过程中变得不可达（比如在海里），NPC 会一直走、一直 RUNNING，行为树被锁死在这个分支上。

**正确做法**：设置合理的 timeout：

```lua
DoAction(inst, GetPickupAction, "Pickup", true, 10)  -- 10 秒超时
```

BrainCommon 中的 NodeAssistLeaderDoAction 就给 DoAction 设了 3 秒超时。

#### 陷阱 6：直接用 locomotor:RunInDirection 而不是 DoAction

```lua
-- 不推荐：在自定义叶子节点里直接操作 locomotor
function MyNode:Visit()
    self.inst.components.locomotor:RunInDirection(math.random() * 360)
    self.status = RUNNING
end
```

**问题**：直接操作 locomotor 绕过了 BufferedAction 系统。没有 TestForStart 验证、没有 ActionHandler 匹配、没有成功/失败回调。如果只是移动（不涉及动作执行），这样做是可以的（Panic、RunAway 就是这样做的）。但如果需要执行具体动作（吃、砍、捡），必须通过 BufferedAction。

**规则**：
- **纯移动**（跑、走、逃跑）→ 可以直接操作 locomotor
- **执行动作**（吃、砍、捡、攻击）→ 必须用 BufferedAction → DoAction

#### 设计经验三条

**经验一：理解 DoAction 的"委托"模式**

DoAction 不做任何实际工作。它是一个**委托者**：
1. 委托 getactionfn 决定"做什么"
2. 委托 locomotor 负责"走过去"
3. 委托 StateGraph 负责"播放动画和执行"
4. 通过回调得知"结果"

这种设计让 Brain 代码保持简洁——你只需要写一个 getactionfn 函数就能让 NPC 执行任何动作。

**经验二：Brain 的其他行为节点也在创建 BufferedAction**

DoAction 不是唯一创建 BufferedAction 的地方。其他行为节点的工作方式各不相同：

| 行为节点 | 如何驱动 NPC |
|----------|-------------|
| `DoAction` | 创建 BufferedAction → locomotor:PushAction |
| `ChaseAndAttack` | 直接操作 locomotor:GoToPoint + combat:TryAttack |
| `Follow` | 直接操作 locomotor:GoToPoint / WalkInDirection |
| `RunAway` | 直接操作 locomotor:RunInDirection |
| `Wander` | 直接操作 locomotor:GoToPoint / WalkInDirection |
| `Panic` | 直接操作 locomotor:RunInDirection |
| `Leash` | 直接操作 locomotor:GoToPoint |

**只有 DoAction 和 FindFlower/FindFarmPlant 使用 BufferedAction**。其他行为节点直接操作 locomotor 完成纯移动任务。

**经验三：BufferedAction 是"一次性"的**

这在 7.4 节讲过，但在 AI 上下文中特别重要：每次 DoAction 被 Reset 后重新 Visit 时，`getactionfn` 必须返回一个**全新的** BufferedAction。旧的 BufferedAction 在 Succeed 或 Fail 后就失效了。

这就是为什么 getactionfn 是一个**函数**而不是一个预先创建好的 BufferedAction——它每次被调用都创建新的。

---

### 12.4.8 小结

| 概念 | 一句话总结 |
|------|-----------|
| Brain → BufferedAction | DoAction 的 getactionfn 创建 BufferedAction，是两个系统的桥梁 |
| locomotor:PushAction | 根据动作类型决定走路/即时/面向，然后提交给 entity |
| EntityScript:PushBufferedAction | 把 BufferedAction 交给 StateGraph 处理 |
| StateGraph:StartAction | 找到匹配的 ActionHandler，切换到对应 State |
| PerformBufferedAction | 在动画接触帧执行 bufferedaction:Do() |
| AI vs 玩家 | AI 不需要客户端预测和网络同步，完全在服务端执行 |
| 中断机制 | 行为切换时旧 BufferedAction 被 Fail，不是"取消" |
| 纯移动 vs 动作 | 纯移动直接操作 locomotor；执行动作必须通过 BufferedAction |

**下一节**（12.5）我们将讲解常见的 AI 行为模式——巡逻、追击、逃跑、采集、归巢等经典场景的实现方式。

---

## 12.5 常见 AI 模式：巡逻、追击、逃跑、采集、归巢

### 本节导读

12.1-12.4 我们讲完了行为树的底层机制。本节换一个角度——**从游戏设计的需求出发**，看饥荒如何用行为树实现五种最常见的 AI 行为模式。每种模式我们都会给出"在游戏里的表现"→"对应的行为树节点组合"→"原版代码实例"→"Mod 中如何复用"。

> **新手**从 12.5.1-12.5.3 起步——理解巡逻（Wander+Leash）、追击（ChaseAndAttack）、逃跑（RunAway+Panic）三种最基础的模式；**进阶读者**继续看 12.5.4-12.5.5，掌握采集（DoAction+条件组合）和归巢（GoHome+时间条件）模式；**老手**跳到 12.5.6-12.5.7，学习如何组合这些模式形成完整的 Brain，六个设计经验。

---

### 12.5.1 快速入门：巡逻模式（Wander + Leash）

#### 第一步：游戏中的表现

观察一只没有主人的猪人白天的行为：它在猪舍附近走来走去，走一会儿停一会儿，永远不会跑太远。

这就是**巡逻模式**——NPC 在一个区域内随机移动，但不超出活动范围。

#### 第二步：核心节点组合

巡逻模式由两个节点配合完成：

```
PriorityNode({
    Leash(inst, homePos, maxDist, returnDist),  -- 拉回来
    Wander(inst, homePos, maxDist, times),       -- 随机走
}, period)
```

**Leash 在 Wander 前面**：当 NPC 走得太远（超过 maxDist），Leash 优先触发，把 NPC 拉回 returnDist 以内。回来后 Leash SUCCESS → PriorityNode 下一帧重评估 → Leash FAILED（已在范围内）→ 执行 Wander。

#### 第三步：原版实例——猪人的白天巡逻

```349:355:scripts/brains/pigbrain.lua
            Leash(self.inst, GetNoLeaderHomePos, LEASH_MAX_DIST, LEASH_RETURN_DIST),
            -- ...中间省略一些社交行为...
            Wander(self.inst, GetNoLeaderHomePos, MAX_WANDER_DIST)
```

参数：
- `GetNoLeaderHomePos`：没有主人时返回猪舍位置，有主人时返回 nil
- `LEASH_MAX_DIST = 30`：超过 30 格就往回走
- `LEASH_RETURN_DIST = 10`：走到离家 10 格以内停止
- `MAX_WANDER_DIST = 20`：闲逛最远 20 格

**动态锚点**：注意 homePos 是一个**函数**——当猪人有主人时返回 nil，Leash 和 Wander 都不生效（改为 Follow 模式）。这是饥荒 AI 的灵活之处：锚点可以动态切换。

#### 第四步：Wander 的 times 参数控制节奏

```lua
Wander(inst, homePos, maxDist, {
    minwalktime = 2,    -- 最少走 2 秒
    randwalktime = 3,   -- 额外随机 0~3 秒
    minwaittime = 1,    -- 最少站 1 秒
    randwaittime = 3,   -- 额外随机 0~3 秒
})
```

NPC 会交替进行"走一会儿"和"站一会儿"，模拟自然的巡逻感觉。

#### 第五步：Mod 中如何使用

```lua
-- 最简单的巡逻 Brain
function MyCreatureBrain:OnStart()
    local root = PriorityNode({
        BrainCommon.PanicTrigger(self.inst),
        BrainCommon.ElectricFencePanicTrigger(self.inst),
        
        Leash(self.inst,
            function() return self.inst.components.knownlocations:GetLocation("home") end,
            30, 10),
        Wander(self.inst,
            function() return self.inst.components.knownlocations:GetLocation("home") end,
            20),
    }, 1)
    self.bt = BT(self.inst, root)
end

function MyCreatureBrain:OnInitializationComplete()
    self.inst.components.knownlocations:RememberLocation("home", self.inst:GetPosition())
end
```

**OnInitializationComplete** 在 Brain 启动后调用，用来记住出生点。这是蜘蛛、蜜蜂等很多生物的做法。

---

### 12.5.2 快速入门：追击模式（ChaseAndAttack）

#### 第一步：游戏中的表现

一只猪人看到蜘蛛：冲过去、打一下、蜘蛛跑了追上去、再打、直到蜘蛛死或跑太远放弃。

#### 第二步：核心节点

```lua
WhileNode(function() return HasTarget(inst) end, "HasTarget",
    ChaseAndAttack(inst, maxChaseTime, giveUpDist))
```

或者更常见的组合——攻击 + 躲闪：

```lua
-- 攻击中
WhileNode(function() return not inst.components.combat:InCooldown() end, "Attack",
    ChaseAndAttack(inst, MAX_CHASE_TIME, MAX_CHASE_DIST)),
-- 攻击冷却中躲闪
WhileNode(function() return inst.components.combat:InCooldown() end, "Dodge",
    RunAway(inst, { getfn = GetTarget }, RUN_AWAY_DIST, STOP_RUN_AWAY_DIST)),
```

#### 第三步：原版实例——蜜蜂的追击+躲闪

```63:66:scripts/brains/beebrain.lua
        WhileNode(function() return not self.inst.components.combat:HasTarget() or not self.inst.components.combat:InCooldown() end, "AttackMomentarily",
            ChaseAndAttack(self.inst, SpringCombatMod(MAX_CHASE_TIME), SpringCombatMod(MAX_CHASE_DIST))),
        WhileNode(function() return self.inst.components.combat:HasTarget() and self.inst.components.combat:InCooldown() end, "Dodge",
            RunAway(self.inst, { getfn = GetRunAwayTarget }, RUN_AWAY_DIST, STOP_RUN_AWAY_DIST)),
```

蜜蜂的战斗策略：
- 能攻击时（不在冷却）→ 追击并攻击
- 在攻击冷却中 → 远离目标（躲闪）

`SpringCombatMod` 函数根据季节（春天）调整战斗参数。

#### 第四步：ChaseAndAttack 的内部机制回顾

1. READY：验证目标 → 发出战吼 → RUNNING
2. RUNNING：每 0.125 秒检查一次——
   - 目标死亡 → SUCCESS
   - 目标消失/超距/超时 → FAILED（放弃追击）
   - 在攻击范围内 → `combat:TryAttack()`
   - 不在攻击范围 → `locomotor:GoToPoint(targetPos)` 追过去
3. 监听 "onattackother" 事件重置追击计时器

#### 第五步：Mod 中的追击模式

```lua
-- 简单的追击 Brain
function AggressiveBrain:OnStart()
    local root = PriorityNode({
        BrainCommon.PanicTrigger(self.inst),
        
        -- 有敌人就追击，最多追 15 秒或 30 格
        ChaseAndAttack(self.inst, 15, 30),
        
        -- 没敌人就闲逛
        Wander(self.inst, nil, 20),
    }, 0.5)
    self.bt = BT(self.inst, root)
end
```

**注意**：ChaseAndAttack 直接放在 PriorityNode 里即可，不需要 WhileNode 包装。因为 ChaseAndAttack 在没有目标时自己返回 FAILED。

但如果你想在攻击冷却期做其他事情（如躲闪），就需要 WhileNode 来细分状态。

---

### 12.5.3 快速入门：逃跑模式（RunAway + Panic）

#### 第一步：两种逃跑的区别

| 行为 | 方向 | 使用场景 |
|------|------|----------|
| `RunAway` | **远离**特定威胁 | 看到天敌、被攻击 |
| `Panic` | **随机**方向乱跑 | 着火、被鬼吓、极端恐慌 |

#### 第二步：RunAway 的三种 hunterparams

```lua
-- 方式 1：字符串（标签名）
RunAway(inst, "spider", 4, 8)   -- 看到"spider"标签的就跑

-- 方式 2：table（复杂过滤）
RunAway(inst, {
    tags = { "pig", "_combat" },
    fn = function(guy, inst)
        return guy.components.combat:TargetIs(inst)
    end,
}, 5, 7)   -- 看到正在攻击自己的猪人就跑

-- 方式 3：function + getfn
RunAway(inst, { getfn = function(inst)
    return inst.components.combat.target
end }, 6, 10)   -- 远离自己的战斗目标
```

#### 第三步：原版实例——猪人的多层逃跑

```358:379:scripts/brains/pigbrain.lua
    local night = WhileNode( function() return not TheWorld.state.isday end, "IsNight",
        PriorityNode{
            ChattyNode(self.inst, "PIG_TALK_RUN_FROM_SPIDER",
                RunAway(self.inst, "spider", 4, 8)),                    -- 逃离蜘蛛
            -- ...
            ChattyNode(self.inst, "PIG_TALK_PANIC",
                Panic(self.inst)),                                       -- 找不到安全地方就恐慌
        }, 1)
```

猪人夜晚的逃跑优先级：
1. 远离蜘蛛（RunAway，有方向性）
2. 回家（DoAction + GoHome）
3. 在光源附近闲逛
4. 找光源（FindLight）
5. 找不到光源 → 恐慌乱跑（Panic，最后的兜底）

#### 第四步：Panic 的实现原理

```7:22:scripts/behaviours/panic.lua
function Panic:Visit()
    if self.status == READY then
        self:PickNewDirection()
        self.status = RUNNING
    else
        if GetTime() > self.waittime then
            self:PickNewDirection()
        end
        self:Sleep(self.waittime - GetTime())
    end
end

function Panic:PickNewDirection()
    self.inst.components.locomotor:RunInDirection(math.random() * 360)
    self.waittime = GetTime() + .25 + math.random() * .25
end
```

每 0.25-0.5 秒随机选一个方向跑。**Panic 永远返回 RUNNING**——它不会自己停下来。必须用外层 WhileNode 来控制：

```lua
WhileNode(function() return inst.components.health.takingfiredamage end, "OnFire",
    Panic(inst))
```

火灭了 → WhileNode FAILED → Panic 被中断。

---

### 12.5.4 进阶：采集模式（DoAction + 条件组合）

#### 第一步：游戏中的表现

猪人看到地上有肉 → 走过去 → 吃掉。蜜蜂看到花 → 飞过去 → 采蜜。

#### 第二步：基础采集模式

```lua
DoAction(inst, function()
    local target = FindEntity(inst, SEARCH_DIST, ValidateTarget, mustTags, cantTags)
    return target and BufferedAction(inst, target, ACTION) or nil
end, "Collect", true)
```

三要素：搜索范围、过滤条件、执行动作。

#### 第三步：原版实例——猪人找食物

```81:119:scripts/brains/pigbrain.lua
local function FindFoodAction(inst)
    if inst.sg:HasStateTag("busy") then
        return
    end

    -- 先检查背包里有没有可吃的
    if inst.components.inventory ~= nil and inst.components.eater ~= nil then
        local target = inst.components.inventory:FindItem(function(item) return inst.components.eater:CanEat(item) end)
        if target ~= nil then
            return BufferedAction(inst, target, ACTIONS.EAT)
        end
    end

    -- 检查是否在看比赛（看比赛时不吃东西）
    if inst.components.minigame_spectator ~= nil then
        return
    end

    -- 检查上次吃东西的时间（不要太频繁地找吃的）
    local time_since_eat = inst.components.eater:TimeSinceLastEating()
    if time_since_eat ~= nil and time_since_eat <= TUNING.PIG_MIN_POOP_PERIOD * 2 then
        return
    end

    -- 刚吃过不久只找肉（不吃素）
    local noveggie = time_since_eat ~= nil and time_since_eat < TUNING.PIG_MIN_POOP_PERIOD * 4

    -- 搜索地上的食物
    inst.brain_noveggie = noveggie
    local target = FindEntity(inst, SEE_FOOD_DIST, IsFoodValid, nil, FINDFOOD_CANT_TAGS, inst.components.eater:GetEdibleTags())
    inst.brain_noveggie = nil

    if target ~= nil then
        return BufferedAction(inst, target, ACTIONS.EAT)
    end
    
    -- 还可以从架子上拿食物
    -- ...
end
```

**关键设计点**：

1. **状态检查**：`sg:HasStateTag("busy")` → 正忙着就不找吃的
2. **优先级**：先检查背包 → 再搜索地面
3. **频率控制**：根据上次进食时间控制搜索频率
4. **口味偏好**：刚吃过的猪人不吃素菜
5. **过滤条件**：食物必须在地上至少 8 秒（防止抢玩家刚丢的食物）

#### 第四步：带条件的采集——"饥饿时才找吃的"

```lua
-- IfNode：饥饿时找吃的（一旦开始就做完）
IfNode(function() return IsHungry(inst) end, "IsHungry",
    DoAction(inst, FindFoodAction))

-- WhileNode：条件每帧检查（更灵敏但更耗 CPU）
WhileNode(function() return IsHungry(inst) end, "WhileHungry",
    DoAction(inst, FindFoodAction))
```

#### 第五步：多目标采集——蜜蜂采花

蜜蜂使用专门的 `FindFlower` 行为节点而不是 DoAction，因为采花有更复杂的逻辑（采够花粉后回家）。但基础模式是一样的：搜索 → 走过去 → 采集。

---

### 12.5.5 进阶：归巢模式（GoHome + 时间/条件触发）

#### 第一步：游戏中的表现

夜晚来临 → 猪人跑回猪舍。冬天来了 → 蜜蜂飞回蜂巢。花粉满了 → 蜜蜂飞回蜂巢。

#### 第二步：核心模式

```lua
IfNode(function() return ShouldGoHome(inst) end, "TimeToGoHome",
    DoAction(inst, function()
        local home = inst.components.homeseeker and inst.components.homeseeker.home
        return home and home:IsValid() and BufferedAction(inst, home, ACTIONS.GOHOME) or nil
    end, "GoHome", true))
```

要素：
- 条件函数判断"是否该回家"
- DoAction 创建 GOHOME 动作
- homeseeker 组件提供家的引用

#### 第三步：原版实例——蜜蜂的三种回家条件

```70:75:scripts/brains/beebrain.lua
        IfNode(function() return not TheWorld.state.iscaveday or not self.inst:IsInLight() end, "IsNight",
            DoAction(self.inst, function() return beecommon.GoHomeAction(self.inst) end, "go home", true )),
        IfNode(function() return self.inst.components.pollinator:HasCollectedEnough() end, "IsFullOfPollen",
            DoAction(self.inst, function() return beecommon.GoHomeAction(self.inst) end, "go home", true )),
        IfNode(function() return TheWorld.state.iswinter end, "IsWinter",
            DoAction(self.inst, function() return beecommon.GoHomeAction(self.inst) end, "go home", true )),
```

蜜蜂在三种情况下回家：
1. 天黑了或不在光线中
2. 采够花粉了
3. 冬天来了

注意都是用 **IfNode** 而非 WhileNode——一旦开始回家，即使条件变了（比如突然天亮了），也会走完全程回到家。

#### 第四步：猪人的归巢——更复杂的条件

```133:139:scripts/brains/pigbrain.lua
local function GoHomeAction(inst)
    if not GetLeader(inst) and
        HasValidHome(inst) and
        not inst.components.combat.target then
            return BufferedAction(inst, inst.components.homeseeker.home, ACTIONS.GOHOME)
    end
end
```

猪人回家需要同时满足：没有主人（没被玩家喂肉）+ 家没烧掉 + 没在打架。

```365:367:scripts/brains/pigbrain.lua
            ChattyNode(self.inst, "PIG_TALK_GO_HOME",
                WhileNode( function() return not TheWorld.state.iscaveday or not self.inst:IsInLight() end, "Cave nightness",
                    DoAction(self.inst, GoHomeAction, "go home", true ))),
```

猪人用 **WhileNode** 而非 IfNode——如果天亮了（条件不满足），猪人会**中断回家**，转而做白天的行为。这与蜜蜂不同。

**设计选择**：
- 蜜蜂用 IfNode：回家是"一次性任务"，中途不应被打断
- 猪人用 WhileNode：猪人更"聪明"，天亮了就不急着回家了

#### 第五步：GOHOME 动作的实际效果

当 NPC 执行 GOHOME 动作到达家门口时，`ACTIONS.GOHOME.fn` 会调用 `homeseeker.home.components.childspawner:GoHome(inst)` 或类似方法——NPC 实体被移除（"进入"了房子），childspawner 记录这只 NPC 在家里。下次天亮时 childspawner 再把它"放出来"。

---

### 12.5.6 老手进阶：五种模式的组合——构建完整的 Brain

#### 第一步：模板——最简单的完整 Brain

```lua
require "behaviours/wander"
require "behaviours/chaseandattack"
require "behaviours/runaway"
require "behaviours/doaction"
require "behaviours/leash"
local BrainCommon = require "brains/braincommon"

local MyBrain = Class(Brain, function(self, inst)
    Brain._ctor(self, inst)
end)

local function FindFoodAction(inst)
    local target = FindEntity(inst, 15, function(item)
        return inst.components.eater:CanEat(item) and item:IsOnValidGround()
    end, nil, {"INLIMBO"})
    return target and BufferedAction(inst, target, ACTIONS.EAT) or nil
end

local function GoHomeAction(inst)
    local home = inst.components.homeseeker and inst.components.homeseeker.home
    return home and home:IsValid() and not home:HasTag("burnt")
        and BufferedAction(inst, home, ACTIONS.GOHOME) or nil
end

function MyBrain:OnStart()
    local root = PriorityNode({
        -- 第一层：生存（最高优先级）
        BrainCommon.PanicWhenScared(self.inst, 0.25),
        BrainCommon.PanicTrigger(self.inst),
        BrainCommon.ElectricFencePanicTrigger(self.inst),
        
        -- 第二层：战斗
        ChaseAndAttack(self.inst, 10, 30),
        
        -- 第三层：需求
        DoAction(self.inst, FindFoodAction, "Eat", true),
        
        -- 第四层：归巢
        IfNode(function() return not TheWorld.state.isday end, "Night",
            DoAction(self.inst, GoHomeAction, "GoHome", true)),
        
        -- 第五层：巡逻（最低优先级）
        Leash(self.inst,
            function() return self.inst.components.knownlocations:GetLocation("home") end,
            30, 10),
        Wander(self.inst,
            function() return self.inst.components.knownlocations:GetLocation("home") end,
            20),
    }, 0.5)
    
    self.bt = BT(self.inst, root)
end

function MyBrain:OnInitializationComplete()
    self.inst.components.knownlocations:RememberLocation("home", self.inst:GetPosition())
end

return MyBrain
```

这个模板涵盖了全部五种模式，按 12.1 中的"优先级金字塔"排列。

#### 第二步：Chester 的极简 Brain——当你不需要战斗和采集

```29:40:scripts/brains/chesterbrain.lua
function ChesterBrain:OnStart()
    local root =
    PriorityNode({
        BrainCommon.PanicTrigger(self.inst),
        BrainCommon.ElectricFencePanicTrigger(self.inst),
        Follow(self.inst, function() return GetLeader(self.inst) end, MIN_FOLLOW_DIST, TARGET_FOLLOW_DIST, MAX_FOLLOW_DIST),
        FaceEntity(self.inst, GetFaceTargetFn, KeepFaceTargetFn),
        Wander(self.inst, function() return self.inst.components.knownlocations:GetLocation("home") end, MAX_WANDER_DIST),
    }, .25)
    self.bt = BT(self.inst, root)
end
```

Chester 只需要：恐慌 → 跟随主人 → 面向主人 → 闲逛。没有战斗、采集、归巢。Period 是 0.25 秒，因为需要紧跟主人。

#### 第三步：蜘蛛的模块化 Brain——把行为分组管理

```132:201:scripts/brains/spiderbrain.lua
function SpiderBrain:OnStart()
    local pre_nodes = PriorityNode({
        BrainCommon.PanicWhenScared(self.inst, .3),
        BrainCommon.PanicTrigger(self.inst),
        BrainCommon.ElectricFencePanicTrigger(self.inst),
    })

    local post_nodes = PriorityNode({
        DoAction(self.inst, function() return InvestigateAction(self.inst) end ),
        WhileNode(function() return ... end, "IsDay",
            DoAction(self.inst, function() return GoHomeAction(self.inst) end )),
        FaceEntity(self.inst, GetTraderFn, KeepTraderFn),
        Wander(self.inst, ...),
    })

    local hider_nodes = PriorityNode({...})
    local attack_nodes = PriorityNode({...})
    local follow_nodes = PriorityNode({...})

    local root = PriorityNode({
        pre_nodes,      -- 恐慌/电击
        hider_nodes,    -- 盾蛛防御
        attack_nodes,   -- 战斗
        follow_nodes,   -- 跟随
        post_nodes,     -- 归巢/闲逛
    }, 1)

    self.bt = BT(self.inst, root)
end
```

蜘蛛 Brain 把行为分成了 5 个子 PriorityNode，然后在根 PriorityNode 中组合。**这是大型 Brain 的最佳实践**——把相关行为分组，让代码结构清晰。

---

### 12.5.7 老手进阶：六个设计经验

#### 经验一：五种模式的标准优先级

```
生存恐慌（PanicTrigger）
    ↓
战斗（ChaseAndAttack + 躲闪）
    ↓
社交（Follow + FaceEntity + Trade）
    ↓
需求（Eat + 采集）
    ↓
归巢（GoHome）
    ↓
巡逻（Leash + Wander）← 永远是最后的兜底
```

如果你的 NPC 行为异常（比如该打架的不打架、该回家的不回家），首先检查优先级顺序。

#### 经验二：Wander 是"万能兜底"

Wander 几乎永远返回 RUNNING（它一直在走或等待），所以它**必须放在 PriorityNode 的最后**。如果放在其他行为前面，后面的行为永远不会被执行。

原版所有 Brain 无一例外都把 Wander 放最后。

#### 经验三：WhileNode vs IfNode 的选择取决于"中断代价"

| 行为 | 推荐 | 理由 |
|------|------|------|
| 白天/夜晚切换 | WhileNode | 时间变化需要即时响应 |
| 战斗中的攻击/躲闪切换 | WhileNode | 战斗状态频繁变化 |
| 回家 | 看情况 | 能被中断用 WhileNode，不能用 IfNode |
| 吃东西 | IfNode | 一旦开始吃就应该吃完 |
| 帮主人干活 | IfThenDoWhileNode | 启动条件严格、维持条件宽松 |

#### 经验四：给搜索操作加缓存

FindEntity 是 AI 中最耗性能的操作。如果一个条件函数或 getactionfn 里调用了 FindEntity，而这个函数可能被每帧调用，就需要加缓存。

原版的 `BrainCommon.AnchorToSaltlick` 就是一个好榜样——`FindSaltlick` 函数缓存了结果，只在缓存失效时重新搜索。

#### 经验五：用 ChattyNode 让 AI 更有生命力

在 PriorityNode 的每个分支上包装 ChattyNode，NPC 做不同事情时说不同的话。这是成本最低的"AI 拟人化"手段。

猪人的 Brain 大量使用了这个模式：打架喊"PIG_TALK_FIGHT"、找吃的喊"PIG_TALK_FIND_MEAT"、恐慌喊"PIG_TALK_PANIC"。

#### 经验六：OnInitializationComplete 用于初始化 knownlocations

很多 Brain 需要记住"出生点"作为巡逻中心。正确的做法是在 `OnInitializationComplete` 中记录，而不是在 `OnStart` 中：

```lua
function MyBrain:OnInitializationComplete()
    self.inst.components.knownlocations:RememberLocation("home", self.inst:GetPosition())
end
```

`OnStart` 是构建行为树的地方，`OnInitializationComplete` 是 Brain 完全启动后的回调（包括 Mod 的 postinitfn 都已执行）。

---

### 12.5.8 小结

| 模式 | 核心节点 | 典型用法 |
|------|----------|----------|
| **巡逻** | Leash + Wander | 在家附近随机走动，走远了拉回来 |
| **追击** | ChaseAndAttack (+ RunAway) | 发现敌人→追过去打→打不过或跑太远放弃 |
| **逃跑** | RunAway / Panic | 有方向地远离威胁 / 无方向随机乱跑 |
| **采集** | DoAction + FindEntity | 搜索目标→走过去→执行动作 |
| **归巢** | IfNode/WhileNode + DoAction(GOHOME) | 条件满足时回家（天黑/冬天/满载） |

**下一节**（12.6）我们将讲解团队战斗 AI——TeamAttacker / TeamLeader 的协同攻击机制。

---

## 12.6 团队战斗 AI——TeamAttacker / TeamLeader 的协同攻击机制

### 本节导读

12.1-12.5 我们讲完了**单个 NPC 的决策机制**——一只猪人怎么决定打架/逃跑/回家。但有些怪物的"威慑力"不是来自单体的强度，而是来自**群体的协同**：

- 一群企鹅围攻偷蛋的玩家，一部分围着发出警告，另一部分轮流冲上来啄你
- 一群醋酸蝙蝠围着玩家盘旋，定时派几只俯冲下来抢硝石
- 它们不是各打各的，而是**有阵型、有节奏、有指令**

这种"集团军作战"的核心实现，就是本节的主角—— **TeamAttacker / TeamLeader 协同攻击系统**。

> **新手**从 12.6.1-12.6.3 起步——在游戏里观察一群企鹅的行为、理解"队员（TeamAttacker）"和"虚拟指挥官（TeamLeader）"的角色分工、看一次完整的"组队 → 围攻 → 解散"流程；**进阶读者**继续看 12.6.4-12.6.5，深入 TeamAttacker / TeamLeader 两个组件的所有方法和字段、HOLD/WARN/ATTACK 三态指令系统、圆形阵型的数学原理；**老手**跳到 12.6.6-12.6.7，了解多 leader 嵌套阵型、求救广播 BroadcastDistress、完整生命周期，六个常见陷阱与设计经验。

---

## 12.6 团队战斗 AI——TeamAttacker / TeamLeader 的协同攻击机制

#### 第一步：在游戏里观察一群企鹅

冬天的海岸线上，一群企鹅（pengull）聚在自己的"巢穴"（rookery）里产蛋。当你**走近偷蛋**时——

1. **第一阶段（个体反应）**：最近的企鹅注意到你，转向你
2. **第二阶段（组队）**：它发出"求救信号"，附近的企鹅纷纷围过来
3. **第三阶段（成阵）**：5-8 只企鹅排成**一个圆环**绕着你，**逆时针缓慢旋转**
4. **第四阶段（轮流攻击）**：每隔几秒，**1-2 只**企鹅冲出阵型扑向你，攻击完后归位
5. **第五阶段（解散）**：你跑得太远，或者所有企鹅死了，阵型崩溃

醋酸蝙蝠（acidbat）的行为也几乎一样——围着玩家盘旋、轮流派代表冲下来抢东西。

**关键问题**：这不是 5-8 只独立的怪物各打各的——它们**同步行动、轮流攻击、保持阵型**。代码层面是怎么实现的？

#### 第二步：核心思路——"虚拟指挥官"模式

最朴素的实现可能会想：让某一只企鹅成为"队长"，其他企鹅听它的。但 Klei 用了一个更优雅的设计：

> **专门生成一个"虚拟指挥官"实体**——它没有模型、没有动画、不参与战斗，只负责**"思考"和"下命令"**；真正打架的是普通企鹅。

这个虚拟指挥官就是 `teamleader` prefab：

```1:13:scripts/prefabs/teamleader.lua
local function fn()
	local inst = CreateEntity()

	inst.entity:AddTransform()

	inst:AddComponent("teamleader")
	inst:AddTag("teamleader")
    --[[Non-networked entity]]

	return inst
end

return Prefab("teamleader", fn)
```

只有一个 Transform（用于位置）+ 一个 `teamleader` 组件——什么都不渲染，连网络都不同步（`Non-networked entity`）。

#### 第三步：两个组件的分工

整个系统由**两个组件**协同工作：

| 组件 | 挂在谁身上 | 职责 |
|------|-----------|------|
| **TeamAttacker** | 普通怪物（企鹅、蝙蝠） | "我是队员"——执行指令、保持阵型、可被招募 |
| **TeamLeader** | 虚拟指挥官实体 | "我是指挥部"——招募队员、组织阵型、轮流下命令 |

```
        玩家 P  ←——— 攻击目标（threat）
         ▲
         │ 
         │ 半径 5 圆环，theta 角度旋转
         │
   ┌──── ★ ────┐
   │     │     │      ★ = 虚拟 TeamLeader（无模型，位于队员中心）
   │ ┌───┴───┐ │
  企鹅      企鹅      普通企鹅有 TeamAttacker 组件
   │ │     │ │
   └─企鹅─企鹅┘
```

> **新手记忆**：**TeamAttacker = 队员的"听令系统"，TeamLeader = 指挥部的"发令系统"。** 一个虚拟的 leader 实体被 SpawnPrefab 出来，它招募附近的同类，把它们排成圆环，然后定时下令"3 号 5 号你们俩冲！"。

---

### 12.6.2 快速入门：一次完整的"组队 → 围攻 → 解散"流程

让我们用蝙蝠（bat）的攻击流程，追踪一次完整的协同攻击：

#### 第一阶段：触发组队

**触发器**：玩家走近一只孤立的蝙蝠 A，A 在 Retarget 时找到玩家。

```54:66:scripts/prefabs/bat.lua
local function Retarget(inst)
    local ta = inst.components.teamattacker

    local newtarget = FindEntity(inst, TUNING.BAT_TARGET_DIST, IsValidTarget, nil, RETARGET_CANT_TAGS, RETARGET_ONEOF_TAGS )

    if newtarget and not ta.inteam and CanMakeOrJoinTeam(inst, newtarget) and not ta:SearchForTeam() then
        MakeTeam(inst, newtarget)
    end

    if ta.inteam and not ta.teamleader:CanAttack() then
        return newtarget
    end
end
```

A 不在团队里 → 调用 `SearchForTeam()`：

```37:47:scripts/components/teamattacker.lua
function TeamAttacker:SearchForTeam()
	local pt = self.inst:GetPosition()
	local potential_leaders = TheSim:FindEntities(pt.x, pt.y, pt.z, self.searchradius, self.teamsearchtags)

	for _, potential_leader in pairs(potential_leaders) do
		if not potential_leader.components.teamleader:IsTeamFull() then
			potential_leader.components.teamleader:NewTeammate(self.inst)
			return true
		end
    end
end
```

附近 50 格内搜索"`teamleader_<team_type>`"标签的实体——没找到（场上还没有任何 leader）。

#### 第二阶段：创建 leader

`SearchForTeam` 返回 false → 走 `MakeTeam` 路径：

```42:46:scripts/prefabs/bat.lua
local function MakeTeam(inst, attacker)
    local leader = SpawnPrefab("teamleader")
    leader.components.teamleader:SetUp(attacker, inst)
    leader.components.teamleader:BroadcastDistress(inst)
end
```

三步：
1. **`SpawnPrefab("teamleader")`** —— 在世界里生成一个虚拟指挥官
2. **`SetUp(attacker, inst)`** —— 设置 threat（攻击目标）和首个队员
3. **`BroadcastDistress(inst)`** —— "求救广播"，把附近的同类全拉进队伍

#### 第三阶段：广播求救招募队员

```156:168:scripts/components/teamleader.lua
function TeamLeader:BroadcastDistress(member)
	member = member or self.inst

	if member:IsValid() then
		local x,y,z = member.Transform:GetWorldPosition()
		local potential_teammembers = TheSim:FindEntities(x,y,z, self.searchradius, self.teamsearchtags)
		for _, potential_teammember in pairs(potential_teammembers) do
			if potential_teammember ~= member and self:ValidMember(potential_teammember) then
				self:NewTeammate(potential_teammember)
			end
		end
	end
end
```

以 A 为中心 50 格内，找所有 "team_bat" 标签的同类，逐个调用 `NewTeammate` 拉进队。每个被拉入的队员：

```135:154:scripts/components/teamleader.lua
function TeamLeader:NewTeammate(member)
	if self:ValidMember(member) then
		member.deathfn = function() self:OnLostTeammate(member) end
		member.attackedfn = function() self:BroadcastDistress(member) end
		member.attackedotherfn = function()
			self.chasetime = 0
			member.components.combat:DropTarget()
			member.components.teamattacker.orders = ORDERS.HOLD
		end

		self.team[member] = member
		self.inst:ListenForEvent("death", member.deathfn, member)
		self.inst:ListenForEvent("attacked", member.attackedfn, member)
		self.inst:ListenForEvent("onattackother", member.attackedotherfn, member)
		self.inst:ListenForEvent("onremove", member.deathfn, member)
		self.inst:ListenForEvent("onenterlimbo", member.deathfn, member)
		member.components.teamattacker.teamleader = self
		member.components.teamattacker.inteam = true
	end
end
```

队员被加入 `self.team` 表，并监听三个事件：
- `death` / `onremove` / `onenterlimbo` → 队员失踪 → 从队伍移除
- `attacked` → 队员被攻击 → 立即再次广播求救（让更多同伴加入）
- `onattackother` → 队员攻击成功 → 立即让自己进入 HOLD（归位）

#### 第四阶段：阵型旋转

`TeamLeader:OnUpdate` 每帧执行：

```322:350:scripts/components/teamleader.lua
function TeamLeader:OnUpdate(dt)
	self:ManageChase(dt)
	self:CenterLeader()
	self.lifetime = self.lifetime + dt
	self:OrganizeTeams()
	self:TeamSizeControl()

	-- Is there a target, and is the team strong enough?
	if self.threat ~= nil and self:CanAttack() then
		--Spin the formation!
		self.theta = self:GetTheta(dt)

		self:GetFormationPositions()

		if self:AllInState(ORDERS.HOLD) then
			self.timebetweenattacks = self.timebetweenattacks - dt

			if self.timebetweenattacks <= 0 then
				self.timebetweenattacks = self.attackinterval
				self:GiveOrders(ORDERS.WARN, self:NumberToAttack())
				self.inst:DoTaskInTime(0.5, function() self:GiveOrdersToAllWithOrder(ORDERS.ATTACK, ORDERS.WARN) end)
			end
		end
	end

	if not self.threat or self:IsTeamEmpty() then
		self:DisbandTeam()
	end
end
```

每帧做四件事：
1. **CenterLeader**：把虚拟 leader 移动到所有队员的中心
2. **OrganizeTeams**：如果场上有多个 leader，决定谁是"主圈"、谁是"外圈"
3. **GetTheta**：阵型角度旋转（每秒 1 弧度，反向交替）
4. **GetFormationPositions**：计算每个队员在圆环上的目标位置 → 写入队员的 `formationpos`

#### 第五阶段：轮流下令攻击

`AllInState(ORDERS.HOLD)` 表示所有队员都在归位状态——这时开始倒计时下次攻击：

```336:343:scripts/components/teamleader.lua
		if self:AllInState(ORDERS.HOLD) then
			self.timebetweenattacks = self.timebetweenattacks - dt

			if self.timebetweenattacks <= 0 then
				self.timebetweenattacks = self.attackinterval
				self:GiveOrders(ORDERS.WARN, self:NumberToAttack())
				self.inst:DoTaskInTime(0.5, function() self:GiveOrdersToAllWithOrder(ORDERS.ATTACK, ORDERS.WARN) end)
			end
		end
```

每隔 `attackinterval` 秒（默认 3 秒）：
1. 随机选 N 只（默认 1-2）队员，给它们下 **WARN（警告）** 指令
2. 0.5 秒后，把所有 WARN 状态的队员升级成 **ATTACK（攻击）** 指令

队员怎么响应这些指令？看 TeamAttacker 的 OnUpdate：

```90:114:scripts/components/teamattacker.lua
function TeamAttacker:OnUpdate(dt)
	if self:ShouldGoHome() then self:LeaveTeam() end

	if self.teamleader and self.teamleader:CanAttack() then --did you find a team?
		local has_warn_orders = (self.orders == ORDERS.WARN)
		if not self.orders or has_warn_orders or self.orders == ORDERS.HOLD then --if you don't have anything to do.. look menacing
			self.inst.components.combat:DropTarget()
			if self.formationpos then
				local destpos = self.formationpos
                if destpos and not self.ignoreformation then
                    local mypos = self.inst:GetPosition()
                    if distsq(destpos, mypos) >= 0.15 then	--if you're almost at your target just stop.
                        self.inst.components.locomotor:GoToPoint(self.formationpos)
                    end
                end

				if not has_warn_orders and self.inst.components.health.takingfiredamage then
					self.orders = ORDERS.ATTACK
				end
			end
		elseif self.orders == ORDERS.ATTACK then	--You have been told to attack. Get the target from your leader.
			self.inst.components.combat:SuggestTarget(self.teamleader.threat)
		end
	end
end
```

- **HOLD / WARN / 无指令**：放弃自己的 target（`combat:DropTarget`），走向 `formationpos`（保持阵型）
- **ATTACK**：通过 `combat:SuggestTarget(teamleader.threat)` 攻击 leader 指定的目标

#### 第六阶段：解散

队伍解散有三种方式：

1. **威胁消失**：threat（玩家）死亡或被 Remove → `_onthreatremoved` 回调 → `DisbandTeam()`
2. **追击超时**：`chasetime` 累积超过 `maxchasetime`（默认 30 秒）→ `DisbandTeam()`
3. **队伍空了**：所有队员都死了/逃了 → `DisbandTeam()`

```114:121:scripts/components/teamleader.lua
function TeamLeader:DisbandTeam()
	for member in pairs(self.team) do
		self:OnLostTeammate(member)
	end
	--assert(next(self.team) == nil)
	self.threat = nil
	self.inst:Remove()
end
```

DisbandTeam 把所有队员逐个 OnLostTeammate（清空 leader 引用、移除事件监听），最后**虚拟 leader 自我销毁**（`self.inst:Remove()`）——因为它的使命完成了。

> **新手记忆**：**孤立怪物 → SearchForTeam 找队 → 找不到就 SpawnPrefab("teamleader") → leader.BroadcastDistress 拉同伴 → 阵型旋转 → 每 3 秒选 1-2 只 WARN → 0.5 秒后升级 ATTACK → 攻完归位 → 威胁消失 → DisbandTeam → leader 自杀。**

---

### 12.6.3 快速入门：HOLD / WARN / ATTACK 三态指令

队员的"心理状态"由 `self.orders` 字段表示，取值来自全局常量 `ORDERS`：

```2413:2419:scripts/constants.lua
ORDERS =
{
    NONE = 0,
    HOLD = 1,
    WARN = 2,
    ATTACK = 3,
}
```

| 状态 | 字面含义 | 队员行为 | 谁会变成这个状态 |
|------|---------|---------|----------------|
| **NONE** | 无指令 | 走向 formationpos（同 HOLD） | 刚加入团队、被攻击后等 |
| **HOLD** | "归位" | DropTarget + 走向 formationpos | 攻击完成后（onattackother）、leader 主动指派 |
| **WARN** | "警告" | DropTarget + 走向 formationpos（往内缩 1 格） | leader 在 GiveOrders 阶段随机选中 |
| **ATTACK** | "进攻" | SuggestTarget(leader.threat) | WARN 0.5 秒后升级 |

**WARN 的视觉意义**：在 `GetFormationPositions` 中，WARN 状态的队员会被往圆心方向移动 1 格——给玩家一个"这只蝙蝠正瞄准我，要冲过来了！"的视觉提示。

```205:219:scripts/components/teamleader.lua
function TeamLeader:GetFormationPositions()
    local target, theta = self.threat, self.theta
    local radius
    local pt = target:GetPosition()
    local steps = self:GetTeamSize()
	local step_decrement = (TWOPI / steps)

    for member in pairs(self.team) do
        radius = self.radius - ((member.components.teamattacker.orders == ORDERS.WARN and 1) or 0)

        local offset = Vector3(radius * math.cos(theta), 0, -radius * math.sin(theta))
        member.components.teamattacker.formationpos = pt + offset
        theta = theta - step_decrement
    end
end
```

**指令流转图**：

```
              [新加入]
                 │
                 ▼
              NONE/HOLD ──────────► (默认状态：归位)
                 │
                 │ leader.GiveOrders 选中
                 ▼
                WARN ──────────────► (圆环往内缩1格，预示要冲)
                 │
                 │ 0.5s 后 GiveOrdersToAllWithOrder
                 ▼
                ATTACK ────────────► (扑向 leader.threat)
                 │
                 │ onattackother 事件触发（attackedotherfn）
                 ▼
                HOLD ──────────────► (归位)
```

**为什么要分两步（先 WARN 再 ATTACK）**？这是一个"**预告攻击**"的设计——给玩家 0.5 秒的反应时间看到"哪几只要冲了"，让玩家有机会闪避。这种"telegraphed attack"是良好游戏设计的标志。

---

### 12.6.4 进阶：TeamAttacker 组件全字段与方法详解

#### 第一步：构造函数

```13:29:scripts/components/teamattacker.lua
local TeamAttacker = Class(function(self, inst)
	self.inst = inst
	self.inteam = false
	--self.teamleader = nil
	--self.formationpos = nil
	--self.order = nil
    --self.ignoreformation = nil
	--self.validmemberfn = nil
	self.searchradius = 50
	self.leashdistance = 70
	self.inst:StartUpdatingComponent(self)
	self.team_type = "monster"
end,
nil,
{
    team_type = onteamtype,
})
```

**所有字段**：

| 字段 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `inst` | EntityScript | - | 自身（队员实体） |
| `inteam` | boolean | false | 是否已在某个队伍中 |
| `teamleader` | teamleader 组件/nil | nil | 当前所属的虚拟指挥官的组件引用 |
| `formationpos` | Vector3/nil | nil | 当前帧应当走向的阵型位置（leader 计算） |
| `orders` | ORDERS 常量/nil | nil | 当前指令（HOLD/WARN/ATTACK/nil） |
| `ignoreformation` | boolean/nil | nil | 是否忽略阵型（脱队但仍在团队中） |
| `validmemberfn` | function/nil | nil | 自定义"是否可加入"判断 |
| `searchradius` | number | 50 | 搜索 leader 时的范围 |
| `leashdistance` | number | 70 | 离家超过此距离自动脱队 |
| `team_type` | string | "monster" | 团队类型（"bat"/"penguin"/...） |

#### 第二步：`team_type` 字段的标签机制

`team_type` 字段非常关键——它决定了"哪些怪物可以组成同一个队伍"。注意它**带有 setter 钩子**：

```1:11:scripts/components/teamattacker.lua
local function onteamtype(self, team, oldteam)
    if oldteam ~= nil then
        self.inst:RemoveTag("team_"..oldteam)
    end
    if team ~= nil then
        self.inst:AddTag("team_"..team)
		self.teamsearchtags = {"teamleader_"..team}
	else
		self.teamsearchtags = nil
	end
end
```

设置 `team_type = "bat"` 时：
1. 实体加上 `"team_bat"` 标签 —— 表示"我是 bat 团队的潜在成员"
2. 设置 `teamsearchtags = {"teamleader_bat"}` —— 用于搜索同类型 leader

**对应的 leader 端**：

```1:13:scripts/components/teamleader.lua
local function onteamtype(self, team, oldteam)
    if oldteam ~= nil then
        self.inst:RemoveTag("teamleader_"..oldteam)
    end
    if team ~= nil then
        self.inst:AddTag("teamleader_"..team)
		self.teamleadersearchtags = {"teamleader_"..team}
		self.teamsearchtags = {"team_"..team}
    else
		self.teamleadersearchtags = nil
		self.teamsearchtags = nil
    end
end
```

leader 加上 `"teamleader_bat"` 标签 —— 让其他蝙蝠能 `FindEntities` 找到它。

**这样设计的好处**：bat 团队和 penguin 团队完全隔离，蝙蝠不会被企鹅 leader 招募。

#### 第三步：核心方法

| 方法 | 作用 |
|------|------|
| `SearchForTeam()` | 搜索附近的同类 leader 并加入。成功返回 true |
| `LeaveTeam()` | 主动脱队（leader 的 OnLostTeammate） |
| `LeaveFormation() / JoinFormation()` | 临时忽略阵型 / 重新加入阵型 |
| `GetOrders()` | 获取当前指令 |
| `SetValidMemberFn(fn)` | 设置"是否可加入"判断函数（被 leader.ValidMember 调用） |
| `ShouldGoHome()` | 离家超过 leashdistance 时返回 true |
| `OnEntitySleep` / `OnEntityWake` | 离屏时停止更新组件、回屏时恢复 |

#### 第四步：实战代码——OnUpdate 的"双模式"

```90:114:scripts/components/teamattacker.lua
function TeamAttacker:OnUpdate(dt)
	if self:ShouldGoHome() then self:LeaveTeam() end

	if self.teamleader and self.teamleader:CanAttack() then --did you find a team?
		local has_warn_orders = (self.orders == ORDERS.WARN)
		if not self.orders or has_warn_orders or self.orders == ORDERS.HOLD then
			self.inst.components.combat:DropTarget()
			if self.formationpos then
				...
				self.inst.components.locomotor:GoToPoint(self.formationpos)
				...
			end
		elseif self.orders == ORDERS.ATTACK then
			self.inst.components.combat:SuggestTarget(self.teamleader.threat)
		end
	end
end
```

**关键设计**：TeamAttacker 的 OnUpdate **直接操作 locomotor 和 combat**，绕过 Brain。这意味着——
- 在团队中的怪物，**Brain 的 ChaseAndAttack 等行为节点会被这层逻辑"覆盖"**
- 队员不主动选 target，而是被动接受 leader 的 threat
- 队员的移动也不归 Brain 管，而是 OnUpdate 直接 `GoToPoint(formationpos)`

**Brain 中的"团队感知"**：在 PenguinBrain 中你会看到大量这样的判断：

```157:166:scripts/brains/penguinbrain.lua
local function GetMigrateLeashPos(inst)
    if inst.components.teamattacker.teamleader or inst.components.combat.target then
        return nil
    end
    local homePos = inst.components.knownlocations and
        (inst.components.knownlocations:GetLocation("rookery") or
        inst.components.knownlocations:GetLocation("home"))
    return homePos
end
```

"如果在团队中，就不要执行 Migrate Leash"——把团队行为和 Brain 行为协调起来。

---

### 12.6.5 进阶：TeamLeader 组件全字段与圆形阵型数学

#### 第一步：构造函数与字段

```15:41:scripts/components/teamleader.lua
local TeamLeader = Class(function(self, inst )

	self.inst = inst
	self.team_type = "monster"
	self.min_team_size = 3
	self.max_team_size = 6
	self.team = {}
	self.threat = nil
	self.searchradius = 50
	self.theta = 0
	self.thetaincrement = 1
	self.radius = 5
	self.reverse = false
	self.timebetweenattacks = 3
	self.attackinterval = 3
	self.inst:StartUpdatingComponent(self)
	self.lifetime = 0
    self.attack_grp_size = nil
    self.chk_state = true

	self.maxchasetime = 30
	self.chasetime = 0
end,
nil,
{
    team_type = onteamtype,
})
```

**关键字段**：

| 字段 | 默认 | 含义 |
|------|------|------|
| `team` | `{}` | 队员表（key=member, value=member） |
| `threat` | nil | 攻击目标（玩家） |
| `min_team_size` | 3 | 至少 3 个队员才开始组织攻击（CanAttack 判断） |
| `max_team_size` | 6 | 最多 6 个队员 |
| `searchradius` | 50 | 招募搜索范围 |
| `radius` | 5 | 圆环半径 |
| `theta` | 0 | 当前旋转角度（弧度） |
| `thetaincrement` | 1 | 每秒角速度（弧度/秒） |
| `reverse` | false | 是否反向旋转 |
| `timebetweenattacks` | 3 | 距离下次"派攻击"的剩余时间 |
| `attackinterval` | 3 | 两次"派攻击"的间隔（每次重置） |
| `attack_grp_size` | nil | 每次派几个去攻击（nil 时随机 1-2） |
| `maxchasetime` | 30 | 最大追击时间，超时解散 |
| `chk_state` | true | AllInState 是否考虑 frozen / burning |

#### 第二步：圆形阵型的数学

`GetFormationPositions` 的核心是把 N 个队员均匀分布在圆周上：

```205:219:scripts/components/teamleader.lua
function TeamLeader:GetFormationPositions()
    local target, theta = self.threat, self.theta
    local radius
    local pt = target:GetPosition()
    local steps = self:GetTeamSize()
	local step_decrement = (TWOPI / steps)

    for member in pairs(self.team) do
        radius = self.radius - ((member.components.teamattacker.orders == ORDERS.WARN and 1) or 0)

        local offset = Vector3(radius * math.cos(theta), 0, -radius * math.sin(theta))
        member.components.teamattacker.formationpos = pt + offset
        theta = theta - step_decrement
    end
end
```

**数学步骤**：
1. **N 等分圆周**：`step_decrement = 2π / N`，每个队员相邻 `2π/N` 弧度
2. **极坐标转笛卡尔**：`x = radius * cos(theta)`，`z = -radius * sin(theta)`
3. **WARN 队员往内缩**：`radius = self.radius - 1`，让"准备冲锋"的队员更靠近玩家
4. **整体平移**：`pt + offset`，圆心是 threat（玩家）

**旋转逻辑**：

```291:294:scripts/components/teamleader.lua
function TeamLeader:GetTheta(dt)
	local direction = (self.reverse and -1) or 1
	return self.theta + (direction * dt * self.thetaincrement)
end
```

每帧 theta 增加 `dt * thetaincrement`，整圈 6.28 秒（thetaincrement=1 时）。

#### 第三步：`AllInState` 的"等齐了再打"机制

```253:267:scripts/components/teamleader.lua
function TeamLeader:AllInState(state)
    for member in pairs(self.team) do
        if not (self.chk_state
				and (member:HasTag("frozen")
					or (member.components.burnable ~= nil
						and member.components.burnable:IsBurning())
					)
				) and
            not (member.components.teamattacker.orders == nil
				or member.components.teamattacker.orders == state) then
            return false
        end
    end
    return true
end
```

**只有所有"健康（未冻结、未燃烧）"队员都进入指定状态时返回 true**。这保证了攻击节奏的整齐——上一波攻击结束、所有队员都归位（HOLD）后，才会启动下一波。

如果 `chk_state` 为 false，被冻结/燃烧的队员也会被纳入判断（适用于不希望"被冻队员卡住攻击节奏"的场景）。

#### 第四步：`GiveOrders` 的随机派遣

```221:243:scripts/components/teamleader.lua
function TeamLeader:GiveOrders(order, num)
	local temp = {}

	for member in pairs(self.team) do
		member.components.teamattacker.orders = nil
		table.insert(temp, member)
	end

	num = math.min(num, #temp)

	local successfulorders = 0
	while successfulorders < num do
		local attempt = temp[math.random(#temp)]
		if attempt.components.teamattacker.orders == nil then
			attempt.components.teamattacker.orders = order
			successfulorders = successfulorders + 1
		end
	end

	for member in pairs(self.team) do
		member.components.teamattacker.orders = member.components.teamattacker.orders or ORDERS.HOLD
	end
end
```

**三步**：
1. **清空所有指令**：`member.orders = nil`
2. **随机抽 N 个**：用 `math.random(#temp)` 抽，重复直到 N 个不同队员被设置
3. **未中签的全部 HOLD**：剩下的队员状态变为 HOLD（继续归位转圈）

`NumberToAttack` 决定每次派几个：

```300:305:scripts/components/teamleader.lua
function TeamLeader:NumberToAttack()
	return (type(self.attack_grp_size) == "function" and self.attack_grp_size())
		or (type(self.attack_grp_size) == "number" and self.attack_grp_size)
		or (math.random() > 0.25 and 1)
		or 2
end
```

如果没设置 `attack_grp_size`，75% 概率派 1 个，25% 概率派 2 个。

#### 第五步：CanAttack——队伍达标判断

```187:189:scripts/components/teamleader.lua
function TeamLeader:CanAttack()
	return self:GetTeamSize() >= self.min_team_size
end
```

只有队伍人数达到 `min_team_size`（默认 3）时，CanAttack 返回 true。**TeamAttacker 也会检查这个**——人数不足时队员不响应阵型，先继续单干（如蝙蝠会用 ChaseAndAttack）。

#### 第六步：SetUp——便捷的"队伍初始化"

```53:60:scripts/components/teamleader.lua
function TeamLeader:SetUp(target, first_member)
    self:SetNewThreat(target)
    local teamattacker = first_member.components.teamattacker
    if teamattacker then
        self.team_type = teamattacker.team_type
        self:NewTeammate(first_member)
    end
end
```

一行调用完成"设置威胁 + 复制 team_type + 加入第一个队员"。蝙蝠的 MakeTeam 用的就是这个。

---

### 12.6.6 老手进阶：多 leader 嵌套阵型——OrganizeTeams

如果场上**同时有两群蝙蝠攻击同一个玩家**，会怎么样？答案是—— **它们会自动组成同心圆阵型！**

```62:98:scripts/components/teamleader.lua
local function teamleader_sort(t1, t2)
	return t1.components.teamleader.lifetime > t2.components.teamleader.lifetime
end
function TeamLeader:OrganizeTeams()
	local teams = nil
	local x,y,z = self.inst.Transform:GetWorldPosition()
	local ents = TheSim:FindEntities(x,y,z, self.searchradius, self.teamleadersearchtags)
	for _, potential_member in pairs(ents) do
		if potential_member.components.teamleader and potential_member.components.teamleader.threat == self.threat then
			teams = teams or {}
			table.insert(teams, potential_member)
		end
	end

	if not teams then return end

	table.sort(teams, teamleader_sort)

	if teams[1] ~= self.inst then return end

	local radius = 5
	local reverse = false
	local thetaincrement = 1
	local maxteam = 6

	for _, v in pairs(teams) do
		local teamleader = v.components.teamleader
		teamleader.radius = radius
		teamleader.reverse = reverse
		teamleader.thetaincrement = thetaincrement
		teamleader.max_team_size = maxteam
		radius = radius + 5
		reverse = not reverse
		thetaincrement = thetaincrement * 0.6
		maxteam = maxteam + 6
	end
end
```

**算法**：
1. 找附近所有"攻击同一个 threat"的同类型 leader
2. 按 lifetime 排序（最早组建的排第一）
3. **只有最年长的 leader 执行后续逻辑**（避免重复执行）
4. 给每个 leader 分配不同的圆环参数：

| 圈层 | radius | reverse | thetaincrement | maxteam |
|------|--------|---------|---------------|---------|
| 内圈 | 5 | false（顺时针） | 1.0 | 6 |
| 中圈 | 10 | true（逆时针） | 0.6 | 12 |
| 外圈 | 15 | false（顺时针） | 0.36 | 18 |
| ... | +5 | 反向交替 | ×0.6 | +6 |

**视觉效果**：双层蝙蝠群同时围攻——内圈快速顺时针、外圈缓慢逆时针，如同两道相反方向的旋涡。

**为什么 thetaincrement 要随圈层递减**？因为外圈周长更大，相同角速度下线速度太快——递减后所有圈层的视觉移动速度趋近一致。

---

### 12.6.7 老手进阶：完整生命周期与"求救链式反应"

#### 第一步：求救广播的连锁反应

`BroadcastDistress` 有一个隐藏的强大特性——**触发链**。看 NewTeammate 注册的 `attackedfn`：

```138:138:scripts/components/teamleader.lua
		member.attackedfn = function() self:BroadcastDistress(member) end
```

只要任何一个**已在队中**的队员被攻击（attacked 事件），它就**以自己为中心再次广播**。这意味着——

**典型场景**：你在 50 格内的蝙蝠 A 处招了团队，团队中只有 A、B、C 三只。你打了 B，B 触发 BroadcastDistress——以 B 的位置为中心 50 格内再找。如果有蝙蝠 D 在 B 附近 50 格但不在 A 附近 50 格，D 也会被拉进来。

**一次又一次**：D 加入后被攻击 → 以 D 的位置广播 → 拉来 E、F... 这就是**蝙蝠"波次涌来"**的代码原理。

**实战意义**：
- 单个怪物的 `searchradius` 是 50，但通过链式 BroadcastDistress，团队招募范围实际上可以扩散到任意距离
- 玩家在洞穴里被一群蝙蝠围攻时，往一个方向跑，会不断有"新一波"蝙蝠从远处赶来——就是这个机制

#### 第二步：完整生命周期

```
[条件] 单个怪物 Retarget 找到玩家
     │
     ▼
[1] 怪物 SearchForTeam → 找到 leader 加入
     │ │
     │ └─失败──▶ MakeTeam → SpawnPrefab("teamleader") → SetUp + BroadcastDistress
     ▼
[2] leader.team 中加入 N 个队员（监听 5 个事件 / 设置 inteam=true）
     │
     ▼
[3] leader.OnUpdate 每帧：
       ├─ ManageChase（chasetime 累加）
       ├─ CenterLeader（位置=队员中心）
       ├─ OrganizeTeams（多 leader 同心圆）
       ├─ TeamSizeControl（裁掉超员）
       ├─ GetTheta（旋转）
       ├─ GetFormationPositions（计算队员阵型坐标）
       └─ if AllInState(HOLD) and timebetweenattacks<=0:
              GiveOrders(WARN, NumberToAttack())
              0.5s 后 → GiveOrdersToAllWithOrder(ATTACK, WARN)
     │
     ▼
[4] 队员 OnUpdate 响应：
       ├─ ShouldGoHome → LeaveTeam
       ├─ HOLD/WARN/无指令 → DropTarget + locomotor:GoToPoint(formationpos)
       └─ ATTACK → combat:SuggestTarget(leader.threat)
     │
     ▼
[5a] 队员 onattackother：
        chasetime = 0
        DropTarget
        orders = HOLD
[5b] 队员 attacked：
        BroadcastDistress(member)（链式招募）
[5c] 队员 death/onremove/onenterlimbo：
        OnLostTeammate → 移除监听 + inteam=false
     │
     ▼
[6] 解散（DisbandTeam）：
       ├─ threat 被 Remove
       ├─ chasetime > maxchasetime
       └─ team 空了
     │
     ▼
[7] OnLostTeammate(每个队员) → leader 实体 inst:Remove()
```

#### 第三步：实战 Mod——给自定义生物加上协同攻击

假设你做了一个"狼群"的 Mod，想让狼自动协同攻击。基础步骤：

**第一步：给 wolf prefab 加组件**

```lua
local function wolf_fn()
    local inst = CreateEntity()
    -- ...其他组件...
    
    inst:AddTag("monster")
    inst:AddTag("wolf")
    
    inst:AddComponent("teamattacker")
    inst.components.teamattacker.team_type = "wolf"
    inst.components.teamattacker.searchradius = 30  -- 狼群间距更近
    inst.components.teamattacker.leashdistance = 50
    
    inst:AddComponent("combat")
    -- ...
end
```

**第二步：在 Retarget 中触发组队**

```lua
local function MakeWolfTeam(inst, attacker)
    local leader = SpawnPrefab("teamleader")
    leader.components.teamleader:SetUp(attacker, inst)
    -- 自定义参数
    leader.components.teamleader.radius = 4         -- 圆环半径小一些
    leader.components.teamleader.min_team_size = 2  -- 2 只就够攻击
    leader.components.teamleader.max_team_size = 5
    leader.components.teamleader.attackinterval = 2  -- 攻击节奏更快
    leader.components.teamleader.maxchasetime = 60   -- 追更久
    leader.components.teamleader:SetAttackGrpSize(2) -- 每次派 2 只
    leader.components.teamleader:BroadcastDistress(inst)
end

local function WolfRetarget(inst)
    local ta = inst.components.teamattacker
    local newtarget = FindEntity(inst, 20, function(guy)
        return inst.components.combat:CanTarget(guy)
    end, nil, nil, {"character"})

    if newtarget and not ta.inteam and not ta:SearchForTeam() then
        MakeWolfTeam(inst, newtarget)
    end

    if ta.inteam and not ta.teamleader:CanAttack() then
        return newtarget  -- 队伍人数不够，单干
    end
end
```

**第三步：在 Brain 中处理团队感知**

```lua
function WolfBrain:OnStart()
    local root = PriorityNode({
        BrainCommon.PanicTrigger(self.inst),
        
        -- 团队人数不够时，单独追击
        WhileNode(function() 
            local ta = self.inst.components.teamattacker
            return not ta.inteam or not ta.teamleader:CanAttack()
        end, "SoloChase",
            ChaseAndAttack(self.inst, 10, 30)),
        
        -- 在团队中时，TeamAttacker.OnUpdate 接管，Brain 只兜底闲逛
        Wander(self.inst, ...),
    }, 0.5)
    
    self.bt = BT(self.inst, root)
end
```

**注意要点**：
- TeamAttacker 的 OnUpdate **会 DropTarget 并 GoToPoint(formationpos)**——所以在团队中的怪物，Brain 的 ChaseAndAttack 会被覆盖，这是正确的设计
- 在 Brain 中用 `inst.components.teamattacker.teamleader == nil` 或 `not inst.components.teamattacker:CanAttack()` 来判断"是否独自行动"

---

### 12.6.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：忘记设置 `team_type`，所有团队混在一起

```lua
-- 错误：没设 team_type，使用默认值 "monster"
inst:AddComponent("teamattacker")
-- 默认 team_type = "monster"
```

**问题**：默认 team_type 是 "monster"。如果场上有多种使用 TeamAttacker 但都没设置 team_type 的怪物，它们会**互相招募对方**——蝙蝠和企鹅可能站在同一个圆圈里。

**正确做法**：每种生物都给独立的 team_type：

```lua
inst.components.teamattacker.team_type = "wolf"
```

#### 陷阱 2：在 Retarget 里没检查 `CanMakeOrJoinTeam`，导致着火/恐慌的怪物加入团队

```lua
-- 错误：只要找到目标就组队
local function Retarget(inst)
    local ta = inst.components.teamattacker
    local newtarget = FindEntity(inst, ...)
    if newtarget and not ta.inteam and not ta:SearchForTeam() then
        MakeTeam(inst, newtarget)
    end
end
```

**问题**：如果蝙蝠正在着火（应当恐慌乱跑），但 Retarget 时仍然组队，TeamAttacker.OnUpdate 会让它走向 formationpos——**与"恐慌乱跑"行为冲突**。

**正确做法**：参考蝙蝠的实现：

```35:41:scripts/prefabs/bat.lua
local function IsValidMember(inst)
    return not BrainCommon.ShouldTriggerPanic(inst) and not BrainCommon.ShouldAvoidElectricFence(inst)
        and not (inst.components.burnable and inst.components.burnable:IsBurning())
end
local function CanMakeOrJoinTeam(inst, attacker)
    return IsValidMember(inst)
end
```

并在 Retarget 中调用 `CanMakeOrJoinTeam`，同时用 `SetValidMemberFn` 注册到 TeamAttacker：

```272:274:scripts/prefabs/bat.lua
    local teamattacker = inst:AddComponent("teamattacker")
    teamattacker:SetValidMemberFn(IsValidMember)
    teamattacker.team_type = "bat"
```

这样 leader 在 ValidMember 检查时也会调用这个函数。

#### 陷阱 3：Brain 忽略团队状态——队员在 Brain 中乱跑

```lua
-- 错误：Brain 没考虑 teamattacker
function MyBrain:OnStart()
    local root = PriorityNode({
        ChaseAndAttack(self.inst, 10, 30),  -- 即使在团队中也追击
        Wander(self.inst, ...),
    }, 0.5)
end
```

**问题**：Brain 的 ChaseAndAttack 和 TeamAttacker.OnUpdate 会争夺 locomotor 控制权——一会儿走向 formationpos，一会儿走向 target，怪物表现得很抽搐。

**正确做法**：在 Brain 中用 WhileNode 包装 ChaseAndAttack，仅在"未在团队"或"团队不能攻击"时才执行单独追击：

```lua
WhileNode(function() 
    local ta = self.inst.components.teamattacker
    return not ta.teamleader or not ta.teamleader:CanAttack()
end, "SoloChase",
    ChaseAndAttack(self.inst, 10, 30)),
```

#### 陷阱 4：threat 是非网络实体（TeamLeader）但被 ListenForEvent

```lua
-- 错误：把 leader 实体当作 threat
local leader = SpawnPrefab("teamleader")
leader.components.teamleader:SetNewThreat(some_other_leader)
```

**问题**：`SetNewThreat` 内部会监听 threat 的 onremove 事件。如果 threat 是非网络实体或在客户端不存在，可能会出错。

**正确做法**：threat 应当是真实的、可被攻击的实体（玩家、其他怪物）。

#### 陷阱 5：没有使用 `OnEntitySleep` 适配休眠机制

TeamAttacker 内置了离屏优化：

```49:58:scripts/components/teamattacker.lua
function TeamAttacker:OnEntitySleep()
	if self.teamleader then
		self.teamleader:OnLostTeammate(self.inst)
	end
	self.inst:StopUpdatingComponent(self)
end

function TeamAttacker:OnEntityWake()
	self.inst:StartUpdatingComponent(self)
end
```

**注意**：当队员离屏（被 EntitySleep）时，**它会自动从团队中脱离**。这避免了"远处的队员还在阵型里旋转"的问题——但意味着**只有玩家屏幕附近的怪物会形成团队**。

如果你的 Mod 重写了 OnEntitySleep 但没调用 super，会破坏这个机制。

#### 陷阱 6：在团队组件中直接修改其他实体的状态——网络同步陷阱

```lua
-- 错误：跨实体修改可能影响客户端预测
function MyTeamLeader:CustomLogic()
    for member in pairs(self.team) do
        member.components.health:SetVal(100)  -- 直接改血量
    end
end
```

**问题**：TeamLeader 是非网络实体，但队员是网络实体。从一个非网络实体跨实体修改网络实体的状态时，要确保只在服务端执行（这通常没问题，因为 TeamLeader 只在服务端 Spawn）。但如果队员是其他玩家的 follower 或被托管实体，仍然要小心。

**安全做法**：用 `inst:PushEvent` 或调用接受性的接口（`combat:SuggestTarget`），而不是直接 SetVal。

#### 设计经验三条

**经验一：TeamLeader 是"虚拟单元"模式的范例**

TeamLeader 的设计体现了一个重要架构思想—— **"逻辑实体"与"渲染实体"分离**。TeamLeader 本身没有模型、不消耗渲染资源、不参与战斗判定——它只是一个"在世界里漂浮的状态机"。

**这个模式还可以用于**：
- 任务管理器（一个不可见实体管理一群 NPC 的任务）
- 战场管理器（控制一场战斗的节奏、玩家/敌方人数动态平衡）
- 群体行为协调器（一群鸟一起转向、一群鱼形成鱼群）

**经验二：`team_type` 的标签机制是 ECS 思想的体现**

通过给实体加 `team_<type>` / `teamleader_<type>` 标签，搜索时只需要 `FindEntities(..., teamsearchtags)` —— 完全不需要遍历全局列表，C++ 引擎层用空间分割快速过滤。这是 ECS（实体-组件-系统）模式的优雅应用：**不要去 `if-elseif-else` 判断对方是什么；让对方用标签自报家门**。

**经验三：协同 AI 不一定要写在 Brain 里**

整个 TeamAttacker / TeamLeader 系统**几乎完全绕过了行为树**——队员的移动和攻击都是在组件的 OnUpdate 里直接操作 locomotor / combat。Brain 只负责"我现在不在团队中应该干嘛"的兜底。

**这给我们的启示**：当一个行为非常专精、状态切换频繁、需要跨实体协调时，**直接写在组件里比写在 Brain 里更清晰**。Brain 适合"从外部看"的决策（要不要开打、要不要回家），组件适合"从内部看"的执行（具体怎么走、怎么瞄准）。

---

### 12.6.9 小结

| 概念 | 一句话总结 |
|------|-----------|
| **TeamAttacker** | 队员的"听令系统"，OnUpdate 直接操作 locomotor/combat |
| **TeamLeader** | 虚拟指挥官的组件，OnUpdate 计算阵型、下指令、解散 |
| **teamleader prefab** | 非网络的虚拟实体，只有 Transform + teamleader 组件 |
| **team_type** | 通过标签 `team_<type>` / `teamleader_<type>` 实现团队隔离 |
| **searchradius** | 50 格——招募和搜索的默认范围 |
| **HOLD / WARN / ATTACK** | 三态指令系统，WARN→ATTACK 间隔 0.5 秒做"预告攻击" |
| **GetFormationPositions** | 极坐标计算队员在圆环上的位置，WARN 队员往内缩 1 格 |
| **CanAttack / min_team_size** | 默认队伍 ≥3 人才开始组织攻击 |
| **BroadcastDistress** | 链式求救——队员被攻击会再次广播，可拉来远处同伴 |
| **OrganizeTeams** | 多 leader 同心圆——内外圈 radius、reverse、thetaincrement 交替递减 |
| **DisbandTeam** | 三种解散条件：threat 消失 / 超时 / 队伍空 |
| **OnEntitySleep** | 离屏队员自动脱队——让团队总是聚焦在玩家屏幕附近 |

**下一节**（12.7）我们将讲解寻路系统——PathFinder API 与可行走区域判定。

## 12.7 寻路系统——PathFinder API 与可行走区域判定

### 本节导读

12.1-12.6 我们讲完了 NPC 怎么"决定做什么"和"协同攻击"。但还有一个最基础的问题没解决：**NPC 决定走过去之后，怎么真的走过去？**

- 玩家在墙后面，蜘蛛要怎么绕墙过来？
- 海岛之间有水，普通陆地怪物走不到对面，蜜蜂可以飞过去——AI 怎么知道？
- 一只猪人想去吃远处的肉，中间有一片蛛网（creep），它该绕路还是穿过？

这些问题的答案，全部藏在饥荒的**寻路系统**里——位于 `TheWorld.Pathfinder`（C++ 实现）和 `LocoMotor` 组件中。本节我们把这个"看不见但永远在跑"的系统拆开看。

> **新手**从 12.7.1-12.7.3 起步——理解"地图是网格，PathFinder 是 A* 搜索器"、`IsClear` 直线检查、最常用的判定函数 `Map:IsPassableAtPoint`；**进阶读者**继续看 12.7.4-12.7.5，深入 LocoMotor 怎么把 GoToPoint 翻译成异步寻路、`pathcaps` 能力字段如何决定"哪些格子能走"、动态墙的 `AddWall`/`RemoveWall`；**老手**跳到 12.7.6-12.7.7，掌握扇形搜索 `FindValidPositionByFan`、跨水跳台机制 `StaticHoppablePlatform`、寻路系统的性能与同步问题，六个常见陷阱与设计经验。

---

## 12.7 寻路系统——PathFinder API 与可行走区域判定


#### 第一步：饥荒的世界是网格的

虽然玩家看到的世界是连续的（移动用浮点坐标），但**底层的地图是 16×16 像素的网格**——每个格子（tile）是一个独立单元，有自己的"地形类型"：草地、岩石、海洋、悬崖……

打开任意带寻路的代码，你会看到这样的 API：

```1155:1156:scripts/components/locomotor.lua
        pathtile_x, pathtile_y = ground.Pathfinder:GetPathTileIndexFromPoint(self.inst.Transform:GetWorldPosition())
        tile_x, tile_y = ground.Map:GetTileCoordsAtPoint(self.inst.Transform:GetWorldPosition())
```

`GetPathTileIndexFromPoint` 把世界坐标 (x, y, z) 转成网格索引 (tile_x, tile_y)。每次寻路前的"对齐"都靠这个。

#### 第二步：PathFinder 是什么

`TheWorld.Pathfinder` 是世界实体上的一个 **C++ 组件**（不是 Lua 组件），全局唯一。它的职责：

1. 维护一张**导航网格**——记录每个格子的"通行属性"
2. 提供 **`IsClear`**——快速直线检查（A 到 B 之间无障碍？）
3. 提供 **`SubmitSearch` / `GetSearchStatus` / `GetSearchResult`**——异步 A* 寻路
4. 提供 **`AddWall` / `RemoveWall`**——运行时动态修改导航网格
5. 提供 **`GetPathTileIndexFromPoint`**——坐标转换

**为什么是异步**？A* 算法在最坏情况下要扫描整张地图，几百格的搜索可能耗时几毫秒。如果同步执行，多只怪物同时寻路会卡顿。所以 PathFinder 把搜索请求放进 C++ 端的队列，每帧分摊执行。

#### 第三步：游戏中的可见演示

在游戏里按 `~` 打开控制台输入：

```lua
c_select()  -- 选中你最近的怪物
local pig = c_sel()
print(pig.components.locomotor:GetDebugString())
-- 输出类似：RUN, (6.00) [pos: x,y,z] [...] (15, 23):(15, 23) +/-0.00
-- 第一个 (15, 23) 是 Map tile，第二个 (15, 23) 是 PathFinder tile
```

`GetPathTileIndexFromPoint` 返回的就是 PathFinder 内部的网格坐标。**Map tile 用于地形渲染，Pathfinder tile 用于 AI 寻路**——两者通常对齐，但概念上独立。

> **新手记忆**：**世界是一张网格 → PathFinder 维护这张网格的"通行属性" → IsClear 看直线、SubmitSearch 找绕路、AddWall 动态加障碍。** Lua 这边只调 API，真正的搜索算法在 C++ 里。

---

### 12.7.2 快速入门：`Pathfinder:IsClear` —— 直线检查

`IsClear` 是寻路系统**最常用的入口**——它只问一个简单问题：

> "从 A 点到 B 点，**走直线**有没有障碍？"

#### 第一步：API 签名

```lua
TheWorld.Pathfinder:IsClear(x1, y1, z1, x2, y2, z2, pathcaps) → boolean
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `x1, y1, z1` | number | 起点世界坐标 |
| `x2, y2, z2` | number | 终点世界坐标 |
| `pathcaps` | table/nil | 寻路能力（决定哪些障碍要忽略） |

**返回**：true 表示直线无阻碍，false 表示中间有墙/水/不能穿越的格子。

#### 第二步：使用场景一——攻击前判断"能不能直接打到"

```1814:1816:scripts/prefabs/hermitcrab.lua
    if TheWorld.Pathfinder:IsClear(x, 0, z, cx, 0, cz) then
```

寄居蟹判断"我和目标之间有没有障碍"——如果有，就跳到下一个判断分支。

#### 第三步：使用场景二——FindWalkableOffset 的内部判断

`FindWalkableOffset`（在 `simutil.lua`）用来"在某点周围找一个能走的偏移位置"：

```329:343:scripts/simutil.lua
function FindWalkableOffset(position, start_angle, radius, attempts, check_los, ignore_walls, customcheckfn, allow_water, allow_boats)
    return FindValidPositionByFan(start_angle, radius, attempts,
            function(offset)
                local x = position.x + offset.x
                local y = position.y + offset.y
                local z = position.z + offset.z
                return (TheWorld.Map:IsAboveGroundAtPoint(x, y, z, allow_water) or (allow_boats and TheWorld.Map:GetPlatformAtPoint(x,z) ~= nil))
                    and (IsTeleportingPermittedFromPointToPoint(position.x, position.y, position.z, x, y, z))
                    and (not check_los or
                        TheWorld.Pathfinder:IsClear(
                            position.x, position.y, position.z,
                            x, y, z,
                            { ignorewalls = ignore_walls ~= false, ignorecreep = true, allowocean = allow_water }))
                    and (customcheckfn == nil or customcheckfn(Vector3(x, y, z)))
            end)
end
```

每个候选点要满足三个条件：
1. **地形可走**（`IsAboveGroundAtPoint`）
2. **不被传送限制阻挡**（`IsTeleportingPermittedFromPointToPoint`）
3. **直线可达**（`Pathfinder:IsClear`，可选）

注意 `IsClear` 的 `pathcaps` 参数：`ignorewalls = false` 意味着**不忽略墙**——墙在路上就 false。`ignorecreep = true` 意味着**忽略蛛网**——蛛网在路上不算障碍。

#### 第四步：使用场景三——LocoMotor 的"先直接走再寻路"优化

```1834:1838:scripts/components/locomotor.lua
        if ground.Pathfinder:IsClear(p0.x, p0.y, p0.z, p1.x, p1.y, p1.z, self.pathcaps) then
            --print("HAS LOS")
            self:ResetPath()
        else
            --print("NO LOS - PATHFIND")
```

LocoMotor 在每次 `FindPath` 时**先调 IsClear** —— 如果能直接走就不寻路，省掉整个 A*。**这是性能优化的关键**：大多数情况下 NPC 的目标就在视线内，不需要复杂搜索。

#### 第五步：何时用 IsClear，何时用 SubmitSearch

| 需求 | 工具 | 性能 |
|------|------|------|
| 判断 A 到 B 之间是否畅通 | `IsClear` | 极快（毫秒级） |
| 找 A 到 B 的具体路径 | `SubmitSearch` | 慢（几十毫秒） |
| 视线检查（攻击/视觉判定） | `IsClear` | ✓ |
| 让 NPC 走过去 | LocoMotor 自己处理 | ✓ |

**90% 场景用 IsClear 就够了**——直线检查的成本远低于完整寻路。

> **新手记忆**：**IsClear = "A 到 B 走直线行不行？"** 攻击前判断、视线判定、找点筛选，都用它。返回 false 时才考虑用 `SubmitSearch`。

---

### 12.7.3 快速入门：地图通行性判定——`Map:IsPassableAtPoint` 系列

`TheWorld.Map` 是另一个 C++ 组件，负责**单点判定**——某个坐标是不是可以走。它和 PathFinder 互补：PathFinder 关心"路径"，Map 关心"格子"。

#### 第一步：四个核心函数

```lua
TheWorld.Map:IsAboveGroundAtPoint(x, y, z, allow_water)       → boolean
TheWorld.Map:IsPassableAtPoint(x, y, z, include_water, exclude_floating_platforms) → boolean
TheWorld.Map:IsVisualGroundAtPoint(x, y, z)                   → boolean
TheWorld.Map:IsOceanAtPoint(x, y, z, allow_boats)             → boolean
```

| 函数 | 含义 | 典型使用 |
|------|------|---------|
| `IsAboveGroundAtPoint` | 这个点是不是地面（陆地或可选海洋） | 找一个"能站"的位置 |
| `IsPassableAtPoint` | 这个点是不是可通过的（陆地+船+可选海洋） | 移动验证 |
| `IsVisualGroundAtPoint` | 这个点视觉上是陆地 | 区分海陆（更严格） |
| `IsOceanAtPoint` | 这个点是海洋（可选包括船） | 鱼类生成、船只判定 |

**`IsAboveGroundAtPoint` vs `IsPassableAtPoint`**：前者偏"地形角度"，后者偏"实体能否通过"。`IsPassableAtPoint` 还会考虑船只浮台。

#### 第二步：实例——猪人玩家死亡时的骨骼放置

```3727:3727:scripts/stategraphs/SGwilson.lua
                    local skeleton = TheWorld.Map:IsPassableAtPoint(inst.Transform:GetWorldPosition())
```

死亡时如果当前位置可通过 → 放骨骼；如果在不可通过区（如海里）→ 不放骨骼。

#### 第三步：实例——`EntityScript` 上的封装

```1814:1825:scripts/entityscript.lua
function EntityScript:IsOnValidGround() -- this currently does not support boats. IsOnPassablePoint may be what you actually want to call
	return TheWorld.Map:IsVisualGroundAtPoint(self.Transform:GetWorldPosition())
end

function EntityScript:IsOnPassablePoint(include_water, floating_platforms_are_not_passable)
    local x, y, z = self.Transform:GetWorldPosition()
    return TheWorld.Map:IsPassableAtPoint(x, y, z, include_water or false, floating_platforms_are_not_passable or false)
end

function EntityScript:IsOnOcean(allow_boats)
    local x, y, z = self.Transform:GetWorldPosition()
    return TheWorld.Map:IsOceanAtPoint(x, y, z, allow_boats)
end
```

写 Brain 或 Action 时，**优先用 `inst:IsOnValidGround()` / `inst:IsOnPassablePoint()`** —— 比直接调 Map API 简洁。

#### 第四步：搜索可走点的标准模板

```lua
-- 在 inst 周围找一个可放置的点
local x, y, z = inst.Transform:GetWorldPosition()
local offset = FindWalkableOffset(
    Vector3(x, y, z),
    math.random() * TWOPI,  -- 起始角度
    8,                       -- 半径
    8,                       -- 尝试次数
    true,                    -- 是否检查直线可达
    true                     -- 是否忽略墙
)
if offset then
    local target_pos = Vector3(x + offset.x, y, z + offset.z)
    -- 在 target_pos 处生成实体
end
```

`FindWalkableOffset` 内部用了 `FindValidPositionByFan` —— 从给定角度开始扇形遍历 8 个方向，找第一个满足条件的偏移。这是饥荒中**最常用的"找点"模式**。

---

### 12.7.4 进阶：异步寻路——LocoMotor 的 SubmitSearch 流程

当 `IsClear` 返回 false（中间有障碍），LocoMotor 才会启动 A* 搜索。让我们追踪一次完整流程：

#### 第一步：寻路触发

```1817:1862:scripts/components/locomotor.lua
    local ground = TheWorld
    if ground then
        local desttile_x, desttile_y = ground.Pathfinder:GetPathTileIndexFromPoint(p1.x, p1.y, p1.z)

        if desttile_x and desttile_y and self.lastdesttile then
            if desttile_x == self.lastdesttile.x and desttile_y == self.lastdesttile.y then
                return  -- 目标格没变，不重新寻路
            end
        end

        self.lastdesttile = {x = desttile_x, y = desttile_y}

        if ground.Pathfinder:IsClear(p0.x, p0.y, p0.z, p1.x, p1.y, p1.z, self.pathcaps) then
            self:ResetPath()  -- 直线可达，清掉旧路径
        else
            if (self.path and self.path.steps) or not self:WaitingForPathSearch() then
                self:KillPathSearch()
                local handle = ground.Pathfinder:SubmitSearch(p0.x, p0.y, p0.z, p1.x, p1.y, p1.z, self.pathcaps)
                if handle then
                    self.path = self.path or {}
                    self.path.handle = handle
                end
            end
        end
    end
```

**三个关键优化**：

1. **目标格没变就不重算**：`lastdesttile` 缓存上次目标的网格坐标。只要还在同一个 tile 内，不重启搜索（追玩家时尤其有用）
2. **直线可达跳过 A***：先 IsClear 是免费的，能跳过昂贵的 SubmitSearch
3. **避免重复提交**：`WaitingForPathSearch()` 防止上次搜索还没完成就发新请求

#### 第二步：异步等待结果

```1524:1561:scripts/components/locomotor.lua
            if self:WaitingForPathSearch() then
                local pathstatus = TheWorld.Pathfinder:GetSearchStatus(self.path.handle)
                if pathstatus ~= STATUS_CALCULATING then
                    if pathstatus == STATUS_FOUNDPATH then
                        local foundpath = TheWorld.Pathfinder:GetSearchResult(self.path.handle)
                        if foundpath then
                            if #foundpath.steps > 2 then
                                self.path.steps = foundpath.steps
                                self.path.currentstep = 2
                            else
                                self.path.steps = nil
                                self.path.currentstep = nil
                            end
                        end
                    end
                    TheWorld.Pathfinder:KillSearch(self.path.handle)
                    self.path.handle = nil
                end
            end
```

**状态码**（locomotor.lua 第 8-10 行）：

```lua
local STATUS_CALCULATING = 0  -- 还在算
local STATUS_FOUNDPATH = 1    -- 找到了
local STATUS_NOPATH = 2       -- 没路（被注释掉了，统一通过 nil/STATUS_FOUNDPATH 判断）
```

每帧轮询：
- 还在算 → 继续等，先按直线方向走（`wantstomoveforward = true`）
- 找到了 → 取出 `steps`（路径点数组），从第 2 个点开始走
- 没找到 → KillSearch 清掉句柄

**关键的"提前出发"策略**：等待寻路结果时 NPC 不会傻站着——它会先按"直线朝向目标"的方向走。等结果出来后再切换到精确路径。这避免了"思考时间"造成的视觉卡顿。

#### 第三步：路径跟随

```1569:1597:scripts/components/locomotor.lua
                if self.path and self.path.steps and self.path.currentstep < #self.path.steps then
                    local step = self.path.steps[self.path.currentstep]
                    local steppos_x, steppos_y, steppos_z = step.x, step.y, step.z

                    local step_distsq = distsq(mypos_x, mypos_z, steppos_x, steppos_z)

                    local maxsteps = #self.path.steps
                    if self.path.currentstep < maxsteps then
                        local physdiameter = self.inst:GetPhysicsRadius(0)*2
                        step_distsq = step_distsq - physdiameter * physdiameter
                    end

                    if step_distsq <= (self.arrive_step_dist)*(self.arrive_step_dist) then
                        self.path.currentstep = self.path.currentstep + 1
                        ...
                    end
                    facepos_x, facepos_y, facepos_z = steppos_x, steppos_y, steppos_z
                end
```

NPC 沿着 `steps` 数组逐点走——到达第 N 步附近，切换到第 N+1 步。物理半径会被减去（NPC 不需要恰好走到点上，碰到边缘就算到达）。

---

### 12.7.5 进阶：`pathcaps` 能力字段——决定"哪些格子能走"

`pathcaps` 是 LocoMotor 的一个 table 字段，定义了**这只 NPC 能跨越什么样的地形/障碍**。

#### 第一步：常见 pathcaps 字段

| 字段 | 含义 | 谁会设置 |
|------|------|---------|
| `allowocean` | 可以走海洋格 | 蜜蜂、阿比盖尔、月亮草 |
| `ignoreLand` | **不能**走陆地（反向） | 纯水生生物（鱼） |
| `ignorewalls` | 寻路时无视墙 | （主要用于 `IsClear` 调用） |
| `ignorecreep` | 寻路时无视蛛网 | 大多数怪物 |
| `player` | 玩家专属标记 | 玩家（实际不影响搜索） |
| `allowplatformhopping` | 可以跳船 | 自动根据 allow_platform_hopping 设置 |

#### 第二步：实例——蜜蜂卫士可飞过水面

```525:525:scripts/prefabs/beeguard.lua
    inst.components.locomotor.pathcaps = { allowocean = true }
```

蜜蜂卫士的 pathcaps 加了 `allowocean = true`——它寻路时**把海洋格也当成可通过**。这就是"飞虫能直接飞过湖"的代码原理。

#### 第三步：实例——鱼人无视蛛网

```1467:1467:scripts/prefabs/merm.lua
    locomotor.pathcaps = { ignorecreep = true }
```

鱼人寻路时忽略蛛网——它不会因为蛛网减速被 PathFinder 视作"成本高"。注意这是**寻路层面**的忽略，实际进入蛛网格还是会被减速（那是 LocoMotor 速度计算的另一套逻辑）。

#### 第四步：实例——蜘蛛根据子类型变化

```630:630:scripts/prefabs/spider.lua
    inst.components.locomotor.pathcaps = (extra_data and extra_data.pathcaps) or BASE_PATHCAPS
```

```1008:1010:scripts/prefabs/spider.lua
    pathcaps = { ignorecreep = true, allowocean = true },
```

水蜘蛛（waterspider）的 pathcaps 是 `{ ignorecreep = true, allowocean = true }`——它**两栖**，能走水也无视蛛网。

#### 第五步：动态调整——AdjustPathCaps

```1777:1791:scripts/components/locomotor.lua
function LocoMotor:AdjustPathCaps(enabled, capname)
    if enabled then
        if self.pathcaps == nil then
            self.pathcaps = {}
        end
        self.pathcaps[capname] = true
    else
        if self.pathcaps ~= nil then
            self.pathcaps[capname] = nil
            if next(self.pathcaps) == nil then
                self.pathcaps = nil
            end
        end
    end
end
```

允许运行时增删 caps。比如鱼上岸短暂能走陆地、玩家穿戴某种装备能跨水——可以通过 `AdjustPathCaps(true, "allowocean")` 临时开启。

#### 第六步：CanPathfindOnLand / CanPathfindOnWater

```1769:1775:scripts/components/locomotor.lua
function LocoMotor:CanPathfindOnWater()
    return self.pathcaps ~= nil and (not self.pathcaps.allowocean) and (not self.pathcaps.ignoreLand)
end

function LocoMotor:CanPathfindOnLand()
    return self.pathcaps == nil or (not self.pathcaps.ignoreLand)
end
```

> 注：第一个函数命名上有点反常——`CanPathfindOnWater` 实际上是"**陆地生物**"的判断（pathcaps 不为 nil 且不允许海洋且不忽略陆地）。建议读 LocoMotor 源码时配合具体 prefab 反推。

---

### 12.7.6 老手进阶：动态墙与跳台——`AddWall` / `AddStaticHoppablePlatform`

#### 第一步：`AddWall` —— 让 PathFinder 把某个点视为障碍

PathFinder 的导航网格可以**在运行时被修改**——这是为了支持"动态生成的障碍物"（如玩家放置的墙）。

```1:13:scripts/prefabs/walls.lua
require "prefabutil"

local function OnIsPathFindingDirty(inst)
    if inst._ispathfinding:value() then
        if inst._pfpos == nil and inst:GetCurrentPlatform() == nil then
            inst._pfpos = inst:GetPosition()
            TheWorld.Pathfinder:AddWall(inst._pfpos:Get())
        end
    elseif inst._pfpos ~= nil then
        TheWorld.Pathfinder:RemoveWall(inst._pfpos:Get())
        inst._pfpos = nil
    end
end
```

**墙的工作流程**：

1. 玩家放置石墙 → 服务端 prefab 创建 → `_ispathfinding:set(true)` → `OnIsPathFindingDirty` 被触发 → `AddWall(x, y, z)` 注册到 PathFinder
2. 之后所有寻路自动绕开这个点
3. 墙被破坏 → `_ispathfinding:set(false)` → `RemoveWall(x, y, z)` 注销

**`_ispathfinding` 是网络变量**——所以服务端和客户端的 PathFinder 网格保持同步（避免 NPC 在客户端预测时穿墙）。

#### 第二步：地穴大门的特殊处理——4 个墙

```468:479:scripts/prefabs/atrium_gate.lua
    TheWorld.Pathfinder:AddWall(x - 0.5, 0, z - 0.5)
    TheWorld.Pathfinder:AddWall(x - 0.5, 0, z + 0.5)
    TheWorld.Pathfinder:AddWall(x + 0.5, 0, z - 0.5)
    TheWorld.Pathfinder:AddWall(x + 0.5, 0, z + 0.5)
```

```476:479:scripts/prefabs/atrium_gate.lua
    TheWorld.Pathfinder:RemoveWall(x - 0.5, 0, z - 0.5)
    ...
```

地穴大门一个实体覆盖 2×2 个网格，需要注册 4 个墙点。这种"多格碰撞体"的写法可以应用到任何超大障碍物 mod 上。

#### 第三步：`AddStaticHoppablePlatform` —— 跨水跳台

深渊柱（abysspillar）的实现：

```479:480:scripts/prefabs/abysspillar.lua
            TheWorld.Pathfinder:AddStaticHoppablePlatform(point.x, 0, point.z)
```

```487:488:scripts/prefabs/abysspillar.lua
            TheWorld.Pathfinder:RemoveStaticHoppablePlatform(point.x, 0, point.z)
```

注册"静态跳板"——某个点虽然在水中，但 NPC 可以跳到上面，作为路径的中转。配合 `pathcaps.allowplatformhopping`，让陆地生物可以跨水。

LocoMotor 检查跳板存在的逻辑：

```1678:1678:scripts/components/locomotor.lua
				if self.inst.isplayer or TheWorld.Pathfinder:HasStaticHoppablePlatform(mypos_x, 0, mypos_z) then
```

#### 第四步：墙、跳板、寻路的统一图景

```
   网格状态
   ┌─────────────────────────────┐
   │ 1. 静态地形（陆地/海洋/悬崖）│ ← 由地图生成时注册，运行时不变
   │ 2. 墙（AddWall/RemoveWall） │ ← 玩家建造/破坏时动态变化
   │ 3. 跳板（StaticHoppablePf） │ ← 特殊 prefab 注册，定义"虽然不能走但能跳"
   └─────────────────────────────┘
              │
              ▼
        PathFinder 综合
              │
              ▼
   IsClear / SubmitSearch 返回不同结果
```

---

### 12.7.7 老手进阶：`FindValidPositionByFan` —— 扇形搜索的核心算法

`FindValidPositionByFan` 是饥荒 AI 中**最优雅的工具函数**，它实现了"在某点周围按扇形顺序找一个满足条件的偏移"。

#### 第一步：源码

```293:323:scripts/simutil.lua
function FindValidPositionByFan(start_angle, radius, attempts, test_fn)
    attempts = attempts or 8

    local attempt_angle = TWOPI / attempts
    local tmp_angles = {}
    for i = 0, attempts - 1 do
        local a = i * attempt_angle
        table.insert(tmp_angles, a > PI and a - TWOPI or a)
    end

    -- Make the angles fan out from the original point
    local angles = {}
    local iend = math.floor(attempts / 2)
    for i = 1, iend do
        table.insert(angles, tmp_angles[i])
        table.insert(angles, tmp_angles[attempts - i + 1])
    end
    if iend * 2 < attempts then
        table.insert(angles, tmp_angles[iend + 1])
    end

    for i, v in ipairs(angles) do
        local check_angle = start_angle + v
        if check_angle > TWOPI then
            check_angle = check_angle - TWOPI
        end
        local offset = Vector3(radius * math.cos(check_angle), 0, -radius * math.sin(check_angle))
        if test_fn(offset) then
            return offset, check_angle, i > 1 --deflected if not first try
        end
    end
end
```

#### 第二步：扇形搜索的几何意义

假设 attempts=8，start_angle=0：
- 8 个候选角度：0°, 45°, 90°, 135°, 180°, 225°, 270°, 315°
- 经过"fan out"重排后：0°, 315°, 45°, 270°, 90°, 225°, 135°, 180°

**重排的目的**——优先尝试**靠近原方向**的候选，再向两边扩散，最后才尝试反方向。这样找到的点尽量保持原方向感。

```
         start_angle (0°)
              │
        ┌─────┼─────┐
       45°    │   315°    ← 第 2/3 优先（"扇出"开始）
        │     │     │
       90°    │   270°    ← 第 4/5 优先
        │     │     │
       135°   │   225°    ← 第 6/7 优先
        │     │     │
        └────180°───┘     ← 最后尝试（反方向）
```

#### 第三步：返回值

| 返回值 | 含义 |
|-------|------|
| `offset` | Vector3 偏移量（要加到原 position 上得到目标点） |
| `check_angle` | 实际成功的角度 |
| `deflected` | 是否被"偏转"（true 表示首选方向不行） |

**deflected 的用途**：判断目标点是否被障碍偏转。比如玩家面向北但前方有墙，最终扇形搜索可能选了东北—— deflected = true 提示业务层要不要重新计算朝向。

#### 第四步：使用模板

```lua
local mypos = inst:GetPosition()
local offset, angle, deflected = FindValidPositionByFan(
    inst.Transform:GetRotation() * DEGREES,  -- 起始方向 = 当前朝向
    8,                                        -- 半径 8 格
    8,                                        -- 8 次尝试
    function(offset)
        local x = mypos.x + offset.x
        local z = mypos.z + offset.z
        return TheWorld.Map:IsPassableAtPoint(x, 0, z)  -- 自定义条件
            and TheWorld.Pathfinder:IsClear(mypos.x, 0, mypos.z, x, 0, z)
    end
)

if offset then
    local target = Vector3(mypos.x + offset.x, 0, mypos.z + offset.z)
    -- 用 target ...
end
```

`FindWalkableOffset` / `FindSwimmableOffset` / `FindNearbyLand` / `FindNearbyOcean` 都是它的特化版本。

---

### 12.7.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：滥用 `SubmitSearch` 而不是 `IsClear`

```lua
-- 错误：每次都直接用 SubmitSearch 找路径
local function CanReachTarget(inst, target)
    local handle = TheWorld.Pathfinder:SubmitSearch(...)
    -- 等待结果...
end
```

**问题**：`SubmitSearch` 是**异步且昂贵**的——它不能直接返回结果，要么轮询要么挂回调。每帧调用会塞满 C++ 端的搜索队列。

**正确做法**：90% 的"能不能到达"判断用 `IsClear` 就够：

```lua
local function CanReachTarget(inst, target)
    local x1, y1, z1 = inst.Transform:GetWorldPosition()
    local x2, y2, z2 = target.Transform:GetWorldPosition()
    return TheWorld.Pathfinder:IsClear(x1, y1, z1, x2, y2, z2, inst.components.locomotor.pathcaps)
end
```

只有真的需要绕路时（IsClear 返回 false 但仍想找路），才调用 SubmitSearch。

#### 陷阱 2：`Map:IsPassableAtPoint` 和 `Map:IsAboveGroundAtPoint` 混淆

```lua
-- 风险：用错函数导致船上的玩家被判为"在水里"
if not TheWorld.Map:IsAboveGroundAtPoint(x, y, z) then
    -- 错误处理
end
```

**问题**：`IsAboveGroundAtPoint` 严格判断地形，不考虑船。如果玩家站在船上，这个函数返回 false——但玩家**实际可以站立**。

**正确做法**：

| 我想知道 | 用哪个 |
|---------|--------|
| 这里能不能站人/放物体（含船） | `IsPassableAtPoint(x, y, z, false, false)` |
| 这里是不是地形上的陆地（不含船） | `IsAboveGroundAtPoint(x, y, z, false)` |
| 这里视觉上是不是陆地（最严格） | `IsVisualGroundAtPoint(x, y, z)` |
| 这里是不是水（开放海域） | `IsOceanAtPoint(x, y, z, false)` |

#### 陷阱 3：忘记在墙体破坏时 `RemoveWall`

```lua
-- 错误：墙被销毁但没解除注册
function MyWall:OnRemove()
    -- 忘了：TheWorld.Pathfinder:RemoveWall(...)
end
```

**后果**：PathFinder 网格上残留一个"幽灵墙"，所有 NPC 永远绕开这个点。

**正确做法**：注册和注销必须配对。`walls.lua` 用网络变量 `_ispathfinding` + 监听 `onispathfindingdirty` 事件管理生命周期，这是最稳健的模式。

#### 陷阱 4：在客户端调用 `Pathfinder` 修改 API

```lua
-- 错误：在客户端调用 AddWall
if not TheNet:IsDedicated() then
    TheWorld.Pathfinder:AddWall(x, 0, z)  -- 客户端的修改不会同步到服务端！
end
```

**问题**：寻路网格的修改是**服务端权威**的。客户端的 AddWall 只影响本地客户端，不会同步到服务端，可能导致客户端预测和服务端实际行为不一致。

**正确做法**：所有 `AddWall` / `RemoveWall` 应当**只在服务端执行**：

```lua
if TheWorld.ismastersim then
    TheWorld.Pathfinder:AddWall(x, 0, z)
end
```

或者用网络变量 + `onispathfindingdirty` 事件，让两端各自更新本地的 PathFinder。

#### 陷阱 5：自定义 prefab 没设 `pathcaps` 导致水生 NPC 不会游泳

```lua
-- 问题：水生怪物没设 allowocean
local function water_creature_fn()
    inst:AddComponent("locomotor")
    -- 忘了：inst.components.locomotor.pathcaps = { allowocean = true, ignoreLand = true }
end
```

**后果**：寻路时 PathFinder 把海洋格视为不可通过，NPC 卡在岸边走不动。

**正确做法**：

```lua
inst.components.locomotor.pathcaps = { allowocean = true, ignoreLand = true }
```

`ignoreLand = true` 确保它不会"上岸寻路"。

#### 陷阱 6：用 `FindValidPositionByFan` 但 attempts 设得太小

```lua
-- 风险：attempts=2 几乎找不到点
local offset = FindValidPositionByFan(angle, 5, 2, test_fn)
```

**问题**：`attempts=2` 只检查"原方向"和"反方向"两个点。如果两个方向都被障碍挡住，函数返回 nil——但实际上稍微偏一点角度可能就有可走点。

**正确做法**：

| attempts | 适用场景 |
|----------|---------|
| 4 | 简单"四个方向"的搜索（少用） |
| 8 | **默认推荐值**（45° 步长） |
| 12 | 拥挤地形、需要更高成功率 |
| 16 | 罕见情况下使用，性能开销显著 |

#### 设计经验三条

**经验一：性能层次——IsClear → 缓存 → SubmitSearch**

寻路相关代码的性能金字塔：

```
最快   ┌──────────────────┐
       │ Map:IsXxxAtPoint  │  <0.01ms 单点判断
       ├──────────────────┤
       │ Pathfinder:IsClear│  ~0.1ms  直线扫描
       ├──────────────────┤
       │ 缓存的搜索结果     │  0       直接读取已有 path
       ├──────────────────┤
最慢   │ SubmitSearch + 等  │  10ms+   完整 A*
       └──────────────────┘
```

**优化原则**：能用 IsXxxAtPoint 的不用 IsClear，能用 IsClear 的不用 SubmitSearch。LocoMotor 用 `lastdesttile` 缓存上次目标格——这是正版的"前一秒就找过路了，目标没变就别再算"。

**经验二：寻路是"软约束"，不是"硬约束"**

寻路系统给出的 `steps` 数组是**建议路径**，不是强制路径。NPC 在跟随时仍然要避免实际碰撞、可以在路径外被推动、动画播放等都可能导致偏离。

**这意味着**：
- 寻路结果出来后，每帧仍要做局部碰撞检测（locomotor 自动处理）
- 如果一段时间没移动到下一个 step，NPC 会重新调用 `FindPath` 找新路径
- 不要假设"NPC 一旦开始跟路径，就会精确按 steps 走"

**经验三：地图判定优先用 `EntityScript` 封装**

写代码时，**优先用 inst 上的方法**而不是直接调 Map：

```lua
-- 推荐
if inst:IsOnPassablePoint() then ... end
if inst:IsOnValidGround() then ... end
if inst:IsOnOcean() then ... end

-- 不推荐（除非要传特殊参数）
if TheWorld.Map:IsPassableAtPoint(inst.Transform:GetWorldPosition()) then ... end
```

`EntityScript` 的封装：
1. 默认参数符合大多数场景
2. 命名更直观
3. 未来 Klei 改 API 时只需要改一处

---

### 12.7.9 小结

| 概念 | 一句话总结 |
|------|-----------|
| **TheWorld.Pathfinder** | 全局唯一 C++ 组件，管理导航网格和异步搜索队列 |
| **TheWorld.Map** | 全局唯一 C++ 组件，管理单点地形判定 |
| **网格** | 16×16 像素一格，PathFinder 在网格上跑 A* |
| **IsClear** | 直线检查——"A 到 B 中间有阻碍吗？" |
| **SubmitSearch** | 异步 A*，返回 handle，每帧 GetSearchStatus 轮询 |
| **GetPathTileIndexFromPoint** | 世界坐标转网格坐标 |
| **AddWall / RemoveWall** | 运行时注册/注销静态障碍 |
| **AddStaticHoppablePlatform** | 注册"虽然不能走但能跳"的格子 |
| **pathcaps** | LocoMotor 上的能力 table，决定"哪些格子能走" |
| `allowocean` | 可走海洋（飞虫、阿比盖尔） |
| `ignoreLand` | 不能走陆地（鱼类） |
| `ignorecreep` | 寻路时无视蛛网 |
| `ignorewalls` | 寻路时无视墙（主要 IsClear 用） |
| **Map:IsPassableAtPoint** | 含船的"能站立"判断（最常用） |
| **Map:IsAboveGroundAtPoint** | 不含船的"地形是地面"判断 |
| **FindWalkableOffset** | 扇形搜索可走点的标准工具 |
| **FindValidPositionByFan** | 通用扇形搜索器，自定义 test_fn |
| **LocoMotor 流程** | GoToPoint → 设 dest → FindPath → IsClear / SubmitSearch → 跟随 steps |
| **lastdesttile 优化** | 目标格没变时不重启搜索（追玩家时尤其有用） |

**下一节**（12.8）我们将讲解 EntitySleep / Wake 对 AI 的影响——离屏优化机制。


## 12.8 EntitySleep / Wake 对 AI 的影响——离屏优化

### 本节导读

12.1-12.7 我们一直在讨论"**怎么让 AI 决策**"——行为树、节点、调度、寻路、团队配合。但是我们一直假设：**只要这只 AI 存在，它的 brain 每帧都在跑**。

这个假设**只对一半**。饥荒地图上同时存在的实体可能有数千只——猪人、兔人、蜘蛛、各种小动物、远处的 boss、海里的鱼……如果**每只都每帧 tick**，CPU 早就爆了。

引擎用的优化叫 **EntitySleep**：当一个实体**离所有玩家足够远**时，引擎自动把它"**冻结**"——brain 停跑、stategraph 暂停、组件 OnUpdate 不调用、动画也不再播。等玩家走近时再"**唤醒**"。这就是本节要讲的"离屏优化"。

> **新手**从 12.8.1-12.8.3 起步——理解"什么是 sleep"、用控制台亲眼看一只猪人睡和醒、记住三个最常用的 API（`IsAsleep`、`entitysleep`/`entitywake` 事件）；**进阶读者**继续看 12.8.4-12.8.6，深入 `BrainWrangler` 的三个分类集合（updaters / hibernaters / tickwaiters）、`_DisableBrain_Internal` / `_EnableBrain_Internal` 的内部状态机、`LongUpdate` 时间补偿机制；**老手**跳到 12.8.7-12.8.8，逐行拆解 `RemoveFromScene` / `ReturnToScene` 的"假死"机制、`SetCanSleep(false)` 的应用场景、**八个最容易踩的"睡眠相关 bug"**。

读完本节，你能理解为什么"我远远走开后再回来，那只猪人为什么状态错乱"——并且**写出能正确处理睡醒的 AI**。

---

### 12.8.1 快速入门：从一只走远的猪人理解 EntitySleep

#### 第一步：在游戏里观察"消失的"猪人

打开饥荒，找一只猪人，让它执行某个长任务（比如砍一棵远处的树）。

接下来：

1. **远离猪人**——一直走，直到猪人脱离你的视野屏幕之外几屏的距离
2. 等几秒
3. **走回来**

你会发现：

- 走开时——猪人"看起来还在原地砍树"（屏幕上看不到了，但你认为它在）
- 走回来——猪人**居然没砍完**！甚至**还在原来那一帧**！

为什么？因为**当离玩家足够远时，引擎已经把它"冻结"了**——brain 停了、stategraph 停了、动画也不更新了。它**不是真的在砍树**，是**完全静止**。等你回来才"恢复"运行。

#### 第二步：用控制台观察 EntitySleep 状态

按 `~` 打开控制台：

```lua
local pig = c_findnext("pigman")
print(pig:IsAsleep())   -- 屏幕上能看到 pig 时一般返回 false
-- 远离再回来后立刻查
print(pig:IsAsleep())   -- 走远时这只 pig 是 true（如果引擎冻结了它）
```

`IsAsleep()` 是判断"实体当前是否处于离屏冻结状态"的标准 API（`scripts/entityscript.lua:1465`）：

```1465:1467:scripts/entityscript.lua
function EntityScript:IsAsleep()
    return not self.entity:IsAwake()
end
```

它只是 C++ 端 `entity:IsAwake()` 的反函数。**真正决定何时 sleep / wake 的是 C++ 引擎，不是 Lua 代码**——Lua 这边只是被动接收"sleep / wake"事件。

#### 第三步：sleep / wake 的"临界距离"

引擎判断"是否离屏"的逻辑大致是：

> **离所有玩家最近的那一个的距离**超过"sleep 半径"——这只实体进入 sleep；
> 任何玩家走到"wake 半径"以内——这只实体被 wake

**默认 sleep 距离**：约 30 个游戏单位（屏幕大约能看到 24 单位左右，所以是"屏幕外一两屏"的距离开始 sleep）。

**关键**：sleep 不是"暂停游戏"——它是"**这只实体暂时不需要计算**"。它的位置数据、组件状态、库存物品、所有 Lua 字段全部都还在内存里——只是 brain / sg / OnUpdate 不被调用了。**像被定格的电影画面**。

> **新手记忆**：EntitySleep 是**离屏优化**——离玩家太远就冻结、走近就解冻。**`IsAsleep()` 判断当前状态**。

---

### 12.8.2 快速入门：sleep 时三件事会停止

当一个实体进入 sleep 状态，引擎会**自动**做这三件事（你不需要手动管）：

#### 第一件：brain 停止运行

`scripts/entityscript.lua:1119-1132`：

```1119:1132:scripts/entityscript.lua
--V2C: should only be called from OnEntitySleep, RemoveFromScene
function EntityScript:_DisableBrain_Internal()
	if self.brainfn then
		--_braindisabled flag is only valid if we have a brainfn.
		--Don't bother checking "not self._braindisabled".  This code is safe to run
		--multiple times, and we can also assume that the engine properly calls this
		--only when necessary.
		self._braindisabled = true
		if self.brain then
			self.brain:_Stop_Internal()
			self.brain = nil
		end
	end
end
```

**关键**：

- `self._braindisabled = true` —— 标记位
- `self.brain:_Stop_Internal()` —— 调用 brain 内部停止（会把它从 BrainWrangler 的列表移除，行为树停 tick）
- **`self.brain = nil`** —— **整个 brain 对象被销毁**！

这意味着——**sleep 期间 brain 完全不存在**！等 wake 时会用 `brainfn()` **重新创建一个全新的 brain**。所以：**brain 内部的状态（lastsearchpos、当前正在执行的节点等）会被清空**！

#### 第二件：stategraph 停止

`scripts/entityscript.lua:356-358`（注：sleep 走的不是 RemoveFromScene 但行为类似）——

实际上 SGManager 在自己的 update 里检查 `inst:IsAsleep()`：

```136:141:scripts/brain.lua
    local count = 0
    for k, _ in pairs(self.updaters) do
        if k.inst.entity:IsValid() and not k.inst:IsAsleep() then
            count = count + 1
            self._safe_updaters[count] = k
        end
    end
```

可以看到 BrainWrangler 在 `Update` 时**直接跳过 `IsAsleep()` 的实体**——所以哪怕 brain 还在 updaters 列表里，sleep 时也不 tick。

#### 第三件：组件的 OnUpdate 不调用

`scripts/update.lua:255-268`：

```255:268:scripts/update.lua
    TheSim:ProfilerPush("updating components")
    for k, v in pairs(UpdatingEnts) do
        if v.updatecomponents then
            --TheSim:ProfilerPush(v.prefab or "unknown")
            for cmp in pairs(v.updatecomponents) do
                --TheSim:ProfilerPush(v:GetComponentName(cmp))
                if cmp.OnUpdate and not StopUpdatingComponents[cmp] then
                    cmp:OnUpdate(dt)
                end
                --TheSim:ProfilerPop()
            end
            --TheSim:ProfilerPop()
        end
    end
```

注意——这里**没**直接判 IsAsleep。但是大部分组件的 `StartUpdatingComponent` / `StopUpdatingComponent` 在 sleep 时会被引擎间接触发停止。**而且**，C++ 引擎会**直接跳过 sleep 实体的 transform/render/collision 等更新**，从根上省掉了大头开销。

#### 第四件（也是关键的）：`entitysleep` / `entitywake` 事件

引擎进入/退出 sleep 时**推送**这两个事件——你的 prefab 代码可以监听：

```lua
inst:ListenForEvent("entitysleep", function(inst)
    -- 实体即将冻结，做必要的清理
end)

inst:ListenForEvent("entitywake", function(inst)
    -- 实体被唤醒了，做恢复
end)
```

来看一个真实案例——`scripts/prefabs/clockwork_common.lua:7-16`：

```7:16:scripts/prefabs/clockwork_common.lua
local function DoInitHomePosition(inst)
	inst:RemoveEventCallback("entitysleep", DoInitHomePosition)
	inst:RemoveEventCallback("entitywake", DoInitHomePosition)
	inst.components.knownlocations:RememberLocation("home", inst:GetPosition(), true)
end

local function InitHomePosition(inst)
	inst:ListenForEvent("entitysleep", DoInitHomePosition)
	inst:ListenForEvent("entitywake", DoInitHomePosition)
end
```

棋子（chess）刚生成时还**没有 home 位置**。它监听 `entitysleep` / `entitywake`——**任意一个事件先到达**，就把当前位置记成 home。这种"延迟初始化"模式利用了"**实体一定会经历一次 sleep 或 wake**"的事实。

> **新手记忆**：sleep 时 **brain 完全销毁、sg 暂停、组件 OnUpdate 停**。**`entitysleep` / `entitywake` 事件**是你监听睡醒的标准 API。

---

### 12.8.3 快速入门：sleep 不是"假死"——三个常被混淆的状态

新手最常搞混的是这四个状态：

| 状态 | 触发方式 | 实体在场景中？ | brain 状态 | 用途 |
|------|---------|--------------|-----------|------|
| **active** | 默认 | ✅ | 运行 | 正常状态 |
| **asleep**（离屏睡眠）| 引擎自动（玩家远离）| ✅（看不见但还在）| **被销毁**，`_braindisabled = true` | **CPU 优化** |
| **inlimbo**（进入虚空）| 手动 `RemoveFromScene()` | ❌（被隐藏）| **被销毁**，`_braindisabled = true` | 玩家把它放进库存、容器、商店 |
| **dead/removed** | 实体被销毁 | ❌（已不存在）| 不存在 | 永久消失 |

`asleep` 和 `inlimbo` 看起来都"消失了"但区别巨大：

- **asleep** —— 引擎层面的**透明优化**——玩家走近会自动 wake；位置不变；其他实体仍能看到它
- **inlimbo** —— **代码主动调用** `RemoveFromScene()`——比如塞进玩家库存。它**完全不参与世界**——其他实体的 FindEntities 也搜不到它（因为带了 `INLIMBO` 标签）

来看 `RemoveFromScene` 和 sleep 共享的逻辑：

```348:376:scripts/entityscript.lua
function EntityScript:RemoveFromScene()
    self.entity:AddTag("INLIMBO")
    self.entity:SetInLimbo(not self.forcedoutoflimbo)
    self.inlimbo = true
    self.entity:Hide()

	self:_DisableBrain_Internal()

    if self.sg then
        self.sg:Stop()
    end
    if self.Physics then
        self.Physics:SetActive(false)
    end
    if self.Light and self.Light:GetDisableOnSceneRemoval() then
        self.Light:Enable(false)
    end
    if self.AnimState then
        self.AnimState:Pause()
    end
    if self.DynamicShadow then
        self.DynamicShadow:Enable(false)
    end
    if self.MiniMapEntity then
        self.MiniMapEntity:SetEnabled(false)
    end

    self:PushEvent("enterlimbo")
end
```

可以看到——`RemoveFromScene` 和 sleep 都调用了 `_DisableBrain_Internal`——**brain 的销毁逻辑一致**。但是 sleep 不会 Hide、不会 stop 物理。

`ReturnToScene` 是反过来的：

```378:413:scripts/entityscript.lua
function EntityScript:ReturnToScene()
    if not self:IsValid() then
        print("[ERROR] Calling ReturnToScene on an invalid entity!", self.prefab) -- Keep this for debug logs to come.
    end

    self.entity:RemoveTag("INLIMBO")
    self.entity:SetInLimbo(false)
    self.inlimbo = false
    self.entity:Show()
    if self.Physics then
        self.Physics:SetActive(true)
    end
    if self.Light then
        self.Light:Enable(true)
    end
    if self.AnimState then
        self.AnimState:Resume()
    end
    if self.DynamicShadow then
        self.DynamicShadow:Enable(true)
    end
    if self.MiniMapEntity then
        self.MiniMapEntity:SetEnabled(true)
    end

	if self.brainfn or self.sg then
		local asleep = self:IsAsleep()
		if self.brainfn and not (asleep or self.sleepstatepending) then
			self:_EnableBrain_Internal()
		end
		if self.sg then
			self.sg:Start(asleep)
		end
    end
    self:PushEvent("exitlimbo")
end
```

**关键**：`ReturnToScene` 时如果实体仍处于 sleep 状态（asleep == true），**brain 不会重新启用**。要等玩家走近触发 wake 才启用。

> **新手记忆**：**asleep 是引擎层的离屏优化**；**inlimbo 是代码层的"放进虚空"**——例如装进背包。两者都会停 brain，但 inlimbo 还会隐藏、停物理。

---

### 12.8.4 进阶：BrainWrangler——三个集合的精妙调度

12.1.7 我们提过 `BrainWrangler` 是"调度器"。这一节来真正理解它怎么处理 sleep 实体。

`scripts/brain.lua:2-8` 定义了它的内部数据结构：

```2:8:scripts/brain.lua
BrainWrangler = Class(function(self)
        self.instances = {}
        self.updaters = {}
        self._safe_updaters = {} -- NOTES(JBK): Internal use for safely iterating over self.updaters.
        self.tickwaiters = {}
        self.hibernaters = {}
end)
```

**三个核心集合**（除了 `instances` 是总索引、`_safe_updaters` 是临时迭代缓冲外）：

| 集合 | 含义 | 何时进入 |
|------|------|---------|
| `updaters` | **每帧 tick** | 默认所有 brain；wake 后 |
| `hibernaters` | **永久休眠**——除非显式 ForceUpdate | brain `GetSleepTime()` 返回 nil（行为树根节点 sleep 永远） |
| `tickwaiters[N]` | **第 N tick 后再唤醒**——定时唤醒 | brain `GetSleepTime()` 返回正数 |

#### 第一步：调度核心 `BrainWrangler:Update`

```106:157:scripts/brain.lua
function BrainWrangler:Update(current_tick)

	--[[
	local num = 0;
	local types = {}
	for k,v in pairs(self.instances) do

		num = num + 1
		types[k.inst.prefab] = types[k.inst.prefab] and types[k.inst.prefab] + 1 or 1
	end
	print ("NUM BRAINS:", num)
	for k,v in pairs(types) do
		print ("    ",k,v)
	end
	--]]


    local waiters = self.tickwaiters[current_tick]
    if waiters then
        for k,v in pairs(waiters) do
            --print ("BRAIN COMES ONLINE", k.inst)
            self.updaters[k] = true
            self.instances[k] = self.updaters
        end
        self.tickwaiters[current_tick] = nil
    end


    -- NOTES(JBK): We need to make a copy of the keys to safely iterate over the table because brains will remove and add onto the self.updaters table during iteration.
    local count = 0
    for k, _ in pairs(self.updaters) do
        if k.inst.entity:IsValid() and not k.inst:IsAsleep() then
            count = count + 1
            self._safe_updaters[count] = k
        end
    end
    for i = 1, count do
        local k = self._safe_updaters[i]
        self._safe_updaters[i] = nil

        k:OnUpdate()
        local sleep_amount = k:GetSleepTime()
        if sleep_amount then
            if sleep_amount > GetTickTime() then
                self:Sleep(k, sleep_amount)
            else
            end
        else
            self:Hibernate(k)
        end
    end
end
```

**算法分四步**：

1. **唤醒到期的 tickwaiters**——如果当前 tick 命中 `tickwaiters[current_tick]`，把里面所有 brain 移到 updaters
2. **快照 updaters**——复制到 `_safe_updaters`（防止迭代时增删导致崩溃）
3. **跳过 IsAsleep 的实体**——这就是 sleep 实体不被 tick 的关键
4. **每只 brain 调用 `OnUpdate`，根据返回的 `sleep_amount` 决定下一步**：
   - `sleep_amount > GetTickTime()`（即"几 tick 后再叫我"）—— 进 tickwaiters
   - `sleep_amount` 为 `nil` —— 进 hibernaters（行为树暂时无事可做）
   - `sleep_amount` 为 0 或负 —— 留在 updaters（下帧继续 tick）

**关键洞察**：BrainWrangler 的 sleep 机制**和 EntitySleep 是两回事**！

- **BrainWrangler.Sleep / Hibernate**——单纯的"行为树调度优化"——某只 brain 暂时没事干，挪到等待集合，省 CPU
- **EntitySleep（C++ 端）**——离屏优化——整个实体被冻结

在 `BrainWrangler.Update` 里看到的 `IsAsleep()` 检查就是把两者**结合**——只有"在 updaters 集合 + entity 没离屏"才真的 tick。

#### 第二步：`Wake`、`Hibernate`、`Sleep` 三个 API

```56:86:scripts/brain.lua
function BrainWrangler:Wake(inst)
    if self.instances[inst] then
        self:SendToList(inst, self.updaters)
    end
end

function BrainWrangler:Hibernate(inst)
    if self.instances[inst] then
        self:SendToList(inst, self.hibernaters)
    end
end

function BrainWrangler:Sleep(inst, time_to_wait)
    local sleep_ticks = time_to_wait/GetTickTime()
    if sleep_ticks == 0 then sleep_ticks = 1 end

    local target_tick = math.floor(GetTick() + sleep_ticks)

    if target_tick > GetTick() then
        local waiters = self.tickwaiters[target_tick]

        if not waiters then
            waiters = {}
            self.tickwaiters[target_tick] = waiters
        end

        --print ("BRAIN SLEEPS", inst.inst)
        self:SendToList(inst, waiters)

    end
end
```

- `Wake(inst)` —— 把 brain 从任何集合移到 updaters
- `Hibernate(inst)` —— 把 brain 从任何集合移到 hibernaters（永久休眠）
- `Sleep(inst, time_to_wait)` —— 计算目标 tick 编号，移到 tickwaiters[target_tick]

#### 第三步：手动唤醒——`Brain:ForceUpdate()`

```170:176:scripts/brain.lua
function Brain:ForceUpdate()
    if self.bt then
        self.bt:ForceUpdate()
    end

    BrainManager:Wake(self)
end
```

**用途**：当某个事件需要让 brain "立刻评估"，而不等到下一个 sleep 周期。比如：玩家给猪人喂了一块肉，希望它立刻反应过来开始跟随——

```lua
inst:ListenForEvent("itemget", function(inst)
    if inst.brain then
        inst.brain:ForceUpdate()
    end
end)
```

> **进阶记忆**：BrainWrangler 三个集合（updaters / hibernaters / tickwaiters）+ 两层 sleep（brain 调度 sleep + 离屏 EntitySleep）= 完整的 brain 调度机制。

---

### 12.8.5 进阶：`SetBrain` 和 `_EnableBrain_Internal` 的关系

新手以为 brain 是"创建一次永远存在"。**事实上 brain 在 sleep / wake 中**反复**被销毁和重建**——这是个非常容易踩坑的设计。

#### 第一步：`SetBrain` 的注册流程

```1103:1117:scripts/entityscript.lua
function EntityScript:SetBrain(brainfn)
    self.brainfn = brainfn
	--V2C: -sleepstatepending check to prevent brain starting at construction when :IsAsleep() always returns false
	self._braindisabled = brainfn and (self.sleepstatepending or self:IsInLimbo() or self:IsAsleep()) or nil
	if self.brain then
		self.brain:_Stop_Internal()
	end
	self.brain = brainfn and not self._braindisabled and brainfn() or nil
	if self.brain then
		self.brain.inst = self
		if not self._brainstopped then
			self.brain:_Start_Internal()
		end
	end
end
```

**注意**——`SetBrain` 接收的是 `brainfn`（**brain 工厂函数**），不是 brain 实例。原因：每次 wake 时引擎要**重新调用 brainfn() 创建一个新 brain**。

`self.brain = brainfn() or nil`——只有当**实体当前没在 sleep / limbo 中**时，才立刻创建 brain；否则只存 brainfn 等以后用。

#### 第二步：sleep → wake 的 brain 重建流程

引擎内部的 sleep / wake 流程（C++ 端伪代码）：

```
玩家远离实体 →
  C++ 检测到距离超过 sleep 阈值 →
    Lua 端调用 inst:_DisableBrain_Internal()
      → self._braindisabled = true
      → self.brain:_Stop_Internal()    -- brain 内部清理
      → self.brain = nil               -- brain 对象被销毁
    Lua 端 PushEvent("entitysleep")    -- 你的回调
  inst.entity:SetAwake(false)

玩家走近实体 →
  C++ 检测到距离回到 wake 阈值 →
    inst.entity:SetAwake(true)
    Lua 端调用 inst:_EnableBrain_Internal()
      → self._braindisabled = nil
      → self.brain = self.brainfn()    -- 全新 brain 实例
      → self.brain.inst = self
      → self.brain:_Start_Internal()   -- 触发 OnStart 回调
    Lua 端 PushEvent("entitywake")     -- 你的回调
```

```1134:1146:scripts/entityscript.lua
--V2C: should only be called from OnEntityWake, ReturnToScene
function EntityScript:_EnableBrain_Internal()
	if self._braindisabled then
		self._braindisabled = nil
		self.brain = self.brainfn()
		if self.brain then
			self.brain.inst = self
			if not self._brainstopped then
				self.brain:_Start_Internal()
			end
		end
	end
end
```

**含义**：

- **brain 是无状态的**——每次 wake 时全新创建。如果你想在 brain 里存"上次扫到的目标"等状态——**不要存在 brain 上**，存在 `inst` 上！
- **`OnStart` 在每次 wake 都被调用**——这意味着如果你在 `OnStart` 里启动 task，会在每次 wake 重新启动。注意去重！

#### 第三步：`StopBrain` 和 `RestartBrain`——主动控制

`scripts/entityscript.lua` 还提供了一组**程序员主动控制**的 API：

- `inst:StopBrain(reason)` —— 强制停 brain（独立于 sleep 机制）
- `inst:RestartBrain(reason)` —— 重启 brain
- `inst.brain:Pause()` / `Resume()` —— 暂停 / 恢复

实际场景：玩家驯服了一只猪人作为追随者——驯服动画播放期间需要停 brain。这时手动 `inst:StopBrain("being_tamed")`，动画播完 `inst:RestartBrain("done_taming")`。

#### 第四步：`brain:_Start_Internal` 的入口

```206:226:scripts/brain.lua
function Brain:_Start_Internal()
	if not self.stopped then
		return
	elseif self.OnStart then
        self:OnStart()
    end
    self.stopped = false
	if not self.paused then
		BrainManager:AddInstance(self)
	end
	if self.OnInitializationComplete then
		self:OnInitializationComplete()
	end

	-- apply mods
	if self.modpostinitfns then
		for i,modfn in ipairs(self.modpostinitfns) do
			modfn(self)
		end
	end
end
```

**重要顺序**：`OnStart` → `BrainManager:AddInstance` → `OnInitializationComplete`。

**`OnStart`** 是你定义行为树根节点的地方：

```lua
function PigBrain:OnStart()
    local root = PriorityNode({...})
    self.bt = BT(self.inst, root)
end
```

**每次 wake 时这棵行为树都被重建**——**不要在 brain 类成员上存"持久状态"**。

> **进阶记忆**：brain 在 sleep ↔ wake 之间**反复销毁/重建**。**brain 类是"无状态调度逻辑"，持久状态必须存在 `self.inst`（EntityScript）上**。

---

### 12.8.6 进阶：`LongUpdate` —— 时间补偿

#### 第一步：什么是 LongUpdate

考虑这个场景——玩家进入洞穴，**很久**之后回到地面。地面的实体在玩家洞穴期间**完全没 tick**——它们的所有定时器（饥饿、疾病传播、植物生长）都没走。如果直接 wake，会出现"这只猪人居然还和 1 小时前一样不饿"的 bug。

**解决方案**：`LongUpdate(dt)` —— 一次性补偿过去的时间。

`scripts/entityscript.lua:2026-2036`：

```2026:2036:scripts/entityscript.lua
function EntityScript:LongUpdate(dt)
    if self.OnLongUpdate ~= nil then
        self:OnLongUpdate(dt)
    end

    for k, v in pairs(self.components) do
        if v.LongUpdate ~= nil then
            v:LongUpdate(dt)
        end
    end
end
```

引擎在玩家从洞穴回来时**自动**对每只地面实体调用 `LongUpdate(elapsed_seconds)`。每个**支持长时间补偿**的组件（`hunger`、`health.regen`、`perishable`、`temperature` 等）都实现了 `LongUpdate(dt)` 方法，一次性把 dt 秒的状态变化算掉。

**但是 brain 没有 LongUpdate**——因为 AI 决策无法"批量计算"。你不能说"这只猪人在过去 1 小时内决策了 300 次"——它只能在 wake 后**重新开始决策**。

#### 第二步：你自己的 prefab 怎么处理时间补偿

如果你的 AI 涉及"在 sleep 期间应该自然发生的事"——比如"猪人睡觉时房子里的火慢慢熄灭"——你需要自己写 `OnLongUpdate`：

```lua
inst.OnLongUpdate = function(inst, dt)
    -- 处理 dt 秒的"假定行为"
    if inst.fire_remaining then
        inst.fire_remaining = inst.fire_remaining - dt
        if inst.fire_remaining <= 0 then
            -- 火熄灭了
            inst.AnimState:PlayAnimation("burnt")
        end
    end
end
```

或者监听 `entitywake` 事件——计算 sleep 持续了多久：

```lua
local function OnSleep(inst)
    inst._sleep_start_time = GetTime()
end

local function OnWake(inst)
    if inst._sleep_start_time then
        local elapsed = GetTime() - inst._sleep_start_time
        inst._sleep_start_time = nil
        -- 补偿 elapsed 秒
        DoCatchUp(inst, elapsed)
    end
end

inst:ListenForEvent("entitysleep", OnSleep)
inst:ListenForEvent("entitywake", OnWake)
```

#### 第三步：哪些常见组件有 LongUpdate

通过 grep `function .*:LongUpdate` 在 scripts 目录里，可以发现这些组件支持 LongUpdate：

- `hunger` —— 累加饥饿值
- `health.regen` —— 累加自然回血
- `perishable` —— 累加食物腐败
- `temperature` —— 体温变化
- `fueled` —— 燃料消耗
- `growable` —— 植物生长（重点！）
- `pickable` —— 浆果丛重生
- `season` —— 季节进度
- ...

**模式**：所有"按时间累加 / 衰减某个数值"的组件都实现 LongUpdate。所有"基于即时决策"的组件（brain、stategraph、combat）**都不实现** LongUpdate——它们的状态在 wake 时**重新计算**。

> **进阶记忆**：sleep 期间"靠时间累计的状态"在 wake 时被 `LongUpdate(dt)` 一次性补齐。**brain 不补偿**——它在 wake 时重新决策。

---

### 12.8.7 老手：`SetCanSleep(false)` —— 禁止离屏冻结的应用

某些实体**不能离屏冻结**——比如玩家、依赖位置精确同步的网络实体、定时刷怪器、永久存在的特效。这时用 `inst.entity:SetCanSleep(false)`：

#### 案例 1：玩家本身

玩家**永远不能 sleep**——否则地图远端的玩家就消失了。在 `player_common.lua` 里玩家实体被构造时设置 `SetCanSleep(false)`。

#### 案例 2：永不消失的特效——`hermithotspring.lua:28-30`

```28:30:scripts/prefabs/hermithotspring.lua
	--[[Non-networked entity]]
	fx.entity:SetCanSleep(false)
	fx.persists = false
```

热泉特效是**纯客户端效果**——必须始终运行（即使主玩家走远了，**该客户端**所在的玩家可能还能看到）。所以 SetCanSleep(false) + persists = false 的组合：永远 tick + 不存档（因为不需要持久化）。

#### 案例 3：永远活着的家畜——`voidcloth_umbrella.lua:174`

```174:174:scripts/prefabs/voidcloth_umbrella.lua
	inst.entity:SetCanSleep(false)
```

虚空织物雨伞的某个 fx 实体也禁 sleep——这是因为它的位置是**实时跟着玩家的**，sleep 一下位置就会停滞，伞看起来就"飘到天上去了"。

#### 案例 4：刷怪器——`gestalt_cage.lua`、`globalmapicon.lua`

全局地图标记和某些刷怪器需要**始终知道距离玩家多远**，sleep 后无法判断。所以禁 sleep。

#### 注意：禁 sleep 的代价

**每只禁 sleep 的实体都是固定 CPU 开销**。绝对不要为"调试方便"给所有实体禁 sleep——那等于关闭了引擎最重要的优化。**只在确实必要时才禁**。

---

### 12.8.8 老手：八个最容易踩的"睡眠相关 bug"

#### 坑 1：在 brain 类成员上存状态——wake 后丢失

**错误代码**：

```lua
function PigBrain:OnStart()
    self.lastTargetSeen = nil  -- ❌ 错！sleep 后下次 OnStart 时这个变量是新的 brain 的，旧值丢了
    local root = ...
end
```

**正确做法**：

```lua
function PigBrain:OnStart()
    self.inst.lastTargetSeen = self.inst.lastTargetSeen or nil  -- ✅ 存在 inst 上
    local root = ...
end
```

#### 坑 2：sleep 期间的 DoTaskInTime 仍然运行！

**坑**：很多人以为"sleep 时一切都停了"——**错**！`inst:DoTaskInTime(N, fn)` 在 sleep 期间**仍然会到时执行**！它依赖 SchedulerManager，不依赖 brain。

**实际场景**：你给猪人 `DoTaskInTime(60, ResetState)`。10 秒后猪人 sleep 了。再过 50 秒——**ResetState 仍然被调用**！但是此时 brain 已经销毁，如果 ResetState 里访问 `self.brain`，就 nil 错误。

**正确做法**：

```lua
local function ResetState(inst)
    if inst:IsAsleep() then
        return  -- 实体 sleep，跳过
    end
    -- ... 真正的逻辑
end
```

或者在 `entitysleep` 事件里 cancel task：

```5611:5612:scripts/prefabs/merm.lua
local function OnEntitySleepMerm(inst)
    CancelRunHomeTask(inst) -- Cancel it here in case behaviour changes due to components.
```

引用 `merm.lua:625-626`——美人鱼在 sleep 时**主动取消** runhometask，避免 wake 后状态错乱。

#### 坑 3：sleep 期间的 ListenForEvent 仍然触发

类似坑 2——事件系统**不受 sleep 影响**。如果你监听了 `health.componentdamaged`，sleep 期间 health 仍然可能被改（比如远程伤害组件），事件仍然触发，回调仍然跑。

**坑中坑**：回调里引用 `self.sg:GoToState(...)` 时，sg 已经被 sleep 暂停了——`GoToState` 调用没有效果或者出错。

**调试方法**：每个事件回调最前面加 `if inst:IsAsleep() then return end`。

#### 坑 4：wake 后 brain 状态错乱——FindEntity 返回 nil

wake 时引擎重新创建 brain——brain 的"上次执行到哪个节点"全部清空。**如果你的 PriorityNode 写了**"先 condition A，A 成功就执行 B"，wake 后**重新从 A 开始评估**。

实际场景：猪人正在追玩家。玩家在屏幕外让猪人 sleep。回来时 wake——猪人重新评估"有没有威胁"——这一帧玩家恰好不在视野（FindEntity 还没运行），猪人**回到 idle**——下一帧又看到玩家，又开始追——**这一帧的"卡顿"很常见**。

**解决方案**：在 prefab 上记下"上次的 target"，wake 时直接给 combat 组件 `:SetTarget(last_target)`：

```lua
inst:ListenForEvent("entitywake", function(inst)
    if inst._last_target and inst._last_target:IsValid() then
        inst.components.combat:SetTarget(inst._last_target)
    end
end)
```

#### 坑 5：sleep 时位置静止——回来后位置突变

如果实体的位置由 LocoMotor 驱动（比如正在走路），sleep 时 LocoMotor 停了——位置定格。wake 后**位置从定格点重新开始**。

但是有些组件依赖"实时位置"——比如 `homeseeker:GoHome()`，wake 后会**从 sleep 时的位置**开始走回家——可能**离家更远了**。

**解决方案**——`merm.lua:625-640`：

```625:640:scripts/prefabs/merm.lua
local function OnEntitySleepMerm(inst)
    CancelRunHomeTask(inst) -- Cancel it here in case behaviour changes due to components.

    if not inst.wantstoteleport then
        return -- It did not want to teleport anyway, bail.
    end

    if inst.components.follower and inst.components.follower:GetLeader() then
        return -- Leader component takes care of this case by teleporting the entity to the leader.
    end

    local hometraveltime = inst.components.homeseeker and inst.components.homeseeker:GetHomeDirectTravelTime() or nil
    if hometraveltime ~= nil then
        inst.runhometask = inst:DoTaskInTime(hometraveltime, OnRanHome)
    end
end
```

美人鱼**离屏后会"假装"走回家**——计算"如果它正常跑回家需要多少秒"，然后 `DoTaskInTime` 那么久后**直接 teleport** 回家。这是经典的"sleep 期间补偿"模式。

#### 坑 6：sleep 实体被 FindEntity 找不到——是因为 INLIMBO 标签吗？

**易错点**：

- **EntitySleep（离屏）的实体仍然能被 `TheSim:FindEntities` 找到**——它没有 INLIMBO 标签
- **inlimbo 实体**才有 INLIMBO 标签——`scripts/entityscript.lua:349`：`self.entity:AddTag("INLIMBO")`

所以——`FindEntity` 默认**包括** sleep 实体。如果你不希望被 sleep 的实体干扰寻找——**自己加判断 `not target:IsAsleep()`**。

#### 坑 7：玩家穿过 sleep 范围时事件可能不触发

**症状**：玩家**飞快**经过一只实体的视野——你期望 entitywake → ... → entitysleep 都触发——但是**实际可能只触发一次甚至都不触发**。

**原因**：引擎对 sleep / wake 有**滞回**（hysteresis）——避免实体在临界距离边缘反复抖动。如果玩家移动太快，跨过 wake 阈值后立刻又跨出，引擎可能合并掉这次 wake / sleep。

**解决方案**：不要依赖 `entitywake` 触发的次数——用**幂等**的逻辑（重复触发不会出错）。

#### 坑 8：服务端 sleep ≠ 客户端 sleep

**多人模式下**：

- **服务端**的 sleep 判断：实体离**任意服务端代理玩家**的距离
- **客户端**的 sleep 判断：实体离**当前客户端的本地玩家**的距离

所以可能出现：

- 服务端：实体没 sleep（远端玩家仍在范围内）
- 你的客户端：实体 sleep（你走远了）

**坑**：客户端 prefab 代码里**判断 IsAsleep**——可能和服务端判断不一致。**`entitysleep` / `entitywake` 在两端都会触发**，但触发时刻**不同**。

**解决方案**：网络同步状态用 `inst.replica.xxx` / `Net_*` 网络变量，**不要**用 `IsAsleep` 判断。

---

### 12.8 小结：关于 EntitySleep 你必须记住的

```
        玩家走近 ──┐                              ┌── 玩家走远
                  ▼                              ▼
              [active]  ←─── EntityWake ───  [asleep]
              brain 跑                        brain = nil
              sg 跑                           sg 暂停
              组件 Update                     组件 Update 停
                                             entitysleep 事件
                                             ↓
                                   [LongUpdate(dt) 时间补偿]
                                             ↓
                                       wake 时一次性应用
```

**新手核心三句**：sleep 是**离屏 CPU 优化**——离玩家太远就冻结；**`IsAsleep()`** 判断状态、**`entitysleep`/`entitywake`** 是事件；**brain 在每次 wake 都重新创建**。

**进阶核心三句**：BrainWrangler 三集合（updaters / hibernaters / tickwaiters）+ 离屏检查（`IsAsleep`）= 完整调度；**brain 是无状态调度逻辑**——持久状态必须存 `self.inst`；**`LongUpdate(dt)`** 一次性补偿"按时间累加"的状态。

**老手核心三句**：`SetCanSleep(false)` 只在必要时禁 sleep；**`DoTaskInTime` 和 `ListenForEvent` 不受 sleep 影响**——回调里要主动判 IsAsleep；**服务端和客户端的 sleep 状态可能不一致**——网络敏感逻辑用 net var 同步。

下一节（12.9）我们将深入"**仇恨表与目标选择的实现细节**"——12.5 我们提到过 combat 组件的 retarget，但是没有展开"目标优先级、仇恨衰减、广播分享"等真正的细节。看完 12.9 你就能理解为什么"打猪人爸爸全村追你"。


## 12.9 仇恨表与目标选择的实现细节


> **写在前面**——12.5 我们提到过 `Combat:SetRetargetFunction`，知道猪人会"周期搜索敌人"。但是真正写过 mod 的人都知道：**事情没那么简单**。"打一只猪人，全村追你"是怎么做到的？为什么"狗一旦盯上你，你跑到天涯海角它都追"？为什么"蜘蛛在洞外不打你，你打它后队友会一拥而上"？这些行为都不是行为树写的——而是 **Combat 组件**这套精密的"**仇恨表 + 目标分享 + 持有判定**"机制做到的。
>
> 严格说，饥荒**没有传统 RPG 那种"威胁值表"**——每只 NPC 只有**一个 `target`**。但是组合起来的几个机制（`Retarget` 周期搜索、`KeepTarget` 持有判定、`SuggestTarget` 外部推荐、`ShareTarget` 群体共享、`shouldaggrofn` 阻止条件）实际上构成了一个**分布式仇恨系统**——每只 NPC 自己维护"是否打、打谁"的判定，配合事件广播实现群体反应。
>
> **新手**先看 12.9.1-12.9.3——理解"目标只有一个"、`Retarget` 是怎么周期搜索的、`SetTarget` / `DropTarget` / `HasTarget` 三件套；**进阶读者**继续看 12.9.4-12.9.6，深入 `KeepTargetFn` 的"放弃判定"、`SuggestTarget` 的"外部推荐"、`ShareTarget` 怎么把愤怒传染给同类；**老手**直接跳到 12.9.7-12.9.9，掌握 `ShouldAggro` 阻止机制、`StartTrackingTarget` 的事件监听链、`lastwasattackedtime` 等仇恨记忆字段、八个常见的目标选择陷阱与设计经验。

### 12.9.1 快速入门：饥荒不是"威胁表"——它是"单目标 + 周期搜索"

很多从 RPG/MMO 来的玩家会问："饥荒的仇恨表有几个槽？怎么计算威胁值？"——**这个问题问错了方向**。饥荒的 `Combat` 组件就一个字段：

```lua
self.target = nil  -- 当前目标（要么是某个实体，要么是 nil）
```

整个目标系统的核心信息就这一行。判断"我有目标吗"也就一句：

```746:748:scripts\components\combat.lua
function Combat:HasTarget()
    return self.target ~= nil
end
```

那么**多只 NPC 怎么"协同攻击"同一个玩家**呢？答案是：

> **每只 NPC 独立维护自己的 `self.target`**。"群体追击"不是一个全局表，而是 N 只 NPC 的 target 字段**碰巧都指向同一个玩家**。怎么让它们同时指过去？——靠 `ShareTarget` / `SuggestTarget` 这两个"传染机制"（详见 12.9.5）。

**目标选择的两条主入口**：

1. **`Retarget` 主动搜索**（周期任务）——"我每 N 秒扫描一次周围，看有没有该打的"
2. **`SetTarget` / `SuggestTarget` 被动接收**——"队友推了一个目标过来，我接受"

源码 `components/combat.lua` 第 270 行：

```270:282:scripts\components\combat.lua
function Combat:SetRetargetFunction(period, fn)
    self.targetfn = fn
    self.retargetperiod = period

    if self.retargettask ~= nil then
        self.retargettask:Cancel()
        self.retargettask = nil
    end

	if period and fn and not self.inst:IsAsleep() then
        self.retargettask = self.inst:DoPeriodicTask(period, dotryretarget, period*math.random(), self)
    end
end
```

注意：

- `period` 单位是秒——猪人是 3 秒、蜘蛛战士是 2 秒、守卫是 1 秒。
- **第三个参数是随机延迟** `period*math.random()`——避免所有同类 NPC 在同一帧扫描，造成 CPU 尖峰。
- **`IsAsleep()` 检查**——离屏不开周期任务（详见 12.8）。

`fn` 就是 prefab 文件里写的 `Retarget` 函数，必须返回**找到的目标**（Entity）或者 `nil`（没找到）。

**新手只需要记住三个 API**：

```lua
-- 注册"我会怎么找目标"
inst.components.combat:SetRetargetFunction(3, function(inst)
    return FindEntity(inst, 15, function(guy)
        return guy.isplayer and inst.components.combat:CanTarget(guy)
    end, {"_combat"}, {"INLIMBO", "playerghost"})
end)

-- 立刻把谁设为目标
inst.components.combat:SetTarget(player)

-- 询问当前是否有目标
if inst.components.combat:HasTarget() then ... end
```

### 12.9.2 快速入门：`TryRetarget` —— 一次"重新评估"的标准流程

理解 `TryRetarget` 是理解整个仇恨系统的关键。源码 `components/combat.lua` 第 245 行：

```245:268:scripts\components\combat.lua
function Combat:TryRetarget()
    if self.targetfn ~= nil
        and not (self.inst.components.health ~= nil and
                self.inst.components.health:IsDead())
        and not (self.inst.components.sleeper ~= nil and
                self.inst.components.sleeper:IsInDeepSleep()) then

        local newtarget, forcechange = self.targetfn(self.inst)
        if newtarget ~= nil and newtarget ~= self.target and not newtarget:HasTag("notarget") then

            if forcechange then
                self:SetTarget(newtarget)
			    self.lastwasattackedbytargettime = GetTime()
            elseif self.target ~= nil and self.target:HasTag("structure") and not newtarget:HasTag("structure") then
                self:SetTarget(newtarget)
			    self.lastwasattackedbytargettime = GetTime()
            else
                if self:SuggestTarget(newtarget) then
					self.lastwasattackedbytargettime = GetTime()
				end
            end
        end
    end
end
```

逐行解释：

1. **死了不搜索**——`IsDead()` 时直接返回。
2. **真睡着了也不搜索**——`Sleeper:IsInDeepSleep()`（注意这是 12.8 说的"游戏层睡觉"，和 EntitySleep 不同）。
3. **调用 `targetfn`**——你写的那个函数，可以返回**两个**值：`newtarget` 和 `forcechange`。
4. **目标无效则忽略**——`newtarget == nil` 或 `newtarget == self.target`（已经在打它了）或 `HasTag("notarget")`（被标记免目标）。
5. **三种"切换策略"**：
   - **`forcechange == true`**——强制替换当前目标（哪怕已经在打别人）。
   - **当前在打"建筑物"，新目标不是建筑物**——切换（NPC 不会因为忙着拆墙而忽略真正的敌人）。
   - **默认**——只在**当前没目标时才接受**（用 `SuggestTarget` 走"温和路径"）。

**这个三段式逻辑**是设计上的精妙之处：

- 默认情况下 `targetfn` 只是"探测器"，不会强制打断当前战斗。
- 但**结构敌（如猪皇）想强迫狼人切换目标**时，可以让 `targetfn` 返回 `target, true` 强制切换。
- 拆建筑的时候不会"魔怔"——一旦真的玩家闯进来，立刻切到玩家。

**新手实战**：

> 如果你写的怪应该"专注当前目标，绝不分心"，那 `Retarget` 周期可以拉很长（例如 10 秒）；如果应该"很警惕，谁威胁谁优先"，则 1 秒甚至 0.5 秒。**永远不要小于 0.5 秒**——否则一只怪一秒搜两次邻域，地图上 100 只就是 200 次搜索/秒。

### 12.9.3 快速入门：`SetTarget` / `DropTarget` / `EngageTarget` —— 目标的生命周期

`SetTarget` 是**唯一的修改入口**，所有其他改 target 的方式都最终调它。源码 `components/combat.lua` 第 466 行：

```466:475:scripts\components\combat.lua
function Combat:SetTarget(target)
    if target ~= self.target and
        (target == nil or (self:IsValidTarget(target) and self:ShouldAggro(target))) and
		not (target and target.isplayer and target.sg and target.sg:HasStateTag("hiding"))
	then
		local oldtarget = self.target
        self:DropTarget(target ~= nil)
		self:EngageTarget(target, oldtarget)
    end
end
```

`SetTarget` 会先 `DropTarget`（清掉旧目标的事件监听），再 `EngageTarget`（挂上新目标的事件监听）。

`EngageTarget`（第 381 行）：

```381:395:scripts\components\combat.lua
function Combat:EngageTarget(target, oldtarget)
    if target then
		oldtarget = self.target or oldtarget
        self.target = target
        self.inst:PushEvent("newcombattarget", {target=target, oldtarget=oldtarget})
        self:StartTrackingTarget(target)
        if self.keeptargetfn then
            self.inst:StartUpdatingComponent(self)
        end
        local leader = self.inst.components.follower and self.inst.components.follower:GetLeader()
        if leader and leader == target and leader.components.leader and not self.inst.components.follower.keepleaderonattacked then
            leader.components.leader:RemoveFollower(self.inst)
        end
    end
end
```

关键的三件事：

1. **推送 `newcombattarget` 事件**——大脑、状态机、其它系统可以监听这个事件触发战斗动画/AI 切换。
2. **`StartTrackingTarget`**——给目标挂上一系列事件监听（详见 12.9.7）。
3. **如果有 `keeptargetfn`，启动逐帧 update**——每秒检查"还该打它吗"。

`DropTarget`（第 363 行）：

```363:375:scripts\components\combat.lua
function Combat:DropTarget(hasnexttarget)
    if self.target then
        self:SetLastTarget(self.target)
        self:StopTrackingTarget(self.target)
        self.inst:StopUpdatingComponent(self)
        local oldtarget = self.target
        self.target = nil
        if not hasnexttarget then
            self.inst:PushEvent("droppedtarget", {target=oldtarget})
        end
		self.lastwasattackedbytargettime = 0
    end
end
```

注意 `SetLastTarget(self.target)`——目标会**记到 `lasttargetGUID` 字段**作为"上一个目标"，可以用 `IsRecentTarget(target)` 查询：

```343:345:scripts\components\combat.lua
function Combat:IsRecentTarget(target)
    return target ~= nil and (target == self.target or target.GUID == self.lasttargetGUID)
end
```

这就是饥荒"仇恨记忆"的核心——**只记一个最近目标**。当你打了这只猪人，跑开 30 秒，再回来打它，它通过 `IsRecentTarget` 知道"是你"，可能立刻进入战斗状态而不需要重新搜索。

**两个常见事件**：

| 事件 | 何时推送 | 数据 |
|------|---------|------|
| `newcombattarget` | `EngageTarget` 中（设了新目标） | `{target, oldtarget}` |
| `droppedtarget` | `DropTarget` 中（清空目标，且没有下一个） | `{target}` |
| `losttarget` | `KeepTargetFn` 判定为"该放弃" | （无） |

**新手最爱用的 1-2-3**：

```lua
-- 1. 监听战斗开始
inst:ListenForEvent("newcombattarget", function(inst, data)
    print("我开始打:", data.target.prefab)
    inst.SoundEmitter:PlaySound("dontstarve/scary/enter_combat")
end)

-- 2. 监听战斗结束
inst:ListenForEvent("droppedtarget", function(inst, data)
    print("我放弃了:", data.target.prefab)
end)

-- 3. 主动放弃目标（比如逃跑结束）
inst.components.combat:DropTarget()
```

### 12.9.4 进阶：`KeepTargetFn` —— 持有判定与"何时放弃"

12.9.2 讲的是"如何**找到**目标"，本节讲的是"如何**放弃**目标"。每帧（1 秒一次）NPC 都会问自己："这个目标我还该打吗？"。源码 `components/combat.lua` 第 306 行：

```306:341:scripts\components\combat.lua
function Combat:OnUpdate(dt)
    if self.target == nil then
        self.inst:StopUpdatingComponent(self)
        return
    end

    if self.keeptargetfn ~= nil then
        self.keeptargettimeout = self.keeptargettimeout - dt
        if self.keeptargettimeout < 0 then
            if self.inst:IsAsleep() then
                self.inst:StopUpdatingComponent(self)
                return
            end
            self.keeptargettimeout = 1

			local drop
			if not self.target:IsValid() or self.target:IsInLimbo() then
				drop = true
			else
				local iframeskeepaggro_combat = self.target.sg and self.target.sg:HasStateTag("iframeskeepaggro") and self.inst.replica.combat or nil --V2C: intentionally using replica on server
				if iframeskeepaggro_combat then
					iframeskeepaggro_combat.temp_iframes_keep_aggro = true
				end
				drop = not (self.target.components.combat and self.keeptargetfn(self.inst, self.target) and self.target.components.combat:CanBeAttacked(self.inst))
				if iframeskeepaggro_combat then
					iframeskeepaggro_combat.temp_iframes_keep_aggro = nil
				end
			end

			if drop then
                self.inst:PushEvent("losttarget")
                self:DropTarget()
            end
        end
    end
end
```

核心要点：

1. **每秒一次**（`keeptargettimeout = 1`，注意这是写死的 1 秒）。
2. **目标无效则丢弃**——`!IsValid()` 或 `IsInLimbo()`（被刷新的实体）。
3. **调用你写的 `keeptargetfn`**——如果返回 `false` 则丢弃。
4. **离屏直接 stop update**——交给 `OnEntityWake` 时的逻辑重启。
5. **特殊处理 iframeskeepaggro**——目标在攻击无敌帧期间也保持仇恨（避免 boss 闪避后猪人立刻"忘了它"）。

**典型的 `KeepTargetFn`**——蜘蛛的实现，源码 `prefabs/spider.lua` 第 234 行：

```234:241:scripts\prefabs\spider.lua
local function keeptargetfn(inst, target)
   return target ~= nil
        and target.components.combat ~= nil
        and target.components.health ~= nil
        and not target.components.health:IsDead()
        and not (inst.components.follower ~= nil and
                (inst.components.follower:GetLeader() == target or inst.components.follower:IsLeaderSame(target)))
end
```

含义：

- 目标存在、有 combat、有 health、没死。
- **且我和它不是同一阵营的小弟**（避免被人格化的玩家偶然攻击后我反咬他）。

**猪人的 `KeepTargetFn` 更复杂**——源码 `prefabs/pigman.lua` 第 249 行：

```249:253:scripts\prefabs\pigman.lua
local function NormalKeepTargetFn(inst, target)
    --give up on dead guys, or guys in the dark, or werepigs
    return inst.components.combat:CanTarget(target) and target:IsInLight()
        and not (target.sg ~= nil and target.sg:HasStateTag("transform"))
end
```

注意 `target:IsInLight()`——猪人的"放弃逻辑"：**目标进黑暗就不打了**。这是经典的"夜晚追击玩家、玩家躲进火堆就放弃"机制。

**进阶设计经验**：

> `RetargetFn` 决定"什么进入仇恨"，`KeepTargetFn` 决定"什么离开仇恨"。这两个函数**应该是不对称的**——
>
> - `Retarget` 严格（要求很多条件才会盯上你）
> - `KeepTarget` 宽松（一旦盯上你，离开条件少一些）
>
> 这样可以避免 NPC "看见你 → 盯住你 → 你后退一步出搜索范围 → 立刻放弃 → 你前进一步又盯住你"这种**仇恨抖动**。猪人就是经典案例——`Retarget` 要求"在光下"才会发现你，但 `KeepTarget` 也要求"在光下"，所以**只要进黑暗就脱仇恨**——这是有意设计的玩法机制（用火堆躲避）。

`SetKeepTargetFunction` 没有时间参数，因为定死是 1 秒一次。如果你需要"立刻判定一次"，可以手动调 `Combat:ValidateTarget()`（第 481 行），但比较少用。

### 12.9.5 进阶：`SuggestTarget` 与 `ShareTarget` —— 仇恨的传染

12.9.1 提到"群体追击不是全局表，而是 N 只 NPC 的 target 都指向同一目标"——那么是**怎么同步的**？答案是这两个函数。

**`SuggestTarget`**——源码第 229 行：

```229:235:scripts\components\combat.lua
function Combat:SuggestTarget(target)
    if self.target == nil and target ~= nil and (self.cansuggesttargetfn == nil or self.cansuggesttargetfn(self.inst, target)) then
        --print("Combat:SuggestTarget", self.inst, target)
        self:SetTarget(target)
        return true
    end
end
```

**注意它和 `SetTarget` 的关键区别**：

| 函数 | 强制 | 何时设置 |
|------|------|---------|
| `SetTarget(t)` | **强制**替换（满足 ShouldAggro 时） | 任何时候 |
| `SuggestTarget(t)` | **温和**——只在**当前没目标**时才接受 | 当前没目标 |

中间还有个 `cansuggesttargetfn` 钩子，可以拒绝特定的推荐（如 wx78 一些组件用它来阻止"被推荐打主人"）。

**`ShareTarget`**——是 `SuggestTarget` 的**群体广播版**，源码第 185 行：

```185:215:scripts\components\combat.lua
function Combat:ShareTarget(target, range, fn, maxnum, musttags)
    --NOTE: true param ignores my own forbidden tags when sharing someone else's aggro
    if not self:ShouldAggro(target, true) then
        return
    end
    if maxnum <= 0 then
        return
    end

    --print("Combat:ShareTarget", self.inst, target)

    local x, y, z = self.inst.Transform:GetWorldPosition()
    local ents = TheSim:FindEntities(x, y, z, SpringCombatMod(range), musttags or DEFAULT_SHARE_TARGET_MUST_TAGS)

    local num_helpers = 0
    for i, v in ipairs(ents) do
        if v ~= self.inst
            and not (v.components.health ~= nil and
                    v.components.health:IsDead())
            and (fn == nil or fn(v, self.inst))
            and v.components.combat:SuggestTarget(target) then

            --print("    share with", v)
            num_helpers = num_helpers + 1

            if num_helpers >= maxnum then
                return
            end
        end
    end
end
```

逻辑很直观：

1. 在自己周围 `range` 范围内找带 `_combat` 标签的实体。
2. 对每一个，调用过滤函数 `fn`（"是不是我的同伴"）。
3. 给同伴调 `SuggestTarget(target)`——**温和推荐**（同伴还没目标才会接受）。
4. 上限 `maxnum` 个——避免一次广播 100 只蜘蛛全部锁定你。

**经典使用场景**——猪人被打时全村追，源码 `prefabs/pigman.lua` 第 179-200 行：

```179:200:scripts\prefabs\pigman.lua
local function OnAttacked(inst, data)
    --print(inst, "OnAttacked")
    local attacker = data.attacker
    inst:ClearBufferedAction()

	if attacker ~= nil then
		if attacker.prefab == "deciduous_root" and attacker.owner ~= nil then
			OnAttackedByDecidRoot(inst, attacker.owner)
		elseif attacker.prefab ~= "deciduous_root" and not attacker:HasTag("pigelite") then
			inst.components.combat:SetTarget(attacker)

            if inst:HasTag("shadowthrall_parasite_hosted") then
                inst.components.combat:ShareTarget(attacker, SHARE_TARGET_DIST, IsHost, MAX_TARGET_SHARES)
			elseif inst:HasTag("werepig") then
				inst.components.combat:ShareTarget(attacker, SHARE_TARGET_DIST, IsWerePig, MAX_TARGET_SHARES)
			elseif inst:HasTag("guard") then
				inst.components.combat:ShareTarget(attacker, SHARE_TARGET_DIST, attacker:HasTag("pig") and IsGuardPig or IsPig, MAX_TARGET_SHARES)
			elseif not (attacker:HasTag("pig") and attacker:HasTag("guard")) then
				inst.components.combat:ShareTarget(attacker, SHARE_TARGET_DIST, IsNonWerePig, MAX_TARGET_SHARES)
			end
		end
	end
end
```

这一段就是"打猪村全村追你"的实现。看几个关键点：

- **不同身份分享给不同人**——普通猪人传染给"非狼人猪人"、狼人传染给狼人、守卫则视攻击者身份决定（如果攻击者是猪人，只传染给守卫；否则传染给所有猪人）。
- **过滤函数** `IsPig` / `IsWerePig` / `IsGuardPig` / `IsNonWerePig`——只有同身份的猪人才会被传染。
- **`SHARE_TARGET_DIST` 通常是 30**——半径 30 格内的同伴。
- **`MAX_TARGET_SHARES` 通常是 5**——最多 5 个同伴会被通知（防止整个地图全村追你）。

蜘蛛的传染稍有不同——源码 `prefabs/spider.lua` 第 317 行：

```317:329:scripts\prefabs\spider.lua
        inst.components.combat:ShareTarget(data.attacker, 30, function(dude)
                local should_share = dude:HasTag("spider")
                    and not dude.components.health:IsDead()
                    and dude.components.follower ~= nil
                    and dude.components.follower:GetLeader() == inst.components.follower:GetLeader()

                if should_share and dude.defensive and not dude.no_targeting then
                    dude.defensive = false
                end

                return should_share
            end, 10)
```

**只传染给"同一蛛网"的蜘蛛**——`GetLeader() == inst.components.follower:GetLeader()`。这就是为什么"打 A 蛛网的蜘蛛，B 蛛网的蜘蛛不会出来打你"——它们 leader 不同。

**进阶设计经验**：

| 场景 | 推荐做法 |
|------|---------|
| 狼群、群居 NPC | `OnAttacked` 时 `SetTarget` + `ShareTarget`，过滤函数判"同种" |
| 单兵 NPC（巨鹿、独狼） | 只 `SetTarget`，**不**广播 |
| 母性 NPC（蜘蛛后产卵后保护） | 只传染"自己生的"——过滤 `dude.parent == inst` |
| 阵营怪（猪村对蜘蛛） | 攻击者是 X 阵营 → 广播给 Y 阵营 |
| 不希望传染的 boss | 不调 `ShareTarget`；boss 自己用 `Retarget` 主动找 |

### 12.9.6 进阶：`OnAttacked` 与 `SetTarget` —— 不传染但被打

很多新 mod 作者直接把"被打 → 设目标"这一步**忘了**，结果 NPC 被打了一直发呆。原因是：**`Retarget` 只会主动搜索陌生人，不会自动把"刚才打我的人"设为目标**。

正确模式（最简版，无群体传染）：

```lua
local function OnAttacked(inst, data)
    if data.attacker and inst.components.combat:CanTarget(data.attacker) then
        inst.components.combat:SetTarget(data.attacker)
    end
end

-- 在 master_postinit 中
inst:ListenForEvent("attacked", OnAttacked)
```

加上简单群体传染：

```lua
local function OnAttacked(inst, data)
    if data.attacker == nil then return end
    inst.components.combat:SetTarget(data.attacker)
    inst.components.combat:ShareTarget(data.attacker, 20, function(dude)
        return dude:HasTag(inst.prefab)  -- 同种
    end, 5)
end
```

注意 `attacked` 事件的 data 字段（在 `Combat:GetAttacked` 第 686 行推送）：

```lua
self.inst:PushEvent("attacked", {
    attacker = attacker,           -- 攻击者
    damage = damage,               -- 实际伤害
    damageresolved = damageresolved, -- 真正掉的血
    original_damage = original_damage, -- 原始伤害（armor 之前）
    weapon = weapon,
    stimuli = stimuli,             -- 类型："electric" / "ice" / nil
    spdamage = spdamage,           -- 特殊伤害（"planar" 等）
    redirected = damageredirecttarget, -- 是否被骑乘者吸收
    noimpactsound = self.noimpactsound,
})
```

**进阶要点**：

- `data.attacker` 可能是 `nil`（火灾、毒气、世界伤害）——必须先判 nil。
- `data.attacker` 可能是 `weapon`（投射物自爆、陷阱）——通常不要 SetTarget 给武器。**安全做法**：判 `attacker.components.combat ~= nil` 才接受。
- `redirected` 表示"伤害被宠物/骑乘者吸收"——通常不要切换 target 到宠物。

### 12.9.7 老手进阶：`StartTrackingTarget` —— 目标的"事件监听链"

12.9.3 提到 `EngageTarget` 会调 `StartTrackingTarget`，挂上一系列事件监听。这是仇恨系统的一个**精妙设计**——目标自己消失/转移时，攻击者**自动同步状态**，不需要每帧轮询。

源码 `components/combat.lua` 第 347 行：

```347:361:scripts\components\combat.lua
function Combat:StartTrackingTarget(target)
    if target then
        self.inst:ListenForEvent("enterlimbo", self.losetargetcallback, target)
        self.inst:ListenForEvent("onremove", self.losetargetcallback, target)
		self.inst:ListenForEvent("transfercombattarget", self.transfertargetcallback, target)
		self.inst:ListenForEvent("leaderchanged", self.allycheckcallback, target)
    end
end

function Combat:StopTrackingTarget(target)
	self.inst:RemoveEventCallback("leaderchanged", self.allycheckcallback, target)
	self.inst:RemoveEventCallback("transfercombattarget", self.transfertargetcallback, target)
    self.inst:RemoveEventCallback("enterlimbo", self.losetargetcallback, target)
    self.inst:RemoveEventCallback("onremove", self.losetargetcallback, target)
end
```

监听了**目标实体上推送的四个事件**：

| 事件 | 触发条件 | 反应 |
|------|---------|------|
| `enterlimbo` | 目标进入 limbo（被 `RemoveFromScene` 隐藏） | `losetargetcallback` → `DropTarget()` |
| `onremove` | 目标被 `Remove()` 销毁 | `losetargetcallback` → `DropTarget()` |
| `transfercombattarget` | 目标主动推送"转移仇恨"（如猪皇被打转移给守卫） | `transfertargetcallback` → 切到新目标 |
| `leaderchanged` | 目标的 leader 改变（如玩家被附身/解除阵营） | `allycheckcallback` → 如果变成同盟则丢弃 |

`transfertargetcallback`（构造函数中定义，第 85 行）：

```85:99:scripts\components\combat.lua
	self.transfertargetcallback = function(target, newtarget)
		if newtarget ~= nil and self:CanTarget(newtarget) then
			self:SetTarget(newtarget)
			if self.target == target and self.target ~= newtarget then
				self:DropTarget()
			end
		else
			self:DropTarget()
		end
	end
```

**实战用法**——你写一个"挑衅 boss 的玩家手套"，希望让 boss 当前的目标转移到玩家：

```lua
-- 在玩家手套的 onattackother 里
target:PushEvent("transfercombattarget", player)
```

只要 `target` 是某只 NPC，并且别的 NPC 正在打它，那些 NPC 都会**自动切换到 player**——不需要手动遍历每只敌人。**这就是事件机制的威力**：你不需要知道"谁正在仇恨我"，只要广播一下，监听者自己处理。

`allycheckcallback`：

```95:99:scripts\components\combat.lua
	self.allycheckcallback = function(target)
		if self:CanBeAlly(target) then
			self:DropTarget()
		end
	end
```

当目标的 leader 改变时，如果新 leader 让目标变成"我的盟友"（`CanBeAlly` 返回 true），就丢弃目标。例：你的兔人原本和你为敌，但被附身改成你的小弟，原本追它的怪会自动放弃它。

**老手实战经验**：

> 写自定义 boss/精英怪时，**主动推送 `transfercombattarget`** 事件而不是直接修改 `inst.components.combat.target` ——后者绕过了所有事件监听，会导致 NPC 在 `target` 切换后还监听着旧目标的 onremove，造成订阅泄漏。

### 12.9.8 老手进阶：`ShouldAggro` 与"敌意阻止"机制

`SetTarget` / `SuggestTarget` 都会调用 `ShouldAggro` 做最终判定。这是**敌意系统的总闸门**。源码 `components/combat.lua` 第 419 行：

```419:445:scripts\components\combat.lua
function Combat:ShouldAggro(target, ignore_forbidden)
    if target ~= nil and
        (self.shouldaggrofn == nil or self.shouldaggrofn(self.inst, target)) and
        (self.shouldavoidaggro == nil or not self.shouldavoidaggro[target]) and
        (target.components.combat == nil or target.components.combat.shouldavoidaggrofn == nil or target.components.combat.shouldavoidaggrofn(self.inst, target))
        then
        if not ignore_forbidden and self.forbiddenaggrotags ~= nil then
            for _, tag in ipairs(self.forbiddenaggrotags) do
                if target:HasTag(tag) then
                    return false
                end
            end
        end
		if target:HasTag("stealth") then
			return false
		end
		if target.components.health ~= nil and (target.components.health.minhealth or 0) > 0 and not target:HasTag("hostile") then
			target = target.components.follower ~= nil and target.components.follower:GetLeader() or target
			if not target.isplayer then
				--npc should not aggro on things that can't be killed (unless hostile!)
				return false
			end
		end
        return true
    end
    return false
end
```

**四道闸门**：

1. **`shouldaggrofn(inst, target)`**——你自定义的判定函数（设置：`SetShouldAggroFn`）。
2. **`shouldavoidaggro` 表**——存放"暂时不要打这个"的目标键值表（`SetShouldAvoidAggro` / `RemoveShouldAvoidAggro`）。**带计数器**：可以多次注册，必须等所有注册者解除才生效。
3. **目标自己的 `shouldavoidaggrofn`**——目标可以"自我保护"（少见但存在）。
4. **`forbiddenaggrotags` 列表**——禁止打带特定标签的目标（设置：`SetNoAggroTags` / `AddNoAggroTag`）。

**特殊规则**：

- **`stealth` 标签** —— 永远打不到（玩家潜行状态）。
- **`minhealth > 0` + 非 `hostile`** —— 不能打（如盟友 NPC，**除非它是 hostile** 标签的特殊敌人）。
- **该目标是 follower** —— 检查它的 leader（NPC 不打有玩家保护的小兵）。

**老手实战**——一个"暂时让所有怪不打玩家"的 buff：

```lua
-- 玩家进入"和平状态"
for _, ent in pairs(Ents) do
    if ent.components.combat ~= nil then
        ent.components.combat:SetShouldAvoidAggro(player)
    end
end

-- buff 结束
for _, ent in pairs(Ents) do
    if ent.components.combat ~= nil then
        ent.components.combat:RemoveShouldAvoidAggro(player)
    end
end
```

或者**禁止某个 NPC 打"友军" 标签的目标**：

```lua
inst.components.combat:AddNoAggroTag("friendly")
```

之后任何带 `friendly` 标签的目标都不会被这个 NPC 锁定。

**`SetShouldAggroFn` 的高级用法**——基于状态的智能判定：

```lua
inst.components.combat:SetShouldAggroFn(function(inst, target)
    -- 半血以下不主动攻击玩家（怕死）
    if target.isplayer and inst.components.health:GetPercent() < 0.5 then
        return false
    end
    -- 白天只打怪物，晚上谁都打
    if TheWorld.state.isday and not target:HasTag("monster") then
        return false
    end
    return true
end)
```

### 12.9.9 老手进阶：仇恨记忆字段——`lastattacker` / `lastwasattackedtime` / `lastwasattackedbytargettime` / `lasttargetGUID`

`Combat` 组件维护了几个**时间戳字段**，构成一个简陋但有效的"仇恨记忆系统"。

| 字段 | 含义 | 何时更新 |
|------|------|---------|
| `lastattacker` | 上一次攻击我的实体 | `GetAttacked` 中 |
| `lastattacktype` | 上次攻击类型（"projectile" / nil） | `GetAttacked` 中 |
| `laststimuli` | 上次伤害刺激（"electric" / "ice" / nil） | `GetAttacked` 中 |
| `lastwasattackedtime` | 我上次被攻击的时刻 | `GetAttacked` 中（用 `GetTime()`） |
| `lastwasattackedbytargettime` | 当前目标上次攻击我的时刻 | 多处更新 |
| `lasttargetGUID` | 上次锁定的目标的 GUID | `DropTarget` 中（保存到 GUID） |
| `laststartattacktime` | 上次发动攻击的时刻 | `StartAttack` 中 |

**`lastwasattackedbytargettime` 的用途**：判断"目标是不是真的在打我，还是只是它存在那里"。例：

```lua
-- 在 KeepTargetFn 里使用
local function KeepTargetWhileAggressive(inst, target)
    -- 如果 5 秒内目标没真打过我，放弃
    if GetTime() - inst.components.combat.lastwasattackedbytargettime > 5 then
        return false
    end
    return inst.components.combat:CanTarget(target)
end
```

**`lasttargetGUID` 的用途**——做"复仇"AI：

```lua
local function VengefulRetarget(inst)
    -- 优先重新打上一个目标（如果它还活着）
    if inst.components.combat.lasttargetGUID ~= nil then
        local last = Ents[inst.components.combat.lasttargetGUID]
        if last ~= nil and last.components.combat:IsValidTarget(last) then
            return last
        end
    end
    -- 否则做正常搜索
    return FindEntity(inst, 15, ..., RETARGET_MUST_TAGS)
end
```

**`lastattacker` 的危险性**——它**不会自动清空**，目标死后这个字段还指向死者。永远要先判 `IsValid`：

```lua
if inst.components.combat.lastattacker ~= nil
    and inst.components.combat.lastattacker:IsValid()
    and not inst.components.combat.lastattacker:IsInLimbo() then
    -- 安全使用
end
```

### 12.9.10 老手进阶：八个常见的目标选择陷阱与设计经验

#### 陷阱一：`SetTarget` 静默失败

```lua
inst.components.combat:SetTarget(player)
print(inst.components.combat.target)  -- 还是 nil！为什么？
```

`SetTarget` 内部有四道判断：

```466:475:scripts\components\combat.lua
function Combat:SetTarget(target)
    if target ~= self.target and
        (target == nil or (self:IsValidTarget(target) and self:ShouldAggro(target))) and
		not (target and target.isplayer and target.sg and target.sg:HasStateTag("hiding"))
	then
		...
	end
end
```

任意一道失败，`SetTarget` **静默不做任何事**——没有报错、没有日志。常见失败原因：

- 目标没有 `_combat` 标签（未添加 combat 组件）
- 目标 `IsInLimbo`（被刷新或离屏）
- `ShouldAggro` 被某个 `forbiddenaggrotags` 拦下了
- 玩家躲进 hiding 状态

**调试技巧**：

```lua
local target = player
print("IsValidTarget:", inst.components.combat:IsValidTarget(target))
print("ShouldAggro:", inst.components.combat:ShouldAggro(target))
print("CanTarget:", inst.components.combat:CanTarget(target))
```

#### 陷阱二：忘记给 NPC 写 `OnAttacked → SetTarget`

```lua
-- ❌ 错误：只设 Retarget，没设 OnAttacked
inst.components.combat:SetRetargetFunction(3, MyRetargetFn)
-- 结果：玩家偷袭一下，NPC 几秒后才反应（要等下一次 Retarget）
```

修正：监听 `attacked` 事件立刻设目标。

#### 陷阱三：`Retarget` 周期太短，CPU 爆炸

```lua
-- ❌ 错误：0.1 秒一次
inst.components.combat:SetRetargetFunction(0.1, function(inst)
    return FindEntity(inst, 30, ..., {"_combat"}, ...)  -- 30 格扫描，每秒 10 次
end)
```

地图上 50 只这种怪 = 每秒 500 次 30 格 `FindEntities`，服务器直接爆。**最低 1 秒，常用 2-3 秒**，紧迫场合最低 0.5 秒。

#### 陷阱四：`Retarget` 函数里调了昂贵的 API

```lua
-- ❌ 错误：在 Retarget 里调用 Pathfinder
local function BadRetargetFn(inst)
    local target = FindEntity(inst, 30, ..., RETARGET_MUST_TAGS)
    if target ~= nil then
        -- 检查能不能寻路到
        if TheWorld.Pathfinder:IsClear(...) then
            return target
        end
    end
end
```

`IsClear` 在 30 格扫到 5-10 个候选时，每秒就是 30+ 次直线检查——能避免就避免。把"能不能寻路"留给 brain 的 `ChaseAndAttack` 节点决定。

#### 陷阱五：`ShareTarget` 的 maxnum 设太大

```lua
inst.components.combat:ShareTarget(attacker, 50, IsPig, 100)  -- 50 格内 100 只猪都广播
```

50 格 `FindEntities` 已经很贵了，再每个调 `SuggestTarget`——一次 `OnAttacked` 跑掉 200ms。**maxnum 一般 5-15**，半径一般 20-30 格。

#### 陷阱六：在 `OnAttacked` 里直接 `inst.components.combat.target = data.attacker`

```lua
-- ❌ 错误：绕过事件机制
inst:ListenForEvent("attacked", function(inst, data)
    inst.components.combat.target = data.attacker  -- 直接赋值！
end)
```

后果：

- 不会推送 `newcombattarget` 事件——大脑、SG、UI 全都不知道你切了目标。
- 不会调 `StartTrackingTarget`——目标死后的 `onremove` 没人接，旧 target 一直是 dangling reference。
- `keeptargetfn` update 不会启动。

**永远使用 `SetTarget`** 而不是 `inst.components.combat.target = ...`。

#### 陷阱七：忘记在死亡时清理目标

```lua
-- 玩家被打死后，仇敌还在追"已死玩家"
-- 这通常不是问题——目标 onremove 时会自动调 DropTarget
-- 但有些自定义"复活"流程不发 onremove 事件
```

如果你的"自定义死亡"会让玩家暂时离场再回归（如苏醒花），`Combat` 组件**不会自动清理**。建议手动推 `enterlimbo` 或者直接 `DropTarget`：

```lua
-- 在玩家"假死"逻辑中
player:PushEvent("enterlimbo")  -- 让所有正在追玩家的怪自动 DropTarget
-- 复活
player:PushEvent("exitlimbo")
```

#### 陷阱八：`ShareTarget` 过滤函数返回 `nil` 当 `false` 用

```lua
-- ❌ 危险：忘了显式 return false
inst.components.combat:ShareTarget(attacker, 30, function(dude)
    if dude:HasTag("pig") then
        return true
    end
    -- 没写 return false，隐式返回 nil
end, 5)
```

Lua 里 `nil` 在 if 判断中和 `false` 等价，**这里恰好不会出错**。但当过滤函数复杂起来（多个 if-else 嵌套），漏一个 `return` 容易让本不该接受的目标被接受。**永远显式 `return false`**：

```lua
function(dude)
    if not dude:HasTag("pig") then return false end
    if dude.components.health:IsDead() then return false end
    return true
end
```

### 12.9.11 小结

```
                    +------------------------+
                    |    OnAttacked 事件     |
                    +-----------+------------+
                                |
                  +-------------+-------------+
                  |                           |
            SetTarget(attacker)        ShareTarget(attacker, ...)
                  |                           |
                  v                           v
           +------+------+              +-----+-----+
           | EngageTarget|              | 邻域同伴 |
           +------+------+              | SuggestT.|
                  |                     +-----------+
                  v
           +------+------+              +-----------+
           |   target    |<----周期----+ Retarget   |
           +------+------+              +-----------+
                  |
                  v
           +------+------+   每秒
           | KeepTarget? +-------> 否 -> losttarget -> DropTarget
           +-------------+
```

| 概念 | 速记 |
|------|------|
| **`self.target`** | 唯一的目标字段（不是表） |
| **`SetTarget`** | 唯一应该用的"切换目标"入口；走完整流程 |
| **`SuggestTarget`** | 温和推荐——只在没目标时接受 |
| **`ShareTarget`** | 邻域广播——给同伴 SuggestTarget |
| **`SetRetargetFunction(period, fn)`** | 周期主动搜索（period 不少于 1 秒） |
| **`SetKeepTargetFunction(fn)`** | 每秒一次的"放弃判定"，决定何时丢目标 |
| **`TryRetarget`** | 一次手动重新评估；`forcechange` 可以强制切换 |
| **`StartTrackingTarget`** | 自动监听目标的 enterlimbo / onremove / transfercombattarget / leaderchanged |
| **`ShouldAggro`** | 终极敌意判定——4 道闸门 + 标签过滤 |
| **`forbiddenaggrotags`** | 禁打的标签列表（`SetNoAggroTags`） |
| **`shouldavoidaggro`** | 临时不打的目标表（带计数） |
| **核心事件 `newcombattarget`** | 目标被设置 |
| **核心事件 `losttarget`** | KeepTargetFn 判定放弃 |
| **核心事件 `droppedtarget`** | DropTarget 被调用 |
| **核心事件 `transfercombattarget`** | 推送给目标实体，让追它的怪转移仇恨 |
| **`lastattacker` / `lasttargetGUID`** | 简陋的"仇恨记忆"——上次打我的、上次的目标 |

**最重要的设计哲学**：

> 饥荒**没有威胁值**——它用"**单目标 + 事件驱动**"实现群体仇恨。每只 NPC 自己维护 target，靠 `ShareTarget` 同步，靠 `Retarget` 主动搜索，靠 `KeepTarget` 决定何时放弃，靠 `transfercombattarget` 转移。**这套设计的核心是"分布式 + 事件化"**——没有全局协调器，每只 NPC 独立决策，但通过事件协调出群体行为。

**下一节**（12.10）将是本章的实战篇——我们将完整地为一个自定义生物从零编写一个 AI 大脑：定义 prefab、写 Retarget、构造行为树、添加团队战斗——把 12.1-12.9 的所有内容串起来。

## 12.10 实战：为自定义生物编写 AI 大脑


> **写在前面**——12.1~12.9 我们已经把行为树、节点、`braincommon`、动作系统、AI 模式、团队战斗、寻路、离屏优化、目标选择**九大模块**全部讲了一遍。但单独看一个模块还是有点抽象——本节将把这些**全部串起来**：用一个完整的自定义生物 demo，**从零写出一个能跑能打、能巡逻能逃跑、能组团能归家的 AI**。
>
> **demo 主角**：我们设计一种叫"**苔灵（mossling）**"的中等敌对生物。它有以下需求：
>
> - 平时在家（地刺/蘑菇巢）附近游荡（**Wander**）
> - 看到玩家会追击攻击（**ChaseAndAttack** + **Retarget**）
> - 受伤超过 70% 会恐慌一秒，然后继续战斗（**PanicTrigger**）
> - 如果同伴在附近（≥3 只），自动组队从扇形围攻玩家（**TeamAttacker / TeamLeader**）
> - 离屏完全冻结，醒来时不能"暴走"或"出错"（**EntitySleep / Wake**）
> - 白天会回家"打盹"（**GoHome + Sleeper 组件**）
> - 被打时同伴会被通知一起追杀（**ShareTarget**）
>
> **新手**先看 12.10.1-12.10.4——理解整体设计结构、最小可用 prefab 模板、最简 Brain 模板、添加 Combat 仇恨；**进阶读者**继续看 12.10.5-12.10.7，给生物加 BufferedAction 自定义动作、做正确的离屏处理、添加日夜行为切换；**老手**直接跳到 12.10.8-12.10.9，加上团队战斗、自定义事件协调、性能优化与陷阱回顾。
>
> **本节的代码全部为参考实现**——按照工作区规则，我不修改项目代码，所有 lua 都是教学用的"读了就懂、改了就能跑"的模板。

### 12.10.1 第一步：把需求拆成"行为优先级表"

在写代码之前，先把需求**列成优先级表**——这是行为树设计的核心。为什么这一步重要？因为 12.2 讲过，`PriorityNode` 是从上到下扫描的，**写错优先级 = AI 行为完全错乱**。

**苔灵行为优先级表**（从高到低）：

| 优先级 | 触发条件 | 行为 | 用什么节点 |
|------|---------|------|------------|
| **0** | 着火、僵硬、被吓飞 | 强制恐慌 | `PanicTrigger`（来自 `braincommon`）|
| **1** | 血量 < 30% | 逃跑回家 | `WhileNode + RunAway` |
| **2** | 有目标 + 不在攻击冷却 | 追击攻击 | `WhileNode + ChaseAndAttack` |
| **3** | 队伍战斗模式 | 听队长指令（formation） | `TeamAttacker:OnUpdate` 自驱动 |
| **4** | 白天 + 离家近 + 没目标 | 在家附近打盹 | `WhileNode + Wander` |
| **5** | 夜晚 + 没目标 | 在家附近游荡 | `WhileNode + Wander` |
| **6** | 离家太远 | 拉回家 | `Leash` |
| **7** | 默认 | 自由游荡 | `Wander` |

**新手设计原则**：

- **越特殊的越靠前**——着火、低血量是少见但必须立刻响应的，放最前面。
- **战斗在常态前**——`ChaseAndAttack` 必须在 `Wander` 之前，否则发现敌人也只会瞎逛。
- **常态在最后**——`Wander` 是兜底，永远 RUNNING。

> **进阶提示**：行为优先级表写在大脑的注释里，比写在文档里有用得多——半年后回来改 mod，看注释就知道当初为什么这么排。

### 12.10.2 第二步：最小可用 prefab 文件

先写一个**没有任何 AI** 的"硬核空壳"——确保实体能在世界中正常加载、动画、移动、被打。这是所有 mod 调试的起点。

```lua
-- mods/mymod/scripts/prefabs/mossling.lua
local assets = {
    Asset("ANIM", "anim/mossling.zip"),
}

local prefabs = {
    "meat",
    "moss",
}

local brain = require("brains/mosslingbrain")

-- 战斗参数（建议放 TUNING）
local DAMAGE = 25
local ATTACK_PERIOD = 2
local ATTACK_RANGE = 2
local HIT_RANGE = 2.5
local HEALTH = 200
local WALK_SPEED = 2
local RUN_SPEED = 5

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddDynamicShadow()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeCharacterPhysics(inst, 50, 0.75)

    inst.AnimState:SetBank("mossling")
    inst.AnimState:SetBuild("mossling")
    inst.AnimState:PlayAnimation("idle")

    inst:AddTag("monster")
    inst:AddTag("hostile")
    inst:AddTag("character")
    inst:AddTag("scarytoprey")
    inst:AddTag("mossling")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    -- ==== 主从端的服务端逻辑 ====
    inst:AddComponent("locomotor")
    inst.components.locomotor.walkspeed = WALK_SPEED
    inst.components.locomotor.runspeed = RUN_SPEED

    inst:AddComponent("health")
    inst.components.health:SetMaxHealth(HEALTH)

    inst:AddComponent("combat")
    inst.components.combat:SetDefaultDamage(DAMAGE)
    inst.components.combat:SetAttackPeriod(ATTACK_PERIOD)
    inst.components.combat:SetRange(ATTACK_RANGE, HIT_RANGE)

    inst:AddComponent("lootdropper")
    inst.components.lootdropper:SetLoot({"meat", "moss"})

    inst:AddComponent("inspectable")
    inst:AddComponent("knownlocations")

    inst:SetStateGraph("SGmossling")
    inst:SetBrain(brain)

    MakeHauntablePanic(inst)

    return inst
end

return Prefab("mossling", fn, assets, prefabs)
```

这个"骨架"提供了 AI 工作的最低条件：

| 组件 | 必需吗？ | 用途 |
|------|---------|------|
| `locomotor` | ✅ 必需 | 移动、寻路（12.7） |
| `health` | ✅ 必需 | 死亡判定 |
| `combat` | ✅ 战斗系统必需 | 攻击、目标管理（12.9） |
| `inspectable` | ⚠️ 强烈推荐 | 可以右键查看 |
| `knownlocations` | ⚠️ 推荐 | 记录"家"的位置 |
| `lootdropper` | ⚠️ 推荐 | 死后掉落 |
| `sleeper` | 可选 | 游戏层睡觉（夜晚兔人那种）|
| `homeseeker` | 可选 | 寻找回家的实体 |

注意**几个关键标签**：

- `_combat`：**自动**由 `combat` 组件添加（不用手动 AddTag），让别的怪能 `FindEntities` 找到它。
- `monster`：决定属于"怪物"，影响别的 NPC 是否敌视它。
- `hostile`：决定 `ShouldAggro` 中的 `minhealth` 检查（敌对怪可以打非可杀实体）。
- `scarytoprey`：让兔子、河狸等"猎物"看到它会逃。
- `character`：被一些寻路/搜索逻辑使用。

### 12.10.3 第三步：最小可用 Brain 文件（新手版）

新手只需要把"行为优先级表"翻译成 PriorityNode 的子节点列表。这是最朴素的 brain：

```lua
-- mods/mymod/scripts/brains/mosslingbrain.lua
require "behaviours/wander"
require "behaviours/chaseandattack"
require "behaviours/runaway"
require "behaviours/panic"
require "behaviours/leash"

local BrainCommon = require "brains/braincommon"

local SEE_DIST = 12
local MAX_CHASE_TIME = 8
local MAX_CHASE_DIST = 25
local WANDER_DIST = 10
local LEASH_RETURN_DIST = 8
local LEASH_MAX_DIST = 25
local LOW_HEALTH = 0.3
local START_RUNAWAY_DIST = 4
local STOP_RUNAWAY_DIST = 8

local function GetHomePos(inst)
    return inst.components.knownlocations:GetLocation("home")
end

local function HasLowHealth(inst)
    return inst.components.health:GetPercent() < LOW_HEALTH
end

local MosslingBrain = Class(Brain, function(self, inst)
    Brain._ctor(self, inst)
end)

function MosslingBrain:OnStart()
    local root = PriorityNode(
    {
        BrainCommon.PanicTrigger(self.inst),                       -- 0. 强制恐慌
        WhileNode(function() return HasLowHealth(self.inst) end,    -- 1. 低血逃跑
            "Wounded",
            RunAway(self.inst, "player", START_RUNAWAY_DIST, STOP_RUNAWAY_DIST)),
        ChaseAndAttack(self.inst, MAX_CHASE_TIME, MAX_CHASE_DIST),  -- 2. 追击攻击
        Leash(self.inst, GetHomePos, LEASH_MAX_DIST, LEASH_RETURN_DIST), -- 6. 拉回家
        Wander(self.inst, GetHomePos, WANDER_DIST),                 -- 7. 默认游荡
    }, 0.5)

    self.bt = BT(self.inst, root)
end

return MosslingBrain
```

**逐行解读**：

1. **`require` 行为节点**——每个 `Wander`、`ChaseAndAttack` 等都是独立的文件，必须 require 才能使用。
2. **常量提取**——把所有 magic number 提到顶部，方便调参。新手最容易写成"distance = 12"散落在 brain 里，调起来痛苦。
3. **辅助函数**——`GetHomePos` / `HasLowHealth` 写成 module 级函数，**让 brain 节点的 lambda 写成单行**——这样 PriorityNode 列表非常清晰。
4. **`PriorityNode({...}, 0.5)`**——0.5 是 sleeptime hint（详见 12.1），表示"节点空闲时多 0.5 秒重评估一次"。
5. **`BT(self.inst, root)`**——把根节点交给 BT 包装。

**新手要避免的两个坑**：

- ❌ **没注册 `targetfn`**——这个 brain 只追击你**已经设了 target 的目标**。新手常常忘了 `combat:SetTarget`，结果苔灵看见你也不动——它确实没目标。修正：在 prefab 文件里加 `inst.components.combat:SetRetargetFunction(2, MyRetargetFn)`（详见 12.10.4）。
- ❌ **`Wander` 没设范围**——`Wander(inst)` 只传一个参数会报错。最少要传 `(inst, getpointfn, distance)`。

**至此，运行游戏：**

- `c_spawn("mossling")` 出来一只苔灵。
- 它会在脚下 10 格内游荡（因为没 home，所以 GetHomePos 返回 nil，Wander 就在原地附近转）。
- 用 `c_force_aggro(c_select())` 强制它打你，它会追，打完会 RUNNING 在 ChaseAndAttack 里。

但**它看见你不会主动追**——这就要进入 12.10.4。

### 12.10.4 第四步：加上 Retarget / OnAttacked / ShareTarget（让仇恨系统真正运作）

回顾 12.9：苔灵需要"看到玩家就追"，必须设置 `SetRetargetFunction`。同时"被打了反击 + 通知同伴"也需要 `OnAttacked` + `ShareTarget`。

回到 prefab 文件，在 master_postinit 中追加：

```lua
local SHARE_TARGET_DIST = 18
local MAX_TARGET_SHARES = 5

local RETARGET_MUST_TAGS = { "_combat" }
local RETARGET_CANT_TAGS = { "INLIMBO", "playerghost", "notarget" }
local RETARGET_ONEOF_TAGS = { "character", "monster" }

local function NormalRetargetFn(inst)
    if inst:HasTag("being_followed") or inst:IsInLimbo() then return nil end
    return FindEntity(
        inst,
        SEE_DIST,
        function(guy)
            return inst.components.combat:CanTarget(guy)
                and not guy:HasTag("mossling")  -- 不打同类
        end,
        RETARGET_MUST_TAGS,
        RETARGET_CANT_TAGS,
        RETARGET_ONEOF_TAGS)
end

local function NormalKeepTargetFn(inst, target)
    return inst.components.combat:CanTarget(target)
        and inst:GetDistanceSqToInst(target) < (LEASH_MAX_DIST * 1.5) ^ 2
end

local function IsMossling(dude)
    return dude:HasTag("mossling")
end

local function OnAttacked(inst, data)
    if data.attacker == nil
        or data.attacker == inst
        or data.attacker:HasTag("mossling") then
        return  -- 忽略：来自世界、自己、同类
    end
    inst.components.combat:SetTarget(data.attacker)
    inst.components.combat:ShareTarget(
        data.attacker,
        SHARE_TARGET_DIST,
        IsMossling,
        MAX_TARGET_SHARES)
end

local function OnNewTarget(inst, data)
    -- 设置新目标时，主动通知附近同伴一起来
    if data.target ~= nil then
        inst.components.combat:ShareTarget(
            data.target,
            SHARE_TARGET_DIST,
            IsMossling,
            MAX_TARGET_SHARES)
    end
end

-- 在 master_postinit 中追加：
inst.components.combat:SetRetargetFunction(2, NormalRetargetFn)
inst.components.combat:SetKeepTargetFunction(NormalKeepTargetFn)

inst:ListenForEvent("attacked", OnAttacked)
inst:ListenForEvent("newcombattarget", OnNewTarget)
```

**进阶要点逐条解读**：

1. **`RETARGET_MUST_TAGS = {"_combat"}`** —— 详见 12.9，`FindEntity` 必须传 must tags 否则会扫描整个地图所有实体（性能灾难）。`_combat` 由 combat 组件自动加，可作为"有战斗组件的实体"快速过滤。
2. **`RETARGET_CANT_TAGS = {"INLIMBO", "playerghost", "notarget"}`** —— 12.9.8 提到的"标签过滤"——避免锁定鬼魂、limbo、被标记免目标的玩家。
3. **`RETARGET_ONEOF_TAGS = {"character", "monster"}`** —— 至少要有"character"或"monster"标签——这样不会去打没意义的物品。
4. **`function(guy)` 的检查**——双重保险：除了 must/cant/oneof，还要 `CanTarget`（验证不能瞄准的特例）和 `not guy:HasTag("mossling")`（不打同类——这是关键的"阵营保护"）。
5. **`KeepTargetFn` 的距离检查**——目标跑到 `LEASH_MAX_DIST * 1.5` 之外就放弃（不会追到天涯海角）。
6. **`OnAttacked` 三层过滤**——不是 nil、不是自己、不是同类。漏掉同类检查会出现"两只苔灵互殴到死"的喜剧。
7. **`OnNewTarget` 的二次广播**——不光被打时广播，**主动锁定时也广播**。这样如果只苔灵 A 通过 Retarget 发现了你，它的设置 target 会通知所有同伴——连环追击。

**测试**：

- `c_spawn("mossling")` 出来一只
- 走过去站在它面前
- 大约 2 秒内（一次 Retarget 周期），它会冲过来打你
- 打它一下，附近的同伴都会一拥而上

### 12.10.5 第五步：加 BufferedAction 与自定义动作（进阶）

12.4 讲了 BufferedAction。来给苔灵加一个**主动行为**：看到地上有"moss"（苔藓物品）会去吃掉补血。这不是必需的，但可以演示如何把"自由 AI 行为"嵌进树。

```lua
-- 在 brain 文件中新增：
local SEE_FOOD_DIST = 8

local FINDFOOD_CANT_TAGS = { "INLIMBO", "outofreach" }

local function FindFoodAction(inst)
    if inst.sg:HasStateTag("busy") then return nil end
    if inst.components.combat:HasTarget() then return nil end  -- 战斗时不吃

    local target = FindEntity(
        inst,
        SEE_FOOD_DIST,
        function(item)
            return item.prefab == "moss"
                and item:IsOnPassablePoint(true)  -- 12.7：站得上去
        end,
        { "_inventoryitem" },  -- 物品才有 _inventoryitem
        FINDFOOD_CANT_TAGS)

    return target ~= nil and BufferedAction(inst, target, ACTIONS.EAT) or nil
end
```

然后在 brain root 的 PriorityNode 中**插入到 ChaseAndAttack 之后、Wander 之前**：

```lua
function MosslingBrain:OnStart()
    local root = PriorityNode(
    {
        BrainCommon.PanicTrigger(self.inst),
        WhileNode(function() return HasLowHealth(self.inst) end, "Wounded",
            RunAway(self.inst, "player", START_RUNAWAY_DIST, STOP_RUNAWAY_DIST)),
        ChaseAndAttack(self.inst, MAX_CHASE_TIME, MAX_CHASE_DIST),

        DoAction(self.inst, FindFoodAction),  -- ← 新加：吃苔藓回血

        Leash(self.inst, GetHomePos, LEASH_MAX_DIST, LEASH_RETURN_DIST),
        Wander(self.inst, GetHomePos, WANDER_DIST),
    }, 0.5)
    self.bt = BT(self.inst, root)
end
```

要让 `EAT` 动作真的能吃，prefab 还需要加 `eater` 组件：

```lua
inst:AddComponent("eater")
inst.components.eater:SetDiet({ FOODGROUP.OMNI }, { FOODGROUP.OMNI })
```

**进阶要点**：

- `DoAction(inst, fn)` 的 fn 可以**返回 nil**——表示"我现在没事可做"，节点 FAILED，让位给后面的节点。
- `BufferedAction(inst, target, ACTIONS.EAT)` 创建一次"挂着的"动作。当苔灵走到 target 附近，自动触发 EAT 状态机（12.4）。
- **必须先 `if inst.components.combat:HasTarget() then return nil end`** —— 避免战斗中走神去吃苔藓。

### 12.10.6 第六步：离屏正确处理（进阶 + 老手）

回顾 12.8：你写的每一个 `DoPeriodicTask` 都应该被 `OnEntitySleep` 取消。来给苔灵加一个"光环效果"——有同类在附近时移速 +20%——这个检查需要周期任务。

```lua
local AURA_RADIUS = 6
local AURA_TAGS = { "mossling" }

local function CheckAura(inst)
    if inst:IsAsleep() then return end  -- 第一道保险
    local x, y, z = inst.Transform:GetWorldPosition()
    local has_friend = false
    for _, v in ipairs(TheSim:FindEntities(x, 0, z, AURA_RADIUS, AURA_TAGS)) do
        if v ~= inst and not v:IsAsleep() then
            has_friend = true
            break
        end
    end

    inst.components.locomotor:SetExternalSpeedMultiplier(
        inst, "mossling_aura", has_friend and 1.2 or 1.0)
end

local function StartAuraCheck(inst)
    if inst._auratask == nil then
        inst._auratask = inst:DoPeriodicTask(1, CheckAura, math.random())
    end
end

local function StopAuraCheck(inst)
    if inst._auratask ~= nil then
        inst._auratask:Cancel()
        inst._auratask = nil
        inst.components.locomotor:RemoveExternalSpeedMultiplier(
            inst, "mossling_aura")
    end
end

inst.OnEntityWake = StartAuraCheck
inst.OnEntitySleep = StopAuraCheck
StartAuraCheck(inst)  -- 默认开启（spawn 时未睡眠）
```

**老手要点**：

1. **`StartAuraCheck` 内部判 `_auratask == nil`** —— 防止重复启动。
2. **`math.random()` 作为初始延迟** —— 50 只苔灵不会在同一帧扫描，避免 CPU 尖峰（参考 `Combat:OnEntityWake` 的 `period*math.random()`）。
3. **`SetExternalSpeedMultiplier` 加来源 key** —— `"mossling_aura"`——这样多个加速来源不会互相覆盖。
4. **`StopAuraCheck` 必须 `RemoveExternalSpeedMultiplier`** —— 否则苔灵睡着后还保留加速。
5. **不要直接 `inst.components.locomotor.runspeed = ...`** —— 这会写死，光环消失后无法恢复。

### 12.10.7 第七步：日夜行为切换（进阶）

苔灵白天会回家睡觉。给 brain 加一个"日夜分支"：

```lua
local function GoHomeAction(inst)
    local homePos = GetHomePos(inst)
    if homePos == nil then return nil end
    if inst:GetDistanceSqToPoint(homePos:Get()) < 4 then return nil end
    return BufferedAction(inst, nil, ACTIONS.WALKTO, nil, homePos)
end

function MosslingBrain:OnStart()
    local day = WhileNode(
        function() return TheWorld.state.isday end, "IsDay",
        PriorityNode({
            DoAction(self.inst, GoHomeAction, "go home", true),
            Wander(self.inst, GetHomePos, 3),
        }, 0.5)
    )

    local night = WhileNode(
        function() return not TheWorld.state.isday end, "IsNight",
        PriorityNode({
            DoAction(self.inst, FindFoodAction),
            Wander(self.inst, GetHomePos, WANDER_DIST),
        }, 0.5)
    )

    local root = PriorityNode(
    {
        BrainCommon.PanicTrigger(self.inst),
        WhileNode(function() return HasLowHealth(self.inst) end, "Wounded",
            RunAway(self.inst, "player", START_RUNAWAY_DIST, STOP_RUNAWAY_DIST)),
        ChaseAndAttack(self.inst, MAX_CHASE_TIME, MAX_CHASE_DIST),
        Leash(self.inst, GetHomePos, LEASH_MAX_DIST, LEASH_RETURN_DIST),
        day,
        night,
    }, 0.5)

    self.bt = BT(self.inst, root)
end
```

**进阶要点**：

- **`WhileNode + PriorityNode` 是分支模式** —— 每个分支在条件下展开成自己的优先级树。
- **白天分支没有 `Wander`**——只在家门口转，不会跑远。
- **夜晚分支才有 `FindFoodAction`**——白天回家睡觉，不到处吃东西。
- **`day` 和 `night` 互斥** —— `state.isday` 是布尔值，永远只有一个 WhileNode 会进入子树。

**`TheWorld.state.isday` 是 master 端可访问的 NetVar**——参见 `worldstate.lua`。可以监听 `clocktick` 或者 `phasechanged` 事件做更精细控制。

### 12.10.8 第八步：加上团队战斗（老手）

回顾 12.6：用 `TeamAttacker` + `TeamLeader` 组件。给苔灵加上：

```lua
-- 在 prefab 文件 master_postinit 中：
inst:AddComponent("teamattacker")
inst.components.teamattacker.team_type = "mossling_team"
inst.components.teamattacker.searchradius = 15
inst.components.teamattacker.max_team_size = 5
inst.components.teamattacker.min_team_size = 3
```

修改 `OnAttacked`/`OnNewTarget`，触发 `MakeTeam`：

```lua
local function MakeTeam(inst)
    if inst.components.teamattacker.teamleader == nil then
        inst.components.teamattacker:SearchForTeam()
    end
end

local function OnAttacked(inst, data)
    if data.attacker == nil
        or data.attacker == inst
        or data.attacker:HasTag("mossling") then
        return
    end
    inst.components.combat:SetTarget(data.attacker)
    inst.components.combat:ShareTarget(data.attacker, SHARE_TARGET_DIST, IsMossling, MAX_TARGET_SHARES)
    MakeTeam(inst)  -- ← 新加：拉队
end

local function OnNewTarget(inst, data)
    if data.target ~= nil then
        inst.components.combat:ShareTarget(data.target, SHARE_TARGET_DIST, IsMossling, MAX_TARGET_SHARES)
        MakeTeam(inst)
    end
end
```

修改 brain，加入"队伍模式"分支（在战斗前面）：

```lua
local function HasTeam(inst)
    return inst.components.teamattacker.teamleader ~= nil
end

function MosslingBrain:OnStart()
    local root = PriorityNode(
    {
        BrainCommon.PanicTrigger(self.inst),
        WhileNode(function() return HasLowHealth(self.inst) end, "Wounded",
            RunAway(self.inst, "player", START_RUNAWAY_DIST, STOP_RUNAWAY_DIST)),

        -- 队伍战斗：让 TeamAttacker:OnUpdate 接管移动 / 攻击
        WhileNode(function() return HasTeam(self.inst) end, "InTeam",
            PriorityNode({
                ChaseAndAttack(self.inst, MAX_CHASE_TIME, MAX_CHASE_DIST),
            }, 0.25)),

        -- 单兵战斗
        ChaseAndAttack(self.inst, MAX_CHASE_TIME, MAX_CHASE_DIST),

        Leash(self.inst, GetHomePos, LEASH_MAX_DIST, LEASH_RETURN_DIST),
        day,
        night,
    }, 0.5)

    self.bt = BT(self.inst, root)
end
```

**老手要点**：

1. **`teamleader` 是个虚拟实体**——12.6 提到，`TeamLeader` 组件运行在一个非网络化的内部实体上，不显示在地图上。
2. **`searchradius` 决定"我能找到多远的同伴组队"**——15 是经验值，太大会拉太远的同伴。
3. **`min_team_size` / `max_team_size`**——少于 3 只就不组队（避免"两只蘑菇组个 1 人队"），最多 5 只（避免"50 只苔灵围攻一个目标"的 CPU 灾难）。
4. **`TeamAttacker:OnUpdate` 自驱动**——12.6 说过，`OnUpdate` 会让我去 `formationpos`，这部分是**组件自己跑**的，**不需要 brain 写节点**。brain 只要不阻碍它就行——所以队伍模式分支只放 `ChaseAndAttack`（在 ATTACK 命令时执行）。
5. **`HasTeam` 检查**——只在确实有 leader 时进队伍模式，避免"组队失败但一直走队伍逻辑"。

### 12.10.9 第九步：性能与陷阱回顾（老手）

来检查这个 demo 哪些地方有性能/正确性陷阱：

#### 性能审计

| 位置 | 频率 | 代价 | 评价 |
|------|------|------|------|
| `NormalRetargetFn` | 2 秒 / 只 | 12 格 FindEntities | ✅ OK |
| `CheckAura` | 1 秒 / 只 | 6 格 FindEntities | ✅ OK |
| `FindFoodAction` (DoAction) | brain tick / 只 | 8 格 FindEntities | ⚠️ 需注意 |
| `OnAttacked` ShareTarget | 被打时 | 18 格 FindEntities | ✅ OK（事件驱动）|
| `MakeTeam` | 被打时 | 15 格 FindEntities | ✅ OK |

**潜在问题**——`DoAction(inst, FindFoodAction)` 在 PriorityNode 0.5 秒 sleeptime 下，每次 brain tick 都会调一次 `FindFoodAction`，**每只苔灵每 0.5 秒一次 8 格 FindEntities**——比 Retarget 还频繁。

**修正**：在 `FindFoodAction` 里加内部冷却：

```lua
local function FindFoodAction(inst)
    if inst.sg:HasStateTag("busy") then return nil end
    if inst.components.combat:HasTarget() then return nil end
    -- 5 秒一次内部冷却
    local now = GetTime()
    if (inst._lastfoodcheck or 0) + 5 > now then return nil end
    inst._lastfoodcheck = now

    local target = FindEntity(...)
    return target ~= nil and BufferedAction(...) or nil
end
```

#### 正确性回顾

逐条对应 12.9.10 / 12.8.8 的陷阱列表：

- ✅ **不打同类**——`OnAttacked` 和 `NormalRetargetFn` 都过滤 `mossling` 标签。
- ✅ **`OnEntitySleep` 取消周期任务**——`StopAuraCheck` 取消 `_auratask`。
- ✅ **`OnEntityWake` 重启**——`StartAuraCheck` 重新挂上。
- ✅ **`SetTarget` 走完整流程**——使用 `combat:SetTarget`，没直接赋值 `combat.target`。
- ✅ **`Wander` 有 fallback 点**——`GetHomePos` 返回 nil 时 `Wander` 自己用脚下位置。
- ⚠️ **没处理 `inst._lastfoodcheck` 在 entitysleep 时**——影响不大（GetTime() 是绝对时间）。
- ⚠️ **`KeepTargetFn` 距离检查没考虑船** —— 如果苔灵在岛上玩家上船跑远，不会自动放弃（因为还在 leash 范围内）。可以在 KeepTargetFn 中加入 `target:IsOnOcean()` 检查。

#### 升级建议（老手扩展）

如果要把这个 demo 扩展为真正可用的 mod，还应该做：

1. **加 `Sleeper` 组件**：白天回家后真的"睡觉"（GoToSleep），可以被吹箭打到延长睡眠时间。详见 12.8.1（注意这是和 EntitySleep 完全不同的概念）。
2. **加 `homeseeker` 组件**：自动管理"家"——蘑菇巢摧毁时自动清空 `home`，重生时自动设回。
3. **加 NetVars 同步状态**——让客户端能看到苔灵的"队伍模式"状态，做特殊视觉效果。
4. **加 `localization` 字符串**：苔灵的检视文本、talker 喊话。
5. **加 `SaveGame` 钩子**：保存/读取 home 位置。

### 12.10.10 全章总结

恭喜——你已经把第十二章九大模块**全部串起来**了。现在我们用一张表回顾整个苔灵 demo 用到了哪些章节的内容：

| 模块 | 章节 | demo 中的体现 |
|------|------|--------------|
| **行为树框架** | 12.1 | `BT(inst, root)`、`PriorityNode + WhileNode + DoAction + Wander` 组合 |
| **节点类型** | 12.2 | `WhileNode("Wounded", ...)`、`PriorityNode({...}, sleeptime)`、`ChaseAndAttack`、`Leash`、`Wander`、`RunAway` |
| **`braincommon`** | 12.3 | `BrainCommon.PanicTrigger(inst)` |
| **BufferedAction** | 12.4 | `BufferedAction(inst, target, ACTIONS.EAT)` 在 `FindFoodAction` 中 |
| **AI 模式** | 12.5 | 巡逻（Wander）、追击（ChaseAndAttack）、逃跑（RunAway）、归巢（Leash + GoHomeAction）|
| **团队战斗** | 12.6 | `TeamAttacker` 组件 + `MakeTeam` + `HasTeam` 分支 |
| **寻路系统** | 12.7 | `IsOnPassablePoint(true)` 检查食物位置 + `locomotor` 自动寻路 |
| **离屏优化** | 12.8 | `OnEntitySleep` 取消 `_auratask` + `OnEntityWake` 重启 + `IsAsleep()` 内部判断 |
| **目标选择** | 12.9 | `SetRetargetFunction(2, NormalRetargetFn)` + `SetKeepTargetFunction` + `SuggestTarget` + `ShareTarget` + `OnAttacked` + `OnNewTarget` 监听 |

**最重要的写 mod 心法**（按重要性排序）：

1. **先列优先级表，再写代码**——先想清楚"这只怪在什么状态下做什么"，再翻译成 PriorityNode。
2. **复用 `braincommon` 比自己写好**——`PanicTrigger` 这种通用模块千万不要重写，否则 mod 兼容性会很差。
3. **永远用 `combat:SetTarget`、永远用 `combat:DropTarget`**——别直接改 `combat.target` 字段。
4. **每个 `DoPeriodicTask` 都要在 `OnEntitySleep` 取消**——这是离屏优化的铁律。
5. **`FindEntity` 必须传 must tags**——否则全图扫描，性能崩溃。
6. **战斗共享要过滤同类**——否则会出现"同伴互殴"。
7. **`BufferedAction` 不能在 brain 节点里 `:Do()`**——交给 `DoAction` 节点处理，让 `LocoMotor` 走完整流程。
8. **测试时一定要用 `c_spawn` 多刷几只**——单个测试看不出团队战斗、ShareTarget、性能问题。

**调试技巧**：

```lua
-- 看一只苔灵的状态
local inst = c_select()
print("target:", inst.components.combat.target)
print("hasleader:", inst.components.teamattacker.teamleader ~= nil)
print("asleep:", inst:IsAsleep())
print("home:", inst.components.knownlocations:GetLocation("home"))
print("brain:", inst.brain)

-- 看 brain 当前状态
print(tostring(inst.brain.bt))
```

`tostring(inst.brain.bt)` 会输出整棵行为树的当前状态——RUNNING / SUCCESS / FAILED 都会标在节点旁边，配合 12.1 学的"四态机"，调试问题非常直观。

---

至此，**第十二章——AI 系统全部完成**。我们从最底层的 BehaviourTree 起步，经历了节点类型、通用模块、动作系统、AI 模式、团队战斗、寻路、离屏优化、目标选择，最后用一个完整的苔灵 demo 把所有内容串起来。读完本章，你应该可以：

- ✅ 看懂任何官方生物 brain 文件的结构
- ✅ 设计自己的怪物行为优先级
- ✅ 写出能正确处理离屏、寻路、团队战斗的 AI
- ✅ 调试"为什么我的怪发呆 / 行为错乱 / 性能爆炸"问题
- ✅ 在不破坏现有系统的前提下，给原版 NPC 注入新行为