---
id: 182
title: BlackHat之路第三期之后续：第一个注册机的注册算法分析
date: '2009-03-18T11:33:51+08:00'
layout: post
guid: 'http://www.colordancer.net/wordpress/?p=182'
permalink: /2009/03/18/blackhat%e4%b9%8b%e8%b7%af%e7%ac%ac%e4%b8%89%e6%9c%9f%e4%b9%8b%e5%90%8e%e7%bb%ad%ef%bc%9a%e7%ac%ac%e4%b8%80%e4%b8%aa%e6%b3%a8%e5%86%8c%e6%9c%ba%e7%9a%84%e6%b3%a8%e5%86%8c%e7%ae%97%e6%b3%95%e5%88%86/
views:
    - '1043'
categories:
    - 安全研究
---

PEID查看，壳是PESpin 1.3，该壳挺难脱，需要找到Stolen Code，然后修复OEP和IAT。由于本文只谈算法分析，所以脱壳步骤不在其中。

壳脱完后，运行程序，输入几个测试数据，发现有两种结果：1·提示格式不对；2·提示失败。

最简单的方法，在MessageBoxA设断，输入测试数据：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_31.jpg)](http://photo.blog.sina.com.cn/showpic.html)

点击注册，断在以下位置：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_32.jpg)](http://photo.blog.sina.com.cn/showpic.html)

按Ctrl + F9，运行至以下代码处：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_33.jpg)](http://photo.blog.sina.com.cn/showpic.html)

按Ctrl + F9，运行至以下代码处：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_34.jpg)](http://photo.blog.sina.com.cn/showpic.html)

简单看一下之后，发现这个函数调用的API就只有MessageBox和EnableWindow等，因此判断算法逻辑不在此函数里。因此需要判断该函数的调用者。这有几个方法：1·在该函数的开始处设断，然后看调用堆栈；2·用IDA的”x”功能

这里我们用IDA分析，因为后面主要也要IDA分析算法流程。在IDA里来到00426A7A处：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_35.jpg)](http://photo.blog.sina.com.cn/showpic.html)

按X后，选择调用者，进去看一下：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_36.jpg)](http://photo.blog.sina.com.cn/showpic.html)

发现是个简单的判断函数，因此笔者怀疑是MFC静态编译的原因，这里不做深究。由于该函数的功能是根据输入的字符串来弹出对话框，因此对算法逻辑的判断没有作用，因此只好用OD的调用堆栈来找到注册按钮调用的函数。

一步一步设断、堆栈平衡后（事实上只要两步），我们终于来到函数入口处：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_37.jpg)](http://photo.blog.sina.com.cn/showpic.html)

在IDA中跳到004016F0，大致看一下，流程蛮复杂：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_38.jpg)](http://photo.blog.sina.com.cn/showpic.html)

根据IDA的智能分析和标识出的API，我们简单看一下，程序的流程大致如下：

1·先判断输入的用户名和注册码的格式

2·注册码生成

3·注册码判断

4·注册成功、注册失败

由于我们要分析算法，而且IDA将API都标识了出来，所以我们从IDA导出Map文件，用OD来分析。根据IDA的分析流程，我们调试几次后来到算法的入口处，分析如下：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_39.jpg)](http://photo.blog.sina.com.cn/showpic.html)

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_310.jpg)](http://photo.blog.sina.com.cn/showpic.html)

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_311.jpg)](http://photo.blog.sina.com.cn/showpic.html)

综合以上的分析，所以我们得出以下的算法结论：

1·去用户名的前二位做运算，得到注册码的前四位相关

2·对用户名的所有字母做ascii码之和，得到注册码的第五位相关

3·由于注册码须大于等于8位，所以剩下的3位可以随便填

4·有一个通用注册码

因此，根据用户名生成注册码的C++代码如下：

\[cpp\]

TCHAR\* arr = m\_username.GetBuffer();  
int i = 10;

int j = 0;  
int k = 0;  
int l = 0;  
int m = 0;

char a = arr\[0\];  
a |= 68;  
j = a % i;

a = arr\[1\];  
a |= 66;  
k = a %i;

a = arr\[0\];  
a |= 67;  
l = a %i;

a = arr\[1\];  
a |= 68;  
m = a % i;

int all = 0;  
for (int it = 0; it &lt; m\_username.GetLength() ; it ++)  
{  
all += arr\[it\];  
}

m\_keyword = (char)(j+48);  
m\_keyword += (char)(k+48);  
m\_keyword += (char)(l+48);  
m\_keyword += (char)(m+48);

all %= i;  
all += 48;  
m\_keyword += (char)all;

//随机生成大于等于三个的字母  
for (int it = 0; it &lt;= 3 + rand() % 10; it++)  
{  
m\_keyword += (char)(rand() % 57 + 65);  
}

\[/cpp\]

经测试，该算法有效。

经过这次简单的算法逆向分析，笔者发现，怎样结合IDA和OD用来分析，决定了逆向分析的效率和轻松度；笔者认为：从OD动态调试从而找到函数入口，然后在IDA中标注、简单分析该函数的流程，导成Map文件，进而继续用OD动态调试 是一个很不错的步骤；同时，由于汇编语言本身的缘故（寄存器数目有限），在反汇编中基本看不到”变量”这个东西，而根据笔者的目前知识，函数的局部变量通常都是在函数开始处push几个预留栈空间，然后对该变量操作时，就使用寄存器相对寻址等寻址方法获得,所以在调试的时候，最好将这用到的变量记下来。如下：

[![](http://www.colordancer.net/blog/wp-content/uploads/2011/06/061911_0811_312.jpg)](http://photo.blog.sina.com.cn/showpic.html)