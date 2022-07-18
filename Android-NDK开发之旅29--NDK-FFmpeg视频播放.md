###前言

上一篇文章我们对视频进行了解码，那么这次我们队解码后的数据进行播放。也就是绘制到界面上。

###视频播放

####创建自动以SurfaceView

因为视频是需要快速实时刷新界面的，因此要用到SurfaceView。

	public class VideoView extends SurfaceView {
	
	    public VideoView(Context context) {
	        this(context, null, 0);
	    }
	
	    public VideoView(Context context, AttributeSet attrs) {
	        this(context, attrs, 0);
	    }
	
	    public VideoView(Context context, AttributeSet attrs, int defStyleAttr) {
	        super(context, attrs, defStyleAttr);
	        init();
	    }
	
	    private void init() {
	        //初始化像素绘制的格式为RGBA_8888（色彩最丰富）
	        SurfaceHolder holder = getHolder();
	        holder.setFormat(PixelFormat.RGBA_8888);
	    }
	
	}

这里我们自定义了一个SurfaceView，指定输出格式为RGBA_8888，这种格式色彩丰富度最高的。

然后添加到根布局当中：

	<?xml version="1.0" encoding="utf-8"?>
	<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:app="http://schemas.android.com/apk/res-auto"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent">
	
	    <com.nan.ffmpeg.view.VideoView
	        android:id="@+id/sv_video"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent" />
	
	    <Button
	        android:id="@+id/btn_play"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="播放" />
	
	</FrameLayout>

####创建播放控制器类

	public class VideoPlayer {
	
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
	
	    public native void render(String input, Surface surface);
	
	}

####native方法实现

	//编码
	#include "libavcodec/avcodec.h"
	//封装格式处理
	#include "libavformat/avformat.h"
	//像素处理
	#include "libswscale/swscale.h"
	//使用这两个Window相关的头文件需要在CMake脚本中引入android库
	#include <android/native_window_jni.h>
	#include <android/native_window.h>
	#include "libyuv.h"
	
	JNIEXPORT void JNICALL
	Java_com_nan_ffmpeg_utils_VideoPlayer_render(JNIEnv *env, jobject instance, jstring input_,
	                                             jobject surface) {
	
	    //需要转码的视频文件(输入的视频文件)
	    const char *input = (*env)->GetStringUTFChars(env, input_, 0);
	
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
	    //4.根据编解码上下文中的编码id查找对应的解码器
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
	
	    //准备读取
	    //AVPacket用于存储一帧一帧的压缩数据（H264）
	    //缓冲区，开辟空间
	    AVPacket *packet = (AVPacket *) av_malloc(sizeof(AVPacket));
	
	    //AVFrame用于存储解码后的像素数据(YUV)
	    //内存分配
	    AVFrame *yuv_frame = av_frame_alloc();
	    AVFrame *rgb_frame = av_frame_alloc();
	
	    int got_picture, ret;
	    int frame_count = 0;
	
	    //窗体
	    ANativeWindow *pWindow = ANativeWindow_fromSurface(env, surface);
	    //绘制时的缓冲区
	    ANativeWindow_Buffer out_buffer;
	
	    //6.一帧一帧的读取压缩数据
	    while (av_read_frame(pFormatCtx, packet) >= 0) {
	        //只要视频压缩数据（根据流的索引位置判断）
	        if (packet->stream_index == v_stream_idx) {
	            //7.解码一帧视频压缩数据，得到视频像素数据
	            ret = avcodec_decode_video2(pCodecCtx, yuv_frame, &got_picture, packet);
	            if (ret < 0) {
	                LOGE("%s", "解码错误");
	                return;
	            }
	
	            //为0说明解码完成，非0正在解码
	            if (got_picture) {
	
	                //1、lock window
	                //设置缓冲区的属性：宽高、像素格式（需要与Java层的格式一致）
	                ANativeWindow_setBuffersGeometry(pWindow, pCodecCtx->width, pCodecCtx->height,
	                                                 WINDOW_FORMAT_RGBA_8888);
	                ANativeWindow_lock(pWindow, &out_buffer, NULL);
	
	                //2、fix buffer
	
	                //初始化缓冲区
	                //设置属性，像素格式、宽高
	                //rgb_frame的缓冲区就是Window的缓冲区，同一个，解锁的时候就会进行绘制
	                avpicture_fill((AVPicture *) rgb_frame, out_buffer.bits, AV_PIX_FMT_RGBA,
	                               pCodecCtx->width,
	                               pCodecCtx->height);
	
	                //YUV格式的数据转换成RGBA 8888格式的数据
	                //FFmpeg可以转，但是会有问题，因此我们使用libyuv这个库来做
	                //https://chromium.googlesource.com/external/libyuv
	                //参数分别是数据、对应一行的大小
	                //I420ToARGB(yuv_frame->data[0], yuv_frame->linesize[0],
	                //           yuv_frame->data[1], yuv_frame->linesize[1],
	                //           yuv_frame->data[2], yuv_frame->linesize[2],
	                //           rgb_frame->data[0], rgb_frame->linesize[0],
	                //           pCodecCtx->width, pCodecCtx->height);
	
	                I420ToARGB(yuv_frame->data[0], yuv_frame->linesize[0],
	                           yuv_frame->data[2], yuv_frame->linesize[2],
	                           yuv_frame->data[1], yuv_frame->linesize[1],
	                           rgb_frame->data[0], rgb_frame->linesize[0],
	                           pCodecCtx->width, pCodecCtx->height);
	
	                //3、unlock window
	                ANativeWindow_unlockAndPost(pWindow);
	
	                frame_count++;
	                LOGI("解码绘制第%d帧", frame_count);
	            }
	        }
	
	        //释放资源
	        av_free_packet(packet);
	    }
	
	    av_frame_free(&yuv_frame);
	    avcodec_close(pCodecCtx);
	    avformat_free_context(pFormatCtx);
	    (*env)->ReleaseStringUTFChars(env, input_, input);
	}

代码里面需要注意的是，我们使用了yuvlib这个库对YUV数据转换为RGB数据。这个库的下载地址是https://chromium.googlesource.com/external/libyuv。

修改Android.mk文件最后一行，由输出静态库改为输出动态库：

	include $(BUILD_SHARED_LIBRARY)

自己新建jni目录，把所有文件拷贝到里面，Linux执行下面的命令编译yuvlib：

	ndk-build jni

然后就会输出预编译的so库，并且在Android Studio中使用了。CMake脚本需要添加：

	#YUV转RGB需要的库
	add_library( yuv
	             SHARED
	             IMPORTED
	            )
	
	set_target_properties(
	                       yuv
	                       PROPERTIES IMPORTED_LOCATION
	                       ${path_project}/app/libs/${ANDROID_ABI}/libyuv.so
	                    )

如果需要的话设置一些头文件的包含路径：

	#配置头文件的包含路径
	include_directories(${path_project}/app/src/main/cpp/include/yuv)

最后在使用的时候包含对应的头文件即可：

	#include "libyuv.h"

还有一个注意的地方就是我们要使用到窗口的原生绘制，那么就需要引入window相关的头文件：

	//使用这两个Window相关的头文件需要在CMake脚本中引入android库
	#include <android/native_window_jni.h>
	#include <android/native_window.h>

这些头文件使用需要把android这个库编译进来，使用方法跟log库一样：

	#找到Android的Window绘制相关的库（这个库已经在Android平台中了）
	find_library(
	            android-lib
	            android
	            )

记得链接到自己的库里面：

	#把需要的库链接到自己的库中
	target_link_libraries(
	            ffmpeg-lib
	            ${log-lib}
	            ${android-lib}
	            avutil
	            swresample
	            avcodec
	            avformat
	            swscale
	            postproc
	            avfilter
	            avdevice
	            yuv
	            )


####窗口的原生绘制流程

绘制需要一个surface对象。

原生绘制步骤：

1. lock Window。
2. 初始化缓冲区，设置大小，缓冲区赋值。
3. 解锁然后就绘制到窗口中了。

####测试

    @Override
    public void onClick(View v) {
        String inputVideo = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separatorChar
                + "input.mp4";

        switch (v.getId()) {

            case R.id.btn_play:
				sv_video = (SurfaceView) findViewById(R.id.sv_video);
                //native方法中传入SurfaceView的Surface对象，在底层进行绘制
                mPlayer.render(inputVideo, sv_video.getHolder().getSurface());
                break;

        }
    }

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
