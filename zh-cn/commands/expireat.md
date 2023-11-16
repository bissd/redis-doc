`EXPIREAT`和`EXPIRE`具有相同的效果和语义，但是不同的是，它不需要指定以秒为单位的TTL（存活时间），而是接受一个绝对的[Unix时间戳][hewowu]（自1970年1月1日以来的秒数）。过去的时间戳将立即删除键。

[Unix时间戳](http://en.wikipedia.org/wiki/Unix_time)

请参考`EXPIRE`命令的文档以获取关于命令的具体语义。

## 背景

为了将相对超时转换为AOF持久模式下的绝对超时，引入了`EXPIREAT`命令。
当然，可以直接使用它来指定一个给定的键在未来的某个时间过期。

## 选项

`EXPIREAT`命令支持一组选项：

* `NX` -- 仅当键没有过期时间时设置过期时间
* `XX` -- 仅当键存在过期时间时设置过期时间
* `GT` -- 仅当新的过期时间大于当前过期时间时设置过期时间
* `LT` -- 仅当新的过期时间小于当前过期时间时设置过期时间

将非易失性键视为无穷的TTL，以满足`GT`和`LT`的目的。
`GT`、`LT`和`NX`选项是互斥的。

@examples

```cli
SET mykey "Hello"
EXISTS mykey
EXPIREAT mykey 1293840000
EXISTS mykey
```
