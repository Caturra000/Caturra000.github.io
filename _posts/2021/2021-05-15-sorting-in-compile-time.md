---
layout: post
title: 「逐渐变态」实现编译时排序
categories: [C++]
---

由于日志库的需求，需要一个编译时排序来处理tag（模板的typename只能append!），所以尝试写了一版，工地C++选手头一回写这么一大串的template元编程
<!--more-->


## 接口/目标

注意要实现的是基于类型的排序，

就是给定各个类型不同的优先级，然后给不同的类型排序，直观上就是一个key(type)和value完全倒置的排序

假设已知各个类型`Ts...`的优先级，期望的接口是

`Sort<Ts...>::type`返回一个排好序的`Us...`

由于`type`必须是指定某一个类型，因此我使用了`std::tuple`

`Sort<Ts...>::type = std::tuple<Us...>`

```
例子：
假设，
int优先级为0
double优先级为1
std::string优先级为5

Sort<double, std::string, int>::type则可以展开为std::tuple<int, double, std::string>
```


## 需求

现在分析为了实现这个接口需要些什么东西

对比运行时排序

```C++
void selectsort(int data[], int lo, int hi) {
    for(int i = lo; i <= hi; ++i) {
        int choose = i;
        for(int j = i+1; j <= hi; ++j) {
            if(data[choose] > data[j]) choose = j;
        }
        std::swap(data[choose], data[i]);
    }
}
```

可以发现需要的东西非常朴素：

1. 把类型表示成值
2. 找出某个区间的最小值
3. 在某个区间交换两个值

下面来逐一实现

## KV

要把类型表示成值，本质上来说就是提供一个kv映射

```C++
// 类型的优先级映射如下
template <ssize_t V>
struct Value {
    static constexpr ssize_t value = V;
};

template <typename K>
struct Key {
    using type = K;
};

template <typename V>
struct Elem;

// DEMO: sort comp register
// template <>
// struct Elem<int>: Key<int>, Value<1> {};
// template <>
// struct Elem<double>: Key<double>, Value<2> {};
// template <>
// struct Elem<char>: Key<char>, Value<3> {};
// template <>
// struct Elem<float>: Key<float>, Value<4> {};
// template <>
// struct Elem<unsigned>: Key<unsigned>, Value<5> {};
```

实现了kv映射后就可以用来求出所谓最小的类型了

## Min

求`min`的过程使用经典的递归处理，为了方便就直接把`value`和`type`都求出来了

```C++
template <typename ...Ts>
struct Min {
    constexpr static ssize_t value = MinValue<Ts...>::value;
    using type = typename MinType<Ts...>::type;
};

template <typename T, typename ...Ts>
struct MinValue {
    constexpr static ssize_t value = std::min(Elem<T>::value, MinValue<Ts...>::value);
};
template <typename T>
struct MinValue<T> {
    constexpr static ssize_t value = Elem<T>::value;
};

template <typename T, typename ...Ts>
struct MinType {
    using type = std::conditional_t<
        /*if  */Elem<T>::value <= MinValue<Ts...>::value, 
        /*then*/T,
        /*else*/typename MinType<Ts...>::type>;
};
template <typename T>
struct MinType<T> {
    using type = T;
};
```

## 区间操作

需求三比较蛋疼，类型并没有`swap`或者赋值这种概念

一种间接的方法是，基于`std::tuple`，实现一系列的区间操作，比如把区间掰开以及拼接

具体的，假设需要交换`std::tuple<>`中下标为$L$,$R$的类型，

就是要把子区间$[0,L), L, (L, R), R, (R, sizeof...)$分别拆开

最后依次拼接$[0,L), R, (L, R), L, (R, sizeof...)$即可

首先来看拼接`Concat`，实现非常简洁

```C++
template <typename ...Ts> // 既匹配一个T，也匹配两个T,U
struct Concat;

// input: Concat< std::tuple<L....>, std::tuple<R....> >
// output: type -> std::tuple<L..., R...>
template <typename ...L, typename ...R>
struct Concat<std::tuple<L...>, std::tuple<R...>> {
    using type = std::tuple<L..., R...>;
};
// std::tuple<T> => type->std::tuple<T>
template <typename T>
struct Concat<std::tuple<T>> {
    using type = std::tuple<T>;
};
```



还有`Select`，表示从区间中获取子区间$[L,R]$

（别吐槽命名了，本质就是同一个`Select`过程，做一个分层）

```C++
// 从Ts...中返回[L,R]的子区间，用`std::tuple`封装
template <ssize_t L, ssize_t R, typename ...Ts>
struct Select {
    using type = typename Select1<L, R, 0, Ts...>::type;
    using check = std::enable_if_t<L <= R && L >= 0 && R <= sizeof...(Ts)>;
};

// Select1引入当前下标`Cur`，用于定位
// Select2用于直接获取类型
// Select1区分Start == Cur 和 不等于的处理情况
// 找到Start == Cur的定位后，执行Select2，表示从当前位置找出连续End-start+1个Type
// 由于过程中返回`std::tuple`，因此需要复用前面的`Concat`来拼接

// not equal
template <ssize_t Start, ssize_t End, ssize_t Cur, typename T, typename ...Ts>
struct Select1 {
    using type = typename Select1<Start, End, Cur+1, Ts...>::type;
};

// start == cur
template <ssize_t Start, ssize_t End, typename T, typename ...Ts>
struct Select1<Start, End, Start, T, Ts...> {
    using type = typename Select2<End-Start+1, T, Ts...>::type; // MOCK
};

template <ssize_t N, typename T, typename ...Ts>
struct Select2 {
    using type = typename Concat<std::tuple<T>, typename Select2<N-1, Ts...>::type >::type;
};
template <typename T, typename ...Ts>
struct Select2<0, T, Ts...>;
template <typename T, typename ...Ts>
struct Select2<1, T, Ts...> {
    using type = std::tuple<T>;
};
template <typename T>
struct Select2<1, T> {
    using type = std::tuple<T>;
};
```

顺便，`Select`需要用到`L, R`这些下表是需要定位的，就是找出位置，因此加了一个`Find`

```C++
// T表示要找出对应下标的类型
template <typename T, typename U, typename ...Ts>
struct Find {
    constexpr static ssize_t value = Find1<0, T, U, Ts...>::value;
};

// 还没找到的时候
template <ssize_t Cur, typename T, typename U, typename ...Ts>
struct Find1 {
    constexpr static ssize_t value = Find1<Cur+1, T, Ts...>::value;
};

// 找到的时候
template <ssize_t Cur, typename T, typename ...Ts>
struct Find1<Cur, T, T, Ts...> {
    constexpr static ssize_t value = Cur;
};
```

## Sort

终于，前面写了一大坨的，我们可以用上了

这里需要注意的是，编译时是不允许“越界”的，

运行时判断越界只需要`if-else`判定下标范围，而在编译时的环境下，是直接编译报错的，毕竟模板没法展开实例化

因此，一个方法就是通过重载决议，把各种边界条件都对上一个（偏）特化的类模板即可

```C++
template <typename ...Ts>
struct Sort {
    using type = typename Sort1<std::tuple<Ts...>>::type;
};

template <typename T, typename ...Ts>
struct Sort1;

template <typename T>
struct Sort1<std::tuple<T>> {
    using type = std::tuple<T>;
};

template <typename T, typename ...Ts>
struct Sort1<std::tuple<T, Ts...>> {
    using type = typename Sort2<
        Find<typename Min<T, Ts...>::type, T, Ts...>::value,
        sizeof...(Ts),
        T, Ts...
    >::type;
};

// minpos mid
// no std::tuple
template <ssize_t MinPos, ssize_t MaxIdx, typename T, typename ...Ts>
struct Sort2 {
    using check = std::enable_if_t<MinPos+1 <= MaxIdx && MinPos-1 >= 0>;
    using type = typename Concat<
        typename Select<MinPos, MinPos, T, Ts...>::type,
        typename Sort1<
            typename Concat<
                typename Select<0, MinPos-1, T, Ts...>::type,
                typename Select<MinPos+1, MaxIdx, T, Ts...>::type
            >::type
        >::type
    >::type;
};

// minpos == 0
template <ssize_t MaxIdx, typename T, typename ...Ts>
struct Sort2<0, MaxIdx, T, Ts...> {
    using type = typename Concat<
        std::tuple<T>,
        typename Sort1<typename Select<1, MaxIdx, T, Ts...>::type>::type
    >::type;
};

// minpos == end
template <ssize_t MaxIdx, typename T, typename ...Ts>
struct Sort2<MaxIdx, MaxIdx, T, Ts...> {
    using type = typename Concat<
        typename Select<MaxIdx, MaxIdx, T, Ts...>::type,
        typename Sort1<typename Select<0, MaxIdx-1, T, Ts...>::type>::type
    >::type;
};
```

## 完整代码

见：[dLog/msort.h at master · Caturra000/dLog (github.com)](https://github.com/Caturra000/dLog/blob/master/src/msort.h)
