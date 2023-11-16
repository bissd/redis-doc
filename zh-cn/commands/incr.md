将存储在`key`上的数字加一。
如果键不存在，则在执行操作前将其设置为`0`。
如果键包含错误类型的值或包含无法表示为整数的字符串，则返回错误。
此操作仅适用于64位有符号整数。

**注意**：这是一项字符串操作，因为Redis没有专用的整型类型。
存储在键中的字符串被解释为一个十进制的**64位有符号整数**来执行操作。

Redis 将整数以其整数表示形式存储，因此对于实际保存整数的字符串值，存储整数的字符串表示形式不会有额外开销。

@examples

```cli
SET mykey "10"
INCR mykey
GET mykey
```

## 模式：计数器

计数器模式是使用Redis原子增量操作最直观的方法。
简单来说，每次发生操作时，只需向Redis发送一个“INCR”命令。
例如在Web应用程序中，我们可能想知道用户每天在一年中的页面浏览次数。

为了实现这一点，网络应用可以简单地在用户执行页面查看操作时递增一个键，创建键名将用户 ID 与表示当前日期的字符串连接起来。

这个简单的模式可以以多种方式进行扩展：

* 可以在每次页面访问时同时使用`INCR`和`EXPIRE`，以实现仅计算最新的N次页面访问次数，这些访问次数的时间间隔小于指定的秒数。
* 客户端可以使用GETSET来原子地获取当前计数器值并将其重置为零。
* 使用其他原子增加/减少命令，如`DECR`或`INCRBY`，可以处理根据用户执行的操作而变大或变小的值。
  想象一下在线游戏中不同用户的分数，例如。

## 模式：速率限制器

速率限制器模式是一种特殊的计数器，用于限制执行操作的速率。
这种模式的经典实现涉及限制可以针对公共API执行的请求数量。

我们提供两种使用`INCR`的模式实现，我们假设要解决的问题是限制每个IP地址每秒最多十次API调用。

## 模式：速率限制器 1

这种模式的更简单和直接的实现方式如下：

```
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
MULTI
    INCR(keyname)
    EXPIRE(keyname,10)
EXEC
current = RESPONSE_OF_INCR_WITHIN_MULTI
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    PERFORM_API_CALL()
END
```

基本上，我们为每个 IP 地址的每个不同的秒数都设置了一个计数器。
但是这些计数器总是以每 10 秒自动过期的方式递增，所以它们会在当前秒数不同时由 Redis 自动删除。

注意在每个API调用中使用`MULTI`和`EXEC`来确保我们同时增加和设置到期时间。

## 模式: 速率限制器 2

使用单个计数器的另一种实现方式稍微复杂些，需要避免竞争条件才能正确实现。
我们将研究不同的变种。

```
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
ELSE
    value = INCR(ip)
    IF value == 1 THEN
        EXPIRE(ip,1)
    END
    PERFORM_API_CALL()
END
```

计数器的创建方式是它只会在从当前秒开始的一秒内存活。
如果在同一秒内有多于10个请求，计数器将达到大于10的值，否则它将过期并重新从0开始。

**在上面的代码中存在竞争条件**。
如果由于某种原因客户端执行了`INCR`命令但未执行`EXPIRE`命令，那么该密钥将被泄露，直到我们再次看到相同的IP地址。

可以很容易地通过将`INCR`与可选的`EXPIRE`转换为Lua脚本来修复此问题，并使用`EVAL`命令发送（仅适用于Redis版本2.6以后的版本）。

```
local current
current = redis.call("incr",KEYS[1])
if current == 1 then
    redis.call("expire",KEYS[1],1)
end
```

有一种不使用脚本来解决这个问题的不同方法，可以使用 Redis 列表而不是计数器。
该实现更复杂，使用更高级的特性，但有一个优点，能够记住当前执行 API 调用的客户端的 IP 地址，根据应用程序的需要可能有用也可能没用。

```
FUNCTION LIMIT_API_CALL(ip)
current = LLEN(ip)
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    IF EXISTS(ip) == FALSE
        MULTI
            RPUSH(ip,ip)
            EXPIRE(ip,1)
        EXEC
    ELSE
        RPUSHX(ip,ip)
    END
    PERFORM_API_CALL()
END
```

`RPUSHX` 命令只有在键已存在时才会推入元素。

请注意，虽然我们在这里存在竞争，但这不是问题：
`EXISTS` 可能返回 false，但是在我们在 `MULTI` / `EXEC` 块内创建它之前，另一个客户端可能已经创建了该键。
然而，在极少数情况下，此竞争只会导致一个 API 调用被错过，因此速率限制仍然会正常工作。
