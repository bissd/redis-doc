该命令与`GEORADIUS`完全相同，唯一的区别是它不是使用经度和纬度的值作为查询区域的中心，而是使用已存在于有序集合中的地理空间索引的成员名称作为中心。

查询使用指定成员的位置作为中心点。

请参考下面的示例和 `GEORADIUS` 文档以获取有关该命令和其选项的更多信息。

请注意，`GEORADIUSBYMEMBER_RO`在Redis 3.2.10和Redis 4.0.0中也可用，以提供在副本中使用的只读命令。有关更多信息，请参阅`GEORADIUS`页面。

@examples

```cli
GEOADD Sicily 13.583333 37.316667 "Agrigento"
GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
GEORADIUSBYMEMBER Sicily Agrigento 100 km
```
