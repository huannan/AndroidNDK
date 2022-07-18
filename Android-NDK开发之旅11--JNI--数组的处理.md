###数组的处理（主要是同步问题）

Java方法中，通过调用accessField，利用C修改静态属性

	//数组处理
	public native void sortArray(int array[]);

	public static void main(String[] args) {

		JniTest test = new JniTest();
		int arr[] = { 3, 2, 4, 5, 1, 0 };
		//调用C的函数进行快速排序
		test.sortArray(arr);
		System.out.println(Arrays.toString(arr));

	}

C代码如下：

	int compare(const int * a, const int * b){
		return (*a) - (*b);
	}
	
	JNIEXPORT void JNICALL Java_com_test_JniTest_sortArray
	(JNIEnv * env, jobject jobj, jintArray arr){
	
		//创建Java数组
		//(*env)->NewIntArray(env, len);
	
		//通过Java的数组，拿到C的数组的指针
		jint* c_arr = (*env)->GetIntArrayElements(env, arr, NULL);
		//获取Java数组的大小
		jsize len = (*env)->GetArrayLength(env, arr);
		//排序，其中compare是函数指针，用于比较大小，与Java类似
		qsort(c_arr, len, sizeof(jint), compare);
	
		//操作完之后需要同步C的数组到Java数组中
		(*env)->ReleaseIntArrayElements(env, arr, c_arr, 0);
	}

####注意：

1、通过GetIntArrayElements拿到C类型的数组的指针，然后才能进行C数组的处理。

2、C拿到Java的数组进行操作或者修改以后，需要调用ReleaseIntArrayElements进行更新，这时候Java的数组也会同步更新过来。

这个方法的最后一个参数是模式：

	0：			Java数组进行更新，并且释放C/C++数组。
	JNI_ABORT：	Java数组不进行更新，但是释放C/C++数组。
	JNI_COMMIT：	Java数组进行更新，不释放C/C++数组（函数执行完，数组还是会释放）。



如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
