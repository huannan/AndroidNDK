###Eclipse下NDK开发流程

先来个Eclipse版本的，后续再上AS版本的。工具的使用不同而已。

###具体流程

1. 编写Java层Native方法
2. javah命令生成头文件
3. 创建jni目录
4. 配置NDK路径，添加本地支持add native support，配置ADT需要包含的头文件
5. 实现头文件中定义的函数
6. 编译生成.so动态库
7. 加载动态库

下面以创建一个简单的文件加密解决APP来作为示范：

####一、编写Java层Native方法

我们专门创建一个Cryptor类用于文件的加密解密：

	public class Cryptor {
	
		public static native void cryptFile(String src , String dest);
	
		public static native void decryptFile(String src , String dest);
	
	}

这里写两个native方法，分别用于文件加密解密。

####二、使用javah命令生成对应的头文件

在命令行下，通过cd命令转到工程的src目录（代码目录），然后执行以下命令：

	javah 完整类名

就会生成对应的头文件。

####三、创建jni目录

直接通过Eclipse来创建即可，然后把我们刚刚生成的头文件放进去。

####四、配置NDK路径，添加本地支持add native support，配置ADT需要包含的头文件

没有配置NDK路径的需要先配置一下：

Windows-->Preference-->Android-->NDK：

![配置NDK路径.png](http://upload-images.jianshu.io/upload_images/2570030-35c5d25d96c8ab31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

添加native support，右击项目，选择Android Tools，选择add native support，然后弹出下面的界面，输入lib的名字，前缀系统默认为lib。

![添加native支持.png](http://upload-images.jianshu.io/upload_images/2570030-cd3ad17575aac03d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后会在jni目录自动生成一些文件：

![自动生成的文件.png](http://upload-images.jianshu.io/upload_images/2570030-b25e5e83e8fb24fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中Application.mk是我们自己添加的，用于指定输出什么架构的so动态库文件，Application.mk如下：

	# armeabi armeabi-v7a arm64-v8a mips mips64 x86 x86_64
	APP_ABI := armeabi

其中第一行是注释。

配置ADT中需要包含的NDK头文件：右击工程-->属性-->C/C++常规-->路径和符号，添加NDK的以下目录：

	G:\android-ndk-r10\toolchains\arm-linux-androideabi-4.9\prebuilt\windows-x86_64\lib\gcc\arm-linux-androideabi\4.9\include
	G:\android-ndk-r10\toolchains\arm-linux-androideabi-4.9\prebuilt\windows-x86_64\lib\gcc\arm-linux-androideabi\4.9\include-fixed
	G:\android-ndk-r10\platforms\android-L\arch-arm\usr\include

######注意：笔者的版本可能跟你的版本不一样，不过大同小异

配置好的图如下：（某些目录是后来生成的）

![配置包含目录.png](http://upload-images.jianshu.io/upload_images/2570030-5e564894abcffc4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####五、实现头文件中定义的函数

我们在crypt.c中实现吧，本来是cpp文件的，我们目前先改为c实现：

	#include "com_example_ndk_file_crypt_utils_Cryptor.h"
	#include <stdlib.h>
	#include <stdio.h>
	#include <string.h>
	
	char password[] = "I AM MI MA";
	
	JNIEXPORT void JNICALL Java_com_example_ndk_1file_1crypt_utils_Cryptor_cryptFile(
			JNIEnv * env, jclass jclz, jstring src, jstring dest) {
	
		const char* c_src = (*env)->GetStringUTFChars(env, src, NULL);
		const char* c_dest = (*env)->GetStringUTFChars(env, dest, NULL);
	
		FILE* f_read = fopen(c_src, "rb");
		FILE* f_write = fopen(c_dest, "wb");
	
		//判断文件是否正确打开
		if (f_read == NULL || f_write == NULL ) {
			printf("file open field");
			return;
		}
	
		//一次读取一个字符
		int ch;
		int i = 0;
		int pwd_len = strlen(password);
		while ((ch = fgetc(f_read)) != EOF) {
			//通过异或运算进行加密
			fputc(ch ^ password[i % pwd_len], f_write);
			i++;
		}
	
		//关闭文件
		fclose(f_read);
		fclose(f_write);
	
	}
	
	JNIEXPORT void JNICALL Java_com_example_ndk_1file_1crypt_utils_Cryptor_decryptFile(
			JNIEnv * env, jclass jclz, jstring src, jstring dest) {
	
		const char* c_src = (*env)->GetStringUTFChars(env, src, NULL);
		const char* c_dest = (*env)->GetStringUTFChars(env, dest, NULL);
	
		FILE* f_read = fopen(c_src, "rb");
		FILE* f_write = fopen(c_dest, "wb");
	
		//判断文件是否正确打开
		if (f_read == NULL || f_write == NULL ) {
			printf("file open field");
			return;
		}
	
		//一次读取一个字符
		int ch;
		int i = 0;
		int pwd_len = strlen(password);
		while ((ch = fgetc(f_read)) != EOF) {
			//通过异或运算进行加密
			fputc(ch ^ password[i % pwd_len], f_write);
			i++;
		}
	
		//关闭文件
		fclose(f_read);
		fclose(f_write);
	
	}


然后记得配置Android.mk文件，把LOCAL_SRC_FILES的值改为crypt.c（原来是cpp）。

	LOCAL_PATH := $(call my-dir)
	
	include $(CLEAR_VARS)
	
	LOCAL_MODULE    := crypt
	LOCAL_SRC_FILES := crypt.c
	
	include $(BUILD_SHARED_LIBRARY)


####六、编译生成.so动态库

直接make project一下即可。

####七、加载动态库

通过System.loadLibrary方法加载动态库.so文件，注意不用写lib前缀，系统会默认添加。

	public class Cryptor {
	
		static {
			//加载动态库.so文件，注意不用写lib前缀，系统会默认添加
			System.loadLibrary("crypt");
		}
	
		public static native void cryptFile(String src , String dest);
	
		public static native void decryptFile(String src , String dest);
	
	}

剩下的就在Android布局中写两个按钮，测试一下即可，记得加上SD卡访问权限：

	<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/>

最后，说明一下，如果要适配所有平台，需要修改Application.mk为：

	APP_ABI := armeabi armeabi-v7a arm64-v8a mips mips64 x86 x86_64

由于生成的过程需要交叉编译，速度非常慢，大概五分钟左右，因此，如果c代码没有改变的话，我们可以通过下面的配置，不编译c代码：

![不编译so文件.png](http://upload-images.jianshu.io/upload_images/2570030-6d50ad858160c595.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

把两个勾去掉即可。
