这个命令与`SUNION`相同，但不返回结果集，而是将结果集存储在`destination`中。

如果 `destination` 已存在，则被覆盖。

@examples

```cli
SADD key1 "a"
SADD key1 "b"
SADD key1 "c"
SADD key2 "c"
SADD key2 "d"
SADD key2 "e"
SUNIONSTORE key key1 key2
SMEMBERS key
```
