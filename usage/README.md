# 使用方法

```c
#include <sodium.h>

int main(void)
{
    if (sodium_init() == -1) {
        return 1;
    }
    ...
}
```

`sodium.h` 是唯一需要包含的头文件。

库的名字是 `sodium` (使用 `-lsodium` 来链接)。 在支持的系统上，一般可以使用 `pkg-config` 来获取 编译 和 链接 的参数。 

```bash
CFLAGS=$(pkg-config --cflags libsodium)
LDFLAGS=$(pkg-config --libs libsodium)
```

对静态链接，Visual Studio 用户需要定义  `SODIUM_STATIC=1` 和 `SODIUM_EXPORT=` ，其他平台不需要。

`sodium_init()` 初始化 libsodium 库，并且必须在 libsodium 库的其他任何函数之前被调用。这个函数可以被多次重复调用，但是不能多个线程并发调用，如果你的程序中有这种场景，你应该自己加锁。

`sodium_init()`返回后，libsodium 的其他所有函数都是 线程安全 的。

`sodium_init()` 不会做任何 内存分配。 但是，在 Unix 系统中， 会打开 `/dev/urandom` 文件，并且会保持这个文件 fd 打开，因此在`chroot()`之后，`/dev/urandom`仍然能使用。多次调用 `sodium_init()` 会导致多个 fd 被打开。


`sodium_init()` 返回 `0` 表示 成功, `-1` 表示 失败,  `1` 表示库已经被初始化过了。
