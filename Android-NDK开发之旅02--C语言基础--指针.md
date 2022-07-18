### 指针

指针是一个变量，也叫指针变量，存放的是变量的地址（内存地址，系统给数据分配的编号（门牌号）），有类型之分。

### 指针的意义

没有指针就没有Java的面向对象（OOP）。

### 指针的定义与基本使用示例

```c
void change(int* p){
    *p = 300;
}

void main(){

    int i = 90;
    int* p = &i;
    // 输出指针指向的地址
    printf("%#x\n" , p); // p的值就是i这个变量的内存地址
    
    // 通过指针间接给i赋值
    *p = 200;
    printf("%d\n" , i);

    // 可以定义一个带指针参数的函数，在函数中就可以改变i的值
    change(p);
    printf("%d\n" , i);

    system("pause");
}
```

###### 代码说明

1. 指针通过 类型* 指针名字来定义，可以赋值为变量的地址。
2. 可以通过指针间接给变量赋值。
3. 在函数传参的时候，通过指针的方式，函数对值的修改会影响到调用的地方。

### 趣味例子--游戏外挂原理

先写一个游戏，运行，初始化一个游戏时间。

```c
#include <stdlib.h>
#include <stdio.h>
#include <Windows.h>

void main(){
    // 游戏时间
    int time = 600;

    // 打印出time的地址
    printf("%#x\n",&time);

    while(time>0){
        time--;
        printf("游戏时间剩余%d秒\n",time);
        Sleep(1000); // #include <Windows.h>
    }
    system("pause");
}
```

再创建一个工程，作为外挂，去修改游戏的时间：

```c
#include <stdlib.h>
#include <stdio.h>

__declspec(dllexport) void go(){
    // 修改游戏时间
    int* p =(int*)0x55feb4; // 注意：这个地址是游戏打印出来的time的地址
    *p = 99999;
}
```

###### 代码说明

1. 添加动态库DLL的输出声明：__declspec(dllexport)。
2. 把项目的输出改为DLL，而不是EXE。（VS里面：解决方案--属性--常规）。
3. 通过DllInject.exe软件把DLL注入到游戏中，就可以发现游戏时间修改为99999了。

### 空指针NULL

```c
void main(){
    int i = 0;
    int* p = NULL;
    printf("%#x\n" , p); // 输出NULL的地址，就是0x0

    // 某些比较小的地址操作系统不允许访问（访问冲突）
    //*p =100;

    system("pause");
}
```

### 多级指针（主要讨论二级）

指针变量保存的是变量的地址，可以是一个指针变量的地址，这时候就是多级指针了。

#### 多级指针的意义

1. 动态内存分配二维数组，操作数组的时候。
2. 在jni.h中的struct JNIEnv结构体等有用到。

例子：

```
void main(){
    int a = 50;
    int* p1 = &a;
    int** p2 = &p1; // p2是一个二级指针，存放的是p1的值（即a的地址）

    printf("%d",**p2);

    system("pause");
}
```

### 指针运算（加减法）（与数组的操作相结合）

```c
void main(){
    // 数组在内存中是连续存储的
    // 指针的运算在数组遍历的时候才有意义
    // ids+i等价于&ids[i]

    // 二维数组的行指针、列指针
    //*(ids+i)等价于ids[i]

    int ids[] = {0,1,2,3,4,5};

    // 数组的首地址，下面都是一个意思
    int* p = ids;
    printf("%#x\n",ids);
    printf("%#x\n",&ids[0]);
    printf("%#x\n",&ids);
    printf("%d\n",*p);

    p++; // 指针的加减法，移动sizeof(数据类型)个字节
    printf("%d\n",*p); // 输出的是数组中的数据1

    // 通过指针给数组赋值
    // 高级写法（[]运算符的重载）
    int i =0;
    for (;i<5;i++)
    {
        ids[i] =10;
    }

    // 普通写法（利用指针）
    p=ids;
    for (;p<ids+5;p++)
    {
        *p=10;
    }

    system("pause");
}
```

### 函数指针（与Java中的回调类似）

#### 函数指针的定义与基本使用

```c
void msg(char* msg , char* title){
    // C语言里面字符串用char指针代替
    MessageBox(0,msg,title,0);
}

void main(){
    // 直接调用
    msg("内容","标题");

    // 函数指针的定义：返回类型，指针名字，参数列表（后面也可以通过typedef关键字来定义一个新的类型）
    // 参数名可以省略
    // 定义一个函数指针，指向msg函数
    void(*fun)(char* msg, char* title) = msg;

    // 使用函数指针去间接调用msg函数
    fun("我爱你","胡小盼");

    system("pause");
}
```

#### 函数指针的例子

```c
int add(int a , int b){
    return a+b;
}

int minus(int a , int b){
    return a-b;
}

void msg(int(*function_p)(int a , int b) ,int a,int b){
    // 调用函数指针的函数
    int res = function_p(a,b);
    printf("%d\n",res);
}

void main(){
    msg(add, 1, 2);
    msg(minus, 2, 1);

    system("pause");
}
```

###### 代码说明

1. 可以看到，msg函数调用的时候需要传一个函数指针，参数是两个整型，返回值是整型。msg函数的执行需要执行我们手动传入的代码（函数），从而实现了回调（注入代码）。
2. 函数指针的使用，与Java中new匿名内部类，类似Java的回调（比回调更加强大）。（只不过Java中必须要套一个类）
3. 函数指针，提高复用性，在C语音的回调机制里面非常重要。
