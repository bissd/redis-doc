这个命令返回关于存储在`<key>`位置的流的信息。

此命令提供的详细信息为：

* **length**: 流中条目的数量（参见 `XLEN`）
* **radix-tree-keys**: 底层基数数据结构中的键数
* **radix-tree-nodes**: 底层基数数据结构中的节点数
* **groups**: 为该流定义的消费者组数
* **last-generated-id**: 添加到流中的最近一条条目的ID
* **max-deleted-entry-id**: 从流中删除的最大条目ID
* **entries-added**: 流的生命周期中添加的所有条目数
* **first-entry**: 流中第一条条目的ID和字段值元组
* **last-entry**: 流中最后一条条目的ID和字段值元组

可选的 `FULL` 修饰符提供更详细的回复。
当提供 `FULL` 回复时，它包括一个以升序排列的**entries**数组，该数组包含流条目（ID 和字段值元组）。
此外，**groups** 也是一个数组，对于每个消费者组，它包含 `XINFO GROUPS` 和 `XINFO CONSUMERS` 报告的信息。

`COUNT` 选项可用于限制返回的流和 PEL 条目数量（返回前 `<count>` 个条目）。
默认的 `COUNT` 为 10，`COUNT` 为 0 的意思是将返回所有条目（如果流中有很多条目，执行时间可能很长）。

@examples

## Default Reply

您好！感谢您联系我们。我们的客服人员暂时无法回复您的消息，但我们会尽快回复您。如果您有任何问题或需要帮助，请随时联系我们。谢谢！

```
> XINFO STREAM mystream
 1) "length"
 2) (integer) 2
 3) "radix-tree-keys"
 4) (integer) 1
 5) "radix-tree-nodes"
 6) (integer) 2
 7) "last-generated-id"
 8) "1638125141232-0"
 9) "max-deleted-entry-id"
10) "0-0"
11) "entries-added"
12) (integer) 2
13) "groups"
14) (integer) 1
15) "first-entry"
16) 1) "1638125133432-0"
    2) 1) "message"
       2) "apple"
17) "last-entry"
18) 1) "1638125141232-0"
    2) 1) "message"
       2) "banana"
```

完整回复：

```
> XADD mystream * foo bar
"1638125133432-0"
> XADD mystream * foo bar2
"1638125141232-0"
> XGROUP CREATE mystream mygroup 0-0
OK
> XREADGROUP GROUP mygroup Alice COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) "1638125133432-0"
         2) 1) "foo"
            2) "bar"
> XINFO STREAM mystream FULL
 1) "length"
 2) (integer) 2
 3) "radix-tree-keys"
 4) (integer) 1
 5) "radix-tree-nodes"
 6) (integer) 2
 7) "last-generated-id"
 8) "1638125141232-0"
 9) "max-deleted-entry-id"
10) "0-0"
11) "entries-added"
12) (integer) 2
13) "entries"
14) 1) 1) "1638125133432-0"
       2) 1) "foo"
          2) "bar"
    2) 1) "1638125141232-0"
       2) 1) "foo"
          2) "bar2"
15) "groups"
16) 1)  1) "name"
        2) "mygroup"
        3) "last-delivered-id"
        4) "1638125133432-0"
        5) "entries-read"
        6) (integer) 1
        7) "lag"
        8) (integer) 1
        9) "pel-count"
       10) (integer) 1
       11) "pending"
       12) 1) 1) "1638125133432-0"
              2) "Alice"
              3) (integer) 1638125153423
              4) (integer) 1
       13) "consumers"
       14) 1) 1) "name"
              2) "Alice"
              3) "seen-time"
              4) (integer) 1638125133422
              5) "active-time"
              6) (integer) 1638125133432
              7) "pel-count"
              8) (integer) 1
              9) "pending"
              10) 1) 1) "1638125133432-0"
                     2) (integer) 1638125133432
                     3) (integer) 1
```
