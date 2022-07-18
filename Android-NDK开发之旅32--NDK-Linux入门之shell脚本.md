###Shell脚本

当命令比较复杂的时候，就需要专门写一个脚本文件。

bash、b shell实质是程序，我们写的命令通过它们翻译成Linux内核能够执行的指令。

一个最简单的例子：

	#!/bin/bash
	
	#直接执行命令
	date
	who
	
	#创建变量，如果是字符串的话，有空格的一定要用双引号，否则会被解析成命令
	NDK=10
	text="i love you"
	
	#命令的执行结果作为变量的值，例如以当前时间作为文件名
	text1=`date`
	text2=$(who)
	
	#输出
	echo $NDK
	echo $text
	echo $text1
	echo $text2
	#字符串拼接，同理：有空格需要用双引号
	echo "$text very much"
	#如果需要输出$的话，需要使用转义字符
	echo "\$"

####输入输出重定向

#输出重定向

	命令的执行结果输出到文件，而不是命令行中
	ls -al > test.txt

#输入重定向

	wc命令是输出文本的行数，单词数，字节数
	wc < test.txt

######wc命令在命令行中的使用  wc << EOF 然后输入一些文本，最后输入一个EOF结束

####数学运算

#####基本运算

	#1、使用命令，输出的结果作为c的值
	#注意除号左右两边要有空格
	c=$(expr $b / $a)
	#2、使用方括号运算符代替expr命令
	d=$[$b/$a]
	
	echo "c:$c"
	echo "d:$d"

#####浮点数的处理

基本运算不支持浮点数，为了支持浮点数，需要用bc命令：

	e=$(echo "scale=4; $a / 3" | bc)
	echo "e:$e"

######其中|是管道的意思，一个命令的输出作为另外一个命令的输入
######scale是自带的内建变量

	#内联输入重定向，多次运算的时候
	f=$(bc << EOF
	scale=4
	a1=($a * $b)
	a1 / 3
	EOF
	)

####命令退出的状态

命令执行退出的状态：

1. 0 成功
2. 127 没有找到命令
3. 1 未知错误
4. 126 命令不可执行

查看与退出状态指定：

	查看上一次命令的执行状态
	echo $?

	在shell脚本中，自己指定退出的状态
	exit 状态码

####if语句

	#!/bin/bash
	testuser=lady
	#查找passwd里面是否有对应的目录
	if grep $testuser /etc/passwd 
	then
		echo "ok"
		ls -a /home/$testuser/
	#如果passwd中没有目录，再判断是否有这个目录
	elif ls -d /home/$testuser
	then
		echo "用户不存在，主目录仍然存在"
	#其他情况	
	else
		#可以继续嵌套if
		#if 
		#then
		echo "$testuser not exist"
	fi

######grep命令是查找命令，例如查找test文本在test.txt中所在的行数：grep -n test test.txt

######可以结合test命令，如果条件成立，test命令以状态为0退出，if条件成立。

test命令简单形式，用中括号，注意空格要加上，例子：

#####比较大小：

	#!/bin/bash
	a=10
	b=5
	#test命令简单形式
	if [ $a -gt $b ]
	then
		echo "$a greater than $b"
	else
		echo "$a smaller than $b"
	fi

test数值比较：

1. -gt 大于
2. -eq 等于
3. -le 小于
4. -ne 不等于

#####判空：

	#!/bin/bash
	str1=""
	if [ str1 = "" ]                                                            
	then
		echo "有内容"
	else
		echo "没内容"
	fi

test字符串比较：

1. str1 == str2
2. str1 != str2
3. str1 < str2
4. -n str1 长度是否非0
5. -z str1 长度是否为0

#####检查目录是否存在：

	#!/bin/bash
	mydir=/usr/jason
	#-d检查目录是否存在
	if [ -d $mydir ]
	then
		echo "$mydir exist"
		cd $mydir
		ls
	else
		echo "mydir not exist"
	fi

test文件比较：

1. -d 检查是否存在，并且是一个目录
2. -e 检查file是否存在
3. -f 检查是否存在，并且是一个文件
4. -r 检查是否存在，并且可读，余此类推：-w、-x
6. file1 -nt file2 file1比file2新
7. file1 -ot file2 file1比file2旧

#####多个条件的时候：

	#!/bin/bash
	#多个条件 []
	mydir=/usr/jason/shell
	#任意数学赋值或者比较表达式
	#if (( a++ > 90 ))
	if [ -d $HOME ] && [ -w $mydir ]
	then
		cd $mydir
		touch test5
	else
		echo "no"
	fi

####case语句

基本格式是：

	case命令
	case 变量 in
	pattern1) 命令;;
	pattern2) 命令;;
	*) 默认命令;;
	esac

例子：

	#!/bin/bash
	testuser=rose
	
	case $testuser in
	rose)
		echo "hi,$testuser";;
	ricky)
		echo "hello, ricky";;
	*)
		echo "defaults";;
	esac

####for循环

基本格式：

	for命令
	for var in list
	do
		命令
	done

例子1：

	#!/bin/bash
	
	for item in 1 2 3 4 5
	do
		echo $item
	done

例子2：

	#使用字符串的时候一定要注意转义字符，\#、\'等
	list="windows--linux--macos"
	#IFS字段分隔符
	IFS=$--
	for item in $list
	do
		echo $item
	done

####While循环

基本格式：

	while test command（或者[]）
	do
		命令
	done

例子：

	#!/bin/bash
	
	a=10
	while [ $a -gt 0 ]
	do
		echo "num:$a"
		#赋值不用使用$符号
		a=$[ $a - 1 ]
	done

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
