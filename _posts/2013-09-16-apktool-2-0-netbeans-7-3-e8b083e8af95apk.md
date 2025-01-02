---
id: 1782
title: 'apktool 2.0+netbeans 7.3调试apk'
date: '2013-09-16T11:31:08+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1782'
permalink: /2013/09/16/apktool-2-0-netbeans-7-3-%e8%b0%83%e8%af%95apk/
views:
    - '5051'
categories:
    - 安全研究
tags:
    - apktool
    - netbeans
---

 Apktool 2.0优化了调试的支持，相对1.5.3的版本，主要做了以下改进：  
 1. 可以配合Netbeans 7.3调试apk  
 2. 支持显示寄存器的值，这是非常好的改进（1.5.3只可以显示函数参数P的值）  
   
 目前2.0还没有release，这是我自己编译的版本，下载地址：http://pan.baidu.com/share/link?shareid=1438815794&amp;uk=369410799  
 netbeans 7.3下载地址：https://netbeans.org/downloads/  
   
<span style="line-height: 1.6em;">具体调试方法，以test.apk为例：</span>

 步骤1：生成支持调试的apk  
 1. 反编译：  
 java -jar apktool-2.0.0.jar d -d test.apk -o test.debug  
 2. 找到入口activity的oncreate()函数，在  
 invoke-super {p0, p1}, Landroid/app/Activity;-&gt;onCreate(Landroid/os/Bundle;)V  
 后添加：  
 invoke-static {}, Landroid/os/Debug;-&gt;waitForDebugger()V  
 3. 回编：  
 java -jar apktool-2.0.0.jar b -d test.debug -o test.debug.apk  
 4. 签名  
 5. 安装  
   
 步骤2：netbeans设置  
 1. 删除test.debug目录下的build文件夹  
 2. 打开netbeans，选择“文件”-“新建项目”-“基于现有源代码的java项目”  
 3. 在“项目文件夹处”选择test.debug目录  
 4. 在“源包文件夹”出选择test.debug.smali目录  
 5. 点击完成，项目创建完毕  
   
 步骤3：调试  
 1. 在模拟器中运行重新打包的test.apk，test.apk会处于挂起状态  
 2. 在netbeans中找到入口activity的oncreate函数，在刚才invoke-static {}, Landroid/os/Debug;-&gt;waitForDebugger()V  
 下一行下断点  
 3. 在netbeans中，选择“调试”-“连接调试器”  
 4. 依次设置：  
 调试器：JPDA  
 连接器：SocketAttach  
 传输：dt\_socket  
 主机：127.0.0.1  
 端口：8700  
 超时：\[可不填\]  
 5. 确定，即完成连接调试，可以发现IP停在了刚才下断点的地方。  
 6. 可以看到，apktool2.0+netbeans7.3支持显示寄存器的值：

 [![netbeans 7.3 debug](http://www.colordancer.net/blog/wp-content/uploads/2013/09/netbeans-7.3-debug-600x467.png)](http://www.colordancer.net/blog/wp-content/uploads/2013/09/netbeans-7.3-debug.png)