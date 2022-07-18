###缓存策略

有两种：

####一、使用局部的static变量：

	JNIEXPORT void JNICALL Java_com_test_JniTest_cache
	(JNIEnv * env, jobject jobj){
	
		static jfieldID fid = NULL;
		jclass clz = (*env)->GetObjectClass(env, jobj);
	
		if (fid == NULL){
			fid = (*env)->GetFieldID(env, clz, "key", "Ljava/lang/String;");
			printf("fid inti once");
		}
	}

测试：


	public native void cache();

	public static void main(String[] args) {

		JniTest test = new JniTest();
		for (int i = 0; i < 100; i++) {
			test.cache();
		}
	}

说明：

获取jfieldID只获取一次。局部静态变量只能在本方法的作用域中使用。

也可以搞个全局，但是fid不同函数可以使用，但是值不一样，值很难统一。因此官方推荐局部的。

局部静态变量的生命周期：

1. 初始化，函数第一次执行
2. 结束，作用域被销毁了，但是这个变量还会存在内存当中，直到程序结束。


####二、动态库加载的时候初始化全局变量


	public static native void initIds();

	static {
		System.loadLibrary("JniTest");
		initIds();
	}

	public static void main(String[] args) {

		JniTest test = new JniTest();
		for (int i = 0; i < 100; i++) {
			test.cache();
		}
	}

C代码如下：

	//初始化两个全局变量，动态库加载完成之后，立刻缓存起来
	//以后可以在其他函数使用，声明周期也是跟应用程序（进程）一致
	jfieldID key_fid;
	jmethodID random_mid;
	JNIEXPORT void JNICALL Java_com_dongnaoedu_jni_JniTest_initIds(JNIEnv *env, jclass jcls){
		key_fid = (*env)->GetFieldID(env, jcls, "key", "Ljava/lang/String;");
		random_mid = (*env)->GetMethodID(env, jcls, "genRandomInt", "(I)I");
	}

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
