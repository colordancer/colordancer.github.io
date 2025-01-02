---
id: 2199
title: 'Android 4.3中SE Android展望'
date: '2025-01-02T12:16:46+08:00'
author: colordancer
layout: revision
guid: 'http://colordancer.net/blog/?p=2199'
permalink: '/?p=2199'
---

 <span style="line-height: 1.6em;">Android 4.3中增加了一些功能，比如OpenGL SE 3.0。但是大部分报道中却没有提到安全性方面的一个改进：SEAndroid。Android在4.3中"正式"引进了SEAndroid，这将对Android系统未来的安全体制带来不可忽略的影响。</span>

 <span style="line-height: 1.6em;">SEAndroid是SELinux in Android。SELinux全称Security Enhanced Linux，即安全增强版Linux，它并非一个Linux发布版，而是一组可以套用在类Unix操作系统（如Linux、BSD等）的修改，主要由美国国家安全局（NSA）开发，已经被集成到2.6版的Linux核心之中，现已有十几年的开发和使用历史，是Linux上最杰出的安全子系统。</span>

 <span style="line-height: 1.6em;">标准的UNIX安全模型是"任意的访问控制"DAC（Discretionary Access Control），任何程序对其资源享有完全的控制权，这些控制权由用户自己定义。SELinux的安全模型则是MAC（Mandatory Access Control），即强制访问控制，它的做法是"最小权限原则"，通过定义哪些用户可以访问哪些文件，从而提供了一系列的机制，来限制用户和文件的权限。</span>

 ![](/images/wp-content/uploads/2013/08/082713_0221_1.gif)

 <span style="line-height: 1.6em;">SELinux有disabled、permissive、enforcing3种模式：</span>

- <div style="text-align: justify"> <span style="font-size:10pt">disabled：禁用SELinux</span> </div>
- <div style="text-align: justify"> <span style="font-size:10pt">permissive：SELinux开启状态，但不做任何策略的拦截。即使你违反了某策略，SELinux在permissive模式下回让你继续操作，只不过会把这条违反行为记录下来。</span> </div>
- <div style="text-align: justify"> <span style="font-size:10pt">Enforcing：相比permissive模式，在enforcing模式下，你违反了某策略，SELinux就会把你拦截下来，无法继续操作</span> </div>

 <span style="line-height: 1.6em;">在4.3版本之前，比如4.2，Android也或多或少在代码里集成了SElinux的部分模块，但是这些代码并没有被激活。从4.3版本开始，Android正式引入SELinux，并且开启了SELinux功能。目前的4.3版本中，作为一个SEAndroid "Beta"版本，SELinux的运行模式为permissive，即只会记录，不会拦截。</span>

 <span style="line-height: 1.6em;">可以在Android shell通过setenforce命令来设置SELinux的模式。</span>

 ![](/images/wp-content/uploads/2013/08/082713_0221_2.png)

 <span style="line-height: 1.6em;">查看文件的权限状态：</span>

 ![](/images/wp-content/uploads/2013/08/082713_0221_3.png)

 <span style="line-height: 1.6em;">其中u:object\_r:rootfs:s0中有4个字段，分别是user, role, type, security-level，其中最重要的是type，所有的policy都围绕type展开。</span>

 <span style="line-height: 1.6em;">在Android 4.3源码中，这些策略文件被放到了源码目录external/sepolicy中，在Android.mk文件中描述了相关编译过程，首先会使用m4预处理器将sepolicy中的所有相关文件整合成一个源文件plicy.conf，然后通过checkpolicy 编译器将policy.conf策略源文件编译成sepolicy.24的二进制策略文件（24为策略版本号）。</span>

 ![](/images/wp-content/uploads/2013/08/082713_0221_4.png)

 <span style="line-height: 1.6em;">通过对policy.conf的查看，我们可以看到Android已经为系统默认配置了一些权限，编译后的seplicy文件位于系统根目录下，可以用seinfo命令查看。</span>

 ![](/images/wp-content/uploads/2013/08/082713_0221_5.png)

 <span style="line-height: 1.6em;">可以看到一共有43个Permissive的权限设置，我们尝试enforce一下：</span>

 ![](/images/wp-content/uploads/2013/08/082713_0221_6.png)

 可以看到setenforce其实并没有起作用，进一步确认了Android 4.3中Selinux是在permissive模式下运行的，手动setenforce也不起作用。

 <span style="line-height: 1.6em;">Android引入SELinux之后，被讨论最多的一个变化就是root。由于Android目前为每个应用设立了一个单独用户用来限制每个进程的访问权先，所以只要不root，android平台就相对来讲安全很多。在没有使用SELinux的android系统上，一旦手机被root，用户就获得了su权限，就可以对系统文件和其他应用进行操作。如果启用了SELinux，管理员就可以设置策略，限定su的访问，比如可以设置su不可以修改系统文件，这样就算手机被root，也可以保障android系统不被恶意篡改。</span>

 ![](/images/wp-content/uploads/2013/08/082713_0221_7.png)

 <span style="line-height: 1.6em;">（图片转自：</span><http://blog.csdn.net/yiyaaixuexi/article/details/8490886><span style="line-height: 1.6em;">）</span>

 <span style="line-height: 1.6em;">另外一个关于root的敏感话题是"root手机"。目前android上大部分的root方法是在注入代码到具有su权限系统模块中去，对于启用了SELinux的android系统，就可以配置策略禁用对系统模块进行类似的操作，比如发送socket数据，所以SELinux的启用一定程度上也增加了root的难度。</span>

 <span style="line-height: 1.6em;">由于SELinux引入android不久，还有很多不完善的地方。在DEFCON 21上，来自viaForensics的Pau Oliva就演示了几个方法来绕过SEAndroid(</span><https://viaforensics.com/mobile-security/implementing-seandroid-defcon-21-presentation.html><span style="line-height: 1.6em;">)：</span>

1. <div style="text-align: justify"> <span style="font-size:10pt">用恢复模式（recovery）刷回permissive模式的镜像</span> </div>
2. <div style="text-align: justify"> <span style="font-size:10pt">Su超级用户没有设置SELinux的模式，但是system user系统用户可以。</span> </div>
3. <div style="text-align: justify"> <span style="font-size:10pt">Android通过/system/app/SEAndroidManager.apk来设置SELinux模式，所以只要在recovery模式下将其删除就可以绕过</span> </div>
4. <div style="text-align: justify"> <span style="font-size:10pt">在Android启动时直接操作内核内存，通过将内核里的unix\_ioctl符号改写成reset\_security\_ops重置LSM（Linux Security Modules）</span> </div>

 <span style="line-height: 1.6em;">但是从上面的4个方法不难看出，这些方法都是基于APP之外，也就是说无法通过应用本身，或者说应用利用系统的漏洞来绕过SELinux的保护。从目前PC上SELinux的效果来看，我们有理由相信Android引进SELinux之后，Android上的系统安全会得到很大改善。</span>