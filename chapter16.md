# 第16章 UI 与界面系统

## 16.1 Widget 基础：Text、Image、ImageButton、AnimButton

> **写在前面**——前面 15 章我们一直在做"游戏世界"的事——AI、动画、地形、生物。本章开始转向"**屏幕上的事**"——血条、菜单、弹窗、合成栏、设置面板……所有玩家在 HUD 看到的、点的、读的内容，都属于 **UI 系统**。
>
> 饥荒的 UI 系统**全部建立在一套叫 `Widget` 的基类上**——文本是 `Text`、图片是 `Image`、按钮是 `ImageButton` / `AnimButton` / `Button`、整个屏幕是 `Screen`、HUD 也是大型 `Screen`。掌握 `Widget` 就等于掌握了所有 UI 的"原子"。
>
> **本节是整章的地基**——所有后续 16.2-16.10 都依赖你对 Widget 体系的理解。
>
> **新手**先看 16.1.1-16.1.3——理解 `Widget` 是所有 UI 的基类、用 5 行代码显示一行文字、用 5 行代码显示一张图片；**进阶读者**继续看 16.1.4-16.1.6，深入按钮的"五态"模型（normal/focus/down/disabled/selected）、`Button` 基类的回调系统（onclick/ondown/whiledown）、`AnimButton` 用 Spriter 动画做按钮；**老手**直接跳到 16.1.7-16.1.8，掌握 Widget 树的层级管理（AddChild/SetParent/MoveToBack）、焦点流（focus_flow）的设计、八个常见的 Widget 陷阱。

### 16.1.1 快速入门：Widget 是所有 UI 的基类

打开源码 `scripts/widgets/widget.lua` 第 1-32 行——这是整个 UI 系统的"根"：

```1:32:scripts\widgets\widget.lua
local Widget = Class(function(self, name)
	name = name or "widget"
    self.children = {}
    self.callbacks = {}
    self.name = name

    self.inst = CreateEntity()
    --if your widget does something that is based on gameplay, use these over the default, so that pausing freezes the effect.
    self.inst.DoSimPeriodicTask = self.inst.DoPeriodicTask
    self.inst.DoSimTaskInTime = self.inst.DoTaskInTime
    self.inst.widget = self
	self.inst.name = name

    self.inst:AddTag("widget")
    self.inst:AddTag("UI")
    self.inst.entity:SetName(name)
    self.inst.entity:AddUITransform()
    self.inst.entity:CallPrefabConstructionComplete()

    self:AddComponent("uianim")
                                    -- 注：此行实际是 self.inst:AddComponent("uianim")
    self:UpdateWhilePaused(true)

    self.enabled = true
    self.shown = true
    self.focus = false
    self.focus_target = false
    self.can_fade_alpha = true

    self.focus_flow = {}
    self.focus_flow_args = {}
end)
```

理解几件事：

1. **每个 Widget 内部都有一个 `self.inst` 实体** —— 但和游戏世界里的 prefab 不一样，这是 **UI 实体**——`AddUITransform`（不是 AddTransform）、带 `"UI"` tag、不参与碰撞、不参与寻路。
2. **Widget 树形结构** —— `self.children = {}` 存所有子 widget，`AddChild(child)` 添加后子 widget 的位置/旋转/缩放都**相对于父**。
3. **三个基础状态字段**：
   - `self.enabled` —— 启用/禁用（`Enable()` / `Disable()`）
   - `self.shown` —— 显示/隐藏（`Show()` / `Hide()`）
   - `self.focus` —— 是否被选中焦点（手柄/键盘高亮）
4. **`self.focus_flow`** —— 焦点流向表，控制方向键如何在 widget 间跳。

**新手只需记住一句话**：

> 所有 UI 控件——`Text`、`Image`、`ImageButton`、`Screen`、`HUD` ——都**继承自 `Widget`**。它们的差异只在"内部用 `AddTextWidget` / `AddImageWidget` / `AddAnimState` 之类引擎组件挂上具体显示能力"——但它们的**层级、显示、焦点机制全部一致**。

学 Widget 系统的第一步是建立这个统一感——不要把 `Text` 当 "字符串"、`Image` 当"图片"——它们都**只是 Widget 的特化版**。

### 16.1.2 快速入门：`Text` 控件——5 行代码显示一行文字

`scripts/widgets/text.lua` 是最简单的 Widget 子类。看它的构造函数：

```3:18:scripts\widgets\text.lua
local Text = Class(Widget, function(self, font, size, text, colour)
    Widget._ctor(self, "Text")

    self.inst.entity:AddTextWidget()

    self.inst.TextWidget:SetFont(font)
    self.font = font

	self:SetSize(size)

    self:SetColour(colour or { 1, 1, 1, 1 })

    if text ~= nil then
        self:SetString(text)
    end
end)
```

**完整使用流程**：

```lua
local Text = require "widgets/text"

-- 在某个 Screen 的构造函数里：
local label = self:AddChild(Text(NEWFONT, 30, "Hello DST!", {1, 1, 1, 1}))
label:SetPosition(0, 100)
```

参数说明：

| 参数 | 类型 | 说明 |
|------|------|------|
| `font` | string | 字体名（见下表） |
| `size` | number | 字号（建议 18-50） |
| `text` | string | 初始文字（可空） |
| `colour` | {r,g,b,a} | 颜色，0-1 浮点 |

**Klei 标准字体表**（来自 `scripts/fonts.lua`）：

| 常量 | 字体名 | 用途 |
|------|-------|------|
| `NEWFONT` | spirequal | 默认 UI 字体 |
| `NEWFONT_OUTLINE` | spirequal_outline | 带描边版（HUD 上用，看得清） |
| `TITLEFONT` | bp100 | 大标题（菜单标题） |
| `UIFONT` | bp50 | 中等 UI 文字 |
| `BUTTONFONT` | buttonfont | 按钮文字 |
| `DIALOGFONT` | opensans | 对话框文字（科技感） |
| `BODYTEXTFONT` | stint-ucr | 长段落（食谱、地图说明） |
| `HEADERFONT` | hammerhead | 副标题 |
| `CHATFONT` | bellefair | 聊天 |

**`Text` 的常用 API**：

```lua
label:SetString("New text")           -- 改文字
label:SetColour(1, 0, 0, 1)            -- 改颜色（红）
label:SetSize(40)                      -- 改字号
label:SetFont(TITLEFONT)               -- 改字体

label:SetVAlign(ANCHOR_MIDDLE)         -- 垂直对齐
label:SetHAlign(ANCHOR_LEFT)           -- 水平对齐
label:SetRegionSize(400, 100)          -- 限制区域
label:EnableWordWrap(true)             -- 自动换行

label:SetTruncatedString("Long...", 200, 30, "...")   -- 长文字截断
label:SetMultilineTruncatedString(str, 5, 400, ...)   -- 多行截断
```

**新手要避免的两个坑**：

- ❌ **字体没注册却用了** —— 用了 `"my_custom_font"` 但 mod 没声明字体资源——Text 会显示空白。新手别动字体，用 Klei 的常量就行。
- ❌ **Size 太大没区域限制** —— `SetSize(80)` 但没 `SetRegionSize`——长文字溢出屏幕。

### 16.1.3 快速入门：`Image` 控件——5 行代码显示一张图片

`scripts/widgets/image.lua` 第 3-15 行：

```3:15:scripts\widgets\image.lua
Image = Class(Widget, function(self, atlas, tex, default_tex)
    Widget._ctor(self, "Image")

    self.inst.entity:AddImageWidget()

    assert( ( atlas == nil and tex == nil ) or ( atlas ~= nil and tex ~= nil ) )

    self.tint = {1,1,1,1}

    if atlas and tex then
		self:SetTexture(atlas, tex, default_tex)
    end
end)
```

注意 **assert** —— `atlas` 和 `tex` 必须**同时**给或同时不给，不能只给一个。

**完整使用流程**：

```lua
local Image = require "widgets/image"

-- 显示一张 Klei 内置图（HUD atlas 里的某个元素）
local icon = self:AddChild(
    Image("images/hud.xml", "inv_slot.tex"))
icon:SetPosition(0, 0)
icon:SetSize(64, 64)

-- 显示自己 mod 的图
local mylogo = self:AddChild(
    Image("images/mymod.xml", "logo.tex"))
mylogo:SetSize(128, 64)
mylogo:SetTint(1, 0.5, 0.5, 1)        -- 染色（粉红）
```

**`Image` 的常用 API**：

```lua
image:SetTexture(atlas, tex)          -- 改图
image:SetTint(r, g, b, a)             -- 染色乘子
image:SetSize(w, h)                   -- 显示尺寸
image:ScaleToSize(w, h)               -- 等比缩放到指定尺寸
image:SetVRegPoint(ANCHOR_MIDDLE)     -- 垂直锚点
image:SetHRegPoint(ANCHOR_MIDDLE)     -- 水平锚点

image:SetMouseOverTexture(atlas, tex)  -- 鼠标悬停时换图（比如按钮高亮）
image:SetDisabledTexture(atlas, tex)   -- 禁用时换图

image:SetUVScale(2, 2)                 -- UV 缩放（图集动画）
image:SetBlendMode(BLENDMODE.Additive) -- 叠加模式（光晕/特效）
image:SetEffect("shaders/myshader.ksh") -- 应用 shader
image:SetEffectParams(r, g, b, a)      -- shader 参数
```

**进阶提示**——`SetMouseOverTexture` 是怎么工作的？看 `image.lua` 第 162-176 行：

```162:176:scripts\widgets\image.lua
function Image:OnMouseOver()
	--print("Image:OnMouseOver", self)
	if self.enabled and self.mouseovertex then
		self.inst.ImageWidget:SetTexture(self.atlas, self.mouseovertex)
	end
	Widget.OnMouseOver( self )
end

function Image:OnMouseOut()
	--print("Image:OnMouseOut", self)
	if self.enabled and self.mouseovertex then
		self.inst.ImageWidget:SetTexture(self.atlas, self.texture)
	end
	Widget.OnMouseOut( self )
end
```

鼠标进入/离开时自动切贴图——不需要你手动监听事件。这是**饥荒 UI 设计的标准模式**：常见交互行为内置在 widget 里，使用者只要"声明数据"（哪个图、哪个贴图）。

### 16.1.4 进阶：`ImageButton` —— 按钮的"五态"模型

按钮看起来比 Image 复杂，但本质就是**带状态的 Image + 文字 + 鼠标响应**。源码 `scripts/widgets/imagebutton.lua` 第 4-21 行：

```4:21:scripts\widgets\imagebutton.lua
local ImageButton = Class(Button, function(self, atlas, normal, focus, disabled, down, selected, scale, offset)
    Button._ctor(self, "ImageButton")

    self.image = self:AddChild(Image())
    self.image:MoveToBack()

	self:SetTextures( atlas, normal, focus, disabled, down, selected, scale, offset )

    self.scale_on_focus = true
    self.move_on_click = true

    self.focus_scale = {1.2, 1.2, 1.2}
    self.normal_scale = {1, 1, 1}

    self.focus_sound = nil
end)
```

`ImageButton` 内部就是 `Button`（详见 16.1.6）+ 一个子 `Image`。重点是**五个状态贴图**：

| 状态 | 何时显示 | 字段名 |
|------|---------|--------|
| **normal** | 默认（没鼠标、没禁用、没选中） | `image_normal` |
| **focus** | 鼠标悬停或手柄 highlight | `image_focus` |
| **disabled** | `Disable()` 后 | `image_disabled` |
| **down** | 按下还没松开（瞬间） | `image_down` |
| **selected** | `Select()` 后（"已选中页"持续状态） | `image_selected` |

**完整用法**：

```lua
local ImageButton = require "widgets/imagebutton"

-- 5 状态都用同一张图（最简单）
local btn = self:AddChild(ImageButton(
    "images/mymod.xml",
    "btn_normal.tex"))

-- 完整指定 5 状态
local btn = self:AddChild(ImageButton(
    "images/mymod.xml",
    "btn_normal.tex",       -- normal
    "btn_hover.tex",        -- focus
    "btn_disabled.tex",     -- disabled
    "btn_pressed.tex",      -- down
    "btn_active.tex"))      -- selected

-- 设文字
btn:SetText("Click Me")
btn:SetFont(BUTTONFONT)
btn:SetTextSize(30)
btn:SetTextColour(0, 0, 0, 1)
btn:SetTextFocusColour(1, 1, 1, 1)
btn:SetTextDisabledColour(0.5, 0.5, 0.5, 1)

-- 设回调
btn:SetOnClick(function()
    print("Clicked!")
end)

-- 大小（不调默认是图原始尺寸）
btn:ForceImageSize(200, 60)

-- focus 时的额外缩放（默认 1.2 倍）
btn:SetFocusScale(1.1, 1.1)
```

**自动状态切换**——`ImageButton:OnGainFocus`（第 108 行）会自动切到 focus 贴图、`OnDisable` 自动切到 disabled 贴图。**你只管声明数据，状态切换是免费的**。

**Klei 默认按钮**——如果不传 atlas，会用默认的 `images/frontend.xml` + `button_long*.tex`：

```43:53:scripts\widgets\imagebutton.lua
function ImageButton:SetTextures(atlas, normal, focus, disabled, down, selected, image_scale, image_offset)
    local default_textures = false
    if not atlas then
        atlas = atlas or "images/frontend.xml"
        normal = normal or "button_long.tex"
        focus = focus or "button_long_halfshadow.tex"
        disabled = disabled or "button_long_disabled.tex"
        down = down or "button_long_halfshadow.tex"
        selected = selected or "button_long_disabled.tex"
        default_textures = true
    end
```

**进阶用法**：`SetImageNormalColour` / `SetImageFocusColour` / `SetImageDisabledColour` —— 不换贴图，只染色。一张白图 + 不同颜色 = 多态按钮（省美术资源）。

### 16.1.5 进阶：`AnimButton` —— 用 Spriter 动画做按钮

`AnimButton` 是 `ImageButton` 的"高配版"——用**Spriter 动画**而不是静态图作为按钮表面。源码 `scripts/widgets/animbutton.lua` 全文：

```1:18:scripts\widgets\animbutton.lua
local Widget = require "widgets/widget"
local Button = require "widgets/button"
local UIAnim = require "widgets/uianim"

local AnimButton = Class(Button, function(self, animname, states)
    Button._ctor(self, "AnimButton")
    self.anim = self:AddChild(UIAnim())
    self.anim:MoveToBack()
    self.anim:GetAnimState():SetBuild(animname)
    self.anim:GetAnimState():SetBank(animname)

    if states then
    	self.animstates = states
    end

    self.anim:GetAnimState():PlayAnimation(self.animstates and self.animstates.idle or "idle")
    self.anim:GetAnimState():SetRayTestOnBB(true);
end)
```

它内部用一个 **UIAnim**（widgets/uianim.lua）作为容器，UIAnim 包装了 `AnimState`（13.4 学过）——所以你可以**给按钮播 idle / over / disabled 三个动画**，按钮自动根据状态切：

```20:34:scripts\widgets\animbutton.lua
function AnimButton:OnGainFocus()
	AnimButton._base.OnGainFocus(self)

	if self:IsEnabled() and not self:IsSelected() then
		self.anim:GetAnimState():PlayAnimation(self.animstates and self.animstates.over or "over")
	end
end

function AnimButton:OnLoseFocus()
	AnimButton._base.OnLoseFocus(self)

	if not (self:IsSelected() or self:IsDisabledState()) then
		self.anim:GetAnimState():PlayAnimation(self.animstates and self.animstates.idle or "idle")
    end
end
```

#### 用法

**第一步**——在 Spriter 里做一个按钮 build，含三个 anim：`idle`、`over`、`disabled`。

**第二步**——编译为 `anim/my_button.zip`（13.3 章方法）。

**第三步**——使用：

```lua
local AnimButton = require "widgets/animbutton"

-- 默认动画名（idle / over / disabled）
local btn = self:AddChild(AnimButton("my_button"))

-- 自定义动画名（如果你的 zip 里 anim 名不一样）
local btn = self:AddChild(AnimButton("my_button", {
    idle = "loop",
    over = "highlight",
    disabled = "gray",
}))

btn:SetText("Begin Game")
btn:SetOnClick(function() ... end)
```

**何时用 AnimButton 而不是 ImageButton**？

| 场景 | 推荐 |
|------|------|
| 矩形按钮、有 hover 略微变色 | `ImageButton`（资源少、性能好） |
| 主菜单"开始游戏"那种带闪光特效的 | `AnimButton`（动画 idle 循环） |
| 角色选择界面那种点中后剧烈缩放 | `AnimButton`（用 Spriter 制作复杂过渡） |
| 物品栏槽位、设置界面普通按钮 | `ImageButton`（够用） |

**老手提示**：`AnimButton` 的 `UIAnim` 用的还是动画系统（`AnimState`）——意味着你能用 13.4 的所有 API：`OverrideSymbol`、`SetMultColour`、`PushAnimation` 全都可用。比如可以做一个"按钮按下时播炸开特效"的视觉效果。

### 16.1.6 进阶：`Button` 基类的回调系统

`ImageButton` 和 `AnimButton` 都继承自 `Button`（`scripts/widgets/button.lua`）。`Button` 提供了**所有按钮共用的"事件回调"系统**——按下、点击、悬停、焦点。

源码 `button.lua` 第 56-101 行的 `OnControl`：

```56:101:scripts\widgets\button.lua
function Button:OnControl(control, down)
	if Button._base.OnControl(self, control, down) then return true end

	if not self:IsEnabled() or not self.focus then return false end

	if self:IsSelected() and not self.AllowOnControlWhenSelected then return false end

	if control == self.control and (not self.mouseonly or TheFrontEnd.isprimary) then

		if down then
			if not self.down then
                if not self.stopclicksound then
					TheFrontEnd:GetSound():PlaySound(self.overrideclicksound or "dontstarve/HUD/click_move")
                end
				self.o_pos = self:GetLocalPosition()
				self:SetPosition(self.o_pos + self.clickoffset)
				self.down = true
				if self.whiledown then
					self:StartUpdating()
				end
				if self.ondown then
					self.ondown()
				end
			end
		else
			if self.down then
				self.down = false
                self:ResetPreClickPosition()
				if self.onclick then
					self.onclick()
				end
				self:StopUpdating()
			end
		end

		return true
	end

end
```

**生命周期**：

```
鼠标按下 → ondown() 调用 → button.down = true
按住期间 → 每帧调 whiledown()（如果有）
松开 → onclick() 调用 → button.down = false
```

**API 对照表**：

| 钩子 | 何时触发 | 设置方法 |
|------|---------|---------|
| `ondown` | 按下瞬间 | `btn.ondown = fn` 或 `btn:SetOnDown(fn)` |
| `whiledown` | 按住期间每帧 | `btn:SetWhileDown(fn)` |
| `onclick` | 松开瞬间 | `btn.onclick = fn` 或 `btn:SetOnClick(fn)` |
| `onselect` | `btn:Select()` 后 | `btn:SetOnSelect(fn)` |
| `onunselect` | `btn:Unselect()` 后 | `btn:SetOnUnselect(fn)` |
| `OnGainFocus` (override) | 鼠标进入或手柄 highlight | 子类重写 |
| `OnLoseFocus` (override) | 鼠标离开或失焦 | 子类重写 |

**实战例子**——一个"长按充能"的能量法术按钮：

```lua
local btn = self:AddChild(ImageButton("images/mymod.xml", "spell_btn.tex"))
btn:SetText("Charge")

local charge = 0
btn:SetOnDown(function()
    charge = 0
    print("Started charging")
end)
btn:SetWhileDown(function()
    charge = charge + 0.1
    btn:SetText(string.format("Charge: %.0f%%", charge * 100))
end)
btn:SetOnClick(function()
    if charge >= 1 then
        cast_spell()
    else
        print("Insufficient charge")
    end
end)
```

**进阶要点**：

- **`SetControl(CONTROL_ATTACK)`** —— 改变按钮响应的输入。默认是 `CONTROL_ACCEPT`（鼠标左键 + 确定键）。
- **`mouseonly = true`** —— 只响应鼠标，不响应手柄/键盘。HUD 上的某些按钮（如背包扩展插槽）会用。
- **`SetHelpTextMessage(str)`** —— 设置手柄提示语（屏幕底部"X 选择"那种文字）。

### 16.1.7 老手进阶：Widget 的层级管理

`AddChild`、`MoveToBack`、`MoveToFront`——这是控制 widget 渲染层级的核心 API。

源码 `widget.lua` 第 300-309 行：

```300:309:scripts\widgets\widget.lua
function Widget:AddChild(child)
	if child.parent ~= nil then
		child.parent.children[child] = nil
	end

    self.children[child] = true
    child.parent = self
    child.inst.entity:SetParent(self.inst.entity)
	return child
end
```

注意三件事：

1. **重新 SetParent** —— 一个 widget 同时只能有一个父；调 `AddChild` 会**自动从原父解除**。
2. **`return child`** —— 返回值是 child 本身，所以可以**链式赋值**：`local label = self:AddChild(Text(...))`。
3. **位置/旋转/缩放都相对于父** —— 父移动子也跟着。

#### 渲染层级

源码 `widget.lua` 第 64-70 行：

```64:70:scripts\widgets\widget.lua
function Widget:MoveToBack()
    self.inst.entity:MoveToBack()
end

function Widget:MoveToFront()
    self.inst.entity:MoveToFront()
end
```

UI 渲染顺序——**同一父下的子按 AddChild 顺序从后往前渲染**（先 add 的在底层）。`MoveToBack` 把当前 widget 移到父的子列表最前面（最先渲染 = 最底层），`MoveToFront` 反之。

**典型场景**——按钮的 image 必须在文字下面：

```lua
-- imagebutton.lua 第 7-8 行
self.image = self:AddChild(Image())
self.image:MoveToBack()
-- 后续 self.text 会自动在 image 之上
```

#### 焦点流（Focus Flow）

`SetFocusChangeDir` 设置"按下方向键时焦点跳到哪个 widget"。源码 `widget.lua` 第 583-590 行：

```583:590:scripts\widgets\widget.lua
function Widget:SetFocusChangeDir(dir, widget, ...)
	if widget == nil then
		self.focus_flow[dir] = nil
		self.focus_flow_args[dir] = nil
	else
		self.focus_flow[dir] = widget
		self.focus_flow_args[dir] = arg.n > 0 and arg or nil
	end
end
```

**典型用法**——三个按钮上下排列、↓ 键依次跳转：

```lua
local btn1 = self:AddChild(ImageButton(...))
local btn2 = self:AddChild(ImageButton(...))
local btn3 = self:AddChild(ImageButton(...))

btn1:SetFocusChangeDir(MOVE_DOWN, btn2)
btn2:SetFocusChangeDir(MOVE_UP, btn1)
btn2:SetFocusChangeDir(MOVE_DOWN, btn3)
btn3:SetFocusChangeDir(MOVE_UP, btn2)
```

`MOVE_UP` / `MOVE_DOWN` / `MOVE_LEFT` / `MOVE_RIGHT` 是引擎常量。

**焦点流可以用函数返回值**——动态计算下一焦点：

```lua
btn1:SetFocusChangeDir(MOVE_DOWN, function()
    return self.has_advanced_options and btn_advanced or btn_default
end)
```

#### 显示控制

```lua
widget:Show()           -- 自身可见
widget:Hide()           -- 自身不可见（同时所有子也不可见）

widget:Enable()         -- 启用（响应输入）
widget:Disable()        -- 禁用（不响应、按钮变灰）

widget:SetClickable(false)  -- 不响应点击但仍显示

widget:SetCanFadeAlpha(false) -- 全屏 fade 时这个 widget 不参与（如固定在屏幕上的 logo）

widget:SetFadeAlpha(0.5)    -- 临时半透明
```

**注意 `Hide()` ≠ `Disable()`**：

- `Hide` 是**视觉**上的隐藏（不渲染、不响应）。
- `Disable` 是**逻辑**上的禁用（仍渲染但变灰、不响应输入）。
- `Hide` 自动隐藏所有子；`Disable` **不会**自动禁用子（详见 16.1.8 陷阱七）。

### 16.1.8 老手进阶：八个常见的 Widget 陷阱

#### 陷阱一：忘记 `AddChild`

```lua
-- ❌ 错误：创建了但没挂载
local label = Text(NEWFONT, 30, "Hello")
label:SetPosition(0, 100)
-- 屏幕上看不到！label 的 inst 没 SetParent，没人渲染它

-- ✅ 正确
local label = self:AddChild(Text(NEWFONT, 30, "Hello"))
```

`AddChild` 不仅是树管理——它**调用 `inst.entity:SetParent(self.inst.entity)`**，否则 widget 是"孤儿"，引擎不知道在哪儿渲染。

#### 陷阱二：`SetTexture` atlas/tex 只给一个

```lua
-- ❌ 错误：触发 assert
image:SetTexture(nil, "my.tex")
-- assert: ( atlas == nil and tex == nil ) or ( atlas ~= nil and tex ~= nil )

-- ✅ 正确
image:SetTexture("images/mymod.xml", "my.tex")
```

#### 陷阱三：在 master 端创建 widget

```lua
-- ❌ 错误：服务端创建 widget
if TheWorld.ismastersim then
    local btn = self:AddChild(ImageButton(...))  -- 客户端没收到
end

-- ✅ 正确：UI 永远只在客户端创建
-- HUD widget 应该在 ClassifyHUD 或 PlayerHud:postinit 里创建
```

UI **不通过网络同步**——服务端创建的 widget 客户端看不见。所有 UI 代码必须在**客户端运行**（即 `TheWorld.ismastersim == false` 或共用代码段）。

#### 陷阱四：在 mod 加载阶段创建 widget

```lua
-- modmain.lua
-- ❌ 错误：modmain 执行时游戏还没启动
local label = Text(NEWFONT, 30, "Hello")  -- 引擎崩溃 / 无效

-- ✅ 正确：通过 hook 在合适的生命周期创建
AddClassPostConstruct("screens/playerhud", function(self)
    -- self 是 PlayerHud 实例，已经初始化完毕
    self.mylabel = self:AddChild(Text(NEWFONT, 30, "Hello"))
end)
```

#### 陷阱五：忘记取消周期任务

```lua
-- ❌ 错误：widget 销毁后任务还在跑
function MyWidget:OnAppear()
    self._task = self.inst:DoStaticPeriodicTask(1, function()
        self:Update()  -- widget 销毁后这里 self.inst 是 nil
    end)
end
```

**修正**——重写 `Kill` 取消所有任务：

```lua
function MyWidget:Kill()
    if self._task then
        self._task:Cancel()
        self._task = nil
    end
    MyWidget._base.Kill(self)
end
```

#### 陷阱六：`SetSize` vs `SetScale` 混淆

```lua
-- 这两个完全是不同的事

image:SetSize(100, 100)   -- 改变 ImageWidget 的实际像素尺寸
image:SetScale(2, 2)       -- 把整个 widget（含子）缩放 2 倍
```

**经验**：

- 调 `SetSize` 时，**子 widget 不会跟着变** —— 用于精确控制单图大小。
- 调 `SetScale` 时，**整个子树跟着缩放** —— 用于"整体放大缩小一组 widget"。
- **按钮要等比放大用 `SetScale` 而不是 `SetSize`**——后者会拉伸、模糊。

#### 陷阱七：`Disable` 不会禁用子

```lua
-- 父 widget 禁用了，但子 button 还能点
parent:Disable()
print(child_button:IsEnabled())  -- nil 还是 true？
```

源码 `widget.lua` 第 233-241 行：

```233:241:scripts\widgets\widget.lua
function Widget:IsEnabled()
	if not self.enabled then
		return false
	elseif self.parent then
		return self.parent:IsEnabled()
	end
	return true
end
```

`IsEnabled` **检查整条链**：父禁用了，`child:IsEnabled()` 返回 false。但**`child.enabled` 字段还是 true** —— 这就是为什么 `OnEnable` / `OnDisable` 不会自动通知子（因为子的 `enabled` 没变）。

**实战**：用 `IsEnabled()` 判断"实际能否响应"，用 `enabled` 字段判断"子自己是不是被显式禁用"。

#### 陷阱八：focus 测试用 `IsFocusedState` 而不是 `focus`

```lua
-- ❌ 错误：直接读字段
if btn.focus then ...

-- ✅ 推荐：用方法
if btn:IsFocusedState() then ...
```

`IsFocusedState`（`button.lua` 第 225-227 行）是 `self.focus and self:IsEnabled() and not self:IsSelected()`——更"语义化"。直接读 `focus` 字段在按钮被 selected/disabled 时也会是 true，可能误判。

### 16.1.9 小结

```
       ┌──────────────────────────────────┐
       │       Widget (基类)              │
       │  - children / focus_flow         │
       │  - Show/Hide/Enable/Disable      │
       └─────────────┬────────────────────┘
                     │
       ┌─────────────┼──────────────┬─────────────┬─────────────────┐
       ▼             ▼              ▼             ▼                 ▼
    Text          Image          Button        UIAnim         其它（如
   (文字)        (图片)         (按钮基类)    (动画容器)       Spinner、
                                  │                            Menu...）
                                  │
                       ┌──────────┼──────────┐
                       ▼                     ▼
                  ImageButton           AnimButton
                  (静态贴图按钮)       (动画按钮)
                  - 5 状态贴图        - 3 anim 状态
                  - 5 状态颜色
                  - SetText/Font/Size
```

| 概念 | 速记 |
|------|------|
| **`Widget` 是所有 UI 的基类** | `AddUITransform` + `"UI"` tag + 树形 children + focus 状态 |
| **`Text(font, size, str, colour)`** | 显示文字，5 行能搞定 |
| **`Image(atlas, tex)`** | 显示图，atlas/tex 必须同时给 |
| **`ImageButton(atlas, normal, focus, disabled, down, selected)`** | 五态贴图按钮 |
| **`AnimButton(animname, states)`** | Spriter 动画按钮（idle/over/disabled）|
| **`Button` 基类回调** | `ondown` / `whiledown` / `onclick` / `onselect` |
| **`AddChild(c)`** | 加子，自动 SetParent，返回 c |
| **`MoveToBack` / `MoveToFront`** | 控制渲染顺序（同父子之间） |
| **`SetFocusChangeDir(dir, w)`** | 焦点流——方向键如何跳 |
| **`Show / Hide` vs `Enable / Disable`** | 视觉显隐 vs 逻辑可用性 |
| **`IsEnabled()`** | 检查整条链；`self.enabled` 是单个 widget 状态 |
| **`SetSize` vs `SetScale`** | 改单 widget 像素 vs 整子树缩放 |

**新手核心三句**：所有 UI 都继承自 `Widget`；`Text`/`Image` 5 行代码就能用；按钮分 `ImageButton`（静态）和 `AnimButton`（动画）。

**进阶核心三句**：`ImageButton` 五状态（normal/focus/disabled/down/selected）+ 自动切换；`Button` 基类提供 ondown/whiledown/onclick 三件套；`AnimButton` 内部用 `UIAnim` 包装 `AnimState`，所以 13.4 的所有动画 API 都可用。

**老手核心三句**：`AddChild` 自动 SetParent，焦点流通过 `SetFocusChangeDir` 配置；`Disable` **不会**自动禁用子（`IsEnabled` 才检查链）；八个常见坑里"忘 AddChild"和"master 端建 widget"占多数——**永远在客户端通过 ClassPostConstruct 创建 UI** 能避开 90% 问题。

下一节（16.2）我们将讨论**布局系统与锚点**——`SetVAnchor` / `SetHAnchor` / `SetScaleMode` 如何让 UI 在不同分辨率下正确显示，以及如何把多个 widget"对齐排列"做出整洁界面。


## 16.2 布局系统与锚点

### 本节导读

16.1 我们学了 Widget 如何"创建"——`Text`、`Image`、`ImageButton`、`AnimButton`。但是有一个非常重要的问题没解决——**这些 widget 在屏幕上到底放在哪里**？

新手 mod 作者常犯的错——`label:SetPosition(100, 100)` 写死坐标。但是玩家用 1080p 显示器和 4K 显示器看到的画面**完全不同**：

- 1080p：(100, 100) 在屏幕中下偏左
- 4K：(100, 100) 几乎在屏幕中心
- 21:9 超宽屏：(100, 100) 完全错位

**Klei 的 UI 是怎么解决这个问题的？** 三套机制配合：

1. **锚点（Anchor）** —— "我贴在屏幕的哪个边角？"（左上 / 右下 / 中心）
2. **缩放模式（ScaleMode）** —— "屏幕变大变小时我怎么响应？"（拉伸 / 等比缩放 / 不变）
3. **HUD 缩放（HUDScale）** —— 玩家在设置里调"UI 大小"时整个 HUD 缩放

理解了这三套机制+锚点常量+对齐 API，你就能写出**在任何分辨率下都正确显示的 UI**——这是和 16.1 同样重要的"地基"。

> **新手**从 16.2.1-16.2.3 起步——理解锚点的含义、3 个最常用的锚点模式、用 5 行代码做一个"贴在屏幕右上角"的 widget；**进阶读者**继续看 16.2.4-16.2.6，深入 5 种 SCALEMODE 的差异、`SetMaxPropUpscale` 的"最大放大倍数"、Klei HUD 的"分屏八区"标准模式（topleft_root / bottom_root / topright_over_root 等）；**老手**跳到 16.2.7-16.2.8，逐行拆解 `controls.lua:493-553` 的 `MakeScalingNodes`、HUDScale 玩家配置如何作用、Text 的 `SetVAlign` / `SetHAlign` / `SetRegionSize` 文字对齐机制、八个最常见的"我的 UI 在 4K 屏崩了"的坑。

读完本节，你能写出**在 720p / 1080p / 1440p / 4K / 超宽屏全部正常显示的 mod UI**——并且能立刻看懂 Klei 任意 widget 文件里的"锚点 + 缩放"模板。

---

### 16.2.1 快速入门：屏幕坐标系——原点在哪？

#### 第一步：游戏的 UI 坐标系

打开任何 widget 文件——`SetPosition(100, 100)`。**这个 (100, 100) 到底是哪？**

饥荒 UI 默认坐标系：

- **原点 (0, 0) 在屏幕几何中心**
- **x 轴向右为正**（不是屏幕左上角！）
- **y 轴向上为正**（不是向下！屏幕坐标系反着的）
- **单位是"虚拟像素"** —— 设计基准分辨率 `RESOLUTION_X = 1280, RESOLUTION_Y = 720`（来自 `scripts/constants.lua:20-21`）

```20:21:scripts/constants.lua
RESOLUTION_X = 1280
RESOLUTION_Y = 720
```

也就是说，所有 UI 设计**都是按 1280×720 的画布写**——意味着：

- 屏幕左边沿 ≈ x = -640
- 屏幕右边沿 ≈ x = +640
- 屏幕顶边沿 ≈ y = +360
- 屏幕底边沿 ≈ y = -360

```
                    ↑ y = +360 (顶)
                    │
                    │
       x = -640 ────●──── x = +640
       (左)         │     (右)
                    │
                    │
                    ↓ y = -360 (底)

                  原点 = 屏幕中心
```

#### 第二步：当屏幕分辨率变化时

如果玩家用的是 1080p（1920×1080）或者 4K（3840×2160）—— **坐标系数值不变**！还是从 (-640, +360) 到 (+640, -360)！这就是"虚拟像素"——**Klei 引擎自动把虚拟坐标缩放到真实分辨率**。

但是这只解决了一半问题——如果你把按钮放在 `(640, 360)`（1280×720 的右上角），在 4K 上**也还是右上角**。但是如果你把按钮放在 `(0, 0)`（中心）——在任何分辨率都是中心。

**问题来了**——`(640, 360)` 是 1280×720 的右上角，但是 4K 屏宽高比变成 16:9 反过来宽屏 21:9 时，**屏幕实际宽度比 1280 大**——按钮可能"看起来还是中心偏右"，**离右边沿有空白**。

这就是**锚点（Anchor）系统**要解决的问题。

> **新手记忆**：UI 坐标系**原点在屏幕中心**，y 轴向上，单位是虚拟像素（1280×720 基准）。**写死坐标在多分辨率下会出问题——必须配合锚点系统**。

---

### 16.2.2 快速入门：锚点（Anchor）—— 把 widget 贴在屏幕的哪个边

锚点让你**告诉引擎**："我希望我贴在屏幕的左边/右边/上面/下面"——而不是用相对中心的虚拟坐标。

#### 第一步：5 个 Anchor 常量

`scripts/constants.lua:76-80`：

```76:80:scripts/constants.lua
ANCHOR_MIDDLE = 0
ANCHOR_LEFT = 1
ANCHOR_RIGHT = 2
ANCHOR_TOP = 1
ANCHOR_BOTTOM = 2
```

**注意**——这是 5 个常量但**只有 3 个值（0/1/2）**！原因——`H` 和 `V` 共用同一组数字：

| 常量 | 值 | 用途 |
|-----|----|----|
| `ANCHOR_MIDDLE` | 0 | 横向居中 / 纵向居中 |
| `ANCHOR_LEFT` | 1 | 横向贴左 |
| `ANCHOR_RIGHT` | 2 | 横向贴右 |
| `ANCHOR_TOP` | 1 | 纵向贴上 |
| `ANCHOR_BOTTOM` | 2 | 纵向贴下 |

`ANCHOR_LEFT` 和 `ANCHOR_TOP` 都是 1，`ANCHOR_RIGHT` 和 `ANCHOR_BOTTOM` 都是 2——**约定俗成的复用**——意思是"靠最小坐标的一边"和"靠最大坐标的一边"。

#### 第二步：`SetHAnchor` 和 `SetVAnchor`

```416:422:scripts/widgets/widget.lua
function Widget:SetVAnchor(anchor)
    self.inst.UITransform:SetVAnchor(anchor)
end

function Widget:SetHAnchor(anchor)
    self.inst.UITransform:SetHAnchor(anchor)
end
```

**含义**：

- `SetHAnchor(ANCHOR_LEFT)` —— widget 的 (0, 0) 对齐到**屏幕左边沿**
- `SetHAnchor(ANCHOR_RIGHT)` —— widget 的 (0, 0) 对齐到**屏幕右边沿**
- `SetHAnchor(ANCHOR_MIDDLE)` —— widget 的 (0, 0) 对齐到**屏幕水平中线**（默认）
- `SetVAnchor(ANCHOR_TOP)` —— widget 的 (0, 0) 对齐到**屏幕顶边沿**
- `SetVAnchor(ANCHOR_BOTTOM)` —— widget 的 (0, 0) 对齐到**屏幕底边沿**
- `SetVAnchor(ANCHOR_MIDDLE)` —— widget 的 (0, 0) 对齐到**屏幕垂直中线**（默认）

**关键**：锚点是**改变 (0, 0) 的位置**——而不是改变 widget 自己的"画布位置"。设了锚点之后，再 `SetPosition(x, y)` 是**相对锚点的偏移**。

#### 第三步：贴右上角的 widget

```lua
local panel = self:AddChild(Image("images/mymod.xml", "panel.tex"))
panel:SetHAnchor(ANCHOR_RIGHT)   -- widget 自己的 (0,0) = 屏幕右边
panel:SetVAnchor(ANCHOR_TOP)     -- widget 自己的 (0,0) = 屏幕顶边
panel:SetPosition(-100, -50)     -- 离右上角 100 像素左、50 像素下
panel:SetSize(180, 80)
```

**结果**：在 720p / 1080p / 4K / 超宽屏，**这个 panel 永远贴在右上角**，和右上角的距离都是 100×50 虚拟像素。

#### 第四步：贴底部居中的 widget（HUD 物品栏）

```lua
local invbar = self:AddChild(Image(...))
invbar:SetHAnchor(ANCHOR_MIDDLE)   -- 水平居中
invbar:SetVAnchor(ANCHOR_BOTTOM)   -- 贴底部
invbar:SetPosition(0, 50)          -- 离底部 50 像素上
```

**结果**：物品栏始终在屏幕底部水平居中，这就是饥荒物品栏的标准做法（实际位于 `bottom_root` 下，看 `controls.lua:508-512`）：

```508:512:scripts/widgets/controls.lua
    self.bottom_root = self:AddChild(Widget("bottom"))
    self.bottom_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.bottom_root:SetHAnchor(ANCHOR_MIDDLE)
    self.bottom_root:SetVAnchor(ANCHOR_BOTTOM)
    self.bottom_root:SetMaxPropUpscale(MAX_HUD_SCALE)
```

> **新手记忆**：`SetHAnchor` + `SetVAnchor` = 决定 widget 的 (0, 0) 在屏幕哪个边角；3×3 = **9 种锚点组合**（左上 / 上中 / 右上 / 左中 / 中心 / 右中 / 左下 / 下中 / 右下）。**贴边设计不要写死坐标**。

---

### 16.2.3 快速入门：5 种 SCALEMODE —— 屏幕变大时我怎么变？

锚点解决了"贴在哪个边角"的问题，但还有一个问题——**屏幕越来越大时，UI 元素本身要不要变大？**

`scripts/constants.lua:82-86`：

```82:86:scripts/constants.lua
SCALEMODE_NONE = 0
SCALEMODE_FILLSCREEN = 1 --stretch art to fit/fill window
SCALEMODE_PROPORTIONAL = 2 --preserve aspect ratio (picks the smaller of horizontal/vertical scale)
SCALEMODE_FIXEDPROPORTIONAL = 3 --same as SCALEMODE_FIXEDSCREEN_NONDYNAMIC, except for safe area on consoles
SCALEMODE_FIXEDSCREEN_NONDYNAMIC = 4 --scale same amount as window scaling from 1280x720
```

#### 5 种模式速查表

| 模式 | 数值 | 行为 | 典型用途 |
|------|-----|-----|---------|
| `SCALEMODE_NONE` | 0 | **不缩放**——按虚拟像素显示 | 调试覆盖、临时元素 |
| `SCALEMODE_FILLSCREEN` | 1 | **拉伸填满**——非等比缩放，可能拉变形 | 全屏黑色覆盖、暗角 |
| `SCALEMODE_PROPORTIONAL` | 2 | **等比缩放**——取 H/V 较小者，保持宽高比 | 99% 的常规 UI |
| `SCALEMODE_FIXEDPROPORTIONAL` | 3 | **等比 + 安全区**——主机平台留边 | 主机版 UI |
| `SCALEMODE_FIXEDSCREEN_NONDYNAMIC` | 4 | **固定缩放**——按 1280×720 缩放系数 | 不希望随分辨率变化 |

#### 第二步：实战——用 `SCALEMODE_PROPORTIONAL` 99% 用对

在 `controls.lua` 里，所有"分屏 root"都用 `SCALEMODE_PROPORTIONAL`——这是 Klei 的**标准做法**。来看右下角的 root 节点：

```526:530:scripts/widgets/controls.lua
    self.bottomright_root = self:AddChild(Widget("bottomright"))
    self.bottomright_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.bottomright_root:SetHAnchor(ANCHOR_RIGHT)
    self.bottomright_root:SetVAnchor(ANCHOR_BOTTOM)
    self.bottomright_root:SetMaxPropUpscale(MAX_HUD_SCALE)
```

**SCALEMODE_PROPORTIONAL 的算法**：

```
横向缩放 = 实际宽度 / 1280
纵向缩放 = 实际高度 / 720

最终缩放 = min(横向缩放, 纵向缩放)
```

**举例**：

- 1920×1080：横向 1.5 / 纵向 1.5 → 整体放大 1.5 倍
- 4K (3840×2160)：横向 3.0 / 纵向 3.0 → 整体放大 3 倍
- 21:9 超宽屏 (3440×1440)：横向 2.69 / 纵向 2.0 → 取小者 **2.0**——意味着图标不会因为屏幕宽就过度放大

这就是"等比缩放 + 防止过度放大"——保证 UI 在任何分辨率下**比例正确、清晰可读**。

#### 第三步：`SCALEMODE_FILLSCREEN` 的特殊用途

```84:91:scripts/widgets/controls.lua
    self.blackoverlay = self:AddChild(Image("images/global.xml", "square.tex"))
    self.blackoverlay:SetVRegPoint(ANCHOR_MIDDLE)
    self.blackoverlay:SetHRegPoint(ANCHOR_MIDDLE)
    self.blackoverlay:SetVAnchor(ANCHOR_MIDDLE)
    self.blackoverlay:SetHAnchor(ANCHOR_MIDDLE)
    self.blackoverlay:SetScaleMode(SCALEMODE_FILLSCREEN)
    self.blackoverlay:SetClickable(false)
    self.blackoverlay:SetTint(0,0,0,.5)
```

`SCALEMODE_FILLSCREEN` 用在 **`blackoverlay`** 上——这是个全屏的黑色半透明遮罩（用于淡入淡出转场）。它的图本来只是 1×1 的纯色块，**用 FILLSCREEN 拉伸到撑满屏幕**——就是个动态尺寸的"屏幕涂层"。

**典型用途**：

- 死亡时屏幕变红
- 受冷时屏幕泛蓝
- 进入睡眠时屏幕变黑
- 弹窗背后的"灰色遮罩"

#### 第四步：`SetMaxPropUpscale(MAX_HUD_SCALE)` —— 防止过度放大

`scripts/constants.lua:42`：

```42:42:scripts/constants.lua
MAX_HUD_SCALE = 1.25
```

`SetMaxPropUpscale(1.25)` 的意思——**最大放大倍数为 1.25**。即使分辨率超大（比如 8K），HUD 也只放大 1.25 倍——避免"血条占半个屏幕"的尴尬场面。

```500:500:scripts/widgets/controls.lua
    self.top_root:SetMaxPropUpscale(MAX_HUD_SCALE)
```

每个 HUD root 都设了这个上限——这是 Klei "在大屏上 UI 不要太大"的设计。**做菜单、弹窗时**通常用 `MAX_FE_SCALE = 3`（前端 UI 可以更大一点，因为是全屏占据）。

> **新手记忆**：99% 的 UI 用 `SCALEMODE_PROPORTIONAL` + `SetMaxPropUpscale`；全屏遮罩用 `SCALEMODE_FILLSCREEN`；不想随分辨率变的用 `SCALEMODE_NONE`。

---

### 16.2.4 进阶：Klei HUD 的"分屏八区"标准模式

打开 `scripts/widgets/controls.lua:493-553`——这是 Klei HUD 的**核心架构**——把整个屏幕分成 8 个 root 节点，每个节点带正确的锚点+缩放：

```493:553:scripts/widgets/controls.lua
function Controls:MakeScalingNodes()

    --these are auto-scaling root nodes
    self.top_root = self:AddChild(Widget("top"))
    self.top_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.top_root:SetHAnchor(ANCHOR_MIDDLE)
    self.top_root:SetVAnchor(ANCHOR_TOP)
    self.top_root:SetMaxPropUpscale(MAX_HUD_SCALE)

    self.topleft_root = self:AddChild(Widget("topleft"))
    self.topleft_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.topleft_root:SetHAnchor(ANCHOR_LEFT)
    self.topleft_root:SetVAnchor(ANCHOR_TOP)
    self.topleft_root:SetMaxPropUpscale(MAX_HUD_SCALE)

    self.bottom_root = self:AddChild(Widget("bottom"))
    self.bottom_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.bottom_root:SetHAnchor(ANCHOR_MIDDLE)
    self.bottom_root:SetVAnchor(ANCHOR_BOTTOM)
    self.bottom_root:SetMaxPropUpscale(MAX_HUD_SCALE)

    self.topright_root = self:AddChild(Widget("side"))
    self.topright_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.topright_root:SetHAnchor(ANCHOR_RIGHT)
    self.topright_root:SetVAnchor(ANCHOR_TOP)
    self.topright_root:SetMaxPropUpscale(MAX_HUD_SCALE)

    self.right_root = self:AddChild(Widget("right_root"))
    self.right_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.right_root:SetHAnchor(ANCHOR_RIGHT)
    self.right_root:SetVAnchor(ANCHOR_MIDDLE)
    self.right_root:SetMaxPropUpscale(MAX_HUD_SCALE)

    self.bottomright_root = self:AddChild(Widget("bottomright"))
    self.bottomright_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.bottomright_root:SetHAnchor(ANCHOR_RIGHT)
    self.bottomright_root:SetVAnchor(ANCHOR_BOTTOM)
    self.bottomright_root:SetMaxPropUpscale(MAX_HUD_SCALE)

    self.left_root = self:AddChild(Widget("left_root"))
    self.left_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.left_root:SetHAnchor(ANCHOR_LEFT)
    self.left_root:SetVAnchor(ANCHOR_MIDDLE)
    self.left_root:SetMaxPropUpscale(MAX_HUD_SCALE)

    self.topright_over_root = self:AddChild(Widget("topright_over"))
    self.topright_over_root:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.topright_over_root:SetHAnchor(ANCHOR_RIGHT)
    self.topright_over_root:SetVAnchor(ANCHOR_TOP)
    self.topright_over_root:SetMaxPropUpscale(MAX_HUD_SCALE)
	
    --these are for introducing user-configurable hud scale
    self.topleft_root = self.topleft_root:AddChild(Widget("tl_scale_root"))
    self.topright_root = self.topright_root:AddChild(Widget("tr_scale_root"))
    self.bottom_root = self.bottom_root:AddChild(Widget("bottom_scale_root"))
    self.top_root = self.top_root:AddChild(Widget("top_scale_root"))
    self.left_root = self.left_root:AddChild(Widget("left_scale_root"))
    self.right_root = self.right_root:AddChild(Widget("right_scale_root"))
    self.bottomright_root = self.bottomright_root:AddChild(Widget("br_scale_root"))
    self.topright_over_root = self.topright_over_root:AddChild(Widget("tr_over_scale_root"))
end
```

#### 第一步：8 个 root 的对应位置

```
┌─────────────────────────────────────────────────────────┐
│ topleft_root        top_root              topright_root │
│ ANCHOR_LEFT/TOP    ANCHOR_MIDDLE/TOP   ANCHOR_RIGHT/TOP │
│ ─────────                ─────────             ─────────│
│                                                         │
│                                          topright_over  │
│                                                         │
│ left_root                                right_root     │
│ ANCHOR_LEFT/MIDDLE                       ANCHOR_RIGHT/  │
│                                          MIDDLE         │
│ ─────────                                ─────────      │
│                                                         │
│                                                         │
│                                          bottomright    │
│                                          ANCHOR_RIGHT/  │
│                                          BOTTOM         │
│                                                         │
│                  bottom_root                            │
│                  ANCHOR_MIDDLE/BOTTOM                   │
└─────────────────────────────────────────────────────────┘
```

#### 第二步：这 8 个 root 装的什么？

回到 `controls.lua:111-128`，看具体哪些 widget 被加到哪个 root：

```111:128:scripts/widgets/controls.lua
    self.item_notification = self.topleft_root:AddChild(GiftItemToast(self.owner, self))
    self.item_notification:SetPosition(115, 150, 0)
	table.insert(self.toastitems, self.item_notification)

    self.yotb_notification = self.topleft_root:AddChild(YotbToast(self.owner, self))
    self.yotb_notification:SetPosition(215, 150, 0)
	table.insert(self.toastitems, self.yotb_notification)

    self.skilltree_notification = self.topleft_root:AddChild(SkillTreeToast(self.owner, self))
    self.skilltree_notification:SetPosition(315, 150, 0)
	table.insert(self.toastitems, self.skilltree_notification)

    self.scrapbook_notification = self.topleft_root:AddChild(ScrapbookToast(self.owner, self))
    self.scrapbook_notification:SetPosition(415, 0, 0)

    --self.worldresettimer = self.bottom_root:AddChild(WorldResetTimer(self.owner))
    self.worldresettimer = self.bottom_root:AddChild(PlayerDeathNotification(self.owner))
    self.inv = self.bottom_root:AddChild(Inv(self.owner))
```

| Root | 内容 |
|------|------|
| `topleft_root` | 礼物提示、技能树解锁提示、皮肤箱通知、地图册新条目通知 |
| `top_root` | 顶部状态栏（饱食 / 三围条等） |
| `topright_root` | 角色头像、HUD scale 设置 |
| `topright_over_root` | "已保存"小图标（覆盖在 topright 之上）|
| `bottom_root` | **物品栏**、世界重置计时器、死亡通知 |
| `bottomright_root` | 鼠标右键提示 |
| `left_root` | 制作菜单（按 Tab 召唤）|
| `right_root` | （主机版的辅助菜单）|

#### 第三步：mod 添加自定义 HUD 元素

**做法**——找到合适的 root，作为父节点：

```lua
AddClassPostConstruct("widgets/controls", function(self)
    -- 在右下角加一个图标
    self.my_widget = self.bottomright_root:AddChild(
        Image("images/mymod.xml", "myicon.tex"))
    self.my_widget:SetPosition(-100, 100)
    self.my_widget:SetSize(64, 64)
end)
```

**关键**：直接挂到 `bottomright_root` 后，你的 widget **自动继承所有锚点和缩放设置**——你只需要 `SetPosition` 和 `SetSize`，分辨率适配是免费的。

#### 第四步：双层 root 的玄机

注意 `controls.lua:545-552`：

```545:552:scripts/widgets/controls.lua
    self.topleft_root = self.topleft_root:AddChild(Widget("tl_scale_root"))
    self.topright_root = self.topright_root:AddChild(Widget("tr_scale_root"))
    self.bottom_root = self.bottom_root:AddChild(Widget("bottom_scale_root"))
    self.top_root = self.top_root:AddChild(Widget("top_scale_root"))
    self.left_root = self.left_root:AddChild(Widget("left_scale_root"))
    self.right_root = self.right_root:AddChild(Widget("right_scale_root"))
    self.bottomright_root = self.bottomright_root:AddChild(Widget("br_scale_root"))
    self.topright_over_root = self.topright_over_root:AddChild(Widget("tr_over_scale_root"))
```

**奇怪的代码**——把 `self.topleft_root` 重新赋值成它的子 widget。为什么？

**原因**——下面 `SetHUDSize`（`controls.lua:559-571`）会调用 `SetScale` 来响应玩家的"HUD 大小"设置：

```555:572:scripts/widgets/controls.lua
function Controls:SetHUDSize()
    local scale = TheFrontEnd:GetHUDScale()
	local crafting_scale = TheFrontEnd:GetCraftingMenuScale()

    self.topleft_root:SetScale(scale)
    self.topright_root:SetScale(scale)
    self.bottom_root:SetScale(scale)
    self.top_root:SetScale(scale)
    self.bottomright_root:SetScale(scale)
    self.containerroot:SetScale(scale)
	self.containerroot_under:SetScale(scale)
	self.containerroot_over:SetScale(scale)
    self.containerroot_side:SetScale(scale)
    self.containerroot_side_behind:SetScale(scale)

    self.hover:SetScale(scale)
    self.topright_over_root:SetScale(scale)
```

`SetScale` 会**改变锚点+缩放设置**导致冲突——所以 Klei 的策略是**双层 root**：

- **外层** root —— 持有锚点 + 缩放模式（不会被改）
- **内层** root —— 通过 SetScale 响应玩家配置

这样玩家把 HUD 大小调到 1.5 倍时，每个 root 的内层 widget 都被放大 1.5 倍——**而锚点保持不变**。

> **进阶记忆**：Klei HUD 的 8 个 root 是**标准位置框架**——mod 加 widget 时直接挂到对应 root 就行；外层负责锚点+缩放，内层负责 HUDScale——双层结构防冲突。

---

### 16.2.5 进阶：`SetVRegPoint` 和 `SetHRegPoint` —— Image 的"自身锚点"

刚才讲的 `SetHAnchor` / `SetVAnchor` 是"widget 在屏幕上的锚点"——但是还有另一组 API：`SetVRegPoint` / `SetHRegPoint`——它们是"图片自身的对齐点"。

#### 第一步：两套 API 的区别

```
SetHAnchor / SetVAnchor   ← widget 在父容器/屏幕上的"贴边方向"
                            （决定 (0,0) 在父的哪个位置）

SetVRegPoint / SetHRegPoint ← 图片本身的"对齐点"
                              （决定图片中心点在 widget 坐标系的哪里）
```

**类比**：

- 锚点（Anchor）是"我贴在房间的哪面墙"
- RegPoint 是"图片自己的'中心'是哪里"——图片的左上角对齐还是中心对齐还是右下角对齐到 widget 的位置坐标

#### 第二步：默认行为

默认所有 Image 的 RegPoint 是 `ANCHOR_MIDDLE`——意味着**图片中心**对齐到 widget 的 (0, 0)：

```lua
local img = Image("images/mymod.xml", "myicon.tex")  -- 假设是 100×100
img:SetPosition(0, 0)
-- 图片的中心在 (0, 0)，所以图片的边沿在 (-50, -50) 到 (+50, +50)
```

#### 第三步：改变 RegPoint

```lua
img:SetVRegPoint(ANCHOR_TOP)    -- 图片顶边对齐到 widget 的 y=0
img:SetHRegPoint(ANCHOR_LEFT)   -- 图片左边对齐到 widget 的 x=0
img:SetPosition(0, 0)
-- 图片的左上角在 (0, 0)，所以图片的边沿在 (0, -100) 到 (+100, 0)
```

**实际用途**：

- 默认 ANCHOR_MIDDLE：适合"图标围绕一个点"——比如玩家头像围绕中心
- ANCHOR_LEFT + ANCHOR_TOP：适合"列表项"——每个 item 从左上角对齐
- ANCHOR_RIGHT + ANCHOR_BOTTOM：适合"右下角弹窗"——从右下角对齐

#### 第四步：例子—— blackoverlay 同时用两套

`controls.lua:84-91`：

```84:91:scripts/widgets/controls.lua
    self.blackoverlay = self:AddChild(Image("images/global.xml", "square.tex"))
    self.blackoverlay:SetVRegPoint(ANCHOR_MIDDLE)
    self.blackoverlay:SetHRegPoint(ANCHOR_MIDDLE)
    self.blackoverlay:SetVAnchor(ANCHOR_MIDDLE)
    self.blackoverlay:SetHAnchor(ANCHOR_MIDDLE)
    self.blackoverlay:SetScaleMode(SCALEMODE_FILLSCREEN)
```

**两套都设了 MIDDLE**——意思是：

- 锚点：widget 的 (0, 0) = 屏幕中心
- RegPoint：图片中心对齐到 widget 的 (0, 0)
- 缩放：拉伸到屏幕大小

合起来——**这张图永远撑满屏幕，并且中心对中心**——完美的全屏遮罩。

> **进阶记忆**：`SetH/VAnchor` = widget 在父中的位置；`SetH/VRegPoint` = 图片自身相对 widget 坐标的对齐点。**两套常配合使用**——锚点决定"贴墙"，RegPoint 决定"图片本身怎么对齐到那个点"。

---

### 16.2.6 进阶：Text 的对齐 —— `SetVAlign` / `SetHAlign` / `SetRegionSize`

Text 的对齐机制和 Image 又有些不同——它有**两层概念**：

1. **Text widget 在父中的锚点** —— `SetHAnchor` / `SetVAnchor`（同 Widget）
2. **文字在 Text 自身"区域"内的对齐** —— `SetVAlign` / `SetHAlign` + `SetRegionSize`

#### 第一步：常规用法

```lua
local label = Text(NEWFONT, 30, "Hello DST!", {1,1,1,1})
label:SetPosition(0, 100)
-- 默认行为——文字以中心点为锚，居中显示
```

**默认情况下**——Text widget 的 (0, 0) 就是文字中心。这个适合"短标签"。

#### 第二步：长文字、列表、限制宽度

如果文字长（"Welcome to the world of DST, brave survivor!"），又想限制宽度（比如 400px），就需要**给文字设个"区域"**：

```lua
label:SetRegionSize(400, 50)        -- 限制 400×50 的显示区域
label:SetHAlign(ANCHOR_LEFT)        -- 文字在区域内左对齐
label:SetVAlign(ANCHOR_TOP)          -- 文字在区域内顶对齐
```

#### 第三步：实际例子—— skilltreebuilder 的对齐

```95:95:scripts/widgets/redux/skilltreebuilder.lua
    self.root.xp_tospend:SetHAlign(ANCHOR_LEFT)  
```

```126:138:scripts/widgets/redux/skilltreewidget.lua
    self.root.infopanel.title:SetVAlign(ANCHOR_TOP)

    self.root.infopanel.desc:SetHAlign(ANCHOR_LEFT)
    self.root.infopanel.desc:SetVAlign(ANCHOR_TOP)
```

技能树面板的"信息文本"——左对齐 + 顶对齐——典型的"段落文字"格式。

#### 第四步：`EnableWordWrap`

如果文字超出 RegionSize 的宽度，可以**自动换行**：

```lua
label:EnableWordWrap(true)
label:SetRegionSize(400, 200)
label:SetString("A long story that goes on and on. The text will wrap automatically when it exceeds the 400-pixel-wide region.")
```

#### 第五步：长文字截断

```lua
label:SetTruncatedString("A very long string", 200, 30, "...")
-- 第二参数 200 = 最大宽度
-- 第三参数 30 = 字号（影响字符宽度计算）
-- 第四参数 "..." = 截断后追加的省略号
```

如果文字超过 200 像素宽，自动变成 "A very long st..."。

> **进阶记忆**：Text 的对齐有**两层**——widget 锚点（在父中位置）+ 文字对齐（在自身 region 内位置）。**长文字必须 SetRegionSize 才能控制对齐**；用 `EnableWordWrap` 自动换行；用 `SetTruncatedString` 加省略号。

---

### 16.2.7 老手：HUD Scale —— 玩家配置如何作用 + 完整流程

`SetScale` 是 widget 的本地缩放——但是还有一层"玩家可调缩放"。让我们追一下完整流程。

#### 第一步：玩家配置入口

游戏菜单里"设置 → 显示 → HUD 大小"——这是个 0.6 ~ 1.6 之间的滑块。玩家调它会触发 `TheFrontEnd:GetHUDScale()` 返回新值。

#### 第二步：Controls 监听并应用

`controls.lua:555-572` 的 `SetHUDSize`（已引用）——`scale` 来自 `GetHUDScale`，然后**调用每个 root 的 `SetScale(scale)`**。

#### 第三步：分层独立 scale

注意 `controls.lua:586-592`：

```586:592:scripts/widgets/controls.lua
	if IsGameInstance(Instances.Player1) then
		self.left_root:SetScale(crafting_scale)
	    self.right_root:SetScale(scale)
	else
		self.left_root:SetScale(scale)
	    self.right_root:SetScale(crafting_scale)
	end
```

`left_root`（制作菜单）有**独立的 crafting scale**——玩家可以单独调"制作菜单大小"。这种"分层独立缩放"是 Klei 给玩家精细控制权的设计。

#### 第四步：mod 怎么响应玩家的 HUD scale

如果你的 mod widget 直接挂到某个 root（比如 `bottomright_root`）——**自动响应**所有 scale。但是如果 mod 不挂到 root，需要手动监听：

```lua
local function UpdateScale(self)
    self:SetScale(TheFrontEnd:GetHUDScale())
end

inst:ListenForEvent("hud_scale_changed", UpdateScale, TheFrontEnd)
```

或者更简单——直接挂到 root，省心。

#### 第五步：`MAX_HUD_SCALE` vs `MAX_FE_SCALE`

```41:42:scripts/constants.lua
MAX_FE_SCALE = 3 --Default if you don't call SetMaxPropUpscale
MAX_HUD_SCALE = 1.25
```

- `MAX_HUD_SCALE = 1.25` —— HUD 元素**最大放大 1.25 倍**（避免占满屏）
- `MAX_FE_SCALE = 3` —— Front End（菜单/弹窗）**默认最大 3 倍**——更激进

**为什么 HUD 比 FE 更保守**？因为 HUD 长时间显示在游戏画面上，**太大会挡视线**；而 FE 是全屏 modal，可以放更大。

> **老手记忆**：HUD scale 通过 `SetScale` 应用到内层 scale_root；外层 root 保持锚点和 SCALEMODE_PROPORTIONAL 不变；`MAX_HUD_SCALE = 1.25` < `MAX_FE_SCALE = 3` 因为 HUD 不能挡视线。

---

### 16.2.8 老手：八个最容易踩的"分辨率适配坑"

#### 坑 1：写死像素坐标在大屏崩了

**新手最常见**：

```lua
btn:SetPosition(640, 360)  -- 想放在 1280x720 的右上角
-- 在 1080p 上看起来是中心偏右上！
```

**原因**：(640, 360) 是相对**当前锚点**的坐标。如果没设锚点，默认 (0, 0) 在屏幕中心——(640, 360) 是离中心 640 像素右、360 像素上——但是**屏幕变大时这个偏移不变**，所以不再贴右上角。

**修复**：

```lua
btn:SetHAnchor(ANCHOR_RIGHT)
btn:SetVAnchor(ANCHOR_TOP)
btn:SetPosition(-50, -50)  -- 离右上角偏 50px 内
```

#### 坑 2：忘记 SetScaleMode

```lua
local widget = self:AddChild(MyWidget())
widget:SetHAnchor(ANCHOR_RIGHT)
widget:SetVAnchor(ANCHOR_BOTTOM)
-- 缺 SetScaleMode！
-- 在不同分辨率下 widget 实际像素尺寸**不变**——大屏上看起来超小
```

**修复**：

```lua
widget:SetScaleMode(SCALEMODE_PROPORTIONAL)
widget:SetMaxPropUpscale(MAX_HUD_SCALE)
```

#### 坑 3：嵌套 widget 反复 SetScaleMode

```lua
local outer = self:AddChild(Widget(""))
outer:SetScaleMode(SCALEMODE_PROPORTIONAL)
outer:SetHAnchor(ANCHOR_LEFT)

local inner = outer:AddChild(Widget(""))
inner:SetScaleMode(SCALEMODE_PROPORTIONAL)  -- 错——会重复缩放！
inner:SetHAnchor(ANCHOR_LEFT)               -- 错——锚点会冲突
```

**铁律**：**只在 root 节点设锚点和 ScaleMode**——子 widget 用相对坐标即可。

#### 坑 4：HUD 在 21:9 超宽屏被裁剪

**症状**：玩家在 3440×1440 屏幕反映"血条/物品栏的左右两边被裁掉了"。

**原因**：`SCALEMODE_PROPORTIONAL` 取**横纵两者较小**——超宽屏上纵向缩放小，所以整体只放大 2×（不是 2.69×）——但是某些 widget 设了死宽度（如背景框）超出可用宽度。

**修复**：用 `TheSim:GetScreenSize()` 动态计算：

```lua
local function UpdateLayout(widget)
    local w, h = TheSim:GetScreenSize()
    local aspect = w / h
    if aspect > 16 / 9 then
        -- 超宽屏，紧凑布局
        widget.bg:SetSize(800, 200)
    else
        widget.bg:SetSize(1000, 200)
    end
end
```

#### 坑 5：Anchor 设了但没生效

```lua
local widget = Widget("")
widget:SetHAnchor(ANCHOR_RIGHT)
-- 屏幕上看起来还是中心
```

**原因**：widget **没被 AddChild 到任何东西**——没父，锚点无意义。

**修复**：先 AddChild 到 screen / hud root，再设锚点。

#### 坑 6：Image 的 RegPoint 错了导致点不准

```lua
local btn_img = Image(...)
btn_img:SetVRegPoint(ANCHOR_TOP)  -- 图片顶边对齐到 (0, 0)
btn:AddChild(btn_img)
btn:SetOnClick(...)
-- 玩家点击 (0, 0) 附近**点不到按钮**——因为按钮其实在 (0, -100) 到 (+100, 0)
```

**修复**：保持默认 ANCHOR_MIDDLE，或者**调整 button 的 hitbox**（`button:SetHitRegionFromTexture`）。

#### 坑 7：Text 在不同分辨率下"模糊"

**症状**：4K 屏文字看起来"糊了"。

**原因**：Text 的字体是位图字体（`.fnt + .tex`）。**位图字体被放大就会糊**。

**修复**：

- 用更大的字号——`SetSize(50)` 而不是 `SetSize(20)`
- 或者切换 vector 字体（如果有）
- 或者用 `SetSize(20)` + `widget:SetMaxPropUpscale(2)` 限制放大倍数

#### 坑 8：`SetPosition` 用 (x, y) vs (x, y, z)

```lua
widget:SetPosition(100, 50)         -- 2D
widget:SetPosition(100, 50, 0)       -- 3D（z 通常是 0）
widget:SetPosition(Vector3(100, 50, 0))  -- Vector3
```

**坑**——某些情况下 z ≠ 0 会让 widget 进入 3D 渲染层，**可能被场景物体遮挡**！UI 必须严格 z = 0。

```lua
-- 别这么写
widget:SetPosition(100, 50, 5)  -- z = 5 —— 可能消失或被遮挡
```

---

### 16.2.9 小结：从"贴位置"到"响应分辨率"

```
              UI 坐标系（虚拟像素 1280×720）
                       │
                       ▼
              SetHAnchor / SetVAnchor
              "我贴在父的哪个边角"
              ANCHOR_LEFT / RIGHT / TOP / BOTTOM / MIDDLE
                       │
                       ▼
              SetScaleMode
              "屏幕变大变小时怎么响应"
              SCALEMODE_PROPORTIONAL（最常用）
              SCALEMODE_FILLSCREEN（全屏遮罩）
              SCALEMODE_NONE（不变）
                       │
                       ▼
              SetMaxPropUpscale
              "最大放大几倍"
              MAX_HUD_SCALE = 1.25 (HUD)
              MAX_FE_SCALE = 3 (FE)
                       │
                       ▼
              SetVRegPoint / SetHRegPoint (Image)
              SetVAlign / SetHAlign (Text)
              "图片/文字本身怎么对齐"
                       │
                       ▼
              SetScale (受玩家 HUDScale 影响)
              "玩家在设置里调 UI 大小"
                       │
                       ▼
              最终在屏幕上的位置和大小
```

| 概念 | 作用 |
|-----|------|
| **9 种锚点组合** | 决定 widget 在屏幕的"贴位置" |
| **SCALEMODE_PROPORTIONAL** | 99% 的 UI 用这个——等比响应分辨率 |
| **SetMaxPropUpscale** | 防止"屏幕大就 UI 巨大" |
| **8 个 root 标准模式** | controls.lua 提供的"分屏 root"——mod 加 widget 直接挂上 |
| **双层 root** | 外层管锚点+ScaleMode、内层管 HUDScale，避免冲突 |
| **SetVAlign/SetHAlign** | Text 的"区域内对齐" |
| **SetVRegPoint/SetHRegPoint** | Image 的"自身对齐点" |
| **EnableWordWrap + SetRegionSize** | 长文字自动换行 |

**新手核心三句**：UI 坐标原点在屏幕中心、y 轴向上、单位是虚拟像素（1280×720）；**贴边设计用 SetHAnchor + SetVAnchor**；**99% 用 SCALEMODE_PROPORTIONAL + MAX_HUD_SCALE**。

**进阶核心三句**：Klei HUD 的**8 个 root** 是标准位置框架——mod 挂到对应 root 自动适配；外层 root 管锚点+ScaleMode、内层 scale_root 管 HUDScale；`SetVRegPoint` ≠ `SetVAnchor`——前者是图自身锚点，后者是 widget 在父中的锚点。

**老手核心三句**：HUD scale 通过 `SetScale` 应用到内层 scale_root；超宽屏问题用 `TheSim:GetScreenSize()` 动态计算；八个常见坑里"写死坐标"和"忘 SetScaleMode"占多数——**永远挂到 controls 的 root 节点上**能避开 90% 问题。

下一节（16.3）我们将讨论**Screen 系统**——弹窗、菜单、设置面板的"全屏 widget"层级，以及 Screen 之间的栈管理（`TheFrontEnd:PushScreen` / `PopScreen`）和焦点流。


## 16.3 Screen 系统——弹窗与菜单

> **写在前面**——上两节我们学会了用 `Widget` 做"零件"、用锚点和 ScaleMode 把零件"摆到屏幕的对的位置"。**但这些零件还浮在空中**——必须有人**接管它们的输入**（鼠标点击、键盘按键、手柄按键）、**有人决定它们什么时候出现什么时候消失**、**有人协调多个面板的层级关系**（弹窗在菜单上、菜单在 HUD 上）。
>
> 这个"管理者"就是 **`Screen`** —— Widget 的特殊子类，是**所有"占满屏幕的 UI 单元"的容器**。HUD 是一个 Screen、主菜单是一个 Screen、暂停菜单是一个 Screen、错误提示弹窗是一个 Screen——它们由 `TheFrontEnd` 统一管理在一个**栈**里。
>
> 学会 Screen 系统等于学会了：
>
> - 怎么**从 mod 弹出一个对话框**（"确定要删除存档吗？"）
> - 怎么**做一个 mod 设置面板**（点 mod 配置打开自己的设置界面）
> - 怎么**响应键盘/手柄/鼠标输入**（按 ESC 关闭、按 Tab 切焦点）
> - 怎么**在 Screen 之间淡入淡出过场**（黑屏切换、白屏闪现）
>
> **新手**先看 16.3.1-16.3.3——理解 Screen 是 Widget 的子类、用 `PushScreen` / `PopScreen` 显示和关闭、模仿 `PopupDialogScreen` 写一个最简对话框；**进阶读者**继续看 16.3.4-16.3.6，深入屏幕栈机制（`screenstack`、`OnBecomeActive` / `OnBecomeInactive`）、`OnControl` / `OnRawKey` 的事件分发链、淡入淡出与场景过渡（`Fade` / `FadeToScreen`）；**老手**直接看 16.3.7-16.3.8，掌握 `default_focus` 与焦点恢复、自定义 mod 配置面板（`AddClassPostConstruct` 注入既有 Screen）、五个常见的 Screen 陷阱。

### 16.3.1 快速入门：Screen 是 Widget 的子类

打开 `scripts/widgets/screen.lua` 全文（这文件特别短）：

```1:11:scripts\widgets\screen.lua
local Widget = require "widgets/widget"
require "widgets/image"

local Screen = Class(Widget, function(self, name)
    Widget._ctor(self, name)
	--self.focusstack = {}
	--self.focusindex = 0
	self.handlers = {}
	--self.inst:Hide()
    self.is_screen = true
end)
```

注意三件事：

1. **Screen 继承自 Widget** —— 所有 Widget 的能力（`AddChild`、`SetVAnchor`、`Show`、`Hide`、`SetFocus`）一并继承。
2. **`self.handlers = {}`** —— 自己的"事件处理器表"（`AddEventHandler` / `HandleEvent`）。
3. **`self.is_screen = true`** —— 类型标记，引擎用它区分"普通 widget"和"屏幕级 widget"。

新增的关键方法（同文件第 13-77 行）：

```13:77:scripts\widgets\screen.lua
function Screen:OnCreate()
end

function Screen:GetHelpText()
	return ""
end

function Screen:OnDestroy()
	self:Kill()
end

function Screen:OnUpdate(dt)
	return true
end

function Screen:OnBecomeInactive()
	self.last_focus = self:GetDeepestFocus()
end

function Screen:OnBecomeActive()
    LastUIRoot = self.inst.entity
	TheSim:SetUIRoot(self.inst.entity)
	if self.last_focus and self.last_focus.inst.entity:IsValid() then
		self.last_focus:SetFocus()
	else
		self.last_focus = nil
		if self.default_focus then
			self.default_focus:SetFocus()
		end
	end
end

function Screen:AddEventHandler(event, fn)
	if not self.handlers[event] then
		self.handlers[event] = {}
	end

	self.handlers[event][fn] = true

	return fn
end

function Screen:RemoveEventHandler(event, fn)
	if self.handlers[event] then
		self.handlers[event][fn] = nil
	end
end

function Screen:HandleEvent(type, ...)
	local handlers = self.handlers[type]
	if handlers then
		for k,v in pairs(handlers) do
			k(...)
		end
	end
end

function Screen:SetDefaultFocus()
	if self.default_focus then
		self.default_focus:SetFocus()
		return true
	end
end
```

**Screen 比 Widget 多了什么**：

| 方法 | 作用 |
|------|------|
| `OnCreate()` | 创建后调用（多数 Screen 不重写）|
| `OnDestroy()` | 销毁时调用，默认 `Kill()` |
| `OnUpdate(dt)` | 每帧调用（HUD 用得多，菜单极少）|
| `OnBecomeActive()` | 变成栈顶时调（被显示）|
| `OnBecomeInactive()` | 不再是栈顶时调（被新 Screen 盖住）|
| `default_focus` | 字段——Screen 显示时默认聚焦哪个 widget |
| `last_focus` | 字段——上次失活前焦点在哪（恢复时用）|
| `AddEventHandler(event, fn)` | 监听全局事件（`OnLoad`、`OnSave`） |
| `GetHelpText()` | 返回屏幕底部的"操作提示"字符串 |

**新手需要记住一句话**：**Screen 是带"显示状态生命周期"的 Widget**。普通 Widget 只有"显示/隐藏"两态，Screen 有"激活/失活/栈顶/被覆盖"四态——因为它要参与`TheFrontEnd` 管理的"栈"。

### 16.3.2 快速入门：`PushScreen` / `PopScreen`

`TheFrontEnd:PushScreen(screen)` 把 Screen "推入栈顶"——立即显示。`TheFrontEnd:PopScreen()` "弹出栈顶"——关闭最上面那个。

源码 `scripts/frontend.lua` 第 960-993 行的核心：

```960:993:scripts\frontend.lua
function FrontEnd:PushScreen(screen)
	self.focus_locked = false
	self:SetForceProcessTextInput(false)
	TheInputProxy:FlushInput()
    Print(VERBOSITY.DEBUG, 'FrontEnd:PushScreen', screen.name)
    if #self.screenstack > 0 then
        self.screenstack[#self.screenstack]:OnBecomeInactive()
    end

    self.screenroot:AddChild(screen)
    table.insert(self.screenstack, screen)
    self.consoletext:MoveToFront()
    self.serverpausewidget:MoveToFront()
    self.serverpausewidget:SetOffset(0, 0)

    if screen.OffsetServerPausedWidget then
        screen:OffsetServerPausedWidget(self.serverpausewidget)
    end

    if not self.tracking_mouse then
        screen:SetDefaultFocus()
    end
    screen:OnBecomeActive()
    self:Update(0)
end
```

整个流程：

```
TheFrontEnd:PushScreen(myscreen)
    ↓
1. 当前栈顶 :OnBecomeInactive()  → 它的 last_focus 被记下
2. self.screenroot:AddChild(myscreen)  → 挂到屏幕根
3. table.insert(screenstack, myscreen)  → 入栈
4. consoletext / serverpausewidget MoveToFront  → 总是在最上
5. myscreen:SetDefaultFocus()  → 焦点跳到 default_focus
6. myscreen:OnBecomeActive()  → 触发激活回调
```

**`PopScreen` 反过来**：

```
TheFrontEnd:PopScreen()        -- 不传参 = 弹栈顶
    ↓
1. 栈顶 :OnBecomeInactive()
2. table.remove(screenstack)
3. screen:OnDestroy()  → 默认调 Kill()
4. screenroot:RemoveChild(screen)
5. 新栈顶 :OnBecomeActive()  → 焦点恢复
```

**API 总览**：

```lua
TheFrontEnd:PushScreen(screen)            -- 入栈+显示
TheFrontEnd:PopScreen()                   -- 弹栈顶
TheFrontEnd:PopScreen(specific_screen)    -- 弹指定 screen（不一定栈顶）
TheFrontEnd:GetActiveScreen()             -- 返回栈顶 Screen
TheFrontEnd:GetOpenScreenOfType("name")   -- 按名字查找
TheFrontEnd:GetScreenStackSize()          -- 栈深
TheFrontEnd:ClearScreens()                -- 全部弹出（场景切换用）
```

**栈的视觉示例**——玩家在主菜单按"开始游戏"再按"建立服务器"再点"修改房间"：

```
┌──────────────────────────────┐
│ ServerCreationScreen (栈顶)  │ ← 当前活跃，处理输入
├──────────────────────────────┤
│ MultiplayerMainScreen        │ ← 失活但仍渲染（被 SCS 部分覆盖）
├──────────────────────────────┤
│ MainScreen                   │ ← 失活
└──────────────────────────────┘
```

按 ESC 一次 → 弹掉 SCS → MainMP 变栈顶 → 再按 ESC → 弹掉 MainMP → MainScreen 变栈顶。

### 16.3.3 快速入门：模仿 `PopupDialogScreen` 写一个对话框

`scripts/screens/popupdialog.lua` 是**学习 Screen 最好的入门样板**——精短、典型、覆盖常用模式。看核心构造函数（第 38-86 行）：

```38:86:scripts\screens\popupdialog.lua
local PopupDialogScreen = Class(Screen, function(self, title, text, buttons, scale_bg, spacing_override, style)
	Screen._ctor(self, "PopupDialogScreen")

    self.style = style or "light"
    assert(STYLES[self.style])

    self.black = self:AddChild(ImageButton("images/global.xml", "square.tex"))
    self.black.image:SetVRegPoint(ANCHOR_MIDDLE)
    self.black.image:SetHRegPoint(ANCHOR_MIDDLE)
    self.black.image:SetVAnchor(ANCHOR_MIDDLE)
    self.black.image:SetHAnchor(ANCHOR_MIDDLE)
    self.black.image:SetScaleMode(SCALEMODE_FILLSCREEN)
    self.black.image:SetTint(0,0,0,0)
    self.black:SetOnClick(function() --[[ eat the click ]] end)

    self.proot = self:AddChild(Widget("ROOT"))
    self.proot:SetVAnchor(ANCHOR_MIDDLE)
    self.proot:SetHAnchor(ANCHOR_MIDDLE)
    self.proot:SetPosition(0,0,0)
    self.proot:SetScaleMode(SCALEMODE_PROPORTIONAL)

    self.bg = STYLES[self.style].bgconstructor(self.proot)

    self.title = self.proot:AddChild(Text(STYLES[self.style].title.font, STYLES[self.style].title.size))
    self.title:SetPosition(5, 88, 0)
    self.title:SetString(title)

    self.text = self.proot:AddChild(Text(STYLES[self.style].text.font, STYLES[self.style].text.size))
    self.text:SetPosition(5, -15, 0)
    self.text:SetString(text)
    self.text:EnableWordWrap(true)
    self.text:SetRegionSize(500, 160)
    self.text:SetVAlign(ANCHOR_MIDDLE)

    local spacing = spacing_override or 200
	self.menu = self.proot:AddChild(Menu(buttons, spacing, true))
	self.menu:SetPosition(-(spacing*(#buttons-1))/2, -127, 0)
    for i,v in pairs(self.menu.items) do
        v:SetScale(.7)
    end
	self.buttons = buttons

	self.default_focus = self.menu     -- ★关键
end)
```

**理解这段代码的"五段式"**：

1. **吃点击的全屏黑层 `self.black`** —— `SCALEMODE_FILLSCREEN` + 透明、不响应任何点击但**遮住底层 Screen 的输入**。这是"模态对话框"的标准做法（防止玩家点到下面的菜单）。
2. **`self.proot` 标准内容根** —— `MIDDLE / MIDDLE / PROPORTIONAL`（16.2.4 学过的 fixed_root 模式）。
3. **背景图** —— 由 `STYLES[style].bgconstructor` 生成（light/dark 两套外观）。
4. **标题 + 正文** —— 普通 `Text` widget。
5. **`self.menu` Menu widget** —— 一组按钮的容器，传入按钮配置数组。
6. **`self.default_focus = self.menu`** —— 这一行很关键，让对话框出现时焦点直接在按钮上（手柄玩家不用动鼠标）。

**完整使用方式**（弹一个"是/否"对话框）：

```lua
local PopupDialogScreen = require "screens/popupdialog"

local function OnYes()
    print("Player chose YES")
    TheFrontEnd:PopScreen()
end

local function OnNo()
    print("Player chose NO")
    TheFrontEnd:PopScreen()
end

local dialog = PopupDialogScreen("Confirm", "Delete save?", {
    {text = "Yes", cb = OnYes},
    {text = "No",  cb = OnNo},
})

TheFrontEnd:PushScreen(dialog)
```

**新手也可以在自己的 Screen 内调用 PopupDialogScreen** —— 它是 Klei 提供的"现成对话框"，不必自己重新写。

### 16.3.4 进阶：屏幕栈与生命周期

#### 多屏栈的渲染层级

`TheFrontEnd.screenstack` 是普通 Lua 数组，**索引 1 是底层、`#screenstack` 是栈顶**。栈顶的 Screen：

- **接管所有输入**（OnControl、OnRawKey、OnTextInput、OnMouseButton 都先到栈顶）
- **渲染在最上层**（`AddChild` 的顺序决定渲染前后）
- **拥有焦点**（`SetFocus`、`default_focus` 在它上面生效）

**栈下的 Screen 不被销毁**——只是 `OnBecomeInactive`，它们的 widget 树仍然挂着、仍在内存里——所以从弹窗回主菜单时，主菜单不必重建。

#### `OnBecomeActive` / `OnBecomeInactive` 的实战用法

```lua
-- 我自己的 Screen
function MyScreen:OnBecomeActive()
    MyScreen._base.OnBecomeActive(self)   -- 别忘调基类

    -- 暂停游戏（如果是单机）
    if not TheNet:IsDedicated() then
        SetPause(true, "MyMod")
    end

    -- 开始播 BGM
    TheFrontEnd:GetSound():PlaySound("dontstarve/HUD/page_open")

    -- 取消淡入
    self:Show()
end

function MyScreen:OnBecomeInactive()
    MyScreen._base.OnBecomeInactive(self) -- 必须调，否则 last_focus 没记录

    -- 取消暂停
    SetPause(false)
end
```

**为什么基类必须调**？因为 `Screen:OnBecomeInactive` 会**记下 `last_focus`**——下次 `OnBecomeActive` 时焦点能恢复到上次的按钮上。**不调基类的话玩家关掉子菜单回到主菜单，焦点会丢**——手柄玩家会看不到光标。

#### `OnDestroy` 的实战用法

```lua
function MyScreen:OnDestroy()
    -- 取消所有自己开的周期任务
    if self._update_task then
        self._update_task:Cancel()
        self._update_task = nil
    end

    -- 解除事件监听
    if self._listener then
        self.inst:RemoveEventCallback("playerentered", self._listener, TheWorld)
        self._listener = nil
    end

    MyScreen._base.OnDestroy(self)        -- 调基类，触发 Kill()
end
```

不释放任务/监听器是 mod Screen 最常见的内存泄漏来源（详见 16.3.8 陷阱二）。

### 16.3.5 进阶：`OnControl` / `OnRawKey` 事件分发链

**输入到底怎么从硬件流到你的 Screen 上**？看 `scripts/frontend.lua` 第 412-489 行的 `FrontEnd:OnControl`：

```412:443:scripts\frontend.lua
function FrontEnd:OnControl(control, down)
    if self.textProcessorWidget ~= nil and not self.textProcessorWidget.focus and not down and control == CONTROL_PRIMARY then
        self:SetForceProcessTextInput(false, self.textProcessorWidget)
    end

    self.isprimary = control == CONTROL_PRIMARY
    if self:IsControlsDisabled() then
        self.isprimary = false
        return false
    elseif #self.screenstack > 0
        and not (self.textProcessorWidget ~= nil and not self.textProcessorWidget.focus and self.textProcessorWidget:OnControl(control == CONTROL_PRIMARY and CONTROL_ACCEPT or control, down))
		and self.screenstack[#self.screenstack]:OnControl(control == CONTROL_PRIMARY and CONTROL_ACCEPT or control, down)
	then
		self.isprimary = false
```

**核心一句**：`self.screenstack[#self.screenstack]:OnControl(...)` —— **栈顶 Screen 接管输入**。

**完整链条**：

```
键盘按 ESC
   ↓
TheInput:OnControl(CONTROL_CANCEL, true)
   ↓
TheFrontEnd:OnControl(CONTROL_CANCEL, true)
   ↓
[栈顶 Screen]:OnControl(CONTROL_CANCEL, true)
   ↓ （Widget:OnControl 默认行为）
   ↓
栈顶 Screen 的"焦点 widget" :OnControl
   ↓
焦点 widget 处理或返回 false
   ↓
没人处理 → ESC 透传到游戏世界（可能触发暂停）
```

#### 在自定义 Screen 重写 OnControl

模仿 `PopupDialogScreen:OnControl`（第 96-106 行）：

```96:106:scripts\screens\popupdialog.lua
function PopupDialogScreen:OnControl(control, down)
    if PopupDialogScreen._base.OnControl(self,control, down) then return true end

    if control == CONTROL_CANCEL and not down then
        if #self.buttons > 1 and self.buttons[#self.buttons] then
            self.buttons[#self.buttons].cb()
            TheFrontEnd:GetSound():PlaySound("dontstarve/HUD/click_move")
            return true
        end
    end
end
```

模式很标准：

```lua
function MyScreen:OnControl(control, down)
    if MyScreen._base.OnControl(self, control, down) then return true end
    -- ↑ 让基类先分发给焦点 widget；如果消费了就返回

    if control == CONTROL_CANCEL and not down then
        self:Close()
        return true
    end
    if control == CONTROL_MENU_MISC_2 and not down then  -- TAB
        self:CycleTabs()
        return true
    end
    -- 没处理就什么都不返回（隐式 nil = false）
end
```

**`return true`** 至关重要——告诉引擎"我消费了这个控制信号，不要再传"。**不返回 true 的话**——按 ESC 会同时关闭你的 Screen 和触发游戏暂停（双重响应）。

#### `OnRawKey` —— 处理"键码"（不是"控制"）

`OnControl` 用的是**逻辑控制**（CONTROL_ACCEPT、CONTROL_CANCEL……）——可以被玩家在设置里改键。

**`OnRawKey`** 处理**物理键码**（KEY_F、KEY_TAB……）——不被改键影响。**适合调试快捷键、文字输入**。

```lua
function MyScreen:OnRawKey(key, down)
    if MyScreen._base.OnRawKey(self, key, down) then return true end

    if key == KEY_F1 and down then
        self:ToggleDebugMode()
        return true
    end
end
```

#### `OnTextInput` —— 文字输入（输入框）

```lua
function MyScreen:OnTextInput(text)
    if MyScreen._base.OnTextInput(self, text) then return true end
    -- 通常你会让 TextEdit 子 widget 自己处理；自己只做拦截
end
```

#### 帮助提示——`GetHelpText`

屏幕底部的"X 选择 / B 取消"提示由 `GetHelpText` 返回：

```113:120:scripts\screens\popupdialog.lua
function PopupDialogScreen:GetHelpText()
	local controller_id = TheInput:GetControllerID()
	local t = {}
	if #self.buttons > 1 and self.buttons[#self.buttons] then
        table.insert(t, TheInput:GetLocalizedControl(controller_id, CONTROL_CANCEL) .. " " .. STRINGS.UI.HELP.BACK)
    end
	return table.concat(t, "  ")
end
```

`TheInput:GetLocalizedControl(controller_id, CONTROL_X)` 返回当前键位的本地化字符串（如 `"ESC"` 或 手柄 `"⊗"`）。

### 16.3.6 进阶：淡入淡出 / FadeToScreen / FadeBack

#### `Fade(in_or_out, time, cb)`

`scripts/frontend.lua` 第 1025-1050 行：

```1025:1050:scripts\frontend.lua
function FrontEnd:Fade(in_or_out, time_to_take, cb, fade_delay_time, delayovercb, fadeType)
	self.fadedir = in_or_out
	self.total_fade_time = time_to_take
	self.fadecb = cb
	self.fade_time = 0
	self.fade_type = fadeType or "black"
	if in_or_out == FADE_IN then
		self:SetFadeLevel(1)
	else
		if self.fade_type == "white" then
			self.topwhiteoverlay:SetTint(FADE_WHITE_COLOUR[1], FADE_WHITE_COLOUR[2], FADE_WHITE_COLOUR[3], 0)
			self.topvigoverlay:SetTint(1,1,1,0)
		elseif self.fade_type == "black" then
			self.topblackoverlay:SetTint(0,0,0,0)
		elseif self.fade_type == "swipe" then
			self.topswipeoverlay:SetTint(1,1,1,0)
			self.topswipeoverlay:SetEffectParams(0,0,0,0)
		end
		self:ShowTopFade()
        DoAutopause()
	end
	self.fade_delay_time = fade_delay_time
	self.delayovercb = delayovercb
end
```

**三种淡入淡出类型**：

- `"black"` —— 黑屏（**默认**，剧情过渡）
- `"white"` —— 白屏闪现（剧烈过渡）
- `"swipe"` —— 卷帘擦除（特殊场景）

**FADE_IN / FADE_OUT** 是引擎常量。

```lua
-- 黑屏淡入（屏幕从黑变明）
TheFrontEnd:Fade(FADE_IN, 1.0)

-- 黑屏淡出（屏幕从明变黑），之后回调
TheFrontEnd:Fade(FADE_OUT, 0.5, function()
    print("Fade complete")
end)
```

#### `FadeToScreen(existing, new_screen_fn, cb, fadetype)`

```1052:1066:scripts\frontend.lua
function FrontEnd:FadeToScreen( existing_screen, new_screen_fn, fade_complete_cp, fade_type )
	local fade_time = SCREEN_FADE_TIME
	if fade_type == "swipe" then
		fade_time = SWIPE_FADE_TIME
	end

	self:Fade(FADE_OUT, fade_time,
		function()
			local new_screen = new_screen_fn()
			TheFrontEnd:PushScreen( new_screen )
            TheFrontEnd:Fade(FADE_IN, fade_time, fade_complete_cp and function() fade_complete_cp(new_screen) end, 0, nil, fade_type )
            existing_screen:Hide()
		end,
	0, nil, fade_type)
end
```

"先淡出 → 推新 Screen → 隐藏老 Screen → 淡入"——**实现"剧情场景平滑过渡"**：

```lua
-- 主菜单 → 角色选择
TheFrontEnd:FadeToScreen(
    self,
    function() return CharacterSelectScreen() end,
    function(newscreen) print("Done") end,
    "black"
)
```

#### `FadeBack(cb, fadetype, fade_out_cb)`

```1068:1084:scripts\frontend.lua
function FrontEnd:FadeBack( fade_complete_cb, fade_type, fade_out_complete_cb )
	local fade_time = SCREEN_FADE_TIME
	if fade_type == "swipe" then
		fade_time = SWIPE_FADE_TIME
	end

	self:Fade(FADE_OUT, fade_time,
		function()
			if fade_out_complete_cb ~= nil then
				fade_out_complete_cb()
			end
			TheFrontEnd:PopScreen()
            TheFrontEnd:Fade(FADE_IN, fade_time, fade_complete_cb, 0, nil, fade_type)
            TheFrontEnd:GetActiveScreen():Show()
		end,
	0, nil, fade_type)
end
```

"先淡出 → 弹栈 → 让新栈顶 Show → 淡入"——**和 `FadeToScreen` 配对使用**：

```lua
-- 在子菜单按返回键
function MyScreen:OnControl(control, down)
    if control == CONTROL_CANCEL and not down then
        TheFrontEnd:FadeBack()  -- 平滑回上一屏
        return true
    end
end
```

### 16.3.7 老手进阶：`default_focus` 与焦点恢复

#### default_focus 的设计

`Screen:SetDefaultFocus` 在 PushScreen 时被调用——把焦点设到 `self.default_focus`：

```70:75:scripts\widgets\screen.lua
function Screen:SetDefaultFocus()
	if self.default_focus then
		self.default_focus:SetFocus()
		return true
	end
end
```

**为什么这关键**——**手柄玩家没有鼠标**，进 Screen 第一帧就需要"光标"在某个按钮上才能玩。鼠标玩家也喜欢"按 ESC 键就能直接确认默认按钮"——也需要焦点在默认按钮上。

**实战**：

```lua
local MyScreen = Class(Screen, function(self)
    Screen._ctor(self, "MyScreen")
    self.proot = self:AddChild(Widget("proot"))
    -- ...
    self.btn_yes = self.proot:AddChild(...)
    self.btn_no = self.proot:AddChild(...)

    -- ★关键
    self.default_focus = self.btn_yes  -- 默认聚焦"是"按钮
end)
```

#### last_focus 自动恢复

`OnBecomeInactive` 记录 `last_focus`，`OnBecomeActive` 自动恢复——**不需要你写代码**。这是父类已经做的工作。

但有一个坑——**如果你重写了 `OnBecomeInactive` 没调基类**，`last_focus` 不被记录，回来时焦点全丢。模式：

```lua
function MyScreen:OnBecomeInactive()
    MyScreen._base.OnBecomeInactive(self)  -- ★必须
    -- 自己的逻辑
end
```

#### 多 Tab 焦点流

复杂 Screen 常有多个 Tab，每个 Tab 内部一组按钮——**焦点流应该在 Tab 内循环**，不能跨 Tab 跳：

```lua
-- 在每个 Tab 切换时更新 default_focus
function MyScreen:SwitchTab(tab_index)
    self._current_tab = tab_index
    -- 重置焦点流：只在本 Tab 内的按钮间循环
    local btns = self.tabs[tab_index].buttons
    for i, b in ipairs(btns) do
        b:ClearFocusDirs()
        if i > 1 then b:SetFocusChangeDir(MOVE_LEFT, btns[i-1]) end
        if i < #btns then b:SetFocusChangeDir(MOVE_RIGHT, btns[i+1]) end
    end
    -- 焦点跳到第一个按钮
    btns[1]:SetFocus()
end
```

### 16.3.8 老手进阶：五个常见的 Screen 陷阱

#### 陷阱一：忘了 `Screen._base.OnControl(self, ...)`

```lua
-- ❌ 不调基类
function MyScreen:OnControl(control, down)
    if control == CONTROL_CANCEL and not down then
        self:Close()
        return true
    end
end
-- 后果：focus widget（如按钮）的点击不响应！因为 Screen 基类没机会分发到子 widget
```

`Widget:OnControl` 会**沿 widget 树向下传递**给焦点 widget——你跳过它，焦点 widget 永远收不到 CONTROL_ACCEPT。

**修复**：

```lua
function MyScreen:OnControl(control, down)
    if MyScreen._base.OnControl(self, control, down) then return true end
    -- 自定义逻辑
end
```

#### 陷阱二：`OnDestroy` 没释放周期任务

```lua
-- ❌ 任务永远跑下去
function MyScreen:Init()
    self._task = self.inst:DoStaticPeriodicTask(0.1, function()
        self:Tick()  -- screen 销毁后 self.inst 已无效
    end)
end
```

**症状**：Screen 关闭后日志报"AttemptToCallNilField"或更糟糕——内存泄漏。

**修复**：

```lua
function MyScreen:OnDestroy()
    if self._task then
        self._task:Cancel()
        self._task = nil
    end
    MyScreen._base.OnDestroy(self)
end
```

#### 陷阱三：`PushScreen` 没限制重复

```lua
-- ❌ 玩家狂按打开按钮
btn:SetOnClick(function()
    TheFrontEnd:PushScreen(MyScreen())  -- 每按一次推一个新的
end)
-- 玩家按 5 次 → 屏幕栈里 5 个相同 Screen 叠在一起
```

**修复**用 `GetOpenScreenOfType`：

```lua
btn:SetOnClick(function()
    if TheFrontEnd:GetOpenScreenOfType("MyScreen") == nil then
        TheFrontEnd:PushScreen(MyScreen())
    end
end)
```

或单例模式：

```lua
local _instance = nil
function MyScreen.Open()
    if _instance then return end
    _instance = MyScreen()
    TheFrontEnd:PushScreen(_instance)
end

function MyScreen:OnDestroy()
    _instance = nil
    MyScreen._base.OnDestroy(self)
end
```

#### 陷阱四：模态对话框忘加全屏吃点击层

```lua
-- ❌ 没有 self.black 那层
local MyDialog = Class(Screen, function(self)
    Screen._ctor(self, "MyDialog")
    self.proot = self:AddChild(Widget("proot"))
    -- ... 标题 + 按钮
end)
-- 后果：玩家可以点穿透到下层 Screen 的按钮
```

**修复**——参照 `PopupDialogScreen` 第 44-51 行：

```lua
self.black = self:AddChild(ImageButton("images/global.xml", "square.tex"))
self.black.image:SetVAnchor(ANCHOR_MIDDLE)
self.black.image:SetHAnchor(ANCHOR_MIDDLE)
self.black.image:SetScaleMode(SCALEMODE_FILLSCREEN)
self.black.image:SetTint(0,0,0,0)              -- 完全透明
self.black:SetOnClick(function() end)            -- 吃点击不做事
```

#### 陷阱五：在服务端（master）创建 Screen

```lua
-- ❌ 仅服务端运行
if TheWorld.ismastersim then
    TheFrontEnd:PushScreen(MyScreen())  -- 服务端无 UI！
end
```

**修复**：UI 永远只在客户端创建。如果要"服务端发消息让客户端开 Screen"——通过 RPC：

```lua
-- modmain.lua（共享）
AddModRPCHandler("MyMod", "OpenSettings", function(player)
    -- 客户端会收到这个，服务端不会触发本地 RPC
    TheFrontEnd:PushScreen(MySettingsScreen())
end)

-- 服务端发消息
SendModRPCToClient(GetClientModRPC("MyMod", "OpenSettings"), player.userid)
```

### 16.3.9 mod 实战：一个 mod 配置面板

把前面学到的整合起来——做一个"按 K 键打开 mod 设置面板"的小例子：

```lua
-- screens/mymodsettings.lua
local Screen = require "widgets/screen"
local Widget = require "widgets/widget"
local Image = require "widgets/image"
local ImageButton = require "widgets/imagebutton"
local Text = require "widgets/text"
local TEMPLATES = require "widgets/templates"

local MyModSettings = Class(Screen, function(self)
    Screen._ctor(self, "MyModSettings")

    self.black = self:AddChild(ImageButton("images/global.xml", "square.tex"))
    self.black.image:SetVAnchor(ANCHOR_MIDDLE)
    self.black.image:SetHAnchor(ANCHOR_MIDDLE)
    self.black.image:SetScaleMode(SCALEMODE_FILLSCREEN)
    self.black.image:SetTint(0,0,0,0.5)
    self.black:SetOnClick(function() end)

    self.proot = self:AddChild(Widget("proot"))
    self.proot:SetVAnchor(ANCHOR_MIDDLE)
    self.proot:SetHAnchor(ANCHOR_MIDDLE)
    self.proot:SetScaleMode(SCALEMODE_PROPORTIONAL)

    self.bg = self.proot:AddChild(Image("images/fepanels.xml", "wideframe.tex"))
    self.bg:SetScale(0.7, 0.7)

    self.title = self.proot:AddChild(Text(TITLEFONT, 50, "MyMod Settings"))
    self.title:SetPosition(0, 150)

    self.close_btn = self.proot:AddChild(
        ImageButton("images/frontend.xml", "button_long.tex"))
    self.close_btn:SetText("Close")
    self.close_btn:SetTextSize(30)
    self.close_btn:ForceImageSize(180, 60)
    self.close_btn:SetPosition(0, -150)
    self.close_btn:SetOnClick(function() self:Close() end)

    self.default_focus = self.close_btn
end)

function MyModSettings:OnControl(control, down)
    if MyModSettings._base.OnControl(self, control, down) then return true end
    if control == CONTROL_CANCEL and not down then
        self:Close()
        return true
    end
end

function MyModSettings:Close()
    TheFrontEnd:PopScreen(self)
end

return MyModSettings
```

modmain.lua 注册按 K 打开：

```lua
-- modmain.lua
local function OpenSettings()
    if TheFrontEnd:GetOpenScreenOfType("MyModSettings") == nil then
        TheFrontEnd:PushScreen(require("screens/mymodsettings")())
    end
end

GLOBAL.TheInput:AddKeyUpHandler(GLOBAL.KEY_K, OpenSettings)
```

按 K 弹出、按 ESC 或点 Close 关闭——一个完整的 mod 设置面板雏形。

### 16.3.10 小结

```
            ┌──────────────────────────┐
            │     TheFrontEnd          │
            │                          │
            │   screenstack:           │
            │   ┌──────────────────┐   │
            │   │ Screen N (top)   │ ← OnControl/OnRawKey
            │   ├──────────────────┤      OnBecomeActive
            │   │ Screen N-1       │   失活但渲染
            │   ├──────────────────┤
            │   │  ...             │
            │   ├──────────────────┤
            │   │ HUD / MainScreen │
            │   └──────────────────┘
            │                          │
            └──────────────────────────┘

            生命周期：
            PushScreen → OnBecomeInactive(老栈顶) → AddChild
              → OnBecomeActive(新) → SetDefaultFocus
            PopScreen  → OnBecomeInactive → OnDestroy → Kill
              → OnBecomeActive(下一栈顶，焦点恢复)
```

| 概念 | 速记 |
|------|------|
| **Screen 继承自 Widget** | 多了 `OnBecomeActive/Inactive`、`default_focus`、`handlers` |
| **`TheFrontEnd:PushScreen(s)`** | 入栈+显示+激活 |
| **`TheFrontEnd:PopScreen()`** | 弹栈顶+OnDestroy+焦点恢复给下面 |
| **`TheFrontEnd:GetActiveScreen()`** | 返回栈顶 |
| **`TheFrontEnd:GetOpenScreenOfType(name)`** | 检查是否已打开避免重复 |
| **`OnBecomeActive/Inactive`** | 必须调基类，否则 last_focus 丢 |
| **`OnControl(control, down)`** | 重写时先调 `_base.OnControl`，处理后 `return true` |
| **`OnRawKey` vs `OnControl`** | 物理键 vs 逻辑控制（被改键） |
| **`Fade` / `FadeToScreen` / `FadeBack`** | 黑/白/卷帘的过渡动画 |
| **`default_focus`** | Screen 显示时初始焦点 widget |
| **模态对话框** | 加全屏 ImageButton 吃点击 |
| **`PopupDialogScreen`** | Klei 提供的通用确认/选择弹窗 |

**新手核心三句**：Screen 是 Widget 的子类；`PushScreen`/`PopScreen` 进栈出栈；模仿 `PopupDialogScreen` 写最简对话框。

**进阶核心三句**：每个事件回调（`OnBecomeActive` / `OnBecomeInactive` / `OnDestroy` / `OnControl`）都**先调基类再写自己的逻辑**；`OnControl` 处理完返回 true 阻止冒泡；`Fade` 三类型（black/white/swipe）配 `FadeToScreen` / `FadeBack` 做剧情过渡。

**老手核心三句**：`default_focus` 决定手柄玩家进 Screen 时光标在哪；模态对话框必须加全屏 `ImageButton` 吃点击防穿透；`OnDestroy` 必须释放所有周期任务和事件监听器。

下一节（16.4）我们将讨论**HUD 控件：状态栏、Badge、物品栏扩展**——HUD 上玩家随时能看到的"血/精/饿"状态条、各种 Badge 通知图标、以及如何为 mod 角色或新机制扩展物品栏栏位的标准方法。


## 16.4 HUD 控件：状态栏、Badge、物品栏扩展

> **写在前面**——前几节我们学了 Widget、布局、Screen——这些是"普适 UI"的基础。但**饥荒的 HUD（玩家平视显示）有它独特的一套约定**：右下角永远是状态栏（生命/精神/饥饿），左上角是地图，底部中央是物品栏，圆形 Badge 用 UIAnim 而不是静态图……
>
> 这一节我们专门学 **HUD 上那些"游戏专用"的复合控件**——它们不是 Widget 系统的"基础组件"，而是 Klei 在 Widget 上**封装出来的、专门为游戏机制服务的高层组件**。学会它们能让你：
>
> - **修改/扩展角色状态栏**（比如给自定义角色加一个"魔力槽"）
> - **创建自定义 Badge**（比如季节进度、buff 持续时间、boss 阶段）
> - **扩展物品栏**（比如加一个"项链槽"或一个数码宝贝栏）
> - **理解为什么 HUD 上的图标会"脉动""闪烁""带数字"**
>
> **新手**先看 16.4.1-16.4.3——理解 `Badge` 是什么、它内部为什么用 `UIAnim` 而不是 `Image`、写一个最简自定义 Badge；**进阶读者**继续看 16.4.4-16.4.6，深入 `StatusDisplays` 的结构（`brain`/`stomach`/`heart` 三联体）、跟随玩家组件的 delta 事件实时更新、自定义 Badge 完整 mod 实战；**老手**直接看 16.4.7-16.4.8，掌握 `Inv` 物品栏扩展（自定义 `EQUIPSLOTS`）、通过 `AddClassPostConstruct` 注入 HUD、五个常见的 HUD 陷阱。

### 16.4.1 快速入门：Badge 是 HUD 的"圆形进度图标"

`Badge` 是饥荒所有 **圆形可视进度图标** 的基类——血/精/饿三个图标都继承自它。

打开 `scripts/widgets/badge.lua` 第 6-10 行：

```6:10:scripts\widgets\badge.lua
local Badge = Class(Widget, function(self, anim, owner, tint, iconbuild, circular_meter, use_clear_bg, dont_update_while_paused, bonustint)
    Widget._ctor(self, "Badge")
    self:UpdateWhilePaused(not dont_update_while_paused)
    self.owner = owner
```

构造参数（9 个但常用的就 4 个）：

| 参数 | 类型 | 说明 |
|------|------|------|
| `anim` | string/nil | 自定义动画的 build 名（nil 用默认 `status_meter`） |
| `owner` | entity | 拥有者（通常是玩家），用于事件监听 |
| `tint` | {r,g,b,a} | 进度条本身的染色（红色血、绿色精、橙色饥）|
| `iconbuild` | string | 中心图标的 build 名（如 `"status_health"`）|
| `circular_meter` | bool | 用圆形进度条而不是上升液面 |
| `use_clear_bg` | bool | 用透明背景而不是默认环 |
| `dont_update_while_paused` | bool | 暂停时不动画 |
| `bonustint` | {r,g,b,a} | 副进度条的颜色（如生命惩罚阴影）|

**`Badge` 内部由这些 UIAnim 组成**（第 16-95 行）：

```
Badge
├─ pulse (UIAnim) - 数值变化时的脉冲圆环（绿增/红减）
├─ warning (UIAnim) - 危险警告（持续脉冲红光）
├─ backing (UIAnim) - 背景圈（status_meter 的 bg anim）
├─ anim (UIAnim) - 主进度条（用 SetPercent 控制液面）
├─ anim_bonus (UIAnim) - 加成进度条（如装备加血上限）
├─ circleframe (UIAnim) - 外圈框 + 中心图标（OverrideSymbol 注入）
├─ underNumber (Widget) - 数字层下面的容器（额外特效叠层）
└─ num (Text) - 中心数字（鼠标悬停才显示）
```

**为什么用 `UIAnim` 而不是 `Image`**？因为状态栏需要"**液面上升下降的连续动画**" + "**脉冲特效**"。这些用 Spriter 动画做（`SetPercent("anim", 1-val)`）比拼贴图省力得多。

### 16.4.2 快速入门：`SetPercent` —— 改变进度

Badge 的核心 API 第 127-152 行：

```127:152:scripts\widgets\badge.lua
function Badge:SetPercent(val, max, bonusval)
    val = val or self.percent
    max = max or 100

    if self.circular_meter ~= nil then
        self.circular_meter:GetAnimState():SetPercent("meter", val)
    else
        self.anim:GetAnimState():SetPercent("anim", 1 - val)
        if self.circleframe ~= nil and not self.dont_animate_circleframe then
            self.circleframe:GetAnimState():SetPercent("frame", 1 -val)
        end
    end

    if self.anim_bonus then
        if bonusval then
            self.anim_bonus:GetAnimState():SetPercent("anim", 1 - bonusval)
            self.anim_bonus:Show()
        else
            self.anim_bonus:Hide()
        end
    end

    self.num:SetString(tostring(math.ceil(val * max)))
    self.percent = bonusval or val
end
```

**关键细节**：

- **`val` 是百分比 `[0, 1]`** —— `SetPercent(0.5)` 表示半满。
- **`max` 用于显示数字** —— `SetPercent(0.5, 200)` → 中心显示 `100`（=0.5×200）。
- **`bonusval` 用于"加成上限"** —— 比如装备 +20% 血上限，bonus 多出来那一段单独着色。
- **注意 `1-val`** —— 引擎用"下降的百分比"驱动动画（液面是从满到空的，所以传 `1-val` 让动画从 100% 收缩到 val%）。

**完整实战**——监听血量变化更新 badge：

```lua
local heart = HealthBadge(player)
heart:SetPosition(40, 20)

-- 实时更新
player.inst:ListenForEvent("healthdelta", function(inst, data)
    heart:SetPercent(data.newpercent, player.replica.health:Max())
    if data.newpercent < data.oldpercent then
        heart:PulseRed()    -- 受伤时红色脉冲
    elseif data.newpercent > data.oldpercent then
        heart:PulseGreen()  -- 回血时绿色脉冲
    end
end)

-- 危险警告（血 < 25%）
player.inst:ListenForEvent("healthdelta", function(inst, data)
    if data.newpercent < 0.25 then
        heart:StartWarning()  -- 持续闪红
    else
        heart:StopWarning()
    end
end)
```

`PulseGreen` / `PulseRed` / `StartWarning` / `StopWarning` 是 Badge 提供的"反馈动画 API"——见 `badge.lua` 第 163-216 行。

### 16.4.3 快速入门：写一个最简自定义 Badge

假设我们做一个 mod 角色"月光术士"，需要一个**月光蓄能进度条**——简单的 0-100 进度。

```lua
-- widgets/moonpowerbadge.lua
local Badge = require "widgets/badge"

local MoonPowerBadge = Class(Badge, function(self, owner)
    -- 调基类：用默认 status_meter 动画 + 蓝色染色 + 自定义 icon
    Badge._ctor(self, nil, owner,
        {0.5, 0.6, 1.0, 1},        -- 蓝白色染色
        "status_health",            -- 用现有 build 当 icon（占位）
        nil,                         -- 不用圆形进度（用液面）
        nil, nil)
end)

function MoonPowerBadge:UpdatePower(percent, max)
    self:SetPercent(percent, max)
    if percent > 0.99 then
        self:PulseGreen()             -- 满了绿色提示
    end
end

return MoonPowerBadge
```

挂到 HUD（通过 `AddClassPostConstruct` 注入 StatusDisplays）：

```lua
-- modmain.lua
AddClassPostConstruct("widgets/statusdisplays", function(self)
    if not self.owner:HasTag("moonsigil") then return end

    local MoonPowerBadge = require "widgets/moonpowerbadge"
    self.moonpower = self:AddChild(MoonPowerBadge(self.owner))
    self.moonpower:SetPosition(self.column1, 20, 0)  -- 借用 column1（最左列）

    -- 监听 mod 自定义事件
    self.inst:ListenForEvent("moonpowerdelta", function(owner, data)
        self.moonpower:UpdatePower(data.newpercent, 100)
    end, self.owner)
end)
```

**新手要记住的三件事**：

1. **Badge 默认有现成的 status_meter 动画** —— 不传 `anim` 参数就用默认，省事。
2. **`tint` 决定 badge 的整体颜色** —— RGBA 元组，0-1 浮点。
3. **`SetPercent(val, max)` 既改液面也改数字** —— 不要分两次调。

### 16.4.4 进阶：`StatusDisplays` —— 三联体生命/精神/饥饿

`scripts/widgets/statusdisplays.lua` 是**所有 HUD 状态栏的容器**。它在 `controls.lua` 第 153 行被加到 HUD 的右下角：

```153:153:scripts\widgets\controls.lua
                self.status = self.topleft_root:AddChild(StatusDisplays(self.owner))
```

> 注：变量名 `topleft_root` 是 Klei 的代码遗留——实际位置在屏幕**右下**（看 controls.lua 顶部对锚点设置）。

**StatusDisplays 内部就是一个 Widget**——它用 `column1..column5` 几个数字布置 5 列三联体。第 295-308 行：

```295:308:scripts\widgets\statusdisplays.lua
    local is_splitscreen = IsSplitScreen()
    if is_splitscreen and IsGameInstance(Instances.Player1) then
        self.column1 = 80
        self.column2 = 40
        self.column3 = 0
        self.column4 = -40
        self.column5 = 120
    else
        self.column1 = -80
        self.column2 = -40
        self.column3 = 0
        self.column4 = 40
        self.column5 = -120
    end
```

#### 三联体布局

```
PC 单屏（默认）：
                  HUD 右下角
                  ┌────────────────┐
                  │   stomach      │ column2 = -40
                  │   ●     heart  │ column4 = +40
                  │  brain          │ column3 = 0
                  │  moisturemeter │
                  │  boatmeter     │ column1 = -80
                  └────────────────┘
```

实际创建（第 316-340 行）：

```316:340:scripts\widgets\statusdisplays.lua
    self.brain = self:AddChild(owner.CreateSanityBadge ~= nil and owner.CreateSanityBadge(owner) or SanityBadge(owner))
    self.brain:SetPosition(self.column3, -40, 0)

    self.stomach = self:AddChild(owner.CreateHungerBadge ~= nil and owner.CreateHungerBadge(owner) or HungerBadge(owner))
    self.stomach:SetPosition(self.column2, 20, 0)

    self.heart = self:AddChild(owner.CreateHealthBadge ~= nil and owner.CreateHealthBadge(owner) or HealthBadge(owner, nil, "status_abigail"))
    self.heart:SetPosition(self.column4, 20, 0)

    self.moisturemeter = self:AddChild(owner.CreateMoistureMeter ~= nil and owner.CreateMoistureMeter(owner) or MoistureMeter(owner))
    self.moisturemeter:SetPosition(self.column3, -115, 0)

    self.boatmeter = self:AddChild(BoatMeter(owner))
    self.boatmeter:SetPosition(self.column1, -40, 0)
```

#### 角色覆写——`CreateHealthBadge` 模式

注意上面那一行——`owner.CreateSanityBadge ~= nil and owner.CreateSanityBadge(owner) or SanityBadge(owner)`。

这是**Klei 留给角色 mod 的"覆写口"**。如果你的 prefab 上挂了 `inst.CreateHealthBadge = MyCustomHealthBadge` 函数，StatusDisplays 会用你的版本而不是默认 HealthBadge。

**官方例子**：Wanda 用 `WandaAgeBadge` 替换标准血条。`scripts/prefabs/wanda.lua` 第 384-386 行：

```384:386:scripts\prefabs\wanda.lua
		if not TheNet:IsDedicated() then
			inst.CreateHealthBadge = WandaAgeBadge
		end
```

**mod 应用**——你做了一个"机械人"角色，血量显示成"能量百分比"环：

```lua
-- prefabs/myrobot.lua
local function common_postinit(inst)
    if not TheNet:IsDedicated() then
        inst.CreateHealthBadge = function(owner)
            return require("widgets/robotenergybadge")(owner)
        end
    end
end
```

#### Delta 事件订阅

`StatusDisplays` 在 `OnSetPlayerMode`（第 16-46 行）订阅一堆 delta 事件：

```24:39:scripts\widgets\statusdisplays.lua
    if self.onhealthdelta == nil then
        self.onhealthdelta = function(owner, data) self:HealthDelta(data) end
        self.inst:ListenForEvent("healthdelta", self.onhealthdelta, self.owner)
        self:SetHealthPercent(self.owner.replica.health:GetPercent())
    end

    if self.onhungerdelta == nil then
        self.onhungerdelta = function(owner, data) self:HungerDelta(data) end
        self.inst:ListenForEvent("hungerdelta", self.onhungerdelta, self.owner)
        self:SetHungerPercent(self.owner.replica.hunger:GetPercent())
    end

    if self.onsanitydelta == nil then
        self.onsanitydelta = function(owner, data) self:SanityDelta(data) end
        self.inst:ListenForEvent("sanitydelta", self.onsanitydelta, self.owner)
        self:SetSanityPercent(self.owner.replica.sanity:GetPercent())
    end
```

这是**HUD 实时更新的标准模式**——监听玩家 prefab 上的 `xxxdelta` 事件，事件 data 里有 `newpercent` / `oldpercent`，把它转给 Badge 的 `SetPercent`。

**事件清单**：

| 事件 | 触发时机 | data |
|------|---------|------|
| `healthdelta` | `Health:DoDelta` | `{newpercent, oldpercent, amount, cause}` |
| `hungerdelta` | `Hunger:DoDelta` | `{newpercent, oldpercent}` |
| `sanitydelta` | `Sanity:DoDelta` | `{newpercent, oldpercent}` |
| `moisturedelta` | 玩家潮湿度变化 | `{new, old}` |
| `werenessdelta` | 狼人化进度（Woodie） | `{newpercent, oldpercent}` |
| `inspirationdelta` | Wigfrid 的灵感 | `{newpercent, slots_available}` |
| `mightinessdelta` | Wolfgang 的力量 | `{newpercent}` |

**mod 自定义状态值**——发自定义 delta 事件让你的 Badge 监听：

```lua
-- prefabs/myrobot.lua
function ChargePower(inst, amount)
    local oldp = inst.energy / inst.maxenergy
    inst.energy = math.min(inst.energy + amount, inst.maxenergy)
    local newp = inst.energy / inst.maxenergy
    inst:PushEvent("energydelta", {newpercent = newp, oldpercent = oldp})
end
```

### 16.4.5 进阶：Pulse / Warning 反馈动画

`Badge` 的 `PulseGreen` / `PulseRed` / `StartWarning` / `StopWarning` 让数值变化"看得见、感受到"。**HUD 设计的核心原则之一——所有数值变化都要有视觉反馈**。

**Pulse 一次性闪烁**（受到伤害、回血等）：

```163:175:scripts\widgets\badge.lua
function Badge:PulseGreen()
    self.pulse:GetAnimState():SetMultColour(0, 1, 0, 1)
    self.pulse:GetAnimState():PlayAnimation("pulse")

    if self.warning.shown then
        self.warning:Hide()
    end

    if self.warningdelaytask ~= nil then
        self.warningdelaytask:Cancel()
    end
    self.warningdelaytask = self.inst:DoTaskInTime(2 * self.pulse:GetAnimState():GetCurrentAnimationLength(), CheckWarning, self)
end
```

注意 **Pulse 暂时打断 Warning**——脉冲动画播完之后再恢复警告（通过 `warningdelaytask`）。

**Warning 持续脉冲**（血/精/饿低于警戒线）：

```201:216:scripts\widgets\badge.lua
function Badge:StartWarning(r, g, b, a)
    if r == nil or g == nil or b == nil or a == nil then
        r, g, b, a = 1, 0, 0, 1
    end
    self.warning:GetAnimState():SetMultColour(r, g, b, a)

    if not self.warningstarted then
        self.warningstarted = true

        if self.warningdelaytask == nil and not self.warning.shown then
            self.warning:Show()
            self.warning:GetAnimState():PlayAnimation("pulse", true)  -- 循环播放
        end
    end
end
```

**StatusDisplays 自动判断警戒线**——以 `HealthDelta` 为例（第 641-668 行的逻辑里），当 `newpercent < SOMETHRESHOLD` 时自动调 `heart:StartWarning`。

**实战**：

```lua
function MoonPowerBadge:OnDelta(data)
    self:SetPercent(data.newpercent, 100)
    if data.newpercent > data.oldpercent then
        self:PulseGreen()
    elseif data.newpercent < data.oldpercent then
        self:PulseRed()
    end

    if data.newpercent < 0.1 then
        self:StartWarning(0.5, 0.6, 1.0, 1)  -- 蓝色警告
    else
        self:StopWarning()
    end
end
```

### 16.4.6 进阶：完整 mod 实战——给自定义角色加"魔力槽"

把前面学到的整合起来。需求：mod 角色"月光术士"有一个 0-100 的魔力值（mana），用 HUD 上的 badge 显示。

#### 第一步：玩家组件

```lua
-- components/manapool.lua
local ManaPool = Class(function(self, inst)
    self.inst = inst
    self.current = 100
    self.max = 100
end)

function ManaPool:DoDelta(amount)
    local oldp = self.current / self.max
    self.current = math.clamp(self.current + amount, 0, self.max)
    local newp = self.current / self.max
    self.inst:PushEvent("manadelta", {
        newpercent = newp, oldpercent = oldp,
        amount = amount,
    })
end

function ManaPool:GetPercent() return self.current / self.max end

return ManaPool
```

#### 第二步：自定义 Badge

```lua
-- widgets/manabadge.lua
local Badge = require "widgets/badge"

local MANA_TINT = {0.5, 0.7, 1.0, 1}  -- 蓝色

local ManaBadge = Class(Badge, function(self, owner)
    Badge._ctor(self, nil, owner, MANA_TINT, "status_health", false, false, false)
    self.warninglevel = 0.2
end)

function ManaBadge:OnManaDelta(data)
    self:SetPercent(data.newpercent, 100)

    if data.amount > 0 then
        self:PulseGreen()
    elseif data.amount < 0 then
        self:PulseRed()
    end

    if data.newpercent < self.warninglevel then
        self:StartWarning(MANA_TINT[1], MANA_TINT[2], MANA_TINT[3], 1)
    else
        self:StopWarning()
    end
end

return ManaBadge
```

#### 第三步：注入 StatusDisplays

```lua
-- modmain.lua
AddClassPostConstruct("widgets/statusdisplays", function(self)
    -- 只有标记 "manauser" 的角色才有
    if not self.owner:HasTag("manauser") then return end

    local ManaBadge = require "widgets/manabadge"
    self.manabadge = self:AddChild(ManaBadge(self.owner))
    self.manabadge:SetPosition(self.column1, 20, 0)  -- 在 stomach 左侧

    -- 初始值
    if self.owner.replica.manapool then
        self.manabadge:SetPercent(self.owner.replica.manapool:GetPercent(), 100)
    end

    -- 订阅 delta
    self.onmanadelta = function(owner, data)
        self.manabadge:OnManaDelta(data)
    end
    self.inst:ListenForEvent("manadelta", self.onmanadelta, self.owner)
end)

-- 玩家 prefab 加 tag
AddPrefabPostInit("moonsigil_player", function(inst)
    inst:AddTag("manauser")
    if not TheNet:IsDedicated() then
        if not inst.components.manapool then
            inst:AddComponent("manapool")
        end
    end
end)
```

#### 调用方式

```lua
-- 法术消耗 mana
ThePlayer.components.manapool:DoDelta(-20)
-- HUD 上 mana 槽自动闪红、播脉冲、扣数字
```

### 16.4.7 老手进阶：物品栏扩展（自定义 EQUIPSLOT）

`scripts/constants.lua` 第 650-656 行定义了**官方四个装备槽**：

```650:656:scripts\constants.lua
EQUIPSLOTS =
{
    HANDS = "hands",
    HEAD = "head",
    BODY = "body",
    BEARD = "beard",
}
```

mod 可以**新增 EQUIPSLOT**——比如做一个项链栏、戒指栏、披风栏。

#### 第一步：扩展常量表

```lua
-- modmain.lua
GLOBAL.EQUIPSLOTS.NECK = "neck"
GLOBAL.EQUIPSLOT_IDS = nil  -- 让引擎重建 ID 表
```

> **注意**——直接改全局表通常有风险，**Klei 提供了更稳的途径**：用 `AddInventoryClassPostConstruct` 让 inventory 组件认识新槽（部分高级 mod 会改 EquipSlots，但兼容性差，本节略）。

#### 第二步：物品声明 equipslot

```lua
-- prefabs/moonpendant.lua
local function fn()
    local inst = CreateEntity()
    -- ... 常规 prefab 设置 ...

    inst:AddComponent("equippable")
    inst.components.equippable.equipslot = EQUIPSLOTS.NECK   -- ★

    return inst
end
```

#### 第三步：在 inventory bar 加槽

`scripts/widgets/inventorybar.lua` 第 150-155 行的 `AddEquipSlot`：

```150:155:scripts\widgets\inventorybar.lua
function Inv:AddEquipSlot(slot, atlas, image, sortkey)
    sortkey = sortkey or #self.equipslotinfo
    table.insert(self.equipslotinfo, {slot = slot, atlas = atlas, image = image, sortkey = sortkey})
    table.sort(self.equipslotinfo, function(a,b) return a.sortkey < b.sortkey end)
    self.rebuild_pending = true
end
```

mod 调用：

```lua
-- modmain.lua
AddClassPostConstruct("widgets/inventorybar", function(self)
    self:AddEquipSlot(EQUIPSLOTS.NECK,
        "images/mymod/equip_slots.xml",  -- 你 mod 的 atlas
        "equip_slot_neck.tex",            -- 槽位空时显示的图
        2)                                 -- 排序键（0=hands, 1=body, ...）
end)
```

`sortkey` 决定槽位**显示顺序**——和默认槽（hands=0、body=1、head=2 之类）插值即可。

#### 第四步：装备同步

服务端 `Equippable` 组件会自动通过网络同步装备状态——**前端不需要写代码**。`Inv:OnItemEquip` 监听 `"equip"` 事件自动刷新槽位贴图。

### 16.4.8 老手进阶：通过 `AddClassPostConstruct` 注入 HUD

#### `AddClassPostConstruct` 是什么

mod 想给已有 widget 添加东西——直接改源码不可能。Klei 提供了 **类构造后钩子**：

```lua
AddClassPostConstruct("widgets/statusdisplays", function(self)
    -- 这里 self 是 StatusDisplays 实例，已经初始化完毕
    -- 可以 self:AddChild(...)、订阅 self.owner 的事件
end)
```

**钩子在 `Class()` 返回的最后一刻被调用**——比 `_ctor` 晚但比第一次使用早。

**常用注入点**：

| 注入目标 | 用途 |
|---------|------|
| `widgets/statusdisplays` | 在状态栏加自定义 badge |
| `widgets/controls` | 在 HUD 上加新控件（按钮、面板）|
| `widgets/inventorybar` | 加自定义装备槽 |
| `widgets/mapwidget` | 给地图加自定义图标 |
| `screens/playerhud` | 加全屏遮罩、特效层 |

#### 实战：在 HUD 加一个"季节进度环"

```lua
-- modmain.lua
AddClassPostConstruct("widgets/controls", function(self)
    if not TheWorld.ismastersim and TheWorld.state.season then
        local SeasonBadge = require("widgets/seasonbadge")
        self.seasonbadge = self:AddChild(SeasonBadge())
        self.seasonbadge:SetHAnchor(ANCHOR_LEFT)
        self.seasonbadge:SetVAnchor(ANCHOR_TOP)
        self.seasonbadge:SetPosition(80, -80)
    end
end)
```

#### 移除 Klei 默认 HUD 元素

有些 mod 想**隐藏官方控件**（比如做"极简 HUD"）：

```lua
AddClassPostConstruct("widgets/statusdisplays", function(self)
    self.boatmeter:Hide()  -- 隐藏船血条
    self.moisturemeter:Hide()  -- 隐藏潮湿度
end)
```

**注意**：`Hide` 比直接 `Kill` 安全——保留对象但不显示，不破坏 Klei 内部其他代码引用。

### 16.4.9 老手进阶：五个常见的 HUD 陷阱

#### 陷阱一：在 `_ctor` 阶段访问 `replica` 组件

```lua
-- ❌ 构造时 replica 还没同步
AddClassPostConstruct("widgets/statusdisplays", function(self)
    local val = self.owner.replica.manapool:GetPercent()  -- 可能是 nil 报错
end)
```

**修复**——先检查 nil 或在网络同步事件里再取：

```lua
AddClassPostConstruct("widgets/statusdisplays", function(self)
    if self.owner.replica.manapool ~= nil then
        local val = self.owner.replica.manapool:GetPercent()
        ...
    end

    -- 或监听 dirty 事件
    self.inst:ListenForEvent("manaclassifieddirty", function()
        ...
    end, self.owner)
end)
```

#### 陷阱二：HUD widget 不监听 `playerentered`

```lua
-- 玩家死亡复活 / 重连 → HUD 重建
-- 你的自定义 badge 应该重建，否则状态错乱
```

**修复**——监听 `playerdeactivated` 释放、`playerentered` 重建。或更稳——直接用 `AddClassPostConstruct` 的 self 作为唯一入口（每次重建会重新触发）。

#### 陷阱三：多人服务器忘记区分客户端/服务端

```lua
-- ❌ AddPrefabPostInit 在服务端也跑，但服务端没有 HUD
AddPrefabPostInit("wilson", function(inst)
    inst.HUD:AddBadge(...)  -- 服务端 inst.HUD = nil
end)
```

**修复**：

```lua
AddPrefabPostInit("wilson", function(inst)
    if not TheWorld.ismastersim or inst.HUD then  -- 客户端检测
        ...
    end
end)
```

或者**永远在 `AddClassPostConstruct("widgets/statusdisplays", ...)` 里**——这个钩子只在客户端触发。

#### 陷阱四：`column1..5` 自动布局会冲突

```lua
-- ❌ 两个 mod 都把自己的 badge 放 column1
-- 结果一个盖住另一个
self.mybadge:SetPosition(self.column1, 20, 0)
```

**修复**——用相对偏移而不是直接占用 column：

```lua
-- 在已有 column1 的左边一格
self.mybadge:SetPosition(self.column1 - 60, 20, 0)
```

或检查是否已被占用：

```lua
local x_offset = -60  -- 默认在最左
for _, child in pairs(self.children) do
    if child.is_mod_badge then
        x_offset = x_offset - 60   -- 已有 mod badge → 再左移
    end
end
self.mybadge.is_mod_badge = true
self.mybadge:SetPosition(self.column1 + x_offset, 20, 0)
```

#### 陷阱五：`StartWarning` 没对应 `StopWarning`

```lua
-- ❌ 一旦 StartWarning，再也不停
function MyBadge:OnDelta(data)
    if data.newpercent < 0.2 then
        self:StartWarning()
    end
    -- 大于 0.2 时没有 stop
end
```

**症状**：玩家从低血量回到健康，但 HUD 仍在警告闪烁。

**修复**——总是配对：

```lua
function MyBadge:OnDelta(data)
    if data.newpercent < 0.2 then
        self:StartWarning()
    else
        self:StopWarning()    -- ★关键
    end
end
```

### 16.4.10 小结

```
                   PlayerHUD（屏幕级）
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
          Controls               Overlays
         （HUD 控件根）          （遮罩层）
              │
    ┌─────────┼─────────┬─────────────┐
    ▼         ▼         ▼             ▼
 status     inv     hudcompass   minimap...
（状态栏） （物品栏）  （指南针）
    │         │
 三联体      InvSlot/EquipSlot
    │
 ┌──┼──┬──────┬──────┬───────┐
 ▼  ▼  ▼      ▼      ▼       ▼
heart stomach brain moisture boat ...
（血）（饿）（精）（湿）（船）

     每个 = Badge 子类
     每个 Badge = pulse + warning + backing + anim
                  + circleframe + num
```

| 概念 | 速记 |
|------|------|
| **`Badge` 是 HUD 圆形进度的基类** | 内含 pulse / warning / backing / anim / num |
| **`SetPercent(val, max)`** | 改液面 + 改数字（val 是 0-1）|
| **`PulseGreen / PulseRed`** | 一次性闪烁（数值变化）|
| **`StartWarning / StopWarning`** | 持续脉冲（危险线）|
| **`StatusDisplays` 三联体** | brain / stomach / heart + moisturemeter / boatmeter |
| **`column1..5`** | 5 列布局，column3 = 0（中央）|
| **`owner.CreateXxxBadge`** | 角色 prefab 覆写默认 Badge（Wanda 模式）|
| **`xxxdelta` 事件** | 状态变化的标准推送方式 |
| **`AddClassPostConstruct(widget_path, fn)`** | 在 widget 构造后注入 mod 代码 |
| **`Inv:AddEquipSlot(slot, atlas, image, sortkey)`** | 添加自定义装备槽 |
| **`EQUIPSLOTS`** | hands / head / body / beard 四个官方槽 |
| **`UIAnim`** | HUD 用动画而非静态图——能 SetPercent / Pulse |

**新手核心三句**：HUD 上圆形进度图标都是 Badge；Badge 用 `SetPercent` 同时改液面和数字；监听玩家的 `xxxdelta` 事件实时更新。

**进阶核心三句**：StatusDisplays 用 `column1..5` 五列布置三联体；用 `owner.CreateHealthBadge` 钩子让 mod 角色覆写默认 badge；Pulse / Warning 是 HUD 视觉反馈的两大支柱。

**老手核心三句**：`AddClassPostConstruct("widgets/statusdisplays", ...)` 是 mod 注入 HUD 的正解；`AddEquipSlot(slot, atlas, image, sortkey)` 加新装备槽；多 mod 共存时不要直接占用 column，用相对偏移。

下一节（16.5）我们将讨论**Container Widget**——自定义容器界面（背包、宝箱、烹饪锅等"打开后弹出网格"的界面），如何为 mod 容器（比如新箱子、新工作站）做一个完整的容器 UI。


## 16.5 Container Widget——自定义容器界面

> **写在前面**——除了玩家自身的物品栏，饥荒里**任何"打开后能存放物品的实体"都用统一的容器系统**：背包、宝箱、冰箱、烹饪锅、晾肉架、行李、调味罐……它们的"打开后弹出网格界面"看起来千差万别，但其实**全部走同一套机制**：
>
> ```
> 实体 (cookpot/chester/backpack)
>     ├─ Container 组件（服务端，存放真实数据）
>     ├─ Container_replica（客户端，网络同步副本）
>     └─ ContainerWidget（客户端，弹出的 UI）
> ```
>
> 真正决定"长什么样"的是 `scripts/containers.lua` 里的 **`params.<prefab>.widget`** 配置块——`slotpos` 决定槽位坐标、`animbank/animbuild` 决定外观皮肤、`buttoninfo` 决定底部按钮、`itemtestfn` 决定能放什么。**学会这套配置就能给 mod 容器做任意 UI**。
>
> **新手**先看 16.5.1-16.5.3——理解容器三件套（Container/Replica/Widget）、看 cookpot/icebox 的标准 params 配置、用 30 行代码做一个 mod 宝箱；**进阶读者**继续看 16.5.4-16.5.6，深入 `slotpos` 网格布局算法、`itemtestfn` 物品过滤、`buttoninfo`（烹饪按钮）+ 进度动画 `animfn` / `animloop`；**老手**直接看 16.5.7-16.5.8，掌握**侧面板容器**（背包跟随玩家）、运行时切换 widget（精灵球 / 模式背包）、五个常见的容器陷阱。

### 16.5.1 快速入门：容器系统的三件套

打开任意一个有容器的 prefab——`scripts/prefabs/cookpot.lua` 第 317-322 行：

```317:322:scripts\prefabs\cookpot.lua
        inst:AddComponent("container")
        --inst.components.container:WidgetSetup("cookpot")
        inst.components.container.onopenfn = onopen
        inst.components.container.onclosefn = onclose
        inst.components.container.skipclosesnd = true
        inst.components.container.skipopensnd = true
```

注意——**这里只挂了 `container` 组件，没显式调 `WidgetSetup`**。原因是引擎在组件构造时**自动用 prefab 名查 `containers.params[prefab]`**。

容器系统三件套：

| 组件 | 端 | 职责 |
|------|----|------|
| **`Container`**（`scripts/components/container.lua`） | 仅服务端 | 存放真实物品数据、`GiveItem`/`GetItemInSlot`、网络授权 |
| **`Container_replica`**（`scripts/components/container_replica.lua`） | 客户端镜像 | 网络同步、客户端 UI 读它、`GetWidget()` 返回 widget 配置 |
| **`ContainerWidget`**（`scripts/widgets/containerwidget.lua`） | 仅客户端 | 弹出的网格界面本身——`Open` 创建槽位、`Close` 销毁 |

**打开容器的完整流程**：

```
玩家右键宝箱
    ↓
PlayerController:OpenChest → BufferedAction(ACTIONS.RUMMAGE)
    ↓
服务端：Container:Open(player) → 推送 "container_opened" 事件
    ↓
客户端 Container_replica 收到 → 调 PlayerHud:OpenContainer
    ↓
PlayerHud 创建 ContainerWidget
    ↓
ContainerWidget:Open(container, doer) 读取 params.<prefab>.widget 配置
    ↓
按 widget.slotpos 创建一组 InvSlot 子 widget
    ↓
按 widget.animbank/animbuild 播放打开动画
    ↓
监听 "itemget"/"itemlose"/"refresh" 事件实时更新槽位
```

新手只需记住：**Container 是数据，ContainerWidget 是界面**。**配置在 `containers.params.<prefab>.widget`**。

### 16.5.2 快速入门：看 cookpot/icebox 的标准配置

打开 `scripts/containers.lua` 第 348-371 行——烹饪锅的完整配置：

```348:371:scripts\containers.lua
params.cookpot =
{
    widget =
    {
        slotpos =
        {
            Vector3(0, 64 + 32 + 8 + 4, 0),
            Vector3(0, 32 + 4, 0),
            Vector3(0, -(32 + 4), 0),
            Vector3(0, -(64 + 32 + 8 + 4), 0),
        },
        animbank = "ui_cookpot_1x4",
        animbuild = "ui_cookpot_1x4",
        pos = Vector3(200, 0, 0),
        side_align_tip = 100,
        buttoninfo =
        {
            text = STRINGS.ACTIONS.COOK,
            position = Vector3(0, -165, 0),
        }
    },
    acceptsstacks = false,
    type = "cooker",
}
```

**逐字段解读**：

| 字段 | 作用 |
|------|------|
| `widget.slotpos` | **数组**——每个 `Vector3` 是一个槽位的坐标（相对 widget 中心）|
| `widget.animbank` / `animbuild` | **背景动画的 bank / build 名**（必须存在 anim/zip）|
| `widget.pos` | widget 整体相对屏幕中心的位置 |
| `widget.side_align_tip` | 控件向边沿对齐时的偏移（手柄玩家 UI）|
| `widget.buttoninfo` | 底部按钮（`text` + `position` + 可选 `fn` 回调）|
| `acceptsstacks` | 是否允许同槽位堆叠 |
| `type` | "cooker"/"chest"/"pack"/"hand_inv"/"top_rack" 等——影响打开行为 |

#### 冰箱配置（3×3 网格）

`containers.lua` 第 1000-1041 行：

```1000:1017:scripts\containers.lua
params.icebox =
{
    widget =
    {
        slotpos = {},
        animbank = "ui_chest_3x3",
        animbuild = "ui_chest_3x3",
        pos = Vector3(0, 200, 0),
        side_align_tip = 160,
    },
    type = "chest",
}

for y = 2, 0, -1 do
    for x = 0, 2 do
        table.insert(params.icebox.widget.slotpos, Vector3(80 * x - 80 * 2 + 80, 80 * y - 80 * 2 + 80, 0))
    end
end
```

**注意"循环生成槽位坐标"**——9 个 80px 间距的格子。这是 mod 制作 NxM 网格的标准模式。

加上 **`itemtestfn`** 让冰箱**只接受食物**（第 1019-1041 行）：

```1019:1041:scripts\containers.lua
function params.icebox.itemtestfn(container, item, slot)
    if item:HasTag("icebox_valid") then
        return true
    end

    if not (item:HasTag("fresh") or item:HasTag("stale") or item:HasTag("spoiled")) then
        return false
    end

	if item:HasTag("smallcreature") then
		return false
	end

    for k, v in pairs(FOODTYPE) do
        if item:HasTag("edible_"..v) then
            return true
        end
    end

    return false
end
```

**`itemtestfn` 在客户端和服务端都跑**——服务端用它阻止非法物品进入，客户端用它在拖拽时显示"红色禁止图标"。**返回 `true` 允许、`false` 拒绝**。

### 16.5.3 快速入门：30 行代码做一个 mod 宝箱

需求：mod 加一个"魔法宝箱"，6 槽（2 行 3 列），只接受带 `magic` 标签的物品。

**第一步——prefab**：

```lua
-- prefabs/magicchest.lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    inst.AnimState:SetBank("chest")
    inst.AnimState:SetBuild("treasure_chest")
    inst.AnimState:PlayAnimation("closed")

    MakeObstaclePhysics(inst, .3)

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inspectable")
    inst:AddComponent("container")
    inst.components.container:WidgetSetup("magicchest")  -- ★关键
    inst:AddComponent("workable")
    inst.components.workable:SetWorkAction(ACTIONS.HAMMER)
    inst.components.workable:SetWorkLeft(4)
    inst.components.workable:SetOnFinishCallback(function(inst, worker)
        inst.components.lootdropper:DropLoot()
        inst:Remove()
    end)
    return inst
end

return Prefab("magicchest", fn, {Asset("ANIM", "anim/treasure_chest.zip")})
```

**第二步——容器配置**（modmain.lua）：

```lua
-- modmain.lua
local containers = GLOBAL.require("containers")

containers.params.magicchest = {
    widget = {
        slotpos = {},
        animbank = "ui_chest_3x3",        -- 复用官方 3x3 宝箱外观
        animbuild = "ui_chest_3x3",
        pos = GLOBAL.Vector3(0, 200, 0),
    },
    type = "chest",
}

-- 6 个槽位（2 行 3 列）
for y = 1, 0, -1 do
    for x = 0, 2 do
        table.insert(containers.params.magicchest.widget.slotpos,
            GLOBAL.Vector3(80 * x - 80, 80 * y - 40, 0))
    end
end

-- 只接受 "magic" 标签
containers.params.magicchest.itemtestfn = function(container, item, slot)
    return item:HasTag("magic")
end
```

**第三步——注册 prefab**：

```lua
-- modmain.lua（接续）
PrefabFiles = {"magicchest"}

-- 让物品掉落表里能出现
AddPrefabPostInit("forest", function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end
    -- 测试用：在世界里生成
    inst:DoTaskInTime(2, function()
        GLOBAL.SpawnPrefab("magicchest")
    end)
end)
```

测试：玩家右键这个魔法宝箱，弹出 6 槽窗口，只接受 magic 物品。**完整流程 30 行代码**。

### 16.5.4 进阶：网格布局公式

观察官方 `slotpos` 生成代码——**循环 + 偏移**是标准模式。

#### 标准 NxM 网格

```lua
-- N 行 × M 列，槽位间距 80
local SPACING = 80
local cols = 3
local rows = 3

for y = rows - 1, 0, -1 do
    for x = 0, cols - 1 do
        table.insert(slotpos, Vector3(
            SPACING * x - SPACING * (cols - 1) / 2,
            SPACING * y - SPACING * (rows - 1) / 2,
            0))
    end
end
```

**注意 y 倒序**——从顶到底——因为 UI 坐标 y 向上为正，槽位编号通常从上到下。

**官方常用间距**：

| 容器 | 槽位间距 | 槽位尺寸 |
|------|---------|---------|
| 3×3 宝箱 | 80px | 75px |
| 1×4 烹饪锅 | 76px | 64px |
| 2×4 背包 | 75px | 70px |
| 3×4 暗影箱 | 75px | 70px |

#### 单列（烹饪锅式）

```lua
slotpos = {
    Vector3(0, 64 + 32 + 8 + 4, 0),
    Vector3(0, 32 + 4, 0),
    Vector3(0, -(32 + 4), 0),
    Vector3(0, -(64 + 32 + 8 + 4), 0),
}
```

注意——**4 个 64px 槽位中央偏移由 32+4 微调**，让上 2 槽和下 2 槽对称。这种"自定义偏移"在带按钮的容器（buttoninfo 在底部）很常见——让槽位避开按钮区。

#### 不规则布局

完全可以**手动写每个 Vector3**——比如做一个"L 形"工具栏：

```lua
slotpos = {
    Vector3(-80,  0, 0),
    Vector3(  0,  0, 0),
    Vector3( 80,  0, 0),
    Vector3( 80,-80, 0),  -- 折下来
    Vector3( 80,-160, 0),
}
```

### 16.5.5 进阶：`itemtestfn` 物品过滤

`itemtestfn(container, item, slot)` 是**容器最重要的过滤钩子**——决定"哪些物品能放进来"。

**返回 true** = 允许，**返回 false** = 拒绝（拖拽时显示红 X）。

#### 标签过滤

```lua
-- 只接受食物
function params.mychest.itemtestfn(container, item, slot)
    return item:HasTag("edible")
end

-- 只接受植物种子
function params.seedbox.itemtestfn(container, item, slot)
    return item:HasTag("plantable")
end
```

#### 槽位特定限制

`slot` 参数告诉你"目标槽位编号"——可以**让不同槽位接受不同物品**：

```lua
-- 槽 1：只放钥匙；槽 2-5：只放金币
function params.lockbox.itemtestfn(container, item, slot)
    if slot == 1 then
        return item:HasTag("key")
    else
        return item.prefab == "goldnugget"
    end
end
```

#### 容器状态依赖

通过 `container` 引用容器实体——可以检查容器自己的状态：

```lua
-- 烧毁的烹饪锅不接受食材
function params.cookpot.itemtestfn(container, item, slot)
    return cooking.IsCookingIngredient(item.prefab)
        and not container.inst:HasTag("burnt")
end
```

#### 食材类别（食谱辅助）

`scripts/containers.lua` 里 fishnet/pseudobox 等用 cooking 模块：

```lua
local cooking = require("cooking")

function params.mycook.itemtestfn(container, item, slot)
    return cooking.IsCookingIngredient(item.prefab)
end
```

### 16.5.6 进阶：`buttoninfo` 按钮 + 动画进度

#### `buttoninfo` ——容器底部按钮

烹饪锅的"煮"按钮、晾肉架的"开始晾"按钮、宝石机的"激活"按钮——**全是 `buttoninfo`** 配置。

```363:387:scripts\containers.lua
        buttoninfo =
        {
            text = STRINGS.ACTIONS.COOK,
            position = Vector3(0, -165, 0),
        }
    },
    acceptsstacks = false,
    type = "cooker",
}

function params.cookpot.itemtestfn(container, item, slot)
    return cooking.IsCookingIngredient(item.prefab) and not container.inst:HasTag("burnt")
end

function params.cookpot.widget.buttoninfo.fn(inst, doer)
    if inst.components.container ~= nil then
        BufferedAction(doer, inst, ACTIONS.COOK):Do()
    elseif inst.replica.container ~= nil and not inst.replica.container:IsBusy() then
        SendRPCToServer(RPC.DoWidgetButtonAction, ACTIONS.COOK.code, inst, ACTIONS.COOK.mod_name)
    end
end

function params.cookpot.widget.buttoninfo.validfn(inst)
    return inst.replica.container ~= nil and inst.replica.container:IsFull()
end
```

`buttoninfo` 字段：

| 字段 | 作用 |
|------|------|
| `text` | 按钮文字（用 `STRINGS.ACTIONS.XXX` 已本地化）|
| `position` | 按钮在 widget 内的坐标 |
| `fn(container, doer)` | 点击时执行——通常发 BufferedAction 或 RPC |
| `validfn(container)` | 返回 false 时按钮变灰（锅没装满时不能煮）|

**关键的网络模式**——`fn` 在客户端被点击：

- 如果客户端有 `container` 组件（单机/服务端兼有）——直接 `BufferedAction:Do()`。
- 否则（纯客户端）——`SendRPCToServer(RPC.DoWidgetButtonAction, ACTION.code, inst)`——服务端收到后才执行 `BufferedAction`。

mod 实战：

```lua
local STRINGS = GLOBAL.STRINGS
STRINGS.ACTIONS.MYACTION = "Activate"

containers.params.mychest.widget.buttoninfo = {
    text = STRINGS.ACTIONS.MYACTION,
    position = GLOBAL.Vector3(0, -165, 0),
}

containers.params.mychest.widget.buttoninfo.fn = function(inst, doer)
    if inst.components.container ~= nil then
        -- 服务端：直接做事
        DoMagic(inst)
    elseif inst.replica.container ~= nil then
        -- 客户端：发 RPC
        GLOBAL.SendModRPCToServer(
            GLOBAL.GetModRPC("MyMod", "ChestActivate"), inst)
    end
end

-- 仅当装满时按钮可用
containers.params.mychest.widget.buttoninfo.validfn = function(inst)
    return inst.replica.container ~= nil
        and inst.replica.container:IsFull()
end
```

#### `animfn` / `animloop` ——播放动画进度

烹饪锅、暗影门户等"打开后持续动画"的容器，用 `animfn` 或 `animloop`：

```36:48:scripts\widgets\containerwidget.lua
    if widget.animbank ~= nil then
        local animbank = isinfinitestacksize and widget.animbank_upgraded or widget.animbank
        self.bganim:GetAnimState():SetBank(animbank)
    end

    if widget.animbuild ~= nil then
        local animbuild = isinfinitestacksize and widget.animbuild_upgraded or widget.animbuild
        self.bganim:GetAnimState():SetBuild(animbuild)
    end

    if widget.bganim_visualfn ~= nil then
        widget.bganim_visualfn(self.bganim, container, doer)
    end
```

**`animfn(container, doer, action)`** 返回不同状态的动画名（"open"/"open_loop"/"close"/...）——让 widget 根据容器状态切换外观。**`animloop = true`** 让"open" 后自动 push "open_loop" 循环。

```lua
-- 暗影箱：打开持续旋转动画
params.shadow_container.widget.animloop = true
```

```lua
-- 自定义：根据容器内是否装满切换 build
containers.params.mychest.widget.animfn = function(container, doer, action)
    if action == "open" then
        if container.replica.container:IsFull() then
            return "open_full"
        else
            return "open_empty"
        end
    end
    return action
end
```

### 16.5.7 老手进阶：侧面板容器（背包跟随玩家）

#### `issidewidget` ——背包模式

`scripts/containers.lua` 第 23-43 行是**背包配置**：

```23:36:scripts\containers.lua
params.backpack =
{
    widget =
    {
        slotpos = {},
        animbank = "ui_backpack_2x4",
        animbuild = "ui_backpack_2x4",
        --pos = Vector3(-5, -70, 0),
        pos = Vector3(-5, -80, 0),        
    },
    issidewidget = true,
    type = "pack",
    openlimit = 1,
}
```

**`issidewidget = true`** 让 widget **挂到 HUD 的侧栏（containerroot_side）而不是中央**——因为背包是"装备状态"应该常显，不该挡视线。

`type = "pack"` + `openlimit = 1` 让玩家**最多只能打开 1 个 pack**（防止同时打开两个背包）。

**mod 应用**——做一个"工具腰带"自动挂边栏：

```lua
containers.params.toolbelt = {
    widget = {
        slotpos = {},
        animbank = "ui_backpack_2x4",
        animbuild = "ui_backpack_2x4",
        pos = GLOBAL.Vector3(-5, -80, 0),
    },
    issidewidget = true,
    type = "pack",
    openlimit = 1,
}

for y = 0, 0 do
    for x = 0, 3 do
        table.insert(containers.params.toolbelt.widget.slotpos,
            GLOBAL.Vector3(-162 + 75 * x, -75 * y, 0))
    end
end
```

#### `posfn` —— 动态位置

`pos` 是静态偏移；如果你想**根据 doer/container 状态动态计算位置**，用 `posfn`：

```50:53:scripts\widgets\containerwidget.lua
	local pos = widget.posfn and widget.posfn(container, doer) or widget.pos
	if pos then
		self:SetPosition(pos)
    end
```

mod 应用——**让背包根据玩家身材调整位置**：

```lua
containers.params.toolbelt.widget.posfn = function(container, doer)
    if doer:HasTag("smallplayer") then
        return GLOBAL.Vector3(-5, -50, 0)  -- 矮一点
    else
        return GLOBAL.Vector3(-5, -80, 0)
    end
end
```

#### `slotposfn` —— 动态槽位

类似的 `slotposfn(container, doer)` 让你**运行时改槽位数量/位置**——例如做一个"等级越高槽越多"的容器：

```134:142:scripts\widgets\containerwidget.lua
	local slotpos = widget.slotposfn and widget.slotposfn(container, doer) or widget.slotpos
	for i, v in ipairs(slotpos or {}) do
        local bgoverride = widget.slotbg ~= nil and widget.slotbg[i] or nil
        local slot = InvSlot(i,
            bgoverride ~= nil and bgoverride.atlas or "images/hud.xml",
            bgoverride ~= nil and bgoverride.image or (constructionmats ~= nil and "inv_slot_construction.tex" or "inv_slot.tex"),
            self.owner,
            container.replica.container
        )
```

```lua
containers.params.expandingbag.widget.slotposfn = function(container, doer)
    local level = doer.components.expbag and doer.components.expbag.level or 1
    local positions = {}
    for i = 1, level * 2 do
        table.insert(positions, GLOBAL.Vector3(80 * (i-1) - 80, 0, 0))
    end
    return positions
end
```

### 16.5.8 老手进阶：五个常见的容器陷阱

#### 陷阱一：`widgetsetup` 没在 prefab 里调

```lua
-- ❌ 只挂了 component 没设置 widget
inst:AddComponent("container")
-- 后果：服务端 GiveItem 正常但客户端打开窗口空白
```

**修复**——必须显式 `WidgetSetup` 或在 `containers.params` 里有同名 prefab 配置（引擎自动查）：

```lua
inst:AddComponent("container")
inst.components.container:WidgetSetup("mychest")  -- 必须！
```

实际上**如果 prefab 名 == params key 名**（第 9-17 行），可以不显式调——`Container` 构造时会自动查表：

```9:17:scripts\containers.lua
function containers.widgetsetup(container, prefab, data)
    local t = data or params[prefab or container.inst.prefab]
    if t ~= nil then
        for k, v in pairs(t) do
            container[k] = v
        end
		container:SetNumSlots(container.widget.numslots or (container.widget.slotpos and #container.widget.slotpos or 0))
    end
end
```

但**新手为安全起见仍建议显式调**。

#### 陷阱二：`itemtestfn` 在客户端引用服务端组件

```lua
-- ❌ 客户端没 components.container（只有 replica.container）
function params.mychest.itemtestfn(container, item, slot)
    if container.components.container:IsFull() then  -- ❌ 客户端 nil
        return false
    end
    return true
end
```

**修复**——`itemtestfn` 用 **`container.replica.container`** 而不是 `container.components.container`，或同时检查：

```lua
function params.mychest.itemtestfn(container, item, slot)
    local rep = container.replica.container
    if rep and rep:IsFull() then
        return false
    end
    return true
end
```

#### 陷阱三：忘记声明 `acceptsstacks`

```lua
-- 默认 acceptsstacks = true（同 prefab 物品自动堆叠到一格）
-- 烹饪锅必须 acceptsstacks = false（每格放一个食材）
params.cookpot.acceptsstacks = false
```

**没设的话，烹饪时四个一样的食材自动堆到一格**——食谱失败。

#### 陷阱四：`animbank/animbuild` 不存在导致界面空白

```lua
-- ❌ 用了不存在的资源
params.mychest.widget.animbank = "ui_my_chest_3x3"  -- 你 mod 没这个 zip
```

**修复**——要么用官方现有 build（`ui_chest_3x3` / `ui_backpack_2x4` / `ui_cookpot_1x4` 等），要么**编译自己的 zip 资源**（13.3 章方法）并用 `Asset` 注册。

最简单——**复用官方资源**：

```lua
animbank = "ui_chest_3x3",      -- Klei 内置
animbuild = "ui_chest_3x3",
```

#### 陷阱五：自定义按钮的 RPC 没注册

```lua
-- ❌ 服务端会报 "Unknown RPC"
buttoninfo.fn = function(inst, doer)
    SendModRPCToServer(GetModRPC("MyMod", "ChestActivate"), inst)
end

-- 但 modmain 里没 AddModRPCHandler！
```

**修复**——必须 `modmain` 里注册：

```lua
-- modmain.lua
AddModRPCHandler("MyMod", "ChestActivate", function(player, inst)
    if inst and inst.components.container then
        DoMagic(inst)
    end
end)
```

### 16.5.9 老手实战：一个"按等级扩容"的智能背包

把所有内容整合：mod 加一个"经验背包"——玩家每升级 5 级，背包多一行槽位（最多 4 行 × 4 列 = 16 格）。

```lua
-- modmain.lua
local containers = GLOBAL.require("containers")

containers.params.expbag = {
    widget = {
        slotpos = {},                         -- 空 — 用 slotposfn 动态生成
        animbank = "ui_backpack_2x4",         -- 复用官方
        animbuild = "ui_backpack_2x4",
        pos = GLOBAL.Vector3(-5, -80, 0),
    },
    issidewidget = true,
    type = "pack",
    openlimit = 1,
}

-- 动态槽位
containers.params.expbag.widget.slotposfn = function(container, doer)
    local level = doer.components.experience and
        doer.components.experience.level or 1
    local rows = math.min(math.ceil(level / 5), 4)  -- 1-4 行
    local cols = 4
    local positions = {}
    for y = 0, rows - 1 do
        for x = 0, cols - 1 do
            table.insert(positions, GLOBAL.Vector3(
                75 * x - 75 * 1.5,
                -75 * y + 75 * (rows - 1) / 2,
                0))
        end
    end
    return positions
end

-- 数量也要同步动态
containers.params.expbag.widget.numslots = nil  -- 用 slotpos 长度
```

```lua
-- prefabs/expbag.lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    inst.AnimState:SetBank("backpack")
    inst.AnimState:SetBuild("swap_backpack")
    inst.AnimState:PlayAnimation("anim")

    inst:AddTag("backpack")

    MakeInventoryPhysics(inst)
    MakeInventoryFloatable(inst)

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")
    inst:AddComponent("equippable")
    inst.components.equippable.equipslot = EQUIPSLOTS.BODY
    inst:AddComponent("container")
    inst.components.container:WidgetSetup("expbag")

    return inst
end

return Prefab("expbag", fn,
    {Asset("ANIM", "anim/swap_backpack.zip")})
```

**测试**——玩家等级 1 时背包 4 格、5 级时 8 格、20 级时 16 格满格。

### 16.5.10 小结

```
        ┌──────────────────────────────────────────────┐
        │           cookpot/icebox/chester/...         │
        │                                              │
        │   ┌──────────────────────┐                   │
        │   │ Container 组件（服务端）│                  │
        │   │  - 真实物品数据       │                   │
        │   │  - GiveItem/RemoveItem│                  │
        │   └──────────────┬───────┘                   │
        │                  │ 网络同步                  │
        │                  ▼                            │
        │   ┌──────────────────────┐                   │
        │   │ Container_replica（客）│                  │
        │   │  - 网络副本           │                   │
        │   │  - GetWidget()→params │                  │
        │   └──────────────┬───────┘                   │
        │                  │ 玩家右键打开              │
        │                  ▼                            │
        │   ┌──────────────────────┐                   │
        │   │  ContainerWidget(UI) │                   │
        │   │  - bganim/bgimage    │                   │
        │   │  - InvSlot 数组      │                    │
        │   │  - buttoninfo 按钮   │                   │
        │   └──────────────────────┘                   │
        │                                              │
        │   关键配置在 containers.params.<prefab>:     │
        │   • slotpos / slotposfn   - 槽位坐标         │
        │   • animbank / animbuild  - 外观             │
        │   • itemtestfn            - 物品过滤         │
        │   • buttoninfo            - 底部按钮         │
        │   • issidewidget          - 是否常驻侧栏     │
        │   • acceptsstacks         - 同槽堆叠        │
        │   • type                  - chest/cooker/pack│
        └──────────────────────────────────────────────┘
```

| 概念 | 速记 |
|------|------|
| **三件套** | Container（服务端数据）+ replica（客户端副本）+ ContainerWidget（UI）|
| **`containers.params.<prefab>`** | 容器 UI 配置的中心 |
| **`widget.slotpos`** | 槽位坐标数组（决定网格布局）|
| **`widget.animbank/animbuild`** | 弹窗外观（用 zip 动画）|
| **`itemtestfn(container, item, slot)`** | 物品过滤——返回 false 拒绝 |
| **`buttoninfo`** | 底部按钮（fn/text/position/validfn）|
| **`acceptsstacks`** | 同槽堆叠（烹饪锅 false）|
| **`issidewidget`** | 侧栏（背包风格）|
| **`type`** | "chest"/"cooker"/"pack"/"top_rack" 等 |
| **`slotposfn` / `posfn`** | 运行时动态计算槽位/位置 |
| **`animfn` / `animloop`** | 不同状态切动画 / 循环动画 |
| **`SendRPCToServer`** | 客户端按钮要触发服务端动作（必备）|

**新手核心三句**：容器界面三件套——Container（数据）、replica（同步）、ContainerWidget（UI）；配置都写在 `containers.params.<prefab>.widget`；30 行代码能做一个 mod 宝箱。

**进阶核心三句**：`slotpos` 用 NxM 循环 + 偏移生成；`itemtestfn` 决定物品过滤（客户端服务端都跑）；`buttoninfo` 加底部按钮 + RPC 触发服务端动作。

**老手核心三句**：`issidewidget = true` 让背包常驻侧栏；`slotposfn` / `posfn` 实现"运行时动态槽位"；五个常见陷阱里"`itemtestfn` 引用 components 不引用 replica" 是最高频的。

下一节（16.6）我们将讨论 **ScrollableList / Grid** ——复杂列表与网格布局，包括如何展示长条目列表（mod 列表、好友列表、皮肤列表）、`TEMPLATES.ScrollingGrid` 的使用、高效的虚拟滚动技巧。


## 16.6 ScrollableList / Grid——复杂列表与网格布局

> **写在前面**——前面我们做的 UI 都很"小"——状态栏几个 badge、对话框几个按钮、容器几格槽位。**但有些场景天生就是"长长的列表"**：
>
> - **服务器列表**——可能有几千个公开服务器
> - **mod 列表**——配置面板里几十个 mod
> - **皮肤列表**——上千个角色皮肤
> - **食谱书**——一百多种食谱
> - **聊天记录**——历史滚动
> - **生物图鉴**——上百种生物
>
> 这些列表都有共同问题：**屏幕只能放下 5-10 行，剩下几百几千行怎么显示？** 答案是 **滚动 + 虚拟化**——只渲染屏幕能看到的那几行，滚动时**把已经看不见的 widget 拿来重用**显示新数据。这是 UI 性能优化的核心技巧之一。
>
> 饥荒提供了三套机制：**`Grid`**（静态网格布局，不滚动）、**`ScrollableList`**（旧版可滚动列表，每行一个真实 widget）、**`TrueScrollList`**（虚拟滚动，固定数量 widget 重用——这是性能最优的现代方案）。**`TEMPLATES.ScrollingGrid`** 是 Klei 在前两者基础上做的"高层封装"——绝大多数 mod 用它就够了。
>
> **新手**先看 16.6.1-16.6.3——理解 `Grid` 的基本用法（NxM 网格 + 焦点导航）、`ScrollableList` 简单列表场景；**进阶读者**继续看 16.6.4-16.6.6，深入 `TrueScrollList` 的虚拟滚动原理、`SetItemsData` / `update_fn` / `create_widgets_fn` 三联模式、`TEMPLATES.ScrollingGrid` 的高层封装；**老手**直接看 16.6.7-16.6.8，掌握滚动条样式定制（`SCROLLBAR_STYLE`）、`scissor` 裁剪与焦点流、五个常见的列表陷阱。

### 16.6.1 快速入门：`Grid` —— 静态网格

**`Grid` 是不滚动的固定网格**——适合"屏幕能放下的 NxM 项"——比如设置面板上的 8 个分类按钮、合成菜单的 4×3 物品图标。

源码 `scripts/widgets/grid.lua` 第 3-11 行：

```3:11:scripts\widgets\grid.lua
local Grid = Class(Widget, function(self)
    Widget._ctor(self, "GRID")
    self.h_offset = 100
    self.v_offset = 100
    self.items_by_coords = {}
    self.rows = 0
    self.cols = 0
    self.layout_left_to_right_top_to_bottom = false
end)
```

**最简用法（10 行代码）**：

```lua
local Grid = require "widgets/grid"
local Text = require "widgets/text"

local g = self:AddChild(Grid())
g:FillGrid(2, 100, 100, {              -- 2 列、列间距 100、行间距 100
    Text(UIFONT, 20, "Apple"),
    Text(UIFONT, 20, "Banana"),
    Text(UIFONT, 20, "Cherry"),
})
-- 结果：
--   Apple   Banana
--   Cherry
```

`FillGrid(num_columns, h_offset, v_offset, items)` 一行搞定布局——第 200-205 行：

```200:205:scripts\widgets\grid.lua
function Grid:FillGrid(num_columns, coffset, roffset, items)
    self.num_rows = math.ceil(#items / num_columns)
    self:UseNaturalLayout()
    self:InitSize(num_columns, self.num_rows, coffset, roffset)
    self:AddList(items)
end
```

#### `Grid` 的两套坐标系

第 36-40 行 + 第 164-171 行：

```36:40:scripts\widgets\grid.lua
-- Use the expected english text flow style layout.
function Grid:UseNaturalLayout()
    assert(#self.children == 0, "Call UseNaturalLayout before adding any items to the grid!")
    self.layout_left_to_right_top_to_bottom = true
end
```

```164:171:scripts\widgets\grid.lua
function Grid:_Layout(c,r, widget)
    local row_on_screen = r - 1
    local col_on_screen = c - 1
    if self.layout_left_to_right_top_to_bottom then
        row_on_screen = 1 - r
    end
	widget:SetPosition(Vector3(self.h_offset*col_on_screen, self.v_offset*row_on_screen, 0))
end
```

**默认坐标系**（数学习惯）——`(c, r)` 中 r 越大越往**上**（y 正方向）。

**自然坐标系**（阅读习惯）——`r=1` 在最上面、`r=2` 往下、`r=N` 在最底。`UseNaturalLayout()` 切到这个模式。**绝大多数文字列表用自然坐标更顺手**。

#### Grid 的焦点导航——免费

第 51-80 行 `DoFocusHookups` —— 每次 `AddItem` 后自动调用，**给每个格子设置 4 个方向的焦点流**。手柄玩家按上下左右就能在格子间跳——你不写一行代码。

```lua
g:SetLooping(true, true)  -- 横向循环 + 纵向循环（最右一列按 → 跳到最左）
```

**实战**——做一个"角色选择"6×3 网格：

```lua
local g = self:AddChild(Grid())
g:UseNaturalLayout()
g:InitSize(6, 3, 80, 100)             -- 6 列 3 行、列间距 80、行间距 100

local heroes = {"wilson", "willow", "wolfgang", "wendy", "wx78", "wickerbottom",
                "woodie", "wes",  "wickerbottom2", "wathgrithr", "webber", "winona",
                "warly", "wortox", "wormwood", "wurt", "walter", "wanda"}

for i, name in ipairs(heroes) do
    local btn = ImageButton("images/character/"..name..".xml", name..".tex")
    btn:SetOnClick(function() SelectCharacter(name) end)
    local c = ((i - 1) % 6) + 1
    local r = math.floor((i - 1) / 6) + 1
    g:AddItem(btn, c, r)
end

g:SetFocus(1, 1)                       -- 默认聚焦左上角
```

### 16.6.2 快速入门：`ScrollableList` —— 简单滚动列表

**`ScrollableList` 是"屏幕放不下时给一个滚动条"的简单列表**——适合 mod 列表（几十项）、世界设置（一百项）等中等长度。

源码 `scripts/widgets/scrollablelist.lua` 第 36-37 行：

```36:37:scripts\widgets\scrollablelist.lua
local ScrollableList = Class(Widget, function(self, items, listwidth, listheight, itemheight, itempadding, updatefn, widgetstoupdate, widgetXOffset, always_show_static, starting_offset, yInit, bar_width_scale_factor, bar_height_scale_factor, scrollbar_style)
    Widget._ctor(self, "ScrollBar")
```

**两种使用模式**：

#### 模式一：直接传 widget 数组（适合短列表）

```lua
local items = {}
for i, mod in ipairs(GetModList()) do
    local row = Widget("modrow")
    local txt = row:AddChild(Text(NEWFONT, 25, mod.name))
    txt:SetPosition(0, 0)
    table.insert(items, row)
end

local list = self:AddChild(ScrollableList(
    items,                  -- 直接传 widget 数组
    400,                    -- listwidth
    300,                    -- listheight
    50,                     -- 每行高度
    5,                      -- 行间距
    nil, nil,               -- 不用 update 模式
    nil, false, 0, 0))
```

每个 item **是一个完整 widget**——内存占用与列表长度成正比。**< 200 行用这种**。

#### 模式二：复用 widget + updatefn（千行列表）

```lua
local widgetstoupdate = {}
for i = 1, 10 do                         -- 只创建 10 个 widget（屏幕能见的）
    local row = Widget("row")
    row.txt = row:AddChild(Text(NEWFONT, 25, ""))
    row.btn = row:AddChild(ImageButton(...))
    table.insert(widgetstoupdate, row)
end

local data = {}                          -- 数据数组（可以是几千项）
for i = 1, 5000 do
    table.insert(data, {name = "Item "..i, ...})
end

local list = self:AddChild(ScrollableList(
    data, 400, 300, 50, 5,
    function(context, widget, index, dataitem)  -- updatefn
        widget.txt:SetString(dataitem.name)
        -- ... 更新其他子 widget
    end,
    widgetstoupdate))                            -- 10 个固定 widget 在里面循环重用
```

**新手只用模式一**——简单直观；遇到性能问题（千行+）再切模式二或更上层的 `TEMPLATES.ScrollingGrid`。

### 16.6.3 快速入门：何时用 Grid / ScrollableList / ScrollingGrid

| 场景 | 项目数 | 推荐 |
|------|-------|------|
| 6 个分类按钮 | < 30 | `Grid` |
| 角色选择 (3×6) | < 30 | `Grid` |
| mod 配置（10 项滚动） | < 100 | `ScrollableList` 模式一 |
| 世界设置 | 50-200 | `ScrollableList` 模式一 |
| **皮肤库存（几千皮肤）** | 1000+ | **`TEMPLATES.ScrollingGrid`** |
| **服务器列表（数千服务器）** | 1000+ | **`TEMPLATES.ScrollingGrid`** |
| **聊天历史（不限长）** | ∞ | **`TrueScrollList`**（自定义） |

**新手原则**——能不滚动就不滚动；要滚动用 `ScrollableList`；超过 200 项性能开始堪忧时升级到 `TEMPLATES.ScrollingGrid`。

### 16.6.4 进阶：`TrueScrollList` 虚拟滚动原理

打开 `scripts/widgets/truescrolllist.lua` 第 8-23 行的注释——它讲了**虚拟滚动的核心思想**：

```8:23:scripts\widgets\truescrolllist.lua
-------------------------------------------------------------------------------------------------------

-- A scrolling list of items.
--
-- We have a visible set of widgets and shift data between them to simulate
-- scrolling. SetItemsData and update_fn apply new data to widgets.
--
-- create_widgets_fn must create a set of widgets that can be updated by
-- update_fn. TrueScrollList will move `parent` for scrolling motion.
--
-- update_fn must apply the input data (an element of items from SetItemsData)
-- into the widget.
--
-- scissor_x/y/width/height are a bottom-left anchored box of the visible
-- region.
local TrueScrollList = Class(Widget, function(self, context, create_widgets_fn, update_fn, scissor_x, scissor_y, scissor_width, scissor_height, scrollbar_offset, scrollbar_height_offset, scroll_per_click)
```

**虚拟滚动的精髓**：

```
数据：1000 项 [data1, data2, ..., data1000]
                  ↑
        屏幕能放：5 行（visible_rows）
                  ↓
   只创建 5+2 = 7 个 widget（多 2 个为缓冲）
                  ↓
   滚动到位置 X：让 widget[1..5] 显示 data[X..X+4]
                  ↓
   滚动到 X+1：用同样的 widget[1..5] 显示 data[X+1..X+5]
                                                    ↑
                                              update_fn 把新数据写进去
```

**所以无论数据 100 项还是 1000 万项，UI 内存占用都恒定**——只跟 `visible_rows` 有关。

#### 三件套：`create_widgets_fn` / `update_fn` / `SetItemsData`

构造 `TrueScrollList(context, create_widgets_fn, update_fn, scissor_x, scissor_y, scissor_w, scissor_h, ...)` 时：

**`create_widgets_fn(context, parent, scroll_list)`** —— 创建固定数量的 widget 池子。返回 `(widgets, widgets_per_row, row_height, visible_rows, end_offset)`。

**`update_fn(context, widget, data, index)`** —— 把 `data`（数据数组的某一项）应用到 `widget`。每次滚动时被调用。

**`SetItemsData(items)`** —— 设置数据数组（任意长度）。修改后调用 `RefreshView`。

**官方例子**——`scripts/widgets/redux/itemexplorer.lua` 第 137-160 行：

```137:160:scripts\widgets\redux\itemexplorer.lua
    if list_options.item_ctor_fn == nil then
        list_options.item_ctor_fn = function(context, index)
            return self:_CreateScrollingGridItem(
                context,
                index,
                list_options.widget_width,
                list_options.widget_height)
        end
    end

    -- Most cases should use our default implementation -- especially if using
    -- CreateScrollingGridItem.
    if list_options.apply_fn == nil then
        list_options.apply_fn = ItemExplorer._ApplyDataToWidget
    end

    -- Pad the scissor region to ensure embiggened items don't have ugly
    -- clipping.
    list_options.scissor_pad = list_options.widget_width * 0.18

    -- Ensure full and empty screens look the same by always applying peek.
    list_options.peek_percent = 0.25

    self.scroll_list = self:AddChild(TEMPLATES.ScrollingGrid(contained_items, list_options))
```

`item_ctor_fn` 对应 `create_widgets_fn`、`apply_fn` 对应 `update_fn`、`contained_items` 是数据。

### 16.6.5 进阶：`TEMPLATES.ScrollingGrid` 高层封装

源码 `scripts/widgets/redux/templates.lua` 第 1961-2033 行——`TEMPLATES.ScrollingGrid` **把 Grid + TrueScrollList 拼到一起**：

```1961:1989:scripts\widgets\redux\templates.lua
function TEMPLATES.ScrollingGrid(items, opts)
    local peek_height = opts.peek_height or (opts.widget_height * 0.25) -- how much of row to see at the bottom.
    if opts.peek_percent then
        peek_height = opts.widget_height * opts.peek_percent
    elseif not opts.force_peek and #items < math.floor(opts.num_visible_rows) * opts.num_columns then
        peek_height = 0
    end
    local function ScrollWidgetsCtor(context, parent, scroll_list)
        local NUM_ROWS = opts.num_visible_rows + 2

        local widgets = {}
        for y = 1,NUM_ROWS do
            for x = 1,opts.num_columns do
                local index = ((y-1) * opts.num_columns) + x
                table.insert(widgets, parent:AddChild(opts.item_ctor_fn(context, index)))
            end
        end

        parent.grid = parent:AddChild(Grid())
        parent.grid:FillGrid(opts.num_columns, opts.widget_width, opts.widget_height, widgets)
```

**关键参数**：

| 参数 | 作用 |
|------|------|
| `widget_width` / `widget_height` | 每个格子尺寸 |
| `num_columns` / `num_visible_rows` | 网格列数 / 一屏能见行数 |
| `item_ctor_fn(context, index)` | 创建单个 widget（在 grid 里） |
| `apply_fn(context, widget, data, index)` | 把 data 写到 widget |
| `peek_percent` | "下一行偷看一点"的比例（提示玩家可滚动）|
| `scrollbar_offset` | 滚动条相对网格的位置 |
| `scissor_pad` | 裁剪区周围的留白（防 hover 缩放被裁）|

**实战——做一个 mod 物品图鉴**：

```lua
local TEMPLATES = require "widgets/redux/templates"
local TrueScrollList = require "widgets/truescrolllist"
local Image = require "widgets/image"

-- 数据
local items_data = {}
for prefab, info in pairs(MyModItemDB) do
    table.insert(items_data, {prefab = prefab, name = info.name, atlas = info.atlas, image = info.image})
end

-- 一个 widget 的构造函数
local function ItemCtor(context, index)
    local w = Widget("itemtile")
    w.bg = w:AddChild(Image("images/global.xml", "square.tex"))
    w.bg:SetSize(80, 80)
    w.icon = w:AddChild(Image())
    w.icon:SetSize(64, 64)
    w.label = w:AddChild(Text(NEWFONT, 16))
    w.label:SetPosition(0, -50)
    return w
end

-- 应用数据到 widget
local function ApplyFn(context, widget, data, index)
    if data == nil then
        widget:Hide()
        return
    end
    widget:Show()
    widget.icon:SetTexture(data.atlas, data.image)
    widget.label:SetString(data.name)
end

-- 创建滚动网格
local list = self:AddChild(TEMPLATES.ScrollingGrid(items_data, {
    widget_width = 100,
    widget_height = 100,
    num_columns = 5,
    num_visible_rows = 4,
    item_ctor_fn = ItemCtor,
    apply_fn = ApplyFn,
    scissor_pad = 20,
    peek_percent = 0.25,
}))
list:SetPosition(0, 0)
```

5×4 = 20 个格子 + 2 行缓冲 = 30 个 widget——**无论数据 100 还是 100,000 项都只创建这么多**。

### 16.6.6 进阶：`SetItemsData` 动态更新数据

`TrueScrollList` 第 275 行的 `SetItemsData(items)` —— 数据变化时调用：

```lua
-- 用户在搜索框输入文字 → 过滤数据 → 刷新列表
function MyScreen:OnSearchTextChanged(query)
    local filtered = {}
    for _, item in ipairs(self.all_items) do
        if string.find(item.name:lower(), query:lower(), 1, true) then
            table.insert(filtered, item)
        end
    end
    self.scroll_list:SetItemsData(filtered)   -- 一行更新
end
```

`SetItemsData` 内部会：

1. 重新计算总滚动范围
2. 重置滚动条
3. 调用 `apply_fn` 把当前可见区域的 widget 重新刷新

**常见模式——异步加载列表**：

```lua
-- 启动时显示 loading
self.scroll_list:SetItemsData({})  -- 空

-- 异步获取数据
TheNet:RequestModList(function(modlist)
    self.scroll_list:SetItemsData(modlist)  -- 数据回来后填进去
end)
```

### 16.6.7 老手进阶：滚动条样式 + Scissor 裁剪

#### 滚动条样式

`scrollablelist.lua` 第 14-32 行有两套预设：

```14:32:scripts\widgets\scrollablelist.lua
local SCROLLBAR_STYLE = {
    BLACK = {
        atlas = "images/ui.xml",
        up = "arrow_scrollbar_up.tex",
        down = "arrow_scrollbar_down.tex",
        bar = "scrollbarline.tex",
        handle = "scrollbarbox.tex",
        --~ scale = 0.4,
        scale = 1.0,
    },
    GOLD = {
        atlas = "images/global_redux.xml",
        up = "scrollbar_arrow_up.tex",
        down = "scrollbar_arrow_down.tex",
        bar = "scrollbar_bar.tex",
        handle = "scrollbar_handle.tex",
        scale = 0.3,
    }
}
```

**`BLACK`** 是老 UI 风格（mod 列表用），**`GOLD`** 是 redux 新风格（设置/服务器列表用）。

mod 想换成自己的图：

```lua
local list = ScrollableList(items, w, h, ih, ip, nil, nil, 0, false, 0, 0, 1, 1,
    "GOLD")  -- 用 GOLD 样式
```

或在 `widget._init` 后**直接修改子 widget**：

```lua
list.up_button:SetTextures("images/mymod.xml", "my_up.tex")
list.down_button:SetTextures("images/mymod.xml", "my_down.tex")
list.bar:SetTexture("images/mymod.xml", "my_bar.tex")
```

#### Scissor 裁剪

源码 `truescrolllist.lua` 第 43-44 行：

```43:44:scripts\widgets\truescrolllist.lua
	self.scissored_root = self:AddChild(Widget("scissored_root"))
    self.scissored_root:SetScissor(scissor_x, scissor_y, scissor_width, scissor_height)
```

**`SetScissor(x, y, w, h)`** 是 Widget 基类的方法（widget.lua 第 836 行）——**告诉引擎"这个 widget 及其子只在这个矩形区域内可见"**。区域外的部分被裁掉。

为什么需要——**滚动列表里上下两端有"半截子 widget"，没裁剪会"漏出来"**：

```
没 scissor：
    ┌─────────────┐
    │ Item 1     │ ← 半截
    │─────────────│ ← 滚动区上边界
    │ Item 2     │
    │ Item 3     │
    │ Item 4     │
    │─────────────│ ← 滚动区下边界
    │ Item 5     │ ← 半截，溢出
    └─────────────┘

有 scissor：
    ┌─────────────┐
    │             │
    │─────────────│
    │ Item 2     │  ← 完整
    │ Item 3     │  ← 完整
    │ Item 4     │  ← 完整
    │─────────────│
    │             │
    └─────────────┘
```

**坐标系**——`SetScissor(x, y, w, h)` 中 (x, y) 是**左下角**（不是中心、不是左上）。`TrueScrollList` 第 2014-2015 行：

```2014:2015:scripts\widgets\redux\templates.lua
    local scissor_x = -scissor_width/2
    local scissor_y = -scissor_height/2
```

把 scissor 矩形的中心放在 widget 中心。

#### `peek_height` —— 暗示可滚动

注意 `TEMPLATES.ScrollingGrid` 第 1962-1973 行的 peek_height：

```1962:1973:scripts\widgets\redux\templates.lua
    local peek_height = opts.peek_height or (opts.widget_height * 0.25) -- how much of row to see at the bottom.
    if opts.peek_percent then
        peek_height = opts.widget_height * opts.peek_percent
    elseif not opts.force_peek and #items < math.floor(opts.num_visible_rows) * opts.num_columns then
        peek_height = 0
    end
```

**peek_height 是"底部多露一点点下一行"的高度**——0.25 = 25% 的下一行可见。**这是 UI 设计的小心机**——告诉玩家"列表还有更多内容，可以滚下去"。

如果数据少于一屏（无需滚动），自动设 peek=0。

### 16.6.8 老手进阶：五个常见的列表陷阱

#### 陷阱一：`update_fn` 里 new 大量 widget

```lua
-- ❌ 每次刷新都重新创建子 widget
local function ApplyFn(context, widget, data, index)
    if widget.icon then widget.icon:Kill() end
    widget.icon = widget:AddChild(Image(data.atlas, data.image))  -- ❌ 每次滚动都 new！
end
```

**症状**——快速滚动时 GC 压力大、卡顿。

**修复**——`item_ctor_fn` 里**预先创建**所有子 widget，`apply_fn` 只**改属性**：

```lua
local function ItemCtor(context, index)
    local w = Widget("item")
    w.icon = w:AddChild(Image())                 -- 预创建（无图）
    return w
end

local function ApplyFn(context, widget, data, index)
    widget.icon:SetTexture(data.atlas, data.image)  -- ✅ 只换贴图
end
```

#### 陷阱二：`SetItemsData(nil)` 与空数组的差异

```lua
-- ❌ 想清空列表
self.list:SetItemsData()  -- nil

-- ✅
self.list:SetItemsData({})  -- 空数组
```

`SetItemsData(nil)` 是构造时的"未初始化"信号、不一定能正确清空已渲染的内容。**清空总用空数组 `{}`**。

#### 陷阱三：`apply_fn` 没处理 nil data

```lua
-- ❌ 滚到底部时 apply_fn 收到 data == nil（缓冲行）
local function ApplyFn(context, widget, data, index)
    widget.label:SetString(data.name)  -- ❌ data 可能 nil
end
```

**修复**：

```lua
local function ApplyFn(context, widget, data, index)
    if data == nil then
        widget:Hide()    -- ✅ 没数据就藏起来
        return
    end
    widget:Show()
    widget.label:SetString(data.name)
end
```

#### 陷阱四：网格列表的项之间互相覆盖（z 顺序）

```lua
-- ❌ 上面一行的 hover 提示被下面一行的 widget 盖住
```

**原因**——Grid `AddItem` 顺序导致后加的覆盖先加的。

**修复**——`TEMPLATES.ScrollingGrid` 的内部代码（第 1996-1998 行）已经处理：

```1996:1998:scripts\widgets\redux\templates.lua
        for i,w in ipairs(widgets) do
            w:MoveToBack()
        end
```

但你**自己手写时要注意**——上面行用 `MoveToFront`、下面行用 `MoveToBack`：

```lua
for i, w in ipairs(widgets) do
    if i > #widgets / 2 then
        w:MoveToBack()
    else
        w:MoveToFront()
    end
end
```

#### 陷阱五：滚动条不响应手柄

```lua
-- 鼠标滚轮可以滚，但手柄玩家用不了
```

**修复**——`TrueScrollList` 默认绑定 `CONTROL_SCROLLBACK` / `CONTROL_SCROLLFWD`（第 34-35 行）：

```34:35:scripts\widgets\truescrolllist.lua
	self.control_up = CONTROL_SCROLLBACK
	self.control_down = CONTROL_SCROLLFWD
```

但 **Screen 必须把 OnControl 转发给滚动列表**。模式：

```lua
function MyScreen:OnControl(control, down)
    if MyScreen._base.OnControl(self, control, down) then return true end
    -- 滚动列表自己已经监听了 SCROLLBACK/SCROLLFWD（在它的 widget 里）
    -- 但如果手柄玩家焦点不在列表上，要用 SetFocus 让焦点传过去
end
```

实战——`default_focus = self.scroll_list`，让玩家进 Screen 焦点直接在列表上、能直接滚。

### 16.6.9 老手实战：mod 物品图鉴

```lua
-- screens/myitemcompendium.lua
local Screen = require "widgets/screen"
local Widget = require "widgets/widget"
local Image = require "widgets/image"
local Text = require "widgets/text"
local ImageButton = require "widgets/imagebutton"
local TEMPLATES = require "widgets/redux/templates"

local function ItemTileCtor(context, index)
    local w = Widget("tile")
    w.bg = w:AddChild(Image("images/global.xml", "square.tex"))
    w.bg:SetSize(80, 80)
    w.bg:SetTint(0.2, 0.2, 0.2, 1)

    w.icon = w:AddChild(Image())
    w.icon:SetSize(64, 64)

    w.label = w:AddChild(Text(BODYTEXTFONT, 18))
    w.label:SetPosition(0, -50)
    w.label:SetRegionSize(90, 30)
    w.label:EnableWordWrap(true)
    return w
end

local function ApplyDataToTile(context, widget, data, index)
    if data == nil then
        widget:Hide()
        return
    end
    widget:Show()
    if data.discovered then
        widget.icon:SetTexture(data.atlas, data.image)
        widget.icon:SetTint(1, 1, 1, 1)
        widget.label:SetString(data.name)
    else
        widget.icon:SetTexture(data.atlas, data.image)
        widget.icon:SetTint(0, 0, 0, 1)
        widget.label:SetString("???")
    end
end

local MyCompendium = Class(Screen, function(self)
    Screen._ctor(self, "MyCompendium")

    self.proot = self:AddChild(Widget("root"))
    self.proot:SetVAnchor(ANCHOR_MIDDLE)
    self.proot:SetHAnchor(ANCHOR_MIDDLE)
    self.proot:SetScaleMode(SCALEMODE_PROPORTIONAL)

    self.title = self.proot:AddChild(Text(TITLEFONT, 50, "Compendium"))
    self.title:SetPosition(0, 280)

    -- 数据收集
    local data = {}
    for prefab, info in pairs(MyMod.ItemDB) do
        table.insert(data, {
            prefab = prefab,
            atlas = info.atlas,
            image = info.image,
            name = info.name,
            discovered = MyMod.IsDiscovered(prefab),
        })
    end

    -- 滚动网格
    self.list = self.proot:AddChild(TEMPLATES.ScrollingGrid(data, {
        widget_width = 100,
        widget_height = 110,
        num_columns = 6,
        num_visible_rows = 4,
        item_ctor_fn = ItemTileCtor,
        apply_fn = ApplyDataToTile,
        scissor_pad = 30,
        peek_percent = 0.25,
        scrollbar_offset = 20,
    }))
    self.list:SetPosition(0, 0)

    -- 关闭按钮
    self.close = self.proot:AddChild(ImageButton(
        "images/frontend.xml", "button_long.tex"))
    self.close:SetText("Close")
    self.close:SetPosition(0, -280)
    self.close:SetOnClick(function() TheFrontEnd:PopScreen(self) end)

    self.default_focus = self.list

    -- 搜索框
    self.search_btn = self.proot:AddChild(ImageButton(...))
    self.search_btn:SetOnClick(function()
        local query = ...  -- 弹个输入框
        local filtered = {}
        for _, d in ipairs(data) do
            if string.find(d.name:lower(), query:lower(), 1, true) then
                table.insert(filtered, d)
            end
        end
        self.list:SetItemsData(filtered)
    end)
end)

return MyCompendium
```

### 16.6.10 小结

```
                  数据规模决定方案

                  < 30 项
                     ↓
                ┌─────────┐
                │  Grid   │ ← 静态网格，自动焦点流
                └─────────┘

                30-200 项
                     ↓
            ┌────────────────┐
            │ ScrollableList │ ← 简单滚动，每项一 widget
            └────────────────┘

                200+ 项
                     ↓
        ┌───────────────────────┐
        │ TEMPLATES.ScrollingGrid│ ← 虚拟滚动，固定 widget 池
        │   ↓ 内部用             │
        │ TrueScrollList + Grid  │
        └───────────────────────┘

           虚拟滚动核心：
           visible_rows + 2 个缓冲 widget
                  ↑
           滚动时 update_fn 把不同数据写进
           已有 widget，永远不 new
```

| 概念 | 速记 |
|------|------|
| **`Grid`** | 静态 NxM 网格，`FillGrid(cols, h, v, items)` 一行搞定 |
| **`Grid:SetLooping(h, v)`** | 焦点循环（最右按 → 跳最左）|
| **`Grid:UseNaturalLayout()`** | 改阅读坐标（r=1 在最上）|
| **`ScrollableList`** | 滚动列表，模式一传 widget 数组、模式二 updatefn+池 |
| **`TrueScrollList`** | 虚拟滚动核心，`create_widgets_fn` + `update_fn` + `SetItemsData` |
| **`TEMPLATES.ScrollingGrid(items, opts)`** | 高层封装，绝大多数场景用它 |
| **`item_ctor_fn(context, index)`** | 创建单个格子 widget（在 grid 里）|
| **`apply_fn(context, widget, data, index)`** | 把数据应用到 widget；data 可能 nil 要处理 |
| **`SetItemsData(items)`** | 动态换数据（搜索过滤）|
| **`peek_percent`** | "下一行偷看"高度，暗示可滚 |
| **`SetScissor(x, y, w, h)`** | 矩形裁剪，避免半截子溢出 |
| **`SCROLLBAR_STYLE.GOLD`** | 新版金色滚动条样式 |

**新手核心三句**：< 30 项用 `Grid`、30-200 项用 `ScrollableList`、再多用 `TEMPLATES.ScrollingGrid`；`Grid:FillGrid(cols, h, v, items)` 一行搞定 NxM；焦点流由 `DoFocusHookups` 自动设置。

**进阶核心三句**：虚拟滚动核心是"固定 widget + 动态数据"，无论数据多少 widget 数恒定；`TEMPLATES.ScrollingGrid(items, opts)` 把这套封装到 5 个参数；`SetItemsData` 是动态换数据的标准入口。

**老手核心三句**：`apply_fn` 必须处理 `data == nil`（缓冲行）；`item_ctor_fn` 一次性创建所有子 widget，`apply_fn` 只改属性不 new；scissor 矩形坐标是"左下角原点"且必须 `SetScissor(-w/2, -h/2, w, h)` 把矩形居中。

下一节（16.7）我们将讨论 **Focus 系统** ——手柄/键盘导航焦点在 Widget 间的转移、`SetFocusChangeDir` / `SetDefaultFocus` / `focus_flow` / `GetDeepestFocus` 的完整工作原理。


## 16.7 Focus 系统——手柄/键盘导航焦点在 Widget 间的转移

> **写在前面**——前面我们写 UI 时遇到过 `default_focus`、`SetFocus`、`SetFocusChangeDir` 这些 API，但都是"用一下就过"，没有系统讲过它背后的机制。本节我们终于把这块讲清楚——**焦点（Focus）系统是怎么让玩家用手柄/键盘"在 widget 间走来走去"的**。
>
> 鼠标玩家不太需要焦点系统——鼠标到哪儿哪儿就是焦点。**焦点系统真正的服务对象是手柄玩家**——他们没有指针，**所有 UI 必须能用方向键 + 确认/取消键操作**。Klei 的整个 UI 必须**100% 手柄可玩**——主菜单、设置面板、库存、地图、调色板、好友列表……都要能用手柄走完。
>
> 焦点系统看似简单——"按 ↓ 跳到下面那个 widget"——但实际上有**七层抽象**：
>
> 1. **`self.focus`** —— 单个 widget 是否被聚焦
> 2. **`self.focus_flow[dir]`** —— 该 widget 在 4 个方向上的"下一焦点"
> 3. **`OnFocusMove(dir, down)`** —— 焦点移动的递归路由器
> 4. **`SetFocus()` / `ClearFocus()` / `SetFocusFromChild`** —— 焦点状态的传递
> 5. **`focus_forward`** —— 委托焦点给"嵌套子 widget"
> 6. **`default_focus`** —— Screen 出现时的默认焦点
> 7. **`TheFrontEnd:OnFocusMove`** —— 顶层路由器
>
> **新手**先看 16.7.1-16.7.3——理解 `self.focus` 字段、`SetFocus`/`ClearFocus` 的基本流程、`SetFocusChangeDir` 在按钮间手动连线；**进阶读者**继续看 16.7.4-16.7.6，深入 `OnFocusMove` 递归路由原理、`focus_forward` 委托机制、动态焦点目标（函数返回 widget）；**老手**直接看 16.7.7-16.7.8，掌握 Tab 键焦点（`next_in_tab_order`）、`SetFocusFromChild`、五个常见的焦点陷阱。

### 16.7.1 快速入门：`self.focus` —— 焦点的"存在性"

打开 `scripts/widgets/widget.lua` 第 31 行——Widget 构造时：

```lua
self.focus = false
```

每个 widget 自带 `focus` 字段——`true` 表示**当前被聚焦**。被聚焦的 widget：

- 显示"高亮"或"放大"效果（按钮 `OnGainFocus` 触发）
- **接收键盘/手柄的 `OnControl` 事件**（按 A/确认/Enter 时调用 onclick）
- **接收方向键的 `OnFocusMove`** 用于跳转

新手最直观的体验——**按钮"变大"** 就是焦点：

```108:139:scripts\widgets\imagebutton.lua
function ImageButton:OnGainFocus()
	ImageButton._base.OnGainFocus(self)

    if self.hover_overlay then
        self.hover_overlay:Show()
    end

	if self:IsSelected() or self:IsDisabledState() then return end

    if self:IsEnabled() then
        self.image:SetTexture(self.atlas, self.image_focus)
        ...

		if not self.ignore_standard_scaling  and self.image_focus == self.image_normal and self.scale_on_focus and self.focus_scale then
			self.image:SetScale(self.focus_scale[1], self.focus_scale[2], self.focus_scale[3])
		end
        ...
		if self.focus_sound then
			TheFrontEnd:GetSound():PlaySound(self.focus_sound)
		end
    end
end
```

`OnGainFocus` 被自动调用——切换到 hover 贴图、放大、播声音。**这一切都因为 `focus = true`**。

#### `SetFocus()` 设置焦点

源码 `widget.lua` 第 659-688 行：

```659:688:scripts\widgets\widget.lua
function Widget:SetFocus()
    local focus_forward = FunctionOrValue(self.focus_forward)
    if focus_forward then
        focus_forward:SetFocus()
        return
    end

    if not self.focus then
        self.focus = true

        if self.OnGainFocus then
            self:OnGainFocus()
        end

        if self.ongainfocusfn then
            self.ongainfocusfn()
        end

        if self.parent then
            self.parent:SetFocusFromChild(self)
        end
    end

    for k,v in pairs(self.children) do
        v:ClearFocus()
    end
end
```

**`SetFocus()` 干的三件事**：

1. **`self.focus = true`**
2. **触发 `OnGainFocus()` 回调**（widget 子类重写，imagebutton 在这里换 hover 贴图）+ `ongainfocusfn` 用户自定义回调
3. **通知父节点 `SetFocusFromChild(self)`** —— 让父节点把"哪个子被聚焦"也记一下、并取消其他兄弟的焦点

**最简使用**：

```lua
local btn = self:AddChild(ImageButton(...))
btn:SetFocus()    -- Screen 显示时让玩家直接看到这个按钮被高亮
```

#### `ClearFocus()`

第 615-630 行——和 `SetFocus` 完全对称，但递归往下清：

```615:630:scripts\widgets\widget.lua
function Widget:ClearFocus()
    if self.focus then
        self.focus = false
        if self.OnLoseFocus then
            self:OnLoseFocus()
        end
        if self.onlosefocusfn then
            self.onlosefocusfn()
        end
        for k,v in pairs(self.children) do
            if v.focus then
                v:ClearFocus()
            end
        end
    end
end
```

### 16.7.2 快速入门：`SetFocusChangeDir` —— 手动连线

让方向键能跳到指定 widget——`SetFocusChangeDir(dir, target)`：

源码第 583-590 行：

```583:590:scripts\widgets\widget.lua
function Widget:SetFocusChangeDir(dir, widget, ...)
    if not next(self.focus_flow) then
        self.next_in_tab_order = widget
    end

    self.focus_flow[dir] = widget
    self.focus_flow_args[dir] = toarrayornil(...)
end
```

`dir` 可以是 `MOVE_UP` / `MOVE_DOWN` / `MOVE_LEFT` / `MOVE_RIGHT`（`scripts/constants.lua` 第 93-96 行）：

```93:96:scripts\constants.lua
MOVE_UP = 1
MOVE_DOWN = 2
MOVE_LEFT = 3
MOVE_RIGHT = 4
```

**实战**——三个垂直按钮：

```lua
local btn1 = self:AddChild(ImageButton(...))
local btn2 = self:AddChild(ImageButton(...))
local btn3 = self:AddChild(ImageButton(...))

btn1:SetPosition(0, 100)
btn2:SetPosition(0, 0)
btn3:SetPosition(0, -100)

btn1:SetFocusChangeDir(MOVE_DOWN, btn2)
btn2:SetFocusChangeDir(MOVE_UP, btn1)
btn2:SetFocusChangeDir(MOVE_DOWN, btn3)
btn3:SetFocusChangeDir(MOVE_UP, btn2)

self.default_focus = btn1
```

按 ↓ 焦点 1→2→3，按 ↑ 反向。

#### Grid 自动连线

`Grid` 的 `DoFocusHookups`（16.6 章学过）**自动设置整网格的焦点流**——你不需要手动写：

```51:80:scripts\widgets\grid.lua
function Grid:DoFocusHookups()
	for c = 1, self.cols do
		for r = 1, self.rows do
			local item = self:GetItemInSlot(c,r)
			if item then
				item:ClearFocusDirs()
				local up = r > 1 and self:GetItemInSlot(c,r-1)
				local down = r < self.rows and self:GetItemInSlot(c,r+1)
				local left = c > 1 and self:GetItemInSlot(c-1,r)
				local right = c < self.cols and self:GetItemInSlot(c+1,r)
                ...
				if up then item:SetFocusChangeDir(MOVE_UP, up) end
				if down then item:SetFocusChangeDir(MOVE_DOWN, down) end
				if left then item:SetFocusChangeDir(MOVE_LEFT, left) end
				if right then item:SetFocusChangeDir(MOVE_RIGHT, right) end
			end
		end
	end
end
```

**新手用 `Grid` 做按钮组就有"免费"的焦点导航**——连 `SetFocusChangeDir` 都不用写。

### 16.7.3 快速入门：`default_focus` 与 `OnGainFocus` 钩子

#### `default_focus`

我们在 16.3 学过 Screen 的 `default_focus`：

```70:75:scripts\widgets\screen.lua
function Screen:SetDefaultFocus()
	if self.default_focus then
		self.default_focus:SetFocus()
		return true
	end
end
```

Screen 显示时自动调 `SetDefaultFocus`——这就是"默认聚焦哪个按钮"的实现。

**实战**——总是把 default_focus 指向"最常用按钮"：

```lua
function MyConfirmDialog:_ctor()
    Screen._ctor(self, "MyConfirmDialog")
    self.proot = ...
    self.btn_confirm = ...
    self.btn_cancel = ...

    self.default_focus = self.btn_confirm  -- "确定"通常是默认
end
```

#### `SetOnGainFocus` / `SetOnLoseFocus`

不想重写 `OnGainFocus` 而想加一个"额外回调"——用 `SetOnGainFocus(fn)`：

```569:575:scripts\widgets\widget.lua
function Widget:SetOnGainFocus( fn )
    self.ongainfocusfn = fn
end

function Widget:SetOnLoseFocus( fn )
    self.onlosefocusfn = fn
end
```

**实战——焦点切换时显示提示**：

```lua
btn:SetOnGainFocus(function()
    self.tooltip:SetString(btn.tooltip_text)
    self.tooltip:Show()
end)

btn:SetOnLoseFocus(function()
    self.tooltip:Hide()
end)
```

### 16.7.4 进阶：`OnFocusMove` 递归路由

**焦点移动的核心算法**——`Widget:OnFocusMove(dir, down)`（第 72-99 行）：

```72:99:scripts\widgets\widget.lua
function Widget:OnFocusMove(dir, down)
    if not self.focus then return false end

    for k,v in pairs (self.children) do
        if v.focus and v:OnFocusMove(dir, down) then return true end
    end

    if down and self.focus_flow[dir] then
        local dest = FunctionOrValue(self.focus_flow[dir], self)

        -- Can we pass the focus down the chain if we are disabled/hidden?
        if dest and dest:IsVisible() and dest.enabled then
            if self.focus_flow_args[dir] then
                dest:SetFocus(unpack(self.focus_flow_args[dir]))
            else
                dest:SetFocus()
            end
            return true
        end
    end

    if self.parent_scroll_list then
        return self.parent_scroll_list:OnFocusMove(dir, down)
    end

    return false
end
```

**逐行解读**：

1. **`if not self.focus then return false end`** —— 自己不被聚焦就不参与处理（剪枝）
2. **递归子节点** —— 找到聚焦的子让它先处理
3. **检查自己的 `focus_flow[dir]`** —— 自己有该方向的目标
4. **检查目标是否"能用"**（`dest:IsVisible()` 和 `dest.enabled`）—— 隐藏/禁用的不能拿焦点
5. **`dest:SetFocus()`** —— 转移焦点
6. **fallback 给 `parent_scroll_list`** —— 滚动列表里的项可以让列表代为处理（自动滚到下一项）

#### 路由顺序：从内向外冒泡

```
┌─ Screen
   ├─ Container (focus=true)
   │   ├─ Grid (focus=true)
   │   │   ├─ btn1 (focus=true) ← 当前焦点
   │   │   └─ btn2
   │   └─ btn_close
   └─ ...

按 ↓：
1. Screen:OnFocusMove(MOVE_DOWN) - Screen 没 focus，skip
   或→ Screen 是顶层，焦点子是 Container
2. Container:OnFocusMove(MOVE_DOWN) - 有 focus，递归子
3. Grid:OnFocusMove(MOVE_DOWN) - 有 focus，递归子
4. btn1:OnFocusMove(MOVE_DOWN) - 有 focus，没子有 focus
   - 检查 btn1.focus_flow[MOVE_DOWN] = btn2
   - btn2 可见且启用 → btn2:SetFocus()
   - return true
5. true 沿调用链返回，结束
```

这个递归算法的好处——**多层 widget 嵌套都能正确路由**。Container 被聚焦时按 ↓，会一直递归到当前最深焦点 widget（`btn1`），再用它的 focus_flow 决定跳哪。

### 16.7.5 进阶：`focus_forward` —— 委托焦点

`focus_forward` 是焦点系统**最难理解但最强大**的概念。看它在 `SetFocus` 第 661-664 行的作用：

```661:664:scripts\widgets\widget.lua
    local focus_forward = FunctionOrValue(self.focus_forward)
    if focus_forward then
        focus_forward:SetFocus()
        return
    end
```

**用人话讲**——"如果一个 widget 设了 `focus_forward = X`，那么对它调 `SetFocus()` 时**实际焦点跳到 X**"。

**为什么需要**——容器型 widget（Grid、List、Tab）**自己不该被聚焦**——它只是个布局容器，应该让焦点直接落到内部某个具体按钮上。

#### 例子：`Grid` 的 focus_forward

`Grid` 没有显式设 `focus_forward`，但 `TEMPLATES.ScrollingGrid` 第 1992 行：

```1992:1992:scripts\widgets\redux\templates.lua
        parent.focus_forward = parent.grid
```

意思是——"如果有人对 scroll_list 调 `SetFocus()`，让它转给内部的 `grid`"。

#### 例子：`scrapbookscreen` 的多层委托

```161:161:scripts\screens\redux\scrapbookscreen.lua
	self.focus_forward = self.item_grid
```

```809:809:scripts\screens\redux\scrapbookscreen.lua
		w.focus_forward = w.item_root.button
```

**两层委托**——Screen 把焦点转给 `item_grid`、`item_grid` 里每个 item 把焦点转给 `item_root.button`。最终焦点直接落到按钮上——但调用方根本不需要知道这层结构。

#### `focus_forward` 可以是函数

第 661 行 `FunctionOrValue` 让它能是函数——**运行时计算"该转给谁"**：

```lua
self.focus_forward = function()
    if self.tab == "items" then
        return self.item_list
    else
        return self.recipe_list
    end
end
```

切换 Tab 时焦点自动跳到不同列表。

### 16.7.6 进阶：动态焦点目标

`SetFocusChangeDir(dir, fn)` 第二参数也可以是函数——**运行时计算"下一焦点是谁"**。

#### 实战 1：跳过隐藏按钮

```lua
btn1:SetFocusChangeDir(MOVE_DOWN, function()
    if self.btn2:IsVisible() then
        return self.btn2
    else
        return self.btn3   -- btn2 被隐藏时直接跳到 btn3
    end
end)
```

#### 实战 2：根据状态选择不同分支

```lua
self.tab_btn:SetFocusChangeDir(MOVE_DOWN, function()
    if self.has_advanced_options then
        return self.advanced_section
    else
        return self.simple_section
    end
end)
```

#### 实战 3：循环列表

```lua
-- 最后一个按钮按 ↓ 回到第一个
btn_last:SetFocusChangeDir(MOVE_DOWN, function()
    return btn_first
end)
```

或直接 `Grid:SetLooping(true, true)` —— 但**只对 Grid 适用**。

### 16.7.7 老手进阶：Tab 键焦点（`next_in_tab_order`）

`SetFocusChangeDir` 第 583-586 行有一个隐藏行为：

```583:586:scripts\widgets\widget.lua
function Widget:SetFocusChangeDir(dir, widget, ...)
    if not next(self.focus_flow) then
        self.next_in_tab_order = widget
    end
```

**第一次调用 `SetFocusChangeDir` 时（`focus_flow` 还为空），把目标记为 `next_in_tab_order`**。

`next_in_tab_order` 的作用——**Tab 键按下时跳到这个 widget**。Klei 的部分输入框序列（如设置中的几个文本框）用它实现"按 Tab 切到下一个输入框"。

**手动设置**：

```lua
input1.next_in_tab_order = input2
input2.next_in_tab_order = input3
input3.next_in_tab_order = input1   -- 循环
```

**注意**——`next_in_tab_order` 只在某些响应 Tab 的 widget（如 TextEdit）中被使用，**不是通用机制**。普通按钮就不会自动响应 Tab。

#### `ClearFocusDirs` —— 清除所有方向

```577:581:scripts\widgets\widget.lua
function Widget:ClearFocusDirs()
    self.focus_flow = {}
	self.focus_flow_args = {}
	self.next_in_tab_order = nil
end
```

**何时用**：

- 切 Tab 时清掉旧布局，重设新布局
- 列表项被回收（虚拟滚动）时清掉

```lua
-- 切 Tab 时重置焦点流
function MyScreen:SwitchTab(name)
    -- 清掉所有按钮的旧 focus_flow
    for _, btn in ipairs(self.all_buttons) do
        btn:ClearFocusDirs()
    end
    -- 只给当前 Tab 的按钮重新连线
    local btns = self.tabs[name].buttons
    for i, b in ipairs(btns) do
        if i > 1 then b:SetFocusChangeDir(MOVE_LEFT, btns[i-1]) end
        if i < #btns then b:SetFocusChangeDir(MOVE_RIGHT, btns[i+1]) end
    end
    btns[1]:SetFocus()
end
```

### 16.7.8 老手进阶：`SetFocusFromChild` 与 `GetDeepestFocus`

#### `SetFocusFromChild`

`SetFocus` 内部调 `parent:SetFocusFromChild(self)`——**通知父节点"我的某个子获得了焦点"**：

```632:657:scripts\widgets\widget.lua
function Widget:SetFocusFromChild(from_child)
    if self.parent == nil and not self.is_screen then
        print("Warning: Widget:SetFocusFromChild is happening on a widget outside of the screen/widget hierachy. This will cause focus moves to fail. Is ", self.name, "not a screen?")
        print(debugstack())
    end
    for k,v in pairs(self.children) do
        if v ~= from_child and v.focus then
            v:ClearFocus()
        end
    end

    if not self.focus then
        self.focus = true
        if self.OnGainFocus then
             self:OnGainFocus()
        end

        if self.ongainfocusfn then
            self.ongainfocusfn()
        end

        if self.parent then
            self.parent:SetFocusFromChild(self)
        end
    end
end
```

**核心要点**：

1. **取消所有兄弟的焦点** —— 同一父下只能有一个 focus child
2. **自己也变成 focus**（沿父链一路向上传递）
3. **递归调父的 SetFocusFromChild** —— 让 Screen 一路上的所有祖先都"知道焦点路径"

**为什么 Screen 需要知道**——Screen 的 `OnFocusMove` 沿"focus 路径"递归到具体按钮，必须每层都标记 `focus = true` 才能正常传递。

#### `GetDeepestFocus`

```592:602:scripts\widgets\widget.lua
function Widget:GetDeepestFocus()
    if self.focus then
        for k,v in pairs(self.children) do
            if v.focus then
                return v:GetDeepestFocus()
            end
        end

        return self
    end
end
```

**返回当前最深的焦点 widget**——比如 Screen → Container → Grid → btn1 都 focus，`Screen:GetDeepestFocus()` 返回 btn1。

**`Screen:OnBecomeInactive` 用它记录 last_focus**（16.3 学过）：

```28:30:scripts\widgets\screen.lua
function Screen:OnBecomeInactive()
	self.last_focus = self:GetDeepestFocus()
end
```

下次 `OnBecomeActive` 时直接 `last_focus:SetFocus()`——焦点恢复到上次那个具体按钮。

#### `TheFrontEnd:OnFocusMove` 顶层路由

源码 `frontend.lua` 第 396-409 行：

```396:409:scripts\frontend.lua
function FrontEnd:OnFocusMove(dir, down)
    if self.focus_locked or self:IsControlsDisabled() then
        return true
	elseif #self.screenstack > 0 then
		if self.screenstack[#self.screenstack]:OnFocusMove(dir, down) then
	   		self:GetSound():PlaySound("dontstarve/HUD/click_mouseover_controller")
			self.tracking_mouse = false
			return true
		elseif self.tracking_mouse and down and self.screenstack[#self.screenstack]:SetDefaultFocus() then
			self.tracking_mouse = false
			return true
		end
	end
end
```

**两个细节**：

1. **如果鼠标在跟踪状态（`tracking_mouse = true`），按方向键先调 `SetDefaultFocus`** —— 让玩家从鼠标切到手柄时焦点能"从无到有"出现。
2. **成功移动焦点后播放音效** —— 这是手柄玩家"听到自己在移动"的反馈。

### 16.7.9 老手进阶：五个常见的焦点陷阱

#### 陷阱一：`focus_flow` 目标是 `Hidden` 的 widget

```lua
-- ❌ 隐藏的 widget 也被设为目标
btn1:SetFocusChangeDir(MOVE_DOWN, btn2)
btn2:Hide()
-- 按 ↓ 时焦点想跳过去但 btn2 不可见 → focus 卡住
```

实际上 `OnFocusMove` 第 84 行已经检查：

```84:84:scripts\widgets\widget.lua
        if dest and dest:IsVisible() and dest.enabled then
```

不可见就不跳——但**焦点也不会自动跳到下一个**——它停在 btn1 上。

**修复**——用函数动态计算：

```lua
btn1:SetFocusChangeDir(MOVE_DOWN, function()
    if btn2:IsVisible() then return btn2 end
    if btn3:IsVisible() then return btn3 end
    return nil
end)
```

#### 陷阱二：`SetFocus` 在 widget 还没挂到 screen 时调

```lua
-- ❌ 还没 AddChild 到 screen
local btn = ImageButton(...)
btn:SetFocus()  -- 警告："Widget outside screen hierarchy"
```

**原因**——`SetFocusFromChild` 检查 `parent == nil and not is_screen`：

```633:636:scripts\widgets\widget.lua
function Widget:SetFocusFromChild(from_child)
    if self.parent == nil and not self.is_screen then
        print("Warning: Widget:SetFocusFromChild is happening on a widget outside of the screen/widget hierachy. This will cause focus moves to fail. Is ", self.name, "not a screen?")
```

焦点系统依赖完整的 widget 树才能工作。

**修复**——先 `AddChild` 再 `SetFocus`：

```lua
local btn = self:AddChild(ImageButton(...))
btn:SetFocus()
```

#### 陷阱三：`focus_forward` 形成循环

```lua
-- ❌ 循环引用
A.focus_forward = B
B.focus_forward = A
A:SetFocus()  -- 死循环！
```

**修复**——`focus_forward` 永远是单向的（叶子节点）。

#### 陷阱四：动态切 Tab 后旧 focus_flow 仍然连着

```lua
-- 切 Tab：把 Tab1 隐藏、Tab2 显示
self.tab1:Hide()
self.tab2:Show()
-- 但 self.tab_btn:SetFocusChangeDir(MOVE_DOWN, self.tab1.first_btn)
-- 旧的 focus_flow 还指向 tab1.first_btn ← 它已经隐藏了
```

**修复**——切 Tab 时 `ClearFocusDirs` + 重设：

```lua
self.tab_btn:ClearFocusDirs()
self.tab_btn:SetFocusChangeDir(MOVE_DOWN, self.tab2.first_btn)
```

#### 陷阱五：`OnGainFocus` 重写时忘调基类

```lua
-- ❌ 没调基类
function MyButton:OnGainFocus()
    self:DoCustomEffect()
    -- 忘了 ImageButton._base.OnGainFocus(self)
end
-- 后果：标准的高亮贴图切换、缩放动画都不再触发
```

**修复**——总是先调基类：

```lua
function MyButton:OnGainFocus()
    MyButton._base.OnGainFocus(self)  -- ★
    self:DoCustomEffect()
end
```

### 16.7.10 老手实战：自适应焦点流的 mod 设置面板

需求：mod 设置面板有动态的 N 个开关 + 关闭按钮——**N 是配置驱动的**——希望焦点能在所有可见开关之间流畅跳转。

```lua
local function BuildSettingsPanel(self, opts)
    self.option_widgets = {}
    for i, opt in ipairs(opts) do
        local w = self.proot:AddChild(MakeOptionRow(opt))
        w:SetPosition(0, 200 - i * 50)
        if not opt.visible then w:Hide() end
        table.insert(self.option_widgets, w)
    end

    self.btn_close = self.proot:AddChild(ImageButton(...))
    self.btn_close:SetPosition(0, -250)
end

local function HookupFocus(self)
    -- 清掉旧的连线
    for _, w in ipairs(self.option_widgets) do
        w:ClearFocusDirs()
    end
    self.btn_close:ClearFocusDirs()

    -- 找出所有可见的选项
    local visible = {}
    for _, w in ipairs(self.option_widgets) do
        if w:IsVisible() then table.insert(visible, w) end
    end

    -- 上下连线（含跳过隐藏）
    for i = 1, #visible do
        if i > 1 then
            visible[i]:SetFocusChangeDir(MOVE_UP, visible[i-1])
        end
        if i < #visible then
            visible[i]:SetFocusChangeDir(MOVE_DOWN, visible[i+1])
        end
    end

    -- 最后一个 → close
    if #visible > 0 then
        visible[#visible]:SetFocusChangeDir(MOVE_DOWN, self.btn_close)
        self.btn_close:SetFocusChangeDir(MOVE_UP, visible[#visible])
    end

    -- 默认焦点
    self.default_focus = visible[1] or self.btn_close
end

-- 切换某个开关后重连
function MyScreen:OnOptionChanged(idx)
    HookupFocus(self)
end
```

**这是焦点系统的高阶用法**——动态布局变化后，**主动重建 focus_flow 网络**保证手柄玩家任何时候都能正常导航。

### 16.7.11 小结

```
                  焦点系统的七层

    1. self.focus 字段                ← 单 widget 是否被选
    2. focus_flow[dir] 表             ← 4 方向的下一焦点
    3. SetFocusChangeDir(dir, w)      ← 手动连线
    4. SetFocus / ClearFocus          ← 状态切换
    5. SetFocusFromChild              ← 父递归记录路径
    6. focus_forward                  ← 委托：自己→子
    7. default_focus                  ← Screen 首个焦点

    ─────────────────────────────────────────
              焦点移动的递归路由

    玩家按 ↓
        ↓
    TheFrontEnd:OnFocusMove(MOVE_DOWN, true)
        ↓
    screenstack[top]:OnFocusMove
        ↓ 递归子直到
    最深焦点 widget:OnFocusMove
        ↓
    检查 focus_flow[MOVE_DOWN]
        ↓ 是函数？
        ├─ 调函数得目标 widget
        └─ 直接是 widget
        ↓
    target:IsVisible() && target.enabled
        ↓
    target:SetFocus()
        ↓
    target.parent:SetFocusFromChild(target)
        ↓ 递归
    一路向上更新所有祖先的 focus
        ↓
    沿路 OnGainFocus / OnLoseFocus 触发
```

| 概念 | 速记 |
|------|------|
| **`self.focus`** | 布尔字段，true = 当前聚焦 |
| **`focus_flow[dir]`** | 4 方向的下一焦点（widget 或函数）|
| **`SetFocusChangeDir(dir, w)`** | 手动连线两个 widget |
| **`SetFocus()` / `ClearFocus()`** | 切换焦点状态，触发回调 |
| **`SetFocusFromChild(child)`** | 子告知父"我得到焦点"，沿父链递归 |
| **`OnFocusMove(dir, down)`** | 焦点移动的递归路由器 |
| **`focus_forward`** | 委托焦点给指定子 widget 或函数 |
| **`default_focus`** | Screen 显示时的首个焦点 |
| **`OnGainFocus / OnLoseFocus`** | 焦点变化的钩子（重写时记得调基类）|
| **`SetOnGainFocus(fn)`** | 不重写而加额外回调 |
| **`GetDeepestFocus()`** | 返回当前最深焦点 widget |
| **`ClearFocusDirs()`** | 清掉 focus_flow（动态布局时用）|
| **`MOVE_UP/DOWN/LEFT/RIGHT`** | 方向常量（1/2/3/4）|

**新手核心三句**：每个 widget 自带 `self.focus` 字段；`SetFocusChangeDir(dir, w)` 在两个按钮间连线；`Grid` 自动连线整网格不用手写。

**进阶核心三句**：`OnFocusMove` 递归子→自己 focus_flow→父 scroll_list 三层冒泡；`focus_forward` 让容器把焦点转给具体子；focus_flow 目标可以是函数、运行时算。

**老手核心三句**：动态布局变化后必须 `ClearFocusDirs` 重连，否则 focus 卡在已隐藏 widget 上；`focus_forward` 不能成环；`OnGainFocus` 重写时永远先调 `_base.OnGainFocus(self)`。

下一节（16.8）我们将讨论**数据收集 UI** ——`Cookbook`（食谱书）、`PlantRegistry`（植物图鉴）、`Scrapbook`（剪贴簿）这三种"边玩边记录"的展示性 UI 的实现机制。


## 16.8 数据收集 UI：Cookbook、PlantRegistry 与 Scrapbook

### 本节导读

16.1-16.7 我们一直在讲"**主动 UI**"——玩家点击、弹出、交互。本节换一个方向：**被动积累型 UI**——这类界面不需要玩家主动操作，它们在玩家游玩过程中**悄悄积累数据**，等玩家打开时展现"已解锁的知识"。

饥荒联机版有三个这样的系统：

| 系统 | 触发方式 | 记录什么 | 全局变量 |
|------|----------|----------|---------|
| **Cookbook（食谱书）** | 用锅煮出食物 / 吃掉食物 | 菜谱配方 + 营养值 | `TheCookbook` |
| **PlantRegistry（植物图鉴）** | 靠近农作物/杂草 | 各生长阶段 + 肥料效果 | `ThePlantRegistry` |
| **Scrapbook（剪贴簿）** | 游戏中见过物体 / 角色检查 | 生物/物品的属性与掉落 | `TheScrapbookPartitions` |

三个系统**架构惊人相似**：
```
游戏事件 → 更新器组件（CookbookUpdater / PlantRegistryUpdater） → 数据类（*Data）→ 持久化
```
UI 打开时**只是"展示"** `*Data` / `TheScrapbookPartitions` 里的内容——不影响游戏逻辑。

> **新手**先看 16.8.1-16.8.4——理解三个系统"积累知识"的触发条件、看懂 `TheCookbook:IsUnlocked` / `ThePlantRegistry:KnowsPlantStage` / `TheScrapbookPartitions:GetLevelFor` 这三个查询入口；**进阶读者**继续看 16.8.5-16.8.7，深入 `CookbookData:AddRecipe` 的排序记忆机制、`PlantRegistry` 的位编码持久化、`ScrapbookPartitions` 的 32 位分桶架构；**老手**跳到 16.8.8-16.8.10，掌握 mod 如何接入 Cookbook（`IsModCookerFood` 豁免机制）、如何让 mod prefab 进 Scrapbook（`scrapbookable` + `scrapbook_proxy`），以及五个常见的"数据写了但 UI 显示不对"的坑。

读完本节，你能**不靠猜**地理解为什么有些 mod 食物进了食谱书有些没进、为什么 Scrapbook 里有两个"见过等级"、以及怎么让自己的 mod 物品出现在这三个展示页里。

---

### 16.8.1 快速入门：三个系统的整体架构——数据层 + 更新器 + UI

在深入每个系统之前，先看**共同骨架**：

#### 第一步："数据层"—— `*Data` 类

每个系统都有一个纯数据类，负责**存储"玩家已知"的知识**和**持久化**：

| 文件 | 全局实例 | 存储键 |
|------|----------|--------|
| `scripts/cookbookdata.lua` | `TheCookbook` | `"cookbook"` |
| `scripts/plantregistrydata.lua` | `ThePlantRegistry` | `"plantregistry"` |
| `scripts/scrapbookpartitions.lua` | `TheScrapbookPartitions` | 分桶 16 个 key |

持久化全部走 `TheSim:SetPersistentString(key, str, false)` / `TheSim:GetPersistentString`——这是**客户端本地存档**，**不随世界存档保存**（服务器上不调用）。

#### 第二步："更新器组件"—— 挂在玩家上的 Component

每个系统都有一个组件，挂在 **Player** 身上：

```lua
-- scripts/prefabs/player_common.lua（示意）
inst:AddComponent("cookbookupdater")      -- Cookbook
inst:AddComponent("plantregistryupdater") -- PlantRegistry
```

组件的职责：**捕获游戏事件 → 调用数据层的 Learn/Add 方法 → 可选地向客户端 RPC 同步**。

Scrapbook 稍有不同——它没有独立更新器组件，更新入口直接在 `TheScrapbookPartitions` 的公共方法上。

#### 第三步："UI 层"

| 系统 | 入口 |
|------|------|
| Cookbook | 玩家持有 `cookbook` 物品并"阅读"它，或在 HUD 点击食谱按钮 |
| PlantRegistry | HUD 上的植物图鉴按钮 |
| Scrapbook | 按 `Tab` → 点击剪贴簿按钮，或游戏内解锁后弹出 Toast |

> **新手记忆**：三层结构——数据层存知识、更新器组件触发学习、UI 层展示。**数据存在客户端本地，不是世界存档的一部分**——服务器上完全不读写这些数据。

---

### 16.8.2 快速入门：Cookbook——吃过才知道怎么做

Cookbook 记录两件事：
1. **用什么食材做出了这道菜**（`AddRecipe`，烹饪时触发）
2. **吃过这道菜**（`LearnFoodStats`，进食时触发）

#### 第一步：更新器的两个方法

```9:28:scripts/components/cookbookupdater.lua
local CookbookUpdater = Class(function(self, inst)
    self.inst = inst

	self.cookbook = require("cookbookdata")()
	inst:ListenForEvent("playeractivated", onplayeractivated)
end)

function CookbookUpdater:LearnRecipe(product, ingredients)
	if product ~= nil and ingredients ~= nil then
		local updated = self.cookbook:AddRecipe(product, ingredients)
		--print("CookbookUpdater:LearnRecipe", product, updated, unppack(ingredients))

		-- Servers will only tell the clients if this is a new recipe in this world
		-- Since the servers do not know the client's actual cookbook data, this is the best we can do for reducing the amount of data sent
		if updated and (TheNet:IsDedicated() or (TheWorld.ismastersim and self.inst ~= ThePlayer)) and self.inst.userid then
			--can't send tables via rpc, so unpack the table before sending.
			SendRPCToClient(CLIENT_RPC.LearnRecipe, self.inst.userid, product, unpack(ingredients))
		end
	end
end
```

- **`LearnRecipe(product, ingredients)`** —— product 是食物名，ingredients 是食材名列表。在锅煮完后由 `cookpot.lua` 调用
- **`LearnFoodStats(product)`** —— product 是食物名。在玩家吃下食物后由 `eater.lua` 调用

#### 第二步：什么食物才能被记录？

`CookbookData:IsValidEntry` 检查食物**是否在 `cooking.cookbook_recipes` 里**——只有"能在锅里做出来"的食物才算有效条目。

```131:138:scripts/cookbookdata.lua
function CookbookData:IsValidEntry(product)
	for cooker, recipes in pairs(cooking.cookbook_recipes) do
		if recipes[product] ~= nil then
			return true
		end
	end
	return false
end
```

原味食物（直接摘的浆果、肉）不在 `cookbook_recipes` 里——**进不了食谱书**。

#### 第三步：两种查询

```lua
-- 检查这道菜是否已解锁（只要煮过或吃过）
TheCookbook:IsUnlocked("meatballs")  -- 返回解锁数据 or nil

-- 检查是否吃过（解锁了营养值数据）
local entry = TheCookbook:IsUnlocked("meatballs")
if entry and entry.has_eaten then
    -- 玩家知道营养数值
end
```

> **新手记忆**：Cookbook 有两级——**煮过**解锁配方栏（`recipes`）、**吃过**解锁营养值（`has_eaten`）。这两步都需要才能在食谱书里看到完整信息。

---

### 16.8.3 快速入门：PlantRegistry——靠近就能解锁

PlantRegistry 记录**玩家观察到的植物生长阶段**和**使用过的肥料**。

#### 第一步：三个更新器方法

```16:48:scripts/components/plantregistryupdater.lua
function PlantRegistryUpdater:LearnPlantStage(plant, stage)
    if plant and stage then
		local updated = self.plantregistry:LearnPlantStage(plant, stage)

		if updated and TheFocalPoint.entity:GetParent() == self.inst then
			TheFocalPoint.SoundEmitter:PlaySound("dontstarve/HUD/get_gold")
		end
		-- ...
		if updated and (TheNet:IsDedicated() or ...) and self.inst.userid then
			SendRPCToClient(CLIENT_RPC.LearnPlantStage, self.inst.userid, plant, stage)
		end
	end
end
```

- **`LearnPlantStage(plant, stage)`** —— plant 是植物名（如 `"asparagus"`），stage 是整数阶段编号（1 = 第一阶段 seedling，以此类推）。由农田相关代码在玩家靠近时调用
- **`LearnFertilizer(fertilizer)`** —— 使用肥料时记录
- **`TakeOversizedPicture(plant, weight, beardskin, beardlength)`** —— 巨型作物摄影时调用（植物图鉴的"照片"功能）

#### 第二步：三种查询

```17:40:scripts/plantregistrydata.lua
function PlantRegistryData:GetKnownPlants()
	return self.plants
end

function PlantRegistryData:GetKnownPlantStages(plant)
	if self.plants[plant] then
		return self.plants[plant]
	end
	return {}
end

-- ...

function PlantRegistryData:KnowsPlantStage(plant, stage)
	if self.plants[plant] then
		return self.plants[plant][stage] == true
	end
	return false
end
```

- `ThePlantRegistry:KnowsPlantStage("asparagus", 1)` —— 是否知道芦笋第 1 阶段
- `ThePlantRegistry:KnowsFertilizer("poop")` —— 是否用过粪便作肥料
- `ThePlantRegistry:HasOversizedPicture("asparagus")` —— 是否拍过巨型芦笋

#### 第三步：完成度进度

```76:99:scripts/plantregistrydata.lua
function PlantRegistryData:GetPlantPercent(plant, plantregistryinfo)
	local totalstages = 0
	local knownstages = 0
	-- ...
	return knownstages / totalstages
end
```

这个值是 UI 里那个圆形进度条的来源——只统计 `growing` 和 `fullgrown` 阶段，不包括种子阶段。

> **新手记忆**：PlantRegistry 靠**距离触发**（不需要主动操作）——玩家走到农田附近，游戏自动调 `LearnPlantStage`；只有 `PLANT_DEFS` / `WEED_DEFS` 里定义的植物才会被记录。

---

### 16.8.4 快速入门：Scrapbook——见过就记录，检查解锁详情

Scrapbook（剪贴簿）有**两个解锁等级**：

| 等级 | 触发方式 | 意义 |
|------|----------|------|
| **Level 1（见过）** | 游戏中见到该物体（`SetSeenInGame`）| 剪贴簿里有条目，但详情是灰色问号 |
| **Level 2（检查过）** | 特定角色对其使用"检查"动作（`SetInspectedByCharacter`）| 详情完全解锁，包括属性、掉落 |

#### 第一步：三个核心查询

```304:322:scripts/scrapbookpartitions.lua
function ScrapbookPartitions:GetLevelFor(thing)
    thing = self:RedirectThing(thing)

    if type(thing) ~= "string" then
        return 0
    end

    local hashed = hash(thing)
    local data = self.storage[hashed]

    if data == nil then
        return 0 -- If a thing is unknown it is level 0.
    end

    if band(data, LOOKUP_LIST_MASK) == 0 then
        return 1 -- If a thing has been seen but not inspected it is level 1.
    end

    -- ...
    return 2 -- If a thing has been seen and inspected once it is level 2.
end
```

```lua
TheScrapbookPartitions:GetLevelFor("evergreen")       -- 0 / 1 / 2
TheScrapbookPartitions:WasSeenInGame("deerclops")     -- bool
TheScrapbookPartitions:WasInspectedByCharacter("deerclops", "wilson")  -- bool
```

#### 第二步：数据来源 —— 自动生成的 `scrapbookdata.lua`

Scrapbook 里每个条目的**显示内容**（血量、伤害、掉落）来自 `scripts/screens/redux/scrapbookdata.lua`：

```2:9:scripts/screens/redux/scrapbookdata.lua
-- AUTOGENERATED FROM d_createscrapbookdata()   < debugcommands.lua >

return {
    abigail = {name="abigail", tex="abigail.tex", type="creature", prefab="abigail", speechstatus={"LEVEL1", "1"}, health=150, damage="15-40", build="ghost_abigail_build", bank="ghost", anim="idle"},
    abigail_flower = {name="abigail_flower", tex="abigail_flower.tex", type="item", prefab="abigail_flower", ...},
    -- ...
```

注意文件头的 `AUTOGENERATED FROM d_createscrapbookdata()` ——这个文件**不是手写的**，是由调试命令自动生成。每条记录里的 `type` 字段决定分类（`"creature"`, `"item"`, `"thing"`, `"food"`, `"giant"`, `"POI"` 等）。

#### 第三步：哪些 prefab 进了 Scrapbook？

`scripts/scrapbook_prefabs.lua` 是一份白名单——只有在这里列出的 prefab，才有可能被 `SetSeenInGame` 记录：

```1:10:scripts/scrapbook_prefabs.lua
local PREFABS =
{
    ["statue_marble_muse"] = true,
    ["statue_marble_pawn"] = true,

    ["abigail"] = true,
    ["wobybig"] = true,
    ["chester"] = true,
    -- ...
```

`scrapbook_page.lua`（剪贴簿页物品）掉落时会读这张表，随机解锁若干条目（`TryToTeachScrapbookData_Random`）。

> **新手记忆**：Scrapbook 有两级解锁——**Level 1 = 见过**（条目变白）、**Level 2 = 检查过**（完整属性）；条目来自自动生成的 `scrapbookdata.lua`，不是每个 prefab 都在里面。

---

### 16.8.5 进阶：CookbookData 的记忆排序机制与双路持久化

#### 第一步：`AddRecipe` 的配方槽排序

`CookbookData` 为每道菜最多保存 `MAX_RECIPES = 6` 条不同的配方记录，并实现了**"最近使用靠前"**的排序逻辑：

```3:3:scripts/cookbookdata.lua
local MAX_RECIPES = 6
```

```191:237:scripts/cookbookdata.lua
function CookbookData:AddRecipe(product, ingredients)
	if product == nil or ingredients == nil then
		print("Invalid cookbook recipe:", product, unpack(ingredients or {"(empty)"}))
		return
	elseif not self:IsValidEntry(product) then
		--silent fail
		return false
	end

	ingredients = self:RemoveCookedFromName(ingredients)
	table.sort(ingredients)

	local updated = false

	local preparedfood = UnlockPreparedFood(self, product)
	if preparedfood.recipes == nil then
		preparedfood.recipes = {ingredients}

		self.newfoods[product] = true
		updated = true
	else
		local recipes = preparedfood.recipes
		local known_index = IsKnownRecipe(recipes, ingredients)
		if known_index ~= nil then
			if known_index > 2 then
				table.remove(recipes, known_index)
				table.insert(recipes, 1, ingredients)
				updated = true
			end
		else
			if #recipes >= MAX_RECIPES then
				table.remove(recipes, #recipes)
			end
			table.insert(recipes, 1, ingredients)
			self.newfoods[product] = true
			updated = true
		end
	end

	if updated and self.save_enabled then
		if not cooking.IsModCookerFood(product) and not TheNet:IsDedicated() then
			TheInventory:SetCookBookValue(product, EncodeCookbookEntry(preparedfood))
		end
		self:Save(true)
	end

	return updated
end
```

**核心逻辑**：
1. 食材名先经过 `RemoveCookedFromName` 去掉 `cooked_` / `_cooked` 前后缀 → 再 `table.sort` 字母排序——**防止"肉+浆果"和"浆果+肉"被视为不同配方**
2. 如果这个配方已知且排名在第 3 位以内（`known_index <= 2`），不触发更新；如果在第 3 位之后，把它**移到最前**（LRU 行为）
3. 如果是全新配方且槽位满了，踢掉最末尾那条；新配方插到最前

这就是食谱书里"最近使用的配方靠前"背后的实现。

#### 第二步：双路持久化

CookbookData 有两个持久化途径，互为备份：

| 途径 | 函数 | 存储位置 |
|------|------|----------|
| **本地** | `TheSim:SetPersistentString("cookbook", ...)` | 本地文件（`persistent_string_num.lua`）|
| **在线** | `TheInventory:SetCookBookValue(product, encoded)` | Klei 账号服务器 |

`CookbookData:Load` 先读本地文件，失败时调 `ApplyOnlineProfileData` 从账号服务器恢复：

```26:66:scripts/cookbookdata.lua
function CookbookData:Load()
	self.preparedfoods = {}
    self.filters = {}
    local needs_save = false
    local really_bad_state = false
	TheSim:GetPersistentString("cookbook", function(load_success, data)
		if load_success and data ~= nil then
			local status, recipe_book = pcall( function() return json.decode(data) end )
		    if status and recipe_book then
                -- ...
			else
                really_bad_state = true
				print("Failed to load the cookbook!", status, recipe_book)
			end
		end
	end)
    if really_bad_state then
        print("Trying to apply online cache of cookbook data..")
        if self:ApplyOnlineProfileData() then
            -- 恢复成功
        end
    end
end
```

**关键**：`save_enabled` 标志位控制是否真正写入——只有**本机玩家**且**已激活（playeractivated）**后才允许写入：

```1:6:scripts/components/cookbookupdater.lua
local function onplayeractivated(inst)
	local self = inst.components.cookbookupdater
	if not TheNet:IsDedicated() and inst == ThePlayer then
		self.cookbook = TheCookbook
		self.cookbook.save_enabled = true
	end
end
```

这个设计**避免了"服务器帮客户端写档"**——服务端只通过 RPC 告知客户端"有新菜谱"，客户端自己写本地存档。

#### 第三步：编码格式

本地存档用 JSON；在线存档用紧凑的字符串编码：

```68:87:scripts/cookbookdata.lua
local function DecodeCookbookEntry(value)
	local data = {recipes = {}}
	local recipes = string.split(value, "|")
	for i = 1, #recipes-1 do
		table.insert(data.recipes, string.split(recipes[i], ","))
	end
	data.has_eaten = recipes[#recipes] == "true"
	return data
end

local function EncodeCookbookEntry(entry)
	local str = ""
	if entry.recipes ~= nil then
		for i = 1, math.min(MAX_RECIPES, #entry.recipes) do
			local r = entry.recipes[i]
			str = str .. table.concat(r, ",") .. "|"
		end
	end
	str = str .. (entry.has_eaten and "true" or "false")
	return str
end
```

**格式**：`ingredient1,ingredient2|ingredient3,ingredient4|true`——每条配方用 `,` 分隔食材，配方间用 `|` 分隔，最后一段是 `has_eaten` 标志。

> **进阶记忆**：`AddRecipe` 内部先规范化食材（去 cooked 前缀 + 字母排序），再做 LRU 更新；双路持久化让账号换机器不丢菜谱；`save_enabled` 标志确保只有本机玩家才写档。

---

### 16.8.6 进阶：PlantRegistryData 的阶段追踪与位编码

#### 第一步：`LearnPlantStage` 的完整流程

```213:248:scripts/plantregistrydata.lua
function PlantRegistryData:LearnPlantStage(plant, stage)
	if plant == nil or stage == nil then
		print("Invalid plant or stage", plant, stage)
		return
	end

	local def = PLANT_DEFS[plant] or WEED_DEFS[plant]

	local previouspercent = def and def.plantregistryinfo and self:GetPlantPercent(plant, def.plantregistryinfo) or 0

	local stages = UnlockPlant(self, plant)
	local updated = stages[stage] == nil
	stages[stage] = true

	if updated and self.save_enabled then
		if def and not def.modded and not TheNet:IsDedicated() then
			TheInventory:SetPlantRegistryValue(plant, EncodePlantRegistryStages(stages))
		end
		local currentpercent = def and def.plantregistryinfo and self:GetPlantPercent(plant, def.plantregistryinfo) or 0

		if previouspercent < 1 and currentpercent >= 1 and def.plantregistrysummarywidget then
			self:SetLastSelectedCard(plant, "summary")
		else
			local higheststage = 0
			for k in pairs(stages) do
				higheststage = math.max(higheststage, k)
			end
			if higheststage == stage then
				self:SetLastSelectedCard(plant, stage)
			end
		end
		self:Save(true)
	end

	return updated
end
```

**关键细节**：
- `def.modded` 标志——如果是 mod 植物，**不**调用 `TheInventory:SetPlantRegistryValue`，只写本地存档
- `previouspercent` 和 `currentpercent` 比对——当完成度**从 < 1 升到 ≥ 1（完整解锁）**时，自动把最后选中卡片设为 `"summary"`（植物图鉴里的"总结页"）；否则记住玩家解锁的最高阶段
- 同样依赖 `save_enabled` 防止服务端写档

#### 第二步：阶段的位编码

在线存档用十六进制字符串保存哪些阶段已知：

```137:154:scripts/plantregistrydata.lua
local function DecodePlantRegistryStages(value)
	local bitstages = tonumber(value, 16)
	local stages = {}
	for i = 1, 8 do
		if checkbit(bitstages, 2^(i-1)) then
			stages[i] = true
		end
	end
	return stages
end

local function EncodePlantRegistryStages(stages)
	local bitstages = 0
	for i in pairs(stages) do
		bitstages = setbit(bitstages, 2^(i-1))
	end
	return string.format("%x", bitstages)
end
```

**设计**：8 位 bit，每位对应一个阶段（stage 1~8）。8 种阶段用一个 hex 字符串（最多 2 个字符）存储——极度紧凑，适合写入 Klei 账号服务器。

比如 `stages = {1=true, 3=true}` → `bitstages = 0b00000101 = 0x5` → 存 `"5"`。

#### 第三步：巨型作物照片的特殊持久化

```277:327:scripts/plantregistrydata.lua
function PlantRegistryData:TakeOversizedPicture(plant, weight, player, beardskin, beardlength)
	-- ...

	picture.weight = weight
	picture.player = player.prefab
	local clienttable = TheNet:GetClientTableForUser(player.userid)
	picture.clothing = {
		body = clienttable.body_skin,
		hand = clienttable.hand_skin,
		legs = clienttable.legs_skin,
		feet = clienttable.feet_skin,
	}
	picture.base = clienttable.base_skin
	-- ...
	if def and not def.modded and not TheNet:IsDedicated() then
		TheInventory:SetPlantRegistryValue("oversized_"..plant, TheSim:ZipAndEncodeString(DataDumper(picture, nil, true)))
	end
```

**照片数据**包含：重量、拍摄角色、当前穿的皮肤（body/hand/legs/feet/base）、胡子数据——这些在 UI 里用来**渲染一张带角色的"纪念照"**。数据用 `ZipAndEncodeString` 压缩再 base64 编码，才写入在线存档。

> **进阶记忆**：PlantRegistry 阶段用 8 bit 编码；mod 植物设 `def.modded = true` 可豁免在线同步；解锁完整图鉴时自动跳到 "summary" 卡片；照片数据里含皮肤信息，渲染时可还原玩家外观。

---

### 16.8.7 进阶：ScrapbookPartitions 的位存储与分桶架构

Scrapbook 的存储设计最为复杂，因为它需要**同时记录"哪些角色检查过哪些物体"**，并且**不能一次把所有数据推给 Klei 后端**（条目太多，大约 1000+ 个 prefab）。

#### 第一步：32 位存储格式

每个 prefab 对应一个 32 位整数：

```16:27:scripts/scrapbookpartitions.lua
local FLAGS = { -- DO NOT REARRANGE ORDER OR CHANGE VALUES
    ["VIEWED_IN_SCRAPBOOK"] = 0x00000001, -- bit 0
}

local LOOKUP_LIST = { -- DO NOT REARRANGE ORDER OR CHANGE VALUES
    ["wilson"]       = 0x00000100, --  bit 8
    ["willow"]       = 0x00000200, --  bit 9
    ["wolfgang"]     = 0x00000400, --  bit 10
    -- ...（18 个角色，占 bit 8~31）
    ["wanda"]        = 0x02000000, --  bit 25
}
```

**布局**：

```
Bit 31 ... Bit 8         Bit 7 ... Bit 1   Bit 0
[角色检查位 × 18]          [保留]            [VIEWED_IN_SCRAPBOOK]
```

- **Bit 0 = `VIEWED_IN_SCRAPBOOK`**：玩家是否已在剪贴簿界面点击查看过（用于"新条目"红点）
- **Bit 8~25 = 角色位**：Wilson 检查过→ bit 8 置 1，Willow 检查过→ bit 9 置 1……

只要**任意一个角色位为 1**，`GetLevelFor` 就返回 2（完整解锁）；所有角色位都为 0 但条目在 `storage` 里（即使为 0），就是 Level 1（已见过）。

**mod 角色的特殊处理**：

```244:247:scripts/scrapbookpartitions.lua
    if table.contains(MODCHARACTERLIST, character) then
        character = "wilson" -- Modded characters do not save instead use Wilson as a fallback.
    end
```

mod 自定义角色**统一归入 Wilson 槽**——避免需要扩展 32 位表。

#### 第二步：分桶存储

为了不一次上传太多数据，`ScrapbookPartitions` 把 hash 空间切成 **16 个桶**：

```70:70:scripts/scrapbookpartitions.lua
local BUCKETS_MASK = 0xF -- 16 buckets 0 to 15
```

```88:90:scripts/scrapbookpartitions.lua
local function GetBucketForHash(hashed)
    return band(BUCKETS_MASK, hashed)
end
```

每个桶独立读写——某个桶 dirty 时，只上传该桶的数据，不需要重传所有记录。

#### 第三步：四个核心方法对比

| 方法 | 作用 | 调用时机 |
|------|------|---------|
| `SetSeenInGame(thing)` | 置 Level 1（在游戏中见到） | 实体进入玩家视野时 |
| `WasSeenInGame(thing)` | 查询是否 Level ≥ 1 | UI 渲染、检查逻辑 |
| `SetInspectedByCharacter(thing, char)` | 置 Level 2（角色检查） | 玩家执行检查动作时 |
| `WasInspectedByCharacter(thing, char)` | 查询特定角色是否检查过 | UI 显示角色图标 |

注意 `SetInspectedByCharacter` 还有一个副作用：

```295:295:scripts/scrapbookpartitions.lua
    self:SetViewedInScrapbook(thing, false) -- Mark as new.
```

每次角色检查后，`VIEWED_IN_SCRAPBOOK` 位被**清零**——下次打开剪贴簿，这个条目会显示"新"红点，直到玩家在 UI 里点击它（`SetViewedInScrapbook(true)`）才清除。

> **进阶记忆**：每个 prefab 的数据是 1 个 32 位整数：bit 0 是"已看过 UI"、bit 8-25 是各角色检查位；mod 角色合并到 Wilson 位；16 桶分片避免全量上传。

---

### 16.8.8 老手：给 mod 食物接入 Cookbook

#### 第一步：让食物通过 `IsValidEntry` 检查

`IsValidEntry` 检查食物是否在 `cooking.cookbook_recipes` 里，关键是**先用官方 API 注册菜谱**。假设 mod 在自定义锅（`mod_pot`）里做 `mod_dish`：

```lua
-- modmain.lua / cooking 相关文件
-- 用 AddCookerRecipe 向自定义锅注册食谱
AddCookerRecipe("mod_pot", {
    name = "mod_dish",
    -- ...
})
```

注册成功后 `cooking.cookbook_recipes["mod_pot"]["mod_dish"]` 存在，`IsValidEntry("mod_dish")` 返回 `true`。

#### 第二步：`IsModCookerFood` 豁免机制

```37:39:scripts/cooking.lua
local function IsModCookerFood(prefab)
	return not official_foods[prefab] -- note: we cannot test against cookbook_recipes[MOD_COOKBOOK_CATEGORY] because if the mod is unloaded, it would return true
end
```

`official_foods` 是在 `cooking.lua` 加载时把所有**内置食物**加进去的表——**mod 食物默认不在里面**，`IsModCookerFood` 返回 `true`。

这个标志在 `AddRecipe` / `LearnFoodStats` 里的含义：

```230:234:scripts/cookbookdata.lua
	if updated and self.save_enabled then
		if not cooking.IsModCookerFood(product) and not TheNet:IsDedicated() then
			TheInventory:SetCookBookValue(product, EncodeCookbookEntry(preparedfood))
		end
		self:Save(true)
	end
```

mod 食物：**只写本地存档，不上传账号服务器**——避免 mod 卸载后在线数据留垃圾。本地存档照常写，所以玩家在有 mod 的情况下重开游戏，食谱书里的 mod 食物记录不会丢。

#### 第三步：触发时机

mod 锅煮完食物时，需要手动调：

```lua
-- 在 cookpot 类组件的 DoneStewing 等回调中
if doer.components.cookbookupdater then
    doer.components.cookbookupdater:LearnRecipe(product, ingredients)
end

-- 在 eater 组件的 OnEat 回调中
if eater.inst.components.cookbookupdater then
    eater.inst.components.cookbookupdater:LearnFoodStats(product)
end
```

官方 `cookpot.lua` 和 `eater.lua` 已经有这些调用。如果 mod 完全复用官方组件，**无需额外操作**；如果实现了自定义烹饪系统，需要自行调用上述方法。

> **老手记忆**：mod 食物只需通过 `AddCookerRecipe` 注册菜谱即可自动进入食谱书；`IsModCookerFood` 豁免在线同步，本地存档正常写；自定义烹饪系统需手动调 `LearnRecipe` / `LearnFoodStats`。

---

### 16.8.9 老手：给 mod 实体加入 Scrapbook（scrapbookable + proxy 模式）

Scrapbook 的数据来源是**自动生成的** `scrapbookdata.lua`，mod 无法直接修改它。但游戏提供了两种途径接入。

#### 途径一：`scrapbookable` 组件（角色检查触发）

为实体添加 `scrapbookable` 组件，当玩家对其执行"检查"动作时，会调用 `Scrapbookable:Teach(doer)`：

```1:16:scripts/components/scrapbookable.lua
local Scrapbookable = Class(function(self, inst)
    self.inst = inst
end)

function Scrapbookable:SetOnTeachFn(fn)
    self.onteach = fn
end

function Scrapbookable:Teach(doer)
    if self.onteach ~= nil then
        self.onteach(self.inst, doer)
    end

    return true
end
```

`onteach` 回调里调用 `TheScrapbookPartitions:SetInspectedByCharacter(inst.prefab, doer.prefab)` 即可触发等级升为 Level 2。但**前提是** `scrapbookdata.lua` 里有这个 prefab 的条目。

#### 途径二：`scrapbook_proxy`——让 mod 实体重定向到已有条目

```139:144:scripts/scrapbookpartitions.lua
function ScrapbookPartitions:RedirectThing(thing) -- Use this wrapper function to redirect an object into a string if available.
    if EntityScript.is_instance(thing) then
        return thing.scrapbook_proxy or thing.prefab
    end

    return thing
end
```

如果实体身上有 `scrapbook_proxy` 属性，则把它"当作" `scrapbook_proxy` 这个名字去查询——这允许 mod 自定义实体**共享一个官方条目**的 Scrapbook 数据。

例如，mod 新增一种"变种猪人"，可以：

```lua
-- 在 prefab fn 里：
inst.scrapbook_proxy = "pigman"   -- 在 Scrapbook 里显示官方猪人的数据
```

这样玩家检查 mod 猪人，实际解锁的是 `"pigman"` 的 Level 2 数据（内置条目），界面上显示猪人的统计数据。

#### 途径三：让 mod prefab 进 `scrapbookdata`

最彻底的方式是**在 mod 的 modmain.lua 里向 `scrapbookdata` 表插入条目**。由于 `scrapbookdata.lua` 是 require 加载的，在 mod 代码执行时可以直接修改：

```lua
-- modmain.lua（参考做法，非官方 API）
local scrapbookdata = require("screens/redux/scrapbookdata")
scrapbookdata["my_mod_creature"] = {
    name = "my_mod_creature",
    tex  = "my_mod_creature.tex",
    type = "creature",
    prefab = "my_mod_creature",
    health = 200,
    damage = "15-30",
    build = "my_creature_build",
    bank  = "my_creature",
    anim  = "idle",
}
```

注意这是**非官方方式**，Klei 没有提供 `AddScrapbookEntry` 这样的 mod API。字段的含义参考 `scrapbookdata.lua` 里其他条目的格式。

> **老手记忆**：mod 接入 Scrapbook 有三条路——① `scrapbookable` 组件响应检查动作；② `scrapbook_proxy` 重定向到内置条目；③ 直接向 `scrapbookdata` 表插入自定义条目（非官方）。前两条稳定，第三条需要配合 scrapbook UI 支持 mod 的 tex/build/bank/anim 格式。

---

### 16.8.10 老手：五个常见坑

#### 坑 1：mod 食物进不了食谱书——`IsValidEntry` 返回 false

**症状**：吃了 mod 食物，食谱书里没有条目。

**原因**：食物没有通过 `AddCookerRecipe` 注册，`cookbook_recipes` 里找不到，`IsValidEntry` 返回 `false`，`AddRecipe` / `LearnFoodStats` 静默失败（`return false`）。

**修复**：确保 mod 用官方方式注册菜谱。如果是完全自定义烹饪系统，可以用 `cooking.cookbook_recipes["MY_COOKER"] = {}` 的方式手动插入，但注意命名不要与官方冲突。

#### 坑 2：Cookbook 本地数据丢失——忘记 `save_enabled`

**症状**：客户端重启后食谱书清空，或在 dedicated server 上食谱书始终为空。

**原因**：`save_enabled` 只在 `playeractivated` 事件后被设为 `true`——如果代码在玩家激活之前就调用 `AddRecipe`，写入会被跳过。

**修复**：确保烹饪/进食回调在玩家激活**之后**触发；dedicated server 完全没有本地存档，这是正常行为——不是 bug。

#### 坑 3：PlantRegistry 阶段解锁触发了但 UI 没变——mod 植物没设定义

**症状**：调了 `LearnPlantStage`，`GetLevelFor` 或 `KnowsPlantStage` 返回正确，但 PlantRegistry UI 里看不到植物。

**原因**：PlantRegistry **UI 只显示 `PLANT_DEFS` / `WEED_DEFS` 里定义的植物**。mod 植物如果没有在这两个表里注册，数据存了但 UI 找不到对应的卡片配置，不渲染。

**修复**：在 `PLANT_DEFS` 里为 mod 植物注册完整的 `plantregistryinfo`（每个阶段的 growing/fullgrown/learnseed 等字段）。

#### 坑 4：Scrapbook 角色检查了但等级是 1 而非 2——角色名写错

**症状**：`SetInspectedByCharacter(thing, "my_char")` 调了，`WasInspectedByCharacter` 返回 `false`。

**原因**：`LOOKUP_LIST` 里没有 `"my_char"`——mod 角色会被重定向到 `"wilson"`，但**只有 `MODCHARACTERLIST` 里明确列出的角色才走 wilson 回退**；如果 mod 角色连 `MODCHARACTERLIST` 都不在，`LOOKUP_LIST[character]` 为 nil，函数直接 `return`，不写任何数据。

**修复**：mod 角色应在注册时正确添加到 `MODCHARACTERLIST`；使用 `TheScrapbookPartitions:SetInspectedByCharacter` 时，检查角色名拼写。

#### 坑 5：`RedirectThing` 没有按预期重定向——`scrapbook_proxy` 赋值时机错

**症状**：mod 实体设了 `inst.scrapbook_proxy = "pigman"` 但 Scrapbook 里没有解锁。

**原因**：`RedirectThing` 接受的是 EntityScript 实例**或**字符串。如果传入的是 **string 类型的 prefab 名**（而非 inst），`scrapbook_proxy` 不会被读取——重定向只在传入实体实例时才生效。

**修复**：在调用 `SetSeenInGame(inst)` / `SetInspectedByCharacter(inst, char)` 时，传入**实体实例本身**，而不是 `inst.prefab`：

```lua
-- 正确
TheScrapbookPartitions:SetSeenInGame(inst)

-- 错误（string 路径不读 scrapbook_proxy）
TheScrapbookPartitions:SetSeenInGame(inst.prefab)
```

---

### 16.8 小结

```
Cookbook                    PlantRegistry               Scrapbook
────────                    ─────────────               ─────────
CookbookData                PlantRegistryData           ScrapbookPartitions
  preparedfoods{            plants{                       storage{
    product: {                plant: {stage:true}           hash(prefab): 32bit
      recipes:[[]]            fertilizers{fer:true}           bit0: viewed_in_ui
      has_eaten:bool          pictures{plant:{...}}           bit8-25: char mask
    }                       }                             }
  }
    ↑                          ↑                              ↑
CookbookUpdater            PlantRegistryUpdater          scrapbookable +
  LearnRecipe()              LearnPlantStage()            proximity check
  LearnFoodStats()           LearnFertilizer()
    ↑(RPC)                     ↑(RPC)
Server → Client            Server → Client
(仅 new entry 时)           (仅 new entry 时)

持久化：
  本地：SetPersistentString  SetPersistentString       分桶 SetPersistentString
  在线：TheInventory           TheInventory              TheInventory
       SetCookBookValue        SetPlantRegistryValue      (按桶)
       (官方食物)               (官方植物)
       IsModCookerFood → 跳    def.modded → 跳
```

**新手核心三句**：Cookbook 两级解锁——煮过解锁配方、吃过解锁营养；PlantRegistry 靠近自动触发，看进度用 `GetPlantPercent`；Scrapbook Level 1 = 见过、Level 2 = 角色检查过。

**进阶核心三句**：`AddRecipe` 内部做食材规范化（去 cooked 前缀 + 字母排序）+ LRU 排序，最多 6 条记录；PlantRegistry 阶段用 8 bit 十六进制编码节省在线存储；Scrapbook 每条记录是 1 个 32 位整数，低位是 UI 标志，高位是 18 角色检查位。

**老手核心三句**：mod 食物通过 `AddCookerRecipe` 注册菜谱即可进入食谱书，在线同步被 `IsModCookerFood` 豁免（只写本地）；mod 实体接入 Scrapbook 最稳是 `scrapbook_proxy` 重定向到官方条目；`SetInspectedByCharacter` 传**实体实例**而非 prefab 字符串，否则 proxy 不生效。

下一节（16.9）我们将讨论**控制台命令与调试面板**——如何在游戏内用 `c_` 前缀命令快速验证逻辑、如何为 mod 注册自己的控制台命令、以及 DebugDraw 系统的使用方式。

## 16.9 控制台命令与调试面板

### 本节导读

会写组件、会做 UI——但代码写完怎么**快速验证**？靠 print 一行行看日志太慢；靠重开游戏太麻烦。饥荒内置的**控制台（Console）系统**解决了这个问题：它是一个**实时 Lua 执行环境**，允许你在游戏运行时输入任意代码、调用任意函数，立竿见影地观察结果。

本节讲三件事：

1. **控制台界面与执行模式**——按什么键打开、本地/远程执行的区别、历史记录
2. **`c_` 前缀命令**——Klei 内置的 50+ 个调试快捷函数，从生成物品到传送到控制时间
3. **mod 扩展控制台**——如何在 modmain.lua 里加全局函数让它在控制台可用、如何用 `customcommands.lua` 持久化个人快捷命令

> **新手**先看 16.9.1-16.9.3——打开控制台、用 `c_spawn` / `c_give` 测试物品、用 `c_sel` 选中实体；**进阶读者**继续看 16.9.4-16.9.6，深入本地/远程执行的差异（什么情况下 `c_spawn` 不生效）、`customcommands.lua` 的加载机制、mod 如何向控制台暴露自己的调试函数；**老手**跳到 16.9.7-16.9.9，了解 `d_` 前缀命令、`debugkeys.lua` 快捷键系统、`DebugMenuScreen`（F1 调试菜单）、以及五个最常见的"命令没有效果"陷阱。

读完本节，你能**在不重启游戏的情况下**快速测试任何 mod 逻辑，并在自己的 mod 里提供标准的控制台调试接口。

---

### 16.9.1 快速入门：控制台界面与执行模式

#### 第一步：打开控制台

| 平台 | 快捷键 |
|------|--------|
| PC（默认） | **\` （反引号）** 或 Ctrl+L |
| 控制台 | 取决于平台绑定 |

控制台打开后，游戏**自动暂停**（`SetConsoleAutopaused(true)`）；关闭时恢复。

```23:24:scripts/screens/consolescreen.lua
	SetConsoleAutopaused(true)
end)
```

#### 第二步：本地执行 vs 远程执行

这是最重要的概念，也是最常踩的坑：

| 模式 | 颜色提示 | 代码运行在 |
|------|----------|-----------|
| **远程（Remote）** | 蓝色文字 | **服务器** |
| **本地（Local）** | 红色文字 | **本机客户端** |

控制台初始化时的默认模式：

```33:33:scripts/screens/consolescreen.lua
	self:ToggleRemoteExecute(InGamePlay()) -- if we are admin, start in remote mode
```

- **单人/主机模式**：`InGamePlay()` 为 true → 默认远程（服务器就是本机，无区别）
- **客户端模式**：同上，如果是管理员则默认远程

在控制台里**按 Ctrl** 切换：

```149:151:scripts/screens/consolescreen.lua
	elseif (key == KEY_LCTRL or key == KEY_RCTRL) and not self.ctrl_pasting then
       self:ToggleRemoteExecute()
	end
```

#### 第三步：命令执行流程

按回车后走 `ConsoleScreen:Run`：

```160:175:scripts/screens/consolescreen.lua
function ConsoleScreen:Run()
	local fnstr = self.console_edit:GetString()

    SuUsedAdd("console_used")

	if fnstr ~= "" then
		ConsoleScreenSettings:AddLastExecutedCommand(fnstr, self.toggle_remote_execute)
	end

	if self.toggle_remote_execute and TheNet:GetIsClient() and (TheNet:GetIsServerAdmin() or IsConsole()) then
        local x, y, z = TheSim:ProjectScreenPos(TheSim:GetPosition())
		TheNet:SendRemoteExecute(fnstr, x, z)
	else
		ExecuteConsoleCommand(fnstr)
	end
end
```

- 远程模式 → 用 `TheNet:SendRemoteExecute(fnstr, x, z)` 把字符串 + 光标世界坐标 发送给服务器
- 本地模式 → 直接 `ExecuteConsoleCommand(fnstr)` 在本机执行

服务器收到远程命令后，同样调 `ExecuteConsoleCommand`，但会临时把 `ThePlayer` 换成发送者：

```2129:2146:scripts/mainfunctions.lua
function ExecuteConsoleCommand(fnstr, guid, x, z)
    local saved_ThePlayer
    if guid ~= nil then
        saved_ThePlayer = ThePlayer
        ThePlayer = guid ~= nil and Ents[guid] or nil
    end
    TheInput.overridepos = x ~= nil and z ~= nil and Vector3(x, 0, z) or nil

    local status, r = pcall(loadstring(fnstr))
    if not status then
        nolineprint(r)
    end

    if guid ~= nil then
        ThePlayer = saved_ThePlayer
    end
    TheInput.overridepos = nil
end
```

关键：`TheInput.overridepos` 被设为光标的世界坐标——所以 `ConsoleWorldPosition()` 在服务端执行时返回的是**你（客户端）的光标位置**，而不是服务器光标。

#### 第四步：历史记录

按 **↑ / ↓** 翻历史；每条命令执行时调 `AddLastExecutedCommand(fnstr, remote)` 记录。

> **新手记忆**：控制台有**本地（红）/远程（蓝）**两种模式，**Ctrl** 切换。单机测试用远程即可；多人服务器上确认自己是管理员才能发远程命令。

---

### 16.9.2 快速入门：最常用的 c_ 命令速查

`c_` 前缀是 Klei 对所有控制台快捷函数的命名约定——定义在 `scripts/consolecommands.lua`，文件开头明确写道：

```38:40:scripts/consolecommands.lua
---------------------------------------------------------------------------------------
-- Console Functions -- These are simple helpers made to be typed at the console.
---------------------------------------------------------------------------------------
```

#### 生成 / 给予物品

```lua
c_spawn("prefab", count)      -- 在光标处生成 count 个 prefab
c_give("prefab", count)       -- 把 prefab 给当前玩家背包（放不下则掉地上）
c_equip("prefab")             -- 给玩家并装备第一件
c_giveingredients("prefab")   -- 给玩家合成 prefab 所需的全部材料
```

`c_give` 内部实现：

```486:507:scripts/consolecommands.lua
function c_give(prefab, count, dontselect)
    local MainCharacter = ConsoleCommandPlayer()

    prefab = string.lower(prefab)

    if MainCharacter ~= nil then
        local first_inst = nil
        for i = 1, count or 1 do
            local inst = DebugSpawn(prefab)
            if inst ~= nil then
                if first_inst == nil then first_inst = inst end
                print("giving ", inst)
                MainCharacter.components.inventory:GiveItem(inst)
                -- ...
            end
        end
        return first_inst
    end
end
```

`MainCharacter` = `ConsoleCommandPlayer()`——优先用**当前选中的实体（c_sel）中的玩家**，没有则用 `ThePlayer`（服务端执行时临时换成发送命令的玩家）。

#### 玩家状态

```lua
c_sethealth(1)           -- 设血量为满（n 是 0~1 的比例）
c_setsanity(0.5)         -- 设精神值为 50%
c_sethunger(0)           -- 设饥饿值为 0%
c_settemperature(25)     -- 设体温为 25°C
c_setmoisture(0)         -- 设湿度为 0
c_godmode()              -- 切换无敌模式
c_freecrafting()         -- 获得所有配方（自由合成）
```

`c_godmode` 的核心逻辑：

```793:817:scripts/consolecommands.lua
function c_godmode(player)
    if TheWorld ~= nil and not TheWorld.ismastersim then
        c_remote("c_godmode()")
        return
    end

    player = ListingOrConsolePlayer(player)
    if player ~= nil then
        -- ...
        elseif player.components.health ~= nil then
            local godmode = player.components.health.invincible
            player.components.health:SetInvincible(not godmode)
            print("God mode: "..tostring(not godmode))
        end
    end
end
```

注意 `c_godmode` 如果不在主机端（`not TheWorld.ismastersim`），会**自动把命令转发给服务器**（`c_remote("c_godmode()")`）——这是"安全写法"的标准模式，很多 c_ 命令都这样写。

#### 查找 / 统计

```lua
c_list("prefab")           -- 列出世界里所有该 prefab 的位置
c_listtag("tag")           -- 列出所有带该 tag 的实体
c_countprefabs("prefab")   -- 统计数量
c_find("prefab", radius)   -- 找最近的该 prefab 并跳过去
c_gonext("prefab")         -- 依次跳到每个同名 prefab（反复调用循环）
```

#### 移动 / 位置

```lua
c_teleport()               -- 传送玩家到光标处
c_teleport(x, y, z)        -- 传送玩家到指定坐标
c_goto(otherplayer)        -- 传送到某玩家旁边
c_move(inst)               -- 把选中的实体移到光标处
```

#### 服务器控制（需要主机权限）

```lua
c_save()                   -- 手动保存
c_reset()                  -- 重新加载上一个存档
c_regenerateworld()        -- 重新生成整个世界
c_announce("msg")          -- 全服公告
c_skip(num)                -- 跳过 num 天
```

> **新手记忆**：最常用的 5 个命令——`c_spawn`（生成）、`c_give`（给物）、`c_godmode`（无敌）、`c_freecrafting`（自由合成）、`c_teleport`（传送）。全部需要**远程模式**才能在联机服务器上生效。

---

### 16.9.3 快速入门：实体选中（c_sel）与实体操作

`c_sel` 和 `c_select` 是控制台调试里的"选择工具"：

```297:310:scripts/consolecommands.lua
-- Get the currently selected entity, so it can be modified etc.
-- Has a gimpy short name so it's easier to type from the console
function c_sel()
    return GetDebugEntity()
end

function c_select(inst)
    if not inst then
        inst = ConsoleWorldEntityUnderMouse()
    end
    print("Selected "..tostring(inst or "<nil>") )
    SetDebugEntity(inst)
    return inst
end
```

- `c_select()` —— 不传参时选中**鼠标下的实体**（`ConsoleWorldEntityUnderMouse`）
- `c_sel()` —— 返回当前选中的实体，结果可赋给变量继续操作

典型工作流：

```lua
-- 选中鼠标下的猪人
c_select()

-- 查看它的血量组件
c_sel().components.health.currenthealth

-- 直接设置它的血量为 1
c_sel().components.health:SetPercent(0.5)

-- 把它删掉
c_sel():Remove()

-- 查看所有 tags
for k, v in pairs(c_sel().pendingtags) do print(k, v) end
```

`c_sel()` 在很多命令里被当作默认目标——比如 `c_move()`（不传参则移动选中实体）、`c_doscenario`（对选中实体应用场景脚本）。

**调试输出**：两个常用工具函数（来自 `debughelpers.lua`）：

```5:42:scripts/debughelpers.lua
function DumpComponent( comp )
    for name,value in pairs(comp) do
        if type(value) == "function" then
            local info = debug.getinfo(value,"LnS")
            print(string.format("      %s = function - %s", name, info.source..":"..tostring(info.linedefined)))
        else
            -- ...
            print(string.format("      %s = %s", name, tostring(value)))
        end
    end
end

function DumpEntity(ent)
    print("============================================ Dumping entity ",ent)
    print(ent.entity:GetDebugString())
    -- ...
    for i,v in pairs(ent.components) do
        print("   Dumping component",i)
        DumpComponent(v)
    end
end
```

- `DumpEntity(c_sel())` —— 打印选中实体的全部字段 + 所有组件的所有成员
- `DumpComponent(c_sel().components.health)` —— 只打印指定组件

> **新手记忆**：`c_select()` 选中鼠标下的物体，`c_sel()` 返回它的引用。选中后可以直接 `.components.xxx` 查看或修改任何数据。`DumpEntity(c_sel())` 是"啥都不懂时看一眼"的万能命令。

---

### 16.9.4 进阶：c_remote 与本地/远程的双重执行路径

#### 第一步：`c_remote` 函数

```200:204:scripts/consolecommands.lua
-- Remotely execute a lua string
function c_remote( fnstr )
    local x, y, z = TheSim:ProjectScreenPos(TheSim:GetPosition())
    TheNet:SendRemoteExecute(fnstr, x, z)
end
```

**用途**：在本地执行代码时**手动发远程命令**。很多 c_ 命令内部用它实现"从客户端也能正常工作"：

```123:134:scripts/consolecommands.lua
function c_reset()
    if TheWorld ~= nil and not TheWorld.ismastersim then
        c_remote("c_reset()")
        return
    end

    if not InGamePlay() then
        StartNextInstance()
    elseif TheWorld ~= nil and TheWorld.ismastersim then
        TheNet:SendWorldRollbackRequestToServer(0)
    end
end
```

这个模式意思是：如果当前代码运行在**客户端**（不是主机），就把命令字符串发给服务器，让服务器执行；否则本地执行。

#### 第二步：mod 写调试命令的安全模板

当你为 mod 写调试命令时，如果命令需要修改服务器数据（实体状态、世界状态），必须在客户端时转发给服务器：

```lua
-- 安全写法模板
function c_my_debug_cmd(arg)
    if TheWorld ~= nil and not TheWorld.ismastersim then
        c_remote(string.format('c_my_debug_cmd(%q)', tostring(arg)))
        return
    end
    -- 真正的逻辑（只在服务端执行）
    local inst = c_sel() or ThePlayer
    -- ...
end
```

#### 第三步：`ConsoleCommandPlayer()` 的优先级

```2:4:scripts/consolecommands.lua
function ConsoleCommandPlayer()
    return (c_sel() ~= nil and c_sel():HasTag("player") and c_sel()) or ThePlayer or AllPlayers[1]
end
```

优先级：**选中的玩家实体 > `ThePlayer`（命令发送者）> `AllPlayers[1]`（第一个玩家）**。

**在联机 dedicated server 上**：`ThePlayer` 会被替换成命令发送者（见 `ExecuteConsoleCommand` 的 `guid` 参数），因此 `ConsoleCommandPlayer()` 正确地返回发送命令的那个玩家。如果你想操作**其他玩家**，先 `c_select(target)` 再调命令。

#### 第四步：`ConsoleWorldPosition()` 与 `overridepos`

```6:8:scripts/consolecommands.lua
function ConsoleWorldPosition()
    return TheInput.overridepos or TheInput:GetWorldPosition()
end
```

远程命令发送时，服务器会设置 `TheInput.overridepos = Vector3(x, 0, z)`（客户端光标坐标）——所以即使在服务端执行，`ConsoleWorldPosition()` 也能拿到正确的**客户端光标位置**。`c_spawn` 靠这个在光标处生成实体。

> **进阶记忆**：mod 调试命令的标准写法是"检查 `ismastersim`，不是则 `c_remote` 转发"；`ConsoleCommandPlayer()` 自动解析正确的目标玩家；`ConsoleWorldPosition()` 在服务端也能拿到客户端光标位置。

---

### 16.9.5 进阶：customcommands.lua——持久化个人快捷命令

#### 第一步：文件位置与加载机制

游戏启动时，从**客户端配置目录的上一级**加载 `customcommands.lua`：

```1425:1432:scripts/mainfunctions.lua
    TheSim:GetPersistentString("../customcommands.lua",
        function(load_success, str)
            if load_success then
                local fn = loadstring(str)
                known_assert(fn ~= nil, "CUSTOM_COMMANDS_ERROR")
                xpcall(fn, debug.traceback)
            end
        end)
```

路径 `"../customcommands.lua"` 是相对于 Klei 存档目录（如 `Documents/Klei/DoNotStarveTogether/`）的上一级——实际路径因系统而异，通常是：

- Windows: `%USERPROFILE%\Documents\Klei\customcommands.lua`
- macOS: `~/Documents/Klei/customcommands.lua`

**注意**：
- 这个文件**随游戏启动加载**，修改后需要**重启游戏**才生效
- 加载失败（语法错误）会触发 `known_assert`，但不会崩溃游戏
- 控制台文档里有提示：`ConsoleScreenSettings:AddLastExecutedCommand('c_give("batbat"', true)` 可以预置历史记录

#### 第二步：示例内容

```lua
-- customcommands.lua 示例
-- 快速测试：给玩家全套装备并无敌
function c_fullsetup()
    c_godmode()
    c_freecrafting()
    c_sethealth(1)
    c_setsanity(1)
    c_sethunger(1)
end

-- 快速传送到指定坐标
function c_home()
    c_teleport(0, 0, 0)
end

-- 打印选中实体的 prefab 名
function c_name()
    local sel = c_sel()
    if sel then
        print(sel.prefab)
    end
end
```

定义在这里的函数成为**全局函数**，在控制台里直接输入函数名即可调用。

#### 第三步：也可以用脚本预置控制台历史

```lua
-- 在 customcommands.lua 里：
ConsoleScreenSettings:AddLastExecutedCommand('c_give("meatballs", 10)', true)
ConsoleScreenSettings:AddLastExecutedCommand('c_godmode()', true)
```

这样游戏启动后，控制台历史里就预设了这两条命令，按 ↑ 就能快速调用。

> **进阶记忆**：`customcommands.lua` 是开发者的"个人工具库"——在这里写好常用命令，重启后在任何世界都能用；它不随 mod 分发，是纯本地的快捷键。

---

### 16.9.6 进阶：mod 如何向控制台暴露调试命令

#### 第一步：最简单的方式——在 modmain.lua 定义全局函数

控制台执行的是 Lua 全局环境——**任何在 modmain.lua 里定义的全局函数**，都可以在控制台里调用：

```lua
-- modmain.lua
function my_debug_spawn_all()
    local items = {"my_sword", "my_shield", "my_helmet"}
    for _, v in ipairs(items) do
        c_give(v)
    end
end

function my_debug_reset_quest()
    local player = ThePlayer
    if player and player.components.myquest then
        player.components.myquest:Reset()
        print("Quest reset!")
    end
end
```

在控制台输入 `my_debug_spawn_all()` 即可调用。

**命名建议**：用 mod 前缀避免与官方命令冲突，比如 `dm_`（勋章 mod 的做法）、`myth_` 等。

#### 第二步：勋章 mod 的调试命令范式

`mods/联机版mod/勋章/scripts/medal_debugcommands.lua` 展示了一个良好的 mod 调试命令组织方式：

```27:48:mods/联机版mod/勋章/scripts/medal_debugcommands.lua
--生成所有勋章(isfinal为true则只生成最终形态勋章)
function dm_allmedal(isfinal)
	local items = {"large_multivariate_certificate"}
	if not isfinal then
		table.insert(items, "medium_multivariate_certificate")
		table.insert(items, "multivariate_certificate")
	end
	for k, v in pairs(require("medal_defs/functional_medal_defs").MEDAL_DEFS) do
		if (not isfinal or v.isfinal) and not v.nodebug then
			table.insert(items, v.name)
		end
	end
	_spawn_list(items, 1, ...)
end
```

**要点**：
1. 命令名带 mod 前缀（`dm_`）
2. 用单独的文件组织调试命令（在 modmain.lua 里 `require`）
3. 复用 `ConsoleWorldPosition()` 做生成位置

#### 第三步：在 modmain.lua 里引入调试命令文件

```lua
-- modmain.lua
if BRANCH == "dev" or true then  -- 生产版也可用（控制台本来就是给开发者的）
    require("medal_debugcommands")  -- 或者你自己的调试命令文件
end
```

> **进阶记忆**：mod 暴露调试命令的最简方式是在 modmain.lua 里定义带前缀的全局函数；独立调试命令文件更整洁；始终复用 `ConsoleCommandPlayer()` 和 `ConsoleWorldPosition()` 来定位目标。

---

### 16.9.7 老手：d_ 命令与 debugkeys.lua

#### `d_` 前缀命令

`d_` 前缀命令定义在 `scripts/debugcommands.lua` 和 `scripts/debugkeys.lua`，比 `c_` 更复杂、更底层：

| 函数 | 作用 |
|------|------|
| `d_spawnlist(list, spacing, fn)` | 把一个 prefab 列表展开生成在光标附近（用于批量测试）|
| `d_playeritems()` | 生成所有有 builder_tag 的玩家专属物品 |
| `d_createscrapbookdata()` | 生成/更新 `scrapbookdata.lua`（自动化工具）|
| `d_allcircuits()` | 生成所有 WX-78 模组电路 |

`d_spawnlist` 是 mod 调试的利器——生成一批测试物品：

```3:43:scripts/debugcommands.lua
function d_spawnlist(list, spacing, fn)
    local created = {}
	spacing = spacing or 2
	local num_wide = math.ceil(math.sqrt(#list))

	local pt = ConsoleWorldPosition()
	pt.x = pt.x - num_wide * 0.5 * spacing
	pt.z = pt.z - num_wide * 0.5 * spacing

	for y = 0, num_wide-1 do
		for x = 0, num_wide-1 do
			if list[(y*num_wide + x + 1)] then
				-- ...
				local inst = SpawnPrefab(prefab)
				inst.Transform:SetPosition((pt + Vector3(x*spacing, 0, y*spacing)):Get())
			end
		end
	end
    return created
end
```

用法：

```lua
-- 把 mod 里所有武器排列生成在光标附近
d_spawnlist({"my_sword", "my_spear", "my_bow"}, 2)
```

#### `debugkeys.lua` 的快捷键系统

`debugkeys.lua` 注册了一系列**快捷键调试动作**，覆盖了 F1-F12 及其他组合键。它在 `scripts/debugkeys.lua` 里 require 了 `consolecommands` 并使用 `TheInput:AddKeyHandler` 绑定动作。

注意：这个文件只在**本机开启 debug 模式（`BRANCH == "dev"`）**时才完全可用；普通玩家/开发者在发行版里无法使用这些快捷键。

#### `debughelpers.lua` 的实用函数

```lua
DumpEntity(c_sel())          -- 完整转储实体信息
DumpComponent(comp)           -- 转储单个组件
DumpUpvalues(func)            -- 转储函数的 upvalue
```

这些函数生成的输出会打印到游戏日志（也会出现在控制台窗口）。

> **老手记忆**：`d_spawnlist` 是批量测试的神器；`DumpEntity(c_sel())` 是"看不懂这个实体到底有什么"时的首选；`debugkeys.lua` 的快捷键只在 dev build 下完整可用。

---

### 16.9.8 老手：DebugMenuScreen（F1 调试菜单）

游戏在 **BRANCH == "dev"**（开发版）时，按 **F1** 会弹出 `DebugMenuScreen`：

```14:38:scripts/screens/DebugMenuScreen.lua
local DebugMenuScreen = Class(Screen, function(self)
	Screen._ctor(self, "DebugMenuScreen")

   	self.blackoverlay = self:AddChild(Image("images/global.xml", "square.tex"))
    -- ...
	self.blackoverlay:SetTint(0,0,0,.75)

	self.text = self:AddChild(Text(BODYTEXTFONT, ... 16 ..., "blah"))
    -- ...
	TheFrontEnd:HideConsoleLog()
end)
```

```62:80:scripts/screens/DebugMenuScreen.lua
function DebugMenuScreen:OnBecomeActive()
	DebugMenuScreen._base.OnBecomeActive(self)
	SetPause(true,"console")

	self.menu = menus.TextMenu(InGamePlay() and "IN GAME DEBUG MENU" or "FRONT END DEBUG MENU")
	local main_options = {}

	-- ...
	local craft_menus = {}
	for k,v in pairs(AllRecipes) do
        if IsRecipeValid(v.name) and v.tab then
    		craft_menus[v.tab] = craft_menus[v.tab] or {}
    		table.insert(craft_menus[v.tab], menus.DoAction(v.name, function() for kk,vv in pairs(v.ingredients) do ConsoleRemote('c_give("%s", %d)',{vv.type, vv.amount}) end end))
        end
	end
```

`debugmenu.lua` 提供了菜单选项类型：

```3:55:scripts/debugmenu.lua
local MenuOption = Class(function(self, str)
	self.str = str
end)
-- ...
local DoAction = Class(MenuOption, function(self, str, fn)
-- ...
local Submenu = Class(MenuOption, function(self, str, options, name)
```

**对 mod 开发者的意义**：
- 在 dev build 里可以通过 F1 菜单快速给自己物品、切换季节、调整天气——比打控制台命令方便
- **发行版**（普通玩家的游戏）里 F1 菜单不可用

---

### 16.9.9 老手：五个常见的"命令没有效果"陷阱

#### 陷阱 1：忘记切换远程模式——命令在本地执行

**症状**：`c_spawn("my_creature")` 在本地执行，实体出现了但立刻消失，或者服务器上没有变化。

**原因**：控制台处于**本地模式**，命令运行在客户端——客户端生成的实体不会同步到服务器，游戏下一帧就被清除。

**修复**：在控制台里按 **Ctrl** 切到远程（蓝色），再执行命令。

#### 陷阱 2：`c_spawn` / `c_give` 找不到 mod prefab——prefab 未加载

**症状**：`c_spawn("my_creature")` 执行后 print 显示 prefab 为 nil。

**原因**：`c_spawn` 内部调用 `DebugSpawn(prefab)`，如果这个 prefab 没有被注册（`Prefabs["my_creature"]` 为 nil），会返回 nil。

**修复**：确认 mod 已经正确用 `Prefab("my_creature", fn, assets)` 注册；在控制台用 `print(Prefabs["my_creature"])` 验证是否存在。

#### 陷阱 3：命令自动转发到服务器但参数丢失

**症状**：`c_my_cmd("hello", 42)` 在客户端调用，服务器执行时参数变成了默认值或 nil。

**原因**：mod 写了 `c_remote("c_my_cmd()")` 但没有带上参数——字符串格式化遗漏了。

**修复**：正确地把参数序列化进命令字符串：
```lua
function c_my_cmd(name, count)
    if TheWorld ~= nil and not TheWorld.ismastersim then
        c_remote(string.format('c_my_cmd(%q, %d)', tostring(name), count))
        return
    end
    -- 服务端逻辑...
end
```

#### 陷阱 4：`ConsoleCommandPlayer()` 返回 nil——没有玩家

**症状**：`c_sethealth(1)` 报错 "attempt to index nil (ConsoleCommandPlayer returned nil)"。

**原因**：在**主界面**（非游戏中）打开控制台执行，此时 `ThePlayer` 和 `AllPlayers[1]` 都是 nil，`ConsoleCommandPlayer()` 返回 nil。

**修复**：只在游戏中使用这类命令；如果 mod 需要在主界面做调试，不要依赖 `ConsoleCommandPlayer()`。

#### 陷阱 5：`customcommands.lua` 修改后没生效——忘记重启

**症状**：在 `customcommands.lua` 里加了新函数，控制台里调用报"attempt to call a nil value"。

**原因**：`customcommands.lua` 在**游戏启动时**加载一次，修改后需要**完全重启游戏**（不是重新进存档）才能生效。

**修复**：重启游戏；或者在当前游戏会话里用控制台临时定义函数（只对当前会话有效）：
```lua
-- 控制台里直接定义（临时）
my_temp_fn = function() c_give("meatballs", 10) end
my_temp_fn()
```

---

### 16.9 小结

```
控制台架构：
  打开：` (反引号) / Ctrl+L
    ↓
  ConsoleScreen
    ├── toggle_remote_execute = true  → TheNet:SendRemoteExecute(fnstr, x, z)
    │                                     ↓
    │                           服务器 ExecuteConsoleCommand(fnstr, guid, x, z)
    │                                     (ThePlayer 临时换成发送者)
    └── toggle_remote_execute = false → ExecuteConsoleCommand(fnstr)（本机执行）

c_ 命令（consolecommands.lua）：
  生成：c_spawn / c_give / c_equip / c_giveingredients
  状态：c_godmode / c_sethealth / c_setsanity / c_sethunger / c_settemperature
  选中：c_sel / c_select (ConsoleWorldEntityUnderMouse)
  移动：c_teleport / c_goto / c_gonext / c_find
  统计：c_list / c_listtag / c_countprefabs
  服务：c_save / c_reset / c_announce

d_ 命令（debugcommands.lua）：
  批量：d_spawnlist(list, spacing)
  特殊：d_playeritems / d_createscrapbookdata / d_allcircuits

mod 扩展控制台：
  1. modmain.lua 定义全局函数（带前缀）
  2. 单独的 *_debugcommands.lua 文件（require 进 modmain）
  3. customcommands.lua 写个人快捷命令（需重启才生效）

安全命令模板：
  function c_my_cmd(arg)
    if TheWorld ~= nil and not TheWorld.ismastersim then
      c_remote(string.format('c_my_cmd(%q)', tostring(arg)))
      return
    end
    -- 服务端逻辑
  end
```

**新手核心三句**：控制台有**本地（红）/远程（蓝）**两种模式，按 Ctrl 切换，联机测试必须用远程；`c_give` / `c_spawn` 是最常用的两个命令；`c_select()` + `c_sel().components.xxx` 能实时读写任何实体数据。

**进阶核心三句**：`ConsoleCommandPlayer()` 优先返回选中玩家、其次 ThePlayer、最后 AllPlayers[1]；服务端执行时 `ThePlayer` 临时换成命令发送者；mod 调试命令需要"检查 `ismastersim`，不是则 `c_remote` 转发"的安全模板。

**老手核心三句**：`d_spawnlist` 是批量测试物品的利器；`DumpEntity(c_sel())` 是实体调试的万能工具；`customcommands.lua` 修改后必须重启游戏，临时测试直接在控制台里定义匿名函数。

下一节（16.10）是本章的**实战收尾**——从零开始为 mod 添加一个完整的设置面板，把 16.1-16.9 的所有知识综合运用。

## 16.10 实战：为 Mod 添加设置面板

### 本节导读

16.1-16.9 我们把 UI 系统的**所有零件**都讲完了——Widget 树、锚点布局、Screen 栈、HUD 注入、容器控件、滚动列表、焦点系统、调试工具。本节是**装配工厂**：用这些零件，从零开始做一个**完整的 mod 设置面板**。

目标产品：一个带**三个选项的设置面板**——

1. **选项 1（Spinner）**：控制 mod 某个功能的开启/关闭/自动
2. **选项 2（Spinner）**：一个数值选择（低/中/高）
3. **关闭按钮**

面板功能需求：
- 玩家点击 HUD 上的按钮打开面板
- 设置修改后**自动持久化**到本地，重开游戏不丢失
- **随 HUD 缩放比例**自动缩放
- 按 Esc / M 键关闭

整个实现分 5 个文件：

| 文件 | 职责 |
|------|------|
| `scripts/screens/mymod_settingsscreen.lua` | 设置面板 Screen |
| `scripts/mymod_settings.lua` | 存读设置数据 |
| `scripts/widgets/mymod_hud_button.lua` | HUD 入口按钮 |
| `scripts/mymod_hud_hook.lua` | 注入 HUD 的胶水代码 |
| `modmain.lua` | 入口，引入上面的文件 |

> **新手**先看 16.10.1-16.10.3——理解需求结构、看懂最小化 Screen 框架、用 `TEMPLATES.CurlyWindow` + `TEMPLATES.StandardButton` 让一个面板显示出来；**进阶读者**继续看 16.10.4-16.10.7，加入 `TEMPLATES.LabelSpinner` 下拉选项、用 `TheSim:SetPersistentString` 持久化、监听 `refreshhudsize` 事件做 HUD 缩放适配、用 `AddClassPostConstruct` 把面板挂入 HUD；**老手**跳到 16.10.8-16.10.10，了解需要服务器同步时如何用 RPC 传播设置、焦点流与控制器支持、以及五个最常见的"设置面板装上去但不工作"的坑。

读完本节，你能在 **2 小时内**为任何 mod 加上一个规范的、可持久化的、HUD 缩放自适应的设置面板。

---

### 16.10.1 快速入门：需求分析与五个文件的职责

#### 第一步：为什么需要 5 个文件？

一个偷懒的做法是把所有代码都堆在 `modmain.lua` 里。但这样：
- 模块间耦合高，改一处可能影响全局
- 代码难以阅读和维护
- Screen 类通常要 `require`，不能直接在 `modmain.lua` 里写 Class

标准的分层做法：

```
modmain.lua
  └─ require "mymod_settings"          ← 启动时加载 & 恢复设置
  └─ require "mymod_hud_hook"          ← 注入 HUD 按钮 & 打开面板的方法
       └─ require "widgets/mymod_hud_button"    ← HUD 按钮 Widget
       └─ require "screens/mymod_settingsscreen" ← 面板 Screen
            └─ require "mymod_settings"           ← 读写设置
```

#### 第二步：理清数据流

```
玩家点击 HUD 按钮
    ↓
playerhud.ShowMySettingsScreen()（通过 AddClassPostConstruct 注入）
    ↓
创建 MySettingsScreen(owner)
    ↓
面板里每个 Spinner 的 onchanged_fn
    ↓
修改 MYMOD_SETTINGS[key] = newvalue
    ↓
SaveMySettings()  → TheSim:SetPersistentString(...)
```

#### 第三步：定义设置数据结构

在 `scripts/mymod_settings.lua` 里定义设置的**默认值**和**可选项**：

```lua
-- scripts/mymod_settings.lua

MYMOD_SETTINGS = {
    MY_FEATURE = "auto",   -- "off" / "auto" / "on"
    MY_LEVEL   = "medium", -- "low" / "medium" / "high"
}

local SETTING_KEYS = {"MY_FEATURE", "MY_LEVEL"}
```

把设置放到一个全局表里，方便在游戏任何地方访问。

> **新手记忆**：5 个文件各司其职——Screen 只管 UI 显示、settings.lua 只管数据读写、hud_button 只管 HUD 按钮、hud_hook 只管注入 HUD、modmain 只管引导。**不要把 Screen 类代码直接写在 modmain.lua 里**。

---

### 16.10.2 快速入门：最小化 Screen 框架——让面板显示出来

参考 `mods/联机版mod/勋章/scripts/screens/medalsettingsscreen.lua`，一个最小化的设置 Screen 骨架如下：

```lua
-- scripts/screens/mymod_settingsscreen.lua
local Screen   = require "widgets/screen"
local Widget   = require "widgets/widget"
local Text     = require "widgets/text"
local Image    = require "widgets/image"
local TEMPLATES = require "widgets/redux/templates"

local MySettingsScreen = Class(Screen, function(self, owner)
    Screen._ctor(self, "MySettingsScreen")
    self.owner = owner

    -- 1. HUD 缩放根节点（所有内容挂在这里）
    self.scalingroot = self:AddChild(Widget("scaling_root"))
    self.scalingroot:SetVAnchor(ANCHOR_MIDDLE)
    self.scalingroot:SetHAnchor(ANCHOR_MIDDLE)
    self.scalingroot:SetScaleMode(SCALEMODE_PROPORTIONAL)
    self.scalingroot:SetScale(TheFrontEnd:GetHUDScale())

    -- 2. 全屏透明遮罩：点击遮罩关闭面板
    self.black = self.scalingroot:AddChild(Image("images/global.xml", "square.tex"))
    self.black:SetVRegPoint(ANCHOR_MIDDLE)
    self.black:SetHRegPoint(ANCHOR_MIDDLE)
    self.black:SetVAnchor(ANCHOR_MIDDLE)
    self.black:SetHAnchor(ANCHOR_MIDDLE)
    self.black:SetScaleMode(SCALEMODE_FILLSCREEN)
    self.black:SetTint(0, 0, 0, 0)
    self.black.OnMouseButton = function() self:OnCancel() end

    -- 3. 主面板容器
    local PANEL_W, PANEL_H = 240, 300
    self.panel = self.scalingroot:AddChild(TEMPLATES.CurlyWindow(PANEL_W, PANEL_H))
    self.panel:SetPosition(0, 0)

    -- 4. 标题
    self.title = self.panel:AddChild(Text(HEADERFONT, 30, "Mod 设置"))
    self.title:SetPosition(0, PANEL_H / 2 - 40)
    self.title:SetColour(1, 0.8, 0.2, 1)

    SetAutopaused(true)
end)

function MySettingsScreen:OnCancel()
    TheFrontEnd:PopScreen(self)
    SetAutopaused(false)
end

function MySettingsScreen:OnControl(control, down)
    if MySettingsScreen._base.OnControl(self, control, down) then return true end
    if not down and (control == CONTROL_CANCEL or control == CONTROL_MAP) then
        self:OnCancel()
        return true
    end
end

return MySettingsScreen
```

**关键设计**：
1. `scalingroot` 负责跟随 HUD 缩放比例——见下面 16.10.6 的详细说明
2. 全屏透明 `black`：点击面板外关闭，是"点击遮罩关闭"的标准实现
3. `TEMPLATES.CurlyWindow` 是官方风格的圆角卷轴窗口——尺寸约束在 190-1000 × 90-500 之间

```1725:1739:scripts/widgets/redux/templates.lua
function TEMPLATES.CurlyWindow(sizeX, sizeY, title_text, bottom_buttons, button_spacing, body_text)
    local w = NineSlice("images/dialogcurly_9slice.xml")
    local top = w:AddCrown("crown-top-fg.tex", ANCHOR_MIDDLE, ANCHOR_TOP, 0, 68)
    local top_bg = w:AddCrown("crown-top.tex", ANCHOR_MIDDLE, ANCHOR_TOP, 0, 44)
    top_bg:MoveToBack()

    local bottom = w:AddCrown("crown-bottom-fg.tex", ANCHOR_MIDDLE, ANCHOR_BOTTOM, 0, -14)
    bottom:MoveToFront()

    -- Ensure we're within the bounds of looking good and fitting on screen.
    sizeX = math.clamp(sizeX or 200, 190, 1000)
    sizeY = math.clamp(sizeY or 200, 90, 500)
    w:SetSize(sizeX, sizeY)
    w:SetScale(0.7, 0.7)
```

注意 `CurlyWindow` 内部 `SetScale(0.7, 0.7)`——它的尺寸参数是"逻辑尺寸"，实际渲染时缩小到 70%，所以 `PANEL_H = 300` 实际显示约 210 像素高。

> **新手记忆**：Screen 最小结构 = scalingroot（跟随 HUD 缩放）+ black（全屏点击关闭遮罩）+ panel（TEMPLATES.CurlyWindow 主容器）+ 标题文字。

---

### 16.10.3 快速入门：用 StandardButton 添加关闭按钮

在面板底部加一个关闭按钮：

```lua
-- 继续在 MySettingsScreen 的构造函数里
self.close_btn = self.panel:AddChild(
    TEMPLATES.StandardButton(
        function() self:OnCancel() end,  -- onclick
        "关闭",                          -- 按钮文字
        {160, 40}                        -- {width, height}
    )
)
self.close_btn:SetPosition(0, -PANEL_H / 2 + 40)
```

`TEMPLATES.StandardButton` 的定义：

```554:599:scripts/widgets/redux/templates.lua
function TEMPLATES.StandardButton(onclick, txt, size, icon_data)
    local prefix = "button_carny_long"
    if size and #size == 2 then
        local ratio = size[1] / size[2]
        if ratio > 4 then
            prefix = "button_carny_xlong"
        elseif ratio < 1.1 then
            prefix = "button_carny_square"
        end
    end
    local btn = ImageButton("images/global_redux.xml",
        prefix.."_normal.tex",
        prefix.."_hover.tex",
        prefix.."_disabled.tex",
        prefix.."_down.tex")
    btn:SetOnClick(onclick)
    btn:SetText(txt)
    btn:SetFont(CHATFONT)
    -- ...
    return btn
```

**宽高比决定按钮外形**：
- ratio > 4 → `button_carny_xlong`（超宽按钮）
- ratio ≈ 1 → `button_carny_square`（方形按钮）
- 其他 → `button_carny_long`（默认长条按钮）

> **新手记忆**：`TEMPLATES.StandardButton(onclick_fn, text, {width, height})` 是官方风格按钮的标准创建方式；宽高比决定外形；用 `SetPosition` 把它放在面板内的合适位置。

---

### 16.10.4 进阶：用 LabelSpinner 添加下拉选择设置项

`TEMPLATES.LabelSpinner` 是"标签 + 下拉选择"的标准控件组合——左边文字说明、右边 Spinner 控件：

```lua
-- 在构造函数里，添加第一个设置项
local YSTART = PANEL_H / 2 - 80  -- 从顶部向下偏移 80 开始

self.feature_spinner = self.panel:AddChild(
    TEMPLATES.LabelSpinner(
        "功能模式",              -- 标签文字
        {                        -- spinnerdata: 选项列表
            {text = "关闭", data = "off"},
            {text = "自动", data = "auto"},
            {text = "开启", data = "on"},
        },
        120,                     -- width_label
        110,                     -- width_spinner
        40,                      -- height（可不传，默认 40）
        5,                       -- spacing（可不传，默认 5）
        nil,                     -- font（默认 CHATFONT）
        22,                      -- font_size
        nil,                     -- horiz_offset
        function(new_data)       -- onchanged_fn（选项切换时回调）
            MYMOD_SETTINGS.MY_FEATURE = new_data
            SaveMySettings()
        end
    )
)
self.feature_spinner:SetPosition(0, YSTART)
-- 初始化时显示当前存档的值
self.feature_spinner.spinner:SetSelected(MYMOD_SETTINGS.MY_FEATURE)
```

`TEMPLATES.LabelSpinner` 的函数签名和内部结构：

```1132:1156:scripts/widgets/redux/templates.lua
function TEMPLATES.LabelSpinner(labeltext, spinnerdata, width_label, width_spinner, height, spacing, font, font_size, horiz_offset, onchanged_fn, colour, tooltip_text)
    width_label = width_label or 220
    width_spinner = width_spinner or 150
    height = height or 40
    spacing = spacing or 5
    -- ...
    local wdg = Widget("labelspinner")
    wdg.label = wdg:AddChild( Text(font, font_size, labeltext) )
    -- ...
    wdg.spinner = wdg:AddChild(TEMPLATES.StandardSpinner(spinnerdata, width_spinner, height, font, font_size, onchanged_fn, colour))
    -- ...
    wdg.focus_forward = wdg.spinner   -- 焦点传递给 spinner
    return wdg
end
```

**关键细节**：
- 返回的是一个 `Widget("labelspinner")`，它包含 `wdg.label`（Text）和 `wdg.spinner`（Spinner）
- `wdg.focus_forward = wdg.spinner` 已设置好——焦点移到这个 Widget 时自动聚焦 Spinner
- `spinner:SetSelected(value)` 接受的是 `data` 值，而不是 `text` 值——用来恢复已保存的设置

**添加第二个设置项**（数值选择）：

```lua
self.level_spinner = self.panel:AddChild(
    TEMPLATES.LabelSpinner(
        "品质等级",
        {
            {text = "低", data = "low"},
            {text = "中", data = "medium"},
            {text = "高", data = "high"},
        },
        120, 110, 40, 5, nil, 22, nil,
        function(new_data)
            MYMOD_SETTINGS.MY_LEVEL = new_data
            SaveMySettings()
        end
    )
)
self.level_spinner:SetPosition(0, YSTART - 50)
self.level_spinner.spinner:SetSelected(MYMOD_SETTINGS.MY_LEVEL)
```

> **进阶记忆**：`LabelSpinner` 的 `spinnerdata` 是 `{text, data}` 对的列表；`spinner:SetSelected(data_value)` 用 data 值定位当前选项；`onchanged_fn` 接收的参数是 `data` 值，不是 `text` 值。

---

### 16.10.5 进阶：持久化设置——TheSim:GetPersistentString / SetPersistentString

参照勋章 mod 的 `SaveMedalSettingData` / `LoadMedalSettingData` 实现 mod 设置的存读：

```636:659:mods/联机版mod/勋章/scripts/medal_globalfn.lua
function SaveMedalSettingData()
	local setting_data={}
	for _, v in ipairs(setting_name) do
		setting_data[v]=TUNING[v]
	end
	local str = DataDumper(setting_data, nil, true)
	TheSim:SetPersistentString("medal_setting_data", str, false)
end
--加载勋章设置信息
function LoadMedalSettingData()
	TheSim:GetPersistentString("medal_setting_data", function(load_success, data)
		if load_success and data ~= nil then
            local success, setting_data = RunInSandbox(data)
		    if success and setting_data then
				for _, v in ipairs(setting_name) do
					if setting_data[v]~=nil then
						TUNING[v] = setting_data[v]
					end
				end
			end
		end
	end)
end
LoadMedalSettingData()--游戏开始直接调用一下
```

为 mod 仿照实现：

```lua
-- scripts/mymod_settings.lua（完整版）

MYMOD_SETTINGS = {
    MY_FEATURE = "auto",
    MY_LEVEL   = "medium",
}

local SETTING_KEYS = {"MY_FEATURE", "MY_LEVEL"}

function SaveMySettings()
    local data = {}
    for _, k in ipairs(SETTING_KEYS) do
        data[k] = MYMOD_SETTINGS[k]
    end
    local str = DataDumper(data, nil, true)
    TheSim:SetPersistentString("mymod_settings", str, false)
end

function LoadMySettings()
    TheSim:GetPersistentString("mymod_settings", function(load_success, data)
        if load_success and data ~= nil then
            local success, saved = RunInSandboxSafe(data)
            if success and type(saved) == "table" then
                for _, k in ipairs(SETTING_KEYS) do
                    if saved[k] ~= nil then
                        MYMOD_SETTINGS[k] = saved[k]
                    end
                end
            end
        end
    end)
end

-- 游戏启动时立刻加载
LoadMySettings()
```

**几个重要细节**：

| 细节 | 说明 |
|------|------|
| `DataDumper(data, nil, true)` | 把 Lua 表序列化成字符串；第三个参数 `true` 开启紧凑格式 |
| `RunInSandboxSafe(data)` | 安全地把字符串反序列化为 Lua 值；遇到语法错误不崩溃 |
| `TheSim:SetPersistentString("key", str, false)` | 第三个参数 `encrypt`，通常传 `false` |
| `TheSim:GetPersistentString("key", callback)` | 回调是**异步**的——代码后面的逻辑不要依赖回调里的结果 |
| 键名（`"mymod_settings"`）| 用 mod 前缀避免与其他 mod / 官方存档键冲突 |

> **进阶记忆**：持久化用 `TheSim:SetPersistentString` + `DataDumper`；恢复用 `GetPersistentString` + `RunInSandboxSafe`；回调是**异步**的，`LoadMySettings()` 要在模块加载时立刻调用；键名加 mod 前缀防止冲突。

---

### 16.10.6 进阶：HUD 缩放适配——监听 refreshhudsize 事件

玩家修改 HUD 缩放设置时，游戏会推送 `refreshhudsize` 事件。设置面板如果不监听，缩放后看起来会过大或过小。

勋章 mod 的标准做法：

```44:52:mods/联机版mod/勋章/scripts/screens/medalsettingsscreen.lua
        self.inst:ListenForEvent(
            "refreshhudsize",
            function(hud, scale)
                if self.isopen then
                    self.scalingroot:SetScale(scale)
                end
            end,
            owner.HUD.inst
        )
```

在 `MySettingsScreen` 里加上类似监听：

```lua
-- 在构造函数里（在 scalingroot 创建之后）
self.isopen = true

self.inst:ListenForEvent(
    "refreshhudsize",
    function(hud, scale)
        if self.isopen then
            self.scalingroot:SetScale(scale)
        end
    end,
    owner.HUD.inst  -- 事件由 HUD 实体推送
)
```

同时在 `OnCancel` 里标记关闭：

```lua
function MySettingsScreen:OnCancel()
    self.isopen = false
    TheFrontEnd:PopScreen(self)
    SetAutopaused(false)
end
```

`isopen` 标志防止面板已关闭后回调还在调用 `SetScale`（HUD 实体可能还在）。

---

### 16.10.7 进阶：挂载到 HUD——AddClassPostConstruct + OpenScreenUnderPause

面板写好了，但怎么打开它？标准方式是**向 PlayerhHUD 注入打开/关闭方法**，然后在 HUD 按钮里调用。

```1210:1258:mods/联机版mod/勋章/scripts/medal_ui.lua
AddClassPostConstruct("screens/playerhud",function(self, anim, owner)
    -- ...
    self.ShowMedalSettingsScreen = function(_, attach)
		self.medalsettingsscreen = MedalSettingsScreen(self.owner)
		self:OpenScreenUnderPause(self.medalsettingsscreen)
		return self.medalsettingsscreen
	end

	self.CloseMedalSettingsScreen = function(_)
		if self.medalsettingsscreen then
			self.medalsettingsscreen:Close()
			self.medalsettingsscreen = nil
		end
	end
```

为 MyMod 仿照实现：

```lua
-- scripts/mymod_hud_hook.lua

local MySettingsScreen = require "screens/mymod_settingsscreen"

AddClassPostConstruct("screens/playerhud", function(self, anim, owner)
    -- 打开设置面板的方法
    self.ShowMySettingsScreen = function(_)
        if self.mysettingsscreen == nil then
            self.mysettingsscreen = MySettingsScreen(self.owner)
            self:OpenScreenUnderPause(self.mysettingsscreen)
        end
    end

    -- 关闭设置面板的方法
    self.CloseMySettingsScreen = function(_)
        if self.mysettingsscreen ~= nil then
            self.mysettingsscreen:OnCancel()
            self.mysettingsscreen = nil
        end
    end
end)
```

**`OpenScreenUnderPause`** 是 PlayerhHUD 的方法，它调用 `TheFrontEnd:PushScreen(screen)`，但会处理好暂停状态（避免重复暂停/取消暂停的冲突）。

在 HUD 按钮的点击回调里调用：

```lua
-- 在 HUD 按钮的 onclick 里：
ThePlayer.HUD:ShowMySettingsScreen()
```

在 modmain.lua 里引入 hook：

```lua
-- modmain.lua
require "mymod_settings"       -- 最先加载，立刻 Load 设置
require "mymod_hud_hook"       -- 注入 HUD 方法
```

> **进阶记忆**：`AddClassPostConstruct("screens/playerhud", fn)` 注入 HUD 后处理；`OpenScreenUnderPause(screen)` 是标准的"在 HUD 里推 Screen"的安全方式；打开前检查 `mysettingsscreen == nil` 防止重复打开。

---

### 16.10.8 老手：设置需要同步到服务器——使用 mod RPC

有些设置（如"影响所有玩家的游戏行为"）需要服务器知道，**纯本地持久化是不够的**。这时需要用 **Mod RPC** 把设置广播给服务器。

#### 第一步：注册 RPC 处理器

在 modmain.lua 里：

```lua
-- modmain.lua

-- 定义 RPC ID（用字符串防止 ID 冲突）
MOD_RPC = MOD_RPC or {}
MOD_RPC.MyMod = MOD_RPC.MyMod or {}
MOD_RPC.MyMod.UpdateSetting = GetModRPC("MyMod", "UpdateSetting")

-- 服务器收到 RPC 时的处理函数
AddModRPCHandler("MyMod", "UpdateSetting", function(player, key, value)
    -- 注意：这里运行在服务器上
    -- 可以在这里修改服务器端的全局配置
    if key == "MY_FEATURE" then
        TheWorld.mymod_feature = value
    end
end)
```

#### 第二步：在设置变化时发送 RPC

在 `onchanged_fn` 里补充 RPC 发送：

```lua
onchanged_fn = function(new_data)
    MYMOD_SETTINGS.MY_FEATURE = new_data
    SaveMySettings()

    -- 把设置发给服务器（如果需要的话）
    SendModRPCToServer(MOD_RPC.MyMod.UpdateSetting, "MY_FEATURE", new_data)
end
```

**`SendModRPCToServer(rpc_id, ...)` 是 mod 向服务器发数据的标准方式**——参数会被序列化后发送，服务器端的 handler 函数会接收到玩家实例 + 你的参数。

**重要**：只有**需要服务器感知的设置**才用 RPC；纯客户端 UI 偏好（比如"是否显示某个提示"）不需要同步，只存本地即可。

---

### 16.10.9 老手：焦点流与控制器支持

如果 mod 需要支持手柄或键盘导航，必须正确设置焦点流。

#### 第一步：为 Screen 设置 default_focus

```lua
-- 在构造函数末尾（所有控件都加好之后）
self.default_focus = self.feature_spinner
```

`default_focus` 是 Screen 被推入后**第一个获得焦点的 Widget**。详见 16.7.3。

#### 第二步：连接各控件的焦点链

```lua
-- 上下方向导航：feature → level → close_btn → feature（循环）
self.feature_spinner.spinner:SetFocusChangeDir(MOVE_DOWN, self.level_spinner.spinner)
self.level_spinner.spinner:SetFocusChangeDir(MOVE_UP,   self.feature_spinner.spinner)
self.level_spinner.spinner:SetFocusChangeDir(MOVE_DOWN, self.close_btn)
self.close_btn:SetFocusChangeDir(MOVE_UP,   self.level_spinner.spinner)
self.close_btn:SetFocusChangeDir(MOVE_DOWN, self.feature_spinner.spinner)
self.feature_spinner.spinner:SetFocusChangeDir(MOVE_UP, self.close_btn)
```

注意：`LabelSpinner` 返回的 wdg 已经设置了 `focus_forward = wdg.spinner`，所以对 `wdg`（LabelSpinner 本体）设置方向时，焦点会**通过 focus_forward 传递到内部的 Spinner**。但直接对 `wdg.spinner` 设方向更明确、不易出错。

#### 第三步：面板本身的 focus_forward

```lua
self.focus_forward = self.feature_spinner
```

当面板 Screen 被聚焦时，焦点通过 `focus_forward` 传递给第一个控件。

> **老手记忆**：`default_focus` 控制面板初始焦点；`SetFocusChangeDir` 连接 Spinner 之间的 Up/Down 导航；`focus_forward` 确保 Screen 的焦点能透传到子 Widget。

---

### 16.10.10 老手：五个常见坑

#### 坑 1：面板弹出后 HUD 缩放了但面板没跟着变——忘了监听 refreshhudsize

**症状**：玩家在设置面板打开时调整 HUD 缩放，面板大小不变（显得太大或太小）。

**原因**：没有监听 `"refreshhudsize"` 事件更新 `scalingroot` 的比例。

**修复**：在构造函数里添加事件监听，在回调里调 `self.scalingroot:SetScale(scale)`。注意监听的实体是 `owner.HUD.inst`，而不是 `TheWorld`。

#### 坑 2：`SetSelected` 调了但 Spinner 显示的还是第一个选项——data 类型不匹配

**症状**：`spinner:SetSelected("medium")` 调了，但 UI 显示的是第一个选项"低"。

**原因**：`spinnerdata` 里的 `data` 字段是字符串 `"medium"`，但存档读回来的值因为某种原因变成了 `nil` 或者不同类型。

**调试**：在 `selected_fn` 调用前 `print(type(MYMOD_SETTINGS.MY_LEVEL), MYMOD_SETTINGS.MY_LEVEL)` 确认类型正确。

**修复**：检查 `LoadMySettings` 里是否正确地把字符串/数字类型还原（Lua 的 `DataDumper` + `RunInSandbox` 会保留类型，但 `json.decode` 会把整数变 float）。

#### 坑 3：面板打开了两个——没检查 `mysettingsscreen == nil`

**症状**：快速点击两次 HUD 按钮，弹出两个设置面板叠在一起。

**原因**：`ShowMySettingsScreen` 里没有检查 `self.mysettingsscreen ~= nil`，每次点击都创建新 Screen。

**修复**：

```lua
self.ShowMySettingsScreen = function(_)
    if self.mysettingsscreen ~= nil then return end  -- 已经打开了
    -- ...
end
```

#### 坑 4：`OnCancel` 后面板没有真正关闭——忘了 `SetAutopaused(false)`

**症状**：按 Esc 关闭面板后，游戏仍处于暂停状态（菜单灰色，但面板消失了）。

**原因**：`SetAutopaused(true)` 在构造函数里调了，但 `OnCancel` 里忘了调 `SetAutopaused(false)`。

**修复**：确保每条关闭路径（Esc、点击遮罩、点关闭按钮）都调了 `SetAutopaused(false)`。

#### 坑 5：`AddClassPostConstruct` 里引用 `require` 路径错误——找不到 Screen 文件

**症状**：`require "screens/mymod_settingsscreen"` 报错 "module not found"。

**原因**：mod 的 require 路径是相对于 `mods/MyMod/scripts/` 的——`"screens/mymod_settingsscreen"` 对应 `mods/MyMod/scripts/screens/mymod_settingsscreen.lua`。路径拼写错误或文件名大小写不匹配（macOS/Linux 区分大小写）。

**修复**：确认文件实际存在的路径与 require 路径完全一致；在 modmain.lua 顶部先 `print(require("screens/mymod_settingsscreen"))` 验证能否加载。

---

### 16.10 小结——完整文件清单

**`scripts/mymod_settings.lua`**：
```lua
MYMOD_SETTINGS = { MY_FEATURE = "auto", MY_LEVEL = "medium" }
function SaveMySettings() ... TheSim:SetPersistentString("mymod_settings", str, false) end
function LoadMySettings() ... TheSim:GetPersistentString("mymod_settings", callback) end
LoadMySettings()
```

**`scripts/screens/mymod_settingsscreen.lua`**：
```
MySettingsScreen = Class(Screen, fn)
  scalingroot（HUD 缩放根）
    black（全屏遮罩，点击关闭）
    panel（TEMPLATES.CurlyWindow）
      title（Text）
      feature_spinner（TEMPLATES.LabelSpinner）
      level_spinner（TEMPLATES.LabelSpinner）
      close_btn（TEMPLATES.StandardButton）
  ListenForEvent("refreshhudsize", → scalingroot:SetScale)
OnCancel → PopScreen + SetAutopaused(false)
OnControl → Esc/M 键触发 OnCancel
```

**`scripts/mymod_hud_hook.lua`**：
```lua
AddClassPostConstruct("screens/playerhud", function(self)
    self.ShowMySettingsScreen = function(_) ... OpenScreenUnderPause(...) end
    self.CloseMySettingsScreen = function(_) ... mysettingsscreen:OnCancel() end
end)
```

**`modmain.lua`**：
```lua
require "mymod_settings"     -- 优先加载，立刻 LoadMySettings()
require "mymod_hud_hook"     -- 注入 HUD 方法
```

---

**新手核心三句**：设置面板 = Screen + scalingroot（缩放）+ black（遮罩）+ CurlyWindow（主体）；`TEMPLATES.StandardButton` 做按钮，`TEMPLATES.LabelSpinner` 做选项；`SetAutopaused(true/false)` 配对调用，不然游戏会一直卡在暂停状态。

**进阶核心三句**：持久化用 `TheSim:SetPersistentString("mymod_settings", DataDumper(data), false)` + `GetPersistentString` + `RunInSandboxSafe`；监听 `refreshhudsize` 在 `owner.HUD.inst` 上更新 `scalingroot` 缩放；`AddClassPostConstruct("screens/playerhud", fn)` 注入 `ShowMySettingsScreen` 是挂入 HUD 的标准方式。

**老手核心三句**：影响全服游戏逻辑的设置用 `SendModRPCToServer` + `AddModRPCHandler` 同步；`SetFocusChangeDir` + `focus_forward` 保证手柄导航；开/关面板的每条路径（Esc、按钮、遮罩）都必须调 `SetAutopaused(false)` 并清空 `self.mysettingsscreen = nil`。

**第 16 章到此收官**——从 Widget 原子（16.1）、布局锚点（16.2）、Screen 弹窗（16.3）、HUD 控件（16.4）、容器界面（16.5）、列表滚动（16.6）、焦点导航（16.7）、数据收集 UI（16.8）、控制台调试（16.9），到本节的设置面板实战（16.10），你已经掌握了饥荒联机版 UI 系统的完整技能树。
