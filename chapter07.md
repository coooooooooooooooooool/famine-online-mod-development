# 第7章 Action（动作）系统

## 7.1 ACTIONS 表的结构与注册机制

### 本节导读

第 5 章我们把"图纸"（Prefab）讲透了，第 6 章把"能力"（Component）讲透了——**但有一个问题始终没解决**：玩家右键点击一棵松树、引擎是怎么认识到"要做 CHOP（砍）"的？玩家按下空格键、为什么有时候是"攻击"、有时候是"捡起"、有时候是"对话"？玩家把胡萝卜拖到威尔逊嘴上、引擎怎么知道是"喂玩家"而不是"丢东西"？

这些"**操作意图的识别**"，全部由饥荒的 **Action（动作）系统** 完成。它是**玩家输入 → 实体行为**的中间层——从一个鼠标点击或按键，最终变成 `inst.components.weapon:Attack(target)`、`inst.components.workable:WorkedBy(...)`、`inst.components.eater:Eat(...)` 这种具体方法调用，**中间走过的所有路径**就是 Action 系统。

这一节我们打开 Klei 的"动作字典"——`scripts/actions.lua` 这个 6800 多行的核心文件——把它的**整体结构**和**注册机制**讲清楚：

> **新手**从 7.1.1-7.1.3 起步——理解一次"砍树"在代码里是什么、`ACTIONS` 这个全局表长什么样、Action 类有哪些核心字段；**进阶读者**继续看 7.1.4-7.1.6，深入"动作编码"`ACTIONS_BY_ACTION_CODE` 的存在原因、原版 Action 与 Mod Action 的隔离机制、`fn / strfn / validfn / canforce / rangecheckfn` 这些回调各自的分工；**老手**跳到 7.1.7-7.1.8，逐行拆解 `AddAction` 的源码、列出六个最容易踩的坑——尤其是写自定义动作时**网络同步、动作冲突、字符串本地化**这三个最容易翻车的领域。

---

### 7.1.1 快速入门：从一次"砍树"看 Action 是什么

#### 第一步：在游戏里观察一次完整的"砍树"

打开饥荒联机版，找一棵松树，**手里拿着斧头**，**左键点击树**——

**你看到的**：

1. 角色走到树前
2. 角色挥起斧头
3. 砍击音效
4. 树上掉一片树叶飘落特效
5. 树的"健康度"减 1
6. 重复几次后树倒下、地上多了几根木头

**引擎在背后做了什么？**

让我们用控制台搞清楚——按 `Backquote`（反引号 `~`）打开控制台，然后在树倒下前**对着一棵松树**输入：

```lua
print(ThePlayer.bufferedaction)
```

你会看到类似这样的输出：

```
BufferedAction CHOP - tree_normal[123456]
```

**`CHOP`** 就是当前玩家正在执行的 Action 名字。**`tree_normal`** 是这个动作的目标。**整个 `BufferedAction` 是一个把"动作 + 执行者 + 目标 + 工具 + 是否成功"打包起来的对象**——它就是 Action 系统的"运行时载体"，我们 7.4 节会专门讲它。

**核心结论**：**饥荒里玩家做的每一件事——砍树、挖矿、捡花、攻击、吃饭——都是一个 Action**。整个游戏的"动词词典"被收纳在一个全局表 **`ACTIONS`** 里。

#### 第二步：拆开"砍树"的代码路径

我们点击树的那一刻，引擎走过的路径**简化版**是这样的：

```
[1] 玩家左键点击树
       ↓
[2] PlayerController 检测到点击事件，根据"手里拿着斧头 + 目标是树"
    决定要执行的动作 → ACTIONS.CHOP
       ↓
[3] 创建一个 BufferedAction(玩家, 树, ACTIONS.CHOP, 斧头)
       ↓
[4] 玩家走过去（路径动作 LOCOMOTE）
       ↓
[5] 到达后，StateGraph 切到 chop_loop 状态——播放砍击动画
       ↓
[6] 动画里的"接触帧"触发 ACTIONS.CHOP.fn(act) 这个动作回调
       ↓
[7] CHOP.fn 内部调用 act.target.components.workable:WorkedBy(act.doer, 1)
       ↓
[8] workable 组件减少树的工作进度，到 0 时 PushEvent("worked")
       ↓
[9] 树的 prefab 监听这个事件 → 倒下 + 生成木头
```

**整条链路里 Action 起的作用**：

- **第 [2] 步**：决定**要做什么**（识别动作意图）
- **第 [6] 步**：执行**核心业务**（调用具体组件方法）

**Action 既是"决策的依据"，也是"执行的入口"** —— 它不做任何具体业务（比如"减 1 工作进度"那是 `workable` 组件的事），它只**把决策和执行串起来**。

#### 第三步：看一眼 `ACTIONS.CHOP` 的真身

打开 `scripts/actions.lua`，第 342 行：

```342:342:scripts/actions.lua
	CHOP = Action({ distance=1.75, invalid_hold_action=true }),
```

**就一行！** 一个 `Action` 类的实例，配置了两个字段：`distance=1.75`（必须靠近 1.75 距离才能砍）、`invalid_hold_action=true`（按住鼠标不能连续触发，必须每次单独点击）。

然后第 1690 行：

```1690:1703:scripts/actions.lua
ACTIONS.CHOP.fn = function(act)
    local work_success, work_fail_reason = DoToolWork(act, ACTIONS.CHOP)
    if work_success and
            act.doer ~= nil and
            act.doer.components.spooked ~= nil and
            act.target:IsValid() then
        act.doer.components.spooked:Spook(act.target)
    end
    if not work_success and work_fail_reason ~= nil then
        return false, work_fail_reason
    else
        return true
    end
end
```

**这就是 `CHOP` 的"执行函数"** —— 它接收一个 `act`（BufferedAction 实例），调用 `DoToolWork` 通用工具函数（内部会去找 `target.components.workable`），如果成功就返回 `true`，失败就返回 `false, 原因`。

**`act` 这个参数包含什么？** —— 至少有 `act.doer`（执行者）、`act.target`（目标）、`act.invobject`（手里拿的物品）、`act.pos`（如果是定位动作）。这些就是动作执行所需的"上下文"，由 BufferedAction 在创建时填好。

> **新手记忆**：**Action 是一个"动词"对象**，挂在全局 `ACTIONS` 表里，每个动作有：① 一些**配置参数**（距离、是否右键、能否远程指定）、② 一个 **`fn` 执行函数**（真正干活）。整个游戏的**所有动作不过 100 多个**——掌握这张表的结构，你就掌握了饥荒"玩家能做什么"的全貌。

---

### 7.1.2 快速入门：`ACTIONS` 全局表 —— 它在哪里、长什么样

#### 第一步：定位 `ACTIONS` 表的源码位置

`ACTIONS` 这个全局变量定义在 **`scripts/actions.lua`** 第 336 行：

```336:338:scripts/actions.lua
ACTIONS =
{
    REPAIR = Action({ mount_valid=true, encumbered_valid=true }),
```

**它是一个 Lua table（哈希表）**——key 是动作名字符串（大写）、value 是一个 `Action` 类的实例。

整个 ACTIONS 表的**初始化代码占了大约 370 行**（从 336 到 708 行），里面有约 **180+ 个动作**。让我们把最常见的列出来：

#### 第二步：原版 ACTIONS 表里有哪些动作？

按"用途分类"看：

**① 工具/采集类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `CHOP` | `distance=1.75` | 砍树（用斧/路法） |
| `MINE` | `invalid_hold_action=true` | 挖矿/敲岩石（用镐子） |
| `DIG` | `rmb=true` | 挖坟/掘地（用铲子） |
| `HAMMER` | `priority=3` | 拆建筑（用锤子） |
| `PICK` | `canforce=true, rangecheckfn=PickRangeCheck` | 摘花/摘草/拔萝卜 |
| `PICKUP` | `priority=1` | 捡起地上的物品 |
| `NET` | `priority=3, canforce=true` | 用网捕蝴蝶/萤火虫 |
| `FISH` / `FISH_OCEAN` / `OCEAN_FISHING_CAST` / `OCEAN_FISHING_REEL` | 多种 | 钓鱼相关的一串动作 |

**② 战斗类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `ATTACK` | `priority=2, canforce=true, mount_valid=true` | 普通攻击（左键） |
| `BAIT` | 默认 | 给陷阱放诱饵 |
| `CHECKTRAP` | `priority=2, mount_valid=true` | 检查陷阱（取出战利品） |

**③ 物品交互类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `EAT` | `mount_valid=true, floating_valid=true` | 吃食物 |
| `EQUIP` | `priority=0, instant=true, mount_valid=true` | 装备 |
| `UNEQUIP` | `priority=-2, instant=true, mount_valid=true` | 卸下装备 |
| `DROP` | `priority=-1, mount_valid=true` | 丢掉手里物品 |
| `STORE` | `mount_valid=true` | 存进容器 |
| `RUMMAGE` | `priority=-1, mount_valid=true` | 翻箱子 / 检查容器 |
| `GIVE` | `mount_valid=true, canforce=true` | 给某个 NPC（合成、献祭） |
| `FEEDPLAYER` | `priority=3, rmb=true, canforce=true` | 喂另一个玩家 |

**④ 检视/对话类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `LOOKAT` | `priority=-3, instant=true, distance=3, ghost_valid=true` | 检查（角色说一句话） |
| `TALKTO` | `priority=3, instant=true, mount_valid=true` | 跟 NPC 对话 |
| `WALKTO` | `priority=-4, ghost_valid=true` | 单纯走过去（没有别的动作匹配时的兜底） |

**⑤ 建造/部署类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `BUILD` | `mount_valid=true` | 合成物品 |
| `DEPLOY` | `distance=1.1, mount_valid=true` | 部署（放下种子、放下围墙等） |
| `TERRAFORM` | `tile_placer="gridplacer"` | 用泥铲改变地形 |

**⑥ 火/燃料类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `LIGHT` | `priority=-4` | 点火 |
| `EXTINGUISH` | `priority=0` | 浇灭 |
| `STOKEFIRE` | `rmb=true, distance=8` | 拨火（让火更旺） |
| `ADDFUEL` | `mount_valid=true` | 添加燃料（木头进营火） |
| `ADDWETFUEL` | `mount_valid=true` | 添加湿燃料 |

**⑦ 农业/烹饪类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `PLANT` | 默认 | 种植 |
| `HARVEST` | 默认 | 收获 |
| `COOK` | `priority=1, mount_valid=true` | 烹饪 |
| `FERTILIZE` | `priority=1, mount_valid=true` | 施肥 |
| `POLLINATE` | 默认 | 蜜蜂授粉（NPC 用） |

**⑧ 移动/交通类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `JUMPIN` | `ghost_valid=true` | 跳进虫洞/天体之门 |
| `TRAVEL` | 默认 | 进入下一个分片（cave/forest 切换） |
| `GOHOME` | 默认 | NPC 回家（猪人晚上） |

**⑨ 地图类（DST 新增）**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `MAPDELIVER_MAP` | `map_only=true, closes_map=true` | 在地图上规划送货 |
| `JUMPIN_MAP` | `map_action=true, closes_map=true` | 在地图上选传送目标 |

**⑩ 生命周期/特殊类**：

| Action | 配置 | 含义 |
|--------|-----|-----|
| `MURDER` | 默认 | 杀掉手里的小动物（小兔子等） |
| `RESURRECT` | 默认 | 复活别的玩家（用复活雕像） |
| `REVIVE_CORPSE` | 默认 | 复活鬼魂（用心花） |

> **新手提示**：**完整 180+ 个动作不需要全记**——常用的有 30-40 个，其他都是"特殊场景的极小众动作"（比如 `OCEAN_TRAWLER_LOWER`、`PIRATE_TICKLE`）。**记住这 30-40 个就够支撑 90% 的 Mod 开发**。

#### 第三步：观察"配置参数"的规律

仔细看刚才那些表格里的配置——你会发现 Klei 在反复用同一组参数：

```lua
priority           -- 优先级（决定多个动作可选时哪个胜出）
distance           -- 默认距离（玩家要走多近才能执行）
rmb                -- 是否用右键触发
mount_valid        -- 骑牛时能不能做（吃可以、砍树不行）
encumbered_valid   -- 拿着大件物品时能不能做（雕像在手里不能砍树）
ghost_valid        -- 鬼魂状态能不能做（看世界可以、攻击不行）
floating_valid     -- 在浮板上能不能做
canforce           -- 能否远程"强制"指定（按住 Shift + 点击）
rangecheckfn       -- 自定义距离检查函数
instant            -- 是否瞬间完成（不需要走过去）
invalid_hold_action  -- 按住能不能连发
```

**这些字段就是 Action 类的"DSL 词汇"**——一个动作怎么"工作"由这十几个参数描述。**你要写自定义动作时，最常做的事就是组合这些参数**。

下一节我们逐个讲清楚每个字段的含义。

> **进阶提示**：在控制台输入 `for k, v in pairs(ACTIONS) do print(k) end` —— 你能在游戏里看到完整的动作列表（包括所有装载的 Mod 添加的动作）。**这是查看"当前游戏环境实际可用的动作集"的最快方式**——比读源码更准确，因为 Mod 加的动作只在 `AddAction` 之后才出现。

---

### 7.1.3 快速入门：Action 类的核心字段

`Action` 类的构造函数定义在 `actions.lua` 第 275-330 行——这是一个**"配置容器"**，所有字段都是从 `data` 表里读出来：

```275:330:scripts/actions.lua
Action = Class(function(self, data, instant, rmb, distance, ghost_valid, ghost_exclusive, canforce, rangecheckfn)
    if data == nil then
        data = {}
    end
    if type(data) ~= "table" then
        --#TODO: get rid of the positional parameters all together, this warning here is for mods that may be using the old interface.
        print("WARNING: Positional Action parameters are deprecated. Please pass action a table instead.")
        print(string.format("Action defined at %s", debugstack_oneline(4)))
        local priority = data
        data = {priority=priority, instant=instant, rmb=rmb, ghost_valid=ghost_valid, ghost_exclusive=ghost_exclusive, canforce=canforce, rangecheckfn=rangecheckfn}
    end

    self.priority = data.priority or 0
    self.fn = function() return false end
    self.strfn = nil
    self.instant = data.instant or false
    self.rmb = data.rmb or nil -- note! This actually only does something for tools, everything tests 'right' in componentactions
    self.distance = data.distance or nil
    self.mindistance = data.mindistance or nil
    self.arrivedist = data.arrivedist or nil
    self.ghost_exclusive = data.ghost_exclusive or false
    self.ghost_valid = self.ghost_exclusive or data.ghost_valid or false -- If it's ghost-exclusive, then it must be ghost-valid
    self.mount_valid = data.mount_valid or false
    self.encumbered_valid = data.encumbered_valid or false
	self.floating_valid = data.floating_valid or false
    self.canforce = data.canforce or nil
    self.rangecheckfn = self.canforce ~= nil and data.rangecheckfn or nil
    self.mod_name = nil
	self.silent_fail = data.silent_fail or nil
	self.silent_generic_fail = data.silent_generic_fail or nil

    --new params, only supported by passing via data field
    self.paused_valid = data.paused_valid or false
    self.actionmeter = data.actionmeter or nil
    self.customarrivecheck = data.customarrivecheck
    self.is_relative_to_platform = data.is_relative_to_platform
    self.disable_platform_hopping = data.disable_platform_hopping
    self.skip_locomotor_facing = data.skip_locomotor_facing
    self.do_not_locomote = data.do_not_locomote
    self.extra_arrive_dist = data.extra_arrive_dist
    self.tile_placer = data.tile_placer
    self.show_tile_placer_fn = data.show_tile_placer_fn
	self.theme_music = data.theme_music
	self.theme_music_fn = data.theme_music_fn -- client side function
    self.pre_action_cb = data.pre_action_cb -- runs on client and server
    self.invalid_hold_action = data.invalid_hold_action

    self.show_primary_input_left = data.show_primary_input_left
    self.show_secondary_input_right = data.show_secondary_input_right

    self.map_action = data.map_action -- Should only be handled from the map and has action translations.
    self.closes_map = data.closes_map -- Should immediately close the minimap on action start.
    self.map_only = data.map_only -- Action only exists from a map.
    self.map_works_on_unexplored = data.map_works_on_unexplored -- Bypass seeable checks.
    self.map_works_on_impassable = data.map_works_on_impassable -- Allow impassable tiles for selection.
end)
```

**总共 30 多个字段**——看起来吓人，但**绝大多数你永远用不到**。我们按"使用频率"分三档讲。

#### 第一档：90% 场景都要用的"核心 6 个字段"

**(1) `priority`（优先级）—— 默认 0**

**含义**：当多个动作同时可用时，谁胜出。**数字越大优先级越高**。

实例：玩家手里拿着草、目标既是"可摘的花"又是"可走过去的位置"——`PICK.priority=0`、`WALKTO.priority=-4`，所以 `PICK` 胜出。

**典型值**：

- `priority=10`（HIGH_ACTION_PRIORITY，高级别动作如 `JUMPIN_MAP`）
- `priority=3`（重要的玩家间动作如 `FEEDPLAYER`、`TALKTO`、`HAMMER`）
- `priority=2`（`ATTACK`、`CHECKTRAP`）
- `priority=1`（`PICKUP`、`COOK`）
- `priority=0`（默认值，如 `EAT`、`CHOP`、`MINE`）
- `priority=-1`（兜底类，如 `DROP`、`RUMMAGE`、`HITCHUP`）
- `priority=-3`（更弱的兜底，如 `LOOKAT`）
- `priority=-4`（最弱兜底，如 `WALKTO`、`LIGHT`）

> **新手记忆**：**WALKTO 是最低优先级的兜底动作**——意思是"如果完全找不到其他动作，那就走过去"。这是为什么你点空地角色会走过去——`WALKTO` 在所有动作都不匹配时胜出。

**(2) `fn`（执行函数）—— 默认 `function() return false end`**

**含义**：动作真正"执行业务"的回调。接收一个 `act`（BufferedAction）参数，**返回值约定**：

- `return true` —— 成功
- `return false, "原因字符串"` —— 失败带本地化原因
- `return nil` —— 中性（既不算成功也不算失败，少见）

回顾 7.1.1 的 `ACTIONS.CHOP.fn`——返回 `true / false, reason` 的标准模式。

**(3) `distance`（执行距离）—— 默认 `nil`**

**含义**：玩家执行这个动作时，必须靠近目标到这个距离。`nil` 表示用默认距离（约 1.5 单位）。

例：`CHOP.distance=1.75`、`STOKEFIRE.distance=8`（远距离拨火）、`MARK.distance=2`、`LOOKAT.distance=3`。

**(4) `rmb`（右键触发）—— 默认 `nil`**

**含义**：是否用**右键**触发。注意源码里的注释："This actually only does something for tools, everything tests 'right' in componentactions"——**对于"工具类动作"（如 DIG、STOKEFIRE）这个字段决定操作方式，对其他大部分动作其实是在 `componentactions.lua` 里检查的**（7.2 节会讲）。

**(5) `instant`（瞬间完成）—— 默认 `false`**

**含义**：动作是否**不需要走过去就能完成**。

典型 `instant=true` 的动作：`LOOKAT`（说一句话不需要走过去）、`EQUIP` / `UNEQUIP`（装备/卸下不走路）、`TALKTO`（瞬间触发对话）。

**(6) `canforce`（强制指定）—— 默认 `nil`**

**含义**：玩家能否通过 **`Shift + 点击`** 远程"指定这个动作"。

例：`ATTACK.canforce=true`（按住 Shift 远程攻击）、`PICK.canforce=true`（远程标记摘花，角色会自动走过去）。**走路过去用的是后面 7.5 节会讲的 PlayerController 的"自动路径系统"**。

**配合 `rangecheckfn`** —— 如果 canforce=true，引擎会用 rangecheckfn 检查"这个目标真的能被远程指定吗"（防止玩家点屏幕外的怪兽）。

#### 第二档：开发自定义动作偶尔需要用的"6 个字段"

**(7) `mount_valid`（骑牛时可用）—— 默认 `false`**

**含义**：玩家骑在牛背上时这个动作能不能做。

典型 `mount_valid=true`：`EAT`、`PICKUP`、`ATTACK`（骑战斗）、`COOK`。**骑着不能做的动作**：`CHOP`（默认）、`MINE`（默认）—— 因为骑着挥斧子动画做不出来。

**(8) `encumbered_valid`（负重时可用）—— 默认 `false`**

**含义**：玩家拿着"大件物品"（如雕像、巨型礼物盒）时能不能做。

典型 `encumbered_valid=true`：`DROP`、`LOOKAT`（拿着雕像也能检查物品）、`WALKTO`（必须能走）。

**(9) `ghost_valid`（鬼魂可用）—— 默认 `false`**

**含义**：玩家死后变成鬼魂时能不能做。

典型 `ghost_valid=true`：`WALKTO`（鬼魂能漂移）、`LOOKAT`（鬼魂能看东西）、`HAUNT`（鬼魂专属动作）、`JUMPIN`（鬼魂能跳进天体之门）。

**`ghost_exclusive=true`** —— 鬼魂专属动作（活人不能做）。看 `HAUNT`、`REVIVE_CORPSE`。

**(10) `paused_valid`（暂停时可用）—— 默认 `false`**

**含义**：游戏暂停（PvE 服务器、控制台 `c_pause()`）时这个动作能不能继续。

典型：`EQUIP`、`UNEQUIP`、`ADDFUEL`——这些"轻量级动作"在暂停时也允许，不会影响游戏平衡。

**(11) `floating_valid`（浮板上可用）—— 默认 `false`**

**含义**：玩家站在浮板上时这个动作能不能做。

**(12) `invalid_hold_action`（不能按住）—— 默认 `nil`**

**含义**：是否**禁止"按住鼠标连发"**——必须每次单独点击。

例：`CHOP`、`MINE`、`HAMMER` 都是 true——避免玩家"按住一直砍"导致的体验/性能问题。

#### 第三档：少数特殊场景才用的"高阶字段"

剩下十几个字段都是某些特殊动作专用的：

- **`tile_placer` / `show_tile_placer_fn`**：动作准备阶段显示一个"地砖预览"（如 `TERRAFORM` 显示地形格子）
- **`do_not_locomote`**：动作完全不需要"走过去"（如 `OCEAN_FISHING_REEL` 在原地拉杆）
- **`customarrivecheck`**：自定义"已到达"判断（如 `FISH_OCEAN` 检查"水面位置"）
- **`extra_arrive_dist`**：到达时的额外距离补偿（如 `DEPLOY` 在水边时多让一点距离）
- **`map_action / map_only / closes_map`**：地图模式专属（DST 新增）
- **`actionmeter`**：动作进度条（钓鱼专用）
- **`silent_fail / silent_generic_fail`**：失败时不要让角色说话
- **`pre_action_cb`**：动作正式开始前调用的回调（client + server 都跑）
- **`disable_platform_hopping`**：动作执行期间禁止"跳到浮板"
- **`skip_locomotor_facing`**：到达后不需要面向目标
- **`is_relative_to_platform`**：动作的距离判断按相对浮板位置（钓鱼/挖矿在浮板上很有用）
- **`theme_music / theme_music_fn`**：动作期间播放主题音乐（农耕动作有 farming 音乐）
- **`map_works_on_unexplored / map_works_on_impassable`**：地图模式下能不能在未探索/不可通行区域使用

> **新手记忆**：**先掌握第一档 6 个**，后两档**写到再来查表**就行。Klei 自己的 180 多个动作里，**80% 也只是用了第一档的字段**。

#### 第四步：实例对比——三种典型动作的配置

**(a) 简单工具类动作 `MINE`**：

```lua
MINE = Action({ invalid_hold_action=true }),
```

只配了 `invalid_hold_action`——其他全用默认。`distance=nil`（默认 1.5）、`priority=0`、`fn` 来自后面 `ACTIONS.MINE.fn = ...` 的赋值。

**(b) 中复杂度的 `ATTACK`**：

```lua
ATTACK = Action({priority=2, canforce=true, mount_valid=true, invalid_hold_action=true }),
```

四个字段：高优先级 + 远程指定 + 骑战斗 + 不能按住。

**(c) 高复杂度的 `OCEAN_FISHING_CAST`**：

```lua
OCEAN_FISHING_CAST = Action({
    priority=3, rmb=true,
    customarrivecheck=CheckOceanFishingCastRange,
    is_relative_to_platform=true,
    disable_platform_hopping=true,
    invalid_hold_action=true
}),
```

七个字段——这是因为海钓涉及"在浮板上瞄准水面、抛竿后不能跳到别的浮板"等复杂逻辑，**字段越多说明这个动作越特殊**。

> **进阶提示**：**写自定义动作时，先看哪个原版动作和你的需求最像——直接把它的配置复制过来改几个字段**。比如想做"用斧头敲冰"——直接抄 `CHOP` 的配置，改个 `distance` 就好。

---

### 7.1.4 进阶：ACTIONS_BY_ACTION_CODE 与"动作编码"机制

#### 第一步：看一个奇怪的现象

在 actions.lua 第 710-719 行：

```710:719:scripts/actions.lua
ACTIONS_BY_ACTION_CODE = {}

ACTION_IDS = {}
for k, v in orderedPairs(ACTIONS) do
    v.str = STRINGS.ACTIONS[k] or "ACTION"
    v.id = k
    table.insert(ACTION_IDS, k)
    v.code = #ACTION_IDS
    ACTIONS_BY_ACTION_CODE[v.code] = v
end
```

这段代码做了什么？

1. 创建空表 `ACTIONS_BY_ACTION_CODE`
2. 创建空表 `ACTION_IDS`
3. **按字母顺序**遍历 `ACTIONS`（注意是 `orderedPairs` 而不是 `pairs`——保证每次启动顺序一致！）
4. 给每个动作设置：
   - `v.str` —— 本地化字符串（来自 STRINGS.ACTIONS）
   - `v.id` —— 动作 ID（如 `"CHOP"`）
   - `v.code` —— **递增的整数编号**（从 1 开始）
5. 把 `ACTIONS_BY_ACTION_CODE[code] = action`——**用整数索引这个动作**

**问题**：为什么需要 `code`？为什么需要"反查表" `ACTIONS_BY_ACTION_CODE`？

#### 第二步：答案藏在网络同步里

回顾第 6 章我们讲过的 net 变量原理——**网络包里的每个 byte 都是钱**。

想象一下：客户端点击"砍树"，需要把"我要执行 CHOP 动作"告诉服务端。**两种方案**：

**方案 A**：把字符串 `"CHOP"` 通过 RPC 发给服务端

- 4 个字符（CHOP）= 4 字节
- 加上字符串长度前缀 = 5 字节起步
- 复杂动作如 `OCEAN_FISHING_CAST` = 18 字节
- **每秒玩家点击多次**——浪费带宽

**方案 B**：把整数 `code` 发给服务端

- 一个 byte（最多 255 个动作）= 1 字节
- 即使用 short（2 字节）也能编码 65535 个动作
- **省 80% 以上带宽**

Klei 选了方案 B。

#### 第三步：看 RPC 调用里的 code 用法

在 `playercontroller.lua` 里有这样的代码（简化版）：

```lua
SendRPCToServer(RPC.LeftClick, ACTIONS.CHOP.code, target.GUID)
```

——发出去的不是 `"CHOP"`，是一个数字（比如 `15`）。服务端收到后通过 `ACTIONS_BY_ACTION_CODE[15]` 反查回 `ACTIONS.CHOP` 这个对象，然后调用它的 `fn`。

**这就是 `ACTIONS_BY_ACTION_CODE` 存在的核心理由——网络同步时的"动作 ID 压缩"**。

#### 第四步：`orderedPairs` 的隐藏意义

```lua
for k, v in orderedPairs(ACTIONS) do
```

为什么不能用普通 `pairs`？

**因为 Lua 表的 `pairs` 遍历顺序是不确定的**——"A 和 B 谁先遍历到"由 hash 实现细节决定，**不同 Lua 版本/不同硬件可能不同**。

**如果用 `pairs`**：

- 服务端启动：`CHOP.code = 15`（CHOP 是第 15 个被遍历到的）
- 客户端启动：`CHOP.code = 23`（在客户端的 hash 里 CHOP 排第 23 个）
- 客户端发 RPC `code=23`，服务端用 `ACTIONS_BY_ACTION_CODE[23]` 取出来发现是 `EAT`！
- **大爆炸**：你点树却开始吃树

`orderedPairs` 是一个工具函数，定义在 `util.lua`——**按 key 的字典序遍历**。这样**所有运行该代码的进程对 ACTIONS 表的遍历顺序完全一致**，code 编号在所有 client/server 上完全一致——网络同步永远正确。

> **开发逻辑**：**这种"看似多余的细节"是分布式系统的灵魂**——所有可能产生顺序不一致的地方都要"显式排序"。客户端服务端各自启动时，如果有任何一处用 `pairs` 而不是 `orderedPairs`，**多人游戏就有几率随机崩溃**。Klei 在很多关键位置都用 `orderedPairs`——值得你专门用 `rg "orderedPairs"` 看一遍学习。

#### 第五步：用控制台验证 code 的值

进游戏，控制台：

```lua
print(ACTIONS.CHOP.code)
print(ACTIONS.EAT.code)
print(ACTIONS.PICK.code)
print(ACTIONS_BY_ACTION_CODE[ACTIONS.CHOP.code] == ACTIONS.CHOP)  -- 应该是 true
```

每次启动游戏的 code 值**应该是一样的**（因为 `orderedPairs` + 原版 ACTIONS 表内容固定）。**但加了不同 Mod 后，code 值可能会变**——这是 7.1.5 节要讲的。

---

### 7.1.5 进阶：原版 Action 与 Mod Action 的两套注册表

#### 第一步：观察一个隐含问题

设想：你写了一个 Mod 添加新动作 `PSIONIC_BURST`、另一个 Mod 添加 `PSIONIC_DRAIN`。两个 Mod 各自给自己的动作分配了 `code = 1`（在自己的命名空间里都是第 1 个）。

**冲突！** 客户端发 RPC `code=1`，服务端不知道该去哪个 Mod 的命名空间里查。

Klei 的解决方案是 **`MOD_ACTIONS_BY_ACTION_CODE`** 这个 **二维表**。

#### 第二步：看 Mod Action 注册表的定义

`actions.lua` 第 721-723 行：

```721:723:scripts/actions.lua
MOD_ACTIONS_BY_ACTION_CODE = {}

ACTION_MOD_IDS = {} --This will be filled in when mods add actions via AddAction in modutil.lua
```

**两张表**：

- **`ACTION_MOD_IDS[mod_name]`** = mod 添加的所有动作 ID 数组
- **`MOD_ACTIONS_BY_ACTION_CODE[mod_name][code]`** = mod 内部的 code → action 反查表

**结构是 `mod_name → code → action`** 三层嵌套。

#### 第三步：看 `AddAction` 怎么填这两张表

`modutil.lua` 第 442-476 行：

```442:476:scripts/modutil.lua
	env.AddAction = function( id, str, fn )
		local action
        if type(id) == "table" and id.is_a and id:is_a(Action) then
			--backwards compatibility with old AddAction
            action = id
        else
			assert( str ~= nil and type(str) == "string", "Must specify a string for your custom action! Example: \"Perform My Action\"")
			assert( fn ~= nil and type(fn) == "function", "Must specify a fn for your custom action! Example: \"function(act) --[[your action code]] end\"")
			action = Action()
			action.id = id
			action.str = str
			action.fn = fn
		end
		action.mod_name = env.modname

		assert( action.id ~= nil and type(action.id) == "string", "Must specify an ID for your custom action! Example: \"MYACTION\"")

		initprint("AddAction", action.id)
		ACTIONS[action.id] = action

		--put it's mapping into a different IDS table, one for each mod
		if ACTION_MOD_IDS[action.mod_name] == nil then
			ACTION_MOD_IDS[action.mod_name] = {}
		end
		table.insert(ACTION_MOD_IDS[action.mod_name], action.id)
		action.code = #ACTION_MOD_IDS[action.mod_name]
		if MOD_ACTIONS_BY_ACTION_CODE[action.mod_name] == nil then
			MOD_ACTIONS_BY_ACTION_CODE[action.mod_name] = {}
		end
		MOD_ACTIONS_BY_ACTION_CODE[action.mod_name][action.code] = action

		STRINGS.ACTIONS[action.id] = action.str

		return ACTIONS[action.id]
	end
```

**关键的两段**：

```lua
table.insert(ACTION_MOD_IDS[action.mod_name], action.id)
action.code = #ACTION_MOD_IDS[action.mod_name]
```

**新动作的 code 是在它所属 Mod 的命名空间里递增**——而不是在全局命名空间里。这样：

- Mod A 的第一个动作 code = 1
- Mod B 的第一个动作 code = 1（不同 Mod 命名空间，互不干扰）
- 网络同步时 RPC 发 `code + mod_name`，服务端查 `MOD_ACTIONS_BY_ACTION_CODE[mod_name][code]` —— 不冲突

#### 第四步：网络同步时的 mod_name 处理

回看 actions.lua 第 262-268 行的 `SetClientRequestedAction`：

```262:268:scripts/actions.lua
function SetClientRequestedAction(actioncode, mod_name)
    if mod_name then
        CLIENT_REQUESTED_ACTION = MOD_ACTIONS_BY_ACTION_CODE[mod_name] and MOD_ACTIONS_BY_ACTION_CODE[mod_name][actioncode] or nil
    else
        CLIENT_REQUESTED_ACTION = ACTIONS_BY_ACTION_CODE[actioncode]
    end
end
```

**如果 RPC 包带了 `mod_name`** —— 走 Mod 的二维表
**如果没带 `mod_name`** —— 走原版的一维表

这就是网络同步时**区分原版动作和 Mod 动作**的逻辑。RPC 包多带 4-20 个字节的 mod_name 字符串——但只在 Mod 动作时才需要带，原版动作不带——总体的网络成本依然小。

> **开发逻辑**：这种**双注册表 + 命名空间隔离**的设计，让饥荒成为了**一个非常 Mod 友好的引擎**——你可以在你的 Mod 里随便加 50 个动作，不会和原版冲突，也不会和别的 Mod 冲突。**这是好的引擎设计的典范**。

#### 第五步：实例验证

进游戏后，控制台：

```lua
print(ACTIONS.CHOP.mod_name)        -- nil（原版动作没有 mod_name）
print(ACTIONS.CHOP.code)            -- 一个数字，比如 15

-- 假设你装了一个加 EATSPIDER 动作的 Mod
print(ACTIONS.EATSPIDER.mod_name)    -- "workshop-12345" 或 "spidermod"
print(ACTIONS.EATSPIDER.code)        -- 1（Mod 命名空间内的第 1 个）
```

**`mod_name == nil` 是判断"是不是原版动作"的标志**——这在很多代码里都会出现：

```lua
if action.mod_name then
    -- Mod action: 网络同步时要带 mod_name
else
    -- 原版 action: 不需要带 mod_name
end
```

---

### 7.1.6 进阶：fn / strfn / canforce / rangecheckfn / validfn 几类回调的分工

Action 类支持**5 种回调函数**——它们各自负责动作生命周期里的不同环节。我们一个个讲。

#### 第一步：`fn` —— 动作的"执行函数"（必填）

回顾 7.1.1 我们已经看过 `CHOP.fn`：

```lua
ACTIONS.CHOP.fn = function(act)
    -- ...
    return true / false, reason
end
```

**职责**：**真正"做事"的函数**。在这里你调用组件方法、修改世界状态、PushEvent 通知。

**调用时机**：玩家走到位置后、StateGraph 切到对应状态、动画里的"接触帧"触发。

**返回约定**：

- `true` —— 成功
- `false, "REASON_KEY"` —— 失败，让角色说一句相关的对白
- `nil` —— 既不算成功也不算失败（少见）

**`act` 参数包含**：

```lua
act.doer        -- 执行者（一般是玩家）
act.target      -- 目标实体
act.invobject   -- 手里的物品
act.pos         -- 位置（用于定位类动作）
act.recipe      -- 配方（用于 BUILD）
act.action      -- 这个 BufferedAction 对应的 Action 对象（self 引用）
```

#### 第二步：`strfn` —— 动态字符串函数（可选）

看 `actions.lua` 第 748-758 行的 `EAT.strfn`：

```748:758:scripts/actions.lua
ACTIONS.EAT.strfn = function(act)
    if act.invobject ~= nil then
        return (act.doer ~= nil and
                (act.doer:HasTag("spoiledprocessor") and act.invobject:HasTag("spoiledfood"))
                or (act.doer:HasTag("allspoiledprocessor") and act.invobject:HasTag("spoiled"))) and "PROCESS"
            or act.invobject:HasTag("fooddrink") and "DRINK"
            or nil
    end

    return nil
end
```

**职责**：**动态决定动作显示的字符串**——返回一个字符串后缀，引擎会去 `STRINGS.ACTIONS.EAT[returned_string]` 找对应翻译。

**实例**：

- 默认 EAT 显示"吃"
- 如果手里是饮料 → 显示"喝"（对应 `STRINGS.ACTIONS.EAT.DRINK`）
- 如果是 WX-78 + 腐烂食物 → 显示"处理"（对应 `STRINGS.ACTIONS.EAT.PROCESS`）

**调用时机**：UI 显示动作菜单时调用——**不会影响动作执行**。

> **进阶提示**：`strfn` 让一个 Action **能在不同上下文显示不同动词**——避免"塞 30 个 EAT_*  常量"。但写复杂的 `strfn` 容易踩坑：UI 渲染对性能敏感，**`strfn` 必须很轻量**——不要在里面做 IO、遍历大表。

#### 第三步：`validfn` —— 验证函数（可选）

看 `actions.lua` 第 1705-1707 行的 `CHOP.validfn`：

```1705:1707:scripts/actions.lua
ACTIONS.CHOP.validfn = function(act)
    return ValidToolWork(act, ACTIONS.CHOP)
end
```

**职责**：**在动作真正执行前再确认一次"这个动作是否有效"**——返回 `true/false`。

**调用时机**：BufferedAction 即将提交执行前。如果 validfn 返回 false，整个动作流程会**优雅地取消**——不会抛错、不会触发 fn。

**实例（PICK.validfn）**：

```lua
ACTIONS.PICK.validfn = function(act)
    return act.target
        and (
            (act.target.components.pickable and act.target.components.pickable:CanBePicked())
            or (act.target.components.searchable and act.target.components.searchable.canbesearched)
        )
end
```

**作用**：在玩家"决定要摘"和"真的摘到"之间，可能花了 1-2 秒走过去——这段时间花可能被别人摘了、变成不可摘的状态。`validfn` 在执行前**重新验证一次**，避免触发"摘到 nil 目标"的 bug。

> **设计哲学**：**fn 和 validfn 是分工的**——
>
> - `fn` 是"**做**"——做完返回 true / false
> - `validfn` 是"**该做吗**"——返回能否做
>
> 不是所有动作都有 `validfn`，只在"动作请求时和执行时之间状态可能变化"的动作里出现：摘花、砍树、对话、攻击都有；EQUIP/UNEQUIP 这种瞬时动作就不需要。

#### 第四步：`canforce` 和 `rangecheckfn`（远程指定）

**`canforce`**：**bool 字段**——这个动作能不能远程指定（按住 Shift + 点击）

**`rangecheckfn`**：**函数字段**——只在 `canforce=true` 时有效，定义"远程指定的有效范围"

例如 `ACTIONS.PICK`（actions.lua 第 345 行）：

```lua
PICK = Action({ canforce=true, rangecheckfn=PickRangeCheck, extra_arrive_dist=ExtraPickRange, mount_valid = true }),
```

`rangecheckfn` 是这样定义的（actions.lua 第 18-33 行）：

```18:33:scripts/actions.lua
local function PickRangeCheck(doer, target)
    if target == nil then
        return
    end
    local extrarange = 0
    if doer.replica.combat then
        if target:HasAnyTag("jostlepick", "jostlerummage", "jostlesearch") then
            extrarange = doer.replica.combat:GetWeaponAttackRange()
        end
    end
    local target_x, target_y, target_z = target.Transform:GetWorldPosition()
    local doer_x, doer_y, doer_z = doer.Transform:GetWorldPosition()
    local target_r = target:GetPhysicsRadius(0) + 4 + extrarange
    local dst = distsq(target_x, target_z, doer_x, doer_z)
    return dst <= target_r * target_r
end
```

**返回 true 表示"可以远程指定"**。这里加了一个特殊规则——如果目标是"挤撞采摘"类（`jostlepick`），可以**用武器攻击范围**作为"远程指定"的最大距离。

**为什么要分 canforce + rangecheckfn 两个字段？** —— `canforce` 是"开关"，`rangecheckfn` 是"规则"。**不是所有 canforce 动作都要自定义规则**——比如 `ATTACK.canforce=true` 但**没有 rangecheckfn**（attack 自己有距离判断）。

#### 第五步：`pre_action_cb` —— 前置回调

```lua
PICK = Action({ ..., pre_action_cb = function(act) ... end }),
```

**职责**：**动作正式开始前**调用——可以做一些"准备工作"或"取消逻辑"。**注意它在 client + server 都跑**——和 `fn` 不同，`fn` 一般只在服务端跑。

**实例**：某些动作的 `pre_action_cb` 会**预设一个客户端的视觉效果**（如准备砍树时屏幕轻微震动），让玩家感觉响应更快。

#### 第六步：把 5 种回调串起来看

整个动作的"回调时序"：

```
[玩家点击]
    ↓
[1] strfn（决定显示什么字符串）
    ↓
[2] 玩家选择执行 → 创建 BufferedAction
    ↓
[3] pre_action_cb（client + server 同步执行预处理）
    ↓
[4] 玩家走到位置（如果不是 instant）
    ↓
[5] validfn（执行前再次验证）
    ↓
[6] StateGraph 触发动画
    ↓
[7] 动画里的"接触帧"触发 fn（业务执行）
    ↓
[8] fn 返回 true → 动作成功
    fn 返回 false → 动作失败
```

> **新手记忆**：**90% 的自定义动作只需要写 `fn`**。`strfn` 在你想"一个动作显示多种动词"时才用、`validfn` 在你担心"目标可能变化"时才用、`canforce/rangecheckfn` 在你想"远程指定"时才用、`pre_action_cb` 几乎从来不用。

---

### 7.1.7 老手进阶：`AddAction` 内部完整流程拆解

`AddAction` 是 Mod 添加自定义动作的**唯一接口**。我们逐行拆解它做了什么。

#### 第一步：完整源码（再贴一次）

```442:476:scripts/modutil.lua
	env.AddAction = function( id, str, fn )
		local action
        if type(id) == "table" and id.is_a and id:is_a(Action) then
			--backwards compatibility with old AddAction
            action = id
        else
			assert( str ~= nil and type(str) == "string", "Must specify a string for your custom action! Example: \"Perform My Action\"")
			assert( fn ~= nil and type(fn) == "function", "Must specify a fn for your custom action! Example: \"function(act) --[[your action code]] end\"")
			action = Action()
			action.id = id
			action.str = str
			action.fn = fn
		end
		action.mod_name = env.modname

		assert( action.id ~= nil and type(action.id) == "string", "Must specify an ID for your custom action! Example: \"MYACTION\"")

		initprint("AddAction", action.id)
		ACTIONS[action.id] = action

		--put it's mapping into a different IDS table, one for each mod
		if ACTION_MOD_IDS[action.mod_name] == nil then
			ACTION_MOD_IDS[action.mod_name] = {}
		end
		table.insert(ACTION_MOD_IDS[action.mod_name], action.id)
		action.code = #ACTION_MOD_IDS[action.mod_name]
		if MOD_ACTIONS_BY_ACTION_CODE[action.mod_name] == nil then
			MOD_ACTIONS_BY_ACTION_CODE[action.mod_name] = {}
		end
		MOD_ACTIONS_BY_ACTION_CODE[action.mod_name][action.code] = action

		STRINGS.ACTIONS[action.id] = action.str

		return ACTIONS[action.id]
	end
```

#### 第二步：拆成 8 个步骤

**步骤 1：参数模式判断**

```lua
if type(id) == "table" and id.is_a and id:is_a(Action) then
    action = id
else
    -- 三参数模式...
end
```

`AddAction` **支持两种调用方式**：

**(A) 三参数简洁模式**：

```lua
AddAction("PSIONIC_BURST", "释放灵能", function(act) ... end)
```

`AddAction` 内部 `Action()` 一个空配置，再把 id/str/fn 塞进去。

**(B) 单参数高级模式**（向后兼容老 Mod）：

```lua
local act = Action({ priority=2, distance=1.5, rmb=true })
act.id = "PSIONIC_BURST"
act.str = "释放灵能"
act.fn = function(act) ... end
AddAction(act)
```

**为什么要支持两种**？因为最初的 `AddAction` 只支持简洁模式，**后来发现"想配优先级、距离、rmb 等参数"无法表达**——于是增加了模式 B。

> **进阶提示**：**简洁模式 (A) 只能配置 id/str/fn 三个字段，所有其他参数都用默认值**。如果你想自定义 priority、distance、rmb 等——**必须用模式 B**！这是新手最容易困惑的地方："我的自定义动作为什么 distance 不生效？" —— 因为你用了简洁模式，没法传 distance。

**步骤 2：必填断言**

```lua
assert( str ~= nil and type(str) == "string", "Must specify a string..." )
assert( fn ~= nil and type(fn) == "function", "Must specify a fn..." )
```

**没传 str 或 fn 直接崩溃**。这是"early failure"哲学——**让 Mod 作者第一次启动游戏就发现错误**，而不是埋到运行时。

**步骤 3：标记 mod_name**

```lua
action.mod_name = env.modname
```

`env.modname` 是 Mod 加载器在加载这个 Mod 时设置的全局变量。这一步给 action 打上"我是哪个 Mod 的"标签——是 7.1.5 节命名空间隔离的基础。

**步骤 4：注册到全局 ACTIONS 表**

```lua
ACTIONS[action.id] = action
```

**这一行让 `ACTIONS.PSIONIC_BURST` 在全局可访问**——所有代码都能 `if ACTIONS.PSIONIC_BURST then ... end` 检查它存在。

**步骤 5：注册到 ACTION_MOD_IDS**

```lua
if ACTION_MOD_IDS[action.mod_name] == nil then
    ACTION_MOD_IDS[action.mod_name] = {}
end
table.insert(ACTION_MOD_IDS[action.mod_name], action.id)
```

**第一次加 Mod 动作时创建一个空表**，把 id 插到末尾。多个动作就形成一个**有序数组**。

**步骤 6：分配 code**

```lua
action.code = #ACTION_MOD_IDS[action.mod_name]
```

**code = 当前数组长度**——即"我是这个 Mod 的第 N 个动作"。第一个 = 1，第二个 = 2，等等。

**步骤 7：登记到 MOD_ACTIONS_BY_ACTION_CODE 反查表**

```lua
if MOD_ACTIONS_BY_ACTION_CODE[action.mod_name] == nil then
    MOD_ACTIONS_BY_ACTION_CODE[action.mod_name] = {}
end
MOD_ACTIONS_BY_ACTION_CODE[action.mod_name][action.code] = action
```

**网络同步时，服务端通过 `MOD_ACTIONS_BY_ACTION_CODE[mod_name][code]` 反查到 action**。

**步骤 8：注册本地化字符串**

```lua
STRINGS.ACTIONS[action.id] = action.str
```

`STRINGS.ACTIONS` 是全局字符串表（5.7.5 节我们讲过 STRINGS）。这一行让动作的"显示文本"出现在右键菜单里。

**最后**：`return ACTIONS[action.id]`——返回新建的 action，让调用方可以**继续配置**：

```lua
local act = AddAction("PSIONIC_BURST", "释放灵能", function(act) ... end)
act.distance = 2     -- 后续配置
act.priority = 5
```

#### 第三步：完整调用范例

写在 modmain.lua 顶层：

```lua
-- 模式 A 简洁版
AddAction("PSIONIC_BURST", "释放灵能", function(act)
    if act.target ~= nil and act.target.components.psionic ~= nil then
        return act.target.components.psionic:Burst()
    end
    return false
end)

-- 模式 B 高级版（如果想自定义 priority/distance）
local burstAction = Action({
    priority = 2,
    distance = 1.5,
    rmb = true,           -- 右键触发
    invalid_hold_action = true,
})
burstAction.id = "PSIONIC_BURST"
burstAction.str = "释放灵能"
burstAction.fn = function(act)
    if act.target ~= nil and act.target.components.psionic ~= nil then
        return act.target.components.psionic:Burst()
    end
    return false
end
AddAction(burstAction)
```

> **新手提示**：**先用简洁模式（A）跑通**，等需要配置 priority/distance/rmb 时再升级到模式 B。简洁模式覆盖了 80% 的 Mod 使用场景。

#### 第四步：注册成功后还要做的两件事

`AddAction` 只完成了"动作的注册"——但要让玩家**真正能在游戏里触发它**，还需要：

1. **`AddComponentAction(actiontype, component, fn)`** —— 告诉引擎"哪些组件能产生这个动作"（7.2 节会讲）
2. **`AddStategraphActionHandler(stategraph_name, ActionHandler(...))`** —— 告诉 StateGraph "玩家想做这个动作时切到哪个状态"（7.3 节会讲）

**只 AddAction 不接 ComponentAction**——动作虽然存在但没人会触发它。这是新手最常见的"我的自定义动作不生效"原因。

> **老手记忆**：**`AddAction` 是"声明动词存在"，`AddComponentAction` 是"声明动词如何被触发"，`AddStategraphActionHandler` 是"声明动词的执行动画"** —— 三步缺一不可。

---

### 7.1.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：动作 ID 用小写或包含空格

```lua
-- ❌ 错：小写 id
AddAction("psionic_burst", "释放灵能", function(act) ... end)

-- ❌ 错：含空格
AddAction("PSIONIC BURST", "释放灵能", function(act) ... end)

-- ✅ 对：全大写 + 下划线
AddAction("PSIONIC_BURST", "释放灵能", function(act) ... end)
```

**原因**：饥荒的所有 ACTIONS 表 key 都是**全大写 + 下划线分隔**——这是 Klei 的命名约定。`STRINGS.ACTIONS[id]` 也按这个约定查找。**用小写或空格虽然不会崩，但会破坏一致性、影响可读性、可能在某些代码路径里出 bug**。

#### 陷阱 2：忘记简洁模式不能配 distance/priority

```lua
-- ❌ 错：以为这样能配 distance
local act = AddAction("PSIONIC_BURST", "释放灵能", function(act) ... end)
act.distance = 1.5  -- 没问题，能改
act.priority = 5    -- 也能改

-- 但是 Klei 推荐的是模式 B：
-- ✅ 对：用 Action() 一次配齐
local burstAction = Action({distance = 1.5, priority = 5, rmb = true})
burstAction.id = "PSIONIC_BURST"
burstAction.str = "释放灵能"
burstAction.fn = function(act) ... end
AddAction(burstAction)
```

**为什么模式 B 更好**：
- **集中可读** —— 所有配置在一个 table 里
- **不容易漏** —— 模式 A 后续 `act.xxx = yyy` 容易写在 modmain 末尾，让别人难以找到完整配置
- **某些字段必须在构造时传** —— 比如 `data.canforce` 会触发 `rangecheckfn` 的解析逻辑：`self.rangecheckfn = self.canforce ~= nil and data.rangecheckfn or nil`。**如果你后来 `act.canforce = true` 再 `act.rangecheckfn = fn`，这个解析逻辑没机会跑**——某些情况下可能行为不一致。

#### 陷阱 3：fn 里访问 nil 目标

```lua
-- ❌ 错：直接访问 act.target 不判 nil
ACTIONS.PSIONIC_BURST.fn = function(act)
    return act.target.components.psionic:Burst()  -- target 可能 nil！
end

-- ✅ 对：先检查
ACTIONS.PSIONIC_BURST.fn = function(act)
    if act.target ~= nil and act.target.components.psionic ~= nil then
        return act.target.components.psionic:Burst()
    end
    return false, "NOTARGET"
end
```

**原因**：`fn` 被调用时**目标可能已经被销毁**——典型场景：玩家走过去 1 秒，期间另一个玩家把目标摘了 / 怪兽吃了 / 时间到了腐烂消失了。**fn 永远要防御式编程**——nil 检查、IsValid 检查、组件存在检查全部上。

看 `actions.lua` 第 1690-1703 行的 `CHOP.fn`——里面有 `act.target:IsValid()` 等多重检查。

#### 陷阱 4：返回值不规范

```lua
-- ❌ 错：什么都不返回
ACTIONS.MYACTION.fn = function(act)
    -- 做事...
    -- 没 return → Lua 默认返回 nil
end

-- ❌ 错：返回非 bool
ACTIONS.MYACTION.fn = function(act)
    return "ok"  -- 字符串被当成 truthy，但容易 confusing
end

-- ✅ 对：明确返回 true / false
ACTIONS.MYACTION.fn = function(act)
    if 成功条件 then
        return true
    else
        return false, "REASON_KEY"
    end
end
```

**原因**：饥荒引擎对 `fn` 的返回值有约定——`true` 触发"成功反馈"（角色挥手、UI 提示等），`false, reason` 触发"失败反馈"（角色说话）。**返回 nil 或非 bool 会让引擎走入"中性分支"——既没成功反馈也没失败反馈，玩家会觉得"动作执行了但没反应"**。

#### 陷阱 5：忘记给 STRINGS.ACTIONS 加翻译

```lua
-- ❌ 错：动作名是中文，但没给 STRINGS.ACTIONS 加 entry
AddAction("PSIONIC_BURST", "释放灵能", function(act) ... end)
-- 玩家右键菜单显示："释放灵能"  (从 AddAction 的 str 参数)
-- 但如果有别的地方查 STRINGS.ACTIONS.PSIONIC_BURST_CHARGED 这种带后缀的...
-- 就要手动补：
-- ✅ 对：加一行
STRINGS.ACTIONS.PSIONIC_BURST = "释放灵能"
STRINGS.ACTIONS.PSIONIC_BURST_CHARGED = "释放（已充能）"
```

**原因**：`AddAction` 的第二个参数 `str` 会自动写入 `STRINGS.ACTIONS[id]`——**但只是默认 entry**。如果你的 `strfn` 返回了"CHARGED"这种后缀，引擎会查 `STRINGS.ACTIONS.PSIONIC_BURST.CHARGED`（如果是 table）或 `STRINGS.ACTIONS.PSIONIC_BURST_CHARGED`（如果是 string）——这些**默认是不存在的**。

**`strfn` 用法的标准搭配**：

```lua
-- 让 STRINGS.ACTIONS.PSIONIC_BURST 是一个 table 而不是字符串
STRINGS.ACTIONS.PSIONIC_BURST = {
    GENERIC = "释放灵能",
    CHARGED = "释放（已充能）",
}

ACTIONS.PSIONIC_BURST.strfn = function(act)
    if act.target and act.target:HasTag("psionic_full") then
        return "CHARGED"
    end
    -- 返回 nil 走 GENERIC
end
```

#### 陷阱 6：忘记 `AddComponentAction` 联动

```lua
-- ❌ 错：只 AddAction，没 AddComponentAction
AddAction("PSIONIC_BURST", "释放灵能", function(act) ... end)
-- 进游戏：右键灵能晶簇没有"释放灵能"选项
-- 因为引擎不知道"哪些实体应该出现这个动作"

-- ✅ 对：两步走
AddAction("PSIONIC_BURST", "释放灵能", function(act) ... end)

AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_full") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
```

**这就是 7.2 节要详细讲的 `ComponentAction` 机制**——它告诉引擎"在什么场景下这个动作会出现"。**`AddAction` 只是"声明这个词存在"，没有 `AddComponentAction` 就没人会调用它**。

> **老手记忆**：**饥荒的 Action 系统是"双向注册"的**——`AddAction` 注册动作本身、`AddComponentAction` 注册"什么情况下动作可用"。**两个都是必须的，缺一不可**。

#### 设计经验三条

最后是写自定义动作时的几条思维原则：

**经验 1：`fn` 应该薄，业务在组件里**

```lua
-- ❌ 错：fn 里写一堆业务
ACTIONS.PSIONIC_BURST.fn = function(act)
    -- 50 行业务逻辑：扣灵能、找附近玩家、加理智、播粒子...
end

-- ✅ 对：fn 只做"调用组件方法"
ACTIONS.PSIONIC_BURST.fn = function(act)
    if act.target and act.target.components.psionic then
        return act.target.components.psionic:Burst()
    end
    return false
end
```

**业务永远写在组件里**——动作只是"触发器"。这样：
- **业务可以在多个动作里复用**（控制台命令、其他动作触发也能 Burst）
- **测试更容易**（直接调 `cmp:Burst()` 测，不需要走整个 Action 流程）
- **组件代码可以独立审查**（`Burst` 的逻辑读一遍就懂，不用跨文件看 Action）

**经验 2：动作的 id 要"动词 + 名词"**

```
✅ ATTACK / CHOP / EAT / PICK    （单一动词，目标隐含）
✅ FEEDPLAYER / GIVETOPLAYER     （动词 + 目标限定）
✅ OCEAN_FISHING_CAST            （领域 + 动词）
✅ PSIONIC_BURST                 （领域 + 动词）

❌ DOIT                          （太模糊）
❌ MyAction1                     （命名不规范）
❌ DO_THE_THING                  （没语义）
```

**好的命名让代码自解释**——半年后回头看代码，从 id 就能立刻明白这个动作做什么。

**经验 3：自定义动作要为多人游戏考虑**

每次设计自定义动作问自己 5 个问题：

1. **客户端能预测这个动作吗？** —— 客户端是否需要在按下按键时立刻播一段动画（不等服务端响应）？如果是，需要 PlayerActionPicker / ActionHandler 双端处理。
2. **`fn` 在哪一端跑？** —— 默认服务端。如果你想在客户端跑（如修改 UI），需要专门设计。
3. **动作执行期间，其他玩家能看到吗？** —— 走 NetSync 的 StateGraph 状态、还是仅本地视觉？
4. **动作冲突怎么处理？** —— 两个玩家同时对一棵树 CHOP，谁胜出？
5. **重连后状态怎么恢复？** —— 玩家断线时正在执行某个动作，重连后应该取消还是继续？

**这些问题不是 7.1 节能讲完的**——但**写自定义动作时要带着这些问题去设计**，避免单机思维写出多人灾难。

---

### 7.1.9 小结

- **Action 是饥荒的"动词词典"**——所有玩家能做的事被收纳在全局 `ACTIONS` 表里。**整个游戏 180+ 个动作**，常用的 30-40 个。
- **Action 类**是一个**配置容器**——30 多个字段描述"动作如何工作"。**核心 6 个字段**：`priority` / `fn` / `distance` / `rmb` / `instant` / `canforce`。
- **`fn / strfn / validfn / canforce / rangecheckfn / pre_action_cb` 是 6 类回调**——分别在动作的不同阶段被调用。**90% 自定义动作只需要写 `fn`**。
- **`ACTIONS_BY_ACTION_CODE`** 用整数 code 而不是字符串做网络同步——节省 80% 以上带宽。**`orderedPairs` 保证 client/server code 一致**——这是 Klei 在引擎设计上的精细手笔。
- **`MOD_ACTIONS_BY_ACTION_CODE` + `ACTION_MOD_IDS`** 是 Mod 命名空间隔离的双注册表——让多个 Mod 的自定义动作互不冲突。
- **`AddAction` 是 Mod 添加动作的唯一接口**——8 步流程（参数判断 / 断言 / 标记 mod_name / 注册到 ACTIONS / 加入 ACTION_MOD_IDS / 分配 code / 写入反查表 / 注册 STRINGS）。**支持简洁模式 (id, str, fn) 和高级模式 (Action({...}))**。
- **6 个常见陷阱**：① 命名不规范 ② 简洁模式无法配高级参数 ③ 不防御 nil 目标 ④ 返回值不规范 ⑤ 忘记 STRINGS 翻译 ⑥ 忘记 AddComponentAction 联动。
- **`AddAction` 只是"动词存在"**——配合 7.2 的 `AddComponentAction`（"什么时候可用"）和 7.3 的 `AddStategraphActionHandler`（"如何执行"）才形成完整链路。

> **下一节预告**：7.2 节我们打开 `componentactions.lua` —— 这是回答"**为什么右键斧头会出现 CHOP 选项**"的源码。组件通过一个**注册表**告诉引擎"我能产生哪些动作"——这个注册表就是 `COMPONENT_ACTIONS`。我们会看到三种主要的 ActionType（INVENTORY/USEITEM/SCENE），理解 Klei 为什么这样分类，以及如何用 `AddComponentAction` 给我们的 `psionic` 组件挂上 `PSIONIC_BURST` 动作。

---
## 7.2 ComponentAction——组件如何声明可执行的动作（componentactions.lua）

### 本节导读

7.1 节的末尾我们留下了一个悬念——**`AddAction` 只是"声明这个词存在"，没有人会自动触发它**。换句话说：

> 你可以写 `AddAction("PSIONIC_BURST", "释放灵能", fn)` —— `ACTIONS.PSIONIC_BURST` 现在存在了。但是玩家右键你的灵能晶簇——**右键菜单里不会出现"释放灵能"选项**。为什么？因为引擎不知道"在什么场景下、对什么实体、什么组件存在时，这个动作应该出现"。

回答这个问题的，就是 **`scripts/componentactions.lua`** —— 一个 3000 多行的"**动作触发条件总表**"。它解决的核心问题是：

> 给定 **(场景, 玩家, 目标实体, 是否右键)**，引擎要列出"可以做哪些动作"——这张列表怎么来？

答案：**遍历目标实体身上的组件，每个组件查 `COMPONENT_ACTIONS` 表，看它能产生哪些 Action**。

这一节我们打开这个文件，把它的**整体结构**、**6 种 ActionType**、**collector 函数签名**、**网络同步机制**、**`AddComponentAction` 接口**全部讲清楚：

> **新手**从 7.2.1-7.2.3 起步——理解一次"右键菜单弹出"在代码里走过哪些路径、`COMPONENT_ACTIONS` 是个什么样的二维表、6 种 ActionType（SCENE / USEITEM / POINT / EQUIPPED / INVENTORY / ISVALID）各自负责什么；**进阶读者**继续看 7.2.4-7.2.6，理解 collector 函数 5 种不同签名的参数含义、`actioncomponents` 字节数组怎么同步给客户端、`ISVALID` 这个特殊 actiontype 为什么存在；**老手**跳到 7.2.7-7.2.8，看 `AddComponentAction` 内部完整流程、Mod 命名空间隔离、最容易踩的六大坑——尤其是"客户端服务端 AddComponentAction 不一致就崩溃"这种最难调的 bug。

---

### 7.2.1 快速入门：从右键斧头出现 CHOP 选项 谈起

#### 第一步：在游戏里观察右键菜单的出现

打开饥荒，**手里拿着斧头**，**鼠标悬停在松树上**——你会看到屏幕中央显示：

```
砍 树
[左键执行]
```

**手里改成普通石头**——同一棵树，悬停时显示：

```
检查 树
[左键执行]
```

——同一棵树，**因为你手里拿的物品不同，可执行的动作变了**。

**手里拿着斧头，鼠标悬停在岩石上**——

```
（什么都没有，最多显示"行走"）
```

——同一把斧头，**因为目标的"组件构成"不同，能做的动作也不同**。

**这就是 ComponentAction 系统在工作**：**根据 "(目标, 手里物品, 玩家)" 三方的组件组合，动态生成可用动作列表**。

#### 第二步：拆开"右键斧头出现 CHOP"的代码路径

简化版执行链：

```
[1] 玩家鼠标悬停在松树上
       ↓
[2] PlayerActionPicker 检测悬停目标 → 收集所有可用 actions
       ↓
[3] 调用 tree.components.workable 的 collector：
    "我（workable 组件）能产生 ACTIONS.CHOP"
    （只在斧头是 chopper 时）
       ↓
[4] 同时调用 tree.components.inspectable 的 collector：
    "我（inspectable 组件）能产生 ACTIONS.LOOKAT"
       ↓
[5] PlayerActionPicker 拿到 actions = {CHOP, LOOKAT}
       ↓
[6] 按 priority 排序：CHOP(0) > LOOKAT(-3) → CHOP 胜出
       ↓
[7] UI 显示"砍 树"
```

**整个流程的核心**就在 **第 [3] 步**——**组件如何告诉引擎"我能产生什么动作"**。这件事由 `componentactions.lua` 里的 `COMPONENT_ACTIONS` 表负责。

#### 第三步：看一眼 `workable` 组件是怎么"声明" CHOP 动作的

在 `componentactions.lua` 里搜 `workable`，会看到（简化版）：

```lua
SCENE = {
    workable = function(inst, doer, actions, right)
        if (right or action ~= ACTIONS.HAMMER) and
           inst:HasTag("CHOP_workable") and
           doer.replica.inventory:EquipHasTag("chopper")
        then
            table.insert(actions, ACTIONS.CHOP)
        end
        -- ... 类似的还有 MINE / DIG / HAMMER 的判断
    end,
}
```

**这就是"声明"——一个普通的 Lua 函数**：

- 接收**(inst, doer, actions, right)**——目标、执行者、动作列表（输出参数）、是否右键
- 内部**检查各种 tag 和组件状态**
- 满足条件就 `table.insert(actions, ACTIONS.CHOP)` —— 把动作 push 到列表里

**这个函数有一个标准名字**：**collector function（采集器函数）**——它的工作就是"从组件状态里收集出对外可见的动作"。

> **新手记忆**：**ComponentAction 的本质是一张"组件 → 动作"的查找表**。每当玩家鼠标悬停某个实体，引擎会**遍历这个实体的所有 actioncomponents**（注册过的组件），调用每个组件对应的 collector 函数，把它们 push 进去的动作汇总起来——这就是右键菜单的来源。

---

### 7.2.2 快速入门：`COMPONENT_ACTIONS` 表的整体结构

#### 第一步：定位源码

`COMPONENT_ACTIONS` 定义在 **`scripts/componentactions.lua`** 第 139 行——这是一个**巨大的二维 Lua 表**：

```139:140:scripts/componentactions.lua
local COMPONENT_ACTIONS =
{
```

整个表占了 **2900 多行**——从第 139 行一直到第 3032 行才闭合。

#### 第二步：表结构（俯视图）

```
COMPONENT_ACTIONS = {
    SCENE = {                                 -- ActionType 1
        activatable = function(...) end,      -- 组件 → collector 函数
        anchor      = function(...) end,
        attunable   = function(...) end,
        ...
    },

    USEITEM = {                               -- ActionType 2
        appraisable = function(...) end,
        bait        = function(...) end,
        ...
    },

    POINT = { ... },                          -- ActionType 3
    EQUIPPED = { ... },                       -- ActionType 4
    INVENTORY = { ... },                      -- ActionType 5
    ISVALID = { ... },                        -- ActionType 6（特殊）
}
```

**两层嵌套**：

- **第一层 key**：`actiontype`（动作场景类型）—— 6 种
- **第二层 key**：`组件名`（小写，如 `workable`、`edible`）
- **value**：collector 函数

**每种 actiontype 下大约有 50-100 个组件 collector**——加起来全表大约 **300-400 个 collector 函数**，对应 Klei 的全部组件 → 动作映射。

#### 第三步：从一个具体例子看表项

**SCENE 下的 `book` 组件**（第 229 行）：

```229:233:scripts/componentactions.lua
        book = function(inst, doer, actions)
            if doer:HasTag("reader") and not inst:HasTag("fire") and not inst:HasTag("smolder") then
                table.insert(actions, ACTIONS.READ)
            end
        end,
```

**意思**：**当一个带 `book` 组件的物品出现在场景里被玩家鼠标悬停时**，如果玩家有 `reader` tag 且物品没着火——push `ACTIONS.READ` 进 actions。

**SCENE 下的 `battery` 组件**（第 180 行）：

```180:184:scripts/componentactions.lua
        battery = function(inst, doer, actions)
            if inst:HasTag("battery") and doer:HasTag("batteryuser") then
                table.insert(actions, ACTIONS.CHARGE_FROM)
            end
        end,
```

**意思**：场景里的电池实体被玩家悬停时——如果玩家有 `batteryuser` tag——push `ACTIONS.CHARGE_FROM` 进 actions。

#### 第四步：观察规律——所有 collector 都做三件事

```lua
component = function(inst, doer, actions, [其他参数])
    if 各种条件判断 then
        table.insert(actions, ACTIONS.XXX)
        -- 可能还有别的 ACTION
    end
end
```

**三件事**：

1. **接收参数** —— 至少有 inst（自己）、doer（执行者）、actions（输出列表）
2. **判断条件** —— 各种 `HasTag` / 组件状态检查
3. **push 动作** —— 满足就 `table.insert`

**这是 ComponentAction 的统一模式**——所有 collector 都长这样。**没有奇技淫巧、没有反射魔法**——就是一张 if-then-table.insert 的查找表。

#### 第五步：表的"字典序"重要吗？

回顾 7.1.4 我们讲过 `orderedPairs` —— ACTIONS 表的遍历顺序对网络同步至关重要。**ComponentAction 表也一样**！

看 `componentactions.lua` 第 3037-3047 行的 `RemapComponentActions`：

```3037:3047:scripts/componentactions.lua
local function RemapComponentActions()
    for k, v in orderedPairs(COMPONENT_ACTIONS) do
        for cmp, fn in orderedPairs(v) do
            if ACTION_COMPONENT_IDS[cmp] == nil then
                table.insert(ACTION_COMPONENT_NAMES, cmp)
                ACTION_COMPONENT_IDS[cmp] = #ACTION_COMPONENT_NAMES
            end
        end
    end
end
RemapComponentActions()
assert(#ACTION_COMPONENT_NAMES <= 255, "Increase actioncomponents network data size.")
```

**两层 `orderedPairs`** —— 保证组件名的编号在 client/server 完全一致。最后那个 `assert` 也很有意思——**组件名的编号必须 ≤ 255**，因为它要塞进一个 byte 里同步。**这就是为什么 `ACTION_COMPONENT_NAMES` 数组是网络同步的关键**——后面 7.2.5 节会详讲。

> **新手记忆**：`COMPONENT_ACTIONS` 是一张**"6 种场景 × 200+ 组件 × 不定数量动作"**的查找表。**每个表项都是一个 collector 函数，做一件事：根据组件状态决定要 push 哪些动作进 actions 列表**。理解这一点，你就理解了 ComponentAction 的全部本质。

---

### 7.2.3 快速入门：6 种 ActionType 各自负责什么

回顾 7.2.2 我们看到了 6 种 ActionType——它们对应**6 种不同的"操作场景"**。每种场景的 collector 函数签名也不同，因为参数不一样。

#### 第一步：6 种 ActionType 一览

| ActionType | 触发场景 | 函数签名 | 典型应用 |
|-----------|---------|---------|---------|
| **SCENE** | 玩家鼠标悬停**世界里的某个实体** | `(inst, doer, actions, right)` | 砍树、检查、捡花、攻击 |
| **USEITEM** | 玩家**手里拿着 inst**，鼠标悬停 **target** | `(inst, doer, target, actions, right)` | 用斧头砍树、用胡萝卜喂玩家 |
| **POINT** | 玩家**手里拿着 inst**，鼠标悬停**地面位置 pos** | `(inst, doer, pos, actions, right, target)` | 用铁铲挖泥（点空地）、放种子 |
| **EQUIPPED** | 玩家**装备着 inst**，鼠标悬停 **target** | `(inst, doer, target, actions, right)` | 装备的锤子拆建筑、装备的网捕昆虫 |
| **INVENTORY** | 玩家**右键背包里的 inst 自身** | `(inst, doer, actions, right)` | 装备 / 卸下 / 翻箱子 / 食用药品 |
| **ISVALID** | 验证某个 action 在某 inst 上**是否仍然有效** | `(inst, action, right)` | 防御式验证（不直接 push action） |

#### 第二步：详细解释每种场景

##### **(1) SCENE —— 世界中的实体**

**玩家做了什么**：移动鼠标到世界里某个实体上（或站在它旁边等待 ActionPicker 自动选）。

**典型 collector**（第 229-233 行的 `book`）：

```lua
book = function(inst, doer, actions)
    if doer:HasTag("reader") and not inst:HasTag("fire") and not inst:HasTag("smolder") then
        table.insert(actions, ACTIONS.READ)
    end
end,
```

**注意**：SCENE 通常**不带 `right` 参数检查**——因为 SCENE 里的物品大多支持左右键都能触发"主要动作"。少数 SCENE 项目会用 `right`，比如 `attune` 组件：

```169:174:scripts/componentactions.lua
        attunable = function(inst, doer, actions)
            if doer.components.attuner ~= nil and --V2C: this is on clients too
                not doer.components.attuner:IsAttunedTo(inst) then
                table.insert(actions, ACTIONS.ATTUNE)
            end
        end,
```

##### **(2) USEITEM —— 手里物品 + 目标**

**玩家做了什么**：鼠标拖动一个**背包里的物品**到**世界里另一个实体**上。

**关键区别**：USEITEM 比 SCENE **多一个 `target` 参数**——`inst` 是手里的物品，`target` 是被指向的目标。

**典型 collector**（第 1037-1041 行的 `bait`）：

```1037:1041:scripts/componentactions.lua
        bait = function(inst, doer, target, actions)
            if target:HasTag("canbait") then
                table.insert(actions, ACTIONS.BAIT)
            end
        end,
```

**意思**：玩家手里拿着 `bait` 组件的物品（如蜂蜜），拖到带 `canbait` tag 的目标（如陷阱）上——push `BAIT` 动作。

**典型对比**：

| 场景 | inst | target | 动作 |
|------|------|--------|------|
| 把蜜浆糊放陷阱 | 蜜浆糊（bait 组件） | 陷阱 | BAIT |
| 用胡萝卜喂玩家 | 胡萝卜（edible 组件） | 另一个玩家 | FEEDPLAYER |
| 把树苗种在地上 | 树苗（deployable 组件） | 地面位置（这其实是 POINT） | DEPLOY |

##### **(3) POINT —— 手里物品 + 地面位置**

**玩家做了什么**：鼠标拖动一个**背包里的物品**到**地面**（不是某个实体）上。

**和 USEITEM 的区别**：POINT 的第三个参数是 **`pos`（位置坐标）** 而不是 `target`（实体）。但**第六个参数仍然是 `target`**——某些情况下既有位置又有实体，需要两个都传。

**典型 collector**（POINT 段是 `componentactions.lua` 第 1975 行起）：

```lua
deployable = function(inst, doer, pos, actions, right, target)
    -- 检查能不能在 pos 位置部署
    if can_deploy(pos) then
        table.insert(actions, ACTIONS.DEPLOY)
    end
end,
```

**实际场景**：把"种子"扔到泥地上 → 触发 DEPLOY 动作（种植）。

##### **(4) EQUIPPED —— 装备的物品 + 目标**

**玩家做了什么**：玩家**已经装备**了某个物品（武器、工具、护身符），然后鼠标悬停某个目标。

**和 USEITEM 的区别**：USEITEM 是"拖动"未装备的物品；EQUIPPED 是"已装备的物品对目标自动触发动作"。**最常见的就是攻击**——玩家装备着武器，悬停怪物→自动产生 ATTACK 动作。

**典型 collector**（第 2241-2244 行的 `cooker`）：

```2241:2244:scripts/componentactions.lua
        cooker = function(inst, doer, target, actions, right)
            if right and
                (not inst:HasTag("dangerouscooker") or doer:HasTag("expertchef")) and
                target:HasTag("cookable") and
```

**意思**：装备了带 `cooker` 组件的物品（如手持烹饪锅），右键悬停某个 `cookable` 目标——push `COOK` 动作。

> **重要区别**：**`USEITEM` 是"我手里有 X 想用在 Y 上"，`EQUIPPED` 是"我装备了 X，Y 能不能成为我的使用对象"**。
>
> - 用 USEITEM：玩家要**手动选择**这个物品并指向目标（拖拽操作）
> - 用 EQUIPPED：物品**自动响应**——玩家只要悬停目标就生效

##### **(5) INVENTORY —— 背包栏内右键**

**玩家做了什么**：右键点击**背包栏里的物品**（在物品栏 UI 内、不在世界里）。

**典型 collector**（第 2500-2503 行的 `book`）：

```2500:2503:scripts/componentactions.lua
        book = function(inst, doer, actions)
            if doer:HasTag("reader") then
                table.insert(actions, ACTIONS.READ)
            end
        end,
```

**和 SCENE 下的 `book` 几乎一样**——但少了 `not inst:HasTag("fire")` 的检查（背包里的书不会着火）。

**INVENTORY 下最经典的就是 `edible`**：

```2564:2570:scripts/componentactions.lua
        edible = function(inst, doer, actions, right)
			local rider = doer.replica.rider
			local mount = rider and rider:GetMount() or nil
			local isactiveitem = doer.replica.inventory:GetActiveItem() == inst
```

**右键背包里的食物 → 出现"吃"选项**。如果只在 SCENE 注册 edible，那玩家必须把食物拖到自己身上才能吃——非常笨拙。**INVENTORY 类型让"右键直接吃"成为可能**。

##### **(6) ISVALID —— 动作有效性二次校验**

**和上面 5 种完全不同**——ISVALID 不"产生"动作，而是"验证已知动作是否有效"。

**典型 collector**（第 3027-3030 行的 `workable`）：

```3027:3030:scripts/componentactions.lua
        workable = function(inst, action, right)
            return (right or action ~= ACTIONS.HAMMER) and
                inst:HasTag(action.id.."_workable")
        end,
```

**注意签名**：`(inst, action, right)` —— **没有 doer、没有 actions 参数**。**返回 true/false**——表示"这个 action 在这个 inst 上是否合法"。

**调用时机**：当 PlayerActionPicker 拿到一个候选 action 后，会调用 `inst:IsActionValid(action, right)` 复核——这个方法内部就会查 `ISVALID` 这张表（看 `componentactions.lua` 第 3167-3194 行的实现）。

**作用**：**避免"该动作存在于 actions 列表里但目标实体不再支持"** —— 比如树倒了的瞬间还残留 CHOP 动作，ISVALID 在执行前再确认一次。

> **进阶提示**：ISVALID 是 **"事后过滤"**机制。前 5 种 actiontype 是"产生动作"，ISVALID 是"删除无效动作"。**两者配合让动作列表既丰富又准确**。绝大多数自定义动作不需要 ISVALID——只在出现"动作存在但不应该再生效"的微妙时机问题时才用。

#### 第三步：用一张表速查

**给你的自定义组件挂哪个 ActionType？**

| 你的组件特点 | 用哪个 ActionType |
|------------|------------------|
| 这个组件让实体能在世界里被点击操作（不需要手里物品） | **SCENE** |
| 这个组件让物品**作为工具**操作场景中的目标 | **USEITEM** |
| 这个组件让物品**部署/投掷到地面位置** | **POINT** |
| 这个组件让物品**装备后**对目标产生效果 | **EQUIPPED** |
| 这个组件让物品**右键背包就能用**（不需要拖到目标上） | **INVENTORY** |
| 这个组件需要**事后否决某个动作**的有效性 | **ISVALID** |

> **新手记忆**：**最常用的是 SCENE 和 INVENTORY**——它们对应"场景里直接点击"和"背包里右键"两种最常见操作。**USEITEM/POINT/EQUIPPED 在写"工具型物品"时才用**。**ISVALID 用得很少**。

---

### 7.2.4 进阶：collector 函数的签名与返回约定

#### 第一步：5 种签名速查

把 6 种 ActionType 的 collector 函数签名再列一遍：

```lua
-- SCENE
collector = function(inst, doer, actions, right) ... end

-- USEITEM
collector = function(inst, doer, target, actions, right) ... end

-- POINT
collector = function(inst, doer, pos, actions, right, target) ... end

-- EQUIPPED
collector = function(inst, doer, target, actions, right) ... end

-- INVENTORY
collector = function(inst, doer, actions, right) ... end

-- ISVALID
collector = function(inst, action, right) ... end       -- 注意：返回 bool，不 push！
```

**5 种签名**（USEITEM 和 EQUIPPED 完全相同，但语义不同）。

#### 第二步：参数详解

**`inst`** —— **永远是"持有这个组件的实体"**。注意：

- 在 SCENE 里：`inst` 是**世界里被悬停的实体**
- 在 USEITEM 里：`inst` 是**玩家手里的物品**（不是 target！）
- 在 POINT 里：`inst` 是**玩家手里的物品**
- 在 EQUIPPED 里：`inst` 是**装备着的物品**
- 在 INVENTORY 里：`inst` 是**背包里被右键的物品**
- 在 ISVALID 里：`inst` 是**被验证的目标实体**

> **新手陷阱**：很多人以为 `inst` 总是"目标"——错了！**`inst` 是"声明这个 collector 的组件所在的实体"**。USEITEM 的 inst 是手里的物品，target 才是目标。

**`doer`** —— **执行者**，几乎总是玩家实体。但要小心**远程操作的情况**：在某些 mod 里 doer 可能是 NPC 或别的实体。

**`actions`** —— **输出参数**——一个空表（或已经有别人 push 过的动作的表）。collector 的工作就是 `table.insert(actions, ACTIONS.XXX)` 往里塞。**不要 return**——直接修改这个表。

**`right`** —— **bool 值**：**当前是右键还是左键触发**。`true` 表示右键、`false` 表示左键、有时也是 `nil`（一般等同于 false）。

**`target`** —— USEITEM/EQUIPPED/POINT 多出来的——**鼠标指向的世界里的目标**。

**`pos`** —— POINT 多出来的——**鼠标指向的位置坐标**（一个带 `:GetPoint()` 方法的对象，调用返回 `x, y, z`）。

**`action`** —— ISVALID 多出来的——**被验证的 Action 对象**（不是字符串名字，是 `ACTIONS.XXX` 这种 Action 实例）。

#### 第三步：返回约定

**SCENE/USEITEM/POINT/EQUIPPED/INVENTORY** 这 5 种：**不需要 return**——直接修改 `actions` 表。

```lua
-- ✅ 标准写法
component = function(inst, doer, actions)
    if 条件 then
        table.insert(actions, ACTIONS.XXX)
    end
end,

-- ❌ 错：return 是没用的
component = function(inst, doer, actions)
    if 条件 then
        return ACTIONS.XXX  -- 引擎不看返回值！
    end
end,
```

**ISVALID** 不同：**必须 return bool**。

```lua
-- ✅ ISVALID 标准写法
component = function(inst, action, right)
    return action == ACTIONS.MYACTION and inst:HasTag("xxx")
end,
```

#### 第四步：如何 push 多个动作？

一个 collector 可以 push **多个不同动作**：

```lua
SCENE = {
    boatcannon = function(inst, doer, actions, right)
        if not inst:HasTag("occupied") and not inst:HasTag("burnt") then
            if inst:HasTag("ammoloaded") then
                table.insert(actions, ACTIONS.BOAT_CANNON_START_AIMING)
            elseif right then
                table.insert(actions, ACTIONS.BOAT_CANNON_LOAD_AMMO)
            end
        end
    end,
}
```

**意思**：船炮在不同状态下 push 不同动作——已装弹时 push 瞄准、未装弹时 push 装弹。**用 if-elseif 区分状态、用 if 多条件就允许同时 push**。

> **进阶提示**：**一个 collector push 多个动作时**——动作之间会通过 `priority` 自动排序。比如同时 push 了 `ATTACK(priority=2)` 和 `WALKTO(priority=-4)`——ActionPicker 会选 ATTACK 显示。**写 collector 的时候不需要担心顺序问题**——只管 push 你认为合理的动作集合，让 priority 系统去筛。

#### 第五步：实战例子——`weapon` 的 EQUIPPED 看起来怎么样？

我们没在源码里直接搜到 `weapon`，但能猜到它的样子：

```lua
EQUIPPED = {
    -- 假想代码（实际 ATTACK 是从 combat 组件的逻辑产生的）
    weapon = function(inst, doer, target, actions, right)
        if not right and target:HasTag("hostile") and not target:HasTag("notarget") then
            table.insert(actions, ACTIONS.ATTACK)
        end
    end,
}
```

**这是 EQUIPPED 的典型应用**——装备武器，自动悬停目标产生 ATTACK 选项。

实际上 ATTACK 在饥荒里不是通过 ComponentAction 触发的——它走 `combat` 组件的特殊路径（更高优先级、专用 RPC）。但**自定义"近战武器型物品"**通常会写自己的 EQUIPPED collector 来产生攻击动作。

> **新手记忆**：**collector 的工作只有"判断条件 + push 动作"**。**不要在里面做副作用**（修改组件状态、PushEvent、改 tag）——它的语义是"声明能力"，不是"执行能力"。**所有"做事"的代码应该在 `ACTIONS.XXX.fn` 里**——回顾 7.1.6 节关于 fn 的讲解。

---

### 7.2.5 进阶：actioncomponents 数组与网络同步

#### 第一步：先看一个谜——`self.actioncomponents` 是什么？

回顾 7.2.2 的 `EntityScript:CollectActions` 实现：

```3138:3149:scripts/componentactions.lua
function EntityScript:CollectActions(actiontype, ...)
    local t = COMPONENT_ACTIONS[actiontype]
    if t == nil then
        print("Action type", actiontype, "doesn't exist in the table of component actions. Is your component name correct in AddComponentAction?")
        return
    end
    for i, v in ipairs(self.actioncomponents) do
        local collector = t[ACTION_COMPONENT_NAMES[v]]
        if collector ~= nil then
            collector(self, ...)
        end
    end
```

**注意 `for i, v in ipairs(self.actioncomponents) do`** —— 引擎遍历的不是组件表 `self.components`，而是一个叫 `self.actioncomponents` 的数组。

**问题**：`self.actioncomponents` 是什么？为什么不直接遍历 `self.components`？

#### 第二步：看 `RegisterComponentActions` 的实现

```3085:3107:scripts/componentactions.lua
function EntityScript:RegisterComponentActions(name)
    local id = ACTION_COMPONENT_IDS[name]
    if id ~= nil then
        table.insert(self.actioncomponents, id)
        if self.actionreplica ~= nil then
            self.actionreplica.actioncomponents:set(self.actioncomponents)
        end
    end
    for modname, idmap in pairs(MOD_ACTION_COMPONENT_IDS) do
        id = idmap[name]
        if id ~= nil then
            if self.modactioncomponents == nil then
                self.modactioncomponents = { [modname] = {} }
            elseif self.modactioncomponents[modname] == nil then
                self.modactioncomponents[modname] = {}
            end
            table.insert(self.modactioncomponents[modname], id)
            if self.actionreplica ~= nil then
                self.actionreplica.modactioncomponents[modname]:set(self.modactioncomponents[modname])
            end
        end
    end
end
```

**核心几行**：

```lua
local id = ACTION_COMPONENT_IDS[name]
if id ~= nil then
    table.insert(self.actioncomponents, id)
    if self.actionreplica ~= nil then
        self.actionreplica.actioncomponents:set(self.actioncomponents)
    end
end
```

**逐行解释**：

1. **`ACTION_COMPONENT_IDS[name]`** —— 查表，得到组件名对应的**整数 ID**（比如 `workable` → 47）。
2. **`table.insert(self.actioncomponents, id)`** —— 把这个 ID 加入实体的 `actioncomponents` 数组。
3. **`self.actionreplica.actioncomponents:set(...)`** —— 把整个数组同步给客户端（通过 actionreplica 这个特殊 replica）。

**这就是 7.1.4 节那个 `assert(#ACTION_COMPONENT_NAMES <= 255, ...)` 的伏笔**——组件 ID 是一个 **byte（最多 255 种）**，整个数组用一串 byte 同步——**省网络带宽**。

#### 第三步：何时调用 `RegisterComponentActions`？

回看回顾 6.1.2 节的 `EntityScript:AddComponent`（`entityscript.lua` 第 610-646 行）的最后一行：

```lua
self:RegisterComponentActions(name)
```

**每次 AddComponent 时自动调用！** —— 这是 7.1.7 节末尾我们说的"动作注册"那一步。

**整个流程**：

```
inst:AddComponent("workable")
      ↓
[1] LoadComponent("workable")
[2] ReplicateComponent("workable")
[3] cmp(self) 实例化
[4] self.components.workable = cmp
[5] AddComponentPostInit 回调
[6] RegisterComponentActions("workable")
        ├── 查 ACTION_COMPONENT_IDS["workable"] → 假设是 47
        ├── self.actioncomponents push 47
        └── actionreplica.actioncomponents:set(...)  网络同步
```

**结果**：实体身上的 `actioncomponents` 数组就是"我有哪些组件能产生动作"的精简列表。**这个数组只装"在 COMPONENT_ACTIONS 表里出现过的组件"**——比如 `inspectable`、`burnable`、`fueled` 等只挂"内部状态"的组件**不会进 actioncomponents**（因为它们没在 COMPONENT_ACTIONS 里登记）。

> **进阶提示**：**`self.components` 是"实体身上的所有组件"，`self.actioncomponents` 是"其中能产生动作的组件 ID 列表"** —— 后者是前者的子集。`CollectActions` 用后者，避免遍历无关组件浪费 CPU。

#### 第四步：客户端的 actionreplica 是什么？

注意源码里反复出现 `self.actionreplica`：

```lua
if self.actionreplica ~= nil then
    self.actionreplica.actioncomponents:set(self.actioncomponents)
end
```

**`actionreplica`** 是一个特殊的 **classified 实体**——专门用于同步"这个实体能做哪些动作"给客户端。它有两个 net 数组字段：

```lua
self.actioncomponents       -- 原版组件 ID 数组
self.modactioncomponents[modname]  -- Mod 组件 ID 数组（按 mod 分组）
```

**为什么要单独同步**？因为客户端的 ActionPicker 也要做 `CollectActions`——要选出"我可以做什么动作"显示给玩家。**客户端没有完整的 components 表**（很多组件不需要 replica），所以必须**单独同步 actioncomponents 数组**。

**这就是为什么 7.2.8 节会强调"客户端服务端 AddComponentAction 必须一致"**——如果 client/server 注册的组件名不同，编号会错位，整个动作系统乱套。

#### 第五步：观察一棵树的 actioncomponents

进游戏，控制台：

```lua
c_select(c_findnext("evergreen"))   -- 选中一棵松树
print(c_sel().actioncomponents)
-- 输出: 一个 table，里面是几个数字
print(#c_sel().actioncomponents)
-- 输出: 3 或 4 之类的小整数
```

每个数字是 `ACTION_COMPONENT_IDS` 里的 ID。要查它代表哪个组件：

```lua
for i, v in ipairs(c_sel().actioncomponents) do
    print(i, v, "→", COMPONENT_ACTIONS_MASTER and "?" or "?")
    -- 实际查 ACTION_COMPONENT_NAMES[v] 即可
end
```

注意 `ACTION_COMPONENT_NAMES` 是 `componentactions.lua` 的 `local` 变量——**不在控制台直接可见**。要看的话需要在 modmain 里 `local ACN = require("componentactions") ... ` 或者直接读源码。

> **新手记忆**：**actioncomponents 数组就是实体"动作能力的指纹"** —— 你看一个实体的 actioncomponents = `{47, 23, 15}` 就知道它有哪些能产生动作的组件。**这个数组在网络上只占 3 个 byte**——比同步组件名字字符串省 90% 以上带宽。

---

### 7.2.6 进阶：ISVALID —— 动作"事后校验"机制

#### 第一步：再看 ISVALID 的特殊性

回顾 7.2.3 节我们说：**ISVALID 不产生动作，只验证已知动作**。

源码定义在 `componentactions.lua` 第 3013-3031 行：

```3013:3030:scripts/componentactions.lua
    ISVALID = --args: inst, action, right
    {
        pickable = function(inst, action, right)
            local valid = right and action == ACTIONS.SCYTHE and inst:HasTag("pickable")

            if not valid then return false end

            return IsValidScytheTarget(inst)
        end,

        lunarhailbuildup = function(inst, action, right)
            return action == ACTIONS.REMOVELUNARBUILDUP and inst:HasTag("LunarBuildup")
        end,

        workable = function(inst, action, right)
            return (right or action ~= ACTIONS.HAMMER) and
                inst:HasTag(action.id.."_workable")
        end,
```

**整个 ISVALID 表只有 3 个 collector**——用得很少，但有特殊用途。

#### 第二步：为什么需要 ISVALID？

考虑这样一个场景：

1. 玩家手里有镰刀（`scythe` tag），鼠标悬停在花上
2. SCENE 里 `pickable` 组件 push 了 `ACTIONS.PICK`（普通采摘）
3. SCENE 或者 USEITEM 里其他组件 push 了 `ACTIONS.SCYTHE`（用镰刀割草）
4. 现在 actions = {PICK, SCYTHE}

**但是问题来了**——ACTIONS.SCYTHE 是不是真的能在这朵花上执行？它是用来割**草丛**的，不是用来割花的——但它的 collector 没有那么细粒度的检查。

**这时 ISVALID 上场**：

```lua
pickable = function(inst, action, right)
    local valid = right and action == ACTIONS.SCYTHE and inst:HasTag("pickable")
    if not valid then return false end
    return IsValidScytheTarget(inst)
end,
```

**"如果是 SCYTHE 动作并且这是 pickable 实体——再做一次 IsValidScytheTarget 检查"**。如果该实体不是合法的 scythe 目标（比如是花而不是草），返回 false——**这个 SCYTHE 动作被从列表里剔除**。

#### 第三步：ISVALID 的调用时机

看 `componentactions.lua` 第 3167-3194 行的 `EntityScript:IsActionValid`：

```3167:3194:scripts/componentactions.lua
function EntityScript:IsActionValid(action, right)
    if action.rmb and action.rmb ~= right then
        return false
    end
    local isvalid_list = COMPONENT_ACTIONS.ISVALID
    for _, v in ipairs(self.actioncomponents) do
        local validator = isvalid_list[ACTION_COMPONENT_NAMES[v]]
        if validator ~= nil and validator(self, action, right) then
            return true
        end
    end
    if self.modactioncomponents then
        for modname, cmplist in pairs(self.modactioncomponents) do
            isvalid_list = CheckModComponentActions(self, modname)
            isvalid_list = (isvalid_list and isvalid_list.ISVALID) or nil
            if isvalid_list then
                local namemap = CheckModComponentNames(self, modname)
                for _, v in ipairs(cmplist) do
                    local validator = isvalid_list[namemap[v]]
                    if validator ~= nil and validator(self, action, right) then
                        return true
                    end
                end
            end
        end
    end
    return false
end
```

**逻辑**：

1. **首先检查 `action.rmb` 和 `right` 是否匹配**（如果动作只允许右键、当前是左键，直接 false）
2. **遍历实体的 actioncomponents** —— 任何一个 ISVALID collector 返回 true 就视为有效
3. **遍历 mod 的 actioncomponents** —— 同上

**注意是 OR 逻辑**：**只要有一个 ISVALID collector 同意就行**。这意味着 ISVALID 是"**白名单**"风格——而不是"黑名单"。

#### 第四步：什么时候真的需要 ISVALID？

**绝大多数自定义动作不需要 ISVALID**——直接在 SCENE/USEITEM 等 collector 里做条件检查就够了。

**唯一需要 ISVALID 的场景**：**动作可能从多个组件被 push 进来**，但**只有部分实体真正应该接受它**。这时你需要一个"统一的有效性入口"。

**`workable` 的例子最经典**：

```lua
workable = function(inst, action, right)
    return (right or action ~= ACTIONS.HAMMER) and
        inst:HasTag(action.id.."_workable")
end,
```

`workable` 组件可以接受 CHOP / MINE / DIG / HAMMER 等多种动作——但**不是所有 workable 都接受所有动作**。**树**接受 CHOP（有 `CHOP_workable` tag）但不接受 MINE；**岩石**接受 MINE 但不接受 CHOP。

这个 ISVALID 通过 **`inst:HasTag(action.id.."_workable")`** 这种动态 tag 检查—— **"action 的名字 + `_workable`"** —— 来判断该实体是否接受某个具体的工具动作。

**这是 ISVALID 最优雅的应用**——一个 collector 处理多种动作的有效性。

> **老手提示**：**写自定义动作时**，先问自己：动作的"是否有效"逻辑能不能在 collector 里写完？如果可以，就**不需要** ISVALID。如果"动作可能从外部其他组件 push 进来、需要统一过滤"——那就用 ISVALID。**新手 90% 用不到，老手偶尔用一次**。

---

### 7.2.7 老手进阶：`AddComponentAction` 完整流程 + Mod 命名空间

#### 第一步：两个层级的 `AddComponentAction`

`AddComponentAction` 有两个版本：

**(A) Mod 用的（modmain.lua 里调用）**——`modutil.lua` 第 478-481 行：

```478:481:scripts/modutil.lua
	env.AddComponentAction = function(actiontype, component, fn)
		-- just past this along to the global function
		AddComponentAction(actiontype, component, fn, env.modname)
	end
```

**只是 wrapper**——把 `env.modname` 作为第 4 参数传给全局的 `AddComponentAction`。

**(B) 全局的 `AddComponentAction`（componentactions.lua 第 3072 行）**——所有逻辑在这里：

```3072:3083:scripts/componentactions.lua
function AddComponentAction(actiontype, component, fn, modname)
    if MOD_COMPONENT_ACTIONS[modname] == nil then
        MOD_COMPONENT_ACTIONS[modname] = { [actiontype] = {} }
        MOD_ACTION_COMPONENT_NAMES[modname] = {}
        MOD_ACTION_COMPONENT_IDS[modname] = {}
    elseif MOD_COMPONENT_ACTIONS[modname][actiontype] == nil then
        MOD_COMPONENT_ACTIONS[modname][actiontype] = {}
    end
    MOD_COMPONENT_ACTIONS[modname][actiontype][component] = fn
    table.insert(MOD_ACTION_COMPONENT_NAMES[modname], component)
    MOD_ACTION_COMPONENT_IDS[modname][component] = #MOD_ACTION_COMPONENT_NAMES[modname]
end
```

#### 第二步：拆解全局 `AddComponentAction` 的 5 步

**步骤 1：初始化 Mod 命名空间**

```lua
if MOD_COMPONENT_ACTIONS[modname] == nil then
    MOD_COMPONENT_ACTIONS[modname] = { [actiontype] = {} }
    MOD_ACTION_COMPONENT_NAMES[modname] = {}
    MOD_ACTION_COMPONENT_IDS[modname] = {}
elseif MOD_COMPONENT_ACTIONS[modname][actiontype] == nil then
    MOD_COMPONENT_ACTIONS[modname][actiontype] = {}
end
```

**第一次添加这个 mod 的 ComponentAction 时**：创建三张空表：

- `MOD_COMPONENT_ACTIONS[modname]` —— 这个 mod 的 actiontype → component → fn 嵌套表
- `MOD_ACTION_COMPONENT_NAMES[modname]` —— 这个 mod 的组件名数组
- `MOD_ACTION_COMPONENT_IDS[modname]` —— 这个 mod 的组件名 → ID 反查

**和原版的 `COMPONENT_ACTIONS` / `ACTION_COMPONENT_NAMES` / `ACTION_COMPONENT_IDS` 三件套对应**——只是多了一层 modname 命名空间。

**步骤 2：注册 collector 函数**

```lua
MOD_COMPONENT_ACTIONS[modname][actiontype][component] = fn
```

**写入 mod 的二维表**——以后 `CollectActions` 会从这里查。

**步骤 3：分配组件 ID**

```lua
table.insert(MOD_ACTION_COMPONENT_NAMES[modname], component)
MOD_ACTION_COMPONENT_IDS[modname][component] = #MOD_ACTION_COMPONENT_NAMES[modname]
```

**和 7.1.5 节 ACTION_MOD_IDS / MOD_ACTIONS_BY_ACTION_CODE 同款**——给这个 mod 的组件分配一个**该 mod 命名空间内的递增 ID**。第一个 ID = 1，第二个 ID = 2。

#### 第三步：Mod 命名空间的运行时使用

回看 `RegisterComponentActions` 的下半段（第 3093-3106 行）：

```3093:3106:scripts/componentactions.lua
    for modname, idmap in pairs(MOD_ACTION_COMPONENT_IDS) do
        id = idmap[name]
        if id ~= nil then
            if self.modactioncomponents == nil then
                self.modactioncomponents = { [modname] = {} }
            elseif self.modactioncomponents[modname] == nil then
                self.modactioncomponents[modname] = {}
            end
            table.insert(self.modactioncomponents[modname], id)
            if self.actionreplica ~= nil then
                self.actionreplica.modactioncomponents[modname]:set(self.modactioncomponents[modname])
            end
        end
    end
```

**遍历所有 mod 的 idmap**——如果某个 mod 注册过 `name` 这个组件，把对应 ID 加进 `self.modactioncomponents[modname]`。

**最后**：**每个 mod 的 modactioncomponents 单独同步**到客户端（通过 `actionreplica.modactioncomponents[modname]:set(...)`）。

**这种设计的核心好处**：**两个 mod 都注册了 `psionic` 组件——两个 mod 各自的 ID 都从 1 开始，互不干扰**。客户端按 modname + id 反查就能精确取到对应的 collector。

#### 第四步：`CollectActions` 怎么处理 mod 命名空间？

回看 `EntityScript:CollectActions` 的下半段（第 3150-3164 行）：

```3150:3164:scripts/componentactions.lua
    if self.modactioncomponents ~= nil then
        for modname, cmplist in pairs(self.modactioncomponents) do
            t = CheckModComponentActions(self, modname)
            t = t and t[actiontype] or nil
            if t ~= nil then
                local namemap = CheckModComponentNames(self, modname)
                for i, v in ipairs(cmplist) do
                    local collector = t[namemap[v]]
                    if collector ~= nil then
                        collector(self, ...)
                    end
                end
            end
        end
    end
```

**对每个 mod 单独走一遍**——按 modname 取出对应的 actiontype 表 + 组件名查询表，然后**遍历该 mod 在该实体上注册过的组件**，调用 collector。

**整个 CollectActions 的执行**：

```
[1] 处理原版组件
       ├── 遍历 self.actioncomponents 数组
       └── 对每个 ID，查 ACTION_COMPONENT_NAMES → 取出组件名 → 查 COMPONENT_ACTIONS[actiontype][组件名] → 调用 collector

[2] 处理每个 mod 的组件
       └── 对每个 mod，遍历 self.modactioncomponents[modname]
              └── 查 MOD_ACTION_COMPONENT_NAMES[modname] → 取出组件名 → 查 MOD_COMPONENT_ACTIONS[modname][actiontype][组件名] → 调用 collector
```

**复杂吗？是。但严密吗？也是。**——这种"双层命名空间"让饥荒成为了**Mod 友好**的引擎。

#### 第五步：完整的 `AddComponentAction` 调用范例

写在 modmain.lua 顶层：

```lua
-- 给 psionic 组件挂 SCENE 类型的 collector
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_full") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
```

**意思**：**当带有 `psionic` 组件的实体在场景中被玩家右键悬停**——如果实体有 `psionic_full` tag（充满），push `ACTIONS.PSIONIC_BURST`。

**注意几个细节**：

- **第一参数 `"SCENE"` 是字符串**——不要传 `SCENE`（会找不到这个全局变量）
- **第二参数 `"psionic"` 必须是组件名（小写）**——和 `inst.components.psionic` 的 key 完全一致
- **第三参数是 collector 函数**——签名按 SCENE 的 `(inst, doer, actions, right)`

#### 第六步：跟 7.1 联动看完整链路

回顾 7.1.7 末尾我们说**完整的"自定义动作三步走"**：

```lua
-- 1. AddAction：声明动作存在
AddAction("PSIONIC_BURST", "释放灵能", function(act)
    if act.target ~= nil and act.target.components.psionic ~= nil then
        return act.target.components.psionic:Burst()
    end
    return false
end)

-- 2. AddComponentAction：声明动作什么时候出现
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_full") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)

-- 3. AddStategraphActionHandler：声明动作怎么执行（动画）
-- 这个我们 7.3 节再讲
```

**前两步合起来——你的灵能晶簇右键就有"释放灵能"选项了**。第 3 步管动画响应——下一节讲。

> **新手记忆**：**`AddComponentAction(actiontype, component_name, collector_fn)`** —— 三个参数。**actiontype 决定"哪种场景出现这个动作"、component_name 决定"绑在哪个组件上"、collector_fn 决定"什么条件下产生动作"**。三者缺一不可。

---

### 7.2.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：actiontype 字符串拼写错误

```lua
-- ❌ 错：actiontype 拼错（小写）
AddComponentAction("scene", "psionic", function(...) end)
-- ❌ 错：写成中文拼音或别的
AddComponentAction("SCENERY", "psionic", function(...) end)

-- ✅ 对：必须是 6 种之一的精确字符串
AddComponentAction("SCENE", "psionic", function(...) end)
```

**原因**：`COMPONENT_ACTIONS[actiontype]` 是字符串查表——拼错就查不到。但**Klei 不会报错**（注册到 `MOD_COMPONENT_ACTIONS` 里，但永远不会被 `CollectActions` 命中）——你只会看到"我的动作怎么不出现"。

**调试方法**：在 `componentactions.lua` 第 3140-3142 行有这样的提示：

```3140:3142:scripts/componentactions.lua
    if t == nil then
        print("Action type", actiontype, "doesn't exist in the table of component actions. Is your component name correct in AddComponentAction?")
        return
```

——但这只在 **CollectActions 调用了不存在的 actiontype** 时才打印。如果你 AddComponentAction 拼错了，反向是没人会调用那个错误的 actiontype——**没有报错就只能凭经验判断**。

**6 种合法 actiontype**：`SCENE` / `USEITEM` / `POINT` / `EQUIPPED` / `INVENTORY` / `ISVALID`。

#### 陷阱 2：组件名大小写错误

```lua
-- ❌ 错：用了大写
AddComponentAction("SCENE", "Psionic", function(...) end)
-- ❌ 错：用了驼峰
AddComponentAction("SCENE", "psionicComponent", function(...) end)

-- ✅ 对：组件名必须和 inst.components.xxx 的 key 完全一致
AddComponentAction("SCENE", "psionic", function(...) end)
```

**原因**：`AddComponent("psionic")` 时引擎用 `name` 直接做 key —— `RegisterComponentActions(name)` 内部用同一个字符串查 `MOD_ACTION_COMPONENT_IDS[modname][name]`。**大小写不匹配——查不到 ID——RegisterComponentActions 是空操作——CollectActions 永远不会调用你的 collector**。

**这是最常见的"我的 collector 不被触发"原因**。**写组件时坚持小写组件名**——和你 `AddComponent("xxx")` 时用的一字不差。

#### 陷阱 3：忘了 collector 不返回值

```lua
-- ❌ 错：尝试 return 一个 action
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_full") then
        return ACTIONS.PSIONIC_BURST    -- 引擎不看返回值！
    end
end)

-- ✅ 对：用 table.insert(actions, ...)
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_full") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
```

**原因**：除了 ISVALID，所有 collector 都是**通过修改 actions 表参数**输出动作——return 是没用的（Lua 不报错只是会被忽略）。**ISVALID 才需要 return bool**。

#### 陷阱 4：客户端服务端 AddComponentAction 不一致

```lua
-- ❌ 错：在某个条件分支里 AddComponentAction
if SOME_CONFIG_FROM_USER then
    AddComponentAction("SCENE", "psionic", function(...) end)
end
```

**问题**：**modmain.lua 在客户端和服务端各跑一次**——如果两端的执行路径不同，两端的 `MOD_COMPONENT_ACTIONS` / `MOD_ACTION_COMPONENT_IDS` 内容就不一致——**ID 编号错位**——客户端发 RPC 时按客户端 ID 编码，服务端按服务端 ID 解码——**完全不同的 collector 被调用**！

**典型崩溃方式**：

```
[client] mod: 注册 "psionic" → ID 1, 注册 "myothercomp" → ID 2
[server] mod: 注册 "myothercomp" → ID 1, 注册 "psionic" → ID 2 (条件分支不同)
[client] 玩家右键灵能晶簇 → ID 1（client 认为是 psionic）
[server] 收到 → ID 1（server 看是 myothercomp）→ 调用错误的 collector → 崩溃 / 错误动作
```

**解决方案**：

```lua
-- ✅ 对：AddComponentAction 永远在 modmain 顶层无条件执行
AddComponentAction("SCENE", "psionic", function(...) end)

-- 如果需要"按配置开关"——在 collector 内部检查
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if not GetModConfigData("ENABLE_BURST") then return end
    if right and inst:HasTag("psionic_full") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
```

`componentactions.lua` 第 3054-3070 行有专门的**警告函数 `ModComponentWarning`**——它会在不一致时打印 `"ERROR: Mod component actions are out of sync for mod xxx"`——**看到这条日志就要立刻修**。

#### 陷阱 5：collector 里访问 doer.components.xxx（客户端没有）

```lua
-- ❌ 错：客户端可能崩溃
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if doer.components.sanity:GetPercent() > 0.5 then  -- ❌ 客户端 doer.components.sanity 是 nil！
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)

-- ✅ 对：用 replica
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if doer.replica.sanity ~= nil and doer.replica.sanity:GetPercent() > 0.5 then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
```

**原因**：collector 在**客户端服务端都跑**——客户端没有完整的 `inst.components`（很多组件不复制），但有 `inst.replica`。**collector 里访问字段必须考虑双端兼容**——回顾 6.2.6 节关于 replica 的讨论。

**经验**：**collector 里做条件判断**——优先用 `HasTag` / `replica.xxx` / 普通字段——**避免直接访问 components.xxx**。

#### 陷阱 6：忘了考虑 right 参数

```lua
-- ❌ 错：左键右键都触发
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if inst:HasTag("psionic_full") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
-- 结果：左键悬停灵能晶簇 → 自动开始走过去执行 burst（玩家莫名其妙）
-- 右键悬停 → 也产生 burst
-- 两个都触发，左键变成"自动激活"——非常糟糕的 UX

-- ✅ 对：明确区分左右键
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_full") then  -- 仅右键
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
```

**原因**：**饥荒的左键是"主要动作"、右键是"次要/特殊动作"**——绝大多数自定义动作应该用右键。**忘记 right 检查会让你的动作和左键的捡起/检查/攻击产生冲突**——即使 priority 排序避开了，玩家体验也会很奇怪。

> **设计经验**：**右键专属动作 = `if right and ... then`**。**左键专属动作 = `if not right and ... then`**。**双键都触发的极少**（PICKUP 是这种特例——左右键都能捡起）。

#### 设计经验三条

**经验 1：collector 必须很轻量**

**collector 在每帧都被调用**（玩家鼠标悬停目标的每一帧）——做太多事会让游戏卡顿。

```lua
-- ❌ 错：collector 里查找全图实体
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    local players = TheSim:FindEntities(0, 0, 0, 100, {"player"})  -- 每帧扫全图！
    if #players > 5 then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)

-- ✅ 对：把扫描挪到组件的 task 里、collector 只查 tag/字段
-- 在 psionic 组件里：
function Psionic:OnUpdate()
    if #nearbyplayers > 5 then
        self.inst:AddTag("psionic_burst_ready")
    end
end

-- collector 里：
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_burst_ready") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)
```

**心法**：**把"重逻辑"放在组件的 task 里、把结果"投影"成一个 tag、collector 只查 tag**。回顾 6.5.7 节我们讲过的"用 SourceModifierList 把多源叠加移到组件内部"——是一样的"轻 collector / 重组件" 哲学。

**经验 2：一个组件可以挂多种 ActionType**

```lua
-- 同时挂 SCENE、USEITEM、INVENTORY
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    -- 场景里点击灵能晶簇
end)

AddComponentAction("INVENTORY", "psionic", function(inst, doer, actions, right)
    -- 在背包里右键灵能晶簇
end)

AddComponentAction("USEITEM", "psionic", function(inst, doer, target, actions, right)
    -- 拖动灵能晶簇到另一个目标上
end)
```

**好处**：让你的物品**在不同场景下都能响应**——场景里能激活、背包里能激活、拖到目标上能转移灵能。**根据物品需要支持的交互方式选择 ActionType**。

**经验 3：collector 里不要 PushEvent**

```lua
-- ❌ 错：在 collector 里推事件
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    inst:PushEvent("checking_burst")  -- ❌ 每帧都推！
    table.insert(actions, ACTIONS.PSIONIC_BURST)
end)
```

**原因**：collector 每帧调用——PushEvent 也每帧推送——监听者每帧都被通知——**性能灾难**。**collector 是"声明能力"**——纯查询、纯添加 action。

**所有"做事"逻辑应该在 ACTIONS.XXX.fn 里**——回顾 7.1.6 节关于 fn 的讲解。

---

### 7.2.9 小结

- **`COMPONENT_ACTIONS` 是一张二维查找表**——`actiontype → component_name → collector_fn`。引擎根据"玩家想做什么操作场景"+"目标实体有哪些组件"，动态生成可用动作列表。
- **6 种 ActionType**：`SCENE`（场景实体）/ `USEITEM`（手里物品+目标）/ `POINT`（手里物品+地面）/ `EQUIPPED`（装备物品+目标）/ `INVENTORY`（背包内右键）/ `ISVALID`（事后校验）。**最常用是 SCENE 和 INVENTORY**。
- **collector 函数 5 种签名**——核心模式都是 `(inst, doer, ..., actions, right)` —— `inst` 是组件所在实体（不一定是目标！），`doer` 是执行者，`actions` 是输出参数（直接 push）。**ISVALID 例外，签名是 `(inst, action, right)`，要 return bool**。
- **`actioncomponents` 数组 + `actionreplica`** 让客户端也能高效查 "我能做什么动作"——**用 byte 编码代替字符串名字，省网络带宽**。每次 `AddComponent` 自动调 `RegisterComponentActions` 把组件 ID 加进去。
- **Mod 的双注册表（`MOD_COMPONENT_ACTIONS` / `MOD_ACTION_COMPONENT_NAMES` / `MOD_ACTION_COMPONENT_IDS`）** 让多个 Mod 互不冲突——和 7.1.5 节 ACTIONS 的 mod 命名空间设计完全一致。
- **`AddComponentAction(actiontype, component, fn)` 是 Mod 的标准接口**——**必须在 modmain.lua 顶层无条件调用**，否则 client/server 不一致会出大问题。
- **6 个常见陷阱**：① actiontype 拼写错 ② 组件名大小写错 ③ collector 错误地 return ④ client/server 注册不一致 ⑤ collector 里访问 components 而不是 replica ⑥ 忘记区分 right 左右键。
- **三条设计经验**：① collector 必须轻量（不要每帧扫全图）② 一个组件可挂多种 ActionType ③ collector 不做副作用（不 PushEvent / 不改字段）。

> **下一节预告**：7.3 节我们打开 **`stategraph` 系统**——一个动作被 push 进 actions 列表、再被 ActionPicker 选中后，**动画响应**是怎么实现的？StateGraph 通过 `ActionHandler` 注册"收到这个动作时切换到哪个状态"——而每个状态控制动画播放、声音播放、伤害判定时机。我们会看到为什么 `chop_loop` 状态会在第 N 帧调用 `ACTIONS.CHOP.fn`、为什么"动画里的某一帧"和"动作的执行"必须严格对齐。

---

## 7.3 ActionHandler——StateGraph 如何响应动作

### 本节导读

7.1 节我们看了 **ACTIONS 表**（动词字典），7.2 节看了 **ComponentAction**（哪些动作"可见"）。**现在动作已经被 PlayerActionPicker 选中、被打包成 BufferedAction，准备执行了——但是**：

> **接下来是动画环节**。玩家不能凭空"砍树成功"——他需要走过去、举起斧头、挥舞、击中。这些动作背后是一套**状态机（StateGraph）**——`idle → chop_start → chop → idle` 的状态切换、每个状态对应一段动画、动画里某一帧触发"实际砍树伤害"。**`ACTIONS.CHOP` 是怎么被翻译成"切到 chop_start 状态"的？答案就是 ActionHandler**。

这一节我们打开 **`scripts/stategraph.lua`** 和**玩家专用的 `scripts/stategraphs/SGwilson.lua`**——把 StateGraph 系统里和 Action 直接相关的部分讲清楚：

- `ActionHandler` 是什么、它的 3 个字段如何工作
- `actionhandlers` 注册表在玩家 SG 里长什么样
- `State` 类的完整结构（`onenter / onexit / onupdate / timeline / events`）
- **`TimeEvent`** —— 动画帧驱动业务的关键机制（为什么 `PerformBufferedAction` 在第 2 帧而不是第 0 帧）
- StateTag 怎么影响 ActionHandler 的 condition
- `AddStategraphActionHandler` 给自定义 SG / 给玩家 SG 加新动作的标准流程

> **新手**从 7.3.1-7.3.3 起步——看一次砍树的完整状态切换、理解 ActionHandler 是个什么东西、知道 actionhandlers 表在哪里；**进阶读者**继续看 7.3.4-7.3.6，深入 State 类的 5 个核心字段、TimeEvent 的"动画驱动业务"模式、StateTag 在 condition 里的妙用；**老手**跳到 7.3.7-7.3.8，看 `AddStategraphActionHandler` 的内部加载机制（`ModManager:GetPostInitData`）、三种典型注册模式、六个最容易踩的坑——尤其是"动画文件不匹配 / 状态 tag 漏 RemoveStateTag / 动作中途切换状态" 这些隐蔽 bug。

---

### 7.3.1 快速入门：从一次"砍树"看 StateGraph 是什么

#### 第一步：在游戏里观察玩家的状态序列

进入饥荒，**手里拿着斧头**对着一棵松树点击——

**你看到的**：

1. 玩家**走向松树**（动画：`run`）
2. 走到位置后**停下**（动画：`run_stop`）
3. **举起斧头**（动画：`chop_pre` ≈ 6 帧）
4. **挥砍**（动画：`chop_loop` ≈ 12 帧）
5. **回到 idle**（动画：`idle_loop`）

**你不知道的**：每一段动画对应玩家身上 StateGraph 的**一个状态**——`run → run_stop → chop_start → chop → idle`。状态切换由**事件**触发——`run` 状态收到"到达目标"事件 → 切到 `run_stop`、`run_stop` 收到"动画结束"事件 → 切到 `chop_start`、`chop_start` 收到"动画结束"事件 → 切到 `chop`。

**整个 StateGraph 系统**就是一个**状态机引擎**——每个游戏实体可以挂一份 SG（玩家的、猪人的、蜘蛛的等等），管理"当前在做什么"。

#### 第二步：用控制台亲眼看到状态切换

进游戏，控制台：

```lua
-- 实时看自己的当前状态
print(ThePlayer.sg.currentstate.name)

-- 看完整 SG 调试信息
print(ThePlayer.sg)
-- 输出: sg="wilson", state="chop", time=0.13, tags = "prechop,chopping,working,"
```

**砍树的过程中持续运行这个命令**，你会看到 `state` 字段变化：`run → run_stop → chop_start → chop → idle`。

#### 第三步：StateGraph 在 Action 流程中的位置

把 7.1-7.2 + 7.3 的内容串起来，从"右键斧头点击树"到"树倒下"的完整链路：

```
[1] 玩家鼠标悬停松树
       ↓
[2] PlayerActionPicker.GetActions
       ├── 调 tree:CollectActions("SCENE", doer, actions, right)
       │     └── workable 的 collector push CHOP
       └── 返回 actions = {CHOP, ...}
       ↓
[3] PlayerActionPicker 按 priority 选中 CHOP
       ↓
[4] 玩家点击 → 创建 BufferedAction(doer, target, ACTIONS.CHOP, axe)
       ↓
[5] BufferedAction 通过 PlayerController 进入 SG 的事件队列
       ↓
[6] 玩家 SG 当前是 "idle" 状态、收到 "doaction" 事件
       ↓
[7] ★★★ 玩家 SG 的 actionhandlers[ACTIONS.CHOP] 被触发：
       - ActionHandler 的 deststate("chop_start") 返回新状态名
       - SG 切换到 "chop_start" 状态  ★★★
       ↓
[8] chop_start.onenter 播放 chop_pre 动画
       ↓
[9] 动画结束 → events.animover → 切到 "chop" 状态
       ↓
[10] chop.onenter 播放 chop_loop 动画
       ↓
[11] chop.timeline[第 2 帧] → inst:PerformBufferedAction()
       ↓
[12] PerformBufferedAction → 执行 ACTIONS.CHOP.fn(act)
       ↓
[13] CHOP.fn → DoToolWork → tree.components.workable:WorkedBy(...)
       ↓
[14] workable 减少进度、PushEvent("worked")
       ↓
[15] tree 监听 worked 事件 → 倒下、生成木头
```

**ActionHandler 出现在第 [7] 步**——这是连接"动作系统"和"状态机系统"的**桥梁**。

#### 第四步：看 ActionHandler 这个"桥梁"的真身

打开 `SGwilson.lua` 第 850-868 行：

```850:868:scripts/stategraphs/SGwilson.lua
local actionhandlers =
{
    ActionHandler(ACTIONS.CHOP,
        function(inst)
            if inst:HasTag("beaver") then
                return not inst.sg:HasStateTag("gnawing") and "gnaw" or nil
			elseif inst.GetModuleTypeCount and inst:GetModuleTypeCount("spin") > 0 then
				return not inst.sg:HasStateTag("prespin")
					and (inst.sg:HasStateTag("spinning") and
						"wx_spin" or
						"wx_spin_start")
					or nil
            end
            return not inst.sg:HasStateTag("prechop")
                and (inst.sg:HasStateTag("chopping") and
                    "chop" or
                    "chop_start")
                or nil
        end),
```

**这就是 `ACTIONS.CHOP` → `chop_start` 的桥**——一个 `ActionHandler` 实例，第一参数是动作对象、第二参数是**返回目标状态名的函数**。

逻辑很有意思：

- 如果玩家是**海狸形态**（`HasTag("beaver")`）→ 切到 `gnaw` 状态（海狸不用斧头，啃）
- 如果玩家是**WX-78 + 装了 spin 模块** → 切到 `wx_spin_start` 或 `wx_spin`
- **正常情况**：根据当前 SG 的 StateTag 判断——
  - 如果**正在 prechop**（已经在举斧头）→ 不再切（返回 nil 表示不响应）
  - 如果**正在 chopping**（已经在挥砍）→ 切到 `chop`（继续连击）
  - 否则 → 切到 `chop_start`（新一轮）

> **新手记忆**：**ActionHandler 是一个"如果收到这个 Action，应该切到哪个 State"的规则**。它接收 Action 对象、返回目标状态名（字符串）。**整个 Action 系统的核心就是这一句**——你写自定义动作时，最关键的就是定义这条规则。

---

### 7.3.2 快速入门：ActionHandler 类的三个核心字段

#### 第一步：源码位置和定义

`ActionHandler` 类定义在 **`scripts/stategraph.lua`** 第 159-171 行：

```159:171:scripts/stategraph.lua
ActionHandler = Class(
    function(self, action, state, condition)

        self.action = action

        if type(state) == "string" then
            self.deststate = function(_) return state end
        else
            self.deststate = state
        end

        self.condition = condition
    end)
```

**就一个 `Class` + 12 行**——非常简单。**3 个字段**：

| 字段 | 类型 | 含义 |
|------|------|------|
| `self.action` | Action 对象 | 哪个动作触发它（如 `ACTIONS.CHOP`） |
| `self.deststate` | function(inst, action) → string | 返回目标状态名的函数 |
| `self.condition` | function(inst, action) → bool | 可选：是否允许这个 ActionHandler 响应（默认 true） |

#### 第二步：`deststate` 字段的两种写法

**写法 A：传字符串（最简单）**

```lua
ActionHandler(ACTIONS.TERRAFORM, "terraform"),
```

——`ACTIONS.TERRAFORM` 触发时切到 `"terraform"` 状态。**ActionHandler 内部把字符串包装成函数**：

```lua
if type(state) == "string" then
    self.deststate = function(_) return state end
```

**写法 B：传函数（动态决策）**

```lua
ActionHandler(ACTIONS.CHOP,
    function(inst)
        if inst:HasTag("beaver") then return "gnaw" end
        return "chop_start"
    end),
```

——根据**实体当前状态**（HasTag、SG 状态等）决定切到哪个状态。**这是绝大多数动作的写法**——因为不同角色 / 不同形态需要不同动画。

**函数签名**：`function(inst, action)` —— **两个参数都可选**：

- `inst` —— SG 所属的实体（一般是玩家）
- `action` —— BufferedAction 实例（带 doer/target/invobject 等）

**返回值**：

- 字符串 —— 目标状态名
- `nil` —— 表示"不切换"（**这次动作请求被忽略**，但不报错）

#### 第三步：`condition` 字段的作用

**`condition` 是可选的"额外门控"** —— 在 SG 决定调用 deststate 之前先判断"这个 ActionHandler 是否应该响应"。**返回 true 才会进入 deststate；false 就被跳过**。

**实例**（不同 SG 里的写法）：

```lua
ActionHandler(ACTIONS.LOOKAT,
    function(inst, action)
        return "describeghost"
    end,
    function(inst, action)
        return inst:HasTag("ghost")
    end),
```

——只有玩家是鬼魂时这个 ActionHandler 才生效。

**condition 和 deststate 函数内 if 的区别**：

- 在 deststate 里写 `if cond then return state else return nil end` —— 也能起到过滤作用
- **使用 condition 字段更清晰**——明确表达"门控"和"决策"的两层逻辑

但实际 Klei 源码里——**80% 的 ActionHandler 都不用 condition 字段**——把所有逻辑塞进 deststate 里。**新手期间也可以不用**。

#### 第四步：运行时的调用流程

当 SG 收到一个 BufferedAction 时，引擎会做这个流程：

```
[1] inst.sg:HandleEvents() 处理事件队列
       ↓
[2] 找到 "doaction" 类型的事件，提取 BufferedAction
       ↓
[3] handler = sg.actionhandlers[buffered_action.action]
       ↓
[4] if handler.condition and not handler.condition(inst, buffered_action) then
        skip      -- 不响应
[5] state_name = handler.deststate(inst, buffered_action)
       ↓
[6] if state_name == nil then
        skip      -- deststate 决定不切换
       ↓
[7] sg:GoToState(state_name)
```

**就这 7 步**——非常清晰的"决策树"。

> **新手记忆**：**ActionHandler = "Action 对象 + 一个函数（决定切到哪个 state）"**。**3 个字段**——但 90% 时候只用前 2 个。**condition 是高级 condition 玩法**——新手可以先不学。

---

### 7.3.3 快速入门：actionhandlers 表的位置与查找逻辑

#### 第一步：玩家 SG 里的 actionhandlers 表

回顾 7.3.1 我们看到的 `SGwilson.lua` 第 850 行：

```lua
local actionhandlers =
{
    ActionHandler(ACTIONS.CHOP, function(inst) ... end),
    ActionHandler(ACTIONS.MINE, function(inst) ... end),
    ActionHandler(ACTIONS.HAMMER, function(inst) ... end),
    ActionHandler(ACTIONS.TERRAFORM, "terraform"),
    ActionHandler(ACTIONS.DIG, function(inst) ... end),
    -- ... 100+ 个 ActionHandler ...
}
```

**这是一个数组（不是 dict）**——存放所有 `ActionHandler` 实例的列表。**玩家 SG 里大约有 150-200 个 ActionHandler**——对应玩家能做的所有动作。

#### 第二步：StateGraph 类如何处理这张表？

源码 `stategraph.lua` 第 274-295 行：

```274:295:scripts/stategraph.lua
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
```

**关键**：

```lua
self.actionhandlers = {}
for k,v in pairs(actionhandlers) do
    self.actionhandlers[v.action] = v
end
```

**StateGraph 把"数组"重建为"以 Action 对象为 key 的字典"**。

**为什么？**

- **传入的是数组**（`{ACTIONHandler1, ACTIONHandler2, ...}`）—— 写起来漂亮、易读
- **存储的是字典**（`{[ACTIONS.CHOP] = handler1, [ACTIONS.MINE] = handler2}`）—— 查找 O(1)

每次玩家执行 BufferedAction 时，引擎只需 `sg.actionhandlers[bufferedaction.action]` —— 直接拿到对应的 ActionHandler。**这是工程上的小细节**——但 Klei 用得很巧妙。

#### 第三步：Mod 注册的 ActionHandler 怎么进来？

注意源码第 289-294 行：

```289:294:scripts/stategraph.lua
	for k,modhandlers in pairs(ModManager:GetPostInitData("StategraphActionHandler", self.name)) do
		for i,v in ipairs(modhandlers) do
			assert( v:is_a(ActionHandler),"Non-action handler added in mod actionhandler table!")
			self.actionhandlers[v.action] = v
		end
	end
```

**`ModManager:GetPostInitData("StategraphActionHandler", "wilson")`** —— 拉取所有 Mod 通过 `AddStategraphActionHandler("wilson", handler)` 注册的 handler。

**遍历 mod handlers 数组**，把每个 handler 也写进 `self.actionhandlers` 字典。

**注意一个细节**：**Mod 的 handler 是后写入的**——会**覆盖**原版 handler（如果是同一个 action）。**这给了 Mod 强大的"修改原版动作行为"能力**——你可以用 `AddStategraphActionHandler("wilson", ActionHandler(ACTIONS.CHOP, "my_chop_state"))` 完全替换原版的砍树状态切换逻辑。

> **进阶提示**：这种"原版优先 + Mod 后写覆盖"的设计在 Klei 引擎里到处都是——非常 Mod 友好，但也很容易**两个 Mod 都覆盖同一 action 时互相冲突**——只有最后加载的那个生效。**所以多 Mod 兼容时 ActionHandler 是高发冲突区**——7.3.8 节会详谈。

#### 第四步：怎么验证一个 SG 当前注册了哪些 ActionHandler？

进游戏，控制台：

```lua
-- 看玩家 SG 注册了多少 ActionHandler
local count = 0
for _ in pairs(ThePlayer.sg.sg.actionhandlers) do count = count + 1 end
print("玩家 SG 共有 ActionHandler 数量:", count)
-- 输出大约: 150-200（具体数字看你装的 Mod）

-- 看 CHOP 对应的 ActionHandler
local handler = ThePlayer.sg.sg.actionhandlers[ACTIONS.CHOP]
print(handler)  -- 一个 ActionHandler 实例
print(handler.action == ACTIONS.CHOP)  -- true
print(type(handler.deststate))         -- "function"
```

**注意 `ThePlayer.sg.sg`**——`ThePlayer.sg` 是 `StateGraphInstance`（实例），`ThePlayer.sg.sg` 是 `StateGraph`（类定义）。前者是"我现在的状态"，后者是"我的状态机定义"。Klei 的命名有点反直觉——但记住就行。

---

### 7.3.4 进阶：State 类的完整结构

`State` 类是 SG 的核心——它定义"一个状态长什么样"。看 `stategraph.lua` 第 213-262 行：

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

**State 接收一个 `args` 表，里面有 8-10 个字段**。我们一个个讲。

#### 第一步：State 的核心字段一览

| 字段 | 类型 | 何时被调用 |
|------|------|----------|
| `name` | string | 状态唯一标识 |
| `tags` | string[] | StateTag——状态身份标记（影响 ActionHandler 的 condition） |
| `onenter` | function(inst) | 进入状态瞬间——播放动画、设置初始值 |
| `onexit` | function(inst) | 离开状态瞬间——清理 |
| `onupdate` | function(inst, dt) | 每帧调用 |
| `ontimeout` | function(inst) | 状态超时（配合 `inst.sg:SetTimeout(t)`） |
| `events` | table<string, EventHandler> | 状态内事件订阅——animover、attacked 等 |
| `timeline` | TimeEvent[] | 进入状态后特定时间点触发的回调 |

#### 第二步：实例分析—— `chop_start` 状态

回顾 SGwilson.lua 第 4858-4884 行：

```4858:4884:scripts/stategraphs/SGwilson.lua
    State{
        name = "chop_start",
        tags = { "prechop", "working" },

        onenter = function(inst)
            inst.components.locomotor:Stop()
            inst.AnimState:PlayAnimation(inst:HasTag("woodcutter") and "woodie_chop_pre" or "chop_pre")
			inst:AddTag("prechop")
        end,

        events =
        {
            EventHandler("unequip", function(inst) inst.sg:GoToState("idle") end),
            EventHandler("animover", function(inst)
                if inst.AnimState:AnimDone() then
					inst.sg.statemem.chopping = true
                    inst.sg:GoToState("chop")
                end
            end),
        },

		onexit = function(inst)
			if not inst.sg.statemem.chopping then
				inst:RemoveTag("prechop")
			end
		end,
    },
```

**逐段拆解**：

**`name = "chop_start"`** —— 状态名。

**`tags = { "prechop", "working" }`** —— 这个状态的"身份"。在这两个 tag 状态下：

- 7.3.1 节我们看到 `ActionHandler(ACTIONS.CHOP, ...)` 的逻辑里有 `if not inst.sg:HasStateTag("prechop") then ...`——这就是这个 tag 的作用：**正在 prechop 时再点击 CHOP 不会重启状态**。

**`onenter`**：

```lua
inst.components.locomotor:Stop()                      -- 停止移动
inst.AnimState:PlayAnimation("chop_pre")              -- 播放"举起斧头"动画
inst:AddTag("prechop")                                -- 给实体加 prechop tag（不只 SG，整个 inst）
```

**`events`**：状态内事件订阅。

- **`EventHandler("unequip", ...)`** —— 玩家解除装备（斧头不在手里了）→ 回 idle
- **`EventHandler("animover", ...)`** —— 动画播完 → 切到 `chop` 状态

**`onexit`**：离开 `chop_start` 时——如果**不是切到 chop**，移除 `prechop` tag。**但如果切到 chop 状态**（在 onenter 里设置了 `statemem.chopping = true`），就**保留 prechop tag**——因为 chop 状态自己会管理它。

> **进阶提示**：`statemem` 是**状态记忆**——一个临时存储区，**只在当前状态周期内有效**。它能在 onenter / events / onexit 之间共享数据。**比 inst 字段更安全**——离开状态后自动清空。

#### 第三步：实例分析—— `chop` 状态的 timeline

`chop` 状态（SGwilson.lua 第 4886+ 行）的 **timeline** 是 State 系统最强大的特性。**timeline 让你可以在动画的特定帧触发业务逻辑**。

```4886:4952:scripts/stategraphs/SGwilson.lua
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

            TimeEvent(5 * FRAMES, function(inst)
                if inst.sg.statemem.iswoodcutter then
                    inst.sg:RemoveStateTag("prechop")
					inst:RemoveTag("prechop")
                end
            end),
            ...
            ----------------------------------------------
            --Normal chop

            TimeEvent(2 * FRAMES, function(inst)
                if not inst.sg.statemem.iswoodcutter then
                    inst:PerformBufferedAction()
                end
            end),

            TimeEvent(9 * FRAMES, function(inst)
                if not inst.sg.statemem.iswoodcutter then
                    inst.sg:RemoveStateTag("prechop")
					inst:RemoveTag("prechop")
```

**核心逻辑**：

- **第 2 帧 `inst:PerformBufferedAction()`**——**调用 ACTIONS.CHOP.fn**！这才是真正"砍到树"的瞬间！
- **第 5 帧（伍迪）/ 第 9 帧（普通）—— RemoveStateTag("prechop")**——表示"现在可以接受下一次 CHOP 输入了"

**为什么是第 2 帧而不是第 0 帧**？因为**动画在前 2 帧是"举斧加速"**——视觉上还没接触到树。**业务和动画必须严格对齐**：

- 视觉上斧头第 2 帧接触树 → 同时触发"砍到"业务逻辑
- 木屑特效在第 2 帧出现 → 整体感觉"砍中了"

> **新手记忆**：**timeline 是 State 系统的灵魂**——动画的每一帧都可以挂业务回调。**`PerformBufferedAction` 永远在 timeline 里调用**，不是 onenter——这样视觉和业务才能严格同步。

#### 第四步：events 字段的常见用法

`events` 是状态内的事件订阅——在某个状态下**响应某个事件**。

**最常见的事件**：

- **`animover`** —— 动画播完
- **`unequip`** —— 装备被解除
- **`attacked`** —— 受到攻击
- **`death`** —— 死亡

**写法**：

```lua
events = {
    EventHandler("animover", function(inst)
        inst.sg:GoToState("idle")
    end),
    EventHandler("attacked", function(inst, data)
        inst.sg:GoToState("hit")
    end),
}
```

**和实体级别的 ListenForEvent 的区别**：

- `ListenForEvent("xxx", fn)` —— 永久监听，不管 SG 在哪个状态
- `events = { EventHandler("xxx", fn) }` —— **只在这个状态下生效**——离开状态自动取消

> **进阶提示**：**State 内的 events 是"状态局部事件订阅"**——非常优雅地避免了"事件监听内存泄漏"问题。**你写自定义状态时，应该把"只有这个状态需要响应的事件"放在 events 里，不要用 ListenForEvent**。

---

### 7.3.5 进阶：TimeEvent 与"动画帧 → 业务回调"的关键机制

#### 第一步：FRAMES 常量

很多代码里出现的 `2 * FRAMES`、`9 * FRAMES`、`12 * FRAMES`——`FRAMES` 是什么？

```lua
FRAMES = 1/30   -- 大约 0.0333
```

**饥荒动画帧率是 30 fps**——`FRAMES` 是一帧的秒数。**`2 * FRAMES` ≈ 0.066 秒**。

为什么不直接写秒数？因为**动画师按帧设计**——告诉程序员"砍击在第 2 帧"比"砍击在 0.066 秒"清晰得多。

#### 第二步：TimeEvent 类

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

——**就是一个时间点 + 一个函数**。简洁优雅。

`State` 类的构造函数会**按时间排序** timeline：

```lua
table.sort(self.timeline, Chronological)
```

——使 timeline 按时间升序排列，引擎依次触发。

#### 第三步：FrameEvent 是 TimeEvent 的语法糖

```193:195:scripts/stategraph.lua
function FrameEvent(frame, fn)
	return TimeEvent(frame * FRAMES, fn)
end
```

——直接传"第几帧"。**用 FrameEvent 比 `TimeEvent(N * FRAMES, ...)` 更简洁**：

```lua
-- 写法 A (常见旧风格)
TimeEvent(2 * FRAMES, function(inst) inst:PerformBufferedAction() end),

-- 写法 B (新风格，更清晰)
FrameEvent(2, function(inst) inst:PerformBufferedAction() end),
```

**但实际 SGwilson.lua 大量使用写法 A**——保持一致性、Klei 老代码不重构。新写自定义 SG 时建议用写法 B。

#### 第四步：SoundFrameEvent —— 音效专用快捷方式

```203:207:scripts/stategraph.lua
function SoundFrameEvent(frame, sound_event)
    return TimeEvent(frame * FRAMES, function(inst)
        inst.SoundEmitter:PlaySound(sound_event)
    end)
end
```

**用于"动画第 N 帧播某个音效"**：

```lua
timeline = {
    SoundFrameEvent(2, "dontstarve/wilson/chop_tree"),  -- 砍击音效
    FrameEvent(2, function(inst) inst:PerformBufferedAction() end),  -- 同帧业务
}
```

**音效和业务可以放在同一帧**——视觉、听觉、业务逻辑严格对齐。**这就是好游戏"手感"的秘密**——所有反馈精确到帧。

#### 第五步：动画驱动业务的设计哲学

**为什么饥荒采用"动画驱动业务"而不是"业务驱动动画"？**

**对比两种思路**：

**思路 A（业务驱动动画）**：

```
[1] 玩家点击 → 调 components.workable:WorkedBy() 减进度
[2] 检查进度变化 → PushEvent("worked")
[3] 树监听到 → 倒下
[4] 同时玩家播放砍击动画（独立运行）
```

**问题**：**业务和动画异步**——可能业务已经"砍完"了但动画还没播完，玩家看到"树先倒下、然后人才挥完最后一斧"。**不真实**。

**思路 B（动画驱动业务）**：

```
[1] 玩家点击 → SG 切到 chop 状态
[2] 状态播放 chop_loop 动画
[3] 动画第 2 帧（接触帧） → PerformBufferedAction → 触发业务
[4] 业务结果在第 2 帧瞬间发生（树受伤）
[5] 动画继续播完
```

**好处**：**业务紧贴动画**——你看到的接触瞬间就是业务发生的瞬间。**手感真实**。

> **设计哲学**：**饥荒所有"接触"都按这个模式**——攻击的伤害判定在挥砍动画的接触帧、采花在弯腰动画的接触帧、扔东西在投掷动画的释放帧。**这是为什么你玩饥荒"打击感"很扎实**——每一次接触都精确同步视觉/音效/业务。

> **新手记忆**：**写自定义动作的核心就是写好 timeline**——把动画帧和业务回调对齐。**写错帧位 = 玩家觉得"打没打到"** —— 这是最容易破坏游戏感的地方。

---

### 7.3.6 进阶：StateTag 的"瞬时状态身份"作用

#### 第一步：StateTag 是什么？

回顾 7.3.4 节我们看到 `chop_start` 的 `tags = { "prechop", "working" }`——这就是 StateTag。

**StateTag 是状态的"身份标签"**——和 5.5 节讲过的 EntityTag 类似但**作用域不同**：

| 维度 | EntityTag | StateTag |
|------|-----------|----------|
| 存放位置 | `inst:HasTag(...)` | `inst.sg:HasStateTag(...)` |
| 生命周期 | 实体级别（永久或手动管理） | 状态级别（进入/离开自动管理） |
| 作用 | 描述实体身份（"我是 player"、"我是 sharp"） | 描述当前活动（"我在挥斧"、"我在受击"） |

#### 第二步：StateTag 怎么用？

**进入状态时自动设置**——State 的 `tags` 字段会被 SG 自动加进 `sg.tags`：

```lua
State{
    name = "chop_start",
    tags = { "prechop", "working" },
    -- 进入这个状态时 sg.tags = {prechop=true, working=true}
}
```

**离开状态时自动清除**——下个状态的 tags 会替换。

**手动操作**：

```lua
inst.sg:AddStateTag("xxx")          -- 添加
inst.sg:RemoveStateTag("xxx")       -- 移除（看 SGwilson 用得很多）
inst.sg:HasStateTag("xxx")          -- 查询
```

#### 第三步：StateTag 在 ActionHandler 里的妙用

回顾 7.3.1 节的 `ACTIONS.CHOP` ActionHandler：

```lua
ActionHandler(ACTIONS.CHOP,
    function(inst)
        return not inst.sg:HasStateTag("prechop")
            and (inst.sg:HasStateTag("chopping") and
                "chop" or
                "chop_start")
            or nil
    end),
```

**用了 4 个 HasStateTag 检查**：

- `HasStateTag("prechop")` —— **正在举斧/接触帧前** → 拒绝再切（返回 nil）
- `HasStateTag("chopping")` —— **接触帧后**（进入了挥砍动画） → 切到 `chop`（连击）
- 都不在 → 切到 `chop_start`（新一轮）

**这种"用 StateTag 做状态判断"的模式**比"`if currentstate.name == 'chop_start'`"**更精细**——一个 StateTag 可以**横跨多个状态**：

```
chop_start (tags=prechop,working)
   ↓
chop (tags=prechop,chopping,working)
   ↓ TimeEvent 移除 prechop
chop (tags=chopping,working)
   ↓ TimeEvent 移除 chopping
chop (tags=working)
   ↓ animover → idle
```

**`prechop` tag 持续到第 5 帧（伍迪）/ 第 9 帧（普通）才被移除**——而不是离开 `chop_start` 状态就消失。这给了 ActionHandler **"在挥斧的中后段允许下一次输入"**的能力。

#### 第四步：StateTag 的常见组合

看 SGwilson.lua 里反复出现的 StateTag：

| StateTag | 含义 |
|---------|------|
| `working` | 正在使用工具（砍/挖/凿/敲） |
| `prechop` / `premine` / `predig` / `prehammer` | 工具动作的"准备阶段" |
| `chopping` / `mining` / `digging` / `hammering` | 工具动作的"动作阶段" |
| `attack` | 正在攻击 |
| `idle` | 空闲 |
| `busy` | 不能被外部打断 |
| `nointerrupt` | 完全不能打断（即使是死亡也要等完） |
| `moving` | 正在移动 |
| `casting` | 正在施法 |

**这些 StateTag 是各种 ActionHandler 的过滤条件**——比如"在 busy 状态下不响应任何 Action"——常见的"防止输入"机制。

#### 第五步：State.tags 和实体的 inst:AddTag 区别

注意 SGwilson 的 chop_start：

```lua
onenter = function(inst)
    ...
    inst:AddTag("prechop")    -- 给实体加 tag
end,
```

——同时**给实体加了 `prechop` tag**！为什么 State.tags 已经有 prechop 了，还要 inst:AddTag？

**因为它们的可见范围不同**：

- **`sg.tags`** —— 只能用 `inst.sg:HasStateTag(...)` 查询。**不会被同步到客户端、不会被网络传输**。
- **`inst.tags`** —— 用 `inst:HasTag(...)` 查询。**会同步到客户端**——其他玩家、客户端 UI 都能看到。

**为什么需要两者并存**？

- **内部逻辑用 sg.tags** —— 状态机自己用，性能好、生命周期自动
- **跨实体/客户端用 inst.tags** —— 让其他玩家知道"那个角色正在 prechop"，可以做协同动画

> **新手记忆**：**StateTag 是 SG 内部的"状态标签"** —— 状态间共享、不跨网络。**ActionHandler 的 condition / deststate 函数里通常用 sg:HasStateTag 而不是 inst:HasTag**——精度更高、不会误判远程玩家的状态。

---

### 7.3.7 老手进阶：`AddStategraphActionHandler` 完整流程 + 三类典型注册模式

#### 第一步：modutil.lua 接口

```518:524:scripts/modutil.lua
	env.AddStategraphActionHandler = function(stategraph, handler)
		initprint("AddStategraphActionHandler", stategraph)
		if not env.postinitdata.StategraphActionHandler[stategraph] then
			env.postinitdata.StategraphActionHandler[stategraph] = {}
		end
		table.insert(env.postinitdata.StategraphActionHandler[stategraph], handler)
	end
```

**只是把 handler 存到 `env.postinitdata.StategraphActionHandler[stategraph]`**——按 SG 名分组的数组。

**注意**：这一步**不会立刻**修改任何 SG——只是登记到 `postinitdata` 等待。**真正的注册发生在 SG 实例化时**——回顾 7.3.3 第三步的 StateGraph 构造函数：

```lua
for k,modhandlers in pairs(ModManager:GetPostInitData("StategraphActionHandler", self.name)) do
    for i,v in ipairs(modhandlers) do
        self.actionhandlers[v.action] = v
    end
end
```

——**SG 实例化时拉取所有相关 mod 的 handler，逐个写入 actionhandlers 字典**。

**整个流程是"延迟注册 + 实例化时统一应用"**——一种很优雅的延迟绑定。

#### 第二步：完整的"自定义动作三步走"实战

回顾 7.2.7 第六步我们承诺过的"完整三步"——现在我们补上第三步：

```lua
-- 在 modmain.lua 里：

-- 1. AddAction：声明动词
AddAction("PSIONIC_BURST", "释放灵能", function(act)
    if act.target ~= nil and act.target.components.psionic ~= nil then
        return act.target.components.psionic:Burst()
    end
    return false
end)

-- 2. AddComponentAction：声明哪些场景出现
AddComponentAction("SCENE", "psionic", function(inst, doer, actions, right)
    if right and inst:HasTag("psionic_full") then
        table.insert(actions, ACTIONS.PSIONIC_BURST)
    end
end)

-- 3. AddStategraphActionHandler：声明动画响应
AddStategraphActionHandler("wilson",
    ActionHandler(ACTIONS.PSIONIC_BURST, "give"))
```

**第 3 步用了"复用现有状态"的简化模式**——把"释放灵能"动画切到现有的 `give` 状态（玩家递东西的动作）。**给玩家手要伸出去碰到晶簇这种"轻接触"动作**——`give` 是合理的近似。

进游戏验证：

```
1. c_give("psionic_crystal")，扔在地上
2. 等到充能满
3. 鼠标右键 → 玩家走过去 → 播放 give 动画 → Burst 触发
4. 控制台：c_findnext("player") 看 sg.currentstate.name 在 burst 时变成 "give"
```

**完整三步走完成！**

#### 第三步：三类典型注册模式

**模式 A：复用现有状态（最简单）**

```lua
AddStategraphActionHandler("wilson",
    ActionHandler(ACTIONS.PSIONIC_BURST, "give"))
```

**适用**：你的动作的视觉效果**类似已有动作**——直接复用。

**典型例子**：
- 释放灵能 / 喂宠物 / 给 NPC → 复用 `give`
- 检查物品 / 阅读说明 → 复用 `read`
- 播种 / 部署 → 复用 `doshortaction`

**模式 B：根据上下文动态切换状态（中等）**

```lua
AddStategraphActionHandler("wilson",
    ActionHandler(ACTIONS.PSIONIC_BURST, function(inst)
        if inst:HasTag("woodcutter") then
            return "woodie_burst"
        elseif inst:HasTag("ghost") then
            return nil   -- 鬼魂不能释放
        else
            return "give"
        end
    end))
```

**适用**：不同角色/形态需要不同动画——和 7.3.1 节 CHOP 的 ActionHandler 同款思路。

**模式 C：注册一个全新状态（高级）**

```lua
-- 1. AddStategraphState 注册新状态
AddStategraphState("wilson", State{
    name = "psionic_burst_cast",
    tags = { "casting", "busy", "doing" },

    onenter = function(inst)
        inst.components.locomotor:Stop()
        inst.AnimState:PlayAnimation("idle_loop")
        inst.AnimState:PushAnimation("dial_loop", true)  -- 播放法术动画
    end,

    timeline = {
        FrameEvent(15, function(inst)
            inst:PerformBufferedAction()  -- 第 15 帧触发 PSIONIC_BURST.fn
            inst.SoundEmitter:PlaySound("dontstarve/common/...")
        end),
        FrameEvent(30, function(inst)
            inst.sg:GoToState("idle")
        end),
    },

    events = {
        EventHandler("animover", function(inst) inst.sg:GoToState("idle") end),
    },
})

-- 2. AddStategraphActionHandler 注册到这个状态
AddStategraphActionHandler("wilson",
    ActionHandler(ACTIONS.PSIONIC_BURST, "psionic_burst_cast"))
```

**适用**：**完全自定义的动画/音效/特效**——做"专属于你这个动作的视觉表现"。

**实际开发流程**：

1. **第一版**：先用模式 A 快速跑通——`ActionHandler(ACTIONS.PSIONIC_BURST, "give")`，动画是给东西的样子，但功能 OK
2. **第二版**：发现"给"动作不太合适——做一个新的 state，配套自定义动画和音效（模式 C）
3. **第三版**：发现不同角色应该有不同表现——加 condition 函数（模式 B 升级）

> **新手记忆**：**先用模式 A 跑通**，再升级到 C / B。**别一上来就写自定义状态**——你会在动画文件、timeline 时机、StateTag 这堆东西里迷失方向。

---

### 7.3.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：动画文件不存在

```lua
-- ❌ 错：PlayAnimation 一个不存在的动画
State{
    name = "psionic_burst_cast",
    onenter = function(inst)
        inst.AnimState:PlayAnimation("psionic_burst_animation")  -- 这个动画不存在！
    end,
}
```

**症状**：动作触发后玩家**完全不动**或者**动画错位**——卡在某个静止 pose。

**原因**：`PlayAnimation` 找不到动画文件时**不会报错**，但 AnimState 没有"当前动画"——**动画/动画事件全部失效**——`animover` 事件永远不触发——状态卡住。

**调试方法**：

```lua
-- 在 onenter 末尾加
print("Playing animation:", inst.AnimState:GetCurrentAnimation())
-- 如果输出空字符串或者"none"，就是动画文件没找到
```

**解决**：

- 用现有动画（最简单）——参考别的状态的 PlayAnimation 调用
- 或确保你的动画包（`.zip`）已经在 `Assets` 列表里、并且 `SetBuild`/`SetBank` 设置正确

#### 陷阱 2：StateTag 漏了 RemoveStateTag

```lua
-- ❌ 错：进入状态加了 tag、但出状态没移除
State{
    name = "casting",
    tags = { "busy" },

    onenter = function(inst)
        inst:AddTag("invulnerable")   -- 给实体加无敌 tag
    end,

    onexit = function(inst)
        -- 忘了 inst:RemoveTag("invulnerable")
    end,
}
```

**症状**：动作执行完后，**玩家依然带着 invulnerable tag**——一直无敌。

**原因**：**inst.tags 是实体级，不会因为 SG 状态切换自动清除**。**只有 sg.tags（来自 State.tags 字段）才自动管理**。

**这是 6.1.6 节"OnRemoveFromEntity 必须 RemoveTag"原则在 SG 里的对应**——**自己加的 inst tag 必须自己 RemoveTag**。

**正解**：

```lua
onexit = function(inst)
    inst:RemoveTag("invulnerable")
end,
```

#### 陷阱 3：timeline 在 onenter 之前触发

```lua
State{
    name = "myaction",
    onenter = function(inst)
        inst.sg.statemem.target = SomeFunction()  -- 设置 target
    end,

    timeline = {
        FrameEvent(0, function(inst)
            print(inst.sg.statemem.target)  -- 可能是 nil！
        end),
    },
}
```

**问题**：`FrameEvent(0, ...)` 是**当前状态进入后第 0 帧**触发——理论上 onenter 在前——但实际有时序竞态：

- `onenter` 先执行（同步）
- 第一次 update 时 timelineindex 从头扫——`time >= 0` 的 timeline 立刻触发
- **但如果 onenter 里有异步操作或 PostInit 钩子**——可能 statemem 还没设置完

**经验**：**重要的初始化放在 onenter，timeline 里第 0-1 帧的回调要做防御**：

```lua
FrameEvent(0, function(inst)
    if inst.sg.statemem.target == nil then return end
    print(inst.sg.statemem.target)
end),
```

或者**把所有初始化放 onenter**，timeline 从 1 帧开始：

```lua
timeline = {
    FrameEvent(1, ...),  -- 不要 FrameEvent(0, ...)
},
```

#### 陷阱 4：忘记 PerformBufferedAction

```lua
-- ❌ 错：自定义状态里忘了调 PerformBufferedAction
State{
    name = "psionic_burst_cast",
    onenter = function(inst)
        inst.AnimState:PlayAnimation("burst_anim")
    end,
    timeline = {
        FrameEvent(15, function(inst)
            -- 忘了 PerformBufferedAction！
            inst.SoundEmitter:PlaySound("...")
        end),
    },
    events = {
        EventHandler("animover", function(inst) inst.sg:GoToState("idle") end),
    },
}
```

**症状**：动作触发了、动画播了、音效响了——**但 ACTIONS.PSIONIC_BURST.fn 永远没被调用**——灵能没有被释放。

**原因**：**动画驱动业务的核心就是 `PerformBufferedAction`**。这是 `EntityScript` 的方法（看 `entityscript.lua`）——**找到当前 BufferedAction 并调用它的 action.fn**。**没有这一行——业务永远不发生**。

**正解**：

```lua
timeline = {
    FrameEvent(15, function(inst)
        inst:PerformBufferedAction()  -- ★ 关键
        inst.SoundEmitter:PlaySound("...")
    end),
},
```

#### 陷阱 5：PerformBufferedAction 在错误时机调用

```lua
-- ❌ 错：在 onenter 里立刻 PerformBufferedAction
State{
    name = "psionic_burst_cast",
    onenter = function(inst)
        inst.AnimState:PlayAnimation("burst_anim")
        inst:PerformBufferedAction()  -- ❌ 太早了！
    end,
}
```

**问题**：**业务和动画完全不同步**——玩家看到"灵能爆发了，然后才挥手"。

**正解**：永远在 timeline 的"接触帧"调 PerformBufferedAction——和动画的关键帧对齐。

#### 陷阱 6：两个 Mod 都注册同一 ActionHandler

```lua
-- Mod A:
AddStategraphActionHandler("wilson",
    ActionHandler(ACTIONS.CHOP, "my_chop_state_A"))

-- Mod B:
AddStategraphActionHandler("wilson",
    ActionHandler(ACTIONS.CHOP, "my_chop_state_B"))
```

**结果**：**最后加载的那个生效**，前一个被覆盖。**玩家可能看到"两个 Mod 都说改了砍树状态，但实际只有一个生效"**——很难调试。

**根本原因**：回顾 7.3.3 第三步：

```lua
self.actionhandlers[v.action] = v
```

——**字典写入是覆盖**。最后写的胜出。

**解决方案**：

- **多 Mod 兼容时不要直接覆盖原版核心动作**（CHOP / MINE / ATTACK 等）——用 `AddStategraphPostInit` 在 SG 实例化后再做"叠加修改"
- **或者用条件分支**：

```lua
-- 检查原版 handler 还在不在
AddStategraphPostInit("wilson", function(sg)
    local original = sg.actionhandlers[ACTIONS.CHOP]
    sg.actionhandlers[ACTIONS.CHOP] = ActionHandler(ACTIONS.CHOP,
        function(inst, action)
            -- 优先用我的逻辑
            if inst:HasTag("my_special_woodcutter") then
                return "my_chop_state"
            end
            -- 否则交回原版
            return original.deststate(inst, action)
        end)
end)
```

**这种"包装原版而非覆盖"的模式是 Mod 兼容的灵魂**——和 6.1.8 节的"AddComponentPostInit 保留原版方法"思路完全一致。

#### 设计经验三条

**经验 1：自定义状态尽量"短而纯净"**

一个状态的 onenter / events / timeline 里**做太多事**——容易和别的 Mod 冲突、容易出 timing bug。**每个状态只做一件事**：

- `chop_start` —— 只播放"举斧子"的动画
- `chop` —— 只播放"挥砍"的动画 + timeline 触发业务
- 不要把这两件事塞同一个状态

**经验 2：状态间共享数据用 `sg.statemem` 而不是 `inst` 字段**

```lua
-- ❌ 不好：
inst._mychopdata = { ... }  -- 写实体字段，可能跨状态污染

-- ✅ 好：
inst.sg.statemem.mychopdata = { ... }  -- 状态记忆，离开状态自动清空
```

**经验 3：写完 ActionHandler 一定要测试"中途打断"的情况**

每个自定义动作都要测：

- 动作刚开始就被攻击 → 切到 hit 状态？业务还触发吗？
- 动作进行中玩家死了 → 切到 death 状态？BufferedAction 怎么处理？
- 动作进行中目标被销毁了 → fn 里有没有 nil check？
- 玩家蛇形走位、目标在脚下消失 → ActionHandler 还能正常切换吗？

**这些"边角 case"是饥荒最常见的崩溃来源**。**写完动作后用控制台主动制造这些 case 测试**——比线上玩家撞 bug 强一万倍。

---

### 7.3.9 小结

- **StateGraph 是饥荒的"角色行为状态机"**——每个实体可以挂一份 SG（玩家/猪人/蜘蛛各有自己的），管理"当前在做什么"。`SG → State → 动画/事件/timeline`。
- **ActionHandler 是 ACTIONS 系统和 SG 系统之间的桥**——3 个字段（`action / deststate / condition`）。**收到动作时决定切到哪个状态**。
- **`actionhandlers` 数组**在每个 SG 文件里——StateGraph 类构造函数把它**重建为以 Action 为 key 的字典**实现 O(1) 查找。**Mod 注册的 handler 会覆盖原版**——这是兼容性的关键点。
- **State 类**：`name / tags / onenter / onexit / onupdate / ontimeout / events / timeline` —— **8-10 个核心字段**。`tags` 是 sg 局部 tag、`events` 是状态局部事件订阅、`timeline` 是动画帧驱动业务的核心。
- **TimeEvent / FrameEvent / SoundFrameEvent**——动画帧到秒的转换 (`FRAMES = 1/30`)、按时间排序、依次触发。**好游戏的"打击感"就来自 timeline 和动画严格对齐**。
- **StateTag** 是 SG 内部的状态身份标记（`prechop / chopping / working / busy`）——和 EntityTag 不同，**自动随状态切换管理、不跨网络**。**ActionHandler 的 condition 大量用 HasStateTag** 做精细判断。
- **`AddStategraphActionHandler`** 是 Mod 注册的标准接口——`postinitdata.StategraphActionHandler[sg_name]` 暂存、SG 实例化时由 `ModManager:GetPostInitData` 拉取应用。**三类典型注册模式**：① 复用现有状态、② 动态决策 deststate、③ 注册全新自定义状态。
- **6 个常见陷阱**：① 动画文件不存在 ② StateTag/inst tag 漏 RemoveTag ③ timeline 第 0 帧时序 ④ 忘记 PerformBufferedAction ⑤ PerformBufferedAction 时机错误 ⑥ 多 Mod ActionHandler 覆盖冲突。
- **三条设计经验**：① 状态短而纯净、② 用 sg.statemem 而不是 inst 字段、③ 必须测试"中途打断"边界 case。

> **下一节预告**：7.4 节我们打开 **`bufferedaction.lua`** —— 把 7.1-7.3 这三节"动作系统"的运行时载体讲透：BufferedAction 是个怎样的对象？它从创建（PlayerActionPicker / 控制台 c_doaction）到执行（PerformBufferedAction）再到销毁（成功 / 失败 / 取消），**完整生命周期**经过哪些方法、哪些事件？为什么饥荒里"行动队列"只有一个 buffered（不像 RTS 那样能排队多个）？我们会逐行读 `BufferedAction:Do()` 的源码——把这 200 行的核心串起来。

---

## 7.4 BufferedAction 的完整生命周期


### 本节导读

7.1 节我们看了**ACTIONS 表**（动词字典），7.2 看了 **ComponentAction**（哪些动作可见），7.3 看了 **ActionHandler**（动作触发哪个 State）——但所有这些代码都在反复处理一个对象：**`BufferedAction`**。它是连接整个 Action 系统的"**运行时载体**"——

> 每次玩家点击屏幕、按下按键、控制台执行 `c_doaction(...)`，引擎都会**创建一个 `BufferedAction` 实例**，把"执行者"+"目标"+"动作"+"工具"+"位置"全部打包进去。然后这个实例被推到玩家身上、被 SG 状态机引导、在某一帧被执行、最后成功或失败、调用回调、销毁。**它的生命周期就是 Action 系统的运行时主线**。

这一节我们打开 **`scripts/bufferedaction.lua`**——这是个**只有 110 行**的核心文件，但它的每一行都涉及到 Action 系统的关键决策。结合 `entityscript.lua` 中 5 个相关方法（`PushBufferedAction` / `PerformBufferedAction` / `ClearBufferedAction` / `PreviewBufferedAction` / `GetBufferedAction`），把 BufferedAction **从创建到销毁**的完整生命周期讲透：

> **新手**从 7.4.1-7.4.3 起步——理解 BufferedAction 是个什么对象、它的 14 个字段各是什么、生命周期"六阶段"总览；**进阶读者**继续看 7.4.4-7.4.6，深入 `PushBufferedAction` 的 5 步流程、`Do` 方法的执行逻辑、`IsValid` 的 6 项检查、`Succeed/Fail` 的回调链；**老手**跳到 7.4.7-7.4.8，看 `AddSuccessAction / AddFailAction` 的链式回调实战、6 个最容易踩的坑——尤其是"动作中途目标被偷"和"PushBufferedAction 与 PerformBufferedAction 时序错乱"这种隐蔽 bug。

---

### 7.4.1 快速入门：BufferedAction 是什么 —— 一个"动作快递包"

#### 第一步：用快递做类比

想象你在网上下单买一件商品——

- **下单时**：你创建一张订单——里面写明"**谁买的（doer）/ 卖给谁的（target，即接收实体）/ 买什么（action）/ 用什么支付（invobject）/ 送到哪个地址（pos）**"——这张订单就是 BufferedAction 的精确比喻。
- **运输中**：订单进入仓库（PushBufferedAction）→ 配送员开始打包（SG 切换到执行状态）→ 在某个时刻送达（PerformBufferedAction）。
- **签收**：成功 → 订单完成（Succeed）；失败 → 退回（Fail）。

**BufferedAction 就是 Action 系统的"订单"** —— 不是动作本身、也不是动作的执行结果、而是"**这次具体动作请求的所有必要信息打包在一起的载体**"。

#### 第二步：在游戏里看一个真实的 BufferedAction

进游戏，**手持斧头点击松树**——在玩家走过去的过程中（不是动画结束后）控制台立刻输入：

```lua
print(ThePlayer.bufferedaction)
```

输出类似：

```
砍 tree[123456] With Inv: axe[789012]
```

——**这就是当前玩家身上挂的 BufferedAction**。它包含：

- `doer = ThePlayer`（执行者）
- `target = tree`（被砍的目标）
- `action = ACTIONS.CHOP`
- `invobject = axe`（手里的斧头）
- `pos = nil`（不需要点击地面位置）
- `recipe = nil`（不是建造）
- `forced = false`（不是远程指定）
- `onsuccess / onfail = {}`（暂时没有回调）

继续在控制台：

```lua
local ba = ThePlayer.bufferedaction
print(ba.doer)      -- ThePlayer
print(ba.target)    -- 那棵树
print(ba.action)    -- ACTIONS.CHOP
print(ba.invobject) -- 你的斧头
print(ba:IsValid()) -- true
```

——**所有"这次砍树请求"的上下文都被这一个 ba 对象承载**。

#### 第三步：拆解为什么需要 BufferedAction 这一层抽象

你可能会问：**为什么不直接调 `ACTIONS.CHOP.fn(player, tree, axe)` 就行了？为什么要包装成一个对象？**

答案是 **3 个核心原因**：

**(1) 解耦时间**

玩家**点击树**和**实际触发砍树**之间**有时间间隔**——玩家要走过去（约 1-3 秒）。**BufferedAction 是这段时间里"动作请求"的载体**——没有它，引擎要么用全局变量（线程不安全）、要么每帧重新检查（性能差）。

**(2) 解耦决策与执行**

**ActionHandler 决定 SG 状态切换**、**State 决定动画播放**、**timeline 决定业务执行帧**——这三件事**互相不知道对方的存在**。但它们都需要"这次动作的上下文"——**BufferedAction 是它们共享的载体**。

**(3) 失效再校验**

玩家走向树的 1-3 秒内，**世界可能变了**——树被别的玩家砍了、玩家的斧头被偷了、玩家自己被怪物击杀。**BufferedAction 的 IsValid() 方法**在执行前**再次检查所有前提条件**——避免 7.1.6 节讲过的"摘到 nil 目标"灾难。

> **新手记忆**：**BufferedAction 是 Action 系统的"运行时载体"** —— 像快递订单一样，把一次动作请求的所有信息打包，让它能跨时间、跨模块、跨网络流转。**90% 的 Action 系统逻辑都围绕它展开**。

---

### 7.4.2 快速入门：BufferedAction 类的 14 个字段

#### 第一步：看构造函数

```2:20:scripts/bufferedaction.lua

BufferedAction = Class(function(self, doer, target, action, invobject, pos, recipe, distance, forced, rotation, arrivedist)
    self.doer = doer
    self.target = target
    self.initialtargetowner = target ~= nil and target.components.inventoryitem ~= nil and target.components.inventoryitem.owner or nil
    self.action = action
    self.invobject = invobject
    self.doerownsobject = doer ~= nil and invobject ~= nil and invobject.replica.inventoryitem ~= nil and invobject.replica.inventoryitem:IsHeldBy(doer)
    self.pos = pos ~= nil and DynamicPosition(pos) or nil
    self.rotation = rotation or 0
    self.onsuccess = {}
    self.onfail = {}
    self.recipe = recipe
    self.options = {}
    self.distance = distance or action.distance
    self.arrivedist = arrivedist or action.arrivedist
    self.forced = forced
    self.autoequipped = nil --true if invobject should've been auto-equipped
    self.skin = nil
end)
```

**14 个字段**（外加 1 个隐式 `reason` 字段，Do() 后赋值）。

#### 第二步：14 个字段分类讲解

按"重要性 + 使用频率"分组：

##### **A 组：核心三要素（永远会用到）**

**`self.doer`**：执行者实体。**几乎总是玩家**——但有些自动化机制（如 `chester` 自动捡东西）的 doer 可能是 NPC 或者宠物。`fn` 里访问 doer 的组件做事用。

**`self.target`**：被作用的目标实体。可以是 `nil`（如 `WALKTO` 不需要目标）。**`fn` 里 `act.target` 是最常被访问的字段**。

**`self.action`**：`ACTIONS.XXX` 对象本身——回顾 7.1 我们讲过的 Action 类。**用来获取动作的 fn / priority / distance 等元信息**。

##### **B 组：工具与位置（中频使用）**

**`self.invobject`**：玩家手里的物品。**对工具类动作**很关键——`CHOP.fn` 里要用 `act.invobject` 检查"用的是不是斧头"。

**`self.pos`**：位置——但**不是 Vector3 而是 `DynamicPosition` 对象**。后者是一种"会跟随浮板移动"的智能位置（看 7.4.7 节末尾）。要取实际坐标用 `act:GetActionPoint()` 返回 Vector3。

**`self.recipe`**：配方名——**仅 `BUILD` 动作用**。其他动作通常 nil。

##### **C 组：所有权防御（重要但隐式）**

**`self.initialtargetowner`**：构造时的"target 所有者"快照。

```lua
self.initialtargetowner = target ~= nil and target.components.inventoryitem ~= nil and target.components.inventoryitem.owner or nil
```

——**如果 target 是个物品（有 inventoryitem 组件）**，记录它**当时的拥有者**。`IsValid` 里会复查这个值——**如果 target 中途被人捡走了、所有者变了——动作失效**。

**这是防御"动作中途目标易主"的关键设计**——非常隐蔽但非常重要。

**`self.doerownsobject`**：构造时的"invobject 是不是 doer 持有的"快照。

```lua
self.doerownsobject = doer ~= nil and invobject ~= nil and invobject.replica.inventoryitem ~= nil and invobject.replica.inventoryitem:IsHeldBy(doer)
```

——**判断手里物品最初是不是这个 doer 的**。如果中途被偷走、丢出去——`IsValid` 也会发现。

##### **D 组：可选参数（按需使用）**

**`self.distance`**：自定义距离——可以**覆盖 `action.distance`**。某些动作的距离需要按上下文动态决定（如不同武器射程）。

**`self.arrivedist`**：到达距离——动作触发的"足够近"阈值。

**`self.forced`**：是否强制（远程指定）——按住 Shift + 点击触发，bool。回顾 7.1.6 节 `canforce` 的设计。

**`self.rotation`**：朝向（角度）——某些动作需要预设朝向（如部署旋转方向）。

##### **E 组：回调与选项（高级用）**

**`self.onsuccess`**：成功回调列表（数组）。**可通过 `:AddSuccessAction(fn)` 注册**。

**`self.onfail`**：失败回调列表（数组）。**可通过 `:AddFailAction(fn)` 注册**。

**`self.options`**：额外选项的 table——常见 keys：
- `options.instant = true` —— 立即执行（不走 SG，跳过动画）
- `options.no_predict_fastforward = true` —— 客户端不要预测/快进

##### **F 组：动态字段（运行时设置）**

**`self.autoequipped`**：是否被自动装备（玩家点击场景中物品时，引擎可能自动从背包拿出工具——这个标记记录"这件工具是被自动选出来的"）。

**`self.skin`**：皮肤配置（构造时 nil，特殊路径会设置）。

**`self.reason`**：失败原因（**Do() 调用后才有**）—— 是 `action.fn` 返回的第二个返回值。用于显示"为什么这次动作没成功"——回顾 7.1.6 节 `fn` 的返回约定。

**`self.ispreviewing`**：客户端预测专用——回顾 `entityscript.lua` 第 1605 行的 `PerformPreviewBufferedAction` 设置。

#### 第三步：14 个字段速查表

| 字段 | 类型 | 重要性 | 含义 |
|------|------|--------|------|
| `doer` | Entity | ⭐⭐⭐ | 执行者（玩家/NPC） |
| `target` | Entity \| nil | ⭐⭐⭐ | 目标实体 |
| `action` | Action 对象 | ⭐⭐⭐ | ACTIONS.XXX |
| `invobject` | Entity \| nil | ⭐⭐ | 手里物品 |
| `pos` | DynamicPosition \| nil | ⭐⭐ | 位置（点击地面才有） |
| `recipe` | string \| nil | ⭐ | 配方名（BUILD 动作） |
| `initialtargetowner` | Entity \| nil | ⭐ | 构造时 target 的拥有者 |
| `doerownsobject` | bool | ⭐ | invobject 是否 doer 持有 |
| `distance` | number \| nil | ⭐ | 自定义距离 |
| `arrivedist` | number \| nil | ⭐ | 到达距离 |
| `forced` | bool | ⭐ | 是否远程强制 |
| `rotation` | number | ⭐ | 朝向 |
| `onsuccess` | function[] | ⭐⭐ | 成功回调列表 |
| `onfail` | function[] | ⭐⭐ | 失败回调列表 |
| `options` | table | ⭐ | 额外选项 |

> **新手记忆**：**核心字段就 4 个——`doer / target / action / invobject`**。其他都是高级或隐式字段。**写自定义动作的 fn 时**，90% 的代码只访问这 4 个字段。

---

### 7.4.3 快速入门：从创建到销毁 —— 完整生命周期总览

#### 第一步：六阶段全景图

把 BufferedAction 从"出生到死亡"的完整链路画出来：

```
┌─────────────────────────────────────────────────────────────┐
│  阶段 1: 创建                                                  │
├─────────────────────────────────────────────────────────────┤
│  来源 A: PlayerActionPicker（鼠标点击）                       │
│         └── BufferedAction(doer, target, action, ...)         │
│  来源 B: 控制台 c_doaction("CHOP", target)                   │
│  来源 C: AI 自动 → brain 生成 BufferedAction                  │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 2: 推送                                                  │
├─────────────────────────────────────────────────────────────┤
│  inst:PushBufferedAction(ba)                                  │
│    ├── 重复检查                                                │
│    ├── 失效旧 ba                                               │
│    ├── ba:TestForStart() 校验                                  │
│    ├── 三个分支:                                               │
│    │     ├── WALKTO       → 立刻 Succeed                      │
│    │     ├── instant=true → 立刻 Do                           │
│    │     └── 一般情况     → SG:StartAction(ba)                │
│    └── inst.bufferedaction = ba (保留引用)                    │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 3: SG 状态切换 + 走路                                     │
├─────────────────────────────────────────────────────────────┤
│  ActionHandler.deststate(inst, ba) → "chop_start"             │
│        ↓                                                      │
│  SG:GoToState("chop_start") → 玩家开始走 → 到达 → 切到 chop  │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 4: 业务执行                                              │
├─────────────────────────────────────────────────────────────┤
│  State.timeline[第N帧] → inst:PerformBufferedAction()         │
│        ↓                                                      │
│  ba:Do() ← 真正"做"动作的入口                                  │
│    ├── ba:IsValid() 再次校验（6 项检查）                       │
│    ├── action.fn(ba) ← 执行业务（CHOP.fn / EAT.fn / 等）      │
│    └── 根据返回值:                                             │
│          ├── true   → ba:Succeed()                            │
│          └── false  → ba:Fail()                               │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 5: 回调触发                                              │
├─────────────────────────────────────────────────────────────┤
│  ba:Succeed() / ba:Fail()                                     │
│    ├── 遍历 onsuccess / onfail 列表                           │
│    ├── 调用每个回调                                           │
│    └── 清空两个列表                                           │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 6: 销毁                                                  │
├─────────────────────────────────────────────────────────────┤
│  inst.bufferedaction = nil (PerformBufferedAction 末尾)       │
│  Lua GC 回收                                                  │
└─────────────────────────────────────────────────────────────┘
```

**注意**：阶段 3 和 4 之间的"走路 → 到达"耗时不定（0 秒到几秒）——而 BufferedAction **在整段时间里都"挂在" `inst.bufferedaction` 上**。这就是 7.4.1 我们说的"快递在配送中"。

#### 第二步：异常路径——动作失败/中断

```
[阶段 X] 玩家走到一半被打断
       ├── inst:ClearBufferedAction()
       │     ├── ba:Fail() (触发 onfail 回调)
       │     └── inst.bufferedaction = nil
       └── 阶段直接结束，跳过阶段 4-5

[阶段 4] 执行时 IsValid 失败
       ├── ba:Do() 内部 return false
       ├── 不调 fn (因为前置失败)
       └── 不触发 Succeed/Fail（因为没进 fn 流程）
```

**注意**：**`IsValid` 失败的处理稍微反直觉**——`ba:Do()` 第一行检查 `if not self:IsValid() then return false end`——**但没有调 `Fail()`**！所以 `onfail` 不会触发。**这是一个小细节，老手都得提防的"静默失败"路径**。

#### 第三步：网络同步的角色

回顾 6.x 章节的多人游戏哲学——BufferedAction **只在服务端真正存在**。客户端有"预测版本"：

```
客户端：
   PreviewBufferedAction → 临时挂在 inst.bufferedaction
   ├── PerformPreviewBufferedAction
   │     └── 通过 RPC 把"我想做这个动作"发给服务端
   └── 等待服务端确认

服务端：
   收到 RPC → PushBufferedAction → 走完整流程
   状态同步回客户端 → 客户端按状态播动画
```

**这种"客户端预测 + 服务端确认"的设计让网络延迟下玩家也能立刻看到反馈**——但代价是**复杂度**。`bufferedaction.lua` 第 9 行的 `DynamicPosition(pos)` 包装就是为了让位置在浮板上能被网络一致地表示。

> **新手记忆**：**BufferedAction 的生命周期 = 6 阶段** —— 创建 → 推送 → SG 切换 → 业务执行 → 回调 → 销毁。**记住"它在阶段 3 时停在 inst.bufferedaction 上"** —— 这是为什么 SG 的 timeline 能通过 `inst:GetBufferedAction()` 拿到它的关键。

---

### 7.4.4 进阶：`PushBufferedAction` —— 起点

`PushBufferedAction` 是阶段 2 的核心方法。看它的完整实现（`entityscript.lua` 第 1609-1652 行）：

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

    --walkto is kind of a nil action - the locomotor will have put us at the destination by now if we get to here
    if bufferedaction.action == ACTIONS.WALKTO then
        self:PushEvent("performaction", { action = bufferedaction })
        bufferedaction:Succeed()
        self.bufferedaction = nil
	elseif bufferedaction.action.instant or bufferedaction.options.instant then
        if bufferedaction.target ~= nil and bufferedaction.target.Transform ~= nil and (self.sg == nil or self.sg:HasStateTag("canrotate")) then
            self:FacePoint(bufferedaction.target.Transform:GetWorldPosition())
        end
        self:PushEvent("performaction", { action = bufferedaction })
        bufferedaction:Do()
        self.bufferedaction = nil
    else
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

#### 第一步：5 步流程拆解

**步骤 1：重复检查**

```lua
if bufferedaction ~= nil and
    self.bufferedaction ~= nil and
    bufferedaction.target == self.bufferedaction.target and
    bufferedaction.action == self.bufferedaction.action and
    bufferedaction.invobject == self.bufferedaction.invobject and
    not (self.sg ~= nil and self.sg:HasStateTag("idle")) then
    return
end
```

**意思**：如果当前已经有同一个动作（target/action/invobject 完全一致），并且**SG 不在 idle 状态**——直接 return（不重复推送）。

**为什么需要这个检查**？因为玩家**频繁点击同一棵树**——每次点击都会触发 PlayerActionPicker 创建新 BufferedAction、调 PushBufferedAction。如果不去重——每次都会取消上一个、创建新的——结果是**砍树动画反复重启**。这个检查让"同动作连点"被忽略——动作正常完成。

**`HasStateTag("idle")` 的特殊处理**：如果**当前 SG 在 idle**——说明动作已经做完一次了——这次新点击应该重新触发——所以**不视为重复**。

**步骤 2：失效旧动作**

```lua
if self.bufferedaction ~= nil then
    self.bufferedaction:Fail()
    self.bufferedaction = nil
end
```

**新动作要进来——把旧的 fail 掉**。这就是为什么"砍着树突然换攻击怪物"——砍树动作的 onfail 会被触发——你能在 onfail 里做"清理工作"（如停止砍树音效）。

**步骤 3：TestForStart 校验**

```lua
local success, reason = bufferedaction:TestForStart()
if not success then
    self:PushEvent("actionfailed", { action = bufferedaction, reason = reason })
    return
end
```

**`TestForStart` 是 `IsValid` 的别名**（`bufferedaction.lua` 第 50 行）—— 6 项检查（7.4.5 节会讲）。

**失败时**触发 `actionfailed` 事件——可以让 UI 提示玩家"为什么动作没启动"。

**步骤 4：三个分支处理**

**分支 A：WALKTO 特殊处理**

```lua
if bufferedaction.action == ACTIONS.WALKTO then
    self:PushEvent("performaction", { action = bufferedaction })
    bufferedaction:Succeed()
    self.bufferedaction = nil
```

`WALKTO` 是"走过去"动作——**走到这一步说明 locomotor 已经走完了**——直接 Succeed。

**分支 B：instant 动作立即执行**

```lua
elseif bufferedaction.action.instant or bufferedaction.options.instant then
    if bufferedaction.target ~= nil and bufferedaction.target.Transform ~= nil and (self.sg == nil or self.sg:HasStateTag("canrotate")) then
        self:FacePoint(bufferedaction.target.Transform:GetWorldPosition())
    end
    self:PushEvent("performaction", { action = bufferedaction })
    bufferedaction:Do()
    self.bufferedaction = nil
```

**`action.instant=true` 或 `options.instant=true` 的动作不走 SG 状态切换**——直接 `FacePoint + performaction event + Do() + nil`。**典型的 instant 动作**：`LOOKAT`、`EQUIP`、`UNEQUIP`、`TALKTO`——回顾 7.1.3 节。

**分支 C：一般动作走 SG**

```lua
else
    self.bufferedaction = bufferedaction
    if self.sg == nil then
        self:PushEvent("startaction", { action = bufferedaction })
    elseif not self.sg:StartAction(bufferedaction) then
        self:PushEvent("performaction", { action = bufferedaction })
        self.bufferedaction:Fail()
        self.bufferedaction = nil
    end
end
```

**关键的两行**：

- `self.bufferedaction = bufferedaction` —— **缓存到实体上**——这就是为什么 timeline 能 `inst:GetBufferedAction()` 拿到它
- `self.sg:StartAction(bufferedaction)` —— **触发 SG 处理**——SG 内部找到对应 ActionHandler、切换状态

**`StartAction` 返回 false 的情况**——SG 决定不处理这个动作（比如当前 state 是 busy/casting，不能被打断）——立刻 Fail。

**注意 `if self.sg == nil`**——某些实体没有 SG（如纯静态物品），这种情况触发 `startaction` 事件——让组件自己监听响应。

#### 第二步：用控制台触发 PushBufferedAction

控制台手动测试：

```lua
-- 创建一个 BufferedAction 给玩家
local target = c_findnext("evergreen")
local axe = ThePlayer.replica.inventory:GetEquippedItem(EQUIPSLOTS.HANDS)
local ba = BufferedAction(ThePlayer, target, ACTIONS.CHOP, axe)

-- 推送
ThePlayer:PushBufferedAction(ba)

-- 玩家会自动走过去并砍树！
```

**这就是 `c_doaction` 控制台命令的内部实现**——本质就是构造一个 BufferedAction 并 push。

---

### 7.4.5 进阶：`PerformBufferedAction` & `Do` —— 中点

#### 第一步：`PerformBufferedAction` 完整流程

`entityscript.lua` 第 1654-1685 行：

```1654:1685:scripts/entityscript.lua
function EntityScript:PerformBufferedAction()
    if self.bufferedaction then
        if self.bufferedaction.target and self.bufferedaction.target:IsValid() and self.bufferedaction.target.Transform then
            self:FacePoint(self.bufferedaction.target.Transform:GetWorldPosition())
        end

        self:PushEvent("performaction", { action = self.bufferedaction })

		local action_theme_music = self:HasTag("player") and (self.bufferedaction.action.theme_music or (self.bufferedaction.action.theme_music_fn ~= nil and self.bufferedaction.action.theme_music_fn(self.bufferedaction)))
		if action_theme_music then
			self:PushEvent("play_theme_music", {theme = action_theme_music})
		end

		local bufferedaction = self.bufferedaction
		--@V2C: - cahced in case self.bufferedaction changes
		--      - ideally, should clear self.bufferedaction here
		--      - however, legacy code might rely on inst.bufferedaction during Do()

		local success, reason = bufferedaction:Do()
        if success then
			if self.bufferedaction == bufferedaction then
				self.bufferedaction = nil
			end
            return true
        end
```

**5 步流程**：

1. **检查存在性**：`if self.bufferedaction then`
2. **面向目标**：`self:FacePoint(...)` ——确保业务执行时朝向正确
3. **触发 performaction 事件**：让监听者（UI、特效、音效）能反应
4. **触发主题音乐**（如有）：某些动作有专属音乐（钓鱼、农耕）
5. **缓存 ba 引用 + 调 Do()**：`local bufferedaction = self.bufferedaction; bufferedaction:Do()`

**注意 `local bufferedaction = self.bufferedaction` 这个临时变量**——V2C（Klei 程序员）在注释里说"以防 Do() 期间 self.bufferedaction 被改"——**这是防御链式动作的设计**——某些 action.fn 内部会再 push 新的 bufferedaction（比如战斗连击）——临时变量保证我们处理的是"刚才那个"。

#### 第二步：`BufferedAction:Do()` 拆解

```22:37:scripts/bufferedaction.lua
function BufferedAction:Do()
    if not self:IsValid() then
        return false
    end
    local success, reason = self.action.fn(self)
    self.reason = reason
    if success then
        if self.invobject ~= nil and self.invobject:IsValid() then
            self.invobject:OnUsedAsItem(self.action, self.doer, self.target)
        end
        self:Succeed()
    else
        self:Fail()
    end
    return success, reason
end
```

**5 步**：

1. **`IsValid` 校验** —— 不通过直接 return false
2. **调用 `action.fn(self)`** —— 这是真正"做事"的地方
3. **保存 reason** —— `self.reason = reason`
4. **invobject 通知**（如果成功）——`OnUsedAsItem` 让物品知道"我刚被用过了"，可用于消耗耐久、统计使用次数等
5. **触发 Succeed / Fail 回调**

**两个微妙之处**：

- **失败时也会赋值 `self.reason = reason`**——但 `IsValid` 失败时 reason 是 nil（因为没走到 fn）——**老手要记住这个区别**
- **`OnUsedAsItem` 只在成功时调**——**这是耐久消耗的关键**——失败的动作不消耗耐久

#### 第三步：`IsValid` 的 6 项检查

```39:47:scripts/bufferedaction.lua
function BufferedAction:IsValid()
    return (self.invobject == nil or self.invobject:IsValid()) and
           (self.doer == nil or (self.doer:IsValid() and (not self.autoequipped or self.doer.replica.inventory:GetActiveItem() == nil))) and
           (self.target == nil or (self.target:IsValid() and self.initialtargetowner == (self.target.components.inventoryitem ~= nil and self.target.components.inventoryitem.owner or nil))) and
           (self.pos == nil or self.pos.walkable_platform == nil or self.pos.walkable_platform:IsValid()) and
           (not self.doerownsobject or (self.doer ~= nil and self.invobject ~= nil and self.invobject.replica.inventoryitem ~= nil and self.invobject.replica.inventoryitem:IsHeldBy(self.doer))) and
           (self.validfn == nil or self.validfn(self)) and
           (not TheWorld.ismastersim or (self.action.validfn == nil or self.action.validfn(self)))
end
```

**6 项 AND**——任何一项失败整个返回 false：

**(1) invobject 仍然存在**

```lua
self.invobject == nil or self.invobject:IsValid()
```

——**手里的物品没被 Remove**。包括"被吃掉"、"被消耗光耐久"、"被 mod 移除"。

**(2) doer 仍然有效 + 自动装备校验**

```lua
self.doer == nil or (self.doer:IsValid() and (not self.autoequipped or self.doer.replica.inventory:GetActiveItem() == nil))
```

——doer 没死/没被销毁。**`autoequipped` 检查**：如果工具是被自动装备的——验证 doer 的"主动手"是空的（防止自动装备和手动装备冲突）。

**(3) target 存在 + 所有者未变**

```lua
self.target == nil or (self.target:IsValid() and self.initialtargetowner == (self.target.components.inventoryitem ~= nil and self.target.components.inventoryitem.owner or nil))
```

——target 没死，**且 inventoryitem 的 owner 和 BufferedAction 创建时一致**。回顾 7.4.2 第二步 C 组——这是防御"动作中途目标被偷"的关键。

**实际场景**：你在地上看到一颗紫宝石，点击"捡起"——还没走到——**别的玩家先一步捡走了**——你走到位置后 `IsValid` 返回 false——动作不触发——避免"捡到 nil 物品"。

**(4) pos 仍然有效**

```lua
self.pos == nil or self.pos.walkable_platform == nil or self.pos.walkable_platform:IsValid()
```

——如果位置是在浮板上、浮板**还存在**（没被沉没/移除）。

**(5) doerownsobject 一致性**

```lua
not self.doerownsobject or (self.doer ~= nil and self.invobject ~= nil and self.invobject.replica.inventoryitem ~= nil and self.invobject.replica.inventoryitem:IsHeldBy(self.doer))
```

——**如果创建 ba 时 invobject 是 doer 持有的**，现在还应该是。**防御"动作中途工具被偷"**——比如你拿着斧头去砍树，途中被别人偷走斧头——`IsValid` 失败。

**(6) 自定义 validfn + action.validfn**

```lua
self.validfn == nil or self.validfn(self)
not TheWorld.ismastersim or (self.action.validfn == nil or self.action.validfn(self))
```

——回顾 7.1.6 节我们讲过 `action.validfn`。**注意 `not TheWorld.ismastersim` 的限定**——`action.validfn` 只在服务端校验。**为什么？** 因为 validfn 可能访问服务端独有的组件——客户端无法验证——所以**客户端不做这层校验，相信服务端的判断**。

#### 第四步：`IsValid` 的两次调用

注意 `IsValid` 在生命周期中**被调用两次**：

- **第一次**：`PushBufferedAction` 阶段——`TestForStart()` 别名调用
- **第二次**：`Do()` 阶段——执行前再校验

**两次的间隔**：玩家走过去的 1-3 秒。**两次都通过才能真正执行业务**。这就是 `IsValid` 设计的意义——**双保险**。

> **新手记忆**：**`Do() = IsValid + fn + Succeed/Fail`**——**`IsValid 是 6 项 AND`**——**任何一项失败动作就静默取消**。这就是为什么写 fn 时**不需要再做 nil 检查 IsValid 的内容**——已经保证过了。但**对组件存在性、tag 状态等**还是要做防御——`IsValid` 不检查这些。

---

### 7.4.6 进阶：`Succeed` / `Fail` / `Clear` / `Interrupt` —— 终点

#### 第一步：`Succeed` 与 `Fail`

```82:88:scripts/bufferedaction.lua
function BufferedAction:Succeed()
    for k, v in pairs(self.onsuccess) do
        v()
    end
    self.onsuccess = {}
    self.onfail = {}
end
```

```104:110:scripts/bufferedaction.lua
function BufferedAction:Fail()
    for k,v in pairs(self.onfail) do
        v()
    end
    self.onsuccess = {}
    self.onfail = {}
end
```

**两者结构完全对称**——遍历对应回调列表 + 清空两个列表。

**注意"双清空"**：

- `Succeed` 清空 `onsuccess` **和** `onfail`
- `Fail` 也是清空两者

**为什么？** 防止重复触发——比如 onsuccess 触发后某些代码又调一次 ba:Succeed()——已经清空了不会重复。

**什么时候 onfail 应该被触发**？

- **`Do() 内部 fn 返回 false`**——会调 `Fail`
- **`PushBufferedAction` 时 ba 已经存在**——旧 ba 被 `Fail`
- **`ClearBufferedAction`**——主动取消

**什么时候 onfail 不会被触发**？

- **`IsValid` 失败导致 `Do()` 提前 return false**——**重要陷阱**！`Do()` 第一行就 return，没调 Fail——**onfail 不会触发**！

> **老手陷阱**：**onfail 不能保证一定被调用**——如果你依赖 onfail 做关键清理（如释放资源），**要做好"可能不被调用"的防御**。**这是 7.4.8 节会讲的陷阱之一**。

#### 第二步：`AddSuccessAction` / `AddFailAction` —— 注册回调

```74:80:scripts/bufferedaction.lua
function BufferedAction:AddFailAction(fn)
    table.insert(self.onfail, fn)
end

function BufferedAction:AddSuccessAction(fn)
    table.insert(self.onsuccess, fn)
end
```

**就一行 push**——把回调加到对应列表。

**典型用法**（在 ActionHandler 之外的地方注册）：

```lua
local ba = BufferedAction(player, tree, ACTIONS.CHOP, axe)
ba:AddSuccessAction(function()
    print("砍树成功！")
    -- 可以做：弹奖励、统计、加成就...
end)
ba:AddFailAction(function()
    print("砍树失败原因:", ba.reason)
end)
player:PushBufferedAction(ba)
```

**这是动作系统**给开发者**最灵活的扩展点**——**不修改 fn、不修改 SG**——直接给 ba 挂回调就能做自定义反应。

#### 第三步：`ClearBufferedAction` 与 `InterruptBufferedAction`

```1563:1570:scripts/entityscript.lua
function EntityScript:ClearBufferedAction()
    if self.bufferedaction ~= nil then
        self.bufferedaction:Fail()
        self.bufferedaction = nil
    end
end

EntityScript.InterruptBufferedAction = EntityScript.ClearBufferedAction
```

**两个名字一个实现**——`Clear` 和 `Interrupt` 是别名。**Klei 用两个名字让代码语义更清楚**——

- `Clear` 用在"完成后清理"
- `Interrupt` 用在"中途打断"

**调用时机举例**：

- **玩家被攻击**——SG 切到 hit 状态——`onenter` 里 `inst:InterruptBufferedAction()`——砍树动作被打断
- **玩家死亡**——`death` 状态 onenter `inst:ClearBufferedAction()`——清理所有未完成动作

#### 第四步：`PreviewBufferedAction` 与客户端预测

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

**结构和 PushBufferedAction 几乎一样**——但**只在客户端用、走 PreviewAction 而不是 StartAction**。

**作用**：**客户端预测**——在没有等到服务端确认之前，**先在本地播一段动画**让玩家感觉"立刻有反应"。然后通过 `PerformPreviewBufferedAction` 把"我想做这个动作"的 RPC 发给服务端。

**这是网络延迟优化的核心**——**100ms 延迟下玩家也能感觉"按下按钮立刻有反应"**。但**预测可能错**——服务端如果拒绝（IsValid 失败），客户端会"回滚"——回到 idle 状态。

> **新手记忆**：**Preview 是客户端预测专属，Push 是服务端真正执行**。绝大多数自定义代码不需要操心 Preview——它由 PlayerController 自动处理。**只有写"有特殊客户端预测需求"的动作时才会涉及**。

---

### 7.4.7 老手进阶：`AddSuccessAction` / `AddFailAction` 链式回调实战

#### 第一步：经典使用模式

**模式 A：动作完成奖励**

```lua
-- 给某个特殊 BufferedAction 挂"完成额外奖励"
ba:AddSuccessAction(function()
    if math.random() < 0.1 then
        ThePlayer.components.inventory:GiveItem(SpawnPrefab("goldnugget"))
    end
end)
```

**适用**：自定义"幸运掉落"——某些动作（采花、砍树、挖矿）有概率奖励——不修改原版 fn 就能加。

**模式 B：动作失败时"补偿"**

```lua
-- 玩家想合成但条件不满足——给个软提示
local ba = BufferedAction(player, nil, ACTIONS.BUILD, nil, nil, "myrecipe")
ba:AddFailAction(function()
    if ba.reason == "NEED_FUEL" then
        player.components.talker:Say("需要更多燃料...")
    elseif ba.reason == "NOTNOW" then
        player.components.talker:Say("现在还不行...")
    end
end)
```

**适用**：让玩家**清楚为什么失败**——通过 `ba.reason` 区分原因、定制反馈。

**模式 C：链式动作**

```lua
-- 玩家先 PICK 后 EAT（合一为"吃"操作）
local pickba = BufferedAction(player, berry, ACTIONS.PICK)
pickba:AddSuccessAction(function()
    -- PICK 成功后立刻发起 EAT
    local berryitem = player.components.inventory:FindItem(function(item)
        return item.prefab == "berries"
    end)
    if berryitem then
        local eatba = BufferedAction(player, nil, ACTIONS.EAT, berryitem)
        player:PushBufferedAction(eatba)
    end
end)
player:PushBufferedAction(pickba)
```

**适用**：**复合操作的简化** —— 一次"吃浆果"按理说要"摘 → 吃"两步——通过链式 ba 让玩家点一次完成两步。

> **进阶提示**：**链式 BufferedAction 时要小心循环引用** —— 如果 ba1 的 onsuccess 推 ba2、ba2 的 onsuccess 又推 ba1——会无限循环。**永远在链里设置一个"终止条件"**。

#### 第二步：onsuccess / onfail 的内部机制

注意 `Succeed` / `Fail` 内部用 `pairs` 而不是 `ipairs`：

```lua
for k, v in pairs(self.onsuccess) do
    v()
end
```

**意思**：**回调顺序不保证**——按照 table 的 hash 顺序遍历。**如果你的回调有顺序依赖**——这是个隐患。

**安全用法**：

- **多个独立的回调** —— `pairs` 顺序无所谓
- **有顺序依赖的逻辑** —— **写在同一个回调里**，不要分多个 `AddSuccessAction`

#### 第三步：DynamicPosition —— pos 字段的高级特性

`bufferedaction.lua` 第 9 行：

```lua
self.pos = pos ~= nil and DynamicPosition(pos) or nil
```

——**pos 不是 Vector3 直接存——而是包装成 `DynamicPosition`**。

**`DynamicPosition` 是什么**？一个**"会跟着浮板移动的位置"**。

**问题**：玩家在浮板上点击地面位置 → 创建 BufferedAction(pos=...) → 走过去要 1 秒 → 期间浮板移动了——**点击的"位置"也应该跟着移动**。

**`DynamicPosition` 的解决方案**：

- 内部存 `walkable_platform`（浮板引用）+ 相对浮板的偏移
- 调 `:GetPosition()` 时实时计算 = 浮板当前位置 + 偏移

**取实际位置**：

```lua
local pos = ba:GetActionPoint()  -- 返回 Vector3（实时计算）
-- 而不是 ba.pos （那是 DynamicPosition 对象）
```

**这是为什么 `bufferedaction.lua` 提供 `GetActionPoint` / `GetDynamicActionPoint` / `SetActionPoint` 这三个 setter/getter**——封装 DynamicPosition 的复杂性。

> **老手记忆**：**写浮板相关动作时永远用 `ba:GetActionPoint()` 取位置**——绝不直接 `ba.pos.x` —— 后者拿到的是 DynamicPosition 对象不是 Vector3。

---

### 7.4.8 老手进阶：六个常见陷阱与设计经验

#### 陷阱 1：在 fn 里修改 BufferedAction 字段

```lua
-- ❌ 错：在 fn 里改 act.target
ACTIONS.MYACTION.fn = function(act)
    act.target = SomeOtherEntity()  -- ❌ 修改 ba 字段
    return true
end
```

**症状**：`Succeed` 回调执行时 ba.target 已经被改——**onsuccess 里再读 target 拿到错误值**——玩家可能"以为我砍 A 但奖励来自 B"。

**正解**：fn 内**只读**——做计算就好——返回 true/false。**修改 ba 只能在 PushBufferedAction 之前**。

#### 陷阱 2：onfail 里依赖动作已"完成"

```lua
-- ❌ 错：onfail 假设 fn 已经跑过
ba:AddFailAction(function()
    if ba.reason == "NOFUEL" then        -- 可能是 IsValid 失败导致的——reason 是 nil！
        player.components.talker:Say("没燃料")
    end
end)
```

**问题**：`IsValid` 失败时 `Fail` 不被调用——**但 `PushBufferedAction` 第二步"失效旧 ba" 时会调 Fail**——这时 reason 永远是 nil（旧 ba 没经过 fn）。

**正解**：永远做 nil 检查：

```lua
ba:AddFailAction(function()
    if ba.reason == nil then
        return  -- 不是 fn 触发的失败，跳过
    end
    if ba.reason == "NOFUEL" then
        player.components.talker:Say("没燃料")
    end
end)
```

#### 陷阱 3：忘记 BufferedAction 是"一次性"的

```lua
-- ❌ 错：复用同一个 ba 多次
local ba = BufferedAction(player, tree, ACTIONS.CHOP, axe)
player:PushBufferedAction(ba)
-- ... 等砍完
player:PushBufferedAction(ba)  -- ❌ 复用！
```

**问题**：**ba 内部的 onsuccess/onfail 第一次调用后被清空**——第二次 push 时这些回调全没了。**而且 IsValid 检查里的 `initialtargetowner` 是基于第一次创建时的状态——可能已经过期**。

**正解**：**每次都创建新的 BufferedAction**——它就是一次性的"快递订单"。

#### 陷阱 4：在 onsuccess 内部 PushBufferedAction 同实体

```lua
ba:AddSuccessAction(function()
    -- 触发新动作
    player:PushBufferedAction(BufferedAction(...))
end)
```

**症状**：可能正常、可能异常——取决于具体时机。最危险的是**时序竞态**——onsuccess 在 Do() 末尾被调——而 self.bufferedaction 在 PerformBufferedAction 末尾才被清空——**期间 push 新 ba 可能被认为"重复"或"失败"**。

**安全方案**：用 `inst:DoTaskInTime(0, ...)` 延迟到下一帧：

```lua
ba:AddSuccessAction(function()
    player:DoTaskInTime(0, function()  -- 延迟 1 帧
        player:PushBufferedAction(BufferedAction(...))
    end)
end)
```

#### 陷阱 5：用 `act.pos` 直接当 Vector3

```lua
-- ❌ 错：当 Vector3 用
ACTIONS.MYACTION.fn = function(act)
    print(act.pos.x, act.pos.z)   -- ❌ act.pos 是 DynamicPosition 不是 Vector3！
    -- 在浮板上时拿到的 x, z 可能是相对偏移而不是世界坐标
end

-- ✅ 对：用 GetActionPoint
ACTIONS.MYACTION.fn = function(act)
    local pt = act:GetActionPoint()  -- Vector3
    print(pt.x, pt.z)
end
```

**回顾 7.4.7 第三步**——浮板上的动作必须用 GetActionPoint。

#### 陷阱 6：忽视 forced 字段的语义

```lua
-- ❌ 错：忘了考虑远程指定
ACTIONS.MYACTION.fn = function(act)
    -- 假设玩家就在 target 旁边
    PlaySound(act.target.Transform:GetWorldPosition())
end
```

**问题**：`forced=true`（远程 Shift+Click）时——动作创建瞬间玩家**离 target 还远**——但因为是 forced，引擎会让玩家自动走过去。**fn 执行时玩家应该已经走到了——但有些边角 case（如目标移动、强制路径阻挡）—— fn 里假设玩家在身边可能错**。

**正解**：写 fn 时不要假设玩家位置——**所有距离计算用 act.doer 的实时位置**——并做防御。

#### 设计经验三条

**经验 1：fn 永远只做"业务执行"——不做"业务决策"**

**业务决策**（"该不该执行"）放在 ActionHandler、collector、validfn 里。**fn 只负责"既然走到这了——把事做了"**。

```lua
-- ❌ 不好：fn 里再做大量条件判断
ACTIONS.MYACTION.fn = function(act)
    if act.doer.components.sanity:GetCurrent() < 30 then return false end
    if act.target:HasTag("ghost") then return false end
    -- ... 一堆判断 ...
    -- 然后做事
end

-- ✅ 好：判断在 collector / ActionHandler condition 里、fn 只做事
AddComponentAction("SCENE", "myxxx", function(inst, doer, actions, right)
    if doer.replica.sanity:GetPercent() > 0.3 and not inst:HasTag("ghost") then
        table.insert(actions, ACTIONS.MYACTION)
    end
end)

ACTIONS.MYACTION.fn = function(act)
    -- 不再判断、直接做
    DoMyAction(act)
    return true
end
```

**经验 2：用 reason 做"细粒度失败原因"**

```lua
ACTIONS.MYACTION.fn = function(act)
    if not act.target.components.psionic then
        return false, "NO_PSIONIC"
    end
    if act.target.components.psionic:IsEmpty() then
        return false, "EMPTY"
    end
    if act.doer.components.health:IsLow() then
        return false, "TOOWEAK"
    end
    -- 成功
    return true
end
```

——**配合 `STRINGS.ACTIONFAIL.MYACTION = { NO_PSIONIC = "...", EMPTY = "...", TOOWEAK = "..." }`**——回顾 7.1.8 节关于 STRINGS 的讲解。

**经验 3：onsuccess / onfail 用 closure 而不是字符串方法名**

```lua
-- ❌ 不好：方法名容易拼错、不能跨实体
function MyComponent:HandleAttackSuccess()
    -- ...
end
ba:AddSuccessAction("HandleAttackSuccess")  -- 错的，不支持字符串

-- ✅ 好：closure
ba:AddSuccessAction(function()
    if self.inst.components.mycomponent then
        self.inst.components.mycomponent:HandleAttackSuccess()
    end
end)
```

**closure 让你可以捕获组件实例、做完整 nil 检查**——这是 Lua 中"回调"的标准写法。

---

### 7.4.9 小结

- **BufferedAction 是 Action 系统的"运行时载体"**——一个 110 行的小类，承载了 Action 系统的所有运行时状态。**它是连接 ACTIONS / ComponentAction / ActionHandler / SG / fn 五大模块的胶水**。
- **14 个字段** —— 核心是 `doer / target / action / invobject` 4 个；其他是位置、所有权防御、可选参数、回调列表、动态字段。**90% 时候只用前 4 个**。
- **生命周期 6 阶段**：创建 → 推送（PushBufferedAction）→ SG 状态切换 + 走路 → 业务执行（PerformBufferedAction → Do）→ 回调（Succeed / Fail）→ 销毁。**整段时间它挂在 `inst.bufferedaction` 上**——这是 timeline 能 GetBufferedAction 的关键。
- **`PushBufferedAction` 5 步流程**：重复检查 → 失效旧 ba → TestForStart 校验 → 三个分支（WALKTO/instant/一般）→ 缓存 + 触发 SG。
- **`Do()` 5 步流程**：IsValid 校验 → action.fn → 保存 reason → invobject.OnUsedAsItem（成功）→ 触发 Succeed/Fail。
- **`IsValid` 6 项检查**：invobject 存在、doer 有效、target 有效 + 所有者未变、pos 浮板有效、doerownsobject 一致、validfn 通过。**两次调用（Push 和 Do）做双保险**。
- **`Succeed` / `Fail` 触发回调列表 + 双清空**——但 **`IsValid 失败时 Fail 不被调用**——这是隐藏陷阱。
- **`AddSuccessAction` / `AddFailAction` 是 Action 系统最灵活的扩展点**——不修改 fn / SG 就能挂额外逻辑。
- **6 个常见陷阱**：fn 内修改 ba 字段、onfail 假设 fn 跑过、复用 ba、onsuccess 内 push 同实体、`act.pos` 当 Vector3 用、忽视 forced 语义。
- **3 条设计经验**：fn 只做"业务执行"不做"业务决策"、用 reason 做细粒度失败、回调用 closure。

> **下一节预告**：7.5 节我们打开 **`playercontroller.lua`** —— 玩家系统中**最大的组件**之一（5000+ 行）。回答 BufferedAction 的"创造之谜"——**鼠标点击屏幕**怎么变成一个 BufferedAction？**键盘按 F 键**怎么变成 ACTIONS.ATTACK？**手柄方向键**怎么变成 LOCOMOTE？我们会看到 PlayerActionPicker 如何遍历周围实体收集 actions、如何按 priority 选择最佳、如何把选择结果包装成 BufferedAction 通过 RPC 传给服务端——**这是把"输入"变成"动作"的最后一公里**。

---

## 7.5 PlayerController——玩家输入如何转化为 Action

（待编写）

## 7.6 动作的优先级与冲突解决

（待编写）

## 7.7 实战：添加一个完全自定义的新动作

（待编写）
