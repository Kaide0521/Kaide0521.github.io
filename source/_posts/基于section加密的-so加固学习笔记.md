---
title: 基于section加密的.so加固学习笔记
date: 2020-01-07 21:38:19
comments: true
toc: true
categories:
- 移动安全
tags: 
- App加固
- so加固

---
标签（空格分隔）： APK逆向与保护

---

## 1. 前言
APK的加固技术研究已有很多年了，有很多成熟的厂商提供相关的服务，加固的目的一方面是为了保护应用不被恶意反编译和篡改，得到应用的源代码。另一方面防止应用中为发现的漏洞被攻击者发现利用。早期的加固技术主要是基于Dex文件实现的，通过将源程序的dex文件加密，在运行的过程中由解壳程序动态加载、解密和运行。但是由于基于dex的加固技术，很容易被破解，在内存中dump出dex文件，因此衍生了基于.so的加固技术，虽然基于.so的加固技术的安全性有所提高，但是道高一尺，魔高一丈，基于各种断点调试的人肉脱壳也是很容易恢复出dex文件，据说目前厂商又在研究基于vmp的加固技术，通过自定义指令和解释器等各种技术实现更高级保护等。作为一名“入坑”不久的菜鸟，在这里不谈那么多，先简单学习下基于so加固的基本原理，安全、逆向任重而道远啦~~~

## 2. so加固的基本思路
前段时间，折腾了下ELF文件，对ELF文件有了基本的认识，虽然有些地方理解的还是比较模糊，比如ELF文件在动态加载、链接的细节方面，看了《深入理解计算机系统》之后，理解的还是不到位，期待神作《程序员的自我修养-链接、加载和库》到货，认真研读，以求理解的更加透彻。

古人学问无遗力，少壮工夫老始成。
纸上得来终觉浅，绝知此事要躬行。
                    ————陆游

看书归看书，还是得彻彻底底实践一遍

首先得感谢这些博主的无私分享
[http://bbs.pediy.com/thread-191649.htm][1]
[http://blog.csdn.net/jiangwei0910410003/article/details/49966719][2]
[http://zke1ev3n.me/2015/12/27/Android-So%E7%AE%80%E5%8D%95%E5%8A%A0%E5%9B%BA/][3]
在这里，菜鸟我也是照着你们的神作，依葫芦画瓢，把整个过程梳理一遍，顺便写写自己的感受。
学习逆向的过程是艰辛的，**But, 人皆向死而生，又有何所惧？**

好了，废话少说，言归正传

先来看实现so加固的两种基本思路：

### **2.1 通过将核心函数实现在自定义section中，并进行加密**
基本的思路是自定义一个section，然后将核心函数的实现放在自定义的section中，并且对其进行加密。然后我们再来看ELF文件的格式：

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzA0MTkyMTIzOTE4?x-oss-process=image/format,png)

我们可以看到其中有两个section，分别是.init_array和.fini_array,前面一个在动态链接库加载到进程映像后执行一些初始化操作，后面的终止的时候执行。

也就是.init_array节中的代码先与程序的Main函数开始之前执行，因此我们只要将解密函数定义成.init_array属性的节中，就可以在main函数开始之前解密我们先已加密的section，正常调用其中的函数。

### **2.2.1 加密过程**

```
/**
     * 实现对so的section进行加密操作
     *
     * @param elfFilePath       输入的ELF文件路径
     * @param outPath           加密操作完成后输出路径
     * @param encodeSectionName 要进行加密的节区名称
     */
    public static void doShell(String elfFilePath, String outPath, String encodeSectionName) {
        byte[] fileContent = Utils.readFile(elfFilePath);
        if (fileContent == null) {
            System.out.println("read file byte failed...");
            return;
        }
        /** 首先将ELF文件解析成 ElfType32的对象格式，ElfType32封装了ELF文件各个部分的属性信息 */
        ElfType32 type_32 = ELFParser.parseElfToType32(fileContent);
        /** 对我们指定的section进行加密操作 */
        doEncryptionSection(fileContent, type_32, encodeSectionName);

        ElfType32 otype_32 = ELFParser.parseElfToType32(fileContent);

        Utils.saveFile(outPath, fileContent);

    }
```

具体加密过程如下：

```
 /**
     * 执行具体的加密操作
     *
     * @param fileByteArys
     * @param type_32
     * @param encodeSectionName
     */
    private static void doEncryptionSection(byte[] fileByteArys, ElfType32 type_32, String encodeSectionName) {

        /** ELFheader 定义了.shstrtab节区在节区头部表中的索引，也就是第几个表项 */
        int shstrab_index = Utils.byte2Short(type_32.hdr.e_shstrndx);

        /** 获取.shstrtab节区的结构,.shstrtab节区保存了各个节区的名称 */
        elf32_shdr shdr = type_32.shdrList.get(shstrab_index);
        /**.shstrtab节区的大小**/
        int shstrab_size = Utils.byte2Int(shdr.sh_size);
        /**.shstrtab节区的偏移，相对于文件**/
        int shstrab_offset = Utils.byte2Int(shdr.sh_offset);
        /** 记录要找的节区偏移**/
        int mySectionOffset = 0;
        /** 记录要找的节区大小**/
        int mySectionSize = 0;

        /** 找到名称为encodeSectionName的节区，并执行加密操作 */
        for(elf32_shdr t_shdr : type_32.shdrList){

            /** t_shdr.sh_name定义了节区名称在.shstrtab节区中的大小偏移，可以理解为索引 **/
            int sectionNameOffset = shstrab_offset + Utils.byte2Int(t_shdr.sh_name);

            /** 如果.shstrtab节区在sectionNameOffset偏移出的字符串与encodeSectionName相等，说明找到了**/
            if (Utils.isEqualByteAry(fileByteArys, sectionNameOffset, encodeSectionName)) {
                /** 这里需要读取section段然后进行数据加密 **/
                mySectionOffset = Utils.byte2Int(t_shdr.sh_offset);
                mySectionSize = Utils.byte2Int(t_shdr.sh_size);
                byte[] sectionAry = Utils.copyBytes(fileByteArys, mySectionOffset, mySectionSize);
                for (int i = 0; i < sectionAry.length; i++) {
                    sectionAry[i] = (byte) (sectionAry[i] ^ 0xFF);
                }
                Utils.replaceByteAry(fileByteArys, mySectionOffset, sectionAry);
            }
        }
        if(mySectionOffset == 0 && mySectionSize == 0){
            throw new IllegalArgumentException("Can not find the section of 'encodeSectionName' !");
        }
        /** 修改Elf Header中的e_entry和e_shoff值 为加密的section的offset和size
         *
         * 为什么要修改Header中的e_entry和e_shoff值呢？
         *
         * 1.方便解密的时候快速定位到加密的section，方便解密
         *
         * 既然修改了Header中的e_entry和e_shoff值，难道程序的加载运行的时候找不到入口地址，不会报错吗？
         *
         * 2.在这里我们又要理解下目标文件的装载视图和链接视图，首先对于动态库来说，程序被加载时，设定的跳转地址是动态连接器的地址
         * 通过GOT表和PLT表实现动态调用，与e_entry无关，另外在装载过程中，用不到链接视图中的一些字段，比如e_shoff
         * 所以这两个字段可以被额外数据填充.
         *
         * **/
        int nSize = mySectionSize/4096 + (mySectionSize%4096 == 0 ? 0 : 1);
        byte[] entry = new byte[4];
        entry = Utils.int2Byte((mySectionSize<<16) + nSize);
        Utils.replaceByteAry(fileByteArys, 24, entry);
        byte[] offsetAry = new byte[4];
        offsetAry = Utils.int2Byte(mySectionOffset);
        Utils.replaceByteAry(fileByteArys, 32, offsetAry);
    }
}
```

### **2.2.2 解密过程**
由于很多细节的描述的代码中一一称述，这里就不啰嗦了

**1.首先我们见解密函数定义为**

    void init_decryption() __attribute__((constructor));
函数声明为__attribute__((constructor))属性会先于Main函数之前执行

**2.然后，获取动态库加载在进程内存映像中的虚拟首地址**

```
unsigned long getLibVPAddr(){
    unsigned long ret = 0;
    char name[] = "libdemo.so";
    char buf[4096], *temp;
    int pid;
    FILE *fp;
    pid = getpid();
    /** 获取当前进程的虚拟地址映射表 **/
    sprintf(buf, "/proc/%d/maps", pid);
    fp = fopen(buf, "r");
    if(fp == NULL)
    {
        puts("open failed");
        goto _error;
    }
    while(fgets(buf, sizeof(buf), fp)){
        /** 找到当前进程加载的so库的虚拟地址 **/
        if(strstr(buf, name)){
            temp = strtok(buf, "-");
            ret = strtoul(temp, NULL, 16);
            break;
        }
    }
    _error:
    fclose(fp);
    return ret;
}
```

**3.执行具体的解密过程**

```
void init_decryption(){
    unsigned int nblock;
    unsigned int nsize;
    unsigned long base;
    unsigned long text_addr;
    /** ELF Header 结构体指针 **/
    Elf32_Ehdr *elfHeader;

    /** 获取加载进内存的so的起始地址 **/
    base = getLibVPAddr();

    /** 获取指定section的偏移值和size **/
    elfHeader = (Elf32_Ehdr *)base;
    /** 在我们加密的过程中e_shoff保存的是目标section的偏移，此时加上基址就是目标section在内存中的虚拟地址 **/
    text_addr = elfHeader->e_shoff + base;
    /** 获取目标section的字节空间大小 **/
    nblock = elfHeader->e_entry >> 16;
    /** 获取目标section所占的页数，每页对应4 X 1024B **/
    nsize = elfHeader->e_entry & 0xffff;

    __android_log_print(ANDROID_LOG_INFO, "JNITag", "nblock =  0x%x,nsize:%d", nblock,nsize);
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "base =  0x%x", text_addr);
    printf("nblock = %d\n", nblock);

    /** 修改内存的操作权限，因为不同的虚拟地址空间的内存段是有权限限制的，比如代码段是只读，数据段：可读可写**/
    if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
        puts("mem privilege change failed");
        __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
    }
    unsigned int i;
    /** 执行具体的解密操作 **/
    for(i=0;i< nblock; i++){
        char *addr = (char*)(text_addr + i);
        *addr = ~(*addr);
    }
    /** 解密完成后，要修改会权限 **/
    if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC) != 0){
        puts("mem privilege change failed");
    }
    puts("Decrypt success");
}
```