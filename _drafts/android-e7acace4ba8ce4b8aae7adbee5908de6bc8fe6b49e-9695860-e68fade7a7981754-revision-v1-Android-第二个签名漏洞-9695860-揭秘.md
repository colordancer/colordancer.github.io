---
id: 2200
title: 'Android 第二个签名漏洞 #9695860 揭秘'
date: '2025-01-02T12:16:46+08:00'
author: colordancer
layout: revision
guid: 'http://colordancer.net/blog/?p=2200'
permalink: '/?p=2200'
---

<span style="line-height: 1.6em;">在android master key （#8219321）签名漏洞爆出来不久，国内的安全团队”安卓安全小分队”在其博客（</span><http://blog.sina.com.cn/u/3194858670><span style="line-height: 1.6em;">）发布文章称发现另外一个android签名漏洞，但是小分队提出的利用该漏洞的方法存在一定的局限性，所以理论上可行，但是实际危害并不大。不久之后，Cydia创始人saurik在其网站（</span><http://www.saurik.com/><span style="line-height: 1.6em;">）上发布文章，称其发现了利用这个漏洞更好的方法。可能是因为构造特殊zip包的难度较大，也或者Master key的影响还未散去，这个新发现的漏洞虽然比Master key具有更大的破坏性，却一直没得到重视。</span>

该漏洞（编号9695860）的主要原理是，android在解析Zip包时，C代码和Java代码存在不一致性：Java将short整数作为有符号数读取，而C将其作为无符号数。

了解该漏洞的具体细节之前，需先要了解下Zip包的文件结构。

如果一个压缩包里有多个文件，可以认为每个文件都是被单独压缩成一个包，然后再拼成一个大Zip包。下图中的每个FileEntry即代表一个压缩后的文件数据。

![](/images/wp-content/uploads/2013/08/082613_1549_Android1.png)

为了便于理解，在这里我们通俗地把Zip包文件分为三部分：数据段（FileEntry），索引段（Index）和文件头（Header），Index和Header组合成CentralDirectory（中心目录段）。

- <div style="text-align: justify;"><span style="font-size: 10pt;">文件头在Zip包文件末尾，里面记录了索引段的偏移、大小和索引个数等。</span></div>
- <div style="text-align: justify;"><span style="font-size: 10pt;">索引段可以认为是一个索引的数组，每个索引记录了其在数据段中对应数据的信息，包括CRC和偏移等。</span></div>
- <div style="text-align: justify;"><span style="font-size: 10pt;">数据段中的数据分为FileHeader和压缩的文件数据。</span></div>
    - <div style="text-align: justify;"><span style="font-size: 10pt;">FileHeader同样记录了数据的一些基本信息，可以用来跟索引段中记录的数据比较，从而检查压缩包的完整性。</span></div>
    - <div style="text-align: justify;"><span style="font-size: 10pt;">Data区域即是文件被压缩的数据存储区。</span></div>

通常来说，一般要读取一个Zip压缩包，应分为三步：

1. <div style="text-align: justify;"><span style="font-size: 10pt;">先定位到Header，读取索引段的偏移、大小和索引的个数；</span></div>
2. <div style="text-align: justify;"><span style="font-size: 10pt;">定位到索引段，根据每个索引中个字段定义的大小计算偏移，挨个读取索引</span></div>
3. <div style="text-align: justify;"><span style="font-size: 10pt;">根据索引中记录的数据段的偏移定位到真正的压缩数据。</span></div>

android校验apk签名是在java层处理，java解析zip的文件格式，读取包里的每个文件，计算哈希进而校验签名。Java在计算zip文件结构偏移时，将整数作为有符号数读取却没有做处理，所以当这些偏移的大小符合一定条件（大于32767）时，java就会出现读取错误的问题。

![](/images/wp-content/uploads/2013/08/082613_1549_Android2.png)

上图的extraLength，如果大于32767（0x7FFF），java就会将其当成负数。所以java计算下一段数据的偏移时就会出错。小分队提到的方法是，设置数据段中classes.dex所在FileEntry中FileHeader的extraLenth值为0xFFFD，即-3，这样java计算FileEntry中data的起始地址就会往后移3为，正好指向文件名中的”dex”，而”dex”恰好是dex文件头的魔术字，所以可以将这段空间作为正常dex的数据段，而将正确偏移（0xFFFD，即65533）指向的空间写成恶意数据，这样java在校验签名时读取的是正常的dex，而C在加载文件时读取的是恶意的dex文件。

![](/images/wp-content/uploads/2013/08/082613_1549_Android3.gif)

小分队的方法是针对数据段做修改，存在一定缺陷，一是因为无符号short型整数的最大值是64k，所以恶意classes.dex的地址最大只能在FileHeader+64K以外，这其间的空间供正常的classes.dex使用，所以这就要求正常dex的大小不能超过64K，二是因为要求文件名后三位和对应的文件头都是是”dex”，所以这个方法只能伪造dex。

Saruik发现java不仅在处理数据段结构体时存在错误，而且在处理Central Directory时也存在类似错误。所以可以对索引段做手脚，伪造索引。

Java在读取Central Directory时，也是将整数做有符号数处理，不过不同的是如果发现值小于0，就将其忽略，即当0。

![](/images/wp-content/uploads/2013/08/082613_1549_Android4.png)

所以，只要大于32767的值都会被java当作0。这样我们就可以修改某个index（称作A）的extraLength值大于0x7FFF，然后紧接着在该index写入后为正常的index（称作B），而真正的偏移处写入为恶意的index（称作B’）。然后修改B的extraLength，使得其的大小能够让指针指向下一个index（称作C）是正常的。这样一来，java解析时读到的是ABC，而C读到的是AB’C。前文提到，FileEntry的位置是索引段指向的，所以只要索引被”劫持”，我们就可以zip包的任意位置写入我们的恶意数据，然后让B’指向它即可。

![](/images/wp-content/uploads/2013/08/082613_1549_Android5.png)

下图是我构造的假冒系统浏览器的截图。笔者修改了系统默认浏览器的图标，android系统版本4.1.1，可以成功绕过签名并安装执行。需要说明的是，此漏洞在2.3之后的版本才有效。

![](/images/wp-content/uploads/2013/08/082613_1549_Android6.png)

相比Master key，这个漏洞的通用性和危害性更强：

1. <div style="text-align: justify;"><span style="font-size: 10pt;">不需要apk包里有同名文件，几乎可以往apk里添加任何文件</span></div>
2. <div style="text-align: justify;"><span style="font-size: 10pt;">因为修改的是索引段，通常的压缩软件用的是C读取压缩包的方法，只要恶意索引的名称不太明显，就很难看出该压缩包是非法篡改的，所以隐蔽性很高</span></div>
3. <div style="text-align: justify;"><span style="font-size: 10pt;">因为需要解析zip文件结构里特定属性的值才能判断apk是否非法，所以检测的难度也相对更高。</span></div>