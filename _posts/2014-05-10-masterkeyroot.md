---
id: 1837
title: PC端利用masterkey漏洞和jdi接口进行root
date: '2014-05-10T12:16:04+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1837'
permalink: /2014/05/10/masterkeyroot/
views:
    - '4065'
categories:
    - 安全研究
tags:
    - android
    - masterkey
    - root
---

 <span style="font-size: 13px;">最近要实现PC端root的方案，逆了某客户端后发现可以通过masterkey漏洞来完成4.3版本系统（+之前）的</span><span style="font-size: 13px;">大部分root。具体调研之后发现其实</span>是2013年的技术，最先应该是Saurik提出的，他的Cydia Impactor中已经实现了该方案。

 该方案主要的原理是：在我前面分析masterkey漏洞的文章里提到，利用masterkey漏洞的主要方法是替换文件，然后在替换后的文件里做一些事情。这样的方法有一些局限性。Saurik提出可以通过替换AndroidManifest.xml提权为sytem用户，然后再通过一些方法提权为root用户，这样就可以只通过替换AndroidManifest.xml完成root。

 提权为system用户的方法，需要结合Adb + JDI接口。JDI是Java Debug Interface，主要是通过JDWP接口利用socket通信。

 主要的步骤可以在Saurik的文章中找到：http://www.saurik.com/id/17

 JDWP报文数据格式可以在这里找到：http://docs.oracle.com/javase/6/docs/platform/jpda/jdwp/jdwp-protocol.html

 通过JDWP发送和接收的报文数据都是little-endian的，所以在处理时需要注意。

 Demo图：

 [![root](/images/wp-content/uploads/2014/05/root-600x547.png)](/images/wp-content/uploads/2014/05/root.png)