在字符串中返回第一个设置为1或0的位的位置。

返回位置，将字符串从左到右视为比特数组，

其中第一个字节的最高有效位位于位置0，

第二个字节的最高有效位位于位置8，依此类推。

`GETBIT`和`SETBIT`都遵循相同的位位置约定。

默认情况下，将检查字符串中包含的所有字节。
可以通过传递额外的参数_start_和_end_仅在指定的区间内查找位（也可以只传递_start_，操作将假定结束位置是字符串的最后一个字节。但是有语义差异，请参见后文）。
默认情况下，区间被解释为字节范围而不是位范围，因此`start=0`和`end=2`表示查看前三个字节。

您可以使用可选的`BIT`修饰符来指定范围应被解释为一系列位。
因此`start=0`和`end=2`意味着查看前三位。

请注意，位位置始终作为绝对值从位零返回，即使使用 _start_ 和 _end_ 来指定范围。

在 `GETRANGE` 命令中，start 和 end 可以包含负数值，以便从字符串末尾索引字节，其中 -1 是最后一个字节，-2 是倒数第二个字节，依此类推。当指定 `BIT` 时，-1 是最后一位，-2 是倒数第二位，依此类推。

存在的键，会处理为空字符串。

@examples

```cli
SET mykey "\xff\xf0\x00"
BITPOS mykey 0
SET mykey "\x00\xff\xf0"
BITPOS mykey 1 0
BITPOS mykey 1 2
BITPOS mykey 1 2 -1 BYTE
BITPOS mykey 1 7 15 BIT
set mykey "\x00\x00\x00"
BITPOS mykey 1
BITPOS mykey 1 7 -3 BIT
```
