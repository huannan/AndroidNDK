###异常处理

与Java类似，C++也有异常处理。

###异常类型

C++中，异常的类型是任意的，如下：

	void main(){
		
		try{
			//throw 1;
			//throw "异常信息";
			throw 9.8;
		}
		catch (int a){
			cout << "int异常：" << a << endl;
		}
		catch (char* s){
			cout << "char*异常：" << s << endl;
		}
		catch (...){
			cout << "未知异常"<< endl;
		}
	
		system("pause");
	}

throw不同类型的异常，就会在相应的catch块里面捕获到。其中...能够捕获所有类型的异常，catch有先后，如果先捕获到异常，后面的catch块就不会执行了。

###不断向外抛出

异常在函数里面抛出的话，可以一层一层往外抛出，如果最终没有catch到的，程序就会停止。我们可以在函数定义的时候声明抛出的异常的类型列表。

	void fun1() throw(char*){
		throw "我是一个异常";
	}
	
	void main(){
	
		try{
			fun1();
		}
		catch (char* s){
			cout << "char*异常：" << s << endl;
		}
		system("pause");
	}

###C++标准异常与自定义异常

C++提供了一系列标准的异常类，例如下面的out_of_range：

	#include <stdexcept>

	void fun1() throw(out_of_range){
		throw out_of_range("超出范围");
	}
	
	void main(){
	
		try{
			fun1();
		}
		catch (out_of_range err){
			cout << "out_of_range异常：" << err.what() << endl;
		}
		system("pause");
	}

我们也可以自定义异常：

自定义的异常需要继承exception类。

	class MyException :public exception{
	public:
		MyException(char* msg)
			:exception(msg){
			
		}
	};
	
	void fun1() throw(MyException){
		throw MyException("自定义异常");
	}
	
	void main(){
	
		try{
			fun1();
		}

		//不适用引用的话，会产生副本
		//catch (MyException err){
		//	cout << "MyException异常：" << err.what() << endl;
		//}

		//推荐这种写法，使用引用就不会产生副本
		catch (MyException &err){
			cout << "MyException异常：" << err.what() << endl;
		}

		system("pause");
	}

下面这条语句是抛出异常指针：

	throw new MyException("自定义异常指针");

捕获的时候需要加上指针：

	catch (MyException *err){
		cout << "MyException异常：" << (*err).what() << endl;
	}

但是我们一般不推荐抛出异常指针，因为new出来的类是存在于堆内存的，我们还需要手动去delete。

###抛出内部类的异常

	class Err{
	public:
		class MyException{
		public:
			MyException(){}
		};
	};
	
	
	void fun1() throw(Err::MyException){
		throw Err::MyException();
	}

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
