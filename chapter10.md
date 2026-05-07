# 第10章 Save/Load 持久化系统

## 10.1 存档的整体结构与加载流程

### 本节导读

第 9 章我们把"运行时网络同步"讲透了——但**网络同步只解决"游戏运行期间的状态广播"**。**关闭游戏 / 重启服务器之后**——所有这些 NetVar、所有玩家位置、所有箱子物品、所有怪物 AI 状态都会丢失——除非**写到磁盘**。

这就是 **存档系统**——把"内存中的世界"序列化成磁盘文件，下次启动时反序列化回来。

> 想象你刚好砍倒一棵树、刚捡了 3 根木头、刚走到地图东边——按 ESC 退出。**几秒后重启游戏**——树还是倒的、木头还在背包、你还站在地图东边。**这一切是怎么做到的**？

饥荒联机版的存档系统**比单机版复杂得多**：
- 多个 shard（地表、洞穴）各自存档
- 每个玩家的私有数据要存
- 实体之间的引用要用 GUID 重建
- 世界状态、玩家状态、Mod 配置全部要持久化
- **mod 升级后旧存档不能崩**

这一节我们把存档系统的**整体架构**讲清楚——从磁盘上的目录结构、到 lua 层的"实体序列化"、到游戏启动时"反序列化"重建世界的完整流程。

> **新手**从 10.1.1-10.1.3 起步——理解存档的存在意义、存档目录树、5 步加载流程；**进阶读者**继续看 10.1.4-10.1.6，深入 world prefab 中 ents 的数据结构、references 与 GUID 解析机制、`savefileupgrades.lua` 兼容升级；**老手**跳到 10.1.7-10.1.8，看存档调试控制台命令、6 个最容易踩的隐蔽坑。

---

### 10.1.1 快速入门：从一次"退出 + 重进"看存档系统是什么

#### 第一步：观察一次完整的"存档周期"

**进游戏 → 砍树 → 退出 → 重进**：

```
[T=0]   玩家进入服务器
        引擎从磁盘读取存档文件
        反序列化为 lua table
        逐一 SpawnPrefab 重建世界
        每个 entity 调用 OnLoad 恢复状态
        TheWorld 准备完毕
        玩家可以开始玩

[T=300] 玩家砍倒一棵树
        服务端 health.lua 改了树的 currenthealth
        BUT 这只是**内存中**的修改
        磁盘上的存档没变

[T=600] 玩家按 ESC → 退出服务器
        引擎触发 SaveGame
        遍历所有 entity → 调 GetSaveRecord
        每个组件 OnSave 把自己的数据塞 record.data
        所有 record 收集成大表 → 序列化 → 写入磁盘

[T=900] 玩家重新进入
        ...回到 T=0 的流程
        刚才砍的那棵树**没了**（已经从存档移除）
        刚才捡的木头**仍在玩家背包**
        玩家位置**和退出时一致**
```

**这就是 OnSave / OnLoad 的全部作用**——把"内存状态"和"磁盘文件"互相同步。

#### 第二步：lua 层如何"看到"存档？

引擎调用 `EntityScript:GetSaveRecord()`：

```248:316:scripts/entityscript.lua
function EntityScript:GetSaveRecord()
	local record =
	{
		prefab = self.prefab,
	}

	local x, y, z = self.Transform:GetWorldPosition()

	if self.temp_save_platform_pos or self.entity:HasTag("player") then
		record.age = self.Network:GetPlayerAge()
		...
    end
	...

	record.x = x and math.floor(x * 1000 + 0.5) * 0.001 or 0
	record.z = z and math.floor(z * 1000 + 0.5) * 0.001 or 0
    ...
    record.skinname = self.skinname
    record.skin_id = self.skin_id
    record.alt_skin_ids = self.alt_skin_ids

    local references
    record.data, references = self:GetPersistData()

    return record, references
end
```

**每个 entity 序列化成一个 record**：

```lua
{
    prefab = "tree",
    x = 12.345,           -- 位置 X
    z = -8.123,           -- 位置 Z
    skinname = nil,
    data = {              -- 各组件的 OnSave 数据汇总
        workable = { workleft = 9, ... },
        burnable = { ... },
        growable = { stage = 3, ... },
        ...
    },
}
```

**整个世界被序列化**：所有 entity 的 record 收集到一个大数组，写到 `0000000xxx` 文件。

#### 第三步：保存只是序列化、加载是反序列化

```
保存：内存 entity 树 → 一个个 record → 序列化 → 磁盘字符串
加载：磁盘字符串 → 反序列化 → 一个个 record → SpawnPrefab + SetPersistData → 内存 entity 树
```

> **核心结论**：**存档系统 = 把内存中的实体树序列化为磁盘文件、反过来反序列化重建**。**99% 的工作由引擎自动做**——mod 开发者只需要写 OnSave / OnLoad 处理自己的特殊数据。

---

### 10.1.2 快速入门：饥荒的"存档目录树"

#### 第一步：磁盘上的目录结构

**单机版 / 主机端联机** 存档位置：
```
%USERPROFILE%/Documents/Klei/DoNotStarveTogether/<userid>/
├── client_save/
│   ├── 0000000001                ← session_xx 的存档文件
│   ├── 0000000002
│   ...
│   ├── server_temp/              ← 主机端临时数据
│   ├── shardindex                ← 元数据（地图配置、生成的种子）
│   └── ...
└── client_temp/
    └── client_log.txt
```

**Dedicated 服务器** 存档位置（更复杂）：
```
.klei/DoNotStarveTogether/<cluster_name>/
├── Cluster_1/
│   ├── cluster.ini               ← 集群配置
│   ├── Master/
│   │   ├── server.ini            ← 主 shard 配置
│   │   ├── shardindex            ← 主 shard 元数据
│   │   └── save/
│   │       ├── session/<sessionid>/
│   │       │   ├── 0000000001    ← 实际世界数据
│   │       │   └── 0000000002
│   │       └── meta/
│   ├── Caves/                    ← 洞穴 shard 同样结构
│   │   ├── server.ini
│   │   ├── shardindex
│   │   └── save/...
│   └── modoverrides.lua          ← Mod 配置
└── Cluster_2/...
```

#### 第二步：3 个核心存档文件

**1. shardindex** — 存档**元数据**（创建时间、地图设置、玩家列表、种子）  
**2. session/<sessionid>/000000XX** — 实际**世界数据**（所有 entity 的 record + world 状态）  
**3. meta** — 存档**封面信息**（用于"载入存档"菜单显示标题、玩家头像、最后游戏时间等）

#### 第三步：sessionid 的作用

每个"游戏会话"有一个唯一 `sessionid`。**为什么不用一个固定文件？**

- **断电恢复**：当前 session 写中途断电——还有上一个 session 完整文件
- **快照管理**：游戏可以保留最近 N 个 session 作为"快照"
- **回滚机制**：管理员命令 `c_rollback(N)` 可以回到 N 个 session 之前

> 看 `scripts/shardindex.lua:170-187` —— `LoadShardInSlot` 等接口处理 dedicated 和 client-hosted 两种部署。

---

### 10.1.3 快速入门：游戏启动时存档加载的 5 步流程

```
[1] 引擎初始化、ModManager 加载 mod、modmain.lua 跑
        ↓
[2] 读取 shardindex 元数据 → 知道地图配置、种子等
        ↓
[3] 读取 session/<sid>/0000000XX → 反序列化为 lua table
    table 结构：
    {
        ents = {
            tree = { record1, record2, ... },
            pigman = { record1, record2 },
            ...
        },
        map = { ... },
        meta = { ... },
        snapshot = { ... },
    }
        ↓
[4] SpawnSaveRecord(record):
    a) SpawnPrefab(record.prefab)
    b) entity.Transform:SetPosition(record.x, record.y, record.z)
    c) entity:SetSkin(record.skinname)
    d) entity:SetPersistData(record.data, newents)
        ↓
[5] LoadPostPass:
    所有 entity SetPersistData 完成后，再走一遍 entity:LoadPostPass
    用于"等所有实体存在后才能解析的引用"
```

**关键观察**：

- **Step 4d 是触发各组件 OnLoad 的地方**——见 `entityscript.lua:1986-1988`
- **Step 5 是 LoadPostPass**——专为"引用解析"设计（你 follower 的 leader 必须等 leader 也 spawn 出来才能 link）

#### 整个流程的"双阶段"

**阶段 1：构造**（spawn + SetPersistData）—— 每个 entity 有了，但**互相之间的引用可能还指向 nil**。

**阶段 2：引用解析**（LoadPostPass）—— 每个 entity 找到自己引用的目标 entity（用 GUID 转换），完成绑定。

---

### 10.1.4 进阶：world prefab 内 ents 的数据结构

#### 第一步：savedata 的顶层结构

存档反序列化后是个 lua table，结构如下：

```lua
savedata = {
    ents = {
        ["tree"] = {
            { x = 1, z = 2, prefab = "tree", data = {workable = {workleft = 9}}, id = 12345 },
            { x = 3, z = 4, prefab = "tree", data = {workable = {workleft = 5}}, id = 12346 },
            ...
        },
        ["pigman"] = {
            { x = 10, z = 20, prefab = "pigman", data = {...}, id = 12350 },
        },
        ...
    },
    map = {
        prefab = "forest",            -- 主 world prefab
        topology = {...},
        nav = "...",                  -- 导航数据（编码字符串）
        roads = {...},
    },
    meta = {
        clock = {...},
        seasons = {...},
        ...
    },
    snapshot = {
        users = {...},                -- 玩家列表
        playerdata = {...},           -- 各玩家数据
    },
    super = false,
    mode = "survival",
}
```

#### 第二步：ents[prefab] 是数组

**注意 `ents.tree`** 是个**数组**——不是按 GUID 索引的字典——而是按 prefab **分类**的列表：

- 加载时按 prefab 分组遍历
- 每组先把所有同 prefab 的 entity spawn 完
- 然后遍历这组的 record，挨个 `SpawnPrefab + SetPersistData`

**这种结构的优点**：连续 spawn 同 prefab 时，引擎能复用缓存的 prefab 模板；**缺点**：record 之间的关系丢失（按 GUID 引用要靠 LoadPostPass 解决）。

#### 第三步：record.id 用于引用解析

每个 record 有一个 `id`（旧 GUID）—— **这是序列化时的 GUID 快照**——和加载后 entity 的真实 GUID **不一定相同**——但 `references` 表用旧 ID 标记关系。

**典型场景**：玩家有个 follower 猪人——存档：

```lua
-- 玩家 record
{ prefab = "wilson", data = { ... } }
-- 猪人 record
{ prefab = "pigman", data = { follower = { leader = 42 } } }  -- ★ 42 是玩家在存档时的 GUID
```

**加载后**：玩家的真实 GUID 可能变成 99——猪人的 follower.leader 字段需要被**翻译**：旧 42 → 新 99。**这就是 LoadPostPass 的工作**。

---

### 10.1.5 进阶：references 与 GUID 解析机制

#### 第一步：references 是什么

回到 `entityscript.lua:1903-1937`：

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

**`references`** 是组件 `OnSave` 返回的**第二个值**——一个数组，里面是**该 entity 引用的其他 entity 的 GUID**。

#### 第二步：组件 OnSave 怎么返回 references

**典型 follower 组件的 OnSave**：

```lua
function Follower:OnSave()
    if self.leader and self.leader:IsValid() then
        return { leader = self.leader.GUID }, { self.leader.GUID }
        --     ↑ data table                    ↑ references 数组
    end
end
```

**两件事**：
1. 把 leader 的 GUID 写进 data
2. 同时把 GUID 加入 references —— **告诉引擎："这个 entity 引用了 GUID=X，请确保它一起被加载"**

#### 第三步：LoadPostPass 解析

`entityscript.lua:1939-1952`：

```1939:1952:scripts/entityscript.lua
function EntityScript:LoadPostPass(newents, savedata)
    if savedata ~= nil then
        for k, v in pairs(savedata) do
            local cmp = self.components[k]
            if cmp ~= nil and cmp.LoadPostPass ~= nil then
                cmp:LoadPostPass(newents, v)
            end
        end
    end

    if self.OnLoadPostPass ~= nil then
        self:OnLoadPostPass(newents, savedata)
    end
end
```

**`newents`** 参数 —— 一个**全局映射表**：旧 GUID → 新 entity。**LoadPostPass 里**用它解析引用：

```lua
function Follower:LoadPostPass(newents, savedata)
    if savedata.leader and newents[savedata.leader] then
        local leader = newents[savedata.leader].entity
        self:SetLeader(leader)
    end
end
```

**`newents[oldid]`** 拿到一个 wrapper 对象，取 `.entity` 得到真实的新 entity。

#### 第四步：为什么需要两阶段？

**阶段 1（SetPersistData）**：spawn 完所有 entity——但**没有的 entity 还没 spawn**——比如猪人在存档前面、玩家在后面。

**阶段 2（LoadPostPass）**：所有 entity 都 spawn 完了——`newents` 表完整——可以安全做引用绑定。

---

### 10.1.6 进阶：`savefileupgrades.lua` 兼容升级

#### 第一步：版本号 + 升级器

`scripts/savefileupgrades.lua` 有一个 **upgrade chain**：

```lua
local upgrades = {
    -- 版本 1 → 2 的升级器
    { version = 2, fn = function(savedata)
        -- 旧 savedata 只有 'cycles_left'，新版本要 'cycles_passed'
        savedata.world.cycles_passed = (savedata.world.cycles_left and savedata.world.maxcycles - savedata.world.cycles_left) or 0
        savedata.world.cycles_left = nil
    end },
    -- 版本 2 → 3
    { version = 3, fn = function(savedata)
        ...
    end },
    -- ...
}
```

**加载时**：
1. 读取 savedata.version（如果没有就当 1）
2. 从当前版本走到最新版本，依次执行升级器
3. 升级完成后开始正常加载

#### 第二步：保证向前兼容

**Klei 的设计哲学**：**老存档永远能加载到最新版本**——不允许"老存档不能玩了"——通过升级链实现。

**对 mod 开发者的启示**：mod 自己也应该实现兼容升级——用版本号 + 升级 fn。

#### 第三步：mod 自己的升级机制

```lua
-- 自己组件里
function MyComp:OnSave()
    return {
        version = 2,             -- 当前版本
        new_field = self.x,
    }
end

function MyComp:OnLoad(data)
    if data == nil then return end
    
    -- 兼容老存档
    if data.version == nil or data.version == 1 then
        -- v1 存档没有 new_field，从旧字段迁移
        self.x = data.old_field or 0
    else
        self.x = data.new_field
    end
end
```

> **设计经验**：**永远把 mod 数据视为"可能来自任何旧版本"** —— 加 version 字段——升级时迁移而不是丢弃。详见 10.5 节。

---

### 10.1.7 老手进阶：保存/加载控制台调试

#### 第一步：手动触发保存

```lua
-- 主机端控制台
TheWorld:PushEvent("ms_save")  -- 触发存档
-- 等几秒，看到"World Saved"消息
```

#### 第二步：手动触发回滚

```lua
c_rollback(1)  -- 回到 1 个 session 之前的存档
c_rollback(0)  -- 重载当前 session（撤销内存中的未保存变更）
```

#### 第三步：dump 单个 entity 的 save record

```lua
local ent = c_select()
local record, refs = ent:GetSaveRecord()
print(json.encode(record))
-- 看到 {"prefab":"tree","x":12.5,"z":-3.2,"data":{...}}
```

#### 第四步：dump 单个组件的 OnSave 数据

```lua
local ent = c_select()
local data = ent.components.workable:OnSave()
print(json.encode(data))
```

#### 第五步：观察 newents 表

加载时 newents 是个临时变量——通常你看不到。**调试方法**：在你的组件 LoadPostPass 第一行加 `print("newents:", newents)`——或用 `inst:DoTaskInTime(2, function() ... end)` 等待加载完成后查看。

---

### 10.1.8 老手进阶：六个常见陷阱

#### 陷阱 1：OnSave 写入循环引用

**症状**：序列化时栈溢出或卡死。  
**原因**：

```lua
function MyComp:OnSave()
    return { ref_to_self = self }  -- ❌ 循环引用
end
```

**修复**：永远只存基本类型（number / string / bool）+ 别的 entity 的 GUID：

```lua
function MyComp:OnSave()
    return { ref_id = self.target and self.target.GUID }, { self.target and self.target.GUID }
end
```

#### 陷阱 2：OnLoad 第一行没判 nil

**症状**：mod 第一次启用时——存档没有这个组件的数据——`data` 是 nil——报错。  
**修复**：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    -- ...
end
```

#### 陷阱 3：在 SetPersistData 阶段访问其他实体

**症状**：你的组件 OnLoad 里 `inst.target = SomeOtherEntity` —— SomeOtherEntity 还没 spawn。  
**修复**：把"引用解析"移到 LoadPostPass：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    self._saved_target_id = data.target_id  -- 暂存
end

function MyComp:LoadPostPass(newents, savedata)
    if self._saved_target_id and newents[self._saved_target_id] then
        self.target = newents[self._saved_target_id].entity
    end
end
```

#### 陷阱 4：保存的字段太大

**症状**：存档文件膨胀到几十 MB——加载时间变长——某些情况下读取失败。  
**原因**：把"运行时缓存"也存进了 OnSave。  
**修复**：永远问"这个字段需要存吗？" —— 临时缓存、客户端预测数据、视觉状态都不存。

#### 陷阱 5：persists = false 的 entity 被存了

**症状**：mod 创建了"特效实体"——本应一闪即逝——但存档里也有它——加载后又冒出来。  
**原因**：忘了 `inst.persists = false`。  
**修复**：所有"临时实体"（特效、UI 中介、自动生成的预览）必须：

```lua
inst.persists = false
```

#### 陷阱 6：SetPersistData 没传 data 直接索引

**症状**：

```lua
function MyComp:OnLoad(data)
    self.x = data.x  -- ❌ data 可能是 nil
end
```

**修复**：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    self.x = data.x or 0
end
```

#### 设计经验三条

**经验 ①：永远只存"恢复必需"的数据**

存档**不是数据库**——只存"重建实体所必需"的最小数据：
- 当前血量、最大血量 → 必存
- "最近几次攻击的伤害历史" → 不存（运行时缓存，丢了无所谓）
- 网络变量 → 不存（NetVar 是协议层，不写入磁盘）

**经验 ②：版本号优先**

mod 第一版的 OnSave 就**加 version**：

```lua
function MyComp:OnSave()
    return { version = 1, x = self.x }
end
```

未来 v2 加新字段时：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    if data.version == 1 then
        -- v1 → v2 的升级
        self.x = data.x
        self.y = 0  -- 新字段默认值
    elseif data.version == 2 then
        self.x = data.x
        self.y = data.y
    end
end
```

**经验 ③：服务端永远走完整 save/load 流程，客户端不处理存档**

存档**只在服务端**。客户端**不需要 OnSave/OnLoad**——它的状态完全靠 NetVar 同步。**mod 写存档代码时永远在 `if TheWorld.ismastersim` 路径**。

---

### 10.1.9 小结

**存档系统一句话总结**：**所有 entity 序列化为 record（含 prefab + 位置 + 各组件 OnSave 数据 + references），加载时按 prefab 分组 spawn + SetPersistData + LoadPostPass 双阶段重建**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 给组件加 OnSave | `function MyComp:OnSave() return {x=self.x} end` |
| 给组件加 OnLoad | `function MyComp:OnLoad(data) if data then self.x = data.x end end` |
| 给组件加 LoadPostPass | `function MyComp:LoadPostPass(newents, savedata) ... end` |
| 让 entity 不存档 | `inst.persists = false` |
| 手动触发保存 | `TheWorld:PushEvent("ms_save")` |
| 回滚到上次保存 | `c_rollback(0)` |
| dump entity 的 save record | `json.encode(c_select():GetSaveRecord())` |

**5 步加载流程**

1. 加载 mod / shardindex 元数据
2. 反序列化 session 文件为 lua table
3. 按 prefab 分组遍历 ents
4. SpawnPrefab + SetPersistData（含组件 OnLoad）
5. LoadPostPass（含组件 LoadPostPass + entity OnLoadPostPass）

**6 个陷阱排雷顺序**

1. OnSave 循环引用 → 只存基本类型 + GUID
2. OnLoad 没判 nil → 第一行 if data == nil then return
3. SetPersistData 访问其他实体 → 移到 LoadPostPass
4. 字段太大 → 只存恢复必需
5. 临时实体被存 → persists = false
6. data 直接索引 → 加 nil 判

**3 条设计经验**

- ① **只存恢复必需**
- ② **版本号优先**——v1 就加，未来兼容
- ③ **服务端独占**——客户端不写存档

> **下一节预告**：10.2 节我们将**深入 OnSave / OnLoad** 的完整 API——参数详解、返回值规范、组件级 vs entity 级、读写时机、典型 mod 用法。这是把 10.1 的"整体架构"落地到具体代码的关键章节。读完 10.2，你将能给任何自定义组件加上"持久化能力"。

## 10.2 OnSave / OnLoad——组件数据如何随存档保存

### 本节导读

10.1 节我们把存档系统的**整体架构**讲清楚了——保存时遍历所有 entity 的所有组件，调它们的 `OnSave` 收集数据；加载时反过来调 `OnLoad` 恢复状态。**机制层面**已经清楚——但每次具体写组件 mod 时——**怎么写一对正确的 OnSave/OnLoad**？

- 应该返回什么数据格式？
- 多个字段怎么组织？
- 哪些字段必须存？哪些可以省？
- 引用其他实体怎么处理？
- 加载顺序对吗？

这一节是**组件持久化的"实操手册"**——我们打开 Klei 几个核心组件（`health`、`container`、`follower`、`inventory`）的真实 OnSave/OnLoad 代码——逐句拆解 Klei 的设计选择——再给出 mod 写自定义组件的标准模板。

> **新手**从 10.2.1-10.2.3 起步——理解 OnSave/OnLoad 的存在意义、API 速查、组件级 vs entity 级两套 hook；**进阶读者**继续看 10.2.4-10.2.6，深入嵌套保存（containers 用 GetSaveRecord）、references + GUID 的 follower 完整模式、`OnPreLoad` 这个鲜为人知但极有用的钩子；**老手**跳到 10.2.7-10.2.8，看自定义组件完整范例 + 6 个最容易踩的坑。

---

### 10.2.1 快速入门：从 health 组件看 OnSave / OnLoad 是什么

#### 第一步：看 Klei 最简单的 OnSave / OnLoad 范例

`scripts/components/health.lua:154-184`：

```154:184:scripts/components/health.lua
function Health:OnSave()
    return
    {
        health = self.currenthealth,
        penalty = self.penalty > 0 and self.penalty or nil,
		maxhealth = self.save_maxhealth and self.maxhealth or nil
    }
end

function Health:OnLoad(data)
	if data.maxhealth ~= nil then
		self.maxhealth = data.maxhealth
	end

    local haspenalty = data.penalty ~= nil and data.penalty > 0 and data.penalty < 1
    if haspenalty then
        self:SetPenalty(data.penalty)
    end

    if data.invincible ~= nil then
        self.invincible = data.invincible
    end
    if data.health ~= nil then
        self:SetVal(data.health, "file_load")
        self:ForceUpdateHUD(true)
    elseif data.percent ~= nil then
        -- used for setpieces!
        -- SetPercent already calls ForceUpdateHUD
        self:SetPercent(data.percent, true, "file_load")
    elseif haspenalty then
        self:ForceUpdateHUD(true)
```

**关键观察**：

1. **OnSave 返回一个 table**——里面是要保存的字段
2. **`penalty > 0 and self.penalty or nil`** —— 默认值不存（节省空间）
3. **`save_maxhealth` 是个 flag** —— 默认情况下 `maxhealth` 不存（用 prefab 默认值），仅当 `save_maxhealth = true` 时才保存
4. **OnLoad 接收 data 参数** —— 是 OnSave 返回的同一个 table
5. **OnLoad 多个分支** —— health / percent / 都没有，分别处理

#### 第二步：health 数据"上链"全流程

```
[保存路径]
[T-3] 玩家退出 → SaveGame
[T-2] 引擎遍历所有 entity，调 GetSaveRecord
[T-1] GetSaveRecord 内部：
       record.data, references = self:GetPersistData()
       (GetPersistData 遍历 components，每个调 OnSave)
[T-0] 最终 record.data.health = { health = 80, penalty = nil, maxhealth = nil }
[T+1] 序列化 record，写入磁盘文件

[加载路径]
[T-3] 玩家进入 → LoadGame
[T-2] 反序列化磁盘 → 拿到所有 record
[T-1] 对每个 record：
       inst = SpawnPrefab(record.prefab)
       inst.Transform:SetPosition(record.x, ...)
       inst:SetPersistData(record.data, newents)
[T-0] SetPersistData 遍历 data：
       cmp.OnLoad(v) for each (k, v) in data
       于是 health 组件的 OnLoad(data.health) 被调用
[T+1] 玩家血量恢复为 80
```

> **核心结论**：**OnSave/OnLoad 是组件级"持久化对儿"**——OnSave 把组件状态序列化为 table，OnLoad 反过来把 table 恢复为状态。**99% 的 mod 自定义组件需要这一对**。

#### 第三步：什么时候不需要写 OnSave / OnLoad？

- **客户端的 replica** —— 数据来自 NetVar，不需要存档
- **运行时缓存** —— 比如 "上次攻击时间"、"动画播放状态"——丢失后能从其他状态推算
- **临时实体** —— `inst.persists = false` 的实体根本不会调 OnSave
- **没有运行时变化的组件** —— 全部是常量配置——直接用 prefab 默认值就行

---

### 10.2.2 快速入门：API 速查

#### 第一步：4 个核心 hook

| Hook | 类型 | 时机 | 参数 | 返回值 |
| --- | --- | --- | --- | --- |
| `Component:OnSave()` | 组件级 | 存档时 | 无 | data 或 (data, references) |
| `Component:OnLoad(data, newents?)` | 组件级 | 加载阶段 1（构造） | data 是同 OnSave 返回值 | 无 |
| `Component:LoadPostPass(newents, data)` | 组件级 | 加载阶段 2（引用解析） | newents 是 GUID 映射表，data 是 OnSave 返回值 | 无 |
| `EntityScript:OnSave(data)` | entity 级 | 存档时 | data 是已收集的组件数据 | references |

#### 第二步：组件 OnSave 的返回值规范

**单返回值**：

```lua
function MyComp:OnSave()
    return { x = 1, y = "abc" }
end
```

**双返回值（含 references）**：

```lua
function MyComp:OnSave()
    return { target_id = self.target.GUID }, { self.target.GUID }
    --      ↑ data                              ↑ references
end
```

**返回 nil**（表示无数据要存）：

```lua
function MyComp:OnSave()
    if not self.dirty then
        return  -- 不保存
    end
    return { ... }
end
```

#### 第三步：组件 OnLoad 的参数

```lua
function MyComp:OnLoad(data, newents)
    if data == nil then return end  -- 关键：永远第一行判 nil
    
    -- data 是 OnSave 当时返回的 table
    self.x = data.x or 0
    self.y = data.y or ""
end
```

**`newents`** 是**全局 entity 映射**（旧 GUID → entity wrapper）——大多组件**不需要它**，少数（如 follower）需要解析引用。

#### 第四步：entity 级 OnSave 的特殊语法

如果你想在 **entity 自己**（不是组件）上保存数据——挂在 prefab fn 里：

```lua
local function fn()
    local inst = CreateEntity()
    -- ...
    
    inst.OnSave = function(inst, data)
        -- data 已经被组件填充过——可以加自定义字段
        data.my_extra_field = inst._some_state
    end
    
    inst.OnLoad = function(inst, data)
        if data.my_extra_field then
            inst._some_state = data.my_extra_field
        end
    end
    
    return inst
end
```

**注意 entity 级 OnSave 的签名**：`function(inst, data)` —— **第二个参数是已收集的组件 data 表** —— 你可以**修改它** —— 而组件级 OnSave 是**返回新 table**。

---

### 10.2.3 快速入门：组件级 vs entity 级 OnSave / OnLoad

#### 第一步：两套机制对比

| 维度 | 组件级 | entity 级 |
| --- | --- | --- |
| 定义位置 | `Component:OnSave()` 是 component 类的方法 | `inst.OnSave = function(inst, data)` 直接挂 entity |
| 数据载体 | 组件返回**新** table | 修改已有的 entity data table |
| 数据键名 | `data[component_name] = ret` | 自由（直接挂 data 顶层） |
| 用途 | 99% 场景 —— 组件持久化 | 1% 场景 —— entity 级特殊状态 |

#### 第二步：组合使用

`entityscript.lua:1903-1937` 显示：**两套都会被调用** —— 组件先调一遍、entity 再调一遍：

```lua
for k, v in pairs(self.components) do
    if v.OnSave then
        local t, refs = v:OnSave()
        if t then data[k] = t end
        ...
    end
end

if self.OnSave then
    local refs = self.OnSave(self, data)  -- 第二个参数是已收集的 data
    ...
end
```

**对开发者影响**：
- 组件的数据进 `data[component_name]`
- entity 级的数据进 `data` 顶层（任意键名）

#### 第三步：典型 entity 级用法

**例 1**：保存"上次砍树时间"——不属于任何组件

```lua
inst.OnSave = function(inst, data)
    data.last_chop_time = inst._last_chop_time
end
```

**例 2**：保存"自定义皮肤变体"

```lua
inst.OnSave = function(inst, data)
    data.skin_variant = inst._skin_variant
end
```

#### 第四步：选哪种？

**默认用组件级**——它有 5 个好处：
1. 数据按组件分类——容易管理
2. 移除组件时数据自动不存
3. add_component_if_missing 兼容机制只对组件级有效
4. LoadPostPass 双阶段只有组件级支持
5. 卸载 mod 时清理简单

**entity 级仅用于**：数据**不属于任何组件**——比如 prefab 自己的"模式标记"、"皮肤变体"等。

---

### 10.2.4 进阶：containers 的"嵌套保存"——用 GetSaveRecord

#### 第一步：容器的特殊问题

容器（箱子、背包）里**装的是物品 entity**——存档时不能只存"物品的 GUID"——因为**物品本身也要被存**：物品的 Transform、AnimState、各组件状态。

**Klei 的做法**：**把物品当成"嵌套 entity"递归保存**。

#### 第二步：Container:OnSave 源码

`scripts/components/container.lua:874-889`：

```874:889:scripts/components/container.lua
function Container:OnSave()
    local data = {items= {}}
    local references = {}
	local refs
    for k,v in pairs(self.slots) do
        if v:IsValid() and v.persists then --only save the valid items
            data.items[k], refs = v:GetSaveRecord()
            if refs then
                for k,v in pairs(refs) do
                    table.insert(references, v)
                end
            end
        end
    end
    return data, references
end
```

**关键观察**：
- **遍历 slots** —— 每个 slot 一个物品
- **`v:IsValid() and v.persists`** —— 只保存有效且 persists=true 的物品
- **`v:GetSaveRecord()`** —— **递归保存物品 entity**（含位置/data/references）
- **references 累积** —— 引用从 sub-record 一路冒泡上来

#### 第三步：Container:OnLoad

```891:900:scripts/components/container.lua
function Container:OnLoad(data, newents)
    if data.items then
        for k,v in pairs(data.items) do
            local inst = SpawnSaveRecord(v, newents)
            if inst then
                self:GiveItem(inst, k)
            end
        end
    end
end
```

**关键**：
- **`SpawnSaveRecord(record, newents)`** —— 反向操作 GetSaveRecord——**递归创建** entity
- **`self:GiveItem(inst, k)`** —— 放回 slot k

#### 第四步：嵌套层级的处理

如果"箱子里有一个箱子"（背包套背包）——递归处理：

```
存档：
  outer_chest:
    GetSaveRecord
    → slots[1] = inner_backpack:GetSaveRecord()
        → slots[1] = log:GetSaveRecord()
        → slots[2] = stone:GetSaveRecord()
    → slots[2] = torch:GetSaveRecord()

加载：
  outer_chest:OnLoad
  → SpawnSaveRecord(inner_backpack record)
      → 递归创建 backpack
      → 调 backpack 的 OnLoad
        → SpawnSaveRecord(log record) → 创建木头
        → SpawnSaveRecord(stone record) → 创建石头
      → 把木头石头塞 backpack
  → 把 backpack 塞 outer_chest
  → SpawnSaveRecord(torch record) → 创建火把
  → 把火把塞 outer_chest
```

**优雅的递归**——支持任意嵌套深度。

---

### 10.2.5 进阶：references 与 GUID 的 follower 完整模式

#### 第一步：Follower:OnSave 真实源码

`scripts/components/follower.lua:416-436`：

```416:436:scripts/components/follower.lua
function Follower:OnSave()
    local data = {}

    local time = GetTime()
    if self.targettime and self.targettime > time then
        data.time = math.floor(self.targettime - time)
    end

    if self.cached_player_leader_userid then
        data.cached_player_leader_userid = self.cached_player_leader_userid

        if self.cached_player_leader_timeleft and self.cached_player_leader_timeleft > time then
            data.cached_player_leader_timeleft = math.floor(self.cached_player_leader_timeleft - time)
        end
    elseif self.leader and self.leader:HasTag("player") and self.leader.userid then
        data.cached_player_leader_userid = self.leader.userid
        data.cached_player_leader_timeleft = data.time
    end

    return not IsTableEmpty(data) and data or nil
end
```

**Follower 没有保存 leader 的 GUID** —— 而是保存**玩家的 userid**。**为什么？**

#### 第二步：玩家 entity 的 GUID 是不稳定的

每次玩家进入服务器——会创建一个新的 player entity——**GUID 变化**。如果 Follower 存了**旧 GUID**——下次加载时找不到。

**但** `userid`（Steam ID 等）是稳定的——重连后还是同一个 userid。

**所以 Follower 用 userid 标记 leader**：

```lua
function Follower:LoadPostPass(newents, savedata)
    if savedata.cached_player_leader_userid then
        -- 找到对应玩家
        local leader = LookupPlayerByUserId(savedata.cached_player_leader_userid)
        if leader then
            self:SetLeader(leader)
        end
    end
end
```

#### 第三步：非玩家引用——用 GUID + references

如果引用的是**普通 entity**（不是玩家）——用 GUID + references：

```lua
function MyComp:OnSave()
    if self.linked_entity and self.linked_entity:IsValid() then
        local refs = { self.linked_entity.GUID }
        return { linked = self.linked_entity.GUID }, refs
    end
end

function MyComp:LoadPostPass(newents, savedata)
    if savedata and savedata.linked and newents[savedata.linked] then
        self.linked_entity = newents[savedata.linked].entity
    end
end
```

#### 第四步：references 的真正作用

**`references` 不只是"标记我引用了谁"**——还有**强制加载**功能：

- 如果你引用的 entity 在世界的另一个角落——**默认它可能不在加载列表里**
- 把 GUID 加进 references —— 引擎**强制加载**这个 entity——保证 LoadPostPass 时 newents 能找到

**没加 references 的常见症状**：你的 mod 的"引用"**有时能加载、有时不能** —— 取决于 leader 是否被其他原因加载了。

---

### 10.2.6 进阶：`OnPreLoad` —— 在 component OnLoad 之前就插队

#### 第一步：OnPreLoad 是什么？

`entityscript.lua:1963-1965`：

```1963:1965:scripts/entityscript.lua
    if self.OnPreLoad ~= nil then
        self:OnPreLoad(data, newents)
    end
```

**时机**：在所有组件 OnLoad **之前**调用。

**用途**：
- 添加缺失的组件（如果存档要求但当前 prefab 没有）
- 修改 data table（在组件读到之前调整数据）
- 紧急兼容老存档

#### 第二步：典型用法——"add_component_if_missing"

`entityscript.lua:1955-1961`：

```1955:1961:scripts/entityscript.lua
	if data and data.add_component_if_missing then
		for k, v in pairs(data) do
			if self.components[k] == nil and type(v) == "table" and v.add_component_if_missing then
				self:AddComponent(k)
			end
		end
	end
```

**Klei 内部机制**：如果 OnSave 返回的数据里有 `add_component_if_missing = true`——加载时会自动给 entity 添加这个组件。

**典型场景**：mod 发了一个版本更新，给某 prefab 加了新组件——老存档里没有这个组件——通过 add_component_if_missing 让加载时自动添加。

#### 第三步：mod 自定义 OnPreLoad

```lua
inst.OnPreLoad = function(inst, data, newents)
    if data == nil then return end
    
    -- 修复老存档：v1 的 data 里 my_field 是 string，v2 改成 number
    if data.mycomp and type(data.mycomp.my_field) == "string" then
        data.mycomp.my_field = tonumber(data.mycomp.my_field) or 0
    end
end
```

**好处**：在组件 OnLoad 拿到 data 之前**先转换格式**——OnLoad 看到的是干净的新格式。

#### 第四步：什么时候用 OnPreLoad 而不是 OnLoad？

**用 OnPreLoad 的场景**：
- 跨组件的数据迁移（一个字段从组件 A 移到组件 B）
- 旧存档结构和新组件结构不匹配
- 需要在组件实例化之前修正数据

**用 OnLoad 的场景**：
- 99% 普通持久化
- 组件内部的版本兼容

---

### 10.2.7 老手进阶：自定义组件完整范例

#### 第一步：完整模板——一个 mod 自定义 "my_skillsystem" 组件

```lua
-- mymod/scripts/components/my_skillsystem.lua

local SkillSystem = Class(function(self, inst)
    self.inst = inst
    
    -- 业务字段
    self.skill_levels = { fire = 0, ice = 0, wind = 0 }
    self.unlocked_skills = {}
    self.cached_target_id = nil  -- 临时缓存：上次解锁某技能的引用 entity
    self.target_entity = nil
end)

--------------------------------------------------------------------------
-- OnSave：把状态序列化
--------------------------------------------------------------------------
function SkillSystem:OnSave()
    -- 永远先 version
    local data = {
        version = 1,
        skill_levels = self.skill_levels,
    }
    
    -- 解锁列表只存数组
    if next(self.unlocked_skills) then
        data.unlocked = {}
        for k, _ in pairs(self.unlocked_skills) do
            table.insert(data.unlocked, k)
        end
    end
    
    -- 引用 entity ——存 GUID + references
    local references
    if self.target_entity and self.target_entity:IsValid() then
        data.target_id = self.target_entity.GUID
        references = { self.target_entity.GUID }
    end
    
    return data, references
end

--------------------------------------------------------------------------
-- OnLoad：恢复状态（不处理引用）
--------------------------------------------------------------------------
function SkillSystem:OnLoad(data, newents)
    if data == nil then return end
    
    -- 版本兼容
    if data.version == nil or data.version == 1 then
        if data.skill_levels then
            self.skill_levels = data.skill_levels
        end
        
        if data.unlocked then
            self.unlocked_skills = {}
            for _, k in ipairs(data.unlocked) do
                self.unlocked_skills[k] = true
            end
        end
        
        -- 缓存 target_id 待 LoadPostPass 处理
        self.cached_target_id = data.target_id
    end
end

--------------------------------------------------------------------------
-- LoadPostPass：解析引用
--------------------------------------------------------------------------
function SkillSystem:LoadPostPass(newents, data)
    if self.cached_target_id and newents[self.cached_target_id] then
        self.target_entity = newents[self.cached_target_id].entity
        self.cached_target_id = nil
    end
end

return SkillSystem
```

#### 第二步：测试存档/加载

```lua
-- 控制台
ThePlayer:AddComponent("my_skillsystem")
ThePlayer.components.my_skillsystem.skill_levels.fire = 5
ThePlayer.components.my_skillsystem.unlocked_skills.fireball = true

-- 触发存档
TheWorld:PushEvent("ms_save")

-- 关闭服务器、重启服务器、进入游戏

-- 加载后查询
print(ThePlayer.components.my_skillsystem.skill_levels.fire)
-- => 5（已恢复）
```

#### 第三步：升级到 v2

未来 mod 加新字段 `mana_pool`：

```lua
function SkillSystem:OnSave()
    return {
        version = 2,                  -- ★
        skill_levels = self.skill_levels,
        unlocked = ...,
        mana_pool = self.mana_pool,    -- 新字段
    }
end

function SkillSystem:OnLoad(data, newents)
    if data == nil then return end
    
    if data.version == nil or data.version == 1 then
        -- v1 → v2 迁移
        self.skill_levels = data.skill_levels or {}
        self.unlocked_skills = data.unlocked and {} or self.unlocked_skills
        if data.unlocked then
            for _, k in ipairs(data.unlocked) do self.unlocked_skills[k] = true end
        end
        self.mana_pool = 100  -- v1 没有这个字段，用默认值
    elseif data.version == 2 then
        -- v2 直接读
        self.skill_levels = data.skill_levels or {}
        self.unlocked_skills = data.unlocked and {} or self.unlocked_skills
        if data.unlocked then
            for _, k in ipairs(data.unlocked) do self.unlocked_skills[k] = true end
        end
        self.mana_pool = data.mana_pool or 100
    end
end
```

---

### 10.2.8 老手进阶：六个常见陷阱

#### 陷阱 1：OnSave 写了 entity / function

**症状**：存档失败或读出来不能用。  
**原因**：lua 序列化只支持基本类型（number/string/bool）+ table——entity 和 function 不可序列化。  
**修复**：永远只存基本类型——entity 用 GUID、function 不要存。

#### 陷阱 2：OnLoad 没判 data 为 nil

**症状**：mod 第一次启用——entity 没存档数据——OnLoad 收到 nil——`data.x` 报错。  
**修复**：永远第一行 `if data == nil then return end`。

#### 陷阱 3：OnSave 返回空 table

**症状**：

```lua
function MyComp:OnSave()
    return {}  -- 空 table
end
```

**问题**：从 `entityscript.lua:1909` 看 —— `if type(t) == "table" and not IsTableEmpty(t) then data[k] = t end` —— 空 table 不会被存——但你想让"组件存在"被知道——失败。  
**修复**：要么不写 OnSave、要么返回真有数据：

```lua
function MyComp:OnSave()
    if self.has_data then
        return { x = self.x }
    end
    -- 返回 nil 即可
end
```

#### 陷阱 4：references 漏写

**症状**：你的 LoadPostPass 里 `newents[id]` 总是 nil——leader 实体没被加载。  
**原因**：OnSave 没返回 references——引擎不知道要保留 leader。  
**修复**：

```lua
return data, { leader_id, target_id }
```

#### 陷阱 5：在 OnLoad 里 spawn 新实体

**症状**：每次重启服务器——你的实体增加了一个新的 child entity。  
**原因**：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    self.child = SpawnPrefab("foo")  -- ❌ 每次加载都 spawn 新的
end
```

**修复**：保存的应该是 child 的 GUID，加载时找已 spawn 的：

```lua
function MyComp:LoadPostPass(newents, data)
    if data and data.child_id and newents[data.child_id] then
        self.child = newents[data.child_id].entity
    end
end
```

或者**完全不持久化** child——下次 spawn 时 prefab fn 重新创建即可。

#### 陷阱 6：用错 OnLoad 参数顺序

```lua
function MyComp:OnLoad(newents, data)  -- ❌ 顺序反了
```

**正确**：`OnLoad(data, newents)` —— data 在前。**LoadPostPass(newents, data) —— newents 在前**。**容易混淆**。

#### 设计经验三条

**经验 ①：data 永远加 version 字段**

第一版就加：

```lua
return { version = 1, ... }
```

后续兼容简单。

**经验 ②：引用解析永远走 LoadPostPass**

不要在 OnLoad 里做实体引用解析——OnLoad 阶段其他实体可能还没 spawn。

**经验 ③：OnSave 数据"小而精"**

只存"恢复必需"的最少字段——加快序列化、减小存档体积、降低升级风险。

---

### 10.2.9 小结

**OnSave/OnLoad 一句话总结**：**OnSave 返回组件状态 table、OnLoad 反向恢复——通过 references + LoadPostPass 处理跨实体引用**。

**速查表**

| 想做的事 | 一行代码 |
| --- | --- |
| 简单存数据 | `function MyComp:OnSave() return {x=self.x} end` |
| 加载数据 | `function MyComp:OnLoad(data) if data then self.x = data.x or 0 end end` |
| 存引用 | `return {id=ent.GUID}, {ent.GUID}` |
| 解引用 | `function MyComp:LoadPostPass(newents, data) ... newents[id].entity ... end` |
| entity 级 OnSave | `inst.OnSave = function(inst, data) data.foo = inst._foo end` |
| OnPreLoad 修复格式 | `inst.OnPreLoad = function(inst, data, newents) ... end` |

**6 个陷阱排雷顺序**

1. OnSave 写 entity/function → 只存基本类型 + GUID
2. OnLoad 没判 data nil → 第一行 if data==nil then return
3. OnSave 返回空 table → 直接返回 nil
4. references 漏写 → return data, {ids}
5. OnLoad 里 spawn 新实体 → 用 LoadPostPass 找已存在的
6. OnLoad / LoadPostPass 参数顺序 → data, newents vs newents, data

**3 条设计经验**

- ① **永远加 version 字段**
- ② **引用解析走 LoadPostPass**
- ③ **OnSave 数据小而精**

> **下一节预告**：10.3 节我们将打开 **persistdata 与世界级别数据持久化** —— TheWorld 自己的存档机制、metadata 怎么存、shard 间持久化数据的协调。读完 10.3，你将能写"世界级"持久化逻辑——比如 mod 自定义全局开关、玩家联机进度统计等。

## 10.3 persistdata 与世界级别数据的持久化

### 本节导读

10.1、10.2 我们讲清了**普通 entity** 怎么持久化——SpawnPrefab + SetPersistData。但**还有几类数据**不挂在 entity 上，需要**世界级 / 全局级**的持久化：

- **TheWorld 自己的组件数据**：季节进度、月相、天气、worldsettings 配置——它们**不属于某个普通 entity**，但必须存
- **world_network 的 NetVar 状态**：客户端可见的"世界状态"
- **shard_network 的跨 shard 数据**：地表/洞穴间共享的元信息
- **完全独立的 mod 持久化文件**：比如 mod 想存"所有玩家通关次数"——和某个具体世界绑定不合适
- **跨服务器持久化**：比如玩家的"成就解锁列表"——所有服务器共享

这一节我们打开**世界级持久化**的底层 API——`TheSim:GetPersistentString` / `TheSim:SetPersistentString` —— 这是 Klei 提供给开发者**直接读写磁盘文件**的接口——加上 `TheWorld:GetPersistData` 这个组件级递归收集机制——构成了"超出普通 entity 范围"的持久化能力。

> **新手**从 10.3.1-10.3.3 起步——理解世界级数据存档结构、TheWorld 组件 OnSave/OnLoad、world_network/shard_network；**进阶读者**继续看 10.3.4-10.3.6，深入 `TheSim:GetPersistentString`、cluster slot 持久化、`generickv` 模式；**老手**跳到 10.3.7-10.3.8，看 mod 实战（全局排行榜数据）+ 6 个最容易踩的坑。

---

### 10.3.1 快速入门：从 SaveGame 看世界级数据结构

#### 第一步：SaveGame 的全景代码

打开 `scripts/mainfunctions.lua:1068-1166`：

```1068:1166:scripts/mainfunctions.lua
function SaveGame(isshutdown, cb)
    if not TheNet:GetIsServer() then
        print("SaveGame disabled for Clients in Don't Starve Together")
        if cb ~= nil then
            cb(true)
        end
        return
    end

    TheNet:StartWorldSave()

    local save = {}
	local savedata_entities = {}

    --print("Saving...")
    --save the entities
    local nument = 0
    local saved_ents = {}
    local references = {}
    for k, v in pairs(Ents) do
        if v.persists and v.prefab ~= nil and v.Transform ~= nil and v.entity:GetParent() == nil and v:IsValid() then
            local record, new_references = v:GetSaveRecord()
            record.prefab = nil
            ...
            table.insert(savedata_entities[v.prefab], record)
            nument = nument + 1
        end
    end

    --save out the map
    save.map =
    {
        tiles = "",
        roads = Roads,
    }

    local new_refs = nil
    local ground = TheWorld
    assert(ground ~= nil, "Cant save world without ground entity")
    if ground ~= nil then
        save.map.prefab = ground.worldprefab
        save.map.tiles = ground.Map:GetStringEncode()
        save.map.world_tile_map = GetWorldTileMap()
        save.map.tiledata = ground.Map:GetDataStringEncode()
        save.map.nav = ground.Map:GetNavStringEncode()
        save.map.nodeidtilemap = ground.Map:GetNodeIdTileMapStringEncode()
        save.map.width, save.map.height = ground.Map:GetSize()
        save.map.topology = ground.topology
        save.map.generated = ground.generated
        save.map.persistdata, new_refs = ground:GetPersistData()
        save.meta = ground.meta
        save.map.hideminimap = ground.hideminimap
		save.map.has_ocean = ground.has_ocean
		...

        local world_network = ground.net
        assert(world_network, "Cant save world without world_network entity")
        if world_network ~= nil then
            save.world_network = {}
            save.world_network.persistdata, new_refs = world_network:GetPersistData()
            ...
        end

        local shard_network = ground.shard -- NOTES(JBK): This data is optional.
        if shard_network ~= nil then
            local persistdata, new_refs = shard_network:GetPersistData()
            if persistdata ~= nil then
                save.shard_network = {}
                save.shard_network.persistdata = persistdata
                ...
            end
        end
    end
```

**关键观察**：

1. **`Ents` 全表遍历** → 每个 persists 的 entity 都序列化到 `savedata_entities`
2. **`ground` 就是 `TheWorld`** → 调它的 `:GetPersistData()` 拿到 map.persistdata
3. **`world_network = ground.net`** → 客户端可见的世界 entity，单独存它的 persistdata
4. **`shard_network = ground.shard`** → 跨 shard entity，**可选**存它的 persistdata

#### 第二步：savedata 顶层的 4 个数据来源

存档反序列化后是个 lua table，**顶层结构**：

```lua
savedata = {
    ents = {                       -- 来自普通 entity 遍历
        ["tree"] = {...},
        ["pigman"] = {...},
        ...
    },
    map = {
        prefab = "forest",
        tiles = "...",              -- 地图编码
        topology = {...},
        persistdata = {              -- ★ TheWorld 的组件 OnSave 数据汇总
            seasons = {progress = 0.5, season = "winter", ...},
            clock = {...},
            worldstate = {...},
            ...
        },
    },
    world_network = {
        persistdata = {              -- ★ world_network 的组件 OnSave
            ...
        },
    },
    shard_network = {
        persistdata = {              -- ★ shard_network 的组件 OnSave（可选）
            shardstate = {...},
        },
    },
    meta = {...},
    snapshot = {...},
}
```

**`map.persistdata` / `world_network.persistdata` / `shard_network.persistdata`** —— 这就是"世界级数据"的存放点。

#### 第三步：和 entity 持久化的对比

| | 普通 entity | TheWorld | world_network | shard_network |
| --- | --- | --- | --- | --- |
| 是否 SpawnPrefab | 是 | 否（独立 prefab world） | 否（特殊客户端镜像） | 否（独立 shard 实体） |
| 数据存放 | `ents[prefab][i].data` | `map.persistdata` | `world_network.persistdata` | `shard_network.persistdata` |
| OnSave 调用 | 组件 + entity 级 | 同 | 同 | 同 |
| OnLoad 调用 | 同 | 同 | 同 | 同 |

**关键**：**世界级数据走的是同一套 OnSave/OnLoad 机制**——只是**调用对象不是普通 entity，而是 TheWorld / TheWorld.net / TheWorld.shard**。

---

### 10.3.2 快速入门：TheWorld 上的组件 OnSave/OnLoad

#### 第一步：TheWorld 的组件特殊性

TheWorld 上挂了一堆"世界级组件"：

- `clock` —— 时钟系统
- `seasons` —— 季节系统
- `worldstate` —— 世界状态快照（见 8.3）
- `weather` —— 天气
- `worldsettings` —— 配置
- `playerspawner` —— 玩家生成
- `birdspawner` / `beargerspawner` 等 —— 各类怪物刷新管理器
- ... 几十个

**每个组件都可以有自己的 OnSave/OnLoad** —— 写法和普通 entity 组件一致。

#### 第二步：worldstate 的 OnSave/OnLoad

打开 `scripts/components/worldstate.lua:331-347`：

```331:347:scripts/components/worldstate.lua
function self:OnSave()
    local data = {}
    for k, v in pairs(self.data) do
        data[k] = v
    end

    return data
end

function self:OnLoad(data)
    for k, v in pairs(data) do
        if self.data[k] ~= nil then
            self.data[k] = v
            print("setting ", k, v)
        end
    end
end
```

**简洁的实现**：
- OnSave：把 `self.data`（含 phase / season / cycles 等约 60 个字段）整个拷贝出来
- OnLoad：反向恢复——但**只恢复 self.data 里已存在的字段** —— 这是兼容性设计（旧存档可能没新字段）

#### 第三步：写一个 mod 自定义"世界级"组件

```lua
-- mymod/scripts/components/myworldstate.lua
local MyWorldState = Class(function(self, inst)
    assert(inst == TheWorld, "MyWorldState should be added to TheWorld!")
    self.inst = inst
    self.player_kills = {}      -- 各玩家的击杀数
    self.global_event_count = 0
end)

function MyWorldState:RecordKill(playername)
    self.player_kills[playername] = (self.player_kills[playername] or 0) + 1
    self.global_event_count = self.global_event_count + 1
end

function MyWorldState:OnSave()
    return {
        version = 1,
        player_kills = self.player_kills,
        global_event_count = self.global_event_count,
    }
end

function MyWorldState:OnLoad(data)
    if data == nil then return end
    if data.version == 1 then
        self.player_kills = data.player_kills or {}
        self.global_event_count = data.global_event_count or 0
    end
end

return MyWorldState
```

**挂到 TheWorld**：

```lua
-- modmain.lua
AddPrefabPostInit("world", function(inst)
    if inst.ismastersim then
        inst:AddComponent("myworldstate")
    end
end)
```

**测试**：

```lua
-- 控制台
TheWorld.components.myworldstate:RecordKill("wilson")
TheWorld:PushEvent("ms_save")  -- 触发存档
-- 重启服务器后
print(TheWorld.components.myworldstate.global_event_count)
-- => 1（已恢复）
```

---

### 10.3.3 快速入门：world_network、shard_network 的 persistdata

#### 第一步：world_network 是什么？

`scripts/prefabs/world.lua` 创建 TheWorld 时——同时创建一个 `world_network` 实体——挂在 `TheWorld.net`。

**作用**：
- TheWorld 是**主世界 entity**——一些状态（client_postinit 跑不到的、客户端不需要看到的）只在主世界
- TheWorld.net 是**镜像 entity**——专门挂"客户端也要看的世界级 NetVar"

**典型组件**：
- `clock`（时钟同步给客户端）
- `worldstate`（部分状态）
- `seasons`（季节同步）

#### 第二步：world_network 的 OnSave 调用

回看 `mainfunctions.lua:1140-1151`：

```1140:1151:scripts/mainfunctions.lua
        local world_network = ground.net
        assert(world_network, "Cant save world without world_network entity")
        if world_network ~= nil then
            save.world_network = {}
            save.world_network.persistdata, new_refs = world_network:GetPersistData()

            if new_refs ~= nil then
                for k, v in pairs(new_refs) do
                    references[v] = world_network
                end
            end
        end
```

**和 TheWorld 用同一套 GetPersistData**——独立持久化为 `save.world_network.persistdata`。

#### 第三步：shard_network 的特殊性

```1153:1165:scripts/mainfunctions.lua
        local shard_network = ground.shard -- NOTES(JBK): This data is optional.
        if shard_network ~= nil then
            local persistdata, new_refs = shard_network:GetPersistData()
            if persistdata ~= nil then
                save.shard_network = {}
                save.shard_network.persistdata = persistdata
                if new_refs ~= nil then
                    for k, v in pairs(new_refs) do
                        references[v] = shard_network
                    end
                end
            end
        end
```

**关键差异**：

- **`-- NOTES(JBK): This data is optional.`** —— **可能为 nil**，单 shard 模式下 shard_network 可能不存在
- **专门承载跨 shard 数据** —— 比如**"地表玩家的解锁状态"** 同步到洞穴

#### 第四步：mod 给 world_network / shard_network 挂组件

```lua
-- modmain.lua
AddPrefabPostInit("forest_network", function(inst)
    -- forest_network 就是 world_network 的具体 prefab
    if inst.ismastersim then
        inst:AddComponent("mynetworkdata")
    end
end)
```

**注意**：`world_network` 的 prefab 名是 `forest_network`（地表）/ `cave_network`（洞穴）等——按地图类型有不同名称。

**`shard_network`** 的 prefab 名通常是 `shard_network`。

---

### 10.3.4 进阶：`TheSim:GetPersistentString` —— mod 自己的"独立持久化文件"

#### 第一步：API 速查

| API | 用途 |
| --- | --- |
| `TheSim:SetPersistentString(name, data, encode, callback)` | 写文件 |
| `TheSim:GetPersistentString(name, callback)` | 读文件 |
| `TheSim:ErasePersistentString(name, callback)` | 删文件 |

**特点**：
- **完全独立于 SaveGame 流程** —— 不会写入 session 文件
- **跨 session 持久** —— 重启服务器后仍存在
- **跨世界共享** —— 不同世界（地表 / 洞穴）都能读到同一份

#### 第二步：典型用法

```lua
-- 写
local data = json.encode({ score = 100, kills = 50 })
TheSim:SetPersistentString("mymod_savefile", data, false, function(success)
    if success then
        print("save succeeded")
    end
end)

-- 读
TheSim:GetPersistentString("mymod_savefile", function(success, data)
    if success and data then
        local decoded = json.decode(data)
        print(decoded.score)  -- 100
    end
end)
```

#### 第三步：编码参数

`SetPersistentString(name, data, encode, callback)` 的 **encode** 参数：

- **`false`** —— 普通字符串
- **`true`** —— 加密 + base64 编码（适合敏感数据）

**对开发者影响**：
- 大多 mod 用 `false`
- 想防玩家直接修改存档文件用 `true`

#### 第四步：异步回调的注意事项

`SetPersistentString` / `GetPersistentString` **都是异步**：

```lua
-- ❌ 错误期望
TheSim:GetPersistentString("xx", function(s, d) print(d) end)
print("hello")  -- "hello" 先打印，data 后面才回调

-- ✓ 正确链式
TheSim:GetPersistentString("xx", function(s, d)
    if s then
        process_data(d)
        TheSim:SetPersistentString("xx", encode(...), false)
    end
end)
```

#### 第五步：与 SaveGame 的协调

**注意**：mod 自己写的 PersistentString **不会随 SaveGame 一起写盘**——所以**写入时机要自己掌握**：

- 数据更新时**立刻** Set
- 服务器关闭时调一次（用 ms_save 事件 hook）
- 定期 Set 备份（DoPeriodicTask）

---

### 10.3.5 进阶：cluster slot 持久化（跨 shard 的 KV 存储）

#### 第一步：cluster 模式下的特殊存储

dedicated 服务器的 cluster 模式 —— 一个集群有多个 shard（地表 + 洞穴）—— 每个 shard 有**自己的存档目录**——但**有些数据需要跨 shard 共享**——比如"地表玩家解锁了空中堡垒、通知洞穴 boss 改变状态"。

**Klei 提供了"cluster slot"持久化**：

```lua
TheSim:SetPersistentStringInClusterSlot(slot, shard, filename, str, encode, cb)
TheSim:GetPersistentStringInClusterSlot(slot, shard, filename, cb)
```

**参数**：
- `slot` —— 存档槽位（用户的存档槽）
- `shard` —— 目标 shard（"Master" / "Caves" 或 nil 表示集群级）
- `filename` —— 文件名
- `str` —— 内容

#### 第二步：典型用法

```lua
-- 在地表 shard 写一个集群级文件
TheSim:SetPersistentStringInClusterSlot(
    Settings.save_slot,  -- 当前存档槽位
    nil,                  -- nil = 整个集群共享
    "mymod_global_unlock", 
    json.encode({ unlocked = {"sky_castle", "deep_dungeon"} }),
    false
)

-- 在洞穴 shard 读这个文件
TheSim:GetPersistentStringInClusterSlot(
    Settings.save_slot, 
    nil, 
    "mymod_global_unlock",
    function(success, data)
        if success then
            local decoded = json.decode(data)
            -- 知道地表已解锁哪些
        end
    end
)
```

#### 第三步：对比表

| API | 作用域 | 单机版 | dedicated cluster |
| --- | --- | --- | --- |
| `TheSim:SetPersistentString` | 当前 shard | ✓ | 仅当前 shard |
| `TheSim:SetPersistentStringInClusterSlot(_, "Master", _, _, _, _)` | 指定 shard | 同上 | 写指定 shard 目录 |
| `TheSim:SetPersistentStringInClusterSlot(_, nil, _, _, _, _)` | 集群级 | 同上 | 写集群根目录（所有 shard 共享） |

#### 第四步：跨 shard 协调的另一种方法——ShardRPC

**ShardRPC（9.2.5 节）**走"运行时通信"——但**关闭服务器后不持久**。

**cluster slot 持久化**走"持久存储"——重启后仍在。

**两套配合**：
- 运行时事件触发 → ShardRPC 通知对方 shard
- 重启后恢复 → cluster slot 文件读取
- 两层冗余 → 保证数据一致

---

### 10.3.6 进阶：`generickv` 模式——跨 session/服务器的持久数据

#### 第一步：generickv 是什么？

`scripts/generickv.lua` 是 Klei 自己实现的"通用 KV 存储"——**用于成就系统等需要"跨服务器同步到云"的数据**。

打开它：

```45:59:scripts/generickv.lua
function GenericKV:Load()
    --print("[GenericKV] Load")
    self.kvs = {}
    TheSim:GetPersistentString("generickv", function(load_success, data)
        if load_success and data ~= nil then
            local status, generickv_data = pcall(function() return json.decode(data) end)
            if status and generickv_data then
                self.kvs = generickv_data.kvs
                self.loaded = true
            else
                print("Failed to load the data in generickv!", status, generickv_data)
            end
        end
    end)
end
```

**就一个 GetPersistentString + json.decode** —— 简单的"全局单文件存储"。

#### 第二步：使用场景

- 玩家成就解锁列表
- 玩家 UI 设置（音量、字体大小、键位）
- mod 全局配置（不针对单个存档）

#### 第三步：和"存档绑定"的对比

| 数据类型 | 用什么 |
| --- | --- |
| 跟当前存档绑定（这局游戏的状态）| 普通 OnSave/OnLoad（10.2 节） |
| 跟 mod 配置绑定（这局/那局都用）| `TheSim:SetPersistentString` |
| 跟玩家身份绑定（成就/解锁）| `generickv` 风格（持久 KV）|
| 跨服务器云同步 | `TheInventory` 提供的接口（不展开）|

#### 第四步：mod 自己写一个 KV 存储

```lua
-- mymod/scripts/myglobalstore.lua
local MyGlobalStore = {}
local data = {}
local loaded = false

function MyGlobalStore:Set(key, value)
    data[key] = value
    self:Save()
end

function MyGlobalStore:Get(key)
    return data[key]
end

function MyGlobalStore:Save()
    TheSim:SetPersistentString("mymod_globalstore", json.encode({ data = data }), false)
end

function MyGlobalStore:Load(callback)
    TheSim:GetPersistentString("mymod_globalstore", function(success, str)
        if success and str then
            local ok, decoded = pcall(json.decode, str)
            if ok and decoded then
                data = decoded.data or {}
            end
        end
        loaded = true
        if callback then callback() end
    end)
end

return MyGlobalStore
```

**用法**：

```lua
-- modmain.lua
local Store = require("myglobalstore")

-- mod 启动时加载
AddSimPostInit(function()
    Store:Load(function()
        print("loaded:", Store:Get("achievements"))
    end)
end)

-- 业务里使用
Store:Set("achievements", { "kill_boss", "build_castle" })
```

---

### 10.3.7 老手进阶：mod 实战——全局排行榜数据持久化

#### 第一步：场景

mod 想做一个"全局排行榜"——记录每个玩家在所有游戏会话累计的击杀数 / 建造数。**这种数据**：
- 不属于当前世界（所有世界共享）
- 玩家进入服务器才有效（按 userid 索引）
- 关闭服务器后保留
- 不需要跟 ents 一起存

#### 第二步：完整实现

```lua
-- mymod/scripts/myleaderboard.lua
local Leaderboard = {}
local data = {}  -- userid → { kills = X, builds = Y, ... }
local dirty = false
local loaded = false

local FILENAME = "mymod_leaderboard"

local function Save()
    if not loaded then return end
    TheSim:SetPersistentString(FILENAME, json.encode({
        version = 1,
        data = data,
    }), false)
    dirty = false
end

local function Load(callback)
    TheSim:GetPersistentString(FILENAME, function(success, str)
        if success and str then
            local ok, decoded = pcall(json.decode, str)
            if ok and decoded then
                if decoded.version == 1 then
                    data = decoded.data or {}
                end
            end
        end
        loaded = true
        if callback then callback() end
    end)
end

function Leaderboard:RecordKill(player)
    if not loaded then return end
    if not player.userid then return end
    
    if data[player.userid] == nil then
        data[player.userid] = { kills = 0, builds = 0, name = player:GetDisplayName() }
    end
    data[player.userid].kills = data[player.userid].kills + 1
    dirty = true
end

function Leaderboard:RecordBuild(player)
    if not loaded then return end
    if not player.userid then return end
    
    if data[player.userid] == nil then
        data[player.userid] = { kills = 0, builds = 0, name = player:GetDisplayName() }
    end
    data[player.userid].builds = data[player.userid].builds + 1
    dirty = true
end

function Leaderboard:GetTop(N)
    local sorted = {}
    for uid, d in pairs(data) do
        table.insert(sorted, { uid = uid, name = d.name, score = d.kills + d.builds })
    end
    table.sort(sorted, function(a, b) return a.score > b.score end)
    
    local result = {}
    for i = 1, math.min(N, #sorted) do
        table.insert(result, sorted[i])
    end
    return result
end

function Leaderboard:Init()
    Load()
    
    -- 监听玩家行为
    AddSimPostInit(function()
        TheWorld:ListenForEvent("entity_death", function(world, data)
            if data.afflicter and data.afflicter:HasTag("player") then
                Leaderboard:RecordKill(data.afflicter)
            end
        end)
        
        TheWorld:ListenForEvent("buildstructure", function(world, data)
            if data.builder then  -- 这个事件实际是 player 推的
                Leaderboard:RecordBuild(data.builder)
            end
        end)
    end)
    
    -- 定期保存
    AddSimPostInit(function()
        TheWorld:DoPeriodicTask(60, function()
            if dirty then
                Save()
            end
        end)
    end)
    
    -- 服务器关闭时立即保存
    AddSimPostInit(function()
        TheWorld:ListenForEvent("ms_save", Save)
    end)
end

return Leaderboard
```

**用法**：

```lua
-- modmain.lua
local Leaderboard = require("myleaderboard")
Leaderboard:Init()

-- 控制台查询
print(json.encode(Leaderboard:GetTop(10)))
```

#### 第三步：数据安全考量

**问题 1**：磁盘写入失败怎么办？  
**修复**：`SetPersistentString` 的 callback 检查 success 标志：

```lua
TheSim:SetPersistentString(FILENAME, str, false, function(success)
    if not success then
        print("Failed to save leaderboard, retrying in 30s...")
        TheWorld:DoTaskInTime(30, Save)
    end
end)
```

**问题 2**：玩家伪造数据怎么办？  
**修复**：服务端**只存 server-authority 的数据**——不接受客户端 RPC 直接写排行榜——只通过事件触发。

**问题 3**：数据结构未来要升级？  
**修复**：永远 `version` 字段——见 10.5 节。

---

### 10.3.8 老手进阶：六个常见陷阱

#### 陷阱 1：以为 `TheWorld:OnSave` 自动调用

**症状**：mod 给 TheWorld 加了 OnSave 函数——但**根本不被调用**。  
**原因**：TheWorld 的 OnSave 数据不是直接挂 OnSave 字段——而是通过 `GetPersistData` 收集**组件**的 OnSave。  
**修复**：把数据放进**世界级组件**：

```lua
AddPrefabPostInit("world", function(inst)
    if inst.ismastersim then
        inst:AddComponent("myworldstate")
    end
end)
```

#### 陷阱 2：mod 自定义文件名和 Klei 内置文件冲突

**症状**：调 SetPersistentString("save")—— 真存档文件被覆盖。  
**修复**：永远加 mod 前缀：`"mymod_xxx"`。

#### 陷阱 3：在客户端调 SetPersistentString

**症状**：客户端调用——本地写文件 —— 但联机模式下**根本无意义**——客户端的"持久数据"通常应该走服务端。  
**修复**：判 `if TheNet:GetIsServer()` 或者**用客户端配置**（profile 系统）。

#### 陷阱 4：异步加载——Init 时直接读

**症状**：

```lua
local data = nil
TheSim:GetPersistentString("mymod_data", function(s, d) data = json.decode(d) end)
print(data)  -- 还是 nil（异步未完成）
```

**修复**：永远在 callback 里处理——或用"loaded" flag。

#### 陷阱 5：定期保存 dirty flag 用错

**症状**：每 60 秒强制保存——即使没变化也写盘。  
**修复**：设置 `dirty = true` 仅在数据**真正修改**时——保存前判断。

#### 陷阱 6：`SetPersistentString` 写入大文件

**症状**：mod 存了 10 MB 的"详细日志"—— 每次保存都阻塞——磁盘 I/O 负担大。  
**修复**：
- 只存"必需"
- 大数据分文件
- 增量保存 vs 全量保存

#### 设计经验三条

**经验 ①：选择存储位置很重要**

```
当前世界绑定 → OnSave/OnLoad（10.2 节）
TheWorld 级（跟世界一起存） → 世界级组件 + OnSave/OnLoad
跨世界 / 跨 session → TheSim:SetPersistentString
跨 shard → cluster slot
跨服务器 → generickv 风格
```

**经验 ②：异步意识**

`TheSim:GetPersistentString` 是异步——所有"读取后处理"的逻辑都要在 callback 里——**不能假定立刻有数据**。

**经验 ③：数据备份策略**

mod 重要数据（排行榜等）应该**定期备份**：

```lua
-- 每 10 分钟存一次"备份文件"
TheWorld:DoPeriodicTask(600, function()
    TheSim:SetPersistentString("mymod_backup_" .. tostring(os.time()), serialize_all_data(), false)
end)
```

避免主存档损坏时全部丢失。

---

### 10.3.9 小结

**世界级持久化一句话总结**：**TheWorld / world_network / shard_network 的组件 OnSave/OnLoad 用同一套机制；mod 跨世界数据走 `TheSim:SetPersistentString`；跨 shard 走 cluster slot；跨服务器走 generickv**。

**速查表**

| 数据归属 | 存储方式 |
| --- | --- |
| 普通 entity 数据 | 组件 OnSave/OnLoad |
| TheWorld 自身 | 挂世界级组件 + 该组件的 OnSave/OnLoad |
| 客户端可见的世界级 | world_network 上的组件 |
| 跨 shard 共享的世界级 | shard_network 上的组件 |
| 跨世界 mod 数据 | `TheSim:SetPersistentString("mymod_xxx", ...)` |
| 跨 shard mod 数据 | `TheSim:SetPersistentStringInClusterSlot(slot, nil, ...)` |
| 跨服务器（玩家身份） | `generickv` 风格自建 |

**6 个陷阱排雷顺序**

1. TheWorld:OnSave 自动调用 → 用世界级组件
2. 文件名冲突 → 加 mod 前缀
3. 客户端调 SetPersistentString → 判 GetIsServer
4. 异步当同步用 → 用 callback / loaded flag
5. dirty flag 用错 → 真修改才置 true
6. 大文件 → 分割 / 增量

**3 条设计经验**

- ① **选对存储位置**：按数据归属选机制
- ② **永远异步意识**：用 callback / loaded flag
- ③ **数据备份**：重要数据定期备份多份

> **下一节预告**：10.4 节我们将专门讲 **Mod 自定义数据的正确存取方式** —— 整合 10.1 ~ 10.3 学到的所有持久化机制——给出 mod 各种场景的最佳实践模板。读完 10.4，你的 mod 就能"按数据特性自动选对存储方式"——**告别"想存数据但不知道存哪"**的纠结。


## 10.4 Mod 自定义数据的正确存取方式

### 本节导读

10.1 ~ 10.3 我们把饥荒持久化系统的所有"机制"都讲过了——`OnSave/OnLoad`、`persistdata`、`TheSim:SetPersistentString`、`generickv`、cluster slot——但**每次写 mod 时**——你都会遇到这个**两难选择**：

> "我这条数据该存哪？"

举几个真实的 mod 场景：
- 玩家**击杀某 boss 的次数** → 跟玩家走、跟服务器走、跟存档走？
- mod 的**全局开关**（"开启实验功能"）→ 跟当前世界走、跟整个 mod 走？
- 玩家在**自定义 mod UI 里设置的字号** → 跟存档绑、跟 mod 配置绑、跟玩家本机绑？
- 自定义 boss 的**重生倒计时** → 跟 boss 实体绑、跟世界绑、跟 mod 配置绑？

**选错了就是 bug**——比如把"全局开关"存到 entity OnSave 里——卸载 mod 后玩家在另一个存档里又看到开关——困惑。**或者把"玩家身份信息"存到 entity OnSave**——entity 销毁后数据消失——下次进入复活了却没了——丢失。

这一节我们把 10.1 ~ 10.3 学到的**所有持久化机制综合**——做成一张**清晰的决策图**：**按数据特性自动选对存储方式**——读完 10.4，你写 mod 时**永远不再纠结"存哪"**。

> **新手**从 10.4.1-10.4.3 起步——理解 mod 数据的 5 大类型、按数据特性的决策树、5 个标准模板；**进阶读者**继续看 10.4.4-10.4.6，深入 mod 配置 vs 玩家数据 vs 世界数据三类的边界、客户端配置的特殊处理、组件级 + entity 级数据的协调；**老手**跳到 10.4.7-10.4.8，看完整 mod 案例（RPG 角色养成）+ 6 个最容易踩的坑。

---

### 10.4.1 快速入门：mod 数据的 5 大类型

#### 第一步：按"归属"分类

mod 想存的数据，按"**归属对象**"可以分成 5 大类：

| 类型 | 归属 | 例子 |
| --- | --- | --- |
| 1️⃣ **实体级** | 某个特定 entity（树、玩家、容器） | 树的"剩余可砍次数"、玩家自定义 buff |
| 2️⃣ **世界级** | 当前世界（这局游戏） | 当前世界的 boss 重生计数、世界级开关 |
| 3️⃣ **集群级** | 跨 shard 的世界群 | 地表/洞穴共享的解锁状态 |
| 4️⃣ **玩家级** | 跟玩家身份（userid）走 | 玩家的成就、玩家的 mod 进度 |
| 5️⃣ **mod 级** | 跟 mod 走（不绑特定世界/玩家）| mod 的全局配置、玩家本机偏好 |

#### 第二步：每类对应的存储机制

| 类型 | 存储机制 | 章节 |
| --- | --- | --- |
| 1️⃣ 实体级 | 组件 OnSave/OnLoad | 10.2 |
| 2️⃣ 世界级 | TheWorld 组件 OnSave/OnLoad（含 world_network） | 10.3.2-10.3.3 |
| 3️⃣ 集群级 | shard_network 组件 + cluster slot 文件 | 10.3.5 |
| 4️⃣ 玩家级 | 玩家 entity 上的组件 OnSave；或 generickv | 10.2 + 10.3.6 |
| 5️⃣ mod 级 | `TheSim:SetPersistentString` | 10.3.4 |

#### 第三步：决策的核心问题

**问 1**：这条数据**和谁绑定**？
- 和某个 entity 一起销毁就消失 → 实体级
- 跟整局游戏走 → 世界级
- 跟玩家身份走 → 玩家级
- 跟 mod 走 → mod 级

**问 2**：这条数据**何时失效**？
- entity 销毁 → 实体级
- 当前世界结束 → 世界级
- 玩家退出 → 玩家级
- mod 卸载 → mod 级

**问 3**：这条数据**谁可以读**？
- 仅一个 entity 内部 → 实体级
- 整个世界 → 世界级
- 跨 shard 共享 → 集群级
- 跨服务器共享 → 玩家级 / mod 级

---

### 10.4.2 快速入门：按数据特性的决策树

#### 完整决策树

```
mod 想存的某条数据
│
├── 这条数据"绑在某个 entity 上"吗？
│   ├── 是 → 用 component OnSave/OnLoad（10.2）
│   └── 否 → 进入下一步
│
├── 这条数据"跟当前世界（这局游戏）绑定"吗？
│   ├── 是 → 是不是要客户端也能看？
│   │   ├── 是 → 用 world_network 上的组件（10.3.3）
│   │   └── 否 → 用 TheWorld 上的组件（10.3.2）
│   └── 否 → 进入下一步
│
├── 这条数据"需要跨 shard 共享"吗？
│   ├── 是 → 是不是只需启动时读一次？
│   │   ├── 是 → cluster slot 文件（10.3.5）
│   │   └── 否（频繁同步）→ shard_network 组件 + ShardRPC
│   └── 否 → 进入下一步
│
├── 这条数据"跟玩家身份（userid）绑定"吗？
│   ├── 是 → 是不是跨服务器共享？
│   │   ├── 是 → 自建 generickv（10.3.6）
│   │   └── 否 → 玩家 entity 上的 component OnSave
│   └── 否 → 进入下一步
│
└── 这条数据"跟 mod 自身绑定"
    └── 用 TheSim:SetPersistentString（10.3.4）
```

#### 用 5 个真实场景验证决策树

**场景 1**：自定义"树自身的可砍次数"
- 绑 entity 吗？ ✓
- → 用组件 OnSave/OnLoad

**场景 2**：当前世界的"boss 击杀计数"
- 绑 entity 吗？ ✗
- 绑当前世界吗？ ✓
- 客户端要看吗？通常 ✓（HUD 显示）
- → 用 world_network 组件

**场景 3**：mod 的"全局实验功能开关"
- 绑 entity 吗？ ✗
- 绑当前世界吗？ ✗（卸载到下一局也想保持）
- 跨 shard 吗？ ✗
- 绑玩家身份吗？ ✗（所有玩家共享）
- → mod 级，用 SetPersistentString

**场景 4**：玩家"自定义键位偏好"
- 绑 entity 吗？ ✗
- 绑世界吗？ ✗（不跟存档走）
- 绑玩家吗？ ✓
- 跨服务器吗？ ✓（在朋友的服务器也用同样键位）
- → 自建 generickv

**场景 5**：地表 → 洞穴共享的"已激活的传送门列表"
- 绑 entity 吗？ ✗
- 绑当前世界吗？ ✓（这局游戏）
- 跨 shard 吗？ ✓
- 频繁同步吗？ ✗（玩家激活一个 → 通知一次）
- → cluster slot 文件 + ShardRPC 通知

---

### 10.4.3 快速入门：5 个标准模板

#### 模板 1：实体级数据（组件 OnSave/OnLoad）

```lua
-- mymod/scripts/components/myentitydata.lua
local MyEntityData = Class(function(self, inst)
    self.inst = inst
    self.value = 0
end)

function MyEntityData:OnSave()
    return { version = 1, value = self.value }
end

function MyEntityData:OnLoad(data)
    if data == nil then return end
    if data.version == 1 then
        self.value = data.value or 0
    end
end

return MyEntityData
```

```lua
-- modmain.lua
AddPrefabPostInit("某 prefab", function(inst)
    if TheWorld.ismastersim then
        inst:AddComponent("myentitydata")
    end
end)
```

#### 模板 2：世界级数据（TheWorld 组件）

```lua
-- mymod/scripts/components/myworlddata.lua
local MyWorldData = Class(function(self, inst)
    assert(inst == TheWorld)
    self.inst = inst
    self.boss_kill_count = 0
end)

function MyWorldData:RecordBossKill()
    self.boss_kill_count = self.boss_kill_count + 1
end

function MyWorldData:OnSave()
    return { version = 1, boss_kill_count = self.boss_kill_count }
end

function MyWorldData:OnLoad(data)
    if data == nil then return end
    if data.version == 1 then
        self.boss_kill_count = data.boss_kill_count or 0
    end
end

return MyWorldData
```

```lua
-- modmain.lua
AddPrefabPostInit("world", function(inst)
    if inst.ismastersim then
        inst:AddComponent("myworlddata")
    end
end)
```

#### 模板 3：客户端可见的世界级数据（world_network 组件）

```lua
-- mymod/scripts/components/myworldnetdata.lua（自定义组件）
local MyWorldNetData = Class(function(self, inst)
    self.inst = inst
    self.boss_phase = net_byte(inst.GUID, "myworld.boss_phase", "bossphasedirty")
end)

function MyWorldNetData:SetBossPhase(p)
    self.boss_phase:set(p)
end

function MyWorldNetData:GetBossPhase()
    return self.boss_phase:value()
end

function MyWorldNetData:OnSave()
    return { boss_phase = self.boss_phase:value() }
end

function MyWorldNetData:OnLoad(data)
    if data == nil then return end
    if data.boss_phase then
        self.boss_phase:set(data.boss_phase)
    end
end

return MyWorldNetData
```

```lua
-- modmain.lua
AddReplicableComponent("myworldnetdata")
AddPrefabPostInit("forest_network", function(inst)  -- 注意是 forest_network 不是 world
    inst:AddComponent("myworldnetdata")
end)
AddPrefabPostInit("cave_network", function(inst)
    inst:AddComponent("myworldnetdata")
end)
```

#### 模板 4：玩家级数据（自建 generickv）

```lua
-- mymod/scripts/myplayerdata.lua
local PlayerData = {}
local data = {}  -- userid → { ... }
local loaded = false

local FILENAME = "mymod_playerdata"

function PlayerData:Load(callback)
    TheSim:GetPersistentString(FILENAME, function(success, str)
        if success and str then
            local ok, decoded = pcall(json.decode, str)
            if ok and decoded then
                data = decoded.data or {}
            end
        end
        loaded = true
        if callback then callback() end
    end)
end

function PlayerData:Save()
    if not loaded then return end
    TheSim:SetPersistentString(FILENAME, json.encode({
        version = 1,
        data = data,
    }), false)
end

function PlayerData:GetPlayer(userid)
    if not loaded then return nil end
    return data[userid]
end

function PlayerData:SetPlayerField(userid, field, value)
    if not loaded then return end
    data[userid] = data[userid] or {}
    data[userid][field] = value
    self:Save()
end

return PlayerData
```

```lua
-- modmain.lua
local PlayerData = require("myplayerdata")

AddSimPostInit(function()
    PlayerData:Load(function()
        print("PlayerData loaded")
    end)
end)

-- 玩家加入时初始化
AddPlayerPostInit(function(player)
    if TheWorld.ismastersim and player.userid then
        local pd = PlayerData:GetPlayer(player.userid)
        if pd then
            -- 应用持久数据
            player.components.health:SetMaxHealth(pd.maxhp or 150)
        end
    end
end)
```

#### 模板 5：mod 级配置（SetPersistentString）

```lua
-- mymod/scripts/myconfig.lua
local Config = {}
local data = {}
local loaded = false

local FILENAME = "mymod_config"

function Config:Load(callback)
    TheSim:GetPersistentString(FILENAME, function(success, str)
        if success and str then
            local ok, decoded = pcall(json.decode, str)
            if ok and decoded then
                data = decoded.data or {}
            end
        end
        loaded = true
        if callback then callback() end
    end)
end

function Config:Save()
    if not loaded then return end
    TheSim:SetPersistentString(FILENAME, json.encode({
        version = 1,
        data = data,
    }), false)
end

function Config:Get(key, default)
    if not loaded then return default end
    return data[key] or default
end

function Config:Set(key, value)
    if not loaded then return end
    data[key] = value
    self:Save()
end

return Config
```

---

### 10.4.4 进阶：mod 配置 vs 玩家数据 vs 世界数据

#### 第一步：三类数据的边界

| 维度 | mod 配置 | 玩家数据 | 世界数据 |
| --- | --- | --- | --- |
| 跨世界（不同存档）| 共享 | 部分共享（按 userid）| 不共享 |
| 跨服务器 | 共享（仅本机）| 取决于实现 | 不共享 |
| 跟谁走 | mod 自身 | 玩家 userid | 当前存档 |
| 失效条件 | mod 卸载 | 玩家更换账号 | 存档删除 |
| 写入位置 | TheSim PersistentString | 玩家 component / KV | TheWorld 组件 |

#### 第二步：常见混淆场景

**场景 A**：mod 想做"玩家累计游戏时间"

- 跟某个 entity 走？✗（玩家会复活、player entity 会换）
- 跟当前世界走？✗（换存档应该继续累计）
- 跟玩家走？✓（按 userid）
- → **玩家级数据**

**场景 B**：mod 想做"世界出生时刻的随机种子"

- 跟当前世界走？✓（每局生成不同）
- 跟玩家走？✗
- → **世界级数据**

**场景 C**：mod 想做"显示伤害数字的开关"

- 跟世界走？✗（每局都要点一遍开关 = 反人类）
- 跟玩家走？取决于 mod 设计
- 跟 mod 走？✓（最简单，影响所有用户）
- → **mod 级数据**（但通常用 mod 配置菜单管理，不需要持久化文件）

#### 第三步：mod 配置菜单 vs 持久化文件

**Klei 提供的"mod 配置菜单"**（`modinfo.lua` 的 `configuration_options`）—— 玩家可以在游戏前配置 —— **数据存在 mod 设置文件**——和 mod 一起加载。

**对比**：

| 用途 | 用什么 |
| --- | --- |
| 游戏前由玩家配置（"难度"、"频率"等） | mod 配置菜单（modinfo.lua） |
| 运行时由 mod 自动写入（"上次游戏时间"） | TheSim:SetPersistentString |

> **设计经验**：能用 modinfo 配置就不要用 SetPersistentString —— 玩家有 UI 可视化、设置可保留。

---

### 10.4.5 进阶：客户端配置（用户偏好）的特殊处理

#### 第一步：客户端独占数据

某些 mod 数据**只属于客户端**：
- HUD 元素位置（玩家拖动了 mod UI）
- 客户端音量偏好
- mod UI 的展开/收起状态

**这些**不应该放服务端——也不应该跟存档绑定——它们是**本机用户偏好**。

#### 第二步：客户端 SetPersistentString

`TheSim:SetPersistentString` **在客户端也能用** —— 写入客户端本地的存档目录。

```lua
-- 客户端代码（modmain 早期 / widget 内）
TheSim:SetPersistentString("mymod_uipos", json.encode({x=100, y=200}), false)

TheSim:GetPersistentString("mymod_uipos", function(s, str)
    if s and str then
        local pos = json.decode(str)
        widget:SetPosition(pos.x, pos.y)
    end
end)
```

#### 第三步：客户端 vs 服务端的判断

```lua
local function SaveMyData(...)
    if TheNet:GetIsServer() then
        -- 服务端逻辑
    else
        -- 客户端逻辑
        TheSim:SetPersistentString("mymod_clientdata", ...)
    end
end
```

#### 第四步：联机模式下两端的数据隔离

**联机时**：
- 服务端的 SetPersistentString 写到**服务端机器的磁盘**
- 客户端的 SetPersistentString 写到**客户端机器的磁盘**
- **两边数据完全隔离**

**对开发者影响**：mod 想"客户端的某偏好同步给服务端"——必须**用 RPC 主动发送**——不能假定 SetPersistentString 跨端。

---

### 10.4.6 进阶：组件级 + entity 级数据的协调

#### 第一步：什么时候组件级、什么时候 entity 级？

**默认用组件级**——好处见 10.2.3 第四步。

**但 entity 级 OnSave 在某些场景更合适**：
- mod 想给 entity 加"独立于组件"的元信息（不想为了一个字段建一个组件）
- 数据**横跨多个组件**——属于 entity 整体，不属于任何单一组件
- mod 不想引入新组件文件，只想在 prefab fn 里挂数据

#### 第二步：混合使用

某 mod 既要保存"自定义皮肤选择"（entity 级），又要保存"装备耐久"（组件级）：

```lua
local function fn()
    local inst = CreateEntity()
    -- ...
    
    -- entity 级
    inst.OnSave = function(inst, data)
        data.skin_variant = inst._skin_variant
    end
    
    inst.OnLoad = function(inst, data)
        if data and data.skin_variant then
            inst._skin_variant = data.skin_variant
            ApplySkin(inst, data.skin_variant)
        end
    end
    
    -- ...
    
    if not TheWorld.ismastersim then return inst end
    
    -- 组件级
    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(100)
    
    return inst
end
```

**两套并存** —— 互不干扰——OnSave 调用顺序：先组件、再 entity（见 10.2.3）。

#### 第三步：何时把 entity 级数据"升级"为组件级？

**早期**：1 个 mod 字段——直接挂 inst。  
**中期**：3 个相关字段——还能挂 inst。  
**后期**：5+ 个字段 + 业务逻辑——**应该升级为组件**。

**升级带来的好处**：
- 业务逻辑放在组件方法里——可重用、可单元测试
- 数据访问统一——`inst.components.xxx.field` 比 `inst._xxx_field` 清晰
- 持久化集中——单个 OnSave 包含所有相关数据
- 客户端可读——通过 replica 抽象访问

---

### 10.4.7 老手进阶：完整 mod 案例——RPG 角色养成 mod 的数据存取设计

#### 第一步：场景

mod 想做一个"RPG 角色养成"系统：

- 玩家有 5 种"职业"可选（战士、法师、盗贼、牧师、猎人）
- 每个职业有"等级"、"经验值"、"技能点"
- 玩家可以解锁"技能"
- 每个 mod 实体（武器/装备）可以"附魔" —— 不同附魔类型
- 全 mod 用户共享"成就榜"
- 服务器管理员可设"经验倍率"

#### 第二步：数据归属分析

| 数据 | 类型 | 存储 |
| --- | --- | --- |
| 玩家当前职业 | 玩家 entity | 玩家 component（绑当前角色，复活后从存档恢复）|
| 玩家等级 / XP / 技能点 | 玩家 entity | 玩家 component |
| 玩家解锁的技能列表 | 玩家 entity | 玩家 component |
| 装备附魔类型 | 装备 entity | 装备 component |
| 全 mod 玩家累计游戏时间 | 玩家身份 | 自建 generickv |
| 全 mod 成就榜（所有玩家） | mod 级 | TheSim PersistentString |
| 服务器管理员的经验倍率配置 | 当前世界 | TheWorld 组件 |
| 客户端 UI 字号偏好 | 客户端本机 | TheSim PersistentString（客户端写） |

#### 第三步：实现示例（核心模块）

**myrpg_player.lua**（玩家 component）：

```lua
local MyRPGPlayer = Class(function(self, inst)
    self.inst = inst
    self.classname = "warrior"
    self.level = 1
    self.xp = 0
    self.skillpoints = 0
    self.unlocked_skills = {}
end)

function MyRPGPlayer:GainXP(amount)
    -- 应用世界级"经验倍率"
    if TheWorld.components.myrpg_world then
        amount = amount * TheWorld.components.myrpg_world.xp_multiplier
    end
    self.xp = self.xp + amount
    -- 升级判断
    while self.xp >= self:GetXPForNextLevel() do
        self.xp = self.xp - self:GetXPForNextLevel()
        self.level = self.level + 1
        self.skillpoints = self.skillpoints + 1
    end
end

function MyRPGPlayer:OnSave()
    return {
        version = 1,
        classname = self.classname,
        level = self.level,
        xp = self.xp,
        skillpoints = self.skillpoints,
        unlocked_skills = self.unlocked_skills,
    }
end

function MyRPGPlayer:OnLoad(data)
    if data == nil then return end
    if data.version == 1 then
        self.classname = data.classname or "warrior"
        self.level = data.level or 1
        self.xp = data.xp or 0
        self.skillpoints = data.skillpoints or 0
        self.unlocked_skills = data.unlocked_skills or {}
    end
end

return MyRPGPlayer
```

**myrpg_world.lua**（世界 component）：

```lua
local MyRPGWorld = Class(function(self, inst)
    assert(inst == TheWorld)
    self.inst = inst
    self.xp_multiplier = 1.0
    self.boss_kill_count = 0
end)

function MyRPGWorld:SetXPMultiplier(m)
    self.xp_multiplier = m
end

function MyRPGWorld:OnSave()
    return {
        version = 1,
        xp_multiplier = self.xp_multiplier,
        boss_kill_count = self.boss_kill_count,
    }
end

function MyRPGWorld:OnLoad(data)
    if data == nil then return end
    if data.version == 1 then
        self.xp_multiplier = data.xp_multiplier or 1.0
        self.boss_kill_count = data.boss_kill_count or 0
    end
end

return MyRPGWorld
```

**myrpg_kvstore.lua**（玩家级 KV 存储——累计游戏时间）：

```lua
local MyRPGKV = {}
local data = {}
local loaded = false

local FILENAME = "myrpg_kv"

function MyRPGKV:Load(cb)
    TheSim:GetPersistentString(FILENAME, function(s, str)
        if s and str then
            local ok, decoded = pcall(json.decode, str)
            if ok and decoded then data = decoded.data or {} end
        end
        loaded = true
        if cb then cb() end
    end)
end

function MyRPGKV:Save()
    if not loaded then return end
    TheSim:SetPersistentString(FILENAME, json.encode({version = 1, data = data}), false)
end

function MyRPGKV:RecordPlaytime(userid, seconds)
    if not loaded or not userid then return end
    data[userid] = data[userid] or { playtime = 0 }
    data[userid].playtime = data[userid].playtime + seconds
    self:Save()
end

function MyRPGKV:GetPlaytime(userid)
    return loaded and data[userid] and data[userid].playtime or 0
end

return MyRPGKV
```

**myrpg_achievement.lua**（mod 级——全 mod 排行榜）：

```lua
-- 完全独立的存储——不绑当前世界
local MyRPGAchievement = {}
local achievements = {}  -- userid → { achievementlist }
local loaded = false

local FILENAME = "myrpg_achievement"

function MyRPGAchievement:Load(cb)
    TheSim:GetPersistentString(FILENAME, function(s, str)
        if s and str then
            local ok, decoded = pcall(json.decode, str)
            if ok and decoded then achievements = decoded.data or {} end
        end
        loaded = true
        if cb then cb() end
    end)
end

function MyRPGAchievement:Save()
    if not loaded then return end
    TheSim:SetPersistentString(FILENAME, json.encode({version = 1, data = achievements}), false)
end

function MyRPGAchievement:Unlock(userid, name)
    if not loaded or not userid then return end
    achievements[userid] = achievements[userid] or {}
    if not achievements[userid][name] then
        achievements[userid][name] = os.time()
        self:Save()
        return true
    end
    return false
end

return MyRPGAchievement
```

#### 第四步：modmain 的整合

```lua
-- modmain.lua
local MyRPGKV = require("myrpg_kvstore")
local MyRPGAchievement = require("myrpg_achievement")

-- 启动时加载
AddSimPostInit(function()
    MyRPGKV:Load()
    MyRPGAchievement:Load()
end)

-- 给玩家挂组件
AddPlayerPostInit(function(player)
    if TheWorld.ismastersim then
        player:AddComponent("myrpg_player")
    end
end)

-- 给世界挂组件
AddPrefabPostInit("world", function(inst)
    if inst.ismastersim then
        inst:AddComponent("myrpg_world")
    end
end)

-- 监听玩家击杀事件 → 给经验
AddPrefabPostInit("world", function(inst)
    if inst.ismastersim then
        inst:ListenForEvent("entity_death", function(world, data)
            if data.afflicter and data.afflicter.components.myrpg_player then
                data.afflicter.components.myrpg_player:GainXP(10)
                
                -- 玩家级累计
                MyRPGKV:RecordPlaytime(data.afflicter.userid, 1)
                
                -- 全局成就
                if data.inst:HasTag("epic") then
                    MyRPGAchievement:Unlock(data.afflicter.userid, "killed_epic_boss")
                end
            end
        end)
    end
end)
```

---

### 10.4.8 老手进阶：六个常见陷阱

#### 陷阱 1：把"应该跟玩家走的数据"绑到 entity 上

**症状**：玩家死了复活——RPG 等级/技能点全部清零。  
**原因**：mod 把数据存在某个 entity 的组件里——但 entity 销毁了。  
**修复**：判断"这条数据应该绑哪"——用决策树。**玩家级数据应该绑到 player entity 的 component**（player 复活后从存档恢复），或者用 generickv 持久化。

#### 陷阱 2：把"全 mod 共享数据"存到当前世界

**症状**：玩家在 A 存档解锁了某成就——切换到 B 存档没有了。  
**原因**：用了 TheWorld 组件 OnSave —— 数据被绑到当前存档。  
**修复**：用 TheSim:SetPersistentString —— 跨存档共享。

#### 陷阱 3：所有数据都用 SetPersistentString

**症状**：mod 自定义的"树的可砍次数"——也用 SetPersistentString —— 性能极差，**每次砍一下都写盘**。  
**原因**：实体级数据用了 mod 级机制。  
**修复**：实体级数据**永远用组件 OnSave** —— 它跟随 SaveGame 流程批量写盘。

#### 陷阱 4：忘记 mod 卸载时清理

**症状**：玩家卸载 mod 后——磁盘上还残留 PersistentString 文件——下次重装 mod 又用回旧数据。  
**原因**：mod 没在卸载时删文件。  
**修复**：通常**不删**——这是 feature（用户重装 mod 时数据保留）。**真要删**，提供"重置数据"的命令让用户主动调。

#### 陷阱 5：数据层级混乱

**症状**：mod 里有一个数据，**既存在 entity component 里**，**又存在 SetPersistentString 文件里**——两边可能不一致——业务读哪个？  
**原因**：数据归属未定义清楚。  
**修复**：单一数据源——每个数据**有且仅有一个权威存储位置**。

#### 陷阱 6：客户端和服务端写同一个文件

**症状**：dedicated 服务器和客户端都用 `TheSim:SetPersistentString("mymod_data", ...)` —— 但两边写的是**不同物理路径**（不同机器上的磁盘）。  
**原因**：客户端和服务端是不同进程/机器。  
**修复**：明确数据归属——服务端数据只服务端写、客户端数据只客户端写——跨端通过 RPC 同步。

#### 设计经验三条

**经验 ①：每条数据都问 5W1H**

写存代码前问：
- **What**：这是什么数据？
- **Who**：跟谁走（entity / 世界 / 玩家 / mod）？
- **When**：什么时候失效？
- **Where**：谁可以读？
- **Why**：为什么需要持久化？（运行时缓存其实不用存）
- **How**：用什么机制？

回答清楚再下笔——避免"先写代码后纠错"。

**经验 ②：单一数据源原则**

每条数据**只有一个权威存储位置**——其他地方都通过查询/同步获取——不要"两份独立但功能相同的存储"。

**经验 ③：永远 version 字段优先**

mod 第一版的所有 OnSave / SetPersistentString 都加 version：

```lua
return { version = 1, ... }
```

未来兼容老存档时**有这个字段就能从容应对**——见 10.5 节。

---

### 10.4.9 小结

**Mod 数据存取一句话总结**：**按数据归属选机制**——entity 级用组件 OnSave、世界级用 TheWorld 组件、跨世界用 SetPersistentString、跨 shard 用 cluster slot、跨服务器用 generickv。

**5 大类 / 5 个机制 速查表**

| 数据类型 | 存储机制 | 一行代码 |
| --- | --- | --- |
| 实体级 | 组件 OnSave/OnLoad | `function MyComp:OnSave() return {x=self.x} end` |
| 世界级 | TheWorld 组件 OnSave | 同上，但 component 挂在 world |
| 客户端可见的世界级 | world_network 组件 + NetVar | 加 net_xxx 创建 |
| 跨 shard 共享 | cluster slot 文件 | `TheSim:SetPersistentStringInClusterSlot(slot, nil, ...)` |
| 玩家级（按 userid）| 自建 generickv 风格 | `TheSim:SetPersistentString("mymod_player", ...)` |
| mod 级 | TheSim PersistentString | 同上 |

**6 个陷阱排雷顺序**

1. 玩家数据绑 entity → 绑 player 或 generickv
2. 全 mod 数据绑世界 → 用 SetPersistentString
3. 实体数据用 SetPersistentString → 用组件 OnSave
4. mod 卸载没清理 → 提供"重置"命令
5. 数据双源 → 单一数据源
6. 跨端写同名 → 明确归属

**3 条设计经验**

- ① **5W1H 分析每条数据**
- ② **单一数据源**
- ③ **version 字段优先**

> **下一节预告**：10.5 节是整章收尾——**存档兼容性：更新 Mod 后旧存档不崩溃的策略**。读完 10.5，你的 mod 更新版本时——**老用户的存档永远能加载**——这是发布 Steam Workshop mod 的**职业素养**。

## 10.5 存档兼容性——更新 Mod 后旧存档不崩溃的策略

### 本节导读

10.1 ~ 10.4 我们把存档系统的**机制**讲透了。**最后一关**是关于"**软件演化**"——你的 mod 不会一次写完——你会**反复发版本**：

- v1：基础功能
- v2：加了新字段 → "解锁的技能列表"
- v3：删掉旧字段 → "弃用的成就计数"
- v4：重命名字段 → "level" 改名为 "char_level"
- v5：组件结构调整 → 新增了"装备"组件
- v6：数据类型变化 → "score" 从 number 改成 table

**每一次发版**——**老用户的存档可能会崩**——除非你**主动做兼容**。

**这是 Steam Workshop mod 的"职业素养"问题**——发了一个新版本——10000 个老玩家的存档全部崩了——你会被骂到下架。**Klei 自己有完整的"savefileupgrades"系统**——保证 10 年前的存档现在还能加载——这是值得 mod 开发者学习的工业级实践。

这一节我们把"**mod 升级时如何保证旧存档不崩**"的所有套路讲清楚——从最简单的"加新字段"到最复杂的"组件结构重构"——给出可工业化使用的兼容策略。

> **新手**从 10.5.1-10.5.3 起步——理解兼容性问题的存在、3 类破坏性变更、版本号 + 升级链标准模板；**进阶读者**继续看 10.5.4-10.5.6，深入 `savefileupgrades.lua` 兼容机制、`add_component_if_missing` 加新组件、删除/重命名/类型变化处理；**老手**跳到 10.5.7-10.5.8，看完整的 mod 多版本升级链实战 + 6 个最容易踩的兼容陷阱。

---

### 10.5.1 快速入门：从一次"mod 更新"看兼容性问题

#### 第一步：典型场景——mod v1 → v2

mod v1 的 OnSave：

```lua
-- v1
function MyComp:OnSave()
    return { hp = self.hp }
end

function MyComp:OnLoad(data)
    if data == nil then return end
    self.hp = data.hp or 100
end
```

玩家 A 用 v1 玩了 100 小时——存档里全是 v1 格式的数据。

mod 升级到 v2 加了"魔法值"：

```lua
-- v2 第一次尝试
function MyComp:OnSave()
    return { hp = self.hp, mp = self.mp }
end

function MyComp:OnLoad(data)
    if data == nil then return end
    self.hp = data.hp or 100
    self.mp = data.mp  -- ❌ 旧存档没 mp，self.mp 会是 nil
end
```

**问题**：`self.mp` 是 nil ——后面的代码 `self.mp + 10` 报错。

#### 第二步：第一次修复——加默认值

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    self.hp = data.hp or 100
    self.mp = data.mp or 50  -- ✓ 加默认值
end
```

**这是最简单的兼容**——**新字段加默认值**——老存档自动用默认值。

#### 第三步：更复杂的场景—— v2 改了"hp 计算公式"

v1：`hp` 直接是当前血量（最大 100）  
v2：`hp` 改成"百分比"（0.0-1.0）

**简单加默认值不够**——必须**主动转换数据格式**：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    
    if data.hp and data.hp > 1 then
        -- 旧 v1 数据：hp 是 0-100
        self.hp = data.hp / 100  -- 转成百分比
    else
        -- v2 新数据：hp 已经是 0-1
        self.hp = data.hp or 1.0
    end
end
```

**问题**：`data.hp > 1` 这种判断**脆弱**——如果 v1 玩家存了 `hp = 1`，会被错误识别为 v2。

#### 第四步：用 version 字段——一劳永逸

```lua
-- v2 OnSave 加 version
function MyComp:OnSave()
    return { version = 2, hp = self.hp, mp = self.mp }
end

function MyComp:OnLoad(data)
    if data == nil then return end
    
    local version = data.version or 1
    
    if version == 1 then
        -- v1 → v2 升级
        self.hp = (data.hp or 100) / 100
        self.mp = 50  -- v1 没有，默认值
    elseif version == 2 then
        -- 直接读 v2
        self.hp = data.hp or 1.0
        self.mp = data.mp or 50
    end
end
```

**优势**：**明确版本边界**——不依赖数据值的"启发式判断"——升级链可扩展（v3、v4）。

> **核心结论**：**版本号 + 升级链是兼容性的核心模式**——v1 第一次写 mod 时就该加上。**没加 version 的 mod 升级时极其痛苦**。

---

### 10.5.2 快速入门：3 类破坏性变更 + 应对策略

#### 类 1：加新字段（非破坏性）

**场景**：v2 加了 `mp` 字段——v1 没有。

**应对**：永远给新字段**默认值**：

```lua
self.mp = data.mp or 50
```

**变种**：新字段是 table

```lua
self.unlocked_skills = data.unlocked_skills or {}
```

**变种**：新字段是嵌套 table 时**深拷贝默认值**

```lua
self.config = data.config and deepcopy(data.config) or { volume = 1, fontsize = 14 }
```

#### 类 2：删除字段（半破坏性）

**场景**：v2 决定不再保存 `useless_field`。

**应对**：OnLoad 直接忽略——但**OnSave 还是要写一次**（可选——为了保持向后兼容，让 v1 mod 用户可以读 v2 存档）：

```lua
-- v2 OnSave 仍写
function MyComp:OnSave()
    return {
        version = 2,
        hp = self.hp,
        mp = self.mp,
        useless_field = self.useless_field,  -- 保留向后兼容
    }
end

-- v2 OnLoad 不读
function MyComp:OnLoad(data)
    if data == nil then return end
    -- 直接忽略 data.useless_field
    self.hp = data.hp or 1
    self.mp = data.mp or 50
end
```

**或者**：彻底丢掉——**接受"用户回退到 v1 时数据丢失"** 的代价。

#### 类 3：改变字段语义（破坏性）

**场景**：v2 把 `level` 字段从"整数"改为"table（含 main + sub）"。

**应对**：**版本号 + 转换函数**：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    local version = data.version or 1
    
    if version == 1 then
        -- v1: level 是数字
        self.level = { main = data.level or 1, sub = 0 }
    elseif version == 2 then
        -- v2: level 是 table
        self.level = data.level or { main = 1, sub = 0 }
    end
end
```

**变种**：字段改了名字（rename）

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    local version = data.version or 1
    
    if version == 1 then
        -- v1: 字段叫 my_old_name
        self.value = data.my_old_name or 0
    elseif version == 2 then
        -- v2: 字段叫 my_new_name
        self.value = data.my_new_name or 0
    end
end

function MyComp:OnSave()
    return { version = 2, my_new_name = self.value }
end
```

#### 选哪种策略？

**优先级**：
1. 加新字段（默认值兜底）—— 最简单
2. 保留旧字段（即使不再用）—— 让回退兼容
3. 改语义（version + 转换函数）—— 标准方案
4. 删字段（接受用户损失）—— 最激进

---

### 10.5.3 快速入门：版本号 + 升级链的标准模板

#### 第一步：完整的"版本化"OnSave/OnLoad

```lua
-- 当前版本
local CURRENT_VERSION = 3

-- 升级链
local upgrades = {
    [1] = function(data)
        -- v1 → v2
        data.mp = data.mp or 50
        return data
    end,
    [2] = function(data)
        -- v2 → v3
        data.skills = data.skills or {}
        data.gold = data.gold or 0
        return data
    end,
    -- 未来 [3], [4], ...
}

-- 标准 OnSave —— 永远写最新版本
function MyComp:OnSave()
    return {
        version = CURRENT_VERSION,
        hp = self.hp,
        mp = self.mp,
        skills = self.skills,
        gold = self.gold,
    }
end

-- 标准 OnLoad —— 走升级链
function MyComp:OnLoad(data)
    if data == nil then return end
    
    local version = data.version or 1
    
    -- 一路升级到当前版本
    while version < CURRENT_VERSION do
        if upgrades[version] then
            data = upgrades[version](data)
        end
        version = version + 1
    end
    
    -- 现在 data 是最新格式
    self.hp = data.hp or 1
    self.mp = data.mp or 50
    self.skills = data.skills or {}
    self.gold = data.gold or 0
end
```

#### 第二步：升级链的优势

- **每次升级 fn 单独写** —— 老 v1 fn 不会被未来代码破坏
- **跨版本升级**自动处理 —— v1 存档加载时**依次** v1→v2→v3 转换
- **测试简单** —— 每个升级 fn 是纯函数，可单独单元测试
- **回滚简单** —— 想撤销某次升级，只删一行代码

#### 第三步：用 Klei 的 `savefileupgrades.lua` 做参考

打开 `scripts/savefileupgrades.lua` 看 Klei 的真实升级链代码（10.5.4 节会详细解析）—— **完全相同的模式**：

- 每个版本一个升级 fn
- fn 接收 savedata（或子 table）、修改、返回
- 最外层 wrapper 按版本号依次调用

**Klei 的代码经过多年演化、多次大版本升级**——验证了这个模式的工业级实用性。

---

### 10.5.4 进阶：`savefileupgrades.lua` 的兼容机制

#### 第一步：Klei 的升级 fn 长什么样

`scripts/savefileupgrades.lua:32-87`（10.1.6 节简略看过）：

```32:87:scripts/savefileupgrades.lua
        UpgradeUserPresetFromV1toV2 = function(preset, custompresets)
            if preset.version ~= nil and preset.version >= 2 then
                return preset
            end
            print(string.format("Upgrading user preset data for '%s' from v1 to v2", preset.data))

            local newid = preset.data
            local newname = preset.text
            local newdesc = preset.desc
            local location = preset.location
            local basepreset = preset.basepreset
            local overrides = preset.overrides

            local Levels = require"map/levels"
            ...
            return ret
        end,
```

**关键观察**：
- **第一行 `if preset.version >= 2 then return preset end`** —— **幂等保护**——多次调用安全
- **每次升级都打 print log** —— "Upgrading user preset data ..."—— 调试时容易追踪
- **完整重构数据格式** —— v1 → v2 是结构性变化（preset 从扁平变成 hierarchical）

#### 第二步：分模块升级

`savefileupgrades.lua` 把不同模块的升级 fn 分开：

```
utilities = {
    UpgradeUserPresetFromV1toV2 = ...,
    UpgradeUserPresetFromV2toV3 = ...,
    UpgradeUserPresetFromV3toV4 = ...,
    ...
}
```

**好处**：每个模块独立升级——不互相干扰。

#### 第三步：retrofitforestmap_anr 模式

文件开头：

```2:14:scripts/savefileupgrades.lua
local function FlagForRetrofitting_Forest(savedata, flag_name)
    if savedata ~= nil and savedata.map ~= nil and savedata.map.prefab == "forest" then
        if savedata.map.persistdata == nil then
            savedata.map.persistdata = {}
        end

        if savedata.map.persistdata.retrofitforestmap_anr == nil then
            savedata.map.persistdata.retrofitforestmap_anr = {}
        end
        savedata.map.persistdata.retrofitforestmap_anr[flag_name] = true
    end
end
```

**这是 Klei 的"标记 + 延后处理"模式**：
- savefileupgrades 阶段——**只标记**"这个存档需要做某种 retrofit"
- 真正的 retrofit 由 `retrofitforestmap_anr` 组件在加载时执行

**优势**：复杂的世界结构变更（地图新增区域、新增 prefab 类型）可以**异步处理**——不卡 savefileupgrades 流程。

#### 第四步：mod 借鉴 Klei 模式

```lua
-- 在 mod modmain 加 retrofit 标记
local function MarkRetrofit(savedata, flag)
    if savedata.map and savedata.map.persistdata then
        savedata.map.persistdata.mymod_retrofit = savedata.map.persistdata.mymod_retrofit or {}
        savedata.map.persistdata.mymod_retrofit[flag] = true
    end
end

-- 在 mymod_retrofit 组件的 OnLoad 里处理标记
function MyModRetrofit:OnLoad(data)
    if data == nil then return end
    
    if data.add_new_npc then
        -- 在地图上 spawn 新 NPC
        SpawnNewNPCs()
        data.add_new_npc = nil  -- 处理过就清掉
    end
end
```

---

### 10.5.5 进阶：`add_component_if_missing` 与"加新组件"

#### 第一步：场景

mod v1 没有 `myxp` 组件——v2 给玩家加了 myxp 组件。**v1 玩家的存档**没有 myxp 数据——加载后这些玩家**没 myxp 组件**——业务断了。

#### 第二步：`add_component_if_missing` 标记

`entityscript.lua:1955-1961`（10.2.6 节看过）：

```lua
if data and data.add_component_if_missing then
    for k, v in pairs(data) do
        if self.components[k] == nil and type(v) == "table" and v.add_component_if_missing then
            self:AddComponent(k)
        end
    end
end
```

**机制**：如果 OnSave 数据里有 `add_component_if_missing = true`——加载时**自动给 entity 加这个组件**。

#### 第三步：mod v1 没考虑，怎么办？

mod v1 时**没人给 OnSave 加 add_component_if_missing**——v1 的存档里**根本没有 myxp 数据**——v2 怎么知道要加？

**答案**：**用 PrefabPostInit + 检查 component 是否存在**：

```lua
-- modmain.lua（v2）
AddPlayerPostInit(function(player)
    if TheWorld.ismastersim then
        if not player.components.myxp then
            player:AddComponent("myxp")
        end
    end
end)
```

**`AddPlayerPostInit`** 在玩家 prefab 加载完后调用——**不论存档是新的还是 v1 的**——确保 myxp 组件总是存在。

#### 第四步：升级期间组件初始化

新加的组件 OnLoad 收到的 data **可能是 nil**（v1 存档没数据）——按 10.2.8 陷阱 2 处理：

```lua
function MyXP:OnLoad(data)
    if data == nil then
        -- v1 存档第一次加载——给默认值
        self.level = 1
        self.xp = 0
    else
        -- 走标准升级链
        local version = data.version or 1
        if version == 1 then
            self.level = data.level or 1
            self.xp = data.xp or 0
        end
    end
end
```

#### 第五步：跨"加组件"和"删组件"

**v3 把 myxp 组件**改名为 `myrpg`——**双向兼容**：

```lua
-- modmain.lua（v3）
AddPlayerPostInit(function(player)
    if TheWorld.ismastersim then
        if not player.components.myrpg then
            player:AddComponent("myrpg")
            
            -- 如果有旧的 myxp 数据，迁移过来
            if player.components.myxp then
                local xp = player.components.myxp.xp
                local level = player.components.myxp.level
                player.components.myrpg.level = level
                player.components.myrpg.xp = xp
                player:RemoveComponent("myxp")
            end
        end
    end
end)
```

---

### 10.5.6 进阶：删除字段 / 重命名字段 / 类型变化的处理

#### 删除字段：保留兼容兜底

**v3 决定不再用 `obsolete_field`**——但**v2 玩家可能升回 v3 又升回 v2**——为了保护这种来回升级，**还是写**这个字段：

```lua
-- v3 OnSave
function MyComp:OnSave()
    return {
        version = 3,
        new_data = self.new_data,
        obsolete_field = nil,  -- 不再写
    }
end

-- v3 OnLoad
function MyComp:OnLoad(data)
    if data == nil then return end
    
    -- 不读 obsolete_field
    self.new_data = data.new_data or {}
end
```

**变种**：如果删字段会**真的破坏老版本**（玩家从 v3 回到 v2 时报错）——**v3 的 OnSave 仍写**这个字段，给个兼容值：

```lua
function MyComp:OnSave()
    return {
        version = 3,
        new_data = self.new_data,
        obsolete_field = self:_GetObsoleteValue(),  -- 派生兼容值
    }
end
```

#### 重命名字段：双键过渡 → 净化

**v2 把 `oldname` 改成 `newname`**：

**第 1 步：双键过渡（v2.0）**

```lua
function MyComp:OnSave()
    return {
        version = 2,
        oldname = self.value,    -- 老名字（向后兼容）
        newname = self.value,    -- 新名字
    }
end

function MyComp:OnLoad(data)
    if data == nil then return end
    self.value = data.newname or data.oldname or 0  -- 优先 new，兜底 old
end
```

**第 2 步：净化（v2.5 / v3）**

经过几个版本"双键过渡"——所有用户都升到 v2 后——可以**去掉 oldname**：

```lua
function MyComp:OnSave()
    return {
        version = 3,
        newname = self.value,
    }
end

function MyComp:OnLoad(data)
    if data == nil then return end
    self.value = data.newname or data.oldname or 0  -- 仍然保留兜底
end
```

**第 3 步：彻底删除（远期 v5）**

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    self.value = data.newname or 0
end
```

#### 类型变化：必须 version + 转换 fn

**v2 把 `level` 从 number 改成 table**——**永远不能依赖类型 hint**——必须用 version：

```lua
function MyComp:OnLoad(data)
    if data == nil then return end
    local version = data.version or 1
    
    if version == 1 then
        -- v1: level 是数字 5
        self.level = { main = data.level or 1, sub = 0 }
    elseif version == 2 then
        -- v2: level 是 table {main=5, sub=2}
        self.level = data.level or { main = 1, sub = 0 }
    end
end
```

---

### 10.5.7 老手进阶：mod 多版本升级链完整实战

#### 场景

mod 经历了 5 个版本，存档兼容情况：

- v1 (2023-01)：基础版，只有 `level` 数字
- v2 (2023-06)：加了 `xp` / `gold` 字段
- v3 (2024-01)：`level` 从 number 改成 table（main + sub）
- v4 (2024-06)：删除了 `gold`，引入"货币系统"组件
- v5 (2025-01)：重命名 `xp` 为 `experience`

#### 第一步：完整的 myrpg.lua

```lua
-- mymod/scripts/components/myrpg.lua

local CURRENT_VERSION = 5

-- 升级链
local upgrades = {
    [1] = function(data)
        -- v1 → v2: 加 xp、gold 默认值
        data.xp = data.xp or 0
        data.gold = data.gold or 0
        return data
    end,
    [2] = function(data)
        -- v2 → v3: level number → table
        if type(data.level) == "number" then
            data.level = { main = data.level, sub = 0 }
        end
        return data
    end,
    [3] = function(data)
        -- v3 → v4: 删除 gold（迁移到货币系统）—— 这里只设标记，让 modmain 处理
        data.migrate_gold = data.gold  -- 暂存
        data.gold = nil
        return data
    end,
    [4] = function(data)
        -- v4 → v5: rename xp → experience
        data.experience = data.experience or data.xp
        data.xp = nil
        return data
    end,
}

local MyRPG = Class(function(self, inst)
    self.inst = inst
    self.level = { main = 1, sub = 0 }
    self.experience = 0
    -- gold 已经迁移走，不在这里
end)

function MyRPG:OnSave()
    return {
        version = CURRENT_VERSION,
        level = self.level,
        experience = self.experience,
    }
end

function MyRPG:OnLoad(data)
    if data == nil then return end
    
    local version = data.version or 1
    
    -- 走升级链
    while version < CURRENT_VERSION do
        if upgrades[version] then
            data = upgrades[version](data)
        end
        version = version + 1
    end
    
    -- 现在 data 是 v5 格式
    self.level = data.level or { main = 1, sub = 0 }
    self.experience = data.experience or 0
    
    -- 处理"迁移到货币系统"
    if data.migrate_gold and self.inst.components.mycurrency then
        self.inst.components.mycurrency:AddGold(data.migrate_gold)
    end
end

return MyRPG
```

#### 第二步：完整的 mycurrency.lua（v4 新增）

```lua
local MyCurrency = Class(function(self, inst)
    self.inst = inst
    self.gold = 0
end)

function MyCurrency:AddGold(amount)
    self.gold = self.gold + amount
end

function MyCurrency:OnSave()
    return { version = 1, gold = self.gold }
end

function MyCurrency:OnLoad(data)
    if data == nil then return end
    self.gold = data.gold or 0
end

return MyCurrency
```

#### 第三步：modmain.lua

```lua
-- modmain.lua

-- v4 引入的：所有玩家自动加 mycurrency 组件
AddPlayerPostInit(function(player)
    if TheWorld.ismastersim then
        if not player.components.mycurrency then
            player:AddComponent("mycurrency")
        end
        if not player.components.myrpg then
            player:AddComponent("myrpg")
        end
    end
end)
```

#### 第四步：v1 玩家加载到 v5 的全过程

```
v1 存档：
  player.data.myrpg = { level = 5 }
  
加载 v5 mod：
  AddPlayerPostInit
    → 给 player 加 mycurrency 组件（mycurrency:OnLoad(nil) → gold = 0）
    → 给 player 加 myrpg 组件
  
  myrpg:OnLoad({ level = 5 })
    version = 1
    
    升级 1→2: data.xp = 0, data.gold = 0
    升级 2→3: data.level = { main = 5, sub = 0 }
    升级 3→4: data.migrate_gold = 0, data.gold = nil
    升级 4→5: data.experience = 0, data.xp = nil
    
    最终 data = { version = 1, level = {main=5, sub=0}, experience = 0, migrate_gold = 0 }
    
    self.level = { main = 5, sub = 0 }   ✓
    self.experience = 0                  ✓
    
    self.inst.components.mycurrency:AddGold(0)
    
v5 mod 看到：玩家等级 5、经验 0、金币 0—— 所有数据正确恢复
```

---

### 10.5.8 老手进阶：六个常见陷阱

#### 陷阱 1：v1 没加 version 字段

**症状**：v2 想升级时**完全没法判断"这是 v1 还是 v2"**——只能靠数据值的启发式判断——脆弱。  
**修复**：v1 第一次写 OnSave 就加 version。**未来你会感谢现在的自己**。

#### 陷阱 2：升级 fn 不是幂等的

```lua
upgrades[1] = function(data)
    data.gold = (data.gold or 0) + 50  -- ❌ 多次调用会累加
    return data
end
```

**症状**：v2 玩家被错误识别为 v1 调用 upgrade —— gold 多了 50。  
**修复**：升级 fn 必须**幂等**——可重复调用结果相同——通常 `or 默认值` 模式：

```lua
upgrades[1] = function(data)
    data.gold = data.gold or 50  -- ✓ 幂等
    return data
end
```

#### 陷阱 3：删字段没保留兼容兜底

**症状**：v2 删了 `obsolete_field`——v2 玩家想用回 v1 mod —— v1 报错"字段不存在"。  
**修复**：**保留写入**——即使不再用：

```lua
function MyComp:OnSave()
    return {
        version = 2,
        new_data = self.new_data,
        obsolete_field = self:_DeriveOld(),  -- 派生兼容值
    }
end
```

或者**接受用户回退时损失**——但要在 changelog 里**警告用户**。

#### 陷阱 4：版本号跳跃

**症状**：mod 直接从 v1 跳到 v3——没写 v2 →v3 的升级 fn。  
**修复**：**永远连续递增 version**——v1 → v2 → v3——每个升级 fn 都要写。

#### 陷阱 5：升级链改了"老版本的 OnSave 数据格式"

**症状**：你以为 v2 玩家的存档里有某字段——但你忘了 v2 的 OnSave 没写——升级 fn 内部 `data.field` 是 nil 报错。  
**修复**：升级 fn **永远先判 nil**：

```lua
upgrades[2] = function(data)
    data.new_field = data.old_field or default_value  -- ✓
    return data
end
```

#### 陷阱 6：跨组件数据迁移没处理

**症状**：v4 把 myrpg 的 gold 迁到 mycurrency——但 mycurrency 的 OnLoad **先于** myrpg 的 OnLoad —— 迁移时 mycurrency 已经 load 完，gold 是 0 而不是从老数据迁移过来的。  
**修复**：**跨组件迁移用 LoadPostPass**——它在所有组件 OnLoad 完成后才调：

```lua
function MyRPG:LoadPostPass(newents, savedata)
    if savedata.migrate_gold and self.inst.components.mycurrency then
        self.inst.components.mycurrency:AddGold(savedata.migrate_gold)
        savedata.migrate_gold = nil  -- 处理过就清掉
    end
end
```

#### 设计经验三条

**经验 ①：v1 第一天就加 version + 升级链**

哪怕 v1 没有什么"升级"逻辑——也要：

```lua
local CURRENT_VERSION = 1

function MyComp:OnSave()
    return { version = CURRENT_VERSION, ... }
end
```

未来 v2 时——加 upgrades[1] = function(data) ... end —— 平滑过渡。

**经验 ②：每次发版本前测试老存档**

发布前的标准测试流程：
1. 用旧版 mod 玩一会，触发各种存档场景
2. 关闭服务器，**保留存档**
3. 替换为新版 mod
4. 启动服务器，**验证旧存档能正常加载**
5. 验证旧数据已正确升级

**没做这个测试的 mod 发版本是不负责任的**。

**经验 ③：changelog 明确说明兼容性**

每次发版本写 changelog：

```
v3.0
- 新功能：加了魔法值系统
- 数据迁移：v1/v2 玩家的 hp 自动转换为百分比
- 已弃用：obsolete_field（仍可读，将在 v5 完全移除）
- 警告：从 v3 回退到 v2 会丢失"魔法值"数据
```

让用户**知情**——避免"突然崩溃"的负面体验。

---

### 10.5.9 小结

**存档兼容性一句话总结**：**version 字段 + 升级链 + 幂等升级 fn + 默认值兜底**——四件套保证 mod 任何升级老存档都不崩。

**3 类破坏性变更对照表**

| 变更类型 | 破坏程度 | 应对策略 |
| --- | --- | --- |
| 加新字段 | 非破坏 | 默认值兜底 |
| 删字段 | 半破坏 | 保留写入兼容兜底 / 接受损失 |
| 改语义 / 重命名 / 改类型 | 破坏 | version + 转换 fn |

**6 个陷阱排雷顺序**

1. 没 version → v1 第一天就加
2. 升级 fn 不幂等 → 永远 `or 默认值`
3. 删字段没兜底 → 保留写入
4. 版本号跳跃 → 连续递增
5. 升级 fn 没判 nil → 永远先判
6. 跨组件迁移用 OnLoad → 用 LoadPostPass

**3 条核心经验**

- ① **v1 第一天加 version + 升级链**
- ② **每次发版前测老存档**
- ③ **changelog 写清兼容性**

---

### 整章 10.x 总结

第 10 章 **Save/Load 持久化系统** 到此完整收尾——

- **10.1** 存档整体结构与加载流程：5 步加载、ents/map.persistdata 双层结构、references + GUID 解析
- **10.2** OnSave/OnLoad 详解：组件级 vs entity 级、双返回值含 references、LoadPostPass 双阶段
- **10.3** persistdata 与世界级数据：TheWorld 组件 OnSave、`TheSim:SetPersistentString`、cluster slot
- **10.4** Mod 自定义数据正确存取：5 大类型决策树、5 个标准模板、完整 RPG 案例
- **10.5** 存档兼容性：version 字段、升级链、3 类破坏性变更应对、Klei 真实模式

读完整章——**你将拥有为 mod 数据"任何场景都选对存储 + 任何升级都不崩老存档"的能力**。结合第 7 章 Action 系统、第 8 章 事件系统、第 9 章 网络同步——**饥荒联机版 mod 开发的完整工程能力**已经全部覆盖。
