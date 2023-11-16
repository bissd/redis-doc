---
title: "Redis地理空间"
linkTitle: "地理空间"
weight: 80
description: >
    Redis地理空间数据类型介绍
---

Redis地理空间索引允许您存储坐标并对其进行搜索。
该数据结构对于在给定的半径或边界框内找到附近点非常有用。

## 基本命令

* `GEOADD` 将位置添加到指定的地理空间索引（注意，经度在纬度之前）。
* `GEOSEARCH` 返回带有给定半径或边界框的位置。

请参阅[完整的地理空间索引命令列表](https://redis.io/commands/?group=geo)。


## 示例

假设您正在开发一款移动应用，它可以帮助您查找离您当前位置最近的所有自行车租赁站点。

将几个位置添加到地理空间索引中：
{{< clients-example geo_tutorial geoadd >}}
> GEOADD bikes:rentable -122.27652 37.805186 station:1
(integer) 1
> GEOADD bikes:rentable -122.2674626 37.8062344 station:2
(integer) 1
> GEOADD bikes:rentable -122.2469854 37.8104049 station:3
(integer) 1
{{< /clients-example >}}

通过提供的位置，在半径5公里范围内找到所有位置，并返回每个位置的距离：
{{< clients-example geo_tutorial geosearch >}}
> GEOSEARCH bikes:rentable FROMLONLAT -122.2612767 37.7936847 BYRADIUS 5 km WITHDIST
1) 1) "station:1"
   2) "1.8523"
2) 1) "station:2"
   2) "1.4979"
3) 1) "station:3"
   2) "2.2441"
{{< /clients-example >}}

## 了解更多

* [Redis地理空间解释](https://www.youtube.com/watch?v=qftiVQraxmI)通过展示如何构建一个本地公园景点地图来介绍地理空间索引。
* [Redis大学的RU101](https://university.redis.com/courses/ru101/)详细介绍了Redis地理空间索引。
