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

真正的源文件定义几个额外的这里没有出现的命令。

**`ioctl` 命令号到底是什么？**`ioctl(cmd, arg)` 里的 **`cmd` 不是随便的整数**，而是一个 **32bit 编码值**，里面打包了 4 类信息：

```
31            30 29        16 15        8 7        0
+---------------+------------+------------+----------+
| 方向(direction)| 数据大小    | magic号     | 序号(nr) |
+---------------+------------+------------+----------+
```

Linux 用宏把这 4 个字段拼出来。

**`_IO / _IOR / _IOW / _IOWR` 的本质**

这些宏在 `<linux/ioctl.h>` 里定义：

| 宏      | 含义         | 用户态 ↔ 内核态 |
| ------- | ------------ | --------------- |
| `_IO`   | 无数据       | 仅命令          |
| `_IOR`  | Read         | 内核 → 用户     |
| `_IOW`  | Write        | 用户 → 内核     |
| `_IOWR` | Read + Write | 双向            |

⚠️ **Read / Write 是站在“用户态”的角度**

------

**`SCULL_IOC_MAGIC`**

```
#define SCULL_IOC_MAGIC  'k'
```

作用

- **区分不同驱动的 ioctl 命令**、防止命令号冲突

内核要求：

- magic 是 **8bit**、通常用 **字符**

例如：

- `'f'`：framebuffer
- `'T'`：tty
- `'k'`：scull（随便选的）

--------------------

**S / T / G / Q / X / H 这组注释是什么意思？**

这是 **scull 作者为了教学而人为定义的命名规范**：

| 字母 | 含义                        |
| ---- | --------------------------- |
| S    | Set（通过指针写入）         |
| T    | Tell（直接用 arg 传值）     |
| G    | Get（通过指针返回）         |
| Q    | Query（通过返回值返回）     |
| X    | eXchange（交换）            |
| H    | sHift（返回旧值并设置新值） |

⚠️ **不是内核规定，是示例约定**

Set：用户 → 内核（带指针）

```
#define SCULL_IOCSQUANTUM _IOW(SCULL_IOC_MAGIC, 1, int)
```

Tell：直接传值（不用指针）

```
#define SCULL_IOCTQUANTUM _IO(SCULL_IOC_MAGIC, 3)
```

> 教学示例而已，**实际上没有用 arg 的值**

Get：内核 → 用户（指针）

```
#define SCULL_IOCGQUANTUM _IOR(SCULL_IOC_MAGIC, 5, int)
```

 Query：直接返回值

```
#define SCULL_IOCQQUANTUM _IO(SCULL_IOC_MAGIC, 7)
```

Exchange：交换值（双向）

```
#define SCULL_IOCXQUANTUM _IOWR(SCULL_IOC_MAGIC, 9, int)
```

Shift：返回旧值，设置新值（不用指针）

```
#define SCULL_IOCHQUANTUM _IO(SCULL_IOC_MAGIC, 11)
```

| 宏      | 数据方向    | 是否用指针 |
| ------- | ----------- | ---------- |
| `_IO`   | 无          | 否         |
| `_IOR`  | 内核 → 用户 | 是         |
| `_IOW`  | 用户 → 内核 | 是         |
| `_IOWR` | 双向        | 是         |

> **现实驱动中：**

- 90% 情况：只用 `_IOW / _IOR / _IOWR`
- 很少用 `Query / Tell`
- `magic + nr` 一定要唯一
- 永远校验 `_IOC_TYPE(cmd)` 和 `_IOC_NR(cmd)`



### 6.1.2 返回值

`ioctl` 的实现常常是一个 `switch` 语句, 基于命令号。但是当命令号没有匹配一个有效的操作时缺省的选择应当是什么? 这个问题是有争议的。几个内核函数返回 `-ENIVAL`("Invalid argument"), 它有意义是因为命令参数确实不是一个有效的。POSIX 标准, 但是, 说如果一个不合适的 ioctl 命令被发出, 那么 `-ENOTTY` 应当被返回。这个错误码被 C 库解释为"设备的不适当的 `ioctl`", 这常常正是程序员需要听到的。然而, 它仍然是相当普遍的来返回 `-EINVAL`, 对于响应一个无效的 `ioctl` 命令。



### 6.1.3 预定义的命令

尽管 ioctl 系统调用最常用来作用于设备, 内核能识别几个命令。注意这些命令, 当用到你的设备时, 在你自己的文件操作被调用之前被解码。因此, 如果你选择相同的号给一个你的 ioctl 命令, 你不会看到任何的给那个命令的请求, 并且应用程序获得某些不期望的东西, 因为在 ioctl 号之间的冲突。

预定义命令分为 3 类:

* 可对任何文件发出的(常规, 设备, FIFO, 或者 socket) 的那些.
* 只对常规文件发出的那些.
* 对文件系统类型特殊的那些.

最后一类的命令由宿主文件系统的实现来执行(这是 `chattr` 命令如何工作的)。设备驱动编写者只对第一类命令感兴趣, 它们的魔数是 "`T`"。 查看其他类的工作留给读者作为练习;`ext2_ioctl` 是最有趣的函数(并且比预期的要容易理解), 因为它实现 `append-only` 标志和 `immutable` 标志。

~~~c
FIOCLEX
~~~

> 设置 `close-on-exec` 标志(File IOctl Close on EXec)。设置这个标志使文件描述符被关闭, 当调用进程执行一个新程序时.

------------------------

~~~c
FIONCLEX
~~~

> 清除 `close-no-exec` 标志(File IOctl Not CLose on EXec)。这个命令恢复普通文件行为, 复原上面 `FIOCLEX` 所做的。`FIOASYNC` 为这个文件设置或者复位异步通知(如同在本章中"异步通知"一节中讨论的)。注意直到 Linux 2.2.4 版本的内核不正确地使用这个命令来修改 `O_SYNC` 标志。因为两个动作都可通过 `fcntl` 来完成, 没有人真正使用 `FIOASYNC` 命令, 它在这里出现只是为了完整性。

------------------------

~~~c
FIOQSIZE
~~~

> 这个命令返回一个文件或者目录的大小; 当用作一个设备文件, 但是, 它返回一个`ENOTTY` 错误。

------------------------

~~~c
FIONBIO
~~~

> "File IOctl Non-Blocking `I/O`"(在"阻塞和非阻塞操作"一节中描述)。这个调用修改在 `filp->f_flags` 中的 `O_NONBLOCK` 标志。给这个系统调用的第 3 个参数用作指示是否这个标志被置位或者清除。(我们将在本章看到这个标志的角色)。注意常用的改变这个标志的方法是使用 `fcntl` 系统调用, 使用 `F_SETFL` 命令。

------------------------

列表中的最后一项介绍了一个新的系统调用, `fcntl`, 它看来象 `ioctl`。事实上, ``fcntl``调用非常类似 `ioctl`, 它也是获得一个命令参数和一个额外的(可选地)参数。它保持和`ioctl` 独立主要是因为历史原因: 当 Unix 开发者面对控制 `I/O` 操作的问题时, 他们决定文件和设备是不同的。那时, 有 `ioctl` 实现的唯一设备是 `ttys`, 它解释了为什么` -ENOTTY` 是标准的对不正确 `ioctl` 命令的回答。事情已经改变, 但是 `fcntl` 保留为一个独立的系统调用。



### 6.1.4 使用 ioctl 参数

在看 scull 驱动的 ioctl 代码之前, 我们需要涉及的另一点是如何使用这个额外的参数。如果它是一个整数, 就容易: 它可以直接使用。如果它是一个指针, 但是, 必须小心些。

当用一个指针引用用户空间, 我们必须确保用户地址是有效的。试图存取一个没验证过的用户提供的指针可能导致不正确的行为, 一个内核 oops, 系统崩溃, 或者安全问题。它是驱动的责任来对每个它使用的用户空间地址进行正确的检查, 并且返回一个错误如果它是无效的。

在第 3 章, 我们看了 `copy_from_user` 和 `copy_to_user` 函数, 它们可用来安全地移动数据到和从用户空间。这些函数也可用在 `ioctl` 方法中, 但是 `ioctl` 调用常常包含小数据项, 可通过其他方法更有效地操作。开始, 地址校验(不传送数据)由函数 `access_ok`实现, 它定义在 `<asm/uaccess.h>`:

~~~c
int access_ok(int type, const void *addr, unsigned long size);
~~~

第一个参数应当是 `VERIFY_READ` 或者 `VERIFY_WRITE`, 依据这个要进行的动作是否是读用户空间内存区或者写它. `addr` 参数持有一个用户空间地址, `size` 是一个字节量。例如,如果 `ioctl` 需要从用户空间读一个整数, `size` 是`sizeof(int)`。如果你需要读和写给定地址, 使用 `VERIFY_WRITE`, 因为它是 `VERIRY_READ` 的超集。

不象大部分的内核函数, `access_ok` 返回一个布尔值: 1 是成功(存取没问题)和 0 是失败(存取有问题)。如果它返回假, 驱动应当返回 `-EFAULT` 给调用者。

关于 `access_ok` 有多个有趣的东西要注意。首先, 它不做校验内存存取的完整工作; 它只检查看这个内存引用是在这个进程有合理权限的内存范围中。特别地, `access_ok` 确保这个地址不指向内核空间内存。第2, 大部分驱动代码不需要真正调用 `access_ok`。后面描述的内存存取函数为你负责这个。但是, 我们来演示它的使用, 以便你可见到它如何完成.scull 源码利用了 `ioclt` 号中的位段来检查参数, 在 `switch` 之前:

~~~c
/*
 * @file access_ok_version.h
 * @date 10/13/2019
 *
 */

#include <linux/version.h>

#if LINUX_VERSION_CODE < KERNEL_VERSION(5,0,0)
#define access_ok_wrapper(type,arg,cmd) \
	access_ok(type, arg, cmd)
#else
#define access_ok_wrapper(type,arg,cmd) \
	access_ok(arg, cmd)
#endif
/*------------------------------------------------*/
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
switch(cmd) {...}
~~~

在调用 `access_ok` 之后, 驱动可安全地进行真正的传输。加上 `copy_from_user` 和 `copy_to_user_` 函数, 程序员可利用一组为被最多使用的数据大小(1, 2, 4, 和 8 字节) 而优化过的函数。这些函数在下面列表中描述, 它们定义在` <asm/uaccess.h>`:



~~~c
put_user(datum, ptr)
__put_user(datum, ptr)
~~~

> 这些宏定义写 `datum` 到用户空间; 它们相对快, 并且应当被调用来代替`copy_to_user` 无论何时要传送单个值时。这些宏已被编写来允许传递任何类型的指针到 `put_user`, 只要它是一个用户空间地址。传送的数据大小依赖 `ptr` 参数的类型, 并且在编译时使用 `sizeof` 和 `typeof` 等编译器内建宏确定。结果是, 如果`ptr` 是一个 `char` 指针, 传送一个字节, 以及对于 2, 4, 和 可能的 8 字节。
>
> `put_user` 检查来确保这个进程能够写入给定的内存地址。它在成功时返回 0, 并且在错误时返回 `-EFAULT`。`__put_user` 进行更少的检查(它不调用 `access_ok`),但是仍然能够失败如果被指向的内存对用户是不可写的。因此, `__put_user` 应当只用在内存区已经用 `access_ok` 检查过的时候。
>
> 作为一个通用的规则, 当你实现一个 `read` 方法时, 调用 `__put_user` 来节省几个周期, 或者当你拷贝几个项时, 因此, 在第一次数据传送之前调用 `access_ok` 一次, 如同上面 `ioctl` 所示。

------------------------

~~~c
get_user(local, ptr)
__get_user(local, ptr)
~~~

> 这些宏定义用来从用户空间接收单个数据。它们象 `put_user` 和 `__put_user`, 但是在相反方向传递数据。获取的值存储于本地变量 `local`; 返回值指出这个操作是否成功。再次, `__get_user` 应当只用在已经使用 `access_ok` 校验过的地址。

------------------------

如果做一个尝试来使用一个列出的函数来传送一个不适合特定大小的值, 结果常常是一个来自编译器的奇怪消息, 例如"`coversion to non-scalar type requested`"。在这些情况中, 必须使用 `copy_to_user` 或者 `copy_from_user`。



### 6.1.5 兼容性和受限操作

存取一个设备由设备文件上的许可权控制, 并且驱动正常地不涉及到许可权的检查。但是,有些情形, 在保证给任何用户对设备的读写许可的地方, 一些控制操作仍然应当被拒绝。例如, 不是所有的磁带驱动器的用户都应当能够设置它的缺省块大小, 并且一个已经被给予对一个磁盘设备读写权限的用户应当仍然可能被拒绝来格式化它。在这样的情况下, 驱动必须进行额外的检查来确保用户能够进行被请求的操作。

传统上 unix 系统对超级用户帐户限制了特权操作。这意味着特权是一个全有-或-全无的东西 -- 超级用户可能任意做任何事情, 但是所有其他的用户被高度限制了。Linux 内核提供了一个更加灵活的系统, 称为能力。一个基于能力的系统丢弃了全有-或全无模式,并且打破特权操作为独立的子类。这种方式, 一个特殊的用户(或者是程序)可被授权来进行一个特定的特权操作而不必泄漏进行其他的, 无关的操作的能力。内核在许可权管理上排他地使用能力, 并且输出 2 个系统调用 `capget` 和 `capset`, 来允许它们被从用户空间管理。

全部能力可在 `<linux/capability.h>` 中找到。这些是对系统唯一可用的能力; 对于驱动作者或者系统管理员, 不可能不修改内核源码而来定义新的。设备驱动编写者可能感兴趣的这些能力的一个子集, 包括下面:

~~~c
CAP_DAC_OVERRIDE
~~~

> 这个能力来推翻在文件和目录上的存取的限制(数据存取控制, 或者 DAC)。

------------------------

~~~c
CAP_NET_ADMIN
~~~

> 进行网络管理任务的能力, 包括那些能够影响网络接口的。

------------------------

~~~c
CAP_SYS_MODULE
~~~

> 加载或去除内核模块的能力。

------------------------

~~~c
CAP_SYS_RAWIO
~~~

> 进行 "raw" I/O 操作的能力。例子包括存取设备端口或者直接和 USB 设备通讯。

------------------------

~~~c
CAP_SYS_ADMIN
~~~

> 一个捕获-全部的能力, 提供对许多系统管理操作的存取。

------------------------

~~~c
CAP_SYS_TTY_CONFIG
~~~

> 进行 tty 配置任务的能力。

------------------------

在进行一个特权操作之前, 一个设备驱动应当检查调用进程有合适的能力; 不这样做可能导致用户进程进行非法的操作, 对系统的稳定和安全有坏的后果. 能力检查是通过capable 函数来进行的(定义在 <linux/sched.h>):

~~~c
int capable(int capability);
~~~

在 scull 例子驱动中, 任何用户被许可来查询 `quantum` 和 `quantum` 集的大小。只有特权用户, 但是, 可改变这些值, 因为不适当的值可能很坏地影响系统性能。当需要时,`ioctl` 的 scull 实现检查用户的特权级别, 如下:

~~~c
if (! capable (CAP_SYS_ADMIN))
	return -EPERM;
~~~

在这个任务缺乏一个更加特定的能力时, `CAP_SYS_ADMIN` 被选择来做这个测试。

### 6.1.6 ioctl 命令的实现

`ioctl` 的 scull 实现只传递设备的配置参数, 并且象下面这样容易:

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

scull 还包含 6 个入口项作用于 `scull_qset`。这些入口项和给 `scull_quantum` 的是一致的, 并且不值得展示出来。从调用者的观点看(即从用户空间), 这 6 种传递和接收参数的方法看来如下:

~~~c
int quantum;
ioctl(fd,SCULL_IOCSQUANTUM, &quantum); /* Set by pointer */
ioctl(fd,SCULL_IOCTQUANTUM, quantum); /* Set by value */
ioctl(fd,SCULL_IOCGQUANTUM, &quantum); /* Get by pointer */
quantum = ioctl(fd,SCULL_IOCQQUANTUM); /* Get by return value */
ioctl(fd,SCULL_IOCXQUANTUM, &quantum); /* Exchange by pointer */

quantum = ioctl(fd,SCULL_IOCHQUANTUM, quantum); /* Exchange by value */
~~~

当然, 一个正常的驱动不可能实现这样一个调用模式的混合体。我们这里这样做只是为了演示做事情的不同方式。但是, 正常地, 数据交换将一致地进行, 通过指针或者通过值,并且要避免混合这 2 种技术。

#### ioctl 的时序

**一、总体时序（从用户态到驱动）**

```
用户态 (User Space)                     内核态 (Kernel Space)
───────────────────                    ─────────────────────
应用程序
   |
   | ioctl(fd, cmd, arg)
   |---------------------------------->  sys_ioctl()
                                       |
                                       |  fd → struct file *
                                       |  file->f_op->unlocked_ioctl ?
                                       |
                                       |--------------------------------
                                       |   scull_ioctl(file, cmd, arg)
                                       |--------------------------------
                                       |
                                       |  switch(cmd)
                                       |     |
                                       |     +--> copy_from_user()
                                       |     |
                                       |     +--> copy_to_user()
                                       |     |
                                       |     +--> return value
                                       |
   |<----------------------------------  return to user
   |
应用程序继续执行
```

------

**二、带 `_IOW / _IOR / _IOWR` 的详细时序**

以：

```
#define SCULL_IOCSQUANTUM _IOW('k', 1, int)
```

为例（**用户写内核**）

```
User Space                                Kernel Space
──────────                                ─────────────
int q = 4096;
ioctl(fd, SCULL_IOCSQUANTUM, &q)
 |
 | trap / syscall
 |-------------------------------------> sys_ioctl()
                                        |
                                        | cmd = SCULL_IOCSQUANTUM
                                        | arg = 用户虚拟地址(&q)
                                        |
                                        | file->f_op->unlocked_ioctl
                                        |--------------------------------
                                        | scull_ioctl()
                                        |
                                        | _IOC_TYPE(cmd) == 'k' ?
                                        | _IOC_NR(cmd)   <= MAXNR ?
                                        |
                                        | copy_from_user()
                                        |   ┌─────────────────────────┐
                                        |   │ 用户页 → 内核缓冲区        │
                                        |   └─────────────────────────┘
                                        |
                                        | scull_quantum = q
                                        |
                                        | return 0
 |<-------------------------------------
 |
继续执行
```

------

**三、Get（`_IOR`）：内核 → 用户**

```
#define SCULL_IOCGQUANTUM _IOR('k', 5, int)
User Space                                Kernel Space
──────────                                ─────────────
int q;
ioctl(fd, SCULL_IOCGQUANTUM, &q)
 |
 |-------------------------------------> sys_ioctl()
                                        |
                                        | scull_ioctl()
                                        |
                                        | tmp = scull_quantum
                                        |
                                        | copy_to_user()
                                        |   ┌─────────────────────────┐
                                        |   │ 内核缓冲区 → 用户页        │
                                        |   └─────────────────────────┘
                                        |
                                        | return 0
 |<-------------------------------------
 |
q = scull_quantum
```

------

**四、Query（返回值）**

```
#define SCULL_IOCQQUANTUM _IO('k', 7)
User Space                                Kernel Space
──────────                                ─────────────
q = ioctl(fd, SCULL_IOCQQUANTUM)
 |
 |-------------------------------------> sys_ioctl()
                                        |
                                        | scull_ioctl()
                                        |
                                        | return scull_quantum
 |<-------------------------------------
 |
q = 返回值
```

------

**五、Exchange（`_IOWR`）：双向**

```
#define SCULL_IOCXQUANTUM _IOWR('k', 9, int)
User Space                                Kernel Space
──────────                                ─────────────
int q = 1024;
ioctl(fd, CMD, &q)
 |
 |-------------------------------------> scull_ioctl()
                                        |
                                        | copy_from_user(new)
                                        |
                                        | old = scull_quantum
                                        | scull_quantum = new
                                        |
                                        | copy_to_user(old)
                                        |
                                        | return 0
 |<-------------------------------------
 |
q == old_value
```

------

**六、内核真实调用栈（帮助你定位代码）**

```
ioctl()
 └─ libc ioctl()
     └─ syscall(SYS_ioctl)
         └─ __x64_sys_ioctl   / __arm64_sys_ioctl
             └─ do_vfs_ioctl
                 └─ file->f_op->unlocked_ioctl
                     └─ scull_ioctl
```

👉 在 **ARM / MIPS / x86** 上路径一致，名字略有差异。

-----------------------

### 6.1.7 不用 ioctl 的设备控制

有时控制设备最好是通过写控制序列到设备自身来实现。例如, 这个技术用在控制台驱动中, 这里所谓的 `escape` 序列被用来移动光标, 改变缺省的颜色, 或者进行其他的配置任务。这样实现设备控制的好处是用户可仅仅通过写数据控制设备, 不必使用(或者有时候写)只为配置设备而建立的程序。当设备可这样来控制, 发出命令的程序甚至常常不需要运行在和它要控制的设备所在的同一个系统上。

例如, `setterm` 程序作用于控制台(或者其他终端)配置, 通过打印 `escape` 序列。控制程序可位于和被控制的设备不同的一台计算机上, 因为一个简单的数据流重定向可完成这个配置工作。这是每次你运行一个远程 `tty` 会话时所发生的事情: `escape` 序列在远端被打印但是影响到本地的 `tty`; 然而, 这个技术不局限于 `ttys`。

通过打印来控制的缺点是它给设备增加了策略限制; 例如, 它仅仅当你确信在正常操作时控制序列不会出现在正被写入设备的数据中。这对于 ttys 只是部分正确的。尽管一个文本显示意味着只显示 ASCII 字符, 有时控制字符可潜入正被写入的数据中, 并且可能,因此, 影响控制台的配置。例如, 这可能发生在你显示一个二进制文件到屏幕时; 产生的乱码可能包含任何东西, 并且最后你常常在你的控制台上出现错误的字体。

通过写来控制是当然的使用方法了, 对于不用传送数据而只是响应命令的设备, 例如遥控设备。

例如, 被你们作者当中的一个编写来好玩的驱动, 移动一个 2 轴上的摄像机。在这个驱动里, 这个"设备"是一对老式步进电机, 它们不能真正读或写。给一个步进电机"发送数据流"的概念没有任何意义。在这个情况下, 驱动解释正被写入的数据作为 ASCII 命令并且转换这个请求为脉冲序列, 来操纵步进电机。这个概念类似于, 有些, 你发给猫的 AT命令来建立通讯, 主要的不同是和猫通讯的串口必须也传送真正的数据。直接设备控制的好处是你可以使用 `cat` 来移动摄像机, 而不必写和编译特殊的代码来发出 `ioctl` 调用。

当编写面向命令的驱动, 没有理由实现 ioctl 命令。一个解释器中的额外命令更容易实现并使用。

有时, 你可能选择使用其他的方法:不必转变 `write` 方法为一个解释器和避免`ioctl`, 你可能选择完全避免写并且专门使用 `ioctl` 命令, 而实现驱动为使用一个特殊的命令行工具来发送这些命令到驱动。这个方法转移复杂性从内核空间到用户空间, 这里可能更易处理, 并且帮助保持驱动小, 而拒绝使用简单的 `cat` 或者 `echo` 命令。



## 6.2 阻塞 I/O

回顾第 3 章, 我们看到如何实现 `read` 和 `write` 方法。在此, 但是, 我们跳过了一个重要的问题:一个驱动当它无法立刻满足请求应当如何响应? 一个对 `read` 的调用可能当没有数据时到来, 而以后会期待更多的数据。或者一个进程可能试图写, 但是你的设备没有准备好接受数据, 因为你的输出缓冲满了。调用进程往往不关心这种问题; 程序员只希望调用 `read` 或 `write` 并且使调用返回, 在必要的工作已完成后。这样, 在这样的情形中,你的驱动应当(缺省地)阻塞进程, 使它进入睡眠直到请求可继续。

本节展示如何使一个进程睡眠并且之后再次唤醒它。但是, 我们必须首先解释几个概念。

### 6.2.1 睡眠的介绍

对于一个进程"睡眠"意味着什么? 当一个进程被置为睡眠, 它被标识为处于一个特殊的状态并且从调度器的运行队列中去除。直到发生某些事情改变了那个状态, 这个进程将不被在任何 CPU 上调度, 并且, 因此, 将不会运行。一个睡着的进程已被搁置到系统的一边, 等待以后发生事件。

对于一个 Linux 驱动使一个进程睡眠是一个容易做的事情。但是, 有几个规则必须记住以安全的方式编码睡眠。

这些规则的第一个是: 当你运行在原子上下文时不能睡眠。我们在第 5 章介绍过原子操作; 一个原子上下文只是一个状态, 这里多个步骤必须在没有任何类型的并发存取的情况下进行。这意味着, 对于睡眠, 是你的驱动在持有一个自旋锁, `seqlock`, 或者 RCU 锁时不能睡眠。如果你已关闭中断你也不能睡眠。在持有一个旗标时睡眠是合法的, 但是你应当仔细查看这样做的任何代码。如果代码在持有一个旗标时睡眠, 任何其他的等待这个旗标的线程也睡眠。因此发生在持有旗标时的任何睡眠应当短暂, 并且你应当说服自己, 由于持有这个旗标, 你不能阻塞这个将最终唤醒你的进程。

另一件要记住的事情是, 当你醒来, 你从不知道你的进程离开 CPU 多长时间或者同时已经发生了什么改变。你也常常不知道是否另一个进程已经睡眠等待同一个事件; 那个进程可能在你之前醒来并且获取了你在等待的资源。结果是你不能关于你醒后的系统状态做任何的假设, 并且你必须检查来确保你在等待的条件是真的。

一个另外的相关的点, 当然, 是你的进程不能睡眠除非确信其他人, 在某处的, 将唤醒它。做唤醒工作的代码必须也能够找到你的进程来做它的工作。确保一个唤醒发生, 是深入考虑你的代码和对于每次睡眠, 确切知道什么系列的事件将结束那次睡眠。使你的进程可能被找到, 真正地, 通过一个称为等待队列的数据结构实现的。一个等待队列就是它听起来的样子:一个进程列表, 都等待一个特定的事件。

在 Linux 中, 一个等待队列由一个"等待队列头"来管理, 一个 `wait_queue_head_t` 类型的结构, 定义在`<linux/wait.h>`中。一个等待队列头可被定义和初始化, 使用:

~~~c
DECLARE_WAIT_QUEUE_HEAD(name);
~~~

或者动态地, 如下:

~~~c
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
~~~

我们将很快返回到等待队列结构, 但是我们知道了足够多的来首先看看睡眠和唤醒。

### 6.2.2 简单睡眠

当一个进程睡眠, 它这样做以期望某些条件在以后会成真。如我们之前注意到的, 任何睡眠的进程必须在它再次醒来时检查来确保它在等待的条件真正为真。Linux 内核中睡眠的最简单方式是一个宏定义, 称为 wait_event(有几个变体); 它结合了处理睡眠的细节和进程在等待的条件的检查。wait_event 的形式是:

~~~c
wait_event(queue, condition)
wait_event_interruptible(queue, condition)
wait_event_timeout(queue, condition, timeout)
wait_event_interruptible_timeout(queue, condition, timeout)
~~~

在所有上面的形式中, `queue` 是要用的等待队列头。注意它是"通过值"传递的。条件是一个被这个宏在睡眠前后所求值的任意的布尔表达式; 直到条件求值为真值, 进程继续睡眠.注意条件可能被任意次地求值, 因此它不应当有任何边界效应。

如果你使用 `wait_event`, 你的进程被置为不可中断地睡眠, 如同我们之前已经提到的,它常常不是你所要的。首选的选择是 `wait_event_interruptible`, 它可能被信号中断.这个版本返回一个你应当检查的整数值; 一个非零值意味着你的睡眠被某些信号打断, 并且你的驱动可能应当返回 `-ERESTARTSYS`。最后的版本(`wait_event_timeout` 和`wait_event_interruptible_timeout`)等待一段有限的时间; 在这个时间期间(以嘀哒数表达的, 我们将在第 7 章讨论)超时后, 这个宏返回一个 0 值而不管条件是如何求值的。

图片的另一半, 当然, 是唤醒。一些其他的执行线程(一个不同的进程, 或者一个中断处理, 也许)必须为你进行唤醒, 因为你的进程, 当然, 是在睡眠. 基本的唤醒睡眠进程的函数称为 wake_up。它有几个形式(但是我们现在只看其中 2 个):

~~~c
void wake_up(wait_queue_head_t *queue);
void wake_up_interruptible(wait_queue_head_t *queue);
~~~

`wake_up` 唤醒所有的在给定队列上等待的进程(尽管这个情形比那个要复杂一些, 如同我们之后将见到的)。其他的形式(`wake_up_interruptible`)限制它自己到处理一个可中断的睡眠。通常, 这 2 个是不用区分的(如果你使用可中断的睡眠); 实际上, 惯例是使用`wake_up` 如果你在使用 `wait_event` , `wake_up_interruptible` 如果你在使用`wait_event_interruptible`。

我们现在知道足够多来看一个简单的睡眠和唤醒的例子. 在这个例子代码中, 你可找到一个称为 `sleepy` 的模块。它实现一个有简单行为的设备:任何试图从这个设备读取的进程都被置为睡眠。无论何时一个进程写这个设备, 所有的睡眠进程被唤醒。这个行为由下面的 `read` 和 `write` 方法实现:

~~~c
static int sleepy_major = 0;

static DECLARE_WAIT_QUEUE_HEAD(wq);
static int flag = 0;

static ssize_t sleepy_read (struct file *filp, char __user *buf, size_t count, loff_t *pos)
{
	printk(KERN_DEBUG "process %i (%s) going to sleep\n",
			current->pid, current->comm);
	wait_event_interruptible(wq, flag != 0);
	flag = 0;
	printk(KERN_DEBUG "awoken %i (%s)\n", current->pid, current->comm);
	return 0; /* EOF */
}

static ssize_t sleepy_write (struct file *filp, const char __user *buf, size_t count,
		loff_t *pos)
{
	printk(KERN_DEBUG "process %i (%s) awakening the readers...\n",
			current->pid, current->comm);
	flag = 1;
	wake_up_interruptible(&wq);
	return count; /* succeed, to avoid retrial */
}
~~~

注意这个例子里 flag 变量的使用。因为 wait_event_interruptible 检查一个必须变为真的条件, 我们使用 flag 来创建那个条件。

有趣的是考虑当 `sleepy_write` 被调用时如果有 2 个进程在等待会发生什么。因为`sleepy_read` 重置 `flag` 为 0 一旦它醒来, 你可能认为醒来的第 2 个进程会立刻回到睡眠。在一个单处理器系统, 这几乎一直是发生的事情。但是重要的是要理解为什么你不能依赖这个行为。`wake_up_interruptible` 调用将使 2 个睡眠进程醒来。完全可能它们都注意到 flag 是非零, 在另一个有机会重置它之前。对于这个小模块, 这个竞争条件是不重要的。在一个真实的驱动中, 这种竞争可能导致少见的难于查找的崩溃。如果正确的操作要求只能有一个进程看到这个非零值, 它将必须以原子的方式被测试。我们将见到一个真正的驱动如何处理这样的情况。但首先我们必须开始另一个主题。















~~~c

~~~

> 1

------------------------

