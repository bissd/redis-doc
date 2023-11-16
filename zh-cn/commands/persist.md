将`key`的现有超时时间删除，将该键从“易失性”（具有设置到期时间的键）转变为“持久性”（没有关联超时的永不过期的键）。

@examples

```cli
SET mykey "Hello"
EXPIRE mykey 10
TTL mykey
PERSIST mykey
TTL mykey
```
