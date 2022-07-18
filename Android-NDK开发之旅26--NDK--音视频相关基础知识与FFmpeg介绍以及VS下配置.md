###音视频基础知识

####视频播放原理

![视频播放器](https://upload-images.jianshu.io/upload_images/2570030-c4fc35fa0df2bb4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们先从一个简单的视频播放器的原理开始讲述，下图是一个最简单的视频播放的过程（不包括视频加密等等过程）：

![视频播放原理](http://upload-images.jianshu.io/upload_images/2570030-378928c16d48f49d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是一个视频播放的最基本的原理流程图，从这个图可以很整体得看到视频处理的一些主要步骤，后面我们会详细介绍一些这里提到的基本概念。

######注意：我们利用FFmpeg进行编程的时候几乎就是基于这个流程图来进行。比如说，编程的时候我们会拿到解码器，解码读取数据，绘制到屏幕上面的时候可能还需要把YUV数据转换为RGB等等。

我们常见的封装视频的格式有：flv（音视频分开）、mp4、avi等等。后面我们会详细说明。

####为什么视频需要经过封装处理呢？

因为摄像头采集到的画面、以及麦克风采集到的音频数据是经过压缩的处理，不然视频文件就会很大。

也就是说：

1. 录像、录音，实质是一个压缩采集的图像或者声音的过程。这个过程就是视频编码压缩的过程。
2. 播放视频、音频文件实质上就是解压缩的过程，这个过程又称为解码。

####视频的封装格式介绍

封装格式的作用是：视频码流和音频码流按照一定的格式存储在一个文件中。

封装格式分析工具：Elecard Format Analyzer

为什么要音视频分开存储呢？因为音视频的编码格式各种各样，同时编码必然会造成混乱。

常见的视频封装格式有：

![视频封装格式](http://upload-images.jianshu.io/upload_images/2570030-8d0b33ee909d23a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以两个格式为例子，介绍一下原理：

![常见视频封装格式原理](http://upload-images.jianshu.io/upload_images/2570030-380bc36cf0a4e26f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. MPEG2-TS格式是由一个一个数据大小固定的TS-Packet组成，因此可以支持快进。
2. FLY格式由FLV HEADER以及一个一个大小不固定的Tag组成。因为FLV格式直接能够用flash（浏览器）播放，因此常用于视频直播邻域。我们在做RTMP推流的时候，一开始就需要发送头信息。因为数据单元大小不固定，因此原生的视频播放器不支持FLV视频的快进（有些播放器进行了处理可以快进）。

####视频编解码常见格式介绍

视频的压缩算法很多，因此编码格式就会有很多种，下面介绍一些常见的编解码格式：

#####视频编解码格式：

1. 常见的视频编码格式有：H.264、MPEG2、VP8等（谷歌收购的WebRTC视频通话就是用VP8）。
2. 视频解码得到的像素数据YUV、RGB。YUV格式中，Y代表亮度，UV代表色度，人眼对亮度比较敏感，两者比例为4:1，与生物学的理论有关。

![视频编码格式](http://upload-images.jianshu.io/upload_images/2570030-9254164f6838c8aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

原理分析：

![视频编码格式原理分析](http://upload-images.jianshu.io/upload_images/2570030-56998f35bca5f31b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以H264为例，H264是由大小不固定的NALU构成。（NALU实质是一种数据结构）。H264里面有很多子压缩算法，原理比较复杂，包括了熵编码，环路滤波，帧内检测，帧间检测等知识。H264编码原理比较复杂，因此H264的压缩效率是几百到几千倍。

######我们需要学会FFmpeg即可，因为这个库封装了H264等格式的处理。

视频解码（摄像机获取）得到的是视频像素数据，保存了屏幕上每个像素点的像素值。常见的像素数据格式有RGB24， RGB32， YUV420P，YUV422P，YUV444P等。压缩编码中一般使用的是YUV格式的像素数据，最为常见的格式为YUV420P。

YUV视频格式是没有经过压缩的，很大。早期在电视上面用得比较多，比如古老的黑白、彩电。彩电播放早期的黑白视频实质上是只播放了Y（亮度）的数据，因为黑白视频只有Y的数据嘛。

![YUV格式简介](http://upload-images.jianshu.io/upload_images/2570030-6095892b656be550.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RGB也有很多种，比如RGB24，不同的RGB编码色彩丰富度不同。

![RGB格式简介](http://upload-images.jianshu.io/upload_images/2570030-4a49dbbd7f8d862f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####音视频编解码格式：

1. 常见的音频编码格式有：AAC、MP3。
2. 音频解码得到的是音频采样数据，然后喇叭才能播放。常见格式是PCM，实质是一个一个的采样值。单位时间内震动的数据，包括振幅和频率。常用采样率44100，人耳朵能够擦觉到的最高采样率。

![常见的音频编码格式](http://upload-images.jianshu.io/upload_images/2570030-7d2ae10dae49e7c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

######在做视频直播的时候：音频常用AAC来进行编码，用FAAC库来处理；视频用H264编码。

![音频编码格式AAC介绍](http://upload-images.jianshu.io/upload_images/2570030-0227273d33bd806d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

音频采样数据PCM：保存了音频中每个采样点的值，音频采样数据体积很大，一般需要进过压缩，我们平常说的“无损”实质上是没有损失的压缩的意思。

![PCM格式简介](http://upload-images.jianshu.io/upload_images/2570030-52fed0426fc8f091.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####相关播放（编辑）工具

1. YUV：YUV Player
2. PCM：Adobe Audition
3. 查看视频信息：MediaInfo
4. 视频编码数据：Elecard Format Analyzer
5. 视频编码分析工具：Elecard Stream Eye

有兴趣可以下载玩玩。

###FFmpeg介绍

![FFmpeg](https://upload-images.jianshu.io/upload_images/2570030-9b746adfa726d801.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![FFmpeg百科](https://upload-images.jianshu.io/upload_images/2570030-28a5d203a0f48ee8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


FFmpeg是开源的C/C++音视频处理的类库，这个库十分优秀，以至于很多大公司都在用。主流的视频播放器几乎都使用了FFmpeg。

####FFmpeg的八个函数库的基本介绍

如下图所示：

![FFmpeg库简介](http://upload-images.jianshu.io/upload_images/2570030-241003f297926d1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###Visual Studio下FFmpeg的项目配置

####前言

我们一般是在VS中写好代码然后放到Android中的，因此有必要搭建VS的开发环境。

####FFmpeg资源获取

首先我们需要去FFmpeg的官网去获取源码，因为获取的步骤比较麻烦，固下个笔记记录下来，我们打开[http://ffmpeg.org/](http://ffmpeg.org/ "FFmpeg官网")：

![FFmpeg官网.png](http://upload-images.jianshu.io/upload_images/2570030-fdeac336402fa53d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击官网中大大的Download按钮，跳转到下面的界面：

![下载Windows版本的资源.png](http://upload-images.jianshu.io/upload_images/2570030-38097c53d675f0ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择对应的系统，这里我们先介绍Windows版本的，点击下面的Windows Builds，跳转到下面的界面：

![下载Windows版本的资源.png](http://upload-images.jianshu.io/upload_images/2570030-f3f1abf9f8d9b993.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们推荐使用旧版的FFmpeg库，因为如果使用新版的话，除了问题很难去百度。笔者的电脑是64位的，于是就点击All 64-bit Downloads。然后我们会跳转到下面这个仓库页面：

![FFmpeg仓库.png](http://upload-images.jianshu.io/upload_images/2570030-40d60648bc9ffb26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，我们需要下载的FFmpeg版本是2.8系列的，我们推荐使用2.8或者以下的版本。其中，dev是开发版本的库，shared是一些动态链接库，static是一些已经编译好的exe（Windows版本）可执行文件。这三个我们都需要下载下来。

如果你嫌麻烦的话，我下面直接给出下载地址：

https://ffmpeg.zeranoe.com/builds/win64/dev/2015/ffmpeg-20151105-git-c878082-win64-dev.7z

https://ffmpeg.zeranoe.com/builds/win64/shared/2015/ffmpeg-20151105-git-c878082-win64-shared.7z

https://ffmpeg.zeranoe.com/builds/win64/static/2015/ffmpeg-20151105-git-c878082-win64-static.7z

下面并解压的效果如下：

![效果.png](http://upload-images.jianshu.io/upload_images/2570030-5039bfd792ad2f11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：下面分别用dev、static、shared来代表这三个文件夹。

####在命令行玩一玩static中的可执行文件

我们打开static文件夹，里面有个bin目录，有三个exe文件。这就是我们即将要玩的东西：

![可执行文件.png](http://upload-images.jianshu.io/upload_images/2570030-f5a02001607187b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了简化操作，我们不妨把bin目录添加到环境变量path中。

然后我们准备一个测试用的视频，例如笔者准备了一个test.flv视频文件。

打开命令行，输入：

	ffmpeg -i test.flv test.avi

然后这就完成了一次简单的视频格式转换。相信细心的你也会发现，FFmpeg的官网上面有这么一幅图：

![官网的例子.png](http://upload-images.jianshu.io/upload_images/2570030-274e3752def86c3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实这就是一个最简单的例子。

下面我们再来搞一个是视频转GIF，在命令行输入下面的语句：

	ffmpeg -ss 0 -t 11 -i test.flv -s 1366x768 -b:v 1500k test.gif

意思就是把test.flv转换为test.gif文件，其中需要指定转换的时间范围，分辨率大小，比特率。

######码率（比特率），单位时间每一帧画面以及音频的大小，也叫作比特每秒，单位时间内播放连续的媒体例如压缩后的音频或者视频的比特数量。码率越高，音视频越清晰。

######把视频转gif是很有意义的，例如微信中的小视频，我们没有点开的时候，其实播放的是一个gif文件（可能使用的是libgif这个NDK库），用户点击打开的时候，才会从服务器下载真正的视频文件。这样做大大降低了服务器的压力。

最后我们看一个播放器的例子，输入下面的命令，就会打开一个播放器，所以说如果你想研究一个Android平台的播放器，那么需要研究ffplay相关的代码：

	ffplay test.flv

####FFmpeg的VS项目配置

我们首先创建一个空项目，然后把dev中的include、lib两个目录拷贝到项目的根路径下的源代码目录中（默认是与项目名一样）。

然后在项目属性中配置附加库目录：

![包含目录.png](http://upload-images.jianshu.io/upload_images/2570030-155273ba90eadcae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后配置附加库目录：

![附加库目录.png](http://upload-images.jianshu.io/upload_images/2570030-8ce99a72c5d22204.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后配置有哪些附加库（附加依赖项），如下面所示：

	avcodec.lib
	avdevice.lib
	avfilter.lib
	avformat.lib
	avutil.lib
	postproc.lib
	swresample.lib
	swscale.lib

![附加依赖项配置.png](http://upload-images.jianshu.io/upload_images/2570030-c159f2108e1a0322.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后我们创建一个测试用的CPP文件：

	#include <stdlib.h>
	#include <stdio.h>
	#include <iostream>
	
	using namespace std;
	
	//由于FFmpeg是C和C++混编的，因此使用extern是为了解决兼容问题
	extern "C"{
	#include "libavcodec/avcodec.h"
	}
	
	void main(){
		//输出FFmpeg的配置信息，检查是否配置好
		cout << avcodec_configuration() << endl;
		system("pause");
	}

然后你会发现编译不过，因为我们用的FFmpeg库是64位的，因此需要把我们的平台改为64位的：

![配置平台为64位.png](http://upload-images.jianshu.io/upload_images/2570030-c133dbbf8019bdb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后编译通过，输出的结果如下：

![配置结果.png](http://upload-images.jianshu.io/upload_images/2570030-a7e564c3843f5e78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####题外话——关于extern关键字的基本解释

extern可以置于变量或者函数前，以标示变量或者函数的定义在别的文件中，提示编译器遇到此变量和函数时在其他模块中寻找其定义。此外extern也可用来进行链接指定。也就是说extern有两个作用：

1. 第一个,当它与"C"一起连用时，如: extern "C" void fun(int a, int b);则告诉编译器在编译fun这个函数名时按着C的规则去翻译相应的函数名而不是C++的，C++的规则在翻译这个函数名时会把fun这个名字变得面目全非，可能是fun@aBc_int_int#%$也可能是别的，这要看编译器的"脾气"了(不同的编译器采用的方法不一样)，为什么这么做呢，因为C++支持函数的重载啊，在这里不去过多的论述这个问题，如果你有兴趣可以去网上搜索，相信你可以得到满意的解释!
2. 第二，当extern不与"C"在一起修饰变量或函数时，如在头文件中: extern int g_Int; 它的作用就是声明函数或全局变量的作用范围的关键字，其声明的函数和变量可以在本模块活其他模块中使用，记住它是一个声明不是定义!也就是说B模块(编译单元)要是引用模块(编译单元)A中定义的全局变量或函数时，它只要包含A模块的头文件即可,在编译阶段，模块B虽然找不到该函数或变量，但它不会报错，它会在连接时从模块A生成的目标代码中找到此函数。

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
