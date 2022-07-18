####前言

先来了解一下视频直播的基本架构：

![直播基本架构](http://upload-images.jianshu.io/upload_images/2570030-e3565e21b756577e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们需要有一个主播客户端进行音视频采集，压缩，然后通过RTMP协议进行推流，推到流媒体服务器，然后其他客户端统一从流媒体服务器引流，播放。关于这里的过程的一些细节我将会在后续文章中慢慢道来。

####Linux（Ubuntu系统或者虚拟机）搭建流媒体服务器

先来了解一下俄罗斯人开发的Nginx服务器，Nginx ("engine x") 是一个高性能的HTTP和反向代理服务器，也是一个IMAP/POP3/SMTP服务器。Nginx是由Igor Sysoev为俄罗斯访问量第二的Rambler.ru站点开发的，第一个公开版本0.1.0发布于2004年10月4日。其将源代码以类BSD许可证的形式发布，因它的稳定性、丰富的功能集、示例配置文件和低系统资源的消耗而闻名。

因为Nginx服务器支持RTMP协议，因此这里我们作为流媒体服务器。

######Tips：实际开发可能是多台服务器，需要高并发架构支持。反向代理：外网的请求先转发到内网，然后返回的时候再转到外网。

Nginx是一种模块化的服务器，开源，我们可以自由添加删除我们自己的模块，不同的模块使用不同的端口号，互不冲突，如下图所示：

![模块化的Nginx服务器](http://upload-images.jianshu.io/upload_images/2570030-220d6eb81fa41aac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一、先下载安装  nginx 和 nginx-rtmp 编译依赖工具。

	sudo apt-get install build-essential libpcre3 libpcre3-dev libssl-dev

二、创建一个工作目录，并切换到工作目录。

因为后续可能还需要进行编译等操作，最好先给目录递归赋予权限：

	mkdir Live
	chmod +x -R Live/

三、下载 nginx 和 nginx-rtmp源码（wget是一个从网络上自动下载文件的自由工具）

	wget http://nginx.org/download/nginx-1.8.1.tar.gz
	wget https://github.com/arut/nginx-rtmp-module/archive/master.zip

四、解压

如果你用的是Ubuntu系统，直接通过界面操作解压即可。

安装unzip工具，解压下载的安装包
	sudo apt-get install unzip

5.解压 nginx 和 nginx-rtmp安装包

	tar -zxvf nginx-1.8.1.tar.gz
	unzip master.zip

其中：

	-zxvf分别是四个参数
	x : 从 tar 包中把文件提取出来
	z : 表示 tar 包是被 gzip 压缩过的，所以解压时需要用 gunzip 解压
	v : 显示详细信息
	f : xxx.tar.gz :  指定被处理的文件是 xxx.tar.gz

五、添加 nginx-rtmp 模板编译到 nginx

切换到 nginx-目录

	cd nginx-1.8.1
	./configure --with-http_ssl_module --add-module=../nginx-rtmp-module-master

六、编译、安装nginx

	编译nginx源码
	make
	安装需要超级权限
	sudo make install

######Tips：make、安装编译默认就是调用configure脚本进行编译安装，因此安装路径可以在configure找到。编译过程就是先生成目标文件.o，然后进行链接得到可执行程序。安装的过程就是把一些文件复制到系统目录里面去。

七、安装nginx init 脚本

	下载init脚本到/etc/init.d/nginx目录中，其中/etc/init.d目录放是Linux进程启动的时候会执行的一些脚本
	sudo wget https://raw.github.com/JasonGiedymin/nginx-init-ubuntu/master/nginx -O /etc/init.d/nginx
	给目录添加执行权限
	sudo chmod +x /etc/init.d/nginx
	刷新一下
	sudo update-rc.d nginx defaults

八、启动和停止nginx 服务，生成配置文件

	sudo service nginx start

这时候在浏览器输入http://127.0.0.1/，就可以看到页面，证明服务器已经成功安装好了。然后我们停止服务器，进行后续操作。

	sudo service nginx stop

![Nginx安装成功之后的首页](http://upload-images.jianshu.io/upload_images/2570030-2313833fef5c2efb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

九、安装 FFmpeg

这里FFmpeg是用于做音视频的编解码的。同时我们测试直播功能的时候，也可以通过ffplay进行测试，ffplay支持RTMP协议。

	cd ffmpeg-2.6.9

	编译FFmpeg
	./configure --disable-yasm
	安装FFmpeg（这个过程比较久，耐心等待）
	sudo make install

输入下面的命令测试是否安装好：

	输出安装信息
	ffmpeg -v

######Tips：服务器内存不足需要配置虚拟内存。

十、配置 nginx-rtmp 服务器

Nginx安装在/usr/local/nginx中，其中：

1. conf是配置目录
2. html是网页的目录

打开 /usr/local/nginx/conf/nginx.conf

在末尾添加如下配置：

	rtmp {
	    server {
	            listen 1935;
	            chunk_size 4096;
	
	            application live {
	                    live on;
	                    record off;
	                    exec ffmpeg -i rtmp://localhost/live/$name -threads 1 -c:v libx264 -profile:v baseline -b:v 350K -s 640x360 -f flv -c:a aac -ac 1 -strict -2 -b:a 56k rtmp://localhost/live360p/$name;
	            }
	            application live360p {
	                    live on;
	                    record off;
	        }
	    }
	}

主要是配置RTMP协议，Nginx是一种模块化的服务器，可以自由添加功能。这里主要是配置RTMP模块的一些参数，包括端口号，视频的编解码参数、格式等等。

保存上面配置文件，然后重新启动nginx服务：

	sudo service nginx restart

######Tips：Nginx的安装目录可以通过在configure查找“install”关键字找到。
######这里也可以看到http的配置，默认是80端口，如果有冲突的话，可以修改其他端口。不同功能使用不同端口，互不冲突。
######如果你使用了防火墙，请允许端口 tcp 1935。

####Windows平台下流媒体服务器搭建

由于Windows平台编译Nginx源码比较麻烦，因此我们用其他人编译好的版本即可，下载地址如下：

	https://github.com/illuspas/nginx-rtmp-win32

同理，我们需要配置nginx-rtmp-win32\conf\nginx.conf文件，然后点击根目录下面的nginx.exe即可打开服务器。

####流媒体服务器测试

引流有两个方法测试（将来也可以通过手机客户端进行测试）：

一、服务器配置测试播放器：将Flash播放器的所有文件复制到目录：/usr/local/nginx/html（Windows是www目录）/，然后修改播放地址

播放器下载地址：

	链接：http://pan.baidu.com/s/1bo9ePRp 密码：627r

######方法一需要把index.html的推流IP地址改为你自己的

二、用ffplay播放RTMP直播流：

在终端输入一下命令即可：

	ffplay rtmp://172.17.120.44/live/test

######方法二需要把命令中的推流IP地址改为你自己的，并且最好把FFmpeg添加到环境变量


推流的测试：

目前来说也是有两种方法（将来会添加Android端引流测试）：

一、使用之前创建的FFmpeg Visual项目，博客地址：

	http://www.jianshu.com/p/5b7c18285667

二、使用之前创建的FFmpeg Android Studio项目，相关博客地址：

	http://www.jianshu.com/p/91e07b7dc8ca
	http://www.jianshu.com/p/da140cffadba

其中，核心代码如下：

	JNIEXPORT void JNICALL
	Java_com_nan_ffmpeg_utils_FFmpegUtils_push(JNIEnv *env, jclass type, jstring input_,
	                                           jstring output_) {
	
	    const char *input = (*env)->GetStringUTFChars(env, input_, NULL);
	    const char *output = (*env)->GetStringUTFChars(env, output_, NULL);
	    LOGI("%s\n", input);
	    LOGI("%s\n", output);
	
	    //变量初始化
	    AVFormatContext *inFmtCtx = NULL, *outFmtCtx = NULL;
	    int ret;
	    //注册组件
	    av_register_all();
	    //初始化网络
	    avformat_network_init();
	
	    //打开输入文件
	    if ((ret = avformat_open_input(&inFmtCtx, input, 0, 0)) < 0) {
	        LOGE("无法打开文件");
	        goto end;
	    }
	    //获取文件信息
	    if ((ret = avformat_find_stream_info(inFmtCtx, 0)) < 0) {
	        LOGE("无法获取文件信息");
	        goto end;
	    }
	    //输出的封装格式上下文，使用RTMP协议推送flv封装格式的流
	    avformat_alloc_output_context2(&outFmtCtx, NULL, "flv", output); //RTMP
	    //avformat_alloc_output_context2(&ofmt_ctx, NULL, "mpegts", output_str);//UDP
	
	    int i = 0;
	    for (; i < inFmtCtx->nb_streams; i++) {
	        //根据输入封装格式中的AVStream流，来创建输出封装格式的AVStream流
	        //解码器，解码器上下文都要一致
	        AVStream *in_stream = inFmtCtx->streams[i];
	        AVStream *out_stream = avformat_new_stream(outFmtCtx, in_stream->codec->codec);
	        //复制解码器上下文
	        ret = avcodec_copy_context(out_stream->codec, in_stream->codec);
	        //全局头
	        out_stream->codec->codec_tag = 0;
	        if (outFmtCtx->oformat->flags == AVFMT_GLOBALHEADER) {
	            out_stream->codec->flags = CODEC_FLAG_GLOBAL_HEADER;
	        }
	    }
	
	    //打开输出的AVIOContext IO流上下文
	    AVOutputFormat *ofmt = outFmtCtx->oformat;
	    if (!(ofmt->flags & AVFMT_NOFILE)) {
	        ret = avio_open(&outFmtCtx->pb, output, AVIO_FLAG_WRITE);
	    }
	
	    //先写一个头
	    ret = avformat_write_header(outFmtCtx, NULL);
	    if (ret < 0) {
	        LOGE("推流发生错误\n");
	        goto end;
	    }
	    //获取视频流的索引位置
	    int videoindex = -1;
	    for (i = 0; i < inFmtCtx->nb_streams; i++) {
	        if (inFmtCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO) {
	            videoindex = i;
	            break;
	        }
	    }
	    int frame_index = 0;
	    int64_t start_time = av_gettime();
	    AVPacket pkt;
	    while (1) {
	        AVStream *in_stream, *out_stream;
	        //读取AVPacket
	        ret = av_read_frame(inFmtCtx, &pkt);
	        if (ret < 0)
	            break;
	        //没有封装格式的裸流（例如H.264裸流）是不包含PTS、DTS这些参数的。在发送这种数据的时候，需要自己计算并写入AVPacket的PTS，DTS，duration等参数
	        //PTS：Presentation Time Stamp。PTS主要用于度量解码后的视频帧什么时候被显示出来
	        //DTS：Decode Time Stamp。DTS主要是标识读入内存中的流在什么时候开始送入解码器中进行解码
	        if (pkt.pts == AV_NOPTS_VALUE) {
	            //Write PTS
	            AVRational time_base1 = inFmtCtx->streams[videoindex]->time_base;
	            //Duration between 2 frames (us)
	            int64_t calc_duration =
	                    (double) AV_TIME_BASE / av_q2d(inFmtCtx->streams[videoindex]->r_frame_rate);
	            //Parameters
	            pkt.pts = (double) (frame_index * calc_duration) /
	                      (double) (av_q2d(time_base1) * AV_TIME_BASE);
	            pkt.dts = pkt.pts;
	            pkt.duration = (double) calc_duration / (double) (av_q2d(time_base1) * AV_TIME_BASE);
	        }
	
	        if (pkt.stream_index == videoindex) {
	            //FFmpeg处理数据速度很快，瞬间就能把所有的数据发送出去，流媒体服务器是接受不了
	            //这里采用av_usleep()函数休眠的方式来延迟发送，延时时间根据帧率与时间基准计算得到
	            AVRational time_base = inFmtCtx->streams[videoindex]->time_base;
	            AVRational time_base_q = {1, AV_TIME_BASE};
	            int64_t pts_time = av_rescale_q(pkt.dts, time_base, time_base_q);
	            int64_t now_time = av_gettime() - start_time;
	            if (pts_time > now_time) {
	                av_usleep(pts_time - now_time);
	            }
	        }
	
	        in_stream = inFmtCtx->streams[pkt.stream_index];
	        out_stream = outFmtCtx->streams[pkt.stream_index];
	        /* copy packet */
	        //Convert PTS/DTS
	        pkt.pts = av_rescale_q_rnd(pkt.pts, in_stream->time_base, out_stream->time_base,
	                                   AV_ROUND_NEAR_INF | AV_ROUND_PASS_MINMAX);
	        pkt.dts = av_rescale_q_rnd(pkt.dts, in_stream->time_base, out_stream->time_base,
	                                   AV_ROUND_NEAR_INF | AV_ROUND_PASS_MINMAX);
	        pkt.duration = av_rescale_q(pkt.duration, in_stream->time_base, out_stream->time_base);
	        pkt.pos = -1;
	        //输出进度
	        if (pkt.stream_index == videoindex) {
	            LOGI("第%d帧", frame_index);
	            frame_index++;
	        }
	        //推送
	        ret = av_interleaved_write_frame(outFmtCtx, &pkt);
	
	        if (ret < 0) {
	            LOGE("Error muxing packet");
	            break;
	        }
	        av_free_packet(&pkt);
	
	    }
	    //输出结尾
	    av_write_trailer(outFmtCtx);
	    end:
	    //释放资源
	    avformat_free_context(inFmtCtx);
	    avio_close(outFmtCtx->pb);
	    avformat_free_context(outFmtCtx);
	
	    (*env)->ReleaseStringUTFChars(env, input_, input);
	    (*env)->ReleaseStringUTFChars(env, output_, output);
	}

注意：

1. 如果是Android Studio项目进行测试的话，记得添加网络、SDCard访问权限。
2. 如果是Visual Studio项目测试，只需要把上面的JNI方法修改成main函数即可。
3. 代码先不作解释。

![测试结果](http://upload-images.jianshu.io/upload_images/2570030-265fd519deec17af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
