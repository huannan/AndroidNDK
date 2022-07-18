###常用的Linux命令

shell命令有多种，Ubuntu默认使用 B shell。

####命令行相关的命令

命令自动补全：

	按tab

清空或者重置命令行，reset是完全清空，clear实质只是翻下一个空白页：

	reset
	clear

####文件操作相关命令

列出目录下的文件（夹），-l是带访问权限，-a是all显示所有隐藏文件的意思，可选：

	ls -la

通配符匹配，*是多个字符，?是单个字符：

	ls *.txt

切换目录：

	cd 路径

回到上级目录，最后面的/可以省略，..代表上级目录，.代表当前目录：

	cd ..（/）

显示当前目录：

	pwd

创建目录：

	mkdir 目录名字

创建文件：

	touch test.txt

目录以及子目录一系列文件都删除掉，-rf是递归强制删除的意思，可选：

	rm -rf 目录

文件复制：

	文件拷贝
	cp src.txt dest.txt

	文件拷贝到指定目录
	cp src.txt /home/wuhuannan/

	如果所在路径文件已经存在，-i参数会提示你是否覆盖已有的文件
	cp -i src.txt /home/wuhuannan/

	复制整个目录（递归）
	cp test/ -R Downloads/

####文件的查看与编辑

####使用自带的命令

查看文件类型：

	file src.txt

查看文件内容，-n是显示行号的意思，可选：

	cat -n src.txt

	只看前3行
	head -n 3 src.txt

	只看后3行
	tail -n 3 src.txt

####软件的安装与卸载命令

安装与卸载软件（用vim为例子）:

	sudo apt-get install vim-gtk
	sudo apt remove vim

更新软件源：

	apt-get update

####用户与分组

创建与删除用户：

	创建用户，同时也会创建Home目录
	sudo useradd -m lubaobao
	
	删除用户
	sudo userdel -r lubaobao

	修改用户密码
	sudo passwd lubaobao

	切换用会 
	su 用户
	退出
	exit

创建与删除组，整一个组中的权限一样：

	创建组
	groupadd androidgroup
	分配用户到组
	usermod -G androidgroup jack

####文件权限

r读 w写 x执行，-代表没有对应权限，示例：

	drwxr-xr-x

第一个除外；后面的三个为一组，为一个二进制数：

	d rwx r-x r-x

其中：

0. d代表是目录
1. 文件所属用户（文件创建的用户）具备的权限。
2. 文件所属用户的所属组具备的权限。
3. 系统的其他用户具备的权限。

####权限控制相关权限

ndk目录下面以及子目录的文件能够删除执行写入等等。chmod命令是linux上用于改变权限的命令,-R 是递归遍历子目录，可选：

	chmod 777 -R ndk

	给用户加上执行权限
	chmod u+x file

赋予权限之后可执行的文件变成绿色，可以直接执行。

改变创建目录的默认权限：

	创建出来的权限就是 777 - 022 = 755权限
	umask 022（默认是026）

####改变文件的所属

	更改用户
	chown user file
	更改用户组
	chown user.group file
	chown .group file

####环境变量配置（ndk为例）

每一个用户的Hmoe目录都会有bash、bashrc文件。

bash、bashrc是每一个用户登录的时候会加载的文件，里面配置的是全局的配置，例如环境变量。

	gedit ~/.bashrc(~代表当前用户的Home目录)

	编辑，在最后加上你需要的环境变量：
	export NDKROOT=.....
	export PATH=$NDKROOT:$PATH

	更新（不然的话就需要重启命令行） 
	source ~/.bashrc


####vim的使用

用vim打开文本文件

	vim test.txt

vim有两种模式，命令模式、插入模式。

命令模式不能编辑，但是可以进行一系列命令操作。

#####命令模式中的一些基本操作：

1. 删除一行：dd ， 删除一个字符x
2. 撤销：u，重做：ctrl+r
3. 进入插入模式：i
4. 翻页：ctrl + f（上一页） ctrl + b（下一页）
5. 复制行：yy , 粘贴：p
6. 文本搜索： ?正则enter   搜索n或者N控制方向
6. 保存： :w
7. 保存退出： :wq
8. 不保存强制退出： q!

#####插入模式的一些基本操作：

1. 返回命令模式：按ESC

#####vim的配置

高亮显示、行号等：

	vim etc/vim/vimrc
	
文件末尾追加：

	set nu 
	set tabstop
	set cursorline
	set ruler

如果觉得我的文字对你有所帮助的话，欢迎关注我的公众号：

![公众号：Android开发进阶](http://upload-images.jianshu.io/upload_images/2570030-83ea355270eebc16?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的群欢迎大家进来探讨各种技术与非技术的话题，有兴趣的朋友们加我私人微信**huannan88**，我拉你进群**交（♂）流（♀）**。
