# 第13章 动画与美术资源

## 13.1 资源管线总览：.tex / .xml / build.bin / anim.bin 的关系

### 本节导读

前面 12 章我们一直在讨论"**逻辑**"——组件、动作、行为树、网络同步。但是玩家在屏幕上看到的从来不是逻辑，是**像素**。Wilson 挥动斧头时屏幕上出现的那两帧动画、库存格里那张 64×64 的小图标、地图上那个绿色三角形——这些"看得见的东西"从硬盘里的某个文件被搬运到 GPU 显存的整个旅程，就是本章要讲的"**美术资源管线**"。

而 13.1 是这条管线的**鸟瞰图**：

> **新手**从 13.1.1-13.1.3 起步——理解"为什么不直接用 PNG"、看懂一个 mod 的 anim 目录里到底有什么、记住 4 种最常见的资源文件后缀；**进阶读者**继续看 13.1.4-13.1.6，深入 `Asset` 构造函数、12 种资源类型的差异、`RegisterPrefabsImpl` → `resolvefilepath` 的完整加载链路；**老手**跳到 13.1.7-13.1.8，逐行拆解 `softresolvefilepath_internal` 的 mod 路径搜索算法、动态资源（`.dyn`）的延迟加载机制、八个最容易踩的坑。

读完本节，你不一定能立刻做出一只全新的猪人模型——但是**你看到任何 mod 报错"Could not find anim/xxx.zip"时，立刻能知道去哪里找问题**。

---

### 13.1.1 快速入门：为什么饥荒不直接用 PNG？

#### 第一步：打开一个 mod 的 anim 文件夹

随便打开一个 mod 的 `anim/` 目录，比如 `mods/联机版mod/神话未加密/anim/`，你会看到一堆 `.zip` 文件：

```
myth_redlantern.zip
myth_well.zip
myth_food_table.zip
swap_dyc_swords.zip
...
```

**每一个 .zip 都是一份"完整的角色形象包"**——里面有 build（皮肤）、有 anim（动作）、有所有需要的图片。

如果你直接把 `myth_redlantern.zip` 用解压软件打开，会看到这种结构：

```
myth_redlantern.zip
├── anim.bin          ← 动作数据（关键帧、骨骼变换）
├── build.bin         ← 骨架数据（哪些图组成模型、缩放、锚点）
├── atlas-0.tex       ← 真正的图像数据（所有图被打包成一张大图）
└── atlas-0.xml       ← 描述 atlas-0.tex 中每个小图的位置
```

这就是 Spriter 编译后的"四大文件"——**.tex、.xml、build.bin、anim.bin** 就是本节要讲的核心。

#### 第二步：为什么不能直接用 PNG？

新手的第一反应是："直接给我 PNG，我让代码 `LoadImage("xxx.png")` 不就完了？" 工业级游戏引擎不这么干，原因有三：

**原因一：性能——批次合并（Batching）**

GPU 每渲染一张图都要"切换状态"——更换纹理、更换材质、提交一次绘制调用。如果 Wilson 身上 20 个部件分别是 20 张独立 PNG，**每帧要 20 次 DrawCall**。但是把 20 个部件打包到一张大图（atlas）里，**一次 DrawCall 就能画完**。这就是为什么饥荒的 `.tex` 文件实际上是"图集"（atlas），里面拼着大量小图。

**原因二：体积——压缩格式**

PNG 是无损压缩的 RGBA 图，硬盘占用大、加载慢。`.tex` 是 Klei 自定义的纹理容器，**底层用的是 GPU 直接支持的 DXT5 / BC3 压缩格式**——不需要 CPU 解码，文件读到内存可以直接送去 GPU。一个角色的 PNG 总和可能 8MB，转成 `.tex` 之后只剩 1MB。

**原因三：动画——骨骼+蒙皮**

Wilson 不是"一帧一帧画"出来的——这种逐帧动画占空间、不灵活。Wilson 是**骨骼动画**：身体由"头、躯干、左臂、右臂、左腿、右腿、武器……"等几十个部件（"Symbol"）组成，动画数据只描述**每一帧每个部件的位移、旋转、缩放**。这样：

- 一个建模能复用上百个动作（idle、walk、attack、cast、die……）
- 换装时只要"用别的图替换某个 Symbol"——比如把 `swap_object` 替换成斧头或剑（这就是 13.5 要讲的 `OverrideSymbol`）

骨骼数据存在 `build.bin`，动作数据存在 `anim.bin`。它们必须是**二进制紧凑格式**才能高效解析——所以是 `.bin` 而不是 JSON。

> **新手记忆**：饥荒不用 PNG 是为了**速度**（DXT 压缩 + atlas 合批）、**体积**（压缩纹理）、**灵活**（骨骼动画 + 部件替换）。每一个 `.zip` 就是一个"角色/物品的完整美术包"。

---

### 13.1.2 快速入门：四种核心文件的"职责分工"

在 `anim/xxx.zip` 解压出的四个文件，每一个都有独立的职责。可以用一个**人偶剧团**的比喻来记：

| 文件 | 比喻 | 真实作用 |
|------|------|---------|
| `atlas-0.tex` | **道具箱**——里面塞着所有要用的小图 | 二进制纹理图集，DXT 压缩，可以直接送 GPU |
| `atlas-0.xml` | **道具清单**——记录道具箱里每件东西放在哪个抽屉 | 文本 XML，描述每个 Symbol 在 .tex 中的 UV 矩形坐标 |
| `build.bin` | **木偶骨架图纸**——告诉你"头接在脖子上、手接在肩膀上" | 二进制建模数据，包含每个 Symbol 的 frame 数量、锚点、缩放、所属 atlas 索引 |
| `anim.bin` | **剧本**——"第 1 秒头转向左、第 2 秒手抬起" | 二进制动作数据，每个 anim 的关键帧、变换矩阵、事件回调点 |

**每一帧的渲染流程**（简化版）：

1. 引擎读 `anim.bin` —— 决定当前帧每个 Symbol 在哪、转多少、缩放多少
2. 引擎读 `build.bin` —— 知道每个 Symbol 用 atlas 里的哪个矩形
3. 引擎读 `atlas-0.xml` —— 拿到那个矩形的 UV 坐标
4. 引擎从 `atlas-0.tex` 中**裁剪**出那个矩形的像素送 GPU
5. GPU 把像素**贴**到相应位置，完成一个部件

把所有部件叠加，就是你看到的那一帧 Wilson。

#### 第二步：为什么有时候是 atlas-0.tex 而不只一张？

如果一个角色非常复杂（比如某些 boss、皮肤），单张 atlas 不够装，编译器会自动拆成 `atlas-0.tex` / `atlas-1.tex` / ……，对应 `atlas-0.xml` / `atlas-1.xml` / ……。`build.bin` 里会记录每个 Symbol 属于哪个 atlas 编号。

#### 第三步：还有一种 .tex 不在 zip 里

除了 anim 包里的 atlas，**还有一类独立的 .tex 文件**——专门做 UI 图标用，比如：

```
images/inventoryimages/myth_banana_tree.tex
images/inventoryimages/myth_banana_tree.xml
images/map_icons/peachtree.tex
images/map_icons/peachtree.xml
```

它们**不在 zip 里**，是一对一对独立放在 `images/` 目录下的"散装 atlas"——专门给 HUD、库存图标、地图图标等 UI 系统用。这种是后面 13.6 要讲的 Inventoryimages / 自定义 Atlas。

> **新手记忆**：`.tex = 图`、`.xml = 图的索引`、`build.bin = 骨架`、`anim.bin = 动作`。**它们必须四个一起出现**——少了任何一个，要么模型不显示、要么动作不播。

---

### 13.1.3 快速入门：从 .scml 到 .zip 的"编译概念"

#### 第一步：美术工作流程长什么样

美术从零做一个新角色的流程（13.2 / 13.3 会详讲）：

1. 用 Photoshop 把每个 Symbol 画成 PNG：`head_1.png`、`torso_1.png`、`arm_left_1.png`……
2. 在 Spriter（Klei 内置的免费动画工具）里把这些 PNG 拖进去拼成"骨架"，再在时间轴上拖动每个 Symbol 做出动画
3. 保存为 `xxx.scml`（Spriter 的源文件，文本 XML 格式）+ 一堆 PNG
4. **编译**——用 Klei 提供的工具（`autocompiler.exe` 或 `mod_tools` 里的 `scml.exe`）把 `xxx.scml` + PNG 编译成 `xxx.zip`
5. 把 `xxx.zip` 放进 mod 的 `anim/` 目录，在代码里 `Asset("ANIM", "anim/xxx.zip")` 引用

**编译这一步做了什么？**

- 把所有 PNG 重新打包成 atlas（合并到 `atlas-0.tex`）——同时换成 DXT 压缩格式
- 生成 `atlas-0.xml`（每个 PNG 在大图里的位置）
- 把 `xxx.scml` 里的"骨架定义"和"动作时间线"分别编译成**二进制** `build.bin` 和 `anim.bin`
- 把这四个文件压缩成一个 `.zip`

**关键理解**：`.zip` 文件**不需要解压就能用**——饥荒引擎自带 zip 加载器，直接把 `.zip` 当容器从里面读 `build.bin`、`anim.bin`、`atlas-0.tex`。所以你**不要**手动解压 zip 然后丢一堆散文件给引擎，那样会出错。

#### 第二步：编译输出还有别的吗？

观察一下 Klei 的官方 `data/anim/dynamic/` 目录，会发现一些"奇怪的双胞胎"——

```
data/anim/dynamic/wilson_lava.zip
data/anim/dynamic/wilson_lava.dyn
```

`.zip` 是动画包本身，`.dyn` 是 Klei 引擎用的"延迟加载清单"——告诉引擎这个 zip 里**有哪些 build 名、哪些 anim 名**，以便不解压 zip 就能查询。

这就是后面会讲的 **DYNAMIC_ANIM**（动态加载，按需解压，省内存）。它和普通 ANIM 的区别在于"何时把数据真正读进内存"。

#### 第三步：第一个最小例子

新手如果只想"做一个图标"，最小工作量是：

1. 准备一张 PNG（比如 64×64）
2. 用 `TEXcreator` 或者 `ktools` 把它转成 `myitem.tex`
3. 写一个 `myitem.xml` 描述 `myitem.tex` 里只有一个图
4. 在 mod 的 `modmain.lua` 里：

```lua
Assets = {
    Asset("ATLAS", "images/myitem.xml"),
    Asset("IMAGE", "images/myitem.tex"),
}
```

5. 在代码中用 `inst.components.inventoryitem:ChangeImageName("myitem")` 显示它

**注意**：这一套流程**完全不需要 build.bin / anim.bin**——因为 UI 图标不是骨骼动画，只是一张静态图。所以 `.tex + .xml` 是"最简化版"，`build.bin + anim.bin` 是为了"动起来"才加上的。

> **新手记忆**：`.scml + PNG` 是美术的**源文件**；`.zip` 是编译后的**成品**。普通 mod 直接拿 `.zip` 用就行——除非你要重做美术，否则不需要碰 Spriter。

---

### 13.1.4 进阶：`Asset` 类——所有资源声明的"统一接口"

#### 第一步：定位源码

资源声明的核心是 `Asset` 类，定义在 `scripts/prefabs.lua` 第 25-29 行：

```25:29:scripts/prefabs.lua
Asset = Class( function(self, type, file, param)
    self.type = type
    self.file = file
    self.param = param
end)
```

**简单到极致**——它就是一个三字段容器：

| 字段 | 类型 | 含义 |
|------|------|------|
| `type` | string | 资源类型（`"ANIM"` / `"ATLAS"` / `"IMAGE"` / ...）|
| `file` | string | 资源文件相对路径 |
| `param` | any（可选）| 类型特定参数。比如 `ATLAS_BUILD` 用它表示纹理大小 |

每个 prefab 文件在最开头都会声明一个 `assets` 数组，告诉引擎"我这个 prefab 需要哪些美术资源"。来看 `scripts/prefabs/phonograph.lua` 这个干净的例子：

```1:6:scripts/prefabs/phonograph.lua
local assets =
{
    Asset("ANIM", "anim/phonograph.zip"),
    Asset("INV_IMAGE", "phonograph"),
    Asset("MINIMAP_IMAGE", "phonograph"),
}
```

它声明了三件事：
- 我要用 `anim/phonograph.zip` 这个动画包
- 我的库存图标叫 `phonograph`（需要在 inventoryimages 中能找到）
- 我的小地图图标叫 `phonograph`（需要在某个 minimap atlas 中能找到）

#### 第二步：12 种资源类型——完整速查表

在饥荒源码里翻所有 prefab 文件，能找到下面 12 种 type：

| type 字符串 | 用途 | 典型 file 格式 | 备注 |
|------------|-----|--------------|------|
| `"ANIM"` | 动画包（含 build + anim + atlas）| `anim/xxx.zip` | 最常用。骨骼动画都靠它 |
| `"ATLAS"` | 散装图集（XML 索引）| `images/xxx.xml` | 必须和 IMAGE 配对 |
| `"IMAGE"` | 散装纹理（TEX 像素）| `images/xxx.tex` | 必须和 ATLAS 配对 |
| `"PKGREF"` | 包引用——只声明文件存在但不主动加载 | `xxx.dyn` / `xxx.tex` 等 | 见 `mainfunctions.lua:76`，`.dyn` 文件会被忽略 |
| `"INV_IMAGE"` | 库存图标声明（无文件路径）| 直接写图片名 | 见下面"特殊"说明 |
| `"MINIMAP_IMAGE"` | 小地图图标声明 | 直接写图片名 | 见下面"特殊"说明 |
| `"DYNAMIC_ATLAS"` | 动态加载图集（运行时按需解压）| `images/xxx.xml` | 多用于皮肤、UI |
| `"DYNAMIC_ANIM"` | 动态加载动画包 | `anim/dynamic/xxx.zip` | 配合 PKGREF 的 `.dyn` 使用 |
| `"ATLAS_BUILD"` | 用 atlas 当 build 直接 SetBuild | `images/xxx.xml`, 256 | 第三参数是边长。多用于皮肤 |
| `"SOUND"` | 音效组（fsb 流式音频）| `sound/xxx.fsb` | 见 14 章 |
| `"SOUNDPACKAGE"` | 音效包（fev 元数据）| `sound/xxx.fev` | 见 14 章 |
| `"SHADER"` | GLSL 片元/顶点着色器 | `shaders/xxx.ksh` | 自定义渲染时用 |
| `"SCRIPT"` | 脚本依赖（强制 mod 解析路径）| `scripts/xxx.lua` | 见 `wx78.lua` 第 8-9 行 |

**特殊说明：`INV_IMAGE` 和 `MINIMAP_IMAGE` 没有文件路径**

它们的 `file` 字段直接写图片名（不带后缀、不带 atlas 路径）。引擎会**自动遍历所有已注册的 atlas** 来找这张图。具体流程在 13.1.5 讲。

它们在 `scripts/mainfunctions.lua:69-78` 中**会被 ShouldIgnoreResolve 直接跳过路径解析**：

```69:78:scripts/mainfunctions.lua
function ShouldIgnoreResolve( filename, assettype )
    if assettype == "INV_IMAGE" then
        return true
    end
    if assettype == "MINIMAP_IMAGE" then
        return true
    end
    if filename:find(".dyn") and assettype == "PKGREF" then
        return true
    end
    ...
```

意思是："这种声明仅供 ShelfManager / SimDLL 知道，不要走 resolvefilepath 真的去找文件"。

#### 第三步：一份"完整角色"的资源声明长什么样

来看一个稍微复杂的例子——`w_radio.lua`（沃格斯坦的收音机）：

```1:6:scripts/prefabs/w_radio.lua
local assets =
{
	Asset("ANIM", "anim/w_radio.zip"),
	Asset("DYNAMIC_ATLAS", "images/w_radio_parts.xml"),
	Asset("PKGREF", "images/w_radio_parts.tex"),
}
```

读懂它：
- 主体动画用 `anim/w_radio.zip`
- 收音机的"零件库存图"用 `images/w_radio_parts.xml`，**动态加载**
- 同时声明 `.tex` 文件存在（`PKGREF` 不真的读，但保证文件被打包/校验）

> **进阶记忆**：`Asset(type, file, param?)` 是声明三元组，prefab 文件靠它告诉引擎"我需要哪些美术资源"。**12 种 type 中，95% 的 mod 只会用到 6 种**：ANIM / ATLAS / IMAGE / INV_IMAGE / MINIMAP_IMAGE / SOUND。

---

### 13.1.5 进阶：从 prefab 注册到资源加载——完整链路

#### 第一步：注册总入口

每次饥荒启动加载 prefab，会走 `RegisterPrefabs`（`scripts/mainfunctions.lua:133`）：

```133:141:scripts/mainfunctions.lua
function RegisterPrefabs(...)
    for i, prefab in ipairs({...}) do
		RegisterPrefabsImpl(prefab, RegisterPrefabsResolveAssets)
	end
end

function RegisterSinglePrefab(prefab)
	RegisterPrefabsImpl(prefab, RegisterPrefabsResolveAssets)
end
```

核心逻辑在 `RegisterPrefabsImpl`：

```103:117:scripts/mainfunctions.lua
function RegisterPrefabsImpl(prefab, resolve_fn)
    --print ("Register " .. tostring(prefab))
    -- allow mod-relative asset paths

    for i,asset in ipairs(prefab.assets) do
        if not ShouldIgnoreResolve(asset.file, asset.type) then
       		resolve_fn(prefab, asset)
        end
    end

    modprefabinitfns[prefab.name] = ModManager:GetPostInitFns("PrefabPostInit", prefab.name)
    Prefabs[prefab.name] = prefab

    TheSim:RegisterPrefab(prefab.name, prefab.assets, prefab.deps)
end
```

**核心三步**：

1. **遍历 `prefab.assets`**——对每个资源调用 `resolve_fn`（默认是 `RegisterPrefabsResolveAssets`）
2. **跳过特殊类型**（`INV_IMAGE`、`MINIMAP_IMAGE`、`.dyn` 的 PKGREF）——它们走另一套机制
3. **`TheSim:RegisterPrefab`**——把 prefab 名 + 完整资源清单 + 依赖列表交给 C++ 引擎，引擎会在合适时机真正去硬盘读这些文件

#### 第二步：路径解析——`RegisterPrefabsResolveAssets`

```119:125:scripts/mainfunctions.lua
local function RegisterPrefabsResolveAssets(prefab, asset)
	--print(" - - RegisterPrefabsResolveAssets: " .. asset.file, debugstack())
    local resolvedpath = resolvefilepath(asset.file, prefab.force_path_search, prefab.search_asset_first_path)
    assert(resolvedpath, "Could not find "..asset.file.." required by "..prefab.name)
    TheSim:OnAssetPathResolve(asset.file, resolvedpath)
    asset.file = resolvedpath
end
```

**这一步发生了什么**：

- 输入：相对路径，比如 `anim/phonograph.zip`
- 调用 `resolvefilepath` 把它转成**绝对路径**或**最终生效路径**——比如 `data/anim/phonograph.zip` 或 `mods/MyMod/anim/phonograph.zip`
- 把原路径 → 解析后路径的映射告诉 C++ 引擎（`TheSim:OnAssetPathResolve`）
- **直接把 `asset.file` 改成解析后的路径**——以后所有引用都用这个绝对路径

#### 第三步：路径搜索——`resolvefilepath` 的查找算法

`resolvefilepath` 在 `scripts/util.lua:636` 开始，核心查找在 `softresolvefilepath_internal`（第 585-620 行）：

```585:620:scripts/util.lua
local function softresolvefilepath_internal(filepath, force_path_search, search_first_path)
    force_path_search = force_path_search or false

	if IsConsole() and not force_path_search then
		return filepath -- it's already absolute, so just send it back
	end

	--on PC platforms, search all the possible paths

	--mod folders don't have "data" in them, so we strip that off if necessary. It will
	--be added back on as one of the search paths.
	filepath = string.gsub(filepath, "^/", "")

    --sometimes from context we can know the most likely path for an asset, this can result in less time spent searching the tons of mod search paths.
    if search_first_path then
        local filename = search_first_path..filepath
        if kleifileexists(filename) then
            return filename
        end
    end

	local searchpaths = package.assetpath
    for i, pathdata in ipairs_reverse(searchpaths) do
        local filename = string.gsub(pathdata.path..filepath, "\\", "/")
        if kleifileexists(filename, pathdata.manifest, filepath) then
            return filename
        end
    end

	--as a last resort see if the file is an already correct path (incase this asset has already been processed)
	if kleifileexists(filepath) then
		return filepath
	end

	return nil
end
```

**算法分四步**：

1. **主机平台直通**——PS4/Switch 等主机上，文件路径已经是绝对路径，直接返回
2. **快速路径**（`search_first_path`）——如果调用方提供了"最可能的目录"，先在这里查一次，命中就返回（性能优化）
3. **遍历所有 mod 路径 + data 路径**（`package.assetpath`）—— **倒序**遍历！意味着**最后被加载的 mod 优先级最高**，可以覆盖先前的资源
4. **最后兜底**——直接拿原路径试一次

**`package.assetpath` 长什么样**？这是 Klei 自定义的 Lua 全局变量，类似：

```lua
package.assetpath = {
    {path = "data/", manifest = nil},
    {path = "mods/workshop-XXXXXX/", manifest = ...},
    {path = "mods/MyLocalMod/", manifest = nil},
    -- 倒序遍历，越靠后越优先
}
```

#### 第四步：缓存——`memoizedFilePaths`

第一次解析后，结果会被缓存，避免每次都遍历整个 `package.assetpath`：

```574:582:scripts/util.lua
local memoizedFilePaths = {}

function GetMemoizedFilePaths()
    return ZipAndEncodeSaveData(memoizedFilePaths)
end
function SetMemoizedFilePaths(memoized_file_paths)
    memoizedFilePaths = DecodeAndUnzipSaveData(memoized_file_paths)
end
```

这个缓存还会被序列化保存，下次启动时反序列化恢复——大大加快冷启动速度。

#### 第五步：加载完整时序图

```
prefab 文件返回 Prefab 对象（含 assets 数组）
        ↓
LoadPrefabFile 调用 RegisterSinglePrefab
        ↓
RegisterPrefabsImpl 遍历 assets
        ↓
每个 Asset → ShouldIgnoreResolve 过滤
        ↓
通过的 → RegisterPrefabsResolveAssets
        ↓
resolvefilepath 在 package.assetpath 中搜索
        ↓
asset.file 被替换为绝对路径
        ↓
TheSim:RegisterPrefab(name, assets, deps)
        ↓
（C++ 端）引擎在玩家进入世界、看到这个 prefab 时
        ↓
真正打开 .zip / .tex / .xml，加载到内存
```

> **进阶记忆**：从声明 `Asset` 到真正读硬盘，要经过**注册 → 路径解析 → C++ 端按需加载**三阶段。`resolvefilepath` 是关键——它让你写"`anim/xxx.zip`"这种相对路径，而不用关心 mod 装在哪个目录。

---

### 13.1.6 进阶：六种"特殊资源"的加载机制

不是所有资源都走 `resolvefilepath` 普通流程。下面是六种特殊机制，每一种都有自己的加载逻辑。

#### 第一种：库存图标（INV_IMAGE）—— 自动 atlas 查找

声明：`Asset("INV_IMAGE", "phonograph")`（**注意没有 .tex 后缀**）。

它**不走路径解析**——它的作用只是告诉引擎"我会用到一张叫 phonograph 的库存图"。真正决定从哪个 atlas 拿图，发生在运行时——`scripts/simutil.lua:666-695`：

```666:695:scripts/simutil.lua
function GetInventoryItemAtlas_Internal(imagename, no_fallback)
    local images1 = "images/inventoryimages1.xml"
    local images2 = "images/inventoryimages2.xml"
    local images3 = "images/inventoryimages3.xml"
    local images4 = "images/inventoryimages4.xml"
    return TheSim:AtlasContains(images1, imagename) and images1
            or TheSim:AtlasContains(images2, imagename) and images2
            or TheSim:AtlasContains(images3, imagename) and images3
            or (not no_fallback or TheSim:AtlasContains(images4, imagename)) and images4
            or nil
end

-- Testing and viewing skins on a more close level.
if CAN_USE_DBUI then
    require("dbui_no_package/debug_skins_data/hooks").Hooks("inventoryimages")
end

function GetInventoryItemAtlas(imagename, no_fallback)
	local atlas = inventoryItemAtlasLookup[imagename]
	if atlas then
		return atlas
	end

    atlas = GetInventoryItemAtlas_Internal(imagename, no_fallback)

	if atlas ~= nil then
		inventoryItemAtlasLookup[imagename] = atlas
	end
	return atlas
end
```

**关键**：

- 引擎依次查询 `inventoryimages1.xml` ~ `inventoryimages4.xml` 这 **4 个官方汇总 atlas**，看哪个里面有 `phonograph` 这张图
- mod 可以通过 `RegisterInventoryItemAtlas("自己的atlas路径", "图名.tex")` **手动注册**——见 `scripts/prefabs/deck_of_cards.lua:14-22`
- 找到的 atlas 路径会被**缓存**到 `inventoryItemAtlasLookup`，下次直接返回

#### 第二种：小地图图标（MINIMAP_IMAGE）—— 类似机制

声明：`Asset("MINIMAP_IMAGE", "phonograph")`。

引擎在 `scripts/simutil.lua:698-720` 查询 `minimap/minimap_data1.xml` 和 `minimap_data2.xml` 这两个官方 atlas。

**mod 添加自定义小地图图标的标准流程**（见神话未加密 mod 的 `modmain.lua:355-359`）：

```355:359:mods/联机版mod/神话未加密/modmain.lua
for k,v in pairs(mk_map_icons) do
	table.insert(Assets, Asset( "IMAGE", "images/map_icons/"..v..".tex" ))
    table.insert(Assets, Asset( "ATLAS", "images/map_icons/"..v..".xml" ))
    AddMinimapAtlas("images/map_icons/"..v..".xml")
end
```

`AddMinimapAtlas` 在 `scripts/modutil.lua:512-515` 把 atlas 路径登记到 `MinimapAtlases` 列表，运行时由 `scripts/prefabs/minimap.lua:44-48` 全部塞给 `MiniMap:AddAtlas`：

```44:48:scripts/prefabs/minimap.lua
    for _, atlases in ipairs(ModManager:GetPostInitData("MinimapAtlases")) do
        for _, path in ipairs(atlases) do
            inst.MiniMap:AddAtlas(resolvefilepath(path))
        end
    end
```

**注意**：mod 注册的小地图 atlas **不在** `minimap_data1.xml`/`minimap_data2.xml` 里——它走的是另一条 `MiniMap:AddAtlas` 通道，由 `MiniMapEntity` 在显示图标时遍历查找。

#### 第三种：动态动画（DYNAMIC_ANIM + PKGREF .dyn）

适用于"皮肤、可换装的部件、玩家自定义图集"。声明示例（来自神话未加密 mod 第 374-377 行）：

```374:377:mods/联机版mod/神话未加密/modmain.lua
for _,v in ipairs(mk_skin_assets) do
	table.insert(Assets, Asset("DYNAMIC_ANIM", "anim/dynamic/"..v..".zip"))
    table.insert(Assets, Asset("PKGREF", "anim/dynamic/"..v..".dyn"))
end
```

**两件事发生**：

- `DYNAMIC_ANIM` 让引擎知道"这是个 zip 包，但是不要立刻读到内存——等谁 `SetBuild()` 时再加载"
- `PKGREF` `.dyn` 文件是个**索引清单**——告诉引擎这个 zip 里有哪些 build 名、哪些 anim 名（这样不解压就能查询）

`mainfunctions.lua:76-78` 的特判：

```76:78:scripts/mainfunctions.lua
    if filename:find(".dyn") and assettype == "PKGREF" then
        return true
    end
```

**意思**：`.dyn` 这个 PKGREF 不走 resolvefilepath——因为 `.dyn` 不是真实文件，只是个虚拟标记，C++ 引擎自己处理。

#### 第四种：图集即骨架（ATLAS_BUILD）

声明示例：`Asset("ATLAS_BUILD", "images/monkey_king_item.xml", 256)`

这是**皮肤系统**的特殊用法——把一个 atlas 当成"build"来用，调用 `AnimState:SetBuild("monkey_king_item")` 时引擎会从这个 atlas 找 Symbol。第三个参数 `256` 是 atlas 边长（单位像素），用于 GPU 端布局。

#### 第五种：动态图集（DYNAMIC_ATLAS）

类似 `DYNAMIC_ANIM`，但针对 `.xml + .tex` 这种散装 atlas：用到时才加载。多见于 UI、皮肤"零件库"。

#### 第六种：脚本依赖（SCRIPT）

声明示例（来自 `scripts/prefabs/wx78.lua:8`）：

```lua
Asset("SCRIPT", "scripts/prefabs/player_common.lua"),
```

它**不是真的让引擎读这个 lua**——lua 加载是 `require` 的事。`SCRIPT` 类型的目的是**让 mod 知道这个脚本要被打包**，避免某些 mod 优化把没引用过的 lua 误删。

> **进阶记忆**：90% 的 mod 用普通 `ANIM/ATLAS/IMAGE` 就够了。剩下 10% 的"特殊资源"——`INV_IMAGE`、`MINIMAP_IMAGE`、`DYNAMIC_ANIM`、`ATLAS_BUILD`、`PKGREF .dyn`——都是为了**性能、内存、皮肤系统**才存在的。看到它们时知道"这是优化路径"就好。

---

### 13.1.7 老手：mod 路径搜索的边界条件——七个细节

下面这些边界条件，每一个都对应至少一个我亲眼见过的 mod bug。

#### 细节一：`ipairs_reverse`——后加载 mod 优先

`scripts/util.lua:607` 用的是 `ipairs_reverse`：

```607:611:scripts/util.lua
    for i, pathdata in ipairs_reverse(searchpaths) do
        local filename = string.gsub(pathdata.path..filepath, "\\", "/")
        if kleifileexists(filename, pathdata.manifest, filepath) then
            return filename
        end
    end
```

**含义**：`package.assetpath` 是**按 mod 加载顺序追加**的——第一个加载的 mod 排第二位（第一位通常是 `data/`），最后加载的 mod 排末尾。倒序遍历意味着**最后加载的 mod 优先匹配**。

**实战影响**：如果两个 mod 都有 `anim/wilson.zip`，**模组加载顺序排在后面那个会覆盖前面的**。这就是"mod 优先级覆盖"的底层机制。

#### 细节二：路径分隔符强制 `/`

`string.gsub(pathdata.path..filepath, "\\", "/")` —— 即使你在 mod 里写 `anim\wilson.zip`，最终都被替换成 `anim/wilson.zip`。**好处**：跨平台一致；**坑**：你 `print` 调试时看到的永远是正斜杠，别再纠结"为什么我写反斜杠引擎不认"。

#### 细节三：`force_path_search` 参数

`resolvefilepath(path, force_path_search, search_first_path)` 第二个参数：

```589:590:scripts/util.lua
	if IsConsole() and not force_path_search then
		return filepath -- it's already absolute, so just send it back
	end
```

**含义**：在主机平台上（PS4/Switch），默认不搜索——直接当绝对路径用。**只有 mod 资源**才会传 `force_path_search = true` 强制走搜索逻辑。

**`Prefab` 类**通过 `prefab.force_path_search` 字段传递这个标志：

```6:12:scripts/prefabs.lua
Prefab = Class( function(self, name, fn, assets, deps, force_path_search)
    self.name = string.sub(name, string.find(name, "[^/]*$"))  --remove any legacy path on the name
    self.desc = ""
    self.fn = fn
    self.assets = assets or {}
    self.deps = deps or {}
    self.force_path_search = force_path_search or false
```

mod prefab 文件会自动设置 `force_path_search = true`（在 `modutil.lua` 里），保证主机平台也能找到 mod 资源。

#### 细节四：`kleifileexists` 是核心存在检查

`kleifileexists(filename, manifest, originalpath)` 是 C++ 端导出函数——它检查**Klei 文件系统**里是否存在某文件。它会处理：

- 普通磁盘文件
- mod 包内文件（mod 可以是 zip 形式发布）
- `manifest` 校验（防篡改）

**坑**：mod 大小写敏感性根据宿主操作系统而定——Windows 不敏感，Linux/Mac 敏感。所以**永远写小写文件名**，永远写 `anim/wilson.zip` 不要写 `Anim/Wilson.zip`。

#### 细节五：`asset.file` 被原地修改

```119:124:scripts/mainfunctions.lua
local function RegisterPrefabsResolveAssets(prefab, asset)
    local resolvedpath = resolvefilepath(asset.file, prefab.force_path_search, prefab.search_asset_first_path)
    assert(resolvedpath, "Could not find "..asset.file.." required by "..prefab.name)
    TheSim:OnAssetPathResolve(asset.file, resolvedpath)
    asset.file = resolvedpath
end
```

`asset.file = resolvedpath` 是**原地修改**——`prefab.assets[i]` 数组里那个 Asset 对象的 file 字段从 `anim/xxx.zip` 变成了 `mods/MyMod/anim/xxx.zip`。

**坑**：如果你在 prefab 内部又写 `print(asset.file)`，**第一次注册前是相对路径，注册后是绝对路径**——别被绕晕。

#### 细节六：`async_batch_validation` 和 `VerifyPrefabAssetExistsAsync`

```127:131:scripts/mainfunctions.lua
local function VerifyPrefabAssetExistsAsync(prefab, asset)
	-- this is being done to prime the HDD's file cache and ensure all the assets exist before going into game
	--TheSim:VerifyFileExistsAsync(asset.file)
	TheSim:AddBatchVerifyFileExists(asset.file)
end
```

**用途**：`LoadPrefabFile(filename, async_batch_validation, ...)` 第二个参数 `true` 时，引擎不立刻 resolve，而是**批量异步验证文件存在**——主要在加载界面期间利用 IO 空闲做"硬盘缓存预热"，让真正进游戏时秒读。

#### 细节七：`PREFABDEFINITIONS` 全局表

```174:174:scripts/mainfunctions.lua
                PREFABDEFINITIONS[val.name] = val
```

每个被加载的 Prefab 对象会被存入 `PREFABDEFINITIONS[name]`——你可以在控制台 `print(PREFABDEFINITIONS["wilson"].assets)` 看完整资源清单（路径已被解析过）。**调试 mod 资源问题的神器**。

> **老手记忆**：mod 路径搜索是**倒序、跨平台、原地修改**的。debug 时记住三件事：① 路径全用 `/` 写小写；② `print(PREFABDEFINITIONS[name].assets)` 看真实路径；③ 报 `Could not find` 时先查 mod 加载顺序和 `package.assetpath`。

---

### 13.1.8 老手：八个最容易踩的坑

把整个 13.1 章学到的所有知识合到这 8 个场景里——每一个都是 mod 论坛常见高频问题。

#### 坑 1：缺一不可——ATLAS 和 IMAGE 必须成对

```lua
-- 错误
Assets = {
    Asset("ATLAS", "images/myicon.xml"),
    -- 缺了 IMAGE！
}

-- 正确
Assets = {
    Asset("ATLAS", "images/myicon.xml"),
    Asset("IMAGE", "images/myicon.tex"),
}
```

**原因**：`.xml` 是索引，告诉引擎"这张大图里有哪些小图"；`.tex` 是真正的像素数据。引擎读 `.xml` 时会去找对应的 `.tex`——找不到就显示紫红色"missing texture"格子图。

#### 坑 2：图标显示为紫红色 missing 标记

**症状**：库存格子里出现一张紫红色棋盘图。

**原因有三**：
1. `.tex` 文件没声明 `Asset("IMAGE", ...)` —— 引擎不知道有这文件
2. `.xml` 里没有这张图的 `<Element name="xxx.tex" .../>` 条目 —— atlas 索引里查不到
3. `inst.components.inventoryitem:ChangeImageName("xxx")` 时图名拼错，或者 `RegisterInventoryItemAtlas` 没注册自定义 atlas

**调试**：`print(GetInventoryItemAtlas("myicon"))` —— 如果返回 `nil`，是没注册；如果返回错误的 atlas，是注册到了别的 atlas。

#### 坑 3：动画播放但模型不显示

**症状**：`PlayAnimation("idle")` 没报错，控制台 `c_select():GetAnimState()` 看 build/bank 都对，但是屏幕上空空如也。

**核心 API 三件套**（13.4 详讲）：

```lua
inst.AnimState:SetBank("phonograph")   -- 用哪个 anim.bin（动作）
inst.AnimState:SetBuild("phonograph")  -- 用哪个 build.bin（骨架）
inst.AnimState:PlayAnimation("idle")    -- 播放指定 anim
```

**常见原因**：
1. `SetBank` 错了——`bank` 名（不带后缀）必须和 zip 包内 `anim.bin` 的 bank 名匹配
2. `SetBuild` 错了——`build` 名必须和 `build.bin` 内声明的 build 名匹配
3. **bank 名和 build 名是不同概念**！很多人以为它们一样——其实 `wilson` 这个 bank 可以用 `wilson` build 也可以用 `wendy_skin` build（这就是换皮肤）

**调试**：在控制台 `c_select().AnimState:GetCurrentFacing()` / `:GetCurrentAnimationName()` / `:GetBuild()` / `:GetBank()` 都打一遍。

#### 坑 4：换装失败——OverrideSymbol 找不到 build

```lua
inst.AnimState:OverrideSymbol("swap_object", "myaxe", "swap_axe")
-- 没效果！
```

**原因**：你声明的资源是 `Asset("ANIM", "anim/myaxe.zip")` 还是 `Asset("DYNAMIC_ANIM", ...)`？

- 如果是普通 `ANIM`——引擎启动时已加载，`OverrideSymbol` 立刻生效
- 如果是 `DYNAMIC_ANIM`——需要先 `inst.AnimState:AddOverrideBuild("myaxe")` 触发加载，**再** `OverrideSymbol`

**坑中坑**：`anim/myaxe.zip` 内必须真的有 `swap_axe` 这个 Symbol——你在 Spriter 里建立 Symbol 时要叫这个名字。Symbol 名和 build 名是 zip 内部数据，**不是文件名**。

#### 坑 5：`Could not find an asset matching ...`

`scripts/util.lua:641` 抛的：

```lua
assert(resolved ~= nil, "Could not find an asset matching "..filepath.." in any of the search paths.")
```

**排查清单**：
1. 文件**真的存在**于 mod 目录？大小写是否一致？
2. mod 是否成功加载？在游戏 mod 设置界面是否启用？
3. mod 安装路径是否进了 `package.assetpath`？控制台 `for k,v in pairs(package.assetpath) do print(k, v.path) end` 检查
4. 如果是从其他 mod 来的资源——必须**显式列出**，不能假设另一个 mod 加载就能直接用

#### 坑 6：`.dyn` 文件丢失

```
Asset("PKGREF", "anim/dynamic/myskin.dyn")
```

如果你只放了 `myskin.zip` 没放 `myskin.dyn`，**没有 `.dyn` 时 DYNAMIC_ANIM 也能勉强用**——但是某些"按 build 名查询"的 API 会失败。

**如何生成 `.dyn`**？Klei 的 mod 工具链 (`mod_tools/krane.exe` 等) 编译时会自动生成。手写也可以，是个简单的二进制清单。

#### 坑 7：自定义 atlas 在 dedicated server 上崩溃

```80:96:scripts/mainfunctions.lua
    if TheNet:IsDedicated() then
        if assettype == "SOUNDPACKAGE" then
            return true
        end
        if assettype == "SOUND" then
            return true
        end
        if filename:find(".ogv") then
            return true
        end
        if filename:find(".fev") and assettype == "PKGREF" then
            return true
        end
        if filename:find("fsb") then
            return true
        end
	end
    return false
```

**dedicated server**（专用服务器）上 `SOUND` / `SOUNDPACKAGE` / `.ogv` / `.fev` / `.fsb` 都被自动忽略——服务器不需要音频/视频。

**坑**：但是 **ATLAS / IMAGE / ANIM 在 server 上也会被解析**——如果你的 mod 在 server-only 代码里 require 了一个只在 client 才该有的 atlas，server 启动时会找不到资源。**server-only 资源声明要手动判断 `TheNet:IsDedicated()`**。

#### 坑 8：mod 资源跨 mod 共享失败

**场景**：MOD A 提供一个角色模型 `anim/witcher.zip`，MOD B 想做一个"witcher 专属武器" prefab，引用 `anim/witcher.zip`。

**问题**：每个 mod 的资源搜索是**相对自己 mod 目录**的——MOD B 写 `Asset("ANIM", "anim/witcher.zip")`，引擎会在 `mods/MOD_B/anim/` 找，找不到。

**解法**：
- **拷贝资源**——MOD B 自己也复制一份 `witcher.zip`（最简单，但占空间）
- **依赖加载顺序**——`force_path_search = true` 时引擎会遍历**所有** mod 路径——后加载的 MOD B 能搜到 MOD A 的资源（但是不可靠，因为玩家可能调整加载顺序）
- **显式告知 `search_first_path`**——`Prefab(name, fn, assets, deps, force_path_search)` 手动传这个参数（生产中很少用）

> **老手记忆**：8 个坑可以归为三类——**资源声明错误**（坑 1、5）、**资源**和 **代码**不匹配**（坑 3、4、6）、**加载平台**和 **加载顺序问题**（坑 2、7、8）。每次出问题先问自己"这是哪一类"。

---

### 13.1 小结：关于"美术资源"你必须记住的

读完本节，你应该能在脑子里画出下面这张图：

```
     美术 PNG + Spriter (.scml)
              │
              ▼  [编译 - 13.3]
       xxx.zip 包
       ├── atlas-N.tex   ← 像素 (DXT 压缩)
       ├── atlas-N.xml   ← Symbol UV 索引
       ├── build.bin     ← 骨架/锚点
       └── anim.bin      ← 动作时间线
              │
              ▼  [Asset 声明]
       prefab.assets = {
         Asset("ANIM", "anim/xxx.zip"),
         ...
       }
              │
              ▼  [RegisterPrefab]
       resolvefilepath 在 package.assetpath 中搜索
              │
              ▼
       asset.file 被替换为绝对路径
              │
              ▼  [TheSim:RegisterPrefab]
       C++ 引擎按需加载到内存
              │
              ▼  [AnimState API - 13.4]
       SetBank / SetBuild / PlayAnimation
              │
              ▼
       屏幕上那只 Wilson 出现了
```

**新手核心三句**：四个文件分别是图、索引、骨架、动作；mod 用 `Asset("ANIM", ...)` 声明；`AnimState` 三件套（SetBank / SetBuild / PlayAnimation）让它动起来。

**进阶核心三句**：12 种 Asset 类型按"普通文件 / 特殊查询（INV/MINIMAP）/ 动态加载（DYNAMIC）"三类记；`RegisterPrefabsImpl → resolvefilepath → kleifileexists` 是路径解析三段；`asset.file` 注册后会被原地改成绝对路径。

**老手核心三句**：`package.assetpath` 倒序匹配，最后加载 mod 优先；`PREFABDEFINITIONS[name].assets` 是调试神器；八个常见坑里"资源声明错"和"资源/代码不匹配"占七成。

接下来 13.2-13.8 是这套管线的具体环节——Spriter 怎么用（13.2）、scml 怎么编译（13.3）、AnimState API 完整列表（13.4）、换装原理（13.5）、库存和小地图图标制作（13.6/13.7）、最终一个完整动画 demo（13.8）。先有 13.1 这个"鸟瞰图"，后面每一节才不会迷路。

---


## 13.2 Spriter 动画工具使用

### 本节导读

13.1 我们看到 `anim/xxx.zip` 的内部是 `build.bin + anim.bin + atlas.tex + atlas.xml`。但是这些 `.bin` 不是手写出来的——它们是 **Spriter** 这个编辑器的"编译产物"。

Spriter（[BrashMonkey Spriter](https://brashmonkey.com/) 出品）是 Klei 在饥荒、Don't Starve Together、Mark of the Ninja、Oxygen Not Included 等多款游戏里**官方采用**的 2D 骨骼动画工具。它的源文件后缀是 `.scml`（Spriter Character Markup Language，文本 XML 格式）。每一个角色、每一个怪物、每一件武器，**美术都是先在 Spriter 里"演"出来，再编译成 zip**。

所以，本节的目标不是教你"怎么熟练用 Spriter"——那是一本几百页的工具书。本节的目标是：

> **新手**从 13.2.1-13.2.3 起步——理解 Spriter 是个什么东西、它的三个核心概念（Symbol / Frame / Anim）、第一次打开 Spriter 看到什么；**进阶读者**继续看 13.2.4-13.2.6，掌握"建一个 build"和"做一个 anim"的标准操作步骤、Klei 约定的命名规范（怎么和 `OverrideSymbol` 配合）；**老手**跳到 13.2.7-13.2.8，理解 `.scml` 内部 XML schema、Spriter 项目目录组织、八个生产中常见的坑和解决方案。

读完本节，你**不一定能立刻独立做出一个完整角色**——做角色是美术的工作。但是你能：① 看懂别人 mod 的 `.scml` 项目；② 在 Spriter 里临时改一个 Symbol 的位置；③ 准确告诉美术"我代码里调用的是 anim 名 idle、Symbol 名 swap_object，请你按这个命名"。

---

### 13.2.1 快速入门：Spriter 是什么？

#### 第一步：用一句话定位 Spriter

Spriter 在饥荒美术管线中只做**一件事**：

> **把一堆静态 PNG 拼成"骨骼动画"，然后导出成饥荒引擎能读的 `xxx.zip`**

它的输出物就是 13.1 讲的：
- `build.bin`（这堆 PNG 怎么拼成骨架）
- `anim.bin`（每一帧每个 Symbol 在哪、转多少度、缩放多少）
- `atlas-N.tex` + `atlas-N.xml`（PNG 被打包成图集）

如果用一个**戏剧的比喻**：

| 戏剧元素 | Spriter 中对应 |
|---------|---------------|
| 演员 | Symbol（每一张可以独立移动/旋转的图） |
| 演员的服装/造型 | Build（一整套 Symbol 的集合，定义"角色长什么样"） |
| 一段戏的剧本 | Anim（一段动画，比如 idle、walk、attack） |
| 演员表 + 剧本 = 一场演出 | `xxx.scml` 项目文件 |

**关键认识**：Spriter 不画画——你需要先在 Photoshop 里把每个部件画成 PNG，再**导入** Spriter 来编排它们的运动。Spriter 是"导演工具"，不是"画图工具"。

#### 第二步：去哪里下载 Spriter？

- **Klei 官方推荐**：直接用 Spriter Pro 或 Spriter R11 (免费版) —— [BrashMonkey 官网](https://brashmonkey.com/)
- **Klei 自带 mod tools**：在 Steam 库里搜索 "Don't Starve Mod Tools"——这是 Klei 官方发布的免费工具包，里面**自带 Spriter 编辑器**和编译工具链（`scml.exe`、`autocompiler.exe`、`krane.exe`），是做饥荒 mod 的**官方推荐方式**

> **重要**：如果你只是想做 mod，**强烈建议直接装 Don't Starve Mod Tools**——它附带的 Spriter 已经预设好了和饥荒兼容的导出参数，不用自己折腾。

#### 第三步：Spriter 的本体是什么文件

打开 mod tools 后会看到 Spriter 的工作目录：

```
mod_tools/
├── Spriter/
│   ├── SpriterPro.exe       ← 主程序
│   ├── exported/            ← 你导出的 .scml 项目放这里
│   └── examples/            ← Klei 提供的官方角色源工程（极有学习价值！）
└── tools/
    ├── autocompiler.exe     ← 一键编译 .scml → .zip
    ├── scml.exe             ← scml 编译器（autocompiler 内部调用它）
    └── krane.exe            ← 反向工具（可以把 .zip 解出来变 .scml）
```

**examples 目录是宝藏**——里面有 Wilson、Wendy、各种猪人/兔人的完整 Spriter 源工程！想学动画约定就去翻它们。

> **新手记忆**：Spriter 是 2D 骨骼动画"导演工具"，把 PNG 拼成动作并导出成饥荒能读的 zip。**不要去网上找其他 2D 动画工具**——只有 Spriter 是 Klei 官方支持的，其他工具导出的格式饥荒都读不了。

---

### 13.2.2 快速入门：Symbol、Frame、Anim 三个核心概念

任何骨骼动画工具都绕不开这三个概念。在 Spriter 中它们叫这些名字（Klei 也沿用了这套术语）：

#### 第一个概念：Symbol（符号）

**定义**：一个 Symbol 就是"动画的最小可移动单位"——通常对应一张 PNG。

**例子**：Wilson 这个角色，他的 Symbol 包括：
- `head` —— 头
- `torso` —— 躯干
- `arm_upper_l`、`arm_upper_r` —— 左右大臂
- `arm_lower_l`、`arm_lower_r` —— 左右小臂
- `hand_l`、`hand_r` —— 左右手
- `leg_upper_l`、`leg_upper_r` —— 左右大腿
- `leg_lower_l`、`leg_lower_r` —— 左右小腿
- `swap_object` —— 手中物体的"占位 Symbol"
- ……（一个完整角色通常有 30-50 个 Symbol）

**关键性质**：Symbol 是**可以被代码替换的**。这就是 13.5 讲的 `AnimState:OverrideSymbol`——比如玩家拿斧头时，代码会把 `swap_object` 这个 Symbol 替换成斧头的图。来看 `nightstick.lua` 第 30 行：

```30:30:scripts/prefabs/nightstick.lua
        owner.AnimState:OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")
```

这行的意思就是："把当前 build 里的 `swap_object` Symbol 替换成 `swap_nightstick` 这个 build 里的 `swap_nightstick` Symbol"。

#### 第二个概念：Frame（帧）

**定义**：一个 Symbol 在某个时刻的一个静态 PNG 图。

**例子**：
- `head-0` —— 头部正面
- `head-1` —— 头部侧面
- `head-2` —— 头部后面
- `head-3` —— 头部低头
- ……

一个 Symbol 可以有多个 Frame（在 Spriter 里叫 "files"）。代码可以指定使用哪一帧：

```lua
inst.AnimState:OverrideSymbol("head", "wilson", "head-2")
-- 用 wilson build 里 head Symbol 的第 2 帧
```

**Frame 命名约定**：饥荒里 Symbol 的 frame 通常用编号 `-N` 后缀：`head-0`、`head-1`、`head-2`……。在 Spriter 中导入 PNG 时，把文件命名成 `head-0.png`、`head-1.png` 即可，编译器自动识别。

#### 第三个概念：Anim（动画）

**定义**：一段时间内"所有 Symbol 怎么变换"的剧本。

**例子**：Wilson 的常见 anim：
- `idle` —— 站立
- `idle_loop` —— 站立循环（小幅度呼吸）
- `walk_pre` / `walk_loop` / `walk_pst` —— 走路前奏 / 循环 / 收尾
- `run_pre` / `run_loop` / `run_pst` —— 跑步
- `attack` —— 攻击
- `chop_pre` / `chop_loop` / `chop_pst` —— 砍树
- `death` —— 死亡

一个 anim 由若干**关键帧（keyframe）**组成。每个关键帧记录"在这一时刻，每个 Symbol 应该在哪个位置、什么角度、什么缩放、用哪个 Frame"。Spriter 在两个关键帧之间**自动补间（tween）**生成中间帧，得到平滑动作。

**代码中调用 anim**——还是 `phonograph.lua`：

```112:112:scripts/prefabs/phonograph.lua
    inst.AnimState:PushAnimation("play_loop", true)
```

这一行的意思是"在动画队列里追加 anim 名为 `play_loop` 的动画，第二个参数 true 表示循环播放"。这个 `play_loop` 必须是 `phonograph.scml` 里定义过的 anim，否则会报错。

#### 第四步：把三者串起来

```
.scml 文件
  ├── 一个 build（这里也叫 entity）
  │   ├── Symbol "head"
  │   │   ├── Frame: head-0.png
  │   │   ├── Frame: head-1.png
  │   │   └── ...
  │   ├── Symbol "torso"
  │   │   └── ...
  │   └── ...
  └── 一个 anim "idle"
      ├── 第 0 帧：head 在 (0, 50)、用 Frame head-0
      ├── 第 5 帧：head 在 (1, 51)、用 Frame head-0
      ├── 第 10 帧：head 在 (0, 50)、用 Frame head-0
      └── ...（这就是"呼吸"动画）
```

> **新手记忆**：**Symbol 是"角色的零件"**（可被换装替换）；**Frame 是"零件的不同角度/姿态"**；**Anim 是"零件们随时间运动的剧本"**。

---

### 13.2.3 快速入门：Spriter 界面五大区域

第一次打开 Spriter，会看到这样的界面（用文字示意，因为篇幅有限）：

```
┌──────────────┬─────────────────────────────┬──────────────┐
│              │                             │              │
│  [1]         │           [3]               │   [4]        │
│  File Tree   │      Canvas (画布)          │   Properties │
│  (文件树)    │                             │   (属性面板) │
│              │                             │              │
│              │                             │              │
├──────────────┼─────────────────────────────┴──────────────┤
│              │                                            │
│  [2]         │           [5]                              │
│  Entity      │      Timeline (时间轴)                     │
│  Tree        │                                            │
│  (实体树)    │                                            │
└──────────────┴────────────────────────────────────────────┘
```

| 区域 | 名称 | 作用 |
|------|------|------|
| 1 | File Tree（左上）| 显示项目里所有 PNG 文件——拖动它们到 Canvas 中即可创建 Symbol |
| 2 | Entity / Anim Tree（左下）| 切换 entity（=build）和 anim 的列表 |
| 3 | Canvas（中央）| 主画布——你拼角色、摆动作的地方 |
| 4 | Properties（右上）| 当前选中物体的位置、旋转、缩放、Symbol 名等 |
| 5 | Timeline（底部）| 时间轴——拖动关键帧创建动画 |

**最常用的快捷操作**：

| 快捷键 | 作用 |
|-------|------|
| `Space` 或 `Play` 按钮 | 播放当前 anim |
| 鼠标拖动 Canvas 上的部件 | 移动 Symbol（自动产生关键帧）|
| 时间轴右键 → Insert Key | 在当前时间插入一个关键帧 |
| 选中部件 + R | 旋转模式 |
| 选中部件 + S | 缩放模式 |
| `Ctrl+S` | 保存 .scml 项目 |

> **新手记忆**：Spriter 界面 5 个区——左上看 PNG 库、左下切 anim、中央拼角色、右边改属性、底部拉时间轴。**不会做动画也没关系——本节核心目标只是"能看懂别人的 Spriter 项目"**。

---

### 13.2.4 进阶：建立一个 Build——把 PNG 拼成骨架

#### 第一步：准备 PNG

在做任何 Spriter 操作之前，**先在 Photoshop 把每个部件画成独立的透明 PNG**：

```
my_character/
├── head-0.png       ← 头部，正面
├── torso.png        ← 躯干
├── arm_upper.png    ← 大臂
├── arm_lower.png    ← 小臂
├── hand.png         ← 手
├── leg_upper.png    ← 大腿
├── leg_lower.png    ← 小腿
├── foot.png         ← 脚
└── swap_object.png  ← 占位的"手持物"，通常是个空白透明小图
```

**几个铁律**：

1. **每张 PNG 必须有透明通道**（不能是 JPG）
2. **每张 PNG 的画布大小要包含部件本身**（不要让部件超出画布边缘——会被裁剪）
3. **PNG 的"原点"（中心点）很重要**——Spriter 默认把 PNG 中心当锚点，但是你可以改

#### 第二步：在 Spriter 中创建项目

1. 打开 SpriterPro.exe
2. `File → New Project` —— 选择一个目录（建议直接是 mod 的 anim 源工程目录）
3. **把所有 PNG 拖进项目目录**——Spriter 会自动检测到它们，在左上 File Tree 中列出

#### 第三步：拖出第一个 Symbol

1. 在 File Tree 里点 `torso.png`
2. **拖到 Canvas 中央** —— 在 Canvas 上出现一个躯干
3. 这就创建了一个名为 `torso` 的 Symbol（默认用文件名）

**注意**：在 Spriter 里**Symbol 名 = PNG 文件名（不带后缀和 -N 编号）**。所以 PNG 取名很重要——直接决定了代码中 `OverrideSymbol("torso", ...)` 用的是什么字符串。

#### 第四步：拼组完整角色

继续把其他 PNG 拖到 Canvas，按解剖学顺序摆放：

- 头摆在躯干上方
- 大臂连在肩膀
- 小臂连在大臂末端
- 手连在小臂末端
- 大腿连在髋部
- 小腿连在大腿
- 脚连在小腿末端

#### 第五步：调整渲染层次（z-order）

部件之间会**互相遮挡**——头要在躯干上面，但是手要在头旁边。在 Spriter 中，**Canvas 上的图层顺序**就是 z-order（越上面渲染越靠前）。可以在右键菜单里 Move Up / Move Down 调整。

**Klei 约定的渲染顺序**（从后到前）：

```
最后面 ← arm_back / leg_back
       ← torso
       ← head
       ← arm_front / leg_front
       ← hand_front
最前面 ← swap_object（玩家手持物）
```

#### 第六步：保存为 build

在 Entity Tree 里默认有一个 `entity_000`——**右键改名**为你的 build 名（比如 `myhero`）。这就是后面代码中 `AnimState:SetBuild("myhero")` 用的字符串。

> **关键约定**：**Symbol 名、Build 名、Frame 名都不能有空格、不能有大写、不要超过 32 字符**——否则编译时会被截断或报错。Klei 自己的命名都是全小写下划线分隔。

#### 第七步：Klei 标准 Symbol 命名清单

如果你做的是**玩家角色**，必须遵循这套命名（否则代码里调用的 `Show("ARM_carry")`、`OverrideSymbol("swap_object", ...)` 都会失效）：

| Symbol 名 | 用途 |
|----------|------|
| `head_hat` / `head_hair` / `head_hair_hat` / `headbase` / `headbase_hat_nohat` | 头部各部位（戴帽子时显隐切换） |
| `face` | 面部（独立 Symbol，可以单独换表情） |
| `torso` / `torso_pelvis` | 躯干 |
| `arm_upper_skin` / `arm_upper` / `arm_lower` | 上下臂 |
| `hand` | 手 |
| `ARM_carry` / `ARM_normal` | "手中拿东西"和"空手"两种状态——见 voidcloth_scythe 第 179-180 行 |
| `swap_object` | 玩家手中持有物的 placeholder，被武器/工具的 OverrideSymbol 替换 |
| `swap_hat` | 头上的帽子 placeholder |
| `swap_body` | 身上的护甲/衣服 placeholder |

**这些不是引擎硬编码的**——但是 Klei 自己的所有 prefab 代码都基于这套命名。如果你做新角色不用这套，所有官方装备就**穿不上你的角色**。

> **进阶记忆**：Symbol 命名是**契约**——美术和程序员之间约定好的字符串。**做新角色时一定要照抄 Klei 的 Symbol 命名清单**，否则装备/换装系统全部失效。

---

### 13.2.5 进阶：制作一个 Anim——动作时间线

有了 build（角色骨架），就可以做 anim（动作）了。我们以最简单的"呼吸 idle"为例。

#### 第一步：创建一个 anim

1. 切到 Anim Tree（左下角）
2. 点 `New Animation`，命名为 `idle`
3. 时间轴自动跳到这个新 anim

#### 第二步：在第 0 帧记录"原姿"

时间游标在 0 ms 位置——**在 Canvas 上的部件应该是默认姿势**。Spriter 会自动在这个时刻记录所有 Symbol 的当前位置 → **这就是关键帧 0**。

#### 第三步：在中间时刻调整姿态

1. 把时间游标拖到 500 ms
2. 在 Canvas 上**轻微拖动** torso 向上 1 像素（模拟吸气）
3. **自动产生新的关键帧**——表示"500 ms 时 torso 在 (0, 1)"

#### 第四步：在结束时刻回到原姿

1. 把时间游标拖到 1000 ms
2. 把 torso 拖回原点 (0, 0)
3. 关键帧 3 完成

#### 第五步：标记循环

1. 在 Anim 属性里勾上 `Looping`
2. 设置 `Length` 为 1000 ms
3. 按 `Space` 看到角色自动呼吸

#### 第六步：补间（Tween）—— 自动生成中间帧

你只画了 3 个关键帧（0、500、1000 ms），但是动画看起来**完全丝滑**——这是因为 Spriter 在每两个关键帧之间**线性插值**（也可以选 quadratic、cubic 等缓动函数）。这就是骨骼动画相比逐帧动画的核心优势：**手画的关键帧少，得到的动画却很流畅**。

#### 第七步：Klei 约定的 anim 命名

Klei 的 stategraph 系统会按特定 anim 名调用 PlayAnimation——你做新角色一定要遵守：

| Anim 名前缀 | 用途 |
|------------|------|
| `idle_loop` | 站立循环 |
| `walk_pre` / `walk_loop` / `walk_pst` | 走路三段：前奏 / 循环 / 收尾 |
| `run_pre` / `run_loop` / `run_pst` | 跑步三段 |
| `attack` | 单次攻击 |
| `attack_pre` / `attack_loop` / `attack_pst` | 蓄力攻击三段 |
| `hit` | 受击 |
| `death` | 死亡 |
| `pickup` | 拾取 |
| `eat_pre` / `eat` / `eat_pst` | 吃东西 |

**`_pre`/`_loop`/`_pst` 三段式**是饥荒动画的核心模式——这样一段动作可以**任意拉长**：在 `_loop` 阶段循环播放，玩家点鼠标停止时再播 `_pst` 收尾。

#### 第八步：anim 中的事件标记

在 Spriter 时间轴上可以标记**meta event**——比如在 attack anim 的 0.3 秒处放个标记 `'attack_hit'`。代码里可以监听这个事件：

```lua
inst.AnimState:SetSound("attack_hit", "dontstarve/wilson/attack_whoosh")
-- 或者直接监听
inst:ListenForEvent("animover", function() ... end)
```

但是 Klei 的体系里更常用的是**stategraph 中的 timeline**（见 8.3 讲过），不是 Spriter 内置事件。

> **进阶记忆**：Anim = 关键帧序列；用 `_pre`/`_loop`/`_pst` 三段式让动作可以拉长；anim 命名要遵守 Klei 约定，stategraph 才能正确驱动。

---

### 13.2.6 进阶：导出 .scml + 资源——Spriter 的输出物

#### 第一步：保存项目

`File → Save` 会得到 `.scml` 文件 + 项目目录里的所有 PNG。

**项目目录长这样**：

```
my_anim_source/
├── myhero.scml           ← Spriter 项目文件（XML 格式）
├── myhero.scon           ← 自动生成的 JSON 备份（兼容 Spriter 较新版本）
├── head-0.png
├── head-1.png
├── torso.png
├── arm_upper.png
├── ...
└── swap_object.png
```

**`.scml` 是文本 XML**——可以用记事本打开看（13.2.7 会拆解结构）。**所以它支持 git diff、可以人工修补**。

#### 第二步：调用编译器

最常用的是 mod tools 里的 **autocompiler.exe**：

```bash
# 进入 mod 目录
cd mods/MyMod/

# 跑编译
../../mod_tools/tools/autocompiler.exe
```

`autocompiler.exe` 会：

1. 扫描 mod 目录下所有 `exported/` 下的 Spriter 项目
2. 调用 `scml.exe` 把每个 `.scml` + PNG → `xxx.zip`
3. 把生成的 zip 放到 `mods/MyMod/anim/` 目录
4. 打印日志，告诉你哪些项目编译成功 / 失败

**约定**：源工程放在 `mods/MyMod/exported/myhero/myhero.scml`，编译产物自动出现在 `mods/MyMod/anim/myhero.zip`。这就是为什么 mod 目录下你**只看见 zip 看不到 scml**——大部分 mod 作者只发布编译产物，不发布源工程（保护美术成果）。

#### 第三步：编译时干了什么

`scml.exe` 内部干的事（13.3 详讲）：

1. 解析 `.scml` 的 XML
2. 把所有 PNG **打包**成一张大图 `atlas-0.tex`（Klei 用的是 DXT5 压缩算法）
3. 写出 `atlas-0.xml`（每个 Symbol 在 atlas 中的 UV 矩形）
4. 把 `.scml` 中的 entity（build）部分编译成 `build.bin`（二进制）
5. 把所有 anim 编译成 `anim.bin`（二进制）
6. 把 `build.bin + anim.bin + atlas-0.tex + atlas-0.xml` 打包到一个 zip

#### 第四步：手动编译单个 .scml

如果只想编译一个项目，可以直接调 `scml.exe`：

```bash
mod_tools/tools/scml.exe path/to/myhero.scml --output mods/MyMod/anim/myhero.zip
```

#### 第五步：反向工具——把 zip 还原成 scml

`krane.exe` 是个救命工具——它可以**反向**把别人发布的 zip "解开"成 .scml + PNG：

```bash
mod_tools/tools/krane.exe mods/AnotherMod/anim/their_character.zip --output extracted/
```

**用途**：
- 学习别人的动画做法
- 修复一个 zip 里某帧的 bug
- 给别人 mod 的角色"打补丁"

**注意**：Klei 官方角色（Wilson、Wendy 等）的 zip **理论上**也能 krane 出来——但是 mod 作者**不应该未经授权重新发布解出来的 Klei 资源**。仅限学习。

> **进阶记忆**：Spriter → `.scml` → autocompiler / scml.exe → `xxx.zip`。**反过来 krane.exe 可以把 zip 还原成 scml**。

---

### 13.2.7 老手：`.scml` 文件内部 XML 结构拆解

`.scml` 是文本 XML，老手可以直接编辑它。来看一份精简的 scml 长什么样：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<spriter_data scml_version="1.0" generator="BrashMonkey Spriter" generator_version="r10.1">

  <!-- 第一段：folder——所有 PNG 资源的索引 -->
  <folder id="0" name="">
    <file id="0" name="head-0.png" width="64" height="64" pivot_x="0.5" pivot_y="0.5"/>
    <file id="1" name="head-1.png" width="64" height="64" pivot_x="0.5" pivot_y="0.5"/>
    <file id="2" name="torso.png" width="48" height="60" pivot_x="0.5" pivot_y="0.5"/>
    <file id="3" name="swap_object.png" width="32" height="32" pivot_x="0.5" pivot_y="0.5"/>
  </folder>

  <!-- 第二段：entity——一个 build（角色骨架） -->
  <entity id="0" name="myhero">

    <!-- entity 内部：所有 Symbol 的关键帧（timeline） -->
    <obj_info name="head" type="sprite" w="64" h="64"/>
    <obj_info name="torso" type="sprite" w="48" h="60"/>
    <obj_info name="swap_object" type="sprite" w="32" h="32"/>

    <!-- entity 内部：每一个 anim -->
    <animation id="0" name="idle" length="1000" looping="true">

      <!-- mainline——记录所有关键帧的"对齐时刻" -->
      <mainline>
        <key id="0" time="0">
          <object_ref id="0" name="head"   timeline="0" key="0" z_index="2"/>
          <object_ref id="1" name="torso"  timeline="1" key="0" z_index="1"/>
        </key>
        <key id="1" time="500">
          <object_ref id="0" name="head"   timeline="0" key="1" z_index="2"/>
          <object_ref id="1" name="torso"  timeline="1" key="1" z_index="1"/>
        </key>
      </mainline>

      <!-- 每个 Symbol 一条 timeline——记录这个 Symbol 在每个时刻的具体状态 -->
      <timeline id="0" name="head">
        <key id="0" time="0">
          <object folder="0" file="0" x="0" y="50" angle="0" scale_x="1" scale_y="1"/>
        </key>
        <key id="1" time="500">
          <object folder="0" file="0" x="0" y="51" angle="0" scale_x="1" scale_y="1"/>
        </key>
      </timeline>

      <timeline id="1" name="torso">
        <key id="0" time="0">
          <object folder="0" file="2" x="0" y="0" angle="0" scale_x="1" scale_y="1"/>
        </key>
        <key id="1" time="500">
          <object folder="0" file="2" x="0" y="1" angle="0" scale_x="1" scale_y="1"/>
        </key>
      </timeline>

    </animation>
  </entity>
</spriter_data>
```

**关键节点**：

| 节点 | 作用 |
|------|------|
| `<folder>/<file>` | 资源声明——每个 PNG 一个 `<file>`，记录尺寸和 pivot |
| `<entity>` | 一个 build，里面包含 obj_info 和 animation |
| `<obj_info>` | Symbol 元数据（名字、类型、宽高） |
| `<animation>` | 一个 anim，包含 length、looping、mainline 和 timeline |
| `<mainline>` | "对齐时刻"——每个关键帧记录哪些 timeline 在此时有 key |
| `<timeline>` | 单个 Symbol 的运动轨迹，按 time 排列的 key |
| `<object>` | 在某时刻这个 Symbol 的具体属性（位置、角度、缩放、用哪个 file） |

#### 高级技巧 1：手动编辑 .scml

有时候 Spriter UI 不好微调（比如要把所有 head 的 angle 减 5 度），可以**直接用 VSCode 打开 .scml**，搜索 `name="head"` 的 timeline，手动改 `angle` 值。**注意**：

- Spriter 在保存时会重写整个文件——你的注释和格式会丢失
- 修改后要**重新打开 Spriter** 让它"消化"一下，再保存为标准格式
- 改完最好用 git diff 验证只改了你想改的部分

#### 高级技巧 2：scml 之间的资源共享

`<folder>` 段是局部的——**每个 .scml 自带它需要的 PNG 索引**。两个不同的 .scml 即使引用同一个 head-0.png，编译时也会**各自生成 atlas**，导致资源重复。

如果想共享资源，**把它们放在同一个 .scml 项目里**（一个项目可以有多个 entity）。或者用代码层面的 `OverrideSymbol("head", "another_build", "head-0")` 跨 zip 引用。

#### 高级技巧 3：bone 骨骼层级

Spriter 还支持"骨骼"概念——子部件可以**绑定到父骨骼**上，父骨骼旋转时子部件自动跟随旋转。这在 .scml 里表现为 `<bone_ref>` 节点。但是 Klei 在饥荒里**几乎不用 bone**——所有 Symbol 都是平铺的，靠手动 keyframe 实现关节运动。这是为了简化和性能。

> **老手记忆**：`.scml` 是 XML，可以手编辑、可以 git diff。核心结构是 `<folder>(资源) → <entity>(build) → <animation>(anim) → <mainline>(对齐) + <timeline>(每个 Symbol)`。

---

### 13.2.8 老手：八个最容易踩的坑

#### 坑 1：Symbol 名拼写错误，OverrideSymbol 没效果

```lua
inst.AnimState:OverrideSymbol("Swap_Object", "myaxe", "swap_axe")
-- 大写 S！但 Spriter 里 Symbol 名是小写 swap_object！
```

**Symbol 名大小写敏感**。Spriter 里的命名习惯是**全小写下划线分隔**——代码里调用时也必须保持一致。**调试方法**：在控制台 `c_select():GetAnimState():BuildSymbolIsOverridden("swap_object")` 看是否真的覆盖成功。

#### 坑 2：Frame 编号跳号

PNG 文件命名：

```
head-0.png
head-2.png   ← 跳过了 -1！
head-3.png
```

**编译会成功**，但是 `OverrideSymbol("head", "myhero", "head-1")` 调用时**找不到第 1 帧**，AnimState 会**回退到 head-0**——结果脸不对了。

**铁律**：Frame 编号必须**从 0 开始连续**——0、1、2、3……不能跳号。

#### 坑 3：PNG 尺寸超过 atlas 限制

Klei 的 atlas 默认是 1024×1024（DXT5 压缩 1MB）。如果你单张 PNG 就 1024×1024，编译会失败 / 自动拆成多个 atlas / 性能下降。

**最佳实践**：单张 PNG **不要超过 256×256**。如果一定要做大图（比如 boss 的特殊形态），考虑**拆分**成多个 256×256 的 Symbol 拼起来。

#### 坑 4：Anim 名重复

一个 .scml 里**两个 entity 都叫 "idle"** —— 没问题，因为 anim 是 entity 私有的。

但是：**两个 .scml 项目都用同一个 build 名 "wilson"**，加载时**后来的覆盖前面的**——这就是无意中"破坏官方角色"的常见原因。

**铁律**：mod 的 build 名一定要加自己的前缀（`mymod_hero` 而不是 `hero`）。

#### 坑 5：编译报错 "missing pivot"

```
Error: file "head-0.png" has no pivot point
```

每个 PNG 必须有 pivot（中心点）。Spriter 默认会给 0.5/0.5（图片中心），但是**手编辑过 .scml 后可能丢失**。

**修复**：在 .scml 里给每个 `<file>` 加上 `pivot_x="0.5" pivot_y="0.5"`。

#### 坑 6：anim 切换时"瞬移"——缺少过渡帧

代码里 `inst.AnimState:PlayAnimation("attack")` 之后立刻 `PlayAnimation("idle")`——会看到角色**瞬间切换**，没有过渡。

**正确做法**：用 stategraph 的状态机管理 anim 序列，每个 anim 设计 `_pre`/`_loop`/`_pst` 三段，让相邻 anim 的最后一帧和下一帧的第一帧**视觉上能接得上**（即"对位"）。这是美术工作的精细活。

#### 坑 7：导入 PNG 后透明区域变黑

**症状**：PNG 在 Photoshop 里看着是透明的，导入 Spriter 后透明像素变成黑色。

**原因**：保存 PNG 时**未启用透明通道**。Photoshop 里要 `File → Export As → PNG-24` 并勾选 `Transparency`。

#### 坑 8：编译产物 zip 加载后整个角色"扭曲"

**症状**：游戏里角色显示出来，但是**头大、身体小、手脚错位**。

**原因有三**：

1. **PNG 的画布尺寸不一致**——同一个 Symbol 的不同 Frame，画布大小必须一样。head-0 是 64×64，head-1 也必须是 64×64
2. **pivot 不一致**——head-0 的 pivot 是 (0.5, 0.5)，head-1 的 pivot 是 (0.5, 1.0)，结果切换时锚点跳变
3. **scml 中 Symbol 的 z_index 错乱**——头被身体遮住了

**调试**：用 krane.exe 把 zip 解出来看里面的 atlas，肉眼确认每张图位置正确。

> **老手记忆**：Spriter 的坑分三类——**命名约定错**（坑 1、4）、**资源规范错**（坑 2、3、5、7）、**美术细节错**（坑 6、8）。**做新角色时一定要照抄 Klei 的 examples 项目**——它已经把所有约定都示范了。

---

### 13.2 小结：关于 Spriter 你必须记住的

```
            Photoshop 画 PNG（每个部件独立透明图）
                      │
                      ▼
            Spriter 中导入 PNG → 拼装 build
                      │
                      ▼
            Spriter 中创建 anim → 拖关键帧
                      │
                      ▼
            File → Save → 得到 .scml + PNG
                      │
                      ▼
            autocompiler.exe / scml.exe 编译
                      │
                      ▼
            mods/MyMod/anim/myhero.zip
                      │
                      ▼
            代码里 Asset("ANIM", "anim/myhero.zip") 引用
                      │
                      ▼
            inst.AnimState:SetBuild("myhero")
            inst.AnimState:PlayAnimation("idle")
```

**新手核心三句**：Spriter 是 2D 骨骼动画导演工具；三大概念是 Symbol（部件）/ Frame（部件不同帧）/ Anim（动作剧本）；**做角色一定要装 Don't Starve Mod Tools**，它带了 Spriter + 编译器。

**进阶核心三句**：build = entity = 一整套 Symbol；anim 用 `_pre`/`_loop`/`_pst` 三段式可以拉长；Symbol 命名（swap_object、ARM_carry、head 等）是**契约**，必须照抄 Klei 约定。

**老手核心三句**：`.scml` 是 XML，可以手编辑+git diff；`<folder>(资源) → <entity>(build) → <animation>(anim) → <mainline> + <timeline>` 是核心结构；八个常见坑里"命名错"和"资源不一致"占七成，多看 Klei examples 项目就能避免。

下一节 13.3 我们就来拆解 `scml.exe` 和 `autocompiler.exe` 的工作原理，理解从 `.scml + PNG` 到 `xxx.zip` 这一步**编译过程**到底发生了什么，以及如何写脚本批量自动化编译。

---


## 13.3 .scml → .zip 的编译流程

### 本节导读

13.2 我们看到 Spriter 输出的是 `.scml` + 一堆 PNG。13.1 我们看到游戏运行时读的是 `.zip` 里的 `build.bin / anim.bin / atlas-N.tex / atlas-N.xml`。**中间这一步**——把"美术能编辑的源文件"翻译成"GPU 能直接吃的二进制"——就是本节的主角：**编译流水线**。

很多 mod 作者把这一步当黑盒——丢一个 `autocompiler.exe` 上去，等它吐出 zip。但是当编译失败时（明明 PNG 都在却报"找不到 image"），或者编译成功但游戏里贴图错位——你必须能**打开黑盒往里看**。

而幸运的是，**Klei 的整条编译流水线是开源的 Python 脚本**——就装在 `Don't Starve Mod Tools\mod_tools\tools\scripts\` 里。本节就是带着你**逐行**读完这套脚本的核心。

> **新手**从 13.3.1-13.3.3 起步——理解编译这一步到底干了什么、autocompiler 的标准目录约定、第一次跑一次完整编译；**进阶读者**继续看 13.3.4-13.3.6，深入流水线三大阶段（XML 解析 / Atlas 打包 / 二进制编码）、`build.bin` 和 `anim.bin` 的真实字段格式、8 方位 facing 自动识别；**老手**跳到 13.3.7-13.3.8，掌握 Atlas 矩形打包算法、DXT 纹理转换、字符串哈希算法、八个最容易踩的坑。

读完本节，你**可以做到**：① 看 mod 编译报错日志知道错在哪一阶段；② 能写自己的 Python 脚本批量编译多个 .scml；③ 能反推出别人 zip 里的 build.bin 大概是什么样、为什么这么编码。

---

### 13.3.1 快速入门：编译这一步到底在做什么？

#### 第一步：一图看懂"编译"在管线中的位置

```
[美术工作]  画 PNG ─→ Spriter 拼装 ─→ Spriter 保存 ─→ .scml + PNG
                                                       │
                                                       ▼
[编译流水线]                                 ┌─────────────────┐
                                             │ autocompiler.exe │ ←─ 本节主角
                                             └─────────────────┘
                                                       │
                                                       ▼
[运行时]    游戏启动 ←─ Asset 声明 ←─ resolvefilepath ←─ xxx.zip
                                                       └─ atlas-0.tex
                                                       └─ atlas-0.xml
                                                       └─ build.bin
                                                       └─ anim.bin
```

**编译的本质**：把**美术友好**的格式（XML 文本 + 一堆 PNG）转换成**引擎友好**的格式（紧凑二进制 + 压缩纹理图集）。这一步发生在**美术保存之后**、**游戏运行之前**——属于"构建期"，不属于"运行期"。

#### 第二步：编译要做的 5 件事

| 步骤 | 输入 | 输出 | 干了什么 |
|------|------|------|----------|
| 1. **解析 .scml** | 文本 XML | 一棵 XML DOM 树 | 用 Python 的 `xml.dom.minidom` 解析整个 .scml 拿到 `<folder>` / `<entity>` / `<animation>` 节点 |
| 2. **拆分图集（Atlas Packing）**| 一堆零散 PNG | 1~N 张大图 + 每张小图的 UV 矩形 | 用矩形装箱算法把所有 PNG 合并到 1024×1024（或更大）的大图里 |
| 3. **纹理转换** | PNG（无损 RGBA）| `.tex`（DXT5 压缩） | 调用 `textureconverter.Convert` 把 PNG 转成 GPU 直接吃的压缩格式 |
| 4. **build.bin 编码** | XML 中的 entity 节点 | 紧凑二进制 | 把每个 Symbol、每个 Frame 的几何信息（位置、尺寸、UV 坐标）按特定格式写入 `build.bin` |
| 5. **anim.bin 编码** | XML 中的 animation 节点 | 紧凑二进制 | 把每一帧每个 Symbol 的变换矩阵 (m_a, m_b, m_c, m_d, m_tx, m_ty) 写入 `anim.bin` |

最后把这 4 个产物（`atlas-0.tex` + `atlas-0.xml` + `build.bin` + `anim.bin`）打包到一个 `xxx.zip` 里。

#### 第三步：为什么不直接读 .scml？

新手的疑问："`.scml` 不就是 XML 吗？引擎写个 XML 解析器直接读它不就完了？" 不行的三个原因：

**原因一：解析速度**

XML 解析需要**读字符串、构造 DOM、查找节点、转换数值**——全是 CPU 密集的字符串操作。一个角色的 .scml 可能有 5MB，引擎启动时要解析十几个角色，加起来几百毫秒卡顿。而 `build.bin` 是**紧凑二进制**——直接 `memcpy` 到结构体就完事了。

**原因二：图像格式**

`.scml` 引用的是**散装 PNG**——每帧渲染要对 N 张图分别采样纹理。而编译后的 `atlas-0.tex` 是**一张大图**——一次性绑定到 GPU，所有 Symbol 都从这张图里采样，**DrawCall 大幅减少**（13.1 讲过）。

**原因三：哈希预计算**

游戏中调用 `OverrideSymbol("swap_object", ...)` 时，引擎不能每帧拿"swap_object"这个字符串去和所有 Symbol 名字比对——太慢。所以**编译时**就会把所有字符串名字提前哈希成 32 位整数，运行时只比对整数。这就是 `buildanimation.py` 里 `strhash()` 函数的作用：

```84:90:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
def strhash(str, hashcollection):
    hash = 0
    for c in str:
        v = ord(c.lower())
        hash = (v + (hash << 6) + (hash << 16) - hash) & 0xFFFFFFFFL
    hashcollection[hash] = str
    return hash
```

这就是经典的 **DJB-style 哈希**——`hash * 65 + char_code`（因为 `(h<<6) + (h<<16) - h = h*65601`，约等于 65×左右）。**重要**：哈希前先 `.lower()` ——所以 Symbol 名**大小写不敏感**（13.2.8 坑 1 我们说"小心大写"，是因为 Lua 调用方层会出错，但底层哈希是不区分的）。

> **新手记忆**：编译就是**美术格式 → 引擎格式**的翻译。这一步做了 5 件事——解析 XML / 打包图集 / 转纹理 / 编 build.bin / 编 anim.bin。**为了速度、显存效率和哈希预计算**，引擎不直接读 .scml。

---

### 13.3.2 快速入门：autocompiler 的标准目录约定

#### 第一步：mod 目录的标准布局

打开 Steam 的 "Don't Starve Mod Tools"，会看到它给 mod 推荐的目录长这样：

```
mods/
└── MyMod/                   ← mod 根目录
    ├── modinfo.lua
    ├── modmain.lua
    ├── scripts/
    │   └── prefabs/
    │       └── myitem.lua   ← prefab 代码
    │
    ├── exported/            ← Spriter 源工程目录（autocompiler 输入）
    │   └── myitem/
    │       ├── myitem.scml  ← Spriter 项目
    │       ├── myitem.png   ← 部件 PNG
    │       └── ...
    │
    └── anim/                ← 编译产物目录（autocompiler 输出）
        └── myitem.zip       ← 编译生成的动画包
```

**关键约定**：

| 目录 | 性质 | 内容 |
|------|------|------|
| `exported/` | **源工程**（开发期可见，发布时**通常不打包**）| `.scml` + PNG 等 Spriter 工程文件 |
| `anim/` | **编译产物**（运行期被加载）| 编译生成的 `.zip` 文件 |

`autocompiler.exe` 的工作就是：**遍历 `exported/` 下所有 .scml，逐个编译，把产物写到 `anim/`**。

#### 第二步：autocompiler 调用的命令

最简单的用法（在 mod 根目录下）：

```bash
cd mods/MyMod/
"C:\Program Files (x86)\Steam\steamapps\common\Don't Starve Mod Tools\mod_tools\autocompiler.exe"
```

`autocompiler.exe` 会**自动**：

1. 在当前目录（mod 根）下搜 `exported/` 子目录
2. 在 `exported/` 下搜所有 `.scml` 文件
3. 对每个 `.scml` 调用底层的 Python 脚本（`buildanimation.py`）编译
4. 把生成的 `.zip` 放到 `anim/` 目录
5. 同时**还会编译 lua**（把 `scripts/*.lua` 编译成字节码以加快加载）

#### 第三步：autocompiler 的内部工具链

`autocompiler.exe` 是一个 C++ 调度器，它内部会调用 Python 27（Klei 自带的）来执行多个脚本。在 `Don't Starve Mod Tools/mod_tools/tools/scripts/` 下你会看到这些 `.py` 文件：

| 脚本 | 作用 |
|------|------|
| **`buildanimation.py`** | **核心**：把 `.scml` 解析后的中间 zip → `build.bin + anim.bin + atlas.tex` 的最终 zip |
| `compilemodel.py` | 编译 3D model（饥荒里几乎不用——主要是 .obj 形式的特殊物体）|
| `optimizeimage.py` | 找出 PNG 的"非透明区域"——用于 Atlas 打包时的 tight bbox |
| `check_assets.py` | 验证 prefabs.xml 中声明的所有资源是否真的存在 |
| `CompileLuaDirectory.py` | 把整个 lua 目录批量编译成字节码 |

整条流水线由 `autocompiler.exe` 协调——你不直接调用 Python 脚本，但**理解它们能让你看懂错误日志**。

#### 第四步：神话 mod 的"反向工程"目录

观察项目里 `mods/联机版mod/神话未加密/anim_decode/`：

```
神话未加密/
└── anim_decode/
    └── zip_output/
        ├── peach/
        │   ├── peach.scml         ← 反编译出来的 scml
        │   └── peach/peach-0.png  ← 反编译出来的 PNG
        ├── nian_mount/
        ├── ...（200+ 个）
```

这是 mod 作者**用 `krane.exe`（13.2.6 提到过）反向**得到的——把已发布的 zip "解开"成可编辑源工程。这告诉我们：**编译过程是有损但可逆**——krane 重新走一遍 buildanimation.py 的反向逻辑就能还原 .scml。

> **新手记忆**：autocompiler 约定 `exported/` 放源工程、`anim/` 放产物。它本质上是个调度器，真正干活的是 Python 脚本（特别是 `buildanimation.py`）。

---

### 13.3.3 快速入门：实际跑一次完整编译

#### 第一步：准备最小可编译项目

新建目录 `mods/MyMod/exported/test_icon/`，往里放：

```
test_icon/
├── test_icon.scml
└── test_icon-0.png   ← 64×64 透明 PNG
```

最小 `.scml` 内容（手写而不是 Spriter 导出，便于学习）：

```xml
<?xml version="1.0" ?>
<spriter_data scml_version="1.0" generator="BrashMonkey Spriter" generator_version="b5">
    <folder id="0" name="test_icon">
        <file id="0" name="test_icon-0.png" width="64" height="64" pivot_x="0.5" pivot_y="0.5"/>
    </folder>
    <entity id="0" name="test_icon">
        <animation id="0" name="idle" length="100">
            <mainline>
                <key id="0" time="0">
                    <object_ref id="0" timeline="0" key="0" z_index="0"/>
                </key>
            </mainline>
            <timeline id="0" name="test_icon">
                <key id="0" time="0">
                    <object folder="0" file="0" x="0" y="0" angle="0" scale_x="1" scale_y="1"/>
                </key>
            </timeline>
        </animation>
    </entity>
</spriter_data>
```

它声明了：1 个 build (`test_icon`)、1 个 Symbol (`test_icon`)、1 个 anim (`idle`)、1 个关键帧。

#### 第二步：跑 autocompiler

```bash
cd mods/MyMod
"C:\Program Files (x86)\Steam\steamapps\common\Don't Starve Mod Tools\mod_tools\autocompiler.exe"
```

正常情况下输出会像这样：

```
Compiling: exported/test_icon/test_icon.scml
[Atlas Packing] 1 image(s), packed into 1 atlas(es)
[Texture Convert] atlas-0.tex (DXT5, 64x64)
[Build Encode] build.bin (1 symbol, 1 frame)
[Anim Encode] anim.bin (1 animation, 1 frame)
Compiled: anim/test_icon.zip (size: 4.8 KB)
```

成功的话，`anim/test_icon.zip` 就出现了。**用解压软件打开**它，里面应该有 4 个文件：

```
test_icon.zip
├── atlas-0.tex
├── atlas-0.xml      ← 部分 mod tools 版本不会输出 .xml，由 .bin 里的元数据替代
├── build.bin
└── anim.bin
```

#### 第三步：验证编译结果

在 mod 的 `modmain.lua` 里声明：

```lua
Assets = {
    Asset("ANIM", "anim/test_icon.zip"),
}

PrefabFiles = {
    "test_icon",
}
```

在 `scripts/prefabs/test_icon.lua` 里：

```lua
local assets = {
    Asset("ANIM", "anim/test_icon.zip"),
}

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst.AnimState:SetBank("test_icon")
    inst.AnimState:SetBuild("test_icon")
    inst.AnimState:PlayAnimation("idle", true)
    return inst
end

return Prefab("test_icon", fn, assets)
```

进游戏，控制台 `c_spawn("test_icon")` —— 应该能看到那张 64×64 的图浮现在世界上。

#### 第四步：编译失败怎么办？

最常见的三种错误日志：

| 错误信息 | 含义 | 解决 |
|---------|------|------|
| `Could not open file: xxx.png` | scml 中 `<file name="xxx.png">` 引用的 PNG 不存在 | 检查 PNG 文件名拼写、是否在 .scml 同目录 |
| `Image size exceeds atlas max (1024)` | 单张 PNG 超过 1024×1024 | 缩小 PNG 或拆分成多个 Symbol |
| `KeyError: "z_index"` | scml 的 mainline 里某个 object_ref 缺 z_index 属性 | 在 Spriter 里重新保存（会自动补全），或手编辑添加 |

调试技巧：autocompiler 跑失败时**保留终端窗口**——日志的最后几行会指出具体的 .scml 文件和报错行号。

> **新手记忆**：编译 = 把 scml + PNG 喂给 autocompiler，得到 zip。**第一次失败正常**，照着错误日志查 PNG 名字、scml 属性、文件大小三个方向就能定位。

---

### 13.3.4 进阶：编译流水线的三大阶段

打开 `Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py` 这个 562 行的 Python 脚本——它就是真正干活的编译器。让我们按它的执行顺序拆成**三大阶段**。

#### 阶段一：XML 解析与节点遍历

入口在 `if __name__ == "__main__":` 块（481-559 行）。简化后流程：

```python
# 第一步：参数解析
parser = argparse.ArgumentParser(...)
parser.add_argument('infile', ...)             # 输入文件（中间 zip）
parser.add_argument('--platform', ...)         # 'opengl' / 'pc' / ...
parser.add_argument('--textureformat', ...)    # 默认 'bc3'（DXT5）
parser.add_argument('--hardalphatextureformat', ...)  # 默认 'bc1'（DXT1）

# 第二步：打开输入 zip（autocompiler 中间产物）
zip_file = zipfile.ZipFile(results.infile, "r")

# 第三步：分别读取 animation.xml 和 build.xml
if "animation.xml" in zip_file.namelist():
    animxml = zip_file.read("animation.xml")
    ExportAnim(endianstring, animxml, outzip, ...)

if "build.xml" in zip_file.namelist():
    buildxml = zip_file.read("build.xml")
    ExportBuild(endianstring, zip_file, buildxml, outzip, ...)

# 第四步：把 outzip 写到 mod/anim/xxx.zip
outfilename = data_path + "\\anim\\" + base_name + ".zip"
```

**重要发现**：`buildanimation.py` 的输入**不是 .scml 本身**——而是 autocompiler 的"前置脚本"已经把 .scml 转成了一个**中间 zip**（包含 `animation.xml` + `build.xml` + 所有 PNG）。这个中间 zip 才是 `buildanimation.py` 的输入。

中间 zip 长这样（你可以从 autocompiler 的临时目录里抓到）：

```
intermediate.zip
├── animation.xml   ← 从 .scml 的 animation 节点重整后的 XML
├── build.xml       ← 从 .scml 的 entity 节点重整后的 XML
├── face/face-0.png
├── face/face-1.png
├── ...（所有原始 PNG）
```

为什么要拆分成 `animation.xml` + `build.xml`？因为 .scml 把 build（骨架）和 anim（动作）放一起，但是引擎运行时**只读 build 也能渲染静态模型**——把它们分开方便引擎按需加载。

#### 阶段二：图集打包（Atlas Packing）

`ExportBuild` 函数（383-476 行）的开头干的事：

```383:399:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
def ExportBuild(endianstring, inzip, buildxml, outzip, antialias, platform, textureformat, hardalphatextureformat, force, ignore_exceptions):
    hashcollection = {}

    doc = xml.dom.minidom.parseString(buildxml)
    
    imnames = {frame.attributes["image"].value.encode('ascii') for frame in doc.getElementsByTagName("Frame")}

    images = []

    for imname in imnames:
        img = Image.open( StringIO( inzip.read( imname + ".png" ) ) )
        img.name = imname
        img.regions = optimizeimage.GetImageRegions(img, 32)

        images.append( img )
    
    atlases = AtlasImages( images, "atlas", outzip, 2048, antialias, platform, textureformat, hardalphatextureformat, force, ignore_exceptions)
```

**核心步骤**：

1. **遍历所有 `<Frame>`** —— 收集每帧引用的 PNG 名字（`imname`）
2. **去重**——用 Python 的 `set` 推导式 `{frame.attributes["image"].value ... for frame in ...}`，自动去掉重复 PNG
3. **加载 PIL Image** —— 把 zip 里每张 PNG 解压成 PIL `Image` 对象
4. **找非透明区域** —— `optimizeimage.GetImageRegions(img, 32)` 把 PNG 切成"alpha 区域"和"opaque 区域"——后续打包时只装真正有像素的部分（透明背景不占空间）
5. **打包到大图** —— `AtlasImages(...)` 是核心，调用 `klei.atlas.Atlas` 库做矩形装箱

`AtlasImages` 函数（242-304 行）调用 `atlas.Atlas` 后的工作：

```253:262:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
            for idx, atlasdata in atlases.items():
                tex_filename = atlasdata.mips[0].name + ".tex"
                dest_filename = os.path.join( temp_dir, tex_filename )

                mip_filenames = [ os.path.join( temp_dir, atlasdata.mips[0].name + "_" + str( mip_idx ) + ".png" ) for mip_idx in range( len( atlasdata.mips ) ) ]

                valid_filenames = []
                textureformat = textureformat if antialias else hardalphatextureformat
```

**注意 `mips`**——每个 atlas 会生成多个 **mipmap 层级**：原图、1/2 缩小、1/4 缩小、1/8 缩小……GPU 在远距离观察物体时自动用小图，节省带宽。这就是为什么你看到大型 mod 的 `.tex` 文件比 PNG 总和还大——不是没压缩，而是多了 4-5 个缩小版本。

#### 阶段三：纹理转换 + 二进制编码

```285:296:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
                textureconverter.Convert(
                        src_filenames=valid_filenames,
                        dest_filename=dest_filename,
                        texture_format=textureformat,
                        platform=platform,
                        force=force,
                        generate_mips=True,
                        ignore_exceptions=ignore_exceptions)

                info = zipfile.ZipInfo( tex_filename, date_time=ZIP_ZERO_TIME )
                info.compress_type = zipfile.ZIP_DEFLATED
                outzip.writestr( info, open( dest_filename, 'rb' ).read() )
```

**`textureconverter.Convert`** 是关键的"PNG → .tex"调用——它**不是 Python 实现的**，而是 Klei 自带的 **C++ DLL**（`textureconverter.dll`）。这个 DLL 内部做：

- 把多张 mip 的 PNG 编码成 **DXT5（BC3）压缩纹理**
- 写出 `.tex` 文件——这是 Klei 自定义的容器格式：`KTEX` 4 字节魔数 + 版本 + 每个 mip 的尺寸/数据
- 不同平台用不同压缩格式（PC/OpenGL 用 DXT5；Switch 可能用 BC7；某些手机平台用 ETC2）

写完 atlas 后，`ExportBuild` 紧接着写 `build.bin`（406-476 行）。这部分二进制格式我们 13.3.5 单独拆。然后 `ExportAnim` 写 `anim.bin`（13.3.6）。

最后，整个流水线的产物**就是一个 zip**（在 Python 里是一个内存 StringIO），最后落盘到 `data_path + "\\anim\\" + base_name + ".zip"`。

#### 一图总结流水线

```
intermediate.zip                       outzip (内存)
├── animation.xml                      ├── atlas-0.tex      ← 阶段二/三
├── build.xml                          ├── atlas-1.tex
├── face/face-0.png                    ├── ...
├── face/face-1.png    ─[ExportBuild]─→├── build.bin       ← 阶段三
├── ...                ─[ExportAnim ]─→└── anim.bin        ← 阶段三
                                              │
                                              ▼
                                        anim/xxx.zip
```

> **进阶记忆**：编译三阶段是 **解析 → 打包 → 编码**。`buildanimation.py` 的核心函数是 `ExportBuild`（建模+打包+纹理）和 `ExportAnim`（动作编码）。Atlas 打包用 PIL 加 Klei 自家的 `klei.atlas` 库，纹理转换调 C++ DLL。

---

### 13.3.5 进阶：build.bin 二进制格式拆解

`buildanimation.py` 第 23-46 行的注释**官方公开了 build.bin 的字节布局**：

```23:46:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
ZIP_ZERO_TIME = ( 1980, 0, 0, 0, 0, 0 )
BUILDVERSION = 6
#BUILD format 6
# 'BILD'
# Version (int)
# total symbols;
# total frames;
# build name (int, string)
# num materials
#   material texture name (int, string)
#for each symbol:
#   symbol hash (int)
#   num frames (int)
#       frame num (int)
#       frame duration (int)
#       bbox x,y,w,h (floats)
#       vb start index (int)
#       num verts (int)

# num vertices (int)
#   x,y,z,u,v,w (all floats)
#
# num hashed strings (int)
#   hash (int)
#   original string (int, string)
```

#### 第一步：文件头

```python
outfile.write(struct.pack(endianstring + 'cccci', 'B', 'I', 'L', 'D', BUILDVERSION))
```

- 4 字节魔数 `'BILD'`（用于校验文件类型）
- 1 个 32 位 int 版本号（当前是 6）

`endianstring` 默认是 `'<'`（小端字节序）——除非编译目标是大端平台（如某些主机），才会传 `--bigendian`。

#### 第二步：总体计数

```408:413:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
    outfile.write(struct.pack(endianstring + 'I', len(doc.getElementsByTagName("Symbol"))))
    outfile.write(struct.pack(endianstring + 'I', len(doc.getElementsByTagName("Frame"))))

    buildname = doc.getElementsByTagName("Build")[0].attributes["name"].value.encode('ascii')
    buildname = os.path.splitext(buildname)[0]
    outfile.write(struct.pack(endianstring + 'i' + str(len(buildname)) + 's', len(buildname), buildname))
```

- 4 字节 `total_symbols` —— 总 Symbol 数
- 4 字节 `total_frames` —— 所有 Symbol 的所有 Frame 总数
- `int + string` —— **build 名字**：先写一个 4 字节长度，再写字符串本身

**关键细节**：`buildname = os.path.splitext(buildname)[0]` —— **去掉扩展名**。所以你写 `build.xml` 时 build 名是 `build`（如果 .scml 写 `name="myhero"` 就直接是 `myhero`）。

#### 第三步：atlas 索引表

```415:422:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
    #write out the number of atlases:
    outfile.write(struct.pack(endianstring + 'I', len(atlases)))
    for atlas_idx in range( len( atlases ) ):
        atlasdata = atlases[ atlas_idx ]
        mip = atlasdata.mips[0]
        name = mip.name + ".tex"

        outfile.write(struct.pack(endianstring + 'i' + str(len(name)) + 's', len(name), name))
```

- 4 字节 `num_materials`（实际就是 atlas 数量）
- 每个 atlas 一个 `int + string`：纹理文件名（比如 `"atlas-0.tex"`）

引擎读到这一段就知道："要渲染这个 build，得先加载 `atlas-0.tex`、`atlas-1.tex`……"

#### 第四步：每个 Symbol 的数据

```424:459:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
    symbol_nodes = sorted(doc.getElementsByTagName("Symbol"), key=lambda x: strhash(x.attributes["name"].value.encode('ascii'), hashcollection))

    for symbol_node in symbol_nodes:#doc.getElementsByTagName("Symbol"):
        symbolname = symbol_node.attributes["name"].value.encode('ascii')
        outfile.write(struct.pack(endianstring + 'I', strhash(symbolname, hashcollection)))
        outfile.write(struct.pack(endianstring + 'I', len(symbol_node.getElementsByTagName("Frame"))))

        for frame_node in symbol_node.getElementsByTagName("Frame"):
            framenum = int(frame_node.attributes["framenum"].value)
            duration = int(frame_node.attributes["duration"].value)

            w = float(frame_node.attributes["w"].value)
            h = float(frame_node.attributes["h"].value)
            x = float(frame_node.attributes["x"].value)
            y = float(frame_node.attributes["y"].value)

            imagename = frame_node.attributes["image"].value.encode('ascii')

            xoff = x - w/2
            yoff = y - h/2
            z = 0

            alphaidx = len(alphaverts)
            alphacount = AddVertsToVB( imagename, "alpha", atlases, xoff, yoff, z, alphaverts )
            
            #opaqueidx = len(opaqueverts)
            alphacount += AddVertsToVB( imagename, "opaque", atlases, xoff, yoff, z, alphaverts )

            outfile.write(struct.pack(endianstring + 'I', framenum))
            outfile.write(struct.pack(endianstring + 'I', duration))
            outfile.write(struct.pack(endianstring + 'ffff', x,y,w,h))
            
            outfile.write(struct.pack(endianstring + 'I', alphaidx))
            outfile.write(struct.pack(endianstring + 'I', alphacount))
```

每个 Symbol 写：
- 4 字节 `symbol_hash` —— Symbol 名的 32 位哈希（**不是字符串**！）
- 4 字节 `num_frames` —— 这个 Symbol 的 Frame 数

每个 Frame 写：
- `framenum` (int) —— 帧编号（对应 .scml 里 `head-0`、`head-1` 的 0、1）
- `duration` (int) —— 帧持续时间（毫秒）
- `bbox: x, y, w, h` (4 个 float) —— 这帧图的边界盒（用于碰撞检测和居中）
- `vb_start_index` (int) —— 顶点缓冲区的起始索引
- `num_verts` (int) —— 用了多少顶点

**注意**：Symbol 在写入前先 `sorted(... key=strhash)` —— **按哈希值排序**！这样运行时引擎可以**二分查找** Symbol——`O(log N)` 而不是 `O(N)`。

#### 第五步：顶点缓冲区（Vertex Buffer）

```461:466:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
    outfile.write(struct.pack(endianstring + 'I', len(alphaverts)))
    for vert in alphaverts:
        outfile.write(struct.pack(endianstring + 'ffffff', vert.x, vert.y, vert.z, vert.u, vert.v, vert.w))
```

- 4 字节 `num_vertices`
- 每个顶点 6 个 float：`x, y, z, u, v, w` —— 位置 (xyz) + 纹理坐标 (uv) + 采样器索引 (w)

每个 Frame 在 atlas 里是一个矩形，2 个三角形 = 6 个顶点。如果 Frame 既有"alpha 区域"又有"opaque 区域"，可能会切成多个矩形（每块矩形 6 顶点）—— `AddVertsToVB` 函数（309-381 行）就在干这个。

#### 第六步：哈希反查表

```468:472:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
    #write out a lookup table of the pre-hashed strings
    outfile.write(struct.pack(endianstring + 'I', len(hashcollection)))
    for hash_idx,name in hashcollection.iteritems():
        outfile.write(struct.pack(endianstring + 'I', hash_idx))
        outfile.write(struct.pack(endianstring + 'i' + str(len(name)) + 's', len(name), name))
```

- 4 字节 `num_hashed_strings`
- 每个：`hash` (int) + `original_string` (int + string)

这是**反查表**——运行时如果要打印"Symbol head 的位置"，引擎拿哈希值反查这张表就能找回字符串。**调试用**——发布版可能会被剥离。

#### 整体布局图

```
build.bin
├── 'BILD' (4字节魔数) + Version (4字节)
├── total_symbols (4) + total_frames (4)
├── buildname (4字节长度 + N字节字符串)
├── num_atlases (4) + 每个 atlas: name (4+N)
├── 每个 Symbol（按哈希值排序）：
│   ├── symbol_hash (4)
│   ├── num_frames (4)
│   └── 每个 Frame：framenum(4) + duration(4) + bbox(16) + vb_start(4) + num_verts(4)
├── num_vertices (4) + 每个顶点 (24)
└── 反查表：num_strings (4) + 每条：hash(4) + len(4) + chars
```

> **进阶记忆**：build.bin = `'BILD' + 计数 + buildname + atlas表 + Symbols + 顶点 + 哈希反查表`。所有名字都被预哈希成 32 位 int，按哈希排序好让二分查找。

---

### 13.3.6 进阶：anim.bin 二进制格式 + 8 方位 facing 编码

#### 第一步：anim.bin 文件头

```48:72:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
ANIMVERSION = 4
#ANIM format 4
# 'ANIM'
# Version (int)
# total num element refs (int)
# total num frames (int)
# total num events (int)
# Numanims (int)
#   animname (int, string)
#   validfacings (byte bit mask) (xxxx dlur)
#   rootsymbolhash int
#   frame rate (float)
#   num frames (int)
#       x, y, w, h : (all floats)
#       num events(int)
#           event hash
#       num elements(int)
#           symbol hash (int)
#           symbol frame (int)
#           folder hash (int)
#           mat a, b, c, d, tx, ty, tz: (all floats)
#
# num hashed strings (int)
#   hash (int)
#   original string (int, string)
```

总体结构和 build.bin 类似，但一个文件可以有**多个 anim**（idle / walk / attack...）。

#### 第二步：方位（facing）自动识别——这是亮点

`buildanimation.py` 第 110-164 行**通过解析 anim 名后缀**，自动推断这个 anim 适用于哪些方位：

```110:164:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
    def LocalExport( anim_node ):
        name = anim_node.attributes["name"].value.encode('ascii')
        
        dirs = (re.search("(.*)_up\Z", name),
                re.search("(.*)_down\Z", name),
                re.search("(.*)_side\Z", name),
                re.search("(.*)_left\Z", name),
                re.search("(.*)_right\Z", name),
                re.search("(.*)_upside\Z", name),
                re.search("(.*)_downside\Z", name),
                re.search("(.*)_upleft\Z", name),
                re.search("(.*)_upright\Z", name),
                re.search("(.*)_downleft\Z", name),
                re.search("(.*)_downright\Z", name),
                re.search("(.*)_45s\Z", name),
                re.search("(.*)_90s\Z", name))
        
        facingbyte = FACING_RIGHT | FACING_LEFT | FACING_UP | FACING_DOWN | FACING_UPLEFT | FACING_UPRIGHT | FACING_DOWNLEFT | FACING_DOWNRIGHT
        
        if dirs[0]:
            name = dirs[0].group(1)
            facingbyte = FACING_UP
        elif dirs[1]:
            name = dirs[1].group(1)
            facingbyte = FACING_DOWN
        elif dirs[2]:
            name = dirs[2].group(1)
            facingbyte = FACING_LEFT | FACING_RIGHT
        elif dirs[3]:
            name = dirs[3].group(1)
            facingbyte = FACING_LEFT
        ...
```

**facing 位掩码定义**（75-82 行）：

```75:82:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
FACING_RIGHT = 1<<0
FACING_UP = 1<<1
FACING_LEFT = 1<<2
FACING_DOWN = 1<<3
FACING_UPRIGHT = 1<<4
FACING_UPLEFT = 1<<5
FACING_DOWNRIGHT = 1<<6
FACING_DOWNLEFT = 1<<7
```

**含义**：anim 的实际名字 = "去掉后缀后的真名"，但是会带一个**位标志**告诉引擎"这个动画只在某些方向播"。

举例：

| Spriter 里的 anim 名 | 实际存的名 | facingbyte |
|---------------------|-----------|------------|
| `idle` | `idle` | 全部方向（0xFF）|
| `walk_loop_up` | `walk_loop` | `FACING_UP` |
| `walk_loop_down` | `walk_loop` | `FACING_DOWN` |
| `walk_loop_side` | `walk_loop` | `FACING_LEFT | FACING_RIGHT` |
| `attack_45s` | `attack` | 4 个对角线方向 |
| `attack_90s` | `attack` | 4 个正方向 |

**实战意义**：玩家面朝上走路时引擎播 `walk_loop` 但 facing=UP——它**自动**找到 `walk_loop_up` 这个 anim 数据。代码里调用 `PlayAnimation("walk_loop")` 不需要关心方向，引擎自己处理。

这就是 **`SetFacing` 方法**的底层支持（13.4 会讲）。**Wilson、Wendy 等玩家角色**有大量 `_up`/`_down`/`_side` 后缀的 anim，对应"上视图、下视图、侧视图"三套素材。

#### 第三步：每个 anim 的 frame 数据

```174:193:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
        for frame_node in anim_node.getElementsByTagName("frame"):
            outfile.write(struct.pack(endianstring + 'ffff',
                float(frame_node.attributes["x"].value),
                float(frame_node.attributes["y"].value),
                float(frame_node.attributes["w"].value),
                float(frame_node.attributes["h"].value)))
            num_events = len(frame_node.getElementsByTagName("event"))                
            outfile.write(struct.pack(endianstring + 'I', num_events))
            
            for event_node in frame_node.getElementsByTagName("event"):
                outfile.write(struct.pack(endianstring + 'I', strhash(event_node.attributes["name"].value.encode('ascii'), hashcollection)))
            
            elements = frame_node.getElementsByTagName("element")
            try:
                elements = sorted(elements, key=get_z_index)
            except:
                pass
            
            num_elements = len(elements)
            outfile.write(struct.pack(endianstring + 'I', num_elements))
```

每帧写：
- `bbox: x, y, w, h` (4 float) —— 当前帧的边界盒（对碰撞检测用）
- `num_events` (4 byte) —— 当前帧上有多少 meta event
- 每个 event 一个 4 字节哈希
- `num_elements` (4 byte) —— 当前帧渲染了多少个 Symbol

注意 `elements = sorted(elements, key=get_z_index)` —— **按 z_index 排序**！这就是 13.2.4 讲的"渲染顺序"——z_index 越大越靠后，**排序后写入的顺序就是渲染顺序**。

#### 第四步：每个 element 的变换矩阵

```196:212:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
            eidx = 0
            for element_node in elements:
                outfile.write(struct.pack(endianstring + 'I', strhash(element_node.attributes["name"].value.encode('ascii'), hashcollection)))
                outfile.write(struct.pack(endianstring + 'I', int(element_node.attributes["frame"].value)))
                layername = element_node.attributes["layername"].value.encode('ascii').split('/')[-1]
                outfile.write(struct.pack(endianstring + 'I', strhash(layername, hashcollection)))
                        
                z = (eidx/float(num_elements)) * float(EXPORT_DEPTH) - EXPORT_DEPTH*.5
                outfile.write(struct.pack(endianstring + 'fffffff',
                    float(element_node.attributes["m_a"].value),
                    float(element_node.attributes["m_b"].value),
                    float(element_node.attributes["m_c"].value),
                    float(element_node.attributes["m_d"].value),
                    float(element_node.attributes["m_tx"].value),
                    float(element_node.attributes["m_ty"].value),
                    z))

                eidx += 1
```

每个 element 写：
- `symbol_hash` (4 byte) —— Symbol 名哈希（用来从 build.bin 找几何）
- `frame_num` (4 byte) —— 用 Symbol 的第几帧（比如 head-2）
- `folder_hash` (4 byte) —— layername 的哈希（用于 Override）
- **变换矩阵 6 个 float**：`m_a, m_b, m_c, m_d, m_tx, m_ty` 加一个 z 深度

**关于 2D 仿射变换矩阵**：

```
[ m_a  m_c  m_tx ]   [ x ]   [ new_x ]
[ m_b  m_d  m_ty ] · [ y ] = [ new_y ]
[ 0    0    1    ]   [ 1 ]   [ 1     ]
```

- `m_a = scale_x * cos(angle)` 
- `m_b = scale_x * sin(angle)`
- `m_c = -scale_y * sin(angle)`
- `m_d = scale_y * cos(angle)`
- `m_tx, m_ty` = 平移

这就是为什么 Spriter 时间轴可以同时存储**位置 + 旋转 + 缩放**——只用 4 个 float（旋转和缩放糅合在 a/b/c/d 里）。

`z = (eidx/num_elements) * EXPORT_DEPTH - EXPORT_DEPTH*.5` —— **自动深度分配**。每个 element 在 [-5, +5] 范围内均分一个 z 值，这样 GPU 的 z-buffer 能正确处理遮挡。`EXPORT_DEPTH = 10` 在第 95 行。

> **进阶记忆**：anim.bin 的 frame 数据 = `bbox + events + 一组 elements`。每个 element 的"变换"用 6 个 float 表示 2D 仿射变换矩阵。anim 名带 `_up/_down/_side` 等后缀会被自动识别成 facing 位掩码——这是玩家方向动画的底层。

---

### 13.3.7 老手：Atlas 打包算法、纹理转换、字符串哈希

#### 第一步：Atlas 矩形打包算法（Bin Packing）

`buildanimation.py` 调用的 `klei.atlas.Atlas` 是个矩形装箱实现。**问题描述**：给一堆不同尺寸的小矩形，把它们塞进一张 1024×1024 的大矩形（且不重叠），求最紧凑的布局。

这是经典的 **NP-hard 问题**——只有近似算法。Klei 的实现走的是 **Guillotine 算法**或 **MaxRects 算法**：

**Guillotine 算法**（简化思想）：
1. 把所有图按高度（或面积）从大到小排序
2. 用"切刀"遍历大图——每塞下一块小图，把剩余空间切成两个矩形（横切 + 竖切）
3. 后续小图在这些剩余矩形中找最适合的塞

**MaxRects 算法**（性能更好）：
- 维护一个"自由矩形列表"
- 每塞下一块就更新列表（合并相邻的、移除被覆盖的）
- 启发式选择"最佳适应"位置（剩余面积最小、贴边最多等）

**Klei 的特殊优化**——`optimizeimage.GetImageRegions` 把每张 PNG 切成"alpha 区域 + opaque 区域"——透明区域不参与打包。这让"周围一圈透明的法杖图"能省下大量 atlas 空间。

#### 第二步：纹理压缩——DXT5 (BC3) 是什么

```python
parser.add_argument('--textureformat', default='bc3')          # 默认 BC3 = DXT5
parser.add_argument('--hardalphatextureformat', default='bc1') # 硬 alpha = BC1 = DXT1
```

**BC3 / DXT5**：
- 4×4 像素块 = 16 字节
- 颜色：用 2 个 16 位 RGB 端点 + 16 个 2 位查找表（4×4 个像素，每个用 2 位查找一个 4 级颜色表）
- alpha：用 2 个 8 位 alpha 端点 + 16 个 3 位查找表
- **压缩比 4:1**（原 RGBA 32 位 = 64 字节/4×4 块，BC3 = 16 字节）

**BC1 / DXT1**：
- 没有 alpha 通道
- 每个 4×4 块只有 8 字节（**压缩比 8:1**）
- 用于"硬 alpha"——alpha 只有 0 或 255 的图（剪贴画类型），单独处理 alpha 浪费空间

**`antialias` 参数**——`textureformat = textureformat if antialias else hardalphatextureformat`（第 262 行）：如果用户传 `--skipantialias`，那么所有图都用 BC1（硬 alpha 模式）——产物体积更小但边缘锯齿明显。

#### 第三步：字符串哈希算法

```84:90:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
def strhash(str, hashcollection):
    hash = 0
    for c in str:
        v = ord(c.lower())
        hash = (v + (hash << 6) + (hash << 16) - hash) & 0xFFFFFFFFL
    hashcollection[hash] = str
    return hash
```

**算法分析**：
- 每个字符 `c` 转成它的 ASCII 码 `v`
- 公式 `hash = v + h*65 + h*65536 - h = v + h * (65 + 65536 - 1) = v + h * 65600`
- 等价于 `hash = hash * 65600 + v`
- 最后 `& 0xFFFFFFFF` 截断到 32 位

**哈希碰撞概率**：32 位空间有 4×10⁹ 个值，一般 mod 顶多几千个 Symbol——按生日悖论，碰撞概率约 `n²/(2*4×10⁹)` ≈ 千分之一以下，可以接受。

**重要细节**：先 `c.lower()` ——所以 `strhash("Wilson") == strhash("wilson") == strhash("WILSON")`。这就是为什么饥荒里 Symbol 名**大小写不敏感**（在引擎层）—— 但 13.2.8 我们说"小心大写"，是因为 Lua 调用层做了字符串相等比较，所以最佳实践还是统一小写。

如果哈希碰撞了会怎样？看代码：`hashcollection[hash] = str` —— **后写的会覆盖先写的**！所以如果你的 mod 不幸有两个 Symbol 哈希碰撞，运行时反查表会丢失其中一个 Symbol 的字符串。但是**渲染不受影响**——因为渲染只用哈希。

#### 第四步：`AddVertsToVB` 的 UV 计算

```345:380:Don't Starve Mod Tools/mod_tools/tools/scripts/buildanimation.py
            for region in src_regions:
                assert region.x <= img.size[0]
                assert region.y <= img.size[1]
                assert region.x + region.w <= img.size[0]
                assert region.y + region.h <= img.size[1]

                left = xoff + region.x
                right = left + region.w

                top = yoff + region.y
                bottom = top + region.h

                z = zoff

                umin = max( 0.0, min( 1.0, ( dest_bbox.x + region.x ) / imsizex ) )
                umax = max( 0.0, min( 1.0, ( dest_bbox.x + region.x + region.w ) / imsizex ) )
                vmin = max( 0.0, min( 1.0, 1 - ( dest_bbox.y + region.y ) / imsizey ) )
                vmax = max( 0.0, min( 1.0, 1 - ( dest_bbox.y + region.y + region.h ) / imsizey ) )

                ...

                VB.append(Vert(left, top , z, umin, vmin, sampler))
                VB.append(Vert(right, top, z, umax, vmin, sampler))
                VB.append(Vert(left, bottom, z, umin, vmax, sampler))
                VB.append(Vert(right, top, z, umax, vmin, sampler))
                VB.append(Vert(right, bottom, z, umax, vmax, sampler))
                VB.append(Vert(left, bottom, z, umin, vmax, sampler))
```

**核心公式**：
- `umin = (dest_bbox.x + region.x) / atlas_width` —— UV 坐标 (0~1)
- `vmin = 1 - (dest_bbox.y + region.y) / atlas_height` —— **注意减 1**！图形学里 V 轴是上下翻转的（PNG 原点在左上，OpenGL 纹理原点在左下）

**6 个顶点**——Direct3D / OpenGL 的标准三角带画一个矩形：
```
左上 → 右上 → 左下     ← 第一个三角形
右上 → 右下 → 左下     ← 第二个三角形
```

这就是为什么 build.bin 里每个 Frame 的 `num_verts` 通常是 6 的倍数——每块矩形 6 顶点。

#### 第五步：跨平台编译参数

| 平台 | textureformat | platform |
|------|--------------|----------|
| Windows / Linux / Mac | `bc3` (DXT5) | `opengl` |
| iOS | `pvr4` | `pvr` |
| Android | `etc2` | `etc` |
| Switch | `bc7` | `nintendo` |
| PS4 / Xbox | `bc7` | `console` |

普通 mod 作者**只需要关心 PC 平台**——默认值就是对的。除非你要为 Klei 官方做主机移植，否则不动这些参数。

> **老手记忆**：Atlas 打包用 MaxRects/Guillotine 启发式装箱；DXT5 (BC3) 是默认压缩格式 4:1；strhash 是 DJB-style 哈希、大小写不敏感、概率碰撞但运行时只用哈希所以不影响渲染；UV 计算时记得 V 轴翻转。

---

### 13.3.8 老手：八个最容易踩的坑

#### 坑 1：autocompiler 静默失败——日志被关闭

**症状**：在文件管理器里双击 `autocompiler.exe`，黑窗一闪就消失，`anim/` 目录里没新 zip。

**原因**：autocompiler 默认输出到 stdout，错误也是 stdout——你看不到任何东西就退出了。

**解决**：从命令行调用，**保留终端**：

```bash
"C:\Program Files (x86)\Steam\steamapps\common\Don't Starve Mod Tools\mod_tools\autocompiler.exe" > compile.log 2>&1
```

然后看 `compile.log` 里的错误信息。

#### 坑 2：scml 中 image 路径用了反斜杠

`peach.scml` 里 PNG 引用是这样的：

```xml
<file id="0" name="peach/peach-0.png" .../>
```

注意是**正斜杠** `/`！如果用反斜杠 `\`，autocompiler 在 Linux/Mac 下编译会报"找不到文件"。

**铁律**：scml 的 `name` 永远写正斜杠。即使在 Windows 下手编辑 .scml 也别用反斜杠。

#### 坑 3：编译产物比预期大很多

**症状**：原来美术给的 PNG 总和 800KB，编译出来的 zip 居然 3MB。

**原因**：
1. **mipmap 层级**——大图自动生成 5-6 个缩小副本，倍数体积膨胀
2. **没用 `optimizeimage`**——透明区域占了 atlas 空间
3. **DXT5 对小图反而效率低**——一张 32×32 的 PNG 压缩后可能比原图大（因为压缩头开销）

**优化方法**：
- 把多个小 Symbol 合并到一个大 atlas
- 必要时手动指定 `--textureformat bc1` 跳过 alpha 通道
- 用 PIL 预处理，把"周围一圈纯透明的图"裁紧

#### 坑 4：动画名带数字后缀被误识别

**症状**：anim 名为 `idle_45s`，本意是"空闲 45 秒变体"——但是编译后被识别成 8 方位 anim！

**原因**：`buildanimation.py` 第 121-122 行的正则：

```python
re.search("(.*)_45s\Z", name),
re.search("(.*)_90s\Z", name)
```

匹配到 `_45s` 后缀就当成"对角四方位"！

**解决**：永远**不要**用 `_up`、`_down`、`_left`、`_right`、`_side`、`_45s`、`_90s` 等关键字作为 anim 名后缀——除非你真的想用它们做方向区分。

#### 坑 5：build 名和文件名不一致

```xml
<entity id="0" name="myhero">
```

但是文件保存为 `super_hero.scml`——编译会成功，**生成 super_hero.zip**，但是 zip 内部的 `build.bin` 里 build 名是 `myhero`。

代码中：

```lua
inst.AnimState:SetBuild("super_hero")  -- 错！这是文件名
inst.AnimState:SetBuild("myhero")      -- 对！这是 entity name
```

**铁律**：**`SetBuild()` 用的是 .scml 里 `<entity name="">` 的值，不是文件名**。最佳实践——让两者一致。

#### 坑 6：`pivot_y` 大于 1 或小于 0

Spriter 允许 pivot 超出 [0, 1] 范围（用于"虚拟轴"在图外面的情况）。看 sample_build.scml：

```xml
<file ... id="4" name="foot/foot-4.png" pivot_x="-0.286083333333" pivot_y="0.491680555556" .../>
```

`pivot_x = -0.286` —— **负的**！表示"中心点在图的左外侧"。

**编译能过**，但是某些版本的 mod tools 会在运行时把 pivot 限制到 [0, 1]，导致部件位置错误。

**调试**：用 krane.exe 反编译别人的 zip 时如果发现整体偏移，**先看 pivot 是否超界**。

#### 坑 7：`auto-compiler` 漏编 `anim/` 已存在的 zip

**症状**：你修改了 `exported/myitem/myitem.scml`，跑 autocompiler，结果 `anim/myitem.zip` **没更新**——还是旧的内容。

**原因**：autocompiler 内置了"时间戳比较"——如果 `anim/myitem.zip` 比 `exported/myitem/myitem.scml` 新，**跳过编译**。但是 **Spriter 保存 .scml 时不会更新文件夹时间戳**——只更新 .scml 自身。

**解决**：
1. **手动删除 `anim/myitem.zip`**——强制重新编译
2. **传 `--force`** 参数（如果 autocompiler 支持）
3. **`touch` 一下 .scml**——更新它的修改时间

#### 坑 8：autocompiler 编译完游戏没刷新

**症状**：编译完游戏里看的还是旧动画。

**原因**：饥荒在启动时把所有 .zip 加载到内存——**编译完不重启游戏不会生效**。

**解决**：
1. **完全退出游戏**——任务管理器确认没残留进程
2. **重启游戏**进世界
3. 调试时开 console 输入 `c_reset()` ——重启当前世界但不退游戏，但是这只重新加载世界，**不重新加载美术资源**——美术修改还是要重启游戏

> **老手记忆**：8 个编译坑里——3 个是**配置错误**（坑 1、5、6）、3 个是**约定违反**（坑 2、4、7）、2 个是**性能/缓存问题**（坑 3、8）。养成"删 anim/zip 强制重编、重启游戏验收"的习惯。

---

### 13.3 小结：关于编译流水线你必须记住的

```
[输入]                          [处理]                       [输出]
.scml + PNG ──→ autocompiler ──→ buildanimation.py         ──→ xxx.zip
                  │              │
                  │              ├─ 阶段1：xml.dom.minidom 解析
                  │              ├─ 阶段2：klei.atlas 矩形打包
                  │              ├─ 阶段3a：textureconverter.dll DXT 编码
                  │              ├─ 阶段3b：ExportBuild → build.bin
                  │              └─ 阶段3c：ExportAnim  → anim.bin
                  │
                  └─ 同时：CompileLuaDirectory.py 编译 lua 字节码
```

**新手核心三句**：编译就是把 `.scml + PNG` → `.zip` 的翻译；autocompiler 找 `exported/`、产物到 `anim/`；编译失败 90% 是 PNG 路径或 .scml 属性写错。

**进阶核心三句**：流水线三阶段是 **解析 → 打包 → 编码**；`build.bin` = 头 + buildname + atlas表 + Symbols + 顶点 + 哈希反查表；`anim.bin` 的 anim 名后缀（`_up`/`_down`/`_side`/`_45s`/`_90s`）会被自动识别成 facing 位掩码。

**老手核心三句**：Atlas 用 MaxRects 启发式装箱、DXT5/BC3 默认压缩 4:1；`strhash` 是 DJB-style + 小写预处理 + 32 位截断；8 个坑里多数是"约定违反"——背熟 13.3.8 这一节能省一半 debug 时间。

下一节 13.4 我们就来看运行时的另一面——`AnimState` 这个组件提供的所有 API：`SetBuild` / `SetBank` / `PlayAnimation` / `PushAnimation` / `OverrideSymbol` / `Hide` / `Show` / `SetSkin` / `SetMultColour`……总共 30+ 个方法。掌握 13.3 的 build.bin/anim.bin 格式后，再看 13.4 这些 API 你就明白"它在底层操作哪个字段"了。

---


## 13.4 AnimState API：播放、叠加、符号替换

### 本节导读

13.1-13.3 我们花了三节讲"美术资源是怎么变成 zip 的"。本节翻到背面——**运行时**：当代码拿到这个 zip，要怎么"驱动"它播放出动画？

答案是 `AnimState`——这是引擎给每个有动画的实体配的一个"动画播放器"组件。它由 C++ 实现（不是普通 Lua 组件，没有 `scripts/components/animstate.lua`），通过 Lua 绑定暴露出**40+ 个方法**。一个 mod 作者一辈子能用到的，可能只是其中 15 个；但是这 15 个**必须背得滚瓜烂熟**。

> **新手**从 13.4.1-13.4.3 起步——理解 `AnimState` 在实体里的位置、最小三件套（SetBank / SetBuild / PlayAnimation）、五大类 API 速查表；**进阶读者**继续看 13.4.4-13.4.6，深入动画队列与 `animover`/`animqueueover` 事件机制、颜色/亮度/像素化等渲染参数、Symbol 显隐控制；**老手**跳到 13.4.7-13.4.8，掌握渲染分层（Layer / SortOrder / FinalOffset）、帧级精确控制（SetFrame / SetTime / SetPercent）、八个最容易踩的坑。

读完本节，你**应该能做到**：① 看任何 prefab 文件第一眼就能看懂它在播什么动画；② 写出"角色受击变红 0.2 秒"这种渲染特效；③ 在控制台用 `c_select():GetAnimState():XXX` 实时调试任何动画问题。

---

### 13.4.1 快速入门：AnimState 是什么、最小三件套

#### 第一步：AnimState 在实体里的位置

每个有动画的实体在创建时都会调用 `inst.entity:AddAnimState()`，往里塞一个 AnimState 组件。看最简单的例子（`scripts/prefabs/phonograph.lua`）：

```141:151:scripts/prefabs/phonograph.lua
local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddSoundEmitter()
    inst.entity:AddMiniMapEntity()
    inst.entity:AddNetwork()

    inst.AnimState:SetBank("phonograph")
    inst.AnimState:SetBuild("phonograph")
```

注意：

- `inst.entity:AddAnimState()` —— 调用一次后，引擎自动在 `inst` 上挂出一个 `AnimState` 字段
- 之后所有动画操作都通过 `inst.AnimState:XXX(...)` 调用
- 它不是普通 Lua component（没有 `scripts/components/animstate.lua`），而是 **C++ 端**注册的"内置组件"

#### 第二步：最小三件套——播一个动画必须的三步

```lua
inst.AnimState:SetBank("phonograph")     -- 步骤 1：选 anim.bin（动作集）
inst.AnimState:SetBuild("phonograph")    -- 步骤 2：选 build.bin（骨架/皮肤）
inst.AnimState:PlayAnimation("idle")     -- 步骤 3：播放某个 anim
```

| 方法 | 对应 13.3 的 | 字符串怎么来 |
|------|-------------|-------------|
| `SetBank(name)` | anim.bin 的 root（在 .scml 中是 `<entity name="...">` 派生）| **是 zip 内部 anim.bin 的"bank 名"**，不是文件名 |
| `SetBuild(name)` | build.bin 的 buildname | **是 .scml 中 `<entity name>` 的字符串**，不是文件名 |
| `PlayAnimation(name)` | anim.bin 中某个 `<anim>` 节点的 name | **是动作名**，比如 `idle`、`walk_loop`、`attack` |

**Bank 和 Build 可以不一样**——这就是后面 13.5 讲"换皮肤/换装"的基础。同一个 `wilson` bank 可以配 `wilson` build（默认皮肤）也可以配 `wendy_skin` build（变身温蒂）。

#### 第三步：第二参数 loop

`PlayAnimation(name, loop)` 的第二个参数 `loop` 是布尔值：

```lua
inst.AnimState:PlayAnimation("idle", true)   -- 循环播放 idle
inst.AnimState:PlayAnimation("attack", false) -- 播一次 attack（默认）
inst.AnimState:PlayAnimation("attack")        -- 等价于 false
```

**`loop=true` 的特殊含义**：动画播完不发 `animover` 事件，自动从头再播一次。**所以 stategraph 不会自动切换状态**——这是为什么 idle、walk_loop 这类常驻动画总是 `loop=true`，而 attack、death 这类一次性动画总是 `loop=false` 或省略。

#### 第四步：`prefab.assets` 和 SetBank/SetBuild 的关系

新手最常踩的坑：**Asset 声明里写的是 zip 文件路径，但 SetBank/SetBuild 用的是 zip 里的 build/bank 名字！它们可能完全不一样！**

```lua
local assets = {
    Asset("ANIM", "anim/myhero.zip"),   -- 路径用 zip 文件名
}

local function fn()
    -- ...
    inst.AnimState:SetBank("myhero")     -- 名字必须和 anim.bin 的 root 匹配
    inst.AnimState:SetBuild("myhero")    -- 名字必须和 build.bin 的 buildname 匹配
end
```

**怎么知道 zip 内部 bank/build 叫什么**？

- 用 `krane.exe` 反编译 zip → 看 `.scml` 里的 `<entity name="..."/>` 就是 build 名
- 控制台 `c_select().AnimState:GetBuild()` —— 直接读出当前 build
- 直接打开 `.scml` 看 `<entity name="..."/>` 节点

> **新手记忆**：`AnimState` 是 C++ 内置组件，加上 `AddAnimState()` 就有。**最小三件套**是 `SetBank` + `SetBuild` + `PlayAnimation`——少任何一个动画都不显示。注意区分**文件名**和**bank/build 名**——它们是两个独立概念。

---

### 13.4.2 快速入门：动画播放与查询的核心 API

#### 第一步：单次播放 vs 队列追加

`PlayAnimation` 和 `PushAnimation` 都能"开始一个动画"，但行为完全不同：

| 方法 | 行为 | 何时用 |
|------|------|--------|
| `PlayAnimation(name, loop?)` | **立即中断**当前动画，从头开始播新动画 | 玩家被打断、重置状态 |
| `PushAnimation(name, loop?)` | **排队**——等当前动画播完再播这个 | 三段式动作衔接（pre → loop → pst）|

来看 `scripts/prefabs/phonograph.lua` 的实际用法：

```111:114:scripts/prefabs/phonograph.lua
local function PlayMusic(inst)
    inst.AnimState:PlayAnimation("play_pre")
    inst.AnimState:PushAnimation("play_loop", true)
    inst.AnimState:PushAnimation("play_pst")
```

读懂这三行：

1. `PlayAnimation("play_pre")` —— 立即开始播 `play_pre`（前奏，比如留声机点头开关）
2. `PushAnimation("play_loop", true)` —— 排队：`play_pre` 播完后开始 `play_loop` 并**循环**
3. `PushAnimation("play_pst")` —— 这一行**永远不会自然触发**——因为上一步是 loop=true，会一直循环。但是当代码执行 `PushAnimation("idle")` 之类切换时，引擎会**清空队列**，重新计算下一个要播什么

实际上更常见的是**两段式**：

```lua
inst.AnimState:PlayAnimation("attack")        -- 一次性的攻击动作
-- 在 stategraph 里监听 "animover"，再切回 idle
```

#### 第二步：查询当前动画状态

| 方法 | 返回 | 用途 |
|------|------|------|
| `IsCurrentAnimation(name)` | bool | 当前是不是这个 anim？ |
| `GetCurrentAnimationName()` | string | 当前 anim 名（debug 用）|
| `GetCurrentAnimationLength()` | float | 当前 anim 的总长度（秒）|
| `GetCurrentAnimationNumFrames()` | int | 当前 anim 的总帧数 |
| `GetCurrentAnimationTime()` | float | 当前播到第几秒了 |
| `AnimDone()` | bool | 当前 anim 是否播完了 |

来看 `scripts/prefabs/warg.lua` 的判断：

```707:707:scripts/prefabs/warg.lua
	if inst.AnimState:IsCurrentAnimation("mutate") then
```

来看 `scripts/stategraphs/SGwx78_shadowdrone_debuffer.lua` 设置 timeout：

```323:323:scripts/stategraphs/SGwx78_shadowdrone_debuffer.lua
			inst.sg:SetTimeout(inst.AnimState:GetCurrentAnimationLength() + 2 + math.random())
```

读懂：用 `GetCurrentAnimationLength()` 拿到当前动画长度，**加上 2 秒缓冲 + 随机量**，作为 stategraph 的超时——这样即使引擎事件未及时触发，状态也能在合理时间退出。

来看 `scripts/stategraphs/commonstates.lua` 监听 `animover`：

```446:447:scripts/stategraphs/commonstates.lua
            EventHandler("animover", function(inst)
```

**`animover` 是 stategraph 最常用的事件**——一个非循环动画播完时引擎自动推送，触发状态切换。但是注意：**循环动画 (`loop=true`) 不会推送 `animover`！**

#### 第三步：精确控制——SetFrame / SetTime / SetPercent

| 方法 | 用途 |
|------|------|
| `SetFrame(idx)` | 跳到指定帧（idx 从 0 开始）|
| `SetTime(seconds)` | 跳到指定时刻（秒）|
| `SetPercent(name, percent)` | **直接跳到某 anim 的某百分比** |
| `Pause()` | 暂停播放 |
| `Resume()` | 恢复播放 |

`SetPercent` 经常用于"动画从中间开始"。来看 `scripts/stategraphs/SGwilson.lua` 死亡动画的"重置到尾帧"：

```3922:3922:scripts/stategraphs/SGwilson.lua
                inst.AnimState:SetPercent(inst.deathanimoverride or "death", 1)
```

`SetPercent("death", 1)` 把 anim 跳到 100% 处——也就是"死亡完成姿势"，作为玩家复活前的初始状态。

`SetFrame` 用于"随机起始相位"，避免大量同类怪物动作整齐划一。来看 `scripts/stategraphs/SGwx78_shadowdrone_debuffer.lua`：

```136:138:scripts/stategraphs/SGwx78_shadowdrone_debuffer.lua
			inst.AnimState:PlayAnimation("idle_loop", true)
            if inst.sg.statemem.skipidleanimrandom then
				inst.AnimState:SetFrame(math.random(inst.AnimState:GetCurrentAnimationNumFrames()) - 1)
```

读懂：**先**调 `PlayAnimation` 启动 idle_loop，**再**调 `SetFrame` 随机跳到某帧——这样 10 只无人机不会同步呼吸，看起来更自然。

#### 第四步：朝向（facing）

8 方位 facing 是 13.3.6 讲过的——anim 名后缀 `_up/_down/_side/_left/_right` 编译后会被自动识别成 facing 位掩码。

```lua
inst.Transform:SetFourFaced()      -- 转 4 方向（玩家常用）
inst.Transform:SetSixFaced()       -- 转 6 方向
inst.Transform:SetEightFaced()     -- 转 8 方向

inst.AnimState:GetCurrentFacing()  -- 拿到当前 facing（位掩码 byte）
```

注意 `SetFourFaced` 等是 **`Transform`** 组件的方法（不是 AnimState）——它告诉引擎"这个实体在 8 方位中只能朝 4 个方向"。

> **新手记忆**：`PlayAnimation` 立即中断、`PushAnimation` 排队等待。`AnimDone()` 配合 stategraph 的 `animover` 事件做状态切换。`SetFrame` 用来做"随机起始相位"避免同步。

---

### 13.4.3 快速入门：常用的五大类 API 速查表

下表把 AnimState 最常用的方法分成 5 大类——背熟这张表就能写 90% 的 mod。

#### 类别一：Bank / Build / Animation 基本控制

| 方法 | 参数 | 含义 |
|------|------|------|
| `SetBank(name)` | string | 选 anim.bin |
| `SetBuild(name)` | string | 选 build.bin |
| `GetBuild()` | — | 返回当前 build 名 |
| `GetSkinBuild()` | — | 返回当前 skin build（皮肤）|
| `PlayAnimation(name, loop?)` | string, bool | 立即播放 |
| `PushAnimation(name, loop?)` | string, bool | 排队播放 |
| `IsCurrentAnimation(name)` | string | 当前是不是这个 anim |
| `AnimDone()` | — | 当前动画播完了吗 |
| `GetCurrentAnimationLength()` | — | 总长度（秒）|
| `GetCurrentAnimationNumFrames()` | — | 总帧数 |
| `GetCurrentAnimationTime()` | — | 当前播到第几秒 |
| `SetFrame(idx)` | int | 跳到第 idx 帧 |
| `SetTime(t)` | float | 跳到第 t 秒 |
| `SetPercent(anim, p)` | string, [0,1] | 跳到指定 anim 的百分比位置 |
| `Pause()` / `Resume()` | — | 暂停 / 恢复 |

#### 类别二：颜色 / 亮度 / 滤镜

| 方法 | 参数 | 效果 |
|------|------|------|
| `SetMultColour(r, g, b, a)` | 4 floats | 整体颜色**乘法**叠加（变暗、变红等）|
| `SetAddColour(r, g, b, a)` | 4 floats | 整体颜色**加法**叠加（变亮）|
| `SetLightOverride(intensity)` | float | 自发光（不受场景光照影响），值在 0-1 |
| `SetBloomEffectHandle(shader_path)` | string | 绑定 bloom 着色器 |
| `SetSymbolBloom(symbol)` | string | 让某个 Symbol 发光 |
| `SetSymbolBrightness(symbol, scale)` | string, float | 单 Symbol 亮度（>1 增亮，<1 减暗）|
| `SetSymbolMultColour(symbol, r,g,b,a)` | | 单 Symbol 颜色乘法 |
| `SetSymbolAddColour(symbol, r,g,b,a)` | | 单 Symbol 颜色加法 |
| `UsePointFiltering(bool)` | bool | 关闭双线性过滤（像素风）|

#### 类别三：Symbol 显隐 + 替换（13.5 详讲）

| 方法 | 用途 |
|------|------|
| `Hide(symbol)` / `Show(symbol)` | 整组的显隐（如 ARM_carry 组）|
| `HideSymbol(symbol)` / `ShowSymbol(symbol)` | 单 Symbol 显隐 |
| `OverrideSymbol(symbol, build, source)` | 用其他 build 的 Symbol 替换 |
| `ClearOverrideSymbol(symbol)` | 取消替换 |
| `AddOverrideBuild(build)` | 加载备选 build（用于动态加载）|
| `ClearOverrideBuild(build)` | 移除备选 build |
| `BuildSymbolIsOverridden(symbol)` | 这个 Symbol 当前被 override 了吗 |
| `SetSkin(skin, default_build)` | 设置玩家皮肤 |

#### 类别四：尺寸 / 朝向

| 方法 | 用途 |
|------|------|
| `SetScale(x, y, z?)` | 缩放（z 通常省略 = 用 1）|
| `GetCurrentFacing()` | 拿当前 facing 字节 |
| `MakeFacingDirty()` | 标记 facing 需要刷新 |

#### 类别五：渲染分层（13.4.7 详讲）

| 方法 | 用途 |
|------|------|
| `SetLayer(layer)` | 大层级（LAYER_WORLD / LAYER_BACKGROUND 等）|
| `SetSortOrder(int)` | 同层内细排序 |
| `SetFinalOffset(int)` | 同实体内多渲染层细分 |
| `SetSortWorldOffset(x, y, z)` | 世界空间的排序偏移 |
| `SetRayTestOnBB(bool)` | 鼠标命中测试用 bbox 而非像素 |

> **新手记忆**：5 类 = **基础控制 + 颜色 + Symbol + 尺寸 + 分层**。**前两类基本天天用**，第三类（13.5 详讲）做换装和换皮肤，第五类做特殊视觉效果（鬼影、地板贴图、UI 角色）。

---

### 13.4.4 进阶：动画队列与 animover / animqueueover 事件

#### 第一步：理解动画队列

`AnimState` 内部有一个**FIFO 队列**——`PlayAnimation` 清空队列并加入第一项；`PushAnimation` 仅追加到队尾。引擎在每一帧推进当前动画，播完时弹出下一个。

**用一段简化的伪代码模拟引擎逻辑**：

```
queue = []
function PlayAnimation(name, loop):
    queue = [(name, loop)]
    SwitchTo(queue[0])

function PushAnimation(name, loop):
    queue.append((name, loop))

# 每帧：
function Update(dt):
    current = queue[0]
    advance(current, dt)
    if anim_done(current):
        if not current.loop:
            push_event("animover")  -- 只在非循环时推
        queue.pop(0)
        if len(queue) > 0:
            SwitchTo(queue[0])
        else:
            push_event("animqueueover")  -- 整个队列都空了
```

**两个事件的区别**：

| 事件 | 触发时机 | 用法 |
|------|---------|------|
| `animover` | 一个**非循环** anim 播完时 | 单次动作完成后切状态（attack → idle）|
| `animqueueover` | 整个动画队列都空时（包括 push 序列）| 三段式动作全部播完时切状态 |

#### 第二步：单段 attack —— 用 animover

来看 `scripts/stategraphs/commonstates.lua` 的 attack 状态片段（简化）：

```lua
State {
    name = "attack",
    onenter = function(inst)
        inst.AnimState:PlayAnimation("atk")
    end,
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
}
```

读懂：进入 `attack` 状态时 PlayAnimation("atk")，监听 `animover`——动画一播完就回 idle。

#### 第三步：三段式 walk —— 用 PushAnimation + animqueueover

来看 `scripts/stategraphs/commonstates.lua` 第 656 行 walk 的标准模式：

```656:656:scripts/stategraphs/commonstates.lua
            EventHandler("animqueueover", idleonanimover),
```

实际的 walk 通常是：

```lua
State {
    name = "walk_start",
    onenter = function(inst)
        inst.AnimState:PlayAnimation("walk_pre")
        inst.AnimState:PushAnimation("walk_loop", true)  -- 循环！
    end,
    events = {
        -- 没有 animover —— 因为 walk_loop 是 loop 不会触发
    },
}

State {
    name = "walk_stop",
    onenter = function(inst)
        inst.AnimState:PlayAnimation("walk_pst")  -- 中断 loop，播收尾
    end,
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
}
```

**关键**：`walk_loop` 是 loop=true 永远不发 animover；只有当玩家停下来切到 walk_stop 状态、PlayAnimation("walk_pst") 中断 loop 后，walk_pst 播完才发 animover 切回 idle。

#### 第四步：循环动画的"自然结束"——为什么有 animqueueover

**这是一个微妙的细节**：

```lua
inst.AnimState:PlayAnimation("attack_pre")
inst.AnimState:PushAnimation("attack_loop", false)  -- 非循环！
inst.AnimState:PushAnimation("attack_pst")
```

这种写法播完后会触发**两个 animover**（每段都触发）+ 一个 `animqueueover`（队列空了）。如果你的状态切换需要"整个序列结束"才触发——**用 animqueueover 而不是 animover**，否则第一段播完就切走了。

#### 第五步：不能信任的状态

调用 `PlayAnimation("attack")` 后**立即** `IsCurrentAnimation("attack")` —— 不一定返回 true！原因：动画切换在引擎下一帧才生效，本帧还是上一个动画的状态。

**最佳实践**：在状态机的 onenter 里调 PlayAnimation 后，**不要立刻**判断 IsCurrentAnimation。等下一帧或在 timeline 里再判断。

> **进阶记忆**：动画队列 = FIFO；`animover` 是单段播完、`animqueueover` 是队列空。`loop=true` 的 anim **不会**触发 animover——必须靠新的 PlayAnimation 中断它。

---

### 13.4.5 进阶：颜色、亮度、滤镜、像素化

#### 第一步：SetMultColour vs SetAddColour

这是最常用的两个颜色 API，含义截然不同：

```
final_pixel = (texture_pixel * MultColour) + AddColour
```

| API | 公式 | 视觉效果 |
|-----|------|---------|
| `SetMultColour(0.5, 0.5, 0.5, 1)` | RGB 都 ×0.5 | **整体变暗**（保留色调）|
| `SetMultColour(1, 0, 0, 1)` | R 留下、GB 变 0 | **变红**（去掉绿蓝）|
| `SetAddColour(0.5, 0, 0, 0)` | R 加 0.5 | **泛红光**（不替换原色调）|
| `SetMultColour(1, 1, 1, 0.6)` | alpha=0.6 | **半透明** |

#### 第二步：闪烁红色 = 受击效果

来看 `scripts/components/locomotor.lua` 周围的常见模式：

```lua
inst.AnimState:SetMultColour(1, 0.3, 0.3, 1)  -- 立刻变红
inst:DoTaskInTime(0.1, function()
    inst.AnimState:SetMultColour(1, 1, 1, 1)  -- 0.1 秒后恢复正常
end)
```

这就是"受击闪红"特效——非常常用。

#### 第三步：SetLightOverride —— 自发光

`SetLightOverride(intensity)` 让实体**自身发光**——不受场景昏暗光照影响。来看 `scripts/prefabs/warg.lua` 的火焰特效：

```647:653:scripts/prefabs/warg.lua
	inst.AnimState:SetBank("lunar_flame")
	inst.AnimState:SetBuild("lunar_flame")
	inst.AnimState:PlayAnimation("gestalt_eye", true)
	inst.AnimState:SetMultColour(1, 1, 1, 0.6)
	inst.AnimState:SetLightOverride(0.1)
	inst.AnimState:SetBloomEffectHandle("shaders/anim.ksh")
	inst.AnimState:UsePointFiltering(true)
```

这一段干了 5 件事：

1. SetBank/SetBuild/PlayAnimation —— 基础三件套
2. `SetMultColour(1,1,1,0.6)` —— 60% 透明
3. `SetLightOverride(0.1)` —— 即使在夜晚也能看见 10% 自发光
4. `SetBloomEffectHandle("shaders/anim.ksh")` —— 绑定 bloom 着色器（让发光部分有"光晕"）
5. `UsePointFiltering(true)` —— 关闭线性过滤，保留像素风

#### 第四步：SetSymbolBloom + SetSymbolBrightness —— 单 Symbol 发光

整体发光太亮可能影响美感——很多场景**只想让某个部位发光**（眼睛、武器、装饰）。来看 `warg.lua`：

```860:861:scripts/prefabs/warg.lua
				inst.AnimState:SetSymbolBloom("breath_02")
				inst.AnimState:SetSymbolBrightness("breath_02", 1.5)
```

`SetSymbolBloom("breath_02")` —— 只让 `breath_02` 这个 Symbol 进入 bloom。
`SetSymbolBrightness("breath_02", 1.5)` —— 让这个 Symbol 亮度 ×1.5。

可以做**眼神发光**、**武器尖端发光**、**伤口闪烁**等等针对性特效。

#### 第五步：UsePointFiltering —— 像素风

GPU 默认对纹理用**双线性过滤**（bilinear filtering）——把相邻像素混色显示，看起来平滑。但是某些 mod 想要"复古像素风"或"像素马赛克"效果——`UsePointFiltering(true)` 关掉过滤，只显示最近邻像素。

适用场景：8-bit 风格 mod、像素艺术皮肤、低分辨率 UI。

#### 第六步：SetBloomEffectHandle —— 着色器绑定

```lua
inst.AnimState:SetBloomEffectHandle("shaders/anim.ksh")
```

`shaders/anim.ksh` 是 Klei 自带的 bloom 着色器（KSH 是 Klei Shader）。除了 anim.ksh，还有：

- `shaders/anim_lightoverride.ksh` —— 带 light override 的 bloom
- `shaders/ghost.ksh` —— 鬼魂效果
- `shaders/distort.ksh` —— 扭曲

**自定义 mod 着色器**：`Asset("SHADER", "shaders/myshader.ksh")` 声明，然后 `SetBloomEffectHandle("shaders/myshader.ksh")`——但是写 KSH 着色器需要懂 GLSL，超出本节范围。

> **进阶记忆**：颜色叠加用 **mult**（乘法，留色调）+ **add**（加法，泛光）；自发光用 `SetLightOverride`；单 Symbol 发光用 `SetSymbolBloom + SetSymbolBrightness`。**像素风**关 `UsePointFiltering`。

---

### 13.4.6 进阶：Symbol 显隐 + 替换（13.5 详讲入门版）

13.5 会深入讲 OverrideSymbol 的换装原理。本节先把 4 个最基本的 API 讲清楚——**让你看懂 prefab 里那些 Hide/Show 的代码在干嘛**。

#### 第一步：Hide / Show ——"组"级显隐

```lua
inst.AnimState:Hide("ARM_carry")    -- 隐藏 ARM_carry 组
inst.AnimState:Show("ARM_normal")   -- 显示 ARM_normal 组
```

**注意**：`Hide("ARM_carry")` 中 `ARM_carry` 是**Symbol "组" 名（组前缀大写约定）**。在 build.bin 中，多个 Symbol 可以共享一个"组"——隐藏一个组就是隐藏组里所有 Symbol。

来看 `scripts/prefabs/voidcloth_scythe.lua` 的"举手"逻辑（简化）：

```lua
-- 玩家拿起武器：
owner.AnimState:Show("ARM_carry")    -- 显示"持物姿势"
owner.AnimState:Hide("ARM_normal")   -- 隐藏"自然垂手"

-- 玩家放下武器：
owner.AnimState:Hide("ARM_carry")
owner.AnimState:Show("ARM_normal")
```

这就是为什么 Wilson 拿斧头和不拿斧头时手臂姿态不同——**是显隐两组手臂的 Symbol，不是替换。**

#### 第二步：HideSymbol / ShowSymbol —— 单 Symbol 显隐

```lua
inst.AnimState:HideSymbol("hat")  -- 隐藏 hat 这个 Symbol
inst.AnimState:ShowSymbol("hat")  -- 显示
```

适用场景：动态隐藏某个部件——比如"砍头"特效（隐藏 head Symbol）、"穿斗篷"（显示 cloak Symbol）。

#### 第三步：OverrideSymbol —— 替换（13.5 详讲）

```lua
inst.AnimState:OverrideSymbol("swap_object", "swap_axe", "swap_axe")
```

3 个参数：

1. **目标 Symbol 名**——当前 build 里要被替换的 Symbol 名（通常是 `swap_object`、`swap_hat`、`swap_body` 等占位 Symbol）
2. **源 build 名**——从哪个 build 里取替换 Symbol
3. **源 Symbol 名**——源 build 里那个 Symbol 的名字

来看 `scripts/prefabs/nightstick.lua`：

```30:30:scripts/prefabs/nightstick.lua
        owner.AnimState:OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")
```

这一行让玩家手中的 `swap_object`（占位空白）被替换成 `swap_nightstick.zip` 里的 `swap_nightstick` Symbol——也就是亮起来的电棍。

#### 第四步：AddOverrideBuild —— 预加载 build

如果你的资源声明用了 `DYNAMIC_ANIM`（13.1.6 讲过——延迟加载，省内存），那么 `OverrideSymbol` 调用时引擎可能还**没加载**那个 build。需要先：

```lua
inst.AnimState:AddOverrideBuild("swap_nightstick")  -- 触发加载
inst.AnimState:OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")
```

来看 `scripts/prefabs/warg.lua` 的实例：

```987:987:scripts/prefabs/warg.lua
            inst.AnimState:AddOverrideBuild("gingerbread_pigman")
```

如果是普通 `ANIM`（非 DYNAMIC），可以省略 `AddOverrideBuild`——引擎已经预加载好了。

#### 第五步：ClearOverrideSymbol —— 取消替换

```lua
inst.AnimState:ClearOverrideSymbol("swap_object")
```

这相当于"放下武器"——swap_object 恢复成默认（透明占位）。

#### 第六步：BuildSymbolIsOverridden —— 状态查询

```lua
if inst.AnimState:BuildSymbolIsOverridden("swap_object") then
    -- 玩家手里有东西
end
```

**调试时**经常用——在控制台敲 `c_select():GetAnimState():BuildSymbolIsOverridden("swap_object")` 看玩家是否拿着武器。

> **进阶记忆**：Hide/Show 是**组**级（大写组前缀）、HideSymbol/ShowSymbol 是单 Symbol、OverrideSymbol 三参数（被替换的、源 build、源 Symbol）。DYNAMIC_ANIM 资源要先 AddOverrideBuild 再 Override。

---

### 13.4.7 老手：渲染分层与排序——SetLayer / SetSortOrder / SetFinalOffset

这一节是老手才需要关心的——**默认值通常就够用**，但是某些视觉效果（地表贴图、UI 实体、鬼影、电流）必须精确控制渲染顺序。

#### 第一步：渲染分层（Layer）—— 大层级

每个 AnimState 实体有一个 **Layer**（大层级）。Layer 决定了渲染的"主桶"——同一个 Layer 内的实体一起渲染，不同 Layer 之间有严格的先后关系。

主要 Layer 常量（在 `scripts/constants.lua` 或类似地方定义）：

| Layer | 数值 | 何时渲染 | 适用 |
|-------|------|----------|------|
| `LAYER_BELOW_GROUND` | 较小 | 最先（在地面之下）| 阴影、地表贴图、坑洞 |
| `LAYER_GROUND` | | 地面层 | 地砖、足迹 |
| `LAYER_BACKGROUND` | | 背景层 | 远景、装饰 |
| `LAYER_WORLD` | 默认 | 普通世界物体 | 大部分实体 |

来看 `scripts/prefabs/voidcloth_umbrella.lua` 的特殊设定：

```184:185:scripts/prefabs/voidcloth_umbrella.lua
	inst.AnimState:SetLayer(LAYER_BACKGROUND)
	inst.AnimState:SetSortOrder(3)
```

这个伞的"特效部分"被设置在 LAYER_BACKGROUND——比普通实体更后渲染（也就是"看起来在更后面"）。

来看 `scripts/stategraphs/SGwx78_possessedbody.lua` 第 4427 行的"埋入地下"特效：

```4427:4427:scripts/stategraphs/SGwx78_possessedbody.lua
            inst.AnimState:SetLayer(LAYER_BELOW_GROUND)
```

把实体放进 LAYER_BELOW_GROUND，看起来就是"在地面下面"。

#### 第二步：SortOrder —— Layer 内细排序

同一个 Layer 内的实体按什么顺序渲染？默认是**按 Y 坐标从大到小**（远的先画、近的后画）。但是某些情况下要打破这个顺序——`SetSortOrder(int)` 提供 Layer 内的细排序：

```lua
inst.AnimState:SetSortOrder(3)   -- 这个实体在同 Layer 内排序值是 3
```

数值越大越后渲染（=越靠前显示）。默认是 0。

来看 `scripts/prefabs/gestalt_cage.lua`：

```728:728:scripts/prefabs/gestalt_cage.lua
    inst.AnimState:SetSortOrder(-1)
```

负数把它压到背景。

#### 第三步：FinalOffset —— 同实体多渲染层

最精细的层级控制是 `FinalOffset`——它在 Layer + SortOrder 之外还细分。来看 `scripts/prefabs/gelblob.lua`：

```534:534:scripts/prefabs/gelblob.lua
	inst.AnimState:SetFinalOffset(7)
```

```628:628:scripts/prefabs/gelblob.lua
	inst.AnimState:SetFinalOffset(-7)
```

正数是"更前面"（最后画 = 看起来在最上层）、负数是"更后面"。

**实战**：UI 上的"角色头像"用 `SetFinalOffset(7)` 让它压过其他 UI 元素；地面贴花用 `SetFinalOffset(-7)` 让它在玩家脚下。

#### 第四步：SetSortWorldOffset —— 世界空间偏移

```lua
inst.AnimState:SetSortWorldOffset(0, 0.5, 0)  -- 排序时假装这个实体高 0.5 单位
```

这是个魔法 API——**它不改变实体真实位置**，只在排序时假装它在那里。常用于"挂起的物体应该排在举着它的角色前面"——给挂起物 `SetSortWorldOffset(0, +1, 0)` 假装它高 1 单位，于是排序时它在角色之前渲染。

#### 第五步：SetRayTestOnBB —— 鼠标命中测试

默认情况下，鼠标命中测试是按**像素**做的——只有当鼠标准确落在非透明像素上才命中。但是某些大型实体（树、建筑）这样太严格——树叶之间的缝隙鼠标就点不到。

```lua
inst.AnimState:SetRayTestOnBB(true)
```

打开后，鼠标命中测试改用**实体的 bounding box**——只要在矩形范围内就算命中。来看 `scripts/prefabs/evergreens.lua`：

```207:207:scripts/prefabs/evergreens.lua
    inst.AnimState:SetRayTestOnBB(true)
```

针叶林这种"枝叶稀疏"的树用 BB 命中——玩家用鼠标更容易选中它。

#### 第六步：层级问题的调试

控制台命令查询当前实体的渲染参数：

```lua
local ent = c_select()
print(ent.AnimState:GetCurrentAnimationName())     -- 当前 anim 名
print(ent.AnimState:GetBuild())                    -- 当前 build
print(ent.AnimState:IsCurrentAnimation("idle"))    -- 是否在 idle
-- AnimState 没有 GetLayer/GetSortOrder 公开接口，需要用 mod_dbui 调试
```

如果有元素显示不出来 / 被遮住——优先检查：
1. SetMultColour 的 alpha 是不是 0
2. SetLayer 是不是在 LAYER_BELOW_GROUND
3. SetFinalOffset 是不是太小

> **老手记忆**：渲染分层 4 级粒度——**Layer**（大层级）→ **SortOrder**（同层细排）→ **FinalOffset**（最细微调）→ **SortWorldOffset**（虚拟位置）。SetRayTestOnBB 是鼠标命中调整。

---

### 13.4.8 老手：八个最容易踩的坑

#### 坑 1：忘了 SetBank 或 SetBuild

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    -- 忘了 SetBank/SetBuild！
    inst.AnimState:PlayAnimation("idle")  -- 不显示任何东西
    return inst
end
```

**症状**：实体被创建出来（`c_select()` 能选到），但是屏幕上什么都没有。

**原因**：没有 build 引擎不知道要画什么；没有 bank 引擎不知道动画数据从哪读。

**修复**：3 件套不能少。

#### 坑 2：bank 名和 build 名混淆

很多 mod 作者随手写：

```lua
inst.AnimState:SetBank("anim/myhero.zip")    -- 错！传了路径
inst.AnimState:SetBuild("myhero.zip")        -- 错！带 .zip
```

正确：

```lua
inst.AnimState:SetBank("myhero")
inst.AnimState:SetBuild("myhero")
```

**bank/build 名不是文件路径，也不带后缀**——它们是 .scml 中 `<entity name="..."/>` 的字符串。

#### 坑 3：循环动画期望 animover

```lua
State {
    onenter = function(inst)
        inst.AnimState:PlayAnimation("idle_loop", true)  -- loop=true！
    end,
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("next")  -- 永远不会触发！
        end),
    },
}
```

**症状**：状态卡死。

**原因**：循环动画**不发** animover。

**修复**：要么去掉 loop=true、要么用其他方式切换状态（timeout、外部事件）。

#### 坑 4：PlayAnimation + 立即查询

```lua
inst.AnimState:PlayAnimation("attack")
if inst.AnimState:IsCurrentAnimation("attack") then  -- 不一定是 true！
    -- ...
end
```

**原因**：动画切换在引擎下一帧才生效。

**修复**：改用 `inst:DoTaskInTime(0, function() ... end)` 延后一帧，或者直接信任你刚刚 PlayAnimation 的字符串。

#### 坑 5：SetMultColour alpha 为 0 后忘记恢复

```lua
inst.AnimState:SetMultColour(1, 1, 1, 0)  -- 透明
-- 后面忘了恢复 → 实体永远不显示
```

**调试技巧**：在控制台 `c_select().AnimState:SetMultColour(1, 1, 1, 1)` 强制恢复，看是不是这个原因。

#### 坑 6：DYNAMIC_ANIM 没 AddOverrideBuild 就 OverrideSymbol

```lua
local assets = {
    Asset("DYNAMIC_ANIM", "anim/dynamic/swap_axe.zip"),
}

inst.AnimState:OverrideSymbol("swap_object", "swap_axe", "swap_axe")
-- 没效果，因为 swap_axe build 还没加载
```

**修复**：

```lua
inst.AnimState:AddOverrideBuild("swap_axe")  -- 先加载
inst.AnimState:OverrideSymbol("swap_object", "swap_axe", "swap_axe")
```

#### 坑 7：facing 后缀和真实 anim 名混淆

如果你的 .scml 里有 `idle_up`、`idle_down`、`idle_side`，编译后**真实存的 anim 名是 `idle`**——只是 facing 不同。代码里调：

```lua
inst.AnimState:PlayAnimation("idle_up")  -- 错！这个名字不存在
inst.AnimState:PlayAnimation("idle")     -- 对，引擎根据当前 facing 选合适的
```

**必须用"去后缀名"**——参考 13.3.6 讲的 facing 编码。

#### 坑 8：实体不渲染但所有参数看起来都对

终极检查清单——按顺序逐项排查：

1. `inst.entity:AddAnimState()` 调用了吗？
2. `inst.entity:AddTransform()` 调用了吗？（没有 Transform 渲染系统不知道画在哪里）
3. SetBank / SetBuild 都调用了吗？字符串拼写都对吗？
4. PlayAnimation 调用了吗？anim 名拼写正确吗？
5. SetMultColour 的 alpha 是 0 吗？
6. SetLayer 是 LAYER_BELOW_GROUND 吗？被埋在地下了？
7. SetScale 是 0 吗？被缩放成 0 了？
8. inst:Hide() 调用过吗？整个实体被隐藏了？

控制台一行排查命令：

```lua
local e = c_select()
print(string.format("bank/build: %s/%s, anim: %s, mult-a: %.2f, layer: ?",
    "?", e.AnimState:GetBuild(),
    e.AnimState:GetCurrentAnimationName(), 1.0))
```

> **老手记忆**：8 个 AnimState 坑里——3 个是**API 漏调**（坑 1、6、8）、3 个是**API 误用**（坑 2、3、5）、2 个是**理解错误**（坑 4、7）。**记住"实体不显示"的 8 步调试清单**比记 API 列表更重要。

---

### 13.4 小结：关于 AnimState API 你必须记住的

```
AnimState（C++ 内置组件，AddAnimState 启用）
│
├─ 类别一：基础控制
│  ├─ SetBank / SetBuild —— 选 anim.bin / build.bin
│  └─ PlayAnimation / PushAnimation —— 立刻播 / 排队播
│
├─ 类别二：颜色 & 滤镜
│  ├─ SetMultColour（RGB 乘）
│  ├─ SetAddColour（RGB 加）
│  ├─ SetLightOverride（自发光）
│  └─ SetSymbolBloom + SetSymbolBrightness（单 Symbol）
│
├─ 类别三：Symbol 显隐 + 替换（13.5 详讲）
│  ├─ Hide/Show（组级）
│  ├─ HideSymbol/ShowSymbol（单个）
│  └─ OverrideSymbol（换装核心）
│
├─ 类别四：尺寸 & 朝向
│  ├─ SetScale
│  └─ GetCurrentFacing
│
└─ 类别五：渲染分层（高阶）
   ├─ SetLayer（大层级）
   ├─ SetSortOrder（同层细排）
   ├─ SetFinalOffset（细微调）
   └─ SetRayTestOnBB（鼠标命中）
```

**新手核心三句**：基础三件套是 `SetBank + SetBuild + PlayAnimation`，少任何一个都不显示；`PlayAnimation` 立即中断、`PushAnimation` 排队等待；循环动画**不发** animover。

**进阶核心三句**：`animover` 是单段播完、`animqueueover` 是整个队列空；`SetMultColour` 是乘法（保色调）、`SetAddColour` 是加法（泛光）；DYNAMIC_ANIM 资源要先 AddOverrideBuild 再 OverrideSymbol。

**老手核心三句**：渲染分层 4 级粒度（**Layer → SortOrder → FinalOffset → SortWorldOffset**）；用 `SetTime/SetFrame/SetPercent` 做帧级精确控制和"随机起始相位"避免同步；八个坑里"实体不显示" 8 步调试清单是救命稻草。

下一节 13.5 我们将单独把 **OverrideSymbol** 这个最核心的"换装"API 拆透——配合 build.bin 内部数据结构、`AddOverrideBuild` 的延迟加载、SetSkin 系统的封装——理解为什么 Wilson 拿斧头不是"把斧头模型贴到他手上"，而是"用 swap_axe 替换他的 swap_object Symbol"。

---


## 13.5 符号覆盖（OverrideSymbol）——角色换装的原理

### 本节导读

13.4 我们点到了 `OverrideSymbol` 这个 API 的存在；本节深入它——这是**饥荒动画系统中最有创造力的一个机制**。

为什么这么说？看下面这两件事：

1. Wilson 拿斧头时屏幕上出现的"Wilson + 斧头"动画，**不是**美术画了 50 个"Wilson 拿斧头"的关键帧，而是**Wilson 默认 build 中本来就有的 `swap_object` 占位 Symbol，被运行时换成了斧头模型**——同一套 anim 数据复用了无数次。
2. 玩家选了一个皮肤"沙漠 Wilson"——本质上是同一个 `wilson` bank（动作集），只是 **build 名换成了 `wilson_skin_desert`**——所有动作零修改，但是穿出了截然不同的造型。

这两个机制——**装备穿戴**与**皮肤系统**——背后的核心都是同一个 API：`OverrideSymbol`（以及它的兄弟 `OverrideSkinSymbol`）。掌握本节，你就**真正理解了为什么饥荒能用一个 zip 表现出无数种角色形象**。

> **新手**从 13.5.1-13.5.3 起步——理解"换装 = Symbol 替换"的核心思想、`swap_object/swap_hat/swap_body` 三大占位 Symbol、装备一把武器的完整代码流程；**进阶读者**继续看 13.5.4-13.5.6，掌握 `OverrideSymbol` 全家族 API、`HAT/HAIR_HAT/HAIR_NOHAT` 等组级 Hide/Show 配合换装、`AddOverrideBuild` 延迟加载机制；**老手**跳到 13.5.7-13.5.8，吃透 `Skinner` 组件 + `OverrideSkinSymbol` 多层覆盖（torso/torso_pelvis/skirt/leg/foot）、八个最容易踩的坑。

读完本节，你**应该能做到**：① 给一把自定义武器写出正确的 onequip / onunequip；② 给一顶自定义帽子做出"戴帽时显眉毛、不戴时显头发"的逻辑；③ 看懂 Klei 官方皮肤系统的源码、知道为什么 `Skinner:SetSkinName` 改一个字符串就能让玩家变身。

---

### 13.5.1 快速入门：为什么"换装 = 替换 Symbol"

#### 第一步：观察一个 Wilson 默认 build

13.2.4 讲过 Wilson 这个角色由 30+ 个 Symbol 组成——`head`、`torso`、`hand`、`leg`……除了这些"身体部件"，还有几个特殊的 Symbol：

| Symbol 名 | 默认内容 | 用途 |
|----------|---------|------|
| `swap_object` | **空白透明** Symbol（没有任何像素）| 装备武器/工具时被替换 |
| `swap_hat` | **空白透明** | 戴帽子时被替换 |
| `swap_body` | **空白透明** | 穿护甲时被替换 |
| `swap_face` | 默认是带胡子的脸 | 戴某些头盔时被替换/隐藏 |

**关键认识**：这些 `swap_xxx` Symbol 是**美术故意留的"挂载点"**——它们在 Wilson 默认形象里看不到（透明或被遮挡），但是它们**参与所有动画的关键帧序列**。当玩家挥动右臂攻击时，引擎正确地把 `swap_object` 这个 Symbol 移动到剑应该在的位置。

代码所要做的事，**只是在 `swap_object` 这个 Symbol 上"贴"一张剑的图**——这就是 `OverrideSymbol` 的工作。

#### 第二步：换装的核心调用

来看 `scripts/prefabs/nightstick.lua` 第 22-45 行（电棍的 onequip）：

```22:35:scripts/prefabs/nightstick.lua
local function onequip(inst, owner)
    inst.components.burnable:Ignite()

    local skin_build = inst:GetSkinBuild()
    if skin_build ~= nil then
        owner:PushEvent("equipskinneditem", inst:GetSkinName())
        owner.AnimState:OverrideItemSkinSymbol("swap_object", skin_build, "swap_nightstick", inst.GUID, "swap_nightstick")
    else
        owner.AnimState:OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")
    end

    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
```

**核心三行**：

1. `OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")` —— 这是**换装核心**。让玩家身上的 `swap_object` 占位被 `swap_nightstick.zip` 里的 `swap_nightstick` Symbol 替换——也就是"贴"上电棍图
2. `Show("ARM_carry")` —— 显示"持物姿态"的手臂（指向前方）
3. `Hide("ARM_normal")` —— 隐藏"自然垂下"的手臂

注意 prefab 顶部声明了两个动画包：

```1:6:scripts/prefabs/nightstick.lua
local assets =
{
    Asset("ANIM", "anim/nightstick.zip"),
    Asset("ANIM", "anim/swap_nightstick.zip"),
    Asset("SOUND", "sound/common.fsb"),
}
```

- `nightstick.zip` —— 电棍**作为掉落物在地上时**的形象（旋转的小物件）
- `swap_nightstick.zip` —— **被装备时**塞到玩家手里的形象——zip 内只有一个 `swap_nightstick` Symbol（一根电棍图）

#### 第三步：为什么不直接画 50 帧"持电棍 Wilson"？

**理论上**美术也可以这么干：给每个角色每件装备做一套完整动画。但是**代价巨大**：

- 假设 30 个角色 × 50 件装备 × 30 个动作 × 12 帧 = **54 万张图**
- 增加一件新装备要重做 30 个角色的所有动作

**OverrideSymbol 的天才**：动画数据**完全不变**，只是替换 swap_object 这个 Symbol。因此：

- 1 套通用动作（不管角色）+ 1 张电棍图 = 任意角色都能拿电棍
- 增加新装备只要画一张 `swap_xxx` 图（一张！）

这就是饥荒能在保持小体积的同时支持几百件装备 + 几百个皮肤的秘密。

> **新手记忆**：换装 = 把 `swap_object/swap_hat/swap_body` 这些**默认透明**的占位 Symbol，**临时贴**上其他 zip 里的图。动画数据**完全不变**——这是饥荒动画系统的"高复用低成本"核心。

---

### 13.5.2 快速入门：OverrideSymbol 三参数详解

```lua
inst.AnimState:OverrideSymbol(target_symbol, source_build, source_symbol)
```

#### 参数 1：target_symbol（被替换的 Symbol）

**这是当前 build（玩家身上）里要被覆盖的 Symbol 名**。绝大多数情况下是这 5 个之一：

| target_symbol | 用途 |
|---------------|------|
| `swap_object` | 武器、工具、手持物（手部 placeholder）|
| `swap_hat` | 帽子（头部 placeholder）|
| `swap_body` | 护甲、衣服（躯干 placeholder）|
| `swap_face` | 替换面部表情（独立面部 Symbol）|
| `headbase_hat` | 全包头盔（替换"戴帽子时的脑袋底"）|

#### 参数 2：source_build（源 build 名）

**这是要拿来贴的图所在的 build 名**——也就是 zip 里的 `<entity name>` 字符串（13.4.1 讲过）。

**关键**：这个 build **必须已经被加载到内存**——要么是 prefab.assets 里声明的 `ANIM`（启动时加载），要么是 `DYNAMIC_ANIM` + 已经 `AddOverrideBuild` 过的（13.5.6）。

#### 参数 3：source_symbol（源 build 里的 Symbol 名）

**从源 build 里取哪个 Symbol 贴到 target**。

最常见的命名约定：**source_symbol = target_symbol** —— 也就是 `OverrideSymbol("swap_object", "swap_axe", "swap_object")`，意思是"用 swap_axe build 的 swap_object Symbol 来替换玩家的 swap_object"。

但是如果美术给你的 zip 里只有一个叫 `swap_axe` 的 Symbol（不叫 swap_object），那就得写：

```lua
OverrideSymbol("swap_object", "swap_axe", "swap_axe")
```

电棍的例子就是后者：`OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")` —— 因为 swap_nightstick.zip 内的 Symbol 叫 `swap_nightstick`。

#### 第四步：内存层面发生了什么

执行 `OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")` 后：

```
玩家 AnimState 内部的 Symbol 表：
{
    head        → 默认 build 的 head Symbol
    torso       → 默认 build 的 torso Symbol
    hand        → 默认 build 的 hand Symbol
    swap_object → swap_nightstick build 的 swap_nightstick Symbol  ← 被替换！
    swap_hat    → 默认 build 的 swap_hat Symbol（透明）
    ...
}
```

引擎在每帧渲染时，**先**查 anim.bin 知道某个时刻 `swap_object` 应该在 (x=10, y=20)、旋转 30°、缩放 1.0；**然后**查 Symbol 表知道现在 swap_object 是从 swap_nightstick.zip 来的；**最后**从那个 zip 的 atlas 里取像素绘制。

**重要**：变换矩阵（位置/旋转/缩放）来自**当前播放的 anim**——也就是 Wilson 的 anim.bin。但是绘制的图像来自**override 后的 build**——也就是 swap_nightstick.zip。两者**互不影响**。

> **新手记忆**：`OverrideSymbol(被替换名, 源build名, 源Symbol名)` 三参数。一般情况下被替换名是 `swap_xxx`、源 Symbol 名也是 `swap_xxx`，源 build 名是放着这张图的 zip 的 build 名。**位置/旋转来自 anim、贴图来自 build**。

---

### 13.5.3 快速入门：装备一把武器、戴一顶帽子的完整流程

#### 第一步：自定义武器的 onequip / onunequip

照抄 `nightstick.lua` 的模式做一把"自定义剑"：

```lua
local assets =
{
    Asset("ANIM", "anim/myaxe.zip"),         -- 地上掉落物的形象
    Asset("ANIM", "anim/swap_myaxe.zip"),    -- 玩家手里的形象
}

local function onequip(inst, owner)
    -- 步骤 1：替换 swap_object Symbol
    owner.AnimState:OverrideSymbol("swap_object", "swap_myaxe", "swap_myaxe")
    
    -- 步骤 2：切换持物姿态
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end

local function onunequip(inst, owner)
    -- 步骤 1：清掉 swap_object 的 override
    owner.AnimState:ClearOverrideSymbol("swap_object")
    
    -- 步骤 2：恢复空手姿态
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
end

local function fn()
    local inst = CreateEntity()
    -- ...
    inst.AnimState:SetBank("myaxe")
    inst.AnimState:SetBuild("myaxe")
    inst.AnimState:PlayAnimation("idle")
    
    inst:AddComponent("equippable")
    inst.components.equippable:SetOnEquip(onequip)
    inst.components.equippable:SetOnUnequip(onunequip)
    -- ...
end
```

**两个 zip 的区别**：

| zip | bank | 内容 |
|-----|------|------|
| `myaxe.zip` | `myaxe` | 武器掉落在地上时的旋转/呼吸动画（idle anim、可能还有 break anim） |
| `swap_myaxe.zip` | `swap_myaxe` | **没有动画**，只有一个 `swap_myaxe` Symbol——一张静态图 |

#### 第二步：自定义帽子的 onequip / onunequip

戴帽子比武器**复杂得多**——因为还要处理头发的显隐：

```lua
local assets =
{
    Asset("ANIM", "anim/myhat.zip"),        -- 地上掉落物
    Asset("ANIM", "anim/hat_myhat.zip"),    -- 玩家头上的（注意命名约定）
}

local function onequip(inst, owner)
    -- 步骤 1：替换 swap_hat
    owner.AnimState:OverrideSymbol("swap_hat", "hat_myhat", "swap_hat")
    
    -- 步骤 2：处理头发显隐（戴帽子要切换头发组）
    owner.AnimState:Show("HAT")
    owner.AnimState:Show("HAIR_HAT")
    owner.AnimState:Hide("HAIR_NOHAT")
    owner.AnimState:Hide("HAIR")
    
    -- 步骤 3：玩家专属——头部表情切换
    if owner.isplayer then
        owner.AnimState:Hide("HEAD")
        owner.AnimState:Show("HEAD_HAT")
    end
end

local function onunequip(inst, owner)
    owner.AnimState:ClearOverrideSymbol("swap_hat")
    
    owner.AnimState:Hide("HAT")
    owner.AnimState:Hide("HAIR_HAT")
    owner.AnimState:Show("HAIR_NOHAT")
    owner.AnimState:Show("HAIR")
    
    if owner.isplayer then
        owner.AnimState:Show("HEAD")
        owner.AnimState:Hide("HEAD_HAT")
    end
end
```

**关键发现**：玩家身上有**两套头发** Symbol——`HAIR`（无帽子时显示，飘逸的样子）和 `HAIR_HAT`（戴帽子时露出的部分）。戴帽子时切换显示 `HAIR_HAT`，不戴时显示 `HAIR`。

这套约定**对应到 Klei 官方的 hats.lua**——逐字段对照看 `scripts/prefabs/hats.lua` 的 `_onequip/_onunequip`：

```49:74:scripts/prefabs/hats.lua
    local function _onequip(inst, owner, symbol_override, headbase_hat_override)
		_base_onequip(inst, owner, symbol_override)

        owner.AnimState:ClearOverrideSymbol("headbase_hat") --clear out previous overrides
        if headbase_hat_override ~= nil then
            local skin_build = owner.AnimState:GetSkinBuild()
            if skin_build ~= "" then
                owner.AnimState:OverrideSkinSymbol("headbase_hat", skin_build, headbase_hat_override )
            else 
                local build = owner.AnimState:GetBuild()
                owner.AnimState:OverrideSymbol("headbase_hat", build, headbase_hat_override)
            end
        end

        owner.AnimState:Show("HAT")
        owner.AnimState:Show("HAIR_HAT")
        owner.AnimState:Hide("HAIR_NOHAT")
        owner.AnimState:Hide("HAIR")

		if owner.isplayer then
            owner.AnimState:Hide("HEAD")
            owner.AnimState:Show("HEAD_HAT")
			owner.AnimState:Show("HEAD_HAT_NOHELM")
			owner.AnimState:Hide("HEAD_HAT_HELM")
        end
    end
```

`HEAD_HAT_NOHELM` / `HEAD_HAT_HELM` 是更细致的区分——普通帽子（半罩）和全罩头盔（如骑士头盔）显示不同的脑袋形状。

#### 第三步：onequiptomodel —— 给容器穿戴的特殊钩子

某些 prefab（比如假人模型 sewing_mannequin）也能"穿"装备——但它没有完整的 player 模型。`equippable` 组件有第三个回调 `onequiptomodel`：

```lua
function Equippable:Equip(owner, from_ground)
    if self.onequipfn ~= nil then
        self.onequipfn(self.inst, owner, from_ground)
    end

    if self.onequiptomodelfn ~= nil and owner:HasTag("equipmentmodel") then
        self.onequiptomodelfn(self.inst, owner, from_ground)
    end
end
```

如果 owner 有 `equipmentmodel` tag（假人、模型展示），会调 `onequiptomodelfn`。这种情况下你可能要做一些特殊的视觉处理——比如不切换 ARM 组（因为模型没有 ARM_carry/ARM_normal）。

> **新手记忆**：装备一把武器要**OverrideSymbol("swap_object", ...) + Show("ARM_carry") + Hide("ARM_normal")**；戴一顶帽子要**OverrideSymbol("swap_hat", ...) + 一堆 HAIR/HAT 组的显隐切换**。卸下时**ClearOverrideSymbol + 反向显隐**。

---

### 13.5.4 进阶：OverrideSymbol 全家族 API

OverrideSymbol 不是只有一个 API——它是一组**至少 6 个**变体。每个针对不同的场景。来看完整家族。

#### 家族成员一：OverrideSymbol（最基础）

```lua
inst.AnimState:OverrideSymbol(target, source_build, source_symbol)
```

**用途**：基本替换。前面已经讲过。

#### 家族成员二：ClearOverrideSymbol（清除）

```lua
inst.AnimState:ClearOverrideSymbol(target)
```

**用途**：把 target Symbol 恢复回默认 build 的版本。这是 onunequip 必做的一步——否则脱下武器后 swap_object 依然是武器图（虽然透明 ARM_carry 已隐藏，但是数据依然占用内存）。

#### 家族成员三：OverrideSkinSymbol（皮肤专用）

```lua
inst.AnimState:OverrideSkinSymbol(target, skin_build, source_symbol)
```

**用途**：和 `OverrideSymbol` 几乎一样——但是它标记这个 override 是"皮肤来的"。当玩家切换皮肤时，引擎会**自动清掉**所有 SkinSymbol override，普通 OverrideSymbol 不动。

来看 `scripts/prefabs/hats.lua` 第 56 行：

```56:56:scripts/prefabs/hats.lua
                owner.AnimState:OverrideSkinSymbol("headbase_hat", skin_build, headbase_hat_override )
```

这是**皮肤化的头盔**——使用 OverrideSkinSymbol 让它能配合皮肤系统。

#### 家族成员四：OverrideItemSkinSymbol（装备 + 皮肤合体）

```lua
inst.AnimState:OverrideItemSkinSymbol(target, skin_build, source_symbol, item_guid, fname)
```

**用途**：装备本身有皮肤（比如"金色长矛"）。比基础 Override 多两个参数：

- `item_guid` —— 装备实例的 GUID（用于追踪同一件装备的多个 override）
- `fname` —— 备用的"无皮肤回退路径"

来看 nightstick.lua 第 28 行：

```28:28:scripts/prefabs/nightstick.lua
        owner.AnimState:OverrideItemSkinSymbol("swap_object", skin_build, "swap_nightstick", inst.GUID, "swap_nightstick")
```

如果电棍有皮肤（`inst:GetSkinBuild() ~= nil`），用 `OverrideItemSkinSymbol` 而不是普通 OverrideSymbol。

#### 家族成员五：BuildSymbolIsOverridden（查询）

```lua
local is_overridden = inst.AnimState:BuildSymbolIsOverridden("swap_object")
```

**用途**：判断某个 Symbol 当前是不是被替换了。常用调试。

#### 家族成员六：相关的"build 级"管理

```lua
inst.AnimState:AddOverrideBuild(build)        -- 加载并准备某个 build
inst.AnimState:ClearOverrideBuild(build)      -- 卸载某个 override build
inst.AnimState:OverrideMaterial(...)          -- 替换材质（高级）
```

13.5.6 会详讲 AddOverrideBuild。

#### 完整家族汇总表

| API | 主要用途 | 场景 |
|-----|---------|------|
| `OverrideSymbol(t, b, s)` | 基础替换 | 任意 |
| `OverrideSkinSymbol(t, b, s)` | 皮肤替换 | 皮肤系统 |
| `OverrideItemSkinSymbol(t, b, s, guid, fname)` | 装备的皮肤 | 装备 + 皮肤合体 |
| `ClearOverrideSymbol(t)` | 清除 override | 卸下装备时 |
| `BuildSymbolIsOverridden(t)` | 查询状态 | 调试 |
| `AddOverrideBuild(b)` | 预加载 build | DYNAMIC_ANIM |
| `ClearOverrideBuild(b)` | 卸载 override build | 释放内存 |
| `Hide(group)` / `Show(group)` | 组级显隐 | ARM_carry / HAT 等 |
| `HideSymbol(s)` / `ShowSymbol(s)` | 单 Symbol 显隐 | 隐藏特定部位 |

> **进阶记忆**：OverrideSymbol 家族 6 个核心成员——基础（`OverrideSymbol`）/ 皮肤（`OverrideSkinSymbol`）/ 装备的皮肤（`OverrideItemSkinSymbol`）/ 清除（`Clear...`）/ 查询（`BuildSymbolIsOverridden`）/ build 管理（`AddOverrideBuild`）。

---

### 13.5.5 进阶：Hide/Show 组系统——HAT/HAIR/ARM 的显隐配合

`OverrideSymbol` 解决"贴什么图"的问题；但是装备系统还要解决"显示哪些原本部件"。这就是组系统的工作。

#### 第一步：玩家身上的"显隐组"清单

通过 grep 整个 `hats.lua`，能列出玩家身上常见的"组前缀"：

| 组名（大写约定）| 含义 |
|---------|------|
| `HAT` | 帽子整体（戴帽时 Show） |
| `HAIR_HAT` | "戴帽时露出的头发"（如发尾） |
| `HAIR_NOHAT` | "不戴帽时显示的头发" |
| `HAIR` | 完整头发（备用） |
| `HEAD` | 默认无帽脑袋（带表情） |
| `HEAD_HAT` | 戴帽时的脑袋（无完整头发） |
| `HEAD_HAT_NOHELM` | 戴普通帽（不全罩） |
| `HEAD_HAT_HELM` | 戴全罩头盔 |
| `ARM_carry` | 持物的手臂姿态 |
| `ARM_normal` | 自然垂下的手臂 |

**约定**：组名**大写、下划线分隔**——这和 Symbol 名（小写、下划线）形成对比，便于美术和程序员区分。

#### 第二步：组怎么和 Symbol 关联

在 .scml 项目里，可以给一个 Symbol 标记"它属于哪个组"。**实际上，Klei 的 build.bin 里每个 Symbol 都有一个"组成员标志"**——具体见 `buildanimation.py` 但是这个细节没有完全公开。

简化理解：**组就是"一组 Symbol 一起显隐的别名"**。比如 `HAT` 组可能包含：

```
HAT 组
├── swap_hat（默认透明）
├── headbase_hat（戴帽时的头颅形状）
└── face_hat（戴帽时的面部）
```

执行 `Show("HAT")` 让这一整组都可见；执行 `Hide("HAT")` 让一整组都隐藏。

#### 第三步：戴帽完整组切换流程

回看 hats.lua 第 49-74 行的 `_onequip`，我们逐行解析每个 Show/Hide：

```lua
owner.AnimState:Show("HAT")        -- 戴上帽子（让 HAT 组所有 Symbol 可见）
owner.AnimState:Show("HAIR_HAT")   -- 显示"戴帽时的头发部分"
owner.AnimState:Hide("HAIR_NOHAT") -- 隐藏"不戴帽时的头发部分"
owner.AnimState:Hide("HAIR")       -- 完整头发也隐藏（避免重叠）

-- 玩家专属（怪物没有这套表情切换）
if owner.isplayer then
    owner.AnimState:Hide("HEAD")          -- 默认脑袋（带眉毛/眼睛）
    owner.AnimState:Show("HEAD_HAT")      -- 戴帽脑袋
    owner.AnimState:Show("HEAD_HAT_NOHELM")  -- 普通帽
    owner.AnimState:Hide("HEAD_HAT_HELM")    -- 不显示全罩头盔形态
end
```

**关键洞察**：玩家有**4 种脑袋形态**：
- `HEAD` —— 不戴帽（默认）
- `HEAD_HAT` —— 戴帽（通用）
- `HEAD_HAT_NOHELM` —— 戴普通帽（露出眉毛眼睛）
- `HEAD_HAT_HELM` —— 戴全罩头盔（全黑只露眼缝）

戴普通帽时显示 `HEAD_HAT + HEAD_HAT_NOHELM`，戴全罩头盔时显示 `HEAD_HAT + HEAD_HAT_HELM`——两层组合标志精确控制。

#### 第四步：全罩头盔的特殊处理

来看 hats.lua 第 129-148 行的 `fullhelm_onequip`：

```129:148:scripts/prefabs/hats.lua
	fns.fullhelm_onequip = function(inst, owner)
		if owner.isplayer then
			_base_onequip(inst, owner, nil, "headbase_hat")

			owner.AnimState:Hide("HAT")
			owner.AnimState:Hide("HAIR_HAT")
			owner.AnimState:Hide("HAIR_NOHAT")
			owner.AnimState:Hide("HAIR")

			owner.AnimState:Hide("HEAD")
			owner.AnimState:Show("HEAD_HAT")
			owner.AnimState:Hide("HEAD_HAT_NOHELM")
			owner.AnimState:Show("HEAD_HAT_HELM")

			owner.AnimState:HideSymbol("face")
			owner.AnimState:HideSymbol("swap_face")
			owner.AnimState:HideSymbol("beard")
			owner.AnimState:HideSymbol("cheeks")

			owner.AnimState:UseHeadHatExchange(true)
```

读懂：

- 全罩头盔**不显示** HAT 组（没有"普通帽子贴在头顶"）
- **显示** HEAD_HAT + HEAD_HAT_HELM（全罩脑袋形态）
- **额外隐藏** face / swap_face / beard / cheeks 这些面部 Symbol——因为头盔是封闭的，里面不需要显示面部
- `UseHeadHatExchange(true)` —— 启用"头盔交换"特殊渲染（让头盔正确遮挡头发的渲染层级）

#### 第五步：手臂显隐 —— ARM_carry vs ARM_normal

回看 nightstick.lua：

```33:34:scripts/prefabs/nightstick.lua
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
```

`ARM_carry` 和 `ARM_normal` 是 Wilson build 里的**两套手臂关键帧**：

- `ARM_carry` —— 手臂"举起来拿东西"的姿势（hand 在 swap_object 附近）
- `ARM_normal` —— 手臂"自然垂下"的姿势（hand 在臀部旁）

引擎不会自动判断"玩家手里有没有东西"——你必须在 onequip / onunequip 里**手动切换**。

> **进阶记忆**：组系统是装备/换装的**第二支柱**（第一支柱是 OverrideSymbol）。重要组——HAT / HAIR_HAT / HAIR_NOHAT / HAIR / HEAD / HEAD_HAT / HEAD_HAT_NOHELM / HEAD_HAT_HELM / ARM_carry / ARM_normal。**组级 Show/Hide 必须配合 OverrideSymbol 一起调用**。

---

### 13.5.6 进阶：AddOverrideBuild + DYNAMIC_ANIM

#### 第一步：何时要用 DYNAMIC_ANIM

13.1.6 讲过 `DYNAMIC_ANIM` 是"延迟加载"模式。**什么场景用它**？

- 皮肤——一个角色可能有 50 个皮肤，全加载浪费内存
- 罕见装备——只有少数玩家会用到的特殊武器
- 季节性内容——只在特定节日显示的装饰

```lua
-- modmain.lua
Assets = {
    Asset("DYNAMIC_ANIM", "anim/dynamic/swap_legendary_sword.zip"),
    Asset("PKGREF", "anim/dynamic/swap_legendary_sword.dyn"),
}
```

`PKGREF` 的 `.dyn` 是"虚拟标记"（13.1.6 讲过）——告诉引擎"这个 zip 里有哪些 build/anim"，不真的读 zip。

#### 第二步：用 DYNAMIC_ANIM 时的换装代码

```lua
local function onequip(inst, owner)
    -- 步骤 1：先加载 build！（DYNAMIC_ANIM 必须显式触发加载）
    owner.AnimState:AddOverrideBuild("swap_legendary_sword")
    
    -- 步骤 2：然后才能 OverrideSymbol
    owner.AnimState:OverrideSymbol("swap_object", "swap_legendary_sword", "swap_legendary_sword")
    
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end

local function onunequip(inst, owner)
    owner.AnimState:ClearOverrideSymbol("swap_object")
    
    -- 步骤 3：可选——卸载 build 释放内存
    owner.AnimState:ClearOverrideBuild("swap_legendary_sword")
    
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
end
```

**注意区别普通 ANIM**：

| 资源类型 | onequip 操作 |
|---------|-------------|
| `Asset("ANIM", ...)` | 引擎启动时已加载，直接 OverrideSymbol |
| `Asset("DYNAMIC_ANIM", ...)` | 必须先 `AddOverrideBuild`，再 OverrideSymbol |

#### 第三步：AddOverrideBuild 的内存模型

调用 `AddOverrideBuild("swap_legendary_sword")` 后：

1. 引擎打开 `anim/dynamic/swap_legendary_sword.zip`
2. 解压 build.bin、atlas.tex 到内存
3. 标记当前实体"持有这个 build 的引用"
4. 同一个 build 被多次 AddOverrideBuild 时只加载一次（引用计数）

调用 `ClearOverrideBuild("swap_legendary_sword")` 后：

1. 减少引用计数
2. 计数归零时释放 build 占用的内存

**陷阱**：如果你 AddOverrideBuild 但是从不 ClearOverrideBuild，**内存只增不减**。某些 mod 因此造成"长时间游玩后内存膨胀"——必须配对调用。

#### 第四步：实际案例——神话 mod 的皮肤批量加载

来看神话 mod 的 modmain.lua（13.1.6 引用过的代码）：

```lua
for _,v in ipairs(mk_skin_assets) do
    table.insert(Assets, Asset("DYNAMIC_ANIM", "anim/dynamic/"..v..".zip"))
    table.insert(Assets, Asset("PKGREF", "anim/dynamic/"..v..".dyn"))
end
```

这里批量声明了几十个 DYNAMIC_ANIM 资源——但是它们**全部不会在游戏启动时加载**。只有当玩家真正穿上某件神话装备时，对应的 build 才被 `AddOverrideBuild` 加载。

#### 第五步：何时一定要 AddOverrideBuild、何时不需要

| 场景 | 需要 AddOverrideBuild? |
|------|----------------------|
| Asset("ANIM", "anim/swap_axe.zip") | 不需要 |
| Asset("DYNAMIC_ANIM", ...) | **需要** |
| Klei 的官方皮肤（系统已自动 AddOverrideBuild）| 不需要手动调（除非你改了系统）|
| `OverrideSymbol` 用了**没声明过**的 build | 失败，必须先声明 |

> **进阶记忆**：DYNAMIC_ANIM 用于"按需加载"——节省启动时内存。**必须先 `AddOverrideBuild` 再 `OverrideSymbol`**，**配对 `ClearOverrideBuild` 避免泄漏**。

---

### 13.5.7 老手：Skinner 系统 + OverrideSkinSymbol 多层覆盖

掌握了 OverrideSymbol 三参数后，你已经能写出 95% 的 mod 了。剩下的 5% 是**皮肤系统**——Klei 用一个独立的组件 `Skinner` 把这套逻辑封装起来，让玩家选择皮肤时只需调一个 API。

#### 第一步：Skinner 组件总览

`scripts/components/skinner.lua` 是玩家专用组件——管理玩家的当前皮肤。它的核心方法：

```lua
inst.components.skinner:SetSkinName("wilson_formal")  -- 切换到正装 Wilson
inst.components.skinner:SetSkinName("wilson_none")    -- 恢复默认
```

调用 `SetSkinName` 后，组件会自动调用 `SetSkinMode`，最终调到 `SetSkinsOnAnim`——这个函数才是真正干活的：

```11:14:scripts/components/skinner.lua
function SetSkinsOnAnim( anim_state, prefab, base_skin, clothing_names, monkey_curse, skintype, default_build )
```

#### 第二步：SetSkin —— 第一层覆盖

```44:44:scripts/components/skinner.lua
		anim_state:SetSkin(base_skin, default_build)
```

`SetSkin(skin_build, default_build)` 是 AnimState 的特殊 API。它**不像 SetBuild 那样替换整个 build**——而是建立一个**两层 build 体系**：

- 上层（皮肤）—— `skin_build`（比如 `wilson_formal`）：包含修改过的 Symbol（西装、不同发型等）
- 下层（默认）—— `default_build`（`wilson`）：所有皮肤没修改的 Symbol 走这里

引擎渲染每个 Symbol 时**先**查皮肤层有没有，**没有**才查默认层——这就是为什么皮肤只画"修改的部分"就够了，不用重做整个角色。

#### 第三步：OverrideSkinSymbol —— 第二层覆盖

某些皮肤还会**针对特定部位**做更精细的修改。比如要让"沙漠 Wilson"的躯干和默认 Wilson 不一样——但是**用的不是 wilson_desert build 里的 torso Symbol，而是另一个 zip**。

来看 skinner.lua 第 326-340 行（简化）：

```lua
if torso_symbol ~= nil then
    anim_state:OverrideSkinSymbol("torso", base_skin, torso_symbol )
end
if pelvis_symbol ~= nil then
    anim_state:OverrideSkinSymbol("torso_pelvis", base_skin, pelvis_symbol )
end
if skirt then
    anim_state:OverrideSkinSymbol("skirt", base_skin, "skirt_wide")
end
```

**这是 Klei 的"组件化皮肤"系统**——一个皮肤可以由多个独立的"零件"组成：torso（躯干）+ pelvis（骨盆）+ skirt（裙子/裤子）+ leg（腿）+ foot（脚）。每个零件独立 OverrideSkinSymbol。

#### 第四步：换皮肤时为什么不用 ClearOverrideSymbol

注意 skinner.lua 中**几乎没有** ClearOverrideSymbol——只有 OverrideSkinSymbol。这是因为：

> **OverrideSkinSymbol 标记的是"皮肤来源"——切皮肤时引擎自动清掉所有 SkinSymbol，普通 OverrideSymbol（装备）不动。**

这个机制让"换皮肤"和"换装备"互不干扰。你脱掉斧头、换一个皮肤、再戴帽子——三个操作各走各的覆盖通道，不会互相清掉。

#### 第五步：内部数据结构

简化一下 AnimState 内部的"覆盖表"：

```
override_table = {
    "swap_object" → { source = "swap_axe", build = "swap_axe", type = ITEM },     -- 装备
    "headbase_hat" → { source = "headbase_hat", build = "wilson_formal", type = SKIN }, -- 皮肤帽子
    "torso" → { source = "torso", build = "wilson_formal", type = SKIN },          -- 皮肤躯干
}
```

每条记录有个 `type` 字段（ITEM / SKIN / ITEM_SKIN）。**切皮肤**时清掉所有 SKIN 类型；**卸装备**时清掉特定 ITEM 类型——精确管理。

#### 第六步：实战——给玩家加一个临时变身

mod 里想做"使用药水变身 5 秒"的效果：

```lua
local function onuse(inst, doer)
    -- 切到一个临时 build
    doer.AnimState:OverrideSymbol("torso", "monster_torso", "torso")
    doer.AnimState:OverrideSymbol("head", "monster_head", "head")
    doer.AnimState:OverrideSymbol("hand", "monster_hand", "hand")
    
    -- 5 秒后还原
    doer:DoTaskInTime(5, function()
        doer.AnimState:ClearOverrideSymbol("torso")
        doer.AnimState:ClearOverrideSymbol("head")
        doer.AnimState:ClearOverrideSymbol("hand")
    end)
end
```

**注意**：这种"临时变身"用普通 `OverrideSymbol` + `ClearOverrideSymbol`——**不要**用 SkinSymbol，否则玩家切皮肤时变身会被意外清除。

> **老手记忆**：Skinner = 玩家皮肤管理组件、自动调 SetSkin + 多个 OverrideSkinSymbol。皮肤体系是**两层 build**（skin_build 上层 + default_build 下层）+ **多个组件 override**（torso/torso_pelvis/skirt/leg/foot）。**切皮肤时引擎自动清 SkinSymbol，装备的 OverrideSymbol 不受影响**。

---

### 13.5.8 老手：八个最容易踩的坑

#### 坑 1：Override 的 Symbol 名拼错

```lua
inst.AnimState:OverrideSymbol("swap_object", "swap_myaxe", "swap_my_axe")
                                                            -- 多了下划线！
```

**症状**：调用没报错，但是 swap_object 没被替换——还是透明的。

**原因**：source_symbol 必须**精确**等于源 build 里的 Symbol 名。

**调试**：用 krane.exe 解 zip，看 build.xml 里到底有哪些 Symbol。

#### 坑 2：忘了 Show("ARM_carry")

```lua
local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_object", "swap_myaxe", "swap_myaxe")
    -- 忘了切 ARM 组！
end
```

**症状**：玩家手里出现剑——但是手臂还是垂着的，剑漂浮在屁股旁边。

**修复**：永远记得 `Show("ARM_carry") + Hide("ARM_normal")`。

#### 坑 3：onunequip 漏掉 ClearOverrideSymbol

```lua
local function onunequip(inst, owner)
    -- 忘了 ClearOverrideSymbol("swap_object")！
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
end
```

**症状**：玩家脱掉武器，手臂垂下了——但是某些动画里还能看到"幽灵剑"漂浮（因为 swap_object 还有数据）。

**修复**：永远成对——onequip 调 OverrideSymbol，onunequip 必须调 ClearOverrideSymbol。

#### 坑 4：DYNAMIC_ANIM 漏 AddOverrideBuild

```lua
local assets = {
    Asset("DYNAMIC_ANIM", "anim/dynamic/myaxe.zip"),
}

local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_object", "myaxe", "swap_myaxe")
    -- 没有 AddOverrideBuild！
end
```

**症状**：装备时玩家手里啥也没有（透明）。

**原因**：DYNAMIC_ANIM 不会自动加载——必须 AddOverrideBuild。

**修复**：

```lua
owner.AnimState:AddOverrideBuild("myaxe")
owner.AnimState:OverrideSymbol("swap_object", "myaxe", "swap_myaxe")
```

#### 坑 5：戴帽子忘了 Hide("HAIR")

```lua
local function onequip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_hat", "hat_myhat", "swap_hat")
    owner.AnimState:Show("HAT")
    -- 忘了 Hide("HAIR_NOHAT") + Hide("HAIR")
end
```

**症状**：戴帽子之后头发依然是不戴帽时的造型——发尾盖住帽子。

**修复**：戴帽时**4 个组都要切**：Show("HAIR_HAT") + Hide("HAIR_NOHAT") + Hide("HAIR") + Show("HAT")。

#### 坑 6：误用 OverrideSkinSymbol 给装备贴图

```lua
-- 错！装备应该用 OverrideSymbol 不是 OverrideSkinSymbol
owner.AnimState:OverrideSkinSymbol("swap_object", "swap_myaxe", "swap_myaxe")
```

**症状**：玩家切皮肤时手里的剑**意外消失**。

**原因**：OverrideSkinSymbol 标记是"皮肤来的"——切皮肤时引擎自动清掉。装备应该用普通 OverrideSymbol。

**修复**：**装备用 `OverrideSymbol`、皮肤用 `OverrideSkinSymbol`、装备的皮肤用 `OverrideItemSkinSymbol`**——三者各司其职。

#### 坑 7：Hide 组之后又 Show（影响新装备）

```lua
local function onequip(inst, owner)
    owner.AnimState:Hide("HAT")  -- 这件帽子不显示帽顶？
end

local function onunequip(inst, owner)
    -- 没有 Show("HAT")！
end
```

**症状**：脱掉这件帽子之后，**下一次戴别的帽子**也不显示。

**原因**：Hide("HAT") 的状态会**保留**——即使切到别的装备。

**修复**：onunequip 里要**显式恢复**所有 Show/Hide 调用。或者用更细的 HideSymbol 而不是组级 Hide。

#### 坑 8：AddOverrideBuild 不配对 ClearOverrideBuild

```lua
-- 频繁切换装备
local function onequip(inst, owner)
    owner.AnimState:AddOverrideBuild("myaxe")
    owner.AnimState:OverrideSymbol("swap_object", "myaxe", "swap_myaxe")
end

local function onunequip(inst, owner)
    owner.AnimState:ClearOverrideSymbol("swap_object")
    -- 没有 ClearOverrideBuild！
end
```

**症状**：玩家反复装备/卸下装备，**内存稳步上涨**——5 小时游戏后崩溃。

**修复**：配对调用——AddOverrideBuild 配 ClearOverrideBuild。

注意：实际生产中，**绝大多数装备其实用普通 ANIM**（启动时加载、永不释放）——这种情况下不用 AddOverrideBuild。**只有 DYNAMIC_ANIM** 才会有这个泄漏问题。

> **老手记忆**：8 个 OverrideSymbol 坑里——3 个是**漏调** API（坑 2、3、4）、2 个是**API 误用**（坑 6、8）、3 个是**约定不全**（坑 1、5、7）。养成"成对调用 + 显式恢复 + 选对 API 变体"的习惯。

---

### 13.5 小结：关于 OverrideSymbol 你必须记住的

```
                ┌────────────────────────────────────┐
                │   AnimState 内部覆盖表             │
                │                                    │
                │  swap_object  ─┐                   │
                │  swap_hat     ─┤  ITEM 类型         │
                │  swap_body    ─┘  (装备)           │
                │                                    │
                │  torso        ─┐                   │
                │  torso_pelvis ─┤  SKIN 类型         │
                │  skirt        ─┘  (皮肤，自动清)    │
                │                                    │
                │  swap_legendary─┐                  │
                │                 ┴ ITEM_SKIN 类型   │
                │                   (装备的皮肤)      │
                └────────────────────────────────────┘
                               ▲
              ┌────────────────┴───────────────┐
              │                                │
   OverrideSymbol(t,b,s)              OverrideSkinSymbol(t,b,s)
   OverrideItemSkinSymbol(...)        ClearOverrideSymbol(t)
   AddOverrideBuild(b)                BuildSymbolIsOverridden(t)
```

**新手核心三句**：换装 = OverrideSymbol("swap_xxx", source_build, source_symbol)。**位置/旋转来自 anim、贴图来自 build**。永远成对调用 OverrideSymbol + ClearOverrideSymbol 和 Show/Hide 配对。

**进阶核心三句**：OverrideSymbol 家族 6 个变体——基础 / 皮肤（OverrideSkinSymbol）/ 装备的皮肤（OverrideItemSkinSymbol）/ 清除 / 查询 / build 管理。组系统（HAT / HAIR_HAT / HAIR_NOHAT / ARM_carry 等）是装备/换装的**第二支柱**。DYNAMIC_ANIM 必须先 AddOverrideBuild 再 OverrideSymbol。

**老手核心三句**：Skinner 用两层 build 体系（skin 上层 + default 下层）+ 多个组件 OverrideSkinSymbol（torso/torso_pelvis/skirt/leg/foot）实现皮肤系统。**切皮肤时引擎自动清 SkinSymbol，装备的 OverrideSymbol 不受影响**——这是为什么"装备用 OverrideSymbol、皮肤用 OverrideSkinSymbol、装备的皮肤用 OverrideItemSkinSymbol"的三选一规则必须遵守。

下一节 13.6 我们换一个完全不同的话题——**散装 atlas 和 inventoryimages**。前面我们都在讨论"骨骼动画"（角色 + 装备的运动），但是**库存图标**、**配方图标**、**地图标记**这些 UI 元素是**静态散装图集**——用一对 `.tex + .xml` 而不是 zip 来管理。从 13.6 开始，会切换到 UI 资源的世界。

---


## 13.6 地图图标、物品图标的制作（Atlas 与 Inventoryimages）

### 本节导读

13.1 ~ 13.5 我们花了五节讲**骨骼动画**——zip 包、Symbol 替换、换装系统。但是饥荒里有大量的"美术资源"**根本不是骨骼动画**：

- 库存格里的物品图标（64×64 的小图）
- 鼠标 hover 时弹出的 tooltip 图
- 配方面板里的食物预览
- 地图上的 POI 标记
- 图鉴页面的物种插画

这些都是**纯静态图片**——没有骨架、不需要动画、就是一张矩形像素。它们用一种**完全不同的资源管线**：

> **散装 Atlas**（一对 `.tex + .xml`，不是 zip）

本节就来吃透这套管线。

> **新手**从 13.6.1-13.6.3 起步——理解散装 atlas 和 zip 的本质区别、库存图标从美术 PNG 到游戏中显示的最短路径、Klei 的 inventoryimages1~4.xml 自动查找机制；**进阶读者**继续看 13.6.4-13.6.6，深入 `RegisterInventoryItemAtlas` / `GetInventoryItemAtlas` 完整链路、`inventoryitem` 组件的 `atlasname/imagename/ChangeImageName` 三件套、自定义 atlas 的 `.xml` 内部结构和编译工具链；**老手**跳到 13.6.7-13.6.8，掌握 scrapbook 图鉴 atlas、皮肤变体图标、动态切换场景、八个最容易踩的坑。

读完本节，你**应该能做到**：① 给一件自定义物品做出库存图标和场景图标；② 让物品在配方面板/图鉴页面正确显示；③ 调试"图标显示成紫色棋盘格"这种经典坑。

---

### 13.6.1 快速入门：散装 atlas 是什么？

#### 第一步：对比 zip 和散装 atlas 的本质

13.1 我们看到 anim/xxx.zip 的内部是 4 个文件：

```
xxx.zip
├── atlas-0.tex
├── atlas-0.xml
├── build.bin    ← 骨架
└── anim.bin     ← 动作
```

**散装 atlas** 直接是这 4 个文件中的前 2 个，没有 zip 包装、没有 build.bin、没有 anim.bin：

```
images/
├── myicon.tex   ← 像素数据（DXT5 压缩）
└── myicon.xml   ← 索引（每个小图在 .tex 里的 UV 坐标）
```

| 性质 | zip 骨骼动画 | 散装 atlas |
|------|------------|-----------|
| 文件结构 | 一个 .zip 内含 4 个文件 | 一对独立的 .tex + .xml |
| 是否有动画 | 是（关键帧序列）| 否（纯静态）|
| 是否有骨架 | 是（Symbol 之间的位置关系）| 否（每张图独立矩形）|
| 用途 | 角色 / 怪物 / 装备 / 大型实体 | 库存图标 / UI / 地图标记 / 配方图 |
| 资源声明 | `Asset("ANIM", "anim/xxx.zip")` | **`Asset("ATLAS", ...) + Asset("IMAGE", ...)` 一对** |

#### 第二步：散装 atlas 的最简案例

最小的散装 atlas——**一张图 + 一个索引**：

`images/myitem.tex` —— 二进制 DXT 纹理（Klei 自定义格式）

`images/myitem.xml` —— 描述这张纹理里的每个小图：

```xml
<Atlas>
    <Texture filename="myitem.tex" />
    <Elements>
        <Element name="myitem.tex" u1="0.0" u2="1.0" v1="0.0" v2="1.0" />
    </Elements>
</Atlas>
```

字段含义：

| 字段 | 含义 |
|------|------|
| `<Texture filename>` | 关联的 .tex 文件（同目录下）|
| `<Element name>` | "图标的名字"——代码中通过这个名字访问 |
| `u1, v1, u2, v2` | 这张小图在大图中的 UV 矩形坐标（0-1 归一化）|

如果整张 .tex 只有一个图，UV 范围就是 `(0,0)-(1,1)`——占满整张。如果是多图集，每个小图占一个矩形。

#### 第三步：和 zip 内的 atlas-0.xml 完全同样的格式

打开任何 zip 包内的 `atlas-0.xml`——结构和散装 atlas 的 .xml**完全一样**！

这就是为什么散装 atlas 不需要 build.bin / anim.bin——它**就是 zip 内 atlas 部分的"独立版本"**。

#### 第四步：为什么用散装而不是 zip

对于 UI 图标这种**纯静态、单帧、无动画**的东西，把它强行塞进 zip + 写一个空的 anim.bin 是浪费——所以 Klei 给了散装 atlas 这个轻量级方案。

| 优势 | 散装 atlas | zip |
|------|-----------|-----|
| 文件大小 | 小（少了 build.bin + anim.bin 元数据）| 大 |
| 加载速度 | 快 | 慢 |
| 编辑工具 | 任意图像编辑器 + ktools | 必须用 Spriter |
| 适用 | 静态 UI / 图标 | 角色 / 怪物 / 装备 |

> **新手记忆**：散装 atlas = `.tex（像素）+ .xml（索引）`；用于**纯静态**的库存图标、UI、地图标记。和 zip 内的 atlas 结构相同，但少了 build.bin / anim.bin。

---

### 13.6.2 快速入门：库存图标从无到有

#### 第一步：美术资源准备

最简单的库存图标做法——**画一张 64×64 的透明 PNG**，然后转换成 .tex + .xml：

```
my_icon_source.png  (64x64, RGBA)
        │
        ▼  [TEXcreator / ktools / krane]
images/myitem.tex    ← Klei 自定义纹理格式
images/myitem.xml    ← 一张小图的索引
```

工具链选择：

| 工具 | 来源 | 难度 |
|------|------|------|
| **TEXcreator** | Klei mod 论坛社区 | 易 |
| **ktools** | 开源命令行工具（GitHub）| 中 |
| **mod_tools/autocompiler.exe** | Klei 官方 | 易（自动）|

**最简单**：把 PNG 放在 mod 的 `exported/` 目录下，跑 autocompiler，自动转成 .tex + .xml。

#### 第二步：声明资源

库存图标必须**用一对 ATLAS + IMAGE 声明**——少一个引擎都会报错（13.1.8 坑 1）：

```lua
local assets =
{
    Asset("ATLAS", "images/inventoryimages/myitem.xml"),
    Asset("IMAGE", "images/inventoryimages/myitem.tex"),
}
```

`Asset("ATLAS", ...)` 告诉引擎索引 .xml 在哪；`Asset("IMAGE", ...)` 告诉引擎像素 .tex 在哪。引擎会自动配对。

#### 第三步：让物品用上这个图标

inventoryitem 组件有两个字段控制库存图标：

```lua
inst.components.inventoryitem.atlasname = "images/inventoryimages/myitem.xml"
inst.components.inventoryitem.imagename = "myitem"
```

| 字段 | 含义 |
|------|------|
| `atlasname` | 完整 atlas 路径（默认会用 inventoryimages1.xml ~ 4.xml）|
| `imagename` | atlas 内的元素名（不带 .tex 后缀，引擎会自动加）|

**或者**用 `Asset("INV_IMAGE", "myitem")` 声明 + 注册到 Klei 的查找路径——见 13.6.3。

#### 第四步：完整 prefab 模板

```lua
local assets =
{
    Asset("ANIM", "anim/myitem.zip"),  -- 物品在地上的形象
    Asset("ATLAS", "images/inventoryimages/myitem.xml"),
    Asset("IMAGE", "images/inventoryimages/myitem.tex"),
}

local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    inst.AnimState:SetBank("myitem")
    inst.AnimState:SetBuild("myitem")
    inst.AnimState:PlayAnimation("idle")

    MakeInventoryPhysics(inst)

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then return inst end

    inst:AddComponent("inventoryitem")
    inst.components.inventoryitem.atlasname = "images/inventoryimages/myitem.xml"
    inst.components.inventoryitem.imagename = "myitem"

    inst:AddComponent("inspectable")

    return inst
end

return Prefab("myitem", fn, assets)
```

进游戏 `c_give("myitem")` —— 物品出现在库存格里、显示我们做的图标。**完成**。

> **新手记忆**：库存图标 = **ATLAS + IMAGE 一对资源 + atlasname/imagename 两个字段**。最简流程：画 PNG → autocompiler 转 .tex/.xml → Asset 声明 → inventoryitem 字段。

---

### 13.6.3 快速入门：Klei 的 inventoryimages1~4.xml 查找机制

每次新做一个 mod 物品都要在 prefab 里写 `atlasname = "images/..."` 太麻烦——所以 Klei 提供了**自动查找**：你只声明 `Asset("INV_IMAGE", "myitem")`，引擎会在**4 个汇总 atlas** 里查找名为 `myitem.tex` 的小图。

#### 第一步：4 个汇总 atlas 是什么

```666:676:scripts/simutil.lua
function GetInventoryItemAtlas_Internal(imagename, no_fallback)
    local images1 = "images/inventoryimages1.xml"
    local images2 = "images/inventoryimages2.xml"
    local images3 = "images/inventoryimages3.xml"
    local images4 = "images/inventoryimages4.xml"
    return TheSim:AtlasContains(images1, imagename) and images1
            or TheSim:AtlasContains(images2, imagename) and images2
            or TheSim:AtlasContains(images3, imagename) and images3
            or (not no_fallback or TheSim:AtlasContains(images4, imagename)) and images4
            or nil
end
```

**这 4 个 atlas 是 Klei 官方的"大杂烩"**——把所有官方物品的库存图标打包在一起。打开 `data/databundles/anim.zip` 解压会看到 `images/inventoryimages1.xml` ~ `images/inventoryimages4.xml` 各自包含上百个图标。

#### 第二步：查找逻辑

`TheSim:AtlasContains(atlas_path, image_name)` —— 这是 C++ 端导出的函数，**不解压 .tex**，只读 .xml 索引看里面有没有对应的 `<Element name>`。

整个查找是**短路求值**：

```lua
return AtlasContains(images1, name) and images1
    or AtlasContains(images2, name) and images2
    or AtlasContains(images3, name) and images3
    or AtlasContains(images4, name) and images4
```

依次检查 4 个 atlas——找到就返回那个 atlas 的路径。

#### 第三步：缓存机制

```683:695:scripts/simutil.lua
function GetInventoryItemAtlas(imagename, no_fallback)
	local atlas = inventoryItemAtlasLookup[imagename]
	if atlas then
		return atlas
	end

    atlas = GetInventoryItemAtlas_Internal(imagename, no_fallback)

	if atlas ~= nil then
		inventoryItemAtlasLookup[imagename] = atlas
	end
	return atlas
end
```

第一次查询时遍历 4 个 atlas；之后**结果存入 `inventoryItemAtlasLookup` 表**，下次直接返回。这是**性能优化**——避免每次开库存都遍历查找。

#### 第四步：mod 怎么"插入"自己的 atlas

普通 mod 的物品图标**不在** inventoryimages1~4.xml 里——所以 mod 要么显式指定 atlasname（13.6.2 的方法）、要么**手动注册**到查找表里。后者更优雅——见 13.6.4。

> **新手记忆**：Klei 把所有官方库存图标打包成 inventoryimages1.xml~4.xml 4 个汇总文件，引擎用 `GetInventoryItemAtlas` 自动查找+缓存。**mod 物品不在这 4 个里**——必须**指定 atlasname** 或**注册到查找表**。

---

### 13.6.4 进阶：RegisterInventoryItemAtlas 完整链路

`RegisterInventoryItemAtlas(atlas, imagename)` 把"图名 → atlas 路径"塞进 `inventoryItemAtlasLookup` 缓存表，让后续 `GetInventoryItemAtlas("imagename")` 能直接返回。

#### 第一步：源码定位

```652:664:scripts/simutil.lua
local inventoryItemAtlasLookup = {}

function RegisterInventoryItemAtlas(atlas, imagename)
	if atlas ~= nil and imagename ~= nil then
		if inventoryItemAtlasLookup[imagename] ~= nil then
			if inventoryItemAtlasLookup[imagename] ~= atlas then
				print("RegisterInventoryItemAtlas: Image '" .. imagename .. "' is already registered to atlas '" .. atlas .."'")
			end
		else
			inventoryItemAtlasLookup[imagename] = atlas
		end
	end
end
```

**关键**：

- 直接往 `inventoryItemAtlasLookup` 表里塞 `imagename → atlas` 映射
- **重复注册同一个 imagename 到不同 atlas 会打印警告**（不是 error，但是冲突要小心）
- 注册的 imagename 必须**带 `.tex` 后缀**——这是 GetInventoryItemAtlas 内部的查询 key

#### 第二步：deck_of_cards 的实战示范

来看 `scripts/prefabs/deck_of_cards.lua` 的标准做法：

```14:22:scripts/prefabs/deck_of_cards.lua
if rawget(_G, "RegisterInventoryItemAtlas") then
    for s=1, TUNING.PLAYINGCARDS_NUM_SUITS do
        for n=1, TUNING.PLAYINGCARDS_NUM_PIPS do
            RegisterInventoryItemAtlas("images/playingcards.xml", "playingcardstand".. (s*100 + n) ..".tex")
        end
    end

    RegisterInventoryItemAtlas("images/playingcards.xml", "playingcardstandback.tex")
end
```

读懂：

- `if rawget(_G, "RegisterInventoryItemAtlas")` —— 先检查这个全局函数是否存在（兼容旧版本/某些受限上下文）
- 循环注册 `playingcardstand101.tex` ~ `playingcardstand409.tex`（4 花色 × 9 数字）和背面图
- 全部指向同一个 atlas：`images/playingcards.xml`

之后任何代码（比如卡牌的 inventoryitem 组件）调用 `GetInventoryItemAtlas("playingcardstand101.tex")` 都能找到 `images/playingcards.xml`——不再需要遍历 4 个 inventoryimages。

#### 第三步：mod 端的标准注册流程

```lua
-- modmain.lua
Assets = {
    Asset("ATLAS", "images/myicons.xml"),
    Asset("IMAGE", "images/myicons.tex"),
}

-- 注册 atlas 中所有图标到查找表
local my_inventory_icons = {
    "myitem1", "myitem2", "myitem3",
}

if rawget(_G, "RegisterInventoryItemAtlas") then
    for _, name in ipairs(my_inventory_icons) do
        RegisterInventoryItemAtlas("images/myicons.xml", name..".tex")
    end
end
```

**这种方式的好处**：

1. mod 物品的 inventoryitem 组件**不需要设 atlasname**——自动查找
2. `GetInventoryItemAtlas` 缓存让后续查询 O(1)
3. 多个物品共享同一个 atlas 节省内存

#### 第四步：有 RegisterInventoryItemAtlas 还要 INV_IMAGE 吗？

```lua
local assets = {
    Asset("ATLAS", "images/myicons.xml"),
    Asset("IMAGE", "images/myicons.tex"),
    Asset("INV_IMAGE", "myitem"),  -- 还要这个吗？
}
```

**不需要**——`INV_IMAGE` 是 13.1.6 讲过的特殊 Asset 类型，会被 `ShouldIgnoreResolve` 跳过路径解析。它实际上等于"提示引擎我会用到这张图"，但是没有 RegisterInventoryItemAtlas + ATLAS + IMAGE 也能正常工作。

实际上**ATLAS + IMAGE 是必须的**（保证文件被加载），`RegisterInventoryItemAtlas` 是建议的（让自动查找命中），`INV_IMAGE` 是可选的（旧版兼容）。

> **进阶记忆**：`RegisterInventoryItemAtlas("atlas路径", "图名.tex")` 是 mod 的标准做法。**注册的 imagename 必须带 .tex 后缀**。同一图名重复注册会打印警告。

---

### 13.6.5 进阶：inventoryitem 组件的 atlasname / imagename / ChangeImageName

#### 第一步：组件初始化时的字段

```72:73:scripts/components/inventoryitem.lua
    self.atlasname = nil
    self.imagename = nil
```

inventoryitem 组件创建时这两个字段默认是 nil。**nil 时引擎会调 GetInventoryItemAtlas 自动查找**。

#### 第二步：netvar 同步

```93:104:scripts/components/inventoryitem.lua
{
    atlasname = onatlasname,
    imagename = onimagename,
    owner = onowner,
    ...
})
```

这两个字段是**网络同步**的——服务器侧改变时客户端会收到。回调函数：

```1:7:scripts/components/inventoryitem.lua
local function onatlasname(self, atlasname)
    self.inst.replica.inventoryitem:SetAtlas(atlasname)
end

local function onimagename(self, imagename)
    self.inst.replica.inventoryitem:SetImage(imagename)
end
```

也就是说服务器侧 `inst.components.inventoryitem.atlasname = "..."` 会通过 replica 同步到客户端 UI。

#### 第三步：ChangeImageName —— 动态切换图标

```364:367:scripts/components/inventoryitem.lua
function InventoryItem:ChangeImageName(newname)
    self.imagename = newname
    self.inst:PushEvent("imagechange")
end
```

`ChangeImageName` 是切换图标的标准方法。它做两件事：

1. **改 imagename** —— 触发上面的 onimagename netvar 回调，同步到客户端
2. **推 imagechange 事件** —— 让监听者知道要刷新（库存格 UI 收到这个事件会重绘）

#### 第四步：动态切换的实战场景

**场景一：玩家选了皮肤**——`prefabskin.lua:120`：

```120:120:scripts/prefabskin.lua
        inst.components.inventoryitem:ChangeImageName(skin_name)
```

玩家给一个普通箱子贴了皮肤 `chest_classy`——库存图标也要换成对应的皮肤版本。`ChangeImageName("chest_classy")` 让 UI 自动找到那个图标。

**场景二：箱子打开/关闭状态切换**——`prefabskin.lua:117-119`：

```117:121:scripts/prefabskin.lua
        if inst.components.container ~= nil and inst.components.container:IsOpen() then
            skin_name = skin_name .. "_open"
        end
        inst.components.inventoryitem:ChangeImageName(skin_name)
```

如果箱子打开，图标名加 `_open` 后缀——显示"打开的箱子"图。

**场景三：墓碑随机外观**——`prefabskin.lua:1816`：

```1816:1816:scripts/prefabskin.lua
    inst.components.inventoryitem:ChangeImageName("dug_gravestone" .. (tostring(inst.random_stone_choice) == "1" and "" or inst.random_stone_choice))
```

挖出来的墓碑有 4 种外观，根据 random_stone_choice 字段切换图标名（dug_gravestone / dug_gravestone2 / dug_gravestone3 / dug_gravestone4）。

#### 第五步：完整调用链

当玩家把物品捡起来放进库存格的时候：

```
inventoryitem 组件
    │ atlasname / imagename 字段
    ▼
ConvertNetVar (Lua → C++)
    │
    ▼
Replica 同步到客户端
    │
    ▼
inventoryitem_replica:SetAtlas/SetImage
    │
    ▼
库存格 widget 绘制时
    │
    ▼
读 atlas + image，从 .tex 取像素
    │
    ▼
显示在屏幕上
```

如果 atlasname 是 nil，绘制时会**回退**到 `GetInventoryItemAtlas(imagename)` 自动查找——所以**两个字段你设一个就够**：

| 设置组合 | 行为 |
|---------|------|
| 只设 imagename | 引擎调 `GetInventoryItemAtlas` 自动查找 atlas（推荐）|
| 同时设 atlasname + imagename | 直接用指定的 atlas（最快、最确定）|
| 只设 atlasname 不设 imagename | 错误——找不到具体哪张图 |

> **进阶记忆**：inventoryitem 的两个字段 atlasname / imagename 是 netvar 同步的；`ChangeImageName(name)` 是动态切换图标的标准方法（同时推 `imagechange` 事件让 UI 刷新）。设 atlasname 是显式查找、不设是自动查找。

---

### 13.6.6 进阶：自定义 atlas 的 .xml 内部结构 + 工具链

#### 第一步：完整 .xml 多图集格式

如果你的 atlas 包含多张小图——比如 4 个不同角度的物品图标——`.xml` 长这样：

```xml
<Atlas>
    <Texture filename="myitem_atlas.tex" />
    <Elements>
        <Element name="myitem_front.tex" u1="0.0"  u2="0.5"  v1="0.5" v2="1.0" />
        <Element name="myitem_back.tex"  u1="0.5"  u2="1.0"  v1="0.5" v2="1.0" />
        <Element name="myitem_side.tex"  u1="0.0"  u2="0.5"  v1="0.0" v2="0.5" />
        <Element name="myitem_open.tex"  u1="0.5"  u2="1.0"  v1="0.0" v2="0.5" />
    </Elements>
</Atlas>
```

UV 坐标含义（约定 0,0 在左下、1,1 在右上，但是某些工具相反）：

```
v=1  ┌──────┬──────┐
     │front │back  │
v=0.5├──────┼──────┤
     │side  │open  │
v=0  └──────┴──────┘
     u=0   u=0.5   u=1
```

**铁律**：所有 4 张小图必须**同尺寸**——atlas 是规则网格切割（实际上理论上可以不规则，但是工具链一般生成规则的）。

#### 第二步：工具链推荐

| 工具 | 流程 |
|------|------|
| **autocompiler.exe**（推荐）| 把多张 PNG 放在 mod 的 `exported/myitem_atlas/` 目录 → 自动打包成 atlas |
| **TEXcreator** | GUI 工具，单张转换 |
| **ktools (krane / ktech)** | 命令行批处理 |
| **手写 .xml + 用 ktech 单转** | 老手用 |

**autocompiler 的散装 atlas 流程**（不是骨骼动画的那个流程）：

```
exported/
└── myitem_atlas/
    ├── myitem_atlas.png        ← 整张大图（512x512，4 区域）
    └── myitem_atlas.xml        ← 索引（要么手写、要么用 atlas 拼接工具生成）
```

跑 autocompiler 后：

```
images/
├── myitem_atlas.tex
└── myitem_atlas.xml
```

#### 第三步：散装 IMAGE（单图）的最简方法

如果只有一张图（不需要打包）：

```
exported/
└── myicon/
    └── myicon.png    ← 直接 PNG
```

autocompiler 会自动转成 `images/myicon.tex` + `images/myicon.xml`（一个图占满整张）。

#### 第四步：在代码中访问

```lua
local Image = require "widgets/image"

local img = Image("images/myitem_atlas.xml", "myitem_front.tex")
img:SetSize(64, 64)
img:SetPosition(0, 0)
```

`Image` widget 接受两个参数——atlas 路径 + atlas 内的图名（**带 .tex 后缀**）。这是**所有 UI 组件**显示散装 atlas 图的标准方式（库存格、按钮、tooltip……）。

#### 第五步：UV 计算的常见陷阱

UV 系统中 V 轴的方向**和 PNG 像素 Y 轴相反**——这就是 13.3.7 讲过的 V 翻转：

```
PNG 像素坐标系                UV 纹理坐标系
y=0 ┌───────┐                  v=1 ┌───────┐
    │ TOP   │                      │ TOP   │
    │       │                      │       │
y=H └───────┘                  v=0 └───────┘
```

如果你**手写 .xml**，`v1` 应该是图像顶部、`v2` 是底部，但是 v1 < v2（值上升对应像素下降）。这一点很多新手会写反——结果图标显示成上下翻转。

**最佳实践**：让工具链生成 .xml，**不要手写 UV**。

> **进阶记忆**：`.xml` 的 `<Element>` 用 UV 矩形 (u1,v1,u2,v2) 切割大图。**自动用 autocompiler 打包**，**用 `Image(atlas, name.tex)` 在 UI 中显示**。手写 UV 容易翻转——尽量用工具。

---

### 13.6.7 老手：scrapbook icon、皮肤变体图标、动态切换

#### 第一步：scrapbook（图鉴）系统的独立 atlas

```722:730:scripts/simutil.lua
----------------------------------------------------------------------------------------------

local scrapbookIconAtlasLookup = {}

function RegisterScrapbookIconAtlas(atlas, imagename)
	if atlas ~= nil and imagename ~= nil then
		if scrapbookIconAtlasLookup[imagename] ~= nil then
			if scrapbookIconAtlasLookup[imagename] ~= atlas then
```

scrapbook（图鉴）系统**不是用 inventoryimages**——它有自己独立的 `scrapbookIconAtlasLookup` 表。注册 API 是 `RegisterScrapbookIconAtlas`，结构和 inventoryitem 类一致。

为什么独立？因为：

- 库存图标是**小图**（64×64），强调即时识别
- 图鉴图标是**大图**（256×256+），强调艺术表现
- 一个 mod 可能给同一个物品提供两个版本——库存格里显示简化图、图鉴页显示精美插画

#### 第二步：minimap atlas（13.7 详讲）

```697:720:scripts/simutil.lua
function GetMinimapAtlas_Internal(imagename)
    local images1 = "minimap/minimap_data1.xml"
    local images2 = "minimap/minimap_data2.xml"
    return TheSim:AtlasContains(images1, imagename) and images1
            or TheSim:AtlasContains(images2, imagename) and images2
            or nil
end
```

地图 POI 标记也走类似的 2 文件查找——`minimap/minimap_data1.xml` 和 `minimap_data2.xml`。13.7 章节会详讲。

#### 第三步：mod 系统的多 atlas 注册

一个大 mod 可能有几十张图标——按主题分散到多个 atlas 是好习惯：

```lua
-- 武器图标
RegisterInventoryItemAtlas("images/myweapons.xml", "myaxe.tex")
RegisterInventoryItemAtlas("images/myweapons.xml", "mybow.tex")

-- 食物图标
RegisterInventoryItemAtlas("images/myfoods.xml", "myapple.tex")
RegisterInventoryItemAtlas("images/myfoods.xml", "mybread.tex")

-- 建筑图标（独立大 atlas，因为建筑图通常更大）
RegisterInventoryItemAtlas("images/mybuildings.xml", "mywall.tex")
```

**好处**：

1. 玩家只装备武器时，只加载 `myweapons.tex` 到显存
2. 修改一个 atlas 不影响其他 atlas（增量编译友好）
3. 调试时可以单独检查某个 atlas

#### 第四步：动态切换 + skinner 集成

`prefabskin.lua` 里有大量 `inventoryitem:ChangeImageName(skin_name)` 调用——**皮肤切换时自动改图标**。一段经典模式：

```lua
local function init_fn(inst, build_name)
    inst.AnimState:SetSkin(build_name, "default_build")
    
    if inst.components.inventoryitem ~= nil then
        inst.components.inventoryitem:ChangeImageName(inst:GetSkinName())
    end
end

local function clear_fn(inst)
    inst.AnimState:SetBuild("default_build")
    
    if inst.components.inventoryitem ~= nil then
        inst.components.inventoryitem:ChangeImageName()  -- nil 参数 = 清空覆盖
    end
end
```

读懂：穿皮肤时一起改 build（场景上的形象）+ imagename（库存格的图标）；脱皮肤时两者都恢复默认。

#### 第五步：inventoryItemAtlasLookup 的可见性

如果你想调试 mod 是否成功注册了图标——在控制台：

```lua
-- 不直接公开，但是通过函数能间接知道
print(GetInventoryItemAtlas("myitem.tex"))
-- 应该返回 "images/myicons.xml" 而不是 nil 或 inventoryimages4.xml
```

如果返回 `inventoryimages4.xml`（fallback）—— 说明你的 atlas 没注册成功，引擎走到了"找不到时的兜底"。

> **老手记忆**：scrapbook、inventoryimages、minimap 各有独立的注册函数和查找表。**多 atlas 分主题** 让大 mod 更易维护。皮肤切换时**配套调** SetBuild + ChangeImageName。

---

### 13.6.8 老手：八个最容易踩的坑

#### 坑 1：图标显示成紫色棋盘格

**症状**：库存格里出现一张紫色棋盘格图——这是 GPU 找不到纹理时的"missing texture"占位。

**原因**：4 选 1：

1. `Asset("IMAGE", ...)` 漏声明——.tex 没被加载
2. `Asset("ATLAS", ...)` 漏声明——.xml 没被加载
3. atlasname/imagename 字段拼写错（**大小写必须完全一致**）
4. `RegisterInventoryItemAtlas` 没注册

**调试**：

```lua
print(GetInventoryItemAtlas("myitem.tex"))  -- 看返回是不是正确的 atlas
```

#### 坑 2：imagename 带 .tex 后缀

```lua
inst.components.inventoryitem.imagename = "myitem.tex"  -- 错！
```

**症状**：图标不显示。

**原因**：imagename 字段**不带后缀**。引擎内部会自动加 `.tex`。

**修复**：

```lua
inst.components.inventoryitem.imagename = "myitem"  -- 对！
```

**对比**：`RegisterInventoryItemAtlas` 的第二参数**要带 .tex**。两个 API 用法不一致——背熟。

#### 坑 3：忘了加 atlasname 但是 mod 没注册到 inventoryimages

```lua
inst.components.inventoryitem.imagename = "myitem"
-- atlasname 没设
-- 也没 RegisterInventoryItemAtlas
```

**症状**：图标显示成 inventoryimages4.xml 里的某张错误图（fallback 机制把 myitem 错误匹配到了官方 atlas 里）；或显示成紫色棋盘。

**原因**：13.6.3 的 fallback 机制——找不到时回退到 inventoryimages4.xml，可能误命中。

**修复**：始终 `RegisterInventoryItemAtlas` 或显式设 atlasname。

#### 坑 4：UV 上下翻转

手写的 .xml：

```xml
<Element name="myitem.tex" u1="0" u2="1" v1="1" v2="0" />
```

**症状**：图标显示成上下翻转。

**原因**：Klei 约定 v1 < v2 表示从顶部到底部——你写反了。

**修复**：v1=0 表示顶部 / v2=1 表示底部，**不是**直觉中的"v=0 在底"。

#### 坑 5：atlas 中两个图叫同一个名字

```xml
<Element name="myitem.tex" u1="0.0" u2="0.5" v1="0.0" v2="0.5" />
<Element name="myitem.tex" u1="0.5" u2="1.0" v1="0.0" v2="0.5" />  ← 名字重复！
```

**症状**：图标显示总是同一张（第一个或最后一个），不管你怎么切换 imagename。

**修复**：name 必须唯一。

#### 坑 6：散装 atlas 在 dedicated server 加载失败

`Asset("IMAGE", "images/myicon.tex")` 在 dedicated server 上**会被解析**——服务器找不到这个文件。

**修复**：在 modmain 里判断：

```lua
if not TheNet:IsDedicated() then
    Assets = {
        Asset("ATLAS", "images/myicons.xml"),
        Asset("IMAGE", "images/myicons.tex"),
    }
end
```

或者在 prefab 文件里**条件声明**——但是这要求 prefab 在客户端 / 服务端的执行路径不同，复杂。

**最简单的折衷**：把图标资源**都打包**——dedicated server 上多浪费几百 KB 内存，但是部署简单。

#### 坑 7：ChangeImageName(nil) 不恢复默认

```lua
inst.components.inventoryitem:ChangeImageName()  -- 期望恢复默认
```

**症状**：图标变成 nil 报错或显示棋盘。

**原因**：`ChangeImageName(nil)` 真的把 imagename 设为 nil——之后引擎拿不到图名。

**修复**：必须传一个有效的图名。如果要"恢复默认"，传**默认 imagename**（通常是 `inst.prefab`）：

```lua
inst.components.inventoryitem:ChangeImageName(inst.prefab)
```

#### 坑 8：跨 mod 资源共享冲突

```lua
-- MOD A
RegisterInventoryItemAtlas("mods/mod_a/images/icons.xml", "myitem.tex")

-- MOD B（同样有 myitem 物品）
RegisterInventoryItemAtlas("mods/mod_b/images/icons.xml", "myitem.tex")
```

**症状**：后注册的覆盖前注册的——两个 mod 的同名物品图标互相干扰。

**修复**：**永远给图名加 mod 前缀**：

```lua
RegisterInventoryItemAtlas("mods/mod_a/images/icons.xml", "moda_myitem.tex")
RegisterInventoryItemAtlas("mods/mod_b/images/icons.xml", "modb_myitem.tex")
```

> **老手记忆**：8 个图标坑里——3 个是**配置错误**（坑 1、3、6）、3 个是**API 误用**（坑 2、4、7）、2 个是**约定违反**（坑 5、8）。养成"图名带 mod 前缀 + 工具生成 .xml + 显式注册 atlas"的习惯。

---

### 13.6 小结：关于 Atlas 与 Inventoryimages 你必须记住的

```
.png ──[autocompiler / TEXcreator]──→ .tex + .xml
                                           │
                                           ▼
                          Asset("ATLAS", ...) + Asset("IMAGE", ...)
                                           │
                          ┌────────────────┴────────────────┐
                          │                                 │
                  方案一：显式 atlasname            方案二：注册 atlas
                          │                                 │
                          ▼                                 ▼
              inst.components.inventoryitem.    RegisterInventoryItemAtlas(
                  atlasname = "..."                  "atlas路径", "name.tex")
                  imagename = "..."                       │
                          │                               │
                          └────────────┬──────────────────┘
                                       │
                                       ▼
                          UI widget 调 GetInventoryItemAtlas
                                       │
                                       ▼
                          从 .xml 找到 UV，从 .tex 取像素
                                       │
                                       ▼
                                显示在库存格里
```

**新手核心三句**：散装 atlas = `.tex + .xml` 一对（不是 zip）；用 `Asset("ATLAS",...)` + `Asset("IMAGE",...)` 一对声明、少一个就显示棋盘；inventoryitem 的 `atlasname/imagename` 控制库存格图标。

**进阶核心三句**：`RegisterInventoryItemAtlas("路径", "图名.tex")` 注册到查找表（**imagename 带 .tex 后缀**）；`ChangeImageName(name)` 是动态切换标准方法（**name 不带后缀**）；4 个 inventoryimages1~4.xml 是 Klei 内置的官方汇总 atlas + fallback。

**老手核心三句**：scrapbook / inventoryimages / minimap 各有独立查找表；皮肤系统通过 `init_fn/clear_fn` 配套调 `SetSkin + ChangeImageName`；八个常见坑里"图名+后缀"和"UV 翻转"占多数——**用工具生成 .xml + 图名加 mod 前缀**能避开 90% 问题。

下一节 13.7 我们就来讲**MiniMap 图标**——它和 inventoryimages 几乎是一个套路（独立 .xml + 自动查找），但是有自己的注册系统（`AddMinimapAtlas`）和特殊的"全局唯一性"语义。理解了 13.6 之后再读 13.7 几乎是无缝过渡。

---


## 13.7 MiniMap 图标——在小地图上显示自定义标记

### 本节导读

13.6 我们讲了**库存图标和散装 atlas**——那是给 HUD 和库存槽用的图。本节聊一种**性质很相似但用途完全不同**的图：**小地图（minimap）图标**。

打开饥荒按 `Tab`，会看到右下角小地图——上面密密麻麻一堆标记：绿色的树、灰色的石头、红色的猪窝、玩家头像、传送门……每一个图标都是 **MiniMap 图标系统**的输出。

它和 inventoryimages 系统**几乎一模一样**：

| 维度 | inventoryimages（13.6） | MiniMap（13.7） |
|------|----------------------|----------------|
| 资源声明 | `Asset("ATLAS", ...)` + `Asset("IMAGE", ...)` | 同上 |
| Klei 内置 atlas | `images/inventoryimages1~4.xml` | `minimap/minimap_data1~2.xml` |
| 自动查找函数 | `GetInventoryItemAtlas` | `GetMinimapAtlas` |
| mod 注册 API | `RegisterInventoryItemAtlas` | `AddMinimapAtlas`（更高级——直接挂到 MiniMap 实体） |
| 显示 API | `inventoryitem:ChangeImageName` | `MiniMapEntity:SetIcon` |
| 是否依赖 entity | 否（纯库存数据）| **是**——必须有 `MiniMapEntity` 组件 |

但是 MiniMap 也有 inventoryimages **没有**的特殊功能：**SetPriority**（图标层级）、**SetDrawOverFogOfWar**（穿透迷雾）、**SetRestriction**（玩家可见性限制）、**SetCanUseCache**（关闭缓存）、**Global Map Icon**（即使离屏也显示）。

> **新手**从 13.7.1-13.7.3 起步——用 5 行代码给一棵树加图标、看懂 `MiniMapEntity:SetIcon` 的工作原理、记住 mod 注册图标的标准三件套（IMAGE+ATLAS+AddMinimapAtlas）；**进阶读者**继续看 13.7.4-13.7.6，深入 `GetMinimapAtlas` 的查找算法、`MiniMap` 实体的 atlas 注册流程（`scripts/prefabs/minimap.lua` 第 42-48 行）、`MiniMapEntity` 的 5 大 API（SetIcon / SetPriority / SetDrawOverFogOfWar / SetRestriction / SetCanUseCache）；**老手**跳到 13.7.7-13.7.8，逐行拆解 `globalmapicon.lua` 这个 200+ 行的"全局地图图标"代理实体、`maprevealable` 组件、`RegisterGlobalMapIcon` 全局表，以及八个最容易踩的"图标显示不出来 / 显示位置错 / 离屏看不见"的坑。

读完本节，你能给任何 prefab 加上一个**正确显示、正确分层、正确权限控制**的小地图图标——并且明白为什么有些 mod 的图标"远处看得见近处反而消失"。

---

### 13.7.1 快速入门：5 行代码给一棵树加图标

来看一个最干净的例子。Klei 的常绿树 (`scripts/prefabs/evergreens.lua`) 第 819-841 行：

```819:841:scripts/prefabs/evergreens.lua
        inst.entity:AddMiniMapEntity()
        inst.entity:AddNetwork()

        MakeObstaclePhysics(inst, .25)

		inst:SetDeploySmartRadius(DEPLOYSPACING_RADIUS[DEPLOYSPACING.DEFAULT] / 2) --seed/planted_tree deployspacing/2

        if build == "twiggy" then

            inst:AddTag("renewable")

            inst.MiniMapEntity:SetIcon("twiggy.png")
        else
            --petrifiable (from petrifiable component) added to pristine state for optimization
            inst:AddTag("petrifiable")

            inst:AddTag("evergreens")
            inst.MiniMapEntity:SetIcon(build == "sparse" and "evergreen_lumpy.png" or "evergreen.png")

            inst:AddTag("shelter")
        end

        inst.MiniMapEntity:SetPriority(-1)
```

整个流程拆开看：

1. **`inst.entity:AddMiniMapEntity()`** —— 给实体附加一个 C++ 端的 MiniMapEntity 组件。**没这一行，后面所有 SetIcon 都没用**
2. **`inst.MiniMapEntity:SetIcon("twiggy.png")`** —— 设定这个实体在小地图上显示的图标名。**注意三点**：
   - 名字带 `.png` 后缀（和 inventoryimages 不同！）
   - 名字**不带路径**——引擎自己去查找哪个 atlas 里有这张图
   - 这张图必须**已经在某个已注册的 minimap atlas 中**（13.7.3 详讲）
3. **`inst.MiniMapEntity:SetPriority(-1)`** —— 设定图标的渲染优先级。负数表示"低优先级"——当玩家附近图标太多时，**优先级低的会被剔除**避免界面拥挤

#### 第二步：勾对应的资源声明

光在代码里调 `SetIcon` 还不够——还要在 prefab 文件最上方的 `assets` 数组里**声明这个图标存在**：

```1:6:scripts/prefabs/phonograph.lua
local assets =
{
    Asset("ANIM", "anim/phonograph.zip"),
    Asset("INV_IMAGE", "phonograph"),
    Asset("MINIMAP_IMAGE", "phonograph"),
}
```

`Asset("MINIMAP_IMAGE", "phonograph")` 告诉引擎"我会用到一张叫 `phonograph` 的小地图图标"。这一步在 13.1.4 表格里讲过——**这个 Asset 类型不走文件路径解析**（`mainfunctions.lua:73-74` 的特判），它只是个**声明性标记**——告诉资源系统"这个图标会被用到，请确保某个 atlas 里有它"。

#### 第三步：图标到底从哪来？

`SetIcon("evergreen.png")` 时，引擎查询路径：

```
"evergreen.png"
    ↓
GetMinimapAtlas("evergreen.png")（scripts/simutil.lua:707）
    ↓
查官方 atlas: minimap/minimap_data1.xml → 有？返回路径
                                       ↓ 没有
查官方 atlas: minimap/minimap_data2.xml → 有？返回路径
                                       ↓ 没有
查所有 mod 通过 AddMinimapAtlas 注册的 atlas
```

**Klei 已经把所有官方实体的图标都打包到 `minimap_data1.xml` / `minimap_data2.xml` 里了**——所以你给 `evergreen` 这种官方实体加图标无需声明 atlas。但是**自己 mod 的图标必须自己提供 atlas**——见 13.7.3。

> **新手记忆**：**3 步给实体加图标**——① `AddMiniMapEntity` 加组件；② `SetIcon("name.png")` 设图（**带 .png 后缀**）；③ `Asset("MINIMAP_IMAGE", "name")` 声明（**不带后缀**）。Klei 内置图标直接用，自定义图标走 13.7.3 的注册流程。

---

### 13.7.2 快速入门：MiniMapEntity 的 5 个核心 API

C++ 端的 `MiniMapEntity` 暴露给 Lua 的接口主要是这 5 个：

| API | 用途 | 例子 |
|-----|------|------|
| `SetIcon(name)` | 设置图标文件名 | `inst.MiniMapEntity:SetIcon("evergreen.png")` |
| `SetPriority(p)` | 设置渲染优先级 | `inst.MiniMapEntity:SetPriority(-1)` |
| `SetDrawOverFogOfWar(over [, seeable])` | 是否绘制在迷雾上方 | `inst.MiniMapEntity:SetDrawOverFogOfWar(true)` |
| `SetRestriction(tag)` | 谁能看到这个图标（按 tag 过滤）| `inst.MiniMapEntity:SetRestriction("plantkin")` |
| `SetCanUseCache(can)` | 是否启用图标缓存 | `inst.MiniMapEntity:SetCanUseCache(false)` |

#### API 1：`SetIcon` —— 设置图标

如前所述。**几乎所有 prefab 都用它**——见 evergreens.lua、phonograph.lua、daywalker_pillar.lua 等等。

#### API 2：`SetPriority` —— 渲染优先级

```841:841:scripts/prefabs/evergreens.lua
        inst.MiniMapEntity:SetPriority(-1)
```

```679:680:scripts/prefabs/daywalker_pillar.lua
	inst.MiniMapEntity:SetIcon("daywalker_pillar.png")
	inst.MiniMapEntity:SetPriority(4)
```

**含义**：当多个图标重叠在小地图上的同一个像素附近时，引擎会**优先显示高优先级的**。

**Klei 约定的优先级规范**（建议）：

| 优先级 | 用途 |
|--------|------|
| `-2` ~ `-1` | 资源（树、石头、浆果丛——多到可能挤满）|
| `0` | 默认，普通建筑物 |
| `1` ~ `3` | 重要建筑（炼金机、冰箱、传送门）|
| `4` ~ `9` | 非常重要的目标（boss 巢穴、远古遗物）|
| `10`+ | 玩家自己的标记（指南针、地图传送）|

#### API 3：`SetDrawOverFogOfWar` —— 穿透迷雾

```58:58:scripts/prefabs/globalmapicon.lua
    inst.MiniMapEntity:SetDrawOverFogOfWar(true)
```

**含义**：默认情况下，地图上**未探索**的区域被"迷雾"遮盖，里面的实体即使有 MiniMapEntity 也**看不见**。`SetDrawOverFogOfWar(true)` 让图标**强制显示在迷雾之上**——比如：

- **玩家头像**——即使队友没探索过的地方也能看到玩家位置
- **指南针指向的目标**
- **某些 boss 警告标记**

第二参数 `seeable`（`SetDrawOverFogOfWar(true, true)`）——见 globalmapicon.lua:103——更高级：图标在迷雾上方，**且**该位置还会被标记为"已探索"。这个用得很少。

#### API 4：`SetRestriction` —— 可见性限制

```14:14:scripts/prefabs/globalmapicon.lua
        inst.MiniMapEntity:SetRestriction(restriction)
```

参数是一个 tag 字符串。**只有带这个 tag 的玩家**才能在小地图上看到此图标——比如薇克巴顿专用的 `plantkin` tag——只有薇克巴顿能看到农作物的特殊小地图标记。

#### API 5：`SetCanUseCache` —— 是否缓存

```42:42:scripts/prefabs/globalmapicon.lua
    inst.MiniMapEntity:SetCanUseCache(false)
```

**含义**：默认引擎会**缓存**小地图图标——一旦实体出现过，即使后来移走了或销毁了，**之前的位置还是会显示一段时间**（这是为了在迷雾系统里"留痕"）。`SetCanUseCache(false)` 关掉缓存——图标实时跟随实体的位置。

**典型场景**：

- 静态建筑（房子、树）—— 默认缓存（移除后还能看见痕迹）
- 移动实体（玩家代理、追踪器）—— 关闭缓存（必须实时跟随）
- `globalmapicon` 这种"代理实体"—— **必须**关闭缓存（否则旧位置会留下幽灵）

> **新手记忆**：5 个 API——SetIcon 设图、SetPriority 排层、SetDrawOverFogOfWar 穿迷雾、SetRestriction 限玩家、SetCanUseCache 控缓存。**90% 的 prefab 只用前两个**。

---

### 13.7.3 快速入门：mod 注册自定义图标的标准流程

这是 mod 开发最常做的事——给自己的 prefab 一个独特图标。来看神话未加密 mod 的标准做法（13.1.6 已经引用过）：

```355:359:mods/联机版mod/神话未加密/modmain.lua
for k,v in pairs(mk_map_icons) do
	table.insert(Assets, Asset( "IMAGE", "images/map_icons/"..v..".tex" ))
    table.insert(Assets, Asset( "ATLAS", "images/map_icons/"..v..".xml" ))
    AddMinimapAtlas("images/map_icons/"..v..".xml")
end
```

**三件套**：

1. **`Asset("IMAGE", "images/map_icons/myicon.tex")`** —— 声明这个 .tex 文件存在，会被资源系统验证
2. **`Asset("ATLAS", "images/map_icons/myicon.xml")`** —— 声明这个 .xml 文件存在
3. **`AddMinimapAtlas("images/map_icons/myicon.xml")`** —— 把这个 atlas 注册到 MiniMap 实体的 atlas 列表

#### 第一步：准备图标文件

通常每个 mod 自定义图标对应**一对** `.xml + .tex`：

```
mods/MyMod/images/map_icons/
├── myhouse.tex      ← 32×32 的图标（推荐尺寸）
├── myhouse.xml      ← atlas 索引文件
├── myaltar.tex
├── myaltar.xml
└── ...
```

每张图标通常很小（16×16 或 32×32 都可以），**单图独享一个 atlas 也是常见做法**——这样图标更新时只需重新生成一个 .tex/.xml，不影响其他图标。

`myhouse.xml` 的内容（最小化）：

```xml
<Atlas>
    <Texture filename="myhouse.tex" />
    <Elements>
        <Element name="myhouse.tex" u1="0.0" u2="1.0" v1="0.0" v2="1.0" />
    </Elements>
</Atlas>
```

#### 第二步：modmain.lua 里声明 + 注册

```lua
Assets = {
    Asset("IMAGE", "images/map_icons/myhouse.tex"),
    Asset("ATLAS", "images/map_icons/myhouse.xml"),
}

AddMinimapAtlas("images/map_icons/myhouse.xml")
```

`AddMinimapAtlas` 是 modutil.lua 第 512-515 行定义的 mod 专用函数：

```512:515:scripts/modutil.lua
	env.AddMinimapAtlas = function( atlaspath )
		initprint("AddMinimapAtlas", atlaspath)
		table.insert(env.postinitdata.MinimapAtlases, atlaspath)
	end
```

它把你的 atlas 路径**追加**到 `MinimapAtlases` 列表里，引擎在初始化 MiniMap 实体时会**遍历**这个列表，把每个 atlas 都"挂载"上去（详见 13.7.4 的 minimap.lua）。

#### 第三步：prefab 中使用

```lua
local function fn()
    local inst = CreateEntity()
    inst.entity:AddTransform()
    inst.entity:AddMiniMapEntity()
    inst.entity:AddNetwork()

    inst.MiniMapEntity:SetIcon("myhouse.tex")  -- ← 注意！

    -- ...
end
```

**关键**——这里 SetIcon 的参数是 `"myhouse.tex"`（**带 .tex 后缀**），不是 `"myhouse.png"`！这是因为 mod 自定义 atlas 的 Element name 是 `myhouse.tex`（参见上面 xml 的 `<Element name="myhouse.tex" .../>`）。**Klei 官方 atlas 用的是 .png 后缀**（编译过程中保留的），**mod 自定义图标用的是 .tex 后缀**——这是 13.7.8 坑 1 里要重点讲的。

> **新手记忆**：mod 加图标三件套——`Asset("IMAGE", ...)` + `Asset("ATLAS", ...)` + `AddMinimapAtlas(...)`。**Klei 内置图标 SetIcon 用 .png 后缀，mod 自定义用 .tex 后缀**。

---

### 13.7.4 进阶：`GetMinimapAtlas` 查找算法 + minimap.lua 注册流程

理解 mod 自定义图标"如何被查找到"的内部机制。

#### 第一步：`GetMinimapAtlas` 函数

`scripts/simutil.lua:707-720`：

```706:720:scripts/simutil.lua
local minimapAtlasLookup = {}
function GetMinimapAtlas(imagename)
	local atlas = minimapAtlasLookup[imagename]
	if atlas then
		return atlas
	end

    atlas = GetMinimapAtlas_Internal(imagename)

	if atlas ~= nil then
		minimapAtlasLookup[imagename] = atlas
	end

	return atlas
end
```

```698:704:scripts/simutil.lua
function GetMinimapAtlas_Internal(imagename)
    local images1 = "minimap/minimap_data1.xml"
    local images2 = "minimap/minimap_data2.xml"
    return TheSim:AtlasContains(images1, imagename) and images1
            or TheSim:AtlasContains(images2, imagename) and images2
            or nil
end
```

**和 `GetInventoryItemAtlas` 几乎一样的模式**：

1. 检查内存 lookup 表，命中直接返回
2. 没命中——调内部函数遍历 Klei 内置 atlas（这里只有 2 个，没有 inventoryimages 的 4 个那么多）
3. 命中后**缓存**到 lookup 表

**注意**：这个函数**只查 Klei 官方 atlas**，**不查 mod atlas**！那 mod 自定义图标是怎么显示的？答案在下一步——`minimap.lua` 的注册流程。

#### 第二步：`scripts/prefabs/minimap.lua` 完整流程

```1:48:scripts/prefabs/minimap.lua
local shader_filename = "shaders/minimap.ksh"
local fs_shader = "shaders/minimapfs.ksh"

local GroundTiles = require("worldtiledefs")

local assets =
{
    Asset("DYNAMIC_ATLAS", "minimap/minimap_data.xml"), -- Legacy for mods.
    Asset("PKGREF", "minimap/minimap_atlas.tex"), -- Legacy for mods.
    Asset("ATLAS", "minimap/minimap_data1.xml"),
    Asset("IMAGE", "minimap/minimap_atlas1.tex"),
    Asset("ATLAS", "minimap/minimap_data2.xml"),
    Asset("IMAGE", "minimap/minimap_atlas2.tex"),

    Asset("ATLAS", "images/hud.xml"),
    Asset("IMAGE", "images/hud.tex"),

    Asset("ATLAS", "images/hud2.xml"),
    Asset("IMAGE", "images/hud2.tex"),

    Asset("SHADER", shader_filename),
    Asset("SHADER", fs_shader),

    Asset("IMAGE", "images/minimap_paper.tex"),
}

for k, v in pairs(GroundTiles.minimapassets) do
    table.insert(assets, v)
end

local function fn()
    local inst = CreateEntity()
    inst.entity:AddUITransform()
    inst.entity:AddMiniMap() --c side renderer
    -- ... 略 ...

    inst.MiniMap:AddAtlas(resolvefilepath("minimap/minimap_data1.xml"))
    inst.MiniMap:AddAtlas(resolvefilepath("minimap/minimap_data2.xml"))
    for _, atlases in ipairs(ModManager:GetPostInitData("MinimapAtlases")) do
        for _, path in ipairs(atlases) do
            inst.MiniMap:AddAtlas(resolvefilepath(path))
        end
    end
    -- ... 略 ...
end
```

**关键 4 步**：

1. **`inst.entity:AddMiniMap()`** —— 创建 C++ 端的 MiniMap 渲染器
2. **`inst.MiniMap:AddAtlas("minimap/minimap_data1.xml")`** —— 把 Klei 官方 atlas-1 挂载到 MiniMap 实体
3. **`inst.MiniMap:AddAtlas("minimap/minimap_data2.xml")`** —— 同上 atlas-2
4. **遍历 `ModManager:GetPostInitData("MinimapAtlases")`** —— 把所有 mod 通过 `AddMinimapAtlas` 注册的 atlas **逐个**挂载

**关键**：mod 的 atlas 是直接挂到 **MiniMap 渲染器实体**上的，**不是**通过 `GetMinimapAtlas` 查的！这就是为什么 `GetMinimapAtlas` 函数里没看到 mod 的逻辑——mod 的查找完全在 C++ 端 `MiniMap:AddAtlas` 内部完成。

#### 第三步：渲染时的查找流程

C++ 端的 MiniMapEntity 在每帧渲染图标时：

```
对每个有 MiniMapEntity 的 entity：
    iconname = entity.MiniMapEntity:GetIcon()  -- 比如 "myhouse.tex"
    
    遍历挂载到 MiniMap 实体的所有 atlas（按挂载顺序）：
        atlas-1 (Klei 内置)
        atlas-2 (Klei 内置)
        mod-A 注册的 atlas
        mod-B 注册的 atlas
        ...
    
    在第一个**包含 iconname 的 atlas** 中找到 UV 坐标
    用该 atlas 的 .tex 渲染图标
```

**所以**——mod 注册的 atlas 是**追加**在 Klei 内置 atlas 后面的——理论上**官方图标优先**。但是因为 Klei 不会跟你的 mod 命名重名（`myhouse.tex` 在 Klei 里查不到），实际上不冲突。

#### 第四步：legacy 注释

注意源码里写的：

```lua
Asset("DYNAMIC_ATLAS", "minimap/minimap_data.xml"), -- Legacy for mods.
Asset("PKGREF", "minimap/minimap_atlas.tex"), -- Legacy for mods.
```

`minimap_data.xml`（**没有数字后缀**）是**老版本 Klei 留下的兼容路径**——以前所有图标都打包到这一个 atlas 里。新版本拆成 `minimap_data1.xml` / `minimap_data2.xml`。Klei 故意保留 legacy 路径，是因为**很多老 mod 还在用它**——直接 `AddMinimapAtlas("minimap/minimap_data.xml")` 来覆盖某个 Klei 图标。

> **进阶记忆**：`GetMinimapAtlas` 只查 Klei 官方 2 个 atlas；mod atlas 通过 `AddMinimapAtlas` → `MiniMap:AddAtlas` 直接挂到 MiniMap 实体上，C++ 端按挂载顺序遍历查找。

---

### 13.7.5 进阶：MiniMapEntity 状态变化的两种模式

实体的小地图图标**不是一成不变**——状态变化时，图标也要切换。来看两种典型模式。

#### 模式 1：直接调用 `SetIcon` 切换

`scripts/prefabs/evergreens.lua` 第 210 行（树被烧成炭）：

```210:210:scripts/prefabs/evergreens.lua
    inst.MiniMapEntity:SetIcon(inst.build == "twiggy" and "twiggy_burnt.png" or "evergreen_burnt.png")
```

第 473 行（树被砍掉变残桩）：

```473:473:scripts/prefabs/evergreens.lua
    inst.MiniMapEntity:SetIcon(inst.build == "twiggy" and "twiggy_stump.png" or "evergreen_stump.png")
```

第 966 行（残桩重新长出）：

```966:966:scripts/prefabs/evergreens.lua
            inst.MiniMapEntity:SetIcon(build == "twiggy" and "twiggy_stump.png" or "evergreen_stump.png")
```

**核心**：在状态切换的回调（`OnBurnt`、`OnChopped`）里**直接重新 SetIcon**——这是最直接、最干净的方式。

```lua
local function OnBurnt(inst)
    inst.MiniMapEntity:SetIcon("evergreen_burnt.png")
    -- ... 其他烧掉后的处理
end
```

#### 模式 2：通过容器/状态机驱动

某些"皮肤系统"或"复杂状态"的实体——直接调 SetIcon 不够——需要在每次刷新时统一计算图标。`scripts/prefabskin.lua:1763`：

```lua
inst.MiniMapEntity:SetIcon(build_name .. ".png")
```

这是**皮肤系统**的统一入口——把当前皮肤的 build 名作为图标名，比如玩家皮肤切换时，地图上的标记也跟着变。

#### 模式 3：`CopyIcon` —— 复制其他实体的图标

`scripts/prefabs/globalmapicon.lua:19`：

```19:19:scripts/prefabs/globalmapicon.lua
        inst.MiniMapEntity:CopyIcon(target.MiniMapEntity)
```

**用途**——globalmapicon 是个**代理实体**——它本身没有"自己的图标"，是为了**追踪另一个实体**而存在的。`CopyIcon(target.MiniMapEntity)` 直接**克隆** target 的图标设置（包括 icon name + priority + restriction）。

#### 模式 4：基于事件的图标更新

```140:148:scripts/prefabs/globalmapicon.lua
local function gclass_RefreshIcon(inst)
	if inst.iconnear then
		local icon = (inst.selected and inst.icondata.selectedicon or inst.icondata.icon)..".png"
		local priority = inst.selected and inst.icondata.selectedpriority or inst.icondata.priority or 0
		inst.iconnear.MiniMapEntity:SetIcon(icon)
		inst.iconfar.MiniMapEntity:SetIcon(icon)
		inst.iconnear.MiniMapEntity:SetPriority(priority)
		inst.iconfar.MiniMapEntity:SetPriority(priority)
	end
end

local function gclass_OnMapSelected(inst)
	inst.selected = true
	gclass_RefreshIcon(inst)
end
```

**模式**：图标有"选中态"和"未选中态"两套。监听 `mapselected` / `cancelmaptarget` 事件，触发 `RefreshIcon`——根据 `selected` 状态切换图标和优先级。

> **进阶记忆**：图标切换 4 种模式——直接 SetIcon、皮肤系统统一驱动、CopyIcon 代理、事件触发 RefreshIcon。

---

### 13.7.6 进阶：Asset("MINIMAP_IMAGE", ...) 的特殊行为

13.1.6 我们提过 `MINIMAP_IMAGE` 类型有特殊处理。这里再深入一点。

**`scripts/mainfunctions.lua:73-75`**：

```73:75:scripts/mainfunctions.lua
    if assettype == "MINIMAP_IMAGE" then
        return true
    end
```

`ShouldIgnoreResolve` 直接返回 true —— 意味着 `MINIMAP_IMAGE` 类型**不走 resolvefilepath**，引擎不会尝试找这个文件。

**那它到底有什么用？**

实际上 `Asset("MINIMAP_IMAGE", "phonograph")` 这个声明是**给 mod 系统看的**：

1. 让 mod 工具链知道这个 mod 用了 `phonograph` 这个图标——便于 mod 上传/下载/资源校验
2. 让代码层面有一个"明示意图"——读 prefab 文件的时候人能看出"这个 prefab 会显示一个叫 phonograph 的小地图图标"
3. **不会触发文件存在检查**——因为图标文件本来就在 Klei 内置 atlas 里，不存在独立的 `phonograph.tex` 文件

**对比 INV_IMAGE 的处理也一样**：

```70:75:scripts/mainfunctions.lua
function ShouldIgnoreResolve( filename, assettype )
    if assettype == "INV_IMAGE" then
        return true
    end
    if assettype == "MINIMAP_IMAGE" then
        return true
    end
```

两者都被跳过——因为它们指代的是"已存在于某个 atlas 里的图"，不是独立文件。

#### 特殊场景：mod 自定义图标用了 IMAGE+ATLAS

mod 自己的图标用的是 `Asset("IMAGE", ...)` + `Asset("ATLAS", ...)`，**不是** `Asset("MINIMAP_IMAGE", ...)`。原因——mod 自定义图标对应的是**真实的 .tex / .xml 文件**——它们必须走 resolvefilepath 走完整路径解析。

#### 老的 globalicon 模式：MINIMAP_IMAGE

```272:275:scripts/prefabs/globalmapicon.lua
	local assets =
	{
		Asset("MINIMAP_IMAGE", icondata.icon),
	}
```

这里用 `MINIMAP_IMAGE` 是因为这些是**Klei 内置图标**——`globalmapicon` 这一类 prefab 用 Klei 已经打包好的图标（"globalmapicon"、"globalmapiconnamed" 等图都在 minimap_data1/2.xml 里）。

> **进阶记忆**：`MINIMAP_IMAGE` 是**声明性**Asset，不走文件解析；mod 自定义图标用 `IMAGE+ATLAS+AddMinimapAtlas` 三件套；用了 `MINIMAP_IMAGE` 的图标必须**已经存在**于某个已注册 atlas 中。

---

### 13.7.7 老手：`globalmapicon.lua` —— 全局地图图标的代理实体模式

普通 `MiniMapEntity` 有个限制——**只在屏幕附近的实体才显示图标**（13.1.7 sleep 机制）。但是某些场景需要**地图任意位置都能看到的标记**——比如玩家头像、传送门追踪、boss 警告。这就是"全局地图图标（GlobalMapIcon）"。

`scripts/prefabs/globalmapicon.lua` 实现了 4 种全局图标 prefab + 1 个工厂函数 `MakeGlobalTrackingIcons`。

#### 第一步：什么是"代理实体"

直接给 boss 加 `MiniMapEntity` 的问题：boss 离玩家很远时（比如另一片大陆），boss 实体会被 EntitySleep 优化掉——它的 MiniMapEntity 不会更新，地图上看不到。

**解决方案**：在 boss 这个**真实实体**附近**生成一个新的、不会 sleep 的"代理 entity"**——这个代理只有 Transform + MiniMapEntity，每帧更新自己的位置跟随 boss。

#### 第二步：代理实体的核心代码

```32:48:scripts/prefabs/globalmapicon.lua
local function common_fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddMiniMapEntity()
    inst.entity:AddNetwork()

    inst:AddTag("globalmapicon")
    inst:AddTag("CLASSIFIED")

    inst.MiniMapEntity:SetCanUseCache(false)
    inst.MiniMapEntity:SetIsProxy(true)

    inst.entity:SetCanSleep(false)

    return inst
end
```

**关键设置**：

| 设置 | 作用 |
|------|------|
| `SetCanUseCache(false)` | 关闭缓存——必须实时跟随 |
| `SetIsProxy(true)` | 标记为"代理实体"——告诉 C++ MiniMap 系统这不是真实实体 |
| `SetCanSleep(false)` | **核心**——禁止 sleep——保证全图都能 tick |
| `AddTag("CLASSIFIED")` | 网络分类——只有特定玩家能看到（用于权限控制）|

#### 第三步：跟踪目标实体

```1:30:scripts/prefabs/globalmapicon.lua
local function UpdatePosition(inst)
	local x, y, z = inst._target.Transform:GetWorldPosition()
    if inst._x ~= x or inst._z ~= z then
        inst._x = x
        inst._z = z
        inst.Transform:SetPosition(x, 0, z)
    end
end

local function TrackEntity(inst, target, restriction, icon, noupdate)
    -- TODO(JBK): This function is not able to be ran twice without causing issues.
    inst._target = target
    if restriction ~= nil then
        inst.MiniMapEntity:SetRestriction(restriction)
    end
    if icon ~= nil then
        inst.MiniMapEntity:SetIcon(icon)
    elseif target.MiniMapEntity ~= nil then
        inst.MiniMapEntity:CopyIcon(target.MiniMapEntity)
    else
        inst.MiniMapEntity:SetIcon(target.prefab..".png")
    end
    inst:ListenForEvent("onremove", function() inst:Remove() end, target)

	if not noupdate then
		inst:AddComponent("updatelooper")
		inst.components.updatelooper:AddOnUpdateFn(UpdatePosition)
	end
    UpdatePosition(inst, target)
end
```

**核心机制**：

1. `_target` 字段保存被追踪的实体
2. `UpdatePosition` 每帧把代理位置同步到 target 位置（**只在位置变化时更新**——优化）
3. 用 `updatelooper` 组件实现持续更新
4. 监听 target 的 `onremove` 事件——target 销毁时**自动删除代理**（避免野指针）

#### 第四步：全局表 `RegisterGlobalMapIcon`

```871:883:scripts/simutil.lua
function RegisterGlobalMapIcon(inst, name)
    if GlobalMapIconsDB.insts[inst] ~= nil then
        print("RegisterGlobalMapIcon called for a second time for inst", inst)
        print(_TRACEBACK())
        return
    end
	name = name or inst.prefab
	inst._GlobalMapIconsDB_Name = name ~= inst.prefab and name or nil
    GlobalMapIconsDB.insts[inst] = true
	GlobalMapIconsDB.prefabs[name] = GlobalMapIconsDB.prefabs[name] or {}
	GlobalMapIconsDB.prefabs[name][inst] = true
    inst:ListenForEvent("onremove", UnregisterGlobalMapIcon)
end
```

**作用**：维护一个**全局查找表** `GlobalMapIconsDB`——可以快速：

```885:914:scripts/simutil.lua
function FindClosestMapIconInRangeSq(name, x, y, z, rangesq, restricted_doer)
	local mapent
	local ents_bin = GlobalMapIconsDB.prefabs[name]
	if ents_bin then
		local ismastersim = TheWorld.ismastersim
		for ent in pairs(ents_bin) do
			local isrestricted
			if restricted_doer then
				if ent.MiniMapEntity then
					--old style global icons use MiniMapEntity:SetRestriction(...)
					if not ent.MiniMapEntity:EntityHasRestriction(restricted_doer.GUID) then
						isrestricted = true
					end
				elseif ismastersim and ient.owner ~= restricted_doer then
					--see global tracking icons (host needs to validate this way, clients don't because the icon should be classified.)
					isrestricted = true
				end
			end
			if not isrestricted then
				local x1, _, z1 = ent.Transform:GetWorldPosition()
				local dsq = math2d.DistSq(x, z, x1, z1)
				if dsq < rangesq then
					rangesq = dsq
					mapent = ent
				end
			end
		end
	end
	return mapent
end
```

**用途**：游戏内"指南针指向最近 boss"、"罗盘搜索最近传送门"等功能，都靠 `FindClosestMapIcon` 在全局表里搜——**不需要遍历所有实体**。

#### 第五步：4 种全局图标 prefab

```402:405:scripts/prefabs/globalmapicon.lua
return Prefab("globalmapicon", overfog_fn),
    Prefab("globalmapiconnamed", overfog_named_fn),
    Prefab("globalmapiconunderfog", underfog_fn),
    Prefab("globalmapiconseeable", overfog_seeable_fn)
```

| Prefab | 特点 |
|--------|------|
| `globalmapicon` | 普通全局图标，迷雾上方可见 |
| `globalmapiconnamed` | 带名称（玩家鼠标悬停时显示文字）|
| `globalmapiconunderfog` | 迷雾下方——**未探索区域看不见**（适合"需要先发现"的目标）|
| `globalmapiconseeable` | 迷雾上方 + 自动揭开迷雾（罕用）|

#### 第六步：使用方式

`maprevealable` 组件是常见的入口：

```222:227:scripts/prefabs/wx78_shadowdrone_debuffer.lua
	inst.components.maprevealable:SetIconPrefab("globalmapiconunderfog")
```

`MapRevealable:StartRevealing` 内部会 SpawnPrefab 这个全局图标，并 TrackEntity 到自己：

```133:148:scripts/components/maprevealable.lua
function MapRevealable:StartRevealing(restriction)
    if self.icon == nil then
        self.icon = SpawnPrefab(self.iconprefab)
        if self.icontag ~= nil then
            self.icon:AddTag(self.icontag)
        end
        if self.iconpriority ~= nil then
            self.icon.MiniMapEntity:SetPriority(self.iconpriority)
        end
        if self.oniconcreatedfn ~= nil then -- Keep before TrackEntity but after anything else used to setup the prefab.
            self.oniconcreatedfn(self.inst, self.icon)
        end
        self.icon:TrackEntity(self.inst, restriction, self.iconname)
    else
        self.icon.MiniMapEntity:SetRestriction(restriction or "")
    end
end
```

> **老手记忆**：全局地图图标 = **代理实体（不会 sleep）+ 跟踪 target 位置 + 注册到 GlobalMapIconsDB**。`maprevealable` 组件是标准入口；`globalmapiconunderfog` / `globalmapiconseeable` 等 4 种 prefab 覆盖不同场景。

---

### 13.7.8 老手：八个最容易踩的坑

#### 坑 1：`.png` 还是 `.tex` 后缀？

**这是 mod 开发者最容易混淆的点**。

| 场景 | 后缀 |
|------|------|
| Klei 内置图标 SetIcon | **`.png`**——例如 `"evergreen.png"` |
| mod 自定义图标 SetIcon | **`.tex`**——例如 `"myhouse.tex"` |
| `Asset("MINIMAP_IMAGE", ...)` | **不带后缀**——例如 `"phonograph"` |
| `Asset("IMAGE", ...)` | **`.tex`**——例如 `"images/map_icons/myhouse.tex"` |

**为什么不一致**？因为 Klei 编译 `minimap_data1.xml` 的时候，xml 里的 Element name 都保留了 `.png` 后缀（虽然实际打包的是 .tex）。但是 mod 自己写 xml 时，习惯写 `.tex`（因为对应的就是 tex 文件）。

**调试技巧**：用文本编辑器打开你 mod 的 xml，看 `<Element name="..." />` 的具体后缀，**SetIcon 必须**和 xml 里一致。

#### 坑 2：忘了 `AddMiniMapEntity()`

```lua
local inst = CreateEntity()
inst.entity:AddTransform()
inst.entity:AddNetwork()

inst.MiniMapEntity:SetIcon("myhouse.tex")  -- ❌ inst.MiniMapEntity 是 nil！
```

必须**先** `AddMiniMapEntity()`，再 `SetIcon`。**报错形式**：`attempt to index field 'MiniMapEntity' (a nil value)`。

#### 坑 3：忘了 `AddMinimapAtlas`

只声明 `Asset("ATLAS", "images/map_icons/myhouse.xml")` 是不够的——要**额外**调 `AddMinimapAtlas("images/map_icons/myhouse.xml")` 才能挂载到 MiniMap 实体。**症状**：图标在 atlas 里、文件存在、SetIcon 调用了——但是地图上**完全看不见**。

#### 坑 4：图标显示"灰色叉叉"或"空白方框"

**原因**：

1. SetIcon 名字写错（拼写、大小写、后缀）
2. Atlas 没注册（坑 3）
3. 图标对应的 `<Element>` 不存在于该 atlas 中
4. .tex 文件损坏或维度不对

**调试**：控制台 `print(TheSim:AtlasContains("minimap/minimap_data1.xml", "evergreen.png"))` —— 检查这张图是否在该 atlas 中。

#### 坑 5：图标缓存 + 实体移动 = 残影

如果实体会**移动**（比如猪人、boss），并且**没设** `SetCanUseCache(false)`——**地图上会留下旧位置的"残影图标"**。

**修复**：

```lua
inst.MiniMapEntity:SetCanUseCache(false)
```

#### 坑 6：dedicated server 没有 MiniMap 实体

**dedicated server**（专用服务器）上**没有 MiniMap 渲染器**——`AddMinimapAtlas` 注册的 atlas 不会真的被使用。**注意**：`Asset("IMAGE", ...)` 仍然会被服务器尝试加载（因为没在 ShouldIgnoreResolve 里特判），可能导致服务器启动失败。

**解决**：

```lua
if not TheNet:IsDedicated() then
    AddMinimapAtlas("images/map_icons/myhouse.xml")
end
```

或者用 `Asset("MINIMAP_IMAGE", ...)` 代替——它**不走文件解析**，server 上不会出错。

#### 坑 7：图标位置漂移（追踪代理 entity 位置不准）

**症状**：用 globalmapicon 追踪 boss——boss 移动时图标**有延迟**或者**完全不动**。

**原因**：

1. `updatelooper` 没正确添加（看 `TrackEntity` 函数的 `noupdate` 参数）
2. target 实体本身被 sleep 了——它的 Transform 位置不更新
3. 网络延迟——客户端的代理拿到的位置数据滞后

**修复**：检查 `inst._target.Transform:GetWorldPosition()` 是否实时变化。如果 target 是网络实体，可能需要 `inst.entity:SetCanSleep(false)` 给 target 加上。

#### 坑 8：SetRestriction 没生效

**症状**：SetRestriction 之后所有玩家**仍然能看到**图标。

**原因 1**：写错了 tag 名——SetRestriction 只是字符串匹配。

**原因 2**：玩家**没有这个 tag**——比如 `SetRestriction("plantkin")` 但是测试玩家不是薇克巴顿。

**原因 3**：网络不同步——服务端设了 restriction，但客户端的 MiniMapEntity 状态还是旧的。**对网络实体**：在主机端调 SetRestriction 后立刻 SetDirty，让客户端更新。

**调试**：

```lua
print(inst.MiniMapEntity:EntityHasRestriction(ThePlayer.GUID))
-- 返回 true 表示玩家有这个 tag、可以看到
```

---

### 13.7 小结：关于 MiniMap 图标你必须记住的

```
            实体 inst
                │
                ▼
        AddMiniMapEntity()         ← C++ 端 MiniMapEntity 组件
                │
                ▼
        ┌───────────────────┐
        │ SetIcon(name)     │      ← 图标名（Klei 用 .png，mod 用 .tex）
        │ SetPriority(p)    │      ← 渲染优先级
        │ SetDrawOverFog(b) │      ← 穿透迷雾
        │ SetRestriction(t) │      ← tag 限制玩家可见性
        │ SetCanUseCache(b) │      ← 是否缓存（移动实体应 false）
        └───────────────────┘
                │
                ▼
        渲染时按 atlas 挂载顺序查找：
        ① minimap/minimap_data1.xml   ← Klei 内置
        ② minimap/minimap_data2.xml   ← Klei 内置
        ③ mod-A AddMinimapAtlas       ← mod 自定义
        ④ mod-B AddMinimapAtlas       ← mod 自定义
        ...
                │
                ▼
        找到 → 显示图标
        没找到 → 灰色叉叉 / 不显示

如果实体离屏远 → 用 globalmapicon 代理实体（SetCanSleep=false + SetIsProxy=true）
```

**新手核心三句**：3 步加图标——`AddMiniMapEntity` + `SetIcon("name.png")` + `Asset("MINIMAP_IMAGE", "name")`；自定义图标三件套——`Asset("IMAGE",...)` + `Asset("ATLAS",...)` + `AddMinimapAtlas(...)`；**Klei 用 .png 后缀，mod 用 .tex 后缀**。

**进阶核心三句**：5 大 API——SetIcon / SetPriority / SetDrawOverFogOfWar / SetRestriction / SetCanUseCache；`GetMinimapAtlas` 只查 Klei 官方 2 个 atlas，mod atlas 通过 `MiniMap:AddAtlas` 直接挂载到 MiniMap 实体；`MINIMAP_IMAGE` Asset 是声明性标记，**不**走 resolvefilepath。

**老手核心三句**：全局图标 = 代理实体（SetCanSleep=false）+ TrackEntity 跟踪 + 注册到 GlobalMapIconsDB；`maprevealable` 组件是标准接入点；八个常见坑里"后缀错"和"忘加 AddMinimapAtlas"占七成——**用 MINIMAP_IMAGE 代替 IMAGE+ATLAS** 在引用 Klei 内置图标时更稳。

下一节 13.8 是本章的**实战集大成**——我们将从零开始为一个**自定义物品**做一套完整的美术资源——包括 Spriter build / anim、地面图标、库存图标、小地图图标，并把所有 13.1-13.7 的知识串起来落地。


## 13.8 实战：为自定义物品制作完整动画

### 本节导读

13.1-13.7 我们逐个拆解了**资源管线**（13.1）、**Spriter 工具**（13.2）、**编译流程**（13.3）、**AnimState API**（13.4）、**符号覆盖换装**（13.5）、**地面/库存图标制作**（13.6）、**小地图图标**（13.7）。每一节都很扎实，但它们是**独立部件**——本节是**装配工厂**：从一张白纸开始，做一个**完整的自定义物品**，把全部 7 节的知识串起来落地。

我们的目标物品：**"暴风眼魔晶"（stormeye_crystal）**——

- 一颗会浮在地面的**会动的水晶**——idle 时缓慢转动并发光（**用到 13.2 的 Spriter / 13.3 的编译 / 13.4 的 AnimState API**）
- **库存里的图标**——一颗紫色水晶（**用到 13.6 的 inventoryimages**）
- **小地图上的特殊标记**——掉在地上时显示一个紫色三角形（**用到 13.7 的 MiniMap**）
- **手持时角色挥动它**——挥动时 swap_object 替换成水晶（**用到 13.5 的 OverrideSymbol**）
- **使用时**消耗一颗水晶给玩家加状态（普通物品逻辑）

> **新手**从 13.8.1-13.8.3 起步——目录规划、最小化版本（一个**静态**物品的最小代码 + 资源清单）；**进阶读者**继续看 13.8.4-13.8.6，给物品加上完整动画、库存图标、地图图标、武器换装；**老手**跳到 13.8.7-13.8.8，加入皮肤系统、动态加载（DYNAMIC_ANIM）、网络同步、调试技巧、八个最容易踩的"实战集大成坑"。

读完本节，你能从零开始做出一个**完整的、可用的、有美术的自定义物品**——这就是饥荒 mod 真正的"成人礼"。

---

### 13.8.1 快速入门：项目目录规划

#### 第一步：mod 文件夹结构

一个标准的物品 mod 至少是这样的目录：

```
mods/MyStormCrystal/
├── modinfo.lua                       ← mod 元数据（名字/作者/兼容版本）
├── modmain.lua                       ← mod 入口（加 prefab、声明资源、注册图标）
├── exported/                         ← Spriter 源工程（仅开发期）
│   └── stormeye_crystal/
│       ├── stormeye_crystal.scml
│       ├── crystal-0.png
│       ├── crystal-1.png
│       └── ...
├── anim/                             ← 编译后的动画包
│   └── stormeye_crystal.zip          ← 由 Spriter 编译生成
├── images/
│   ├── inventoryimages/              ← 库存图标
│   │   ├── stormeye_crystal.tex
│   │   └── stormeye_crystal.xml
│   └── map_icons/                    ← 小地图图标
│       ├── stormeye_crystal.tex
│       └── stormeye_crystal.xml
└── scripts/
    └── prefabs/
        └── stormeye_crystal.lua     ← 物品 prefab 文件
```

**关键约定**：

- `exported/` 目录**只在开发期存在**——发布 mod 时可以删除（玩家只需要编译产物）
- `anim/` 目录里的 .zip 是 **Spriter 编译产物**——见 13.3 流程
- `images/inventoryimages/` 和 `images/map_icons/` 分两个子目录是为了**清晰分离 UI 用图和小地图用图**
- `scripts/prefabs/` 是 prefab 文件位置——`modmain.lua` 中通过 `PrefabFiles` 注册

#### 第二步：modinfo.lua 最小化版本

```lua
-- modinfo.lua
name = "Stormeye Crystal"
description = "A custom mystical crystal item."
author = "YourName"
version = "1.0.0"

api_version = 10           -- 联机版固定 10
dst_compatible = true       -- 联机版兼容
dont_starve_compatible = false
all_clients_require_mod = true  -- 服务端 + 所有客户端都要装

icon_atlas = "modicon.xml"
icon = "modicon.tex"
```

`api_version = 10` 是**联机版（Don't Starve Together）的固定值**——见 modutil.lua 处理逻辑。

#### 第三步：modmain.lua 骨架

```lua
-- modmain.lua

-- 注册要加载的 prefab 文件
PrefabFiles = {
    "stormeye_crystal",
}

-- 声明所有美术资源
Assets = {
    -- 动画包
    Asset("ANIM", "anim/stormeye_crystal.zip"),

    -- 库存图标（mod 自定义路径，必须独立 atlas）
    Asset("IMAGE", "images/inventoryimages/stormeye_crystal.tex"),
    Asset("ATLAS", "images/inventoryimages/stormeye_crystal.xml"),

    -- 小地图图标
    Asset("IMAGE", "images/map_icons/stormeye_crystal.tex"),
    Asset("ATLAS", "images/map_icons/stormeye_crystal.xml"),
}

-- 把库存 atlas 注册到查找表（13.6.5 讲过）
AddPrefabPostInit("stormeye_crystal", function(inst)
    -- nothing yet
end)

-- 把小地图 atlas 注册到 MiniMap 实体（13.7.3 讲过）
AddMinimapAtlas("images/map_icons/stormeye_crystal.xml")

-- 让 mod 的库存图标系统认识它（13.6.5 讲过）
GLOBAL.RegisterInventoryItemAtlas(
    "images/inventoryimages/stormeye_crystal.xml",
    "stormeye_crystal.tex"
)

-- 添加配方（让玩家可以制作）
local Recipe = GLOBAL.Recipe2 or GLOBAL.Recipe
local IngredientList = {
    Ingredient("nightmarefuel", 2),
    Ingredient("redgem", 1),
}
AddRecipe2(
    "stormeye_crystal",
    IngredientList,
    GLOBAL.TECH.MAGIC_TWO,
    {
        atlas = "images/inventoryimages/stormeye_crystal.xml",
        image = "stormeye_crystal.tex",
    }
)

-- 添加自定义文本
STRINGS.NAMES.STORMEYE_CRYSTAL = "Stormeye Crystal"
STRINGS.CHARACTERS.GENERIC.DESCRIBE.STORMEYE_CRYSTAL = "It hums with stormy power."
```

> **新手记忆**：mod 目录有标准 5 件套——`modinfo.lua` / `modmain.lua` / `anim/` / `images/` / `scripts/prefabs/`。modmain 里要做四件事：声明 PrefabFiles、声明 Assets、注册 atlas、加配方/文本。

---

### 13.8.2 快速入门：最小化静态物品（不带动画）

实战的第一步——**先不做动画**，做一个**静态**的物品 prefab。这样我们能立刻验证"基础流程通"——再逐步加上动画。

```lua
-- scripts/prefabs/stormeye_crystal.lua
local assets = {
    Asset("ANIM", "anim/stormeye_crystal.zip"),
}

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    -- 用一个**已存在**的 Klei 内置 build 临时占位
    -- 等做好自己的 build 再换
    inst.AnimState:SetBank("redgem")
    inst.AnimState:SetBuild("redgem")
    inst.AnimState:PlayAnimation("idle")

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")
    inst:AddComponent("stackable")
    inst.components.stackable.maxsize = 20

    MakeHauntableLaunch(inst)

    return inst
end

return Prefab("stormeye_crystal", fn, assets)
```

**这个最小版本的特点**：

- 用 Klei 内置的 `redgem` build 临时占位（你能立刻看到一颗红宝石的样子，但 prefab 名字是 stormeye_crystal）
- 完整的 AddXxx 三件套（Transform / AnimState / Network）
- 标准 inventoryitem 组件（让物品能被捡起、放进库存）
- stackable（可堆叠）

#### 验证步骤

1. 启动游戏，进入开发者控制台（按 `~`）
2. 输入：`c_spawn("stormeye_crystal")`
3. 如果地上出现一颗红宝石——**基础流程通了**
4. 走过去 `c_pick()` 把它捡起来——能进库存就说明 inventoryitem 也工作了

**调试**：如果 `c_spawn` 报错 `Could not find prefab stormeye_crystal`——检查 modmain.lua 的 `PrefabFiles` 是否包含它。

> **新手记忆**：先用 Klei 内置 build 跑通最小版本，再换自己的——避免一次性踩太多坑。

---

### 13.8.3 快速入门：第一次替换为自己的 build

假设你已经按 13.2 / 13.3 流程做好了 `anim/stormeye_crystal.zip`——里面有一个名为 `stormeye_crystal` 的 build（**注意**：build 名要和 prefab 名一致是 Klei 约定，**强烈建议遵守**）和一个名为 `idle` 的 anim。

修改 prefab：

```lua
-- scripts/prefabs/stormeye_crystal.lua

-- ...

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    MakeInventoryPhysics(inst)

    -- ← 改成自己的 build
    inst.AnimState:SetBank("stormeye_crystal")
    inst.AnimState:SetBuild("stormeye_crystal")
    inst.AnimState:PlayAnimation("idle", true)  -- true = 循环

    -- ...
end
```

#### 第二步：加上库存图标

虽然 prefab 没显式调 `ChangeImageName`，但是 inventoryitem 组件**默认会用 prefab 名当图标名**——只要你在 modmain.lua 里 `RegisterInventoryItemAtlas` 注册过、并且 atlas 中有 `stormeye_crystal.tex` 这张图，库存槽就会自动显示它。

**测试**：捡起 `stormeye_crystal`，看库存槽——应该出现你的紫色水晶图标。

**如果出现紫红色 missing 标记**：

1. 检查 `images/inventoryimages/stormeye_crystal.xml` 是否存在
2. 检查 xml 里的 `<Element name="stormeye_crystal.tex" .../>` 拼写
3. 控制台 `print(GetInventoryItemAtlas("stormeye_crystal"))` —— 看是否返回正确路径

#### 第三步：加上小地图图标

prefab 的 `fn` 函数里加上：

```lua
local function fn()
    -- ... 前面的代码 ...

    inst.entity:AddMiniMapEntity()
    inst.MiniMapEntity:SetIcon("stormeye_crystal.tex")  -- mod 自定义后缀 .tex
    inst.MiniMapEntity:SetPriority(2)

    MakeInventoryPhysics(inst)
    -- ...
end
```

**测试**：把水晶丢在地上、按 `Tab` 看小地图——应该有个图标。

> **新手记忆**：从最小化版本到完整版本——分三步加（动画 → 库存图标 → 小地图图标）——每加一步立即测试，错了好定位。

---

### 13.8.4 进阶：完整 prefab——掌握 13.4 + 13.6 + 13.7 综合

下面给出**完整版** stormeye_crystal.lua——包含从 idle 动画到 highlight 状态、从 inventoryimages 到 minimap、从 inspectable 到 stackable 的所有功能：

```lua
-- scripts/prefabs/stormeye_crystal.lua

local assets = {
    Asset("ANIM", "anim/stormeye_crystal.zip"),

    -- 显式声明（虽然 modmain 已经声明，prefab 这里再写一次确保被资源系统看到）
    Asset("ATLAS", "images/inventoryimages/stormeye_crystal.xml"),
    Asset("IMAGE", "images/inventoryimages/stormeye_crystal.tex"),
    Asset("ATLAS", "images/map_icons/stormeye_crystal.xml"),
    Asset("IMAGE", "images/map_icons/stormeye_crystal.tex"),
}

local prefabs = {
    "stormeye_crystal_fx",  -- 13.8.5 中要做的特效
}

------------------------------------------------------------
-- AnimState 辅助函数
------------------------------------------------------------

local function PlayIdle(inst)
    inst.AnimState:PlayAnimation("idle", true)  -- 循环
end

local function PlayHighlight(inst)
    -- 玩家鼠标悬停时高亮——增强亮度 + 加点彩色
    inst.AnimState:SetMultColour(1.4, 1.2, 1.6, 1)
end

local function ClearHighlight(inst)
    inst.AnimState:SetMultColour(1, 1, 1, 1)
end

------------------------------------------------------------
-- 组件回调
------------------------------------------------------------

local function OnEquip(inst, owner)
    -- 玩家装备这件物品时——见 13.5 OverrideSymbol
    owner.AnimState:OverrideSymbol(
        "swap_object",
        "stormeye_crystal",  -- 来自我们的 build
        "swap_stormeye_crystal"  -- 这个 Symbol 必须在 build 里存在
    )
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end

local function OnUnequip(inst, owner)
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
    owner.AnimState:ClearOverrideSymbol("swap_object")
end

local function OnDropped(inst)
    -- 落地时——重新播 idle
    PlayIdle(inst)
end

------------------------------------------------------------
-- 主构造函数
------------------------------------------------------------

local function fn()
    local inst = CreateEntity()

    --------------------------------------------------------
    -- 实体基础组件（都是必须的）
    --------------------------------------------------------
    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()
    inst.entity:AddMiniMapEntity()  -- 13.7

    MakeInventoryPhysics(inst)

    --------------------------------------------------------
    -- 动画状态（13.4）
    --------------------------------------------------------
    inst.AnimState:SetBank("stormeye_crystal")
    inst.AnimState:SetBuild("stormeye_crystal")
    inst.AnimState:PlayAnimation("idle", true)
    inst.AnimState:UsePointFiltering(true)  -- 像素艺术风格不糊

    --------------------------------------------------------
    -- 小地图图标（13.7）
    --------------------------------------------------------
    inst.MiniMapEntity:SetIcon("stormeye_crystal.tex")
    inst.MiniMapEntity:SetPriority(2)

    --------------------------------------------------------
    -- 标签（性能优化——客户端能快速过滤）
    --------------------------------------------------------
    inst:AddTag("magical")
    inst:AddTag("crystal")
    inst:AddTag("fx_carrier")

    MakeInventoryFloatable(inst)  -- 让物品能浮在水上

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    --------------------------------------------------------
    -- 服务端组件
    --------------------------------------------------------

    inst:AddComponent("inspectable")
    inst:AddComponent("inventoryitem")
    inst.components.inventoryitem:SetOnDroppedFn(OnDropped)

    inst:AddComponent("stackable")
    inst.components.stackable.maxsize = 20

    inst:AddComponent("equippable")
    inst.components.equippable.equipslot = EQUIPSLOTS.HANDS
    inst.components.equippable:SetOnEquip(OnEquip)
    inst.components.equippable:SetOnUnequip(OnUnequip)

    -- 武器属性
    inst:AddComponent("weapon")
    inst.components.weapon:SetDamage(20)
    inst.components.weapon:SetAttackRange(1.5)

    inst:AddComponent("finiteuses")
    inst.components.finiteuses:SetMaxUses(40)
    inst.components.finiteuses:SetUses(40)
    inst.components.finiteuses:SetOnFinished(function(item)
        item:Remove()
    end)

    inst:AddComponent("waterproofer")
    inst.components.waterproofer:SetEffectiveness(0.5)

    MakeHauntableLaunch(inst)

    return inst
end

return Prefab("stormeye_crystal", fn, assets, prefabs)
```

#### 关键点拆解

**1. AddXxx 顺序**

`AddTransform → AddAnimState → AddNetwork → AddMiniMapEntity` 是**几乎所有可见可移动实体的固定模板**。注意 `AddNetwork` 在 `SetPristine()` 之前——这是网络同步的硬要求。

**2. 客户端 / 服务端代码分离**

`SetPristine()` 之前是**客户端 + 服务端都执行**的代码（C++ 端组件 + 渲染相关）。`if not TheWorld.ismastersim then return inst end` 之后是**仅服务端**——添加逻辑组件（inventoryitem、weapon、health 等）。

**3. swap_object Symbol 必须在 build 里存在**

`OnEquip` 里 `OverrideSymbol("swap_object", "stormeye_crystal", "swap_stormeye_crystal")`——这要求你的 Spriter 项目里**必须有一个名为 `swap_stormeye_crystal` 的 Symbol**。如果做不出来，可以**复用现有 build 的 swap Symbol**（比如 swap nightmare crystal 复用 nightmarefuel 的 swap symbol——参考 horrorfuel.lua）。

> **进阶记忆**：完整 prefab 6 大块——AddXxx 基础 / AnimState / MiniMap / Tags / Pristine 客户端 / Pristine 后服务端。**记住模板**，每次新 prefab 复制粘贴。

---

### 13.8.5 进阶：装备时的"光晕特效" prefab

我们再做一个相关的 fx prefab，让水晶**装备时玩家手上发出紫色光晕**：

```lua
-- scripts/prefabs/stormeye_crystal_fx.lua

local assets = {
    Asset("ANIM", "anim/stormeye_crystal_fx.zip"),  -- 光晕动画
}

local function fn()
    local inst = CreateEntity()

    inst.entity:AddTransform()
    inst.entity:AddAnimState()
    inst.entity:AddNetwork()

    inst:AddTag("FX")
    inst:AddTag("NOCLICK")  -- 不能被点击

    inst.AnimState:SetBank("stormeye_crystal_fx")
    inst.AnimState:SetBuild("stormeye_crystal_fx")
    inst.AnimState:PlayAnimation("idle", true)

    -- 加色——半透明紫色光晕
    inst.AnimState:SetMultColour(0.7, 0.4, 1.0, 0.5)

    -- 加发光（13.4 提到的 SetLightOverride）
    inst.AnimState:SetLightOverride(0.3)

    -- 让光晕"叠加渲染"（看起来更亮）
    inst.AnimState:SetSymbolBloom("crystal_glow")  -- 可选

    -- 监听 anim 结束自动移除（如果不是 looping）
    inst:ListenForEvent("animover", function() inst:Remove() end)

    inst.entity:SetPristine()
    if not TheWorld.ismastersim then
        return inst
    end

    inst.persists = false  -- 不存档

    return inst
end

return Prefab("stormeye_crystal_fx", fn, assets)
```

#### 装备时触发特效

回到主 prefab 的 `OnEquip` 里加：

```lua
local function OnEquip(inst, owner)
    owner.AnimState:OverrideSymbol("swap_object", "stormeye_crystal", "swap_stormeye_crystal")
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")

    -- 在玩家身上 spawn 一个特效，挂载到玩家
    if inst.fxinst == nil then
        inst.fxinst = SpawnPrefab("stormeye_crystal_fx")
        inst.fxinst.entity:SetParent(owner.entity)
        -- 跟着玩家身上 'swap_object' Symbol
        inst.fxinst.entity:AddFollower()
        inst.fxinst.Follower:FollowSymbol(owner.GUID, "swap_object", 0, 0, 0)
    end
end

local function OnUnequip(inst, owner)
    owner.AnimState:Hide("ARM_carry")
    owner.AnimState:Show("ARM_normal")
    owner.AnimState:ClearOverrideSymbol("swap_object")

    -- 移除特效
    if inst.fxinst ~= nil then
        inst.fxinst:Remove()
        inst.fxinst = nil
    end
end
```

**关键 API**——`Follower:FollowSymbol(parent_guid, symbol_name, x, y, z)`：

- 让一个实体**跟随**另一个实体的指定 Symbol
- 当 owner 的 swap_object Symbol 移动时（比如挥剑动作），这个 fxinst 也跟着移动
- x/y/z 是相对偏移
- 这是饥荒里"装备特效"的标准模式

> **进阶记忆**：装备特效用 `SpawnPrefab("xxx_fx") + SetParent(owner.entity) + Follower:FollowSymbol(...)` 三件套；记得在 OnUnequip 里 `:Remove()` 防止内存泄漏。

---

### 13.8.6 进阶：把使用动作做成"挥砍特技"

让水晶不只是个装饰物——**当玩家攻击时**，水晶要发出一道紫色法术弧光，并消耗一次使用次数。

#### 第一步：监听攻击事件

修改 `OnEquip`：

```lua
local function OnAttack(inst, owner, target)
    if target == nil or owner == nil then
        return
    end

    -- 在攻击点 spawn 弧光特效
    local x, y, z = target.Transform:GetWorldPosition()
    local fx = SpawnPrefab("stormeye_crystal_arc_fx")
    if fx then
        fx.Transform:SetPosition(x, y + 1, z)
    end

    -- 给目标加 debuff
    if target.components.health and not target.components.health:IsDead() then
        target.components.combat:GetAttacked(owner, 5)  -- 额外 5 点伤害
    end
end

local function OnEquip(inst, owner)
    -- ... 之前的代码 ...

    -- 监听 owner 的"完成攻击"事件
    inst.attackslistener = function(_, data)
        OnAttack(inst, owner, data.target)
    end
    inst:ListenForEvent("onattackother", inst.attackslistener, owner)
end

local function OnUnequip(inst, owner)
    -- ... 之前的代码 ...

    -- 取消监听
    if inst.attackslistener then
        inst:RemoveEventCallback("onattackother", inst.attackslistener, owner)
        inst.attackslistener = nil
    end
end
```

**事件解释**：

| 事件名 | 何时触发 | 数据 |
|-------|---------|-----|
| `onattackother` | 当 owner 完成一次攻击命中目标 | `{ target = entity, weapon = inst, ... }` |
| `onequip` | owner 装备物品时 | `{ item = inst, eslot = "hands" }` |
| `onunequip` | owner 卸下物品时 | `{ item = inst, eslot = "hands" }` |

#### 第二步：消耗次数

```lua
local function OnAttack(inst, owner, target)
    -- ... 之前的代码 ...

    -- 消耗一次使用次数
    if inst.components.finiteuses then
        inst.components.finiteuses:Use(1)
    end
end
```

#### 第三步：场景特效（参考 horrorfuel.lua）

`horrorfuel.lua` 还展示了一个高级模式——**根据玩家精神值调节物品发光强度**：

```19:37:scripts/prefabs/horrorfuel.lua
local function CalcTargetLightOverride(player)
    if player ~= nil then
        local sanity = player.replica.sanity
        if sanity ~= nil and sanity:IsInsanityMode() then
            local k = sanity:GetPercent()
            if k < 0.6 then
                k = 1 - k / 0.6
                return k * k
            end
        end
    end
    return 0
end

local function UpdateLightOverride(inst, instant)
    inst.targetlight = CalcTargetLightOverride(_player)
    inst.currentlight = instant and inst.targetlight or inst.targetlight * .1 + inst.currentlight * .9
    inst.AnimState:SetLightOverride(inst.currentlight)
end
```

我们可以借鉴这个模式——**当玩家精神值低于 50% 时水晶变得更亮**：

```lua
local function UpdateGlow(inst, instant)
    local player = ThePlayer
    if player and player.replica.sanity then
        local sanity_pct = player.replica.sanity:GetPercent()
        local target = sanity_pct < 0.5 and (1 - sanity_pct) * 1.5 or 0
        local current = instant and target or (inst.currentlight or 0) * 0.9 + target * 0.1
        inst.currentlight = current
        inst.AnimState:SetLightOverride(current)
    end
end

local function OnEntityWake(inst)
    if not inst.task then
        inst.task = inst:DoPeriodicTask(1, UpdateGlow, math.random(), false)
    end
end

local function OnEntitySleep(inst)
    if inst.task then
        inst.task:Cancel()
        inst.task = nil
    end
end

local function fn()
    -- ... 之前的代码 ...

    if not TheNet:IsDedicated() then
        inst.OnEntityWake = OnEntityWake
        inst.OnEntitySleep = OnEntitySleep
    end

    -- ...
end
```

**关键**——`OnEntityWake` / `OnEntitySleep` 在 12.8 讲过——离屏时不需要更新发光，省 CPU。

> **进阶记忆**：物品的"挥动效果" = onequip 时 ListenForEvent + onunequip 时 RemoveEventCallback；"发光效果" = OnEntityWake 时 DoPeriodicTask + OnEntitySleep 时 task:Cancel；这两套模式覆盖 90% 的"特殊物品"需求。

---

### 13.8.7 老手：皮肤系统接入与 DYNAMIC_ANIM 的应用

到这里你的水晶已经是"完整自定义物品"——但是还有两个**进阶 mod 玩家会问的问题**：

1. 怎么让玩家在游戏内**给水晶换皮肤**（比如蓝色版、绿色版、金色版）
2. 资源加载太多导致游戏启动慢——怎么**按需加载**

#### 问题 1：皮肤系统

Klei 的皮肤系统通过 **`ATLAS_BUILD`** 类型 + **`OverrideItemSkinSymbol`** API 实现。来看 nightstick.lua 的标准模式：

```22:31:scripts/prefabs/nightstick.lua
local function onequip(inst, owner)
    inst.components.burnable:Ignite()

    local skin_build = inst:GetSkinBuild()
    if skin_build ~= nil then
        owner:PushEvent("equipskinneditem", inst:GetSkinName())
        owner.AnimState:OverrideItemSkinSymbol("swap_object", skin_build, "swap_nightstick", inst.GUID, "swap_nightstick")
    else
        owner.AnimState:OverrideSymbol("swap_object", "swap_nightstick", "swap_nightstick")
    end
```

**关键**：

- `inst:GetSkinBuild()` —— 如果有皮肤，返回皮肤 build 名（比如 `nightstick_lava`）
- `OverrideItemSkinSymbol(symbol, skin_build, default_symbol, item_guid, fallback_symbol)` —— 比 OverrideSymbol 多了 GUID 跟踪——支持**穿戴动画的精确同步**

我们的 stormeye_crystal 接入皮肤——只需把 `OnEquip` 改成同样的 if/else 模式：

```lua
local function OnEquip(inst, owner)
    local skin_build = inst:GetSkinBuild()
    if skin_build then
        owner:PushEvent("equipskinneditem", inst:GetSkinName())
        owner.AnimState:OverrideItemSkinSymbol(
            "swap_object",
            skin_build,
            "swap_stormeye_crystal",
            inst.GUID,
            "swap_stormeye_crystal"
        )
    else
        owner.AnimState:OverrideSymbol(
            "swap_object",
            "stormeye_crystal",
            "swap_stormeye_crystal"
        )
    end

    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
end
```

但是定义皮肤需要更多步骤（`prefabskins.lua` 的 PREFAB_SKINS 表 + 每个皮肤一个 `_init.lua`）——超出本节范围。**模式记住即可**。

#### 问题 2：DYNAMIC_ANIM 按需加载

如果你的 mod 有**几十种水晶皮肤**，每个皮肤都是独立的 zip 包——游戏启动时**全部加载**会消耗大量内存。解决方案——**DYNAMIC_ANIM**。

参考神话未加密 mod 的做法（13.1.6 中已经引用）：

```374:377:mods/联机版mod/神话未加密/modmain.lua
for _,v in ipairs(mk_skin_assets) do
	table.insert(Assets, Asset("DYNAMIC_ANIM", "anim/dynamic/"..v..".zip"))
    table.insert(Assets, Asset("PKGREF", "anim/dynamic/"..v..".dyn"))
end
```

然后在 prefab 里，**只在需要时**加载：

```lua
local function OnEquip(inst, owner)
    local skin_build = inst:GetSkinBuild() or "stormeye_crystal"

    -- 在装备时才把这个 build 加载到内存
    owner.AnimState:AddOverrideBuild(skin_build)
    owner.AnimState:OverrideSymbol("swap_object", skin_build, "swap_stormeye_crystal")

    -- ...
end

local function OnUnequip(inst, owner)
    local skin_build = inst:GetSkinBuild() or "stormeye_crystal"

    owner.AnimState:ClearOverrideSymbol("swap_object")
    owner.AnimState:RemoveOverrideBuild(skin_build)  -- 卸下时释放内存

    -- ...
end
```

**关键 API**：

- `AnimState:AddOverrideBuild(buildname)` —— 显式加载一个 build 到内存
- `AnimState:RemoveOverrideBuild(buildname)` —— 释放它

这种"按需加载"模式让 mod 启动快、内存占用低。

> **老手记忆**：皮肤接入 = `OverrideItemSkinSymbol`；DYNAMIC_ANIM 按需加载 = `AddOverrideBuild` / `RemoveOverrideBuild`；两者配合让 mod 有大量皮肤却**不影响性能**。

---

### 13.8.8 老手：八个最容易踩的"实战集大成坑"

这一节集中本章所有内容相关的"集大成"问题。每一个都是真实 mod 论坛的高频帖子。

#### 坑 1：build 名和 prefab 名不一致

**新手最常见**：

```lua
inst.AnimState:SetBuild("stormeye_crystal_v2")  -- 但是 zip 里的 build 叫 stormeye_crystal
```

**症状**：模型不显示、控制台报 `failed to load build stormeye_crystal_v2`。

**铁律**：**Spriter 里 entity 名 = 代码里 SetBuild 字符串 = 通常 = prefab 名**。三者保持一致最稳。

#### 坑 2：anim 包内的 anim 名和代码不一致

如果你的 .scml 里 anim 叫 `idle_loop`，但代码里 `PlayAnimation("idle")`——anim 不播。

**调试**：解压 zip 看 `anim.bin`（用 krane.exe 反编译），看里面真实的 anim 名。

#### 坑 3：swap_object Symbol 在 build 里不存在

OnEquip 时 `OverrideSymbol("swap_object", build, "swap_stormeye_crystal")`——但是你的 Spriter 项目里**根本没有** `swap_stormeye_crystal` 这个 Symbol——**装备后玩家手上空空**。

**铁律**：每件武器/工具 build **必须**包含一个 `swap_xxx` Symbol。

#### 坑 4：`Show("ARM_carry")` / `Hide("ARM_normal")` 调反

**症状**：装备后玩家手臂消失。

**正确顺序**——OnEquip：`Show("ARM_carry")` + `Hide("ARM_normal")`；OnUnequip：相反。

引用 nightstick 的标准代码作为参照：

```33:34:scripts/prefabs/nightstick.lua
    owner.AnimState:Show("ARM_carry")
    owner.AnimState:Hide("ARM_normal")
```

#### 坑 5：库存图标显示成紫红色 missing

**原因**——常见有四：

1. `RegisterInventoryItemAtlas` 没调用
2. atlas xml 里的 `<Element name>` 后缀不对
3. `images/inventoryimages/stormeye_crystal.tex` 文件不存在
4. mod 加载顺序——你的 mod 加载早于查找时刻

**调试**：控制台打印：

```lua
print(GetInventoryItemAtlas("stormeye_crystal"))
print(TheSim:AtlasContains("images/inventoryimages/stormeye_crystal.xml", "stormeye_crystal.tex"))
```

#### 坑 6：合成菜单看不到水晶

**两个原因**：

1. `AddRecipe2` 没调用，或者参数错误（图标路径错）
2. tech 等级（`TECH.MAGIC_TWO`）玩家还没解锁——这是**正常的**

**调试**：在控制台 `c_settech("MAGIC", 2)` 立刻解锁魔法二级——再看合成菜单。

#### 坑 7：dedicated server 启动时报缺资源

**症状**：自己测试 OK，但是发布后 dedicated server 启动失败。

**原因**：

- `Asset("ATLAS", ...)` / `Asset("IMAGE", ...)` 在 dedicated server 上**仍然要找文件**——服务端没有这个文件就报错
- mod 工具上传时**漏传**了 images 目录

**修复**：

1. 检查上传到 Steam Workshop 的 mod 文件清单——确保 `images/` 目录被包含
2. 如果某些资源 server 真的不需要——用 `Asset("MINIMAP_IMAGE", ...)` 代替（不走文件解析）

#### 坑 8：动画在客户端跑客户端，不同步

**症状**：服务端 `inst.AnimState:PlayAnimation("attack")`——客户端的玩家**没看到攻击动画**。

**原因**：`PlayAnimation` 本身**不通过网络同步**！它只在调用方（通常是服务端 SimMaster）执行。客户端要看到动画，必须**也调一次**——通常通过：

1. **stategraph**——同时运行在客户端和服务端，自动同步
2. **net 变量 + 监听 dirty 事件**——在客户端 onDirty 时调 PlayAnimation
3. **PushEvent("attack") + componentreplica**——靠组件副本同步

**最常见正确做法**：用 stategraph 而不是直接 PlayAnimation——stategraph 在客户端自动跑同样的状态机。

---

### 13.8 小结：从一张白纸到完整物品

```
       ┌───────────────────────────────────────────────────┐
       │     1. 美术：Spriter 做 build + anim → .scml      │ ← 13.2
       └─────────────────────────┬─────────────────────────┘
                                 │
       ┌─────────────────────────▼─────────────────────────┐
       │     2. 编译：autocompiler.exe → anim/xxx.zip      │ ← 13.3
       └─────────────────────────┬─────────────────────────┘
                                 │
       ┌─────────────────────────▼─────────────────────────┐
       │     3. UI 图标：images/inventoryimages/xxx.tex/xml │ ← 13.6
       │     4. 地图图标：images/map_icons/xxx.tex/xml     │ ← 13.7
       └─────────────────────────┬─────────────────────────┘
                                 │
       ┌─────────────────────────▼─────────────────────────┐
       │     5. modmain.lua：声明 Assets + 注册 atlas       │ ← 13.1
       │     6. PrefabFile：scripts/prefabs/xxx.lua        │
       │        ├── AddTransform / AddAnimState / Network  │ ← 模板
       │        ├── SetBank / SetBuild / PlayAnimation     │ ← 13.4
       │        ├── MiniMapEntity:SetIcon                  │ ← 13.7
       │        ├── inventoryitem / stackable              │ ← 模板
       │        ├── equippable + OverrideSymbol            │ ← 13.5
       │        └── 业务组件 (weapon/finiteuses/...)        │
       └─────────────────────────┬─────────────────────────┘
                                 │
       ┌─────────────────────────▼─────────────────────────┐
       │     7. AddRecipe2：让玩家可制作                    │
       │     8. STRINGS：本地化文本                         │
       └─────────────────────────┬─────────────────────────┘
                                 │
       ┌─────────────────────────▼─────────────────────────┐
       │     一个完整的、可发布到 Workshop 的物品 mod        │
       └───────────────────────────────────────────────────┘
```

**新手核心三句**：mod 标准 5 件套（modinfo / modmain / anim / images / scripts）；先用 Klei 内置 build 跑通最小化版本，再换自己的；从最小到完整分三步加（动画 → 库存图标 → 小地图图标）——每加一步立即测试。

**进阶核心三句**：完整 prefab 6 大块（AddXxx 基础 / AnimState / MiniMap / Tags / Pristine 客户端 / Pristine 后服务端）；装备特效用 SpawnPrefab + SetParent + Follower:FollowSymbol 三件套；OnEntityWake/Sleep 控制 DoPeriodicTask 是性能优化的关键模式。

**老手核心三句**：皮肤接入用 `OverrideItemSkinSymbol`；DYNAMIC_ANIM 按需加载用 `AddOverrideBuild`/`RemoveOverrideBuild`；八个常见坑里 build/anim 命名错占四成、Symbol 不存在占三成、网络同步问题占两成——**记住模板 + 用 stategraph** 能避开 90% 问题。

---

### 第13章全章总结

至此，**第13章 动画与美术资源**完整结束。回顾一下我们走过的路：

- **13.1** 鸟瞰资源管线 —— `.tex / .xml / build.bin / anim.bin` 四件套 + 12 种 Asset 类型 + RegisterPrefabsImpl 加载流程
- **13.2** Spriter 动画工具 —— Symbol / Frame / Anim 三大概念 + Klei 命名约定 + .scml 内部结构
- **13.3** .scml → .zip 编译流程 —— autocompiler.exe / scml.exe / krane.exe 工具链
- **13.4** AnimState API —— SetBank / SetBuild / PlayAnimation 核心 + PushAnimation / OverrideSymbol / SetMultColour 等 30+ API
- **13.5** OverrideSymbol 换装原理 —— Symbol 的 build 隔离 + skin 系统 + DYNAMIC_ANIM 按需加载
- **13.6** Atlas 与 Inventoryimages —— 散装 atlas 的注册和查找 + RegisterInventoryItemAtlas 全局表
- **13.7** MiniMap 图标 —— MiniMapEntity 5 大 API + AddMinimapAtlas 注册机制 + globalmapicon 全局代理实体
- **13.8** 实战 —— 把 13.1-13.7 全部串起来做一个**完整的自定义物品**


