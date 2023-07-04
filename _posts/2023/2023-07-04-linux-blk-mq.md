---
layout: post
title: Linux Block IO层多队列机制（blk-mq）介绍
categories: [kernel]
description: 一些关于multiqueue的笔记
---

## 什么是blk-mq

```
The Multi-Queue Block IO Queueing Mechanism is an API to enable fast storage
devices to achieve a huge number of input/output operations per second (IOPS)
through queueing and submitting IO requests to block devices simultaneously,
benefiting from the parallelism offered by modern storage devices.
```

**TL;DR** blk-mq是block层的多队列IO框架。它是Linux内核实现高IOPS的财富密码，适用于多队列存储设备

## 框架概览

![overview](/img/blk-mq-overview.png)

`blk-mq`框架中有2种形式的队列：
- per-cpu级别的软件队列Software Staging Queue
    - 一般称为software queue、ctx(context)
    - 对应于数据结构`blk_mq_ctx`
- 对应于存储设备硬件队列的Hardware Dispatch Queue
    - 一般有hardware queue、hctx(hardware context)、hwq等奇怪命名
    - 对应于数据结构`blk_mq_hw_ctx`
    - 进入该队列的request意味着已经经过了调度

而每个存储设备有一个controlling structure，为`blk_mq_tag_set`，用于维护队列的关系：

| 字段       | 类型                                                         | 用途                                                     | 备注                                                         |
| ---------- | ------------------------------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------------ |
| `.mq_maps` | `int*`，实际作为一个`int[]`数组使用，长度为CPU个数           | 实现CPU到硬件队列的映射                                  | 下标为CPU编号，对应值为映射的硬件队列编号。比如`set->mq_map[cpu_j] = hw_queue_i`，其中`i`和`j`互不相干 |
| `.tags`    | `blk_mq_tags**`，实际作为一个`(blk_mq_tags*)[]`数组使用，长度为CPU个数 | 管理`request`分配，为每个hwq分配`set->tags[hw_queue_id]` |                                                              |


硬件队列关联了tag（从而间接关联到`request`），其结构体对应于`blk_mq_tags`：

| 字段          | 类型                                                         | 用途                                                         | 备注                                                         |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `.static_rqs` | `request**`，实际作为一个`(request*)[]`数组使用，长度为队列深度参数`set->queue_depth` | 从`buddy`中预先分配`set->queue_depth`个`request`实例，等待后续使用 | 该数组需要tag分配，由搭配的一个位图（`sbitmap`）来快速获得空闲的tag。每一次派发`request`前需要获取到tag与之绑定 |
| `.rqs`        | `request**`，实际作为一个`(request*)[]`数组使用，长度为队列深度参数`set->queue_depth` | 从`static_rqs[tag]`中获得的`request`实例在非电梯调度下会放入该数组中（同下标），表示in-flight request | <del>实际我也不知道是干嘛的，</del>一种可能的用法是给driver提供一个遍历所有使用中的request的迭代器。所有细节都在[这里](https://elixir.bootlin.com/linux/v4.18.20/A/ident/rqs) |

## 不太重要的细节

* `blk-mq`的硬件队列与驱动层的队列无关
* 虽然软件队列一般认为是per-cpu级别，但是maintainer也指出：如果在NUMA架构中，L3缓存足够大的话，软件队列可以设置为per-socket级别，这样也许能从cache友好和锁竞争中获取一个平衡点
* 硬件队列个数在不同的场合下是有歧义的，因为kernel里面会把超过CPU个数的硬件队列数目当作看不见（原因是超出部分没有意义），所以并不绝对等于硬件意义上的硬件队列个数
* tag虽然是给硬件队列使用，但是`blk_mq_tags`实际长度是按CPU个数给的
* tag对应的`request`数虽然是`set`提供的队列深度数，但是每次分配失败的话，会尝试把队列深度数目折半，这也会实际影响到`set->queue_depth`
* 预分配`request`的每个实例中其实还藏有driver层所需要的payload，详见[blk_mq_alloc_rqs](https://elixir.bootlin.com/linux/v4.18.20/source/block/blk-mq.c#L1964)

## 一些使用建议

Q. 我用的是比较传统的单队列设备，需要回归单队列框架吗？

不需要。一是`blk-mq`实测性能仍高于`sq`框架，二是内核版本5.0后已经把`sq`框架彻底删了

Q. 多队列框架下是否需要使用IO调度器？

如果是HDD，要。对于（足够高性能的）SSD的话，比方说你需要公平调度，或者做一些QoS也许用得着，但只看性能的话，这是一个开放的话题（[不服跑个分](https://mobile.zol.com.cn/760/7601256.html)吧）

Q. 如果确实需要选用IO调度器，该怎么选？

[redhat文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/monitoring_and_managing_system_status_and_performance/setting-the-disk-scheduler_monitoring-and-managing-system-status-and-performance)中给出了一些场景选择，为了节省你的IO，我把要点摘抄下来了：

| Use case                                                     | Disk scheduler                                               |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Traditional HDD with a SCSI interface                        | Use `mq-deadline` or `bfq`.                                  |
| High-performance SSD or a CPU-bound system with fast storage | Use `none`, especially when running enterprise applications. Alternatively, use `kyber`. |
| Desktop or interactive tasks                                 | Use `bfq`.                                                   |
| Virtual guest                                                | Use `mq-deadline`. With a host bus adapter (HBA) driver that is multi-queue capable, use `none`. |

## Work In Progress!

剩余章节仍在施工中

TODO 架构上的说明，具体的实现流程

## References

[Multi-Queue Block IO Queueing Mechanism (blk-mq)](https://www.kernel.org/doc/html/latest/_sources/block/blk-mq.rst.txt)

[Linux Block IO: Introducing Multi-queue SSD Access on Multi-core Systems](https://kernel.dk/systor13-final18.pdf)

[Chapter 11. Setting the disk scheduler Red Hat Enterprise Linux 9 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/monitoring_and_managing_system_status_and_performance/setting-the-disk-scheduler_monitoring-and-managing-system-status-and-performance)
