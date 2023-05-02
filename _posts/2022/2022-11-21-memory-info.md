---
layout: post
title: 「氵」关于各种各样的memory
categories: [Memory, kernel]
description: 诈尸氵一篇meminfo
---

（诈尸氵一篇meminfo）

文章还处于work in progress状态，先直接从OneNote上复制过来吧

<!--more-->

## 示例

 
meminfo的分类有：

```
	MemTotal:       16642364 kB
	MemFree:         9337232 kB
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
	SwapFree:       26120500 kB
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

 

## 公式换算

 

meminfo可以看mm模块的代码，虽然类型很多，但计算方式还算透明

 

| 内存类型               | 简化换算                                                     | 其它备注                                                     |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `MemTotal`             | physical - .text                                             | total并不是全部，需要抠掉一些kernel代码段的大小              |
| `MemFree`              | lowFree  + HighFree                                          | HighFree基本就是0的意思                                      |
| `MemAvailable`         |                                                              | 一个简单的估算方式是  所有zone的free  减去 low reserved 和 high watermark  加上 可回收的page cache 和 slab  这里的可回收是个估算值，可能直接取总数的1/2 |
| `Buffers`              | for each bdev:    ret +=  bdev->bd_inode->i_mapping->nr_pages | 裸设备读写的临时存储  如果是文件系统操作则应对应于metadata，如inode，superblock  否则要算上直接读写bdev的mapping（本质就是bdev） |
| `Cached`               | page  cache, in-memory files only                            | 这里只算in-memory引起的  需要减去SwapCached和Buffers     可以认为代表文件映射的内存和tmpfs（shmem）映射（因为也是file-backed）的内存 |
| `SwapCached`           |                                                              | 曾经换出，然后换入回来的内存，但是仍存在于swap文件  这是用于节省IO的，当已经换回来后，如果再次缺少内存，那就直接舍去这一部分，而不是换出回swap文件，从而节省IO |
| `Active`               |                                                              | LRU中标记active，一般不reclaim，但也有可能                   |
| `Inactive`             |                                                              | LRU中标记inactive，很可能reclaim                             |
| `HighTotal,  HighFree` |                                                              | ~                                                            |
| `LowTotal,  LowFree`   |                                                              | ~                                                            |
| `SwapTotal`            |                                                              | ~                                                            |
| `SwapFree`             |                                                              | 已经被换出的内存，临时存储在disk                             |
| `Dirty`                |                                                              | 标记为dirty的内存，仍未写回                                  |
| `Writeback`            |                                                              | 正在写回的内存                                               |
| `AnonPages`            |                                                              | 映射到userspace页表的无后备文件的内存                        |
| `HardwareCorrupted`    |                                                              | ~                                                            |
| `AnonHugePages`        |                                                              | ~                                                            |
| `Mapped`               |                                                              | 已经被mapped映射的文件                                       |
| `Shmem`                | shmem + tmpfs                                                |                                                              |
| `ShmemHugePages`       |                                                              |                                                              |
| `ShmemPmdMapped`       |                                                              |                                                              |
| `KReclaimable`         | SReclaimable + ...direct allocations                         | kernel分配出来的，但可被reclaim的内存                        |
| `Slab`                 |                                                              | ~                                                            |
| `SReclaimable`         | part of  Slab                                                | Slab的一部分，允许reclaim                                    |
| `SUnreclaim`           | part of  Slab                                                | Slab的一部分，不允许reclaim                                  |
| `PageTables`           |                                                              | ~                                                            |
| `VmallocTotal`         | vmalloc()                                                    |                                                              |
| `VmallocUsed`          | used  vmalloc()                                              |                                                              |
| `VmallocChunk`         | largest  contiguous and free vmalloc()                       |                                                              |

 

meminfo在具体的实现上，仍然是获取来自vmstat的全局变量信息

其实vmstat也是从kernel提供的API去更新值的，涉及到整个物理内存zone管理的细节

 

注：关于pagecache

在[统计代码实现](https://github.com/Caturra000/RTFSC/blob/7836cbb7f5224e1fed6f6ac45e02e2d3ee358f96/linux/mm/meminfo/MemAvailable统计.c#L36)上，pagecache相当于active file LRU加上inactive file LRU

我觉得应该不只是这些，可能是个大概的值

而在[接口](https://github.com/Caturra000/RTFSC/blob/7836cbb7f5224e1fed6f6ac45e02e2d3ee358f96/linux/mm/meminfo/统计接口.c#L23)层面，Cached成分相对复杂，是NR_FILE_PAGES - SwapCached - Buffers

这个NR_FILE_PAGES就是所有的mapping之和了，就是不确定是否等于上述的pagecache（两个LRU加起来）

有一种观点是Buffers + Cached = inactive(file) + active(file) + shmem

Shmem是算到Cached里面的

Buffers算是bdev files mapping特例排除掉

因此一个完整的pagecache = inactive(file) + active(file) + shmem，而上面available统计实现是为了快速估算没算上吧
