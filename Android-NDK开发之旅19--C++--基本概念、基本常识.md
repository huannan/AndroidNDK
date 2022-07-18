###前言——C与C++的关系

1. C++可以与C代码进行混编，C++里面可以写C，但是反过来不可以。
2. C++是面向对象编程语言、C是面向过程的编程。
3. C++对C的一个增强。C++有class、引用的概念，堆内存的分配释放除了C语言的malloc、free，还有new、delete关键字。


###C++的命名空间

####基本概念

命名空间，也叫名字空间，类似于Java中包（归类）。当项目比较大的时候，用于区分不同人写的代码、不同库的代码。不同名字空间下面，函数名、变量名、类名等都可以重复。

####最基本的例子：

	#include <iostream>
	
	using namespace std;
	
	void main(){
	
		cout << "我爱你" << endl;
	
		system("pause");
	}

这是一个Hello Word的例子，需要注意的地方有：

1. 通过using namespace来使用std这个命名空间。也可以通过std::的方式来时用命名空间，::叫做访问修饰符。
2. cout是头文件iostream的输出函数，C++中头文件的包含不需要写.h后缀。其中<<是运算符重载。endl是换行的意思。

####自定义命名空间

通过namespace关键字来自定义命名空间：

	#include <iostream>
	
	using namespace std;
	
	namespace name1{
		int a = 1;
		namespace name1_1{
			int a = 3;
		}
	}
	
	namespace name2{
		int a = 2;
	}
	
	void main(){
	
		cout << name1::a << endl;
		cout << name2::a << endl;
		cout << name1::name1_1::a << endl;
		system("pause");
	}

从上面的例子中可以看到，名字空间可以嵌套。

###C++中的结构体、结构体与命名空间

1. C++是C的增强，C++中的结构体可以有访问修饰符，修饰符是多个变量或者函数共用的一个的。
2. C++中的结构体与Java中的类非常类似，在C语音中只能用函数指针当作成员函数，在C++中直接就是函数。
3. C++中的结构体有一个this指针，指向自身。
4. C++中通过using使用了命名空间之后，在使用结构体的时候，struct关键字可以省略。

	#include <iostream>
	
	using namespace std;
	
	namespace name1{
	
		struct Teacher{
		private:
			char* name;
		public:
			int age;
			void say(){
				cout << "Hello My age is :" << this->age << endl;
			}
		};
	
	}
	
	void main(){
	
		using name1::Teacher;
		Teacher t;
		t.age = 20;
	
		system("pause");
	}

###C++中的类

	#include <iostream>
	
	using namespace std;
	
	class AClass
	{
	public:
		int getA(){
			//return this->a;
			return a;
		}
	
		void setA(int a){
			this->a = a;
		}
	
	private:
		int a;
	
	};
	
	void main(){
	
		AClass c;
		c.setA(4);
		cout << c.getA() << endl;
	
		system("pause");
	}

注意：

1. 与Java类似，有一个tihs指针指向自身。
2. 类的定义末尾需要有分号。
3. 这里的main函数中的c是存在于栈内存的，可以通过new关键字在堆内存中创建对象的实体，但是我们需要

###C++中的布尔类型

C++中有布尔类型bool，C语言没有。

	#include <iostream>
	
	using namespace std;
	
	void main(){
	
		bool b = true;
		if (b){
			cout << b << endl;
			cout << sizeof(bool) << endl;
		}
	
		system("pause");
	}

bool的值有true、false，实质是1和0，占一个字节大小。

######在C/C++中，条件表达式中，大于0就为真，小于等于0就为假。而Java中只能用true和false。

###C++中的函数

####函数参数可以有默认值，与PHP一样。

	void test1(int a = 10){
	
	}
	
一旦某个参数值定了默认值，那么后面的所有参数都需要有

	void test2(int a, int b = 10, int c = 20){
	
	}

函数重载的时候需要注意二义性。

####函数的可变参数

需要使用头文件stdarg.h。

1. 通过va_start开始读取可变参数，其中形参i是最后一个固定参数。
2. 通过va_arg读取，需要指定类型。
3. 通过va_end结束读取。

	#include <iostream>
	#include <stdarg.h>
	
	using namespace std;
	
	void test(int i, ...){
	
		va_list args_p;
		va_start(args_p, i);
		//开始读取
		int a = va_arg(args_p, int);
		char b = va_arg(args_p, char);
		int c = va_arg(args_p, int);
		va_end(args_p);
	
		cout << a << endl;
		cout << b << endl;
		cout << c << endl;
	}
	
	
	void main(){
		test(1, 10, 'a', 20);
		system("pause");
	}

也可以循环读取，但是需要有条件（比如可变参数都大于0），且可变参数的类型都一样，例子：

	void test(int i, ...){
	
		va_list args_p;
		va_start(args_p, i);
		//开始读取
	
		while (true){
			int a = va_arg(args_p, int);
			if (a <= 0){
				break;
			}
			cout << a << endl;
			continue;
		}
		va_end(args_p);
	}
	
	
	void main(){
		test(1, 10, 20, 30, -1);//-1相当于结束符
		system("pause");
	}

###C++中类的写法

定义一般写在头文件中，实现在源文件中，例如：

头文件：

	#pragma once
	class Teacher
	{
	public:
		Teacher();
		~Teacher();
	};

源文件：

	#include "Teacher.h"
	
	Teacher::Teacher()
	{
	}
	
	
	Teacher::~Teacher()
	{
	}

使用：

	#include "Teacher.h"
	
	void main(){
		Teacher t;
		system("pause");
	}

###C++中的函数模板

模板函数，相当于Java中的泛型。

例如，为了实现两个东西交换，可以实现两个函数：

	void myswap(int& a,int& b){
		int tmp = 0;
		tmp = a;
		a = b;
		b = tmp;
	}
	
	void myswap(char& a, char& b){
		char tmp = 0;
		tmp = a;
		a = b;
		b = tmp;
	}
	
但是我们发现：这两个函数业务逻辑一样，数据类型不一样，因此这个时候就可以使用函数模板来减少代码量：

	template <typename T>
	void myswap(T& a, T& b){
		T tmp = 0;
		tmp = a;
		a = b;
		b = tmp;
	}
	
	void main(){
		//根据实际类型，自动推导（当然，也可以指定类型，跟Java很类似）
		int a = 10, b = 20;
		myswap<int>(a,b);
		cout << a << "," << b << endl;
	
		char x = 'v', y = 'w';
		myswap(x, y);
		cout << x << "," << y << endl;
	
		system("pause");
	}

###模板类

自定义模板类

	template<class T>
	class A{
	public:
		A(T a){
			this->a = a;
		}
	protected:
		T a;
	};
	
自定义普通类继承模板类

	class B :public A<int>{
	public:
		B(int a)
			:A<int>(a){
	
			}
	};
	
自定义模板类继承模板类

	template<class T>
	class C :public A<T>{
	public:
		C(int a)
			:A<T>(a){
	
			}
	};
	
模板类的实例化

	A<int> a(10);
	


如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
