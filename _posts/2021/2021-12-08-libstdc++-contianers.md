---
layout: post
title: 「时局图」libstdc++的实现
categories: [C++, RTFSC]
description: 不言而喻，一目了然
---

不言而喻，一目了然

<!--more-->

## 前言

好的design应该铭记心中，万一我要造轮子，对着抄都方便点

因此尝试用`UML`类图来描述实现，期望能做到风格上的一致（指我看得都顺眼）

注1：图比较大，文章用的是缩略图，需要通过链接才能看到原图

注2：一些大家都知道的公有函数不一定会写上，因为这里填上函数名是为了方便跟踪用的（用了萃取之后IDE都变蠢了）

注3：图要看，代码也要看，这才称得上是健全


## 源码和注释

请看这里：[GitHub/Caturra000/RTFSC/libstdc++](https://github.com/Caturra000/RTFSC/tree/master/STL/libstdc%2B%2B)

## shared_ptr

![shared_ptr](/img/shared_ptr.png)

TODO: 尚未添加`weak_ptr`和`enable_shared_from_this`的结构

从图中可以看到：
- 并发安全的处理，线程安全的支持是达到了什么程度
- 为什么推荐用`std::make_shared`
- `custom deleter`是怎样支持的
- `shared_ptr`的额外成本

我还分析了EASTL的`shared_ptr`实现，代码结构上会更加直接点：[EASTL/shared_ptr](https://github.com/Caturra000/RTFSC/tree/master/STL/EASTL/smart_ptr/shared_ptr)

## unordered_map

![unordered_map](/img/unordered_map.png)

`unordered_map`用`UML`图看不出什么，其中很多复杂的类型萃取还得看源码才能理清

虽说实现上有一定独到之处，但是阅读难度太大了，很多时候都需要靠这张图才能看清调用链（非主流的`GNU-sytle`加上魔怔一样的template应用）

如果只是需要看特定的`unordered_map`而不是极高灵活性的`_Hashtable`，可以参考[libstdc++/unordered_map](https://github.com/Caturra000/RTFSC/tree/master/STL/libstdc%2B%2B/unordered_map)下面的一些流程，我都把template特化给展开了，方便查阅

其实就算法而言，`unordered_map`还是很朴素的，无非就是取余 + next prime + `std::hash`，即使调用方能通过特化来提高性能，又有谁会写这么复杂的东西

比较好奇的是为啥负载因子是1.0，为啥它会这么自信（这就是被当年Codeforces出题者疯狂针对性卡数据的原因？）

## deque

![deque](/img/deque.png)

相比前面几位重量级，`std::deque`内部实现简略太多了

（而且它使用`base`做继承是为了更方便处理异常，没别的意思）

但是，我在想`deque`的分配策略，对于`std::stack`和`std::queue`这些适配器来说是不是很不利？

~~简单的来说`deque`喜欢三等分的做法，而那些屏蔽某一端接口（双端变单端）的适配器的`load factor`自然不会高到哪里去~~

~~不过我还没细看，只是把想法放在这里~~

收回前言，应该是不对等的三分，但仍需要跟踪一下`reallocate`后的布局

update. 我看完reallocate了，也看懂了它的布局，然而[槽点](https://github.com/Caturra000/RTFSC/blob/master/STL/libstdc%2B%2B/deque/%E9%87%8D%E5%88%86%E9%85%8D%E6%B5%81%E7%A8%8B)已经写满屏幕了

## queue/stack

略，适配器没有资格配得上我做图

## vector

![vector](/img/vector.png)

## TO BE CONTINUED

其他待更新，自己一幅幅画很累的

另外严重不推荐Graphviz，浪费时间
