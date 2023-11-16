`BZPOPMAX`是sorted set `ZPOPMAX`原语的阻塞变体。

这是阻塞版本，因为当没有成员可以从给定的有序集合中弹出时，它会阻塞连接。
从第一个非空的有序集合中弹出具有最高分数的成员，按照给定的键的顺序进行检查。

`timeout`参数被解释为双精度值，指定最大阻塞时间（以秒为单位）。0表示可以无限期阻塞。

有关确切语义，请参阅[BZPOPMIN文档][cb]，因为`BZPOPMAX`与`BZPOPMIN`完全相同，唯一的区别在于它弹出具有最高分数的成员，而不是弹出具有最低分数的成员。

[cb]：/commands/bzpopmin

@examples

```
redis> DEL zset1 zset2
(integer) 0
redis> ZADD zset1 0 a 1 b 2 c
(integer) 3
redis> BZPOPMAX zset1 zset2 0
1) "zset1"
2) "c"
3) "2"
```
