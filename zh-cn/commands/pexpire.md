这个命令的功能和`EXPIRE`一样，只是键的生存时间以毫秒为单位指定，而不是秒。

## Options

`PEXPIRE`命令自Redis 7.0版本开始支持一组选项：

* `NX` - 只有在键没有过期时才设置过期时间
* `XX` - 只有在键已经设置了过期时间时才设置过期时间
* `GT` - 只有在新的过期时间大于当前过期时间时才设置过期时间
* `LT` - 只有在新的过期时间小于当前过期时间时才设置过期时间

非易失键在 `GT` 和 `LT` 的目的上被视为无限 TTL。
`GT`、`LT` 和 `NX` 选项是互斥的。

@examples

```cli
SET mykey "Hello"
PEXPIRE mykey 1500
TTL mykey
PTTL mykey
PEXPIRE mykey 1000 XX
TTL mykey
PEXPIRE mykey 1000 NX
TTL mykey
```
