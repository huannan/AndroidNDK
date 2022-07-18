###C语音的字符串有两种：

1. 字符数组实现。数组可以修改其中某一个值，不可以整体赋值。
2. 字符指针实现。字符指针不可以修改其中某一个值，可以整体赋值。使用指针加法，结合结束符，可以进行截取。

####示例代码如下：

	#include <stdlib.h>
	#include <stdio.h>
	
	void main(){
	
		//使用字符数组，内存连续，可以修改（StringBuilder、Buffer）
		char str1[] = {'a','b','c','\0'};//可以不指定长度，但是需要有结束符
		char str2[6] = {'a','b','c'};//可以指定长度，后面就不会乱码
		char str3[] = "abcdeabcde";//直接用双引号
		str3[0] = 's';//可以修改其中某一个字符
	
		//字符数组不能整体赋值，只能在声明的时候整体赋值，优点是可以局部修改某一个字符。需要重新整体赋值的话，需要使用strcpy
		//str1 = "abcde";
	
		//-------------------------------------------------------
	
		//字符指针，不需要、不能修改某一个字符串（String）
		char* str4 = "abcdefg";
		//不能修改某一个字符串，否则会提示访问冲突
		//str4[0] = '7';str4++;str4 = "哈哈";
  
		//但是可以整体赋值
		str4 = "123456";
	
		//使用指针加法，截取字符串
		str4 += 3;
		while (*str4){
			printf("%c",*str4);
			str4++;
		}
		system("pause");
	}

###字符串常用的方法

在线手册：[http://www.kuqin.com/clib/](http://www.kuqin.com/clib/)

相关的头文件：#include <string.h>

####strcpy字符串赋值

 	原型：extern char *stpcpy(char *dest,char *src);
        
  	用法：#include <string.h>
  
  	功能：把src所指由NULL结束的字符串复制到dest所指的数组中。
  
  	说明：src和dest所指内存区域不可以重叠且dest必须有足够的空间来容纳src的字符串。
	
	返回：指向dest结尾处字符(NULL)的指针。

	注意：因为底层需要进行单个字符的操作，因此dest需要是一个字符数组类型，且空间必须足够大。

示例代码：

	char dest[10];
	char* src = "123";
	strcpy(dest,src);

	printf("%s", dest);

####strcat字符串拼接

	原型：extern char *strcat(char *dest,char *src);
	    
	用法：#include <string.h>
	
	功能：把src所指字符串添加到dest结尾处(覆盖dest结尾处的'\0')并添加'\0'。
	
	说明：src和dest所指内存区域不可以重叠且dest必须有足够的空间来容纳src的字符串。
	
	返回：指向dest的指针。

示例代码：

	char dest[50];	
	char *a = "china";
	char *b = " is powerful!";
	strcpy(dest, a);
	strcat(dest, b);

	printf("%s\n", dest);

####strchr字符查找

	原型：extern char *strchr(char *s,char c);
	    
	用法：#include <string.h>
	
	功能：查找字符串s中首次出现字符c的位置
	
	说明：返回首次出现c的位置的指针，如果s中不存在c则返回NULL。

示例代码：

	char *str = "I want go to USA!";
	printf("%#x\n", str);

	char* p = strchr(str, 'w');
	if (p){
		printf("索引位置：%d\n", p - str);
	}
	else{
		printf("没有找到");
	}

####strstr字符串查找

	原型：extern char *strstr(char *haystack, char *needle);
	    
	用法：#include <string.h>
	
	功能：从字符串haystack中寻找needle第一次出现的位置（不比较结束符NULL)。
	
	说明：返回指向第一次出现needle位置的指针，如果没找到则返回NULL。

示例代码：

	char *haystack = "I want go to USA!";
	char *needle = "wa";

	char* p = strstr(haystack, needle);
	if (p){
		printf("索引位置：%d\n", p - haystack);
	}
	else{
		printf("没有找到");
	}
