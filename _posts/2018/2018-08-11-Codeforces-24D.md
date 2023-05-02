---
layout: post
title: Codeforces - 24D 有后效性的DP处理
categories: [ICPC]
description: 题意:在$$n*m$$的网格中,某个物体初始置于点$$(x,y)$$,每一步行动都会等概率地停留在原地/往左/往右/往下走,求走到最后一行的的步数的数学期望,其中$$n,m<1000$$
---

题意:在$$n*m$$的网格中,某个物体初始置于点$$(x,y)$$,每一步行动都会等概率地停留在原地/往左/往右/往下走,求走到最后一行的的步数的数学期望,其中$$n,m<1000$$
<!--more-->



lyd告诉我们这种题目要倒推处理.设$$f[i][j]$$为(i,j)到(n,k)的步数期望,k为任意数

那么对于各种边界有如下情况

$$f[n][j]=0$$

$$f[i][1]=\frac{1}{3}f[i][1]+\frac{1}{3}f[i][2]+\frac{1}{3}f[i+1][1]+1$$

$$f[i][m]=\frac{1}{3}f[i][m]+\frac{1}{3}f[i][m-1]+\frac{1}{3}f[i+1][m]+1$$

$$f[i][j]=\frac{1}{4}f[i][j]+\frac{1}{4}f[i][j-1]+\frac{1}{4}f[i][j+1]+\frac{1}{4}f[i+1][j]+1$$

式子2~4需要满足$$i<n$$,式子4需要满足$$1<j<m$$

这种式子存在后效性,一种可行的方法是使用高斯消元处理,详见挑战程序设计竞赛P289 Random Walk

但以上方法仅适用于n*m<300左右的式子,这种需要手动消元才能在$$O(n*m)$$的时间内处理(其实硬上魔改应该也没问题?

首先要确认的是行内存在后效性,但行间无后效性,如果逆推的话就意味着在求解$$f[i][j]$$的时候$$f[i+1][j]$$是已知的,可作为常数处理

我们把常数项和未知数项分离

$$f[i][1]=(3+f[i+1][1])/2+\frac{1}{2}f[i][2]$$

$$f[i][1]=A+B*f[i][2],A=(3+f[i+1][1])/2,B=1/2$$

$$f[i][2]=(4+f[i][3]+f[i][1]+f[i+1][2])/3=(4+f[i][3]+A+B*f[i][2]+f[i+1][2])/3=(4+f[i][3]+A+f[i+1][2])/(3-B)$$

$$f[i][2]=(4+A+f[i+1][2])/(3-B)+\frac{1}{3-B}f[i][3]$$

$$f[i][2]=A'+B'*f[i][3],A'=(4+A+f[i+1][2])/(3-B),B'=1/(3-B)$$

以此类推直到

$$f[i][m-1]=A^{(m-1)}+B^{(m-1)}f[i][m]$$

又

$$f[i][m]=(3+A^{(m-1)}+f[i+1][m])/(2-B^{(m-1)})$$

该项可以初始化

通过逆推计算就可以得出全部项,而$$f[x][y]$$就是答案

```C++
#include<iostream>
#include<algorithm>
#include<cstdio>
#include<cstring>
#include<queue>
#include<set>
#define rep(i,j,k) for(register int i=j;i<=k;i++)
#define rrep(i,j,k) for(register int i=j;i>=k;i--)
#define erep(i,u) for(register int i=head[u];~i;i=nxt[i])
#define iter(i,j) for(int i=0;i<(j).size();i++)
#define print(a) printf("%lld",(ll)a)
#define println(a) printf("%lld\n",(ll)a)
#define printbk(a) printf("%lld ",(ll)a)
#define IOS ios::sync_with_stdio(0)
using namespace std;
const int MAXN = 1e3+11;
const int oo = 0x3f3f3f3f;
typedef long long ll;
const double EPS = 1e-8;
typedef vector<double> vec;
typedef vector<vec> mat;
ll read(){
    ll x=0,f=1;register char ch=getchar();
    while(ch<'0'||ch>'9'){if(ch=='-')f=-1;ch=getchar();}
    while(ch>='0'&&ch<='9'){x=x*10+ch-'0';ch=getchar();}
    return x*f;
}
int n,m;
double a[MAXN],b[MAXN],f[MAXN][MAXN];
int main(){
    while(cin>>n>>m){
        int x=read();
        int y=read();
        if(m==1){
            printf("%.4lf\n",(double)(n-x)*2.0);
            continue;
        }
        rep(i,0,m) f[n][i]=0;
        rrep(i,n-1,1){
            a[1]=(double)(3.0+f[i+1][1])/2.0;b[1]=0.5;
            rep(j,2,m-1){
                a[j]=(double)(4.0+a[j-1]+f[i+1][j])/(3.0-b[j-1]);
                b[j]=(double)1.0/(3.0-b[j-1]);
            }
            f[i][m]=(double)(3.0+a[m-1]+f[i+1][m])/(2.0-b[m-1]);
            rrep(j,m-1,1){
                f[i][j]=a[j]+b[j]*f[i][j+1];
            }
        }
        printf("%.10lf\n",f[x][y]);
    }
    return 0;
}
```
