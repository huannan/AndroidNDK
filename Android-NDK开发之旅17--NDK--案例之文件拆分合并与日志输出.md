###前言

这里再次啰嗦一下，我们为什么要学习NDK开发呢？因为很多大公司，为了节省开发资源，很多时候核心技术都是用C/C++去实现的，一套代码，可以给Android、IOS、后端调用，这也是一种跨平台的实现方案，大大节省了人力成本，但是对开发者的要求就提高了。这也是为什么像QQ、淘宝等众多大型APP都是采用了很多动态库文件的原因。所以说很多时候，大公司的笔试考的都是C/C++。

如果你也有志气想进大公司，那就学习吧，勇往直前！

###文件案例与拆分

先说说为什么要做文件拆分和合并。举一些例子，我们做多媒体开发的时候，尤其是大视频上传，我们就需要把视频拆分成多个分别上传，减轻服务端压力的同时，也防止了上传失败导致全盘重新上传。而且这次的文章是为了下一次的增量更新做铺垫。

上一篇博客介绍了Android Studio2.3的NDK开发流程，下面我们通过一个实际案例来进一步加强、熟悉这个开发流程。

首先我们创建项目，配置我们的CMake构建脚本：

	cmake_minimum_required(VERSION 3.4.1)
	
	add_library(
	             testJni
	             SHARED
	             src/main/jni/testJni.c )
	
	find_library( log-lib
	              log )
	
	target_link_libraries(
	                       testJni
	                       ${log-lib} )

由于我们在NDK中不能够使用printf输出logcat，所以将会使用NDK自带的log-lib来进行日志打印。

其中：

1. find_library命令是找到log-lib这个库。
2. target_link_libraries是把我们自己的库跟log-lib关联起来。相当于我们的库依赖log-lib库。
	
如果你使用ndk-build的方式来构建项目的话，那么请在Android.mk文件中添加LOCAL_LDLIBS配置，完整的配置如下：

	LOCAL_PATH := $(call my-dir)
	
	include $(CLEAR_VARS)
	
	LOCAL_MODULE    := testJni
	LOCAL_SRC_FILES := testJni.c
	LOCAL_LDLIBS := -llog
	include $(BUILD_SHARED_LIBRARY)


其它的就不赘述了，参考上一篇文章，下面我们创建一个类，专门用于文件的拆分与合并：

	package com.nan.testjni;
	
	public class FileUtil {
	    //文件拆分
	    public static native void diff(String path, String pattern, int count);
	
	    //文件合并
	    public static native void patch(String path, String pattern, int count);
	}

下面我们在Application里面加载我们的动态库，因为Application是全局的，一般来说Application只会初始化一次（单进程的情况下）。
	
	public class App extends Application {
	
	    static {
	        //加载动态库.so文件，注意不用写lib前缀，系统会默认添加
	        System.loadLibrary("testJni");
	    }
	
	    @Override
	    public void onCreate() {
	        super.onCreate();
	
	    }
	}

记得在清单文件中配置一下：

    <application
        android:name=".App"
    </application>

###文件拆分

先举个例子，比如说我们的文件大小是110个单位（字节），需要被分成9个小文件：

1. 如果110能够被9整除的话，那么每个文件大小是：110/9
2. 很显然，这里不能够整除，那么我们前面8（9-1）个文件需要大一些，大小为：110/（9-1）。最后一个文件的大小稍微小一些，大小为：110%（9-1）。

示例代码如下：

	JNIEXPORT void JNICALL
	Java_com_nan_testjni_FileUtil_diff(JNIEnv *env, jclass type, jstring path_, jstring pattern_,
	                                   jint count) {
	
	    //需要分割的文件路径
	    const char *path = (*env)->GetStringUTFChars(env, path_, NULL);
	    //分割之后的小文件命名法则
	    const char *pattern = (*env)->GetStringUTFChars(env, pattern_, NULL);
	
	    //得到分割之后的小文件的命名列表（字符串数组），分配内存
	    char **patches = malloc(sizeof(char *) * count);
	    int i = 0;
	    for (; i < count; i++) {
	        //为字符数组的每一个条目分配内存
	        patches[i] = malloc(sizeof(char) * 100);
	        sprintf(patches[i], pattern, i);
	
	        //打印日志
	        //__android_log_print(ANDROID_LOG_DEBUG, "TAG_NDK", "File Name : %s\n", patches[i]);
	        LOGE("name:%s\n", patches[i]);
	    }
	
	    FILE *fTotal = fopen(path, "rb");
	    int fileSize = getFileSize(path);
	
	    if (fileSize / count == 0) {
	        //可以整除
	        int partSize = fileSize / count;
	        i = 0;
	        for (; i < count; i++) {
	            FILE *fPart = fopen(patches[i], "wb");
	            int j = 0;
	            for (; j < partSize; j++) {
	                fputc(fgetc(fTotal), fPart);
	            }
	            fclose(fPart);
	        }
	    } else {
	        //不可以整除
	        int partSize = fileSize / (count - 1);
	        i = 0;
	        for (; i < count - 1; i++) {
	            FILE *fPart = fopen(patches[i], "wb");
	            int j = 0;
	            for (; j < partSize; j++) {
	                fputc(fgetc(fTotal), fPart);
	            }
	            fclose(fPart);
	        }
	        FILE *fLast = fopen(patches[count - 1], "wb");
	        i = 0;
	        for (; i < fileSize % (count - 1); i++) {
	            fputc(fgetc(fTotal), fLast);
	        }
	        fclose(fLast);
	    }
	
	    //释放资源
	    fclose(fTotal);
	    i = 0;
	    for (; i < count; i++) {
	        free(patches[i]);
	    }
	    free(patches);
	
	    (*env)->ReleaseStringUTFChars(env, path_, path);
	    (*env)->ReleaseStringUTFChars(env, pattern_, pattern);
	}


解释一下上面的代码：

1. 首先我们需要一个分割之后的字符串数组（也就是char二级指针），用于存放分割之后的文件路径。
2. 然后我分整除和不整除两种情况来分别处理，不断读取被分割的文件，循环写入每个子文件中。
3. 释放资源，尤其注意的是，malloc分配的堆内存需要及时释放。

这里我们封装了一个获取文件大小的函数，之前的C语音的文件有介绍，不在赘述：

	//获取文件大小
	long get_file_size(char *path){
		FILE *fp = fopen(path,"rb");
		fseek(fp,0,SEEK_END);
		return ftell(fp);
	}

###NDK日志输出

在上面配置了我们的CMake构建脚本之后（否则找不到打印日志函数），我们可以使用__android_log_print来输出日志，参数分别为日志级别、TAG，剩下的两个参数跟printf一样。使用的例子如下：

先通过include包含文件：

	#include <android/log.h>

然后调用__android_log_print输出日志：

	__android_log_print(ANDROID_LOG_DEBUG, "TAG_NDK", "File Name : %s\n", patches[i]);

这里为了简化，我们通过一些宏定义来简化操作，在C语音的预编译的那篇文章有介绍，这里不再赘述：

	#define LOGI(FORMAT, ...) __android_log_print(ANDROID_LOG_INFO, "huannan",FORMAT,__VA_ARGS__)
	#define LOGD(FORMAT, ...) __android_log_print(ANDROID_LOG_DEBUG,"huannan",FORMAT,__VA_ARGS__)
	#define LOGE(FORMAT, ...) __android_log_print(ANDROID_LOG_ERROR,"huannan",FORMAT,__VA_ARGS__)


###文件合并

下面来看看文件合并的代码，相对来说比较简单，就是通过一个循环，不断读取每个子文件，然后合并到总文件中。代码如下：

	JNIEXPORT void JNICALL
	Java_com_nan_testjni_FileUtil_patch(JNIEnv *env, jclass type, jstring path_, jstring pattern_,
	                                    jint count) {
	    //合并之后的文件路径
	    const char *path = (*env)->GetStringUTFChars(env, path_, NULL);
	    //分割之后的小文件命名法则
	    const char *pattern = (*env)->GetStringUTFChars(env, pattern_, NULL);
	
	    //得到被分割的小文件的命名列表（字符串数组），分配内存
	    char **patches = malloc(sizeof(char *) * count);
	    int i = 0;
	    for (; i < count; i++) {
	        //为字符数组的每一个条目分配内存
	        patches[i] = malloc(sizeof(char) * 100);
	        sprintf(patches[i], pattern, i);
	
	        //打印日志
	        //__android_log_print(ANDROID_LOG_DEBUG, "TAG_NDK", "File Name : %s\n", patches[i]);
	        LOGE("name:%s\n", patches[i]);
	    }
	
	    //合并的文件路径
	    FILE *fTotal = fopen(path, "wb");
	    i = 0;
	    for (; i < count; i++) {
	        int patchSize = getFileSize(patches[i]);
	        FILE *fPart = fopen(patches[i], "rb");
	        int j = 0;
	        for (; j < patchSize; j++) {
	            fputc(fgetc(fPart), fTotal);
	        }
	        fclose(fPart);
	    }
	
	    //释放资源
	    fclose(fTotal);
	    i = 0;
	    for (; i < count; i++) {
	        free(patches[i]);
	    }
	    free(patches);
	
	    (*env)->ReleaseStringUTFChars(env, path_, path);
	    (*env)->ReleaseStringUTFChars(env, pattern_, pattern);
	}

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
