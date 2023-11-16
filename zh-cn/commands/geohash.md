返回有效的 [Geohash](https://en.wikipedia.org/wiki/Geohash) 字符串，表示排序集合中一个或多个元素的位置，该集合值表示一个地理空间索引（元素是使用 `GEOADD` 添加的）。

通常情况下，Redis使用一种变种的Geohash技术来表示元素的位置，其中位置使用52位整数进行编码。与标准的Geohash编码和解码过程中使用的初始最小和最大坐标不同，编码也是不同的。然而，此命令以字符串形式返回标准的Geohash，该字符串的格式描述在[Wikipedia的文章](https://en.wikipedia.org/wiki/Geohash)中，并且与[geohash.org](http://geohash.org)网站兼容。

地理哈希字符串属性
---


该命令返回11个字符的Geohash字符串，因此与Redis内部的52位表示相比，不会丢失任何精度。返回的Geohashes具有以下属性：

1. 可以从右边删除字符以缩短它们。这样会丢失精度，但仍然指向相同的区域。
2. 可以在 `geohash.org` 的 URL 中使用它们，比如 `http://geohash.org/<geohash-string>`。这是一个[此类 URL 的示例](http://geohash.org/sqdtr74hyu0)。
3. 具有相似前缀的字符串是相邻的，但相反并非如此，不同前缀的字符串也可能是相邻的。

@examples

```cli
GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
GEOHASH Sicily Palermo Catania
```
