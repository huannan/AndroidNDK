### 结构体的概念、定义与初始化方式

一种综合的自定义的数据结构。

与Java中的类类似，可以有成员，包括变量和函数（函数指针）。

只有在声明的时候才能用{}进行初始化，否则只能逐一赋值。

结构体定义与初始化方式示例代码：

```c
#include <stdio.h>
#include <stdlib.h>

struct Person{
    char* name;
    int age;
    void(*speek)(); // 函数指针，类似于Java中的成员方法
};

void speek(){
    printf("说话\n");
}

void main(){

    // 结构体的初始化方式一
    struct Person p1 = {"小盼",20,speek};
    printf("我的名字是：%s，年龄：%d\n",p1.name,p1.age);
    p1.speek();
    
    // 结构体的初始化方式二
    struct Person p2;
    p2.name = "小楠";
    p2.age = 20;
    p2.speek = speek;
    printf("我的名字是：%s，年龄：%d\n",p2.name,p2.age);
    p2.speek();

    system("pause");
}
```

结构体可以在定义之后跟着声明或者初始化变量：

```c
struct Person{
    char* name;
    int age;
    void(*speek)();
}p1,p2={"小楠",14,speek};
```

### 匿名结构体

匿名结构体可以很方便控制结构体变量的个数（限量版），相当于单例模式：

```c
struct{
    char* name;
    int age;
    void(*speek)();
}p1,p2={"小楠",14,speek};
```

### 结构体的嵌套

结构体嵌套初始化的时候{}嵌套即可，或者连.操作：

```c
struct Teacher{
    char* name;
};

struct Student{
    struct Teacher t;
    char* name;
};
```

### 结构体与指针

示例代码：

```c
struct Person* p = &p2;

printf("%s,%d",(*p).name,(*p).age); // 等价于
printf("%s,%d",p->name,p->age);
```

也就是说，p->xxx 等价于 (*p).xxx

### 结构体数组

示例代码：

```c
struct Person persons[] = {{"小楠",20,speek},{"小盼",20,speek}};

// 求出结构体数组的大小：总内存大小/每一个的内存大小
int size = sizeof(persons) / sizeof(struct Person);

// 遍历方式一，通过指针去遍历
struct Person* p = persons;
for ( ; p < persons + size ; p++){
    printf("%s,%d\n",p->name,p->age);
}

// 遍历方式二，一般的数组方式去遍历
int i = 0;
for (; i < size ; i++){
    printf("%s,%d\n",persons[i].name,persons[i].age);
}
```

### 结构体的大小

字节对齐，结构体变量的大小，必须是最宽基本数据类型的整数倍。通过空间换取时间来提升读取效率。联想：刘翔跨栏。

意义：提升读取的效率。

示例代码：

```c
struct Man{
    int age;        // 4字节
    double weight;  // 8字节
}m1 = {19, 20.0};

void main(){

    // 大小应该是16
    printf("%d", sizeof(m1));

    system("pause");
}
```

### 结构体与动态内存分配

动态分配10个结构体示例代码：

```c
// 动态分配10个Man结构体
struct Man* p = (struct Man*)malloc(sizeof(struct Man) * 10);

// 初始化
int size = sizeof(p) / sizeof(struct Man);
struct Man* loop = p;
for (; loop<p+size ; loop++){
    loop->age = 20;
    loop->weight = 100.0;
}

// 释放内存
free(p);
```

### typedef取别名

typedef可以给任意类型类型取别名，typedef取别名可以定义新的类型，方便使用，书写简洁。

* 不同名称代表在干不同的事情

    ```c
    typedef int jint;
    ```
  
* 不同情况下，使用不同的别名

    ```c
    #if defined(__cplusplus)
        typedef _JNIEnv JNIEnv;
        typedef _JavaVM JavaVM;
    ```

整型取别名：

```c
typedef int Age;

// 实际使用
Age age = 10;
```

Person结构体取别名：	

```c
typedef struct Person P;

// 实际使用
P p;
```


Person结构体指针取别名：

```c
typedef struct Person* PP;

// 实际使用
PP p = &p2;
```

在结构体定义的时候取别名（对应以上两种）：

```c
typedef struct Person{
    char* name;
    int age;
}P1,*P2; // P1是结构体的别名，P2是结构体指针的别名，与变量的声明区分开（没有typedef）
```

使用的时候更加简洁，更加像面向对象编程：

```c
P1 p1;
P2 p2 = &p1;
```

### 终极例子

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <Windows.h>

// 定义一个Girl结构体，包括属性和方法
typedef struct Girl{
    char *name;
    int age;
    //函数指针
    void(*sayHi)(char*);
}Girl; // 给结构体取一个别名Girl（别名可以与结构体原本的名字相同）

// Girl结构体指针取别名GirlP
typedef Girl* GirlP;

// 结构体的成员函数
void sayHi(char* text){
    MessageBoxA(0, text, "title", 0);
}

// 自定义的一个函数
void rename(GirlP gp1){
    gp1->name = "Lily";
}

void main(){
    Girl g1 = { "Lucy", 18, sayHi };
    GirlP gp1 = &g1;
    gp1->sayHi("Byebye!");
    // 传递指针，改名（只有传递指针才能修改值，所以指针是比较常用的方式）
    rename(gp1);

    getchar();
}
```
