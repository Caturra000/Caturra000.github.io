---
layout: post
title: 非常简洁的后缀数组教程
categories: [Algorithms]
description: 非常简洁的后缀数组教程
---

## 定义

$$sa[i]$$：字典序第$$i$$小的后缀（`suffix array`）

$$ra[i]$$：后缀$$str[i...n]$$在$$sa$$中的下标（`rank`）




## 思路

假定字符串长度为$$n$$，它必然是一个2的幂（不足就认为后面是空）

后缀数组$$sa[i]$$可认为是长为$$n$$的情况下字典序第$$i$$小的子串$$sa_n[i]$$（不足就认为后面是空）

它可以通过长度为$$n/2$$的字典序$$sa_{n/2}[i]$$（以及当前的$$ra$$）的所有情况来$$O(1)$$判断得出

（也就是说对于$$str[i,i+2^j)$$可通过$$str[i,i+2^{j/2})$$和$$str[i+2^{j/2},i+2^j)$$的关键字组合得出）

因此$$sa[i,n]$$的集合可通过$$sa_1[i],sa_2[i],sa_4[i] \dots sa_{n/2}[i]$$各个集合来倍增递推得出

所以倍增法的最优实现复杂度是$$O(nlogn)$$

（当前介绍的具体实现是基于快排的$$O(nlog^2n)$$）





## 具体过程

举个例子

$$str=ababaaab$$

$$n = 8$$

我们认为最小的合法字典序为0，空子串的字典序为-1

通过自下而上的先从$$ra_0[i]$$算起

其中横坐标代表$$i$$,空使用$$X$$来表示

每一次迭代的$$str$$是表示$$str[i,i+2^j)$$的字串

（比如$$j=2$$时的$$str$$中从0数起第5个串表示为$$aabX$$，既$$str[5,9)$$）

我们保证不可能存在$$ra$$出现第一关键字为-1的情况

```
j = 0
[str] a b a b a a a b
[ra0] 0 1 0 1 0 0 0 1

j = 1
[str] (a,b) (b,a) (a,b) (b,a) (a,a) (a,a) (a,b) (b,X)
[ra0] (0,1) (1,0) (0,1) (1,0) (0,0) (0,0) (0,1) (1,-1)
[ra1]   1     3     1     3     0     0     1     2

j = 2
[str] (ab,ab) (ba,ba) (ab,aa) (ba,aa) (aa,ab) (aa,bX) (ab,XX) (bX,XX)
[ra1]  (1,1)   (3,3)   (1,0)   (3,0)   (0,1)   (0,2)   (1,-1)  (2,-1)
[ra2]   (4)     (7)     (3)     (6)     (0)     (1)     (2)     (5)

j = 3
[str] (abab,aaab) (baba,aabX) (abaa,abX) (baaa,bX) (aaab,X) (aabX,X) (abX,X) (bX,X)
[ra2]    (4,0)       (7,1)       (3,2)      (6,5)   (0,-1)   (1,-1)   (2,-1)  (5,-1)
[ra3]     (4)         (7)         (3)        (6)     (0)      (1)      (2)     (5)
```

（其实可以看出，当第一关键字互不相同时，已经不需要再多余递推下去了，这种优化策略在大量随机串很大幅度提高效率，比如$$str = hfkdjfhkjsfhdksja$$只需计算一遍）



如果需要倍增法最优的实现，可参考国家集训队大佬给出的基数排序版本

也有$$O(n)$$实现的SA-IS算法，不过我没了解过



## 定义2

我们为$$sa$$数组衍生另外两个定义

$$hgt[i]$$：后缀$$str[sa[i]...]$$与$$str[sa[i-1]...]$$的最长公共前缀（`height`）

$$LCP(i,j)$$：后缀$$str[sa[i]...]$$和$$str[sa[j]...]$$的最长公共前缀



## 思路2

设$$j=sa[ra[i]-1]$$

当依次计算$$str[i...]$$和$$str[j...]$$(对应$$i$$在$$sa$$数组上的前一个后缀)

如果已经得知$$h[i]=hgt[ra[i]]$$（表明$$str[i]$$和$$str[j]$$的最多前$$h[i]$$个字符相同）

而对比之下$$str[i+1]$$和$$str[j+1]$$则分别对应于前面砍掉了首字符，所以至少前$$h[i]-1$$个字符相同，$$h[i+1] \ge h[i]-1$$，对于该串只需从下标$$h[i]-1$$开始判断即可

因此求出$$hgt$$数组的复杂度是$$O(n)$$



如果对于两个串$$str[i...]$$和$$str[j...]$$且满足$$ra[i] < ra[j]$$

由于$$i$$和$$j$$排名不相邻（相邻就直接是$$hgt$$了，其结果肯定不会比$$sa$$当中对应$$i$$到$$j$$的$$hgt$$更优

由$$hgt$$的传递性可以得出结论

$$LCP(i,j) = min(hgt[ra[i]+1],hgt[ra[i]+2],...,hgt[ra[j]])$$

这一部分用$$RMQ$$预处理后即可$$O(1)$$求解任意$$LCP(i,j)$$

## 举例2

```
ababaaab
h[i] str[i] str[j]
3 ababaaab abaaab
2 babaaab baaab(注意这和上一个str[j+1]没有关系)
2 abaaab ab
1 baaab b
0 aaab
2 aab aaab
1 ab aab
0 b ababaaab
```



## 完成版

```C++
#include<bits/stdc++.h>
#define fast_io() do{ios::sync_with_stdio(0); cin.tie(0); cout.tie(0);}while(0)
#define print(a) printf("%a",(ll)(a))
#define printbk(a) printf("%lld ",(ll)(a))
#define println(a) printf("%lld\n",(ll)(a))
#define rep(i,j,k) for(int i = (j); i <= (k); ++i)
#define rrep(i,j,k) for(int i = (j); i >= (k); --i)
#define erep(i,u,v) for(int i = head[u], v = to[i]; ~i; v = to[i = nxt[i]])
#define mod(a,b) (((a)%b+b)%b)
#define int int_fast32_t
using namespace std;
using ll = long long;
const int MAXN = 1e5+11;
struct SA {
    string str;
    int n;
    vector<int> sa,ra,tmp,hgt;
    void build(const string &s,bool lcp = false) {
        str = s;
        n = s.length();
        sa = ra = tmp = vector<int>(n);
        for(int i = 0; i < n; i++) {
            sa[i] = i;
            ra[i] = str[i];
        }
        auto k = 0;
        auto cmp = [&](int i,int j) -> bool {
            if(ra[i] != ra[j]) return ra[i] < ra[j];
            int ri = i+k < n ? ra[i+k] : -1;
            int rj = j+k < n ? ra[j+k] : -1;
            return ri < rj;
        };
        for(k = 1; k <= n; k <<= 1) {
            sort(sa.begin(),sa.end(),cmp);
            tmp[sa[0]] = 0;
            for(int i = 1; i < n; i++) {
                tmp[sa[i]] = tmp[sa[i-1]]+cmp(sa[i-1],sa[i]); // 依靠ra的结果,不能覆盖 
            }
            auto flag = false;
            for(auto i = 0; i < n; i++) {
                ra[i] = tmp[i];
                if(ra[i] == n-1) flag = true; // opt
            } 
            if(flag) break;  // opt
        }
        if(!lcp) return;
        hgt = vector<int>(n);
        int h = 0;
        for(int i = 0; i < n; i++) {
            if(ra[i]-1 < 0) { // 有溢出风险 
                hgt[ra[i]] = h = 0;
                continue;
            }
            int j = sa[ra[i]-1];
            if(h) h--;
            for(int t=max(i,j); t+h < n; h++) {
                if(str[i+h]!=str[j+h]) break;
            }
            hgt[ra[i]] = h;
        }
    }
}sa;
int32_t main() {
    fast_io();
    string s;
    cin >> s;
    sa.build(s,1);
    for(int i = 0; i < sa.str.length(); i++) {
        cout << sa.hgt[sa.ra[i]] << " ";
        cout << sa.str.substr(i) << " " << (sa.ra[i]-1 < 0 ? " " : sa.str.substr(sa.sa[sa.ra[i]-1])) << endl;
        
    }
    return 0;
}
```

Q：如何验证你的算法正确性？

A：https://www.luogu.com.cn/problem/P3809

更多查询：https://zhuanlan.zhihu.com/p/28331415
