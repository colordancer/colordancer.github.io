---
id: 1024
title: 用NetFilter在Linux下实现一个小型防火墙MiniFirewall。
date: '2011-03-27T21:22:07+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1024'
permalink: /2011/03/27/%e7%94%a8netfilter%e5%9c%a8linux%e4%b8%8b%e5%ae%9e%e7%8e%b0%e4%b8%80%e4%b8%aa%e5%b0%8f%e5%9e%8b%e9%98%b2%e7%81%ab%e5%a2%99minifirewall%e3%80%82/
views:
    - '1682'
classic-editor-remember:
    - classic-editor
categories:
    - 开发设计
---

这是去年十月份刚来北京时帮朋友做的一个poc。现在发出来应该没什么问题了。当时做的时候，遇到了不少问题，发出来希望能给他人有所帮助。  
[![](/images/wp-content/uploads/2011/03/无标题.gif "无标题")](http://www.colordancer.net/blog/2011_03_%e7%94%a8netfilter%e5%9c%a8linux%e4%b8%8b%e5%ae%9e%e7%8e%b0%e4%b8%80%e4%b8%aa%e5%b0%8f%e5%9e%8b%e9%98%b2%e7%81%ab%e5%a2%99minifirewall%e3%80%82/%e6%97%a0%e6%a0%87%e9%a2%98-2)  
基本原理：  
1\. server端：  
a.编译成Kernal Module，然后利用NetFilter建立挂钩函数，在挂钩函数里做包的处理工作。  
b.建立文件系统/proc存储防火墙规则  
2\. Client端：通过对文件系统/proc的读写访问，与server端交互，进行防火墙规则的设定

遇到的问题：  
1\. Linux编译的问题  
我下了几个版本的开源linux系统，包括redhat和ubuntu，几乎大部分版本都有一个问题：linux的内核版本和source的版本不一致。而我们的服务端模块要编译成内核模块，编译的时候可以成功，但是加载到内核的时候，会提示“insmod failed…”之类的问题。解决方法是更新linux内核。更新linux内核是个痛苦的事情，要确保你的source和内核的名字是一字不差。网上有很多编译内核的方法，基本都是复制+粘贴，无效。以下是我编译ubuntu的过程：

\[code\]  
1\. 去kernel.org下载与你编译程序时版本一下的source  
2\. 将该source解压到 /usr/src/ 目录  
3\. 在控制台依次执行以下命令，完成内核编译：

\* cd /usr/src/Linux-X.X.XX  
\* make menuconfig  
\* make  
\* make modules  
\* make modules\_install  
\* make install  
\* make bxImage  
\* grub-update2  
\* reboot  
\[/code\]

2\. 从socket包里获得各属性的方法，随着linux内核版本的不一样而不一样。这时候你要学会从linux source中找出对应的头文件去查找

3\. 文件系统读写。在做到过程中，我发现不是每个文件API都能成功读写/proc文件系统的。最后我索性使用了memcpy

下面就直接上代码了：

server端：

```
[c]

#include <linux/kernel.h>
#include <linux/module.h>

#include <linux/proc_fs.h>
#include <linux/string.h>

#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <linux/skbuff.h>
#include <linux/tcp.h>
#include <linux/ip.h>

#include "minifw_common.h"

MODULE_LICENSE("GPL");

static struct proc_dir_entry *proc_entry;
static struct nf_hook_ops nfho;

unsigned int hook_func(unsigned int hooknum, struct sk_buff* skb, const struct net_device* in, const struct net_device* out, int(*okfn)(struct sk_buff*))
{
 if (!skb)
 return NF_ACCEPT;

 //get skb value
 struct iphdr* pIph = (struct iphdr *)skb_network_header(skb);
 unsigned int srcIp = pIph->saddr;
 unsigned int desIp = pIph->daddr;
 unsigned int protocol = pIph->protocol;
 //struct tcphdr* pTcph=(struct tcphdr*)(sb->data+(sb->nh.iph->ihl*4));//old version
 struct tcphdr* pTcph = (struct tcphdr *)skb_transport_header(skb);
 unsigned int srcPort = pTcph->source;
 unsigned int desPort = pTcph->dest;

 printk("minifw: Received ");
 switch(protocol)
 {
 case P_UDP:
 printk("UDP");break;
 case P_TCP:
 printk("TCP");break;
 case P_ICMP:
 printk("ICMP");break;
 default:
 printk("Other");break;
 }
 
 printk(" packet from %d.%d.%d.%d:%d to %d.%d.%d.%d:%d",
 srcIp&0x000000FF,
 (srcIp&0x0000FF00)>>8,
 (srcIp&0x00FF0000)>>16,
 (srcIp&0xFF000000)>>24,
 ((srcPort&0xFF00)>>8|(srcPort&0x00FF)<<8),
 desIp&0x000000FF,
 (desIp&0x0000FF00)>>8,
 (desIp&0x00FF0000)>>16,
 (desIp&0xFF000000)>>24,
 ((desPort&0xFF00)>>8|(desPort&0x00FF)<<8)                
 );

 //find skb value in policy or not
 int i = 0;
 for (; i/6 < sz_policies[MAX_ARR_COUNT - 1]; i+=6)
 {
 if ( sz_policies[i+5] == ACTION_BLOCK )
 {
 if ( sz_policies[i+4] == protocol || sz_policies[i+4] == P_ALLP)
 {
 if ( ( (srcIp == sz_policies[i+0] || sz_policies[i+0] == 0) && (srcPort == sz_policies[i+1] || sz_policies[i+1] == 0) ) ||
 ( (desIp == sz_policies[i+2] || sz_policies[i+2] == 0) && (desPort == sz_policies[i+3] || sz_policies[i+3] == 0) ) )
 {
 printk(" - Action: BLOCK\n");
 return NF_DROP;
 }
 }
 }    
 }

 printk(" - Action: UNBLOCK\n");
 return NF_ACCEPT;
}



ssize_t minifw_write( struct file *filp, const char __user *buff, unsigned long len, void *data )
{       
 if (copy_from_user( sz_policies, buff, len )) {
 return -EFAULT;
 }
 
 printk("minifw: Write ok. Policy count:%d\n", sz_policies[MAX_ARR_COUNT-1]);

 return len;
}

int minifw_read( char *buff, char **start, off_t off, int count, int *eof, void *data )
{
 if (off > 0)
 {
 return 0;
 }
 memcpy(buff,sz_policies, sizeof(sz_policies));
 printk("minifw: Read ok. Policy count:%d\n", sz_policies[MAX_ARR_COUNT-1]);
 return sizeof(sz_policies);
}

int minifw_init()
{
 int ret = 0;
 proc_entry = create_proc_entry( "minifw", 0644, NULL );
 if (proc_entry == NULL) {
 ret = -ENOMEM;
 printk(KERN_INFO "minifw: Couldn't create proc entry.\n");
 } else {
 proc_entry->read_proc = minifw_read;
 proc_entry->write_proc = minifw_write;
 printk(KERN_INFO "minifw: Module loaded.\n");
 
 nfho.hook=hook_func;    
 nfho.hooknum=NF_INET_PRE_ROUTING;    
 nfho.pf=PF_INET;    
 nfho.priority=NF_IP_PRI_FIRST;
 
 nf_register_hook(&nfho);
 }

 printk("minifw: Loading of module successful.\n");
 
 return ret;
}

void minifw_cleanup()
{    
 remove_proc_entry("fortune", NULL);
 nf_unregister_hook(&nfho);
 printk("minifw: Removal of module successful.\n");
 return;
}

module_init( minifw_init );
module_exit( minifw_cleanup);


[/c]

控制台：
 [c]

#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/in.h>

#include "minifw_common.h"

void displayPolicy()
{
 int i = 0;
 printf("Policy count: %d\n",policyCount);
 for (; i/6 < policyCount; i+=6)
 {
 printf("Policy %d: ",i/6+1);
 //action
 if (sz_policies[i+5] == ACTION_BLOCK)
 printf("Block ");
 else
 printf("Unblock ");

 //protocol
 switch(sz_policies[i+4])
 {
 case P_UDP:
 printf("UDP");break;
 case P_TCP:
 printf("TCP");break;
 case P_ICMP:
 printf("ICMP");break;
 case P_ALLP:
 printf("ALL");break;
 default:
 printf("Unknown");break;
 }

 printf(" packets from ");                    

 //source ip
 if (sz_policies[i] == 0)
 printf("[all ip]");
 else
 printf("%d.%d.%d.%d",sz_policies[i]&0x000000FF,(sz_policies[i]&0x0000FF00)>>8,    (sz_policies[i]&0x00FF0000)>>16,
 (sz_policies[i]&0xFF000000)>>24);
 
 printf(":");

 //source port
 if (sz_policies[i+1] == 0)
 printf("[all ports]");
 else
 printf("%d",((sz_policies[i+1]&0xFF00)>>8|(sz_policies[i+1]&0x00FF)<<8));

 printf(" to ");

 //des ip
 if (sz_policies[i+2] == 0)
 printf("[all ip]");
 else
 printf("%d.%d.%d.%d",sz_policies[i+2]&0x000000FF,(sz_policies[i+2]&0x0000FF00)>>8,    (sz_policies[i+2]&0x00FF0000)>>16,
 (sz_policies[i+2]&0xFF000000)>>24);

 printf(":");

 //des port
 if (sz_policies[i+3] == 0)
 printf("[all ports]");
 else
 printf("%d",((sz_policies[i+3]&0xFF00)>>8|(sz_policies[i+3]&0x00FF)<<8));

 printf("\n");
 }

}

void comments()
{
 printf ("Example:\nminifw_console.o -a BLOCK -s 127.0.0.1 -i 17664 -d 127.0.0.1 -o 84 -p ALL\n");
 printf("-a: action\n");
 printf("-s: from ip\n");
 printf("-i: from port\n");
 printf("-d: destination ip\n");
 printf("-o: destination port\n");
 printf("-p: protocol\n");
 printf("-l: display all policies\n");
 printf("Notice: If you want to BLOCK/UNBLOCK all IP or all ports, please type 0\n");
}

//minifw.o -a BLOCK -s 127.0.0.1 -i 1111 -d 123.123.123.123 -o 1111 -p ALL
int main(int argc, char* argv[])
{
 int protocalFlag     = 0;
 int srcIpFlag         = 0;
 int desIpFlag        = 0;
 int srcPortFlag     = 0;
 int desPortFlag        = 0;
 int actionFlag         = 0;
 int listFlag        = 0;
 char *protocalVal     = NULL;
 char *srcIpVal        = NULL;
 char *desIpVal        = NULL;
 char *srcPortVal    = NULL;
 char *desPortVal    = NULL;
 char *actionVal        = NULL;
 
 //read from /proc
 FILE* f = fopen("/proc/minifw","r+");
 if (f)
 {
 //fgets(sz_policies,MAX_ARR_COUNT+1,f);
 if (fread(sz_policies,sizeof(sz_policies),1,f) > 0)
 {
 policyCount = sz_policies[MAX_ARR_COUNT-1];
 printf("Read policy ok! Policy count: %d\n",policyCount);
 }
 fclose(f);    
 }
 else
 {
 printf("Open /proc/minifw/ failed\n");
 return 0;
 }

 //read input
 int c;
 while ((c = getopt (argc, argv, "a:s:i:d:o:p:l")) != -1)
 {
 switch (c)
 {
 case 'a':
 actionFlag = 1;
 actionVal  = optarg;
 break;
 case 's':
 srcIpFlag = 1;
 srcIpVal  = optarg;
 break;
 case 'i':
 srcPortFlag = 1;
 srcPortVal  = optarg;
 break;
 case 'd':
 desIpFlag    = 1;
 desIpVal    = optarg;
 break;
 case 'o':
 desPortFlag    = 1;
 desPortVal    = optarg;
 break;
 case 'p':
 protocalFlag= 1;
 protocalVal = optarg;
 break;
 case 'l':
 listFlag = 1;
 displayPolicy();
 break;
 case '?':
 default:
 comments();
 return;    
 }
 }

 if (!actionFlag && !srcIpFlag && !srcPortFlag && !desIpFlag && !desPortFlag && !protocalFlag)
 {
 if (!listFlag)
 {
 printf ("Format error!\n");
 comments();
 }
 return;
 }

 if (policyCount >= MAX_POLICY_COUNT)
 {
 printf ("Max policies already: %d\n",MAX_POLICY_COUNT);
 return 0;
 }

 //set policy value
 int srcIp,srcPort,desIp,desPort;
 srcIp = srcPort = desIp = desPort = 0;
 if (srcIpFlag)
 srcIp = inet_addr(srcIpVal);
 if (srcPortFlag)
 srcPort = htons(atoi(srcPortVal));
 if (desIpFlag)
 desIp = inet_addr(desIpVal);
 if (desPortFlag)
 desPort = htons(atoi(desPortVal));
 
 int protocal = P_ALLP;
 if (!strcmp(protocalVal,"UDP"))
 protocal = P_UDP;
 else if (!strcmp(protocalVal,"TCP"))
 protocal = P_TCP;
 else if (!strcmp(protocalVal,"ICMP"))
 protocal = P_ICMP;
 else if (!strcmp(protocalVal,"ALL"))
 protocal = P_ALLP;

 int action = ACTION_UNBLOCK;//unblock
 if (!strcmp(actionVal,"BLOCK"))
 action = ACTION_BLOCK;

 //check policy exists
 int exist = 0;
 int i = 0;
 for (; i/6 < policyCount ; i+=6)
 {
 if ( sz_policies[i] == srcIp &&
 sz_policies[i+1] == srcPort &&
 sz_policies[i+2] == desIp &&
 sz_policies[i+3] == desPort &&
 sz_policies[i+4] == protocal )
 {
 //if ip and port are the same, only action not the same
 //just update the action, do not add into the policy
 //if action also the same, return
 if (sz_policies[i+5] == action)
 {
 exist = 1;
 printf("Policy already exists!\n");
 return;
 }
 else
 {
 //update action
 sz_policies[i+5] = action;
 break;
 }
 }
 }

 //add policy
 if ( !exist)
 {
 sz_policies[policyCount*6] = srcIp;
 sz_policies[policyCount*6+1] = srcPort;
 sz_policies[policyCount*6+2] = desIp;
 sz_policies[policyCount*6+3] = desPort;
 sz_policies[policyCount*6+4] = protocal;
 sz_policies[policyCount*6+5] = action;
 
 policyCount++;
 sz_policies[MAX_ARR_COUNT-1] = policyCount;       
 }
 

 //write /proc
 f = fopen("/proc/minifw","r+");
 if (f)
 {
 fwrite(sz_policies,sizeof(sz_policies),1,f);
 fclose(f);
 printf("Policy add successfully!\n");
 }
 else
 {
 printf("Open /proc/minifw/ failed\n");
 }
 
 return 0;
}

[/c]

代码下载：<a href="http://www.colordancer.net/blog/2011_03_%e7%94%a8netfilter%e5%9c%a8linux%e4%b8%8b%e5%ae%9e%e7%8e%b0%e4%b8%80%e4%b8%aa%e5%b0%8f%e5%9e%8b%e9%98%b2%e7%81%ab%e5%a2%99minifirewall%e3%80%82/assignment3_src" rel="attachment wp-att-1042">assignment3_src</a>
```