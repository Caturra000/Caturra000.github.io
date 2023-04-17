---
layout: post
title: 浅谈Linux Kernel的预读算法
categories: [kernel, RTFSC]
---

性能优化的关键在于解决性能的瓶颈，而IO从来都是难以解决的瓶颈之一

这篇文章主要描述Linux Kernel对于读操作下的按需预读算法，包括流程和实现
<!--more-->



## 基本的IO优化策略

在开始说预取算法前，还是要看一下常规的IO优化策略

1. 避免IO，常见的缓存策略其实就是避免（外存）IO，不涉及IO就没有IO问题
2. 顺序IO，显然，顺序访问比随机访问要快，如果多IO负载下无法完全顺序执行，那就增大IO的大小，减少寻道定位，提高吞吐量
3. 异步IO，CPU和磁盘同时工作，把所需的数据提前载入内存
4. 并行IO，通过RAID实现并行IO

 

然而，对于一个常规的应用来说，其IO行为是小且串行的：既无法异步，也无法并行，更无法提高吞吐量

针对这些行为，我们需要对数据进行预读取和延迟写（他们的关系并非正交，延迟写如write back等方法我们不独立讨论）

 

至于为什么需要，一个简单的原因就是通过预读取来满足上述1、2、3的优化方法

 

## 预读取的两种类型

预读取的方式可以分为

- 通知式预读
- 启发式预读

通知式预读是知情的，需要用户配合，如`posix_fadvise`，不展开讨论

而启发式是完全用户透明的，其难点在于算法要求较高，也是后面讲介绍的具体内容

 

## 两种启发式的预读取策略

除了direct read，Linux对于读接口会进行一定的预读策略，如

- 以`mmap`为代表的`read-around`
- 以`read`为代表的`read-ahead`

它们的策略区别在于更加侧重于随机读还是顺序读（或者是通用的、任意类型的读操作）

可以以简单的形式来描述`mmap`过程中的`read-around`：

1. `mmap`仅提供`vma`和对应映射关系返回给用户态
2. 当用户态触发对应的page fault时，查看是否有缓存，如果没有，以当前页为中心预取一定的页

而`read-ahead`则更为复杂得多，也更加难以实现

 

## read ahead的复杂性

`read ahead`相比`read around`更加关注顺序读的性能，但也需要考虑多种IO负载下的情况，因为它位于`VFS`下的`generic read`层，`read/pread/readv/sendfile`等接口都会统一到`generic read`，要面临的情况也更为复杂，如

- 顺序访问更为复杂，可能有非对齐读、重试读、交织读等行为
- 顺序访问可能不需要预取，可能有预读缓存命中和IO队列阻塞的情况
- 预取后的页面可能会在访问前被回收，也就是预读抖动现象

要实现`read ahead`，就是要处理上述的复杂情况

 

## on demand read ahead

目前kernel在通用块层提供on demand read ahead的算法框架，按字面意思，就是【按需预读】

我觉得大概分三个模块来描述整体框架比较合适

1. 需要的数据结构
2. 算法的入口
3. 算法的实现

（注：下述算法描述是基于Linux 4.18.20实现）

## read ahead状态

`read ahead`需要的数据结构位于`/include/linux/fs.h`

```C
/*
 * Track a single file's readahead state
 */
struct file_ra_state {
	pgoff_t start;			/* where readahead started */
	unsigned int size;		/* # of readahead pages */
	unsigned int async_size;	/* do asynchronous readahead when
					   there are only # of pages ahead */

	unsigned int ra_pages;		/* Maximum readahead window */
	unsigned int mmap_miss;		/* Cache miss stat for mmap accesses */
	loff_t prev_pos;		/* Cache last read() position */
};
```

需要关注的成员为`start` / `size` / `async_size`，它们通过$$O(1)$$的空间来维护`ahead`窗口

以作者的图来描述就是

```C
/*
 * The fields in struct file_ra_state represent the most-recently-executed
 * readahead attempt:
 *
 *                        |<----- async_size ---------|
 *     |------------------- size -------------------->|
 *     |==================#===========================|
 *     ^start             ^page marked with PG_readahead
 */
```

- `start`和`size`二元组构成预读窗口，记录最近一次预读请求的位置`start`和连续大小`size`，
- `async_size`为预读提前量，表示还剩余`async_size`个未访问页时启动下一次预读

后面会讲述通过`ahead`窗口来实现一个IO流水线的操作



## 通用层入口

以`read`/`ext4`作为跟踪，经过

```C
// read -> ksys_read -> vfs_read -> __vfs_read -> file_operations->read_iter
// -> new_sync_read -> call_read_iter -> generic_file_read_iter
// -> generic_file_buffered_read

static ssize_t generic_file_buffered_read(struct kiocb *iocb,
		struct iov_iter *iter, ssize_t written)
{
	struct file *filp = iocb->ki_filp;
	struct address_space *mapping = filp->f_mapping;
	struct inode *inode = mapping->host;
	struct file_ra_state *ra = &filp->f_ra;
	loff_t *ppos = &iocb->ki_pos;
	pgoff_t index;
	pgoff_t last_index;
	pgoff_t prev_index;
	unsigned long offset;      /* offset into pagecache page */
	unsigned int prev_offset;
	int error = 0;

	if (unlikely(*ppos >= inode->i_sb->s_maxbytes))
		return 0;
	iov_iter_truncate(iter, inode->i_sb->s_maxbytes);

	index = *ppos >> PAGE_SHIFT;
	prev_index = ra->prev_pos >> PAGE_SHIFT;
	prev_offset = ra->prev_pos & (PAGE_SIZE-1);
	last_index = (*ppos + iter->count + PAGE_SIZE-1) >> PAGE_SHIFT;
	offset = *ppos & ~PAGE_MASK;

	for (;;) {
		struct page *page;
		pgoff_t end_index;
		loff_t isize;
		unsigned long nr, ret;

		cond_resched();
find_page:
		if (fatal_signal_pending(current)) {
			error = -EINTR;
			goto out;
		}

		page = find_get_page(mapping, index);
		if (!page) {
			if (iocb->ki_flags & IOCB_NOWAIT)
				goto would_block;
			page_cache_sync_readahead(mapping, // ⭐
					ra, filp,
					index, last_index - index);
			page = find_get_page(mapping, index);
			if (unlikely(page == NULL))
				goto no_cached_page;
		}
		if (PageReadahead(page)) {
			page_cache_async_readahead(mapping, // ⭐
					ra, filp, page,
					index, last_index - index);
		}
        // ...
```

其中`page_cache_sync_readahead`和`page_cache_async_readahead`则为readahead入口

可以看出，进入预读的条件有两种

- radix tree中没有找到页缓存
- 找到页缓存且该页面标记上`PG_Readahead`



## 简略流程

我尝试以简单的方式描述一次完全顺序执行读请求的预读过程

记`| =`为单个页，`#`为被标记为`PG_readahead`的页，`^`为`ra->start`，`~`为`offset`当前访问到的值

当对一个没有访问过的文件/页进行顺序读时，且从`offset = 0, request_size = 1`（页为单位）开始，会触发**同步预读**

```
|
~
```

由于没有访问过，`file_ra_state`（简记为`ra`）中的四元组信息为

```
.start = 0
.size = 0
.async_size = 0
.prev_pos = -1
```

预读流程会根据`prev_pos`构造出预读窗口，求出`size`和`async_size`，并且在`offset = #`打上`PG_readahead`标记

由于页缓存并不存在，因此对应进程会挂起从而等待IO

```
|===#======|
^
~
```

其中`#`直到右边`|`为`async_size`范围，共读入`size`大小的页

假设下一次顺序读的起始页（`offset`）在`^`后和`#`前，则不触发预读

```
|===#======|
^  ~
```

继续顺序读至`offset = #`，则会发出下一次的**异步预读**

```
|==========|#======================|
    ~       ^
```

由于这个过程中`~`是处于已预取状态的页，因此这次预读是完全CPU与IO异步的

可以看出，预读窗口是这一次的预读请求，也是下一次预读时机的判断依据

而窗口的增长通常是倍增的形式，往后会按照**没有页缓存**或者**触发标记页**来继续执行上述预读



## 整体流程

1. 页缓存缺失，走3
2. 或者遍历到`PG_readahead`页，如果条件不允许，退出，否则走3
3. 尝试处理初始的顺序读或者首部读，成功则走9
4. 尝试处理连续的顺序读，成功则走9
5. 尝试处理交织的顺序读，成功则走9
6. 尝试处理过大读，成功走9
7. 尝试处理历史缓存上下文，成功则走9
8. 确认随机读，提交IO，退出
9. 尝试为可能的同步预读或上下文合并请求
10. 标记某一页为`PG_readahead`，提交IO

可以看出，`read ahead`非常注重顺序读问题（处理步骤最多，也是最高的判断优先级）但同时也考虑通用读的情况

代码集中在单文件`/mm/readahead.c`，但仍然太大了不方便贴出来，只展示核心部分

```C
/*
 * A minimal readahead algorithm for trivial sequential/random reads.
 */
static unsigned long
ondemand_readahead(struct address_space *mapping,
		   struct file_ra_state *ra, struct file *filp,
		   bool hit_readahead_marker, pgoff_t offset,
		   unsigned long req_size)
{
	struct backing_dev_info *bdi = inode_to_bdi(mapping->host);
	unsigned long max_pages = ra->ra_pages;
	unsigned long add_pages;
	pgoff_t prev_offset;

	/*
	 * If the request exceeds the readahead window, allow the read to
	 * be up to the optimal hardware IO size
	 */
	if (req_size > max_pages && bdi->io_pages > max_pages)
		max_pages = min(req_size, bdi->io_pages);

	/*
	 * start of file
	 */
	if (!offset)
		goto initial_readahead;

	/*
	 * It's the expected callback offset, assume sequential access.
	 * Ramp up sizes, and push forward the readahead window.
	 */
	if ((offset == (ra->start + ra->size - ra->async_size) ||
	     offset == (ra->start + ra->size))) {
		ra->start += ra->size;
		ra->size = get_next_ra_size(ra, max_pages);
		ra->async_size = ra->size; // ##flag #0
		goto readit;
	}

	/*
	 * Hit a marked page without valid readahead state.
	 * E.g. interleaved reads.
	 * Query the pagecache for async_size, which normally equals to
	 * readahead size. Ramp it up and use it as the new readahead size.
	 */
	if (hit_readahead_marker) {
		pgoff_t start;

		rcu_read_lock();
		start = page_cache_next_hole(mapping, offset + 1, max_pages);
		rcu_read_unlock();

		if (!start || start - offset > max_pages)
			return 0;

		ra->start = start;
		ra->size = start - offset;	/* old async_size */
		ra->size += req_size;
		ra->size = get_next_ra_size(ra, max_pages);
		ra->async_size = ra->size; // ##flag #0
		goto readit;
	}

	/*
	 * oversize read
	 */
	if (req_size > max_pages)
		goto initial_readahead;

	/*
	 * sequential cache miss
	 * trivial case: (offset - prev_offset) == 1
	 * unaligned reads: (offset - prev_offset) == 0
	 */
	prev_offset = (unsigned long long)ra->prev_pos >> PAGE_SHIFT;
	if (offset - prev_offset <= 1UL)
		goto initial_readahead; // ##flag #3

	/*
	 * Query the page cache and look for the traces(cached history pages)
	 * that a sequential stream would leave behind.
	 */
	if (try_context_readahead(mapping, ra, offset, req_size, max_pages))
		goto readit;

	/*
	 * standalone, small random read
	 * Read as is, and do not pollute the readahead state.
	 */
	return __do_page_cache_readahead(mapping, filp, offset, req_size, 0);

initial_readahead:
	ra->start = offset;
	ra->size = get_init_ra_size(req_size, max_pages);
	ra->async_size = ra->size > req_size ? ra->size - req_size : ra->size; // ##flag #1

readit:
	/*
	 * Will this read hit the readahead marker made by itself?
	 * If so, trigger the readahead marker hit now, and merge
	 * the resulted next readahead window into the current one.
	 * Take care of maximum IO pages as above.
	 */
	if (offset == ra->start && ra->size == ra->async_size) {
		add_pages = get_next_ra_size(ra, max_pages);
		if (ra->size + add_pages <= max_pages) {
			ra->async_size = add_pages;
			ra->size += add_pages;
		} else {
			ra->size = max_pages;
			ra->async_size = max_pages >> 1;
		}
	}

	return ra_submit(ra, mapping, filp);
}
```



## 实现细节

虽然总体框架看着简单（得益于算法作者的精妙设计），但仍然有很多细节值得琢磨

1. 怎么启发式的确认框架中所谓的顺序读、交织读、随机读等行为
2. 在复杂性章节提到的不必要的预读和预读抖动又是怎么处理的
3. 预读窗口该如何更新，取值怎么取？

其中问题3将直接影响问题1、2的解决策略



## 初始的窗口和迭代的窗口

先来看问题3，窗口的更新策略

两个函数分别对应于`get_init_ra_size` / `get_next_ra_size`，它们决定`ra->size`的大小

```C
/*
 * Set the initial window size, round to next power of 2 and square
 * for small size, x 4 for medium, and x 2 for large
 * for 128k (32 page) max ra
 * 1-8 page = 32k initial, > 8 page = 128k initial
 */
static unsigned long get_init_ra_size(unsigned long size, unsigned long max)
{
	unsigned long newsize = roundup_pow_of_two(size);

	if (newsize <= max / 32)
		newsize = newsize * 4;
	else if (newsize <= max / 4)
		newsize = newsize * 2;
	else
		newsize = max;

	return newsize;
}

/*
 *  Get the previous window size, ramp it up, and
 *  return it as the new window size.
 */
static unsigned long get_next_ra_size(struct file_ra_state *ra,
						unsigned long max)
{
	unsigned long cur = ra->size;
	unsigned long newsize;

	if (cur < max / 16)
		newsize = 4 * cur;
	else
		newsize = 2 * cur;

	return min(newsize, max);
}
```

- 对于`size`：总的来说采用倍增策略（2倍或4倍），如果当前值越大则越保守，并且大小不会超过`max`（文件系统或设备的最大允许IO大小）

- 对于`async_size`：通常贪心地尝试最大的异步可能性，既`async_size = size`，见`ondemand_readahead`流程中的`##flag #0`和`##flag #1`处

其设计目的是为了保证程序在任何时刻终止顺序访问都能得到可观的命中率



## 预读缓存命中

预读缓存命中是不必要的缓存命中行为，它的问题会减少IO的吞吐量

```C
// ra_submit -> __do_page_cache_readahead

/*
 * __do_page_cache_readahead() actually reads a chunk of disk.  It allocates
 * the pages first, then submits them for I/O. This avoids the very bad
 * behaviour which would occur if page allocations are causing VM writeback.
 * We really don't want to intermingle reads and writes like that.
 *
 * Returns the number of pages requested, or the maximum amount of I/O allowed.
 */
unsigned int __do_page_cache_readahead(struct address_space *mapping,
		struct file *filp, pgoff_t offset, unsigned long nr_to_read,
		unsigned long lookahead_size)
{
	struct inode *inode = mapping->host;
	struct page *page;
	unsigned long end_index;	/* The last page we want to read */
	LIST_HEAD(page_pool);
	int page_idx;
	unsigned int nr_pages = 0;
	loff_t isize = i_size_read(inode);
	gfp_t gfp_mask = readahead_gfp_mask(mapping);

	if (isize == 0)
		goto out;

	end_index = ((isize - 1) >> PAGE_SHIFT);

	/*
	 * Preallocate as many pages as we will need.
	 */
	for (page_idx = 0; page_idx < nr_to_read; page_idx++) {
		pgoff_t page_offset = offset + page_idx;

		if (page_offset > end_index)
			break;

		rcu_read_lock();
		page = radix_tree_lookup(&mapping->i_pages, page_offset);
		rcu_read_unlock();
		if (page && !radix_tree_exceptional_entry(page)) {
			/*
			 * Page already present?  Kick off the current batch of
			 * contiguous pages before continuing with the next
			 * batch.
			 */
			if (nr_pages)
				read_pages(mapping, filp, &page_pool, nr_pages, // ⭐
						gfp_mask);
			nr_pages = 0;
			continue;
		}

		page = __page_cache_alloc(gfp_mask);
		if (!page)
			break;
		page->index = page_offset;
		list_add(&page->lru, &page_pool);
		if (page_idx == nr_to_read - lookahead_size)
			SetPageReadahead(page); // ##flag #2
		nr_pages++;
	}

	/*
	 * Now start the IO.  We ignore I/O errors - if the page is not
	 * uptodate then the caller will launch readpage again, and
	 * will then handle the error.
	 */
	if (nr_pages)
		read_pages(mapping, filp, &page_pool, nr_pages, gfp_mask);
	BUG_ON(!list_empty(&page_pool));
out:
	return nr_pages;
}
```

在提交IO的处理流程中，`read_pages`（既`mapping->a_ops->readpages`）是需要连续的非缓存页，如果`[page_idx, page_idx + nr_to_read)`的范围内存在页缓存，则需要执行多次分离的`readpages`

如果需要解决，那就要观察缓存页面的规律：多数的已缓存页面是连续相邻的

因此我们要检测出该缓存区域，并且禁止内部启动预读行为

按需算法对于预读缓存命中的解决策略有：

1. 限制标志设置条件：只对新的预读页面标记`PG_readahead`，见`##flag #2`
2. 避免频繁执行预读例程：只有在页面缺失或者触发`PG_readahead`才执行预读

（注：对于第一点，为了避免只因少量缓存存在导致的错误的拒绝打标记行为，按需算法可以更严格的执行只有全部页面都触发预读缓存命中才拒绝设置`PG_readahead`，但是算法中并没有用到这一严格策略）



## IO队列阻塞

当同步预读发起时，应用程序是必须要挂起的，因为确实需要触发IO，CPU需要等待IO的结果

但是异步预读是尝试CPU与IO异步执行，CPU不需要等待IO的结果

因此，如果异步预读时当前的IO队列是阻塞的，那就直接结束异步流程

```C
void
page_cache_async_readahead(struct address_space *mapping,
			   struct file_ra_state *ra, struct file *filp,
			   struct page *page, pgoff_t offset,
			   unsigned long req_size)
{
	/* no read-ahead */
	if (!ra->ra_pages)
		return;

	/*
	 * Same bit is used for PG_readahead and PG_reclaim.
	 */
	if (PageWriteback(page))
		return;

	ClearPageReadahead(page);

	/*
	 * Defer asynchronous read-ahead on IO congestion.
	 */
	if (inode_read_congested(mapping->host)) // ⭐
		return;

	/* do read-ahead */
	ondemand_readahead(mapping, ra, filp, true, offset, req_size);
}
```



## 预读抖动

预读抖动是出于页面置换/回收机制的冲突。按需预读算法认为，预读抖动大概率发生在大规模并发的顺序访问负载中

因此它的检测机制判断如下

- 正在顺序访问
- 访问的页不在缓存中

当检测确认后，其预读抖动保护机制如下

- 在当前位置重新启动预读，而非单页面小IO
- 依据当前读请求，重设倍增窗口大小

它的实现与非对齐行为合并在初始窗口流程，见`##flag #3`



## 更为复杂的顺序读

在常见的应用场合中，顺序读也不是简单的线性增长

- 在non-blocking IO中，它的行为可以称为重试读，其若干的读请求区间$$[offset_{start}, offset_{end})$$中，$$offset_{end}$$总是固定的，而$$offset_{start}$$可能是不变的或者少量递增的，如$$[0, 1000), [2, 1000), [6, 1000) \dots$$
- 在并发处理流程中，同一个文件的IO可能被多个并发流处理，这种行为称为交织读，其若干的读请求区间经过合并确实是顺序读，但在文件系统的角度可能是$$[0, 2), [100, 103), [2, 5), [103, 105) \dots$$

按需预读算法的解决策略十分简单：

- 它的触发预读条件是按页（实际访问页）而非按请求触发
- 对于重试读，依然可以处理为线性的顺序读
- 对于交织读，预读触发条件的限制会避免认为是随机读而被强制打断，如因误触发`PG_readahead`而进入预读例程，则往前判断一个页面缓存是否存在，若存在则证明这其实是一个空间上连续访问的流（时间上是交织的行为），不做重复预读处理，不存在则重新预读

（注：交织读的处理方式其实就是为`PG_readahead`的标记方法寻找弥补方案）



## 总结

基本上，我大概梳理了预取算法中的核心流程，以及各种corner case下所设计的解决方案，

预读算法的核心问题就是：如何为复杂场合设计一套简洁、高效而可扩展的算法

它的简洁和可扩展性在于整个算法流程是按流水线的形式来设计的，可读性极高（你可以直接看到任何处理方式的优先级），如果针对性地需要定制更为合适的解决方案，往流水线里插入额外的算法即可

另一个层面是算法上的优化，首先空间复杂度是仅有的`O(1)`，维护的变量少也意味着更加明确的状态转移，流水线参数其实就是`async_size`；其次是时间复杂度上是尽可能的被动处理（有意思的是明明称为预读却从来都不积极），也通过简单的标记做法把调用预读例程的可能性降低，甚至微妙地避免了错误的状态转移，我觉得在工程意义上也有很大的借鉴价值



## 更多

预读算法仍然有不少小细节可以挖掘，比如过大读的处理，或者根据历史缓存上下文找出潜在的顺序读等等，

感兴趣的可以看算法作者吴峰光的论文《Linux内核中的预取算法》，

以及一些更加简洁的`commit message`，顺手po几个供参考

- https://github.com/torvalds/linux/commit/122a21d11cbfda6d1e33cbc8ae9e4c4ee2f1886e
- https://github.com/torvalds/linux/commit/10be0b372cac50e2e7a477852f98bf069a97a3fa
- https://github.com/torvalds/linux/commit/2cad401801978b16ac6e43f10b8d60039670fcbc

## 差点忘了

按照惯例，源码解析的文章要补上某侦探的图

![Erika](/img/Erika2.jpg)
