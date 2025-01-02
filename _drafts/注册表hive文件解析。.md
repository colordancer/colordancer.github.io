---
id: 1075
title: 注册表hive文件解析。
date: '2011-03-30T23:00:08+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1075'
permalink: '/?p=1075'
views:
    - '3'
categories:
    - 开发设计
---

hive 就是注册表的磁盘文件。有一定的格式，按照一定的格式去分析一个HIVE文件就是HIVE解析，一般好处就是能检测到隐藏的注册表键，或者绕过权限检测访问注册表。

因为HIVE文件比较特殊，一般不能被打开来直接读其内容，所以获取HIVE数据比较重要，一般有3种：

1 RegSaveKey来把一个键及其以 下保存为HIVE文件。缺点是速度太慢，而且可能RegSaveKey会调用失败

2 读DISK。获取HIVE文件的扇区分布，读出来。一般比较简 单是用FSCTL\_GET\_RETRIEVAL\_POINTERS，但可能在某些版本的操作系统不能正确复制或者读。解析文件格式的话又太繁琐，而且速度 还是慢

3 直接访问内存。这种办法很好，速度快。而且用cache的话。可能访问数据不会产生IRP（如果不产生缺页的话） 。这样也可以避免被FSD层或者之下的拦截导致数据被改而隐藏了注册表键。

<span style="color: #ff6600;">1.HIVE结构</span>

首先,要明白的是注册表是由多个hive文件组成.

而一个hive是由许多bin组成,一个bin是由很多cell组成.

而cell可以有好几种类型.比如 key cell(cm\_key\_node) value cell(CM\_KEY\_VALUE) subbkey-list cell,value-list cell等

当新的数据要扩张一个hive时,总是按照block的粒度(4kb)来增加,一个hive的第一个块是base block.包含了有关该hive的全局信息.参考结构 \_hbase\_block

bin也有头部,其中包含了一个特征签名hbin,一个记录该bin在hive中的offset(偏移),以及该bin的大小. bin头的大小一般为0x20.

所以一个hive的磁盘映像看起来如下图;

<span style="color: #00ff00;"> +===============+</span>

<span style="color: #00ff00;"> + \_hbase\_block (4kb) +</span>

<span style="color: #00ff00;"> +===============+</span>

<span style="color: #00ff00;"> + \_bin0 +</span>

<span style="color: #00ff00;"> +===============+</span>

<span style="color: #00ff00;"> + \_bin1 +</span>

<span style="color: #00ff00;"> +===============+</span>

<span style="color: #00ff00;"> + \_bin2 +</span>

<span style="color: #00ff00;"> +===============+</span>

<span style="color: #00ff00;"> + ……… +</span>

<span style="color: #00ff00;"> +===============+</span>

<span style="color: #00ff00;"> + \_bin N +</span>

<span style="color: #00ff00;"> +===============+</span>

bin头的结构\_hbin如下:

\[c\]

typedef struct \_HBIN

{

+0x000 ULONG Signature; //”hbin”字符串  
+0x004 ULONG FileOffset; //本BIN相差0x1000的偏移  
+0x008 ULONG Size; //本BIN的大小  
+0x00c ULONG Reserved1\[2\];  
+0x014 LARGE\_INTEGER TimeStamp;  
+0x01c ULONG Spare;  
+0x020 ULONG Reserved2;  
+0x024  
//  
//这里开始是cell数据  
//  
}  
\[/c\]

所以定位到第一个cell的步骤如下:

<span style="color: #ff0000;">1,将磁盘上的hive文件\*.dat通过CreateFile()和CreateFileMapping()映射到内存,得到hFileMap.</span>

<span style="color: #ff0000;">2,hFileMap加上一个\_base\_block大小(0x1000),定位到bin头,即RootBin=hFileMap+0x1000.</span>

<span style="color: #ff0000;">3,RootBin加上一个\_hbin的大小,即可定位到第一个cell,即pKeyNode=RootBin+0x24;(假设该hive的第一个cell为\_CM\_KEY\_NODE结构)</span>

<span style="color: #ff6600;">2.关于cell的几个结构.</span>

一个cell可以容纳一个键,一个值,一个安全描述,一列子键,或者一列键值.

\[c\]  
typedef struct \_CELL\_DATA  
{  
union \_u //注意它是一个联合体  
{  
CM\_KEY\_NODE KeyNode; //键结构  
CM\_KEY\_VALUE KeyValue;//值的结构, 包含一个键的值的信息.  
CM\_KEY\_SECURITY KeySecurity;//安全描述  
CM\_KEY\_INDEX KeyIndex;//一列子键,又一系列的键cell的cellindex所构成的cell,这些cell是一个公共parent key的所有subkey.  
CM\_BIG\_DATA ValueData;  
HCELL\_INDEX KeyList\[1\];//一列键值,  
WCHAR KeyString\[1\];  
} u;  
} CELL\_DATA, \*PCELL\_DATA;  
\[/c\]

\[c\]  
lkd&gt; DT \_CM\_KEY\_NODE  
nt!\_CM\_KEY\_NODE  
+0x000 Signature : Uint2B //”nk”字符串  
+0x002 Flags : Uint2B  
+0x004 LastWriteTime : \_LARGE\_INTEGER  
+0x00c Spare : Uint4B  
+0x010 Parent : Uint4B  
+0x014 SubKeyCounts : \[2\] Uint4B //SubKeyCounts\[0\]子键的个数  
+0x01c SubKeyLists : \[2\] Uint4B //SubKeyLists\[0\]子键列表相差本BIN的偏移  
+0x024 ValueList : \_CHILD\_LIST //ValueList.Count值的个数  
//ValueList.List值列表相差本BIN的偏移  
+0x01c ChildHiveReference : \_CM\_KEY\_REFERENCE  
+0x02c Security : Uint4B  
+0x030 Class : Uint4B  
+0x034 MaxNameLen : Pos 0, 16 Bits  
+0x034 UserFlags : Pos 16, 4 Bits  
+0x034 VirtControlFlags : Pos 20, 4 Bits  
+0x034 Debug : Pos 24, 8 Bits  
+0x038 MaxClassLen : Uint4B  
+0x03c MaxValueNameLen : Uint4B  
+0x040 MaxValueDataLen : Uint4B  
+0x044 WorkVar : Uint4B  
+0x048 NameLength : Uint2B //键名的长度  
+0x04a ClassLength : Uint2B  
+0x04c Name : \[1\] Uint2B //键名

\[/c\]

补充下\_CHILD\_LIST结构:

\[c\]  
nt!\_CHILD\_LIST  
+0x000 Count : Uint4B //ValueList.Count值的个数  
+0x004 List : Uint4B //ValueList.List值列表相差本BIN的偏移  
\[/c\]

通过观察\_CM\_KEY\_NODE结构,

<span style="color: #3366ff;">我们发现pKeyNode+0x4c获得键名,即RootKeyName=pKeyNode-&gt;Name;</span>

<span style="color: #ff6600;"> 通过ValueList域,可得到值列表相差本bin的偏移,可以定位该键的值结构.即\_CM\_KEY\_VALUE.</span>

<span style="color: #ff6600;"> ValueIndex=(ULONG \*)(Bin+ValueLists+0x4);</span>

<span style="color: #ff6600;"> value=(PCM\_KEY\_VALUE)(Bin+ValueIndex\[i\]+0x4);</span>

<span style="color: #0000ff;">而通过SubKeyLists域,可得到子键列表相差本bin的偏移.</span>

<span style="color: #0000ff;"> DWORD SubKeyLists=KeyNode-&gt;SubKeyLists\[0\];</span>

<span style="color: #0000ff;"> KeyIndex=(PCM\_KEY\_INDEX)(RootBin+SubKeyLists+0x4);</span>

\[c\]

lkd&gt; DT \_CM\_KEY\_VALUE  
nt!\_CM\_KEY\_VALUE  
+0x000 Signature : Uint2B //”vk”字符串  
+0x002 NameLength : Uint2B  
+0x004 DataLength : Uint4B //数据长度，以字节计，包括结束符  
+0x008 Data : Uint4B //注意：数据偏移或数据判断：如果DataLenth最高位为1，那么它就是数据，且DataLenth&amp;0x7FFFFFFF为数据长度  
+0x00c Type : Uint4B  
+0x010 Flags : Uint2B  
+0x012 Spare : Uint2B  
+0x014 Name : \[1\] Uint2B  
\[/c\]

<span style="color: #0000ff;">通过\_CM\_KEY\_VALUE结构,可以获得该键值的类型(type),名字(Name),以及数据(Data)</span>

\[c\]  
lkd&gt; dt \_CM\_KEY\_INDEX  
nt!\_CM\_KEY\_INDEX  
+0x000 Signature : Uint2B  
+0x002 Count : Uint2B //这里也是表示子键的数量，与前面的相同  
+0x004 List : \[1\] Uint4B //这里要注意了  
\[/c\]

如果Signature==CM\_KEY\_FAST\_LEAF或CM\_KEY\_HASH\_LEAF，那么从0x004偏移开始的数组元素结构如下：

\[c\]  
struct

{  
ULONG offset;

ULONG HashKey;  
}

\[/c\]

否则：

\[c\]  
ULONG offset;  
\[/c\]

<span style="color: #0000ff;">对于\_CM\_KEY\_INDEX结构,通过List域可以定位所有子键的\_CM\_KEY\_NODE结构,具体实现如下:</span>

<span style="color: #0000ff;"> DWORD KeyNodeOffset=KeyIndex-&gt;List\[i\*2\]; //(其中的i是,USHORT i=0,i&lt;count,i++)</span>

<span style="color: #0000ff;"> KeyNode=(PCM\_KEY\_NODE)(Bin+KeyNodeOffset+0x4);</span>

这样定位到每个子键的\_CM\_KEY\_NODE结构,就能获得该子键的键值,以及该子键下面的子子键,

这样重复上述的步骤就能实现对整个hive的解析.

ps下:通过是ring3的RegSaveKey(),可以生成一个Hive文件.

具体实现的代码如下:

\[c\]

\#include &lt;windows.h&gt;  
\#include &lt;stdio.h&gt;  
\#include &lt;stdlib.h&gt;

\#define REGF 0x66676572 //fger  
\#define HBIN 0x6e696268 //nibh  
\#define CM\_KEY\_FAST\_LEAF 0x666c // fl  
\#define CM\_KEY\_HASH\_LEAF 0x686c // hl

//数据结构定义  
typedef struct \_CHILD\_LIST  
{  
ULONG Count;  
ULONG List;  
}CHILD\_LIST;

typedef struct \_CM\_KEY\_NODE  
{  
USHORT Signature;  
CHAR Reserve\_1\[18\];  
ULONG SubKeyCounts\[2\];  
ULONG SubKeyLists\[2\];  
CHILD\_LIST ValueList;  
CHAR Reserve\_2\[28\];  
USHORT NameLength;  
SHORT ClassName;  
CHAR Name;  
} CM\_KEY\_NODE,\*PCM\_KEY\_NODE;

typedef struct \_CM\_KEY\_INDEX  
{  
USHORT Signature;  
USHORT Count;  
ULONG List\[1\];  
} CM\_KEY\_INDEX, \*PCM\_KEY\_INDEX;

typedef struct \_CM\_KEY\_VALUE  
{  
USHORT Signature;  
SHORT NameLength;  
ULONG DataLength;  
ULONG Data;  
ULONG Type;  
CHAR Reserve\_1\[4\];  
CHAR Name;  
}CM\_KEY\_VALUE,\*PCM\_KEY\_VALUE;

VOID TpyeKeyAndValue(PVOID FileMemAddr);  
VOID TypeSubKey(PCHAR Bin,PCM\_KEY\_INDEX KeyIndex);  
VOID TypeSubKeyName(PCHAR Bin,PCM\_KEY\_NODE KeyNode);  
VOID TypeValue(PCHAR Bin,PCM\_KEY\_VALUE value);

int main(int argc,char \*argv\[\])  
{  
HANDLE hFile;  
HANDLE hFileMap;  
DWORD FileSize;  
PVOID FileMem;

\#ifndef \_DEBUG  
if(argc!=2)  
{  
printf(“\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\\n”);  
printf(“Useage:\\nHivePase FileName\\n”);  
printf(“\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\\n”);  
system(“pause”);  
return 1;  
}  
hFile=CreateFile(argv\[1\],GENERIC\_READ,0,NULL,  
OPEN\_EXISTING,FILE\_FLAG\_OVERLAPPED,NULL);  
\#else  
hFile=CreateFile(“c:\\\\test\_root.dat”,GENERIC\_READ,0,NULL,  
OPEN\_EXISTING,FILE\_FLAG\_OVERLAPPED,NULL);  
\#endif

if(hFile==INVALID\_HANDLE\_VALUE)  
{  
printf(“Error:File doesn’s Exist!\\n”);  
system(“pause”);  
return 1;  
}  
FileSize=GetFileSize(hFile,NULL);  
hFileMap=CreateFileMapping(hFile,NULL,PAGE\_READONLY | SEC\_COMMIT,0,FileSize,NULL);  
if(hFileMap==NULL)  
{  
printf(“CreateFileMapping Error!\\n”);  
CloseHandle(hFile);  
system(“pause”);  
return 1;  
}  
CloseHandle(hFile);  
FileMem=MapViewOfFile(hFileMap,FILE\_MAP\_READ,0,0,0);  
if(FileMem==NULL)  
{  
printf(“MapViewOfFile Error!\\n”);  
CloseHandle(hFileMap);  
system(“pause”);  
return 1;  
}  
CloseHandle(hFileMap);  
TpyeKeyAndValue(FileMem);  
printf(“\\nSuccess!\\n”);  
system(“pause”);  
return 0;  
}

VOID TpyeKeyAndValue(PVOID FileMemAddr)  
{  
char RootKeyName\[256\];  
PCHAR RootBin;  
PCM\_KEY\_NODE KeyNode;  
PCM\_KEY\_INDEX KeyIndex;  
if(\*(ULONG\*)FileMemAddr!=REGF)  
{  
printf(“Not a Hive File!\\n”);  
system(“pause”);  
}  
RootBin=(char\*)FileMemAddr+0x1000;  
KeyNode=(PCM\_KEY\_NODE)(RootBin+0x24);  
if(\*(ULONG\*)RootBin!=HBIN)  
{  
printf(“Hive File Error!\\n”);  
system(“pause”);  
}  
ZeroMemory(RootKeyName,256);  
strncpy(RootKeyName,&amp;KeyNode-&gt;Name,KeyNode-&gt;NameLength);  
printf(“Root Key: %s\\n”,RootKeyName);  
DWORD SubKeyLists=KeyNode-&gt;SubKeyLists\[0\];  
KeyIndex=(PCM\_KEY\_INDEX)(RootBin+SubKeyLists+0x4);  
TypeSubKey(RootBin,KeyIndex);

}

VOID TypeSubKey(PCHAR Bin,PCM\_KEY\_INDEX KeyIndex)  
{  
USHORT KeyCount;  
PCM\_KEY\_NODE KeyNode;  
KeyCount=KeyIndex-&gt;Count;  
for(USHORT i=0;i&lt;KeyCount;i++)  
{  
if(KeyIndex-&gt;Signature==CM\_KEY\_FAST\_LEAF || KeyIndex-&gt;Signature==CM\_KEY\_HASH\_LEAF)  
{  
DWORD KeyNodeOffset=KeyIndex-&gt;List\[i\*2\];  
KeyNode=(PCM\_KEY\_NODE)(Bin+KeyNodeOffset+0x4);  
TypeSubKeyName(Bin,KeyNode);  
}  
else  
{  
DWORD KeyNodeOffset=KeyIndex-&gt;List\[i\*2\];  
KeyNode=(PCM\_KEY\_NODE)(Bin+KeyNodeOffset+0x4);  
TypeSubKeyName(Bin,KeyNode);  
}  
}

}

VOID TypeSubKeyName(PCHAR Bin,PCM\_KEY\_NODE KeyNode)  
{  
char SubKeyName\[256\];  
ULONG ValueLists;  
ULONG ValueCount;  
ULONG \*ValueIndex;  
PCM\_KEY\_VALUE value;  
PCM\_KEY\_INDEX KeyIndex;  
ZeroMemory(SubKeyName,256);  
strncpy(SubKeyName,&amp;KeyNode-&gt;Name,KeyNode-&gt;NameLength);  
printf(“Sub Key: %s\\n”,SubKeyName);  
ValueLists=KeyNode-&gt;ValueList.List;  
ValueCount=KeyNode-&gt;ValueList.Count;  
if(ValueLists!=-1)  
{  
ValueIndex=(ULONG \*)(Bin+ValueLists+0x4);  
for(ULONG i=0;i&lt;ValueCount;i++)  
{  
value=(PCM\_KEY\_VALUE)(Bin+ValueIndex\[i\]+0x4);  
TypeValue(Bin,value);  
}  
}  
if(KeyNode-&gt;SubKeyLists\[0\]!=-1)  
{  
KeyIndex=(PCM\_KEY\_INDEX)(Bin+KeyNode-&gt;SubKeyLists\[0\]+0x4);  
TypeSubKey(Bin,KeyIndex);  
}  
}

VOID TypeValue(PCHAR Bin,PCM\_KEY\_VALUE value)  
{  
char ValueName\[256\];  
ULONG DataLenth;  
PCHAR Data;  
ZeroMemory(ValueName,256);  
strncpy(ValueName,&amp;value-&gt;Name,value-&gt;NameLength);  
printf(“Value Name: %s\\n”,ValueName);  
switch(value-&gt;Type)  
{  
case REG\_SZ:  
printf(“REG\_SZ “);  
break;  
case REG\_BINARY:  
printf(“REG\_BINARY “);  
break;  
case REG\_DWORD:  
printf(“REG\_DWORD “);  
break;  
case REG\_MULTI\_SZ:  
printf(“REG\_MULIT\_SZ “);  
break;  
case REG\_EXPAND\_SZ:  
printf(“REG\_EXPAND\_SZ “);  
break;  
default:  
break;  
}  
if(value-&gt;Type==REG\_DWORD)  
{  
Data=(PCHAR)&amp;value-&gt;Data;  
printf(“%08x”,\*(ULONG\*)Data);  
}  
else  
{  
if(value-&gt;DataLength &amp; 0x80000000)  
{  
DataLenth=value-&gt;DataLength &amp; 0x7FFFFFFF;  
Data=(PCHAR)&amp;value-&gt;Data;  
for(ULONG i=0;i&lt;DataLenth;i++)  
{  
if(value-&gt;Type==REG\_BINARY)  
{  
printf(“%1x”,Data\[i\]);  
}  
else  
{  
printf(“%c”,Data\[i\]);  
}  
}  
}  
else  
{  
DataLenth=value-&gt;DataLength;  
DWORD DataOffset=value-&gt;Data;  
Data=Bin+DataOffset+0x4;  
for(ULONG i=0;i&lt;DataLenth;i++)  
{  
if(value-&gt;Type==REG\_BINARY)  
{  
printf(“%1x”,Data\[i\]);  
}  
else  
{  
printf(“%c”,Data\[i\]);  
}  
}  
}  
}  
printf(“\\n”);  
}

\[/c\]

关于上面code的一些补充说明:(by linxer)

<span style="color: #ff9900;">1.对nk节点下二级索引子键的情况没有处理(有ri节点情况,当子键过多的时候系统会使用ri节点的)</span>

<span style="color: #ff9900;">2.TypeValue里对值名称的处理有问题,问题出在这下面两行代码</span>

<span style="color: #ff9900;"> strncpy(ValueName,&amp;value-&gt;Name,value-&gt;NameLength);</span>

<span style="color: #ff9900;"> printf(“Value Name: %s\\n”,ValueName);</span>

<span style="color: #ff9900;">vk节点的值名称未必就是ascii字符串,因此用str系列函数是不可以的,正确用法应该是</span>

<span style="color: #ff9900;"> memcpy(ValueName,&amp;value-&gt;Name,value-&gt;NameLength); 因此显示语句也不能用printf来显示,目前我看到的很多代码都有这样的问题(chntpw),还有一些使用的hive解析的工具 (SnipeSword0225)也有这种问题</span>

<span style="color: #ff9900;"> </span>

<span style="color: #ff9900;">相关参考<span style="color: #ff0000;">:</span></span>

<span style="color: #ff9900;"><span style="color: #ff0000;">&lt;&lt;[windows注册表文件格式的简单分析](http://bbs.pediy.com/showthread.php?t=74337)&gt;&gt;</span></span>

<span style="color: #ff0000;"> &lt;&lt;[简单认识下注册表的HIVE文件格式](http://bbs.pediy.com/showthread.php?t=64000)&gt;&gt;</span>

<div class="mcePaste" id="_mcePaste" style="position: absolute; left: -10000px; top: 4332px; width: 1px; height: 1px; overflow: hidden;">\#include &lt;windows.h&gt;  
\#include &lt;stdio.h&gt;  
\#include &lt;stdlib.h&gt; \#define REGF 0x66676572 //fger  
\#define HBIN 0x6e696268 //nibh  
\#define CM\_KEY\_FAST\_LEAF 0x666c // fl  
\#define CM\_KEY\_HASH\_LEAF 0x686c // hl

//数据结构定义  
typedef struct \_CHILD\_LIST  
{  
ULONG Count;  
ULONG List;  
}CHILD\_LIST;

typedef struct \_CM\_KEY\_NODE  
{  
USHORT Signature;  
CHAR Reserve\_1\[18\];  
ULONG SubKeyCounts\[2\];  
ULONG SubKeyLists\[2\];  
CHILD\_LIST ValueList;  
CHAR Reserve\_2\[28\];  
USHORT NameLength;  
SHORT ClassName;  
CHAR Name;  
} CM\_KEY\_NODE,\*PCM\_KEY\_NODE;

typedef struct \_CM\_KEY\_INDEX  
{  
USHORT Signature;  
USHORT Count;  
ULONG List\[1\];  
} CM\_KEY\_INDEX, \*PCM\_KEY\_INDEX;

typedef struct \_CM\_KEY\_VALUE  
{  
USHORT Signature;  
SHORT NameLength;  
ULONG DataLength;  
ULONG Data;  
ULONG Type;  
CHAR Reserve\_1\[4\];  
CHAR Name;  
}CM\_KEY\_VALUE,\*PCM\_KEY\_VALUE;

VOID TpyeKeyAndValue(PVOID FileMemAddr);  
VOID TypeSubKey(PCHAR Bin,PCM\_KEY\_INDEX KeyIndex);  
VOID TypeSubKeyName(PCHAR Bin,PCM\_KEY\_NODE KeyNode);  
VOID TypeValue(PCHAR Bin,PCM\_KEY\_VALUE value);

int main(int argc,char \*argv\[\])  
{  
HANDLE hFile;  
HANDLE hFileMap;  
DWORD FileSize;  
PVOID FileMem;

\#ifndef \_DEBUG  
if(argc!=2)  
{  
printf(“\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\\n”);  
printf(“Useage:\\nHivePase FileName\\n”);  
printf(“\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\\n”);  
system(“pause”);  
return 1;  
}  
hFile=CreateFile(argv\[1\],GENERIC\_READ,0,NULL,  
OPEN\_EXISTING,FILE\_FLAG\_OVERLAPPED,NULL);  
\#else  
hFile=CreateFile(“c:\\\\test\_root.dat”,GENERIC\_READ,0,NULL,  
OPEN\_EXISTING,FILE\_FLAG\_OVERLAPPED,NULL);  
\#endif

if(hFile==INVALID\_HANDLE\_VALUE)  
{  
printf(“Error:File doesn’s Exist!\\n”);  
system(“pause”);  
return 1;  
}  
FileSize=GetFileSize(hFile,NULL);  
hFileMap=CreateFileMapping(hFile,NULL,PAGE\_READONLY | SEC\_COMMIT,0,FileSize,NULL);  
if(hFileMap==NULL)  
{  
printf(“CreateFileMapping Error!\\n”);  
CloseHandle(hFile);  
system(“pause”);  
return 1;  
}  
CloseHandle(hFile);  
FileMem=MapViewOfFile(hFileMap,FILE\_MAP\_READ,0,0,0);  
if(FileMem==NULL)  
{  
printf(“MapViewOfFile Error!\\n”);  
CloseHandle(hFileMap);  
system(“pause”);  
return 1;  
}  
CloseHandle(hFileMap);  
TpyeKeyAndValue(FileMem);  
printf(“\\nSuccess!\\n”);  
system(“pause”);  
return 0;  
}

VOID TpyeKeyAndValue(PVOID FileMemAddr)  
{  
char RootKeyName\[256\];  
PCHAR RootBin;  
PCM\_KEY\_NODE KeyNode;  
PCM\_KEY\_INDEX KeyIndex;  
if(\*(ULONG\*)FileMemAddr!=REGF)  
{  
printf(“Not a Hive File!\\n”);  
system(“pause”);  
}  
RootBin=(char\*)FileMemAddr+0x1000;  
KeyNode=(PCM\_KEY\_NODE)(RootBin+0x24);  
if(\*(ULONG\*)RootBin!=HBIN)  
{  
printf(“Hive File Error!\\n”);  
system(“pause”);  
}  
ZeroMemory(RootKeyName,256);  
memcpy(RootKeyName,&amp;KeyNode-&gt;Name,KeyNode-&gt;NameLength);  
printf(“Root Key: %s\\n”,RootKeyName);  
DWORD SubKeyLists=KeyNode-&gt;SubKeyLists\[0\];  
KeyIndex=(PCM\_KEY\_INDEX)(RootBin+SubKeyLists+0x4);  
TypeSubKey(RootBin,KeyIndex);

}

VOID TypeSubKey(PCHAR Bin,PCM\_KEY\_INDEX KeyIndex)  
{  
USHORT KeyCount;  
PCM\_KEY\_NODE KeyNode;  
KeyCount=KeyIndex-&gt;Count;  
for(USHORT i=0;i&lt;KeyCount;i++)  
{  
if(KeyIndex-&gt;Signature==CM\_KEY\_FAST\_LEAF || KeyIndex-&gt;Signature==CM\_KEY\_HASH\_LEAF)  
{  
DWORD KeyNodeOffset=KeyIndex-&gt;List\[i\*2\];  
KeyNode=(PCM\_KEY\_NODE)(Bin+KeyNodeOffset+0x4);  
TypeSubKeyName(Bin,KeyNode);  
}  
else  
{  
DWORD KeyNodeOffset=KeyIndex-&gt;List\[i\*2\];  
KeyNode=(PCM\_KEY\_NODE)(Bin+KeyNodeOffset+0x4);  
TypeSubKeyName(Bin,KeyNode);  
}  
}

}

VOID TypeSubKeyName(PCHAR Bin,PCM\_KEY\_NODE KeyNode)  
{  
char SubKeyName\[256\];  
ULONG ValueLists;  
ULONG ValueCount;  
ULONG \*ValueIndex;  
PCM\_KEY\_VALUE value;  
PCM\_KEY\_INDEX KeyIndex;  
ZeroMemory(SubKeyName,256);  
strncpy(SubKeyName,&amp;KeyNode-&gt;Name,KeyNode-&gt;NameLength);  
printf(“Sub Key: %s\\n”,SubKeyName);  
ValueLists=KeyNode-&gt;ValueList.List;  
ValueCount=KeyNode-&gt;ValueList.Count;  
if(ValueLists!=-1)  
{  
ValueIndex=(ULONG \*)(Bin+ValueLists+0x4);  
for(ULONG i=0;i&lt;ValueCount;i++)  
{  
value=(PCM\_KEY\_VALUE)(Bin+ValueIndex\[i\]+0x4);  
TypeValue(Bin,value);  
}  
}  
if(KeyNode-&gt;SubKeyLists\[0\]!=-1)  
{  
KeyIndex=(PCM\_KEY\_INDEX)(Bin+KeyNode-&gt;SubKeyLists\[0\]+0x4);  
TypeSubKey(Bin,KeyIndex);  
}  
}

VOID TypeValue(PCHAR Bin,PCM\_KEY\_VALUE value)  
{  
char ValueName\[256\];  
ULONG DataLenth;  
PCHAR Data;  
ZeroMemory(ValueName,256);  
strncpy(ValueName,&amp;value-&gt;Name,value-&gt;NameLength);  
printf(“Value Name: %s\\n”,ValueName);  
switch(value-&gt;Type)  
{  
case REG\_SZ:  
printf(“REG\_SZ “);  
break;  
case REG\_BINARY:  
printf(“REG\_BINARY “);  
break;  
case REG\_DWORD:  
printf(“REG\_DWORD “);  
break;  
case REG\_MULTI\_SZ:  
printf(“REG\_MULIT\_SZ “);  
break;  
case REG\_EXPAND\_SZ:  
printf(“REG\_EXPAND\_SZ “);  
break;  
default:  
break;  
}  
if(value-&gt;Type==REG\_DWORD)  
{  
Data=(PCHAR)&amp;value-&gt;Data;  
printf(“%08x”,\*(ULONG\*)Data);  
}  
else  
{  
if(value-&gt;DataLength &amp; 0x80000000)  
{  
DataLenth=value-&gt;DataLength &amp; 0x7FFFFFFF;  
Data=(PCHAR)&amp;value-&gt;Data;  
for(ULONG i=0;i&lt;DataLenth;i++)  
{  
if(value-&gt;Type==REG\_BINARY)  
{  
printf(“%1x”,Data\[i\]);  
}  
else  
{  
printf(“%c”,Data\[i\]);  
}  
}  
}  
else  
{  
DataLenth=value-&gt;DataLength;  
DWORD DataOffset=value-&gt;Data;  
Data=Bin+DataOffset+0x4;  
for(ULONG i=0;i&lt;DataLenth;i++)  
{  
if(value-&gt;Type==REG\_BINARY)  
{  
printf(“%1x”,Data\[i\]);  
}  
else  
{  
printf(“%c”,Data\[i\]);  
}  
}  
}  
}  
printf(“\\n”);  
}

</div>