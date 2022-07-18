###POSIX

POSIX是一种标准，例如有多线程编程标准、网络编程标准等。

####POSIX多线程

Linux下，一般多线程的实现由POSIX多线程编程实现。Android系统属于Linux系统，因此NDK原生支持POSIX多线程编程。

Windows平台一般用Windows自带的API。

####Visual Studio平台搭建POSIX多线程环境

因为POSIX多线程是Linux的，因此如果需要在Visual Studio下开发，需要搭建可开发环境。

1. 首先需要下载POSIX，地址为：[ftp://sourceware.org/pub/pthreads-win32/pthreads-w32-2-9-1-release.zip](ftp://sourceware.org/pub/pthreads-win32/pthreads-w32-2-9-1-release.zip "ftp://sourceware.org/pub/pthreads-win32/pthreads-w32-2-9-1-release.zip")
2. 创建VS空项目
3. 添加包含目录：pthreads-w32-2-9-1-release\Pre-built.2\include以及pthreads-w32-2-9-1-release\pthreads.2
4. 添加库目录：pthreads-w32-2-9-1-release\Pre-built.2\lib\x86
5. 添加附加依赖项：pthreadVC2.lib
6. 把pthreads-w32-2-9-1-release\Pre-built.2\dll\x86下面的动态库文件复制到工程的正确位置。

POSIX的编译：

1. VS平台中直接编译运行即可。
2. 在Linux平台中采用gcc编译（先编译生成目标文件然后链接生成可执行程序）：gcc test.c -o test -lpthread，执行：./test。

POSIX帮助文档的查看：

1. 在Linux系统中，安装POSIX帮助文档：sudo apt-get install manpages-posix-dev
2. 列出所有函数man -k pthread；查看某个函数：man pthread_create

####创建线程与线程的结束

	#include <stdlib.h>
	#include <stdio.h>
	
	//必须引入的头文件
	#include "pthread.h"
	
	//一个相当于Java的run方法
	void *start_fun(void* arg){
	
		//得到线程创建的参数
		char* no = (char*)arg;
		int i = 0;
		for (; i < 10; i++){
			printf("%s thread : %d \n", no, i);
			if (i == 5){
				//线程自杀，需要返回参数
				pthread_exit((void*)2);
				//线程他杀
				//pthread_cancel()
			}
		}
	
		//run方法执行完，线程结束，返回
		return (void*)1;
	}
	
	void main(){
	
		printf("main thread\n");
	
		pthread_t thread;
		//创建线程，指定run方法，并且可以传入参数，在run方法的arg中可以取出
		pthread_create(&thread, NULL, start_fun, "no1");
	
		void* r_val;
		//等待线程结束，获取线程返回参数
		pthread_join(thread, &r_val);
		printf("return value : %d", (int)r_val);
	
		system("pause");
	}

在代码中：

1. 通过pthread_create创建线程，需要传入一个函数指针，相当于Java线程中的run方法。然后还需要传参，参数可以在run方法中取出。
2. 线程被创建以后，就会执行“run”方法，该方法中可以拿到线程创建的参数，可以自杀掉线程。线程的结束需要参数。
3. 可以通过pthread_join方法等待线程结束，并且可获取线程结束的参数。


####锁

我们创建两个线程：

	#include <stdlib.h>
	#include <stdio.h>
	#include "pthread.h"
	#include <Windows.h>
	
	int i = 0;
	
	//一个相当于Java的run方法
	void *start_fun(void* arg){
	
		//得到线程创建的参数
		char* no = (char*)arg;
		for (; i < 10; i++){
			Sleep(10);
			printf("%s thread : %d \n", no, i);
		}
		i = 0;
	
		//run方法执行完，线程结束，返回
		return (void*)1;
	}
	
	void main(){
	
		printf("main thread\n");
	
		pthread_t thread1;
		pthread_t thread2;
		//创建线程，指定run方法，并且可以传入参数，在run方法的arg中可以取出
		pthread_create(&thread1, NULL, start_fun, "no1");
		pthread_create(&thread2, NULL, start_fun, "no2");
	
		void* r_val1;
		void* r_val2;
		//等待线程结束，获取线程返回参数
		pthread_join(thread1, &r_val1);
		pthread_join(thread2, &r_val2);
		printf("return value : %d\n", (int)r_val1);
		printf("return value : %d\n", (int)r_val2);
	
		system("pause");
	}

打印的结果如下：

	main thread
	no2 thread : 0
	no1 thread : 0
	no2 thread : 2
	no1 thread : 2
	no1 thread : 4
	no2 thread : 4
	no1 thread : 5
	return value : 2
	return value : 2
	请按任意键继续. . .

可见，两个线程是并行执行的。i变量同时被两个线程访问。但是我们现在要求线程1先执行完，然后才到线程2执行，那么两个线程i的所有情况都会被打印出来。

这时候我们需要使用互斥锁：

	#include <stdlib.h>
	#include <stdio.h>
	#include "pthread.h"
	#include <Windows.h>
	
	int i = 0;
	
	//互斥锁
	pthread_mutex_t m;
	
	//一个相当于Java的run方法
	void *start_fun(void* arg){
	
		//加锁
		pthread_mutex_lock(&m);
	
		//得到线程创建的参数
		char* no = (char*)arg;
		for (; i < 10; i++){
			Sleep(10);
			printf("%s thread : %d \n", no, i);
		}
		i = 0;
	
		//解锁
		pthread_mutex_unlock(&m);
	
		//run方法执行完，线程结束，返回
		return (void*)1;
	}
	
	void main(){
	
		printf("main thread\n");
	
		//初始化互斥锁
		pthread_mutex_init(&m, NULL);
	
		pthread_t thread1;
		pthread_t thread2;
		//创建线程，指定run方法，并且可以传入参数，在run方法的arg中可以取出
		pthread_create(&thread1, NULL, start_fun, "no1");
		pthread_create(&thread2, NULL, start_fun, "no2");
	
		void* r_val1;
		void* r_val2;
		//等待线程结束，获取线程返回参数
		pthread_join(thread1, &r_val1);
		pthread_join(thread2, &r_val2);
		printf("return value : %d\n", (int)r_val1);
		printf("return value : %d\n", (int)r_val2);
	
		//销毁互斥锁
		pthread_mutex_destroy(&m);
	
		system("pause");
	}

输出的结果如下：

	main thread
	no1 thread : 0
	no1 thread : 1
	no1 thread : 2
	no1 thread : 3
	no1 thread : 4
	no1 thread : 5
	no1 thread : 6
	no1 thread : 7
	no1 thread : 8
	no1 thread : 9
	no2 thread : 0
	no2 thread : 1
	no2 thread : 2
	no2 thread : 3
	no2 thread : 4
	no2 thread : 5
	no2 thread : 6
	no2 thread : 7
	no2 thread : 8
	no2 thread : 9
	return value : 1
	return value : 1
	请按任意键继续. . .

在代码中：

1. 我们通过pthread_mutex_init初始化了一把互斥锁，最后通过pthread_mutex_destroy进行销毁。
2. 在线程执行的时候，我们可以通过pthread_mutex_lock、pthread_mutex_unlock进行加锁和解锁。
3. 使用互斥锁可以解决线程死锁（ABBA）的问题。


互斥锁是先让一个线程做完，然后另外一个线程做。还有一种情况就是，一个线程先执行，生产，然后另外一个线程就会去消费。

其实视频解码的绘制使用的就是生产者--消费者的模式。图片的下载显示也是基于这种模式。比如说我们生产者生成的产品，放到一个队列里面，当生产者生产出产品的时候就会发送信号通知消费者去消费，例如RTMP推流的时候，我们本地采集音视频的时候就需要一种队列，因为本地的压缩比网络上传要快。

使用这一种模式，就需要条件变量。例子：

	#include <stdlib.h>
	#include <stdio.h>
	#include "pthread.h"
	#include <Windows.h>
	
	//模拟产品队列
	int productNum = 0;
	
	//互斥锁
	pthread_mutex_t m;
	//条件变量
	pthread_cond_t c;
	
	void *produce(void* arg){
	
		char* no = (char*)arg;
	
		for (;;){
			//加锁
			pthread_mutex_lock(&m);
	
			//生产者生产产品
			productNum++;
			printf("%s生产产品：%d\n", no, productNum);
			//通知消费者进行消费
			pthread_cond_signal(&c);
	
			//解锁
			pthread_mutex_unlock(&m);
	
			Sleep(100);
		}
		return (void*)1;
	}
	
	void *comsume(void* arg){
	
		char* no = (char*)arg;
	
		for (;;){
			pthread_mutex_lock(&m);
			//使用while是为了防止惊群效应唤醒条件变量
			while (productNum == 0){
				//1.没有产品可以消费，等待生产者生产，即等待条件变量被唤醒
				//2.释放互斥锁，使得其他消费者可以进来等待
				//3.被唤醒的时候，解除阻塞，重新申请获得互斥锁，保证只有一个消费者消费
				pthread_cond_wait(&c, &m);
			}
			productNum--;
			printf("%s消费者消费产品：%d\n", no, productNum);
			pthread_mutex_unlock(&m);
			Sleep(1000);
		}
		return (void*)1;
	}
	
	void main(){
	
		printf("main thread\n");
	
		//初始化互斥锁
		pthread_mutex_init(&m, NULL);
		//初始化条件变量
		pthread_cond_init(&c, NULL);
	
		pthread_t thread_producer;
		pthread_t thread_comsumer;
		//创建线程，指定run方法，并且可以传入参数，在run方法的arg中可以取出
		pthread_create(&thread_producer, NULL, produce, "producer");
		pthread_create(&thread_comsumer, NULL, comsume, "comsumer");
	
		//等待线程结束，获取线程返回参数
		pthread_join(thread_producer, NULL);
		pthread_join(thread_comsumer, NULL);
	
		//销毁互斥锁
		pthread_mutex_destroy(&m);
		//销毁条件变量
		pthread_cond_destroy(&c);
		system("pause");
	}

输出的结果如下：

	main thread
	producer生产产品：1
	comsumer消费者消费产品：0
	producer生产产品：1
	producer生产产品：2
	producer生产产品：3
	producer生产产品：4
	producer生产产品：5
	producer生产产品：6
	producer生产产品：7
	producer生产产品：8
	producer生产产品：9
	comsumer消费者消费产品：8
	producer生产产品：9
	producer生产产品：10
	producer生产产品：11
	producer生产产品：12
	producer生产产品：13
	producer生产产品：14
	producer生产产品：15
	producer生产产品：16
	producer生产产品：17
	comsumer消费者消费产品：16

这里我通过Sleep的方式控制了生产者与消费者的效率，一般来说生产的速度要比消费的速度快。

上面是只有一个生产者和一个消费者的示例代码。一般开说，生产者和消费者都会有多个。这里我们通过线程数组的方式来实现。

示例代码如下：

	#include <stdlib.h>
	#include <stdio.h>
	#include "pthread.h"
	#include <Windows.h>
	
	#define NUM_PRODUCER 2
	#define NUM_COMSUMER 2
	pthread_t threads[NUM_PRODUCER + NUM_COMSUMER];
	
	int productNum = 0;
	
	//互斥锁
	pthread_mutex_t m;
	//条件变量
	pthread_cond_t c;
	
	void *produce(void* arg){
	
		int no = (int)arg;
	
		for (;;){
			//加锁
			pthread_mutex_lock(&m);
	
			//生产者生产产品
			productNum++;
			printf("%d生产产品：%d\n", no, productNum);
			//通知消费者进行消费
			pthread_cond_signal(&c);
	
			//解锁
			pthread_mutex_unlock(&m);
	
			Sleep(100);
		}
		return (void*)1;
	}
	
	void *comsume(void* arg){
	
		int no = (int)arg;
	
		for (;;){
			pthread_mutex_lock(&m);
			while (productNum == 0){
				//没有产品可以消费，等待生产者生产
				pthread_cond_wait(&c, &m);
			}
			productNum--;
			printf("%d消费者消费产品：%d\n", no, productNum);
			pthread_mutex_unlock(&m);
			Sleep(1000);
		}
		return (void*)1;
	}
	
	void main(){
	
		printf("main thread\n");
	
		//初始化互斥锁
		pthread_mutex_init(&m, NULL);
		//初始化条件变量
		pthread_cond_init(&c, NULL);
	
		int i = 0;
		//创建生产者线程
		for (i = 0; i < NUM_PRODUCER; i++){
			pthread_create(&threads[i], NULL, produce, (void*)i);
		}
	
	
		//创建消费者线程
		for (i = 0; i < NUM_COMSUMER; i++){
			pthread_create(&threads[NUM_PRODUCER + i], NULL, comsume, (void*)i);
		}
	
		//等待线程结束，获取线程返回参数
		for (i = 0; i < NUM_PRODUCER + NUM_COMSUMER; i++){
			pthread_join(threads[i], NULL);
		}
	
		//销毁互斥锁
		pthread_mutex_destroy(&m);
		//销毁条件变量
		pthread_cond_destroy(&c);
		system("pause");
	}

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
