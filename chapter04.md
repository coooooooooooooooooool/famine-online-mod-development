# 第4章 调试与排错

## 4.1 日志系统：server_log.txt / client_log.txt 的阅读与分析

### 本节导读

日志文件是 Mod 开发者最基础、最核心的调试工具——它忠实地记录了游戏从启动到运行的一切关键信息。当你的 Mod 不生效、报错、崩溃时，日志文件往往是你定位问题的**第一线索**。

> **新手**可以从「日志文件在哪、怎么打开」开始，学会在日志中找到自己 Mod 的加载信息和报错；**进阶读者**可以深入理解日志的生成机制、不同日志的区别、以及如何在自己的 Mod 中输出调试信息；**老手**则可以直接跳到「深入引擎」部分，了解日志管线的底层架构、`print` 重写机制、以及高级调试技巧。

---

### 4.1.1 快速入门：日志文件在哪？

饥荒联机版会在运行时生成两种核心日志文件：

| 文件名 | 生成者 | 内容 |
|--------|--------|------|
| `server_log.txt` | 服务端进程 | 世界生成、Mod 加载、存档、玩家连接/断开、服务端逻辑错误 |
| `client_log.txt` | 客户端进程 | 界面渲染、资源加载、客户端 Mod 加载、输入处理、客户端逻辑错误 |

**文件位置**（Windows）：

```
文档\Klei\DoNotStarveTogether\<集群编号>\
├── Master\           ← 地面世界的服务端
│   └── server_log.txt
├── Caves\            ← 洞穴世界的服务端
│   └── server_log.txt
└── client_log.txt    ← 客户端日志（在集群根目录）
```

> **新手提示**：如果你是主机（host），你的电脑上会**同时存在** `server_log.txt` 和 `client_log.txt`，因为你既运行着服务端，又运行着客户端。如果你只是加入别人的服务器，只有 `client_log.txt` 有意义。

**日志会在每次启动游戏时被覆盖**，所以如果遇到 Bug 想保留日志，记得在重启前备份。此外，有一份 `server_log_previous.txt` / `client_log_previous.txt` 保存了**上一次**运行的日志。

---

### 4.1.2 读懂日志：Mod 加载的完整生命周期

一个典型的 Mod 从加载到运行会在日志中留下以下痕迹，让我们逐阶段拆解。

#### 阶段一：Mod 信息扫描

游戏启动后，`ModIndex`（模组索引管理器）会扫描所有已安装的 Mod，读取每个 Mod 的 `modinfo.lua`。如果读取失败，日志中会出现：

```
Error loading mod: workshop-123456789
WARNING loading modinfo.lua: workshop-123456789 does not specify if it is compatible with Don't Starve Together.
```

这段日志来自 `scripts/modindex.lua` 的 `InitializeModInfo` 函数。游戏在这个阶段会检查 `modinfo.lua` 中的必填字段，缺少关键字段会导致 Mod 被标记为 `failed`（具体检查逻辑已在 3.1 节详细分析过）。

如果是**专用服务器**（Dedicated Server），你还会看到：

```
ModIndex: Beginning normal load sequence for dedicated server.
```

#### 阶段二：Mod 加载

当玩家启动游戏或开服时，`ModWrangler:LoadMods` 函数开始依次加载已启用的 Mod。每个 Mod 加载时会打印：

```
Loading mod: 智能锅 (workshop-123456789) Version:2.1
```

这行日志来自 `scripts/mods.lua` 第 518–522 行：

```lua
local loadmsg = "Loading mod: "..ModInfoname(modname).." Version:"..env.modinfo.version
if initenv.modinfo_message and initenv.modinfo_message ~= "" then
    loadmsg = loadmsg .. " ("..initenv.modinfo_message..")"
end
print(loadmsg)
```

**为什么开发者要这么做？** 因为 Mod 加载是一个有序过程——Mod 按 `priority` 排序后依次加载（优先级高的先加载），打印加载信息可以让开发者和服务器管理员确认：

1. Mod 确实被识别并加载了
2. 加载的版本号是否正确
3. 加载顺序是否符合预期

紧接着，游戏会执行 Mod 的入口文件。每个 Mod 会依次执行 `modworldgenmain.lua`（世界生成逻辑）和 `modmain.lua`（主逻辑）：

```
Mod: 智能锅 (workshop-123456789) Loading modworldgenmain.lua
Mod: 智能锅 (workshop-123456789)   Mod had no modworldgenmain.lua. Skipping.
Mod: 智能锅 (workshop-123456789) Loading modmain.lua
```

这段日志来自 `scripts/mods.lua` 的 `InitializeModMain` 函数（第 587–618 行）：

```lua
function ModWrangler:InitializeModMain(modname, env, mainfile, safe)
    if not KnownModIndex:IsModCompatibleWithMode(modname) then return end

    print("Mod: "..ModInfoname(modname), "Loading "..mainfile)

    local fn = kleiloadlua(MODS_ROOT..modname.."/"..mainfile)
    if type(fn) == "string" then
        -- 编译失败：fn 是错误信息字符串
        print("Mod: "..ModInfoname(modname), "  Error loading mod!\n"..fn.."\n")
        table.insert( self.failedmods, {name=modname,error=fn} )
        return false
    elseif not fn then
        -- 文件不存在
        print("Mod: "..ModInfoname(modname), "  Mod had no "..mainfile..". Skipping.")
        return true
    else
        -- 在沙箱环境中执行
        local status, r = RunInEnvironment(fn,env)
        if status == false then
            moderror("Mod: "..ModInfoname(modname), "  Error loading mod!\n"..r.."\n")
            table.insert( self.failedmods, {name=modname,error=r} )
            return false
        end
        return true
    end
end
```

**这段代码揭示了三种情况**：

| 日志表现 | 含义 | 原因 |
|----------|------|------|
| `Loading modmain.lua`（后续无错误） | 加载成功 | 一切正常 |
| `Mod had no modmain.lua. Skipping.` | 跳过 | 文件不存在（对 `modworldgenmain.lua` 很正常） |
| `Error loading mod!` + 错误信息 | 加载失败 | 语法错误或运行时错误 |

#### 阶段三：Prefab 注册

Mod 加载完成后，游戏会注册 Mod 声明的 Prefab（预制体）：

```
Mod: 智能锅 (workshop-123456789) Registering prefabs
Mod: 智能锅 (workshop-123456789)   Registering prefab file: prefabs/smartpot
Mod: 智能锅 (workshop-123456789)     smartpot
Mod: 智能锅 (workshop-123456789)   Registering default mod prefab
```

来自 `scripts/mods.lua` 的 `RegisterPrefabs` 函数（第 675–718 行）。如果 Prefab 文件中存在错误，`runmodfn` 会捕获异常并输出详细的错误栈：

```lua
local runmodfn = function(fn,mod,modtype)
    return (function(...)
        if fn then
            local status, r = xpcall( function() return fn(unpack(arg)) end, debug.traceback)
            if not status then
                print("error calling "..modtype.." in mod "..ModInfoname(mod.modname)..": \n"..(r or ""))
                ModManager:RemoveBadMod(mod.modname,r)
                ModManager:DisplayBadMods()
            else
                return r
            end
        end
    end)
end
```

**为什么使用 `xpcall` + `debug.traceback`？** 这是 Lua 的"保护性调用"模式——即使 Mod 代码崩溃，也不会导致整个游戏崩溃。`debug.traceback` 会附带完整的函数调用栈，让你能追溯错误发生的完整路径。

#### 阶段四：运行时错误

如果 Mod 在运行中（比如某个 PostInit 回调里）出错，日志中会出现：

```
error calling <类型> in mod 智能锅 (workshop-123456789):
scripts/mods/workshop-123456789/modmain.lua:42: attempt to index a nil value
stack traceback:
    scripts/mods/workshop-123456789/modmain.lua:42: in function 'fn'
    scripts/modutil.lua:xxx: in function ...
```

出错的 Mod 会被自动禁用，日志中会打印：

```
Disabling 智能锅 (workshop-123456789) because it had an error.
```

> **新手关键技能**：学会在日志中搜索 `Error`、`error`、`SCRIPT CRASH`、`Disabling` 这几个关键词，它们是定位问题的最快捷径。

---

### 4.1.3 server_log 与 client_log 的区别

初学者最常困惑的问题之一是：**为什么有两个日志文件？我该看哪个？**

这要从饥荒联机版的**客户端-服务端架构**说起。

饥荒联机版采用经典的 C/S（Client/Server）架构。即使你在本地玩单人游戏，游戏也会在你的电脑上同时运行一个「服务端进程」和一个「客户端进程」。源码中用几个关键 API 来区分当前代码运行在哪一侧：

```lua
-- 当前进程是否为服务端？
TheNet:GetIsServer()

-- 当前进程是否拥有权威模拟权？（和 GetIsServer 类似，但在分片架构下有细微差别）
TheNet:GetIsMasterSimulation()

-- 当前进程是否为纯客户端？
TheNet:GetIsClient()

-- 当前进程是否为专用服务器（无本地界面）？
TheNet:IsDedicated()
```

在 `scripts/prefabs/world.lua` 第 425 行，世界实体在创建时就会保存这个标记：

```lua
inst.ismastersim = TheNet:GetIsMasterSimulation()
```

之后整个游戏中的无数代码分支都依赖 `TheWorld.ismastersim` 来决定执行逻辑——服务端执行权威逻辑（伤害计算、物品生成、AI 决策），客户端执行表现逻辑（动画、音效、UI）。

**这意味着**：

| 你的角色 | 该看哪个日志 | 原因 |
|----------|-------------|------|
| 纯客户端（加入别人的房间） | `client_log.txt` | 你只有客户端进程 |
| 主机 + 玩家 | **两个都看** | 你同时运行服务端和客户端 |
| 专用服务器管理员 | `server_log.txt` | 专用服没有客户端 |

**一个经典的例子**——在 `scripts/mainfunctions.lua` 第 1068 行，存档功能的入口：

```lua
function SaveGame(...)
    if not TheNet:GetIsServer() then
        print("SaveGame disabled for Clients in Don't Starve Together")
        return
    end
    -- ... 执行存档逻辑 ...
end
```

如果你在 `client_log.txt` 里看到 `"SaveGame disabled for Clients"`，这说明客户端试图调用存档函数——这很正常，不是 Bug。但如果你在 `server_log.txt` 里看到它，那就有问题了。

---

### 4.1.4 进阶：理解日志管线的底层机制

本节面向已经有基础的开发者，我们来深入研究日志系统是怎么工作的。

#### 日志系统的核心：`debugprint.lua`

饥荒联机版的日志系统并不复杂——它的核心就是**重写全局 `print` 函数**。

在 `scripts/main.lua` 的启动流程中，极早的位置就加载了日志模块：

```lua
require("strict")
require("debugprint")
-- 把引擎输出接口注册为打印监听器
AddPrintLogger(function(...) TheSim:LuaPrint(...) end)
```

`scripts/debugprint.lua` 的完整架构非常精巧，让我们逐层拆解：

**第一层：保存原始 `print` 并替换**

```lua
PRINT_SOURCE = false      -- 是否在输出中附带源码行号

local print_loggers = {}  -- 所有注册的日志监听器

function AddPrintLogger( fn )
    table.insert(print_loggers, fn)
end

local oldprint = print    -- 保存 Lua 原始 print
```

**第二层：新的 `print` 函数**

```lua
print = function(...)
    local str = ""
    if PRINT_SOURCE then
        -- 如果开启了源码追踪，附带调用者的文件名和行号
        local info = debug.getinfo(2, "Sl")
        local source = info and info.source
        if source then
            str = string.format("%s(%d,1) %s", source, info.currentline, packstring(...))
        else
            str = packstring(...)
        end
    else
        str = packstring(...)
    end

    -- 将拼接好的字符串广播给所有注册的监听器
    for i,v in ipairs(print_loggers) do
        v(str)
    end
end
```

**为什么要这么设计？** 这是一个典型的**观察者模式（Observer Pattern）**。游戏不知道（也不需要知道）日志最终输出到哪里——可能输出到控制台窗口、写入文件、显示在游戏内的调试叠加层，或者被 Mod 捕获做自定义处理。所有这些需求都通过 `AddPrintLogger` 注册各自的回调来实现。

**默认注册的监听器**：

1. **引擎输出**（`main.lua` 第 180 行）：`TheSim:LuaPrint(...)` —— 这是最关键的一个，引擎（C++ 侧）负责把文本写入 `server_log.txt` 或 `client_log.txt`
2. **控制台缓冲**（`debugprint.lua` 第 100–101 行）：`consolelog` 函数 —— 维护最近 20 行输出的环形缓冲，供游戏内控制台叠加层使用

**第三层：`nolineprint` —— 不带行号的打印**

```lua
nolineprint = function(...)
    for i,v in ipairs(print_loggers) do
        v(...)
    end
end
```

`nolineprint` 跳过了 `PRINT_SOURCE` 的行号拼接，直接把参数传给所有监听器。它的主要使用场景是**游戏内控制台**——当你在控制台执行命令时，错误信息通过 `nolineprint` 输出：

```lua
-- scripts/mainfunctions.lua 第 2137-2140 行
function ExecuteConsoleCommand(fnstr, guid, x, z)
    -- ...
    local status, r = pcall(loadstring(fnstr))
    if not status then
        nolineprint(r)  -- 控制台命令执行失败时输出错误
    end
    -- ...
end
```

#### 世界生成进程的独立日志管线

值得注意的是，**世界生成**运行在一个独立的 Lua 虚拟机中，有自己的日志管线。在 `scripts/worldgen_main.lua` 中：

```lua
require("debugprint")
AddPrintLogger(function(...) WorldSim:LuaPrint(...) end)
```

它注册的是 `WorldSim:LuaPrint` 而非 `TheSim:LuaPrint`。这意味着世界生成过程中的 `print` 输出会通过另一个引擎接口写入日志，但最终仍然进入 `server_log.txt`（因为世界生成是服务端的工作）。

#### 游戏内控制台日志叠加层

`debugprint.lua` 的后半部分还实现了游戏内可见的控制台日志叠加层：

```lua
local debugstr = {}
local MAX_CONSOLE_LINES = 20

local consolelog = function(...)
    local str = packstring(...)
    str = string.gsub(str, dir, "")  -- 去掉绝对路径前缀，让输出更简洁

    for idx,line in ipairs(string.split(str, "\r\n")) do
        table.insert(debugstr, line)
    end

    while #debugstr > MAX_CONSOLE_LINES do
        table.remove(debugstr,1)  -- 环形缓冲，保留最近 20 行
    end
end

function GetConsoleOutputList()
    return debugstr
end
```

这个环形缓冲被 `scripts/frontend.lua` 的 `UpdateConsoleOutput` 函数定期读取，用于渲染屏幕上的日志叠加层（按 `Ctrl+L` 或对应快捷键可以切换显示/隐藏）。

> **进阶提示**：`PRINT_SOURCE = true` 是一个非常有用的调试开关。开启后，所有 `print` 输出都会附带源文件名和行号（形如 `@scripts/modmain.lua(42,1) ...`），这在追踪「到底是哪里在打印这行日志」时非常有帮助。你可以在 `modmain.lua` 开头临时加上 `PRINT_SOURCE = true` 来开启它。

---

### 4.1.5 进阶：Mod 专用的日志函数

除了全局 `print`，游戏还为 Mod 提供了几个专用的日志/错误函数。它们定义在 `scripts/modutil.lua` 中，并被注入到每个 Mod 的沙箱环境里。

#### `moderror(message, level)` —— Mod 错误报告

```lua
function moderror(message, level)
    local modname = (global('env') and env.modname) or ModManager.currentlyloadingmod or "unknown mod"
    local message = string.format("MOD ERROR: %s: %s", ModInfoname(modname), tostring(message))
    if KnownModIndex:IsModErrorEnabled() then
        level = level or 1
        if level ~= 0 then
            level = level + 1
        end
        return error(message, level)  -- 抛出真正的 Lua error，会中断执行
    else
        print(message)                -- 只打印警告，不中断
        return
    end
end
```

`moderror` 的行为取决于 `IsModErrorEnabled()` 开关：

- **开启时**：调用 `error()` 抛出真正的 Lua 错误，程序中断，弹出错误对话框——适合**开发阶段**
- **关闭时**：只调用 `print()` 输出一行 `MOD ERROR: ...`，不中断程序——适合**发布后的正常使用**

这个开关可以在你的 `modsettings.lua` 中通过 `EnableModError()` 来开启。

**为什么要区分这两种模式？** 对于 Mod 用户来说，一个小的兼容性问题不应该直接崩溃游戏。但对于 Mod 开发者来说，你希望任何问题都能被立刻发现——所以开发时应该开启 `ModError`，发布时关闭。

#### `modassert(test, message)` —— Mod 断言

```lua
function modassert(test, message)
    if not test then
        return moderror(message)
    else
        return test
    end
end
```

和标准 `assert` 的语义一样，但错误走 `moderror` 管线——也就是说它也受 `IsModErrorEnabled()` 开关影响。

#### `modprint(...)` —— 条件打印

```lua
function modprint(...)
    if KnownModIndex:IsModErrorEnabled() then
        print(...)
    end
end
```

只在 `ModError` 开启时才打印，适合放一些开发期间需要、但发布后不想看到的调试信息。

#### `initprint(...)` —— 初始化阶段调试

```lua
local function initprint(...)
    if KnownModIndex:IsModInitPrintEnabled() then
        local modname = getfenvminfield(3, "modname")
        print(ModInfoname(modname), ...)
    end
end
```

这是游戏内部使用的函数（不直接暴露给 Mod 开发者），它在各种 `AddXxxPostInit`、`AddModRPCHandler` 等 API 被调用时输出初始化信息，方便追踪 Mod 的 PostInit 注册过程。

> **进阶提示**：在 `modsettings.lua`（位于 Mod 目录外的全局位置）中添加 `EnableModDebugPrint()` 可以开启 `initprint`，这样日志中会显示每个 Mod 注册了哪些 PostInit、RPC 等钩子。

---

### 4.1.6 进阶：常见错误日志模式与诊断

#### 模式一：`SCRIPT CRASH`

```
COROUTINE 12345 SCRIPT CRASH:
attempt to index a nil value
stack traceback:
    scripts/components/mycomponent.lua:28: in function 'OnUpdate'
    scripts/scheduler.lua:xxx: in function ...
```

这是**协程（coroutine）崩溃**，来自 `scripts/scheduler.lua` 第 248 行：

```lua
if not success then
    print (debug.traceback(v.co, "\nCOROUTINE "..tostring(v.id).." SCRIPT CRASH:\n".. tostring(yieldtype)))
    self:KillTask(v)
    return
end
```

**为什么用协程？** 饥荒的任务调度器（scheduler）大量使用 Lua 协程来实现延迟执行、周期性更新等。当协程内部发生错误时，`coroutine.resume` 返回 `false`，此时引擎会打印完整的栈追踪并杀死这个协程——不会影响其他协程和主循环。

**诊断方法**：关注栈追踪的第一行，它指出了错误发生的精确位置（文件名 + 行号）。`attempt to index a nil value` 通常意味着你在一个 `nil` 值上做了 `.` 或 `:` 操作。

#### 模式二：组件加载失败

```
component mycomponent already exists on entity 12345 - player!debugstack_oneline...
```

或者：

```
could not load component mycomponent
```

前者来自 `scripts/entityscript.lua` 的 `AddComponent` 函数——说明你对同一个实体重复添加了同名组件：

```lua
function EntityScript:AddComponent(name)
    local lower_name = string.lower(name)
    if self.lower_components_shadow[lower_name] ~= nil then
        print("component "..name.." already exists on entity "..tostring(self).."!"..debugstack_oneline(3))
        -- ...
    end
end
```

后者来自 `LoadComponent`——通常是组件文件路径或返回值有问题。

#### 模式三：Prefab 加载失败

```
Error loading file prefabs/myitem.lua
<具体的语法错误信息>
```

来自 `scripts/mainfunctions.lua` 的 `LoadPrefabFile` 函数。`loadfile` 在**编译阶段**就失败了，说明你的 Lua 文件有语法错误——多半是缺少 `end`、括号不匹配、逗号遗漏之类的问题。

#### 模式四：RPC 限流

```
Rate limiting RPCs from KU_xxxxx_xxx KU_xxxxx_xxx last one being ID 42
```

这行出现在 `server_log.txt` 中，来自 `scripts/networkclientrpc.lua` 的 `HandleRPC` 函数。当某个玩家发送 RPC 的频率超过了引擎的限流阈值 `RPC_QUEUE_RATE_LIMIT` 时，多余的 RPC 会被丢弃。

```lua
local limit = RPC_Queue_Limiter[sender] or 0
if limit < RPC_QUEUE_RATE_LIMIT then
    RPC_Queue_Limiter[sender] = limit + 1
    table.insert(RPC_Queue, { fn, sender, data, tick })
else
    if not RPC_Queue_Warned[sender] then
        RPC_Queue_Warned[sender] = true
        print("Rate limiting RPCs from", sender, userid, "last one being ID", tostring(code))
    end
end
```

**对 Mod 开发者的启示**：如果你的 Mod 需要频繁发送 RPC（比如每帧同步位置），你的 RPC 可能会被限流丢弃。正确的做法是**降低发送频率**，或者使用 NetVar（网络变量）来同步数据。

#### 模式五：无效 RPC

```
Invalid XXX RPC from (KU_xxxxx) PlayerName
```

来自 `scripts/networkclientrpc.lua` 的 `printinvalid` 函数，说明服务端收到了一个无法识别的 RPC 请求。常见原因：

1. 客户端和服务端的 Mod 版本不一致（RPC code 对不上）
2. 恶意客户端发送伪造的 RPC
3. Mod 的 RPC 注册顺序不一致导致 code 错位

#### 模式六：分片通信日志

```
[SyncWorldSettings] Sending master world option xxx = yyy to secondary shards.
World 2 is now connected
Validating portal[3] <-> 2[1] (VALID)
```

这些来自 `scripts/shardnetworking.lua`，记录了分片（主世界与洞穴）之间的通信状态。如果你看到 `disconnected` 或 `Skipping portal`，说明分片之间的连接出了问题。

---

### 4.1.7 老手进阶：高级调试技巧

#### 技巧一：注册自定义 Print Logger

既然 `print` 背后是一个监听器链，你完全可以注册自己的监听器来做自定义处理：

```lua
-- 在 modmain.lua 中
AddPrintLogger(function(str)
    -- 只关注和你的 Mod 相关的错误
    if string.find(str, "mymod") then
        -- 可以写入单独的文件、发送到远程服务器、触发自定义报警等
        -- 例如最简单的：把相关日志复制到一个全局表里
        if _G.MYMOD_LOGS == nil then
            _G.MYMOD_LOGS = {}
        end
        table.insert(_G.MYMOD_LOGS, str)
    end
end)
```

> **注意**：`AddPrintLogger` 回调中不要再调用 `print`，否则会无限递归。

#### 技巧二：利用 `debugtools.lua` 的调试工具

`scripts/debugtools.lua` 提供了一套强大的调试工具，虽然主要为引擎开发者设计，但 Mod 开发者也可以利用：

- **`dprint(...)`**：受 `CHEATS_ENABLE_DPRINT` 全局变量控制的条件打印，可以通过用户名过滤（`DPRINT_USERNAME`），只有特定开发者在线时才输出
- **`dumptable(t)`**：递归打印一个表的所有内容，调试复杂数据结构时非常有用

`dprint` 使用的 `oldprint`（定义在 `debugtools.lua` 第 209 行）**直接调用 `TheSim:LuaPrint`**，绕过了 `debugprint.lua` 的监听器链。这意味着它的输出会直接写入日志文件，但不会出现在游戏内控制台叠加层里。

#### 技巧三：理解 `RunInEnvironment` 与 `RunInEnvironmentSafe` 的区别

`scripts/util.lua` 中有两个用于在沙箱环境中执行代码的函数：

```lua
function RunInEnvironment(fn, fnenv)
    setfenv(fn, fnenv)
    return xpcall(fn, debug.traceback)
end

function RunInEnvironmentSafe(fn, fnenv)
    setfenv(fn, fnenv)
    return xpcall(fn, function(msg) print(msg) StackTraceToLog() print(debugstack()) return "" end )
end
```

区别在于错误处理：

- `RunInEnvironment`：错误时只做 `debug.traceback`（生成栈追踪字符串），由调用者决定怎么输出
- `RunInEnvironmentSafe`：错误时**立即打印**错误消息、输出完整的栈追踪到日志、再打印 `debugstack()` —— 信息更完整，但不可定制

Mod 的 `modmain.lua` 通常使用 `RunInEnvironment` 执行，前端部分（FrontendLoadMod）使用 `RunInEnvironmentSafe`。这就是为什么**前端 Mod 加载的错误信息通常更详细**。

#### 技巧四：`StackTrace` 与 `StackTraceToLog`

`scripts/stacktrace.lua` 提供了增强版的栈追踪功能：

```lua
function DoStackTrace(err)
    local res = {}
    if err then
        for idx,line in ipairs(string.split(err, "\n")) do
            res[#res+1] = "#"..line
        end
        res[#res+1] = "#LUA ERROR stack traceback:"
    end
    res = getdebugstack(res,5)
    local retval = concat(res, "\n")
    return retval
end

function StackTraceToLog()
    local s = StackTrace()
    print(s)
end
```

`StackTrace` 不仅输出调用栈，还会输出**每一层的局部变量**（通过 `getdebuglocals`），这比标准的 `debug.traceback` 提供了更多的上下文信息。在日志中看到以 `#` 开头的栈追踪行，就是来自这个系统。

#### 技巧五：专用服务器的日志特殊性

专用服务器（Dedicated Server）启动时会打印：

```
Starting Dedicated Server Game
```

如果专用服务器的 `dedicated_server_mods_setup.lua` 加载失败，会打印一段醒目的错误并直接关停：

```lua
print("########################################################")
print("#ERROR: Failure to load dedicated_server_mods_setup.lua:", err)
print("#Shutting down")
print("########################################################")
Shutdown()
```

**为什么要直接关停？** 因为对于专用服来说，Mod 配置失败意味着游戏可能处于不正确的状态。与其让玩家连入一个有问题的服务器，不如直接拒绝启动，让管理员修复配置。

---

### 4.1.8 实用速查：日志关键词检索表

最后，总结一张日志关键词速查表，方便你在遇到问题时快速定位：

| 搜索关键词 | 含义 | 出现在 | 优先级 |
|-----------|------|--------|--------|
| `SCRIPT CRASH` | 协程崩溃 | 两者皆可 | 🔴 高 |
| `Error loading mod` | Mod 加载/编译失败 | 两者皆可 | 🔴 高 |
| `MOD ERROR` | Mod 逻辑错误（`moderror` 输出） | 两者皆可 | 🔴 高 |
| `error calling` | 运行时回调错误 | 两者皆可 | 🔴 高 |
| `Disabling` ... `because it had an error` | Mod 因错误被自动禁用 | 两者皆可 | 🔴 高 |
| `could not load component` | 组件文件缺失或返回值错误 | 两者皆可 | 🔴 高 |
| `could not load stategraph` | 状态图文件缺失 | 两者皆可 | 🔴 高 |
| `Loading mod:` | Mod 正在加载 | 两者皆可 | 🟢 信息 |
| `Registering prefabs` | Prefab 注册阶段 | 两者皆可 | 🟢 信息 |
| `Rate limiting RPCs` | RPC 发送过于频繁被限流 | server_log | 🟡 警告 |
| `Invalid ... RPC` | 收到无效 RPC | server_log | 🟡 警告 |
| `SyncWorldSettings` | 分片世界设置同步 | server_log | 🟢 信息 |
| `is now connected/disconnected` | 分片连接状态 | server_log | 🟡 注意 |
| `SaveGame disabled for Clients` | 客户端尝试存档（正常） | client_log | ⚪ 忽略 |
| `WARNING loading modinfo.lua` | modinfo 缺少兼容性字段 | 两者皆可 | 🟡 警告 |
| `#LUA ERROR stack traceback` | 增强版栈追踪 | 两者皆可 | 🔴 高 |
| `#ERROR: Failure to load` | 专用服关键文件加载失败 | server_log | 🔴 高 |
| `component ... already exists` | 重复添加组件 | 两者皆可 | 🟡 警告 |

---

### 4.1.9 小结

- **日志文件**（`server_log.txt` / `client_log.txt`）是你最忠实的调试伙伴，遇到问题第一时间去看日志
- **`print` 被重写**为一个监听器管线（`debugprint.lua`），所有 `print` 输出都会写入日志文件并可选地显示在游戏内
- **`server_log`** 记录服务端逻辑，**`client_log`** 记录客户端逻辑——理解 C/S 架构是正确阅读日志的前提
- **Mod 专用的 `moderror`、`modassert`、`modprint`** 提供了可控的错误报告机制，开发时建议开启 `EnableModError()`
- **学会搜索关键词**：`SCRIPT CRASH`、`Error`、`MOD ERROR`、`Disabling`——这四个关键词能帮你快速定位 90% 的问题
- **高级技巧**包括注册自定义 `PrintLogger`、利用 `PRINT_SOURCE` 追踪日志来源、理解 `StackTrace` 增强栈追踪

> **下一节预告**：4.2 节我们将学习控制台调试命令（`c_spawn`、`c_give`、`c_select()` 等），这些命令可以让你在游戏运行时实时检查和修改游戏状态，是比日志更「实时」的调试手段。

## 4.2 控制台调试：c_spawn、c_give、c_select()、c_find() 等常用命令

### 本节导读

游戏内控制台是 Mod 开发者的「实时调试器」——你可以在游戏运行时生成物品、查看实体状态、修改数值、甚至直接执行任意 Lua 代码。比起看日志的「事后分析」，控制台让你能**即时观察和干预**游戏状态。

> **新手**可以从「怎么打开控制台、怎么用常用命令」开始；**进阶读者**可以了解命令的实现原理、`c_select` 的工作流程、以及本地执行与远程执行的区别；**老手**可以直接跳到「深入引擎」部分，了解控制台的底层执行管线、自动补全机制、以及如何编写自定义控制台命令。

---

### 4.2.1 快速入门：打开控制台并执行第一条命令

**打开方式**：在游戏中按 **`~`**（波浪号）键，屏幕下方会弹出一个输入框，这就是控制台。

按下 `~` 后，你会看到：
- 底部的文本输入区域
- 左侧可能显示 **`REMOTE`**（蓝色）或 **`LOCAL`**（红色）标签
- 按 `Ctrl+Shift` 可在远程/本地模式间切换

**第一条命令——生成一个物品**：

```lua
c_spawn("log", 5)
```

按回车，你的鼠标位置会出现 5 根木头。恭喜，你已经学会使用控制台了！

> **新手注意**：如果你在玩联机版并且是**主机**，控制台默认是 `REMOTE` 模式（命令发到服务端执行）。如果你只是**加入**别人的服务器，只有管理员才能使用远程模式。

---

### 4.2.2 新手必备：最常用的控制台命令速查

#### 生成与获取

| 命令 | 作用 | 示例 |
|------|------|------|
| `c_spawn("prefab", n)` | 在鼠标位置生成 n 个预制体 | `c_spawn("spear")` 生成一把长矛 |
| `c_give("prefab", n)` | 直接放入背包 n 个 | `c_give("cutgrass", 40)` 给 40 个割草 |
| `c_equip("prefab")` | 给一个并自动装备 | `c_equip("armorwood")` 装备木甲 |

**为什么有 `c_spawn` 和 `c_give` 两个？** 看源码就能理解。`c_spawn` 调用 `DebugSpawn` 在世界中创建实体（生成在鼠标位置），而 `c_give` 创建后还会调用 `inventory:GiveItem(inst)` 把实体放进背包：

```lua
-- c_give 的核心逻辑（scripts/consolecommands.lua 第 479-500 行）
function c_give(prefab, count, dontselect)
    local MainCharacter = ConsoleCommandPlayer()
    prefab = string.lower(prefab)
    if MainCharacter ~= nil then
        local first_inst = nil
        for i = 1, count or 1 do
            local inst = DebugSpawn(prefab)
            if inst ~= nil then
                if first_inst == nil then first_inst = inst end
                MainCharacter.components.inventory:GiveItem(inst)
                if not dontselect then
                    SetDebugEntity(inst)
                end
            end
        end
        return first_inst
    end
end
```

#### 选择与查找

| 命令 | 作用 | 示例 |
|------|------|------|
| `c_select()` | 选中鼠标下的实体 | 把鼠标指向一棵树，输入 `c_select()` |
| `c_sel()` | 获取当前选中的实体 | 选中后用 `c_sel()` 引用它 |
| `c_find("prefab")` | 找到最近的某种预制体 | `c_find("beefalo")` 找到最近的牛 |
| `c_findnext("prefab")` | 依次遍历所有同名预制体 | 连续调用可逐个查看 |
| `c_list("prefab")` | 列出所有同名预制体的位置 | `c_list("pigman")` 列出所有猪人 |
| `c_listtag("tag")` | 列出所有带指定标签的实体 | `c_listtag("monster")` |

**`c_select()` 和 `c_sel()` 的区别**：`c_select()` 是"选中操作"——它获取鼠标下的实体并设为调试目标；`c_sel()` 是"读取操作"——它只是返回当前已选中的实体。看源码一目了然：

```lua
-- scripts/consolecommands.lua 第 299-310 行
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

**选中后能做什么？** 这是控制台调试最强大的地方——选中一个实体后，你可以对它执行任意操作：

```lua
c_select()                          -- 选中鼠标下的实体
c_sel().components.health:Kill()    -- 杀死它
c_sel().components.health:SetPercent(1) -- 或者满血
c_sel():Remove()                    -- 直接删除
```

#### 玩家状态

| 命令 | 作用 | 示例 |
|------|------|------|
| `c_sethealth(n)` | 设置血量（0~1 比例） | `c_sethealth(1)` 满血 |
| `c_sethunger(n)` | 设置饱食度 | `c_sethunger(1)` 吃饱 |
| `c_setsanity(n)` | 设置精神值 | `c_setsanity(1)` 满脑力 |
| `c_godmode()` | 切换无敌模式 | 再次输入取消 |
| `c_supergodmode()` | 无敌 + 补满所有状态 | 开发测试神器 |
| `c_speedmult(n)` | 设置移动速度倍率 | `c_speedmult(4)` 四倍速 |

这些设置状态的命令都依赖 `ConsoleCommandPlayer()` 来确定目标玩家：

```lua
-- scripts/consolecommands.lua 第 2-4 行
function ConsoleCommandPlayer()
    return (c_sel() ~= nil and c_sel():HasTag("player") and c_sel()) or ThePlayer or AllPlayers[1]
end
```

**这个优先级很重要**：
1. 如果你通过 `c_select()` 选中了一个**玩家**实体，命令会作用于**被选中的玩家**
2. 否则作用于 `ThePlayer`（你自己）
3. 再否则作用于 `AllPlayers[1]`（第一个玩家）

这意味着你可以先 `c_select()` 点击别人的角色，再 `c_sethealth(1)` 给别人加血。

#### 服务器管理

| 命令 | 作用 | 示例 |
|------|------|------|
| `c_save()` | 手动存档 | |
| `c_rollback(n)` | 回滚 n 天 | `c_rollback(1)` 回滚一天 |
| `c_reset()` | 回滚到最近存档 | 等价于 `c_rollback(0)` |
| `c_regenerateworld()` | 重新生成世界 | 不可撤销！ |
| `c_shutdown()` | 关闭服务器 | 默认会先存档 |
| `c_announce("msg")` | 全服公告 | `c_announce("5分钟后重启")` |

---

### 4.2.3 进阶：理解控制台的执行管线

控制台看起来只是"输入命令→执行"，但背后有一套完整的管线。理解它有助于你搞清楚：**命令在哪里执行？为什么有些命令只在服务端生效？**

#### 控制台 UI：`ConsoleScreen`

当你按 `~` 打开控制台时，游戏推入一个 `ConsoleScreen` 界面（`scripts/screens/consolescreen.lua`）。当你按回车时，执行流程如下：

```lua
-- scripts/screens/consolescreen.lua 第 160-174 行
function ConsoleScreen:Run()
    local fnstr = self.console_edit:GetString()

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

**两条执行路径**：

1. **本地执行**（`LOCAL` 模式）：直接调用 `ExecuteConsoleCommand(fnstr)`，命令在**当前进程**的 Lua 环境中执行
2. **远程执行**（`REMOTE` 模式）：调用 `TheNet:SendRemoteExecute(fnstr, x, z)`，命令字符串被发到**服务端**执行

#### `ExecuteConsoleCommand` 的工作方式

```lua
-- scripts/mainfunctions.lua 第 2129-2146 行
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

**为什么需要 `overridepos`？** 这是为远程执行设计的。当客户端发送远程命令时，会附带客户端当前的鼠标世界坐标 `(x, z)`。服务端收到后，通过 `TheInput.overridepos` 临时伪造鼠标位置——这样 `c_spawn` 之类需要"鼠标位置"的命令就能在正确的位置生成物品。

**为什么用 `pcall` + `loadstring`？** `loadstring` 把你输入的字符串编译成 Lua 函数，`pcall` 保护性调用这个函数。如果你的命令有语法错误或运行时错误，`pcall` 会捕获异常并通过 `nolineprint` 输出错误信息，而不会导致游戏崩溃。

#### 本地 vs 远程：哪些命令需要远程？

看 `c_save` 的实现就能理解这个设计模式：

```lua
function c_save()
    if TheWorld ~= nil and not TheWorld.ismastersim then
        c_remote("c_save()")  -- 不是服务端？转发到服务端
        return
    end

    if TheWorld ~= nil and TheWorld.ismastersim then
        TheWorld:PushEvent("ms_save")  -- 在服务端执行真正的存档
    end
end
```

**模式**：先检查 `TheWorld.ismastersim`——如果当前不是主模拟（服务端），就用 `c_remote()` 把命令发到服务端。这个模式在 `c_reset`、`c_shutdown`、`c_godmode`、`c_regenerateworld` 等命令中反复出现。

`c_remote` 本身非常简单：

```lua
function c_remote( fnstr )
    local x, y, z = TheSim:ProjectScreenPos(TheSim:GetPosition())
    TheNet:SendRemoteExecute(fnstr, x, z)
end
```

**简而言之**：涉及游戏世界状态修改的命令必须在服务端执行（存档、回滚、生成实体等），涉及 UI 或客户端本地状态的命令可以在本地执行。

#### 权限控制

不是任何人都能远程执行命令的。在 `ConsoleScreen:ToggleRemoteExecute` 中：

```lua
local is_valid_time_to_use_remote = TheNet:GetIsClient() and (TheNet:GetIsServerAdmin() or IsConsole())
```

只有**服务器管理员**（`GetIsServerAdmin()`）或**主机**（`IsConsole()`）才能使用远程执行模式。普通玩家打开控制台只能执行本地命令——这就是为什么普通玩家不能在别人的服务器上作弊。

---

### 4.2.4 进阶：深入理解关键命令的实现

#### `c_spawn` —— 生成实体

```lua
-- scripts/consolecommands.lua 第 208-225 行
function c_spawn(prefab, count, dontselect)
    count = count or 1
    local inst = nil

    prefab = string.lower(prefab)

    for i = 1, count do
        inst = DebugSpawn(prefab)
        if inst and inst.components.skinner ~= nil and IsRestrictedCharacter(prefab) then
            inst.components.skinner:SetSkinMode("normal_skin")
        end
    end
    if not dontselect then
        SetDebugEntity(inst)
    end
    SuUsed("c_spawn_"..prefab, true)
    return inst
end
```

几个值得注意的细节：

1. **`string.lower(prefab)`**：prefab 名称会被强制转为小写——所以 `c_spawn("Log")` 和 `c_spawn("log")` 是一样的
2. **`DebugSpawn`**：这不是 `SpawnPrefab`，而是一个封装函数，会在鼠标位置（`ConsoleWorldPosition()`）生成实体
3. **`SetDebugEntity(inst)`**：默认会选中最后生成的实体——这意味着生成后你可以立刻用 `c_sel()` 引用它
4. **`SuUsed`**：这是一个作弊使用记录函数，用来标记玩家使用了控制台命令（在某些成就系统中可能禁用对应成就）
5. **返回值**：`c_spawn` 会**返回**最后生成的实体，你可以在控制台中直接链式调用：`c_spawn("spear").components.finiteuses:SetPercent(0.1)`

#### `c_find` 与 `c_findnext` —— 查找实体

`c_find` 找到**最近**的一个同名实体：

```lua
function c_find(prefab, radius, inst)
    inst = ListingOrConsolePlayer(inst)
    radius = radius or 9001     -- 默认搜索范围：9001（基本覆盖整个地图）

    local trans = inst.Transform
    local found = nil
    local founddistsq = nil

    local x,y,z = trans:GetWorldPosition()
    local ents = TheSim:FindEntities(x,y,z, radius)
    for k,v in pairs(ents) do
        if v ~= inst and v.prefab == prefab then
            if not founddistsq or inst:GetDistanceSqToInst(v) < founddistsq then
                found = v
                founddistsq = inst:GetDistanceSqToInst(v)
            end
        end
    end
    return found
end
```

**为什么用 `GetDistanceSqToInst` 而不是 `GetDistanceToInst`？** 因为计算距离平方比计算距离本身快（省去了开方运算）。在需要比较大小时，比较距离平方和比较距离是等价的——这是游戏开发中常见的性能优化技巧。

`c_findnext` 更复杂——它维护了一个 `lastfound` 变量（上次找到的实体 GUID），每次调用时跳过已找到的，实现**逐个遍历**：

```lua
function c_findnext(prefab, radius, inst)
    -- ...
    for k,v in pairs(ents) do
        if v ~= inst and v.prefab == prefab then
            total = total+1
            -- 找 GUID 比 lastfound 大的最小实体
            if v.GUID > lastfound and (foundlowestid == nil or v.GUID < foundlowestid) then
                idx = total
                found = v
                foundlowestid = v.GUID
            end
            -- 同时记录 GUID 最小的实体（用于循环回绕）
            if not reallowestid or v.GUID < reallowestid then
                reallowest = v
                reallowestid = v.GUID
            end
        end
    end
    if not found then
        found = reallowest  -- 遍历到头了，回到 GUID 最小的
    end
    -- ...
end
```

**实用技巧**：`c_findnext` 配合 `c_goto` 可以快速巡视地图上的所有同类实体——`c_goto(c_findnext("pighouse"))` 会依次传送到每个猪人房。

#### `c_godmode` 与 `c_supergodmode`

`c_godmode` 的实现揭示了一个有趣的设计——它不只是切换无敌，遇到鬼魂玩家还会自动复活：

```lua
function c_godmode(player)
    if TheWorld ~= nil and not TheWorld.ismastersim then
        c_remote("c_godmode()")
        return
    end

    player = ListingOrConsolePlayer(player)
    if player ~= nil then
        if player:HasTag("playerghost") then
            player:PushEvent("respawnfromghost")
            print("Reviving "..player.name.." from ghost.")
            return
        elseif player:HasTag("corpse") then
            player:PushEvent("respawnfromcorpse")
            print("Reviving "..player.name.." from corpse.")
            return
        elseif player.components.health ~= nil then
            local godmode = player.components.health.invincible
            player.components.health:SetInvincible(not godmode)
            print("God mode: "..tostring(not godmode))
        end
    end
end
```

`c_supergodmode` 在此基础上还会调用：

```lua
c_sethealth(1)
c_setsanity(1)
c_sethunger(1)
c_settemperature(25)
c_setmoisture(0)
```

**注意**：`c_supergodmode` 调用的 `c_sethealth` 等函数使用 `ConsoleCommandPlayer()` 来确定目标。如果你之前 `c_select` 了另一个玩家，状态修改可能会作用到那个玩家身上。这是一个容易踩的坑。

#### `c_dump` —— 转储实体信息

```lua
function c_dump()
    local ent = GetDebugEntity()
    if not ent then
        ent = ConsoleWorldEntityUnderMouse()
    end
    DumpEntity(ent)
end
```

`c_dump` 会打印实体的所有组件、标签、变量等详细信息——这是检查实体状态的终极工具。配合 `c_select()` 使用效果最佳。

---

### 4.2.5 进阶：控制台中的实用组合技

#### 组合一：精确生成并配置

```lua
-- 生成一把武器并设置耐久为 10%
local spear = c_spawn("spear")
spear.components.finiteuses:SetPercent(0.1)
```

控制台支持多行逻辑（通过 `local` 变量或利用返回值），不过每次回车只能执行一行。更实用的做法是：

```lua
c_spawn("spear").components.finiteuses:SetPercent(0.1)
```

#### 组合二：查找并传送

```lua
c_goto(c_find("chester"))    -- 传送到切斯特身边
c_goto(c_findnext("pigking")) -- 传送到猪王
```

#### 组合三：批量检查实体

```lua
c_countprefabs()   -- 列出世界中所有 prefab 的数量
c_list("spider")   -- 列出所有蜘蛛的位置
c_listtag("tree")  -- 列出所有带 "tree" 标签的实体
```

#### 组合四：选中后操作

```lua
c_select()                                     -- 选中鼠标下的实体
c_sel().components.health:SetPercent(0.5)       -- 设为半血
c_sel().components.combat:GetAttacked(ThePlayer, 100) -- 让它受到100点伤害
c_sel():PushEvent("death")                     -- 直接触发死亡事件
c_sel():Remove()                               -- 从世界中删除
```

#### 组合五：修改世界状态

```lua
c_dumpworldstate()      -- 查看当前世界状态
c_simphase("night")     -- 立刻切换到夜晚
TheWorld:PushEvent("ms_advanceseason")  -- 跳过当前季节（需远程模式）
TheWorld:PushEvent("ms_setseason", "winter") -- 设置为冬天
```

#### 组合六：针对其他玩家操作

```lua
c_listplayers()                -- 查看所有在线玩家
-- 输出形如：*[1] (KU_xxxxx) PlayerName <wilson>
-- * 号表示管理员

c_find("wilson", nil, UserToPlayer("KU_xxxxx"))  -- 以特定玩家为中心搜索
```

---

### 4.2.6 老手进阶：控制台的底层机制

#### 自动补全系统

控制台支持 Tab 自动补全。当你输入 `c_sp` 按 Tab 时，会自动补全为 `c_spawn`。这个功能在 `ConsoleScreen:DoInit` 中初始化：

```lua
-- scripts/screens/consolescreen.lua DoInit 中
if prediction_command_c == nil then
    prediction_command_c = {}
    for k, v in pairs(_G) do
        if type(k) == "string" and k:sub(1, 2) == "c_" and (type(v) == "function" or type(v) == "number") then
            prediction_command_c[#prediction_command_c + 1] = k
        end
    end
    table.sort(prediction_command_c)
end
```

**这意味着什么？** 如果你在 `modmain.lua` 中定义了以 `c_` 开头的全局函数，它会**自动出现在控制台的补全列表中**！这是编写自定义调试命令的基础。

#### 编写自定义控制台命令

只需在 `modmain.lua` 中定义 `c_` 开头的全局函数：

```lua
-- 在 modmain.lua 中
function c_mymod_test()
    local player = ConsoleCommandPlayer()
    if player then
        print("当前玩家:", player.name)
        print("位置:", player.Transform:GetWorldPosition())
        print("血量:", player.components.health:GetPercent())
    end
end

function c_mymod_killall(prefab)
    local x, y, z = ConsoleCommandPlayer().Transform:GetWorldPosition()
    local ents = TheSim:FindEntities(x, y, z, 50)
    local count = 0
    for _, v in pairs(ents) do
        if v.prefab == prefab and v.components.health then
            v.components.health:Kill()
            count = count + 1
        end
    end
    print("已击杀 " .. count .. " 个 " .. prefab)
end
```

然后在控制台中输入 `c_mymod_test()` 或 `c_mymod_killall("spider")` 即可。

> **命名建议**：自定义命令建议使用 `c_<modname>_<command>` 格式（如 `c_mymod_debug`），避免与其他 Mod 或未来的官方命令冲突。

#### `SetDebugEntity` 的底层实现

控制台选中系统的核心是 `SetDebugEntity`（定义在 `scripts/mainfunctions.lua` 第 650-660 行）：

```lua
function SetDebugEntity(inst)
    if debug_entity ~= nil and debug_entity:IsValid() then
        debug_entity.entity:SetSelected(false)   -- 取消旧实体的选中高亮
    end
    if inst ~= nil and inst:IsValid() then
        debug_entity = inst
        inst.entity:SetSelected(true)            -- 新实体显示选中高亮
    else
        debug_entity = nil
    end
end
```

`entity:SetSelected(true)` 会在引擎层给实体添加**可见的选中高亮**（黄色轮廓），这就是你 `c_select()` 后能在游戏中看到实体被高亮的原因。`debug_entity` 是模块局部变量，通过 `GetDebugEntity()` 和 `SetDebugEntity()` 对外暴露。

#### `ConsoleWorldEntityUnderMouse` 的工作原理

```lua
function ConsoleWorldEntityUnderMouse()
    if TheInput.overridepos == nil then
        return TheInput:GetWorldEntityUnderMouse()  -- 正常情况：读取鼠标下的实体
    else
        -- 远程执行情况：用 overridepos 模拟的坐标查找附近实体
        local x, y, z = TheInput.overridepos:Get()
        local ents = TheSim:FindEntities(x, y, z, 1)
        for i, v in ipairs(ents) do
            if v.entity:IsVisible() then
                return v
            end
        end
    end
end
```

**两种情况**：
- **本地执行**：直接调用 `TheInput:GetWorldEntityUnderMouse()`，这个函数读取当前帧鼠标所在位置的实体（基于引擎的碰撞检测）
- **远程执行**：鼠标位置是客户端发过来的坐标，通过 `TheSim:FindEntities` 在该坐标附近 1 单位半径内搜索可见实体

#### 控制台命令执行的完整时序

```
玩家按 ~ → PushScreen(ConsoleScreen)
  ↓
输入命令文本，按 Enter
  ↓
ConsoleScreen:OnTextEntered()
  ↓
DoTaskInTime(0, DoRun)  ← 延迟一帧，避免输入状态冲突
  ↓
ConsoleScreen:Run()
  ├─ 保存到命令历史
  ├─ [REMOTE 模式] TheNet:SendRemoteExecute(fnstr, x, z)
  │     → 网络传输到服务端
  │     → 服务端 ExecuteConsoleCommand(fnstr, guid, x, z)
  │         → 临时设置 ThePlayer 和 overridepos
  │         → pcall(loadstring(fnstr))
  │         → 恢复原始状态
  └─ [LOCAL 模式] ExecuteConsoleCommand(fnstr)
        → pcall(loadstring(fnstr))
        → 错误时 nolineprint(r) 输出到控制台
  ↓
ConsoleScreen:Close()
  ↓
ConsoleScreenSettings:Save()  ← 保存命令历史到磁盘
```

#### `d_` 前缀的调试命令

除了 `c_` 命令，游戏还有一套 `d_` 前缀的开发者调试命令（定义在 `scripts/debugcommands.lua`）。但这些命令只在 `CHEATS_ENABLED` 为 `true` 时才会加载：

```lua
-- scripts/main.lua 第 534-536 行
if CHEATS_ENABLED then
    require "debugcommands"
end
```

`d_` 命令通常更底层，比如 `d_spawnlist`（批量生成一组预制体）、`d_anim`（操作实体动画）等。对于普通 Mod 开发者来说，`c_` 命令已经足够了。

---

### 4.2.7 实用速查：控制台命令分类表

#### 生成与物品

| 命令 | 说明 |
|------|------|
| `c_spawn("prefab", n)` | 在鼠标位置生成 |
| `c_give("prefab", n)` | 放入背包 |
| `c_equip("prefab")` | 给予并装备 |
| `c_countprefabs()` | 统计世界中所有 prefab 类型及数量 |

#### 查找与定位

| 命令 | 说明 |
|------|------|
| `c_select()` | 选中鼠标下的实体 |
| `c_sel()` | 返回当前选中实体 |
| `c_find("prefab", radius)` | 查找最近的 prefab |
| `c_findnext("prefab")` | 依次遍历同名 prefab |
| `c_list("prefab")` | 列出所有同名 prefab 位置 |
| `c_listtag("tag")` | 列出所有带指定 tag 的实体 |
| `c_goto(inst)` | 传送到指定实体 |
| `c_gonext("prefab")` | 传送到下一个同名实体 |
| `c_inst(guid)` | 通过 GUID 获取实体引用 |

#### 玩家状态

| 命令 | 说明 |
|------|------|
| `c_sethealth(n)` | 设置血量百分比 (0~1) |
| `c_sethunger(n)` | 设置饱食度百分比 |
| `c_setsanity(n)` | 设置精神值百分比 |
| `c_settemperature(n)` | 设置体温（数值） |
| `c_setmoisture(n)` | 设置湿度百分比 |
| `c_godmode()` | 切换无敌 |
| `c_supergodmode()` | 无敌 + 补满状态 |
| `c_speedmult(n)` | 移动速度倍率 |

#### 服务器管理

| 命令 | 说明 |
|------|------|
| `c_save()` | 存档 |
| `c_rollback(n)` | 回滚 n 天 |
| `c_reset()` | 回滚到最近存档 |
| `c_regenerateworld()` | 重新生成世界 |
| `c_shutdown(save)` | 关服（默认存档） |
| `c_announce("msg")` | 全服公告 |
| `c_listplayers()` | 列出在线玩家 |
| `c_listallplayers()` | 列出 AllPlayers 表 |
| `c_despawn(player)` | 踢回角色选择 |

#### 信息查看

| 命令 | 说明 |
|------|------|
| `c_dump()` | 转储选中实体详细信息 |
| `c_dumpseasons()` | 打印季节信息 |
| `c_dumpworldstate()` | 打印世界状态 |
| `c_tile()` | 查看光标下的地皮类型 |
| `c_remote("lua代码")` | 远程执行 Lua 代码 |

---

### 4.2.8 小结

- **控制台**（按 `~` 打开）是实时调试利器，支持执行任意 Lua 代码
- **`c_spawn` / `c_give`** 用于生成物品，**`c_select` / `c_sel`** 用于选中和操作实体
- **REMOTE / LOCAL** 模式决定命令在服务端还是客户端执行——涉及世界状态的命令必须在服务端执行
- **控制台命令本质上就是全局 Lua 函数**——你可以在 Mod 中定义 `c_` 开头的函数来创建自定义命令
- **`ConsoleCommandPlayer()`** 决定了命令作用的目标玩家——先看 `c_sel()`，再看 `ThePlayer`，最后看 `AllPlayers[1]`
- **`SetDebugEntity` / `GetDebugEntity`** 是选中系统的核心——选中后可以对实体做任意操作

> **下一节预告**：4.3 节我们将学习 `print` 调试法与 `debugtools.lua`，这是在 Mod 代码中主动输出调试信息的方法——和控制台的"外部干预"互补，`print` 调试让你能从代码内部追踪执行流程。

## 4.3 print 调试法与 debugtools.lua

（待编写）

## 4.4 常见崩溃类型：nil 访问、栈溢出、无限循环的排查流程

（待编写）

## 4.5 网络相关 bug 的调试思路（客户端 vs 服务端）

（待编写）

## 4.6 性能问题的初步定位

（待编写）

## 4.7 Mod 冲突的快速排查方法

（待编写）
