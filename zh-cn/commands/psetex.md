`PSETEX` 的工作原理与 `SETEX` 完全相同，唯一的区别是过期时间以毫秒为单位进行指定。

@examples

```cli
PSETEX mykey 1000 "Hello"
PTTL mykey
GET mykey
```
