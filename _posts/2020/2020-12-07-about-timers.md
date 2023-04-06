---
layout: post
title: 定时器的简单讨论
categories: [C++]
---

轮子所用的定时器方案大概定型了，觉得可以对一下思路，讨论一下逐步扩展的实现
<!--more-->


# 需求

要设计任何一个东西其实要基于需求比较好，虽然本质上我就是想自己造轮子，但是需求能给人一种方向感，以及能说服自己为什么要这么写

- 功能相对丰富
- 性能不太拉跨
- 线程安全
- 接口友善
- 代码简洁



## 功能与接口

接口的友善基于两点：`user define literal`和`builder`模式

在说明前我先介绍在设计之初就构思的两个类：`TimerEvent`和`Timer`，

`TimerEvent`包含时间参数和可调用对象，`Timer`封装调度用的容器

具体操作就是构造`TimerEvent`然后丢给`Timer`去调度

在功能的提供方面，我至少确定了：

- 什么时候开始（`when`）
- 调度的内容（`what`）
- 允许多次（`atMost`）
- 多次调度的间隔（`interval`）
- 相同时间下可能的优先级（`ticket`）

### user define literal

这个是`C++11`比较少见的特性，但是用在定时器上是一个非常exciting的搭配

基本解决了怎么传时间参数的痛点

比如你要设计一个`runAfter(/*传入时间*/)`的接口，一般要怎么设计

是通过一个`runAfter(int timeMs)`这样的声明

还是构造一堆的`class Second / Minute / Hour`然后重载多个`runAfter(Second / Minute / Hour)`再通过`runAfter(Second.valueOf(2))`来实现

前面用整型语义的问题是不同时间单位难表示，调用方也无法清晰明白是基于什么单位

后面的基本解决了上面提到的两个问题，但是`user define literal`可以更好地完成

`用户自定义字面值`只需这样就足够了：`runAfter(2s) / runAfter(1h)`

并且不需要考虑重载，`std::chrono`已经帮你搞定了，使用上参数类型选择`std::chrono::nanoseconds`就可以兼容`ns/ms/us/s/min/h`

### Builder

当`TimerEvent`参数越来越多时自然会考虑一个构造函数的参数顺序问题

现在的`TimerEvent`大概是这样

```C++
struct TimerEvent {
    Timestamp _when;
    Nanosecond _interval;
    uint64_t _atMost;
    uint64_t _ticket;
    Callable _what;
};
```

当需要一个构造函数时，哪些参数放在前面，哪些放在后面，哪些提供默认可忽略都难以抉择

所以干脆点也用户自行选择

```C++
class Timer {
    // ...
    struct TimerHelper {
        Timestamp _when;
        Nanosecond _interval;
        uint64_t _atMost;
        Timer *_thisTimer;

        template <typename ...Args>
        TimerHelper& with(Args &&...args) { 
            _thisTimer->append(_when, 
                Callable::make(std::forward<Args>(args)...), 
                _interval, _atMost); 
            return *this;
        }

        TimerHelper& per(Nanosecond interval) {
            _interval = interval;
            return *this;
        }
        TimerHelper& atMost(uint64_t times) {
            _atMost = times;
            return *this;
        }
        TimerHelper at(Timestamp when) {
            _when = when;
            return *this;
        }

        TimerHelper priority(URGENCY urgency) {
            _ticket = urgency;
            return *this;
        }
    };

    // usage:
    // runAfter(2s).per(2s).atMost(3).with(func, arg0, arg1);
    // runEvery(500ms).at(now()+1s).with([]{std::cerr<<"b";});

    TimerHelper runAt(Timestamp when) 
        { return TimerHelper{when, Millisecond::zero(), 1, this}; }
    TimerHelper runAfter(Nanosecond interval) 
        { return TimerHelper{nowAfter(interval), Millisecond::zero(), 1, this}; }
    TimerHelper runEvery(Nanosecond interval) 
        { return TimerHelper{now(), interval, std::numeric_limits<uint64_t>::max(), this}; }
};
```

实现在`Timer`中甚至不需要知道`TimerEvent`的存在，只需由`timer`方调用即可

### 选择的权利-续

`Builder`是一种将选择留给用户的做法，在定时器的启动函数`run()`中也需要这种考虑

在最初我的设计是

```C++
class Timer {
    // ...
    void run() {
        for(; !_stop; ) {
            // 处理TimerEvent
        }
    }
};

```

这个问题很明显，一旦`run()`就必须强占（至少）一个线程

尽管我觉得大部分简单使用还是`for`完事，但是很明显是否强占线程应该留下余地

```C++
class Timer {
    // ...
    void run() {
        // 处理一次到来的TimerEvent
    }
};

// 调用方
int func() {
    // ...
    for(;;) timer.run();
}
```

更进一步的，应该尽可能提供足够多的信息

```C++
class Timer {
    // ...
    Millisecond run() {
        // 处理一次到来的TimerEvent
        // return 距离下一次调度的最近时间差
    }
};
```

这个会在与`epoll`搭配时起作用

```C++
void loop() {
    // ...
    auto timeout = timer.run();
    epoller.poll(std::max(timeout, 1ms));
}
```

至少可省下为`epoll`提供`wakeup`的操作

## 性能和简洁

### 封装

定时器常见的三种算法实现之间的对比并不是讨论重点，我是毫不犹豫使用堆来实现的

原因是堆本身性能足够好，并且`STL`可以非常简洁的做到这个实现

```C++
class Timer {
    // ...
    using EventHeap = std::priority_queue<
            TimerEvent, std::vector<TimerEvent>, std::greater<TimerEvent>>;
    EventHeap _container;
};
```

容器的封装三行就完成了（其实还需要`operator>`，不过也很简洁）

### 实现

接下来讨论`run`的具体实现

该实现依赖于轮子已有的`Callable`，可以翻我前面讨论回调接口的文章查看实现

这是一个朴素实现

```C++
// 不会阻塞，只会执行到不符合条件的时间就返回
Millisecond run() {
    if(_container.empty()) return Millisecond::zero();
    Timestamp current = now();
    while(!_container.empty()) {
        auto &e = _container.top();
        if(e._when > current) break;
        Defer _ {[this] { _container.pop(); }}; // 避免手残忘记、call抛出异常而失效
        if(e._atMost > 0) e._what.call();
        if(e._atMost > 1) {
            _reenterables.push_back(std::move(const_cast<TimerEvent&>(e)));
            auto &b = _reenterables.back();
            b.next(); // 一个更新when atMost等成员为下一次调度做准备的函数
        }
    }
    for(auto &e : _reenterables) _container.push(std::move(e));
    _reenterables.clear();
    if(_container.empty()) return Millisecond::zero();
    return std::chrono::duration_cast<Millisecond>(_container.top()._when - now());
}
```

该实现非常浅显易懂，得益于堆的性质，直接把时间按最早到最晚pop对比当前时间，合适的就调用call，并即时更新、抛弃TimerEvent，最后插回原来的堆中

那接下来考虑线程安全问题，最简单上锁就可以了

```C++
Millisecond run() {
    std::lock_guard<std::mutex> _ {_mutex};
    if(_container.empty()) return Millisecond::zero();
    Timestamp current = now();
    // 跟前面一样....
}
```

考虑优化：能不能把持锁的粒度缩小一点？

考虑参考`muduo`中使用`swap`来减小锁粒度，可行不？

具体就是：

1. 实现依然和前面一样不需变更
2. 当mainThread用到timer时，`swap(timer, localTimer)`
3. 调度操作全在`localTimer`，其它线程的`TimerEvnet`添加往`timer`里加

很遗憾，`timer`最后是需要合并的，因为调度完成后还是会`timer`和`localTimer`依然持有`TimerEvent`，

而STL优先队列并不是快速的可并堆，`swap`的技巧并不适用

但是我们可以考虑把可能最耗时的`call`移出到其它线程中

在这里使用传入传出引用来避免不必要的`vector`构造

```C++
Millisecond run(std::vector<TimerEvent> &tasks) {
    std::lock_guard<std::mutex> _ {_mutex};
    Timestamp current = now();
    for(size_t n = tasks.size(); n; --n) {
        auto &task = tasks.back();
        if(task._atMost > 1) {
            task.next();
            _container.emplace(std::move(task));
        }
        tasks.pop_back();
    }
    if(_container.empty()) return Millisecond::zero();
    while(!_container.empty()) {
        auto &task = _container.top();
        if(task._when > current) break;
        tasks.emplace_back(std::move(std::move(const_cast<TimerEvent&>(task))));
        _container.pop();
    }
    if(_container.empty()) return Millisecond::zero();
    return std::chrono::duration_cast<Millisecond>(_container.top()._when - now());
}
```



考虑优化2：这种情况，真的需要全部push回原来的堆中吗

确实有些不需要，我们杂糅一下链表的做法，使部分需要$$O(logn)$$的地方降为$$O(1)$$

并且`_reenterables`仍是成员变量，就是说`capacity`是单调递增的，不用每次都分配

```C++
Millisecond run(std::vector<TimerEvent> &tasks) {
    std::lock_guard<std::mutex> _ {_mutex};
    Timestamp current = now();
    for(size_t n = tasks.size(); n; --n) {
        auto &task = tasks.back();
        if(task._atMost > 1) {
            task.next();
            if(task._when > current) {
                _container.emplace(std::move(task));
            } else {
                _reenterables.emplace_back(std::move(task));
            }
        }
        tasks.pop_back();
    }
    while(!reenterables.empty()) {
        auto &task = reenterables.back();
        tasks.emplace_back(std::move(task));
        reenterables.pop_back();
    }
    if(_container.empty()) return Millisecond::zero();
    // ...略
}
```



考虑优化3：reenter的转移是必要的吗

注意到必然为空的性质，`swap`一波就好，部分$$O(n)$$降为$$O(1)$$

```C++
Millisecond run(std::vector<TimerEvent> &tasks) {
    std::lock_guard<std::mutex> _ {_mutex};
    Timestamp current = now();
    for(size_t n = tasks.size(); n; --n) {
        auto &task = tasks.back();
        if(task._atMost > 1) {
            task.next();
            if(task._when > current) {
                _container.emplace(std::move(task));
            } else {
                _reenterables.emplace_back(std::move(task));
            }
        }
        tasks.pop_back();
    }
     _reenterables.swap(tasks);
    if(_container.empty()) return Millisecond::zero();
    // ...略
}
```

## 杂碎的讨论：优先级

说下`ticket`实现的优先级

这个优先级是指代相同的时间内，优先处理谁的意思

本来我的想法是让`ticket`只供内部使用，是用于解决每次相同时间内第$$i$$个`event`总是第$$j$$次调用的问题

```C++
// TimerEvent内
bool operator > (const TimerEvent &rhs) const {
    if(_when != rhs._when) return _when > rhs._when;
    return _ticket > rhs._ticket;
}

void next() {
    _when += _interval;
    _atMost--;
    if(!(_atMost & 7)) _ticket = random<uint64_t>(); // shuffle一波
}
```

可是后来想到，虽然初衷是要只用于解决这种场合

但是第一次调度的优先级是可以通过ticket得到保证的，因此在`TimerHelper`中开放了`priority`的`setter`

进一步的其实还可以考虑这样扩展优先级的定义：同一次调度中，尽可能地优先处理

也就是说，在单次的`while(!container.empty())...`调度中，只要优先级够高，就会优先处理

如果期望$$n$$个这样的`TimerEvent`的额外调度成本是$$O(n)$$，有没有一种容易扩展的方法？

其实还是很容易做到的，首先看我们接口最后的定案`Millisecond run(std::vector<TimerEvent> &tasks)`

如果`tasks`也是一个`priority_queue`，并且`operator>`重载只按照`ticket`比较，是不是可以了呢？

不过可惜的是这种$$n$$次插入的复杂度是$$O(nlogn)$$的，因为严格基于比较的做法的复杂度必然在$$O(nlogn)$$量级

如果不在乎优先级的精度且仍要求严格优先，可以进行桶排序并记录一个`size`来及时剪枝

能不能即使是`std::vector<TimerEvent> &tasks`也可以做到$$O(n)$$并排好优先级？

给出`Hint`：

1. 已有数据上直接构造堆是$$O(n)$$的复杂度
2. 对堆进行$$BFS$$尽管不太严格，但能实现高优先级的一半严格优先于低优先级
3. 最高的优先级永远得到严格保证

## 杂碎的讨论：定时与调度

`Timer`的定位永远就只是一个`Timer`吗，能不能在前面的`run`实现发现点什么？

也许不太明显，但是可以提示一下`muduo`的线程安全是如何实现的：把`event`放到`loop`线程下

那跟`Timer`有什么关系？

```C++
void loop() {
    for(std::vector<TimerEvent> tasks; !stop;) {
        auto timeout = timer.run(tasks);
        for(auto &&task : tasks) task._what();
        epoller.poll(std::max(timeout, 1ms));
        // ...
    }
}
```

这下是不是明显了？`Timer`同样是一个用于调度`event`到`loop`线程的工具

也就是说，在这种实现下，不仅完成了一个`Timer`，还间接做到了`runInLoop`的功能

## THE END

总的来说基本定时器的方案就是这样了，并不需要一个特别复杂实现的实例，只是简单分享一个简洁并且用到一点技巧的实现方案

--------------

2021.1 Update.

## 这好吗？这不好

当我意识到在开发过程中`timer`更多地应用于线程调度时，我就为它的插入性能感到捉急

对于线程调度而言，大量定时时间为0并且可能会尝试retry，但是在堆中却要经历漫长的`push up`是不应该的

因此我决定有空再造一个时间轮的轮子，谁不搀它的插入性能呢
