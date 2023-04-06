---
layout: post
title: 线性筛小总结
categories: [ICPC]
---

<!--more-->

线性筛的思想是每个数只会被筛一遍，因此时间复杂度是线性的（实际全局vis是大于n的，但以每个数的遍历次数来说恰好是1）

设一个合数的来源只有两种：素数 × 素数，或者是 合数 × 最小的素数，这样可以保证每个合数不会重复遍历

如果i为素数，那就令prime[j]小于等于i来避免重复遍历，所以i%prime[j]保证了prime[j]不大于i

如果i为合数，那假如i%prime[j]不为0，那prime[j]就是i×prime[j]分解的最小的素数，否则就意味着i里面含有prime[j]，而prime[]表是递增的，后者的i×prime[j+1]所含素数的最小素数肯定不是prime[j+1]（prime[j]最小），所以提前break掉

令人惊讶的是一个i%prime[j]就完成了这么繁杂的活


下面是测试用例

观察一下数据，感受感受

15 = 5 × 3

12 = 6 × 2

18 = 9 × 2

45 = 15 × 3

```C++
#include<bits/stdc++.h>
#define rep(i,j,k) for(int i=j;i<=k;i++)
#define rrep(i,j,k) for(int i=j;i>=k;i--)
using namespace std;
#define MAXN 9999  
#define MAXSIZE 1010  
const int maxn = 1e6+11;
#define DLEN 4 
const int mod = 100000007;
typedef long long ll;
bool isprime[maxn];
int prime[maxn],cnt,vis;
int sai(int n){
    rep(i,2,n) isprime[i]=1;
    rep(i,2,n){
        cout<<"**** i = "<<i<<endl;
        if(isprime[i])
            prime[++cnt]=i,vis++;
        for(int j=1;j<=cnt&&i*prime[j]<=n;j++){
            
            cout<<"** j = "<<j<<endl;
            cout<<"** i*prime[j] = "<<i*prime[j]<<endl;
            
            isprime[i*prime[j]]=0;  vis++;
            if(i%prime[j]==0){
                
                cout<<"!break!  "<<i<<"%"<<prime[j]<<endl;
                
                break;
            }
        }
    }
}
int main(){
    sai(50);
    cout<<"tot="<<vis<<endl;
}
```
```
**** i = 2
** j = 1
** i*prime[j] = 4
!break! 2%2
**** i = 3
** j = 1
** i*prime[j] = 6
** j = 2
** i*prime[j] = 9
!break! 3%3
**** i = 4
** j = 1
** i*prime[j] = 8
!break! 4%2
**** i = 5
** j = 1
** i*prime[j] = 10
** j = 2
** i*prime[j] = 15
** j = 3
** i*prime[j] = 25
!break! 5%5
**** i = 6
** j = 1
** i*prime[j] = 12
!break! 6%2
**** i = 7
** j = 1
** i*prime[j] = 14
** j = 2
** i*prime[j] = 21
** j = 3
** i*prime[j] = 35
** j = 4
** i*prime[j] = 49
!break! 7%7
**** i = 8
** j = 1
** i*prime[j] = 16
!break! 8%2
**** i = 9
** j = 1
** i*prime[j] = 18
** j = 2
** i*prime[j] = 27
!break! 9%3
**** i = 10
** j = 1
** i*prime[j] = 20
!break! 10%2
**** i = 11
** j = 1
** i*prime[j] = 22
** j = 2
** i*prime[j] = 33
**** i = 12
** j = 1
** i*prime[j] = 24
!break! 12%2
**** i = 13
** j = 1
** i*prime[j] = 26
** j = 2
** i*prime[j] = 39
**** i = 14
** j = 1
** i*prime[j] = 28
!break! 14%2
**** i = 15
** j = 1
** i*prime[j] = 30
** j = 2
** i*prime[j] = 45
!break! 15%3
**** i = 16
** j = 1
** i*prime[j] = 32
!break! 16%2
**** i = 17
** j = 1
** i*prime[j] = 34
**** i = 18
** j = 1
** i*prime[j] = 36
!break! 18%2
**** i = 19
** j = 1
** i*prime[j] = 38
**** i = 20
** j = 1
** i*prime[j] = 40
!break! 20%2
**** i = 21
** j = 1
** i*prime[j] = 42
**** i = 22
** j = 1
** i*prime[j] = 44
!break! 22%2
**** i = 23
** j = 1
** i*prime[j] = 46
**** i = 24
** j = 1
** i*prime[j] = 48
!break! 24%2
**** i = 25
** j = 1
** i*prime[j] = 50
**** i = 26
**** i = 27
**** i = 28
**** i = 29
**** i = 30
**** i = 31
**** i = 32
**** i = 33
**** i = 34
**** i = 35
**** i = 36
**** i = 37
**** i = 38
**** i = 39
**** i = 40
**** i = 41
**** i = 42
**** i = 43
**** i = 44
**** i = 45
**** i = 46
**** i = 47
**** i = 48
**** i = 49
**** i = 50
tot=49
```

小修改一下
```C++
#include<bits/stdc++.h>
#define rep(i,j,k) for(int i=j;i<=k;i++)
#define rrep(i,j,k) for(int i=j;i>=k;i--)
using namespace std;
#define MAXN 9999  
#define MAXSIZE 1010  
const int maxn = 1e6+11;
#define DLEN 4 
const int mod = 100000007;
typedef long long ll;
bool isprime[maxn];
int prime[maxn],cnt,vis;
int b[maxn];
int sai(int n){
    rep(i,2,n) isprime[i]=1;
    rep(i,2,n){
        // cout<<"**** i = "<<i<<endl;
        if(isprime[i])
            b[i]=i,prime[++cnt]=i,vis++;
        for(int j=1;j<=cnt&&i*prime[j]<=n;j++){
            
            // cout<<"** j = "<<j<<endl;
            // cout<<"** i*prime[j] = "<<i*prime[j]<<endl;
            
            isprime[i*prime[j]]=0;  vis++;
            b[i*prime[j]]=prime[j];
            if(i%prime[j]==0){
                
                // cout<<"!break!  "<<i<<"%"<<prime[j]<<endl;
                
                break;
            }
        }
    }
    return vis+1;
}
int main(){
    sai(50);
    rep(i,1,50)cout<<i<<" "<<b[i]<<endl;
}
```
```
1 0
2 2
3 3
4 2
5 5
6 2
7 7
8 2
9 3
10 2
11 11
12 2
13 13
14 2
15 3
16 2
17 17
18 2
19 19
20 2
21 3
22 2
23 23
24 2
25 5
26 2
27 3
28 2
29 29
30 2
31 31
32 2
33 3
34 2
35 5
36 2
37 37
38 2
39 3
40 2
41 41
42 2
43 43
44 2
45 3
46 2
47 47
48 2
49 7
50 2
```
再改一改
```C++
#include<bits/stdc++.h>
#define rep(i,j,k) for(int i=j;i<=k;i++)
#define rrep(i,j,k) for(int i=j;i>=k;i--)
using namespace std;
#define MAXN 9999  
#define MAXSIZE 1010  
const int maxn = 1e6+11;
#define DLEN 4 
const int mod = 100000007;
typedef long long ll;
bool isprime[maxn];
int prime[maxn],cnt,vis;
int b[maxn],miu[maxn];
int sai(int n){
    miu[1]=1;
    rep(i,2,n) isprime[i]=1;
    rep(i,2,n){
        if(isprime[i])
            b[i]=i,prime[++cnt]=i,vis++,miu[i]=-1;
        for(int j=1;j<=cnt&&i*prime[j]<=n;j++){
            isprime[i*prime[j]]=0;  vis++;
            b[i*prime[j]]=prime[j];
            if(i%prime[j]==0){
                miu[i*prime[j]]=0;
                break;
            }
            miu[i*prime[j]]=-miu[i]; //
        }
    }
}
int main(){
    sai(50);
    rep(i,1,50)cout<<i<<" "<<miu[i]<<endl;
}
```
```
1 1
2 -1
3 -1
4 0
5 -1
6 1
7 -1
8 0
9 0
10 1
11 -1
12 0
13 -1
14 1
15 1
16 0
17 -1
18 0
19 -1
20 0
21 1
22 1
23 -1
24 0
25 0
26 1
27 0
28 0
29 -1
30 -1
31 -1
32 0
33 1
34 1
35 1
36 0
37 -1
38 1
39 1
40 0
41 -1
42 -1
43 -1
44 0
45 0
46 1
47 -1
48 0
49 0
50 0
```
