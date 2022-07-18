###预编译（预处理include、define）

####C语音的执行流程

C语言执行的流程：

1. 编译：形成目标代码(.obj)。
2. 连接：将目标代码与C函数库连接合并，形成最终的可执行文件。
3. 执行。

预编译（预处理），为编译做准备工作，完成代码文本的替换工作。

头文件告诉编译器有这样一个函数，连接器负责找到这个函数的实现，通过include引入。实现的话，在哪里都可以。类似于Android布局文件中的include标签。

一个简单的例子：

创建text.txt文件：

	printf("我被包含进来了");

在主函数里面使用：

	#include<stdio.h>
	#include<stdlib.h>
	
	void main(){
		
		#include "text.txt"
	
		system("pause");
	}

实质上会把include标签替换成我们自己的text.txt文件里面的内容。

######VS源码的目录：C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\crt\src

####宏定义、宏替换

作用：

1. 定义标识。
2. 定义常数（便于修改与阅读）。
3. 定义“宏函数”。

#####1、定义标识

作用：

1、例如通过判断一些标识是否定义来判断是否支持某种语法、平台等等：

	//表示支持C++语法
	#ifdef __cplusplus
	
	#endif // __cplusplus

	//表示支持Android、Windows、苹果平台等等
	#ifdef ANDROID
	
	#endif // ANDROID

2、防止问价你重复引入：

举个例子，我们有三个文件a.h、b.h、Test.cpp，分别如下：

这是a.h：

	#include "b.h"
	
	void a();

这是b.h：

	#include "a.h"
	
	void b();

最后Test.cpp里面引用了a.h

	#include "a.h"

这样，当Test包含a的时候，a又会去包含b，b又会包含a，这样就会造成循环包含。类似于Hibernate里面的SQL循环引用。最终会报如下错误：

	fatal error C1014: 包含文件太多 : 深度 = 1024

通过宏定义判断就可以解决这个问题：（b.h省略）

	#if A_H
	
	#include "b.h"
	
	void a();
	
	#endif

另外，新版本的时候通过#pragma once语句即可自动解决这个问题。

	//该头文件只被包含一次，编译器自动处理循环包含问题
	#pragma once


#####2、定义常熟，方便阅读

一个简单的例子：

	#define MAX 100
	
	void main(){
		
		int i = 100;
		if (i == MAX){
			printf("哈哈");
		}
	
		system("pause");
	}


#####3、定义“宏函数”。

实质上就是一个替换的过程。

简单实用例子：

	#define LOG(FORMAT , ...) printf("info"); printf(##FORMAT , __VA_ARGS__);

LOG会有级别，于是进一步升级：

	#define LOG_I(FORMAT , ...) printf("info"); printf(##FORMAT , __VA_ARGS__);
	#define LOG_E(FORMAT , ...) printf("error"); printf(##FORMAT , __VA_ARGS__);

进一步简化重复代码，重复LEVEL日志级别：

	#define LOG(LEVEL , FORMAT , ...) printf(##LEVEL); printf(##FORMAT , __VA_ARGS__);
	#define LOG_I(FORMAT , ...) LOG("info" , ##FORMAT , __VA_ARGS__)
	#define LOG_E(FORMAT , ...) LOG("error" , ##FORMAT , __VA_ARGS__)

在Android JNI开发的时候，我们打印一句日志是通过__android_log_print函数来实现的，因此我们可以通过宏定义简化代码：

	#define LOGI(FORMAT,...) __android_log_print(ANDROID_LOG_INFO,"jason",FORMAT,##__VA_ARGS__);
	LOGI("%s","fix");
	//替换
	__android_log_print(ANDROID_LOG_INFO, "jason", "%s", "fix");

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
