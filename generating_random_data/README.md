# 生成随机数据


libsodium库提供一组函数，来产生不可预测的数据，适合用来作为密钥。


- 在 Windows 系统, 使用 `RtlGenRandom()` 函数
- 在 OpenBSD 和 Bitrig 上, 使用 `arc4random()` 函数
- 在较新的 Linux 内核上,  使用 `getrandom`系统调用 (从 Sodium 1.0.3 开始)
- 在其它 Unix 类系统上, 使用 `/dev/urandom` 设备
- 如果以上选项都无法安全地使用，自定义的实现可以很容易地 hook 进来。

## 用法

```c
uint32_t randombytes_random(void);
```

  `randombytes_random()` 函数返回一个在  `0` 和 `0xffffffff` (包含) 之间的不可预测的值。
  
```c
uint32_t randombytes_uniform(const uint32_t upper_bound);
```

 `randombytes_uniform()` 函数返回一个  `0` 和 `upper_bound` (不包含) 之间的不可预测的值. 不是简单地 `randombytes_random() % upper_bound`, 本函数尽可能保证可能输出值的概率均匀分布。

```c
void randombytes_buf(void * const buf, const size_t size);
```

 `randombytes_buf()` 函数用不可预测的字节序列填充 `buf` 开始的 `size` 字节。

```c
int randombytes_close(void);
```

本函数释放伪随机数生成器使用的全局资源。具体地说，如果使用了 `/dev/urandom`，本函数关闭描述符。
基本不需要显式调用本函数。

```c
void randombytes_stir(void);
```

如果支持， `randombytes_stir()` 函数 重设 伪随机数生成器的种子。对默认的伪随机数生成器，不需要调用本函数。就算 `fork()`之后也不需要。
除非调用 `randombytes_close()` 关闭了 `/dev/unrandom` 的描述符 fd 。

如果使用了一个非默认的实现， (看 `randombytes_set_implementation()`), `randombytes_stir()` 必须 在 `fork()` 之后的子进程中调用 。

## Note

如果在一个 VM 中的应用程序中使用这些函数，并且 VM 被做了快照并恢复了，那这些函数有可能产生相同的输出。
