# 第3章 第一个 Mod——从零开始

## 3.1 modinfo.lua——Mod 的身份证

### 本节导读

`modinfo.lua` 是每个 Mod 的「身份证」，游戏在加载任何 Mod 代码之前，首先读取这个文件来获取 Mod 的名称、版本、兼容性、配置选项等基本信息。没有它，游戏甚至不会知道你的 Mod 存在。

> **新手**可以从「最小可运行范例」开始，快速理解每个字段的含义；**进阶读者**可以深入了解每个字段背后的游戏行为；**老手**则可以直接跳到「深入引擎」部分，看引擎源码到底怎么解析这个文件。

---

### 3.1.1 快速入门：最小可运行的 modinfo.lua

对于新手来说，一个能正常加载的 `modinfo.lua` 只需要以下几行：

```lua
name = "我的第一个 Mod"
description = "这是一个测试 Mod"
author = "你的名字"
version = "1.0"

api_version = 10

dst_compatible = true
dont_starve_compatible = false

all_clients_require_mod = true
client_only_mod = false

icon_atlas = "modicon.xml"
icon = "modicon.tex"
```

这就是一个合格的「身份证」了。将这个文件放入你的 Mod 文件夹根目录，游戏就能在 Mod 列表中找到它。

**为什么需要这些字段？** 因为游戏在加载 Mod 时，会强制检查一组必填字段。如果缺少，Mod 会被标记为「失败」而无法加载。我们来看引擎是怎么做的。

在 `scripts/modindex.lua` 的 `InitializeModInfo` 函数中（第 628 行），引擎定义了一个检查列表：

```lua
local checkinfo = {
    "name", "description", "author", "version", "api_version",
    "dont_starve_compatible", "reign_of_giants_compatible",
    "configuration_options", "dst_compatible"
}
local missing = {}

for i,v in ipairs(checkinfo) do
    if env[v] == nil then
        if v == "dont_starve_compatible" then
            -- 只警告，不阻止加载
        elseif v == "reign_of_giants_compatible" then
            -- 只警告，不阻止加载
        elseif v == "dst_compatible" then
            -- 只警告，不阻止加载
            print("WARNING loading modinfo.lua: "..modname.." does not specify if it is compatible with Don't Starve Together.")
        elseif v == "configuration_options" then
            -- 没有配置选项完全没问题
        else
            table.insert(missing, v)
        end
    end
end

if #missing > 0 then
    local e = "Error loading modinfo.lua. These fields are required: " .. table.concat(missing, ", ")
    print(e)
    env.failed = true  -- Mod 被标记为失败，无法加载
end
```

从这段代码可以得出结论：

- **真正的硬性必填字段**只有 5 个：`name`、`description`、`author`、`version`、`api_version`——缺少任何一个都会直接 `failed = true`，Mod 无法加载。
- `dont_starve_compatible`、`reign_of_giants_compatible`、`dst_compatible` 缺少只会打印警告，不会导致加载失败。
- `configuration_options` 完全可以不写。

> **新手记忆口诀**：名字、描述、作者、版本、API 版本——这五个缺一不可。

---

### 3.1.2 字段全览：每个字段的含义与作用

下面按功能分类，逐一讲解所有可用字段。

#### 一、基本信息（必填）

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | Mod 的显示名称，出现在游戏 Mod 列表中 |
| `description` | string | Mod 的描述文字，支持 `\n` 换行 |
| `author` | string | 作者名称 |
| `version` | string | 版本号，游戏会自动 trim 空白并转为小写 |
| `api_version` | number | Mod API 版本，**联机版必须填 10** |

**关于 `version` 的处理**，引擎会在 `LoadModInfo` 中做标准化：

```lua
-- scripts/modindex.lua 第 528-532 行
info.version = TrimString(info.version or "")
info.version = string.lower(info.version)
info.version_compatible = type(info.version_compatible) == "string" and info.version_compatible or info.version
info.version_compatible = TrimString(info.version_compatible)
info.version_compatible = string.lower(info.version_compatible)
```

这意味着 `"1.0"` 和 `" 1.0 "` 效果一样。`version_compatible` 用于多人联机时的版本匹配——如果你没有专门设置它，引擎会自动用 `version` 的值填充。

**关于 `api_version` 的检查**，引擎会和全局常量 `MOD_API_VERSION`（定义在 `scripts/mods.lua` 第 6 行，值为 10）比较：

```lua
-- scripts/modindex.lua 第 621-626 行
if env.api_version == nil or env.api_version < MOD_API_VERSION then
    -- 低于当前版本：警告 "Old API!" 但仍可加载
    modinfo_message = modinfo_message.."Old API! (mod: "..tostring(env.api_version).." game: "..MOD_API_VERSION..")"
elseif env.api_version > MOD_API_VERSION then
    -- 高于当前版本：直接拒绝，mod 无法加载
    env.failed = true
end
```

结论：
- 小于 10：警告 "Old API" 但仍可加载（可能有兼容性问题）
- 大于 10：直接 `failed = true`，Mod 被拒绝
- 等于 10：正常通过

> **进阶提示**：如果你的 Mod 需要同时兼容单机版和联机版，可以用 `api_version_dst` 字段。引擎会在第 614 行优先使用它覆盖 `api_version`：
>
> ```lua
> -- scripts/modindex.lua 第 613-616 行
> if env.api_version_dst ~= nil then
>     env.api_version = env.api_version_dst
> end
> ```
>
> 这样你就可以写 `api_version = 6`（单机版）和 `api_version_dst = 10`（联机版），一份 modinfo 同时支持两个游戏。像「神话书说」Mod 就是这么做的：
>
> ```lua
> api_version = 6
> api_version_dst = 10
> ```

#### 二、兼容性标志

| 字段 | 类型 | 说明 |
|------|------|------|
| `dst_compatible` | boolean | 是否兼容联机版（Don't Starve Together） |
| `dont_starve_compatible` | boolean | 是否兼容单机版基础版 |
| `reign_of_giants_compatible` | boolean | 是否兼容巨人国 DLC |
| `shipwrecked_compatible` | boolean | 是否兼容船难 DLC |
| `hamlet_compatible` | boolean | 是否兼容哈姆雷特 DLC |
| `forge_compatible` | boolean | 是否兼容熔炉活动 |
| `gorge_compatible` | boolean | 是否兼容暴食活动 |

**对于联机版 Mod，你至少需要**：

```lua
dst_compatible = true
dont_starve_compatible = false
```

如果你根本不设置这些字段，引擎会默认全部设为 `true`，同时设置一个 `*_compatibility_specified = false` 的标记——这会让游戏 UI 显示一条「作者没有标明兼容性」的警告：

```lua
-- scripts/modindex.lua 第 664-676 行
if env.dont_starve_compatible == nil then
    env.dont_starve_compatible = true
    env.dont_starve_compatibility_specified = false  -- UI 会据此显示警告
end
if env.reign_of_giants_compatible == nil then
    env.reign_of_giants_compatible = true
    env.reign_of_giants_compatibility_specified = false
end
if env.dst_compatible == nil then
    env.dst_compatible = true
    env.dst_compatibility_specified = false
end
```

> **为什么要明确写 `false`？** 因为不写等于没有声明，引擎会默认为 `true` 并在 UI 中提示「作者未指定」。这在单机版游戏中可能导致你的联机版 Mod 被错误地显示出来（虽然大概率无法运行），造成玩家困惑。

#### 三、联机网络字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `all_clients_require_mod` | boolean | 所有加入服务器的玩家都必须安装此 Mod |
| `client_only_mod` | boolean | 纯客户端 Mod（如 UI 美化、小地图等），不需要服务器安装 |
| `server_only_mod` | boolean | 纯服务器端 Mod（非官方字段，部分 Mod 自行使用） |

**这是联机版最重要的概念之一。** 一个 Mod 的运行方式取决于这两个标志的组合：

| `client_only_mod` | `all_clients_require_mod` | 行为 | 典型用途 |
|:-:|:-:|------|------|
| `false` | `true` | 服务器 Mod，所有客户端必装 | 角色 Mod、物品 Mod、玩法 Mod |
| `false` | `false` | 服务器 Mod，客户端不需要 | 纯服务器逻辑 Mod |
| `true` | `false` | 纯客户端 Mod，只在本地运行 | UI Mod、信息显示 Mod、小地图 |
| `true` | `true` | **矛盾！** 引擎会打印警告 | 不要这样做 |

引擎对最后一种矛盾情况的处理：

```lua
-- scripts/modindex.lua 第 678-679 行
if env.client_only_mod and env.all_clients_require_mod then
    print("WARNING loading modinfo.lua: "..modname.." specifies client_only_mod and all_clients_require_mod. These flags are mutually exclusive.")
end
```

为什么说这两个字段互斥？因为 `client_only_mod` 意味着「这个 Mod 只在客户端运行，和服务器无关」，而 `all_clients_require_mod` 意味着「服务器要求所有客户端都安装这个 Mod」。一个不需要服务器参与的 Mod，却要求服务器去通知所有客户端安装——这在逻辑上自相矛盾。

在引擎内部，`client_only_mod` 直接决定了 Mod 被归入哪个列表：

```lua
-- scripts/modindex.lua
function ModIndex:GetServerModNames()
    local names = {}
    for modname,_ in pairs(self.savedata.known_mods) do
        if not self:GetModInfo(modname).client_only_mod then
            table.insert(names, modname)  -- 非客户端 Mod → 服务器列表
        end
    end
    return names
end

function ModIndex:GetClientModNames()
    local names = {}
    for modname,_ in pairs(self.savedata.known_mods) do
        if self:GetModInfo(modname).client_only_mod then
            table.insert(names, modname)  -- 客户端 Mod → 客户端列表
        end
    end
    return names
end
```

而在 `mods.lua` 中，`all_clients_require_mod` 决定了服务器是否会在房间信息中通告此 Mod，从而要求所有加入的客户端也安装它：

```lua
-- scripts/mods.lua 第 147-161 行（GetEnabledModsModInfoDetails）
for k,mod_name in pairs(ModManager:GetEnabledServerModNames()) do
    local modinfo = KnownModIndex:GetModInfo(mod_name)
    table.insert(modinfo_details, {
        name = mod_name,
        info_name = modinfo ~= nil and modinfo.name or mod_name,
        version = modinfo ~= nil and modinfo.version or "",
        version_compatible = modinfo ~= nil and modinfo.version_compatible or "",
        all_clients_require_mod = modinfo ~= nil and modinfo.all_clients_require_mod == true,
    })
end
```

这个函数的返回值会被发送给所有尝试加入服务器的客户端。客户端收到后，会检查自己是否安装了这些 `all_clients_require_mod = true` 的 Mod——如果没有，就会自动从创意工坊下载，或者弹出提示。

> **新手建议**：如果你不确定怎么选，大多数情况用 `all_clients_require_mod = true` + `client_only_mod = false` 就对了。只有当你做的是纯 UI/显示类功能时，才考虑用 `client_only_mod = true`。

#### 四、图标

| 字段 | 类型 | 说明 |
|------|------|------|
| `icon_atlas` | string | 图标 atlas 文件的相对路径（如 `"modicon.xml"`） |
| `icon` | string | 图标 tex 文件名（如 `"modicon.tex"`），尺寸 128×128 |

图标文件需要你用 Klei 的工具（如 autocompiler 或第三方工具）将一张 128×128 的图片转换为 `.tex` + `.xml` 格式。

引擎会验证这些路径是否真正存在（`scripts/modindex.lua` 第 536-556 行）：

```lua
if info.icon_atlas ~= nil and info.icon ~= nil and info.icon_atlas ~= "" and info.icon ~= "" then
    local atlaspath = MODS_ROOT..modname.."/"..info.icon_atlas
    local iconpath = string.gsub(atlaspath, "/[^/]*$", "") .. "/"..info.icon
    if softresolvefilepath(atlaspath) and softresolvefilepath(iconpath) then
        info.icon_atlas = atlaspath   -- 转换为绝对路径
        info.iconpath = iconpath
    else
        -- 路径无效：清空图标字段，防止游戏崩溃
        print(string.format("WARNING: icon paths for mod %s are not valid.", ModInfoname(modname)))
        info.icon_atlas = nil
        info.iconpath = nil
        info.icon = nil
    end
else
    info.icon_atlas = nil
    info.iconpath = nil
    info.icon = nil
end
```

注意两点：
1. 路径无效**不会导致 Mod 加载失败**，只是图标不显示——引擎做了优雅降级处理，避免因为一张图片的问题而让整个 Mod 挂掉
2. 引擎会把相对路径转换为绝对路径存入 `info.icon_atlas`，这样游戏其他地方（如 UI 代码）可以直接使用而不用再拼接路径

#### 五、加载顺序与优先级

| 字段 | 类型 | 说明 |
|------|------|------|
| `priority` | number | 数字越大，越早加载。默认为 0 |

**为什么需要 priority？** 如果你的 Mod 是一个「API / 库 Mod」，被其他 Mod 依赖，你需要确保自己先加载。否则其他 Mod 在 `modmain.lua` 中尝试调用你的 API 时，你的代码可能还没执行。

来看引擎的排序逻辑（`scripts/mods.lua` 第 527-559 行）：

```lua
-- 按 priority 排序，让"库"类型的 Mod 先加载
local function sanitizepriority(priority)
    local prioritytype = type(priority)
    if prioritytype == "string" then
        return tonumber(priority) or 0  -- 字符串尝试转数字，失败则为 0
    elseif prioritytype == "number" then
        return priority
    end
    return 0  -- 其他类型一律按 0 处理
end

local function modPrioritySort(a, b)
    if a.modinfo and b.modinfo then
        local apriority = sanitizepriority(a.modinfo.priority)
        local bpriority = sanitizepriority(b.modinfo.priority)
        if apriority == bpriority then
            -- priority 相同时，按名字排序
            return stringidsorter(aname, bname)
        end
        return apriority > bpriority  -- 数字越大越靠前
    end
    return stringidsorter(a.modname, b.modname)
end

table.sort(self.mods, modPrioritySort)
```

排序规则总结：
1. **`priority` 数值越大，越先加载**（`apriority > bpriority`）
2. priority 相同时，按 Mod 名字的字母顺序排序
3. priority 不填默认为 0（通过 `sanitizepriority` 函数处理）
4. 即使你传了字符串类型的 priority，引擎也会尝试用 `tonumber()` 转换

> **实战案例**：
> - **Gem Core**（宝石核心 API）将 priority 设为 `1.79769313486231e+308`（Lua 能表示的最大浮点数），确保它在所有 Mod 之前加载
> - **Modded Skins API** 设为 `2147483647`（32 位整数最大值），目的也是尽早加载
> - **人物模板** 设为 `-99999999999`，表示「我要最后加载」——因为人物 Mod 通常依赖各种 API，需要等它们就位
>
> 一般的 Mod 不需要设置 priority，默认的 0 就够了。只有当你明确知道自己需要在其他 Mod 之前/之后加载时，才需要调整。

#### 六、服务器标签

| 字段 | 类型 | 说明 |
|------|------|------|
| `server_filter_tags` | table | 字符串数组，用于服务器列表的标签筛选 |

当玩家在服务器列表中搜索特定标签时，游戏会收集所有启用 Mod 的 `server_filter_tags`，合并到服务器的标签列表中：

```lua
server_filter_tags = {
    "character",
    "magic",
}
```

引擎中的实现（`scripts/modindex.lua` 第 1281-1294 行）：

```lua
function ModIndex:GetEnabledModTags()
    local tags = {}
    for name,data in pairs(self.savedata.known_mods) do
        if data.enabled then
            local modinfo = self:GetModInfo(name)
            if modinfo ~= nil and modinfo.server_filter_tags ~= nil then
                for i,tag in pairs(modinfo.server_filter_tags) do
                    table.insert(tags, tag)
                end
            end
        end
    end
    return tags
end
```

这些标签最终会出现在服务器列表的标签信息中（由 `scripts/mainfunctions.lua` 调用），帮助玩家找到运行了特定 Mod 的服务器。

> **建议**：为你的 Mod 添加有意义的标签，如 `"character"`、`"item"`、`"magic"`、`"adventure"` 等，方便玩家搜索。

#### 七、配置选项（configuration_options）

这是 `modinfo.lua` 中最复杂也最强大的部分。它让玩家在 Mod 设置界面中调整你的 Mod 行为，而不需要改代码。

##### 基本结构

```lua
configuration_options = {
    {
        name = "difficulty",           -- 选项的唯一标识符（程序中用这个名字取值）
        label = "难度设置",             -- 显示在设置界面的标题
        hover = "调整 Mod 的难度",      -- 鼠标悬停时的提示文字
        options = {                    -- 可选值列表
            { description = "简单", data = 1, hover = "适合新手" },
            { description = "普通", data = 2 },
            { description = "困难", data = 3 },
        },
        default = 2,                   -- 默认值（必须与某个 option 的 data 对应）
    },
}
```

各子字段说明：

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `name` | string | 唯一标识符，在 `modmain.lua` 中通过 `GetModConfigData(name)` 读取 |
| `label` | string | 显示在 Mod 设置界面上的选项标题 |
| `hover` | string | 鼠标悬停在选项标题上时显示的提示文字（可选） |
| `options` | table | 一个数组，每项包含 `description`（显示文字）、`data`（实际值）、`hover`（提示，可选） |
| `default` | any | 默认选中的值，应与某个 option 的 `data` 一致 |

##### 在 modmain.lua 中读取配置

在 `modmain.lua` 中通过以下方式读取玩家的选择：

```lua
local difficulty = GetModConfigData("difficulty")
-- difficulty 的值就是玩家选择的那个 option 的 data 值
```

**`GetModConfigData` 的工作原理**（源自 `scripts/modutil.lua` 第 34-61 行）：

```lua
function GetModConfigData(optionname, modname, get_local_config)
    local config, temp_options = KnownModIndex:GetModConfigurationOptions_Internal(modname, force_local_options)
    if config and type(config) == "table" then
        if temp_options then
            return config[optionname]
        else
            for i,v in pairs(config) do
                if v.name == optionname then
                    if v.saved_server ~= nil and not get_local_config then
                        return v.saved_server    -- 优先级 1：服务器端保存的值
                    elseif v.saved_client ~= nil and get_local_config then
                        return v.saved_client    -- 优先级 2：客户端保存的值
                    elseif v.saved ~= nil then
                        return v.saved           -- 优先级 3：通用保存值
                    else
                        return v.default         -- 优先级 4：modinfo 中的默认值
                    end
                end
            end
        end
    end
    return nil
end
```

读取优先级从高到低：
1. `saved_server`——服务器端保存的配置值
2. `saved_client`——客户端保存的配置值（仅在请求本地配置时）
3. `saved`——通用保存值
4. `default`——modinfo.lua 中定义的默认值

玩家在游戏 UI 中修改配置后，新值会被保存到 `mod_config_data/` 目录下的文件中，下次加载时会覆盖默认值。

##### 进阶技巧：用配置选项做分类标题

很多 Mod 的配置选项很多，为了让设置界面更清晰，可以用一个「假选项」作为分隔线：

```lua
configuration_options = {
    {
        name = "SECTION_COMBAT",
        label = "═══ 战斗设置 ═══",
        options = { { description = "", data = 0 } },
        default = 0,
    },
    {
        name = "weapon_damage",
        label = "武器伤害",
        -- ...正常选项...
    },
    {
        name = "SECTION_UI",
        label = "═══ 界面设置 ═══",
        options = { { description = "", data = 0 } },
        default = 0,
    },
    -- ...更多选项...
}
```

这个「分隔线选项」永远不会被 `GetModConfigData` 读取，纯粹是为了美化设置界面。万物书、勋章等大型 Mod 都使用了这种技巧。

##### 进阶技巧：按键绑定配置

人物模板中展示了一种为快捷键配置生成选项列表的写法：

```lua
local Choices = {"A","B","C","D","E","F","G","H","I","J","K","L","M","N","O","P","Q","R","S","T","U","V","W","X","Y","Z"}

local function keyslist(default)
    local list = {
        { description = default, data = default.."-key" },
    }
    for i = 1, #Choices do
        list[#list + 1] = { description = Choices[i], data = Choices[i].."-key", hover = Choices[i].."键" }
    end
    return list
end

configuration_options = {
    {
        name = "skill_key",
        label = "技能快捷键",
        hover = "选择释放技能的按键",
        options = keyslist("R"),
        default = "R-key",
    },
}
```

记住：modinfo.lua 本质上是 Lua 代码，你可以在其中定义局部变量和函数来简化配置的生成。

#### 八、其他字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `forumthread` | string | 论坛帖子链接，可以留空 |
| `version_compatible` | string | 多人联机版本匹配用，不设则自动等于 `version` |
| `restart_required` | boolean | 设为 `false` 可允许不重启游戏就启用/停用 Mod |
| `standalone` | boolean | 独占模式——设为 `true` 时此 Mod 会阻止其他所有 Mod 加载 |
| `forcemanifest` | boolean | 控制是否强制使用 manifest（创意工坊 Mod 默认开启，本地 Mod 默认关闭） |
| `mod_dependencies` | table | 声明依赖的其他 Mod（仅非 `client_only_mod` 有效） |
| `game_modes` | table | 注册新的游戏模式 |

关于 `standalone`，引擎中的检查非常简单（`scripts/modindex.lua` 第 919-922 行）：

```lua
function ModIndex:IsModStandalone(modname)
    local known_mod = self.savedata.known_mods[modname]
    return known_mod and known_mod.modinfo and known_mod.modinfo.standalone == true
end
```

如果某个启用的 Mod 的 `standalone` 为 `true`，`GetModsToLoad` 在准备加载列表时会排除其他所有 Mod。这个字段极少使用，通常用于全面修改（Total Conversion）类型的 Mod。

关于 `mod_dependencies`，引擎会在 `LoadModInfo` 中解析（第 571-588 行）：

```lua
if info.mod_dependencies and not info.client_only_mod then
    local dependencies = {}
    self.savedata.known_mods[modname].dependencies = dependencies
    for i, v in ipairs(info.mod_dependencies) do
        local enabledmod = FindEnabledMod(self, v)
        if enabledmod then
            table.insert(dependencies, {enabledmod})
        else
            local mods = BuildModPriorityList(self, v, IsWorkshopMod(modname))
            if #mods == 0 then
                modprint("no valid dependent mod found for mod "..modname)
                self:DisableBecauseBad(modname)  -- 找不到依赖 → 禁用 Mod
            end
            table.insert(dependencies, mods)
        end
    end
end
```

注意：`mod_dependencies` 目前**不支持客户端 Mod**（代码中有 `not info.client_only_mod` 的条件判断，注释写着 `--todo(Zachary): support client mods in the future`）。

---

### 3.1.3 深入引擎：modinfo.lua 到底是怎么被加载的

> 本节面向老手，带你走一遍引擎从磁盘读取 modinfo.lua 到最终使用的完整流程。

#### 第一步：发现 Mod 文件夹

游戏启动时，`main.lua`（第 516-525 行）调用：

```lua
KnownModIndex:Load(function()
    KnownModIndex:BeginStartupSequence(function()
        ModSafeStartup()
    end)
end)
```

`KnownModIndex:Load()` 会先从磁盘加载之前保存的 modindex 文件（记录了哪些 Mod 被启用），然后调用 `UpdateModInfo()`。

`UpdateModInfo()`（第 326-364 行）做三件事：
1. 通过引擎 C++ 函数 `TheSim:GetModDirectoryNames()` 扫描 `mods/` 目录下所有子文件夹
2. 清理已被删除的 Mod 的记录（除非它被临时启用）
3. 对每个 Mod 文件夹调用 `LoadModInfo(modname)`

#### 第二步：执行 modinfo.lua（沙盒环境）

核心代码在 `ModIndex:InitializeModInfo`（`modindex.lua` 第 593 行）：

```lua
function ModIndex:InitializeModInfo(modname)
    local env = {
        folder_name = modname,               -- Mod 文件夹名，如 "workshop-123456789"
        locale = LOC.GetLocaleCode(),         -- 当前游戏语言代码，如 "zh"
        ChooseTranslationTable = function(tbl)  -- 国际化辅助函数
            local locale = LOC.GetLocaleCode()
            return tbl[locale] or tbl[1]
        end,
    }
    local fn = kleiloadlua(MODS_ROOT..modname.."/modinfo.lua")
```

引擎首先建立一个**沙盒环境 `env`**，里面只预置了三个东西：
- `folder_name`：Mod 的文件夹名称
- `locale`：当前游戏语言代码
- `ChooseTranslationTable()`：一个辅助函数，用于按语言选择翻译表

然后用 `kleiloadlua()` 加载文件（类似 Lua 的 `loadfile`，但由 Klei 引擎实现），最后用 `RunInEnvironment()` 在沙盒中执行：

```lua
local status, r = RunInEnvironment(fn, env)
```

**这就是 modinfo.lua 的本质：它不是一个配置文件，而是一段 Lua 程序。** 你在里面写的 `name = "xxx"` 实际上是在 `env` 表上赋值。执行完成后，`env` 表中就包含了所有你在 modinfo.lua 中定义的字段。

这也是为什么你可以在 modinfo.lua 中写 Lua 逻辑，例如：

```lua
-- 根据语言切换名称（神话书说的做法）
local L = locale ~= "zh" and locale ~= "zhr"
name = L and "[DST] Myth Words" or "[DST] 神话书说主题初心版"

-- 判断是否为创意工坊版本（Gem Core 的做法）
if not folder_name:find("workshop-") then
    name = name .. " - GitLab Version"
end

-- 动态生成描述
version = "2.0"
description = "Version: " .. version .. "\n这是一个很棒的 Mod"
```

但要注意：**沙盒环境中只有 `folder_name`、`locale`、`ChooseTranslationTable` 这三个预置变量。** 你无法使用 `ThePlayer`、`GLOBAL`、`require`、`TheNet` 等任何游戏运行时才有的东西，因为 modinfo 在游戏启动的早期阶段就被加载了，此时游戏世界还不存在。

#### 第三步：后处理

`LoadModInfo`（第 511 行）在 `InitializeModInfo` 返回后，还会做以下后处理：

1. **标准化版本号**：trim 空白 + 转小写
2. **处理图标路径**：转为绝对路径，验证文件是否存在
3. **注册游戏模式**：如果定义了 `game_modes`，调用 `AddGameMode` 注册
4. **解析依赖关系**：如果定义了 `mod_dependencies`，查找依赖 Mod 是否可用
5. **版本缓存优化**：如果版本号没变（`prev_info.version == info.version`），直接返回上一次的解析结果，跳过后续所有处理——这是一个性能优化，避免每次都重新处理图标和依赖

```lua
-- 版本缓存优化：如果版本没变，直接复用旧的解析结果
if prev_info ~= nil and prev_info.version == info.version then return prev_info end
```

#### 第四步：在 modmain.lua 中可用

最终，当 `ModWrangler:LoadMods()` 加载你的 `modmain.lua` 时，会将完整的 modinfo 数据存入 Mod 的运行环境：

```lua
-- scripts/mods.lua 第 513-521 行
local initenv = KnownModIndex:GetModInfo(modname)
local env = CreateEnvironment(modname, self.worldgen)
env.modinfo = initenv  -- modinfo 的所有字段都可以通过 env.modinfo 访问
```

这意味着在 `modmain.lua` 中你可以通过 `env.modinfo.xxx` 访问自己的 modinfo 字段。虽然一般不需要这么做（因为 `GetModConfigData` 已经封装好了配置读取），但在某些特殊场景下（如读取自定义字段）这很有用。

#### 完整流程图

```
游戏启动
  │
  ├─ main.lua: KnownModIndex:Load()
  │    │
  │    ├─ 从磁盘加载 modindex（启用/禁用状态）
  │    │
  │    └─ UpdateModInfo()
  │         │
  │         ├─ TheSim:GetModDirectoryNames() → 扫描 mods/ 目录
  │         │
  │         └─ 对每个 Mod 调用 LoadModInfo(modname)
  │              │
  │              ├─ InitializeModInfo(modname)
  │              │    ├─ 创建沙盒 env（folder_name, locale, ChooseTranslationTable）
  │              │    ├─ kleiloadlua("mods/<modname>/modinfo.lua")
  │              │    ├─ RunInEnvironment(fn, env) → 执行 modinfo.lua
  │              │    ├─ api_version_dst → api_version 替换
  │              │    ├─ 必填字段检查（name, description, author, version, api_version）
  │              │    └─ 兼容性默认值处理
  │              │
  │              ├─ 版本号标准化（trim + lowercase）
  │              ├─ 图标路径验证与转换
  │              ├─ game_modes 注册
  │              └─ mod_dependencies 解析
  │
  └─ ModWrangler:LoadMods()
       │
       ├─ 按 priority 排序
       ├─ 为每个 Mod 创建运行环境 env
       ├─ env.modinfo = KnownModIndex:GetModInfo(modname)
       └─ 执行 modmain.lua
```

---

### 3.1.4 自定义字段：引擎之外的约定

由于 `modinfo.lua` 本质上是一个 Lua 表，你可以在里面写任何自定义字段。引擎不认识的字段会被忽略，但**其他 Mod 可以读取它们**。

例如 Gem Core 定义了一个自定义标志：

```lua
-- Gem Core 的 modinfo.lua
GemCore = true
```

其他 Mod 可以在运行时通过 `KnownModIndex:GetModInfo(modname).GemCore` 来判断 Gem Core 是否安装并获取其信息。

再如神话书说定义了：

```lua
-- 神话书说的 modinfo.lua
StaticAssetsReg = {'gourd'}
```

这是一个自定义字段，用于控制资源的静态加载行为。引擎本身不处理它，但 Mod 内部的代码会读取并使用这个值。

> **老手提示**：这种跨 Mod 通信方式简单但有限——你只能读取字段值，无法调用方法或建立双向通信。更复杂的 Mod 互操作应该在 `modmain.lua` 层面通过全局变量、全局函数或共享的 component 实现。此外要注意，modinfo 数据在游戏启动时就已经加载完成，后续不会再刷新，所以自定义字段的值是静态的。

---

### 3.1.5 完整范例：一个功能齐全的 modinfo.lua

下面是一个综合运用了本节所有知识点的完整范例，包含国际化、配置选项、服务器标签等：

```lua
-- 利用引擎预置的 locale 做国际化
local is_chinese = locale == "zh" or locale == "zhr"

name = is_chinese and "超级武器包" or "Super Weapons Pack"
description = is_chinese
    and "添加了 10 把全新的武器到游戏中！\n版本：1.2.0"
    or "Adds 10 new weapons to the game!\nVersion: 1.2.0"
author = "YourName"
version = "1.2.0"

forumthread = ""

-- 联机版 API 版本，固定填 10
api_version = 10

-- 兼容性声明
dst_compatible = true
dont_starve_compatible = false
reign_of_giants_compatible = false

-- 网络模式：服务器 Mod，所有客户端必装
all_clients_require_mod = true
client_only_mod = false

-- 加载优先级（默认为 0，一般不用改）
priority = 0

-- 图标（128x128 的 tex + xml）
icon_atlas = "modicon.xml"
icon = "modicon.tex"

-- 服务器搜索标签
server_filter_tags = {
    "weapons",
    "items",
}

-- 配置选项
configuration_options = {
    {
        name = "weapon_damage_mult",
        label = is_chinese and "武器伤害倍率" or "Weapon Damage Multiplier",
        hover = is_chinese and "调整所有武器的伤害" or "Adjust damage for all weapons",
        options = {
            { description = "0.5x", data = 0.5 },
            { description = "1.0x", data = 1.0 },
            { description = "1.5x", data = 1.5 },
            { description = "2.0x", data = 2.0 },
        },
        default = 1.0,
    },
    {
        name = "drop_on_death",
        label = is_chinese and "死亡掉落" or "Drop on Death",
        hover = is_chinese and "死亡时是否掉落特殊武器" or "Drop special weapons on death?",
        options = {
            { description = is_chinese and "是" or "Yes", data = true },
            { description = is_chinese and "否" or "No", data = false },
        },
        default = true,
    },
}
```

---

### 3.1.6 常见问题与陷阱

**Q: Mod 在列表中显示但无法启用？**

A: 最常见的原因是 `api_version` 不对。必须为 10（联机版当前值）。如果大于 10 会直接 `failed`；小于 10 会有 "Old API" 警告。另一个可能是缺少了五个必填字段中的某个。

**Q: 开了服务器但朋友加入时看不到我的 Mod？**

A: 确认 `all_clients_require_mod = true` 且 `client_only_mod = false`。如果 `all_clients_require_mod = false`，服务器不会通告这个 Mod，客户端自然不会去下载。

**Q: `client_only_mod` 和 `all_clients_require_mod` 可以同时为 true 吗？**

A: 技术上可以写，引擎不会因此拒绝加载你的 Mod，但会打印一条警告。这两个标志在语义上互斥：一个说「只在客户端运行」，另一个说「服务器要求所有客户端安装」——逻辑矛盾。建议不要这样做。

**Q: modinfo.lua 中可以 `require` 其他文件吗？**

A: 不可以。沙盒环境中没有 `require` 函数。modinfo.lua 的执行环境是极度受限的，只有 `folder_name`、`locale`、`ChooseTranslationTable` 以及 Lua 的基本语法。如果你需要复用代码，只能直接写在 modinfo.lua 中（定义局部变量和函数是可以的）。

**Q: `priority` 能设为负数吗？**

A: 可以。负数表示比默认（0）更晚加载。例如人物模板用了 `-99999999999`，确保在所有 API 类 Mod 加载完之后才加载。

**Q: 我在 modinfo.lua 中写了 `print()`，能在日志中看到输出吗？**

A: 沙盒环境中没有注入 `print` 函数，因此大概率无法使用。如果你需要调试 modinfo，建议直接检查游戏日志中是否有 `Error loading mod` 或 `WARNING loading modinfo.lua` 相关的错误信息。

**Q: `version` 字段有格式要求吗？**

A: 没有严格的格式要求。引擎只做 trim 和小写处理。你可以写 `"1.0"`、`"1.0.0"`、`"v2.3-beta"` 甚至 `"6.6.6.10"` 都行。但建议使用一致的语义化版本号（如 `"主版本.次版本.修订号"`），方便自己和其他 Mod 开发者追踪变更。

---

### 3.1.7 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | 5 个必填字段（name, description, author, version, api_version）+ `dst_compatible` + `all_clients_require_mod` / `client_only_mod` + 图标，这就够了 |
| **进阶** | `configuration_options` 的完整结构与 `GetModConfigData` 的读取逻辑、`priority` 的排序机制、`server_filter_tags` 的作用、利用 `locale` 做国际化 |
| **老手** | modinfo.lua 是在沙盒 `env` 中被 `RunInEnvironment` 执行的 Lua 代码；`folder_name`、`locale`、`ChooseTranslationTable` 是唯一的预置变量；自定义字段可实现跨 Mod 通信；版本缓存优化机制；`api_version_dst` 的覆盖逻辑 |


## 3.2 modmain.lua——Mod 的入口

（待编写）

## 3.3 Mod 运行环境：沙盒 env、GLOBAL 表与 modimport

（待编写）

## 3.4 modutil.lua 详解——所有 Mod API 的实际实现

（待编写）

## 3.5 Mod API 速览：AddPrefabPostInit、AddComponentPostInit 等

（待编写）

## 3.6 客户端 Mod vs 服务端 Mod——client_only_mod 与 all_clients_require_mod

（待编写）

## 3.7 Mod 加载顺序与 priority 的影响

（待编写）

## 3.8 在 Mod 中覆盖与添加 STRINGS（物品名、描述、角色对话）

（待编写）

## 3.9 实战：修改一把武器的伤害

（待编写）
