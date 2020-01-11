---
title: IDA Pro脱dex壳过反调试
date: 2020-01-07 21:47:34
comments: true
toc: true
categories:
- 移动安全
tags: 
- App脱壳
- IDA Pro
- Apk逆向

---
### IDA Pro脱dex壳过反调试

标签（空格分隔）： Apk逆向

---

### **1. 前言**
之前总结了IDA pro脱壳的基本步骤，包括调试步骤和在libdvm.so的dvmDexFileOpenPartial函数出下断点，从内存中dump出dex文件。
这次以360一代加壳为例，试着在mmap出下断点和过一些基本的反调试技术。

### **2. 脱壳环境搭建**

这里的环境搭建和上一篇博客一样[http://blog.csdn.net/daide2012/article/details/75675210][1]
启动android_server
端口转发
以调试模式启动应用
打开IDA附件进程
设置Debugger Option选项
运行jdb调试

### **3. 开始脱壳**

**3.1 下断点**

找到mmap函数，在此下断点，一般我们读取文件都是使用fopen和fgets函数，但是360壳通过mmap函数读取/proc/pid/status，来检查TracerPid，因此我们应该在mmap函数处下断点

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1MjUwODY5?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1MzA5NDky?x-oss-process=image/format,png)

单步调试过程中发现一直重复执行一段代码如下：

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1MzIxMDMz?x-oss-process=image/format,png)

我们发现，这是一段循环，测试是增加破解难度，故意设置的，因此我们将循环条件的寄存器值进行更改，让其跳出循环，否则要进行多次单步，按F8按的手痛

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1MzM3MjI1?x-oss-process=image/format,png)

在寄存器列表中点击鼠标右键，修改寄存器R11的值为Ox000000A7,提早退出循环，然后单步调试
注意单步都标号为loc_760BF08C处的时候，要进入该函数，然后继续单独调试，我试了好几次，每次执行到此处的代码，自动退出了，因此反调试的实现肯定是由该段代码实现，所以要进入调试

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1MzU3MjUw?x-oss-process=image/format,png)

继续单独调试

过程中会发现程序读取/proc/pid/status的内容，比较TracerPid的值与0的大小

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1NDEzOTEz?x-oss-process=image/format,png)

此时R0寄存器的值恰好是16进制的值11EF,对应的TracerPid为4591，因此要修改R0寄存器的值为0,绕过范调试检测，然后继续单步
执行完后，按F9继续运行mmap函数，因为此时读取的并不一定是dex文件，动态库在装载的时候很多都调用了mmap函数，所以回到mmap继续调试

注意，程序会多次进行反调试检测，所以在进入反调试的函数处建议下断点，这样可以快速的F9到反调试检测的位置。我在调试的过程中一共按了6,7次才把反调试检测过完，中间按错了好几次，每次都得重头再来，耐心很重要啊！！!

过了反调试之后，在memcmp处下断点，不断的F9，同时在Hex View窗口主要出现dex.035的字符

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1NDI5MzU0?x-oss-process=image/format,png)

终于看到dex.035这个字样了，下面的工作使用脚本将dex文件从内存中dump出

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzI4MjI1NDQ0MTg0?x-oss-process=image/format,png)

之后即可直接采用脚本dump出dex文件


  [1]: http://blog.csdn.net/daide2012/article/details/75675210