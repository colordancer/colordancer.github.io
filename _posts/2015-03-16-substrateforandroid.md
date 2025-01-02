---
id: 1914
title: 'Android Cydia Substrate hook用法'
date: '2015-03-16T18:58:29+08:00'
author: colordancer
layout: post
guid: 'http://www.colordancer.net/blog/?p=1914'
permalink: /2015/03/16/substrateforandroid/
views:
    - '2884'
categories:
    - 安全研究
    - 技术
tags:
    - android
    - Cydia
    - hook
    - Substrate
---

 前一段时间给Saurik发了几封邮件，想让Substrate for Android开源，以更好地支持Android 5.0，可惜Saurik一直没回我。Substrate文档较少，又没有开源社区支持，再加上Frida出来了，估计以后用Substrate的会更少。这里分享一下Substrate的各种方法，供参考。

 Substrate hook步骤:

 1. 在Manifest里进行权限的声明

```

<permission
        android:name="cydia.permission.SUBSTRATE"
        android:label="modify code from other packages"
        android:permissionGroup="android.permission-group.DEVELOPMENT_TOOLS"
        android:protectionLevel="dangerous" />
<uses-permission android:name="cydia.permission.SUBSTRATE" />
```

 2. Java hook:

 (1) 声明meta data, 指向Java hook的入口类:

```

<meta-data android:name="com.saurik.substrate.main" android:value=".Main" />
```

 (2) 在Main类里实现static void initialize() {}函数, 在模块安装后, Cydia会自动调用initialize函数. 可以在这里进行hook的编写, 当然也可以手动调用hook函数. 如修该系统自体:

```

static void hookColor() {
		MS.hookClassLoad("android.content.res.Resources",
				new MS.ClassLoadHook() {
					public void classLoaded(Class<?> resources) {
						Method getColor;
						try {
							getColor = resources.getMethod("getColor",
									Integer.TYPE);
						} catch (NoSuchMethodException e) {
							getColor = null;
						}
						if (getColor != null) {
							final MS.MethodPointer old = new MS.MethodPointer();
							MS.hookMethod(resources, getColor,
									new MS.MethodHook() {
										public Object invoked(Object resources,
												Object... args)
												throws Throwable {
											int color = (Integer) old.invoke(
													resources, args);
											return color & ~0x0000ff00
													| 0x00ff0000;
										}
									}, old);
						}
					}
				});
	}
```

<div> <span style="white-space: nowrap;">3. C hook: </span>

 (1) 把libsubstrate.so和libsubstrate-dvm.so拷贝到工程JNI目录. 前者是c函数hook依赖库,后者是dvm函数hook依赖库. 在Android. mk里加入以下代码, 以把依赖库编译模块安装包里.

```

include $(CLEAR_VARS)
LOCAL_MODULE:= substrate
LOCAL_SRC_FILES := libsubstrate.so
include $(PREBUILT_SHARED_LIBRARY)
```

 在hook模块的编译脚本里链接libsubstrate.so:

```

LOCAL_LDLIBS += -L$(LOCAL_PATH) -lsubstrate
```

 <span style="white-space: nowrap;">(2) 初始化设置:</span>

 <span style="white-space: nowrap;">a. 定义要hook的模块, </span><span style="white-space: nowrap;">如果是so</span>

```

MSConfig(MSFilterLibrary, "/system/lib/libdvm.so")
```

<div> 如果是exe

```

MSConfig(MSFilterExecutable, "/system/bin/app_process")
```

 <span>b. 初始化入口:</span>

```

MSInitialize
{

//TODO:模块安装后, Cydia会自动调用这里的代码,你可以在这里进行hook,也可以以后手动hook

hook();

}
```

 <span>(3) 进行hook, 以hook libdvm.so里的dvmCallMethodV为例:</span>

```

void hook()
{
	MSImageRef image = MSGetImageByName("/system/lib/libdvm.so");
	if (image != NULL)
	{
		void * symbole = MSFindSymbol(image,
				"_Z14dvmCallMethodVP6ThreadPK6MethodP6ObjectbP6JValueSt9__va_list");
		if (symbole == NULL)
		{
			LOGE("error find _Z21dvmDexFileOpenPartialPKviPP6DvmDex ");
		}
		else
		{
			MSHookFunction(symbole, (void*) &MydvmCallMethodV,
					(void **) &OriDvmCallMethodV);
			LOGD("hook dvmCallMethodV ok");
		}
	}
	else {
		LOGE("error find libdvm");
	}
}
```

 <span style="white-space: nowrap;">4. JNI hook</span>

 <span style="white-space: nowrap;">与C hook基本类似, 主要区别:</span>

 <span style="white-space: nowrap;">(1) 把C函数hook链接的so改为libsubstrate-dvm.so</span>

 <span style="white-space: nowrap;">(2) 在C代码里进行hook:</span>

```

MSConfig(MSFilterExecutable, "/system/bin/app_process")
static jint (*_Resources$getColor)(JNIEnv *jni, jobject _this, ...);
static jint $Resources$getColor(JNIEnv *jni, jobject _this, jint rid) {
    jint color = _Resources$getColor(jni, _this, rid);
    return color & ~0x0000ff00 | 0x00ff0000;
}
static void OnResources(JNIEnv *jni, jclass resources, void *data) {
    jmethodID method = jni->GetMethodID(resources, "getColor", "(I)I");
    if (method != NULL)
        MSJavaHookMethod(jni, resources, method,
            &$Resources$getColor, &_Resources$getColor);
}
MSInitialize {
    MSJavaHookClassLoad(NULL, "android/content/res/Resources", &OnResources);
}
```


</div>
</div>