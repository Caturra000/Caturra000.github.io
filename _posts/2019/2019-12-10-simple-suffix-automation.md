---
layout: post
title: 非常简洁的后缀自动机教程
categories: [Algorithms]
description: 对于一个关于只接受$$str$$后缀的后缀自动机，其后缀肯定能从起始状态$$S$$合理地转移，而非后缀必然无法转移
---

对于一个关于只接受$$str$$后缀的后缀自动机，其后缀肯定能从起始状态$$S$$合理地转移，而非后缀必然无法转移

## 定义

定义相关的函数

$$trans[u][ch]=v$$：表示$$u$$状态沿着$$ch$$边转移到了$$v$$状态的转移函数

$$endpos(s)$$：表示$$str$$的任意子串$$s$$的终止位置集合，用于子串归类

性质1：当$$len(s1) \le len(s2)$$且$$endpos(s1)$$包含$$endpos(s2)$$所有子集时，$$s2$$是$$s1$$的后缀（充要）

当两个集合$$endpos(s1)==endpos(s2)$$，则认为子串$$s1$$和$$s2$$属于同一类

对于每一个相同的归类都设为一个状态，假定其中一个为$$u$$

$$substr(u)$$：表示归为同一状态$$u$$的所有子串集合

$$longest(u)$$：$$substr(u)$$中最长的子串

$$shortest(u)$$：$$substr(u)$$中最短的子串

性质2：$$substr(u)$$中每一个子串$$s$$都是$$longest(u)$$的后缀；$$longest(u)$$的后缀中满足$$len(s) \le len(shortest(u))$$的都属于状态$$u$$，既$$substr(u)$$

有了前面的东西就可以考虑一定的优化如下

$$slink[u]=v$$：我们用`suffix link`来表示不属于该状态的$$longest(u)$$的后缀（由性质1，由于长度小而无法表示不包含的$$endpos$$则用$$slink$$连接起来，直到起始状态$$S$$为止；$$slink[u]=v$$的沿向一定是$$endpos(v)$$逐渐包含$$endpos(u)$$的过程，而$$endpos(S)$$则包含所有位置）

$$minlen[u],maxlen[u]$$：分别表示状态$$u$$中最短和最长的字符串的长度（由性质2，这些都是连续的后缀，无需多余记录真实的字符串）

为此我们用$$slink$$替代了$$endpos$$，同时$$minlen,maxlen$$替代了$$substr,shortest,longest$$

## 构造过程

- 考虑增量法，假设已经构造好$$str[1...i]$$的后缀自动机，如何在线增加一个$$str[i+1]$$（设为$$ch$$）以满足$$str[j...i+1],1 \le j \le i+1$$的构造
- 至少$$str[1...i+1]$$肯定没有重复的出现在原有的任一状态中，假设新增的状态为$$u$$，令$$str[1...i]$$所在的状态$$v$$（此前最后一个新增的状态）新增转移$$trans[v][ch]=u$$（此时也表示了$$str[j...i],1 \le j \le n-minlen(u)+1$$的转移），并沿着$$slink[v]=w$$的路径，对所有的$$trans[w][ch]=null$$都把对应转移指向$$u$$
- 对于**第一个**已经存在先前的状态$$x$$满足$$trans[w][ch]=x$$的情况，需要分两类考虑
- 一类是$$maxlen[w]+1=maxlen[x]$$，此时表明该状态$$x$$完全重复且无多余地覆盖了$$str[j...i+1]$$的后缀，那么只需$$slink[u]=x$$，结束流程
- 另一类是$$maxlen[w]+1 \lt maxlen[x]$$，此时需要划分原有的$$x$$的子串为三个部分，$$minlen[x]...maxlen[w]+1...maxlen[x]$$，考虑把状态$$x$$拆分出一部分$$y$$来继承原有$$[minlen[w],maxlen[w]+1]$$的部分，$$x$$只包含剩下的部分（此时$$slink[y]=slink[x],slink[x]=y$$），那么状态$$y$$的出现使得问题回归到上一类的情况，只需$$trans[w][ch]=y,slink[u]=y$$
- 对于上面的转移可能不止一个$$w$$转移向原$$x$$，所以需要把连续的$$trans[w][ch]$$置为$$y$$

## 完成版

```C++
#include<bits/stdc++.h>
#define fast_io() do{ios::sync_with_stdio(0); cin.tie(0); cout.tie(0);}while(0)
using namespace std;

namespace util {
    template<typename T>
    inline void alloc(vector<T> &v) {
        v.emplace_back();
    }
    template<typename T>
    inline void empb(vector<T> &v,T &arg) {
        v.emplace_back(arg);
    }
}

using namespace util;
struct SuffixAutomaton {
    vector<array<int,26>> trans;
    vector<int> slink,minlen,maxlen;
    int last;
    
    inline void init() {
        alloc(trans);
        alloc(slink);
        alloc(minlen);
        alloc(maxlen);
    }
    
    SuffixAutomaton() {
        init();
        last = 0;
        slink[0] = -1;
        minlen[0] = maxlen[0] = 0;
        trans[0].fill(-1);
    } 
    
    int state(int tr,int suf,int mn,int mx) {
        int cur = trans.size();
        alloc(trans);
        if(~tr) trans[cur] = trans[tr];
        else trans[cur].fill(-1);
        empb(slink,suf);
        empb(minlen,mn);
        empb(maxlen,mx);
        return cur;
    }
    
    int add(char ch) {
        int c = ch - 'a';
        int u = state(-1,-1,-1,maxlen[last]+1);
        int v = last;
        while(v!=-1 && trans[v][c] == -1) {
            trans[v][c] = u;
            v = slink[v];
        }
        if(v == -1) {
            minlen[u] = 1;
            slink[u] = 0;
            return last = u;
        }
        int x = trans[v][c];
        if(maxlen[v]+1 == maxlen[x]) {
            slink[u] = x;
            minlen[u] = maxlen[x]+1;
            return last = u;
        }
        int y = state(x,-1,-1,maxlen[v]+1);
        minlen[y] = maxlen[slink[y] = slink[x]]+1;
        minlen[x] = maxlen[slink[x] = y]+1;
        minlen[u] = maxlen[slink[u] = y]+1;
        while(v != -1 && trans[v][c] == x) {
            trans[v][c] = y;
            v = slink[v];
        }
        return last = u;
    }
};

int main() {
    fast_io();
    string str;
    cin >> str;
    SuffixAutomaton sam;
    for(int i = 0; str[i]; i++) {
        sam.add(str[i]);
    }
    long long res = 0;
    for(int i = 1; i < sam.trans.size(); i++) {
        res += sam.maxlen[i]-sam.minlen[i]+1;
    } 
    cout << res << endl;
    return 0;
}
```

old version: https://paste.ubuntu.com/p/TjFs5wVS8d/

## 参考

hihocoder127/128解题方法提示
