---
layout: post
title: 像位运算一样构造tuple
categories: [C++]
---

<!--more-->

## 前言

使用位运算写`flag`是一种很直观优雅的做法

比如`flag = flagA | flagB | flagC`

这里`flag`是一个简单的`bitmap`，一般用`unsigned int`的简单类型表示

但这个问题在于：

1. 类型限定，你只能是操作一些位，一般是整数类型或者布尔类型
2. 范围限定，范围取决于`flag`的最大值，毕竟就是一个值
3. 单一类型，能否把位运算用于不同类型，也就是说，允许任意多类型绑定

那这些问题能不能全部解决掉，实现一个动态类型的bitwise or？

## 期望

已知C++标准库已经提供了`std::tuple`，期望实现代码如下

```C++
// 假设 typename ...Ts == int, double, std::string, std::vector<int>
std::tuple<int, double, std::string, std::vector<int>> test() {
    int a = 123;
    double b = 4.5;
    std::string c = "678";
    std::vector<int> d { 9, 10 };
    return a | b | std::move(c) | std::move(d); // 期望的做法
}
```



## 做法

很显然这种写法是不对的

但是可以曲线救国

大概方法：

- 实现一个类型，重载`operator|`，并允许链式调用
- 实现隐式转型到`std::tuple<Ts...>`
- 由于这种特殊条件下的`tuple`大小是不可推导的，因此可以用`std::vector`等可变长的类型来支持存储
- 由于`std::shared_ptr<void>`自带析构器，因此可以当作低配版`std::any`来使用

我的话说完了，你应该能get到我要干嘛

```C++
class use_or {
public:
    template <typename T>
    use_or& operator|(T t) {
        _cont.emplace_back(std::make_shared<T>(std::move(t)));
        return *this;
    }

    template <typename ...Ts>
    operator std::tuple<Ts...>() {
        using seq = std::make_index_sequence<sizeof...(Ts)>;
        return make<std::tuple<Ts...>>(seq{});
    }

private:
    template <typename Tuple, size_t ...Is>
    Tuple make(std::index_sequence<Is...>) {
        Tuple tup;
        std::initializer_list<int> { ((get<Is>(tup)), 0)... };
        return tup;
    }

    template <size_t I, typename Tuple>
    void get(Tuple &tup) {
        if(I >= _cont.size()) return;
        using T = typename std::remove_reference<decltype(std::get<I>(tup))>::type;
        auto ptr = reinterpret_cast<T*>(_cont[I].get());
        std::get<I>(tup) = std::move(*ptr);
    }

private:
    std::vector<std::shared_ptr<void>> _cont;
};
```

## 测试

```C++
std::tuple<int, double, std::string, std::vector<int>> test() {
    int a = 123;
    double b = 4.5;
    std::string c = "678";
    std::vector<int> d { 9, 10 };
    return use_or{} | a | b | std::move(c) | std::move(d); // ok!
}

int main() {
    auto tup = test();
    std::cout << std::get<2>(tup) << ' '
              << std::get<3>(tup)[0] << std::endl;
    return 0;
}
```

## 然而

标准库其实已经提供了`std::make_tuple`，不也很香吗？

- 标准的做法当然是好，不仅性能好，而且类型安全，但，我喜欢`|`
- `std::make_tuple`不允许省略参数，比如在上述例子中少了`d`，则直接不允许编译（毕竟类型推导直接放到接口上了）



## 更多？

要是愿意花更多时间还可以实现像`&`、`&= ~(...)`这种常用操作

不过也没啥意思，就是多提供一个comparator和换更复杂的容器的事情，不写了



## 后记

把静态语言写成动态语言，除了爽我也没觉得有啥意义啊

看来这篇文章只能当作爽文了
