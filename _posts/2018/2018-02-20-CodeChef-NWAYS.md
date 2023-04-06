---
layout: post
title: CodeChef - NWAYS 组合数 朱世杰恒等式
categories: [ICPC]
---

题意:求$$\sum_{i=1}^n\sum_{j=1}^n{|i-j|+k \choose k}$$
<!--more-->


知识点: 朱世杰恒等式,$$\sum_{i=r}^n{i \choose r}={n+1 \choose r+1},r<n$$

题解:首先去除式子中的绝对值,考虑对称性还有i=j时的重复,原式可转化为$$2\sum_{i=1}^n\sum_{j=i}^n{j-i+k \choose k}-n$$

对式子内部循环调用一遍朱世杰恒等式$$\sum_{j=i}^n{j-i+k \choose k}=\sum_{j=k}^{k+n-i}{j \choose k}={\ {k+n-i+1} \choose {k+1}}$$ 

再对外部循环调用一遍$$\sum_{i=1}^n{\ {k+n-i+1} \choose {k+1}}=\sum_{i=k+1}^{k+n}{i \choose {k+1}}={\ {k+n+1} \choose {k+2}}$$

福利:$$\sum_{i=m}^n{i \choose r}={n+1 \choose r+1}-{m \choose r+1}$$



```C++
#include<bits/stdc++.h>
#define rep(i,j,k) for(int i=j;i<=k;i++)
using namespace std;
typedef long long ll;
const ll mod = 1000000007;
const int maxn = 2e6+111;////
ll jie[maxn],inv[maxn];
ll fpw(ll a,ll n){
	ll ans=1;
	while(n){
		if(n&1) ans=(ans*a)%mod;
		n>>=1; a=(a*a)%mod;
	}
	return ans;
}
ll C(ll n,ll k){
	ll up=jie[n];
	ll down=inv[k]*inv[n-k]%mod;
	return (up*down)%mod;
}
int main(){
	ll T; scanf("%lld",&T);
	jie[0]=inv[0]=1;
	rep(i,1,maxn-2) jie[i]=(jie[i-1]*i)%mod;
	rep(i,1,maxn-2) inv[i]=fpw(jie[i],mod-2);
	while(T--){
    	ll n,k; scanf("%lld%lld",&n,&k);
    	ll tmp=C(n+k+1,k+2);
    	ll ans=((tmp*2)%mod-n+mod)%mod;
    	printf("%lld\n",ans);
	}
    return 0;
}
```
