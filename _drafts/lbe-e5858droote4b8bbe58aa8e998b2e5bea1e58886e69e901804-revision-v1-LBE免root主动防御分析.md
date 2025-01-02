---
id: 2197
title: LBE“免root主动防御”分析
date: '2025-01-02T12:16:46+08:00'
author: colordancer
layout: revision
guid: 'http://colordancer.net/blog/?p=2197'
permalink: '/?p=2197'
---

 <span style="color:#FF0000;">**注：转载请注明出处**</span>

 <span style="line-height: 1.6em;">LBE在新版本V5.1中增加了"免ROOT"的功能，可以在不root的情况下实现root后才有的功能，比如卸载系统软件、主动防御等。</span>

 经过百度安全实验室的分析，LBE的免ROOT功能是利用Android签名验证漏洞（编号9695860）替换了系统应用SettingsProvider，然后在SettingsProvider进程里以System权限加载执行LBE自己的功能模块，达到root后才能实现的功效。下面我们来分析这个"免ROOT"实现的主要步骤。

 开启LBE的"免root启动"主动防御后，LBE会提示用户修复MasterKey漏洞，如果用户同意修复，LBE会连接到服务器端下载所在系统SettingsProvider对应版本的补丁到/data/data/com.lbe.security/files/lbe\_patch，然后安装。安装完毕后，提示重启，重启后即可实现免root主动防御功能。

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot1.png)

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot2.png)

 安装lbe\_patch：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot3.png)

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot4.png)

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot5.png)

 这个补丁文件lbe\_patch是一个修改过的SettingsProvider安装包。安装过程中，利用了Android签名验证漏洞#9695860，覆盖系统自带的SettingsProvider。关于该漏洞的具体分析，可以参见百度安全实验室发布的另一篇文章(链接)。

 从lbe\_patch的文件结构可以看到，CERT.RSA的Comment大小被设置为0x8000，致使java在解析lbe\_patch时，处理完cert.rsa这个central directory后，会紧跟着解析后续的central director；而C++则跳转到0x8000后的地址解析。

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot6.png)

 所以，在验证签名时，PackageParser.java将lbe\_patch解析成如下结构：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot7.png)

 而C++会将其解析成：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot8.png)

 对比下系统自带的SettingsProvider包的结构：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot9.png)

 可以看到， java解析lbe\_patch后的变化有两个：

1. <div style="text-align: justify"> <span style="font-size:10pt">新增了随机命名的两个文件夹</span> </div>
2. <div style="text-align: justify"> <span style="font-size:10pt">缺少AndroidManifest.xml</span> </div>

 那lbe\_patch是怎么绕过Android签名验证的？PackageParser.java在验证JarEntry的签名代码如下：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot10.png)

 从上面的代码可以看到：

1. <div style="text-align: justify"> <span style="font-size:10pt">PackageParser会直接略过文件夹的签名验证，所以新增的两个文件夹不会对签名校验有影响，它们的作用是保证java解析CentralDirectory的个数达到ZipEndLocator中记录的CentralDirectory的个数，这两个文件夹正好用来冲抵AndroidManifest.xml和classes.dex。</span> </div>
2. <div style="text-align: justify"> <span style="font-size:10pt">验证签名时，PackageParser会遍历Apk里的每个ZipEntry，获得它的校验值，然后和apk的meta-inf里存储的每个文件的校验值做比较，不难发现，缺少的文件不会做比较，不会影响签名的校验。所以lbe\_patch在java层解析时虽然缺少AndroidManifest.xml，但是签名验证还是可以通过。</span> </div>

 至此，lbe\_patch安装成功，而实际在运行中调用的是c++层解析后的安装包，里面包含了classes.dex和AndroidManifest.xml。

 AndroidManifest.xml中记录了patch version，在安装时校验，并且新注册了个com.lbe.security.mkservice服务，供lbe后续调用：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot11.png)

 Classes.dex代码结构：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot12.png)

 可以看到里面包含了com.android.providers.settings代码。MkPayload是lbe\_patch的入口类，里面包含了main函数，用来做一些初始化，反射调用和加载主防等模块：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot13.png)

 这个main函数何时被调用？LBE修改了com.android.providers.setting的代码，在SettingsProvider的构造函数里调用了这个main函数，所以LBE的代码会跟着SettingsProvider一起加载执行：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot14.png)

 至此，LBE的代码已经获得System权限，即可在SettingsProvider进程里加载其他功能模块，实现"rootFree"功能：

 ![](/images/wp-content/uploads/2013/10/102813_0846_LBEroot15.png)

 对于LBE提出的免root功能，确实在一定程度上解决了root带来的问题，比如root后可能有些手机将无法保修，或者部分普通用户不会root，但某些场景下又确实需要root带来的功能。但是目前LBE提出的这个方案也存在一些问题：

1. <div style="text-align: justify"> <span style="font-size:10pt">首先LBE的rootFree功能基于Android系统漏洞的前提，安全厂商利用系统漏洞替换系统模块，这并不应该是一个安全厂商应有的行为；</span> </div>
2. <div style="text-align: justify"> <span style="font-size:10pt">其次，LBE利用#9695860漏洞绕过签名验证，但是却是在没有给出该漏洞解决方案的前提下利用了该漏洞，这样势必会导致该漏洞的爆发，我们预计利用该漏洞的案例将会越来越多，甚至包括病毒也将会利用该漏洞进行破坏和牟利；</span> </div>
3. <div style="text-align: justify"> <span style="font-size:10pt">最后，经过我们的分析和测试，鉴于该技术涉及到技术环节较多，加上Android系统的开发性，LBE的rootFree功能尚不成熟稳定，使用时存在一定的风险。</span> </div>

 百度安全实验室近日已经推出DroidSploit方案，可以查杀该漏洞和利用该漏洞的程序。