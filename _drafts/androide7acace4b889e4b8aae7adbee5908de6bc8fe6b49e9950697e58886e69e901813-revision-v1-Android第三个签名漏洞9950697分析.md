---
id: 2196
title: 'Android第三个签名漏洞#9950697分析'
date: '2025-01-02T12:16:46+08:00'
author: colordancer
layout: revision
guid: 'http://colordancer.net/blog/?p=2196'
permalink: '/?p=2196'
---

 <span style="line-height: 1.6em;">上周末Google发布了Android 4.4，随着一系列新功能包括安全措施的发布，我们也从AOSP的源码中看到了google悄悄修复了一个bug：</span>

 [https://android.googlesource.com/platform/libcore/+/2da1bf57a6631f1cbd47cdd7692ba8743c993ad9%5E%21/#F0](https://android.googlesource.com/platform/libcore/+/2da1bf57a6631f1cbd47cdd7692ba8743c993ad9%5E%21/)

 ![](http://www.colordancer.net/blog/wp-content/uploads/2013/11/110713_1045_Android1.png)

 <span style="line-height: 1.6em;">从上图中可以看到，这个bug编号为9950697，bug存在于"filename 字段的长度在central directory和在local file entry里解析不一致"，而且这个bug早在7月23号就已经被修复：</span>

 ![](http://www.colordancer.net/blog/wp-content/uploads/2013/11/110713_1045_Android2.png)

 <span style="line-height: 1.6em;">下面我们分析一下该漏洞的具体成因。我们在《[Android第二个签名漏洞](http://www.colordancer.net/blog/2013/08/26/android-%e7%ac%ac%e4%ba%8c%e4%b8%aa%e7%ad%be%e5%90%8d%e6%bc%8f%e6%b4%9e-9695860-%e6%8f%ad%e7%a7%98/)》一文中详细解释了APK包的文件结构。为了方便理解，我们至下往上把一个Zip文件分为：目录段、索引段、数据段。在索引段和数据段中，每一个ZipEntry的header里都存储了关于这个ZipEntry的信息，包括filename，CRC等。索引段和数据段中的这两份相同的信息，用来做Zip文件的完整性检验。</span>

 <span style="line-height: 1.6em;">Android在校验签名时，解析Apk包用的是java代码（ZipFile.java和ZipEntry.java），而在安装apk时，包括解压、dexopt等，用的是c代码（ZipArchive.cpp），这两份代码在解析ZipEntry时的步骤都是先从索引段读取ZipEntry的信息，然后定位到数据段。</span>

 <span style="line-height: 1.6em;">数据段中的ZipEntry的Data\[\]字段是用来存放真正的压缩数据的。Java和c定位到Data\[\]字段的方法都是根据Data\[\]字段之前的字段的长度计算偏移：</span>

 <span style="line-height: 1.6em;">Data\[\]的偏移 = ZipEntry的偏移 + 固定header的长度 + extraFieldLength + fileNameLength</span>

 <span style="line-height: 1.6em;">我们在之前的文章中讨论过针对索引段的extraField和comment的攻击，这次轮到了fileName。Java使用的fileNameLength是从索引段的ZipEntry获得的，而C则是从数据段的ZipEntry获得的，所以就这里造成了不一致性，导致java和c定位到的data\[\]不一致。因此可以在数据段的ZipEntry中构造一个不一样的fileNameLength，进而让java校验签名时读取的是合法的文件，而c在安装是读取的是恶意的文件，绕过签名验证。</span>

 ![](http://www.colordancer.net/blog/wp-content/uploads/2013/11/110713_1045_Android3.png)

 <span style="line-height: 1.6em;">攻击模型：</span>

 ![](http://www.colordancer.net/blog/wp-content/uploads/2013/11/110713_1045_Android4.png)

 <span style="line-height: 1.6em;">不过需要注意的是，由于fileNameLength这个字段的类型是一个无符号short，最大值为64K，所以为了能让C代码成功能跳转到fake classes.dex，必须要求 real classes.dex加上"classes.dex"字符串的长度小于64k。也就是说，对这个漏洞的利用要求被绕过的原始正常文件的大小必须小于64k，这对这个漏洞的利用带来了很大的限制，所以实际的危害并不很大。</span>