---
title: App虚拟化技术初探
date: 2020-01-07 21:43:45
comments: true
toc: true
categories:
- 移动安全
tags: 
- App虚拟化
- 应用多开

---
### **Plugin Technology Background**

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAyOTM3MTA1?x-oss-process=image/format,png)

App插件开发技术或App虚拟化技术以及App热修复技术，最近几年非常火热和流行，图中列举了两种主要的需求，第一种需求是很多时候我们想在Android手机上同时登陆多个社交应用，比如QQ或者微信，在Android原生系统中肯定是无法支持的，只有登出一个账号后然后在换另外一个，因此衍生APP虚拟化技术；另外一种需求是一个大的APK文件我想分批次发布，防止一次性发布下载时间过长而损害用户体验，因此我把不同的功能以插件的形式发布，这个插件同样是APK文件，同样对于某一种功能的升级或者bug的修复都可以采用这种技术来实现。

### **What is Android Plugin Technology?**

 - Launch an APK file within an Android app.
在一个APP内部启动一个APP
 - In the unrooted device.
不需要root手机
 - “Host App” = Android app.
 - “Plugin” = APK file.
 - No need to install the plugin.
这里的无需安装主要是指对系统来说，并没有实地安装，，创建沙箱环境，以用户的视角来看还是与host app一样，点击安装，然后正常运行

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzMjAzNTQw?x-oss-process=image/format,png)


### **现有的插件框架** 

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzMjMxODQ1?x-oss-process=image/format,png)

现有的能够实现上述需求的框架主要有DroidPlugin、VirtualApp、DynamicAPK
典型的应用有Parallel Space，截止2017年下载了达到千万级别
虽然很多类型的这种框架，但是他们得底层实现机制基本相同

### **Demystify Plugin Technology**

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzMzM5NTE0?x-oss-process=image/format,png)

插件技术最主要的中原理就是，为了让插件中的方法能够正确的调用。Host APP把所有的插件和系统（Android Framework交互）的方法都在内部Hook并替换掉了。

**为什么这样做呢？** 

因为，这个是一个插件系统，Activity是虚拟Proxy出来的。四大组件，并不是真正的，即Context拿到不一定是真的。系统也不会有此类的回调。所以，很多单例都需要执行Context才能使用。比如：InputMethodManager.getInstance(Context context)；这时，插件中的代码将不能够直接运行。当然，还有其他原因。

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzNDA5NDM3?x-oss-process=image/format,png)

### **动态加载运行一个Activity的基本机制**

为了进一步的理解这种插件运行机制中的HOOK技术，我们来看一个简单的实例，即如何动态加载运行一个Apk中的Activity,

既然说到动态加载，我们很容易想到Davilk虚拟机中的类加载器

(1) DexClassLoader可以加载任何路径的apk/dex/jar/zip
(2) PathClassLoader只能加载缓存在/data/app中的apk，也就是已经安装到手机中的apk

那么如何加载一个apk并运行其中的Activity呢？我们很容易想到使用如下代码实现

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzNTAzMTc2?x-oss-process=image/format,png)

这样显然是不行的
因为Android中的四大组件都有一个特点就是他们有自己的启动流程和生命周期，我们使用DexClassLoader加载进来的Activity是不会涉及到任何启动流程和生命周期的概念，说白了，就是一个普普通通的类，所以启动肯定会出错。
那么如何解决呢？

#### **思路一：替换LoadedApk中mClassLoader**

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzNjIxMDQ4?x-oss-process=image/format,png)

由于这个mClassLoader是非static类型的引用，因此我们通过反射的机制无法更改当前运行时数据，那么该怎么办呢？我们再看ActivityThread这个类

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzNjQzODYz?x-oss-process=image/format,png)

这个类中有一个静态变量，我们可以通过反射这个类，来获取LoadedApk对象，然后更改其mClassLoader

#### **思路二：更新DexPathList数据结构**

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzNzMwMjM3?x-oss-process=image/format,png)

APK在启动的过程中加载的dex文件都存放在一个叫dexElements的数据结构中，我们通过将插件APK的dex文件解析后的数据结构也放入此数据中，使其成为host App的一部分，那么host app在启动其中的Activity的时候就具有上下文环境了，相应的组件也会有生命周期

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzNzUxODI1?x-oss-process=image/format,png)

#### **思路三：使用静态代理**

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzODI0NTIz?x-oss-process=image/format,png)

基本思路是使用一个在host app中具有生命周期和执行上下文的Activity代理插件Activity执行其各个生命周期的回调函数

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzODQ5NzQw?x-oss-process=image/format,png)

#### **之前三种方式共有的缺陷**

(1) 需要在宿主应用的Manifest文件中注册要启动的Plugin中的组件，权限等信息，只能加载运行特定已知的apk，因此不可能将所有需要以插件方式进行运行的APK中的组件全部进行注册

#### **思路四：使用hook技术代理实现**

我们先看下启动一个Activity的基本过程，图中已说明详细，这里不再赘述了

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTAzOTQ0NzY4?x-oss-process=image/format,png)

由于前三种方式的缺陷，第四重方式通过预先在Host APP的Manifest文件中预注册一些启动插件app会用到的一些Dummy 组件，也就是图中的StubActivity，当使用intent启动一个PluginActivity的时候，使用Hook拦截此消息，将PluginActivity更换为StubActivity，通过Binder系统调用进行进程间过程调用，ActivityManngerService执行Activity的栈管理、验证等过程，由于我们使用将PluginActivity更换为StubActivity，而StubActivity又在Host APP的Manifest文件中预注册了。所以验证会通过，然后ActivityManngerService将控制权移交给APP进程（同样通过进程间的RPC调用），然后APP进程开始执行真正创建Activity的过程，这创建之前又通过hook将StubActivity更换为PluginActivity，启动真正要启动的Activity，因此这样就可以瞒天过海，欺上瞒下，骗过System_server进程，同时使得启动的Activity又具有相应的生命周期，整个过程还是比较直观的。

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNjI0MTA0MDE4Mjg0?x-oss-process=image/format,png)