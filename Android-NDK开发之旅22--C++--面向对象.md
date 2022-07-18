###前言

C++是面向对象的编程语言，因此有类的概念。下面介绍面向对象中一些比较重要的知识点。

###继承

继承主要是提高代码的重用性。

下面是一个继承的例子：

	class Human
	{
	public:
		Human(char* name, int age){
			this->name = name;
			this->age = age;
		}
		void say(){
			cout << "人说话" << endl;
		}
	
	private:
		char* name;
		int age;
	};
	
	class Woman :public Human{
	public:
		Woman(char* name, int age, float weight)
			//指定调用父类的哪一个构造函数、初始化属性对象
			:Human(name, age), m(name, age)
		{
			this->weight = weight;
		}
		void shopping(){
			cout << "购物" << endl;
		}
		void say(){
			cout << "女人说话" << endl;
		}
	private:
		float weight;
		Human m;
	};
	
	void main(){
		Woman w("haha", 20, 90.0);
		system("pause");
	}

需要注意的地方有：

1. 通过:public来继承Human。
2. 通过在自身构造的时候给父类构造函数传参。（同时给属性对象赋值）因为子类构造的时候，会先构造父类。
3. 构造与析构的调用顺序：构造的时候先调用父类的构造函数，然后调用子类的构造函数。析构的时候相反。总结就是，先构造，先释放。
4. 上面的继承中，子类覆盖了父类的say方法，但是这并不是多态（没有virtual关键字）。只是父类的say方法被隐藏了。子类可以对象调用父类的方法。

例如，我们可以通过下面的方式去指定调用父类还是子类的方法：

	void work(Human &m){
		m.say();
	}
	
	void main(){
		Woman w("haha", 20, 90.0);
	
		//调用子类的say方法（这两种实质是一样）
		w.say();
		w.Woman::say();
	
		//调用父类的say方法（显示指明，或者用子类去初始化父类的时候）
		w.Human::say();
		Human h = w;
		h.say();
		work(w);
	
		system("pause");
	}

例如Java中就无法使用被覆盖的父类方法了，但是通过JNI的方式可以访问到。

####继承的访问修饰符

主要的规律就是，两个之中的最小值。

	基类中      继承方式             子类中
	public     ＆ public继承        => public
	public     ＆ protected继承     => protected   
	public     ＆ private继承       => private
	
	protected  ＆ public继承        => protected
	protected  ＆ protected继承     => protected   
	protected  ＆ private继承       => private
	
	private    ＆ public继承        => 子类无权访问
	private    ＆ protected继承     => 子类无权访问
	private    ＆ private继承       => 子类无权访问

补充说明：

1. public关键字能够在类本身、子类、类以外的地方使用。
2. protect关键字只能够在 类本身、子类 中使用。
3. private关键字只能够在 类本身 中使用。

####多继承

多个父类通过,隔开：

	class Person{
	
	};
	
	//公民
	class Citizen{
	
	};
	
	//学生，既是人，又是公民
	class Student : public Person, public Citizen{
	
	};

####多继承的二义性问题

	class A{
	public:
		char* name;
	};
	
	class A1 : public A{
		
	};
	
	class A2 : public A{
	
	};
	
	class B : public A1, public A2{
	
	};

	void main(){
		B b;

		//这句话编译不通过，因为存在二义性
		b.name = "3";

		//B中有两个name的副本，我们可以指定使用哪个父类的name
		b.A1::name = "1";
		b.A2::name = "2";

		system("pause");
	}

![继承的二义性.png](http://upload-images.jianshu.io/upload_images/2570030-ef62616b7d78be27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

问题：B中有两个name的副本。但是我们一般情况下只需要一份。

解决办法：虚继承，不同路径继承来的同名成员只有一份拷贝，解决不明确的问题：

如果某个类B的两个父类（A1、A2）都继承于同一父类（A），那么这两个父类（A1、A2）可以通过在继承的时候加上virtual关键字来解决二义性问题：

	class A{
	public:
		char* name;
	};
	
	class A1 : virtual public A{
	
	};
	
	class A2 : virtual public A{
	
	};
	
	class B : public A1, public A2{
	
	};
	
	void main(){
		B b;
		//这时候name的副本只有一份了
		b.name = "3";
	
		system("pause");
	}

###多态

多态的作用：程序的扩展性
继承的作用：程序的重用性

多态分类：

1. 动态多态：程序运行过程中，觉得哪一个函数被调用（重写）。
2. 静态多态：重载。

这里主要讨论动态多态，动态多态的条件：

1. 继承
2. 父类的引用或者指针指向子类的对象
3. 函数的重写（需要增加virtual关键，否则的话就是覆盖）

简单的例子，注意添加virtual关键字（虚函数）：

	class Human
	{
	public:
		virtual void say(){
			cout << "人说话" << endl;
		}
	};
	
	class Woman :public Human{
	public:
		virtual void say(){
			cout << "女人说话" << endl;
		}
	};
	
	class Man :public Human{
	public:
		virtual void say(){
			cout << "男人说话" << endl;
		}
	};
	
	void say(Human &h){
		h.say();
	}
	
	void main(){
		Woman w;
		Man m;
	
		say(w);
		say(m);
	
		system("pause");
	}

####纯虚函数

纯虚函数(抽象类)

1. 当一个类具有一个纯虚函数，这个类就是抽象类
2. 抽象类不能实例化对象
3. 子类继承抽象类，必须要实现纯虚函数，如果没有，子类也是抽象类

抽象类（逻辑上就是Java中的接口）的作用：为了继承约束（新增加的类必须实现父类，提高代码重用性，保证系统的条理性、封闭性），根本不知道未来的实现（例如JDBC中的驱动就是接口）

	class Shape{
	public:
		//纯虚函数
		virtual void sayArea() = 0;
	};
	
	//圆
	class Circle : public Shape{
	public:
		Circle(int r){
			this->r = r;
		}
		void sayArea(){
			cout << "圆的面积：" << (3.14 * r * r) << endl;
		}
	private:
		int r;
	};
	
	void main(){
		//Shape s;抽象类不能创建对象
		Circle c(10);
	
		system("pause");
	}

C++中的接口是按照逻辑上区分出来的，不像Java中有专门的interface关键字。例如在C++中我们可以把只有纯虚函数的类看做一个接口：

	class Drawble{
		virtual void draw();
	};

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
