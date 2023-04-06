---
layout: post
title: 反汇编调研this offset和vtable thunk
categories: [C++]
---

## TL;DR

 

在类设计含有多重继承的内存布局下，对象的`this`指针是存在编译器隐含的偏移修正行为


现象如下：

- 首个base class并不需要修正，因为内存布局与derived class是重叠共用的，无论有无`vptr`的介入
- 其它base class由编译器修正`this`指针，需要付出轻微的运行时成本（多一条加法指令）
- 如果derived class隐藏了base class的函数，那调用时无需修正（因为调用的是derived class实现的函数）
- 对于含有非首个base class且具有`virtual`函数的base class，函数调用时`this`指针将通过访问non-virtual [thunk](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#:~:text=thunk)进行修正，当然是要更多的额外运行时成本

<!--more-->

累了，不想水文章，既然解释复杂就看代码吧
 

## case1

 

假设存在多重继承，代码如下：



```C++
struct A {
  int a;
  void func_a() {}
};
 
struct B {
  int b;
  void func_b() {}
};
 
struct C: A, B {
  int c;
  void func_c() {}
};
 
int main() {
  C c;
  c.func_a();
  c.func_b();
  c.func_c();
}
```


```
0000000000401106 <main>:
  401106:	55                   	push   %rbp
  401107:	48 89 e5             	mov    %rsp,%rbp
  40110a:	48 83 ec 10          	sub    $0x10,%rsp
  40110e:	48 8d 45 f4          	lea    -0xc(%rbp),%rax
  this等于%rbp-$0xc
  401112:	48 89 c7             	mov    %rax,%rdi
  401115:	e8 24 00 00 00       	callq  40113e <_ZN1A6func_aEv>
  40111a:	48 8d 45 f4          	lea    -0xc(%rbp),%rax
  40111e:	48 83 c0 04          	add    $0x4,%rax
  在这里this+4然后调用func_b
  401122:	48 89 c7             	mov    %rax,%rdi
  401125:	e8 20 00 00 00       	callq  40114a <_ZN1B6func_bEv>
  作为成员函数接受的是B*类型，因此需要调整
  40112a:	48 8d 45 f4          	lea    -0xc(%rbp),%rax
  40112e:	48 89 c7             	mov    %rax,%rdi
  401131:	e8 20 00 00 00       	callq  401156 <_ZN1C6func_cEv>
  401136:	b8 00 00 00 00       	mov    $0x0,%eax
  40113b:	c9                   	leaveq 
  40113c:	c3                   	retq   
  40113d:	90                   	nop
```

 

## case2

 

 

```C++
struct A {
    int a;
    void func_a() {}
};

struct B {
    int b;
    void func_b() {}
};

struct C: A, B {
    int c;
    // hide B::func_b
    void func_b() {}
    void func_c() {}
};

int main() {
    C c;
    c.func_a();
    c.func_b();
    c.func_c();
}
```

 

```
0000000000401106 <main>:
  401106:	55                   	push   %rbp
  401107:	48 89 e5             	mov    %rsp,%rbp
  40110a:	48 83 ec 10          	sub    $0x10,%rsp
  40110e:	48 8d 45 f4          	lea    -0xc(%rbp),%rax
  401112:	48 89 c7             	mov    %rax,%rdi
  401115:	e8 20 00 00 00       	callq  40113a <_ZN1A6func_aEv>
  40111a:	48 8d 45 f4          	lea    -0xc(%rbp),%rax
  40111e:	48 89 c7             	mov    %rax,%rdi
  401121:	e8 20 00 00 00       	callq  401146 <_ZN1C6func_bEv>
  this并不需要偏移，符合直觉
  401126:	48 8d 45 f4          	lea    -0xc(%rbp),%rax
  40112a:	48 89 c7             	mov    %rax,%rdi
  40112d:	e8 20 00 00 00       	callq  401152 <_ZN1C6func_cEv>
  401132:	b8 00 00 00 00       	mov    $0x0,%eax
  401137:	c9                   	leaveq 
  401138:	c3                   	retq   
  401139:	90                   	nop
```



## case3

 

```C++
struct A {
    int a;
    virtual void func_a() {}
};
struct B {
    int b;
    virtual void func_b() {}
};
struct C: A, B {
    int c;
    void func_b() override {}
    virtual void func_c() {}
};
int main() {
    C *c = new C();
    B *b = c;
    c->func_a();
    c->func_b();
    c->func_c();
    b->func_b();
}
```

 

注：此时的vtable for C布局如下：

```
vtable for C:
        .quad   0
        .quad   typeinfo for C
        .quad   A::func_a()
        .quad   C::func_b()
        .quad   C::func_c()
        .quad   -16
        .quad   typeinfo for C
        .quad   non-virtual thunk to C::func_b()

```



dump解释如下

```
0000000000401126 <main>:
  401126:	55                   	push   %rbp
  401127:	48 89 e5             	mov    %rsp,%rbp
  40112a:	53                   	push   %rbx
  40112b:	48 83 ec 18          	sub    $0x18,%rsp
  40112f:	bf 20 00 00 00       	mov    $0x20,%edi
  401134:	e8 f7 fe ff ff       	callq  401030 <_Znwm@plt>
  执行operator new
  401139:	48 89 c3             	mov    %rax,%rbx
  40113c:	48 c7 03 00 00 00 00 	movq   $0x0,(%rbx)
  401143:	c7 43 08 00 00 00 00 	movl   $0x0,0x8(%rbx)
  40114a:	48 c7 43 10 00 00 00 	movq   $0x0,0x10(%rbx)
  401151:	00 
  401152:	c7 43 18 00 00 00 00 	movl   $0x0,0x18(%rbx)
  401159:	c7 43 1c 00 00 00 00 	movl   $0x0,0x1c(%rbx)
  401160:	48 89 df             	mov    %rbx,%rdi
  401163:	e8 e4 00 00 00       	callq  40124c <_ZN1CC1Ev>
  执行C::C()
  401168:	48 89 5d e8          	mov    %rbx,-0x18(%rbp)
  this_c位于%rbp-0x18
  40116c:	48 83 7d e8 00       	cmpq   $0x0,-0x18(%rbp)
  401171:	74 0a                	je     40117d <main+0x57>
  401173:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
  401177:	48 83 c0 10          	add    $0x10,%rax
  this_b相对this_c做了偏移$0x10修正
  在gdb中运行时信息如下：
    $ p c
    > (C *) 0x416eb0
    $ p b
    > (B *) 0x416ec0
  40117b:	eb 05                	jmp    401182 <main+0x5c>
  40117d:	b8 00 00 00 00       	mov    $0x0,%eax
  401182:	48 89 45 e0          	mov    %rax,-0x20(%rbp)
  this_b位于%rbp-0x20
  401186:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
  获取this_c
  40118a:	48 8b 00             	mov    (%rax),%rax
  获取vptr，既vtable for C+16
  40118d:	48 8b 10             	mov    (%rax),%rdx
  获取func_a in vtable
  401190:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
  401194:	48 89 c7             	mov    %rax,%rdi
  传入this指针
  401197:	ff d2                	callq  *%rdx
  调用func_a
  401199:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
  40119d:	48 8b 00             	mov    (%rax),%rax
  获取vptr
  4011a0:	48 83 c0 08          	add    $0x8,%rax
  使得访问vtable偏移+0x8（以func_a或者说vtable for C+16为基准）
  4011a4:	48 8b 10             	mov    (%rax),%rdx
  4011a7:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
  4011ab:	48 89 c7             	mov    %rax,%rdi
  4011ae:	ff d2                	callq  *%rdx
  调用func_b
  4011b0:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
  4011b4:	48 8b 00             	mov    (%rax),%rax
  4011b7:	48 83 c0 10          	add    $0x10,%rax
  4011bb:	48 8b 10             	mov    (%rax),%rdx
  4011be:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
  4011c2:	48 89 c7             	mov    %rax,%rdi
  4011c5:	ff d2                	callq  *%rdx
  调用func_c
  4011c7:	48 8b 45 e0          	mov    -0x20(%rbp),%rax
  获取this_b
  4011cb:	48 8b 00             	mov    (%rax),%rax
  获取vptr，既vtable for C+56
  4011ce:	48 8b 10             	mov    (%rax),%rdx
  获取non-virtual thunk to C::func_b()
  4011d1:	48 8b 45 e0          	mov    -0x20(%rbp),%rax
  4011d5:	48 89 c7             	mov    %rax,%rdi
  传入B*类型的this_b
  4011d8:	ff d2                	callq  *%rdx
  调用non-virtual thunk to C::func_b()，既_ZThn16_N1C6func_bEv
  4011da:	b8 00 00 00 00       	mov    $0x0,%eax
  4011df:	48 8b 5d f8          	mov    -0x8(%rbp),%rbx
  4011e3:	c9                   	leaveq 
  4011e4:	c3                   	retq   
  4011e5:	90                   	nop


0000000000401209 <_ZThn16_N1C6func_bEv>:
  401209:	48 83 ef 10          	sub    $0x10,%rdi
  修正this指针，既从B*到C*
  40120d:	eb ef                	jmp    4011fe <_ZN1C6func_bEv>
  再传入到重写的C::func_b()
  40120f:	90                   	nop
```
