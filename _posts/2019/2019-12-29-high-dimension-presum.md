---
layout: post
title: 高维前缀和笔记
categories: [Algorithms]
description: 高维前缀和笔记
---

算是个小知识点吧

对于常见的前缀和，正常人知道的求法是通过容斥

（为了方便表示就用了1开始的下标和伪代码）

```C++
loop i:[1,x] loop j:[1,y] loop k:[1,z]
	sum[i][j][k] = arr[i][j][k]
    + sum[i-1][j][k] + sum[i][j-1][k] + sum[i][j][k-1]
    - sum[i-1][j-1][k] - sum[i-1][j][k-1] - sum[i][j-1][k-1]
    + sum[i-1][j-1][k-1]
```

其实还有更低复杂度的写法

```C++
loop i:[1,x] loop j:[1,y] loop k:[1,z] arr[i][j][k] += arr[i][j][k-1]
loop i:[1,x] loop j:[1,y] loop k:[1,z] arr[i][j][k] += arr[i][j-1][k]
loop i:[1,x] loop j:[1,y] loop k:[1,z] arr[i][j][k] += arr[i-1][j][k]
```

而高维前缀和是对于二进制的$$n$$个维度$$arr[2][2][2][2][2]...$$，求出$$sum[2][2][2][2][2]...$$

（换成$$n$$个数的$$arr[1,n]$$数组就是求所有子集的和）

就刚才第二种的原地求和算法，可以模拟出从低位到高位维度的求和过程

（为了方便枚举位就用了0开始的下标）

```C++
for(int j = 0; (1<<j) < n; j++)
    for(int i = 0; i < n; i++)
        if(i>>j&1) arr[i] += arr[i^(1<<j)]; 
// 因为每一维就只有0和1，因此对于i的第j位xor一下就是1变为0
```

（这tm就是状压DP的最简单形式。。）
