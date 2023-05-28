---
layout: post
title: 浅谈C++内存模型
categories: [C++, Concurrency]
description: 本文章是从较为实际的角度去分析C++内存模型，涉及到memory order，modification order和release sequence
---

## 声明

本文章是从较为实际的角度去分析C++内存模型，涉及到memory order，modification order和release sequence

虽然内容和概念相比于标准是有所删减的，但我希望这篇文章相比标准和cppreference都更易于理解

<!--more-->


 

## 问题

先给出一个问题：如果有2个线程并发执行以下代码，结果是否正确

 

```C++
int val;
bool flag;
void writer() {
    val = 1;
    flag = true;
}
void reader() {
    while(!flag);
    assert(val == 1);
}
```

 

虽然很过程很直观，`writer`线程写完`val`数据后设置`flag`表示已完成，`reader`等待`flag`设置后读取`val`数据

但是答案是`assert`不一定满足

因为这些**非原子**的读写操作并没有满足强制的**内存访问顺序**

 

> 注：这里不必从具体的arch去分析，比如不同CPU的cache副本、乱序执行等角度去考虑，我们需要观察的是语言标准提供的内存模型


而所需要的强制内存访问顺序在C++中需要2种关系去确认：
- happens-before（先行关系）
- synchronizes-with（同步关系）

 

## 先行关系

先行关系是一个简单的概念，用于界定哪些操作看见其它哪些操作产生的结果（表达式求值和副作用）

在实际应用中，单线程下满足sequenced-before（既按照program order去理解）即可得到一种先行关系：

比如又如下代码：

```C++
int x = 5;
int y = 7;
```

在代码顺序上，第一行代码（L1）早出现在第二行代码（L2）的前面，因此L1先行于L2

 

但是此时只有先行关系并不足与保证线程安全

因为这只形成了偏序关系

 

> 注：sequenced-before关系只适用于单个线程

 

回到问题提到的代码里，

```C++
void writer() {
    val = 1;            // xL1
    flag = true;        // xL2
}
void reader() {
    while(!flag);       // yL1
    assert(val == 1);   // yL2
}

```

按照先行关系，显然xL1 < xL2，且yL1 < yL2

但是xL2和yL1的关系并没有确认，换句话说就是根本不能假定y能看到x操作产生的结果

 

## 同步关系

同步关系能限制一个原子变量能读到的值，而且具有可传递性，构成线程间的线性关系（inter-thread happens-before）

 

同步关系相比线性关系，性质比较特殊：
- 只作用于**同一个原子变量**的读写操作
- 需要加上合适的**标记**，才能让读操作与写操作之间产生的同步关系
- 有不同的标记提供不同强度的同步关系

 

而上面提到的标记既是memory order，用于指定如何进行内存访问

一般来说，memory order有三种语义提供给开发者使用：
- `relaxed`
- `acq-rel`
- `seq-cst`

这种分类也是按照同步的强弱程度划分的：不需要同步（relaxed宽松模型）、需要同步（acquire-release获取释放模型）、需要同步且要求全局顺序一致（sequential consistency，这种一致性指的是所有的线程能同一个顺序地观测到所有的改动，既只要某个线程观察到某个操作的结果，那么所有线程也都能观察到）

 

> 注：memory order需要完整标记，如果有一个writer thread的原子变量的`store`操作施加`seq_cst`而对应reader thread同一原子变量的`load`操作却只施加`relaxed`，那么在语言层面上并不提供同步和全局顺序一致性的保证

 

> 注：memory order中还存在一种consume标记，但对大多数架构来说意义不大，这里不做讨论

 

> 注：我不想提memory order具体解释需要的reorder和线程可见的事情，因为我觉得只靠同步关系这个概念就足够提供正确的编程方式

 

现在把问题中的代码修改为`atomic`类型，来提供同步关系

```C++
int val;
std::atomic<bool> flag {false};
void writer() {
    val = 1;                                        // xL1
    flag.store(true, std::memory_order_release);    // xL2
}
void reader() {
    while(!flag.load(std::memory_order_acquire));   // yL1
    assert(val == 1);                               // yL2
}

```

 

此时使用了获取释放模型来让xL2同步于yL1，只要yL1读到xL2提供的值（因为`flag`是`atomic`，原子变量**本身**是可以看到自身的改动），那么就产生同步关系构成线程间的先行关系，既xL1 < xL2 < yL1 < yL2

 

简单来说，通过synchronizes-with和happens-before的可传递性，即可保证线程安全

![acq-rel](/img/cpp_memory_model_acq_rel.png)

`*po = program order, *sw = synchronizes-with`

> 注：为什么不把同步关系等同视为线程间先行关系？这说的好像同一回事啊——那当然是概念上有简化，比如inter-thread happens-before还可以通过dependency-ordered before来构成，需要更细致的了解可以翻文档

 

> 注：严格来说先行关系是包含同步关系的，在cppreference描述中，先行关系满足两者之一即可构成，一是sequenced-before，二是inter-thread happens-before。而前面文章中提到的先行关系其实是概念上简化为sequenced-before

 

## 改动序列

前面提到，原子变量本身可以看到自身的改动，其实这里有一个概念modification order（改动序列）

这个概念不影响编程上的正确性，只是让你了解一个atomic load所得到的值到底来自于哪里

 

可以认为，每个原子变量都维护一个全局（所有线程可见）的改动序列，其序列中的每一个值由`store`操作提供，而`load`操作则读入改动序列中的某一个值

 

严格来说这里又得引入读写一致性（read-write coherence）的概念，但这里简化一些，知道一些感性的规则即可：
- 序列中的每个值由store操作提供
- 序列中的第一个值就是原子变量初始化时提供的值
- 如果某线程的load操作得到某个值v，那么这个线程此后load的结果要么是v，要么是序列中比v更靠后（更新）的值，这个线程后续的store操作提供的值也会append到序列的后方
- 如果某线程进行store后接着load，那么该线程load得到的值要么是刚才最后一次store的结果v，要么是序列中比v更新的值

 

## 释放序列 

当一个原子变量执行了release操作（即为A），此后任意线程对该原子变量的RMW操作所形成的最长子序列就称为——以A为首的释放序列

 

这个概念有什么用？其实是用于补全释放获取模型的完整性

在释放获取模型中，直接的理解就是：标记acquire的load操作能看到release前的所有操作（只要能读到对应的值）

但其实加上释放序列的概念后还可以如此补充：标记acquire的load操作能看到释放序列前的所有操作（只要能读到对应的值）

 

可以看如下的生产者-消费者例子，如果缺少释放序列，那么这段代码可能是错误的

```C++
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <mutex>
std::vector<int> queue_data;
std::atomic<int> count;
std::mutex m;

void process(int i) {
    std::lock_guard<std::mutex> lock(m);
    std::cout << "id " << std::this_thread::get_id() << ": " << i << std::endl;
}

void populate_queue() {
    unsigned const number_of_items = 20;
    queue_data.clear();
    for (unsigned i = 0;i<number_of_items;++i) {
        queue_data.push_back(i);
    }
    count.store(number_of_items, std::memory_order_release); //#1 The initial store
}

void consume_queue_items() {
    while (true) {
        int item_index;
        if ((item_index = count.fetch_sub(1, std::memory_order_acquire)) <= 0) //#2 An RMW operation
        {
            std::this_thread::sleep_for(std::chrono::milliseconds(500)); //#3
            continue;
        }
        process(queue_data[item_index - 1]); //#4 Reading queue_data is safe
    }
}

int main() {
    std::thread a(populate_queue);
    std::thread b(consume_queue_items);
    std::thread c(consume_queue_items);
    a.join();
    b.join();
    c.join();
}
```

 

这段代码中有一个生产者`a`，两个消费者`b`和`c`

`a`处理好私有数据后通过`release`操作发布`queue_data`，后续被`b`和`c`不断消费

这段代码是符合开发者直觉的，但是如果没有释放序列的话，虽然有以下的同步关系：
- `a#1` < `b#2`
- `a#1` < `c#2`

但是注意如果线程是按照这样的顺序执行：
1. `a`生产`queue_data`，通过`release`发布出去
2. `b`消费了一个`item_index`
3. `c`消费了一个`item_index`

这里由于步骤2的操作是RMW，改动了`atomic`类型`count`的值，那么步骤3的操作并没有与步骤1形成同步关系，此时`c`接着读入`queue_data`就是一个未定义行为（因为此时`count`已经不是由`a#1`通过`release`标记写入的值，`c#2`中`acquire`标记已经无济于事，所以`queue_data`也未发布给`c`）

而如果引入释放序列来修正获取释放模型，那么`c#2`读到的是以`a#1`为首的释放序列当中的任意值，因此仍然满足`acquire-release`语义

 

> 注：RMW操作即使是relaxed标记，也能形成release sequence

 

> 注：操作A是release操作而非store操作

 

> 注：“读到对应的值”指的不仅是release操作写入的值，还包括以它为首的整个释放序列中的任意值

 

> 注：如果该例子中只有一个消费者，那无需释放序列也能保证正确性

 

## References

cppreference

C++ Concurrency in Action, 2nd

[c++ - What does "release sequence" mean? - Stack Overflow](https://stackoverflow.com/questions/38565650/what-does-release-sequence-mean)

[Release-acquire ordering: must load the value that was stored](https://en.cppreference.com/w/Talk:cpp/atomic/memory_order#:~:text=Release-acquire%3Aordering%3A%3Amust%3Aload%3Athe%3Avalue%3Athat%3Awas%3Astored)

[Concurrency: Algorithms and Theories](https://hongjin-liang.github.io/teaching/concurrency/index.html)
