---
layout: post
title: 又一个raft实现
categories: [C++, 轮子]
---

写了一个C++实现的raft，这里就简单做些记录

<!--more-->


## repo

[github/Caturra000/cxxraft](https://github.com/Caturra000/cxxraft)

实现raft要用到一些基础库，当然也要自己写了：
* [json库](https://github.com/Caturra000/vsJSON)
* [日志库](https://github.com/Caturra000/dLog)
* [协程库](https://github.com/Caturra000/co)
* [RPC库](https://github.com/Caturra000/tRPC)

## 耗时

耗时约三周，基本一周一个part，其实如果不是2天写代码3天摸鱼的话，还能省一半时间
（比如持久化这一部分，你要写其实半天不用，但我还是摸了一周）

另外我觉得debug花的时间远比写的时间要多，因为你需要认真琢磨paper

如果有机会再写一遍，不妨把写代码的节奏放慢一点，多想想实现上的正确性和后面是否可维护

## 参考

参考资料主要看[raft paper](https://raft.github.io/raft.pdf)，前人踩过的坑我没有意识到，但其实paper里面都有提

raft paper不只是paper，还是非常好的实现参考手册。细节上的推敲比前司的代码注释还要靠谱多几倍，我以后写文档能有这水平该多好

测试是通过适配MIT-6.824的[test_test.go](https://github.com/chaozh/MIT-6.824/blob/master/src/raft/test_test.go)文件来完成。但是由于它们是lab形式，改造成本那是相当大（后面实现细节再提）


## 实现上的细节

这里会提一些paper上不怎么强调（我觉得）的实现细节

### 协程模型

paper只是说要怎么跑，没说要如何设计哪个模块在哪个线程/协程上

方法很多，我这里是基于协程实现

每个`raft`服务都至少开启2个协程：
* 一个是RPC监听协程co0
* 另一个是FSM（状态机）协程co1

依据FSM不同的状态（`leader` / `follower` / `candidate`）会产生更多的协程

比如RPC每次处理是固定拉出一个协程，像appendEntries这种需要发出给不同`peer`的也会生成`peer`个协程进行RPC call

一旦发生状态上的转换（比如接收RPC时，发现自己的`currentTerm`已经落后于收到的`term`，要求转为`follower`），这些拉出去的协程就需要进行取消操作

一个简单的做法是协程带上事务标记`transaction`，转换时对标记递增即可，只要协程在对应的取消点检查标记无法对应，则立刻退出

### 状态间的转换

对照raft paper中的状态转换图可以知道

FSM中的每一种状态（`leader` / `follower` / `candidate`）都会处于一个大循环内

并且通过逻辑时钟`term`、选举投票`voteFor`或者本地RPC超时等行为进行任意状态间的转换

这里会有一些坑

#### follower维持

比如，paper中提到`follower`没有收到RPC时会转换为`candidate`参与选举，反之收到RPC则保持`follower`。

这里需要注意，如果收到`requestVoteRPC`，那是不计入保持follower状态的，该超时还是会超时

#### 心跳处理

还有`leader`心跳收到的回复，如果实现了日志副本，那就不应该区分空包心跳和`appendEntriesRPC`，收到`peer`回复就要处理返回值，而不是一直心跳就不管了

#### 转换的处理

一个略显隐蔽的地方是，不建议直接通过调用函数去转换状态

比如从`leader`到`follower`，就不要在`becomeLeader()`跑一堆循环满足条件后再调用`becomeFollower()`

因为状态机是一个环，你这样写其实是是一个调用链很深的递归，如果实现方式不对，那是会爆栈的（编译器不会聪明到所有尾递归都能转换）

可以写成事件循环的形式，投递一个事件，消费一个事件，固定在某个协程中执行，这样就避免了递归问题


#### 过期的回复

收到RPC时如何处理stale term，paper没有细说

一个简单的策略就是回复一个term为负数的RPC response

原发送方收到后优先处理term（得知比自己落后，表明接收方没有`followUp()`到最新的term而是选择了忽略），再看其它字段，因为过期就应该忽略而不是处理

### RPC上的实现

#### 连接管理

RPC在初期实现时client端建议用短连接，否则容易在不可靠网络上hang住无法实现下一次对应`peer`的心跳或者投票，换句话说就是不要在原来拉出的协程继续执行，这样就陷入和同步写法相同的blocking状态

另外这里用长连接感觉收益并不大，因为下一次需要用到client时都是需要一定的定时时间的，本身就需要花费数百毫秒

但是后面使用到一定优化技巧时可以考虑每次定时周期对每个`peer`维护单个长连接RPC client


#### 不可靠网络

由于test_test.go用到的是一个假的网络，而我的实现就是走TCP协议栈的，因此后面要做一个不可靠网络模拟就非常纠结了

一个简单的做法是实现一个代理层，比如收到未处理的`request`或者要发出前的`response`都走到某一个（或多个）`proxy`上，那就方便操作了，

协程上`usleep`（注意不是系统调用）一定的时间就可以模拟RTT延迟

靠一定概率把`response`丢弃也可以模拟丢包

包乱序就不需要实现了，只要你保证每一个session都用单独的TCP连接去处理，本身就不会有乱序问题，不然还用什么可靠传输协议

### 日志副本

#### Safety

`appendEntriesRPC`用于同步日志副本

其中关键的一点是`prevLogIndex`对于前缀空的日志是默认允许的（假设为从`index=-1`的日志开始发出的`entries[]`），既对端收到后要reply true

但不能说`prevLogIndex`对应日志为空就是上面的行为，比如当前leader crash后又选出一个新的leader，而此前leader发出的RPC因为日志不一致而做了强制截断（paper中提到的overwrite）处理，那么后来的leader发出的prevLogIndex可能过于超前，从而误判为前缀空的日志

以及前面说到的要和心跳一致处理，否则对端收到`leaderCommit`并更新自身commitIndex后，`leader`还不知情

#### 简单的优化

对着paper做实现不一定能过所有的test，因为模拟了一些极端场合，你要在一定时限内完成，所以不仅要正确性，还要一定的性能

简单点的策略有对发出的`entries[]`进行批量发出，这里做了个倍增处理，默认64个，如果成功，下次再尝试128个，失败则退回，如此递推

还有别人提到的快速恢复，失败时考虑整个term都跳过，而不是`nextIndex`只减一，还没有想过是否有正确性问题，暂时没举出反例

以及快速重试，`appendEntries`是定时触发的，paper提到如果失败则进行立刻重试（而不是等到下一次定时触发），我在实现中进一步如果成功且下一次预估不会发出空包心跳时也会立刻重试（发出更多的`entries[]`）

paper还提到`conflictTerm`，我没做，但也是个思路

还有

### 持久化

MIT lab用到的持久化看着是全局一次性落盘的，我这里用的是write-ahead log，每次重启时对着redo一遍即可（[给个SQLite的做法以供参考](https://www.sqlite.org/wal.html)）

当然并不是一定要用write-ahead，只是提供在持久化的同时实现随机写改为顺序写的能力

### 字段维护

一个比较诡异的是，raft中提到的`lastApplied`我从来都没用过

虽然commit和apply也算是两回事，但因为这里写的是纯算法层，并没有提交到上层的`apply`操作，因此没管

往[知乎](https://www.zhihu.com/question/382888510/answer/1555866662)搜了一遍，确实不影响正确性，那算了吧

### 问题排查

分布式上的问题排查还是靠打日志

因为异步事件、定时超时事件太多了，运行时打断点并没有好的回报，

除非你是要排查某些地方突然hang住了但是又没有crash，这个时候可以确实可以debug看下，但最好debug完立刻在对应地方打上日志，方便下次排查

## END

暂时就想到这么多，先这样吧


