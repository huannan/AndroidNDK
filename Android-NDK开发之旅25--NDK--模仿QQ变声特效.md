###前言

我们这次用到的是fmod这个库，fmod是音效引擎游戏开发革命引擎，著名的游戏开发引擎CosCos2D、U3D都封装了这个库。

学习NDK的目的就是为了让我们的APP能够使用C/C++开源了那么多年的优秀库。例如我们Android本身就已经把OpenGL、SQLite等优秀的C/C++开源库打包进Android平台里面了，我们只需要使用上层的Java native接口就可以很方便地去使用这些功能。

###NDK中使用开源库fmod的流程

在这个流程中我们主要是要学会怎么去引入一个完全没有用过的C/C++库，而不是这个库的使用本身。

####创建native support的APP项目

打开Android Studio，创建native support的工程。

###拷贝资源

到fmod的官网进行注册，然后下载官方的Android版本的API。

![下载的Android版本的Fmod库.png](http://upload-images.jianshu.io/upload_images/2570030-0d2483c840aa1a4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解压压缩包以后，打开api文件夹，会有fsbank、lowlevel、studio：

![api文件夹.png](http://upload-images.jianshu.io/upload_images/2570030-2d8bf40cfbd38e8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. fsbank，bank是一个fmod的概念，包含里一些声音的事件，也可以通过fmod studio来制作，这里是通过代码去制作。
2. lowlevel，我们需要的文件都在这里，包含了基础的声音处理功能。
3. studio，这里存放的api跟界面相关的，我们不需要使用这些界面。

我们需要的资源在fmodstudioapi10903android\api\lowlevel里面，其中有三个文件夹：

![lowlevel.png](http://upload-images.jianshu.io/upload_images/2570030-ce7555eb0eb8d862.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. examples示例程序，里面有一个Activity，以及对应的cpp文件，我们可以参考。
2. 需要的inc目录，这个我们需要直接拷贝到我们的Android项目当中。
3. lib目录里面有一些预编译好的so库，以及一个jar包，都需要拷贝到我们的项目当中。

然后创建cpp目录，把fmod需要的头文件、源文件添加进来。（需要哪些就拷贝哪些，这其中需要分析fmod的头文件，仔细阅读）

fmod提供了一些预编译的so库、以及jar包，因此我们需要拷贝到libs目录当中，然后记得在app的build.gradle脚本中添加：

    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }

###修改CMakeLists.txt文件

在这个文件中，我们除了添加自己的库以外，还需要添加fmod的预编译库，并且把它们链接到我们自己的库里面来。

	#配置CMake的最低版本
	cmake_minimum_required(VERSION 3.4.1)
	
	#配置工程路径
	#set(path_project /home/wuhuannan/AndroidPrj/VoiceChange)
	set(path_project D:/AndroidStudio/VoiceChange)
	
	#添加自己写的库
	add_library(effect-lib
	            SHARED
	            src/main/cpp/effect_fix.cpp
	            )
	
	#添加两个预编译的so库
	add_library( fmod
	             SHARED
	             IMPORTED
	            )
	
	add_library( libfmodL
	             SHARED
	             IMPORTED
	            )
	
	#设置两个预编译的库的路径，注意这里最好要使用绝对路径，不然会链接错误
	set_target_properties(
	                       fmod
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libfmod.so
	                    )
	
	set_target_properties(
	                       libfmodL
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libfmodL.so
	                    )
	
	#找到Android的log库（这个库已经在Android平台中了）
	find_library(
	            log-lib
	            log
	            )
	
	#把需要的库链接到自己的库中
	target_link_libraries(
	            effect-lib
	            ${log-lib}
	            fmod
	            libfmodL
	            )

其中需要注意的是跟ndk-build的区别：

ndk-build的写法如下：

Android.mk文件：

	LOCAL_PATH := $(call my-dir)
	
	##添加两个预编译的库
	include $(CLEAR_VARS)
	LOCAL_MODULE := fmod
	LOCAL_SRC_FILES := libfmod.so
	include $(PREBUILT_SHARED_LIBRARY)
	
	include $(CLEAR_VARS)
	LOCAL_MODULE := fmodL
	LOCAL_SRC_FILES := libfmodL.so
	include $(PREBUILT_SHARED_LIBRARY)
	
	include $(CLEAR_VARS)
	
	##添加自己的库，并且链接起来
	LOCAL_MODULE    := effect-lib
	LOCAL_SRC_FILES := effect_fix.cpp
	LOCAL_SHARED_LIBRARIES := fmod fmodL
	LOCAL_LDLIBS := -llog
	##添加异常处理功能（CMake是默认支持的）
	LOCAL_CPP_FEATURES := exceptions
	include $(BUILD_SHARED_LIBRARY)

Application.mk文件：

	##支持C++异常处理，标准莫板块（STL）
	APP_STL := gnustl_static

###同步项目

在同步的过程中，我们可能会遇到很多问题，例如：

1. 没有把所需要的头文件添加进来。
2. include头文件的时候路径是否正确。 
3. 预编译的so库的路径是否正确。
4. 如果是ndk-build的方式，检查是否支持了异常处理、STL等等。

###创建Java native方法

	package com.nan.voicechange;
	
	public class VoiceEffectUtils {
	
	    public static final int TYPE_NORMAL = 0;
	    public static final int TYPE_LUOLI = 1;
	    public static final int TYPE_DASHU = 2;
	    public static final int TYPE_JINGSONG = 3;
	    public static final int TYPE_GAOGUAI = 4;
	    public static final int TYPE_KONGLING = 5;
	
	    static {
	        System.loadLibrary("fmodL");
	        System.loadLibrary("fmod");
	        System.loadLibrary("effect-lib");
	    }
	
	    public native static void playFixVoice(String path, int type);
	
	}

###在C/C++层中实现native方法

	#include "inc/fmod.hpp"
	#include <stdlib.h>
	#include <unistd.h>
	#include <jni.h>
	
	#include <android/log.h>
	//日志输出宏定义
	#define LOGI(FORMAT, ...) __android_log_print(ANDROID_LOG_INFO,"jason",FORMAT,##__VA_ARGS__);
	#define LOGE(FORMAT, ...) __android_log_print(ANDROID_LOG_ERROR,"jason",FORMAT,##__VA_ARGS__);
	
	//变声的类型
	#define TYPE_NORMAL 0//正常
	#define TYPE_LUOLI 1//萝莉
	#define TYPE_DASHU 2//大叔
	#define TYPE_JINGSONG 3//惊悚
	#define TYPE_GAOGUAI 4//搞怪
	#define TYPE_KONGLING 5//空灵
	
	//使用fmod的命名空间
	using namespace FMOD;
	
	extern "C" {
	
	//Java native方法的实现
	JNIEXPORT void JNICALL
	Java_com_nan_voicechange_VoiceEffectUtils_playFixVoice(JNIEnv *env, jclass type_, jstring path_, jint type) {
	
	    const char *path = env->GetStringUTFChars(path_, NULL);
	
	    System *system = NULL;
	    Sound *sound = NULL;
	    Channel *channel = NULL;
	    DSP *dsp = NULL;
	    bool isPlaying = true;
	    float frequency;
	
	    //fmod初始化
	    System_Create(&system);
	    //指定最大的声轨数等参数
	    system->init(32, FMOD_INIT_NORMAL, NULL);
	
	    //创建声音对象
	    system->createSound(path, FMOD_DEFAULT, NULL, &sound);

	    switch (type) {
	        case TYPE_NORMAL:
	            //原声播放
	            //指定的是音轨0，最后update的时候才会播放
	            system->playSound(sound, 0, false, &channel);
	            break;
	
	        case TYPE_LUOLI:
	            //创建一个数字信号处理对象DSP
	    		//DSP（数字信号处理）主要原理是：通过改变声音的两个参数：响度（振幅） 声调（频率）
	            system->createDSPByType(FMOD_DSP_TYPE_PITCHSHIFT, &dsp);
	            //设置参数，提高频率，升高一个八度
	            dsp->setParameterFloat(FMOD_DSP_PITCHSHIFT_PITCH, 2.5);
	            //把处理对象添加到Channel的音轨0中(注意这里要先播放然后添加音轨特效)
	            system->playSound(sound, 0, false, &channel);
	            channel->addDSP(0, dsp);
	            LOGE("%s", "luo li");
	            break;
	
	        case TYPE_DASHU:
	            //创建一个数字信号处理对象DSP
	            system->createDSPByType(FMOD_DSP_TYPE_PITCHSHIFT, &dsp);
	            //设置参数，提高频率，升高一个八度
	            dsp->setParameterFloat(FMOD_DSP_PITCHSHIFT_PITCH, 0.8);
	            //把处理对象添加到Channel的音轨0中
	            system->playSound(sound, 0, false, &channel);
	            channel->addDSP(0, dsp);
	            LOGE("%s", "da shu");
	            break;
	
	        case TYPE_JINGSONG:
	            //惊悚
	            system->createDSPByType(FMOD_DSP_TYPE_TREMOLO, &dsp);
	            dsp->setParameterFloat(FMOD_DSP_TREMOLO_SKEW, 0.5);
	            system->playSound(sound, 0, false, &channel);
	            channel->addDSP(0, dsp);
	            break;
	
	        case TYPE_GAOGUAI:
	            //搞怪
	            //提高说话的速度
	            system->playSound(sound, 0, false, &channel);
	            channel->getFrequency(&frequency);
	            frequency = frequency * 1.6;
	            channel->setFrequency(frequency);
	            LOGI("%s", "fix gaoguai");
	            break;
	
	        case TYPE_KONGLING:
	            //空灵
	            system->createDSPByType(FMOD_DSP_TYPE_ECHO, &dsp);
	            dsp->setParameterFloat(FMOD_DSP_ECHO_DELAY, 300);
	            dsp->setParameterFloat(FMOD_DSP_ECHO_FEEDBACK, 20);
	            system->playSound(sound, 0, false, &channel);
	            channel->addDSP(0, dsp);
	            LOGI("%s", "fix kongling");
	            break;
	
	        default:
	            break;
	    }
	
	    //update的时候才会播放
	    system->update();
	
	    //每秒钟判断一下是否在播放
	    while (isPlaying) {
	        channel->isPlaying(&isPlaying);
	        usleep(1 * 1000 * 1000);//单位是微妙，这里是1秒延时
	    }
	
	    //CMake默认支持异常处理。
		//播放的时候可能会有异常，例如文件找不到等等，然后把异常抛给Java，这里就省略了
	    //    try {
	    //
	    //    } catch (...) {
	    //
	    //    }
	
	
	    //释放资源
	    sound->release();
	    system->close();
	    system->release();
	
	    env->ReleaseStringUTFChars(path_, path);
	}
	
	}

相关源码：https://github.com/huannan/VoiceChange

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
