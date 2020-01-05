---
title: Android Studio调试smali代码
date: 2020-01-05 21:27:12
comments: true
toc: true
categories:
- 移动安全
tags: 
- android安全
- smali代码
- APK逆向分析

---
### Android Studio调试smali代码

标签： APK逆向分析


## 1. 前言
经过一段时间的学习，现在总结下针对APK逆向的一些基本调试技术，以阿里移动安全比赛题目为例

## 2. 使用Android Studio调试smali代码

步骤一：下载安全Android Stuio,下载地址[http://www.android-studio.org/][1]

步骤二：下载插件smalidea
地址: [https://bitbucket.org/JesusFreke/smali/downloads][2]

步骤三：下载完成后，打开Android studio的Settings——>Plugins，选择 Install plugin from disk

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDgzNjUzMDA3?x-oss-process=image/format,png)

步骤四：反编译apk，修改AndroidManifest.xml文件的属性android:debuggable="true"

    java -jar apktool.jar d -d ./apk/AliCrackme_1.apk -o out

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDgzNzEzMzk0?x-oss-process=image/format,png)

修改完成之后，回编译apk

    java -jar apktool.jar b -d out -o debug.apk

回编之后，进行签名

    java -jar .\sign\signapk.jar .\sign\testkey.x509.pem .\sign\testkey.pk8 debug.apk debug.sig.apk

步骤五：安装签名之后的应用

    adb install debug.sig.apk

使用backsmali得到apk的smali代码

    java -jar backsmali.jar debug.sig.apk

将得到的smali代码导入Android Studio中

步骤六：配置远程调试的选项，选择Run–>Edit Configurations：

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDgzNzM4NzA0?x-oss-process=image/format,png)

增加一个Remote调试的调试选项，端口选择:8700

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDgzNzU3OTg4?x-oss-process=image/format,png)

步骤七：下好断点，针对本APK我们在button的onclik函数出下断点（点击鼠标右键）

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDgzODI0Nzg2?x-oss-process=image/format,png)

步骤八：以调试状态启动app

    adb shell am start -D -n com.example.simpleencryption/.MainActivity
下好断点之后Run->Debug
执行完

    invoke-virtual {v6}, Lcom/example/simpleencryption/MainActivity;->getTableFromPic()Ljava/lang/String;

得到图6内容

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDgzOTE5MzM4?x-oss-process=image/format,png)

通过分析我们大体知道个整个函数的执行逻辑

 1. 通过MainActivity中的getTableFromPic方法，获取一个类似密码表的东西
 2. 通过MainActivity中的getPwdFromPic方法，获取正确的密码"义弓么丸广之"
 3. 获取我们输入内容的utf-8的字节码，然后调用MainActivity的access$0方法，获取加密之后的内容

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDgzOTQzNzIy?x-oss-process=image/format,png)

 4. 再看MainActivity的access$0方法的实现

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDg0MDE0MTcw?x-oss-process=image/format,png)
 
里面又调用了bytesToAliSmsCode，传了两个参数，一个是我们输入的qwer的bytes数组，一个是获取的密码表

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzE5MDg0MTI3Mzk5?x-oss-process=image/format,png)

具体的逻辑如下

```
.method private static bytesToAliSmsCode(Ljava/lang/String;[B)Ljava/lang/String;
    .registers 5
    .param p0, "table"    # Ljava/lang/String;
    .param p1, "data"    # [B

    .prologue
    .line 144
    new-instance v1, Ljava/lang/StringBuilder;

    invoke-direct {v1}, Ljava/lang/StringBuilder;-><init>()V

    .line 145
    .local v1, "sb":Ljava/lang/StringBuilder;    //创建一个StringBuilder的变量sb
    const/4 v0, 0x0

    .local v0, "i":I    // 定义一个int变量i = 0
    :goto_6
    array-length v2, p1

    if-lt v0, v2, :cond_e   // 比较i和data数组的长度，如果小于，跳到cond_e

    .line 148
    invoke-virtual {v1}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v2

    return-object v2

    .line 146
    :cond_e
    aget-byte v2, p1, v0   // 取出data[i]

    and-int/lit16 v2, v2, 0xff

    invoke-virtual {p0, v2}, Ljava/lang/String;->charAt(I)C   // 获取data[i]在table数组的字符

    move-result v2
    // 调用sb的append函数进行拼接
    invoke-virtual {v1, v2}, Ljava/lang/StringBuilder;->append(C)Ljava/lang/StringBuilder;

    .line 145
    add-int/lit8 v0, v0, 0x1

    goto :goto_6
.end method
```

整体逻辑就是一个循环，将输入字符转化为utf-8格式的bye数组，然后取出data[i]table表中所对应的字符，然后拼接返回
 5. 返回字符串与正确密码"义弓么丸广之"比较
 invoke-virtual {v4, v6}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z
 此时我们输入的qwer加密后与"义弓么丸广之"
肯定不相等，所以这里我们可以想办法是的输入的字符串加密等于"义弓么丸广之"

**怎么办呢？**

我们可以将bytesToAliSmsCode进行反向实现，

```
public static void main(String args[]){
		String table = "一乙二十丁厂七卜人入八九几儿了力乃刀又三于干亏士工土才寸下大丈与万上小口巾山千乞川亿个勺久凡及夕丸么广亡门义之尸弓己已子卫也女飞刃习叉马乡丰王井开夫天无元专云扎艺木五支厅不太犬区历尤友匹车巨牙屯比互切瓦止少日中冈贝内水见午牛手毛气升长仁什片仆化仇币仍仅斤爪反介父从今凶分乏公仓月氏勿欠风丹匀乌凤勾文六方火为斗忆订计户认心尺引丑巴孔队办以允予劝双书幻玉刊示末未击打巧正扑扒功扔去甘世古节本术可丙左厉右石布龙平灭轧东卡北占业旧帅归且旦目叶甲申叮电号田由史只央兄叼叫另叨叹四生失禾丘付仗代仙们仪白仔他斥瓜乎丛令用甩印乐";
		String dataString = "义弓么丸广之";
		String passwd  = "";
		for(int i = 0 ; i < dataString.length(); i++){
			passwd += (char)table.indexOf(dataString.charAt(i));
		}
		System.out.println("passwd = " + passwd);
	}
```

运行出的结果为passwd = 581026就是我们输入框里面要输入的字符串

  [1]: http://www.android-studio.org/
  [2]: https://bitbucket.org/JesusFreke/smali/downloads