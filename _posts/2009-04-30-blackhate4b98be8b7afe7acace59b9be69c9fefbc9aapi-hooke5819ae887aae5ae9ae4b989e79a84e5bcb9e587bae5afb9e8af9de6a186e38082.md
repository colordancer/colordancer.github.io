---
id: 200
title: 'BlackHat之路第四期：API Hook做自定义的弹出对话框。'
date: '2009-04-30T16:04:49+08:00'
layout: post
guid: 'http://www.colordancer.net/wordpress/?p=200'
permalink: /2009/04/30/blackhat%e4%b9%8b%e8%b7%af%e7%ac%ac%e5%9b%9b%e6%9c%9f%ef%bc%9aapi-hook%e5%81%9a%e8%87%aa%e5%ae%9a%e4%b9%89%e7%9a%84%e5%bc%b9%e5%87%ba%e5%af%b9%e8%af%9d%e6%a1%86%e3%80%82/
views:
    - '1203'
categories:
    - 安全研究
---

Windows自定义的弹出对话框太丑了，而由于对话框是通过调用MessageBox这个API，由Windows生成一个Model窗口产生的，所以我们没办法为这个Windows生成的窗口自定义皮肤。

所以可以通过Hook MessageBox这个API来自定义对话框。  
这里的Hook有两种，一个是Inline Hook，一个是IAT Hook。  
前者是修改当前进程空间里的API所在内存，后者是修改当前进程的导入表中的函数指向地址。  
  
我这里使用的是前者，因为由于杀毒手调用了很多dll，如果为每个dll修改导入表，显得麻烦了点，所以直接修改进程空间内的API比较方便。  
因此，把当前进程空间内的MessageBox函数的前几个操作变为“跳转到指向我们的自定义函数的地址”，就可以产生我们想要的对话框了。  
当我们的程序调用MessageBox时，就会自动跳转到我们自己的对话框函数。

这个跳转的指令如下：  
{0xB8, 0x0F, 0x10, 0x40, 0x00, 0xFF, 0xE0}  
后面再跟上我们自己的函数地址。  
首先用GetProcAddress获得MessageBox的地址，然后VirtualProtect修改内存页属性，再调用WriteProcessMemory修改内存。完工。

但是，需要注意的是，这个我们自定义的函数，需要考虑得和MessageBox API一样全面，函数参数数目、参数类型等等。

另外，还有一个方法能实现自定义对话框，就是消息Hook。这里就不详述了。

效果：  
hook前  
![](http://www.colordancer.net/blog/attachments/month_0904/y200943016333.jpg)

hook后  
![](http://www.colordancer.net/blog/attachments/month_0904/m200943016342.jpg)