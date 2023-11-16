这个命令类似于 `GEOSEARCH`，但将结果存储在目标键中。

这个命令替代了现在已弃用的`GEORADIUS`和`GEORADIUSBYMEMBER`。

默认情况下，它将结果存储在具有其地理空间信息的“destination”有序集合中。

在使用`STOREDIST`选项时，该命令会将项目存储在一个有序集合中，该集合根据它们与圆形或矩形中心的距离以浮点数的形式进行填充，单位与该形状指定的单位相同。

@examples

```cli
GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
GEOADD Sicily 12.758489 38.788135 "edge1"   17.241510 38.788135 "edge2" 
GEOSEARCHSTORE key1 Sicily FROMLONLAT 15 37 BYBOX 400 400 km ASC COUNT 3
GEOSEARCH key1 FROMLONLAT 15 37 BYBOX 400 400 km ASC WITHCOORD WITHDIST WITHHASH
GEOSEARCHSTORE key2 Sicily FROMLONLAT 15 37 BYBOX 400 400 km ASC COUNT 3 STOREDIST
ZRANGE key2 0 -1 WITHSCORES
```
