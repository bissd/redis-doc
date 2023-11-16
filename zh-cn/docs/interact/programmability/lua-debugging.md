---
title: 在Redis中调试Lua脚本
linkTitle: 调试 Lua
description: 如何使用内置的Lua调试器
weight: 4
aliases:
  - /topics/ldb
  - /docs/manual/programmability/lua-debugging/
---

从3.2版本开始，Redis包含了一个完整的Lua调试器，可以用来简化编写复杂Redis脚本的任务。

Redis Lua调试器，代号为LDB，具有以下重要特性：

* 它使用服务器-客户端模型，因此是一个远程调试器。
Redis服务器充当调试服务器，而默认客户端是`redis-cli`。
然而，通过遵循服务器实现的简单协议，也可以开发其他客户端。
* 默认情况下，每个新的调试会话都是一个分叉会话。
这意味着在调试Redis Lua脚本时，服务器不会阻塞，并且可用于开发或以便同时执行多个调试会话。
这也意味着在脚本调试会话结束后，更改会被**回滚**，这样就可以再次启动新的调试会话，使用与上一个调试会话完全相同的Redis数据集。
* 在需求上提供了一种替代的同步（非分叉）调试模型，以便保留数据集的更改。
在此模式下，服务器在调试会话处于活动状态时被阻塞。
* 支持逐步执行。
* 支持静态和动态断点。
* 支持将调试脚本记录到调试器控制台。
* Lua变量的检查。
* 跟踪脚本执行的Redis命令。
* Redis和Lua值的漂亮打印。
* 发现无限循环和长时间执行，模拟断点。

## 快速入门

一个简单的方法来开始使用Lua调试器是观看这个视频
介绍：

<iframe width="560" height="315" src="https://www.youtube.com/embed/IMvRfStaoyM" frameborder="0" allowfullscreen></iframe>

## 重要提示：请确保避免在生产服务器上调试Lua脚本。请使用开发服务器代替。
还请注意，使用同步调试模式（不是默认模式）会导致Redis服务器在整个调试会话期间阻塞。

使用`redis-cli`启动一个新的调试会话，请按照以下步骤进行操作：

以您喜欢的编辑器在某个文件中创建您的脚本。假设您正在编辑位于`/tmp/script.lua`的Redis Lua脚本。
2. 使用以下命令开始调试会话：

    ./redis-cli --ldb --eval /tmp/script.lua

要注意，在 `redis-cli` 的 `--eval` 选项中，您可以通过逗号将键名和参数传递给脚本，就像以下示例中一样：

```
./redis-cli --ldb --eval /tmp/script.lua mykey somekey , arg1 arg2
```

您将进入一个特殊模式，`redis-cli`不再接受正常的命令，而是打印一个帮助窗口，并直接将未修改的调试命令传递给Redis。

只有以下命令不会传递给Redis调试器：

* `quit` - 这将终止调试会话。就像移除所有断点并使用`continue`调试命令一样。此外，该命令将退出`redis-cli`。
* `restart` - 调试会话将从头开始重新启动，**重新加载脚本文件中的新版本**。因此，正常的调试循环涉及在一些调试后修改脚本，并调用`restart`以便使用新的脚本更改重新开始调试。
* `help` - 此命令将传递给Redis Lua调试器，它将打印如下的命令列表：

```
lua debugger> help
Redis Lua debugger help:
[h]elp               Show this help.
[s]tep               Run current line and stop again.
[n]ext               Alias for step.
[c]ontinue           Run till next breakpoint.
[l]ist               List source code around current line.
[l]ist [line]        List source code around [line].
                     line = 0 means: current position.
[l]ist [line] [ctx]  In this form [ctx] specifies how many lines
                     to show before/after [line].
[w]hole              List all source code. Alias for 'list 1 1000000'.
[p]rint              Show all the local variables.
[p]rint <var>        Show the value of the specified variable.
                     Can also show global vars KEYS and ARGV.
[b]reak              Show all breakpoints.
[b]reak <line>       Add a breakpoint to the specified line.
[b]reak -<line>      Remove breakpoint from the specified line.
[b]reak 0            Remove all breakpoints.
[t]race              Show a backtrace.
[e]val <code>        Execute some Lua code (in a different callframe).
[r]edis <cmd>        Execute a Redis command.
[m]axlen [len]       Trim logged Redis replies and Lua var dumps to len.
                     Specifying zero as <len> means unlimited.
[a]bort              Stop the execution of the script. In sync
                     mode dataset changes will be retained.

Debugger functions you can call from Lua scripts:
redis.debug()        Produce logs in the debugger console.
redis.breakpoint()   Stop execution as if there was a breakpoint in the
                     next line of code.
```

请注意，当您启动调试器时，它将以**步进模式**启动。
它将在脚本实际执行之前停止在第一行执行操作的代码。

通常情况下，您需要调用`step`来执行当前行并进入下一行。
在您执行`step`的同时，Redis会显示服务器执行的所有命令，如以下示例所示：

```
* Stopped at 1, stop reason = step over
-> 1   redis.call('ping')
lua debugger> step
<redis> ping
<reply> "+PONG"
* Stopped at 2, stop reason = step over
```

这个`<redis>`和`<reply>`行显示了刚刚执行的命令和服务器的回复。请注意，这只会在步进模式下发生。
如果使用`continue`命令来执行脚本直到下一个断点，为了防止输出太多，在屏幕上不会显示命令的转储信息。

## 调试会话的终止


当脚本自然终止时，调试会话将结束，`redis-cli` 会返回到正常的非调试模式。您可以像往常一样使用 `restart` 命令重新启动会话。

停止调试会话的另一种方法是通过手动按下 "Ctrl+C" 来中断 `redis-cli`。请注意，任何中断 `redis-cli` 和 `redis-server` 之间连接的事件也将中断调试会话。

当服务器关闭时，所有分叉的调试会话都会被终止。

## 缩写调试命令

调试可能是一项非常重复的任务。因此，每个Redis调试器命令都以不同的字符开头，并且您可以使用单个初始字符来引用该命令。

所以例如，你可以只输入`s`，而不是输入`step`。

# 断点

添加和删除断点非常简单，如在线帮助中所述。
只需使用`b 1 2 3 4`在第1、2、3、4行添加断点。
命令`b 0`可删除所有断点。可以使用带有减号前缀的行参数来删除所选断点。
例如，`b -3`将删除第3行的断点。

请注意，为Lua从未执行的代码行（如局部变量的声明或注释）添加断点是无效的。
虽然断点会被添加，但由于脚本的这部分永远不会被执行，程序将永远不会停止。

## 动态断点

使用`breakpoint`命令可以在特定行中添加断点。然而，有时我们只希望在发生特定情况时停止程序的执行。为了实现这个目的，您可以在Lua脚本中使用`redis.breakpoint()`函数。当调用它时，它会在下一行执行时模拟一个断点。

```
if counter > 10 then redis.breakpoint() end
```
这个功能在调试时非常有用，可以避免多次手动执行脚本直到满足特定条件为止。

## 同步模式

如前所述，默认情况下LDB使用分叉会话，并在调试期间回滚脚本操作的所有数据更改。在调试过程中具有确定性通常是一件好事，这样就可以在无需将数据库内容重置为其原始状态的情况下启动连续的调试会话。

然而，为了追踪某些错误，您可能希望保留每个调试会话对键空间所做的更改。当这是一个好主意时，您应该在 `redis-cli` 中使用特殊选项 `ldb-sync-mode` 来启动调试器。

```
./redis-cli --ldb-sync-mode --eval /tmp/script.lua
```

注意：在调试模式下，Redis服务器将无法访问，因此请谨慎使用。

在这种特殊模式下，`abort`命令可以在操作数据集时停止脚本的执行。
请注意，这与正常结束调试会话是不同的。
如果你只是中断 `redis-cli`，脚本将被完全执行，然后会话被终止。
相反，使用 `abort` 可以在脚本执行过程中中断，并在需要时开始新的调试会话。

## 从脚本中记录日志

`redis.debug（）`命令是一个强大的调试工具，可以在Redis Lua脚本中调用，将信息记录到调试控制台中：

```
lua debugger> list
-> 1   local a = {1,2,3}
   2   local b = false
   3   redis.debug(a,b)
lua debugger> continue
<debug> line 3: {1; 2; 3}, false
```

如果脚本在调试会话外执行，`redis.debug()` 根本没有任何效果。
请注意，该函数接受多个参数，输出时使用逗号和空格进行分隔。

为了便于程序员调试脚本，表格和嵌套表格将正确显示以使值易于观察。

## 使用 `print` 和 `eval` 检查程序状态


在 Lua 脚本内部，可以使用 `redis.debug()` 函数直接打印值，但是在进行步进调试或者停在断点时，观察程序的局部变量往往更有用。

`print`命令就是这样操作，并在调用堆栈中从当前帧回溯到前面的帧，直到顶层。这意味着即使我们在Lua脚本的嵌套函数中，我们仍然可以使用`print foo`来查看`foo`在调用函数的上下文中的值。当没有变量名时调用`print`，将打印所有变量及其对应的值。

`eval` 命令在**不在当前调用帧的上下文中评估**的情况下执行小片Lua脚本（使用当前Lua内部不可能在当前调用帧的上下文中评估）。
然而，您可以使用此命令来测试Lua函数。

```
lua debugger> e redis.sha1hex('foo')
<retval> "0beec7b5ea3f0fdbc95d0dd47f3c5bc275da8a33"
```

## 调试客户端

LDB使用客户端服务器模型，其中Redis服务器充当使用[RESP](/topics/protocol)进行通信的调试服务器。虽然`redis-cli`是默认的调试客户端，但只要满足以下条件之一，就可以使用任何[客户端](/clients)进行调试：

1. 客户端提供了原生接口，用于设置调试模式和控制调试会话。
2. 客户端提供了一个接口，用于通过 RESP 发送任意命令。
3. 客户端允许向 Redis 服务器发送原始消息。

例如，为[ZeroBrane Studio](http://studio.zerobrane.com/)开发的[Redis插件](https://redis.com/blog/zerobrane-studio-plugin-for-redis-lua-scripts)利用[redis-lua](https://github.com/nrk/redis-lua)与LDB集成。下面的Lua代码是插件实现此功能的简化示例：

```Lua
local redis = require 'redis'

-- add LDB's Continue command
redis.commands['ldbcontinue'] = redis.command('C')

-- script to be debugged
local script = [[
  local x, y = tonumber(ARGV[1]), tonumber(ARGV[2])
  local result = x * y
  return result
]]

local client = redis.connect('127.0.0.1', 6379)
client:script("DEBUG", "YES")
print(unpack(client:eval(script, 0, 6, 9)))
client:ldbcontinue()
```
