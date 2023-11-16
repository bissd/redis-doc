`BZPOPMIN` 是有阻塞功能的有序集合操作命令 `ZPOPMIN` 的变种。

这是阻塞版本，因为当给定的排序集合中没有成员可弹出时，它会阻塞连接。
从非空的第一个排序集合中弹出具有最低分数的成员，检查的顺序与给定的键的顺序相同。

`timeout` 参数被解释为一个双精度值，用于指定最大阻塞的秒数。超时时间为零可用于无限阻塞。

有关确切语义，请参阅[BLPOP 文档][cl]，因为 `BZPOPMIN` 与 `BLPOP` 完全相同，唯一的区别是弹出的数据结构不同。

[cl]：/commands/blpop

@examples

```
redis> DEL zset1 zset2
(integer) 0
redis> ZADD zset1 0 a 1 b 2 c
(integer) 3
redis> BZPOPMIN zset1 zset2 0
1) "zset1"
2) "a"
3) "0"
```
