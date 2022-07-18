### 读写文本文件

主要用到了fgets和fputs两个函数（函数名中的s是指String，字符串的意思）：

```c
#define _CRT_SECURE_NO_WARNINGS
#include<stdio.h>
#include<stdlib.h>

void main(){
    
    // 文件路径
    char* path_read = "D:\\test_read.txt";
    char* path_write = "D:\\test_write.txt";

    // 打开文件，返回文件的指针
    FILE* f_read = fopen(path_read, "r");
    FILE* f_write = fopen(path_write, "w");

    // 文件是否成功打开
    if (f_read == NULL || f_write == NULL){
        printf("文件打开失败");
        return;
    }

    // 读取文件
    char buff[50]; // 缓冲区
    while (fgets(buff, 50, f_read)){
        // 写文件
        fputs(buff, f_write);
    }

    // 关闭文件
    fclose(f_read);
    fclose(f_write);

    system("pause");
}
```

###### 小知识

_CRT_SECURE_NO_WARNINGS用于Visual Studio2013编译去掉警告。

### 读写二进制文件

计算机的文件存储在物理上都是二进制，文本文件和二进制之分，其实是一个人为的逻辑之分。

C读写文本文件与二进制文件的差别仅仅体现在回车换行符：

1. 写文本时，每遇到一个'\n'，会将其转换成'\r\n'(回车换行)。
2. 读文本时，每遇到一个'\r\n'，会将其转换成'\n'。
3. 但是读写二进制文件的时候并不会做以上转换。

下面是二进制文件读写的例子（图片的复制）：

主要用到了fread和fwrite两个函数：

```c
#define _CRT_SECURE_NO_WARNINGS
#include<stdio.h>
#include<stdlib.h>

//Ctrl k d

void main(){

    // 文件路径
    char* path_read = "D:\\test_read.png";
    char* path_write = "D:\\test_write.png";

    // 打开文件，返回文件的指针，b代表是二进制文件
    FILE* f_read = fopen(path_read, "rb");
    FILE* f_write = fopen(path_write, "wb");

    // 文件是否成功打开
    if (f_read == NULL || f_write == NULL){
        printf("文件打开失败");
        return;
    }

    // 读取文件
    int buff[50]; // 缓冲区，注意，读写二进制文件的时候，是用int类型的缓冲区
    int len = 0;  // 每次读取到的长度
    // 读到的内容放到缓冲区，缓冲区的单位大小，缓冲区大小（一个性读50个int的大小（4字节）的数据）
    // 也就是一次读50 X 4 200字节的数据
    // 返回的len是读取到的长度，小于等于50
    while ((len = fread(buff, sizeof(int), 50, f_read)) != 0){
        //写文件
        fwrite(buff, sizeof(int), len, f_write);
    }

    //关闭文件
    fclose(f_read);
    fclose(f_write);

    system("pause");
}
```

###### 注意

1. 缓冲区用int类型的数组。
2. 文件读写模式后面加上b代表读写二进制文件。

### 获取文件大小

主要用到了fseek和ftell函数：

```c
#define _CRT_SECURE_NO_WARNINGS
#include<stdio.h>
#include<stdlib.h>

void main(){

    // 文件路径
    char* path = "D:\\test.png";

    // 打开文件，返回文件的指针，b代表是二进制文件
    FILE* f_read = fopen(path, "rb");

    // 文件是否成功打开
    if (f_read == NULL){
        printf("文件打开失败");
        return;
    }

    // 类似于多线程下载的概念，首先将文件长度按N段分，然后将每段文件读取并写入到相应的临时文件
    fseek(f_read, 0L, SEEK_END); // seek到文件的结尾，0L代表向前偏移几个字节
    long len = ftell(f_read); // 返回当前的文件指针相对于文件开头的位移量
    printf("%ld", len);

    // 关闭文件
    fclose(f_read);

    system("pause");
}
```

### 文件加密、解密

用简单的异或运算进行加密，解密的话就是一个逆过程。

规则：1^1=0, 0^0=0, 1^0=1, 0^1=1 同为0，不同为1

```c
void crpypt(char* filePath, char* crpyptFilePath){

    // 打开文件，返回文件的指针
    FILE* f_read = fopen(filePath, "r");
    FILE* f_write = fopen(crpyptFilePath, "w");

    // 文件是否成功打开
    if (f_read == NULL || f_write == NULL){
        printf("文件打开失败");
        return;
    }

    int ch; // 一次读取一个字符，每一个字符都是一个数据，用int来表示
    // EOF End Of File = -1
    while ((ch = fgetc(f_read)) != EOF){
        fputc(ch ^ 9, f_write); // 用异或运算进行加密
    }

    // 关闭文件
    fclose(f_read);
    fclose(f_write);
}

void main(){

    // 文件路径
    char* path_source = "D:\\test_source.txt";
    char* path_crpypt = "D:\\test_crpypt.txt";

    // 文件加密crpypt
    crpypt(path_source, path_crpypt);

    // 文件解密decrpypt，是一个逆过程，注意先把原来的文件删除
    crpypt(path_crpypt, path_source);

    system("pause");
}
```

### 二进制文件加解密

```c
// 二进制文件加解密
// 读取二进制文件中的数据时，一个一个字符读取
// 密码：password
void crpypt(char normal_path[], char crypt_path[], char password[]){
    // 打开文件
    FILE *normal_fp = fopen(normal_path, "rb");
    FILE *crypt_fp = fopen(crypt_path, "wb");
    //一次读取一个字符
    int ch;
    int i = 0; // 循环使用密码中的字母进行异或运算
    int pwd_len = strlen(password); // 密码的长度
    while ((ch = fgetc(normal_fp)) != EOF){ // End of File
        // 写入（异或运算）
        fputc(ch ^ password[i % pwd_len], crypt_fp);
        i++;
    }
    // 关闭
    fclose(crypt_fp);
    fclose(normal_fp);
}
```

核心逻辑通过下面的代码进行加密：

```c
ch ^ password[i % pwd_len]
```

其中：

```c
int pwd_len = strlen(password); // 密码的长度
```

二进制文件记得加上小b，否则加密以后文件不能正常打开了，因为文件损坏了。

微信的数据库是加密的，用C语言（动态库so反编译很难）加密。不会用Java去做，因为安全性不够，Java的反编译比较容易。

想往上发展，NDK、Linux还是必须的。

### 文件的分割合并，待续。。。
