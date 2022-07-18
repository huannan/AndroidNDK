###前言

有关APK更新的技术比较多，例如：增量更新，插件式开发，热修复，RN
静默安装。下面简单介绍一下：

1. 增量更新：主要是更新宿主APK。主要技术是C/C++，本文中会一一详细介绍。
2. 插件式开发：给宿主APK提供插件，扩展（需要的时候再下载），可以动态地替换。主要技术是动态代理的知识。
3. 热修复：通过NDK底层去修复，也是C/C++的技术。
4. RN：通过JS脚本去修复APK。
5.静默安装：很麻烦，需要root权限，适配不同手机ROM很麻烦。

###增量更新的流程

1. APP检测最新版本：把当前版本告诉服务端，服务端进行判断。
2. 如果有新版本，服务端需要对当前版本的APK与最新版本的APK进行一次差分，产生patch差分文件。（或者新版本的APK上传到服务端的时候就已经差分好了）
3. APP在后台下载差分文件，进行文件的MD5校验，在本地进行合并（跟本地的data目录下面的APK文件合并），合并出最新的APK之后，提示用户安装。

增量更新的最终目的：省流量地更新宿主APK。

######差分的处理比较麻烦的地方就是要针对不同的应用市场渠道和众多不同版本进行差分。

######注意：新版本有可能比旧版本小，差分只是把变化的部分记录下来。


###增量更新开发流程

####使用到的第三方库

我们使用到的第三方库是：Binary diff，简称bsdiff，这个库专门用来实现文件的差分和合并的，它的官网如下：

[http://www.daemonology.net/bsdiff/](http://www.daemonology.net/bsdiff/ "bsdiff")

![bsdiff.png](http://upload-images.jianshu.io/upload_images/2570030-a5ff63f9488429f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这里我们可以点击文中的"here"下载源码，这是Linux源码。也可以下载Windows版本的源码，点击"Windows port"。

这个库引用了bzip2这个库，官网如下：

[http://www.bzip.org/](http://www.bzip.org/ "bzip2")

####编译第三方库源码

由于第三方库都是用C/C++写的，因此我们需要在VS2013下面操作。

我们把下载好的bsdiff源码的头文件以及源文件拷贝到一个新创建的VS空工程里面，并且通过添加现有项添加进来。

然后我们暂时不需要合并，因此把bspatch.cpp移除。

然后编译的时候会发现很多问题，例如：

使用了不安全的函数：

	1>d:\ndk\bsdiff\bsdiff\bzlib.c(1416): error C4996: 'strcat': This function or variable may be unsafe. Consider using strcat_s instead. To disable deprecation, use _CRT_SECURE_NO_WARNINGS. See online help for details.
	1>          e:\program files (x86)\microsoft visual studio 12.0\vc\include\string.h(118) : 参见“strcat”的声明

函数过时：

	1>d:\ndk\bsdiff\bsdiff\bzlib.c(1422): error C4996: 'setmode': The POSIX name for this item is deprecated. Instead, use the ISO C++ conformant name: _setmode. See online help for details.
	1>          e:\program files (x86)\microsoft visual studio 12.0\vc\include\io.h(338) : 参见“setmode”的声明

上面这两个错误都可以通过添加宏定义进行解决，这里为了方便，我们直接在预编译的宏定义里面添加：

	-D _CRT_SECURE_NO_WARNINGS -D _CRT_NONSTDC_NO_DEPRECATE

如下图所示：

![添加宏定义.png](http://upload-images.jianshu.io/upload_images/2570030-ae51872a558a840e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 


使用了可能未初始化的本地指针变量，因为外国人比较喜欢用苹果电脑开发，所以做过多的语法检查。

	未初始化
	1>d:\ndk\bsdiff\bsdiff\bsdiff.cpp(272): error C4703: 使用了可能未初始化的本地指针变量“old”
	1>d:\ndk\bsdiff\bsdiff\bsdiff.cpp(279): error C4703: 使用了可能未初始化的本地指针变量“V”
	1>d:\ndk\bsdiff\bsdiff\bsdiff.cpp(301): error C4703: 使用了可能未初始化的本地指针变量“_new”

解决方法是关闭SDL检查：

![关闭SDL检查.png](http://upload-images.jianshu.io/upload_images/2570030-2016b1fcd2fa46d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

另外，可能报头文件找不到的错误，这有可能是编码问题，因为外国人使用的苹果电脑跟Windows电脑的编译不一致产生的。可以通过Notepad++的转码功能进行转码，全部转为UTF-8无BOM格式编码即可，Windows、Linux通用的。

我们项目属性里面的生成配置里面选择DLL，并且修改解决方案为你的电脑的对应平台，然后编译，生成DLL动态库文件。

####Java代码调用

创建Web项目，用来做APP的服务端。创建工具类专门用于产生差分包：

	package com.nan.diff;
	
	public class DiffUtil {
	
		static{
			System.loadLibrary("bsdiff");
		}
		
		// 产生APK差分包
		public static native void diff(String oldPath, String newPath, String patchPath);
	
	}

其中JNI的实现如下（JNI的使用不再赘述）：

	JNIEXPORT void JNICALL Java_com_nan_diff_DiffUtil_diff
	(JNIEnv *env, jclass clz, jstring oldPath_, jstring newPath_, jstring patchPath_){
	
		char* oldPath = (char*)env->GetStringUTFChars(oldPath_, NULL);
		char* newPath = (char*)env->GetStringUTFChars(newPath_, NULL);
		char* patchPath = (char*)env->GetStringUTFChars(patchPath_, NULL);
	
		char *argv[4];
		argv[0] = "bsdiff";
		argv[1] = oldPath;
		argv[2] = newPath;
		argv[3] = patchPath;
		bsdiff_main(4, argv);
	
		env->ReleaseStringUTFChars(oldPath_, oldPath);
		env->ReleaseStringUTFChars(newPath_, newPath);
		env->ReleaseStringUTFChars(patchPath_, patchPath);
	}

通过研究bsdiff的源码，我们发现bsdiff.cpp里面的main函数就是入口函数，我们把main函数改为bsdiff_main，然后通过JNI去调用。

参数分别是：

		argv[0] = "bsdiff";//这个参数没用
		argv[1] = oldPath;//旧APK文件路径
		argv[2] = newPath;/新APK文件路径
		argv[3] = patchPath;//APK差分文件路径


然后我们准备两个APK文件，不同版本的，最好Java代码、资源都不一样。

我们写一个测试类测试：

	public class Test {
	
		public static final String OLD_APK_PATH = "D:\\WebPrj\\apk\\app_old.apk";
		public static final String NEW_APK_PATH = "D:\\WebPrj\\apk\\app_new.apk";
		public static final String PATCH_APK_PATH = "D:\\WebPrj\\apk\\apk.patch";
		
		public static void main(String[] args) {
			
			DiffUtil.diff(OLD_APK_PATH, NEW_APK_PATH, PATCH_APK_PATH);
			System.out.println("差分包产生完毕");
			
		}
	}

生成结果如下图所示：

![结果.png](http://upload-images.jianshu.io/upload_images/2570030-12cef27ba49c1514.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####Android端差分文件的下载与合并

在上一节中，我们将生成的apk.patch文件放到Java Web项目的WebContent目录里面来，供android这边下载。

在Android端，我们需要把bzip2以及bsdiff的文件拷贝到jni目录里面，同样的，我们只需要编译一个bspatch.c源文件即可。

由于Android手机本来就是Linux系统，因此我们直接使用bsdiff的Linux版本的库即可。

跟上面一样，我们把main函数改为bspatch_main，并且提供JNI调用：

	JNIEXPORT void JNICALL
	Java_com_nan_diff_utils_BsPatch_patch(JNIEnv *env, jclass type, jstring oldFile_, jstring newFile_,
	                                      jstring patchFile_) {
	    char *oldFile = (char *) (*env)->GetStringUTFChars(env, oldFile_, NULL);
	    char *newFile = (char *) (*env)->GetStringUTFChars(env, newFile_, NULL);
	    char *patchFile = (char *) (*env)->GetStringUTFChars(env, patchFile_, NULL);
	
	    char *argv[4];
	    argv[0] = "bspatch";
	    argv[1] = oldFile;
	    argv[2] = newFile;
	    argv[3] = patchFile;
	    bspatch_main(4, argv);
	
	    (*env)->ReleaseStringUTFChars(env, oldFile_, oldFile);
	    (*env)->ReleaseStringUTFChars(env, newFile_, newFile);
	    (*env)->ReleaseStringUTFChars(env, patchFile_, patchFile);
	}

对应的Java代码如下：

	package com.nan.diff.utils;
	
	public class BsPatch {
	
		//APK合并
		public native static void patch(String oldFile, String newFile, String patchFile);
	
		static{
			System.loadLibrary("bspatch");
		}
	}


最后我们的MainActivity如下：

	public class MainActivity extends AppCompatActivity {
	
	    public static final int CODE_REQUEST_PERMISSIONS = 100;
	    private Context mCtx;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	
	        setContentView(R.layout.activity_main);
	
	        mCtx = this;
	
	        //申请文件读写、创建删除文件权限
	        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
	            requestPermissions(new String[]{Manifest.permission.READ_EXTERNAL_STORAGE,
	                    Manifest.permission.WRITE_EXTERNAL_STORAGE,
	                    Manifest.permission.MOUNT_UNMOUNT_FILESYSTEMS
	            }, CODE_REQUEST_PERMISSIONS);
	        }
	
	        findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
	            @Override
	            public void onClick(View v) {
	                Toast.makeText(MainActivity.this, "这是版本1", Toast.LENGTH_SHORT).show();
	                update();
	            }
	        });
	    }
	
	    private void update() {
	
	        Observable.create(new ObservableOnSubscribe<String>() {
	            @Override
	            public void subscribe(ObservableEmitter<String> e) throws Exception {
	                //下载差分包
	                //DownloadUtils.download(URL_PATCH_DOWNLOAD);
	                String oldFile = ApkUtils.getSourceApkPath(mCtx, PACKAGE_NAME);
	                //合并差分包
	                BsPatch.patch(oldFile, NEW_APK_PATH, PATCH_FILE_PATH);
	                e.onNext(NEW_APK_PATH);
	                Toast.makeText(MainActivity.this, "差分包合成完毕", Toast.LENGTH_SHORT).show();
	            }
	        })
	        .subscribeOn(Schedulers.io())
	        .observeOn(AndroidSchedulers.mainThread())
	        .subscribe(new Consumer<String>() {
	            @Override
	            public void accept(@NonNull String newApkPath) throws Exception {
	                Toast.makeText(MainActivity.this, "开始安装", Toast.LENGTH_SHORT).show();
	                ApkUtils.installApk(mCtx, newApkPath);
	            }
	        });
	    }
	}

主要的逻辑在update方法中，我们先下载差分包，然后在本地合成，最后提示用户安装。

为了达到明显的效果，两个版本可以增删一些资源文件、修改Java代码、布局文件等。


源码：https://github.com/huannan/Diff

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
