# AEAD 认证加密 (Authenticated Encryption with Additional Data)

本操作:
- 用一个 key 和 nonce 加密一个消息，来保密。
- 计算一个 认证tag。这个tag 用来确保这个消息的密文，和一段可选的不保密不加密的附加数据(Additional Data)， 没有被篡改过。

一段  additional data 的典型场景，是存储协议特定的，关于消息的元数据，例如消息的长度和编码。

## 支持的构造

Libsodium 支持两种流行的构造：AES256-GCM 和 ChaCha20-Poly1305.

### AES256-GCM

当前的 AES256-GCM 实现是硬件加速的，并且要求 Intel  SSE3 扩展指令集，和 `aesni` 和 `pclmul` 系列指令。

Intel Westmere 处理器 ( 2010 出现)  和更新的处理器满足这些要求。

不计划支持 非硬件加速的 AES-GCM 实现。

如果不用考虑可移植性，, AES256-GCM 是最快的。


### ChaCha20-Poly1305

尽管 AES 在专用硬件上非常快，但是它在没有硬件加速的平台上，性能一般认为不高。另一个问题是很多 AES 的纯软件实现有 cache-collision timing 攻击漏洞，缓存冲突时间侧通道漏洞。

就纯软件实现来说， 一般认为 ChaCha20 的 性能比 AES 好, 在缺乏 AES 硬件的平台上，ChaCha20 大概是 AES 的 三倍 性能。 并且 ChaCha20 免疫  时间侧信道攻击。

Poly1305 是一个高性能的 MAC ( message authentication code， 消息认证码 )。

2014年，提出 ChaCha20 流加密算法和 Poly1305 消息认证码的组合，作为被深入研究的 Salsa20-Poly1305 构造的更快的替代品。

 ChaCha20-Poly1305 在主流操作系统上都有实现，Web 浏览器 和 密码学库 中随后也都有了实现。2015 年最终成为了正式的  IETF 标准。

libsodium 中的 ChaCha20-Poly1305 实现在所有支持的平台上都是可移植的，并且对绝大多数应用程序来说，都是推荐的首选。
