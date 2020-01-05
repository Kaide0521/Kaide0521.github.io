---
title: Redis底层数据结构之简单动态字符串
date: 2020-01-05 21:23:18
comments: true
toc: true
categories:
- redis学习
tags: 
- redis

---
## **Redis底层数据结构之简单动态字符串**

我们知道在C语言中常常使用空字符'\0'作为字符串的结尾标志，也就是使用N+1的字符数组来表示长度为N的字符串，Redis没有直接使用C语言中的字符串表示，而是构建了自己的一套字符串表示抽象，称为简单动态字符串SDS(simple dynamic string)，至于为什么Redis不采用C语言中的字符串表示方法，这也是我们接下来要探讨的问题，我们首先给出Redis简单动态字符串的数据结构，然后说明相比C语言的字符串表示方法具有的优势

### **一、Redis简单动态字符串的数据结构**
接下来我们来看Redis中是如何定义字符串的

```

    /* Redis简单动态字符串的数据结构 */
	struct sdshdr {
			//字符长度，记录buf数组中已使用的字节数量
    		unsigned int len;
    		//当前可用空间，记录buf数组中未使用的字节数量
    		unsigned int free;
    		//具体存放字符的buf
    		char buf[];
	};
```

### **二、Redis采用这种结构保存字符串的原因**

C语言中的字符表示，并不能满足Redis对字符串在安全性、效率以及功能方面的要求，具体体现在一下几点：

- **获取字符串长度的时间复杂度**

	在C语言中，要获取一个字符串的长度使用strlen函数，要对字符串进行遍历，其时间复杂度为O(N)，而SDS本身记录了字符串的长度即len属性，所以获取一个字符串的长度的实践复杂度为O(1)，特别是Redis的使用环境中存在大量、频繁的字符串操作，如果每次都调用strlen将会严重影响系统性能。

- **缓冲区溢出**

	在C语言中，我们要对一个字符串进行连接操作是，很容易造成缓冲区溢出，比如字符串char A[10] = {'a','b','c','d','\n'}，现在使用连接操作strcat(A,B)，如果在进行连接操作之前未能检查A和B的长度，就会产生缓冲区溢出，这使得程序员在写程序的时候要非常小心，一不小心手滑，就是一个严重的Bug

	而在redis中，则不存在这样的问题，因为Redis保存了字符串的当前长度和可用空间，在进行连接操作的时候，会自动检查空间是否足够，不够空间系统会自动分配，程序员无需手动修改空间大小，也不会造成缓冲区溢出

- **内存分配与释放**

	在C语言中，我们对字符串进行拼接操作时，容易造成缓冲区溢出，对字符串进行缩减/截断操作时候，如果未能及时释放未使用的字节空间，又很容易造成内存泄露，因此，在Redis中如果每次对字符串的操作都涉及空间重分配，并且在分配的过程中可能涉及到系统调用，通常将是一个非常耗时的操作；

	**因此在Redis中使用两种优化手段，进行优化**

	**(1) 空间预分配：**
	当对字符串进行拼接操作时，Redis不仅分配给满足拼接操作所必要的空间，通常还会额外分配一定量的空间供下次拼接操作使用，避免每次拼接操作进行过多的内存重分配。

	**(2) 分配原则：** 
	如果操作后的字符串长度 < 1MB ，则len的长度和free的长度一样，也就是会额外的分配一倍的空间（具体为什么这么设定还有待考究）
	如果操作后的字符串长度 >= 1MB，则Redis会分配额外的1MB未使用空间

	**(3) 惰性空间释放：** 
	在对字符串进行缩减操作时，Redis不会立即回收缩减掉的部分空间，而是使用free字段记录下来，供下次使用，同时，Redis也提供了相应的API，可以在需要的时候释放掉这些空间，以免造成内存浪费。

### **三、源码实例分析**

#### **3.1 返回字符串长度的函数sdslen**

```
   	/* 计算sds的长度，返回的size_t类型的数值 */
	/* size_t,它是一个与机器相关的unsigned类型，其大小足以保证存储内存中对象的大小。 */
	static inline size_t sdslen(const sds s) {
    		struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    		return sh->len;
	}

	/* 根据sdshdr中的free标记获取可用空间 */
	static inline size_t sdsavail(const sds s) {
    		struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    		return sh->free;
	}
```

首先来看定义内联函数的好处：

	因为定义为内联函数，所有要用到的地方可能不止一个文件。为了避免多个文件中定义同一个函数出错，所以放到头文件中。
	

看到这里大家应该还会有疑问，返回长度为什么要这样操作？我们首先看sds的定义

	typedef char *sds;

sds是一个char * 类型的指针，这和我们得sdshdr有什么关系呢？我们再看sds的构造过程函数：

```
	/* 根据init函数指针创建字符串 */
	sds sdsnew(const char *init) {
	    size_t initlen = (init == NULL) ? 0 : strlen(init);
	    return sdsnewlen(init, initlen);
	}

	/* 创建新字符串方法，传入目标长度，初始化方法 */
	sds sdsnewlen(const void *init, size_t initlen) {
	    struct sdshdr *sh;
	
	    if (init) {
			/* 调用zmalloc申请size个大小的空间 */  
	        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
	    } else {
	    	//当init函数为NULL时候，调用系统函数calloc函数申请空间 */  
	        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
	    }
	    if (sh == NULL) return NULL;
	    sh->len = initlen;
	    sh->free = 0;
	    if (initlen && init)
	        memcpy(sh->buf, init, initlen);
	   //最末端同样要加‘\0’结束符
	    sh->buf[initlen] = '\0';
	    //最后是通过返回字符串结构体sdshdr中的buf代表新的字符串的地址
	    return (char*)sh->buf;
	}

```

由代码可以看出sdsnewlen函数的返回值是buf的首地址，这样在看sdslen函数，通过给定的sds首地址减去sizeof(sdshdr)，那么就应该是该sds所对应的sdshdr数据结构首地址，自然就能得到sh->len与sh->free。


#### **3.2 字符串的拼接操作**

```
/* 以t作为新添加的len长度buf的数据，实现追加操作 */
	sds sdscatlen(sds s, const void *t, size_t len) {
	    struct sdshdr *sh;
	    size_t curlen = sdslen(s);
		
		//为原字符串扩展len长度空间
	    s = sdsMakeRoomFor(s,len);
	    if (s == NULL) return NULL;
	    sh = (void*) (s-(sizeof(struct sdshdr)));
	    //多余的数据以t作初始化
	    memcpy(s+curlen, t, len);
	    //更改相应的len,free值
	    sh->len = curlen+len;
	    sh->free = sh->free-len;
	    s[curlen+len] = '\0';
	    return s;
	}
	
	/* Append the specified null termianted C string to the sds string 's'.
	 *
	 * After the call, the passed sds string is no longer valid and all the
	 * references must be substituted with the new pointer returned by the call. */
	/* 追加t字符串 */
	sds sdscat(sds s, const char *t) {
	    return sdscatlen(s, t, strlen(t));
	}
```

#### **参考书籍：**

黄健宏的《Redis设计与实现》

#### **参考博客：**

[http://blog.csdn.net/androidlushangderen/article/](http://blog.csdn.net/androidlushangderen/article/)

[ http://blog.csdn.net/xiejingfa/article]( http://blog.csdn.net/xiejingfa/article)
