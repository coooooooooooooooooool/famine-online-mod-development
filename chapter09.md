# 第9章 网络同步机制

## 9.1 NetVar 系统——客户端需要知道什么（全部 NetVar 类型详解）

### 本节导读

第 8 章我们把"事件系统"讲透了——但**所有事件都是单端的**：服务端推到服务端的实体上、客户端推到客户端的实体上。**事件本身不会跨网络传播**——主机端推 `"hungerdelta"`，**客户端那边不会自动收到**。

但联机版 mod 一定会遇到**这种需求**：

- 服务端给玩家**加了 buff**——客户端的 HUD 上的 buff 图标得**立刻显示**
- 服务端**改变了某物品的颜色 / 状态**——客户端必须看到一致的外观
- 服务端**触发了一次特效**——客户端要在那个位置播放粒子

**这些跨主机/客户端的同步**需要走**网络变量**（NetVar）系统——它是 Klei 设计的**带"脏标记"的状态广播机制**：服务端 `:set(val)` → 引擎自动通过网络通知客户端 → 客户端通过 `:value()` 读到新值 + 收到 dirty 事件触发响应。

> **新手**从 9.1.1-9.1.3 起步——理解 NetVar 的存在意义、8 种类型速查、三件套 API（创建 / set / value）+ dirty 事件；**进阶读者**继续看 9.1.4-9.1.6，深入 `SetPristine` / `ismastersim` 分支机制、`net_event` 的"无值纯通知"用法、dirty 事件命名约定与多 NetVar 聚合到同一个 dirty 事件；**老手**跳到 9.1.7-9.1.8，理解 NetVar 的**字节预算 / 性能优化**、6 个网络同步最容易踩的坑——尤其是**"客户端调 set 报错"**和**"NetVar 在 SetPristine 之后才创建导致客户端永远收不到"**这两个翻车场景。

---

### 9.1.1 快速入门：从一只"眼睛会发光的克劳斯怪"看 NetVar 是什么

#### 第一步：在游戏里观察"主机改状态、客户端立刻看到"

打开冬季 boss "克劳斯怪"（Clay Warg）的源码——`scripts/prefabs/warg.lua` 第 848 行：

```848:849:scripts/prefabs/warg.lua
                inst._eyeflames = net_bool(inst.GUID, "claywarg._eyeflames", "eyeflamesdirty")
				inst:ListenForEvent("eyeflamesdirty", Clay_OnEyeFlamesDirty)
```

**这两行做了什么**：

- 第 848 行：**创建一个 `net_bool` 网络变量**——挂在 inst 上，名字 `claywarg._eyeflames`，dirty 事件名 `eyeflamesdirty`。
- 第 849 行：**所有人**（主机 + 客户端）监听 `"eyeflamesdirty"` 事件——一旦这个网络变量在主机端被 `:set(true)`，**所有客户端都会触发 `eyeflamesdirty` 事件**——每个客户端各自调一次 `Clay_OnEyeFlamesDirty(inst)` —— 在那里去执行**眼睛发光的特效**。

**典型链路**：

```
[主机端] AI 状态机判断 "该眼睛发光了"
    ↓
[主机端] inst._eyeflames:set(true)
    ↓
[引擎] 自动通过网络协议同步给所有客户端
    ↓
[客户端 A] eyeflamesdirty 事件触发 → Clay_OnEyeFlamesDirty 调用 → 启动客户端特效
[客户端 B] 同上
[客户端 C] 同上
```

> **核心结论**：**NetVar 是"网络层的状态值"——主机 set，所有客户端通过 dirty 事件感知变化、通过 :value() 拿到值**。比起 RPC 的"主动调用"模式，NetVar 是**"被动观察"模式**——更适合"持续状态"。

#### 第二步：玩家自己的"血条 / 饥饿条"也是 NetVar

打开 `scripts/prefabs/player_classified.lua`——这是**每个玩家专属的 classified 实体**——存放仅这个玩家的客户端能看到的网络变量：

```1341:1349:scripts/prefabs/player_classified.lua
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

**仅 health 一个组件就有 8 个 NetVar** ——当前血量、最大血量、惩罚、是否在火伤、是否睡眠回血、是否治疗 pulse 上升/下降、月炽 flags。**每一项**主机端的 health 组件改变值时——通过 `health_replica.lua` 中的 `self.classified:SetValue(...)` 写入对应 NetVar——客户端**HUD 通过这些 NetVar 实时画血条**。

**关键观察**：
- **第 1346 行 `issleephealing`** 没传 dirty 事件名——**客户端不会被通知"变化"**——但客户端**仍可以读 `:value()`** 拿当前值。**仅"读快照"的 NetVar 不需要 dirty 事件**。
- **第 1347-1348 行 `ishealthpulseup/down`** 共享同一个 dirty 事件 `"healthdirty"` —— **多个 NetVar 可以共用一个 dirty 事件名**——9.1.6 节专讲。

#### 第三步：服务端 → 客户端的"单向广播"

> **NetVar 是单向的**——服务端 `:set(val)` 同步到客户端；**客户端 `:set(val)` 不会传回主机**——会被引擎拒绝（且通常会报错或被静默忽略）。

如果你需要**客户端 → 服务端**的通信——用 **RPC 系统**（9.2 节）。**NetVar 仅服务于"服务端持续广播状态给客户端"** 这一个方向。

> 唯一的例外是 `:set_local(val)` ——**仅在客户端设值，不向其他端广播**——通常只用于客户端 UI 自己内部的状态预测（罕见，95% 用不上）。

---

### 9.1.2 快速入门：8 种 NetVar 类型速查

#### 第一步：完整类型表

| 类型 | 字节数 | 范围 | 典型用例 |
| --- | --- | --- | --- |
| `net_bool` | 1 bit | true / false | 标志、开关（"是否发光"、"是否在火伤"） |
| `net_tinybyte` | 3 bit | 0–7 | 小枚举（"3 段歌曲选了哪段"、"4 种皮肤选了哪种"） |
| `net_smallbyte` | 6 bit | 0–63 | 中等枚举（百分比 0–63、3 位数计数） |
| `net_byte` | 8 bit | 0–255 | 大枚举或归一化百分比（健康惩罚 0-200） |
| `net_shortint` | 16 bit | -32768–32767 | 中等数值，可负 |
| `net_ushortint` | 16 bit | 0–65535 | 中等数值，仅正（**血量、饥饿、精神当前/最大值常用**） |
| `net_int` | 32 bit | -2³¹–2³¹-1 | 大数（GUID、ID） |
| `net_uint` | 32 bit | 0–2³²-1 | 大数仅正 |
| `net_float` | 32 bit | IEEE 754 | 浮点（**坐标、温度等连续值**） |
| `net_smallfloat` | 16 bit | 半精度 | 浮点低精度（视觉旋转角度等） |
| `net_string` | 变长 | UTF-8 | 字符串（**慎用**：变长占带宽） |
| `net_hash` | 32 bit | 字符串 hash | 短期识别符（prefab 名等） |
| `net_entity` | 实体 GUID + 版本号 | 任意实体 | 引用别的实体（"我的目标"、"我的所有者"） |
| `net_event` | 0 bit | （无值） | 纯通知，set 一次触发一次 dirty 事件 |

#### 第二步：选型决策

**问 1：要传什么？**
- 布尔 → `net_bool`
- 小整数 0–7 → `net_tinybyte`
- 整数 0–255 → `net_byte`
- 整数 0–65535（血量、饥饿）→ `net_ushortint`
- 浮点（坐标、温度）→ `net_float`
- 字符串（慎用）→ `net_string`
- 实体引用（目标、所有者）→ `net_entity`
- **只通知，不传值** → `net_event`

**问 2：能用更小的吗？**
- 血量上限 = 200 → `net_byte`（255 容纳）够用 → 比 `net_ushortint` 省 1 字节
- 0/1 二选一 → `net_bool`（1 bit）远胜 `net_byte`（8 bit）

**问 3：精度需要多高？**
- 屏幕坐标显示用，1 像素精度 → `net_smallfloat` 半精度可能够
- 物理位置 / 时间戳 → `net_float`

#### 第三步：实战类型选型对比

回到 player_classified.lua 第 1341-1349 行——**Klei 的取舍很讲究**：

| 字段 | 用了哪种 | 为什么 |
| --- | --- | --- |
| `currenthealth` / `maxhealth` | `net_ushortint`（16 bit, 0–65535） | 血量值可能超过 255（疯子模式下打 boss），但不会到 65535 |
| `healthpenalty` | `net_byte`（0–255） | 惩罚值乘 200 后是整数 0–200——`net_byte` 够用 |
| `istakingfiredamage` | `net_bool` | 是/否——1 bit 最省 |
| `lunarburnflags` | `net_tinybyte`（0–7） | 月炽 flags 是 3 位 bitmask——`net_tinybyte` 正好 |

**注意**：**总字节数 = 16+16+8+1+1+1+1+1+3 ≈ 6 字节**——**所有玩家每变一次状态就传 6 字节**。**类型选小**对带宽有显著影响。

---

### 9.1.3 快速入门：NetVar 三件套（创建 / set / value） + dirty 事件

#### 第一步：创建 NetVar

```lua
-- 通用模板
inst.varname = net_TYPE(parent_guid, "namespace.fieldname", "dirty_event_name")
```

**3 个参数**：
1. **`parent_guid`** —— 通常是 `inst.GUID`（NetVar 挂在 inst 上）
2. **`name`** —— 调试用名称（**全局唯一推荐**："prefab.field"——比如 `"player.maxhealth"`、`"warg._eyeflames"`）
3. **`dirty_event_name`**（可选）—— 值变化时**所有端**触发的事件名；**省略 = 仅写值不通知**

#### 第二步：set 值（仅服务端）

```lua
inst._eyeflames:set(true)         -- 写值；如果值真的变了，自动同步并触发 dirty
inst._eyeflames:set_local(true)   -- 仅本机写值，不同步
```

**关键**：
- **`:set(val)`** 是**服务端调用** —— 客户端调会被忽略（且 mod 应避免，可能崩溃）
- **`:set_local(val)`** 客户端也能调——但**不会跨端同步**——主要用于客户端 UI 临时预测

**服务端调 `set` 的副作用**：
1. 引擎记录新值
2. 下一帧通过网络协议同步给所有客户端
3. **每端**（含主机自己）触发对应 dirty 事件

#### 第三步：value 读值（任何端）

```lua
local val = inst._eyeflames:value()
```

**`:value()` 在两端都有效**——主机/客户端都能读当前值。

#### 第四步：dirty 事件——客户端响应入口

```lua
local function OnEyeFlamesDirty(inst)
    if inst._eyeflames:value() then
        inst:DoEyeFlameFx()
    else
        inst:RemoveEyeFlameFx()
    end
end

inst:ListenForEvent("eyeflamesdirty", OnEyeFlamesDirty)
```

**关键**：
- **dirty 事件**和普通事件一样用 `ListenForEvent` 监听
- **第一参数是 inst（自己）**，**没有 data 字段**——值要自己 `:value()` 去读
- **事件在两端都触发**（主机自己 set 也会触发自己的 dirty）——所以 listener 写一份就行

#### 第五步：完整 5 行代码示范

```lua
-- prefab fn 里
inst.entity:AddNetwork()        -- 必须！否则 NetVar 不工作
inst.boss_phase = net_byte(inst.GUID, "myboss.phase", "phasedirty")

inst:ListenForEvent("phasedirty", function(inst)
    inst:UpdatePhaseVisual(inst.boss_phase:value())
end)

-- 在主机端 AI 中
inst.boss_phase:set(2)
```

> **新手 MVP 检查点**：能写出"`net_xxx` 创建 + `ListenForEvent dirty + :value()` 读 + 主机 `:set()` 写"四件套——掌握 80% NetVar 用法。

---

### 9.1.4 进阶：`SetPristine` / `ismastersim` 分支与"必须先创建后冻结"

#### 第一步：先看 prefab fn 的标准模板

```lua
local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()        -- ★ 必须有这一行

    inst:AddTag("xxx")
    -- ...

    -- ★ NetVar 创建必须在 SetPristine 之前
    inst.boss_phase = net_byte(inst.GUID, "myboss.phase", "phasedirty")
    inst._eyeflames = net_bool(inst.GUID, "myboss.eyeflames", "eyeflamesdirty")

    if not TheNet:IsDedicated() then
        inst:ListenForEvent("phasedirty", OnPhaseDirty)
        inst:ListenForEvent("eyeflamesdirty", OnEyeFlamesDirty)
    end

    inst.entity:SetPristine()       -- ★ 客户端在此就停止了

    if not TheWorld.ismastersim then
        return inst
    end

    -- ★★★ 仅服务端往下跑：组件、SG、brain、所有业务
    inst:AddComponent("health")
    inst:SetStateGraph("SGwarg")
    -- ...

    return inst
end
```

**关键 3 段**：

1. **共享部分**（两端都跑）：`AddNetwork`、所有 `net_xxx` 创建、客户端要监听的 dirty 事件
2. **`SetPristine()`**：**客户端在这里返回**——客户端的 prefab fn 到此结束
3. **服务端独有**：组件、SG、brain、业务逻辑

#### 第二步：为什么 NetVar 创建**必须在 SetPristine 之前**？

`SetPristine()` 内部告诉引擎：**"这个 entity 的网络结构已经冻结、可以序列化给客户端了"**。**之后再加 NetVar 客户端就同步不到**——这是新手最常翻车的地方。

**翻车症状**：
- 在 `master_postinit`（即 `if not TheWorld.ismastersim then return inst end` **之后**）创建 NetVar
- 主机端这个 NetVar 存在并能 set
- **客户端那个 inst 上根本没有这个 NetVar 字段**——`inst.xxx:value()` 报 nil 索引错误

**修复**：所有 `net_xxx` 必须在 `inst.entity:SetPristine()` **之前**，且**必须在两端都跑的代码段**中。

#### 第三步：`AddNetwork` 是必备前提

`inst.entity:AddNetwork()` —— 给实体附加"网络组件"。**没这一行**：
- entity 不能跨网络同步
- NetVar 报错或被静默忽略
- 实体本身也无法被客户端"看到"

**所有需要联机的 prefab 必须有这一行**——通常**第一行 transform 之后立刻调**。

#### 第四步：`TheWorld.ismastersim` 与 `TheNet:IsDedicated()` 的区别

```lua
TheWorld.ismastersim  -- 当前进程是不是"主机模拟"——本机即主机
TheNet:IsDedicated()  -- 当前进程是不是"独立服务器"——专门的 dedicated server 进程
TheNet:GetIsServer()  -- 是不是服务端（包括主机和 dedicated）
TheNet:GetIsClient()  -- 是不是客户端
```

**4 种部署模式**：

| 模式 | ismastersim | IsDedicated | 含义 |
| --- | --- | --- | --- |
| 单机 | true | false | 单机版/单人联机 |
| 主机端联机（host） | true | false | 玩家自己当主机 |
| 客户端联机（join） | false | false | 玩家加入别人主机 |
| Dedicated 服务器 | true | true | 专门的服务器进程，无 GUI |

**典型用法**：
- `if TheWorld.ismastersim then` —— 服务端独占代码
- `if not TheWorld.ismastersim then return inst end` —— 客户端到此返回（`SetPristine` 之后的标准模板）
- `if not TheNet:IsDedicated() then` —— 跳过 UI/特效在 dedicated 上

> **9.4 节**会专门把这 4 个判断的边界讲清楚。

---

### 9.1.5 进阶：`net_event` —— "无值纯通知"

#### 第一步：什么是 net_event？

```lua
inst.triggerfx = net_event(inst.GUID, "voidcloth_umbrella.triggerfx")
```

**特点**：
- **不存储值** —— 没有 `:value()`
- **每次 `:push()` 都触发对应 dirty 事件**
- 等价于"通过网络推一次事件"

**用法对比**：

| 普通 NetVar | net_event |
| --- | --- |
| 服务端 `:set(val)` | 服务端 `:push()` |
| 仅当 val 变化时同步 | 每次 push 都同步 |
| 客户端 `:value()` 读当前值 | 客户端没有"当前值"概念，只看 dirty 事件 |

#### 第二步：典型用例

`scripts/prefabs/voidcloth_umbrella.lua:369`：

```lua
inst.triggerfx = net_event(inst.GUID, "voidcloth_umbrella.triggerfx")

inst:ListenForEvent("voidcloth_umbrella.triggerfx", function(inst)
    SpawnPrefab("voidcloth_umbrella_fx").Transform:SetPosition(inst.Transform:GetWorldPosition())
end)

-- 主机端调
inst.triggerfx:push()
```

**用法**：服务端"放出特效"——push 一下——所有客户端都收到 dirty 事件——**自己生成本地特效**。**特效本身不存在网络上**——每个客户端**自己 SpawnPrefab 一个本地特效**。

> **设计哲学**：**特效 / 音效不应跨网络传输** —— 通过 net_event 通知 + 客户端各自播放本地特效。

#### 第三步：为什么不用 RPC？

**RPC 也能做"通知客户端"** —— 但 net_event 的**优势**：
- **跟随实体生命周期** —— 实体销毁时 net_event 自动清理
- **网络协议层面更轻** —— 0 字节 vs RPC 的协议开销
- **不需要在 modmain 注册** —— RPC 必须 `AddModRPCHandler`

**RPC 的优势**：
- 可以**带任意 data**
- 不挂在某个实体上 —— 适合**全局事件**

> **判断标准**：**通知和某个实体绑定 + 不带 data → net_event；带 data 或全局 → RPC**。

---

### 9.1.6 进阶：dirty 事件命名约定 + 多 NetVar 聚合到同一个 dirty 事件

#### 第一步：命名约定

观察 player_classified.lua 的命名：

```lua
inst.currenthealth = net_ushortint(inst.GUID, "health.currenthealth", "healthdirty")
inst.maxhealth = net_ushortint(inst.GUID, "health.maxhealth", "healthdirty")
inst.healthpenalty = net_byte(inst.GUID, "health.penalty", "healthdirty")
```

**3 个 NetVar 共用一个 dirty 事件 `"healthdirty"`** —— **任何一个变化都触发同一个事件**。

**为什么这样做？**
- HUD 血条**任何一项变化都需要重绘**——一个 listener 一次性处理"血量变了 / 最大血量变了 / 惩罚变了"——逻辑统一
- 节省 listener 数量

**反例**：每个 NetVar 配一个独立 dirty 事件——3 个 listener 分别处理——**每次变化都走 3 次 listener** —— 性能浪费。

#### 第二步：dirty 事件命名规范

**Klei 风格**：`<systemname>dirty`——比如 `healthdirty`、`hungerdirty`、`sanitydirty`。

**mod 风格建议**：`mymod_<systemname>_dirty`——避免与原版/其他 mod 冲突。

#### 第三步：复合状态时的处理

**典型场景**：3 个 NetVar 共用 dirty——但 listener 必须知道**到底哪个变了**——怎么办？

**方案 A**：在 listener 里**逐个比较** old / new：

```lua
local function OnHealthDirty(inst)
    local h = inst.currenthealth:value()
    local m = inst.maxhealth:value()
    local p = inst.healthpenalty:value()
    
    if inst._lasthealth ~= h then
        -- 处理血量变化
    end
    if inst._lastmax ~= m then
        -- 处理最大血量变化
    end
    -- ...
    inst._lasthealth = h
    inst._lastmax = m
    -- ...
end
```

**方案 B**：**用独立 dirty**（适合差异处理逻辑差很多的场景）：

```lua
inst.iswet = net_bool(inst.GUID, "iswet", "wetdirty")           -- 单独
inst.iscold = net_bool(inst.GUID, "iscold", "colddirty")        -- 单独
```

**判断标准**：**逻辑相似/相互关联**用聚合；**逻辑独立差异大**用独立。

#### 第四步：多 NetVar 同帧同 dirty 的"合并"

**精妙细节**：**同一帧同一 dirty 事件被 set 多次时，dirty 事件只触发一次**（引擎层去重）。**避免 listener 被同帧调多次**。

**对开发者影响**：服务端可以**一次操作**改 5 个 NetVar——客户端**只触发一次** dirty 事件——**性能良好且行为一致**。

---

### 9.1.7 老手进阶：NetVar 的字节预算与性能优化

#### 第一步：网络带宽预算

饥荒联机版的网络带宽**有上限**——通常每秒每个客户端 ~500 KB。**所有 entity / NetVar / RPC 共享这个预算**。

**典型 entity 的 NetVar 占用**：

| Entity | NetVar 数量 | 总字节 |
| --- | --- | --- |
| `pigman` | ~5 | ~10 字节 |
| `player_classified`（每玩家） | ~80 | ~150 字节 |
| `worldoverseer` | ~30 | ~60 字节 |

**全图 200 个 NPC × 平均 5 NetVar × 平均 1 字节 = 1KB**——**问题不在静态值，而在变化频率**。

#### 第二步：变化频率才是性能关键

**网络协议只在 NetVar 变化时同步**——**永远不变的 NetVar 几乎零开销**。**问题来自高频变化**：

- ❌ `inst._currenthp = net_ushortint(...)` 每帧 set 一次（30 Hz）—— 30 帧 × 2 字节 × 200 个怪 = **每秒 12 KB**
- ✅ `inst._currenthp = net_ushortint(...)` 仅在血量变化时 set —— 每秒可能 0~3 字节

**修复反例的方法**：
- **服务端用 lua local 存当前值**，**只在变化时 set NetVar**
- **如果业务真需要每帧同步**——考虑改用客户端预测（9.5 节）

#### 第三步：NetVar 数量的"连乘"问题

如果 mod 给 100 种 prefab 都加 5 个 NetVar——**100 × 5 = 500 个不同 NetVar 类型**——**协议表膨胀**——**握手成本**也增加。

**优化**：
- **共用 NetVar**——在 component_postinit 里给某基础 prefab 类加，而不是每个 prefab 单独
- **bitmask 替代多 bool**——8 个 bool → 1 个 net_byte（8 个 bit）

#### 第四步：典型优化案例

**优化前**：

```lua
inst.is_wet     = net_bool(inst.GUID, "buff.wet",     "buffdirty")
inst.is_cold    = net_bool(inst.GUID, "buff.cold",    "buffdirty")
inst.is_burning = net_bool(inst.GUID, "buff.burning", "buffdirty")
inst.is_freezing= net_bool(inst.GUID, "buff.freezing","buffdirty")
inst.is_dazed   = net_bool(inst.GUID, "buff.dazed",   "buffdirty")
inst.is_invul   = net_bool(inst.GUID, "buff.invul",   "buffdirty")
-- 6 个 NetVar，6 bit
```

**优化后**：

```lua
inst.buffflags = net_byte(inst.GUID, "buff.flags", "buffdirty")  -- 1 个 NetVar，8 bit
-- 用位运算
local function SetWet(inst, v)
    local f = inst.buffflags:value()
    f = v and bit.bor(f, 1) or bit.band(f, bit.bnot(1))
    inst.buffflags:set(f)
end
local function IsWet(inst) return bit.band(inst.buffflags:value(), 1) ~= 0 end
```

**好处**：
- 协议表只多 1 个类型（不是 6 个）
- 同步时**所有 buff 一次同步完**（旧版每改一个状态都同步一次，多个状态变化要同步 N 次）

**坏处**：代码更复杂——但**对 boss / 玩家这种"buff 多"的实体，绝对值得**。

---

### 9.1.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：客户端调 `:set()`

**症状**：客户端代码调 `inst.varname:set(val)` —— 报错或静默失败、其他客户端看不到变化。  
**原因**：NetVar 是**单向广播（服务端→客户端）**——客户端不能写。  
**修复**：客户端要"通知服务端改"用 RPC。**写 NetVar 之前永远先判**：

```lua
if TheWorld.ismastersim then
    inst.varname:set(val)
end
```

#### 陷阱 2：在 SetPristine 之后创建 NetVar

**症状**：客户端 `inst.varname:value()` 报 nil 错误——**主机端却正常**——非常难定位。  
**原因**：见 9.1.4 第二步——**NetVar 必须在 SetPristine 之前创建**。  
**修复**：**所有 `net_xxx` 写在 prefab fn 的 `if not TheWorld.ismastersim then return inst end` 之前**。

#### 陷阱 3：忘记 `inst.entity:AddNetwork()`

**症状**：NetVar 创建了但客户端永远看不到值——主机改了客户端读到默认值（false / 0）。  
**原因**：实体没有网络组件——NetVar 无效。  
**修复**：**联机 prefab 永远第一行加上**：

```lua
inst.entity:AddNetwork()
```

#### 陷阱 4：dirty 事件名字与 NetVar name 重名

```lua
inst.health = net_ushortint(inst.GUID, "health", "health")  -- 危险！
```

**症状**：`ListenForEvent("health", fn)` 触发不一致——**因为 NetVar 名字被解析成事件名时和别的事件冲突**。  
**修复**：**dirty 事件名永远以 `dirty` 结尾**——`"healthdirty"`、`"phasedirty"`。

#### 陷阱 5：NetVar 名字与别的实体重复

**症状**：调试时报警告"重复 NetVar 名字"——某些值被覆盖。  
**原因**：name 不唯一 —— 引擎用 name 做调试索引。  
**修复**：永远 `<prefab_name>.<field>`：

```lua
inst.eyeflames = net_bool(inst.GUID, "claywarg._eyeflames", "eyeflamesdirty")
                                       ↑ 加 prefab 前缀
```

#### 陷阱 6：在 dirty 事件 listener 里直接访问 components

**症状**：联机客户端 dirty listener 里 `inst.components.health` 报 nil。  
**原因**：客户端**没有 components**——只有 replica。  
**修复**：dirty listener 里**只用 NetVar 自身和 replica**——绝不碰 components：

```lua
inst:ListenForEvent("healthdirty", function(inst)
    local h = inst.currenthealth:value()  -- ✅
    -- inst.components.health.currenthealth  -- ❌ 客户端没有
end)
```

#### 设计经验三条

**经验 ①：把"通信结构"定型在最早**

NetVar 是 prefab 网络协议的一部分——**改变 NetVar 类型/数量会影响 mod 兼容性**——**用旧版本 mod 的客户端连不上新版本服务器**。**第一次设计时多花 5 分钟想清楚结构**——比上线后改成本低 100 倍。

**经验 ②：状态值用 NetVar、变化通知用 net_event、双向交互用 RPC**

| 需求 | 用 |
| --- | --- |
| "客户端要能查到当前血量" | NetVar |
| "客户端要在某事件时播特效" | net_event |
| "客户端按按钮要让服务端响应" | RPC |
| "服务端给客户端独占信息（仅自己看血条）" | classified entity（9.3 节） |

**经验 ③：测试 mod 联机要在 dedicated 上**

**主机端测试**和 **dedicated 服务器**测试**行为不一定一致**——主机端 `TheWorld.ismastersim == true` 且**主机本身是个客户端**——很多 NetVar 路径不暴露。**真正的联机问题**只在 dedicated 上才显示。

**dev 推荐流程**：
1. 单机/主机端**初步开发**——逻辑跑通
2. dedicated 服务器**联机测试**——验证 NetVar / RPC / classified 路径
3. 多客户端测试——验证状态一致性

---

### 9.1.9 小结

**NetVar 一句话总结**：**服务端写 → 客户端通过 dirty 事件通知 + :value() 读快照 → 实现持续状态广播**。这是 mod 联机最基础的同步机制。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 创建 NetVar | `inst.x = net_byte(inst.GUID, "ns.x", "xdirty")` |
| 服务端写 | `inst.x:set(val)` |
| 任何端读 | `local v = inst.x:value()` |
| 监听变化 | `inst:ListenForEvent("xdirty", OnXDirty)` |
| 创建无值通知 | `inst.fx = net_event(inst.GUID, "ns.fx")` |
| push 通知 | `inst.fx:push()` |
| 仅本机改值 | `inst.x:set_local(val)` |

**8 种类型选型**

- bool → `net_bool`（1 bit）
- 0–7 → `net_tinybyte`
- 0–63 → `net_smallbyte`
- 0–255 → `net_byte`
- 0–65535 → `net_ushortint`
- 浮点 → `net_float` / `net_smallfloat`
- 字符串（慎用）→ `net_string`
- 实体引用 → `net_entity`
- 纯通知 → `net_event`

**6 个陷阱排雷顺序**

1. 客户端 set → 加 ismastersim 判
2. SetPristine 后创建 → 在前面创建
3. 忘记 AddNetwork → 加上
4. dirty 名重名 → 永远以 dirty 结尾
5. NetVar name 重复 → 加 prefab 前缀
6. listener 访问 components → 用 replica

**3 条设计经验**

- ① **结构早定型**——避免 mod 升级时破坏兼容
- ② **NetVar / net_event / RPC 各司其职**——见 8.1.8 决策表
- ③ **dedicated 测试不能省**

> **下一节预告**：9.2 节我们打开 **RPC 系统** —— 客户端 → 服务端的通信机制——`AddModRPCHandler` / `SendModRPCToServer` 等 API、**RPC 的"信任边界"问题**（客户端伪造请求）、为什么 Klei 的 RPC 都要做合法性校验。读完 9.2，你将能写出"客户端按按钮触发服务端业务"这种联机 mod 的核心交互——和 NetVar 配合，**双向通信链路完整打通**。

## 9.2 RPC 系统——客户端向服务端发请求（networkclientrpc.lua）

### 本节导读

9.1 节我们把 NetVar **服务端 → 客户端**的状态广播讲透了。但 mod 联机里**还有一半通信**——**客户端 → 服务端**：

- 玩家在客户端**按下某 mod 自定义按钮**——业务逻辑必须在服务端执行（修改世界状态、扣除资源、生成实体）
- 客户端**输入聊天命令**——需要服务端验证权限并执行
- 客户端**请求服务端做某事**——比如打开某 UI、查询某数据

**NetVar 不能解决这个**——它是单向的（服务端 → 客户端）。**RPC 系统**就是为这种**反向通信**设计的：客户端**调用一个函数 → 数据通过网络发到服务端 → 服务端的 handler 接收数据并执行业务**——典型的"远程过程调用"模式。

但**RPC 比 NetVar 危险得多**——**客户端是不可信的**：恶意玩家可能用 mod 修改 / 伪造 RPC，发送非法参数想"作弊"。所以 RPC handler **第一件事永远是参数校验**——这是和单端逻辑**完全不同**的编程范式。

> **新手**从 9.2.1-9.2.3 起步——理解 RPC 的存在意义、`AddModRPCHandler` / `SendModRPCToServer` / `GetModRPC` 三件套、完整的注册和调用流程；**进阶读者**继续看 9.2.4-9.2.6，深入"信任边界"问题、`AddClientModRPCHandler` 反向通道、`AddShardModRPCHandler` 跨 shard 通信、限流和 USERID_RPCS 机制；**老手**跳到 9.2.7-9.2.8，看 RPC vs NetVar vs net_event 的决策矩阵、6 个最容易踩的坑。

---

### 9.2.1 快速入门：从一次"远程左键"看 RPC 是什么

#### 第一步：客户端按左键到底发生了什么？

7.5 节讲过——客户端按左键，PlayerController 走完客户端预测后，**还要把这个动作发给服务端实际执行**。**怎么发？走 RPC**。

打开 `scripts/networkclientrpc.lua` 第 61 行：

```61:111:scripts/networkclientrpc.lua
    LeftClick = function(player, action, x, z, target, isreleased, controlmods, noforce, mod_name, platform, platform_relative, spellbook, spell_id)
		if not (	(	--these are either all nil
						action == nil and
						x == nil and
						z == nil and
						target == nil and
						isreleased == nil and
						controlmods == nil and
						noforce == nil and
						mod_name == nil and
						platform == nil and
						platform_relative == nil and
						spellbook == nil and
						spell_id == nil
					) or
					(	--or all validated
						checknumber(action) and
						checknumber(x) and
						checknumber(z) and
						optentity(target) and
						optbool(isreleased) and
						optnumber(controlmods) and
						optbool(noforce) and
						optstring(mod_name) and
						optentity(platform) and
						checkbool(platform_relative) and
						optentity(spellbook) and
						optuint(spell_id)
					)
				)
		then
            printinvalid("LeftClick", player)
            return
        end
		local playercontroller = player.components.playercontroller
		if playercontroller ~= nil then
			if action == nil then
				playercontroller:OnRemoteLeftClick()
				return
			end
```

**关键观察**：
1. **第一个参数 `player`** —— 服务端**自动注入**——是发送 RPC 的玩家实体（**客户端无法伪造这个**）
2. **后续参数** —— 客户端发送过来的——可能被恶意篡改
3. **第 76-89 行**：**所有参数都做类型校验**——`checknumber` / `optbool` / `checkstring` 等
4. **第 92 行**：如果验证失败 → `printinvalid` —— 记录"非法 RPC"并直接 return

**整条链路**：

```
[客户端] PlayerController 接到鼠标点击
    ↓
[客户端] SendRPCToServer(RPC.LeftClick, action, x, z, target, ...)
    ↓
[网络] 数据通过 UDP/TCP 发送
    ↓
[服务端] 接收数据 → 查 RPC_HANDLERS["LeftClick"]
    ↓
[服务端] 调用 fn(player, action, x, z, target, ...)
    ↓
[服务端] 参数校验
    ↓
[服务端] 执行 playercontroller:OnRemoteLeftClick(...)
```

> **核心结论**：**RPC = 客户端→服务端的远程函数调用**。客户端调"伪函数" → 网络层把参数序列化 → 服务端反序列化 → 调用 handler 执行。**和普通函数调用的差异**：参数会过网络、不可信——必须校验。

#### 第二步：mod 也能定义 RPC 吗？

**当然能**——这就是 9.2 节的核心内容。**Klei 给 mod 提供了 3 套 RPC 系统**：

| RPC 类型 | 函数 | 方向 |
| --- | --- | --- |
| 普通 ModRPC | `AddModRPCHandler` / `SendModRPCToServer` | **客户端 → 服务端** |
| Client ModRPC | `AddClientModRPCHandler` / `SendModRPCToClient` | **服务端 → 客户端** |
| Shard ModRPC | `AddShardModRPCHandler` / `SendModRPCToShard` | **shard ↔ shard**（地表/洞穴间） |

**99% 的 mod 用第一种**（普通 ModRPC）——本节先聚焦它，9.2.5 再讲后两种。

#### 第三步：用 mod RPC 做什么？

**典型场景**：

| 场景 | 用什么 |
| --- | --- |
| 玩家点击 mod UI 上的 "升级" 按钮 → 服务端扣除资源并改属性 | ModRPC |
| 客户端输入"/dance"聊天命令 → 服务端给玩家挂 buff | ModRPC |
| 客户端按某热键打开 mod 的菜单 → 服务端记录"该玩家打开了菜单" | ModRPC |
| 客户端 mod 完成本地计算 → 通知服务端持久化 | ModRPC |
| 服务端某事件 → 通知客户端弹一个**专属**通知 UI | Client ModRPC |
| 地表玩家完成任务 → 通知洞穴 shard 解锁某 boss | Shard ModRPC |

---

### 9.2.2 快速入门：mod 的 RPC 三件套

#### 第一步：API 速查

| API | 签名 | 用途 |
| --- | --- | --- |
| `AddModRPCHandler(namespace, name, fn)` | 注册 RPC handler | 服务端注册"如果客户端发 RPC，调这个 fn" |
| `GetModRPC(namespace, name)` | 获取 RPC id_table | 客户端用来调 RPC 的"句柄" |
| `SendModRPCToServer(id_table, ...)` | 发送 RPC | 客户端调用——把数据发给服务端 |

#### 第二步：3 个参数详解

**`AddModRPCHandler(namespace, name, fn)`**
- **namespace** —— 通常是 `modname`（**必须**，避免和其他 mod 冲突）
- **name** —— RPC 的名字（mod 内部唯一）
- **fn** —— handler 函数，**第一个参数是发送者 player**，后面是客户端发来的参数

**`SendModRPCToServer(id_table, ...)`**
- **id_table** —— `GetModRPC(namespace, name)` 返回的句柄（含 namespace 和 id）
- **`...`** —— 任意参数，**会被序列化**——只能传基础类型（number / string / bool / entity）

#### 第三步：5 行示范

**modmain.lua**（服务端 + 客户端**都跑**）：

```lua
-- 注册 RPC handler（仅服务端会执行 fn 体内逻辑）
AddModRPCHandler("mymod", "DoSomething", function(player, value)
    -- 第一件事：参数校验！
    if type(value) ~= "number" then
        return  -- 静默忽略非法请求
    end
    
    -- 业务
    player.components.health:DoDelta(value)
end)
```

**客户端代码**（任何客户端 UI 文件 / 按钮回调）：

```lua
-- 客户端按钮被按下时调用
SendModRPCToServer(GetModRPC("mymod", "DoSomething"), 10)
```

**就这么简单**——3 行注册、1 行调用。

#### 第四步：注册时机要在哪里？

**`AddModRPCHandler` 必须在 modmain.lua 的顶层调用**——**两端都要注册**——否则会报"Invalid RPC namespace"。

```lua
-- modmain.lua 顶层
AddModRPCHandler("mymod", "DoSomething", function(player, value) ... end)
AddModRPCHandler("mymod", "OtherRPC",    function(player, ...) ... end)
-- 不要包在 PostInit 里——modmain 直接注册
```

> **新手常见疑问**：客户端注册 fn 浪费？  
> **答**：`AddModRPCHandler` **本身只是把 fn 存到 MOD_RPC_HANDLERS 表**——**fn 体内逻辑只在服务端被调用**。所以注册没有副作用——但**注册必须两端都做**——因为客户端要靠这张表生成 RPC 的 id。

---

### 9.2.3 快速入门：完整的 RPC 注册和调用流程

#### 第一步：完整代码示范——"客户端点 mod UI，服务端给玩家加血"

**modmain.lua**：

```lua
-- 注册 RPC（两端）
AddModRPCHandler("myhealmod", "RequestHeal", function(player, amount)
    -- 校验：玩家有效、参数合法
    if not player or not player:IsValid() then return end
    if type(amount) ~= "number" or amount <= 0 or amount > 50 then
        return
    end
    
    -- 业务
    if player.components.health then
        player.components.health:DoDelta(amount)
        player.components.talker:Say(string.format("回血 %d", amount))
    end
end)

-- 给玩家挂上"按 H 触发回血"快捷键
GLOBAL.TheInput:AddKeyDownHandler(GLOBAL.KEY_H, function()
    if GLOBAL.ThePlayer then
        SendModRPCToServer(GetModRPC("myhealmod", "RequestHeal"), 10)
    end
end)
```

#### 第二步：跑一遍流程（控制台验证）

启动联机服务器，进入客户端，按 H ——

**主机端控制台**：
```
[Mod RPC] received RequestHeal from <player>
回血 10
```

**客户端**：
```
角色头顶冒：回血 10
血量从 90 → 100
```

#### 第三步：把它放在 mod UI 按钮里

```lua
-- 假设你写了一个 widget
local Widget = require "widgets/widget"
local Button = require "widgets/button"

local function OnHealClick(button)
    SendModRPCToServer(GetModRPC("myhealmod", "RequestHeal"), 10)
end

local btn = self:AddChild(Button())
btn:SetText("回血")
btn:SetOnClick(OnHealClick)
```

**用户点击 → 客户端调 SendModRPCToServer → 服务端执行业务**。

#### 第四步：用控制台直接 push RPC

```lua
-- 客户端控制台
SendModRPCToServer(GetModRPC("myhealmod", "RequestHeal"), 10)
-- 等价于按 H
```

> **新手 MVP 检查点**：能写出"AddModRPCHandler 注册 + GetModRPC 取句柄 + SendModRPCToServer 调用"3 行——掌握 70% RPC 用法。

---

### 9.2.4 进阶：RPC 的"信任边界"问题——为什么必须做参数校验

#### 第一步：客户端 = 不可信

**核心安全原则**：**客户端代码可被任意修改**——**任何来自客户端的数据都可能是恶意的**。

**典型攻击**：
- 修改 mod 客户端代码：`SendModRPCToServer(GetModRPC("myhealmod", "RequestHeal"), 9999)` —— 一次回 9999 滴血
- 修改参数类型：传 string / table / nil 而不是 number —— 服务端 fn 内部 `value + 10` 报错
- 高频发送：每秒 100 次 RPC —— 服务端 CPU 爆炸

**Klei 的 RPC handler 永远做 3 件事**：
1. **参数类型校验** —— `checknumber` / `checkstring` 等
2. **范围校验** —— 距离、位置、数值上限
3. **权限/状态校验** —— 玩家是否还活着、是否有权操作

#### 第二步：6 个标准校验函数

```4:13:scripts/networkclientrpc.lua
function checkbool(val) return val == nil or type(val) == "boolean" end
function checknumber(val) return type(val) == "number" end
function checkuint(val) return type(val) == "number" and tostring(val):find("%D") == nil end
function checkstring(val) return type(val) == "string" end
function checkentity(val) return type(val) == "table" end
optbool = checkbool
function optnumber(val) return val == nil or type(val) == "number" end
function optuint(val) return val == nil or (type(val) == "number" and tostring(val):find("%D") == nil) end
function optstring(val) return val == nil or type(val) == "string" end
function optentity(val) return val == nil or type(val) == "table" end
```

| 函数 | 含义 |
| --- | --- |
| `checkbool` | 必须是 bool |
| `checknumber` | 必须是 number |
| `checkuint` | 必须是无符号整数 |
| `checkstring` | 必须是 string |
| `checkentity` | 必须是 table（实体） |
| `optX` | 允许 nil 或 X 类型 |

**用法**：

```lua
AddModRPCHandler("mymod", "DoX", function(player, n, s, e)
    if not (checknumber(n) and checkstring(s) and optentity(e)) then
        printinvalid("DoX", player)
        return
    end
    -- 业务
end)
```

#### 第三步：范围校验

类型对了不等于值对——还要做**业务级范围校验**：

```lua
AddModRPCHandler("mymod", "RequestHeal", function(player, amount)
    if not checknumber(amount) then return end
    if amount <= 0 or amount > 50 then return end  -- 业务上 1-50
    
    -- 距离校验：客户端发的位置必须在合理范围内
    -- 看 networkclientrpc.lua:43-46 的 IsPointInRange 范例
end)
```

`networkclientrpc.lua:43-46`：

```43:46:scripts/networkclientrpc.lua
local function IsPointInRange(player, x, z)
    local px, py, pz = player.Transform:GetWorldPosition()
    return distsq(x, z, px, pz) <= 4096
end
```

**4096 = 64²** —— 玩家点击的位置不能距离自己超过 64 格——**防止远程作弊**。

#### 第四步：状态校验

```lua
AddModRPCHandler("mymod", "BuyItem", function(player, itemname, cost)
    if not (checkstring(itemname) and checknumber(cost)) then return end
    
    -- 状态校验
    if not player or not player:IsValid() then return end
    if player.components.health:IsDead() then return end           -- 死了不能买
    if player.components.inventory:GetGoldCount() < cost then return end  -- 钱不够
    if not GetValidItemList()[itemname] then return end             -- 物品名是否在白名单
    
    -- 业务
    player.components.inventory:Take(cost, "gold")
    player.components.inventory:GiveItem(SpawnPrefab(itemname))
end)
```

**注意 `GetValidItemList()`** —— 永远用**白名单**而不是**黑名单**——黑名单遗漏一个就让玩家无敌。

#### 第五步：`printinvalid` 的作用

```18:28:scripts/networkclientrpc.lua
local function printinvalid(rpcname, player)
    print(string.format("Invalid %s RPC from (%s) %s", rpcname, player.userid or "", player.name or ""))

    --This event is for MODs that want to handle players sending invalid rpcs
    TheWorld:PushEvent("invalidrpc", { player = player, rpcname = rpcname })

    if BRANCH == "dev" then
        --Internal testing
        assert(false, string.format("Invalid %s RPC from (%s) %s", rpcname, player.userid or "", player.name or ""))
    end
end
```

**3 件事**：
1. **打印日志** —— 方便服务器管理员追踪
2. **推 `invalidrpc` 世界事件** —— 让其他 mod 处理（比如自动 ban 玩家）
3. **dev 分支 assert** —— 给 Klei 内部测试用

**mod 自定义 handler 也建议遵循**：

```lua
local function PrintInvalidMod(rpcname, player)
    print(string.format("Invalid mymod %s RPC from %s", rpcname, player.userid or ""))
    TheWorld:PushEvent("invalidrpc", { player = player, rpcname = rpcname })
end

AddModRPCHandler("mymod", "Do", function(player, x)
    if not checknumber(x) then
        PrintInvalidMod("Do", player)
        return
    end
    -- ...
end)
```

---

### 9.2.5 进阶：`AddClientModRPCHandler` vs `AddShardModRPCHandler`

#### 第一步：3 类 RPC 的方向对照

| 类型 | 注册函数 | 发送函数 | 方向 | 典型用例 |
| --- | --- | --- | --- | --- |
| ModRPC | `AddModRPCHandler` | `SendModRPCToServer` | 客户端 → 服务端 | 玩家请求服务端做事 |
| Client ModRPC | `AddClientModRPCHandler` | `SendModRPCToClient` | 服务端 → 客户端 | 服务端给某玩家发专属通知 |
| Shard ModRPC | `AddShardModRPCHandler` | `SendModRPCToShard` | shard 之间 | 地表/洞穴跨世界通信 |

#### 第二步：Client ModRPC（服务端 → 客户端）

**和 NetVar 的区别**：
- NetVar **挂在某 entity 上**——entity 销毁就没了
- Client ModRPC **不挂在 entity 上**——是独立的"远程调用"
- NetVar 是**广播**（所有客户端收到）；Client ModRPC 可以**指定特定玩家**

**典型用例**：服务端某事件 → 给单个玩家弹个通知

```lua
-- modmain.lua
AddClientModRPCHandler("mymod", "ShowToast", function(text, duration)
    -- 这段代码在客户端执行
    if type(text) ~= "string" then return end
    if ThePlayer and ThePlayer.HUD then
        ThePlayer.HUD:ShowToast(text, duration or 5)
    end
end)

-- 服务端某处
SendModRPCToClient(GetClientModRPC("mymod", "ShowToast"), some_player.userid, "你被点名了！", 10)
```

**`SendModRPCToClient`** 的第一个**额外参数**是 `userid` 或 `nil`（nil = 广播）。

> **注意**：Client ModRPC 通常**不需要做严格校验**——服务端是可信的——但仍然要 nil 检查（client UI 还没准备好的边界情况）。

#### 第三步：Shard ModRPC（跨 shard）

**典型场景**：玩家在地表完成某事件——洞穴 shard 上的 boss 也要响应——但**两个 shard 在不同进程**——`TheWorld` 在两边互不可见。

```lua
-- modmain.lua
AddShardModRPCHandler("mymod", "BossUnlocked", function(sender, data)
    -- sender 是发送方 shard 的 ID
    if type(data) ~= "string" then return end
    
    -- 业务：在本 shard 触发对应事件
    TheWorld:PushEvent("mymod_bossunlocked", { type = data })
end)

-- 地表完成事件后
SendModRPCToShard(GetShardModRPC("mymod", "BossUnlocked"), nil, "deerclops_alt")
-- 第一个参数 nil = 广播给所有 shard；指定 shardid 就只发给那个 shard
```

**Shard ModRPC 的特点**：
- **不针对玩家**——shard 之间是同等关系，不是玩家发起
- **作用域是世界**——通常 handler 内部 push 个 TheWorld 事件让本 shard 的业务响应
- **延迟比同 shard 内更高**——shard 之间走的是 master 服务器中转

---

### 9.2.6 进阶：RPC 限流与 USERID_RPCS

#### 第一步：限流机制

回看 `networkclientrpc.lua:1908-1918`：

```1908:1918:scripts/networkclientrpc.lua
                local limit = RPC_Queue_Limiter[sender] or 0
                if limit < RPC_QUEUE_RATE_LIMIT then
                    RPC_Queue_Limiter[sender] = limit + 1
                    table.insert(RPC_Queue, { fn, sender, data, tick })
                else
                     -- This user is sending way too much for normal activity so take note of it.
                    if not RPC_Queue_Warned[sender] then
                        RPC_Queue_Warned[sender] = true
                        print("Rate limiting RPCs from [MOD]", sender, userid, "last one being ID", tostring(code), "of namespace", tostring(namespace))
                    end
                end
```

**关键设计**：
- 每个玩家有独立的 RPC 配额——超过 `RPC_QUEUE_RATE_LIMIT` 就丢弃
- **每注册一个 mod RPC handler，配额 +`RPC_QUEUE_RATE_LIMIT_PER_MOD`**（见 1848 行）
- 超额时打印警告但**不会断开玩家**——只是丢弃多余 RPC

**对开发者影响**：
- **不要为了"绕过限流"分多个 RPC**——配额是按玩家算的，分 RPC 没用
- **批量数据**：用一次 RPC 传 table、不要发 N 次

#### 第二步：USERID_RPCS

某些 RPC 应该**支持"玩家断线后还能发送"**——比如**离线挂机**。这就是 `USERID_RPCS` 的设计——用 userid 而非 player 实体作为 sender。

`networkclientrpc.lua:1900-1906`：

```1900:1906:scripts/networkclientrpc.lua
            local senderistable = type(sender) == "table"
            if USERID_RPCS[fn] or senderistable then
                local userid = senderistable and sender.userid or nil

                if USERID_RPCS[fn] then
                    sender = userid or sender
                end
```

mod 通常**不需要管这个**——95% RPC 在玩家在线时触发。

#### 第三步：RPC 队列

**重要细节**：RPC 不是**立刻执行**——而是**入队，等下一帧统一处理**：

```lua
table.insert(RPC_Queue, { fn, sender, data, tick })
```

**对开发者影响**：
- RPC handler 执行时——**已经过了至少 1 帧**
- 不要假定"客户端 send 完立刻服务端响应完"——有 ping + 1 帧的延迟
- handler 里访问 `player`——可能玩家**中间下线了**——要 `IsValid` 判

---

### 9.2.7 老手进阶：RPC vs NetVar vs net_event 的决策矩阵

#### 第一步：完整决策表

| 需求类型 | 推荐 |
| --- | --- |
| 服务端写状态、客户端持续读 | **NetVar**（持续广播） |
| 服务端某事件、所有客户端响应（不带 data 或简单 data）| **net_event**（如果绑实体） |
| 服务端某事件、所有客户端响应（带复杂 data）| **Client ModRPC**（广播，userid=nil） |
| 服务端某事件、单个客户端响应 | **Client ModRPC**（指定 userid） |
| 客户端某操作、服务端响应 | **ModRPC** |
| 跨 shard | **Shard ModRPC** |
| 客户端→服务端的"持续状态报告" | **混合**：客户端定期 RPC + 服务端 NetVar 广播 |

#### 第二步：性能对比

| 机制 | 网络字节数 | CPU 开销 | 并发能力 |
| --- | --- | --- | --- |
| NetVar `:set` | 1-32 bit + 协议头 | 极低 | 很高（每帧合并） |
| net_event `:push` | 0 + 协议头 | 极低 | 很高 |
| ModRPC | 参数总字节 + 协议头 + handler 内逻辑 | 中等（受限流） | 中等 |
| RPC 远比 NetVar 重 | | | |

**结论**：能用 NetVar / net_event 就**别用 RPC** —— RPC 走完整的协议栈、有限流、有反序列化开销。

#### 第三步：何时用 RPC vs net_event 的纠结点

**场景**：服务端某事件，需要**广播给所有客户端**，**带一些 data**。

**选项 A：net_event + NetVar 数据**（更省）

```lua
-- 用 NetVar 存 data
inst.event_data = net_string(inst.GUID, "ev.data", "evdirty")

-- 服务端
inst.event_data:set(json.encode(some_data))
inst.event_trigger:push()  -- 等价于 dirty 触发
```

**选项 B：Client ModRPC**（更通用）

```lua
SendModRPCToClient(GetClientModRPC("mymod", "ShowEvent"), nil, json.encode(some_data))
```

**判断**：
- 如果**事件高频** → A 更省
- 如果**data 复杂或可变长** → B 更灵活
- 如果**事件不绑实体** → B（Client ModRPC 不需要实体）

#### 第四步：RPC 的命名空间约定

- mod 只用**自己的 namespace**——通常是 `modname` 或 `modname.subsystem`
- **不要用 Klei 的内置 namespace**（"DST"、"Server" 等）
- **所有 mod 都注册 namespace = "mymod"** 时——`AddModRPCHandler` 内部自动按 mod 隔离 id —— 不会冲突

---

### 9.2.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：忘记参数校验，handler 报错

**症状**：恶意/老 mod 客户端发了非法参数 → 服务端 handler 报错 → 服务端崩溃 / 该玩家所有后续 RPC 失效。  
**修复**：handler 第一行**永远校验**——`checknumber` / `checkstring` / `optentity` / 范围 / 状态。

#### 陷阱 2：handler 内访问 `inst.components.xxx` 但没判 nil

**症状**：handler 收到 RPC 时玩家正好被强制 unequip / 断线 / 复活——`player.components.health` 可能 nil。  
**修复**：

```lua
AddModRPCHandler("mymod", "Do", function(player, ...)
    if not (player and player:IsValid()) then return end
    if not player.components.health then return end
    -- 业务
end)
```

#### 陷阱 3：在 modmain 之外注册 RPC

**症状**：报"Invalid RPC namespace"——客户端 send 失败。  
**原因**：RPC 必须**两端都注册**——只在某个 PostInit 里注册导致客户端那边没 namespace。  
**修复**：**永远在 modmain.lua 顶层注册**。

#### 陷阱 4：传 entity 参数实体已经销毁

**症状**：客户端 `SendModRPCToServer(..., target_pigman)` —— 网络传输期间 target_pigman 被销毁——服务端 fn 收到 `nil` 或 `invalid entity`。  
**修复**：handler 里 IsValid 判：

```lua
AddModRPCHandler("mymod", "AttackTarget", function(player, target)
    if not (target and target:IsValid()) then return end
    -- 业务
end)
```

#### 陷阱 5：客户端调 `SendModRPCToServer` 但忘了 ThePlayer 已是 nil

**症状**：客户端在主菜单或 loading 时调 RPC —— ThePlayer / TheNet 状态异常 —— 报错。  
**修复**：

```lua
if ThePlayer and ThePlayer:IsValid() and TheNet:GetServerIsClientHosted() ~= nil then
    SendModRPCToServer(GetModRPC("mymod", "Do"), 10)
end
```

更严格——如果 RPC 是必需的、且玩家**还在游戏中**——加上更多保护。

#### 陷阱 6：用 RPC 做"频繁心跳"

**症状**：每秒 30 次 RPC —— 限流触发 —— 大量 RPC 被丢弃 —— mod 行为异常。  
**修复**：**频繁状态用 NetVar 而非 RPC**——心跳类业务用 NetVar 持续广播。

#### 设计经验三条

**经验 ①：handler 第一行永远参数校验 + IsValid 判**

```lua
AddModRPCHandler("mymod", "Do", function(player, ...)
    if not (player and player:IsValid()) then return end
    -- 参数校验
    -- 状态校验
    -- 业务
end)
```

**经验 ②：批量胜过分散**

把"5 个 RPC 串"合并为"1 个 RPC 带 5 个字段"——**省限流配额、省协议头**。

**经验 ③：不在客户端做权限假设**

错误：客户端代码里 `if player.components.inventory:GetGoldCount() >= 100 then SendModRPCToServer(...) end` —— **客户端没有 components.inventory**——而且即使有也不可信。**所有权限/资源校验在服务端 handler 里做**。

---

### 9.2.9 小结

**RPC 一句话总结**：**客户端调"伪函数"→ 网络传到服务端 → 服务端 handler 校验 + 业务**。这是 mod 联机最重要的"反向通信"机制。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 注册客户端→服务端 RPC | `AddModRPCHandler("mymod", "Do", function(player, x) ... end)` |
| 客户端发送 | `SendModRPCToServer(GetModRPC("mymod", "Do"), 10)` |
| 注册服务端→客户端 RPC | `AddClientModRPCHandler("mymod", "Show", function(text) ... end)` |
| 服务端发送 | `SendModRPCToClient(GetClientModRPC("mymod", "Show"), userid, "msg")` |
| 注册跨 shard RPC | `AddShardModRPCHandler("mymod", "Sync", function(sender, data) ... end)` |
| 跨 shard 发送 | `SendModRPCToShard(GetShardModRPC("mymod", "Sync"), nil, data)` |
| 参数校验 | `checknumber / checkstring / checkbool / checkentity / optX` |

**6 个陷阱排雷顺序**

1. 忘参数校验 → 加 checkX
2. handler 访问 nil components → IsValid 判
3. 在 PostInit 注册 → modmain 顶层注册
4. entity 销毁 → 收到时 IsValid 判
5. 客户端 ThePlayer nil → 调用前判断
6. 高频 RPC 心跳 → 改用 NetVar

**3 条设计经验**

- ① **handler 第一行参数校验 + IsValid**
- ② **批量胜过分散**——合并 RPC 省配额
- ③ **不在客户端做权限假设**——服务端兜底

> **下一节预告**：9.3 节我们打开 **Classified 实体** —— 这是饥荒**最巧妙的网络设计**之一——专门为"每个玩家自己的私有数据"设计的"影子实体"——血量、饥饿、精神这些不能让其他玩家看到的状态全部走 classified。读完 9.3，你将理解为什么 `health_replica.lua` 里到处是 `self.classified.maxhealth:value()` 而不是 `self.maxhealth:value()` —— 这是 Klei 隐藏私有数据的精妙手段。

## 9.3 Classified 的工作原理

### 本节导读

9.1 节我们讲了 NetVar 系统——直接把 NetVar 挂在实体上，主机改值、所有客户端读值。但**联机版有一个关键问题**——

> **威尔逊的血条只有威尔逊自己能看，别人看不到这个数字**。**容器（箱子、背包）里有什么物品也只有打开的人能看到**。**怎么实现"主机端的状态值，只同步给特定的玩家、不同步给其他人"**？

**答案是 Classified 实体**——一种**特殊的"私有数据载体"**。它本身是一个独立 entity，但**只对特定客户端可见**——网络层面就直接屏蔽了不该看到的人。**饥荒里所有"私有信息"** ——玩家自己的健康/饥饿/精神/科技解锁/库存/地图探索状态——**全部走 classified**。

这一节我们打开 classified 的设计原理：**它怎么对玩家隐形、怎么与 replica 协作、mod 怎么自己写 classified 让自己的私有数据隐藏**。

> **新手**从 9.3.1-9.3.3 起步——理解 classified 的存在意义、`CLASSIFIED` 标签的作用、和直接挂 NetVar 的差异；**进阶读者**继续看 9.3.4-9.3.6，深入 `Hide()` 与 `CLASSIFIED` 标签的语义、`OnEntityReplicated` 与 `AttachClassified` 模式、classified 与 replica 的协作（含 `AttachClassifiedToReplicaComponent`）；**老手**跳到 9.3.7-9.3.8，看自定义 classified entity 实战、6 个最容易踩的坑。

---

### 9.3.1 快速入门：从一只"血条"看 classified 是什么

#### 第一步：在游戏里观察"我能看到自己的血但看不到别人的血数字"

打开联机服务器，**右上角你看到自己的血量是 95 / 150**。**走到队友身边**——你**只能看到他头顶的血条**（视觉条比例），**看不到具体数字**。

**为什么？** —— 队友的"具体血量数字"**根本没有同步给你**。客户端代码层面就**没有这个数据**。

#### 第二步：看 player_common.lua 的 classified 创建

`scripts/prefabs/player_common.lua:2633-2634`：

```2633:2634:scripts/prefabs/player_common.lua
        inst.player_classified = SpawnPrefab("player_classified")
        inst.player_classified.entity:SetParent(inst.entity)
```

**就两行**——主机端给每个玩家创建一个 `player_classified` entity，并 `SetParent` 到玩家自己身上——形成**父子关系**。

**`SetParent` 的网络层效果**：classified 跟着 player 实体走——player 离开视野、classified 也消失。**但仅"特定客户端"能看到**——下面解释。

#### 第三步：classified 的"隐身"机制

打开 `player_classified.lua:1330-1338`：

```1330:1338:scripts/prefabs/player_classified.lua
local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform() --So we can follow parent's sleep state
    inst.entity:AddMapExplorer()
    inst.entity:AddNetwork()
    inst.entity:Hide()
    inst:AddTag("CLASSIFIED")
```

**关键 4 行**：
- **`AddTransform`**：给 classified 一个位置——跟随父级
- **`AddNetwork`**：必须有，否则 NetVar 无效
- **`Hide()`**：**视觉层面隐身**——客户端不渲染
- **`AddTag("CLASSIFIED")`**：**网络层面"私有"**——引擎只把它同步给"对应的客户端"

**`CLASSIFIED` 标签**是核心 ——它告诉引擎：**"这个实体不要广播给所有人，只发给应该看到的人"**。**对玩家来说，应该看到的是自己**——所以 `player_classified` **只同步给玩家本人**。

> **核心结论**：**classified entity 是"网络层的私有载体"** —— 通过 `CLASSIFIED` 标签实现"按目标隐形"。挂在 classified 上的 NetVar **天然只对应该看到它的客户端可见**。

#### 第四步：完整的"主机端写血量 → 客户端读血量"链路

```
[主机端] 玩家健康改变 → health.lua:DoDelta()
    ↓
[主机端] 调 health_replica:SetCurrent(newval)
    ↓
[主机端] health_replica → self.classified:SetValue("currenthealth", newval)
    ↓
[主机端] inst.currenthealth:set(newval)（NetVar 写值）
    ↓
[网络] 引擎检测到 classified entity 属于这个玩家
    ↓ 仅同步给本玩家的客户端，其他客户端收不到
[本玩家客户端] inst.currenthealth:value() 拿到新值
    ↓
[本玩家客户端] healthdirty 事件触发 → HUD 重绘血条
```

**其他玩家**：他们的客户端**根本没有这个 classified entity**——`inst.currenthealth` 是 nil——**根本没法读到这个数据**。

---

### 9.3.2 快速入门：classified 实体的标准创建模板

#### 第一步：最小骨架

```lua
-- scripts/prefabs/myownership_classified.lua
local function fn()
    local inst = CreateEntity()

    if TheWorld.ismastersim then
        inst.entity:AddTransform()  -- 跟随 parent
    end
    inst.entity:AddNetwork()
    inst.entity:Hide()              -- 客户端不渲染
    inst:AddTag("CLASSIFIED")       -- 网络层私有

    -- ★ 创建 NetVar（两端都跑）
    inst.someval = net_byte(inst.GUID, "myown.someval", "myowndirty")

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst                  -- 客户端到此返回
    end

    inst.persists = false             -- 不存档
    
    return inst
end

return Prefab("myownership_classified", fn)
```

**对比普通 prefab 的差异**：
- **`AddTransform` 包在 `if TheWorld.ismastersim`** 里——客户端不需要
- **`Hide()`** —— classified 不可视
- **`AddTag("CLASSIFIED")`** —— 网络层标记
- **`inst.persists = false`** —— classified 不写入存档（数据由父级保存）

#### 第二步：创建并挂在父级上

```lua
-- 在父级 prefab 的 master_postinit 里
inst.myownership_classified = SpawnPrefab("myownership_classified")
inst.myownership_classified.entity:SetParent(inst.entity)
```

**`SetParent`** 让 classified 跟着父级移动、销毁。

#### 第三步：服务端写值

```lua
-- 在父级的某 component 里
inst.myownership_classified.someval:set(42)
```

#### 第四步：客户端读值

```lua
-- 客户端
local val = inst.myownership_classified.someval:value()

-- 监听变化
inst:ListenForEvent("myowndirty", function(parent)
    -- 注意 listener 第一参数是 parent，不是 classified
end, inst.myownership_classified)
```

> **关键**：**listener 注册在 classified 上**——但回调签名仍是 `(source_entity, data)` —— 这里 source 就是 classified 本身。

---

### 9.3.3 快速入门：classified vs 直接挂 NetVar 的差异

#### 第一步：对比表

| 维度 | 直接挂 NetVar | 用 classified |
| --- | --- | --- |
| **网络可见性** | 所有客户端都看到 | 仅"目标客户端"看到 |
| **是否跨实体** | 必须挂在普通实体上 | 独立 entity，可挂任意父 |
| **生命周期** | 跟随宿主实体 | 跟随父级（用 SetParent） |
| **存档** | 跟随宿主写存档 | 默认不存档 |
| **NetVar 数量上限** | 取决于宿主类型 | 一个 classified 可有几十/上百 NetVar |
| **客户端引用方式** | `inst.var:value()` | `inst.classified.var:value()` |
| **典型场景** | 共享状态（boss 阶段、世界 buff） | 私有数据（玩家血量、库存内容） |

#### 第二步：什么时候用 classified？

**判断标准**：

> **私有信息 → classified**；**公共信息 → 直接 NetVar**。

**例子**：
- ✅ classified：玩家血量数字、玩家库存物品、玩家科技树解锁、玩家成就进度
- ✅ 直接 NetVar：boss 当前阶段、克劳斯怪眼睛是否发光、天气状态、共享 buff
- ✅ 直接 NetVar（其实可以 classified）：玩家昵称（公开）、玩家位置（公开）

**为什么不能"全部用 classified"** —— classified 的**网络开销稍微更高**（要管理私有目标列表）；且 classified 不**适合**"全员都关心"的状态。

#### 第三步：为什么挂在 player_classified 而不是直接挂在 player 上？

**权宜的方案**：直接给 player 加一堆 NetVar——`player.currenthealth = net_ushortint(...)` —— **能跑**——但**所有人能看到所有玩家的具体数字**。

**Klei 的方案**：挂到 player_classified 上 —— **数字只同步给本人** —— 实现了"自己看得到、别人看不到"。

> **设计哲学**：**信息"知情边界"是网络协议层的事，不是业务层**。在协议层面就过滤掉不该看到的数据——既省带宽又保护隐私。

---

### 9.3.4 进阶：`CLASSIFIED` 标签 + `Hide()` 的语义

#### 第一步：`CLASSIFIED` 标签的引擎处理

`CLASSIFIED` 是**引擎识别的特殊标签**——网络层做以下事：
1. **不广播给所有客户端** —— 引擎仅同步给"知情者"
2. **知情者由父子关系决定** —— 通过 `SetParent` 把 classified 挂在 player 下，引擎识别"这个 classified 属于这个 player"
3. **同步触发**：classified 上的 NetVar set 时，**只发给该 player 的客户端**

**注意**：classified 不仅适用于"挂在 player 上"——也可以挂在**普通实体**上做"管理者私有"——比如 `container_classified` 挂在容器上 —— **只同步给打开容器的玩家**。

#### 第二步：`Hide()` 的视觉效果

`inst.entity:Hide()` 让 classified entity 在客户端**不参与渲染**——它没有可视贴图，但仍然**有 transform** 跟随父级。**这避免了"莫名其妙的发光物"** 出现在玩家屏幕上。

#### 第三步：`SetParent` 的网络层意义

`inst.player_classified.entity:SetParent(player.entity)` 不仅是位置同步——它**告诉引擎**：

- 这个 classified **属于** 这个 player
- 这个 player 的客户端**需要**看到这个 classified
- 其他玩家**不需要**

**没有 SetParent 的 classified 是"无主的"** —— 引擎不知道该同步给谁——可能直接被忽略。

#### 第四步：用控制台验证

主机控制台：

```lua
print(ThePlayer.player_classified)
-- => player_classified[xxxxx]

print(ThePlayer.player_classified:HasTag("CLASSIFIED"))
-- => true

print(ThePlayer.player_classified.entity:GetParent())
-- => 你自己 player[xxxxx]
```

客户端 A 控制台（你自己）：
```lua
print(ThePlayer.player_classified.currenthealth:value())
-- => 100
```

客户端 B 控制台（队友）：
```lua
print(ThePlayer.player_classified)
-- => player_classified[xxxxx] (这是 B 自己的)
print(队友.player_classified)
-- => nil  (B 客户端没有 A 的 classified)
```

---

### 9.3.5 进阶：`OnEntityReplicated` 与 `AttachClassified` 模式

#### 第一步：客户端是怎么"拿到" classified 的？

主机端创建 classified、`SetParent` 到玩家上。**客户端这边**——引擎在网络同步过来后**触发** `OnEntityReplicated` 回调——让客户端 lua 层面知道"这个 classified 来了，可以挂上业务"。

回看 `writeable_classified.lua:11-19`：

```11:19:scripts/prefabs/writeable_classified.lua
local function OnEntityReplicated(inst)
    inst._parent = inst.entity:GetParent()
    if inst._parent == nil then
        print("Unable to initialize classified data for writeable")
	elseif not inst._parent:TryAttachClassifiedToReplicaComponent(inst, "writeable") then
        inst._parent.writeable_classified = inst
        inst.OnRemoveEntity = OnRemoveEntity
    end
end
```

**3 步**：
1. **拿到父级 entity** —— `inst.entity:GetParent()`
2. **尝试 attach 到 replica** —— `TryAttachClassifiedToReplicaComponent` ——9.3.6 节专讲
3. **回退到挂在父级字段上** —— `inst._parent.writeable_classified = inst`

#### 第二步：完整模板——客户端只看 OnEntityReplicated

```lua
-- writeable_classified.lua（最小版）
local function fn()
    local inst = CreateEntity()
    if TheWorld.ismastersim then
        inst.entity:AddTransform()
    end
    inst.entity:AddNetwork()
    inst.entity:Hide()
    inst:AddTag("CLASSIFIED")
    inst.entity:SetPristine()
    
    if not TheWorld.ismastersim then
        inst.OnEntityReplicated = OnEntityReplicated
        return inst
    end
    
    inst.persists = false
    return inst
end
```

**关键**：**`OnEntityReplicated` 只在客户端有用**——主机端 classified 通过 master_postinit 直接挂上业务；客户端则等 OnEntityReplicated 回调来挂。

#### 第三步：为什么不直接在客户端 `_postinit` 里挂？

**两个原因**：
1. **客户端 prefab fn 跑的时候 parent 还不存在** —— prefab fn 里 `inst.entity:GetParent()` 返回 nil
2. **网络同步是异步的** —— classified 实体先 spawn、parent 关系后建立——必须等 `OnEntityReplicated` 时才知道父级

#### 第四步：OnEntityReplicated vs OnNetworkSetParent

某些 mod 还会用 `OnNetworkSetParent` —— **当父子关系发生变化**时触发——比 `OnEntityReplicated` 频率更高。**95% 用 OnEntityReplicated 即可**。

---

### 9.3.6 进阶：classified 与 replica 的协作

#### 第一步：replica 是什么？

8.x 节讲过——**replica 是组件的客户端代理**——客户端没有 `inst.components.health`，只有 `inst.replica.health`。**replica 内部访问 NetVar 拿数据**。

#### 第二步：`AttachClassifiedToReplicaComponent` 的设计

回看 `writeable_classified.lua` 第 15 行：

```15:17:scripts/prefabs/writeable_classified.lua
	elseif not inst._parent:TryAttachClassifiedToReplicaComponent(inst, "writeable") then
        inst._parent.writeable_classified = inst
        inst.OnRemoveEntity = OnRemoveEntity
```

**意图**：**让 replica 持有 classified 引用**——这样 replica 的方法可以通过 `self.classified.var:value()` 直接读 NetVar。

回到 `health_replica.lua:1-9`：

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

**两端逻辑**：
- 主机端：`self.classified = inst.player_classified` —— 直接拿
- 客户端：`self:AttachClassified(...)` —— 在 OnEntityReplicated 之后才能调

#### 第三步：完整的"组件-replica-classified-NetVar"四层结构

```
[主机端]
  inst.components.health.currenthealth = 100   ← 真实数据
  ↓
  health.lua DoDelta 改值
  ↓
  health_replica:SetCurrent(newval)
  ↓
  self.classified:SetValue("currenthealth", newval)
  ↓
  player_classified.currenthealth:set(newval)  ← NetVar 同步

  ─── 网络 ───

[客户端]
  player_classified.currenthealth:value()       ← NetVar 读
  ↑
  通过 healthdirty 事件触发
  ↑
  HUD 监听并重绘血条
```

**对开发者影响**：
- **客户端 mod 永远走 replica.xxx** —— `inst.replica.health:Max()` 这种
- **不要 mod 直接访问 classified** —— `inst.player_classified.currenthealth:value()` 能跑但**绕过了 replica 抽象**——下次 Klei 改 classified 结构 mod 就崩了

---

### 9.3.7 老手进阶：自定义 classified entity 实战

#### 第一步：场景——做一个"私人技能冷却"系统

需求：mod 给每个玩家加 5 个技能，每个技能有冷却。**只有玩家自己能看到自己的冷却时间**——不能让其他玩家通过血条看到对方技能 ready 与否。

#### 第二步：写 myskill_classified.lua

```lua
-- scripts/prefabs/myskill_classified.lua

local SKILL_COUNT = 5

local function fn()
    local inst = CreateEntity()

    if TheWorld.ismastersim then
        inst.entity:AddTransform()
    end
    inst.entity:AddNetwork()
    inst.entity:Hide()
    inst:AddTag("CLASSIFIED")

    -- 5 个技能的冷却剩余时间
    inst.skill_cooldowns = {}
    for i = 1, SKILL_COUNT do
        inst.skill_cooldowns[i] = net_smallfloat(inst.GUID, "skill.cooldown[" .. i .. "]", "skillcooldowndirty")
    end

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        inst.OnEntityReplicated = function(inst)
            local parent = inst.entity:GetParent()
            if parent then
                parent.myskill_classified = inst
            end
        end
        return inst
    end

    inst.persists = false
    return inst
end

return Prefab("myskill_classified", fn)
```

#### 第三步：挂到玩家身上

```lua
-- modmain.lua
AddPlayerPostInit(function(inst)
    if TheWorld.ismastersim then
        inst.myskill_classified = SpawnPrefab("myskill_classified")
        inst.myskill_classified.entity:SetParent(inst.entity)
    end
end)
```

**注意**：`AddPlayerPostInit` 在玩家 prefab 实例化后调用——`master_postinit` 阶段——主机端能直接 spawn classified。

#### 第四步：服务端业务

```lua
-- 服务端：触发某技能
function StartSkillCooldown(player, skill_id, duration)
    if player.myskill_classified then
        player.myskill_classified.skill_cooldowns[skill_id]:set(duration)
        player.myskill_classified:DoTaskInTime(0.5, UpdateCooldownTick, skill_id)
    end
end
```

#### 第五步：客户端 UI

```lua
-- 在 HUD widget 里
function MySkillBar:OnAttach(player)
    self._player = player
    
    -- 监听 dirty 事件
    if player.myskill_classified then
        self.inst:ListenForEvent("skillcooldowndirty", function()
            self:RefreshCooldownDisplay()
        end, player.myskill_classified)
    else
        -- classified 还没 attach，等一下
        player:DoTaskInTime(0.1, function() self:OnAttach(player) end)
    end
end

function MySkillBar:RefreshCooldownDisplay()
    for i = 1, 5 do
        local cd = self._player.myskill_classified.skill_cooldowns[i]:value()
        self.skill_icons[i]:SetCooldown(cd)
    end
end
```

---

### 9.3.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：忘记 `AddTag("CLASSIFIED")`

**症状**：mod 给 entity 挂上一堆 NetVar、SetParent 到玩家——但**所有客户端都能读到** —— 私有变成了广播。  
**原因**：没加 `CLASSIFIED` 标签——引擎不知道这是私有的。  
**修复**：永远加：

```lua
inst:AddTag("CLASSIFIED")
```

#### 陷阱 2：`SetParent` 漏写

**症状**：classified 创建后**任何客户端都看不到**——主机改了客户端读 nil。  
**原因**：没 `SetParent`——引擎不知道这个 classified 该同步给谁。  
**修复**：

```lua
inst.myclassified = SpawnPrefab("myclassified")
inst.myclassified.entity:SetParent(inst.entity)  -- ★
```

#### 陷阱 3：客户端在 prefab fn 里直接访问 classified

**症状**：客户端 prefab fn 里 `inst.player_classified.currenthealth:value()` —— nil 错。  
**原因**：prefab fn 跑的时候 classified 还没 replicate 过来——必须等 OnEntityReplicated。  
**修复**：用 OnEntityReplicated 回调；或者先判 nil。

#### 陷阱 4：客户端 mod 直接访问 classified 而不通过 replica

**症状**：mod 能跑，但兼容性差——Klei 一改 classified 结构就崩。  
**原因**：classified 是**实现细节**——mod 应该通过 replica 抽象。  
**修复**：

```lua
-- ❌ 不推荐
local hp = inst.player_classified.currenthealth:value()

-- ✅ 推荐
local hp = inst.replica.health:Current()
```

#### 陷阱 5：自定义 classified `inst.persists = true`

**症状**：classified 被存档——玩家断线重连时**两份 classified 同时存在**。  
**原因**：classified 应该跟随 player 重新创建——不应自己存档。  
**修复**：永远 `inst.persists = false`。

#### 陷阱 6：classified 父级被销毁但 classified 没被自动清理

**症状**：父级 player 重连时旧 classified 还残留。  
**原因**：通常 `SetParent` 会让 classified 跟随销毁——但**某些边界情况**（migrate / despawn 异常）下可能漏。  
**修复**：父级监听 `onremove` 事件，主动 `Remove` classified。

#### 设计经验三条

**经验 ①：1 个 player + N 个 classified 是常见模式**

不一定要把所有 NetVar 塞到 `player_classified` —— mod 可以**给 player 挂自己的私有 classified**：

```
player
├── player_classified（Klei 内置：health/hunger/sanity/...）
├── myskill_classified（mod 自定义：技能冷却）
├── myquest_classified（mod 自定义：任务进度）
```

**好处**：mod 数据**不污染** Klei 内部 classified；mod 卸载时清理简单。

**经验 ②：classified 上的 NetVar 命名要清晰**

```lua
-- ❌
inst.x1 = net_byte(inst.GUID, "x1")
inst.x2 = net_byte(inst.GUID, "x2")

-- ✅
inst.skill_fire_cooldown = net_smallfloat(inst.GUID, "myskill.fire.cooldown", "myskilldirty")
```

调试时一眼能看出"这是哪个系统的什么数据"。

**经验 ③：客户端 mod 业务永远走 replica，不直接碰 classified**

mod 想给 health 加个新字段？**写 replica 扩展**——把 classified 当成 replica 的"私有存储"，而不是直接的业务接口。

---

### 9.3.9 小结

**Classified 一句话总结**：**带 `CLASSIFIED` 标签的私有实体——通过 SetParent 挂到目标——网络层只同步给"应该看到"的客户端**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 创建 classified 模板 | `inst.entity:AddNetwork() + inst.entity:Hide() + inst:AddTag("CLASSIFIED")` |
| 挂到父级 | `inst.myclassified.entity:SetParent(inst.entity)` |
| 不存档 | `inst.persists = false` |
| 客户端 attach | 在 `OnEntityReplicated` 里设置 `parent.xxx_classified = inst` |
| 服务端写 | `inst.myclassified.var:set(val)` |
| 客户端读 | `inst.myclassified.var:value()` |
| 客户端监听 | `inst:ListenForEvent("vardirty", fn, inst.myclassified)` |

**6 个陷阱排雷顺序**

1. 忘 CLASSIFIED 标签 → 永远加
2. 忘 SetParent → 永远挂
3. prefab fn 里访问 classified → 用 OnEntityReplicated
4. 直接访问 classified（不走 replica）→ 用 replica
5. classified 写存档 → persists=false
6. classified 残留 → 父级 onremove 时清理

**3 条设计经验**

- ① **1 player + N classified**：mod 自己开一份 classified，不污染 Klei 内置
- ② **NetVar 命名清晰**：含 mod 前缀 + 业务系统名
- ③ **客户端走 replica**：classified 是实现细节，replica 才是接口

> **下一节预告**：9.4 节我们把 **`ismastersim` / `TheNet:GetIsServer()` / `TheNet:GetIsClient()`** 这 4 个判断的边界讲清楚。读完 9.4，你将能在 mod 任何位置准确说出"这段代码是在主机/客户端/dedicated/单机的哪一种部署上跑"——这是 mod 联机调试的根本能力。

## 9.4 ismastersim / TheNet:GetIsServer() / TheNet:GetIsClient() 的区别

### 本节导读

9.1 ~ 9.3 我们把网络通信的"管道"讲透了——NetVar、RPC、Classified——但每条管道两端的身份**怎么判断**？

新手翻 Klei 源码，会立刻被一堆"看似一样"的 API 搞晕：

```lua
TheWorld.ismastersim
TheNet:GetIsServer()
TheNet:GetIsClient()
TheNet:IsDedicated()
TheNet:GetIsMasterSimulation()
TheNet:GetIsHosting()
TheNet:GetServerIsClientHosted()
```

**它们一会儿等价、一会儿不等价**——典型案例：

- 单机版 → `ismastersim = true`、`GetIsServer = true`、`GetIsClient = false`
- 主机端联机 → `ismastersim = true`、`GetIsServer = true`、`GetIsClient = true`（**主机自己也是个客户端！**）
- 客户端联机 → `ismastersim = false`、`GetIsServer = false`、`GetIsClient = true`
- Dedicated 服务器 → `ismastersim = true`、`GetIsServer = true`、`GetIsClient = false`、`IsDedicated = true`

**这一节我们把 4 种部署模式 × 7 个判断函数的完整真值表打透**——确保你**任何时候都能准确说出"这段代码在什么部署上跑"**——这是 mod 联机调试的根本能力。

> **新手**从 9.4.1-9.4.3 起步——理解 4 个核心判断函数、4 种部署模式、完整真值表；**进阶读者**继续看 9.4.4-9.4.6，深入 `ismastersim` 在 world.lua 的赋值时机、`IsDedicated` 与"主机端联机"的区别、shard 视角和"主机玩家既是 server 又是 client"的双重身份；**老手**跳到 9.4.7-9.4.8，看常用判断模式实战、6 个最容易搞错的边界场景。

---

### 9.4.1 快速入门：从一只 prefab 看 4 种"我在哪里跑"判断

#### 第一步：典型 prefab 的"分支结构"

打开 `scripts/prefabs/warg.lua`，看 `prefabs/` 下任何稍微复杂的实体——你会反复见到这种结构：

```lua
local function fn()
    local inst = CreateEntity()
    
    -- ★ 段 1：两端都跑（共享）
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    
    inst:AddTag("xxx")
    inst._eyeflames = net_bool(inst.GUID, "...", "eyeflamesdirty")
    
    if not TheNet:IsDedicated() then
        -- ★ 段 2：非 dedicated 才跑（含主机端联机的客户端 + 客户端联机 + 单机）
        --   通常是音乐、特效、UI、客户端动画
        inst:DoPeriodicTask(1, Mutated_PushMusic, 0)
    end
    
    inst.entity:SetPristine()
    
    if not TheWorld.ismastersim then
        -- ★ 段 3：客户端 prefab fn 到此返回
        return inst
    end
    
    -- ★ 段 4：仅服务端跑（含主机 + dedicated）
    --   组件、SG、brain、所有业务逻辑
    inst:AddComponent("health")
    inst:SetStateGraph("SGwarg")
    
    return inst
end
```

**4 段对应 4 类代码**：

| 段 | 跑在哪里 | 内容 |
| --- | --- | --- |
| ① 共享段 | 任何部署，任何端 | NetVar 创建、tag、AddNetwork |
| ② 非 dedicated 段 | 单机 / 主机端 / 客户端联机；**dedicated 不跑** | 客户端音效/特效/UI |
| ③ SetPristine 后的 `if not ismastersim then return` | **客户端联机**到此返回 | （没了） |
| ④ master_postinit | 主机端 / dedicated；**客户端联机不跑** | 业务逻辑、组件、SG、brain |

#### 第二步：4 个核心判断函数

| 函数 | 含义 | 反义 |
| --- | --- | --- |
| `TheWorld.ismastersim` | 当前进程是不是"主机模拟"——能写权威状态 | 客户端联机 |
| `TheNet:GetIsServer()` | 当前进程是不是服务端——含主机和 dedicated | 客户端 |
| `TheNet:GetIsClient()` | 当前进程是不是客户端——含主机自己（主机也是客户端） | dedicated |
| `TheNet:IsDedicated()` | 当前进程是不是独立服务器（无 GUI） | 有 UI 的端（含主机和 client） |

**核心区别**：

- `ismastersim` 和 `GetIsServer` 在 99% 场景**等价** —— 都表示"我能写权威状态"
- `GetIsClient` **不等于** `not GetIsServer` —— **主机端两者都是 true**
- `IsDedicated` **不等于** `not GetIsClient` —— 仅判断是否是无 GUI 的独立服务器

#### 第三步：用控制台快速测试

启动联机服务器（你做主机），输入：

```lua
print("ismastersim:", TheWorld.ismastersim)
print("GetIsServer:", TheNet:GetIsServer())
print("GetIsClient:", TheNet:GetIsClient())
print("IsDedicated:", TheNet:IsDedicated())
print("GetIsHosting:", TheNet:GetIsHosting())
print("GetServerIsClientHosted:", TheNet:GetServerIsClientHosted())
```

**主机端结果**：

```
ismastersim: true
GetIsServer: true
GetIsClient: true              ← 主机自己也是个 client！
IsDedicated: false
GetIsHosting: true
GetServerIsClientHosted: true  ← 服务器是 client-hosted 的
```

**客户端联机结果**：

```
ismastersim: false
GetIsServer: false
GetIsClient: true
IsDedicated: false
GetIsHosting: false
GetServerIsClientHosted: true
```

**Dedicated 服务器结果**：

```
ismastersim: true
GetIsServer: true
GetIsClient: false             ← dedicated 不是 client
IsDedicated: true
GetIsHosting: true             ← dedicated 也算"hosting"
GetServerIsClientHosted: false ← dedicated 不是 client-hosted
```

---

### 9.4.2 快速入门：4 个核心判断函数速查

#### 第一步：完整 API 表

| API | 类型 | 用途 | 典型场景 |
| --- | --- | --- | --- |
| `TheWorld.ismastersim` | EntityScript field（bool） | "我能写权威状态吗" | prefab fn 服务端独有段判断；**最常用** |
| `TheNet:GetIsServer()` | bool 方法 | "我是服务端吗" | 等价于 `ismastersim`，但 `TheWorld` 还没初始化时只能用这个 |
| `TheNet:GetIsClient()` | bool 方法 | "我有客户端 GUI 吗" | 想跳过 dedicated 上的 UI/特效 |
| `TheNet:IsDedicated()` | bool 方法 | "我是 dedicated 服务器吗" | 想跳过 dedicated 上的视觉效果（音乐/光效） |

#### 第二步：辅助 API

| API | 用途 |
| --- | --- |
| `TheNet:GetIsMasterSimulation()` | 等价于 `TheWorld.ismastersim`——但**TheWorld 创建之前就可调** |
| `TheNet:GetIsHosting()` | "我是 hosting 服务器吗"——区分主机/dedicated 都返回 true，仅 client join 时 false |
| `TheNet:GetServerIsClientHosted()` | "服务器是被 client hosted 吗"——主机端 + 客户端联机两者都 true；dedicated 才是 false |

`world.lua:425`：

```425:425:scripts/prefabs/world.lua
        inst.ismastersim = TheNet:GetIsMasterSimulation()
```

**`TheWorld.ismastersim`** 就是从 `TheNet:GetIsMasterSimulation()` 拷贝过来的——**值永远一致**。

#### 第三步：什么时候用哪个？

| 想做的事 | 用什么 |
| --- | --- |
| 在 prefab fn / component 里判断"我能写状态吗" | `TheWorld.ismastersim` |
| 在 modmain / 早期初始化（TheWorld 可能还 nil）里判断 | `TheNet:GetIsServer()` 或 `TheNet:GetIsMasterSimulation()` |
| 跳过 dedicated 上的 UI / 音效 / 视觉 | `not TheNet:IsDedicated()` |
| 仅在客户端跑某 UI 逻辑 | `TheNet:GetIsClient()`（主机端自己也跑——通常是想要的） |
| 区分"我是主机还是 dedicated" | `TheNet:GetIsClient()`（主机端 true、dedicated false） |
| 区分"我是单机还是联机" | 看 `TheNet:GetServerGameMode()` 或专门的单机标识 |

---

### 9.4.3 快速入门：4 种部署模式 × 7 个判断 完整真值表

#### 第一步：4 种部署模式

| 模式 | 部署 |
| --- | --- |
| **单机** | 玩家自己启动游戏，单人 |
| **主机端联机（host）** | 玩家自己当主机，开放给其他玩家加入 |
| **客户端联机（join）** | 玩家加入别人的服务器 |
| **Dedicated 服务器** | 独立进程的专门服务器（无 GUI、命令行） |

#### 第二步：完整真值表

| 判断 | 单机 | 主机端联机 | 客户端联机 | Dedicated |
| --- | --- | --- | --- | --- |
| `TheWorld.ismastersim` | **true** | **true** | false | **true** |
| `TheNet:GetIsServer()` | **true** | **true** | false | **true** |
| `TheNet:GetIsClient()` | **true** | **true** | **true** | false |
| `TheNet:IsDedicated()` | false | false | false | **true** |
| `TheNet:GetIsHosting()` | **true** | **true** | false | **true** |
| `TheNet:GetServerIsClientHosted()` | **true** | **true** | **true** | false |
| `TheNet:GetIsMasterSimulation()` | **true** | **true** | false | **true** |

**核心观察**：

1. **`ismastersim == GetIsServer == GetIsHosting == GetIsMasterSimulation`** —— 4 个**等价**——单机/主机/dedicated 都 true
2. **`GetIsClient`** = 是否有 GUI 客户端进程 —— 4 种模式中只有 dedicated 是 false
3. **`IsDedicated`** = 是否独立服务器 —— 4 种模式中只有 dedicated 是 true
4. **`GetServerIsClientHosted`** = 服务器有 GUI 玩家 —— 单机/主机端/客户端联机都 true，dedicated false

#### 第三步：3 个典型分支模式

**模式 A：仅服务端业务**

```lua
if TheWorld.ismastersim then
    -- 业务、组件、SG、brain
end
```

**覆盖**：单机 + 主机端 + dedicated。**不覆盖**：客户端联机。

**模式 B：跳过 dedicated 的视觉**

```lua
if not TheNet:IsDedicated() then
    -- 音效、特效、UI、动画
end
```

**覆盖**：单机 + 主机端 + 客户端联机。**不覆盖**：dedicated。

**模式 C：服务端业务但**仅在主机或单机**（不要 dedicated）**

```lua
if TheWorld.ismastersim and not TheNet:IsDedicated() then
    -- 罕见——通常是 mod 想用主机的"GUI 信息"做服务端业务
end
```

或者等价：

```lua
if TheNet:GetServerIsClientHosted() and not TheNet:GetIsClient() then
    -- 不可能——上面两个条件不能同时为真
end
```

**覆盖**：单机 + 主机端联机。**不覆盖**：客户端联机 + dedicated。

---

### 9.4.4 进阶：`TheWorld.ismastersim` 的来源

#### 第一步：world.lua 的初始化

`scripts/prefabs/world.lua:425`：

```425:425:scripts/prefabs/world.lua
        inst.ismastersim = TheNet:GetIsMasterSimulation()
```

**这一行在 world prefab 创建时执行**——值由引擎层决定（C++ 端）。

#### 第二步：`TheWorld == nil` 的窗口

游戏启动到 world 创建之间——**TheWorld 是 nil**——访问 `TheWorld.ismastersim` 会**报错**。

**典型出错场景**：modmain.lua 顶层访问：

```lua
-- modmain.lua
if TheWorld.ismastersim then  -- ❌ 启动时 TheWorld 还是 nil
    -- ...
end
```

**修复**：用 `TheNet:GetIsServer()` 或者**等到 PostInit**：

```lua
AddSimPostInit(function()
    if TheWorld.ismastersim then
        -- ✓
    end
end)
```

#### 第三步：什么时候 ismastersim 变化？

**永远不变**——一旦设置就固定。**整个游戏会话期间**：
- 主机的 `ismastersim` 永远 true
- 客户端的永远 false
- shard 切换（地表 ↔ 洞穴）也不变——每个 shard 有自己的 TheWorld，ismastersim 由本 shard 的 master 决定

---

### 9.4.5 进阶：`TheNet:IsDedicated` 与"为什么 dedicated 不需要播音效"

#### 第一步：dedicated 的特殊性

**dedicated 服务器**没有 GUI——没有声卡、没有显示器、没有玩家在屏幕前。**所有"视觉/听觉"代码都是浪费**：

- 播音效 → 没有声卡输出 → 浪费 CPU
- 播粒子 → 没有渲染上下文 → 报错
- spawn 客户端专属特效 → 客户端自己会 spawn，dedicated 这边浪费内存

#### 第二步：典型 dedicated 跳过模式

来自 `scripts/prefabs/warg.lua:421`：

```lua
if TheNet:IsDedicated() then
    -- 跳过逻辑（罕见）
end
```

更常见的是反向：

```614:626:scripts/prefabs/warg.lua
        if not TheNet:IsDedicated() then
            -- 设置音乐
        end
```

**模式**：`if not TheNet:IsDedicated() then ... end` —— **覆盖单机 + 主机 + 客户端联机；跳过 dedicated**。

#### 第三步：完整逻辑分支决策

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddNetwork()
    
    -- 共享部分（所有进程）
    inst._statevar = net_byte(inst.GUID, "warg.state", "statedirty")
    
    -- 客户端可见的预初始化（dedicated 不需要）
    if not TheNet:IsDedicated() then
        -- 视觉、音效准备
        inst:DoPeriodicTask(1, MusicTick, 0)
    end
    
    inst.entity:SetPristine()
    
    if not TheWorld.ismastersim then
        -- 客户端联机到此返回（不需要 master 的业务）
        return inst
    end
    
    -- 服务端业务（主机 + dedicated）
    inst:AddComponent("health")
    inst:SetStateGraph("SGwarg")
    
    return inst
end
```

**4 段执行情况**：

| 段 | 单机 | 主机 | 客户端 | Dedicated |
| --- | --- | --- | --- | --- |
| 共享 | ✓ | ✓ | ✓ | ✓ |
| 非 dedicated | ✓ | ✓ | ✓ | ✗ |
| `if not ismastersim return` | 不返回 | 不返回 | **返回** | 不返回 |
| 服务端业务 | ✓ | ✓ | ✗（已 return） | ✓ |

---

### 9.4.6 进阶：客户端的"shard 视角"和 host 玩家的双重身份

#### 第一步：主机玩家既是 server 又是 client

**主机端联机**：玩家自己启动了游戏并开放给别人加入 —— 这个进程**同时是 server（TheWorld 在这里、master sim 在这里）和 client（玩家自己有 HUD、有控制角色）**。

**双重身份的判断**：

```lua
if TheNet:GetIsServer() and TheNet:GetIsClient() then
    -- 主机端联机的玩家 —— 既能写状态又有 GUI
end

-- 等价
if TheWorld.ismastersim and not TheNet:IsDedicated() then
    -- 同上
end
```

**这种情况下**：
- 服务端代码（如 `if TheWorld.ismastersim`）跑
- 客户端代码（如 `if not TheNet:IsDedicated()`）也跑
- **两份代码都在同一进程的同一帧里跑** —— **客户端和服务端的状态同时存在**

#### 第二步：客户端联机的"伪服务端"

客户端联机的玩家 —— `ismastersim = false` —— **client 根本没有 components.health**——只有 replica。

**典型陷阱**：客户端代码访问 `inst.components.xxx` 报 nil ——必须用 `replica`：

```lua
-- ❌
local hp = ThePlayer.components.health.currenthealth

-- ✓
local hp = ThePlayer.replica.health:Current()
```

#### 第三步：shard 视角

每个 shard 是**独立进程**——有自己的 TheWorld、自己的 ismastersim。

- 地表的主机 → 地表 TheWorld.ismastersim = true
- 地表的客户端 → 地表 TheWorld.ismastersim = false
- 客户端**进入洞穴**的瞬间 → 客户端只是连到了**洞穴 shard 的客户端代理** ——洞穴 shard 在另一个进程，**它的 TheWorld 和地表的 TheWorld 是两份**

---

### 9.4.7 老手进阶：常用判断模式实战

#### 模式 1：仅服务端创建组件

```lua
-- prefab fn 的 master_postinit 段
if TheWorld.ismastersim then
    inst:AddComponent("workable")
end
```

**问**：能省略 `if` 吗？  
**答**：通常能——因为 `master_postinit` 已经在 `if not TheWorld.ismastersim then return inst end` 之后——但**显式 if 更安全**，避免被未来重构破坏。

#### 模式 2：仅客户端 spawn 特效

```lua
local function OnAttacked(inst, data)
    if not TheNet:IsDedicated() then
        SpawnPrefab("hit_fx").Transform:SetPosition(inst.Transform:GetWorldPosition())
    end
end
```

**问**：为什么不用 `not TheWorld.ismastersim`？  
**答**：**单机和主机端**也有客户端 GUI——也要播特效。**`not IsDedicated`** 能覆盖"任何有 GUI 的部署"。

#### 模式 3：双重身份判断——主机的"客户端代码"也要跑

```lua
-- 想给"任何客户端"挂监听（包括主机自己的客户端）
if TheNet:GetIsClient() then
    inst:ListenForEvent("attacked", ClientShakeCamera)
end
```

**等价**：

```lua
if not TheNet:IsDedicated() then
    -- ...
end
```

#### 模式 4：modmain 顶层

```lua
-- modmain.lua 顶层
-- ❌ TheWorld 可能还 nil
-- if TheWorld.ismastersim then ... end

-- ✓ TheNet 是 C++ 单例，永远可用
if TheNet:GetIsServer() then
    -- 服务端 mod 配置
end

if TheNet:IsDedicated() then
    -- dedicated 专属配置（不想客户端跑）
end
```

#### 模式 5：保守模板——三段式

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    
    -- ① 共享段
    inst._var = net_bool(inst.GUID, "x.var", "vardirty")
    inst:AddTag("xxx")
    
    -- ② 客户端可见段（跳过 dedicated）
    if not TheNet:IsDedicated() then
        inst:ListenForEvent("vardirty", ClientFx)
    end
    
    inst.entity:SetPristine()
    
    if not TheWorld.ismastersim then
        return inst
    end
    
    -- ③ 服务端独有段
    inst:AddComponent("health")
    inst:SetStateGraph("SGxxx")
    inst:ListenForEvent("death", OnDeath)  -- 服务端独有事件
    
    return inst
end
```

**这是 90% prefab 的标准结构**——按 ① ② ③ 分段后基本不会写错。

---

### 9.4.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：modmain 顶层用 `TheWorld.ismastersim`

**症状**：`attempt to index nil value (TheWorld)` —— mod 加载就崩。  
**原因**：modmain 顶层在 TheWorld 创建之前。  
**修复**：用 `TheNet:GetIsServer()` 或包在 `AddSimPostInit` 里。

#### 陷阱 2：用 `not TheNet:GetIsClient()` 想跳过 GUI 代码

**症状**：dedicated 跳了 GUI——但**主机端**也跳了——主机玩家看不到特效。  
**原因**：主机端 `GetIsClient = true`（**主机自己也是 client**！）—— `not GetIsClient` **只在 dedicated 是 true**——**和 `IsDedicated` 等价**！  
**修复**：直接用 `TheNet:IsDedicated()` —— 语义更清晰。

#### 陷阱 3：在客户端联机访问 components

**症状**：`ThePlayer.components.health` 报 nil。  
**原因**：客户端没有 components。  
**修复**：`ThePlayer.replica.health` 或者 `ThePlayer:HasTag("xxx")` 等客户端可访问的接口。

#### 陷阱 4：以为 `ismastersim` 在 shard 之间可以"传递"

**症状**：mod 想"地表玩家的 ismastersim 决定洞穴的某事"——直接读地表的 ismastersim。  
**原因**：每个 shard 独立——TheWorld 各自一份。  
**修复**：通过 `SendModRPCToShard` 跨 shard 通信（9.2 节）。

#### 陷阱 5：在 OnEntityReplicated 里访问 `components`

**症状**：客户端 OnEntityReplicated 想做服务端逻辑——nil 错。  
**原因**：OnEntityReplicated 仅在客户端跑——客户端没 components。  
**修复**：OnEntityReplicated 只做客户端业务（attach classified、挂监听等）。

#### 陷阱 6：用 `TheNet:GetIsHosting()` 区分主机/dedicated

**症状**：mod 想"仅主机端跑"——用了 GetIsHosting——结果 dedicated 也跑了。  
**原因**：`GetIsHosting`：单机 + 主机 + dedicated 都 true。**它判断的是"我是 host"，不是"我是 client-host"**。  
**修复**：用 `TheNet:GetServerIsClientHosted()`（主机 true, dedicated false）。

#### 设计经验三条

**经验 ①：默认用 `TheWorld.ismastersim`**

90% 场景判断"是不是服务端"——用它。**例外只有**：modmain 顶层（用 `TheNet:GetIsServer`）。

**经验 ②：跳过 dedicated 永远用 `not TheNet:IsDedicated()`**

不要用 `not GetIsClient`——它和 `IsDedicated` 等价但语义不清晰。

**经验 ③：永远先想清楚部署模式**

写一段 mod 代码前，**先问 4 个问题**：
1. 单机能跑吗？
2. 主机端联机能跑吗？
3. 客户端联机能跑吗？
4. dedicated 能跑吗？

**4 个答案确定**——再选条件。**不要随手加个 `if TheNet:GetIsServer()` 然后 dedicated 出问题**。

---

### 9.4.9 小结

**4 个核心判断一句话总结**：**`ismastersim` 判服务端 / `IsDedicated` 判 dedicated / `GetIsClient` 判有 GUI / `GetIsHosting` 判 host**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 仅服务端业务 | `if TheWorld.ismastersim then ... end` |
| 跳过 dedicated 视觉 | `if not TheNet:IsDedicated() then ... end` |
| modmain 顶层判服务端 | `if TheNet:GetIsServer() then ... end` |
| 仅 client（含主机） | `if TheNet:GetIsClient() then ... end` |
| 仅主机（不含 dedicated） | `if TheNet:GetServerIsClientHosted() and TheWorld.ismastersim then ... end` |
| 任何端的"我是不是有 GUI" | `if not TheNet:IsDedicated() then ... end` |

**完整真值表**（核心简化版）

| | 单机 | 主机 | 客户端 | Dedicated |
| --- | --- | --- | --- | --- |
| `ismastersim` | T | T | F | T |
| `GetIsClient` | T | T | T | F |
| `IsDedicated` | F | F | F | T |

**6 个陷阱排雷顺序**

1. modmain 顶层用 ismastersim → 用 GetIsServer
2. `not GetIsClient` 想跳 GUI → 用 IsDedicated
3. 客户端访问 components → 用 replica
4. shard 间共享 ismastersim → 用 ShardRPC
5. OnEntityReplicated 访问 components → 客户端业务专用
6. GetIsHosting 区分主机/dedicated → GetServerIsClientHosted

**3 条设计经验**

- ① **默认 ismastersim** —— 90% 场景
- ② **跳 dedicated 用 IsDedicated** —— 语义清晰
- ③ **写代码前先想 4 种部署** —— 单机/主机/客户端/dedicated 都过一遍

> **下一节预告**：9.5 节我们讲 **客户端预测与服务端校正——延迟补偿的设计思路**。读完 9.5，你将理解"为什么客户端按下左键玩家立刻动起来、服务端确认后再校正"——这是饥荒联机版**输入流畅度**的根本机制。也是 mod 自定义 Action 时**最容易翻车**的环节。

## 9.5 客户端预测与服务端校正——延迟补偿的设计思路

### 本节导读

9.1 ~ 9.4 我们把网络通信的"管道"和"身份"全部讲清楚。但还有一个**联机版玩家最直接感知**的问题没解决——**延迟**。

想象一下：玩家点击屏幕——客户端通过 RPC 把这个动作发给服务端——服务端处理完——再通过 NetVar 把"玩家走过去了"同步回客户端——客户端**才**看到角色动起来。**这个往返延迟可能 200ms**——按一次键 200ms 后角色才动——**完全无法接受**。

Klei 的解决方案是 **客户端预测（Client Prediction）**：

- 客户端**立刻**让角色动起来（"假装服务端会同意"）
- 同时**异步**把请求发给服务端
- 服务端确认 → 没问题就继续；有问题就**纠正**客户端（snap 回正确位置）

**这种 "先做后问" 的设计**——让玩家感受不到延迟——但**代价是 mod 开发复杂度高一个量级**：必须维护客户端 / 服务端两套状态，处理冲突时还要"撤销"客户端预测。

> **新手**从 9.5.1-9.5.3 起步——理解客户端预测的存在意义、两个核心环节（移动 + 动作）、`PreviewBufferedAction` 工作流程；**进阶读者**继续看 9.5.4-9.5.6，深入服务端 `OnRemoteBufferedAction` 的"接住与校正"逻辑、`SGwilson` 与 `SGwilson_client` 双 SG 设计、locomotor 预测 walk 与回退 snap；**老手**跳到 9.5.7-9.5.8，看 mod 自定义 Action 的预测路径完整实战 + 6 个最容易翻车的陷阱。

---

### 9.5.1 快速入门：从一次"鼠标点击"看预测的存在意义

#### 第一步：没有客户端预测会怎样？

**伪场景**——客户端按下左键 → 砍树：

```
[T=0ms]   客户端按左键
[T=0ms]   客户端发 RPC LeftClick 到服务端
[T=100ms] 服务端收到（一半网络延迟）
[T=100ms] 服务端验证 + 创建 BufferedAction + 玩家 SG 走过去 + 砍
[T=100ms] 服务端通过 NetVar 同步玩家位置/动画
[T=200ms] 客户端收到（另一半网络延迟）
[T=200ms] 客户端的玩家**才**开始动
```

**结果**：玩家点击 → 200ms 后角色才动 → **手感粘滞、像在水里游泳**。

#### 第二步：有了预测会怎样？

```
[T=0ms]   客户端按左键
[T=0ms]   客户端立刻播放走路动画（PreviewBufferedAction）
[T=0ms]   客户端立刻发 RPC LeftClick 到服务端
          ──────以下是异步并行────────
[T=100ms] 服务端收到 + 验证 + 服务端的 SG 也走过去
[T=200ms] 服务端的位置通过 NetVar 同步回客户端
[T=200ms] 客户端检查自己的预测和服务端权威值是否一致
          一致 → 继续；不一致 → snap 到权威位置
```

**结果**：玩家点击 → **0ms 内角色立刻动** → 流畅手感。但**如果客户端预测错了**（比如服务端拒绝了这个动作），需要**视觉上回滚**——这是开发者的负担。

#### 第三步：饥荒里到底"预测"了什么？

**饥荒预测的两件事**：

1. **走路（locomotion）** —— 客户端按 WASD 或点击空地——立刻让玩家走起来
2. **动作（action）** —— 客户端按左键/右键/空格——立刻让玩家**假装**开始执行该动作（动画+SG state）

**没有预测的事**：
- 实际的"业务效果"——比如砍树扣血、捡起物品——只在服务端发生
- NetVar 状态变化——只能通过服务端权威同步

> **核心结论**：**客户端预测只预测"视觉表现"，不预测"业务结果"**。客户端只**假装** 已经走过去了、**假装** 已经开始砍树——但**真的扣 HP / 加经验 / 给物品**完全等服务端。

---

### 9.5.2 快速入门：客户端预测的两个核心环节—— locomotion + action

#### 第一步：locomotion（移动预测）

**触发**：玩家按 WASD 或点击空地。

**客户端流程**：
1. PlayerController 接到输入
2. 直接调 `locomotor:GoToPoint(pt)` —— 客户端的 locomotor 立刻让玩家走
3. 同时通过 RPC `RemotePredictWalking` 把"我打算走到这里"发给服务端
4. 服务端的 PlayerController 收到 → 在权威端执行同样的"走"
5. 服务端的 NetVar（位置）反向同步回客户端
6. 客户端比对：自己预测的位置 vs 服务端权威位置；如果差距太大，**snap** 到权威位置

回看 `playercontroller.lua:5687-5699`：

```5687:5699:scripts/components/playercontroller.lua
function PlayerController:RemoteBufferedAction(buffaction)
	if self.classified and self.classified.iscontrollerenabled:value() then
		if self.client_last_predict_walk.tick then
			local x, y, z = self.inst.Transform:GetWorldPosition() --V2C: not GetPredictionPosition()
			--V2C: Physics:Stop() fixed to stop at last sim position; no need to correct local position anymore
			--self.inst.Transform:SetPosition(x, 0, z) --V2C: correcting local position, no need to account for platform
			self:RemotePredictWalking(x, z, self.locomotor:GetTimeMoving() == 0, self.locomotor:PopOverrideTimeMoving(), self.client_last_predict_walk.direct)
			self.client_last_predict_walk.tick = nil
		end
		buffaction.preview_cb()
	else
		self.client_last_predict_walk.tick = nil
	end
end
```

#### 第二步：action（动作预测）

**触发**：玩家点击物品/实体/技能。

**客户端流程**：
1. PlayerController 计算出 `BufferedAction`
2. 调 `inst:PreviewBufferedAction(act)` —— 客户端的 SG 切到对应 state（播放动画）
3. 同时发 RPC LeftClick / RightClick 到服务端，包含目标和动作信息
4. 服务端 `OnRemoteLeftClick / OnRemoteRightClick` 接收 → 验证 → 创建权威 BufferedAction → 服务端 SG 也切 state → 服务端执行业务
5. 服务端的 SG 状态通过 `currentstate` NetVar 同步回客户端
6. 客户端 SG 比对：如果自己预测的 state 和服务端不一致，**回退到 idle**

回看 `entityscript.lua:1572-1598`：

```1572:1598:scripts/entityscript.lua
function EntityScript:PreviewBufferedAction(bufferedaction)
    if bufferedaction ~= nil and
        self.bufferedaction ~= nil and
        bufferedaction.target == self.bufferedaction.target and
        bufferedaction.action == self.bufferedaction.action and
        bufferedaction.invobject == self.bufferedaction.invobject and
        not (self.sg ~= nil and self.sg:HasStateTag("idle") and self:HasTag("idle")) then
        return
    end

    if bufferedaction.action == ACTIONS.WALKTO then
        self.bufferedaction = nil
	elseif bufferedaction.options.instant then
		self.bufferedaction = bufferedaction
		self:PerformPreviewBufferedAction()
    elseif self.sg ~= nil then
        self.bufferedaction = bufferedaction
        if not self.sg:PreviewAction(bufferedaction) then
            self.bufferedaction = nil
        end
    elseif bufferedaction.action.instant then
        self.bufferedaction = bufferedaction
        self:PerformPreviewBufferedAction()
    else
        self.bufferedaction = nil
    end
end
```

**核心三段**：
1. **重复检测** —— 如果当前 bufferedaction 和新 buffaction 相同，**直接 return**（避免重复预测）
2. **WALKTO 特殊处理** —— 移走的动作不需要预测（已经由 locomotion 处理）
3. **走 SG:PreviewAction** —— 让客户端 SG 切对应 state

#### 第三步：客户端 SG 的"PreviewAction"

回看 `stategraph.lua:405-428`：

```405:428:scripts/stategraph.lua
function StateGraphInstance:PreviewAction(bufferedaction)
    if self.sg.actionhandlers ~= nil then
        local handler = self.sg.actionhandlers[bufferedaction.action]
        if handler ~= nil then
            if handler.condition ~= nil and not handler.condition(self.inst) then
                return
            elseif handler.deststate ~= nil then
                local state = handler.deststate(self.inst, bufferedaction)
                if state ~= nil then
                    self:GoToState(state)
                    return true
                else
                    return
                end
            end
        elseif not bufferedaction.action.instant then
            self:GoToState("previewaction")
            return true
        end
    end

    self.inst:PerformPreviewBufferedAction()
    return true
end
```

**关键观察**：客户端 SG 的 `PreviewAction` 和**服务端**的 SG `actionhandlers` 是**同一套 lookup 表**——只是**客户端用 `SGwilson_client.lua`、服务端用 `SGwilson.lua`**。

> **设计哲学**：**客户端和服务端各跑一份 SG**——两份代码 90% 一致——客户端版去掉了"调 PerformBufferedAction"等业务相关代码——只留下"播动画/进 state"等视觉表现。

---

### 9.5.3 快速入门：`PreviewBufferedAction` 的工作流程

#### 第一步：完整的客户端预测链路

```
[客户端] PlayerController 检测到点击
    ↓
[客户端] PlayerActionPicker 计算 BufferedAction
    ↓
[客户端] PlayerController:DoAction(buffaction)
    ↓
[客户端] inst:PreviewBufferedAction(buffaction)
    ↓
[客户端] SG:PreviewAction(buffaction)
    ↓
[客户端] SG 切到 dolongaction / chop / pickup ...
    ↓
[客户端] PerformPreviewBufferedAction()
    ↓
[客户端] PlayerController:RemoteBufferedAction()
    ↓
[网络] SendRPCToServer(RPC.LeftClick, ...)
```

#### 第二步：客户端的"伪 BufferedAction"

`inst:PreviewBufferedAction(buffaction)` 会**把 buffaction 存到 `inst.bufferedaction`** —— 客户端的 SG 在执行 state 时**仍然能访问 `inst.bufferedaction.target` 等字段**（用于动画方向计算等）。

**关键差异**：客户端的 buffaction **不会调 `:Do()`**——业务执行权完全在服务端。

#### 第三步：客户端 buffaction 的清理

客户端 SG 进入 state 后——**等服务端确认或拒绝时，要么收到 `currentstate` NetVar 同步、要么收到 `failedaction` 通知**——客户端 SG 据此切回 idle 或保持。

如果**服务端拒绝**了这个动作：
- 服务端 `BufferedAction:Fail()` —— 通过 NetVar 通知客户端
- 客户端的 SG 切回 idle
- 客户端的 `inst.bufferedaction` 被清空

#### 第四步：用控制台观察预测过程

启动联机服务器（你做主机），用第二个账号连接做客户端——客户端控制台：

```lua
-- 移动到某点
ThePlayer.components.locomotor:GoToPoint(Vector3(c_select():GetPosition():Get()))

-- 看 bufferedaction
print(ThePlayer.bufferedaction)
-- 客户端预测期间能看到 BufferedAction; 服务端确认后被同步覆盖
```

---

### 9.5.4 进阶：服务端 `OnRemoteBufferedAction` 的"接住与校正"

#### 第一步：服务端如何"接住"客户端的预测

`scripts/components/playercontroller.lua:5702-5739`：

```5702:5739:scripts/components/playercontroller.lua
function PlayerController:OnRemoteBufferedAction()
    if self.ismastersim then
        --If we're starting a remote buffered action, prevent the last
        --movement prediction vector from cancelling us out right away
		local pt = self:GetRemotePredictPosition()
		if pt then
			if pt.y < 5 and not self:IsBusy() then
				--excludes self:IsLocalOrRemoteHopping() as well, ie. y ~= 6
				local x, _, z = self.inst.Transform:GetWorldPosition()
				local dx = pt.x - x
				local dz = pt.z - z
				if dx ~= 0 or dz ~= 0 then
					local should_snap
					if self.remote_authority then
						should_snap = true
					else
						local max_dist = self.locomotor:GetRunSpeed() * FRAMES
						should_snap =
							dx * dx + dz * dz <= math.min(max_dist * max_dist, PREDICT_STOP_ERROR_DISTANCE_SQ) and
							self.map:IsPassableAtPoint(pt:Get())
					end

					if should_snap then
						local dir = math.atan2(-dz, dx) * RADIANS
						if self.inst.sg:HasStateTag("canrotate") then
							self.locomotor:SetMoveDir(dir)
						end
						--Force us to interrupt and go to movement state immediately
						self.inst.sg:HandleEvent("locomote", { dir = dir, force_idle_state = true }) --force idle state in case this tiny motion was meant to cancel an action
						--FIXME(JBK): Boat handling.
						--FIXED(V2C): Remote predict position now resolves platform relative positions from client.
						self.locomotor:Stop()
						self.inst.Transform:SetPosition(pt.x, 0, pt.z)
					end
				end
			end
            self.remote_vector.y = 5
        elseif self.remote_vector.y == 0 then
```

**关键 3 段**：
1. **拿客户端预测位置** —— `self:GetRemotePredictPosition()`
2. **计算服务端 vs 客户端差距** —— `dx, dz` 距离的平方
3. **决定 snap 还是接受** —— `should_snap` 的判定

#### 第二步：snap 的精妙判定

```lua
local max_dist = self.locomotor:GetRunSpeed() * FRAMES
should_snap = dx * dx + dz * dz <= math.min(max_dist * max_dist, PREDICT_STOP_ERROR_DISTANCE_SQ) and
              self.map:IsPassableAtPoint(pt:Get())
```

**两个条件**：
1. **距离平方 ≤ `min(max_dist²,PREDICT_STOP_ERROR_DISTANCE_SQ)`** —— 客户端预测的位置距离服务端不能太远（最多 1 帧的移动距离）
2. **目标位置可通过** —— 不能预测到了水里/墙里

**意图**：**接受合理的小幅校正、拒绝离谱的预测**。

#### 第三步：snap 的执行

```lua
self.inst.sg:HandleEvent("locomote", { dir = dir, force_idle_state = true })
self.locomotor:Stop()
self.inst.Transform:SetPosition(pt.x, 0, pt.z)
```

**3 步**：
1. 强制 SG 切到移动 state（避免被 idle 卡住）
2. 停止当前 locomotor
3. **直接 SetPosition 到客户端预测的位置**

**这就是"snap"** —— 服务端把权威位置**直接拉到** 客户端预测位置——客户端就不会感受到位置撕裂。

#### 第四步：当客户端预测过激

如果客户端预测的位置**距离过远**（比如客户端 mod 想瞬移）—— `should_snap = false` —— 服务端**不接受这个预测** —— 强制服务端的位置广播 → 客户端被强制拉回 → **rubberbanding（橡皮筋效应）**。

> **设计哲学**：**服务端永远是权威**——客户端可以在合理范围内自由预测，越界则被纠正。**这是反作弊的关键**——客户端无法通过"预测错的位置"作弊。

---

### 9.5.5 进阶：`SGwilson` 与 `SGwilson_client` 的双 SG 设计

#### 第一步：双 SG 的存在

打开 `scripts/stategraphs/`：

```
SGwilson.lua            ← 服务端 SG（19000 行）
SGwilson_client.lua     ← 客户端 SG（5000 行）
SGwilsonghost.lua       ← 服务端鬼魂 SG
SGwilsonghost_client.lua← 客户端鬼魂 SG
```

**为什么要两份？**

**服务端 SG**：业务级 state——含 `PerformBufferedAction`、组件操作、事件 push、世界状态修改。  
**客户端 SG**：表现级 state——只播动画、播音效、设 tag——**绝不调 PerformBufferedAction**（业务在服务端）。

#### 第二步：典型差异——dolongaction state

服务端版（`SGwilson.lua:8216-8283`，9.4 节看过）：
```lua
ontimeout = function(inst)
    -- 服务端版：执行业务
    inst:PerformBufferedAction()
end,
```

客户端版（`SGwilson_client.lua:3133`）：
```lua
ontimeout = function(inst)
    -- 客户端版：仅切 state，不执行业务
    inst.sg:GoToState("idle")
end,
```

**关键**：客户端版**没有 PerformBufferedAction**——**业务永远在服务端发生**。

#### 第三步：mod 注册 ActionHandler 的双注册原则

7.7.4 讲过——mod 的自定义 Action 必须注册到**两份** SG：

```lua
AddStategraphActionHandler("wilson",        ActionHandler(ACTIONS.PET, "dolongaction"))
AddStategraphActionHandler("wilson_client", ActionHandler(ACTIONS.PET, "dolongaction"))
```

**两个 handler 的作用**：
- `"wilson"` —— 服务端：玩家走过去 → 切到 dolongaction → ontimeout 调 PerformBufferedAction → 业务发生
- `"wilson_client"` —— 客户端预测：玩家**也立刻**切到 dolongaction（播动画）→ ontimeout 时仅切回 idle（业务靠服务端同步）

#### 第四步：自定义客户端 SG state

如果你自定义了**客户端没有的 state**——服务端 SG 进了那个 state、客户端进不去——客户端会**卡住**。**修复**：

```lua
-- mymod/stategraphs/SGactions_pet.lua

-- 服务端版
local server_states = {
    State{
        name = "pet_action",
        ...
        timeline = {
            TimeEvent(30 * FRAMES, function(inst) inst:PerformBufferedAction() end),
        },
    },
}

-- 客户端版（去掉 PerformBufferedAction）
local client_states = {
    State{
        name = "pet_action",
        ...
        timeline = {
            -- 没有 PerformBufferedAction
        },
    },
}

return server_states, client_states
```

```lua
-- modmain.lua
local server_states, client_states = require("stategraphs/SGactions_pet")
for _, s in ipairs(server_states) do
    AddStategraphState("wilson", s)
end
for _, s in ipairs(client_states) do
    AddStategraphState("wilson_client", s)
end
```

---

### 9.5.6 进阶：locomotor 的预测 walk 与回退 snap

#### 第一步：客户端 locomotor 的工作

`scripts/components/locomotor.lua` —— 客户端版本和服务端版本是**同一份代码**——但运行时**只有客户端版本会"预测"**。

**典型流程**：
1. 客户端 PlayerController 接到 WASD
2. 调 `self.locomotor:RunForward()`
3. locomotor 内部 `OnUpdate(dt)` 每帧推进客户端的位置
4. 同时通过 `RemotePredictWalking` RPC 把"打算移动到"通知服务端
5. 服务端的 PlayerController 接收 → **服务端的 locomotor** 也走

#### 第二步：服务端的"回应"

服务端的位置通过普通的 `Transform` 网络同步——**不是 NetVar，是引擎层的 transform 网络复制**——客户端能看到服务端的权威位置。

**客户端比对**：
- 客户端自己的预测位置：`ThePlayer.Transform:GetWorldPosition()`（客户端模拟）
- 服务端权威位置：通过 transform 网络复制过来的位置

如果差距很小 → 没事；差距很大 → 客户端被**强制 snap** 到服务端位置——视觉上的"橡皮筋"。

#### 第三步：snap 的开发者影响

mod 自定义角色 / 自定义移动方式 时——如果客户端预测和服务端实际行为**差距太大**——会导致**频繁的 snap** —— 玩家感受到"卡顿"。

**修复**：保证客户端预测和服务端实际**逻辑一致**——通常是**同一份代码**。

---

### 9.5.7 老手进阶：mod 自定义 Action 的预测路径完整实战

#### 第一步：场景

假设 mod 加了一个新动作 `MYACTION`：玩家右键某物体 → 走过去 → 触发业务。

#### 第二步：完整链路

```
客户端预测路径：
[客户端] 右键 → PlayerActionPicker push ACTIONS.MYACTION
[客户端] PlayerController:DoAction(buffaction)
[客户端] inst:PreviewBufferedAction(buffaction)
[客户端] SGwilson_client 的 actionhandlers["MYACTION"] → 切到 "dolongaction"
[客户端] 玩家立刻看到自己走过去 + 蹲下动画
[客户端] PerformPreviewBufferedAction → SendRPCToServer(RPC.RightClick, ...)

服务端权威路径：
[服务端] 收到 RPC RightClick
[服务端] PlayerController:OnRemoteRightClick(...)
[服务端] 验证参数 + 创建 BufferedAction
[服务端] inst:PushBufferedAction(buffaction)
[服务端] 玩家走过去
[服务端] SGwilson 的 actionhandlers["MYACTION"] → 切到 "dolongaction"
[服务端] dolongaction.ontimeout → PerformBufferedAction → ACTIONS.MYACTION.fn(act)
[服务端] 业务发生（扣资源、生成实体、修改 NetVar）

同步路径：
[服务端] currentstate NetVar 同步给客户端
[客户端] SG 比对自己预测的 state vs 服务端 state；一致则继续，不一致则 snap
```

#### 第三步：mod 配置

```lua
-- 1) 注册动作
AddAction("MYACTION", "Do", function(act) ... end)

-- 2) 注册 ComponentAction（两端，让客户端也能 collector 出动作）
AddComponentAction("SCENE", "myable", function(inst, doer, actions, right) ... end)

-- 3) 注册双 ActionHandler（关键！）
AddStategraphActionHandler("wilson",        ActionHandler(ACTIONS.MYACTION, "dolongaction"))
AddStategraphActionHandler("wilson_client", ActionHandler(ACTIONS.MYACTION, "dolongaction"))
```

**漏第 3 步的客户端 handler** —— 客户端预测路径**走不通** —— **每次右键都"走过去 → 卡住" → 等 200ms 服务端拉回正常 state** —— 玩家感受为"卡顿"。

#### 第四步：高级——避免预测的 Action

如果你的 Action 涉及**客户端无法判断的业务**（比如"购买物品需要服务端验证金币"）——**不希望客户端预测**：

```lua
ACTIONS.BUY.do_not_locomote = true  -- 不需要走过去
-- 或者直接不注册 wilson_client handler——客户端预测会失败/不进 state
```

这样客户端就**只发 RPC，等服务端响应**——会感受到延迟但避免错误预测。

---

### 9.5.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：只注册 `wilson` 不注册 `wilson_client`

**症状**：联机时客户端按动作 → 走过去后**卡住** → 几秒后服务端"拉回"正常状态。  
**原因**：客户端 SG 收不到 ActionHandler → 切不到对应 state → 等服务端 NetVar 同步纠正。  
**修复**：永远注册双份。

#### 陷阱 2：客户端 SG state 调 `PerformBufferedAction`

**症状**：业务被执行两次（客户端一次 + 服务端一次）—— 双扣资源、双掉物品。  
**原因**：客户端不应该执行业务。  
**修复**：客户端版 state 移除 PerformBufferedAction。

#### 陷阱 3：服务端拒绝了客户端预测但 mod 自定义 SG state 没有"回退"路径

**症状**：客户端 SG 在 dolongaction 状态卡住——服务端拒绝后客户端没切回 idle。  
**原因**：自定义 state 没监听"failedaction" 事件或 NetVar 同步。  
**修复**：自定义 state 加 events handler：

```lua
events = {
    EventHandler("failedaction", function(inst)
        inst.sg:GoToState("idle")
    end),
},
```

#### 陷阱 4：客户端 mod 在 PreviewBufferedAction 里调 `inst.components.xxx`

**症状**：客户端报 nil。  
**原因**：客户端没有 components。  
**修复**：客户端 SG state 永远只用 replica + tag + NetVar。

#### 陷阱 5：自定义动作 Action.distance 不对，导致客户端走过去但服务端说"还没到"

**症状**：客户端预测"已经到了 → 切 dolongaction"——服务端"距离不够 → 失败"。  
**原因**：客户端预测的"到达距离"和服务端不一致。  
**修复**：永远用 `ACTIONS.MYACTION.distance` 配置——两端读同一个值。

#### 陷阱 6：让客户端预测高网络延迟下"瞬移"

**症状**：玩家网络延迟 500ms 时——客户端预测走 1 秒 → 服务端这边只走了 500ms 的位置 → snap 拉回。  
**原因**：客户端预测速度等同服务端实际速度——但**延迟越大、误差累积越大**。  
**修复**：mod 不应该自己加 client predictive movement——用引擎默认的 locomotor 逻辑。

#### 设计经验三条

**经验 ①：永远成对注册 wilson + wilson_client**

mod 自定义 Action 永远写两行 ActionHandler。**写一行就是 bug**。

**经验 ②：客户端只播动画、服务端做业务**

严格分层：
- 客户端 SG state：动画、音效、tag、视觉
- 服务端 SG state：动画 + 调用业务（PerformBufferedAction）

**经验 ③：测试 mod 永远在 dedicated + 远程客户端上**

主机端测试**看不到延迟问题** —— 必须用 dedicated + 远程客户端 + 模拟网络延迟（用 `c_setlatency(0.3)` 等命令）—— 才能发现"snap"和"预测错误"问题。

---

### 9.5.9 小结

**客户端预测一句话总结**：**客户端立刻播视觉、服务端权威决定结果、有差异则 snap 校正**——这是饥荒联机版输入流畅度的根本机制。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 让 mod Action 支持客户端预测 | 双注册 `AddStategraphActionHandler("wilson"/"wilson_client", ...)` |
| 让 mod Action 跳过预测 | `ACTIONS.MYACTION.do_not_locomote = true` 或不注册 client handler |
| 客户端 SG state 切回 idle | events 里 `EventHandler("failedaction", ...)` |
| 服务端 SG state 执行业务 | `inst:PerformBufferedAction()` |
| 客户端 SG state 不执行业务 | 移除 PerformBufferedAction |

**6 个陷阱排雷顺序**

1. 只注册 wilson 不注册 wilson_client → 双注册
2. 客户端调 PerformBufferedAction → 移除
3. 服务端拒绝后无回退 → 加 failedaction handler
4. 客户端访问 components → 用 replica
5. distance 两端不一致 → 用 Action.distance
6. 客户端预测瞬移 → 用引擎默认 locomotor

**3 条设计经验**

- ① **永远双注册 wilson + wilson_client**
- ② **客户端动画、服务端业务**严格分层
- ③ **测试用 dedicated + 远程客户端 + 模拟延迟**

> **下一节预告**：9.6 节我们将打开**库存与容器的网络同步**——这是联机 mod 中最复杂的一类同步——为什么 `inventory_classified.lua` 比 `player_classified.lua` 更复杂、为什么打开容器需要"发布订阅"模式、`itemslot` NetVar 数组如何处理增删改。读完 9.6，你将能写出"自定义容器、自定义库存槽位"的 mod，避免最常见的"客户端打开 UI 看到旧物品"等同步 bug。

## 9.6 Inventory / Container 的网络同步特殊处理

### 本节导读

9.1 ~ 9.5 我们把 NetVar、RPC、classified、双 SG 一一讲透——但**联机版里最复杂的同步系统**还没碰：**Inventory（玩家库存）和 Container（容器：箱子、背包、灶台）**。

为什么它们最复杂？

1. **库存内容是"动态数组"**——15 格物品，每格可能有可能没有——**比血量这种单值难得多**
2. **物品本身是 entity**——slot 里存的不是数字，而是**指向其他 entity 的引用**
3. **物品有"可见性"问题**——你箱子里的木头可能没人需要看，但可能某些情况下又要让别人看到（公开容器 / 多人同开）
4. **打开/关闭容器是"动态共享"**——同一个箱子，A 打开它时 A 才能看到内容，B 不能；B 也打开时**两人同时能看到**

> Klei 用一套**精巧的 classified + opener system + preview 机制**解决了这些问题。**这一节我们把这套系统拆开**——不仅是工作原理，更要搞清楚 **mod 自定义容器/库存时怎么避开 5 大同步坑**。

> **新手**从 9.6.1-9.6.3 起步——理解库存同步的存在意义、私有 vs 公开二元属性、classified slots 机制；**进阶读者**继续看 9.6.4-9.6.6，深入"按需共享"`SetClassifiedTarget`、`inventoryitem_classified` 道具私有数据、客户端的"乐观预览"`_itemspreview` 机制；**老手**跳到 9.6.7-9.6.8，看自定义容器完整实战、6 个最容易翻车的陷阱。

---

### 9.6.1 快速入门：从一次"打开箱子"看库存同步是什么

#### 第一步：在游戏里观察"打开箱子的瞬间发生了什么"

联机服务器，玩家 A 走到一个箱子前**右键打开**——

**A 看到的**：箱子界面**立刻**弹出，里面 6 格物品（3 个木头、1 把火把、2 个空格）清清楚楚。

**B 看到的**（站在隔壁）：**什么都看不到**——他没打开箱子，**他根本不知道里面有什么**。

**A 关闭箱子，走开**：箱子又"恢复神秘"——B 仍然不知道里面有什么。

**问题**：箱子里的物品**何时同步给 A**？**为什么 B 不知道**？**A 关闭后 B 还会一直收到无效的同步数据吗**？

#### 第二步：背后的核心机制——动态 classified 目标

回到 `container_replica.lua:191-206`：

```191:206:scripts/components/container_replica.lua
function Container:AddOpener(opener)
    local opencount = self.inst.components.container.opencount
    if opencount == 1 then
        --standard logic.
        SetOpener(self, opener)
    elseif opencount > 1 then
        self.classified.Network:SetClassifiedTarget(nil)
        if self.inst.components.container ~= nil then
            for k, v in pairs(self.inst.components.container.slots) do
                v.replica.inventoryitem:SetOwner(self.inst)
            end
        end
    end
    self.openers[opener] = self.inst:SpawnChild("container_opener")
    self.openers[opener].Network:SetClassifiedTarget(opener)
end
```

**核心 API**：`self.classified.Network:SetClassifiedTarget(opener)` —— **运行时改变** classified 的"可见目标"。

**3 种状态**：
- **0 人开** —— `SetClassifiedTarget(nil)` —— 谁都看不到（数据不同步）
- **1 人开** —— `SetClassifiedTarget(opener)` —— **只**该玩家能看到
- **多人开** —— `SetClassifiedTarget(nil)` + 给每个 opener spawn 一个 `container_opener` —— **每个 opener 各看一份**

**这就是答案**：箱子的内容**只在有人打开时才同步**，而且**只同步给打开者**——B 永远收不到 A 的箱子内容。

#### 第三步：完整的"打开箱子"链路

```
[客户端 A] 右键箱子 → ACTIONS.RUMMAGE
    ↓
[网络] RPC 到服务端
    ↓
[服务端] 接收 → component.container:Open(playerA)
    ↓
[服务端] container_replica:AddOpener(playerA)
    ↓
[服务端] container_classified.Network:SetClassifiedTarget(playerA)  ← 关键！
    ↓
[网络] 此时引擎才把 container_classified 的 NetVar 同步给 A
    ↓
[客户端 A] container_classified 的 _items[1..6] NetVar 收到值
    ↓
[客户端 A] _items[1]dirty 事件触发
    ↓
[客户端 A] HUD 重绘箱子内容
```

> **核心结论**：**Container 用的是"按需 classified"** —— 只在打开时才把内容同步给打开者，关闭后停止同步。这是 Klei 防止"网络爆炸"的关键设计——一个服务器有几百个箱子，**如果全员实时同步全部物品** —— 带宽会爆。

---

### 9.6.2 快速入门：库存的"私有 vs 公开"二元属性

#### 第一步：玩家 inventory 的"半私有"特点

玩家 inventory（背包 + 装备槽）和 container（箱子）在网络同步上**不完全一样**：

| 维度 | 玩家 inventory | 容器 container |
| --- | --- | --- |
| **可见者** | 玩家本人**永远**能看到 | 仅在打开时该玩家能看到 |
| **载体** | `inventory_classified`（永远活在 player 上） | `container_classified`（活在 container 上） |
| **slots NetVar** | 静态数量（背包 15 + 装备 5） | 动态数量（看 widgetsetup） |
| **触发同步** | 永远同步给本人 | 仅 opener 同步 |
| **替换装备视觉** | 装备**对所有人可见**（手里拿斧头看得见）| 容器内容**仅 opener 可见** |

#### 第二步：装备的"双向同步"

装备同时是 inventory 内容和外观——所以它**两条同步路径并行**：

1. **作为"库存内容"** —— 通过 `inventory_classified` 同步给本人（让本人看到 UI 上的装备槽）
2. **作为"外观"** —— 通过角色的 `AnimState` 网络同步给所有人（让别人看到角色身上拿着斧头）

#### 第三步：典型 inventory 同步路径

```
[服务端] inventory:GiveItem(item) → slot 6 = item
    ↓
[服务端] PushEvent("itemget", { slot = 6, item = item })
    ↓
[服务端] inventory_replica:SetItemAtSlot(item, 6)
    ↓
[服务端] inventory_classified.items[6]:set(item)
    ↓
[网络] 仅同步给本玩家（因为 player_classified.items 挂在 player 下）
    ↓
[客户端 本人] items[6]dirty 事件
    ↓
[客户端 本人] HUD 物品栏第 6 格显示物品
```

**B 玩家 / 其他客户端**：他们的 player_classified 是**他们自己的**，**没有 A 的 inventory 信息**——所以 B 看不到 A 背包里有什么物品。**只能通过角色身上的"装备外观"看到 A 手里拿什么**。

---

### 9.6.3 快速入门：classified slots 机制

#### 第一步：slots 的 NetVar 数组

`scripts/prefabs/container_classified.lua:901-903`：

```901:903:scripts/prefabs/container_classified.lua
    for i = 1, containers.MAXITEMSLOTS do
        table.insert(inst._itemspool, net_entity(inst.GUID, "container._items["..tostring(i).."]", "items["..tostring(i).."]dirty"))
    end
```

**`MAXITEMSLOTS`** 是上限（约 30）——**预先创建一组 NetVar**——后续 `InitializeSlots` 按需启用。

**为什么用 net_entity 数组？**

- 每格 slot 存一个**指向物品 entity 的引用**
- `net_entity` 通过 GUID 同步——客户端能拿到对应物品 entity 的客户端代理
- 增删改全部走"set NetVar"——不需要专门的 RPC 通知

#### 第二步：服务端 `SetSlotItem` 的写入

`container_classified.lua:28-40`：

```28:40:scripts/prefabs/container_classified.lua
local function SetSlotItem(inst, slot, item, src_pos)
    if inst._items[slot] ~= nil then
        inst._items[slot]:set(item)

        if item ~= nil and inst._items[slot]:value() == item then
            local inventoryitem = item.replica.inventoryitem
            inventoryitem:SerializeUsage()
            inventoryitem:SetPickupPos(src_pos)
        else
            inst._items[slot]:set(nil)
        end
    end
end
```

**写入流程**：
1. 服务端 container 的 slot 6 改了 → 调 `SetSlotItem(classified, 6, newitem, pos)`
2. classified 内部 `_items[6]:set(item)`
3. 如果 item 是有效引用 → 同时同步 `inventoryitem` 的子状态

#### 第三步：`InitializeSlots` 的动态分配

```8:22:scripts/prefabs/container_classified.lua
local function InitializeSlots(inst, numslots)
    --Can't re-initialize slots after RegisterNetListeners
    assert(inst._slottasks == nil)

    local curslots = #inst._items
    if numslots > curslots then
        for i = curslots + 1, numslots do
            table.insert(inst._items, table.remove(inst._itemspool, 1))
        end
    elseif numslots < curslots then
        for i = curslots, numslots + 1, -1 do
            table.insert(inst._itemspool, 1, table.remove(inst._items))
        end
    end
end
```

**做什么**：从 `_itemspool`（预创建池）取出指定数量的 NetVar 放到 `_items`（实际使用）—— 实现**按 widget 类型动态分配** slot 数量。

**典型场景**：
- 普通箱子 9 格 → InitializeSlots(9)
- 背包 8 格 → InitializeSlots(8)
- 灶台 4 格 → InitializeSlots(4)

#### 第四步：客户端的 dirty listener

```lua
-- 简化伪码
for i = 1, MAXITEMSLOTS do
    inst:ListenForEvent("items[" .. i .. "]dirty", function()
        OnSlotItemDirty(inst, i)
    end)
end

local function OnSlotItemDirty(inst, slot)
    local item = inst._items[slot]:value()
    -- HUD 更新该 slot 的显示
    inst._parent.HUD:UpdateSlot(slot, item)
end
```

---

### 9.6.4 进阶：Container 的"按需共享"——`SetClassifiedTarget`

#### 第一步：什么是 `SetClassifiedTarget`？

`SetClassifiedTarget(target)` —— **C++ 引擎层的网络 API** —— 设置 classified entity 的"目标客户端"：

- `SetClassifiedTarget(player)` —— 仅同步给该 player 的客户端
- `SetClassifiedTarget(nil)` —— 不同步给任何人（暂停同步）

**和 SetParent 的区别**：

| API | 作用 | 时机 |
| --- | --- | --- |
| `SetParent` | 静态父子关系——网络层附属 | prefab 创建时调一次 |
| `SetClassifiedTarget` | 动态可见目标——可运行时切换 | 任意时刻调，立刻生效 |

#### 第二步：Container 的"3 状态机"

回到 `container_replica.lua:191-226`：

```191:226:scripts/components/container_replica.lua
function Container:AddOpener(opener)
    local opencount = self.inst.components.container.opencount
    if opencount == 1 then
        --standard logic.
        SetOpener(self, opener)
    elseif opencount > 1 then
        self.classified.Network:SetClassifiedTarget(nil)
        if self.inst.components.container ~= nil then
            for k, v in pairs(self.inst.components.container.slots) do
                v.replica.inventoryitem:SetOwner(self.inst)
            end
        end
    end
    self.openers[opener] = self.inst:SpawnChild("container_opener")
    self.openers[opener].Network:SetClassifiedTarget(opener)
end

function Container:RemoveOpener(opener)
    local opencount = self.inst.components.container.opencount
    if opencount == 0 then
        SetOpener(self, nil)
    elseif opencount == 1 then
        SetOpener(self, table.getkeys(self.inst.components.container.openlist)[1])
    elseif opencount > 1 then
        self.classified.Network:SetClassifiedTarget(nil)
        if self.inst.components.container ~= nil then
            for k, v in pairs(self.inst.components.container.slots) do
                v.replica.inventoryitem:SetOwner(self.inst)
            end
        end
    end
    if self.openers[opener] then
        self.openers[opener]:Remove()
        self.openers[opener] = nil
    end
end
```

**3 状态**：

| opencount | 行为 |
| --- | --- |
| 0 | 没人开 → classified 不同步 |
| 1 | 单开 → classified 同步给该 opener |
| 2+ | 多开 → classified.Network:SetClassifiedTarget(**nil**)；改用 container_opener 子实体 |

**多开为什么要变？**

`SetClassifiedTarget` **只能设一个目标**——不能设多个。**两个人同时开同一个箱子**——必须用别的机制。

**Klei 的方案**：给每个 opener spawn 一个 `container_opener` 子 entity——挂自己的 classified target——**让每个 opener 都各有一个"私有视角"**。

#### 第三步：container_opener 子实体的精妙

```204:205:scripts/components/container_replica.lua
    self.openers[opener] = self.inst:SpawnChild("container_opener")
    self.openers[opener].Network:SetClassifiedTarget(opener)
```

**每个 opener 一个 container_opener 子 entity** —— 各自管理"我这个 opener 该看到什么"。**关闭时 Remove 子 entity** —— 自动清理。

> **设计哲学**：**用动态 entity spawn / despawn 实现"按需共享"** —— 这是网络协议层的精巧设计。

---

### 9.6.5 进阶：`inventoryitem_classified` 与"道具私有数据"

#### 第一步：物品的"持久状态"如何同步？

物品有很多 **状态需要同步给持有者**——但**不希望广播给所有人**：

- 武器的"剩余使用次数"（finiteuses）
- 食物的"鲜度"（perishable）
- 装备的"耐久"（armor）
- 工具的"魔力等级"

**直接挂在物品 entity 上的 NetVar** → 全员可见 → 别人能看到你的剑还剩多少耐久（公开 hint）。

**Klei 的方案**：用 `inventoryitem_classified` —— **物品的私有数据载体** —— 仅同步给当前持有者。

#### 第二步：使用 inventoryitem_classified 的物品

不是所有物品都用这套——只有**有"私有持续状态"**的物品才用：

| 物品类型 | 是否用 inventoryitem_classified |
| --- | --- |
| 木头、石头、肉（堆叠物品） | 否（只有 stacksize，普通 NetVar） |
| 武器（耐久） | 是 |
| 食物（鲜度） | 是 |
| 装备（耐久 + 修理状态） | 是 |
| 法术书（冷却） | 是 |

#### 第三步：流程示例

```
[服务端] 玩家挥剑 → finiteuses 减 1
    ↓
[服务端] inventoryitem_classified（属于该剑）.uses:set(newval)
    ↓
[网络] 因为 inventoryitem_classified 的 target 设置为持有者
    ↓
[客户端 持有者] usesdirty 事件
    ↓
[客户端 持有者] HUD 物品图标的"耐久条"重绘
    
[其他玩家] 看不到耐久变化 → 只能通过外观（破损贴图等）粗略推测
```

#### 第四步：取走/丢弃时的目标切换

物品**被丢出**或**被别的玩家捡起**时——`inventoryitem_classified` 的 target 切换：

- 玩家 A 丢下剑 → 服务端 SetClassifiedTarget(nil)
- 玩家 B 捡起剑 → 服务端 SetClassifiedTarget(B)

**通过这个动态切换**——耐久信息**仅当前持有者可见**。

---

### 9.6.6 进阶：客户端的"乐观锁"——`_itemspreview`

#### 第一步：网络延迟下的库存操作

玩家点击"把背包槽 3 的木头移到槽 5"——客户端立刻看到木头从槽 3 跳到槽 5——**这其实是预测**：

- 服务端**还没确认**这个操作
- 客户端**已经把它显示出来了**——给玩家流畅手感
- 服务端确认后 → 一切如客户端预测；服务端拒绝（比如槽 5 满了）→ 回滚

**`_itemspreview`** 就是客户端"预测中的状态"。

#### 第二步：预测/真实双状态共存

```46:60:scripts/prefabs/container_classified.lua
local function ProxyItem(item, context)
	if item then
		local proxy_item = EntityScriptProxy(item)
		proxy_item:SetProxyProperty("stackable_preview_context", context)

		--See if we need to transfer preview stacksize over
		if item.stackable_preview_context and item.stackable_preview_context ~= context then
			local stackable = item.replica.stackable
			if stackable then
				stackable:SetPreviewStackSize(stackable:StackSize(), context, TIMEOUT)
			end
		end
		return proxy_item
	end
end
```

**`EntityScriptProxy`** —— 给物品 entity 包一层"预览代理"——客户端临时显示但不修改真实物品。

#### 第三步：客户端"乐观操作"流程

```
[客户端] 玩家拖拽槽 3 → 槽 5
    ↓
[客户端] _itemspreview = clone(_items)，并修改 _itemspreview[5] = 槽 3 的物品、_itemspreview[3] = nil
    ↓
[客户端] HUD 立刻显示新位置
    ↓
[客户端] 发 RPC 到服务端
    ↓
[服务端] 验证 + 实际操作 + 同步 _items NetVar
    ↓
[客户端] 收到 _items 同步 → 与 _itemspreview 比对
    ↓
   一致 → 清空 _itemspreview，HUD 不变
   不一致 → 清空 _itemspreview，HUD 显示真实状态（视觉回滚）
```

#### 第四步：超时回滚

`TIMEOUT = 2` —— **2 秒内服务端没确认**就视为失败 —— 回滚 `_itemspreview` 到真实状态。**避免客户端长时间显示"伪状态"**。

---

### 9.6.7 老手进阶：自定义容器实战

#### 第一步：场景

mod 想加一个 4 格的"小工具箱"——玩家右键打开 → 看到 4 格 → 可放入物品 → 关闭。

#### 第二步：标准 prefab fn

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    
    inst.AnimState:SetBank("toolbox")
    inst.AnimState:SetBuild("toolbox")
    inst.AnimState:PlayAnimation("idle")
    
    inst:AddTag("structure")
    
    inst.entity:SetPristine()
    
    if not TheWorld.ismastersim then
        return inst
    end
    
    -- 服务端：添加 container 组件 + 配置 widget
    inst:AddComponent("container")
    inst.components.container:WidgetSetup("toolbox")  -- ★ widget 配置定义在 containers.lua
    
    inst:AddComponent("inspectable")
    
    return inst
end
```

#### 第三步：注册 widget 配置

```lua
-- modmain.lua
local containers = require "containers"

-- 给 toolbox 配置 widget
local function toolbox_params(params)
    params.toolbox = {
        widget = {
            slotpos = {
                Vector3(0, 32, 0),
                Vector3(0, -32, 0),
                Vector3(64, 32, 0),
                Vector3(64, -32, 0),
            },
            animbank = "ui_chest_3x2",  -- 用箱子的 UI
            animbuild = "ui_chest_3x2",
            pos = Vector3(0, 200, 0),
        },
        type = "chest",
    }
end

-- 应用到 containers.params
local _MakeNumSlots = containers.params
toolbox_params(containers.params)

-- 让 widgetsetup 能识别 toolbox
function containers.widgetsetup(container, prefab, data)
    local p = containers.params[prefab or container.inst.prefab]
    if p then
        for k, v in pairs(p) do
            container[k] = v
        end
        if data then
            -- ...
        end
    end
end
```

> **关键**：`WidgetSetup("toolbox")` 内部会调 `inst.components.container.classified:InitializeSlots(self:GetNumSlots())` —— 自动按 4 格初始化 classified 的 slots。

#### 第四步：服务端关闭逻辑

```lua
-- 自动关闭：玩家移动太远时
inst.components.container:SetOnOpenFn(function(inst, doer)
    -- ...
end)

inst.components.container:SetOnCloseFn(function(inst, doer)
    -- 关闭时保存状态
end)
```

#### 第五步：测试

```lua
-- 控制台
local box = c_spawn("toolbox")
c_give("log", 5)
-- 右键 toolbox 应该打开界面
-- 拖物品进去应该正常工作
```

---

### 9.6.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：自定义 mod inventory 没用 classified

**症状**：mod 给某 entity 加了"库存"NetVar 数组——所有玩家都能看到——**没有隐私**。  
**原因**：直接挂 NetVar 而不是用 classified。  
**修复**：复用 Klei 的 `container` 组件 + `WidgetSetup` —— 不要自己造库存协议。

#### 陷阱 2：客户端访问 `inst.components.container`

**症状**：客户端 mod 想读容器内容 → 报 nil。  
**原因**：客户端没 components。  
**修复**：用 `inst.replica.container:GetItemInSlot(i)` 或 `inst.replica.container:GetItems()`。

#### 陷阱 3：SetWidgetSetup 时的物品数量超过 MAXITEMSLOTS

**症状**：mod 想做 50 格的大容器 —— 报 assert 错。  
**原因**：`MAXITEMSLOTS` 是 classified 预创建池上限。  
**修复**：分几个 sub container；或 fork 引擎层（不可行，避免）。

#### 陷阱 4：在 container 关闭瞬间访问内容

**症状**：mod 想"玩家关闭箱子时记录里面有什么"——但 onclose 内 container.replica 已经清了。  
**原因**：classified 的 target 已切到 nil——客户端立即停止接收数据。  
**修复**：在 `onclosefn` 服务端 hook 里访问 `inst.components.container.slots` —— 服务端永远有真实数据。

#### 陷阱 5：客户端 `_itemspreview` 长时间不清

**症状**：mod 自定义 RPC 操作库存 —— 服务端没及时同步 NetVar —— 客户端 preview 残留 2 秒后被强制清。  
**原因**：服务端 RPC handler 处理慢或者忘了实际更新 NetVar。  
**修复**：服务端 RPC handler 必须**立刻**调 `SetSlotItem` 同步 classified。

#### 陷阱 6：自定义容器没注册 `inventoryitem_classified` 替换

**症状**：自定义武器 mod —— 耐久通过普通 NetVar 同步给所有人 —— 隐私泄露 + 网络浪费。  
**原因**：忘了用 inventoryitem_classified。  
**修复**：直接用 Klei 的 `finiteuses` / `armor` 组件 —— 它们内部已经走 inventoryitem_classified。

#### 设计经验三条

**经验 ①：**永远复用 Klei 的 container 组件——**不要自己造库存协议**

库存 / 容器同步**极其复杂**——预测 + 多 opener + 物品私有数据 + 自动关闭。**Klei 已经帮你做完了**——通过 `WidgetSetup` 配置一下就能用。**自己写一套** = **9 成的 bug 和 1 成的功能**。

**经验 ②：客户端永远走 replica，不直接碰 classified**

```lua
-- ❌
local item = box.container_classified._items[1]:value()

-- ✓
local item = box.replica.container:GetItemInSlot(1)
```

**经验 ③：自定义物品状态用 Klei 现成组件**

需要"耐久" → `finiteuses`；需要"鲜度" → `perishable`；需要"温度" → `temperature`——**这些组件内部都已用 inventoryitem_classified 隐藏私有数据**——直接用就行。

---

### 9.6.9 小结

**Inventory / Container 同步一句话总结**：**Container 用动态 SetClassifiedTarget 实现"按需共享"——只有 opener 才同步；Inventory 永远同步给本人；物品私有状态走 inventoryitem_classified**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 加自定义容器 | `inst:AddComponent("container") + container:WidgetSetup("yourname")` |
| 客户端读容器 | `inst.replica.container:GetItemInSlot(i)` / `GetItems()` |
| 客户端读玩家库存 | `ThePlayer.replica.inventory:GetItemInSlot(i)` |
| 服务端给玩家物品 | `player.components.inventory:GiveItem(item)` |
| 服务端把物品塞容器 | `box.components.container:GiveItem(item)` |
| 监听容器打开 | `inst:ListenForEvent("onopen", fn)` |
| 监听 slot 变化 | `inst:ListenForEvent("itemget", fn)` / `"itemlose"` |

**6 个陷阱排雷顺序**

1. 自定义 mod inventory 没用 classified → 复用 container 组件
2. 客户端访问 components → 用 replica
3. 容器超 MAXITEMSLOTS → 分多个
4. 关闭瞬间访问 replica → 服务端访问 components
5. 客户端 _itemspreview 残留 → 服务端立刻同步 NetVar
6. 物品状态没用 inventoryitem_classified → 用 finiteuses/perishable 等

**3 条设计经验**

- ① **复用 Klei 的 container** —— 不自造协议
- ② **客户端走 replica** —— 不直接碰 classified
- ③ **物品状态用 Klei 组件** —— 自动私有化

> **下一节预告**：9.7 节是整章收尾——**常见网络 bug 的排查与避免**——把整章 9.1~9.6 学到的所有同步机制串起来，做成一份**网络问题诊断手册**——按症状定位问题、附上解决方案。读完 9.7，你将能在调试联机 mod 时**快速定位**任何同步类 bug（NetVar 没更新 / RPC 没触发 / classified 看不到 / 客户端预测异常 / 跨 shard 数据不一致）。

## 9.7 常见网络 bug 的排查与避免

### 本节导读

第 9 章的 9.1 ~ 9.6 我们把网络同步的**机制**全部讲完了——NetVar、RPC、Classified、双 SG 预测、库存容器同步——**理论上你已经能写联机 mod 了**。

但**真正写起来**——你会发现 95% 的开发时间都花在**调试网络 bug**上。常见症状包括：

- "我的 mod 单机能跑，联机时客户端看不到数据"
- "玩家点了按钮，服务端没反应"
- "动作做了一半客户端突然回到 idle"
- "玩家位置反复闪回（rubberbanding）"
- "网络突然卡爆——服务器吃满 CPU"
- "数据某些情况下能同步，某些情况下不能"

**这些 bug 是 mod 联机最难调试的部分**——单机调试能复现的 bug **联机调试不一定能复现**——反之亦然。

这一节是**整章的诊断手册**——按**症状分类** → **可能原因** → **排查步骤** → **修复方案**。读完 9.7，你将能在调试联机 mod 时**快速定位**任何同步类 bug。

> **新手**从 9.7.1-9.7.4 起步——理解网络 bug 的 3 大类、按症状对症下药；**进阶读者**继续看 9.7.5-9.7.6，掌握调试工具和性能分析；**老手**跳到 9.7.7-9.7.8，看跨 shard / 持久化坑、6 个最难定位的隐蔽陷阱。

---

### 9.7.1 快速入门：网络 bug 的 3 大类 + 一份诊断地图

#### 第一步：3 大类

| 类别 | 表现 | 根因 |
| --- | --- | --- |
| **数据没同步** | 客户端看不到 / 看到旧的 | NetVar 没设值、classified 没创建、replica 没注册 |
| **行为没触发** | 客户端动作没反应 / 服务端没响应 | RPC 没注册 / handler 没跑 / 双 SG 漏一边 |
| **状态不一致** | 闪烁、回弹、超量执行 | 客户端预测错、服务端拒绝后没回退、双方代码有差 |

#### 第二步：诊断决策树

```
症状是什么？
├── 客户端看不到数据 / 看到旧数据
│   ├── NetVar 没创建 → 9.7.2 a
│   ├── classified target 不对 → 9.7.2 b
│   └── replica 没注册 → 9.7.2 c
├── 客户端动作没反应 / 走过去卡住
│   ├── 双 SG 漏注册 → 9.7.3 a
│   ├── RPC 没注册 → 9.7.3 b
│   └── 客户端 collector 不可见 → 9.7.3 c
├── rubberbanding / 闪烁
│   ├── 客户端预测过激 → 9.7.4 a
│   ├── 服务端拒绝没回退 → 9.7.4 b
│   └── 双方逻辑不一致 → 9.7.4 c
└── 性能 / 带宽问题
    ├── NetVar 高频 set → 9.7.6 a
    ├── 全图 update 组件过多 → 9.7.6 b
    └── 超大 net_string → 9.7.6 c
```

#### 第三步：定位的"3 步法"

**第 1 步：精确复现** —— 必须在 dedicated 服务器 + 远程客户端环境下复现，主机端调试通常发现不了。

**第 2 步：确定哪一端有问题**：
- 主机端控制台看主机端状态
- 客户端控制台看客户端状态
- 比对两边——找出**不一致点**

**第 3 步：缩小范围** —— 通过 `print` 日志逐步定位到具体函数 / 具体行。

---

### 9.7.2 症状一：客户端看不到数据 / 看到旧数据

#### 子类 a：NetVar 没创建 / 没设值

**症状**：

```lua
-- 客户端控制台
print(c_select().myvar)
-- => nil
```

**排查步骤**：

1. 客户端 entity 是不是有这个 NetVar？
   ```lua
   for k, v in pairs(c_select()) do
       if type(v) == "userdata" then print(k) end
   end
   -- 看 myvar 在不在
   ```

2. 主机端有没有 set？
   ```lua
   -- 主机端
   c_select().myvar:set(123)  -- 手动 set
   -- 然后看客户端 myvar:value()
   ```

**常见原因 + 修复**：

- ❌ NetVar 创建在 `if not TheWorld.ismastersim then return inst end` 之后 → 客户端没有
- ✅ NetVar 创建在 `SetPristine` 之前、两端都跑的代码段
- ❌ 忘了 `inst.entity:AddNetwork()` → 网络不工作
- ✅ 永远第一行加上

#### 子类 b：classified target 不对

**症状**：

```lua
-- 客户端
print(ThePlayer.player_classified)  -- nil 或没你的 NetVar
```

**排查步骤**：

1. classified entity 是不是创建了？
   ```lua
   -- 主机端
   print(ThePlayer.myclassified)
   -- 应该非 nil
   ```

2. SetParent 调了吗？
   ```lua
   print(ThePlayer.myclassified.entity:GetParent())
   -- 应该是 ThePlayer
   ```

3. 客户端 OnEntityReplicated 跑了吗？
   ```lua
   -- 在客户端 prefab fn 加 print，重连后看是否输出
   ```

**常见原因 + 修复**：

- ❌ 忘了 `AddTag("CLASSIFIED")` → 引擎不知道是私有
- ❌ 忘了 SetParent → 引擎不知道同步给谁
- ❌ classified 创建在主机端 master_postinit 之外 → 没创建
- ✅ 全部按 9.3.2 标准模板写

#### 子类 c：replica 没注册

**症状**：

```lua
-- 客户端
print(ThePlayer.replica.mycomp)  -- nil
```

**常见原因 + 修复**：

- ❌ mod 没调 `AddReplicableComponent("mycomp")`
- ✅ 在 modmain.lua 顶层调一次：
  ```lua
  AddReplicableComponent("mycomp")
  ```
- ❌ replica 文件名不对 → 必须 `scripts/components/mycomp_replica.lua`
- ✅ 文件名后缀是 `_replica.lua`

---

### 9.7.3 症状二：客户端动作没反应 / 走过去卡住

#### 子类 a：双 SG 漏注册

**症状**：联机时玩家右键 mod 自定义动作 → 角色走过去 → **卡住几秒** → 服务端拉回 idle。

**排查步骤**：

1. mod 注册了 `wilson_client` 的 ActionHandler 吗？
   ```lua
   -- modmain.lua 应该有
   AddStategraphActionHandler("wilson_client", ActionHandler(ACTIONS.MYACTION, "dolongaction"))
   ```

2. 客户端 SG 接到 dolongaction 了吗？看客户端 SGwilson_client.lua 是否有这个 state。

**修复**：

```lua
-- 永远成对注册
AddStategraphActionHandler("wilson",        ActionHandler(ACTIONS.MYACTION, "dolongaction"))
AddStategraphActionHandler("wilson_client", ActionHandler(ACTIONS.MYACTION, "dolongaction"))
```

#### 子类 b：RPC 没注册或注册时机不对

**症状**：

```lua
-- 客户端调
SendModRPCToServer(GetModRPC("mymod", "DoX"), 10)
-- 服务端没反应
```

**排查步骤**：

1. AddModRPCHandler 注册位置对吗？
   ```lua
   -- 必须在 modmain.lua 顶层
   AddModRPCHandler("mymod", "DoX", function(player, x) ... end)
   ```

2. 在主机端控制台看 namespace 是否注册：
   ```lua
   print(MOD_RPC.mymod)  -- 应该非 nil
   print(MOD_RPC.mymod.DoX)  -- 应该有 namespace 和 id
   ```

3. handler 第一行加 `print("RPC fired:", ...)` 看是否触发。

**常见原因 + 修复**：

- ❌ 注册在 PostInit 里 → 客户端没注册——发不出去
- ✅ 永远 modmain.lua 顶层
- ❌ namespace 拼写不一致 → "mymod" vs "myMod"
- ✅ 用 `local NS = "mymod"; AddModRPCHandler(NS, ...)` 避免拼写错

#### 子类 c：客户端 collector 不可见

**症状**：客户端 mod 自定义动作的"右键菜单选项"**不显示**。

**排查步骤**：

1. AddComponentAction 是不是两端都跑？  
   `AddComponentAction` 在 modmain 顶层就保证两端都注册。如果在某 PostInit 里就有可能漏。

2. collector 内部是不是访问了 `inst.components.xxx`？  
   ```lua
   AddComponentAction("SCENE", "myable", function(inst, doer, actions, right)
       -- ❌ 客户端没 components.myable
       if inst.components.myable:CanDo() then
           table.insert(actions, ACTIONS.MYACTION)
       end
   end)
   ```

**修复**：

```lua
AddComponentAction("SCENE", "myable", function(inst, doer, actions, right)
    -- ✓ 用 replica
    if inst.replica.myable and inst.replica.myable:CanDo() then
        table.insert(actions, ACTIONS.MYACTION)
    end
end)
```

---

### 9.7.4 症状三：服务端 ↔ 客户端不一致（rubberbanding / 闪烁）

#### 子类 a：客户端预测过激

**症状**：玩家移动看起来正常，但**反复瞬间被拉回**——典型 rubberbanding。

**排查步骤**：

1. 是不是延迟很高？用 `c_setlatency(0.5)` 模拟高延迟。
2. 客户端 mod 是不是改了 locomotor 速度？
3. 服务端的实际位置和客户端预测位置差距多大？
   ```lua
   -- 服务端
   print(ThePlayer:GetPosition())
   -- 客户端
   print(ThePlayer:GetPosition())
   -- 比对
   ```

**修复**：

- 不要 mod 自己改 locomotor 内部状态
- 速度变化通过 `locomotor:SetExternalSpeedMultiplier` 等公开 API
- 不直接 `Transform:SetPosition` 客户端

#### 子类 b：服务端拒绝预测后没回退

**症状**：mod 自定义 Action — 客户端预测进了 dolongaction state — 服务端拒绝了 — 客户端 SG **永远卡在 dolongaction**。

**排查步骤**：

1. 自定义 SG state 有没有 `EventHandler("failedaction", fn)`？
2. 自定义 state 的 events 是否完整？

**修复**：

```lua
State{
    name = "myaction",
    -- ...
    events = {
        EventHandler("animqueueover", function(inst)
            inst.sg:GoToState("idle")
        end),
        EventHandler("failedaction", function(inst)  -- ★ 必须！
            inst.sg:GoToState("idle")
        end),
    },
},
```

#### 子类 c：双方逻辑不一致

**症状**：客户端看到的状态和服务端不一致——比如**血量显示 100 但实际是 80**。

**排查步骤**：

1. 服务端真实值：`c_select().components.health.currenthealth`
2. 客户端 NetVar 值：`c_select().player_classified.currenthealth:value()`
3. 客户端 replica 值：`c_select().replica.health:Current()`
4. 比对——找差异点

**常见原因 + 修复**：

- ❌ 服务端 mod 直接改 `inst.components.health.currenthealth = X`，没走 `DoDelta`
- ✅ 永远用 `inst.components.health:DoDelta(amount)` —— 它会自动同步 classified
- ❌ mod 在客户端用 set_local 改了 NetVar
- ✅ 永远不在客户端改 NetVar

---

### 9.7.5 进阶：调试工具与控制台命令

#### 第一步：核心调试命令

| 命令 | 用途 |
| --- | --- |
| `c_select()` | 选中鼠标下方的 entity |
| `c_listallplayers()` | 列出所有玩家 |
| `c_setlatency(seconds)` | 模拟网络延迟 |
| `c_dump()` | 把当前选中 entity 的所有字段打印 |
| `print(json.encode(t))` | 把 table 打成 JSON 格式（便于阅读复杂 data） |

#### 第二步：双端调试

**主机端控制台 + 客户端控制台同时开**——按 `~` 打开。**主机端能 c_spawn / c_give 等命令；客户端只能查询**。

**典型联调**：

```lua
-- 主机端
local ent = c_spawn("pigman")
print(ent.GUID)  -- 比如 12345

-- 客户端
local ent = Ents[12345]  -- 通过 GUID 拿客户端代理
print(ent.replica.health:Current())
```

#### 第三步：日志和断点

**Klei 没有完整 IDE 调试器**——主要靠 `print`。

**模式**：

```lua
local function trace(label, ...)
    print(string.format("[mymod %s] %s", label, table.concat({...}, " ")))
end

-- 业务里
trace("server", "received RPC", x, y)
trace("client", "send RPC", x, y)
```

服务端日志在控制台或 `client_log.txt`；客户端日志在客户端控制台或 `client_log.txt`。

#### 第四步：用 net_event 做"心跳调试"

```lua
-- modmain
local DEBUG_HEARTBEAT = net_event(GLOBAL.TheWorld.GUID, "mymod.debug.heartbeat")

-- 服务端某关键路径
TheWorld.mymod_debug:push()

-- 客户端
TheWorld:ListenForEvent("mymod.debug.heartbeat", function()
    print("[client] heartbeat received")
end)
```

**用途**：测网络通路是否畅通——如果心跳收不到，说明上游有问题。

---

### 9.7.6 进阶：性能问题——网络带宽爆炸

#### 第一步：症状

- 服务器 ping 持续上涨
- 客户端 FPS 下降但 CPU 占用低（卡在网络上）
- 服务器进程 RAM 持续增长

#### 第二步：原因 a —— NetVar 高频 set

**反例**：

```lua
function MyComp:OnUpdate(dt)
    self.inst.posnetvar:set(self.inst:GetPosition())  -- 每帧 set
end
```

**问题**：30 Hz × 12 字节 = 360 B/s，单实体——但**100 个实体就是 36 KB/s**——很快爆炸。

**修复**：仅在变化时 set。

```lua
function MyComp:OnUpdate(dt)
    local pos = self.inst:GetPosition()
    if not self.last_pos or distsq(pos, self.last_pos) > 1 then
        self.inst.posnetvar:set(pos)
        self.last_pos = pos
    end
end
```

#### 第三步：原因 b —— update 组件过多

**反例**：mod 给 100 种 prefab 都加 OnUpdate。

**修复**：

```lua
-- 集中到 manager 组件
AddPrefabPostInit("world", function(inst)
    if inst.ismastersim then
        inst:AddComponent("mymod_manager")
    end
end)

-- mymod_manager 内部维护 prefab 列表，OnUpdate 一次跑完所有
```

#### 第四步：原因 c —— 超大 net_string

**反例**：

```lua
inst.savedata = net_string(inst.GUID, "savedata", "savedatadirty")
inst.savedata:set(json.encode(huge_table))  -- 序列化 100KB 数据
```

**问题**：每次 set 都传 100KB。

**修复**：

- 只同步**变化部分**
- 用专门的 RPC 一次传过去，不要走 NetVar
- 大数据走持久化（OnSave/OnLoad），不走网络

#### 第五步：监控工具

```
TheNet:GetServerLatency()     -- 服务器延迟
TheSim:GetMemoryStats()       -- 内存统计
print(NUM_NETWORK_VARIABLES)  -- 当前 NetVar 总数
```

---

### 9.7.7 老手进阶：跨 shard / 持久化的常见坑

#### 第一步：跨 shard 同步

**坑**：mod 在地表 shard 推 `TheWorld:PushEvent("foo")` —— 期望洞穴 shard 也响应——**根本不会**。

**修复**：用 `SendModRPCToShard`（9.2.5）+ shard 间持久化。

#### 第二步：持久化 vs 网络

**坑**：mod 用 NetVar 存"长期数据"——**重启游戏后丢失**——NetVar 是运行时状态，不写存档。

**修复**：

- 长期数据走组件的 `OnSave / OnLoad`
- NetVar 只用于运行时同步
- 服务端从 `OnLoad` 读数据后用 NetVar 同步给客户端

#### 第三步：玩家断线

**坑**：玩家断线时——entity 不立刻销毁，5 分钟后才彻底清理。**断线期间所有 NetVar / classified 仍存在**——但**不能 set 新值**——会被静默忽略。

**修复**：

- 监听 `ms_playerleft` —— 主动清理 mod 数据
- 不依赖断线时的 NetVar 同步

#### 第四步：seamless player swap

**坑**：玩家变身（如毛兔变形）——player entity **没换**——但**某些状态切换异常**——player_classified 的 `finishseamlessplayerswap` 事件需要重新初始化部分状态。

**修复**：监听 `finishseamlessplayerswap` 事件，重新设置 mod 自定义状态。

---

### 9.7.8 老手进阶：六个最难定位的隐蔽陷阱

#### 陷阱 1：客户端 NetVar 拿到的是"旧值"

**症状**：客户端某操作后立刻读 NetVar——值还是旧的——稍等一会再读就对。  
**原因**：NetVar 同步是**异步**——服务端 set 后，客户端**下一帧**才能读到。  
**修复**：通过 dirty 事件触发后续逻辑，不要立刻读。

#### 陷阱 2：dedicated 上的某些初始化逻辑客户端没跑

**症状**：mod 在 master_postinit 里设了某状态——dedicated 上正常——但客户端联机时该状态不存在。  
**原因**：master_postinit 只在服务端跑——客户端的 inst 上没有那些数据。  
**修复**：用 NetVar 同步给客户端，或客户端用 `_postinit` 单独设。

#### 陷阱 3：spawn 实体后立刻 set NetVar

**症状**：

```lua
local ent = SpawnPrefab("xxx")
ent.myvar:set(123)  -- 客户端可能拿不到
```

**原因**：实体刚 spawn —— 客户端代理可能还没创建——set 早于 replicate。  
**修复**：用 `inst:DoStaticTaskInTime(0, ...)` 延迟到下一帧：

```lua
local ent = SpawnPrefab("xxx")
ent:DoStaticTaskInTime(0, function(inst)
    inst.myvar:set(123)
end)
```

#### 陷阱 4：NetVar 名字相同的两个 mod 互相干扰

**症状**：mod A 和 mod B 都有 `inst.healthbuff = net_byte(inst.GUID, "healthbuff", "buffdirty")` —— 同一个 entity 上 → 两个 mod 互相覆盖。  
**原因**：NetVar 名字是引擎用的索引——重名会引发问题。  
**修复**：永远 `<modname>.<field>`：`"mymodA.healthbuff"`。

#### 陷阱 5：客户端 RPC handler 没注册导致服务端到客户端通信失败

**症状**：mod 用了 `SendModRPCToClient` —— 服务端调了——客户端不响应。  
**原因**：客户端没注册 `AddClientModRPCHandler`。  
**修复**：客户端 ModRPC 注册必须用 `AddClientModRPCHandler`，不是 `AddModRPCHandler`。

#### 陷阱 6：classified 在 entity 销毁时残留引用

**症状**：mod 自定义 entity 销毁后——某些客户端代码仍在引用旧的 classified——访问到已 invalid 的 NetVar。  
**原因**：classified 的清理是异步——entity 标记为 invalid 但 lua 引用仍存在。  
**修复**：访问前 IsValid 判：

```lua
if inst:IsValid() and inst.myclassified and inst.myclassified:IsValid() then
    -- 访问
end
```

#### 设计经验三条

**经验 ①：永远在 dedicated + 远程客户端复现 bug**

主机端测试**看不到 90% 的网络 bug**——必须**专门**用 dedicated server + 远程客户端**真实场景**复现。

**经验 ②：写 mod 时按"4 部署模式"画表**

每个 mod 的关键代码段——**先想清楚** 4 种部署（单机/主机/客户端/dedicated）都正常工作——再下笔。

**经验 ③：日志 + 心跳 + 比对——三件套**

调试网络 bug 的标准流程：
1. **加日志**：服务端 + 客户端关键路径都加 print
2. **心跳测试**：用 net_event 测网络通路
3. **比对状态**：服务端真实值 vs 客户端 NetVar 值 vs 客户端 replica 值——找差异点

---

### 9.7.9 小结

**网络 bug 排查一句话总结**：**症状分 3 类（数据 / 行为 / 状态）→ 按决策树定位 → 用日志+心跳+比对验证 → 修复**。

**速查表**

| 症状 | 第一直觉检查 |
| --- | --- |
| 客户端读不到 NetVar | 检查 SetPristine 之前创建、AddNetwork 调了 |
| classified 看不到 | 检查 CLASSIFIED 标签、SetParent |
| RPC 没触发 | 检查 modmain 顶层注册、namespace 拼写 |
| 双 SG 卡住 | 检查 wilson + wilson_client 都注册 |
| 客户端动作没反应 | 检查 collector 用 replica 不用 components |
| rubberbanding | 检查 locomotor 修改 + 客户端预测距离 |
| 状态不一致 | 主机用 DoDelta 不直接改字段 |
| 性能爆炸 | NetVar 仅变化时 set + manager 集中 update |

**6 个最隐蔽陷阱**

1. NetVar 同步异步——不要立刻读
2. master_postinit 客户端没跑——用 NetVar 同步
3. spawn 后立刻 set——延一帧
4. NetVar 重名——加 mod 前缀
5. AddClientModRPCHandler ≠ AddModRPCHandler——分清
6. classified 销毁残留——IsValid 判

**3 条核心经验**

- ① **dedicated 测试不能省**
- ② **4 部署模式提前画表**
- ③ **日志 + 心跳 + 比对三件套**

---

### 整章 9.x 总结

第 9 章 **网络同步机制** 到此完整收尾——

- **9.1** NetVar：服务端→客户端的状态广播 + 14 种类型 + dirty 事件
- **9.2** RPC：客户端→服务端的远程调用 + 信任边界 + 限流
- **9.3** Classified：私有数据载体 + CLASSIFIED 标签 + SetParent
- **9.4** ismastersim 系列判断：4 部署模式 × 7 判断函数真值表
- **9.5** 客户端预测：双 SG + PreviewBufferedAction + snap 校正
- **9.6** Inventory / Container 同步：SetClassifiedTarget + 乐观锁 + opener 系统
- **9.7** 网络 bug 诊断手册：3 大类症状 + 决策树 + 6 隐蔽陷阱

读完整章——**你将拥有写出"不仅单机能跑、联机也稳定"的 mod 的能力**。和第 7 章 Action 系统、第 8 章 事件系统配合——**饥荒联机版 mod 开发的"三大支柱"**全部打通。