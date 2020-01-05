---
title: 锁与volatile的内存语义
date: 2020-01-05 21:14:23
comments: true
toc: true
categories:
- java
tags: 
- java内存模型

---
## **锁与volatile的内存语义** ##

- 1.锁的内存语义
- 2.volatile内存语义
- 3.synchronized内存语义
- 4.Lock与synchronized的区别
- 5.ReentrantLock源码实例分析

----------
### **1.锁的内存语义**
锁是java并发编程中最重要的同步机制。锁除了让临界区互斥执行外，还可以让释放锁的线程向获取同一个锁的线程发送消息。
### **1.1 锁释放和获取的内存语义**
当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中；
当线程获取锁时，JMM会当前线程拥有的本地内存共享变量置为无效，从而使得被监视器保护的临界区代码必须要从主内存中去读取共享变量；

### **1.2 CAS操作** 
CAS是单词compare and set的缩写，意思是指在set之前先比较该值有没有变化，只有在没变的情况下才对其赋值。

问题：如何在没有锁的情况下实现i++原子操作？

CAS操作涉及到三个操作数，一个是内存值，一个是旧的预期值，一个是更新后的值，如果内存值和旧的预期值没有发生变化，才设置成新的值。

      public final int incrementAndGet() {
        for (;;) {
        	//得到预期值
            int current = get();
            //得到更新后的值
            int next = current + 1;
            //通过CAS操作验证是否发生变化
            if (compareAndSet(current, next))
                return next;
        }
    }
CAS的原子性实际上是CPU实现的.

CAS操作用途：可以用CAS在无锁的情况下实现原子操作，但要明确应用场合，非常简单的操作且又不想引入锁可以考虑使用CAS操作，当想要非阻塞地完成某一操作也可以考虑CAS。不推荐在复杂操作中引入CAS，会使程序可读性变差，且难以测试，同时会出现ABA问题。


### **2.volatile内存语义** 

2.1 volatile关键字的特性：

	（1）可见性：对一个volatile关键字的读，总是能看到（任意线程）对这个关键字的写

	（2）原子性：对任意单个volatile变量的写操作，具有原子性（注：多个volatile组合操作不具有原子性）

- 执行volatile写的时候，JMM会把该线程对应的本地内存（并不是实际存在的，也称为TLB，线程本地缓冲区）刷新到主内存中。这个过程可以理解为线程1（执行写方法的线程）向接下来要读取这个变量的线程（执行读方法的线程）发送了一条消息
- 执行volatile读的时候，JMM会把该线程对应的本地内存设置为无效。线程接下来将直接从主内存中读取共享变量的值。这个过程可以理解为线程2接收到了线程1发送的消息

**2.2 内存语义**

- 当写一个volatile变量时，JMM会将本地变量中对应的共享变量值刷新到主内存中
- 当读一个volatile变量时，JMM会将线程本地变量存储的值，置为无效值，线程接下来将从主内存中读取共享变量

**2.3 实现原理**

    instance = new Singleton(); 定义一个volatile变量

其对应编译后的cpu指令为：

    0x01a3de1d: movb $0×0,0×1104800(%esi);0x01a3de24: lock addl $0×0,(%esp);

由编译后的汇编指令可以看出，改指令相比其他指令多个一个lock前缀

Lock前缀的指令在多核处理器下会引发了两件事情：

- 将当前处理器缓存行中的数据写回到主内存中
- 这个写回的过程会使得其他处理器中缓存了该内存地址的数据无效

**具体实现细节：**

- 锁总线：早期的实现方式，当CPU读取共享变量时会锁住总线，由于cpu和其他部件的通信都是通过总线实现的，如果锁住总线的话，cpu就不能与其他部件之间进行通信，CPU处于等待的状态，导致整个系统的效率低下。
- 锁缓存&缓存一致性协议：当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-GXgEIpXq-1578230026680)(http://i.imgur.com/i1Fpfis.jpg)]

### **小结：** ###

锁的内存语义的实现与可重入锁相关，可以简要总结锁的内存语义的实现包括以下两种方式：

- 利用volatile变量的内存语义
- 利用CAS附带的volatile语义


### **3. synchronized内存语义**
    synchronized也称为监视器锁，由JVM控制实现，每个对象都有类似监视器一样的锁，当监视器锁被占用是对象将会处于锁定状态，
    每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

1. 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
2. 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
3. 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

当线程执行monitorexit指令时候，过程如下：
执行monitorexit的线程必须是objectref所对应的monitor的所有者。

指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。 

　　通过这两段描述，我们应该能很清楚的看出Synchronized的实现原理，Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。
　　
    
加轻量锁的过程很简单：在当前线程的栈帧（stack frame）中生成一个锁记录（lock record），这个锁记录比前面说的那个对象锁（管理线程队列的monitor）简单多了，它只是对象头的一个拷贝。然后把对象头里的tag改成00，并把这个栈帧里的lock record地址放入对象头里。若操作成功，那就完成了轻量锁操作。如果不成功，说明有线程在竞争，则需要在当前对象上生成重量锁来进行多线程同步，然后将Tag状态改为10，并生成Monitor对象（重量锁对象），对象头里也会放入Monitor对象的地址。最后将当前线程t排队队列中。

    
轻量锁的解锁过程也很简单就是把栈帧里刚才的那个lock record拷贝到对象头里，若替换成功，则解锁完成，若替换不成功，表示在当前线程持有锁的这段时间内，其他线程也竞争过锁，并且发生了锁升级为重量锁，这时需要去Monitor的等待队列中唤醒一个线程去重新竞争锁。

### **4. Lock与synchronized的区别**

1. Lock 拥有Synchronized相同的并发性和内存语义，Lock的实现依赖于cpu级别的指令控制，Synchronized的实现主要由JVM实现控制
2. synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；
3. Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；
4. 通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时（即有大量线程同时竞争），此时Lock的性能要远远优于synchronized。所以说，在具体使用时要根据适当情况选择。

##### **两者在概念上的区别：**
    1. 两者都是可重入的锁；
    2. synchronized就不是可中断锁，而Lock是可中断锁；
    3. synchronized是非公平锁，而lock提供公平锁的实现；
    4. Lock提供读写两种锁操作；
    
##### **性能比较：**

在JDK1.5中，synchronized的性能是比较低的，线程阻塞和唤醒由操作系统内核完成，频繁的加锁和放锁导致频繁的上下文切换，造成效率低下；因此在多线程环境下，synchronized的吞吐量下降的非常严重。但在JDK1.6时对synchronized进行了很多优化，包括偏向锁、自适应自旋、轻量级锁等措施。

当需要以下高级特性时，才应该使用Lock：可定时的、可轮询的与可中断的锁获取操作，公平队列，或者非块结构的锁。否则，请使用synchronized。

### **ReentrantLock源码实例分析**
 参照：[http://www.cnblogs.com/xrq730/p/4979021.html]  
    
[1]: http://www.cnblogs.com/xrq730/p/4979021.html
[2]: http://blog.csdn.net/hqq2023623/article/details/51011434
[3]: http://blog.csdn.net/abyjun/article/details/48860779
[4]: http://blog.csdn.net/suifeng3051/article/details/52611310