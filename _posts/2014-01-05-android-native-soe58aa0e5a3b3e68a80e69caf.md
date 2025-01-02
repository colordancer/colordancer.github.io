---
id: 1823
title: 'Android Native So加壳技术'
date: '2014-01-05T11:45:05+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1823'
permalink: /2014/01/05/android-native-so%e5%8a%a0%e5%a3%b3%e6%8a%80%e6%9c%af/
views:
    - '7087'
categories:
    - 安全研究
tags:
    - android
---

 <span style="line-height: 1.6em;">目前市面上针对Apk的保护主要是基于Dex，公开的有DexGuard、梆梆、爱加密、ApkProtect等，私底下相信很多涉及到技术保密的App开发商都在做自己的保护策略。</span>

 而针对so的保护就相对滞后了一些，这里有so在app中扮演的角色的原因，也有so自身特点的原因。

 我个人理解，elf文件相对Windows的PE来说松散一些，物理磁盘上的文件和内存里的文件镜像差异更大，所以在处理上要解决的问题较多。再加上Arm汇编指令的特点，在处理跳转时考虑的问题较多，所以导致针对Android So的保护成本较高。

 Android so加壳主要需要解决两个问题：  
 1. 对elf文件加壳  
 2. 对Android So的加载、调用机制做特殊处理

 elf文件加壳，市面上还没有公开的比较好的解决方案，连UPX的作者都无心支持，还有一种说法是说，Linux本来就是开源的，所以为什么要加壳？:-)

 其实不管是Linux还是Windows，加壳的思路基本是一致的，简单点无非加密+拆解+混淆，复杂点如Stolen Code+VM等。所以确定好一个加壳方案之后，剩下的就是了解elf的文件结构和加载机制，然后自己写一套壳+loader。

 Android上的loader是/system/bin/linker，跟linux上的ld有一些区别。但主要过程还是一致的：map + relocate + init。

 Android so主要充当的角色是通过JNI与java交互，所以主要是作为一个库存在（也有一些so是可执行的），然后被Android runtime加载，并能被java层调用。所以对elf加完壳之后，还要对Android so做一些特殊处理：

 1. Android so被System.LoadLibrary()加载之后，会将so的信息存储在一个全局链表里，所以要保证脱壳后的so能被这个链表访问到。

 2. Android so库函数被java调用有两种方式：一种是通过registerNative注册，另一种是通过javah命名规则命名。所以一个通用的加壳方案要保证所有的库函数都能被调用，前者好解决，后者需要花点功夫。

 解决掉这两个问题之后，基本上一套Android so加壳框架就成形了，后续就可以增加各种Anti tricks来完善壳的强度。

 （完）