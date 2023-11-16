以零为基础，返回存储在`key`中的列表中索引为`index`的元素。所以`0`表示第一个元素，`1`表示第二个元素，以此类推。
可以使用负索引来指定从列表尾部开始的元素。
在这里，`-1`表示最后一个元素，`-2`表示倒数第二个元素，依此类推。

当`key`处的值不是一个列表时，返回一个错误。

@examples

```cli
LPUSH mylist "World"
LPUSH mylist "Hello"
LINDEX mylist 0
LINDEX mylist -1
LINDEX mylist 3
```
