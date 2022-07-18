### 联合体（共用体）

不同类型的变量共同占用一段内存（相互覆盖），联合变量任何时刻只有一个成员存在，节省内存。

联合体变量的大小=最大的成员所占的字节数（字节对齐 ）

比喻：同穿一条裤子

```c
union MyValue{
    int x;
    int y;
    double z;
};

void main(){
    union MyValue v;
    v.x = 90;
    v.y = 100;
    v.z = 23.8; // 最后一次赋值有效

    printf("%d,%d,%lf\n", v.x, v.y, v.z);
    system("pause");
}
```

JNI头文件中的联合体：

```c
typedef union jvalue {
    jboolean    z;
    jbyte       b;
    jchar       c;
    jshort      s;
    jint        i;
    jlong       j;
    jfloat      f;
    jdouble     d;
    jobject     l;
} jvalue;
```

### 枚举（enumeration）

枚举（列举所有的情况），限定值的取值范围，保证取值的安全性。

```c
enum Day{
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday
};

void main(){
    enum Day d = Tuesday;

    // 打印d的地址以及序号
    printf("%#x,%d\n", &d, d);
    system("pause");
}
```
