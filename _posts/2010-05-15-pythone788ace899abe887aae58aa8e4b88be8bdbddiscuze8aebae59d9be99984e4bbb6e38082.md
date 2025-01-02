---
id: 325
title: Python爬虫自动下载Discuz论坛附件。
date: '2010-05-15T05:55:42+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=325'
permalink: /2010/05/15/python%e7%88%ac%e8%99%ab%e8%87%aa%e5%8a%a8%e4%b8%8b%e8%bd%bddiscuz%e8%ae%ba%e5%9d%9b%e9%99%84%e4%bb%b6%e3%80%82/
views:
    - '15426'
categories:
    - 开发设计
tags:
    - discuz
    - python
    - 爬虫
---

因工作需要，要定期收集卡饭论坛的病毒样本板块的病毒样本，所以就考虑用 Python做个爬虫，然后自动下载附件。

核心功能有3个：

1· 登录  
2· 伪造cookie保持session  
3\. 下载样本  
  
首先，登录就是先抓取登录页面，找到登录表单会post的数据，当然你也可以用firefox的httpfox插件。

需要注意的是，discuz的登录表单里有个hashform字段，是会随时间变的，所以要登录，必须分两个步骤：

1· 先抓取登录页面，找到hashform值

2· 生成post数据，然后登录 登录成功后，服务器端会返回给我们两个cookie字段，我本来是想先解析这些cookie，然后再生成自己的cookie，作为每次post的数据之一。后来发现cookielib可以安装opener，所以你只要用urllib2.urlopen(req)来取代urllib.urlopen(uri)，返回的cookie每次就会被保存，并且自动包在每次发送的请求里。

接下来就是解析网页，获得附件的下载地址了。解析网页无非就是正则。没有什么新的技术含量，就不多说了。

下面上代码，给需要类似功能的朋友做参考。代码写的乱，就不要见怪了。 帖子列表，我是从板块的RSS中获得的。

\[python\]  
import urllib,urllib2,cookielib,re,datetime

def getPageHtml(uri):  
 req = urllib2.Request(uri)  
 return urllib2.urlopen(req).read()  
 #return urllib.urlopen(uri).read()

def login():  
”’登陆论坛

设置cookie，获得formhash，然后提交post数据 ”’

\#获得formhash  
 pattern = re.compile(“<input formhash="" hidden="" name="\" type="\" value="\"></input>“)  
 content = getPageHtml(‘http://bbs.kafan.cn/logging.php?action=login’)  
 formhash = pattern.findall(content)  
 if (len(formhash) &gt; 0):  
 formhash = formhash\[0\]  
 formhash = formhash\[-12:-4\]

 #cookie  
 cookieJar = cookielib.CookieJar()  
 cookie\_support= urllib2.HTTPCookieProcessor(cookieJar)  
 opener = urllib2.build\_opener(cookie\_support, urllib2.HTTPHandler)  
 urllib2.install\_opener(opener)

 #login  
 postdata=urllib.urlencode({  
 ‘loginfield’:’username’,  
 ‘username’:’用户名’,  
 ‘password’:’密码’,  
 ‘referer’:’http://bbs.kafan.cn/’,  
 ‘formhash’:formhash,  
 ‘questionid’:’0′,  
 ‘answer’:”  
 })

 headers = {  
 ‘User-Agent’:’Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.1.6) Gecko/20091201 Firefox/3.5.6′,  
 ‘referer’:’http://bbs.kafan.cn’  
 }

 req = urllib2.Request(  
 url = ‘http://bbs.kafan.cn/logging.php?action=login&amp;loginsubmit=yes&amp;inajax=1’,  
 data = postdata,  
 headers = headers  
 )  
 result = urllib2.urlopen(req).read()

def getPages():  
 page = getPageHtml(‘http://bbs.kafan.cn/rss.php?fid=31&amp;auth=0’)  
 pattern = re.compile(“.\*viewthread.php.\*&lt; \\/link&gt;”)  
 linkArray = pattern.findall(page)  
 return linkArray

def getLinks(urls):  
\#遍历页面  
 count = 1  
 for url in urls:  
 url = url\[6:-7\]  
 print “解析” + url  
 pageContent = getPageHtml(url)  
 #print pageContent  
 pattern = re.compile(‘[.\*;’)  
 anchors = pattern.findall(pageContent)  
 #遍历下载节点  
 for anchor in anchors:  
 print anchor  
 linkPattern = re.compile(‘\\”attachment\\.php\\?aid=\[a-zA-Z0-9\\%&amp;;=\\?-\_\\B\]\*\\”‘)  
 link = linkPattern.findall(anchor)  
 link = “http://bbs.kafan.cn/” + link\[0\]\[1:-1\]  
 namePattern = re.compile(‘&gt;;\[^;\].\*\[^;\]](\"attachment\.php\?aid=.*)