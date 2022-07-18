###前言

上一篇文章我们编译输出了Android工程需要的动态库文件。然后下面我们利用这个库进行视频解码。

###视频解码

####一、创建Android Studio工程

创建Android Studio工程，记得带上C++Support。

然后编写CMake脚本：

	#配置CMake的最低版本
	cmake_minimum_required(VERSION 3.4.1)
	
	#配置工程路径
	#set(path_project /home/wuhuannan/AndroidPrj/FFmpeg)
	set(path_project D:/AndroidStudio/FFmpeg)
	
	#配置头文件的包含路径
	include_directories(${path_project}/app/src/main/cpp/include)
	
	#添加自己写的库
	add_library(ffmpeg-lib
	            SHARED
	            src/main/cpp/ffmpeg.c
	            )
	
	#添加FFmpeg预编译的so库
	add_library( avutil
	             SHARED
	             IMPORTED
	            )
	
	#设置两个预编译的库的路径
	set_target_properties(
	                       avutil
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libavutil-54.so
	                    )
	
	add_library( swresample
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       swresample
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libswresample-1.so
	                    )
	
	add_library( avcodec
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       avcodec
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libavcodec-56.so
	                    )
	
	add_library( avformat
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       avformat
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libavformat-56.so
	                    )
	
	add_library( swscale
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       swscale
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libswscale-3.so
	                    )
	
	add_library( postproc
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       postproc
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libpostproc-53.so
	                    )
	
	add_library( avfilter
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       avfilter
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libavfilter-5.so
	                    )
	
	add_library( avdevice
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       avdevice
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libavdevice-56.so
	                    )
	
	#找到Android的log库（这个库已经在Android平台中了）
	find_library(
	            log-lib
	            log
	            )
	
	#把需要的库链接到自己的库中
	target_link_libraries(
	            ffmpeg-lib
	            ${log-lib}
	            avutil
	            swresample
	            avcodec
	            avformat
	            swscale
	            postproc
	            avfilter
	            avdevice
	            )

主要工作就是加载我们上一篇文章中的几个动态库，然后链接到我们自己的库里面。

####二、编写Java方法进行测试

	package com.nan.ffmpeg.utils;
	
	public class VideoUtils {
	
	    static {
	        System.loadLibrary("avutil-54");
	        System.loadLibrary("swresample-1");
	        System.loadLibrary("avcodec-56");
	        System.loadLibrary("avformat-56");
	        System.loadLibrary("swscale-3");
	        System.loadLibrary("postproc-53");
	        System.loadLibrary("avfilter-5");
	        System.loadLibrary("avdevice-56");
	        System.loadLibrary("ffmpeg-lib");
	    }
	
		//视频解码
	    public native static void decode(String input, String outptu);
	}

######这里需要注意的是，FFmpeg的动态库是有依赖关系的，加载的时候需要按照一定的顺序，先加载最基础的库。

####三、JNI实现

FFmpeg视频解码流程如下：

![FFmpeg视频解码流程](http://upload-images.jianshu.io/upload_images/2570030-34d49a747028627e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

FFmpeg中相关的示例程序所在文件路径：

	ffmpeg-2.6.9\doc\examples

我们编程都是按照这个流程进行的。

	#include <jni.h>
	#include <string.h>
	#include <android/log.h>
	
	//编码
	#include "libavcodec/avcodec.h"
	//封装格式处理
	#include "libavformat/avformat.h"
	//像素处理
	#include "libswscale/swscale.h"
	
	#define LOGI(FORMAT, ...) __android_log_print(ANDROID_LOG_INFO,"wuhuannan",FORMAT,##__VA_ARGS__);
	#define LOGE(FORMAT, ...) __android_log_print(ANDROID_LOG_ERROR,"wuhuannan",FORMAT,##__VA_ARGS__);
	
	//有时候需要用C文件而不是CPP
	
	JNIEXPORT void JNICALL
	Java_com_nan_ffmpeg_utils_VideoUtils_decode(JNIEnv *env, jclass type, jstring input_,
	                                            jstring outptu_) {
	
	    //需要转码的视频文件(输入的视频文件)
	    const char *input = (*env)->GetStringUTFChars(env, input_, 0);
	    const char *outptu = (*env)->GetStringUTFChars(env, outptu_, 0);
	
	    //1.注册所有组件，例如初始化一些全局的变量、初始化网络等等
	    av_register_all();
	    //avcodec_register_all();
	
	    //封装格式上下文，统领全局的结构体，保存了视频文件封装格式的相关信息
	    AVFormatContext *pFormatCtx = avformat_alloc_context();
	
	    //2.打开输入视频文件
	    if (avformat_open_input(&pFormatCtx, input, NULL, NULL) != 0) {
	        LOGE("%s", "无法打开输入视频文件");
	        return;
	    }
	
	    //3.获取视频文件信息，例如得到视频的宽高
	    //第二个参数是一个字典，表示你需要获取什么信息，比如视频的元数据
	    if (avformat_find_stream_info(pFormatCtx, NULL) < 0) {
	        LOGE("%s", "无法获取视频文件信息");
	        return;
	    }
	
	    //获取视频流的索引位置
	    //遍历所有类型的流（音频流、视频流、字幕流），找到视频流
	    int v_stream_idx = -1;
	    int i = 0;
	    //number of streams
	    for (; i < pFormatCtx->nb_streams; i++) {
	        //流的类型
	        if (pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO) {
	            v_stream_idx = i;
	            break;
	        }
	    }
	
	    if (v_stream_idx == -1) {
	        LOGE("%s", "找不到视频流\n");
	        return;
	    }
	
	    //只有知道视频的编码方式，才能够根据编码方式去找到解码器
	    //获取视频流中的编解码上下文
	    AVCodecContext *pCodecCtx = pFormatCtx->streams[v_stream_idx]->codec;
	    //4.根据编解码上下文中的编码id查找对应的解码
	    AVCodec *pCodec = avcodec_find_decoder(pCodecCtx->codec_id);
	    //（迅雷看看，找不到解码器，临时下载一个解码器，比如视频加密了）
	    if (pCodec == NULL) {
	        LOGE("%s", "找不到解码器，或者视频已加密\n");
	        return;
	    }
	
	    //5.打开解码器，解码器有问题（比如说我们编译FFmpeg的时候没有编译对应类型的解码器）
	    if (avcodec_open2(pCodecCtx, pCodec, NULL) < 0) {
	        LOGE("%s", "解码器无法打开\n");
	        return;
	    }
	
	    //输出视频信息
	    LOGI("视频的文件格式：%s", pFormatCtx->iformat->name);
	    LOGI("视频时长：%lld", (pFormatCtx->duration) / 1000000);
	    LOGI("视频的宽高：%d,%d", pCodecCtx->width, pCodecCtx->height);
	    LOGI("解码器的名称：%s", pCodec->name);
	
	    //准备读取
	    //AVPacket用于存储一帧一帧的压缩数据（H264）
	    //缓冲区，开辟空间
	    AVPacket *packet = (AVPacket *) av_malloc(sizeof(AVPacket));
	
	    //AVFrame用于存储解码后的像素数据(YUV)
	    //内存分配
	    AVFrame *pFrame = av_frame_alloc();
	    //YUV420
	    AVFrame *pFrameYUV = av_frame_alloc();
	    //只有指定了AVFrame的像素格式、画面大小才能真正分配内存
	    //缓冲区分配内存
	    uint8_t *out_buffer = (uint8_t *) av_malloc(
	            avpicture_get_size(AV_PIX_FMT_YUV420P, pCodecCtx->width, pCodecCtx->height));
	    //初始化缓冲区
	    avpicture_fill((AVPicture *) pFrameYUV, out_buffer, AV_PIX_FMT_YUV420P, pCodecCtx->width,
	                   pCodecCtx->height);
	
	    //用于转码（缩放）的参数，转之前的宽高，转之后的宽高，格式等
	    struct SwsContext *sws_ctx = sws_getContext(pCodecCtx->width, pCodecCtx->height,
	                                                pCodecCtx->pix_fmt,
	                                                pCodecCtx->width, pCodecCtx->height,
	                                                AV_PIX_FMT_YUV420P,
	                                                SWS_BICUBIC, NULL, NULL, NULL);
	
	
	    int got_picture, ret;
	
	    FILE *fp_yuv = fopen(outptu, "wb+");
	
	    int frame_count = 0;
	
	    //6.一帧一帧的读取压缩数据
	    while (av_read_frame(pFormatCtx, packet) >= 0) {
	        //只要视频压缩数据（根据流的索引位置判断）
	        if (packet->stream_index == v_stream_idx) {
	            //7.解码一帧视频压缩数据，得到视频像素数据
	            ret = avcodec_decode_video2(pCodecCtx, pFrame, &got_picture, packet);
	            if (ret < 0) {
	                LOGE("%s", "解码错误");
	                return;
	            }
	
	            //为0说明解码完成，非0正在解码
	            if (got_picture) {
	                //AVFrame转为像素格式YUV420，宽高
	                //2 6输入、输出数据
	                //3 7输入、输出画面一行的数据的大小 AVFrame 转换是一行一行转换的
	                //4 输入数据第一列要转码的位置 从0开始
	                //5 输入画面的高度
	                sws_scale(sws_ctx, pFrame->data, pFrame->linesize, 0, pCodecCtx->height,
	                          pFrameYUV->data, pFrameYUV->linesize);
	
	                //输出到YUV文件
	                //AVFrame像素帧写入文件
	                //data解码后的图像像素数据（音频采样数据）
	                //Y 亮度 UV 色度（压缩了） 人对亮度更加敏感
	                //U V 个数是Y的1/4
	                int y_size = pCodecCtx->width * pCodecCtx->height;
	                fwrite(pFrameYUV->data[0], 1, y_size, fp_yuv);
	                fwrite(pFrameYUV->data[1], 1, y_size / 4, fp_yuv);
	                fwrite(pFrameYUV->data[2], 1, y_size / 4, fp_yuv);
	
	                frame_count++;
	                LOGI("解码第%d帧", frame_count);
	            }
	        }
	
	        //释放资源
	        av_free_packet(packet);
	    }
	
	    fclose(fp_yuv);
	
	    av_frame_free(&pFrame);
	
	    avcodec_close(pCodecCtx);
	
	    avformat_free_context(pFormatCtx);
	
	    (*env)->ReleaseStringUTFChars(env, input_, input);
	    (*env)->ReleaseStringUTFChars(env, outptu_, outptu);
	}


####四、在Activity中测试

    @Override
    public void onClick(View v) {

        String inputVideo = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separatorChar
                + "input.mp4";

        String outputVideo = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separatorChar
                + "output_1280x720_yuv420p.yuv";

        switch (v.getId()) {

            case R.id.btn_decode:
                VideoUtils.decode(inputVideo, outputVideo);
                break;

        }
    }


相关源代码：https://github.com/huannan/FFmpeg

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
