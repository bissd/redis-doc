默认情况下，回复包含服务器的所有命令。
您可以使用可选的“命令名称”参数来指定一个或多个命令的名称。

回复中包含每个已返回的命令的地图。
映射的回复中可能包含以下键：


* **摘要：** 简短的命令描述。
* **自从：** 添加命令的 Redis 版本（对于模块命令，是模块版本）。
* **分组：** 命令所属的功能组。
  可能的值有：
  - _bitmap_
  - _cluster_
  - _connection_
  - _generic_
  - _geo_
  - _hash_
  - _hyperloglog_
  - _list_
  - _module_
  - _pubsub_
  - _scripting_
  - _sentinel_
  - _server_
  - _set_
  - _sorted-set_
  - _stream_
  - _string_
  - _transactions_
* **复杂度：** 关于命令的时间复杂度的简要解释。
* **文档标志：** 一个文档标志的数组。
  可能的值有：
  - _deprecated:_ 命令已被弃用。
  - _syscmd:_ 一个系统命令，不适用于用户调用。
* **弃用自：** 弃用命令的 Redis 版本（对于模块命令，是模块版本）。
* **替代为：** 弃用命令的替代方式。
* **历史：** 描述命令行为或参数更改的历史记录数组。
  每个条目本身都是一个数组，由两个元素组成：
  1. 适用于条目的 Redis 版本。
  2. 更改的描述。
* **参数：** 描述命令参数的映射数组。
  请参考 [Redis 命令参数][td] 页面获取更多信息。

[td]: /topics/command-arguments

@examples

```cli
COMMAND DOCS SET
```
