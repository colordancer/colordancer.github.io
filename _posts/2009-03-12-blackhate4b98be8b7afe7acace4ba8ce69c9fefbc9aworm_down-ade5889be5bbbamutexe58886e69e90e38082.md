---
id: 178
title: BlackHat之路第二期：Worm_down.ad创建mutex分析。
date: '2009-03-12T12:50:53+08:00'
layout: post
guid: 'http://www.colordancer.net/wordpress/?p=178'
permalink: /2009/03/12/blackhat%e4%b9%8b%e8%b7%af%e7%ac%ac%e4%ba%8c%e6%9c%9f%ef%bc%9aworm_down-ad%e5%88%9b%e5%bb%bamutex%e5%88%86%e6%9e%90%e3%80%82/
views:
    - '983'
categories:
    - 安全研究
---

Worm\_down.a和Worm\_down.ad都是利用 MS08-067漏洞，通过溢出挂载svchost.exe进程从而进行局域网传播的病毒。其中.ad是.a的变种。  
这两个变种都有一些一样的功能，其中一个便是当本身被加载后，会创建一个Mutex，以防止被重复加载。

.a变种创建该Mutex的方法是调用zip.lib中的crc32算法，将计算机名进行crc验算后作为Mutex的名字。由于该方法简单，所以很容易创建相同的Mutex。  
.ad变种意识到了这一点，因此作者自己写了一个算法，用来生成Mutex名。这里我们便来分析一下.ad变种生成该Mutex的方法。

由于要根据计算机名做Mutex，因此我们在GetComputerNameA上设断点，然后F9运行，如下图：  
![](/images/attachments/month_0903/22009312124649.gif)  
  
按Ctrl+F9运行至GetComputerNameA下一行指令：  
![](/images/attachments/month_0903/v2009312124713.gif)

于是我们发现了与.a变种差不多代码。先获得计算机名，然后生成\_snprintf需要的参数，最后通过\_snprintf生成Mutex名。  
我们从\_snprintf往上看，发现有两个函数：009fb647和009fb510，跟进去看看：  
![](/images/attachments/month_0903/52009312124718.gif)

大致一看，我们看到rand函数，inc、cmp、jl在一起，我们于是想到for循环

一步一步分析如下：

![](/images/attachments/month_0903/32009312124811.gif)

于是有了这些分析，我们大概能总结出该函数是如下C++代码：  
\[cpp\]  
if( n != 0)  
{  
 for(int i =0; i

第一个参数0a09e14应该是一个buffer数组，edx应该就是循环的次数了，可是edx哪里来的呢，继续往上看：  
![](/images/attachments/month_0903/a2009312124829.gif)

我们一眼就能看到srand和rand，那看来edx也和随机数有关系。我们一句一句来看：  
![](/images/attachments/month_0903/12009312124833.gif)  
先调用009f8245，然后将返回结果和2f53508b进行异或运算，得到srand的随机数种子seed，接下来调用rand函数生成一个随机数，最后将该随机数除以3，求余数，再加上6，就是我们刚才那个函数的edx了。哈哈，看来这个Mutex的长度是6到8位。

有了这样的分析，我们差不多就知道个大概了。我们从头理一下，我们可以总结出以下c++代码（为了方便显示，我们不采用buffer，而是直接输出）

\[cpp\]  
\#include <stdafx.h>  
\#include <iostream>  
using namespace std;</iostream></stdafx.h>

void output(int n)  
{  
 if( n != 0)  
 {  
 for(int i =0; i

与病毒生成的Mutex一样：  
![](/images/attachments/month_0903/i2009312124842.gif)

所以，现在我们就可以自己生成这个Mutex，用来防止病毒被加载了。

需要说明的是：这个不变的Mutex其实也是利用了rand()函数。当不为rand设置随机数种子时，只要不重启机器，通过rand每次生成的随机数都是一样的。所以，只要不重启机器，病毒每次创建的Mutex都是一样的。