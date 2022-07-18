###JNI数据类型

####基本数据
Java基本数据类型与JNI数据类型的映射关系

	Java类型->JNI类型->C类型

JNI的基本数据类型有（左边是Java，右边是JNI）：

	boolean 			jboolean
	byte 				jbyte
	char 				jchar
	short 				jshort
	int 				jint
	long 				jlong
	float 				jfloat
	double 				jdouble
	void 				void


####引用类型(对象)

	String 				jstring
	Object 				jobject

	数组,基本数据类型的数组
	byte[]				jByteArray
	对象数组
	object[](String[]) 	jobjectArray

###native函数参数说明

每个native函数，都至少有两个参数（JNIEnv*,jclass或者jobject)。

1）当native方法为静态方法时：
jclass 代表native方法所属类的class对象(JniTest.class)。

2）当native方法为非静态方法时：
jobject 代表native方法所属的对象。

######native函数的头文件可以自己写。

###关于属性与方法的签名

####一、属性的签名

属性的签名其实就是属性的类型的简称，对应关系如下：

![Java属性的签名.png](http://upload-images.jianshu.io/upload_images/2570030-a22ab83663ee5ba6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

尤其注意的是，类的签名格式就是：

	L完整包名;

其中完整包名用 / 代替.

末尾的 ; 不能省略

数组的签名就是：

	[类型签名

其中，多为数组就用多个[

####二、方法的签名

获取方法的签名比较麻烦一些，通过下面的方法也可以拿到属性的签名。

打开命令行，输入javap，出现以下信息：

![javap命令.png](http://upload-images.jianshu.io/upload_images/2570030-65a5bd105ce2ae8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上述信息告诉我们，通过以下命令就可以拿到指定类的所有属性、方法的签名了，很方便有木有？！

	javap -s -p 完整类名

我们通过cd命令，来到Java工程的bin目录，然后输入命令：

	D:\WebPrj\TestJni\bin>javap -s -p com.test.JniTest
	Compiled from "JniTest.java"
	public class com.test.JniTest {
	  public java.lang.String str;
	    descriptor: Ljava/lang/String;
	  public static int NUM;
	    descriptor: I
	  static {};
	    descriptor: ()V
	
	  public com.test.JniTest();
	    descriptor: ()V
	
	  public static native java.lang.String getStringFromC();
	    descriptor: ()Ljava/lang/String;
	
	  public native void accessField();
	    descriptor: ()V
	
	  public native void accessStaticField();
	    descriptor: ()V
	
	  public int genRandomInt(int);
	    descriptor: (I)I
	
	  public native void accessMethod();
	    descriptor: ()V
	
	  public native void accessStaticMethod();
	    descriptor: ()V
	
	  public static java.lang.String getUUID();
	    descriptor: ()Ljava/lang/String;
	
	  public static void main(java.lang.String[]);
	    descriptor: ([Ljava/lang/String;)V
	}

其中，descriptor就是我们需要的签名了，注意签名中末尾的分号不能省略。

方法签名的规律就是，括号不可以省略：

	(参数类型签名)返回值类型签名

###C/C++访问Java的属性、方法

有以下几种情况：

1. 访问Java类的非静态属性。
2. 访问Java类的静态属性。
3. 访问Java类的非静态方法。
4. 访问Java类的静态方法。
5. 间接访问Java类的父类的方法。
6. 访问Java类的构造方法。

####一、访问Java的非静态属性

Java方法中，通过调用accessField，利用C修改静态属性

	public String str = "Li lu";
	
	//访问非静态属性str，修改它的值
	public native void accessField();

C代码如下：（头文件可以不写，直接写实现）

	JNIEXPORT void JNICALL Java_com_test_JniTest_accessField
	(JNIEnv * env, jobject jobj){
	
		//通过对象拿到Class
		jclass clz = (*env)->GetObjectClass(env, jobj);
		//拿到对应属性的ID
		jfieldID fid = (*env)->GetFieldID(env, clz, "str", "Ljava/lang/String;");
		//通过属性ID拿到属性的值
		jstring jstr = (*env)->GetObjectField(env, jobj, fid);
	
		//通过Java字符串拿到C字符串，第三个参数是一个出参，用来告诉我们GetStringUTFChars内部是否复制了一份字符串
		//如果没有复制，那么出参为isCopy，这时候就不能修改字符串的值了，因为Java中常量池中的字符串是不允许修改的（但是jstr可以指向另外一个字符串）
		char* cstr = (*env)->GetStringUTFChars(env, jstr, NULL);
		//在C层修改这个属性的值
		char res[20] = "I love you : ";
		strcat(res, cstr);
	
		//重新生成Java的字符串，并且设置给对应的属性
		jstring jstr_new = (*env)->NewStringUTF(env, res);
		(*env)->SetObjectField(env, jobj, fid, jstr_new);
	
		//最后释放资源，通知垃圾回收器来回收
		//良好的习惯就是，每次GetStringUTFChars，结束的时候都有一个ReleaseStringUTFChars与之呼应
		(*env)->ReleaseStringUTFChars(env, jstr, cstr);
	}

最后在Java中测试：

	public static void main(String[] args) {

		JniTest test = new JniTest();
		System.out.println(test.str);
		//修改非静态属性str
		test.accessField();
		System.out.println(test.str);
		
	}

####二、访问Java的静态属性

Java代码如下：

	//访问静态属性NUM，修改它的值
	public static int NUM = 1;
	
	public native void accessStaticField();

C代码如下：

	JNIEXPORT void JNICALL Java_com_test_JniTest_accessStaticField
	(JNIEnv * env, jobject jobj){
		//与上面类似，只不过是某些方法需要加上Static
		jclass clz = (*env)->GetObjectClass(env, jobj);
		jfieldID fid = (*env)->GetStaticFieldID(env, clz, "NUM", "I");
		jint jInt = (*env)->GetStaticIntField(env, clz, fid);
		jInt++;
		(*env)->SetStaticIntField(env, clz, fid, jInt);
	}

最后在Java中测试：

	public static void main(String[] args) {
		
		JniTest test = new JniTest();
		System.out.println(NUM);
		test.accessStaticField();
		System.out.println(NUM);

	}

####三、访问Java的非静态方法

Java代码如下，通过调用accessMethod，在底层用C语言调用genRandomInt方法

	//产生指定范围的随机数
	public int genRandomInt(int max){
		System.out.println("genRandomInt 执行了...max = "+ max);
		return new Random().nextInt(max); 
	}
	
	public native void accessMethod();

C代码如下：

	JNIEXPORT void JNICALL Java_com_test_JniTest_accessMethod
	(JNIEnv * env, jobject jobj){
		jclass clz = (*env)->GetObjectClass(env, jobj);
		//拿到方法的ID，最后一个参数是方法的签名
		jmethodID mid = (*env)->GetMethodID(env, clz, "genRandomInt", "(I)I");
		//调用该方法，最后一个是可变参数，就是调用该方法所传入的参数
		//套路是如果返回是：Call返回类型Method
		jint jInt = (*env)->CallIntMethod(env, jobj, mid, 100);
		printf("output from C ： %d", jInt);
	}

最后在Java中测试：

	public static void main(String[] args) {
		
		JniTest test = new JniTest();
		test.accessMethod();

	}

####四、访问Java的静态方法

Java代码如下，通过调用accessStaticMethod，在底层用C语言调用getUUID方法

	public native void accessStaticMethod();
	
	//产生UUID字符串
	public static String getUUID(){
		System.out.println("getUUID 执行了...");
		return UUID.randomUUID().toString();
	}

C代码如下：

	JNIEXPORT void JNICALL Java_com_test_JniTest_accessStaticMethod
	(JNIEnv * env, jobject jobj){
		jclass clz = (*env)->GetObjectClass(env, jobj);
		jmethodID mid = (*env)->GetStaticMethodID(env, clz, "getUUID", "()Ljava/lang/String;");
	
		//调用java的静态方法，拿到返回值
		jstring jstr = (*env)->CallStaticObjectMethod(env, clz, mid);
	
		//把拿到的Java字符串转换为C的字符串
		char* cstr= (*env)->GetStringUTFChars(env, jstr, NULL);
	
		//后续操作，产生以UUID为文件名的文件
		char fielName[100];
		sprintf(fielName, "D:\\%s.txt", cstr);
		FILE* f = fopen(fielName, "w");
		fputs(cstr, f);
		fclose(f);
	
		printf("output from C : File had saved", jstr);
	}

最后在Java中测试：

	public static void main(String[] args) {

		JniTest test = new JniTest();
		test.accessStaticMethod();

	}


####五、间接访问Java类的父类的方法

Java代码如下：

父类：

	package com.test;
	
	public class Human {
		protected void speek() {
			System.out.println("Human Speek");
		}	
	}

子类：

	package com.test;
	
	public class Man extends Human {
		@Override
		protected void speek() {
			// 可以通过super关键字来访问父类的方法
			// super.speek();
			System.out.println("Man Speek");
		}
	}

在TestJni类中有Human属性：

	//父类的引用指向子类的对象
	Human man= new Man();
	
	public native void accessNonvirtualMethod();

如果是直接使用man.speek()的话，访问的是子类Man的方法
但是通过底层C的方式可以间接访问到父类Human的方法，跳过子类的实现，甚至你可以直接哪个父类（如果父类有多个的话），这是Java做不到的。

下面是C代码实现，无非就是属性和方法的访问：

	JNIEXPORT void JNICALL Java_com_test_JniTest_accessNonvirtualMethod
	(JNIEnv * env, jobject jobj){
		//先拿到属性man
		jclass clz=(*env)->GetObjectClass(env, jobj);
		jfieldID fid = (*env)->GetFieldID(env, clz, "man", "Lcom/test/Human;");
		jobject man = (*env)->GetObjectField(env, jobj, fid);
	
		//拿到父类的类，以及speek的方法id
		jclass clz_human = (*env)->FindClass(env, "com/test/Human");
		jmethodID mid = (*env)->GetMethodID(env, clz_human, "speek", "()V");
	
		//调用自己的speek实现
		(*env)->CallVoidMethod(env, man, mid);
		//调用父类的speek实现
		(*env)->CallNonvirtualVoidMethod(env, man, clz_human, mid);
	}

1. 当有这个类的对象的时候，使用(*env)->GetObjectClass()，相当于Java中的test.getClass()
2. 当有没有这个类的对象的时候，(*env)->FindClass()，相当于Java中的Class.forName("com.test.TestJni")

这里直接使用CallVoidMethod，虽然传进去的是父类的Method ID，但是访问的让然是子类的实现。

最后，通过CallNonvirtualVoidMethod，访问不覆盖的父类方法（C++使用virtual关键字来覆盖父类的实现），当然你也可以指定哪个父类（如果有多个父类的话）。

最后在Java中测试：

	public static void main(String[] args) {

		JniTest test = new JniTest();
		//这时候是调用子类Man的方法
		test.man.speek();
		//但是通过JNI的方式，可以访问到父类的speek方法
		test.accessNonvirtualMethod();

	}


####六、访问Java类的构造方法

Java代码如下，通过调用accessConstructor，在底层用C语言调用java.util.Date产生一个当前的时间戳，并且返回。

	//调用Date的构造函数
	public native long accessConstructor();

C代码如下：

	JNIEXPORT jlong JNICALL Java_com_test_JniTest_accessConstructor
	(JNIEnv * env, jobject jobj){
	
		jclass clz_date = (*env)->FindClass(env, "java/util/Date");
		//构造方法的函数名的格式是：<init>
		//不能写类名，因为构造方法函数名都一样区分不了，只能通过参数列表（签名）区分
		jmethodID mid_Date = (*env)->GetMethodID(env, clz_date, "<init>", "()V");;
	
		//调用构造函数
		jobject date = (*env)->NewObject(env, clz_date, mid_Date);
	
		//注意签名，返回值long的属性签名是J
		jmethodID mid_getTime= (*env)->GetMethodID(env, clz_date, "getTime", "()J");
		//调用getTime方法
		jlong jtime = (*env)->CallLongMethod(env, date, mid_getTime);
	
		return jtime;
	}

最后在Java中测试：

	public static void main(String[] args) {

		JniTest test = new JniTest();
		//直接在Java中构造Date然后调用getTime
		Date date = new Date();
		System.out.println(date.getTime());
		//通过C语音构造Date然后调用getTime
		long time = test.accessConstructor();
		System.out.println(time);

	}

####总结

属性、方法的访问的使用是和Java的反射API类似的。

###综合进阶案例——JNI返回中文乱码问题

测试乱码问题：

	public native void testChineseIn(String chinese);//传进去
	public native String testChineseOut();//取出来会乱码

	public static void main(String[] args) {
	
		//传中文进去，然后转为C字符串，直接在C层输出是没有问题的
		JniTest test = new JniTest();
		test.testChineseIn("我爱你");
		//C层将C字符串转换为JavaString然后输出，就会乱码
		System.out.println(test.testChineseOut());

	}

C代码如下：

	JNIEXPORT void JNICALL Java_com_test_JniTest_testChineseIn
	(JNIEnv * env, jobject jobj, jstring chinese){
	
		char* c_chinese = (*env)->GetStringUTFChars(env, chinese, NULL);
		printf("%s", c_chinese);
	}
	
	JNIEXPORT jstring JNICALL Java_com_test_JniTest_testChineseOut
	(JNIEnv * env, jobject jobj){
	
		char* c_str = "我爱你";
		jstring j_str = (*env)->NewStringUTF(env, c_str);
		return j_str;
	}

结果输出，其中第一条是C返回的乱码，第二条是传进去在C层打印的结果：

	ÎҰ®Ä
	我爱你

######可以看到C执行的速度要比Java快。

#####原因分析，调用NewStringUTF的时候，产生的是UTF-16的字符串，但是我们需要的时候UTF-8字符串。

解决办法，通过Java的String类的构造方法来进行字符集变换。

	JNIEXPORT jstring JNICALL Java_com_test_JniTest_testChineseOut
	(JNIEnv * env, jobject jobj){
	
		//需要返回的字符串
		char* c_str = "我爱你";
		//jstring j_str = (*env)->NewStringUTF(env, c_str);
	
		//通过调用构造方法String string = new String(byte[], charsetName);来解决乱码问题
	
		//0.找到String类
		jclass clz_String =  (*env)->FindClass(env, "java/lang/String");
		jmethodID mid = (*env)->GetMethodID(env, clz_String, "<init>", "([BLjava/lang/String;)V");
	
		//准备new String的参数：byte数组以及字符集
		//1.创建字节数组，并且将C的字符串拷贝进去
		jbyteArray j_byteArray = (*env)->NewByteArray(env, strlen(c_str));
		(*env)->SetByteArrayRegion(env, j_byteArray, 0, strlen(c_str), c_str);
		//2.创建字符集的参数，这里用Windows的more字符集GB2312
		jstring charsetName = (*env)->NewStringUTF(env, "GB2312");
	
		//调用
		jstring j_new_str = (*env)->NewObject(env, clz_String, mid, j_byteArray, charsetName);
		return j_new_str;
	
	}

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
