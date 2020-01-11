---
title: IDA Pro脱dex壳初探
date: 2020-01-07 21:45:43
comments: true
toc: true
categories:
- 移动安全
tags: 
- App脱壳
- IDA Pro
- Apk逆向

---
### IDA Pro脱dex壳初探

标签（空格分隔）： Apk逆向

---

### 1. 前言
关于IDA Pro的介绍，已不用多说了，是目前最棒的一个静态反编译软件，编程天才的杰作。这里的脱壳实战我们以阿里比赛样本AliCrackme_3.apk样本为例。

### 2. 调试环境的搭建

#### **2.1 安装IDA Pro**

[下载地址][1]
直接解压运行

#### **2.2 使用IDA进行调试设置**

在IDA安装目录dbgsrv下获取android_server命令文件
运行命令：

    adb root
    adb remount
    adb push android_server /data   #将文件拷贝到手机data目录
    adb shell
    cd /data
    chmod 755 android_server    #更改权限，赋予可执行权限
    ./android_server            #运行
    root@Che1:/data # ./android_server
    IDA Android 32-bit remote debug server(ST) v1.17. Hex-Rays (c)     2004-2014
    Listening on port #23946...

错误1：error: only position independent executables (PIE) are supported
这个主要是Android5.0以上的编译选项默认开启了pie，在5.0以下编译的原生应用不能运行，有两种解决办法，一种是用Android5.0以下的手机进行操作，还有一种就是用IDA6.6+版本即可。

#### **2.3 配置端口转发**

这里开始监听了设备的23946端口，那么如果要想让IDA和这个android_server进行通信，那么必须让PC端的IDA也连上这个端口，那么这时候就需要借助于adb的一个命令了：
**adb forward tcp:远端设备端口号(进行调试程序端) tcp:本地设备端口(被调试程序端)**
另起一个cmd,运行命令
    adb forward tcp:23946 tcp:23946

#### **2.4 以debug模式启动apk**

    adb shell am start -D -n com.ali.tg.testapp/.MainActivity

#### **2.5 使用IDA进行连接**

选择Debugger->Attach->Remote ARMLinux/Android debugger

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzMjIxMDc3?x-oss-process=image/format,png)

点击OK按钮，按ctrl+F搜索com.ali进行进程附加操作

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzMjM0MjQx?x-oss-process=image/format,png)

#### **2.6 启动jdb调试器**

    jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700
处于等待状态

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200107215447785.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaWRlMjAxMg==,size_16,color_FFFFFF,t_70)

#### **2.7 下断点**

给dvmDexFileOpenPartial函数下断点，原因如下：
dvmDexFileOpenPartial这个函数是最终分析dex文件，加载到内存中的函：

    int dvmDexFileOpenPartial(const void* addr, int len, DvmDex** ppDvmDex);

第一个参数就是dex内存起始地址，第二个参数就是dex大小。

找到dvmDexFileOpenPartial的函数地址
IDA静态分析libdvm.so得到这个函数的相对地址为Ox46CCC

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzMzA1OTg0?x-oss-process=image/format,png)

按Ctrl + S查看libdvm.so在虚拟内存中的映射位置

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzMzI2MDUx?x-oss-process=image/format,png)

然后将两者相加得到绝对地址：Ox46CCC + Ox415CA000 = Ox41610CCC，使用G键，跳转：

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzMzQxNzQ4?x-oss-process=image/format,png)


#### **2.8 点击运行按钮或者F9,然后按F8进行单步调试**

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzMzU0OTg5?x-oss-process=image/format,png)

其中R0和R1就是传递给dvmDexFileOpenPartial的参数，R0对应于addr，R1对应于len,也就是dex文件的长度

然后按下Shirt+F2 调出IDA的脚本运行界面

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzNDIwOTM5?x-oss-process=image/format,png)

然后使用jeb工具打开dump到的dex文件

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzIxMjEzNDM3MDQ2?x-oss-process=image/format,png)

于是得到了被加固的dex文件

#### **2.9 总结**

该样本直接使用dvmDexFileOpenPartial函数实现dex的文件加载，我们直接可在改函数出下断点，然后从内存中直接dump出dex文件，是一个比较简单样例，也算是复习下IDA pro调试的基本步骤，然后真正的现实情况并没有这么简单，很多多应用会采取一系列额反调试手段，比如：

**SO的反调试**
IDA是使用android_server在root环境下注入到被调试的进程中，那么这里用到一个技术就是Linux中的ptrace
那么Android中如果一个进程被另外一个进程ptrace了之后，在他的status文件中有一个字段：TracerPid 可以标识是被哪个进程trace了，我们可以使用命令查看我们的被调试的进行信息：
status文件在：/proc/[pid]/status
如果检测到这个字段不为0，说明当前so文件正在被调试，这是一种常用的反调试技术

还有有些加固的apk对dex文件的加载不一定就走dvmDexFileOpenPartial进行加载，会自己实现dex文件的加载函数，但不管怎么样，最终还是会使用mmap内存映射函数，所以下断点的地方又有所区别。