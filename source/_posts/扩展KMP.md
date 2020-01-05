---
title: 扩展KMP
date: 2020-01-05 21:18:04
comments: true
toc: true
categories:
- 算法练习
tags: 
- 扩展KMP

---
## 扩展KMP

- 什么是扩展KMP

- 扩展KMP的原理

- 扩展KMP的编程实现

- 扩展KMP的用途

###### 参照 ++http://blog.csdn.net/dyx404514/article/details/41831947++
---
### 1 什么是扩展KMP

给出模板串S和T，长度分别为Slen和Tlen，要求在线性时间内，对于每个S[i]（0<=i<Slen），

求出S[i..Slen-1]与T的最长公共前缀长度，记为extend[i]

（或者说，extend[i]为满足S[i..i+z-1]==T[0..z-1]的最大的z值）。

扩展KMP可以用来解决很多字符串问题，如求一个字符串的最长回文子串和最长重复子串。

### 2 扩展KMP的原理

    记母串为S,模式串为T，next[i]表示模式串S的后缀
```math
T[i...len(T)]
```
    与模式串T的最长公共前缀。记为extend[i]为满足
```math
S[i..i+z-1]=T[0..z-1]
```
    的最大的z值，即S串以i开头的后缀与T的最长公共前缀。

####  2.1 拓展kmp算法一般步骤

首先我们从左到右依次计算extend数组，在某一时刻，设extend[0...k]已经计算完毕，并且之

前匹配过程中所达到的最远位置为P，所谓最远位置，严格来说就是i+extend[i]-1的最大值

（0<=i<=k）,并且设取这个最大值的位置为P

            
```

S ：0......p0.....k k+1......p.......len(s) 现在要计算extend[k+1],根据extend的定义

有S[p0....p] = T[0...p-p0] ——> S[k+1,P] = T[k-p0+1,P-po]

令 len = next[k-p0+1]

（前面介绍了next数组的含义,表示T串与T[k-p0+1,len(T)]的最长公共前缀）;

下面通过两种情况讨论：

    1. if k+len < P : 
            
        S[k+1,k+len] = T[0,len] ——> 则S[k+len+1]一定与T[len]不相等，为什么呢？
        
        // 因为T[k-p0+1+len] = S[k+1+len]，由next数组可知T[k-p0+1+len-1] = T[len-1]

        // T[k-p0+1+len] ！= T[len] ——> S[k+len+1] !=  T[len] 
        
        即 extend[k+1] = len
        
    2. if k + len > p : 
    
        S[p+1]之后的字符都是未知的，也就是还未进行过匹配的字符串，
        
        所以在这种情况下，就要从S[P+1]和T[P-k+1]开始一一匹配，直到发生失配为止，
        
        当匹配完成后，如果得到的extend[k+1]+(k+1)大于P则要更新未知P和p0

    至此，拓展kmp算法的过程已经描述完成
    

```

### 扩展KMP的编程实现

```
const int maxn=100010;   //字符串长度最大值  
int next[maxn],ex[maxn]; //ex数组即为extend数组  
//预处理计算next数组  
void GETNEXT(char *str)  
{  
    int i = 0,j,po,len=strlen(str);  
    next[0] = len;//初始化next[0]  
    while( str[i] == str[i+1] && i+1 < len)//计算next[1]  
    i++;  
    next[1] = i;  
    po = 1;//初始化po的位置  
    for( i = 2; i < len; i++ )  
    {  
        if( next[i-po]+i < next[po]+po )//第一种情况，可以直接得到next[i]的值  
        next[i] = next[i-po];  
        else//第二种情况，要继续匹配才能得到next[i]的值  
        {  
            j = next[po]+po-i;  
            if( j < 0 )j = 0;//如果i>po+next[po],则要从头开始匹配  
            while( i+j < len && str[j] == str[j+i] )//计算next[i]  
            j++;  
            next[i] = j;  
            po = i;//更新po的位置  
        }  
    }  
}  
//计算extend数组  
void EXKMP(char *s1,char *s2)  
{  
    int i = 0,j,po,len = strlen(s1),l2 = strlen(s2);  
    GETNEXT(s2);//计算子串的next数组  
    while ( s1[i] == s2[i] && i < l2 && i < len )//计算ex[0]  
    i++;  
    ex[0] = i;  
    po = 0;//初始化po的位置  
    for( i = 1; i < len; i++)  
    {  
        if( next[i-po]+i < ex[po]+po )//第一种情况，直接可以得到ex[i]的值  
        ex[i] = next[i-po];  
        else//第二种情况，要继续匹配才能得到ex[i]的值  
        {  
            j = ex[po]+po-i;  
            if( j < 0 ) j = 0;//如果i>ex[po]+po则要从头开始匹配  
            while( i+j < len && j < l2 && s1[j+i] == s2[j] )//计算ex[i]  
            j++;  
            ex[i] = j;  
            po = i;//更新po的位置  
        }  
    }  
}  
```