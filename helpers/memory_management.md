# 安全的内容分配

## 内存清零

```c
void sodium_memzero(void * const pnt, const size_t len);
```

使用之后，敏感数据会被覆盖。但是 `memset()`手写的代码会被编译器优化或链接器悄悄地删掉。 

 `sodium_memzero()` 函数尝试有效地清零 `pnt` 指向的 `len` 字节，不管编译器如何优化。

## 锁内存

```c
int sodium_mlock(void * const addr, const size_t len);
```

 `sodium_mlock()` 函数锁住 最少从 `addr` 开始的  `len` 字节。 这可以帮助防止 敏感数据被  swap 到磁盘进而泄露出去。
 
另外，在处理敏感数据的机器华山，建议完全禁用 swap 分区，或者，作为第二选择，使用加密的 swap 分区。

为了类似原因， 在 Unix 类系统上，在非开发环境运行密码学代码时，也应该禁用 core dump。
这可以通过使用 shell 内置命令 `ulimit` 来实现，或者通过编程调用 `setrlimit(RLIMIT_CORE, &(struct rlimit) {0, 0})` 实现。
在支持内核 crash dump 的操作系统上，内核 crash dump 也应该禁用掉。

`sodium_mlock()` 封装 了`mlock()` 或 `VirtualLock()`. 
**Note:** 很多系统限制了一个进程可以 锁定的内存的量。当调大这些限制的时候，应该小心。当必要的时候，`sodium_lock()` 会返回 `-1` 表示触发了限制。

```c
int sodium_munlock(void * const addr, const size_t len);
```

被锁住的内存不再需要之后，应该调用 `sodium_munlock()` 函数。
`sodium_munlock()` 函数会用0填充 从 `addr` 开始的 `len` 字节内存，然后把这些页标记成可以 swap。
因此不需要在 `sodium_munlock()` 之前调用 `sodium_memzero()`。

在支持的系统上，`sodium_mlock()` 也封装了 `madvise()` ，并且 建议 内核不要把 锁住的内存区间包括在 core dump 文件中。
`sodium_unlock()` 也会取消这个额外保护。


## 保护的堆内存分配 

Sodium provides heap allocation functions for storing sensitive data.

These are not general-purpose allocation functions. In particular, they are slower than `malloc()` and friends, and they require 3 or 4 extra pages of virtual memory.

`sodium_init()` has to be called before using any of the guarded heap allocation functions.

```c
void *sodium_malloc(size_t size);
```

The `sodium_malloc()` function returns a pointer from which exactly `size` contiguous bytes of memory can be accessed.

The allocated region is placed at the end of a page boundary, immediately followed by a guard page. As a result, accessing memory past the end of the region will immediately terminate the application.

A canary is also placed right before the returned pointer. Modification of this canary are detected when trying to free the allocated region with `sodium_free()`, and also cause the application to immediately terminate.

An additional guard page is placed before this canary to make it less likely for sensitive data to be accessible when reading past the end of an unrelated region.

The allocated region is filled with `0xd0` bytes in order to help catch bugs due to initialized data.

In addition, `sodium_mlock()` is called on the region to help avoid it being swapped to disk. On operating systems supporting `MAP_NOCORE` or `MADV_DONTDUMP`, memory allocated this way will also not be part of core dumps.

The returned address will not be aligned if the allocation size is not a multiple of the required alignment.

For this reason, `sodium_malloc()` should not be used with packed or variable-length structures, unless the size given to `sodium_malloc()` is rounded up in order to ensure proper alignment.

All the structures used by libsodium can safely be allocated using `sodium_malloc()`, the only one requiring extra care being `crypto_generichash_state`, whose size needs to be rounded up to a multiple of 64 bytes.

Allocating `0` bytes is a valid operation, and returns a pointer that can be successfully passed to `sodium_free()`.

```c
void *sodium_allocarray(size_t count, size_t size);
```

The `sodium_allocarray()` function returns a pointer from which `count` objects that are `size` bytes of memory each can be accessed.

It provides the same guarantees as `sodium_malloc()` but also protects against arithmetic overflows when `count * size` exceeds `SIZE_MAX`.

```c
void sodium_free(void *ptr);
```

The `sodium_free()` function unlocks and deallocates memory allocated using `sodium_malloc()` or `sodium_allocarray()`.

Prior to this, the canary is checked in order to detect possible buffer underflows and terminate the process if required.

`sodium_free()` also fills the memory region with zeros before the deallocation.

This function can be called even if the region was previously protected using `sodium_mprotect_readonly()`; the protection will automatically be changed as needed.

`ptr` can be `NULL`, in which case no operation is performed.

```c
int sodium_mprotect_noaccess(void *ptr);
```

The `sodium_mprotect_noaccess()` function makes a region allocated using `sodium_malloc()` or `sodium_allocarray()` inaccessible. It cannot be read or written, but the data are preserved.

This function can be used to make confidential data inaccessible except when actually needed for a specific operation.

```c
int sodium_mprotect_readonly(void *ptr);
```

The `sodium_mprotect_readonly()` function marks a region allocated using `sodium_malloc()` or `sodium_allocarray()` as read-only.

Attempting to modify the data will cause the process to terminate.

```c
int sodium_mprotect_readwrite(void *ptr);
```

The `sodium_mprotect_readwrite()` function marks a region allocated using `sodium_malloc()` or `sodium_allocarray()` as readable and writable, after having been protected using `sodium_mprotect_readonly()` or `sodium_mprotect_noaccess()`.
