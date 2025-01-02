---
id: 1176
title: 'Virustotal Uploader patched &#8211; VTU优化版。'
date: '2011-07-03T13:13:10+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1176'
permalink: /2011/07/03/virustotal-uploader-patched-vtu%e4%bc%98%e5%8c%96%e7%89%88%e3%80%82/
views:
    - '1857'
categories:
    - 安全研究
---

Virustotal uploader，简称VTU，是一个可以上传文件到virustotal.com并常看扫描结果的客户端工具。现在的最新版本是2.0，默认的是安装版本，目前已经被汉化绿化。

用了几次之后，我发现VTU有个不符合使用习惯的地方。VTU如果发现文件的扫描结果在virustotal.com上已经存在，默认会直接打开最新扫描结果的页面，然后才会让你选择是否“重新上传”；如果你选择了“重新上传”，VTU就打开Virustotal上让你选择“reanalyse”还是“view latest report”的页面，也就是说，你还得再点击“reanalyse”才能打开最后的重新扫描的结果页面。

我觉得这个有点累赘了，因为如果你需要重新扫描文件的话，你得操作两次才能得到结果。我觉得应该在一开始就给用户一个选择“直接查看最新扫描结果”还是“重新扫描”的机会。

周末没啥事，也不想贪玩，就给VTU做了点小手术。patch了两块地方：  
1：如果发现文件扫描结果已经在virustotal上存在，弹出对话框，让用户选择是“打开已有的最新报告页面”还是“打开重新扫描的结果页面”。  
2：如果选择了“打开重新扫描的结果页面”，则直接跳转到重新扫描的页面，而非“选择reanalysze还是view latest report”的页面。

修改后的运行界面如下：  
[![](/images/wp-content/uploads/2011/07/VTU-patched-600x360.jpg "VTU patched")](http://www.colordancer.net/blog/2011_07_virustotal-uploader-patched-vtu%e4%bc%98%e5%8c%96%e7%89%88%e3%80%82/vtu-patched)

[](http://www.colordancer.net/blog/2011_07_virustotal-uploader-patched-vtu%e4%bc%98%e5%8c%96%e7%89%88%e3%80%82/vtu-patched)

下载地址：[VTU2.0](http://www.colordancer.net/blog/2011_07_virustotal-uploader-patched-vtu%e4%bc%98%e5%8c%96%e7%89%88%e3%80%82/vtu2-0)

注：本软件的所有权归软件作者所有，请勿将修改后的软件用于商业或不法用途，否则引起的一切法律责任问题，与本人无关。by colordancer