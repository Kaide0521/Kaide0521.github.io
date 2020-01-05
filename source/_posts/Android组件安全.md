---
title: Android组件安全
date: 2020-01-05 21:24:56
comments: true
toc: true
categories:
- 移动安全
tags: 
- android安全

---
## **Android组件安全**

做过Android开发的攻城狮都知道Android四大组件，在开发过程中打交道最多的也是这四大组件，在开发Android应用的过程中我们最基本的要求是要对这四大组件有一个清晰的认识，包括组件的功能特性、生命周期、与系统的交互模型，以及组件之间如何配合使用等等。本文内容主要是关于组件的安全模型以及如何正确的使用组件来降低应用的安全风险的一个学习笔记。


### **一、Android四大组件简介**

#### **1. Service组件**

Service是没有界面且能长时间运行于后台的应用组件，是Android中实现程序后台运行的解决方案，非常适合用于去执行那些不需要和用户交互而且还要求长期运行的任务。这也是为了不阻塞主进程，然而主进程依然能够流畅的处理UI事件，提供良好的用户体验。

根据Service的生命周期不同又分为两种类型：

	（1）start类型：应用程序组件（如activity）调用startService()方法启动服务时，服务处于started状态
	（2）bound类型：组件调用bindService()方法绑定到服务时，服务处于bound状态

**两者的区别：**调用startService()启动的Service其生命周期与启动它的组件无关，调用bindService()启动的Service会在启动它的组件消亡后消亡

**1.1 Service安全**

关于Service安全规范请参照博客：[http://www.droidsec.cn/android-service-security/](http://www.droidsec.cn/android-service-security/)

#### **2. Activity组件**

Activity是一个Android应用程序和用户进行交互界面的Android组件，相当于应用程序的一个交互屏幕，Activity之间通过Intent进行通信。下图是Activity的生命周期变迁图

**2.1 Activity生命周期**

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-bXGiHv3W-1578230663807)(http://i.imgur.com/CH4adYo.jpg)]

**onCreate :** Activity被创建时调用，是生命周期第一个被调用的方法

**onStart :**表示Activity正在启动，还没有到达可见、交互的地步

**onResume :**表示Activity已在前台可见，可与用户交互

**onPause :** 表示Activity正在停止

**onStop :** 一般在onPause方法执行完成直接执行，表示Activity即将停止或者完全被覆盖，此时Activity不可见，仅在后台运行

**onRestart :**表示Activity正在重新启动，当Activity由不可见变为可见状态时，该方法被回调
 
**onDestroy :**此时Activity正在被销毁，也是生命周期最后一个执行的方法，一般我们可以在此方法中做一些回收工作和最终的资源释放。

**2.2 Activity组件的加载模式**

- **standard模式**：无论Activity栈中是否存在改Activity实例，系统都将创建一个实例
- **singleTop模式**：如果目标activity的实例已经存在于栈顶，系统会直接使用该实例，不会重新创建，否则会创建一个实例，入栈
- **singleTask模式**：在同一应用程序中，如果目标activity的实例已经存在于栈顶，系统会直接使用该实例，不会重新创建，否则弹出栈顶实例，直到使目标activity位于栈顶，在不同应用程序中启动Activity时，会创建一个新的task栈，用于保存被启动的Activity实例
- **singleInstance模式**：只有一个实例，并且这个实例独立运行在一个task栈中，这个task只有这个实例，不允许有别的Activity存在

**2.2 Activity组件权限**

Activity权限的使用主要用于从其他应用中调用该应用的Activity进行一些操作，比如第三方登录：进行权限的设置后可以调用该应用中的Activity进行操作。下图是权限的格式定义：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-SAbH4nAO-1578230663808)(http://i.imgur.com/MSPOUeC.png)]

    android:protectionLevel 一共有四种

- **normal：**表示权限是低风险的，不会对系统、用户或其他应用程序造成危害

- **dangerous：**比normal级别要高一等级,可能会赋予应用程序访问敏感数据或控制设备等功能

- **signature：**表示只有当应用程序所用数字签名与声明该权限的应用程序所用数字签名相同时，才能将权限授给它

- **signatureOrSystem ：**表示将权限授给具有相同数字签名的应用程序或android 系统映像/系统app。这一保护级别适和于非常特殊的情况，比如多个供应商需要通过系统映像共享功能时

**2.2 exported属性**

一个Activity组件能否被外部应用启动取决于此属性，设置为true时Activity可以被外部应用启动，设置为false则不能，此时Activity只能被自身app启动。（同user id或者root也能启动）

没有配置intent-filter属性exported默认为false（没有filter只能通过明确的类名来启动activity故相当于只有程序本身能启动），配置了intent-filter属性exported默认为true。

exported属性只是用于限制Activity是否暴露给其他app，通过配置文件中的权限申明也可以限制外部启动activity。

**2.3 activity的声明**

    <activity
            android:name="Activity"
            android:label="@string/title_activity"
            android:permission="packagename.permission.Activity"
            >
            <intent-filter>
                <category android:name="android.intent.category.DEFAULT"/>
                <action android:name="packagename.intent.action.Activity"/>
            </intent-filter>
        </activity>

配置了intent-filter属性，则exported为true，说明别的应用程序可以在外部启动该Activity，可根据action的name发起隐式intent，启动声明了该action的Activity

**2.4 activity的分类**

**2.4.1 private activity**

私有Activity不应被其他应用启动相对是安全的
关于private activity的使用和设置规范请参照：[http://www.bugsec.org/7195.html](http://www.bugsec.org/7195.html)

**2.4.2 public activity**

公开暴露的Activity组件，可以被任意应用启动
关于public activity的使用和设置规范请参照：[http://www.bugsec.org/7195.html](http://www.bugsec.org/7195.html)

#### **3. Broadcast Recevier组件**

Broadcast Recevier 广播接收器是一个专注于接收广播通知信息，并做出对应处理的组件，广播机制是一个典型的发布—订阅模式，即观察者模式。 广播接收器没有用户界面。然而，它们可以启动一个activity来响应它们收到的信息，或者用NotificationManager来通知用户。通知可以用很多种方式来吸引用户的注意力──闪动背灯、震动、播放声音等等。一般来说是在状态栏上放一个持久的图标，用户可以打开它并获取消息。

**3.1 广播注册形式**

**静态注册：**

静态注册是在AndroidManifest.xml中声明广播注册，直接在Manifest.xml文件的<application>节点中配置广播接收者。

    <receiver android:name=".MyBroadCastReceiver">  
            <!-- android:priority属性是设置此接收者的优先级（从-1000到1000） -->
            <intent-filter android:priority="20">
            <actionandroid:name="android.provider.Telephony.SMS_RECEIVED"/>  
            </intent-filter>  
	</receiver>

还要在<application>同级的位置配置可能使用到的权限

	<uses-permission android:name="android.permission.RECEIVE_SMS"></uses-permission>


**动态注册：**

通过使用代码进行注册


**区别：**

- 第一种是常驻型，也就是说当应用程序关闭后，如果有信息广播来，程序也会被系统调用自动运行

- 第二种是非常驻型广播，也就是说广播跟随程序的生命周期

关于Broadcast Recevier安全编码规范，参照：[http://www.droidsec.cn/android-broadcast-security/](http://www.droidsec.cn/android-broadcast-security/)


#### **3. Content Provider组件**

为存储和获取数据提供统一的接口，可以在不同的应用程序之间共享数据。使用ContentProvider，应用程序可以实现数据共享，android内置的许多数据都是使用ContentProvider形式，供开发者调用的(如视频，音频，图片，通讯录等)

关于Content Provider组件的安全问题，参照：[http://blog.csdn.net/alimobilesecurity/article/details/51564968](http://blog.csdn.net/alimobilesecurity/article/details/51564968)

### **参考博文**

[http://www.cnblogs.com/smyhvae/p/4070518.html](http://www.cnblogs.com/smyhvae/p/4070518.html)

[http://blog.csdn.net/javazejian/article/details/51932554](http://blog.csdn.net/javazejian/article/details/51932554)