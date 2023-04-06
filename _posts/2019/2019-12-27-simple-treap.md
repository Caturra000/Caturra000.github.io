---
layout: post
title: 非常简洁的无旋Treap教程
categories: [Algorithms]
---

无旋Treap是一个非常精简的平衡树，定义也挺简单的，旨在替代splay等反人类旋转树

## 定义

$$fix$$：决定节点间父子关系的随机权重（就是作为堆的key）

$$val$$：真实的权重，决定节点间中序关系

$$split(cur,pivot,u,v)$$：把当前的$$cur$$树划分成$$u,v$$子树，按照$$pivot$$大小划分

$$merge(u,v)$$：把$$u,v$$子树合并，按照$$fix$$决定树的结构

## 细节

一般treap更适合用于区间操作（把$$pivot$$按照$$size$$来划分即可，而树形操作则用$$val$$来比较），因此这里没有$$cnt$$来维护重复的节点个数

## 构造过程

构造就是逐个插入过程了，假如我们实现了$$split$$和$$merge$$，每次都按照插入的键值$$val$$来$$split(root)$$来划分小于等于$$val$$和大于$$val$$的子树$$a$$和$$b$$，把$$a$$和单一的新建节点$$t$$进行$$merge$$后再与$$b$$进行$$merge$$即可

够简单粗暴吧

## 其他操作

删除也是split后直接弃置单个节点再merge即可

排名split后拿size比一下就有了

前驱后继还是split后按照根的左右子树遍历就有了

（实在太方便了，但别忘了要merge回去）

## 核心操作

从前面的结论得出，只要实现了$$split$$和$$merge$$，其实已经完成80%的工作了

$$split(cur,pivot,u,v)$$操作中：

如果$$pivot \lt val[cur]$$，那就说明$$b$$囊括了$$cur$$及其右子树$$rc[cur]$$，剩余的$$a$$和$$lc[cur]$$递归处理

另一种情况是相似的

$$merge(u,v)$$操作中：

采用小根堆的做法，假定$$u$$子树中最大的值都小于$$v$$子树中最小的值

如果$$fix[u] \lt fix[v]$$，$$u$$则作为$$v$$的父节点，否则相反，剩余的关系递归处理

## 完成版

```C++
#include <iostream>
#include <vector>

/****************** template begin ******************/
#define TYPE(v) std::decay<decltype(v)>::type
#define ASC(i, j, k) for(TYPE(k) i = j, _##i = k; i <= _##i; ++i)
#define DESC(i, j, k) for(TYPE(k) i = j, _##i = k; i >= _##i; --i)
#define ENDL '\n'
#define UNLIKELY(x)  __builtin_expect(!!(x), 0)
#define INLINE __attribute__((always_inline))

constexpr size_t MAXN = 1e6+11;
using ll = long long;

namespace utils {
    template <typename Container, typename ...Args>
    inline void add(Container &c, Args &&...args) {
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
using namespace utils;


namespace graph {
    using Vertex = unsigned int;
    using Value = long long;
    using EdgeInfo = std::pair<Vertex, Value>;

    struct Edge: public EdgeInfo {
        using EdgeInfo::EdgeInfo;
        Vertex &to = first;
        Value &cost = second;
    };

    struct Graph {
        std::vector<std::vector<Edge>> edges;
        explicit Graph(std::size_t size): edges(size) { }
        void add(Vertex from, Vertex to, Value cost)
        { edges[from].emplace_back(to, cost); }
        std::vector<Edge>& operator[](Vertex vertex)
        { return edges[vertex]; }
    };
}
using namespace graph;
/****************** template end ******************/

using namespace std;


struct Treap {
    using Node = unsigned int;
    using Value = int;
    vector<Node> lc, rc;
    vector<Value> fix, val;
    vector<Node> size;
    Node root;

    Treap(): root(0) {
        node(0);
        size[0] = 0;
    }

    Node node(Value v) {
        auto id = lc.size();
        utils::add(lc, 0);
        utils::add(rc, 0);
        utils::add(fix, random());
        utils::add(val, v);
        utils::add(size, 1);
        return id;
    }

    void pushUp(Node cur) {
        size[cur] = 1 + size[lc[cur]] + size[rc[cur]];
    }

    void split(Node cur, Value pivot, Node &u, Node &v) {
        if(!cur) {
            u = v = 0;
            return;
        }
        if(pivot < val[cur]) {
            v = cur;
            split(lc[cur], pivot, u, lc[cur]);
        } else {
            u = cur;
            split(rc[cur], pivot, rc[cur], v);
        }
        pushUp(cur);
    }

    Node merge(Node u, Node v) {
        if(!u) return v;
        if(!v) return u;
        if(fix[u] < fix[v]) {
            rc[u] = merge(rc[u], v);
            pushUp(u);
            return u;
        } else {
            lc[v] = merge(u, lc[v]);
            pushUp(v);
            return v;
        }
    }

    void add(Value k) {
        if(!root) {
            root = node(k);
            return;
        }
        Node u, v;
        Node t = node(k);
        split(root, k, u, v);
        root = merge(merge(u, t), v);
    }

    void remove(Value k) {
        Node u, v;
        split(root, k, u, v);
        Node x, y;
        split(u, k-1, x, y);
        y = merge(lc[y], rc[y]);
        u = merge(x, y);
        root = merge(u, v);
    }

    Value rank(Value k) {
        Node u, v;
        split(root, k-1, u, v);
        Value ans = size[u] + 1;
        root = merge(u, v);
        return ans;
    }

    Node kthNode(Node cur, unsigned int kth) {
        while(cur) {
            if(kth <= size[lc[cur]]) {
                cur = lc[cur];
            } else if(kth == size[lc[cur]]+1) {
                return cur;
            } else {
                kth -= size[lc[cur]]+1;
                cur = rc[cur];
            }
        }
        return 0;
    }

    Value preVal(Value k) {
        Node u, v;
        split(root, k-1, u, v);
        Node cur = kthNode(u, size[u]);
        root = merge(u, v);
        return val[cur];
    }

    Value succVal(Value k) {
        Node u, v;
        split(root, k, u, v);
        Node cur = kthNode(v, 1);
        root = merge(u, v);
        return val[cur];
    }
};

int main() {
    gkd();
    vector<int> ans;
    for(int n; cin >> n;) {
        Treap treap;
        for(int i = 0; i < n; ++i) {
            int op, val;
            cin >> op >> val;
            switch(op) {
                case 1:
                    treap.add(val);
                break;
                case 2:
                    treap.remove(val);
                break;
                case 3:
                    add(ans, treap.rank(val));
                break;
                case 4:
                    add(ans, treap.val[treap.kthNode(treap.root,val)]);
                break;
                case 5:
                    treap.add(val);
                    add(ans, treap.preVal(val));
                    treap.remove(val);
                break;
                case 6:
                    treap.add(val);
                    add(ans,treap.succVal(val));
                    treap.remove(val);
                break;
                default:
                break;
            }
        }
    }
    for(auto v : ans) {
        cout << v << ENDL;
    }
    return 0;
}
```
