# Linux 设备驱动

# 第六章 高级字符驱动操作

在第 3 章, 我们建立了一个完整的设备驱动, 用户可用来写入和读取。但是一个真正的设备常常提供比同步读和写更多的功能。现在我们已装备有调试工具如果发生错误, 并且一个牢固的并发的理解来帮助避免事情进入错误 -- 我们可安全地前进并且创建一个更高级的驱动。

本章检查几个你需要理解的概念来编写全特性的字符设备驱动。我们从实现 ioctl 系统调用开始, 它是用作设备控制的普通接口。接着我们进入各种和用户空间同步的方法; 在本章结尾, 你有一个充分的认识对于如何使进程睡眠(并且唤醒它们), 实现非阻塞的 I/O,并且通知用户空间当你的设备可用来读或写。我们以查看如何在驱动中实现几个不同的设备存取策略来结束。

这里讨论的概念通过 scull 驱动的几个修改版本来演示。再一次, 所有的都使用内存中的虚拟设备来实现, 因此你可自己试验这些代码而不必使用任何特别的硬件。到此为止,你可能在想亲手使用真正的硬件, 但是那将必须等到第 9 章。

## 6.1 ioctl 接口

大部分驱动需要 -- 除了读写设备的能力 -- 通过设备驱动进行各种硬件控制的能力。大部分设备可进行超出简单的数据传输之外的操作; 用户空间必须常常能够请求, 例如, 设备锁上它的门, 弹出它的介质, 报告错误信息, 改变波特率, 或者自我销毁。这些操作常常通过 ioctl 方法来支持, 它通过相同名子的系统调用来实现。在用户空间, ioctl 系统调用有下面的原型:

~~~c
int ioctl(int fd, unsigned long cmd, ...);
~~~

这个原型由于这些点而凸现于 Unix 系统调用列表, 这些点常常表示函数有数目不定的参数。在实际系统中, 但是, 一个系统调用不能真正有变数目的参数。系统调用必须有一个很好定义的原型, 因为用户程序可存取它们只能通过硬件的"门"。因此, 原型中的点不表示一个变数目的参数, 而是一个单个可选的参数, 传统上标识为 `char *argp`。这些点在那里只是为了阻止在编译时的类型检查。第 3 个参数的实际特点依赖所发出的特定的控制命令( 第 2 个参数 )。一些命令不用参数, 一些用一个整数值, 以及一些使用指向其他数据的指针。使用一个指针是传递任意数据到 `ioctl` 调用的方法; 设备接着可与用户空间交换任何数量的数据。

`ioctl` 调用的非结构化特性使它在内核开发者中失宠。每个 `ioctl` 命令, 基本上, 是一个单独的, 常常无文档的系统调用, 并且没有方法以任何类型的全面的方式核查这些调用。也难于使非结构化的 `ioctl` 参数在所有系统上一致工作; 例如, 考虑运行在 32-位模式的一个用户进程的 64-位 系统。结果, 有很大的压力来实现混杂的控制操作, 只通过任何其他的方法。可能的选择包括嵌入命令到数据流(本章稍后我们将讨论这个方法)或者使用虚拟文件系统, 要么是 `sysfs` 要么是设备特定的文件系统。(我们将在 14 章看看`sysfs`)。但是, 事实是 ``````ioctl`````` 常常是最容易的和最直接的选择,对于真正的设备操作。`ioctl` 驱动方法有和用户空间版本不同的原型:

~~~c
int (*ioctl) (struct inode *inode, struct file *filp, 
              unsigned int cmd, unsigned long arg);
~~~

`inode` 和 `filp` 指针是对应应用程序传递的文件描述符 `fd` 的值, 和传递给 `open` 方法的相同参数。`cmd` 参数从用户那里不改变地传下来, 并且可选的参数 `arg` 参数以一个`unsigned long` 的形式传递, 不管它是否由用户给定为一个整数或一个指针。如果调用程序不传递第 3 个参数, 被驱动操作收到的 `arg` 值是无定义的。因为类型检查在这个额外参数上被关闭, 编译器不能警告你如果一个无效的参数被传递给 `ioctl`, 并且任何关联的错误将难以查找。

如果你可能想到的, 大部分 `ioctl` 实现包括一个大的 `switch` 语句来根据 `cmd` 参数, 选择正确的做法. 不同的命令有不同的数值, 它们常常被给予符号名来简化编码. 符号名通过一个预处理定义来安排. 定制的驱动常常声明这样的符号在它们的头文件中; `scull.h`为 scull 声明它们. 用户程序必须, 当然, 包含那个头文件来存取这些符号.

~~~c
/*
 * The ioctl() implementation
 */

long scull_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{

	int err = 0, tmp;
	int retval = 0;
    
	/*
	 * extract the type and number bitfields, and don't decode
	 * wrong cmds: return ENOTTY (inappropriate ioctl) before access_ok()
	 */
	if (_IOC_TYPE(cmd) != SCULL_IOC_MAGIC) return -ENOTTY;
	if (_IOC_NR(cmd) > SCULL_IOC_MAXNR) return -ENOTTY;

	/*
	 * the direction is a bitmask, and VERIFY_WRITE catches R/W
	 * transfers. `Type' is user-oriented, while
	 * access_ok is kernel-oriented, so the concept of "read" and
	 * "write" is reversed
	 */
	if (_IOC_DIR(cmd) & _IOC_READ)
		err = !access_ok_wrapper(VERIFY_WRITE, (void __user *)arg, _IOC_SIZE(cmd));
	else if (_IOC_DIR(cmd) & _IOC_WRITE)
		err =  !access_ok_wrapper(VERIFY_READ, (void __user *)arg, _IOC_SIZE(cmd));
	if (err) return -EFAULT;

	switch(cmd) {

	  case SCULL_IOCRESET:
		scull_quantum = SCULL_QUANTUM;
		scull_qset = SCULL_QSET;
		break;
        
	  case SCULL_IOCSQUANTUM: /* Set: arg points to the value */
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		retval = __get_user(scull_quantum, (int __user *)arg);
		break;

	  case SCULL_IOCTQUANTUM: /* Tell: arg is the value */
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		scull_quantum = arg;
		break;

	  case SCULL_IOCGQUANTUM: /* Get: arg is pointer to result */
		retval = __put_user(scull_quantum, (int __user *)arg);
		break;

	  case SCULL_IOCQQUANTUM: /* Query: return it (it's positive) */
		return scull_quantum;

	  case SCULL_IOCXQUANTUM: /* eXchange: use arg as pointer */
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		tmp = scull_quantum;
		retval = __get_user(scull_quantum, (int __user *)arg);
		if (retval == 0)
			retval = __put_user(tmp, (int __user *)arg);
		break;

	  case SCULL_IOCHQUANTUM: /* sHift: like Tell + Query */
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		tmp = scull_quantum;
		scull_quantum = arg;
		return tmp;
        
	  case SCULL_IOCSQSET:
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		retval = __get_user(scull_qset, (int __user *)arg);
		break;

	  case SCULL_IOCTQSET:
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		scull_qset = arg;
		break;

	  case SCULL_IOCGQSET:
		retval = __put_user(scull_qset, (int __user *)arg);
		break;

	  case SCULL_IOCQQSET:
		return scull_qset;

	  case SCULL_IOCXQSET:
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		tmp = scull_qset;
		retval = __get_user(scull_qset, (int __user *)arg);
		if (retval == 0)
			retval = put_user(tmp, (int __user *)arg);
		break;

	  case SCULL_IOCHQSET:
		if (! capable (CAP_SYS_ADMIN))
			return -EPERM;
		tmp = scull_qset;
		scull_qset = arg;
		return tmp;

        /*
         * The following two change the buffer size for scullpipe.
         * The scullpipe device uses this same ioctl method, just to
         * write less code. Actually, it's the same driver, isn't it?
         */

	  case SCULL_P_IOCTSIZE:
		scull_p_buffer = arg;
		break;

	  case SCULL_P_IOCQSIZE:
		return scull_p_buffer;


	  default:  /* redundant, as cmd was checked against MAXNR */
		return -ENOTTY;
	}
	return retval;

}
~~~



### 6.1.1 选择 ioctl 命令

在为 ioctl 编写代码之前, 你需要选择对应命令的数字。许多程序员的第一个本能的反应是选择一组小数从0 或1 开始, 并且从此开始向上。但是, 有充分的理由不这样做。ioctl 命令数字应当在这个系统是唯一的, 为了阻止向错误的设备发出正确的命令而引起的错误。这样的不匹配不会不可能发生, 并且一个程序可能发现它自己试图改变一个非串口输入系统的波特率, 例如一个 FIFO 或者一个音频设备。如果这样的 ioctl 号是唯一的, 这个应用程序得到一个 EINVAL 错误而不是继续做不应当做的事情。

为帮助程序员创建唯一的 ioctl 命令代码, 这些编码已被划分为几个位段。Linux 的第一个版本使用 16-位数: 高 8 位是关联这个设备的"魔"数, 低 8 位是一个顺序号, 在设备内唯一。这样做是因为 Linus 是"无能"的(他自己的话); 一个更好的位段划分仅在后来被设想。不幸的是, 许多驱动仍然使用老传统. 它们不得不: 改变命令编码会破坏大量的二进制程序,并且这不是内核开发者愿意见到的。

根据 Linux 内核惯例来为你的驱动选择 ioctl 号, 你应当首先检查`include/asm/ioctl.h` 和 `Documentation/ioctl-number.txt`。这个头文件定义你将使用的位段: `type`(魔数), 序号, 传输方向, 和参数大小。`ioctl-number.txt` 文件列举了在内核中使用的魔数, 因此你将可选择你自己的魔数并且避免交叠。这个文本文件也列举了为什么应当使用惯例的原因。定义 ioctl 命令号的正确方法使用 4 个位段, 它们有下列的含义。这个列表中介绍的新符号定义在 `<linux/ioctl.h>`。

* type

> 魔数. 只是选择一个数(在参考了 ioctl-number.txt 之后)并且使用它在整个驱动中。这个成员是 8 位宽(_IOC_TYPEBITS)。

* number

> 序(顺序)号。它是 8 位(_IOC_NRBITS)宽。

* direction

> 数据传送的方向,如果这个特殊的命令涉及数据传送。可能的值是 `_IOC_NONE`(没有数据传输), `_IOC_READ`, `_IOC_WRITE`, 和 `_IOC_READ|_IOC_WRITE` (数据在2 个方向被传送)。数据传送是从应用程序的观点来看待的; `_IOC_READ` 意思是从设备读,因此设备必须写到用户空间。注意这个成员是一个位掩码, 因此` _IOC_READ` 和`_IOC_WRITE` 可使用一个逻辑 `AND` 操作来抽取。

* size

> 涉及到的用户数据的大小。这个成员的宽度是依赖体系的, 但是常常是 13 或者14 位。你可为你的特定体系在宏 _IOC_SIZEBITS 中找到它的值。你使用这个size 成员不是强制的 - 内核不检查它 -- 但是它是一个好主意。正确使用这个成员可帮助检测用户空间程序的错误并使你实现向后兼容, 如果你曾需要改变相关数据项的大小。如果你需要更大的数据结构, 但是, 你可忽略这个 size 成员。我们很快见到如何使用这个成员。

头文件 `<asm/ioctl.h>`, 它包含在 `<linux/ioctl.h>` 中, 定义宏来帮助建立命令号, 如下: `_IO(type,nr)`(给没有参数的命令), `_IOR(type, nre, datatype)`(给从驱动中读数据的), `_IOW(type,nr,datatype)`(给写数据), 和 `_IOWR(type,nr,datatype)`(给双向传送)。`type` 和 `number` 成员作为参数被传递, 并且 `size` 成员通过应用 `sizeof` 到 `datatype`参数而得到。

这个头文件还定义宏, 可被用在你的驱动中来解码这个号: `_IOC_DIR(nr)`,`_IOC_TYPE(nr)`, `_IOC_NR(nr)`, 和 `_IOC_SIZE(nr)`。我们不进入任何这些宏的细节, 因为头文件是清楚的, 并且在本节稍后有例子代码展示。这里是一些 `ioctl` 命令如何在 scull 被定义的。特别地, 这些命令设置和获得驱动的可配置参数。

~~~c
/*
 * Ioctl definitions
 */

/* Use 'k' as magic number */
#define SCULL_IOC_MAGIC  'k'
/* Please use a different 8-bit number in your code */

#define SCULL_IOCRESET    _IO(SCULL_IOC_MAGIC, 0)

/*
 * S means "Set" through a ptr,
 * T means "Tell" directly with the argument value
 * G means "Get": reply by setting through a pointer
 * Q means "Query": response is on the return value
 * X means "eXchange": switch G and S atomically
 * H means "sHift": switch T and Q atomically
 */
#define SCULL_IOCSQUANTUM _IOW(SCULL_IOC_MAGIC,  1, int)
#define SCULL_IOCSQSET    _IOW(SCULL_IOC_MAGIC,  2, int)
#define SCULL_IOCTQUANTUM _IO(SCULL_IOC_MAGIC,   3)
#define SCULL_IOCTQSET    _IO(SCULL_IOC_MAGIC,   4)
#define SCULL_IOCGQUANTUM _IOR(SCULL_IOC_MAGIC,  5, int)
#define SCULL_IOCGQSET    _IOR(SCULL_IOC_MAGIC,  6, int)
#define SCULL_IOCQQUANTUM _IO(SCULL_IOC_MAGIC,   7)
#define SCULL_IOCQQSET    _IO(SCULL_IOC_MAGIC,   8)
#define SCULL_IOCXQUANTUM _IOWR(SCULL_IOC_MAGIC, 9, int)
#define SCULL_IOCXQSET    _IOWR(SCULL_IOC_MAGIC,10, int)
#define SCULL_IOCHQUANTUM _IO(SCULL_IOC_MAGIC,  11)
#define SCULL_IOCHQSET    _IO(SCULL_IOC_MAGIC,  12)

/*
 * The other entities only have "Tell" and "Query", because they're
 * not printed in the book, and there's no need to have all six.
 * (The previous stuff was only there to show different ways to do it.
 */
#define SCULL_P_IOCTSIZE _IO(SCULL_IOC_MAGIC,   13)
#define SCULL_P_IOCQSIZE _IO(SCULL_IOC_MAGIC,   14)
/* ... more to come */

#define SCULL_IOC_MAXNR 14
~~~









~~~c

~~~

> 1

------------------------

