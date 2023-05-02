---
layout: post
title: 你说的协程，它真的快吗
categories: [C++, Operating System]
description: 聊聊协程的适配与性能测试
---

前段时间抱着玩票的性质搞了个协程库[co](https://github.com/Caturra000/co)

然后打算把它合并到鸽了一年的future网络库[fluentNet](https://github.com/Caturra000/FluentNet)

<!--more-->


## 协程的适配

可这是2种不同的异步实现方式，说要适配协程，我也没啥好思路，不妨理一下：
* 网络库用到的`promise / future`本身仍是一个线程内`event loop`这种操作，它每一个控制流本身仍是函数
* 协程库没有做`hook`，就是协程原语`resume / yield`的实现，能把你的函数拆成多个控制流

这么看，不如把`future`内的回调从函数单体改为协程单体，在仔细想了一下，似乎只需要在`loop`的时候加点轻微的改动就好了

这样子，`future`内的控制流都可以随时随地的`yield`，底层的`looper`会帮你接管好下一次`resume`时机（当成一个调度器来用了）

我确实这么尝试了：
* 为了支持并入网络库后仍然header-only的结构，`co`也需要改造成header-only，但是由于用到汇编，有单独的`.S`文件，需要一些trick来干掉这个`.S`：[co/contextswich](https://github.com/Caturra000/co/blob/header-only/co/contextswitch.h)
* 网络库中针对`event-loop`从`callback`执行改写为`resume`调用：[apply co to future](https://github.com/Caturra000/MagicAndFun/commit/4c4897c67b4dec9a5a30d3ee39dfc31a95f9012e)

试了一些简单的test，没啥问题

至少正确性勉强过关（老实人内心想法：复杂的test需要花更多个人业余时间，而且没啥回报和快感，不想写）

## 性能测试

接下来是性能测试，先说结论：实在是有点失望

（这一块代码没有放到github上，因为写的很shit）

### then测试

如果是对于`then`的调用开销的话，改动前后的差距会非常大

```C++
#include <unistd.h>
#include <fcntl.h>
#include <bits/stdc++.h>
#include "../src/Futures.h"


int main() try {
    SimpleLooper looper;

    constexpr int count = 1e6;
    auto start = std::chrono::system_clock::now();
    int c = 0;
    bool stop = false;
    for(int _ {count}; _--;) {
        auto fut = makeFuture(&looper, &c)
            .then([&](int *c) {
                (*c)++;
                if(*c == count) stop = true;
                return nullptr;
            });
    }
    for(; !stop;) looper.loop();
    auto end = std::chrono::system_clock::now();
    std::chrono::duration<double, std::milli> delta {end - start};
    std::cout << delta.count() << std::endl;
} catch(const std::exception &e) {
    // LOG(...)
}
```

传统`future`加函数实现是183ms

而`future`内嵌协程实现为2721ms

### poll测试

进一步地测试`poll`性能

```C++
int main() try {
    SimpleLooper looper;

    constexpr int count = 1e6;
    auto start = std::chrono::system_clock::now();
    int c = 0;
    bool stop = false;
    auto fut = makeFuture(&looper, &c)
        .poll([&](int *c) {
            (*c)++;
            if(*c == count) {
                stop = true;
                return true;
            }
            return false;
        });
    for(; !stop;) looper.loop();
    auto end = std::chrono::system_clock::now();
    std::chrono::duration<double, std::milli> delta {end - start};
    std::cout << delta.count() << std::endl;
} catch(const std::exception &e) {
    // LOG(...)
}
```

前后对比是14ms上升到34ms

这种差距还算是可接受的

## 测试意味着什么

意味着协程不是万金油，至少这里根本不适合

第一种`then`其实是约等于直接异步创建/调用$10^6$个函数实例/协程

第二种`poll`约等于只有1个反复调用的函数函数实例/协程

换句话说第一种测试中协程翻车就是你“并发”地开了百万个协程，协程实例的创建是个比`std::function`还要成本高的大头

> 注：我来算一算为什么更加大头，`co::Coroutine`除了本身也需要`std::function`以外，还需要14个寄存器保护现场的空间，并且堆上有伪造的栈占$2^17$字节（可以自行调整），以及为了生命周期上的引用计数管理。这对于内存分配算法来说压力确实大，尤其是伪造栈

第二种测试中两者都有相似的性能，但是实现差异较大：
* 函数版本是每次拔出队列后又`std::move`到队尾，给你一种`yield`的假象，但很显然控制流是重新开始执行的，这也是为什么要用`bool`区分是否再次`poll`，是在设计上指引开发者
* 协程版本就简单多了，遇事不决直接`while(...) yield()`，当然为了适配原有的`poll`语义，控制流仍然是重新开始（如果需要真·协程`yield`的话可以直接用`then`然后`this_coroutine::yield`）

## 反思和进一步测试

为什么协程会被吹捧，我要的不就是性能吗？没什么不能满足？

且慢，性能是可以满足的，见：
* 上下文切换测试[perf/contextswitch](https://github.com/Caturra000/Snippets/blob/master/perf/contextswitch)
* 以及下面的`co`切换跑分

```C++
#include <iostream>
#include <memory>
#include <vector>
#include <chrono>
#include "co.hpp"
int main() {
    auto &env = co::open();
    int counter = 1e7;
    auto func = [&] {
        if(counter-- < 0) return;
        co::this_coroutine::yield();
    };
    std::vector<std::shared_ptr<co::Coroutine>> coros;
    for(size_t count = 1; count--;) {
        coros.emplace_back(env.createCoroutine(func));
    }
    auto start = std::chrono::system_clock::now();
    while(counter >= 0) {
        for(auto &&co : coros) {
            co->resume();
        }
    }
    auto end = std::chrono::system_clock::now();
    std::chrono::duration<double, std::milli> delta {end - start};
    std::cout << delta.count() << std::endl;
    return 0;
}
```

在同一台主机的测试中，线程的切换成本有~880ns，而`co`仅需~80ns，10倍的差距

（这里不代表所有协程的性能就是~80ns水平，但是已经符合协程的基本情况，你可以找找市面上其他协程的横向测评）

这在传统的每一个连接都开一条线程去处理的模式中有很明显的改进方式：把线程改为协程，每个线程的`block`行为用`epoll hook`结合`nonblock`和`yield`就能做到

这么做，性能不一定比上面的`callback`要高效，但是写法是最为朴素的同步写法：你不需要接受异步思想，享受着最为朴素的写法（甚至比`per-thread`更加朴素，直接`read / write`），却拥有比`per-thread`高效得多的性能

至于在已有完善异步机制的框架下再添加协程，似乎显得有些刻意，我需要进一步参考其它优秀实现的思想，再做决定要不要加入协程

## END

结论：
1. 协程很快，但目前来看不适合
2. 协程适合同步思想，配合`hook`使用

下一步：我决定再把异步网络库V3鸽一段时机

## 22.6.30 update

协程还是作为V2的一个特性加进来的，而不是V3

原因是`server`端仅少量使用`future`，跑了分发现相比callback并没有造成性能下降

而大量使用该特性的`client`端与其追求性能不如追求好用，测下来也没掉5%以上的吞吐量

也就是说，一个看似很离谱的损耗，放到整个系统中也并没有引起多大的反应

那就先这样吧，算是隔了快一年的小更新

顺手贴上commit：[add coroutine](https://github.com/Caturra000/FluentNet/commit/1f8d62a16f466af159678c82b90df5086c88d633)

## 22.7.2 update

前面提到，`then`测试的性能表现简直是一坨，原因在于内存分配算法无法承受大量模拟栈（上下文容器）的开销

那么延迟上下文的分配就是一个显著的改进点，对于一个尚未`resume`过的`coroutine`来说，并不会分配，而是等待第一次`resume`时才尝试分配，这有点像copy on write

这种策略似乎并没有帮助，因为你再怎么延迟，其实还是无法避免分配压力问题，哪怕是allocator自带cache，对于足够大的size，它也不能挽救

因此`resume`时也尽可能不分配，而是从自定义的复用池中捞出来，且用`FILO`的方式来达到尽可能高的局部性

这两种策略的加持下，能直接把上面的极端例子（`then`测试）给干掉，此时相当于所有的`coroutine`其实都用了同一个栈，十分适用于短连接操作

实际表现是从2721ms降低到276ms，基本接近原生`future`版本

贴上commit：[add context reuse](https://github.com/Caturra000/FluentNet/commit/706b314d04c4fecab6ccce1dac700499c3d462c8)