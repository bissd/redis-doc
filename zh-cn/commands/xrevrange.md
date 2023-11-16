此命令与`XRANGE`完全相同，但有一个显著的区别，即返回的条目顺序相反，并且范围的开始-结束顺序也相反：在`XREVRANGE`中，您需要先指定*结束* ID，然后是*开始* ID，命令将生成从*结束*一侧开始的两个ID之间的所有元素（包括两个ID）的元素。

所以例如，要获取从较高ID到较低ID的所有元素，可以使用以下命令：

    XREVRANGE somestream + -

与此类似，要获取流中添加的最后一个元素，只需发送：

    XREVRANGE somestream + - COUNT 1

@examples

```cli
XADD writers * name Virginia surname Woolf
XADD writers * name Jane surname Austen
XADD writers * name Toni surname Morrison
XADD writers * name Agatha surname Christie
XADD writers * name Ngozi surname Adichie
XLEN writers
XREVRANGE writers + - COUNT 1
```
