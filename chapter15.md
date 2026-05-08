# 第15章 声音系统

## 15.1 SoundEmitter 组件与音效事件

### 本节导读

> **一句话定位**：`SoundEmitter` 是饥荒中**所有 entity 发声的统一接口**——给一个 entity `AddSoundEmitter()` 后，你就能让它"说话、走路有脚步、烧火有噼啪、被砍有惨叫"——本质是 **entity 与 FMOD 音频引擎之间的桥梁**。

#### 一段宏观先讲清楚：饥荒声音从哪来？

打开任何一个饥荒玩家的录像，你能听到的所有"声音"大致分 5 类：

| 类别 | 代表 | 谁触发 | 谁播放 |
|---|---|---|---|
| **角色 / 怪物动作音** | 玩家攻击声、蜘蛛跳声 | StateGraph / combat | **entity.SoundEmitter** |
| **物品触发音** | 火堆噼啪、留声机音乐 | component / prefab | **entity.SoundEmitter** |
| **环境音** | 草原虫鸣、洞穴回声 | World 全局 | **TheWorld.SoundEmitter / ambientsound 组件** |
| **音乐 / BGM** | 战斗音乐、夜晚音乐 | dynamicmusic | **player.SoundEmitter（特殊通道）** |
| **UI 音** | 点击、菜单 | TheFrontEnd | TheFrontEnd:GetSound() |

**15.1 节聚焦的是第 1、2 类——entity 的 SoundEmitter API**——这是 mod 开发者最常用的音效入口。

#### SoundEmitter 在引擎里的位置

```
┌──────────────────────────────────────────────────────────────────┐
│  ▎ FMOD Studio 项目  ▎ (开发期)                                    │
│   bank: dontstarve.fev / dontstarve.fsb                          │
│   events: "dontstarve/wilson/attack_weapon"                      │
│            "dontstarve/common/fireBurstSmall"                    │
│            ...                                                    │
└──────────────────────────────────────────────────────────────────┘
                             │
                             │ 游戏启动时加载
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  ▎ C++ FMOD runtime  ▎ (运行时)                                    │
│   FMODSystem 全局单例                                              │
│   Channel / EventInstance 池                                      │
└──────────────────────────────────────────────────────────────────┘
                             ▲
                             │ Lua 调 entity:AddSoundEmitter()
                             │
┌─────────────────────────────────────────────────────────────────┐
│  ▎ Lua entity layer  ▎                                            │
│   entity 实体（玩家 / 火堆 / 蜘蛛 / 树）                            │
│      ├── inst.entity:AddSoundEmitter()  ← C++ binding             │
│      └── inst.SoundEmitter                                         │
│            ├── :PlaySound(event, [name], [volume], [is_predicted])│
│            ├── :KillSound(name)                                    │
│            ├── :SetParameter(name, param, value)                  │
│            ├── :SetVolume(name, volume)                            │
│            ├── :PlayingSound(name) → bool                          │
│            ├── :PlaySoundWithParams(event, params, [name], …)     │
│            ├── :OverrideVolumeMultiplier(volume)                   │
│            └── :KillAllSounds()                                    │
└──────────────────────────────────────────────────────────────────┘
                             │
                             │ entity 死亡时自动 KillAllSounds
                             ▼
                          C++ FMOD 释放资源
```

#### 15.1 节回答的 4 个核心问题

```
Q1: ──── entity 怎么发声音？
        ↓ 答：AddSoundEmitter + PlaySound

Q2: ──── 怎么停止 / 控制正在播的音效？
        ↓ 答：带 name 播 → KillSound / SetParameter / SetVolume

Q3: ──── 音效事件路径 "dontstarve/xxx/yyy" 怎么来的？
        ↓ 答：FMOD 工程里的 event path，按命名空间组织

Q4: ──── 不同状态切换音效（白天 / 夜晚 / 不同火大小）怎么写？
        ↓ 答：FMOD parameter + SetParameter 动态调整
```

#### 你将看到 4 份核心源码

| 文件 | 角色 | 用途 |
|---|---|---|
| `scripts/components/firefx.lua` | 175 行 | **完整 SoundEmitter 案例** —— 持续 loop + 参数化 + 多帧切换 |
| `scripts/components/ambientsound.lua` | 400+ 行 | 环境音系统（World.SoundEmitter）|
| `scripts/components/combat.lua` | combat.lua:670-685 | 战斗 hurtsound / hitsound 触发 |
| `scripts/stategraphs/commonstates.lua` | 多处 | StateGraph TimeEvent 内调 PlaySound |

#### 本节学习路径

```
15.1.1 新手  ──────  SoundEmitter 是什么 —— 3 行入门
15.1.2 新手  ──────  PlaySound vs KillSound —— 4 大常用模式
15.1.3 新手  ──────  音效事件路径 —— 命名空间速查
                       ↓
15.1.4 进阶  ──────  PlaySound 4 参数详解
15.1.5 进阶  ──────  FMOD 参数化 —— PlaySoundWithParams + SetParameter
15.1.6 进阶  ──────  音效生命周期管理（OnEntitySleep / OnEntityWake / OnRemove）
                       ↓
15.1.7 老手  ──────  FireFX 完整案例剖析
15.1.8 老手  ──────  6 大常见陷阱
15.1.9 小结  ──────  速查表 + 4 行起步代码 + 15.2 预告
```

**给三类读者的承诺**：
- **新手**：你将得到一个**3 行 PlaySound 模板**（15.1.2），并理解为什么"玩家受伤时火堆的噼啪声不会消失"。
- **进阶**：你将完整掌握 PlaySound 4 个参数 + PlaySoundWithParams 参数化模型——能写出"白天 / 夜晚不同强度的火堆音效"。
- **老手**：你将看清 SoundEmitter 与 entity lifecycle、网络预测、FMOD parameter 的全部联动——写出可发布质量的音效系统。

---

### 15.1.1（新手）SoundEmitter 是什么——3 行入门

#### 给 entity 加 SoundEmitter 的最小代码

```lua
local inst = CreateEntity()
inst.entity:AddTransform()
inst.entity:AddSoundEmitter()       -- ① 加上 SoundEmitter 组件
-- 后面任何时候都能：
inst.SoundEmitter:PlaySound("dontstarve/common/deathpoof")  -- ② 立即播
```

**仅此而已**。3 行——你的 entity 就具备了发声能力。

#### 一段最直接的"使用方代码"

打开 `scripts/components/combat.lua` 第 670 ~ 685 行——任何 entity 受伤时：

```670:685:scripts/components/combat.lua
            self.inst.SoundEmitter:PlaySound(hitsound)
        elseif redirect_combat ~= nil and redirect_combat.hurtsound ~= nil then
            self.inst.SoundEmitter:PlaySound(redirect_combat.hurtsound)
        elseif self.hurtsound ~= nil then
            self.inst.SoundEmitter:PlaySound(self.hurtsound)
```

**核心模式**：
- 任何 entity 只要 `AddSoundEmitter()` 过，就能在任何时候 `inst.SoundEmitter:PlaySound("...")`
- 调用方"扔出去就走"——FMOD 引擎自管音效播放、3D 衰减、生命周期

#### SoundEmitter 的 3 大核心特性

##### 特性 1：3D 空间音效

SoundEmitter 自动跟随 entity 的位置——玩家**在 entity 远处时，声音会**：
- 变小（距离衰减）
- 偏左 / 偏右（立体声 pan）
- 高频被遮挡（远处发闷）

——**全部由 FMOD 自动处理**——你只调 PlaySound，剩下的 FMOD 帮你算。

##### 特性 2：多音效并发

一个 SoundEmitter 可以**同时播放多个音效**——只要它们用**不同的 name**：

```lua
inst.SoundEmitter:PlaySound("dontstarve/AMB/forest_amb_LP", "ambient")
inst.SoundEmitter:PlaySound("dontstarve/creatures/spider/walk", "walk")
inst.SoundEmitter:PlaySound("dontstarve/wilson/attack_weapon")  -- 一次性，无 name
```

**3 个音效同时播**——互不干扰。

##### 特性 3：自动清理

当 entity 销毁时，**所有正在播的音效自动停止**——无需 mod 开发者手动 KillAllSounds。

```lua
-- 你的代码
local fire = SpawnPrefab("campfire")
fire.SoundEmitter:PlaySound("dontstarve/common/campfire", "fire_loop")

-- 5 分钟后...
fire:Remove()
-- ↑ 此时 fire_loop 自动停止，无需手动 KillSound
```

#### entity 不带 SoundEmitter 时的失败模式

```lua
local inst = CreateEntity()
inst.entity:AddTransform()
-- 忘了 AddSoundEmitter
inst.SoundEmitter:PlaySound("...")  -- ✗ attempt to index nil
```

**记忆诀**：任何用 PlaySound 的 entity 必须先 `AddSoundEmitter()`。

#### "永远先检查 SoundEmitter 是否存在"

工业级代码：
```lua
if inst.SoundEmitter then
    inst.SoundEmitter:PlaySound("...")
end
```

但是大部分 vanilla 代码不做检查——因为约定是**写 PlaySound 的 prefab 一定加了 AddSoundEmitter**。这是 codebase 内的隐式契约。

---

### 15.1.2（新手）PlaySound vs KillSound——4 大常用模式

#### 4 大模式总览

```
┌────────────────────────────────────────────────────────────────┐
│  ▎ 模式 1：One-shot（最常见，60%）▎                              │
│   PlaySound("dontstarve/common/deathpoof")                    │
│   ↑ 不带 name，FMOD 自管，无法控制                              │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  ▎ 模式 2：Named Loop（第二常见，30%）▎                         │
│   PlaySound("dontstarve/common/campfire", "fire")             │
│   KillSound("fire")                                           │
│   ↑ 带 name，可以后续 Kill / 改音量 / 改参数                   │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  ▎ 模式 3：Loop with Parameter（参数化）▎                       │
│   PlaySound("dontstarve/AMB/fire", "fire")                    │
│   SetParameter("fire", "intensity", 0.7)                      │
│   ↑ 配合 FMOD parameter，动态调整大小 / 颜色                   │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  ▎ 模式 4：Predicted（客户端预测）▎                              │
│   PlaySound(event, name, volume, true)                        │
│   ↑ 第 4 参数 true，支持客户端预测立刻播                        │
└────────────────────────────────────────────────────────────────┘
```

#### 模式 1：One-shot（一次性音效）

**用途**：短暂、单次、不需要后续控制的音效——攻击、爆炸、UI 反馈等。

```lua
-- 玩家攻击
inst.SoundEmitter:PlaySound("dontstarve/wilson/attack_weapon")

-- 物品摔碎
inst.SoundEmitter:PlaySound("dontstarve/wilson/use_axe_chop")

-- FX 系统也常用一次性
inst.SoundEmitter:PlaySound("dontstarve/common/deathpoof")
```

**注意**：one-shot 一旦开始播放，**没有 handle 可以停**——只能等它自然结束。
- 优点：无需管理，自动结束
- 缺点：不能取消（如果你需要取消，必须用模式 2）

#### 模式 2：Named Loop（带 handle 的可控音效）

**用途**：持续音、需要后续控制（停止 / 改音量 / 改参数）的场景。

最经典案例 —— `scripts/components/firefx.lua` 第 117 ~ 134 行：

```117:134:scripts/components/firefx.lua
        if self.playingsound ~= params.sound then
            if self.playingsound ~= nil then
                self.inst.SoundEmitter:KillSound("fire")
            end
            self.playingsound = params.sound
            self.playingsoundintensity = nil
            if params.sound ~= nil and not self.inst:IsAsleep() then
                self.inst.SoundEmitter:PlaySound(params.sound, "fire")
            end
        end

        if self.playingsoundintensity ~= params.soundintensity and params.sound ~= nil then
            self.playingsoundintensity = params.soundintensity
            if params.soundintensity ~= nil and self.inst.SoundEmitter:PlayingSound("fire") then
                self.inst.SoundEmitter:SetParameter("fire", "intensity", params.soundintensity)
            end
        end
```

**关键逻辑**：
1. `PlaySound(params.sound, "fire")` —— 用 name `"fire"` 标记这个 loop
2. `KillSound("fire")` —— 切换大小时停掉旧的 loop
3. `PlayingSound("fire")` —— 检查 `"fire"` 是否还在播
4. `SetParameter("fire", "intensity", value)` —— 动态调音

**模板**：
```lua
-- 启动持续音
inst.SoundEmitter:PlaySound("dontstarve/common/campfire", "fire_loop")

-- 检查是否在播
if inst.SoundEmitter:PlayingSound("fire_loop") then
    print("还在烧")
end

-- 改音量
inst.SoundEmitter:SetVolume("fire_loop", 0.5)

-- 停止
inst.SoundEmitter:KillSound("fire_loop")
```

#### 模式 3：Loop with Parameter（FMOD 参数化）

FireFX 在切换"白天/夜晚"音效时用：

```59:62:scripts/components/firefx.lua
    if self.usedayparamforsound and self.isday ~= TheWorld.state.isday then
        self.isday = TheWorld.state.isday
        self.inst.SoundEmitter:SetParameter("fire", "daytime", self.isday and 1 or 2)
    end
```

**3 步**：
1. PlaySound 时给一个 name（例如 `"fire"`）
2. 后续随时 `SetParameter(name, parameter_name, value)` 改 FMOD parameter
3. FMOD 内部根据 parameter 切换音效层（白天的火 vs 夜晚的火）

**给 mod 启示**：
- FMOD parameter 是 vanilla 火堆音效的一个特性
- mod 自定义 FMOD bank 时也可以做这种"参数化音效"——一个 event 在不同 parameter 下播不同分支

#### 模式 4：Predicted（客户端预测）

走路 footstep 在客户端**立刻预测播放**，而不等服务器同步：

```19:20:scripts/stategraphs/commonstates.lua
        inst.SoundEmitter:PlaySound("dontstarve/movement/run_dirt")
        --inst.SoundEmitter:PlaySound("dontstarve/movement/walk_dirt")
```

更复杂的客户端预测 footstep，看 `SGwilson_client.lua:1440`：

```lua
inst.SoundEmitter:PlaySoundWithParams(
    "dontstarve/characters/walter/woby/big/footstep", 
    { intensity = 1 }, 
    nil,    -- name
    true    -- ← is_predicted
)
```

**含义**：
- 客户端**立即播放**这个声音（不等服务器命令）
- 服务器若也播了同样声音，**FMOD 会去重**（或两端各播一次）

**注意**：is_predicted 仅用于**客户端预测的 SG state**——大部分 mod 不需要碰这个参数。

#### 4 模式选择决策树

```
你的音效要做什么？
   │
   ├─ 一次性（攻击 / 爆炸 / 死亡）
   │     │
   │     └→ 模式 1：PlaySound(event)
   │
   ├─ 持续 loop（火 / 风 / 机器运行）
   │     │
   │     └→ 模式 2：PlaySound(event, "name") + KillSound("name")
   │
   ├─ 持续 loop + 动态变化（火大小 / 季节）
   │     │
   │     └→ 模式 3：模式 2 + SetParameter(name, param, value)
   │
   └─ 玩家走路 / 攻击预测（仅 SG client state）
         │
         └→ 模式 4：PlaySound(..., is_predicted=true)
```

---

### 15.1.3（新手）音效事件路径——命名空间速查

#### 路径长什么样？

打开任意一份 vanilla prefab，你会看到大量这种字符串：

```
dontstarve/wilson/attack_weapon
dontstarve/common/fireBurstSmall
dontstarve/AMB/forest_winter
turnoftides/common/together/water/splash/bird
rifts/ambience/rift_tile_amb
meta4/shadow_snap/snap
```

——这些都是 **FMOD event path**——它们是 FMOD Studio 工程里的层级化路径，类似文件夹。

#### 8 大命名空间

通过统计 vanilla 中所有路径前缀，整理出 8 个主要命名空间：

| 命名空间 | 内容范围 | 典型路径 |
|---|---|---|
| `dontstarve/` | 基础游戏（单机版起家）| `dontstarve/wilson/...`、`dontstarve/AMB/...` |
| `dontstarve_DLC001/` | RoG 资料片 | `dontstarve_DLC001/spring/...` |
| `turnoftides/` | 海洋 / 月岛资料片 | `turnoftides/together_amb/ocean/...` |
| `hookline_2/` | 钓鱼资料片 | `hookline_2/amb/sea_shore` |
| `meta4/` | 月族阴影更新（Shattered Veil）| `meta4/shadow_snap/snap` |
| `rifts/` | 月族裂缝资料片 | `rifts/ambience/rift_tile_amb` |
| `monkeyisland/` | 猴岛资料片 | `monkeyisland/amb/dock_ambience` |
| `webber2/`、`waterlogged2/` 等版本号 | 各小型更新 | `webber2/common/spiderden/in` |

#### 路径常见结构

```
<命名空间>/<类别>/<子类别>/<具体动作>

例：
dontstarve / wilson    / attack_weapon
dontstarve / common    / fireBurstSmall
dontstarve / AMB       / forest_winter
dontstarve / creatures / spider / walk
dontstarve / movement  / run_dirt
turnoftides/ common    / together / water / splash / bird
```

**5 个常见类别**：

| 类别 | 含义 | 示例 |
|---|---|---|
| `wilson/` | 玩家相关动作 | `dontstarve/wilson/hit`, `dontstarve/wilson/attack_weapon` |
| `creatures/` | 怪物 | `dontstarve/creatures/spider/walk` |
| `common/` | 通用音效 | `dontstarve/common/deathpoof` |
| `AMB/` | Ambient（环境音）| `dontstarve/AMB/forest` |
| `movement/` | 脚步音 | `dontstarve/movement/run_dirt` |

#### 80+ 高频音效速查（mod 开发常用）

##### 玩家动作

| 路径 | 用途 |
|---|---|
| `dontstarve/wilson/attack_weapon` | 武器攻击 |
| `dontstarve/wilson/hit` | 受伤 |
| `dontstarve/wilson/use_axe_chop` | 砍木头 |
| `dontstarve/wilson/use_pick_rock` | 挖矿 |
| `dontstarve/wilson/dig` | 挖坑 |
| `dontstarve/wilson/fireball_explo` | 紫火法杖 |
| `dontstarve/wilson/use_gemstaff` | 紫宝石法杖通用 |

##### 通用 / 环境

| 路径 | 用途 |
|---|---|
| `dontstarve/common/deathpoof` | 死亡烟雾（FX 默认音）|
| `dontstarve/common/destroy_smoke` | 物体被破坏 |
| `dontstarve/common/destroy_wood` | 木质破坏 |
| `dontstarve/common/destroy_metal` | 金属破坏 |
| `dontstarve/common/destroy_rock` | 石质破坏 |
| `dontstarve/common/freezecreature` | 冰冻 |
| `dontstarve/common/freezethaw` | 解冻 |
| `dontstarve/common/fireBurstSmall` | 小火苗 |
| `dontstarve/common/fireBurstLarge` | 大火苗 |
| `dontstarve/common/fireOut` | 火灭 |
| `dontstarve/common/campfire` | 篝火 loop |

##### 移动 / 脚步

| 路径 | 用途 |
|---|---|
| `dontstarve/movement/run_dirt` | 跑（土）|
| `dontstarve/movement/run_grass` | 跑（草）|
| `dontstarve/movement/run_marsh` | 跑（沼泽）|
| `dontstarve/movement/run_ice` | 跑（冰）|
| `dontstarve/movement/run_mud` | 跑（泥）|
| `dontstarve/movement/run_web` | 跑（蛛网）|

##### 月岛 / 海洋（turnoftides）

| 路径 | 用途 |
|---|---|
| `turnoftides/common/together/water/splash/bird` | 鸟入水 |
| `turnoftides/common/together/water/splash/small` | 小水花 |
| `turnoftides/together_amb/ocean/shallow` | 浅海环境 |
| `turnoftides/together_amb/ocean/deep` | 深海环境 |
| `turnoftides/sanity/lunacy_LP` | 月光精神 loop |

##### 怪物典型音

| 路径 | 用途 |
|---|---|
| `dontstarve/creatures/spider/walk` | 蜘蛛走 |
| `dontstarve/creatures/spider/attack` | 蜘蛛攻击 |
| `dontstarve/creatures/spider/die` | 蜘蛛死 |
| `dontstarve/creatures/together/toad_stool/spore_cloud_LP` | 蛤蟆王毒雾 loop |
| `dontstarve/creatures/together/dragonfly/death` | 火龙死 |

##### 精神 / 状态

| 路径 | 用途 |
|---|---|
| `dontstarve/sanity/sanity` | 精神低音效 |
| `dontstarve/sanity/gonecrazy_LP` | 完全失智 loop |
| `dontstarve/sanity/gonecrazy_in` | 进入失智 |
| `dontstarve/sanity/gonecrazy_out` | 离开失智 |

#### "我怎么知道有哪些音效路径可用？"

3 种方法：

##### 方法 A：grep vanilla 源码

```bash
rg "PlaySound\(\"" scripts/ -o
```
能列出所有用过的音效路径——**这是最可靠的"音效字典"**。

##### 方法 B：FMOD bank 文件解包

将 `dontstarve.fev` 用 FMOD Studio 工具打开——能看到完整 event 树。但需要安装 FMOD Studio，且 dontstarve.fev 受版权保护。

##### 方法 C：mod 创意工坊参考

打开任何一个流行 mod，搜索 `PlaySound`——**他们用的路径基本都是 vanilla 中已有的**——可以复制粘贴。

#### mod 自定义音效路径

mod 自己加 FMOD bank 后，可以注册自己的命名空间：

```
mymod/character/willowex/howl
mymod/character/willowex/laugh_lp
```

如何加 FMOD bank → 看 15.5 节专门讲。

---

### 15.1.4（进阶）PlaySound 4 参数详解

#### 完整签名

```lua
inst.SoundEmitter:PlaySound(event, name, volume, is_predicted)
```

| 参数 | 类型 | 必填 | 默认 | 作用 |
|---|---|---|---|---|
| `event` | string | ✓ | - | FMOD event path |
| `name` | string | ✗ | nil | 音效 handle 名（用于后续控制）|
| `volume` | number | ✗ | 1.0 | 初始音量倍率（0 ~ 1）|
| `is_predicted` | bool | ✗ | false | 是否是客户端预测音效 |

#### 参数 1：event（事件路径）

详见 15.1.3。可以是任意 FMOD bank 中的 event path。

**注意路径错误的处理**：
- vanilla 项目对 invalid path 通常只 log warning（不崩溃）
- 但 path 写错 → 没声音播 → 玩家以为 mod 坏了

**调试技巧**：在控制台开启 sound debug 显示：
```lua
TheSim:SetDebugCameraTarget(player)  -- 配合一起用
-- 或者监听 fx_spawned 等事件，打印调用链
```

#### 参数 2：name（handle）

**有 name 的优势**：
- 可调 `KillSound(name)` 停止
- 可调 `SetParameter(name, ...)` 改参数
- 可调 `SetVolume(name, ...)` 改音量
- 可调 `PlayingSound(name)` 检查
- **同 name 后播会覆盖前播**（自动停旧的）

**无 name 的优势**：
- 简单，无需管理
- 多个相同 event 可以并行（同时播 5 个攻击声）

**实战取舍**：

| 场景 | 用 name? |
|---|---|
| 短一次性音（attack / hit / break）| ✗ 无 name |
| 持续音（loop）| ✓ 有 name |
| 短音但需要去重（避免连击重复播放）| ✓ 有 name（同 name 自动去重）|

#### 参数 3：volume（初始音量）

```lua
inst.SoundEmitter:PlaySound("dontstarve/common/campfire", "fire", 0.5)  -- 50% 音量
```

**注意**：
- volume 仅设置**初始音量**，后续要改用 `SetVolume(name, vol)` 或 `OverrideVolumeMultiplier(vol)`
- 范围：0 ~ 1（理论上可超过 1，但会爆音）

#### 参数 4：is_predicted（客户端预测）

```10226:10226:scripts/stategraphs/SGwilson.lua
				inst.SoundEmitter:SetVolume("book_layer_sound", .5)
```

```786:786:scripts/stategraphs/commonstates.lua
		inst.SoundEmitter:PlaySound(land_sound, nil, nil, true)
```

第 4 参数 `true` 表示**客户端预测播**——意味着：
- 客户端**立即**播放这个音效
- 不等服务器同步
- 服务器同步过来时，FMOD 会**判断是否重复**——如果客户端已经在播相同 name 的，就不再播

**只在 SGclient state 内使用**——常规 prefab 不要带这个参数。

#### 4 参数组合速查表

```lua
-- 最常见：单参（一次性音）
inst.SoundEmitter:PlaySound("event")

-- 双参（命名 loop）
inst.SoundEmitter:PlaySound("event", "loop_name")

-- 三参（设初始音量的 loop）
inst.SoundEmitter:PlaySound("event", "loop_name", 0.5)

-- 四参（客户端预测，仅 SG client）
inst.SoundEmitter:PlaySound("event", nil, nil, true)
inst.SoundEmitter:PlaySound("event", "name", 0.7, true)
```

#### KillSound / SetParameter / SetVolume 详解

##### KillSound(name)

```lua
inst.SoundEmitter:KillSound("fire")  -- 立即停止 name="fire" 的音
```

**注意**：
- 立即停止，无 fade out
- 如果想要 fade out，要在 FMOD bank 中给 event 配置 fade
- 或者用 `SetVolume(name, 0)` + `DoTaskInTime(0.5, function() KillSound(name) end)`

##### SetParameter(name, parameter, value)

```lua
inst.SoundEmitter:SetParameter("fire", "intensity", 0.7)
```

**前提**：FMOD event 在工程内**确实定义了**这个 parameter——否则无效（不会报错）。

vanilla 中常用 parameter：
- `intensity`（火大小、音乐强度）
- `daytime`（白天/夜晚）
- `season`（季节）
- `health`（boss 血量段）

##### SetVolume(name, volume)

```lua
inst.SoundEmitter:SetVolume("dizzyloop", 0.3)
```

**用法**：动态调整正在播的某个 loop 的音量。例如玩家被击晕时降低环境音音量。

##### PlayingSound(name) → bool

```lua
if inst.SoundEmitter:PlayingSound("fire") then
    -- 还在播，可以做后续操作
end
```

##### OverrideVolumeMultiplier(volume)

```lua
inst.SoundEmitter:OverrideVolumeMultiplier(0.5)
```

**作用**：给**这个 entity 上所有正在播的音效**乘一个倍率——常用于"距离衰减外的额外音量调整"。

例：玩家戴耳塞帽 → entity OverrideVolumeMultiplier(0.5) → 该 entity 所有音都减半。

#### KillAllSounds() —— 紧急停所有

```lua
inst.SoundEmitter:KillAllSounds()
```

**用途**：
- entity 进入特殊状态（被冻住、传送）→ 停一切自身音
- entity 销毁前的清理（虽然 entity 销毁时自动会做）

#### 完整 API 速查表

| API | 用途 | 是否需要 name？ |
|---|---|---|
| `PlaySound(event)` | 一次性 | ✗ |
| `PlaySound(event, name)` | 命名 loop | ✓ |
| `PlaySound(event, name, volume)` | 命名 + 初始音量 | ✓ |
| `PlaySound(event, nil/name, vol, is_predicted)` | 客户端预测 | 可选 |
| `PlaySoundWithParams(event, params, name, is_predicted)` | 带参数初播（15.1.5）| 可选 |
| `KillSound(name)` | 停止 | ✓ |
| `SetParameter(name, param, value)` | 改 FMOD param | ✓ |
| `SetVolume(name, volume)` | 改音量 | ✓ |
| `PlayingSound(name) → bool` | 检查 | ✓ |
| `OverrideVolumeMultiplier(volume)` | entity 整体音量倍率 | ✗ |
| `KillAllSounds()` | 停所有 | ✗ |

---

### 15.1.5（进阶）FMOD 参数化——PlaySoundWithParams + SetParameter

#### 何时需要参数化？

普通 PlaySound 适合"音效是固定的"——但很多场景音效需要**根据状态动态变化**：

| 场景 | 想要的效果 | 用普通 PlaySound 怎么做 | 用参数化怎么做 |
|---|---|---|---|
| 火堆 1-3 级 | 不同大小 | 准备 3 个 event 互相切换 | 1 个 event + intensity 参数 |
| 怪物追逐 | 紧迫感增强 | 准备 N 个 event 切换 | 1 个 event + chase 参数 |
| 走路材质 | 不同地面 | 准备 N 个 event | 1 个 event + surface 参数 |
| BGM 战斗 / 探索 | 平滑过渡 | 切两段 event 不平滑 | 1 个 event + battle 参数 |

**FMOD parameter 是 audio designer 的工具**——它把"分支音效"压缩成 1 个 event 内部的参数化处理——切换更平滑、性能更好。

#### PlaySoundWithParams 函数签名

```lua
inst.SoundEmitter:PlaySoundWithParams(event, params, [name], [is_predicted])
```

| 参数 | 类型 | 用途 |
|---|---|---|
| `event` | string | event path |
| `params` | table | parameter map（key = param name, value = number）|
| `name` | string | optional handle |
| `is_predicted` | bool | optional 客户端预测 |

#### 实战例：Walter 的小狗 Woby 脚步

```11441:11441:scripts/stategraphs/SGwilson.lua
						inst.SoundEmitter:PlaySoundWithParams("dontstarve/characters/walter/woby/big/footstep", { intensity = 1 }, nil, true)
```

**拆解**：
- `event = "dontstarve/characters/walter/woby/big/footstep"`
- `params = { intensity = 1 }`
- `name = nil`（一次性音）
- `is_predicted = true`（客户端预测）

**含义**：FMOD event 内部有个 `intensity` parameter——值为 1 时播放"全力踩踏"的版本，值为 0.5 时播放"小心翼翼"的版本。这一行在玩家奔跑状态下设 intensity = 1，**音效会更厚重**。

#### PlaySoundWithParams vs PlaySound + SetParameter

```lua
-- 方式 A：PlaySoundWithParams（一次设好）
inst.SoundEmitter:PlaySoundWithParams("event", { intensity = 0.7 }, "fire")

-- 方式 B：PlaySound + SetParameter（先播再调）
inst.SoundEmitter:PlaySound("event", "fire")
inst.SoundEmitter:SetParameter("fire", "intensity", 0.7)
```

**取舍**：
- A 在 event 内部第 1 帧就用对的参数——避免"开始时听到错误版本一瞬"
- B 给"播放后才能确定参数"的场景用——例如根据玩家姿态动态调

**推荐**：能用 A 就用 A，更干净。

#### 多参数同时设置

```lua
inst.SoundEmitter:PlaySoundWithParams("event", { 
    intensity = 0.7,
    daytime = 1,
    season = 2,
}, "haze")
```

——一次性传多个 parameter——FMOD 内部按这些参数选层。

#### 参数动态变化

```lua
-- t=0：开始播，初始 intensity = 0
inst.SoundEmitter:PlaySoundWithParams("event", { intensity = 0 }, "buildup")

-- t=2s：渐增到 1
inst:DoTaskInTime(2, function()
    inst.SoundEmitter:SetParameter("buildup", "intensity", 0.5)
end)
inst:DoTaskInTime(4, function()
    inst.SoundEmitter:SetParameter("buildup", "intensity", 1.0)
end)
```

**典型用法**：boss 战 BGM 渐进增强、紧张度逐步升高。

#### 参数化的限制

1. **参数必须 FMOD bank 内已定义**——你不能用 SetParameter 添加未声明的 param
2. **每个 event 自己的 param 互相独立**——你不能把 `"event_A"` 的 intensity 设到 `"event_B"`
3. **vanilla 中的 parameter 大都不公开** —— mod 想用 FMOD parameter 通常需要自己做 bank

#### vanilla 已知的 FMOD parameter 列表

通过 grep `SetParameter` 整理的 vanilla 已知 parameter：

| event 类别 | parameter | 取值含义 |
|---|---|---|
| `fire` 类（火堆 / 篝火）| `intensity` | 0 ~ 1，火越大值越大 |
| `fire` 类 | `daytime` | 1=白天、2=夜晚 |
| `creature/X/breath` | `health` | 0 ~ 1，血量百分比 |
| `together_amb/ocean/...` | `wave_intensity` | 0 ~ 1 |
| `BGM` 类 | `state` | 0=平静、1=战斗 |

---

### 15.1.6（进阶）音效生命周期管理——OnEntitySleep / OnEntityWake / OnRemove

#### Entity Sleep 是什么？

饥荒为了性能，会让**远离玩家视野的 entity** 进入"sleep"状态——OnUpdate / 物理 / AI 等行为都暂停。

但 SoundEmitter 不会自动停——一个 100 米外的火堆**仍然在播 fire_loop**——这浪费 CPU 和 FMOD voice。

#### vanilla 的标准模式

打开 `scripts/components/firefx.lua` 第 161 ~ 172 行：

```161:172:scripts/components/firefx.lua
function FireFX:OnEntitySleep()
    self.inst.SoundEmitter:KillSound("fire")
end

function FireFX:OnEntityWake()
    if self.playingsound ~= nil and not self.inst.SoundEmitter:PlayingSound("fire") then
        self.inst.SoundEmitter:PlaySound(self.playingsound, "fire")
        if self.playingsoundintensity ~= nil then
            self.inst.SoundEmitter:SetParameter("fire", "intensity", self.playingsoundintensity)
        end
    end
end
```

**关键模式**：
1. `OnEntitySleep` —— entity 进入 sleep → 立刻 KillSound（释放 FMOC voice）
2. `OnEntityWake` —— entity 重回视野 → **检查是否还应该播** → 重启

**为什么 OnEntityWake 内要检查 PlayingSound？**
- 防止重复 PlaySound 创建多个 instance
- 防止其他逻辑已经播了

#### 标准 OnEntitySleep / OnEntityWake 模板

```lua
-- prefab fn 内
local function fn()
    local inst = CreateEntity()
    -- ...
    inst.entity:AddSoundEmitter()
    inst.SoundEmitter:PlaySound("dontstarve/myamb", "myloop")
    
    inst.OnEntitySleep = function(inst)
        inst.SoundEmitter:KillSound("myloop")
    end
    
    inst.OnEntityWake = function(inst)
        if not inst.SoundEmitter:PlayingSound("myloop") then
            inst.SoundEmitter:PlaySound("dontstarve/myamb", "myloop")
        end
    end
    
    return inst
end
```

#### 给 mod 的启示

**所有持续音 entity** 都应该处理 OnEntitySleep / OnEntityWake——否则地图上几百个火堆 / 风车 / 机器**全部在远处仍在播音**——内存 / CPU / FMOD voice 池告急。

#### entity 销毁时

entity 销毁时**FMOD 自动停止所有音效**——但有几个细节：

**OnRemoveEntity 钩子**（如果你需要在 entity 销毁前做什么）：
```lua
inst.OnRemoveEntity = function(inst)
    -- entity 即将被销毁
    -- SoundEmitter 内部会自动 KillAllSounds，无需手动调
    -- 但你可能想在销毁前播一段"消失音"
    local x, y, z = inst.Transform:GetWorldPosition()
    local fx = SpawnPrefab("small_puff")
    fx.Transform:SetPosition(x, 0, z)
    -- 这个 fx 会在自己的 SoundEmitter 上播音
end
```

**注意**：在 `OnRemoveEntity` 内**不要再调** `inst.SoundEmitter:PlaySound`——entity 即将销毁，新音效不会被听到。

#### 玩家断线 / 重连

服务器侧 entity 不变——但**客户端 SoundEmitter 状态会重置**——重连后看到的火堆**默认不在播**——客户端会通过 entity 同步**重新触发**。

——这就是为什么每个客户端各自有 SoundEmitter loop，**而不是服务器统一控制**。

---

### 15.1.7（老手）FireFX 完整案例剖析

#### 为什么 FireFX 是最好的 SoundEmitter 教科书？

`scripts/components/firefx.lua` 175 行——它**全方位**地展示了 SoundEmitter 的所有特性：

| 特性 | FireFX 用法 |
|---|---|
| 一次性 PlaySound | `fireBurstSmall / fireBurstLarge`（点火瞬间）|
| 命名 loop | `PlaySound(params.sound, "fire")` |
| KillSound | 切换大小 / 灭火 / sleep |
| SetParameter | `SetParameter("fire", "intensity", ...)` 强度 |
| SetParameter（多 param）| `SetParameter("fire", "daytime", 1/2)` 白天 / 夜晚 |
| PlayingSound | 检查 loop 状态后再 SetParameter |
| OnEntitySleep / OnEntityWake | 离开视野停、回到视野复播 |

#### 关键代码段 1：点火瞬间一次性音

```92:94:scripts/components/firefx.lua
        if self.playignitesound and (self.level == nil or lev > self.level or stopcontrolled) then
            self.inst.SoundEmitter:PlaySound(self.lightsound or (lev >= self.bigignitesoundthresh and "dontstarve/common/fireBurstLarge" or "dontstarve/common/fireBurstSmall"))
        end
```

**模式**：火等级提升时（小→中→大）播一段"火焰爆响"——不带 name → 一次性 → 不需要后续控制。

##### 设计要点
- 根据火 level 选 small / large 不同 event
- 用 `self.lightsound` 让 prefab 可以 override（自定义点火音）

#### 关键代码段 2：持续 loop 切换

```117:126:scripts/components/firefx.lua
        if self.playingsound ~= params.sound then
            if self.playingsound ~= nil then
                self.inst.SoundEmitter:KillSound("fire")
            end
            self.playingsound = params.sound
            self.playingsoundintensity = nil
            if params.sound ~= nil and not self.inst:IsAsleep() then
                self.inst.SoundEmitter:PlaySound(params.sound, "fire")
            end
        end
```

**模式**：火等级变 → loop 音效切换。
- 检测：`self.playingsound ~= params.sound` 才切（避免无意义切换）
- 切换时：先 KillSound("fire") 停旧的、再 PlaySound 新的
- **检查 IsAsleep()**：sleep 状态不启动（避免重新唤醒时多余 PlaySound）

#### 关键代码段 3：动态参数

```128:133:scripts/components/firefx.lua
        if self.playingsoundintensity ~= params.soundintensity and params.sound ~= nil then
            self.playingsoundintensity = params.soundintensity
            if params.soundintensity ~= nil and self.inst.SoundEmitter:PlayingSound("fire") then
                self.inst.SoundEmitter:SetParameter("fire", "intensity", params.soundintensity)
            end
        end
```

**模式**：火等级 1-2-3 各有 soundintensity 值（来自 levels 表）→ SetParameter 动态调音。
- 缓存 `self.playingsoundintensity` 避免重复设置
- **必须 PlayingSound("fire")** 才设 —— 防止 fire 没启动时设置参数（FMOD warning）

#### 关键代码段 4：白天 / 夜晚切换

```51:63:scripts/components/firefx.lua
function FireFX:OnUpdate(dt)
    local time = GetTime() * 30
    self.light.Light:SetRadius(self.current_radius + .025 + (math.sin(time) + math.sin(time + 2) + math.sin(time + .7777)) * .0125)

    if self.usedayparamforsound and self.isday ~= TheWorld.state.isday then
        self.isday = TheWorld.state.isday
        self.inst.SoundEmitter:SetParameter("fire", "daytime", self.isday and 1 or 2)
    end
end
```

**模式**：在 OnUpdate 检测 TheWorld.state.isday 切换。
- usedayparamforsound 是开关（不是所有 fire 都开）
- daytime 参数：1 = 白天、2 = 夜晚

#### 关键代码段 5：灭火

```139:155:scripts/components/firefx.lua
function FireFX:Extinguish(fast)
    if self.playingsound ~= nil then
        self.inst.SoundEmitter:KillSound("fire")
        self.playingsound = nil
        self.playingsoundintensity = nil
    end

    if self.extinguishsoundtest == nil or self.extinguishsoundtest() then
        self.inst.SoundEmitter:PlaySound(self.extinguishsound or "dontstarve/common/fireOut")
		local leveldata = self.levels[self.level]
		local anim = leveldata ~= nil and ((fast and leveldata.pst_fast) or (self.controlled_burn and leveldata.pst_controlled_burn) or  leveldata.pst) or nil
		if anim ~= nil then
			self.inst.AnimState:PlayAnimation(anim)
            return true
        end
    end
end
```

**模式**：
1. 先 KillSound("fire") 停 loop
2. 再播 "fireOut" 一次性熄灭音
3. 同步播 pst 动画

——3 步串联，**视听同步**。

#### 关键代码段 6：sleep / wake 完整闭环

```161:172:scripts/components/firefx.lua
function FireFX:OnEntitySleep()
    self.inst.SoundEmitter:KillSound("fire")
end

function FireFX:OnEntityWake()
    if self.playingsound ~= nil and not self.inst.SoundEmitter:PlayingSound("fire") then
        self.inst.SoundEmitter:PlaySound(self.playingsound, "fire")
        if self.playingsoundintensity ~= nil then
            self.inst.SoundEmitter:SetParameter("fire", "intensity", self.playingsoundintensity)
        end
    end
end
```

**OnEntityWake 的精妙之处**：
1. 检查 `self.playingsound ~= nil` —— 之前是否在播（如果火被灭了，就不要重启）
2. 检查 `not PlayingSound("fire")` —— 当前是否已经在播（避免重复）
3. 重启 PlaySound + SetParameter 还原参数

**这个模式适用于所有持续音 entity** —— 强烈推荐复制到 mod 用。

#### FireFX 完整音效模式总结

```
点火: PlaySound("fireBurstSmall")   ← 一次性
       ↓
loop 启动: PlaySound("campfire", "fire")
       ↓
按 level 切换 loop: KillSound("fire") + PlaySound(new_sound, "fire")
       ↓
按 intensity 调: SetParameter("fire", "intensity", value)
       ↓
按白天 / 夜晚调: SetParameter("fire", "daytime", value)
       ↓
sleep: KillSound("fire")
       ↓
wake: 检查后 PlaySound + 还原参数
       ↓
灭火: KillSound("fire") + PlaySound("fireOut")
```

---

### 15.1.8（老手）6 大常见陷阱

#### 陷阱 1：忘了 AddSoundEmitter

```lua
local inst = CreateEntity()
inst.entity:AddTransform()
inst.SoundEmitter:PlaySound("...")  -- ✗ attempt to index nil
```

**正解**：所有用 PlaySound 的 entity 必须先 `inst.entity:AddSoundEmitter()`。

#### 陷阱 2：one-shot 想 KillSound

```lua
-- ✗ 错误期望
inst.SoundEmitter:PlaySound("dontstarve/common/deathpoof")
inst:DoTaskInTime(0.1, function()
    inst.SoundEmitter:KillSound("dontstarve/common/deathpoof")  -- ✗ 没用
end)
```

**问题**：one-shot 没有 name handle —— KillSound 不知道要杀谁。

**正解**：要可控 → 必须带 name：
```lua
inst.SoundEmitter:PlaySound("dontstarve/common/deathpoof", "poof")
inst:DoTaskInTime(0.1, function()
    inst.SoundEmitter:KillSound("poof")
end)
```

#### 陷阱 3：同 name 多次 PlaySound

```lua
inst.SoundEmitter:PlaySound("event", "fire")  -- 第 1 次
inst.SoundEmitter:PlaySound("event", "fire")  -- 第 2 次 → 自动停第 1 次
```

**行为**：第 2 次 PlaySound 会**自动停止第 1 次**——这是 FMOD 的去重机制。

**好处**：不会出现"火堆音越叠越响"。
**陷阱**：有时你想"叠加多份"——这种场景必须用**不同 name**：
```lua
inst.SoundEmitter:PlaySound("event", "fire1")
inst.SoundEmitter:PlaySound("event", "fire2")  -- 现在两份都在播
```

#### 陷阱 4：在 client 端 PlaySound 持续音

```lua
local function fn()
    local inst = CreateEntity()
    -- ...
    inst.entity:AddSoundEmitter()
    
    -- ✗ 写在 client 段（SetPristine 之前）
    inst.SoundEmitter:PlaySound("event", "loop")
    
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end
    return inst
end
```

**问题**：客户端 / 服务器各自启动 loop —— 玩家听到双倍音量。

**正解**：持续音应该**只在 master 端启动**，靠 entity 网络同步让客户端也启动：
```lua
inst.entity:SetPristine()
if not TheWorld.ismastersim then return inst end

-- 仅 master 段
inst.SoundEmitter:PlaySound("event", "loop")
```

——客户端会通过**entity replication 自动跟随**（FMOD 内部机制）。

#### 陷阱 5：dedicated server 上播 SoundEmitter

dedicated server 没有渲染、没有音频输出——但它仍然会**调 PlaySound**——只是听不到（也不会崩）。

**性能影响**：dedicated 上每个 fire entity 仍在尝试播 fire_loop —— 哪怕没人听 —— 这浪费一些 CPU。

**优化**（vanilla 中常见）：
```lua
if not TheNet:IsDedicated() then
    inst.entity:AddSoundEmitter()
end
```

——dedicated 不加 SoundEmitter，PlaySound 不存在 inst.SoundEmitter，需要后续判空：
```lua
if inst.SoundEmitter then
    inst.SoundEmitter:PlaySound("...")
end
```

#### 陷阱 6：sleep / wake 不处理

```lua
local function fn()
    local inst = CreateEntity()
    -- ...
    inst.entity:AddSoundEmitter()
    inst.SoundEmitter:PlaySound("event", "myloop")
    -- ✗ 没处理 OnEntitySleep
    return inst
end
```

**后果**：
- 100 个 entity 离开玩家视野后**仍在播 myloop**
- FMOD voice 池占满（默认 ~32 个 voice）
- 新音效**无声播放**（被去重）
- 玩家以为"游戏没声音了"

**正解**：所有持续音 entity 都加 OnEntitySleep / OnEntityWake：
```lua
inst.OnEntitySleep = function(inst)
    inst.SoundEmitter:KillSound("myloop")
end

inst.OnEntityWake = function(inst)
    if not inst.SoundEmitter:PlayingSound("myloop") then
        inst.SoundEmitter:PlaySound("event", "myloop")
    end
end
```

---

### 15.1.9（小结）速查表 + 4 行起步代码 + 15.2 预告

#### SoundEmitter 完整 API 速查

| API | 是否需要 name | 用途 |
|---|---|---|
| `AddSoundEmitter()` | - | entity 设置时一次 |
| `PlaySound(event)` | ✗ | 一次性音 |
| `PlaySound(event, name)` | ✓ | 命名 loop |
| `PlaySound(event, name, vol)` | ✓ | 命名 loop + 初始音量 |
| `PlaySound(event, name, vol, true)` | 可选 | 客户端预测 |
| `PlaySoundWithParams(event, params, name, predicted)` | 可选 | 带 FMOD 参数初播 |
| `KillSound(name)` | ✓ | 立即停 |
| `SetParameter(name, param, value)` | ✓ | 动态调 FMOD param |
| `SetVolume(name, value)` | ✓ | 动态调音量 |
| `PlayingSound(name) → bool` | ✓ | 检查 |
| `OverrideVolumeMultiplier(value)` | ✗ | entity 整体倍率 |
| `KillAllSounds()` | ✗ | 紧急停所有 |

#### 4 大模式速查

| 模式 | 适用场景 | 代码 |
|---|---|---|
| One-shot | 攻击 / 爆炸 / 死亡 | `PlaySound(event)` |
| Named loop | 火 / 风 / 持续音 | `PlaySound(event, "name")` + `KillSound("name")` |
| Param loop | 大小 / 季节切换 | `+ SetParameter(name, param, value)` |
| Predicted | 走路 / 攻击预测 | `PlaySound(event, name, vol, true)`（仅 SG client）|

#### 8 大命名空间速查

| 前缀 | 内容 |
|---|---|
| `dontstarve/` | 基础 |
| `dontstarve_DLC001/` | RoG |
| `turnoftides/` | 海洋月岛 |
| `hookline_2/` | 钓鱼 |
| `meta4/` | 月族阴影 |
| `rifts/` | 月族裂缝 |
| `monkeyisland/` | 猴岛 |
| `webber2/`、`waterlogged2/` 等 | 各更新 |

#### 4 行起步代码

##### A：一次性音

```lua
inst.entity:AddSoundEmitter()
inst.SoundEmitter:PlaySound("dontstarve/common/deathpoof")
```

##### B：持续 loop（带 sleep / wake）

```lua
inst.entity:AddSoundEmitter()
inst.SoundEmitter:PlaySound("dontstarve/AMB/forest", "amb")

inst.OnEntitySleep = function(inst)
    inst.SoundEmitter:KillSound("amb")
end
inst.OnEntityWake = function(inst)
    if not inst.SoundEmitter:PlayingSound("amb") then
        inst.SoundEmitter:PlaySound("dontstarve/AMB/forest", "amb")
    end
end
```

##### C：参数化 loop（动态强度）

```lua
inst.SoundEmitter:PlaySoundWithParams("event", { intensity = 0.5 }, "haze")

-- 后续动态调
inst.SoundEmitter:SetParameter("haze", "intensity", 1.0)
```

#### 三个设计原则

1. **一次性用 PlaySound 单参，持续用带 name** —— 区分清楚才能控制。
2. **持续音必处理 sleep / wake** —— 否则远处的 entity 浪费 FMOD voice。
3. **服务器 / 客户端各自 PlaySound 一次** —— 持续音只在 master 端启动，靠 entity 同步给 client。

---

> **下一节预告**：15.2 节我们将进入 **饥荒的音频架构（FMOD 集成）**——前面 15.1 讲了**怎么用** SoundEmitter——15.2 会讲**为什么这样设计** —— FMOD bank 加载、event 路径解析、voice 池管理、3D pan / attenuation 算法、客户端 / 服务器 / dedicated 各自的音频角色——15.2 是音频系统的**架构总章**。

---


## 15.2 饥荒的音频架构（FMOD 集成）

### 本节导读

> **一句话定位**：15.1 节告诉你**怎么调** SoundEmitter——15.2 节告诉你**为什么这样设计**——从 FMOD bank 文件、event 路径解析、Mixer 混音通道、DSP 滤波器、到 dedicated server 的特殊处理——15.2 是饥荒音频系统的**架构总章**。

#### 一段宏观先讲清楚：饥荒为什么用 FMOD？

15.1 节我们看到了 `inst.SoundEmitter:PlaySound("dontstarve/wilson/attack_weapon")` ——但这一行 Lua 代码背后**究竟发生了什么**？

```
Lua 代码：
inst.SoundEmitter:PlaySound("dontstarve/wilson/attack_weapon")
                    │
                    ▼
        ┌───────────────────────────────────────┐
        │   1. C++ binding：调进 game engine     │
        │      ─ FMODSystem 接口                  │
        └───────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────────────────────┐
        │   2. FMOD 解析路径，找到 event         │
        │      ─ 在 dontstarve.fev 里找          │
        │        path = "dontstarve/wilson/..."  │
        └───────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────────────────────┐
        │   3. 检查对应 .fsb 是否已 PreloadFile   │
        │      ─ 若已加载 → 复用                  │
        │      ─ 若未加载 → 现场加载（顿一下）   │
        └───────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────────────────────┐
        │   4. 分配一个 EventInstance             │
        │      ─ FMOD voice 池中找一个空位        │
        │      ─ 满了 → 自动剔除最旧的            │
        └───────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────────────────────┐
        │   5. 应用 3D pan / attenuation         │
        │      ─ 根据 entity.Transform 位置       │
        │      ─ 与玩家相机距离计算音量           │
        └───────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────────────────────┐
        │   6. 通过通道（amb/sfx/music/...）混音  │
        │      ─ TheMixer 当前激活的 mix          │
        │      ─ DSP 滤波器（耳塞帽 lowpass）    │
        └───────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────────────────────┐
        │   7. 输出到玩家的音箱 / 耳机            │
        └───────────────────────────────────────┘
```

——15.2 节就是要把这 7 步**全部讲透**——你将看到饥荒音频系统每一层的设计哲学。

#### FMOD 在饥荒中的"4 层架构"

```
┌────────────────────────────────────────────────────────────────┐
│   ▎ 第 4 层：Lua API 层（开发者面对）▎                          │
│   - SoundEmitter:PlaySound / KillSound / SetParameter         │
│   - TheMixer:PushMix / PopMix                                  │
│   - PostProcessor 各种 SetFilter                               │
└────────────────────────────────────────────────────────────────┘
                             ▲
                             │ Lua → C++ binding
                             │
┌────────────────────────────────────────────────────────────────┐
│   ▎ 第 3 层：饥荒引擎层（C++）▎                                 │
│   - FMODSystem（全局单例）                                       │
│   - SoundEmitter 实例（每个 entity 一个）                       │
│   - Mixer / DSP 处理                                             │
└────────────────────────────────────────────────────────────────┘
                             ▲
                             │ FMOD API
                             │
┌────────────────────────────────────────────────────────────────┐
│   ▎ 第 2 层：FMOD Studio runtime ▎                              │
│   - 解析 .fev 元数据                                             │
│   - 加载 .fsb 音频流                                             │
│   - voice 池管理 / 3D pan                                       │
└────────────────────────────────────────────────────────────────┘
                             ▲
                             │ 文件 I/O
                             │
┌────────────────────────────────────────────────────────────────┐
│   ▎ 第 1 层：磁盘文件 ▎                                         │
│   - sound/dontstarve.fev (元数据 / event 路径定义)              │
│   - sound/forest.fsb (音频流压缩数据)                            │
│   - sound/krampus.fsb (各类怪物 / 角色 / 环境音)                 │
│   - 总计 100+ 个 .fev/.fsb 文件                                 │
└────────────────────────────────────────────────────────────────┘
```

**关键理解**：
- **`.fev` = FMOD Event Bank**：定义了 event 路径（"dontstarve/wilson/attack_weapon"）的元数据，但本身不含音频流——**只是"目录文件"**
- **`.fsb` = FMOD Sound Bank**：包含真正的压缩音频流（OGG / Vorbis 编码），**比 .wav 小 10 倍**
- **PKGREF**：vanilla 的 master event bank（`dontstarve.fev`），其它 .fev 都依赖它
- **SOUNDPACKAGE / SOUND**：mod 注册声音的两种方式（详见 15.2.2）

#### 15.2 节回答的 5 个核心问题

```
Q1: ──── FMOD 是什么？.fev / .fsb 各有什么作用？
        ↓ 答：FMOD = 音频中间件、.fev=元数据、.fsb=音频流

Q2: ──── 我的 mod 怎么注册自己的音频文件？
        ↓ 答：Asset("SOUNDPACKAGE", ".fev") + Asset("SOUND", ".fsb") + PreloadSoundList

Q3: ──── TheMixer 是什么？它和 SoundEmitter 是什么关系？
        ↓ 答：Mixer 是"全局通道音量调度"——SoundEmitter 是"单 entity 发声"

Q4: ──── 耳塞帽、戴头盔、被冰冻时声音变闷怎么实现的？
        ↓ 答：DSP 滤波器（lowpass / highpass）通过 PlayerHearing 组件触发

Q5: ──── 客户端 / 服务器 / dedicated server 各自的音频角色是什么？
        ↓ 答：dedicated 不出声、master 决策、client 渲染——各自独立的 SoundEmitter
```

#### 你将看到 6 份核心源码

| 文件 | 角色 | 主要内容 |
|---|---|---|
| `scripts/mixer.lua` | 267 行 | **Mixer 核心实现**（push / pop / blend / DSP）|
| `scripts/mixes.lua` | 261 行 | **vanilla 的 11 种 mix 配置**（normal / death / pause / 等）|
| `scripts/preloadsounds.lua` | 283 行 | **`.fsb` 文件预加载列表 + `PreloadSoundList` 函数** |
| `scripts/components/playerhearing.lua` | 78 行 | **DSP 滤波器（耳塞帽）实现** |
| `scripts/prefabs/global.lua` | 539+ 行 | **vanilla 全部音频 Asset 注册清单** |
| `scripts/main.lua` | 第 440-485 行 | **音频系统初始化序列** |

#### 本节学习路径

```
15.2.1 新手  ──────  FMOD 是什么 + .fev/.fsb 的角色
15.2.2 新手  ──────  Asset 资产注册：4 种声音 Asset 类型对比
15.2.3 新手  ──────  PreloadSoundList vs Asset 加载机制
                       ↓
15.2.4 进阶  ──────  TheMixer 混音器：11 种 mix + 优先级栈
15.2.5 进阶  ──────  DSP 滤波器（lowpass / highpass）+ 耳塞帽实战
15.2.6 进阶  ──────  10 大声音通道（amb/cloud/music/...）的语义
                       ↓
15.2.7 老手  ──────  服务器 / 客户端 / dedicated 三角色分工
15.2.8 老手  ──────  网络同步、3D 衰减、性能优化
15.2.9 小结  ──────  速查表 + 4 行起步代码 + 15.3 预告
```

**给三类读者的承诺**：
- **新手**：你将真正理解为什么"调用 PlaySound 之前要 Asset 注册"——以及 vanilla 100+ 个 `.fsb` 文件的命名规则。
- **进阶**：你将完整掌握 TheMixer 11 种 mix 的应用场景——能在自己的 mod 里 `PushMix("custom")` 制造特殊氛围。
- **老手**：你将看清饥荒"客户端预测 + 服务器决策 + dedicated 静音"的网络音频架构——并能写出 dedicated friendly 的 mod 音频代码。

---

### 15.2.1（新手）FMOD 是什么 + .fev/.fsb 的角色

#### FMOD 是什么？

**FMOD 是商业级音频中间件**——由芬兰公司 Firelight Technologies 开发——授权使用于：
- Don't Starve / Don't Starve Together（饥荒）
- World of Warcraft（魔兽世界）
- Civilization（文明）
- Forza（极限竞速）
- 等数千款游戏

vanilla 在致谢字符串里明确声明：

```9131:9131:scripts/strings.lua
        FMOD = "FMOD Sound System,\nCopyright Firelight Technologies",
```

**为什么选 FMOD 而不是直接调操作系统的音频 API？**

| 需求 | OS 原生 API | FMOD |
|---|---|---|
| 跨平台（Win / Mac / Linux / Switch / PS）| 各平台不同 | 一套代码 |
| 3D 空间音效（pan / 距离衰减）| 自己实现 | 内置 |
| 同时播放多个音（voice 池）| 自己管理 | 自动调度 |
| 实时滤波（lowpass / reverb）| 自己实现 DSP | 一行 API |
| 设计师工具链（Studio）| 无 | 完整 GUI |

——简而言之：**FMOD 让游戏开发者只关心"什么时候播什么"，把所有音频底层硬活外包出去**。

> **诚实边界**：vanilla 的 `scripts/` 目录中**几乎搜不到 "FMOD"** 关键词——FMOD 的 runtime 实现在 **C++ 层**（不在 Lua 中）。Lua 只看到经过 binding 包装后的 API（如 `inst.SoundEmitter:PlaySound`、`TheMixer:PushMix`、`TheSim:SetSoundVolume`）——这是引擎的设计：**业务逻辑 Lua 化、底层音频 C++ 化**。

#### Don't Starve 的 4 类音频文件

打开饥荒游戏目录的 `sound/` 文件夹（实际通常打包在 `data\sound\` 内），你会看到：

```
sound/
├── dontstarve.fev          ← 主元数据 bank（PKGREF）
├── dontstarve_DLC001.fev   ← RoG 元数据
├── turnoftides.fev         ← 海洋月岛元数据
├── ...
├── forest.fsb              ← 森林环境音流
├── spider.fsb              ← 蜘蛛音效流
├── monkeyisland.fsb        ← 猴岛音效流
├── ...（共 100+ 个 .fsb）
└── DLC_music.fsb           ← BGM 流
```

**4 大文件类型**：

##### 1. `.fev`（FMOD Event Bank / 元数据）

- **作用**：定义 event 路径（`"dontstarve/wilson/attack_weapon"` 长这样）的层次结构、参数、随机变体规则
- **不包含音频数据**——只是路径目录
- **依赖**：`dontstarve.fev` 是**所有其他 .fev 的根 bank**——必须最先加载
- **大小**：通常很小（KB 级）

##### 2. `.fsb`（FMOD Sound Bank / 音频流）

- **作用**：实际音频数据，OGG/Vorbis 压缩
- **大小**：MB 级（每个怪物/场景一个）
- **配对**：通常每个 .fev 对应 1+ 个 .fsb（例如 `webber2.fev` 配 `webber2.fsb`）
- **加载**：通过 PreloadFile 常驻内存，或按需加载（首次播音时）

##### 3. PKGREF（Package Reference / 主 bank 引用）

- 仅 vanilla `prefabs/global.lua` 第 3 行：

```3:3:scripts/prefabs/global.lua
    Asset("PKGREF", "sound/dontstarve.fev"),
```

- **唯一一个 PKGREF**——它是其他 .fev 的依赖根
- mod 不应该用 PKGREF——只用 SOUNDPACKAGE / SOUND

##### 4. mod 工程中可能见到的 `.wav` / `.ogg` 原始素材

- **不是引擎能直接读的**——必须用 FMOD Studio 编辑器**生成 .fev + .fsb**
- 详见 15.5 节"完整 FMOD 流程"

#### 一段对照：vanilla 的全部 .fsb 文件清单

打开 `scripts/preloadsounds.lua` 第 23-258 行——这是 vanilla **全部预加载 fsb 文件清单**——浓缩在一起就是饥荒的"音效字典"：

```23:69:scripts/preloadsounds.lua
local MainSounds =
{
	"bat.fsb",
	"bee.fsb",
	"beefalo.fsb",
	"birds.fsb",
	"bunnyman.fsb",
	"cave_AMB.fsb",
	"cave_mem.fsb",
	"chess.fsb",
	"chester.fsb",
	"common.fsb",
	"deerclops.fsb",
	"dontstarve.fev",
	"forest.fsb",
	"forest_stream.fsb",
    "forge2.fsb",
	"frog.fsb",
	"ghost.fsb",
	"gramaphone.fsb",
	"hound.fsb",
	"koalefant.fsb",
	"krampus.fsb",
    "lava_arena.fsb",
	"leif.fsb",
	"mandrake.fsb",
	"maxwell.fsb",
	"mctusky.fsb",
	"merm.fsb",
	"monkey.fsb",
	"music.fsb",
	"pengull.fsb",
	"perd.fsb",
	"pig.fsb",
	"plant.fsb",
    "quagmire.fsb",
	"rabbit.fsb",
	"rocklobster.fsb",
	"sanity.fsb",
	"sfx.fsb",
	"slurper.fsb",
	"slurtle.fsb",
	"spider.fsb",
	"tallbird.fsb",
	"tentacle.fsb",
    "together.fsb",
	"wallace.fsb",
```

**命名规律**：
- **怪物名直接命名**：`spider.fsb`、`bee.fsb`、`leif.fsb`
- **环境名命名**：`forest.fsb`、`cave_AMB.fsb`、`forest_stream.fsb`
- **通用音**：`common.fsb`（爆炸 / 死亡 / 烟雾等通用）、`sfx.fsb`
- **资料片**：`dontstarve_DLC001.fev`、`turnoftides.fsb`、`webber2.fsb`
- **角色重做版**：`WX_rework.fsb`、`wickerbottom_rework.fsb`、`woodie2.fsb`

#### .fev / .fsb 的工作流程图

```
玩家点开始游戏
        │
        ▼
┌─────────────────────────────────────────┐
│ 1. 引擎加载 dontstarve.fev (PKGREF)      │
│    ─ 解析整个 event 路径树               │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 2. PreloadSoundList(MainSounds)          │
│    ─ 遍历 50+ 个 .fsb 文件                │
│    ─ TheSim:PreloadFile("sound/X.fsb")   │
│    ─ 流数据加载到内存（或 mmap）          │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 3. 玩家进入世界                           │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 4. 蜘蛛 spawn → spider 的 .fsb 已加载    │
│    inst.SoundEmitter:PlaySound(          │
│       "dontstarve/creatures/spider/walk")│
│    ↓                                     │
│    FMOD 在 spider.fsb 内找到流 → 播       │
└─────────────────────────────────────────┘
```

#### 给 mod 的启示

**问题**：mod 如果**没注册 .fsb**就调 PlaySound，会发生什么？

**答**：FMOD 会 log warning："Event path 'mymod/xxx' not found"——**没声音播但不崩**。

**结论**：mod 加自己的音频，必须：
1. **生成 .fev + .fsb**（用 FMOD Studio）
2. **Asset 注册**（让引擎知道）
3. **PreloadSoundList**（提前加载）

——这正是 15.2.2 / 15.2.3 要讲的。

---

### 15.2.2（新手）资产注册——4 种声音 Asset 类型对比

#### Asset 是什么？

`Asset` 是饥荒的**资产声明**——它告诉引擎："这个 prefab 需要用到 X 文件"——引擎据此决定**加载哪些文件**。

```lua
local assets = {
    Asset("ANIM", "anim/wilson.zip"),
    Asset("SOUND", "sound/wilson.fsb"),
}
return Prefab("wilson", fn, assets)
```

#### 4 种声音相关的 Asset 类型

| Asset 类型 | 用途 | 用于 |
|---|---|---|
| `Asset("PKGREF", "sound/X.fev")` | 主 bank 引用 | **仅 vanilla `dontstarve.fev`**——mod 不要用 |
| `Asset("SOUNDPACKAGE", "sound/X.fev")` | event bank | mod 注册自己的 .fev |
| `Asset("SOUND", "sound/X.fsb")` | 音频流 | mod 注册自己的 .fsb |
| `Asset("FILE", "sound/X.fsb")` | 通用文件 | vanilla 部分 .fsb（功能上等同 SOUND）|

#### 实例对照：vanilla 与 mod

##### vanilla 主 bank（独一无二）

```3:13:scripts/prefabs/global.lua
    Asset("PKGREF", "sound/dontstarve.fev"),
    Asset("SOUNDPACKAGE", "sound/dontstarve_DLC001.fev"),
    Asset("FILE", "sound/DLC_music.fsb"),
    Asset("SOUNDPACKAGE", "sound/turnoftides.fev"),
    Asset("FILE", "sound/turnoftides.fsb"),
    Asset("SOUNDPACKAGE", "sound/saltydog.fev"),
    Asset("FILE", "sound/saltydog.fsb"),
    Asset("SOUNDPACKAGE", "sound/hookline.fev"),
    Asset("FILE", "sound/hookline.fsb"),
    Asset("SOUNDPACKAGE", "sound/hookline_2.fev"),
    Asset("FILE", "sound/hookline_2.fsb"),
```

##### vanilla 单个 prefab 的 SOUND 引用

```799:799:scripts/prefabs/warg.lua
        Asset("SOUND", "sound/vargr.fsb"),
```

##### mod 完整声音注册（神话 mod）

```169:189:mods/联机版mod/神话未加密/modmain.lua
	Asset("SOUNDPACKAGE", "sound/buttons.fev"),
	Asset("SOUND", "sound/myth_jssound.fsb"),

	Asset("SOUNDPACKAGE", "sound/Myth_nian.fev"),
	Asset("SOUND", "sound/Myth_nian.fsb"),

	Asset("SOUNDPACKAGE", "sound/bianzhong.fev"),
    Asset("SOUND", "sound/bianzhong.fsb"),

	Asset("SOUNDPACKAGE", "sound/myth_icon_sound.fev"),
    Asset("SOUND", "sound/myth_icon_sound.fsb"),

	Asset("SOUNDPACKAGE", "sound/mythsound_rhino.fev"),
    Asset("SOUND", "sound/mythsound_rhino.fsb"),
	Asset("SOUNDPACKAGE", "sound/Laozi.fev"),
    Asset("SOUND", "sound/Laozi.fsb"),

	Asset("SOUNDPACKAGE", "sound/bamboo_clappers.fev"),
    Asset("SOUND", "sound/bamboo_clappers.fsb"),
	Asset("SOUNDPACKAGE", "sound/crane.fev"),
    Asset("SOUND", "sound/crane.fsb"),
```

**模式**：每个自定义 bank**成对**注册——`SOUNDPACKAGE` + `SOUND`——**两者缺一不可**。

#### "万物书" mod 的批量注册模式

`mods/联机版mod/万物书/imports_of_tbat/06_load_sounds.lua` —— 提供了一份非常清晰的 mod 范本：

```lua
-- mod 内部把 fev / fsb 文件名集中收口
local fev_sound_files = {
    "dontstarve_DLC002.fev",
    "dontstarve_DLC003.fev",
    "tbat_sound_stage_1.fev",
}
for k, file_name in pairs(fev_sound_files) do
    if file_name then
        table.insert(Assets, Asset("SOUNDPACKAGE", "sound/"..file_name))
    end
end

local fsb_sound_files = {
    "DLC003_sfx.fsb",
    "tbat_sound_stage_1.fsb",
}
for k, file_name in pairs(fsb_sound_files) do
    if file_name then
        table.insert(Assets, Asset("SOUND", "sound/"..file_name))
    end
end

-- 立即预加载
PreloadSoundList(fev_sound_files)
PreloadSoundList(fsb_sound_files)
```

**关键模式**：
- 用 table 集中声明所有声音文件名
- 循环 insert 进 Assets
- **同时调用 PreloadSoundList** —— 详见 15.2.3

#### Asset("SOUND") vs Asset("FILE") 的区别

vanilla 用了**两种** —— `SOUND` 和 `FILE` —— 都用于 `.fsb`。

##### 实测结果

- **`Asset("SOUND", ...)`**：标准方式，引擎认识，优先加载
- **`Asset("FILE", ...)`**：通用文件，引擎也认识（但不专门作为音频），有时用于"想用 .fsb 但不想被引擎当音频管理"的场景

##### vanilla 内的实际使用

`prefabs/global.lua` 中混用——很多 `.fsb` 用 `FILE` 而不是 `SOUND`：
```5:7:scripts/prefabs/global.lua
    Asset("FILE", "sound/DLC_music.fsb"),
    Asset("SOUNDPACKAGE", "sound/turnoftides.fev"),
    Asset("FILE", "sound/turnoftides.fsb"),
```

而单 prefab 用 `SOUND`：
```799:799:scripts/prefabs/warg.lua
        Asset("SOUND", "sound/vargr.fsb"),
```

##### 推荐 mod 使用

**新手 mod 一律用 `SOUND`** —— 这是 mod 社区的惯例，最不易出错。

#### "我的 mod 加载的 .fsb 找不到 event 路径怎么办"

**症状**：
```
Event path 'mymod/character/howl' not found
```

**5 个可能原因**：

1. **没注册 SOUNDPACKAGE**：只注册了 `.fsb` 没注册 `.fev`
2. **路径名写错**：FMOD Studio 工程里实际是 `mymod/character/howl_loud`
3. **.fev 文件没和 .fsb 匹配**：用旧 .fev + 新 .fsb（FMOD 项目导出时不一致）
4. **mod 加载顺序问题**：你的 mod 依赖另一个 mod 的音频
5. **dontstarve.fev 没加载**：理论不会，但极少数情况下确实见过

**调试步骤**：
1. 确认 `Asset("SOUNDPACKAGE", "sound/X.fev")` + `Asset("SOUND", "sound/X.fsb")` 都注册
2. 确认 `PreloadSoundList` 调用了
3. 用 FMOD Studio 检查 .fev 内的 event path 是否存在
4. 在游戏控制台 `print(...)` 验证 mod 是否真加载了那个 modmain

---

### 15.2.3（新手）PreloadSoundList vs Asset 加载机制

#### 两者的根本区别

```
Asset 注册                       PreloadSoundList
   │                                   │
   │ 告诉引擎：                       │ 告诉引擎：
   │ "这个文件是我 prefab 的资产"     │ "立即把这个文件载入内存"
   │                                   │
   ▼                                   ▼
"按 prefab 加载策略"             "立即预加载，不等 prefab"
   │                                   │
   ▼                                   ▼
随 prefab 引用动态加载            一次加载，常驻内存
   │                                   │
   ▼                                   ▼
延迟加载，节省内存                启动时间略慢，但运行时无 hitch
```

#### PreloadSoundList 源码

```261:265:scripts/preloadsounds.lua
function PreloadSoundList(list)
	for i,v in pairs(list) do
		TheSim:PreloadFile("sound/"..v)
	end
end
```

——**5 行实现**——本质就是循环调 `TheSim:PreloadFile("sound/X.fsb")`。

#### TheSim:PreloadFile 做了什么？

C++ binding——它告诉 FMOD：**这个文件等会儿要用，请提前打开 + load 到内存**。

效果：
- 文件流提前进内存（首次 PlaySound 不会 hitch）
- 即使 prefab 没 spawn，文件也被持有（不释放）

#### 为什么 vanilla 必须 PreloadSoundList？

`scripts/preloadsounds.lua` 的 `MainSounds` 列出**50+ 个 .fsb**——为什么所有都预加载？

**原因 1：避免运行时 hitch**

如果蜘蛛第一次出现时才加载 spider.fsb，**那一瞬间会卡顿**——因为：
- 文件 I/O 阻塞主线程
- FMOD 元数据解析需要 CPU
- 蜘蛛的攻击声因此延迟

**原因 2：玩家随时可能遇到任何怪物**

饥荒是开放世界——不知道**下一秒**玩家会撞到什么——预加载所有怪物的 .fsb 是最稳的。

**原因 3：内存占用可接受**

50+ 个 .fsb 加起来约 100-200 MB——现代 PC 完全能承受。

#### 不需要预加载的场景

vanilla 中**仅** `frontend.lua` 的部分音乐没有进 `MainSounds`：

```195:198:scripts/prefabs/frontend.lua
    Asset("SOUND", "sound/gramaphone.fsb"),

    Asset("PKGREF", "sound/music_frontend.fsb"),
```

——`music_frontend.fsb` 是主菜单音乐，**仅在主菜单时用**——不需要在游戏中常驻。

#### mod 的预加载模式

打开 vanilla mods，几乎所有都是这个模式：

```
modmain.lua
   │
   ├─ Asset("SOUNDPACKAGE", "sound/mymod.fev")  ← 资产声明
   ├─ Asset("SOUND", "sound/mymod.fsb")         ← 资产声明
   │
   └─ PreloadSoundList({"mymod.fev", "mymod.fsb"})  ← 立即预加载
```

万物书 mod 的实现就是模板：

```lua
PreloadSoundList(fev_sound_files)
PreloadSoundList(fsb_sound_files)
```

#### "我能不能 lazy load？"

**理论上可以**——只 Asset 注册不 PreloadSoundList——首次 PlaySound 时 FMOD 现场加载。

**实测问题**：
- 现场加载有几十 ms hitch（玩家感受得到）
- 多个 PlaySound 同时触发 → 多个文件并发加载 → IO 雪崩

**结论**：**所有 .fev/.fsb 都应该 Preload**——除非你确认那个文件极少用（如主菜单 BGM）。

#### 加载顺序

modmain.lua 内的代码**按从上到下顺序执行**：

```lua
-- modmain.lua
local Assets = {...}                        -- ① 资产声明
PreloadSoundList({...})                     -- ② 立即触发预加载
modimport("scripts/prefabs/myprefab.lua")   -- ③ 加载 prefab
PrefabFiles = {"myprefab"}                  -- ④ 注册 prefab
```

**最佳实践**：
1. Asset 表先全部声明完
2. 然后 PreloadSoundList
3. 最后 modimport / PrefabFiles

——保证音频在 prefab 引用之前已经载入。

---

### 15.2.4（进阶）TheMixer 混音器——11 种 mix + 优先级栈

#### Mixer 是什么？

15.1 节学的 SoundEmitter 控制**单 entity 单声音**——但**整个游戏的声音音量调度**由谁负责？

**答**：`TheMixer`——全局混音器单例——管理"全游戏的所有声音通道音量"。

```
┌────────────────────────────────────────────┐
│ ▎ SoundEmitter ▎                            │
│ 控制单个声音：play / kill / volume         │
│ 范围：单 entity                             │
└────────────────────────────────────────────┘
                  ↓
                  ↓ 通过通道分类
                  ↓
┌────────────────────────────────────────────┐
│ ▎ TheMixer ▎                                │
│ 控制全游戏 10 个通道的 master volume         │
│ 范围：全游戏                                │
└────────────────────────────────────────────┘
                  ↓
                  ↓ 输出
                  ↓
              玩家音箱
```

#### Mixer 的核心数据结构

```35:41:scripts/mixer.lua
local Mixer = Class(function(self)
    self.mixes = {}
    self.stack = {}

    self.lowpassfilters = {}
    self.highpassfilters = {}
end)
```

- `self.mixes` —— 所有定义的 mix 配置（`{ name → Mix object }`）
- `self.stack` —— **当前激活的 mix 栈**（按优先级排序）
- `self.lowpassfilters` / `self.highpassfilters` —— DSP 滤波器状态（详见 15.2.5）

#### 一个 Mix 是什么？

```3:8:scripts/mixer.lua
local Mix = Class(function(self, name)
    self.name = name or ""
    self.levels = {}
    self.priority = 0
    self.fadeintime = 1
end)
```

- `name` —— mix 名（"normal"、"death"、"pause" 等）
- `levels` —— 通道音量表（`{"set_music/soundtrack" = 1, "set_ambience/ambience" = 0.8, ...}`）
- `priority` —— 优先级（越高越靠前，决定 stack 排序）
- `fadeintime` —— 切换到这个 mix 时的渐变时间

#### vanilla 的 11 种核心 mix

打开 `scripts/mixes.lua`——列出了所有内置 mix：

| Mix 名 | priority | fadetime | 用途 |
|---|---|---|---|
| `start` | 0 | 1 | 游戏启动 |
| `normal` | 1 | 2 | **默认游戏内**（最常见）|
| `slurp` | 1 | 1 | 被吃 / 吞下时（精神冲击）|
| `lavaarena_normal` | 1 | 0.1 | 熔炉模式 |
| `supernova_charging` | 2 | 0.6 | 月神蓄力 |
| `high` | 3 | 2 | 高强度场景（boss 战）|
| `flying` | 3 | 2 | 木腿飞行 |
| `supernova` | 3 | 0 | 月神爆发 |
| `pause` | 4 | 1 | 暂停菜单 |
| `wx_screech` | 4 | 0.6 | WX-78 尖啸 |
| `minigamescreen` | 4 | 1 | 小游戏（钓鱼 / 写信）|
| `death` | 6 | 1 | **玩家死亡**（高优先级）|
| `lobby` | 8 | 2 | 大厅（角色选择）|
| `silence` | 8 | 0 | 静音（剧情过场）|
| `moonstorm` | 8 | 2 | 月风暴 |
| `serverpause` | 2147483647 | 0 | **服务器暂停**（最高，强制盖一切）|

#### Mix 的 push / pop / blend 机制

##### PushMix —— 入栈

```132:149:scripts/mixer.lua
function Mixer:PushMix(mixname)

    local mix = self.mixes[mixname]

    local current = self.stack[1]

    if mix then
        table.insert(self.stack, mix)
        table.sort(self.stack, function(l, r) return l.priority > r.priority end)

        if current and current ~= self.stack[1] then
            self:Blend()
        elseif not current then
            mix:Apply()
        end

    end
end
```

**逻辑**：
1. 把 mix 加入 stack
2. **按 priority 降序排列**——high priority 自动浮到栈顶
3. 如果新 mix 占据栈顶 → Blend（渐变到新 mix）
4. 如果第一次 PushMix（栈之前空）→ 直接 Apply

##### Apply —— 实际设置音量

```20:24:scripts/mixer.lua
function Mix:Apply()
    for k,v in pairs(self.levels) do
        TheSim:SetSoundVolume(k, v)
    end
end
```

——遍历 levels 表，**调用引擎 binding `TheSim:SetSoundVolume(channel, value)`**。

##### Blend —— 渐变切换

```68:71:scripts/mixer.lua
function Mixer:Blend()
    self.snapshot = self:CreateSnapshot()
    self.fadetimer = 0
end
```

```82:101:scripts/mixer.lua
function Mixer:Update(dt)

    local top = self.stack[1]
    if self.snapshot and top then
        self.fadetimer = self.fadetimer + dt
        local lerp = self.fadetimer / top.fadeintime

        if lerp > 1 then
            self.snapshot = nil
            top:Apply()
        else
            for k,v in pairs(self.snapshot.levels) do
                local lev = easing.linear(self.fadetimer, v, top:GetLevel(k) - v, top.fadeintime)
                TheSim:SetSoundVolume(k, lev)
            end
        end
    end

    self:UpdateFilters(dt)
end
```

**逻辑**：
1. 记录切换前的 snapshot（旧音量）
2. 在 fadeintime 内，用 easing.linear 从旧音量插值到新音量
3. 完成后清掉 snapshot，应用新 mix 的精确音量

——**这就是为什么 vanilla 各种 mix 切换"丝滑"**——所有变化都过 easing。

##### PopMix —— 出栈

```103:116:scripts/mixer.lua
function Mixer:PopMix(mixname)
    local top = self.stack[1]
    for k, v in ipairs(self.stack) do


        if mixname == v.name then
            table.remove(self.stack, k)
            if top ~= self.stack[1] then
                self:Blend()
            end
            break
        end
    end
end
```

**逻辑**：把 mixname 从 stack 移除——如果它是栈顶 → 重新 blend 到新栈顶。

#### vanilla 实战：death mix

```952:954:scripts/prefabs/player_common.lua
            TheMixer:PushMix("death")
        else
            TheMixer:PopMix("death")
```

**触发时机**：玩家死亡 → PushMix("death")。

死亡 mix 的配置：

```86:98:scripts/mixes.lua
TheMixer:AddNewMix("death", 1, 6,
{
    [amb] = .2,
    [cloud] = .2,
    [music] = 0,
    [voice] = 1,
    [movement] = .8,
    [creature] = .8,
    [player] = 1,
    [HUD] = 1,
    [sfx] = .8,
    [slurp] = .8,
})
```

**含义**：
- `music = 0` —— BGM 静音（凸显死亡氛围）
- `amb = 0.2` / `cloud = 0.2` —— 环境音减弱
- `voice = 1` / `player = 1` / `HUD = 1` —— 角色对白 / UI 全音量

——**视觉是黑屏 + 听觉是凸显死亡 + 失去战斗音乐 = 强烈的失败感设计**。

#### vanilla 实战：飞行 mix（木腿）

```504:508:scripts/prefabs/woodie.lua
        TheMixer:PushMix("flying")
        ...
        TheMixer:PopMix("flying")
```

```201:213:scripts/mixes.lua
TheMixer:AddNewMix("flying", 2, 3,
{
    [amb] = .4,
    [cloud] = .4,
    [music] = .7,
    [voice] = .2,
    [movement] = .2,
    [creature] = .2,
    [player] = 1,
    [HUD] = 1,
    [sfx] = .2,
    [slurp] = 0,
})
```

**含义**：飞行时所有"地面音"减弱（脚步、怪物、sfx）——但 `cloud` 通道**升高**（云层 / 风声等专属高空音）——player + HUD 全音量保留——这是**音频设计师精心调配的"飞行听感"**。

#### Mod 中如何用 TheMixer？

```lua
-- 1. 注册自定义 mix
TheMixer:AddNewMix("mymod_explosion", 1, 5, {
    [amb] = 0.1,
    [music] = 0,
    [voice] = 1,
    [sfx] = 1,
    -- ...
})

-- 2. 触发场景时 PushMix
local function OnExplosion()
    TheMixer:PushMix("mymod_explosion")
    inst:DoTaskInTime(2, function()
        TheMixer:PopMix("mymod_explosion")  -- 2 秒后恢复
    end)
end
```

**陷阱**：忘了 PopMix → mix 永久驻栈 → 玩家觉得"声音怎么变怪了"——**永远成对调用**。

#### 所有 Mixer API

| API | 用途 |
|---|---|
| `AddNewMix(name, fadetime, priority, levels)` | 注册 |
| `PushMix(name)` | 入栈 |
| `PopMix(name)` | 出栈 |
| `DeleteMix(name)` | 强制移除（不 blend）|
| `GetLevel(channel)` | 读取通道当前音量 |
| `SetLevel(name, level)` | 直接设通道音量（绕过 mix）|
| `SetLowPassFilter(category, cutoff, time)` | 设 lowpass DSP |
| `SetHighPassFilter(category, cutoff, time)` | 设 highpass DSP |
| `ClearLowPassFilter(category, time)` | 清 lowpass |
| `ClearHighPassFilter(category, time)` | 清 highpass |

---

### 15.2.5（进阶）DSP 滤波器——lowpass / highpass + 耳塞帽实战

#### 什么是 DSP 滤波器？

DSP = Digital Signal Processing（数字信号处理）。

| 滤波器 | 作用 | 实战用途 |
|---|---|---|
| **Lowpass（低通）** | 砍高频，保留低频 | "声音变闷"——耳塞帽、远处听 |
| **Highpass（高通）** | 砍低频，保留高频 | "声音变薄"——电话感、广播感 |
| **Bandpass（带通）** | 只保留中间频段 | 收音机感（vanilla 不用）|
| **Reverb（混响）** | 回声 | 山洞 / 大厅（vanilla 间接用）|

#### vanilla 的 DSP API

```224:239:scripts/mixer.lua
function Mixer:SetLowPassFilter(category, cutoff, timetotake)
	timetotake = timetotake or 3

	local startfreq = top_val
	if self.lowpassfilters[category] and self.lowpassfilters[category].freq then
		startfreq = self.lowpassfilters[category].freq
	end

	local freq_entry = {startfreq = startfreq, endfreq = cutoff, freq= startfreq, totaltime = timetotake, currenttime = 0}
	self.lowpassfilters[category] = freq_entry

	if timetotake <= 0 then
		freq_entry.freq = cutoff
		TheSim:SetLowPassFilter(category, cutoff)
	end
end
```

**3 参数**：
- `category` —— 通道类别（"set_sfx/sfx" / "set_ambience" / 等）
- `cutoff` —— 截止频率（Hz）
- `timetotake` —— 渐变时间（秒）

**频率值速查**：
- `top_val = 25000` —— 不滤波（25 kHz 已超过人耳）
- `bottom_val = 0` —— 完全砍掉（无声）
- `cutoff = 750` —— 经典"耳塞帽"截止频率（高频被砍，听起来像隔了一面墙）
- `cutoff = 2000` —— 轻度闷
- `cutoff = 8000` —— 几乎无效果

#### PlayerHearing 组件——耳塞帽实战

`scripts/components/playerhearing.lua` 78 行——**完整实现"装备某物 → 声音变闷"机制**：

```1:20:scripts/components/playerhearing.lua
local DURATION = .5

local DSP =
{
    mufflehat =
    {
        lowdsp =
        {
            ["set_music"] = 750,
            ["set_ambience"] = 750,
            ["set_sfx/set_ambience"] = 750,
            ["set_sfx/movement"] = 750,
            ["set_sfx/creature"] = 750,
            ["set_sfx/player"] = 750,
            ["set_sfx/voice"] = 750,
            ["set_sfx/sfx"] = 750,
        },
        duration = DURATION,
    },
}
```

**配置含义**：
- `mufflehat` —— 标签名（任何带 `mufflehat` tag 的装备都触发）
- `lowdsp` —— 全 8 个声音通道都设 750 Hz lowpass
- `duration = 0.5` —— 0.5 秒渐变

#### PlayerHearing 触发机制

```22:35:scripts/components/playerhearing.lua
local function OnEquipChanged(inst)
    local self = inst.components.playerhearing
    local inventory = inst.replica.inventory
    local dirty = false
    for k, v in pairs(DSP) do
        if self[k] == not inventory:EquipHasTag(k) then
            self[k] = not self[k]
            dirty = true
        end
    end
    if dirty then
        self:UpdateDSPTables()
    end
end
```

**逻辑**：
- 监听 equip / unequip 事件
- 遍历 DSP 配置，检查玩家装备是否带对应 tag（`mufflehat`）
- 状态变化 → UpdateDSPTables → 通过 `pushdsp` / `popdsp` 事件交给底层

#### vanilla 中带 mufflehat 标签的装备

通过 grep 查找——以下装备会让玩家"声音变闷"：
- 黑帽（slurper hat）—— 顶在头上的洞穴生物
- 蜘蛛帽（在被蜘蛛附身的特定情境）

**为什么用这个机制？**
- **沉浸感**：戴 slurper 时玩家听到的世界确实"被遮罩"——沉浸感拉满
- **可玩性**：玩家会在该 vs 不戴之间权衡（戴了 sanity 回血 + 失去清晰听力）

#### Mod 中实现 DSP 滤波

##### 例 1：被冰冻时声音变闷

```lua
-- 在玩家被 freeze 时触发
local function OnFrozen(inst)
    TheMixer:SetLowPassFilter("set_sfx", 1000, 1.0)
end

local function OnThaw(inst)
    TheMixer:ClearLowPassFilter("set_sfx", 1.0)
end

inst:ListenForEvent("freeze", OnFrozen)
inst:ListenForEvent("thaw", OnThaw)
```

##### 例 2：进入"水下"音效（玩家泡在海里时）

```lua
local function OnEnterUnderwater(inst)
    TheMixer:SetLowPassFilter("set_ambience", 500, 1.5)  -- 环境音深度闷
    TheMixer:SetLowPassFilter("set_music", 700, 1.5)
    TheMixer:SetLowPassFilter("set_sfx", 1500, 1.5)
end
```

##### 例 3：电话广播感（高通 + 低通组合）

```lua
TheMixer:SetHighPassFilter("set_sfx/voice", 500, 0.5)  -- 砍低频
TheMixer:SetLowPassFilter("set_sfx/voice", 3000, 0.5)  -- 砍高频
-- 中间 500-3000 Hz 留下 → 像电话听筒
```

#### DSP 滤波器的 Update 处理

```154:182:scripts/mixer.lua
function Mixer:UpdateFilters(dt)
    -- First update the filters
    for k,v in pairs(self.lowpassfilters) do
        if v and v.totaltime and v.currenttime and v.totaltime > 0 then
            v.currenttime = v.currenttime + dt
            if v.currenttime < v.totaltime then
                v.freq = easing.linear(v.currenttime, v.startfreq, v.endfreq - v.startfreq, v.totaltime)
                TheSim:SetLowPassFilter(k, v.freq)
            elseif v.currenttime >= v.totaltime and v.freq and v.endfreq and v.freq ~= v.endfreq then
                -- Clamp
                v.freq = v.endfreq
                TheSim:SetLowPassFilter(k, v.freq)
            end
        end
    end
```

**逻辑**：每帧更新滤波器频率（从 startfreq 渐变到 endfreq），用 easing.linear。

——同样 **每个变化都丝滑过渡**。

---

### 15.2.6（进阶）10 大声音通道的语义

#### 通道是什么？

打开 `scripts/mixes.lua` 第 1-12 行：

```1:12:scripts/mixes.lua
local Mixer = require("mixer")

local amb = "set_ambience/ambience"
local cloud = "set_ambience/cloud"
local music = "set_music/soundtrack"
local voice = "set_sfx/voice"
local movement ="set_sfx/movement"
local creature ="set_sfx/creature"
local player ="set_sfx/player"
local HUD ="set_sfx/HUD"
local sfx ="set_sfx/sfx"
local slurp ="set_sfx/everything_else_muted"
```

**含义**：通道是**声音的分类标签**——每条音效在 FMOD 内被绑到一个或多个通道——通过 Mixer 调通道音量，就能批量调整某类音效。

#### 10 大通道详解

| 通道 | 路径 | 装载的内容 | 用途 |
|---|---|---|---|
| **amb** | `set_ambience/ambience` | 草地虫鸣、海浪、洞穴回声 | 环境氛围（远景）|
| **cloud** | `set_ambience/cloud` | 高空云层、风声、月族大气 | 氛围（特殊场景）|
| **music** | `set_music/soundtrack` | BGM 音乐 | 配乐 |
| **voice** | `set_sfx/voice` | 玩家说话、boss 怒吼 | 角色对白 |
| **movement** | `set_sfx/movement` | 脚步声、奔跑 | 移动 |
| **creature** | `set_sfx/creature` | 蜘蛛 / 蜜蜂 / 怪物的吼叫 | 怪物声 |
| **player** | `set_sfx/player` | 玩家受伤、攻击、动作 | 玩家专属 |
| **HUD** | `set_sfx/HUD` | UI 点击、提示 | 界面 |
| **sfx** | `set_sfx/sfx` | 通用音效（爆炸 / 烟雾 / 物品破坏）| 杂项 |
| **slurp** | `set_sfx/everything_else_muted` | 被吞下时的"包裹音" | 极少用，专为 slurper 设计 |

#### 通道层级关系

```
total volume
   ├── set_ambience/
   │      ├── ambience       (amb)
   │      └── cloud          (cloud)
   ├── set_music/
   │      └── soundtrack     (music)
   └── set_sfx/
          ├── voice          (voice)
          ├── movement       (movement)
          ├── creature       (creature)
          ├── player         (player)
          ├── HUD            (HUD)
          ├── sfx            (sfx)
          └── everything_else_muted  (slurp)
```

——这是 FMOD 工程内的**bus 结构**——3 大主类（amb / music / sfx）下分子类。

#### 实战：通过 mix 制造"梦境感"

```lua
TheMixer:AddNewMix("dream", 2, 5, {
    [amb] = 0.1,           -- 环境音几乎消失
    [cloud] = 1.0,         -- 但云层音放大（增加飘渺感）
    [music] = 0.3,         -- BGM 减弱
    [voice] = 1.0,         -- 玩家声音正常
    [movement] = 0.05,     -- 脚步几乎听不到
    [creature] = 0.3,      -- 怪物声减弱
    [player] = 1.0,        -- 玩家音效正常
    [HUD] = 1.0,           -- UI 不变
    [sfx] = 0.5,           -- 杂音减半
    [slurp] = 0,
})
```

——**调通道比例 = 调氛围**——这是音频设计的精髓。

#### vanilla 11 种 mix 的通道值矩阵

| Mix | amb | cloud | music | voice | move | creat | player | HUD | sfx | slurp |
|---|---|---|---|---|---|---|---|---|---|---|
| normal | 0.8 | 0 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| high | 0.2 | 1 | 0.5 | 0.7 | 0.7 | 0.7 | 0.7 | 1 | 0.7 | 1 |
| start | 0.8 | 0 | 1 | 1 | 1 | 1 | 1 | 0.5 | 1 | 1 |
| serverpause | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 1 | 0 | 0 |
| pause | 0.1 | 0.1 | 0 | 0 | 0 | 0 | 0 | 0.6 | 0 | 0 |
| death | 0.2 | 0.2 | 0 | 1 | 0.8 | 0.8 | 1 | 1 | 0.8 | 0.8 |
| slurp | 0.2 | 0.2 | 0.5 | 0.7 | 0.7 | 0.7 | 0.7 | 1 | 0.7 | **1** |
| lobby | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0.6 | 0 | 0 |
| moonstorm | 1 | 0 | 0.3 | 0.3 | 0.3 | 0.3 | 1 | 1 | 0.3 | 0 |
| flying | 0.4 | 0.4 | 0.7 | 0.2 | 0.2 | 0.2 | 1 | 1 | 0.2 | 0 |
| silence | 0 | 0 | 0.2 | 0 | 0 | 0 | 0 | 0 | 1 | 0 |

**观察**：
- **lobby**：只留 music + 0.6 HUD —— 主菜单的"纯音乐 + 空旷感"
- **moonstorm**：cloud 关掉、player + amb 全开 —— 月风暴是"全风声 + 玩家自身音"
- **silence**：基本全 0，只有 sfx 留着 —— **过场剧情专用**（保留关键音效）
- **slurp**：slurp 通道独占 1 —— "被吞下"时其他全降，slurp 通道升起（替代音）

---

### 15.2.7（老手）服务器 / 客户端 / dedicated 三角色的音频职责

#### 三种角色的声音处理

饥荒联机有三种"实例角色"——它们对音频的处理**完全不同**：

```
┌──────────────────────────────────────────────────────────────┐
│ ▎ 角色 1：本地客户端（Local Client）▎                          │
│ - 玩家亲自坐在前面，看屏幕、听音箱                              │
│ - 渲染所有音效                                                 │
│ - 应用 TheMixer / DSP / 3D pan                                 │
│ - 是最完整的音频实例                                            │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ ▎ 角色 2：服务器主机（Master）▎                                │
│ - 服务器决策（什么时候 spawn 怪物、什么时候播音）                │
│ - 通过 entity 同步告诉所有 client：这里有个 spider，你各自播声音 │
│ - 服务器自己是否播音 = 看是不是 dedicated                       │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ ▎ 角色 3：专用服务器（Dedicated Server）▎                       │
│ - 没有玩家在前面、无音箱、无屏幕                                │
│ - 仍然运行 PlaySound 调用（C++ 层 noop）                        │
│ - 所有音效预测 / 决策正常 → 通过网络同步给 client                │
│ - 但不真的发声                                                 │
└──────────────────────────────────────────────────────────────┘
```

#### vanilla 中的 dedicated 检查

通过 grep 查找 `TheNet:IsDedicated()`——很多关键音频代码都有这个检查：

##### 模式 1：fx 系统的 dedicated 短路

`scripts/prefabs/fx.lua` 内的 MakeFx：
```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddNetwork()
    if not TheNet:IsDedicated() then  -- ← dedicated 不 spawn 真 fx
        inst:DoTaskInTime(0, startfx, inst)
    end
    -- ...
end
```

——**dedicated 不创建真 fx，节省开销**——但 entity（proxy）仍然创建，用于网络同步。

##### 模式 2：postprocess shader 的 dedicated 短路

```458:468:scripts/main.lua
	if not TheNet:IsDedicated() then
		BuildColourCubeShader()
		BuildZoomBlurShader()
		BuildBloomShader()
		BuildDistortShader()
		BuildLunacyShader()
		BuildMoonPulseShader()
		BuildMoonPulseGradingShader()
		BuildModShaders()
		SortAndEnableShaders()
	end
```

——dedicated 不构建任何 shader（与音频架构相同思路）。

#### Master 端 vs Client 端的 PlaySound 应该写在哪？

##### 模式 A：声音由 entity 发出 → 写在 master 段

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddSoundEmitter()
    -- ...

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    -- ★ 持续音 loop 在 master 段启动
    inst.SoundEmitter:PlaySound("dontstarve/AMB/cave/litcave", "amb_loop")
    return inst
end
```

**为什么 master 段？**
- entity 网络同步保证 client 一定知道这个 entity 存在
- client 收到 SoundEmitter 状态 → 自动启动相同 loop
- master / client 各自播一份 —— 但因为是同一个 entity 同步，实际听感正常

##### 模式 B：玩家本地操作触发音效 → 客户端预测

```lua
-- 在 SGwilson_client 里
inst.SoundEmitter:PlaySound("dontstarve/wilson/attack_weapon", nil, nil, true)  -- ★ is_predicted
```

**为什么客户端预测？**
- 玩家点击攻击 → 不能等服务器 50ms 延迟才播声
- 客户端**立即播**，给玩家反馈
- 服务器同步过来时，FMOD 检测到客户端已经播了同一 event → 不重复

##### 模式 C：BGM 挂在 TheFocalPoint 而非 entity

`scripts/components/dynamicmusic.lua` 的关键模式：
```lua
local function StartSoundEmitter()
    if _soundemitter == nil then
        _soundemitter = TheFocalPoint.SoundEmitter
        ...
    end
end
```

——**BGM 不挂世界 entity，而挂 TheFocalPoint（相机焦点 entity）**——这样 BGM **不受 3D pan / 距离衰减影响**——永远"贴耳"播。

详见 15.3 节深入讲解。

#### dedicated server 上的 SoundEmitter 行为

dedicated 上：
- `inst.entity:AddSoundEmitter()` —— 仍然创建，但内部 noop
- `inst.SoundEmitter:PlaySound(...)` —— 仍然调，但**不真的发音**
- `inst.SoundEmitter:KillSound(...)` —— 仍然调，noop

——所以代码"原样运行"——不会因为是 dedicated 而出 nil exception。

**优化机会**：节省 dedicated server 的 CPU——可以加判断：
```lua
if not TheNet:IsDedicated() then
    inst.entity:AddSoundEmitter()
end

-- 后续 PlaySound 必须判空
if inst.SoundEmitter then
    inst.SoundEmitter:PlaySound("...")
end
```

——但 vanilla 大部分代码**不这么做**——因为底层 noop 的开销可以忽略——可读性优先。

#### 真实联机场景的音频流

```
玩家 A 攻击怪物 X 的完整音频流：
─────────────────────────────────────────

1. A 点鼠标
        ↓
2. SGwilson_client 进入 attack 状态
   ├─ A 客户端：立即播 "wilson/attack_weapon"（is_predicted=true）
   └─ 发送 attack 命令给 master
        ↓
3. Master 收到 attack 命令
   ├─ Master server（host 是玩家，会自己播音；dedicated 不播）
   ├─ 处理伤害判定，X.combat:GetAttacked
   │     → 触发 X.SoundEmitter:PlaySound("creature/X/hurt")
   └─ entity X 状态网络同步到所有 client
        ↓
4. 所有 client（包括 A）收到 X 受伤同步
   ├─ A 已经在自己的 client 上播了"wilson/attack_weapon"
   ├─ 所有 client（包括 A）听到 X.SoundEmitter:PlaySound("creature/X/hurt")
   └─ B / C / D 客户端听到 wilson/attack_weapon（通过 A 的网络同步）
        ↓
5. 所有客户端各自播放音效——网络延迟下声音可能有 0-100ms 错位
```

#### 给 mod 的启示

| 场景 | 推荐写法 |
|---|---|
| 持续音（loop）| master 段启动，靠 entity 网络同步 |
| 玩家本地动作（攻击 / 走路）| client 预测（is_predicted=true）|
| 全服广播音（boss 刷新）| 服务器 PushEvent + 所有 client ListenForEvent |
| FX prefab 的音效 | 写在 fx 的 fn 内，靠 dedicated 检测自动跳过 |
| dedicated 不需要的（如 BGM）| 挂 TheFocalPoint + `if not TheNet:IsDedicated() then ... end` |

---

### 15.2.8（老手）网络同步、3D 衰减、性能优化

#### 网络同步：SoundEmitter 状态如何过线？

**关键认知**：SoundEmitter **不通过 net_var 同步状态**——而是**靠 entity replication 间接同步**。

```
master 端：
   inst.SoundEmitter:PlaySound("event", "loop")
        ↓
   entity 状态变化（component dirty）
        ↓
   网络同步（仅 entity 元数据，不同步声音）
        ↓
client 端：
   收到 entity 创建通知
        ↓
   重新执行 client side fn（在 SetPristine 之前）
        ↓
   client 自己 PlaySound 一次（仅 prefab 在 fn 内有此调用时）
```

**注意**：如果你**在 master only 段**（`if not TheWorld.ismastersim then return inst end` 之后）调 PlaySound——**client 不会自动跟着播**。

**vanilla 怎么解决？**——大部分持续音 prefab 把 PlaySound **写在 SetPristine 之前**：

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddSoundEmitter()
    
    -- 这里写 PlaySound → master / client 都执行
    inst.SoundEmitter:PlaySound("dontstarve/AMB/forest", "amb")
    
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end
    -- master 段
    return inst
end
```

#### 3D 距离衰减

FMOD 自动处理——但你可以微调。

##### 自动 3D 衰减原理

FMOD event 在 Studio 工程中预设了：
- **min distance**：之内全音量
- **max distance**：之外完全无声
- **rolloff curve**：衰减曲线（默认 logarithmic）

##### Lua 端如何调整？

**通常不调**——音频设计师在 FMOD Studio 内已经配好。

但你**能做的**：
- `inst.SoundEmitter:SetVolume(name, volume)` —— 整体降音量
- `inst.SoundEmitter:OverrideVolumeMultiplier(value)` —— entity 级倍率
- 通过 entity 移动让相机距离变化 —— FMOD 自动重算 pan

##### 一个例子：远距离音效（如雷声）

vanilla 的**雷声**用一个**全局位置**的 SoundEmitter（接玩家）——而不是接雷击点：

```lua
-- 雷击位置远离玩家时
ThePlayer.SoundEmitter:PlaySound("dontstarve/rain/thunder_close")  -- 而不是接到雷击点
```

——这样无论雷击在哪里，玩家都听到"近距离雷声"——更沉浸（vanilla 的设计选择）。

#### 性能优化

##### Voice 池上限

FMOD 默认 voice 池大约 **32 个 voice**（具体数字看 FMOD 配置）——超过会**自动剔除最旧**。

**症状**：
- 玩家走在密集怪物群中，部分怪物声音"消失了"
- 这是 voice 池被吃满

**优化策略**：
1. **不要在 OnUpdate 内 PlaySound**——每帧 30 次 PlaySound 会瞬间塞满 voice 池
2. **持续音用 KillSound 切换**——而不是一直播新的
3. **远距离 entity 处理 OnEntitySleep**——降低同时播放数（详见 15.1.6）

##### CPU 优化

PlaySound 本身**很快**（< 0.1 ms）——但**频繁调用累积 CPU**：

```lua
-- ✗ 坏代码：每帧都尝试播
function MyComponent:OnUpdate(dt)
    if self.shouldplay then
        self.inst.SoundEmitter:PlaySound("event", "loop")  -- 帧帧调
    end
end

-- ✓ 好代码：用 PlayingSound 检查
function MyComponent:OnUpdate(dt)
    if self.shouldplay and not self.inst.SoundEmitter:PlayingSound("loop") then
        self.inst.SoundEmitter:PlaySound("event", "loop")
    end
end
```

##### 内存优化

100+ 个 .fsb 加起来 100-200 MB——大多 mod 不需要那么多——**只 Asset / Preload 你真用的 fsb**。

```lua
-- ✗ 浪费：把所有 vanilla bank 都注册一遍
Asset("SOUND", "sound/together.fsb"),
Asset("SOUND", "sound/grotto_sfx.fsb"),
-- ... 50+ 个

-- ✓ 精确：只用到的
Asset("SOUNDPACKAGE", "sound/mymod.fev"),
Asset("SOUND", "sound/mymod.fsb"),
-- vanilla 的 fsb 已经在 global.lua 内被注册了，mod 不需要重复
```

#### dedicated server 优化

dedicated 上"伪 PlaySound"虽然 cheap，但**累积起来仍然有开销**——尤其密集怪物 / 大型 mod 场景。

```lua
-- 在自定义 component 内 / SoundEmitter 调用前加判断
if not TheNet:IsDedicated() then
    self.inst.SoundEmitter:PlaySound("...")
end
```

——大多 vanilla 代码不做此优化（因为 noop 开销小）——但你的 mod 可以做（特别是密集 OnUpdate 的场景）。

#### 一个综合性能 checklist

| 场景 | 检查项 |
|---|---|
| 持续音 entity | OnEntitySleep / OnEntityWake 处理了吗？|
| OnUpdate 内 PlaySound | PlayingSound 检查了吗？ |
| 大量同时音 | 用 name 去重 vs 多 instance？|
| 远距离音 | 让 entity 接玩家而非世界点（如雷声）？|
| dedicated 优化 | 关键路径加 `IsDedicated` 短路？|
| 资产 | 只 Asset 自己的 .fev/.fsb，别重复 vanilla？|

---

### 15.2.9（小结）速查表 + 4 行起步代码 + 15.3 预告

#### 文件类型速查

| 文件 | 全称 | 含 | 注册类型 | 用途 |
|---|---|---|---|---|
| `.fev` | FMOD Event Bank | event 路径元数据 | `SOUNDPACKAGE` 或 `PKGREF`（仅 vanilla 主 bank）| 路径目录 |
| `.fsb` | FMOD Sound Bank | 压缩音频流 | `SOUND` 或 `FILE` | 真音频 |

#### Mixer API 速查

| API | 用途 |
|---|---|
| `TheMixer:AddNewMix(name, fadetime, priority, levels)` | 注册 mix |
| `TheMixer:PushMix(name)` | 入栈（自动按 priority 排序）|
| `TheMixer:PopMix(name)` | 出栈 |
| `TheMixer:DeleteMix(name)` | 强制删除（不 blend）|
| `TheMixer:SetLevel(channel, value)` | 直接调通道音量 |
| `TheMixer:SetLowPassFilter(category, cutoff, time)` | 应用 lowpass |
| `TheMixer:SetHighPassFilter(category, cutoff, time)` | 应用 highpass |
| `TheMixer:ClearLowPassFilter(category, time)` | 清 lowpass |

#### 10 大通道速查

| 通道 | 用途 |
|---|---|
| amb | 环境音 |
| cloud | 云层 / 高空 |
| music | BGM |
| voice | 角色对白 |
| movement | 脚步 |
| creature | 怪物声 |
| player | 玩家专属 |
| HUD | UI |
| sfx | 杂项 |
| slurp | slurper 专用 |

#### 11 大 mix 速查

| mix | priority | 触发时机 |
|---|---|---|
| start | 0 | 启动 |
| normal | 1 | 默认 |
| slurp | 1 | 被吞 |
| supernova_charging | 2 | 月神蓄力 |
| serverpause | 2147483647 | 服务器暂停 |
| high | 3 | 高强度 |
| flying | 3 | 木腿飞 |
| pause | 4 | 暂停 |
| death | 6 | 死亡 |
| moonstorm | 8 | 月风暴 |
| lobby | 8 | 大厅 |
| silence | 8 | 静音 |

#### 4 行起步代码

##### A：mod 注册自定义音效

```lua
-- modmain.lua
Assets = {
    Asset("SOUNDPACKAGE", "sound/mymod.fev"),
    Asset("SOUND", "sound/mymod.fsb"),
}
PreloadSoundList({"mymod.fev", "mymod.fsb"})
```

##### B：自定义 mix（boss 战）

```lua
-- modmain.lua
TheMixer:AddNewMix("mymod_boss_battle", 1.5, 5, {
    ["set_ambience/ambience"] = 0.3,
    ["set_ambience/cloud"] = 0,
    ["set_music/soundtrack"] = 1,
    ["set_sfx/voice"] = 1,
    ["set_sfx/movement"] = 0.7,
    ["set_sfx/creature"] = 1,
    ["set_sfx/player"] = 1,
    ["set_sfx/HUD"] = 1,
    ["set_sfx/sfx"] = 0.8,
    ["set_sfx/everything_else_muted"] = 0,
})

-- prefab 内
TheMixer:PushMix("mymod_boss_battle")
inst:DoTaskInTime(60, function() TheMixer:PopMix("mymod_boss_battle") end)
```

##### C：DSP 滤波（被冰冻闷音）

```lua
-- 触发时机：onfreeze
TheMixer:SetLowPassFilter("set_sfx", 1000, 0.5)
-- 解除时机：onthaw
TheMixer:ClearLowPassFilter("set_sfx", 0.5)
```

#### 三个设计原则

1. **音效文件资产 + 预加载缺一不可** —— Asset + PreloadSoundList 才能保证 PlaySound 不卡顿。
2. **Mix 是"全游戏调度"，SoundEmitter 是"单 entity 操作"** —— 别用 Mixer 调单声音，别用 SoundEmitter 改全局氛围。
3. **dedicated 上代码运行但不发声** —— mod 写音频代码无需为 dedicated 单独分支，但密集场景可加 `IsDedicated` 短路省 CPU。

---

> **下一节预告**：15.3 节我们将进入 **环境音、音乐状态与动态切换（dynamicmusic.lua）**——15.1 / 15.2 讲了"如何播 + 架构"——15.3 会讲**饥荒怎么决定"什么时候播什么音乐"**——`scripts/components/ambientsound.lua` 怎么按地块切环境音、`scripts/components/dynamicmusic.lua` 怎么根据玩家状态切 BGM、季节 / 战斗 / 探索 / 死亡的音乐过渡逻辑——15.3 是音频系统的**应用层**。

---


## 15.3 环境音、音乐状态与动态切换（dynamicmusic.lua）

### 本节导读

> **一句话定位**：15.1 / 15.2 讲了**怎么播 + 架构**——15.3 讲**饥荒怎么决定"什么时候播什么音乐"**——`scripts/components/ambientsound.lua` 按地块切环境音、`scripts/components/dynamicmusic.lua` 按玩家状态切 BGM——15.3 是音频系统的**应用层 / 决策层**。

#### 一段宏观先讲清楚：饥荒音乐的"3 大引擎"

```
┌──────────────────────────────────────────────────────────────────┐
│  ▎ 引擎 1：AmbientSound（环境音）▎                                 │
│   - 挂在 World entity 上                                           │
│   - 扫描玩家周围 11×11 个 tile（地块）                              │
│   - 根据地块类型播不同环境音（草地虫鸣、海浪、洞穴回响）             │
│   - 自动跟随玩家移动 / 季节 / 雨天 / 时间 / 精神变化                 │
└──────────────────────────────────────────────────────────────────┘
                             │
                             │ 玩家走到哪、听到哪
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  ▎ 引擎 2：DynamicMusic（动态音乐）▎                                │
│   - 挂在 World entity 上（仅 forest / cave）                       │
│   - 监听 21 个事件（buildsuccess、attacked、triggeredevent 等）     │
│   - 根据玩家行为切换 4 大音乐通道：busy / danger / pirates / stinger│
│   - BGM 挂在 TheFocalPoint.SoundEmitter（贴耳播）                   │
└──────────────────────────────────────────────────────────────────┘
                             │
                             │ 玩家在干啥、刷什么 boss
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  ▎ 引擎 3：TheMixer（混音器）▎                                     │
│   - 全局通道音量调度（15.2.4 详讲）                                  │
│   - 死亡 / 暂停 / 飞行等场景下静音 / 凸显                            │
│   - 与上面两个引擎协同：amb 通道由引擎 1 喂、music 通道由引擎 2 喂   │
└──────────────────────────────────────────────────────────────────┘
                             │
                             │ 哪些音可听见、音量多少
                             ▼
                       玩家最终听到的音乐
```

#### 15.3 节回答的 5 个核心问题

```
Q1: ──── 玩家走到森林、海边、洞穴时环境音是怎么自动切的？
        ↓ 答：AmbientSound 扫描 11×11 tile，按 AMBIENT_SOUNDS 表混音

Q2: ──── 为什么 BGM 不挂在 entity 上而是 TheFocalPoint？
        ↓ 答：避免 3D 衰减、跟相机焦点（玩家本人）走

Q3: ──── 玩家"砍树 → 战斗 → 生火 → 入夜" 整套音乐如何衔接？
        ↓ 答：4 个状态通道（busy/danger/pirates/stinger）+ 优先级互斥

Q4: ──── boss 战的"专属音乐"是怎么触发的？
        ↓ 答：boss 的 prefab 在出现时 PushEvent("triggeredevent", {name="kraken"})

Q5: ──── 我的 mod boss 想要专属 BGM 怎么做？
        ↓ 答：定义自己的 TRIGGERED_DANGER_MUSIC 项 + 在 boss 出现时 PushEvent
```

#### 你将看到 4 份核心源码

| 文件 | 行数 | 核心内容 |
|---|---|---|
| `scripts/components/dynamicmusic.lua` | 876 行 | **BGM 状态机核心** —— 4 通道、20+ BUSYTHEMES、22 boss TRIGGERED_DANGER_MUSIC |
| `scripts/components/ambientsound.lua` | 499 行 | **环境音核心** —— 地块扫描、季节 / 雨天 / 时间切换、精神音 |
| `scripts/prefabs/forest.lua` 第 621-622 行 | 2 行 | 在 Forest world 上挂载 dynamicmusic + ambientsound |
| `scripts/prefabs/cave.lua` 第 327-328 行 | 2 行 | 在 Cave world 上挂载 dynamicmusic + ambientsound |

#### 本节学习路径

```
15.3.1 新手  ──────  TheFocalPoint —— BGM 的"耳朵"
15.3.2 新手  ──────  4 大音乐通道：busy / danger / pirates / stinger
15.3.3 新手  ──────  AmbientSound —— 按地块切环境音
                       ↓
15.3.4 进阶  ──────  DynamicMusic 完整状态机（busy + danger 互斥逻辑）
15.3.5 进阶  ──────  季节 / 时间 / 区域驱动音乐切换
15.3.6 进阶  ──────  TriggeredDanger —— boss 战专属音乐
                       ↓
15.3.7 老手  ──────  21 事件驱动入口完整剖析
15.3.8 老手  ──────  自定义场景音乐（mod 实战 + 完整模板）
15.3.9 小结  ──────  速查表 + 4 行起步代码 + 15.4 预告
```

**给三类读者的承诺**：
- **新手**：你将真正理解为什么"砍木头时背景突然出现轻快的工作音乐 → 蜘蛛靠近时音乐转危险 → 杀完蜘蛛后 10 秒音乐才退"。
- **进阶**：你将完整掌握 DynamicMusic 的 4 通道互斥栈、20+ 个 BUSYTHEMES 切换条件、TRIGGERED_DANGER_MUSIC 的多段播放机制。
- **老手**：你将能给自己的 mod boss 加一段"专属 epic 音乐"——并在地块上加自定义环境音——做出"听感连贯"的高质量 mod。

---

### 15.3.1（新手）TheFocalPoint —— BGM 的"耳朵"

#### 一个根本问题：BGM 挂在哪个 entity？

15.1 节告诉你**音效需要 entity.SoundEmitter** —— 那 BGM 应该挂在哪？

**错误答案 1**：挂在玩家 entity 上。
- 问题：玩家移动 → 3D 衰减失真 → BGM 听起来"飘忽"

**错误答案 2**：挂在世界中央 (0,0,0) 的某个虚拟 entity。
- 问题：玩家远离 (0,0,0) → BGM 渐弱（FMOD 自动 3D 衰减）→ 不能稳定播放

**vanilla 的正确答案**：挂在 `TheFocalPoint`。

#### TheFocalPoint 是什么？

**TheFocalPoint** 是饥荒的**相机焦点 entity**——它的位置始终跟着**当前激活玩家的相机焦点**（通常就是玩家本人）。

由于 SoundEmitter 是 3D 的——但 TheFocalPoint 永远在"听众"的位置——所以挂在它上面的音效**永远 0 距离**——不衰减、纯立体声 mix——这就是 BGM 的"耳朵"。

#### dynamicmusic 怎么用 TheFocalPoint？

打开 `scripts/components/dynamicmusic.lua` 第 800-810 行：

```800:810:scripts/components/dynamicmusic.lua
local function StartSoundEmitter()
    if _soundemitter == nil then
        _soundemitter = TheFocalPoint.SoundEmitter
        _extendtime = 0
        if not _iscave then
            _isday = inst.state.isday
            inst:WatchWorldState("phase", OnPhase)
            inst:WatchWorldState("season", OnSeason)
        end
    end
end
```

**关键一行**：`_soundemitter = TheFocalPoint.SoundEmitter` —— **整个 dynamicmusic 组件用的就是这个 emitter**。

后续所有 PlaySound / KillSound / SetParameter 都打到 TheFocalPoint 上：

```304:304:scripts/components/dynamicmusic.lua
                    _soundemitter:PlaySound("dontstarve/music/music_work_cave", "busy")
```

**给 mod 的启示**：
- 你的 mod BGM **应该**挂 `TheFocalPoint.SoundEmitter`
- 普通音效（短音 / 怪物音）挂自己的 `inst.SoundEmitter`
- 通用规则：**贴耳的连续音用 TheFocalPoint，世界里的音用 entity**

#### TheFocalPoint 也用于其他"贴耳"场景

vanilla 中除了 BGM，还有其他"贴耳"场景用 TheFocalPoint：
- 玩家精神低时的耳鸣（`dontstarve/sanity/sanity`）
- 月光精神（`turnoftides/sanity/lunacy_LP`）
- 部分 UI 提示音（虽然多数用 TheFrontEnd:GetSound()）
- 海盗逼近的"脚步逐近音"（`monkeyisland/warning_music/warning_combo`）

#### 在哪里 dynamicmusic 被挂载？

```621:622:scripts/prefabs/forest.lua
        inst:AddComponent("dynamicmusic")
        inst:AddComponent("ambientsound")
```

```327:328:scripts/prefabs/cave.lua
        inst:AddComponent("dynamicmusic")
        inst:AddComponent("ambientsound")
```

**注意**：
- **DynamicMusic 仅挂在 forest 和 cave** —— quagmire / lavaarena 等特殊 world 不挂
- **AmbientSound 挂在 forest / cave / quagmire / lavaarena** —— 多种世界都需要环境音

——这意味着**只有森林 / 洞穴世界有"动态战斗音乐"**——熔炉模式没有。

#### 玩家激活 / 取消激活时 dynamicmusic 的反应

```831:849:scripts/components/dynamicmusic.lua
local function OnPlayerActivated(inst, player)
    if _activatedplayer == player then
        return
    elseif _activatedplayer ~= nil and _activatedplayer.entity:IsValid() then
        StopPlayerListeners(_activatedplayer)
    end
    _activatedplayer = player
    StopSoundEmitter()
    StartSoundEmitter()
    StartPlayerListeners(player)
end

local function OnPlayerDeactivated(inst, player)
    StopPlayerListeners(player)
    if player == _activatedplayer then
        _activatedplayer = nil
        StopSoundEmitter()
    end
end
```

**逻辑**：
- 玩家加入 / 切换主角时 → StartSoundEmitter → 拿 TheFocalPoint.SoundEmitter
- 玩家离开 / 退出时 → StopSoundEmitter → KillSound + 释放引用

——**TheFocalPoint 不变，但 _soundemitter 引用会随玩家激活状态启停**。

---

### 15.3.2（新手）4 大音乐通道——busy / danger / pirates / stinger

#### 4 大通道概览

打开 `scripts/components/dynamicmusic.lua` 全文搜索 `_soundemitter:PlaySound`——你会发现 BGM 用了 **4 个 name handle**：

| 通道（name） | 用途 | 优先级 | 持续时间 |
|---|---|---|---|
| **`busy`** | 砍树 / 工作 / 探索 / 烹饪 等"工作"行为 | 低 | 15-30 秒 |
| **`danger`** | 战斗 / 受攻击 / boss 战 | 中 | 10 秒 |
| **`pirates`** | 海盗逼近警示 | 中（与 danger 互斥）| 持续到海盗远离 |
| **stinger（无 name）** | 短音过场（黎明 / 黄昏 / 失智 / 顿悟）| 短 | 自然结束 |

#### 通道 1：busy（工作音乐）

**什么时候播？**——玩家进行**非战斗动作**时：
- `buildsuccess` —— 成功合成物品
- `gotnewitem` —— 获取新物品
- `performaction` —— 执行动作（砍 / 挖 / 锤等）

**核心代码**（StartBusy 简化版）：

```300:312:scripts/components/dynamicmusic.lua
        if _iscave then
            if IsInRuins(player) then
                if _busytheme ~= BUSYTHEMES.RUINS then
                    _soundemitter:KillSound("busy")
                    _soundemitter:PlaySound("dontstarve/music/music_work_ruins", "busy")
                end
                _busytheme = BUSYTHEMES.RUINS
            else
                if _busytheme ~= BUSYTHEMES.CAVE then
                    _soundemitter:KillSound("busy")
                    _soundemitter:PlaySound("dontstarve/music/music_work_cave", "busy")
                end
                _busytheme = BUSYTHEMES.CAVE
            end
```

**关键模式**：
- **PlaySound 用 name `"busy"`** —— 同 name 后播自动覆盖前播
- **检查 _busytheme** —— 主题没变就不切（避免重复播）
- **缓存 _busytheme** —— 避免重复 PlaySound 浪费

#### 通道 2：danger（危险音乐）

**什么时候播？**——玩家**进入战斗**时：
- `attacked` —— 玩家被攻击
- `performaction` 中的 `attack` —— 玩家主动攻击（CheckAction）
- 注意：只有"危险目标"才触发——`ShouldPlayDangerMusic` 函数判断

**核心代码**：

```594:610:scripts/components/dynamicmusic.lua
local function StartDanger(player)

    if _dangertask ~= nil then
        _extendtime = GetTime() + 10
    elseif _isenabled then
        local x, y, z = player.Transform:GetWorldPosition()
        local epics = TheSim:FindEntities(x, y, z, 30, EPIC_TAGS, NO_EPIC_TAGS)
        StopBusy()        
        _soundemitter:PlaySound(
            #epics > 0
            and ((IsInRuins(player) and "dontstarve/music/music_epicfight_ruins") or
                (_iscave and "dontstarve/music/music_epicfight_cave") or
                (SEASON_EPICFIGHT_MUSIC[inst.state.season]))
            or ((IsInRuins(player) and "dontstarve/music/music_danger_ruins") or
                (_iscave and "dontstarve/music/music_danger_cave") or
                (SEASON_DANGER_MUSIC[inst.state.season])),
            "danger")
        _dangertask = inst:DoTaskInTime(10, StopDanger, true)
```

**精妙之处**：
- 30 米内有 epic 怪物（has tag `epic`）→ 播 **epicfight** 音乐
- 没 epic → 播普通 **danger** 音乐
- 都按**当前世界 + 季节**选择正确变体
- **触发时立即 StopBusy** —— danger 互斥 busy

#### 通道 3：pirates（海盗音乐）

**什么时候播？**——猴岛附近有海盗逼近时：
- 30 米内有 `pirate` tag entity → 持续监测 → 渐近音

**核心代码**（UpdatePirates 简化）：

```545:564:scripts/components/dynamicmusic.lua
    if not _soundemitter:PlayingSound("pirates") and level > 0 then
      _soundemitter:PlaySound("monkeyisland/warning_music/warning_combo", "pirates")
    end

    local intensity = 0
    if _dangertask then
        intensity = 0.5
        if _hasinspirationbuff then
            intensity = intensity + (0.5 * _hasinspirationbuff)
        end
    end

    _soundemitter:SetParameter("pirates", "intensity", intensity) 

    if level > 0 then
        StopBusy()
         _soundemitter:SetVolume("pirates", level)
    else
        StopPirates()
    end
```

**精妙之处**：
- 用 `intensity` parameter 表示战斗状态 —— 海盗刚逼近 vs 已经在战斗，BGM 内部动态切换
- 用 `level` 控制音量 —— 距离越近音量越大（逐渐响起的紧迫感）

#### 通道 4：stinger（短音过场）

**什么时候播？**——日变化、精神状态变化等**关键事件**的瞬间：
- `dontstarve/music/music_dawn_stinger` —— 黎明
- `dontstarve/music/music_dusk_stinger` —— 黄昏
- `dontstarve/sanity/gonecrazy_stinger` —— 失智
- `dontstarve/sanity/lunacy_stinger` —— 顿悟

**核心代码**（OnPhase）：

```771:794:scripts/components/dynamicmusic.lua
local function OnPhase(inst, phase)
    _isday = phase == "day"
    if _dangertask ~= nil or not _isenabled or IsBusyThemeStageplay() then
        return
    end
    --Don't want to play overlapping stingers
    local time
    if _busytask == nil and _extendtime ~= 0 then
        time = GetTime()
        if time < _extendtime then
            return
        end
    end
    if _isday then
        _soundemitter:PlaySound("dontstarve/music/music_dawn_stinger")
    elseif phase == "dusk" then
        _soundemitter:PlaySound("dontstarve/music/music_dusk_stinger")
    else
        return
    end
    StopBusy()
    --Repurpose this as a delay before stingers or busy can start again
    _extendtime = (time or GetTime()) + 15
end
```

**精妙之处**：
- **不带 name** —— 一次性 stinger，不需要后续控制
- **检查 _dangertask** —— 战斗中不打断（"epic 战斗时黎明不响"）
- **检查 _extendtime** —— 之前刚有 stinger 了，15 秒内不再播

#### 4 通道互斥优先级

```
优先级（高 → 低）：

triggeredevent (boss)  ──── 强制盖一切（StopBusy + StopDanger）
       ↓ ↓ ↓
     danger       ──── 中等优先（盖 busy）
       ↓ ↓
   pirates ↔ busy ──── 同级（互斥，pirates 抢占 busy）
       ↓
   stinger        ──── 最低（被 danger 阻塞，被自身防抖延迟阻塞）
```

——**这套优先级让玩家"听感连贯"**：
- 砍树时听到 busy 工作音乐
- 蜘蛛靠近 → 立即 StopBusy + 播 danger
- 战斗 10 秒后 danger 自然结束 → busy 不会立刻重启（_extendtime 防止"音乐切换太频繁"）

---

### 15.3.3（新手）AmbientSound —— 按地块切环境音

#### 环境音是什么？

**环境音**（Ambient Sound）= 不是某个 entity 发出，而是**整个环境**散发的音：
- 草原的虫鸣
- 海浪声
- 洞穴的滴水回响
- 雨天的雨声

#### AmbientSound 组件的设计哲学

**根本问题**：玩家走到哪、听到哪——但**世界里没有数千个虫子 entity 在发声**——而是用**地块**（tile）触发音效。

**vanilla 的解法**：
1. 玩家位置周围**11×11 个 tile** 各自决定它的音
2. 同种 tile 的音**累加**，决定混音权重
3. 取**前 3 大** sound（MAX_MIX_SOUNDS）混合播放

#### AMBIENT_SOUNDS 表 —— 地块到音效的映射

打开 `scripts/components/ambientsound.lua` 第 29-50 行：

```29:38:scripts/components/ambientsound.lua
local AMBIENT_SOUNDS =
{
    [WORLD_TILES.ROAD] = {sound = "dontstarve/AMB/rocky", wintersound = "dontstarve/AMB/rocky_winter", springsound = "dontstarve/AMB/rocky", summersound = "dontstarve_DLC001/AMB/rocky_summer", rainsound = "dontstarve/AMB/rocky_rain"},
    [WORLD_TILES.ROCKY] = {sound = "dontstarve/AMB/rocky", wintersound = "dontstarve/AMB/rocky_winter", springsound = "dontstarve/AMB/rocky", summersound = "dontstarve_DLC001/AMB/rocky_summer", rainsound = "dontstarve/AMB/rocky_rain"},
    [WORLD_TILES.DIRT] = {sound = "dontstarve/AMB/badland", wintersound = "dontstarve/AMB/badland_winter", springsound = "dontstarve/AMB/badland", summersound = "dontstarve_DLC001/AMB/badland_summer", rainsound = "dontstarve/AMB/badland_rain"},
    [WORLD_TILES.WOODFLOOR] = {sound = "dontstarve/AMB/rocky", wintersound = "dontstarve/AMB/rocky_winter", springsound = "dontstarve/AMB/rocky", summersound = "dontstarve_DLC001/AMB/rocky_summer", rainsound = "dontstarve/AMB/rocky_rain"},
    [WORLD_TILES.SAVANNA] = {sound = "dontstarve/AMB/grassland", wintersound = "dontstarve/AMB/grassland_winter", springsound = "dontstarve/AMB/grassland", summersound = "dontstarve_DLC001/AMB/grassland_summer", rainsound = "dontstarve/AMB/grassland_rain"},
    [WORLD_TILES.GRASS] = {sound = "dontstarve/AMB/meadow", wintersound = "dontstarve/AMB/meadow_winter", springsound = "dontstarve/AMB/meadow", summersound = "dontstarve_DLC001/AMB/meadow_summer", rainsound = "dontstarve/AMB/meadow_rain"},
    [WORLD_TILES.FOREST] = {sound = "dontstarve/AMB/forest", wintersound = "dontstarve/AMB/forest_winter", springsound = "dontstarve/AMB/forest", summersound = "dontstarve_DLC001/AMB/forest_summer", rainsound = "dontstarve/AMB/forest_rain"},
```

**4 维参数**（每个 tile 一组配置）：

| 字段 | 何时播 |
|---|---|
| `sound` | 默认（秋天）|
| `wintersound` | 冬天 |
| `springsound` | 春天 |
| `summersound` | 夏天 |
| `rainsound` | 大雨（precipitationrate > 0.5）|

#### 11×11 地块扫描算法

打开 `scripts/components/ambientsound.lua` 第 17 行：

```17:19:scripts/components/ambientsound.lua
local HALF_TILES = 5
local MAX_MIX_SOUNDS = 3
local WAVE_VOLUME_SCALE = 3 / (HALF_TILES * HALF_TILES * 8)
```

**HALF_TILES = 5** → 实际扫描 **(2*5+1)² = 121 个 tile** （11×11 网格）。

##### 扫描逻辑

```349:382:scripts/components/ambientsound.lua
        for x1 = -HALF_TILES, HALF_TILES do
            for y1 = -HALF_TILES, HALF_TILES do
                local tile = _map:GetTile(x + x1, y + y1)
                if TileGroupManager:IsImpassableTile(tile) then
                    wavecount = wavecount + 1
                elseif tile ~= nil then
                    tile = _tileoverrides[tile] or tile
                    local soundgroup = AMBIENT_SOUNDS[tile]
                    if soundgroup ~= nil then
                        local sound =
                                (_rainmix and _heavyrainmix and soundgroup.rainsound) or
                                (_seasonmix and soundgroup[SEASON_SOUND_KEY[_seasonmix]]) or
                                soundgroup.sound
                        local counter = soundmixcounters[sound]
                        local increment = 1
                        if sound == AMBIENT_SOUNDS.ABYSS.sound then
                            increment = 0.5
                        end

                        if counter == nil then
                            counter = { sound = sound, count = increment }
                            soundmixcounters[sound] = counter
                            table.insert(soundmix, counter)
                        else
                            counter.count = counter.count + increment
                        end
                    end
                end
            end
        end

        --Sort by highest count and truncate soundmix to MAX_MIX_SOUNDS
        table.sort(soundmix, SortByCount)
        soundmix[MAX_MIX_SOUNDS + 1] = nil
```

**逻辑**：
1. 遍历 121 个 tile
2. 每个 tile 选音（按"是否大雨 / 季节 / 默认"优先级）
3. **同种音累加 count** —— 玩家在森林中央，121 tile 全是 forest → forest 音 count=121
4. 按 count 降序排，**最多保留前 3 个** sound 同时播

**精妙之处**：
- 玩家走在"森林边缘"——周围 80 个森林 tile + 30 个草原 tile + 10 个海洋 tile —— 同时播 3 种音 —— 营造"边境感"
- 玩家走深森林 —— 全是 forest tile —— 单独 forest 音放最大

#### 实际播放代码

```430:455:scripts/components/ambientsound.lua
    if soundvolumes ~= nil then
        for k, v in pairs(_soundvolumes) do
            if soundvolumes[k] == nil then
                inst.SoundEmitter:KillSound(k)
            end
        end
        for k, v in pairs(soundvolumes) do
            local oldvol = _soundvolumes[k]
            local newvol = v / totalsoundcount
            if oldvol == nil then
                inst.SoundEmitter:PlaySound(k, k)
                inst.SoundEmitter:SetParameter(k, "daytime", GetDayTimeParam(k))
                inst.SoundEmitter:SetVolume(k, newvol * ambientvolume)
            elseif oldvol ~= newvol then
                inst.SoundEmitter:SetVolume(k, newvol * ambientvolume)
            end
            soundvolumes[k] = newvol
        end
        _soundvolumes = soundvolumes
        _ambientvolume = ambientvolume
    elseif _ambientvolume ~= ambientvolume then
        for k, v in pairs(_soundvolumes) do
            inst.SoundEmitter:SetVolume(k, v * ambientvolume)
        end
        _ambientvolume = ambientvolume
    end
```

**5 步**：
1. 旧音里**新混音不需要的** → KillSound
2. 新音里**旧混音没有的** → PlaySound + SetParameter("daytime") + SetVolume
3. 新旧都有的 → SetVolume 调音量
4. 缓存 `_soundvolumes` —— 下次只 diff 不全启停
5. `daytime` parameter —— 让 FMOD 在 event 内部切换"白天虫鸣 vs 夜晚虫鸣"

#### 海浪音的特殊处理

```296:306:scripts/components/ambientsound.lua
local function StartWavesSound()
	inst.SoundEmitter:PlaySound(_wavessound, "waves")
end

local function StopWavesSound()
	inst.SoundEmitter:KillSound("waves")
end

local function SetWavesVolume(volume)
	inst.SoundEmitter:SetVolume("waves", volume)
end
```

**逻辑**：
- 121 tile 中**不可通过的（impassable）tile** = 海洋 → wavecount 累加
- wavecount × WAVE_VOLUME_SCALE = waves 音量
- **特殊 name "waves"** —— 与地块音并存

#### 精神 / 顿悟音的特殊处理

```280:294:scripts/components/ambientsound.lua
local function StartEnlightenmentSound()
    inst.SoundEmitter:PlaySound(ENLIGHTENMENT_SOUND, "ENLIGHT")
end

local function SetEnlightenment(sanity)
    inst.SoundEmitter:SetParameter("ENLIGHT", "sanity", sanity)
end

local function StartSanitySound()
	inst.SoundEmitter:PlaySound(SANITY_SOUND, "SANITY")
end

local function SetSanity(sanity)
	inst.SoundEmitter:SetParameter("SANITY", "sanity", sanity)
end
```

**逻辑**：
- AmbientSound 启动时**就播 SANITY 和 ENLIGHT 两个 loop**——但初始 parameter `sanity = 0` → 听不见
- 玩家精神低 → 提升 sanity parameter → FMOD 内部混入耳鸣
- **永远在播但不一定听得见**——这是数据驱动的精妙设计

#### 地块切换的渐变机制

`OnUpdate(dt)` 第 341 行：

```341:341:scripts/components/ambientsound.lua
    elseif _lastplayerpos == nil or player:GetDistanceSqToPoint(_lastplayerpos:Get()) >= 16 then
```

**逻辑**：玩家移动**距离² ≥ 16**（即 4 米以上）→ 才重新扫描 121 tile + 重新混音。

——避免每帧都扫描 121 tile 浪费 CPU——只在玩家**实际移动到新区域**时刷新。

---

### 15.3.4（进阶）DynamicMusic 完整状态机

#### DynamicMusic 的私有状态变量

打开 `scripts/components/dynamicmusic.lua` 第 232-246 行：

```232:246:scripts/components/dynamicmusic.lua
--Private
local _iscave = inst:HasTag("cave")
local _isenabled = true
local _busytask = nil
local _dangertask = nil
local _pirates_near = nil
local _triggeredlevel = nil
local _isday = nil
local _isbusyruins = nil
local _busytheme = nil
local _extendtime = nil
local _soundemitter = nil
local _activatedplayer = nil --cached for activation/deactivation only, NOT for logic use
local _hasinspirationbuff = nil
```

**8 大状态变量**：

| 变量 | 用途 |
|---|---|
| `_iscave` | 是否在洞穴世界（影响音乐选择）|
| `_isenabled` | 总开关（其他组件可通过 `enabledynamicmusic` 事件关闭）|
| `_busytask` | busy 通道的"自动停止 task"|
| `_dangertask` | danger 通道的"自动停止 task"|
| `_pirates_near` | pirates 通道的周期 task |
| `_triggeredlevel` | TriggeredDanger 当前级别（boss 战阶段）|
| `_isday` | 是否白天 |
| `_busytheme` | 当前 busy 通道主题（CAVE / FOREST / OCEAN / 等）|
| `_extendtime` | "防抖时间戳"（避免音乐切换太频繁）|
| `_hasinspirationbuff` | 薇格弗德的鼓舞 buff（影响音乐 parameter）|

#### busy 通道的 4 段生命周期

```
玩家空闲                         玩家砍树                         15s 超时                           回到空闲
───────────────────────────────────────────────────────────────────────────────────────────────
                                       │                                   │
                                       │ "performaction"                   │
                                       ▼                                   ▼
                                   StartBusy()                       StopBusy(true)
                                       │                                   │
                                       │ ① _busytask = DoTaskInTime(15)    │ ① 检查 _extendtime
                                       │ ② SoundEmitter:PlaySound("busy")  │ ② 还在 extend 区间 →
                                       │ ③ SetParameter("busy", "intensity", 1)   │   重新 DoTaskInTime
                                       │                                   │ ③ 否则 SetParameter(
                                       │                                   │     "intensity", 0)
                                       │                                   │   等 FMOD 自然 fade
                                       │                                   │   （不调 KillSound!）
```

**精妙之处 1**：**fade out 用 SetParameter("intensity", 0) 而非 KillSound**。

```282:287:scripts/components/dynamicmusic.lua
		if IsBusyThemeStageplay() then
			_soundemitter:KillSound("busy")
			_busytheme = nil
		else
			_soundemitter:SetParameter("busy", "intensity", 0)
		end
```

——FMOD event 内部的 `intensity` parameter 控制音量包络——`intensity=0` → FMOD 内部 fade 到无声但**event 仍在播**——下次 StartBusy 时**直接 SetParameter("intensity", 1) 就能秒回**——避免重新 PlaySound 的硬切换。

**精妙之处 2**：`_extendtime` 防抖延长。

```296:298:scripts/components/dynamicmusic.lua
    elseif _busytask ~= nil then
        _extendtime = GetTime() + 15
    elseif _dangertask == nil and _pirates_near == nil and (_extendtime == 0 or GetTime() >= _extendtime) and _isenabled then
```

——玩家**连续砍树** → 每砍一刀重置 _extendtime——音乐**持续延长 15 秒**——避免短促的"砍-停-砍-停"。

#### danger 通道的生命周期

```
玩家空闲                              被攻击                         10s 后                           回到空闲
───────────────────────────────────────────────────────────────────────────────────────────────
                                        │                                    │
                                        │ "attacked" event                   │
                                        ▼                                    ▼
                                   StartDanger()                       StopDanger(true)
                                        │                                    │
                                        │ ① 30 米内查 epic                   │ ① 检查 _extendtime
                                        │ ② StopBusy()                       │   还在 extend → 重新 DoTaskInTime
                                        │ ③ PlaySound(epic? danger?, "danger") │ ② KillSound("danger")
                                        │ ④ _dangertask = DoTaskInTime(10)   │ ③ _dangertask = nil
```

**关键差异**：
- **danger 用 KillSound 直接停**（不用 SetParameter("intensity", 0)）
- **danger 时长固定 10 秒**（busy 是 15 秒）
- **danger 触发立即 StopBusy**（互斥优先级）

#### 互斥逻辑

`StartBusy` 第 298 行：
```lua
elseif _dangertask == nil and _pirates_near == nil and (_extendtime == 0 or GetTime() >= _extendtime) and _isenabled then
```

——**只有在 danger 不活跃 + pirates 不活跃 + _extendtime 已过 + 总开关开启**时，才启动 busy。

`StartDanger` 第 596 行：
```lua
if _dangertask ~= nil then
    _extendtime = GetTime() + 10
elseif _isenabled then
    -- ... 进入新 danger
```

——已经在 danger 中 → 延长 _extendtime 而不重启播。

#### CheckAction —— 决定 busy / danger 入口

```662:673:scripts/components/dynamicmusic.lua
local function CheckAction(player)
    if player:HasTag("attack") then
        local target = player.replica.combat:GetTarget()
		if target and ShouldPlayDangerMusic(player, target) then
			StartDanger(player)
			return
        end
    end
    if player:HasTag("working") then
        StartBusy(player)
    end
end
```

**精妙之处**：
- 玩家在 `performaction` 时此函数被调用
- **优先判断 attack**：玩家正在攻击 + 目标值得"危险音乐" → StartDanger
- **fallback working**：纯工作（砍 / 挖 / 锤）→ StartBusy
- **HasTag**——这是 SG 的状态 tag——玩家进入 attack 状态时有 "attack" tag，进入 chop/mine/hammer 时有 "working" tag

#### ShouldPlayDangerMusic —— 危险目标过滤

```1:26:scripts/components/dynamicmusic.lua
--global
function ShouldPlayDangerMusic(player, target)
	if target.replica.combat == nil then
		return false
	elseif (target:HasTag("prey") and not target:HasTag("hostile")) then
		return false
	elseif target:HasAnyTag(
		"bird",
		"butterfly",
		"shadow",
		"shadowchesspiece",
		"noepicmusic",
		"thorny",
		"smashable",
		"wall",
		"engineering",
		"smoldering",
		"veggie")
	then
		return false
	elseif target:HasAnyTag("shadowminion", "abigail", "possessedbody") then
		local follower = target.replica.follower
		return not (follower and follower:GetLeader() == player)
	end
	return true
end
```

**含义**：以下情况**不播 danger 音乐**：
- 目标无 combat replica（不是战斗 entity）
- 目标是 prey（兽类）且不 hostile（猎物不会反击）
- 目标是 bird / butterfly / wall / engineering 等"非战斗目标"
- 目标是阴影怪、影子棋（玩家自己召唤的不算威胁）
- 目标是召唤物 + 是玩家自己的随从

**精妙之处**：玩家**砍蝴蝶不播 danger** —— 视觉上是"攻击" 但听感上是"采集" —— 设计师对玩家心理的精准把握。

---

### 15.3.5（进阶）季节 / 时间 / 区域驱动音乐切换

#### 季节驱动 —— 4 套季节音乐变体

`dynamicmusic.lua` 第 38-60 行：

```38:60:scripts/components/dynamicmusic.lua
local SEASON_BUSY_MUSIC =
{
    autumn = "dontstarve/music/music_work",
    winter = "dontstarve/music/music_work_winter",
    spring = "dontstarve_DLC001/music/music_work_spring",
    summer = "dontstarve_DLC001/music/music_work_summer",
}

local SEASON_EPICFIGHT_MUSIC =
{
    autumn = "dontstarve/music/music_epicfight",
    winter = "dontstarve/music/music_epicfight_winter",
    spring = "dontstarve_DLC001/music/music_epicfight_spring",
    summer = "dontstarve_DLC001/music/music_epicfight_summer",
}

local SEASON_DANGER_MUSIC =
{
    autumn = "dontstarve/music/music_danger",
    winter = "dontstarve/music/music_danger_winter",
    spring = "dontstarve_DLC001/music/music_danger_spring",
    summer = "dontstarve_DLC001/music/music_danger_summer",
}
```

**3 类音乐** × **4 个季节** = **12 种变体** —— vanilla 给森林世界准备的。

##### 季节切换的代码

```324:324:scripts/components/dynamicmusic.lua
                    _soundemitter:PlaySound(SEASON_BUSY_MUSIC[inst.state.season], "busy")
```

——**简单的 table lookup** —— `inst.state.season` 是 `TheWorld.state.season` 同步过来的状态。

##### OnSeason —— 季节切换监听

```796:798:scripts/components/dynamicmusic.lua
local function OnSeason()
    _busytheme = nil
end
```

——季节变化时**清空 _busytheme** —— 这样下次 StartBusy 时会**强制重新 PlaySound**（即便玩家在同一片森林，也会切到新季节版本）。

#### 时间驱动 —— 黎明 / 黄昏 stinger

```771:794:scripts/components/dynamicmusic.lua
local function OnPhase(inst, phase)
    _isday = phase == "day"
    if _dangertask ~= nil or not _isenabled or IsBusyThemeStageplay() then
        return
    end
    --Don't want to play overlapping stingers
    local time
    if _busytask == nil and _extendtime ~= 0 then
        time = GetTime()
        if time < _extendtime then
            return
        end
    end
    if _isday then
        _soundemitter:PlaySound("dontstarve/music/music_dawn_stinger")
    elseif phase == "dusk" then
        _soundemitter:PlaySound("dontstarve/music/music_dusk_stinger")
    else
        return
    end
    StopBusy()
    --Repurpose this as a delay before stingers or busy can start again
    _extendtime = (time or GetTime()) + 15
end
```

**关键设计**：
- **三段判断**（day / dusk / night）—— **night 不播 stinger**（夜晚直接进入"安静"）
- **战斗中不打断**（_dangertask ~= nil 时直接 return）
- **stinger 触发后 _extendtime + 15 秒** —— "刚 stinger 不要立刻 busy"

#### 区域驱动 —— 洞穴 / 远古废墟 / 月岛

```300:328:scripts/components/dynamicmusic.lua
        if _iscave then
            if IsInRuins(player) then
                if _busytheme ~= BUSYTHEMES.RUINS then
                    _soundemitter:KillSound("busy")
                    _soundemitter:PlaySound("dontstarve/music/music_work_ruins", "busy")
                end
                _busytheme = BUSYTHEMES.RUINS
            else
                if _busytheme ~= BUSYTHEMES.CAVE then
                    _soundemitter:KillSound("busy")
                    _soundemitter:PlaySound("dontstarve/music/music_work_cave", "busy")
                end
                _busytheme = BUSYTHEMES.CAVE
            end
        else
            if IsOnLunarIsland(player) then
                if _busytheme ~= BUSYTHEMES.LUNARISLAND then
                    _soundemitter:KillSound("busy")
                    _soundemitter:PlaySound("turnoftides/music/working", "busy")
                end
                _busytheme = BUSYTHEMES.LUNARISLAND
            else
                if _busytheme ~= BUSYTHEMES.FOREST then
                    _soundemitter:KillSound("busy")
                    _soundemitter:PlaySound(SEASON_BUSY_MUSIC[inst.state.season], "busy")
                end
                _busytheme = BUSYTHEMES.FOREST
            end
        end
```

**4 大区域 busy 音乐**：

| 区域 | 检测函数 | busy 音乐 |
|---|---|---|
| **洞穴** | `_iscave` | `dontstarve/music/music_work_cave` |
| **远古废墟** | `IsInRuins(player)` | `dontstarve/music/music_work_ruins` |
| **月岛** | `IsOnLunarIsland(player)` | `turnoftides/music/working` |
| **森林（默认）** | else | SEASON_BUSY_MUSIC（按季节）|

**判断函数**：

```252:260:scripts/components/dynamicmusic.lua
local function IsInRuins(player)
    return player.components.areaaware ~= nil
        and player.components.areaaware:CurrentlyInTag("Nightmare")
end

local function IsOnLunarIsland(player)
    return player.components.areaaware ~= nil
        and player.components.areaaware:CurrentlyInTag("lunacyarea")
end
```

——**通过 areaaware 组件查 tag**：地图上的 set piece / 区域被标记了 "Nightmare" / "lunacyarea" tag——玩家踩进去就能切换音乐。

#### danger 也按区域切

```602:610:scripts/components/dynamicmusic.lua
        _soundemitter:PlaySound(
            #epics > 0
            and ((IsInRuins(player) and "dontstarve/music/music_epicfight_ruins") or
                (_iscave and "dontstarve/music/music_epicfight_cave") or
                (SEASON_EPICFIGHT_MUSIC[inst.state.season]))
            or ((IsInRuins(player) and "dontstarve/music/music_danger_ruins") or
                (_iscave and "dontstarve/music/music_danger_cave") or
                (SEASON_DANGER_MUSIC[inst.state.season])),
            "danger")
```

**3 维矩阵**：
- 维度 A：epic（30m 内有 epic）vs danger（普通战斗）
- 维度 B：ruins / cave / forest（区域）
- 维度 C：autumn / winter / spring / summer（季节，仅 forest 时）

**总变体数**：2 × (1 ruins + 1 cave + 4 forest seasons) = **12 种 danger 音乐**——细节令人发指。

---

### 15.3.6（进阶）TriggeredDanger —— boss 战专属音乐

#### 什么是 TriggeredDanger？

普通 danger 是"被动"的——玩家进战斗就播——音乐是**通用的危险音**。

但**boss 战**需要**专属 BGM**——不同 boss 的音乐**完全不同**——这就是 `TRIGGERED_DANGER_MUSIC` 表的作用。

#### TRIGGERED_DANGER_MUSIC 表

```62:128:scripts/components/dynamicmusic.lua
local TRIGGERED_DANGER_MUSIC =
{

    wagstaff_experiment =
    {
        "moonstorm/characters/wagstaff/music_wagstaff_experiment",
    },

    crabking =
    {
        "dontstarve/music/music_epicfight_crabking",
    },

    malbatross =
    {
        "saltydog/music/malbatross",
    },

    moonbase =
    {
        "dontstarve/music/music_epicfight_moonbase",
        "dontstarve/music/music_epicfight_moonbase_b",
    },

    toadstool =
    {
        "dontstarve/music/music_epicfight_toadboss",
    },

    beequeen =
    {
        "dontstarve/music/music_epicfight_4",
    },

    dragonfly =
    {
        "dontstarve/music/music_epicfight_3",
    },

    shadowchess =
    {
        "dontstarve/music/music_epicfight_ruins",
    },

    klaus =
    {
        "dontstarve/music/music_epicfight_5a",
        "",
        "dontstarve/music/music_epicfight_5b",
    },
```

**结构**：每个 boss 有一个 array
- 单元素：单段音乐（蛤蟆王、蜜蜂女王）
- 多元素：**按 level 分段播**（克劳斯多阶段、瓦格 boss 多阶段）
- `""`（空字符串）：**沉默间隔**（中段静默，凸显戏剧效果）

#### TriggeredDanger 触发模式

vanilla 中 boss prefab 触发 triggeredevent：

```713:713:scripts/prefabs/warg.lua
		ThePlayer:PushEvent("triggeredevent", { name = "gestaltmutant" })
```

——**boss 出现 → 在 player 上 PushEvent**——dynamicmusic 监听这个事件。

#### StartTriggeredDanger 处理

```626:648:scripts/components/dynamicmusic.lua
local function StartTriggeredDanger(player, data)

    local level = math.max(1, math.floor(data ~= nil and data.level or 1))
    if _triggeredlevel == level then
        _extendtime = math.max(_extendtime, GetTime() + (data.duration or 10))
    elseif _isenabled then
        StopBusy()
        StopDanger()

        local music = data ~= nil and TRIGGERED_DANGER_MUSIC[data.name or "default"] or TRIGGERED_DANGER_MUSIC.default
        music = music[level] or music[1]
        if #music > 0 then
            _soundemitter:PlaySound(music, "danger")
            if _hasinspirationbuff then
                _soundemitter:SetParameter("danger", "wathgrithr_intensity", _hasinspirationbuff)
            end
        end

        _dangertask = inst:DoTaskInTime(data.duration or 10, StopDanger, true)
        _triggeredlevel = level
        _extendtime = 0
    end
end
```

**逻辑**：
1. 解析 `data.name`（boss 名）和 `data.level`（boss 阶段）
2. 已经在同 level → 仅延长 _extendtime（防抖）
3. 新触发 → StopBusy + StopDanger + 播专属音乐 + DoTaskInTime
4. **空字符串 ""** → 不播音（multiline 表中的"沉默间隔"）
5. 应用 `wathgrithr_intensity` parameter（薇格弗德的鼓舞 buff）

#### 多阶段 boss 的播放模式

例：klaus 第 106-111 行
```lua
klaus =
{
    "dontstarve/music/music_epicfight_5a",
    "",
    "dontstarve/music/music_epicfight_5b",
},
```

**3 阶段**：
- level 1（首次触发）→ 播 5a（克劳斯出现，第一次形态）
- level 2（boss 切形态）→ 静默（""）→ "音乐戛然而止"的戏剧时刻
- level 3（boss 终极形态）→ 播 5b（终战音乐）

**boss prefab 内部**会在阶段切换时**重新 PushEvent("triggeredevent", { name="klaus", level=2 })**——dynamicmusic 据此切音。

#### vanilla 中触发 triggeredevent 的 boss

通过 grep `PushEvent\("triggeredevent"`：

| boss prefab | 触发 |
|---|---|
| `warg.lua` | `gestaltmutant` |
| `bearger.lua` | `gestaltmutant` |
| `knight.lua` | `knight_yoth` |

——还有更多在其他 boss 的 SG / brain 内（详见 grep 结果）。

#### Mod boss 添加专属音乐

3 步：

##### 步骤 1：在 modmain 中扩展 TRIGGERED_DANGER_MUSIC 表

**实战难点**：`TRIGGERED_DANGER_MUSIC` 是 dynamicmusic.lua 内的 **local**——mod 无法直接 inject。

**解法 A**：**在 boss prefab 内手动播 BGM**（绕过 dynamicmusic）：
```lua
-- mymodboss.lua
local function OnAttacked(inst, data)
    if ThePlayer and ThePlayer == data.attacker then
        TheFocalPoint.SoundEmitter:PlaySound("mymod/boss_music", "danger")
        TheFocalPoint.SoundEmitter:KillSound("busy")  -- 互斥
    end
end
```

**解法 B**：**用现有 BUSYTHEMES 之一**（最实用）：
```lua
-- 让 boss 触发 vanilla 已有的 epic 音乐（如 "default"）
ThePlayer:PushEvent("triggeredevent", { name = "default", duration = 60 })
-- 这会播 default = "dontstarve/music/music_epicfight_ruins"
```

---

### 15.3.7（老手）21 事件驱动入口完整剖析

#### 完整事件入口表

打开 `scripts/components/dynamicmusic.lua` 第 721-744 行：

```721:744:scripts/components/dynamicmusic.lua
local function StartPlayerListeners(player)
    inst:ListenForEvent("buildsuccess", StartBusy, player)
    inst:ListenForEvent("gotnewitem", ExtendBusy, player)
    inst:ListenForEvent("performaction", CheckAction, player)
    inst:ListenForEvent("attacked", OnAttacked, player)
    inst:ListenForEvent("goinsane", OnInsane, player)
    inst:ListenForEvent("goenlightened", OnEnlightened, player)
    inst:ListenForEvent("triggeredevent", StartTriggeredDanger, player)
    inst:ListenForEvent("playboatmusic", StartTriggeredWater, player)
    inst:ListenForEvent("isfeasting", StartTriggeredFeasting, player)
    inst:ListenForEvent("playracemusic", StartRacing, player)
    inst:ListenForEvent("playhermitmusic", StartHermit, player)
    inst:ListenForEvent("playtrainingmusic", StartTraining, player)
    inst:ListenForEvent("playpiratesmusic", StartPirates, player)
    inst:ListenForEvent("playfarmingmusic", StartFarming, player)
    inst:ListenForEvent("hasinspirationbuff", OnHasInspirationBuff, player)
    inst:ListenForEvent("playcarnivalmusic", StartCarnivalMusic, player)
    inst:ListenForEvent("stageplaymusic", StartStageplayMusic, player)
    inst:ListenForEvent("playpillowfightmusic", StartPillowFightMusic, player)
    inst:ListenForEvent("playrideofthevalkyrie", StartRideoftheValkyrieMusic, player)
    inst:ListenForEvent("playboatracemusic", StartBoatRaceMusic, player)
	inst:ListenForEvent("playbalatromusic", StartBalatroMusic, player)
	inst:ListenForEvent("wx_performedspinaction", Wx_CheckSpinAction, player)
end
```

——**21 个事件入口**——这是 dynamicmusic 与"游戏其他系统"的**全部交互点**。

#### 事件入口分类

##### A 类：自然 busy 触发（4 个）

| 事件 | 处理函数 | 触发场景 |
|---|---|---|
| `buildsuccess` | `StartBusy` | 玩家成功合成物品 |
| `gotnewitem` | `ExtendBusy` | 获取新物品（延长 busy 而非启动）|
| `performaction` | `CheckAction` | 任意动作执行（→ busy or danger）|
| `wx_performedspinaction` | `Wx_CheckSpinAction` | WX-78 转动技能 |

##### B 类：自然 danger 触发（1 个）

| 事件 | 处理函数 | 触发场景 |
|---|---|---|
| `attacked` | `OnAttacked` | 玩家被攻击 |

##### C 类：精神 stinger（2 个）

| 事件 | 处理函数 | 触发场景 |
|---|---|---|
| `goinsane` | `OnInsane` | 进入失智模式 |
| `goenlightened` | `OnEnlightened` | 进入顿悟模式 |

##### D 类：boss 战专属 BGM（1 个，最强大）

| 事件 | 处理函数 | 触发场景 |
|---|---|---|
| `triggeredevent` | `StartTriggeredDanger` | boss prefab 主动触发 |

##### E 类：场景 BGM（10+ 个）

| 事件 | 处理函数 | 触发场景 |
|---|---|---|
| `playboatmusic` | `StartTriggeredWater` | 上船 |
| `isfeasting` | `StartTriggeredFeasting` | 冬季盛宴 |
| `playracemusic` | `StartRacing` | 老鼠赛跑 |
| `playhermitmusic` | `StartHermit` | 接近隐士 |
| `playtrainingmusic` | `StartTraining` | 训练塔 |
| `playpiratesmusic` | `StartPirates` | 猴岛海盗逼近 |
| `playfarmingmusic` | `StartFarming` | 种田 |
| `playcarnivalmusic` | `StartCarnivalMusic` | 嘉年华 |
| `stageplaymusic` | `StartStageplayMusic` | 舞台剧 |
| `playpillowfightmusic` | `StartPillowFightMusic` | 枕头战 |
| `playrideofthevalkyrie` | `StartRideoftheValkyrieMusic` | 女武神之骑 |
| `playboatracemusic` | `StartBoatRaceMusic` | 船赛 |
| `playbalatromusic` | `StartBalatroMusic` | 巴拉特罗 |

##### F 类：状态变化（1 个）

| 事件 | 处理函数 | 触发场景 |
|---|---|---|
| `hasinspirationbuff` | `OnHasInspirationBuff` | 薇格弗德鼓舞 buff 变化 |

#### vanilla 中触发"场景 BGM"的代码

```1895:1895:scripts/prefabs/player_common.lua
        ThePlayer:PushEvent("playhermitmusic")
```

```1089:1089:scripts/prefabs/player_classified.lua
    inst._parent:PushEvent("playfarmingmusic")
```

——**任何代码都可以 PushEvent**——dynamicmusic 监听就触发。

#### 给 mod 的启示

**自定义 BGM 触发 = PushEvent**——但要 hook 到 dynamicmusic 的事件入口：

##### 方案 A：复用 vanilla 已有事件

最简单——直接 PushEvent 复用 vanilla 入口：

```lua
-- 我的 mod boss 出现时
ThePlayer:PushEvent("triggeredevent", { 
    name = "default",         -- 用 default → 播 ruins 通用 epic 音乐
    duration = 90,             -- 90 秒
})
```

##### 方案 B：自定义事件 + 自己监听

```lua
-- 在 mod boss prefab 内
ThePlayer:PushEvent("playmymodbossmusic")

-- 在另一处 / Player_post_init 内监听
AddPlayerPostInit(function(player)
    player:ListenForEvent("playmymodbossmusic", function(player)
        TheFocalPoint.SoundEmitter:KillSound("busy")
        TheFocalPoint.SoundEmitter:KillSound("danger")
        TheFocalPoint.SoundEmitter:PlaySound("mymod/music/boss", "danger")
        -- 自己管理 stop（DoTaskInTime）
    end)
end)
```

---

### 15.3.8（老手）自定义场景音乐——mod 实战完整模板

#### 目标：给我的 mod 角色加一段"专属砍树音乐"

需求：
- 角色装备某武器时，砍树过程中**播专属音乐**
- 不破坏 vanilla 的 dangerous / busy 音乐
- 自动按 mod 角色 / 武器联动

#### 完整模板

```lua
--[[
mymod/scripts/prefabs/myaxe.lua
专属砍树武器 + 专属砍树 BGM
]]--

local assets = {
    Asset("ANIM", "anim/myaxe.zip"),
    Asset("SOUNDPACKAGE", "sound/mymod_music.fev"),  -- ★ 必须先注册音频 bank
    Asset("SOUND", "sound/mymod_music.fsb"),
}

-- 业务用：开 / 关 mod 专属音乐
local function StartMyModMusic()
    if not TheFocalPoint then return end
    -- 互斥 vanilla 的 busy / danger
    TheFocalPoint.SoundEmitter:KillSound("busy")
    TheFocalPoint.SoundEmitter:KillSound("danger")
    -- 用自定义 name "mymod_music" 避免冲突 vanilla 的 name
    TheFocalPoint.SoundEmitter:PlaySound("mymod/music/chopping_fun", "mymod_music")
end

local function StopMyModMusic()
    if not TheFocalPoint then return end
    TheFocalPoint.SoundEmitter:KillSound("mymod_music")
end

-- Equip / Unequip
local function OnEquip(inst, owner)
    -- 通常的装备逻辑...
    
    -- 如果当前持有者是本地玩家 → 启动 mod 音乐
    if owner == ThePlayer then
        owner:ListenForEvent("performaction", function(player)
            if player:HasTag("working") then
                StartMyModMusic()
            end
        end, owner)
    end
end

local function OnUnequip(inst, owner)
    -- 通常的取下逻辑...
    
    if owner == ThePlayer then
        owner:RemoveEventCallback("performaction", nil, owner)
        StopMyModMusic()
    end
end

local function fn()
    local inst = CreateEntity()
    -- 标准 prefab 初始化...
    
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    inst.AnimState:SetBank("myaxe")
    inst.AnimState:SetBuild("myaxe")
    inst.AnimState:PlayAnimation("idle")
    
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("equippable")
    inst.components.equippable:SetOnEquip(OnEquip)
    inst.components.equippable:SetOnUnequip(OnUnequip)
    inst:AddComponent("weapon")
    inst.components.weapon:SetDamage(40)
    
    return inst
end

return Prefab("myaxe", fn, assets)
```

#### 模板要点

##### 要点 1：Asset 注册

```lua
Asset("SOUNDPACKAGE", "sound/mymod_music.fev"),
Asset("SOUND", "sound/mymod_music.fsb"),
```

——必须先注册（详见 15.2.2），否则 PlaySound 找不到 event。

##### 要点 2：用自定义 name 避免冲突

```lua
TheFocalPoint.SoundEmitter:PlaySound("mymod/music/chopping_fun", "mymod_music")
                                                                  ^^^^^^^^^^^^
                                                                  自定义 name
```

——不要用 `"busy"` / `"danger"` 等 vanilla name——否则会被 vanilla dynamicmusic 覆盖。

##### 要点 3：互斥 vanilla

```lua
TheFocalPoint.SoundEmitter:KillSound("busy")
TheFocalPoint.SoundEmitter:KillSound("danger")
```

——启动 mod 音乐前，**先停掉 vanilla 的 BGM**——否则两层 BGM 重叠刺耳。

##### 要点 4：仅本地玩家触发

```lua
if owner == ThePlayer then
```

——**网络游戏**中：**只有本地玩家装备** mod 武器时**本地客户端**才播——其他客户端不受影响。

##### 要点 5：自己管理生命周期

```lua
inst:DoTaskInTime(15, StopMyModMusic)  -- 15 秒后自动停
```

——和 vanilla 的 _extendtime 防抖类似——你的 mod 也要管理"何时停"。

#### 完整 mod 自定义"地块环境音"模板

```lua
--[[
mymod/scripts/prefabs/myworld_postinit.lua
给 mod 自定义地块加专属环境音
]]--

local function MyWorldPostInit(inst)
    -- 找到 ambientsound 组件
    if inst.components.ambientsound then
        -- 用 overrideambientsound 事件注入新地块音
        inst:PushEvent("overrideambientsound", {
            tile = WORLD_TILES.MYMOD_TILE,                    -- mod 自定义 tile
            override = "mymod/AMB/myworld",                   -- 对应 sound event
        })
    end
end

AddPrefabPostInit("forest", MyWorldPostInit)
AddPrefabPostInit("cave", MyWorldPostInit)
```

**注意**：`overrideambientsound` 事件设置的是**单个 tile 的覆盖**——不是"全部 tile 都改"。

#### Mod 自定义 stinger 实战

```lua
-- 我的 mod boss 死亡时播自定义 victory stinger
local function OnBossDefeated()
    if TheFocalPoint then
        TheFocalPoint.SoundEmitter:KillSound("danger")  -- 停 vanilla danger
        TheFocalPoint.SoundEmitter:PlaySound("mymod/music/victory_stinger")  -- 一次性 stinger
    end
end
```

#### Mod 自定义 mix 联动（综合）

15.2.4 学过 TheMixer——mod boss 出现时**同时**：

```lua
-- 1. 注册自定义 mix
TheMixer:AddNewMix("mymod_boss", 1.5, 5, {
    ["set_ambience/ambience"] = 0.2,
    ["set_music/soundtrack"] = 1,
    ["set_sfx/voice"] = 1,
    ["set_sfx/movement"] = 0.7,
    ["set_sfx/creature"] = 1,
    ["set_sfx/player"] = 1,
    ["set_sfx/HUD"] = 1,
    ["set_sfx/sfx"] = 0.8,
})

-- 2. boss 触发时同时启动
local function OnBossSpawn(boss)
    TheMixer:PushMix("mymod_boss")
    TheFocalPoint.SoundEmitter:KillSound("busy")
    TheFocalPoint.SoundEmitter:KillSound("danger")
    TheFocalPoint.SoundEmitter:PlaySound("mymod/music/boss_battle", "mymod_music")
end

local function OnBossDefeated()
    TheMixer:PopMix("mymod_boss")
    TheFocalPoint.SoundEmitter:KillSound("mymod_music")
    TheFocalPoint.SoundEmitter:PlaySound("mymod/music/victory_stinger")
end
```

——**Mixer 调氛围 + dynamicmusic 入口换 BGM + 自定义 stinger** —— 三联动 = 完整高质量 boss 战听感。

---

### 15.3.9（小结）速查表 + 4 行起步代码 + 15.4 预告

#### DynamicMusic 4 通道速查

| 通道 | name | 触发 | 时长 | 停止 |
|---|---|---|---|---|
| busy | "busy" | buildsuccess / 工作 / 探索 | 15 秒（可延长）| SetParameter intensity=0 |
| danger | "danger" | attacked / 战斗 | 10 秒（可延长）| KillSound |
| pirates | "pirates" | 海盗逼近 | 持续到远离 | KillSound |
| stinger | （无 name）| phase / sanity 变化 | 自然结束 | 自动 |

#### 21 事件入口速查

| 事件 | 触发场景 |
|---|---|
| `buildsuccess` | 合成成功 |
| `gotnewitem` | 获新物品（延长）|
| `performaction` | 动作执行（→ busy / danger）|
| `attacked` | 受攻击 |
| `goinsane` | 失智 |
| `goenlightened` | 顿悟 |
| `triggeredevent` | **boss 战触发**（最强大）|
| `playboatmusic` | 上船 |
| `playracemusic` | 赛跑 |
| `playhermitmusic` | 隐士 |
| `playfarmingmusic` | 种田 |
| `playpiratesmusic` | 海盗 |
| `playcarnivalmusic` | 嘉年华 |
| `stageplaymusic` | 舞台剧 |
| `hasinspirationbuff` | 鼓舞 buff |
| 其它 7 个场景音 | 详见 15.3.7 |

#### AmbientSound 速查

| API | 用途 |
|---|---|
| AMBIENT_SOUNDS 表 | 地块 → 音效映射（每地块 5 维：sound/winter/spring/summer/rain）|
| 11×11 tile 扫描 | 玩家周围 121 tile 决定环境音 |
| MAX_MIX_SOUNDS = 3 | 最多同时播 3 种环境音 |
| `daytime` parameter | 白天 / 夜晚 / 黄昏切换 |
| `sanity` / `enlightenment` parameter | 精神低 / 顿悟时混入耳鸣 |
| WAVE_VOLUME_SCALE | impassable tile 计算海浪音量 |

#### 关键源码引用速查

| 文件 | 行 | 内容 |
|---|---|---|
| `scripts/components/dynamicmusic.lua` | 38-60 | SEASON_BUSY/EPICFIGHT/DANGER MUSIC |
| 同 | 62-199 | TRIGGERED_DANGER_MUSIC（22 个 boss）|
| 同 | 202-223 | BUSYTHEMES 枚举（20 个主题）|
| 同 | 270-291 | StopBusy（fade 机制）|
| 同 | 293-334 | StartBusy（区域 / 季节切换）|
| 同 | 594-624 | StartDanger（epic 检测 + 季节）|
| 同 | 626-648 | StartTriggeredDanger（boss BGM）|
| 同 | 721-744 | 21 事件 listener |
| 同 | 800-810 | StartSoundEmitter（绑 TheFocalPoint）|
| `scripts/components/ambientsound.lua` | 29-97 | AMBIENT_SOUNDS 表 |
| 同 | 329-455 | OnUpdate（11×11 扫描 + 渐变）|

#### 4 行起步代码

##### A：mod 触发 vanilla boss BGM

```lua
-- 在 boss 出现时
ThePlayer:PushEvent("triggeredevent", { name = "default", duration = 60 })
```

##### B：完全自定义 BGM 模板

```lua
TheFocalPoint.SoundEmitter:KillSound("busy")
TheFocalPoint.SoundEmitter:KillSound("danger")
TheFocalPoint.SoundEmitter:PlaySound("mymod/music/special", "mymod_music")
inst:DoTaskInTime(60, function()
    TheFocalPoint.SoundEmitter:KillSound("mymod_music")
end)
```

##### C：自定义 stinger（事件式短音）

```lua
-- 玩家成就解锁时播
TheFocalPoint.SoundEmitter:PlaySound("mymod/music/achievement_stinger")
```

#### 三个设计原则

1. **BGM 永远挂 TheFocalPoint** —— 它是"听众的耳朵"，不衰减、不偏移、不漂移。
2. **使用 vanilla 已有事件入口最省事** —— `triggeredevent` 是 mod boss 加 BGM 的最简方案。
3. **互斥 vanilla 的 name** —— 启动 mod 音乐前必须 KillSound("busy") + KillSound("danger")，否则双层 BGM。

---

> **下一节预告**：15.4 节我们将进入 **角色语音系统（speech + 音效）** —— 15.1/15.2/15.3 都聚焦"播什么"——15.4 会讲**饥荒的 25+ 个角色各自怎么"说话"** —— 角色 prefab 的 `talker` 组件 + speech 文件 + sound 字典 + 性别 / 角色 / 表情联动 —— 15.4 是 mod 角色制作必读篇。

---


## 15.4 角色语音系统（speech + 音效）

### 本节导读

> **一句话定位**：饥荒里每一句"检查台词"、每一声"哎哟"，都经过两条流水线——**文字流水线（speech 文件 → GetString → talker:Say）** 和 **音效流水线（soundsname / talker_path_override → SoundEmitter → FMOD）**——本节讲清楚这两条流水线，以及 mod 角色怎么接入它们。

#### 一段宏观先讲清楚：角色"说话"的全貌

玩家右键检查一棵草，Wilson 说出 `"A tuft of grass."` ——这句话经过了多少层？

```
【玩家右键检查目标】
         │
         ▼
  LOOKAT action 被执行（scripts/actions.lua）
         │  targ.components.inspectable:GetDescription(doer)
         ▼
  GetString(viewer, "DESCRIBE", prefab)
         │  查 STRINGS.CHARACTERS[WILSON].DESCRIBE.GRASS
         │  → fallback → STRINGS.CHARACTERS.GENERIC.DESCRIBE.GRASS
         ▼
  inst.components.talker:Say(desc)
         │  ① 推送 "ontalk" 事件
         │  ② 在玩家头顶显示 FollowText 气泡
         ▼
  SGwilson 收到 "ontalk" → 进入 "talk" state
         │  DoTalkSound(inst)
         │  PlaySound("dontstarve/characters/wilson/talk_LP", "talk")
         ▼
  FMOD 播放角色嘴巴咕哝音
```

**两条流水线**：
| 流水线 | 关键文件 | 关键函数 | 结果 |
|---|---|---|---|
| **文字流水线** | `speech_wilson.lua`、`stringutil.lua` | `GetString` → `talker:Say` | 头顶气泡文字 |
| **音效流水线** | `SGwilson.lua` | `DoTalkSound` → `SoundEmitter:PlaySound` | 嘴型咕哝声 |

#### 15.4 节回答的 5 个核心问题

```
Q1: ──── 检查台词 "A tuft of grass." 是怎么取出来的？
         ↓ 答：GetString(inst, "DESCRIBE", prefab) + STRINGS.CHARACTERS 表

Q2: ──── talker:Say 的参数怎么填？何时触发 ontalk？
         ↓ 答：talker:Say(str, duration, noanim) —— 内部推 ontalk 事件

Q3: ──── 角色咕哝声音路径是怎么拼的？
         ↓ 答：(talker_path_override / "dontstarve/characters/") .. soundsname .. "/talk_LP"

Q4: ──── wanda 老年/少年声音为什么不一样？Wormwood 说话末尾有额外音效？
         ↓ 答：talksoundoverride / endtalksound 覆盖机制

Q5: ──── mod 自定义角色怎么接入这套系统？
         ↓ 答：speech 文件 + STRINGS.CHARACTERS[上大写prefab] + soundsname / talker_path_override
```

#### 你将看到的核心源码

| 文件 | 角色 | 用途 |
|---|---|---|
| `scripts/components/talker.lua` | 311 行 | **talker 组件全文**——Say / ShutUp / Chatter |
| `scripts/stringutil.lua` | `GetString` 函数 | 台词查找核心 |
| `scripts/strings.lua` | 第 15335 行起 | `STRINGS.CHARACTERS` 表挂载点 |
| `scripts/stategraphs/SGwilson.lua` | 多处 | 音效触发：DoHurtSound / DoTalkSound / StopTalkSound |
| `scripts/prefabs/wormwood.lua` | 第 779 行 | `endtalksound` 实例 |
| `scripts/prefabs/wanda.lua` | 第 88/125/155 行 | `talksoundoverride` 动态切换 |
| `scripts/prefabs/waxwell.lua` | 第 306 行 | `soundsname` 实例 |
| `scripts/fonts.lua` | 第 12-15 行 | `TALKINGFONT` 系列常量 |

#### 本节学习路径

```
15.4.1 新手 ─── speech 文件的三层结构 —— DESCRIBE / ACTIONFAIL / ANNOUNCE_
15.4.2 新手 ─── GetString —— 台词查找的核心函数
15.4.3 新手 ─── talker:Say —— 3 步让角色开口
                  ↓
15.4.4 进阶 ─── 语音音效路径系统 —— soundsname + talker_path_override
15.4.5 进阶 ─── 三大音效覆盖字段 —— talksoundoverride / hurtsoundoverride / endtalksound
15.4.6 进阶 ─── talker 样式定制 —— font / fontsize / colour
                  ↓
15.4.7 老手 ─── speechproxy —— 借用他人台词的代理机制
15.4.8 老手 ─── Chatter 系统 —— NPC 专用网络台词
15.4.9 老手 ─── mod 完整自定义角色语音实战
15.4.10     ─── 小结·速查表 + 5 行起步代码 + 15.5 预告
```

---

### 15.4.1（新手）speech 文件的三层结构

#### speech 文件是什么？

每个可玩角色都有一个 `speech_<prefab>.lua`，它是一张**巨大的 Lua 表**，描述"这个角色在各种情况下说什么"。

Wilson 的是 `speech_wilson.lua`（同时也是 GENERIC 模板），Waxwell 的是 `speech_waxwell.lua`，以此类推。

`scripts/strings.lua` 第 15335 行把它们全部挂到 `STRINGS.CHARACTERS`：

```15335:15354:scripts/strings.lua
STRINGS.CHARACTERS =
{
    GENERIC = require "speech_wilson",
    WAXWELL = require "speech_waxwell",
    WOLFGANG = require "speech_wolfgang",
    WX78 = require "speech_wx78",
    WILLOW = require "speech_willow",
    WENDY = require "speech_wendy",
    WOODIE = require "speech_woodie",
    WICKERBOTTOM = require "speech_wickerbottom",
    WATHGRITHR = require "speech_wathgrithr",
    WEBBER = require "speech_webber",
    WINONA = require "speech_winona",
    WORTOX = require "speech_wortox",
    WORMWOOD = require "speech_wormwood",
    WARLY = require "speech_warly",
    WURT = require "speech_wurt",
    WALTER = require "speech_walter",
    WANDA = require "speech_wanda",
}
```

**键名规则**：`string.upper(prefab)`——角色 prefab 是 `"wilson"`，键名就是 `"WILSON"`；但 Wilson 同时也是 `GENERIC`（后备表）。

#### speech 文件的三层顶级结构

打开 `speech_wilson.lua`，顶级键大致分为三类：

```
speech_wilson.lua
├── DESCRIBE          ← 【检查台词】右键检查物品/生物/角色时说的话
│     ├── GRASS       = "A tuft of grass."
│     ├── SPIDER      = { GENERIC="I hate spiders!", DEAD="One down." }
│     └── ...（约 1500+ 个物品/状态条目）
│
├── ACTIONFAIL        ← 【操作失败台词】执行某操作失败时说的话
│     ├── BUILD       = { MOUNTED="I can't place that from way up here." }
│     ├── SHAVE       = { AWAKEBEEFALO="I'm not going to try that..." }
│     └── ...
│
└── ANNOUNCE_xxx      ← 【事件公告台词】特殊事件发生时自动播报
      ├── ANNOUNCE_COLD       = "So cold!"
      ├── ANNOUNCE_BEES       = "BEEEEEEEEEEEEES!!!!"
      ├── ANNOUNCE_BOOMERANG  = "Ow! I should try to catch that!"
      └── ...（40+ 个触发条件）
```

以及一些特殊顶级键（不属于上面三类）：
```
├── BATTLECRY    ← 发起攻击时喊的话（对应 combat.lua battlecrystring）
├── COMBAT_QUIT  ← 放弃追击时说的话
└── DESCRIBE_TOODARK  ← 黑暗中无法检查时的话
```

#### DESCRIBE 的状态修饰符

DESCRIBE 下的条目可以是**字符串**（只有一句话），也可以是**表**（根据状态有不同的话）：

```lua
-- 单字符串：任何状态都说这句
GRASS = "A tuft of grass.",

-- 带状态修饰符的表：
SPIDER =
{
    GENERIC  = "I hate spiders!",  -- 默认/活着时
    DEAD     = "One down.",         -- 死亡状态
    SLEEPING = "Shh...",            -- 睡眠状态
},
```

状态修饰符由 `inspectable.lua` 的 `GetStatus()` 产生——它依次检查 DEAD / SLEEPING / BURNING / DISEASED / WITHERED / BARREN / PICKED / HELD / OCCUPIED / BURNT 等状态。

#### 字符串表的随机选取

如果条目是**无 key 的数组**，系统会随机取其中一条——这是"同一物品说不同话"的机制：

```lua
CARROT =
{
    "A fine, if somewhat bland, vegetable.",
    "Mmm. Root vegetables.",
    "Maybe I should plant some more.",
}
```

`stringutil.lua` 的 `getmodifiedstring` 函数负责这一逻辑：

```1:22:scripts/stringutil.lua
local function getmodifiedstring(topic_tab, modifier)
	if type(modifier) == "table" then
		local ret = topic_tab
		for i,v in ipairs(modifier) do
			if ret == nil then
				return nil
			end
			ret = ret[v]
		end
		return ret
	elseif modifier ~= nil then
        local ret = topic_tab[modifier]
        return (type(ret) == "table" and #ret > 0 and ret[math.random(#ret)])
                or ret
                or topic_tab.GENERIC
                or (#topic_tab > 0 and topic_tab[math.random(#topic_tab)])
                or nil
    else
		return topic_tab.GENERIC
                or (#topic_tab > 0 and topic_tab[math.random(#topic_tab)])
                or nil
	end
end
```

**规则**：modifier 对应的子表是数组 → 随机；modifier 查不到 → 降级到 `GENERIC`；`GENERIC` 也没有 → 尝试整个表随机取一条。

#### "only_used_by_xxx" 占位符

某些条目写着 `"only_used_by_woodie"` 或 `"only_used_by_waxwell_and_wicker"` 等字符串——这是**开发者占位注释**，意思是"Wilson 不说这句，但这个 key 必须存在（格式对齐要求）"。游戏代码本身不过滤这类字符串，只是实际上该触发条件只会在特定角色身上出现。

---

### 15.4.2（新手）GetString —— 台词查找的核心函数

#### GetString 的签名

```265:298:scripts/stringutil.lua
function GetString(inst, stringtype, modifier, nil_missing)
    local character =
        type(inst) == "string"
        and inst
        or (inst ~= nil and inst.prefab or nil)


    if type(inst) ~= "string" and inst.components.talker and inst.components.talker.speechproxy then
        character = inst.components.talker.speechproxy
    end

    character = character ~= nil and string.upper(character) or nil
    stringtype = stringtype ~= nil and string.upper(stringtype) or nil
	if type(modifier) == "table" then
		for i,v in ipairs(modifier) do
			v = string.upper(v)
		end
	else
		modifier = modifier ~= nil and string.upper(modifier) or nil
	end

    local specialcharacter =
        type(inst) == "table"
        and ((inst:HasTag("mime") and "mime") or
        (inst:HasTag("playerghost") and "ghost"))
        or character


	return GetSpecialCharacterString(specialcharacter)
        or getcharacterstring(STRINGS.CHARACTERS[character], stringtype, modifier)
        or getcharacterstring(STRINGS.CHARACTERS.GENERIC, stringtype, modifier)
		or (not nil_missing and ("UNKNOWN STRING: "..(character or "").." "..(stringtype or "").." "..(modifier or "")))
		or nil
end
```

#### 参数说明

| 参数 | 类型 | 含义 |
|---|---|---|
| `inst` | entity 或 string | 说话的角色实例（或直接传 prefab 字符串）|
| `stringtype` | string | speech 表顶级 key，如 `"DESCRIBE"`、`"ANNOUNCE_COLD"` |
| `modifier` | string / table / nil | 子 key 修饰符，如 `"DEAD"`、`"GENERIC"`，nil 表示取默认 |
| `nil_missing` | bool/nil | `true` 表示找不到时返回 nil；默认返回 `"UNKNOWN STRING: ..."` |

#### 查找优先级（3 步降级）

```
Step 1: 特殊角色字符串
        ─ 幽灵 → "Ooooh..."（随机拼接）
        ─ 哑剧(mime) → ""（空字符串）
        ─ 猴子 → 猴语拼接
        ─ 骸骨(wilton) → 随机名言
        ↓ 找不到 or 不适用
Step 2: STRINGS.CHARACTERS[string.upper(inst.prefab)]
        ─ 例：STRINGS.CHARACTERS.WOODIE.DESCRIBE.GRASS
        ↓ 找不到
Step 3: STRINGS.CHARACTERS.GENERIC（即 speech_wilson.lua）
        ─ 例：STRINGS.CHARACTERS.GENERIC.DESCRIBE.GRASS
```

**要点**：GENERIC 是保底，如果角色特有 speech 文件没有某个词条，就用 Wilson 的。

#### 常见调用形式

```lua
-- 检查台词（inspector 自动调，通常不需要手写）
local str = GetString(inst, "DESCRIBE", target.prefab)

-- 公告台词（组件 / prefab 主动调）
inst.components.talker:Say(GetString(inst, "ANNOUNCE_COLD"))

-- 带状态修饰的检查台词
local str = GetString(inst, "DESCRIBE", "FIREPIT")  -- modifier 是 "GENERIC"
local str = GetString(inst, "DESCRIBE", "FIREPIT", "DEAD")  -- 不存在状态修饰，实际用法

-- 操作失败台词
inst.components.talker:Say(GetString(inst, "ACTIONFAIL", {"BUILD", "MOUNTED"}))
```

> **注意**：`stringtype` 本质就是 speech 表的顶级 key——`"DESCRIBE"`、`"ACTIONFAIL"`、`"ANNOUNCE_COLD"` 都是顶级 key，因此只需要一个参数就能定位到 `ANNOUNCE_COLD` 整条字符串（modifier 传 nil 即可）。

---

### 15.4.3（新手）talker:Say —— 3 步让角色开口

#### talker:Say 的签名

```266:295:scripts/components/talker.lua
function Talker:Say(script, time, noanim, force, nobroadcast, colour, text_filter_context, original_author_netid, onfinishedlinesfn, sgparam)

    if TheWorld.speechdisabled then return nil end
    if TheWorld.ismastersim then

        if not force
            and (self.ignoring ~= nil or
                (self.inst.components.health ~= nil and self.inst.components.health:IsDead() and self.inst.components.revivablecorpse == nil) or
                (self.inst.components.sleeper ~= nil and self.inst.components.sleeper:IsAsleep())) then
            return
        elseif self.ontalk ~= nil then
            self.ontalk(self.inst, script)
        end
    elseif not force then
        if self.inst:HasTag("ignoretalking") then
            return
        elseif self.inst.components.revivablecorpse == nil then
            local health = self.inst.replica.health
            if health ~= nil and health:IsDead() then
                return
            end
        end
    end

    CancelSay(self)
    local lines = type(script) == "string" and { Line(script, noanim, time) } or script
    if lines ~= nil then
        self.task = self.inst:StartThread(function() sayfn(self, lines, nobroadcast, colour, text_filter_context, original_author_netid, onfinishedlinesfn, sgparam) end)
    end
end
```

#### 常用参数速查

| 参数位置 | 参数名 | 类型 | 默认 | 含义 |
|---|---|---|---|---|
| 1 | `script` | string 或 Line[] | — | 台词文字，或 Line 对象数组（多段台词）|
| 2 | `time` | number / nil | nil → `DEFAULT_TALKER_DURATION`(2.5s) | 每段台词显示时长 |
| 3 | `noanim` | bool / nil | nil | `true` 时不触发说话动画（头顶只显文字）|
| 4 | `force` | bool / nil | nil | `true` 时忽略 ignoretalking / 死亡 / 睡眠检查，强制说 |
| 5 | `nobroadcast` | bool / nil | nil | `true` 时不广播到网络（只本地显示）|

> `time` 受 `TUNING.MAX_TALKER_DURATION = 8.0` 上限限制（`scripts/tuning.lua` 第 91 行）。

#### 3 步最简用法

**第 1 步**：确保 entity 有 `talker` 组件

玩家 entity 在 `player_common.lua` 里已经自动添加；NPC 需要手动添加：

```lua
inst:AddComponent("talker")
```

**第 2 步**：在服务端调用 talker:Say

```lua
-- 最简形式
inst.components.talker:Say("Hello, world!")

-- 带 GetString 的标准形式
inst.components.talker:Say(GetString(inst, "ANNOUNCE_COLD"))

-- 带时长
inst.components.talker:Say("This will stay for 5 seconds.", 5)
```

**第 3 步**：游戏自动处理后续

1. `talker:Say` 内部调用 `sayfn`（在独立线程里执行）
2. `sayfn` 推送 `"ontalk"` 事件（携带 `{noanim, duration, sgparam}`）
3. SGwilson 收到 `ontalk` → 进入 `"talk"` state → 播放咕哝音 + 嘴型动画
4. `duration` 秒后，`sayfn` 推送 `"donetalking"` 事件
5. SGwilson 收到 `donetalking` → 回到 `"idle"` state → 停止咕哝音

```2068:2093:scripts/stategraphs/SGwilson.lua
    EventHandler("ontalk", function(inst, data)
        if inst:IsActing() and not inst.sg:HasStateTag("talking") and (inst.components.rider == nil or not inst.components.rider:IsRiding()) then
            if not inst.sg.statemem.doing_idle_for_line then
                if inst:HasTag("mime") then
                    inst.sg:GoToState("acting_mime")
                else
                    inst.sg:GoToState("acting_talk")
                end
            end
        elseif inst.sg:HasStateTag("idle") and not inst.sg:HasStateTag("notalking") then
			if data.sgparam and data.sgparam.closeinspect and
				not (	inst.components.rider:IsRiding() or
						inst.components.inventory:IsHeavyLifting() or
						inst:IsChannelCasting()
					)
			then
				inst.sg:GoToState("closeinspect")
			elseif not inst:HasTag("mime") then
				inst.sg:GoToState("talk", data.noanim)
			elseif not inst.components.inventory:IsHeavyLifting() then
				inst.sg:GoToState("mime")
			end
		elseif data.duration ~= nil and not data.noanim then
			inst.sg.mem.queuetalk_timeout = data.duration + GetTime()
		end
    end),
```

#### 多段台词（Line 数组）

如果想让角色说多段话（每段停顿一下），用 `Line` 对象数组：

```lua
local lines = {
    Line("First line.", false, 3),   -- message, noanim, duration
    Line(nil, false, 1),              -- nil message = 短暂停顿（隐藏气泡）
    Line("Second line.", false, 3),
}
inst.components.talker:Say(lines)
```

`Line` 构造函数（`talker.lua` 第 5-9 行）：

```5:9:scripts/components/talker.lua
Line = Class(function(self, message, noanim, duration)
    self.message = message
    self.noanim = noanim
	self.duration = duration
end)
```

#### ShutUp —— 强制打断当前台词

```297:307:scripts/components/talker.lua
function Talker:ShutUp()
    CancelSay(self)

    if self.chatter ~= nil and TheWorld.ismastersim then
        self.chatter.strtbl:set("")
        if self.chatter.task ~= nil then
            self.chatter.task:Cancel()
            self.chatter.task = nil
        end
    end
end
```

`ShutUp()` 无参数，立即终止当前台词并推送 `"donetalking"` 事件。

---

### 15.4.4（进阶）语音音效路径系统 —— soundsname + talker_path_override

#### 音效路径是怎么拼的？

打开 `scripts/stategraphs/SGwilson.lua`，`DoTalkSound` 函数展示了核心逻辑：

```175:183:scripts/stategraphs/SGwilson.lua
local function DoTalkSound(inst)
    if inst.talksoundoverride ~= nil then
        inst.SoundEmitter:PlaySound(inst.talksoundoverride, "talk")
        return true
    elseif not inst:HasTag("mime") then
        inst.SoundEmitter:PlaySound((inst.talker_path_override or "dontstarve/characters/")..(inst.soundsname or inst.prefab).."/talk_LP", "talk")
        return true
    end
end
```

**路径拼接公式**（无覆盖时）：

```
音效路径 = (talker_path_override 或 "dontstarve/characters/")
         .. (soundsname 或 inst.prefab)
         .. "/<音效key>"
```

举例：
- Wilson（无特殊设置）：`"dontstarve/characters/"` + `"wilson"` + `"/talk_LP"` = `"dontstarve/characters/wilson/talk_LP"`
- Waxwell（`soundsname = "maxwell"`）：`"dontstarve/characters/"` + `"maxwell"` + `"/hurt"` = `"dontstarve/characters/maxwell/hurt"`
- Webber（`talker_path_override = "dontstarve_DLC001/characters/"`）：`"dontstarve_DLC001/characters/"` + `"webber"` + `"/talk_LP"` = `"dontstarve_DLC001/characters/webber/talk_LP"`

#### soundsname 字段

`inst.soundsname` 是一个**可选字符串**，当角色 prefab 名与 FMOD bank 中的角色目录名不一致时使用。

典型例子：

```306:306:scripts/prefabs/waxwell.lua
    inst.soundsname = "maxwell"
```

Waxwell 的 prefab 是 `"waxwell"`，但 FMOD bank 里的目录叫 `"maxwell"`（DLC1 时代的遗留命名）——因此需要 `soundsname = "maxwell"` 修正。

**mod 场景**：若你的 mod 角色没有自定义音效 bank，想借用已有角色的音效：

```lua
-- 借用 Willow 的声音（在 master_postinit 中）
inst.soundsname = "willow"
```

#### talker_path_override 字段

`inst.talker_path_override` 指定音效路径前缀，覆盖默认的 `"dontstarve/characters/"`. 目前 vanilla 中使用这个字段的角色有：

| 角色 | talker_path_override | 原因 |
|---|---|---|
| Webber | `"dontstarve_DLC001/characters/"` | DLC1 角色，音效在 DLC1 bank |
| Wathgrithr | `"dontstarve_DLC001/characters/"` | DLC1 角色，音效在 DLC1 bank |
| Wanda | `"wanda2/characters/"` | Wanda 有独立 bank |
| Wonkey | `"monkeyisland/characters/"` | 变身猴王时的特殊 bank |

#### 5 种标准音效 key

`SGwilson.lua` 中对角色的标准音效 key（接在路径后面的部分）总结如下：

| 音效 key | 函数 | 触发时机 | 是否 loop |
|---|---|---|---|
| `/talk_LP` | `DoTalkSound` | 说话时循环 | 是（name="talk"）|
| `/hurt` | `DoHurtSound` | 受伤时 | 否 |
| `/yawn` | `DoYawnSound` | 疲劳/打哈欠 | 否 |
| `/death_voice` | SGwilson 死亡 state | 角色死亡 | 否 |
| `/sinking` | SGwilson 沉船 state | 溺水/沉船 | 否 |
| `/emote` | `DoEmoteSound` | 表情动作 | 否 |

#### DoHurtSound 和 DoYawnSound

对齐 `DoTalkSound` 的设计，`DoHurtSound` 和 `DoYawnSound` 也有相同的路径拼接逻辑：

```159:173:scripts/stategraphs/SGwilson.lua
local function DoHurtSound(inst)
    if inst.hurtsoundoverride ~= nil then
        inst.SoundEmitter:PlaySound(inst.hurtsoundoverride, nil, inst.hurtsoundvolume)
    elseif not inst:HasTag("mime") then
        inst.SoundEmitter:PlaySound((inst.talker_path_override or "dontstarve/characters/")..(inst.soundsname or inst.prefab).."/hurt", nil, inst.hurtsoundvolume)
    end
end

local function DoYawnSound(inst)
    if inst.yawnsoundoverride ~= nil then
        inst.SoundEmitter:PlaySound(inst.yawnsoundoverride)
    elseif not inst:HasTag("mime") then
        inst.SoundEmitter:PlaySound((inst.talker_path_override or "dontstarve/characters/")..(inst.soundsname or inst.prefab).."/yawn")
    end
end
```

注意 `DoHurtSound` 还支持 `inst.hurtsoundvolume` 控制受伤音量。

---

### 15.4.5（进阶）三大音效覆盖字段 —— talksoundoverride / hurtsoundoverride / endtalksound

#### 三个字段一览

| 字段名 | 挂在哪里 | 作用 |
|---|---|---|
| `inst.talksoundoverride` | entity 实例 | 完全替换说话音效路径（loop，name="talk"）|
| `inst.hurtsoundoverride` | entity 实例 | 完全替换受伤音效路径 |
| `inst.endtalksound` | entity 实例 | 说话结束时额外播放一次（非 loop）|

这三个字段都**直接挂在 entity 实例上**（不是组件字段），在 `SGwilson.lua` 中被读取。

#### talksoundoverride —— 动态切换说话音（Wanda 实例）

Wanda 有三个年龄状态（老年 / 正常 / 少年），每个状态说话声音不同：

```88:88:scripts/prefabs/wanda.lua
    inst.talksoundoverride = "wanda2/characters/wanda/talk_old_LP"
```

```125:126:scripts/prefabs/wanda.lua
    inst.talksoundoverride = nil
    inst.hurtsoundoverride = nil
```

```155:155:scripts/prefabs/wanda.lua
    inst.talksoundoverride = "wanda2/characters/wanda/talk_young_LP"
```

逻辑：进入老年时设 `talksoundoverride = "...talk_old_LP"`，恢复正常时置 `nil`（回到默认拼接路径），进入少年时设 `talksoundoverride = "...talk_young_LP"`。

**对 mod 的启示**：
- 角色变身（如狼人、精神崩溃形态）时可以用 `talksoundoverride` 切换语音
- 设 `nil` 会回落到默认路径拼接（`soundsname` + `talker_path_override`）

#### endtalksound —— 说话结束音（Wormwood 实例）

Wormwood 说话结束时有一个特殊的"木质结束音"：

```779:779:scripts/prefabs/wormwood.lua
    inst.endtalksound = "dontstarve/characters/wormwood/end"
```

`StopTalkSound` 函数在停止说话循环音之前会检查这个字段：

```185:190:scripts/stategraphs/SGwilson.lua
local function StopTalkSound(inst, instant)
    if not instant and inst.endtalksound ~= nil and inst.SoundEmitter:PlayingSound("talk") then
        inst.SoundEmitter:PlaySound(inst.endtalksound)
    end
    inst.SoundEmitter:KillSound("talk")
end
```

**逻辑**：`instant=false` + `endtalksound` 存在 + 说话 loop 正在播放 → 先播一次结束音 → 再 KillSound("talk")。

#### hurtsoundoverride —— Woodie 变身实例

Woodie 变鹅形态时替换受伤音：

```1026:1026:scripts/prefabs/woodie.lua
        inst.hurtsoundoverride = "dontstarve/characters/woodie/goose/hurt"
```

变回正常形态时清除：

```645:646:scripts/prefabs/wolfgang.lua
    inst.talksoundoverride = nil
    inst.hurtsoundoverride = nil
```

（Wolfgang 也在体型切换时管理这些字段。）

---

### 15.4.6（进阶）talker 样式定制 —— font / fontsize / colour

#### talker 组件的样式字段

`talker` 组件内部有三个视觉样式字段，在 `sayfn` 函数中被消费：

```11:20:scripts/components/talker.lua
local Talker = Class(function(self, inst)
    self.inst = inst
    self.task = nil
    self.ignoring = nil
    self.mod_str_fn = nil
    self.offset = nil
    self.offset_fn = nil
    self.disablefollowtext = nil
    self.resolvechatterfn = nil
end)
```

可以在 `master_postinit` 或 `common_postinit` 中直接赋值：

| 字段 | 类型 | 默认 | 作用 |
|---|---|---|---|
| `talker.font` | string（FONT 常量）| `TALKINGFONT` | 气泡文字字体 |
| `talker.fontsize` | number | 35 | 气泡文字大小 |
| `talker.colour` | Vector3 | white（1,1,1）| 气泡文字颜色（RGB，0~1）|

#### TALKINGFONT 系列常量

`scripts/fonts.lua` 定义了 4 种说话字体：

```12:15:scripts/fonts.lua
TALKINGFONT = "talkingfont"
TALKINGFONT_WORMWOOD = "talkingfont_wormwood"
TALKINGFONT_TRADEIN = "talkingfont_tradein"
TALKINGFONT_HERMIT = "talkingfont_hermit"
```

| 字体常量 | 外观特征 | 使用角色/NPC |
|---|---|---|
| `TALKINGFONT` | 标准手写风 | 大多数角色 / NPC |
| `TALKINGFONT_WORMWOOD` | 植物藤蔓风 | Wormwood |
| `TALKINGFONT_TRADEIN` | 圆润商业风 | 留声机/交易机 |
| `TALKINGFONT_HERMIT` | 隐士古朴风 | 隐士螃蟹 |

#### 实际使用示例

Wormwood 的字体设置（`scripts/prefabs/wormwood.lua`）：

```765:767:scripts/prefabs/wormwood.lua
        inst.components.talker.fontsize = 40
    end
    inst.components.talker.font = TALKINGFONT_WORMWOOD
```

颜色设置（例：给 NPC 设置暗红色台词）：

```730:732:scripts/prefabs/shadow_battleaxe.lua
    inst.components.talker.fontsize = 28
    inst.components.talker.font = TALKINGFONT
    inst.components.talker.colour = TALK_COLOUR
```

mod 中的典型写法：

```lua
local function master_postinit(inst)
    -- 字体和颜色（颜色 RGB 值各分量 0~1）
    inst.components.talker.font = TALKINGFONT
    inst.components.talker.fontsize = 35
    inst.components.talker.colour = Vector3(0.8, 0.4, 0.9)  -- 淡紫色
end
```

#### lineduration 字段

还有一个不太常用的字段 `talker.lineduration`：如果设置了，每行台词的默认时长改为这个值（而不是 `TUNING.DEFAULT_TALKER_DURATION = 2.5`）。

---

### 15.4.7（老手）speechproxy —— 借用他人台词

#### 什么是 speechproxy？

`talker.speechproxy` 是一个可选字段，设置后 `GetString` 和 `GetLine` 会**把 `inst.prefab` 替换成 `speechproxy`** 来查找台词——即让角色说另一个角色的话。

```272:274:scripts/stringutil.lua
    if type(inst) ~= "string" and inst.components.talker and inst.components.talker.speechproxy then
        character = inst.components.talker.speechproxy
    end
```

#### 使用场景：无缝换人（SeamlessPlayerSwapper）

`scripts/components/seamlessplayerswapper.lua` 中，玩家换身后台词代理被设为原角色 prefab：

```97:102:scripts/components/seamlessplayerswapper.lua
function SeamlessPlayerSwapper:PostTransformSetup()
	if self.main_data.mime then
	    self.inst:AddTag("mime")
	end
	self.inst.components.talker.speechproxy = self.main_data.prefab
end
```

**含义**：玩家换到新身体后，新身体的台词依旧来自原来那个角色的 speech 文件。

#### mod 中的应用

如果你的 mod 角色在某些状态下要说另一个角色的话（例如变身后借用 Woodie 的台词）：

```lua
-- 变身时
inst.components.talker.speechproxy = "woodie"

-- 变回时
inst.components.talker.speechproxy = nil
```

---

### 15.4.8（老手）Chatter 系统 —— NPC 专用网络台词

#### Chatter 是什么？

玩家角色调用 `talker:Say` 只需要在服务端调用，系统会通过 `TheNet:Talker(...)` 广播到客户端。但 **NPC 的 talker** 不一定走同一套流程——它们需要通过 `Chatter` 系统，利用 `net_string` 等网络变量同步。

`Chatter` 系统的核心组成：

```80:95:scripts/components/talker.lua
function Talker:MakeChatter()
    if self.chatter == nil then
        --for npc
        self.chatter =
        {
            strtbl = net_string(self.inst.GUID, "talker.chatter.strtbl", "chatterdirty"),
            strid = net_smallbyte(self.inst.GUID, "talker.chatter.strid", "chatterdirty"),
            strtime = net_tinybyte(self.inst.GUID, "talker.chatter.strtime"),
            forcetext = net_bool(self.inst.GUID, "talker.chatter.forcetext"),
            echotochatpriority = net_tinybyte(self.inst.GUID, "talker.chatter.echotochatpriority"),
        }
        if not TheWorld.ismastersim then
            self.inst:ListenForEvent("chatterdirty", OnChatterDirty)
        end
    end
end
```

#### Chatter 的 5 个网络变量

| 变量名 | 类型 | 含义 |
|---|---|---|
| `strtbl` | `net_string` | 台词所在的 STRINGS 子表路径（如 `"CHARACTERS.GENERIC.DESCRIBE"`)  |
| `strid` | `net_smallbyte` | 台词数组索引（0 = 取整个表，正整数 = 具体索引）|
| `strtime` | `net_tinybyte` | 台词显示时长（秒）|
| `forcetext` | `net_bool` | 是否 force say（跳过 noanim 检查）|
| `echotochatpriority` | `net_tinybyte` | 是否回显到聊天框（0=不显，>0=显示）|

#### Talker:Chatter 的调用

```104:123:scripts/components/talker.lua
function Talker:Chatter(strtbl, strid, time, forcetext, echotochatpriority)
    if self.chatter ~= nil and TheWorld.ismastersim then
        self.chatter.strtbl:set(strtbl)
        --force at least the id dirty, so that it's possible to repeat strings
        strid = strid or 0
        self.chatter.strid:set_local(strid)
        self.chatter.strid:set(strid)
        self.chatter.strtime:set(time or 0)
        self.chatter.forcetext:set(forcetext == true)
        echotochatpriority = (echotochatpriority == true and CHATPRIORITIES.LOW)
            or ((echotochatpriority == false or echotochatpriority == nil) and CHATPRIORITIES.NOCHAT)
            or echotochatpriority
        self.chatter.echotochatpriority:set(echotochatpriority)
        if self.chatter.task ~= nil then
            self.chatter.task:Cancel()
        end
        self.chatter.task = self.inst:DoTaskInTime(1, OnCancelChatter, self)
        OnChatterDirty(self.inst)
    end
end
```

#### Chatter 与 Say 的区别

| 维度 | talker:Say | talker:Chatter |
|---|---|---|
| 适用对象 | 玩家角色（player entity）| NPC（ChattyNode / 战斗喊话）|
| 同步方式 | `TheNet:Talker(...)` 实时广播 | `net_string` dirty 事件 |
| 传参方式 | 直接传字符串 | 传 STRINGS 表路径 + 索引 |
| 需要 MakeChatter | 否 | 是（需提前调用 `MakeChatter()`）|

**mod 一般不需要直接使用 Chatter**——它主要服务于 vanilla NPC 的战斗喊话（`ChattyNode` 组件）。mod 中的 NPC 直接用 `talker:Say` 即可。

---

### 15.4.9（老手）mod 完整自定义角色语音实战

#### 全流程概览

```
第 1 步：创建 speech_<myprefab>.lua
         └── 复制 speech_wilson.lua 框架，修改台词
         
第 2 步：在 STRINGS.CHARACTERS 表注册
         └── STRINGS.CHARACTERS.MYPREFAB = require "speech_myprefab"

第 3 步：在角色 prefab 中配置音效（三选一）
         ├── A. 无自定义 bank → 借用已有角色音效（设 soundsname）
         ├── B. 有自定义 bank → 设 talker_path_override 到自己的 bank 目录
         └── C. 混合 → talker_path_override + 部分 xxxsoundoverride

第 4 步：可选：定制 talker 样式（font / colour）
第 5 步：可选：为特殊状态动态切换 talksoundoverride
```

#### 第 1 步：创建 speech 文件

在 mod 的 `scripts/` 目录下创建 `speech_mychar.lua`（以中文角色为例）：

```lua
-- scripts/speech_mychar.lua
return {
    -- 【检查台词】—— 尽量覆盖常用物品
    DESCRIBE =
    {
        GRASS = "这就是草。",
        SPIDER = {
            GENERIC = "蜘蛛，不吉利。",
            DEAD    = "总算安静了。",
        },
        RABBIT = "小东西，别跑。",
        -- ... 更多检查台词 ...
    },

    -- 【操作失败台词】
    ACTIONFAIL =
    {
        BUILD =
        {
            MOUNTED = "骑着马没法放东西。",
            HASPET  = "已经有宠物了。",
        },
    },

    -- 【事件公告台词】
    ANNOUNCE_COLD      = "好冷啊！",
    ANNOUNCE_HOT       = "太热了！",
    ANNOUNCE_DEERCLOPS = "听到了什么巨大的声音……",
    ANNOUNCE_BEES      = "蜜蜂！到处都是蜜蜂！！",

    -- 【战斗台词】
    BATTLECRY =
    {
        GENERIC = "来吧！",
        PIG     = "抱歉了，猪先生。",
    },
}
```

> **技巧**：未覆盖的条目会自动降级到 `GENERIC`（Wilson 台词）—— 不必一一填写所有 1500+ 条目。

#### 第 2 步：在 modmain.lua 或 mod strings 文件中注册

```lua
-- modmain.lua（或单独的 strings 文件，通过 modimport 引入）
local STRINGS = GLOBAL.STRINGS

STRINGS.CHARACTERS.MYCHAR = require "speech_mychar"

-- 同时设置角色显示名（可选）
STRINGS.NAMES.MYCHAR       = "我的角色"
STRINGS.CHARACTER_TITLES.MYCHAR = "The Custom One"
```

> **注意**：键名必须是 `string.upper(prefab)` 的结果。prefab 是 `"mychar"` → 键名是 `"MYCHAR"`。

#### 第 3 步A：借用已有角色音效（最简方案）

在角色 prefab 的 `master_postinit` 中：

```lua
local function master_postinit(inst)
    -- 借用 Willow 的所有语音（talk/hurt/yawn/death_voice）
    inst.soundsname = "willow"
    -- talker_path_override 不设，保持默认 "dontstarve/characters/"
end
```

这样角色说话就用 `"dontstarve/characters/willow/talk_LP"`，受伤用 `"dontstarve/characters/willow/hurt"`。

#### 第 3 步B：使用自定义 FMOD bank（完整方案）

若你制作了自己的 FMOD bank（`mymod_characters_mychar.fev`），bank 中有 `"mymod/characters/mychar/talk_LP"` 等事件：

```lua
local function common_postinit(inst)
    -- 不需要在这里设（音效字段是服务端逻辑）
end

local function master_postinit(inst)
    -- 指向自己 bank 的路径前缀
    inst.talker_path_override = "mymod/characters/"
    -- soundsname 不设，默认使用 inst.prefab = "mychar"
    -- 最终拼成：mymod/characters/mychar/talk_LP
    
    -- 如果 FMOD 中目录名和 prefab 不一致，额外设 soundsname
    -- inst.soundsname = "mychar_v2"
end
```

在 mod `Assets` 表中声明 bank：

```lua
Assets =
{
    Asset("SOUND", "sound/mymod_characters_mychar.fev"),
    Asset("SOUND", "sound/mymod_characters_mychar.fsb"),
}
```

#### 第 4 步：定制 talker 样式

```lua
local function master_postinit(inst)
    -- 字体
    inst.components.talker.font     = TALKINGFONT      -- 或 TALKINGFONT_WORMWOOD 等
    inst.components.talker.fontsize = 35               -- 默认 35
    -- 颜色（Vector3，RGB 各 0~1）
    inst.components.talker.colour   = Vector3(0.6, 0.9, 1.0)  -- 淡蓝色
    -- 借用声音
    inst.soundsname = "willow"
end
```

#### 第 5 步：状态切换语音（进阶可选）

```lua
-- 变身/特殊状态时切换说话音效
local function OnTransform(inst)
    inst.talksoundoverride  = "mymod/characters/mychar/talk_transformed_LP"
    inst.hurtsoundoverride  = "mymod/characters/mychar/hurt_transformed"
end

local function OnRestoreNormal(inst)
    inst.talksoundoverride = nil
    inst.hurtsoundoverride = nil
end
```

#### 完整最小 prefab 模板

```lua
-- scripts/prefabs/mychar.lua
local MakePlayerCharacter = require "prefabs/player_common"

local assets = { Asset("SCRIPT", "scripts/prefabs/player_common.lua") }
local prefabs = {}

local function common_postinit(inst)
    inst.MiniMapEntity:SetIcon("mychar.tex")
end

local function master_postinit(inst)
    -- 基础属性
    inst.components.health:SetMaxHealth(150)
    inst.components.hunger:SetMax(150)
    inst.components.sanity:SetMax(200)

    -- 语音系统：借用 Willow 的声音
    inst.soundsname = "willow"

    -- talker 样式
    inst.components.talker.fontsize = 35
    inst.components.talker.font     = TALKINGFONT
    inst.components.talker.colour   = Vector3(0.8, 0.5, 1.0)
end

return MakePlayerCharacter("mychar", prefabs, assets, common_postinit, master_postinit)
```

```lua
-- modmain.lua（注册 speech）
GLOBAL.STRINGS.CHARACTERS.MYCHAR = require "speech_mychar"
GLOBAL.STRINGS.NAMES.MYCHAR      = "My Character"
```

---

### 15.4.10（小结）速查表 + 5 行起步代码 + 15.5 预告

#### 文字流水线速查

| 步骤 | 函数/字段 | 文件 | 说明 |
|---|---|---|---|
| 存台词 | `speech_<prefab>.lua` | mod scripts/ | 三层结构：DESCRIBE / ACTIONFAIL / ANNOUNCE_ |
| 注册台词 | `STRINGS.CHARACTERS.UPPER_PREFAB` | modmain.lua | `= require "speech_mychar"` |
| 取台词 | `GetString(inst, type, modifier)` | stringutil.lua | 3 步降级：特殊 → 角色 → GENERIC |
| 说台词 | `talker:Say(str, time, noanim)` | talker.lua | 推 ontalk 事件，同步到网络 |
| 停台词 | `talker:ShutUp()` | talker.lua | 立即打断，推 donetalking |

#### 音效流水线速查

| 步骤 | 字段/函数 | 文件 | 说明 |
|---|---|---|---|
| 路径前缀 | `inst.talker_path_override` | entity | 默认 `"dontstarve/characters/"` |
| 路径目录 | `inst.soundsname`（或 prefab）| entity | FMOD bank 中的角色目录名 |
| 说话音 | `inst.talksoundoverride` | entity | 设非 nil 完全替换默认路径 |
| 受伤音 | `inst.hurtsoundoverride` | entity | 设非 nil 完全替换默认路径 |
| 结束音 | `inst.endtalksound` | entity | 说话结束时额外播一次 |
| 音量 | `inst.hurtsoundvolume` | entity | 受伤音专用音量（0~1）|

#### talker 组件样式速查

| 字段 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `talker.font` | string | `TALKINGFONT` | 气泡字体 |
| `talker.fontsize` | number | 35 | 气泡字号 |
| `talker.colour` | Vector3 | (1,1,1) white | 气泡文字颜色 |
| `talker.lineduration` | number | nil → 2.5s | 每行默认显示时长 |

#### 角色 speech 注册速查

| 步骤 | 代码 |
|---|---|
| 注册 speech 文件 | `STRINGS.CHARACTERS.MYCHAR = require "speech_mychar"` |
| 借用他人 speech | `inst.components.talker.speechproxy = "willow"` |
| 注册角色名 | `STRINGS.NAMES.MYCHAR = "My Character"` |

#### 5 行起步代码

##### A：最简 mod 角色语音（借音 + 自定义台词）

```lua
-- master_postinit 中
inst.soundsname = "willow"   -- 借用 Willow 声音
inst.components.talker.colour = Vector3(0.9, 0.5, 1.0)
-- modmain.lua
GLOBAL.STRINGS.CHARACTERS.MYCHAR = require "speech_mychar"
```

##### B：让角色立刻说一句话

```lua
inst.components.talker:Say(GetString(inst, "ANNOUNCE_COLD"))
-- 或直接说固定台词
inst.components.talker:Say("这里太冷了！", 4)
```

##### C：状态切换语音

```lua
-- 进入特殊形态
inst.talksoundoverride = "mymod/characters/mychar/talk_form2_LP"
-- 恢复正常形态
inst.talksoundoverride = nil
```

##### D：为已有 NPC 添加检查台词

```lua
-- modmain.lua
GLOBAL.STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYNPC = "It looks friendly."
-- 若想给特定角色不同的话：
GLOBAL.STRINGS.CHARACTERS.WENDY.DESCRIBE.MYNPC = "Another soul in these lands."
```

---

> **下一节预告**：15.5 节我们将进入 **Mod 中添加自定义音效的完整流程** —— 15.4 教会了你如何"对接"已有的语音系统；15.5 会讲**如何从零创建自己的 FMOD bank、打包进 mod、并在代码里正确引用** —— 是自定义声音的最后一公里。

---



## 15.5 Mod 中添加自定义音效的完整流程

### 本节导读

> **一句话定位**：15.1~15.4 教会了你"如何在代码里调用音效"——15.5 教会你**如何准备音效文件本身**，从 `.fev/.fsb` 是什么、到 FMOD Studio 创建 bank、到 mod 目录结构，最后到 `RemapSoundEvent` 替换 vanilla 音效——这是 mod 音效制作的最后一公里。

#### 一段宏观先讲清楚：mod 音效的四条路

```
┌──────────────────────────────────────────────────────────────────────────┐
│  四条路                                                                    │
├──────────────────────────────────────────────────────────────────────────┤
│  路线 A（最省事）  借用 vanilla bank 已有音效路径                            │
│                   → inst.soundsname = "willow"                            │
│                   → 不需要任何额外文件                                       │
│                                                                            │
│  路线 B（代码替换）在代码里直接 PlaySound vanilla 路径                        │
│                   → inst.SoundEmitter:PlaySound("dontstarve/sfx/thunder") │
│                   → 不需要额外文件，但你只能用 vanilla 的声音                │
│                                                                            │
│  路线 C（事件重映射）RemapSoundEvent("vanilla/old", "mymod/new")            │
│                   → 需要自己的 .fev/.fsb，但只改路径，不改代码               │
│                   → 可用于全局替换 vanilla 音效（如改角色喊声）              │
│                                                                            │
│  路线 D（完全自定义）FMOD Studio 制作 bank → 打包进 mod                     │
│                   → 需要 FMOD Studio + 音频素材                             │
│                   → 完全自由，适合有新音效需求的角色/物品 mod                │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 15.5 节回答的 5 个核心问题

```
Q1: ──── .fev 和 .fsb 分别是什么？二者是什么关系？
         ↓ 答：.fev = 事件定义文件（路径、参数）；.fsb = 音频数据文件

Q2: ──── vanilla 的 "dontstarve/characters/wilson/talk_LP" 路径是怎么来的？
         ↓ 答：bank 文件名（dontstarve.fev）就是路径前缀

Q3: ──── 如何让 mod 的声音文件被游戏加载？
         ↓ 答：Asset("SOUND", "sound/xxx.fev") + Asset("SOUND", "sound/xxx.fsb")

Q4: ──── FMOD Studio 里怎么创建一个能被饥荒识别的事件？
         ↓ 答：新建 bank，事件名写 "mymod/xxx/yyy"，确保 bank 名是 "mymod"

Q5: ──── mod 怎么全局替换 vanilla 某个音效？
         ↓ 答：RemapSoundEvent("dontstarve/old/path", "mymod/new/path")
```

#### 你将看到的核心源码

| 文件 | 行 | 用途 |
|---|---|---|
| `scripts/preloadsounds.lua` | 全文 | vanilla 声音文件清单，展示 `.fev`/`.fsb` 命名规律 |
| `scripts/preloadsounds.lua` | 261-264 | `PreloadSoundList` → `TheSim:PreloadFile("sound/"..v)` |
| `scripts/modutil.lua` | 842-850 | `RemapSoundEvent` / `RemoveRemapSoundEvent` mod API |
| `scripts/prefabs/wx78.lua` | 11 | `Asset("SOUND", "sound/wx78.fsb")` 单 bank 声明 |
| `scripts/prefabs/player_common.lua` | 2074-2075 | 多 bank 声明实例 |

#### 本节学习路径

```
15.5.1 新手 ─── 饥荒音效文件体系 —— .fev + .fsb 是什么，路径前缀规律
15.5.2 新手 ─── 路线 A/B：不制作 bank 就能用的方法
15.5.3 新手 ─── Asset("SOUND") —— 声明声音依赖的正确方式
                  ↓
15.5.4 进阶 ─── FMOD Studio 工作流 —— 安装、创建 bank、命名事件
15.5.5 进阶 ─── 导出 bank 与 mod 目录结构
15.5.6 进阶 ─── 在代码里使用自定义音效路径
                  ↓
15.5.7 老手 ─── RemapSoundEvent —— 全局替换 vanilla 音效
15.5.8 老手 ─── 完整 mod 音效实战 —— 从 FMOD 到 PlaySound
15.5.9     ─── 小结·速查表 + 常见错误 + 本章总结
```

---

### 15.5.1（新手）饥荒音效文件体系 —— .fev + .fsb 是什么

#### 两种文件各司其职

饥荒使用 **FMOD Studio** 音频引擎，其声音文件分两类：

| 扩展名 | 全称 | 内容 | 类比 |
|---|---|---|---|
| `.fev` | FMOD Event Bank | 事件定义：事件路径、参数、引用哪些音频样本 | "目录清单" |
| `.fsb` | FMOD Sample Bank | 压缩后的实际音频数据（PCM/Vorbis/OPUS）| "音频仓库" |

**关键关系**：`.fev` 是"乐谱"，`.fsb` 是"乐手"——`.fev` 告诉引擎"事件 A 在哪里、用哪些音频"，`.fsb` 里装着对应的音频数据。

#### vanilla 的两种打包结构

打开 `scripts/preloadsounds.lua`，可以看到两种模式：

**模式一：主 .fev + 分散 .fsb**（旧格式，基础包）

```23:36:scripts/preloadsounds.lua
local MainSounds =
{
	"bat.fsb",
	"bee.fsb",
	"beefalo.fsb",
	"birds.fsb",
	"bunnyman.fsb",
	"cave_AMB.fsb",
	"cave_mem.fsb",
	"chess.fsb",
	"chester.fsb",
	"common.fsb",
	"deerclops.fsb",
	"dontstarve.fev",
```

`dontstarve.fev` 是主事件定义文件，它把 `wilson.fsb`、`sfx.fsb`、`common.fsb` 等**分散的 .fsb** 作为样本库引用。所有以 `"dontstarve/"` 开头的事件路径（如 `"dontstarve/characters/wilson/talk_LP"`）都定义在这个 `.fev` 里。

**模式二：配对 .fev + .fsb**（新格式，DLC/大更新）

```149:153:scripts/preloadsounds.lua
	"wanda2.fev",
    "wanda2.fsb",

    "wanda1.fev",
    "wanda1.fsb",
```

`wanda2.fev` + `wanda2.fsb` 是一对自包含的 bank——事件定义和音频数据全在里面。以 `"wanda2/"` 开头的事件路径（如 `"wanda2/characters/wanda/talk_old_LP"`）都定义在 `wanda2.fev` 里。

#### 核心规律：bank 名 = 事件路径前缀

```
bank 文件名（不含扩展名）= 事件路径的第一段

dontstarve.fev  →  事件路径前缀 "dontstarve/"
wanda2.fev      →  事件路径前缀 "wanda2/"
monkeyisland.fev →  事件路径前缀 "monkeyisland/"
```

**给 mod 的直接推论**：
- 若你的 mod 声音文件是 `mymod.fev`
- 那么里面的事件路径必须以 `"mymod/"` 开头
- 代码里就用 `"mymod/characters/mychar/talk_LP"` 引用

#### 游戏如何加载声音文件

vanilla 的声音文件由 `PreloadSounds()` 统一加载：

```261:264:scripts/preloadsounds.lua
function PreloadSoundList(list)
	for i,v in pairs(list) do
		TheSim:PreloadFile("sound/"..v)
	end
end
```

`TheSim:PreloadFile("sound/"..v)` 是 C++ 绑定——它在游戏启动时把 `.fev`/`.fsb` 加载进 FMOD 引擎。**mod 的声音文件不走这个流程**，而是通过 `Asset("SOUND", ...)` 声明，由引擎在加载 mod 时处理。

---

### 15.5.2（新手）路线 A/B：不制作 bank 就能用的方法

#### 路线 A：借用已有角色音效（推荐给无音效需求的角色 mod）

最简单的方案——直接告诉引擎"我的角色用 XX 的声音"：

```lua
-- master_postinit 中
inst.soundsname = "willow"       -- 借用 Willow 的所有标准角色音效
-- 也可以用其他角色：
-- inst.soundsname = "wendy"
-- inst.soundsname = "woodie"
-- ...
```

效果：
- 说话音：`dontstarve/characters/willow/talk_LP`
- 受伤音：`dontstarve/characters/willow/hurt`
- 死亡音：`dontstarve/characters/willow/death_voice`
- 打哈欠：`dontstarve/characters/willow/yawn`

**不需要任何额外文件**，只要你的 mod 加载了包含 Willow 音效的 bank（`willow.fsb` 由游戏主程序预加载，无需 mod 单独声明）。

#### 路线 B：直接播放 vanilla 音效路径

如果你的物品/NPC 需要播放某个 vanilla 音效但不是角色标准音，直接传路径就行：

```lua
-- 使用篝火的噼啪声
inst.SoundEmitter:PlaySound("dontstarve/common/fireBurstSmall")

-- 使用环境音
inst.SoundEmitter:PlaySound("dontstarve/common/rain_splash_LP", "rain")

-- 在 SGwilson.lua 的 TimeEvent 中
TimeEvent(4*FRAMES, function(inst)
    inst.SoundEmitter:PlaySound("dontstarve/creatures/spider/walk")
end)
```

**不需要额外文件**——vanilla bank 已经由游戏加载，直接引用路径即可。

但这条路线的**限制**是：你只能用已经存在的 vanilla 音效，无法添加新声音。

#### 需要哪些 vanilla 音效路径？

所有 vanilla 音效路径都定义在 `sound/dontstarve.fev`（以及 DLC 的 `.fev`）里。你可以通过：
1. 翻看 `scripts/stategraphs/SGwilson.lua` 中的 `PlaySound` 调用来找路径
2. 翻看 `scripts/components/` 中各组件的音效调用
3. 使用游戏的 `SOUNDDEBUG_ENABLED = true` 模式（仅 dev 分支）

常用 vanilla 音效路径示例：

| 路径 | 描述 |
|---|---|
| `"dontstarve/characters/wilson/talk_LP"` | Wilson 说话循环音 |
| `"dontstarve/common/fireBurstSmall"` | 小型点火音 |
| `"dontstarve/common/fireOut"` | 熄灭音 |
| `"dontstarve/creatures/spider/walk"` | 蜘蛛脚步 |
| `"dontstarve/sanity/sanity"` | 精神低时耳鸣 |
| `"dontstarve/common/recipe/make"` | 合成成功音 |
| `"dontstarve/HUD/research_up"` | 科技点获得音 |

---

### 15.5.3（新手）Asset("SOUND") —— 声明声音依赖的正确方式

#### Asset("SOUND") 的作用

`Asset("SOUND", path)` 告诉游戏引擎：**在加载使用这个 asset 的 prefab 时，需要预先加载这个声音文件**。

```lua
-- 声明一个 .fsb 依赖（样本 bank）
Asset("SOUND", "sound/wx78.fsb"),

-- 声明 .fev + .fsb 配对（自包含 bank）
Asset("SOUND", "sound/wanda2.fev"),
Asset("SOUND", "sound/wanda2.fsb"),
```

这些声明放在 prefab 文件的 `local assets = { ... }` 表里：

```7:11:scripts/prefabs/wx78.lua
local assets = JoinArrays({
    Asset("SCRIPT", "scripts/prefabs/player_common.lua"),
    Asset("SCRIPT", "scripts/prefabs/wx78_common.lua"),

    Asset("SOUND", "sound/wx78.fsb"),
```

或者放在 `modmain.lua` 的全局 `Assets` 表里（适合 mod 级别的声明）：

```lua
-- modmain.lua
Assets =
{
    Asset("SOUND", "sound/mymod.fev"),
    Asset("SOUND", "sound/mymod.fsb"),
}
```

#### 路径的根目录规则

`Asset("SOUND", "sound/xxx")` 中的路径：
- **vanilla 中**：相对于游戏数据目录（`data/`）
- **mod 中**：相对于 mod 的根目录

所以 mod 的声音文件应该放在：
```
mods/
└── 你的mod名/
    ├── modmain.lua
    ├── sound/
    │   ├── mymod.fev
    │   └── mymod.fsb
    └── scripts/
        └── ...
```

#### 什么时候用 prefab 的 assets vs modmain 的 Assets？

| 位置 | 适合场景 |
|---|---|
| prefab 的 `local assets = {}` | 音效只被该 prefab 用到；按需加载，节省内存 |
| `modmain.lua` 的 `Assets = {}` | 音效被多个 prefab 或全局代码共享；mod 启动就加载 |

对于角色 mod（多个状态、多个动作都用同一个 bank），推荐放在 `modmain.lua` 的 `Assets` 里确保启动时就加载完毕。

---

### 15.5.4（进阶）FMOD Studio 工作流 —— 安装、创建 bank、命名事件

> **前提条件**：需要安装 **FMOD Studio 1.x**（饥荒使用 FMOD Studio 1.x，不兼容 FMOD Studio 2.x）。官方推荐 1.10.x 版本。

#### 步骤 1：创建 FMOD Studio 项目

1. 打开 FMOD Studio → **File → New Project**
2. 项目命名不影响 bank 名，随意命名（如 `MyModSounds`）

#### 步骤 2：创建 Bank 并命名

**Bank 的命名决定了事件路径的前缀**：

1. 在 FMOD Studio 右侧 **Banks** 面板，右键 → **Add Bank**
2. 将 bank 命名为你的 mod 前缀名（如 `mymod`）
3. 之后导出时会生成 `mymod.bank`（FMOD 2.x 后缀）或 `mymod.fev`/`mymod.fsb`（FMOD 1.x 格式）

> **注意**：饥荒使用 FMOD Studio 1.x，导出格式是 `.fev` + `.fsb`，而非 FMOD Studio 2.x 的 `.bank` 格式。

#### 步骤 3：创建事件（Events）

1. 左侧 **Events** 面板 → 右键 → **New Event**
2. 事件的路径命名遵循以下规则：

```
事件路径 = "<bank名>/<分类>/<具体事件>"
例：mymod/characters/mychar/talk_LP
    mymod/items/myitem/pickup
    mymod/ambience/forest_theme_LP
```

**角色标准音效的命名约定**（与 SGwilson.lua 中的路径模式对应）：

| 功能 | 推荐事件路径 | 是否循环 |
|---|---|---|
| 说话循环音 | `mymod/characters/mychar/talk_LP` | 是（Loop）|
| 受伤音 | `mymod/characters/mychar/hurt` | 否 |
| 死亡音 | `mymod/characters/mychar/death_voice` | 否 |
| 打哈欠 | `mymod/characters/mychar/yawn` | 否 |
| 说话结束音 | `mymod/characters/mychar/end` | 否 |

> **"_LP" 后缀是约定**——FMOD 中循环事件和非循环事件名一样加，只是便于区分（实际循环由事件属性决定，不由名称后缀决定）。

#### 步骤 4：添加音频素材

1. 将你的 `.wav` / `.ogg` 文件拖入 FMOD Studio 的 **Audio Bin**
2. 在事件编辑器中，将音频素材添加到对应 Track
3. 对于循环音，在 Timeline 上右键 → **Loop Region** 设置循环区间
4. 对于随机音（受伤声多种），使用 **Multi Instrument** 或 **Randomize** 工具

#### 步骤 5：将事件分配到 Bank

1. 右键事件 → **Assign to Bank** → 选择你创建的 `mymod` bank
2. 未分配到 bank 的事件**不会被导出**

---

### 15.5.5（进阶）导出 bank 与 mod 目录结构

#### 导出设置

在 FMOD Studio 1.x 中：

1. **File → Build** 或 **File → Export GUIDs**
2. 导出目标格式选择 **Desktop（PC）**
3. 导出目录设置为 mod 的 `sound/` 目录

导出后会得到：
```
sound/
├── mymod.fev     ← 事件定义文件（文本，可用文本编辑器查看事件路径）
└── mymod.fsb     ← 音频数据文件（二进制压缩）
```

#### 完整 mod 目录结构

```
mods/
└── workshop-XXXXXXXXX/        ← mod 根目录
    ├── modinfo.lua
    ├── modmain.lua
    ├── sound/
    │   ├── mymod.fev          ← FMOD 事件定义
    │   └── mymod.fsb          ← FMOD 音频数据
    ├── scripts/
    │   ├── prefabs/
    │   │   └── mychar.lua
    │   └── speech_mychar.lua
    ├── images/
    │   └── ...
    └── anim/
        └── ...
```

#### modinfo.lua 中的说明（可选）

modinfo.lua 本身不需要特别声明音效文件，但建议在描述中注明：

```lua
-- modinfo.lua（无需特殊声明，Asset声明在modmain.lua）
name = "My Character"
description = "A custom character with custom sounds."
author = "Your Name"
version = "1.0"
```

#### modmain.lua 中的 Asset 声明

```lua
-- modmain.lua
Assets =
{
    -- 声音文件（必须声明才会被加载）
    Asset("SOUND", "sound/mymod.fev"),
    Asset("SOUND", "sound/mymod.fsb"),

    -- 其他资源...
    Asset("ANIM", "anim/mychar.zip"),
    Asset("IMAGE", "images/saveslot_portraits/mychar.tex"),
    Asset("ATLAS", "images/saveslot_portraits/mychar.xml"),
}
```

---

### 15.5.6（进阶）在代码里使用自定义音效路径

#### 在 prefab 中配置角色语音

有了自己的 bank 后，在 `master_postinit` 中设置路径覆盖：

```lua
local function master_postinit(inst)
    -- 指向自己 bank 的路径前缀
    inst.talker_path_override = "mymod/characters/"
    -- soundsname 默认使用 inst.prefab
    -- 最终拼成：mymod/characters/mychar/talk_LP

    -- 若 prefab 名和事件目录名不一致，用 soundsname 修正
    -- inst.soundsname = "mychar_v2"
end
```

#### 手动调用音效（非标准角色音）

对于不走 SGwilson 模板的音效（物品音、技能音、特效音），直接调用：

```lua
-- 一次性音效
inst.SoundEmitter:PlaySound("mymod/items/myitem/pickup")

-- 持续循环音（需要 name handle 来停止）
inst.SoundEmitter:PlaySound("mymod/ambience/loop_LP", "myambience")
-- ... 之后停止
inst.SoundEmitter:KillSound("myambience")

-- 带参数的音效（在 FMOD Studio 中定义了参数的事件）
inst.SoundEmitter:PlaySoundWithParams("mymod/combat/attack", { charge = 0.8 })
```

#### 检查 bank 是否被正确加载

如果 bank 未加载就调用 PlaySound，游戏不会崩溃但也不会出声，且控制台会有警告。调试时可以：

1. 在 `modmain.lua` 顶部临时加打印，确认 Asset 声明顺序正确
2. 检查 mod 目录下 `sound/` 文件夹里文件名是否和 Asset 声明一致（**大小写敏感**）
3. 使用 `SoundEmitter:PlayingSound(name)` 检查是否在播放
4. 检查 FMOD Studio 导出时是否选择了正确的 bank 和正确的目标平台

#### 路径大小写问题

Asset 路径和 PlaySound 路径都**大小写敏感**（Windows 开发环境可能不报错，但 Linux 服务器会）。建议统一使用小写字母 + 下划线：

```lua
-- 推荐（全小写，下划线分隔）
Asset("SOUND", "sound/mymod.fev")
PlaySound("mymod/characters/mychar/talk_lp")  -- 注意：vanilla 用 _LP 大写，mod 自定义可以一致

-- 避免（大小写混用）
Asset("SOUND", "sound/MyMod.fev")  -- 在 Linux 服务器上可能找不到
```

---

### 15.5.7（老手）RemapSoundEvent —— 全局替换 vanilla 音效

#### RemapSoundEvent 是什么

`RemapSoundEvent` 是 mod API，可以在全局层面把一个 FMOD 事件路径重映射到另一个：

```842:850:scripts/modutil.lua
	env.RemapSoundEvent = function(name, new_name)
		initprint("RemapSoundEvent", name, new_name)
		TheSim:RemapSoundEvent(name, new_name)
	end

	env.RemoveRemapSoundEvent = function(name) -- Convenience wrapper.
		initprint("RemoveRemapSoundEvent", name)
		TheSim:RemapSoundEvent(name) -- Other second parameter values may be nil / the first parameter.
	end
```

`TheSim:RemapSoundEvent(old, new)` 是 C++ 绑定——FMOD 引擎收到 `old` 路径的播放请求时，会自动播放 `new` 路径的事件。

#### 典型应用场景

**场景 1：给 mod 角色替换系统角色音**

如果你的 mod 角色用 Wilson 的 prefab 名（不可能），或者你想要替换全局 Wilson 音效（慎用，会影响所有玩 Wilson 的玩家）：

```lua
-- modmain.lua（极端案例，影响全局）
RemapSoundEvent("dontstarve/characters/wilson/talk_LP", "mymod/characters/mychar/talk_lp")
```

**场景 2：替换某个物品的 vanilla 音效**

```lua
-- 将火堆的小点火音替换为自定义音
RemapSoundEvent("dontstarve/common/fireBurstSmall", "mymod/sfx/magical_fire_burst")
```

**场景 3：mod 加载时替换，卸载时恢复**

```lua
-- modmain.lua
RemapSoundEvent("dontstarve/sfx/old_path", "mymod/sfx/new_path")

-- 若需要在特定时机恢复：
-- RemoveRemapSoundEvent("dontstarve/sfx/old_path")
```

#### 注意事项

1. **全局生效**：`RemapSoundEvent` 影响所有客户端上所有对该路径的调用，包括其他玩家的体验。慎重使用！
2. **需要目标 bank 已加载**：new_name 指向的 bank 必须已经通过 `Asset("SOUND", ...)` 加载
3. **不可用于字符级定向替换**：如果只想替换"当玩某个角色时的音效"，请用 `talksoundoverride`/`soundsname` 而不是 `RemapSoundEvent`

---

### 15.5.8（老手）完整 mod 音效实战 —— 从 FMOD 到 PlaySound

#### 目标场景

给一个名为 `"mychar"` 的 mod 角色添加以下自定义音效：
- 说话循环音（独特的机械嗡嗡声）
- 受伤音（两种随机选取）
- 一个特殊技能音效（使用技能时播放）

#### 第 1 步：在 FMOD Studio 创建 bank

1. 新建项目，创建 bank 命名为 `mymod`
2. 创建以下事件：

```
事件层级：
mymod/
├── characters/
│   └── mychar/
│       ├── talk_LP         ← 循环，添加机械嗡嗡素材
│       ├── hurt            ← 单发，Multi Instrument（2种随机受伤声）
│       ├── death_voice     ← 单发，死亡音效
│       └── yawn            ← 单发，疲劳音效
└── skills/
    └── mychar/
        └── special_ability ← 单发，技能释放音效
```

3. 将所有事件分配到 `mymod` bank
4. 导出为 `sound/mymod.fev` + `sound/mymod.fsb`

#### 第 2 步：mod 文件结构

```
mods/workshop-mymod/
├── modmain.lua
├── sound/
│   ├── mymod.fev
│   └── mymod.fsb
└── scripts/
    ├── prefabs/
    │   └── mychar.lua
    └── speech_mychar.lua
```

#### 第 3 步：modmain.lua 声明 Assets

```lua
-- modmain.lua
PrefabFiles = { "mychar" }

Assets =
{
    Asset("SOUND", "sound/mymod.fev"),
    Asset("SOUND", "sound/mymod.fsb"),
    -- 其他 assets...
}

local STRINGS = GLOBAL.STRINGS
STRINGS.CHARACTERS.MYCHAR = require "speech_mychar"
STRINGS.NAMES.MYCHAR = "My Character"
```

#### 第 4 步：在 prefab 中配置语音音效

```lua
-- scripts/prefabs/mychar.lua
local MakePlayerCharacter = require "prefabs/player_common"

local assets =
{
    Asset("SCRIPT", "scripts/prefabs/player_common.lua"),
    -- 角色特有 assets（sound 在 modmain 中声明即可）
}

local function master_postinit(inst)
    -- 数值
    inst.components.health:SetMaxHealth(150)
    inst.components.hunger:SetMax(150)
    inst.components.sanity:SetMax(200)

    -- 语音音效：指向自己的 bank
    inst.talker_path_override = "mymod/characters/"
    -- soundsname 不设，默认用 "mychar"
    -- → talk_LP: mymod/characters/mychar/talk_LP
    -- → hurt:    mymod/characters/mychar/hurt
    -- → death:   mymod/characters/mychar/death_voice

    -- talker 样式
    inst.components.talker.fontsize = 35
    inst.components.talker.font = GLOBAL.TALKINGFONT
    inst.components.talker.colour = GLOBAL.Vector3(0.5, 0.8, 1.0)
end

local function common_postinit(inst)
    inst.MiniMapEntity:SetIcon("mychar.tex")
end

return MakePlayerCharacter("mychar", {}, assets, common_postinit, master_postinit)
```

#### 第 5 步：在技能代码中调用特殊音效

```lua
-- 在某个组件或事件回调中
local function OnUseSpecialAbility(inst)
    if inst.SoundEmitter then
        inst.SoundEmitter:PlaySound("mymod/skills/mychar/special_ability")
    end
    -- ... 技能逻辑
end

inst:ListenForEvent("special_ability", OnUseSpecialAbility)
```

#### 验证清单

在测试时，依次核查：
- [ ] `sound/` 目录下 `.fev` 和 `.fsb` 文件名与 `Asset()` 声明完全一致（大小写）
- [ ] FMOD Studio 中事件路径前缀与 bank 名一致（`mymod/...`）
- [ ] `talker_path_override` 末尾有 `/`（`"mymod/characters/"` 而非 `"mymod/characters"`）
- [ ] mod 角色发音时控制台无 `FMOD ERROR` 输出
- [ ] 多人模式下其他玩家也能听到声音（bank 在客户端侧加载）

---

### 15.5.9（小结）速查表 + 常见错误 + 本章总结

#### 四条路线速查

| 路线 | 所需文件 | 适合场景 | 代码量 |
|---|---|---|---|
| **A：借用角色音效** | 无需额外文件 | mod 角色，声音要求不高 | 1 行：`soundsname = "xxx"` |
| **B：直接引用 vanilla 路径** | 无需额外文件 | 借用物品/环境音效 | `PlaySound("dontstarve/...")` |
| **C：RemapSoundEvent** | 自己的 .fev/.fsb | 全局替换 vanilla 音效 | modmain 1~2 行 |
| **D：完全自定义 bank** | 自己的 .fev/.fsb | 全新音效需求 | FMOD Studio + Asset 声明 |

#### Asset 声明速查

| 需求 | 声明方式 |
|---|---|
| 仅依赖 vanilla fsb（如 sfx.fsb）| `Asset("SOUND", "sound/sfx.fsb")` |
| 自定义配对 bank | `Asset("SOUND", "sound/mymod.fev")` + `Asset("SOUND", "sound/mymod.fsb")` |
| mod 全局音效（多 prefab 共用）| 放在 `modmain.lua` 的 `Assets = {}` 里 |
| prefab 独占音效 | 放在 prefab 文件的 `local assets = {}` 里 |

#### bank 命名与路径对应速查

| bank 文件名 | 对应事件路径前缀 | vanilla 实例 |
|---|---|---|
| `dontstarve.fev` | `dontstarve/` | `dontstarve/characters/wilson/talk_LP` |
| `wanda2.fev` | `wanda2/` | `wanda2/characters/wanda/talk_old_LP` |
| `monkeyisland.fev` | `monkeyisland/` | `monkeyisland/characters/wonkey/talk_LP` |
| `mymod.fev` | `mymod/` | `mymod/characters/mychar/talk_LP` |

#### 常见错误与解决

| 错误现象 | 可能原因 | 解决方案 |
|---|---|---|
| 说话时无声，无 FMOD ERROR | bank 路径或事件路径打错 | 检查大小写；用日志打印确认路径 |
| FMOD ERROR: event not found | bank 未加载 / 事件名拼错 | 确认 Asset() 声明 + 事件命名 |
| 只有本地有声，联机他人无声 | bank 未在客户端侧加载 | 将 Asset 声明移至 modmain Assets 表，而非仅服务端 prefab |
| FMOD Studio 导出的 .bank 不识别 | 使用了 FMOD Studio 2.x | 饥荒需要 FMOD Studio 1.x（1.10.x 推荐）|
| Linux/Mac 服务器找不到文件 | 文件名大小写问题 | 统一全小写文件名 |

#### 本节 & 本章总结

```
15.1 → SoundEmitter 组件 —— entity 发声的统一接口（PlaySound/KillSound/SetParameter）
15.2 → PlaySoundWithParams —— FMOD 参数化音效（动态强度、状态切换）
15.3 → DynamicMusic + AmbientSound —— BGM 与环境音的全局系统
15.4 → 角色语音系统 —— speech 文件 + GetString + talker:Say + 音效路径
15.5 → 自定义音效文件 —— .fev/.fsb 体系 + FMOD Studio 工作流 + RemapSoundEvent
```

**完整音效开发路线（15.5 角度）**：
1. **需求判断** → 是借音（路线 A/B）还是全新音效（路线 D）？
2. **路线 A/B**：设 `soundsname`、直接写 vanilla 路径，0 额外文件
3. **路线 D**：FMOD Studio 1.x → 创建 bank（名与路径前缀一致）→ 导出 `.fev`+`.fsb` → 放 `sound/` → `Asset("SOUND", ...)` 声明 → `talker_path_override` + 代码引用
4. **RemapSoundEvent**：有已有 bank 且想替换 vanilla 路径时使用

---
