---
layout: post
title: Codeforces - 71E 状压DP
categories: [ICPC]
description: Codeforces - 71E 状压DP 参考官方题解
---

<!--more-->

参考官方题解

```C++
#include<bits/stdc++.h>
#define rep(i,j,k) for(register int i=j;i<=k;i++)
#define rrep(i,j,k) for(register int i=j;i>=k;i--)
using namespace std;
string s[100]={"H","He","Li","Be","B",
"C","N","O","F","Ne",
"Na","Mg","Al","Si","P",
"S","Cl","Ar","K","Ca",
"Sc","Ti","V","Cr","Mn","Fe","Co","Ni","Cu","Zn","Ga","Ge","As","Se","Br","Kr",
"Rb","Sr","Y","Zr","Nb","Mo","Tc","Ru","Rh","Pd","Ag","Cd","In","Sn","Sb","Te","I","Xe","Cs","Ba","La",
"Ce","Pr","Nd","Pm","Sm","Eu","Gd","Tb","Dy","Ho","Er","Tm","Yb","Lu","Hf","Ta","W","Re","Os","Ir","Pt","Au","Hg","Tl",
"Pb","Bi","Po","At","Rn","Fr","Ra","Ac","Th","Pa","U","Np","Pu","Am","Cm","Bk","Cf","Es","Fm"};
int num[111],ans[111][111],vec[111],que[111],p[1<<18|1];
int dp1[1<<18|1],dp2[1<<18|1];
int n,k,sum1,sum2;
string str;
bool go(){
	memset(dp1,0,sizeof dp1);
	memset(dp2,0,sizeof dp2);
//	memset(dp2,-1,sizeof dp2);
	rep(S,1,(1<<n)-1){
		rep(i,1,n){
			if((S>>(i-1))&1){
				dp1[S]+=vec[i];
			}
		}
	}
	rep(S,1,(1<<n)-1){
		dp2[S]=-1;
		for(int S0=S;S0;S0=(S0-1)&S){
			if(dp2[S^S0]!=-1&&que[dp2[S^S0]+1]==dp1[S0]){
				dp2[S]=dp2[S^S0]+1;
				p[S]=S^S0;
			}
		}
	}
	if(dp2[(1<<n)-1]!=k) return 0;
	int x=(1<<n)-1;
	rrep(i,k,1){
		rep(j,1,n){
			if(1<<(j-1)&(x^p[x])){
				ans[i][++num[i]]=j;
			}
		}
		x=p[x];
	}
	return 1;
}
inline void print(){
	cout<<"YES"<<endl;
	rep(i,1,k){
		cout<<s[vec[ans[i][1]]-1];
		rep(j,2,num[i]){
			cout<<"+"<<s[vec[ans[i][j]]-1];
		}
		cout<<"->"<<s[que[i]-1]<<endl; 
	}
}
int main(){
	while(cin>>n>>k){
		sum1=sum2=0;
		memset(num,0,sizeof num);
		rep(i,1,n){
			cin>>str;
			rep(j,0,100-1){
				if(str==s[j]){
					vec[i]=j+1;
					sum1+=j+1;
					break;
				}
			}
		}
		rep(i,1,k){
			cin>>str;
			rep(j,0,100-1){
				if(str==s[j]){
					que[i]=j+1;
					sum2+=j+1;
					break;
				}
			}
		}
		if(sum1==sum2&&go()) print();
		else cout<<"NO"<<endl;
	}
	return 0;
}
```
