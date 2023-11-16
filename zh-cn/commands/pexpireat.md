`PEXPIREAT`具有与`EXPIREAT`相同的效果和语义，但是指定键过期的Unix时间以毫秒为单位，而不是秒。

## 选项

`PEXPIREAT`命令自Redis 7.0版本支持一组选项：

* `NX` - 仅当键没有过期时设置过期时间
* `XX` - 仅当键已经存在过期时间时设置过期时间
* `GT` - 仅当新的过期时间大于当前过期时间时设置过期时间
* `LT` - 仅当新的过期时间小于当前过期时间时设置过期时间

非易失性键被视为`GT`和`LT`的无限TTL。
`GT`、`LT`和`NX`选项是互斥的。

@examples

```cli
SET mykey "Hello"
PEXPIREAT mykey 1555555555005
TTL mykey
PTTL mykey
```
