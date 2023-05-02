---
layout: post
title: 实现一个variant
categories: [C++, 轮子]
description: 首先你要知道什么是`variant`，还有它的好伙伴`visitor`
---

## 前置

首先你要知道什么是`variant`，还有它的好伙伴`visitor`

[看这里](https://zhuanlan.zhihu.com/p/57530780)

看完之后你应该觉得它是个好东西

## 背景

没啥背景，就是有一天觉得用`variant`来造`JSON`库似乎非常简单，恰好又没在github上找得到，因此想验证一下

但是既然是造一个库，那当然是从零写起最有挑战也最有成就感

因此这篇就是用来探索如何从`varint`写起

尝试造一个简单的`std::variant`，目标500行以内吧（实际上300行就足够了！）

## 需要什么

这是一道填空题，填完了你就能拥有类型安全的`union`、功能强大的`visitor`

（为了便于阅读和思考，做了稍微省略调整）

```C++
template <typename ...Ts>
class Variant {
private:
    int _what; // 当前类型在Ts...中的下标
    char _handle[/*多大？*/]; // 存储实际数据的句柄
    
public:
    
    Variant() {
        _what = -1;
        memset(_handle, 0, sizeof _handle);
    }
    
    template<typename T>
    void init(T &&obj) {
        using DecayT = std::decay_t<T>;
        _what = /*怎么从T获得在Ts中的下标？*/;
        new(_handle) DecayT(std::forward<T>(obj));
    }

    template <typename T,
    typename = /*SFINAE省略*/>
    Variant(T &&obj) { 
        static_assert(Position<std::decay_t<T>, Ts...>::pos != -1,
            "type not found");
        init(std::forward<T>(obj));
    }

    Variant(const Variant &rhs) {
        // 考虑过怎么方便地拷贝构造吗？
    }

    ~Variant() {
        // 怎么方便地根据不同类型来正确析构？
    }

    template <typename Visitor, typename R = /*visitor的返回类型*/>
    R visit(Visitor &visitor) {
        // 怎么实现类似于介绍中的visitor模式，至少你要能从what倒推出类型T吧？
    }

    int what() const { return _what; }
    const char* handle() { return _handle; }

};
```

大概罗列一下

- 处理`what`：需要知道某个类型`T`在一堆类型`Ts...`中的位置
- 处理`handle`：需要知道多个类型`Ts...`中的最大大小（`sizeof`）
- 实现一个类应有的基本能力：构造、析构、拷贝、赋值（含移动特性）

- 实现`visitor`

## 类型萃取

需求一和需求二是显然的类型萃取能实现的功能

虽然我也不大会这些，但是用递推的思想（和足够的耐心）还是能搞出来的

需求一：

```C++
template <typename T, typename ...Ts>
struct Position {
    constexpr static int pos = -1;
};

template <typename T, typename U, typename ...Ts>
struct Position<T, U, Ts...> {
    constexpr static int pos = std::is_same<T, U>::value ? 0 
        : (Position<T, Ts...>::pos == -1 ? -1 : Position<T, Ts...>::pos + 1);
};

// 用法： Position<int,  double, int, std::string>::pos
```

需求二：

```C++
template <typename T, typename ...Ts>
struct MaxSize {
    constexpr static size_t size = std::max(sizeof(T), MaxSize<Ts...>::size);
};
template <typename T>
struct MaxSize<T> {
    constexpr static size_t size = sizeof(T);
};

// 用法： MaxSize<int, double, std::string>::size
```

需求四的子集（从`what`倒推出`T`，那么首先要知道`Ts...`的某一下标是什么类型）：

```C++
template <size_t Pos, typename T, typename ...Ts>
struct TypeAt: public std::conditional_t<Pos == 0, 
    TypeAt<0, T>, TypeAt<Pos-1, Ts...>> {};

template <typename T>
struct TypeAt<0, T> {
    using Type = T;
};

// 用法： TypeAt<2, int, long, double, char>::Type
```

## visitor

刚才的需求四子集的实现已经接近答案的一半了

`visit`作为`visitor`的访问接口，至少需要知道返回类型，约定它为`ReturnType`吧

现在处理的是一个运行时的问题，怎么定位到当前`what`下标对应的`T`类型的`operator()`函数？

一种较为直接的方法就是线性扫描`Ts...`，通过枚举`what == TypeAt<0/1/2/3..., Ts...>`来处理

我这里稍微优化了一下，在`template`上尝试二分

```C++
template <int Lo, int Hi, typename ...Ts>
struct RuntimeChoose {
    template <typename Visitor>
    static typename Visitor::ReturnType apply(int n, void *data, Visitor &visitor) {
        constexpr int Mid = Lo + (Hi - Lo >> 1);
        if(Mid == n) {
            using Type = typename TypeAt<Mid, Ts...>::Type;
            return visitor(*reinterpret_cast<Type*>(data));
        } else if(Mid > n) {
            return RuntimeChoose<Lo, std::max(Mid - 1, Lo), Ts...>::apply(n, data, visitor);
        } else {
            return RuntimeChoose<std::min(Mid + 1, Hi), Hi, Ts...>::apply(n, data, visitor);
        }
    }
};

template <typename ...Ts>
class Variant {
    // ...
    template <typename Visitor, typename R = typename Visitor::ReturnType>
    R visit(Visitor &visitor) {
        return RuntimeChoose<0, sizeof...(Ts)-1, Ts...>
            ::apply(_what, _handle, visitor);
    }
};
```

## 试试看

### Feature

先不管类的其它功能（只让它能正确构造就好了）

假定有一个`variant`支持`int, double, string, vector`四种类型，我们试着用`visitor`的方式去让它支持`ostream`的输出

也就是说要支持这样的操作

```C++
int main() {
    Variant<int, double, std::string, std::vector<int>> var = "aha", var2 = 233;
    std::cout << var << ' ' << var2 << std::endl;
}
```

先给出接口

```C++
class Variant {
    // ...
	friend std::ostream& operator<<(std::ostream &os, Variant &variant) {
        OsVisitor osv(os); // 构造函数接受一个ostream
        return variant.visit(osv);
    }
}
```

好了现在就是要实现一个`OsVisitor`（output stream visitor）

### 标准操作

按照前置中的惯例，我们应该要这么干

```C++
struct OsVisitor {
    std::ostream &_os;
    OsVisitor(std::ostream &os): _os(os) {}
    using ReturnType = std::ostream&;
    
    std::ostream& operator()(int &obj) {
        _os << obj;
        return _os;
    }
    
    std::ostream& operator()(double &obj) {
        _os << obj;
        return _os;
    }
    
    std::ostream& operator()(std::string &obj) { /*...*/ }
    
    std::ostream& operator()(std::vector<int> &obj) { /*...*/ }
};
```

这属于标准操作，但问题是

- 要是`variant`长达10个类型，是不是要写10个`operator()`
- 或者说每一个`variant`都有不同的类型组合，是不是要每一种`varaint`都要适配多个`visitor`（或一个足够大的`visitor`）

### template

别忘了泛型编程很好使

```C++
struct OsVisitor {
    std::ostream &_os;
    OsVisitor(std::ostream &os): _os(os) {}
    using ReturnType = std::ostream&;
    
    template <typename T>
    std::ostream& operator()(T &obj) {
        _os << obj;
        return _os;
    }
};
```

是不是引起舒适了？

并不，是你踩坑了，

由于`std::vector` 不支持`operator<<`，然而模板要求用到的全部都能实例化才行，因此你连编译都过不了

### SFINAE

我们特殊搞一套，对于不支持`operator<<`的，让他能运行时抛出异常（或者自己写个特化版本定制输出），总之绝不允许编译时出错

那么我们再搞一套`<<`判断的工具吧

```C++
struct YesOrNo {
    using Yes = char[1];
    using No = char[2];
};

// 名字不太妥当，理解
template<typename T>
struct HasOperatorLeftShift : YesOrNo {

    template<typename U> static Yes& test(
        size_t (*n)[ sizeof( std::cout << * static_cast<U*>(0) ) ] );

    template<typename U> static No& test(...);

    static bool const value = sizeof( test<T>( nullptr ) ) == sizeof( Yes );
};
```

现在重新回到`OsVisitor`的实现

```C++
struct OsVisitor {
    std::ostream &_os;
    OsVisitor(std::ostream &os): _os(os) {}
    using ReturnType = std::ostream&;
    
    template <typename T>
    std::enable_if_t<HasOperatorLeftShift<T>::value, 
    std::ostream&> operator()(T &obj) { 
        _os << obj;
        return _os;
    }

    template <typename T>
    std::enable_if_t<!HasOperatorLeftShift<T>::value, 
    std::ostream&> operator()(T &obj) { 
        throw std::runtime_error("[type]" + std::string(typeid(obj).name()) 
            + " does not support IO");
        return _os;
    }
};
```

实现一个看似困难的`feature`竟是如此简洁，`visitor`的威力是不是体会到了，不仅实现用的代码很短，而且能做到的事情非常灵活

## 回到类本身

我在前面提需求把类的基础功能放到第三位，其实是坑人的，答案是先实现需求四（`visitor`），再实现需求三

也就是说，应用`visitor`到`variant`本身来实现基础功能，有点自举的感觉

由于篇幅相对长，就挑一个拷贝构造的实现提供参考吧

```C++
// lhs = rhs
template <typename ...Ts>
struct CopyConstructVisitor {
    using ReturnType = void;
    Variant<Ts...> &lhs;
    CopyConstructVisitor(Variant<Ts...> &lhs): lhs(lhs) { }
    template <typename T>
    void operator()(const T &obj) {
        lhs.init(obj);
    }
};

template <typename ...Ts>
class Variant {
    // ...
    Variant(const Variant &rhs) {
        CopyConstructVisitor<Ts...> ccv(*this);
        const_cast<Variant&>(rhs).visit(ccv);
    }
};
```

## 完成

做完四个需求后，一个最小实现的`variant`就已经完成了

以一个功能极其强大的类来说，文章的长度似乎有点短，但已经足够实现了

一个完善的`variant`实现可以参考我的轮子`vsJSON`，在`utils`目录能回答一切问题

https://github.com/Caturra000/vsJSON

