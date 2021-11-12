> &#160; &#160; &#160; &#160;大一到大二这段时间里学习过单片机的相关知识，对单片机有一定的认识和了解。如果要深究其原理可能还差了一些火候。知道如何编写程序来点量一个LED灯，改一改官方提供的例程来实现一些功能做一些小东西，对IIC、SPI底层的通信协议有一定的了解，但是学着学着逐渐觉得单片机我也就只能改改代码了（当然有的代码也不一定能改出来）。对于我这种以后不想从事单片机开发想搬砖的码农来说已经差不多了（仅仅是个人观点）。
> &#160; &#160; &#160; &#160;在单片机开发中我们常常用到的是裸机，并没有用到操作系统（或者接触过ucos/rtos这种实时操作系统），但是嵌入式Linux开发就必须得在Linux系统中进行操作。我们需要熟悉Linux操作系统，知道<font color=black>**Linux的常用命令、文件系统、Linux网络、多线程/多进程，同时要会用vi编辑器、gcc编译器、shell脚本和一些简单的makefile的编写**</font>，在这些的基础之上进行Linux驱动开发的学习就会如步青云。
>
> <font color=black>**往期推荐：**</font>
> &#160; &#160; &#160; &#160;[史上最全的Linux常用命令汇总（超全面！超详细！）收藏这一篇就够了！](https://wlybsy.blog.csdn.net/article/details/105289038)
> &#160; &#160; &#160; &#160;[STM32通过PWM产生频率为20HZ占空比为50%方波，并通过单片机测量频率并显示](https://wlybsy.blog.csdn.net/article/details/108246269)![在这里插入图片描述](https://img-blog.csdnimg.cn/20201021102616962.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDg5NTY1MQ==,size_16,color_FFFFFF,t_70#pic_center)
>
> &#160; &#160; &#160; &#160;嵌入式Linux操作系统具有：开放源码、所需容量小（最小的安装大约需要2MB）、不需著作权费用、成熟与稳定（经历这些年的发展与使用）、良好的支持等特点。因此被广泛应用于移动电话、个人数码等产品中。嵌入式Linux开发主要包括：底层驱动、操作系统内核、应用开发三大类。需要掌握系统移植（Uboot、Linux Kernel的移植和裁剪、根文件系统的构建）、Linux驱动及内核开发（字符设备驱动、块设备驱动、网络设备驱动）应用开发由于博主能力有限所了解的也不多。

[Toc]

## 字符设备驱动简介
&#160; &#160; &#160; &#160;字符设备是Linux驱动中最基本的一类设备驱动，<font color=red>**字符设备就是一个字节，按照字节进行读写操作设备，读写数据是分先后顺序的**</font>。比如我们常见的点灯、按键、IIC、SPI、LCD等都是字符设备，这些设备的驱动就叫做字符设备驱动。
&#160; &#160; &#160; &#160;在Linux中开发一般只能是用户态，也就是用户只能编写应用程序，但是要作用于内核，那么就需要了解Linux中应用程序是如何调用内核中的驱动程序的，Linux 应用程序对驱动程序的调用如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020102110525049.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDg5NTY1MQ==,size_16,color_FFFFFF,t_70#pic_center)
&#160; &#160; &#160; &#160;<font color=blue>**在Linux 中一切皆为文件**</font>，驱动加载成功以后会在“/dev”目录下生成一个相应的文件，应用程序通过对这个名为“/dev/xxx” (xxx 是具体的驱动文件名字)的文件进行相应的操作即可实现对硬件的操作。比如现在有个叫做`/dev/led` 的驱动文件，此文件是 led 灯的驱动文件。应用程序使用 open 函数来打开文件/dev/led，使用完成以后使用 close 函数关闭/dev/led 这个文件。 open和 close 就是打开和关闭 led 驱动的函数，如果要点亮或关闭 led，那么就使用 write 函数来操作，也就是向此驱动写入数据，这个数据就是要关闭还是要打开 led 的控制参数。如果要获取led 灯的状态，就用 read 函数从驱动中读取相应的状态。
&#160; &#160; &#160; &#160;应用程序运行在用户空间，而 Linux 驱动属于内核的一部分，因此驱动运行于内核空间。当我们在用户空间想要实现对内核的操作，比如使用 open 函数打开/dev/led 这个驱动，因为<font color=blue>**用户空间不能直接对内核进行操作**</font>，因此<font color=blue>**必须使用一个叫做“系统调用”的方法来实现从用户空间陷入到内核空间**</font>，这样才能实现对底层驱动的操作。 open、 close、 write 和 read 等这些函数是有 C 库提供的，**在 Linux 系统中，系统调用作为 C 库的一部分**。当我们调用 open 函数的时候流程如图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201021111820191.png#pic_center)
&#160; &#160; &#160; &#160;应用程序使用到的函数在具体的驱动中都有与之对应的函数，比如应用程序中调用了 open 这个函数，那么在驱动程序中也得有一个名为 open 的函数。每一个系统调用，在驱动中都有与之对应的一个驱动函数，<font color=red>在 Linux 内核文件 `include/linux/fs.h` 中有个叫做 `file_operations` 的结构体，此结构体就是 Linux 内核驱动操作函数集合</font>。

```c
struct file_operations {
	struct module *owner;//owner 拥有该结构体的模块的指针，一般设置为 THIS_MODULE
	loff_t (*llseek) (struct file *, loff_t, int);//llseek 函数用于修改文件当前的读写位置
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t*);//read 函数用于读取设备文件
	ssize_t (*write) (struct file *, const char __user *, size_t,loff_t *);//write 函数用于向设备文件写入(发送)数据
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iterate) (struct file *, struct dir_context *);
	unsigned int (*poll) (struct file *, struct poll_table_struct*);//poll 是个轮询函数，用于查询设备是否可以进行非阻塞的读写
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);//unlocked_ioctl 函数提供对于设备的控制功能，与应用程序中的 ioctl 函数对应。
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);//compat_ioctl 函数与 unlocked_ioctl 函数功能一样，区别在于在 64 位系统上，32 位的应用程序调用将会使用此函数。在 32 位的系统上运行 32 位的应用程序调用的是unlocked_ioctl。
	int (*mmap) (struct file *, struct vm_area_struct *);//mmap 函数用于将将设备的内存映射到进程空间中(也就是用户空间)，一般帧缓冲设备会使用此函数，比如 LCD 驱动的显存，将帧缓冲(LCD 显存)映射到用户空间中以后应用程序就可以直接操作显存了，这样就不用在用户空间和内核空间之间来回复制。
	int (*mremap)(struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);//open 函数用于打开设备文件。
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);//release 函数用于释放(关闭)设备文件，与应用程序中的 close 函数对应。
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);//fasync 函数用于刷新待处理的数据，用于将缓冲区中的数据刷新到磁盘中。
	int (*aio_fsync) (struct kiocb *, int datasync);//aio_fsync 函数与 fasync 函数的功能类似，只是 aio_fsync 是异步刷新待处理的数据
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t,loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long,
	unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *,
	loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct
	pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
	#ifndef CONFIG_MMU
		unsigned (*mmap_capabilities)(struct file *);
	#endif
};
```

## 字符设备驱动开发步骤
&#160; &#160; &#160; &#160;在学习裸机或者 STM32 的时候关于驱动的开发就是初始化相应的外设寄存器，在 Linux 驱动开发中肯定也是要初始化相应的外设寄存器，这是毫无疑问的。只是在 Linux 驱动开发中我们需要按照其规定的框架来编写驱动，所以说<font color=blue>**学 Linux 驱动开发重点是学习其驱动框架**</font>。

### 驱动模块的加载和卸载
&#160; &#160; &#160; &#160;**Linux 驱动有两种运行方式**，**第一种**<font color=blue>就是将驱动编译进 Linux 内核中，这样当 Linux 内核启动的时候就会自动运行驱动程序</font>。**第二种**就是<font color=blue>将驱动编译成模块(Linux 下模块扩展名为.ko)，在Linux 内核启动以后使用“insmod”命令加载驱动模块</font>。在调试驱动的时候一般都选择将其编译为模块，这样我们修改驱动以后只需要编译一下驱动代码即可，不需要编译整个 Linux 代码。而且在调试的时候只需要加载或者卸载驱动模块即可，不需要重启整个系统。

&#160; &#160; &#160; &#160;模块有加载和卸载两种操作，我们在编写驱动的时候需要注册这两种操作函数，模块的加载和卸载注册函数如下:

```c
module_init(xxx_init); //注册模块加载函数
module_exit(xxx_exit); //注册模块卸载函数
```
&#160; &#160; &#160; &#160;module_init 函数用来向 Linux 内核注册一个模块加载函数，参数 xxx_init 就是需要注册的具体函数，当使用“insmod”命令加载驱动的时候， xxx_init 这个函数就会被调用。 module_exit()函数用来向 Linux 内核注册一个模块卸载函数，参数 xxx_exit 就是需要注册的具体函数，当使用“rmmod”命令卸载具体驱动的时候 xxx_exit 函数就会被调用。字符设备驱动模块加载和卸载模板如下所示：
```c
 /* 驱动入口函数 */
static int __init xxx_init(void)
{
	/* 入口函数具体内容 */
	return 0;
}
/* 驱动出口函数 */
static void __exit xxx_exit(void)
{
	 /* 出口函数具体内容 */
 }

	/* 将上面两个函数指定为驱动的入口和出口函数 */
	module_init(xxx_init);
	module_exit(xxx_exit);
```

- 第 2 行，定义了个名为 xxx_init 的驱动入口函数，并且使用了“__init”来修饰。
- 第 9 行，定义了个名为 xxx_exit 的驱动出口函数，并且使用了“__exit”来修饰。
- 第 15 行，调用函数 module_init 来声明 xxx_init 为驱动入口函数，当加载驱动的时候 xxx_init函数就会被调用。
- 第16行，调用函数module_exit来声明xxx_exit为驱动出口函数，当卸载驱动的时候xxx_exit函数就会被调用。

&#160; &#160; &#160; &#160;驱动编译完成以后扩展名为.ko，有两种命令可以加载驱动模块： `insmod`和 `modprobe`，insmod是最简单的模块加载命令，此命令用于加载指定的.ko 模块，比如加载 drv.ko 这个驱动模块，命令如下：

```c
	insmod drv.ko
```

&#160; &#160; &#160; &#160;<font color=blue>**insmod 命令不能解决模块的依赖关系**</font>，比如 drv.ko 依赖 first.ko 这个模块，就必须先使用insmod 命令加载 first.ko 这个模块，然后再加载 drv.ko 这个模块。但是 modprobe 就不会存在这个问题， modprobe 会分析模块的依赖关系，然后会将所有的依赖模块都加载到内核中，因此modprobe 命令相比 insmod 要智能一些。<font color=blue> **modprobe 命令主要智能在提供了模块的依赖性分析、错误检查、错误报告等功能，推荐使用 modprobe 命令来加载驱动**</font>。 modprobe 命令默认会去/lib/modules/<kernel-version>目录中查找模块，比如本书使用的 Linux kernel 的版本号为 4.1.15，因此 modprobe 命令默认到`/lib/modules/4.1.15` 这个目录中查找相应的驱动模块，一般自己制作的根文件系统中是不会有这个目录的，所以需要自己手动创建。驱动模块的卸载使用命令“rmmod”即可，比如要卸载 drv.ko，使用如下命令即可：
```bash
	rmmod drv.ko
```

&#160; &#160; &#160; &#160;也可以使用“modprobe -r”命令卸载驱动，比如要卸载 drv.ko，命令如下：

```bash
	modprobe -r drv.ko
```

&#160; &#160; &#160; &#160;使用 modprobe 命令可以卸载掉驱动模块所依赖的其他模块，前提是这些依赖模块已经没有被其他模块所使用，否则就不能使用 modprobe 来卸载驱动模块。所以对于模块的卸载，还是推荐使用 rmmod 命令。
### 字符设备注册与注销
&#160; &#160; &#160; &#160;对于字符设备驱动而言，当驱动模块加载成功以后需要注册字符设备，同样，卸载驱动模块的时候也需要注销掉字符设备。字符设备的注册和注销函数原型如下所示:

```c
static inline int register_chrdev(unsigned int major, const char *name,const struct file_operations *fops)
static inline void unregister_chrdev(unsigned int major, const char *name)
```
- register_chrdev 函数用于注册字符设备，此函数一共有三个参数，这三个参数的含义如下：
- major： 主设备号， Linux 下每个设备都有一个设备号，设备号分为主设备号和次设备号两部分，关于设备号后面会详细讲解。
- name：设备名字，指向一串字符串。
- fops： 结构体 file_operations 类型指针，指向设备的操作函数集合变量。
- unregister_chrdev 函数用户注销字符设备，此函数有两个参数，这两个参数含义如下：
- major： 要注销的设备对应的主设备号。
- name： 要注销的设备对应的设备名。


一般字符设备的注册在驱动模块的入口函数 xxx_init 中进行，字符设备的注销在驱动模块的出口函数 xxx_exit 中进行。在下面代码中字符设备的注册和注销，内容如下所示：

```c
static struct file_operations test_fops;

/* 驱动入口函数 */
static int __init xxx_init(void)
{
	/* 入口函数具体内容 */
	int retvalue = 0;

	/* 注册字符设备驱动 */
	retvalue = register_chrdev(200, "chrtest", &test_fops);
	if(retvalue < 0){
		/* 字符设备注册失败,自行处理 */
	}
	return 0;
}

/* 驱动出口函数 */
static void __exit xxx_exit(void)
{
	/* 注销字符设备驱动 */
	unregister_chrdev(200, "chrtest");
}

/* 将上面两个函数指定为驱动的入口和出口函数 */
module_init(xxx_init);
module_exit(xxx_exit);
```
- 以上代码中，一开始定义了一个 file_operations 结构体变量 `test_fops`， test_fops 就是设备的操作函数集合，只是此时我们还没有初始化 test_fops 中的 open、 release 等这些成员变量，所以这个操作函数集合还是空的。
- 第十行，调用函数 register_chrdev 注册字符设备，主设备号为 200，设备名字为“chrtest”，设备操作函数集合就是第 1 行定义的 test_fops。要注意的一点就是，选择没有被使用的主设备号，输入命令`cat /proc/devices`可以查看当前已经被使用掉的设备号。
- 第二十一行，调用函数 unregister_chrdev 注销主设备号为 200 的这个设备。

### 实现设备的具体操作函数
&#160; &#160; &#160; &#160;file_operations 结构体就是设备的具体操作函数，在示例代码 40.2.2.1 中我们定义了file_operations结构体类型的变量test_fops，但是还没对其进行初始化，也就是初始化其中的open、release、 read 和 write 等具体的设备操作函数。本节小节我们就完成变量 test_fops 的初始化，设置好针对 chrtest 设备的操作函数。在初始化 test_fops 之前我们要分析一下需求，也就是要对chrtest 这个设备进行哪些操作，只有确定了需求以后才知道我们应该实现哪些操作函数。假设对 chrtest 这个设备有如下两个要求：
**1、能够对 chrtest 进行打开和关闭操作**
&#160; &#160; &#160; &#160;设备打开和关闭是最基本的要求，几乎所有的设备都得提供打开和关闭的功能。因此我们需要实现 file_operations 中的 open 和 release 这两个函数。
**2、对 chrtest 进行读写操作**
&#160; &#160; &#160; &#160;假设 chrtest 这个设备控制着一段缓冲区(内存)，应用程序需要通过 read 和 write 这两个函数对 chrtest 的缓冲区进行读写操作。所以需要实现 file_operations 中的 read 和 write 这两个函数。需求很清晰了，修改驱动示例代码在其中加入 test_fops 这个结构体变量的初始化操作，完成以后的内容如下所示：

```c
/* 打开设备 */
static int chrtest_open(struct inode *inode, struct file *filp)
{
	/* 用户实现具体功能 */
	return 0;
}
/* 从设备读取 */
static ssize_t chrtest_read(struct file *filp, char __user *buf,
size_t cnt, loff_t *offt)
{
	/* 用户实现具体功能 */
	return 0;
}

/* 向设备写数据 */
static ssize_t chrtest_write(struct file *filp,
const char __user *buf,
size_t cnt, loff_t *offt)
{
	/* 用户实现具体功能 */
	return 0;
}
/* 关闭/释放设备 */
static int chrtest_release(struct inode *inode, struct file *filp)
{
	/* 用户实现具体功能 */
	return 0;
}

static struct file_operations test_fops = {
	.owner = THIS_MODULE,
	.open = chrtest_open,
	.read = chrtest_read,
	.write = chrtest_write,
	.release = chrtest_release,
};

/* 驱动入口函数 */
static int __init xxx_init(void)
{
	/* 入口函数具体内容 */
	int retvalue = 0;
	
	/* 注册字符设备驱动 */
	retvalue = register_chrdev(200, "chrtest", &test_fops);
	if(retvalue < 0){
		/* 字符设备注册失败,自行处理 */
	}
	return 0;
}

/* 驱动出口函数 */
static void __exit xxx_exit(void)
{
	/* 注销字符设备驱动 */
	unregister_chrdev(200, "chrtest");
}

/* 将上面两个函数指定为驱动的入口和出口函数 */
module_init(xxx_init);
module_exit(xxx_exit);
```
- 在上面代码中，我们一开始编写了四个函数：`chrtest_open`、 `chrtest_read`、 `chrtest_write`和 `chrtest_release`。这四个函数就是 chrtest 设备的 open、 read、 write 和 release 操作函数。第 29行~35 行初始化 test_fops 的 open、read、 write 和 release 这四个成员变量。

### 添加LICENSE和作者信息
&#160; &#160; &#160; &#160;在驱动编写最后，我们需要在驱动中加入LICENSE信息和作者信息，其中LICENSE是必须添加的，否则的话编译时会报错，作者信息可以添加也可以不添加。 LICENSE 和作者信息的添加使用如下两个函数：

```c
	MODULE_LICENSE() //添加模块 LICENSE 信息
	MODULE_AUTHOR() //添加模块作者信息
```

&#160; &#160; &#160; &#160;给示例代码加入 LICENSE 和作者信息，完成以后的内容如下：

```c
/* 打开设备 */
static int chrtest_open(struct inode *inode, struct file *filp)
{
	/* 用户实现具体功能 */
	return 0;
}
......

/* 将上面两个函数指定为驱动的入口和出口函数 */
module_init(xxx_init);
module_exit(xxx_exit);

MODULE_LICENSE("GPL");//LICENSE 采用 GPL 协议。
MODULE_AUTHOR("wly");//添加作者名字
```


&#160; &#160; &#160; &#160;当添加完作者和LICENSE和作者信息后，字符设备驱动的完整流程就基本上结束了，并且也提供了一个完整的Linux驱动的模板，以后字符设备驱动开发就可以修改这个模板。

## Linux设备号
&#160; &#160; &#160; &#160;Linux的设备管理是和文件系统紧密结合的，各种设备都以文件的形式存放在/dev目录下，称为设备文件。应用程序可以打开、关闭和读写这些设备文件，完成对设备的操作，就像操作普通的数据文件一样。为了管理这些设备，系统为设备编了号，这个号就被称为Linux设备号！
### 设备号的组成
&#160; &#160; &#160; &#160;设备号由主设备号和次设备号两部分组成，主设备号表示某一个具体的驱动，次设备号表示使用这个驱动的各个设备。 Linux 提供了一个名为 dev_t 的数据类型表示设备号， dev_t 定义在文件`include/linux/types.h` 里面，定义如下：

```bash
typedef __u32 __kernel_dev_t;
......
typedef __kernel_dev_t dev_t;
```

&#160; &#160; &#160; &#160;可以看出 dev_t 是__u32 类型的，而__u32 定义在文件 `include/uapi/asm-generic/int-ll64.h` 里面，定义如下：

```bash
typedef unsigned int __u32;
```
&#160; &#160; &#160; &#160;dev_t 其实就是 unsigned int 类型，是一个 32 位的数据类型。<font color=blue>这 32 位的数据构成了主设备号和次设备号两部分，其中高 12 位为主设备号，第 20 位为次设备号。因此 **Linux系统中主设备号范围为0~4095**</font>，所以大家在选择主设备号的时候一定不要超过这个范围。在文件 `include/linux/kdev_t.h` 中提供了几个关于设备号的操作函数(本质是宏)，如下所示：

```c
#define MINORBITS 20
#define MINORMASK ((1U << MINORBITS) - 1)
#define MAJOR(dev) ((unsigned int) ((dev) >> MINORBITS))
#define MINOR(dev) ((unsigned int) ((dev) & MINORMASK))
#define MKDEV(ma,mi) (((ma) << MINORBITS) | (mi))
```

- 第 1 行，宏 MINORBITS 表示次设备号位数，一共是 20 位。
- 第 2 行，宏 MINORMASK 表示次设备号掩码。
- 第 3 行，宏 MAJOR 用于从 dev_t 中获取主设备号，将 dev_t 右移 20 位即可。
- 第 4 行，宏 MINOR 用于从 dev_t 中获取此设备号，取 dev_t 的低 20 位的值即可。
- 第 5 行，宏 MKDEV 用于将给定的主设备号和次设备号的值组合成 dev_t 类型的设备号。
### 设备号的分配
<h5>1、静态分配设备号</h5>

注册字符设备的时候需要给设备指定一个设备号，这个设备号可以是驱动开发者静态的指定一个设备号，比如选择 200 这个主设备号。有一些常用的设备号已经被 Linux 内核开发者给分配掉了，具体分配的内容可以查看文档 `Documentation/devices.txt`。并不是说内核开发者已经分配掉的主设备号我们就不能用了，具体能不能用还得看我们的硬件平台运行过程中有没有使用这个主设备号，使用`cat /proc/devices`命令即可查看当前系统中所有已经使用了的设备号。
<h5>2、动态分配设备号</h5>

静态分配设备号需要我们检查当前系统中所有被使用了的设备号，然后挑选一个没有使用的。而且<font color=red>**静态分配设备号很容易带来冲突问题**</font>， Linux 社区推荐使用动态分配设备号，在注册字符设备之前先申请一个设备号，系统会自动给你一个没有被使用的设备号，这样就避免了冲突。卸载驱动的时候释放掉这个设备号即可，设备号的申请函数如下：
```c
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)
```
- dev：保存申请到的设备号。
- baseminor： 次设备号起始地址， alloc_chrdev_region 可以申请一段连续的多个设备号，这些设备号的主设备号一样，但是次设备号不同，次设备号以 baseminor 为起始地址地址开始递增。一般 baseminor 为 0，也就是说次设备号从 0 开始。
- count： 要申请的设备号数量。
- name：设备名字。


注销字符设备之后要释放掉设备号，设备号释放函数如下：
```c
void unregister_chrdev_region(dev_t from, unsigned count)
```
- from：要释放的设备号。
- count： 表示从 from 开始，要释放的设备号数量。

 >&#160; &#160; &#160; &#160;**不积小流无以成江河，不积跬步无以至千里**。而我想要成为万里羊，就必须坚持学习来获取更多知识，<font color=blue>用知识来改变命运，用博客见证成长，用行动证明我在努力。</font>
 > &#160; &#160; &#160; &#160;如果我的博客对你有帮助、如果你喜欢我的博客内容，记得<font color=red>“点赞” “评论” “收藏”一键三连</font>哦！听说点赞的人运气不会太差，每一天都会元气满满呦！如果实在要白嫖的话，那祝你开心每一天，欢迎常来我博客看看。