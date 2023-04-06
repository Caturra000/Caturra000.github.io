---
layout: post
title: CSAPP第七章笔记
categories: [Linkage, Operating System]
---

<!--more-->

CSAPP第7章就是用来讲解链接过程的，比较适合整理总结，因此做一遍笔记


## 什么是链接

如果了解目标文件的概念，那么链接过程就是从编译过程拿到的**可重定位目标文件**中生成出**可执行目标文件**

 

## 为什么需要链接

 

链接使得**分离编译**成为可能

如果平时写的是小项目，那你可认为是没有必要，杀鸡焉用牛刀

但是大型项目中，分离编译可以大量提高编译效率，充分利用并行性能

 

另外，模块化在现实世界中是客观存在的

 

## 为什么需要了解链接

 

以下理由：

- 避免踩坑
- 它会影响性能（轻微）和内存开销，你需要斟酌什么情况下使用
- 它有一些特殊的feature，比如hook，你可能会用到
- 别人给你的就是一个库，没办法啊

 

## 什么时候发生链接

 

如果是按照整个文件生成到执行的过程来看的话，一般情况是这样的：

- 编译阶段
  - 预处理
  - 编译
  - 汇编
- 链接阶段**←here**
- 装载阶段

 

而实际上，链接本身是可以位于不同的时机：

- 编译时
- 装载时
- 运行时

 

第一种是静态库所用到的时机，也是上面提到的“一般情况”

而后两种是动态库用到的

 

## 静态链接

 

静态链接过程中，linker会把可重定向目标文件（作为输入）组合在一起，形成可执行文件（作为输出）

 

在这些可重定向目标文件中，包含多个段（section），比如代码（或者是指令）一个段，未初始化变量一个段等

 

在linux中，静态链接器就是ld

linker的工作分为2个部分：

- 符号解析
- 重定位

 

## 符号解析

 

什么是符号？

你可认为是一个函数，一个全局变量，或者是一个static变量

在后面的ELF文件解析中，有更加具体的“符号”定义

 

目标文件既定义符号，也引用符号

而符号解析就是把每一个符号的引用，关联到对应的、唯一的符号的定义

 

## 重定位

 

编译器（和汇编器）生成代码段和数据段，在编译器眼里，它们是开始于地址0的

而链接器则是把符号的定义，各自关联到某一个内存的地址

此时需要把符号的引用都做出修正，以正确指向对应的内存地址

 

稍后会看到链接器如何利用汇编器生成的可重定位条目，来完成重定位工作

 

## 目标文件

 

目标文件有三种类型：

- 可重定位目标文件
- 共享目标文件（一种特殊的可重定位目标文件）
- 可执行目标文件

 

编译过程负责的工作就是生成可重定位目标文件（含共享目标文件）

而链接过程负责的工作就是生成可执行目标文件

 

我们说的目标文件（object file），就是一个以文件形式，存储在磁盘的目标模块；而目标模块（object module），就是一串字节

 

当然，字节不是随意摆放的，我们有一定的规范来描述字节的布局

而遵循这种规范实现的目标文件，我们称之为ELF（Linux环境下）

 

## ELF布局

 

ELF可重定位目标文件的布局如下

![relocatable-elf](/img/CSAPP-ch7-relocatable-elf.png)

 

ELF可执行目标文件的布局如下

![executable-elf](/img/CSAPP-ch7-executable-elf.png)
 

它们的共性就是自低到高位如下分布：

- ELF header
- 一堆section
- section header     table

 

### ELF header

 

通过readelf -h可以查阅ELF header的内容

任选一个ELF文件查阅如下所示：

 

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x10a0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          55560 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         36
  Section header string table index: 35
```

 

可以看到，ELF开头是64字节大小（size of this header），其实这个大小是固定的

 

字面意思比较好理解的就先不说，这里额外了解一下entry point address

 

Entry point address就是程序启动时首先用到的地址

对于上述ELF文件进行objdump，可以看到地址0x10a0反汇编代码为：

 

```
00000000000010a0 <_start>:
    10a0:	f3 0f 1e fa          	endbr64 
    10a4:	31 ed                	xor    %ebp,%ebp
    10a6:	49 89 d1             	mov    %rdx,%r9
    10a9:	5e                   	pop    %rsi
    10aa:	48 89 e2             	mov    %rsp,%rdx
    10ad:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
    10b1:	50                   	push   %rax
    10b2:	54                   	push   %rsp
    10b3:	4c 8d 05 e6 01 00 00 	lea    0x1e6(%rip),%r8        # 12a0 <__libc_csu_fini>
    10ba:	48 8d 0d 6f 01 00 00 	lea    0x16f(%rip),%rcx        # 1230 <__libc_csu_init>
    10c1:	48 8d 3d c1 00 00 00 	lea    0xc1(%rip),%rdi        # 1189 <main>
    10c8:	ff 15 12 2f 00 00    	callq  *0x2f12(%rip)        # 3fe0 <__libc_start_main@GLIBC_2.2.5>
    10ce:	f4                   	hlt    
    10cf:	90                   	nop
```

 

`__libc_start_main`很显然跟启动流程有关，详见后面References

 

### Section header table

 

可以通过readelf -S读取section header table（section headers），简称为SHT

从前面的ELF header信息可以看到，ELF头部是记录着start of section headers，因此找到它是很轻松的事情

 

 

SHT的作用就是用于定位到文件中所有的section，它本身是一个数组

就不展示section headers的内容了，太长

至少它提供了各个section的偏移和大小等信息

 

### 各种section

 

最常见的就三段：

- .text
- .data
- .bss

 

还有一些常见的：

- .symtab
- .rel
- .strtab

 

### ELF符号表

 

刚才的.symtab就是名副其实的符号表，用于存放当前目标模块所引用和定义的符号

关于符号，就是前面提到的：函数，全局变量和static变量

更细分地，linker根据引用定义关系把符号分为两种全局符号和一种局部符号：

- 被当前目标模块定义，被其它目标模块引用的全局符号
- 被当前目标模块引用，被其它目标模块定义的全局符号
- 只被当前模块定义和引用的局部符号

注意这里的局部符号并不是指局部变量，而是指static关键字作用的函数和全局变量

ELF中通过binding字段来区分局部符号还是全局符号，通过type来区分函数还是变量（详见Elf64_Symbol）

 

有三个伪section在SHT中没有条目：

- ABS
- UNDEF
- COMMON

TODO

 

### 一些小细节

 

由于字符串不定长的原因

符号表所用到的字符串会额外存放于.strtab，便于管理

 

static并不是分配在栈中，它可能是位于.data，也可能位于.text

 

符号表由汇编器构造完成，既.s文件的作用之一就是构造符号表

 

 

## 符号解析

 

符号解析分为局部符号解析和全局符号解析

 

局部符号解析就是对static local variables处理，确保它有唯一的名字即可

 

而全局符号解析，每当遇到一个不在本目标模块内定义的符号，则认为是被定义在其它的目标模块中，因此编译器需要在符号表中生成一个条目，交给后续的链接器处理

 

符号解析需要处理不同模块间相同名字的问题，linker的做法就是把符号分为强符号和弱符号

注意，强弱符号是给全局符号用的，而局部符号本身是对其它目标模块不可见，因此不需要讨论

 

规则如下：

- 强符号不允许多个名字存在
- 同时存在重复名字的强符号和弱符号时，选用强符号
- 同时存在重复名字的弱符号时，任意选择其一

 

强弱符号的分配规则如下：

- 全局符号，要么是强符号，要么是弱符号
- 函数及初始化的全局变量为强符号
- 未初始化的全局变量为弱符号

 

这里涉及到COMMON段的问题

当编译器遇到一个为弱符号的全局符号，因为它不知道这个值是否被其它目标模块所定义，也不知道链接器会选用哪一个符号，因此它把这个符号放到COMMON段

 

### 小细节

 

对于全局变量，如果显式地初始化为0，那也算是强符号

 

对于inline function，它们是弱符号，也恰好符合choose any of的特性（规则三）

 

对于可执行目标文件，全局变量是最终（直到链接时再）放到.data或者.bss，但在此前的过程中可放在COMMON

 

## 重定位

 

上面的符号解析把引用和定义正确关联在一起

接下来的重定位工作就是合并目标模块，并且为符号赋上运行时地址

 

重定位分为2个阶段：

- 对段和符号定义进行重定向
- 对段内的符号引用进行重定向

 

第一个阶段很好理解，就是把符号解析阶段多个重定位目标文件的相同段的合并（如下图所示），以及为符号的定义分配运行时地址（如下图，main()和swap()等定义有了具体的地址）


![relocation](/img/CSAPP-ch7-relocation1.png)
 

 

第二阶段相对复杂点，需要处理的是段内的符号引用，如main里调用swap，则是一个`call <address>`的过程，此时符号引用也需要重定位

 

在开始前，需要了解.rel段

.rel是重定位信息，如.rel.text是.text段的重定位信息，.rel.data是.data段的重定位信息

这些重定位用到的信息，称之为重定位条目，它们是由汇编器认为无法识别的符号引用所生成的

在数据结构层面，条目包含以下字段：

- offset：当前section开始的偏移量，用于标记某个需要重定位的地方
- symbol：符号信息，表示某个函数或变量
- type：重定位类型，分为PC相对地址重定位类型（PC32）和绝对地址重定位类型（32）

 

通过这些字段，可以实现下面的重定位算法

注：在我们的简化流程中，可以视r.addend == *refptr

![relocation algorithm](/img/CSAPP-ch7-relocation-algorithm.png)

![relocation](/img/CSAPP-ch7-relocation2.png) 

小细节：

回到上面ELF的布局图，我们可以看到，.rel在最终的可执行目标模块是没有的，因为它已完成了重定位

 

## 动态库

 

静态库有自己的问题：

- 难以维护更新
- 容易重复占用内存资源

 

动态库能克服这些问题，因为他允许加载时和运行时链接，并且提供COW机制

 

动态库的实现使用了PIC机制

 

## PIC

 

PIC是生成位置无关代码

它允许共享库被加载到任意内存地址

这样做的目的是为了避免固定地址而导致的内存碎片化

 

涉及到位置的就是变量和函数，下面看看如何处理

 

对于变量，我们根据代码段和数据段的距离是已知的，引入了GOT，它位于data段的头部

当处于装载阶段时，动态链接器修改GOT的每一个条目，使得能找到对应变量的绝对地址（真实的符号地址）

也就是说，当我们的程序需要得到共享库的变量时，只需间接访问GOT[]数组即可

 

如果函数也照着这么干（间接访问GOT）是不是可以呢？我觉得可以

但考虑到函数符号的数量问题，对于函数，使用的是一种lazy binding的技巧

除了GOT，还引入了PLT，后者位于代码段

 

那就是把需要重定位的函数不像变量那样加载时全部都放到GOT里

而是需要用到时，才会运行时写到GOT

 

这个过程是怎么做到的，很有意思

首先需要了解一些特殊的条目：

- GOT[0], GOT[1]: 动态链接器解析参数时用到的临时条目
- GOT[2]: 动态链接器的地址
- PLT[1]: 用于跳转到动态链接器
- 用户所用到的下表起始分别是GOT[3]和PLT[2]
- 每个GOT条目初始状态下指向原PLT的下一条指令，既访问PLT[i]时，PLT[i]指向GOT[j]，此时GOT[j]会指向PLT[i]+1

 

好，接下来就看看像跳舞一样的控制流是怎样完成lazy binding的

![plt-and-got](/img/CSAPP-ch7-plt-and-got.png)

上述图示是一个调用addvec()的例子，由于使用了PLT，因此是间接访问了PLT[2]，经过一系列操作，最终才能得到一个具体的addvec()地址：

1. 访问PLT[2]，其指向GOT[4]
2. 根据上述规则5，GOT[4]跳转到PLT[2]的第二行，PLT为代码段，这里会执行4005c6和4005cb。压栈，$0x1为addvec的ID，压栈后跳转到4005a0
3. 4005a0为PLT[0]，压栈GOT[1]，根据上述规则1，GOT[1]是提供给动态链接器的参数；继续执行4005a6，跳转到GOT[2]提供的地址，根据规则3，会跳转到动态链接器，而动态链接器就是利用上面压栈的两个参数来完成重定位，修改GOT[4]为真正的符号地址。修改后，动态链接器只需把控制流跳到PLT[2]，就能获取真正的写好了符号地址的GOT[4]，既&addvec

 

第一次肯定有点绕（相当于做了一个cache操作），但是以后的操作就简单了：

1. 访问PTL
2. 访问GOT



## References

 

CSAPP

[Errata for CS:APP3e and its Instructors Manual (cmu.edu)](http://csapp.cs.cmu.edu/3e/errata.html)

[elf(5) - Linux manual page (man7.org)](https://www.man7.org/linux/man-pages/man5/elf.5.html)

[Start analysis at any position in elf is Entry Point? - Reverse Engineering Stack Exchange](https://reverseengineering.stackexchange.com/questions/18088/start-analysis-at-any-position-in-elf-is-entry-point)

[Inline Functions in C and C++ (etherealwake.com)](https://etherealwake.com/2020/12/inline-functions-c-cpp/)

[bilibili/BV19X4y1P7zW](https://www.bilibili.com/video/BV19X4y1P7zW?p=10)
