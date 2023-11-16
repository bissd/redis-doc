当排序集合中的所有元素使用相同的分数插入时，为了强制按字典顺序排列，该命令返回键为`key`的排序集合中，值在`max`和`min`之间的所有元素。

除了顺序相反外，`ZREVRANGEBYLEX`与`ZRANGEBYLEX`类似。

@examples

```cli
ZADD myzset 0 a 0 b 0 c 0 d 0 e 0 f 0 g
ZREVRANGEBYLEX myzset [c -
ZREVRANGEBYLEX myzset (c -
ZREVRANGEBYLEX myzset (g [aaa
```
