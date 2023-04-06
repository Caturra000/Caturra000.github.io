---
layout: post
title: 一些经典互斥算法的实现
categories: [Operating System]
---

虽然说是没啥卵用的东西，但学习证明它为什么是互斥的还是有点意思

<!--more-->


## 定义

箭头$$A \to B$$表示事件$$A$$先于$$B$$的意思，$$I(A,B)$$为$$A$$和$$B$$之间的间隔，多个不存在$$\to$$关系的间隔就称为是并发的

$$CS_A^k$$表示线程$$A$$第$$k$$次进入临界区里头的时间段

临界区的开始需要调用`lock()`，结束需要调用`unlock()`

## 锁算法的特性

1. 【必须】互斥：不同线程的临界区无重叠，设线程$$A,B$$和任意整数$$j,k$$，要么满足$$CS_A^k \to CS_B^j$$，要么满足$$CS_B^j \to CS_A^k$$
2. 【必须】无死锁：一个线程尝试获得锁，总会成功获得，否则一定存在其它线程多次执行临界区
3. 【可选】无饥饿：每一个试图获得锁的线程最终都能成功

## LockOne算法

`LockOne`是一种双线程锁算法，$$flag$$表示对资源感兴趣，并且过程是谦让的（但不知道是谁让谁）

`LockOne`类内部伪代码如下

```C++
class LockOne {
    bool[] flag = bool[2];
    lock() {
    	i = currentThread.getID();
        // 假设线程ID只有0和1
        // 并且假定这些变量都是满足可见性的
        j = i^1;
        flag[i] = true; // 我感兴趣
        while(flag[j]); // 等待直到对方失去兴趣
	}
    unlock() {
        int i = currentThread.getID();
        flag[i] = false;
    }
}

```

对于一个算法，如何较为严格的证明它是互斥（安全）的？

接下来学习使用反证法来证明该算法的互斥

### 证明互斥

为了防止语文不过关，记$$write_A(x,v)$$为$$A$$将$$x$$赋值为$$v$$，$$read_A(x,v)$$为$$A$$从$$x$$读取到$$v$$

互斥不成立的条件：那就是不满足“要么$$CS_A^k \to CS_B^j$$，要么$$CS_B^j \to CS_A^k$$”的条件

观察算法的`lock`过程：

1. $$write_A(flag[A],true) \to (wait) \to read_A(flag[B],false) \to CS_A$$，这是A进入临界区的前置动作，`read`操作是决定进入临界区的关键
2. 同理，$$write_B(flag[B],true) \to read_B(flag[A],false) \to CS_B$$

考虑不满足互斥的情况下，$$CS_A$$进入后仍能进行$$CS_B$$的执行，那就有$$1 \to 2$$的过程（即满足决定进入$$CS_B$$的`read`操作），而无需`unlock`，化简得出

$$write_A(flag[A],true) \to read_B(flag[A],false)$$

很显然这是矛盾的



### 死锁的存在

再重新观察一下过程

$$write_A(flag[A],true) \to read_A(flag[B],false) \to CS_A$$

$$write_B(flag[B],true) \to read_B(flag[A],false) \to CS_B$$

线程在进入$$CS$$之前是没有约束的

如果线程交叉执行，导致$$write_A(flag[A],true)\&write_B(flag[B],true)$$

这样无法执行$$read_A(flag[B],false) \| read_B(flag[A],false)$$的任意一步

这样就进入死锁状态了

要想保证死锁不发生，那就要一个线程在进入临界区后再启动另一个线程



## LockTwo

同样是伪代码，$$x$$表示谦让别人/自我牺牲的ID（无法感知别人的存在）

```C++
LockTwo {
    int x;
    lock() {
        i = get_threadID();
        x = i; // 依然保证可见性
        while(x == i);
    }
    unlock() {}
}
```

### 证明互斥

同样的分析过程，观察

$$write_A(x,A) \to read_A(x,B) \to CS_A$$

$$write_B(x,B) \to read_B(x,A) \to CS_B$$

要想达成$$read_A(x,B)$$，则需要$$write_B(x,B)$$（先于），既$$write_B(x,B) \to read_A(x,B)$$

如果不满足互斥，$$write_A(x,A) \to write_B(x,B) \to read_B(x,A)$$

这也确定了该情况是矛盾的

### 单线程阻塞

由上面的过程可看出，这种算法的奇葩在于必须并发执行才能实现互斥（自己看看条件吧，写公式真累）



## Peterson锁

当把`LockOne`和`LockTwo`结合时，则有一个完美的双线程互斥锁算法（满足前面提到的三大特性）

```C++
class Peterson {
    bool flag[2];
    int x;
    lock() {
        i = id();
        j = i^1;
        flag[i] = true; // 我感兴趣
        x = i;          // 但你先走
        while(flag[j] && x == i);
    }
    unlock() {
        i = id();
        flag[i] = false;
    }
}
```

### 证明互斥

（一个意思，略）

## 过滤锁

过滤锁是支持$$n$$线程的互斥协议，设有$$n-1$$个称为层的等候室，线程要想获得锁，必须穿过所有的层

$$level[A]$$表示$$A$$尝试进入（感兴趣）的最高层次

总之就是地铁站逐层过滤的意思

```C++
class Filter<n> { // 伪代码，别较真
    int level[n] = {0};
    int x[n]; // index0不使用
    lock() {
        int id = getID();
        for(i:[1,n)) {
            level[id] = i;
            x[i] = id;
            while(level[k] >= i && x[i] == id); // 存在 k != id
        }
    }
    unlock() {
        level[getid()] = 0;
    } 
}
```



## Bakery锁

$$Bakery$$锁是通过分布式版本来保证FCFS：线程获得序号，一直等待直到没有序号比自己更早尝试进入（面包店）

$$flag[A]$$表示线程$$A$$对资源是否感兴趣，$$label[A]$$表示进入资源的次序

```C++
class Bakery<n> {
    bool flag[n]; // false
    int label[n]; // 0
    lock() {
        id = getID();
        flag[id] = true;
        label[i] = max(label[0,n))+1;
        while(flag[k]&&(label[k]<label[i] || (label[k]==label[i]&&k<i)));
    }
    unlock() {
        flag[getID()] = false;
    }
}
```



## 多线程互斥总结

对于Filter和Bakery锁，他们都是互斥、无死锁、无饥饿的

但对于$$n$$个线程的无死锁互斥算法，在最坏情况下至少需要读/写$$n$$个不同的存储单元，而且不可避免

## 参考

《多处理器编程的艺术》 Chapter.2
