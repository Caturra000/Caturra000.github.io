---
layout: post
title: 浅谈x86特定体系下的并发
categories: [Concurrency, arch]
---

<!--more-->

前面的<del>文章</del>提到一些语言抽象层面级别的并发接口（确实很抽象）

正如上文所说，语言层面就不应该看实现

但是特定体系下，就是需要看实现是怎么干的

如果要涉及到硬件实现，可能会脱口而出OOO乱序执行、store buffer、缓存一致性协议等关键字

不妨先理顺一下它们的关系

## Q1. 缓存一致性协议影响并发吗？

TL;DR 只论MESI协议是没有影响的，因为该协议是强一致性的

当你在考虑并发时想到这个东西，那你的方向错了

## Q2. store buffer和缓存一致性的关系？

TL;DR 前面提到，MESI缓存一致性协议是强一致性的，cache与cache间，cache与memory间的交流没有歧义

而增加的第三者store buffer则在此基础上实现弱一致性，从这里开始，并发的正确性会受到影响

PS. 多插一句，paper提到仅使用store buffer是错误设计，需要引入store forwarding，不过这是硬件设计的事情，别介意

## Q3. OOO?不是全都乱序了吗？

我确实没有找到什么的资料能说明乱序执行到底乱序到什么程度，如何去实现，是否只与store buffer相关，与顺序提交有何关系…这对于一个软件开发者来说是不是偏题了点？

其实对于我们开发者来说最关心的问题不过是：什么时候可以只用编译器屏障就足够了？

硬件的复杂性阻拦了我们也没关系，SDM手册告诉我们：哪怕是特定体系，还是要高屋建瓴地看待问题——请看memory order，它是直接告知CPU对内存的影响，既从内存的角度看CPU store load指令的顺序

## x86下给定的内存模型

该内存模型据说简称为TSO(total store order)，不过名字无所谓，我们着重看承诺的效果

```
In a single-processor system for memory regions defined as write-back cacheable, the memory-ordering model respects the following principles (Note the memory-ordering principles for single-processor and multipleprocessor systems are written from the perspective of software executing on the processor, where the term “processor” refers to a logical processor. For example, a physical processor supporting multiple cores and/or Intel Hyper-Threading Technology is treated as a multi-processor systems.):
- Reads are not reordered with other reads.
- Writes are not reordered with older reads.
- Writes to memory are not reordered with other writes, with the following exceptions:
  - streaming stores (writes) executed with the non-temporal move instructions (MOVNTI, MOVNTQ, MOVNTDQ, MOVNTPS, and MOVNTPD); and
  - string operations (see Section 8.2.4.1).
- No write to memory may be reordered with an execution of the CLFLUSH instruction; a write may be reordered with an execution of the CLFLUSHOPT instruction that flushes a cache line other than the one being written. Executions of the CLFLUSH instruction are not reordered with each other. Executions of CLFLUSHOPT that access different cache lines may be reordered with each other. An execution of CLFLUSHOPT may be reordered with an execution of CLFLUSH that accesses a different cache line.
- Reads may be reordered with older writes to different locations but not with older writes to the same location.
- Reads or writes cannot be reordered with I/O instructions, locked instructions, or serializing instructions.
- Reads cannot pass earlier LFENCE and MFENCE instructions.
- Writes and executions of CLFLUSH and CLFLUSHOPT cannot pass earlier LFENCE, SFENCE, and MFENCE instructions.
- LFENCE instructions cannot pass earlier reads.
- SFENCE instructions cannot pass earlier writes or executions of CLFLUSH and CLFLUSHOPT.
- MFENCE instructions cannot pass earlier reads, writes, or executions of CLFLUSH and CLFLUSHOPT.

In a multiple-processor system, the following ordering principles apply:
- Individual processors use the same ordering principles as in a single-processor system.
- Writes by a single processor are observed in the same order by all processors.
- Writes from an individual processor are NOT ordered with respect to the writes from other processors.
- Memory ordering obeys causality (memory ordering respects transitive visibility).
- Any two stores are seen in a consistent order by processors other than those performing the stores
- Locked instructions have a total order.
```


除去特定指令，只看平凡的操作，会发现并不是任何读写都会reorder：
- load与load之间不会
- store与store之间不会
- 早期的load与store不会
- 单个CPU的写仍然保证偏序

## 原子性的提供

x86下普通的读写指令就满足原子性的要求

也就是说`relaxed`下其实和普通的变量读写没啥差异（仍需要拒绝编译器优化）

注：RMW操作则是用`LOCK`前缀表示，特殊的`XCHG`不需要显式声明`LOCK`

```
The Intel486 processor (and newer processors since) guarantees that the following basic memory operations will always be carried out atomically:
- Reading or writing a byte
- Reading or writing a word aligned on a 16-bit boundary
- Reading or writing a doubleword aligned on a 32-bit boundary

The Pentium processor (and newer processors since) guarantees that the following additional memory operations will always be carried out atomically:
- Reading or writing a quadword aligned on a 64-bit boundary
- 16-bit accesses to uncached memory locations that fit within a 32-bit data bus

The P6 family processors (and newer processors since) guarantee that the following additional memory operation will always be carried out atomically:
- Unaligned 16-, 32-, and 64-bit accesses to cached memory that fit within a cache line
```



## volatile的特殊使用

如果在语言抽象的层面来看，`volatile`并不适合作为并发正确性的保证

但是在前面明确了特定体系下内存模型的基础上，当不需要提供内存屏障时，我们可以使用`volatile`

事实上，`kernel`也有大量使用，只是通过临时转换的形式隐藏了起来

以`READ_ONCE`为例
```C

#define READ_ONCE(x) __READ_ONCE(x, 1)
#define __READ_ONCE(x, check)                       \
({                                  \
    union { typeof(x) __val; char __c[1]; } __u;            \
    if (check)                          \
        __read_once_size(&(x), __u.__c, sizeof(x));     \
    else                                \
        __read_once_size_nocheck(&(x), __u.__c, sizeof(x)); \
    smp_read_barrier_depends(); /* Enforce dependency ordering from x */ \
    __u.__val;                          \
})
// ...
#define __READ_ONCE_SIZE                        \
({                                  \
    switch (size) {                         \
    case 1: *(__u8 *)res = *(volatile __u8 *)p; break;      \
    case 2: *(__u16 *)res = *(volatile __u16 *)p; break;        \
    case 4: *(__u32 *)res = *(volatile __u32 *)p; break;        \
    case 8: *(__u64 *)res = *(volatile __u64 *)p; break;        \
    default:                            \
        barrier();                      \
        __builtin_memcpy((void *)res, (const void *)p, size);   \
        barrier();                      \
    }                               \
})
```

注：`volatile`不推荐使用的另一个原因是定义复杂，不说人话（至少我品鉴不出下面的句子说的是啥）

```
This version guarantees that “Accesses through volatile glvalues are evaluated strictly according to the rules of the abstract machine”, that volatile accesses are side effects, that they are one of the four forward-progress indicators, and that their exact semantics are implementation-defined.
```

而它能用于并发正确性是因为本用于MMIO，需要指令层面上要满足一定顺序，而这恰好用于编译器屏障的保证（当然它并不只是提供顺序上的保证，还有很多编译优化都可以阻止，invented load、store fusing等等，考虑另外写一篇文章总结）

## 总结

这篇文章仅仅是对于x86特定体系下做并发编程的一些指引，它指出：

- 一些关于并发的硬件基础问题的讨论
- 语言抽象memory     order和特定体系memory order的差距
- 原子性提供的保证
- `volatile`的特定用途

