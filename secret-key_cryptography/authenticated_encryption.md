# 对称密码学 认证加密 AEAD

## 例子

```c
#define MESSAGE ((const unsigned char *) "test")
#define MESSAGE_LEN 4
#define CIPHERTEXT_LEN (crypto_secretbox_MACBYTES + MESSAGE_LEN)

unsigned char nonce[crypto_secretbox_NONCEBYTES];
unsigned char key[crypto_secretbox_KEYBYTES];
unsigned char ciphertext[CIPHERTEXT_LEN];

randombytes_buf(nonce, sizeof nonce);
randombytes_buf(key, sizeof key);
crypto_secretbox_easy(ciphertext, MESSAGE, MESSAGE_LEN, nonce, key);

unsigned char decrypted[MESSAGE_LEN];
if (crypto_secretbox_open_easy(decrypted, ciphertext, CIPHERTEXT_LEN, nonce, key) != 0) {
    /* message forged! */
}
```

## 目的

这个操作:
- 用 一个 key  和 一个 nonce 加密一个消息，让消息保持保密。
- 计算一个 认证 tag。 这个tag 用于确保消息在解密之前没有被篡改过。


一个单一的 key 同时用于 加密/签名 ,  和 验证/解密 消息。 因此，保证 key 保密，是极其重要的。
nonce不需要保密，但是绝对不能在同一个key下重复使用。生成一个 nonce 的最简单办法是使用 `randombytes_buf()`。


## Combined 模式


在 combined 模式, 认证  和 消息密文 存储在一起。 一般需求都是这样的。

```c
int crypto_secretbox_easy(unsigned char *c, const unsigned char *m,
                          unsigned long long mlen, const unsigned char *n,
                          const unsigned char *k);
```

 `crypto_secretbox_easy()`  函数加密一个消息  `mlen` 字节长的消息`m` , 使用 key  `k` 和 nonce `n`。
 
`k` 应该是 `crypto_secretbox_KEYBYTES` 字节长，并且 `n`应该是 `crypto_secretbox_NONCEBYTES` 字节长.

`c` 应该最少是 `crypto_secretbox_MACBYTES + mlen` 字节长.

消息密文的长度也是 `mlen` 字节， 这个函数先在 `c` 中 写入 `crypto_secretbox_MACBYTES`  字节长的 认证tag ， 然后追加加密后的 消息密文  。

`c` 和 `m` 内存区间可以重叠, 因此可以做 ‘原地加密’ 。当然不要忘记 要有额外的 `crypto_secretbox_MACBYTES` 字节用于写入 认证 tag。


```c
int crypto_secretbox_open_easy(unsigned char *m, const unsigned char *c,
                               unsigned long long clen, const unsigned char *n,
                               const unsigned char *k);
```

`crypto_secretbox_open_easy()` 函数 验证 并 解密  `crypto_secretbox_easy()` 生成的密文。

`c` 是个指针，指向 `crypto_secretbox_easy()` 生成的  认证tag + 消息密文  的组合buffer。`clen` 是 这个组合的buffer的长度，换句话说，就是  `crypto_secretbox_MACBYTES` + 明文消息 的长度。


 nonce `n` 和 key `k` 必须和 加密/认证时候用的  nonce 和  相同。
 
这个函数返回 `-1`， 表示验证失败。
返回 `0` ，表示成功，并且解密出来的 消息 存入 `m` 中。

`m` 和 `c` 内存区间可以重叠，因此可以做原地解密。


## Detached 模式


一些程序可能需要 把 认证 tag  和 消息密文分开存储。

对这种使用场景， 可以使用"detached" 系列变种函数。

```c
int crypto_secretbox_detached(unsigned char *c, unsigned char *mac,
                              const unsigned char *m,
                              unsigned long long mlen,
                              const unsigned char *n,
                              const unsigned char *k);
```
这个函数加密 `mlen` 长度的消息 `m`，使用 key `k` 和 nonce `n`，并且把消息密文 存入 `c`，由于这个函数不会在前面写入 认证tag， 会 精确地写入 `mlen` 字节到 `c`。
 `crypto_secretbox_MACBYTES` 字节的认证tag，会写入 `mac`。
 
```c
int crypto_secretbox_open_detached(unsigned char *m,
                                   const unsigned char *c,
                                   const unsigned char *mac,
                                   unsigned long long clen,
                                   const unsigned char *n,
                                   const unsigned char *k);
```

 `crypto_secretbox_open_detached()` 函数验证并解密 长度是 `clen` 的消息密文 `c`，`clen` 不包括 认证tag，所以长度和明文相等。
 
在使用 key `k` 和 nonce `n` 验证确认 `mac` 是这个密文的合法认证tag后， 明文会写入`m`。

这个函数 返回 `-1` 如果验证失败, 返回 `0` 表示成功。


## 常量

- `crypto_secretbox_KEYBYTES`
- `crypto_secretbox_MACBYTES`
- `crypto_secretbox_NONCEBYTES`

## 算法细节

- 加密算法 Encryption: XSalsa20 流加密算法
- 认证算法 Authentication: Poly1305 MAC算法

## 注意

 NaCl 原来的 `crypto_secretbox` API 也支持， 尽管不推荐。

 
`crypto_secretbox()` 接受 一个指向 消息之前32字节的指针，并且把密文写入 目的地指针之后 16字节， 并把目的地指针的指向的前16字节填充成0 。 `crypto_secretbox_open()` 接受一个指向密文前 16字节的指针，并把消息存入 目的地指针的 32字节后， 并把目的地指针的前32字节填充成0。

 `_easy` 和 `_detached`  API 更快，并且改进了易用性，不再需要 padding 填充，拷贝，和指针计算。
