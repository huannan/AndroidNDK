###前言

热修复也叫热更新，又叫做动态加载、动态修复、动态更新，是指不通过重新安装新的APK安装包的情况下修复一些线上的BUG。

通过这样做，可以免去发版、安装、重新打开等过程，就可以修复线上的BUG，防止用户流失。因此这是几乎每一个APP都需要的一个功能，因此很有学习的必要。

需要注意的是：热修复只是临时的亡羊补牢。在企业中真正的热修复发版与正式版一样，需要测试进行测试。但是热修复也存在一些兼容性问题。因此高质量的版本与热修复框架才是解决问题的最好的手段。

###各大热修复框架的对比图

目前流行的热修复技术有：

* QQ空间的超级补丁方案
* 微信的Tinker
* 阿里巴巴的AndFix、Dexposed
* 美团的Robust
* 饿了么的MiGo
* 百度的HotFix

下面给出一些常见的热修复框架的对比图：

![热修复框架对比.png](http://upload-images.jianshu.io/upload_images/2570030-8bbfd2c70ec4c057.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看出几乎每一个框架都有优势和弊端。其中“即时生效”的意思就是是否能不通过重启来达到修复的效果，就像AndFix，支持即时生效，但是只能做到方法的替换，而不是替换（新增）类、资源等。选择什么框架，还是需要根据APP或者BUG的实际情况出发。

关于更多的热修复框架的对比，可以参考一些网上的文章。今天我们主要分析的是阿里巴巴的AndFix。

###技术选型

既然有这么多的技术，那么我们必将面临技术选型的问题，因此这里给出一些技术选型上的参考标准：

* 框架是否符合项目的需求，需求是衡量一切的标准
* 能够满足需求的前提下，选择学习成本最低的（同时也代表着使用简单、维护起来简单）
* 学习成本差不多的情况下，优先选择大公司的框架

###AndFix的基本使用

####AndFix的引入

首先我们在gradle脚本中添加AndFix的依赖：

	compile 'com.alipay.euler:andfix:0.5.0@aar'

由于热修复需要读写SD卡，因此需要添加一些权限，注意6.0的权限适配问题。如果你的补丁文件是从服务器下载的，那么就需要联网权限。

    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.INTERNET" />

####初始化

自定义一个Application，初始化AndFix，这里为了方便演示，在加载Patch文件的时候，我们省略了校验的步骤：

    public class App extends Application {

        private static final String TAG = App.class.getSimpleName();

		//Patch文件的路径
        private static final String APATCH_PATH = "/out.apatch";

        @Override
        public void onCreate() {
            super.onCreate();

            //初始化PatchManager
            PatchManager mPatchManager = new PatchManager(this);
            mPatchManager.init("1.0");

            //加载已加载过的Patch文件
            mPatchManager.loadPatch();
            //添加外部Patch文件
            try {
                // .apatch file path
                String patchFileString = Environment.getExternalStorageDirectory().getAbsolutePath() + APATCH_PATH;
                mPatchManager.addPatch(patchFileString);
                Log.d(TAG, "加载成功：" + patchFileString);
            } catch (Exception e) {
                Log.e(TAG, "", e);
            }
        }
    }

####AndFix的示例

然后我们写一个有BUG的Test类：

    public class Test {

        public static int test() {
            return 1 / 0;
        }

    }

在Activity中调用这个类：

    findViewById(R.id.btn_test).setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            String res = Test.test() + "";
            Toast.makeText(MainActivity.this, res, Toast.LENGTH_SHORT).show();
        }
    });

把当前的代码签名打包成一个APK，我们修改名字问old.apk。

然后把Test的BUG修改，再次签名打包成一个APK，我们修改名字问new.apk。

    public class Test {

        public static int test() {
            return 1 / 1;
        }

    }

####下载生成Patch文件的工具、生成Patch文件

然后我们去AndFix的[官网](https://github.com/alibaba/AndFix)下载生成Patch文件的工具：

	//Windows电脑用
	apkpatch.bat
    //Linux、苹果电脑用
    apkpatch.sh

然后我们把刚刚生产的两个APK文件、签名文件放到同一个目录。由于笔者使用的是Ubuntu系统，因此需要给apkpatch.sh添加执行的权限，Ubuntu下，签名文件的格式是jks。

然后执行下面的命令，生产Patch文件：

	./apkpatch.sh -f new.apk -t old.apk -o out -k nan.jks -p 123456 -a nan -e 123456

在命令里面我们执行了新旧两个APK文件，输出路径，签名文件，签名密码，签名文件的别名以及密码。

最终我们输出一个文件：

	new-e726f4396cbed42d1cf50fb2d781c9d9.apatch

我们修改名字为out.apatch，放到手机的SD卡根目录下面。

####效果

一开始如果没有放入out.apatch的时候，APP运行的时候是直接因为BUG而崩溃的，但是放入了out.apatch之后，APP的BUG被修复了。

###AndFix源码分析

####准备步骤

为了更方便地查看、分析AndFix的源码，我们将源码导入Android Studio中，由于AndFix源码是使用ndk-build进行构建的，为了更加方便导入，笔者根据Android.mk编写了一个CMake脚本，供读者参考：

	#配置CMake的最低版本
	cmake_minimum_required(VERSION 3.4.1)
	
	#配置工程路径
	#set(path_project /home/wuhuannan/AndroidPrj/AndFix)
	set(path_project D:/AndroidStudio/AndFix)
	#JNI文件夹的路径
	set(path_jni ${path_project}/jni)
	
	#配置头文件的包含路径
	include_directories(${path_project}/jni/art)
	include_directories(${path_project}/jni/dalvik)
	include_directories(${path_project}/jni)
	
	#添加自己写的库
	add_library(andfix
	            SHARED
	            ${path_jni}/andfix.cpp ${path_jni}/art/art_method_replace.cpp ${path_jni}/art/art_method_replace_4_4.cpp ${path_jni}/art/art_method_replace_5_0.cpp ${path_jni}/art/art_method_replace_5_1.cpp ${path_jni}/art/art_method_replace_6_0.cpp ${path_jni}/art/art_method_replace_7_0.cpp ${path_jni}/dalvik/dalvik_method_replace.cpp
	            )
	
	#找到Android的log库（这个库已经在Android平台中了）
	find_library(
	            log-lib
	            log
	            )
	
	#找到Android的的库（这个库已经在Android平台中了），这个库貌似用不上，姑且先加上吧
	find_library(
	            android-lib
	            android
	            )
	
	#把需要的库链接到自己的库中
	target_link_libraries(
	            andfix
	            ${log-lib}
	            ${android-lib}
	            )

####一、Patch文件分析

官网给出的AndFix核心原理如下图所示：

![AndFix核心原理.png](http://upload-images.jianshu.io/upload_images/2570030-34b0f5a998606a1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要是通过Native层的Hook技术，实现方法的动态替换，而替换哪个方法，就需要根据Patch文件中的@MethodReplace注解而决定。

从上面的例子中我们可以直观地看到，我们的BUG是通过加载一个Patch文件来修复，那么我们就从这个Patch文件作为我们源码分析的入口吧。

Patch文件实际上一个zip压缩的文件，因此我们不妨把它的后缀名改为zip，然后用解压缩工具打开。可以看到，

![Patch文件解压.png](http://upload-images.jianshu.io/upload_images/2570030-2cd6d8db7540d277.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到解压出来的是一个MeTA-INF文件夹以及dex文件。其中MeTA-INF文件夹里面的PATCH.MF文件保存的是这个Patch的一些关键信息。等下我们分析源码的时候需要用到。

再者就是下面的这个dex文件，我们不妨利用dex2jar-2.0工具对其进行反编译，Linux中的命令如下：

	./d2j-dex2jar.sh -f -o out.jar classes.dex

反编译出来以后我们我们再次解压jar包，直接拖动AS中打开：

    package com.nan.andfixdemo;

    import com.alipay.euler.andfix.annotation.MethodReplace;

    public class Test_CF {
        public Test_CF() {
        }

        @MethodReplace(
            clazz = "com.nan.andfixdemo.Test",
            method = "test"
        )
        public static int test() {
            return 1;
        }
    }

可以看到，其实这个dex文件就把我们需要替换的方法添加了一个@MethodReplace注解。

####二、AndFix中Java层核心类分析

AndFix里面有几个核心的类，其中包括：

	代表补丁文件的类：Patch
    补丁文件的管理：PatchManager
    修复的类：AndFixManager
	与C/C++层交互的类：AndFix

一个Patch文件实质上是对应一个Patch对象的：

public class Patch implements Comparable<Patch> {

	//省略一些代码
	private final File mFile;
	private String mName;
	private Date mTime;
	private Map<String, List<String>> mClassesMap;

	public Patch(File file) throws IOException {
		mFile = file;
		init();
	}

	@SuppressWarnings("deprecation")
	private void init() throws IOException {
		//省略一些代码
	}

	public String getName() {
		return mName;
	}

	public File getFile() {
		return mFile;
	}

	public Set<String> getPatchNames() {
		return mClassesMap.keySet();
	}

	public List<String> getClasses(String patchName) {
		return mClassesMap.get(patchName);
	}

	public Date getTime() {
		return mTime;
	}

	@Override
	public int compareTo(Patch another) {
		return mTime.compareTo(another.getTime());
	}

}

从上面的构造方法中可以看出，AndFix在创建Patch对象的时候，调用了inti方法：

	private void init() throws IOException {
		JarFile jarFile = null;
		InputStream inputStream = null;
		try {
			jarFile = new JarFile(mFile);
			JarEntry entry = jarFile.getJarEntry(ENTRY_NAME);
			inputStream = jarFile.getInputStream(entry);
			Manifest manifest = new Manifest(inputStream);
			Attributes main = manifest.getMainAttributes();
			mName = main.getValue(PATCH_NAME);
			mTime = new Date(main.getValue(CREATED_TIME));

			mClassesMap = new HashMap<String, List<String>>();
			Attributes.Name attrName;
			String name;
			List<String> strings;
			for (Iterator<?> it = main.keySet().iterator(); it.hasNext();) {
				attrName = (Attributes.Name) it.next();
				name = attrName.toString();
				if (name.endsWith(CLASSES)) {
					strings = Arrays.asList(main.getValue(attrName).split(","));
					if (name.equalsIgnoreCase(PATCH_CLASSES)) {
						mClassesMap.put(mName, strings);
					} else {
						mClassesMap.put(
								name.trim().substring(0, name.length() - 8),// remove
																			// "-Classes"
								strings);
					}
				}
			}
		} finally {
			if (jarFile != null) {
				jarFile.close();
			}
			if (inputStream != null) {
				inputStream.close();
			}
		}

	}

可以看出，在Patch构造的时候，调用了Java提供的一些读取jar文件的API去读取Patch文件。主要就是PATCH.MF文件这个文件：

    Manifest-Version: 1.0
    Patch-Name: new
    Created-Time: 20 Apr 2017 02:45:20 GMT
    From-File: new.apk
    To-File: old.apk
    Patch-Classes: com.nan.andfixdemo.Test_CF
    Created-By: 1.0 (ApkPatch)

例如读取了Patch名、创建日期等，其中最核心的就是读取Patch-Classes，这就是需要修复的类的全名：

    if (name.equalsIgnoreCase(PATCH_CLASSES)) {
        mClassesMap.put(mName, strings);
    } else {
        mClassesMap.put(
                name.trim().substring(0, name.length() - 8),// remove
                                                            // "-Classes"
                strings);
    }

因为这个Patch-Classes有可能会叫Classes，因此这里需要分两种情况处理。

上面就分析了Patch对象的构建，最终通过getClasses()方法就可以得到需要修复的所有的类。

####三、AndFix中Java层基本修复流程分析

我们回到自定义的Application中：

    //初始化PatchManager
    PatchManager mPatchManager = new PatchManager(this);
    mPatchManager.init("1.0");

    //加载已加载过的Patch文件
    mPatchManager.loadPatch();
    //添加外部Patch文件
    try {
        // .apatch file path
        String patchFileString = Environment.getExternalStorageDirectory().getAbsolutePath() + APATCH_PATH;
        mPatchManager.addPatch(patchFileString);
        Log.d(TAG, "加载成功：" + patchFileString);
    } catch (Exception e) {
        Log.e(TAG, "", e);
    }

一开始我们初始化了AndFix，然后调用loadPatch方法进行补丁的加载：

	public void loadPatch() {
		mLoaders.put("*", mContext.getClassLoader());// wildcard
		Set<String> patchNames;
		List<String> classes;
		for (Patch patch : mPatchs) {
			patchNames = patch.getPatchNames();
			for (String patchName : patchNames) {
				classes = patch.getClasses(patchName);
				mAndFixManager.fix(patch.getFile(), mContext.getClassLoader(),
						classes);
			}
		}
	}

在loadPatch方法方法中，先拿到Map中的ClassLoader，最后通过AndFixManager的fix方法进行修复。

由于AndFix在加载Patch之后，会将当前的Patch保存起来，下次将不再加载。那么我们的Patch一开始其实是从下面这段代码加载的：

    try {
        // .apatch file path
        String patchFileString = Environment.getExternalStorageDirectory().getAbsolutePath() + APATCH_PATH;
        mPatchManager.addPatch(patchFileString);
        Log.d(TAG, "加载成功：" + patchFileString);
    } catch (Exception e) {
        Log.e(TAG, "", e);
    }

核心逻辑就是通过PatchManager的addPatch方法进行加载：

	public void addPatch(String path) throws IOException {
		File src = new File(path);
		File dest = new File(mPatchDir, src.getName());
		if(!src.exists()){
			throw new FileNotFoundException(path);
		}
		if (dest.exists()) {
			Log.d(TAG, "patch [" + path + "] has be loaded.");
			return;
		}
		FileUtil.copyFile(src, dest);// copy to patch's directory
		Patch patch = addPatch(dest);
		if (patch != null) {
			loadPatch(patch);
		}
	}

我们可以看到，AndFix是把我们添加进来的Patch文件进行了copy，放到一个特定的目录中去的，下次就不会再加载了。因此我们在使用AndFix的时候，如果再次修复BUG的时候，就需要修改Patch文件的名字了，否则将不会再次加载，这是一个隐藏的大坑啊！不过这样做的好处就是省去了每次重复加载的工作，提高了APP的性能。

最后也是调用loadPatch方法进行加载以及fix。

####四、AndFix中Java层修复流程分析

从上面的分析我们知道，Java层最终会调用AndFixManager的fix方法进行修复（方法替换）。那么我们不妨先进来看一看究竟：

	public synchronized void fix(File file, ClassLoader classLoader,
			List<String> classes) {

		//判断AndFix是否可用，主要是判断是否正确初始化了
		if (!mSupport) {
			return;
		}

		//验证Patch文件（MD5验证，相关代码请自行分析）
		if (!mSecurityChecker.verifyApk(file)) {// security check fail
			return;
		}

		try {
			File optfile = new File(mOptDir, file.getName());
			boolean saveFingerprint = true;
			if (optfile.exists()) {
				if (mSecurityChecker.verifyOpt(optfile)) {
					saveFingerprint = false;
				} else if (!optfile.delete()) {
					return;
				}
			}

			//加载Patch文件中的dex文件
			final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
					optfile.getAbsolutePath(), Context.MODE_PRIVATE);

			if (saveFingerprint) {
				mSecurityChecker.saveOptSig(optfile);
			}

			//自定义一个ClassLoader去加载需要修复的类
			ClassLoader patchClassLoader = new ClassLoader(classLoader) {
				@Override
				protected Class<?> findClass(String className)
						throws ClassNotFoundException {
					Class<?> clazz = dexFile.loadClass(className, this);
					if (clazz == null
							&& className.startsWith("com.alipay.euler.andfix")) {
						return Class.forName(className);// annotation’s class
														// not found
					}
					if (clazz == null) {
						throw new ClassNotFoundException(className);
					}
					return clazz;
				}
			};

            //查找相应的修复注解，如果找到，就进行修复
			Enumeration<String> entrys = dexFile.entries();
			Class<?> clazz = null;
			while (entrys.hasMoreElements()) {
				String entry = entrys.nextElement();
				if (classes != null && !classes.contains(entry)) {
					continue;// skip, not need fix
				}
				clazz = dexFile.loadClass(entry, patchClassLoader);
				if (clazz != null) {
					fixClass(clazz, classLoader);
				}
			}
		} catch (IOException e) {
			Log.e(TAG, "pacth", e);
		}
	}

在AndFixManager的fix方法中，一开始进行了一些是否初始化的验证、Patch文件的验证，然后就加载Patch包中的dex文件，生成一个DexFile对象。

然后通过自定义的类加载器加载DexFile对象中的类。这里由于是加载我们第三方的class，因此需要自定义一个类加载器。加载成功以后，就通过反射的方式循环去读方法上面的注解，如果找到了注解，就进行修复。

下面继续看fixClass方法，这里就是通过循环找到MethodReplace注解，然后调用replaceMethod进行方法替换。（AndFix热修复实质就是方法的替换）

	private void fixClass(Class<?> clazz, ClassLoader classLoader) {
		Method[] methods = clazz.getDeclaredMethods();
		MethodReplace methodReplace;
		String clz;
		String meth;
		for (Method method : methods) {
			methodReplace = method.getAnnotation(MethodReplace.class);
			if (methodReplace == null)
				continue;
			clz = methodReplace.clazz();
			meth = methodReplace.method();
			if (!isEmpty(clz) && !isEmpty(meth)) {
            	//方法替换，参数分别是：类加载器、需要修复的类、修复好的方法、被修复的方法
				replaceMethod(classLoader, clz, meth, method);
			}
		}
	}

MethodReplace注解的定义如下：

    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface MethodReplace {
        String clazz();

        String method();
    }

可以看到，这是一个运行时的注解，只能够使用在方法上面。注解中指定了类名以及方法名。

通过分析方法的调用链，replaceMethod方法最终会调用AndFix的静态方法replaceMethod：

	private static native void replaceMethod(Method dest, Method src);

可以看到这是一个native方法。

####五、AndFix中C/C++层修复流程分析

我们找到andfix.cpp，找到了replaceMethod的实现：

    static void replaceMethod(JNIEnv* env, jclass clazz, jobject src, jobject dest) {
        if (isArt) {
            art_replaceMethod(env, src, dest);
        } else {
            dalvik_replaceMethod(env, src, dest);
        }
    }

这里需要判断当前的虚拟机类型是dalvik还是art。在JNI初始化的时候，需要注册虚拟机（方法替换的实质就是通过Hook虚拟机层的一些流程实现的，下面将会介绍到）：

    JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
        JNIEnv* env = NULL;
        jint result = -1;

        if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
            return -1;
        }
        assert(env != NULL);

        if (!registerNatives(env)) { //注册
            return -1;
        }
        
		//需要返回JNI 1.4以上的版本
        result = JNI_VERSION_1_4;

        return result;
    }

其中注册虚拟机的时候，调用了registerNatives，最终调用registerNativeMethods方法，进行native方法的链接（注册与一些与类关联的native方法，向虚拟机进行登记，这是JNI实现的另外一种方式，具体可以参考JNI相关文档，这里不再深入）：

    static int registerNatives(JNIEnv* env) {
        if (!registerNativeMethods(env, JNIREG_CLASS, gMethods,
                sizeof(gMethods) / sizeof(gMethods[0])))
            return JNI_FALSE;

        return JNI_TRUE;
	}

这里传入的gMethods数组如下：

    static JNINativeMethod gMethods[] = {
    /* name, signature, funcPtr */
    { "setup", "(ZI)Z", (void*) setup }, { "replaceMethod",
            "(Ljava/lang/reflect/Method;Ljava/lang/reflect/Method;)V",
            (void*) replaceMethod }, { "setFieldFlag",
            "(Ljava/lang/reflect/Field;)V", (void*) setFieldFlag }, };

gMethods实质就是Andfix中的一些方法：setup、replaceMethod、setFieldFlag，会在JNI加载的时候调用，进行初始化，下面主要看setup方法中的Java实现：

	public static boolean setup() {
		try {
			final String vmVersion = System.getProperty("java.vm.version");
			boolean isArt = vmVersion != null && vmVersion.startsWith("2");
			int apilevel = Build.VERSION.SDK_INT;
			return setup(isArt, apilevel);
		} catch (Exception e) {
			Log.e(TAG, "setup", e);
			return false;
		}
	}

在Java代码中，判断了当前虚拟机类型是dalvik还是art，获取了API的等级。最终又调用了native层的setup方法，下面来看native层setup方法的实现：

    static jboolean setup(JNIEnv* env, jclass clazz, jboolean isart,
            jint apilevel) {
        isArt = isart;
        LOGD("vm is: %s , apilevel is: %i", (isArt ? "art" : "dalvik"),
                (int )apilevel);
        if (isArt) {
            return art_setup(env, (int) apilevel);
        } else {
            return dalvik_setup(env, (int) apilevel);
        }
    }

具体的虚拟机注册比较复杂，为了简单起见，我们只分析一下dalvik虚拟机的初始化：

	extern jboolean __attribute__ ((visibility ("hidden"))) dalvik_setup(
			JNIEnv* env, int apilevel) {
		//通过dlopen（该方法在系统头文件dlfcn.h中）加载libdvm.so
		void* dvm_hand = dlopen("libdvm.so", RTLD_NOW);
		//进行Hook
		if (dvm_hand) {
			dvmDecodeIndirectRef_fnPtr = dvm_dlsym(dvm_hand,
					apilevel > 10 ?
							"_Z20dvmDecodeIndirectRefP6ThreadP8_jobject" :
							"dvmDecodeIndirectRef");
			if (!dvmDecodeIndirectRef_fnPtr) {
				return JNI_FALSE;
			}
			dvmThreadSelf_fnPtr = dvm_dlsym(dvm_hand,
					apilevel > 10 ? "_Z13dvmThreadSelfv" : "dvmThreadSelf");
			if (!dvmThreadSelf_fnPtr) {
				return JNI_FALSE;
			}
			jclass clazz = env->FindClass("java/lang/reflect/Method");
			jClassMethod = env->GetMethodID(clazz, "getDeclaringClass",
							"()Ljava/lang/Class;");
	
			return JNI_TRUE;
		} else {
			return JNI_FALSE;
		}
	}

先来补充一些基本知识：

![Android虚拟机初始化.png](http://upload-images.jianshu.io/upload_images/2570030-3e9e1ec7e529bba2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 如上图所示，Android虚拟机是有别于Java原生的虚拟机的，它执行的是dex文件而不是class文件。Android虚拟机分为dalvik虚拟机和art虚拟机。
2. 虚拟机（进程）启动的时候会加载一个很重要的动态库文件（libdalvik.so或者libart.so）。
3. Java在虚拟机环境中执行，每个Java方法都会对应一个底层的函数指针，当Java方法被调用的时候，实质虚拟机会找到这个函数指针然后去执行底层的方法，从而Java方法被执行。

我们回到虚拟机初始化的分析中来，dalvik_setup方法主要做了两个步骤：

1. 通过调用dlopen（该方法在系统头文件dlfcn.h中）加载libdvm.so（这个so在APP进程初始化的时候会加载），这个加载是为了下一步的Hook做准备。
2. 加载完libdvm.so之后，就可以进行Hook了。在API10以上、以下，Java方法调用的时候会执行不同的底层的系统函数，因此必须Hook不同的系统函数才会有效。Hook成功以后，在这些系统函数调用的时候，就会调用我们自己的代码，进行替换。

我们在loadPatch的时候，最终会调用AndFixManager的fix方法，根据一系列的调用链，最终会调用dalvik_replaceMethod或者art_replaceMethod。下面继续以dalvik虚拟机为例，继续来看dalvik_replaceMethod方法的实现：

	extern void __attribute__ ((visibility ("hidden"))) dalvik_replaceMethod(
			JNIEnv* env, jobject src, jobject dest) {
		jobject clazz = env->CallObjectMethod(dest, jClassMethod);
	
		//我们可以看到刚刚我们Hook成功的两个函数
		ClassObject* clz = (ClassObject*) dvmDecodeIndirectRef_fnPtr(dvmThreadSelf_fnPtr(), clazz);
		clz->status = CLASS_INITIALIZED;
	
		//进行方法替换
		Method* meth = (Method*) env->FromReflectedMethod(src);
		Method* target = (Method*) env->FromReflectedMethod(dest);
		LOGD("dalvikMethod: %s", meth->name);
	
	//	meth->clazz = target->clazz;
		meth->accessFlags |= ACC_PUBLIC;
		meth->methodIndex = target->methodIndex;
		meth->jniArgInfo = target->jniArgInfo;
		meth->registersSize = target->registersSize;
		meth->outsSize = target->outsSize;
		meth->insSize = target->insSize;
	
		meth->prototype = target->prototype;
		meth->insns = target->insns;
		meth->nativeFunc = target->nativeFunc;
	}

这就是AndFix的核心代码了，当BUG方法被底层系统函数调用的时候，我们的Hook的钩子函数就会调用，然后就是进行有BUG与无BUG的Java方法的所有成员的变量替换，达到一个狸猫换太子的目的。

总结一句话就是，通过Hook系统的底层函数，在我们有BUG的Java方法被调用的时候，通过一句“刀下留人”，然后狸猫换太子一样，调用我们已经修复好的方法。

###结束语

关于AndFix热修复的使用与分析就到这里了，有一些东西可能解析得不是特别清楚，毕竟这些玩意还是非常深入的，对于我们一般的开发者来说，会使用一些常见的热修复框架即可，无需太过深入。深入分析源码通常来说只是为了我们更好去使用而已。

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
