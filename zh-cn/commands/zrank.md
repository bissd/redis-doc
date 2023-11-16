以从低到高排序的得分，返回存储在`key`中的有序集合中`member`的排名。
排名（或索引）是从0开始的，这意味着得分最低的成员的排名为`0`。

可选的 `WITHSCORE` 参数会在命令的回复中附加返回元素的分数。

使用`ZREVRANK`命令按照从高到低的分数顺序获取元素的排名。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZRANK myzset "three"
ZRANK myzset "four"
ZRANK myzset "three" WITHSCORE
ZRANK myzset "four" WITHSCORE
```
