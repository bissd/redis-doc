
`LCS` 命令实现了最长公共子序列算法。请注意，这与最长公共字符串算法不同，因为字符串中的匹配字符不需要连续。

例如，"foo"和"fao"之间的最长公共子序列是"fo"，因为从左到右扫描这两个字符串时，最长的公共字符集由第一个"f"和后面的"o"组成。

LCS在评估两个字符串的相似程度方面非常有用。字符串可以表示许多不同的事物。例如，如果两个字符串是DNA序列，LCS将提供两个DNA序列之间相似程度的衡量标准。如果这些字符串代表某个用户编辑的文本，LCS可以表示新文本与旧文本之间的差异程度，等等。

请注意，此算法的运行时间为`O(N*M)`，其中N是第一个字符串的长度，M是第二个字符串的长度。所以要么启动一个不同的Redis实例来运行此算法，要么确保对非常小的字符串运行它。

```
> MSET key1 ohmytext key2 mynewtext
OK
> LCS key1 key2
"mytext"
```

有时候我们只需要匹配的长度：

```
> LCS key1 key2 LEN
(integer) 6
```

然而，通常非常有用的是，知道每个字符串中的匹配位置：

```
> LCS key1 key2 IDX
1) "matches"
2) 1) 1) 1) (integer) 4
         2) (integer) 7
      2) 1) (integer) 5
         2) (integer) 8
   2) 1) 1) (integer) 2
         2) (integer) 3
      2) 1) (integer) 0
         2) (integer) 1
3) "len"
4) (integer) 6
```

根据算法的工作方式，匹配是从最后一个到第一个生成的，以相同的顺序发出东西更有效。
上述数组表示第一个匹配（数组的第二个元素）在第一个字符串的位置2-3和第二个字符串的位置0-1之间。
然后再次出现匹配，介于4-7和5-8之间。

为了将匹配项列表限制为特定最小长度的项：

```
> LCS key1 key2 IDX MINMATCHLEN 4
1) "matches"
2) 1) 1) 1) (integer) 4
         2) (integer) 7
      2) 1) (integer) 5
         2) (integer) 8
3) "len"
4) (integer) 6
```

最后也需匹配len：

```
> LCS key1 key2 IDX MINMATCHLEN 4 WITHMATCHLEN
1) "matches"
2) 1) 1) 1) (integer) 4
         2) (integer) 7
      2) 1) (integer) 5
         2) (integer) 8
      3) (integer) 4
3) "len"
4) (integer) 6
```
