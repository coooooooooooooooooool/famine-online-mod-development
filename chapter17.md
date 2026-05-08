# 第17章 输入系统

## 17.1 TheInput 的核心 API——键盘、鼠标与手柄

### 本节导读

17.1 是输入章的"基础理论"——理解饥荒的输入系统是如何组织的，以及如何正确地在 mod 中监听和查询各种输入。

阅读路径：

> **新手**从 17.1.1-17.1.2 开始——理解 `TheInput` 是什么、三种输入类型有何区别、最常用的 `IsKeyDown` / `IsControlPressed` 怎么用；**进阶读者**继续看 17.1.3-17.1.4——掌握"回调式"监听（`AddKeyDownHandler`、`AddControlHandler`）与"轮询式"查询的区别，理解鼠标世界坐标和 entity 拾取，以及 `CONTROL_*` 体系如何统一不同设备的操作；**老手**看 17.1.5-17.1.6——理解手柄连接检测与虚拟控制方案、输入处理器的正确生命周期管理，以及何时该用 `RemoveHandler` 避免内存泄漏。

读完本节，你能：

- 知道 `TheInput`、`CONTROL_*`、`KEY_*`、`MOUSEBUTTON_*` 四者的关系
- 在 mod 中正确注册/注销键盘、鼠标和游戏操作的事件回调
- 用 `GetWorldPosition` 和 `GetWorldEntityUnderMouse` 写出鼠标交互逻辑
- 知道手柄适配的正确做法，以及输入处理器泄漏的防范方式

---

### 17.1.1 新手入门：TheInput 是什么，三种输入类型一览

#### TheInput 全局单例

`scripts/input.lua:740`：

```lua
TheInput = Input()
```

`TheInput` 是一个 `Input` 类的全局单例，**整个游戏只有这一个输入管理器**。所有键盘、鼠标、手柄的原始事件都由 C++ 引擎调用 `OnInputKey`、`OnMouseButton`、`OnControl` 等全局函数推入，再由 `TheInput` 分发给注册的回调函数。

你在 mod 代码里直接用 `TheInput:方法名(...)` 即可，无需 require。

#### 三种输入类型

饥荒的输入分三层：

| 层级 | 常量前缀 | 含义 | 场景 |
|------|---------|------|------|
| **游戏操作** | `CONTROL_*` | 逻辑操作（攻击、交互等），键盘/鼠标/手柄都能触发 | **推荐优先使用**，自动适配手柄 |
| **键盘** | `KEY_*` | 具体物理按键 | 功能键、调试键等无对应 CONTROL 的情况 |
| **鼠标** | `MOUSEBUTTON_*` | 鼠标按键和滚轮 | 特殊鼠标操作，不通过 CONTROL 体系时 |

**绝大多数 mod 应优先使用 `CONTROL_*`**，而不是直接监听 `KEY_*`——这样手柄用户也能使用你的功能。

#### 两种使用方式

| 方式 | 适用场景 | 函数 |
|------|----------|------|
| **轮询式**（主动查询） | 在 Update/StateMachine 里每帧检查 | `IsKeyDown` / `IsMouseDown` / `IsControlPressed` |
| **回调式**（被动监听） | 不需要持续检查，只需在"按下"或"松开"瞬间响应 | `AddKeyDownHandler` / `AddControlHandler` 等 |

> **新手记忆**：`TheInput` = 全局输入管理器；CONTROL > KEY > MOUSEBUTTON；轮询用 `IsXxx`，监听用 `AddXxxHandler`。

---

### 17.1.2 新手/进阶：键盘 API

#### KEY_* 常量

定义在 `scripts/constants.lua:285-390`，常用常量：

| 常量 | 值 | 说明 |
|------|-----|------|
| `KEY_A`-`KEY_Z` | 97-122 | 字母键（小写 ASCII 码） |
| `KEY_SPACE` | 32 | 空格 |
| `KEY_ENTER` | 13 | 回车 |
| `KEY_ESCAPE` | 27 | Esc |
| `KEY_TAB` | 9 | Tab |
| `KEY_BACKSPACE` | 8 | 退格 |
| `KEY_SHIFT` | 402 | Shift（左右通用） |
| `KEY_CTRL` | 401 | Ctrl（左右通用） |
| `KEY_ALT` | 400 | Alt（左右通用） |
| `KEY_KP_0`-`KEY_KP_9` | 256-265 | 小键盘数字 |
| `KEY_F1`-`KEY_F12` | 自行在 constants.lua 查询 | 功能键 |

> 注意：`KEY_A = 97` 就是 `string.byte("a")`——字母键常量即其 ASCII 值。

#### 方式一：轮询式——IsKeyDown

`scripts/input.lua:297-299`：

```lua
function Input:IsKeyDown(key)
    return TheSim:IsKeyDown(key)
end
```

**用法**：在 Update 或逐帧任务里每帧主动查询：

```lua
-- 在某个 DoPeriodicTask 的回调里
if TheInput:IsKeyDown(KEY_SHIFT) then
    -- Shift 正在被按住
end

-- 查多个修饰键
if TheInput:IsKeyDown(KEY_CTRL) and TheInput:IsKeyDown(KEY_SHIFT) then
    -- Ctrl+Shift 组合
end
```

这是 `scripts/debugkeys.lua:426-437` 里的真实用法（检查调试按键组合）。

**适合场景**：技能充能期间持续检查某个键是否仍在按住、相机跟随鼠标持续移动等。

#### 方式二：回调式——AddKeyDownHandler / AddKeyUpHandler

`scripts/input.lua:119-125`：

```lua
function Input:AddKeyUpHandler(key, fn)
    return self.onkeyup:AddEventHandler(key, fn)
end

function Input:AddKeyDownHandler(key, fn)
    return self.onkeydown:AddEventHandler(key, fn)
end
```

**用法**：注册按键瞬间触发的回调：

```lua
-- 参考用法（mod 中）
local handler = TheInput:AddKeyDownHandler(KEY_F, function()
    -- 每次按下 F 键时执行
    print("F key pressed!")
end)

-- 不再需要时必须注销！
handler:Remove()
```

`AddKeyDownHandler(key, fn)` 在**按键按下瞬间**调用 `fn`；`AddKeyUpHandler(key, fn)` 在**按键松开瞬间**调用。

#### 方式三：通用键盘回调——AddKeyHandler

`scripts/input.lua:127-129`：

```lua
function Input:AddKeyHandler(fn)
    return self.onkey:AddEventHandler("onkey", fn)
end
```

`fn` 的签名是 `fn(key, down)` ——`key` 是 `KEY_*` 常量，`down` 是 `true`（按下）/ `false`（松开）：

```lua
-- 参考用法（scripts/frontend.lua:213 的真实写法）
TheInput:AddKeyHandler(function(key, down)
    if key == KEY_ESCAPE and down then
        -- Esc 被按下
    end
end)
```

**适合**：需要同时响应"按下"和"松开"两个状态、或需要在同一个回调里处理多个键。

#### handler:Remove()——注销必须

所有 `AddXxxHandler` 都返回一个 `EventHandler` 对象（`scripts/events.lua:1-9`），**必须在不需要时调用 `handler:Remove()` 注销**，否则是内存泄漏：

```lua
local EventHandler = Class(function(self, event, fn, processor)
    self.event = event
    self.fn = fn
    self.processor = processor
end)

function EventHandler:Remove()
    self.processor:RemoveHandler(self)  -- 从 EventProcessor 的事件表里移除
end
```

标准写法（参考 `scripts/components/nis.lua:7-21`）：

```lua
-- 在 Class 构造函数里
self.inputhandlers = {}
table.insert(self.inputhandlers, TheInput:AddKeyDownHandler(KEY_F, function() ... end))
table.insert(self.inputhandlers, TheInput:AddKeyDownHandler(KEY_G, function() ... end))

-- 在 OnRemoveFromEntity 或 OnRemoveEntity 里
function MyComponent:OnRemoveFromEntity()
    for _, handler in pairs(self.inputhandlers) do
        handler:Remove()
    end
    self.inputhandlers = {}
end
```

> **新手/进阶记忆**：按下瞬间用 `AddKeyDownHandler`，松开瞬间用 `AddKeyUpHandler`，持续按住用 `IsKeyDown`。返回的 handler 必须保存并在 entity 销毁时调用 `:Remove()`。

---

### 17.1.3 进阶：鼠标 API

#### MOUSEBUTTON_* 常量

定义在 `scripts/constants.lua:393-397`：

| 常量 | 值 | 说明 |
|------|-----|------|
| `MOUSEBUTTON_LEFT` | 1000 | 鼠标左键 |
| `MOUSEBUTTON_RIGHT` | 1001 | 鼠标右键 |
| `MOUSEBUTTON_MIDDLE` | 1002 | 鼠标中键 |
| `MOUSEBUTTON_SCROLLUP` | 1003 | 滚轮向上 |
| `MOUSEBUTTON_SCROLLDOWN` | 1004 | 滚轮向下 |

#### 轮询式——IsMouseDown

`scripts/input.lua:293-295`：

```lua
function Input:IsMouseDown(button)
    return TheSim:GetMouseButtonState(button)
end
```

查询鼠标按键是否正在被按住：

```lua
if TheInput:IsMouseDown(MOUSEBUTTON_LEFT) then
    -- 鼠标左键正在按住
end
```

注意：鼠标只有在 `self.mouse_enabled` 为 true 时才有效（控制台平台上不启用）。

#### 鼠标坐标——GetScreenPosition / GetWorldPosition

`scripts/input.lua:246-254`：

```lua
-- 获取鼠标的屏幕像素坐标（左上角原点）
function Input:GetScreenPosition()
    local x, y = TheSim:GetPosition()
    return Vector3(x, y, 0)
end

-- 获取鼠标指向的世界 3D 坐标
function Input:GetWorldPosition()
    local x, y, z = TheSim:ProjectScreenPos(TheSim:GetPosition())
    return x ~= nil and y ~= nil and z ~= nil and Vector3(x, y, z) or nil
end
```

**`GetWorldPosition` 是 mod 中最常用的鼠标 API**，用于技能瞄准、AOE 范围预览等：

```lua
-- 在 playercontroller.lua:1644 的真实用法
local pos = TheInput:GetWorldPosition()
if pos then
    local x, y, z = pos:Get()
    SpawnPrefab("small_puff").Transform:SetPosition(x, y, z)
end
```

注意：在没有世界场景（如主菜单）时，`GetWorldPosition()` 返回 `nil`——务必做判空检查。

#### 获取鼠标下的 Entity

`scripts/input.lua:267-291`：

```lua
-- 获取鼠标悬停在的世界 entity（有 Transform 组件的 3D 物体）
function Input:GetWorldEntityUnderMouse()
    return self.mouse_enabled and
        self.hoverinst ~= nil and
        self.hoverinst.entity:IsValid() and
        self.hoverinst.entity:IsVisible() and
        self.hoverinst.Transform ~= nil and
        self.hoverinst or nil
end

-- 获取鼠标悬停在的 HUD entity（没有 Transform 的 UI 元素）
function Input:GetHUDEntityUnderMouse()
    return self.mouse_enabled and
        self.hoverinst ~= nil and
        self.hoverinst.entity:IsValid() and
        self.hoverinst.entity:IsVisible() and
        self.hoverinst.Transform == nil and
        self.hoverinst or nil
end
```

区别：
- `GetWorldEntityUnderMouse` → 返回有 `Transform` 的游戏物体（岩石、树、生物）
- `GetHUDEntityUnderMouse` → 返回没有 `Transform` 的 UI 元素

`hoverinst` 在 `OnUpdate` 里每帧更新（`scripts/input.lua:519-571`），当 hoverinst 变化时还会推送 `mouseover` 和 `mouseout` 事件。

#### 回调式——AddMouseButtonHandler / AddMoveHandler

```lua
-- 监听鼠标按键
local handler = TheInput:AddMouseButtonHandler(function(button, down, x, y)
    if button == MOUSEBUTTON_LEFT and down then
        -- 左键按下，x/y 是屏幕坐标
    end
end)

-- 监听鼠标移动
local movehandler = TheInput:AddMoveHandler(function(x, y)
    -- 鼠标移动到屏幕坐标 (x, y)
end)
```

注意：`AddMoveHandler` 的 `fn` 参数只在 `self.mouse_enabled == true` 时才被调用（见 `scripts/input.lua:155-159`）。

> **进阶记忆**：`GetWorldPosition()` 获取鼠标世界坐标、`GetWorldEntityUnderMouse()` 获取悬停物体，是 AOE 技能和自定义交互最常用的两个 API。记得判空 `GetWorldPosition()` 可能返回 nil。

---

### 17.1.4 进阶：CONTROL 体系——跨设备操作抽象

#### 为什么要用 CONTROL_* 而不是 KEY_*

`CONTROL_*` 是饥荒的"设备无关操作层"——同一个 `CONTROL_ACTION`，键盘上绑定到 Space，手柄上绑定到 A 键，鼠标上绑定到鼠标右键，三者都能触发同一个操作。

这意味着：

```lua
-- ✅ 推荐：用 CONTROL，键盘和手柄都能触发
if TheInput:IsControlPressed(CONTROL_ACTION) then
    -- 玩家按了"交互"键
end

-- ❌ 不推荐：直接用 KEY，手柄用户无法触发
if TheInput:IsKeyDown(KEY_SPACE) then
    -- 只有键盘用户按了 Space
end
```

#### 常用 CONTROL_* 常量（scripts/constants.lua:108-112）

| 常量 | 默认键盘 | 默认鼠标 | 含义 |
|------|---------|---------|------|
| `CONTROL_PRIMARY` (0) | - | 左键 | 主要操作（移动/攻击） |
| `CONTROL_SECONDARY` (1) | - | 右键 | 次要操作（交互） |
| `CONTROL_ATTACK` (2) | - | - | 强制攻击 |
| `CONTROL_INSPECT` (3) | - | - | 检查 |
| `CONTROL_ACTION` (4) | Space | - | 行动 |
| `CONTROL_SPRINT` | - | - | 冲刺（如有） |
| `CONTROL_FORCE_ATTACK` (39) | Ctrl+点击 | - | 强制攻击 |
| `CONTROL_CONTROLLER_ATTACK` (56) | - | 手柄X | 手柄攻击 |
| `CONTROL_CONTROLLER_ACTION` (57) | - | 手柄A | 手柄行动 |

#### 轮询式——IsControlPressed / GetAnalogControlValue

`scripts/input.lua:479-487`：

```lua
-- 查询数字型操作（按下/松开）
function Input:IsControlPressed(control)
    control = self:ResolveVirtualControls(control)
    return control ~= nil and TheSim:GetDigitalControl(control)
end

-- 查询模拟量操作（如摇杆轴、扳机量）
function Input:GetAnalogControlValue(control)
    control = self:ResolveVirtualControls(control)
    return control and TheSim:GetAnalogControl(control) or 0
end
```

`GetAnalogControlValue` 返回 0.0-1.0 的浮点数，适合读取手柄扳机的按压程度：

```lua
-- 参考用法（mod 中，读取缩放操作的模拟量）
local zoom = TheInput:GetAnalogControlValue(CONTROL_ZOOM_IN)
if zoom > 0.1 then
    -- 右扳机按压超过 10%
end
```

#### 回调式——AddControlHandler / AddGeneralControlHandler

`scripts/input.lua:139-145`：

```lua
-- 监听特定操作
function Input:AddControlHandler(control, fn)
    return self.oncontrol:AddEventHandler(control, fn)
end

-- 监听所有操作
function Input:AddGeneralControlHandler(fn)
    return self.oncontrol:AddEventHandler("oncontrol", fn)
end
```

`AddControlHandler` 的 `fn` 参数签名是 `fn(down, analogvalue)` ——`down` 是 `true`（按下）/ `false`（松开），`analogvalue` 是模拟量。

真实用法（参考 `scripts/components/maxwelltalker.lua:17-22`）：

```lua
-- 注册多个 CONTROL 回调，任意操作触发都跳过对话
self.inputhandlers = {}
for _, control in ipairs({ CONTROL_PRIMARY, CONTROL_SECONDARY, CONTROL_ACTION }) do
    table.insert(self.inputhandlers,
        TheInput:AddControlHandler(control, function()
            self:OnAnyInput()
        end)
    )
end
```

`AddGeneralControlHandler` 的 fn 签名是 `fn(control, down, analogvalue)`，适合需要"拦截所有操作"的场景（如全屏 UI 阻塞游戏输入时）。

> **进阶记忆**：优先用 `CONTROL_*` 而非 `KEY_*`——自动适配手柄。`IsControlPressed` 查当前帧，`AddControlHandler` 监听变化瞬间。`GetAnalogControlValue` 读摇杆/扳机的模拟量。

---

### 17.1.5 老手：手柄/控制器检测与适配

#### 连接状态检测

`scripts/input.lua:90-101`：

```lua
-- 手柄是否"已连接且已启用"（即当前活跃中）
function Input:ControllerAttached()
    if self.controllerid_cached ~= nil then
        return self.controllerid_cached > 0
    end
    return IsConsole() or TheInputProxy:IsAnyControllerActive()
end

-- 手柄是否物理连接（只看线，不看是否启用）
function Input:ControllerConnected()
    return IsConsole() or TheInputProxy:IsAnyControllerConnected()
end
```

两者区别：`ControllerConnected()` 只检测线是否插了，`ControllerAttached()` 还要检查是否在使用。**mod 中判断"当前是否手柄模式"应用 `ControllerAttached()`**：

```lua
if TheInput:ControllerAttached() then
    -- 当前玩家在用手柄
    -- 应当显示手柄按键图标提示（如 "按 A 键"）
else
    -- 键鼠模式
    -- 应当显示键盘按键图标提示（如 "按 Space 键"）
end
```

#### 获取控制器 ID

`scripts/input.lua:86-88`：

```lua
function Input:GetControllerID()
    return self.controllerid_cached or TheInputProxy:GetLastActiveControllerIndex() or 0
end
```

当需要给特定手柄发送振动、查询具体按键映射时使用。`0` 是键鼠设备，`> 0` 是手柄设备。

#### 鼠标在控制台平台上不可用

`scripts/input.lua:22`：

```lua
self.mouse_enabled = IsNotConsole() and not TheNet:IsDedicated()
```

控制台（PS、Xbox）和专用服务器上，`mouse_enabled = false`，因此：
- `GetWorldEntityUnderMouse()` 返回 nil
- `AddMoveHandler` 的 fn 永远不被调用
- `IsMouseDown` 永远返回 false

**mod 中如果有鼠标交互逻辑，必须用 `if IsNotConsole() then` 或检查 `TheInput:ControllerAttached()` 决定是否启用**，否则控制台用户会看到功能失效。

> **老手记忆**：`ControllerAttached` = 手柄当前活跃；`mouse_enabled` = PC 专属。控制台不支持鼠标，写鼠标交互时必须加平台判断。

---

### 17.1.6 老手：输入处理器的生命周期管理

#### EventHandler 的 Remove 模式

所有 `AddXxxHandler` 返回的是 `EventHandler` 对象（`scripts/events.lua:1-9`）：

```lua
local EventHandler = Class(function(self, event, fn, processor)
    self.event = event
    self.fn = fn
    self.processor = processor
end)

function EventHandler:Remove()
    self.processor:RemoveHandler(self)
end
```

`EventProcessor:AddEventHandler` 将 handler 放入 `self.events[event]` 哈希表（`scripts/events.lua:17-27`），`RemoveHandler` 则将其置 nil——这是一个"弱引用注册表"模式，**不调用 Remove 则 handler 的 fn 闭包永远被 EventProcessor 引用，不会被 GC 回收**，是常见的内存泄漏源。

#### 标准生命周期模式

**模式 1：Component 中的写法**（参考 `scripts/components/nis.lua:7-21`）

```lua
local MyComponent = Class(function(self, inst)
    self.inst = inst
    self.inputhandlers = {}

    -- 注册时保存 handler
    table.insert(self.inputhandlers,
        TheInput:AddKeyDownHandler(KEY_F, function()
            self:OnFKeyDown()
        end)
    )
end)

function MyComponent:OnRemoveFromEntity()
    -- entity 被移除时统一注销
    for _, handler in pairs(self.inputhandlers) do
        handler:Remove()
    end
    self.inputhandlers = {}
end
```

**模式 2：Widget 中的写法**（参考 `scripts/widgets/widget.lua` 的 followhandler 管理）

Widget 有自己的 `Kill()` 生命周期，应在 `Kill` 或 `OnStop` 里 Remove：

```lua
-- 参考用法（mod 中的 Widget）
function MyWidget:Init()
    self._controlhandler = TheInput:AddControlHandler(CONTROL_CANCEL, function()
        self:Close()
    end)
end

function MyWidget:Kill()
    if self._controlhandler then
        self._controlhandler:Remove()
        self._controlhandler = nil
    end
    MyWidget._base.Kill(self)
end
```

#### 全局 Handler 的特殊情况

有时你需要注册一个**永久存在**的全局输入处理器（如 mod 级别的调试快捷键），此时不需要 Remove，但必须确保 fn 闭包中引用的对象不会变成悬空引用：

```lua
-- modmain.lua 中注册全局调试键（永久存活，不需要 Remove）
TheInput:AddKeyDownHandler(KEY_F9, function()
    if ThePlayer then
        ThePlayer.components.health:SetMaxHealth(999)
    end
end)
```

`if ThePlayer` 那行保证了 fn 在 ThePlayer 不存在时（如切换地图瞬间）不会报错。

> **老手记忆**：`AddXxxHandler` 返回 handler，不 Remove = 内存泄漏。Component 在 `OnRemoveFromEntity` 里批量 Remove；Widget 在 `Kill()` 里 Remove；全局 Handler 不需要 Remove 但要在 fn 内部做判空防御。


## 17.2 键位绑定与 Mod 自定义快捷键（AddModRPCHandler + SendModRPCToServer）

### 本节导读

17.2 是"客户端按键 → 服务端逻辑"这条最常见 mod 链路的完整教程。理解这条链路，你的 mod 按键快捷键才能在联机模式下正常工作。

阅读路径：

> **新手**从 17.2.1-17.2.2 开始——理解为什么按键需要走 RPC、学会最简单的"按键 → SendModRPCToServer → 服务端处理"的三行模式；**进阶读者**继续看 17.2.3-17.2.4——理解 RPC 注册/查询 API 的完整用法，并看懂一个含参数传递的完整技能快捷键示例；**老手**看 17.2.5-17.2.6——掌握服务端反向推送客户端（`SendModRPCToClient`）、速率限制机制，以及什么参数类型可以/不可以通过 RPC 传递。

读完本节，你能：

- 知道"键盘事件在客户端，游戏逻辑在服务端"的根本原因
- 正确使用 `AddModRPCHandler`、`GetModRPC`、`SendModRPCToServer` 三件套
- 写出一个完整的"按 F 键触发技能"mod 模板
- 知道 RPC 的速率限制和参数类型限制，避免踩坑

---

### 17.2.1 新手：为什么按键需要走 RPC

饥荒联机版的核心规则：

| 发生在... | 负责... |
|---------|---------|
| **客户端（每个玩家本地）** | 按键检测、UI 渲染、动画播放 |
| **服务端（权威实例）** | 生命值变化、物品增减、伤害计算、存档 |

这意味着：当玩家按下你 mod 的技能快捷键时，**按键事件只在他自己的客户端触发**。如果你直接在 `AddKeyDownHandler` 的回调里扣血、加 buff、生成 entity——这些操作只会在本地生效，**其他玩家看不到、服务端不知道、重新加载后会消失**。

正确做法是：

```
[客户端] 玩家按下快捷键
    ↓
SendModRPCToServer(...)   ── 告诉服务端"发生了什么"
    ↓
[服务端] AddModRPCHandler 里的 fn(player, ...) 被调用
    ↓
服务端修改 player 的状态（扣血、加 buff、生成 entity...）
    ↓
服务端变化通过网络同步给所有客户端
```

单人游戏也适用这个模式（单人时客户端=服务端，RPC 内部短路直接调用 fn），所以统一用 RPC 是兼容性最好的写法。

> **新手记忆**：按键在客户端，逻辑在服务端，两者靠 RPC 连接。`SendModRPCToServer` = "客户端喊话给服务端"。

---

### 17.2.2 新手/进阶：AddModRPCHandler + GetModRPC——注册和查询 RPC

#### AddModRPCHandler——注册服务端处理器

`scripts/networkclientrpc.lua:1834-1849`：

```lua
function AddModRPCHandler(namespace, name, fn)
    if MOD_RPC[namespace] == nil then
        MOD_RPC[namespace] = {}
        MOD_RPC_HANDLERS[namespace] = {}
        -- ...
    end

    table.insert(MOD_RPC_HANDLERS[namespace], fn)
    MOD_RPC[namespace][name] = { namespace = namespace, id = #MOD_RPC_HANDLERS[namespace] }

    -- 每注册一个 mod RPC，速率上限 +5（见 17.2.6）
    RPC_QUEUE_RATE_LIMIT = RPC_QUEUE_RATE_LIMIT + RPC_QUEUE_RATE_LIMIT_PER_MOD
end
```

三个参数：

| 参数 | 类型 | 含义 |
|------|------|------|
| `namespace` | string | 命名空间，通常填你的 mod 名（避免与其他 mod 冲突） |
| `name` | string | 这个 RPC 操作的名字（你自己起的，如 `"USE_SKILL"`, `"HEAL_SELF"`） |
| `fn` | function | 服务端接收到 RPC 时调用的函数，参数为 `fn(player, ...)` |

`fn(player, ...)` 的 `player` 是**发出这条 RPC 的玩家实体**（服务端上的 entity 对象），`...` 是客户端发送时附带的额外参数。

**必须在 `modmain.lua` 里调用**（或者在 `modmain.lua` 加载的文件里）——RPC 注册必须发生在游戏加载时，不能在运行时动态注册。

#### GetModRPC——获取 id_table

`scripts/networkclientrpc.lua:1968-1970`：

```lua
function GetModRPC(namespace, name)
    return MOD_RPC[namespace][name]
end
```

返回 `{ namespace = namespace, id = <序号> }`，这就是发送时用的 **id_table**，传给 `SendModRPCToServer` 的第一个参数。

> 旧版 mod 直接用 `MOD_RPC[namespace][name]` 访问——这是遗留写法，`GetModRPC` 是推荐用法（见 `scripts/modutil.lua:906`：`env.MOD_RPC = MOD_RPC --legacy`）。

> **进阶记忆**：`AddModRPCHandler` = "告诉服务端，收到这个信号后做什么"；`GetModRPC` = "给我发信号用的地址"。

---

### 17.2.3 进阶：SendModRPCToServer——从客户端发送 RPC

`scripts/networkclientrpc.lua:1881-1884`：

```lua
function SendModRPCToServer(id_table, ...)
    assert(id_table.namespace ~= nil and
           MOD_RPC_HANDLERS[id_table.namespace] ~= nil and
           MOD_RPC_HANDLERS[id_table.namespace][id_table.id] ~= nil)
    TheNet:SendModRPCToServer(id_table.namespace, id_table.id, ...)
end
```

参数说明：

| 参数 | 类型 | 含义 |
|------|------|------|
| `id_table` | table | `GetModRPC(namespace, name)` 的返回值 |
| `...` | 可变参数 | 传给服务端 handler 的额外数据（类型有限制，见 17.2.6） |

**调用位置**：必须在**客户端**调用，即 `if not TheWorld.ismastersim then` 成立的条件下（或者不判断，DST 单人时两端合一，发给自己也没关系）：

```lua
-- 参考写法（mod 中）
-- 客户端直接调用即可，联机模式下会走网络，单人模式下内部直接调用 fn
SendModRPCToServer(GetModRPC("MY_MOD", "USE_SKILL"))
```

`assert` 会在 id_table 无效时报错——确保 `AddModRPCHandler` 已在 `GetModRPC` 之前调用，否则 `MOD_RPC["MY_MOD"]` 不存在，`GetModRPC` 返回 nil，`SendModRPCToServer(nil, ...)` 会触发 assert 报错。

---

### 17.2.4 进阶：完整示例——按 F 键触发服务端技能

下面是一个完整的"按快捷键 → 服务端执行 → 玩家加血"的 mod 模板，展示了从注册到调用的全链路：

**modmain.lua**

```lua
-- 步骤 1：注册 RPC（在游戏加载时）
-- fn 在服务端执行，player 是发消息的玩家，amount 是客户端传来的参数
AddModRPCHandler("MY_MOD", "HEAL_PLAYER", function(player, amount)
    -- 所有逻辑都在服务端执行
    if player and player:IsValid() and player.components.health then
        player.components.health:DoDelta(amount or 20)
        print("Healed", player.name, "for", amount or 20, "HP")
    end
end)

-- 步骤 2：注册按键回调（在客户端监听按键）
-- 注意：AddKeyDownHandler 在客户端执行
local function SetupKeyHandler()
    if TheInput ~= nil then
        TheInput:AddKeyDownHandler(KEY_F, function()
            -- 步骤 3：按键按下时，从客户端向服务端发送 RPC
            -- 第二个参数 20 是传给 handler 的 amount
            SendModRPCToServer(GetModRPC("MY_MOD", "HEAL_PLAYER"), 20)
        end)
    end
end

-- 在玩家加载后设置按键（确保 TheInput 已经初始化）
AddSimPostInit(function()
    SetupKeyHandler()
end)
```

**流程解析**：

1. `AddModRPCHandler("MY_MOD", "HEAL_PLAYER", fn)` — 在所有客户端和服务端的 modmain.lua 加载时注册，服务端会实际调用 fn，客户端只是注册 id 映射
2. `TheInput:AddKeyDownHandler(KEY_F, fn)` — 在本地客户端注册按键回调，只有本地玩家按 F 时触发
3. `SendModRPCToServer(GetModRPC(...), 20)` — 向服务端发送"HEAL_PLAYER"信号，附带参数 20
4. 服务端收到后，在下一个逻辑 tick 调用 `fn(player, 20)`，执行实际的加血逻辑

**单人游戏**：客户端=服务端，`SendModRPCToServer` 内部等效于直接调用 fn，流程完全相同。

> **进阶记忆**：`AddModRPCHandler` 在 modmain 最顶部注册，`AddKeyDownHandler` 在 `AddSimPostInit` 里注册，`SendModRPCToServer(GetModRPC(...), 参数)` 在按键回调里发送。三步顺序不能颠倒。

---

### 17.2.5 老手：AddClientModRPCHandler——服务端反向推送客户端

有时服务端需要主动通知客户端（比如技能命中后在客户端播放特效）。此时用**客户端 RPC**：

| 函数 | 方向 | 说明 |
|------|------|------|
| `AddModRPCHandler` + `SendModRPCToServer` | 客户端 → 服务端 | 最常用 |
| `AddClientModRPCHandler` + `SendModRPCToClient` | 服务端 → 客户端 | 服务端主动推送 |
| `AddShardModRPCHandler` + `SendModRPCToShard` | 分片 → 分片 | 洞穴/主世界通信 |

`AddClientModRPCHandler` 的注册方式（`scripts/networkclientrpc.lua:1851-1864`）：

```lua
function AddClientModRPCHandler(namespace, name, fn)
    -- 与 AddModRPCHandler 结构一致
    -- fn 在客户端执行，没有 player 参数
end
```

`fn` 在客户端调用，参数为服务端传来的额外数据（**没有** player 参数，因为客户端不知道是谁触发的）。

完整示例：

```lua
-- modmain.lua
-- 注册客户端 RPC handler
AddClientModRPCHandler("MY_MOD", "SHOW_SKILL_FX", function(fx_name, x, z)
    -- 客户端执行：在世界位置 (x, 0, z) 生成特效
    if fx_name and x and z then
        SpawnPrefab(fx_name).Transform:SetPosition(x, 0, z)
    end
end)

-- 在服务端某个技能命中逻辑里（已在服务端上下文中）：
-- SendModRPCToClient(GetClientModRPC("MY_MOD", "SHOW_SKILL_FX"), "sparks", target.Transform:GetWorldPosition())
```

注意：`SendModRPCToClient` 会广播给**所有客户端**（不是特定某个），适合需要所有人都看到的效果。

---

### 17.2.6 老手：速率限制与参数类型约束

#### 速率限制

`scripts/networkclientrpc.lua:1662-1663`：

```lua
local RPC_QUEUE_RATE_LIMIT = 20 -- Per logic tick.
local RPC_QUEUE_RATE_LIMIT_PER_MOD = 5 -- +this for every mod RPC added.
```

每个逻辑 tick（约 1/15 秒）每个玩家最多发 20 条 RPC，每注册一个 mod RPC 额外 +5 条上限。**超过上限的 RPC 会被静默丢弃**，并在首次超限时打印警告。

这意味着：
- **不要在 Update/PeriodicTask 里每帧发送 RPC** ——轮询式检测 + 每帧发送会触发速率限制
- **只在"按下瞬间"触发一次**——在 `AddKeyDownHandler` 里发（而非 `IsKeyDown` 每帧检查后发）
- 技能有冷却时间时，也在客户端加一个简单的冷却检查，防止玩家快速按键发送大量 RPC

#### 参数类型约束

`SendModRPCToServer(id_table, ...)` 的 `...` 参数通过 C++ 网络层序列化，支持的类型（对应 `networkclientrpc.lua:4-13` 的校验函数）：

| 类型 | 校验函数 | 说明 |
|------|---------|------|
| `boolean` 或 `nil` | `checkbool` / `optbool` | true/false/nil |
| `number` | `checknumber` | 浮点数 |
| 非负整数 | `checkuint` | 物品槽位等 |
| `string` | `checkstring` | 字符串 |
| entity | `checkentity` | Lua table，通过 GUID 传输 |

**不支持**：
- Lua table（除 entity 外）
- function
- userdata

如果你需要传递多个数字（比如 x、y、z 坐标），直接作为独立参数传递：

```lua
-- ✅ 正确：分开传 3 个数字
SendModRPCToServer(GetModRPC("MY_MOD", "SPAWN_AT"), x, y, z)

-- 对应 handler：
AddModRPCHandler("MY_MOD", "SPAWN_AT", function(player, x, y, z)
    SpawnPrefab("sparks").Transform:SetPosition(x, y, z)
end)

-- ❌ 错误：不能传 table
SendModRPCToServer(GetModRPC("MY_MOD", "SPAWN_AT"), { x = x, y = y, z = z })
```

> **老手记忆**：速率限制 = 每 tick 20 条，别在循环里发 RPC；参数类型 = bool/number/string/entity，不支持 table，多个值就分开传。


## 17.3 鼠标事件处理：点击、悬停、拖拽

### 本节导读

17.3 深入鼠标的三种核心交互模式——点击、悬停和拖拽。17.1 只介绍了 API 签名，本节重点在"如何用这些 API 实现真实的交互逻辑"。

阅读路径：

> **新手**看 17.3.1-17.3.2——掌握最基础的"监听点击 + 获取点击位置"和"检测悬停 entity"；**进阶读者**继续看 17.3.3-17.3.4——理解双击检测、`mouseover`/`mouseout` 事件机制，以及 AnimState 悬停高亮 symbol；**老手**看 17.3.5-17.3.6——理解拖拽检测的"按住计时"模式、Widget 中鼠标跟随实现，以及一个完整的"鼠标选取目标并触发技能"示例。

---

### 17.3.1 新手：鼠标点击——AddMouseButtonHandler 与 CONTROL_PRIMARY

#### 方式一：监听原始鼠标按键（AddMouseButtonHandler）

`scripts/input.lua:131-133`：

```lua
function Input:AddMouseButtonHandler(fn)
    return self.onmousebutton:AddEventHandler("onmousebutton", fn)
end
```

`fn(button, down, x, y)` 的四个参数：

| 参数 | 类型 | 含义 |
|------|------|------|
| `button` | number | `MOUSEBUTTON_*` 常量（1000=左键，1001=右键，1002=中键，1003=滚轮上，1004=滚轮下） |
| `down` | bool | `true` = 按下，`false` = 松开 |
| `x` | number | 鼠标屏幕 X 坐标（像素，左上角原点） |
| `y` | number | 鼠标屏幕 Y 坐标（像素，左上角原点） |

```lua
-- 参考用法（mod 中）
local handler = TheInput:AddMouseButtonHandler(function(button, down, x, y)
    if button == MOUSEBUTTON_LEFT and down then
        -- 左键按下
        local worldpos = TheInput:GetWorldPosition()
        if worldpos then
            print("左键点击世界坐标", worldpos:Get())
        end
    elseif button == MOUSEBUTTON_SCROLLUP then
        -- 滚轮向上
    end
end)
```

注意：`AddMouseButtonHandler` 在 `mouse_enabled == false` 时（控制台平台）其 fn 永远不被调用（见 `scripts/input.lua:179-184`）。

#### 方式二：通过 CONTROL_PRIMARY 监听左键（更推荐）

在"按左键执行游戏操作"的场景中，应优先使用 `CONTROL_PRIMARY`（控制器也能触发）：

```lua
-- 参考用法（mod 中）
local handler = TheInput:AddControlHandler(CONTROL_PRIMARY, function(down)
    if down then
        -- 左键或手柄对应按键按下
    end
end)
```

两种方式的区别：

| `AddMouseButtonHandler` | `AddControlHandler(CONTROL_PRIMARY)` |
|------------------------|-------------------------------------|
| 直接拿到屏幕坐标 `(x, y)` | 不直接给坐标（需调 `GetWorldPosition`） |
| 只在 PC 鼠标模式生效 | 键盘+鼠标+手柄都能触发 |
| 适合：纯 UI 点击、需要坐标的场景 | 适合：游戏内操作（攻击、交互） |

> **新手记忆**：纯 UI 交互用 `AddMouseButtonHandler`，游戏内操作用 `AddControlHandler(CONTROL_PRIMARY)`，两者配合 `GetWorldPosition` 获取坐标。

---

### 17.3.2 新手/进阶：鼠标坐标与 Entity 拾取

#### 获取鼠标世界坐标

`scripts/input.lua:251-254`：

```lua
function Input:GetWorldPosition()
    local x, y, z = TheSim:ProjectScreenPos(TheSim:GetPosition())
    return x ~= nil and y ~= nil and z ~= nil and Vector3(x, y, z) or nil
end
```

典型用法（来自 `scripts/components/playercontroller.lua:4107`）：

```lua
local pt = TheInput:GetWorldPosition()
if pt ~= nil then
    local x, y, z = pt:Get()
    -- 在鼠标位置生成特效
    SpawnPrefab("small_puff").Transform:SetPosition(x, y, z)
end
```

**必须做 nil 检查**——在切换地图、UI 全屏遮挡、没有世界场景时返回 nil。

#### 获取鼠标下所有 Entity

`scripts/input.lua:263-265`：

```lua
function Input:GetAllEntitiesUnderMouse()
    return self.mouse_enabled and self.entitiesundermouse or {}
end
```

返回一个 table，里面是鼠标位置所有 entity（按渲染层从前到后排列），`entitiesundermouse` 由 `OnUpdate` 每帧通过 `TheSim:GetEntitiesAtScreenPoint` 更新（`scripts/input.lua:515-517`）。

#### 获取鼠标下最前面的 Entity

`scripts/input.lua:267-274`：

```lua
function Input:GetWorldEntityUnderMouse()
    return self.mouse_enabled and
        self.hoverinst ~= nil and
        self.hoverinst.entity:IsValid() and
        self.hoverinst.entity:IsVisible() and
        self.hoverinst.Transform ~= nil and    -- 有 Transform = 游戏世界物体
        self.hoverinst or nil
end
```

`hoverinst` 是当前帧鼠标下"最优先"的 entity（会处理 `CanMouseThrough` 的穿透规则，见 `input.lua:519-571`）。

```lua
-- 参考用法（mod 中）：检查鼠标是否悬停在敌人上
local target = TheInput:GetWorldEntityUnderMouse()
if target and target:HasTag("monster") then
    print("鼠标悬停在敌人上:", target.name)
end
```

> **进阶记忆**：`GetAllEntitiesUnderMouse()` 返回所有，`GetWorldEntityUnderMouse()` 返回最前面的（带 CanMouseThrough 过滤）。两者都在 `mouse_enabled == false` 时返回空。

---

### 17.3.3 进阶：悬停（Hover）机制——mouseover/mouseout 事件

#### hoverinst 更新机制

每帧 `Input:OnUpdate()` (`scripts/input.lua:519-571`) 会：
1. 调用 `TheSim:GetEntitiesAtScreenPoint` 获取鼠标下所有 entity
2. 根据 `CanMouseThrough` 规则找出最优先 entity `inst`
3. 如果 `inst ~= self.hoverinst`（悬停目标发生了变化）：
   - 向新目标 `inst` 推送 `"mouseover"` 事件（`input.lua:561`）
   - 向旧目标 `self.hoverinst` 推送 `"mouseout"` 事件（`input.lua:565`）
   - 更新 `self.hoverinst = inst`

关键代码（`scripts/input.lua:559-569`）：

```lua
if inst ~= self.hoverinst then
    if inst ~= nil and inst.Transform ~= nil then
        inst:PushEvent("mouseover")       -- 鼠标进入新 entity
    end

    if self.hoverinst ~= nil and self.hoverinst.Transform ~= nil then
        self.hoverinst:PushEvent("mouseout")  -- 鼠标离开旧 entity
    end

    self.hoverinst = inst
end
```

#### 在 prefab 中监听 mouseover/mouseout

```lua
-- 参考用法（mod 中的 prefab）
-- 仅在客户端（有 Transform 且 mouse_enabled 时）才能接收到这两个事件
inst:ListenForEvent("mouseover", function(inst)
    -- 鼠标移入此 entity
    inst.AnimState:SetMultColour(1.2, 1.2, 1.2, 1)  -- 高亮
end)

inst:ListenForEvent("mouseout", function(inst)
    -- 鼠标移出此 entity
    inst.AnimState:SetMultColour(1, 1, 1, 1)   -- 恢复
end)
```

注意：`mouseover`/`mouseout` 只会在有 `Transform` 的游戏世界 entity 上触发（`input.lua:560`：`inst.Transform ~= nil`），HUD/UI entity 不会收到。

#### AnimState "mouseover" symbol——官方悬停高亮方式

游戏中大量 entity（树、石头、门）使用一种更优雅的悬停高亮方式：在动画里内置一个 `"mouseover"` symbol，平时隐藏，悬停时显示：

```lua
-- 以 gestalt.lua:141 为例（初始化时隐藏）
inst.AnimState:Hide("mouseover")

-- 以 deciduoustrees.lua:281 为例（鼠标悬停时显示）
inst.AnimState:OverrideSymbol("mouseover", "tree_leaf_trunk_build", "toggle_mouseover")

-- 鼠标移出时清除
inst.AnimState:ClearOverrideSymbol("mouseover")
```

这是**纯客户端的视觉效果**，不需要额外的事件监听——引擎内部在 hoverinst 变化时自动控制 symbol 的显隐（由游戏的 `playercontroller` 中的 cursor 系统处理）。

> **进阶记忆**：`mouseover`/`mouseout` entity 事件由 `input.lua` 的 `OnUpdate` 推送，每帧最多推送一次（hoverinst 变化时）。自定义高亮用 `ListenForEvent("mouseover")` + `AnimState:SetMultColour`；使用内置 symbol 方式更优雅但需要美术配合。

---

### 17.3.4 进阶：双击检测

饥荒引擎在 `scripts/constants.lua:2294-2295` 定义了双击的时间和位置阈值：

```lua
DOUBLE_CLICK_TIMEOUT = .5       -- 两次点击间隔不超过 0.5 秒
DOUBLE_CLICK_POS_THRESHOLD = 3  -- 两次点击位置差不超过 3 像素
```

`PlayerController` 内部用 `startdoubleclicktime` 和 `startdoubleclickpos` 记录上次点击（`scripts/components/playercontroller.lua:4728-4729`）：

```lua
if (laststartdoubleclicktime and t < laststartdoubleclicktime + DOUBLE_CLICK_TIMEOUT) and
   (laststartdoubleclickpos and
    math.abs(laststartdoubleclickpos.x - scrnx) <= DOUBLE_CLICK_POS_THRESHOLD and
    math.abs(laststartdoubleclickpos.y - scrny) <= DOUBLE_CLICK_POS_THRESHOLD)
then
    -- 双击！
end
```

**mod 中自己实现双击检测的参考模式**：

```lua
-- 参考用法（mod 中）
local last_click_time = nil
local DOUBLE_CLICK_THRESHOLD = 0.4  -- 自定义阈值

TheInput:AddMouseButtonHandler(function(button, down, x, y)
    if button == MOUSEBUTTON_LEFT and down then
        local t = GetTime()
        if last_click_time and t - last_click_time < DOUBLE_CLICK_THRESHOLD then
            -- 双击！
            last_click_time = nil
        else
            last_click_time = t
        end
    end
end)
```

> **进阶记忆**：官方双击阈值 `DOUBLE_CLICK_TIMEOUT = .5` 秒，位置误差 3 像素（`constants.lua:2294`）。mod 自己实现双击只需记录上次按下时间。

---

### 17.3.5 老手：拖拽检测——按住计时模式

#### 游戏内的地面拖拽走路

`scripts/components/playercontroller.lua:1` 定义了按住判断为"拖拽"的时间阈值：

```lua
local START_DRAG_TIME = 8 * FRAMES   -- 约 8/30 ≈ 0.27 秒
```

拖拽走路的检测逻辑（`playercontroller.lua:2976-2981`）：

```lua
if not self.draggingonground and self.startdragtime ~= nil and
   TheInput:IsControlPressed(CONTROL_PRIMARY) then
    local now = GetTime()
    if now - self.startdragtime > START_DRAG_TIME then
        TheFrontEnd:LockFocus(true)
        self.draggingonground = true   -- 确认进入"拖拽走路"状态
    end
end
```

模式是：
1. 鼠标按下 → 记录 `startdragtime = GetTime()`
2. 每帧检查：如果 `CONTROL_PRIMARY` 仍按着 AND 已持续超过 `START_DRAG_TIME` → 确认拖拽
3. 鼠标松开 → 清除 `draggingonground` 和 `startdragtime`

#### mod 中自己实现"按住拖拽"

```lua
-- 参考用法（mod 中）
local is_dragging = false
local drag_start_time = nil
local DRAG_THRESHOLD = 8 * FRAMES   -- 同官方阈值

-- 按下时记录开始时间
TheInput:AddControlHandler(CONTROL_PRIMARY, function(down)
    if down then
        drag_start_time = GetTime()
    else
        -- 松开，结束拖拽
        if is_dragging then
            is_dragging = false
            OnDragEnd()
        end
        drag_start_time = nil
    end
end)

-- 每帧检查（在 DoPeriodicTask 或 Update 中）
local function OnUpdate()
    if drag_start_time ~= nil and not is_dragging then
        if GetTime() - drag_start_time > DRAG_THRESHOLD
           and TheInput:IsControlPressed(CONTROL_PRIMARY) then
            is_dragging = true
            OnDragStart()
        end
    end
end
```

#### Widget 跟随鼠标——FollowMouse / StopFollowMouse

`scripts/widgets/widget.lua:534-546`：

```lua
function Widget:FollowMouse()
    if self.followhandler == nil then
        -- 注册鼠标移动回调，每次鼠标移动更新 widget 位置
        self.followhandler = TheInput:AddMoveHandler(function(x, y) self:UpdatePosition(x, y) end)
        self:SetPosition(TheInput:GetScreenPosition())  -- 立即对齐当前鼠标位置
    end
end

function Widget:StopFollowMouse()
    if self.followhandler ~= nil then
        self.followhandler:Remove()  -- 注销移动回调
        self.followhandler = nil
    end
end
```

这是做"拖拽物品"UI 的标准做法：
1. 鼠标按下 item → `item_widget:FollowMouse()`
2. 鼠标松开 → `item_widget:StopFollowMouse()`

```lua
-- 参考用法（mod Widget 中）
function MyDraggableWidget:OnMouseButton(button, down, x, y)
    if button == MOUSEBUTTON_LEFT then
        if down then
            self:FollowMouse()
        else
            self:StopFollowMouse()
            self:OnDropped(x, y)  -- 处理放下逻辑
        end
        return true  -- 消耗事件，不传递给父级
    end
end
```

> **老手记忆**：地面拖拽走路阈值 = `8 * FRAMES ≈ 0.27` 秒（`playercontroller.lua:1`）。Widget 拖拽用 `FollowMouse()`/`StopFollowMouse()` 管理 `AddMoveHandler` 的生命周期。

---

### 17.3.6 老手：完整示例——鼠标选取目标触发技能

下面是一个综合运用本节知识的完整示例：**按 F 键后，通过单击鼠标左键选取目标 entity，释放技能**。

**逻辑流程**：
1. 按下 F 键进入"技能瞄准模式"
2. 鼠标悬停 entity 时高亮
3. 左键点击选定 entity → 发送 RPC → 服务端执行技能

```lua
-- modmain.lua

-- 注册服务端 RPC
AddModRPCHandler("MY_MOD", "CAST_SKILL", function(player, target)
    if target and target:IsValid() and player.components.health then
        -- 技能：对目标造成 50 伤害
        if target.components.health then
            target.components.health:DoDelta(-50, false, "skill", player)
        end
    end
end)

-- 技能瞄准状态管理
local is_aiming = false
local aiming_handler = nil
local hover_handler = nil
local last_highlighted = nil

local function HighlightTarget(target)
    if last_highlighted and last_highlighted:IsValid() then
        last_highlighted.AnimState:SetMultColour(1, 1, 1, 1)  -- 取消高亮
    end
    if target and target:IsValid() and target.AnimState then
        target.AnimState:SetMultColour(1.3, 0.8, 0.8, 1)  -- 红色高亮
        last_highlighted = target
    else
        last_highlighted = nil
    end
end

local function ExitAimingMode()
    is_aiming = false
    HighlightTarget(nil)  -- 清除高亮
    if aiming_handler then
        aiming_handler:Remove()
        aiming_handler = nil
    end
end

local function EnterAimingMode()
    is_aiming = true

    -- 在瞄准模式下监听左键点击
    aiming_handler = TheInput:AddMouseButtonHandler(function(button, down, x, y)
        if button == MOUSEBUTTON_LEFT and down then
            local target = TheInput:GetWorldEntityUnderMouse()
            if target and target:HasTag("monster") then
                -- 发送技能 RPC
                SendModRPCToServer(GetModRPC("MY_MOD", "CAST_SKILL"), target)
            end
            ExitAimingMode()
        elseif button == MOUSEBUTTON_RIGHT and down then
            -- 右键取消
            ExitAimingMode()
        end
    end)
end

AddSimPostInit(function()
    -- F 键切换瞄准模式
    TheInput:AddKeyDownHandler(KEY_F, function()
        if is_aiming then
            ExitAimingMode()
        else
            EnterAimingMode()
        end
    end)
end)
```

> **老手记忆**：鼠标选取目标 = `GetWorldEntityUnderMouse()` + `ListenForEvent("mouseover")`；瞄准模式 = 临时注册 `AddMouseButtonHandler`，确认或取消后立即 `handler:Remove()`。模式退出时清理所有临时 handler，避免泄漏。


## 17.4 手柄/控制器适配

### 本节导读

饥荒联机版支持键盘鼠标、Xbox/PlayStation 手柄、Nintendo Switch Joy-Con 等多种输入设备，并且玩家可以在中途随时切换。17.4 教你写出在所有设备上都能正常运行的 mod 输入逻辑，包括正确使用 CONTROL 体系统一适配、动态检测控制器连接状态、显示"当前设备对应按键名"的提示文字。

阅读路径：

> **新手**从 17.4.1-17.4.2 开始——掌握平台检测函数和 `ControllerAttached()` 的基本用法，能写出"手柄/键鼠不同分支"的判断逻辑；**进阶读者**继续看 17.4.3-17.4.4——理解 `CONTROL_*` 如何在键鼠和手柄上映射不同物理按键、以及 `ResolveVirtualControls` 和 7 档控制方案的工作原理；**老手**看 17.4.5-17.4.6——掌握 `GetLocalizedControl` 动态显示"按哪个键"的提示文字，以及手柄适配的完整 mod 实践模板。

读完本节，你能：

- 知道饥荒的所有平台标识符，写出不同平台的条件分支
- 正确区分 `ControllerAttached`（活跃）和 `ControllerConnected`（物理连接）
- 理解 CONTROL 体系如何屏蔽不同设备的差异，以及虚拟控制和控制方案的含义
- 用 `GetLocalizedControl` 在 UI 里自动显示当前设备对应的按键名

---

### 17.4.1 新手：平台检测函数一览

饥荒定义了一系列全局函数用于平台检测，位于 `scripts/main.lua:9-51`：

```lua
function IsConsole()
    return PLATFORM == "PS4" or PLATFORM == "XBONE" or PLATFORM == "SWITCH"
end

function IsNotConsole()
    return not IsConsole()
end

function IsPS4()    return PLATFORM == "PS4"         end
function IsPS5()    return PLATFORM == "PS5"         end
function IsPSN()    return IsPS4() or IsPS5()        end
function IsXB1()    return PLATFORM == "XBONE"       end
function IsSteam()  return PLATFORM == "WIN32_STEAM" or PLATFORM == "LINUX_STEAM" or PLATFORM == "OSX_STEAM" end
function IsWin32()  return PLATFORM == "WIN32_STEAM" or PLATFORM == "WIN32_RAIL"  end
function IsLinux()  return PLATFORM == "LINUX_STEAM" end
function IsRail()   return PLATFORM == "WIN32_RAIL"  end
function IsSteamDeck() return IS_STEAM_DECK          end
```

所有 `PLATFORM` 值：

| PLATFORM 值 | 平台 |
|------------|------|
| `"WIN32_STEAM"` | Windows Steam |
| `"LINUX_STEAM"` | Linux Steam |
| `"OSX_STEAM"` | macOS Steam |
| `"WIN32_RAIL"` | 国服（网易渠道） |
| `"PS4"` | PlayStation 4 |
| `"PS5"` | PlayStation 5 |
| `"XBONE"` | Xbox One |
| `"SWITCH"` | Nintendo Switch |

`IsConsole()` 返回 `true` 的平台（PS4、XBONE、SWITCH）有以下特性：
- 没有鼠标（`mouse_enabled` 强制为 false）
- 必须用手柄操作
- 键盘 API（`AddKeyDownHandler` 等）在这些平台上无法正常工作

#### mod 中的平台分支

```lua
if IsConsole() then
    -- 控制台专属逻辑（手柄用户）
elseif IsSteamDeck() then
    -- Steam Deck：是 Linux Steam，但也支持手柄，通常需要特殊处理
elseif IsNotConsole() then
    -- PC 平台（Windows/Linux/Mac）
end
```

> **新手记忆**：`IsConsole()` = PS/Xbox/Switch，这些平台没有鼠标；`IsNotConsole()` = PC；`IsSteamDeck()` 是 Linux + 触摸屏 + 手柄的特殊组合。PC 上的 mod 用户最多，但别忘了手柄也是常见输入方式（Steam Big Picture 模式下 PC 玩家也常用手柄）。

---

### 17.4.2 新手/进阶：ControllerAttached——实时检测手柄模式

#### ControllerAttached vs ControllerConnected

`scripts/input.lua:90-101`：

```lua
-- 手柄是否"已连接且当前活跃"
function Input:ControllerAttached()
    if self.controllerid_cached ~= nil then
        return self.controllerid_cached > 0
    end
    -- 控制台平台直接返回 true（它们只有手柄）
    -- PC 平台检查是否有手柄处于活跃状态
    return IsConsole() or TheInputProxy:IsAnyControllerActive()
end

-- 手柄是否物理插入（不关心是否活跃）
function Input:ControllerConnected()
    return IsConsole() or TheInputProxy:IsAnyControllerConnected()
end
```

两者的区别：

| API | 含义 | 适用场景 |
|-----|------|----------|
| `ControllerAttached()` | **连接且当前使用中**（最近有输入） | "当前输入模式是手柄吗" → 决定 UI 显示逻辑 |
| `ControllerConnected()` | **物理连接**（可能没在用） | 初始化时检查是否有手柄可用 |

**在几乎所有 mod 场景下应使用 `ControllerAttached()`** 而非 `ControllerConnected()`——插着手柄但玩家用鼠标操作时，`ControllerConnected()` 为 true 但 `ControllerAttached()` 为 false，如果此时用手柄模式的 UI 逻辑会造成混乱。

#### ControllerID 缓存机制

`scripts/input.lua:77-88`：

```lua
function Input:CacheController()
    self.controllerid_cached = IsNotConsole() and
        (TheInputProxy:GetLastActiveControllerIndex() or 0) or nil
    return self.controllerid_cached
end

function Input:GetControllerID()
    return self.controllerid_cached or TheInputProxy:GetLastActiveControllerIndex() or 0
end
```

`controllerid_cached` 含义：
- `nil`：尚未缓存（控制台平台不缓存，始终用 nil 表示"只有手柄"）
- `0`：键盘鼠标（index 0）
- `> 0`：具体的手柄设备 index

`ControllerAttached()` 会优先用缓存值判断——`controllerid_cached > 0` 就是手柄。

#### ToggleController——动态切换控制器模式

当玩家从手柄切换到键鼠（或反之）时，`PlayerController:ToggleController(val)` 被调用（`playercontroller.lua:395-406`）：

```lua
function PlayerController:ToggleController(val)
    if self.isclientcontrollerattached ~= val then
        self.isclientcontrollerattached = val   -- 更新本地缓存
        if self.handler ~= nil then
            self:RefreshReticule()  -- 刷新准星（手柄显示方形准星，键鼠显示鼠标光标）
        end
        if not self.ismastersim then
            SendRPCToServer(RPC.ToggleController, val)  -- 通知服务端
        elseif val and self.inst.components.inventory ~= nil then
            self.inst.components.inventory:ReturnActiveItem()  -- 手柄模式收回手持物
        end
    end
end
```

`OnContinueFromPause` 会在游戏恢复时自动调用 `ToggleController(TheInput:ControllerAttached())`（`playercontroller.lua:300`），确保控制器状态同步。

**mod 中监听手柄切换**：

```lua
-- 在玩家 prefab 的 client 端添加事件监听（伪代码）
-- 目前官方没有直接暴露"控制器模式变化"的事件
-- 推荐的做法是在 Update 里每帧调用 TheInput:ControllerAttached() 并对比上一帧
local last_controller_attached = TheInput:ControllerAttached()

local function CheckControllerChange()
    local current = TheInput:ControllerAttached()
    if current ~= last_controller_attached then
        last_controller_attached = current
        -- 手柄/键鼠模式已切换，刷新 UI
        UpdateMyModUI(current)
    end
end

-- 在 mod 的 Update 里调用
```

> **进阶记忆**：`ControllerAttached()` = "玩家正在用手柄"；`ControllerConnected()` = "插了手柄但可能没在用"。`controllerid_cached > 0` = 手柄，`== 0` = 键鼠，`== nil` = 控制台（只有手柄）。

---

### 17.4.3 进阶：CONTROL 体系对手柄的统一映射

#### 为什么 CONTROL_* 可以统一适配

饥荒的输入管理架构是：

```
[物理设备] 键盘/鼠标/手柄
     ↓
[C++ 输入层] 将按键映射到 CONTROL_* 数字 ID
     ↓
[Lua 层] TheInput:IsControlPressed(CONTROL_ACTION) 等
```

用户可以在"按键设置"里为每个 `CONTROL_*` 重新绑定按键。`CONTROL_ACTION = 4` 默认绑定到键盘 Space，也可能被玩家改成别的键——Lua 层完全不关心这个映射，只查 `CONTROL_ACTION` 是否被激活。

手柄按键默认映射（Xbox 手柄，来自 `constants.lua:108-178` 注释）：

| CONTROL_* | Xbox 默认按键 | PC 键盘/鼠标默认 |
|------------|--------------|-----------------|
| `CONTROL_PRIMARY` | — | 鼠标左键 |
| `CONTROL_SECONDARY` | — | 鼠标右键 |
| `CONTROL_ATTACK` | — | Ctrl+左键 |
| `CONTROL_ACTION` | A 键 | Space |
| `CONTROL_MOVE_UP/DOWN/LEFT/RIGHT` | 左摇杆方向 | WASD |
| `CONTROL_ZOOM_IN` | 左扳机 LT | 鼠标滚轮上 |
| `CONTROL_ZOOM_OUT` | 右扳机 RT | 鼠标滚轮下 |
| `CONTROL_ROTATE_LEFT` | 左肩键 LB | Q |
| `CONTROL_ROTATE_RIGHT` | 右肩键 RB | E |
| `CONTROL_FOCUS_UP/DOWN/LEFT/RIGHT` | 十字键 D-Pad | 方向键 |
| `CONTROL_ACCEPT` | A | Enter |
| `CONTROL_CANCEL` | B | Escape |
| `CONTROL_CONTROLLER_ATTACK` | X | — |
| `CONTROL_CONTROLLER_ACTION` | A | — |
| `CONTROL_CONTROLLER_ALTACTION` | B | — |

注意 `CONTROL_CONTROLLER_ATTACK`、`CONTROL_CONTROLLER_ACTION`、`CONTROL_CONTROLLER_ALTACTION` 是**手柄专用控制**，在键鼠模式下不会被触发（它们没有默认的键盘映射）。

#### 模拟量控制——GetAnalogControlValue

`scripts/input.lua:484-487`：

```lua
function Input:GetAnalogControlValue(control)
    control = self:ResolveVirtualControls(control)
    return control and TheSim:GetAnalogControl(control) or 0
end
```

返回 0.0 到 1.0 的浮点数。适用于手柄摇杆、扳机等需要程度感应的控制：

```lua
-- 读取移动方向（摇杆偏转量）
local move_up = TheInput:GetAnalogControlValue(CONTROL_MOVE_UP)   -- 0 ~ 1
local move_right = TheInput:GetAnalogControlValue(CONTROL_MOVE_RIGHT)

-- 读取镜头缩放（扳机按压量）
local zoom = TheInput:GetAnalogControlValue(CONTROL_ZOOM_IN)  -- LT 按压程度
```

键盘对应的 CONTROL 的模拟量只有 0 或 1（按下/松开），而手柄摇杆可以是 0.0~1.0 的任意值。

> **进阶记忆**：优先使用 `CONTROL_*` 而非 `KEY_*`——这样手柄用户不需要额外处理。摇杆/扳机读模拟量用 `GetAnalogControlValue`。**手柄专用控制**（`CONTROL_CONTROLLER_*`）在键鼠上不生效，需要额外判断。

---

### 17.4.4 进阶：虚拟控制（VIRTUAL_CONTROL_*）与控制方案

#### 为什么需要虚拟控制

问题背景：手柄上只有有限的按键（摇杆、扳机、肩键、面键、方向键），但饥荒有大量操作（移动、镜头、背包、制作菜单...）都需要映射到手柄上。

饥荒的解决方案是**控制方案（Control Scheme）**——通过 7 档不同的方案，让同一个物理按键（如右摇杆）在不同方案下触发不同的 `CONTROL_*`。

`VIRTUAL_CONTROL_*` 常量（`scripts/constants.lua:245-272`）是"尚未解析到具体物理按键"的抽象控制 ID，如：

```lua
VIRTUAL_CONTROL_CAMERA_ZOOM_IN = 10001   -- 抽象镜头缩放入
VIRTUAL_CONTROL_INV_UP = 10009           -- 抽象背包向上
VIRTUAL_CONTROL_AIM_UP = 10005          -- 抽象瞄准向上
```

这些虚拟控制通过 `ResolveVirtualControls` 解析成实际的 `CONTROL_PRESET_RSTICK_*` 或 `CONTROL_PRESET_DPAD_*`（`input.lua:339-477`），具体解析结果取决于当前控制方案。

#### GetActiveControlScheme 和 7 档方案

`scripts/input.lua:489-493`：

```lua
function Input:GetActiveControlScheme(schemeId)
    -- 只有手柄才有控制方案，键鼠始终返回方案 1（经典模式）
    return self:ControllerAttached() and Profile:GetControlScheme(schemeId) or 1
end
```

控制方案 1-7 的含义（由用户在"游戏设置"里选择）：

| 方案编号 | 特点 |
|---------|------|
| 1 | 经典模式（方向键控制物品选取，不支持双摇杆） |
| 2-3 | 右摇杆控制背包/镜头（双摇杆模式的基础方案） |
| 4-7 | 支持双摇杆自由瞄准（`SupportsControllerFreeAiming()`） |

快速判断特性：

```lua
-- 是否支持右摇杆自由相机（方案 2-7）
TheInput:SupportsControllerFreeCamera()   -- input.lua:500-503

-- 是否支持双摇杆自由瞄准（方案 4-7）
TheInput:SupportsControllerFreeAiming()   -- input.lua:495-498
```

**mod 中通常不需要关心控制方案细节**——只要用 `CONTROL_*`（非 VIRTUAL_CONTROL），`ResolveVirtualControls` 会自动处理映射。需要关注方案的场景是：你的 mod 要自定义右摇杆/方向键的用途，此时需要先检查当前方案再决定是否接管这些输入。

#### ResolveVirtualControls 的作用

`scripts/input.lua:339`：

```lua
function Input:ResolveVirtualControls(control)
    if control == nil then
        return
    elseif control < VIRTUAL_CONTROL_START then
        -- 普通 CONTROL_*，直接返回
        ...
        return control
    end
    -- VIRTUAL_CONTROL_*，根据当前控制方案解析成实际控制 ID
    local scheme = self:GetActiveControlScheme(CONTROL_SCHEME_CAM_AND_INV)
    ...
end
```

`IsControlPressed` 和 `GetAnalogControlValue` 内部都先调用 `ResolveVirtualControls`（`input.lua:480`、`485`）——对 mod 开发者透明，不需要手动调用。

> **进阶记忆**：`VIRTUAL_CONTROL_*` 是"还没映射到物理按键"的抽象 ID，`ResolveVirtualControls` 根据当前方案解析它们。mod 开发中直接用 `CONTROL_*` 即可，不需要关心 VIRTUAL_CONTROL 细节。

---

### 17.4.5 老手：GetLocalizedControl——动态显示"按哪个键"

#### API 说明

`scripts/input.lua:593-615`：

```lua
function Input:GetLocalizedControl(deviceId, controlId, use_default_mapping, use_control_mapper)
    if controlId >= VIRTUAL_CONTROL_START then
        return self:GetLocalizedVirtualControl(...)
    end
    local device, numInputs, input1, input2, input3, input4, intParam =
        TheInputProxy:GetLocalizedControl(deviceId, controlId, ...)
    -- 拼接多个按键显示（如 "Ctrl + A"）
    local inputs = { input1, input2, input3, input4 }
    local text = STRINGS.UI.CONTROLSSCREEN.INPUTS[device][input1]
    for idx = 2, numInputs do
        text = text.." + "..STRINGS.UI.CONTROLSSCREEN.INPUTS[device][inputs[idx]]
    end
    return intParam ~= nil and string.format(text, intParam) or text
end
```

参数说明：

| 参数 | 说明 |
|------|------|
| `deviceId` | 输入设备 ID，通常用 `TheInput:GetControllerID()` 获取当前活跃设备 |
| `controlId` | `CONTROL_*` 常量 |
| `use_default_mapping` | 是否显示默认映射（忽略玩家自定义绑定），通常 `false` |
| `use_control_mapper` | 是否使用 ControlMapper，通常省略（默认 true） |

返回值是一个**本地化字符串**，如：
- 键鼠模式：`"Space"` / `"鼠标右键"` / `"Ctrl + 鼠标左键"`
- 手柄模式：`"A"` / `"X"` / `"LT"` 等（根据当前手柄设备的按键名）

#### 在 UI 提示中使用 GetLocalizedControl

来自 `scripts/widgets/controls.lua:612-613` 的真实用法：

```lua
local controller_mode = TheInput:ControllerAttached()
local controller_id = TheInput:GetControllerID()

-- 手柄模式下显示手柄按键提示
if controller_mode then
    local key_str = TheInput:GetLocalizedControl(controller_id, CONTROL_CONTROLLER_ACTION)
    -- key_str 可能是 "A"（Xbox）或 "Cross"（PlayStation）
    print("按", key_str, "交互")
else
    local key_str = TheInput:GetLocalizedControl(controller_id, CONTROL_ACTION)
    -- key_str 可能是 "Space" 或玩家自定义的其他键
    print("按", key_str, "交互")
end
```

#### 完整的按键提示显示函数

```lua
-- 参考用法（mod 中）
-- 获取"执行交互"操作的当前按键名（自动适配键鼠/手柄）
local function GetActionKeyHint()
    local controller_id = TheInput:GetControllerID()
    if TheInput:ControllerAttached() then
        -- 手柄：用 CONTROL_CONTROLLER_ACTION (A 键)
        return TheInput:GetLocalizedControl(controller_id, CONTROL_CONTROLLER_ACTION)
    else
        -- 键鼠：用 CONTROL_ACTION (默认 Space)
        return TheInput:GetLocalizedControl(controller_id, CONTROL_ACTION)
    end
end

-- 在 Widget 的文字提示里动态更新
local hint = "[" .. GetActionKeyHint() .. "] 使用技能"
```

> **老手记忆**：`GetLocalizedControl(TheInput:GetControllerID(), CONTROL_ACTION)` = 获取"交互键"的当前按键名（自动处理键盘重绑定和手柄型号差异）。配合 `ControllerAttached()` 判断选用 `CONTROL_ACTION` 还是 `CONTROL_CONTROLLER_ACTION`，让 UI 提示对所有玩家准确显示。

---

### 17.4.6 老手：mod 手柄适配完整实践

下面是一个同时支持键盘和手柄的 mod 技能系统模板，包含：
- 键盘用 `KEY_F`，手柄用 `CONTROL_CONTROLLER_ACTION`（A 键）
- UI 提示动态显示当前绑定
- 平台安全的输入注册

```lua
-- modmain.lua

-- 注册服务端技能 RPC
AddModRPCHandler("MY_MOD", "USE_SKILL", function(player)
    if player and player:IsValid() and player.components.health then
        player.components.health:DoDelta(30)
        print(player.name, "使用了技能，回复 30 HP")
    end
end)

-- 安全的技能触发函数（从客户端调用）
local function TriggerSkill()
    SendModRPCToServer(GetModRPC("MY_MOD", "USE_SKILL"))
end

-- 获取当前设备对应的技能键提示文字
local function GetSkillKeyHint()
    if IsConsole() or TheInput:ControllerAttached() then
        -- 手柄模式：显示 A 键
        local key = TheInput:GetLocalizedControl(TheInput:GetControllerID(), CONTROL_CONTROLLER_ACTION)
        return "[" .. (key or "A") .. "]"
    else
        -- 键鼠模式：显示 F 键
        return "[F]"
    end
end

AddSimPostInit(function()
    if IsConsole() then
        -- 控制台平台：只注册手柄控制
        TheInput:AddControlHandler(CONTROL_CONTROLLER_ACTION, function(down)
            if down then
                TriggerSkill()
            end
        end)
    elseif IsNotConsole() then
        -- PC 平台：同时注册键盘和手柄

        -- 键盘 F 键
        TheInput:AddKeyDownHandler(KEY_F, function()
            TriggerSkill()
        end)

        -- 手柄 A 键（CONTROL_CONTROLLER_ACTION）
        -- 注意：CONTROL_CONTROLLER_ACTION 只在手柄模式下触发，
        -- 不会与键盘冲突，可以直接注册
        TheInput:AddControlHandler(CONTROL_CONTROLLER_ACTION, function(down)
            if down and TheInput:ControllerAttached() then
                TriggerSkill()
            end
        end)
    end
end)

-- 在 HUD Widget 中更新技能按键提示（伪代码）
-- local hint_text = GetSkillKeyHint() .. " 使用技能"
-- my_skill_widget:SetText(hint_text)
```

**代码解析**：

1. `IsConsole()` 分支：控制台平台没有键盘，只注册 `CONTROL_CONTROLLER_ACTION`
2. `IsNotConsole()` 分支：PC 平台同时注册键盘和手柄——键盘用 `AddKeyDownHandler(KEY_F)`，手柄用 `AddControlHandler(CONTROL_CONTROLLER_ACTION)`
3. 手柄分支里额外检查 `TheInput:ControllerAttached()`——确保只在手柄模式下触发（否则用键盘的 PC 玩家可能因为其他原因触发 `CONTROL_CONTROLLER_ACTION`）
4. `GetSkillKeyHint()` 动态生成"按×键使用技能"的提示文字，无论玩家使用什么设备都显示正确

> **老手记忆**：控制台平台只有手柄，PC 平台要同时兼容键盘和手柄；`CONTROL_CONTROLLER_ACTION` 只在手柄模式下有效；`GetLocalizedControl` 让你的提示文字自动适配键盘重绑定和不同型号的手柄。


## 17.5 PlayerController 的输入分发流程

### 本节导读

17.5 是第 17 章的"压轴"——揭示饥荒输入系统的完整管道，解释"一次按键是如何从物理信号变成游戏行为"的全过程。理解这条流水线，你才能真正读懂 `PlayerController` 的源码，知道为什么某些场景下输入会被屏蔽，以及如何在正确位置"插入"你的 mod 逻辑。

阅读路径：

> **新手**看 17.5.1-17.5.2——理解 PlayerController 在整个输入链路中处于什么位置，了解最粗线条的分发流程；**进阶读者**继续看 17.5.3-17.5.5——深入 `OnControl`、`IsEnabled`、`OnUpdate` 三大核心函数，搞清楚"输入怎么被过滤、怎么被分发、怎么被持续处理"；**老手**看 17.5.6-17.5.7——理解 `MOD_CONTROLS` 编解码机制和"输入抢占"原理，掌握如何在 mod 中安全地在 PlayerController 之前拦截输入。

读完本节，你能：

- 画出一次"按鼠标左键"从 C++ 到游戏行为的完整调用链
- 知道哪些情况下 PlayerController 会屏蔽输入（忙碌、HUD 遮挡、禁用）
- 理解 `IsEnabled` 的双返回值设计和 `ishudblocking` 的用途
- 知道 mod 中如何合理地"提前消耗"输入事件，避免与 PlayerController 冲突

---

### 17.5.1 新手：PlayerController 在输入链路中的位置

#### 整体分工

饥荒的输入系统是一条有层级的**分发链**，每一层都可以"消耗"一个输入事件（消耗后不再向下传递）：

```
[C++ 引擎] 硬件输入信号
    ↓ 调用全局函数
[Lua 全局函数] OnControl / OnInputKey / OnMouseButton
    ↓
[TheInput] 分发给注册的回调
    ↓  先交给
[TheFrontEnd] UI 层（Screens / Widgets）— 如果 UI 消耗，PlayerController 不收到
    ↓  如果 UI 没消耗
[PlayerController.OnControl] 游戏逻辑层——处理攻击/移动/交互等游戏行为
    ↓  如果有 mod 注册了 AddGeneralControlHandler / AddControlHandler
[Mod 回调] 与 PlayerController 同层
```

关键规则：**TheFrontEnd 优先**——如果有 UI 窗口（制作菜单、背包界面）正在响应输入，PlayerController 的输入就会被屏蔽或限制。

#### PlayerController 组件的职责

`PlayerController` 是 `ThePlayer`（本地玩家 entity）上的一个组件，只存在于客户端（包括单人游戏的客户端=服务端的情况）。它负责：

1. 把客户端的用户输入翻译成"意图"（如"想去点击的位置走"、"想攻击鼠标下的敌人"）
2. 在客户端做预测（locomotor 预测走路）
3. 通过 RPC 通知服务端（`SendRPCToServer(RPC.LeftClick, ...)`）

#### Activate 时注册通用输入监听

`scripts/components/playercontroller.lua:313-319`：

```lua
function PlayerController:Activate()
    ...
    -- 使用 AddGeneralControlHandler 监听所有 CONTROL_* 事件
    self.handler = TheInput:AddGeneralControlHandler(
        function(control, value) self:OnControl(control, value) end
    )
    ...
end
```

当玩家 entity 成为 `ThePlayer` 时，`Activate()` 被调用，注册一个**通用控制处理器**（`AddGeneralControlHandler`），接收所有 `CONTROL_*` 事件。

这个 `self.handler` 是 `EventHandler` 对象，在 `Deactivate()` 时调用 `self.handler:Remove()` 注销（`playercontroller.lua:364`）。

> **新手记忆**：PlayerController 坐在 TheFrontEnd 下方、mod AddControlHandler 回调同层。它在 Activate 时用 `AddGeneralControlHandler` 抓住所有 `CONTROL_*`，在 `OnControl` 里决定如何响应。

---

### 17.5.2 新手/进阶：完整输入流水线——从按键到游戏行为

以**鼠标左键点击场景**为例，追踪完整调用链：

```
[C++] 玩家按下鼠标左键
    ↓
[全局函数] OnMouseButton(MOUSEBUTTON_LEFT, true, x, y)
    ↓ (scripts/input.lua:756-758)
TheInput:OnMouseButton(MOUSEBUTTON_LEFT, true, x, y)
    ↓ (input.lua:179-184) -- 如果 mouse_enabled
    ├─→ TheFrontEnd:OnMouseButton(...)  ← UI 优先处理
    │       如果 UI 消耗（如点击了按钮）→ 停止
    │       如果 UI 没消耗 →
    └─→ TheInput.onmousebutton:HandleEvent(...)  ← 触发 AddMouseButtonHandler 注册的回调

[并行分支] 左键同时映射到 CONTROL_PRIMARY (id=0)
    ↓
[C++] SetDigitalControlFromInput → 设置 CONTROL_PRIMARY 为激活
    ↓
[全局函数] OnControl(CONTROL_PRIMARY, true, ...)
    ↓ (input.lua:751-753)
TheInput:OnControl(CONTROL_PRIMARY, true, ...)
    ↓ (input.lua:163-170)
    if mouse_enabled or control != PRIMARY/SECONDARY:
        先给 TheFrontEnd:OnControl(CONTROL_PRIMARY, true)
            如果 UI 消耗 → 停止
            如果 UI 没消耗 →
        oncontrol:HandleEvent(CONTROL_PRIMARY, true)  ← 触发 AddControlHandler 注册的回调
        oncontrol:HandleEvent("oncontrol", CONTROL_PRIMARY, true)  ← 触发 AddGeneralControlHandler 回调
            ↓
            PlayerController:OnControl(CONTROL_PRIMARY, true)
                ↓
                self:OnLeftClick(true)
                    ↓ 分析鼠标下的 entity 和可执行动作
                    ↓ 发送 RPC 到服务端
                    ↓ 启动客户端预测
```

关键路径：`鼠标左键 → CONTROL_PRIMARY → TheFrontEnd → PlayerController.OnControl → OnLeftClick`

**键盘路径更简单**（以 Space → CONTROL_ACTION 为例）：

```
[C++] 按下 Space → OnControl(CONTROL_ACTION, true)
    → TheInput:OnControl → TheFrontEnd:OnControl（通常不消耗）
    → PlayerController:OnControl → self:DoActionButton()
```

> **进阶记忆**：鼠标事件走两条路（OnMouseButton + CONTROL_PRIMARY），键盘事件只走 CONTROL 路。每条路都先经过 TheFrontEnd UI 层，UI 不消耗才到游戏逻辑层。

---

### 17.5.3 进阶：OnControl 分发流程详解

`PlayerController:OnControl(control, down)` 是输入的核心入口，完整逻辑（`playercontroller.lua:601-718`）：

**第一关：基础过滤**

```lua
function PlayerController:OnControl(control, down)
    -- 1. 游戏暂停 → 完全屏蔽
    if IsPaused() then
        return
    end

    -- 2. 控制器是否可用（IsEnabled 返回两个值）
    local isenabled, ishudblocking = self:IsEnabled()
    if not isenabled and not ishudblocking then
        return
    end
    ...
```

**第二关：临时忽略机制（_hack_ignore）**

```lua
    -- 3. 特殊的"忽略本次按下"黑科技（处理某些动作结束后的额外输入问题）
    if down and self._hack_ignore_held_controls then
        self._hack_ignore_ups_for[control] = true
        return true  -- 消耗事件
    end
    ...
```

**第三关：松开事件（control up）提前返回**

```lua
    -- 4. 大多数控制的松开（非 PRIMARY/SECONDARY）→ 发 RemoteStopControl 后立即返回
    if not down and control ~= CONTROL_PRIMARY and control ~= CONTROL_SECONDARY then
        if not self.ismastersim then
            self:RemoteStopControl(control)  -- 通知服务端"这个控制松开了"
        end
        return
    end
```

**第四关：允许在 HUD 遮挡下执行的操作**

```lua
    -- 5. 即使制作菜单打开，这些操作仍然响应
    if isenabled or ishudblocking then
        if control == CONTROL_ACTION then
            self:DoActionButton()
            return
        elseif control == CONTROL_ATTACK then
            self.attack_buffer = CONTROL_ATTACK  -- 或 DoAttackButton
            return
        elseif control == CONTROL_CHARACTER_COMMAND_WHEEL then
            self:DoCharacterCommandWheelButton()
            return
        end
    end

    -- 6. 完全禁用时（如 HUD 遮挡所有其他操作）→ 返回
    if not isenabled then
        return
    end
```

**第五关：主要控制分发**

```lua
    -- 7. 主要控制路由
    if control == CONTROL_PRIMARY then
        self:OnLeftClick(down)          -- 左键行为
    elseif control == CONTROL_SECONDARY then
        self:OnRightClick(down)         -- 右键行为
    elseif control == CONTROL_CANCEL then
        self:CancelPlacement()
    elseif control == CONTROL_INSPECT then
        self:DoInspectButton()
    elseif control == CONTROL_CONTROLLER_ALTACTION then
        self:DoControllerAltActionButton()
    elseif control == CONTROL_CONTROLLER_ACTION then
        self:DoControllerActionButton()
    elseif control == CONTROL_CONTROLLER_ATTACK then
        ...
    elseif self.inst.replica.inventory:IsVisible() then
        -- 背包可见时，处理背包内的操作（CONTROL_INVENTORY_*）
        ...
    end
end
```

**分发规律**：

| 控制 | 处理方式 |
|------|---------|
| `CONTROL_ACTION` | 所有情况下响应（含 HUD 遮挡） |
| `CONTROL_ATTACK` | 所有情况下响应 |
| `CONTROL_PRIMARY/SECONDARY` | 完整处理（包括 up），有拖拽/双击逻辑 |
| `CONTROL_CANCEL` | 取消放置 |
| 其他 down | 只有 `isenabled == true` 时才处理 |
| 其他 up（非 PRIMARY/SECONDARY） | 第三关提前处理（`RemoteStopControl`） |

> **进阶记忆**：`CONTROL_ACTION` 和 `CONTROL_ATTACK` 是"穿透 HUD"的特殊操作——即使制作菜单打开也能攻击和交互；其他大部分操作在 HUD 遮挡时被屏蔽。

---

### 17.5.4 进阶：IsEnabled——两级禁用状态

`scripts/components/playercontroller.lua:491-502`：

```lua
-- 返回两个值：(是否完全启用, 是否仅HUD遮挡但允许有限游戏)
function PlayerController:IsEnabled()
    if self.classified == nil or not self.classified.iscontrollerenabled:value() then
        return false  -- 完全禁用（死亡、cinematic 等）
    elseif self.inst.HUD ~= nil and self.inst.HUD:HasInputFocus() then
        return false,       -- 不完全启用
            (self.inst.HUD:IsCraftingOpen() and TheFrontEnd.textProcessorWidget == nil) or
            self.inst.HUD:IsSpellWheelOpen() or
            self.inst.HUD:IsUpgradeModuleWidgetInputFocus() or
            (self.command_wheel_allows_gameplay and self.inst.HUD:IsCommandWheelOpen())
        -- ishudblocking = true 意味着：HUD 有输入焦点，但允许有限游戏操作
    end
    return true  -- 完全启用
end
```

三种状态：

| 返回值 | 含义 | 效果 |
|-------|------|------|
| `true, nil` | 完全启用 | 所有操作正常响应 |
| `false, true` | HUD 遮挡但允许有限操作 | 只有 `CONTROL_ACTION`、`CONTROL_ATTACK` 等穿透操作生效 |
| `false, false/nil` | 完全禁用 | 所有操作都不响应 |

`iscontrollerenabled` 是一个 `net_bool`——服务端通过 `PlayerController:Enable(false)` 设置，网络同步给客户端，用于在对话（Talker 对话框）、过场动画等场景完全禁用输入。

#### OnUpdate 里的禁用检查

`playercontroller.lua:2644-2748` 中的 `OnUpdate` 每帧也检查 `IsEnabled()`，在禁用时做清理：

```lua
function PlayerController:OnUpdate(dt)
    local isenabled, ishudblocking = self:IsEnabled()

    -- 禁用时：停止拖拽走路、取消放置、停止所有远程控制
    if not isenabled then
        ...
        if not self.ismastersim then
            self:RemoteStopAllControls()  -- 通知服务端所有控制都松开了
        end
    end
    ...
end
```

> **进阶记忆**：`IsEnabled()` 有三态：完全启用、HUD 遮挡、完全禁用。制作菜单打开时是"HUD 遮挡"，会屏蔽大多数操作但保留攻击和交互键。`Enable(false)` 完全禁用所有操作（用于对话、过场）。

---

### 17.5.5 进阶：OnUpdate 中的持续轮询处理

`OnControl` 只处理**瞬间事件**（按下/松开）。许多游戏行为需要**持续状态检测**，这是 `OnUpdate(dt)` 的职责。

`playercontroller.lua:2644` 的 `OnUpdate` 主要处理：

1. **拖拽走路检测**：每帧检查 `CONTROL_PRIMARY` 是否按住超过 `START_DRAG_TIME`，确认则进入 `draggingonground = true` 状态（见 17.3.5）
2. **相机控制**：每帧检查 `CONTROL_ROTATE_LEFT/RIGHT`（旋转）和 `CONTROL_ZOOM_IN/OUT`（缩放），轮询触发（`playercontroller.lua:4452-4474`）：

```lua
-- playercontroller.lua:4453-4458（旋转轮询）
if TheInput:IsControlPressed(invert_rotation and CONTROL_ROTATE_RIGHT or CONTROL_ROTATE_LEFT) then
    self:RotLeft()
    self.lastrottime = time
elseif TheInput:IsControlPressed(CONTROL_ROTATE_RIGHT ...) then
    self:RotRight()
end
```

3. **预测走路**：每帧根据 WASD/摇杆方向更新 locomotor 的移动预测

4. **目标高亮刷新**：每帧更新 `controller_target`（手柄模式下高亮的攻击目标）

**轮询 vs 回调**的分工：

| 输入类型 | 处理位置 |
|---------|---------|
| 单次触发（攻击、交互、放置） | `OnControl` 事件回调 |
| 持续状态（移动、相机旋转、拖拽） | `OnUpdate` 轮询 |

> **进阶记忆**：`OnControl` 处理"按下/松开瞬间"的单次事件；`OnUpdate` 处理"按住期间"的持续状态。两者配合完整覆盖所有输入模式。

---

### 17.5.6 老手：MOD_CONTROLS 编解码——修饰键的网络传输

#### 什么是 MOD_CONTROLS

`scripts/components/playercontroller.lua:722-728`：

```lua
local MOD_CONTROLS =
{
    CONTROL_FORCE_INSPECT,    -- 38
    CONTROL_FORCE_ATTACK,     -- 39
    CONTROL_FORCE_TRADE,      -- 40
    CONTROL_FORCE_STACK,      -- 41
}
```

这四个控制对应的是"修饰操作"（如按住 Ctrl 强制攻击）。当玩家在客户端发出行动（如右键点击某个 entity 时按住了 Ctrl），需要把"Ctrl 按着呢"这个信息传给服务端——但传递完整的 4 个 bool 太浪费，游戏用了一个位编码优化。

#### 编码和解码

`playercontroller.lua:730-758`：

```lua
function PlayerController:EncodeControlMods()
    local code = 0
    local bit = 1
    for i, v in ipairs(MOD_CONTROLS) do
        -- 每个修饰键占一位：bit 1, 2, 4, 8
        code = code + (TheInput:IsControlPressed(v) and bit or 0)
        bit = bit * 2
    end
    return code ~= 0 and code or nil  -- 全 0 返回 nil（无修饰键）
end

function PlayerController:DecodeControlMods(code)
    code = code or 0
    local bit = 2 ^ (#MOD_CONTROLS - 1)
    for i = #MOD_CONTROLS, 1, -1 do
        if code >= bit then
            self.remote_controls[MOD_CONTROLS[i]] = 0  -- 服务端设置为"按下"
            code = code - bit
        else
            self.remote_controls[MOD_CONTROLS[i]] = nil  -- 服务端设置为"未按"
        end
        bit = bit / 2
    end
end
```

`EncodeControlMods()` 返回一个 0-15 的整数（4 位二进制），随 RPC 发给服务端。服务端调用 `DecodeControlMods(code)` 解码，在 `remote_controls` 里模拟出客户端的修饰键状态。

这样服务端的 `PlayerController:IsControlPressed(CONTROL_FORCE_ATTACK)` 就能正确返回 `true`（从 `remote_controls` 读取），即使服务端本身没有直接的键盘输入。

#### 对 mod 的启示

如果你的 mod 需要让服务端知道"玩家发出这个行动时是否按着某个键"，有两种方式：

1. **通过 RPC 额外参数传递**（推荐）：在 `SendModRPCToServer` 时附带一个 bool 参数，让服务端直接知道状态（无需编解码）

2. **注册到 MOD_CONTROLS**（不推荐修改）：`MOD_CONTROLS` 是内部数组，不建议 mod 直接追加

```lua
-- 推荐：直接通过 RPC 参数传修饰键状态
local force_attack = TheInput:IsControlPressed(CONTROL_FORCE_ATTACK)
SendModRPCToServer(GetModRPC("MY_MOD", "SKILL"), force_attack)

-- 服务端 handler：
AddModRPCHandler("MY_MOD", "SKILL", function(player, is_force)
    if is_force then
        -- 玩家按着 Ctrl 触发了技能
    end
end)
```

---

### 17.5.7 老手：在 mod 中"插入"到分发链路

#### 理解分发链的优先级

`input.lua:163-170` 的 `OnControl` 展示了精确的优先级：

```lua
function Input:OnControl(control, digitalvalue, analogvalue)
    if (self.mouse_enabled or
        (control ~= CONTROL_PRIMARY and control ~= CONTROL_SECONDARY)) and
        not TheFrontEnd:OnControl(control, digitalvalue) then   -- ← TheFrontEnd 优先
        self.oncontrol:HandleEvent(control, digitalvalue, analogvalue)
        self.oncontrol:HandleEvent("oncontrol", control, digitalvalue, analogvalue)
    end
end
```

关键：`not TheFrontEnd:OnControl(...)` ——如果 TheFrontEnd 返回 `true`（表示它消耗了这个输入），`oncontrol:HandleEvent` 就**不会被调用**，PlayerController 和你的 `AddControlHandler` 都收不到这个事件。

这意味着：**如果想在 PlayerController 之前拦截输入，必须用 Widget 系统（TheFrontEnd 层）而不是 AddControlHandler**——因为 `AddControlHandler` 和 PlayerController 的 `AddGeneralControlHandler` 是同一层，执行顺序不确定。

#### 方式一：Widget 拦截（在 PlayerController 之前）

当你的 mod 需要全屏 UI 或者需要"吃掉"所有输入时：

```lua
-- 参考用法（mod 的 Screen 类）
-- Screen 继承自 Widget，注册到 TheFrontEnd 的 screen 栈后自动接收 OnControl
function MyFullScreenWidget:OnControl(control, down)
    if control == CONTROL_CANCEL and down then
        self:Close()
        return true   -- 返回 true = 消耗事件，PlayerController 收不到
    end
    -- 不 return true = 事件继续向下传递给 PlayerController
end
```

#### 方式二：AddControlHandler（与 PlayerController 同层）

当你的操作**不冲突**（如新增一个自定义操作，PlayerController 不处理的控制），使用 `AddControlHandler`：

```lua
-- 监听一个 PlayerController 不处理的控制
TheInput:AddControlHandler(CONTROL_INSPECT_SELF, function(down)
    if down and ThePlayer then
        -- CONTROL_INSPECT_SELF 对应默认 [I] 键，PlayerController 不特殊处理
        -- 这里可以安全地执行
    end
end)
```

#### 方式三：在 PlayerController 的 Action 层"注入"

`OnControl` → `DoActionButton` → `GetMouseAction` → `DoAction` 这个链路的某些位置是可以通过 mod 的 `ACTIONS` 机制注入的（详见第 X 章 Action 系统）。这是最"正规"的方式——通过添加自定义 Action，让 PlayerController 的标准流程自然处理它，而不是绕过 PlayerController。

#### 实践原则

| 场景 | 推荐方式 |
|------|---------|
| 全屏 UI（需要屏蔽所有游戏输入） | Screen 继承 Widget，注册到 TheFrontEnd |
| 新增自定义操作（不与现有冲突） | `AddControlHandler(未使用的 CONTROL_*)` |
| 技能触发（需要发 RPC 给服务端） | `AddKeyDownHandler` + `SendModRPCToServer`（17.2） |
| 追加"鼠标下目标"的行为 | 自定义 `ACTIONS`（Action 系统，非本章） |

> **老手记忆**：TheFrontEnd 在 PlayerController 之前——Widget 的 `OnControl` 返回 `true` 就能"吃掉"输入阻止 PlayerController 收到。`AddControlHandler` 与 PlayerController 同层，无法可靠地优先于它。正规的游戏行为注入走 Action 系统。

