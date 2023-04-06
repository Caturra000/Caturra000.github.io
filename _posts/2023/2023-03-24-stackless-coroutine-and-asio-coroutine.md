---
layout: post
title: 从switch-case飞线，到无栈协程和asio协程的实现
categories: [C++, RTFSC]
---

<!--more-->


## 前言

 

最近在翻asio内部有啥好东西，看到很有意思的代码就顺手做下笔记

这篇文章简单描述asio怎样实现一个无栈协程

 

## switch-case基础

 

首先，[switch-case](https://en.cppreference.com/w/cpp/language/switch)语法应该是C语言中的基础；其次，[goto](https://en.cppreference.com/w/cpp/language/goto)语法也是C语言中的基础。`goto`后面衔接的是`label`，而`switch`语法中的`case`也是一种`label`

那意味着什么？意味着可以[飞线](https://zh.wikipedia.org/wiki/飞线)

 

```C++
#include <iostream>
using std::cin;
using std::cout;
using std::endl;
int main() {
    int a;
    cin >> a;
    switch (a) case 0:
        for(; a < 2;) {
        cout<<"hello"<<endl;
        case 2: return 0;
        case 1:
        cout<<"world"<<endl;
    }
}
// >> 0
// hello
// [return]
//
// >> 1
// world
// hello
// [return]
//
// >> 2
// [return]

```



 

 

这段代码你可以观察以下不同的变量输入会产生什么结果

总之，这里case越过了代码块的限制，也不影响原有实现的执行，只是简单的case match，然后fall through

 

## 达夫设备

 

这种反常的特性有一个著名的应用就是Duff's device

用以手动循环展开优化性能

虽然跟后续的话题关联不大，但你可以更加深刻的理解飞线能做到什么程度

 

```C++
#include <iostream>
// 使用case来完成duff's device
void private_memcpy(char *to, const char *from, size_t count) {
    size_t n = (count + 7) / 8;
    #define COPY *to = *from;
    #define ADVANCE to++, from++;
    switch (count % 8) {
    case 0: do { COPY ADVANCE
    case 7:      COPY ADVANCE
    case 6:      COPY ADVANCE
    case 5:      COPY ADVANCE
    case 4:      COPY ADVANCE
    case 3:      COPY ADVANCE
    case 2:      COPY ADVANCE
    case 1:      COPY ADVANCE
            } while (--n > 0);
    }
    #undef COPY
    #undef ADVANCE
}

int main() {
    const char from[] = "jintianxiaomidaobilema";
    char to[sizeof from];
    private_memcpy(to, from, sizeof from);
    std::cout << to << std::endl;
    return 0;
}
```



 

## 协程基础

 

广告：[实现一个简单的协程 – Caturra's blog](/posts/implements-coroutine/)

我在之前写过一篇实现协程轮子（有栈协程）的文章，里面有一些关于协程的基本概念，各位客官对协程不太熟悉的话可以看看

 

## 无栈协程

 

前面提到的有栈协程是从调用栈的角度去考虑的

大概意思是：传统意义上的函数调用是怎样的，我协程也尝试去模仿、改造、封装

但是无栈协程直接是抛弃caller-callee这些概念，从状态转移的角度去入手

 

比如一个写法层面蠢到爆的例子

 ```C++
int fib() {
    static int state_machine {0};
    switch(state_machine) {
        case 0:
            state_machine = 1;
            return 1;
        case 1:
            state_machine = 2;
            return 1;
        case 2:
            state_machine = 3;
            return 2;
        case 3:
            state_machine = 4;
            return 3;
        case 4:
            state_machine = 5;
            return 5;
        case 5:
            state_machine = 6;
            return 8;
        case 6:
            state_machine = 7;
            return 13;
        case 7:
            state_machine = -1;
            return 21;
        default:;
    }
    return -1;
}
 ```



这是一个求斐波那契数列的**协程**——一个函数，可以拆分其控制流，每次返回后再次调用时，他都可以从保存的上下文中恢复

所以是名副其实的协程（虽然算到21后就不能再次重入了）

 

我们对这些过程稍加修饰

 ```C++
#include <iostream>
/// holy shit!
#define CO_BEGIN static int _state = 0; switch(_state) { case 0:
#define CO_YIELD(ret) do {_state = __LINE__; return ret; case __LINE__:;} while(0)
#define CO_END _state = -1; default:;}

int fib() {
    CO_BEGIN
    CO_YIELD(1);
    CO_YIELD(1);
    CO_YIELD(2);
    CO_YIELD(3);
    CO_YIELD(5);
    CO_YIELD(8);
    CO_YIELD(13);
    CO_YIELD(21);
    CO_END
    return -1;
}

int main() {
    for(int ret; (ret = fib()) != -1;) {
        std::cout << ret << std::endl;
    }
    return 0;
}
 ```



 

进一步的，可以继续提供`CO_RETURN`语义，可以直接终止协程并不可再次重入

 ```C++
#include <iostream>
#include <cassert>
/// holy shit!!
#define CO_VAR static
#define CO_BEGIN static int _state = 0; switch(_state) { case 0:
#define CO_YIELD(ret) do {_state = __LINE__; return ret; case __LINE__:;} while(0)
#define CO_RETURN(ret) do {_state = -1; return ret;} while(0)
#define CO_END _state = -1; default:;}

static constexpr int NONE = 0;
// acts like a pipe
static int g_pipe {NONE};

using std::cout;
using std::endl;

int producer() {
    CO_VAR int i;
    CO_BEGIN
    for(i = NONE + 1; i < NONE + 10; ++i) {
        g_pipe = i;
        cout << "[producer] generates: " << i << endl;
        CO_YIELD(i);
    }
    CO_END
    return g_pipe = EOF;
}

void consumer() {
    CO_BEGIN
    for(;;) {
        if(g_pipe == NONE) {
            cout << "[consumer] none, yield." << endl;
            CO_YIELD();
            cout << "[consumer] wakeups and checks again." << endl;
        }
        if(g_pipe == EOF) {
            cout << "[consumer] end-of-file, return." << endl;
            CO_RETURN();
            // unreachable
            cout << "[wtf] jintianxiaomidaobila" << endl;
        }
        cout << "[consumer] consumes:  " << g_pipe << endl;
        g_pipe = 0;
    }
    CO_END
    cout << "cannot consume" << endl;
}

int main() {
    while(~g_pipe) {
        int debug = producer();
        assert(debug == g_pipe);
        consumer();
    }
    cout << "==================" << endl;
    for(auto _{3}; _--;) {
        consumer();
    }
    return 0;
}

// [producer] generates: 1
// [consumer] consumes:  1
// [consumer] none, yield.
// [producer] generates: 2
// [consumer] wakeups and checks again.
// [consumer] consumes:  2
// [consumer] none, yield.
// [producer] generates: 3
// [consumer] wakeups and checks again.
// [consumer] consumes:  3
// [consumer] none, yield.
// [producer] generates: 4
// [consumer] wakeups and checks again.
// [consumer] consumes:  4
// [consumer] none, yield.
// [producer] generates: 5
// [consumer] wakeups and checks again.
// [consumer] consumes:  5
// [consumer] none, yield.
// [producer] generates: 6
// [consumer] wakeups and checks again.
// [consumer] consumes:  6
// [consumer] none, yield.
// [producer] generates: 7
// [consumer] wakeups and checks again.
// [consumer] consumes:  7
// [consumer] none, yield.
// [producer] generates: 8
// [consumer] wakeups and checks again.
// [consumer] consumes:  8
// [consumer] none, yield.
// [producer] generates: 9
// [consumer] wakeups and checks again.
// [consumer] consumes:  9
// [consumer] none, yield.
// [consumer] wakeups and checks again.
// [consumer] end-of-file, return.
// ==================
// cannot consume
// cannot consume
// cannot consume
 ```



 

这个生产者-消费者示例中双方通过管道来传递信息。如果消费者暂时不能获取到信息，则切出协程；控制流交接到主程序再继续切入到生产者去构造信息；生产者一旦完成单次生产操作，也把自身切出给主程序，并进一步调度给曾经切出的消费者从被中断的地方继续执行，直到管道失效，直接终止消费行为，不可再次重入

 

你也可以在这里看出飞线的特性，这里的协程语义都可以嵌入到任何语句当中

且这种无栈协程的实现很短，尽管看着很糟糕的写法，但它不涉及具体的汇编，因此天然有良好的跨平台特性

除此以外，运行时开销（编译器友好）和空间开销（仅一个`int`）更是远胜于有栈协程

（我认为这点是`C++20`采用无栈协程作为标准的重要依据）

 

它自然是有缺点的，比如对局部变量和可重入的要求会比有栈协程更加苛刻，换句话说就是应用场合很局限，你需要在合适的时机使用

 

## asio协程

 

无栈协程既然有这么突出的优势，且适用于严格性能要求的异步IO场合，因此`asio`（在`C++20`落地十年前）也实现了一版无栈协程

当然一个良好的习惯是，看代码前先看文档：[coroutine (think-async.com)](https://think-async.com/Asio/asio-1.18.1/doc/asio/reference/coroutine.html)

 

这里假定你已经阅读过文档。我们继续探讨它的代码实现，同样有上述飞线做法的优势，就是代码非常的短小

 

接口部分：

 

```C++
#ifndef reenter
# define reenter(c) ASIO_CORO_REENTER(c)
#endif
#ifndef yield
# define yield ASIO_CORO_YIELD
#endif
#ifndef fork
# define fork ASIO_CORO_FORK
#endif
```



 

实现部分：

```C++
class coroutine
{
public:
  /// Constructs a coroutine in its initial state.
  coroutine() : value_(0) {}
  /// Returns true if the coroutine is the child of a fork.
  bool is_child() const { return value_ < 0; }
  /// Returns true if the coroutine is the parent of a fork.
  bool is_parent() const { return !is_child(); }
  /// Returns true if the coroutine has reached its terminal state.
  bool is_complete() const { return value_ == -1; }
private:
  friend class detail::coroutine_ref;
  int value_;
};

namespace detail {
class coroutine_ref
{
public:
  coroutine_ref(coroutine& c) : value_(c.value_), modified_(false) {}
  coroutine_ref(coroutine* c) : value_(c->value_), modified_(false) {}
  ~coroutine_ref() { if (!modified_) value_ = -1; }
  operator int() const { return value_; }
  int& operator=(int v) { modified_ = true; return value_ = v; }
private:
  void operator=(const coroutine_ref&);
  int& value_;
  bool modified_;
};
} // namespace detail
} // namespace asio

#define ASIO_CORO_REENTER(c) \
  switch (::asio::detail::coroutine_ref _coro_value = c) \
    case -1: if (_coro_value) \
    { \
      goto terminate_coroutine; \
      terminate_coroutine: \
      _coro_value = -1; \
      goto bail_out_of_coroutine; \
      bail_out_of_coroutine: \
      break; \
    } \
    else /* fall-through */ case 0:

#define ASIO_CORO_YIELD_IMPL(n) \
  for (_coro_value = (n);;) \
    if (_coro_value == 0) \
    { \
      case (n): ; \
      break; \
    } \
    else \
      switch (_coro_value ? 0 : 1) \
        for (;;) \
          /* fall-through */ case -1: if (_coro_value) \
            goto terminate_coroutine; \
          else for (;;) \
            /* fall-through */ case 1: if (_coro_value) \
              goto bail_out_of_coroutine; \
            else /* fall-through */ case 0:

#define ASIO_CORO_FORK_IMPL(n) \
  for (_coro_value = -(n);; _coro_value = (n)) \
    if (_coro_value == (n)) \
    { \
      case -(n): ; \
      break; \
    } \
    else

#if defined(_MSC_VER)
# define ASIO_CORO_YIELD ASIO_CORO_YIELD_IMPL(__COUNTER__ + 1)
# define ASIO_CORO_FORK ASIO_CORO_FORK_IMPL(__COUNTER__ + 1)
#else // defined(_MSC_VER)
# define ASIO_CORO_YIELD ASIO_CORO_YIELD_IMPL(__LINE__)
# define ASIO_CORO_FORK ASIO_CORO_FORK_IMPL(__LINE__)
#endif // defined(_MSC_VER)
```



 

### overview

`asio`协程可分为三个部分：

- `coroutine`协程类
- `coroutine_ref`代理类
- 接口实现


协程类`coroutine`唯一保存的是上下文状态机`value`，而它只能被代理类`coroutine_ref`所修改，状态机初始化为0，会在每一种`yield`初次执行前被修改为特定的状态；除此以外，代理类还有局部私有的`modified`标记，用以区分协程的生命周期，这些部分会在后续接口实现上有更深入的理解

 

### reenter

 

我们首先品鉴`reenter`的实现

 

```C++
#define ASIO_CORO_REENTER(c) \
  switch (::asio::detail::coroutine_ref _coro_value = c) \
    /* 当结束的协程再次调用时（-1会在下面terminate_coroutine赋值），会进入到这里 */ \
    /* 此时也应该满足if语句，因此terminate是无法重入的 */ \
    case -1: if (_coro_value) \
    { \
      goto terminate_coroutine; \
      /* 这个label也会在yield用到 */ \
      terminate_coroutine: \
      /* 这个协程已经结束 */ \
      _coro_value = -1; \
      goto bail_out_of_coroutine; \
      /* 这个label也会在yield用到 */ \
      bail_out_of_coroutine: \
      /* 跳出整个协程block */ \
      break; \
    } \
    /* 正常路径 */ \
    else /* fall-through */ case 0:
```



 

`reenter`描述的是一个协程的入口，整个`reenter(...) { /*code block*/ }`代码块区间都是协程的执行范围；

如果一个协程已经终止（提前`return`，`yield break`，`throw exception`，或者正常走到作用域的末尾），那么你没办法再进入该协程，会跳出这个协程区间，这部分会在`yield`有更详细的分析

而一般情况，一个未曾执行过的协程则会匹配到`case 0`，或者未终止的协程则会匹配到`case __LINE__`

 

这里需要结合一下代理类的实现

 

```C++
class coroutine_ref
{
public:
  coroutine_ref(coroutine& c) : value_(c.value_), modified_(false) {}
  coroutine_ref(coroutine* c) : value_(c->value_), modified_(false) {}
  // 如果是经过了yield，那么不会触发（肯定modifed，见下面operator=的解释）
  // 除非是这个协程内部还没有yield就直接在reenter block内return、抛出异常或者一路走到reenter block尾部等行为
  // 这些情况下，协程都不再是可重入的
  ~coroutine_ref() { if (!modified_) value_ = -1; }
  operator int() const { return value_; }
  // 每次拷贝ref时，既设为modified
  // 注意，构造时不会这样（modified = false)
  // 拷贝行为发生在yield语义上，在##flag #1处
  int& operator=(int v) { modified_ = true; return value_ = v; }
private:
  void operator=(const coroutine_ref&);
  int& value_;
  bool modified_;
};

```



 

代理类会在`switch`的范围结尾析构，会通过私有的`modified_`成员来区分协程本身的生命周期，当设置`value_`为-1时，将不可再次重入，这些部分可以通过下述`yield`来

 

 

### yield

 

接着是`yield`实现

 

```C++
#define ASIO_CORO_YIELD_IMPL(n) \
  /* cr的状态机引用value被赋值为新的状态 */ \
  /* ##flag #1 */ \
  for (_coro_value = (n);;) \
    /* 正常并不会走到这个if吧？ */ \
    if (_coro_value == 0) \
    { \
      /* 对于保存过状态n的情况下有效，下一次reenter就从这里开始 */ \
      /* 应该是执行yield下一行去了（忽略##flag #4单行） */ \
      case (n): ; \
      break; \
    } \
    else \
      /*第一次走到yield执行时，cr已经赋值为n，既case 0，走##flag #4 */ \
      switch (_coro_value ? 0 : 1) \
        /* ##flag #2 */ \
        for (;;) \
          /* fall-through */ case -1: if (_coro_value) \
            goto terminate_coroutine; \
          /* ##flag #3 */ \
          else for (;;) \
            /* fall-through */ case 1: if (_coro_value) \
              goto bail_out_of_coroutine; \
              /* case 0 接上用户代码 */ \
              /* ##flag #4 */ \
            else /* fall-through */ case 0:
            /* 如果是yield statement */
            /*   执行完用户代码后，再回到##flag #3处的for */
            /*   从而进入bail_out_of_coroutine，跳出yield block，直到下次reenter */
            /* 如果是yield return statement，那应该block下面都无法执行了（要给出return结果到caller），等待下一次reenter */
            /* 如果是yield break，则跳到##flag #2，fallthrough到-1，终止整个协程 */
```



 

在我实现的超简陋版无栈协程中，你可以总结出一个`yield`的套路（好吧我只是复述一下`#define CO_YIELD(ret)`）：

- 保存`state = $LINE`
- `return`
- `case $LINE:`

（注：这里`$LINE`认为是同一个值）

因此这里`asio`同样也是先保存一个`state`，紧接着走到`##flag #4`处执行用户提供的代码

注意这里`asio`很细心的定制了不同的`yield`行为：

- `yield statement`
- `yield return statement`
- `yield break`

所以后续的`else`有复杂的嵌套就是为了处理这些行为：

- 如果是`yield statement`，将会让出，直到下次重入再恢复上下文
- 如果是`yield return statement`，将会返回结果，直到下次重入再恢复上下文
- 如果是`yield break`，将会让出，且不可重入

 

再次说明一下代理类，这里`yield`的赋值过程表示已经是`modified`

 

### fork

 

`asio`还做了一个`fork`语义生成子协程

```C++
#define ASIO_CORO_FORK_IMPL(n) \
  for (_coro_value = -(n);; _coro_value = (n)) \
    if (_coro_value == (n)) \
    { \
      case -(n): ; \
      break; \
    } \
    else
```



其实就是一个只执行2次的`for`循环，用负数来表示`child`

## References

[Coroutines in C](https://www.chiark.greenend.org.uk/~sgtatham/coroutines.html)

[asio无栈协程 - chnmagnus](https://www.jianshu.com/p/0395a068e8df)

[coroutine (think-async.com)](https://think-async.com/Asio/asio-1.18.1/doc/asio/reference/coroutine.html)

[coroutine.hpp at 1408e28 · chriskohlhoff/asio · GitHub](https://github.com/chriskohlhoff/asio/blob/1408e2895c94c8e254e9e8ddd66ba083777f0dc2/asio/include/asio/coroutine.hpp)
