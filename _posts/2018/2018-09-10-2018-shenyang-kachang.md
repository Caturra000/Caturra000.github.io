---
layout: post
title: 2018沈阳网络赛 - Ka Chang KD树暴力
categories: [ICPC]
---

题意：给你一棵树，n个点q次操作，操作1查询x子树深度为d的节点权值和，操作2查询子树x权值和
<!--more-->


把每个点按(dfn，depth)的二维关系构造kd树，剩下的只需维护lazy标记即可

```C++
#include<bits/stdc++.h>
#define rep(i,j,k) for(register int i=j;i<=k;i++)
#define rrep(i,j,k) for(register int i=j;i>=k;i--)
#define erep(i,u) for(register int i=head[u];~i;i=nxt[i])
#define print(a) printf("%lld",(ll)(a))
#define printbk(a) printf("%lld ",(ll)(a))
#define println(a) printf("%lld\n",(ll)(a))
using namespace std;
const int MAXN = 1e5+11;
const int INF = 0x3f3f3f3f;
const double EPS = 1e-7;
typedef long long ll;
ll read(){
    ll x=0,f=1;register char ch=getchar();
    while(ch<'0'||ch>'9'){if(ch=='-')f=-1;ch=getchar();}
    while(ch>='0'&&ch<='9'){x=x*10+ch-'0';ch=getchar();}
    return x*f;
}
int to[MAXN<<1],nxt[MAXN<<1],head[MAXN],tot,CLOCK;
void init(){
    memset(head,-1,sizeof head);
    tot=0; CLOCK=0;
}
void add(int u,int v){
    to[tot]=v;
    nxt[tot]=head[u];
    head[u]=tot++;
}
int dep[MAXN],dfn[MAXN],dfned[MAXN],pre[MAXN];
void dfs(int u,int fa,int d){
    dep[u]=d; dfn[u]=++CLOCK; pre[CLOCK]=u;
    for(int i=head[u];~i;i=nxt[i]){
        int v=to[i];
        if(v==fa) continue;
        dfs(v,u,d+1);
    }
    dfned[u]=CLOCK;
}
int D;
struct Point{
    int x[2]; ll v;
    Point(int xx=0,int yy=0,ll vv=0){x[0]=xx,x[1]=yy,v=vv;}
    bool operator < (const Point &orz)const{return x[D]<orz.x[D];}
    int operator [] (const int &orz)const{return x[orz];}
}p[MAXN];
struct KD{
    int lx[MAXN][2],rx[MAXN][2];
    int son[MAXN][2],size[MAXN];
    ll sum[MAXN],lazy[MAXN];
    int root;
    #define lc son[o][0]
    #define rc son[o][1]
    void pu(int o){
        size[o]=1, sum[o]=p[o].v;
        if(lc) size[o]+=size[lc],sum[o]+=sum[lc];
        if(rc) size[o]+=size[rc],sum[o]+=sum[rc];
        rep(i,0,1){
            if(lc&&lx[lc][i]<lx[o][i]) lx[o][i]=lx[lc][i];
            if(rc&&lx[rc][i]<lx[o][i]) lx[o][i]=lx[rc][i];
            if(lc&&rx[lc][i]>rx[o][i]) rx[o][i]=rx[lc][i];
            if(rc&&rx[rc][i]>rx[o][i]) rx[o][i]=rx[rc][i]; 
        }
        
    }
    void pd(int o){
        if(lazy[o]){
            if(lc) p[lc].v+=lazy[o],sum[lc]+=lazy[o]*size[lc],lazy[lc]+=lazy[o];
            if(rc) p[rc].v+=lazy[o],sum[rc]+=lazy[o]*size[rc],lazy[rc]+=lazy[o];
            lazy[o]=0;
        }
    }
    int build(int d,int l,int r){
        D=d;
        int o=l+r>>1; nth_element(p+l,p+o,p+r+1);
        rep(i,0,1) lx[o][i]=rx[o][i]=p[o][i];
        size[o]=1; lc=rc=0; sum[o]=lazy[o]=0;
        if(l<o) lc=build(d^1,l,o-1);
        if(r>o) rc=build(d^1,o+1,r);
        pu(o);
        return o;
    }
    
    void update(int o,int L,int R,int d,ll v){
        if(!o) return;
        if(rx[o][0]<L||lx[o][0]>R||lx[o][1]>d||rx[o][1]<d) return;
        if(lx[o][0]>=L&&rx[o][0]<=R&&lx[o][1]==d&&rx[o][1]==d){
            lazy[o]+=v;
            sum[o]+=v*size[o];
            p[o].v+=v;
            return;
        }
        
        if(p[o][0]>=L&&p[o][0]<=R&&p[o][1]==d){
            sum[o]+=v;
            p[o].v+=v;
        }
        pd(o);
        update(lc,L,R,d,v);
        update(rc,L,R,d,v);
        pu(o);
    }
    ll ANS;
    void query(int o,int L,int R){
        if(!o) return;
        if(rx[o][0]<L||lx[o][0]>R) return;
        if(lx[o][0]>=L&&rx[o][0]<=R){
            ANS+=sum[o];
            return;
        }
        pd(o);
        if(p[o][0]>=L&&p[o][0]<=R) ANS+=p[o].v;
        query(lc,L,R);
        query(rc,L,R);
    }
}kd;
int main(){
    int n,q;
    while(cin>>n>>q){
        init();
        rep(i,1,n-1){
            int u=read();
            int v=read();
            add(u,v);
            add(v,u);
        }
        dfs(1,-1,0);
        rep(i,1,n) p[i]=Point(dfn[i],dep[i],0);
        kd.root=kd.build(0,1,n);
        while(q--){
            int op=read();
            if(op==1){
                int d=read();
                ll  x=read();
                kd.update(kd.root,1,CLOCK,d,x);
            }else{
                int u=read();
                kd.ANS=0;
                kd.query(kd.root,dfn[u],dfned[u]);
                println(kd.ANS);
            }
        }
    }
    return 0;
}
```