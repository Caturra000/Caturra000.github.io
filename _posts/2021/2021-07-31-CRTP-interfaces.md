---
layout: post
title: 使用CRTP实现编译期接口定义
categories: [C++]
description: 使用CRTP实现编译期接口定义
---

> `CRTP`：吾与城北`virtual`孰美？

<!--more-->

## 前言

有一种类型叫`pure virtual`，不仅多态，而且强制要求`Base`类不做实现（子类必须override），只声明为接口，实现了接口语义

但有时候并不需要多余的多态，我就需要一个类做接口声明

`C++`作为追求zero overhead的语言，能不能做到完全不必承担`virtual`开销的接口语义？

那当然可以，只要你能折腾

PS. 全文废话非常多（如果不愿意看这个东西是怎么来的），解决方案只在正文最后一段

## 期望

期望自然是做到接口语义，伪代码如下

```C++
class Interface {
    DEFINE int func();
    DEFINE void func2(int val);
};

class Impl: public Interface {
    // 不实现func和func2，你就得死
};
```

## 初探

既然是编译时，又是继承，还是接口，很容易想到用`CRTP`惯用法

为了简单起见，目前接口只有`void func()`

```C++
template <typename Derived>
class Interface {
private:
    void __require() {
        auto that = static_cast<Derived*>(this);
        that->func();
    }
};
```

然后随便写一个测试

```C++
// Impl "is-an" Interface
class Impl: public Interface<Impl> {}; // 不写实现，直观上应该编译不了

int main() {
    Impl test;
    return 0;
}
```

直观上来说，由于`Impl`继承了奇异递归的`Interface`，

而`Interface`声明的`__require`中调用了子类`Impl`的`func`，

目前显然是没有`func`符号，编译期就会无法通过，起到接口约束的作用

要不编译试试？

## 坑1：模板

很不幸，这段代码是完全可以编译的，这个所谓的`CRTP`完全没有实现接口语义的作用

那是因为踩了__模板实例化__的坑：你不用，编译期间就不会实例化，哪怕你调用了`Interface<Impl>`，由于没有任何一处调用到`__require()`，那么`__require()`部分就不会实例化

也就是说，你需要在每一个子类中显式地调用到`__require()`才能起到约束作用

```C++
template <typename Derived>
class Interface {
protected: // 需要开放权限为protected
    void __require() {
        auto that = static_cast<Derived*>(this);
        that->func();
    }
};

class Impl: public Interface<Impl> {
public:
    void func() {}
private:
    void checkInterface() { // 需要添加一个丑陋的函数
        Interface<Impl>::__require();
    }
};

int main() {
    Impl test;
    return 0;
}
```

开放权限不是大事，但是每一个子类都要加一个极具侵入性的函数是非常丑陋的行为

能不能再改进一点？我不希望子类有任何额外的添加

## 构造既检查

其实思路很简单，把检查的流程写到接口类的构造函数即可

```C++
template <typename Derived>
class Interface {
public:
    Interface() { __require(); }
private: // 回归到private了
    void __require() {
        auto that = static_cast<Derived*>(this);
        that->func();
    }
};

class Impl: public Interface<Impl> {
public:
    void func() {}
};

```

实现类一下子清爽多了！而且做到了接口约束的效果

## 坑2：副作用

上面这段代码忽略的一个很大问题是调用`that->func()`是有__副作用__的

1. 不仅有运行时开销
2. 而且可能会导致整个流程在逻辑上是错误的

怎么解决？

最简单的方法是效仿`linux kernel`里的小技巧：`sizeof`是编译阶段的求值，并不会真的执行

```C++
template <typename Derived>
class Interface {
public:
    Interface() { __require(); }
private:
    void __require() {
        auto that = static_cast<Derived*>(this);
        sizeof (that->func()); // 严格确保是编译时的行为
    }
};
```

但是仍有不足之处：

1. 代码丑陋
2. `sizeof(void)`并不是特别合法的行为

既然如此那就用稍微`modern C++`的做法：`decltype`

```C++
template <typename Derived>
class Interface {
public:
    Interface() { __require(); }
private:
    void __require() {
        auto that = static_cast<Derived*>(this);
        decltype(that->func()) * _ { }; // 一段只有谜语人才敢写的玩意
    }
};
```

这里的意思是说

1. 任何类型都有它的指针形式
2. 任何指针都能赋值`nullptr`
3. `decltype`同样只是编译时推导类型

虽然代码有点令人不适，但它确实是类型安全的

但这代码确实是令人不适，那我们还能换种形式

```C++
template <typename Derived>
class Interface {
public:
    Interface() { __require(); }
private:
    void __require() {
        auto that = static_cast<Derived*>(this);
        using RequireFunc = decltype(that->func()); // 稍微体面一点
    }
};
```

那么，这样就能结束了？一个完美的接口类型？

并没有

## 坑3：参数类型

假设现在要求实现另一个接口

```C++
int func2(int, double);
```

它现在有两个问题

1. 怎么在`decltype`中表达出`int`和`double`参数
2. 怎么得知返回类型真的是`int`

先来解决2：使用`std::is_same<>`再一次判断

再来解决1：反正是编译时推导的东西，可以通过默认构造函数解决，能塞进去就算成功

总结如下

```C++
template <typename Derived>
class Interface {
public:
    Interface() { __require(); }
private:
    void __require() {
        auto that = static_cast<Derived*>(this);

        using RequireFunc = decltype(that->func());
        static_assert(std::is_same<RequireFunc, void>::value, "please check func");

        // 使用默认构造函数，这种统一的方式可以把整个过程封装为template
        // 使用STL提供的类型萃取轻松完成返回值类型判断
        using RequireFunc2 = decltype(that->func2(int{}, double{}));
        static_assert(std::is_same<RequireFunc2, int>::value, "please check func2");
    }
};

class Impl: public Interface<Impl> {
public:
    void func() {}
    int func2(int, double) { return 1; }
};
```

似乎又解决了一个难题？还没完！
假设现在要求实现另外一个接口

```C++
void func3(NoDefaultCtor);

// NoDefaultCtor的定义
struct NoDefaultCtor {
    NoDefaultCtor() = delete;
    NoDefaultCtor(int) {}
};
```

前面那种基于默认构造函数的做法是无效的

这不是只要传入一个`NoDefaultCtor(1)`就能解决的问题

它导致的更严重的后果是：无法使用`template`来封装整个静态检查过程

一个类被限制了只能用某一种特定的构造函数的情况，是非常难通过模板来推导的

## 模板参数推导

既然类型是个坎，那就得正面从类型去考虑，不要停留在值或者对象的角度

对类型最有表达能力的方式自然是模板的推导

已知：给出函数，通过推导可以得出它的返回类型和参数

怎么做的：见[sRpc/FunctionTraits.h at master · Caturra000/sRpc (github.com)](https://github.com/Caturra000/sRpc/blob/master/src/FunctionTraits.h)

我在上述链接实现了一个简单的`FunctionTraits`，可以推导出各种不同的函数（普通函数、成员函数、`lambda`、`std::function`）的元信息

另一方面：希望通过注册的形式来完成接口检查，由使用方主动告知函数需要的信息，然后对比一下是否匹配（`IsSameInterface`）即可

```C++
template <typename T>
struct FunctionTraits;

// 用于推导函数的meta-data，包括返回值和函数参数
template <typename Ret, typename ...Args>
struct FunctionTraits<Ret(Args...)> {
    constexpr static size_t ArgsSize = sizeof...(Args);
    using ReturnType = Ret;
    using ArgsTuple = std::tuple<Args...>;
};

// 针对成员函数的偏特化
template <typename Ret, typename C, typename ...Args>
struct FunctionTraits<Ret(C::*)(Args...)>: public FunctionTraits<Ret(Args...)> {};

// 这里的意思是说，如果返回值和函数参数完全一致，那它是meta-data匹配了
// 这里FT是FunctionTraits的实例化
// 此时我们不知道双方的function的名字是什么，需要在Interface类中主动提供
template <typename FT1, typename FT2>
struct IsSameInterface {
    constexpr static bool value =
        std::is_same<typename FT1::ReturnType, typename FT2::ReturnType>::value
        && std::is_same<typename FT1::ArgsTuple, typename FT2::ArgsTuple>::value;
};

struct NoDefaultCtor {
    NoDefaultCtor() = delete;
    NoDefaultCtor(int) {}
};

template <typename Derived>
class Interface {
public:
    Interface() { __require(); }
private:
    void __require() {
        constexpr bool check = IsSameInterface<
            FunctionTraits<decltype(&Interface::_func3)>,
            FunctionTraits<decltype(&Derived::func3)>>
        ::value;
        static_assert(check, "check func3");
    }

    // 注册一个主动提供的信息，告知func3需要什么样的返回值和参数
    void _func3(NoDefaultCtor);
};


class Impl: public Interface<Impl> {
public:
    void func3(NoDefaultCtor) {};
};
```

从这里我们已经到了使用模板的方案，除了显得繁琐

但这个可以用宏来缓解一下`__require`流程，毕竟过程是很单调重复的（详略）

现在，前面的问题全部都解决了，通过当前的模板甚至能扩展到处理左右值、`const`语义等细节，已经很能打了

## 坑4：死于重载

在上述模板方案中，有一点是模板无法解决的（或者很难处理）

对于`FunctionTraits<decltype(&Interface::_func3)>`这种形式的调用，模板无法区分`_func3`到底是什么

比如要求实现含有重载的接口

```C++
void func3(NoDefaultCtor);
void func3(int);
```

这时候就无法解析了，很显然，因为存在二义性

## 用重载解决重载

这里提供一种新的处理思路：

如果给出成员函数指针`ReturnType Class::*fptr`

只要有对应的函数参数列表，那么`fptr`都是能匹配上的

相比上述的解决方案，这里能匹配的原因是多提供了参数的信息，这是基本语法就提供的特性，不需要额外的模板推导

因此换一种方式：不是从函数推导出参数，而是提供参数和利用上述重载特性找出匹配函数

这里重新定义一套`require-define`的步骤，如下所示

```C++
template <typename Ret, typename ...Args>
struct Require;

template <typename Ret, typename ...Args>
struct Require<Ret(Args...)> {
    template <typename C>
    constexpr static void define(Ret (C::*fp)(Args...)) {}
};

struct NoDefaultCtor {
    NoDefaultCtor() = delete;
    NoDefaultCtor(int) {}
};

template <typename Derived>
class Interface {
public:
    constexpr Interface() { __require(this); }

private:
    // 加上Interface参数防止潜在签名冲突（虽然没多大必要）
    constexpr void __require(Interface*) {
        Require<void(int, double&)>::define(&Derived::func);
        Require<void(int, double)>::define(&Derived::func);
        Require<long()>::define(&Derived::func);
        Require<int()>::define(&Derived::func2);
        Require<int(const NoDefaultCtor&)>::define(&Derived::func2);
    }
};

// Impl "is-an" Interface
class Impl: public Interface<Impl> {
public:
    void func(int, double&) {}
    void func(int, double) {}
    long func() { return 1; }
    int func2() { return 1; }
    int func2(const NoDefaultCtor&) { return 1; }
};
```

是不是非常简洁明了？只要`define`匹配上的，那么接口就是合法

同样的，它处理了前面所有的问题

## THE END

至此，总算是折腾出了一套CRTP下的静态接口，总结有以下优点：

1. 没有`virtual`开销，完全编译时处理
2. 耦合度非常低，子类只有继承接口，无需额外操作
3. 底层实现简洁且紧凑，它并没有多少代码
