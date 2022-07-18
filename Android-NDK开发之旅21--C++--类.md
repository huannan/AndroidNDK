###前言

C++是面向对象的编程语言，因此有类的概念。

类的定义是在头文件，实现在源文件中，这里为了方便，都写在源文件中。


###构造函数

1. 系统会自动创建默认的无惨构造函数，但是一旦提供了有参的，默认的就会去掉了。
2. 构造有两种写法。

	class Teacher
	{
	public:
		Teacher(){
			cout << "无参构造函数" << endl;
		}
		Teacher(char* name, int age){
			cout << "有参构造函数" << endl;
			this->name = name;
			this->age = age;
		}
	
	private:
		char* name;
		int age;
	};
	
	
	void main(){
		Teacher t1 = Teacher("nan",20);
		Teacher t2("lu", 18);
		system("pause");
	}

####构造函数的初始化属性列表

现在有Student类，里面有Teacher的私有成员两个。Student构造的时候需要初始化Teacher，但是又不能直接访问Teacher类的私有属性。

因此只能通过初始化列表的方式来调用。相当于Java利用super(...)调用父类的构造函数初始化父类。

	class Teacher
	{
	public:
		Teacher(char* name, int age){
			this->name = name;
			this->age = age;
		}
		Teacher(){
		}
	private:
		char* name;
		int age;
	};
	
	class Student{
	public:
		Student(int id, char* t1_name, int t1_age, char*t2_name, int t2_age)
			:t1(t1_name, t1_age), t2(t2_name, t2_age)
		{
			//t1.name = t1_name;不可以这样赋值，因为Teacher的私有属性不可以访问
		}
	
	private:
		int id;
		Teacher t1;
		Teacher t2;
	};
	
	void main(){
		Student s(1, "wu", 20, "li", 18);
		system("pause");
	}

###析构函数

析构函数：当对象要被系统释放时（例如函数栈退出的时候），析构函数被调用
作用：善后处理，例如释放动态分配的内存。

	class Teacher
	{
	public:
		Teacher(){
			cout << "无参构造函数" << endl;
			name = (char*)malloc(sizeof(char)* 100);
			strcpy(name, "lu");
			age = 18;
		}
		~Teacher(){
			cout << "析构函数" << endl;
			free(name);
		}
	
	private:
		char* name;
		int age;
	};
	
	void fun(){
		Teacher t;
	}
	
	void main(){
		fun();
		system("pause");
	}

###拷贝构造函数

1. 系统默认的拷贝构造函数就是拷贝值的，跟下面的写法一样。
2. 拷贝构造函数被调用的场景有（实质上都是第一种的变种），有印象即可：

		一、声明时赋值（只有声明的时候才会调用）
		二、作为参数传入，实参给形参赋值（引用传值不会调用，因为指向的都是同一片内存，不存在拷贝问题）
		三、作为函数返回值返回，给变量初始化赋值

例子如下：

	class Teacher
	{
	public:
		Teacher(char* name, int age){
			cout << "有参构造函数" << endl;
			this->name = name;
			this->age = age;
		}
		Teacher(const Teacher &obj){
			cout << "拷贝构造函数" << endl;
			this->name = obj.name;
			this->age = obj.age;
		}
	
	private:
		char* name;
		int age;
	};
	
	Teacher fun(Teacher t){
		return t;
	}
	
	void main(){
		Teacher t1 = Teacher("nan", 20);
		Teacher t2 = t1;
		Teacher t3 = fun(t1);
	
		system("pause");
	}

####浅拷贝

默认的拷贝构造函数实现是浅拷贝，即值拷贝，如下所示：

	class Teacher
	{
	public:
		Teacher(){
			this->name = (char*)malloc(sizeof(char)* 100);
			strcpy(name, "lu");
			this->age = age;
		}
		~Teacher(){
			cout << "析构函数" << endl;
			free(name);
		}
		Teacher(const Teacher &obj){
			cout << "拷贝构造函数" << endl;
			this->name = obj.name;
			this->age = obj.age;
		}
	
	private:
		char* name;
		int age;
	};
	
	void fun(){
		Teacher t1;
		Teacher t2 = t1;
	}
	
	void main(){
		fun();
		system("pause");
	}

值拷贝带来的问题：

如果有动态内存分配的时候，如果单纯是值拷贝，析构的时候就会析构两次。

	例如代码中的char* name，t1、t2都指向了同一个地址，那么析构的时候就会free两次，因为一个内存地址不能释放两次，因此会触发异常中断。

如下图所示：

![浅拷贝的问题.png](http://upload-images.jianshu.io/upload_images/2570030-5cb16cdcadd246f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####深拷贝

重写覆盖默认的浅拷贝，自己写一个深拷贝：

t2的name是新分配的内存，因此t1、t2指向的是两片不同的内存，因此析构的时候分别释放，互不干扰，解决了浅拷贝的问题。

	class Teacher
	{
	public:
		Teacher(){
			this->name = (char*)malloc(sizeof(char)* 100);
			strcpy(name, "lu");
			this->age = age;
		}
		~Teacher(){
			cout << "析构函数" << endl;
			free(name);
		}
		Teacher(const Teacher &obj){
			cout << "拷贝构造函数" << endl;
			int len = strlen(obj.name);
			this->name = (char*)malloc(sizeof(char)* (len + 1));//+1是因为结束符
			strcpy(this->name, obj.name);
			this->age = obj.age;
		}
	
	private:
		char* name;
		int age;
	};
	
	void fun(){
		Teacher t1;
		Teacher t2 = t1;
	}
	
	void main(){
		fun();
		system("pause");
	}

如下图所示：

![深浅拷贝的区别.png](http://upload-images.jianshu.io/upload_images/2570030-5d6e40c96cf77f60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###动态内存分配

1. C++中使用new和delete进行动态内存分配，这时候构造和析构会调用。
2. C中使用malloc和free进行动态内存分配，构造和析构不会调用。

例子：

	void main(){
		cout << "C++中使用new和delete进行动态内存分配" << endl;
		Teacher* t1 = new Teacher("wu", 20);
		delete t1;
	
		cout << "C中使用malloc和free进行动态内存分配" << endl;
		Teacher* t2 = (Teacher*)malloc(sizeof(Teacher));
		free(t2);
		system("pause");
	}

数组的话，delete的时候需要加上[]：

	int* arr1 = (int*)malloc(sizeof(int)* 10);
	arr1[0] = 1;
	free(arr1);

	int* arr2 = new int[10];
	arr2[0] = 1;
	delete[] arr2;

###类的静态属性和函数

1. 类的静态属性必须放在全局的地方初始化。
2. 非静态、静态函数可以访问静态属性，但是静态函数只能访问静态属性。（跟Java一样）
3. 静态的属性和方法可以直接通过 类名:: 来访问（如果是public的话）。也可以通过 对象:: 来访问。

例子：

	class Test
	{
	public:
		void m1(){
			count++;
		}
		static void m2(){
			count++;
		}
	private:
		static int count;
	};
	
	//静态属性初始化
	int Test::count = 0;
	
	void main(){
		//Test::count
		Test::m2();
	
		system("pause");
	}

###类的大小（sizeof）

C++的内存分区：

	栈
	堆
	全局（静态、全局）
	常量区（字符串）
	程序代码区

结构体有字节对齐的概念，那么类的普通属性与结构体相同的内存布局。

例如下面的结构体大小是32 = 4 x 8：

	struct C{
	public:
		int i;//8个字节（由于字节对齐，与double一样为8个字节）
		int j;//8个字节
		int k;//8个字节
		double x;//8个字节
		void(*test)();//指针是4个字节，但是存放在代码区
	};

例如下面的类的大小是24（这里没搞懂，先放一放，以后回来再补）

	class A{
	public:
		int i;//8个字节（由于字节对齐，与double一样为8个字节）
		int j;//8个字节
		int k;//8个字节
		double t;//8个字节
	
		static int m;//4个字节，但是存在于全局静态区
		void test(){//4个字节，相当于函数指针的写法，但是存放在代码区
			cout << "打印" << endl;
		}
	};

###类的this指针常函数

1. const写在函数后面。
2. 此时的const修饰的是this指针，this指针本类就是指针常量，不能修改值。
3. 此处增加了const的修饰之后，tihs指针更加是一个常量指针，指针指向的值不能修改。

例子：

	class Teacher{
	private:
		char* name;
		int age;
	public:
		Teacher(char* name, int age){
			this->name = name;
			this->age = age;
		}
		//常函数，修饰的是this
		//既不能改变指针的值，又不能改变指针指向的内容
		//const Teacher* const this
		void myprint() const{
			printf("%#x\n", this);
			//改变属性的值
			//this->name = "yuehang";
			//改变this指针的值
			//this = (Teacher*)0x00009;
			cout << this->name << "," << this->age << endl;
		}
	};

this，当前对象的指针

C++函数是共享的（多个对象共享代码区中的一个函数），必须要有能够标识当前对象是谁的办法，那就是this指针。

![函数共享.png](http://upload-images.jianshu.io/upload_images/2570030-185392694bb19517.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

例如我们用结构体去模拟类的时候，就需要自定义this指针了。所以说，在JNI开发的时候，JniEnv在C语音里面是二级指针（结构体实现，C语言中结构体没有tihs指针），在C++中是一级指针（类实现，类有this指针）

![C++编译器对普通成员函数的内部处理.png](http://upload-images.jianshu.io/upload_images/2570030-8708d8b72fb489bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

拓展：

Java内存分区：

1. JVM Stack（基本数据类型、对象引用）
2. Native Method Stack（本地方法栈）
3. 方法区
4. 程序计数区
5. 直接内存

热修复的基本概念：拿到错误的函数的指针，指向正确的函数的指针。（两个函数都已经加载的情况下）

###友元函数、友元函数

主要作用：类、类与函之间的数据共享。比如类已经写好了，不想去改动类的结构（比如访问权限，那么最好的办法就是使用友元）

在友元函数中可以访问类的私有的属性。

例如，fun是类A的友元函数，因此fun中可以访问A的私有属性。

	class Test{
		friend void fun(Test &t, int i);//友元函数声明
	public:
		Test(int i){
			this->i = i;
		}
	private:
		int i;
	};
	
	//友元函数实现，可以是全局的，也可以是其他类的
	void fun(Test &t, int i){
		t.i = i;
	}
	
	void main(){
		Test t(10);
		fun(t, 20);
		system("pause");
	}

同理，B是A的友元类，因此B可以访问A的私有属性：

	class A{
		friend class B;
	private:
		int i;
	};
	
	class B
	{
	public:
		void test(){
			a.i = 20;
		}
	private:
		A a;
		int i;
	};

拓展：在Java（Java是C++写的嘛）中Class就可以理解为Object的“友元类”，Class类就可以访问Object的任何成员。

###运算符重载

运算符重载的本质还是函数的调用。

现在有一个类：

	class Point
	{
	public:
		Point(int x, int y){
			this->x = x;
			this->y = y;
		}
	
	public:
		int x;
		int y;
	};

需要重载“+”：

	Point operator+(Point &p1, Point &p2){
		return Point(p1.x + p2.x, p2.y + p2.y);
	}

注意，如果Point的x、y属性都是私有的话，就需要用到友元函数了，例如类已经写好了，不想破坏类的结构：

	class Point
	{
		friend Point operator+(Point &p1, Point &p2);
	public:
		Point(int x, int y){
			this->x = x;
			this->y = y;
		}
	
	private:
		int x;
		int y;
	};
	
	Point operator+(Point &p1, Point &p2){
		return Point(p1.x + p2.x, p2.y + p2.y);
	}

如果是在类内部的运算符重载，因为有this指针，重载的时候可以省略一个参数，类可以直接访问自己的私有成员：

	class Point
	{
	public:
		Point(int x, int y){
			this->x = x;
			this->y = y;
		}
	
		Point operator+(Point &p){
			return Point(this->x + p.x, this->y + p.y);
		}
	
	private:
		int x;
		int y;
	};



如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
