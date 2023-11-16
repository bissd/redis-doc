返回给定键在秒数中将过期的绝对Unix时间戳（自1970年1月1日以来）。

请参考 `PEXPIRETIME` 命令，该命令以毫秒为单位返回相同的信息。

@examples

```cli
SET mykey "Hello"
EXPIREAT mykey 33177117420
EXPIRETIME mykey
```
