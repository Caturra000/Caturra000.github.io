---
layout: post
title: FFT推导过程
categories: [Algorithms]
description: FFT推导过程
---

所有考试总算考完了，于是我被L<sup>A</sup>J<sub>i</sub>学校坑去生产线QAQ

趁着脑袋还记得先马一下（距离遗忘DSP所有内容还有30min
<!--more-->



已知$$X[m]=\sum_{k=0}^{N-1}x[k]W_{N}^{km},m=0,1,...N-1$$

那么

$$X[m]=\sum_{k=0}^{N-1}[k==2r]x[k]W_{N}^{km}+\sum_{k=0}^{N-1}[k==2r+1]x[k]W_{N}^{km}$$

$$X[m]=\sum_{r=0}^{N/2-1}x[2r]W_{N}^{2rm}+\sum_{r=0}^{N/2-1}x[2r+1]W_{N}^{(2r+1)m}$$

$$=\sum_{r=0}^{N/2-1}x[2r]W_{N}^{2rm}+W_{N}^{m}\sum_{r=0}^{N/2-1}x[2r+1]W_{N}^{2rm}$$

$$=\sum_{r=0}^{N/2-1}x_1[r]W_{N}^{2rm}+W_{N}^{m}\sum_{r=0}^{N/2-1}x_2[r]W_{N}^{2rm}$$

由可约,

$$=\sum_{r=0}^{N/2-1}x_1[r]W_{N/2}^{rm}+W_{N}^{m}\sum_{r=0}^{N/2-1}x_2[r]W_{N/2}^{rm}$$

$$X[m]=X_1[m]+X_2[m]$$

由周期,

$$X_1[m+N/2]=X_1[m]$$

$$X_2[m+N/2]=X_2[m]$$

由对称,

$$W_{N}^{m+N/2}=-W_{N}^{m}$$

可得到另一边

$$X[m+N/2]=X_1[m+N/2]+W_{N}^{m+N/2}X_2[m+N/2]=X_1[m]-W_{N}^{m}X_2[m]$$

对比一下

$$X[m]=X_1[m]+W_{N}^{m}X_2[m]$$

$$X[m+N/2]=X_1[m]-W_{N}^{m}X_2[m]$$

复杂度
$$T(n) = 2T(n/2)+O(n)$$

FFT流程图要点

1.过程我觉得按照自底向上的写法比较好

2.原输入顺序是通过二进制的翻转(不是反)来确认的

本来考试前画了一张挺漂亮的图但找不到了..

换了一张灵魂作图


![FFT](/img/fft-draw.png)
