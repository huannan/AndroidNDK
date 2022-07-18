###前言

NDK开发需要用到C/C++语言，为什么需要这两个语言？因为通过NDK开发能够解决Java做不到或者Java做的话效率、安全性会很低的问题。比如说视频处理（直播）、GIF的处理（需要对内存进行大量地分配和释放）、支付SDK（安全性）等。

学习NDK能够大大开阔我们的视野，NDK是一些大公司都要求掌握的技术，因此很有学习的必要。

######本系列介绍的是NDK开发里面会用到的C语言知识，其他的知识暂时不作介绍，要求读者最好有一定（最基本的）C语言（编程语言）基础。

###C语言的基本数据类型：

这次先来介绍C语言的基本数据类型，这里并不会从零开始介绍，而是在学习了Java的基础之上来学习，这样能够大大提高我们的效率，又能反过来更加深刻地理解Java的机制。


####C语言的基本数据类型有：
int short long float double char

####格式化输出的时候：

int %d

short %d

long %ld

float %f

double %lf

char %c

%x 十六进制

%o 八进制

%s 字符串

%#x 地址


####示例代码：

	#include <stdlib.h>
	#include <stdio.h>

	void main(){

		int i;
		printf("请输入一个整数");
		scanf("%d", &i);

		printf("%d\n",i);
		float f = 10.01;
		printf("%f\n",f);
	
		//求某个类型所占的字节数，具体跟操作系统有关
		printf("int类型所占的字节数%d\n",sizeof(int));
		printf("float类型所占的字节数%d\n",sizeof(float));
		printf("double类型所占的字节数%d\n",sizeof(double));
	
		//循环的标准写法，循环变量需要抽取出来
		int n = 0;
		for (;n<10;n++)
		{
			printf("%d\n",n);
		}
	
		//等待输入，目的是使得程序停留
		getchar();
		//也可以使用
		system("pause");

	}

####特别注意的是：

1. 程序如果没有最后一句的话，执行完就会退出了。
2. 循环的标准C写法：循环变量需要抽取出来。
3. 可以通过sizeof关键字（操作符）来求出某个数据类型所占字节数。
4. 可以通过scanf函数来进行输入，第二个参数是变量的地址。
