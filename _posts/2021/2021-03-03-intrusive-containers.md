---
layout: post
title: 浅谈侵入式容器
categories: [C++]
description: 鉴于`C++`标准库并没有提供侵入式容器供我们使用，这里只简单梳理一下侵入式容器的特性
---

鉴于`C++`标准库并没有提供侵入式容器供我们使用，这里只简单梳理一下侵入式容器的特性

<!--more-->

## 最简单的侵入式容器

先看一个最简单的示例：`list_head`

```C++
struct list_head {
    list_head *prev, *next;
};
```

这是`Linux`提供的内核链表，一个最简化的侵入式容器

可以与非侵入式`STL`风格的实现对比一下

```C++
template <typename Tp>
struct ListNode {
    ListNode *prev, *next;
    Tp value;
    // ...
};
```

相比于`STL`风格的区别，可以看出`list_head`不维护`value`，

使用的话是这种画风

```C++
struct Foo {
    int value;
    list_head link; // 指向另外两个Foo类的的list_head link
};

// 区别于
ListNode<int> bar; // 可以指向另外的&ListNode<int>
```

这就是侵入式容器的全部区别了吗？我们再琢磨一下

## 泛型支持

非侵入式的`ListNode`对于不同类型的`value`，需要提供真正意义上不一样的类型`ListNode<T>`

但是`list_head`并不需要维护多余的泛型`typename`，`list_head`就是任意类型兼容的容器

```C++
ListNode<FoOO> n1;
ListNode<Foooo> n2;
```

也许对于`C++`并没什么，但是对于不支持泛型的语言来说这种设计绝对是个优势

另外从生成的实例类型来说，也可带来编译成本的减小

PS. 并不是说所有侵入式容器就会抛弃泛型，像`boost::intrusive`套餐就还是得带上`typename`

## 空间分配

`STL`的容器都是搭上`allocator`来维护内部`element`的分配/销毁，由于`allocator`是放在`template`上的（不考虑`std::pmr`），所以造成了`allocator`注定是无状态的~~（什么是无状态，简单点说就是用了等于白用）~~，大部分情况来说，它就是一个`new`+`ctor`的解耦与抽象，问题就是`new`，不管怎样，它只有一种选择，看下面例子

```C++
template <typename Tp, typename Allocator>
struct List {
    ListNode<Tp> *mData; // 由Allocator分配
    // ...
};
```

大部分情况下，不管`List`是放在堆还是栈，`mData`分配的基本只能是在堆上，你没有选择的余地，因为`allocator`不能运行时更改，它的策略已经是编译时决定好了

相比之下，`list_head`是如何分配的，完全取决（绑定）于调用方`FOooOoO`，可以直接在栈上分配，也可以一并`new FOooOoO`放到堆上

```C++
struct FOooOoO {
    std::string name;
    int uid;
    list_head link;
};
auto pf1 = new FOooOoO(/*...*/);
FOooOoO f2(/*...*/);
```

## 分离的地址

前面提到`new`分配的问题，非侵入式容器也有缺点，它们可能会至少`new`两次

考虑`new List`过程

```C++
auto pList = new List<int>(/*size*/1, /*value*/ 100);
```

由于内部`mData`是额外分配的，因此`&pList`与`pList.mData`指向是分离的

有什么问题？`cache`警察表示这样不好，因为这样不容易利用`cache line`，再抠门一点可认为多了一次（可能有系统调用的）函数调用开销

至于`list_head`是不需要考虑这些问题，空间本身是与调用方连续的

### 补充：并非如此

难道这个缺点，非侵入永远没法克服吗？

不是的，可以看下智能指针`std::make_shared`过程如何处理这种分离的分配问题，它可以做到`Tp`和引用计数块绑定在一起连续分配，当然这很需要技巧性

## 引用计数老问题

前面提到智能指针，那就顺便提一个非侵入式智能指针的经典错误

```C++
auto pValue = new F00(/*...*/);
std::shared_ptr<F00> p1(pValue);
std::shared_ptr<F00> p2(pValue); // oops
```

原因在于引用计数块是在`std::shared_ptr`容器内部的，与`pValue`完全独立，因此会有两个引用计数块来处理同一个值引起错误的RAII

那么侵入式的智能指针会有这个问题吗？

`boost::intrusive_ptr`使用是这样的，它把引用计数与内部实现绑定在一起

```C++
struct FoOo: public boost::intrusive_ptr_base<FoOo> { // 含有引用计数ref_count
    int age;
    std::string name;
};

auto pValue = new FoOo();
boost::intrusive_ptr<FoOo> p1(pValue);
boost::intrusive_ptr<FoOo> p2(pValue); // 由同一个来自pValue的refcount来维护
```

容易看出，侵入式智能指针是通过绑定到`FoOo`内部的`refcount`来实现正确的引用计数

## 抽象与复用

侵入式容器在抽象与复用的层面上是有显然的不足的，

考虑一下，如何用`list_head`表示出`list of list of list`的数据结构，

非侵入式只需要`list<list<list<Tp>>>`就足够了

对于复用，这是最突出的问题，每个实现都要写一份才像样，侵入式容器最多能做到这种程度

```C++
class Type: public IntrusiveContainer {
    // ...
};
```

能不能给力点，也许`mixin`可以吧（用魔法打败魔法，滑稽.jpg）

```C++
template <typename ...Ts>
class Mixin: public Ts... {};
// Mixin<Type, IntrusiveContainer> fOoooo;
```

## 总结

合适的时机使用合适的容器，多一种选择是件好事。

简单来说，侵入式容器的独特设计可以带来以下特性

1. 数据结构拥有更加紧凑的内存空间，且灵活的分配与更加受控的生命周期都更易于提高性能
2. 虽然在类型表达能力与非侵入相比有所不足，但是可以与具体类型解耦
3. 更易于实现本身依赖于具体类型的特性
