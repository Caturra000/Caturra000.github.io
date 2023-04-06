---
layout: post
title: 非常简洁的回文树教程
categories: [Algorithms]
---

## 定义

奇根$$0$$：长度为-1的回文串所在节点（为了便于处理）

偶根$$1$$：长度为0的回文串所在节点

$$len[u]$$：当前节点$$u$$维护的回文长度

$$trans[u][c]$$：转移函数，注意其中所有状态均为回文（单次转移相当于$$u$$节点代表字符串左右两边加上$$c$$）

$$fail[u]$$：失配指针，指向$$u$$最大匹配后缀回文的所在节点



## 前置理论

一个长度为$$n$$的字符串中，本质不同的回文子串个数也不超过$$n$$

对于在一个串中新增一个字符的情况下，本质不同回文子串个数最多新增1个

证明：

考虑增量过程中新增的字符$$str[i]$$，新增的回文肯定跟它有关（也就是肯定是$$str[j...i]$$的形式，$$j$$任取）

我们假设真的存在新增多个本质不同回文子串，从中挑出最长的一个串$$s$$

可以发现其它的回文子串即使存在也是以$$s$$的后缀形式存在

但作为回文，后缀出现过的，那么前缀也肯定出现过，因此是属于本质相同的子串，END

因此我们可以用$$O(n)$$个状态转移来表示所有本质不同的回文子串



## 构造过程

1. 偶根的失配指针指向奇根，奇根必然不会失配（单个字符也能形成回文串）
2. 采用增量的方法一个一个添加字符，假设现在添加字符$$c$$
3. 维护$$str[0...i)$$后缀的最长回文字串$$str[j...i)$$，$$j$$任取，设所在节点为$$u$$
4. 通过不断地寻找$$u$$的`suffix-link`（$$fail[..fail[u]]$$），相当于缩短后缀，找到一个回文串$$X$$,使得满足在原串中符合$$cXc$$形式的回文串，设节点为$$v$$
5. 如果存在，那我们不必理会，因为由状态存在得知是本质相同的（前面存在过的）；否则新增$$trans[u][c]$$状态$$n$$，我们已知其长度就是$$len[v]$$多两个字符的长度（这个时候奇根直接+2就是巧妙的形成单个字符的回文串）
6. 至于$$fail[v]$$，可以发现同样是同样是$$u/w$$的后缀加上$$c$$，直接继续在$$w$$的`suffix-link`上寻找符合$$cYc$$的状态即可，设为$$w$$，令$$fail[v]=trans[w][c]$$

好了已经构造完了，由于找$$fail$$的过程会使得$$str[j...i]$$中的$$j$$不断右移，因此总体来看，其成本还是$$O(n)$$


## 完成版

```C++
#include <bits/stdc++.h>
struct PT {
    const std::string &str;

    std::vector<std::array<int, 127>> trans;
    std::vector<int> fail;
    std::vector<int> len;

    int odd, even;
    int last;

    int make(int f, int l) {
        int node = trans.size();
        trans.push_back({});
        fail.emplace_back(f);
        len.emplace_back(l);
        return node;
    }

    PT(const std::string &str)
        : str(str),
          odd(make(0, -1)),
          even(make(0, 0)),
          last(even) {
        for(int i = 0; i < str.size(); ++i) {
            add(i);
        }
    }

    void add(int index) {
        int c = str[index];
        int u = last;
        int v = failwalk(u, index);
        if(!trans[v][c]) {
            int f, l;
            if(len[v] == -1) {
                f = even;
                l = 1;
            } else {
                int w = failwalk(fail[v], index);
                f = trans[w][c];
                l = len[v] + 2;
            }
            int n = make(f, l);
            trans[v][c] = n;
        }
        last = trans[v][c];
    }

    int failwalk(int u, int index) {
        for(;;) {
            int lo = index - 1 - len[u];
            if(lo >= 0 && str[lo] == str[index]) break;
            u = fail[u];
        }
        return u;
    }
};

int main() {
    std::string str = "aabaaa";
    PT pt(str);
    std::cout << *std::max_element(pt.len.begin(), pt.len.end()) << std::endl;
    return 0;
}
```

## 参考

http://adilet.org/blog/palindromic-tree/