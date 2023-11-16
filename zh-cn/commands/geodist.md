返回由有序集合表示的地理空间索引中两个成员之间的距离。

给定一个表示地理空间索引的有序集合，使用`GEOADD`命令进行填充，该命令返回指定成员之间的距离，以指定的单位表示。

如果一个或两个成员缺失，该命令将返回NULL。

单位必须是以下之一，并默认为米：

* **m** 代表米。
* **km** 代表千米。
* **mi** 代表英里。
* **ft** 代表英尺。

该距离是基于假设地球为完美球体计算的，因此在极端情况下可能存在最高0.5%的误差。

@examples

```cli
GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
GEODIST Sicily Palermo Catania
GEODIST Sicily Palermo Catania km
GEODIST Sicily Palermo Catania mi
GEODIST Sicily Foo Bar
```
