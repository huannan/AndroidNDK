###前言

我们进行Android FFmpeg开发的时候，需要一些FFmpeg预编译的库，这些预编译的so库需要在Linux环境下编译。

###Linux下FFmpeg编译

由于笔者公司的电脑是Ubuntu14.04系统，因此我们直接使用Ubuntu系统进行编译。读者也可以自己搭建Linux虚拟机或者购买云服务器。

####关于云服务器的购买

1. 买阿里云主机  最好是Ubuntu系统的。其中阿里云的华北一核1GB最便宜。
2. 我们需要安装XShell客户端（内含XFTP）来对服务器进行操作、文件传输。通过公网地址进行连接即可。
3. 为了方便操作，直接使用root用户即可，但是一般我们都是需要创建用户的。

####一、准备NDK

编译FFmpeg的时候需要用到NDK。

在Linux环境中，我们需要有一个NDK的压缩包，例如：

	android-ndk-r10e-linux-x86_64.bin

然后执行下面的命令进行解压缩即可（注意权限问题）：

	./android-ndk-r10e-linux-x86_64.bin

####二、配置NDK环境变量

环境变量配置

	vim ~/.bashrc(~代表当前用户)

编辑：

	export NDKROOT=你的NDK根目录
	export PATH=$NDKROOT:$PATH

更新（不然的话就需要重启命令行） 

	source ~/.bashrc

####三、准备FFmpeg

我们需要去FFmpeg官网下载FFmpeg的Linux源码，版本不需要太新：

	ffmpeg-2.6.9.zip

然后执行解压缩命令：

	uzip ffmpeg-2.6.9.zip

####四、编写shell脚本

我们需要编写shell脚本传参调用ffmpeg-2.6.9下的configure脚本，进行编译，我们写的shell脚本如下，build_android.sh：

	#!/bin/bash
	make clean
	export NDK=/home/wuhuannan/Android/Sdk/ndk-bundle
	export SYSROOT=$NDK/platforms/android-14/arch-arm/
	export TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64
	export CPU=arm
	export PREFIX=$(pwd)/android/$CPU
	export ADDI_CFLAGS="-marm"
	
	./configure --target-os=linux \
	--prefix=$PREFIX --arch=arm \
	--disable-doc \
	--enable-shared \
	--disable-static \
	--disable-yasm \
	--disable-symver \
	--enable-gpl \
	--disable-ffmpeg \
	--disable-ffplay \
	--disable-ffprobe \
	--disable-ffserver \
	--disable-doc \
	--disable-symver \
	--cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
	--enable-cross-compile \
	--sysroot=$SYSROOT \
	--extra-cflags="-Os -fpic $ADDI_CFLAGS" \
	--extra-ldflags="$ADDI_LDFLAGS" \
	$ADDITIONAL_CONFIGURE_FLAG
	make clean
	make
	make install

指定NDK的一些路径，配置CPU架构类型，PREFIX是指定动态库输出的路径，然后disable一些不需要的库（可减小输出的动态库的大小）等等。enable-shared是生成共享库的意思。

注意：

1. 换行的时候需要有\，主要不要有额外的空格。
2. 统一转为UTF-8无BOM格式。可以通过note pad++进行转码，这样子Windows和LInux都通用了。也可以通过dos2unix命令进行转码。或者先由Linux创建文件再由Windows编辑。
3. NDK尽量不要使用太新的版本，一般使用Android-9即可。新版本会有一些坑，比如LOG2的问题。

####五、修改configure文件

ffmpeg-2.6.9根目录下有个configure文件，这个文件比较重要。我们自己写的脚本文件就是依据这个文件来写的。

通过这个文件我们可以看到FFmpeg库之间的依赖关系，例如：

	avcodec_example_deps="avcodec avutil"

一些编译指令。

这里我们需要修改一下输出的动态库的命名规则：

	#注释的部分
	#SLIBNAME_WITH_MAJOR='$(SLIBNAME).$(LIBMAJOR)'
	#LIB_INSTALL_EXTRA_CMD='$$(RANLIB) "$(LIBDIR)/$(LIBNAME)"'
	#SLIB_INSTALL_NAME='$(SLIBNAME_WITH_VERSION)'
	#SLIB_INSTALL_LINKS='$(SLIBNAME_WITH_MAJOR) $(SLIBNAME)'

	#自己写的部分
	SLIBNAME_WITH_MAJOR='$(SLIBPREF)$(FULLNAME)-$(LIBMAJOR)$(SLIBSUF)'
	LIB_INSTALL_EXTRA_CMD='$$(RANLIB)"$(LIBDIR)/$(LIBNAME)"'
	SLIB_INSTALL_NAME='$(SLIBNAME_WITH_MAJOR)'
	SLIB_INSTALL_LINKS='$(SLIBNAME)'


####六、执行脚本文件

我们需要把我们自己写的build_android.sh放到ffmpeg-2.6.9根目录下，然后授予权限，执行：

	./build_android.sh开始编译

编译过程中会临时先自动生成c.mak文件，头文件等。编译大概几分钟时间。


####七、最终编译输出的动态库文件如下：

![输出的动态库文件](http://upload-images.jianshu.io/upload_images/2570030-82aa88060692c73d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这些库我们就可以直接放到Android Studio工程中使用了。

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
