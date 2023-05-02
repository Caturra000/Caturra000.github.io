---
layout: post
title: 树上启发式合并 初步
categories: [ICPC]
description: 题意:给出$$n$$个节点的树,每个节点有一种颜色,统计每棵子树的不同颜色的数目
---

题意:给出$$n$$个节点的树,每个节点有一种颜色,统计每棵子树的不同颜色的数目
<!--more-->


直接对每个树$$dfs$$并回溯可以得出$$O(n^2)$$的算法,并不是十分OK

如果顏色種類少可以用尺取法開$$C$$個數組離線處理或者線段樹在線處理

多了的話聽說能用主席樹？（學的太水不會用）

看了下Codeforces里的Tutorial,暂时感受了一种叫做**dsu on tree**的暴力黑科技

核心就是进行树链剖分,分出重儿子和轻儿子,每次离线$$dfs$$时保留重儿子得出的贡献,清除轻儿子的贡献并重新遍历

可以证明是$$O(nlogn)$$,证明方法没看,应该是树链剖分的同款证明?

以下试列出可能正确的代码(没地方交啊orz)


```C++
#include<iostream>
#include<algorithm>
#include<cstdio>
#include<cstring>
#include<cstdlib>
#include<cmath>
#include<string>
#include<vector>
#include<stack>
#include<queue>
#include<set>
#include<map>
#define rep(i,j,k) for(register int i=j;i<=k;i++)
#define rrep(i,j,k) for(register int i=j;i>=k;i--)
#define erep(i,u) for(register int i=head[u];~i;i=nxt[i])
#define iin(a) scanf("%d",&a)
#define lin(a) scanf("%lld",&a)
#define din(a) scanf("%lf",&a)
#define s0(a) scanf("%s",a)
#define s1(a) scanf("%s",a+1)
#define print(a) printf("%lld",(ll)a)
#define enter putchar('\n')
#define blank putchar(' ')
#define println(a) printf("%lld\n",(ll)a)
#define IOS ios::sync_with_stdio(0)
#define clr(a,b) memset(a,b,sizeof a)
using namespace std;
const int maxn = 1e5+11;
const int oo = 0x3f3f3f3f;
const double eps = 1e-7;
typedef long long ll;
ll read(){
    ll x=0,f=1;register char ch=getchar();
    while(ch<'0'||ch>'9'){if(ch=='-')f=-1;ch=getchar();}
    while(ch>='0'&&ch<='9'){x=x*10+ch-'0';ch=getchar();}
    return x*f;
}
int to[maxn<<1],nxt[maxn<<1],head[maxn],tot;
int sz[maxn],st[maxn],ed[maxn],pre[maxn],tot2;
int cnt[maxn],col[maxn],C;
void init(){
	memset(head,-1,sizeof head);
	memset(cnt,0,sizeof cnt);
	tot=tot2=0;
}
void add(int u,int v){
	to[tot]=v;nxt[tot]=head[u];head[u]=tot++;
	swap(u,v);
	to[tot]=v;nxt[tot]=head[u];head[u]=tot++;
}
void prepare(int u,int p){
	sz[u]=1;
	st[u]=++tot2;pre[tot2]=u;
	erep(i,u){
		int v=to[i];
		if(v!=p){
			prepare(v,u);
			sz[u]+=sz[v];
		}
	}
	ed[u]=tot2;
}
void dfs(int u,int p,bool keep){
	int mx=1,son=-1;
	erep(i,u){
		int v=to[i];
		if(v!=p&&sz[v]>mx){
			mx=sz[v];
			son=v;
		}
	}
	erep(i,u){
		int v=to[i];
		if(v==p||v==son)continue;
		dfs(v,u,0);
	}
	if(~son) dfs(son,u,1);
	erep(i,u){
		int v=to[i];
		if(v==p||v==son)continue;
		rep(i,st[v],ed[v]) cnt[col[pre[i]]]++;
	}
	cnt[col[u]]++;
	cout<<"Subtree "<<u<<endl;
	rep(i,1,C){
		if(cnt[i]>0) cout<<"Color "<<i<<" has "<<cnt[i]<<" vertices"<<endl; 
	} 
	cout<<endl;
	if(!keep) rep(i,st[u],ed[u]) cnt[col[pre[i]]]--;
}
int main(){
	int n,m;
	while(cin>>n>>C){
		init();m=n-1;
		rep(i,1,n) col[i]=read();
		rep(i,1,m){
			int u=read();
			int v=read();
			add(u,v);
		}
		prepare(1,0);
		rep(i,1,n) cout<<i<<" "<<sz[i]<<" "<<st[i]<<" "<<ed[i]<<endl;
		cout<<endl;
		rep(i,1,tot2) cout<<i<<" "<<pre[i]<<endl;
		dfs(1,0,0);
	}
	return 0;
}
```

给出一组样例

其中$$n$$是节点个数,$$C$$是颜色总数,$$col[i]$$为第$$i$$个节点的颜色

```
8 3
1 2 1 2 3 1 2 1
1 2
1 5
1 8
2 3
2 4
5 6
6 7
```
