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

**TL;DR** blk-mq是Linux内核实现高IOPS的财富密码，适用于多队列存储设备的block层IO框架

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

> 注：这里的硬件队列与驱动层的队列无关

> 注：maintainer指出，如果在NUMA架构中，L3缓存足够大的话，软件队列可以设置为per-socket级别，这样也许能从cache友好和锁竞争中获取一个平衡点

而每个存储设备有一个controlling structure，为`blk_mq_tag_set`，用于维护队列的关系：
- `.mq_maps`字段
    - 类型：`int*`，实际作为一个`int[]`数组使用，长度为CPU个数
    - 用途：实现CPU到硬件队列的映射
    - key：下标通过CPU编号访问
    - value：映射为硬件队列编号
    - 例子：`set->mq_map[cpu_j] = hw_queue_i`，其中`i`和`j`互不相干
- `.tags`字段
    - 类型：`blk_mq_tags**`，实际作为一个`(blk_mq_tags*)[]`数组使用，长度为CPU个数
    - 用途：管理`request`分配，为每个hwq分配`set->tags[hw_queue_id]`

硬件队列关联了tag，实则简介关联到`request`，其结构体对应于`blk_mq_tags`：
- `.static_rqs`字段
    - 类型：`request**`，实际作为一个`(request*)[]`数组使用，长度为队列深度参数`set->queue_depth`
    - 用途：从`buddy`中预先分配`set->queue_depth`个`request`实例，等待后续使用
    - 备注：该数组需要tag分配，由搭配的一个位图（`sbitmap`）来快速获得空闲的tag
- `.rqs`字段
    - 类型：`request**`，实际作为一个`(request*)[]`数组使用，长度为队列深度参数`set->queue_depth`
    - 用途：从`static_rqs[tag]`中获得的`request`实例在非电梯调度下会放入该数组中（同下标），表示in-flight request
    - 备注：<del>实际我也不知道是干嘛的，</del>一种可能的用法是给driver提供一个遍历所有使用中的request的迭代器。所有细节都在[这里](https://elixir.bootlin.com/linux/v4.18.20/A/ident/rqs)，

## 不太重要的细节

* 硬件队列个数在不同的场合下是有歧义的，因为kernel里面会把超过CPU个数的硬件队列数目当作看不见（原因是超出部分没有意义），所以并不绝对等于硬件意义上的硬件队列个数
* tag虽然是给硬件队列用的，但是`blk_mq_tags`实际长度是按CPU个数给的，虽然理解上我觉得没啥问题
* tag对应的`request`数虽然是`set`提供的队列深度数，但是每次分配失败的话，会尝试把深度数目折半，这也会实际影响到`set->queue_depth`

## Work In Progress!

剩余章节仍在施工中

## References

[Multi-Queue Block IO Queueing Mechanism (blk-mq)](https://www.kernel.org/doc/html/latest/_sources/block/blk-mq.rst.txt)

[Linux Block IO: Introducing Multi-queue SSD Access on Multi-core Systems](https://kernel.dk/systor13-final18.pdf)
