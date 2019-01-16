# 关于malloc

---

c++中的new默认实现是依赖malloc完成的。而malloc是C/C++标准库中规定的内存申请函数，函数的实现由库(操作系统)控制。

在linux上，malloc分为两层，分别是用户层与内核层。

## 用户层

在这个层面上，堆空间分成了*已分配内存块*与*空闲内存块*两个链表。当使用malloc时会从空闲内存块链表中查找适合大小的空闲内存块，将其挂到已分配内存块链表下，并进行返回，当free时，就把分配的内存块挂到空闲内存块链表下，等待下一个malloc使用。

## 内核层

然而当malloc没办法从空闲内存块链表下获得足够的内存空间时，就需要通过系统调用完成内存申请。操作系统通常对虚拟内存空间中，堆部分的限制地址进行修改，并完成虚拟地址空间到物理地址空间的映射关系，从而完成内存的申请并返回。

malloc和free一般不会频繁进行系统调用，而是以链表的方式进行缓存，对于free而言，只有当空闲内存块足够大时(比如40KB)才返回给操作系统。

这也是为什么内存管理器从一开始就申请足够大的内存空间进行内存管理，就是避免过多调用malloc/new/free/delete，从而避免不可控的系统调用。



所以，内存管理器的一个作用，就是通过预先向系统申请足够大的内存空间，并采用自己的内存控制算法管理内存分配，而不依赖系统的默认实现，将绝大部分内存控制的权力掌握在自己手中。



## 内存对齐 alignment

vc和gcc默认会返回8或16字节对齐的内存地址，但这并非一定保证，暂时没有完整的资料支持该说法。所以有的库还是会对malloc返回的内存地址进行检查。

假如需要对齐更大的字节，需要手动设置，考虑以下代码[参考自google的mathfu](https://google.github.io/mathfu)

```c++
#define MATHFU_ALIGNMENT 16
inline void *AllocateAligned(size_t n) {
#if defined(_MSC_VER) && _MSC_VER >= 1900  // MSVC 2015
  return _aligned_malloc(n, MATHFU_ALIGNMENT);
#else
  // 申请n + MATHFU_ALIGNMENT大小的内存
  // 保证一定能够在所申请的内存地址空间中找到MATHFU_ALIGNMENT对齐的地址
  uint8_t *buf = reinterpret_cast<uint8_t *>(malloc(n + MATHFU_ALIGNMENT));
  if (!buf) return NULL;
  // 从buf开始寻找紧邻它的下一个MATHFU_ALIGNMENT对齐的地址
  // 由于buf是堆中的空间，堆是向高地址生长的，而buf指向最低的地址
  // 所以才有 reinterpret_cast<size_t>(buf) + MATHFU_ALIGNMENT
  uint8_t *aligned_buf = reinterpret_cast<uint8_t *>(
      (reinterpret_cast<size_t>(buf) + MATHFU_ALIGNMENT) &
      ~(MATHFU_ALIGNMENT - 1));
  // 检查找出来的地址和申请的堆地址之间的空间是否能够容纳一个指针大小
  // 需要使用这个空间来存储堆的原始地址，以保证释放时候的地址准确
  assert(static_cast<size_t>(aligned_buf - buf) >= sizeof(void *));
  *(reinterpret_cast<uint8_t **>(aligned_buf) - 1) = buf;
  return aligned_buf;
#endif  // defined(_MSC_VER) && _MSC_VER >= 1900 // MSVC 2015
}

inline void FreeAligned(void *p) {
#if defined(_MSC_VER) && _MSC_VER >= 1900  // MSVC 2015
  _aligned_free(p);
#else
  if (p == NULL) return;
  free(*(reinterpret_cast<uint8_t **>(p) - 1));
#endif  // defined(_MSC_VER) && _MSC_VER >= 1900 // MSVC 2015
}
```

