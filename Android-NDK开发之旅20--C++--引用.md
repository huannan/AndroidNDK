###C++的引用

变量就是内存的“门牌号”，人为地取名字，因此可以有多个别名，而这种别名就是引用。

######引用的主要作用：作为函数的参数或者返回值，代替指针，使得程序可读性加强。

1. 单纯给变量取别名没有任何意义，作为函数参数传递，能保证参数传递过程中不产生副本。
2. 引用可以直接操作变量，指针要通过取值(*p)，间接操作变量，指针的可读性差。
3. 引用的大小（sizeof）跟类型一样大。

	#include <iostream>
	
	using namespace std;
	
	void main(){
	
		int a = 10;
		int &b = a;
	
		b = 20;
		cout << b << endl;
	
		system("pause");
	}

例子中，变量a和引用b实质是访问同一个内存地址。跟指针的概念差不多。

值交换的例子：

	#include <iostream>
	
	using namespace std;
	
	//使用指针交换值
	void swap1(int *a, int *b){
		int c = *a;
		*a = *b;
		*b = c;
	}
	
	//使用引用交换值，不用写那么多“*”了
	void swap2(int &a, int &b){
		int c = a;
		a = b;
		b = c;
	}
	
	void main(){
	
		int a = 10;
		int b = 20;
	
		swap1(&a, &b);
		swap2(a, b);
	
		cout << a << endl;
		cout << b << endl;
	
		//找到最大的数，此时为b，然后b赋值为30
		//C++的三目运算符可以进行赋值
		a > b ? a : b = 30;
		cout << b << endl;
	
		system("pause");
	}


####指针的引用，可以代替二级指针

	struct Teacher{
		char* name;
	};
	
	void set1(Teacher** a){
		(*a)->name = "wu";
	}
	
	void set2(Teacher* &a){
		a->name = "lu";
	}
	
	void main(){
		Teacher t;
		Teacher* p_t = &t;
		set1(&p_t);
		cout << t.name << endl;
	
		set2(p_t);
		cout << t.name << endl;
	
		system("pause");
	}

####指针常量与常量指针

指针常量，指针的常量，不改变地址的指针，但是可以修改它指向的内容：

	int* const p1 = &a;
	//p1 = &b;
	*p1 = 30;

常量指针，指向常量的指针，指向的内容不能修改，可以改变地址

	const int* p2 = &a;
	p2 = &b;
	//*p2 = 30;

####常引用

1. 引用必须有值，不为空，指针就不一定。使用指针的时候就要注意判断非空了。
2. 常引用，不能再进行赋值了，相当于final。

	void set(const int &a){
		//引用必须有值，不为空，指针就不一定
		//const的意思是在函数内部不能修改值
		//a = 8;
	}
	
	void main(){
	
		int x = 0;
	
		//引用必须有值，不能为空
		//int &a = NULL;
	
		//常引用，不能再进行赋值了，相当于final
		const int &a = x;
	
		const int &b = 1;//字面量，了解一下即可
	
		system("pause");
	}

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
