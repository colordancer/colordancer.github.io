---
id: 210
title: 'BlackHat之路第五期：一个简单的PE Append感染型病毒的手工修复。'
date: '2009-05-18T10:57:35+08:00'
layout: post
guid: 'http://www.colordancer.net/wordpress/?p=210'
permalink: /2009/05/18/blackhat%e4%b9%8b%e8%b7%af%e7%ac%ac%e4%ba%94%e6%9c%9f%ef%bc%9a%e4%b8%80%e4%b8%aa%e7%ae%80%e5%8d%95%e7%9a%84pe-append%e6%84%9f%e6%9f%93%e5%9e%8b%e7%97%85%e6%af%92%e7%9a%84%e6%89%8b%e5%b7%a5%e4%bf%ae/
views:
    - '887'
categories:
    - 安全研究
---

PE\_DOWNEXEC.O是上周刚发现的新的感染型样本，该病毒会感染exe文件，并新加一个节，节名为hhqg，然后病毒会修改exe文件的eip，指向这个新节，执行完毕后重新跳回原来的eip。

![](http://www.colordancer.net/blog/attachments/month_0905/k200951810540.jpg)

下面我们将手工修复被病毒感染的文件。  
  
OD载入，我们在text段加上内存访问断点：  
![](http://www.colordancer.net/blog/attachments/month_0905/s200951810548.jpg)

然后按F9，执行到以下位置：  
![](http://www.colordancer.net/blog/attachments/month_0905/t2009518105412.jpg)

因此断定004012A0这个位置为eip。  
下面我们来看看这个新节做了什么事情。  
OD重新载入，单步到以下位置，发现一个函数：  
![](http://www.colordancer.net/blog/attachments/month_0905/f2009518105416.jpg)

F7进入，继续单步，发现函数：  
![](http://www.colordancer.net/blog/attachments/month_0905/v2009518105421.jpg)

F7进入，单步：  
![](http://www.colordancer.net/blog/attachments/month_0905/a2009518105425.jpg)

F7进入，单步：  
![](http://www.colordancer.net/blog/attachments/month_0905/62009518105429.jpg)

F7进入，单步：  
![](http://www.colordancer.net/blog/attachments/month_0905/82009518105433.jpg)  
![](http://www.colordancer.net/blog/attachments/month_0905/f2009518105438.jpg)

好，到这里我们发现了关键代码，新节里调用了Loadlibrary和GetProcAddress，先后获得了WinExeC和URLDownloadToCacheFileA函数，我们查看栈里面有什么内容：  
![](http://www.colordancer.net/blog/attachments/month_0905/q2009518105442.jpg)  
![](http://www.colordancer.net/blog/attachments/month_0905/c2009518105446.jpg)

好，这里就真相大白了，新节的目的就是从网上下载一个exe文件，然后调用winexe执行。

下面我们来手动修复。  
使用LoadPE载入这个被感染的文件，选中这个新节，删除掉：  
![](http://www.colordancer.net/blog/attachments/month_0905/g2009518105450.jpg)

然后修复原来的EIP：  
![](http://www.colordancer.net/blog/attachments/month_0905/52009518105454.jpg)

点击保存、确定。

到这里还没有完成，这样的exe是不能运行的，我们必须对EXE重建：  
![](http://www.colordancer.net/blog/attachments/month_0905/i2009518105459.jpg)

好了，这样简单的手工修复就完成了，点击运行，ok没问题，而且因为没了新节的代码，执行速度也快了很多：  
![](http://www.colordancer.net/blog/attachments/month_0905/9200951810553.jpg)