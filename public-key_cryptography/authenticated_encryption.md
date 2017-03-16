# 公钥密码学 认证加密 

## 例子

```c
#define MESSAGE (const unsigned char *) "test"
#define MESSAGE_LEN 4
#define CIPHERTEXT_LEN (crypto_box_MACBYTES + MESSAGE_LEN)

unsigned char alice_publickey[crypto_box_PUBLICKEYBYTES];
unsigned char alice_secretkey[crypto_box_SECRETKEYBYTES];
crypto_box_keypair(alice_publickey, alice_secretkey);

unsigned char bob_publickey[crypto_box_PUBLICKEYBYTES];
unsigned char bob_secretkey[crypto_box_SECRETKEYBYTES];
crypto_box_keypair(bob_publickey, bob_secretkey);

unsigned char nonce[crypto_box_NONCEBYTES];
unsigned char ciphertext[CIPHERTEXT_LEN];
randombytes_buf(nonce, sizeof nonce);
if (crypto_box_easy(ciphertext, MESSAGE, MESSAGE_LEN, nonce,
                    bob_publickey, alice_secretkey) != 0) {
    /* error */
}

unsigned char decrypted[MESSAGE_LEN];
if (crypto_box_open_easy(decrypted, ciphertext, CIPHERTEXT_LEN, nonce,
                         alice_publickey, bob_secretkey) != 0) {
    /* message for Bob pretending to be from Alice has been forged! */
}
```

## 目的

使用公钥密码学认证加密，Bob 可以加密一个保密消息，使用 Alice 的公钥，专门发给 Alice。
使用 Bob 的公钥，Alice 可以在解密消息之前，验证 消息密文是否 真的由 Bob 创建且没有被别人篡改过。
Alice 只需要 Bob 的公钥，nonce 和 消息的密文。 Bob 不需要把自己的密钥公开给别人，甚至不需要公开给 Alice。


并且，为了发送消息给 Alice ，Bob 只需要 Alice 的公钥，Alice 完全不需要公开自己的密钥，甚至不需要公开给 Bob。

Alice 可以使用同样的系统回复给 Bob，不需要再生成不同的 密钥对。

nonce 不需要保密，但是一个nonce，在一对公私钥下，只能使用在 `crypto_box_open_easy()` 的一次调用中，绝对不能复用。
一种生成 nonce 的简单方式是使用  `randombytes_buf()`, 考虑到 nonce 的大小， 随机数发生碰撞的概率是可以忽略的。对某些应用程序， 如果你希望使用 nonce 来探测丢失的消息，或者忽略被中间人重放的消息，简单地使用一个递增的计数器来作为 nonce 也是 ok 的。

当做计算的时候，你必须确保同样的 nonce 值绝对不会被 复用 \(例如你可能有多个 线程 或者 多个机器在用同一组 密钥同时加密消息\）。

这套系统提供了相互认证。然而 ，一种典型的使用场景是加密 一个 server 和 多个client 间的通信，server 的公钥是预先公开的，而 client 匿名地连上来，即没有预先公开的公钥。


## Key pair generation

```c
int crypto_box_keypair(unsigned char *pk, unsigned char *sk);
```

The `crypto_box_keypair()` function randomly generates a secret key and a corresponding public key. The public key is put into `pk` \(`crypto_box_PUBLICKEYBYTES` bytes\) and the secret key into `sk` \(`crypto_box_SECRETKEYBYTES` bytes\).

```c
int crypto_box_seed_keypair(unsigned char *pk, unsigned char *sk,
                            const unsigned char *seed);
```

Using `crypto_box_seed_keypair()`, the key pair can also be deterministically derived from a single key `seed` \(`crypto_box_SEEDBYTES` bytes\).

```c
int crypto_scalarmult_base(unsigned char *q, const unsigned char *n);
```

In addition, `crypto_scalarmult_base()` can be used to compute the public key given a secret key previously generated with `crypto_box_keypair()`:

```c
unsigned char pk[crypto_box_PUBLICKEYBYTES];
crypto_scalarmult_base(pk, sk);
```

## Combined mode

In combined mode, the authentication tag and the encrypted message are stored together. This is usually what you want.

```c
int crypto_box_easy(unsigned char *c, const unsigned char *m,
                    unsigned long long mlen, const unsigned char *n,
                    const unsigned char *pk, const unsigned char *sk);
```

The `crypto_box_easy()` function encrypts a message `m` whose length is `mlen` bytes, with a recipient's public key `pk`, a sender's secret key `sk` and a nonce `n`.

`n` should be `crypto_box_NONCEBYTES` bytes.

`c` should be at least `crypto_box_MACBYTES + mlen` bytes long.

This function writes the authentication tag, whose length is `crypto_box_MACBYTES` bytes, in `c`, immediately followed by the encrypted message, whose length is the same as the plaintext: `mlen`.

`c` and `m` can overlap, making in-place encryption possible. However do not forget that `crypto_box_MACBYTES` extra bytes are required to prepend the tag.

```c
int crypto_box_open_easy(unsigned char *m, const unsigned char *c,
                         unsigned long long clen, const unsigned char *n,
                         const unsigned char *pk, const unsigned char *sk);
```

The `crypto_box_open_easy()` function verifies and decrypts a ciphertext produced by `crypto_box_easy()`.

`c` is a pointer to an authentication tag + encrypted message combination, as produced by `crypto_box_easy()`.  
`clen` is the length of this authentication tag + encrypted message combination. Put differently, `clen` is the number of bytes written by `crypto_box_easy()`, which is `crypto_box_MACBYTES` + the length of the message.

The nonce `n` has to match the nonce used to encrypt and authenticate the message.

`pk` is the public key of the sender that encrypted the message. `sk` is the secret key of the recipient that is willing to verify and decrypt it.

The function returns `-1` if the verification fails, and `0` on success.  
On success, the decrypted message is stored into `m`.

`m` and `c` can overlap, making in-place decryption possible.

## Detached mode

Some applications may need to store the authentication tag and the encrypted message at different locations.

For this specific use case, "detached" variants of the functions above are available.

```c
int crypto_box_detached(unsigned char *c, unsigned char *mac,
                        const unsigned char *m,
                        unsigned long long mlen,
                        const unsigned char *n,
                        const unsigned char *pk,
                        const unsigned char *sk);
```

This function encrypts a message `m` of length `mlen` with a nonce `n` and a secret key `sk` for a recipient whose public key is `pk`, and puts the encrypted message into `c`.

Exactly `mlen` bytes will be put into `c`, since this function does not prepend the authentication tag.

The tag, whose size is `crypto_box_MACBYTES` bytes, will be put into `mac`.

```c
int crypto_box_open_detached(unsigned char *m,
                             const unsigned char *c,
                             const unsigned char *mac,
                             unsigned long long clen,
                             const unsigned char *n,
                             const unsigned char *pk,
                             const unsigned char *sk);
```

The `crypto_box_open_detached()` function verifies and decrypts an encrypted message `c` whose length is `clen` using the recipient's secret key `sk` and the sender's public key `pk`.

`clen` doesn't include the tag, so this length is the same as the plaintext.

The plaintext is put into `m` after verifying that `mac` is a valid authentication tag for this ciphertext, with the given nonce `n` and key `k`.

The function returns `-1` if the verification fails, or `0` on success.

## Precalculation interface

Applications that send several messages to the same receiver or receive several messages from the same sender can gain speed by calculating the shared key only once, and reusing it in subsequent operations.

```c
int crypto_box_beforenm(unsigned char *k, const unsigned char *pk,
                        const unsigned char *sk);
```

The `crypto_box_beforenm()` function computes a shared secret key given a public key `pk` and a secret key `sk`, and puts it into `k` \(`crypto_box_BEFORENMBYTES` bytes\).

```c
int crypto_box_easy_afternm(unsigned char *c, const unsigned char *m,
                            unsigned long long mlen, const unsigned char *n,
                            const unsigned char *k);

int crypto_box_open_easy_afternm(unsigned char *m, const unsigned char *c,
                                 unsigned long long clen, const unsigned char *n,
                                 const unsigned char *k);

int crypto_box_detached_afternm(unsigned char *c, unsigned char *mac,
                                const unsigned char *m, unsigned long long mlen,
                                const unsigned char *n, const unsigned char *k);

int crypto_box_open_detached_afternm(unsigned char *m, const unsigned char *c,
                                     const unsigned char *mac,
                                     unsigned long long clen, const unsigned char *n,
                                     const unsigned char *k);
```

The `_afternm` variants of the previously described functions accept a precalculated shared secret key `k` instead of a key pair.

Like any secret key, a precalculated shared key should be wiped from memory \(for example using `sodium_memzero()`\) as soon as it is not needed any more.

## Constants

* `crypto_box_PUBLICKEYBYTES`
* `crypto_box_SECRETKEYBYTES`
* `crypto_box_MACBYTES`
* `crypto_box_NONCEBYTES`
* `crypto_box_SEEDBYTES`
* `crypto_box_BEFORENMBYTES`

## Algorithm details

* Key exchange: X25519
* Encryption: XSalsa20 stream cipher
* Authentication: Poly1305 MAC

## Notes

The original NaCl `crypto_box` API is also supported, albeit not recommended.

`crypto_box()` takes a pointer to 32 bytes before the message, and stores the ciphertext 16 bytes after the destination pointer, the first 16 bytes being overwritten with zeros. `crypto_box_open()` takes a pointer to 16 bytes before the ciphertext and stores the message 32 bytes after the destination pointer, overwriting the first 32 bytes with zeros.

The `_easy` and `_detached` APIs are faster and improve usability by not requiring padding, copying or tricky pointer arithmetic.

