---
layout: post
title: 又一个RPC轮子
categories: [C++, 轮子]
---

当你想造一个轮子时，你发现需要为这个轮子再造另一个轮子

<!--more-->

曾经我抱怨前司的项目写个RPC要搞一堆IDL，因此在一年前就造了个RPC轮子[sRPC](https://github.com/Caturra000/sRPC)作为一个想法上的验证

但是用着就觉得不好，于是又迭代了个新的RPC轮子[tRPC](https://github.com/Caturra000/tRPC)

不过这个目的不一样，这是冲着造raft轮子来预热的，虽然raft估计还要过个半年一年才能挤出点时间看论文写测试

写的时间不多，就几天，因为基础库（[协程](https://github.com/Caturra000/co)、[网络](https://github.com/Caturra000/FluentNet)、[json](https://github.com/Caturra000/vsJSON)）都是我前面已经写好了的，轮子多了迭代也就快了

实现上的细节都贴到README上了，就不放到blog上了
