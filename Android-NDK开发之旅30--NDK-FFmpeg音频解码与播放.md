###音频解码实现

音频解码也是直接使用FFmpeg的API来做。

	public native void sound(String input,String output);

其中，JNI实现如下：

	//重采样
	#include "libswresample/swresample.h"
	
	#define MAX_AUDIO_FRME_SIZE 48000 * 4
	
	JNIEXPORT void JNICALL
	Java_com_nan_ffmpeg_utils_VideoPlayer_sound(JNIEnv *env, jobject instance, jstring input_,
	                                            jstring output_) {
	    const char *input = (*env)->GetStringUTFChars(env, input_, 0);
	    const char *output = (*env)->GetStringUTFChars(env, output_, 0);
	
	    //注册组件
	    av_register_all();
	    AVFormatContext *pFormatCtx = avformat_alloc_context();
	    //打开音频文件
	    if (avformat_open_input(&pFormatCtx, input, NULL, NULL) != 0) {
	        LOGI("%s", "无法打开音频文件");
	        return;
	    }
	    //获取输入文件信息
	    if (avformat_find_stream_info(pFormatCtx, NULL) < 0) {
	        LOGI("%s", "无法获取输入文件信息");
	        return;
	    }
	    //获取音频流索引位置
	    int i = 0, audio_stream_idx = -1;
	    for (; i < pFormatCtx->nb_streams; i++) {
	        if (pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_AUDIO) {
	            audio_stream_idx = i;
	            break;
	        }
	    }
	
	    //获取解码器
	    AVCodecContext *codecCtx = pFormatCtx->streams[audio_stream_idx]->codec;
	    AVCodec *codec = avcodec_find_decoder(codecCtx->codec_id);
	    if (codec == NULL) {
	        LOGI("%s", "无法获取解码器");
	        return;
	    }
	    //打开解码器
	    if (avcodec_open2(codecCtx, codec, NULL) < 0) {
	        LOGI("%s", "无法打开解码器");
	        return;
	    }
	    //压缩数据
	    AVPacket *packet = (AVPacket *) av_malloc(sizeof(AVPacket));
	    //解压缩数据
	    AVFrame *frame = av_frame_alloc();
	    //frame->16bit 44100 PCM 统一音频采样格式与采样率
	    SwrContext *swrCtx = swr_alloc();
	
	    //重采样设置参数-------------start
	    //输入的采样格式
	    enum AVSampleFormat in_sample_fmt = codecCtx->sample_fmt;
	    //输出采样格式16bit PCM
	    enum AVSampleFormat out_sample_fmt = AV_SAMPLE_FMT_S16;
	    //输入采样率
	    int in_sample_rate = codecCtx->sample_rate;
	    //输出采样率
	    int out_sample_rate = 44100;
	    //获取输入的声道布局
	    //根据声道个数获取默认的声道布局（2个声道，默认立体声stereo）
	    //av_get_default_channel_layout(codecCtx->channels);
	    uint64_t in_ch_layout = codecCtx->channel_layout;
	    //输出的声道布局（立体声）
	    uint64_t out_ch_layout = AV_CH_LAYOUT_STEREO;
	
	    swr_alloc_set_opts(swrCtx,
	                       out_ch_layout, out_sample_fmt, out_sample_rate,
	                       in_ch_layout, in_sample_fmt, in_sample_rate,
	                       0, NULL);
	    swr_init(swrCtx);
	
	    //输出的声道个数
	    int out_channel_nb = av_get_channel_layout_nb_channels(out_ch_layout);
	
	    //重采样设置参数-------------end
	
	    //16bit 44100 PCM 数据
	    uint8_t *out_buffer = (uint8_t *) av_malloc(MAX_AUDIO_FRME_SIZE);
	
	    FILE *fp_pcm = fopen(output, "wb");
	
	    int got_frame = 0, index = 0, ret;
	    //不断读取压缩数据
	    while (av_read_frame(pFormatCtx, packet) >= 0) {
	        //解码
	        ret = avcodec_decode_audio4(codecCtx, frame, &got_frame, packet);
	
	        if (ret < 0) {
	            LOGI("%s", "解码完成");
	        }
	        //解码一帧成功
	        if (got_frame > 0) {
	            LOGI("解码：%d", index++);
	            swr_convert(swrCtx, &out_buffer, MAX_AUDIO_FRME_SIZE, frame->data, frame->nb_samples);
	            //获取sample的size
	            int out_buffer_size = av_samples_get_buffer_size(NULL, out_channel_nb,
	                                                             frame->nb_samples, out_sample_fmt, 1);
	            fwrite(out_buffer, 1, out_buffer_size, fp_pcm);
	        }
	
	        av_free_packet(packet);
	    }
	
	    fclose(fp_pcm);
	    av_frame_free(&frame);
	    av_free(out_buffer);
	
	    swr_free(&swrCtx);
	    avcodec_close(codecCtx);
	    avformat_close_input(&pFormatCtx);
	
	    (*env)->ReleaseStringUTFChars(env, input_, input);
	    (*env)->ReleaseStringUTFChars(env, output_, output);
	}

最终会输出pcm格式的文件。

###音频播放

PCM音频播放的常用方法：

1. OpenSL RS
2. Audio Track（WebRTC就用到）

下面我们通过Audio Track来进行播放。

主要的实现思路就是：

1. 在C层调用Java层的createAudioTrack方法，创建AudioTrack对象。
2. 然后在C层调用AudioTrack的pkay、write、stop等方法进行播放控制。

例子：

	package com.nan.ffmpeg.utils;
	
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
	
	    public native void play(String input);
	
	    /**
	     * 创建一个AudioTrac对象，用于播放
	     *
	     * @return
	     */
	    public AudioTrack createAudioTrack(int sampleRateInHz, int nb_channels) {
	        //固定格式的音频码流
	        int audioFormat = AudioFormat.ENCODING_PCM_16BIT;
	        Log.i("jason", "nb_channels:" + nb_channels);
	        //声道布局
	        int channelConfig;
	        if (nb_channels == 1) {
	            channelConfig = android.media.AudioFormat.CHANNEL_OUT_MONO;
	        } else if (nb_channels == 2) {
	            channelConfig = android.media.AudioFormat.CHANNEL_OUT_STEREO;
	        } else {
	            channelConfig = android.media.AudioFormat.CHANNEL_OUT_STEREO;
	        }
	
	        int bufferSizeInBytes = AudioTrack.getMinBufferSize(sampleRateInHz, channelConfig, audioFormat);
	
	        AudioTrack audioTrack = new AudioTrack(
	                AudioManager.STREAM_MUSIC,
	                sampleRateInHz, channelConfig,
	                audioFormat,
	                bufferSizeInBytes, AudioTrack.MODE_STREAM);
	        //播放
	        //audioTrack.play();
	        //写入PCM
	        //audioTrack.write(audioData, offsetInBytes, sizeInBytes);
	        //播放完调用stop即可
	
	        audioTrack.stop();
	        return audioTrack;
	    }
	
	}


JNI实现如下：

	//音频播放
	JNIEXPORT void JNICALL
	Java_com_nan_ffmpeg_utils_VideoPlayer_play(JNIEnv *env, jobject instance, jstring input_) {
	    const char *input = (*env)->GetStringUTFChars(env, input_, 0);
	
	    LOGI("%s", "sound");
	    //注册组件
	    av_register_all();
	    AVFormatContext *pFormatCtx = avformat_alloc_context();
	    //打开音频文件
	    if (avformat_open_input(&pFormatCtx, input, NULL, NULL) != 0) {
	        LOGI("%s", "无法打开音频文件");
	        return;
	    }
	    //获取输入文件信息
	    if (avformat_find_stream_info(pFormatCtx, NULL) < 0) {
	        LOGI("%s", "无法获取输入文件信息");
	        return;
	    }
	    //获取音频流索引位置
	    int i = 0, audio_stream_idx = -1;
	    for (; i < pFormatCtx->nb_streams; i++) {
	        if (pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_AUDIO) {
	            audio_stream_idx = i;
	            break;
	        }
	    }
	
	    //获取解码器
	    AVCodecContext *codecCtx = pFormatCtx->streams[audio_stream_idx]->codec;
	    AVCodec *codec = avcodec_find_decoder(codecCtx->codec_id);
	    if (codec == NULL) {
	        LOGI("%s", "无法获取解码器");
	        return;
	    }
	    //打开解码器
	    if (avcodec_open2(codecCtx, codec, NULL) < 0) {
	        LOGI("%s", "无法打开解码器");
	        return;
	    }
	    //压缩数据
	    AVPacket *packet = (AVPacket *) av_malloc(sizeof(AVPacket));
	    //解压缩数据
	    AVFrame *frame = av_frame_alloc();
	    //frame->16bit 44100 PCM 统一音频采样格式与采样率
	    SwrContext *swrCtx = swr_alloc();
	
	    //重采样设置参数-------------start
	    //输入的采样格式
	    enum AVSampleFormat in_sample_fmt = codecCtx->sample_fmt;
	    //输出采样格式16bit PCM
	    enum AVSampleFormat out_sample_fmt = AV_SAMPLE_FMT_S16;
	    //输入采样率
	    int in_sample_rate = codecCtx->sample_rate;
	    //输出采样率
	    int out_sample_rate = in_sample_rate;
	    //获取输入的声道布局
	    //根据声道个数获取默认的声道布局（2个声道，默认立体声stereo）
	    //av_get_default_channel_layout(codecCtx->channels);
	    uint64_t in_ch_layout = codecCtx->channel_layout;
	    //输出的声道布局（立体声）
	    uint64_t out_ch_layout = AV_CH_LAYOUT_STEREO;
	
	    swr_alloc_set_opts(swrCtx,
	                       out_ch_layout, out_sample_fmt, out_sample_rate,
	                       in_ch_layout, in_sample_fmt, in_sample_rate,
	                       0, NULL);
	    swr_init(swrCtx);
	
	    //输出的声道个数
	    int out_channel_nb = av_get_channel_layout_nb_channels(out_ch_layout);
	
	    //重采样设置参数-------------end
	
	    //JNI begin------------------
	    //JasonPlayer
	    jclass player_class = (*env)->GetObjectClass(env, instance);
	
	    //AudioTrack对象
	    jmethodID create_audio_track_mid = (*env)->GetMethodID(env, player_class, "createAudioTrack",
	                                                           "(II)Landroid/media/AudioTrack;");
	    jobject audio_track = (*env)->CallObjectMethod(env, instance, create_audio_track_mid,
	                                                   out_sample_rate, out_channel_nb);
	
	    //调用AudioTrack.play方法
	    jclass audio_track_class = (*env)->GetObjectClass(env, audio_track);
	    jmethodID audio_track_play_mid = (*env)->GetMethodID(env, audio_track_class, "play", "()V");
	    jmethodID audio_track_stop_mid = (*env)->GetMethodID(env, audio_track_class, "stop", "()V");
	    (*env)->CallVoidMethod(env, audio_track, audio_track_play_mid);
	
	    //AudioTrack.write
	    jmethodID audio_track_write_mid = (*env)->GetMethodID(env, audio_track_class, "write",
	                                                          "([BII)I");
	    //JNI end------------------
	
	    //16bit 44100 PCM 数据
	    uint8_t *out_buffer = (uint8_t *) av_malloc(MAX_AUDIO_FRME_SIZE);
	
	    int got_frame = 0, index = 0, ret;
	    //不断读取压缩数据
	    while (av_read_frame(pFormatCtx, packet) >= 0) {
	        //解码音频类型的Packet
	        if (packet->stream_index == audio_stream_idx) {
	            //解码
	            ret = avcodec_decode_audio4(codecCtx, frame, &got_frame, packet);
	
	            if (ret < 0) {
	                LOGI("%s", "解码完成");
	            }
	            //解码一帧成功
	            if (got_frame > 0) {
	                LOGI("解码：%d", index++);
	                swr_convert(swrCtx, &out_buffer, MAX_AUDIO_FRME_SIZE,
	                            (const uint8_t **) frame->data, frame->nb_samples);
	                //获取sample的size
	                int out_buffer_size = av_samples_get_buffer_size(NULL, out_channel_nb,
	                                                                 frame->nb_samples, out_sample_fmt,
	                                                                 1);
	
	                //out_buffer缓冲区数据，转成byte数组
	                jbyteArray audio_sample_array = (*env)->NewByteArray(env, out_buffer_size);
	                jbyte *sample_bytep = (*env)->GetByteArrayElements(env, audio_sample_array, NULL);
	                //out_buffer的数据复制到sampe_bytep
	                memcpy(sample_bytep, out_buffer, out_buffer_size);
	                //同步
	                (*env)->ReleaseByteArrayElements(env, audio_sample_array, sample_bytep, 0);
	
	                //AudioTrack.write PCM数据
	                (*env)->CallIntMethod(env, audio_track, audio_track_write_mid,
	                                      audio_sample_array, 0, out_buffer_size);
	                //释放局部引用
	                (*env)->DeleteLocalRef(env, audio_sample_array);
	            }
	        }
	
	        av_free_packet(packet);
	    }
	
	    (*env)->CallVoidMethod(env, audio_track, audio_track_stop_mid);
	
	    av_frame_free(&frame);
	    av_free(out_buffer);
	
	    swr_free(&swrCtx);
	    avcodec_close(codecCtx);
	    avformat_close_input(&pFormatCtx);
	
	    (*env)->ReleaseStringUTFChars(env, input_, input);
	}

测试：

    @Override
    public void onClick(View v) {
        String inputVideo = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separatorChar
                + "input.mp4";

        String inputAudio = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separatorChar
                + "love.mp3";

        switch (v.getId()) {

            case R.id.btn_play_audio:

                mPlayer.play(inputVideo);
                break;

        }
    }

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
