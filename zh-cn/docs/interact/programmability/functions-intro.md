---
title: "Redis函数"
linkTitle: "Functions"
weight: 1
description: >
   脚本化与Redis 7以及更高版本
aliases:
    - /topics/functions-intro
    - /docs/manual/programmability/functions-intro/
---

Redis Functions是用于管理在服务器上执行的代码的API。这个功能在Redis 7中可用，取代了Redis早期版本中[EVAL](/docs/manual/programmability/eval-intro)的使用。

## 序言（或者，脚本评估有什么问题？）

Redis的早期版本只能通过`EVAL`命令来使用脚本，该命令允许将Lua脚本发送给服务器进行执行。
[Eval Scripts](/topics/eval-intro) 的核心用例是在Redis中高效且原子地执行应用程序逻辑的部分。
这样的脚本可以对多个键进行有条件的更新，可能还会组合几种不同的数据类型。

使用`EVAL`命令要求应用程序每次都发送完整的脚本进行执行。
由于这会导致网络和脚本编译的开销，Redis提供了一种优化方式，即`EVALSHA`命令。通过首先调用`SCRIPT LOAD`来获取脚本的SHA1摘要，应用程序之后可以仅使用该摘要来重复调用脚本。

根据设计，Redis仅缓存已加载的脚本。
这意味着脚本缓存可能在任何时候丢失，例如在调用`SCRIPT FLUSH`后、重新启动服务器后或切换到副本时。
如果缺少任何脚本，应用程序负责在运行时重新加载脚本。
基本假设是脚本是应用程序的一部分，而不是由Redis服务器维护的。

这种方法适用于许多轻量级脚本使用场景，但是一旦应用程序变得复杂并且更加依赖脚本语言时，会引入几个困难，即：

1. 所有客户端应用程序实例必须保留所有脚本的副本。这意味着必须有一种机制将脚本更新应用到所有应用程序实例中。
2. 在[事务](/topics/transactions)的上下文中调用缓存脚本会增加事务因缺少脚本而失败的概率。由于更容易失败，缓存脚本作为工作流程的构建块使用的吸引力较低。
3. SHA1散列值是无意义的，使得调试系统变得极其困难（例如在`MONITOR`会话中）。
4. 当被简单地使用时，`EVAL`会促使一个反模式的产生，即客户端应用程序逐字渲染脚本，而不是负责地使用[`!KEYS`和`ARGV` Lua API](/topics/lua-api#runtime-globals)。
5. 由于脚本是短暂的，无法调用另一个脚本。这使得在脚本之间共享和重用代码几乎不可能，除非在客户端进行预处理（参见第一点）。

为了满足这些需求，同时避免对已经建立和喜欢的短暂脚本造成破坏性变化，Redis v7.0引入了Redis函数。

## 什么是Redis函数？

Redis函数是从临时脚本编写的进化。

函数提供与脚本相同的核心功能，但是它们是数据库的一级软件工件。
Redis 将函数作为数据库的一部分进行管理，并通过数据持久性和复制来确保它们的可用性。
因为函数是数据库的一部分，所以在使用之前声明，应用程序在运行时不需要加载它们，也不会出现中断的事务风险。
使用函数的应用程序只依赖于它们的 API，而不依赖于数据库中嵌入的脚本逻辑。

鉴于短暂脚本被视为应用程序的一部分，函数则通过用户提供的逻辑扩展数据库服务器本身。
它们可以用来暴露一个更丰富的 API，由核心 Redis 命令组成，类似于模块，只需开发一次，启动时加载，然后由各种应用程序/客户端重复使用。
每个函数都有一个唯一的用户定义的名称，这样更容易调用和跟踪它的执行。

Redis函数的设计还试图区分编写函数和服务器管理函数所使用的编程语言。
Lua是Redis当前支持的唯一嵌入式执行引擎的语言解释器，它被设计为简单易学。
然而，选择Lua作为语言仍然对许多Redis用户构成了挑战。

Redis函数功能对实现的语言没有做任何假设。
函数定义中的执行引擎负责运行函数。
理论上，执行引擎可以以任何语言执行函数，只要它遵守一些规则（比如能够终止正在执行的函数）。

如上所述，目前 Redis 配套了一个单独的内嵌 [Lua 5.1](/topics/lua-api) 引擎。
未来计划支持更多的引擎。
Redis 函数可以利用 Lua 提供的所有功能来执行临时脚本，
唯一的例外是 [Redis Lua 脚本调试器](/topics/ldb)。

函数还通过实现代码共享来简化开发。每个函数都属于一个单独的库，一个库可以包含多个函数。库的内容是不可变的，不允许对其函数进行选择性更新。而是以一次操作将整个库一起更新。这样可以在同一个库的其他函数中调用函数，或者通过在库内部方法中使用通用代码，在函数之间共享代码，这个通用代码也可以使用语言本地的参数。

函数旨在更好地支持通过逻辑模式保持数据实体的一致视图的使用场景，如上所述。
因此，函数与数据本身存储在一起。
函数也会被持久化到AOF文件，并从主节点复制到副本节点，因此它们与数据本身一样持久。
当Redis用作临时缓存时，需要额外的机制（如下所述）来使函数更持久。

就像Redis中的所有其他操作一样，函数的执行是原子的。
函数的执行在其整个时间内阻塞了所有服务器活动，与[事务](/topics/transactions)的语义类似。
这些语义意味着脚本的所有效果要么还没有发生，要么已经发生了。
执行的函数的阻塞语义始终适用于所有连接的客户端。
由于运行函数将阻塞Redis服务器，因此函数的执行应尽快完成，所以应避免使用长时间运行的函数。

## 加载库和函数

让我们通过一些具体的示例和 Lua 代码片段来探索 Redis 函数。

如果您对Lua以及在Redis中使用Lua没有了解，您可能会受益于阅读 [Introduction to Eval Scripts](/topics/eval-intro) 和 [Lua API](/topics/lua-api) 页面中的一些示例，以更好地掌握这门语言。

每个Redis函数都属于一个单独的库，该库被加载到Redis中。
将库加载到数据库中是使用`FUNCTION LOAD`命令完成的。
该命令将库负载作为输入参数，并且库负载必须以Shebang语句开头，该语句提供有关库的元数据（如使用的引擎和库名称）。
Shebang的格式如下：
```
#!<引擎名称> name=<库名称>
```

让我们尝试加载一个空的库：

```
redis> FUNCTION LOAD "#!lua name=mylib\n"
(error) ERR No functions registered
```

预计会出现错误，因为加载的库中没有函数。每个库都需要包含至少一个注册函数以成功加载。
注册函数具有名称，并且作为库的入口点。
当目标执行引擎处理“FUNCTION LOAD”命令时，会注册库中的函数。

当加载时，Lua引擎会编译和评估库源代码，并期望通过调用`redis.register_function()` API来注册函数。

以下是一个示例，演示了一个简单的库注册了一个名为_knockknock_的函数，并返回一个字符串回复:

```lua
#!lua name=mylib
redis.register_function(
  'knockknock',
  function() return 'Who\'s there?' end
)
```

在上面的示例中，我们针对Lua的`redis.register_function()` API提供了两个参数：它的注册名称和一个回调函数。

我们可以加载我们的库并使用`FCALL`来调用注册的函数：

```
redis> FUNCTION LOAD "#!lua name=mylib\nredis.register_function('knockknock', function() return 'Who\\'s there?' end)"
mylib
redis> FCALL knockknock 0
"Who's there?"
```

注意`FUNCTION LOAD`命令会返回加载的库的名称，该名称随后可以用于`FUNCTION LIST`和`FUNCTION DELETE`。

我们提供了两个参数给`FCALL`函数：函数的注册名称和数值`0`。这个数值表示接下来的键名的数量（就像`EVAL`和`EVALSHA`一样工作）。

我们将立即解释关键字和额外参数如何对函数可用。由于这个简单的示例不涉及关键字，我们暂时使用0.

### 输入键和常规参数

在我们继续下面的例子之前，理解Redis在参数中对待键名和非键名的区别非常重要。

虽然Redis中的键名只是字符串，不同于其他字符串值，它们代表数据库中的键。
键名是Redis中的一个基本概念，也是操作Redis集群的基础。

**重要：**
为了确保在独立和集群部署中 Redis 函数的正确执行，必须将函数访问的所有键的名称作为输入键参数明确提供。

如果函数输入不是一个键的名称，则是一个常规输入参数。

现在，让我们假装我们的应用程序将一些数据存储在Redis哈希中。
我们希望有一种类似于`HSET`的方式来设置和更新所述哈希中的字段，并将最后修改时间存储在一个名为`_last_modified_`的新字段中。
我们可以实现一个函数来完成所有这些。

我们的函数将调用`TIME`来获取服务器的时钟读数，并使用新字段值和修改时间戳更新目标哈希的值。
我们将实现的函数接受以下输入参数：哈希键名和要更新的字段值对

将这些输入作为函数回调的第一个和第二个参数让Lua API for Redis函数可以访问。
回调的第一个参数是一个包含所有键名输入的Lua表。
同样，回调的第二个参数由所有常规参数组成。

以下是我们函数和其库注册的一种可能实现方式：

```lua
#!lua name=mylib

local function my_hset(keys, args)
  local hash = keys[1]
  local time = redis.call('TIME')[1]
  return redis.call('HSET', hash, '_last_modified_', time, unpack(args))
end

redis.register_function('my_hset', my_hset)
```

如果我们创建一个名为_mylib.lua_的新文件，其中包含库的定义，我们可以这样加载它（不删除源代码中有用的空白字符）：

```bash
$ cat mylib.lua | redis-cli -x FUNCTION LOAD REPLACE
```

我们在调用`FUNCTION LOAD`时添加了`REPLACE`修饰符，告诉Redis我们要覆盖现有的库定义。
否则，我们将会收到Redis的错误提示，指出库已经存在。

既然图书馆更新的代码已经加载到Redis中，我们可以继续调用我们的函数：

```
redis> FCALL my_hset 1 myhash myfield "some value" another_field "another value"
(integer) 3
redis> HGETALL myhash
1) "_last_modified_"
2) "1640772721"
3) "myfield"
4) "some value"
5) "another_field"
6) "another value"
```

在这种情况下，我们以_1_作为键名参数的数量调用了`FCALL`。这意味着函数的第一个输入参数是一个键的名称（因此包含在回调函数的`keys`表中）。在第一个参数之后，所有后续的输入参数都被视为常规参数，并构成传递给回调函数的第二个参数的`args`表。

## 扩展图书馆

我们可以向我们的库添加更多的功能来使我们的应用程序受益。
当访问Hash的数据时，我们在Hash中添加的额外元数据字段不应包含在响应中。
另一方面，我们确实希望提供一种获取给定Hash键的修改时间戳的方法。

我们将在我们的库中添加两个新的功能来实现这些目标：

1. `my_hgetall` Redis 函数将返回给定哈希键名称中的所有字段及其对应的值，不包括元数据（即 `_last_modified_` 字段）。
2. `my_hlastmodified` Redis 函数将返回给定哈希键名称的修改时间戳。

图书馆的源代码可能如下所示：

```lua
#!lua name=mylib

local function my_hset(keys, args)
  local hash = keys[1]
  local time = redis.call('TIME')[1]
  return redis.call('HSET', hash, '_last_modified_', time, unpack(args))
end

local function my_hgetall(keys, args)
  redis.setresp(3)
  local hash = keys[1]
  local res = redis.call('HGETALL', hash)
  res['map']['_last_modified_'] = nil
  return res
end

local function my_hlastmodified(keys, args)
  local hash = keys[1]
  return redis.call('HGET', hash, '_last_modified_')
end

redis.register_function('my_hset', my_hset)
redis.register_function('my_hgetall', my_hgetall)
redis.register_function('my_hlastmodified', my_hlastmodified)
```

虽然上述内容都应该很简单明了，但需要注意的是，`my_hgetall` 还调用了 [`redis.setresp(3)`](/topics/lua-api#redis.setresp)。这意味着该函数在调用 `redis.call()` 后，预期接收 [RESP3](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md) 格式的回复，而不是默认的 RESP2 协议。RESP3 提供了字典（关联数组）类型的回复，相对于 RESP2 协议而言。通过这样做，该函数能够从回复中删除特定字段（或将其设置为 `nil`，就像 Lua 表一样），对于我们来说，即 `_last_modified_` 字段。

假设你已经将库的实现保存在名为_mylib.lua_的文件中，你可以用以下代码替换它：

```bash
$ cat mylib.lua | redis-cli -x FUNCTION LOAD REPLACE
```

一旦加载完成，您可以使用`FCALL`调用库的函数：

```
redis> FCALL my_hgetall 1 myhash
1) "myfield"
2) "some value"
3) "another_field"
4) "another value"
redis> FCALL my_hlastmodified 1 myhash
"1640772721"
```

您还可以使用`FUNCTION LIST`命令获取库的详细信息：

```
redis> FUNCTION LIST
1) 1) "library_name"
   2) "mylib"
   3) "engine"
   4) "LUA"
   5) "functions"
   6) 1) 1) "name"
         2) "my_hset"
         3) "description"
         4) (nil)
      2) 1) "name"
         2) "my_hgetall"
         3) "description"
         4) (nil)
      3) 1) "name"
         2) "my_hlastmodified"
         3) "description"
         4) (nil)
```

您可以看到，很容易通过添加新功能来更新我们的库。

## 在库中重用代码

在将函数打包到由数据库管理的软件工件中之外，库还促进了代码共享。
我们可以向库中添加一个名为`check_keys()`的错误处理辅助函数，该函数可从其他函数调用。
辅助函数`check_keys()`验证输入的_keys_表是否有一个键。
成功时返回`nil`，否则返回[错误回复](/topics/lua-api#redis.error_reply)。

更新后的图书馆源代码如下：

```
```

```lua
#!lua name=mylib

local function check_keys(keys)
  local error = nil
  local nkeys = table.getn(keys)
  if nkeys == 0 then
    error = 'Hash key name not provided'
  elseif nkeys > 1 then
    error = 'Only one key name is allowed'
  end

  if error ~= nil then
    redis.log(redis.LOG_WARNING, error);
    return redis.error_reply(error)
  end
  return nil
end

local function my_hset(keys, args)
  local error = check_keys(keys)
  if error ~= nil then
    return error
  end

  local hash = keys[1]
  local time = redis.call('TIME')[1]
  return redis.call('HSET', hash, '_last_modified_', time, unpack(args))
end

local function my_hgetall(keys, args)
  local error = check_keys(keys)
  if error ~= nil then
    return error
  end

  redis.setresp(3)
  local hash = keys[1]
  local res = redis.call('HGETALL', hash)
  res['map']['_last_modified_'] = nil
  return res
end

local function my_hlastmodified(keys, args)
  local error = check_keys(keys)
  if error ~= nil then
    return error
  end

  local hash = keys[1]
  return redis.call('HGET', keys[1], '_last_modified_')
end

redis.register_function('my_hset', my_hset)
redis.register_function('my_hgetall', my_hgetall)
redis.register_function('my_hlastmodified', my_hlastmodified)
```

在将Redis中的库替换为上述内容后，您可以立即尝试新的错误处理机制：

```
127.0.0.1:6379> FCALL my_hset 0 myhash nope nope
(error) Hash key name not provided
127.0.0.1:6379> FCALL my_hgetall 2 myhash anotherone
(error) Only one key name is allowed
```


你的Redis日志文件中应该有类似于以下的行：


```
...
20075:M 1 Jan 2022 16:53:57.688 # Hash key name not provided
20075:M 1 Jan 2022 16:54:01.309 # Only one key name is allowed
```

## 集群中的功能

如前所述，Redis会自动处理将已加载的函数传播到副本的操作。
在Redis集群中，还需要将函数加载到所有集群节点。这不是由Redis集群自动处理的，需要由集群管理员来处理（如模块加载、配置设置等）。

作为函数的目标之一是与客户端应用程序分开生活，这不应该是Redis客户端库的责任部分。相反，可以使用`redis-cli --cluster-only-masters --cluster call host:port FUNCTION LOAD ...`在所有主节点上执行加载命令。

还要注意的是，`redis-cli --cluster add-node` 命令会自动将已加载的函数从现有节点传播到新节点。

## 函数和临时的 Redis 实例

在某些情况下，可能需要启动一个新的Redis服务器并预装一组预加载函数。常见的原因可能包括：

在新环境中启动Redis
重新启动一个临时（仅用于缓存）的Redis，使用函数

在这种情况下, 在Redis接受传入的用户连接和命令之前, 我们需要确保预加载的函数是可用的。

为了实现此功能，可以使用`redis-cli --functions-rdb`从现有服务器中提取函数。这会生成一个RDB文件，可以在Redis启动时加载。

## 功能标志

Redis需要在执行函数时了解一些相关信息，以便正确执行资源使用策略并保持数据一致性。

例如，Redis 需要在只读副本上使用 `FCALL_RO` 执行之前知道某个特定函数是只读的。

默认情况下，Redis假设所有函数都可以执行任意读写操作。函数标志使得可以在注册时声明更具体的函数行为。让我们看看这是如何工作的。

在我们之前的示例中，我们定义了两个只读取数据的函数。我们可以尝试使用`FCALL_RO`对只读副本执行它们。

```
redis > FCALL_RO my_hgetall 1 myhash
(error) ERR Can not execute a function with write flag using fcall_ro.
```

Redis返回此错误是因为函数理论上可以对数据库执行读写操作。
作为一种保护措施，默认情况下，Redis假设函数都这样做，因此阻止其执行。
服务器将在以下情况下返回此错误：

1. 在只读副本上使用`FCALL`执行函数。
2. 使用`FCALL_RO`来执行函数。
3. 检测到磁盘错误（Redis无法持久化，因此拒绝写入）。

在这些情况下，您可以将“no-writes”标志添加到函数的注册中，禁用防护并允许它们运行。
要使用带有标志的函数注册，请使用[命名参数](/topics/lua-api#redis.register_function_named_args)变量的`redis.register_function`变体。

来自库的更新后的注册代码片段如下所示：

```lua
redis.register_function('my_hset', my_hset)
redis.register_function{
  function_name='my_hgetall',
  callback=my_hgetall,
  flags={ 'no-writes' }
}
redis.register_function{
  function_name='my_hlastmodified',
  callback=my_hlastmodified,
  flags={ 'no-writes' }
}
```

一旦我们替换了库，Redis允许在只读副本上使用FCALL_RO同时运行`my_hgetall`和`my_hlastmodified`。

```
redis> FCALL_RO my_hgetall 1 myhash
1) "myfield"
2) "some value"
3) "another_field"
4) "another value"
redis> FCALL_RO my_hlastmodified 1 myhash
"1640772721"
```

对于完整的文档标志，请参阅[脚本标志](/topics/lua-api#script_flags)。
