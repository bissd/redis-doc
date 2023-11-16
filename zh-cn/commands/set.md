将`key`设置为包含字符串`value`的值。
如果`key`已经拥有一个值，则会被覆盖，不论其类型。
成功的`SET`操作将丢弃与该键关联的任何先前到期时间。

## 选项

`SET`命令支持一组选项来修改其行为：

* `EX` *seconds* -- 设置指定的过期时间，单位为秒。
* `PX` *milliseconds* -- 设置指定的过期时间，单位为毫秒。
* `EXAT` *timestamp-seconds* -- 设置指定的Unix时间作为键的到期时间，单位为秒。
* `PXAT` *timestamp-milliseconds* -- 设置指定的Unix时间作为键的到期时间，单位为毫秒。
* `NX` -- 仅在键不存在时设置键。
* `XX` -- 仅在键已存在时设置键。
* `KEEPTTL` -- 保持与键关联的生存时间。
* `!GET` -- 返回键上存储的旧字符串，如果键不存在则返回nil。如果键上存储的值不是字符串，则返回错误并中止`SET`命令。

注意：由于`SET`命令选项可以取代`SETNX`、`SETEX`、`PSETEX`、`GETSET`，所以在Redis的将来版本中可能会弃用并最终移除这些命令。

@examples

```cli
SET mykey "Hello"
GET mykey

SET anotherkey "will expire in a minute" EX 60
```

### 代码示例

{{< clients-example set_and_get />}}

## 模式

**注意：**为了提供更好的保证和容错性，建议使用[Redlock算法](https://redis.io/topics/distlock)，而不是下面的模式，尽管后者的实现可能稍微复杂一些。

使用命令`SET resource-name anystring NX EX max-lock-time`是使用Redis实现锁系统的简单方法。

如果上述命令返回`OK`，客户端可以获得锁（如果命令返回Nil，则可以在一段时间后重试），并只需使用`DEL`来删除锁。

锁定将在到达过期时间后自动释放。

有可能通过以下方式来使该系统更加稳健，即修改解锁模式：

* 不要设置固定字符串，而是设置一个不可预测的大随机字符串，称为令牌。
* 不要使用 `DEL` 来释放锁定，发送一个脚本，只有当值匹配时才删除键。

这样可以避免客户端在到期时间后尝试释放锁，从而删除由后来获得锁的另一个客户端创建的键。

解锁脚本的示例将类似于以下内容：

    if redis.call("get",KEYS[1]) == ARGV[1]
    then
        return redis.call("del",KEYS[1])
    else
        return 0
    end

脚本应该使用`EVAL ...script... 1 资源名称 令牌值`调用。
