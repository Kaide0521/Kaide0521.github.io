---
title: KMP算法的理解
date: 2020-01-05 21:17:09
comments: true
toc: true
categories:
- 算法练习
tags: 
- KMP

---
## KMP算法的理解
- KMP算法的核心思想
- KMP算法的实现原理
- KMP算法的编程实现
- KMP的改进


### KMP算法的核心思想
kmp算法的核心即是计算模式串的每一个位置之前的字符串的前缀和后缀公共部分的最大长度（不包括字符串本身，否则最大长度始终是字符串本身）
### KMP算法的编程实现

```
void get_next(char *T, int *next)  
{  
    int k = -1;  
    int j = 0;  
    next[j] = k;  
    while (j < strlen(T))  
    {  
        if ( (k == -1) || (T[j] == T[k]) ) //注意等号是==，而不是=  
        {  
            ++k; // 注意是先加后使用  
            ++j;  
            next[j] = k;  
        }  
        else  
        {  
            k = next[k];   
        }  
    }  
}  
```
### KMP的改进
注意到，上面的getNext函数还存在可以优化的地方，比如：

S: a   a   a   b   a   a   a   a   b

P: a   a   a   a   b

此时，i=3、j=3时发生失配，next[3]=2，此时还需要进行 3 次比较：

i=3, j=2;  

i=3, j=1;  

i=3, j=0。

而实际上，因为i=3, j=3时就已经知道a!=b，而之后的三次依旧是拿 a 和 b 比较，因此这三次比较都是多余的。

此时应当直接将P向右滑动4个字符，进行 i=4， j=0的比较。

一般而言，在getNext函数中，next[i]=j，也就是说当p[i]与S中某个字符匹配失败的时候，用p[j]继续与S中的这个字符比较。但是如果p[i]==p[j]，那么这次比较是多余的，因为p[i]与母串中的字符不匹配，那么p[j]==p[i]那么一定也和母串的字符不匹配，因此比较是多余的，此时应该直接使next[i]=next[j]。

完整的实现代码如下：

```
void getNextUpdate(const std::string& p, std::vector<int>& next)
{
    next.resize(p.size());
    next[0] = -1;
    int i = 0, j = -1;
    while (i != p.size() - 1)
    {
        //这里注意，i==0的时候实际上求的是nextVector[1]的值，以此类推
        if (j == -1 || p[i] == p[j])
        {
            ++i;
            ++j;
            //注意这里是++i和++j之后的p[i]、p[j]
            next[i] = p[i] != p[j] ? j : next[j];
        }
        else
        {
            j = next[j];
        }
    }
}
```