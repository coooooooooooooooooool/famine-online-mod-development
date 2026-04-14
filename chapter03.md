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

### 本节导读

如果说 `modinfo.lua` 是 Mod 的「身份证」，那么 `modmain.lua` 就是 Mod 的「大脑」——你所有想对游戏做的修改，最终都要从这个文件出发。它是引擎加载 Mod 逻辑代码的第一入口，也是你和游戏世界交互的起点。

> **新手**可以从「最小可运行范例」开始，了解 modmain.lua 长什么样、怎么写第一行代码；**进阶读者**可以深入理解沙盒环境的运作方式、PostInit 系列函数的原理以及代码组织技巧；**老手**可以直接跳到「深入引擎」部分，看 modmain.lua 从磁盘到执行的完整链路。

---

### 3.2.1 快速入门：最小可运行的 modmain.lua

对于新手来说，一个最简单的 `modmain.lua` 可以只有一行：

```lua
print("Hello from my first mod!")
```

没错，这就够了。当你在游戏中启用这个 Mod 后，游戏日志里就会出现这句话。当然，一行 `print` 不会改变任何游戏行为——我们来写一个真正有意义的最小 Mod。

#### 实例：让所有常青树的生命值翻倍

```lua
AddPrefabPostInit("evergreen", function(inst)
    if inst.components.workable ~= nil then
        local original = inst.components.workable.workleft
        inst.components.workable:SetWorkLeft(original * 2)
    end
end)
```

三行有效代码，实现了一个真实的游戏修改。让我们逐句理解：

1. `AddPrefabPostInit("evergreen", function(inst)` ——「当游戏生成名为 `evergreen` 的 prefab（常青树）之后，执行我给你的这个函数」
2. `inst.components.workable` —— 访问这棵树的「可工作」组件（砍伐就是一种「工作」）
3. `:SetWorkLeft(original * 2)` —— 把剩余砍伐次数设为原来的两倍

这就是 Mod 开发的基本模式：**找到你想改的东西（prefab 或 component），在合适的时机（PostInit）插入你的修改逻辑。**

#### 一个更完整的骨架

实际的 Mod 通常包含更多结构化的内容。下面是一个典型的 modmain.lua 骨架：

```lua
-- 1. 声明 Prefab 文件（如果你的 Mod 添加了新实体/物品）
PrefabFiles = {
    "myweapon",
}

-- 2. 声明美术资源
Assets = {
    Asset("IMAGE", "images/inventoryimages/myweapon.tex"),
    Asset("ATLAS", "images/inventoryimages/myweapon.xml"),
}

-- 3. 读取 modinfo 中的配置选项
local damage_mult = GetModConfigData("damage_multiplier") or 1

-- 4. 修改游戏数值
TUNING.SPEAR_DAMAGE = TUNING.SPEAR_DAMAGE * damage_mult

-- 5. 添加物品名称和描述
STRINGS.NAMES.MYWEAPON = "我的武器"
STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON = "看起来很厉害！"
STRINGS.RECIPE_DESC.MYWEAPON = "一把自制武器"

-- 6. 添加合成配方
AddRecipe2("myweapon", {Ingredient("twigs", 2), Ingredient("flint", 2)},
    TECH.NONE, {atlas = "images/inventoryimages/myweapon.xml", image = "myweapon.tex"})

-- 7. 对已有 prefab 做修改
AddPrefabPostInit("spear", function(inst)
    if inst.components.weapon ~= nil then
        inst.components.weapon:SetDamage(TUNING.SPEAR_DAMAGE)
    end
end)
```

你可能注意到了——`AddPrefabPostInit`、`GetModConfigData`、`AddRecipe2` 这些函数是从哪里来的？你没有 `require` 任何东西，也没有定义它们。答案在于 Mod 的**沙盒环境**。

---

### 3.2.2 Mod 的沙盒环境——你能用什么

当引擎加载 `modmain.lua` 时，并不是把它丢进全局环境去执行，而是为每个 Mod 创建了一个独立的**沙盒环境**（sandbox environment）。这个沙盒里预置了一组精心挑选的变量和函数——你在 modmain.lua 里直接使用的一切，都来自这个沙盒。

#### 沙盒中有什么？

引擎通过 `CreateEnvironment` 函数（`scripts/mods.lua` 第 295 行）为每个 Mod 构建沙盒：

```lua
-- scripts/mods.lua 第 301-332 行
local env = {
    -- Lua 标准库（部分）
    pairs = pairs,
    ipairs = ipairs,
    print = print,
    math = math,
    table = table,
    type = type,
    string = string,
    tostring = tostring,
    require = require,
    Class = Class,

    -- 游戏核心
    TUNING = TUNING,

    -- 世界生成相关常量
    LEVELCATEGORY = LEVELCATEGORY,
    GROUND = GROUND,
    WORLD_TILES = WORLD_TILES,
    LOCKS = LOCKS,
    KEYS = KEYS,
    LEVELTYPE = LEVELTYPE,

    -- Mod 专用
    GLOBAL = _G,          -- 全局环境的引用
    modname = modname,    -- 当前 Mod 的目录名
    MODROOT = MODS_ROOT .. modname .. "/",  -- Mod 根目录路径
}
```

然后再通过 `modutil.InsertPostInitFunctions(env)` 注入大量 Mod API 函数（`AddPrefabPostInit`、`AddRecipe2` 等几十个）。

**为什么要用沙盒？** 有三个原因：

1. **隔离**：每个 Mod 的全局变量实际上写在自己的 `env` 表里，不会污染真正的全局环境 `_G`，也不会和其他 Mod 冲突
2. **安全**：沙盒只暴露了「Mod 应该用的」API，而不是整个游戏内部——降低了 Mod 意外破坏引擎的风险
3. **可控**：引擎可以根据加载阶段决定注入哪些函数——世界生成阶段和游戏运行阶段提供的 API 不同

#### 沙盒中「没有」什么？

以下是你在 modmain.lua 中**不能直接使用**的东西（需要通过 `GLOBAL` 访问）：

```lua
-- 这些在 modmain.lua 中直接写会报错：
TheWorld          -- 需要 GLOBAL.TheWorld
ThePlayer         -- 需要 GLOBAL.ThePlayer
TheNet            -- 需要 GLOBAL.TheNet
TheSim            -- 需要 GLOBAL.TheSim
SpawnPrefab       -- 需要 GLOBAL.SpawnPrefab
ACTIONS            -- 需要 GLOBAL.ACTIONS
STRINGS           -- ⚠ 这个比较特殊，见下文
```

等等——前面的骨架里我们直接写了 `STRINGS.NAMES.MYWEAPON = "xxx"` 啊，为什么没报错？

那是因为 `STRINGS` 恰好不在沙盒中……错了，它确实不在沙盒中。实际上能直接写 `STRINGS` 是因为很多 Mod 使用了一种技巧——把 `env` 的元表指向 `GLOBAL`。我们马上就来讲。

---

### 3.2.3 访问游戏全局：GLOBAL 与 env 的关系

#### 方法一：老老实实用 GLOBAL（推荐新手）

最直接的方式是通过 `GLOBAL` 前缀访问全局变量：

```lua
-- 在 modmain.lua 中
local TheWorld = GLOBAL.TheWorld
local SpawnPrefab = GLOBAL.SpawnPrefab
local STRINGS = GLOBAL.STRINGS
local ACTIONS = GLOBAL.ACTIONS
local Action = GLOBAL.Action
local ActionHandler = GLOBAL.ActionHandler

STRINGS.NAMES.MYWEAPON = "我的武器"
```

这种方式的好处是**明确清晰**——读代码的人一眼就知道哪些变量来自游戏全局、哪些是 Mod 自己定义的。饥荒社区很多风格指南都推荐这种写法。

#### 方法二：env 元表魔法（流行但需理解原理）

在许多 Mod 的 modmain.lua 开头，你会看到这么一段「万能密码」：

```lua
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL.rawget(GLOBAL, k)
    end
})
```

这段代码做了什么？它给 Mod 的沙盒环境 `env` 设置了一个元表，当你在 modmain.lua 中访问一个不存在的变量时，Lua 会自动去 `GLOBAL`（也就是 `_G`）中查找。

有了这个元表后，你就可以直接写：

```lua
-- 不需要 GLOBAL 前缀了
STRINGS.NAMES.MYWEAPON = "我的武器"
local inst = SpawnPrefab("axe")
TheWorld:PushEvent("myevent")
```

**为什么需要 `rawget` 而不是直接 `return GLOBAL[k]`？** 因为饥荒的全局环境开启了 `strict` 模式（在 `scripts/strict.lua` 中），访问不存在的全局变量会报错。`rawget` 绕过了 `strict` 的 `__index` 元方法，直接从表中取值——如果不存在就返回 `nil` 而不是报错。

**这种方法的风险：**

1. **命名冲突**：如果你定义了一个和全局变量同名的局部变量，可能会混淆
2. **调试困难**：报错时不容易看出某个变量到底来自 Mod 环境还是全局
3. **写入问题**：`__index` 只影响**读取**，不影响**写入**。当你在 modmain.lua 中写 `STRINGS.NAMES.MYWEAPON = "xxx"` 时，因为 `STRINGS` 已经在 `GLOBAL` 中存在（是一个 table），`__index` 会先找到它，然后你是在修改这个已存在的 table 的子键——所以能正常工作。但如果你写 `MY_NEW_GLOBAL = 42`，这个值会被写进 `env` 表而不是 `GLOBAL`——因为 `__newindex` 没有被重定向

> **建议**：新手先用方法一（显式 `GLOBAL`），理解了原理后再决定是否用方法二。方法二虽然写起来方便，但对于「搞不懂为什么这行代码不报错而那行报错」的疑惑，往往就是元表在搞鬼。

#### 方法三：批量提取到局部变量

一种折中的方式，既不用每次写 `GLOBAL`，又保持清晰：

```lua
-- modmain.lua 开头
local _G = GLOBAL
local TheWorld = _G.TheWorld
local TheNet = _G.TheNet
local SpawnPrefab = _G.SpawnPrefab
local STRINGS = _G.STRINGS
local ACTIONS = _G.ACTIONS
local FOODTYPE = _G.FOODTYPE
```

这在 Leisure 等大型 Mod 中很常见。好处是：
- 变量来源一目了然
- 局部变量比 `GLOBAL.xxx` 访问更快（Lua 局部变量查找是 O(1)）
- 减少代码中 `GLOBAL` 的噪音

---

### 3.2.4 PrefabFiles 与 Assets——注册你的内容

当你的 Mod 要添加新的实体（物品、角色、怪物等）时，需要在 modmain.lua 中声明两样东西：**Prefab 文件**和**美术资源**。

#### PrefabFiles——告诉引擎你有哪些新 Prefab

```lua
PrefabFiles = {
    "myweapon",      -- 对应 scripts/prefabs/myweapon.lua
    "mycharacter",   -- 对应 scripts/prefabs/mycharacter.lua
}
```

引擎会在你的 Mod 的 `scripts/prefabs/` 目录下查找这些文件并加载它们。每个文件应该返回一个或多个 `Prefab` 实例（和原版 prefab 文件的格式一样）。

**为什么需要声明？** 因为原版游戏有一个自动生成的列表 `prefablist.lua` 告诉引擎要加载哪些 prefab 文件。你的 Mod 的 prefab 不在这个列表里，所以需要通过 `PrefabFiles` 手动注册。

引擎在 `scripts/mods.lua` 的 `LoadMods` 中会处理这些声明——把你的 prefab 文件合并到总加载列表中。

#### Assets——告诉引擎你有哪些美术资源

```lua
Assets = {
    Asset("IMAGE", "images/inventoryimages/myweapon.tex"),
    Asset("ATLAS", "images/inventoryimages/myweapon.xml"),
    Asset("ANIM", "anim/myweapon.zip"),
    Asset("ANIM", "anim/swap_myweapon.zip"),
    Asset("SOUNDPACKAGE", "sound/mysound.fev"),
}
```

`Asset` 函数接受两个参数：资源类型和相对路径（相对于 Mod 根目录）。常见的类型有：

| 类型 | 用途 | 文件格式 |
|------|------|---------|
| `"IMAGE"` | 贴图（物品栏图标、UI 图片等） | `.tex` |
| `"ATLAS"` | 贴图的布局描述文件 | `.xml` |
| `"ANIM"` | 动画包（角色、物品、特效等） | `.zip` |
| `"SOUNDPACKAGE"` | 音效包 | `.fev` |
| `"SOUND"` | 音效库 | `.fsb` |

**为什么 prefab 文件里也有 `assets`，modmain 里还要再声明一遍？**

这是一个常见的困惑。答案是：它们在不同时机被加载。

- **prefab 内部的 `assets`**：在 prefab 被注册时加载，确保该 prefab 生成时动画和贴图可用
- **modmain 中的 `Assets`**：在 Mod 初始化时立即加载，用于那些不属于任何特定 prefab 的资源（如 UI 图标、小地图标记等）

如果你把所有资源都声明在 prefab 文件内部，modmain 的 `Assets` 可以留空。但对于物品栏图标、小地图图标等需要在 prefab 生成之前就可用的资源，建议放在 modmain 中。

#### 实际案例：人物 Mod 的资源注册

来看人物 Mod（renwu）是怎么注册大量资源的：

```lua
-- 人物立绘和选人界面资源
PrefabFiles = {
    "haze",            -- 角色 prefab
    "haze_none",       -- 角色相关实体
}

Assets = {
    Asset("IMAGE", "images/saveslot_portraits/haze.tex"),
    Asset("ATLAS", "images/saveslot_portraits/haze.xml"),
    Asset("IMAGE", "bigportraits/haze.tex"),
    Asset("ATLAS", "bigportraits/haze.xml"),
}
```

当资源很多时，可以用循环来简化：

```lua
-- 批量注册物品图标
local items = {"myweapon", "myarmor", "myhat"}
for _, name in ipairs(items) do
    table.insert(Assets, Asset("IMAGE", "images/inventoryimages/"..name..".tex"))
    table.insert(Assets, Asset("ATLAS", "images/inventoryimages/"..name..".xml"))
    -- 同时注册到物品栏 atlas 系统
    RegisterInventoryItemAtlas("images/inventoryimages/"..name..".xml", name..".tex")
end
```

`RegisterInventoryItemAtlas` 是沙盒中提供的函数，它告诉游戏的 UI 系统「当需要显示这个物品的图标时，去这个 atlas 文件里找」。不调用它的话，物品在背包里会显示为白色方块。

---

### 3.2.5 modimport——拆分代码的利器

当你的 Mod 变得复杂时，把所有代码塞进一个 modmain.lua 会导致文件变得臃肿难维护。`modimport` 就是为解决这个问题而生的——它让你可以把代码拆分到多个文件中。

#### 基本用法

```lua
-- modmain.lua
modimport("scripts/main/my_tuning.lua")      -- 加载数值配置
modimport("scripts/main/my_hooks.lua")        -- 加载各种 PostInit 钩子
modimport("scripts/main/my_recipes.lua")      -- 加载合成配方
```

`modimport` 的路径是**相对于 Mod 根目录**的。上面的代码会依次加载：
- `mods/你的mod/scripts/main/my_tuning.lua`
- `mods/你的mod/scripts/main/my_hooks.lua`
- `mods/你的mod/scripts/main/my_recipes.lua`

#### modimport 的实现原理

```lua
-- scripts/mods.lua 第 337-352 行
env.modimport = function(modulename)
    print("modimport: "..env.MODROOT..modulename)
    if string.sub(modulename, #modulename-3,#modulename) ~= ".lua" then
        modulename = modulename..".lua"
    end
    local result = kleiloadlua(env.MODROOT..modulename)
    if result == nil then
        error("Error in modimport: "..modulename.." not found!")
    elseif type(result) == "string" then
        error("Error in modimport: "..ModInfoname(modname).." importing "..modulename.."!\n"..result)
    else
        setfenv(result, env.env)
        result()
    end
end
```

关键点：`setfenv(result, env.env)` ——被导入的文件**共享 modmain 的沙盒环境**。这意味着：
- 在导入文件中可以直接使用 `AddPrefabPostInit`、`TUNING`、`GLOBAL` 等所有 modmain 中可用的东西
- 在导入文件中定义的全局变量（不加 `local`）会写入 `env` 表，modmain 和其他导入文件都能看到
- 和 modmain 本身完全一样的权限，没有额外限制

#### modimport vs require

这是一个重要的区分：

| 特性 | `modimport` | `require` |
|------|-------------|-----------|
| 搜索起点 | Mod 根目录 | `scripts/` 全局路径 |
| 执行环境 | Mod 的沙盒 `env` | 全局环境 `_G` |
| 缓存 | 无缓存，每次都重新执行 | 有缓存，同名文件只执行一次 |
| 路径格式 | 含 `.lua` 后缀（不写会自动补） | 不含 `.lua` 后缀 |
| 典型用途 | 导入 Mod 自己的逻辑代码 | 导入组件、工具库等 |

**什么时候用 `modimport`，什么时候用 `require`？**

- 你的代码需要用 `AddPrefabPostInit` 等 Mod API → 用 `modimport`
- 你的代码是一个独立的组件或工具函数，不依赖 Mod API → 用 `require`
- 你要加载原版或其他 Mod 的文件 → 用 `require`

#### 实际案例：人物 Mod 的代码组织

```lua
-- renwu/modmain.lua
modimport("scripts/main/TS_TUNING")    -- 数值常量（攻击力、血量等）
modimport("scripts/main/TS_HOOK")      -- 钩子函数（PostInit 注册等）
modimport("scripts/main/TS_DATA")      -- 数据表（配方、物品信息等）
```

人物模板甚至用变量拼接路径，实现了「改一个名字就能换角色」的模板化：

```lua
-- 人物模板 modmain.lua
STRINGS.MOD_CHARACTER_PREFAB_NAME = "wisprain"

modimport("scripts/main/"..STRINGS.MOD_CHARACTER_PREFAB_NAME.."_strings")
modimport("scripts/main/"..STRINGS.MOD_CHARACTER_PREFAB_NAME.."_skins")
```

这种写法让同一套模板可以轻松复制出不同角色的 Mod——只需修改 `STRINGS.MOD_CHARACTER_PREFAB_NAME` 和对应的文件名。

---

### 3.2.6 PostInit 系列——Mod 的核心武器

PostInit 函数是饥荒 Mod 开发中最核心的概念之一。它们的工作原理是：**你注册一个回调函数，引擎在特定对象创建/初始化后自动调用它。** 这让你可以在不修改原版文件的前提下，对游戏中的几乎任何东西进行修改。

#### 6 个最常用的 PostInit 函数

**1. `AddPrefabPostInit(prefab, fn)` —— 修改特定 Prefab**

```lua
-- 让长矛多 10 点伤害
AddPrefabPostInit("spear", function(inst)
    if inst.components.weapon ~= nil then
        local base = inst.components.weapon.damage
        inst.components.weapon:SetDamage(base + 10)
    end
end)
```

注册逻辑（`scripts/modutil.lua` 第 592-599 行）：

```lua
env.postinitfns.PrefabPostInit = {}
env.AddPrefabPostInit = function(prefab, fn)
    if env.postinitfns.PrefabPostInit[prefab] == nil then
        env.postinitfns.PrefabPostInit[prefab] = {}
    end
    table.insert(env.postinitfns.PrefabPostInit[prefab], fn)
end
```

函数做的事情很简单——以 prefab 名为键，把你的回调 `fn` 存入一个列表。**真正的执行发生在 prefab 生成时。**

来看引擎端的执行代码（`scripts/mainfunctions.lua` 第 368-385 行）：

```lua
-- 在 SpawnPrefab 的内部
local modfns = modprefabinitfns[inst.prefab or name]
if modfns ~= nil then
    for k, mod in pairs(modfns) do
        mod(inst)  -- 对每个注册的回调，传入新生成的实体
    end
end
```

每当游戏生成一个 prefab 时，引擎会检查是否有 Mod 注册了对应的 PostInit 回调，有的话就依次执行。回调的参数 `inst` 就是刚刚创建好的实体实例——你可以对它做任何修改。

**2. `AddPrefabPostInitAny(fn)` —— 修改所有 Prefab**

```lua
-- 给游戏中所有实体添加一个自定义 tag
AddPrefabPostInitAny(function(inst)
    if inst:HasTag("monster") then
        inst:AddTag("my_mod_tracked")
    end
end)
```

这个函数对**每一个**生成的 prefab 都执行。强大但要小心——如果回调逻辑很重，会严重影响性能（因为游戏中每帧都在生成各种 prefab）。

**3. `AddPlayerPostInit(fn)` —— 修改玩家**

```lua
-- 给所有玩家增加 50 点生命上限
AddPlayerPostInit(function(inst)
    if inst.components.health ~= nil then
        inst.components.health:SetMaxHealth(inst.components.health.maxhealth + 50)
    end
end)
```

这其实是 `AddPrefabPostInitAny` 的一个封装——引擎的实现非常直白：

```lua
-- scripts/modutil.lua 第 586-590 行
env.AddPlayerPostInit = function(fn)
    env.AddPrefabPostInitAny(function(inst)
        if inst and inst:HasTag("player") then fn(inst) end
    end)
end
```

它就是给 `AddPrefabPostInitAny` 加了一层 `HasTag("player")` 过滤。这避免了每个 Mod 作者都自己写同样的判断逻辑。

**4. `AddComponentPostInit(component, fn)` —— 修改组件**

```lua
-- 让所有「可工作」的东西（树、矿、图腾柱等）多需要 1 次砍伐/锤击
AddComponentPostInit("workable", function(self)
    local old_setworkaction = self.SetWorkAction
    self.SetWorkAction = function(self, action)
        old_setworkaction(self, action)
        if self.maxwork ~= nil then
            self:SetWorkLeft(self.maxwork + 1)
        end
    end
end)
```

注意回调的参数是 `self`（组件实例本身），而不是 `inst`（实体）。如果需要访问实体，用 `self.inst`。

引擎执行组件 PostInit 的代码在 `scripts/entityscript.lua` 第 636 行：

```lua
-- 当任何实体调用 AddComponent(name) 时
local postinitfns = ModManager:GetPostInitFns("ComponentPostInit", name)
for _, fn in ipairs(postinitfns) do
    fn(loadedcmp, self)  -- 传入组件实例和实体
end
```

**`AddPrefabPostInit` vs `AddComponentPostInit` 选哪个？**

| 场景 | 推荐用 |
|------|--------|
| 只想改一种具体物品（如「长矛」） | `AddPrefabPostInit("spear", ...)` |
| 想改所有拥有某种能力的物品（如所有武器） | `AddComponentPostInit("weapon", ...)` |
| 想改特定 Mod 添加的物品 | `AddPrefabPostInit("mod_item_name", ...)` |
| 想改游戏核心机制（如所有物品的耐久） | `AddComponentPostInit("finiteuses", ...)` |

**5. `AddSimPostInit(fn)` —— 世界模拟初始化后执行**

```lua
AddSimPostInit(function()
    print("世界已经加载完成！")
end)
```

在世界模拟器（Sim）初始化完成后执行，此时 `TheWorld` 已经可用。适合做全局性的初始化工作。

**6. `AddGamePostInit(fn)` —— 游戏级初始化后执行**

```lua
AddGamePostInit(function()
    print("游戏已经完全就绪！")
end)
```

比 `SimPostInit` 更晚，在游戏完全就绪后执行。

#### PostInit 的执行时序

理解执行顺序有助于排查「为什么我的修改没生效」的问题：

```
1. 引擎启动
2. modinfo.lua 加载（所有 Mod）
3. 按 priority 排序
4. modmain.lua 执行（按排序顺序）
   → 此时 PostInit 函数只是"注册"，还没执行
5. 游戏世界开始加载
6. GamePostInit 回调执行
7. SimPostInit 回调执行
8. Prefab 开始生成
   → 每个 Prefab 生成后，PrefabPostInit 回调执行
   → 每个 Component 添加后，ComponentPostInit 回调执行
```

关键要记住：**modmain.lua 执行时，游戏世界还不存在。** 所以在 modmain 的顶层代码中，你不能直接访问 `TheWorld`、`ThePlayer` 等——它们还是 `nil`。这些操作应该放在 PostInit 回调里。

```lua
-- 错误：modmain 顶层直接访问 TheWorld
local season = TheWorld.state.season  -- 报错！TheWorld 还不存在

-- 正确：放在 PostInit 回调里
AddPrefabPostInit("world", function(inst)
    -- 此时 TheWorld 已经创建
    inst:ListenForEvent("cycleschanged", function()
        print("新的一天！")
    end)
end)
```

#### 更多 PostInit 函数一览

| 函数 | 作用 | 回调参数 |
|------|------|---------|
| `AddStategraphPostInit(sg, fn)` | 修改状态图（动画状态机） | `self`（StateGraph） |
| `AddBrainPostInit(brain, fn)` | 修改 AI 行为树 | `self`（Brain） |
| `AddRecipePostInit(recipe, fn)` | 修改合成配方 | `self`（Recipe） |
| `AddStategraphState(sg, state)` | 给状态图添加新状态 | 无回调，直接添加 |
| `AddStategraphEvent(sg, event)` | 给状态图添加新事件 | 无回调，直接添加 |
| `AddStategraphActionHandler(sg, handler)` | 给状态图添加动作处理器 | 无回调，直接添加 |
| `AddClassPostConstruct(path, fn)` | 修改任意 Class 的构造过程 | 取决于 Class |
| `AddGlobalClassPostConstruct(path, fn)` | 同上，但在全局注册 | 取决于 Class |

> **进阶提示**：`AddClassPostConstruct` 极其强大——它可以修改任何通过 `require` 加载的 Class 文件。比如 `AddClassPostConstruct("screens/playerhud", fn)` 可以修改 HUD 界面，`AddClassPostConstruct("widgets/inventorybar", fn)` 可以修改物品栏。这是很多 UI Mod 的核心手段。

---

### 3.2.7 深入引擎：modmain.lua 到底是怎么被加载的

> 本节面向老手，带你走一遍 modmain.lua 从磁盘到执行的完整流程。

#### 第一步：创建沙盒环境

在 `ModWrangler:LoadMods()` 函数（`scripts/mods.lua` 第 468 行）中，引擎为每个需要加载的 Mod 创建环境：

```lua
-- scripts/mods.lua 第 513-517 行
local initenv = KnownModIndex:GetModInfo(modname)
local env = CreateEnvironment(modname, self.worldgen)
env.modinfo = initenv  -- 把 modinfo 数据挂到环境中
table.insert(self.mods, env)
```

注意 `env.modinfo = initenv`——这意味着在 modmain.lua 中可以通过 `env.modinfo.xxx` 访问自己的 modinfo 字段。

#### 第二步：按 priority 排序

所有 Mod 的环境创建完成后，引擎按 `priority` 排序：

```lua
-- scripts/mods.lua 第 527-559 行
local function modPrioritySort(a, b)
    local apriority = sanitizepriority(a.modinfo.priority)
    local bpriority = sanitizepriority(b.modinfo.priority)
    if apriority == bpriority then
        return stringidsorter(a.modname, b.modname)
    end
    return apriority > bpriority  -- 数字越大越先加载
end
table.sort(self.mods, modPrioritySort)
```

排序后调用 `kleiregistermods(self.mods)` 通知引擎注册所有 Mod 的资源路径。

#### 第三步：依次加载 modworldgenmain.lua 和 modmain.lua

```lua
-- scripts/mods.lua 第 563-584 行
for i, mod in ipairs(self.mods) do
    table.insert(self.enabledmods, mod.modname)
    -- 1. 把 Mod 的 scripts 目录加入搜索路径
    package.path = MODS_ROOT..mod.modname.."\\scripts\\?.lua;"..package.path

    -- 2. 处理 manifest（创意工坊资源管理）
    -- ...

    -- 3. 加载世界生成脚本（如果有的话）
    self:InitializeModMain(mod.modname, mod, "modworldgenmain.lua")

    -- 4. 加载主脚本（非世界生成阶段）
    if not self.worldgen then
        self:InitializeModMain(mod.modname, mod, "modmain.lua")
    end
end
```

注意第 1 步——`package.path` 的修改是**累加**的，每个 Mod 的 `scripts` 目录都被加到了搜索路径前面。这就是为什么你可以在 Mod 中用 `require("components/mycomponent")` 加载自己写的组件——引擎会先搜索你的 Mod 目录，再搜索原版目录。

**这也意味着：如果你的 Mod 中有一个和原版同名的文件（如 `scripts/components/health.lua`），`require` 会优先加载你的版本。** 这是一种替换原版文件的手段，但也容易造成兼容性问题。

#### 第四步：编译并执行 modmain.lua

核心函数 `InitializeModMain`（`scripts/mods.lua` 第 587-619 行）：

```lua
function ModWrangler:InitializeModMain(modname, env, mainfile, safe)
    if not KnownModIndex:IsModCompatibleWithMode(modname) then return end

    local fn = kleiloadlua(MODS_ROOT..modname.."/"..mainfile)
    if type(fn) == "string" then
        -- 加载失败（语法错误等），fn 是错误信息
        table.insert(self.failedmods, {name=modname, error=fn})
        return false
    elseif not fn then
        -- 文件不存在，跳过（不是错误）
        return true
    else
        -- 成功加载，在沙盒环境中执行
        local status, r = RunInEnvironment(fn, env)

        if status == false then
            -- 运行时错误
            moderror("Mod: "..ModInfoname(modname), "  Error loading mod!\n"..r)
            table.insert(self.failedmods, {name=modname, error=r})
            return false
        else
            return true
        end
    end
end
```

`RunInEnvironment` 的实现（`scripts/util.lua` 第 786-788 行）只有两行：

```lua
function RunInEnvironment(fn, fnenv)
    setfenv(fn, fnenv)          -- 把沙盒环境设为函数的环境
    return xpcall(fn, debug.traceback)  -- 安全调用，捕获错误
end
```

`setfenv(fn, fnenv)` 是整个 Mod 系统的灵魂——它把 `kleiloadlua` 编译得到的函数的环境替换为 Mod 的沙盒 `env`。之后 `xpcall` 执行这个函数，其中所有全局变量的读写都发生在 `env` 表上。

#### 第五步：PostInit 回调的收集与分发

当 modmain.lua 执行完毕后，`env.postinitfns` 表中已经存储了所有注册的回调。引擎通过 `ModWrangler:GetPostInitFns` 来查询这些回调：

```lua
-- scripts/mods.lua 第 855-871 行
function ModWrangler:GetPostInitFns(type, id)
    local retfns = {}
    for i, modname in ipairs(self.enabledmods) do
        local mod = self:GetMod(modname)
        if mod.postinitfns[type] then
            local modfns = id and mod.postinitfns[type][id] or mod.postinitfns[type]
            if modfns ~= nil then
                for i, modfn in ipairs(modfns) do
                    table.insert(retfns, runmodfn(modfn, mod, ...))
                end
            end
        end
    end
    return retfns
end
```

这个函数做的事情：遍历所有启用的 Mod，按启用顺序收集特定类型的 PostInit 回调，合并成一个列表返回。`runmodfn` 会包装每个回调，添加错误处理——如果你的回调报错了，不会导致整个游戏崩溃，只会在日志中打印错误信息。

#### 完整流程图

```
ModWrangler:LoadMods()
  │
  ├─ 对每个启用的 Mod:
  │    ├─ CreateEnvironment(modname) → 创建沙盒 env
  │    │    ├─ 注入 Lua 标准库子集
  │    │    ├─ 注入 GLOBAL, modname, MODROOT
  │    │    ├─ 注入 modimport 函数
  │    │    └─ modutil.InsertPostInitFunctions(env) → 注入所有 Mod API
  │    │         ├─ 世界生成阶段 API（AddLevelPreInit 等）
  │    │         └─ 游戏阶段 API（AddPrefabPostInit 等）← worldgen 时不注入
  │    └─ env.modinfo = 从 modinfo.lua 解析的数据
  │
  ├─ 按 priority 排序所有 Mod 环境
  │
  └─ 对每个 Mod（按排序后的顺序）:
       ├─ package.path 加入 Mod 的 scripts 目录
       ├─ InitializeModMain("modworldgenmain.lua")
       │    ├─ kleiloadlua → 编译文件为函数 fn
       │    ├─ setfenv(fn, env) → 替换函数环境为沙盒
       │    └─ xpcall(fn) → 执行
       │         └─ 文件中的 AddLevelPreInit 等调用 → 写入 env.postinitfns
       │
       └─ InitializeModMain("modmain.lua")（仅非世界生成阶段）
            ├─ kleiloadlua → 编译
            ├─ setfenv → 替换环境
            └─ xpcall → 执行
                 └─ 文件中的 AddPrefabPostInit 等调用 → 写入 env.postinitfns
                      │
                      └─ [注册完成，等待游戏运行时触发]
                           ├─ Prefab 生成时 → 调用 PrefabPostInit 回调
                           ├─ Component 添加时 → 调用 ComponentPostInit 回调
                           └─ 世界加载后 → 调用 SimPostInit / GamePostInit 回调
```

---

### 3.2.8 世界生成阶段 vs 游戏阶段——两个 modmain

你可能注意到了，引擎会加载两个文件：`modworldgenmain.lua` 和 `modmain.lua`。它们是什么关系？

| 文件 | 执行时机 | 可用 API | 用途 |
|------|---------|---------|------|
| `modworldgenmain.lua` | 世界生成时 | 只有世界生成相关（`AddLevel`、`AddTask`、`AddRoom` 等） | 修改世界地图生成规则 |
| `modmain.lua` | 进入游戏时 | 完整 API（`AddPrefabPostInit`、`AddRecipe2` 等） | 修改游戏内逻辑 |

引擎的处理逻辑（`scripts/modutil.lua` 第 430-438 行）非常清晰：

```lua
----------------------------------------------------------------------
-- Everything above this point is available in Worldgen or Main.
-- Everything below is ONLY available in Main.
-- This allows us to provide easy access to game-time data without
-- breaking worldgen.
----------------------------------------------------------------------
if isworldgen then
    return  -- 世界生成阶段：到此为止，不注入后面的游戏 API
end
----------------------------------------------------------------------
```

如果你的 Mod 只修改游戏内逻辑（大多数情况），你不需要创建 `modworldgenmain.lua`——引擎在找不到文件时会自动跳过。只有当你需要修改世界生成规则（如自定义地形、自定义房间布局等）时，才需要用到它。

---

### 3.2.9 实战：分析一个真实 Mod 的 modmain.lua 结构

让我们以一个典型的人物 Mod 为例，看看一个完整的 modmain.lua 是如何组织的。以下是精简后的结构框架：

```lua
-- ============================================================
-- 第一部分：环境配置
-- ============================================================
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL.rawget(GLOBAL, k)
    end
})

-- ============================================================
-- 第二部分：资源声明
-- ============================================================
PrefabFiles = {
    "mycharacter",
    "mycharacter_none",
}

Assets = {
    Asset("IMAGE", "images/saveslot_portraits/mycharacter.tex"),
    Asset("ATLAS", "images/saveslot_portraits/mycharacter.xml"),
    Asset("IMAGE", "bigportraits/mycharacter.tex"),
    Asset("ATLAS", "bigportraits/mycharacter.xml"),
}

-- 批量注册物品图标
local item_images = {"myweapon", "myarmor"}
for _, name in ipairs(item_images) do
    table.insert(Assets, Asset("IMAGE", "images/inventoryimages/"..name..".tex"))
    table.insert(Assets, Asset("ATLAS", "images/inventoryimages/"..name..".xml"))
    RegisterInventoryItemAtlas("images/inventoryimages/"..name..".xml", name..".tex")
end

-- 小地图图标
AddMinimapAtlas("images/map_icons/mycharacter.xml")

-- ============================================================
-- 第三部分：导入子模块
-- ============================================================
modimport("scripts/main/my_tuning.lua")     -- 数值配置
modimport("scripts/main/my_strings.lua")    -- 文字/台词
modimport("scripts/main/my_recipes.lua")    -- 合成配方
modimport("scripts/main/my_hooks.lua")      -- 各种 PostInit

-- ============================================================
-- 第四部分：角色注册
-- ============================================================
-- 初始物品
TUNING.GAMEMODE_STARTING_ITEMS.DEFAULT.MYCHARACTER = {
    "flint",
    "flint",
    "twigs",
    "twigs",
}

-- 角色台词
STRINGS.CHARACTERS.MYCHARACTER = require("speech_mycharacter")

-- 添加角色到游戏
AddModCharacter("mycharacter", "FEMALE")

-- ============================================================
-- 第五部分：自定义动作与 RPC（如果有的话）
-- ============================================================
local MY_ACTION = Action({distance = 2})
MY_ACTION.id = "MY_ACTION"
MY_ACTION.str = "使用技能"
MY_ACTION.fn = function(act)
    if act.doer and act.doer.components.myskill then
        act.doer.components.myskill:Cast()
        return true
    end
end
AddAction(MY_ACTION)

-- 客户端/服务端通信
AddModRPCHandler("mymod", "cast_skill", function(player)
    if player.components.myskill then
        player.components.myskill:Cast()
    end
end)
```

这个结构展示了一个典型的中等复杂度人物 Mod 的组织方式。关键的设计原则是：

1. **环境配置放最前**——确保后续代码都能正常访问全局变量
2. **资源声明紧随其后**——引擎需要在执行逻辑代码之前知道有哪些资源
3. **用 modimport 拆分逻辑**——保持 modmain.lua 简洁，具体实现放在子文件中
4. **角色/物品注册放中间**——在所有依赖项（字符串、数值等）都准备好之后
5. **高级功能（Action、RPC）放最后**——这些依赖前面的基础设施

---

### 3.2.10 常见问题与陷阱

**Q: 为什么我在 modmain.lua 里写 `local x = ThePlayer`，x 是 nil？**

A: 因为 modmain.lua 在游戏加载的早期阶段执行，此时玩家实体还不存在。`ThePlayer` 要到玩家实际进入世界后才有值。如果你需要对玩家做操作，应该用 `AddPlayerPostInit` 或在回调中访问。

**Q: 我用了 `AddPrefabPostInit` 但修改没生效？**

A: 几种可能：
1. prefab 名拼错了——注意是 prefab 的注册名（全小写），不是显示名。比如金斧头是 `"goldenaxe"` 不是 `"Golden Axe"`
2. 组件不存在——被修改的 prefab 可能没有你以为的那个组件。在修改前先检查 `if inst.components.xxx ~= nil then`
3. 修改被覆盖了——如果其他 Mod 或游戏逻辑在你之后又修改了同一个属性。可以用 `AddPrefabPostInit` 配合 `inst:DoTaskInTime(0, function() ... end)` 延迟一帧执行，确保在所有初始化完成之后

**Q: `modimport` 和 `require` 能混用吗？**

A: 能，但要理解区别。`modimport` 的文件共享 modmain 的沙盒环境（能直接用 `AddPrefabPostInit` 等），而 `require` 的文件运行在全局环境（不能直接用 Mod API）。对于组件文件（`scripts/components/xxx.lua`），用 `require`；对于 Mod 逻辑文件（需要调用 Mod API 的），用 `modimport`。

**Q: 为什么我的 Mod 导致其他 Mod 崩溃了？**

A: 最常见的原因是全局变量污染。检查你的 modmain.lua 和 modimport 的文件中，是否有不加 `local` 的变量定义。虽然 Mod 环境是沙盒的，但通过 `GLOBAL.xxx = yyy` 或元表 `__index` 方式修改的全局变量，是所有 Mod 共享的。

**Q: `env` 的元表 `__index` 指向 GLOBAL 后，我写 `SpawnPrefab(...)` 和 `GLOBAL.SpawnPrefab(...)` 完全一样吗？**

A: **读取**时一样——`__index` 会自动从 GLOBAL 中找到 `SpawnPrefab`。但**写入**时不一样：如果你写 `SpawnPrefab = my_function`，这会把 `my_function` 写进 `env` 表，遮蔽 GLOBAL 中的原版函数——后续你再调用 `SpawnPrefab` 就是在调用你的版本了（但不影响其他 Mod 和原版代码）。如果你想真正替换全局的 `SpawnPrefab`，需要 `GLOBAL.SpawnPrefab = my_function`。

**Q: `modmain.lua` 中的 `require` 路径怎么写？**

A: `require` 以 `scripts/` 为根目录。因为引擎加载你的 Mod 时把 `mods/你的mod/scripts/` 加入了 `package.path`，所以你可以：
- `require("components/mycomponent")` → 加载你的 `scripts/components/mycomponent.lua`
- `require("components/health")` → 因为你的 Mod 路径在前面，如果你有同名文件会优先加载你的版本；如果没有，就加载原版的

**Q: 能不能不写 modmain.lua？**

A: 可以。如果你的 Mod 只修改世界生成规则（只有 `modworldgenmain.lua`），或者只需要 modinfo 的声明（如纯资源包），`modmain.lua` 可以不存在。引擎在 `InitializeModMain` 中会检测文件是否存在，不存在就打印一行 `"Mod had no modmain.lua. Skipping."` 然后跳过。

---

### 3.2.11 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | modmain.lua 是 Mod 逻辑代码的入口；用 `GLOBAL.xxx` 访问游戏变量；`PrefabFiles` 声明新 prefab，`Assets` 声明美术资源；`AddPrefabPostInit` 是最常用的修改手段；`modimport` 用来拆分代码 |
| **进阶** | 理解沙盒环境 `env` 的构成和 `GLOBAL` 的关系；掌握 `AddComponentPostInit`、`AddPlayerPostInit`、`AddClassPostConstruct` 等高级 PostInit；理解 PostInit 的执行时序；用 `env` 元表简化全局访问但注意读写不对称 |
| **老手** | modmain.lua 通过 `kleiloadlua` → `setfenv(fn, env)` → `xpcall(fn)` 链路执行；`postinitfns` 是一个按类型和名称索引的注册表，由 `ModWrangler:GetPostInitFns` 在运行时查询分发；`modworldgenmain.lua` 和 `modmain.lua` 的 API 差异源于 `InsertPostInitFunctions` 中的 `if isworldgen then return end` 分界线；`package.path` 的累加顺序决定了 `require` 的文件优先级 |

## 3.3 Mod 运行环境：沙盒 env、GLOBAL 表与 modimport

### 本节导读

在 3.2 中我们已经知道 modmain.lua 运行在一个「沙盒环境」中，也简单介绍了 `GLOBAL` 和 `modimport`。但如果你认真写过几个 Mod，一定遇到过这样的困惑："为什么这个变量能直接用？""为什么那个函数找不到？""我在 modimport 的文件里定义的变量，modmain 里能看到吗？"

本节将彻底解答这些问题。我们会从 Lua 5.1 的 `setfenv` 机制讲起，一路深入到 `strict.lua` 的严格模式、`env` 的完整构成、`GLOBAL` 的本质、`modimport` 与 `require` 的路径解析，以及多个 Mod 之间的环境关系。

> **新手**可以重点看 3.3.1-3.3.3，理解「为什么写 `GLOBAL.xxx` 而不是直接写 `xxx`」；**进阶读者**可以深入 3.3.4-3.3.6，掌握 `modimport` 与 `require` 的路径机制和环境共享规则；**老手**可以关注 3.3.7-3.3.8，理解 `setfenv`、`strict.lua`、`package.path` 的全貌。

---

### 3.3.1 什么是「环境」——Lua 5.1 的 setfenv

在深入饥荒的沙盒之前，我们需要理解一个 Lua 5.1 特有的核心概念：**函数环境（function environment）**。

在 Lua 5.1 中，每个函数都有一个关联的**环境表**。当函数内部读写一个"全局变量"时，实际上是在这个环境表上进行操作。默认情况下，所有函数的环境都是 `_G`（真正的全局表）。

```lua
-- 默认情况：函数环境是 _G
function foo()
    x = 42  -- 相当于 _G.x = 42
end
```

但 Lua 5.1 提供了 `setfenv` 函数，可以**替换**一个函数的环境：

```lua
local my_env = { print = print }  -- 一个只有 print 的小环境

local function foo()
    print("hello")  -- 可以用，因为 my_env 里有 print
    x = 42          -- 写入 my_env.x，而不是 _G.x
    os.time()       -- 报错！my_env 里没有 os
end

setfenv(foo, my_env)  -- 替换 foo 的环境
foo()
```

`setfenv(foo, my_env)` 执行后，`foo` 内部的所有"全局"读写都发生在 `my_env` 上。它看不到 `_G` 中的 `os`、`io` 等标准库，除非 `my_env` 里显式包含了它们。

**这就是饥荒 Mod 沙盒的全部秘密：引擎用 `setfenv` 把你的 modmain.lua 的环境替换为一个精心构造的 `env` 表。** 你在 modmain 中直接写的任何"全局变量"——无论是读还是写——都在 `env` 上操作，和真正的全局环境 `_G` 完全隔离。

对应的引擎代码：

```lua
-- scripts/util.lua 第 786-788 行
function RunInEnvironment(fn, fnenv)
    setfenv(fn, fnenv)          -- 把函数环境替换为 Mod 的 env
    return xpcall(fn, debug.traceback)  -- 安全执行
end
```

> **给新手的通俗理解**：想象你在一间房间里写代码。默认情况下，所有人共用同一间大房间（`_G`）。但引擎给每个 Mod 分配了一间独立的小房间（`env`），小房间里只放了一些引擎允许你用的工具。你在小房间里定义的变量不会跑到别人的房间。如果你需要大房间里的东西，得通过一扇特殊的门——`GLOBAL`。

---

### 3.3.2 env 的完整构成——小房间里都有什么

让我们完整列出 `env` 表中的所有内容。它由两部分注入：

#### 第一部分：CreateEnvironment 直接放入的

在 `scripts/mods.lua` 的 `CreateEnvironment` 函数（第 295-357 行）中，引擎直接在 `env` 表上放置了以下内容：

| 分类 | 键 | 值 | 说明 |
|------|----|----|------|
| **Lua 基础** | `pairs` | `pairs` | 键值对遍历 |
| | `ipairs` | `ipairs` | 数组遍历 |
| | `print` | `print` | 打印到日志 |
| | `math` | `math` | 数学库 |
| | `table` | `table` | 表操作库 |
| | `type` | `type` | 类型检查 |
| | `string` | `string` | 字符串库 |
| | `tostring` | `tostring` | 转字符串 |
| | `require` | `require` | 模块加载 |
| | `Class` | `Class` | 饥荒的类系统 |
| **游戏核心** | `TUNING` | `TUNING` | 全局数值表 |
| **世界生成** | `GROUND` | `GROUND` | 地面类型常量 |
| | `WORLD_TILES` | `WORLD_TILES` | 地块类型 |
| | `LEVELCATEGORY` | `LEVELCATEGORY` | 关卡分类 |
| | `LOCKS` | `LOCKS` | 锁类型 |
| | `KEYS` | `KEYS` | 钥匙类型 |
| | `LEVELTYPE` | `LEVELTYPE` | 关卡类型 |
| **Mod 专用** | `GLOBAL` | `_G` | 全局环境的引用 |
| | `modname` | 当前 Mod 目录名 | 如 `"workshop-123456"` |
| | `MODROOT` | Mod 根路径 | 如 `"../mods/workshop-123456/"` |
| | `env` | `env` 自身 | 自引用，用于 `setfenv` |
| | `modimport` | 函数 | 从 Mod 根目录导入文件 |
| **条件** | `CHARACTERLIST` | 角色列表 | 仅非世界生成阶段 |

注意一些**不在**列表中的重要 Lua 标准库：

- `os` —— 不在（出于安全考虑，防止 Mod 访问操作系统）
- `io` —— 不在（同上）
- `debug` —— 不在
- `pcall` / `xpcall` —— 不在（但可通过 `GLOBAL.pcall` 获取）
- `error` —— 不在（但有 `moderror`）
- `assert` —— 不在（但有 `modassert`）
- `select` —— 不在
- `unpack` —— 不在
- `rawget` / `rawset` —— 不在
- `setmetatable` / `getmetatable` —— 不在
- `next` —— 不在
- `tonumber` —— 不在

**这意味着在 modmain.lua 中，你不能直接使用 `tonumber`、`setmetatable`、`next` 等基础函数！** 除非你通过 `GLOBAL` 获取它们，或者使用 env 元表技巧。

这就解释了为什么你在很多 Mod 的开头会看到：

```lua
-- 要么通过 GLOBAL 逐个提取
local setmetatable = GLOBAL.setmetatable
local next = GLOBAL.next
local tonumber = GLOBAL.tonumber

-- 要么直接用"万能密码"让 env 回退到 GLOBAL
GLOBAL.setmetatable(env, { __index = function(t, k) return GLOBAL.rawget(GLOBAL, k) end })
```

> **为什么引擎不把所有标准库都放进 env？** 这是一种「最小权限原则」——只给 Mod 它们通常需要的东西。虽然通过 `GLOBAL` 可以获取一切，但这种设计至少明确了「正常的 Mod 开发只需要这些」，也减少了初学者意外使用危险功能（如 `os.execute`）的可能。

#### 第二部分：InsertPostInitFunctions 注入的

在 `CreateEnvironment` 的最后一步，引擎调用 `modutil.InsertPostInitFunctions(env, isworldgen, isfrontend)` 注入大量 Mod API。这些 API 分为两个阶段：

**世界生成 + 游戏阶段都可用的**（`modutil.lua` 第 178-435 行）：

| 函数 | 用途 |
|------|------|
| `modassert` / `moderror` | 断言与错误处理 |
| `GetModConfigData` | 读取 modinfo 中的配置选项 |
| `AddGamePostInit` / `AddSimPostInit` | 游戏/模拟初始化后回调 |
| `AddGlobalClassPostConstruct` / `AddClassPostConstruct` | 修改任意 Class 的构造过程 |
| `AddCustomizeGroup` / `AddCustomizeItem` | 世界设置自定义 |
| `AddLevelPreInit` / `AddTaskPreInit` / `AddRoomPreInit` | 世界生成钩子 |
| `AddLevel` / `AddTaskSet` / `AddTask` / `AddRoom` | 注册世界生成内容 |
| `AddStartLocation` / `AddLocation` | 注册出生点/地点 |
| `RegisterTileRange` / `AddTile` 等 | 地块系统 |
| `ReleaseID` / `CurrentRelease` | 版本标识 |

**仅游戏阶段可用的**（`modutil.lua` 第 438 行之后）：

| 函数 | 用途 |
|------|------|
| `AddPrefabPostInit` / `AddPrefabPostInitAny` / `AddPlayerPostInit` | Prefab 修改 |
| `AddComponentPostInit` | 组件修改 |
| `AddStategraphPostInit` / `AddStategraphState` / `AddStategraphEvent` | 状态图修改 |
| `AddBrainPostInit` | AI 行为树修改 |
| `AddAction` / `AddComponentAction` | 添加/修改动作 |
| `AddRecipe2` / `AddCharacterRecipe` / `AddDeconstructRecipe` | 合成配方 |
| `AddRecipeTab` / `AddRecipeFilter` | 合成栏目 |
| `AddModCharacter` / `RemoveDefaultCharacter` | 角色管理 |
| `AddIngredientValues` / `AddCookerRecipe` | 烹饪系统 |
| `AddModRPCHandler` / `SendModRPCToServer` | 网络通信 |
| `AddReplicableComponent` | 注册可复制组件 |
| `AddUserCommand` / `AddVoteCommand` | 控制台命令 |
| `AddMinimapAtlas` | 小地图图标 |
| `RegisterInventoryItemAtlas` / `RegisterScrapbookIconAtlas` | 物品/图鉴图标 |
| `AddLoadingTip` / `RemoveLoadingTip` | 加载提示 |
| `Prefab` / `Asset` / `Ingredient` | 构造函数 |
| `LoadPOFile` | 加载翻译文件 |
| `RemapSoundEvent` | 音效重映射 |
| ... | 还有更多 |

两阶段的分界线非常清晰：

```lua
-- scripts/modutil.lua 第 430-438 行
----------------------------------------------------------------------
-- Everything above this point is available in Worldgen or Main.
-- Everything below is ONLY available in Main.
-- This allows us to provide easy access to game-time data without
-- breaking worldgen.
----------------------------------------------------------------------
if isworldgen then
    return  -- 世界生成阶段在此返回，后面的 API 不注入
end
----------------------------------------------------------------------
```

**为什么要分两个阶段？** 因为世界生成时，游戏世界还不存在——没有实体、没有组件、没有 `TheWorld`。如果世界生成阶段也注入了 `AddPrefabPostInit`，Mod 作者可能会在 `modworldgenmain.lua` 中误用它，导致难以排查的错误。

---

### 3.3.3 GLOBAL——通往游戏世界的大门

`GLOBAL` 就是 `_G`——Lua 的真正全局环境表。它包含了游戏运行时的一切：

```lua
-- 在 CreateEnvironment 中
GLOBAL = _G,
```

通过 `GLOBAL`，你可以访问任何全局变量和函数：

```lua
-- 游戏对象
GLOBAL.TheWorld          -- 当前世界实例
GLOBAL.ThePlayer         -- 本地玩家实例（客户端）
GLOBAL.TheNet            -- 网络管理器
GLOBAL.TheSim            -- 模拟器
GLOBAL.TheInput          -- 输入管理器

-- 全局函数
GLOBAL.SpawnPrefab       -- 生成实体
GLOBAL.c_spawn           -- 控制台生成命令
GLOBAL.GetTime           -- 获取游戏时间

-- 全局常量表
GLOBAL.STRINGS           -- 所有文本字符串
GLOBAL.ACTIONS           -- 所有动作
GLOBAL.FOODTYPE          -- 食物类型
GLOBAL.TECH              -- 科技等级
GLOBAL.EQUIPSLOTS        -- 装备栏位

-- Lua 标准库（env 中没有的部分）
GLOBAL.setmetatable
GLOBAL.getmetatable
GLOBAL.rawget
GLOBAL.rawset
GLOBAL.next
GLOBAL.pcall
GLOBAL.xpcall
GLOBAL.tonumber
GLOBAL.select
GLOBAL.unpack
GLOBAL.error
GLOBAL.assert
GLOBAL.os                -- ⚠ 慎用
```

#### GLOBAL 与 strict 模式的关系

饥荒在 `scripts/strict.lua` 中给 `_G` 设置了严格模式：

```lua
-- scripts/strict.lua 第 1-32 行
local mt = getmetatable(_G)
if mt == nil then
    mt = {}
    setmetatable(_G, mt)
end

__STRICT = true
mt.__declared = {}

mt.__newindex = function(t, n, v)
    if __STRICT and not mt.__declared[n] then
        local w = debug.getinfo(2, "S").what
        if w ~= "main" and w ~= "C" then
            error("assign to undeclared variable '"..n.."'", 2)
        end
        mt.__declared[n] = true
    end
    rawset(t, n, v)
end

mt.__index = function(t, n)
    if not mt.__declared[n] and debug.getinfo(2, "S").what ~= "C" then
        error("variable '"..n.."' is not declared", 2)
    end
    return rawget(t, n)
end
```

严格模式做了两件事：
1. **读取未声明的全局变量** → 报错 `"variable 'xxx' is not declared"`
2. **在非主代码块中写入未声明的全局变量** → 报错 `"assign to undeclared variable 'xxx'"`

这就是为什么 Mod 中使用 `env` 元表的 `__index` 时必须用 `rawget`：

```lua
-- 正确：绕过 strict 模式
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL.rawget(GLOBAL, k)  -- rawget 不触发 __index 元方法
    end
})

-- 错误：会触发 strict 模式
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL[k]  -- 如果 k 未声明，strict 会报错！
    end
})
```

`rawget(GLOBAL, k)` 直接从表中取值，完全绕过元表（包括 `strict.lua` 设置的 `__index`）。如果键不存在，安静地返回 `nil`，不会报错。

#### 通过 GLOBAL 修改全局变量

需要特别注意**读取**和**写入**的区别：

**读取** `GLOBAL.STRINGS.NAMES.AXE` 时，你拿到的是 `STRINGS` 表中 `NAMES` 表中 `AXE` 的值。因为 `STRINGS` 是一个 table（引用类型），所以你实际上是在操作同一个 table 对象。

**修改子表** `GLOBAL.STRINGS.NAMES.MYWEAPON = "我的武器"` 是安全的——你只是给一个已存在的 table 添加新键，不涉及创建新的全局变量。

**但是**，如果你试图创建一个全新的全局变量：

```lua
GLOBAL.MY_MOD_DATA = {}  -- 在 _G 上创建了新的全局变量
```

这在 strict 模式下**可能报错**（取决于调用上下文）。更安全的做法是用 `rawset`：

```lua
GLOBAL.rawset(GLOBAL, "MY_MOD_DATA", {})
```

或者干脆不创建全局变量，而是挂在 `TUNING` 或其他已存在的表上：

```lua
TUNING.MY_MOD_DATA = {}  -- TUNING 已经在 env 中，且是一个普通 table
```

---

### 3.3.4 变量的查找顺序——当你写 `x` 时到底发生了什么

理解变量查找顺序是理解 Mod 环境的关键。当你在 modmain.lua 中写 `print(TUNING.AXE_DAMAGE)` 时，Lua 的查找过程是：

```
1. 检查局部变量：当前作用域有没有叫 TUNING 的 local 变量？
   → 没有

2. 检查函数环境（env 表）：env.TUNING 存在吗？
   → 存在！返回 env.TUNING（就是全局的 TUNING 表）

3. 如果 env 有元表且定义了 __index：去元表查找
   → 如果设置了"万能密码"，会去 GLOBAL 中找
   → 如果没设置元表，到第 2 步找不到就是 nil
```

图示：

```
你写的代码: TUNING.AXE_DAMAGE
                │
                ▼
         ①局部变量？ ──否──► ②env 表？ ──否──► ③env.__index？
                │              │                   │
               有→返回        有→返回            有→去 GLOBAL 找
                                                   │
                                                  否→nil（或 error）
```

让我们用一个具体的例子来演示：

```lua
-- modmain.lua（假设已设置了 env 元表指向 GLOBAL）

local a = 1               -- 局部变量
TUNING.MY_VALUE = 42       -- env.TUNING 中添加键（TUNING 在 env 中）
b = 2                      -- 写入 env.b（因为 env 是函数环境）
SpawnPrefab("axe")         -- 通过 __index 在 GLOBAL 中找到

print(a)                   -- 1（局部变量）
print(TUNING.MY_VALUE)     -- 42（env.TUNING）
print(b)                   -- 2（env.b）
print(SpawnPrefab)         -- function（GLOBAL.SpawnPrefab，通过 __index）
print(GLOBAL.b)            -- nil（b 在 env 中，不在 _G 中！）
```

最后一行是关键：`b = 2` 写入的是 `env.b`，**不是** `GLOBAL.b`。即使设置了 `__index` 让读取回退到 `GLOBAL`，**写入仍然是写入 env 本身**。这是因为 `__index` 只影响读取操作，写入走的是 `__newindex`——而我们没有重写 `__newindex`，所以写入默认发生在 `env` 上。

这种读写不对称是一个常见的困惑源：

```lua
-- 假设已设置 env 元表指向 GLOBAL

-- 读取：先查 env，env 没有再查 GLOBAL
SpawnPrefab("axe")  -- ✓ env 没有，__index 去 GLOBAL 找到了

-- 写入：直接写入 env
SpawnPrefab = function() print("hacked!") end
-- 现在 env.SpawnPrefab 遮蔽了 GLOBAL.SpawnPrefab
-- 但只影响这个 Mod！其他 Mod 和原版代码不受影响

-- 如果真的想改全局
GLOBAL.SpawnPrefab = function() print("globally hacked!") end
-- 所有代码都受影响——非常危险！
```

---

### 3.3.5 modimport 深入——共享环境的秘密

#### 核心机制：setfenv(result, env.env)

`modimport` 的核心在于这一行：

```lua
setfenv(result, env.env)
```

`result` 是从文件编译得到的函数，`env.env` 就是 `env` 本身（自引用）。这意味着**被 modimport 的文件与 modmain.lua 共享完全相同的环境**。

具体来说：

```lua
-- modmain.lua
my_global_in_env = 42

modimport("scripts/main/myfile.lua")

print(from_myfile)  -- 能看到！因为 myfile 写入了同一个 env
```

```lua
-- scripts/main/myfile.lua
print(my_global_in_env)  -- 42，能看到 modmain 定义的
from_myfile = "hello"     -- 写入 env，modmain 也能看到
AddPrefabPostInit(...)    -- 直接可用，因为在同一个 env 中
```

这就是为什么 modimport 的文件不需要再写 `GLOBAL.setmetatable(env, ...)` ——只要 modmain.lua 开头设置了一次元表，所有 modimport 的文件都自动继承。

#### modimport 的自动补后缀

源码中有一个细节：

```lua
if string.sub(modulename, #modulename-3,#modulename) ~= ".lua" then
    modulename = modulename..".lua"
end
```

如果文件名不以 `.lua` 结尾，会自动补上。所以以下两种写法等价：

```lua
modimport("scripts/main/myfile.lua")
modimport("scripts/main/myfile")
```

#### modimport 没有缓存

这是和 `require` 的一个重要区别。每次调用 `modimport`，文件都会被重新加载和执行。这意味着：

```lua
-- 如果 myfile.lua 中有副作用（如 print），每次 import 都会执行
modimport("scripts/main/myfile.lua")  -- 执行一次
modimport("scripts/main/myfile.lua")  -- 又执行一次！
```

在实际开发中这通常不是问题，因为你不会重复 import 同一个文件。但如果你的文件中有 `AddPrefabPostInit` 等注册操作，重复 import 会导致回调被注册多次。

#### modimport 的错误处理

```lua
local result = kleiloadlua(env.MODROOT..modulename)
if result == nil then
    error("Error in modimport: "..modulename.." not found!")
elseif type(result) == "string" then
    error("Error in modimport: "..ModInfoname(modname)..
          " importing "..modulename.."!\n"..result)
end
```

两种错误情况：
1. `result == nil`：文件不存在
2. `type(result) == "string"`：文件存在但有语法错误（`kleiloadlua` 返回错误信息字符串）

这两种情况都会调用 `error()` 中止 Mod 加载。所以 `modimport` 一个不存在的文件会导致整个 Mod 加载失败。

---

### 3.3.6 require 在 Mod 中的行为——package.path 的秘密

`require` 和 `modimport` 看起来都是"加载文件"，但底层机制完全不同。

#### 搜索路径的构建

游戏启动时，`main.lua` 设置了初始路径：

```lua
-- scripts/main.lua 第 1-2 行
package.path = "scripts\\?.lua;scriptlibs\\?.lua"
```

然后，当加载每个 Mod 时，`LoadMods` 会把 Mod 的 scripts 目录加到路径最前面：

```lua
-- scripts/mods.lua 第 565 行
package.path = MODS_ROOT..mod.modname.."\\scripts\\?.lua;"..package.path
```

假设启用了 3 个 Mod（按 priority 排序后为 A、B、C），加载完成后 `package.path` 变成：

```
mods/C/scripts/?.lua;       ← Mod C 的路径（最后加的在最前面）
mods/B/scripts/?.lua;       ← Mod B 的路径
mods/A/scripts/?.lua;       ← Mod A 的路径
scripts/?.lua;               ← 原版脚本
scriptlibs/?.lua             ← 原版脚本库
```

当你在 Mod C 的 modmain.lua 中写 `require("components/health")` 时，搜索顺序是：
1. `mods/C/scripts/components/health.lua` ← 最先搜索
2. `mods/B/scripts/components/health.lua`
3. `mods/A/scripts/components/health.lua`
4. `scripts/components/health.lua` ← 原版
5. `scriptlibs/components/health.lua`

**第一个找到的文件会被加载。** 这意味着：

1. **如果你的 Mod 有同名文件，会覆盖原版**——这可以用来完全替换某个组件
2. **后加载的 Mod 路径在前面**——priority 越低（越后加载）的 Mod，其 `require` 优先级反而越高
3. **`require` 有缓存**——同名模块只会被加载一次，后续 `require` 直接返回缓存

#### require 的缓存机制

```lua
-- 第一次 require
local health = require("components/health")  -- 加载并执行文件，结果存入 package.loaded

-- 第二次 require（任何地方，包括其他 Mod）
local health2 = require("components/health")  -- 直接返回缓存，不重新执行
print(health == health2)  -- true，完全是同一个对象
```

这有两个重要推论：

1. **如果你的 Mod 用 require 覆盖了原版组件，所有 Mod 和原版代码都会受到影响**——因为它们共享同一个 `package.loaded` 缓存
2. **require 的文件运行在全局环境 `_G` 中**，不在 Mod 的沙盒里——所以 `require` 的文件不能直接使用 `AddPrefabPostInit` 等 Mod API

#### require vs modimport 决策指南

| 你要加载的文件是... | 用 |
|---------------------|-----|
| 自定义组件（`scripts/components/xxx.lua`） | `require`（它本身不调用 Mod API） |
| 自定义工具库/纯逻辑函数 | `require`（如果不需要 Mod API） |
| Mod 的主逻辑代码（调用 PostInit 等） | `modimport` |
| 需要访问 `env` 中变量的配置文件 | `modimport` |
| 原版的组件/工具文件 | `require` |

#### 自定义 loader——kleiloadlua 的角色

标准 Lua 的 `require` 使用 `fopen` 读取文件，但饥荒的资源可能被打包在压缩包中。引擎在 `main.lua` 中注册了一个自定义 loader：

```lua
-- scripts/main.lua 第 133-157 行
local loadfn = function(modulename)
    local errmsg = ""
    local modulepath = string.gsub(modulename, "[%.\\]", "/")
    for path in string.gmatch(package.path, "([^;]+)") do
        local filename = string.gsub(string.gsub(path, "%?", modulepath), "\\", "/")
        local result = kleiloadlua(filename, ...)  -- 用引擎函数加载
        if result then
            return result
        end
        errmsg = errmsg.."\n\tno file '"..filename.."'"
    end
    return errmsg
end
table.insert(package.loaders, 2, loadfn)
```

这个 loader 被插入到 `package.loaders` 的第二位（优先级高于默认 loader），这样所有 `require` 调用都会走引擎的 `kleiloadlua` 而不是标准的文件读取。

`kleiloadlua` 还接收一个可选的 `manifest` 参数，用于从 Steam 创意工坊的 manifest 文件中定位资源——这对玩家来说是透明的，但对理解"为什么我改了文件但游戏没更新"的问题很有帮助（可能是 manifest 缓存了旧版本）。

---

### 3.3.7 多个 Mod 之间的环境关系

每个 Mod 都有自己独立的 `env` 表，但它们共享一些关键的全局对象。理解这种关系可以帮助你避免 Mod 冲突。

#### 独立的部分（互不影响）

- `env` 表本身及其上定义的变量
- `env.postinitfns`（每个 Mod 的 PostInit 注册表）
- `env.modname`、`env.MODROOT`
- `modimport` 的搜索路径

#### 共享的部分（修改会互相影响）

- `TUNING` —— 所有 Mod 的 `env.TUNING` 指向同一个全局 `TUNING` 表
- `STRINGS` —— 需要通过 `GLOBAL.STRINGS` 访问，但所有 Mod 操作的是同一个表
- `GLOBAL`（`_G`）—— 所有 Mod 的 `GLOBAL` 指向同一个全局环境
- `require` 的缓存 —— `package.loaded` 是全局共享的
- `package.path` —— 搜索路径是累加的

这意味着：

```lua
-- Mod A 的 modmain.lua
TUNING.SPEAR_DAMAGE = 100

-- Mod B 的 modmain.lua（加载顺序在 A 之后）
print(TUNING.SPEAR_DAMAGE)  -- 100（能看到 Mod A 的修改）
TUNING.SPEAR_DAMAGE = 200   -- 覆盖了 Mod A 的修改
```

**TUNING 的竞争条件**是 Mod 冲突最常见的原因。如果两个 Mod 都修改同一个 TUNING 值，最后加载的那个"赢"。这就是为什么最好用乘法（相对修改）而不是赋值（绝对修改）：

```lua
-- 不好：直接赋值（会覆盖其他 Mod 的修改）
TUNING.SPEAR_DAMAGE = 50

-- 好：相对修改（和其他 Mod 兼容）
TUNING.SPEAR_DAMAGE = TUNING.SPEAR_DAMAGE * 1.5
```

#### Mod 间通信的方式

如果你的 Mod 需要和其他 Mod 互操作，有几种安全的方式：

**方式一：通过 GLOBAL 注册共享数据**

```lua
-- Mod A（API 提供者）
GLOBAL.rawset(GLOBAL, "MyModAPI", {
    version = "1.0",
    RegisterItem = function(name, data) ... end,
})

-- Mod B（使用者）——需要确保 Mod A 先加载（用 priority）
if GLOBAL.rawget(GLOBAL, "MyModAPI") then
    GLOBAL.MyModAPI.RegisterItem("sword", {damage = 50})
end
```

**方式二：通过 modinfo 的自定义字段**（在 3.1 中已介绍）

**方式三：通过 PrefabPostInit 间接协作**

多个 Mod 都可以注册同一个 prefab 的 PostInit，它们会按 Mod 加载顺序依次执行：

```lua
-- Mod A
AddPrefabPostInit("spear", function(inst) inst.my_mod_a_data = "hello" end)

-- Mod B
AddPrefabPostInit("spear", function(inst)
    if inst.my_mod_a_data then  -- 检查 Mod A 是否已处理
        print("Mod A was here!")
    end
end)
```

---

### 3.3.8 实战案例：追踪一个变量的生命周期

让我们用一个完整的例子，追踪 `AddPrefabPostInit` 这个函数从诞生到被调用的全过程：

```
1. 引擎启动
   └─ main.lua 执行
       └─ require("mods") → 加载 mods.lua

2. ModWrangler:LoadMods() 被调用
   └─ CreateEnvironment("workshop-12345", false)
       ├─ env = { pairs=pairs, ..., GLOBAL=_G, ... }
       ├─ env.env = env
       ├─ env.modimport = function(modulename) ... end
       └─ modutil.InsertPostInitFunctions(env, false, false)
           └─ isworldgen 是 false，不会提前 return
           └─ env.postinitfns.PrefabPostInit = {}
           └─ env.AddPrefabPostInit = function(prefab, fn)
                  table.insert(env.postinitfns.PrefabPostInit[prefab], fn)
              end

3. InitializeModMain("workshop-12345", env, "modmain.lua")
   └─ fn = kleiloadlua("mods/workshop-12345/modmain.lua")
   └─ setfenv(fn, env)  ← 关键：把函数环境设为 env
   └─ xpcall(fn)
       └─ modmain.lua 开始执行
           └─ AddPrefabPostInit("spear", function(inst)
                  inst.components.weapon:SetDamage(100)
              end)
           └─ 此时 env.postinitfns.PrefabPostInit["spear"] = {fn}

4. 游戏世界加载
   └─ 有人生成了一把 spear
   └─ mainfunctions.lua: SpawnPrefabFromSim("spear")
       └─ modprefabinitfns["spear"] 不为空
       └─ 遍历执行 → 调用 fn(inst)
       └─ inst.components.weapon:SetDamage(100) 被执行
       └─ 长矛的伤害变成了 100
```

这个流程展示了 Mod 系统的核心设计理念：**注册-收集-分发**。modmain.lua 只负责"注册"意图（把回调函数存起来），引擎负责在合适的时机"分发"执行。

---

### 3.3.9 常见问题与陷阱

**Q: 为什么 `tonumber("42")` 在 modmain 中报错？**

A: 因为 `tonumber` 不在 `env` 的预置列表中。你需要 `GLOBAL.tonumber("42")`，或者在文件开头 `local tonumber = GLOBAL.tonumber`，或者设置 env 元表指向 GLOBAL。

**Q: 我在 modimport 的文件中定义了 `local function helper()`，modmain 中能用吗？**

A: 不能。`local` 变量只在定义它的文件（chunk）内可见。如果你想在多个文件间共享，要么不加 `local`（写入 env），要么让被导入的文件通过 return 和赋值传递。

```lua
-- myutils.lua
local function private_helper()  -- 只在 myutils.lua 内可见
    return 42
end

shared_helper = function()  -- 写入 env，modmain 能用
    return private_helper()
end
```

**Q: 为什么两个不同的 Mod 同时 require("components/health") 得到的是同一个对象？**

A: 因为 `require` 有全局缓存（`package.loaded`）。第一个 Mod 加载时执行文件并缓存结果，第二个 Mod 直接拿缓存。这也是为什么不建议用 `require` 替换原版文件——你的替换会影响所有 Mod。

**Q: env 元表的 `__index` 到底用 `GLOBAL[k]` 还是 `GLOBAL.rawget(GLOBAL, k)`？**

A: 用 `rawget`。原因是饥荒的 `strict.lua` 给 `_G` 设了 `__index` 元方法，访问未声明的全局变量会报错。`rawget` 绕过元表，如果键不存在就返回 `nil` 而不报错。如果你用 `GLOBAL[k]`，一旦 `k` 是一个游戏中不存在的变量名（比如拼写错误），strict 模式会抛出错误，而且错误信息指向 `strict.lua` 而不是你的代码——非常难调试。

**Q: 我能不能在 modmain 中 `require("main/myutils")`？**

A: 可以，但要注意：`require` 的文件运行在 `_G` 环境中，不在 Mod 的 `env` 中。所以你 require 的文件里不能直接用 `AddPrefabPostInit` 等 Mod API。如果你需要 Mod API，用 `modimport`。如果你的文件是纯工具函数（不依赖 Mod API），可以用 `require`。

**Q: `env.env = env` 这个自引用有什么意义？**

A: 它让 `modimport` 能够通过 `setfenv(result, env.env)` 来设置环境。虽然直接写 `setfenv(result, env)` 效果一样，但 `env.env` 这种写法让被导入的文件内部也可以通过 `env` 这个名字访问自己的环境表——这在某些高级场景下有用（比如动态修改环境或调试）。

---

### 3.3.10 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | Mod 运行在沙盒 `env` 中，不在全局环境 `_G` 中；`env` 中有 Lua 基础库（但不是全部）+ Mod API + `GLOBAL`/`TUNING` 等；用 `GLOBAL.xxx` 访问游戏的全局变量；`modimport` 加载的文件共享同一个 `env`；`require` 的文件运行在全局环境中 |
| **进阶** | 理解变量查找链：`local` → `env` → `env.__index`（如果有）；掌握 env 元表技巧及其读写不对称问题；`require` 的搜索路径是 `package.path`，Mod 的 scripts 目录会被加到前面（可能覆盖原版）；`require` 有缓存而 `modimport` 没有；多个 Mod 共享 `TUNING`、`STRINGS`、`GLOBAL` |
| **老手** | `setfenv(fn, env)` 是整个 Mod 沙盒系统的灵魂（Lua 5.1 特有）；`strict.lua` 给 `_G` 加了 `__index`/`__newindex` 元方法，是 `rawget` 必要性的来源；`package.path` 的累加顺序决定 `require` 优先级（后加载的 Mod 在前）；`kleiloadlua` 是引擎级的文件加载器，替代了标准 `fopen`，支持 manifest 缓存；`InsertPostInitFunctions` 的 `if isworldgen then return end` 分界线划定了两阶段 API 的边界 |

## 3.4 modutil.lua 详解——所有 Mod API 的实际实现

### 本节导读

在前面几节中，我们已经大量使用了 `AddPrefabPostInit`、`AddRecipe2`、`AddAction` 等 Mod API 函数。你知道它们在 modmain.lua 中可以直接调用，也知道它们来自沙盒环境。但你是否好奇过——**这些函数的内部到底做了什么？注册的回调在什么时候、以什么方式被执行？**

本节将揭开 `modutil.lua` 的全貌。这个 1000+ 行的文件是饥荒 Mod 系统的心脏——所有 Mod 能做到的事情，都由它定义。

> **新手**可以重点看 3.4.1-3.4.2，理解「注册—收集—分发」的设计模式，这有助于你正确使用这些 API；**进阶读者**可以深入每个 API 分类，理解参数含义、回调签名和执行时机；**老手**可以关注底层架构和引擎侧的消费代码，理解 `postinitfns` 和 `postinitdata` 的区别以及缓存优化策略。

---

### 3.4.1 modutil.lua 的整体架构

`modutil.lua` 的结构出奇地简单——整个文件只对外暴露一个函数：

```lua
-- scripts/modutil.lua 最后几行
return {
    InsertPostInitFunctions = InsertPostInitFunctions,
}
```

`InsertPostInitFunctions` 是一个约 830 行的巨大函数，它接收一个 `env` 表，然后在上面挂载几十个 API 函数。这些 API 大致分为两类：

**注册型 API**：把回调函数存入 `env.postinitfns` 或 `env.postinitdata`，等待引擎在合适时机执行。

```
modmain.lua 调用 → 存入 postinitfns/postinitdata → 引擎在生命周期节点取出执行
```

**直接型 API**：立即修改全局数据，不涉及延迟执行。

```
modmain.lua 调用 → 直接修改 TUNING / STRINGS / ACTIONS 等全局表
```

两种注册表的区别：

| 注册表 | 存储内容 | 执行方式 |
|--------|---------|---------|
| `postinitfns` | **函数**（回调） | 引擎调用 `GetPostInitFns` 取出，传入目标对象并执行 |
| `postinitdata` | **数据**（State、EventHandler、ActionHandler 实例） | 引擎调用 `GetPostInitData` 取出，直接合并到目标结构中 |

---

### 3.4.2 核心设计模式：注册—收集—分发

几乎所有 PostInit 类的 API 都遵循同一个三阶段模式：

#### 阶段一：注册（Mod 端）

在 modmain.lua 中，你调用 API 注册回调，它被存入 `env.postinitfns`：

```lua
-- 以 AddPrefabPostInit 为例
env.postinitfns.PrefabPostInit = {}
env.AddPrefabPostInit = function(prefab, fn)
    if env.postinitfns.PrefabPostInit[prefab] == nil then
        env.postinitfns.PrefabPostInit[prefab] = {}
    end
    table.insert(env.postinitfns.PrefabPostInit[prefab], fn)
end
```

结构：`postinitfns.PrefabPostInit["spear"] = {fn1, fn2, ...}`

#### 阶段二：收集（引擎端）

当引擎需要执行某类 PostInit 时，调用 `ModWrangler:GetPostInitFns` 遍历所有启用的 Mod，收集注册的回调：

```lua
-- scripts/mods.lua 第 855-875 行
function ModWrangler:GetPostInitFns(type, id)
    local retfns = {}
    for i, modname in ipairs(self.enabledmods) do
        local mod = self:GetMod(modname)
        if mod.postinitfns[type] then
            local modfns = id and mod.postinitfns[type][id] or mod.postinitfns[type]
            if modfns ~= nil then
                for i, modfn in ipairs(modfns) do
                    table.insert(retfns, runmodfn(modfn, mod, ...))
                end
            end
        end
    end
    return retfns
end
```

`runmodfn` 给每个回调包了一层错误处理——如果你的回调报错了，不会导致游戏崩溃，只是在日志中打印错误信息。

#### 阶段三：分发（引擎端）

在游戏的各个生命周期节点，引擎调用收集到的回调：

| PostInit 类型 | 执行点 | 传入参数 |
|---------------|--------|---------|
| `PrefabPostInit` | `mainfunctions.lua` 生成 Prefab 时 | `inst`（新生成的实体） |
| `PrefabPostInitAny` | 同上，对所有 Prefab | `inst` |
| `ComponentPostInit` | `entityscript.lua` AddComponent 时 | `component, inst`（组件实例, 实体） |
| `StategraphPostInit` | `stategraph.lua` StateGraph 构造时 | `self`（StateGraph 实例） |
| `BrainPostInit` | Brain 构造时 | `self`（Brain 实例） |
| `RecipePostInit` | `recipe.lua` Recipe 构造时 | `self`（Recipe 实例） |
| `GamePostInit` | `mods.lua` SetPostEnv 时 | 无 |
| `SimPostInit` | `mods.lua` SimPostInit 时 | `wilson`（参数传递） |

**一个精巧的优化**：对于 `PrefabPostInit`，引擎在 prefab 注册时就把回调列表缓存好了，而不是每次生成实体时都重新收集：

```lua
-- scripts/mainfunctions.lua 第 113 行
modprefabinitfns[prefab.name] = ModManager:GetPostInitFns("PrefabPostInit", prefab.name)
Prefabs[prefab.name] = prefab
```

这样每次 `SpawnPrefab` 时只需要查一个 table，而不是遍历所有 Mod——这是一个典型的空间换时间优化。

---

### 3.4.3 Prefab 修改 API

#### AddPrefabPostInit(prefab, fn)

最常用的 API。在名为 `prefab` 的实体生成后执行 `fn(inst)`。

```lua
-- 让蜘蛛巢多 200 点生命值
AddPrefabPostInit("spiderden", function(inst)
    if GLOBAL.TheWorld.ismastersim then
        if inst.components.health then
            inst.components.health:SetMaxHealth(
                inst.components.health.maxhealth + 200)
        end
    end
end)
```

**重要**：回调在客户端和服务端都会执行。如果你修改的是服务端数据（如组件），需要加 `TheWorld.ismastersim` 检查。

#### AddPrefabPostInitAny(fn)

对**所有** Prefab 生成都执行。引擎实现在 `mainfunctions.lua` 第 383 行：

```lua
for k, prefabpostinitany in pairs(ModManager:GetPostInitFns("PrefabPostInitAny")) do
    prefabpostinitany(inst)
end
```

**性能警告**：这个回调对每一个生成的实体都会触发，包括特效粒子、落叶、烟雾等。如果你的回调有任何复杂逻辑，务必在开头做快速过滤：

```lua
AddPrefabPostInitAny(function(inst)
    if not inst:HasTag("monster") then return end  -- 快速过滤非怪物
    -- 只对怪物执行昂贵的逻辑
end)
```

#### AddPlayerPostInit(fn)

对玩家实体执行。本质是 `AddPrefabPostInitAny` + `HasTag("player")` 过滤：

```lua
-- scripts/modutil.lua 第 586-590 行
env.AddPlayerPostInit = function(fn)
    env.AddPrefabPostInitAny(function(inst)
        if inst and inst:HasTag("player") then fn(inst) end
    end)
end
```

---

### 3.4.4 组件修改 API

#### AddComponentPostInit(component, fn)

在名为 `component` 的组件被添加到任何实体时执行。

引擎在 `entityscript.lua` 的 `AddComponent` 函数中触发：

```lua
-- scripts/entityscript.lua 第 636-640 行
local postinitfns = ModManager:GetPostInitFns("ComponentPostInit", name)
for _, fn in ipairs(postinitfns) do
    fn(loadedcmp, self)  -- 参数：组件实例, 实体
end
```

回调签名是 `fn(component, inst)`——注意是**两个参数**：第一个是组件实例，第二个是拥有该组件的实体。

```lua
-- 让所有有战斗组件的实体攻击时播放自定义音效
AddComponentPostInit("combat", function(self, inst)
    local old_DoAttack = self.DoAttack
    self.DoAttack = function(self, target_override, ...)
        if inst.SoundEmitter then
            inst.SoundEmitter:PlaySound("dontstarve/impacts/impact_flesh_sm")
        end
        return old_DoAttack(self, target_override, ...)
    end
end)
```

**常用技巧：Hook 已有方法**

上面的例子展示了 Mod 开发中最重要的技巧之一——**方法 Hook（钩子）**：

1. 保存原方法引用（`local old_fn = self.Method`）
2. 替换为新方法
3. 在新方法中调用原方法（保持原有行为）
4. 在调用前后添加你的自定义逻辑

这种模式让你在不破坏原有逻辑的前提下扩展行为，是 Mod 兼容性的基石。

---

### 3.4.5 状态图修改 API

状态图（StateGraph）控制着实体的动画和行为状态——攻击动画、走路动画、被击中的硬直等。Mod 可以用三种方式修改状态图：

#### AddStategraphActionHandler(sg, handler)

给状态图添加新的动作处理器——当玩家执行某个 Action 时，进入哪个状态：

```lua
-- scripts/modutil.lua 第 517-524 行
env.postinitdata.StategraphActionHandler = {}
env.AddStategraphActionHandler = function(stategraph, handler)
    if not env.postinitdata.StategraphActionHandler[stategraph] then
        env.postinitdata.StategraphActionHandler[stategraph] = {}
    end
    table.insert(env.postinitdata.StategraphActionHandler[stategraph], handler)
end
```

注意这里存入的是 `postinitdata` 而不是 `postinitfns`——因为 handler 是一个数据对象，引擎会直接合并到状态图中：

```lua
-- scripts/stategraph.lua 第 289-295 行
for k, modhandlers in pairs(ModManager:GetPostInitData("StategraphActionHandler", self.name)) do
    for i, v in ipairs(modhandlers) do
        assert(v:is_a(ActionHandler), "Non-action handler added in mod actionhandler table!")
        self.actionhandlers[v.action] = v  -- 按 action 名覆盖
    end
end
```

使用示例：

```lua
-- 添加自定义动作的状态图处理器
AddStategraphActionHandler("wilson",
    ActionHandler(GLOBAL.ACTIONS.MY_ACTION, "dolongaction"))
-- 客户端也要添加
AddStategraphActionHandler("wilson_client",
    ActionHandler(GLOBAL.ACTIONS.MY_ACTION, "dolongaction"))
```

**为什么要同时添加 wilson 和 wilson_client？** 因为饥荒联机版有两套状态图：`SGwilson.lua`（服务端）和 `SGwilson_client.lua`（客户端）。如果只添加服务端的，客户端执行动作时会找不到对应的状态处理器。

#### AddStategraphState(sg, state)

给状态图添加全新的状态：

```lua
-- 添加一个自定义状态（例如施法动画）
AddStategraphState("wilson", GLOBAL.State{
    name = "my_cast_spell",
    tags = {"doing", "busy"},
    onenter = function(inst)
        inst.AnimState:PlayAnimation("research")
        inst.SoundEmitter:PlaySound("dontstarve/common/use_gem")
    end,
    timeline = {
        TimeEvent(20 * FRAMES, function(inst)
            -- 在动画第 20 帧执行效果
            inst:PerformBufferedAction()
        end),
    },
    events = {
        EventHandler("animover", function(inst)
            inst.sg:GoToState("idle")
        end),
    },
})
```

#### AddStategraphEvent(sg, event)

给状态图添加新的事件处理器。

#### AddStategraphPostInit(sg, fn)

在状态图构造完成后执行自定义逻辑——最灵活但也最复杂的修改方式：

```lua
-- 修改已有状态的 timeline
AddStategraphPostInit("wilson", function(self)
    -- self 就是 StateGraph 实例
    local attack_state = self.states["attack"]
    if attack_state then
        -- 在攻击状态中添加额外的 timeline 事件
        table.insert(attack_state.timeline,
            TimeEvent(5 * FRAMES, function(inst)
                -- 攻击第 5 帧的自定义逻辑
            end))
    end
end)
```

**执行顺序**：引擎先合并 `StategraphActionHandler`/`StategraphEvent`/`StategraphState` 的数据，然后再执行 `StategraphPostInit` 回调。这意味着在 PostInit 中你可以访问到其他 Mod 添加的状态和事件。

---

### 3.4.6 动作系统 API

#### AddAction(id, str, fn) / AddAction(action)

注册一个全新的游戏动作。支持两种调用方式：

```lua
-- 方式一：直接传参数
AddAction("MY_ACTION", "使用技能", function(act)
    if act.doer.components.myskill then
        act.doer.components.myskill:Cast()
        return true
    end
end)

-- 方式二：传一个 Action 对象（更灵活）
local my_action = GLOBAL.Action({distance = 2, rmb = true})
my_action.id = "MY_ACTION"
my_action.str = "使用技能"
my_action.fn = function(act)
    if act.doer.components.myskill then
        act.doer.components.myskill:Cast()
        return true
    end
end
AddAction(my_action)
```

源码实现（`modutil.lua` 第 442-476 行）的关键逻辑：

```lua
action.mod_name = env.modname
ACTIONS[action.id] = action  -- 注册到全局 ACTIONS 表

-- 为每个 Mod 维护独立的 action ID 映射
if ACTION_MOD_IDS[action.mod_name] == nil then
    ACTION_MOD_IDS[action.mod_name] = {}
end
table.insert(ACTION_MOD_IDS[action.mod_name], action.id)
action.code = #ACTION_MOD_IDS[action.mod_name]

STRINGS.ACTIONS[action.id] = action.str  -- 设置显示文本
```

每个 Mod 的动作有独立的编号系统（`action.code`），避免不同 Mod 添加的动作 ID 冲突。

#### AddComponentAction(actiontype, component, fn)

定义「在什么条件下，对什么组件，可以执行什么动作」：

```lua
-- 当右键点击一个拥有 myskill 组件的实体时，显示"使用技能"选项
AddComponentAction("SCENE", "myskill", function(inst, doer, actions, right)
    if right and doer:HasTag("player") then
        table.insert(actions, GLOBAL.ACTIONS.MY_ACTION)
    end
end)
```

`actiontype` 有以下几种：

| 类型 | 含义 | 典型用途 |
|------|------|---------|
| `"SCENE"` | 场景中的实体互动 | 对建筑、NPC、物品使用动作 |
| `"USEITEM"` | 使用物品栏中的物品 | 对着目标使用物品 |
| `"POINT"` | 对着地面某个点 | 放置物品、使用法杖 |
| `"EQUIPPED"` | 装备物品后的动作 | 装备武器后的特殊攻击 |
| `"INVENTORY"` | 物品栏内操作 | 右键物品的直接使用 |

---

### 3.4.7 合成配方 API

#### AddRecipe2(name, ingredients, tech, config, filters)

添加新的合成配方。这是 `AddRecipe` 的新版本（旧版已废弃）：

```lua
AddRecipe2("myweapon",
    -- 材料
    {Ingredient("twigs", 2), Ingredient("flint", 3)},
    -- 科技等级
    TECH.SCIENCE_ONE,  -- 需要一本科学机器
    -- 配置
    {
        atlas = "images/inventoryimages/myweapon.xml",
        image = "myweapon.tex",
        placer = "myweapon_placer",  -- 如果是建筑物，需要放置器
    },
    -- 过滤器（可选）
    {"WEAPONS", "TOOLS"}
)
```

源码实现（`modutil.lua` 第 732-756 行）的关键逻辑：

```lua
env.AddRecipe2 = function(name, ingredients, tech, config, filters)
    require("recipe")
    mod_protect_Recipe = false  -- 临时解除配方保护
    local rec = Recipe2(name, ingredients, tech, config)

    if not rec.is_deconstruction_recipe then
        if config ~= nil and config.nounlock then
            env.AddRecipeToFilter(name, CRAFTING_FILTERS.CRAFTING_STATION.name)
        else
            env.AddRecipeToFilter(name, CRAFTING_FILTERS.MODS.name)
        end
        if filters ~= nil then
            for _, filter_name in ipairs(filters) do
                env.AddRecipeToFilter(name, filter_name)
            end
        end
    end

    mod_protect_Recipe = true  -- 恢复保护
    rec:SetModRPCID()  -- 设置网络 RPC ID
    return rec
end
```

注意 `mod_protect_Recipe` 的开关——引擎用它防止 Mod 意外创建非法配方。`SetModRPCID()` 为配方分配一个网络 ID，用于客户端-服务端的合成请求通信。

默认情况下，Mod 添加的配方会自动出现在「MODS」分类中（除非你指定了 `nounlock`）。你也可以通过 `filters` 参数把它添加到原版的分类中。

#### AddIngredientValues(names, tags, cancook, candry)

注册新食材到烹饪系统：

```lua
-- 注册一种新食材
AddIngredientValues(
    {"myfruit"},           -- 食材的 prefab 名（数组形式，可以一次注册多个）
    {fruit = 1, veggie = 0.5},  -- 食材标签和权重
    true,                  -- 是否可以烹饪（生成 myfruit_cooked）
    true                   -- 是否可以晾干（生成 myfruit_dried）
)
```

全局实现（`scripts/cooking.lua` 第 46-72 行）会为你的食材创建完整的条目，包括烹饪版和干燥版的标签。

#### AddCookerRecipe(cooker, recipe)

添加新的烹饪食谱：

```lua
AddCookerRecipe("cookpot", {
    name = "mysoup",
    test = function(cooker, names, tags)
        return tags.veggie and tags.veggie >= 2
              and names.myfruit and names.myfruit >= 1
    end,
    priority = 10,
    weight = 1,
    foodtype = GLOBAL.FOODTYPE.VEGGIE,
    health = GLOBAL.TUNING.HEALING_LARGE,
    hunger = GLOBAL.TUNING.CALORIES_LARGE,
    sanity = GLOBAL.TUNING.SANITY_MED,
    perishtime = GLOBAL.TUNING.PERISH_MED,
    cooktime = 2,
})
```

---

### 3.4.8 角色 API

#### AddModCharacter(name, gender, modes)

注册一个新的可选角色：

```lua
AddModCharacter("mycharacter", "FEMALE")
```

源码实现（`modutil.lua` 第 73-90 行）做了三件事：

```lua
local function AddModCharacter(name, gender, modes)
    table.insert(MODCHARACTERLIST, name)         -- 1. 加入 Mod 角色列表
    if not DoesCharacterExistInGendersTable(name) then
        gender = (gender or "NEUTRAL"):upper()
        if not CHARACTER_GENDERS[gender] then
            CHARACTER_GENDERS[gender] = {}
        end
        table.insert(CHARACTER_GENDERS[gender], name)  -- 2. 注册性别
    end
    MODCHARACTERMODES[name] = modes               -- 3. 注册游戏模式限制
end
```

使用 `AddModCharacter` 前，你通常还需要准备：

```lua
-- 1. PrefabFiles 中声明角色 prefab
PrefabFiles = {"mycharacter", "mycharacter_none"}

-- 2. 设置角色的台词
STRINGS.CHARACTERS.MYCHARACTER = require("speech_mycharacter")

-- 3. 设置初始物品
TUNING.GAMEMODE_STARTING_ITEMS.DEFAULT.MYCHARACTER = {"flint", "twigs"}

-- 4. 注册角色
AddModCharacter("mycharacter", "FEMALE")
```

---

### 3.4.9 网络通信 API（RPC）

在联机版中，客户端和服务端是分开运行的。当玩家按下技能键时，客户端需要告诉服务端"执行这个技能"——这就需要 **RPC（Remote Procedure Call）**。

#### AddModRPCHandler(namespace, name, fn)

在服务端注册一个 RPC 处理器：

```lua
AddModRPCHandler("mymod", "cast_skill", function(player, target)
    if player.components.myskill then
        player.components.myskill:Cast(target)
    end
end)
```

全局实现（`scripts/networkclientrpc.lua` 第 1663-1678 行）：

```lua
function AddModRPCHandler(namespace, name, fn)
    if MOD_RPC[namespace] == nil then
        MOD_RPC[namespace] = {}
        MOD_RPC_HANDLERS[namespace] = {}
    end
    table.insert(MOD_RPC_HANDLERS[namespace], fn)
    MOD_RPC[namespace][name] = {
        namespace = namespace,
        id = #MOD_RPC_HANDLERS[namespace]
    }
end
```

#### SendModRPCToServer(id_table, ...)

从客户端发送 RPC 到服务端：

```lua
-- 在客户端代码中（如按键回调）
SendModRPCToServer(GetModRPC("mymod", "cast_skill"), target)
```

典型的完整流程：

```lua
-- modmain.lua 中注册 RPC
AddModRPCHandler("mymod", "cast_skill", function(player, target)
    -- 这个函数在服务端执行
    if player.components.myskill then
        player.components.myskill:Cast(target)
    end
end)

-- 在客户端的按键回调中发送
local KEY_SKILL = GLOBAL.rawget(GLOBAL, GetModConfigData("skill_key"))
GLOBAL.TheInput:AddKeyDownHandler(KEY_SKILL, function()
    if GLOBAL.ThePlayer then
        SendModRPCToServer(GetModRPC("mymod", "cast_skill"))
    end
end)
```

还有对应的 `AddClientModRPCHandler`（服务端到客户端）和 `AddShardModRPCHandler`（分片间通信）。

---

### 3.4.10 高级 API：AddClassPostConstruct

这是最强大也最危险的 Mod API——它可以修改游戏中**任何通过 `require` 加载的 Class** 的构造过程。

#### 实现原理

```lua
-- scripts/modutil.lua 第 153-165 行
local function DoAddClassPostConstruct(classdef, postfn)
    local constructor = classdef._ctor
    classdef._ctor = function(self, ...)
        constructor(self, ...)    -- 先执行原构造函数
        postfn(self, ...)         -- 再执行你的后处理
    end
end

local function AddClassPostConstruct(package, postfn)
    local classdef = require(package)
    assert(type(classdef) == "table",
        "Class file path '"..package.."' doesn't seem to return a valid class.")
    DoAddClassPostConstruct(classdef, postfn)
end
```

它做了一件简单但深刻的事情：**替换类的构造函数**。原来的 `_ctor` 被保存，新的 `_ctor` 先调用原版构造函数，再调用你的 `postfn`。这样每次创建该类的实例时，你的代码都会在构造完成后执行。

#### 实用示例

**修改 HUD 界面**：

```lua
AddClassPostConstruct("screens/playerhud", function(self)
    -- self 是 PlayerHud 实例
    -- 在 HUD 上添加一个自定义 widget
    self.my_widget = self:AddChild(MyCustomWidget(self.owner))
end)
```

**修改物品栏**：

```lua
AddClassPostConstruct("widgets/inventorybar", function(self)
    -- 修改物品栏的布局
end)
```

**修改地图**：

```lua
AddClassPostConstruct("widgets/mapwidget", function(self)
    -- 添加地图标记功能
end)
```

**注意**：`AddClassPostConstruct` 的 `package` 参数必须是 `require` 可以加载的路径，不需要前缀 `scripts/` 也不需要后缀 `.lua`：

```lua
-- 正确
AddClassPostConstruct("screens/playerhud", fn)
AddClassPostConstruct("widgets/inventorybar", fn)
AddClassPostConstruct("components/combat", fn)  -- 也可以修改组件！

-- 错误
AddClassPostConstruct("scripts/screens/playerhud", fn)
AddClassPostConstruct("screens/playerhud.lua", fn)
```

#### AddClassPostConstruct vs AddComponentPostInit

对于组件，两者都能修改，但有区别：

| 特性 | AddComponentPostInit | AddClassPostConstruct |
|------|---------------------|----------------------|
| 触发时机 | 每次 `inst:AddComponent()` 时 | 每次 `Class()` 构造时 |
| 回调参数 | `(component, inst)` | `(self, ...)` |
| 修改的是 | 组件**实例** | 类的**构造函数** |
| 适用范围 | 只能修改组件 | 任何 Class（组件、界面、工具等） |

---

### 3.4.11 其他重要 API 速览

| API | 作用 | 典型用途 |
|-----|------|---------|
| `AddReplicableComponent(name)` | 注册可网络复制的组件 | 自定义组件需要客户端同步时 |
| `AddMinimapAtlas(atlas)` | 注册小地图图标集 | 自定义建筑/实体的地图图标 |
| `RegisterInventoryItemAtlas(atlas, tex)` | 注册物品栏图标 | 自定义物品在背包中的图标 |
| `LoadPOFile(path)` | 加载翻译文件 | Mod 的多语言支持 |
| `RemapSoundEvent(name, new_name)` | 重映射音效事件 | 替换游戏中某个音效 |
| `AddPopup(id, mod_name)` | 注册弹窗 | 自定义 UI 弹窗 |
| `AddRecipeTab(...)` | 添加合成栏标签 | 自定义合成分类 |
| `AddUserCommand(name, data)` | 添加控制台命令 | 自定义管理员命令 |
| `ExcludeClothingSymbolForModCharacter(symbol)` | 排除服装符号 | 防止自定义角色穿戴某些衣服出错 |
| `RegisterSkilltreeBGForCharacter(character, atlas, tex)` | 注册技能树背景 | 自定义角色的技能树界面 |

---

### 3.4.12 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | 所有 Mod API 本质上是「把你的函数存起来，等引擎在合适时机调用」；最常用的 5 个是 `AddPrefabPostInit`、`AddComponentPostInit`、`AddPlayerPostInit`、`AddRecipe2`、`AddAction`；回调中注意检查 `TheWorld.ismastersim` |
| **进阶** | 理解 `postinitfns`（函数回调）和 `postinitdata`（数据合并）的区别；掌握方法 Hook 技巧（保存原方法 → 替换 → 调用原方法 + 自定义逻辑）；状态图修改需要同时处理 `wilson` 和 `wilson_client`；RPC 是客户端-服务端通信的标准方式 |
| **老手** | `GetPostInitFns` 遍历所有 Mod 按启用顺序收集回调，`runmodfn` 包装了错误处理；`PrefabPostInit` 在 prefab 注册时就缓存了回调列表（空间换时间）；`AddClassPostConstruct` 通过替换 `_ctor` 实现——原构造函数被闭包捕获，新构造函数先调原版再调扩展；`mod_protect_Recipe` 控制配方创建的安全性；`ACTION_MOD_IDS` 为每个 Mod 维护独立的动作编号避免冲突 |

## 3.5 Mod API 速览：AddPrefabPostInit、AddComponentPostInit 等

### 本节导读

3.4 节从源码角度剖析了每个 API 的内部实现。本节换一个角度——站在**使用者**的立场，给你一份实战参考手册。对于每个常用 API，我们提供：回调签名、参数说明、真实 Mod 中的使用范例、常见组合模式，以及「什么时候该用 / 不该用」的决策指南。

> **新手**可以把本节当做「速查表」——遇到需求时翻到对应 API，照着范例改就行；**进阶读者**可以关注「组合模式」部分，学习如何把多个 API 搭配起来完成复杂功能；**老手**可以直接跳到最后的「决策矩阵」，快速定位最佳 API 组合。

---

### 3.5.1 实体修改三兄弟

这三个 API 是使用频率最高的，它们分别针对不同粒度的修改目标。

#### AddPrefabPostInit(prefab_name, fn)

| 属性 | 值 |
|------|----|
| **回调签名** | `fn(inst)` |
| **参数** | `inst` —— 新生成的实体实例 |
| **执行时机** | 每次该 prefab 的实体生成后 |
| **客户端/服务端** | 两端都执行 |

**范例一：最基础——修改单个 Prefab 的属性**

```lua
AddPrefabPostInit("spear", function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end
    if inst.components.weapon then
        inst.components.weapon:SetDamage(50)
    end
end)
```

**范例二：同一个回调注册到多个 Prefab**

真实 Mod 中很常见——把函数先定义好，再注册到多个同类 prefab：

```lua
-- 勋章 Mod 的做法：同一个绘画逻辑注册到所有画板类 prefab
local function draw(inst) ... end

AddPrefabPostInit("minisign", draw)
AddPrefabPostInit("minisign_drawn", draw)
AddPrefabPostInit("decor_pictureframe", draw)
```

**范例三：批量注册——表驱动**

```lua
local prefab_hooks = {
    spear = function(inst) ... end,
    axe = function(inst) ... end,
    pickaxe = function(inst) ... end,
}
for name, fn in pairs(prefab_hooks) do
    AddPrefabPostInit(name, fn)
end
```

**范例四：挂在 "world" 上做全局初始化**

`"world"` 是一个特殊的 prefab——整个游戏世界就是一个实体。挂在它上面可以做一次性的全局设置：

```lua
AddPrefabPostInit("world", function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end
    -- 注册世界级事件监听
    inst:ListenForEvent("ms_playerjoined", function(world, player)
        print(player.name .. " 加入了游戏")
    end)
end)
```

**范例五：挂在 "player_classified" 上做网络同步**

`player_classified` 是每个玩家的网络同步实体，常用于客户端-服务端数据桥接：

```lua
AddPrefabPostInit("player_classified", function(inst)
    inst.my_custom_net_var = GLOBAL.net_bool(inst.GUID, "mymod.custom_var")
end)
```

#### AddPrefabPostInitAny(fn)

| 属性 | 值 |
|------|----|
| **回调签名** | `fn(inst)` |
| **执行时机** | 每个 Prefab 生成后（所有类型） |
| **性能影响** | 高——每帧可能触发多次 |

**黄金法则：开头必须有快速过滤**

```lua
AddPrefabPostInitAny(function(inst)
    -- 第一行就过滤，不符合的立即 return
    if not inst:HasTag("monster") then return end
    -- 只对怪物执行
    inst:AddTag("my_mod_tracked")
end)
```

#### AddPlayerPostInit(fn)

| 属性 | 值 |
|------|----|
| **回调签名** | `fn(inst)` |
| **执行时机** | 玩家实体生成后 |
| **本质** | `AddPrefabPostInitAny` + `HasTag("player")` 过滤 |

**范例：给玩家添加自定义组件**

```lua
AddPlayerPostInit(function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end
    inst:AddComponent("myskill")
    inst.components.myskill:SetLevel(1)
end)
```

**决策指南：三兄弟怎么选？**

| 你想做什么 | 用哪个 |
|-----------|--------|
| 改一种具体物品（如长矛） | `AddPrefabPostInit("spear", ...)` |
| 改所有武器 | `AddComponentPostInit("weapon", ...)` |
| 改所有怪物 | `AddPrefabPostInitAny(...)` + `HasTag` 过滤 |
| 改所有玩家 | `AddPlayerPostInit(...)` |
| 做世界级初始化 | `AddPrefabPostInit("world", ...)` |
| 做网络同步变量 | `AddPrefabPostInit("player_classified", ...)` |

---

### 3.5.2 组件修改

#### AddComponentPostInit(component_name, fn)

| 属性 | 值 |
|------|----|
| **回调签名** | `fn(component, inst)` |
| **参数** | `component` —— 组件实例，`inst` —— 拥有者实体 |
| **执行时机** | 每次 `inst:AddComponent(name)` 被调用后 |

**范例一：方法 Hook——最核心的修改技巧**

```lua
AddComponentPostInit("combat", function(self, inst)
    -- 1. 保存原方法
    local old_GetAttacked = self.GetAttacked
    -- 2. 替换为新方法
    self.GetAttacked = function(self, attacker, damage, ...)
        -- 3. 在原方法之前添加逻辑
        if inst:HasTag("my_shield") then
            damage = damage * 0.5  -- 减半伤害
        end
        -- 4. 调用原方法
        return old_GetAttacked(self, attacker, damage, ...)
    end
end)
```

**范例二：给组件添加新字段**

```lua
AddComponentPostInit("hunger", function(self, inst)
    -- 神话书说：让月饼不掉饥饿
    self.my_no_hunger_items = self.my_no_hunger_items or {}
end)
```

**范例三：替换组件方法（完全覆盖）**

```lua
AddComponentPostInit("workable", function(self, inst)
    -- 勋章 Mod：完全替换工作逻辑
    self.MyCustomWork = function(self, worker, numworks)
        -- 自定义的工作逻辑
    end
end)
```

---

### 3.5.3 动作系统——完整工作流

添加一个自定义动作需要**四步联动**：

#### 第一步：创建 Action

```lua
local MY_ACTION = GLOBAL.Action({distance = 2})
MY_ACTION.id = "MY_ACTION"
MY_ACTION.str = "使用技能"
MY_ACTION.fn = function(act)
    if act.doer and act.doer.components.myskill then
        act.doer.components.myskill:Cast(act.target)
        return true
    end
end
AddAction(MY_ACTION)
```

#### 第二步：定义何时显示动作（AddComponentAction）

```lua
AddComponentAction("SCENE", "myskill_target", function(inst, doer, actions, right)
    if right and doer:HasTag("myskill_user") then
        table.insert(actions, GLOBAL.ACTIONS.MY_ACTION)
    end
end)
```

五种 `actiontype` 速查：

| 类型 | 触发条件 | 参数签名 |
|------|---------|---------|
| `"SCENE"` | 鼠标悬浮在场景实体上 | `(inst, doer, actions, right)` |
| `"USEITEM"` | 用物品栏物品对目标使用 | `(inst, doer, target, actions, right)` |
| `"POINT"` | 对地面某点操作 | `(inst, doer, pos, actions, right)` |
| `"EQUIPPED"` | 装备物品后操作 | `(inst, doer, target, actions, right)` |
| `"INVENTORY"` | 物品栏内右键 | `(inst, doer, actions, right)` |

#### 第三步：绑定状态图动画

```lua
-- 服务端状态图
AddStategraphActionHandler("wilson",
    GLOBAL.ActionHandler(GLOBAL.ACTIONS.MY_ACTION, "dolongaction"))

-- 客户端状态图——联机版必须同时添加！
AddStategraphActionHandler("wilson_client",
    GLOBAL.ActionHandler(GLOBAL.ACTIONS.MY_ACTION, "dolongaction"))
```

**常见的状态名**：

| 状态名 | 动画效果 |
|--------|---------|
| `"doshortaction"` | 快速单次动作（如拾取） |
| `"dolongaction"` | 长按进度条动作（如挖矿） |
| `"give"` | 递出物品的动画 |
| `"attack"` | 攻击动画 |
| `"chop_start"` | 砍伐动画 |

也可以自定义状态（见 3.4.5 的 `AddStategraphState` 范例）。

#### 第四步（可选）：通过 RPC 处理网络通信

如果动作需要客户端主动触发（如按键技能），需要 RPC：

```lua
AddModRPCHandler("mymod", "cast_skill", function(player, target)
    if player.components.myskill then
        player.components.myskill:Cast(target)
    end
end)
```

**神话书说的做法**——同一命名空间注册多个 RPC 通道：

```lua
AddModRPCHandler("mymod_rpc", "action.1", function(player, inst, data) ... end)
AddModRPCHandler("mymod_rpc", "action.2", function(player, inst, data) ... end)
AddModRPCHandler("mymod_rpc", "action.3", function(player, inst, data) ... end)
```

---

### 3.5.4 合成系统——配方速查

#### 添加新配方的标准流程

```lua
-- 1. 在 modmain 中注册物品图标
RegisterInventoryItemAtlas("images/inventoryimages/myweapon.xml", "myweapon.tex")

-- 2. 添加物品名称和描述
GLOBAL.STRINGS.NAMES.MYWEAPON = "我的武器"
GLOBAL.STRINGS.RECIPE_DESC.MYWEAPON = "一把威力巨大的武器"
GLOBAL.STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON = "看起来很厉害！"

-- 3. 添加合成配方
AddRecipe2("myweapon",
    {Ingredient("twigs", 3), Ingredient("flint", 2), Ingredient("rope", 1)},
    TECH.SCIENCE_TWO,  -- 需要炼金引擎
    {
        atlas = "images/inventoryimages/myweapon.xml",
        image = "myweapon.tex",
    }
)
```

**科技等级速查**：

| 常量 | 含义 | 需要的制作站 |
|------|------|------------|
| `TECH.NONE` | 无需科技 | 徒手 |
| `TECH.SCIENCE_ONE` | 科学 1 级 | 科学机器 |
| `TECH.SCIENCE_TWO` | 科学 2 级 | 炼金引擎 |
| `TECH.MAGIC_TWO` | 魔法 2 级 | 暗影操纵器 |
| `TECH.ANCIENT_TWO` | 远古 2 级 | 远古伪科学站 |
| `TECH.CELESTIAL_ONE` | 天体 1 级 | 天体祭坛 |
| `TECH.SHADOW_TWO` | 暗影 2 级 | 暗影秘典 |
| `TECH.CARTOGRAPHY_TWO` | 制图 2 级 | 制图桌 |
| `TECH.SEAFARING_TWO` | 航海 2 级 | 锡制渔具箱 |

**添加烹饪食谱的流程**：

```lua
-- 1. 注册食材
AddIngredientValues({"myfruit"}, {fruit = 1}, true, false)

-- 2. 添加烹饪配方
AddCookerRecipe("cookpot", {
    name = "myjam",
    test = function(cooker, names, tags)
        return names.myfruit and names.myfruit >= 2
    end,
    priority = 1,
    weight = 1,
    foodtype = GLOBAL.FOODTYPE.VEGGIE,
    health = GLOBAL.TUNING.HEALING_MED,
    hunger = GLOBAL.TUNING.CALORIES_LARGE,
    sanity = GLOBAL.TUNING.SANITY_SMALL,
    perishtime = GLOBAL.TUNING.PERISH_MED,
    cooktime = 2,
})
```

---

### 3.5.5 界面修改——AddClassPostConstruct

#### 常见的可修改 Class 路径

| 路径 | 类 | 典型修改 |
|------|----|---------|
| `"screens/playerhud"` | PlayerHud | 添加自定义 HUD 元素 |
| `"widgets/controls"` | Controls | 修改控制面板布局 |
| `"widgets/inventorybar"` | InventoryBar | 修改物品栏 |
| `"widgets/hoverer"` | Hoverer | 修改鼠标悬浮提示 |
| `"widgets/itemtile"` | ItemTile | 修改物品格显示 |
| `"widgets/containerwidget"` | ContainerWidget | 修改容器界面 |
| `"components/builder_replica"` | BuilderReplica | 修改合成系统客户端逻辑 |
| `"cameras/followcamera"` | FollowCamera | 修改相机行为 |
| `"components/playercontroller"` | PlayerController | 修改玩家输入处理 |

**范例：在 HUD 上添加自定义元素**

```lua
AddClassPostConstruct("screens/playerhud", function(self)
    -- self 是 PlayerHud 实例
    -- self.owner 是拥有这个 HUD 的玩家
    local MyWidget = require("widgets/mywidget")
    self.my_display = self:AddChild(MyWidget(self.owner))
    self.my_display:SetPosition(0, -100)
end)
```

**范例：修改悬浮提示信息**

```lua
AddClassPostConstruct("widgets/hoverer", function(self)
    local old_SetString = self.text.SetString
    self.text.SetString = function(text_widget, str)
        -- 在悬浮提示后面添加额外信息
        str = str .. "\n[Mod Info: 额外提示]"
        old_SetString(text_widget, str)
    end
end)
```

---

### 3.5.6 角色 Mod 完整检查清单

创建一个角色 Mod 需要用到多个 API 的组合。以下是完整的检查清单：

```lua
-- ==========================================
-- 1. 环境配置
-- ==========================================
GLOBAL.setmetatable(env, {__index = function(t, k) return GLOBAL.rawget(GLOBAL, k) end})

-- ==========================================
-- 2. Prefab 文件
-- ==========================================
PrefabFiles = {"mychar", "mychar_none"}

-- ==========================================
-- 3. 资源
-- ==========================================
Assets = {
    Asset("IMAGE", "images/saveslot_portraits/mychar.tex"),
    Asset("ATLAS", "images/saveslot_portraits/mychar.xml"),
    Asset("IMAGE", "bigportraits/mychar.tex"),
    Asset("ATLAS", "bigportraits/mychar.xml"),
}
AddMinimapAtlas("images/map_icons/mychar.xml")

-- ==========================================
-- 4. 文字
-- ==========================================
STRINGS.CHARACTER_TITLES.mychar = "角色头衔"
STRINGS.CHARACTER_NAMES.mychar = "角色名"
STRINGS.CHARACTER_DESCRIPTIONS.mychar = "角色描述"
STRINGS.CHARACTER_QUOTES.mychar = "\"角色引言\""
STRINGS.CHARACTERS.MYCHAR = require("speech_mychar")

-- ==========================================
-- 5. 数值
-- ==========================================
TUNING.MYCHAR_HEALTH = 200
TUNING.MYCHAR_HUNGER = 150
TUNING.MYCHAR_SANITY = 120

-- ==========================================
-- 6. 初始物品
-- ==========================================
TUNING.GAMEMODE_STARTING_ITEMS.DEFAULT.MYCHAR = {"flint", "twigs", "twigs"}
TUNING.STARTING_ITEM_IMAGE_OVERRIDE.flint = {atlas = "images/inventoryimages.xml", image = "flint.tex"}

-- ==========================================
-- 7. 注册角色
-- ==========================================
AddModCharacter("mychar", "FEMALE")

-- ==========================================
-- 8. 自定义技能（可选）
-- ==========================================
AddModRPCHandler("mychar_rpc", "cast_skill", function(player)
    -- 服务端处理
end)

-- 9. 自定义动作（可选）
-- 10. 合成配方（可选）
-- 11. 皮肤（可选）
```

---

### 3.5.7 API 决策矩阵

当你面对一个修改需求时，用这个矩阵快速定位应该用哪个 API：

| 我想... | 首选 API | 备选 |
|---------|---------|------|
| 改一种物品的属性 | `AddPrefabPostInit` | |
| 改所有同类物品（如所有武器） | `AddComponentPostInit` | `AddPrefabPostInitAny` |
| 给玩家加能力 | `AddPlayerPostInit` | |
| 添加新物品/实体 | `PrefabFiles` + prefab 文件 | |
| 添加合成配方 | `AddRecipe2` | |
| 添加新动作 | `AddAction` + `AddComponentAction` + `AddStategraphActionHandler` | |
| 修改动画/状态 | `AddStategraphPostInit` 或 `AddStategraphState` | |
| 修改 AI 行为 | `AddBrainPostInit` | |
| 修改 UI/HUD | `AddClassPostConstruct` | |
| 客户端→服务端通信 | `AddModRPCHandler` + `SendModRPCToServer` | |
| 添加新角色 | `AddModCharacter` | |
| 修改世界生成 | `AddLevelPreInit` / `AddTaskPreInit`（在 modworldgenmain.lua 中） | |
| 添加烹饪食谱 | `AddCookerRecipe` + `AddIngredientValues` | |
| 修改已有配方 | `AddRecipePostInit` | |
| 注册小地图图标 | `AddMinimapAtlas` | |
| 注册物品栏图标 | `RegisterInventoryItemAtlas` | |

---

### 3.5.8 常见的「ismastersim 检查」模式

联机版中，很多修改只应在服务端执行。以下是标准模式：

```lua
AddPrefabPostInit("spear", function(inst)
    -- 客户端也能执行的代码放这里（如视觉效果、标签）
    inst:AddTag("my_custom_tag")

    -- 服务端专用的代码
    if not GLOBAL.TheWorld.ismastersim then return end

    -- 组件修改必须在服务端
    if inst.components.weapon then
        inst.components.weapon:SetDamage(100)
    end
end)
```

**什么需要 ismastersim 检查？**
- 所有组件（`inst.components.xxx`）的修改
- 数据的持久化操作
- 生成新实体（`SpawnPrefab`）

**什么不需要？**
- 添加/删除 Tag（客户端也需要用来做 UI 判断）
- 视觉效果（动画、粒子）
- 网络变量（`net_*`）的监听

---

### 3.5.9 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | 把本节当速查手册用；最常用的 5 个 API 是 `AddPrefabPostInit`、`AddPlayerPostInit`、`AddComponentPostInit`、`AddRecipe2`、`AddAction`；所有组件修改都要加 `ismastersim` 检查 |
| **进阶** | 掌握「动作四步联动」工作流；用表驱动和函数引用实现批量注册；理解 `AddClassPostConstruct` 是修改 UI 的标准手段；RPC 是客户端→服务端通信的唯一安全方式 |
| **老手** | 用「决策矩阵」秒定 API 组合；善用 `"world"` 和 `"player_classified"` 做全局/网络初始化；用命名空间管理多路 RPC；关注 `AddPrefabPostInitAny` 的性能影响——开头必须快速过滤 |


## 3.6 客户端 Mod vs 服务端 Mod——client_only_mod 与 all_clients_require_mod

### 本节导读

饥荒联机版（DST）是一个**客户端-服务端分离**的游戏——即使你自己开服自己玩，游戏内部也运行着一个服务端和一个客户端。这种架构直接影响了 Mod 的开发方式：有些代码只能在服务端跑，有些只能在客户端跑，有些必须两边都跑。

如果你不理解这个架构，就会遇到各种「我的修改没生效」「其他玩家看不到我的效果」「客户端崩溃但服务端正常」的问题。本节将从游戏架构讲起，帮你彻底搞清楚客户端和服务端的关系。

> **新手**可以重点看 3.6.1-3.6.3，掌握 `ismastersim` 检查和 Mod 类型的区分；**进阶读者**可以深入 3.6.4-3.6.5，理解 Replica 组件和网络同步机制；**老手**可以关注 3.6.6，理解 `net_*` 变量和自定义网络同步的实现。

---

### 3.6.1 DST 的客户端-服务端架构

在单机版饥荒中，所有代码都在同一个进程里运行。但联机版引入了分离架构：

```
┌──────────────────┐         ┌──────────────────┐
│     服务端        │         │     客户端        │
│  (Master Sim)     │◄──网络──►│  (Client)        │
│                  │         │                  │
│ • 游戏逻辑权威    │         │ • 渲染和显示       │
│ • 组件数据        │         │ • 玩家输入         │
│ • AI 行为         │         │ • UI/HUD          │
│ • 物品/战斗/血量   │         │ • 动画预测         │
│ • 世界状态        │         │ • Replica 组件     │
└──────────────────┘         └──────────────────┘
```

**关键概念**：
- **服务端（Master Simulation）** 是「权威」——所有游戏逻辑的最终裁判。一个实体的血量、伤害、物品栏等**只存在于服务端**
- **客户端** 只负责显示和输入。它通过 Replica 组件获取服务端数据的**副本**，用于 UI 显示
- **即使你自己开服自己玩**，你的电脑同时运行着服务端和客户端——`TheWorld.ismastersim` 用来区分当前代码运行在哪一端

`TheWorld.ismastersim` 的值来自引擎：

```lua
-- scripts/prefabs/world.lua 第 421-427 行
TheWorld = inst
inst.ismastersim = TheNet:GetIsMasterSimulation()
inst.ismastershard = inst.ismastersim and not TheShard:IsSecondary()
```

---

### 3.6.2 Mod 的三种网络模式

在 modinfo.lua 中，`client_only_mod` 和 `all_clients_require_mod` 决定了 Mod 的网络行为：

| `client_only_mod` | `all_clients_require_mod` | 模式 | 说明 |
|:-:|:-:|------|------|
| `false` | `true` | **服务端 Mod（全员必装）** | 最常见——服务端运行逻辑，所有客户端也必须安装 |
| `false` | `false` | **服务端 Mod（仅服务端）** | 纯服务端逻辑，客户端不需要安装 |
| `true` | `false` | **纯客户端 Mod** | 只在客户端运行，和服务器无关 |
| `true` | `true` | **矛盾！** | 引擎会打印警告，不要这样做 |

#### 模式一：服务端 Mod（全员必装）——最常见

```lua
-- modinfo.lua
all_clients_require_mod = true
client_only_mod = false
```

**适用场景**：添加新物品、新角色、修改游戏机制——几乎所有需要「改变游戏内容」的 Mod 都用这个模式。

**为什么客户端也要装？** 因为客户端需要知道新物品的贴图、名称、动画等信息。如果客户端没装 Mod，它不知道服务端新增的 `"myweapon"` 该显示成什么样。

当服务器启用了 `all_clients_require_mod = true` 的 Mod 时，引擎会：
1. 在房间信息中通告该 Mod（`scripts/mods.lua` 第 147-158 行）
2. 加入的客户端如果没有该 Mod，会自动从创意工坊下载
3. 定期检查所有客户端的 Mod 版本是否匹配（`StartVersionChecking`，第 934-949 行）

#### 模式二：服务端 Mod（仅服务端）

```lua
-- modinfo.lua
all_clients_require_mod = false
client_only_mod = false
```

**适用场景**：纯服务端逻辑的 Mod——如自动存档、世界规则调整、管理员工具。这些 Mod 不添加新的视觉内容，客户端不需要任何额外信息。

#### 模式三：纯客户端 Mod

```lua
-- modinfo.lua
client_only_mod = true
all_clients_require_mod = false
```

**适用场景**：只影响本地显示的 Mod——如 UI 美化、信息显示、小地图增强、血条样式修改。

**纯客户端 Mod 的特点**：
- 服务器不知道也不关心它的存在
- 不影响其他玩家
- 不能修改游戏逻辑（组件、战斗等）
- 可以修改 UI、显示信息、输入处理

引擎内部用 `client_only_mod` 来分类（`scripts/modindex.lua` 第 144-180 行）：

```lua
function ModIndex:GetServerModNames()
    local names = {}
    for modname, _ in pairs(self.savedata.known_mods) do
        if not self:GetModInfo(modname).client_only_mod then
            table.insert(names, modname)
        end
    end
    return names
end

function ModIndex:GetClientModNames()
    local names = {}
    for modname, _ in pairs(self.savedata.known_mods) do
        if self:GetModInfo(modname).client_only_mod then
            table.insert(names, modname)
        end
    end
    return names
end
```

---

### 3.6.3 ismastersim 实战——什么代码放哪边

这是 Mod 开发中最重要的判断之一。

#### 只能在服务端执行的操作

```lua
AddPrefabPostInit("spear", function(inst)
    if not TheWorld.ismastersim then return end  -- 客户端直接跳过

    -- 以下全部是服务端专属操作
    inst.components.weapon:SetDamage(100)            -- 修改组件数据
    inst.components.inventoryitem.cangoincontainer = false  -- 修改物品属性
    inst:AddComponent("mycomponent")                  -- 添加组件
    inst:ListenForEvent("attacked", function() end)   -- 监听游戏事件
end)
```

**规则**：所有 `inst.components.xxx` 的读写操作都**只能在服务端**。组件是服务端的权威数据，客户端没有。

#### 可以在两端执行的操作

```lua
AddPrefabPostInit("myentity", function(inst)
    -- 这些在客户端和服务端都执行
    inst:AddTag("mytag")              -- Tag 两端同步
    inst.AnimState:SetBuild("mybuild")  -- 动画设置
    inst.AnimState:SetBank("mybank")

    -- 这些只在客户端执行
    if not TheWorld.ismastersim then
        -- 客户端专属：UI 交互、视觉效果
        inst:ListenForEvent("mytag_dirty", function()
            -- 处理网络变量同步
        end)
        return  -- 客户端到此为止
    end

    -- 以下只在服务端执行
    inst:AddComponent("health")
    inst.components.health:SetMaxHealth(100)
end)
```

#### 常见的 ismastersim 模式总结

| 操作 | 哪一端 | 说明 |
|------|--------|------|
| `inst.components.xxx` | 仅服务端 | 组件是服务端权威数据 |
| `inst:AddComponent()` | 仅服务端 | 组件添加也是服务端操作 |
| `inst:AddTag()` / `inst:RemoveTag()` | 两端 | Tag 会自动网络同步 |
| `inst.AnimState:xxx()` | 两端 | 动画状态两端都需要设置 |
| `inst.Transform:xxx()` | 两端 | 位置/旋转两端都需要 |
| `inst:ListenForEvent("xxx_dirty")` | 仅客户端 | 监听网络变量变化 |
| `SpawnPrefab()` | 仅服务端 | 实体生成是服务端操作 |
| `inst:Remove()` | 仅服务端 | 实体删除是服务端操作 |
| `inst.replica.xxx` | 仅客户端 | Replica 是客户端的只读数据 |
| `inst.SoundEmitter:PlaySound()` | 两端 | 但通常只在客户端有意义 |

---

### 3.6.4 Replica 组件——客户端如何获取服务端数据

在服务端，你可以直接访问 `inst.components.health.currenthealth`。但客户端怎么知道这个实体还有多少血？答案是 **Replica 组件**。

每个需要在客户端显示数据的组件（如 `health`、`combat`、`inventory`），都有一个对应的 `xxx_replica.lua` 文件。Replica 组件是**只读**的客户端副本，它通过网络变量（`net_*`）从服务端接收数据。

```
服务端                              客户端
┌──────────────────┐              ┌──────────────────────┐
│ inst.components.  │   网络同步    │ inst.replica.health   │
│   health          │ ──────────► │   :GetCurrent()       │
│   .currenthealth  │   (net_*)    │   :GetMax()           │
│   .maxhealth      │              │   :IsDead()           │
└──────────────────┘              └──────────────────────┘
```

在 modmain.lua 中的用法：

```lua
-- 服务端：通过 components 访问
if TheWorld.ismastersim then
    local hp = inst.components.health.currenthealth
    inst.components.health:DoDelta(-10)
end

-- 客户端：通过 replica 访问（只读）
if not TheWorld.ismastersim then
    local hp = inst.replica.health:GetCurrent()
    local max = inst.replica.health:GetMax()
    local dead = inst.replica.health:IsDead()
end

-- 通用写法：两端都能工作
local function GetHP(inst)
    if TheWorld.ismastersim then
        return inst.components.health.currenthealth
    elseif inst.replica.health then
        return inst.replica.health:GetCurrent()
    end
    return 0
end
```

#### 哪些组件有 Replica？

在 `scripts/entityreplica.lua` 第 5 行起，有一个白名单：

```lua
local REPLICATABLE_COMPONENTS = {
    builder = true,
    combat = true,
    container = true,
    equippable = true,
    follower = true,
    health = true,
    hunger = true,
    inventoryitem = true,
    moisture = true,
    rider = true,
    sanity = true,
    stackable = true,
    -- ... 还有更多
}
```

如果你的 Mod 添加了一个新组件并且需要客户端读取数据，需要：

1. **创建 Replica 文件**：`scripts/components/mycomp_replica.lua`
2. **注册到白名单**：`AddReplicableComponent("mycomp")`
3. **在 Replica 中使用网络变量同步数据**

---

### 3.6.5 Mod 在客户端和服务端的加载

一个重要的事实：**不管是服务端 Mod 还是客户端 Mod，modmain.lua 中的顶层代码在两端都会执行**。

```lua
-- modmain.lua 的顶层代码——两端都执行
print("这行在服务端和客户端都会打印")

TUNING.SPEAR_DAMAGE = 100  -- 两端都执行

AddPrefabPostInit("spear", function(inst)
    -- 这个回调也是两端都执行
    -- 需要 ismastersim 来区分
    if not TheWorld.ismastersim then return end
    inst.components.weapon:SetDamage(TUNING.SPEAR_DAMAGE)
end)
```

这意味着：
- `PrefabFiles` 和 `Assets` 的声明两端都会处理
- `STRINGS` 的修改两端都需要（客户端显示文字，服务端用于某些逻辑判断）
- `TUNING` 的修改两端都需要（但只有服务端真正读取大部分值）
- `AddRecipe2` 两端都执行（客户端需要知道合成列表来显示 UI）

**真正的「服务端专属」代码**需要在 PostInit 回调中用 `ismastersim` 检查来隔离。

---

### 3.6.6 网络变量（net_*）——进阶主题

当你的 Mod 需要**自定义的客户端-服务端数据同步**时（超出 Replica 组件的范围），可以使用网络变量。

#### 基本模式

```lua
-- 在 prefab 文件中（AddNetwork 之前）
local function fn()
    local inst = CreateEntity()
    inst.entity:AddNetwork()

    -- 在 AddNetwork 后定义网络变量
    inst.my_net_var = net_bool(inst.GUID, "mymod.myvar", "myvar_dirty")

    if not TheWorld.ismastersim then
        -- 客户端：监听变量变化
        inst:ListenForEvent("myvar_dirty", function()
            local value = inst.my_net_var:value()
            -- 更新 UI 或视觉效果
        end)
        return inst
    end

    -- 服务端：设置变量值
    inst.my_net_var:set(true)

    return inst
end
```

常用的网络变量类型（引擎提供）：

| 类型 | 数据范围 | 典型用途 |
|------|---------|---------|
| `net_bool` | true/false | 开关状态 |
| `net_byte` | 0-255 | 小数值（等级、状态码） |
| `net_ushortint` | 0-65535 | 中等数值 |
| `net_int` | 32 位整数 | 大数值 |
| `net_float` | 浮点数 | 百分比、坐标 |
| `net_string` | 字符串 | 名称、消息 |
| `net_entity` | 实体 GUID | 引用其他实体 |
| `net_bytearray` | 字节数组 | 复杂数据 |

---

### 3.6.7 如何选择 Mod 类型

| 你的 Mod 做什么 | 推荐模式 | `client_only_mod` | `all_clients_require_mod` |
|----------------|---------|:-:|:-:|
| 添加新角色 | 服务端（全员必装） | `false` | `true` |
| 添加新物品/武器 | 服务端（全员必装） | `false` | `true` |
| 修改游戏机制 | 服务端（全员必装） | `false` | `true` |
| 添加新食谱 | 服务端（全员必装） | `false` | `true` |
| 只修改数值（不加新内容） | 服务端（仅服务端） | `false` | `false` |
| 管理员工具/控制台命令 | 服务端（仅服务端） | `false` | `false` |
| 修改 UI/HUD 显示 | 纯客户端 | `true` | `false` |
| 信息显示（如血条增强） | 纯客户端 | `true` | `false` |
| 小地图增强 | 纯客户端 | `true` | `false` |
| 皮肤/美化 | 纯客户端 | `true` | `false` |

---

### 3.6.8 常见问题与陷阱

**Q: 我在 PostInit 中不加 `ismastersim` 检查，直接访问 `inst.components.health`，为什么有时正常有时崩溃？**

A: 因为你自己开服自己玩时，你的客户端同时也是服务端（`ismastersim = true`），所以 `inst.components.health` 存在。但当别人加入你的服务器时，他们的客户端上 `ismastersim = false`，`inst.components` 是空的——直接访问就会崩溃。

**Q: 为什么我修改了 `TUNING` 值，但只在服务端生效？**

A: `TUNING` 的修改在两端都会执行（因为 modmain.lua 两端都跑），但有些 `TUNING` 值只在服务端被组件读取。如果你改的是客户端也需要的值（如 UI 显示相关），确保 modmain 的顶层代码修改了它——不要放在 `ismastersim` 检查后面。

**Q: 纯客户端 Mod 能用 `AddPrefabPostInit` 吗？**

A: 能。但回调中只能做客户端相关的事情（视觉效果、标签读取等）。不能访问 `inst.components`，因为客户端没有组件数据。

**Q: 如果我的 Mod 既修改了 UI 又修改了游戏逻辑，该选哪种模式？**

A: 选 `all_clients_require_mod = true`。只要你修改了任何服务端逻辑，就必须是服务端 Mod。如果同时修改了 UI，客户端也需要安装。

---

### 3.6.9 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | `TheWorld.ismastersim` 判断当前是服务端还是客户端；所有 `components` 操作必须在服务端；大多数 Mod 用 `all_clients_require_mod = true` + `client_only_mod = false` |
| **进阶** | 理解 Replica 组件是客户端的只读副本；`AddReplicableComponent` 扩展可复制白名单；PostInit 回调两端都执行但组件操作必须加 `ismastersim` 检查；Tag 和 AnimState 两端同步 |
| **老手** | `ismastersim` 来自 `TheNet:GetIsMasterSimulation()`；`net_*` 是引擎级的网络变量 API；`entity:AddNetwork()` 后才能定义网络变量；`ModWrangler:GetServerModsNames()` 根据 `ismastersim` 决定从本地还是网络获取 Mod 列表；`all_clients_require_mod` 驱动版本校验和自动下载 |

## 3.7 Mod 加载顺序与 priority 的影响

### 本节导读

当你启用了多个 Mod 时，它们的加载顺序会影响最终效果——先加载的 Mod 设置的值可能被后加载的覆盖，先注册的 PostInit 会先执行。理解加载顺序机制可以帮助你避免 Mod 冲突，也能让你的 Mod 在正确的时机执行。

> **新手**可以重点看 3.7.1-3.7.2，理解 priority 的基本规则；**进阶读者**可以深入 3.7.3-3.7.4，理解完整的加载流程和 PostInit 的执行顺序；**老手**可以关注 3.7.5-3.7.6，理解 `mod_dependencies`、`standalone` 和 `ForceEnable` 等高级机制。

---

### 3.7.1 priority 的基本规则

在 modinfo.lua 中，`priority` 字段控制 Mod 的加载顺序：

```lua
-- modinfo.lua
priority = 0  -- 默认值
```

**核心规则：priority 越大，越先加载。**

引擎的排序逻辑（`scripts/mods.lua` 第 528-558 行）：

```lua
local function sanitizepriority(priority)
    local prioritytype = type(priority)
    if prioritytype == "string" then
        return tonumber(priority) or 0
    elseif prioritytype == "number" then
        return priority
    end
    return 0
end

local function modPrioritySort(a, b)
    if a.modinfo and b.modinfo then
        local apriority = sanitizepriority(a.modinfo.priority)
        local bpriority = sanitizepriority(b.modinfo.priority)
        if apriority == bpriority then
            -- 优先级相同时，按 Mod 名字的字母顺序排序
            return stringidsorter(aname, bname)
        end
        return apriority > bpriority  -- 越大越前
    end
    return stringidsorter(a.modname, b.modname)
end

table.sort(self.mods, modPrioritySort)
```

排序规则总结：
1. **`priority` 数值越大，越先加载**
2. **priority 相同时，按 Mod 名字的字母顺序**（使用 `stringidsorter` 确保跨平台一致）
3. **不填 priority 默认为 0**
4. **支持字符串类型**——引擎会尝试 `tonumber()` 转换
5. **支持负数**——负数比 0 更晚加载

#### 实际案例：常见的 priority 设置

| Mod 类型 | priority 值 | 目的 |
|---------|------------|------|
| **Gem Core**（宝石核心） | `1.79769313486231e+308` | Lua 双精度浮点最大值——确保最先加载 |
| **普通 Mod** | `0` 或不设置 | 默认顺序 |
| **人物模板** | `-99999999999` | 确保最后加载——等所有 API 和库 Mod 就位 |

为什么 Gem Core 要用如此巨大的数字？因为它是一个被其他 Mod 依赖的「库 Mod」——如果它没有先加载，依赖它的 Mod 在 modmain.lua 中尝试调用它的 API 时，会因为 API 还不存在而报错。

---

### 3.7.2 加载顺序的实际影响

理解加载顺序很重要，因为它影响以下几个方面：

#### 影响一：modmain.lua 的执行顺序

```
priority 大的 Mod 先执行 modmain.lua
→ 先修改 TUNING
→ 先注册 PostInit
→ 先注册 Recipe
```

这意味着：如果两个 Mod 都修改了 `TUNING.SPEAR_DAMAGE`，**后加载的 Mod「赢」**——因为它的赋值覆盖了先加载的。

```lua
-- Mod A（priority = 10，先加载）
TUNING.SPEAR_DAMAGE = 50

-- Mod B（priority = 0，后加载）
TUNING.SPEAR_DAMAGE = 100

-- 最终结果：TUNING.SPEAR_DAMAGE = 100（Mod B 赢了）
```

#### 影响二：PostInit 的执行顺序

PostInit 回调的执行顺序和 Mod 加载顺序**一致**——先加载的 Mod 的回调先执行。

```lua
-- Mod A（先加载）
AddPrefabPostInit("spear", function(inst)
    print("A runs first")
    inst:AddTag("mod_a_processed")
end)

-- Mod B（后加载）
AddPrefabPostInit("spear", function(inst)
    print("B runs second")
    if inst:HasTag("mod_a_processed") then
        print("Mod A was here!")
    end
end)
```

这在引擎中通过 `GetPostInitFns` 保证——它按 `self.enabledmods` 的顺序（即加载顺序）遍历。

#### 影响三：require 的路径优先级

每个 Mod 加载时都会把自己的 `scripts` 目录加到 `package.path` 的**最前面**：

```lua
package.path = MODS_ROOT..mod.modname.."\\scripts\\?.lua;"..package.path
```

这意味着**后加载的 Mod 的 `scripts` 路径在前面**。如果 Mod A 和 Mod B 都有 `scripts/components/health.lua`，后加载的 Mod B 的版本会被 `require` 优先找到。

```
加载顺序：Mod A → Mod B → Mod C

package.path 最终为：
  Mod C/scripts/?.lua;    ← 最后加载的在最前面
  Mod B/scripts/?.lua;
  Mod A/scripts/?.lua;
  scripts/?.lua;           ← 原版最后
```

> **这看起来有点反直觉**：加载顺序靠前（priority 大）的 Mod，其 `require` 路径反而靠后。但这通常不是问题，因为大多数 Mod 不会创建和原版或其他 Mod 同名的 `require` 文件。

---

### 3.7.3 完整的 Mod 加载流程

从游戏启动到 Mod 完全就绪，经历以下阶段：

```
1. 游戏启动
   └─ main.lua 执行
       └─ 设置 package.path = "scripts\\?.lua;scriptlibs\\?.lua"
       └─ require("mods") → 创建 ModManager

2. KnownModIndex:Load()
   └─ 从磁盘加载 modindex（记录哪些 Mod 被启用）
   └─ UpdateModInfo() → 扫描 mods/ 目录
       └─ 对每个 Mod 执行 modinfo.lua（沙盒环境）

3. ModWrangler:LoadMods()
   ├─ KnownModIndex:GetModsToLoad()
   │    └─ 收集所有启用的 Mod 目录名
   │
   ├─ 对每个 Mod: CreateEnvironment() → 创建沙盒 env
   │
   ├─ table.sort(self.mods, modPrioritySort)
   │    └─ 按 priority 降序排列
   │
   └─ 对每个 Mod（按排序后的顺序）:
        ├─ package.path 加入该 Mod 的 scripts/
        ├─ InitializeModMain("modworldgenmain.lua")
        └─ InitializeModMain("modmain.lua")
             └─ 执行 modmain.lua 的顶层代码
                  └─ AddPrefabPostInit 等注册到 postinitfns

4. 游戏世界加载
   └─ Prefab 注册 → PrefabPostInit 回调缓存
   └─ 世界实体生成 → PrefabPostInit 执行
   └─ 组件添加 → ComponentPostInit 执行

5. SetPostEnv()（前端就绪后）
   └─ 注入 TheFrontEnd, TheSim, Point 等到 env
   └─ 执行每个 Mod 的 GamePostInit 回调

6. SimPostInit()（模拟初始化后）
   └─ 执行每个 Mod 的 SimPostInit 回调

7. 玩家进入世界
   └─ PlayerPostInit 回调执行
   └─ 游戏开始
```

---

### 3.7.4 何时该设置 priority

大多数 Mod 不需要设置 priority——默认的 0 就够了。只有以下情况需要调整：

#### 需要高 priority（先加载）

- **库/API 类 Mod**：被其他 Mod 依赖，必须先加载
- **框架类 Mod**：提供基础设施（如自定义皮肤 API、宝石核心等）

```lua
-- 库 Mod 的 modinfo.lua
priority = 10000  -- 确保在普通 Mod 之前加载
```

#### 需要低 priority（后加载）

- **角色 Mod**：通常依赖各种 API，需要等它们就位
- **「补丁」类 Mod**：需要修改其他 Mod 的行为

```lua
-- 角色 Mod 的 modinfo.lua
priority = -100  -- 确保在 API 类 Mod 之后加载
```

#### 不需要设置 priority

- 独立的功能 Mod（不依赖其他 Mod，也不被依赖）
- 大多数物品/武器/建筑 Mod

---

### 3.7.5 mod_dependencies——声明 Mod 依赖

除了 priority，饥荒还提供了正式的依赖声明机制。在 modinfo.lua 中：

```lua
mod_dependencies = {
    {
        workshop = "workshop-123456789",  -- 创意工坊 ID
        "GemCore",                         -- 备选：按 Mod 名查找
    },
}
```

引擎处理依赖的逻辑（`scripts/modindex.lua` 第 571-588 行）：

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
                self:DisableBecauseBad(modname)  -- 依赖不存在 → 禁用 Mod
            end
            table.insert(dependencies, mods)
        end
    end
end
```

如果声明的依赖 Mod 没有安装或启用，你的 Mod 会被自动禁用（`DisableBecauseBad`）。

**注意**：`mod_dependencies` 目前**不支持客户端 Mod**（源码中有 `not info.client_only_mod` 的条件，注释写着 `todo(Zachary): support client mods in the future`）。

---

### 3.7.6 standalone——独占模式

`standalone` 是一个极端的字段——设为 `true` 时，**只加载该 Mod，其他所有 Mod 都被排除**：

```lua
-- modinfo.lua
standalone = true
```

引擎检查（`scripts/modindex.lua` 第 311-314 行）：

```lua
for i, modname in ipairs(ret) do
    if self:IsModStandalone(modname) then
        print("Loading a standalone mod! No other mods will be loaded.")
        return { modname }  -- 只返回这一个 Mod
    end
end
```

这个字段极少使用，通常只用于「完全替换游戏内容」的大型 Total Conversion Mod。

---

### 3.7.7 ForceEnable——强制启用

开发者可以通过 `mods/modsettings.lua` 文件强制启用 Mod，绕过游戏的 UI 设置：

```lua
-- mods/modsettings.lua
ForceEnableMod("workshop-123456789")
```

这主要用于开发调试——引擎会在日志中打印警告：

```
WARNING: Force-enabling mod 'workshop-123456789' from modsettings.lua!
If you are not developing a mod, please use the in-game menu instead.
```

强制启用的 Mod 在 `GetModsToLoad` 中会被特别处理（`scripts/modindex.lua` 第 286-295 行），确保即使 UI 中没有启用，它也会被加载。

---

### 3.7.8 排查加载顺序问题的方法

当你怀疑 Mod 冲突是由加载顺序引起的，可以用以下方法调试：

**方法一：在 modmain.lua 中打印加载时间**

```lua
print("[MyMod] modmain.lua 正在执行，当前时间：" .. os.clock())
```

在日志中可以看到各 Mod 的 modmain.lua 执行顺序。

**方法二：检查 priority**

在控制台中查看所有启用 Mod 的 priority：

```lua
for _, modname in ipairs(ModManager.enabledmods) do
    local info = KnownModIndex:GetModInfo(modname)
    print(modname, "priority:", info.priority or 0)
end
```

**方法三：在 PostInit 中打印执行顺序**

```lua
AddPrefabPostInit("spear", function(inst)
    print("[MyMod] spear PostInit 执行了")
end)
```

如果多个 Mod 都注册了同一个 prefab 的 PostInit，日志会显示它们的执行顺序。

---

### 3.7.9 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | `priority` 越大越先加载；大多数 Mod 不需要设置 priority（默认 0）；如果你的 Mod 依赖某个 API Mod，确保 API Mod 先加载（它的 priority 应该更大）；两个 Mod 改同一个值时，后加载的赢 |
| **进阶** | 理解 PostInit 按加载顺序执行的规则；`package.path` 的累加是反序的（后加载的在前）；用 `mod_dependencies` 声明依赖关系；priority 相同时按名字字母序排列 |
| **老手** | `sanitizepriority` 处理字符串和非数字类型；`GetModsToLoad` 不按 priority 排序——排序在 `LoadMods` 中进行；`standalone = true` 排除其他所有 Mod；`ForceEnableMod` 在 `modsettings.lua` 中可绕过 UI；`SetPostEnv` 分别处理 `ForceEnabled` 和 `Enabled` 的 Mod 的 `GamePostInit` |

## 3.8 在 Mod 中覆盖与添加 STRINGS（物品名、描述、角色对话）

### 本节导读

游戏中你看到的每一行文字——物品名称、合成描述、角色台词、UI 提示——全部来自一张巨大的 `STRINGS` 表。当你的 Mod 添加了新物品或角色，就需要在 `STRINGS` 中添加对应的文字条目；当你想修改已有物品的名字或描述，也需要覆盖 `STRINGS` 中的值。

> **新手**可以重点看 3.8.1-3.8.3，学会给新物品添加名字和描述；**进阶读者**可以深入 3.8.4-3.8.5，掌握角色台词系统和多语言支持；**老手**可以关注 3.8.6，理解翻译管线和 PO 文件的工作原理。

---

### 3.8.1 STRINGS 表的结构

`STRINGS` 定义在 `scripts/strings.lua` 中，是一个多层嵌套的巨大 table。它的核心结构如下：

```lua
STRINGS = {
    -- 物品/实体的显示名称
    NAMES = {
        AXE = "Axe",
        SPEAR = "Spear",
        GOLDENAXE = "Golden Axe",
        -- ... 上千个条目
    },

    -- 合成配方的描述
    RECIPE_DESC = {
        AXE = "A sturdy axe for chopping.",
        SPEAR = "A poking implement.",
        -- ...
    },

    -- 角色台词（每个角色一个子表）
    CHARACTERS = {
        GENERIC = require("speech_wilson"),    -- 威尔逊（默认）
        WILLOW = require("speech_willow"),      -- 薇洛
        WOLFGANG = require("speech_wolfgang"),  -- 沃尔夫冈
        -- ... 所有角色
    },

    -- UI 文本
    UI = {
        MAINSCREEN = { ... },
        CRAFTING = { ... },
        -- ...
    },

    -- 动作文本
    ACTIONS = {
        CHOP = "Chop",
        MINE = "Mine",
        -- ...
    },

    -- 角色选择界面
    CHARACTER_TITLES = { wilson = "The Gentleman Scientist", ... },
    CHARACTER_NAMES = { wilson = "Wilson P. Higgsbury", ... },
    CHARACTER_DESCRIPTIONS = { wilson = "*Grows a magnificent beard", ... },
    CHARACTER_QUOTES = { wilson = "\"I will conquer it all with the power of my MIND!\"", ... },
}
```

**关键规则**：`NAMES` 中的键是 prefab 名的**大写形式**。当游戏需要显示一个实体的名字时，它会用 `STRINGS.NAMES[string.upper(inst.prefab)]` 来查找。

---

### 3.8.2 给新物品添加名字和描述

这是最常见的 STRINGS 操作。假设你的 Mod 添加了一把叫 `"myweapon"` 的武器：

```lua
-- modmain.lua

-- 1. 物品名称（大写 prefab 名）
STRINGS.NAMES.MYWEAPON = "霜之刃"

-- 2. 合成配方描述（在合成面板中显示）
STRINGS.RECIPE_DESC.MYWEAPON = "注入寒冰之力的利刃"

-- 3. 角色检视描述（鼠标悬浮 / 右键检查时显示）
STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON = "好冷！"
```

**注意**：`STRINGS` 不在 Mod 的沙盒 `env` 中——你需要通过 `GLOBAL` 访问，或者使用 env 元表技巧：

```lua
-- 方法一：通过 GLOBAL（推荐新手）
GLOBAL.STRINGS.NAMES.MYWEAPON = "霜之刃"

-- 方法二：设置了 env 元表后直接用（因为 __index 会找到 GLOBAL.STRINGS）
STRINGS.NAMES.MYWEAPON = "霜之刃"
```

#### 批量注册的写法

当你有多个新物品时，用循环更整洁：

```lua
local items = {
    {prefab = "myweapon", name = "霜之刃", desc = "注入寒冰之力的利刃", examine = "好冷！"},
    {prefab = "myarmor", name = "冰甲", desc = "由永恒冰块打造", examine = "穿上感觉凉飕飕的。"},
    {prefab = "myhat", name = "冰冠", desc = "寒冰王冠", examine = "头好冷。"},
}

for _, item in ipairs(items) do
    local upper = string.upper(item.prefab)
    STRINGS.NAMES[upper] = item.name
    STRINGS.RECIPE_DESC[upper] = item.desc
    STRINGS.CHARACTERS.GENERIC.DESCRIBE[upper] = item.examine
end
```

人物 Mod（renwu）就是用这种模式批量注册的：

```lua
-- renwu 的做法：从数据表中批量注册
for _, v in ipairs(item_data) do
    STRINGS.NAMES[string.upper(v.prefabname)] = v.name_cn
    STRINGS.CHARACTERS.GENERIC.DESCRIBE[string.upper(v.prefabname)] = v.describe
end
```

---

### 3.8.3 修改已有物品的名字

如果你想改变原版物品的显示名称，直接覆盖即可：

```lua
-- 把斧头的名字改成中文
STRINGS.NAMES.AXE = "石斧"
STRINGS.NAMES.GOLDENAXE = "金斧"
STRINGS.NAMES.MOONGLASSAXE = "月光石斧"

-- 把斧头的检视描述改掉
STRINGS.CHARACTERS.GENERIC.DESCRIBE.AXE = "一把普通的斧头。"
```

**这会影响所有角色的描述吗？** 不一定。`STRINGS.CHARACTERS` 的结构是这样的：

```lua
STRINGS.CHARACTERS = {
    GENERIC = { DESCRIBE = { AXE = "Choppy chop chop." } },
    WILLOW = { DESCRIBE = { AXE = "It's an axe." } },
    -- 每个角色可以有自己版本的描述
}
```

`GENERIC` 是默认描述（对应 Wilson），每个角色可以覆盖特定物品的描述。如果你只改了 `GENERIC.DESCRIBE.AXE`，只有使用默认台词的角色（Wilson）会受影响。要修改所有角色，需要遍历：

```lua
for char_name, char_table in pairs(STRINGS.CHARACTERS) do
    if char_table.DESCRIBE then
        char_table.DESCRIBE.AXE = "自定义描述"
    end
end
```

---

### 3.8.4 角色台词系统

角色台词是 STRINGS 中最复杂的部分。每个角色有一个完整的「台词树」，存储了该角色对游戏中各种情境的反应。

#### 台词树的结构

以 `speech_wilson.lua`（Wilson 的台词文件）为例：

```lua
-- scripts/speech_wilson.lua
return {
    -- 动作失败时的台词
    ACTIONFAIL = {
        GENERIC = {
            ITEMMIMIC = "Well that's inconvenient.",
        },
        ACTIVATE = {
            LOCKED_GATE = "The gate is locked.",
        },
        -- ...
    },

    -- 检视物品时的台词
    DESCRIBE = {
        AXE = "Choppy chop chop.",
        SPEAR = "Pointy!",
        CAMPFIRE = {
            GENERIC = "A fire!",
            HIGH = "That fire is getting out of hand!",
            NORMAL = "Nice and comfortable.",
            LOW = "The fire is getting low.",
            OUT = "The fire's gone out.",
        },
        -- ...
    },

    -- 宣告台词（对其他玩家说）
    ANNOUNCE_CHARLIE = "What was THAT?!",
    ANNOUNCE_COLD = "I'm freezing!",
    ANNOUNCE_HOT = "I'm overheating!",
    -- ...
}
```

注意 `DESCRIBE.CAMPFIRE` 不是一个字符串，而是一个子表——因为营火有多种状态（熄灭、微弱、正常、旺盛），每种状态有不同的描述。

#### 为 Mod 角色创建完整台词

如果你的 Mod 添加了新角色，需要创建一个完整的台词文件：

```lua
-- scripts/speech_mycharacter.lua
return {
    ACTIONFAIL = {
        GENERIC = {
            ITEMMIMIC = "没想到是个陷阱。",
        },
        -- ... 复制 speech_wilson.lua 的结构，替换为你的角色台词
    },

    DESCRIBE = {
        AXE = "砍柴的好帮手。",
        SPEAR = "尖尖的，很危险。",
        -- ... 为所有物品写台词（或者只写你想定制的，其余留空）
    },

    ANNOUNCE_CHARLIE = "谁在那里？！",
    ANNOUNCE_COLD = "好冷好冷……",
    -- ...
}
```

然后在 modmain.lua 中注册：

```lua
-- modmain.lua
STRINGS.CHARACTERS.MYCHARACTER = require("speech_mycharacter")
```

**实际的做法**：大多数角色 Mod 不会从头写所有台词。通常的做法是复制 `speech_wilson.lua` 的全部内容，然后替换你想定制的条目。没有定制的条目会保持 Wilson 的默认台词。

#### 为新物品添加所有角色的检视台词

如果你想让每个角色对你的新物品说不同的话：

```lua
-- 基本方式：只设置默认台词
STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON = "A powerful weapon."

-- 完整方式：为特定角色设置专属台词
STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON = "A powerful weapon."
STRINGS.CHARACTERS.WILLOW.DESCRIBE.MYWEAPON = "Oooh, can I set it on fire?"
STRINGS.CHARACTERS.WOLFGANG.DESCRIBE.MYWEAPON = "Is mighty weapon for mighty man!"
STRINGS.CHARACTERS.WENDY.DESCRIBE.MYWEAPON = "Its power means nothing in the face of death."
```

---

### 3.8.5 多语言支持——LoadPOFile

如果你希望你的 Mod 支持多种语言，可以使用 PO 文件（Portable Object，标准的国际化文件格式）。

#### 创建 PO 文件

创建一个 `.po` 文件，格式如下：

```
# Mod 的中文翻译
msgctxt "STRINGS.NAMES.MYWEAPON"
msgid "Frost Blade"
msgstr "霜之刃"

msgctxt "STRINGS.RECIPE_DESC.MYWEAPON"
msgid "A blade infused with frost."
msgstr "注入寒冰之力的利刃"

msgctxt "STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON"
msgid "So cold!"
msgstr "好冷！"
```

#### 在 modmain.lua 中加载

```lua
-- modmain.lua
-- 先设置英文默认值
STRINGS.NAMES.MYWEAPON = "Frost Blade"
STRINGS.RECIPE_DESC.MYWEAPON = "A blade infused with frost."
STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON = "So cold!"

-- 然后根据语言加载翻译
local lang = GLOBAL.LanguageTranslator.defaultlang or "en"
if lang == "zh" or lang == "zhr" then
    -- 中文：直接覆盖
    STRINGS.NAMES.MYWEAPON = "霜之刃"
    STRINGS.RECIPE_DESC.MYWEAPON = "注入寒冰之力的利刃"
    STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYWEAPON = "好冷！"
end

-- 或者使用 PO 文件（更规范的方式）
-- LoadPOFile("scripts/languages/zh.po", "zh")
```

`LoadPOFile` 的实现（`scripts/modutil.lua` 第 836-840 行）：

```lua
env.LoadPOFile = function(path, lang)
    require("translator")
    LanguageTranslator:LoadPOFile(path, lang)
end
```

#### 翻译管线的工作原理

当游戏切换语言时，会调用 `TranslateStringTable(STRINGS)`（`scripts/translator.lua` 第 199-225 行）：

```lua
local function DoTranslateStringTable(base, tbl)
    for k, v in pairs(tbl) do
        local path = base.."."..k
        if type(v) == "table" then
            DoTranslateStringTable(path, v)  -- 递归处理子表
        else
            local str = LanguageTranslator:GetTranslatedString(path)
            if str and str ~= "" then
                tbl[k] = str  -- 用翻译版本覆盖原始值
            end
        end
    end
end

function TranslateStringTable(tbl)
    DoTranslateStringTable("STRINGS", tbl)
end
```

它会递归遍历整个 `STRINGS` 表，对每个字符串叶子节点，用 `"STRINGS.NAMES.AXE"` 这样的路径去 PO 文件中查找翻译。如果找到了就替换，否则保持原值。

**注意**：`TranslateStringTable` 可能在你的 Mod 加载之后再次执行（比如玩家切换语言）。如果 PO 文件中没有你新增键的条目，你的中文值会被保持不变（因为查找返回空）；但如果 PO 文件中恰好有同名键的旧条目，就会被意外覆盖。

#### 神话书说的多语言方案

大型 Mod 通常把字符串拆到独立文件中：

```lua
-- scripts/languages/strings_myth_theme_en.lua
STRINGS.NAMES.MYTH_SWORD = "Celestial Sword"
STRINGS.CHARACTERS.GENERIC.DESCRIBE.MYTH_SWORD = "A divine weapon."
```

然后在 modmain 中根据语言导入对应文件：

```lua
local lang = GLOBAL.LanguageTranslator.defaultlang
if lang == "zh" or lang == "zhr" then
    modimport("scripts/languages/strings_myth_theme_zh.lua")
else
    modimport("scripts/languages/strings_myth_theme_en.lua")
end
```

---

### 3.8.6 常见的 STRINGS 子表速查

| 路径 | 用途 | 示例 |
|------|------|------|
| `STRINGS.NAMES.XXX` | 物品/实体显示名 | `STRINGS.NAMES.AXE = "Axe"` |
| `STRINGS.RECIPE_DESC.XXX` | 合成配方描述 | `STRINGS.RECIPE_DESC.AXE = "A sturdy tool."` |
| `STRINGS.CHARACTERS.GENERIC.DESCRIBE.XXX` | 默认角色检视描述 | `"Choppy chop."` |
| `STRINGS.CHARACTERS.GENERIC.ACTIONFAIL.XXX` | 动作失败台词 | `{LOCKED = "It's locked."}` |
| `STRINGS.CHARACTERS.GENERIC.ANNOUNCE_XXX` | 宣告台词 | `"I'm freezing!"` |
| `STRINGS.CHARACTER_TITLES.xxx` | 角色选择标题 | `"The Gentleman Scientist"` |
| `STRINGS.CHARACTER_NAMES.xxx` | 角色全名 | `"Wilson P. Higgsbury"` |
| `STRINGS.CHARACTER_DESCRIPTIONS.xxx` | 角色选择描述 | `"*Grows a magnificent beard"` |
| `STRINGS.CHARACTER_QUOTES.xxx` | 角色选择引言 | `"\"I will conquer...!\""` |
| `STRINGS.ACTIONS.XXX` | 动作按钮文本 | `"Chop"` |
| `STRINGS.UI.XXX` | UI 界面文本 | 各种菜单和按钮 |
| `STRINGS.TABS.XXX` | 合成栏标签 | `"Tools"` |

---

### 3.8.7 常见问题与陷阱

**Q: 为什么我设置了 `STRINGS.NAMES.MYWEAPON`，但游戏中显示 "MISSING NAME"？**

A: 检查键名是否是 prefab 名的**大写形式**。如果你的 prefab 叫 `"my_weapon"`（带下划线），键名应该是 `STRINGS.NAMES.MY_WEAPON`。

**Q: 我修改了 `STRINGS.CHARACTERS.GENERIC.DESCRIBE.AXE`，但我的自定义角色还是说原来的话？**

A: 因为你的角色可能有自己独立的台词表。检查 `STRINGS.CHARACTERS.MYCHARACTER.DESCRIBE.AXE` 是否存在——如果存在，它会覆盖 `GENERIC` 的值。

**Q: 中文显示为乱码怎么办？**

A: 确保你的 Lua 文件保存为 **UTF-8** 编码。饥荒要求所有字符串文件使用 UTF-8。

**Q: 我用 `LoadPOFile` 加载了翻译，但切换语言后没效果？**

A: `LoadPOFile` 只是把翻译条目加载到 `LanguageTranslator` 中。你还需要在加载后调用 `TranslateStringTable(STRINGS)` 让翻译生效，或者直接在 modmain 中根据当前语言手动设置字符串（推荐后者，更可控）。

---

### 3.8.8 小结

| 你是谁 | 你应该记住的 |
|--------|------------|
| **新手** | `STRINGS.NAMES` 用大写 prefab 名做键；三个常改的表：`NAMES`（名字）、`RECIPE_DESC`（配方描述）、`CHARACTERS.GENERIC.DESCRIBE`（检视台词）；通过 `GLOBAL.STRINGS` 或 env 元表访问 |
| **进阶** | 台词树 `speech_xxx.lua` 是 `return { ... }` 格式，由 `strings.lua` 中 `require` 组装到 `STRINGS.CHARACTERS` 下；多语言用 `modimport` 按语言选择不同字符串文件；批量注册用循环 + 数据表 |
| **老手** | `TranslateStringTable` 递归遍历 `STRINGS` 用路径查 PO 翻译；该函数可能在 Mod 之后再次执行（语言切换）；`LoadPOFile` 只是往 `LanguageTranslator` 写条目，不直接改 `STRINGS`；`DESCRIBE` 子表可以是字符串或 `{GENERIC = ..., HIGH = ...}` 子表形式（多状态实体） |

## 3.9 实战：修改一把武器的伤害

### 本节导读

终于到了把前面所有知识串在一起的时候。在这一节中，我们将从零开始创建一个完整的 Mod——修改长矛的伤害，并逐步扩展到更复杂的功能。每一步都会引用前面章节的知识点，帮你把「理论」变成「实践」。

> **新手**可以照着 3.9.1-3.9.3 一步步做，完成你的第一个 Mod；**进阶读者**可以深入 3.9.4-3.9.5，学习配置选项和多种修改方式；**老手**可以直接看 3.9.6，理解如何优雅地组织一个功能完善的武器修改 Mod。

---

### 3.9.1 第一步：建立 Mod 文件夹

在 `mods/` 目录下创建你的 Mod 文件夹，结构如下：

```
mods/
  my_weapon_mod/
    modinfo.lua      ← Mod 的身份证（3.1 节）
    modmain.lua      ← Mod 的入口（3.2 节）
    modicon.xml      ← 图标描述文件（可选）
    modicon.tex      ← 图标贴图（可选）
```

---

### 3.9.2 第二步：编写 modinfo.lua

运用 3.1 节的知识，写一个包含配置选项的 modinfo：

```lua
-- modinfo.lua
name = "武器伤害调整"
description = "修改长矛的伤害和耐久度\n支持自定义倍率"
author = "你的名字"
version = "1.0"

api_version = 10

dst_compatible = true
dont_starve_compatible = false

all_clients_require_mod = false  -- 纯数值修改，不需要客户端安装
client_only_mod = false

icon_atlas = "modicon.xml"
icon = "modicon.tex"

server_filter_tags = {"weapon", "balance"}

configuration_options = {
    {
        name = "damage_multiplier",
        label = "伤害倍率",
        hover = "调整长矛的伤害倍率",
        options = {
            {description = "0.5x", data = 0.5},
            {description = "1.0x（原版）", data = 1.0},
            {description = "1.5x", data = 1.5},
            {description = "2.0x", data = 2.0},
            {description = "3.0x", data = 3.0},
        },
        default = 1.0,
    },
    {
        name = "uses_multiplier",
        label = "耐久倍率",
        hover = "调整长矛的耐久度倍率",
        options = {
            {description = "0.5x", data = 0.5},
            {description = "1.0x（原版）", data = 1.0},
            {description = "2.0x", data = 2.0},
            {description = "无限耐久", data = -1},
        },
        default = 1.0,
    },
}
```

**知识点回顾**：
- `api_version = 10`（3.1.2：联机版必须为 10）
- `all_clients_require_mod = false`（3.6.2：纯数值修改不需要客户端）
- `configuration_options`（3.1.2 第七节：配置选项的完整结构）
- `GetModConfigData` 的读取优先级（3.1.2：saved_server > saved_client > saved > default）

---

### 3.9.3 第三步：编写 modmain.lua

#### 方法一：最简单——直接修改 TUNING

先看原版长矛的数值定义。在 `scripts/tuning.lua` 中：

```lua
-- scripts/tuning.lua 第 165, 354 行
local wilson_attack = 34   -- 威尔逊的基础攻击力

SPEAR_DAMAGE = wilson_attack,   -- 34（等于威尔逊一拳）
SPEAR_USES = 150,               -- 150 次使用
```

然后在 `scripts/prefabs/spear.lua` 中，这些值被使用：

```lua
-- scripts/prefabs/spear.lua 第 58-65 行
inst:AddComponent("weapon")
inst.components.weapon:SetDamage(TUNING.SPEAR_DAMAGE)

inst:AddComponent("finiteuses")
inst.components.finiteuses:SetMaxUses(TUNING.SPEAR_USES)
inst.components.finiteuses:SetUses(TUNING.SPEAR_USES)
```

因此，最简单的修改方式是**在 modmain.lua 中覆盖 TUNING 值**：

```lua
-- modmain.lua —— 方法一：修改 TUNING

-- 读取配置（3.1.2 第七节）
local damage_mult = GetModConfigData("damage_multiplier") or 1
local uses_mult = GetModConfigData("uses_multiplier") or 1

-- 修改数值（3.3.2：TUNING 在 env 中直接可用）
TUNING.SPEAR_DAMAGE = TUNING.SPEAR_DAMAGE * damage_mult

if uses_mult == -1 then
    TUNING.SPEAR_USES = math.huge  -- 无限耐久
else
    TUNING.SPEAR_USES = math.floor(TUNING.SPEAR_USES * uses_mult)
end
```

**优点**：代码极简，3 行有效代码。

**缺点**：只影响修改之后**新生成**的长矛。已经存在于世界中的长矛不受影响（因为它们的数值已经在生成时从 TUNING 读取并设置好了）。

#### 方法二：使用 AddPrefabPostInit——推荐

更可靠的方式是用 `AddPrefabPostInit`，在每把长矛生成后直接修改它的组件值：

```lua
-- modmain.lua —— 方法二：AddPrefabPostInit

local damage_mult = GetModConfigData("damage_multiplier") or 1
local uses_mult = GetModConfigData("uses_multiplier") or 1

AddPrefabPostInit("spear", function(inst)
    -- 客户端没有组件，跳过（3.6.3：ismastersim 检查）
    if not GLOBAL.TheWorld.ismastersim then return end

    -- 修改伤害
    if inst.components.weapon ~= nil then
        local base_damage = inst.components.weapon.damage
        inst.components.weapon:SetDamage(base_damage * damage_mult)
    end

    -- 修改耐久
    if inst.components.finiteuses ~= nil then
        if uses_mult == -1 then
            -- 无限耐久：移除耐久耗尽回调
            inst.components.finiteuses:SetOnFinished(nil)
            inst.components.finiteuses:SetMaxUses(999999)
            inst.components.finiteuses:SetUses(999999)
        else
            local base_uses = inst.components.finiteuses.total
            local new_uses = math.floor(base_uses * uses_mult)
            inst.components.finiteuses:SetMaxUses(new_uses)
            inst.components.finiteuses:SetUses(new_uses)
        end
    end
end)
```

**知识点回顾**：
- `AddPrefabPostInit`（3.4.3 / 3.5.1：对特定 prefab 生成后执行）
- `GLOBAL.TheWorld.ismastersim`（3.6.3：组件操作必须在服务端）
- `inst.components.weapon:SetDamage()`（原版 `weapon.lua` 的 API）
- `inst.components.finiteuses:SetMaxUses()`（原版 `finiteuses.lua` 的 API）

---

### 3.9.4 扩展：同时修改多种武器

如果你想修改多种武器，不需要为每把武器写一个 PostInit——用 `AddComponentPostInit` 更优雅：

```lua
-- modmain.lua —— 方法三：修改所有武器

local damage_mult = GetModConfigData("damage_multiplier") or 1

AddComponentPostInit("weapon", function(self, inst)
    -- self 是 weapon 组件实例，inst 是拥有者实体
    local old_SetDamage = self.SetDamage
    self.SetDamage = function(self, dmg)
        old_SetDamage(self, dmg * damage_mult)  -- 在原始设置基础上乘以倍率
    end
    -- 对已经设置的伤害也生效
    if self.damage ~= nil then
        self.damage = self.damage * damage_mult
    end
end)
```

**知识点回顾**：
- `AddComponentPostInit`（3.4.4：修改所有拥有该组件的实体）
- 方法 Hook 技巧（3.4.4：保存原方法 → 替换 → 调用原方法）

**优点**：所有武器（长矛、斧头、火把等）的伤害都会被修改。

**缺点**：影响范围太大——可能不是你想要的。需要加过滤逻辑：

```lua
AddComponentPostInit("weapon", function(self, inst)
    -- 只修改特定武器
    local target_weapons = {spear = true, goldenaxe = true, moonglassaxe = true}

    -- 等 prefab 名确定后再检查
    inst:DoTaskInTime(0, function()
        if target_weapons[inst.prefab] and self.damage ~= nil then
            self.damage = self.damage * damage_mult
        end
    end)
end)
```

---

### 3.9.5 进阶：添加特殊效果

现在让我们给长矛添加一个原版没有的效果——攻击时有 30% 概率冻结目标：

```lua
-- modmain.lua —— 进阶：冰冻长矛

AddPrefabPostInit("spear", function(inst)
    if not GLOBAL.TheWorld.ismastersim then return end

    if inst.components.weapon ~= nil then
        -- Hook 攻击回调
        local old_onattack = inst.components.weapon.onattack
        inst.components.weapon:SetOnAttack(function(weapon, attacker, target)
            -- 先执行原有的攻击逻辑
            if old_onattack then
                old_onattack(weapon, attacker, target)
            end

            -- 30% 概率冻结目标
            if target:IsValid() and math.random() < 0.3 then
                if target.components.freezable ~= nil then
                    target.components.freezable:AddColdness(2)
                    target.components.freezable:SpawnShatterFX()
                end
            end
        end)
    end
end)
```

如果你还想修改长矛的名字和描述（3.8 节的知识）：

```lua
-- 修改名字和描述
GLOBAL.STRINGS.NAMES.SPEAR = "冰霜长矛"
GLOBAL.STRINGS.RECIPE_DESC.SPEAR = "注入了冰霜之力的长矛"
GLOBAL.STRINGS.CHARACTERS.GENERIC.DESCRIBE.SPEAR = "矛尖散发着寒气。"
```

---

### 3.9.6 完整的 Mod 示例

把前面所有的知识综合起来，这是一个功能完善的 modmain.lua：

```lua
-- modmain.lua —— 完整版本

-----------------------------------------------------------------------
-- 环境配置（3.3.3：使用 env 元表简化 GLOBAL 访问）
-----------------------------------------------------------------------
GLOBAL.setmetatable(env, {
    __index = function(t, k)
        return GLOBAL.rawget(GLOBAL, k)
    end
})

-----------------------------------------------------------------------
-- 读取配置（3.1.2 第七节）
-----------------------------------------------------------------------
local damage_mult = GetModConfigData("damage_multiplier") or 1
local uses_mult = GetModConfigData("uses_multiplier") or 1

-----------------------------------------------------------------------
-- 修改文字（3.8.2）—— 如果倍率不是 1.0，在名字中提示
-----------------------------------------------------------------------
if damage_mult ~= 1 then
    local original_name = STRINGS.NAMES.SPEAR or "Spear"
    STRINGS.NAMES.SPEAR = original_name ..
        string.format(" (%sx伤害)", damage_mult)
end

-----------------------------------------------------------------------
-- 核心逻辑（3.4.3 / 3.5.1 / 3.6.3）
-----------------------------------------------------------------------
AddPrefabPostInit("spear", function(inst)
    if not TheWorld.ismastersim then return end

    -- 修改伤害
    if inst.components.weapon ~= nil then
        local base = inst.components.weapon.damage
        inst.components.weapon:SetDamage(base * damage_mult)
    end

    -- 修改耐久
    if inst.components.finiteuses ~= nil then
        if uses_mult == -1 then
            inst.components.finiteuses:SetOnFinished(nil)
            inst.components.finiteuses:SetMaxUses(999999)
            inst.components.finiteuses:SetUses(999999)
        elseif uses_mult ~= 1 then
            local base_uses = inst.components.finiteuses.total
            local new_uses = math.floor(base_uses * uses_mult)
            inst.components.finiteuses:SetMaxUses(new_uses)
            inst.components.finiteuses:SetUses(new_uses)
        end
    end
end)

-----------------------------------------------------------------------
-- 打印加载成功信息
-----------------------------------------------------------------------
print("[WeaponMod] 武器修改 Mod 已加载！伤害倍率: "
    .. damage_mult .. ", 耐久倍率: " .. uses_mult)
```

---

### 3.9.7 测试你的 Mod

1. 把 Mod 文件夹放到 `mods/` 目录下
2. 启动游戏，在 Mod 设置中启用你的 Mod
3. 在 Mod 配置中调整伤害和耐久倍率
4. 进入游戏，制作一把长矛
5. 打开控制台（`~` 键），输入以下命令验证：

```lua
-- 生成一把长矛并检查属性
c_give("spear")
local spear = ThePlayer.components.inventory:FindItem(function(item) return item.prefab == "spear" end)
if spear then
    print("伤害:", spear.components.weapon.damage)
    print("耐久:", spear.components.finiteuses.current, "/", spear.components.finiteuses.total)
end
```

如果伤害和耐久与你设置的倍率一致，恭喜——你的第一个 Mod 成功了！

---

### 3.9.8 章节总结——你在第 3 章学到了什么

回顾整个第 3 章，你已经掌握了创建一个 Mod 所需的全部基础知识：

| 章节 | 你学到的 |
|------|---------|
| **3.1** | modinfo.lua 的所有字段及其在引擎中的解析逻辑 |
| **3.2** | modmain.lua 的执行过程、沙盒环境的构成、PostInit 系列函数 |
| **3.3** | env、GLOBAL、setfenv 的深层原理，modimport 与 require 的区别 |
| **3.4** | modutil.lua 中所有 Mod API 的内部实现，注册—收集—分发模式 |
| **3.5** | 每个 API 的实战用法、回调签名、常见组合模式和决策矩阵 |
| **3.6** | 客户端/服务端架构、ismastersim、Replica 组件、网络变量 |
| **3.7** | priority 排序机制、加载顺序的影响、mod_dependencies |
| **3.8** | STRINGS 表的结构、物品命名、角色台词、多语言支持 |
| **3.9** | 从零开始创建一个完整的 Mod，把所有知识串在一起 |

