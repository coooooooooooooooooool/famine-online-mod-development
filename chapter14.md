# 第14章 FX 特效系统

## 14.1 FX Prefab 的设计模式——SpawnPrefab 生成与自动回收

（待编写）

## 14.2 常用 FX Prefab 索引与复用（small_puff、sparks、splash、electricchargedfx 等）

### 本节导读

14.1 我们看了 FX 系统的"骨架"——`MakeFx` 模板、双 entity 模型、`animover` 自动回收。但是当你打开 mod 写一段技能效果时，你真正想知道的是：

> **"想做爆炸用什么、想做电弧用什么、想做闪光用什么、想做水花用什么？"**

而不是再写一遍 `MakeFx` 自己造轮子。**饥荒已经在 `scripts/fx.lua` 里准备好了 1000+ 个开箱即用的 FX**——本节就是这份"FX 索引手册"。

阅读路径：

> **新手**从 14.2.1-14.2.3 起步——记住 8 个"出现频率最高的 FX"和它们的标准用法、学会怎么改大小、怎么改颜色；**进阶读者**继续看 14.2.4-14.2.6，按主题（烟雾、水花、电击、火花、特殊效果）查阅 FX 大类索引、知道每一类用什么场景、读懂 sparks 和 electricchargedfx 这两个**非 MakeFx 模板**的复杂 FX 设计；**老手**跳到 14.2.7-14.2.8，掌握 FX 复用的进阶技巧——同一个 anim 资源派生多个 FX prefab、利用 tint 和 transform 派生效果变体、利用 `nameoverride` 给同一份特效改"调试名"的小窍门。

读完本节，你能：

- 在不查源码的情况下脱口而出 8-10 个常用 FX 名字及其用途
- 给自己的技能选对 FX，不需要重新做美术资源
- 看懂 mod 里别人写的 FX 调用，知道大小、颜色、位置都是怎么调出来的

---

### 14.2.1 快速入门：必会的 8 个常用 FX

下面这 8 个 FX 在饥荒源码里出现频率最高（每个都被用了几十到几百次），mod 作者必须记住：

| 序号 | FX 名 | 视觉效果 | 用法关键词 |
|------|-------|---------|-----------|
| 1 | `small_puff` | 浅灰色小烟雾 | "换形遮羞" |
| 2 | `splash` | 水花飞溅 | 入水/出水 |
| 3 | `sparks` | 橘色火花 + 光照 | 机械/电气损坏 |
| 4 | `die_fx` | 死亡时的红黑色烟雾 | 生物死亡 |
| 5 | `shock_fx` | 蓝色电击 + 光晕 | 被电击 |
| 6 | `groundpound_fx` | 地面震波尘土 | 地震/重击 |
| 7 | `mining_fx` | 采矿时的石头碎屑 | 矿石被破坏 |
| 8 | `electricchargedfx` | 持续的电荷光环 | 充电状态 buff |

#### 1. small_puff —— 万能烟雾遮羞布

`scripts/fx.lua` 第 183-188 行：

```lua
{
    name = "small_puff",
    bank = "small_puff",
    build = "smoke_puff_small",
    anim = "puff",
    sound = "dontstarve/common/deathpoof",
},
```

它的官方应用场景包括：

- `prefabs/hound.lua:723` —— hound 变形回标准形态
- `prefabs/flower.lua:121` —— 花变成玫瑰花
- `prefabs/petals.lua:21` —— 花瓣变成"邪恶花瓣"
- `standardcomponents.lua:1162` —— 通用 entity 替换工具
- `prefabs/meats.lua:97` —— 肉类变质
- `prefabs/chessjunk.lua:240` —— 损坏的棋子残骸

**统一规律**：当一个 entity "突然变成另一个 entity" 时，用 `small_puff` 盖住瞬变的视觉断层。

#### 2. splash —— 入水/出水的水花

`scripts/fx.lua` 第 70-76 行：

```lua
{
    name = "splash",
    bank = "splash",
    build = "splash",
    anim = "splash",
    sound = "turnoftides/common/together/water/splash/bird",
    fn = FinalOffset1,
},
```

它在源码里的典型用法（`prefabs/oceanfish.lua:68`）：

```lua
SpawnPrefab("splash").Transform:SetPosition(x, y, z)
```

简单粗暴。**记住凡是涉及"东西落水"或者"东西从水里跳出来"的瞬间，都用 splash**。

如果要让 splash 跟随某个 entity（比如玩家划水持续溅水），用 `SetParent`：

```lua
local splash = SpawnPrefab("splash")
splash.entity:SetParent(inst.entity)
```

这是 `SGwilson.lua:14887` 的真实写法。

#### 3. sparks —— 火花溅射

`sparks` 是个特殊家伙——**它不是用 `MakeFx` 模板做的**，而是有自己的源码 `scripts/prefabs/sparks.lua`。这是因为 sparks 需要一些 MakeFx 模板做不到的事：

- 需要 `Light` 组件（动态光照）
- 需要根据传入随机数选择 3 个变体之一
- 需要"光强渐弱"的逐帧动画

但是用法和普通 FX 一样简单（`prefabs/wx78.lua:283`）：

```lua
SpawnPrefab("sparks").Transform:SetPosition(x, y + 1 + math.random() * 1.5, z)
```

唯一注意点：**sparks 默认会被自动放大 2 倍**（`sparks.lua:152`：`inst.Transform:SetScale(2, 2, 2)`）。如果你觉得太大，需要手动调整。

#### 4. die_fx —— 死亡烟雾

`scripts/fx.lua` 第 56-62 行：

```lua
{
    name = "die_fx",
    bank = "die_fx",
    build = "die",
    anim = "small",
    sound = "dontstarve/common/deathpoof",
    tint = Vector3(90/255, 66/255, 41/255),
},
```

注意它的 `tint` 是默认值——一个偏褐色的死亡烟雾。但生物死亡时常常想用**自己的颜色**而不是默认褐色——这就要看 14.2.7 的复用技巧。

#### 5. shock_fx —— 电击效果

适合"瞬间被电一下"的场景。看 `scripts/fx.lua` 第 676 行附近会发现还有 `werebeaver_shock_fx`、`weremoose_shock_fx`、`weregoose_shock_fx`、`shock_arc_fx` 等多个变种——分别对应不同变身角色被电击的样式。

#### 6. groundpound_fx —— 地震效果

`scripts/fx.lua` 第 856-862 行附近。它有 `werebeaver_groundpound_fx` 变种。地震 FX 的特点是体积大，建议搭配镜头震动 `ShakeAllCameras` 一起用。

#### 7. mining_fx —— 采矿碎屑

`scripts/fx.lua` 第 330-340 行附近还有兄弟变种 `mining_ice_fx`（冰矿）、`mining_moonglass_fx`（月之玻璃）、`mining_charged_moonglass_fx`（带电月之玻璃）、`mining_crystal_fx`（水晶）。**统一规律**：用什么矿就配什么 fx。

#### 8. electricchargedfx —— 持续电荷状态

这是**带 SetTarget 接口的特殊 FX**——它会**主动跟随一个目标 entity**，直到目标失去状态后被清掉。`prefabs/foodbuffs.lua:89`：

```lua
SpawnPrefab("electricchargedfx"):SetTarget(target)
```

注意这种链式调用——`:SetTarget(target)` 是 electricchargedfx **专门暴露的方法**，不是所有 FX 都有。详见 14.2.5。

> **新手记忆**：8 个 FX 名字背下来，覆盖 90% 的特效需求。

---

### 14.2.2 快速入门：调整大小与位置

最常见的两个"二次加工"动作：

#### 大小：SetScale

```lua
local fx = SpawnPrefab("small_puff")
fx.Transform:SetPosition(x, y, z)
fx.Transform:SetScale(2, 2, 2)   -- 放大 2 倍
```

经验值：

- `0.5` —— 小型生物特效
- `1.0` —— 默认
- `1.5` —— 中型生物
- `2.0` —— 大型生物或重要事件
- `3.0+` —— boss 级特效

源码里 `chessjunk.lua:242` 用 1.5 表示棋子残骸，`spider_water.lua` 用默认 1.0。

#### 位置：SetPosition / GetWorldPosition

最规范的"获取并设置位置"模式：

```lua
local x, y, z = inst.Transform:GetWorldPosition()
SpawnPrefab("small_puff").Transform:SetPosition(x, y, z)
```

或者更紧凑的（多用一次 unpack）：

```lua
SpawnPrefab("small_puff").Transform:SetPosition(inst.Transform:GetWorldPosition())
```

这两种写法**完全等价**——`Transform:SetPosition(x, y, z)` 接收 3 个参数，而 `Transform:GetWorldPosition()` 正好返回 3 个值。

#### 抬高一点：让 FX 在 entity 头顶

`prefabs/wx78.lua:283`：

```lua
SpawnPrefab("sparks").Transform:SetPosition(x, y + 1 + math.random() * 1.5, z)
```

`y + 1` 表示比 entity 脚下抬高 1 米——FX 出现在身体中段。`+ math.random() * 1.5` 是随机抖动，让多个 sparks 不会全堆在同一个高度。这种"抬高+随机"是 mod 制作技能时常用的小窍门。

> **新手记忆**：FX 的二次加工就两件事——`SetScale` 调大小、`SetPosition` 调位置。再加一个 `+ y_offset` 让 FX 不卡地面。

---

### 14.2.3 快速入门：跟随父级 vs 独立位置

新手最常踩的坑是：**"我让 FX 跟着 Wilson，但 Wilson 死了 FX 还在原地飞舞"**。这是因为没用 SetParent。

#### 独立位置（FX 不跟随）

```lua
SpawnPrefab("splash").Transform:SetPosition(x, y, z)
```

**适用**：地面爆炸、矿石碎屑、入水水花——这些都是"瞬间事件"，不需要跟随。

#### 跟随父级（FX 与 entity 绑定）

```lua
local splash = SpawnPrefab("splash")
splash.entity:SetParent(inst.entity)
```

**适用**：玩家身上持续显示的 buff 光环、被电击时的电弧、毒雾覆盖等"持续状态"。

#### 关键区别

- 独立位置 FX：父级（entity）被删了，FX 不受影响
- 跟随父级 FX：父级被删 → FX 跟着自动被删（这是 entity 父子关系的默认行为）

这个区别影响很大。比如某个技能要在敌人脚下释放一团火焰：

- 用独立位置 → 敌人跑了，火焰留在原地（适合 AOE 区域伤害）
- 用 SetParent → 火焰跟着敌人移动（适合 DOT 单体技能）

> **新手记忆**：独立位置用 `SetPosition`，跟随父级用 `SetParent`。两者用途完全不同，**不能搞混**。

---

### 14.2.4 进阶：FX 主题分类索引（按用途速查）

进阶部分开始系统梳理 `scripts/fx.lua` 的 1000+ 个 FX。按"用途"分成 8 个大类：

#### 类别一：通用瞬变烟雾（puff）

| FX 名 | 颜色/效果 | 适用 |
|------|---------|------|
| `small_puff` | 浅灰小烟 | 通用换形 |
| `dirt_puff` | 同上但 FinalOffset1（盖在地面上） | 地面变形 |
| `sand_puff` | 沙黄色小烟 | 沙漠地形效果 |
| `sand_puff_large_front/back` | 沙黄色大烟 | 沙暴/大型沙地动作 |
| `shadow_puff` | 暗紫色小烟 | 暗影生物消失 |
| `shadow_puff_solid/solid_large` | 实体感更强的暗影烟 | shadow boss 动作 |
| `shadow_puff_large_front/back` | 大型暗影烟 | 大型暗影事件 |
| `disease_puff` | 黄绿色小烟 | 疾病/腐烂 |
| `bee_poof_big/small` | 蜜蜂相关黄色烟 | beebox 系统 |
| `itemmimic_puff` | 拟态 mimic 烟 | mimic 物品被识破 |

`shadow_puff_solid` 是个常用的暗色变种，比 `small_puff` 视觉冲击更强，适合"shadow"系生物动作。

#### 类别二：水花与水波（splash / ripple）

| FX 名 | 用途 |
|------|------|
| `splash` | 通用水花 |
| `splash_ocean` | 海洋专用水花（旧版） |
| `frogsplash` | 青蛙跳水声效不同 |
| `ink_splash` / `bile_splash` | 墨汁/胆汁水花 |
| `waterballoon_splash` | 水球破碎 |
| `ocean_splash_med1/med2` | 中等海面水花 |
| `ocean_splash_small1/small2` | 小型海面水花 |
| `ocean_splash_ripple1/ripple2` | 海面涟漪 |
| `ocean_splash_swim1/swim2` | 玩家划水 |
| `merm_splash` / `merm_king_splash` | 鱼人专用 |
| `wurt_water_splash_1/2/3` | Wurt 角色专用 |
| `splash_green/teal/black` | 颜色变体水花 |
| `weregoose_splash*` | 鹅形态变身水花 |

**进阶用法**：船在海里碰撞、玩家跳船等场景，用对应的 `ocean_*` 系列；陆地池塘用 `splash`；boss 战场景用大型变体。

#### 类别三：树叶与采集

| FX 名 | 用途 |
|------|------|
| `pine_needles` / `pine_needles_chop` | 松树（被采集/被砍） |
| `green_leaves` / `green_leaves_chop` | 绿色阔叶树 |
| `red_leaves` / `red_leaves_chop` | 红叶（落叶林） |
| `orange_leaves` / `purple_leaves` / `yellow_leaves` | 季节性变色 |
| `tree_petal_fx_chop` | 樱花树砍伐 |
| `sugarwood_leaf_fx_*` | 糖木 |
| `palmcone_leaf_fx_*` | 棕榈果树 |
| `oceantree_leaf_fx_*` | 海洋树（船树） |

注意它们都是**两个一组**：`xxx_leaves`（被采）和 `xxx_leaves_chop`（被砍）。差异是粒子飞溅角度——chop 偏向四散飞溅，采集偏向缓慢飘落。

#### 类别四：电击与火花

| FX 名 | 用途 |
|------|------|
| `sparks` | 通用机械火花（带光照） |
| `electrichitsparks` | 命中目标时的电火花 |
| `electrichitsparks_electricimmune` | 命中电免疫目标的"无效"火花 |
| `electricchargedfx` | 持续电荷光环（带 SetTarget） |
| `shock_fx` | 通用电击 |
| `shock_arc_fx` | 电弧（链式电击） |
| `werebeaver_shock_fx` | 海狸形态电击 |
| `weremoose_shock_fx` | 麋鹿形态电击 |
| `weregoose_shock_fx` | 鹅形态电击 |
| `lightning_rod_fx` | 避雷针放电 |
| `moonstorm_spark_shock_fx` | 月亮风暴电击 |

#### 类别五：火焰与爆炸

| FX 名 | 用途 |
|------|------|
| `firesplash_fx` | 火焰飞溅 |
| `tauntfire_fx` | 嘲讽用火焰 |
| `attackfire_fx` | 攻击火焰 |
| `vomitfire_fx` | 呕吐火焰（dragon 系列） |
| `halloween_firepuff_1/2/3` | 万圣节火焰 |
| `halloween_firepuff_cold_1/2/3` | 万圣节冰焰 |
| `halloween_moonpuff` | 万圣节月光 |
| `ember_short_fx` | 灰烬残火 |
| `spell_fire_throw` | 投掷火焰技能 |
| `willow_shadow_fire_explode` | Willow 暗影火焰爆炸 |
| `fire_fail_fx` | 点火失败 |
| `moon_geyser_explode` | 月间歇泉爆炸 |

#### 类别六：地面冲击与碎屑

| FX 名 | 用途 |
|------|------|
| `groundpound_fx` | 通用地面震波 |
| `werebeaver_groundpound_fx` | 海狸地面震波 |
| `mining_fx` | 通用采矿碎屑 |
| `mining_ice_fx` | 冰矿 |
| `mining_moonglass_fx` | 月之玻璃 |
| `mining_charged_moonglass_fx` | 带电月之玻璃 |
| `mining_crystal_fx` | 水晶 |
| `shovel_dirt` | 挖洞泥土 |
| `mole_move_fx` | 鼹鼠地下移动 |
| `cavein_debris` | 洞穴坍塌碎石 |
| `glass_fx` | 玻璃破碎 |
| `cannonball_used` | 炮弹爆炸 |
| `mortarball_used` | 迫击炮 |

#### 类别七：复活与变身

| FX 名 | 用途 |
|------|------|
| `spawn_fx_tiny/small/medium/large/huge` | 通用召唤 5 档 |
| `spawn_fx_medium_static/ocean_static` | 静态召唤 |
| `slurper_respawn` | slurper 重生 |
| `chester_transform_fx` | chester 变身 |
| `werebeaver/weremoose/weregoose_transform_fx` | 三种变身角色 |
| `werebeaver/weremoose_revert_fx` | 变身恢复 |
| `lucy_transform_fx` | Lucy 斧头变身 |
| `lucy_ground_transform_fx` | Lucy 落地变身 |
| `beefalo_transform_fx` | 牛宝宝变身 |
| `monkey_morphin_power_players_fx` | 猴子变身玩家 |
| `monkey_de_morphin_fx` | 猴子还原 |
| `oldager_become_younger_*` / `oldager_become_older_*` | Wanda 老化/年轻化 |
| `wormwood_lunar_transformation_finish` | Wormwood 月相变身 |
| `attune_in_fx` / `attune_out_fx` / `attune_ghost_in_fx` | 共鸣（重生石）光效 |

#### 类别八：buff 光环与状态指示

`scripts/fx.lua` 第 992-1426 行有大量 `battlesong_*` 和 `ghostlyelixir_*` —— 它们是 Wigfrid 战歌系统和 Wendy 鬼魂药剂系统的视觉表现。这些 FX 大多带 `nameoverride` 或 `description`，用于 inspect 时显示状态名。

完整列表过长，需要时直接搜 `scripts/fx.lua` 的 `battlesong_` 或 `ghostlyelixir_` 前缀即可定位。

> **进阶记忆**：8 个大类——puff、splash、leaves、电击、火焰、地面冲击、变身、buff 光环。每一类都有大量变种，不同生物/职业有专属 FX，**先从主题分类找，再筛颜色和大小**。

---

### 14.2.5 进阶：sparks 和 electricchargedfx——非 MakeFx 模板的复杂 FX

14.1 我们说"标准 FX 用 MakeFx 模板"，但是有些 FX 需要**模板做不到的事**——这时候要单独写 prefab。`sparks` 和 `electricchargedfx` 是两个典型例子。

#### sparks 为什么不能用 MakeFx？

打开 `scripts/prefabs/sparks.lua`，会发现它至少超出模板能力的三件事：

**1. 需要 Light 组件（动态光照）**

`sparks.lua:42-59`：

```lua
inst.entity:AddLight()
...
inst.Light:Enable(true)
inst.Light:SetRadius(2)
inst.Light:SetFalloff(1)
inst.Light:SetIntensity(.9)
inst.Light:SetColour(235 / 255, 121 / 255, 12 / 255)
```

火花溅出时不仅有视觉，**还应该有暖色光照映在周围地面上**——这是真实感的关键。但 MakeFx 模板没有 `AddLight` 选项。

**2. 需要逐帧光强衰减**

`sparks.lua:8-9`：

```lua
inst.Light:SetIntensity(inst.i)
inst.i = inst.i - dt * 2
```

光强从 0.9 在 0.5 秒内衰减到 0——这是个"逐帧动画"，必须用 `DoPeriodicTask` 实时更新。MakeFx 模板没有这种机制。

**3. 需要随机选 1/2/3 个动画变体**

`sparks.lua:140`：

```lua
inst:DoTaskInTime(0, StartFX, inst._rand:value(), build, sound)
```

注意这里调用 `StartFX(proxy, animindex, build, sound)` 时传入 `animindex` 是服务端生成的 1/2/3 之一。然后 `StartFX` 第 52 行：

```lua
inst.AnimState:PlayAnimation("sparks_"..tostring(animindex))
```

**每次出火花动画都不一样**——这种"播多个变体"的设计 MakeFx 模板用 `anim = function(...) end` 也能勉强做到，但因为还要配合 Light 衰减，整个 prefab 就需要单独写了。

#### electricchargedfx 为什么不能用 MakeFx？

打开 `scripts/prefabs/electric_charged_fx.lua`，关键差异在第 109-122 行：

```lua
local function SetTarget(inst, target)
    inst.entity:SetParent(target.entity)

    if inst.components.updatelooper == nil then
        inst.OnRemoveEntity = OnRemoveFlash
        inst.target = target
        inst.flash = 1
        inst.blink = 0

        inst:AddComponent("updatelooper")
        inst.components.updatelooper:AddOnUpdateFn(OnUpdateFlash)
        OnUpdateFlash(inst)
    end
end
```

它向外暴露了一个 `SetTarget(target)` 方法——**调用方主动告诉 FX "你跟谁绑定"**，然后 FX 内部不仅会 `SetParent`，**还会让目标本身的 AnimState 颜色每帧闪烁**（`OnUpdateFlash` 函数）：

```lua
inst.target.AnimState:SetAddColour(c, c, c, 0)
```

或者通过 `colouradder` 组件：

```lua
inst.target.components.colouradder:PushColour(inst, c, c, c, 0)
```

这种"FX 主动改父级 AnimState"的双向交互，MakeFx 模板没有任何字段能做到。

#### 调用方约定

记住：**FX 是否暴露 SetTarget / KillFX 等接口，由 prefab 自己决定**。判断方法：

```lua
local fx = SpawnPrefab("electricchargedfx")
if fx.SetTarget then
    fx:SetTarget(victim)
end
```

写自己的 mod FX 时，如果你需要"把 FX 绑定到目标 + 让目标产生颜色变化"，可以参考 `electric_charged_fx.lua` 的写法。

> **进阶记忆**：sparks 因为带 Light 和动态衰减、electricchargedfx 因为带 SetTarget 和目标 AnimState 修改——它们都是"模板外"的复杂 FX，但用法上对调用方依然是 `SpawnPrefab + 一两个方法`。

---

### 14.2.6 进阶：通过源码读 FX 表的快速技巧

进阶 mod 作者经常需要"我想做个××效果，饥荒有现成的 FX 吗？"——快速答案的方法是：

#### 技巧 1：按 anim 资源搜

如果你已经在游戏里看到过某个 FX 想复用，按它对应的 anim 资源名搜：

```
Grep pattern: build = "smoke_puff_small"
path: scripts/fx.lua
```

会列出所有用这个建模的 FX 配置。

#### 技巧 2：按 sound 搜

很多 FX 配的音效在多个 FX 共享，按音效搜可以找到一组同类 FX：

```
Grep pattern: dontstarve/common/deathpoof
path: scripts/fx.lua
```

会列出所有"死亡音效"FX。

#### 技巧 3：直接看 SpawnPrefab 调用方

最实战的方法——找一个已经做出你想要效果的官方 prefab，看它 SpawnPrefab 了什么：

```
Grep pattern: SpawnPrefab\(".*"\)
path: scripts/prefabs/wilson.lua
```

会列出 wilson.lua 里所有 FX 调用——你想做"和 wilson 类似的角色 FX"，照搬名字就行。

> **进阶记忆**：FX 复用的本质是"找对名字"。三个搜法（anim、sound、调用方）能覆盖 95% 的查找需求。

---

### 14.2.7 老手：复用同一个 anim 资源派生多个 FX 变体

`scripts/fx.lua` 第 303-309 行：

```lua
{
    name = "dirt_puff",
    bank = "small_puff",          -- ← 注意：bank 仍然是 small_puff
    build = "smoke_puff_small",   -- ← 注意：build 仍然是 smoke_puff_small
    anim = "puff",
    fn = FinalOffset1,            -- ← 唯一差异：加了 FinalOffset1
    --sound = "dontstarve/common/deathpoof",  -- 注释掉了音效
},
```

对比 `small_puff`：

```lua
{
    name = "small_puff",
    bank = "small_puff",
    build = "smoke_puff_small",
    anim = "puff",
    sound = "dontstarve/common/deathpoof",
},
```

**两个 FX 用的是同一个 anim 资源**，差异只有：

1. `dirt_puff` 加了 `fn = FinalOffset1`（设置 layer 让特效贴在地面上）
2. `dirt_puff` 没声音

这就是 FX 复用的精髓——**一份 anim 资源派生多个 FX 变体**，靠 `tint` / `transform` / `fn` / `sound` 四个字段微调。

#### 老手技巧 1：派生颜色变种

`scripts/fx.lua` 给同一个 anim 派生不同颜色的例子很常见：

```lua
-- 红色变体
{
    name = "myfx_red",
    bank = "small_puff",
    build = "smoke_puff_small",
    anim = "puff",
    tint = Vector3(1, 0.3, 0.3),
},
-- 蓝色变体
{
    name = "myfx_blue",
    bank = "small_puff",
    build = "smoke_puff_small",
    anim = "puff",
    tint = Vector3(0.3, 0.3, 1),
},
```

mod 里要给"火焰系/冰系/雷系"角色做一致风格的烟雾时，这种做法极其省美术成本。

#### 老手技巧 2：派生大小变种

```lua
{
    name = "myfx_big",
    bank = "small_puff",
    build = "smoke_puff_small",
    anim = "puff",
    transform = Vector3(2, 2, 2),   -- 比正常大 2 倍
},
```

`scripts/fx.lua` 第 615 行的 `spawn_fx_large` 和 622 行 `spawn_fx_huge` 就是同一个建模派生的不同大小。

#### 老手技巧 3：派生层级变种（FinalOffset）

`fn = FinalOffset1/2/3/-1/-2` 用于控制 FX 在画面中的渲染顺序——数字越大越靠前（盖在更多东西上面）。

```lua
local function FinalOffset1(inst)
    inst.AnimState:SetFinalOffset(1)
end
```

应用场景：

- `FinalOffsetNegative1/-2` —— FX 应该在 Wilson 后面（背景层）
- `FinalOffset1/2/3` —— FX 应该在 Wilson 前面（盖住身体）

`splash` 用 `FinalOffset1` 是因为水花要盖在 Wilson 鞋子上，不能让鞋子盖住水花。

#### 老手技巧 4：派生扬声器变种

```lua
{
    name = "myfx_loud",
    bank = "...",
    build = "...",
    anim = "...",
    sound = "myaudio/event/loud",
    sound2 = "myaudio/event/echo",   -- 第二段叠加音效
    sounddelay2 = 0.3,                -- 第二段延迟 0.3 秒
},
```

`MakeFx` 模板支持两个声音通道（`sound` 和 `sound2`），可以做"主音效+回响"的层次感。

> **老手记忆**：FX 复用 = 一份 anim + 多个配置卡片。tint、transform、fn、sound 四个字段足够派生几十个变体。

---

### 14.2.8 老手：nameoverride 与 description 的进阶用法

`MakeFx` 模板里有两个被新手忽略的字段——`nameoverride` 和 `description`。

#### 不写 = 静默 FX

`fx.lua` 第 30-32 行：

```lua
if t.nameoverride == nil and t.description == nil then
    inst:AddTag("FX")
end
```

注意：**只有 `nameoverride` 和 `description` 都没设的 FX，才会自动 AddTag("FX")**。这是因为带名字带描述的 FX 已经有了"被识别"的能力，不需要额外打 FX 标签。

`small_puff`、`splash`、`die_fx` 这些都是"静默 FX"——没有 nameoverride/description，玩家鼠标移上去看不到任何 inspect 文字。

#### nameoverride —— 让 FX 在调试模式可见

```lua
{
    name = "battlesong_attach_fx",
    ...
    nameoverride = "battlesong_attach",
}
```

设了 `nameoverride` 之后，`MakeFx` 会自动 `AddComponent("inspectable")`（fx.lua:79-85）：

```lua
if t.nameoverride ~= nil then
    if inst.components.inspectable == nil then
        inst:AddComponent("inspectable")
    end
    inst.components.inspectable.nameoverride = t.nameoverride
    inst.name = t.nameoverride
end
```

实战意义：

1. **调试时**——你按 ` ` 进入控制台，鼠标移到 FX 上，可以看到名字而不是空白
2. **战歌系统**——队友看到你身上的战歌名字"勇气"/"忠诚"/"飓风"等，靠的就是 nameoverride

#### description —— 让 FX 可以显示自定义描述

```lua
{
    name = "battlesong_durability_fx",
    ...
    nameoverride = "battlesong_durability_attach",
    description = function(inst, viewer)
        return "Wigfrid 战歌：装备耐久增益"
    end,
}
```

`description` 是一个 function，接收 `inst, viewer` 两个参数，返回一段文字。`MakeFx` 会自动接好 `inspectable.descriptionfn`（fx.lua:87-92）。

实战中，所有给玩家"看到 buff 持续状态"的 FX 都强烈推荐写 description——这样玩家不用查 wiki 就能知道身上飘的是啥效果。

#### 老手警示：nameoverride 设了之后丢失了 FX 标签

如果你设了 nameoverride，`AddTag("FX")` 不会被自动调用——但你的 FX 仍然是 FX，**很多系统依赖 `HasTag("FX")` 来排除特效**（比如 AOE 攻击只伤敌不伤特效）。所以——

**手动补打 FX 标签**：

```lua
{
    name = "myfx",
    ...
    nameoverride = "我的特效",
    description = function(inst, viewer) return "..." end,
    fn = function(inst, proxy)
        inst:AddTag("FX")    -- 手动补回
    end,
}
```

`fn` 字段是 MakeFx 的"客户端 startfx 完成后回调"——可以在这里做任何额外初始化。

> **老手记忆**：nameoverride 决定 FX 是否可见、description 决定 inspect 时显示什么文字、设了它们就要自己补 FX 标签——这是 FX 系统三个隐藏小机关。

---

### 本节小结

读完 14.2，你应该掌握：

| 难度 | 必会内容 |
|------|---------|
| 新手 | 8 个常用 FX 名字、SetScale/SetPosition 调整、独立位置 vs 跟随父级两种用法 |
| 进阶 | 8 个 FX 主题分类（puff、splash、leaves、电击、火焰、地面冲击、变身、buff），sparks 和 electricchargedfx 这两个非 MakeFx 模板的复杂 FX 设计原理 |
| 老手 | 同一个 anim 资源派生多个变体的 4 种技巧（tint/transform/fn/sound）、nameoverride 与 description 字段的隐藏行为（自动 inspectable + 不自动 AddTag("FX")） |

下一节 **14.3 ParticleEmitter——粒子发射器的配置与使用**——我们离开 anim-based FX，进入饥荒 FX 系统的另一条主线：**真正的粒子系统**。会讲 `ParticleEmitter` 组件的全部配置项、粒子寿命/初速度/重力的物理参数、为什么 ParticleEmitter 比 anim FX 更适合"持续粒子流"。

---


## 14.3 ParticleEmitter——粒子发射器的配置与使用

### 本节导读

14.1 / 14.2 我们看的 FX 全部基于 **AnimState 动画系统**——本质是"播一段美术做好的动画 → animover → 自毁"——**强项是确定性 + 美术可控**——但**缺点也很明显**：

1. **粒子数量有限**——一段动画顶多十几个 spriter symbol——做不出"500 个雪花同时飘"
2. **位置随机性差**——动画的轨迹是 spriter 师傅画死的——不能动态发射
3. **持续型效果难表达**——火把要永远燃烧——总不能循环播一段 0.5 秒动画

这时候——**真正的粒子系统登场**——`ParticleEmitter` / **`VFXEffect`**：游戏底层 GPU 粒子——**单个发射器可以生 1000+ 个独立粒子，每个有自己的位置、速度、寿命、颜色曲线、缩放曲线**——是火、烟、雪、雨、瘴气、电弧、传送门光柱**唯一的实现方式**。

读完 14.3 你会理解：

- **`ParticleEmitter` vs `VFXEffect`**——为什么"老 API + 新 API 共存"，新代码全用哪个
- **`AddParticle(idx, lifetime, px, py, pz, vx, vy, vz)`**——粒子诞生的"生死合同"
- **`EnvelopeManager`**——颜色 / 缩放随时间变化的"曲线动画"
- **`EmitterManager`**——周期触发 emit_fn 的"粒子节拍器"
- **`BlendMode` / `Bloom` / `Sort` / `Drag`**——粒子视觉的 5 个调节参数
- mod 怎么从 5 行代码起步——做出"自定义魔法烟柱"

> **新手**从 14.3.1 ~ 14.3.3 起步——理解 VFXEffect 是什么、最小代码、AddParticle 字段；**进阶读者**继续看 14.3.4 ~ 14.3.6——掌握 envelope 曲线、双层 emitter（torchfire 烟+火案例）、BlendMode/Bloom/Sort/Drag 5 参数；**老手**跳到 14.3.7 ~ 14.3.8——mod 自定义粒子标准模板 + 6 个常见陷阱。

---

### 14.3.1 快速入门：从一个火把看 ParticleEmitter 是什么

#### 第一步：观察一个火把在游戏里"长什么样"

打开游戏——点燃一个火把——盯着火苗看：

```
火把火苗 = 烟（灰色，向上飘）+ 火（橙红色，向上扇形扩散）
         ↑           ↑
         50 个粒子    30 个粒子
         循环出生    循环出生
         半透明      Additive 叠加
```

**你看到的不是动画——而是"一秒钟生了 80 个 GPU 粒子，每个有自己的命运"**——这就是粒子系统。

#### 第二步：源码——torchfire.lua 的核心

打开 `scripts/prefabs/torchfire.lua`——核心 70 行：

```119:148:scripts/prefabs/torchfire.lua
local effect = inst.entity:AddVFXEffect()
effect:InitEmitters(2)

--SMOKE
effect:SetRenderResources(0, SMOKE_TEXTURE, SHADER)
effect:SetMaxNumParticles(0, 64)
effect:SetMaxLifetime(0, SMOKE_MAX_LIFETIME)
effect:SetColourEnvelope(0, COLOUR_ENVELOPE_NAME_SMOKE)
effect:SetScaleEnvelope(0, SCALE_ENVELOPE_NAME_SMOKE)
effect:SetBlendMode(0, BLENDMODE.Premultiplied)
effect:EnableBloomPass(0, true)
effect:SetUVFrameSize(0, .25, 1)
effect:SetSortOrder(0, 0)
effect:SetSortOffset(0, 1)
effect:SetRadius(0, 2) --only needed on a single emitter

--FIRE
effect:SetRenderResources(1, TEXTURE, SHADER)
effect:SetMaxNumParticles(1, 64)
effect:SetMaxLifetime(1, FIRE_MAX_LIFETIME)
effect:SetColourEnvelope(1, COLOUR_ENVELOPE_NAME)
effect:SetScaleEnvelope(1, SCALE_ENVELOPE_NAME)
effect:SetBlendMode(1, BLENDMODE.Additive)
effect:EnableBloomPass(1, true)
effect:SetUVFrameSize(1, .25, 1)
effect:SetSortOrder(1, 0)
effect:SetSortOffset(1, 2)
```

**逐行读**：

- **`AddVFXEffect()`**——给 entity 加 VFXEffect 组件——这是"现代粒子 API"的入口
- **`InitEmitters(2)`**——初始化 2 个并行子发射器——索引 0（烟）+ 索引 1（火）
- **`SetRenderResources(idx, tex, shader)`**——绑定贴图 + GPU shader——`fx/smoke.tex` + `shaders/vfx_particle.ksh`
- **`SetMaxNumParticles(idx, 64)`**——发射器最多同时存在 64 个粒子——超过会"挤掉"最老的
- **`SetMaxLifetime(idx, .7)`**——粒子最长寿命 0.7 秒——到点自动消失
- **`SetColourEnvelope(idx, name)`**——粒子的颜色随寿命的变化曲线（14.3.4 详谈）
- **`SetScaleEnvelope(idx, name)`**——粒子的缩放随寿命的变化曲线
- **`SetBlendMode(idx, mode)`**——混合模式——烟用 Premultiplied（不发光），火用 Additive（发光叠加）
- **`EnableBloomPass(idx, true)`**——加 Bloom 光晕（让火更亮）
- **`SetUVFrameSize(idx, .25, 1)`**——UV 动画——把 1 张贴图分成 4 帧（0.25 = 1/4）

> **核心结论**：**一个火把 = 一个 VFXEffect + 2 个 Emitter（idx=0 烟 / idx=1 火）+ 一堆参数配置**——**完全没有动画文件——纯 GPU 实时计算**。

#### 第三步：ParticleEmitter vs VFXEffect

**老 API**：`inst.entity:AddParticleEmitter()`（单 emitter）

**新 API**：`inst.entity:AddVFXEffect()` + `effect:InitEmitters(N)`（多 emitter 复合）

**两者区别**：

| 维度 | ParticleEmitter（老） | VFXEffect（新） |
| --- | --- | --- |
| Emitter 数量 | 1 个 | N 个（InitEmitters(N)） |
| 表达力 | 单一颜色 / 大小 | 多层叠加（火+烟+灰烬） |
| 性能 | 略低 | 优化 |
| API 风格 | `emitter:AddParticle(...)` | `effect:AddParticle(idx, ...)` |
| 现代代码 | **几乎不用** | **新代码全用** |

**源码 `scripts/components/emitter.lua` 的兼容代码**：

```lua
local effect = self.inst.VFXEffect
local emitter = self.inst.ParticleEmitter --legacy

if effect then
    effect:SetMaxNumParticles( 0, ...)
else
    emitter:SetMaxNumParticles( ...)
end
```

——`Emitter` component 同时支持两种 API——但**新代码全部走 VFXEffect**。**mod 推荐 VFXEffect**。

> **铁律**：新写粒子代码——**永远用 `AddVFXEffect()`**——`AddParticleEmitter()` 是为了兼容旧代码保留的。

#### 第四步：粒子系统适用场景对照表

| 视觉效果 | 用动画 fx (14.1) | 用粒子 (14.3) |
| --- | --- | --- |
| 一次性烟雾爆开 | ✓ small_puff | × 杀鸡用牛刀 |
| 持续燃烧的火 | × 难循环 | ✓ torchfire |
| 持续雾气 | × 资源大 | ✓ miasma_cloud_fx |
| 雪 / 雨 / 落叶 | × 数量少 | ✓ snow / rain / pollen |
| 角色头顶 buff 光环 | ✓ ghostlyelixir | × 多此一举 |
| 传送门发光柱 | × 光强弱难控 | ✓ portal_fx |
| 武器拖尾 | × 数量难控 | ✓ pocketwatch_weapon_fx |

> **判断标准**：**"持续 + 大量 + 动态位置"——粒子；"一次性 + 视觉细节"——动画 fx**——通常**两者搭配使用**（火苗用粒子 + 火把本体用动画）。

---

### 14.3.2 快速入门：5 行代码加一个最简粒子源

#### 第一步：最最最简单的粒子 prefab

```lua
-- scripts/prefabs/myfx_simple_smoke.lua
local TEXTURE = "fx/smoke.tex"
local SHADER = "shaders/vfx_particle.ksh"

local assets = {
    Asset("IMAGE", TEXTURE),
    Asset("SHADER", SHADER),
}

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst:AddTag("FX")
    inst.persists = false

    if not TheNet:IsDedicated() then
        local effect = inst.entity:AddVFXEffect()
        effect:InitEmitters(1)
        effect:SetRenderResources(0, TEXTURE, SHADER)
        effect:SetMaxNumParticles(0, 32)
        effect:SetMaxLifetime(0, 1.5)
        effect:SetBlendMode(0, BLENDMODE.AlphaBlended)

        -- 每 0.05 秒发一个粒子
        inst:DoPeriodicTask(0.05, function()
            local px, py, pz = 0, 0, 0
            local vx, vy, vz = 0.01 * (math.random() - 0.5), 0.5, 0.01 * (math.random() - 0.5)
            effect:AddParticle(
                0,        -- emitter index
                1.5,      -- lifetime
                px, py, pz,
                vx, vy, vz
            )
        end)
    end

    return inst
end

return Prefab("myfx_simple_smoke", fn, assets)
```

**这 30 行代码——已经能在游戏里出现一个"持续向上飘的简易烟雾源"**——比从零做动画 fx 节省 80% 工作量。

#### 第二步：逐行解读"最少必要参数"

| API | 含义 | 必填？ |
| --- | --- | --- |
| `AddVFXEffect()` | 加 VFXEffect 组件 | ✓ |
| `InitEmitters(N)` | 创建 N 个子发射器 | ✓ |
| `SetRenderResources(idx, tex, shader)` | 绑贴图 + shader | ✓ |
| `SetMaxNumParticles(idx, n)` | 最大并发数 | ✓ |
| `SetMaxLifetime(idx, t)` | 粒子最长存活 | ✓ |
| `SetBlendMode(idx, mode)` | 混合模式 | 推荐 |
| `AddParticle(idx, lt, px, py, pz, vx, vy, vz)` | 真正生粒子 | ✓ |

> **6 个 API + 一个 AddParticle 调用——粒子系统 90% 的能力你都摸到了**。剩下 10% 是 envelope / blend / bloom / sort（14.3.4 ~ 14.3.6）。

#### 第三步：Dedicated 服务器要绕开

注意 `if not TheNet:IsDedicated() then` 这个条件——和 14.1.5 学过的一样——**dedicated 服务器没渲染层**——AddVFXEffect 会报错或浪费 CPU。

**铁律**：**任何 VFXEffect 相关代码都包在 `if not TheNet:IsDedicated()` 里**——14.3.7 老手陷阱再强调。

#### 第四步：用 SpawnPrefab 触发它

```lua
-- 业务方
local fx = SpawnPrefab("myfx_simple_smoke")
fx.Transform:SetPosition(x, y, z)

-- 5 秒后销毁
fx:DoTaskInTime(5, fx.Remove)
```

**注意**：粒子 prefab **没有 animover 自动回收**——粒子源本身**不会自己死**——必须**外部调 Remove**（14.3.8 陷阱 1 详细讲）。

> **这是粒子系统和动画 fx 最大的区别**——动画 fx "播完就死"——粒子 fx "永远活着，除非你杀它"。

#### 第五步：和动画 fx 联用——粒子 + small_puff

很多场景两者**搭配**最好：

```lua
-- 法术爆裂——粒子 + 动画 fx 组合
local function Cast(caster)
    local x, y, z = caster.Transform:GetWorldPosition()

    -- 一次性动画爆烟
    SpawnPrefab("small_puff").Transform:SetPosition(x, y, z)

    -- 持续粒子（用完即销）
    local fx = SpawnPrefab("myfx_simple_smoke")
    fx.Transform:SetPosition(x, y, z)
    fx:DoTaskInTime(2, fx.Remove)  -- 2 秒粒子源
end
```

**视觉**：瞬间一个大爆烟 → 接着 2 秒的小烟雾"残留"——更有层次感。

---

### 14.3.3 快速入门：粒子的"出生"机制——AddParticle + EmitterManager

#### 第一步：AddParticle 的 8 个参数

```lua
effect:AddParticle(
    emitter_index,    -- 第几个子发射器（0-based）
    lifetime,         -- 寿命（秒）
    px, py, pz,       -- 出生位置（相对于 entity）
    vx, vy, vz        -- 初速度（米/秒）
)
```

**3 个关键概念**：

1. **位置是"相对"的**——`(px, py, pz)` 是相对于 entity（粒子源）的偏移——**默认 (0, 0, 0) = entity 中心**
2. **速度是"米/秒"**——`vy = 0.5` 表示每秒向上 0.5 米——粒子寿命 1.5 秒 → 最终位置 y = 0.75
3. **寿命由两道闸门决定**——AddParticle 传的 `lifetime` 不能超过 `SetMaxLifetime`——**超了就被裁剪**

#### 第二步：AddParticleUV / AddRotatingParticleUV 变体

VFXEffect 还有更高阶的"出生函数"：

```lua
-- 带 UV 偏移（从贴图的不同位置取像素——做"多种粒子样式"）
effect:AddParticleUV(
    idx, lifetime,
    px, py, pz,
    vx, vy, vz,
    u_offset, v_offset    -- 0~1 范围
)

-- 带旋转（粒子自转）
effect:AddRotatingParticleUV(
    idx, lifetime,
    px, py, pz,
    vx, vy, vz,
    angle,                -- 起始角度（度）
    angle_velocity,       -- 旋转速度（度/秒）
    u_offset, v_offset
)
```

**典型用法**：

```lua
-- torchfire 用 AddParticleUV——4 帧贴图随机切片
local uv_offset = math.random(0, 3) * .25  -- 0 / 0.25 / 0.5 / 0.75
effect:AddParticleUV(
    0, lifetime,
    px, py, pz,
    vx, vy, vz,
    uv_offset, 0
)
```

> 为什么 4 帧？因为火苗 sprite 画了 4 种形状——通过随机 UV 让每个粒子**长得不一样**——视觉更丰富。

#### 第三步：EmitterManager —— 周期发射的"节拍器"

如果每帧都要 emit——`DoPeriodicTask` 不够精细——用 **`EmitterManager`**：

```lua
EmitterManager:AddEmitter(inst, sleeper_check, update_fn)
```

**3 个参数**：

- **`inst`**——粒子源 entity
- **`sleeper_check`**——一般传 `nil`（emitter 自己跟随 entity 的睡眠状态）
- **`update_fn`**——每帧调一次的发射函数

#### 第四步：标准发射节奏代码

```lua
local effect = inst.entity:AddVFXEffect()
effect:InitEmitters(1)
effect:SetRenderResources(0, TEXTURE, SHADER)
effect:SetMaxNumParticles(0, 64)
effect:SetMaxLifetime(0, 0.7)

-- 每秒 80 个粒子（"密度"参数）
local desired_pps = 80
local tick_time = TheSim:GetTickTime()  -- 一帧时长
local particles_per_tick = desired_pps * tick_time

-- 累加器——避免"每帧 0.x 个"被截断
local num_to_emit = 0

EmitterManager:AddEmitter(inst, nil, function()
    num_to_emit = num_to_emit + particles_per_tick
    while num_to_emit >= 1 do
        -- emit 一个粒子
        local vx, vy, vz = .01 * UnitRand(), .05, .01 * UnitRand()
        local lifetime = 0.7 * (.9 + UnitRand() * .1)  -- 寿命随机化
        effect:AddParticle(0, lifetime, 0, 0, 0, vx, vy, vz)
        num_to_emit = num_to_emit - 1
    end
end)
```

**关键技巧**：

1. **`particles_per_tick = pps * tick_time`**——把"每秒粒子数"转换为"每帧粒子数"
2. **`num_to_emit` 累加器**——浮点数累加 → 整数 emit 一次 → 减 1——**避免每帧丢小数**
3. **`UnitRand()` 是 `-1 ~ 1` 的随机数**——`.01 * UnitRand()` = `-0.01 ~ 0.01`——给粒子"轻微飘移"
4. **`lifetime * (.9 + UnitRand() * .1)`**——寿命在 `0.9 ~ 1.1` 倍变化——**避免"所有粒子同时死"造成断层**

#### 第五步：CreateSphereEmitter / CreateCircleEmitter ——区域发射

如果粒子要在**一个区域**而不是单点出生——用区域发射器：

```lua
local sphere_emitter = CreateSphereEmitter(.05)  -- 半径 0.05 球内随机出生

EmitterManager:AddEmitter(inst, nil, function()
    local px, py, pz = sphere_emitter()  -- 调用就生成一个球内随机点
    effect:AddParticle(0, lifetime, px, py, pz, vx, vy, vz)
end)
```

**`miasma_cloud_fx`（瘴气云）的真实代码**：

```lua
local smoke_circle_emitter = CreateCircleEmitter(SMOKE_RADIUS)

local ox, oz = smoke_circle_emitter()
effect:AddRotatingParticleUV(
    0, lifetime,
    ox, oy, oz,
    vx, vy, vz,
    math.random() * 360,
    UnitRand() * 0.1,
    uv_offset, 0
)
```

——粒子在**半径 SMOKE_RADIUS 的圆内随机出生**——这就是"一团雾气"而不是"一个点"的原因。

> 区域发射器三个**：`CreateSphereEmitter(r)` / `CreateCircleEmitter(r)` / `CreateBoxEmitter(w, h, d)`**——任选一个匹配你的视觉需求。

---

### 14.3.4 进阶：Envelope 曲线——颜色 / 缩放随时间变化

#### 第一步：什么是 Envelope（包络）

**Envelope** = "随时间变化的值"——在粒子系统里——**粒子的颜色 / 缩放 / 不透明度 / 速度**都可以接 envelope——粒子从生到死的整个生命周期**自动按曲线变化**。

**典型场景**：

- 火粒子——出生时半透明黄 → 中段最亮红 → 末段熄灭
- 烟粒子——出生时密 → 末段稀 → 完全消失
- 雪粒子——大小恒定 → 但透明度从 0 渐变到 100%

**没 envelope = 整条命都同一个颜色 / 大小 → 视觉非常假**。

#### 第二步：颜色 Envelope 注册

源码 `torchfire.lua`：

```47:57:scripts/prefabs/torchfire.lua
EnvelopeManager:AddColourEnvelope(
    COLOUR_ENVELOPE_NAME,
    {
        { 0,    IntColour(187, 111, 60, 128) },
        { .49,  IntColour(187, 111, 60, 128) },
        { .5,   IntColour(255, 255, 0, 128) },
        { .51,  IntColour(255, 30, 56, 128) },
        { .75,  IntColour(255, 30, 56, 128) },
        { 1,    IntColour(255, 7, 28, 0) },
    }
)
```

**`AddColourEnvelope(name, points)` 解读**：

- **`name`**——曲线名（字符串）——后续 `SetColourEnvelope` 引用
- **`points`** —— 数组形式的关键点——`{ t, RGBA }`
- **`t` 是 0 ~ 1**——0 = 粒子出生瞬间，1 = 粒子最后一刻——**所有粒子共享这条曲线**

torchfire 火苗的"7 个关键帧"：

```
t=0    ├── 187,111,60,128  暗黄褐色（暖色）
t=0.49 ├── 187,111,60,128  保持暖色
t=0.5  ├── 255,255,0,128   突变亮黄
t=0.51 ├── 255,30,56,128   突变红
t=0.75 ├── 255,30,56,128   保持红
t=1    └── 255,7,28,0      渐隐到完全透明
```

**视觉**：火苗出生时是暖色 → 中段瞬间黄 → 然后变红 → 渐渐熄灭——**这就是"火"的真实颜色变化**。

#### 第三步：IntColour Helper

**为什么要 IntColour**：底层 Envelope 用的是 0~1 浮点数 RGBA——但人类喜欢 0~255 整数——helper 函数做转换：

```lua
local function IntColour(r, g, b, a)
    return { r / 255, g / 255, b / 255, a / 255 }
end
```

> **mod 推荐复用**——直接 copy 这个 helper 函数。

#### 第四步：缩放 Envelope（Vector2）

```60:66:scripts/prefabs/torchfire.lua
local max_scale = 3
EnvelopeManager:AddVector2Envelope(
    SCALE_ENVELOPE_NAME,
    {
        { 0,    { max_scale * .5, max_scale } },
        { 1,    { max_scale * .5 * .5, max_scale * .5 } },
    }
)
```

**缩放是 Vector2**——`{ scale_x, scale_y }`——粒子是 2D 平面 sprite——所以**只有 x/y**没有 z。

**火苗**：

- 出生时——`x=1.5, y=3`（**高瘦**——刚冒出火舌）
- 死亡时——`x=0.75, y=1.5`（**矮宽**——熄灭时分散）

**视觉**：火苗从"挺直" → "矮宽"——**真实火焰扩散感**。

#### 第五步：连接 envelope 到 emitter

```lua
effect:SetColourEnvelope(0, COLOUR_ENVELOPE_NAME)
effect:SetScaleEnvelope(0, SCALE_ENVELOPE_NAME)
```

**注意**：

- **`COLOUR_ENVELOPE_NAME`** 是字符串名字——必须在前面 `AddColourEnvelope` 注册过
- **`Set*Envelope(idx, name)`** 是给第 idx 个 emitter 设——多 emitter 各设自己的

> **一个 envelope 可以被多个粒子源复用**——比如 mod 注册一次"我的火曲线"——所有自定义火 prefab 都用——节省内存。

#### 第六步：完整 envelope 对比 5 个例子

下面是**游戏内 5 个真实 envelope** —— mod 作者**直接抄**：

**例 1：torchfire 烟（淡入持平淡出）**

```lua
{
    { 0,    IntColour(35, 32, 30, 0) },
    { .3,   IntColour(35, 32, 30, 100) },
    { .55,  IntColour(30, 30, 30, 28) },
    { 1,    IntColour(30, 30, 30, 0) },
}
```

**例 2：torchfire 火（黄→红渐隐，14.3.4 第二步代码）**

**例 3：miasma_cloud 烟（持平 + 头尾淡出）**

```lua
{
    { 0,    IntColour(255, 255, 255, 0) },
    { .1,   IntColour(255, 255, 255, 255) },
    { .9,   IntColour(255, 255, 255, 255) },
    { 1,    IntColour(255, 255, 255, 0) },
}
```

**例 4：miasma_cloud 余烬（多色变化）**

```lua
{
    { 0,    IntColour(200, 85, 60, 25) },
    { .2,   IntColour(230, 140, 90, 200) },
    { .3,   IntColour(255, 90, 70, 255) },
    { .6,   IntColour(255, 90, 70, 255) },
    { .9,   IntColour(255, 90, 70, 230) },
    { 1,    IntColour(255, 70, 70, 0) },
}
```

**例 5：通用淡入淡出（最简）**

```lua
{
    { 0,   { 1, 1, 1, 0   } },
    { 0.2, { 1, 1, 1, 1   } },
    { 0.8, { 1, 1, 1, 1   } },
    { 1,   { 1, 1, 1, 0   } },
}
```

> **5 个 envelope 当模板用——足以表达 80% mod 视觉需求**。

---

### 14.3.5 进阶：双层 emitter——torchfire 烟+火案例完整解读

#### 第一步：为什么要双 emitter

火把视觉**至少**有两层：

```
[最底层]  烟雾 —— 不发光 / 半透明 / 慢
   ↓
[中间层]  火焰本体 —— 发光 / Additive / 快
   ↓
[最上层]  火星余烬 —— 极快 / 极小 / 偶尔
```

**单 emitter 表达不了"两种粒子并存且参数完全不同"**——必须用 **`InitEmitters(2)` 双层**。

#### 第二步：完整 torchfire 文件（拆解）

```119:178:scripts/prefabs/torchfire.lua
local effect = inst.entity:AddVFXEffect()
effect:InitEmitters(2)

--SMOKE
effect:SetRenderResources(0, SMOKE_TEXTURE, SHADER)
effect:SetMaxNumParticles(0, 64)
effect:SetMaxLifetime(0, SMOKE_MAX_LIFETIME)
effect:SetColourEnvelope(0, COLOUR_ENVELOPE_NAME_SMOKE)
effect:SetScaleEnvelope(0, SCALE_ENVELOPE_NAME_SMOKE)
effect:SetBlendMode(0, BLENDMODE.Premultiplied)
effect:EnableBloomPass(0, true)
effect:SetUVFrameSize(0, .25, 1)
effect:SetSortOrder(0, 0)
effect:SetSortOffset(0, 1)
effect:SetRadius(0, 2)

--FIRE
effect:SetRenderResources(1, TEXTURE, SHADER)
effect:SetMaxNumParticles(1, 64)
effect:SetMaxLifetime(1, FIRE_MAX_LIFETIME)
effect:SetColourEnvelope(1, COLOUR_ENVELOPE_NAME)
effect:SetScaleEnvelope(1, SCALE_ENVELOPE_NAME)
effect:SetBlendMode(1, BLENDMODE.Additive)
effect:EnableBloomPass(1, true)
effect:SetUVFrameSize(1, .25, 1)
effect:SetSortOrder(1, 0)
effect:SetSortOffset(1, 2)

local tick_time = TheSim:GetTickTime()

local smoke_desired_pps = 80
local smoke_particles_per_tick = smoke_desired_pps * tick_time
local smoke_num_particles_to_emit = -50 --start delay

local fire_desired_pps = 40
local fire_particles_per_tick = fire_desired_pps * tick_time
local fire_num_particles_to_emit = 1

local sphere_emitter = CreateSphereEmitter(.05)

EmitterManager:AddEmitter(inst, nil, function()
    --SMOKE
    while smoke_num_particles_to_emit > 1 do
        emit_smoke_fn(effect, sphere_emitter)
        smoke_num_particles_to_emit = smoke_num_particles_to_emit - 1
    end
    smoke_num_particles_to_emit = smoke_num_particles_to_emit + smoke_particles_per_tick

    --FIRE
    while fire_num_particles_to_emit > 1 do
        emit_fire_fn(effect, sphere_emitter)
        fire_num_particles_to_emit = fire_num_particles_to_emit - 1
    end
    fire_num_particles_to_emit = fire_num_particles_to_emit + fire_particles_per_tick
end)
```

**核心模式**：

1. **InitEmitters(2)** —— 准备 2 槽
2. **idx=0 用一套配置**（烟 / Premultiplied / 较慢） / **idx=1 用另一套**（火 / Additive / 较快）
3. **`EmitterManager` 单循环**——同时跑两套发射逻辑——一次循环**同时 emit 烟 + 火**

#### 第三步：双发射器的"差异表"

| 字段 | 烟 (idx=0) | 火 (idx=1) |
| --- | --- | --- |
| Texture | smoke.tex | torchfire.tex |
| MaxNumParticles | 64 | 64 |
| MaxLifetime | 0.7 (SMOKE_MAX_LIFETIME) | 0.3 (FIRE_MAX_LIFETIME) |
| BlendMode | Premultiplied（不发光） | Additive（发光叠加） |
| Bloom | 开 | 开 |
| SortOffset | 1 | 2（火渲染在烟之上） |
| pps | 80 | 40 |
| start delay | -50（先冒一会儿烟） | 1（立即出火） |

> **关键设计**：**火出现得比烟晚（start delay -50 vs 1）**——视觉上**先有烟再有火**——**符合现实**。

#### 第四步：emit_smoke_fn / emit_fire_fn

```77:105:scripts/prefabs/torchfire.lua
local function emit_smoke_fn(effect, sphere_emitter)
    local vx, vy, vz = .01 * UnitRand(), .05, .01 * UnitRand()
    local lifetime = SMOKE_MAX_LIFETIME * (.9 + UnitRand() * .1)
    local px, py, pz = sphere_emitter()
    local uv_offset = math.random(0, 3) * .25

    effect:AddParticleUV(
        0,
        lifetime,
        px, py, pz,
        vx, vy, vz,
        uv_offset, 0
    )
end

local function emit_fire_fn(effect, sphere_emitter)
    local vx, vy, vz = .01 * UnitRand(), 0, .01 * UnitRand()
    local lifetime = FIRE_MAX_LIFETIME * (.9 + UnitRand() * .1)
    local px, py, pz = sphere_emitter()
    local uv_offset = math.random(0, 3) * .25

    effect:AddParticleUV(
        1,
        lifetime,
        px, py, pz,
        vx, vy, vz,
        uv_offset, 0
    )
end
```

**对比两个函数**：

- 都用同一个 `sphere_emitter`（半径 0.05 的小球）取出生位置
- 烟有上升速度 `vy = .05`——**烟向上飘**
- 火没有上升速度 `vy = 0`——**火只在原地舞动**
- 都用 4 帧 UV——`math.random(0, 3) * .25`

> **对比的精妙之处**：**两套粒子用相似 emit_fn——但参数不同——视觉效果完全不同**——这是粒子系统**参数化设计**的力量。

#### 第五步：mod 复用模板

如果你 mod 想做"自定义魔法烟柱（蓝色）"——**抄 torchfire 改 4 行**：

```lua
-- 改 1：贴图
local SMOKE_TEXTURE = "fx/smoke.tex"   -- 沿用游戏自带
-- 或：local SMOKE_TEXTURE = "anim/myfx_smoke.tex"  -- 自己的

-- 改 2：颜色 envelope —— 改成蓝色
EnvelopeManager:AddColourEnvelope(
    "myfx_blue_smoke",
    {
        { 0,    IntColour(60, 90, 200, 0) },
        { .3,   IntColour(60, 90, 200, 100) },
        { .55,  IntColour(80, 110, 230, 28) },
        { 1,    IntColour(80, 110, 230, 0) },
    }
)

-- 改 3：连接到 emitter
effect:SetColourEnvelope(0, "myfx_blue_smoke")

-- 改 4：去掉 fire emitter（只保留烟）
effect:InitEmitters(1)  -- 只 1 个
```

**4 行改动 = 一个全新的"蓝魔法烟柱"** —— 这就是 envelope + emitter 系统的复用威力。

---

### 14.3.6 进阶：BlendMode / Bloom / Sort / Drag —— 5 大视觉参数详解

#### 第一步：BlendMode —— 混合模式

`SetBlendMode(idx, mode)` 共 4 种：

| Mode | 视觉 | 适用 |
| --- | --- | --- |
| `BLENDMODE.AlphaBlended` | 标准半透明叠加 | 烟 / 雾 / 默认 |
| `BLENDMODE.Premultiplied` | 预乘 alpha——更高效 | 复杂烟 / 大量粒子 |
| `BLENDMODE.Additive` | 颜色叠加——发光 | 火 / 闪电 / 魔法 |
| `BLENDMODE.Disabled` | 无混合 —— 完全不透明 | 实体粒子 / 雪花 |

**核心原则**：**发光物用 Additive；半透明物用 AlphaBlended / Premultiplied**——不会反过来。

#### 第二步：Bloom —— 光晕

```lua
effect:EnableBloomPass(idx, true)
```

**含义**：粒子加上 Bloom 后处理——**亮的部分会向外"渗出"光晕**——视觉上**更亮、更梦幻**。

**适用**：火 / 魔法 / 月亮 / 等离子——**所有"应该发光"的粒子**。

**反面例子**：雪 / 灰尘 / 烟 —— **不该发光**——不要开 Bloom——会"白茫茫一片很假"。

#### 第三步：Sort —— 排序

```lua
effect:SetSortOrder(idx, 0)        -- 整体层
effect:SetSortOffset(idx, 1)       -- 在层内的偏移
```

**含义**：**多个 emitter 之间的 Z 排序**——后渲染的盖住先渲染的。

**torchfire 的 SortOffset**：

- 烟 SortOffset=1
- 火 SortOffset=2
- **火渲染在烟之上**——视觉正确

**反面**：如果烟 SortOffset=2、火=1——**烟会盖住火**——火被烟挡——**视觉错乱**。

> **多 emitter 永远要设 SortOffset**——按"最终视觉的前后关系"分配数字大小。

#### 第四步：UVFrameSize —— UV 帧动画

```lua
effect:SetUVFrameSize(idx, .25, 1)   -- 0.25 = 横向 4 帧
```

**含义**：把贴图分成 N 列 × M 行——`AddParticleUV` 时通过 `uv_offset` 选具体哪一帧。

**典型用法**：

```lua
-- 4 帧贴图（torchfire）
effect:SetUVFrameSize(0, .25, 1)  -- 0.25 横向 → 4 帧

local uv_offset = math.random(0, 3) * .25  -- 选 0/1/2/3 帧
effect:AddParticleUV(0, lt, px, py, pz, vx, vy, vz, uv_offset, 0)
```

> **UV 帧让粒子"长得不一样"**——4 个粒子用 4 种 sprite 形状——视觉丰富度 ×4。

#### 第五步：Drag / DragCoefficient —— 阻力

```lua
effect:SetDragCoefficient(idx, 0.1)
```

**含义**：粒子运动时受到的"空气阻力"——**让粒子从快变慢**——更自然。

**0 = 没阻力（粒子永远以初速度飞）；1 = 高阻力（粒子立刻减速到 0）**。

**典型用法**：

- 烟 / 雾 ——`0.1`（轻微阻力——看起来像"空气稠密"）
- 火星 / 余烬 ——`0.07`（轻阻——上升一段后慢下来）
- 雪 / 雨 ——`0`（自由落体，无阻力）
- 流体 / 水滴 ——`0.5`（强阻力——重而黏）

#### 第六步：Radius —— 单 emitter 的辐射范围

```lua
effect:SetRadius(idx, 2)
```

**含义**：**告诉引擎这个 emitter 的"视觉范围"**——用于"距离剔除"——粒子源**距离玩家很远**时——引擎可以**完全跳过更新**——节省性能。

**典型值**：

- 火把 / 蜡烛 ——`2` 米
- 火堆 ——`3 ~ 5` 米
- 大雾 ——`SMOKE_RADIUS`（几十米）
- 雪 / 雨 ——按摄像机 frustum 算

> **设 Radius 是性能优化的关键**——大世界场景下**让远距离粒子源不更新**——10 个火把同时存在也不卡。

#### 第七步：5 参数总配比

最佳实践组合：

| 视觉 | BlendMode | Bloom | UVFrame | Drag | Radius |
| --- | --- | --- | --- | --- | --- |
| 火 | Additive | 开 | (.25, 1) | 0.05 | 2 |
| 烟 | Premultiplied | 开 | (.25, 1) | 0.1 | 2 |
| 雪 | AlphaBlended | 关 | (1, 1) | 0 | frustum |
| 雨 | AlphaBlended | 关 | (1, 1) | 0 | frustum |
| 闪电 | Additive | 开 | (.5, 1) | 0 | 5 |
| 魔法 | Additive | 开 | (.25, 1) | 0.07 | 4 |
| 雾 | AlphaBlended/Premultiplied | 关 | (.5, 1) | 0.1 | 大半径 |

> **5 个参数 + 这张表 = 80% 视觉风格能直接套**。

---

### 14.3.7 老手进阶：mod 自定义粒子的标准模板

#### 第一步：完整 mod 粒子 prefab 模板

下面是**生产级模板**——直接复制改参数即可：

```lua
-- scripts/prefabs/myfx_magic_pillar.lua
local TEXTURE = "fx/torchfire.tex"     -- 沿用游戏火贴图（节省美术）
local SHADER = "shaders/vfx_particle.ksh"

local COLOUR_ENVELOPE_NAME = "myfx_magic_pillar_colour"
local SCALE_ENVELOPE_NAME = "myfx_magic_pillar_scale"

local PARTICLE_MAX_LIFETIME = 0.5

local assets = {
    Asset("IMAGE", TEXTURE),
    Asset("SHADER", SHADER),
}

local function IntColour(r, g, b, a)
    return { r / 255, g / 255, b / 255, a / 255 }
end

local function InitEnvelope()
    EnvelopeManager:AddColourEnvelope(
        COLOUR_ENVELOPE_NAME,
        {
            { 0,   IntColour(80, 100, 230, 0) },
            { 0.3, IntColour(120, 140, 255, 200) },
            { 0.7, IntColour(180, 100, 255, 200) },
            { 1,   IntColour(220, 80, 180, 0) },
        }
    )

    local max_scale = 2
    EnvelopeManager:AddVector2Envelope(
        SCALE_ENVELOPE_NAME,
        {
            { 0,   { max_scale * 0.4, max_scale * 0.4 } },
            { 0.3, { max_scale,       max_scale       } },
            { 1,   { max_scale * 0.5, max_scale * 0.5 } },
        }
    )

    InitEnvelope = nil
    IntColour = nil
end

local function emit_fn(effect, sphere_emitter)
    local px, py, pz = sphere_emitter()
    local vx, vy, vz = .01 * UnitRand(), .3, .01 * UnitRand()
    local lifetime = PARTICLE_MAX_LIFETIME * (0.9 + UnitRand() * 0.1)
    local uv_offset = math.random(0, 3) * 0.25

    effect:AddParticleUV(
        0,
        lifetime,
        px, py, pz,
        vx, vy, vz,
        uv_offset, 0
    )
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst:AddTag("FX")
    inst.persists = false
    inst.entity:SetCanSleep(false)

    if not TheNet:IsDedicated() then
        if InitEnvelope ~= nil then
            InitEnvelope()
        end

        local effect = inst.entity:AddVFXEffect()
        effect:InitEmitters(1)

        effect:SetRenderResources(0, TEXTURE, SHADER)
        effect:SetMaxNumParticles(0, 64)
        effect:SetMaxLifetime(0, PARTICLE_MAX_LIFETIME)
        effect:SetColourEnvelope(0, COLOUR_ENVELOPE_NAME)
        effect:SetScaleEnvelope(0, SCALE_ENVELOPE_NAME)
        effect:SetBlendMode(0, BLENDMODE.Additive)
        effect:EnableBloomPass(0, true)
        effect:SetUVFrameSize(0, 0.25, 1)
        effect:SetSortOrder(0, 0)
        effect:SetSortOffset(0, 1)
        effect:SetRadius(0, 2)
        effect:SetDragCoefficient(0, 0.05)

        local tick_time = TheSim:GetTickTime()
        local desired_pps = 100
        local particles_per_tick = desired_pps * tick_time
        local num_to_emit = 0

        local sphere_emitter = CreateSphereEmitter(.1)

        EmitterManager:AddEmitter(inst, nil, function()
            num_to_emit = num_to_emit + particles_per_tick
            while num_to_emit >= 1 do
                emit_fn(effect, sphere_emitter)
                num_to_emit = num_to_emit - 1
            end
        end)
    end

    return inst
end

return Prefab("myfx_magic_pillar", fn, assets)
```

```lua
-- modmain.lua
PrefabFiles = {
    "myfx_magic_pillar",
}
```

```lua
-- 业务方使用
local fx = SpawnPrefab("myfx_magic_pillar")
fx.Transform:SetPosition(x, y, z)
fx:DoTaskInTime(3, fx.Remove)  -- 3 秒后销毁
```

#### 第二步：模板中的"7 个必修"

| 必修项 | 代码位置 | 作用 |
| --- | --- | --- |
| ① `if not TheNet:IsDedicated()` | fn 入口 | 服务端不跑 |
| ② `inst.persists = false` | fn 入口 | 不存档 |
| ③ `inst.entity:SetCanSleep(false)` | fn 入口 | 粒子不休眠 |
| ④ `InitEnvelope()` 单次注册 | 第一次 fn 时 | envelope 全局注册 |
| ⑤ `InitEmitters(N)` + 全套 Set 配置 | AddVFXEffect 后 | 配置参数 |
| ⑥ `EmitterManager:AddEmitter` + emit_fn | 配置完成后 | 周期发射 |
| ⑦ 业务方负责 `Remove` | 调用方 | 粒子源销毁 |

> **缺一不可**——少任一项要么粒子不出现、要么内存泄漏、要么 dedicated 报错。

#### 第三步：粒子源生命周期管理 4 种模式

**模式 1：固定时长**

```lua
local fx = SpawnPrefab("myfx_magic_pillar")
fx:DoTaskInTime(3, fx.Remove)
```

**模式 2：跟随父对象——父死跟死**

```lua
local fx = SpawnPrefab("myfx_magic_pillar")
fx.entity:SetParent(target.entity)
-- 不需要 Remove——target 销毁时 fx 自动销毁
```

**模式 3：业务驱动手动 Remove**

```lua
inst.fx = SpawnPrefab("myfx_magic_pillar")
inst.fx.entity:SetParent(inst.entity)

inst:ListenForEvent("buff_end", function()
    if inst.fx then
        inst.fx:Remove()
        inst.fx = nil
    end
end)
```

**模式 4：周期短粒子重生（替代长期粒子源）**

```lua
inst:DoPeriodicTask(0.5, function()
    SpawnPrefab("myfx_magic_pillar"):DoTaskInTime(0.5, function(fx) fx:Remove() end)
end)
```

> **生产级 mod 永远显式管理粒子源生命周期**——不像动画 fx 有 animover 自动死。

#### 第四步：mod 视觉性能 4 个建议

1. **MaxNumParticles 适中**——64 ~ 128 通常够用——别上 1024（性能爆炸）
2. **Radius 不要太大**——大半径关闭距离剔除——所有玩家附近都更新——性能浪费
3. **emit pps 不要太高**——80 ~ 100 通常够——超 200 会有可见 stutter
4. **复用 envelope**——多个粒子 prefab 共享一个 envelope——节省内存

---

### 14.3.8 老手进阶：六个常见陷阱

#### 陷阱 1：粒子源永不销毁——内存泄漏

**症状**：mod 加自定义粒子——玩 1 小时后帧数从 60 → 30 → 10。

**原因**：粒子 prefab **没 animover**——开发者忘了 `Remove`——粒子源累积成几十几百个。

**修复**：业务方**永远显式 `Remove`**——参考 14.3.7 的 4 种生命周期模式。

```lua
-- ❌
SpawnPrefab("myfx_magic_pillar")  -- 没绑父、没定时——永远活着

-- ✓
local fx = SpawnPrefab("myfx_magic_pillar")
fx.entity:SetParent(caster.entity)  -- 跟随父对象死
-- 或
fx:DoTaskInTime(2, fx.Remove)
```

---

#### 陷阱 2：dedicated 服务器报错

**症状**：本地测试完美——上 dedicated 服务器**第一次进入触发粒子的位置**——"attempt to call AddVFXEffect (a nil value)" 红字 + 服务器 crash。

**原因**：dedicated 没 graphics——`inst.entity:AddVFXEffect()` 是 nil。

**修复**：**所有 VFXEffect 代码必须包 `if not TheNet:IsDedicated()`**——14.3.2 强调过。

```lua
if not TheNet:IsDedicated() then
    local effect = inst.entity:AddVFXEffect()
    effect:InitEmitters(1)
    -- ...
end
```

---

#### 陷阱 3：EnvelopeManager 重复注册

**症状**：mod 写了 envelope——但**第二次进入世界**——envelope 失效——粒子变成纯白色。

**原因**：

```lua
local function InitEnvelope()
    EnvelopeManager:AddColourEnvelope("name", {...})
end

local function fn()
    InitEnvelope()  -- ★ 每次 fn 都调一次——重复注册——后注册的覆盖前注册的——但有时引擎清空导致失效
end
```

**修复**：**只注册一次——用闭包 trick**：

```lua
local function InitEnvelope()
    EnvelopeManager:AddColourEnvelope("name", {...})
    InitEnvelope = nil  -- ★ 设为 nil——只跑一次
end

local function fn()
    if InitEnvelope ~= nil then
        InitEnvelope()
    end
end
```

> 这是 torchfire / miasma_cloud_fx 等所有源码用的**标准模式**——直接抄。

---

#### 陷阱 4：emit_fn 里调 components

**症状**：mod 在 emit_fn 里写了 `inst.components.health:DoDelta(-1)`——服务端 crash。

**原因**：emit_fn 跑在**客户端**（dedicated 不跑）——`inst.components.health` 在客户端是 nil。

**修复**：**emit_fn 只做 effect:AddParticle 调用——业务逻辑放别处**：

```lua
-- ❌ 在 emit_fn 里加业务
EmitterManager:AddEmitter(inst, nil, function()
    inst.components.health:DoDelta(-1)  -- ★ 错位置
    effect:AddParticle(...)
end)

-- ✓ 业务和 emit 分开
inst:DoPeriodicTask(1, function()
    if inst.components.health then  -- ★ 服务端业务
        inst.components.health:DoDelta(-1)
    end
end)

EmitterManager:AddEmitter(inst, nil, function()
    effect:AddParticle(...)  -- ★ 客户端粒子
end)
```

> **粒子是客户端纯视觉——业务逻辑用 component / SG / etc 表达**。

---

#### 陷阱 5：MaxNumParticles 太小——粒子断流

**症状**：mod 火堆——pps 设了 80——但**只有 16 个粒子在飞**——感觉断断续续。

**原因**：

```lua
effect:SetMaxNumParticles(0, 16)  -- ★ 太小
```

——粒子 emit 速度快于"老粒子死亡速度"——新粒子被**挤掉**——视觉上"少了一半"。

**计算公式**：

```
MaxNumParticles ≥ pps × max_lifetime
```

举例：`pps = 80, max_lifetime = 0.7`——`MaxNumParticles ≥ 56`——**至少 64**才安全。

**修复**：用上面公式 + 留 20% 余量。

---

#### 陷阱 6：BlendMode + Bloom 用错

**症状**：mod 做"暗影粒子"——但视觉上看着像"满屏白光"——压根看不见暗影感。

**原因**：

```lua
effect:SetBlendMode(0, BLENDMODE.Additive)  -- ★ 暗色用加法 → 颜色叠加变白
effect:EnableBloomPass(0, true)             -- ★ 又加 Bloom → 越叠越白
```

——Additive **永远让画面变亮**——暗色粒子用 Additive **失效**。

**修复**：**暗色 / 半透明粒子用 AlphaBlended 或 Premultiplied + 关 Bloom**：

```lua
effect:SetBlendMode(0, BLENDMODE.AlphaBlended)  -- ★ 半透明
effect:EnableBloomPass(0, false)                -- ★ 关 Bloom
```

> **铁律**：**亮发光粒子 → Additive + Bloom；暗 / 半透粒子 → AlphaBlended/Premultiplied + 关 Bloom**——按 14.3.6 第 7 步配比表。

---

#### 设计经验三条

**经验 ①：从 torchfire / miasma_cloud_fx 抄起**

> 不要从零写——抄一份生产代码改 4 ~ 8 个参数——80% 工作完成。

**经验 ②：服务端永远绕过粒子代码**

> `if not TheNet:IsDedicated()` 是粒子 mod 的"安全帽"——无脑加。

**经验 ③：粒子源生命周期必须显式管理**

> 粒子 prefab 没有 animover 自动死——业务方用 SetParent / DoTaskInTime / 手动 Remove 三选一。

---

### 14.3.9 小结

#### 一句话总结

**ParticleEmitter / VFXEffect 是 GPU 粒子系统——`AddVFXEffect + InitEmitters(N) + SetRenderResources / Envelope / BlendMode 等参数 + EmitterManager 周期发射 emit_fn 调用 AddParticle`——能做出火 / 烟 / 雪 / 雨 / 雾 / 闪电 / 魔法等"持续 + 大量 + 动态"的视觉——是动画 fx 表达不了的领域；mod 标准模板 100 行直接抄即可**。

#### 速查 6 表

**粒子 vs 动画 fx 选择表**

| 视觉 | 选 |
| --- | --- |
| 一次性 / 美术细节 | 动画 fx |
| 持续 / 大量 / 动态 | 粒子 |

**VFXEffect 6 步使用流程**

| 步 | 代码 |
| --- | --- |
| 1 | `inst.entity:AddVFXEffect()` |
| 2 | `effect:InitEmitters(N)` |
| 3 | `SetRenderResources / MaxNumParticles / MaxLifetime` |
| 4 | `SetColourEnvelope / SetScaleEnvelope` |
| 5 | `SetBlendMode / Bloom / Sort / UVFrameSize / Drag / Radius` |
| 6 | `EmitterManager:AddEmitter(inst, nil, function() effect:AddParticle(...) end)` |

**5 参数视觉风格速查**

| 视觉 | BlendMode | Bloom | UVFrame | Drag | Radius |
| --- | --- | --- | --- | --- | --- |
| 火 | Additive | 开 | (.25, 1) | 0.05 | 2 |
| 烟 | Premultiplied | 开 | (.25, 1) | 0.1 | 2 |
| 雪 / 雨 | AlphaBlended | 关 | (1, 1) | 0 | frustum |
| 闪电 | Additive | 开 | (.5, 1) | 0 | 5 |
| 魔法 | Additive | 开 | (.25, 1) | 0.07 | 4 |
| 雾 | AlphaBlended | 关 | (.5, 1) | 0.1 | 大半径 |

**Envelope 5 个模板**

| 模板 | 用途 |
| --- | --- |
| 持平 + 头尾淡出 | 通用烟 / 雾 |
| 暖色 → 黄 → 红 → 渐隐 | 火 |
| 蓝色淡入持平淡出 | 魔法 |
| 0.4 → 1.0 → 0.5 缩放 | 火苗扩散 |
| 1 → 0.1 缩放 | 余烬消散 |

**生命周期 4 模式**

| 模式 | 适用 |
| --- | --- |
| `DoTaskInTime(t, Remove)` | 固定时长 |
| `SetParent(target.entity)` | 跟随对象 |
| 业务事件 → `Remove` | buff / state 驱动 |
| 周期短粒子重生 | 替代长期源 |

**6 个陷阱排雷顺序**

1. 粒子源永不销毁 → 内存泄漏 → 必须显式 Remove
2. 没包 IsDedicated → 服务端 crash → 必加保护
3. Envelope 重复注册 → 用闭包 InitEnvelope = nil
4. emit_fn 调 components → 业务和 emit 分开
5. MaxNumParticles 太小 → ≥ pps × lifetime
6. BlendMode + Bloom 用错 → 按表配比

#### 3 条设计经验

- **① 抄 torchfire / miasma_cloud_fx 起步**——节省 80% 工作量
- **② Dedicated 永远绕过粒子代码**——`if not TheNet:IsDedicated()`
- **③ 显式管理粒子源生命周期**——SetParent / DoTaskInTime / Remove 三选一

#### mod 自定义粒子 4 行起步

```lua
-- 抄 14.3.7 模板 → 改 envelope 颜色 + texture
PrefabFiles = { "myfx_magic_pillar" }

-- 业务方 3 行触发
local fx = SpawnPrefab("myfx_magic_pillar")
fx.Transform:SetPosition(x, y, z)
fx:DoTaskInTime(3, fx.Remove)
```

> **下一节预告**：14.4 节我们将进入**ColourCube 与视觉后处理**——前面 14.1 / 14.2 / 14.3 讲的都是"局部视觉"——而 ColourCube 是**全屏后处理**——一秒钟可以把整个游戏画面**变冷 / 变暖 / 变成日落色 / 变成毒雾色**——是夜晚 / 洞穴 / 月相 / 噩梦战的关键氛围工具——14.4 会把它的 LUT 表配置、运行时切换、mod 自定义 cube 一次讲透。

---


## 14.4 ColourCube 与视觉后处理——画面氛围的动态切换

### 本节导读

14.1 / 14.2 / 14.3 都是"局部视觉"——FX 是单点爆发、粒子是局部范围——**它们改变不了"整张屏幕的色调"**。

但是游戏里**最重要的氛围切换**——比如：

- 春夏秋冬切换——画面整体冷暖完全不同
- 白天 → 黄昏 → 黑夜——画面整体偏黄 → 偏红 → 偏蓝紫
- 玩家发疯（insanity）——画面**饱和度下降 + 偏暗红**
- 玩家月狂（lunacy）——画面**偏冷蓝紫 + 略带眩晕扭曲**
- 月暴（moonstorm）——画面**电流闪光 + 蓝白噪点**
- 钻洞穴——一秒钟从草原变成漆黑洞穴

——这些**全屏色调变化**——背后是**同一个系统**：**ColourCube + PostProcessor**。

读完 14.4 你会理解：

- **ColourCube 是什么**——LUT (Look-Up Table) 32×32×32 颜色查找表的 GPU 实现
- **3 个通道**——ambient / insanity / lunacy 各自的职责分工
- **Blend 渐变机制**——`SetColourCubeData(idx, src, dest)` + `SetColourCubeLerp(idx, t)` 双参数
- **事件驱动切换**——phasechanged / seasontick / sanitydelta / overridecolourcube / ccoverrides 5 大事件
- **mod 修改 3 种路径**——简单 override / 自定义 cctable / 自定义 phase fn
- **PostProcessor 全家桶**——除 ColourCube 外的 Bloom / Distort / Lunacy / MoonPulse 4 大后处理 shader

> **新手**从 14.4.1 ~ 14.4.3 起步——理解 ColourCube 是什么、3 通道职责、mod 触发 3 种最简方式；**进阶读者**继续看 14.4.4 ~ 14.4.6——Blend 渐变源码、事件驱动机制、自定义 cctable 完整方案；**老手**跳到 14.4.7 ~ 14.4.8——PostProcessor 全家桶 + 6 个常见陷阱。

---

### 14.4.1 快速入门：从一次"季节切换"看 ColourCube 是什么

#### 第一步：观察一次秋 → 冬切换

游戏跑进冬天的瞬间——**整个屏幕**：

```
[秋季白天 cc]              [冬季白天 cc]
     ↓                            ↓
  暖黄褐色 ←——10 秒线性渐变——→  冷白蓝色
```

10 秒里——**草地 / 树木 / 角色 / UI** 全部颜色慢慢偏移——草从青黄变冷绿、树叶从橙红变深棕——**没有任何 entity 重新加载**——整个画面**通过一张"颜色查找表"瞬间换皮**。

#### 第二步：ColourCube 是什么——LUT 表

**LUT (Look-Up Table)** —— 一张 **32×32×32 的颜色查找表**（实际存为 1024×32 的 2D 贴图）：

```
输入：屏幕上某个像素的 RGB（比如 (200, 150, 80)）
        ↓
查表：在 LUT 里找 row=200, col=150, layer=80 的对应像素
        ↓
输出：变换后的 RGB（比如 (180, 130, 100) —— 略微冷一些）
```

**整个屏幕每一个像素都查这张表**——一帧之内 **1920×1080 = 200 万次查询**——但**全在 GPU 上跑**——**几乎免费**。

**对比**：如果用 CPU 算"全屏调色"——**直接卡死游戏**。GPU LUT 是**唯一可行的方案**。

#### 第三步：源码——cube 资源全在 `images/colour_cubes/`

```19:64:scripts/components/colourcube.lua
local INSANITY_COLOURCUBES =
{
    day = "images/colour_cubes/insane_day_cc.tex",
    dusk = "images/colour_cubes/insane_dusk_cc.tex",
    night = "images/colour_cubes/insane_night_cc.tex",
    full_moon = "images/colour_cubes/insane_night_cc.tex",
}

local LUNACY_COLOURCUBES =
{
	regular = "images/colour_cubes/lunacy_regular_cc.tex",
    full_moon = "images/colour_cubes/purple_moon_cc.tex",
    moon_storm = "images/colour_cubes/moonstorm_cc.tex",
}

local SEASON_COLOURCUBES =
{
    autumn =
    {
        day = "images/colour_cubes/day05_cc.tex",
        dusk = "images/colour_cubes/dusk03_cc.tex",
        night = "images/colour_cubes/night03_cc.tex",
        full_moon = "images/colour_cubes/purple_moon_cc.tex"
    },
    winter =
    {
        day = "images/colour_cubes/snow_cc.tex",
        dusk = "images/colour_cubes/snowdusk_cc.tex",
        night = "images/colour_cubes/night04_cc.tex",
        full_moon = "images/colour_cubes/purple_moon_cc.tex"
    },
    -- ... spring / summer 略 ...
}
```

**关键观察**：

- **每个季节**（autumn / winter / spring / summer）**有 4 张 cc**：day / dusk / night / full_moon
- **疯狂状态**有 3 张：day / dusk / night
- **月狂状态**有 3 张：regular / full_moon / moon_storm
- **洞穴**只有 1 张：caves_default
- **identity_colourcube**：无变化的"恒等" cc——画面什么都不改

#### 第四步：identity_colourcube ——基准

```17:17:scripts/components/colourcube.lua
local IDENTITY_COLOURCUBE = "images/colour_cubes/identity_colourcube.tex"
```

**含义**：**输入什么颜色就输出什么颜色**——画面不变化——和"没启用 ColourCube"等价。

**3 个通道初始化时全用它**——确保游戏启动后画面是"原色"。

> **mod 自定义 cube 也用这个作"中立基准"**——制作 mod cc 时**先复制 identity 再 Photoshop 调色**——保证关键参考点。

#### 第五步：ColourCube 系统总览

```
[ 输入：屏幕原色 ]
          ↓
[ Channel 0: Ambient CC（季节+时段+洞穴+override）]
          ↓
[ Channel 1: Insanity CC（疯狂滤镜，和 ambient 叠加）]
          ↓
[ Channel 2: Lunacy CC（月狂滤镜，和前两者叠加）]
          ↓
[ Distort（玩家疯狂时的画面扭曲，独立 effect）]
          ↓
[ Bloom（粒子发光叠加，独立 effect）]
          ↓
[ MoonPulse / MoonPulseGrading（觉醒月相专用）]
          ↓
[ 输出：玩家看到的最终画面 ]
```

**总管 = `PostProcessor`（C++ 全局对象）**——Lua 通过 `postprocesseffects.lua` 包装的 API 调用。

> **核心结论**：**ColourCube 是"GPU 后处理调色"的 Lua 接口——通过 3 个通道叠加 + Blend 渐变 + 事件驱动 + override 钩子——表达整个游戏所有"全屏氛围"**。

---

### 14.4.2 快速入门：3 通道（ambient / insanity / lunacy）职责分工

#### 第一步：Channel 0 — Ambient（环境）

**用途**：**季节 + 时段 + 洞穴 + override**——决定"这个时空地点的基础色调"。

**优先级**（高 → 低）：

```
1. _overridecctable（mod / 道具临时改）
2. _iscave 时强制 CAVE_COLOURCUBES
3. SEASON_COLOURCUBES[_season]
4. fallback autumn
```

**触发事件**：`seasontick`（换季）、`phasechanged`（换昼夜）、`moonphasechanged2`（换月相）、`ccoverrides`（mod / 道具 override）

#### 第二步：Channel 1 — Insanity（疯狂）

**用途**：**玩家精神值低时的"暗黑滤镜"**——和 Ambient 叠加。

**资源**：

```lua
INSANITY_COLOURCUBES = {
    day = "images/colour_cubes/insane_day_cc.tex",
    dusk = "images/colour_cubes/insane_dusk_cc.tex",
    night = "images/colour_cubes/insane_night_cc.tex",
    full_moon = "images/colour_cubes/insane_night_cc.tex",
}
```

**强度**：通过 `SetColourCubeLerp(1, sanity_distortion)` 控制——**满精神时 lerp=0（不影响），完全疯狂时 lerp=1（最大叠加）**。

```256:258:scripts/components/colourcube.lua
local lunacy_distortion = 1 - easing.outQuad(lunacy_percent, 0, 1, 1)
local sanity_distortion = 1 - easing.outQuad(sanity_percent, 0, 1, 1)
```

**关键函数**：`OnSanityDelta` —— 玩家精神值变化时被调——动态调整 lerp。

#### 第三步：Channel 2 — Lunacy（月狂）

**用途**：**月狂角色专属**（如 Wagstaff、月异变形角色）—— 类似 insanity 但**画面偏冷蓝紫**。

**资源**：

```lua
LUNACY_COLOURCUBES = {
	regular = "images/colour_cubes/lunacy_regular_cc.tex",
    full_moon = "images/colour_cubes/purple_moon_cc.tex",
    moon_storm = "images/colour_cubes/moonstorm_cc.tex",
}
```

**强度控制**：`_lunacyintensity`——0 ~ 1 渐变。当玩家从 sanity 模式切换到 lunacy 模式时——`_lunacyintensity` 从 0 慢慢涨到 1。

#### 第四步：3 通道叠加顺序

```440:447:scripts/components/colourcube.lua
--Channel 0: ambient colour cube
--Channel 1: insanity colour cube
--Channel 2: lunacy colour cube
PostProcessor:SetColourCubeData(0, _ambientcc[1], _ambientcc[2])
PostProcessor:SetColourCubeData(1, _insanitycc[1], _insanitycc[2])
PostProcessor:SetColourCubeData(2, _lunacycc[1], _lunacycc[2])
PostProcessor:SetColourCubeLerp(0, 1)
PostProcessor:SetColourCubeLerp(1, 0)
```

**叠加顺序**：

1. 先经过 Channel 0（ambient）—— Lerp = 0/1（src/dest 渐变）
2. 再经过 Channel 1（insanity）—— Lerp = 0~1（强度由精神值决定）
3. 最后经过 Channel 2（lunacy）—— Lerp = 0~1（强度由月狂值决定）

> **3 通道是"加法"叠加**——画面**先确定基础色调（ambient），再加疯狂滤镜，最后加月狂滤镜**——三者**正交、可并存**。

#### 第五步：3 通道职责对照表

| 通道 | idx | 用途 | 触发 | 强度 |
| --- | --- | --- | --- | --- |
| Ambient | 0 | 季节 / 时段 / 洞穴 / mod override | seasontick / phasechanged | 总是 1 |
| Insanity | 1 | 疯狂滤镜 | sanitydelta | sanity_distortion |
| Lunacy | 2 | 月狂滤镜 | sanitydelta + lunacy mode | lunacy_distortion |

> **mod 改 ambient 最常见**——比如做"魔法森林"切到自定义偏紫绿色调；改 insanity / lunacy 极少。

---

### 14.4.3 快速入门：mod 触发"全屏变色"的 3 种最简方式

#### 第一步：方式 1 —— `overridecolourcube` 事件（最简）

**适用**：**临时全屏变色**——比如玩家戴了"红色眼镜道具"——按下时全屏变红。

```lua
-- 触发
TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/red_filter_cc.tex")

-- 取消
TheWorld:PushEvent("overridecolourcube", nil)
```

**源码**：

```412:427:scripts/components/colourcube.lua
local function OnOverrideColourCube(inst, cc)
    if _overridecc ~= cc then
        _overridecc = cc

        if cc ~= nil then
            PostProcessor:SetColourCubeData(0, cc, cc)
            PostProcessor:SetColourCubeData(1, cc, cc)
            PostProcessor:SetColourCubeLerp(0, 1)
            PostProcessor:SetColourCubeLerp(1, 0)
        else
            PostProcessor:SetColourCubeData(0, _ambientcc[2], _ambientcc[2])
            PostProcessor:SetColourCubeData(1, _insanitycc[2], _insanitycc[2])
            PostProcessor:SetColourCubeData(2, _lunacycc[2], _lunacycc[2])
        end
    end
end
```

**关键点**：

- **直接覆盖 Channel 0 + Channel 1**——绕过所有渐变 / 季节 / 时段逻辑
- **传 nil 恢复**——回到正常 ambient
- **没有渐变**——瞬间切换（适合"道具开/关"的硬切）

#### 第二步：方式 2 —— `ccoverrides` 事件（带季节切换的覆盖）

**适用**：**mod 角色的"专属画风"**——比如 Wickerbottom 看书时**进入梦境模式**——但**昼夜还是要随着外界正常变化**。

```lua
-- 触发：传一个完整的 cctable
local DREAM_CCTABLE = {
    day = "images/colour_cubes/mymod_dream_day.tex",
    dusk = "images/colour_cubes/mymod_dream_dusk.tex",
    night = "images/colour_cubes/mymod_dream_night.tex",
    full_moon = "images/colour_cubes/mymod_dream_moon.tex",
}
player:PushEvent("ccoverrides", DREAM_CCTABLE)

-- 取消
player:PushEvent("ccoverrides", nil)
```

**源码**：

```281:284:scripts/components/colourcube.lua
local function OnOverrideCCTable(player, cctable)
    _overridecctable = cctable
    UpdateAmbientCCTable(DEFAULT_BLEND_TIME)
end
```

**关键点**：

- 传一个 **table**（包含 day / dusk / night / full_moon 4 个 cc 资源路径）
- ColourCube 系统**根据当前时段自动选**对应的 cc
- **保留昼夜切换的渐变**——只是把"原版季节 cc"换成"mod cc"

#### 第三步：方式 3 —— PlayerVision 组件（玩家专属，自动管理）

**适用**：**给某个玩家加专属画风** —— 戴 mole 帽、装备月眼镜——画面有不同效果。

```lua
local PlayerVision = inst:AddComponent("playervision")
inst.components.playervision:SetCustomCCTable(DREAM_CCTABLE)
-- 装备 / 取下时自动切换
```

源码 `OnPlayerActivated` 里：

```351:352:scripts/components/colourcube.lua
OnOverrideCCTable(player, player.components.playervision ~= nil and player.components.playervision:GetCCTable() or nil)
OnOverrideCCPhaseFn(player, player.components.playervision ~= nil and player.components.playervision:GetCCPhaseFn() or nil)
```

——`PlayerVision` 自动给 colourcube 系统提供 cctable / phasefn——**最规范的"玩家专属画风"实现**。

#### 第四步：3 种方式对比

| 方式 | 适用场景 | 是否带渐变 | 全局/玩家 |
| --- | --- | --- | --- |
| `overridecolourcube` | 道具瞬开瞬关 | ❌ 硬切 | 全局 |
| `ccoverrides` | mod 角色梦境模式 | ✓ 带渐变 | 玩家 |
| `PlayerVision:SetCustomCCTable` | 装备 / 视觉道具 | ✓ 带渐变 | 玩家 |

> **mod 推荐**：**临时画面用 `overridecolourcube`；持久画面用 `ccoverrides`；专属道具用 `PlayerVision`**。

#### 第五步：mod 实战代码——3 行红色滤镜

```lua
-- modmain.lua
Assets = {
    Asset("IMAGE", "images/colour_cubes/mymod_red_filter.tex"),
}

-- 触发（在某 mod 道具的 onuse 里）
TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/mymod_red_filter.tex")
inst:DoTaskInTime(3, function()
    TheWorld:PushEvent("overridecolourcube", nil)  -- 3 秒后恢复
end)
```

**结果**：玩家瞬间看到全屏变红 → 3 秒后渐变回正常——**3 行代码做到的视觉冲击**——比写一堆 fx 强 10 倍。

---

### 14.4.4 进阶：Blend 渐变源码——SetColourCubeData + SetColourCubeLerp

#### 第一步：Blend 函数核心逻辑

```150:206:scripts/components/colourcube.lua
local function Blend(time)
    local ambientcctarget = _ambientcctable[GetCCPhase()] or IDENTITY_COLOURCUBE
    local insanitycctarget = _insanitycctable[GetInsanityPhase()] or IDENTITY_COLOURCUBE
    local lunacycctarget = _lunacycctable[GetLunacyPhase()] or IDENTITY_COLOURCUBE

    if _overridecc ~= nil then
        -- override 时，只更新"目标值"，等 override 取消时直接接上
        _ambientcc[2] = ambientcctarget
        _insanitycc[2] = insanitycctarget
        _lunacycc[2] = lunacycctarget
        return
    end

    local newtarget = _ambientcc[2] ~= ambientcctarget or _insanitycc[2] ~= insanitycctarget or _lunacycc[2] ~= lunacycctarget

    if _remainingblendtime <= 0 then
        --No blends in progress, so we can start a new blend
        if newtarget then
            _ambientcc[1] = _ambientcc[2]              -- old → new src
            _ambientcc[2] = ambientcctarget            -- new dest
            _insanitycc[1] = _insanitycc[2]
            _insanitycc[2] = insanitycctarget
            _lunacycc[1] = _lunacycc[2]
            _lunacycc[2] = lunacycctarget
            _remainingblendtime = time
            _totalblendtime = time
            PostProcessor:SetColourCubeData(0, _ambientcc[1], _ambientcc[2])
            PostProcessor:SetColourCubeData(1, _insanitycc[1], _insanitycc[2])
            PostProcessor:SetColourCubeData(2, _lunacycc[1], _lunacycc[2])
            PostProcessor:SetColourCubeLerp(0, 0)
        end
    elseif newtarget then
        --Skip any blend in progress and restart new blend
        ...
    end
end
```

**核心 5 步**：

1. **算目标 cc**——根据当前 phase / season / moonphase 算出 ambient / insanity / lunacy 三个 target
2. **比较新旧 target**——决定要不要触发渐变
3. **存 src + dest**：`_ambientcc[1] = 上一个_ambientcc[2]`（之前的目标变成现在的起点），`_ambientcc[2] = ambientcctarget`（新目标）
4. **`SetColourCubeData(idx, src, dest)`**——告诉 GPU "从 src 渐变到 dest"
5. **`SetColourCubeLerp(idx, 0)`**——重置 lerp 到 0（起点）

#### 第二步：OnUpdate 里的 lerp 推进

```469:478:scripts/components/colourcube.lua
function self:OnUpdate(dt)
    if _overridecc == nil then
        if _remainingblendtime > dt and not ShouldSkipBlend() then
            _remainingblendtime = _remainingblendtime - dt
            PostProcessor:SetColourCubeLerp(0, 1 - _remainingblendtime / _totalblendtime)
        elseif _remainingblendtime > 0 then
            _remainingblendtime = 0
            PostProcessor:SetColourCubeLerp(0, 1)
        end
    end
    -- ...
end
```

**核心**：

- `_remainingblendtime` 从 `time` 倒数到 0
- `lerp = 1 - _remainingblendtime / _totalblendtime` —— 从 0 涨到 1
- **每帧推进**——一帧 1/60 秒——10 秒渐变 = 600 帧的连续 lerp 变化

#### 第三步：BLEND_TIME 常量

```71:80:scripts/components/colourcube.lua
local PHASE_BLEND_TIMES =
{
    day = 4,
    dusk = 6,
    night = 8,
    full_moon = 8,
}

local SEASON_BLEND_TIME = 10
local DEFAULT_BLEND_TIME = .25
```

**含义**：

- 切到白天 —— 4 秒渐变（最快——白天感觉应该来得快）
- 切到黄昏 —— 6 秒
- 切到黑夜 —— 8 秒（夜晚来得慢——更有沉浸感）
- 满月 —— 8 秒
- 季节切换 —— 10 秒（最慢——4 季中最重要的氛围切换）
- override / mod 默认 —— 0.25 秒（几乎瞬切——快速反馈）

> **不同事件用不同 blend time**——是**精心调过的玩家体验**——mod 改时务必注意。

#### 第四步：SetColourCubeData / SetColourCubeLerp 详解

源码 `postprocesseffects.lua`：

```24:35:scripts/postprocesseffects.lua
function PostProcessor__index:SetColourCubeData(index, src, dest)
    if index == 0 then
        self:SetTextureSampler(TexSamplers.CC0_SOURCE, src)
        self:SetTextureSampler(TexSamplers.CC0_DEST, dest)
    elseif index == 1 then
        self:SetTextureSampler(TexSamplers.CC1_SOURCE, src)
        self:SetTextureSampler(TexSamplers.CC1_DEST, dest)
    elseif index == 2 then
        self:SetTextureSampler(TexSamplers.CC2_SOURCE, src)
        self:SetTextureSampler(TexSamplers.CC2_DEST, dest)
    end
end
```

**含义**：每个通道**有 2 个 texture sampler**——CC0_SOURCE / CC0_DEST——shader 会**同时采样 src 和 dest**，用 lerp 系数混合。

```37:45:scripts/postprocesseffects.lua
function PostProcessor__index:SetColourCubeLerp(index, lerp)
    if index == 0 then
        self:SetUniformVariable(UniformVariables.CC_LERP_PARAMS, lerp, lerp, lerp)
    elseif index == 1 then
        self:SetUniformVariable(UniformVariables.CC_LAYER_PARAMS, lerp)
    elseif index == 2 then
        self:SetUniformVariable(UniformVariables.CC_LAYER_PARAMS, nil, lerp)
    end
end
```

**注意**：

- Channel 0 lerp 是 `(lerp, lerp, lerp)` —— RGB 各分量独立 lerp（理论上可以分别控制）
- Channel 1 / 2 lerp 是单值 —— 整体强度

#### 第五步：渐变时序图

```
T=0    seasontick → autumn → winter
       Blend(10) 被调
       _ambientcc[1] = day05_cc        (autumn 当前)
       _ambientcc[2] = snow_cc          (winter 目标)
       SetColourCubeData(0, day05_cc, snow_cc)
       SetColourCubeLerp(0, 0)
       _remainingblendtime = 10
       _totalblendtime = 10
       ↓
每帧 OnUpdate：
       _remainingblendtime -= dt
       lerp = 1 - _remainingblendtime / 10
       SetColourCubeLerp(0, lerp)
       ↓
T=10   _remainingblendtime = 0
       SetColourCubeLerp(0, 1)
       画面完全是 snow_cc 效果
```

**这就是季节渐变的完整生命周期**——src / dest / lerp 三参数协奏。

#### 第六步：mod 调试技巧——直接改 lerp 看效果

```lua
-- 控制台
PostProcessor:SetColourCubeData(0, "images/colour_cubes/snow_cc.tex", "images/colour_cubes/snow_cc.tex")
PostProcessor:SetColourCubeLerp(0, 1)
```

——**绕过整个 colourcube.lua 系统**——直接和 GPU 沟通——**调试 cc 资源的最快方法**。

> **mod 调画面时——先用 console 看效果——再写代码**。

---

### 14.4.5 进阶：事件驱动的 cc 切换 5 大事件

#### 第一步：5 大触发事件

```451:461:scripts/components/colourcube.lua
inst:ListenForEvent("playeractivated", OnPlayerActivated)
inst:ListenForEvent("playerdeactivated", OnPlayerDeactivated)
inst:ListenForEvent("phasechanged", OnPhaseChanged)
inst:ListenForEvent("moonphasechanged2", OnMoonPhaseChanged2)
inst:ListenForEvent("moonphasestylechanged", OnMoonPhaseStyleChanged)

if not _iscave then
    inst:ListenForEvent("seasontick", OnSeasonTick)
end
inst:ListenForEvent("overridecolourcube", OnOverrideColourCube)
inst:ListenForEvent("overridecolourmodifier", OnOverrideColourModifier)
```

**完整触发列表**：

| 事件 | 来源 | 效果 |
| --- | --- | --- |
| `phasechanged` | 时段切换（day/dusk/night） | Blend(PHASE_BLEND_TIMES) |
| `seasontick` | 季节变化 | UpdateAmbientCCTable + Blend(10) |
| `moonphasechanged2` | 月相切换（满月） | Blend(PHASE_BLEND_TIMES) |
| `moonphasestylechanged` | 觉醒月相 alter_active | Blend |
| `sanitydelta` | 玩家精神值变化 | OnSanityDelta（调 insanity / lunacy lerp） |
| `stormlevel` | 月暴等级变化 | Blend |
| `enterraindome` / `exitraindome` | 进/出雨穹 | 鱼眼镜头开关 |
| `overridecolourcube` | mod / 道具临时覆盖 | OnOverrideColourCube |
| `ccoverrides` | mod / vision 覆盖 cctable | OnOverrideCCTable |
| `ccphasefn` | mod / vision 覆盖 phase 函数 | OnOverrideCCPhaseFn |
| `playeractivated` / `playerdeactivated` | 玩家进入 / 退出 | 监听玩家专属事件 |

#### 第二步：phasechanged 处理

```370:379:scripts/components/colourcube.lua
local function OnPhaseChanged(inst, phase)
    if _phase ~= phase then
        _phase = phase

        local blendtime = PHASE_BLEND_TIMES[GetCCPhase()]
        if blendtime ~= nil then
            Blend(blendtime)
        end
    end
end
```

**简单 4 步**：phase 变 → 存新 phase → 取对应 blend time → Blend。

**示例**：玩家 09:00（dusk）→ 21:00（night）：

```
phase: dusk → night
Blend(8) 调用
ambient: dusk03_cc → night03_cc 8 秒渐变
insanity: insane_dusk_cc → insane_night_cc 8 秒渐变
```

#### 第三步：seasontick 处理

```407:410:scripts/components/colourcube.lua
local OnSeasonTick = not _iscave and function(inst, data)
    _season = data.season
    UpdateAmbientCCTable(SEASON_BLEND_TIME)
end or nil
```

**关键**：**洞穴里没有季节**——所以 OnSeasonTick **只在地表注册**——洞穴永远是 CAVE_COLOURCUBES。

#### 第四步：sanitydelta 处理（最复杂）

```252:279:scripts/components/colourcube.lua
local function OnSanityDelta(player, data)
    local is_lunacy = player.replica.sanity:IsLunacyMode()
	local sanity_percent = player.replica.sanity:GetPercent()
    local lunacy_percent = 1 - sanity_percent

	local lunacy_distortion = 1 - easing.outQuad(lunacy_percent, 0, 1, 1)
	local sanity_distortion = 1 - easing.outQuad(sanity_percent, 0, 1, 1)
	if player ~= nil and player:HasTag("dappereffects") then
		lunacy_distortion = lunacy_distortion * lunacy_distortion
		sanity_distortion = sanity_distortion * sanity_distortion
	end

    if is_lunacy then
        _lunacyspeed = easing.outQuad(1 - lunacy_percent, 0.4, 1.6, 1)
    else
        _lunacyspeed = easing.outQuad(1 - sanity_percent, -0.4, -1.6, 1)
    end
    -- ...
```

**复杂在哪**：

1. **要区分 sanity / lunacy 两种"疯狂模式"**
2. **用 outQuad 缓动**——避免线性变化的"假"
3. **dappereffects tag 加倍效果**——绅士玩家疯狂效果更强
4. **3 种状态各自处理**：完全 sanity / 完全 lunacy / 过渡中

**结果**：玩家精神值掉到 50%——画面**逐渐变暗变红 + 出现 distortion 扭曲 + 视野中央有 zoom blur**。

#### 第五步：moonphasestylechanged —— 觉醒月相

```393:405:scripts/components/colourcube.lua
local function OnMoonPhaseStyleChanged(inst, data)
    local alter_awake = data.style == "alter_active"
	if _alter_awake ~= alter_awake then
		_alter_awake = alter_awake

		if _fullmoonphase  then
			local blendtime = PHASE_BLEND_TIMES[GetCCPhase()]
			if blendtime ~= nil then
				Blend(blendtime)
			end
		end
    end
end
```

**含义**：觉醒（alter_active）月相变化时——`Blend` 重新算 cc——配合 `ccphasefn` 选择特殊 cc。

#### 第六步：mod 监听 phasechanged 做小事

```lua
TheWorld:ListenForEvent("phasechanged", function(world, phase)
    if phase == "night" then
        TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/mymod_night_horror.tex")
    elseif phase == "day" then
        TheWorld:PushEvent("overridecolourcube", nil)
    end
end)
```

**效果**：mod 自定义"夜晚惊悚滤镜"——只在夜晚触发——白天恢复正常。

> **不修改 colourcube.lua、只监听事件**——这是 mod 接入 cc 系统的**最干净路径**。

---

### 14.4.6 进阶：mod 自定义 cctable + ccphasefn 完整方案

#### 第一步：场景——为什么需要"自定义 phase fn"

`ccoverrides` 让你换 cc 资源——但**phase 触发逻辑是写死的**（day/dusk/night/full_moon）。

**问题**：mod 角色"科学家薇克斯顿"使用专属望远镜——按下时**进入"研究模式"**——但**研究模式有自己的 phase 概念**（idle / focus / done）——不应该映射到游戏的昼夜——这时需要 **`ccphasefn`**。

#### 第二步：ccphasefn 注册

```286:308:scripts/components/colourcube.lua
local function OnOverrideCCPhaseFn(player, fn)
    local blendtime = nil
    if _overridephase ~= nil then
        if _overridephase.blendtime ~= nil then
            blendtime = _overridephase.blendtime
        end
        for i,event in ipairs(_overridephase.events) do
            inst:RemoveEventCallback(event, OnOverridePhaseEvent)
        end
    end
    _overridephase = fn
    if _overridephase ~= nil then
        if _overridephase.blendtime ~= nil then
            blendtime = blendtime ~= nil and math.min(blendtime, _overridephase.blendtime) or _overridephase.blendtime
        end
        for i,event in ipairs(_overridephase.events) do
            inst:ListenForEvent(event, OnOverridePhaseEvent)
        end
    end
    UpdateAmbientCCTable(blendtime or DEFAULT_BLEND_TIME)
end
```

**`fn` 是个 table**——不是普通函数：

```lua
local MY_PHASE_FN = {
    fn = function()
        -- 返回当前 phase 名（必须是 _ambientcctable 里的 key）
        if some_condition then
            return "focus"
        else
            return "idle"
        end
    end,
    events = { "research_mode_changed" },  -- 触发重算的事件名
    blendtime = 0.5,                        -- 渐变时长
}

-- 注册
player:PushEvent("ccphasefn", MY_PHASE_FN)
```

#### 第三步：完整 mod 案例——「研究模式」cctable

```lua
-- modmain.lua
Assets = {
    Asset("IMAGE", "images/colour_cubes/research_idle_cc.tex"),
    Asset("IMAGE", "images/colour_cubes/research_focus_cc.tex"),
    Asset("IMAGE", "images/colour_cubes/research_done_cc.tex"),
}
```

```lua
-- mod prefab fn 里
local RESEARCH_CCTABLE = {
    idle = "images/colour_cubes/research_idle_cc.tex",
    focus = "images/colour_cubes/research_focus_cc.tex",
    done = "images/colour_cubes/research_done_cc.tex",
}

local RESEARCH_PHASE_FN = {
    fn = function()
        local mode = ThePlayer.research_mode
        if mode == "focus" then return "focus"
        elseif mode == "done" then return "done"
        else return "idle"
        end
    end,
    events = { "research_mode_changed" },
    blendtime = 1.0,
}

-- 玩家进入研究模式时
player:PushEvent("ccoverrides", RESEARCH_CCTABLE)
player:PushEvent("ccphasefn", RESEARCH_PHASE_FN)

-- 切换 mode
player.research_mode = "focus"
player:PushEvent("research_mode_changed")  -- 触发 cc 重算

-- 离开研究模式
player:PushEvent("ccoverrides", nil)
player:PushEvent("ccphasefn", nil)
```

**效果**：

- 进入研究模式 → 画面切到 idle cc（蓝调静谧）
- 切到 focus → 画面变为 focus cc（橙红聚焦）
- 切到 done → 画面切到 done cc（绿色满足）
- **完全脱离游戏的昼夜逻辑**——纯由 mod 状态驱动

#### 第四步：制作 mod cc 资源——4 步流程

**第 1 步：复制 identity_colourcube.png 作底**

游戏 `images/colour_cubes/` 文件夹有 identity_colourcube.png——是一张 1024×32 的 LUT 模板。

**第 2 步：用 Photoshop / Affinity 调色**

```
Photoshop 操作：
1. 打开 identity_colourcube.png
2. 图层 > 调整图层 > 色彩平衡 / 色阶 / 曲线
3. 调到你想要的画面色调
4. 拼合所有图层
5. 导出为 .png
```

**第 3 步：转 .tex（饥荒专用格式）**

游戏自带工具 `mods/tools/scripts/krane.exe` 或 mod 工具链（具体看 mod 教程主章 12）——把 .png 转成 .tex。

**第 4 步：放到 mod 的 `images/colour_cubes/`**

**修改 modinfo.lua**：

```lua
all_clients_require_mod = false  -- cc 是纯客户端资源
client_only_mod = true           -- 推荐
```

> **cc 资源是纯客户端 / 视觉资源——服务器不需要——所以可以 client_only_mod**。

#### 第五步：3 种 mod 模式总结

| 模式 | 难度 | 触发 | 资源 |
| --- | --- | --- | --- |
| 单 cc 临时 override | ★ | overridecolourcube 事件 | 1 张 cc |
| 替换 cctable（带昼夜） | ★★ | ccoverrides 事件 | 4 张 cc（day/dusk/night/full_moon） |
| 自定义 phase fn（脱离昼夜） | ★★★ | ccphasefn 事件 + cctable | N 张 cc + 自定义 phase 字符串 |

---

### 14.4.7 老手进阶：PostProcessor 全家桶——4 大后处理 shader

#### 第一步：PostProcessor 全家桶

ColourCube 只是 PostProcessor 的"一员"——还有 **4 大 shader**：

```275:297:scripts/postprocesseffects.lua
function SortAndEnableShaders()
    PostProcessor:SetBasePostProcessEffect(PostProcessorEffects.ColourCube)
    PostProcessor:SetPostProcessEffectBefore(PostProcessorEffects.Distort, PostProcessorEffects.ColourCube)
    PostProcessor:SetPostProcessEffectBefore(PostProcessorEffects.Bloom, PostProcessorEffects.Distort)
    PostProcessor:SetPostProcessEffectBefore(PostProcessorEffects.ZoomBlur, PostProcessorEffects.Bloom)
    PostProcessor:SetPostProcessEffectAfter(PostProcessorEffects.Lunacy, PostProcessorEffects.ColourCube)
    PostProcessor:SetPostProcessEffectAfter(PostProcessorEffects.MoonPulse, PostProcessorEffects.Lunacy)
    PostProcessor:SetPostProcessEffectAfter(PostProcessorEffects.MoonPulseGrading, PostProcessorEffects.MoonPulse)

    PostProcessor:EnablePostProcessEffect(PostProcessorEffects.ColourCube, true)
    --[[
    CurrentOrder:
    ZoomBlur
    Bloom
    Distort
    ColourCube --Base Effect
    Lunacy
    MoonPulse
    MoonPulseGrading
    --]]
end
```

**完整渲染顺序（自上而下）**：

```
1. ZoomBlur       —— 距离模糊（疯狂时眩晕感）
2. Bloom          —— 光晕扩散（粒子发光）
3. Distort        —— 鱼眼扭曲（雨穹 / 疯狂）
4. ColourCube     —— LUT 调色（基础）
5. Lunacy         —— 月狂叠加 overlay
6. MoonPulse      —— 月相脉冲（觉醒月）
7. MoonPulseGrading —— 月相分级
```

**每个 shader 都可以独立启用 / 禁用**。

#### 第二步：4 大 shader 详解

**Bloom（光晕）**

```lua
PostProcessor:SetBloomEnabled(true)
```

含义：**画面亮的地方发散光晕** —— 火堆、月亮、闪电、Bloom 字段为 true 的粒子——视觉上"梦幻"。

**Distort（扭曲）**

```lua
PostProcessor:SetDistortionEnabled(true)
PostProcessor:SetDistortionFactor(0.5)         -- 0~1
PostProcessor:SetDistortionRadii(0.5, 0.685)   -- 内圆、外圆
PostProcessor:SetDistortionFishEyeIntensity(-0.01)
```

含义：**画面扭曲（鱼眼镜效果）**——疯狂时、雨穹中、月狂时使用。

**Lunacy（月狂叠加）**

```lua
PostProcessor:SetLunacyEnabled(true)
PostProcessor:SetLunacyIntensity(0.7)
PostProcessor:SetOverlayBlend(0.3)
PostProcessor:SetOverlayTex("images/overlays_lunacy.tex")
```

含义：**叠加一张 overlay 贴图**——画面有"月光透出来的网纹"——月狂角色专属。

**MoonPulse / MoonPulseGrading（月脉冲）**

```lua
PostProcessor:SetMoonPulseParams(p1, p2, p3, p4)
PostProcessor:SetMoonPulseGradingParams(p1, p2, p3, p4)
```

含义：**觉醒月相专属**——画面**周期性脉冲**——配合 boss 战使用。

#### 第三步：PostProcessor API 全家桶速查

```lua
-- ColourCube
PostProcessor:SetColourCubeData(idx, src, dest)
PostProcessor:SetColourCubeLerp(idx, lerp)
PostProcessor:SetColourModifier(modifier)

-- Bloom
PostProcessor:SetBloomEnabled(enabled)
PostProcessor:IsBloomEnabled()

-- Distort
PostProcessor:SetDistortionEnabled(enabled)
PostProcessor:SetDistortionFactor(factor)
PostProcessor:SetDistortionRadii(inner, outer)
PostProcessor:SetDistortionFishEyeIntensity(intensity)
PostProcessor:SetDistortionFishEyeTime(time)
PostProcessor:SetDistortionEffectTime(time)
PostProcessor:SetDistortionFishLensRadius(r)
PostProcessor:SetDistortionFishLensAspectRatio(aspect_ratio)

-- Lunacy
PostProcessor:SetLunacyEnabled(enabled)
PostProcessor:SetLunacyIntensity(intensity)
PostProcessor:SetOverlayTex(tex)
PostProcessor:SetOverlayBlend(blend)

-- MoonPulse
PostProcessor:SetMoonPulseParams(p1, p2, p3, p4)
PostProcessor:SetMoonPulseGradingParams(p1, p2, p3, p4)

-- ZoomBlur
PostProcessor:SetZoomBlurEnabled(enabled)
```

#### 第四步：mod 用 PostProcessor 直接操作的场景

**场景 1：Boss 战时强制开 Distort**

```lua
inst:ListenForEvent("startboss", function()
    PostProcessor:SetDistortionEnabled(true)
    PostProcessor:SetDistortionFactor(0.6)
end)
inst:ListenForEvent("endboss", function()
    PostProcessor:SetDistortionEnabled(false)
end)
```

**场景 2：mod 自定义 overlay**

```lua
PostProcessor:SetOverlayTex("images/mymod_blood_overlay.tex")
PostProcessor:SetOverlayBlend(0.2)
```

> **mod 改 PostProcessor 应当克制**——这是**全屏效果**——影响所有玩家——一不小心就破坏游戏体验。**永远在退出条件下重置**。

#### 第五步：AddModShadersInit / AddModShadersSortAndEnable

mod 还可以**注册自己的 shader**：

```lua
AddModShadersInit(function()
    -- 注册新 shader uniform
    UniformVariables.MYMOD_PARAMS = PostProcessor:AddUniformVariable("MYMOD_PARAMS", 4)

    PostProcessorEffects.MyMod = PostProcessor:AddPostProcessEffect("shaders/mymod_effect.ksh")
    PostProcessor:SetEffectUniformVariables(PostProcessorEffects.MyMod, UniformVariables.MYMOD_PARAMS)
end)

AddModShadersSortAndEnable(function()
    PostProcessor:SetPostProcessEffectAfter(PostProcessorEffects.MyMod, PostProcessorEffects.MoonPulseGrading)
    PostProcessor:EnablePostProcessEffect(PostProcessorEffects.MyMod, true)
end)
```

**含义**：**mod 加自己的 GPU shader**——和官方 7 个 shader 共同组成 PostProcessor pipeline——**最高阶的 mod 视觉**。

> **这是高阶玩法**——需要懂 GLSL / KSH shader 语法——不是 90% mod 作者的需求——**先掌握 ColourCube 即可**。

---

### 14.4.8 老手进阶：六个常见陷阱

#### 陷阱 1：override 不取消——画面卡住

**症状**：mod 道具使用完——画面**永远停在 override cc**——玩家以为游戏 bug。

**原因**：

```lua
-- ❌
TheWorld:PushEvent("overridecolourcube", "mymod_red.tex")
-- 结束时漏 PushEvent(nil)
```

**修复**：**任何 override 必须配对取消**：

```lua
TheWorld:PushEvent("overridecolourcube", "mymod_red.tex")
inst:DoTaskInTime(3, function()
    if TheWorld then
        TheWorld:PushEvent("overridecolourcube", nil)
    end
end)
```

> **像 SG state 的 onenter/onexit 一样——override 必须有"出口"**。

---

#### 陷阱 2：cc 资源没注册到 Assets

**症状**：mod 触发 override —— 画面没变 —— 控制台报"missing texture"。

**原因**：

```lua
-- modinfo.lua / modmain.lua 漏写
-- Assets = {
--     Asset("IMAGE", "images/colour_cubes/mymod_red.tex"),  -- ★ 漏写
-- }
```

**修复**：**所有 mod cc 资源必须在 Assets 注册**：

```lua
Assets = {
    Asset("IMAGE", "images/colour_cubes/mymod_red.tex"),
}
```

---

#### 陷阱 3：cc 资源不是 32×32×32 LUT 格式

**症状**：mod cc 加载成功——但**画面变成乱七八糟的颜色拼贴**——好像 GPU 在"读错图"。

**原因**：自制 cc 资源是普通 PNG/TEX——**不是 1024×32 的 LUT 排布**——shader 按 LUT 索引采样 → 完全错误。

**修复**：**永远用 identity_colourcube.png 作底**——保证 32×32×32 LUT 数据排布——只调色不改尺寸。

> **mod 论坛里至少 50% 的"画面乱"问题**都是这个——**LUT 格式是不可破的**。

---

#### 陷阱 4：在 dedicated 调 PostProcessor

**症状**：mod 改 PostProcessor 参数——dedicated 服务器报错 "attempt to call ... (a nil value)"。

**原因**：dedicated 没渲染——`PostProcessor` 大部分函数在 dedicated 上是**空函数 / nil**。

**修复**：**所有 PostProcessor 调用包 `if not TheNet:IsDedicated()`**：

```lua
if not TheNet:IsDedicated() then
    PostProcessor:SetDistortionEnabled(true)
end
```

> 14.3 学过的同款规则——**任何视觉代码包 IsDedicated**。

---

#### 陷阱 5：blend time 太短或太长

**症状**：

- 太短（< 0.1s）—— 玩家感觉"画面闪了一下"——突兀
- 太长（> 30s）—— 玩家**意识不到画面变了**——失去氛围效果

**原因**：mod 自定义 ccoverrides 时——blend time 没参考官方常量。

**修复**：**参考 PHASE_BLEND_TIMES 常量**：

```lua
-- 推荐 mod blendtime
道具开/关：0.5 ~ 1 秒
角色专属画风：4 ~ 8 秒
季节级别变化：10 ~ 15 秒
```

---

#### 陷阱 6：mod 改 ambient cc 但不还原

**症状**：mod 角色在某 state 时——`ccoverrides` 设了自定义 cctable——**state 退出时没 reset** —— 玩家**永远停在 mod 画风**——其他 mod 也无法 override。

**原因**：

```lua
onenter = function(inst)
    inst:PushEvent("ccoverrides", MY_CCTABLE)
end,

-- ❌ 漏写 onexit
```

**修复**：**state 退出 / 道具卸下 / 玩家死亡**——**任何"退出条件"都要 reset**：

```lua
onexit = function(inst)
    inst:PushEvent("ccoverrides", nil)
end,
```

> **mod 视觉系统的"out of game state"必修课**——和 SG onexit 同款铁律。

---

#### 设计经验三条

**经验 ①：override 必须配对取消**

> 任何 PushEvent("overridecolourcube", X) 都必须有对应的 PushEvent(nil)——像 SG onenter / onexit。

**经验 ②：mod cc 资源用 identity 作底**

> 永远从 identity_colourcube.png 出发调色——保证 LUT 格式——不会变成"画面乱码"。

**经验 ③：blendtime 参考官方常量**

> day=4 / dusk=6 / night=8 / season=10 / default=0.25——mod 取相近值——保持"游戏内自然感"。

---

### 14.4.9 小结

#### 一句话总结

**ColourCube 是"32×32×32 GPU LUT 全屏调色"系统——3 通道（ambient/insanity/lunacy）叠加 + Blend(src→dest, lerp=0~1) 渐变 + 5 大事件驱动（phasechanged/seasontick/sanitydelta/overridecolourcube/ccoverrides）+ 3 种 mod 修改路径——配合 PostProcessor 的 Bloom/Distort/Lunacy/MoonPulse 4 大后处理 shader——是游戏所有"全屏氛围"的统一底座**。

#### 速查 7 表

**3 通道职责**

| 通道 | idx | 触发 |
| --- | --- | --- |
| Ambient | 0 | 季节 / 时段 / 洞穴 / override |
| Insanity | 1 | 精神值 |
| Lunacy | 2 | 月狂值 |

**Blend 三参数**

| 参数 | 作用 |
| --- | --- |
| `SetColourCubeData(idx, src, dest)` | 起点 / 终点 cc 资源 |
| `SetColourCubeLerp(idx, t)` | 0~1 渐变进度 |
| `_remainingblendtime` | 倒计时器 |

**官方 BLEND_TIME 常量**

| 事件 | 秒 |
| --- | --- |
| 切日 | 4 |
| 切黄昏 | 6 |
| 切夜 / 满月 | 8 |
| 切季节 | 10 |
| override 默认 | 0.25 |

**mod 修改 3 种路径**

| 路径 | 适用 |
| --- | --- |
| `overridecolourcube` 事件 | 临时全屏 |
| `ccoverrides` 事件 | 自定义带昼夜 cctable |
| `ccphasefn` 事件 | 自定义 phase fn |

**PostProcessor 7 大 shader 顺序**

| 顺序 | shader |
| --- | --- |
| 1 | ZoomBlur |
| 2 | Bloom |
| 3 | Distort |
| 4 | ColourCube（base） |
| 5 | Lunacy |
| 6 | MoonPulse |
| 7 | MoonPulseGrading |

**mod cc 资源 4 步制作**

1. 复制 identity_colourcube.png
2. Photoshop 调色
3. 转 .tex 格式（krane.exe）
4. 注册到 Assets

**6 个陷阱排雷顺序**

1. override 不取消 → 画面卡住 → 必配对 nil
2. cc 资源没在 Assets → 找不到贴图
3. 自制 cc 不是 LUT 格式 → 用 identity 作底
4. dedicated 调 PostProcessor → 必加 IsDedicated 保护
5. blendtime 太短/太长 → 参考官方常量
6. mod state 不重置 cc → 配对 onexit 清理

#### 3 条设计经验

- **① override 必须配对取消**——和 SG onenter/onexit 同款铁律
- **② cc 资源永远从 identity 出发**——保证 LUT 格式
- **③ blendtime 参考官方常量**——day=4 / season=10 / default=0.25

#### mod 自定义画风 4 行起步

```lua
-- modmain.lua
Assets = { Asset("IMAGE", "images/colour_cubes/mymod_horror.tex") }

-- 进入恐怖模式
TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/mymod_horror.tex")

-- 退出
TheWorld:PushEvent("overridecolourcube", nil)
```

> **下一节预告**：14.5 节我们将进入 **GroundCreep / GroundTiles —— 地面特效覆盖**——前面 14.1 / 14.2 / 14.3 / 14.4 都是"屏幕内 / 镜头内"的视觉——而 GroundCreep 是**附着在地面上的视觉层**——蜘蛛网、毒气区、雨水痕、月光草地——是**地图层"持久视觉"**的实现方式——14.5 会把它的 prefab 模式、tile 切换、mod 自定义草地一次讲透。

---


## 14.5 GroundCreep / GroundTiles——地面特效覆盖

### 本节导读

> **一句话定位**：GroundCreep 是**贴在地面上的、可读可写的"污染层"**——它既是**视觉层**（蜘蛛网/毒雾区的画面），又是**逻辑层**（locomotor 检测、生物 AI 触发、存档同步）。

前面 14.1 ~ 14.4 全是"镜头内"的视觉：FX prefab、粒子、ColourCube——它们都**不属于地图本身**，玩家走过去也不会"踩到"它们，存档时也不会"留下痕迹"。

而 14.5 节要讲的是**完全相反**的一类系统：

| 维度 | FX 系统（14.1-14.3） | GroundCreep 系统（14.5） |
|---|---|---|
| 渲染层 | 镜头内 entity / VFX 粒子 | 地图渲染层（MapLayerManager） |
| 生命周期 | 短时（animover / DoTaskInTime） | 长时（与地图共存，持久化） |
| 存档 | 不存档（persists=false） | **整个 creep 状态写入存档**（GetAsString） |
| 玩家交互 | 不能交互（纯视觉） | **可踩（locomotor）+ 可触发 AI 事件** |
| 谁决定形状 | prefab 内 SpawnPrefab + 自身位置 | **emitter 半径**（GroundCreepEntity:SetRadius）|
| 数据结构 | 散点 entity | 网格化 tile（按 chunk 存储） |

**这意味着**：

- 当蜘蛛巢长大→ 蛛网会**沿着地面铺开**（14.5.4 spiderden）；当蜘蛛巢被烧毁→ 蛛网会**慢慢消失**（14.5.4 SetRadius(0)）。
- 玩家走在蛛网上→ 自动减速 + 播放"沙沙"脚步声（14.5.3 locomotor）。
- 蜘蛛走在蛛网上→ 不触发蜘蛛巢（triggerscreep=false，14.5.3）。
- 玩家走过蛛网激活附近的蜘蛛巢→ 蜘蛛巢派出 investigator 蜘蛛（14.5.5 creepactivate 事件链）。
- 鸟群刷新点检测到 creep→ 不在蜘蛛网上刷新（14.5.5 birdspawner 联动）。
- 退出游戏后再加载→ 蛛网形状**完整恢复**（14.5.7 OnSave/OnLoad）。

而 GroundTiles（地面贴图）是它的"邻居系统"：

- **GroundTiles**：地图的**主层**——草地、岩石、洞穴、海洋……由 `tiledefs.lua` 注册，由 `worldgen` 在世界生成时铺开。
- **GroundCreep**：地图的**覆盖层**——蛛网、月光草、毒气区……由 `groundcreepdefs.lua` 注册，由**运行时 entity** 动态铺开 / 收缩。

它们都通过 **`TileManager`**（`scripts/tilemanager.lua`）注册，都用 **`MapLayerManager`** 创建渲染层（`scripts/prefabs/world.lua`）。但是用法、生命周期、API 完全不同。

**本节学习路径**：

```
14.5.1 新手  ──────  GroundCreep 是什么？为什么不是 FX？
14.5.2 新手  ──────  4 行让你的怪物拥有"领地"（spider_web_spit_creep 模板）
14.5.3 新手  ──────  locomotor 联动：triggerscreep / fasteroncreep / slowmultiplier
                       ↓
14.5.4 进阶  ──────  官方 5 大 GroundCreep prefab 对照表
14.5.5 进阶  ──────  creepactivate / walkoncreep / walkoffcreep 三大事件链
14.5.6 进阶  ──────  mod 添加自定义草地（AddTile + AddGroundCreep + AddFalloffTexture）
                       ↓
14.5.7 老手  ──────  底层架构：3 个 GroundCreep entity 角色
14.5.8 老手  ──────  6 大常见陷阱 + 3 条设计经验
14.5.9 小结  ──────  速查表 + 4 行起步代码 + 14.6 预告
```

**给三类读者的承诺**：
- **新手**：你将得到 1 个最小 prefab 模板（5 行）+ 3 个 locomotor 开关，让你的怪物**拥有持久化的"地盘"**。
- **进阶**：你将完整理解 `creepactivate` / `walkoncreep` 事件链是怎么从**玩家脚下**一路传到**蜘蛛巢内部**的——以及如何用它做"瘟疫感染区域"、"野怪刷新带"。
- **老手**：你将看到 `World.GroundCreep` / `inst.GroundCreepEntity` / `components/groundcreep.lua` 这**三个名字相似实则各司其职**的 entity 角色——以及 mod 注册自定义 tile / creep 的完整 4 步流程。

---

### 14.5.1（新手）GroundCreep 是什么——一张地图上的"动态污染层"

#### 用一个直觉对比开始

打开饥荒，走到任意一个**蜘蛛巢**附近——你会看到：

- **草地不一样了**：蜘蛛巢周围一圈**白色的蛛网纹理**铺在地上，**和原来的草贴图叠加**。
- **范围不固定**：小蜘蛛巢一圈很小，大蜘蛛巢一圈很大——而且**蛛网的形状不是规则圆**，而是有**自然的边缘噪声**。
- **持久化**：你退出存档再加载——**蛛网还在**。
- **可读**：你走在上面会减速；蜘蛛走在上面则没事。

**这一切都是 GroundCreep 系统**。它**不是 prefab**——你**找不到**一个叫 "spiderweb_visual" 的 entity 站在那里——蛛网完全是**地图本身的一层贴图**。

#### 它"不是 FX"——4 个本质区别

| 你以为的 FX | GroundCreep 真相 |
|---|---|
| "应该是个 entity 站在地上播放贴图" | 不是 entity，是**地图渲染层**（MapLayerManager 的 RenderLayer） |
| "可以 prefab 化用 SpawnPrefab" | 没有 prefab，**只能用 entity 携带 GroundCreepEntity emitter** |
| "应该有动画 / 动态效果" | **完全静态贴图**（噪声纹理 + 边缘 falloff），没有动画 |
| "退出游戏就消失" | **整张地图的 creep 状态会写入存档**，下次开档完整恢复 |

#### 一张图看懂——3 层架构

```
┌─────────────────────────────────────────────────────────────────┐
│  ▎ Lua 数据层 ▎                                                  │
│  scripts/groundcreepdefs.lua                                   │
│      └──► TileManager.AddGroundCreep(WEBCREEP, {              │
│             name = "web", noise_texture = "web_noise" })      │
│                          │                                     │
│                          │ 注册到 GroundTiles.creep 表        │
│                          ▼                                     │
└─────────────────────────────────────────────────────────────────┘
                             │
                             │ 世界初始化时
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  ▎ Entity 渲染层 ▎                                               │
│  scripts/prefabs/world.lua                                      │
│      ├── inst.entity:AddGroundCreep()                          │
│      └── 为每个 creep def 创建 RenderLayer 并 AddRenderLayer    │
│                          │                                     │
│                          ▼                                     │
│  TheWorld.GroundCreep（World 上的 entity component，全局唯一） │
│      ├── :OnCreep(x, y, z) → bool（检测某点是否在 creep 上）   │
│      ├── :GetTriggeredCreepSpawners(x, y, z) → list           │
│      ├── :GetAsString() / :SetFromString(s) → 存档            │
│      └── :FastForward() → 玩家激活时一次性算完渲染            │
└─────────────────────────────────────────────────────────────────┘
                             ▲
                             │ "我要在这一圈铺 creep"
                             │
┌─────────────────────────────────────────────────────────────────┐
│  ▎ Entity 发射层 ▎                                               │
│  任意产生 creep 的 prefab（spiderden / dropperweb …）          │
│      ├── inst.entity:AddGroundCreepEntity()                    │
│      └── inst.GroundCreepEntity:SetRadius(5)                   │
│                                                                 │
│  ↑ 一个个体一个 emitter，用半径控制铺设范围                     │
└─────────────────────────────────────────────────────────────────┘
                             │
                             │ 联动
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  ▎ Lua 联动层 ▎                                                  │
│  • locomotor:UpdateGroundSpeedMultiplier()                     │
│      └── 调 TheWorld.GroundCreep:OnCreep(...)                  │
│      └── 推 walkoncreep / walkoffcreep / creepactivate 事件    │
│  • worldtiledefs（脚步声）                                      │
│      └── 在 creep 上播 dontstarve/movement/run_web              │
│  • birdspawner（生物 AI）                                       │
│      └── 不在 creep 上刷新鸟                                    │
│  • components/groundcreep.lua                                  │
│      └── OnSave / OnLoad / playeractivated → FastForward       │
└─────────────────────────────────────────────────────────────────┘
```

#### 看一眼最小注册代码

打开 `scripts/groundcreepdefs.lua`——你会震惊于它的简短：

```1:9:scripts/groundcreepdefs.lua
local TileManager = require("tilemanager")

TileManager.AddGroundCreep(
    GROUND_CREEP_IDS.WEBCREEP,
    {
        name = "web",
        noise_texture = "web_noise",
    }
)
```

**整个游戏的"蛛网"系统**——只有 9 行。其中：
- `GROUND_CREEP_IDS.WEBCREEP` 是 `scripts/constants.lua` 779 行定义的常量（值=1）：
  ```779:781:scripts/constants.lua
  GROUND_CREEP_IDS = {
      WEBCREEP = 1,
  }
  ```
- `name = "web"` → 真正的贴图路径会被 `tilemanager.lua` 解析成 `levels/textures/web.tex`
- `noise_texture = "web_noise"` → 边缘噪声 → `levels/textures/noise/web_noise.tex`

为什么这么简短？因为**真正的渲染、网格化存储、tile 检测、跨 chunk 同步**——**全都在 C++ 引擎里**。Lua 这一层只负责**告诉引擎**："请加一个名为 WEBCREEP 的渲染层，用这两张贴图。"

#### 它和 GroundTiles 的关系

| 系统 | 何时铺 | 在哪铺 | 永久 / 临时 | 谁铺 | 注册函数 |
|---|---|---|---|---|---|
| **GroundTiles**（草地、海洋） | 世界生成时 | 整个地图 | 永久 | worldgen | `TileManager.AddTile()` |
| **GroundCreep**（蛛网） | **运行时**，由 emitter 决定 | emitter 半径内 | **可回退**（SetRadius(0)） | 任意 entity | `TileManager.AddGroundCreep()` |

它们**共享同一套贴图渲染管线**（`MapLayerManager:CreateRenderLayer`），但渲染层目标不同：
- GroundTiles → `inst.Map:AddRenderLayer(handle)`（地图层）
- GroundCreep → `inst.GroundCreep:AddRenderLayer(handle)`（覆盖层，叠在地图之上）

#### 新手只要记住三件事

1. **GroundCreep 是地图的覆盖层**——不是 entity，不是 FX，不能 SpawnPrefab。
2. **谁产生 creep？** 任何调用了 `inst.entity:AddGroundCreepEntity()` + `inst.GroundCreepEntity:SetRadius(r)` 的 entity。半径 = 0 等于停止生产。
3. **它会持久化**——不要用它做"短时特效"。短时用 14.1-14.3 的 FX，长时持久才用 GroundCreep。

---

### 14.5.2（新手）4 行让你的怪物拥有"领地"——spider_web_spit_creep 模板

#### 全游戏最简 GroundCreep prefab

打开 `scripts/prefabs/spider_web_spit_creep.lua`——只有 21 行：

```1:21:scripts/prefabs/spider_web_spit_creep.lua
local function fn()
	local inst = CreateEntity()

	inst.entity:AddTransform()
	inst.entity:AddGroundCreepEntity()
    inst.entity:AddNetwork()

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

	inst.GroundCreepEntity:SetRadius(3)

	inst:DoTaskInTime(5, inst.Remove)
	inst.persists = false

	return inst
end

return Prefab("spider_web_spit_creep", fn)
```

这是**蜘蛛吐丝法术**留下的临时蛛网——它的核心只有 4 行：

```lua
inst.entity:AddTransform()           -- ① 必须有位置
inst.entity:AddGroundCreepEntity()   -- ② 添加 creep 发射器
inst.entity:AddNetwork()              -- ③ 必须联网（GroundCreep 是服务器权威）
inst.GroundCreepEntity:SetRadius(3)   -- ④ 设置半径
```

剩下两行：
- `inst:DoTaskInTime(5, inst.Remove)` —— 5 秒后自删（emitter 死了，但**铺出去的 creep 会保留**）
- `inst.persists = false` —— entity 不存档

**注意细节**：这里用的是 `Remove`，但没有 `SetRadius(0)`——为什么？因为 spider_web_spit_creep 的设计目的就是**留下永久蛛网**——蛛网由 World.GroundCreep 持久化，所以 emitter 删了也不影响。

#### mod 自定义"持久污染地"4 行起步

把上面 4 行抄到自己的 prefab 里，加 1 行 SpawnPrefab——你就有了一个**会铺设地图覆盖层的怪物巢穴**：

```lua
-- prefabs/myfx_poison_zone.lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddGroundCreepEntity()
    inst.entity:AddNetwork()
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst.GroundCreepEntity:SetRadius(5)  -- 5 格半径
    return inst
end
return Prefab("myfx_poison_zone", fn)
```

业务方调用：
```lua
local zone = SpawnPrefab("myfx_poison_zone")
zone.Transform:SetPosition(x, 0, z)
-- 后续要消失：
zone.GroundCreepEntity:SetRadius(0)  -- ← 让 emitter 不再生产 creep
zone:DoTaskInTime(2, zone.Remove)    -- ← 等 GroundCreep 系统重算后再删
```

#### 三件新手要避免踩雷

**雷点 1：忘 AddNetwork**——GroundCreep 是**服务器权威**系统，没有 Network 会直接报错。

**雷点 2：在客户端 SetRadius**——SetRadius 必须在 `if not TheWorld.ismastersim then return inst end` **之后**调用：
```lua
inst.entity:AddNetwork()
inst.entity:SetPristine()
if not TheWorld.ismastersim then
    return inst   -- ← 客户端到这里就 return 了，下面不执行
end
inst.GroundCreepEntity:SetRadius(3)  -- ← 服务器才执行
```
理解：客户端只看见**渲染好的 creep 层**，不能修改它。修改逻辑全在 server。

**雷点 3：以为复用 WEBCREEP 就万事大吉**——目前 `GROUND_CREEP_IDS` **只有一个 WEBCREEP**（值=1）。所有调用 `AddGroundCreepEntity` 的 prefab，铺出去的都是**同一种 creep**（蛛网纹理）。如果你想要**不同纹理**的 creep（比如毒气紫色），需要：
1. 扩展 `GROUND_CREEP_IDS`（加 `MYPOISON = 2`）；
2. 在 modmain 里 `AddGroundCreep(MYPOISON, {...})`；
3. 但是**C++ 引擎层默认只识别 WEBCREEP 这一种 emitter 输出**——你需要研究 modutil + 引擎绑定（14.5.6 详解）。

> **新手建议**：先用 WEBCREEP 跑通流程；自定义贴图等进入 14.5.6。

#### 半径数值参考表（来自源码）

| 用途 | 半径 | 来源 |
|---|---|---|
| 蜘蛛吐丝 | 3 | `spider_web_spit_creep.lua:13` |
| 蜘蛛巢小阶段 | `TUNING.SPIDERDEN_CREEP_RADIUS[1]` ≈ 3 | `spiderden.lua:165` |
| 蜘蛛巢中阶段 | `TUNING.SPIDERDEN_CREEP_RADIUS[2]` ≈ 4 | 同上 |
| 蜘蛛巢大阶段 | `TUNING.SPIDERDEN_CREEP_RADIUS[3]` ≈ 5 | 同上 |
| 月蜘蛛巢 | `TUNING.MOONSPIDERDEN_CREEPRADIUS[LARGE]` ≈ 6 | `moonspiderden.lua:74` |
| 蜘蛛洞 | 5 | `spiderhole.lua:148` |
| 蜘蛛吊线巢 | 5 | `dropperweb.lua:92` |

**经验法则**：
- 短时特效（魔法陷阱）：3 ~ 4
- 怪物巢穴小型：4 ~ 5
- 怪物巢穴大型：5 ~ 7
- 巨型 boss 领域：8+

---

### 14.5.3（新手）locomotor 联动——3 个开关决定"谁踩到谁慢、谁踩到谁触发"

#### 玩家走在蛛网上为什么会减速？

打开 `scripts/components/locomotor.lua` 第 604 ~ 635 行——所有 creep 联动逻辑都在这里：

```604:635:scripts/components/locomotor.lua
function LocoMotor:UpdateGroundSpeedMultiplier()
    local x, y, z = self.inst.Transform:GetWorldPosition()
    local oncreep = TheWorld.GroundCreep:OnCreep(x, y, z)

    if oncreep and self.triggerscreep then
        -- if this ever needs to happen when self.enablegroundspeedmultiplier is set, need to move the check for self.enablegroundspeedmultiplier above
        if not self.wasoncreep then
            local spawners = TheWorld.GroundCreep:GetTriggeredCreepSpawners(x, y, z)
            local eventdata = { target = self.inst, spawners = spawners, }
            for _, v in ipairs(spawners) do
                v:PushEvent("creepactivate", eventdata)
            end
            self.inst:PushEvent("walkoncreep", eventdata)
            self.wasoncreep = true
        end

        if not self.inst:HasTag("vigorbuff") then
            self.groundspeedmultiplier = self.slowmultiplier
        end
    else
        if self.wasoncreep and self.triggerscreep then
            self.inst:PushEvent("walkoffcreep")
        end
        self.wasoncreep = false

        local current_ground_tile = TheWorld.Map:GetTileAtPoint(x, 0, z)
        self.groundspeedmultiplier = (self:IsFasterOnGroundTile(current_ground_tile) or
                                     (self:FasterOnRoad() and ((RoadManager ~= nil and RoadManager:IsOnRoad(x, 0, z)) or GROUND_ROADWAYS[current_ground_tile])) or
                                     (oncreep and self:FasterOnCreep()))
									 and self.fastmultiplier
									 or 1
```

读完这段代码——3 个开关、3 个事件，全部呈现：

#### 三个开关的完整含义

| 开关 | 默认值 | 设置方法 | 作用 |
|---|---|---|---|
| **`triggerscreep`** | `true` | `loco:SetTriggersCreep(false)` | 是否会"触发" creep 反应（推 walkoncreep/creepactivate）+ 是否减速 |
| **`fasteroncreep`** | `false` | `loco:SetFasterOnCreep(true)` | 是否在 creep 上**反而加速**（蜘蛛低语者专属） |
| **`slowmultiplier`** | 0.6 | `loco:SetSlowMultiplier(0.6)` | 减速倍率（默认 0.6 即 60%） |

#### 为什么蜘蛛走在自己的网上不会陷？

打开 `scripts/prefabs/spider.lua`：
```lua
-- 第 629 行
inst.components.locomotor:SetTriggersCreep(false)
```

蜘蛛**不触发** creep——所以 `triggerscreep=false`，整个 if 分支不进，不会减速也不会推事件。

#### 蜘蛛低语者（webber）为什么走蛛网上更快？

打开 `scripts/prefabs/player_common_extensions.lua`：

```22:38:scripts/prefabs/player_common_extensions.lua
local function ConfigurePlayerLocomotor(inst)
    inst.components.locomotor:SetSlowMultiplier(0.6)
    inst.components.locomotor.pathcaps = { player = true, ignorecreep = true } -- 'player' cap not actually used, just useful for testing
    inst.components.locomotor.walkspeed = TUNING.WILSON_WALK_SPEED -- 4
    inst.components.locomotor.runspeed = TUNING.WILSON_RUN_SPEED -- 6
    inst.components.locomotor.fasteronroad = true
    inst.components.locomotor:SetFasterOnCreep(inst:HasTag("spiderwhisperer"))
    inst.components.locomotor:SetTriggersCreep(not inst:HasTag("spiderwhisperer"))
    ...
```

注意第 28 ~ 29 两行——同时操作两个开关：
- `SetFasterOnCreep(inst:HasTag("spiderwhisperer"))` → webber 的 fasteroncreep = true
- `SetTriggersCreep(not inst:HasTag("spiderwhisperer"))` → webber 的 triggerscreep = false

合起来效果：**webber 走蛛网不减速、不触发蜘蛛巢、反而提速** —— 完美契合"蜘蛛朋友"人设。

#### 哪些怪物 / 实体不触发 creep？

通过 `Grep "SetTriggersCreep(false)"` 一搜，全游戏共有 **17 个 prefab** 关闭 triggerscreep：

| prefab | 角色 | 关闭原因 |
|---|---|---|
| spider | 蜘蛛 | 自己人 |
| beeguard | 蜂卫 | 飞行单位 |
| abigail | 阿比盖尔 | 鬼魂 |
| wobysmall | 小沃比 | 宠物 |
| merm | 鱼人 | 阵营 |
| shadowwaxwell | 影戏分身 | 不实体 |
| corpse_gestalt | 月族尸体 | 飞行 |
| lunarthrall_plant_gestalt | 月族灵魂 | 不实体 |
| lunar_grazer | 月族游荡者 | 飞行 |
| lightflier | 萤火虫 | 飞行 |
| wormwood_lightflier | 沃姆伍德萤火虫 | 飞行 |
| wagdrone_flying | 瓦格无人机 | 飞行 |
| wx78_scanner | WX78 扫描器 | 飞行 |
| wx78_shadowdrone_debuffer | WX78 阴影无人机 | 飞行 |
| wx78_shadowdrone_harvester | 同上 | 飞行 |
| itemmimic_revealed | 拟态怪 | 阵营/特殊 |

**经验法则**：
- 飞行单位 → 关闭
- 蜘蛛阵营成员 → 关闭
- 鬼魂 / 影分身 / 不实体存在 → 关闭
- 普通玩家、普通陆地怪物 → 默认开启

#### 鸡？牛？怎么不在 17 个里？

普通的牛、兔、狗——它们**默认 triggerscreep=true**，所以走在蛛网上也会减速 + 触发蜘蛛巢——这是 game design 的一部分（"蛛网阻拦野兽"）。

#### mod 例子：让自定义怪物"无视蛛网"

```lua
-- 在 modmain.lua 或 prefab 内
AddPrefabPostInit("mybat", function(inst)
    if not TheWorld.ismastersim then return end
    inst.components.locomotor:SetTriggersCreep(false)  -- 飞行 → 关闭
end)
```

#### 半小时 demo——给玩家加一个"穿网鞋"装备

```lua
-- 装备时关闭触发
local function onequip(inst, owner)
    owner.components.locomotor:SetTriggersCreep(false)
end

-- 卸下时恢复
local function onunequip(inst, owner)
    -- 仅普通玩家恢复（蜘蛛低语者要保持自己的逻辑）
    if not owner:HasTag("spiderwhisperer") then
        owner.components.locomotor:SetTriggersCreep(true)
    end
end
```

**注意**：装备穿脱时不要直接 SetSlowMultiplier 改速度——这会**永久改变玩家**。`SetTriggersCreep(false)` 才是"穿网鞋"的正确实现。

#### 三个事件——14.5.5 详解

```text
walkoncreep     ← 玩家踩上 creep 时推（target = 玩家, spawners = 附近 emitters）
walkoffcreep    ← 玩家走出 creep 时推（无 data）
creepactivate   ← 推给"产生 creep 的 emitter"（target = 踩它的玩家, spawners = 自己 + 邻居）
```

新手只要知道有这 3 个事件就行——14.5.5 会展示**蜘蛛巢如何利用 creepactivate 派出 investigator 蜘蛛**——这是整个 GroundCreep 系统**最妙的设计**。

---

### 14.5.4（进阶）官方 5 大 GroundCreep prefab 对照表

GroundCreep 系统的 5 个官方 prefab——按"复杂度递增"排列：

| prefab | 半径 | 半径管理方式 | 持久 | 联动事件 | 用途 |
|---|---|---|---|---|---|
| **spider_web_spit_creep** | 3（固定）| `SetRadius(3)` 一次性 | 5 秒后 emitter 删 | 无 | 蛛丝法术留下的蛛网 |
| **dropperweb** | 5（固定）| `SetRadius(5)` 一次性 | 永久 | 通过子蜘蛛自动间接 | 蜘蛛吊线巢 |
| **spiderhole** | 5（固定 / 可关）| `SetRadius(5)` if hascreep | 永久 | 无 | 蜘蛛洞（commonfn 复用） |
| **spiderden** | 3 ~ 5 动态 | `SetRadius(TUNING.SPIDERDEN_CREEP_RADIUS[stage])` | 永久 | `creepactivate → SpawnInvestigators` | 标准蜘蛛巢 |
| **moonspiderden** | 6 ~ 8 动态 | `iscaveday` 切换 | 永久 | 多重 | 月蜘蛛巢 |

下面逐个剖析。

#### 14.5.4.1 spider_web_spit_creep——最简临时蛛网

已在 14.5.2 完整展示。**特点**：emitter 5 秒删但 creep 留下。**用途**：蛛丝法术、网格陷阱触发后留痕。

#### 14.5.4.2 dropperweb——固定半径永久蛛网

```84:103:scripts/prefabs/dropperweb.lua
local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddGroundCreepEntity()
    inst.entity:AddMiniMapEntity()
    inst.entity:AddNetwork()

    inst.GroundCreepEntity:SetRadius(5)
    inst:AddTag("cavedweller")
    inst:AddTag("spiderden")
    inst.MiniMapEntity:SetIcon("whitespider_den.png")

    inst.entity:SetPristine()

    if not TheWorld.ismastersim then
        return inst
    end

    inst:ListenForEvent("creepactivate", SpawnInvestigators)
```

**特点**：
- **固定 5 半径**，从生成到死亡不变化。
- 与 standard spiderden 的区别：用了 `iscaveday` 联动 / `mushtree_webbed` 缠绕逻辑（OnEntityWake 函数）来扩展自己的网络——但**creep 半径**始终不动。
- 仍然监听 `creepactivate` → 派 investigator 蜘蛛（通过 SpawnInvestigators 的简化版本）。

**适合做**：boss 区域 / 永久封锁带。

#### 14.5.4.3 spiderhole——commonfn 模板的"hascreep 开关"

```137:159:scripts/prefabs/spiderhole.lua
local function commonfn(anim, minimap_icon, tag, hascreep)
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddMiniMapEntity()
    inst.entity:AddNetwork()

    if hascreep then
        inst.entity:AddGroundCreepEntity()
        inst.GroundCreepEntity:SetRadius(5)
    end

    MakeObstaclePhysics(inst, 2)

    inst.AnimState:SetBank("spider_mound")
    inst.AnimState:SetBuild("spider_mound")
    inst.AnimState:PlayAnimation(anim)
```

**亮点**：commonfn 通过 `hascreep` 形参控制是否添加 GroundCreepEntity——**同一份代码生成"会铺网"和"不会铺网"两类蜘蛛洞**。

**mod 启示**：
- 如果你的 mod 有"普通巢"和"领主巢"两种变体——用同一个 commonfn + hascreep 形参即可分支。
- 注意**条件式 AddGroundCreepEntity**——如果加了又不 SetRadius，等于半径默认 0（不铺）。

#### 14.5.4.4 spiderden——动态半径 + 染目联动（最复杂的 vanilla 案例）

##### 设阶段时改半径

```141:175:scripts/prefabs/spiderden.lua
local function SetStage(inst, stage, skip_anim)
    -- if childspawner doesn't exist, then this den is burning down
    if stage <= 3 and inst.components.childspawner ~= nil then
        inst.SoundEmitter:PlaySound("dontstarve/creatures/spider/spiderLair_grow")
        inst.components.childspawner:SetMaxChildren(math.floor(SpringCombatMod(TUNING.SPIDERDEN_SPIDERS[stage])))
        inst.components.childspawner:SetMaxEmergencyChildren(TUNING.SPIDERDEN_EMERGENCY_WARRIORS[stage])
        inst.components.childspawner:SetEmergencyRadius(TUNING.SPIDERDEN_EMERGENCY_RADIUS[stage])
        inst.components.health:SetMaxHealth(TUNING.SPIDERDEN_HEALTH[stage])

        inst.MiniMapEntity:SetIcon("spiderden_" .. tostring(stage) .. ".png")

        if not skip_anim then
            inst.AnimState:PlayAnimation(inst.anims.init)
            inst.AnimState:PushAnimation(inst.anims.idle, true)
        end
    end

    inst.components.upgradeable:SetStage(stage)
    inst.data.stage = stage -- track here, as growable component may go away

    if POPULATING then
        if not inst.loadtask then
            inst.loadtask = inst:DoTaskInTime(0, function()
                if inst:GetCurrentPlatform() == nil then
                    inst.GroundCreepEntity:SetRadius(TUNING.SPIDERDEN_CREEP_RADIUS[inst.data.stage])
                end
                inst.loadtask = nil
            end)
        end
    else
        if inst:GetCurrentPlatform() == nil then
            inst.GroundCreepEntity:SetRadius(TUNING.SPIDERDEN_CREEP_RADIUS[inst.data.stage])
        end
    end
end
```

**关键点**：
1. **半径与阶段绑定**：每次 SetStage 都根据 `TUNING.SPIDERDEN_CREEP_RADIUS[stage]` 重设。
2. **POPULATING 兜底**：如果是世界生成阶段，**延迟 1 帧执行**——避免 creep 在地图未完全生成时铺出去。
3. **平台检测**：`inst:GetCurrentPlatform() == nil` —— **如果蜘蛛巢被放到船上，不铺 creep**（因为 GroundCreep 是世界层贴图，不能附在船上）。

##### 烧毁时清空 creep

```615:618:scripts/prefabs/spiderden.lua
        --- ...
        inst.GroundCreepEntity:SetRadius(0)
```

**SetRadius(0) 不是"瞬间清"**——它是停止 emitter 生产新 creep。已经铺的 creep 会**慢慢消失**（由 GroundCreep 系统的 falloff 机制重算）。这正是为什么蜘蛛巢被烧时蛛网会"渐隐"的原因。

##### bedazzlement 染目联动

打开 `scripts/components/bedazzlement.lua`：

```lua
self.inst.GroundCreepEntity:SetRadius(TUNING.SPIDERDEN_CREEP_RADIUS_BEDAZZLED)
-- 退染目时
self.inst.GroundCreepEntity:SetRadius(TUNING.SPIDERDEN_CREEP_RADIUS[self.inst.data.stage])
```

**亮点**：染目状态用**单独的较小半径**（visual signal that it's "tamed"），退染目时**回到 stage 对应的标准半径**。

##### creepactivate 联动 SpawnInvestigators

```393:412:scripts/prefabs/spiderden.lua
local function IsInvestigator(child)
    return child.components.knownlocations:GetLocation("investigate") ~= nil
end

local function SpawnInvestigators(inst, data)
    if not inst.components.health:IsDead() and not (inst.components.freezable ~= nil and inst.components.freezable:IsFrozen()) then
        inst.AnimState:PlayAnimation(inst.anims.hit)
        inst.AnimState:PushAnimation(inst.anims.idle)

        if inst.components.childspawner ~= nil then
            ...
```

```lua
inst:ListenForEvent("creepactivate", SpawnInvestigators)
```

**事件链全貌**：
1. 玩家踩到 spiderden 周围的蛛网
2. locomotor:UpdateGroundSpeedMultiplier 检测到 oncreep + triggerscreep
3. 调 `TheWorld.GroundCreep:GetTriggeredCreepSpawners(x, y, z)` → 返回**空间内所有 GroundCreepEntity**
4. 对每个 spawner 推 `creepactivate` 事件
5. spiderden 接到事件 → SpawnInvestigators → 派出最多 N 只蜘蛛去玩家位置 investigate

这就是为什么**踩别人家蛛网会引来蜘蛛**的实现。

#### 14.5.4.5 moonspiderden——iscaveday 联动 + 跨阶段动态半径

##### 跨阶段重设

```360:372:scripts/prefabs/moonspiderden.lua
local function moonspiderden_fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddMiniMapEntity()
    inst.entity:AddNetwork()

    inst.entity:AddGroundCreepEntity()
    inst.GroundCreepEntity:SetRadius(TUNING.MOONSPIDERDEN_CREEPRADIUS[LARGE])

    MakeObstaclePhysics(inst, 2)
```

##### 阶段切换时同时改 creep 半径

```74:78:scripts/prefabs/moonspiderden.lua
    inst.GroundCreepEntity:SetRadius(TUNING.MOONSPIDERDEN_CREEPRADIUS[new_stage])
    inst._num_investigators = TUNING.MOONSPIDERDEN_MAX_INVESTIGATORS[new_stage]

    inst._stage = new_stage
end
```

##### iscaveday 切换 watcher

`OnInit` 内 `inst:WatchWorldState("iscaveday", OnIsCaveDay)`——day/night 切换时刷新阶段——**creep 半径会根据 day/night 自动收缩或扩张**。

**给 mod 的启示**：
- **白天/夜晚不同地盘** → 用 `iscaveday` watcher
- **季节差异化** → 用 `seasontick` watcher
- **被攻击时收缩** → 用 healthchange listener

#### 5 大模板对照——你应该用哪个？

| 你的 prefab 是… | 选哪个模板 |
|---|---|
| 一次性魔法陷阱、留 5 秒蛛网 | spider_web_spit_creep |
| 永久性怪物巢穴、固定半径 | dropperweb |
| 同源多变体（有 / 无 creep） | spiderhole（hascreep 开关） |
| 多阶段成长 / 烧毁 / 染目联动 | spiderden（最复杂、最完整） |
| 多 watcher（昼夜 / 季节）联动 | moonspiderden |

---

### 14.5.5（进阶）creepactivate / walkoncreep / walkoffcreep 三大事件链

#### 三事件全景图

```
玩家 entity 走动（1 帧 1 次）
   │
   ▼
locomotor:UpdateGroundSpeedMultiplier()
   │
   ├── oncreep = TheWorld.GroundCreep:OnCreep(x,y,z)
   │
   ▼
   ┌── if oncreep and triggerscreep ──┐
   │                                  │
   │   if not wasoncreep:              │   ← 状态机：刚踩上时
   │     ├── spawners = GetTriggered…  │      
   │     ├── for v in spawners do       │
   │     │     v:PushEvent("creepactivate", {target,spawners})
   │     │   end                        │   ← ① 推给所有 spawner
   │     ├── inst:PushEvent("walkoncreep",{target,spawners})
   │     │                              │   ← ② 推给玩家
   │     └── wasoncreep = true          │
   │                                    │
   │   slowmultiplier 应用              │
   │                                    │
   └── else if wasoncreep ─────────────┘
              │
              ├── inst:PushEvent("walkoffcreep")  ← ③ 推给玩家
              └── wasoncreep = false
```

#### 三事件核心数据

| 事件 | 推送对象 | 数据 keys | 推送时机 |
|---|---|---|---|
| `creepactivate` | spawner（产生 creep 的 entity） | `target`：刚踩上的 entity；`spawners`：list（含自己 + 邻居） | 玩家从 nocreep → oncreep |
| `walkoncreep` | entity 自己 | `target`：自己；`spawners`：list | 同上 |
| `walkoffcreep` | entity 自己 | 无 data | 玩家从 oncreep → nocreep |

#### 监听 walkoncreep 做什么？

**全游戏只有 5 个文件监听 walkoncreep / walkoffcreep / creepactivate**：

| 文件 | 监听事件 | 用途 |
|---|---|---|
| `spiderden.lua` | creepactivate | SpawnInvestigators |
| `dropperweb.lua` | creepactivate | SpawnInvestigators 简化版 |
| `mushtree_webbed.lua` | creepactivate（手动 push） | 缠绕树被破坏时通知附近蜘蛛巢 |
| `locomotor.lua` | walkoncreep / walkoffcreep（推送方） | - |

注意 `mushtree_webbed.lua` 是**主动推送**——而不是监听。它**模拟玩家踩到 creep**：

```175:182:scripts/prefabs/mushtree_webbed.lua
    local pos = inst:GetPosition()
    local dens = TheSim:FindEntities(pos.x, pos.y, pos.z, TUNING.MUSHTREE_WEBBED_SPIDER_RADIUS, SPIDERDEN_TAGS)
    if #dens > 0 then
        local creepactivate_data = {target = worker}
        for _, den in ipairs(dens) do
            den:PushEvent("creepactivate", creepactivate_data)
        end
    end
```

——树被砍时，向附近蜘蛛巢"伪装"一次 creepactivate，触发蜘蛛巢派出 investigator。**这是 mod 的好启示**：你也可以**手动推 creepactivate 事件**给附近的 creep emitters，模拟"激惹"。

#### 一个完整的 SpawnInvestigators 简化模板

```lua
local function SpawnInvestigators(inst, data)
    if inst.components.health:IsDead() then return end
    if inst.components.childspawner == nil then return end
    
    local target = data.target
    if target == nil or target.components.health == nil or target.components.health:IsDead() then
        return
    end
    
    local num = math.min(2, inst.components.childspawner.childreninside)
    local x, y, z = target.Transform:GetWorldPosition()
    
    for i = 1, num do
        local child = inst.components.childspawner:SpawnChild(target, nil, 5)
        if child ~= nil then
            child:DoTaskInTime(0, function()
                child.components.knownlocations:RememberLocation(
                    "investigate",
                    Vector3(x, y, z),
                    false  -- 不持久
                )
            end)
        end
    end
end

inst:ListenForEvent("creepactivate", SpawnInvestigators)
```

#### 监听 walkoncreep —— 玩家端做"立即反应"

`creepactivate` 推给的是 **emitter 本身**——意味着 emitter 知道有人踩了它。但**玩家本人**怎么知道？通过 `walkoncreep`：

```lua
-- 给自定义角色添加："踩到蛛网时弹个气泡 / 触发减益"
AddPlayerPostInit(function(inst)
    if not TheWorld.ismastersim then return end
    inst:ListenForEvent("walkoncreep", function(inst, data)
        -- inst.components.talker:Say("好黏！")
        inst.components.debuffable:AddDebuff("slowdebuff", "slowdebuff")
    end)
    inst:ListenForEvent("walkoffcreep", function(inst, data)
        inst.components.debuffable:RemoveDebuff("slowdebuff")
    end)
end)
```

#### birdspawner——不监听事件，但用 GroundCreep:OnCreep 做"反检测"

```465:476:scripts/components/birdspawner.lua
        local spawnpoint_x, spawnpoint_y, spawnpoint_z = (pt + offset):Get()
        if TheWorld.Map:IsPointInWagPunkArenaAndBarrierIsUp(spawnpoint_x, spawnpoint_y, spawnpoint_z) then
            return false
        end
        local allow_water = true
        local in_moonstorm = TheWorld.net.components.moonstorms and TheWorld.net.components.moonstorms:IsXZInMoonstorm(spawnpoint_x, spawnpoint_z)

        return _map:IsPassableAtPoint(spawnpoint_x, spawnpoint_y, spawnpoint_z, allow_water) and
               #(TheSim:FindEntities(spawnpoint_x, 0, spawnpoint_z, 4, BIRDBLOCKER_TAGS)) == 0 and
               --A corpse isn't gonna care if it's the moonstorm or on creep!
               (is_corpse or (not in_moonstorm and not _groundcreep:OnCreep(spawnpoint_x, spawnpoint_y, spawnpoint_z)))
    end
```

**这是反向利用**：
- 鸟群在**鸟点候选位置**调 `_groundcreep:OnCreep(x,y,z)`
- 如果这个点在 creep 上 → **不刷鸟**
- 设计逻辑："鸟不会在蛛网区域出现"

**mod 启示**：
- 你的 mod 有"刷怪点"逻辑 → 用 `OnCreep` 排除 creep 区域
- 你的 mod 有"采集稀有植物"判定 → 用 `OnCreep` 反向区分**蛛网生物群系**和**正常生物群系**

#### worldtiledefs——脚步声系统的 creep 联动

```140:151:scripts/worldtiledefs.lua
				local x, y, z = inst.Transform:GetWorldPosition()
				local oncreep = TheWorld.GroundCreep:OnCreep(x, y, z)
				local onsnow = not tileinfo.nogroundoverlays and TheWorld.state.snowlevel > 0.15
				local onmud = not tileinfo.nogroundoverlays and TheWorld.state.wetness > 15

				if isplayer and not oncreep and RoadManager and RoadManager:IsOnRoad(x, 0, z) then
					--this is only for players for the time being because isonroad is suuuuuuuper slow.
					tile = WORLD_TILES.ROAD
					tileinfo = GetTileInfo(WORLD_TILES.ROAD) or tileinfo
				end

				soundpath =
					(oncreep and "dontstarve/movement/run_web") or
```

**逻辑**：脚步声优先级：creep > snow > mud > 默认 tile sound

——这意味着**只要踩到 creep，就播 run_web**——和具体哪种 creep（WEBCREEP 还是其他 mod 自定义）无关——这是 mod 自定义 creep 的**一个限制**：脚步声目前**写死成 run_web**，要改的话需要 hook worldtiledefs 或自己监听 walkoncreep 播放音效。

---

### 14.5.6（进阶）mod 自定义草地——4 步注册 AddTile + AddGroundCreep + AddFalloffTexture

#### TileManager 三大注册函数

打开 `scripts/tilemanager.lua` 第 269 ~ 312 行：

```269:294:scripts/tilemanager.lua
local function ValidateGroundCreepDef(groundcreep_def)
    assert(groundcreep_def.name, "groundcreep_def must contain a name")
    assert(groundcreep_def.noise_texture, "groundcreep_def must contain a noise_texture")

    groundcreep_def.texture_name = GroundImage(groundcreep_def.name)
    groundcreep_def.atlas = GroundAtlas(groundcreep_def.atlas or groundcreep_def.name)
    groundcreep_def.noise_texture = GroundNoise(groundcreep_def.noise_texture)
end

local function AddGroundCreep(groundcreep_id, groundcreep_def)
    if groundcreep_def then
        ValidateGroundCreepDef(groundcreep_def)

        table.insert(GroundTiles.creep, {groundcreep_id, groundcreep_def})

        AddAssets(groundcreep_def, assets)
    end
end
```

`AddGroundCreep` 做三件事：
1. **校验 def**——必须有 `name` 和 `noise_texture`
2. **路径解析**——`GroundImage("web")` → `levels/textures/web.tex`；`GroundNoise("web_noise")` → `levels/textures/noise/web_noise.tex`
3. **入表**——添加到 `GroundTiles.creep` 列表 + 资产表

#### modutil 的 mod 接口

mod 端使用的不是 `TileManager.AddGroundCreep` 直接调，而是 modutil 包装的 `env.AddTile / env.AddFalloffTexture`：

```364:411:scripts/modutil.lua
	env.AddTile = function(tile_name, tile_range, tile_data, ground_tile_def, minimap_tile_def, turf_def)
		initprint("AddTile", tile_name)
		mod_protect_TileManager = false
		TileManager.AddTile(
			tile_name,
			tile_range,
			tile_data,
			ground_tile_def,
			minimap_tile_def,
			turf_def
		)
		mod_protect_TileManager = true
	end
	...
	env.AddFalloffTexture = function(falloff_id, falloff_def)
		initprint("AddFalloffTexture", falloff_id)
		mod_protect_TileManager = false
		TileManager.AddFalloffTexture(falloff_id, falloff_def)
		mod_protect_TileManager = true
	end
```

**注意**：modutil **没有** `env.AddGroundCreep`！这意味着 mod 想自定义 creep——需要：

**方法 A：直接用 require**
```lua
-- modmain.lua
local TileManager = require("tilemanager")
local mod_protect_TileManager = _G.mod_protect_TileManager
_G.mod_protect_TileManager = false
TileManager.AddGroundCreep(MY_CREEP_ID, {
    name = "my_creep",
    noise_texture = "my_creep_noise"
})
_G.mod_protect_TileManager = mod_protect_TileManager
```

**方法 B：扩展 GROUND_CREEP_IDS**
```lua
-- modmain.lua
GLOBAL.GROUND_CREEP_IDS.MY_POISON = 2  -- 新加 ID
```

#### 自定义 turf —— 完整 4 步流程

mod 想做"我的草坪"（玩家可挖出来当 turf 物品）的完整流程：

##### 第 1 步：注册资源

```lua
-- modmain.lua
Assets = {
    Asset("IMAGE", "levels/textures/my_grass.tex"),
    Asset("IMAGE", "levels/textures/noise/my_grass_noise.tex"),
    Asset("IMAGE", "levels/textures/my_grass.xml"),  -- atlas
    Asset("IMAGE", "minimap/my_grass.tex"),
    Asset("IMAGE", "minimap/my_grass.xml"),
    Asset("IMAGE", "images/inventoryimages/turf_my_grass.tex"),
    Asset("IMAGE", "images/inventoryimages/turf_my_grass.xml"),
}
```

##### 第 2 步：注册 tile range

```lua
-- modmain.lua
AddTileRange("MYMOD_LAND", 1024, 1027)  -- 4 个 tile id 区间
```

##### 第 3 步：AddTile（含 turf_def）

```lua
AddTile(
    "MY_GRASS",                       -- tile_name → 会被加到 WORLD_TILES.MY_GRASS
    "MYMOD_LAND",                     -- range
    { ground_name = "我的草地" },     -- tile_data
    {                                  -- ground_tile_def
        name = "my_grass",
        noise_texture = "my_grass_noise",
        runsound = "dontstarve/movement/run_grass",
        walksound = "dontstarve/movement/walk_grass",
        snowsound = "dontstarve/movement/run_ice",
        mudsound = "dontstarve/movement/run_mud",
        colors = {
            primary_color =        {220, 240, 220, 60},
            secondary_color =      {21,  120, 90,  150},
            secondary_color_dusk = {0,   0,   0,   50},
            minimap_color =        {30,  90,  60,  102},
        },
    },
    {                                  -- minimap_tile_def
        name = "minimap_my_grass",
        noise_texture = "my_grass_noise",
    },
    {                                  -- turf_def
        anim = "turf_my_grass",
        bank_build = "turf_my_grass",
        bank_build_file = "anim/turf_my_grass.zip",
    }
)
```

##### 第 4 步（可选）：AddGroundCreep（覆盖层）

```lua
-- 仅当你想让 mod 物体在地面铺"覆盖层"
GLOBAL.GROUND_CREEP_IDS.MYMOD_POISON = 2

local TileManager = require("tilemanager")
GLOBAL.mod_protect_TileManager = false
TileManager.AddGroundCreep(GLOBAL.GROUND_CREEP_IDS.MYMOD_POISON, {
    name = "mymod_poison",          -- → levels/textures/mymod_poison.tex
    noise_texture = "mymod_poison_noise",
})
GLOBAL.mod_protect_TileManager = true
```

##### 第 5 步（可选）：AddFalloffTexture（边缘过渡）

```lua
AddFalloffTexture(99, {
    name = "mymod_falloff",
    noise_texture = "mymod_falloff_noise",
    -- 决定与邻居 tile 的过渡
    should_have_falloff_result = true,
    neighbor_needs_falloff_result = true,
})
```

#### 关于 `mod_protect_TileManager`

```360:362:scripts/modutil.lua
		mod_protect_TileManager = false
		TileManager.RegisterTileRange(range_name, range_start, range_end)
		mod_protect_TileManager = true
```

每次注册前后切换这个全局标志——是为了**防止 mod 在不该调用时（运行时）误调 TileManager**。所有 TileManager 内部函数都会 assert：
```lua
assert(mod_protect_TileManager == false, "Calling AddTile directly is not allowed")
```

**给 mod 的限制**：必须**在 modmain.lua 顶层执行期间**调用 TileManager（即环境刚刚加载完，还没运行时）；运行时调用会 assert 失败。

#### 实战参考：whisperer mod 的"自定义蛛网"做法

这是**蜘蛛低语者（webber）**风格 mod 的常见做法——它们一般不重写 GroundCreep，而是：

1. 复用 `WEBCREEP`（默认蛛网纹理），通过 `inst.entity:AddGroundCreepEntity()` 让自己的怪物铺"伪蛛网"
2. 修改的是**locomotor 行为**：
   - 给玩家加 spiderwhisperer tag → 自动 fasteroncreep + 不 triggerscreep
   - 给怪物 SetTriggersCreep(false) → 不触发反应

**只有当你想做完全不同纹理的覆盖层（紫色毒雾、金色月光草）才需要 AddGroundCreep。**

#### 为什么 `worldtiledefs.lua` 写死 `run_web` 是个坑？

之前提到——只要 oncreep 就播 `run_web` 脚步声。如果你做了"毒雾 creep"，脚步声还是蛛网音效——这显然不对。

**workaround**：
```lua
-- modmain
AddPlayerPostInit(function(inst)
    if not TheWorld.ismastersim then return end
    inst:ListenForEvent("walkoncreep", function(inst)
        local x, y, z = inst.Transform:GetWorldPosition()
        -- 自己做空间检测：在我的毒雾 emitter 范围内吗？
        local emitters = TheSim:FindEntities(x, 0, z, 5, {"my_poison_emitter"})
        if #emitters > 0 then
            -- 覆盖播自己的音效
            inst.SoundEmitter:PlaySound("mymod/movement/run_poison")
        end
    end)
end)
```

---

### 14.5.7（老手）底层架构——3 个 GroundCreep entity 角色

#### 三个名字相似的"GroundCreep"——它们是不同的对象

老手第一道坎：饥荒里有**三个 entity 角色**都叫"GroundCreep 什么的"——但作用完全不同：

| 名字 | 出处 | 类型 | 数量 | 作用 |
|---|---|---|---|---|
| `inst.entity:AddGroundCreep()` | C++ entity | World 上的 entity component | **全局唯一**（只在 World 上）| 渲染 creep 层 + 检测 + 存档 |
| `inst.GroundCreep` | World 上的属性 | C++ entity component（同上）| 全局唯一 | 提供 :OnCreep / :GetTriggeredCreepSpawners / :GetAsString / :SetFromString / :FastForward / :AddRenderLayer 等 API |
| `inst.entity:AddGroundCreepEntity()` | C++ entity | 个体 entity 上的 emitter | **每个生产 creep 的物体一个** | 在自己周围按半径铺 creep |
| `inst.GroundCreepEntity` | 个体 entity 上的属性 | 同上 | 同上 | 提供 :SetRadius |
| `scripts/components/groundcreep.lua` | Lua component | 挂在 World 上的 lua component | 全局唯一 | 仅做 OnSave / OnLoad 兜底（调用 inst.GroundCreep:GetAsString / SetFromString）|

**关键区分**：
- `inst.GroundCreep`（无 Entity 后缀）= **World 上的 entity component**——读 / 写 / 持久化整个地图的 creep 状态。
- `inst.GroundCreepEntity`（有 Entity 后缀）= **每个个体的发射器**——只能 SetRadius。

#### World.GroundCreep —— 全局渲染 + 存档主管

打开 `scripts/prefabs/world.lua` 第 446 行：

```446:504:scripts/prefabs/world.lua
        inst.entity:AddGroundCreep()
        inst.entity:AddSoundEmitter()
        ...
        for i, data in ipairs(GroundTiles.creep) do
            local tile_id, layer_properties = unpack(data)
            local handle = MapLayerManager:CreateRenderLayer(
                tile_id,
                layer_properties.atlas,
                layer_properties.texture_name,
                layer_properties.noise_texture
            )
            inst.GroundCreep:AddRenderLayer(handle)
        end
```

**世界初始化时**：
1. `inst.entity:AddGroundCreep()` —— 给 World entity 添加 creep 系统
2. 遍历 `GroundTiles.creep` 表（来自 `groundcreepdefs.lua`）
3. 对每条 creep def，用 `MapLayerManager:CreateRenderLayer` 创建渲染层 handle
4. 用 `inst.GroundCreep:AddRenderLayer(handle)` 把渲染层附加到 GroundCreep 系统上

**对比 GroundTiles 渲染**（同文件第 459 行）：
```lua
for i, data in ipairs(GroundTiles.ground) do
    ...
    inst.Map:AddRenderLayer(handle)  -- 主地图层
end
```
GroundTiles → `inst.Map`，GroundCreep → `inst.GroundCreep`。两个系统共享 MapLayerManager 但各自有自己的渲染管线。

#### components/groundcreep.lua —— 存档兜底

```1:48:scripts/components/groundcreep.lua
return Class(function(self, inst)

self.inst = inst

local function OnPlayerActivated()
    inst.GroundCreep:FastForward()
end

inst:ListenForEvent("playeractivated", OnPlayerActivated)

if inst.ismastersim then function self:OnSave()
    return inst.GroundCreep:GetAsString()
end end

if inst.ismastersim then function self:OnLoad(data)
    inst.GroundCreep:SetFromString(data)
end end

end)
```

**只做 3 件事**：
1. **playeractivated → FastForward**：玩家激活（连入服务器、视野进入新区域）→ 一次性把 creep 推算到当前应有的状态
2. **OnSave → GetAsString**：把整张地图的 creep 状态序列化为字符串（C++ 实现，存档时调用）
3. **OnLoad → SetFromString**：从存档反序列化恢复

**为什么名字混淆**？这个 lua 文件的目的就是**给 World entity 提供 SaveLoad 钩子**——但**真正的逻辑全在 C++ 的 GroundCreep entity component**。lua 只是个**胶水层**。

> **源码注释自嘲**：第 4 行就说了 "This exists solely to make serializing the ground creep from the map sane, as opposed to insane special case code that lives god knows where"——开发者自己都觉得这个组件名字让人迷糊。

#### inst.GroundCreepEntity —— 个体发射器

它**唯一暴露的 API**：

| API | 类型 | 作用 |
|---|---|---|
| `inst.GroundCreepEntity:SetRadius(r)` | server-only | 设置半径（0 = 不铺，>0 = 铺。立即生效但 falloff 是渐进的） |

**为什么这么简单**？因为 emitter 的设计哲学是**纯声明式**——你只需要告诉它"在我周围 N 格内铺 creep"，**剩下的全在 C++**：网格位置计算、渐变 falloff、邻居合并、跨 chunk 同步——开发者完全不用管。

#### 跨 Lua 调用的完整链路

老手要理解的最重要架构图：

```
玩家移动 1 帧
   │
   ▼ Lua 端
scripts/components/locomotor.lua:UpdateGroundSpeedMultiplier
   │
   │ 调 entity component
   ▼ C++ 边界
TheWorld.GroundCreep:OnCreep(x, y, z)  
   │
   │ C++ 内部
   ▼
查询 4 字节 tile id 网格 → 是否非 0  
   │
   │ 返回 bool
   ▼ Lua 端
locomotor 决定是否减速 / 推事件

──────────────────────────────────────

蜘蛛巢生成 / 阶段变化
   │
   ▼ Lua 端
inst.GroundCreepEntity:SetRadius(5)
   │
   │ 调 emitter component
   ▼ C++ 边界
GroundCreepEntity 写入"我要铺 5 格"  
   │
   │ C++ 内部，每隔几帧
   ▼
World.GroundCreep 系统轮询所有 emitters → 重算每格 tile 的 creep 度
   │
   ▼
更新 4 字节 tile 网格

──────────────────────────────────────

存档 / 加载
   │
   ▼ Lua 端
components/groundcreep.lua:OnSave → inst.GroundCreep:GetAsString()
   │
   │ C++ 实现
   ▼
序列化整个 tile 网格 + 所有 emitter 状态 → string
   │
   ▼ Lua 端
保存 string 到存档
```

#### 玩家激活时的 FastForward

```lua
local function OnPlayerActivated()
    inst.GroundCreep:FastForward()
end
```

**作用**：玩家从未激活区域 / 重新连接 → 触发 GroundCreep 系统**一次性把所有 emitter 的应铺范围都算完**——避免玩家进入区域时看到 creep "慢慢长出来"。

**给 mod 的启示**：如果你做了一个**大区域瞬间铺设 creep 的事件**（boss 战开始 → 全场紫雾），可以**手动调 FastForward**：
```lua
TheWorld:DoTaskInTime(0, function()
    TheWorld.GroundCreep:FastForward()
end)
```

#### MapLayerManager：渲染层管理器

```469:489:scripts/prefabs/world.lua
            local handle = MapLayerManager:CreateRenderLayer(
                tile_id, --embedded map array value
                layer_properties.atlas or resolvefilepath(GroundAtlas(layer_properties.name)),
                layer_properties.texture_name or resolvefilepath(GroundImage(layer_properties.name)),
                resolvefilepath(layer_properties.noise_texture)
            )
			layer_properties._render_layer = i

            local colors = layer_properties.colors
            if colors ~= nil then
				local primary_color = colors.primary_color
                MapLayerManager:SetPrimaryColor(handle, primary_color[1] / 255, primary_color[2] / 255, primary_color[3] / 255, primary_color[4] / 255)
				local secondary_color = colors.secondary_color
				MapLayerManager:SetSecondaryColor(handle, secondary_color[1] / 255, secondary_color[2] / 255, secondary_color[3] / 255, secondary_color[4] / 255)
				local secondary_color_dusk = colors.secondary_color_dusk
				MapLayerManager:SetSecondaryColorDusk(handle, secondary_color_dusk[1] / 255, secondary_color_dusk[2] / 255, secondary_color_dusk[3] / 255, secondary_color_dusk[4] / 255)
                local minimap_color = colors.minimap_color
                MapLayerManager:SetMinimapColor(handle, minimap_color[1] / 255, minimap_color[2] / 255, minimap_color[3] / 255, minimap_color[4] / 255)
            end
```

**两类配置**：
1. **`CreateRenderLayer(tile_id, atlas, texture, noise)`** —— 创建渲染层
2. **`SetPrimaryColor / SetSecondaryColor / SetSecondaryColorDusk / SetMinimapColor`** —— 给地面 tile 染色（不适用于 GroundCreep——GroundCreep 默认用纹理本身的颜色）

**给 mod 的启示**：mod 想给自定义 GroundCreep 加颜色变化——**不能直接调 MapLayerManager**——因为 GroundCreep 的 def 不接 colors 字段。要改的话，**要改 noise_texture 和 atlas 本身**（贴图层面）。

---

### 14.5.8（老手）6 大常见陷阱 + 3 条设计经验

#### 陷阱 1：在客户端调 SetRadius

```lua
-- 错误！客户端会崩
local fx = SpawnPrefab("my_creep_emitter")
fx.GroundCreepEntity:SetRadius(5)  -- ← 在客户端 prefab.fn 内调用就会崩
```

**正解**：所有 SetRadius 必须在 `if not TheWorld.ismastersim then return inst end` 之后。

#### 陷阱 2：以为 SetRadius(0) = 立即清空

```lua
-- 蜘蛛巢被毁
inst.GroundCreepEntity:SetRadius(0)
inst:Remove()  -- ← 立即删
```

**问题**：emitter 删了——但 GroundCreep 系统内还有"上一帧应铺的 creep"——结果**玩家看到 creep 残留几秒**。

**正解**：
```lua
inst.GroundCreepEntity:SetRadius(0)
inst:DoTaskInTime(2, inst.Remove)  -- 等 GroundCreep 重算后再删
```

#### 陷阱 3：在船 / 平台上 AddGroundCreepEntity

GroundCreep 是**世界层贴图**，**不能附在船 / 平台 / 移动 entity 上**。蜘蛛巢的 SetStage 内做了显式平台检测：
```lua
if inst:GetCurrentPlatform() == nil then
    inst.GroundCreepEntity:SetRadius(...)
end
```

**给 mod 的启示**：做"会移动的 creep 源"（船上的污染装置）→ 必须做平台检测——上船时 SetRadius(0)，下船时 SetRadius(5)。

#### 陷阱 4：把 components/groundcreep 当 emitter 用

```lua
-- 错误！这是给 World 加 lua component，不是给个体加 emitter
inst:AddComponent("groundcreep")
```

正解：
```lua
inst.entity:AddGroundCreepEntity()
inst.GroundCreepEntity:SetRadius(5)
```

**记忆诀**：GroundCreepEntity（**有 Entity**）= 个体发射器；groundcreep（**全小写无 Entity**）= World 的 SaveLoad 组件。

#### 陷阱 5：AddGroundCreep 在运行时调

```lua
-- 错误！TileManager 有 mod_protect 守卫
local TileManager = require("tilemanager")
SomePrefab.fn = function()
    TileManager.AddGroundCreep(...)  -- ← assert 失败
end
```

**正解**：必须在 modmain.lua 顶层执行（不是任何延迟回调内）。

#### 陷阱 6：以为 mod 自定义 ID 会自动注册到引擎

```lua
GLOBAL.GROUND_CREEP_IDS.MY_POISON = 2
TileManager.AddGroundCreep(2, {...})

-- 然后...
inst.entity:AddGroundCreepEntity()
inst.GroundCreepEntity:SetRadius(5)  -- ← 但发射的是 WEBCREEP 还是 MY_POISON？
```

**问题**：emitter 默认发射什么 creep id 是**C++ 决定**的——可能没有暴露给 mod 接口让你选 creep_id。

**workaround / 经验**：
- mod 自定义 creep 主要用于**修改 vanilla 蛛网的视觉**（替换 web.tex）
- 想"独立的紫色毒雾"——可以做**独立 prefab + AddRenderLayer hack**——这超出了 mod 接口范围，需要 hack `prefabs/world.lua`

**实际推荐方案**：直接用 vanilla WEBCREEP，但**生产逻辑做差异化**（不同 emitter 不同 SetRadius、不同 creepactivate handler）。

#### 设计经验 1：不要把 GroundCreep 用作"短时特效"

| 场景 | 推荐方案 |
|---|---|
| < 5 秒可见效果 | 用 14.1-14.3 的 FX prefab + 粒子 |
| > 30 秒 / 永久效果 + 无玩家交互 | 用静态 FX prefab |
| > 30 秒 / 永久 + 玩家交互（减速、生物 AI 反应） | **用 GroundCreep** |
| 全屏调色 | 用 14.4 ColourCube |

**根本原因**：GroundCreep 的 OnSave / OnLoad 会序列化**整张地图的 creep 状态**——短时使用会浪费存档体积。

#### 设计经验 2：emitter 半径不要做"瞬间巨变"

```lua
-- 不推荐：从 0 跳到 10
inst.GroundCreepEntity:SetRadius(10)

-- 推荐：渐进
local target_r = 10
local current_r = 0
inst:DoPeriodicTask(0.5, function()
    current_r = math.min(current_r + 1, target_r)
    inst.GroundCreepEntity:SetRadius(current_r)
    if current_r >= target_r then
        inst.spread_task:Cancel()
    end
end)
```

**原因**：
- 视觉上"渐进"更符合自然（蛛网慢慢长出比突然出现自然）
- 性能上 GroundCreep 系统对"半径瞬间增大"的网格重算成本较高

#### 设计经验 3：creepactivate 是"领地反应"的最佳钩子

如果你的 mod 设计需要"踩到怪物领地→ 触发反应"——**永远优先选择 creepactivate**——比手动 FindEntities 检测要快、健壮：

| 方案 | 性能 | 可靠性 |
|---|---|---|
| `OnUpdate + TheSim:FindEntities` | 慢（每帧）| 中（漏掉跨 tile 边界） |
| `walkoncreep` + emitter 监听 creepactivate | **快**（事件驱动）| **高**（locomotor 帧检测）|

**模板**：
```lua
-- emitter prefab
inst.entity:AddGroundCreepEntity()
inst.GroundCreepEntity:SetRadius(5)
inst:ListenForEvent("creepactivate", function(inst, data)
    -- data.target = 踩进来的玩家 / 怪物
    -- 反应：派出兵 / 推减益 / 播警告音
    if data.target.components.health and not data.target.components.health:IsDead() then
        -- ...
    end
end)
```

---

### 14.5.9（小结）速查表 + 4 行起步代码 + 14.6 预告

#### 一表速查 GroundCreep 全 API

| API | 调用方 | 类型 | 用途 |
|---|---|---|---|
| `inst.entity:AddGroundCreep()` | World prefab | 一次性 | 给 world 添加 creep 系统（vanilla 已做） |
| `inst.GroundCreep:AddRenderLayer(h)` | World 初始化 | 一次性 | 注册渲染层（vanilla 已做） |
| `inst.GroundCreep:OnCreep(x,y,z)` | 任意 lua（server / client）| 查询 | bool，是否在 creep 上 |
| `inst.GroundCreep:GetTriggeredCreepSpawners(x,y,z)` | server | 查询 | list，附近的 emitters |
| `inst.GroundCreep:FastForward()` | server | 一次性 | 立即推算 creep 状态 |
| `inst.GroundCreep:GetAsString()` | OnSave | 序列化 | 整张地图的 creep 状态 |
| `inst.GroundCreep:SetFromString(s)` | OnLoad | 反序列化 | 同上 |
| `inst.entity:AddGroundCreepEntity()` | server prefab | 一次性 | 给个体添加发射器 |
| `inst.GroundCreepEntity:SetRadius(r)` | server | 频繁 | 设置铺设半径（0 = 不铺）|

#### locomotor 三开关速查

| 开关 | 默认 | 设置方法 | 应用场景 |
|---|---|---|---|
| `triggerscreep` | true | `loco:SetTriggersCreep(false)` | 关闭 = 蜘蛛 / 飞行 / 鬼魂 |
| `fasteroncreep` | false | `loco:SetFasterOnCreep(true)` | 开启 = 蜘蛛低语者 |
| `slowmultiplier` | 0.6 | `loco:SetSlowMultiplier(0.5)` | 调减速强度（普通 0.6，重负 0.4）|

#### 三事件速查

| 事件 | 推送方向 | 数据 | 监听者举例 |
|---|---|---|---|
| `creepactivate` | locomotor → spawner | {target, spawners} | spiderden / dropperweb |
| `walkoncreep` | locomotor → entity | {target, spawners} | mod 自定义角色减益 |
| `walkoffcreep` | locomotor → entity | 无 | mod 自定义角色解除减益 |

#### 5 大官方 prefab 速查

| prefab | 用途 | 半径 | 特点 |
|---|---|---|---|
| spider_web_spit_creep | 临时蛛网 | 3 | 5 秒删 emitter，creep 留 |
| dropperweb | 永久巢 | 5 | 固定半径 |
| spiderhole | 蜘蛛洞 | 5 / 关 | hascreep 形参开关 |
| spiderden | 标准巢 | 3-5 | 阶段动态 + 染目联动 |
| moonspiderden | 月蜘蛛巢 | 6-8 | iscaveday watcher |

#### 注册自定义 creep 4 步

```lua
-- modmain.lua

-- 1. 资产
Assets = {
    Asset("IMAGE", "levels/textures/my_creep.tex"),
    Asset("IMAGE", "levels/textures/my_creep.xml"),
    Asset("IMAGE", "levels/textures/noise/my_creep_noise.tex"),
}

-- 2. 扩展 ID
GLOBAL.GROUND_CREEP_IDS.MY_POISON = 2

-- 3. 注册
local TileManager = GLOBAL.require("tilemanager")
GLOBAL.mod_protect_TileManager = false
TileManager.AddGroundCreep(GLOBAL.GROUND_CREEP_IDS.MY_POISON, {
    name = "my_creep",
    noise_texture = "my_creep_noise",
})
GLOBAL.mod_protect_TileManager = true

-- 4. 在你的 prefab 里使用
PrefabFiles = { "my_creep_emitter" }
```

#### mod 自定义 creep emitter 4 行起步

```lua
-- prefabs/my_creep_emitter.lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddGroundCreepEntity()
    inst.entity:AddNetwork()
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst.GroundCreepEntity:SetRadius(5)

    inst:ListenForEvent("creepactivate", function(inst, data)
        -- 踩到我的领地的玩家 / 怪物
        print("有人踩到了我的毒雾区:", data.target)
    end)

    return inst
end

return Prefab("my_creep_emitter", fn)
```

业务方调用：
```lua
local emitter = SpawnPrefab("my_creep_emitter")
emitter.Transform:SetPosition(x, 0, z)
-- 烧毁时
emitter.GroundCreepEntity:SetRadius(0)
emitter:DoTaskInTime(2, emitter.Remove)
```

#### 三个设计原则

1. **GroundCreep 是"持久 + 可交互"专用**——别用作短时特效（用 FX）。
2. **半径变化要渐进**——SetRadius 频繁调用没问题，但避免 0 → 10 的跳变（性能 + 视觉）。
3. **creepactivate 是"领地反应"的最佳钩子**——优于 OnUpdate + FindEntities。

---

> **下一节预告**：14.6 节我们将进入 **实战：为自定义技能添加释放特效**——前面 14.1 / 14.2 / 14.3 / 14.4 / 14.5 是"系统理论"——而 14.6 是**完整的端到端实战**：以一个自定义法术（"召唤毒雾领域"）为例——从 stategraph 触发、SpawnPrefab 闪光、ParticleEmitter 持续粒子、ColourCube 全屏氛围、到 GroundCreep 持久领地——**5 个 FX 系统组合在一个技能上**——14.6 会一次串完。




## 14.6 实战：为自定义技能添加释放特效

### 本节导读

> **一句话定位**：这是 14 章的收官实战节——把 14.1（FX 设计模式）、14.2（常用 FX 索引）、14.3（ParticleEmitter）、14.4（ColourCube）、14.5（GroundCreep）**五个系统**串成一个完整、可玩、可发布的**自定义法术**。

#### 为什么需要"实战节"？

前 5 节像是**5 张分散的工具卡**：
- 14.1 教你怎么用 SpawnPrefab + animover 自动回收一个 FX
- 14.2 给你了 449 个常用 FX 的索引，可以直接 `SpawnPrefab("small_puff")` 复用
- 14.3 教你 ParticleEmitter 配置 GPU 粒子（火焰、烟、毒雾）
- 14.4 教你 ColourCube 切换全屏氛围（紫色、夜晚、月色）
- 14.5 教你 GroundCreep 在地面铺设持久"领地"贴图

**单独看每张卡都很简单**——但 mod 玩家真正想做的是："**我自己的角色按 X 释放一个法术，全场要有反应**"——这就需要**5 张卡同时打出**。

#### 我们要造什么

**法术名**：**「毒雾领域」（Mystic Haze Dome）**——一个 AOE 群控法术。

**完整释放流程**（按时间轴）：

```
Frame  0 ─┬─  玩家右键"毒雾法杖"→ buffaction 排入  ◄────  [输入]
          │
          │  SGwilson:GoToState("castspell")
Frame  0 ─┼─  ① 法杖动画 staff_pre 开始播放
          │   ② 玩家身边产生 staffcastfx（紫色光圈）  ◄────  [14.2 复用]
          │   ③ 玩家脚下产生 staff_castinglight（紫光晕）
          │
Frame 13 ─┼─  播放释放音效 dontstarve/wilson/use_gemstaff
          │
Frame 53 ─┼─  PerformBufferedAction → spellfn 触发
          │   ④ SpawnPrefab("myfx_haze_pillar_pre") 落点          ◄──  [14.1 pre/loop/pst]
          │   ⑤ SpawnPrefab("myfx_haze_particles") 持续粒子柱     ◄──  [14.3 ParticleEmitter]
          │   ⑥ TheWorld:PushEvent("overridecolourcube",          ◄──  [14.4 ColourCube]
          │       "images/colour_cubes/myfx_haze_purple.tex")
          │   ⑦ SpawnPrefab("myfx_haze_zone")                     ◄──  [14.5 GroundCreep]
          │       :SetRadius(5)
          │
Frame 69 ─┼─  仪式动画结束 → idle
          │
T+ 0.5s  ─┼─  ④ pillar_pre animover → 自动切到 myfx_haze_pillar_loop
          │
T+ 5.0s  ─┼─  spellfn DoTaskInTime(5) 触发收尾：
          │   ⑧ pillar_loop:Remove() → 切到 myfx_haze_pillar_pst
          │   ⑨ haze_particles 调 Disperse() → fade out
          │   ⑩ TheWorld:PushEvent("overridecolourcube", nil)    ◄──  ColourCube 退场
          │   ⑪ haze_zone.GroundCreepEntity:SetRadius(0)         ◄──  GroundCreep 收缩
          │
T+ 6.0s  ─┴─  ⑫ haze_zone:Remove()（清完 emitter）              ◄──  全部清场
                ⑬ pillar_pst animover → 自动 Remove
```

**视觉感受**：玩家会看见——
- 抬手→ 紫色光圈在身边
- 抬手→ 落点出现紫色魔法柱（pre 动画）
- 法柱周围喷出持续紫雾（粒子柱往上）
- 整个画面变成紫调（ColourCube）
- 地面铺出紫色毒雾领域（GroundCreep）
- 5 秒后毒雾散去，画面恢复，地面收缩

#### 为什么是"5 系统"组合而不是 1 系统？

| 视觉元素 | 用 1 个系统能做吗？ | 为什么需要专门系统 |
|---|---|---|
| 落点的"魔法柱"动画 | 一个 FX prefab + animation | ✓ |
| 持续上升的紫雾 | 一个 FX prefab loop | ✗ 单 anim 性能差，**用粒子高效** |
| 全屏紫调 | 改 anim 颜色 / SetMultColour | ✗ **只影响 entity，不影响场景**——必须 ColourCube |
| 地面紫毒雾 | SpawnPrefab 一个贴地 anim | ✗ **不持久 + 不可踩交互**——必须 GroundCreep |
| 释放仪式（手部光圈、光晕） | 自己画 anim | ✗ **完全可以复用 vanilla 的 staffcastfx** |

**这就是"系统组合"的精髓**：每个系统专注一类问题，**组合起来产生立体效果**。

#### 本节学习路径

```
14.6.1 新手  ──────  你将造什么？技能"毒雾领域"全景
14.6.2 新手  ──────  最少代码版（30 行 modmain）
14.6.3 新手  ──────  3 类必备资产清单（anim / tex / sound）
                       ↓
14.6.4 进阶  ──────  设计 5 层 FX 流水线
14.6.5 进阶  ──────  4 个释放阶段的状态转换（pre / cast / sustain / dispel）
14.6.6 进阶  ──────  复用 vanilla 的 staffcastfx + castinglight + staff_pre
                       ↓
14.6.7 老手  ──────  完整代码：modmain + 4 个 prefab + 1 个状态
14.6.8 老手  ──────  性能、网络同步、跨平台陷阱
14.6.9 小结  ──────  14 章总结 + 全章衔接路线图
```

**给三类读者的承诺**：
- **新手**：你将拿到一个**能直接复制运行**的最简法术（Section 14.6.2）——30 行 modmain，调一个 vanilla FX + 一个自定义 GroundCreep emitter，效果可见。
- **进阶**：你将完整理解一个综合法术应该如何**分阶段、分系统、分文件**——避免"全部塞在一个 onspell 函数里"的反模式。
- **老手**：你将看到完整的 modmain.lua + 4 个 prefab + 1 个 state 的可发布工程结构——以及性能、网络、清理的所有细节。

---

### 14.6.1（新手）你将造什么——技能「毒雾领域」全景

#### 法术配置一览

| 字段 | 值 | 说明 |
|---|---|---|
| **法术名** | 毒雾领域（Mystic Haze Dome） | UI 显示 |
| **载体** | 紫色法杖 `myfx_haze_staff` | 手持武器 |
| **释放方式** | AOE 地面点选（reticule） | 鼠标右键拖出范围 |
| **范围** | 半径 5 格 | GroundCreep 半径 |
| **持续** | 5 秒 | timer disperse |
| **冷却** | 法杖耐久 -1 | finiteuses |
| **效果** | 范围内敌人持续中毒 | aura 组件（参考 sporecloud）|
| **视觉** | 紫色光柱 + 持续粒子 + 全屏紫调 + 地面紫雾 | 5 系统组合 |
| **音效** | 释放音 / 持续低频 / 退散音 | 3 段 |

#### 4 个 prefab + 1 个 state 文件结构

```
mymod/
├── modmain.lua                              ◄── 注册资产 / GroundCreep ID / SG state
│
├── prefabs/
│   ├── myfx_haze_staff.lua                  ◄── 主 prefab：法杖（武器 + spellfn）
│   ├── myfx_haze_pillar.lua                 ◄── 落点 FX：pre + loop + pst 三段
│   ├── myfx_haze_particles.lua              ◄── 紫雾粒子柱（VFXEffect）
│   └── myfx_haze_zone.lua                   ◄── GroundCreep emitter
│
├── anim/                                     ◄── Spriter 导出
│   ├── myfx_haze_pillar.zip                 ◄── 3 个动画：pre / loop / pst
│   └── swap_myfx_haze_staff.zip             ◄── 法杖手持外观
│
├── images/
│   └── colour_cubes/
│       └── myfx_haze_purple.tex             ◄── 32×32×32 LUT
│
├── levels/
│   └── textures/
│       ├── myfx_haze_zone.tex               ◄── GroundCreep 主纹理
│       └── noise/myfx_haze_noise.tex        ◄── GroundCreep 噪声
│
└── sound/
    └── mymod_sounds.fsb                     ◄── 释放 / 持续 / 退散
```

#### 5 系统在 4 个 prefab 上的分布

| prefab | 14.1 FX 模式 | 14.2 复用 | 14.3 粒子 | 14.4 CC | 14.5 GC |
|---|---|---|---|---|---|
| `myfx_haze_staff` | - | staffcastfx, staff_castinglight | - | spellfn 内 push event | spellfn 内 SpawnPrefab |
| `myfx_haze_pillar` | ✓ pre+loop+pst | - | - | - | - |
| `myfx_haze_particles` | - | - | ✓ VFXEffect 双 emitter | - | - |
| `myfx_haze_zone` | - | - | - | - | ✓ GroundCreepEntity |

**关键观察**：
- **14.1 FX 模式**只在 `myfx_haze_pillar` 上——它需要 pre / loop / pst 三段动画。
- **14.2 复用**在 `myfx_haze_staff` 的 spellfn 内——直接 SpawnPrefab `staffcastfx` / `staff_castinglight` / `small_puff`。
- **14.3 ParticleEmitter** 独占 `myfx_haze_particles`——粒子是 GPU 系统，独立 prefab 最干净。
- **14.4 / 14.5** 也由法杖触发——但**资源体（cube tex / creep tex）**在资源层注册，prefab 层只是"调用"。

#### 法术的生命周期（一图速看）

```
PROCESS                          OWNER                    DURATION
─────────────────────────────────────────────────────────────────────
[ Right-click on ground ]         Player                   instant
        ↓
ACTION CASTSPELL                  Player                   instant
        ↓
SG: "castspell" state             Player                   ~1.2s
        ↓
spellfn(staff, player)            Staff                    instant
        ↓
SpawnPrefab × 4                   Staff (one-shot)         instant
        ↓
─────────  PARALLEL FX  ─────────────────────────────────────────────
pillar_pre + pillar_loop          myfx_haze_pillar         5s
particles (VFX)                   myfx_haze_particles      5s
overridecolourcube                TheWorld                 5s
GroundCreep zone                  myfx_haze_zone           5s
─────────────────────────────────────────────────────────────────────
        ↓ T+5s
DoTaskInTime(5, dispel)           Staff (callback)
        ↓
pillar_pst + particles fade       FX prefabs               1s
overridecolourcube nil            TheWorld                 instant
zone:SetRadius(0)                 myfx_haze_zone           1s lerp
        ↓ T+6s
all FX prefabs Remove()           garbage collected
```

---

### 14.6.2（新手）最少代码版——30 行 modmain 跑通"释放有特效"

**目标**：让新手能**5 分钟内**复制粘贴跑出一个能用的"释放法术"——**先效果优于先精致**。

#### Step 1：modmain.lua（最简版本）

```lua
-- modmain.lua
PrefabFiles = {
    "myfx_haze_staff",
    "myfx_haze_zone",
}

Assets = {
    Asset("ANIM", "anim/staffs.zip"),  -- 复用 vanilla 法杖外观
}

-- 给玩家添加一个测试用的 console 命令：c_giveall()
```

#### Step 2：prefabs/myfx_haze_staff.lua（最简法杖）

```lua
local function spellfn(staff, target_pos)
    -- ✦ 14.5 GroundCreep ✦
    local zone = SpawnPrefab("myfx_haze_zone")
    zone.Transform:SetPosition(target_pos:Get())
    
    -- ✦ 14.4 ColourCube ✦  
    -- 这里先用 vanilla 的 dusk cube 占位
    -- TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/dusk01_cc.tex")
    
    -- ✦ 5 秒后自动收尾 ✦
    staff:DoTaskInTime(5, function()
        if zone and zone:IsValid() then
            zone.GroundCreepEntity:SetRadius(0)
            zone:DoTaskInTime(2, zone.Remove)
        end
        -- TheWorld:PushEvent("overridecolourcube", nil)
    end)
end

local function onattack(inst, attacker, target)
    -- 法杖近战触发也能释放
    local x, y, z = (target or attacker).Transform:GetWorldPosition()
    spellfn(inst, Vector3(x, 0, z))
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    
    inst.AnimState:SetBank("staffs")
    inst.AnimState:SetBuild("staffs")
    inst.AnimState:PlayAnimation("redstaff")
    inst.AnimState:SetMultColour(1, 0.3, 1, 1)  -- 紫色

    inst:AddTag("weapon")
    
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inventoryitem")
    inst:AddComponent("inspectable")
    inst:AddComponent("equippable")
    inst:AddComponent("weapon")
    inst.components.weapon:SetDamage(0)
    inst.components.weapon:SetOnAttack(onattack)
    
    return inst
end

return Prefab("myfx_haze_staff", fn)
```

#### Step 3：prefabs/myfx_haze_zone.lua（最简 GroundCreep）

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddGroundCreepEntity()
    inst.entity:AddNetwork()
    
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst.GroundCreepEntity:SetRadius(5)
    return inst
end

return Prefab("myfx_haze_zone", fn)
```

#### Step 4：测试

进游戏：
```lua
c_give("myfx_haze_staff")  -- 给自己一根法杖
-- 装备 → 攻击任何目标 → 范围内铺出蛛网（因为 GroundCreep 默认是蛛网纹理）
```

**预期效果**：
- 玩家攻击目标→ 5 秒内目标周围出现蛛网领域
- 5 秒后蛛网开始消退
- 7 秒后完全消失

#### 为什么这就够新手了？

- ✓ 无需自己画 anim（用 vanilla staffs.zip）
- ✓ 无需自己做 colour cube（先注释掉 ColourCube 部分）
- ✓ 无需自己注册 AddGroundCreep（默认复用 WEBCREEP）
- ✓ 无需自己写 SG state（直接挂在 onattack）
- ✓ 一个 modmain + 两个 prefab + 0 张图片 = **跑通**

#### 进阶之路

接下来要做的事，按顺序：
1. **Section 14.6.3**：给法杖配上专属 anim、自定义 cube、自定义 creep 纹理
2. **Section 14.6.5**：把"近战 onattack"换成"AOE 地面点选 + SG castspell state"
3. **Section 14.6.4**：加上 myfx_haze_pillar（pre / loop / pst 三段动画）
4. **Section 14.6.4**：加上 myfx_haze_particles（VFXEffect 粒子柱）
5. **Section 14.6.7**：完整可发布版本

---

### 14.6.3（新手）3 类必备资产清单——anim / tex / sound

#### 资产清单总览

要完整发布"毒雾领域"法术，需要准备的所有资源：

```
┌──────────────────────────────────────────────────────────────────┐
│  ▎ ANIM (Spriter)  ▎                                              │
│   anim/myfx_haze_pillar.zip                                      │
│      ├── animation: myfx_haze_pillar_pre     (~0.5s 升起动画)    │
│      ├── animation: myfx_haze_pillar_loop    (~1s 循环)          │
│      └── animation: myfx_haze_pillar_pst     (~0.5s 消散动画)    │
│                                                                   │
│   anim/swap_myfx_haze_staff.zip                                  │
│      ├── symbol: swap_object                                     │
│      └── animation: idle                                         │
│                                                                   │
│   anim/myfx_haze_staff.zip                                       │
│      ├── symbol: myfx_haze_staff                                 │
│      └── animation: idle (法杖物品图标动画)                       │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  ▎ TEX (静态贴图)  ▎                                              │
│   levels/textures/myfx_haze_zone.tex          ◄── GroundCreep 主层 │
│   levels/textures/myfx_haze_zone.xml          ◄── atlas             │
│   levels/textures/noise/myfx_haze_noise.tex   ◄── 边缘噪声          │
│                                                                   │
│   images/colour_cubes/myfx_haze_purple.tex    ◄── 32×32×32 LUT      │
│   images/colour_cubes/myfx_haze_purple.xml                       │
│                                                                   │
│   images/inventoryimages/myfx_haze_staff.tex  ◄── 物品栏图标        │
│   images/inventoryimages/myfx_haze_staff.xml                     │
│                                                                   │
│   levels/textures/myfx_haze_particles.tex     ◄── 粒子图集          │
│   levels/textures/myfx_haze_particles.xml                        │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  ▎ SOUND  ▎                                                       │
│   sound/mymod_sounds.fsb                                         │
│      ├── event: mymod/haze/cast        (释放音 ~0.8s)             │
│      ├── event: mymod/haze/sustain_LP  (持续音 loop)              │
│      └── event: mymod/haze/dispel      (退散音 ~0.6s)             │
│   sound/mymod_sounds.fev                                         │
└──────────────────────────────────────────────────────────────────┘
```

#### modmain.lua 中的 Assets 注册

```lua
-- modmain.lua
Assets = {
    -- ANIM
    Asset("ANIM", "anim/myfx_haze_pillar.zip"),
    Asset("ANIM", "anim/swap_myfx_haze_staff.zip"),
    Asset("ANIM", "anim/myfx_haze_staff.zip"),
    
    -- 复用 vanilla
    Asset("ANIM", "anim/staffs.zip"),
    
    -- GroundCreep TEX
    Asset("IMAGE", "levels/textures/myfx_haze_zone.tex"),
    Asset("IMAGE", "levels/textures/myfx_haze_zone.xml"),
    Asset("IMAGE", "levels/textures/noise/myfx_haze_noise.tex"),
    
    -- ColourCube LUT
    Asset("IMAGE", "images/colour_cubes/myfx_haze_purple.tex"),
    Asset("IMAGE", "images/colour_cubes/myfx_haze_purple.xml"),
    
    -- 物品栏图标
    Asset("IMAGE", "images/inventoryimages/myfx_haze_staff.tex"),
    Asset("ATLAS", "images/inventoryimages/myfx_haze_staff.xml"),
    Asset("ATLAS_BUILD", "images/inventoryimages/myfx_haze_staff.xml", 256),
    
    -- 粒子贴图
    Asset("IMAGE", "levels/textures/myfx_haze_particles.tex"),
    Asset("IMAGE", "levels/textures/myfx_haze_particles.xml"),
    
    -- SOUND
    Asset("SOUNDPACKAGE", "sound/mymod_sounds.fev"),
    Asset("SOUND", "sound/mymod_sounds.fsb"),
}
```

#### "我没有美术资源怎么办"——降级方案

新手 mod 开发常常遇到的卡点。**逐个降级**：

| 缺什么 | 降级方案 |
|---|---|
| Spriter 动画 | 复用 vanilla 的 `nightsword_curve_fx` 或 `purplegem_fx` 等紫色 FX |
| ColourCube tex | 复用 vanilla 的 `images/colour_cubes/insane_day_cc.tex`（紫红色 already） |
| GroundCreep tex | **不注册自定义 creep**，复用默认 WEBCREEP（蛛网纹理）|
| 粒子图集 | 复用 vanilla 的 `lavaarena_particle.tex` 或 `flame_particle.tex` |
| 音效 | 复用 `dontstarve/wilson/use_gemstaff` / `dontstarve/sanity/gonecrazy_LP` |

**一行降级法术（最低预算版）**：
```lua
-- 资产 0 字节
Assets = { Asset("ANIM", "anim/staffs.zip") }

-- spellfn 内
local function spellfn(staff, pos)
    SpawnPrefab("nightsword_curve_fx").Transform:SetPosition(pos:Get())
    SpawnPrefab("purplegem_fx").Transform:SetPosition(pos:Get())
    
    local zone = SpawnPrefab("myfx_haze_zone")
    zone.Transform:SetPosition(pos:Get())
    
    TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/insane_day_cc.tex")
    
    staff:DoTaskInTime(5, function()
        zone.GroundCreepEntity:SetRadius(0)
        zone:DoTaskInTime(2, zone.Remove)
        TheWorld:PushEvent("overridecolourcube", nil)
    end)
end
```

**视觉效果**：紫色魔剑光波 + 紫宝石粒子 + 蛛网领域 + 紫色全屏调——**完全 0 美术资产，仅 modmain + 一个 prefab**。

---

### 14.6.4（进阶）设计 5 层 FX 流水线——pre / loop / pst + GroundCreep + ColourCube

#### 5 层流水线分解

把"毒雾领域"按时间维度拆成**5 个独立流水线**——每层独立管理 lifecycle，**互不耦合**：

```
┌─ Layer 1: 释放仪式 FX (staffcastfx + castinglight)  [玩家身边，~1s] ─┐
│                                                                       │
│  Player ──┬── SpawnPrefab("staffcastfx")                              │
│           │     └─ entity:SetParent(player.entity)                    │
│           │     └─ animover → Remove                                  │
│           └── SpawnPrefab("staff_castinglight")                       │
│                 └─ Transform:SetPosition(player...)                   │
│                 └─ SetUp(colour, intensity, duration)                 │
└───────────────────────────────────────────────────────────────────────┘

┌─ Layer 2: 落点光柱 FX (myfx_haze_pillar) ──────  [target 位置，5s] ─┐
│                                                                       │
│  pillar_pre   →  pillar_loop  →  pillar_pst                          │
│   ↑(~0.5s)        ↑(loop 5s)        ↑(~0.5s)                          │
│   PlayAnimation   PushAnimation     PushAnimation                     │
│   animover →                       animover →                         │
│     PlayAnim "pillar_loop"           Remove()                         │
└───────────────────────────────────────────────────────────────────────┘

┌─ Layer 3: 持续粒子 (myfx_haze_particles) ────  [target 位置，5s] ─┐
│                                                                       │
│  VFXEffect:                                                           │
│    Emitter[0] = Smoke (粒子寿命 4s, 大颗粒)                           │
│    Emitter[1] = Sparkle (粒子寿命 1s, 小亮点)                         │
│    EmitterManager:AddEmitter(0, fn) periodic emit                     │
│    DoTaskInTime(5, FadeOut)                                          │
│      → 停止 EmitterManager → 已发射粒子自然衰减                       │
│    DoTaskInTime(9, Remove)                                           │
└───────────────────────────────────────────────────────────────────────┘

┌─ Layer 4: 全屏调色 (ColourCube) ──────────────  [TheWorld，5s] ──┐
│                                                                       │
│  T=0:  TheWorld:PushEvent("overridecolourcube",                       │
│                          "images/colour_cubes/myfx_haze_purple.tex")  │
│  T=5:  TheWorld:PushEvent("overridecolourcube", nil)                  │
│        ColourCube 自动 BLEND_TIME=2s lerp 退出                         │
└───────────────────────────────────────────────────────────────────────┘

┌─ Layer 5: 地面领域 (GroundCreep) ────────────  [target 位置，5s+] ──┐
│                                                                       │
│  T=0:  myfx_haze_zone:SpawnPrefab + SetRadius(5)                      │
│  T=5:  zone.GroundCreepEntity:SetRadius(0)                           │
│        ↓ GroundCreep 系统 lerp 收缩 ~1.5s                            │
│  T=7:  zone:Remove()                                                  │
└───────────────────────────────────────────────────────────────────────┘
```

#### 5 层在时间轴上的叠加

```
Time:  T0   T1   T2   T3   T4   T5   T6   T7
       │    │    │    │    │    │    │    │
L1 ────█████░──── (staffcastfx + castinglight)
L2 ────█────████████████████─────█─── (pre / loop / pst)
L3 ────░───███████████████████░─── (粒子 + fade)
L4 ────░──████████████████░─── (cube + blend)
L5 ────░─███████████████░───── (creep + 收缩)
```

**关键观察**：
1. **L1 短**（~1s）—— 玩家手部仪式
2. **L2 / L3 / L4 / L5 同步开始** —— 5 秒主体效果
3. **L2 / L3 / L4 / L5 退场各不同** ——
   - L2 切 pst 动画（视觉过渡）
   - L3 停止 emitter 但已生成粒子 fade
   - L4 PushEvent nil → ColourCube lerp
   - L5 SetRadius(0) → GroundCreep lerp

——这种"**统一开始，分别退场**"是综合 FX 法术的最佳模式。

#### 为什么不要"统一退场"？

新手常见反模式：用一个 `DoTaskInTime(5, ...)` 把所有 5 层一起 Remove：

```lua
-- ✗ 反模式
DoTaskInTime(5, function()
    pillar:Remove()         -- 突然消失
    particles:Remove()      -- 突然消失（粒子不 fade）
    zone:Remove()           -- 突然消失（GroundCreep 残留几秒）
    TheWorld:PushEvent("overridecolourcube", nil)  -- ColourCube 还会 lerp，但其他都消失了
end)
```

**问题**：5 个层"瞬间消失"——视觉上"咔"一下断片——非常突兀。

**正解**：每层有自己的退场节奏：
```lua
DoTaskInTime(5, function()
    pillar.AnimState:PlayAnimation("pillar_pst")  -- 播退场动画
    particles:Disperse()                           -- 调用粒子的退场方法
    zone.GroundCreepEntity:SetRadius(0)            -- 半径回 0，GroundCreep 自动 lerp
    TheWorld:PushEvent("overridecolourcube", nil)  -- CC 自动 lerp
end)

DoTaskInTime(7, function()
    if pillar:IsValid() then pillar:Remove() end
    if particles:IsValid() then particles:Remove() end
    if zone:IsValid() then zone:Remove() end
end)
```

#### 14.1 FX 模式 —— myfx_haze_pillar 完整实现

参考 `scripts/prefabs/sporecloud.lua` 的 pre / loop / pst 模式：

```lua
-- prefabs/myfx_haze_pillar.lua
local assets = {
    Asset("ANIM", "anim/myfx_haze_pillar.zip"),
}

local function OnPreOver(inst)
    inst.AnimState:PlayAnimation("myfx_haze_pillar_loop", true)  -- loop
end

local function OnPstOver(inst)
    inst:Remove()
end

local function StartDisperse(inst)
    inst.AnimState:PlayAnimation("myfx_haze_pillar_pst", false)
    inst:RemoveEventCallback("animover", OnPreOver)
    inst:ListenForEvent("animover", OnPstOver)
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    
    inst:AddTag("FX")
    inst:AddTag("NOCLICK")
    
    inst.AnimState:SetBank("myfx_haze_pillar")
    inst.AnimState:SetBuild("myfx_haze_pillar")
    inst.AnimState:PlayAnimation("myfx_haze_pillar_pre")
    inst.AnimState:SetBloomEffectHandle("shaders/anim.ksh")  -- 自带 bloom

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst.persists = false
    inst:ListenForEvent("animover", OnPreOver)
    
    inst.StartDisperse = StartDisperse
    
    return inst
end

return Prefab("myfx_haze_pillar", fn, assets)
```

**核心点**：
- `pre` → 监听 animover → `OnPreOver` → 切 `loop`
- 业务方调 `inst:StartDisperse()` → 切 `pst` → animover → Remove
- **不需要 timer**——业务方决定何时退场

#### 14.3 ParticleEmitter —— myfx_haze_particles 完整实现

参考 14.3 章 + `scripts/prefabs/torchfire.lua`：

```lua
-- prefabs/myfx_haze_particles.lua
local assets = {
    Asset("IMAGE", "levels/textures/myfx_haze_particles.tex"),
    Asset("IMAGE", "levels/textures/myfx_haze_particles.xml"),
}

local SMOKE_COLOUR_ENVELOPE = "myfx_haze_smoke_colour"
local SMOKE_SCALE_ENVELOPE = "myfx_haze_smoke_scale"

local function InitEnvelopes()
    if EnvelopeManager:GetColourEnvelope(SMOKE_COLOUR_ENVELOPE) == nil then
        EnvelopeManager:AddColourEnvelope(SMOKE_COLOUR_ENVELOPE, {
            { 0,    {0.5, 0.0, 0.8, 0.0} },     -- 淡紫透明
            { 0.3,  {0.7, 0.2, 1.0, 0.7} },     -- 中紫高亮
            { 0.7,  {0.4, 0.0, 0.6, 0.5} },     -- 深紫渐隐
            { 1.0,  {0.2, 0.0, 0.3, 0.0} },     -- 深紫透明
        })
    end
    if EnvelopeManager:GetVector2Envelope(SMOKE_SCALE_ENVELOPE) == nil then
        EnvelopeManager:AddVector2Envelope(SMOKE_SCALE_ENVELOPE, {
            { 0,    {0.5, 0.5} },
            { 1.0,  {2.0, 2.0} },
        })
    end
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddVFXEffect()
    inst.entity:AddNetwork()
    inst:AddTag("FX")
    inst:AddTag("NOCLICK")
    
    InitEnvelopes()
    
    local vfx = inst.VFXEffect
    vfx:InitEmitters(1)  -- 一个 emitter
    vfx:SetRenderResources(0,
        resolvefilepath("levels/textures/myfx_haze_particles.tex"),
        "shaders/vfx_particle.ksh")
    vfx:SetMaxNumParticles(0, 200)
    vfx:SetMaxLifetime(0, 4)
    vfx:SetColourEnvelope(0, SMOKE_COLOUR_ENVELOPE)
    vfx:SetScaleEnvelope(0, SMOKE_SCALE_ENVELOPE)
    vfx:SetBlendMode(0, BLENDMODE.AlphaBlended)
    vfx:EnableBloomPass(0, true)
    
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst.persists = false
    
    -- 持续发射粒子
    inst._emit_task = EmitterManager:AddEmitter(inst, nil, function(inst, ...)
        local pos_x, pos_y, pos_z = math.random() * 1.5 - 0.75, 0, math.random() * 1.5 - 0.75
        local vel_x, vel_y, vel_z = 0, 1.5 + math.random() * 1.5, 0  -- 向上
        local lifetime = 3 + math.random()
        inst.VFXEffect:AddParticle(0, lifetime, pos_x, pos_y, pos_z, vel_x, vel_y, vel_z)
    end, 0.05)  -- 每 0.05s 发射一个
    
    inst.Disperse = function(inst)
        if inst._emit_task ~= nil then
            EmitterManager:RemoveEmitter(inst._emit_task)
            inst._emit_task = nil
        end
        inst:DoTaskInTime(4, inst.Remove)  -- 等已发射粒子自然消逝
    end
    
    return inst
end

return Prefab("myfx_haze_particles", fn, assets)
```

**核心点**：
- 单 emitter，紫色烟雾粒子
- 包络（envelope）控制颜色 / 尺寸渐变
- `Disperse` 方法：停止发射 + 4 秒后 Remove

---

### 14.6.5（进阶）4 个释放阶段——pre / cast / sustain / dispel 状态转换

#### 4 阶段定义

把法术释放分成 4 个清晰阶段：

| 阶段 | 时长 | 玩家可控？ | 视觉重点 | 实现位置 |
|---|---|---|---|---|
| **pre**（蓄力） | ~0.5s | 否（busy）| 抬手 / 法杖光圈 | SG state castspell onenter |
| **cast**（释放点） | 1 frame | 否 | PerformBufferedAction | SG state TimeEvent |
| **sustain**（持续） | 5s | 是 | 全屏紫调 + 粒子 + 领域 | spellfn 内 SpawnPrefab |
| **dispel**（退散） | ~1.5s | 是 | 粒子 fade + 领域收缩 | DoTaskInTime callback |

#### SG state 模板（基于 vanilla castspell 简化）

```lua
-- modmain.lua 内 AddStategraphState
AddStategraphState("wilson", State{
    name = "myfx_haze_castspell",
    tags = { "doing", "busy", "canrotate" },

    onenter = function(inst)
        if inst.components.playercontroller ~= nil then
            inst.components.playercontroller:Enable(false)
        end
        inst.AnimState:PlayAnimation("staff_pre")
        inst.AnimState:PushAnimation("staff", false)
        inst.components.locomotor:Stop()

        --====== Layer 1: 释放仪式 FX ======
        inst.sg.statemem.stafffx = SpawnPrefab(
            inst.components.rider:IsRiding() and "staffcastfx_mount" or "staffcastfx"
        )
        inst.sg.statemem.stafffx.entity:SetParent(inst.entity)
        inst.sg.statemem.stafffx:SetUp({1, 0.3, 1})  -- 紫色

        inst.sg.statemem.stafflight = SpawnPrefab("staff_castinglight")
        inst.sg.statemem.stafflight.Transform:SetPosition(inst.Transform:GetWorldPosition())
        inst.sg.statemem.stafflight:SetUp({1, 0.3, 1}, 1.9, .33)
    end,

    timeline =
    {
        TimeEvent(13 * FRAMES, function(inst)
            inst.SoundEmitter:PlaySound("dontstarve/wilson/use_gemstaff")
        end),
        TimeEvent(53 * FRAMES, function(inst)
            inst.sg.statemem.stafffx = nil
            inst.sg.statemem.stafflight = nil
            inst:PerformBufferedAction()  -- ←─── 触发 sustain 阶段
        end),
        TimeEvent(69 * FRAMES, function(inst)
            inst.sg:RemoveStateTag("busy")
            if inst.components.playercontroller ~= nil then
                inst.components.playercontroller:Enable(true)
            end
        end),
    },

    events =
    {
        EventHandler("animqueueover", function(inst)
            if inst.AnimState:AnimDone() then
                inst.sg:GoToState("idle")
            end
        end),
    },

    onexit = function(inst)
        if inst.components.playercontroller ~= nil then
            inst.components.playercontroller:Enable(true)
        end
        if inst.sg.statemem.stafffx ~= nil and inst.sg.statemem.stafffx:IsValid() then
            inst.sg.statemem.stafffx:Remove()
        end
        if inst.sg.statemem.stafflight ~= nil and inst.sg.statemem.stafflight:IsValid() then
            inst.sg.statemem.stafflight:Remove()
        end
    end,
})
```

#### 引用 vanilla castspell state 源码

```15815:15903:scripts/stategraphs/SGwilson.lua
        name = "castspell",
        tags = { "doing", "busy", "canrotate" },

        onenter = function(inst)
            if inst.components.playercontroller ~= nil then
                inst.components.playercontroller:Enable(false)
            end
            inst.AnimState:PlayAnimation("staff_pre")
            inst.AnimState:PushAnimation("staff", false)
            inst.components.locomotor:Stop()

            --Spawn an effect on the player's location
            local staff = inst.components.inventory:GetEquippedItem(EQUIPSLOTS.HANDS)
            local colour = staff ~= nil and staff.fxcolour or { 1, 1, 1 }

            inst.sg.statemem.stafffx = SpawnPrefab(inst.components.rider:IsRiding() and "staffcastfx_mount" or "staffcastfx")
            inst.sg.statemem.stafffx.entity:SetParent(inst.entity)
            inst.sg.statemem.stafffx:SetUp(colour)

            inst.sg.statemem.stafflight = SpawnPrefab("staff_castinglight")
            inst.sg.statemem.stafflight.Transform:SetPosition(inst.Transform:GetWorldPosition())
            inst.sg.statemem.stafflight:SetUp(colour, 1.9, .33)
            ...
        end,
```

——可以看到，**仅复用 vanilla castspell state**就能拿到完整的"举手 + 光圈 + 光晕 + 音效"释放仪式。**不需要重新写 state**——只需要让你的法杖**走 castspell action 路径**就能复用整套。

#### 让法杖走 castspell action

```lua
-- prefabs/myfx_haze_staff.lua（核心配置）
local function spellfn(staff, target_pos, caster)
    --====== Layer 2 & 3 ======
    local pillar = SpawnPrefab("myfx_haze_pillar")
    pillar.Transform:SetPosition(target_pos:Get())
    
    local particles = SpawnPrefab("myfx_haze_particles")
    particles.Transform:SetPosition(target_pos:Get())
    
    --====== Layer 4 ======
    TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/myfx_haze_purple.tex")
    
    --====== Layer 5 ======
    local zone = SpawnPrefab("myfx_haze_zone")
    zone.Transform:SetPosition(target_pos:Get())
    
    --====== dispel 5s 后 ======
    staff:DoTaskInTime(5, function()
        if pillar and pillar:IsValid() then pillar:StartDisperse() end
        if particles and particles:IsValid() then particles:Disperse() end
        if zone and zone:IsValid() then 
            zone.GroundCreepEntity:SetRadius(0)
            zone:DoTaskInTime(2, zone.Remove) 
        end
        TheWorld:PushEvent("overridecolourcube", nil)
    end)
end
```

**关键设计**：spellfn 是**法杖的方法**——它由 **SG state 在 PerformBufferedAction 时调用**——这样就**完全复用了 vanilla 的 castspell state**！

#### action handler 注册

```lua
-- modmain.lua
local CASTSPELL = GLOBAL.ACTIONS.CASTSPELL

AddComponentAction("USEITEM", "spellbook", function(inst, doer, actions, right)
    if inst.prefab == "myfx_haze_staff" and right then
        table.insert(actions, CASTSPELL)
    end
end)
```

**或者**用更标准的 `AddPrefabPostInit` + `aoetargeting` 组件：
```lua
-- 在 prefabs/myfx_haze_staff.lua 内
inst:AddComponent("aoetargeting")
inst.components.aoetargeting:SetSpellFn(spellfn)
inst.components.aoetargeting:SetTargetFXAtMouse(true)
inst.components.aoetargeting:SetRange(8)
```

---

### 14.6.6（进阶）复用 vanilla 资源——staffcastfx + castinglight + staff_pre

#### 哪些 vanilla 资源可以"零成本"复用？

| Vanilla 资源 | 你的法术能用 | 说明 |
|---|---|---|
| `staffcastfx` | ✓ | 法杖端的光圈，14.2 索引 |
| `staffcastfx_mount` | ✓ | 骑乘版 |
| `staff_castinglight` | ✓ | 玩家脚下的光晕（带颜色参数）|
| `cointosscastfx` | 偶尔 | 投硬币法术专用，但可以复用作"硬币爆裂" |
| `pocketwatch_cast_fx` | 偶尔 | 怀表施法专用 |
| anim `staffs.zip` 中的 `staff_pre` / `staff` | ✓ | 玩家抬手举杖动画 |
| anim `staffs.zip` 中的 `redstaff` 等 | ✓ | 法杖物品手持外观（改 SetMultColour 即可换色）|
| sound `dontstarve/wilson/use_gemstaff` | ✓ | 紫宝石法杖音效 |
| sound `dontstarve/wilson/fireball_explo` | ✓ | 火球爆炸音效 |
| sound `dontstarve/sanity/gonecrazy_LP` | ✓ | 精神错乱循环音（毒雾持续音）|
| `images/colour_cubes/insane_day_cc.tex` | ✓ | 紫红色全屏调（可作毒雾占位）|
| `images/colour_cubes/lunacy_regular_cc.tex` | ✓ | 月色蓝调（可作冰雪法术占位）|
| `levels/textures/web.tex` (默认 GroundCreep) | ✓ | 蛛网纹理（默认）|
| FX prefab `purplegem_fx` | ✓ | 紫宝石碎裂粒子 |
| FX prefab `nightsword_curve_fx` | ✓ | 紫色魔剑光波 |

#### 例：用 vanilla 资源拼装的"满配版"

完全不用自己画图、做音效——纯 vanilla 资源的法术：

```lua
local function spellfn(staff, target_pos, caster)
    -- ✦ Layer 1: 释放仪式 ✦ —— vanilla castspell state 里已自动产生
    
    -- ✦ Layer 2: 落点光波 ✦
    local fx1 = SpawnPrefab("nightsword_curve_fx")
    fx1.Transform:SetPosition(target_pos:Get())
    
    -- ✦ 落点闪光 ✦
    local fx2 = SpawnPrefab("purplegem_fx")
    fx2.Transform:SetPosition(target_pos:Get())
    
    -- ✦ Layer 3: 持续粒子 (这里复用 sporecloud 配色，不写自己的 VFX)
    -- 实际上 sporecloud 是绿色——要紫色还是要做自己的 myfx_haze_particles
    
    -- ✦ Layer 4: 全屏紫调 ✦
    TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/insane_day_cc.tex")
    
    -- ✦ Layer 5: 蛛网领域 (默认 WEBCREEP，不需要注册自定义 GroundCreep) ✦
    local zone = SpawnPrefab("myfx_haze_zone")
    zone.Transform:SetPosition(target_pos:Get())
    
    -- ✦ 持续音 ✦
    caster.SoundEmitter:PlaySound("dontstarve/sanity/gonecrazy_LP", "haze_loop")
    
    -- 5s 后 dispel
    staff:DoTaskInTime(5, function()
        if zone:IsValid() then 
            zone.GroundCreepEntity:SetRadius(0)
            zone:DoTaskInTime(2, zone.Remove) 
        end
        TheWorld:PushEvent("overridecolourcube", nil)
        if caster:IsValid() then
            caster.SoundEmitter:KillSound("haze_loop")
            caster.SoundEmitter:PlaySound("dontstarve/wilson/fireball_explo")  -- 退散音
        end
    end)
end
```

**视觉效果**：
- 玩家抬手 → 紫光圈（vanilla staffcastfx）
- 落点 → 紫色魔剑光波（nightsword_curve_fx）+ 紫宝石碎裂（purplegem_fx）
- 全屏 → 紫色调（insane_day_cc）
- 地面 → 蛛网领域
- 持续 → 精神错乱低吟音
- 5s 后 → 火球爆炸音 → 全部退场

**资产成本**：**0 字节自创**——全是 vanilla。

#### 调色：SetUp(colour) 的妙用

`staffcastfx:SetUp(colour)` 内部就是 `inst.AnimState:SetMultColour(colour[1], colour[2], colour[3], 1)`：

```23:25:scripts/prefabs/staffcastfx.lua
local function SetUp(inst, colour)
    inst.AnimState:SetMultColour(colour[1], colour[2], colour[3], 1)
end
```

**给 mod 启示**：你可以用同一个 staffcastfx prefab，**任意调色**：
```lua
fx:SetUp({1, 0.3, 1})    -- 紫色（毒雾法术）
fx:SetUp({0.3, 0.5, 1})   -- 蓝色（冰雪法术）
fx:SetUp({1, 0.7, 0.2})   -- 金色（神圣法术）
fx:SetUp({0.5, 1, 0.3})   -- 绿色（自然法术）
```

**完全无需画 5 个不同颜色的法杖光圈 anim**——一个 anim + 5 种调色 = 5 种法术视觉。

---

### 14.6.7（老手）完整代码——modmain + 4 prefab + 1 state

#### 完整文件结构

```
mymod/
├── modinfo.lua
├── modmain.lua
├── modicon.tex / modicon.xml
├── scripts/
│   └── prefabs/
│       ├── myfx_haze_staff.lua           # 法杖
│       ├── myfx_haze_pillar.lua          # 落点光柱（pre/loop/pst）
│       ├── myfx_haze_particles.lua       # 紫雾粒子
│       └── myfx_haze_zone.lua            # GroundCreep emitter
├── anim/...
├── images/...
└── levels/textures/...
```

#### modmain.lua（完整版）

```lua
PrefabFiles = {
    "myfx_haze_staff",
    "myfx_haze_pillar",
    "myfx_haze_particles",
    "myfx_haze_zone",
}

Assets = {
    Asset("ANIM", "anim/myfx_haze_pillar.zip"),
    Asset("ANIM", "anim/staffs.zip"),
    Asset("IMAGE", "levels/textures/myfx_haze_zone.tex"),
    Asset("IMAGE", "levels/textures/myfx_haze_zone.xml"),
    Asset("IMAGE", "levels/textures/noise/myfx_haze_noise.tex"),
    Asset("IMAGE", "images/colour_cubes/myfx_haze_purple.tex"),
    Asset("IMAGE", "images/colour_cubes/myfx_haze_purple.xml"),
    Asset("IMAGE", "levels/textures/myfx_haze_particles.tex"),
    Asset("IMAGE", "levels/textures/myfx_haze_particles.xml"),
    Asset("IMAGE", "images/inventoryimages/myfx_haze_staff.tex"),
    Asset("ATLAS", "images/inventoryimages/myfx_haze_staff.xml"),
}

GLOBAL.STRINGS.NAMES.MYFX_HAZE_STAFF = "毒雾法杖"
GLOBAL.STRINGS.RECIPE_DESC.MYFX_HAZE_STAFF = "召唤紫色毒雾领域。"
GLOBAL.STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYFX_HAZE_STAFF = "灵魂的低语...在召唤！"

-- 注册自定义 GroundCreep id（可选）
local TileManager = GLOBAL.require("tilemanager")
GLOBAL.GROUND_CREEP_IDS.MYFX_HAZE = 2
GLOBAL.mod_protect_TileManager = false
TileManager.AddGroundCreep(GLOBAL.GROUND_CREEP_IDS.MYFX_HAZE, {
    name = "myfx_haze_zone",
    noise_texture = "myfx_haze_noise",
})
GLOBAL.mod_protect_TileManager = true
```

#### prefabs/myfx_haze_staff.lua（完整法杖）

```lua
local assets = {
    Asset("ANIM", "anim/staffs.zip"),
    Asset("ATLAS", "images/inventoryimages/myfx_haze_staff.xml"),
    Asset("IMAGE", "images/inventoryimages/myfx_haze_staff.tex"),
}

local prefabs = {
    "staffcastfx",
    "staff_castinglight",
    "myfx_haze_pillar",
    "myfx_haze_particles",
    "myfx_haze_zone",
}

local FX_COLOUR = {1, 0.3, 1, 1}  -- 紫色

local function spellfn(staff, target_pos, caster)
    if target_pos == nil then
        local x, y, z = caster.Transform:GetWorldPosition()
        target_pos = Vector3(x, 0, z)
    end
    
    --====== Layer 2: 落点光柱 ======
    local pillar = SpawnPrefab("myfx_haze_pillar")
    pillar.Transform:SetPosition(target_pos:Get())
    pillar.AnimState:SetMultColour(FX_COLOUR[1], FX_COLOUR[2], FX_COLOUR[3], 1)
    
    --====== Layer 3: 持续粒子 ======
    local particles = SpawnPrefab("myfx_haze_particles")
    particles.Transform:SetPosition(target_pos:Get())
    
    --====== Layer 4: 全屏调色 ======
    TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/myfx_haze_purple.tex")
    
    --====== Layer 5: 地面领域 ======
    local zone = SpawnPrefab("myfx_haze_zone")
    zone.Transform:SetPosition(target_pos:Get())
    
    --====== 音效 ======
    caster.SoundEmitter:PlaySound("dontstarve/sanity/gonecrazy_LP", "haze_loop_" .. tostring(caster.GUID))
    
    --====== 5 秒后 dispel ======
    staff:DoTaskInTime(5, function()
        if pillar and pillar:IsValid() then
            pillar:StartDisperse()  -- pillar 自管退场
        end
        if particles and particles:IsValid() then
            particles:Disperse()  -- particles 自管退场
        end
        if zone and zone:IsValid() then
            zone.GroundCreepEntity:SetRadius(0)
            zone:DoTaskInTime(2, function() 
                if zone:IsValid() then zone:Remove() end 
            end)
        end
        TheWorld:PushEvent("overridecolourcube", nil)
        
        if caster and caster:IsValid() then
            caster.SoundEmitter:KillSound("haze_loop_" .. tostring(caster.GUID))
            caster.SoundEmitter:PlaySound("dontstarve/wilson/fireball_explo")
        end
        
        --====== 法杖耗损 ======
        if staff:IsValid() and staff.components.finiteuses then
            staff.components.finiteuses:Use(1)
        end
    end)
end

local function ReticuleTargetFn()
    local player = ThePlayer
    local pos = Vector3()
    for r = 8, 4, -.25 do
        pos.x, pos.y, pos.z = player.entity:LocalToWorldSpace(r, 0, 0)
        if TheWorld.Map:IsPassableAtPoint(pos:Get()) and 
           not TheWorld.Map:IsGroundTargetBlocked(pos) then
            return pos
        end
    end
    return pos
end

local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_object", "staffs", "redstaff")
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end

local function onunequip(inst, owner)
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)
    
    inst.AnimState:SetBank("staffs")
    inst.AnimState:SetBuild("staffs")
    inst.AnimState:PlayAnimation("redstaff")
    inst.AnimState:SetMultColour(FX_COLOUR[1], FX_COLOUR[2], FX_COLOUR[3], 1)
    
    inst:AddTag("aoeweapon_lunge")  -- 触发 castspell action
    inst:AddTag("rangedlighter")
    inst:AddTag("staff")
    inst.fxcolour = FX_COLOUR
    inst.castsound = "dontstarve/wilson/use_gemstaff"
    
    inst:AddComponent("reticule")
    inst.components.reticule.targetfn = ReticuleTargetFn
    inst.components.reticule.ease = true

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inventoryitem")
    inst.components.inventoryitem.atlasname = "images/inventoryimages/myfx_haze_staff.xml"
    
    inst:AddComponent("inspectable")
    
    inst:AddComponent("equippable")
    inst.components.equippable:SetOnEquip(onequip)
    inst.components.equippable:SetOnUnequip(onunequip)
    
    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(20)
    inst.components.finiteuses:SetUses(20)
    inst.components.finiteuses:SetOnFinished(inst.Remove)
    
    inst:AddComponent("aoespell")
    inst.components.aoespell:SetSpellFn(spellfn)
    
    return inst
end

return Prefab("myfx_haze_staff", fn, assets, prefabs)
```

#### prefabs/myfx_haze_pillar.lua

（已在 14.6.4 完整给出）

#### prefabs/myfx_haze_particles.lua

（已在 14.6.4 完整给出）

#### prefabs/myfx_haze_zone.lua

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddGroundCreepEntity()
    inst.entity:AddNetwork()

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst.GroundCreepEntity:SetRadius(5)
    inst.persists = false  -- 不存档（法术效果，不该持久化）
    
    -- 触发 creepactivate 给踩入领域的玩家造成减速 / 减益
    inst:ListenForEvent("creepactivate", function(inst, data)
        if data.target and data.target.components.debuffable then
            data.target.components.debuffable:AddDebuff("hazedebuff", "hazedebuff")
        end
    end)
    
    return inst
end

return Prefab("myfx_haze_zone", fn)
```

#### 总结：5 个文件干了什么

| 文件 | 行数（参考）| 负责什么 |
|---|---|---|
| modmain.lua | ~30 | 注册资产 / 注册自定义 creep id |
| myfx_haze_staff.lua | ~120 | 法杖（aoespell + spellfn）|
| myfx_haze_pillar.lua | ~50 | pre/loop/pst 落点光柱 |
| myfx_haze_particles.lua | ~70 | VFXEffect 紫雾粒子柱 |
| myfx_haze_zone.lua | ~25 | GroundCreep 领域 |

**总代码量**：约 **295 行**——一个完整、可发布、5 系统综合的法术 mod。

---

### 14.6.8（老手）性能、网络同步、跨平台陷阱

#### 性能陷阱

##### 陷阱 1：粒子数量过多

```lua
-- ✗ 错误：每帧发射 10 个粒子，5s 内累计 1500 个，FPS 暴跌
EmitterManager:AddEmitter(inst, nil, function(inst)
    for i = 1, 10 do
        inst.VFXEffect:AddParticle(0, 5, math.random(), 0, math.random(), 0, 1, 0)
    end
end, FRAMES)  -- 每帧

-- ✓ 正确：每 0.05s（3 帧）发射 1 个粒子
EmitterManager:AddEmitter(inst, nil, function(inst)
    inst.VFXEffect:AddParticle(0, 4, math.random(), 0, math.random(), 0, 1, 0)
end, 0.05)
```

##### 陷阱 2：未限制 SetMaxNumParticles

```lua
-- ✗ 没限制 → 默认 100 → 法术全开后立刻溢出
inst.VFXEffect:SetMaxNumParticles(0, 100)

-- ✓ 根据 emit 速率 + lifetime 算出上限
-- emit 速率 0.05s/个 × lifetime 4s = 80 个，预留 ×1.5 = 120
inst.VFXEffect:SetMaxNumParticles(0, 120)
```

##### 陷阱 3：法术叠加导致 GroundCreep 渲染层堆积

10 个玩家同时在地图各处释放毒雾领域 → 10 个 myfx_haze_zone emitter → GroundCreep 系统 OK，但 vanilla 引擎对 emitter 数量有上限，可能漏 render。

**workaround**：mod 自己做"领域合并"——同一玩家短时间内多次施法，先 SetRadius(0) 老的再创建新的。

#### 网络同步陷阱

##### 陷阱 4：客户端没看到 GroundCreep 出现

```lua
-- ✗ 错误：客户端只见 prefab，没看见铺创建的 creep（网络同步未完成）
local zone = SpawnPrefab("myfx_haze_zone")
zone.Transform:SetPosition(x, 0, z)
zone.GroundCreepEntity:SetRadius(5)  -- 客户端调 → 错
```

**正解**：所有 SetRadius 必须在服务器调（在 prefab 的 fn 内的 server 段，14.5.2 已强调）。

##### 陷阱 5：ColourCube 不同步

`overridecolourcube` 是 PushEvent 给 TheWorld——只在**调用 push 的那一端**生效。

- 服务器调 → 服务器生效，客户端不变
- 客户端调 → 客户端生效，服务器不变（dedicated server 没 PostProcessor 也无所谓）

**正解**：通常应在**服务器**调，让 TheWorld 同步——但 ColourCube 本身是**客户端渲染**——所以**实际需要给所有客户端各自 push**：

```lua
-- 服务器端调 → 通过 net_event 同步给客户端
local function spellfn(staff, target_pos, caster)
    -- ...
    -- 发广播事件给所有玩家
    for _, player in ipairs(AllPlayers) do
        if player and player:IsValid() then
            player:PushEvent("haze_cc_start")
        end
    end
end

-- 客户端 AddPlayerPostInit
AddPlayerPostInit(function(inst)
    inst:ListenForEvent("haze_cc_start", function()
        TheWorld:PushEvent("overridecolourcube", "images/colour_cubes/myfx_haze_purple.tex")
        inst:DoTaskInTime(5, function()
            TheWorld:PushEvent("overridecolourcube", nil)
        end)
    end)
end)
```

##### 陷阱 6：复用 staff_castinglight 的 Light 组件没同步

`Light` 组件是**服务器权威**，客户端读取——SetUp(colour) 内部调 `Light:SetColour` 会自动同步。但如果你**写了自定义的 light fx**——要确保在服务器端调用 `Light:Enable / SetIntensity`。

#### 跨平台陷阱

##### 陷阱 7：dedicated server 没 PostProcessor

dedicated server 没有 PostProcessor 全局对象——但好消息是**postprocesseffects.lua 已经做了空函数兜底**（14.4.7 老手节讲过），所以服务器调 `PostProcessor:SetColourCubeData` 不会崩——但**也不会有效果**——这就是为什么应该用**客户端 PushEvent**而不是直接 PostProcessor。

##### 陷阱 8：iOS / 主机平台粒子上限更严格

console 平台 `SetMaxNumParticles` 上限可能是 50（对比 PC 200）——超出会被 silent drop。

**正解**：考虑平台兼容时取下限（30~50）。

##### 陷阱 9：低端机器自动关闭 PostProcessing

玩家在 Settings → Graphics → 关闭 Postprocessor → ColourCube 会**完全失效**——但**法术效果不应该崩**。

**正解**：mod 应该**用 ColourCube 做"额外氛围"**而非"必须效果"——L2-L5 还在，只是 L4 缺失——视觉降级而不是 broken。

#### 网络生命周期陷阱

##### 陷阱 10：玩家断线后 ongoing 法术泄漏

玩家施法 → DoTaskInTime(5, ...) → 玩家断线 → DoTaskInTime 仍然在 staff（item）上执行——但 caster 引用已失效。

**正解**：所有 closure 内访问 caster / pillar 都加 `:IsValid()` 检查（已在 spellfn 中体现）。

##### 陷阱 11：法术分阶段死亡导致 zombie effect

5s 内玩家 ng+ → world 重启 → ColourCube 被新 World 重置 → GroundCreep 也重置——但 caster 的 DoTaskInTime 可能仍然 fire（在新 world 上 push 一个 nil cube）。

**正解**：用 timer 组件代替 DoTaskInTime——timer 在 OnRemove / world 切换时自动清理。

---

### 14.6.9（小结）14 章总结 + 整本教程衔接路线图

#### 14.6 实战速查表

| 文件 | 责任 | 引用系统 |
|---|---|---|
| modmain.lua | 注册资产 + 注册自定义 creep | 全部 |
| myfx_haze_staff | 法杖 + spellfn | 14.2 复用 + 触发 |
| myfx_haze_pillar | 落点光柱 | 14.1 (pre/loop/pst) |
| myfx_haze_particles | 持续粒子 | 14.3 (ParticleEmitter) |
| myfx_haze_zone | 持续领域 | 14.5 (GroundCreep) |
| ColourCube tex | 全屏调色 | 14.4 (LUT) |

#### 5 系统组合检查清单

发布前自检：

- [ ] **14.1 FX 模式**：pre/loop/pst 三段是否齐全？loop 由 animover 自动接 pre 后？pst 由业务方调 StartDisperse 启动？
- [ ] **14.2 复用**：staffcastfx / staff_castinglight 是否被复用？SetUp(colour) 是否调过？
- [ ] **14.3 粒子**：MaxNumParticles / MaxLifetime 是否合理？ColourEnvelope / ScaleEnvelope 是否定义？BlendMode 是否合适？
- [ ] **14.4 ColourCube**：override 时是否记得 push nil？lut tex 文件是否注册到 Assets？dedicated server 兜底？
- [ ] **14.5 GroundCreep**：SetRadius 是否在 server 段？SetRadius(0) 后是否 DoTaskInTime 删除？noise_texture 是否注册？

#### 14 章整体回顾

```
14.1 FX Prefab 设计模式
     ├── SpawnPrefab + 自动回收 (animover, persists=false)
     └── pre/loop/pst 三段式

14.2 常用 FX Prefab 索引
     ├── 449 个 vanilla FX 速查
     └── small_puff / sparks / splash / electricchargedfx 等

14.3 ParticleEmitter
     ├── VFXEffect GPU 粒子
     ├── EnvelopeManager (色 / 尺寸渐变)
     └── EmitterManager (持续发射)

14.4 ColourCube
     ├── 32×32×32 LUT 全屏调色
     ├── 3 通道叠加 (Ambient / Insanity / Lunacy)
     └── PostProcessor 全家桶 (Bloom / Distort / Lunacy 等)

14.5 GroundCreep / GroundTiles
     ├── 地图覆盖层（持久 + 可踩 + 存档）
     ├── locomotor 联动 + creepactivate 事件链
     └── AddTile / AddGroundCreep / AddFalloffTexture

14.6 实战 (本节)
     └── 5 系统组合的"毒雾领域"自定义法术
```

**5 节的关系**：
- 14.1 / 14.2 是**基础**——所有 FX 都要懂
- 14.3 是**进阶**——做高视觉密度效果时上
- 14.4 / 14.5 是**专精**——前者全屏后处理、后者地图持久层
- 14.6 是**整合**——综合实战

#### 整本教程衔接路线图

> **本章地位**：第 14 章是 mod 教程的"视觉效果总章"——把 prefab、stategraph、动画、粒子、后处理、地图层全部串起来。

```
第 1-3 章   [基础]      lua / 项目结构 / modinfo / modmain
第 4-6 章   [资源]      anim / sound / image / 物品创建
第 7-9 章   [行为]      component / brain / stategraph
第 10 章    [事件]      event 系统
第 11 章    [角色]      自定义角色 / 11.6 自定义动作 ★
第 12 章    [战斗]      武器 / 伤害 / combat
第 13 章    [世界]      worldgen / 怪物刷新 ★
第 14 章    [视觉]      FX / 粒子 / ColourCube / GroundCreep ★ (本章)
第 15 章    [发布]      modicon / steam workshop / 兼容性
```

**本章在整本中**：
- 前置依赖：第 11 章（StateGraph）、第 7-9 章（component / animation）
- 后续：第 15 章发布前应回到本章检查 5 系统是否完整、性能是否达标

**本章学完后，你能做**：
- ✓ 任意 mod 物品 / 怪物的释放特效（14.6 实战可直接迁移）
- ✓ 自定义角色专属技能的视觉表现
- ✓ Boss 战的全屏氛围切换（14.4 ColourCube）
- ✓ 自定义生物群系的地面贴图（14.5 GroundTiles）
- ✓ 任意复杂的粒子系统（14.3 ParticleEmitter）

#### 推荐进阶练习

1. **复刻一个 vanilla 法术**：从 firestaff 或 icestaff 入手，分析它用了哪些系统。
2. **造 1 个全新法术**：参考 14.6 实战，但**换主题**——比如"治疗领域"（绿色）、"冰雪领域"（蓝色）、"雷电领域"（黄色）。
3. **自定义角色技能**：把 14.6 法术绑定到自定义角色的特殊键。
4. **Boss 战氛围**：在 worldgen 设定下，让 Boss 出场触发 ColourCube 切换+ 全屏粒子+ 地面 GroundCreep。

#### 致 mod 开发者

> 视觉效果是玩家**对你 mod 的第一印象**——一个"释放有反馈"的法术——
> 即使它在数据层面只是 1 点伤害——也比一个**没有任何 FX 的 100 点伤害法术****更让玩家觉得有重量感**。
> 14 章的所有内容——本质上是**为"重量感"服务**——多花一行代码 SpawnPrefab，就多一份玩家粘性。

---




