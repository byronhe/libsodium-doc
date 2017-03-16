#  Argon2 内存耗费函数

从 version 1.0.9 开始, libsodium 提供名叫 Argon2 的 用户密码哈希函数。

Argon2 集成了目前在内存耗费函数设计领域 最先进的技术。

Argon2 的目标是 最高的 内存填充率，和对多个计算单元的有效利用，同时仍然提供针对 tradeoff 攻击的有效防御。

Argon2 避免了 ASIC 硬件相对 软件实现 的性能优势。


## Example 1: 密码存储 password storage

```c
#define PASSWORD "Correct Horse Battery Staple"

char hashed_password[crypto_pwhash_STRBYTES];

if (crypto_pwhash_str
    (hashed_password, PASSWORD, strlen(PASSWORD),
     crypto_pwhash_OPSLIMIT_SENSITIVE, crypto_pwhash_MEMLIMIT_SENSITIVE) != 0) {
    /* out of memory */
}

if (crypto_pwhash_str_verify
    (hashed_password, PASSWORD, strlen(PASSWORD)) != 0) {
    /* wrong password */
}
```



## Example 2: 密钥衍生 key derivation

```c
#define PASSWORD "Correct Horse Battery Staple"
#define KEY_LEN crypto_box_SEEDBYTES

unsigned char salt[crypto_pwhash_SALTBYTES];
unsigned char key[KEY_LEN];

randombytes_buf(salt, sizeof salt);

if (crypto_pwhash
    (key, sizeof key, PASSWORD, strlen(PASSWORD), salt,
     crypto_pwhash_OPSLIMIT_INTERACTIVE, crypto_pwhash_MEMLIMIT_INTERACTIVE,
     crypto_pwhash_ALG_DEFAULT) != 0) {
    /* out of memory */
}
```

## 密码存储 Password storage

```c
int crypto_pwhash_str(char out[crypto_pwhash_STRBYTES],
                                           const char * const passwd,
                                           unsigned long long passwdlen,
                                           unsigned long long opslimit,
                                           size_t memlimit);
```

 `crypto_pwhash_str()` 函数把  ASCII 编码的 字符串写入 `out`, 包含:
- 一个 对 长度 `passwdlen` 的密码 `passwd ` 进行 内存耗费， CPU 密集 计算，产生的结果。
- 为进行上述计算自动生成的盐 salt。
- 用来验证密码 password 需要的其它参数，包括算法标识符，算法版本， `opslimit` 和 `memlimit` 参数.

`out` 必须足够长来保存 `crypto_pwhash_STRBYTES` 字节, 但是实际的输出字符串可能会更短。

输出字符串是 `'\0'` 结尾的，只包括  ASCII 字符串， 并且可以安全的存入  SQL 数据库，和其它存储中。 不需要额外再存其它字段用来 验证 密码 password。

这个函数返回 `0` 表示成功，`-1`表示没有成功完成。

```c
int crypto_pwhash_str_verify(const char str[crypto_pwhash_STRBYTES],
                                                  const char * const passwd,
                                                  unsigned long long passwdlen);
```

这个函数验证  `str` 是否 对 长度 `passwdlen` 的 `passwd` 是一个 合法的 密码验证字符串 (作为 `crypto_pwhash_str()` 生成的输出) 。

`str` 必须是 `'\0'` 结尾的。

这个函数返回 `0` 表示验证成功了。 `-1`表示验证出错。


## 密钥衍生 Key derivation

```c
int crypto_pwhash(unsigned char * const out,
                                       unsigned long long outlen,
                                       const char * const passwd,
                                       unsigned long long passwdlen,
                                       const unsigned char * const salt,
                                       unsigned long long opslimit,
                                       size_t memlimit, int alg);
```

The `crypto_pwhash()` function derives an `outlen` bytes long key from a password `passwd` whose length is `passwdlen` and a salt `salt` whose fixed length is `crypto_pwhash_SALTBYTES` bytes. `outlen` should be at least `16` (128 bits).

The computed key is stored into `out`.

`opslimit` represents a maximum amount of computations to perform. Raising this number will make the function require more CPU cycles to compute a key.

`memlimit` is the maximum amount of RAM that the function will use, in bytes.

`alg` is an identifier for the algorithm to use and should be currently set to `crypto_pwhash_ALG_DEFAULT`.

For interactive, online operations, `crypto_pwhash_OPSLIMIT_INTERACTIVE` and `crypto_pwhash_MEMLIMIT_INTERACTIVE` provide base line for these two parameters. This requires 32 Mb of dedicated RAM. Higher values may improve security (see below).

Alternatively, `crypto_pwhash_OPSLIMIT_MODERATE` and `crypto_pwhash_MEMLIMIT_MODERATE` can be used. This requires 128 Mb of dedicated RAM, and takes about 0.7 seconds on a 2.8 Ghz Core i7 CPU.

For highly sensitive data and non-interactive operations, `crypto_pwhash_OPSLIMIT_SENSITIVE` and `crypto_pwhash_MEMLIMIT_SENSITIVE` can be used. With these parameters, deriving a key takes about 3.5 seconds on a 2.8 Ghz Core i7 CPU and requires 512 Mb of dedicated RAM.

The `salt` should be unpredictable. `randombytes_buf()` is the easiest way to fill the `crypto_pwhash_SALTBYTES` bytes of the salt.

Keep in mind that in order to produce the same key from the same password, the same salt, and the same values for `opslimit` and `memlimit` have to be used. Therefore, these parameters have to be stored for each user.

The function returns `0` on success, and `-1` if the computation didn't complete, usually because the operating system refused to allocate the amount of requested memory.



## 选择参数的指导

从确定这个函数能使用多少内存开始。最多有多少个线程/进程并发执行这些函数? (理想情况是，每个 CPU 不超过 1个线程/进程)。有多少物理内存可以确保可用？


把 `memlimit` 设置成你想保留用来做 password 哈希的内存大小。

然后 把 `opslimit ` 设置成 `3` 并测试它 哈希一个密码使用的时间。

如果这对你的应用程序对你来说太久了，减少 `memlimit`，但是保持 `opslimit` 为 `3`。

如果这个函数太快了，你可以接受它计算更密集一些，而没有可用性问题。 那就增大 `opslimit`。

对在线应用来说  ( 例如登录一个网站 ), 1 秒钟应该是个可以接受的最大值。 

对交互应用来说 （例如桌面应用）， 输入密码后，5 秒钟的暂停应该是可以接受的，如果在一次会话中不需要多次输入密码的话。

对 非交互应用 和 不频繁应用（比如 恢复一个加密的备份）来说， 更慢的计算也是可接受的。


但是针对 brute-force 暴力攻击密码破解的最好防御，仍然是使用强密码，像 [passwdqc](http://www.openwall.com/passwdqc/) 这样的库，可以用来确保强密码。

## 常量

- `crypto_pwhash_ALG_DEFAULT`
- `crypto_pwhash_SALTBYTES`
- `crypto_pwhash_STRBYTES`
- `crypto_pwhash_STRPREFIX`
- `crypto_pwhash_OPSLIMIT_INTERACTIVE`
- `crypto_pwhash_MEMLIMIT_INTERACTIVE`
- `crypto_pwhash_OPSLIMIT_MODERATE`
- `crypto_pwhash_MEMLIMIT_MODERATE`
- `crypto_pwhash_OPSLIMIT_SENSITIVE`
- `crypto_pwhash_MEMLIMIT_SENSITIVE`

## Notes

`opslimit`, 计算的遍数，最少必须是 `3`。 对更小的值，`crypto_pwhash()` 和 `crypto_pwhash_str()` 会失败返回 `-1` 。

对 `memlimit` 没有 "不安全"值，尽管内存越多越好。

不要忘了用  `sodium_init()` 初始化整个库。 `crypto_pwhash_*` 在没有初始化的情况下仍然会工作，但是可能会变慢。

不要使用常量  (包括 `crypto_pwhash_OPSLIMIT_*` 和 `crypto_pwhash_MEMLIMIT_*`) 来验证一个 password ，把这些参数和 hash 保存在一起，并且使用保存的参数来做验证。

或者,使用 `crypto_pwhash_str()` 和 `crypto_pwhash_str_verify()`, 这组函数 会自动处理 包含 和提取这些参数的细节。

通过这样做，以后如果需要，password 以后可以使用不同的参数重新哈希。

明文的密码不再需要以后，就不应该再停留在内存里。

高度推荐使用 `sodium_mlock()` 来把存储明文密码的内存区域锁定住，并在 `crypto_pwhash_str()` 和 `crypto_pwhash_str_verify()` 返回后，调用 `sodium_munlock()` 。

`sodium_munlock()` 在 unlock 之前， 会用 0 覆盖这个区域，所以不需要再自己做覆盖操作。

## Algorithm details

- [Argon2i v1.3](https://github.com/P-H-C/phc-winner-argon2/raw/master/argon)

