---
layout: post
title: KMP / exKMP / AC自动机教程
categories: [Algorithms]
description: KMP / exKMP / AC自动机教程
---

## 定义1

$$Pattern$$：模式串

$$Text$$：文本串

$$next$$：失配函数，
$$next[i]=j$$
表示$$Pattern$$的最大自匹配的真前缀（$$|pre| \lt len$$）的下标（这里是为了方便代码实现），即模式串的子串$$P[0,j] = P[i-j,i]$$，并且$$next[0] = -1$$


## 构造过程

构造过程就是初始化$$next$$的过程

假设存在$$i,j$$指针，$$i$$为当前要处理的指针，而$$j$$对应于上一次$$i-1$$时匹配的失配指针

就是说有$$nxt[i-1]=j$$

说明最大的自匹配为$$P[0,j]=P[t,i-1]$$，$$t$$多少不必在意，只需知道两子串长度相等即可

最理想的情况下，$$P[0,j+1] = P[t,i]$$，则直接完成当前初始化

否则需要$$j = next[j]$$进行迭代，贪心找到尽可能长的自匹配，使得满足$$P[0,next[next...[j]]+1] = P[t',i]$$，$$j$$增加$$1$$

（因为$$next$$都是描述真前缀和真后缀的自匹配关系，所以只需判断当前的$$P[i]$$和$$P[next...[j]]$$是否相等，这是减少复杂度的关键）

> $$i$$永远单调增大，$$next[j]$$每次最多单调增加1

极端情况就是减少到$$next[j]=-1$$仍未满足匹配关系



## 匹配过程

其思路是$$T$$匹配的后缀可能性由$$P$$来提供

以$$T$$的后缀和$$P$$的前缀作为比较，此时$$nxt$$用于对付$$T$$的后缀失配的情况

整体流程和构造过程相似，但需要多处理一个越界的情况



## 完成版



```C++
#include <bits/stdc++.h>

struct Matcher {
    std::string pattern;
    std::vector<int> next;

    Matcher(std::string p):
        pattern(std::move(p)), next(pattern.length()) 
        { initNext(); }

    void initNext() {
        next[0] = -1;
        for(int i = 1, j = -1; i < pattern.length(); ++i) {
            for(; j > -1 && pattern[i] != pattern[j+1]; j = next[j]);
            if(pattern[i] == pattern[j+1]) ++j;
            next[i] = j;
        }
    }

    int execute(const std::string &text) {
        int count = 0;
        for(int i = 0, j = -1; i < text.length(); ++i) {
            // 需要处理**越界**或失配
            for(; j > -1 && (j+1 >= text.length() || text[i] != pattern[j+1]); j = next[j]);
            if(text[i] == pattern[j+1]) ++j;
            // matchAt[i] = j;
            if(j+1 == pattern.length()) ++count;
        }
        return count;
    }
};

int main() {
    for(std::string p, t; std::cin >> p >> t;) {
        std::cout << Matcher(p).execute(t) << std::endl;
    }
    return 0;
}
```




--------------------------

感觉`EXKMP`用处并不大，教程不写了

--------

至于`AC自动机`，其实就是定义$$fail[u]$$为$$root \to u$$ 的最长相同后缀的转移节点

其余还有构造时的路径压缩，以及查询时可能用到的拓扑排序优化

然后没了

```C++
#include <bits/stdc++.h>


namespace utils {
    template <typename Container, typename ...Args>
    inline void append(Container &c, Args &&...args) {
        c.emplace_back(std::forward<Args>(args)...);
    }

    inline void gkd() {
        std::ios::sync_with_stdio(false);
        std::cin.tie(nullptr);
        std::cout.tie(nullptr);
    }

    template <unsigned int init = 19260817>
    inline int random() {
        static auto seed = init;
        seed = seed * 998244353 + 12345;
        return static_cast<int>(seed / 1024);
    }

    void debug() { std::cout << std::endl; }
    template <typename Arg, typename ...Args>
    void debug(Arg arg, Args ...args) {
        std::cout << arg << ' ';
        debug(args...);
    }
}





struct Trie {
    std::vector<std::array<int, 26>> dict;
    std::vector<bool> flag;
    std::vector<int> end;
    int root;

    Trie(): root(node()) { }

    int node() {
        auto cur = dict.size();
        utils::append(dict);
        utils::append(flag, false);
        utils::append(end, 0);
        return cur;
    }

    void add(const std::string &str) {
        auto cur = root;
        for(auto ch : str) {
            auto c = ch - 'a';
            if(!dict[cur][c]) {
                auto t = node();
                dict[cur][c] = t;
            }
            cur = dict[cur][c];
        }
        flag[cur] = true;
        ++end[cur];
    }

    std::array<int, 26>& operator[](size_t pos) {
        return dict[pos];
    }
};

struct AcAutomaton {
    Trie trie;
    std::vector<int> fail;
    int root;

    AcAutomaton(const std::vector<std::string> &strs): root(0) {
        for(auto &str : strs) trie.add(str);
        fail.resize(trie.dict.size());
        std::queue<int> que;
        fail[root] = root;
        for(int i = 0, j = 0; i < 26; ++i) {
            if(j = trie[root][i]) {
                fail[j] = root;
                que.push(j);
            }
        }
        while(!que.empty()) {
            auto u = que.front();
            que.pop();
            for(int i = 0; i < 26; ++i) {
                auto v = trie[u][i];
                auto f = fail[u];
                if(v) {
                    fail[v] = trie[f][i];
                    que.push(v);
                } else {
                    trie[u][i] = trie[f][i];
                }
            }
        }
    }

    long long query(const std::string &str) {
        auto cur =  root;
        long long res = 0;
        for(auto ch : str) {
            auto c = ch - 'a';
            cur = trie[cur][c];
            for(auto u = cur; u && ~trie.end[u]; u = fail[u]) {
                res += trie.end[u];
                trie.end[u] = -1;
            }
        }
        return res;
    }
    
};

int main() {
    utils::gkd();
    int testcase;
    std::cin >> testcase;
    while(testcase--) {
        int n; 
        std::cin >> n;
        std::vector<std::string> vec(n);
        for(auto &i : vec) std::cin >> i;
        AcAutomaton ac(vec);
        std::string s;
        std::cin >> s;
        std::cout << ac.query(s) << std::endl;
    }
    return 0;
}
```
