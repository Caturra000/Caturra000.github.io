---
layout: post
title: 记录一些Linux Kernel下的「虚拟」文件系统的流程细节
categories: [kernel, RTFSC]
---

本来是想把整个Linux IO栈都大概整理一遍，限于工作繁忙，也只是把VFS往下一点的流程粗略翻了遍

下面会做一些简单的总结，由于说来话长，我不打算把每一处都说的特别详尽

毕竟（优质的）代码才是最好的文档

<!--more-->

![Erika](/img/Erika3.png)

## 源码和注释

原始的记录都在这：[RTFSC/linux/fs at master · Caturra000/RTFSC · GitHub](https://github.com/Caturra000/RTFSC/tree/master/linux/fs)

分类不一定很准确，像block mapping和elevator我也放到fs下了

## mount

在`mount`前，内核其实已经注册好对应的文件系统信息`file_system_type`

因此给定一个字符串字面值`fstype`也可以在全局的`file_systems`链表里遍历得到`file_system_type`

这个`file_system_type`关键用途就是`mount`回调（`type->mount`），通过这个回调可以得到`vfsmount`

这个`mount`回调一般会干这些事情：

- 通过`dev_name(/dev/xx)`得到`block_device`
- 通过`test`和`set`回调得到`super_block`，比较设备的参数来得知有没有对应的`super_block`，没有就构造：`super_block`的构造有具体文件系统的`fill_super`给出，并插入到全局`super_blocks`链表
- `super_block`实例已经有`(struct dentry*)s_root`，这是一个以后放入`vfsmount`的`dentry`

`vfsmount`可以认为是三元组`<dentry*, super_block*, flags>`：

- `dentry`是个庞大的话题，在这里可以相当于一个组件，可在以后用于寻找挂载后的`root`分量
- `super_block`是具体文件系统和虚拟文件系统的开端，可以通过它来构造出整个庞大的流程
- `flags`就是用于一些特殊的策略，这是多数系统调用都用的套路

得到的`vfsmount`就插入到全局目录树（namespace's mount tree）中，也就是说，挂载点维护于一棵树中

## open

`open`以一句话来说就是通过字符串求出打开文件实例，并返回给用户空间一个与打开文件实例相关联的`fd`

说着容易，但其实内部实现特别恶心，这里简单描述一下不带O_CREAT的open流程：

1. 构造`open_flags`，这些都是影响`open`流程的状态机参数
2. 从用户空间拷贝`filename`到内核空间，其内核实例就是`struct filename`
3. 分配`fd`
4. 初始化表示路径查找上下文的`struct nameidata`，这是存储多个分量查找/解析过程的临时类型
5. **启用rcu-walk**，先说疗效，就是得到打开文件`struct file`
6. `fsnotify`
7. `fd`和`file`关联

第5点是关键操作

由于要处理众多问题，比如：

- `open`时的`mode`和`flag`
- 跟踪符号链接
- `.`以及`..`和`//////////`
- 根本就没有目录
- `mount`路径跳转

等等麻烦问题，而且还要用上RCU操作，因此变得非常复杂

这里展开说一下第5点最为简略的过程

1. 获得空分配的file实例
2. 初始化查找路径，就是构造nd，更细点就是构造`<root, path, inode>`三元组，携带上前面提到的open_flags状态机
3. 循环处理分量，或者说名字解析，换到代码来看就是从一个`filename`实例（靠nd上下文的更新）转为最终分量对应的`dentry`
   1. 这个找`dentry`的过程是最重要的优化了，内核实现分2种方法：`fast lookup`和`slow lookup`
   2. `fast lookup`: `dentry`是放在`dcache`（大概这名字）一个大的缓存块里面，因此可以尝试hash（以最原始的name为key做hash）直接得到已经在cache里的`dentry`，找不到再看5.3.3
4. 打开文件的`vfs_open`执行，前面得到的信息想办法填充到打开文件`file`实例中，这里也会回调具体文件系统的`f_op->open`，干的事情也应该是填充file实例

## read

`read`其实是我VFS入门时第一个看的实现流程

`read`需要用到上述`open`给出的`fd`，在`open`阶段，内核为每一个`fd`都对应地构造一个内部使用的打开文件示例`struct file`

而`read`会进一步封装为`struct fd`，其实也没差，`fd.file`就是`struct file`

开始阅读前，先给出一些前置技能：

- `(struct file_operations*) f_op`：存放于`file`中的一个字段，由具体文件系统提供给VFS，`read()`流程用到它内部的`f_op->read_iter`函数指针执行读流程
- `(struct address_space*) f_mapping`：可以认为是`page cache`，IO的基本优化手段，内部数据结构是`radix tree` / `xarray`（视内核版本而定），用于维护`pages`
- `struct page`：`mm`模块下非常复杂的结构体，我们只需要知道`fs`模块下需要关注的点就可以了：比如前面提到的`page cache`，就是用来维护若干个`page`用于加速寻址，但是相邻的`page`之间也是可以通过链表来寻址完成，也就是说，如果是`page cache`的话会有两种数据结构来同时维护；而`page`本身就存放着`read()`所期望获取到的数据，具体以后细说

整体流程如下：

1. 获得参数`fd`对应的`(struct fd) f` / `(struct file) f.file`
2. 获得当前的`(loff_t) pos`，用于以后得到数据的字节大小后更新偏移
3. 调用具体文件系统注册的`f_op->read_iter`
   1. 获取之前的`read ahead`预读信息
   2. 往`page cache`查找`page` （包含read page或者get cached-page）
   3. cache this page
   4. page up-to-date?
   5. copy to user
   6. loop, goto step 3.1
4. 步骤3会得到整体的字节大小，依此更新`pos`

这里需要补充步骤2：

- read page认为是`page cache`里找不到`page`执行的过程，而get cached-page就是一个cache hit的过程
- 不管是read page还是get cached-page，都需要执行read ahead预读
- 如果被read ahead流程认为需要真的往外存去读，则会执行`readpage` / `readpages`（我后面再细说），否则，可以返回了

__PS. 关于预读可以参考我以往的文章（[浅谈Linux Kernel的预读算法](/posts/kernel-readahead)）__

`readpage` / `readpages`的差异在于你要读单个`page`还是多个`page`，接口来自于`(struct address_space_operations*) a_ops->readpage[s]`，至于为什么要写两个接口，那自然是kernel认为，`readpages`比`for`循环多次`readpage`更具有优化的潜能，比如合并一些操作

很显然它们来自于具体文件系统给出的回调

其中`readpages`的一个通用实现为`mpage_readpages()`

从`mpage_readpages()`开始，会逐步接触到`struct bio`和`struct buffer_head`，历史上它们和`page` / `page cache`是有各种微妙的关系

先给出两个结构体的整体印象：

- `bio`：IO的基本单位，一次IO就是一个`bio`（`submit_bio()`），它是允许scatter-IO的（通过`struct bio_vec`）
- `buffer_head`：用于提取block映射关系（通过`get_block()`）、跟踪页的状态（`map_bh()`）、封装`bio`的提交（`submit_bh()`，出于历史原因，最终也会走到`submit_bio()`）

而`readpages`流程如下：

1. 假定待读入的页面集合为`pages`，对于每一个`page`执行循环step 2
2. 把`page`插入到`f_mapping`和`page->lru`中（对于page cache虽然是插入了page，但其实只是填充了`page`自身关于`index`的信息）
3. 把`page`转为按`block`处理，整个`readpages`会尽可能贪心地只用一个`bio`来处理，一旦发生`confused`现象（如不可合并、`page`状态不符合预期等），就把当前循环流程提交IO上去，更换`bio`。其过程需要一个`buffer_head`通过`get_block`回调来获取实际的`b_blocknr`
4. 完成循环，如有剩余`bio`，再做出一次提交

提交`bh/bio`通过`generic_make_request()`转换为`request`，这里会进入到`elevator`层

`elevator`同样需要各种回调来描述电梯调度算法，比如`noop`，目的就是为了符合预期`IO`行为，对延迟和吞吐的一些需求做调整



// TODO 这里还没说到具体文件系统的`get_block`行为，以及`vfs inode`分配，`super_block`到`inode`和`bh->b_data`的关联等等，我打一会游戏再更新吧
