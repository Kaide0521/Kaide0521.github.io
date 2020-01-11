---
title: 基于函数加密的.so加固学习笔记
date: 2020-01-07 21:41:53
comments: true
toc: true
categories:
- 移动安全
tags: 
- App加固
- so加固

---
标签（空格分隔）： Apk逆向与保护

---

### **1. 前言**

之前学习了简单对ELF文件中的Section进行加密实现简单暴力的加固，这个section是自定义的，一般很容易被识别出来，更好的做法是对函数进行加密。既然是对函数加密，我们就需要找到函数的地址和大小。既然是找到函数的地址，必然是基于ELF文件的转载视图来进行操作的。

### **2. 基本思路**

### **2.1 如何找到要加密的函数**

基于section的加密，我们很容易根据elf文件的header找到.shstrtab节区，从而找到我们要对其进行加密的section，此处我们该如何找到函数的定义呢？这就要求对ELF文件的结构要有进一步较深的认识，通过ELF文件的格式我们知道，ELF文件中存在.dynsym和.dynstr这两个节区，.dynsym主要是描述动态链接的符号或者函数结构信息，.dynstr主要描述动态链接的符号字符串或函数名称。

那么我们可以通过type来获取.dynsym和.dynstr吗？

答案是否定的，因为不同的section的type可能相同，比如.dynstr和.shstrtab的type都是STRTAB
在ELF中，每个函数的结构描述放在.dynsym中，函数的名称放在.dynstr中，我们怎么去查找它们的对应关系呢？
有一个叫.hash的section，这个节描述的是一个hash表，其结构图如下：

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwNzA1MjA0ODM4MDY2?x-oss-process=image/format,png)

bucket 数组包含 nbucket 个项目,chain 数组包含 nchain 个项目,下标都是从 0 开始。bucket 和 chain 中都保存符号表索引。chain 表项和符号表存在对应。符号表项的数目应该和 nchain 相等,所以符号表的索引也可用来选取 chain 表项。

假如符号s = "aaaa",hash(s) = 5,则index = bucket[5 % 10] = 5;
则给出其存放的初始索引为5，如果发现索引为index出的符号不是s,那么chain[index]则给出与s具有相同哈希值的符号的索引（可以理解为不同字符串通过哈希映射取模会产生冲突），因此一直这么搜索下去，知道找到符号s所在的索引为止。

### **2.1 如何找到.hash节区**

通过对ELF文件的研究，我们发现，当so被加载后，其section结构对于加载视图来说是无意义的，so的加载视图中有个段叫做.dynmic，此段包含了动态链接信息，因此.dynsym和.dynstr，.hash节区肯定包含在这里（一个段可以包含多个节区）,因此我们只要找到装载视图中的.dynamic段，然后解析成dynmic结构信息，然后根据type类型找到.dynsym和.dynstr，.hash节区
**具体过程：**
(1) 找到.dynamic段；.dynamic段的描述可根据程序头部表来找到，该段在程序头部表中用符号DYNAMIC来标记

(2) 找到.dynamic段之后解析成相关的数据结构

```
typedef struct dynamic {
  Elf32_Sword d_tag;    /* Dynamic entry type */
  /* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
  union {
    Elf32_Sword d_val;
    Elf32_Addr d_ptr;
  } d_un;
  /* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
} Elf32_Dyn;
```

对于每个这种类型的结构，d_tag控制了d_un的解释含义：

 - d_val 此 Elf32_Word 对象表示一个整数值，可以有多种解释。
 - d_ptr 此 Elf32_Addr 对象代表程序的虚拟地址。如前所述，文件的虚拟地址可能与执行过程中的内存虚地址不匹配。在解释包含于动态结构中的地址时，动态链接程序基于原来文件值和内存基地址计算实际地址。为了保持一致性，文件中不包含用来“纠正”动态结构中重定位项地址的重定位项目。

(3) 根据d_tag标志，找到.dynsym和.dynstr，.hash节区

为什么这里又可以根据type类型来找到相应的节区呢？因为此时我们在.dynamic段里面进行匹配，type相对来说是唯一的

### **2. 加密具体过程**

```
/**
     * 实现对so的函数进行加密操作
     *
     * @param elfFilePath  输入的ELF文件路径
     * @param outPath      加密操作完成后输出路径
     * @param functionName 要进行加密的函数名称
     */
    public static void doFunctionEncryption(String elfFilePath, String outPath, String functionName) {
        byte[] fileContent = Utils.readFile(elfFilePath);
        if (fileContent == null) {
            System.out.println("read file byte failed...");
            return;
        }
        /** 首先将ELF文件解析成 ElfType32的对象格式，ElfType32封装了ELF文件各个部分的属性信息 */
        ElfType32 type_32 = ELFParser.parseElfToType32(fileContent);
        /** 对我们指定的section进行加密操作 */
        doEncryptionFunction(fileContent, type_32, functionName);

        ElfType32 otype_32 = ELFParser.parseElfToType32(fileContent);

        Utils.saveFile(outPath, fileContent);

    }
```

**实现加密操作：**

```
private static void doEncryptionFunction(byte[] fileByteArys, ElfType32 type_32, String functionName) {
        /** 得到Dynamic段的偏移值和大小**/
        int dy_offset = 0, dy_size = 0;
        for (elf32_phdr phdr : type_32.phdrList) {
            if (Utils.byte2Int(phdr.p_type) == ElfType32.PT_DYNAMIC) {
                dy_offset = Utils.byte2Int(phdr.p_offset);
                dy_size = Utils.byte2Int(phdr.p_filesz);
                break;
            }
        }
        System.out.println("dy_size:" + dy_size);
        /** 解析得到Dynamic段中的每个节区**/
        ELFParser.getDynamicTableList(fileByteArys, dy_size, dy_offset, type_32);
        /** .dynstr的大小和偏移**/
        int dynstr_size = 0, dynstr_offset = 0;
        /** .dynsym的偏移**/
        int dynsym_offset = 0;
        /** .hash的偏移**/
        int hash_offset = 0;
        /** 根据type解析每个字段的值 **/
        for (Elf32_dyn dyn : type_32.dynList) {
            if (Utils.byte2Int(dyn.d_tag) == ElfType32.DT_HASH) {
                hash_offset = Utils.byte2Int(dyn.d_ptr);
            } else if (Utils.byte2Int(dyn.d_tag) == ElfType32.DT_STRTAB) {
                System.out.println("strtab:" + dyn);
                dynstr_offset = Utils.byte2Int(dyn.d_ptr);
            } else if (Utils.byte2Int(dyn.d_tag) == ElfType32.DT_SYMTAB) {
                System.out.println("systab:" + dyn);
                dynsym_offset = Utils.byte2Int(dyn.d_ptr);
            } else if (Utils.byte2Int(dyn.d_tag) == ElfType32.DT_STRSZ) {
                System.out.println("strsz:" + dyn);
                dynstr_size = Utils.byte2Int(dyn.d_val);
            }
        }
        /** 获取.dynstr节的字节数组，也就是符号字符数据集合 **/
        byte[] dynstr = Utils.copyBytes(fileByteArys, dynstr_offset, dynstr_size);
        int funcIndex = 0;
        for (Elf32_dyn dyn : type_32.dynList) {
            if (Utils.byte2Int(dyn.d_tag) == ElfType32.DT_HASH) {
                int nbucket = Utils.byte2Int(Utils.copyBytes(fileByteArys, hash_offset, 4));
                int nchian = Utils.byte2Int(Utils.copyBytes(fileByteArys, hash_offset + 4, 4));
                int hash = (int) Utils.getElfHash(functionName.getBytes());
                hash = (hash % nbucket);
                /** 得到函数的索引值，这里的8是读取nbucket和nchian的两个值**/
                funcIndex = Utils.byte2Int(Utils.copyBytes(fileByteArys, hash_offset + hash * 4 + 8, 4));
                System.out.println("nbucket:" + nbucket + ",hash:" + hash + ",funcIndex:" + funcIndex + ",chian:" + nchian);
                System.out.println("sym:" + Utils.bytes2HexString(Utils.int2Byte(dynsym_offset)));
                System.out.println("hash:" + Utils.bytes2HexString(Utils.int2Byte(hash_offset)));
                int dynsym_size = 16;
                byte[] des = new byte[dynsym_size];
                /** 根据函数的索引值在.dynsym节中找到其具体的描述结构信息**/
                System.arraycopy(fileByteArys, dynsym_offset + funcIndex * dynsym_size, des, 0, dynsym_size);
                Elf32_Sym sym = ELFParser.parseSymbolTable(des);
                System.out.println("sym:" + sym);
                boolean isFindFunc = Utils.isEqualByteAry(dynstr, Utils.byte2Int(sym.st_name), functionName);
                while(true){
                    if(isFindFunc){
                        System.out.println("find func...");
                        /** 得到函数体的字节大小 **/
                        int functionSize = Utils.byte2Int(sym.st_size);
                        /** 得到函数体的偏移量 **/
                        int functionOffset = Utils.byte2Int(sym.st_value);
                        System.out.println("size:" + functionSize + ",funcOffset:" + functionOffset);
                        /** 进行目标函数代码部分进行加密 **/
                        byte[] funcAry = Utils.copyBytes(fileByteArys, functionOffset - 1, functionSize);
                        for (int i = 0; i < funcAry.length - 1; i++) {
                            funcAry[i] = (byte) (funcAry[i] ^ 0xFF);
                        }
                        /** 将加密后的函数体替换回去 **/
                        Utils.replaceByteAry(fileByteArys, functionOffset - 1, funcAry);
                        break;
                    }
                    /** 如果没有找到，则通过chain[index]找到具有相同hash值的下个符号索引 **/
                    funcIndex = Utils.byte2Int(Utils.copyBytes(fileByteArys, hash_offset + 4 * ( 2 + nbucket + funcIndex), 4));
                    System.out.println("funcIndex:"+funcIndex);
                    System.arraycopy(fileByteArys, dynsym_offset + funcIndex*dynsym_size, des, 0, dynsym_size);
                    sym = ELFParser.parseSymbolTable(des);
                    isFindFunc = Utils.isEqualByteAry(dynstr, Utils.byte2Int(sym.st_name), functionName);
                }
                break;
            }
        }
    }
```
### **3. 解密具体过程**
**定义解密函数**
```
/** 定义解密函数为__attribute__((constructor))，在Main函数之前执行 **/
void init_decryption() __attribute__((constructor));
```
**执行解密操作**

```
/** 执行解密操作 **/
void init_decryption() {
    const char *functionName = "Java_idc_hust_edu_cn_functionso_JNIDemo_stringFromJNI";
    unsigned int base_addr = getLibVPAddr();
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "base addr =  0x%x", base_addr);
    /** 用来保存函数的结构信息 **/
    funcInfo info;
    if (getTargetFuncInfo(base_addr, functionName, &info) == -1) {
        print_debug("Find Java_com_example_shelldemo2_MainActivity_getString failed");
        return;
    }
    /** 函数体占的物理页块数目 **/
    unsigned int npage = info.st_size / PAGE_SIZE + ((info.st_size % PAGE_SIZE == 0) ? 0 : 1);
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "npage =  0x%d", npage);
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "npage =  0x%d", PAGE_SIZE);

    /** 修改内存的操作权限，因为不同的虚拟地址空间的内存段是有权限限制的，比如代码段是只读，数据段：可读可写**/
    if (mprotect((void *) ((base_addr + info.st_value) / PAGE_SIZE * PAGE_SIZE), 4096 * npage,
                 PROT_READ | PROT_EXEC | PROT_WRITE) != 0) {
        print_debug("mem privilege change failed");
    }
    /** 执行具体的解密操作 **/
    for (int i = 0; i < info.st_size - 1; i++) {
        char *addr = (char *) (base_addr + info.st_value - 1 + i);
        *addr = ~(*addr);
    }
    /** 解密完成后，要修改回权限 **/
    if (mprotect((void *) ((base_addr + info.st_value) / PAGE_SIZE * PAGE_SIZE), 4096 * npage,
                 PROT_READ | PROT_EXEC) != 0) {
        print_debug("mem privilege change failed");
    }
}
```

```
/**
 * 获得指定函数的符号表结构，整个过程和加密过程定位符号表基本类似
 *
 * @param base_addr  虚拟基址
 * @param funcName 目标函数
 * @param info  函数的结构信息
 * @return
 */
static char getTargetFuncInfo(unsigned long base_addr, const char *funcName, funcInfo *info) {
    Elf32_Ehdr *ehdr;
    Elf32_Phdr *phdr;
    /** elf header **/
    ehdr = (Elf32_Ehdr *) base_addr;
    /** 程序头部表结构 **/
    phdr = (Elf32_Phdr *) (base_addr + ehdr->e_phoff);
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "phdr =  0x%p, size = 0x%x\n", phdr, ehdr->e_phnum);
    int i = 0, isFind = 1;
    /** 在程序头部表结构定位.dynamic段 **/
    for (i = 0; i < ehdr->e_phnum; ++i) {
		__android_log_print(ANDROID_LOG_INFO, "JNITag", "phdr =  0x%p\n", phdr);
        if (phdr->p_type == PT_DYNAMIC) {
            isFind = 0;
            print_debug("Find .dynamic segment");
            break;
        }
        phdr++;
    }
    if(isFind == 0) goto _error;
    /** 找到.dynamic的虚拟地址和大小 **/
    Elf32_Off dynamic_vaddr = base_addr + phdr->p_vaddr;
    Elf32_Off dynamic_size = phdr->p_filesz;
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "dynamic_vaddr =  0x%x, dynamic_size =  0x%x", dynamic_vaddr, dynamic_size);
    /** 查找.dynsym和.dynstr以及.hash节区的信息**/
    Elf32_Dyn *dyn;
    Elf32_Addr dyn_symtab_offset, dyn_strtab_offset, dyn_hash_offset;
    int flag = 0;
    for(i = 0; i < (dynamic_size / sizeof(Elf32_Dyn)); i++){
        dyn = (Elf32_Dyn *)(dynamic_vaddr + i * sizeof(Elf32_Dyn));
        if (dyn->d_tag == DT_SYMTAB) {
            dyn_symtab_offset = (dyn->d_un).d_ptr;
            flag += 1;
            __android_log_print(ANDROID_LOG_INFO, "JNITag", "Find .dynsym section, addr = 0x%x\n", dyn_symtab_offset);
        }
        if (dyn->d_tag == DT_HASH) {
            dyn_hash_offset = (dyn->d_un).d_ptr;
            flag += 2;
            __android_log_print(ANDROID_LOG_INFO, "JNITag", "Find .hash section, addr = 0x%x\n", dyn_hash_offset);
        }
        if (dyn->d_tag == DT_STRTAB) {
            dyn_strtab_offset = (dyn->d_un).d_ptr;
            flag += 4;
            __android_log_print(ANDROID_LOG_INFO, "JNITag", "Find .dynstr section, addr = 0x%x\n", dyn_strtab_offset);
        }
    }
    /** 如果没有找到，则出错，无法进行解密操作 **/
    if ((flag & 0x0f) != 0x0f) {
        print_debug("Find needed .section failed\n");
        goto _error;
    }
    /** 各个偏移加上虚拟基地址定位到实际映射的虚拟地址 **/
    dyn_symtab_offset += base_addr;
    dyn_hash_offset += base_addr;
    dyn_strtab_offset += base_addr;

    unsigned funHash, nbucket;
    unsigned *bucket, *chain;
    funHash = getElfHash(funcName);
    /** .dynsym结构信息 **/
    Elf32_Sym *funSym = (Elf32_Sym *)dyn_symtab_offset;
    /** .dynstr结构信息 **/
    char *dynstr = (char *)dyn_strtab_offset;
    /** .nbucket **/
    nbucket = *((int *) dyn_hash_offset);
    /** 指向bucket，nbucket和nchian的两个值的大小为8，跳过这两个字段 **/
    bucket = (unsigned int *) (dyn_hash_offset + 8);
    /** 指向chain**/
    chain = (unsigned int *) (dyn_hash_offset + 4 * (2 + nbucket));
    flag = -1;
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "hash = 0x%x, nbucket = 0x%x\n", funHash, nbucket);
    /** 得到funcName的哈希值为funHash的符号索引 **/
    int sym_index = (funHash % nbucket);
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "mod = %d\n", sym_index);
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "i = 0x%d\n", bucket[sym_index]);
    /** 循环查找funcName在符号表中的索引 **/
    for (i = bucket[sym_index]; i != 0; i = chain[i]) {
        __android_log_print(ANDROID_LOG_INFO, "JNITag", "Find index = %d\n", i);
        if (strcmp(dynstr + (funSym + i)->st_name, funcName) == 0) {
            flag = 0;
            __android_log_print(ANDROID_LOG_INFO, "JNITag", "Find %s\n", funcName);
            break;
        }
    }
    if (flag) goto _error;
    info->st_value = (funSym + i)->st_value;
    info->st_size = (funSym + i)->st_size;
    __android_log_print(ANDROID_LOG_INFO, "JNITag", "st_value = %d, st_size = %d", info->st_value, info->st_size);
    return 0;
    _error:
    return -1;
}
```