---
layout: post
title: PAXOS小记
categories: [Distributed System]
---

随便敲的，看看就好（被书折腾后凭感觉写的，可能小误
<!--more-->


PAXOS针对2PC的保守策略改为少数服从多数的更为合理的策略

每个Acceptor可批准多个提案

每个Proposer有唯一的身份标记$M_i$，以及对应的提案内容$V_i$,用$<M,V>$表示一个提案

注意提案者$M$其实是会暗中附和其它人的提案内容，因此$V$并不唯一，所谓的选定提案更为关注的是内容$V$

提案超过半数即大于等于$\lfloor \frac{n}{2} \rfloor+1$的Acceptor批准时，该提案被选定，内容由Learner发布

规定：

P1.Accpetor必然批准接收到的第一个$<M_i,V_i>$

P2.当Accpetor批准$<M_i,V_i>$后，不会再接受$M_j \lt M_i$的任意请求，批准的$M_j \gt M_i$对应的$V_j=V_i$（解决未提交既完成选定，实际是下放到生成的时刻）

推论：

当$<M_i,V_i>$被选定时，必存在一个大小大于等于$\lfloor \frac{n}{2} \rfloor+1$的Accpetor多数集全部批准该提案

当$<M_i,M_{i+1}...M_j,V_i>$被选定时，必存在一个大小大于等于$\lfloor \frac{n}{2} \rfloor+1$的Accpetor多数集全部批准$M_i$到$M_j$的任一提案

当$<M_i,V_i>$产生时，多数集必满足任意其一 1.集合未曾批准过$M_j \lt M_i$的任意提案 2.存在一个选定的$M_j \lt M_i$的提案，$V_j = V_i$（由超过半数得选定必有交集，并且若已存在编号小的选定，那肯定是符合2）

当存在$M_j \lt M_i$的提案，那$V_i$的值一定是最大的批准的$M_i$所对应的$V_i$

因此当$<M_i,M_{i+1}...M_j,V_i>$被选定时，$M_{j+1}$的$V_{j+1} = V_i$

目的：

1.尽快达成一致

2.少数服从多数

算法步骤：暂略