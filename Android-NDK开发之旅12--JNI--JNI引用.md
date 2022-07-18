###JNI引用

JNI引用概念：引用变量。

引用类型：局部引用和全局引用（全局引用里面包含全局弱引用）。

作用：在JNI中告知虚拟机何时回收一个JNI变量。

####局部引用

局部引用，通过DeleteLocalRef手动释放对象。

典型使用场景：

1. 访问一个很大的java对象，使用完之后，还要进行复杂的耗时操作。
2. 创建了大量的局部引用，占用了太多的内存，而且这些局部引用跟后面的操作没有关联性。

例子：

Java代码：

	// 局部引用
	public native void localRef();

C代码如下，这里模拟了局部引用的大量创建与删除：

	JNIEXPORT void JNICALL Java_com_test_JniTest_localRef
	(JNIEnv * env, jobject jobj){
	
		int i = 0;
		for (; i < 10;i++){
	
			//在循环中创建了占用内存大的对象（例子）
			jclass clz_date = (*env)->FindClass(env, "java/util/Date");
			jmethodID mid = (*env)->GetMethodID(env, clz_date, "<init>", "()V");
			jobject jobj_date = (*env)->NewObject(env, clz_date, mid);
	
			//此处省略一万行代码
	
			//不再使用jobject对象了
			//通知垃圾回收器回收这些对象，确保内存充足
			(*env)->DeleteLocalRef(env, jobj_date);
			printf("DeleteLocalRef\n");
		}
	}

测试：

	public static void main(String[] args) {

		JniTest test = new JniTest();
		test.localRef();
		
	}


####全局引用

主要作用：共享(可以跨多个线程)，手动控制内存使用。

Java中提供三个方法，分别用于创建、获取、删除JNI全局引用。

	public native void createGlobalRef();

	public native String getGlobalRef();

	public native void deteleGlobalRef();


C代码如下：

	//全局引用的字符串对象
	jstring global_jstr;
	
	JNIEXPORT void JNICALL Java_com_test_JniTest_createGlobalRef
	(JNIEnv * env, jobject jobj){
		global_jstr = (*env)->NewStringUTF(env, "I Love You");
		//通过NewGlobalRef创建全局引用
		(*env)->NewGlobalRef(env, global_jstr);
	}
	
	JNIEXPORT jstring JNICALL Java_com_test_JniTest_getGlobalRef
	(JNIEnv * env, jobject jobj){
		//获取全局引用
		return global_jstr;
	}
	
	JNIEXPORT void JNICALL Java_com_test_JniTest_deteleGlobalRef
	(JNIEnv * env, jobject jobj){
		//通过DeleteGlobalRef删除全局引用
		(*env)->DeleteGlobalRef(env, global_jstr);
	}

测试：

	public static void main(String[] args) {

		JniTest test = new JniTest();
		test.localRef();

		test.createGlobalRef();
		System.out.println(test.getGlobalRef());
		test.deteleGlobalRef();
		
		//删除之后再取出会抛出空指针异常
		System.out.println(test.getGlobalRef());
		
	}

####弱全局引用

使用方法与全局引用类似:

1. 通过NewWeakGlobalRef创建全局引用。
2. 通过DeleteWeakGlobalRef删除全局引用。

与全局引用不一样的是，弱全局引用：

1. 节省内存，在内存不足时可以是释放所引用的对象。
2. 可以引用一个不常用的对象，如果为NULL的时候（被回收了），可以手动再临时创建一个。

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
