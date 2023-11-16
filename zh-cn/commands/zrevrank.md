以从高到低的顺序返回存储在`key`中的有序集合中`member`的排名。
排名（或索引）从0开始，这意味着得分最高的成员的排名为`0`。

可选的 `WITHSCORE` 参数将命令的回复补充为返回的元素的分数。

使用`ZRANK`命令按照从低到高的顺序获取元素的排名。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZREVRANK myzset "one"
ZREVRANK myzset "four"
ZREVRANK myzset "three" WITHSCORE
ZREVRANK myzset "four" WITHSCORE
```
