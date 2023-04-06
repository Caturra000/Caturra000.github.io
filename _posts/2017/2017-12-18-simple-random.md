---
layout: post
title: 简易随机数
categories: [ICPC]
---

实测手动实现1e7范围比内置快3倍左右

<!--more-->

```C++
unsigned int SEED = 17;
inline int Rand(){
    SEED=SEED*1103515245+12345;
    return (SEED/65536)%666;
}
```

PS xjb乱敲也是可以的

```C++
#include<stdio.h>
unsigned int xjb=2333333;
int Rand(){
    return (xjb=xjb*12345+23333)%7;
}
int arr[33],cnt,i;
int main(){
    while(cnt++<233333333) arr[Rand()]++;
    for(i=1;i<=6;i++) printf("%d\n",arr[i]);
    return 0;
}
```
