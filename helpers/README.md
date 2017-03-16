# 辅助函数

## 常数时间的比较函数

```c
int sodium_memcmp(const void * const b1_, const void * const b2_, size_t len);
```

当在 秘密数据（比如 密钥， 认证的tag gcm的tag 等 ）上进行 比较操作的时候，非常重要的是:一定要使用一个 常数时间的比较函数，来免疫侧信道攻击 。

`sodium_memcmp()` 就是用来做常数时间比较的。  
 `sodium_memcmp()` 返回 `0` 如果  `b1_` 指向的 `len` 字节  和  `b2_` 指向的 `len` 相同. 否则，返回 `-1`.

**注意:** `sodium_memcmp()` 不会做字典序比较，不判断大小关系，只判断是否相等。并且也不是 `memcmp()` 的通用替代品。

## 十六进制 encoding/decoding

```c
char *sodium_bin2hex(char * const hex, const size_t hex_maxlen,
                     const unsigned char * const bin, const size_t bin_len);
```

`sodium_bin2hex()` 把 `bin` 指向的 `bin_len` 转换成 十六进制的 字符串。  
结果十六进制字符串存入 `hex` 中，并且包含一个 null \( `\0` \) 结束字节。

`hex_maxlen` 是从 `hex` 开始可以写入的最大长度，`hex_maxlen`最少应该是  `bin_len * 2 + 1`.

这个函数返回 `hex` 表示成功，返回`NULL`表示 `hex`长度不够。
对一个给定的输入大小，这个函数的执行时间是个常量。


```c
int sodium_hex2bin(unsigned char * const bin, const size_t bin_maxlen,
                   const char * const hex, const size_t hex_len,
                   const char * const ignore, size_t * const bin_len,
                   const char ** const hex_end);
```

 `sodium_hex2bin()` 函数解析一个十六进制字符串 `hex` 并且把它转换成一个字节序列。
 
`hex` 不需要以 null \('\0'\) 结尾，因为需要解析的字符串的长度已经由 `hex_len` 指定了。


`ignore` 是需要跳过的字符组成的字符串，例如，字符串 `": "` 允许空格和冒号在十六进制字符串中的任何位置。这些字符会被忽略，这样, `"69:FC"`, `"69 FC"`, `"69 : FC"` 和 `"69FC"` 就是合法的输入，并产生相同的输出。

`ignore` 可以设成  `NULL`，这样就不允许任何非十六进制的字符。


`bin_maxlen` 是 `bin` 能容纳的长度。

解析器会在发现非十六进制的非忽略字符，或 `bin_maxlen`限制到达的时候停止。

这个函数返回 `-1` 表示 `bin_maxlen`不够长，放不下结果。返回`0`表示成功，并且设置 `hex_end`。如果 `hex_end`不是 `NULL`，那就指向已经解析完成的字符的下一个字符。
 
对给定的长度和格式，这个函数的执行时间是常量。

## 增长大数字

```c
void sodium_increment(unsigned char *n, const size_t nlen);
```

 `sodium_increment()` 函数接受一个指向任意长度的无符号数字的指针，对其做增长操作。
 
这个函数对给定长度，运行时间是个常量。并且假设数字是小端字节序的。

`sodium_increment()` 可以用于以常量时间增长nonce字段。

这个函数是 libsodium 1.0.4 引入的。

## 大数字的加法 

```c
void sodium_add(unsigned char *a, const unsigned char *b, const size_t len);
```

  `sodium_add()` 函数接受两个指向无符号小端字节序数字的指针，`a` 和 `b`, 两个都是 `len` 字节长。
  
并计算 `(a + b) mod 2^(8*len)` ，对给定长度，运行时间是常数，并且用结果覆盖 `a`。

这个函数是 libsodium 1.0.7 引入的.

## 比较大数字

```c
int sodium_compare(const void * const b1_, const void * const b2_, size_t len);
```

给定 `b1_` 和 `b2_`, 两个 `len` 字节长度的小端字节序数字， 本函数返回:

* `-1` 如果 `b1_` 小于 `b2_`
* `0`  如果 `b1_` 等于 `b2_`
* `1`  如果 `b1_` 大于 `b2_` 


这个比较对给定长度，是常数时间的操作。
这个函数可以用于nonce的比较，用来阻止 重放攻击。
这个函数是 libsodium 1.0.6 引入的.

## 全零测试

```c
int sodium_is_zero(const unsigned char *n, const size_t nlen);
```

这个函数返回 `1`，表示 `n`指向的 `nlen`字节的数组只包含0 。
返回`0`表示发现有非0的比特。

对给定长度，本函数的执行时间是常量。
本函数是 libsodium 1.0.7 引入的。
