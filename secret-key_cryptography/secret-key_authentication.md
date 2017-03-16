# 对称密码学 认证算法

## 例子

```c
#define MESSAGE (const unsigned char *) "test"
#define MESSAGE_LEN 4

unsigned char key[crypto_auth_KEYBYTES];
unsigned char mac[crypto_auth_BYTES];

randombytes_buf(key, sizeof key);
crypto_auth(mac, MESSAGE, MESSAGE_LEN, key);

if (crypto_auth_verify(mac, MESSAGE, MESSAGE_LEN, key) != 0) {
    /* message forged! */
}
```

## 目的

这个操作使用一个 key 计算一个消息的 认证tag，并提供一种方式来验证一个 tag 对 给定消息和 key 是否合法。

这个函数确定地计算 认证tag： 相同的 (消息， key) 二元组，总是产生出相同的输出 tag。
然后，就算消息是公开的，也要求知道 key 才能计算出合法的 tag。 因此，key 应该保持保密。 tag 是可以公开的。


一个典型的使用案例是：
- `A` 准备一个消息，加上一个认证tag，发送给 `B`。
- `A` 不存储这条消息。
- 过一会，`B` 发送这条消息和认证 tag 给 `A`。
- `A` 使用认证 tag 来验证 确实是自己创建了这条消息。

这种操作 **并不加密** 这条消息，只是生成和验证 认证tag。


## 使用

```c
int crypto_auth(unsigned char *out, const unsigned char *in,
                unsigned long long inlen, const unsigned char *k);
```

 `crypto_auth()` 函数用  key `k`，为 长度是 `inlen` 的消息 `in` 计算出 tag， `k` 必须是 `crypto_auth_KEYBYTES` 字节长.
本函数把 tag 放入  `out`，tag 是 `crypto_auth_BYTES` 字节长.

```c
int crypto_auth_verify(const unsigned char *h, const unsigned char *in,
                       unsigned long long inlen, const unsigned char *k);
```

 `crypto_auth_verify()` 函数 用 key `k` 来验证 tag `h` 是 长度 `inlen` 的 消息 `in` 的合法 tag 。
 返回 `-1` 表示验证失败，`0`表示验证成功通过。
 
 
## 常量

- `crypto_auth_BYTES`
- `crypto_auth_KEYBYTES`

## 算法细节 

- HMAC-SHA512256

