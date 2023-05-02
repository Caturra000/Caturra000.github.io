---
layout: post
title: 实现lockfree容器：freelist，stack和queue
categories: [C++, Concurrency, 轮子]
description: 实现一个xyz系列算是本博客的月指活了，这次写的是lockfree容器
---

实现一个xyz系列算是本博客的月指活了，这次写的是lockfree容器<del>（硬核程度越来越高，以后咋低成本水文章啊）</del>


## 声明

 

不建议自己从零造lockfree轮子，至少需要有paper支撑，或者从已有的项目中改进

否则无法证明代码是正确的

为此本文参考了模板库`boost::lockfree`的实现以及MS Queue的[paper](https://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf)（Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms）

<!--more-->



另外本文实现的容器只关注lockfree部分。因此，作为类设计可能是不及格的，比如完全忽视`const`语义接口支持和不支持基本异常安全

 

## 无锁freelist

 

万事开头难，`freelist`虽然实现上较为朴素，但是需要讨论基本的无锁实现注意事项还是挺多的

 

freelist的关键在于：使用过程中获取到的内存资源绝不返还给OS/libc（这不是内存泄漏，会在析构时再全部返还）

为什么要这么做呢？这是因为lockfree容器处理内存回收是个棘手问题，你需要hazard pointer或者epoch-based等技巧去处理。而保证资源可用则是一个简洁取巧的方案，实际上`boost::lockfree`库使用的就是这种方案

 

除了内存回收策略以外，还引入一个`tagged pointer`唯一指定指针，以解决ABA问题。另外一提，如果使用了上述不返还而反复reuse的方式去使用内存的话，高并发下ABA问题会变得非常严重（高频发生撞车惨案。。）

 

注：`tagged pointer`实现是基于x86-64在四级分页下只使用低48位有效地址的事实前提来完成的，不具有可移植性。如果平台支持double word CAS的话，也有更加简单的实现方式，这里不展开了

 

永久链接：[Snippets/Tagged_ptr.hpp at 55d403f52b6cbd6 · Caturra000/Snippets (github.com)](https://github.com/Caturra000/Snippets/blob/55d403f52b6cbd65721d44fd6999da5a35b40226/Concurrency/Tagged_ptr.hpp)

 

```C++
// 48bit pointer
// 只适用于x86_64四级分页的情况
// 使用高16bit作为tag避免ABA问题
template <typename T>
class Tagged_ptr {
public:
    Tagged_ptr() = default;
    Tagged_ptr(T *p, uint16_t t = 0): _ptr_tag(make(p, t)) {}
    uint16_t get_tag() { return _ptr_tag >> 48; }
    uint16_t next_tag() { return (get_tag() + 1) & TAG_MASK;}
    T* get_ptr() { return reinterpret_cast<T*>(_ptr_tag & PTR_MASK); }
    void set_ptr(T *p) { _ptr_tag = make(p, get_tag()); }
    void set_tag(uint16_t t) { _ptr_tag = make(get_ptr(), t); }
    T* operator->() { return get_ptr(); }
    T& operator*() { return *get_ptr(); }
    operator bool() { return !!get_ptr(); }
    bool operator==(T *p) { return make(get_ptr(), 0) == make(p, 0); }
    bool operator!=(T *p) { return !operator==(p); }
    bool operator==(std::nullptr_t) { return !operator bool(); }
    bool operator!=(std::nullptr_t) { return operator bool(); }
private:
    constexpr static size_t PTR_MASK = (1ull<<48)-1;
    constexpr static size_t TAG_MASK = (1ull<<16)-1;
    static uint64_t make(T *p, uint16_t t) {
        union {uint64_t x; uint16_t y[4];};
        x = uint64_t(p);
        y[3] = t;
        return x;
    }
private:
    uint64_t _ptr_tag;
};
```

 

还有放入`freelist`的元素将直接通过类型双关的技巧来直接作为链表节点使用，以减少内存分配的压力

因此，`freelist`容器的内部类`Node`节点类型只有一个`next`成员

```C++
struct Node { Tagged_ptr<Node> next; };
```



但是需要注意的是，类模板`freelist<T, Alloc>`中的元素类型`T`需要本身大小大于等于`sizeof(Node)`，因此要么是在`allocator`上动手脚，要么是`T`在内部做封装



```C++
template <typename T, typename Alloc_of_T = std::allocator<T>>
struct Wrapped_elem {
    using Padding = std::byte[64];
    union { T data; Padding _; };
    Wrapped_elem(): data() {}
    template <typename ...Args>
    Wrapped_elem(Args &&...args): data(std::forward<Args>(args)...) {}
    ~Wrapped_elem() {data.~T();}
    // allocator for wrapped_elem
    using Internel_alloc = typename std::allocator_traits<Alloc_of_T>
        ::template rebind_alloc<Wrapped_elem<T, Alloc_of_T>>;
    struct Alloc: private Internel_alloc {
        T* allocate(size_t n, const void* h = 0) {
            [](...){}(h); // unused
            auto ptr = Internel_alloc::allocate(n);
            return &ptr->data;
        }
        void deallocate(T *p, size_t n) {
            Internel_alloc::deallocate(reinterpret_cast<Wrapped_elem*>(p), n);
        }
    };
};

```



这里通过`union`的技巧保证类型大小，但是`union`本身阻止了类型`T`的RAII机制，因此需要手动触发一下

其次，封装一个`Alloc`代理类型给上层`freelist`使用，实现了基本的`allocate`和`deallocate`接口，并保持是与`T`类型交互，因此这个中间层基本不需要过多的改动`freelist`本身的实现

 

（这个`allocator`写的挺半桶水的。。。能用就行了吧大概）

 

剩下的就是一个基本的EBO套上`Alloc`，并且整个`freelist`仿造一个简单的类似`allocator`的接口规范

整体接口如下

 

```C++
template <typename T, typename Alloc = std::allocator<T>>
class Freelist: private Wrapped_elem<T, Alloc>::Alloc {
    struct Node { Tagged_ptr<Node> next; };
    // 分配T类型时实际使用的allocator
    using Custom_alloc = typename Wrapped_elem<T, Alloc>::Alloc;
public:
    Freelist();
    ~Freelist();
// 公开接口全部为线程安全实现
// 仿造std::allocator的基本接口
public:
    T* allocate();
    void deallocate(T* p);
    template <typename ...Args>
    void construct(T *p, Args &&...);
    void destroy(T *p);
private:
    std::atomic<Tagged_ptr<Node>> _pool;
};

```



 

完整代码如下，除了上述讨论过的细节，其余的算法部分就是CAS的应用

 

永久链接：[Snippets/Freelist.hpp at 55d403f52b · Caturra000/Snippets (github.com)](https://github.com/Caturra000/Snippets/blob/55d403f52b6cbd65721d44fd6999da5a35b40226/Concurrency/Freelist.hpp)

 

```C++
template <typename T, typename Alloc = std::allocator<T>>
class Freelist: private Wrapped_elem<T, Alloc>::Alloc {
    struct Node { Tagged_ptr<Node> next; };
    // 分配T类型时实际使用的allocator
    using Custom_alloc = typename Wrapped_elem<T, Alloc>::Alloc;
public:
    Freelist();
    ~Freelist();
// 公开接口全部为线程安全实现
// 仿造std::allocator的基本接口
public:
    T* allocate();
    void deallocate(T* p);
    template <typename ...Args>
    void construct(T *p, Args &&...);
    void destroy(T *p);
private:
    std::atomic<Tagged_ptr<Node>> _pool;
};
template <typename T, typename Alloc>
Freelist<T, Alloc>::Freelist()
    : _pool{nullptr}
{}
template <typename T, typename Alloc>
Freelist<T, Alloc>::~Freelist() {
    Tagged_ptr<Node> cur = _pool.load();
    while(cur) {
        auto ptr = cur.get_ptr();
        if(ptr) cur = ptr->next;
        Custom_alloc::deallocate(reinterpret_cast<T*>(ptr), 1);
    }
}
template <typename T, typename Alloc>
T* Freelist<T, Alloc>::allocate() {
    Tagged_ptr<Node> old_head = _pool.load(std::memory_order_acquire);
    for(;;) {
        if(old_head == nullptr) {
            return Custom_alloc::allocate(1);
        }
        Tagged_ptr<Node> new_head = old_head->next;
        new_head.set_tag(old_head.next_tag());
        if(_pool.compare_exchange_weak(old_head, new_head,
                std::memory_order_release, std::memory_order_relaxed)) {
            void *ptr = old_head.get_ptr();
            return reinterpret_cast<T*>(ptr);
        }
    }
}
template <typename T, typename Alloc>
void Freelist<T, Alloc>::deallocate(T *p) {
    Tagged_ptr<Node> old_head = _pool.load(std::memory_order_acquire);
    auto new_head_ptr = reinterpret_cast<Node*>(p);
    for(;;) {
        Tagged_ptr new_head {new_head_ptr, old_head.get_tag()};
        new_head->next = old_head;
        if(_pool.compare_exchange_weak(old_head, new_head,
                std::memory_order_release, std::memory_order_relaxed)) {
            return;
        }
    }
}
template <typename T, typename Alloc>
template <typename ...Args>
void Freelist<T, Alloc>::construct(T* p, Args &&...args) {
    new (p) T(std::forward<Args>(args)...);
}
template <typename T, typename Alloc>
void Freelist<T, Alloc>::destroy(T *p) {
    p->~T();
}
```



 

## 无锁stack

 

`stack`部分能讨论的地方不多，需要注意每个数据成员独占一个cache line大小（x84-64的话一般为64字节，可以看cpuinfo确认），`C++`提供了`alignas`特性直接支持对齐操作

 

接口上为了简化，只提供线程安全的版本。为了性能，理应实现非线程安全版本并应用在合适的地方

 

永久链接：[Snippets/Stack_lockfree.cpp at 55d403f52b6cbd · Caturra000/Snippets (github.com)](https://github.com/Caturra000/Snippets/blob/55d403f52b6cbd65721d44fd6999da5a35b40226/Concurrency/Stack_lockfree.cpp)

 

```C++
template <typename T, typename Alloc_of_T = std::allocator<T>>
class Stack {
public:
    struct Node {
        T data;
        Tagged_ptr<Node> next;
        template <typename ...Args> Node(Args&&...args)
            : data(std::forward<Args>(args)...), next(nullptr){}
    };
    // Stack中实际分配使用的allocator
    using Node_Alloc = typename std::allocator_traits<Alloc_of_T>
                        ::template rebind_alloc<Node>;
    using Node_ptr = Tagged_ptr<Node>;
public:
    Stack();
    ~Stack();
public:
    template <typename ...Args>
    bool push(Args &&...);
    std::optional<T> pop();
    bool empty();
private:
    // cacheline == 64 bytes
    // 将跨越2个cacheline
    alignas(64)
    std::atomic<Node_ptr> _head;
    alignas(64)
    Freelist<Node, Node_Alloc> _pool;
};
template <typename T, typename Alloc_of_T>
Stack<T, Alloc_of_T>::Stack()
    : _head(nullptr)
{}
template <typename T, typename Alloc_of_T>
Stack<T, Alloc_of_T>::~Stack() {
    while(pop());
}
template <typename T, typename Alloc_of_T>
template <typename ...Args>
bool Stack<T, Alloc_of_T>::push(Args &&...args) {
    Node *ptr = _pool.allocate();
    // 为了简化，这里并不处理异常安全
    if(!ptr) return false;
    _pool.construct(ptr, std::forward<Args>(args)...);
    Tagged_ptr<Node> old_head = _head.load(std::memory_order_acquire);
    for(;;) {
        Node_ptr new_head {ptr, old_head.get_tag()};
        new_head->next = old_head;
        if(_head.compare_exchange_weak(old_head, new_head,
                std::memory_order_release, std::memory_order_relaxed)) {
            return true;
        }
    }
}
template <typename T, typename Alloc_of_T>
std::optional<T> Stack<T, Alloc_of_T>::pop() {
    Tagged_ptr<Node> old_head = _head.load(std::memory_order_acquire);
    for(;;) {
        if(!old_head) return std::nullopt;
        Tagged_ptr<Node> new_head = old_head->next;
        new_head.set_tag(old_head.next_tag());
        if(_head.compare_exchange_weak(old_head, new_head,
                std::memory_order_release, std::memory_order_relaxed)) {
            auto opt = std::make_optional<T>(old_head->data);
            auto ptr = old_head.get_ptr();
            _pool.destroy(ptr);
            _pool.deallocate(ptr);
            return opt;
        }
    }
}
template <typename T, typename Alloc_of_T>
bool Stack<T, Alloc_of_T>::empty() {
    return !_head.load(std::memory_order_relaxed);
}
```

 

## [WIP] 无锁Queue

 

无锁`Queue`部分目前仍在进行中，代码有较多的地方不严谨，仅作记录

`Queue`实现不同于前两个容器，它需要的算法不够直观（因为涉及到`head`和`tail`两部分）

已写入到注释中

 

永久链接：[Snippets/Queue_lockfree.cpp at 55d403f52b6cbd · Caturra000/Snippets (github.com)](https://github.com/Caturra000/Snippets/blob/55d403f52b6cbd65721d44fd6999da5a35b40226/Concurrency/Queue_lockfree.cpp)

 ```C++
template <typename T, typename Alloc_of_T = std::allocator<T>>
class Queue {
public:
    struct Node;
    using Node_ptr = Tagged_ptr<Node>;
    using Requirements = std::enable_if_t<std::is_trivial_v<T> /* && TODO */>;
    struct alignas(64) Node {
        T data;
        std::atomic<Node_ptr> next;
        template <typename ...Args> Node(Args &&...args)
            : data(std::forward<Args>(args)...)
        {
            next.store(Node_ptr{nullptr}, std::memory_order_relaxed);
        }
    };
    using Node_alloc = typename std::allocator_traits<Alloc_of_T>
                        :: template rebind_alloc<Node>;
public:
    Queue();
    ~Queue();
public:
    // 在MS queue原来的算法中，push必然成功
    // 但是freelist可能会失败，因此仍保留bool接口
    template <typename ...Args>
    bool push(Args &&...);
    std::optional<T> pop();
private:
    Freelist<Node, Node_alloc> _pool;
    // 各自独占cacheline
    alignas(64) std::atomic<Node_ptr> _head;
    alignas(64) std::atomic<Node_ptr> _tail;
};
template <typename T, typename A>
Queue<T, A>::Queue() {
    Node *dummy = _pool.allocate();
    if(!dummy) throw std::bad_alloc();
    _pool.construct(dummy);
    Node_ptr tagged_dummy {dummy, 0x5a5a};
    _head.store(tagged_dummy);
    _tail.store(tagged_dummy);
}
template <typename T, typename A>
Queue<T, A>::~Queue() {
    while(pop());
}
template <typename T, typename A>
template <typename ...Args>
bool Queue<T, A>::push(Args &&...args) {
    auto node_ptr = _pool.allocate();
    if(!node_ptr) return false;
    _pool.construct(node_ptr, std::forward<Args>(args)...);
    Tagged_ptr<Node> node {node_ptr};
    // 假设Tx是当前线程，Ty是第二个线程
    // （虽然是lockfree，但算法中有实际进展的是任意2条线程）
    for(;;) {
        auto tail = _tail.load(std::memory_order_acquire);
        auto next = tail->next.load(std::memory_order_acquire);
        auto tail2 = _tail.load(std::memory_order_acquire);
        // 需要tag比较，确保tail和next是一致的
        if(tail != tail2) continue;
        // L1能确保原子性的append（需要后续L2 CAS的保证）
        // 因为MS Queue存在不断链的性质
        if(next == nullptr) {                                                   // L1
            node.set_tag(next.next_tag());
            // failed意味着有其他线程至少提前完成了L2，导致当前L1条件违反
            if(tail->next.compare_exchange_weak(next, node,
                    std::memory_order_release, std::memory_order_relaxed)) {    // L2
                node.set_tag(tail.next_tag());
                // 我认为这一步不可能failed
                // Tx执行L2成功，意味着Ty即使能执行到L1，也不能CAS到L2，
                // 因为L1的条件是整个链表中唯一存在的，
                // 而此时符合next==nullptr的node并没有发布出去（L3）
                //
                // Note: 不应使用timed lock（避免spurious failure），因此不是weak
                //
                // Note: 实际L3上可能返回false，见L4，其实是在不同线程上完成相同的工作
                _tail.compare_exchange_strong(tail, node,
                    std::memory_order_release, std::memory_order_relaxed);      // L3
                return true;
            }
        // Ty完成了L2甚至L3
        } else {
            next.set_tag(tail.next_tag());
            // _tail在MS Queue中是指向最后一个或者倒数第二个结点
            // - 指向最后一个结点就不必多说了，常规数据结构的形态
            // - 指向倒数第二个是因为存在Ty完成了L2，但是L3尚未完成
            // 
            // 如果Ty L3未完成，失败方会“推波助澜”，帮助Tx完成tail向前移动到最后一个结点的操作
            // （next如果可见了，那也是唯一确定的）
            // 这样可以提高并发吞吐
            _tail.compare_exchange_strong(tail, next,
                std::memory_order_release, std::memory_order_relaxed);          // L4
        }
    }
}
template <typename T, typename A>
std::optional<T> Queue<T, A>::pop() {
    for(;;) {
        auto head = _head.load(std::memory_order_acquire);
        auto tail = _tail.load(std::memory_order_acquire);
        auto next = head->next.load(std::memory_order_acquire);
        // 保证head tail next一致性
        auto head2 = _head.load(std::memory_order_acquire);
        if(head != head2) continue;
        if(head.get_ptr() == tail.get_ptr()) {
            // is dummy
            if(!next) return std::nullopt;
            next.set_tag(tail.next_tag());
            // Ty push进行中，Tx推波助澜
            _tail.compare_exchange_strong(tail, next,
                std::memory_order_release, std::memory_order_relaxed);
        } else {
            if(!next) continue;
            // copy
            auto opt = std::make_optional<T>(next->data);
            next.set_tag(head.next_tag());
            if(_head.compare_exchange_weak(head, next,
                    std::memory_order_release, std::memory_order_relaxed)) {
                auto old_head_ptr = head.get_ptr();
                _pool.destroy(old_head_ptr);
                _pool.deallocate(old_head_ptr);
                return opt;
            }
        }
    }
}
 ```



 

## References

 

[Chapter 20. Boost.Lockfree - 1.81.0](https://www.boost.org/doc/libs/1_81_0/doc/html/lockfree.html)

[Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms (rochester.edu)](https://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf)

[c++ - Non-POD types in lock-free data structures - Stack Overflow](https://stackoverflow.com/questions/34009485/non-pod-types-in-lock-free-data-structures)

[data structures - Lock-free queue algorithm, repeated reads for consistency - Stack Overflow](https://stackoverflow.com/questions/3873689/lock-free-queue-algorithm-repeated-reads-for-consistency)
