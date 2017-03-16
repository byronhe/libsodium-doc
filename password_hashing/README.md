# 用户密码 password 哈希函数

用来对 机密数据做加密或签名 的密钥 key ，必须从一个非常大的 key 空间中选择。

然而，用户密码 password 通常很短，人类生成的字符串，使得字典攻击变得可行。

用户密码哈希函数从一个用户密码 password 和一个盐salt，衍生出来任意长度的密钥 key 。

- 生成的 key 可以是 应用程序指定的任意长度，不管 password 多长。
- 相同的密码 password 和相同的参数，总是hash计算出相同的输出。
- 相同的密码 password 和不同的 盐 salt，会产生不同的输出。
- 从一个 password 和 salt 衍生出 密钥 key 的函数，是 CPU 密集型计算，并且故意地要求很大量的内存。因此要求每次密码验证操作都需要很大的耗费，这样做到免疫暴力破解。

通用的使用案例:
- 密码 password 存储，或者 存储验证密码 password 所需要的，而不用实际存储密码 password。
- 从一个 password 衍生出 加密 key， 例如 用于磁盘加密。

Sodium 的 高层 `crypto_pwhash_*`  API 系列实际使用  Argon2 函数。

特定的  `crypto_pwhash_scryptsalsa208sha256_*` API 使用更保守的，更广泛部署的 Scrypt 函数。


## Argon2

Argon2 针对 x86 架构优化，并且利用了 较新的 Intel 和  AMD 处理器的 cache 和 内存组织。 但是 它的实现 在其它架构上保持 可移植，和快速。

Argon2 有两个变种: Argon2d 和 Argon2i. Argon2i 使用 数据依赖的内存访问，更适合用于 用户密码 password 哈希 ，和 基于 password 的 密钥衍生。
 Argon2i 同时还对内存有多遍访问，来免疫 tradeoff 攻击。
 
这是 从 libsodium 1.0.9 开始实现的变种。

推荐用户首选 Argon2 ，其次 Scrypt 。 如果 不需要顾忌 要求 libsodium 版本 >= 1.0.9 的话.

## Scrypt

Scrypt 也需要耗费大量内存资源，来使得进行大规模定制硬件攻击耗费巨大。

尽管 Scrypt 的内存耗费可以通过一定额外计算的办法显著减少，只要选择了恰当的参数，Scrypt 仍然是当前一个很棒的选择。

Scrypt 在 libsodium 中从版本 0.5.0 开始可用, 因此如果需要考虑和老版本 libsodium 的兼容性，Scrypt 是比  Argon2 更好的选择。
