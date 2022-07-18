###异常处理

异常测试例子：

	public native void testException1();

	public static void main(String[] args) {

		JniTest test = new JniTest();

		try {
			test.testException();
			System.out.println("程序无法继续执行1，这句话不会被打印\n");
		} catch (Throwable t) {
			System.out.println("捕获到JNI抛出的异常(Throwable)，这句话会被打印" + t.getMessage() + "\n");
		}

		System.out.println("程序继续执行2，这句话会被打印\n");
		
	}

C代码如下：

	//异常处理
	JNIEXPORT void JNICALL Java_com_test_JniTest_testException1
	(JNIEnv * env, jobject jobj){

		jclass clz=  (*env)->GetObjectClass(env, jobj);
		//属性名字不小心写错了，拿到的是空的jfieldID
		jfieldID fid = (*env)->GetFieldID(env, clz, "key1", "Ljava/lang/String;");
	
		//此处抛出的异常，Java可以通过Throwable来捕获

		printf("C can run , this will print");
		//这里竟然还可以继续执行
		jstring key =  (*env)->GetObjectField(env, jobj, fid);
		//遇到这句话的时候，C程序Crash了
		char* c_str = (*env)->GetStringUTFChars(env, key, NULL);
		printf("C could not run , this will not print");
	}

通过例子可以知道，JNI层自己抛出的异常是Error类型，Java可以通过Throwable或者Error来捕获得到，捕获异常后Java代码可以继续执行下去。

为了确保Java、C/C++代码可以正常执行下去，需要：

1. 在JNI层手动清空异常信息（ExceptionClear），保证代码可以运行。
2. 补救措施保证C/C++代码继续运行。

例如：
		
	//异常处理
	JNIEXPORT void JNICALL Java_com_test_JniTest_testException1
	(JNIEnv * env, jobject jobj){
		jclass clz = (*env)->GetObjectClass(env, jobj);
		//属性名字不小心写错了，拿到的是空的jfieldID
		jfieldID fid = (*env)->GetFieldID(env, clz, "key1", "Ljava/lang/String;");
	
		jthrowable err = (*env)->ExceptionOccurred(env);
		if (err != NULL){
			//手动清空异常信息，保证Java代码能够继续执行
			(*env)->ExceptionClear(env);
			//提供补救措施，例如获取另外一个属性
			fid = (*env)->GetFieldID(env, clz, "key", "Ljava/lang/String;");
		}
	
	
		jstring key = (*env)->GetObjectField(env, jobj, fid);
		char* c_str = (*env)->GetStringUTFChars(env, key, NULL);
	}

测试代码如下：

	public static void main(String[] args) {

		JniTest test = new JniTest();

		try {
			test.testException();
			System.out.println("程序没有异常，这句话会被打印\n");
		} catch (Exception e) {
			System.out.println("没有捕获到JNI抛出的异常，这句话不会被打印" + e.getMessage() + "\n");
		}

		System.out.println("程序继续执行，这句话会被打印\n");
		
	}

用户可以手动通过ThrowNew函数抛出异常，同样可以被Java代码捕获：

	//异常处理
	JNIEXPORT void JNICALL Java_com_test_JniTest_testException
	(JNIEnv * env, jobject jobj){
		jclass clz = (*env)->GetObjectClass(env, jobj);
		//属性名字不小心写错了，拿到的是空的jfieldID
		jfieldID fid = (*env)->GetFieldID(env, clz, "key1", "Ljava/lang/String;");
	
		jthrowable err = (*env)->ExceptionOccurred(env);
		if (err != NULL){
			//手动清空异常信息，保证Java代码能够继续执行
			(*env)->ExceptionClear(env);
			//提供补救措施，例如获取另外一个属性
			fid = (*env)->GetFieldID(env, clz, "key", "Ljava/lang/String;");
		}
	
	
		jstring key = (*env)->GetObjectField(env, jobj, fid);
		char* c_str = (*env)->GetStringUTFChars(env, key, NULL);
	
		//参数不正确，程序员自己抛出异常，可以在Java中捕获
		if (_stricmp(c_str,"efg") != 0){
			jclass err_clz = (*env)->FindClass(env, "java/lang/IllegalArgumentException");
			(*env)->ThrowNew(env, err_clz, "key value is invalid!");
		}
	}

测试代码如下：

	public static void main(String[] args) {

		JniTest test = new JniTest();

		try {
			test.testException();
			System.out.println("JNI手动抛出了异常，Java不会继续执行，这句话不会被打印\n");
		} catch (Exception e) {
			System.out.println("捕获到JNI手动抛出的异常，这句话会被打印：" + e.getMessage() + "\n");
		}

		System.out.println("程序继续执行，这句话会被打印\n");

	}

###异常处理总结

1. JNI自己抛出的异常，是Error类型，Java可以通过Throwable或者Error来捕获得到，捕获异常后Java代码可以继续执行下去。在C层可以清空（ExceptionClear），保证try中的代码Java代码继续执行，并且最好要提供补救措施，确保JNI层代码正常继续运行。
2. 用户通过ThrowNew手动抛出的异常，同样可以在Java层捕捉得到。

####题外话——异常的种类

![异常分类.jpg](http://upload-images.jianshu.io/upload_images/2570030-8c8bdbeb6c49280d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Throwable： 有两个重要的子类：Exception（异常）和 Error（错误），二者都是 Java 异常处理的重要子类，各自都包含大量子类。
 
Error（错误）:是程序无法处理的错误，表示运行应用程序中较严重问题。大多数错误与代码编写者执行的操作无关，而表示代码运行时 JVM（Java 虚拟机）出现的问题。例如，Java虚拟机运行错误（Virtual MachineError），当 JVM 不再有继续执行操作所需的内存资源时，将出现 OutOfMemoryError。这些异常发生时，Java虚拟机（JVM）一般会选择线程终止。这些错误表示故障发生于虚拟机自身、或者发生在虚拟机试图执行应用时，如Java虚拟机运行错误（Virtual MachineError）、类定义错误（NoClassDefFoundError）等。这些错误是不可查的，因为它们在应用程序的控制和处理能力之 外，而且绝大多数是程序运行时不允许出现的状况。对于设计合理的应用程序来说，即使确实发生了错误，本质上也不应该试图去处理它所引起的异常状况。在 Java中，错误通过Error的子类描述。


Exception（异常）:是程序本身可以处理的异常。
Exception 类有一个重要的子类 RuntimeException。RuntimeException 类及其子类表示“JVM 常用操作”引发的错误。例如，若试图使用空值对象引用、除数为零或数组越界，则分别引发运行时异常（NullPointerException、ArithmeticException）和 ArrayIndexOutOfBoundException。 

注意：异常和错误的区别：异常能被程序本身可以处理，错误是无法处理。


通常，Java的异常(包括Exception和Error)分为可查的异常（checked exceptions）和不可查的异常（unchecked exceptions）。
可查异常（编译器要求必须处置的异常）：正确的程序在运行中，很容易出现的、情理可容的异常状况。可查异常虽然是异常状况，但在一定程度上它的发生是可以预计的，而且一旦发生这种异常状况，就必须采取某种方式进行处理。

除了RuntimeException及其子类以外，其他的Exception类及其子类都属于可查异常。这种异常的特点是Java编译器会检查它，也就是说，当程序中可能出现这类异常，要么用try-catch语句捕获它，要么用throws子句声明抛出它，否则编译不会通过。

不可查异常(编译器不要求强制处置的异常):包括运行时异常（RuntimeException与其子类）和错误（Error）。

Exception 这种异常分两大类运行时异常和非运行时异常(编译异常)。程序中应当尽可能去处理这些异常。

运行时异常：都是RuntimeException类及其子类异常，如NullPointerException(空指针异常)、IndexOutOfBoundsException(下标越界异常)等，这些异常是不检查异常，程序中可以选择捕获处理，也可以不处理。这些异常一般是由程序逻辑错误引起的，程序应该从逻辑角度尽可能避免这类异常的发生。

1. 运行时异常的特点是Java编译器不会检查它，也就是说，当程序中可能出现这类异常，即使没有用try-catch语句捕获它，也没有用throws子句声明抛出它，也会编译通过。
2. 非运行时异常 （编译异常）：是RuntimeException以外的异常，类型上都属于Exception类及其子类。从程序语法角度讲是必须进行处理的异常，如果不处理，程序就不能编译通过。如IOException、SQLException等以及用户自定义的Exception异常，一般情况下不自定义检查异常。


如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
