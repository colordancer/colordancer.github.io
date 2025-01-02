---
id: 1740
title: 'Android签名漏洞#8219321揭秘'
date: '2013-07-11T13:33:48+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1740'
permalink: /2013/07/11/android%e7%ad%be%e5%90%8d%e6%bc%8f%e6%b4%9e%e6%8f%ad%e7%a7%98/
views:
    - '1901'
categories:
    - 安全研究
tags:
    - android
    - 'master key'
---

 上周，移动安全公司Bluebox Security研究人员宣布他们发现了一个Android严重漏洞，这个漏洞允许攻击者修改应用程序的代码但不会改变其加密签名。据说，这个漏洞自Android 1.6（Donut）以来就一直存在，隐藏长达4年之久，当今市场上99%的Android产品都面临这一问题。

 每个Android程序是一个apk文件，从文件格式上来说，apk其实是一个zip压缩包。这个压缩包里包含了android程序的所有内容，包括配置文件，编译后的程序代码（classes.dex），程序依赖的资源文件，以及加密签名。Android 系统会根据这个签名来验证apk文件是否合法，以防止正常软件被非法篡改。同一个应用（指包名相同）如果签名不同则不能覆盖安装。所以，可想而知，如果一旦签名被绕过，android应用就会真假难辨。

 **原理：**

 Android签名漏洞的原因在于android系统在安装apk的过程中，检验签名的逻辑存在一点纰漏。Android程序安装模块利用一个HashMap数据结构（下图中的mEntries）存放压缩包里的文件信息：

 [![](/images/wp-content/uploads/2013/07/071113_0533_Android1.jpg)](http://seclab.safe.baidu.com/wp-content/uploads/wp-display-data.php?filename=1373442827bluebox1.jpg&type=image%2Fjpeg&width=490&height=105)

 HashMap是一个字典型数据结构，不允许索引重复。而这个mEntries里的索引正是压缩包里的文件名，所以android如果发现压缩包里同一路径下存在两个同名的文件，先前存放在map中的文件信息会被后存放的同名文件覆盖。如果仅仅是覆盖，那还不会引起问题。可是android程序在执行的时候，却是根据文件名从压缩包里获取程序代码和资源文件。

 [![](/images/wp-content/uploads/2013/07/071113_0533_Android2.png)](http://seclab.safe.baidu.com/wp-content/uploads/wp-display-data.php?filename=1373442828bluebox2.png&type=image%2Fpng&width=591&height=631)

 同名的两个文件，在文件流上靠前的那个文件会被android加载。所以，只要保证添加进apk压缩包里的恶意文件在文件流上处于同名正常文件之前，就能保证该恶意文件绕过签名严重，并能被android加载。

 **构造方法：**

 用一般的方法往zip压缩包里写两个同名文件是不可能的，旧的文件会被新的文件覆盖掉，所以只能依赖文件流的操作。这里我们用的python的zipfile库。

 [![](/images/wp-content/uploads/2013/07/071113_0533_Android3.png)](http://seclab.safe.baidu.com/wp-content/uploads/wp-display-data.php?filename=13735201431.PNG&type=image%2Fpng&width=516&height=102)

 用Python打开apk，然后直接往里面写文件，就可以实现同名文件。

 但新写进去的同名文件（这里称作B）在文件流上实际上处于旧文件（称作A）的后面，这样还是绕不过android的签名检测，这里有个小技巧：

 <span style="line-height: 1.6em;">把A从压缩包里提取出来，然后把它再写到压缩包里，这里称作C。所以在压缩包里，这3个文件的顺序就是ABC，而C==A，所以再把A从压缩包里删除，这样就实现了原始文件在新文件之后了。</span>

 [![](/images/wp-content/uploads/2013/07/071113_0533_Android4.png)](http://seclab.safe.baidu.com/wp-content/uploads/wp-display-data.php?filename=13735201442.PNG&type=image%2Fpng&width=627&height=423)

 下图显示研究人员成功修改android 2.3中默认浏览器的图标：

 [![](/images/wp-content/uploads/2013/07/071113_0533_Android5.png)](http://seclab.safe.baidu.com/wp-content/uploads/wp-display-data.php?filename=1373442829bluebox3.png&type=image%2Fpng&width=320&height=480)

 [![](/images/wp-content/uploads/2013/07/071113_0533_Android6.png)](http://seclab.safe.baidu.com/wp-content/uploads/wp-display-data.php?filename=1373442830bluebox4.png&type=image%2Fpng&width=319&height=478)

 另外，如果仅仅替换classes.dex或者资源文件，因为版本号一致的缘故，android重启之后会将恶意的apk删除，保留原先正常的apk。所以，还需要用同样的方法修改一份AndroidManifest.xml配置文件。

 **危害：**

 1·如果您的机器已root，您必须多加小心，因为android系统程序会被病毒通过这个漏洞替换。一旦系统应用被篡改，那后果将不堪设想。

 2·您下载的第三方应用可能是"假冒"的，因为虽然签名可能一致，但apk压缩包里的文件可能已经被篡改了。

 **防护：**

 对于利用这个漏洞的病毒检测，只要判断压缩包里是否存在同名文件即可。如果不是必须，我们建议您不要root您的手机。另外，最好从可信赖的安全渠道下载应用，比如Google Play，百度应用市场（<http://app.baidu.com.cn/>），以及其他有审核机制的第三方应用市场。尽量避免通过论坛下载应用，各种手机论坛是恶意应用的相对重灾区。