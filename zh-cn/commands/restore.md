使用反序列化获得的提供的序列化值（通过`DUMP`获得）创建一个与值相关联的键。

如果 `ttl` 为0，则创建的键没有任何过期时间，否则设置指定的过期时间（以毫秒为单位）。

如果使用了`ABSTTL`修饰符，那么`ttl`应该表示一个绝对的Unix时间戳（毫秒），在该时间戳之后键将过期。

## 【Unix 时间】
[Unix 时间][hewowu] 是计算机科学领域中表示时间的一种方式。它是一种记录时间的方法，以表示自 1970 年 1 月 1 日 00:00:00 UTC（协调世界时）起经过的秒数。Unix 时间被广泛应用于操作系统、编程语言、数据存储以及其他与时间相关的应用中。

[hewowu]: http://zh.wikipedia.org/wiki/Unix时间

为了清理目的，可以使用`IDLETIME`或`FREQ`修饰符。请参阅 `OBJECT` 获取更多信息。

当 `key` 已经存在时，使用 `!RESTORE` 会返回 "Target key name is busy" 错误，除非您使用 `REPLACE` 修饰符。

`！RESTORE`会检查RDB版本和数据校验和。
如果它们不匹配，则返回错误。

@examples

```
redis> DEL mykey
0
redis> RESTORE mykey 0 "\n\x17\x17\x00\x00\x00\x12\x00\x00\x00\x03\x00\
                        x00\xc0\x01\x00\x04\xc0\x02\x00\x04\xc0\x03\x00\
                        xff\x04\x00u#<\xc0;.\xe9\xdd"
OK
redis> TYPE mykey
list
redis> LRANGE mykey 0 -1
1) "1"
2) "2"
3) "3"
```
