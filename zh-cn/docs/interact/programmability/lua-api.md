---
title: "Redis Lua API参考"
linkTitle: "Lua API"
weight: 3
description: >
   在Redis中执行Lua
aliases:
    - /topics/lua-api
    - /docs/manual/programmability/lua-api/
---

Redis包含一个嵌入式的[Lua 5.1](https://www.lua.org/)解释器。
解释器运行用户定义的[临时脚本](/topics/eval-intro)和[函数](/topics/functions-intro)。脚本在受限的环境中运行，只能访问特定的Lua包。本页描述了执行环境中可用的包和API。

## 沙盒环境上下文

沙盒化的Lua上下文尝试防止意外的误用，并减少来自服务器环境的潜在威胁。

脚本应该禁止尝试访问Redis服务器的底层主机系统。
这包括文件系统、网络，以及任何试图执行系统调用的尝试，除了API支持的那些调用。

脚本应仅对存储在Redis中的数据和提供给其执行的参数数据进行操作。

全局变量和函数

沙盒化的Lua执行环境禁止声明全局变量和函数。
禁止全局变量的目的是确保脚本和函数不会尝试维护除Redis存储的数据之外的任何运行时上下文。
在（相对不常见的）使用情况下，如果需要在执行之间维护上下文，
应该将上下文存储在Redis的key空间中。

Redis 将在尝试执行以下代码片段时返回一个“脚本试图创建全局变量'my_global_variable'”错误:

```lua
my_global_variable = 'some value'
```

以下是全局函数声明的类似方式：

```lua
function my_global_function()
  -- Do something amazing
end
```

当脚本尝试访问运行时上下文中未定义的任何全局变量时，您也会收到类似的错误提示：

```lua
-- The following will surely raise an error
return an_undefined_global_variable
```

```lua
local my_local_variable = 'some value'

local function my_local_function()
  -- Do something else, but equally amazing
end
```

**注意：**
沙箱试图防止使用全局变量。
使用Lua的调试功能或其他方法，比如修改用于实现全局保护的元表，以绕过沙箱并不难。
但是，意外绕过保护是困难的。
如果用户搞乱了Lua全局状态，则无法保证AOF和复制的一致性。
换句话说，就是不要这样做。

### 已导入的Lua模块

在沙盒执行上下文中不支持使用导入的 Lua 模块。
沙盒执行上下文通过禁用 Lua 的 [`require` 函数](https://www.lua.org/pil/8.1.html) 来防止加载模块。

Redis仅随附并可在脚本中使用的库列在 [运行时库](#runtime-libraries) 部分下。

##运行时全局变量

在沙盒中，用户无法声明全局变量，但执行上下文预先加载了其中的一些全局变量。

### _redis_ 单例

_redis_ 单例是一个可以从所有脚本中访问的对象实例。
它提供了与 Redis 进行交互的 API。
它的描述如下 [下方](#redis_object)。

### <a name="the-keys-global-variable"></a>_KEYS_全局变量

* 自版本：2.6.0
* 可在脚本中使用：是
* 可在函数中使用：否

**重要提示：**
为了确保脚本正确执行，无论是独立部署还是集群部署，函数访问的所有键名都必须作为输入键参数明确提供。
脚本**仅应**访问其输入参数中提供的键。
脚本**永远不应该**访问由程序自动生成的键名或基于数据库中存储的数据结构内容的键。

全局变量 _KEYS_ 仅在 [临时脚本](/topics/eval-intro) 中可用。
它已预设为所有键名输入参数。

### <a name="the-argv-global-variable"></a>_ARGV_ 全局变量

* 从版本：2.6.0开始
* 在脚本中可用：是
* 在函数中可用：否

_ARGV_ 全局变量仅可在 [临时脚本](/topics/eval-intro) 中使用。
该变量预先填充了所有常规输入参数。

## <a name="redis_object"></a>_redis_对象

* 自从版本：2.6.0
* 可用于脚本：是
* 可用于函数：是

Redis Lua 执行上下文始终提供一个名为 _redis_ 的单例对象实例。
_redis_ 实例使脚本能够与运行脚本的 Redis 服务器交互。
以下是 _redis_ 对象实例所提供的 API。

### <a name="redis.call"></a> `redis.call(command [,arg...])`

### <a name="redis.call"></a> `redis.call(command[,arg...])`

* 自版本：2.6.0
* 可用于脚本：是
* 可用于函数：是

`redis.call()`函数调用给定的Redis命令并返回其响应。
它的输入是命令和参数，一旦调用，它会在Redis中执行该命令并返回响应。

例如，我们可以从脚本中调用`ECHO`命令并返回其回复，如下所示：

```lua
return redis.call('ECHO', 'Echo, echo... eco... o...')
```

如果`redis.call（）`触发运行时异常，则原始异常将自动作为错误返回给用户。
因此，尝试执行以下临时脚本将失败并生成运行时异常，因为`ECHO`只接受一个参数：

```lua
redis> EVAL "return redis.call('ECHO', 'Echo,', 'echo... ', 'eco... ', 'o...')" 0
(error) ERR Wrong number of args calling Redis command from script script: b0345693f4b77517a711221050e76d24ae60b7f7, on @user_script:1.
```

注意，调用可能因各种原因失败，请参阅[低内存条件下的执行](/topics/eval-intro#execution-under-low-memory-conditions)和[脚本标志](#script_flags)。

处理Redis运行时错误时，请改用`redis.pcall()`。

### <a name="redis.pcall"></a> `redis.pcall(command [,arg...])`
### <a name="redis.pcall"></a> `redis.pcall(command [,arg...])`

* 自版本：2.6.0
* 可用于脚本：是
* 可用于函数：是

此功能可以处理由Redis服务器触发的运行时错误。
`redis.pcall()`函数的行为与[`redis.call()`](#redis.call)完全相同，唯一的区别是它：

* 始终返回一个回复。
* 从不抛出运行时异常，如果服务器抛出运行时异常，则返回[`redis.error_reply`](#redis.error_reply)。

下面演示了如何使用`redis.pcall()`来拦截并处理临时脚本内的运行时异常。

```lua
local reply = redis.pcall('ECHO', unpack(ARGV))
if reply['err'] ~= nil then
  -- Handle the error sometime, but for now just log it
  redis.log(redis.LOG_WARNING, reply['err'])
  reply['err'] = 'ERR Something is wrong, but no worries, everything is under control'
end
return reply
```

用多个参数评估此脚本将返回：

```
redis> EVAL "..." 0 hello world
(error) ERR Something is wrong, but no worries, everything is under control
```

### <a name="redis.error_reply"></a> `redis.error_reply(x)`
### <a name="redis.error_reply"></a> `redis.error_reply(x)`

* 自版本：2.6.0
* 可用于脚本：是
* 可用于函数：是

这是一个辅助函数，它返回一个[错误回复](/topics/protocol#resp-errors)。
该辅助函数接受一个字符串参数，并返回一个 Lua 表，其中_err_字段设置为该字符串。

以下代码的结果是，对于所有的意图来说，_error1_ 和 _error2_ 是相同的意义。

```lua
local text = 'ERR My very special error'
local reply1 = { err = text }
local reply2 = redis.error_reply(text)
```

因此，这两种形式都是作为从脚本返回错误回复的有效方式：

```
redis> EVAL "return { err = 'ERR My very special table error' }" 0
(error) ERR My very special table error
redis> EVAL "return redis.error_reply('ERR My very special reply error')" 0
(error) ERR My very special reply error
```

关于返回Redis状态回复，请参考[`redis.status_reply()`](#redis.status_reply)。
有关返回其他响应类型，请参考[数据类型转换](#data-type-conversion)。

**注意:**
根据惯例，Redis使用错误字符串的第一个单词作为特定错误的唯一错误代码，或者对于通用错误使用`ERR`。
建议脚本遵循这个惯例，如上面的示例所示，但这不是强制性的。

### <a name="redis.status_reply"></a> `redis.status_reply(x)`

### <a name="redis.status_reply"></a> `redis.status_reply(x)`

* 自版本: 2.6.0
* 可用于脚本:是
* 可用于函数:是

这是一个辅助函数，它返回一个[简单的字符串回复](/topics/protocol#resp-simple-strings)。
"OK" 是标准Redis状态回复的一个示例。
Lua API将状态回复表示为带有单个字段_ok_的表，该字段设置为简单的状态字符串。

以下代码的结果是，_status1_和_status2_从实质上来说是相同的:

```lua
local text = 'Frosty'
local status1 = { ok = text }
local status2 = redis.status_reply(text)
```

因此，这两种形式都可以作为从脚本返回状态回复的有效方式：

```
redis> EVAL "return { ok = 'TICK' }" 0
TICK
redis> EVAL "return redis.status_reply('TOCK')" 0
TOCK
```

对于返回Redis错误回复，请参考[`redis.error_reply()`](#redis.error_reply)。
对于返回其他响应类型，请参考[数据类型转换](#data-type-conversion)。

### <a name="redis.sha1hex"></a> `redis.sha1hex(x)`


* 自版本: 2.6.0
* 可用于脚本: 是
* 可用于函数: 是

这个函数返回其单个字符串参数的SHA1十六进制摘要。

您可以，例如，获取空字符串的SHA1摘要：

```
redis> EVAL "return redis.sha1hex('')" 0
"da39a3ee5e6b4b0d3255bfef95601890afd80709"
```

### <a name="redis.log"></a> `redis.log(level, message)`
### <a name="redis.log"></a> `redis.log（级别，消息）`

* 从版本: 2.6.0
* 可用于脚本: 是
* 可用于函数: 是

这个函数写入到Redis服务器的日志中。

它需要两个输入参数：日志级别和消息。
消息是要写入日志文件的字符串。
日志级别可以是以下之一：

* `redis.LOG_DEBUG`
* `redis.LOG_VERBOSE`
* `redis.LOG_NOTICE`
* `redis.LOG_WARNING`

这些级别对应服务器的日志级别。
日志仅记录与服务器的`loglevel`配置指令相等或更高级别的消息。

以下是代码片段：

```python
def say_hello():
    print("Hello, World!")

say_hello()
```

这段代码会输出：

```
Hello, World!
```

```lua
redis.log(redis.LOG_WARNING, 'Something is terribly wrong')
```

将产生类似于以下内容的行记录在您服务器的日志中：

```
[32343] 22 Mar 15:21:39 # Something is terribly wrong
```

### <a name="redis.setresp"></a> `redis.setresp(x)`
### <a name="redis.setresp"></a> `redis.setresp(x)`

* 自版本：6.0.0
* 可用于脚本：是
* 可用于函数：是

该函数允许执行脚本在[`redis.call()`](#redis.call)和[`redis.pcall()`](#redis.pcall)返回的回复中切换到[Redis序列化协议（RESP）](/topics/protocol)的版本。
它期望一个数字参数作为协议的版本。
默认的协议版本是_2_，但可以切换到版本_3_。

以下是切换到 RESP3 回复的示例：

```lua
redis.setresp(3)
```

请参阅[数据类型转换](#data-type-conversion)以获取有关类型转换的更多信息。

### <a name="redis.set_repl"></a> `redis.set_repl(x)`


* 自版本：3.2.0
* 在脚本中可用：是
* 在函数中可用：否

**注意:**
只有在脚本效果复制启用时才可用此功能。
在使用逐字脚本复制时调用它将导致错误。
从Redis版本2.6.0起，脚本是逐字复制的，意味着脚本的源代码被发送给复制者执行，并存储在AOF中。
从版本3.2.0起，添加了一种替代的复制模式，允许仅复制脚本的效果。
从Redis版本7.0起，不再支持脚本复制，唯一可用的复制模式是脚本效果复制。

**警告:**
这是一个高级功能。滥用可能会违反将Redis主服务器、副本和AOF内容绑定为相同逻辑内容的合约，从而造成损害。

该函数允许脚本对其效果如何传播到副本和AOF进行控制。
脚本的效果是它调用的Redis写命令。

默认情况下，脚本执行的所有写入命令都会被复制。
然而，有时候对此行为有更好的控制是有帮助的。
例如，在仅主节点中存储中间值时，就可能需要这样的情况。

考虑一个脚本，使用`SUNIONSTORE`将两个集合相交，并将结果存储在临时键中。
然后，从交集中随机选择五个元素(`SRANDMEMBER`)，并将它们存储(`SADD`)到另一个集合中。
最后，在返回之前，删除存储两个源集合交集的临时键。

在这种情况下，只需要复制带有其五个随机选择的元素的新集合。
复制`SUNIONSTORE`命令和临时键的`DEL`操作是不必要和浪费的。

`redis.set_repl()` 函数指示服务器如何处理后续的写入命令以进行复制。
它接受一个输入参数，只能是以下之一：

* `redis.REPL_ALL`：将效果复制到AOF和副本。
* `redis.REPL_AOF`：仅将效果复制到AOF。
* `redis.REPL_REPLICA`：仅将效果复制到副本。
* `redis.REPL_SLAVE`：与`REPL_REPLICA`相同，为了向后兼容而保留。
* `redis.REPL_NONE`：完全禁用效果复制。

默认情况下，脚本引擎在脚本开始执行时初始化为 `redis.REPL_ALL` 设置。
您可以在脚本执行期间随时调用 `redis.set_repl()` 函数来在不同的复制模式之间切换。

一个简单的例子如下所示：

```lua
redis.replicate_commands() -- Enable effects replication in versions lower than Redis v7.0
redis.call('SET', KEYS[1], ARGV[1])
redis.set_repl(redis.REPL_NONE)
redis.call('SET', KEYS[2], ARGV[2])
redis.set_repl(redis.REPL_ALL)
redis.call('SET', KEYS[3], ARGV[3])
```

如果您通过调用`EVAL "..." 3 A B C 1 2 3`运行此脚本，结果将是只在副本和AOF上创建键_A_和_C_。

### <a name="redis.replicate_commands"></a> `redis.replicate_commands()`
### <a name="redis.replicate_commands"></a> `redis.replicate_commands()`

* 自版本: 3.2.0
* 至版本: 7.0.0
* 可用于脚本: 是
* 可用于函数: 否

该功能将脚本的复制模式从逐字复制切换到效果复制。您可以使用它来覆盖Redis在7.0版本之前使用的默认逐字复制脚本复制模式。

**注意：**
从Redis v7.0开始，不再支持逐字脚本复制。
默认的并且唯一支持的脚本复制模式是脚本效果的复制。
有关更多信息，请参阅[`复制命令而不是脚本`](/topics/eval-intro#replicating-commands-instead-of-scripts)。

### <a name="redis.breakpoint"></a>  `redis.breakpoint()`

### <a name="redis.breakpoint"></a>  `redis.breakpoint()`

* 自版本: 3.2.0
* 可在脚本中使用: 是
* 可在函数中使用: 否

此函数在使用[Redis Lua调试器](/topics/ldb)时触发断点。

### <a name="redis.debug"></a> `redis.debug(x)`

在调试模式下，执行用户定义的调试命令。

* 自版本：3.2.0
* 可用于脚本：是
* 可用于函数：否

这个函数在[Redis Lua调试器](/topics/ldb)控制台中打印其参数。

### <a name="redis.acl_check_cmd"></a> `redis.acl_check_cmd(command [,arg...])`

* 从版本：7.0.0开始
* 在脚本中可用：是
* 在函数中可用：是

该函数用于检查当前运行脚本的用户是否具有执行给定命令及给定参数的[ACL](/topics/acl)权限。

返回值是一个布尔值，如果当前用户有权限执行该命令（通过调用[redis.call](#redis.call)或[redis.pcall](#redis.pcall)）则为`true`，否则为`false`。

如果传递的命令或其参数无效，该函数将引发错误。

### <a name="redis.register_function"></a> `redis.register_function`
### <a name="redis.register_function"></a> `redis.register_function`

* 从版本：7.0.0 开始
* 可用于脚本：否
* 可用于函数：是

这个函数仅在`FUNCTION LOAD`命令的上下文中可用。
调用时，它将一个函数注册到已加载的库中。
该函数可以使用位置参数或命名参数进行调用。

#### <a name="redis.register_function_pos_args"></a> 位置参数: `redis.register_function(name, callback)`

`redis.register_function` 的第一个参数是一个代表函数名的 Lua 字符串。
`redis.register_function` 的第二个参数是一个 Lua 函数。

使用示例：

```
redis> FUNCTION LOAD "#!lua name=mylib\n redis.register_function('noop', function() end)"
```

#### <a name="redis.register_function_named_args"></a> 命名参数:  `redis.register_function{function_name=name, callback=callback, flags={flag1, flag2, ..}, description=description}`

`named_arguments` 函数接受以下参数：

* _function\_name_: 函数名.
* _callback_: 函数的回调.
* _flags_: 字符串数组，每个字符串代表一个函数标志（可选）.
* _description_: 函数描述（可选）.

两者`_function\_name_`和`_callback_`都是必填的。

用法示例：

```
redis> FUNCTION LOAD "#!lua name=mylib\n redis.register_function{function_name='noop', callback=function() end, flags={ 'no-writes' }, description='Does nothing'}"
```

#### <a name="script_flags"></a> 脚本标记

**重要提示：**
请谨慎使用脚本标志，错误使用可能会产生负面影响。
注意，Eval 脚本的默认值与下面提到的函数的默认值不同，请参阅[Eval 标志](/docs/manual/programmability/eval-intro/#eval-flags)。

当您注册一个函数或加载一个Eval脚本时，服务器不知道它如何访问数据库。
默认情况下，Redis假设所有脚本都会读写数据。
这导致以下行为：

1. 它们可以读取和写入数据。
1. 它们可以在集群模式下运行，但不能运行访问不同哈希插槽键的命令。
1. 为了避免不一致的读取，不允许对过时的副本进行执行。
1. 不允许在内存不足的情况下执行，以避免超过配置的阈值。

您可以使用以下标志，并指示服务器以不同的方式处理脚本的执行：

* `no-writes`：此标志表示脚本仅读取数据而不写入。

    By default, Redis will deny the execution of flagged scripts (Functions and Eval scripts with [shebang](/topics/eval-intro#eval-flags)) against read-only replicas, as they may attempt to perform writes.
    Similarly, the server will not allow calling scripts with `FCALL_RO` / `EVAL_RO`.
    Lastly, when data persistence is at risk due to a disk error, execution is blocked as well.

    Using this flag allows executing the script:
    1. With `FCALL_RO` / `EVAL_RO`
    2. On read-only replicas.
    3. Even if there's a disk error (Redis is unable to persist so it rejects writes).
    4. When over the memory limit since it implies the script doesn't increase memory consumption (see `allow-oom` below)

    However, note that the server will return an error if the script attempts to call a write command.
    Also note that currently `PUBLISH`, `SPUBLISH` and `PFCOUNT` are also considered write commands in scripts, because they could attempt to propagate commands to replicas and AOF file.

    For more information please refer to [Read-only scripts](/docs/manual/programmability/#read-only_scripts)

* `allow-oom`：使用此标志允许在服务器内存不足（OOM）时执行脚本。

    Unless used, Redis will deny the execution of flagged scripts (Functions and Eval scripts with [shebang](/topics/eval-intro#eval-flags)) when in an OOM state.
    Furthermore, when you use this flag, the script can call any Redis command, including commands that aren't usually allowed in this state.
    Specifying `no-writes` or using `FCALL_RO` / `EVAL_RO` also implies the script can run in OOM state (without specifying `allow-oom`)

* `allow-stale`：一个标志，当配置`replica-serve-stale-data`设置为`no`时，它允许在陈旧的副本上运行带有[shebang](/topics/eval-intro#eval-flags)的标记脚本（函数和Eval脚本）。

    Redis can be set to prevent data consistency problems from using old data by having stale replicas return a runtime error.
    For scripts that do not access the data, this flag can be set to allow stale Redis replicas to run the script.
    Note however that the script will still be unable to execute any command that accesses stale data.

* `no-cluster`：该标志在 Redis 集群模式下使脚本返回错误。

    Redis allows scripts to be executed both in standalone and cluster modes.
    Setting this flag prevents executing the script against nodes in the cluster.

`allow-cross-slot-keys`：允许脚本访问多个槽位的标记。

    Redis typically prevents any single command from accessing keys that hash to multiple slots.
    This flag allows scripts to break this rule and access keys within the script that access multiple slots.
    Declared keys to the script are still always required to hash to a single slot.
    Accessing keys from multiple slots is discouraged as applications should be designed to only access keys from a single slot at a time, allowing slots to move between Redis servers.
    
    This flag has no effect when cluster mode is disabled.

请参考[功能标志](/docs/manual/programmability/functions-intro/#function-flags)和[评估标志](/docs/manual/programmability/eval-intro/#eval-flags)以获取详细示例。

### <a name="redis.redis_version"></a> `redis.REDIS_VERSION`

### <a name="redis.redis_version"></a> `redis.REDIS_VERSION`

* 自版本：7.0.0
* 脚本可用：是
* 函数可用：是

以Lua字符串形式返回当前Redis服务器版本。
回复的格式为`MM.mm.PP`，其中：

* **MM：**是主版本号。
* **mm：**是次版本号。
* **PP：**是补丁级别。

### <a name="redis.redis_version_num"></a> `redis.REDIS_VERSION_NUM`
### <a name="redis.redis_version_num"></a> `redis.REDIS_VERSION_NUM`

* 自版本：7.0.0
* 可用于脚本：是
* 可用于函数：是

以数字形式返回当前Redis服务器的版本。
回复是一个十六进制值，结构为`0x00MMmmPP`，其中：

* **MM:** 是主要版本。
* **mm:** 是次要版本。
* **PP:** 是补丁级别。

## 数据类型转换

除非引发运行时异常，否则`redis.call（）`和`redis.pcall（）`会返回Lua脚本执行命令的回复。
Redis从这些函数返回的回复会自动转换为Lua的原生数据类型。

类似地，当Lua脚本使用`return`关键字返回一个回复时，该回复会自动转换为Redis的协议格式。

换句话说，Redis的回复与Lua的数据类型之间存在一对一的映射，Lua的数据类型与[Redis协议](/topics/protocol)数据类型之间也存在一对一的映射。
底层设计的原理是，如果将Redis类型转换为Lua类型，再将其转换回Redis类型，结果与初始值相同。

从 Redis 协议回复（即来自 `redis.call()` 和 `redis.pcall()` 的回复）到 Lua 数据类型的类型转换取决于脚本所使用的 Redis 序列化协议版本。
脚本执行期间的默认协议版本是 RESP2。
脚本可以通过调用 `redis.setresp()` 函数来切换回复的协议版本。

类型转换取决于用户选择的协议（参见`HELLO`命令）。

以下各节描述了 Lua 和 Redis 之间的类型转换规则，根据协议的版本。

### RESP2到Lua类型转换

以下类型转换规则适用于执行上下文的默认情况，以及调用 `redis.setresp(2)` 之后的情况：

* [RESP2整数回复](/topics/protocol#resp-integers) -> Lua数字
* [RESP2批量字符串回复](/topics/protocol#resp-bulk-strings) -> Lua字符串
* [RESP2数组回复](/topics/protocol#resp-arrays) -> Lua表格（可能嵌套其他Redis数据类型）
* [RESP2状态回复](/topics/protocol#resp-simple-strings) -> Lua表格，包含一个名为_ok_的字段，该字段包含状态字符串
* [RESP2错误回复](/topics/protocol#resp-errors) -> Lua表格，包含一个名为_err_的字段，该字段包含错误字符串
* [RESP2空批量回复](/topics/protocol#null-elements-in-arrays)和[空多批量回复](/topics/protocol#resp-arrays) -> Lua假布尔类型

## Lua到RESP2类型转换

下面的类型转换规则默认适用，也适用于用户调用`HELLO 2`之后：

* Lua数字 -> [RESP2整数回复](/topics/protocol#resp-integers)（将数字转换为整数）
* Lua字符串 -> [RESP批量字符串回复](/topics/protocol#resp-bulk-strings)
* Lua表（索引，非关联数组） -> [RESP2数组回复](/topics/protocol#resp-arrays)（在表中遇到的第一个Lua`nil`值处截断，如果有的话）
* Lua表只有一个_ok_字段 -> [RESP2状态回复](/topics/protocol#resp-simple-strings)
* Lua表只有一个_err_字段 -> [RESP2错误回复](/topics/protocol#resp-errors)
* Lua布尔值false -> [RESP2空批量回复](/topics/protocol#null-elements-in-arrays)

一条额外的Lua-to-Redis转换规则没有对应的Redis-to-Lua转换规则：

* Lua布尔值`true` -> [RESP2整数回复](/topics/protocol#resp-integers)，值为1。

有三个额外的规则需要注意将Lua转换为Redis数据类型：

* Lua有一个单一的数值类型，即Lua数字。
  整数和浮点数之间没有区别。
  因此，我们总是将Lua数字转换为整数回复，去掉其中的小数部分（如果有的话）。
  **如果要返回Lua浮点数，应该将其作为字符串返回**，
  就像Redis本身所做的那样（例如，`ZSCORE`命令）。
* 由于Lua的表语义， [在Lua数组中没有简单的办法来含有nil值](http://www.lua.org/pil/19.1.html)。
  因此，当Redis将Lua数组转换为RESP时，转换在遇到Lua `nil`值时停止。
* 当Lua表是一个包含键和它们对应值的关联数组时，转换后的Redis回复将**不包括它们**。

Lua 到 RESP2 类型转换示例：

```
redis> EVAL "return 10" 0
(integer) 10

redis> EVAL "return { 1, 2, { 3, 'Hello World!' } }" 0
1) (integer) 1
2) (integer) 2
3) 1) (integer) 3
   1) "Hello World!"

redis> EVAL "return redis.call('get','foo')" 0
"bar"
```

最后一个示例演示了如何在Lua中以与直接调用命令相同的方式接收和返回`redis.call()`（或`redis.pcall()`）的确切返回值。

下面的示例展示了如何处理包含nil值和键的浮点数和数组：

```
redis> EVAL "return { 1, 2, 3.3333, somekey = 'somevalue', 'foo', nil , 'bar' }" 0
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) "foo"
```

正如您所看到的，_3.333_ 的浮点值被转换为整数 _3_，_somekey_ 键及其值被省略，字符串 "bar" 未返回，因为它之前有一个 `nil` 值。

## 将RESP3转换为Lua类型

[RESP3](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md) 是 [Redis 序列化协议](/topics/protocol) 的最新版本。
从 Redis v6.0 开始，它可作为可选项选择。

执行脚本在执行过程中可以调用 [`redis.setresp`](#redis.setresp) 函数，并切换用于返回 Redis 命令的回复的协议版本（可通过 [`redis.call()`](#redis.call) 或 [`redis.pcall()`](#redis.pcall) 调用该函数）。

一旦Redis的回复采用RESP3协议，所有的[RESP2到Lua转换](#resp2-to-lua-type-conversion)规则都适用，增加如下规则：

* [RESP3映射响应](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#map-type) -> Lua表，包含一个名为_map_的Lua表，表示映射的字段和值。
* [RESP集合响应](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#set-reply) -> Lua表，包含一个名为_set_的Lua表，表示集合的元素作为字段，每个字段的Lua布尔值为`true`。
* [RESP3空值](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#null-reply) -> Lua的`nil`。
* [RESP3真值响应](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#boolean-reply) -> Lua的真布尔值。
* [RESP3假值响应](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#boolean-reply) -> Lua的假布尔值。
* [RESP3双精度响应](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#double-type) -> Lua表，包含一个名为_double_的Lua字段，表示双精度值的Lua数字。
* [RESP3大数响应](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#big-number-type) -> Lua表，包含一个名为_big_number_的Lua字段，表示大数值的Lua字符串。
* [Redis原始字符串响应](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#verbatim-string-type) -> Lua表，包含一个名为_verbatim_string_的Lua字段，表示原始字符串及其格式的Lua表，分别为_string_和_format_。

**注：**
RESP3的[大数](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#big-number-type)和[保真字符串](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#verbatim-string-type)回复仅从Redis v7.0及更高版本开始支持。
此外，目前，Redis Lua API不支持RESP3的[属性](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#attribute-type)、[流式字符串](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#streamed-strings)和[流式聚合数据类型](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#streamed-aggregate-data-types)。

### Lua to RESP3 type conversion
### Lua到RESP3类型转换

无论脚本选择在调用`redis.call()`或`redis.pcall()`时使用哪个协议版本设置回复的 [`redis.setresp()` 函数]，用户可选择用 `HELLO 3` 命令在连接中使用 RESP3。
尽管传入客户端连接的默认协议是 RESP2，但脚本应尊重用户的偏好并返回适当类型的 RESP3 回复，因此在这种情况下除了 [Lua 到 RESP2 类型转换](#lua-to-resp2-type-conversion) 部分规定的规则外，还需遵循以下规则。

* Lua布尔值 -> [RESP3布尔回复](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#boolean-reply)（请注意，这与RESP2有所不同，在RESP2中，返回Lua布尔`true`会返回数字1到Redis客户端，返回`false`会返回`null`。
* Lua表中有一个名为_map_的字段，设置为关联Lua表 -> [RESP3映射回复](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#map-type)。
* Lua表中有一个名为_set_的字段，设置为关联Lua表 -> [RESP3集合回复](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#set-type)。值可以设置为任何值，无论如何都会被丢弃。
* Lua表中只有一个名为_double_的字段，设置为关联Lua表 -> [RESP3双精度回复](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#double-type)。
* Lua空值 -> [RESP3空回复](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#null-reply)。

然而，如果连接设置为使用 RESP2 协议，即使脚本回复 RESP3 类型的响应，Redis 仍会自动将响应进行 RESP3 到 RESP2 的转换，就像处理常规命令一样。
这意味着，例如，将 RESP3 的映射类型返回给 RESP2 连接将导致回复被转换为一个平面的 RESP2 数组，其中包含交替的字段名和它们的值，而不是 RESP3 的映射。

关于脚本的其他注意事项

### 在脚本中使用`SELECT`

您可以像使用任何普通客户端连接一样，从Lua脚本中调用`SELECT`命令。
然而，在Redis 2.8.11和2.8.12版本之间，行为的一个细微方面发生了变化。
在Redis 2.8.12之前的版本中，Lua脚本选择的数据库被*设置为调用它的客户端连接的当前数据库*。
从Redis 2.8.12版本开始，Lua脚本选择的数据库仅影响脚本的执行上下文，并不修改调用脚本的客户端选择的数据库。
这个在补丁级别版本之间的语义变化是必要的，因为旧的行为与Redis的复制是不兼容的，并引入了bug。

## 运行时库

Redis Lua运行时上下文总是包含几个预先导入的库。

以下是可供使用的[标准Lua库](https://www.lua.org/manual/5.1/manual.html#5)：

* [_字符串操作（string）_库](https://www.lua.org/manual/5.1/manual.html#5.4)
* [_表操作（table）_库](https://www.lua.org/manual/5.1/manual.html#5.5)
* [_数学函数（math）_库](https://www.lua.org/manual/5.1/manual.html#5.6)

另外，以下外部库已加载并可供脚本使用：

* [_struct_库](#struct-library)
* [_cjson_库](#cjson-library)
* [_cmsgpack_库](#cmsgpack-library)
* [_bitop_库](#bitop-library)

### <a name="struct-library"></a> _struct_库

* 自版本：2.6.0
* 脚本可用：是
* 函数可用：是

_struct_ 是一个在 Lua 中用于打包和解包 C 类型结构的库。
它提供以下函数：

* [`struct.pack()`](#struct.pack)
* [`struct.unpack()`](#struct.unpack)
* [`struct.size()`](#struct.size)

所有结构的所有函数都希望它们的第一个参数是一个[格式字符串](#struct-formats)。

#### <a name="struct-formats"></a> _struct_ 格式

以下是_struct_函数的有效格式字符串：

- `'b'` - 表示有符号字节
- `'B'` - 表示无符号字节
- `'h'` - 表示有符号短整数
- `'H'` - 表示无符号短整数
- `'i'` - 表示有符号整数
- `'I'` - 表示无符号整数
- `'l'`- 表示有符号长整数
- `'L'` - 表示无符号长整数
- `'q'` - 表示有符号长长整数
- `'Q'` - 表示无符号长长整数
- `'f'` - 表示浮点数
- `'d'`- 表示双精度浮点数
- `'s'` - 表示字符串（字符序列）
- `'p'` - 表示指针（整数）
- `'P'` - 表示指针（整数或None）

* `>`：大端
* `<`：小端
* `![num]`：对齐
* `x`：填充
* `b/B`：有符号/无符号字节
* `h/H`：有符号/无符号短整数
* `l/L`：有符号/无符号长整数
* `T`：size_t
* `i/In`：有大小为_n_的有符号/无符号整数（默认为int的大小）
* `cn`：由_n_个字符组成的序列（从/到字符串）；在打包时，n == 0意味着整个字符串；在解包时，n == 0意味着使用先前读取的数字作为字符串的长度。
* `s`：以零结尾的字符串
* `f`：浮点数
* `d`：双精度浮点数
* ` `（空格）：忽略

#### <a name="struct.pack"></a> `struct.pack(x)`

该函数从值中返回一个编码为struct的字符串。
该函数接受一个[_struct_格式字符串](#struct-formats)作为第一个参数，后跟要编码的值。

使用示例：

```
redis> EVAL "return struct.pack('HH', 1, 2)" 0
"\x01\x00\x02\x00"
```

#### <a name="struct.unpack"></a> `struct.unpack(x)`
#### <a name="struct.unpack"></a> `struct.unpack(x)`

该函数从一个结构体中返回解码后的值。
它的第一个参数是一个[结构体格式字符串](#struct-formats)，接下来是编码后的结构体字符串。

使用示例：

```
redis> EVAL "return { struct.unpack('HH', ARGV[1]) }" 0 "\x01\x00\x02\x00"
1) (integer) 1
2) (integer) 2
3) (integer) 5
```

#### <a name="struct.size"></a> `struct.size(x)`
#### <a name="struct.size"></a> `struct.size(x)`

此函数返回一个结构体的大小，单位为字节。
它接受一个 [_struct_ 格式字符串](#struct-formats) 作为唯一参数。

示例：

```
redis> EVAL "return struct.size('HH')" 0
(integer) 4
```

### <a name="cjson-library"></a> _cjson_ 库

* 自版本：2.6.0
* 可用于脚本：是
* 可用于函数：是

`_cjson_` 库提供了从 Lua 快速进行 [JSON](https://json.org) 编码和解码的功能。
它提供了以下这些函数。

#### <a name="cjson.encode()"></a> `cjson.encode(x)`
#### <a name="cjson.encode()"></a> `cjson.encode(x)`

此函数将其参数提供的Lua数据类型转换为JSON编码的字符串。

用法示例：

```
redis> EVAL "return cjson.encode({ ['foo'] = 'bar' })" 0
"{\"foo\":\"bar\"}"
```

#### <a name="cjson.decode()"></a> `cjson.decode(x)`
#### <a name="cjson.decode()"></a> `cjson.decode(x)`

此函数从作为参数提供的JSON编码字符串中返回一个Lua数据类型。

用法示例：

```
redis> EVAL "return cjson.decode(ARGV[1])['foo']" 0 '{"foo":"bar"}'
"bar"
```

### <a name="cmsgpack-library"></a> _cmsgpack_ library

* 从版本：2.6.0开始
* 可用于脚本：是
* 可用于函数：是

_cmsgpack_库提供了从Lua中快速编码和解码[MessagePack](https://msgpack.org/index.html)的功能.
它提供了以下函数.

#### <a name="cmsgpack.pack()"></a> `cmsgpack.pack(x)`


这个函数返回给定参数的Lua数据类型的打包字符串编码。

使用示例：

```
redis> EVAL "return cmsgpack.pack({'foo', 'bar', 'baz'})" 0
"\x93\xa3foo\xa3bar\xa3baz"
```

#### <a name="cmsgpack.unpack()"></a> `cmsgpack.unpack(x)`


该函数从解码其输入字符串参数中返回解压缩的值。

使用示例：

```
redis> EVAL "return cmsgpack.unpack(ARGV[1])" 0 "\x93\xa3foo\xa3bar\xa3baz"
1) "foo"
2) "bar"
3) "baz"
```

### <a name="bitop-library"></a> _bit_ library

* 从版本2.8.18开始
* 可用于脚本：是
* 可用于函数：是

_bit_库提供针对数字的位运算操作。
其文档位于[Lua BitOp文档](http://bitop.luajit.org/api.html)。
它提供以下函数。

#### <a name="bit.tobit()"></a> `bit.tobit(x)`


将一个数字归一化为位运算的数值范围，并返回它。

使用示例：

```
redis> EVAL 'return bit.tobit(1)' 0
(integer) 1
```

#### <a name="bit.tohex()"></a> `bit.tohex(x [,n])`

将第一个参数转换为十六进制字符串。十六进制数字的位数由可选的第二个参数的绝对值确定。

使用示例：

```
redis> EVAL 'return bit.tohex(422342)' 0
"000671c6"
```

#### <a name="bit.bnot()"></a> `bit.bnot(x)`

返回其参数的按位“非”。

#### <a name="bit.ops"></a> `bit.bnot(x)` `bit.bor(x1 [,x2...])`, `bit.band(x1 [,x2...])` and `bit.bxor(x1 [,x2...])`

返回所有参数的按位或、按位与或按位异或。
请注意，允许多于两个参数。

使用示例：

```
redis> EVAL 'return bit.bor(1,2,4,8,16,32,64,128)' 0
(integer) 255
```

#### <a name="bit.shifts"></a> `bit.lshift(x, n)`, `bit.rshift(x, n)` and `bit.arshift(x, n)`

返回第一个参数按照第二个参数所指定的位数进行的位逻辑**左移**、位逻辑**右移**或位**算术右移**的结果。

#### <a name="bit.ro"></a> `bit.rol(x, n)` 和 `bit.ror(x, n)`

返回根据第二个参数指定的位数，第一个参数的按位左移或按位右移结果。
一侧移出的位会从另一侧移入。

#### <a name="bit.bswap()"></a> `bit.bswap(x)`

按位交换参数的字节并返回它。
这可以用于将小端字节序的32位数字转换为大端字节序的32位数字，反之亦然。
