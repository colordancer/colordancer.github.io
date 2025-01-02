---
id: 231
title: 利用SetUnhandledExceptionFilter对Release程序进行异常处理。
date: '2009-07-02T16:03:02+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/wordpress/?p=231'
permalink: /2009/07/02/%e5%88%a9%e7%94%a8setunhandledexceptionfilter%e5%af%b9release%e7%a8%8b%e5%ba%8f%e8%bf%9b%e8%a1%8c%e5%bc%82%e5%b8%b8%e5%a4%84%e7%90%86%e3%80%82/
views:
    - '7546'
classic-editor-remember:
    - classic-editor
categories:
    - 开发设计
---

闪电杀毒手2.5发布出去有一段时间了，最近收到几个Bug反馈。  
通常发布出去的Release产品，如果遇到没有预料到的异常，通常都会弹出Windows默认的错误窗口Windows Error Reporter.  
这时用户就傻了：交互不友好；RD也傻了：不知道异常的具体信息。

SetUnhandledExceptionFilter可以设置在WER弹出之前的最后一次异常处理的机会。所以只要设置好我们的异常处理函数就可以捕获到Unhandled Exception。SetUnhandledExceptionFilter(CleanToolExceptionFun);  
LONG WINAPI CleanToolExceptionFun(struct \_EXCEPTION\_POINTERS\* ExceptionInfo)  
{

}

在这个函数里你可以做以下几件事：  
1·获得异常的ExceptionRecord和Context  
2·获得异常编号  
3·获得异常的模块  
4·获得堆栈的信息  
5·程序继续执行还是退出

1和2都好办，直接通过传递进来的参数就可以获得：  
ExceptionInfo-&gt;ExceptionRecord-&gt;ExceptionAddress;

3通常的做法是，通过ExceptionRecord获得异常的内存地址，调用VirtualQuery获得Handle，再通过GetModuleFileName获得出现异常的文件路径。  
MEMORY\_BASIC\_INFORMATION mbi = {0};  
if (FALSE == ::VirtualQuery( addr, &amp;mbi, sizeof(mbi) ) ) return;  
UINT\_PTR h\_module = (UINT\_PTR)mbi.AllocationBase;  
::GetModuleFileNameW((HMODULE)h\_module, sz\_module, len);  
return;

4的做法是通过StackWalk64函数便利堆栈。  
StackWalk64(IMAGE\_FILE\_MACHINE\_I386,hCurrentProcess,hCurrentThread,&amp;sStackFrame,pContext,0,0,0,0)

5是通过函数的返回值来确定的。

在实现的过程遇到几个好玩的问题：  
1·如果exe调用dll，dll中要使用AFX\_MANAGE\_STATE(AfxGetStaticModuleState())之后，dll的资源才能被访问到；但是如果dll调用exe，则必须使用AFX\_MANAGE\_STATE(AfxGetAppModuleState())。

2·StackWalk64的问题：在Release版本下，Stackwalk64获得的堆栈信息不完整，具体表现为：如果异常是由MFC的模块抛出的，那么获得的堆栈将缺少栈最上方我们自己的模块的信息。比如A-&gt;B-&gt;MFC，则获得的栈为（至顶向下）：MFC-&gt;A。但是在Debug的版本下不存在这个问题。为此我还特意用OD调试了Release版本，发现堆栈是完整的，那么只能认为是StackWalk的问题了。

3·在便利异常的模块的节表的时候，Release版本下程序偶尔会莫名其妙地退出。

4·Release编译的优化问题：如果我直接写int i = 5/0，编译时会直接报错。如果我写成：  
int i = 0;  
AfxMessageBox(L”%d”, 5 /i)，编译就可以通过。汗……

![](/images/attachments/month_0907/d200972155845.jpg)