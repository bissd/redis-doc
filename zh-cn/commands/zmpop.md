从提供的键名列表中的第一个非空排序集中弹出一个或多个成员-分数对。

`ZMPOP` 和 `BZMPOP` 类似于以下更受限制的命令：

- `ZPOPMIN`或`ZPOPMAX`仅取一个键，并可以返回多个元素。
- `BZPOPMIN`或`BZPOPMAX`取多个键，但只从一个键返回一个元素。

有关此命令的屏蔽变体，请参见`BZMPOP`。

当使用`MIN`修饰词时，弹出的元素是从第一个非空排序集合中具有最低分数的元素。使用`MAX`修饰词会弹出具有最高分数的元素。
可选的`COUNT`可以用来指定要弹出的元素数量，默认设置为1。

弹出的元素数量是从排序集合的基数和`COUNT`的值中取最小值。

@examples

```cli
ZMPOP 1 notsuchkey MIN
ZADD myzset 1 "one" 2 "two" 3 "three"
ZMPOP 1 myzset MIN
ZRANGE myzset 0 -1 WITHSCORES
ZMPOP 1 myzset MAX COUNT 10
ZADD myzset2 4 "four" 5 "five" 6 "six"
ZMPOP 2 myzset myzset2 MIN COUNT 10
ZRANGE myzset 0 -1 WITHSCORES
ZMPOP 2 myzset myzset2 MAX COUNT 10
ZRANGE myzset2 0 -1 WITHSCORES
EXISTS myzset myzset2
```
