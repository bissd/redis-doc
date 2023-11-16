---
title: "Redis命令参数"
linkTitle: "命令参数"
weight: 7
description: 如何以编程方式获取Redis命令的文档说明
aliases:
    - /topics/command-arguments
---

`COMMAND DOCS`命令返回有关可用Redis命令的文档信息。
命令返回的映射回复包括_arguments_键。
该键存储一个描述命令参数的数组。

_arguments_数组中的每个元素都是一个具有以下字段的映射：

* **name:** 参数的名称，始终存在。
  参数的名称仅用于标识目的。
  在命令语法渲染过程中，不显示参数的名称。
  同一个名称可以在整个参数树中出现多次，但与其他同级参数的名称相比是唯一的。
  这样可以为每个参数获取一个唯一标识符（从根路径到任何参数的路径上的所有名称的串联）。
* **display_text:** 参数的显示字符串，存在于具有可显示表示（除了 oneof/block 外的所有参数）的参数中。
  这是命令语法渲染中使用的字符串。
* **type:** 参数的类型，始终存在。
  参数必须具有以下类型之一：
  - **string:** 字符串参数。
  - **integer:** 整数参数。
  - **double:** 双精度参数。
  - **key:** 表示键名称的字符串。
  - **pattern:** 表示类似于通配符的模式的字符串。
  - **unix-time:** 表示 Unix 时间戳的整数。
  - **pure-token:** 参数是一个标记，表示一个保留关键字，可能会提供或不提供。
    不要与自由文本用户输入混淆。
  - **oneof**: 参数是用于嵌套参数的容器。
    此类型允许在几个嵌套参数中进行选择（请参阅下面的 `XADD` 示例）。
  - **block:** 参数是用于嵌套参数的容器。
    此类型允许对参数进行分组，并对所有参数应用属性（例如 _optional_）（请参阅下面的 `XADD` 示例）。
* **key_spec_index:** 对于 _key_ 类型的每个参数都可用的此值。
  它是命令的 [key specifications][tr] 中对应于参数的从 0 开始的索引。
* **token**: 参数（用户输入）本身之前的常量文字。
* **summary:** 参数的简短描述。
* **since:** 参数的首发 Redis 版本（或对于模块命令，模块版本）。
* **deprecated_since:** 弃用命令的 Redis 版本（或对于模块命令，模块版本）。
* **flags:** 参数标志的数组。
  可能的标志有：
  - **optional**: 表示参数是可选的（例如，`SET` 命令的 _GET_ 子句）。
  - **multiple**: 表示参数可以重复（例如，`DEL` 命令的 _key_ 参数）。
  - **multiple-token:** 表示参数与其前导标记可能重复（参见 `SORT` 的 `GET pattern` 子句）。
* **value:** 参数的值。
  对于除了 _oneof_ 和 _block_ 之外的参数类型，这是一个描述命令语法中该值的字符串。
  对于 _oneof_ 和 _block_ 类型，这是一个嵌套参数的数组，每个嵌套参数都是本节中描述的映射。

[tr]: /topics/key-specs

## 示例

`XADD`的修剪子句，即`[MAXLEN|MINID [=|~] threshold [LIMIT count]]`，在顶级上以_block_-类型的参数表示。

它包含四个嵌套的参数：

1. **修剪策略：**此嵌套参数具有_oneof_类型，包含两个嵌套参数。
   每个嵌套参数_MAXLEN_和_MINID_的类型为_pure-token_。
2. **修剪运算符：**此嵌套参数是一个可选的_oneof_类型，包含两个嵌套参数。
   每个嵌套参数_=_和_~_都是_pure-token_类型。
3. **阈值：**此嵌套参数是一个_string_类型。
4. **计数：**此嵌套参数是一个可选的_integer_类型，带有一个_token_（_LIMIT_）。

以下是`XADD`的参数数组：

```
1) 1) "name"
   2) "key"
   3) "type"
   4) "key"
   5) "value"
   6) "key"
2)  1) "name"
    2) "nomkstream"
    3) "type"
    4) "pure-token"
    5) "token"
    6) "NOMKSTREAM"
    7) "since"
    8) "6.2"
    9) "flags"
   10) 1) optional
3) 1) "name"
   2) "trim"
   3) "type"
   4) "block"
   5) "flags"
   6) 1) optional
   7) "value"
   8) 1) 1) "name"
         2) "strategy"
         3) "type"
         4) "oneof"
         5) "value"
         6) 1) 1) "name"
               2) "maxlen"
               3) "type"
               4) "pure-token"
               5) "token"
               6) "MAXLEN"
            2) 1) "name"
               2) "minid"
               3) "type"
               4) "pure-token"
               5) "token"
               6) "MINID"
               7) "since"
               8) "6.2"
      2) 1) "name"
         2) "operator"
         3) "type"
         4) "oneof"
         5) "flags"
         6) 1) optional
         7) "value"
         8) 1) 1) "name"
               2) "equal"
               3) "type"
               4) "pure-token"
               5) "token"
               6) "="
            2) 1) "name"
               2) "approximately"
               3) "type"
               4) "pure-token"
               5) "token"
               6) "~"
      3) 1) "name"
         2) "threshold"
         3) "type"
         4) "string"
         5) "value"
         6) "threshold"
      4)  1) "name"
          2) "count"
          3) "type"
          4) "integer"
          5) "token"
          6) "LIMIT"
          7) "since"
          8) "6.2"
          9) "flags"
         10) 1) optional
         11) "value"
         12) "count"
4) 1) "name"
   2) "id_or_auto"
   3) "type"
   4) "oneof"
   5) "value"
   6) 1) 1) "name"
         2) "auto_id"
         3) "type"
         4) "pure-token"
         5) "token"
         6) "*"
      2) 1) "name"
         2) "id"
         3) "type"
         4) "string"
         5) "value"
         6) "id"
5) 1) "name"
   2) "field_value"
   3) "type"
   4) "block"
   5) "flags"
   6) 1) multiple
   7) "value"
   8) 1) 1) "name"
         2) "field"
         3) "type"
         4) "string"
         5) "value"
         6) "field"
      2) 1) "name"
         2) "value"
         3) "type"
         4) "string"
         5) "value"
         6) "value"
```
