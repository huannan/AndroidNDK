####前言

![直播APP](https://upload-images.jianshu.io/upload_images/2570030-5467ee60e90d8abd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们主要是实现RTMP推流，引流的部分通过一些直播RTMP协议的播放器来实现。

因为项目比较大，设计的知识也比较广，例如h264编码标准，aac编码，RTMP协议。这里我们只概述一些关键的核心逻辑与实现思路，具体的实现可以参考源代码，因为工作原因源代码晚点再上。

####推流的流程

主要分为以下几个步骤：

1. 调用Java的相关API进行音视频的采集。
2. 初始化一些C相关的库，然后用户点击开始推流。
3. 因为摄像头、麦克风采集到的数据是原始的数据，需要经过编码。其中，视频编码使用h264编码格式，对应x264库；音频编码使用aac编码，使用faac库。
4. 使用rtmpdump（librtmp）库进行推流。

下面我们一一进行介绍。

####一、调用Java的相关API进行音视频的采集

#####视频采集

使用采集主要调用Camera的相关API，核心代码如下：

打开摄像头，初始化一些信息，开始预览（预览需要一个SurfaceView的Holder）。

如果需要实时获取摄像头采集数据的时候，还需要调用addCallbackBuffer设置缓冲区，然后添加Callback。

	try {
		//SurfaceView初始化完成，可以进行预览
		mCamera = Camera.open(mVideoParams.getCameraId());
		Camera.Parameters param = mCamera.getParameters();
		//设置预览图像的像素格式为NV-21
		param.setPreviewFormat(ImageFormat.NV21);
		//设置预览画面宽高
		param.setPreviewSize(mVideoParams.getWidth(), mVideoParams.getHeight());
		//设置预览帧频，但是x264压缩的时候还是有另外一个帧频的
		//param.setPreviewFpsRange(mVideoParams.getFps() - 1, mVideoParams.getFps());
		mCamera.setParameters(param);
	
		mCamera.setPreviewDisplay(mSurfaceHolder);
		
		//如果是正在直播的话需要实时获取预览图像数据
		//缓冲区，大小需要根据摄像头的分辨率而定，x4换算为字节
		buffers = new byte[mVideoParams.getWidth() * mVideoParams.getHeight() * 4];
		mCamera.addCallbackBuffer(buffers);
		mCamera.setPreviewCallbackWithBuffer(this);
	
		//开始预览
		mCamera.startPreview();
	} catch (Exception e) {
		e.printStackTrace();
	}

Callback的实现核心逻辑如下：

获取摄像头数据data，传到Native层，然后由Native层负责h264编码并且推流。

    @Override
    public void onPreviewFrame(byte[] data, Camera camera) {
        if (mCamera != null) {
            mCamera.addCallbackBuffer(buffers);
        }

        if (isPushing) {
            mPushNative.fireVideo(data);
        }
    }

#####音频采集

初始化一个AudioRecord：

    int channelConfig = audioParams.getChannel() == 1 ?
            AudioFormat.CHANNEL_IN_MONO : AudioFormat.CHANNEL_IN_STEREO;
    //最小缓冲区大小
    minBufferSize = AudioRecord.getMinBufferSize(audioParams.getSampleRateInHz(), channelConfig, AudioFormat.ENCODING_PCM_16BIT);
    mAudioRecord = new AudioRecord(MediaRecorder.AudioSource.MIC,
            audioParams.getSampleRateInHz(),
            channelConfig,
            AudioFormat.ENCODING_PCM_16BIT, minBufferSize);

开启一个音频的录制线程进行录制，并且实时发送到Native层，由Native层进行aac编码并且推流。

线程的核心逻辑如下：

    @Override
    public void run() {
        //开始录音
        mAudioRecord.startRecording();

        while (isPushing) {
            //通过AudioRecord不断读取音频数据
            byte[] buffer = new byte[minBufferSize];
            int len = mAudioRecord.read(buffer, 0, buffer.length);
            if (len > 0) {
                //传给Native代码，进行音频编码
                mPushNative.fireAudio(buffer, len);
            }
        }
    }

####二、初始化一些C相关的库，然后用户点击开始推流

编译faac、x264、rtmpdump库，输出静态库.a文件，添加到Android Studio里面。

其中x264初始化的核心代码如下：

	JNIEXPORT void JNICALL
	Java_com_nan_live_pusher_PushNative_fireAudio(JNIEnv *env, jobject instance, jbyteArray buffer_,
	                                              jint length) {
	
	    int *pcmbuf;
	    unsigned char *bitbuf;
	    jbyte *b_buffer = (*env)->GetByteArrayElements(env, buffer_, 0);
	    pcmbuf = (short *) malloc(nInputSamples * sizeof(int));
	    bitbuf = (unsigned char *) malloc(nMaxOutputBytes * sizeof(unsigned char));
	    int nByteCount = 0;
	    unsigned int nBufferSize = (unsigned int) length / 2;
	    unsigned short *buf = (unsigned short *) b_buffer;
	    while (nByteCount < nBufferSize) {
	        int audioLength = nInputSamples;
	        if ((nByteCount + nInputSamples) >= nBufferSize) {
	            audioLength = nBufferSize - nByteCount;
	        }
	        int i;
	        for (i = 0; i < audioLength; i++) {//每次从实时的pcm音频队列中读出量化位数为8的pcm数据。
	            int s = ((int16_t *) buf + nByteCount)[i];
	            pcmbuf[i] = s << 8;//用8个二进制位来表示一个采样量化点（模数转换）
	        }
	        nByteCount += nInputSamples;
	        //利用FAAC进行编码，pcmbuf为转换后的pcm流数据，audioLength为调用faacEncOpen时得到的输入采样数，bitbuf为编码后的数据buff，nMaxOutputBytes为调用faacEncOpen时得到的最大输出字节数
	        int byteslen = faacEncEncode(audio_encode_handle, pcmbuf, audioLength,
	                                     bitbuf, nMaxOutputBytes);
	        if (byteslen < 1) {
	            continue;
	        }
	        add_aac_body(bitbuf, byteslen);//从bitbuf中得到编码后的aac数据流，放到数据队列
	    }
	    if (bitbuf)
	        free(bitbuf);
	    if (pcmbuf)
	        free(pcmbuf);
	
	    (*env)->ReleaseByteArrayElements(env, buffer_, b_buffer, 0);
	}

主要就是一下几个步骤：

1. x264_param_default_preset 设置
2. x264_param_apply_profile 设置档次
3. x264_picture_alloc（x264_picture_t输入图像）初始化
4. x264_encoder_open 打开编码器
5. x264_encoder_encode 编码
6. x264_encoder_close( h ) 关闭编码器，释放资源

初始化音频编码的代码如下：

	JNIEXPORT void JNICALL
	Java_com_nan_live_pusher_PushNative_setAudioOptions(JNIEnv *env, jobject instance,
	                                                    jint sampleRateInHz, jint channel) {
	
	    audio_encode_handle = faacEncOpen(sampleRateInHz, channel, &nInputSamples,
	                                      &nMaxOutputBytes);
	    if (!audio_encode_handle) {
	        LOGE("音频编码器打开失败");
	        return;
	    }
	    //设置音频编码参数
	    faacEncConfigurationPtr p_config = faacEncGetCurrentConfiguration(audio_encode_handle);
	    p_config->mpegVersion = MPEG4;
	    p_config->allowMidside = 1;
	    p_config->aacObjectType = LOW;
	    p_config->outputFormat = 0; //输出是否包含ADTS头
	    p_config->useTns = 1; //时域噪音控制,大概就是消爆音
	    p_config->useLfe = 0;
	//	p_config->inputFormat = FAAC_INPUT_16BIT;
	    p_config->quantqual = 100;
	    p_config->bandWidth = 0; //频宽
	    p_config->shortctl = SHORTCTL_NORMAL;
	
	    if (!faacEncSetConfiguration(audio_encode_handle, p_config)) {
	        LOGE("%s", "音频编码器配置失败..");
	        throwNativeError(env, INIT_FAILED);
	        return;
	    }
	
	    LOGI("%s", "音频编码器配置成功");
	
	}

三、进行音视频编码与推流

主要调用x264_encoder_encode方法进行视频编码，然后通过add_264_sequence_header方法添加RTMP头信息，通过add_264_body添加RTMP body。

	JNIEXPORT void JNICALL
	Java_com_nan_live_pusher_PushNative_fireVideo(JNIEnv *env, jobject instance, jbyteArray buffer_) {
	
	    //视频数据转为YUV420P
	    //NV21->YUV420P
	    jbyte *nv21_buffer = (*env)->GetByteArrayElements(env, buffer_, NULL);
	    jbyte *u = pic_in.img.plane[1];
	    jbyte *v = pic_in.img.plane[2];
	    //nv21 4:2:0 Formats, 12 Bits per Pixel
	    //nv21与yuv420p，y个数一致，uv位置对调
	    //nv21转yuv420p  y = w*h,u/v=w*h/4
	    //nv21 = yvu yuv420p=yuv y=y u=y+1+1 v=y+1
	    //如果要进行图像处理（美颜），可以再转换为RGB
	    //还可以结合OpenCV识别人脸等等
	    memcpy(pic_in.img.plane[0], nv21_buffer, y_len);
	    int i;
	    for (i = 0; i < u_len; i++) {
	        *(u + i) = *(nv21_buffer + y_len + i * 2 + 1);
	        *(v + i) = *(nv21_buffer + y_len + i * 2);
	    }
	
	    //h264编码得到NALU数组
	    x264_nal_t *nal = NULL; //NAL
	    int n_nal = -1; //NALU的个数
	    //进行h264编码
	    if (x264_encoder_encode(video_encode_handle, &nal, &n_nal, &pic_in, &pic_out) < 0) {
	        LOGE("%s", "编码失败");
	        return;
	    }
	    //使用rtmp协议将h264编码的视频数据发送给流媒体服务器
	    //帧分为关键帧和普通帧，为了提高画面的纠错率，关键帧应包含SPS和PPS数据
	    int sps_len, pps_len;
	    unsigned char sps[100];
	    unsigned char pps[100];
	    memset(sps, 0, 100);
	    memset(pps, 0, 100);
	    pic_in.i_pts += 1; //顺序累加
	    //遍历NALU数组，根据NALU的类型判断
	    for (i = 0; i < n_nal; i++) {
	        if (nal[i].i_type == NAL_SPS) {
	            //复制SPS数据
	            sps_len = nal[i].i_payload - 4;
	            memcpy(sps, nal[i].p_payload + 4, sps_len); //不复制四字节起始码
	        } else if (nal[i].i_type == NAL_PPS) {
	            //复制PPS数据
	            pps_len = nal[i].i_payload - 4;
	            memcpy(pps, nal[i].p_payload + 4, pps_len); //不复制四字节起始码
	
	            //发送序列信息
	            //h264关键帧会包含SPS和PPS数据
	            add_264_sequence_header(pps, sps, pps_len, sps_len);
	
	        } else {
	            //发送帧信息
	            add_264_body(nal[i].p_payload, nal[i].i_payload);
	        }
	
	    }
	
	    (*env)->ReleaseByteArrayElements(env, buffer_, nv21_buffer, 0);
	}

同理，调用faacEncEncode进行音频编码，然后发送RTMP信息。

	JNIEXPORT void JNICALL
	Java_com_nan_live_pusher_PushNative_fireAudio(JNIEnv *env, jobject instance, jbyteArray buffer_,
	                                              jint length) {
	
	    int *pcmbuf;
	    unsigned char *bitbuf;
	    jbyte *b_buffer = (*env)->GetByteArrayElements(env, buffer_, 0);
	    pcmbuf = (short *) malloc(nInputSamples * sizeof(int));
	    bitbuf = (unsigned char *) malloc(nMaxOutputBytes * sizeof(unsigned char));
	    int nByteCount = 0;
	    unsigned int nBufferSize = (unsigned int) length / 2;
	    unsigned short *buf = (unsigned short *) b_buffer;
	    while (nByteCount < nBufferSize) {
	        int audioLength = nInputSamples;
	        if ((nByteCount + nInputSamples) >= nBufferSize) {
	            audioLength = nBufferSize - nByteCount;
	        }
	        int i;
	        for (i = 0; i < audioLength; i++) {//每次从实时的pcm音频队列中读出量化位数为8的pcm数据。
	            int s = ((int16_t *) buf + nByteCount)[i];
	            pcmbuf[i] = s << 8;//用8个二进制位来表示一个采样量化点（模数转换）
	        }
	        nByteCount += nInputSamples;
	        //利用FAAC进行编码，pcmbuf为转换后的pcm流数据，audioLength为调用faacEncOpen时得到的输入采样数，bitbuf为编码后的数据buff，nMaxOutputBytes为调用faacEncOpen时得到的最大输出字节数
	        int byteslen = faacEncEncode(audio_encode_handle, pcmbuf, audioLength,
	                                     bitbuf, nMaxOutputBytes);
	        if (byteslen < 1) {
	            continue;
	        }
	        add_aac_body(bitbuf, byteslen);//从bitbuf中得到编码后的aac数据流，放到数据队列
	    }
	    if (bitbuf)
	        free(bitbuf);
	    if (pcmbuf)
	        free(pcmbuf);
	
	    (*env)->ReleaseByteArrayElements(env, buffer_, b_buffer, 0);
	}

进行RTMP推流的时候，需要使用生产者消费者的线程模型，编码属于生产者，推流属于消费者。并且需要一个双向链表进行数据的进出。

	void *push_thread(void *arg) {
	    JNIEnv *env;//获取当前线程JNIEnv
	    (*javaVM)->AttachCurrentThread(javaVM, &env, NULL);
	
	    //建立RTMP连接
	    RTMP *rtmp = RTMP_Alloc();
	    if (!rtmp) {
	        LOGE("rtmp初始化失败");
	        goto end;
	    }
	    RTMP_Init(rtmp);
	    rtmp->Link.timeout = 5; //连接超时的时间
	    //设置流媒体地址
	    RTMP_SetupURL(rtmp, rtmp_path);
	    //发布rtmp数据流
	    RTMP_EnableWrite(rtmp);
	    //建立连接
	    if (!RTMP_Connect(rtmp, NULL)) {
	        LOGE("%s", "RTMP 连接失败");
	        throwNativeError(env, CONNECT_FAILED);
	        goto end;
	    }
	    //计时
	    start_time = RTMP_GetTime();
	    if (!RTMP_ConnectStream(rtmp, 0)) { //连接流
	        LOGE("%s", "RTMP ConnectStream failed");
	        throwNativeError(env, CONNECT_FAILED);
	        goto end;
	    }
	    is_pushing = TRUE;
	    //发送AAC头信息
	    add_aac_sequence_header();
	
	    while (is_pushing) {
	        //发送
	        pthread_mutex_lock(&mutex);
	        pthread_cond_wait(&cond, &mutex);
	        //取出队列中的RTMPPacket
	        RTMPPacket *packet = queue_get_first();
	        if (packet) {
	            queue_delete_first(); //移除
	            packet->m_nInfoField2 = rtmp->m_stream_id; //RTMP协议，stream_id数据
	            int i = RTMP_SendPacket(rtmp, packet, TRUE); //TRUE放入librtmp队列中，并不是立即发送
	            if (!i) {
	                LOGE("RTMP 断开");
	                RTMPPacket_Free(packet);
	                pthread_mutex_unlock(&mutex);
	                goto end;
	            } else {
	                LOGI("%s", "rtmp send packet");
	            }
	            RTMPPacket_Free(packet);
	        }
	
	        pthread_mutex_unlock(&mutex);
	    }
	    end:
	    LOGI("%s", "释放资源");
	    free(rtmp_path);
	    RTMP_Close(rtmp);
	    RTMP_Free(rtmp);
	    (*javaVM)->DetachCurrentThread(javaVM);
	    return 0;
	}

	void add_rtmp_packet(RTMPPacket *packet) {
	    pthread_mutex_lock(&mutex);
	    if (is_pushing) {
	        queue_append_last(packet);
	    }
	    pthread_cond_signal(&cond);
	    pthread_mutex_unlock(&mutex);
	}

####RTMP引流

最后，需要进行RTMP引流，我们直接使用Vitamio播放器即可。

相关的核心代码如下：

    video_live = (VideoView) findViewById(R.id.video_live);

	//RTMP地址
    String rtmpUrl = PreferenceUtils.getInstance(this).getRTMPUrl();
    video_live.setVideoPath(rtmpUrl);
    video_live.setMediaController(new MediaController(this));
    video_live.requestFocus();

    video_live.setOnPreparedListener(new MediaPlayer.OnPreparedListener() {
        @Override
        public void onPrepared(MediaPlayer mediaPlayer) {
            mediaPlayer.setPlaybackSpeed(1.0f);
        }
    });


相关源代码：https://github.com/huannan/Live

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
