---
layout: post
title: 「草稿」 ptmalloc的一些参数
categories: [Memory, Operating System]
description: 本文贴一些ptmalloc（glibc2.31版本）简单的参数，以及用到的便利函数
---

<!--more-->


## 前言

本文贴一些ptmalloc（glibc2.31版本）简单的参数，以及用到的便利函数

有必要说明一下，作者对这些取值的解释基本都是in practice（也有少数是必要的设计），因此也不用过于纠结，见识一下吧

## 背景

这里就不介绍`ptmalloc`的完整设计了，一方面是自己也没时间全部分析，另一方面是网上挺多优质介绍的

我目前对`ptmalloc`里的数据结构做了一些简单RTFSC分析，和下面的内容强关联，感兴趣也可看看：

[malloc at master · Caturra000/RTFSC · GitHub](https://github.com/Caturra000/RTFSC/tree/master/glibc/malloc)

## 最小大小

```C
/* The smallest size we can malloc is an aligned minimal chunk */
#define MINSIZE  \
  (unsigned long)(((MIN_CHUNK_SIZE+MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK))

/* The smallest possible chunk */
#define MIN_CHUNK_SIZE        (offsetof(struct malloc_chunk, fd_nextsize))

/* The corresponding bit mask value.  */
#define MALLOC_ALIGN_MASK (MALLOC_ALIGNMENT - 1)

/* MALLOC_ALIGNMENT is the minimum alignment for malloc'ed chunks.  It
   must be a power of two at least 2 * SIZE_SZ, even on machines for
   which smaller alignments would suffice. It may be defined as larger
   than this though. Note however that code and data structures are
   optimized for the case of 8-byte alignment.  */
#define MALLOC_ALIGNMENT (2 * SIZE_SZ < __alignof__ (long double) \
              ? __alignof__ (long double) : 2 * SIZE_SZ)
// 注：long double为16字节

/* The corresponding word size.  */
#define SIZE_SZ (sizeof (INTERNAL_SIZE_T))
// 等同于size_t
```
这里使用`+`和`~`操作进行对齐补位

MASK需要二进制全1111

这样算出来是MASK+1的倍数（包括0）

`MINSIZE`等价于`MIN_CHUNK_SIZE`

<br>

由offset可得，具体完整大小在64位下是32字节

既`chunk header` + `fd` + `bk`

这应该是指带元数据下的大小，如果是用户请求，那就是最小16字节

因此你尝试`malloc(0)`可能是直接padding到16字节的
（但是也允许返回`errno`）

### follow up：最大大小呢？

见上面RTFSC链接

## 对齐判断

```C
/* Check if m has acceptable alignment */
#define aligned_OK(m)  (((unsigned long)(m) & MALLOC_ALIGN_MASK) == 0)
```
## mmap阈值

```C
/* There is only one instance of the malloc parameters.  */
static struct malloc_par mp_ =
{
  .top_pad = DEFAULT_TOP_PAD,
  .n_mmaps_max = DEFAULT_MMAP_MAX,
  .mmap_threshold = DEFAULT_MMAP_THRESHOLD,
  .trim_threshold = DEFAULT_TRIM_THRESHOLD,
#define NARENAS_FROM_NCORES(n) ((n) * (sizeof (long) == 4 ? 2 : 8))
  .arena_test = NARENAS_FROM_NCORES (1)
#if USE_TCACHE
  ,
  .tcache_count = TCACHE_FILL_COUNT,
  .tcache_bins = TCACHE_MAX_BINS,
  .tcache_max_bytes = tidx2usize (TCACHE_MAX_BINS-1),
  .tcache_unsorted_limit = 0 /* No limit.  */
#endif
};

#ifndef DEFAULT_MMAP_THRESHOLD
#define DEFAULT_MMAP_THRESHOLD DEFAULT_MMAP_THRESHOLD_MIN
#endif

/*
  MMAP_THRESHOLD_MAX and _MIN are the bounds on the dynamically
  adjusted MMAP_THRESHOLD.
*/
#ifndef DEFAULT_MMAP_THRESHOLD_MIN
#define DEFAULT_MMAP_THRESHOLD_MIN (128 * 1024)
#endif
```

mmap阈值并不区分32位还是64位，总是128KB

## bin划分

```C
#define NBINS             128
```

除`fast bins`外，总个数为128

```C
#define NFASTBINS  (fastbin_index (request2size (MAX_FAST_SIZE)) + 1)

/* The maximum fastbin request size we support */
#define MAX_FAST_SIZE     (80 * SIZE_SZ / 4)

/* pad request bytes into a usable size -- internal version */
/* Note: This must be a macro that evaluates to a compile time constant
   if passed a literal constant.  */
#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)

/* offset 2 to use otherwise unindexable first 2 bins */
#define fastbin_index(sz) \
  ((((unsigned int) (sz)) >> (SIZE_SZ == 8 ? 4 : 3)) - 2)
```

手算是不可能的，放到vs code里看看

算出`fast bins`个数NFASTBINS为10

并且可以知道，`fast bins`的最大阈值为160字节

```C
#define NSMALLBINS         64
```

`small bins`的个数是64？其实是62，看RTFSC怎么算的

```C
#define MIN_LARGE_SIZE    ((NSMALLBINS - SMALLBIN_CORRECTION) * SMALLBIN_WIDTH)
```

这是用于划分small bins和large bins的值，64位下位1024

既大于等于1024的chunk才有资格放入large bin

TODO 公差数组

## References

[malloc - Glibc source code (glibc-2.31) - Bootlin](https://elixir.bootlin.com/glibc/glibc-2.31/source/malloc)
