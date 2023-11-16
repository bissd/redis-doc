在存储在 _key_ 中的字符串值中设置或清除偏移量为 _offset_ 的位。

位取决于_value_的设置或清除，_value_ 可以是 0 或 1。

当_key_不存在时，创建一个新的字符串值。
为了确保字符串能够容纳_offset_位置的位，字符串将被扩展。
_offset_参数必须大于等于0且小于2^32（这限制了位图的大小为512MB）。
当字符串在_key_位置扩展时，新增的位将被置为0。

**警告**：当将最后一个可行的位（_offset_等于2 ^ 32-1）设置为字符串值，且存储在_key_上的字符串值尚未被持有，或者持有一个小的字符串值时，Redis需要分配所有中间内存，这可能会导致服务器阻塞一段时间。
在2010年的MacBook Pro上，设置位号2 ^ 32-1（512MB分配）需要大约300毫秒，设置位号2 ^ 30-1（128MB分配）需要大约80毫秒，设置位号2 ^ 28-1（32MB分配）需要大约30毫秒，设置位号2 ^ 26 -1（8MB分配）需要大约8毫秒。
请注意，一旦完成了第一次分配，对于相同_key_的后续`SETBIT`调用将不会有分配开销。

@examples

```cli
SETBIT mykey 7 1
SETBIT mykey 7 0
GET mykey
```

## 模式: 访问整个位图

有时候你需要一次性设置单个位图的所有位，比如将其初始化为默认非零值时。可以通过多次调用`SETBIT`命令来实现，每个位都需要调用一次。然而，为了优化，你可以使用一个`SET`命令来设置整个位图。

位图不是一种实际的数据类型，而是一组在String类型上定义的位操作（详细信息请参阅[Data Types Introduction页面的Bitmaps部分][ti]）。这意味着可以将位图与字符串命令一起使用，尤其是与`SET`和`GET`命令一起使用。

因为Redis的字符串是二进制安全的，所以位图可以简单地编码为字节流。字符串的第一个字节对应于位图的0..7偏移量，第二个字节对应于8..15范围，依此类推。

比如，设置了几个位后，获取位图的字符串值将会是这样的：

```
> SETBIT bitmapsarestrings 2 1
> SETBIT bitmapsarestrings 3 1
> SETBIT bitmapsarestrings 5 1
> SETBIT bitmapsarestrings 10 1
> SETBIT bitmapsarestrings 11 1
> SETBIT bitmapsarestrings 14 1
> GET bitmapsarestrings
"42"
```

通过获取位图的字符串表示，客户端可以通过在其本地编程语言中使用本地位操作提取位值来解析响应的字节。相对称地，客户端还可以通过在客户端执行位到字节编码并调用`SET`来设置整个位图，以获得结果字符串。

[ti]: /topics/data-types-intro#位图

## 模式：设置多个位

`SETBIT`在设置单个位上表现出色，当需要设置多个位时可以多次调用。为了优化此操作，您可以将多个`SETBIT`调用替换为一次对可变参数的`BITFIELD`命令的调用和使用`u1`类型的字段。

例如，上面的示例可以替换为：

```
> BITFIELD bitsinabitmap SET u1 2 1 SET u1 3 1 SET u1 5 1 SET u1 10 1 SET u1 11 1 SET u1 14 1
```

## 高级模式：访问位图范围

也可以使用 `GETRANGE` 和 `SETRANGE` 字符串命令来高效地访问位图中的一系列位偏移量。下面是一个使用Redis习惯用法的Lua脚本示例，可以通过`EVAL`命令来运行：

```
--[[
Sets a bitmap range

Bitmaps are stored as Strings in Redis. A range spans one or more bytes,
so we can call `SETRANGE` when entire bytes need to be set instead of flipping
individual bits. Also, to avoid multiple internal memory allocations in
Redis, we traverse in reverse.
Expected input:
  KEYS[1] - bitfield key
  ARGV[1] - start offset (0-based, inclusive)
  ARGV[2] - end offset (same, should be bigger than start, no error checking)
  ARGV[3] - value (should be 0 or 1, no error checking)
]]--

-- A helper function to stringify a binary string to semi-binary format
local function tobits(str)
  local r = ''
  for i = 1, string.len(str) do
    local c = string.byte(str, i)
    local b = ' '
    for j = 0, 7 do
      b = tostring(bit.band(c, 1)) .. b
      c = bit.rshift(c, 1)
    end
    r = r .. b
  end
  return r
end

-- Main
local k = KEYS[1]
local s, e, v = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3])

-- First treat the dangling bits in the last byte
local ms, me = s % 8, (e + 1) % 8
if me > 0 then
  local t = math.max(e - me + 1, s)
  for i = e, t, -1 do
    redis.call('SETBIT', k, i, v)
  end
  e = t
end

-- Then the danglings in the first byte
if ms > 0 then
  local t = math.min(s - ms + 7, e)
  for i = s, t, 1 do
    redis.call('SETBIT', k, i, v)
  end
  s = t + 1
end

-- Set a range accordingly, if at all
local rs, re = s / 8, (e + 1) / 8
local rl = re - rs
if rl > 0 then
  local b = '\255'
  if 0 == v then
    b = '\0'
  end
  redis.call('SETRANGE', k, rs, string.rep(b, rl))
end
```

**注意：**从位图中获取一系列位偏移的实现留给读者作为一个练习。
