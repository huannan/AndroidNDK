###前言

学习的内容并不是最难，最难的是是否能够坚持下来。

###C++的类型转换

C的类型转换：在前面加括号指明转换为什么类型，或者不加，例如整数转换为浮点数。

C++类型转换分类：

1. static_cast 普遍情况。
2. const_cast 去常量。
3. dynamic_cast 子类类型、父类类型互转。
4. reinterpret_cast 函数指针转型，不具备移植性（不常用）。

作用：

1. 提高代码的可读性。
2. 降低存在的风险。

###static_cast

广泛用于类型转换的关键字。

	//这个函数有可能返回不同类型的指针
	void* fun1(int type){
		switch (type){
		case 1:	{
					int i = 9;
					return &i;
		}
		case 2:	{
					int a = 'X';
					return &a;
		}
		default:{
					return NULL;
		}
	
		}
	}
	
	void main(){
		//需要把void* 转换为 int* 或者 char*
		
		//C的写法
		int* p_i = (int*)fun1(1);
		//C++的写法，意图更加明显
		char* p_c = static_cast<char*>(fun1(2));
		system("pause");
	}

###const_cast去除常量

	void fun2(const int &i){
	
		//去除常量
		//C的写法，可读性差
		int* p_i = (int*)&i;
		*p_i = 20;
	
		//C++的写法，可读性好一些
		int* p_i_1 = const_cast<int*>(&i);
		*p_i_1 = 30;
	}

###dynamic_cast

父类、子类的转换：

	class Person{
	public:
		virtual void print(){
			cout << "人" << endl;
		}
	};
	
	class Man : public Person{
	public:
		void print(){
			cout << "男人" << endl;
		}
	
		void chasing(){
			cout << "泡妞" << endl;
		}
	};
	
	
	class Woman : public Person{
	public:
		void print(){
			cout << "女人" << endl;
		}
	
		void carebaby(){
			cout << "生孩子" << endl;
		}
	};
	
	void fun3(Person &p){
	
		//为了调用子类的特有的函数，需要转为实际的类型
	
		//C的写法，这时候并不知道转型是否失败
		//Woman* w = (Woman*)&p;
		//w->carebaby();
	
		//C++的写法，转换失败的时候返回空
		Woman* w = dynamic_cast<Woman*>(&p);
		if (w != NULL){
			w->carebaby();
		}
	
	}

###reinterpret_cast

函数指针的转换，了解即可。

	void func1(){
		cout << "func1" << endl;
	}
	
	char* func2(){
		cout << "func2" << endl;
		return "abc";
	}
	
	//定义函数指针类型
	typedef void(*f_p)();
	
	void main(){
	
		//函数指针数组
		f_p f_array[6];
		//赋值
		f_array[0] = func1;
	
		//C方式
		//f_array[1] = (f_p)(func2);
		//C++方式
		f_array[1] = reinterpret_cast<f_p>(func2);
	
		f_array[1]();
	
		system("pause");
	}

###C++的IO流

####文本文件操作

	#include <fstream>	
	
	void main(){
	
		//写二进制文件
		char* f_name = "D://test.txt";
		ofstream f_out(f_name);
		if (f_out.bad()){
			//文件打开失败
			return;
		}
		f_out << "璐宝宝" << endl;
		f_out << "我爱你" << endl;
		f_out.close();
	
		//读二进制文件
		ifstream f_in(f_name);
		if (f_in.bad()){
			return;
		}
		char ch;
		while (f_in.get(ch)){
			cout << ch;
		}
		f_in.close();
	
		system("pause");
	}

####二进制文件操作

	void main(){
		char* src = "c://src.jpg";
		char* dest = "c://dest.jpg";
	
		//输入流
		ifstream fin(src, ios::binary);
		//输出流
		ofstream fout(dest, ios::binary);
	
		if (fin.bad() || fout.bad()){
			return;
		}
	
		while (!fin.eof()){
			//读取
			char buff[1024] = {0};
			fin.read(buff,1024);
	
			//写入
			fout.write(buff,1024);
		}
	
		//关闭
		fin.close();
		fout.close();
	
		system("pause");
	}

####C++对象的持久化

	//C++对象的持久化，这是一个测试类
	class Person
	{
	public:
		Person()
		{
	
		}
		Person(char * name, int age)
		{
			this->name = name;
			this->age = age;
		}
		void print()
		{
			cout << name << "," << age << endl;
		}
	private:
		char * name;
		int age;
	};
	
	//持久化、逆持久化
	void main()
	{
		Person p1("柳岩", 22);
		Person p2("rose", 18);
		//输出流
		ofstream fout("c://c_obj.data", ios::binary);
		fout.write((char*)(&p1), sizeof(Person)); //指针能够读取到正确的数据，读取内存区的长度
		fout.write((char*)(&p2), sizeof(Person));
		fout.close();
	
		//输入流
		ifstream fin("c://c_obj.data", ios::binary);
		Person tmp;
		fin.read((char*)(&tmp), sizeof(Person));
		tmp.print();
	
		fin.read((char*)(&tmp), sizeof(Person));
		tmp.print();
	
		system("pause");
	}

###STL

####C++字符串

stl：standard template library 标准模板库，相当于Java中的util工具集。

注意这里使用的是C++的string文件，而不是string.h。

	#include <string>
	
	void main(){
		//VS的编译器自带STL，如果使用Eclipse NDK，那么就需要配置
		string s1 = "s1";
		string s2 = "s2";
	
		cout << s1 << endl;
	
		//C++字符串使用与Java的类似，并且提供了一系列方法
		string s3 = s1 + s2;
		cout << s3.at(1) << endl;
	
		//转换为C的字符串
		char* c_str = const_cast<char*>(s3.c_str());
	
		system("pause");
	}

####C++容器

容器有MAP、Vector等等。

例子：

	//容器
	#include <vector>
	
	void main(){
		//动态数组
		//不需要使用动态内存分配，就可以使用动态数组
		vector<int> v;
		v.push_back(12);
		v.push_back(10);
		v.push_back(5);
	
		for (int i = 0; i < v.size(); i++)
		{
			cout << v[i] << endl;
		}
	
		system("pause");
	}


如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
