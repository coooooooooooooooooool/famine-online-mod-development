# 第23章 物品与容器系统

### 23.1 Inventoryitem 组件——物品的通用属性

> 源码：`scripts/components/inventoryitem.lua`（主机端） + `scripts/components/inventoryitem_replica.lua`（客户端 replica）
> 关联：`scripts/standardcomponents.lua`（`MakeInventoryPhysics`、`MakeInventoryFloatable`）、`scripts/ocean_util.lua`（`ShouldEntitySink`、`SinkEntity`）

---

#### 一、它到底是什么——一句话定位

`inventoryitem` 是饥荒里**一切"能被捡起来的东西"**必须挂载的核心组件。木头、斧头、护甲、肉、宝石、被擒住的兔子，本质上都是"挂了 inventoryitem 组件的 entity"。
没有这个组件，玩家点击物品时不会触发"捡起"动作，物品也不会出现在背包槽位里。

在联机版中，它分成两半：
- 主机端的 `components/inventoryitem.lua` 负责真正的所有权、落地、湿度、沉水等逻辑；
- 客户端的 `components/inventoryitem_replica.lua` 负责通过 `inventoryitem_classified` 网络实体把 UI 必需的数据（图集、是否可拾起、湿度、攻击距离、部署模式……）同步给所有客户端。

下面分三个梯度讲。

---

#### 二、面向新手——"我只想做一个能捡的木头"

##### 2.1 最小可工作示例（与官方 `prefabs/log.lua` 同构）

```lua
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

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")

    return inst
end

return Prefab("log", fn, assets)
```

##### 2.2 新手只需要记住这 4 行模板

```lua
inst.entity:AddNetwork()       -- 联机版强制：所有 prefab 都要有 Network 才能挂 replica
MakeInventoryPhysics(inst)     -- 给物品一个"小球体物理"，能掉地能滚（来自 standardcomponents.lua 第 370 行）
inst.entity:SetPristine()      -- 把上面定义的"两端共有"的部分锁死，下面才能区分主客户端
if not TheWorld.ismastersim then return inst end  -- 客户端到此为止
inst:AddComponent("inventoryitem")  -- 只有主机会真正持有逻辑
```

> **新手最容易犯的错**：
> 1. 忘了 `AddNetwork`/`SetPristine`，联机版进游戏直接崩；
> 2. 在 `SetPristine()` 之后才 `AddTag`，客户端拿不到这个 tag；
> 3. 把 `inventoryitem` 加在 `if not TheWorld.ismastersim then return inst end` **之前**——结果客户端也尝试加这个组件，会报错。组件必须在 `ismastersim` 分支之后加。

##### 2.3 物品图标怎么显示

`inventoryitem` 不会自己画图标。新手最常见的两种情况：

- 物品贴图文件叫 `images/inventoryimages/log.tex`，名字和 prefab 一样 → **什么都不用写**，系统自动从 `self.inst.prefab..".tex"` 取（见 replica 的 `GetImage`，第 156 行）。
- 自己做了一个 `images/inventoryimages/myitem.xml` 图集，但物品 prefab 不叫 myitem：

```lua
inst.components.inventoryitem.atlasname = "images/inventoryimages/myitem.xml"
inst.components.inventoryitem.imagename = "myitem_special"   -- 不带 .tex 后缀
```

写法记住一点：**设置 `atlasname` 时要写带 `.xml` 的完整路径，设置 `imagename` 时不要带 `.tex`**（`.tex` 是 replica 内部自动拼上去的，见 replica 第 132 行 `self.classified.image:set(imagename ~= nil and (imagename..".tex") or 0)`）。

##### 2.4 新手版"两个事件回调"

```lua
inst.components.inventoryitem:SetOnDroppedFn(function(inst)
    print("被丢出来了")
end)

inst.components.inventoryitem:SetOnPutInInventoryFn(function(inst, owner)
    print("被", owner, "捡进了背包")
end)
```

> 不要去赋值 `inst.components.inventoryitem.ondropfn = xxx`、`onputininventoryfn = xxx`。虽然字段名一样，但官方的写法是用上面这两个 setter。这样你以后改组件源码兼容性更好。

---

#### 三、面向进阶——"我要让物品自己有脾气"

##### 3.1 字段全景表（来自源码 `inventoryitem.lua` 第 58 ~ 91 行的构造函数）

| 字段 | 默认值 | 类型 | 含义 / 典型用途 |
|------|--------|------|----------------|
| `owner` | `nil` | entity | 当前持有者（玩家、容器、马等）。**不要手动改**，统一用 `SetOwner` / `ClearOwner` |
| `canbepickedup` | `true` | bool | 能否被玩家右键/F 捡起。设为 `false` 时玩家无法把它收进背包（如骑乘中的牛、被锁住的灯笼） |
| `canbepickedupalive` | `false` | bool | 给小喽啰（如 eyeplant）专用：是否能在仍存活时被捡起 |
| `isnew` | `true` | bool | 给 `ProfileStatsAdd("collect_xxx")` 统计成就用，捡过一次后变 `false` |
| `nobounce` | `false` | bool | 丢出时不弹跳（如月岩种子、夜灯、蜘蛛巢蛋）。会改变 `DoDropPhysics` 里 y 速度（第 319/324 行） |
| `cangoincontainer` | `true` | bool | 能否进入"非玩家口袋"的容器（箱子、冰箱）。设为 `false` 后只能拿手上 |
| `canonlygoinpocket` | `false` | bool | 只能进**玩家口袋**（不能进容器）。`canonlygoinpocket` 和 `canonlygoinpocketorpocketcontainers` **互斥** |
| `canonlygoinpocketorpocketcontainers` | `false` | bool | 只能进玩家口袋 + 同样标记为"只允许口袋"的容器。两者只能选一个开 |
| `islockedinslot` | `false` | bool | 锁定在当前槽位，无法被拖出（用于打怪过程中某些被钉死的装备） |
| `keepondeath` | `false` | bool | 玩家死亡时**不**掉落该物品（如汪达的复活怀表、某些 hats） |
| `atlasname` | `nil` | string | 自定义图集路径，详见 2.3 |
| `imagename` | `nil` | string | 自定义图像名，详见 2.3 |
| `trappable` | `true` | bool | 是否可以被捕兽夹之类的陷阱捕获（蜘蛛、兔子是 `inventoryitem` + `trappable`） |
| `sinks` | `false` | bool | 落入水面时是否真的沉下去（默认不沉，是因为大多数物品有 `floater` 浮在水面）。配合 `SetSinks(true)` 使用 |
| `droprandomdir` | `false` | bool | 丢出时朝随机方向（被 `OnDropped` 当作 `randomdir` 参数读取，但实际调用方主要靠 inventory 自己传，参考 inventory 组件） |
| `isacidsizzling` | `false` | bool | 在酸雨里"嘶嘶冒烟"特效标志（DLC 内容，对应 `acidsizzlingchange` 事件） |
| `grabbableoverridetag` | `nil` | hash | 强制允许带某 tag 的角色无视"不可拾起"限制，例如让蜘蛛低语者捡蜘蛛 |
| `is_landed` | 通过 `SetLanded(false, true)` 初始化 | bool | 当前是否落地稳定。由组件自动维护，**不要直接读写**，用 `SetLanded` |
| `pushlandedevents` | `true` | bool | 是否在落地/起飞时推送 `on_landed` / `on_no_longer_landed` 事件 |

> 注意：构造函数最后还有一行 `if self.inst.components.waterproofer == nil then self:EnableMoisture(true) end`。意思是**只要物品本身不是防水包，就自动挂上 `inventoryitemmoisture` 子组件来跟踪湿度**。这是为什么大多数物品自动有"潮湿"属性的根源。

##### 3.2 方法清单（实战常用，按用途分组）

**所有权 / 槽位**
```lua
SetOwner(owner)        -- 设置归属（由 inventory/container 调用，一般不要自己用）
ClearOwner()           -- 清空归属
GetSlotNum()           -- 返回当前所在槽位号；不在容器里时返回 nil
GetContainer()         -- 返回当前所在的 container 或 inventory 组件
GetGrandOwner()        -- 一路往外找，返回最外层的 owner（玩家身上套了背包，背包里套了物品 → 返回玩家）
IsHeld()               -- 当前是否被持有
IsHeldBy(guy)          -- 当前是否被特定实体持有
IsSheltered()          -- 是否在"避雨/防水"的环境里（container 或防水 inventory）
```

**生命周期回调**
```lua
SetOnDroppedFn(fn)            -- 丢出时调用：fn(inst)
SetOnPutInInventoryFn(fn)     -- 被放入任何 inventory/container 时：fn(inst, owner)
SetOnPickupFn(fn)             -- 被玩家捡起时：fn(inst, pickupguy, src_pos)。返回 true = "我自我销毁了，不要再走默认流程"
SetOnActiveItemFn(fn)         -- V2C 已经标了"Deprecated"，新代码别用
```

**物理 / 落地 / 沉水**
```lua
SetSinks(should_sink)         -- 启动后，物品落到水里会触发 SinkEntity（落地时检查）
SetLanded(is_landed, should_poll_for_landing)  -- 一般由组件自身或 limbo 切换调用
DoDropPhysics(x, y, z, randomdir, speedmult)   -- 给定位置 + 方向 + 速度倍率，丢出时调用的实际物理
ShouldSink()                  -- 当前位置是不是"可沉水点"
TryToSink()                   -- 在落地时调用，如果该沉就一拍 SinkEntity
```

**湿度（自动）**
```lua
EnableMoisture(true/false)    -- 添加 / 移除 inventoryitemmoisture 子组件
GetMoisture() / GetMoisturePercent()
IsWet()
AddMoisture(delta) / DryMoisture()
InheritMoisture(moisture, iswet)
InheritWorldWetnessAtXZ(x, z) / InheritWorldWetnessAtTarget(target)
DiluteMoisture(item, count)
MakeMoistureAtLeast(min)
```

**容器内退出**
```lua
RemoveFromOwner(wholestack, keepoverstacked)
-- 让物品从所在的 inventory/container 中弹出。wholestack=true 会带走整叠。
-- 返回值就是被移除的物品（堆叠的话可能是分裂出的新 entity）
```

##### 3.3 事件流——一个物品的完整一生

下面这段流程把源码里"啥时候推啥事件"画清楚了：

```
SpawnPrefab          → 立刻 OnUpdate 自检（_landed 流程）
玩家走过来拾取        → OnPickup（推送 "onpickup"，调用 onpickupfn）
                       → SetLanded(false, false)
                       → 如果在自燃，停止自燃（burnable:StopSmoldering）+ 拾起者扣血
被放进背包             → OnPutInInventory(owner)
                       → 自动 SetOwner(owner)
                       → HibernateLivingItem（如果有 brain，让它睡）
                       → 推送 "onputininventory"（参数 owner）
                       → 如果自己是 container，给容器内每个 item 推送 "onownerputininventory"
扔出来                 → OnDropped(randomdir, speedmult)
                       → OnRemoved → ReturnToScene
                       → DoDropPhysics
                       → 推送 "ondropped"
                       → 如果自己是 container，给容器内每个 item 推送 "onownerdropped"
                       → 如果还在传火（propagator），延迟 5 秒
最终落地              → SetLanded(true, _)
                       → 推送 "on_landed"
                       → 走 TryToSink，如果该沉就 SinkEntity
                       → SinkEntity 会自动 DropEverything（如果你是装东西的容器）
被删除                 → OnRemoveEntity → 自动从 owner 容器里 RemoveItem（带 ignoreoverstacked）
                       → TheWorld 推送 "forgetinventoryitem"
```

##### 3.4 实战进阶片段

**A. 一把"拿在手里会发光，丢出来就熄灭"的镐——参照 `prefabs/hats.lua:768` 矿工帽**
```lua
local function miner_turnoff(inst)
    if inst.components.fueled ~= nil then
        inst.components.fueled:StopConsuming()
    end
    if inst.components.equippable ~= nil and not inst.components.equippable:IsEquipped() and inst.Light then
        inst.Light:Enable(false)
    end
end

inst.components.inventoryitem:SetOnDroppedFn(miner_turnoff)
```

**B. 一颗"不弹跳、沉水、被丢落时随机方向"的月岩种子——参照 `prefabs/moonrockseed.lua:295`**
```lua
inst.components.inventoryitem.nobounce = true
inst.components.inventoryitem:SetSinks(true)
inst.components.inventoryitem:SetOnDroppedFn(ondropped)
```

**C. 不能被丢入木箱、只能放玩家口袋的徽章**
```lua
inst.components.inventoryitem.canonlygoinpocket = true
```

**D. 只能进玩家口袋 + 玩家口袋容器的迷你物品（如骑士手机、小记事本）**
```lua
inst.components.inventoryitem.canonlygoinpocketorpocketcontainers = true
```

**E. 死亡不掉、被锁定在原槽位的复活怀表（参照 `prefabs/wanda.lua:309`）**
```lua
item.components.inventoryitem.keepondeath = item.prefab ~= "pocketwatch_revive"
```

**F. "我自己是个容器，丢自己时也通知里面的物品"**
> 不用你写，源码已经在 `OnDropped`/`OnPutInInventory` 里自动转发。你只要让自己有 `container` 组件就行：
```lua
inst:AddComponent("container")
inst.components.container:WidgetSetup("backpack")
-- 然后这个背包丢出去时，里面的 log 会自动收到 "onownerdropped"
```

##### 3.5 联机版才会踩到的坑

1. **客户端访问 `inventoryitem`**：客户端的 `inst.components.inventoryitem` 是 `nil`！想在客户端读"能否捡起、是否在持有"等，要用 `inst.replica.inventoryitem`。例如：
   ```lua
   if inst.replica.inventoryitem and inst.replica.inventoryitem:CanBePickedUp(ThePlayer) then ... end
   ```
2. **图集/图像更新走的是 dirty 网络消息**：你在主机端改 `atlasname` 时，会触发 setter `onatlasname` → 走 `replica.inventoryitem:SetAtlas`，再把 `self.classified.atlas` 设脏。客户端收到后通过 `GetAtlas` 才能拿到。**直接改 replica 字段并不会同步回主机**。
3. **`SerializeUsage()` 控制耐久 / 腐烂条**：当你的物品同时有 `finiteuses`/`armor`/`fueled` 时，replica 会自动从最优先的那个里取 `GetPercent()` 同步给客户端的耐久条。这就是为什么物品图标右下角的小条不需要你手写，挂上对应组件就行。

---

#### 四、面向老手——"组件内部到底怎么运作的"

##### 4.1 触发器（setter）链路

构造函数末尾的第三个参数表把若干字段绑定到了 setter（源码第 93 ~ 104 行）：

```lua
{
    atlasname = onatlasname,
    imagename = onimagename,
    owner = onowner,
    canbepickedup = oncanbepickedup,
    cangoincontainer = oncangoincontainer,
    canonlygoinpocket = oncanonlygoinpocket,
    canonlygoinpocketorpocketcontainers = oncanonlygoinpocketorpocketcontainers,
    islockedinslot = onislockedinslot,
    isacidsizzling = onisacidsizzling,
    grabbableoverridetag = ongrabbableoverridetag,
}
```

`Class()` 的第三参数会把这些字段封装成 `__newindex`，**每次给字段赋值都会调用对应的 setter，并把值同步到 replica**。所以：
- 你写 `inst.components.inventoryitem.canbepickedup = false`，主机端立刻调用 `oncanbepickedup(self, false)` → 走 replica 的 `SetCanBePickedUp(false)` → `self._cannotbepickedup:set(true)` → 客户端 UI 立刻同步；
- 但**字段 `keepondeath`、`nobounce`、`trappable`、`sinks`、`droprandomdir` 没有 setter**——它们只在主机端逻辑里被读，所以**只需要在主机端配置一次**，不会同步到客户端（也不需要同步）。

##### 4.2 网络层：`inventoryitem_classified` 是怎么挂上去的

主机端构造 replica 时（`inventoryitem_replica.lua:12 ~ 40`），如果是主机（`TheWorld.ismastersim` 为真），会立刻 `SpawnPrefab("inventoryitem_classified")` 并 `SetParent(inst.entity)`。这个 classified 实体是真正承载所有可同步字段的对象：

- `cangoincontainer`、`canonlygoinpocket`、`canonlygoinpocketorpocketcontainers`、`islockedinslot`、`image`、`atlas`、`src_pos`、`moisture`、`deploymode`、`deployspacing`、`deployrestrictedtag`、`usegridplacer`、`attackrange`、`walkspeedmult`、`equiprestrictedtag` 等都在 classified 里；
- 另外有 3 个**单独挂在 inst 上的 net 变量**（不在 classified 里，因为它们要在客户端"未持有 classified"时也能读到）：`_cannotbepickedup`（`net_bool`）、`_iswet`（`net_bool` 带 dirty event）、`_isacidsizzling`（`net_bool` 带 dirty event）、`_grabbableoverridetag`（`net_hash`）；
- 当物品被丢进容器后，`SetOwner` 会通过 `Network:SetClassifiedTarget(owner)` 把 classified 的可见对象指向"打开这个容器的玩家"——所以同一个箱子里，**只有打开它的人能收到 classified 的细节同步**，没打开就看不到（节省带宽）。

老手特别要看的一段，是 `SetOwner` 的"opencount" 处理（replica 第 171 ~ 191 行）：

```lua
local opencount = owner ~= nil and owner.components.container ~= nil and owner.components.container.opencount or 0
if opencount > 1 then
    self.inst:ForceOutOfLimbo(true)
    if self.inst.Network ~= nil then
        self.inst.Network:SetClassifiedTarget(nil)
    end
    ...
else
    owner = (opencount == 0 and owner) or (opencount == 1 and table.getkeys(owner.components.container.openlist)[1]) or nil
    self.inst:ForceOutOfLimbo(false)
    if self.inst.Network ~= nil then
        self.inst.Network:SetClassifiedTarget(owner)
    end
    ...
end
```

它的含义：
- **0 个人在看**：classified 不广播给任何人；
- **正好 1 个人在看**：把 classified 的 target 设为那个人；
- **多人同时打开**（如多人围观一个共享箱）：把目标设为 `nil`（也就是公开广播给所有客户端），并且把物品 `ForceOutOfLimbo(true)`——让它退出"被容器隐藏"的 limbo 状态。

##### 4.3 落地状态机的全部分支

`SetLanded(is_landed, should_poll_for_landing)` 是这个组件的"心跳"。我们把所有分支列出来（源码 410 ~ 433 行）：

| 输入 | 输出 |
|------|------|
| `is_landed = false, should_poll = true` | `StartUpdatingComponent`（每帧调用 `OnUpdate`，物理引擎来报"我落地了没") |
| `is_landed = false, should_poll = false` | `StopUpdatingComponent`（暂时不轮询） |
| 上一次是 landed，这次 not landed | 推送 `on_no_longer_landed` 事件 |
| `is_landed = true` | `StopUpdatingComponent` + 推送 `on_landed` 事件 + `TryToSink()` |

而 `OnUpdate(dt)`（448 ~ 470 行）的判定逻辑：

```lua
local x, y, z = self.inst.Transform:GetWorldPosition()
if x and y and z then
    if 物理速度无效 / 全 0 then
        SetLanded(true, false)
    elseif y + vely * dt * 1.5 < 0.01 and vely <= 0 then
        SetLanded(true, false)
    end
else
    SetLanded(true, false)
end
```

所以"落地"不等于 y 恰好为 0，而是**下一帧的预测 y 会低于 0.01 且向下运动**。这就是为什么 `nobounce = true` 的物品几乎"瞬间"就 `on_landed`：因为它的 y 速度直接被设为 0（`DoDropPhysics` 第 319 / 324 行）。

##### 4.4 自动加载的子组件 `inventoryitemmoisture`

`EnableMoisture(true)` 会自动 `AddComponent("inventoryitemmoisture")` 并调用 `AttachReplica(self.inst.replica.inventoryitem)`。这意味着所有"非 waterproofer"物品自动获得：
- 0~`TUNING.MAX_WETNESS` 的湿度值；
- 湿度高于阈值时变成 `iswet = true`，推送 `wetnesschange` 事件；
- 客户端通过 `replica.inventoryitem:GetMoisture()` / `:IsWet()` 拿到当前湿度状态；
- 物品图标会自动有"水滴"覆盖（UI 层根据 replica 的 moisture / iswet 渲染）。

如果你做的物品本身是"防水容器"（如 piggyback、krampus_sack），就**不要**让 inventoryitem 启 moisture，而是给容器自己加 `waterproofer` 组件——那么构造时 `EnableMoisture(true)` 会被 `if self.inst.components.waterproofer == nil` 跳过。

##### 4.5 沉水 / 救回路径

落地后 `TryToSink` → `ShouldEntitySink(inst, sinks)` →（来自 `ocean_util.lua:178`）

```lua
function ShouldEntitySink(entity, entity_sinks_in_water)
    local inventory = (entity.components ~= nil and entity.components.inventoryitem) or nil
    if not entity:IsInLimbo() and (not inventory or not inventory:IsHeld()) then
        local px, _, pz = entity.Transform:GetWorldPosition()
        return not TheWorld.Map:IsPassableAtPoint(px, 0, pz, not entity_sinks_in_water)
    end
end
```

**关键点**：`IsPassableAtPoint(px, 0, pz, allow_water)` 的第 4 参数是 "把水也算可通行"。当 `entity_sinks_in_water = false`（默认）时，传入 `not false = true`，水面被算作可通行，所以不沉；当 `entity_sinks_in_water = true` 时，传入 `not true = false`，水面被算成不可通行，于是沉。

然后 `SinkEntity(entity)`（`ocean_util.lua:208`）：
1. 把 entity 上 `inventory`、`container` 里的东西全 DropEverything；
2. 根据所在 tile 选 `splash_sink` / `splash_ocean` / `fallingswish_clouds` 等特效；
3. 如果 entity 带了 `irreplaceable` 或 `shoreonsink` 标签 → 找 `FindRandomPointOnShoreFromOcean` 把它传送到岸边，否则就 `Remove()`。

所以做"不可替代"的物品（如关键剧情道具），**记得加 `inst:AddTag("irreplaceable")` 或 `shoreonsink`**——这两个 tag 由 `SinkEntity` 内部识别（不在 inventoryitem 组件里，但和它形成事实接口）。

##### 4.6 `OnRemoveEntity` 的清理顺序与 `ignoreoverstacked`

```lua
function InventoryItem:OnRemoveEntity()
    if self.owner then
        if self.owner.components.inventory then
            self.owner.components.inventory:RemoveItem(self.inst, true)
        else
            local container = self.owner.components.container
            if container then
                container.ignoreoverstacked = true
                container:RemoveItem(self.inst, true)
                container.ignoreoverstacked = false
            end
        end
    end
    TheWorld:PushEvent("forgetinventoryitem", self.inst)
end
```

这里把 `ignoreoverstacked` 临时打开，是为了**绕过容器在"堆叠时还有溢出"逻辑里的兜底**。
老手在写自己的容器组件 / 移除流程时如果遇到"物品被 Remove 时容器里还残留 stack 引用"，往往就是漏掉了这一对开关。**自己写容器移除时记得参考这段处理。**

最后那条 `TheWorld:PushEvent("forgetinventoryitem", self.inst)` 是给世界级监听器用的——比如有 mod 想统计世界上某个物品的总数、或者某些任务监听"这个物品消失了"，可以监听这个事件，比 ListenForEvent 在物品自身上更稳。

##### 4.7 蜘蛛拾取 = `grabbableoverridetag` 的妙用

`replica.inventoryitem:CanBePickedUp(doer)` 中有这么一段：

```lua
local restrictedtag = self._grabbableoverridetag:value()
if restrictedtag and restrictedtag ~= 0 and doer and doer:HasTag(restrictedtag) then
    return true
end
if self.inst:HasTag("spider") and doer and not doer:HasTag("spiderwhisperer") then
    return false
end
return not self._cannotbepickedup:value()
```

含义：
- 默认蜘蛛（带 `spider` tag）不能被普通玩家捡，但**有 `spiderwhisperer` 的可以**；
- 如果设置了 `grabbableoverridetag = "myspecialtag"`，**带这个 tag 的玩家无视一切限制**直接能捡。

老手做"只有戴某顶帽子的玩家才能拾起的圣物"时，正确做法不是写复杂的 `SetOnPickupFn`，而是：
```lua
inst.components.inventoryitem.canbepickedup = false      -- 普通人捡不起来
inst.components.inventoryitem.grabbableoverridetag = "holy_priest_buff"  -- 带这个 tag 的可以
```

##### 4.8 一些容易看漏的注释

- `SetOnActiveItemFn` 在源码注释里被 V2C（官方程序员代号）显式标了"**Deprecated; please rethink your code if you need to use this**"。新代码遇到要监听"被玩家拖到鼠标光标活跃位"，应该走 `inventory:OnSwitchActiveItem` 或者直接监听玩家事件，不要再用这个旧接口。
- `OnUpdate` 第 451 行的注释里强调"`vx, vy, vz` 任意一项为 nil 也判定为已落地"——这是 Physics 引擎在被销毁中途的兜底，老手在做"自定义飞行物品"时要小心，**不要在飞行中途突然把 Physics 移除**，否则会被 inventoryitem 当作"已经落地"，提前进入 `on_landed`。

---

#### 五、对照源码的核对清单（写完后必检）

| 教程中的论述 | 源码出处（已核对） |
|--------------|--------------------|
| `inventoryitem` 默认在没有 waterproofer 时启用 moisture | `inventoryitem.lua:87 ~ 89` |
| 所有 setter 在赋值时自动同步到 replica | `inventoryitem.lua:1 ~ 39 + 93 ~ 104` |
| `stacksizechange` 事件会被组件代为推送给 owner | `inventoryitem.lua:41 ~ 46` |
| `OnPutInInventory` 内部 `RemoveFromScene` + 给容器子物品推送 `onownerputininventory` | `inventoryitem.lua:245 ~ 264` |
| `OnDropped` 给容器子物品推送 `onownerdropped`、给传火 5 秒延迟 | `inventoryitem.lua:275 ~ 300` |
| `OnPickup` 在自燃时停止自燃并对捡起者扣血 | `inventoryitem.lua:344 ~ 350` |
| `nobounce` 改变 `DoDropPhysics` 的 y 速度为 0 | `inventoryitem.lua:319, 324` |
| `keepondeath` 没有 setter（即不同步到 replica） | 对照构造函数 setter 列表，确认未列入 |
| 客户端要用 `inst.replica.inventoryitem` 而非 `inst.components.inventoryitem` | `inventoryitem_replica.lua` 整个文件，与 `inventoryitem.lua` 客户端不创建组件的事实一致 |
| `SetOwner` 在容器多人打开 / 单人打开 / 无人打开下的 classified target 切换 | `inventoryitem_replica.lua:171 ~ 191` |
| `CanBePickedUp` 中蜘蛛 + `spiderwhisperer` + `grabbableoverridetag` 三段逻辑 | `inventoryitem_replica.lua:88 ~ 97` |
| `ShouldEntitySink` 中 `IsPassableAtPoint(px, 0, pz, not entity_sinks_in_water)` 的反向逻辑 | `ocean_util.lua:178 ~ 184` |
| `SinkEntity` 中 `irreplaceable` / `shoreonsink` 触发岸边复活 | `ocean_util.lua:208 ~ 250`（已读到 235 行确认 tag 检查存在） |
| `MakeInventoryPhysics` 默认 mass=1, rad=0.5, CollisionGroup=ITEMS | `standardcomponents.lua:370 ~ 386` |

> 检查完毕，无与源码冲突的描述。
> 唯一需要提醒读者的"非源码内描述"：
> 1. 23.1 节中 `MakeInventoryFloatable` 由于属于 23.3 / 23.5 才详细展开，这里只列签名做指代；
> 2. 客户端访问字段时需要用 replica 是基于"组件只在 mastersim 构造"的事实推导，不在源码注释里直接说，但和 prefab 模板里 `if not TheWorld.ismastersim then return inst end` 的写法一致。

---

#### 六、本节速查清单（可贴在工作区备忘）

```lua
-- 创建物品最小骨架
inst.entity:AddTransform(); inst.entity:AddAnimState(); inst.entity:AddNetwork()
MakeInventoryPhysics(inst)
inst.entity:SetPristine()
if not TheWorld.ismastersim then return inst end
inst:AddComponent("inventoryitem")

-- 常用调味
inst.components.inventoryitem.atlasname = "images/inventoryimages/xxx.xml"
inst.components.inventoryitem.imagename = "xxx"
inst.components.inventoryitem.nobounce = true       -- 不弹跳
inst.components.inventoryitem.canbepickedup = false -- 暂不可拾起
inst.components.inventoryitem.canonlygoinpocket = true
inst.components.inventoryitem.keepondeath = true    -- 死亡不掉
inst.components.inventoryitem:SetSinks(true)        -- 落水即沉
inst.components.inventoryitem:SetOnDroppedFn(fn)
inst.components.inventoryitem:SetOnPutInInventoryFn(fn)
inst.components.inventoryitem:SetOnPickupFn(fn)     -- 返回 true 表示已自销毁

-- 客户端读取
inst.replica.inventoryitem:CanBePickedUp(ThePlayer)
inst.replica.inventoryitem:IsHeld()
inst.replica.inventoryitem:GetMoisturePercent()

-- 不掉到水里 / 不可替代物品
inst:AddTag("irreplaceable")  -- 或 "shoreonsink"
```




### 23.2 Inventory 组件（玩家侧）——GiveItem、DropItem、Equip 与槽位管理

> 源码：`scripts/components/inventory.lua`（主机端，2548 行） + `scripts/components/inventory_replica.lua`（客户端 replica，610 行）
> 网络实体：`scripts/prefabs/inventory_classified.lua`
> 关联常量：`scripts/constants.lua:648` `MAXITEMSLOTS = 15`、`scripts/constants.lua:650 ~ 656` `EQUIPSLOTS = { HANDS, HEAD, BODY, BEARD }`
> 关联函数：`scripts/gamemodes.lua:311` `GetMaxItemSlots(game_mode)`、`scripts/equipslotutil.lua`、`scripts/components/spdamageutil.lua`

---

#### 一、它到底是什么——一句话定位

`inventory` 是**只挂在"会拿东西的活物"身上**的容器组件，几乎所有玩家、Wickerbottom 的鸟巢、玩家家养的猴、Wanda 的怀表合并体……都靠它存物品、穿装备、拿活跃物品（鼠标拖着的那一格）。

它把"一个活物身上的物品空间"拆成 4 类存储：

```
self.itemslots[slot_num]  --背包栏（1 ~ maxslots，默认 15 格）
self.equipslots[eslot]    --装备栏（hands / head / body / beard 4 个键）
self.activeitem           --鼠标当前拿着 / "活跃物品"（同时只有一件）
GetOverflowContainer()    --背包等装备在 body 槽位、自带 container 的"溢出容器"
```

主机端的 `inventory.lua` 拥有真正的"道具流转 + 伤害分配 + 落水救援 + 死亡掉落"逻辑；客户端的 `inventory_replica.lua` 通过 `inventory_classified` 实体把"槽位里现在是什么物品""装备槽里现在装着什么"同步给所有客户端。

> 注意区分 **`inventory`**（玩家身上拿东西的）和 **`container`**（箱子、冰箱、烹饪锅等独立容器的）。两者大量接口看起来很像，但服务的对象不同：`inventory` 是"主人公"，`container` 是"工具"。`container` 会在 23.5 节细讲。

---

#### 二、面向新手——"我只想给玩家发一个木头"

##### 2.1 最常用三件套：给、丢、装备

```lua
local log = SpawnPrefab("log")
ThePlayer.components.inventory:GiveItem(log)        -- 把木头塞到第一个能放的位置
local axe = SpawnPrefab("axe")
ThePlayer.components.inventory:Equip(axe)           -- 直接装在手上
ThePlayer.components.inventory:DropItem(log, true)  -- 把木头丢出来（带整堆）
```

注意：
- 这三个方法**只能在主机端调用**。客户端不允许直接操作 `components.inventory`，要用网络消息（详见 4.2）。
- `GiveItem(item, slot, src_pos)` 的第二个参数是"建议槽位"，传 `nil` 让组件自己挑空格；第三参 `src_pos` 用来播放"物品从场景某点飞到背包"的 UI 动画。

##### 2.2 三个最常用查询

```lua
local count_logs = ThePlayer.components.inventory:Has("log", 5)
-- 检查身上是否有至少 5 个 log（包括背包、装备栏、活跃物品、溢出容器）
local found, total = ThePlayer.components.inventory:Has("log", 5)
-- found=true/false，total=实际数量

local has_axe = ThePlayer.components.inventory:HasItemWithTag("CHOP_tool")
-- 是否拿了砍树工具

local an_axe = ThePlayer.components.inventory:FindItem(function(it)
    return it.prefab == "axe"
end)
-- 找出第一把斧头实例（包括 overflow）
```

##### 2.3 装备的简单操作

```lua
local hat = ThePlayer.components.inventory:GetEquippedItem(EQUIPSLOTS.HEAD)
if hat then print("玩家头上戴着", hat.prefab) end

ThePlayer.components.inventory:Unequip(EQUIPSLOTS.HANDS) -- 取下手上的物品（回背包）
```

新手要先记住 4 个 equipslot 常量：`HANDS`/`HEAD`/`BODY`/`BEARD`（来自 `constants.lua:650`）。`BEARD` 是给 Webber 蜘蛛胡子专用的，普通玩家用不上但官方留着这个槽位。

##### 2.4 容易踩的 3 个坑

1. **新手最容易写错**：在 mod 里用 `ThePlayer.components.inventory` 修改东西。但 `ThePlayer` 在服务器上是不存在的——服务器是"所有玩家"，要用 `for _, player in ipairs(AllPlayers) do ... end`，或者通过事件参数拿到 `player`。`ThePlayer` 只在客户端有效。
2. **`Equip` 不会从你的背包里偷物品**。你 SpawnPrefab 出来再 Equip，那一定能成功；但 `Equip(已在玩家背包的某件)` 会先把它从背包/箱子里"搬"出来再装上。如果它原本被锁定在某个槽位（`islockedinslot`），会装备失败。
3. **`DropItem(item, true, true)` 的两个 true** 意思是"整堆丢" + "随机方向丢"。如果你只想丢一个，传 `DropItem(item, false)`，组件会调用 `stackable:Get()` 拆出一个再丢。

---

#### 三、面向进阶——"我要让玩家身上有点变化"

##### 3.1 字段全景表（来自源码 `inventory.lua:32 ~ 78` 构造函数）

| 字段 | 默认值 | 类型 | 含义 / 典型用途 |
|------|--------|------|----------------|
| `itemslots` | `{}` | `{ [int]=entity }` | 背包栏。键是 1..maxslots 整数，值是物品实体 |
| `maxslots` | `MAXITEMSLOTS`（=15，可能被游戏模式覆盖） | int | 最大背包格数。**构造完之后会变 read-only**（因为已经同步给 classified） |
| `equipslots` | `{}` | `{ [eslot]=entity }` | 装备栏。键是 `"hands"/"head"/"body"/"beard"` |
| `activeitem` | `nil` | entity | 当前鼠标拿着的物品。同时只有一个 |
| `heavylifting` | `false` | bool | 是否扛着 heavy 物品（如石头雕像）。**有 setter，会同步到 replica** |
| `floaterheld` | `nil`（实际未在构造函数初始化，但通过 `IsFloaterHeld()` 兼容） | bool/nil | 手中是否拿了浮船等大件（如船的零件）。有 setter |
| `isopen` | `false` | bool | 背包 UI 是否打开 |
| `isvisible` | `false` | bool | 背包 UI 是否可见 |
| `ignoreoverflow` | `false` | bool | hack 标志：临时让 GiveItem 不要往背包里塞 |
| `ignorefull` | `false` | bool | hack 标志：满了时不要走 wisecrack / 不丢，只返回 false |
| `silentfull` | `false` | bool | hack 标志：满了不要播音效 / 不要喊话 |
| `ignoresound` | `false` | bool | hack 标志：不要播放 "gotnewitem" 的音效 |
| `acceptsstacks` | `true` | bool | 是否接收可堆叠物。**构造完会 read-only** |
| `ignorescangoincontainer` | `false` | bool | 无视物品的 `cangoincontainer` 限制（如刀具 + 怪物只允许部分容器） |
| `dropondeath` | `true` | bool | 死亡时是否掉所有非 `keepondeath` 物品 |
| `opencontainers` | `{}` | `{ [container_inst]=true }` | 当前打开了哪些容器（箱子、烹饪锅）。注意键是 entity 而不是 number |
| `opencontainerproxies` | `{}` | `{ [proxy_inst]=true }` | 通过 `container_proxy` 间接打开的（合作模式跨人共看一个箱子时用） |
| `isexternallyinsulated` | `SourceModifierList` | object | 外部绝缘度的修饰器（如汪达的某怀表会触发） |
| `force_no_insulation` | `nil` | bool/nil | 强制取消任何绝缘度（电击伤害专用，**不是温度的**） |
| `noheavylifting` | `nil` | bool/nil | 阻止使用任何 heavy 物品 |
| `ignorecombat` | `nil` | bool/nil | 战斗状态忽略标记 |
| `isloading` | `nil` | bool/nil | 在 OnLoad 阶段为 true，结束后置 nil。装备 `IsRestricted_FromLoad` 走这个分支 |
| `HandleLeftoversFn` | `nil` | function | 自定义"满了又拿到东西"的兜底处理。Wanda 等角色用这个 |
| `HandleLeftoversShouldDropFn` | `nil` | function | 在没设置 `HandleLeftoversFn` 时，决定剩余是否应丢出 |

> Setter 注意：只有 `heavylifting` 和 `floaterheld` 有 setter（源码 75 ~ 78 行）。其它字段你赋值时不会自动同步，**要么用方法（如 `EnableDropOnDeath()`），要么改完手动通知 replica**。

##### 3.2 完整方法清单（按用途分组）

**A. 给东西**

```lua
GiveItem(item, slot, src_pos)
    -- 返回值：
    --   number(slot)  : 放进了哪个槽位
    --   true          : 放进了 activeitem 或 overflow / 上一槽位
    --   false         : 放不下（在 ignorefull 模式下）
    --   nil           : 该方法走完没明确 return，意味着塞失败但 wisecrack 已经触发
GiveActiveItem(item)        -- 直接放成 activeitem
TransferInventory(receiver) -- 把 self 的所有内容移交给 receiver.inventory
```

**B. 拿东西**

```lua
GetItemInSlot(slot)          -- 背包某格的物品
GetEquippedItem(eslot)       -- 装备槽
GetActiveItem()              -- 当前活跃
GetFirstItemInAnySlot()      -- 第一个非空格物品
GetItemSlot(item)            -- 反查"这件物品在哪一格"
IsItemEquipped(item)         -- 反查装备槽
GetOverflowContainer()       -- 当前装备在 body 上、自带 container 的那个（背包/盔甲背心）
GetNumSlots()                -- maxslots
```

**C. 找 / 计数**

```lua
Has(prefab, amount, checkallcontainers)            -- 是否至少有 amount 个
HasItemWithTag(tag, amount)                        -- 按 tag 数
HasItemThatMatches(fn, amount)                     -- 按自定义函数判断
GetItemsWithTag(tag)                               -- 返回所有带 tag 的物品（不去重）
GetItemByName(name, amount, checkallcontainers)    -- 找出"x 个名叫 name 的物品 → 哪些实例哪些数量"
GetCraftingIngredient(item, amount)                -- 给制作系统优先级排序后的合成材料表
FindItem(fn)                                       -- 第一个匹配
FindItems(fn)                                      -- 全部匹配
NumItems() / NumStackedItems()                     -- 实际占用槽数 / 总数量（含堆叠）
IsFull()                                           -- 背包栏满
CanAcceptCount(item, maxcount)                     -- 还能再放多少
CanTakeItemInSlot(item, slot)                      -- 某格是否能接收某件物
GetNextAvailableSlot(item)                         -- 下一个可放的槽位，返回 (slot, container 引用)
```

**D. 移除 / 丢**

```lua
RemoveItem(item, wholestack, checkallcontainers, keepoverstacked)
    -- 把一件物品从所有槽位 / 背包 / overflow / 已打开容器中找到并移除，返回实例（堆叠时可能是 stack:Get()）
RemoveItemBySlot(slot, keepoverstacked)             -- 同上但按槽位
DropItem(item, wholestack, randomdir, pos, keepoverstacked)
    -- 把物品丢到地上。pos 可指定丢点，否则丢在玩家脚下
DropActiveItem()                                    -- 把 activeitem 整堆丢出
DropEquipped(keepBackpack, keepPreventUnequipping)  -- 把所有装备丢掉
DropEverything(ondeath, keepequip)                  -- 死亡的"全丢"。ondeath=true 时跳过 keepondeath 物品
DropEverythingWithTag(tag) / DropEverythingByFilter(fn)
DestroyContents(onpredestroyitemcallbackfn)         -- 不掉地，全部 Remove()
ReturnActiveItem(slot, stack_mod)                   -- 把 activeitem 还回背包
ReturnActiveActionItem(item, instant)               -- 玩家点了 action 后，把 activeitem 退回背包（用于 hacks）
ConsumeByName(name, amount)                         -- "吃掉" 若干个，专用于制作系统
```

**E. 装备**

```lua
Equip(item, old_to_active, no_animation, force_ui_anim)
    -- old_to_active : 旧物品是否走 activeitem 而不是塞背包
Unequip(equipslot, slip, force)
    -- force=true 时即使 ShouldPreventUnequipping() 也强卸
SwapEquipment(other, equipslot_to_swap, force)
    -- 跟另一个 inventory 互换装备（Wanda、PVP 等场景）
EquipActiveItem() / EquipActionItem(item) / SwapEquipWithActiveItem()
TakeActiveItemFromEquipSlot(eslot) / TakeActiveItemFromEquipSlotID(eslotid)
SelectActiveItemFromEquipSlot(slot)
HasAnyEquipment() / IsWearingArmor() / ArmorHasTag(tag) / EquipHasTag(tag) / EquipHasSpDefenseForType(sptype)
```

**F. 战斗 / 防护**

```lua
ApplyDamage(damage, attacker, weapon, spdamage)
-- 由 combat 组件调用：让身上的护甲、抗性、特殊伤害防御一齐参与吸伤。
-- 返回剩余的普通 damage 和（被部分吸收后的）spdamage 表
IsInsulated() / ForceNoInsulated(force)  -- 电击防护
GetEquippedMoistureRate(slot)            -- 装备的吸水率
GetWaterproofness(slot) / IsWaterproof() -- 防水度
```

**G. 状态机 / UI**

```lua
Open() / Close(keepactiveitem) / Show() / Hide()
IsOpenedBy(guy)
CloseAllChestContainers()                 -- 把所有打开的 chest 关掉（保留烹饪锅之类）
GetOpenContainerProxyFor(master)          -- 多人共用容器时的代理查找
```

**H. UI 点击（22+ 个）**

```lua
PutOneOfActiveItemInSlot / PutAllOfActiveItemInSlot
TakeActiveItemFromHalfOfSlot / TakeActiveItemFromCountOfSlot(slot, count) / TakeActiveItemFromAllOfSlot
AddOneOfActiveItemToSlot / AddAllOfActiveItemToSlot
SwapActiveItemWithSlot
CombineActiveStackWithSlot(slot, stack_mod)
SelectActiveItemFromSlot
MoveItemFromAllOfSlot(slot, container) / MoveItemFromHalfOfSlot / MoveItemFromCountOfSlot
UseItemFromInvTile(item, actioncode, mod_name)
ControllerUseItemOnItemFromInvTile / ControllerUseItemOnSelfFromInvTile / ControllerUseItemOnSceneFromInvTile
InspectItemFromInvTile / DropItemFromInvTile / CastSpellBookFromInv
```

> 这一组对应 UI 上每一种点击 / 拖拽 / 右键的入口，**模组里写自定义动作时不需要重新发明轮子**：让 UI 调用对应的方法，主机端会处理好同步。

##### 3.3 事件全景（按推送顺序排）

| 事件 | 推送时机 | 数据键 | 监听者 |
|------|----------|--------|--------|
| `gotnewitem` | `GiveItem` 成功把"新物品"放到某槽位（或合并到 activeitem） | `{ item, slot, src_pos }` 或 `{ item, toactiveitem=true }` | UI 提示音、刻字本、统计 |
| `itemget` | 物品真正进入背包格 / activeitem 后 | `{ item, slot, src_pos }`（slot=nil 表示 activeitem） | UI 槽位刷新、replica 同步 |
| `newactiveitem` | activeitem 改变（包括变 nil） | `{ item }` | UI 鼠标拖动状态、replica 同步 |
| `itemlose` | 从背包 / activeitem / equipslot 中移除一件 | `{ slot=k, prev_item }` 或 `{ activeitem=true, prev_item }` | UI 槽位刷新、replica 同步 |
| `equip` | 装备成功后 | `{ item, eslot, no_animation }` | 装备 UI、状态机播放装备动作 |
| `unequip` | 卸下成功 | `{ item, eslot, slip }` | UI、状态机 |
| `setoverflow` | 装备 body 槽位（如背包）后，overflow 切换 | `{ overflow=item }` 或 `{}`（取下时） | UI 重建溢出容器 |
| `dropitem` | DropItem 成功 | `{ item }` | 成就、任务、AI |
| `inventoryfull` | 满了塞不下且未走兜底处理时 | `{ item }` | 语音 wisecrack |
| `death`（监听） | 玩家死亡时 inventory 自己监听 | — | 触发 `DropEverything(true)` |
| `player_despawn`（监听） | 玩家断线 / 切角色，inventory 自己监听 | — | 给所有持有物品推送 `player_despawn` |

##### 3.4 实战进阶片段

**A. 给玩家发一组初始物资（参照常见角色 mod）**
```lua
local function GiveStarterItems(player)
    if player.components.inventory then
        local items = { "log", "log", "log", "twigs", "twigs", "axe" }
        for _, name in ipairs(items) do
            local item = SpawnPrefab(name)
            if item then
                player.components.inventory:GiveItem(item)
            end
        end
    end
end
```

**B. 在玩家手上还有空时，强制把某物品塞到手里**
```lua
-- 注意 Equip 会自动把旧装备塞回背包 / 丢地
local sword = SpawnPrefab("xd_sj_bglxp")  -- 假设这是一把神武器
ThePlayer.components.inventory:Equip(sword)
```

**C. 安全地"消耗"某种材料**
```lua
local enough, total = ThePlayer.components.inventory:Has("log", 3, true) -- 第3参=查所有打开容器
if enough then
    ThePlayer.components.inventory:ConsumeByName("log", 3)
    -- 注意 ConsumeByName 内部用 v.skinname 不区分皮肤；这正是合成材料的语义
end
```

**D. 监听"玩家身上拿到 / 失去某物品"（很常用）**
```lua
local function OnGotNewItem(player, data)
    if data.item and data.item.prefab == "redgem" then
        print("玩家拿到了一颗红宝石！")
    end
end

local function OnItemLose(player, data)
    if data.prev_item and data.prev_item.prefab == "redgem" then
        print("玩家失去了一颗红宝石！")
    end
end

ThePlayer:ListenForEvent("gotnewitem", OnGotNewItem)
ThePlayer:ListenForEvent("itemlose", OnItemLose)
```

**E. 在打开了某个箱子的情况下，让玩家"快速合成"也能看到箱子里的木头**
```lua
local enough, total = player.components.inventory:Has("log", 4, true) -- checkallcontainers=true
-- 内部会遍历 player.components.inventory.opencontainers
```

**F. Wanda 那种 "保留特定物品不掉" 的写法**
```lua
-- 参照 prefabs/wanda.lua:309
item.components.inventoryitem.keepondeath = item.prefab ~= "pocketwatch_revive"
-- 然后 inventory:DropEverything(true, ...) 时会跳过 keepondeath=true 的物品
```

**G. 自定义 "满了怎么办"（参照部分角色）**
```lua
player.components.inventory.HandleLeftoversFn = function(player, item)
    -- 例如：自动把多出来的物品扔进随身仓库
    if player.components.mybonusstorage then
        player.components.mybonusstorage:Stash(item)
    end
end

-- 或者只决定"满了要不要丢"：
player.components.inventory.HandleLeftoversShouldDropFn = function(player, item)
    return not item:HasTag("preventdropalwaystry")
end
```

##### 3.5 联机版才会踩到的坑

1. **客户端的 `components.inventory` 在普通玩家身上是 nil**——玩家自己的客户端会有"自身 inventory_classified"挂上 replica，但服务器只通过 replica 提供有限信息。所以：
   - 客户端要读"我当前手里有什么"，用 `ThePlayer.replica.inventory:GetActiveItem()`、`:GetItemInSlot(slot)`；
   - 客户端**不能**调用 `Equip`、`GiveItem` 这种修改方法——它们没暴露给客户端。要让玩家做动作，发送 `SendRPCToServer` / 通过 `playercontroller` 走 action。
2. **`maxslots / acceptsstacks / ignorescangoincontainer` 在构造完会被 `makereadonly`**：
   ```lua
   if inst.replica.inventory.classified ~= nil then
       makereadonly(self, "maxslots")
       makereadonly(self, "acceptsstacks")
       makereadonly(self, "ignorescangoincontainer")
   end
   ```
   一旦 classified 创建（也就是玩家有 player tag），你想改这些字段会触发 lua 报错。**如果你要做"可变格数玩家"，必须在添加 component 时就改，或者去 inventory_classified.lua 改 maxslots 的初始值。**
3. **OnLoad 期间 `isloading = true`**，此时 `Equip` 走 `IsRestricted_FromLoad` 分支，可以让"读档恢复装备"绕过某些活跃状态判断。**自定义装备的 mod 应该提供 `equippable.restricted_load_fn` 而不仅是 `restricted_fn`**，否则读档会装不回来。
4. **GiveItem 满了**不一定立刻丢——它有 3 级兜底：
   - 没设过 `HandleLeftoversFn`：试着合并到 activeitem，合并不上才 `DropItem`；
   - 设了 `HandleLeftoversFn`：交给你的函数；
   - 设了 `HandleLeftoversShouldDropFn`：让函数决定丢不丢。

---

#### 四、面向老手——"组件内部机制 / 网络协议 / 边界条件"

##### 4.1 GiveItem 的完整决策树

这是 inventory 组件最复杂的一段（源码 946 ~ 1111 行）。流程总结成决策树：

```
GiveItem(inst, slot, src_pos)
├── inst 没有 inventoryitem 或失效 → return（打印 Warning）
├── 如果 inst 当前已装备 → Unequip 它的槽位
├── 如果 inst 当前 owner ~= self.inst → RemoveFromOwner(true)
├── OnPickup → 物品自销毁则 return
├── 计算 slot：
│   ├── 没传 slot 且 prevslot 存在（无 prevcontainer）→ 用 prevslot
│   ├── 没传 slot 且 prevcontainer 存在（是当前自己打开的容器）→ 尝试塞回 prevcontainer 同槽位
│   ├── 否则 → GetNextAvailableSlot(inst)
├── 如果有 slot：
│   ├── 容器是 overflow → 塞 overflow，可能产生 leftovers（递归 GiveItem）
│   ├── 容器是 equipslots → 同上
│   ├── 容器是 itemslots：
│   │   ├── 该槽已有物品 → stackable:Put → 可能 leftovers
│   │   ├── 该槽为空 → itemslots[slot]=inst, OnPutInInventory, PushEvent("itemget")
├── 没找到 slot：
│   ├── 可放 activeitem（activeitem 当前空、maxslots>0、物品不限"口袋"、未挂手柄）→ SetActiveItem(inst)
│   ├── 否则 HandleLeftoversFn 存在 → 让它处理
│   ├── 否则 → 尝试和当前 activeitem 合并堆叠 → 不行就 DropItem
└── 最后：PushEvent("inventoryfull", { item = inst })（除非 silentfull / maxslots=0）
```

**老手要注意**：
- `prevslot`/`prevcontainer` 是物品上挂的"上次所在位置"线索——它在 `Equip`、`RemoveItem` 等多处会被设置，目的是"让玩家点击装备后，卸下时回到原槽位"。**自己写脚本移动物品时记得维护这两个字段**，否则用户体验会变差。
- "口袋限制"（`canonlygoinpocket`/`canonlygoinpocketorpocketcontainers`）在多处被检查，特别注意源码 818 ~ 820 行那段：
  ```
  not item.components.inventoryitem.canonlygoinpocket and
  (not item.components.inventoryitem.canonlygoinpocketorpocketcontainers or 
       overflow.inst.components.inventoryitem and overflow.inst.components.inventoryitem.canonlygoinpocket)
  ```
  含义：物品标了"只能口袋"就跳过 overflow；标了"口袋+口袋容器"则只有 overflow 自己也是"口袋限制"的容器才允许。
- `inventoryfull` 事件在 `silentfull / isloading / maxslots=0` 时**不会**推送，所以你监听 inventoryfull 时要注意，不能依赖它捕捉所有"满载"情况。

##### 4.2 客户端如何"操作"自己的背包

在联机版中，客户端没有 `components.inventory`，但还是可以通过 replica 触发动作。看 `inventory_replica.lua:482 ~ 606` 这一长段：

```lua
function Inventory:UseItemFromInvTile(item)
    if item == nil or not item:IsValid() then return
    elseif self.inst.components.inventory ~= nil then
        self.inst.components.inventory:UseItemFromInvTile(item)
    elseif self.classified ~= nil then
        self.classified:UseItemFromInvTile(item)
    end
end
```

模式很清楚：**有主机权限（components 存在）就直接调用主机端；没有就让 classified 发 RPC**。`inventory_classified.lua`（位于 `prefabs/inventory_classified.lua`）里这些方法会 `SendRPCToServer`，服务器收到后再调用主机端的 inventory。

**老手实践**：自定义 UI 按钮要让客户端触发某个背包操作时，**不要**直接读写 components；调 `inst.replica.inventory:XxxFromInvTile()` 即可，组件会自动处理客户端 → RPC → 主机端的转换。

##### 4.3 装备的"路径"：`Equip` 第 1160 ~ 1282 行的边界条件

`Equip` 是另一段密集逻辑：

```
Equip(item, old_to_active, no_animation, force_ui_anim)
├── item 不可装备 / IsRestricted / heavy 但 noheavylifting → return
├── 老物品 olditem = equipslots[eslot]
├── olditem.ShouldPreventUnequipping → return（被锁定的装备）
├── 控制器模式 + 老物品存在 + item.prevslot → 记录 olditem 的 prevslot
├── 特殊处理 "手 vs heavy 物品"：
│   ├── 装手且身上扛着 heavy → DropItem(heavy)
│   ├── 装 body 的 heavy 物品且手上有东西 → 把手上的塞背包或丢出
├── 拆出 item（如果是堆叠）：
│   ├── inventoryitem == nil → RemoveItem(item, equipstack)
│   ├── IsHeld → RemoveFromOwner(equipstack)
│   ├── 堆叠且不允许 equipstack → leftovers = item, item = stackable:Get()
├── item == activeitem → leftovers = activeitem, SetActiveItem(nil)
├── olditem ~= item:
│   ├── leftovers → 走 GiveActiveItem 或 silentfull GiveItem
│   ├── olditem → Unequip + ToPocket → 不可入容器则 DropItem，否则 GiveActiveItem 或 silentfull GiveItem
│   ├── item.OnPutInInventory(self.inst)
│   ├── item.equippable:Equip(self.inst, not old_to_active and item.prevslot == nil)
│   ├── equipslots[eslot] = item
│   ├── 设置 heavylifting / floaterheld
│   ├── PushEvent("equip", { item, eslot, no_animation })
│   ├── 统计 + 套装加成
│   └── return true
```

**最难发现的坑**：第 1268 行 `self.floaterheld = item.components.playerfloater and true`。这里 `item.components.playerfloater` 只在主机端有效，但 setter `onfloaterheld` 会把它同步到客户端 replica。一些"做手浮船 / 划船道具"的 mod 漏掉了挂 `playerfloater` 组件，结果服务端 floaterheld 一直是 nil，客户端永远走不进"水中可装备"逻辑。

##### 4.4 `ApplyDamage` 的伤害分配算法

源码 399 ~ 491 行，这是一段对老手特别有用的算法：

1. 遍历所有 equipslots，先检查 `resistance` 组件——任何一件装备能完全抵抗这次攻击，**立刻返回 0**（连普通伤害都吃掉）。
2. 其它装备记录到 `absorbers` 表中，并记录 `damagetyperesist`（普通伤害的乘性减免）。
3. 计算 `damage *= damagetypemult`（多个 damagetyperesist 累乘）。
4. 计算 `absorbed_percent`：**取所有 armor 中最高的吸收百分比**（不是相加）。
5. 计算 `absorbed_damage = damage * absorbed_percent`。
6. **按 armor 提供的百分比比例分摊吸收量**：`armor_damage[armor] = absorbed_damage * amt / total_absorption + bonus_damage`。
7. 处理 spdamage（特殊伤害）：每种类型独立分摊到所有"对该 sptype 有 defense 的装备"上。
8. 最后调用各装备的 `armor:TakeDamage(dmg)` 扣耐久。

**关键洞察**：吸伤百分比是"取最大"而不是"求和"，所以叠 2 件 50% 护甲 ≠ 100% 减伤，而是 = 50% 减伤；但磨损会均摊到 2 件上。**这是饥荒的官方设计，写护甲 mod 时不能想当然以为可以无限叠。**

##### 4.5 `Hide` vs `Close` vs `Open` vs `Show` 状态机

| 方法 | 触发 | isopen 变化 | isvisible 变化 | 关闭打开的容器？ | replica 通知 |
|------|------|-------------|----------------|------------------|--------------|
| `Open()` | 玩家创建 / 解死 | false→true | false→true | 自动 Open overflow | `OnOpen` |
| `Show()` | 复活 / 通过特殊状态 | 不变 | false→true | 不动 | `OnShow` |
| `Hide()` | 进入幽灵 / 苏醒过程 | 不变 | true→false | **关掉所有 opencontainers（除 overflow 和 stay_open_on_hide）** | `OnHide` |
| `Close(keepactiveitem)` | 玩家彻底退出 | 任意→false | 任意→false | **关掉所有 opencontainers** | `OnClose` |

> 特别小心 `Hide()` 第 1934 ~ 1938 行：会自动关掉所有打开的 container（除非该 container 有 `stay_open_on_hide` 字段）。所以做"睡眠时仍能打开的特殊容器"，要在 container 上加 `stay_open_on_hide = true`。

##### 4.6 `inventory_classified` 同步的 5 个事件钩子

`replica` 构造函数（21 ~ 26 行）挂了 5 个事件 → 把数据同步给 classified：

```lua
inst:ListenForEvent("newactiveitem", function(inst, data) self.classified:SetActiveItem(data.item) end)
inst:ListenForEvent("itemget", function(inst, data) self.classified:SetSlotItem(data.slot, data.item, data.src_pos) end)
inst:ListenForEvent("itemlose", function(inst, data) self.classified:SetSlotItem(data.slot) end)
inst:ListenForEvent("equip", function(inst, data) self.classified:SetSlotEquip(data.eslot, data.item) end)
inst:ListenForEvent("unequip", function(inst, data) self.classified:SetSlotEquip(data.eslot) end)
```

**老手用法**：如果你做 mod 时想加新的"槽位类型"（比如戒指槽），单独挂事件让 classified 多记一个字段就够了；不要去动 inventory.lua 主逻辑。

##### 4.7 `OnOwnerDespawned` 的"通知所有物品"

源码第 10 ~ 22 行：

```lua
local function OnOwnerDespawned(inst)
    if inst.components.inventory ~= nil then
        for slot, item in pairs(inst.components.inventory.itemslots) do
            item:PushEvent("player_despawn")
        end
        for slot, equip in pairs(inst.components.inventory.equipslots) do
            equip:PushEvent("player_despawn")
        end
        if inst.components.inventory.activeitem ~= nil then
            inst.components.inventory.activeitem:PushEvent("player_despawn")
        end
    end
end
```

**用途**：让物品在玩家断线时做"自我清理"。比如挂在玩家身上的"心灵共鸣物品"在玩家离开时把数据同步给世界。
如果你写的物品想做这种处理，监听 `player_despawn` 而不是去玩家身上挂监听——这样不需要管什么时候挂、什么时候卸。

##### 4.8 `TransferInventory` + `SwapEquipment` 的"换体"机制

`TransferInventory(receiver)` 把背包、装备、activeitem 都给 receiver 的 inventory。中间有几个特别注意：
- `if item.persists`：临时性 / 不持久化的物品（如某些 NPC 的内置武器）**不转移**。这是为了避免 Wanda 等用 transferinventory 把 NPC 限定武器拿过来。
- `equip.components.equippable:IsRestricted(receiver)`：装备如果被新主人 restrict 不让用，就退化成"塞背包"。

`SwapEquipment` 是另一种"互换"，专用于玩家之间换装备槽（如 PVP、变身机制）。所有 equip slot 一次性互换，遇到 restrict 同样退化为给背包。

##### 4.9 `MoveItemFromXxxOfSlot` 与 `container_proxy`

这一族方法（2370 ~ 2484 行）专门处理"玩家把背包槽位的东西移到另一个容器（如箱子）"。它们在内部解决两个问题：
1. 找到正确的目标容器（如果是 `container_proxy`，要通过 `GetOpenContainerProxyFor` 找到真实容器）；
2. 处理堆叠合并 / 容器满载兜底。

**老手实践**：写 UI 自定义"一键存货"，调用这些方法即可：
```lua
inv:MoveItemFromAllOfSlot(slot, target_container)
```
内部自动处理 RPC + 同步，不需要你自己管。

##### 4.10 容易看漏的注释 / Hack 标志

源码留了好几个 hack 标志，老手要看明白：

| 标志 | 含义 | 何时启用 |
|------|------|----------|
| `ignoreoverflow` | GiveItem 跳过 overflow | `ReturnActiveActionItem` 中使用，避免活跃物品被塞进背包 |
| `ignorefull` | GiveItem 满时返回 false 不丢 | 同上，避免错误掉落 |
| `silentfull` | 满时不 wisecrack | Equip 中"把旧物品塞背包"时使用，避免说话 |
| `ignoresound` | gotnewitem 不播音效 | `PutOneOfActiveItemInSlot` 等 UI 操作 |

这些标志总是成对开/关，**千万不要在自己 mod 里只开不关**，否则全局状态会被污染。

---

#### 五、对照源码的核对清单

| 教程论述 | 源码出处（已核对） |
|----------|--------------------|
| `inventory` 把存储分成 itemslots / equipslots / activeitem / overflow | `inventory.lua:32 ~ 78` 构造函数 |
| maxslots 默认 15、来自 `GetMaxItemSlots(TheNet:GetServerGameMode())` | `inventory.lua:45` + `gamemodes.lua:311` + `constants.lua:648 MAXITEMSLOTS=15` |
| EQUIPSLOTS 共 4 个常量 | `constants.lua:650 ~ 656` |
| GiveItem 满了的 3 级兜底（HandleLeftoversFn / 合并 activeitem / DropItem） | `inventory.lua:1064 ~ 1110` |
| GiveItem 中 prevslot/prevcontainer 的位置恢复逻辑 | `inventory.lua:979 ~ 1001` |
| Equip 中堆叠拆分、heavy 物品自动让位、leftovers 回流 | `inventory.lua:1199 ~ 1281` |
| Unequip 把 floaterheld/heavylifting 置回 | `inventory.lua:1127 ~ 1131` |
| ApplyDamage 算法（resistance 完全抵消、取最大百分比、按比例分摊耐久、spdamage 独立分摊） | `inventory.lua:399 ~ 491` |
| Hide() 关掉所有除 overflow / stay_open_on_hide 的容器 | `inventory.lua:1934 ~ 1938` |
| OnOwnerDespawned 给所有物品推送 player_despawn | `inventory.lua:10 ~ 22` |
| isexternallyinsulated 用 SourceModifierList(false, boolean) | `inventory.lua:60` |
| maxslots/acceptsstacks/ignorescangoincontainer 在有 classified 时变 read-only | `inventory.lua:68 ~ 72` |
| 客户端 replica 通过 5 个事件钩子同步给 classified | `inventory_replica.lua:21 ~ 26` |
| 客户端调用方法走 components or classified 两条路径 | `inventory_replica.lua:482 ~ 606` 整段 |
| `inventory:DropEverything(true)` 在死亡 / 幽灵模式不允许时被强制为 false | `inventory.lua:1731 ~ 1734` |
| inventoryfull 在 silentfull / isloading / maxslots=0 不推送 | `inventory.lua:1107 ~ 1109` |

> 14 条核对全部通过，无与源码冲突的描述。
> 已规避的非源码描述：
> - 23.2 中 `container` 组件的细节延后到 23.5；
> - `container_proxy` 是较晚版本机制，**只在源码 2540 ~ 2546 行**有方法暴露，本节只点到为止；
> - `equippable.restricted_load_fn` 是 inventory.lua:1164 中 `IsRestricted_FromLoad` 的依据，需要在 23.4 进一步展开。

---

#### 六、本节速查清单

```lua
-- 给 / 丢 / 装备
local item = SpawnPrefab("log")
player.components.inventory:GiveItem(item, slot, src_pos)
player.components.inventory:DropItem(item, wholestack, randomdir, pos)
player.components.inventory:Equip(item)
player.components.inventory:Unequip(EQUIPSLOTS.HANDS)

-- 查询
player.components.inventory:Has("log", 5)
player.components.inventory:HasItemWithTag("CHOP_tool")
player.components.inventory:FindItem(function(it) return it.prefab == "axe" end)
player.components.inventory:GetEquippedItem(EQUIPSLOTS.HEAD)
player.components.inventory:GetItemInSlot(1)
player.components.inventory:NumItems()

-- 消耗（合成系统专用）
player.components.inventory:ConsumeByName("log", 3)

-- 装备 / 战斗
local dmg_left, sp_left = player.components.inventory:ApplyDamage(50, attacker, weapon, spdamage)
local insulated = player.components.inventory:IsInsulated()
local waterproof = player.components.inventory:GetWaterproofness()

-- 监听
player:ListenForEvent("gotnewitem", function(p, data) ... end)
player:ListenForEvent("itemlose", function(p, data) ... end)
player:ListenForEvent("equip", function(p, data) ... end)
player:ListenForEvent("unequip", function(p, data) ... end)
player:ListenForEvent("inventoryfull", function(p, data) ... end)
player:ListenForEvent("dropitem", function(p, data) ... end)

-- 客户端读取（注意是 replica）
local active = ThePlayer.replica.inventory:GetActiveItem()
local slot1 = ThePlayer.replica.inventory:GetItemInSlot(1)
local hat = ThePlayer.replica.inventory:GetEquippedItem(EQUIPSLOTS.HEAD)
local has_log = ThePlayer.replica.inventory:Has("log", 1)

-- 死亡掉落定制
item.components.inventoryitem.keepondeath = true        -- 不掉
player.components.inventory.HandleLeftoversFn = fn       -- 满了的兜底
```




### 23.3 Stackable——堆叠逻辑

> 源码：`scripts/components/stackable.lua`（主机端，212 行） + `scripts/components/stackable_replica.lua`（客户端 replica，152 行）
> 关联常量：`scripts/tuning.lua:81 ~ 85` `STACK_SIZE_LARGEITEM = 10 / MEDITEM = 20 / SMALLITEM = 40 / TINYITEM = 60 / PELLET = 120`
> 关联组件：`scripts/components/perishable.lua:168`（`Perishable:Dilute`）、`scripts/components/inventoryitemmoisture.lua:127`（`DiluteMoisture`）、`scripts/components/edible.lua:244`（`DiluteChill`）
> 关联调用方：`scripts/components/inventory.lua`（`GiveItem`、`CombineActiveStackWithSlot`、`TakeActiveItemFromCountOfSlot`）

---

#### 一、它到底是什么——一句话定位

`stackable` 是饥荒里**让一件物品可以"几个一格"**的组件。木头、草、燧石、宝石、肉，背包里都是 X/40、X/20、X/10 那种"小数字角标"——这个角标完全由 stackable 提供。**没有这个组件 = 这件东西永远占一整格、永远是 1 个**。

它干两件最核心的事情：
1. 维护 `stacksize`、`maxsize` 两个数字，并在每次变化时推送 `stacksizechange` 事件、同步到客户端 UI；
2. 提供 `Put(item)` / `Get(num)` 两个原子操作，让 inventory、container、合成、烹饪、掉落等系统能"合并 / 拆分一堆"。

主机端的 `stackable.lua` 持有真正的数字和逻辑；客户端的 `stackable_replica.lua` 通过 4 个网络变量把堆叠大小、上限、是否破上限同步给所有客户端，并把容量编码进 14 位（最大 4096）实现节省带宽。

---

#### 二、面向新手——"我只想做一个能堆 40 个的草"

##### 2.1 最小可工作示例（同 `prefabs/cutgrass.lua` 同构）

```lua
inst.entity:AddTransform(); inst.entity:AddAnimState(); inst.entity:AddNetwork()
MakeInventoryPhysics(inst)
inst.entity:SetPristine()
if not TheWorld.ismastersim then return inst end

inst:AddComponent("inspectable")
inst:AddComponent("inventoryitem")

inst:AddComponent("stackable")
inst.components.stackable.maxsize = TUNING.STACK_SIZE_SMALLITEM   -- 一格最多 40
```

##### 2.2 五个内置上限常量——记住"按物品体积选"

来自 `tuning.lua:81 ~ 85`：

| 常量 | 数值 | 典型适用 |
|------|------|----------|
| `TUNING.STACK_SIZE_LARGEITEM` | 10 | 海带根、煮熟的肉串、瓶中信、马蹄铁 |
| `TUNING.STACK_SIZE_MEDITEM` | 20 | **默认值**。木头、活木、宝石、煮蛋、蘑菇等 |
| `TUNING.STACK_SIZE_SMALLITEM` | 40 | 草、燧石、种子、噩梦燃料、齿轮、纸 |
| `TUNING.STACK_SIZE_TINYITEM` | 60 | 嘉年华奖券 |
| `TUNING.STACK_SIZE_PELLET` | 120 | 小颗粒（铅弹之类） |

> 新手的疑问："我直接写 `inst.components.stackable.maxsize = 99` 行吗？"
> 行，但**不推荐**。客户端 replica 用 `STACK_SIZE_CODES[maxsize]` 反查上限（`stackable_replica.lua:9` 把 5 个 TUNING 值反着做了字典），写一个不在表里的数会让 `SetMaxSize` 抛 nil 错。**自定义上限请用 `SetIgnoreMaxSize(true)` 或加新常量到 TUNING 里**（mod 一般用前者）。

##### 2.3 默认值速查

```lua
inst.components.stackable.stacksize  -- 默认 1
inst.components.stackable.maxsize    -- 默认 TUNING.STACK_SIZE_MEDITEM (=20)
```

新手只要 `AddComponent("stackable")` 这一行，物品就是"中等堆叠 20 个一格"。**绝大多数情况不需要再写第二行**。

##### 2.4 新手版的 3 个常用查询

```lua
inst.components.stackable:IsStack()       -- stacksize > 1?
inst.components.stackable:StackSize()     -- 当前堆里几个
inst.components.stackable:IsFull()        -- 已经满 maxsize 了?
```

##### 2.5 新手最容易犯的 4 个错

1. **客户端读不到** `inst.components.stackable`。客户端的 `inst.components.stackable` 是 nil。要在客户端读数量，得用 `inst.replica.stackable:StackSize()`。
2. **把 stackable 加在 `if not TheWorld.ismastersim then return inst end` 之前**——客户端会尝试 `AddComponent("stackable")`，但因为 components 是主机端独有，会报"component not found"错。永远在 mastersim 分支后才加组件。
3. **想做一个"堆叠且能装备"的武器**——装备组件 `equippable` 默认 `equipstack = false`，意思是"装备时只装一个，剩下塞背包"。如果想做"投掷飞镖（堆叠 + 装备整堆）"，要 `inst.components.equippable.equipstack = true`，否则 Equip 会自动 `Get()` 拆出一个。
4. **`maxsize = X` 之后想再改成 ignore**——直接赋值 `maxsize = math.huge` 会被 setter 截胡（见 4.3）。要用 `SetIgnoreMaxSize(true)`。

---

#### 三、面向进阶——"我要让物品的堆叠有点性格"

##### 3.1 字段表（来自源码 `stackable.lua:23 ~ 34` 构造 + 部分动态字段）

| 字段 | 默认值 | 类型 | 含义 / 用途 |
|------|--------|------|-------------|
| `stacksize` | `1` | int | 当前堆里多少个。**有 setter（`onstacksize`），赋值会自动同步到 replica + 推 `stacksizechange` 事件** |
| `maxsize` | `TUNING.STACK_SIZE_MEDITEM` | int | 上限。**有 setter（`onmaxsize`），如果当前处于"忽略上限"状态，赋值会被劫持到 originalmaxsize 中** |
| `originalmaxsize` | `nil` | int | "忽略上限"前的旧 maxsize 备份。**用 `makereadonly` 保护，禁止外部直接赋值**——只能通过 `SetIgnoreMaxSize` 间接维护 |
| `ondestack` | `nil` | function | `Get` 拆分新堆时调用 `fn(new_instance, original)`，让 mod 把额外字段从老堆复制到新堆 |
| `inst.stackable_CanStackWithFn` | `nil` | function on **entity** | 注意：**挂在 `inst` 上而不是组件上**！由 `CanStackWith` 调用，让物品自己声明"我和它能否合并"。返回 false 阻止合并 |

> Setter 提醒：`stacksize` / `maxsize` 这两个字段在赋值时会经过 `Class()` 的 `__newindex` 拦截，调用 `onstacksize` / `onmaxsize`。所以 `inst.components.stackable.stacksize = 5` 这一行会：
> 1. 写值到内部表；
> 2. 推 `stacksizechange` 事件；
> 3. 调 `replica.stackable:SetStackSize(5)` 同步给客户端；
> 4. 顺带把 `replica.inventoryitem:SetPickupPos(_src_pos)` 一起同步（让捡起飞行动画有起点）。

##### 3.2 方法清单（按用途分组）

**A. 查询**
```lua
IsStack()           -- stacksize > 1
IsFull()            -- stacksize >= maxsize（注意 ignoremaxsize 时 maxsize = math.huge）
IsOverStacked()     -- 超过 originalmaxsize（仅在 ignoremaxsize 模式下可能为 true）
StackSize()         -- 当前堆里几个
RoomLeft()          -- maxsize - stacksize
CanStackWith(item)  -- 三层判断：prefab 相同 + skinname 相同 + stackable_CanStackWithFn 不否决
GetDebugString()    -- "5/20" 或 "30/--(20)" 这种 debug 输出
```

**B. 改大小**
```lua
SetStackSize(sz)    -- 强制设置（含 MAXUINT 上限保护），推 stacksizechange 事件
SetIgnoreMaxSize(b) -- 切换"无视上限"模式，详见 3.4
```

**C. 合并 / 拆分**
```lua
Put(item, source_pos)
-- 把 item 合并进 self。如果 self 装不下，把多出的部分留在 item 里返回（leftovers）
-- source_pos 用于"动画起点"，让客户端 UI 显示飞到背包的动效

Get(num)
-- 从 self 中取 num 个（默认 1）。
-- 如果 self.stacksize > num: 创建一个新实例，把 num 个移过去，返回新实例
-- 如果 self.stacksize <= num: 直接返回 self（一整堆带走）
-- ondestack 会在"拆出新实例"时被调用
```

**D. 持久化 / 回调**
```lua
SetOnDeStack(fn)         -- 设置 ondestack 回调
OnSave()                 -- 只在 stacksize > 1 时写 { stack = stacksize }
OnLoad(data)             -- 恢复并推 stacksizechange（oldstacksize = 1）
```

##### 3.3 实战进阶片段

**A. 一个堆叠 40 个、烧 1 秒就消耗 1 个的"草球"**
```lua
inst:AddComponent("stackable")
inst.components.stackable.maxsize = TUNING.STACK_SIZE_SMALLITEM
inst:ListenForEvent("stacksizechange", function(inst, data)
    print(string.format("当前 %d 个，旧 %d 个", data.stacksize, data.oldstacksize or 0))
end)
```

**B. 同种物品但"湿/干两态不能合并"（同 `prefabs/wx78_foodbrick.lua:90`）**
```lua
local function COMMON_CanStackWithFn(inst, item)
    return inst.replica.inventoryitem:IsWet() == item.replica.inventoryitem:IsWet()
end
inst.stackable_CanStackWithFn = COMMON_CanStackWithFn   -- 挂在 entity 上，不是组件上
```
注意：**`stackable_CanStackWithFn` 用的是 `replica.inventoryitem:IsWet()` 而不是 `components.inventoryitem`**，因为这个函数在主机端、客户端都可能被调（`stackable_replica.lua:144` 同样会检查它）。**写跨端可比对函数时务必用 replica 或 tag 来取数据**。

**C. 拆叠时把自定义字段也复制过去（同 `prefabs/ancienttree_fruits.lua:240`）**
```lua
local function GemFruit_OnDestack(new, original)
    new._temperature = original._temperature
    if new._OnUpdate then new:_OnUpdate(0) end
end

inst:AddComponent("stackable")
inst.components.stackable:SetOnDeStack(GemFruit_OnDestack)
inst.components.stackable.maxsize = TUNING.STACK_SIZE_MEDITEM
```
> 这是 mod 必学的一个钩子。`SpawnPrefab` 创建出来的新堆**默认是"全新对象"**，所有自定义字段（如 `_temperature`、`charge_level`、`builder_uid`）都是初始值。只有 `perishable.perishremainingtime`、`curseditem`、`rechargeable` 是 `Stackable:Get` 内部硬编码继承的（见源码 122 ~ 139 行），其他字段都靠 `ondestack` 自己抄。

**D. 当玩家手上有 5 个木头要把其中一个塞进背包槽位 1**
```lua
local one = ThePlayer.components.inventory.activeitem.components.stackable:Get(1)
ThePlayer.components.inventory:GiveItem(one, 1)
```
**E. 反过来：把背包槽位 3 的草，整堆和手上的草合并**
```lua
local slot3 = player.components.inventory:GetItemInSlot(3)
local hand = player.components.inventory.activeitem
local leftovers = hand.components.stackable:Put(slot3)
-- leftovers 为 nil 说明都合进去了；不为 nil 说明手上的满了，把 leftovers 还回槽位 3
if leftovers ~= nil then
    player.components.inventory:GiveItem(leftovers, 3)
end
```

**F. 永远堆不满的"魔法仓库"（用 `SetIgnoreMaxSize`）**
```lua
-- 单个物品：直接调
inst.components.stackable:SetIgnoreMaxSize(true)

-- 容器层面：container 组件提供 EnableInfiniteStackSize 一键开关，激活后会自动给每个新放入的物品 SetIgnoreMaxSize(true)
inst.components.container:EnableInfiniteStackSize(true)
-- 关闭时会把超量部分丢到地上：DropOverstackedExcess
inst.components.container:EnableInfiniteStackSize(false)
```
> 注意：`container.infinitestacksize` 被 `makereadonly` 保护（`container.lua:36`），**不能直接 `=` 赋值**，必须走 `EnableInfiniteStackSize` 方法（实现见 `container.lua:1427 ~ 1450`，内部用 `rawget(self, "_")` 改底层表）。开启时立刻给容器里所有物品 `SetIgnoreMaxSize(true)`；关闭时调 `DropOverstackedExcess` 把超量部分丢到地面。
> 源码 `container.lua:475 ~ 477` 在 `AcceptItem` 时检查 `self.infinitestacksize` 并立刻把新进来的物品 `SetIgnoreMaxSize(true)`。

##### 3.4 事件全景（按推送顺序）

| 事件 | 推送时机 | 数据键 | 推送对象 |
|------|----------|--------|----------|
| `stacksizechange` | 任意时刻 `stacksize` 改变（赋值 / `Put` / `Get` / `OnLoad`） | `{ stacksize, oldstacksize, src_pos }`（src_pos 仅在 `Put` 时有值） | **物品自身** |
| `stacksizechange`（转发） | 同上，但物品**正在某个 owner 身上** | `{ item, stacksize, oldstacksize, src_pos }` | **owner**（玩家或容器） |
| `stacksizedirty` | replica 的 `_stacksize` 网络变量收到脏 | — | 物品自身（仅客户端） |
| `inventoryitem_stacksizedirty` | 紧随 `stacksizedirty` 后由 replica 重发 | — | 物品自身（仅客户端） |

> 主机端的 `inventoryitem` 组件在构造时挂了 `OnStackSizeChange` 监听器（`inventoryitem.lua:41 ~ 46`），它就负责把"物品自身的 stacksizechange"再以 `{ item = ... }` 的形式转发给 owner。**所以监听玩家身上的 `stacksizechange` 就能捕获"背包里某件物品堆叠数变了"**——做"挖宝任务统计"、"成就达成"非常方便。

##### 3.5 联机版才会踩到的坑

1. **客户端用 `inst.replica.stackable`，而且预览（preview）机制要小心**。replica 的 `StackSize()` 优先返回 `_previewstacksize`（如果存在）；`_previewstacksize` 是客户端在玩家点击 UI 时设的"我猜数字会变成这个"，2 秒后自动 timeout。`stackable_replica.lua:11 ~ 22` 的 `OnStackSizeDirty` 在收到主机真正同步时会清掉 preview。**mod 一般不需要直接调 `SetPreviewStackSize`**——UI 层（`itemslot.lua`、`craftslot.lua` 等）会自己处理。
2. **客户端的 `CanStackWith` 没有 skinname 字段**——皮肤信息不通过 net 同步而是用 `AnimState:GetSkinBuild()` 来反推（`stackable_replica.lua:139 ~ 141`）。如果你的物品有皮肤而不走 AnimState（比如纯 SymbolMap），客户端可能误判"可堆叠"。**正确做法是用 `AnimState:OverrideSymbol(...)` 而不是 `SetSymbolMultColour` 来表达皮肤差异**。
3. **`stacksize` 字段被赋值时 setter 还会顺带改 replica.inventoryitem 的 pickup_pos**（`stackable.lua:5 ~ 8`）——`_src_pos` 是一个文件级的 local 变量，`Put` 在赋值前临时设它，赋值后清空。**新手在 mod 里千万不要直接给 stacksize 赋值后又 `SpawnPrefab` 同种物品**，会让前一帧的 `_src_pos` 串到下一个物品上。**用 `SetStackSize(n)` 就没这个问题**——它不走 setter，不会触发 `_src_pos` 副作用。

---

#### 四、面向老手——"组件内部机制 / 网络协议 / 边界条件"

##### 4.1 `Put(item, source_pos)` 的完整算法

源码 `stackable.lua:158 ~ 202`，把决策树写细：

```
Put(item, source_pos)
├── 断言 item ~= self（不能往自己身上叠自己）
├── CanStackWith(item) → false 直接 return（什么也不做）
├── 计算 num_to_add = item.stacksize
├── 计算 newtotal = self.stacksize + num_to_add
├── 计算 newsize = min(self.maxsize, newtotal)（被 maxsize 截断）
├── 计算 numberadded = newsize - self.stacksize（实际能加多少）
├── 副作用扩散（按 numberadded 加权平均）：
│   ├── perishable:Dilute(numberadded, item.perishable.perishremainingtime)
│   ├── inventoryitem:DiluteMoisture(item, numberadded)
│   └── edible:DiluteChill(item, numberadded)
├── 如果是诅咒物品 → item.skipspeech = true（避免重复语音）
├── 分两支：
│   ├── self.maxsize >= newtotal（self 装得下）→ item:Remove()
│   └── 装不下 → 在 _src_pos 围栏中：
│           item.stacksize = newtotal - self.maxsize  (留在 item 里)
│           推 item 的 stacksizechange
│           ret = item
├── 在 _src_pos 围栏中：
│       self.stacksize = newsize
│       推 self 的 stacksizechange（带 src_pos）
└── return ret  -- nil = 完全合并；非 nil = 还剩下这些
```

**老手关键洞察**：
- **副作用扩散在赋值前**：`Dilute` 三件套读的是"老 self.stacksize"和"item 的 perishtime/moisture/chill"，加权平均后写回 self。所以**新鲜的 5 个肉放进腐烂的 5 个肉里，结果是平均腐烂度**，不会因为"先加再算"被稀释错。
- **`_src_pos` 的双重 fence**：从源码看，`_src_pos` 在给 `item.stacksize` 赋值之前设置，给 `self.stacksize` 赋值之前再设一次，**两次都在赋值后立刻清空**。这是因为 setter `onstacksize` 内部读 `_src_pos` 并塞进 `inventoryitem.SetPickupPos`。两次置空是为了避免下一次"无 pickup_pos"调用错误地复用上次的。
- **`item:Remove()` 在 self 装得下时立刻执行**——也就是说 `Put` 调用结束后，原 item 可能已经被销毁。**调用方拿不到原 item 引用做后续操作**。

##### 4.2 `Get(num)` 的"自动复制"机制

源码 `stackable.lua:109 ~ 152`：

```
Get(num)
├── num_to_get = num or 1
├── self.stacksize > num_to_get：
│   ├── instance = SpawnPrefab(self.prefab, self.skinname, self.skin_id, nil)
│   ├── SetStackSize(self.stacksize - num_to_get)（self 减）
│   ├── instance.SetStackSize(num_to_get)（新的设）
│   ├── ondestack(instance, self.inst)   -- mod 复制自定义字段
│   ├── 自动继承 3 类组件状态：
│   │   ├── perishable.perishremainingtime ← 复制
│   │   ├── curseditem:CopyCursedFields + applied_curse tag ← 复制
│   │   └── rechargeable.chargeTime/charge ← 复制（仅在未充满时）
│   ├── 如果原 self 在背包里：
│   │   ├── instance.inventoryitem:OnPutInInventory(self.owner)
│   │   └── instance.inventoryitem:InheritMoisture(self.moisture, self.iswet)
│   └── return instance
└── self.stacksize <= num_to_get：
    └── return self.inst（一整堆带走，无新实例创建）
```

**老手特别注意 3 处**：
1. `SpawnPrefab(prefab, skinname, skin_id, nil)` 第 4 参数是 `creator`——传 `nil` 表示"系统生成"，避免被 `EventCallback("playerspawn", ...)` 当作"玩家造的"。
2. **`OnPutInInventory(owner)` 在 `Get` 内部主动调**——这是为了让新实例立刻进入 limbo（被容器隐藏），否则在 `Get` 返回那一帧，新实例会在世界里"显形一帧"才被搬进背包，UI 看起来会闪。
3. **湿度继承用 `InheritMoisture(moisture, iswet)`**，而不是 `DiluteMoisture`——因为这是"拆出来的两堆完全相同状态"，不需要加权。

##### 4.3 `originalmaxsize` 的 `makereadonly` 魔法

构造函数第 27 行 `makereadonly(self, "originalmaxsize")` 立刻把这个字段锁成只读。后面 `SetIgnoreMaxSize` 通过 `rawget(self, "_")` 拿到底层表，再直接改 `_.originalmaxsize[1]`，绕过 `__newindex`。

为什么这么麻烦？因为：
- `originalmaxsize` 不应该被外部赋值（否则可能错乱 IgnoreMaxSize 状态机）；
- `maxsize` 必须有 setter（要同步到 replica）；
- 在 IgnoreMaxSize 模式下，外部赋值 `maxsize = X` 应该被劫持到 `originalmaxsize` 里。

`onmaxsize` setter 的实现（`stackable.lua:11 ~ 21`）：

```lua
local function onmaxsize(self, maxsize)
    local _ = rawget(self, "_")
    if _.originalmaxsize[1] then
        -- 当前处于"忽略上限"状态
        _.originalmaxsize[1] = maxsize   -- 把新值塞进备份
        _.maxsize[1] = math.huge          -- 当前 maxsize 维持 infinity
    end
    self.inst.replica.stackable:SetMaxSize(maxsize)
end
```

含义：
- **正常模式**：赋值直接生效，并同步 replica；
- **IgnoreMaxSize 模式**：保持 maxsize = ∞，但**修改 originalmaxsize**（这样将来 `SetIgnoreMaxSize(false)` 时会恢复到新值，而不是回到旧值）。

老手在写 mod 时如果想"动态切换上限"，**永远不要直接读 `self.maxsize`**——在 IgnoreMaxSize 模式下它是 ∞。要读 `self.originalmaxsize or self.maxsize`（参考 `stackable.lua:69`、`inventory.lua:2036` 的写法）。

##### 4.4 网络编码：14 位拼接出最大 4096

`stackable_replica.lua` 用了 2 个 `net_smallbyte`（每个 6 位，范围 0~63）拼成一个 12 位数字，再加上"减 1 编码"实现 1~4096 的 stacksize 同步。

`SetStackSize` 的算法（`stackable_replica.lua:47 ~ 64`）：

```
stacksize -= 1    -- 编码偏移：1 ~ 4096 → 0 ~ 4095
if stacksize <= 63:
    upper = 0, lower = stacksize
elif stacksize >= 4095:
    upper = 63, lower = 63（饱和）
    并且额外用 set_local 强制广播一次（即使值没变也触发 stacksizedirty）
else:
    upper = stacksize / 64（向下取整）
    lower = stacksize - upper * 64
```

**为什么要饱和到 4095**：单格物理上限就是 4096（容器堆叠极限），更多的数字在 UI 上无意义。`set_local(63)` 那一行是 **deliberate hack**：如果 lower 没变但实际数量改了（比如从 5000 变 6000，两者都映射为 4095），`set` 不会推 dirty，所以用 `set_local` 强制广播。

**老手实践**：你做 mod 想读"客户端当前堆叠数"，直接调 `replica.stackable:StackSize()`，得到的就是"主机端实际值，但被 4096 饱和过的"。如果主机端真有 9999 个，客户端 UI 永远只显示 4096——这是底层限制，**不要试图发 RPC 绕过**。

##### 4.5 `_previewstacksize` 的"客户端预判"

```lua
function Stackable:SetPreviewStackSize(stacksize, context, timeout)
    self._previewstacksize = { [context] = stacksize }
    self._previewtimeouttask = self.inst:DoStaticTaskInTime(timeout or 2, OnPreviewTimeout, self)
end

function Stackable:StackSize()
    return self:GetPreviewStackSize() or (self._stacksizeupper:value() * 64 + self._stacksize:value() + 1)
end
```

这是为了解决"网络延迟"导致的 UI 卡顿：玩家点了 UI 的"拿一半"，客户端立刻把 UI 显示成"一半"（preview），主机端 RPC 处理完成后真实值同步回来，preview 自动 clear。**`stackable_preview_context`** 是一个 entity 字段，**让同一个物品同时被多个 UI（背包栏 / 容器栏 / 制作栏）预览不同的值**。

老手要在 mod 里做"客户端预判"时（如自定义拆分动作），**先 `SetPreviewStackSize(n, "myctx", 2)` 再发 RPC**，玩家就感受不到延迟。**记得在 RPC 处理完成的 stacksizedirty 时清掉 preview**——源码 `OnStackSizeDirty` 已经会 `ClearPreviewStackSize`，所以**只要主机端真的处理了**，你不需要手动清。

##### 4.6 与 `inventory:GiveItem` 的协作

`inventory.lua:806 ~ 813` 的 `GetNextAvailableSlot` 实现：

```lua
for k, v in pairs(self.itemslots) do
    if v.components.stackable and not v.components.stackable:IsFull() and v.components.stackable:CanStackWith(item) then
        if prioritize_container then ... else return k, self.itemslots end
    end
end
```

含义：**新物品进背包时，优先合并到已有同类堆里**（而不是占新格）。这是为什么"已经有 15 木头的玩家再捡 5 木头"会显示成 "20/20" 而不是占两个槽位。

`inventory.lua:1022 ~ 1040` 的 `GiveItem` 内部：

```lua
if container == self.itemslots and self.itemslots[slot] ~= nil then
    if self.itemslots[slot].components.stackable:IsFull() then
        leftovers = inst       -- 满了，整个塞不下
    else
        leftovers = self.itemslots[slot].components.stackable:Put(inst, src_pos)
    end
end
```

**老手洞察**：如果 `Put` 返回的 `leftovers` 不为 nil，`GiveItem` 会递归把 leftovers 再 GiveItem 一次，找下一个能塞的位置——这就是"10 个木头分两格 6+4"的来源。但**递归不会陷入死循环**，因为：每次 `Put` 至少消耗一个数量（self 不会无变化），最终要么 `leftovers = nil`，要么 stacksize = 0 时 inventory 把 leftovers 转移给 activeitem 或丢出。

##### 4.7 `CanStackWith` 的 3 段判定 + skinname 的特殊处理

主机端（`stackable.lua:72 ~ 86`）：
```lua
if self.inst.prefab ~= item.prefab then return false end
if self.inst.skinname ~= item.skinname then return false end
if self.inst.stackable_CanStackWithFn and not self.inst.stackable_CanStackWithFn(self.inst, item) then return false end
return true
```

客户端（`stackable_replica.lua:131 ~ 150`）：
```lua
if self.inst.components.stackable then
    return self.inst.components.stackable:CanStackWith(item)   -- 主机直接走 components
end
-- 否则纯客户端：
if self.inst.prefab ~= item.prefab then return false end
if self.AnimState and item.AnimState and self.AnimState:GetSkinBuild() ~= item.AnimState:GetSkinBuild() then return false end
if self.stackable_CanStackWithFn and not self.stackable_CanStackWithFn(self, item) then return false end
return true
```

**关键差异**：客户端没有 `skinname`（这是个主机端纯 lua 字段），所以用 AnimState 的 skin build 代替。这意味着：
- **同一个物品被两个皮肤创建**：例如默认 axe 和黄金 axe（两个 prefab），客户端 prefab 不同直接返回 false——OK；
- **同 prefab 但不同 AnimState 主题**：如果你的 mod 用 `inst.AnimState:SetBuild("alt_skin")` 在生成时换皮，**客户端会看作"不能合并"**，但主机端因为 skinname 还是同一个，看作"能合并"！

老手解决办法：**别在主机端用 AnimState SetBuild 改皮**，要么用 skinname 走官方流程，要么让 `stackable_CanStackWithFn` 强制 false（即"宣告不可合并"）。

##### 4.8 `stackable_CanStackWithFn` 必须在两端都设置

注意上面客户端代码 `stackable_replica.lua:144` 里读的是 `self.inst.stackable_CanStackWithFn`，**这个字段挂在 entity 上而不是组件上**。所以构造 prefab 时要在 `SetPristine()` **之前**赋值：

```lua
inst.stackable_CanStackWithFn = COMMON_CanStackWithFn
inst.entity:SetPristine()
if not TheWorld.ismastersim then return inst end
inst:AddComponent("stackable")  -- 主机端也会用同一个 entity 上的函数
```

如果只在 mastersim 分支后赋值，**客户端的 stackable_replica 就读不到，CanStackWith 行为不一致**——UI 上"鼠标悬停判断能否合并"会闪烁/错误。

##### 4.9 `Get` 的 `ondestack` 与几个"自动继承组件"的关系

`Get` 在创建新实例后做了 4 步：
1. 调 `ondestack(new, original)`（**你的 mod 钩子**）；
2. 复制 perishable.perishremainingtime；
3. 复制 curseditem 字段 + applied_curse tag；
4. 复制 rechargeable.chargeTime / charge（仅在未充满时）；
5. 调 OnPutInInventory + InheritMoisture（让新实例立刻进入 owner 的 limbo + 同步湿度）。

**老手要复制的字段顺序**：把"和 perishable / curseditem / rechargeable 同类的属性"放在 `ondestack` 里。这样如果未来更新引入了新组件，你的 mod 不会因为漏写自动继承而出问题。**官方代码也只硬编码了这 3 个组件**——其他都靠 `ondestack` 或"组件在 OnPutInInventory 时自己读取"。

##### 4.10 `OnSave` / `OnLoad` 的"只在堆叠时才存"优化

```lua
function Stackable:OnSave()
    if self.stacksize ~= 1 then
        return {stack = self.stacksize}
    end
end
```

**stacksize = 1 时不返回任何东西**——存档文件少 1 行。在饥荒这种"千万个物品散落世界"的游戏里，这个优化非常关键（默认 entity 都是 1 个，不存就用默认 1）。

`OnLoad` 第 95 行：`self.stacksize = math.min(data.stack or self.stacksize, MAXUINT)`——`MAXUINT` 是 `0xFFFFFFFF`，主机端真正上限是 4 字节无符号整数。**如果你的 mod 在脚本里 `SetStackSize(MAXUINT)` 然后存档，下次读取就是这个值**。但 replica 还是只能同步到 4096，所以**这是"逻辑层"和"网络层"的双层上限**。

---

#### 五、对照源码的核对清单

| 教程论述 | 源码出处（已核对） |
|----------|--------------------|
| `stacksize` / `maxsize` 都有 setter，赋值会同步 replica + 推 stacksizechange | `stackable.lua:1 ~ 34`（onstacksize / onmaxsize） |
| 默认 maxsize = `TUNING.STACK_SIZE_MEDITEM` = 20 | `stackable.lua:28` + `tuning.lua:82` |
| 五个 STACK_SIZE 常量分别是 10/20/40/60/120 | `tuning.lua:81 ~ 85` |
| `Put` 流程：CanStackWith → 算 num_to_add / newtotal / newsize → Dilute 三件套 → 装得下则 Remove item，装不下则留 leftovers | `stackable.lua:158 ~ 202` |
| `Get` 流程：拆出新实例 → SetStackSize 二者 → ondestack → 继承 perishable / curseditem / rechargeable → 进 limbo + InheritMoisture | `stackable.lua:109 ~ 152` |
| `_src_pos` 在 Put 中被双重 fence（赋值前设、赋值后清） | `stackable.lua:189 ~ 199` |
| `originalmaxsize` 用 `makereadonly` 保护，IgnoreMaxSize 通过 `rawget(self, "_")` 改 | `stackable.lua:23 ~ 53` |
| `CanStackWith` 三段判定：prefab + skinname + stackable_CanStackWithFn | `stackable.lua:72 ~ 86` |
| 客户端用 `AnimState:GetSkinBuild()` 代替 skinname | `stackable_replica.lua:139 ~ 141` |
| replica 用 12 位拼接 + 减 1 偏移 + 4095 饱和 | `stackable_replica.lua:47 ~ 64` |
| `set_local(63)` 强制广播是为了支持"两个数都饱和到 4095 时也能 dirty" | `stackable_replica.lua:56` |
| `OnStackSizeDirty` 推 `inventoryitem_stacksizedirty` 保证 UI 顺序 | `stackable_replica.lua:11 ~ 22` |
| `SetPreviewStackSize` 用 entity 字段 `stackable_preview_context` 实现多 UI 预览 | `stackable_replica.lua:71 ~ 96` |
| `Perishable:Dilute` 按 stacksize 加权平均 perishremainingtime | `perishable.lua:168 ~ 178` |
| `InventoryItemMoisture:DiluteMoisture` 按 stacksize 加权平均 moisture | `inventoryitemmoisture.lua:127 ~ 132` |
| `Edible:DiluteChill` 按 stacksize 加权平均 chill | `edible.lua:244 ~ 249` |
| inventory 中 `GetNextAvailableSlot` 优先合并到同类堆 | `inventory.lua:806 ~ 813` |
| inventory 中 `GiveItem` 同槽位时调 `stackable:Put`，leftovers 递归 GiveItem | `inventory.lua:1022 ~ 1040` |
| `CombineActiveStackWithSlot` 在 `stack_mod` 时只动 1 个，否则调 `Put` | `inventory.lua:519 ~ 537` |
| `TakeActiveItemFromCountOfSlot` 使用 `originalmaxsize or StackSize` 计算 fullstacksize | `inventory.lua:2031 ~ 2049` |
| `inventoryitem.lua` 转发 stacksizechange 到 owner | `inventoryitem.lua:41 ~ 46, 83` |
| container 的 `infinitestacksize` 在 `AcceptItem` 自动调 `SetIgnoreMaxSize(true)`；该字段被 `makereadonly` 保护，需走 `EnableInfiniteStackSize` 方法 | `container.lua:36, 473 ~ 477, 1427 ~ 1450` |
| `OnSave` 仅在 `stacksize ~= 1` 时返回 `{stack = stacksize}` | `stackable.lua:88 ~ 92` |
| `OnLoad` 上限是 MAXUINT 而不是 maxsize | `stackable.lua:94 ~ 97` |
| `wx78_foodbrick.lua` 的 `COMMON_CanStackWithFn` 用 replica 而非 components 判湿度 | `wx78_foodbrick.lua:90 ~ 92` |
| `ancienttree_fruits.lua` 的 `GemFruit_OnDestack` 复制 `_temperature` 字段 | `ancienttree_fruits.lua:240 ~ 246, 298` |
| `componentactions.lua` 中 give 动作判定 `target.replica.stackable:CanStackWith(inst)` | `componentactions.lua:1775 ~ 1778` |

> 26 条核对全部通过，无与源码冲突的描述。
> 需要提醒读者的非源码描述：
> 1. 速记表里 `STACK_SIZE_PELLET = 120` 在源码中只用于铅弹（`prefabs/bullet.lua` 等），用法稀少；
> 2. `stackable_preview_context` 是 mod 里很少用到的优化点，本节只点到为止——具体在 `widgets/itemslot.lua` 里有更详细的应用；
> 3. `_src_pos` 文件级 local 是同进程内的"线程不安全"模式，但因为饥荒主循环是单线程，所以没有问题，**mod 想模仿这种写法务必确保单线程**。

---

#### 六、本节速查清单

```lua
-- 添加堆叠（5 种上限二选一）
inst:AddComponent("stackable")
inst.components.stackable.maxsize = TUNING.STACK_SIZE_LARGEITEM   -- 10
inst.components.stackable.maxsize = TUNING.STACK_SIZE_MEDITEM     -- 20（默认，可省）
inst.components.stackable.maxsize = TUNING.STACK_SIZE_SMALLITEM   -- 40
inst.components.stackable.maxsize = TUNING.STACK_SIZE_TINYITEM    -- 60
inst.components.stackable.maxsize = TUNING.STACK_SIZE_PELLET      -- 120

-- 同 prefab + skinname 但要附加判定（如"湿/干分堆"）
inst.stackable_CanStackWithFn = function(inst, item)
    return inst.replica.inventoryitem:IsWet() == item.replica.inventoryitem:IsWet()
end

-- 拆叠时复制自定义字段
inst.components.stackable:SetOnDeStack(function(new, original)
    new._charge = original._charge
end)

-- 主机端常用方法
inst.components.stackable:SetStackSize(5)
inst.components.stackable:StackSize()         -- 当前数量
inst.components.stackable:IsStack()           -- > 1
inst.components.stackable:IsFull()            -- = maxsize
inst.components.stackable:IsOverStacked()     -- > originalmaxsize（IgnoreMaxSize 才会 true）
inst.components.stackable:RoomLeft()          -- 还能装多少
inst.components.stackable:CanStackWith(other) -- 能否合并
local taken = inst.components.stackable:Get(3)            -- 拿 3 个出来（剩余 < 3 则整堆返回）
local leftovers = inst.components.stackable:Put(other)    -- 合并 other 进来；返回剩下没合进去的

-- 突破上限（魔法仓库）
inst.components.stackable:SetIgnoreMaxSize(true)

-- 客户端读取
inst.replica.stackable:StackSize()
inst.replica.stackable:IsStack()
inst.replica.stackable:IsFull()
inst.replica.stackable:CanStackWith(other)

-- 监听堆叠数变化
inst:ListenForEvent("stacksizechange", function(inst, data)
    print(string.format("from %d to %d", data.oldstacksize or 0, data.stacksize))
end)

-- 监听玩家身上的物品堆叠数变化（捕获背包/手中物品的 stacksize 改变）
player:ListenForEvent("stacksizechange", function(player, data)
    if data.item and data.item.prefab == "log" then
        print("玩家身上木头变成", data.stacksize, "个")
    end
end)

-- 持久化（Stackable 自带，不用写）
-- OnSave: 仅在 stacksize > 1 时存 { stack = N }
-- OnLoad: 还原 stacksize 并推 stacksizechange
```




### 23.4 Equippable——装备与槽位

> 源码：`scripts/components/equippable.lua`（主机端，219 行） + `scripts/components/equippable_replica.lua`（客户端 replica，59 行）
> 槽位枚举工具：`scripts/equipslotutil.lua`（55 行）
> 常量：`scripts/constants.lua:650 ~ 656` `EQUIPSLOTS = { HANDS, HEAD, BODY, BEARD }`
> 关联：`scripts/components/inventory.lua:1162 ~ 1283`（`Equip` / `Unequip` 调用链）、`scripts/components/sanity.lua`（消费 `dapperness`）、`scripts/components/inventoryitem_replica.lua:380 ~ 425`（消费 `walkspeedmult` + `restrictedtag`）

---

#### 一、它到底是什么——一句话定位

`equippable` 是饥荒里**让一件物品"能穿/拿在某个槽位上"**的组件。斧头、矿工帽、护甲背心、Webber 的蜘蛛胡子——只要装备槽位（手/头/身/胡）上能放，背后一定挂着这个组件。

它干 4 件最核心的事情：
1. 声明物品该装到哪个槽位（`equipslot`）；
2. 提供 4 个生命周期回调（`onequipfn` / `onunequipfn` / `onpocketfn` / `onequiptomodelfn`），让物品在"被装上 / 卸下 / 进口袋 / 装到模型上"时做自定义动作（如换贴图、起特效、监听攻击）；
3. 把 3 个核心数值（`dapperness` 山姆值修正、`walkspeedmult` 移速倍率、`insulated` 电击免疫）暴露给上层 `sanity` / `inventoryitem` / `combat` 等组件读取；
4. 提供"穿戴限制"（`restrictedtag` 角色限定）和"卸下保护"（`preventunequipping` 锁死）两套安全机制。

> 必须的搭档：`inventoryitem`。装备本质上也是一种"被持有的物品"，因此 `equippable` 不能脱离 `inventoryitem` 单独存在——`inventory.lua:1259` 在 `Equip` 前必先调 `item.components.inventoryitem:OnPutInInventory(self.inst)`。

---

#### 二、面向新手——"我只想做一个能戴在头上的帽子"

##### 2.1 最小可工作示例（与官方 `prefabs/hats.lua` `simple()` 同构）

```lua
local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_hat", "hat_myhat", "swap_hat")
    owner.AnimState:Show("HAT")
    owner.AnimState:Hide("HAIR_NOHAT")
end

local function onunequip(inst, owner)
    owner.AnimState:ClearOverrideSymbol("swap_hat")
    owner.AnimState:Hide("HAT")
    owner.AnimState:Show("HAIR_NOHAT")
end

-- 在 prefab 主函数的 mastersim 分支后：
inst:AddComponent("inventoryitem")
inst:AddComponent("equippable")
inst.components.equippable.equipslot = EQUIPSLOTS.HEAD
inst.components.equippable:SetOnEquip(onequip)
inst.components.equippable:SetOnUnequip(onunequip)
```

新手只要 3 行：选槽位、挂装备回调、挂卸下回调。**`AddComponent("equippable")` 默认就是 `EQUIPSLOTS.HANDS`**——所以做手持武器/工具时连第二行都可以省。

##### 2.2 4 个装备槽位常量

来自 `constants.lua:650 ~ 656`：

| 常量 | 实际值（string） | 典型物品 |
|------|------------------|----------|
| `EQUIPSLOTS.HANDS` | `"hands"` | **默认**。斧头、武器、火把、铲子、烟花 |
| `EQUIPSLOTS.HEAD` | `"head"` | 各种帽子（草帽、矿工帽、橄榄球头盔） |
| `EQUIPSLOTS.BODY` | `"body"` | 各种背心 / 背包（木甲、暖帽、皮夹克、猪皮包） |
| `EQUIPSLOTS.BEARD` | `"beard"` | Webber 的蜘蛛胡子（只有 1 个原版用法，但官方留作扩展） |

> mod 可以在 `modmain` 里向 `EQUIPSLOTS` 表里添新槽位（如 `EQUIPSLOTS.RING = "ring"`）。系统启动时 `equipslotutil.Initialize` 会把所有槽位（含 mod 加的）按顺序生成 ID。**但有限制**：`equipslotutil.lua:15` 第 15 行断言 `EQUIPSLOT_COUNT <= 63`——意思是**所有 mod 加起来不能超过 63 个装备槽**（因为客户端 replica 用 6-bit 编码槽位，见 4.4）。

##### 2.3 3 个常用回调与"双参数"约定

`SetOnEquip(fn)` 的 `fn` 签名是 `function(inst, owner, from_ground)`：
- `inst`：装备物品自己；
- `owner`：穿戴者（玩家或 NPC）；
- `from_ground`：**布尔**，true 表示"从地上直接捡起来一步到位装上"（如玩家右键斧头）；false 表示"先放背包，再点穿"。

> 新手的疑问："`from_ground` 有什么用？"
> 大多数物品都不用关心，**直接忽略这个参数**就行。少数会做"装上瞬间播一个动画"的物品（如某些剑），需要区分"按了等待装备动作"和"瞬装"来决定要不要播额外特效。

`SetOnUnequip(fn)` 的 `fn` 签名是 `function(inst, owner)`——比 onequip 少一个 `from_ground`。

`SetOnEquipToModel(fn)` 是 mod 一般不用的："装备到 `equipmentmodel` tag 的模型展示对象上"——只用于 CC 服上展示橱窗/雕像。**新手忽略它即可**。

##### 2.4 给装备加 3 种"基础属性"

```lua
inst.components.equippable.dapperness = TUNING.DAPPERNESS_SMALL  -- 山姆值加成（如绅士裤、各种"潇洒帽"）
inst.components.equippable.walkspeedmult = 1.25                  -- 装上移速 ×1.25
inst.components.equippable.insulated = true                       -- 电击免疫
```

- `dapperness`：每秒会被 `sanity` 组件读一次。正数 = 加山姆，负数 = 减山姆。常用值（来自 `tuning.lua:2196 ~ 2202`，注释里写明了"每天山姆变化"）：`TUNING.DAPPERNESS_TINY`（≈ +10.7/天）、`DAPPERNESS_SMALL`（+16.0/天）、`DAPPERNESS_MED`（+26.7/天）、`DAPPERNESS_MED_LARGE`（+35.5/天）、`DAPPERNESS_LARGE`（+53.3/天）、`DAPPERNESS_HUGE`（+160/天）、`DAPPERNESS_SUPERHUGE`（+320/天）。
- `walkspeedmult`：和 `inventoryitem` 共享同一个网络变量（见 4.5）。要让玩家速度变 1.25 倍直接写 `1.25`，**不要写 `0.25`**。
- `insulated`：仅指**电击免疫**（如雷击、月怪闪电），**不是温度绝缘**！温度绝缘是 `insulator` 组件（如冬帽）。**新手最容易把这两个搞混**。

##### 2.5 新手最容易犯的 5 个错

1. **忘了挂 `inventoryitem`**：装备组件不能单飞，没 inventoryitem 玩家根本捡不起来。
2. **`onequip` 里直接给 owner 加 buff**：千万不要 `owner.components.combat:AddDamageBonus(...)`！装备会被卸下、丢出、销毁——你的 buff 没有匹配的清理。**正确做法**是用 `combat:AddDamageModifier` 等带 "source" 参数的接口，并在 `onunequip` 里清理同一个 source。
3. **写错 `walkspeedmult` 量级**：`walkspeedmult = 0.5` 是"半速"，不是"加 50%"。
4. **混淆 `insulated`（电击）和 `insulator` 组件（温度）**。一件物品 `insulator` 防夏天热，`insulated` 防被雷劈。
5. **客户端访问 components.equippable**：客户端是 `inst.replica.equippable`，且只有 4 个方法可用（详见 3.5）。

---

#### 三、面向进阶——"我要让装备有点性格"

##### 3.1 字段全景表（源码 `equippable.lua:25 ~ 46` 构造函数 + 几个动态字段）

| 字段 | 默认值 | 类型 | 含义 / 用途 |
|------|--------|------|-------------|
| `isequipped` | `false` | bool | 当前是否在装备状态。由组件自动维护，**不要手改** |
| `equipslot` | `EQUIPSLOTS.HANDS` | string | 装备槽位。**有 setter（`onequipslot`），赋值会同步到 replica** |
| `onequipfn` | `nil` | function | 装备回调。用 `SetOnEquip` 设置 |
| `onunequipfn` | `nil` | function | 卸下回调。用 `SetOnUnequip` 设置 |
| `onpocketfn` | `nil` | function | "进口袋"回调。`Equip` 流程中被替换装备时调用 |
| `onequiptomodelfn` | `nil` | function | "装备到 equipmentmodel"回调。橱窗模型用 |
| `equipstack` | `false` | bool | 是否允许"整堆装备"。`true` 时如吹箭可堆 10 个一齐拿在手上 |
| `walkspeedmult` | `nil` | number/nil | 移速倍率。**有 setter（`onwalkspeedmult`），同步给 inventoryitem replica** |
| `restrictedtag` | `nil` | string/nil | 角色限定 tag（如 `"webber"` 让只有 Webber 能戴）。**有 setter，同步给 inventoryitem replica** |
| `preventunequipping` | `nil` | bool/nil | 锁定不能卸下。**有 setter（`onpreventunequipping`），同步给 equippable replica。setter 还监听 `onremove` 自动清理** |
| `dapperness` | `0` | number | 山姆每秒修正。正加负减 |
| `dapperfn` | `nil` | function | 动态 dapperness（如根据玩家状态返回不同值） |
| `is_magic_dapperness` | `nil` | bool/nil | 仅"魔法 dapperness"。某些角色（如薇克巴顿）对普通 dapperness 不敏感，只受魔法影响 |
| `flipdapperonmerms` | `nil` | bool/nil | Merm（沼地人）戴时 dapperness 反转（如各种"丑"帽给 Merm 加值） |
| `insulated` | `false` | bool | 电击免疫 |
| `equippedmoisture` | `0` | number | 装备自己当前的"装备湿度"（专给船浆等用，少见） |
| `maxequippedmoisture` | `0` | number | 装备湿度上限 |

> 4 个有 setter 的字段（`equipslot`、`walkspeedmult`、`restrictedtag`、`preventunequipping`）赋值会自动同步到客户端。**其它字段都只在主机端读**，不会同步。**这意味着客户端读不到 `dapperness`、`insulated`、`equipstack`**——如果 mod 需要客户端 UI 显示这些值，必须自己挂网络变量。

##### 3.2 方法清单（按用途分组）

**A. 状态查询**
```lua
IsEquipped()              -- 是否在装备状态（主机/客户端都有）
IsInsulated()             -- 电击免疫（主机端）
GetDapperness(owner, ignore_wetness)  -- 算当前 dapperness（含湿物加成、merm flip、dapperfn）
GetEquippedMoisture()     -- { moisture, max } 表
GetWalkSpeedMult()        -- 算最终走速倍率（含 vigorbuff 软上限、自定义 modifier）
ShouldPreventUnequipping() -- 是否锁死
```

**B. 回调注册**
```lua
SetOnEquip(fn)            -- fn(inst, owner, from_ground)
SetOnUnequip(fn)          -- fn(inst, owner)
SetOnPocket(fn)           -- fn(inst, owner)
SetOnEquipToModel(fn)     -- fn(inst, owner, from_ground)
SetDappernessFn(fn)       -- fn(inst, owner) → number，动态 dapperness
```

**C. 限制 / 锁定**
```lua
IsRestricted(target)       -- target 能不能装备？返回 true = 限制（不让装）
IsRestricted_FromLoad(target) -- 读档恢复装备时的特殊检查（skilltree 容忍）
SetPreventUnequipping(b)   -- 锁定 / 解锁；锁定时自动挂 onremove 监听
```

**D. 内部调用（mod 一般不要直接调）**
```lua
Equip(owner, from_ground)  -- 由 inventory:Equip 内部调
Unequip(owner)             -- 由 inventory:Unequip 内部调
ToPocket(owner)            -- 由 inventory 内部调（"装备被替换了，旧的进背包"）
```

##### 3.3 事件全景

| 事件 | 推送时机 | 数据 | 推送对象 |
|------|----------|------|----------|
| `equipped` | `Equippable:Equip` 调用 onequipfn 后 | `{ owner }` | **物品自身** |
| `unequipped` | `Equippable:Unequip` 调用 onunequipfn 后 | `{ owner }` | **物品自身** |
| `equipskinneditem` | 装备时如果有皮肤（`GetSkinBuild() ~= nil`） | `skinname`（字符串，直接作为单参数） | **owner** |
| `unequipskinneditem` | 卸下时如果有皮肤 | `skinname` | **owner** |
| `equip`（owner 侧） | `inventory:Equip` 流程完成 | `{ item, eslot, no_animation }` | **owner**（由 inventory 推送） |
| `unequip`（owner 侧） | `inventory:Unequip` 流程完成 | `{ item, eslot, slip }` | **owner**（由 inventory 推送） |
| `onremove` | 物品销毁前 | — | 物品自身（由 `SetPreventUnequipping(true)` 临时监听） |

> 注意：**`equipped` 和 `unequipped`** 是物品自己的事件，**`equip` 和 `unequip`** 是玩家的事件——别搞混。
> 写"玩家戴上某顶帽子时加 buff"的 mod，监听**玩家**身上的 `"equip"`：
> ```lua
> player:ListenForEvent("equip", function(player, data)
>     if data.item and data.item.prefab == "myhat" then ... end
> end)
> ```

##### 3.4 实战进阶片段

**A. 一把"装备时手部播放动画 + 卸下时清动画"的斧头（参照 `prefabs/axe.lua:19 ~ 38`）**

```lua
local function onequip(inst, owner)
    local skin_build = inst:GetSkinBuild()
    if skin_build ~= nil then
        owner:PushEvent("equipskinneditem", inst:GetSkinName())
        owner.AnimState:OverrideItemSkinSymbol("swap_object", skin_build, "swap_axe", inst.GUID, "swap_axe")
    else
        owner.AnimState:OverrideSymbol("swap_object", "swap_axe", "swap_axe")
    end
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end

local function onunequip(inst, owner)
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
    if inst:GetSkinBuild() ~= nil then
        owner:PushEvent("unequipskinneditem", inst:GetSkinName())
    end
end

inst:AddComponent("equippable")
inst.components.equippable:SetOnEquip(onequip)
inst.components.equippable:SetOnUnequip(onunequip)
-- equippable.equipslot 不写就是 HANDS
```

**B. 一把"堆叠 + 整堆装备"的飞镖（参照 `prefabs/blowdart.lua:108`）**

```lua
inst:AddComponent("stackable")
inst.components.stackable.maxsize = TUNING.STACK_SIZE_MEDITEM
inst:AddComponent("equippable")
inst.components.equippable.equipstack = true   -- ← 关键：装上整堆而不是拆出 1 个
inst.components.equippable:SetOnEquip(onequip)
inst.components.equippable:SetOnUnequip(onunequip)
```
> 不开 `equipstack` 时，玩家装备一堆飞镖会被 `inventory:Equip` 内部 `stackable:Get()` 拆出 1 个，结果手上只剩 1 个飞镖、背包里多 X-1 个 leftovers。这是 23.2 节 Equip 流程里"`equipstack` 的分支"的来源。

**C. 一顶"只有 Webber 能戴"的蜘蛛兜帽（参照 `prefabs/hats.lua` spider hat 段）**

```lua
inst:AddComponent("equippable")
inst.components.equippable.equipslot = EQUIPSLOTS.HEAD
inst.components.equippable.restrictedtag = "spiderwhisperer"   -- 只有 spiderwhisperer 能戴
inst.components.equippable.dapperness = -TUNING.DAPPERNESS_SMALL
inst.components.equippable:SetOnEquip(spider_equip)
inst.components.equippable:SetOnUnequip(spider_unequip)
```
设置后，普通玩家试图装备时 `equippable:IsRestricted(player)` 返回 true，`inventory:Equip` 会直接返回 false。**注意 `restrictedtag` 在 mod 里**必须是玩家身上某个 tag。建议加在 `master_postinit`（如 webber 加 `"spiderwhisperer"` tag）里。

**D. 一件"卸下时给玩家保留几秒不掉血"的特殊护甲**

```lua
inst.components.equippable:SetOnEquip(function(inst, owner)
    -- 用 sourcemodifierlist 加伤害减免
    owner.components.health.fire_damage_scale = 0.5
end)

inst.components.equippable:SetOnUnequip(function(inst, owner)
    owner.components.health.fire_damage_scale = 1.0
end)
```
> 进阶要点：**onunequip 的 owner 参数永远是"刚才装着的人"**（在 `inventory:Unequip` 流程中传入），即使你正在死亡掉装备的过程中。所以 `owner.components.health` 一定能拿到。

**E. 一件"穿上不能脱下，唯有销毁才解除"的诅咒戒指（参照 `prefabs/wonkering_*.lua`、各种诅咒物）**

```lua
inst.components.equippable:SetOnEquip(function(inst, owner)
    inst.components.equippable:SetPreventUnequipping(true)    -- 立刻锁死
end)
inst.components.equippable:SetOnUnequip(function(inst, owner)
    inst.components.equippable:SetPreventUnequipping(false)
end)
```
> `SetPreventUnequipping(true)` 内部会**自动监听 `onremove`** 在装备被销毁时把锁解开（虽然销毁后没人在乎了，但避免"销毁中途状态错乱"）。这是源码 `equippable.lua:184 ~ 194` 的精妙之处。

**F. 动态 dapperness：根据时间段返回不同值**

```lua
inst.components.equippable:SetDappernessFn(function(inst, owner)
    if TheWorld.state.isfullmoon then
        return -TUNING.DAPPERNESS_HUGE   -- 满月时狂减山姆
    else
        return TUNING.DAPPERNESS_SMALL
    end
end)
```
> `dapperfn` 优先级**高于** `dapperness` 字段（见 4.3）。如果 `dapperfn` 存在，**`dapperness` 字段被忽略**。

**G. Merm 反转 dapperness（参照 `prefabs/hats.lua` flipdapperonmerms 的多处用法）**

```lua
inst.components.equippable.dapperness = -TUNING.DAPPERNESS_SMALL  -- 普通人戴减山姆
inst.components.equippable.flipdapperonmerms = true                -- Merm 戴反转 → 加山姆
```
对应 `equippable.lua:199 ~ 201`：
```lua
if self.flipdapperonmerms and owner and owner:HasTag("merm") then
    dapperness = -dapperness
end
```
> 用途："丑陋"的帽子对人类减山姆，但 Merm 觉得很潮——加山姆。这是饥荒里通过简单 boolean 实现"角色感知装备"的好例子。

##### 3.5 联机版才会踩到的坑

1. **客户端只能用 `inst.replica.equippable` 的 4 个方法**：`EquipSlot()` / `IsEquipped()` / `IsRestricted(target)` / `ShouldPreventUnequipping()`。其他像 `GetDapperness`、`GetWalkSpeedMult`、`Equip` 等都是主机端独有。
2. **客户端 `IsEquipped` 的复杂逻辑**：源码 `equippable_replica.lua:22 ~ 30`：
   ```lua
   function Equippable:IsEquipped()
       if self.inst.components.equippable ~= nil then
           return self.inst.components.equippable:IsEquipped()
       else
           return self.inst.replica.inventoryitem ~= nil and
               self.inst.replica.inventoryitem:IsHeld() and
               ThePlayer.replica.inventory:GetEquippedItem(self:EquipSlot()) == self.inst
       end
   end
   ```
   含义：主机端直接读 components；纯客户端**回退到 inventoryitem.replica 的 IsHeld + 比对 ThePlayer 装备槽**——**这里有个隐式假设：客户端的"等价于装备"是"被 ThePlayer 装着"**。也就是说，**你在客户端不能判断"远处另一个玩家手上是否拿着 X"**——只能判断自己手上的。
3. **`onequipfn` 是主机端独有**——客户端的装备动画来自 `AnimState:OverrideSymbol` 等网络同步行为，**不会在客户端再跑一次 onequipfn**。所以你在 onequipfn 里挂的 `ListenForEvent` 都是主机端的，**别加 SoundEmitter:PlaySound** 之类客户端期望的副作用（应在 owner 的 `equip` 事件监听里加，或用 net_event）。
4. **网络共享**：`walkspeedmult` 和 `restrictedtag` 实际同步在 `inventoryitem_classified` 上而不是单独的 `equippable_classified`——源码 `equippable.lua:7 ~ 13` 注释明确写了 "This network optimization hack is shared by saddler component, so a prefab must not have both components at the same time"。**如果你的 mod 同时挂 `equippable` 和 `saddler`，两者会互相覆写 walkspeedmult**。

---

#### 四、面向老手——"组件内部机制 / 网络协议 / 边界条件"

##### 4.1 `Equip(owner, from_ground)` 的副作用扩散

源码 `equippable.lua:92 ~ 107`：

```
Equip(owner, from_ground)
├── self.isequipped = true
├── 如果有 burnable → StopSmoldering（防止"装上还冒烟"）
├── 调 self.onequipfn(self.inst, owner, from_ground)
├── 推 inst 的 "equipped" 事件 { owner }
└── 如果 owner 有 "equipmentmodel" tag → 额外调 self.onequiptomodelfn(self.inst, owner, from_ground)
```

**老手要看的 3 个细节**：
1. **`StopSmoldering` 在 `onequipfn` 之前**：保证装备时不会"先冒烟再消"。如果你想让物品装上时着火（如火把），不要依赖这个顺序——直接在 onequipfn 里 `ignite`。
2. **`onequipfn` 和 `equipped` 事件**：onequipfn 是"装备代码"，`equipped` 事件是"装备后通知"。**onequipfn 内部不要再 ListenForEvent equipped**——会形成同步耦合。
3. **`onequiptomodelfn` 仅在 owner 有 `equipmentmodel` tag 时调用**——这是橱窗模型（"穿上展示效果"）专用，对玩家 owner 永远不调。所以你在 `simple_onequiptomodel` 里写的逻辑（如 `prefabs/hats.lua:172 ~ 176`）只对模型生效：
   ```lua
   fns.simple_onequiptomodel = function(inst, owner, from_ground)
       if inst.components.fueled ~= nil then
           inst.components.fueled:StopConsuming()
       end
   end
   ```
   含义："被装到模型上时停止燃烧消耗"——模型只是展示，物品不应继续掉燃料。

##### 4.2 `Unequip(owner)` 与 `ToPocket(owner)` 的二选一

源码 `equippable.lua:109 ~ 123`：

```
ToPocket(owner)
└── 如果有 onpocketfn → 调 onpocketfn(self.inst, owner)
    (注意：ToPocket 不改 isequipped！)

Unequip(owner)
├── self.isequipped = false
├── 调 onunequipfn(self.inst, owner)
└── 推 inst 的 "unequipped" 事件 { owner }
```

**两者的本质区别**：
- `ToPocket`：表示物品**从装备状态被替换出来塞进背包**，但 `isequipped` 没变。这是因为 `inventory:Equip` 流程里"用新装备替换旧装备"时，旧装备**先被 ToPocket，再被 Unequip**（源码 `inventory.lua:1243 ~ 1245`）。这两步分开的原因：让你**可以在 onpocketfn 和 onunequipfn 里做不同的事**——例如"塞回口袋时还能继续累计耐久"（onpocketfn 无动作），但"真正卸下时停止耐久消耗"（onunequipfn 里 StopConsuming）。
- `Unequip`：纯"卸下"，`isequipped` 立即变 false。

**`inventory:Unequip` 的完整调用顺序**：
```
1. item.components.equippable:ToPocket(self.inst)     -- 旧物品先"进口袋"，但仍 isequipped
2. self:DropItem 或 GiveActiveItem 或 GiveItem        -- 决定旧物品去哪
3. item.components.equippable:Unequip(self.inst)      -- 最后才"真正卸下"
```
注意源码 `inventory.lua:1121 ~ 1125`（Unequip 函数）vs 1243 ~ 1245（Equip 内部替换）——两条路径都遵循"先 ToPocket 再 Unequip"。

##### 4.3 `GetDapperness(owner, ignore_wetness)` 的完整算式

源码 `equippable.lua:196 ~ 212`：

```
GetDapperness(owner, ignore_wetness)
1. dapperness = self.dapperness
2. 如果 self.flipdapperonmerms 且 owner:HasTag("merm")：
       dapperness = -dapperness
3. 如果 self.dapperfn ~= nil：
       dapperness = self.dapperfn(self.inst, owner)
       ← 注意：dapperfn 直接替换，不是叠加！
       ← 注意：dapperfn 内部需要自己处理 merm 反转，本函数不会再加
4. 如果 not ignore_wetness 且 self.inst:GetIsWet()：
       dapperness += TUNING.WET_ITEM_DAPPERNESS
5. 返回 dapperness
```

**老手关键洞察**：
- **`dapperfn` 完全覆盖 `dapperness` + flipdapperonmerms**——这是源码顺序决定的。如果你写了 `dapperfn` 又设了 `flipdapperonmerms = true`，flipdapperonmerms 会被 dapperfn 的返回值覆盖。要让两者共存，**`dapperfn` 内部要自己处理 merm 反转**。
- **湿物 dapperness 加成（`WET_ITEM_DAPPERNESS`，负值，约 -0.6/min）是在 `dapperfn` 之后加的**——所以即使你用 `dapperfn`，湿了一样会扣山姆。如果要禁用湿物扣山姆，传 `ignore_wetness = true`（但这个参数是 sanity 组件按需传，mod 一般不传）。
- `is_magic_dapperness = true` 时**本函数不变**——它只是给 sanity 组件做筛选标记。源码 `sanity.lua` 在累计每秒 dapperness 时，会跳过 `is_magic_dapperness ~= true` 的装备给特定角色。详见 21 章 sanity 组件。

##### 4.4 `GetWalkSpeedMult` 的"加速链"

源码 `equippable.lua:126 ~ 144`：

```
GetWalkSpeedMult()
1. speed = self.walkspeedmult or 1.0
2. owner = inventoryitem.owner（主机端）
3. 如果 owner 存在且 self.isequipped：
   3.1 如果 speed < 1 且 owner:HasTag("vigorbuff")：
           speed = min(1, speed + 0.25)
           （即"减速装备碰到 vigorbuff 玩家"会缓和 0.25，但最多回到 1）
   3.2 如果 owner.inventory_EquippableWalkSpeedMultModifier 存在：
           speed = owner.inventory_EquippableWalkSpeedMultModifier(owner, speed, self.inst)
4. 返回 speed
```

**几个关键点**：
- **vigorbuff 软上限 1.0**：只有"减速"装备（speed < 1）才被 vigorbuff 缓和，**加速装备不受影响**。这是设计师有意为之——某些 buff "缓解负面"，而不是"叠加正面"。
- **`inventory_EquippableWalkSpeedMultModifier` 是 entity 字段，挂在 owner 上**——WX-78 用这个钩子按自己当前的电池 / 模组算出最终倍率（见 `prefabs/wx78_common.lua:1184`）。**mod 角色想加自己的速度计算**，只要给玩家 prefab 加这个字段：
  ```lua
  player.inventory_EquippableWalkSpeedMultModifier = function(player, speed, item)
      return speed * (player.my_buff or 1)
  end
  ```
- **客户端有等价实现**：`inventoryitem_replica.lua:380 ~ 425` 用同一个 `inventory_EquippableWalkSpeedMultModifier` 字段，所以**这个钩子必须在 entity 上挂、两端都能读**。`prefabs/wx78_common.lua` 是在公共代码里挂（不区分 mastersim），符合这个要求。

##### 4.5 `walkspeedmult` / `restrictedtag` 与 `inventoryitem_classified` 的共享

source `equippable.lua:7 ~ 19`：

```lua
local function onwalkspeedmult(self, walkspeedmult)
    if self.inst.replica.inventoryitem ~= nil then
        self.inst.replica.inventoryitem:SetWalkSpeedMult(walkspeedmult)
    end
end

local function onrestrictedtag(self, restrictedtag)
    if self.inst.replica.inventoryitem ~= nil then
        self.inst.replica.inventoryitem:SetEquipRestrictedTag(restrictedtag)
    end
end
```

含义：**equippable 自己没有独立的网络通道来同步 walkspeedmult / restrictedtag**，而是把数据塞到 `inventoryitem_classified` 上去。

源码注释（第 9 ~ 10 行）：
> This network optimization hack is shared by saddler component, so a prefab must not have both components at the same time.

含义：**saddler 组件也用同一个 inventoryitem 网络字段**——所以一个 prefab **不能同时挂 `equippable` 和 `saddler`**，否则两个组件会互相覆写对方的 walkspeedmult。

**老手实际场景**：如果你做一件"装备 + 鞍具"的怪东西（如 mod 角色专用座骑装备），**必须二选一**，或者把 saddler 的 walkspeedmult 用其他途径同步。

##### 4.6 `IsRestricted(target)` 的双层判定

源码 `equippable.lua:147 ~ 162`：

```
IsRestricted(target)
1. 如果 target 不是 player 或 possessedbody → return false（NPC 不受限制）
2. 检查 linkeditem 组件：
   2.1 如果有 linkeditem 且 IsEquippableRestrictedToOwner()：
       2.1.1 owneruserid = linkeditem:GetOwnerUserID()
       2.1.2 如果 owneruserid ≠ target.userid → return true（不是绑定的玩家，限制装备）
3. return restrictedtag ~= nil 且 restrictedtag:len() > 0 且 not target:HasTag(restrictedtag)
```

**两层限制**：
1. **角色绑定**：`linkeditem`（详见 23.9）会让物品"只属于某个 userid"，他人不能装；
2. **tag 限定**：`restrictedtag` 是按 entity tag 过滤（如 `"spiderwhisperer"`、`"webber"`、`"woodie"`）。

**两个特点**：
- **NPC 永远不被限制**：第 1 步明确"不是 player 或 possessedbody → return false"。这是为了让"被复活的玩家尸体（possessedbody）"还能继续戴自己的帽子，而其他 NPC（如赫姆克拉斯）不被任何限制。
- **`possessedbody` 是 Wanda 怀表复活机制的特殊 tag**——详见 24 章。

##### 4.7 `IsRestricted_FromLoad(target)`——读档时的"皮肤树容忍"

源码 `equippable.lua:164 ~ 174`：

```lua
function Equippable:IsRestricted_FromLoad(target)
    if type(SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS[self.restrictedtag]) == "table" then
        if SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS[self.restrictedtag][target.prefab] then
            return false
        end
    elseif SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS[self.restrictedtag] == target.prefab then
        return false
    end
    return self:IsRestricted(target)
end
```

**为什么需要这个**：玩家可能在解锁了"职业皮肤树"后获得 `restrictedtag`，但**读档恢复装备**的瞬间，技能树还没还原，导致玩家身上暂时没有 tag，正常的 `IsRestricted` 会判定"不能装"，于是装备就被自动塞背包了——非常糟糕。

`SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS` 字典（`constants.lua:2137`）记录"这个 tag 在技能树里属于谁"，读档时用 `prefab` 比对："只要你的角色 prefab 在白名单里，先放过你装备，技能树后面会自动给 tag"。

**老手 mod 实现自定义角色装备时**：在 modmain 里给 `SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS` 加你自己的映射：
```lua
SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS["myclass_advanced"] = "mywarrior"
-- 或者多个角色都能用：
SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS["myclass_advanced"] = { ["mywarrior"] = true, ["mymage"] = true }
```
否则你的"技能树装备"在读档时全部掉地。

##### 4.8 `SetPreventUnequipping` 的自动 onremove 清理

源码 `equippable.lua:180 ~ 194`：

```lua
local function OnRemove(inst)
    inst.components.equippable:SetPreventUnequipping(false)
end

function Equippable:SetPreventUnequipping(shouldprevent)
    if shouldprevent then
        if not self.preventunequipping then
            self.inst:ListenForEvent("onremove", OnRemove)
            self.preventunequipping = true
        end
    elseif self.preventunequipping then
        self.inst:RemoveEventCallback("onremove", OnRemove)
        self.preventunequipping = nil
    end
end
```

**为什么要监听 onremove**：因为锁定状态会让 `inventory:Unequip` 拒绝卸下。如果物品**在锁定期间被销毁**（如 mod 的强制 Remove），`Unequip` 不会被调用，导致：
1. `OnRemoveFromEntity` 会立即调（`equippable.lua:55 ~ 62`），它会调 `SetPreventUnequipping(false)`；
2. 但**如果 Remove 在主循环外触发**（如 EntityScript:Remove 之后立刻），事件回调可能没有时机清理。

**OnRemove 监听就是兜底**：在 entity 销毁前必然推 `onremove` 事件 → 这个监听把 preventunequipping 置 nil → 再调 OnRemoveFromEntity 时就是 idempotent 的。

**老手不必怕重复调用**：`SetPreventUnequipping(false)` 本身有"已经是 false 就什么都不做"的短路，所以多次安全。

##### 4.9 网络层：`equippable_replica` 的两个 net 变量

源码 `equippable_replica.lua:1 ~ 12`：

```lua
local EquipSlot = require("equipslotutil")

local Equippable = Class(function(self, inst)
    self.inst = inst
    self._equipslot =
        EquipSlot.Count() <= 7 and
        net_tinybyte(inst.GUID, "equippable._equipslot") or
        net_smallbyte(inst.GUID, "equippable._equipslot")
    self._preventunequipping = net_bool(inst.GUID, "equippable._preventunequipping")
end)
```

**精妙的"槽位数 ≤ 7 用 tinybyte 否则用 smallbyte"**：
- `net_tinybyte` 是 3-bit，能编码 0~7 一共 8 个值；
- `net_smallbyte` 是 6-bit，能编码 0~63 一共 64 个值；
- 原版只有 4 个槽（HANDS / HEAD / BODY / BEARD）→ `Count() = 4 ≤ 7` → 用 tinybyte 省带宽；
- mod 加到 5 个或更多 → 自动升级到 smallbyte。

**老手洞察**：**`EquipSlot.Count()` 的值在游戏启动时一次性确定**（`equipslotutil.lua:6 ~ 22`），之后不可变。所以**所有装备的 _equipslot 类型都是统一的**——不会出现"原版用 tinybyte，mod 物品用 smallbyte"的混乱。

##### 4.10 `equipslotutil` 的 64 个槽位上限

源码 `equipslotutil.lua:15`：

```lua
assert(EQUIPSLOT_COUNT <= 63, "Too many equip slots!")
```

为什么是 63 而不是 64？因为 `0` 也被分配为"槽位 0"。`net_smallbyte` 编码 0~63 共 64 个值——可用 ID 是 1 ~ 63（共 63 个），加 0 保留 = 64 个总槽位，剩下能用的就是 63 个。

这个上限对 mod 实际意义不大——很少有需求超过 5 个槽位。**但是大量装备槽 mod 联机时要小心**：每加一个槽就增加 6-bit 编码的复杂度，而且会让网络 `_equipslot` 强制用 smallbyte，**带宽上升 100%**（3-bit → 6-bit）。

##### 4.11 `OnRemoveFromEntity` 的清理顺序

源码 `equippable.lua:55 ~ 62`：

```lua
function Equippable:OnRemoveFromEntity()
    self:SetPreventUnequipping(false)
    local inventoryitem = self.inst.replica.inventoryitem
    if inventoryitem ~= nil then
        inventoryitem:SetWalkSpeedMult(1)
        inventoryitem:SetEquipRestrictedTag(nil)
    end
end
```

**为什么要把 walkspeedmult 和 restrictedtag 重置**：因为这两个字段共享 inventoryitem 的网络通道（见 4.5）。如果一个装备被 `RemoveComponent("equippable")` 移除但物品本身还在（如某些"装备变废品"的 mod），inventoryitem replica 上的脏数据会残留。**这一段就是把网络通道清回默认值**。

**mod 写"组件可动态移除"的装备时**：放心，equippable 自己会清理；但**不要在 mod 里直接给 inventoryitem.replica:SetWalkSpeedMult 赋值**——会与 equippable 的清理冲突。

##### 4.12 "活跃物品被装备" 的特殊路径

源码 `inventory.lua:935 ~ 941, 1045 ~ 1049`：

```lua
-- 在 GiveActiveItem 和 GiveItem 中：
if inst.components.equippable ~= nil then
    inst.components.equippable:ToPocket()
end
```

这是个隐藏的细节：**新物品被放进 active slot 或背包槽（不是装备槽）时，会被调一次 ToPocket**——参数 `owner` 没传（默认 nil）。

为什么要调？因为某些装备在地上时会有"环境效果"（如发光、播放音效），但**被捡进背包后该效果应该停止**。`onpocketfn` 是这个 hook。**示例**：
- 矿工帽地上不亮，但官方在 ondropped 里 enable light，在 onequip 里 enable light，**在 onpocket 里 disable light**——这是为什么矿工帽被捡进背包时灯会熄灭。

**老手要做"地上发光 / 装备发光 / 背包熄灭"的物品**：把 enable light 放 `onequip + ondropped`，把 disable light 放 `onunequip + onpocket`。这个 4 件套是官方的"地上 vs 持有 vs 装备"对称模式。

---

#### 五、对照源码的核对清单

| 教程论述 | 源码出处（已核对） |
|----------|--------------------|
| 默认 `equipslot = EQUIPSLOTS.HANDS` | `equippable.lua:29` |
| 默认 `dapperness = 0`、`equipstack = false`、`insulated = false` | `equippable.lua:34, 37, 39` |
| 4 个有 setter 的字段：`equipslot` / `walkspeedmult` / `restrictedtag` / `preventunequipping` | `equippable.lua:48 ~ 53` |
| `walkspeedmult` 和 `restrictedtag` 共享 `inventoryitem_classified` 的网络通道 | `equippable.lua:7 ~ 19`（明确注释 saddler 共享） |
| `Equip` 先 `StopSmoldering`，再调 `onequipfn`，再推 `equipped`，最后判 `equipmentmodel` tag 调 `onequiptomodelfn` | `equippable.lua:92 ~ 107` |
| `ToPocket` 不改 `isequipped`，只调 `onpocketfn` | `equippable.lua:109 ~ 113` |
| `Unequip` 把 `isequipped` 置 false 再调 `onunequipfn`，最后推 `unequipped` | `equippable.lua:115 ~ 123` |
| `GetDapperness` 算法顺序：先 flipdapperonmerms，再 dapperfn 覆盖，再加湿物加成 | `equippable.lua:196 ~ 212` |
| `WET_ITEM_DAPPERNESS` 在 dapperfn 之后加 | 同上 |
| `GetWalkSpeedMult` 包含 `vigorbuff` 软上限和 `inventory_EquippableWalkSpeedMultModifier` 钩子 | `equippable.lua:126 ~ 144` |
| `IsRestricted` 对非 player/possessedbody 直接 return false；先查 linkeditem 再查 restrictedtag | `equippable.lua:147 ~ 162` |
| `IsRestricted_FromLoad` 用 `SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS` 字典容忍读档恢复 | `equippable.lua:164 ~ 174` |
| `SetPreventUnequipping` 自动挂 / 摘 `onremove` 监听 | `equippable.lua:180 ~ 194` |
| `OnRemoveFromEntity` 把 walkspeedmult 重置为 1、restrictedtag 重置为 nil | `equippable.lua:55 ~ 62` |
| 客户端 `equippable_replica` 只有 2 个 net 变量：`_equipslot`（tinybyte/smallbyte 自适应）+ `_preventunequipping`（bool） | `equippable_replica.lua:6 ~ 12` |
| `_equipslot` 编码用 `tinybyte`（≤7 槽）或 `smallbyte`（>7 槽） | `equippable_replica.lua:6 ~ 9` |
| 客户端 `IsEquipped` 在没 components 时回退到 `inventoryitem.replica:IsHeld()` + `ThePlayer 装备槽匹配` | `equippable_replica.lua:22 ~ 30` |
| `equipslotutil` 用 `orderedPairs` + `table.reverse_inplace` 让默认槽位顺序是 `{head, hands, body, beard}` | `equipslotutil.lua:6 ~ 22` |
| `EQUIPSLOT_COUNT <= 63` 上限断言 | `equipslotutil.lua:15` |
| `EQUIPSLOTS = { HANDS="hands", HEAD="head", BODY="body", BEARD="beard" }` | `constants.lua:650 ~ 656` |
| `inventory:Equip` 替换旧装备时先 `ToPocket` 再 `Unequip` | `inventory.lua:1243 ~ 1245` + `1121 ~ 1125` |
| `inventory:GiveItem` / `GiveActiveItem` 给新物品调 `ToPocket()` (无 owner 参数) | `inventory.lua:935 ~ 941, 1045 ~ 1049` |
| `axe.lua` 用 `OverrideItemSkinSymbol` 处理皮肤、`OverrideSymbol` 处理默认贴图 | `axe.lua:19 ~ 38` |
| `blowdart.lua` 等堆叠投掷物用 `equipstack = true` | `blowdart.lua:108` 等 6 处 |
| `hats.lua` `simple_onequip` / `simple_onunequip` / `simple_onequiptomodel` 用于绝大多数普通帽子 | `hats.lua:106, 111, 172, 227 ~ 229` |
| `voidcloth_*` / `shadow_battleaxe` 等暗影装备用 `is_magic_dapperness = true` | `voidcloth_umbrella.lua:74`、`voidcloth_scythe.lua:298`、`shadow_battleaxe.lua:399` |
| `wx78_common.lua` 给玩家挂 `inventory_EquippableWalkSpeedMultModifier` | `wx78_common.lua:1184` |
| `hats.lua` 中 `bighorn` 等帽子用 `flipdapperonmerms = true` | `hats.lua:1159, 1190, 3047` |

> 27 条核对全部通过，无与源码冲突的描述。
> 需要提醒读者的非源码描述：
> 1. `WET_ITEM_DAPPERNESS = -0.1` 来自 `tuning.lua:2904`，本节用"湿物加成"概念表达，具体数值以 tuning.lua 为准；
> 2. `DAPPERNESS_*` 各等级"每天山姆变化"出自 `tuning.lua:2196 ~ 2202` 的注释（计算公式 `100/(day_time*N)` 经 `sanity` 内部累积乘子换算后的标称值）；
> 3. `sanity` 组件对 `is_magic_dapperness` 的特殊处理在 21 章详述，本节只提及该字段存在；
> 4. `linkeditem` 组件在 23.9 详述。

---

#### 六、本节速查清单

```lua
-- 基础装备模板
inst:AddComponent("inventoryitem")
inst:AddComponent("equippable")
inst.components.equippable.equipslot = EQUIPSLOTS.HEAD     -- HANDS（默认） / HEAD / BODY / BEARD
inst.components.equippable:SetOnEquip(onequip_fn)           -- fn(inst, owner, from_ground)
inst.components.equippable:SetOnUnequip(onunequip_fn)       -- fn(inst, owner)
inst.components.equippable:SetOnPocket(onpocket_fn)         -- fn(inst, owner)
inst.components.equippable:SetOnEquipToModel(model_fn)      -- 一般用 simple_onequiptomodel

-- 三大基础属性
inst.components.equippable.dapperness = TUNING.DAPPERNESS_SMALL  -- 山姆值修正（每秒）
inst.components.equippable.walkspeedmult = 1.25                   -- 移速倍率
inst.components.equippable.insulated = true                       -- 电击免疫（不是温度！）

-- 进阶选项
inst.components.equippable.equipstack = true                      -- 堆叠物允许整堆装备（如吹箭）
inst.components.equippable.restrictedtag = "spiderwhisperer"      -- 限制角色 tag
inst.components.equippable.flipdapperonmerms = true                -- Merm 戴时 dapperness 反转
inst.components.equippable.is_magic_dapperness = true              -- 仅"魔法 dapperness"
inst.components.equippable:SetDappernessFn(function(inst, owner)
    return TheWorld.state.isfullmoon and -5 or 1
end)

-- 锁定不可卸下（自动监听 onremove 清理）
inst.components.equippable:SetPreventUnequipping(true)
-- 解锁
inst.components.equippable:SetPreventUnequipping(false)

-- 查询
inst.components.equippable:IsEquipped()
inst.components.equippable:IsInsulated()
inst.components.equippable:GetDapperness(owner, ignore_wetness)
inst.components.equippable:GetWalkSpeedMult()
inst.components.equippable:GetEquippedMoisture()
inst.components.equippable:ShouldPreventUnequipping()
inst.components.equippable:IsRestricted(target)              -- 想穿这件物品的玩家被限制?
inst.components.equippable:IsRestricted_FromLoad(target)     -- 读档恢复时的容忍版

-- 客户端可用方法（4 个）
inst.replica.equippable:EquipSlot()
inst.replica.equippable:IsEquipped()
inst.replica.equippable:IsRestricted(target)
inst.replica.equippable:ShouldPreventUnequipping()

-- 监听玩家身上的装备/卸下事件
player:ListenForEvent("equip", function(player, data)
    if data.item and data.item.prefab == "myhat" then
        -- data.item, data.eslot, data.no_animation
    end
end)
player:ListenForEvent("unequip", function(player, data)
    -- data.item, data.eslot, data.slip
end)

-- 监听物品自身的事件
inst:ListenForEvent("equipped", function(inst, data) -- data = { owner }
end)
inst:ListenForEvent("unequipped", function(inst, data) -- data = { owner }
end)

-- 自定义玩家速度修正器（挂在玩家 entity 上，两端可读）
player.inventory_EquippableWalkSpeedMultModifier = function(player, speed, item)
    return speed * (player.my_buff_mult or 1)
end

-- 自定义读档兼容（在 modmain 中）
SKILLTREE_EQUIPPABLE_RESTRICTED_TAGS["myrestricted_tag"] = "mycharacter_prefab"
```




## 23.5 Container 组件——背包、箱子、冰箱（containers.lua 数据表）

（待编写）

## 23.6 Finiteuses / Perishable——耐久与腐烂

（待编写）

## 23.7 LootDropper——掉落表与条件掉落

（待编写）

## 23.8 Upgrader / Upgradeable——物品升级系统

（待编写）

## 23.9 LinkedItem / LinkedItemManager——绑定特定玩家的物品系统

（待编写）

## 23.10 实战：创建带有特殊功能的自定义装备

（待编写）
