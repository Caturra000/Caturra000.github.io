---
layout: post
title: 「无用知识」C/C++中的类int类型
categories: [C++]
description: 类int类型考古
---

<!--more-->


## 背景

为什么我会去考据这些东西

只是觉得有点匪夷所思（总不能说是无聊没事干）

在`Java`中一个32位的数就是`int`，没有任何争议

而`C/C++`由于强依赖于硬件体系就显得非常自由了

出于可移植性、可读性和鲁棒性，差不多的类型总有些微妙的不同

因此本文旨在收集一些~~过于自由~~不常见的类型和推测其设计目的，just4fun

## int32_t / int_least32_t / int_fast32_t

`int32_t`在标准中为`exact-width`，就是必然是32位，因为在16位的机子中`int`是16位的，其设计目的就是用于保证可移植性。另外，标准中提到类似`intN_t`的是`optional`，要是硬件体系过于特殊没法精确表示，自然是宁缺毋滥（ provided only if the implementation directly supports the type ），如果真的缺了，C可以用下面的方式补救（事实上部分是必须要求实现的，详见参考部分`cppreference`）

`int_least32_t`在标准中为`minimum-width`，保证至少32位，这个设计目的在于当系统提供有限的类型支持时，尽量满足该需求，比如在年代遥远的未来，世界上只存在x10086的机子，能原生支持就128/256/512位的`intN_t`，那么`int_least32_t`会帮你挑128位，尽管如此，相比`int32_t`，`least`标准下是始终可用的

`int_fast32_t`就是最快的方式以满足32位的需求，然而标准只要求必须**大于**等于32位，至于如何快是"up to implementer"，其设计目的是为了高性能的数学运算提供一定可移植性。比如在未来机子确实支持32/128/512但是128位下的运算更快，你用`int_fast32_t`拿到的宽度可能是128位的

linux x64 g++下的一组数据
- `int_fast8/16/32_t`均为`typedef long int`
- `int_least32_t`为`typedef int`
- `int32_t`为`typedef signed int`


## size_t / ssize_t / size_type

`size_t`是作为一个足够大的数来保证表示该平台最大的可能出现的对象大小

`size_t`为了尽可能大，采用了无符号整型，这样做的意图是尽可能大的表示对象大小和无符号的长地址（更多是为嵌入式设备考虑），并且没有对象大小为-1的可能，基于这种定义和原则下用无符号似乎挺合理的

而在实际应用上，一般用于表示数组长度或者索引（比如64位下用int就显得不足，某种程度上打脸了我多年的数组遍历用`int`。。）

可是无符号还是存在一定的坑，比如`strlen(str) < -1`永远是对的，但自由的`C++`认为这种小事情应该交由合格的编译器给出Warning（`Java`在长度表示方面是直接给出`int`）

`size_type`是`C++`风格的定义，本质上也没差，只是高度封装下满足了容器的访问需求，并且完美的继承了`string1.length()-string2.length()`会出事的传统



## intptr_t / uintptr_t / ptrdiff_t / difference_type

`intptr_t `/`uintptr_t`仅仅是一个确保满足存储指针大小的有符号/无符号类型（ an integer type that can store any *pointer to `void`*.  ），相比较之下约束力比`size_t`更小，然而cppreference指出`uintptr_t`同义于`size_t`

`ptrdiff_t`标准要求至少有符号17位，因为在实践中如16位MSDOS的`size_t`能表示32K，然而这在32位以及现在64位时代下已经足够大而显得不那么重要了。另外，前面没提到作为有符号的`size_t`的`ssize_t`是因为它不是C标准，更为标准的`ptrdiff_t`可以取代它

相似的，`difference_type`也是为了满足容器需求而定义的



## 参考

https://stackoverflow.com/questions/47846890/int-fast32-t-and-int-fast16-t-are-typedefs-of-int-how-is-it-supposed-to-be-used

https://en.wikibooks.org/wiki/C_Programming/stdint.h

https://en.cppreference.com/w/cpp/types/integer

https://stackoverflow.com/questions/51219667/explaining-this-passage-in-about-size-t-and-ptrdiff-t

https://stackoverflow.com/questions/10168079/why-is-size-t-unsigned

https://zh.cppreference.com/w/cpp/types/size_t

http://www.aristeia.com/Papers/C++ReportColumns/sep95.pdf 
