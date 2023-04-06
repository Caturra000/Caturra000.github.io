---
layout: post
title: CodeChef - RIN 最小割应用 规划问题
categories: [ICPC]
---

题意：给定$$n$$门课和$$m$$个学期，每门课在每个学期有不同的得分，需要选定一个学期去完成，但存在约束条件，共有$$k$$对课程需要$$a$$在$$b$$开始学前学会，求最大得分（原问题是求最高平均得分）
<!--more-->


把问题转换为最小损失得分，那么可以用最小割来求解

$$Y[i][j]$$为第$$i$$门课在$$j$$学期损失的学分，若不存在则设为正无穷

那么每一门课$$i$$都要拆$$m$$个点，表示为$$(i,j)$$，源$$S$$和$$(i,1)$$的容量为$$Y[i][1]$$，其他学期相互连边，容量为$$Y[i][j]$$，

注意到汇点$$T$$时是$$(i,m)$$到$$T$$，边不可割，所以也为正无穷

前置课程需要保证$$b$$在第一学期不可能被割（正无穷边），且$$a$$被割都得在$$b$$被割的前面，即对于$$i>1$$的学期都要有$$(a,i-1)$$到$$(b,i)$$的边，容量为正无穷

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
using namespace std;
const int MAXN = 1e6+11;
const int INF = 0x3f3f3f3f;
const double EPS = 1e-7;
typedef long long ll;
ll read(){
    ll x=0,f=1;register char ch=getchar();
    while(ch<'0'||ch>'9'){if(ch=='-')f=-1;ch=getchar();}
    while(ch>='0'&&ch<='9'){x=x*10+ch-'0';ch=getchar();}
    return x*f;
}
int to[MAXN<<1],nxt[MAXN<<1],cap[MAXN<<1],flow[MAXN<<1];
int head[MAXN],tot;
void init(){
    memset(head,-1,sizeof head);
    tot=0;
}
void add(int u,int v,int w){
    to[tot]=v;
    nxt[tot]=head[u];
    cap[tot]=w;
    flow[tot]=0;
    head[u]=tot++;
    swap(u,v);
    to[tot]=v;
    nxt[tot]=head[u];
    cap[tot]=0;
    flow[tot]=0;
    head[u]=tot++;
}
int n,m,s,t;
int dis[MAXN],pre[MAXN],cur[MAXN],gap[MAXN];
bool vis[MAXN];
struct QUEUE{
    int que[MAXN];
    int front,rear;
    void init(){front=rear=0;}
    void push(int u){que[rear++]=u;}
    int pop(){return que[front++];}
    bool empty(){return front==rear;}
}que;
void bfs(){
    memset(vis,0,sizeof vis);
    que.init();
    que.push(t);
    vis[t]=1;dis[t]=0;
    while(que.empty()^1){
        int u = que.pop();
        for(int i = head[u]; ~i; i = nxt[i]){
            register int v=to[i],c=cap[i^1],f=flow[i^1];
            if(!vis[v]&&c>f){
                vis[v]=1;
                dis[v]=dis[u]+1;
                que.push(v);
            }
        }
    }
}
int aug(){
    int u=t,ans=INF;
    while(u!=s){
        ans=min(ans,cap[pre[u]]-flow[pre[u]]);
        u=to[pre[u]^1];
    }
    u=t;
    while(u!=s){
        flow[pre[u]]+=ans;
        flow[pre[u]^1]-=ans;
        u=to[pre[u]^1];
    }
    return ans;
}
int isap(){
    int ans=0;
    bfs();
    memset(gap,0,sizeof gap);
    memcpy(cur,head,sizeof head);
    for(int i = 1; i <= n; i++) gap[dis[i]]++;
    int u = s;
    while(dis[s]<n){
        if(u==t){
            ans+=aug();
            u=s;
        }
        bool ok=0;
        for(int i = cur[u]; ~i; i = nxt[i]){
            int v=to[i],c=cap[i],f=flow[i];
            if(c>f&&dis[u]==dis[v]+1){
                ok=1;
                pre[v]=i;
                cur[u]=i;
                u=v;
                break;
            }
        }
        if(!ok){
            int mn=n-1;
            for(int i = head[u]; ~i; i = nxt[i]){
                int v=to[i],c=cap[i],f=flow[i];
                if(c>f) mn=min(mn,dis[v]);
            }
            if(--gap[dis[u]]==0) break;
            dis[u]=mn+1;gap[dis[u]]++;cur[u]=head[u];
            if(u!=s) u=to[pre[u]^1];
        }
    }
    return ans;
}
int X[233][333],Y[233][333],id[233][333];
int A[233],B[333],k;
int main(){
    while(cin>>n>>m>>k){
        rep(i,1,n){
            rep(j,1,m){
                X[i][j]=read();
                Y[i][j]=(X[i][j]==-1?INF:(100-X[i][j]));
            }
        }
        rep(i,1,k){
            A[i]=read();
            B[i]=read();
        }
        s=n*m+1;t=n*m+2;
        init();int cnt=0;
        rep(i,1,n){
            id[i][1]=++cnt;
            add(s,id[i][1],Y[i][1]);
            rep(j,2,m){
                id[i][j]=++cnt;
                add(id[i][j-1],id[i][j],Y[i][j]);
            }
            add(id[i][m],t,INF);
        }
        rep(i,1,k){
            add(s,id[B[i]][1],INF);
            rep(j,2,m){
                add(id[A[i]][j-1],id[B[i]][j],INF);
            }
        }
        int r=n;
        n=n*m+2; //note
        ll ans=100*r-isap();
        printf("%.2lf\n",(double)ans/r);
    }
    return 0;
}
```