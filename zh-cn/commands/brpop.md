`BRPOP` 是一个阻塞列表弹出的原语。
它是`RPOP`的阻塞版本，因为当给定的列表中没有元素可以弹出时，它会阻塞连接。
从第一个非空列表的尾部弹出一个元素，按照给定的顺序检查给定的键。

请参阅[BLPOP文档][cb]以查看确切的语义，因为`BRPOP`与`BLPOP`唯一的区别在于它从列表的尾部弹出元素而不是从头部弹出。

[cb]: /commands/blpop

@examples

```
redis> DEL list1 list2
(integer) 0
redis> RPUSH list1 a b c
(integer) 3
redis> BRPOP list1 list2 0
1) "list1"
2) "c"
```
