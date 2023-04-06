---
layout: post
title: 网络流模板
categories: [ICPC]
---

<!--more-->

__ISAP__
```C++
#include<iostream>
#include<algorithm>
#include<cstdio>
#include<cstring>
using namespace std;
const int maxn = 1e5+11;
const int oo = 0x7fffffff;
int to[maxn<<1],nxt[maxn<<1],cap[maxn<<1],flow[maxn<<1];
int head[maxn],tot;
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
    cap[tot]=w;
    flow[tot]=0;
    head[u]=tot++;
}
int n,m,s,t;
int gap[maxn],dep[maxn],que[maxn];
int cur[maxn],stk[maxn];
void bfs(){
	memset(dep,-1,sizeof dep);
	memset(gap,0,sizeof gap);
	gap[0]=1;
	int front=0,rear=0;
	que[rear++]=t;dep[t]=0;
	while(front^rear){
		int u = que[front++];
		for(int i = head[u]; ~i; i = nxt[i]){
			int v=to[i];
			if(~dep[v])continue;
			que[rear++]=v;
			dep[v]=dep[u]+1;
			gap[dep[v]]++;
		}
	}	
}
int isap(){
	bfs();
	memcpy(cur,head,sizeof head);
	int ans=0,top=0,u=s;
	while(dep[s]<n){
		if(u==t){
			int mn=oo;
			int inser;
			for(int i = 0; i < top; i++){
				if(mn>cap[stk[i]]-flow[stk[i]]){
					mn=cap[stk[i]]-flow[stk[i]];
					inser=i;
				}
			}
			for(int i = 0; i < top; i++){
				flow[stk[i]]+=mn;
				flow[stk[i]^1]-=mn;
			}
			ans+=mn;
			top=inser;
			u=to[stk[top]^1];
			continue;
		}
		bool flag=0;
		int v;
		for(int i = cur[u]; ~i; i = nxt[i]){
			v=to[i];
			if(cap[i]-flow[i]&&dep[v]+1==dep[u]){
				flag=1;
				cur[u]=i;
				break;
			}
		}
		if(flag){
			stk[top++]=cur[u];
			u=v;
			continue;
		}
		int mn=n;
		for(int i = head[u]; ~i; i = nxt[i]){
			if(cap[i]-flow[i]&&dep[to[i]]<mn){
				mn=dep[to[i]];
				cur[u]=i;
			}
		}
		gap[dep[u]]--;
		if(!gap[dep[u]])return ans;
		dep[u]=mn+1;
		gap[dep[u]]++;
		if(u!=s)u=to[stk[--top]^1];
	}
	return ans;
}
```

__MCMF__
```C++
#include<iostream>
#include<algorithm>
#include<cstdio>
#include<cstring>
using namespace std;
const int maxn = 1e5+11;
const int oo = 0x3f3f3f3f;
int to[maxn<<1],cost[maxn<<1],cap[maxn<<1],flow[maxn<<1],nxt[maxn<<1];
int head[maxn],tot;
void init(){
    memset(head,-1,sizeof head);
    tot=0;
}
void add(int u,int v,int c,int w){
    to[tot]=v;
    cap[tot]=c;
    flow[tot]=0;
    cost[tot]=w;
    nxt[tot]=head[u];
    head[u]=tot++;
    swap(u,v);
    to[tot]=v;
    cap[tot]=0;
    flow[tot]=0;
    cost[tot]=-w;
    nxt[tot]=head[u];
    head[u]=tot++;
}
struct QUEUE{
    int que[maxn];
    int front,rear;
    void init(){front=rear=0;}
    void push(int x){que[rear++]=x;}
    int pop(){return que[front++];}
    bool empty(){return front==rear;}
}que;
int n,m,s,t;
bool vis[maxn];
int pre[maxn],dis[maxn];
bool spfa(){
    que.init();
    memset(vis,0,sizeof vis);
    memset(pre,-1,sizeof pre);
    memset(dis,oo,sizeof dis);
    que.push(s);vis[s]=1;dis[s]=0;
    while(!que.empty()){
        int u=que.pop(); vis[u]=0;
        for(int i = head[u]; ~i; i = nxt[i]){
            int v=to[i],c=cap[i],f=flow[i],w=cost[i];
            if(c>f&&dis[v]>dis[u]+w){
                dis[v]=dis[u]+w;
                pre[v]=i;
                if(!vis[v]){
                    que.push(v);
                    vis[v]=1;
                }
            }
        }
    }
    if(dis[t]==oo) return 0;
    else return 1;
}
int mcmf(){
    int mc=0,mf=0;
    while(spfa()){
        int tf=oo+1;
        for(int i = pre[t]; ~i; i = pre[to[i^1]]){
            tf=min(tf,cap[i]-flow[i]);
        }
        mf+=tf;
        for(int i = pre[t]; ~i; i = pre[to[i^1]]){
            flow[i]+=tf;
            flow[i^1]-=tf;
        }
        mc+=dis[t]*tf;
    }
    return mc;
}
```
__ISAP2.0__
```C++
#include<bits/stdc++.h>
using namespace std;
const int maxn = 1e5+11;
const int oo = 0x7fffffff;
int to[maxn<<1],nxt[maxn<<1],cap[maxn<<1],flow[maxn<<1];
int head[maxn],tot;
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
int dis[maxn],pre[maxn],cur[maxn],gap[maxn];
bool vis[maxn];
struct QUEUE{
    int que[maxn];
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
    int u=t,ans=oo;
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
```
