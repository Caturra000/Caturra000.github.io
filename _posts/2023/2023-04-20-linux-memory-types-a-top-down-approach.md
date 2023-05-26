---
layout: post
title: Linux内存类型：自顶向下方法
categories: [Memory, kernel]
description: 这是一篇翻新的旧文，自己回顾用。另外，标题风格只是neta某本CS经典书籍，没别的意思
---

## 前前言

这是一篇翻新的旧文，自己回顾用。另外，标题风格只是neta某本CS经典书籍，没别的意思

## 前言

我并没有在搜索引擎上找到“内存的类型”这个关键字，有点困惑为什么前人不做点总结。或许是资料上比较零碎，因此打算自己写一篇。至于“自顶向下”，其实是指从简单的工具来展开这个话题。如果是自底向上，那[关于内存的话题](/archives/kernel-mm-overview/)可能永远说不完了

## 工具

我们关心常用的工具：比如`free` / `top` / `meminfo`

### free

最简单的莫过于`free`，它提供的信息基本就是`top`上内存相关的信息

```
caturra@LAPTOP-RU7SB7FE:~$ free -tl
              total        used        free      shared  buff/cache   available
Mem:       16642364     8006296     8406716       17720      229352     8502336
Low:       16642364     8235648     8406716
High:             0           0           0
Swap:      26214396      175388    26039008
Total:     42856760     8181684    34445724
```

### top

上面提到，`top`的信息基本来自`free`

但是仍有一个特别的地方——就是`%MEM`，字面意思是一个内存的占比

那么问题是，这个内存到底是什么内存？

### meminfo

公司里的dump信息全靠`meminfo`。虽然`meminfo`有2套工具，但是看`proc`文件系统下的就足够了（`/proc/meminfo`）

> 注：另一套是谷歌搞出来的，反正我不用

```
MemTotal:       16642364 kB
MemFree:         7243752 kB
Buffers:           34032 kB
Cached:           188576 kB
SwapCached:            0 kB
Active:           167556 kB
Inactive:         157876 kB
Active(anon):     103104 kB
Inactive(anon):    17440 kB
Active(file):      64452 kB
Inactive(file):   140436 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:      26214396 kB
SwapFree:       26043332 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        102824 kB
Mapped:            71404 kB
Shmem:             17720 kB
Slab:              13868 kB
SReclaimable:       6744 kB
SUnreclaim:         7124 kB
KernelStack:        2848 kB
PageTables:         2524 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:      515524 kB
Committed_AS:    3450064 kB
VmallocTotal:     122880 kB
VmallocUsed:       21296 kB
VmallocChunk:      66044 kB
HardwareCorrupted:     0 kB
AnonHugePages:      2048 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:       12280 kB
DirectMap4M:      897024 kB
```

这些内存类型非常的多，有些字面上比较含糊，比如`Buffers`和`Cached`，`MemFree`和`MemAvailable`

我们需要明白这些具体的含义

> 注：`/proc/meminfo`在不同的`linux`内核版本下有较大变化

## 解读top-%MEM

解读的原则很简单，先看文档，没文档或者看不懂再翻代码

在`man top`能找到这一句话：

```
%MEM - simply RES divided by total physical memory
```

total physical memory是很好理解的。但是RES是什么？再往下看：

```
RES  - anything occupying physical memory
```

它是一种占据物理内存的内存。似乎很废话，能不能更加具体一点？当然可以

```

    RES  - anything occupying physical memory which, beginning with
            Linux-4.5, is the sum of the following three fields:
            RSan - quadrant 1 pages, which include any
                former quadrant 3 pages if modified
            RSfd - quadrant 3 and quadrant 4 pages
            RSsh - quadrant 2 pages

...

       For each such process, every memory page is restricted to a single quadrant from the table below.  Both physi‐
       cal  memory  and  virtual memory can include any of the four, while the swap file only includes #1 through #3.
       The memory in quadrant #4, when modified, acts as its own dedicated swap file.



                                     Private | Shared
                                 1           |          2
            Anonymous  . stack               |
                       . malloc()            |
                       . brk()/sbrk()        | . POSIX shm*
                       . mmap(PRIVATE, ANON) | . mmap(SHARED, ANON)
                      -----------------------+----------------------
                       . mmap(PRIVATE, fd)   | . mmap(SHARED, fd)
          File-backed  . pgms/shared libs    |
                                 3           |          4

        2. %MEM  --  Memory Usage (RES)
           A task's currently resident share of available physical memory.
           See `OVERVIEW, Linux Memory Types' for additional details.


...

       23. RSan  --  Resident Anonymous Memory Size (KiB)
           A subset of resident memory (RES) representing private pages not mapped to a file.

       24. RSfd  --  Resident File-Backed Memory Size (KiB)
           A  subset  of resident memory (RES) representing the implicitly shared pages supporting program images and
           shared libraries.  It also includes explicit file mappings, both private and shared.

       25. RSlk  --  Resident Locked Memory Size (KiB)
           A subset of resident memory (RES) which cannot be swapped out.

       26. RSsh  --  Resident Shared Memory Size (KiB)
           A subset of resident memory (RES) representing the explicitly shared anonymous shm*/mmap pages.
```

> 注：`man`文档对`%MEM`的解释有些矛盾，一个是`RES`占比total physical，另一个是`RES`占比available physical，这是两个不同的概念（available很特殊）
。有疑问不如看看源码是拿什么做除法的，但我相信是第一种说法，因为第二种没意义啊

原来RES = **re**sident **s**hare of available physical memory，字面意思就是可用物理内存的驻留份额

简单点的理解就是RES会实际地占据物理内存

按照四象限划分（是否共享和是否映射）的方式来看，`RES = RSan + RSfd + RSsh`。其中：
- `RSan`：尚未映射到文件的私有页面，既所有的私有匿名映射页加上脏的非共享文件页
- `RSfd`：隐式共享页面，既文件页
- `RSsh`：共享匿名页面

说这么抽象我咋懂？翻下代码吧

> 注：在`top`的源码里，其`RES`是从`/proc/pid/statm`中直接获取的（不必再翻源码了，并不好读，关键就这一句话）

我们进一步来看下`proc`是怎么跑的

```C

// 文件：/fs/proc/array.c
int proc_pid_statm(struct seq_file *m, struct pid_namespace *ns,
            struct pid *pid, struct task_struct *task)
{
    unsigned long size = 0, resident = 0, shared = 0, text = 0, data = 0;
    struct mm_struct *mm = get_task_mm(task);
    if (mm) {
        size = task_statm(mm, &shared, &text, &data, &resident);
        mmput(mm);
    }
    /*
     * For quick read, open code by putting numbers directly
     * expected format is
     * seq_printf(m, "%lu %lu %lu %lu 0 %lu 0\n",
     *               size, resident, shared, text, data);
     */
    seq_put_decimal_ull(m, "", size);
    seq_put_decimal_ull(m, " ", resident);
    seq_put_decimal_ull(m, " ", shared);
    seq_put_decimal_ull(m, " ", text);
    seq_put_decimal_ull(m, " ", 0);
    seq_put_decimal_ull(m, " ", data);
    seq_put_decimal_ull(m, " ", 0);
    seq_putc(m, '\n');
    return 0;
}

// 文件：/fs/proc/task_mmu.c
unsigned long task_statm(struct mm_struct *mm,
             unsigned long *shared, unsigned long *text,
             unsigned long *data, unsigned long *resident)
{
    *shared = get_mm_counter(mm, MM_FILEPAGES) +
            get_mm_counter(mm, MM_SHMEMPAGES);
    *text = (PAGE_ALIGN(mm->end_code) - (mm->start_code & PAGE_MASK))
                                >> PAGE_SHIFT;
    *data = mm->data_vm + mm->stack_vm;
    *resident = *shared + get_mm_counter(mm, MM_ANONPAGES);
    return mm->total_vm;
}

// 文件：/include/linux/mm_types_task.h
enum {
    MM_FILEPAGES,   /* Resident file mapping pages */
    MM_ANONPAGES,   /* Resident anonymous pages */
    MM_SWAPENTS,    /* Anonymous swap entries */
    MM_SHMEMPAGES,  /* Resident shared memory pages */
    NR_MM_COUNTERS
};

```

可以看出，`RES = FILE + SHMEM + ANON`

> 注：关于overreport问题，由于`RES/RSS`有部分是直接拿取共享的内存（比如多个进程公用的共享库占用会重复计算），因此所有进程的RES总和是可以超过MemTotal的


> 注：
> 
> 那么这三个值`FILE | SHEMEM | ANON`又是怎么算出来的？
> 
> 其实statm太隐晦了，不如看/proc/pid/status
> 
> 可以直接拿到手，且是明显的加法关系
> 
> ```
> VmRSS:               956 kB
> RssAnon:             480 kB
> RssFile:             476 kB
> RssShmem:              0 kB
> ```

> 注：
>
> 而kernel的这些信息都来自于`get_mm_counter(member)`，当`page`归入到某个`mm`时，根据`PF`（page flags）来实现统计
>
> 这也侧面反映了一个内存分类复杂的原因，就是`PF`种类实在是太多啦

## 解读free

从`man`出发可以看到如下描述

```
DESCRIPTION
       free displays the total amount of free and used physical and swap memory in the system, as well as the buffers
       and caches used by the kernel. The information is gathered by parsing  /proc/meminfo.  The  displayed  columns
       are:

       total  Total installed memory (MemTotal and SwapTotal in /proc/meminfo)

       used   Used memory (calculated as total - free - buffers - cache)

       free   Unused memory (MemFree and SwapFree in /proc/meminfo)

       shared Memory used (mostly) by tmpfs (Shmem in /proc/meminfo)

       buffers
              Memory used by kernel buffers (Buffers in /proc/meminfo)

       cache  Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)

       buff/cache
              Sum of buffers and cache

       available
              Estimation  of how much memory is available for starting new applications, without swapping. Unlike the
              data provided by the cache or free fields, this field takes into account page cache and also  that  not
              all  reclaimable  memory  slabs will be reclaimed due to items being in use (MemAvailable in /proc/mem‐
              info, available on kernels 3.14, emulated on kernels 2.6.27+, otherwise the same as free)

```

可以知道，它的值也是从`meminfo`拿到手并作进一步整理的

其中：

- 所有的值讨论的都是物理内存，而不是只分配不映射的虚拟内存
- 具有简单映射关系的值有`total` / `free` / `shared` / `buffers`
  - `total`对应于`MemTotal` + `SwapTotal`
  - `free`对应于`MemFree` + `SwapFree`
  - `shared`对应于`Shmem`
  - `buffers`对应于`Buffers`
- `cache`包含page cache和slab对象，是`Cached`和`SReclaimable`的映射
- `used`就是所剩余的值，既`total` - `free` - `shared` - `buffers` - `cache`
- `available`是一个估算出来的可用内存值
  - 为什么不用`free`作为可用值？因为无关紧要的cache可以回收
  - 为什么不用`free` + `cache`作为可用值？因为不是所有的cache都能回收
  - 为什么要估算？不能准确计算吗？因为内存的类型很复杂
  - 它还揭露了一个复杂的问题：`reclaimable`并不是真的全部都可以reclaimable

从`free`映射到`meminfo`的表如下所示：

| free域     | meminfo域                        | **说明**                                                     |
| ---------- | -------------------------------- | ------------------------------------------------------------ |
| `total`      | `MemTotal` + `SwapTotal`             |                                                              |
| `used`       | ~                                | used指total减去(free+shared+buffers+cache)                   |
| `free`       | `MemFree` + `SwapFree`               |                                                              |
| `shared`     | `Shmem`                            | 虽然作者说是大多来自Shmem，但其实[代码](https://gitlab.com/procps-ng/procps/-/blob/0e98e0677752272405652fd0cb1b73933236f873/library/meminfo.c#L203)上（还有[这里](https://gitlab.com/procps-ng/procps/-/blob/0e98e0677752272405652fd0cb1b73933236f873/src/free.c#L383)）就是只算Shmem |
| `buffers`    | `Buffers`                          | 作者的解释是kernel buffer，其实这是很含糊的，具体来说是块设备的page cache |
| `cache`      | `Cached` + `SReclaimable`           | Cached作为page  cache大小，但是SReclaimable并不是slab全部    |
| `buff/cache` | `Buffers` + `Cached` + `SReclaimable` | 就是一个buffers+cache的总和                                  |
| `available`  | `MemAvaliable`                     | 这是一个用算法得到的估算值                                   |

> 注：man手册中并没有说到lo和hi，其实这个是HIGHMEM模型的问题，已经是过去的历史了

> 注：在过去的版本中，`used`虽然是个具体值，但并没有过多参考价值。因此在新版本的实现中，改为估算值`used = total - available`

总结：`free`来自`meminfo`，想要知道本质，就解读`meminfo`吧



## 解读meminfo

 

从上面可以知道`meminfo`相当复杂，我们可以从[man proc](https://man7.org/linux/man-pages/man5/proc.5.html#:~:text=defined during compilation.-,/proc/meminfo,-This file reports)文档中得到一些启发（非常非常长的文档）

 

这里做个简单的总结（userspace多是关心上半侧的类型）：

| 内存类型                | 简化换算                                                     | 其它备注                                                     |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `MemTotal`              | physical - .text                                             | total并不是全部，需要抠掉一些kernel代码段的大小              |
| `MemFree`               | lowFree  + HighFree                                          | HighFree基本就是0的意思                                      |
| `MemAvailable`          | MemFree  - (low reserved + high watermark) + pagecache + slab | 一个简单的估算方式是  所有zone的free  减去 low reserved 和 high watermark  加上 可回收的page cache 和 slab  这里的可回收是个估算值，可能直接取总数的1/2 |
| `Buffers`               | foreach bdev: ret += bdev->bd_inode->i_mapping->nr_pages     | 用于块设备的page cache  如果是文件系统操作则应对应于metadata，如inode，superblock  否则要算上直接读写bdev的mapping（本质就是bdev） |
| `Cached`                | page  cache, in-memory files only                            | in-memory引起的page  cache  因此需要排除SwapCached和Buffers     可以认为代表文件映射的内存和tmpfs（shmem）映射（因为也是file-backed）的内存 |
| `SwapCached`            |                                                              | 曾经换出，然后换入回来的内存，但是仍存在于swap文件  这是用于节省IO的，当已经换回来后，如果再次缺少内存，那就直接舍去这一部分，而不是换出回swap文件，从而节省IO |
| `Active`                |                                                              | LRU中标记active，一般不reclaim，但后续降级也有可能回收       |
| `Inactive`              |                                                              | LRU中标记inactive，很可能reclaim                             |
| `HighTotal`, `HighFree` |                                                              | ~                                                            |
| `LowTotal`, `LowFree`   |                                                              | ~                                                            |
| `SwapTotal`             |                                                              | ~                                                            |
| `SwapFree`              |                                                              | 已经被换出的内存，临时存储在disk                             |
| `Dirty`                 |                                                              | 标记为dirty的内存，仍未写回                                  |
| `Writeback`             |                                                              | 正在写回的内存                                               |
| `AnonPages`             |                                                              | 映射到userspace页表的无后备文件的内存                        |
| `HardwareCorrupted`     |                                                              | ~                                                            |
| `AnonHugePages`         |                                                              | 透明大页相关                                                 |
| `Mapped`                |                                                              | 已经被映射的文件                                             |
| `Shmem`                 | Shmem + tmpfs + mmap                                         | mmap中的共享匿名的实现方式就是套用的shmem核心代码            |
| `ShmemHugePages`        |                                                              | ~                                                            |
| `ShmemPmdMapped`        |                                                              | ~                                                            |
| `KReclaimable`          | SReclaimable + ...direct allocations                         | kernel分配出来的，但可被reclaim的内存                        |
| `Slab`                  |                                                              | ~                                                            |
| `SReclaimable`          | part of  Slab                                                | Slab的一部分，允许reclaim                                    |
| `SUnreclaim`            | part of  Slab                                                | Slab的一部分，不允许reclaim                                  |
| `PageTables`            |                                                              | ~                                                            |
| `VmallocTotal`          | vmalloc()                                                    |                                                              |
| `VmallocUsed`           | used  vmalloc()                                              |                                                              |
| `VmallocChunk`          | largest  contiguous and free vmalloc()                       |                                                              |

> 注：关于MemAvailable的计算可以参考这里：[RTFSC/MemAvailable统计](https://github.com/Caturra000/RTFSC/blob/master/linux/mm/meminfo/MemAvailable%E7%BB%9F%E8%AE%A1.c)

> 注：附上一个影响Buffers统计的[调用栈](https://elixir.bootlin.com/linux/v6.0/source/fs/buffer.c#L192)，从文件页逻辑块到扇区的过程中缓存的pagecache会被算上

另外我也写了一个`meminfo parser`来解析这些参数：

[Caturra000/meminfo-parser (github.com)](https://github.com/Caturra000/meminfo-parser)

[Snippets/memory at 0450906bd74c · Caturra000/Snippets (github.com)](https://github.com/Caturra000/Snippets/tree/0450906bd74c4f459046be2a4d98841c8821e76e/memory)

你可以通过这个小工具来实测一些demo，从而理解这些内存类型

当然我也写了些简单的测试，比如用不同的方式对文件分别读写100MB

> 注：其实内核也提供`sysinfo`的方式来获取`meminfo`的**部分**参数，只是它能得到的信息比较有限

`read/write`实测如下：

| meminfo          | read   | write  | read-blk | write-blk |
| ---------------- | ------ | ------ | -------- | --------- |
| `MemFree`        | `-100MB` | `-100MB` | `-200MB`   | `-200MB`    |
| `Buffers`        |        |        | `+100MB`   | `+100MB`    |
| `Cached`         | `+100MB` | `+100MB` | `+100MB`   | `+100MB`    |
| `Inactive(anon)` |        |        |          |           |
| `Inactive(file)` | `+100MB` | `+100MB` | `+200MB`   | `+200MB`    |
| `Dirty`          |        | `+100MB` |          | `+100MB`    |

```
*blk = block device，通过`losetup`模拟块设备
```



 

`mmap`实测如下：

| meminfo          | PA-read | PA-write | PF-read | PF-write | SA-read | SA-write | SF-read | SF-write |
| ---------------- | ------- | -------- | ------- | -------- | ------- | -------- | ------- | -------- |
| `MemFree`        |         | `-100MB`   | `-100MB`  | `-200MB`   | `-100MB`  | `-100MB`   | `-145MB`  | `-100MB`   |
| `MemAvailable`   |         | `-100MB`   |         | `-100MB`   | `-100MB`  | `-100MB`   |         |          |
| `Cached`         |         |          | `+100MB`  | `+100MB`   | `+100MB`  | `+100MB`   | `+135MB`  | `+100MB`   |
| `Inactive(anon)` |         | `+100MB`   |         | `+100MB`   | `+100MB`  | `+100MB`   |         |          |
| `Inactive(file)` |         |          | `+100MB`  | `+100MB`   |         |          | `+135MB`  | `+100MB`   |
| `AnonPages`      |         | `+100MB`   |         | `+100MB`   |         |          |         |          |
| `AnonHugePages`  |         | `+100MB`   |         |          |         |          |         |          |
| `Dirty`          |         |          |         |          |         |          |         | `+100MB`   |
| `Mapped`         |         |          | `+100MB`  |          | `+100MB`  | `+100MB`   | `+120MB`  | `+100MB`   |
| `Shmem`          |         |          |         |          | `+100MB`  | `+100MB`   |         |          |

```
*P = private
*S = shared
*A = anon
*F = file
```



> 注：
>
> 不同的kernel版本下可能有不同表现。比如在5.4的内核版本下，私有匿名的`mmap`写文件的可能直接进入`Active(anon)`
>
> 因此还是建议按下图（[`mm/workingset.c`](https://elixir.bootlin.com/linux/v6.3-rc7/source/mm/workingset.c)）来理解Inactive和Active的关系，而不是一些特定内核版本的表现
>
> ```C
> /*
>  *      Double CLOCK lists
>  *
>  * Per node, two clock lists are maintained for file pages: the
>  * inactive and the active list.  Freshly faulted pages start out at
>  * the head of the inactive list and page reclaim scans pages from the
>  * tail.  Pages that are accessed multiple times on the inactive list
>  * are promoted to the active list, to protect them from reclaim,
>  * whereas active pages are demoted to the inactive list when the
>  * active list grows too big.
>  *
>  *   fault ------------------------+
>  *                                 |
>  *              +--------------+   |            +-------------+
>  *   reclaim <- |   inactive   | <-+-- demotion |    active   | <--+
>  *              +--------------+                +-------------+    |
>  *                     |                                           |
>  *                     +-------------- promotion ------------------+
>  *
>  */
> ```
