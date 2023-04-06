---
layout: post
title: 十行以内实现一个defer
categories: [C++]
---

支持任意可调用对象，任意参数，任意作用域，跑的还快，go看后一言不发，惊呼C++不可战胜(x

```C++
#include <bits/stdc++.h>
/////
class Defer: std::__shared_ptr<Defer, std::_S_single> {
public:
    template <typename T, typename ...Args>
    Defer(T &&func, Args &&...args)
        : std::__shared_ptr<Defer, std::_S_single>
          (nullptr, [=](Defer*) { func(std::move(args)...); }) {}
};
/////
int func(int a, std::string str) {
    std::cout << a << " " << str << std::endl;
    return a;
}
void change(std::string &s) {
    s[0] = '1';
}
int main() {
    auto d1 = Defer(func, 2, "34");
    Defer d2 {func, 4, "566"};
    std::string s("87654321");
    {
        Defer d3(change, std::ref(s)); // or Defer d3([&] { change(s); });
    }
    std::cout << s << std::endl;
}
```

虽然说的很夸张，但都是真的

滑稽.jpg
