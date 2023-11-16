从提供的键名列表中的第一个非空列表键中弹出一个或多个元素。

`LMPOP` 和 `BLMPOP` 类似于下面的更受限制的命令：

- `LPOP` 或 `RPOP` 只接受一个键，可以返回多个元素。
- `BLPOP` 或 `BRPOP` 接受多个键，但只返回一个键中的一个元素。

请参考 `BLMPOP` 命令的阻塞变体。

根据传入的参数，元素可以从第一个非空列表的左侧或右侧弹出。
返回的元素数量被限制为非空列表的长度和计数参数（默认为1）中的较小值。

@examples

```cli
LMPOP 2 non1 non2 LEFT COUNT 10
LPUSH mylist "one" "two" "three" "four" "five"
LMPOP 1 mylist LEFT
LRANGE mylist 0 -1
LMPOP 1 mylist RIGHT COUNT 10
LPUSH mylist "one" "two" "three" "four" "five"
LPUSH mylist2 "a" "b" "c" "d" "e"
LMPOP 2 mylist mylist2 right count 3
LRANGE mylist 0 -1
LMPOP 2 mylist mylist2 right count 5
LMPOP 2 mylist mylist2 right count 10
EXISTS mylist mylist2
```
