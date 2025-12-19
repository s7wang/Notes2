# Linux 设备驱动

# 第三章 字符驱动

本章的目的是编写一个完整的字符设备驱动。我们开发一个字符驱动是因为这一类适合大部分简单硬件设备。字符驱动也比块驱动易于理解(我们在后续章节接触)。我们的最终目的是编写一个模块化的字符驱动, 但是我们不会在本章讨论模块化的事情。

贯串本章, 我们展示从一个真实设备驱动提取的代码片段: scull( Simple Character Utility forLoading Localities). scull 是一个字符驱动, 操作一块内存区域好像它是一个设备。在本章, 因为scull 的这个怪特性, 我们可互换地使用设备这个词和"scull 使用的内存区"。

scull 的优势在于它不依赖硬件。scull 只是操作一些从内核分配的内存。任何人都可以编译和运行scull, 并且 scull 在 Linux 运行的体系结构中可移植。另一方面, 这个设备除了演示内核和字符驱动的接口和允许用户运行一些测试之外, 不做任何有用的事情。



## 3.1 scull 的设计

编写驱动的第一步是定义驱动将要提供给用户程序的能力(机制)。因为我们的"设备"是计算机内存的一部分, 我们可自由做我们想做的事情。它可以是一个顺序的或者随机存取的设备, 一个或多个设备, 等等。为使 scull 作为一个模板来编写真实设备的真实驱动, 我们将展示给你如何在计算机内存上实现几个设备抽象, 每个有不同的个性。

scull 源码实现下面的设备. 模块实现的每种设备都被引用做一种类型。

* scull0 到 scull3

> 4 个设备, 每个由一个全局永久的内存区组成。全局意味着如果设备被多次打开,设备中含有的数据由所有打开它的文件描述符共享。永久意味着如果设备关闭又重新打开, 数据不会丢失。这个设备用起来有意思, 因为它可以用惯常的命令来存取和测试, 例如 cp, cat, 以及 I/O 重定向。

* scullpipe0 到 scullpipe3

> 4 个 FIFO (先入先出) 设备, 行为象管道. 一个进程读的内容来自另一个进程所写的。如果多个进程读同一个设备, 它们竞争数据。scullpipe 的内部将展示**阻塞读写**和**非阻塞读写**如何实现, 而不必采取中断。尽管真实的驱动使用硬件中断来同步它们的设备, 阻塞和非阻塞操作的主题是重要的并且与中断处理是分开的。(在第 10章涉及)。

* scullsingle  scullpriv  sculluid  scullwuid

> 这些设备与 scull0 相似, 但是在什么时候允许打开上有一些限制。
>
> 第一个( **snullsingle**) 只允许一次一个进程使用驱动, 而 **scullpriv** 对每个虚拟终端(或者 X 终端会话)是私有的, 因为每个控制台/终端上的进程有不同的内存区。**sculluid** 和 **scullwuid** 可以多次打开, 但是一次只能是一个用户; 前者返回一个"设备忙"错误, 如果另一个用户锁着设备, 而后者实现阻塞打开。这些 scull 的变体可能看来混淆了策略和机制, 但是它们值得看看, 因为一些实际设备需要这类管理。

每个 scull 设备演示了驱动的不同特色, 并且呈现了不同的难度。本章涉及 scull0 到scull3 的内部; 更高级的设备在第 6 章涉及。scullpipe 在"一个阻塞 I/O 例子"一节中描述, 其他的在"设备文件上的存取控制"中描述。



## 3.2 主次编号

字符设备通过文件系统中的名子来存取。那些名子称为文件系统的特殊文件, 或者设备文件, 或者文件系统的简单结点; 惯例上它们位于 `/dev` 目录。字符驱动的特殊文件由使用`ls -l` 的输出的第一列的"`c`"标识。块设备也出现在 `/dev` 中, 但是它们由"`b`"标识. 本章集中在字符设备, 但是下面的很多信息也适用于块设备。

如果你发出 `ls -l` 命令, 你会看到在设备文件项中有 2 个数(由一个逗号分隔)在最后修改日期前面, 这里通常是文件长度出现的地方。这些数字是给特殊设备的主次设备编号。下面的列表显示了一个典型系统上出现的几个设备。它们的主编号是 `1`, `4`, `7`, 和 `10`, 而次编号是 `1`, `3`, `5`, `64`, `65`, 和 `129`。

~~~
crw-rw-rw- 1 root root 	1, 	3 Apr 11  2002 null
crw------- 1 root root 	10, 1 Apr 11  2002 psaux
crw------- 1 root root 	4, 	1 Oct 28 03:04 tty1
crw-rw-rw- 1 root tty 	4, 64 Apr 11  2002 ttys0
crw-rw---- 1 root uucp 	4, 65 Apr 11  2002 ttyS1
crw--w---- 1 vcsa tty 	7, 	1 Apr 11  2002 vcs1
crw--w---- 1 vcsa tty 	7,129 Apr 11  2002 vcsa1
crw-rw-rw- 1 root root 	1, 	5 Apr 11  2002 zero
~~~

传统上, **主编号标识设备相连的驱动**。例如, `/dev/null` 和 `/dev/zero` 都由驱动 `1` 来管理, 而虚拟控制台和串口终端都由驱动 `4` 管理; 同样, vcs1 和 vcsa1 设备都由驱动 `7` 管理。**现代 Linux 内核允许多个驱动共享主编号, 但是你看到的大部分设备仍然按照一个主编号一个驱动的原则来组织。**

**次编号被内核用来决定引用哪个设备**. 依据你的驱动是如何编写的(如同我们下面见到的),你可以从内核得到一个你的设备的直接指针, 或者可以自己使用次编号作为本地设备数组的索引。不论哪个方法, 内核自己几乎不知道次编号的任何事情, 除了它们指向你的驱动实现的设备。



### 3.2.1 设备编号的内部表示

在内核中, `dev_t` 类型(在 `<linux/types.h>`中定义)用来持有设备编号 -- 主次部分都包括。对于 `2.6.0` 内核, `dev_t` 是 32 位的量, 12 位用作主编号, 20 位用作次编号。你的代码应当, 当然, 对于设备编号的内部组织从不做任何假设; 相反, 应当利用在`<linux/kdev_t.h>`中的一套宏定义。为获得一个 `dev_t` 的主或者次编号, 使用:

~~~c
MAJOR(dev_t dev);
MINOR(dev_t dev);
~~~

相反, 如果你有主次编号, 需要将其转换为一个 `dev_t`, 使用:

~~~c
MKDEV(int major, int minor);
~~~

注意, 2.6 内核能容纳有大量设备, 而以前的内核版本限制在 255 个主编号和 255 个次编号。有人认为这么宽的范围在很长时间内是足够的, 但是计算领域被这个特性的错误假设搞乱了。因此你应当希望 dev_t 的格式将来可能再次改变; 但是, 如果你仔细编写你的驱动, 这些变化不会是一个问题。



### 3.2.2 分配和释放设备编号

在建立一个字符驱动时你的驱动需要做的第一件事是获取一个或多个设备编号来使用。为此目的的必要的函数是 `register_chrdev_region`, 在 `<linux/fs.h>`中声明:

~~~c
int register_chrdev_region(dev_t first, unsigned int count, char *name);
~~~

> 这里, `first` 是你要分配的起始设备编号。`first` 的次编号部分常常是 `0`, 但是没有要求是那个效果。`count` 是你请求的连续设备编号的总数。注意, 如果 `count` 太大, 你要求的范围可能溢出到下一个次编号; 但是只要你要求的编号范围可用, 一切都仍然会正确工作.最后, `name` 是应当连接到这个编号范围的设备的名子; 它会出现在 `/proc/devices` 和 `sysfs` 中。
>
> 如同大部分内核函数, 如果分配成功进行, `register_chrdev_region` 的返回值是 `0`。出错的情况下, 返回一个负的错误码, 你不能存取请求的区域。

如果你确实事先知道你需要哪个设备编号, `register_chrdev_region` 工作得好。然而, 你常常不会知道你的设备使用哪个主编号; 在 Linux 内核开发社团中一直努力使用动态分配设备编号。内核会乐于动态为你分配一个主编号, 但是你必须使用一个不同的函数来请求这个分配。

~~~c
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, 
                        unsigned int count, char *name);
~~~

>  使用这个函数, `dev` 是一个只输出的参数, 它在函数成功完成时持有你的分配范围的第一个数。 `fisetminor` 应当是请求的第一个要用的次编号; 它常常是 0。 `count` 和 `name` 参数如同给 `request_chrdev_region` 的一样。

>  不管你任何分配你的设备编号, 你应当在不再使用它们时释放它。

>  设备编号的释放使用:`void unregister_chrdev_region(dev_t first, unsigned int count);`。

>  调用 `unregister_chrdev_region` 的地方常常是你的模块的 `cleanup` 函数。

上面的函数分配设备编号给你的驱动使用, 但是它们不告诉内核你实际上会对这些编号做什么。在用户空间程序能够存取这些设备号中一个之前, 你的驱动需要连接它们到它的实现设备操作的内部函数上。我们将描述如何简短完成这个连接, 但首先顾及一些必要的枝节问题。



### 3.2.3 主编号的动态分配

一些主设备编号是静态分派给最普通设备的, 这些设备的列表在内核源码树的`Documentation/devices.txt`  中。 分配给你的新驱动使用一个已经分配的静态编号的机会很小, 但是, 并且新编号没在分配。因此, 作为一个驱动编写者, 你有一个选择: 你可以简单地捡一个看来没有用的编号, 或者你以动态方式分配主编号。只要你是你的驱动的唯一用户就可以捡一个编号用; 一旦你的驱动更广泛的被使用了, 一个随机捡来的主编号将导致冲突和麻烦。

因此, 对于新驱动, 我们强烈建议你使用动态分配来获取你的主设备编号, 而不是随机选取一个当前空闲的编号。换句话说, 你的驱动应当几乎肯定地使用 `alloc_chrdev_region`,不是 `register_chrdev_region`。

动态分配的缺点是你无法提前创建设备节点, 因为分配给你的模块的主编号会变化。对于驱动的正常使用, 这不是问题, 因为一旦编号分配了, 你可从 `/proc/devices` 中读它。为使用动态主编号来加载一个驱动, 因此, 可使用一个简单的脚本来代替调用 `insmod`, 在调用 `insmod `后, 读取 `/proc/devices` 来创建特殊文件。

一个典型的 `/proc/devices` 文件看来如下:

~~~
Character devices:
1 mem
2 pty
3 ttyp
4 ttyS
6 lp
7 vcs
10 misc
13 input
14 sound
21 sg
180 usb
Block devices:
2 fd
8 sd
11 sr
65 sd
66 sd
~~~

因此加载一个已经安排了一个动态编号的模块的脚本, 可以使用一个工具来编写, 如 awk ,来从 /proc/devices 获取信息以创建 /dev 中的文件。下面的脚本, scull_load, 是 scull 发布的一部分。以模块发布的驱动的用户可以从系统的 rc.local 文件中调用这样一个脚本, 或者在需要模块时手工调用它。

~~~shell
#!/bin/sh
# $Id: scull_load,v 1.4 2004/11/03 06:19:49 rubini Exp $
module="scull"
device="scull"
mode="664"

# Group: since distributions do it differently, look for wheel or use staff
if grep -q '^staff:' /etc/group; then
    group="staff"
else
    group="wheel"
fi

# invoke insmod with all arguments we got
# and use a pathname, as insmod doesn't look in . by default
insmod ./$module.ko $* || exit 1

# retrieve major number
major=$(awk "\$2==\"$module\" {print \$1}" /proc/devices)

# Remove stale nodes and replace them, then give gid and perms
# Usually the script is shorter, it's scull that has several devices in it.

rm -f /dev/${device}[0-3]
mknod /dev/${device}0 c $major 0
mknod /dev/${device}1 c $major 1
mknod /dev/${device}2 c $major 2
mknod /dev/${device}3 c $major 3
ln -sf ${device}0 /dev/${device}
chgrp $group /dev/${device}[0-3] 
chmod $mode  /dev/${device}[0-3]
~~~

这个脚本可以通过重定义变量和调整 mknod 行来适用于另外的驱动。 这个脚本仅仅展示了创建 4 个设备, 因为 4 是 scull 源码中缺省的。

> 脚本的最后几行可能有些模糊:为什么改变设备的组和模式? 理由是这个脚本必须由超级用户运行, 因此新建的特殊文件由 `root` 拥有。许可位缺省的是只有 root 有写权限, 而任何人可以读。通常, 一个设备节点需要一个不同的存取策略, 因此在某些方面别人的存取权限必须改变。我们的脚本缺省是给一个用户组存取, 但是你的需求可能不同。在第 6 章的"设备文件的存取控制"一节中, `sculluid` 的代码演示了驱动如何能够强制它自己的对设备存取的授权。

还有一个 `scull_unload` 脚本来清理 `/dev` 目录并去除模块。

作为对使用一对脚本来加载和卸载的另外选择, 你可以编写一个 init 脚本, 准备好放在你的发布使用这些脚本的目录中。作为 scull 源码的一部分, 我们提供了一个相当完整和可配置的 init 脚本例子, 称为 `scull.init`; 它接受传统的参数 -- `start`, `stop`, 和`restart` -- 并且完成 `scull_load` 和 `scull_unload` 的角色。

如果反复创建和销毁 /dev 节点, 听来过分了, 有一个有用的办法。如果你在加载和卸载单个驱动, 你可以在你第一次使用你的脚本创建特殊文件之后, 只使用 rmmod 和 insmod: 这样动态编号不是随机的。并且你每次都可以使用所选的同一个编号, 如果你不加载任何别的动态模块。在开发中避免长脚本是有用的。但是这个技巧, 显然不能扩展到一次多于一个驱动。

安排主编号最好的方式, 我们认为, 是缺省使用动态分配, 而留给自己在加载时指定主编号的选项权, 或者甚至在编译时。scull 实现以这种方式工作; 它使用一个全局变量,`scull_major`, 来持有选定的编号(还有一个 `scull_minor` 给次编号)。这个变量初始化为`SCULL_MAJOR`, 定义在 `scull.h`。发布的源码中的 `SCULL_MAJOR` 的缺省值是 0, 意思是"使用动态分配"。用户可以接受缺省值或者选择一个特殊主编号, 或者在编译前修改宏定义或者在 insmod 命令行指定一个值给 `scull_major`。最后, 通过使用 `scull_load` 脚本, 用户可以在 `scull_load `的命令行传递参数给 `insmod`。

这是我们用在 scull 的源码中获取主编号的代码:

~~~c
// int scull_init_module(void)

/*
 * Get a range of minor numbers to work with, asking for a dynamic
 * major unless directed otherwise at load time.
 */
if (scull_major) {
    dev = MKDEV(scull_major, scull_minor);
    result = register_chrdev_region(dev, scull_nr_devs, "scull");
} else {
    result = alloc_chrdev_region(&dev, scull_minor, scull_nr_devs,
                                 "scull");
    scull_major = MAJOR(dev);
}
if (result < 0) {
    printk(KERN_WARNING "scull: can't get major %d\n", scull_major);
    return result;
}
~~~

本书使用的几乎所有例子驱动使用类似的代码来分配它们的主编号。



## 3.3 一些重要数据结构

如同你想象的, 注册设备编号仅仅是驱动代码必须进行的诸多任务中的第一个。我们将很快看到其他重要的驱动组件, 但首先需要涉及一个别的。大部分的基础性的驱动操作包括3 个重要的内核数据结构, 称为 file_operations, file, 和 inode。需要对这些结构的基本了解才能够做大量感兴趣的事情, 因此我们现在在进入如何实现基础性驱动操作的细节之前, 会快速查看每一个。



### 3.3.1 文件操作

到现在, 我们已经保留了一些设备编号给我们使用, 但是我们还没有连接任何我们设备操作到这些编号上。 `file_operation` 结构是一个字符驱动如何建立这个连接. 这个结构, 定义在 `<linux/fs.h>`, 是一个函数指针的集合。 每个打开文件(内部用一个 file 结构来代表, 稍后我们会查看)与它自身的函数集合相关连( 通过包含一个称为 `f_op` 的成员, 它指向一个 `file_operations` 结构)。这些操作大部分负责实现系统调用, 因此, 命名为 `open`,`read`, 等等。我们可以认为文件是一个"对象"并且其上的函数操作称为它的"方法", 使用面向对象编程的术语来表示一个对象声明的用来操作对象的动作。这是我们在 Linux 内核中看到的第一个面向对象编程的现象, 后续章中我们会看到更多。

传统上, 一个 `file_operation` 结构或者其一个指针称为 `fops`( 或者它的一些变体). 结构中的每个成员必须指向驱动中的函数, 这些函数实现一个特别的操作, 或者对于不支持的操作留置为 `NULL`。当指定为 `NULL` 指针时内核的确切的行为是每个函数不同的, 如同本节后面的列表所示。

下面的列表介绍了一个应用程序能够在设备上调用的所有操作。我们已经试图保持列表简短, 这样它可作为一个参考, 只是总结每个操作和在 NULL 指针使用时的缺省内核行为。

在你通读 `file_operations` 方法的列表时, 你会注意到不少参数包含字串 `__user`。这种注解是一种文档形式, 注意, 一个指针是一个不能被直接解引用的用户空间地址。对于正常的编译, `__user` 没有效果, 但是它可被外部检查软件使用来找出对用户空间地址的错误使用。

本章剩下的部分, 在描述一些**其他重要数据结构**后, 解释了最重要操作的角色并且给了提示, 告诫和真实代码例子。我们推迟讨论更复杂的操作到后面章节, 因为我们还不准备深入如内存管理, 阻塞操作, 和异步通知。

~~~c
struct module *owner
~~~

> 第一个 `file_operations` 成员根本不是一个操作; 它是一个指向拥有这个结构的模块的指针。这个成员用来在它的操作还在被使用时阻止模块被卸载。几乎所有时间中, 它被简单初始化为 `THIS_MODULE`, 一个在 `<linux/module.h>` 中定义的宏。

---------------

~~~c
loff_t (*llseek) (struct file *, loff_t, int);
~~~

> `llseek` 方法用作改变文件中的当前读/写位置, 并且新位置作为(正的)返回值。`loff_t` 参数是一个"`long offset`", 并且就算在 32 位平台上也至少 64 位宽。错误由一个负返回值指示。如果这个函数指针是 `NULL`, `seek` 调用会以潜在地无法预知的方式修改 file 结构中的位置计数器( 在"file 结构" 一节中描述)。

---------------

~~~c
ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
~~~

> 用来从设备中获取数据。在这个位置的一个空指针导致 `read` 系统调用以` -EINVAL`("Invalid argument") 失败。一个非负返回值代表了成功读取的字节数( 返回值是一个 "`signed size`" 类型, 常常是目标平台本地的整数类型)。

---------------

~~~c
ssize_t (*aio_read)(struct kiocb *, char __user *, size_t, loff_t);
~~~

> 初始化一个异步读 -- 可能在函数返回前不结束的读操作。如果这个方法是 `NULL`,所有的操作会由 `read` 代替进行(同步地)。

---------------

~~~c
ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
~~~

> 发送数据给设备. 如果 `NULL`, `-EINVAL` 返回给调用 `write` 系统调用的程序。如果非负, 返回值代表成功写的字节数。

---------------

~~~c
ssize_t (*aio_write)(struct kiocb *, const char __user *, size_t, loff_t *);
~~~

> 初始化设备上的一个异步写。

---------------

~~~c
int (*readdir) (struct file *, void *, filldir_t);
~~~

> 对于设备文件这个成员应当为 `NULL`; 它用来读取目录, 并且仅对文件系统有用。

---------------

~~~c
unsigned int (*poll) (struct file *, struct poll_table_struct *);
~~~

> `poll` 方法是 3 个系统调用的后端: `poll`, `epoll`, 和 `select`, 都用作查询对一个或多个文件描述符的读或写是否会阻塞。 poll 方法应当返回一个位掩码指示是否非阻塞的读或写是可能的, 并且, 可能地, 提供给内核信息用来使调用进程睡眠直到`I/O` 变为可能。如果一个驱动的 poll 方法为 NULL, 设备假定为不阻塞地可读可写。

---------------

~~~c
int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
~~~

> `ioctl`   系统调用提供了发出设备特定命令的方法(例如格式化软盘的一个磁道, 这不是读也不是写。另外, 几个 `ioctl`  命令被内核识别而不必引用 `fops` 表。 如果设备不提供 `ioctl` 方法,  对于任何未事先定义的请求  (`-ENOTTY`, "设备无这样的`ioctl`"), 系统调用返回一个错误。

---------------

~~~c
int (*mmap) (struct file *, struct vm_area_struct *);
~~~

> `mmap` 用来请求将设备内存映射到进程的地址空间。如果这个方法是 `NULL`, `mmap` 系统调用返回 `-ENODEV`。

---------------

~~~c
int (*open) (struct inode *, struct file *);
~~~

> 尽管这常常是对设备文件进行的第一个操作, 不要求驱动声明一个对应的方法。如果这个项是 `NULL`, 设备打开一直成功, 但是你的驱动不会得到通知。

---------------

~~~c
int (*flush) (struct file *);
~~~

> `flush` 操作在进程关闭它的设备文件描述符的拷贝时调用; 它应当执行(并且等待)设备的任何未完成的操作。这个必须不要和用户查询请求的 `fsync` 操作混淆了。当前, `flush` 在很少驱动中使用; SCSI 磁带驱动使用它。例如, 为确保所有写的数据在设备关闭前写到磁带上。如果 `flush` 为 `NULL`, 内核简单地忽略用户应用程序的请求。

---------------

~~~c
int (*release) (struct inode *, struct file *);
~~~

> 在文件结构被释放时引用这个操作。如同 `open`, `release `可以为 `NULL`。

---------------

~~~c
int (*fsync) (struct file *, struct dentry *, int);
~~~

> 这个方法是 `fsync` 系统调用的后端, 用户调用来刷新任何挂着的数据. 如果这个指针是 `NULL`, 系统调用返回 `-EINVAL`。

---------------

~~~c
int (*aio_fsync)(struct kiocb *, int);
~~~

> 这是 `fsync` 方法的异步版本。

---------------

~~~c
int (*fasync) (int, struct file *, int);
~~~

> 这个操作用来通知设备它的 `FASYNC` 标志的改变。异步通知是一个高级的主题, 在第 6 章中描述。这个成员可以是 `NULL` 如果驱动不支持异步通知。

---------------

~~~c
int (*lock) (struct file *, int, struct file_lock *);
~~~

> `lock` 方法用来实现文件加锁; 加锁对常规文件是必不可少的特性, 但是设备驱动几乎从不实现它。

---------------

~~~c
ssize_t (*readv) (struct file *, const struct iovec *, unsigned long, loff_t *);
ssize_t (*writev) (struct file *, const struct iovec *, unsigned long, loff_t *);
~~~

> 这些方法实现发散/汇聚读和写操作。应用程序偶尔需要做一个包含多个内存区的单个读或写操作; 这些系统调用允许它们这样做而不必对数据进行额外拷贝。如果这些函数指针为 `NULL`, `read` 和 `write` 方法被调用( 可能多于一次 )。

---------------

~~~c
ssize_t (*sendfile)(struct file *, loff_t *, size_t, read_actor_t, void *);
~~~

> 这个方法实现 `sendfile` 系统调用的读, 使用最少的拷贝从一个文件描述符搬移数据到另一个. 例如, 它被一个需要发送文件内容到一个网络连接的 `web` 服务器使用。设备驱动常常使 `sendfile` 为 `NULL`。

---------------

~~~c
ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
~~~

> `sendpage` 是 `sendfile` 的另一半; 它由内核调用来发送数据, 一次一页, 到对应的文件。设备驱动实际上不实现 `sendpage`。

---------------

~~~c
unsigned long (*get_unmapped_area)(struct file *, unsigned long, 
                                   unsigned long, unsigned long, 
                                   unsigned long);
~~~

> 这个方法的目的是在进程的地址空间找一个合适的位置来映射在底层设备上的内存段中。这个任务通常由内存管理代码进行; 这个方法存在为了使驱动能强制特殊设备可能有的任何的对齐请求。大部分驱动可以置这个方法为 `NULL`。

---------------

~~~c
int (*check_flags)(int);
~~~

> 这个方法允许模块检查传递给 `fnctl(F_SETFL...)` 调用的标志。

----------------------------


~~~c
int (*dir_notify)(struct file *, unsigned long);
~~~

> 这个方法在应用程序使用 `fcntl` 来请求目录改变通知时调用。只对文件系统有用;驱动不需要实现 `dir_notify`。

----------------

scull 设备驱动只实现最重要的设备方法。它的 `file_operations` 结构是如下初始化的:

~~~c
struct file_operations scull_fops = {
	.owner =    THIS_MODULE,
	.llseek =   scull_llseek,
	.read =     scull_read,
	.write =    scull_write,
	.unlocked_ioctl = scull_ioctl,
	.open =     scull_open,
	.release =  scull_release,
};
~~~

这个声明使用标准的 C 标记式结构初始化语法。这个语法是首选的, 因为它使驱动在结构定义的改变之间更加可移植, 并且, 有争议地, 使代码更加紧凑和可读。标记式初始化允许结构成员重新排序; 在某种情况下, 真实的性能提高已经实现, 通过安放经常使用的成员的指针在相同硬件高速存储行中。



### 3.3.2 文件结构

`struct file,` 定义于` <linux/fs.h>`, 是设备驱动中第二个最重要的数据结构。**注意 `file`与用户空间程序的 `FILE` 指针没有任何关系。**一个 `FILE` 定义在 C 库中, 从不出现在内核代码中。一个 `struct file`, 另一方面, 是一个内核结构, 从不出现在用户程序中。

文件结构代表一个打开的文件。(它不特定给设备驱动; 系统中每个打开的文件有一个关联的 `struct file` 在内核空间)。它由内核在 `open` 时创建, 并传递给在文件上操作的任何函数, 直到最后的关闭。在文件的所有实例都关闭后, 内核释放这个数据结构。

> 在内核源码中, `struct file` 的指针常常称为 `file` 或者 `filp`("file pointer")。我们将一直称这个指针为 `filp` 以避免和结构自身混淆。因此, `file` 指的是结构, 而 `filp` 是结构指针。

`struct file` 的最重要成员在这展示. 如同在前一节, 第一次阅读可以跳过这个列表。但是, 在本章后面, 当我们面对一些真实 C 代码时, 我们将更详细讨论这些成员。

~~~c
mode_t f_mode;
~~~

> 文件模式确定文件是可读的或者是可写的(或者都是), 通过位 `FMODE_READ` 和 `FMODE_WRITE`。你可能想在你的 `open` 或者 `ioctl` 函数中检查这个成员的读写许可,但是你不需要检查读写许可, 因为内核在调用你的方法之前检查。当文件还没有为那种存取而打开时读或写的企图被拒绝, 驱动甚至不知道这个情况。

------------------------

~~~c
loff_t f_pos;
~~~

> 当前读写位置。`loff_t` 在所有平台都是 64 位( 在 gcc 术语里是 `long long` )。驱动可以读这个值, 如果它需要知道文件中的当前位置, 但是正常地不应该改变它;读和写应当使用它们作为最后参数而收到的指针来更新一个位置, 代替直接作用于`filp->f_pos`。这个规则的一个例外是在 `llseek` 方法中, 它的目的就是改变文件位置。

------------------------

~~~c
unsigned int f_flags;
~~~

> 这些是文件标志, 例如 `O_RDONLY`, `O_NONBLOCK`, 和 `O_SYNC`。驱动应当检查`O_NONBLOCK` 标志来看是否是请求非阻塞操作( 我们在第一章的"阻塞和非阻塞操作"一节中讨论非阻塞 `I/O` ); 其他标志很少使用。特别地, 应当检查读/写许可, 使用`f_mode` 而不是 `f_flags`。所有的标志在头文件 `<linux/fcntl.h>` 中定义。

------------------------

~~~c
struct file_operations *f_op;
~~~

> 和文件关联的操作. 内核安排指针作为它的 `open` 实现的一部分, 接着读取它当它需要分派任何的操作时。`filp->f_op` 中的值从不由内核保存为后面的引用; 这意味着你可改变你的文件关联的文件操作, 在你返回调用者之后新方法会起作用。例如,关联到主编号 1 (`/dev/null`, `/dev/zero`, 等等)的 `open` 代码根据打开的次编号来替代 `filp->f_op` 中的操作。这个做法允许实现几种行为, 在同一个主编号下而不必在每个系统调用中引入开销。替换文件操作的能力是面向对象编程的"方法重载"的内核对等体。

------------------------

~~~c
void *private_data;
~~~

> `open` 系统调用设置这个指针为 `NULL` ,在为驱动调用 `open` 方法之前。你可自由使用这个成员或者忽略它; 你可以使用这个成员来指向分配的数据, 但是接着你必须记住在内核销毁文件结构之前, 在 `release` 方法中释放那个内存。`private_data` 是一个有用的资源, 在系统调用间保留状态信息, 我们大部分例子模块都使用它。

------------------------

~~~c
struct dentry *f_dentry;
~~~

> 关联到文件的目录入口( `dentry` )结构。设备驱动编写者正常地不需要关心 `dentry` 结构, 除了作为 `filp->f_dentry->d_inode` 存取 `inode` 结构。真实结构有多几个成员, 但是它们对设备驱动没有用处。我们可以安全地忽略这些成员,因为驱动从不创建文件结构; 它们真实存取别处创建的结构。

------------------------



### 3.3.3 inode 结构

`inode` 结构由内核在内部用来表示文件。 因此, 它和代表打开文件描述符的文件结构是不同的。可能有代表单个文件的多个打开描述符的许多文件结构, 但是它们都指向一个单个`inode` 结构。`inode` 结构包含大量关于文件的信息。作为一个通用的规则, 这个结构只有 2 个成员对于编写驱动代码有用:

~~~c
dev_t i_rdev;
~~~

> 对于代表设备文件的节点, 这个成员包含实际的设备编号。

------------------------

~~~c
struct cdev *i_cdev;
~~~

> `struct cdev` 是内核的内部结构, 代表字符设备; 这个成员包含一个指针, 指向这个结构, 当节点指的是一个字符设备文件时。

------------------------

`i_rdev` 类型在 2.5 开发系列中改变了, 破坏了大量的驱动。作为一个鼓励更可移植编程的方法, 内核开发者已经增加了 2 个宏, 可用来从一个 `inode` 中获取主次编号:

~~~c
unsigned int iminor(struct inode *inode);
unsigned int imajor(struct inode *inode);
~~~

为了不要被下一次改动抓住, 应当使用这些宏代替直接操作 `i_rdev`。



## 3.4 字符设备注册

如我们提过的, 内核在内部使用类型 `struct cdev` 的结构来代表字符设备。在内核调用你的设备操作前, 你编写分配并注册一个或几个这些结构。为此, 你的代码应当包含`<linux/cdev.h>`, 这个结构和它的关联帮助函数定义在这里。

有 2 种方法来分配和初始化一个这些结构. 如果你想在运行时获得一个独立的 `cdev` 结构,你可以为此使用这样的代码:

~~~c
struct cdev *my_cdev = cdev_alloc();
my_cdev->ops = &my_fops;
~~~

但是, 偶尔你会想将 `cdev` 结构嵌入一个你自己的设备特定的结构; `scull` 这样做了。在这种情况下, 你应当初始化你已经分配的结构, 使用:

~~~c
void cdev_init(struct cdev *cdev, struct file_operations *fops);
~~~

任一方法, 有一个其他的 `struct cdev` 成员你需要初始化。像 `file_operations` 结构,`struct cdev` 有一个拥有者成员, 应当设置为 `THIS_MODULE`。一旦 `cdev` 结构建立, 最后的步骤是把它告诉内核, 调用:

~~~c
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
~~~

这里, `dev` 是 `cdev` 结构, `num` 是这个设备响应的第一个设备号, `count` 是应当关联到设备的设备号的数目。常常 `count` 是 1, 但是有多个设备号对应于一个特定的设备的情形。例如, 设想 `SCSI` 磁带驱动, 它允许用户空间来选择操作模式(例如密度), 通过安排多个次编号给每一个物理设备。

在使用 `cdev_add` 是有几个重要事情要记住。第一个是这个调用可能失败。如果它返回一个负的错误码, 你的设备没有增加到系统中。它几乎会一直成功, 但是, 并且带起了其他的点: `cdev_add` 一返回, 你的设备就是"活的"并且内核可以调用它的操作. 除非你的驱动完全准备好处理设备上的操作, 你不应当调用 `cdev_add`。

为从系统去除一个字符设备, 调用:

~~~c
void cdev_del(struct cdev *dev);
~~~

显然, 你不应当在传递给 `cdev_del` 后存取 `cdev` 结构。

### 3.4.1 scull 中的设备注册

在内部, `scull` 使用一个 `struct scull_dev` 类型的结构表示每个设备。这个结构定义为:

~~~c
struct scull_dev {
	struct scull_qset *data;  /* Pointer to first quantum set */
	int quantum;              /* the current quantum size */
	int qset;                 /* the current array size */
	unsigned long size;       /* amount of data stored here */
	unsigned int access_key;  /* used by sculluid and scullpriv */
	struct mutex lock;     /* mutual exclusion semaphore     */
	struct cdev cdev;	  /* Char device structure		*/
};
~~~

我们在遇到它们时讨论结构中的各个成员, 但是现在, 我们关注于 `cdev`, 我们的设备与内核接口的 `struct cdev`。这个结构必须初始化并且如上所述添加到系统中; 处理这个任务的 scull 代码是:

~~~c
/*
 * Set up the char_dev structure for this device.
 */
static void scull_setup_cdev(struct scull_dev *dev, int index)
{
	int err, devno = MKDEV(scull_major, scull_minor + index);
    
	cdev_init(&dev->cdev, &scull_fops);
	dev->cdev.owner = THIS_MODULE;
	err = cdev_add (&dev->cdev, devno, 1);
	/* Fail gracefully if need be */
	if (err)
		printk(KERN_NOTICE "Error %d adding scull%d", err, index);
}
~~~

因为 `cdev` 结构嵌在 `struct scull_dev` 里面, `cdev_init` 必须调用来进行那个结构的初始化。



### 3.4.2 老方法

现在（2025-12-03）已经距离2.6版本内核很久了，这个机制可以不用了解。

如果你深入浏览 2.6 内核的大量驱动代码, 你可能注意到有许多字符驱动不使用我们刚刚描述过的 `cdev` 接口。你见到的是还没有更新到 2.6 内核接口的老代码。因为那个代码实际上能用, 这个更新可能很长时间不会发生。为完整, 我们描述老的字符设备注册接口,但是新代码不应当使用它; 这个机制在将来内核中可能会消失。

注册一个字符设备的经典方法是使用:

~~~c
int register_chrdev(unsigned int major, const char *name, struct
file_operations *fops);
~~~

这里, `major` 是感兴趣的主编号, `name` 是驱动的名子(出现在 `/proc/devices`), `fops` 是缺省的 `file_operations` 结构.。一个对 `register_chrdev `的调用为给定的主编号注册 `0~255` 的次编号, 并且为每一个建立一个缺省的 `cdev` 结构。使用这个接口的驱动必须准备好处理对所有 256 个次编号的 `open` 调用( 不管它们是否对应真实设备 ), 它们不能使用大于 `255` 的主或次编号。

如果你使用 `register_chrdev`, 从系统中去除你的设备的正确的函数是:

~~~c
int unregister_chrdev(unsigned int major, const char *name);
~~~

major 和 name 必须和传递给 register_chrdev 的相同, 否则调用会失败.



## 3.5 open 和 release

到此我们已经快速浏览了这些成员, 我们开始在真实的 scull 函数中使用它们。

### 3.5.1 open 方法
`open` 方法提供给驱动来做任何的初始化来准备后续的操作。在大部分驱动中, `open` 应当进行下面的工作:

* 检查设备特定的错误(例如设备没准备好, 或者类似的硬件错误；
* 如果它第一次打开, 初始化设备；
* 如果需要, 更新 `f_op` 指针；
* 分配并填充要放进 `filp->private_data` 的任何数据结构。

但是, 事情的第一步常常是确定打开哪个设备. 记住 open 方法的原型是:

~~~c
int (*open)(struct inode *inode, struct file *filp);
~~~

`inode` 参数有我们需要的信息,以它的 `i_cdev` 成员的形式, 里面包含我们之前建立的 `cdev` 结构。唯一的问题是通常我们不想要 `cdev` 结构本身, 我们需要的是包含 `cdev` 结构的 `scull_dev` 结构。C 语言使程序员玩弄各种技巧来做这种转换; 但是, 这种技巧编程是易出错的, 并且导致别人难于阅读和理解代码。幸运的是, 在这种情况下, 内核 hacker已经为我们实现了这个技巧, 以 `container_of` 宏的形式, 在 `<linux/kernel.h>` 中定义:

~~~c
container_of(pointer, container_type, container_field);
~~~

这个宏使用一个指向 `container_field` 类型的成员的指针, 它在一个 `container_type` 类型的结构中, 并且返回一个指针指向包含结构。在 `scull_open`, 这个宏用来找到适当的设备结构。

识别打开的设备的另外的方法是查看存储在 `inode` 结构的次编号。如果你使用 `register_chrdev` 注册你的设备, 你必须使用这个技术。确认使用 `iminor` 从 `inode` 结构中获取次编号, 并且确定它对应一个你的驱动真正准备好处理的设备。

~~~c
int scull_open(struct inode *inode, struct file *filp)
{
	struct scull_dev *dev; /* device information */

	dev = container_of(inode->i_cdev, struct scull_dev, cdev);
	filp->private_data = dev; /* for other methods */

	/* now trim to 0 the length of the device if open was write-only */
	if ( (filp->f_flags & O_ACCMODE) == O_WRONLY) {
		if (mutex_lock_interruptible(&dev->lock))
			return -ERESTARTSYS;
		scull_trim(dev); /* ignore errors */
		mutex_unlock(&dev->lock);
	}
	return 0;          /* success */
}
~~~

`container_of` 是 Linux 内核中**非常经典且重要的宏**，用来：

> **已知结构体中某个成员的地址，反推出整个结构体的首地址。**

一旦它找到 `scull_dev` 结构, scull 在文件结构的 `private_data` 成员中存储一个它的指针, 为以后更易存取。

代码看来相当稀疏, 因为在调用 open 时它没有做任何特别的设备处理。它不需要, 因为scull 设备设计为全局的和永久的。特别地, 没有如"在第一次打开时初始化设备"等动作,因为我们不为 scull 保持打开计数。唯一在设备上的真实操作是当设备为写而打开时将它截取为长度为 0。这样做是因为, 在设计上, 用一个短的文件覆盖一个 scull 设备导致一个短的设备数据区。这类似于为写而打开一个常规文件, 将其截短为 0。如果设备为读而打开, 这个操作什么都不做。



### 3.5.2 release 方法

`release` 方法的角色是 `open` 的反面。有时你会发现方法的实现称为 `device_close`, 而不是 `device_release`。任一方式, 设备方法应当进行下面的任务:

* 释放 `open` 分配在 `filp->private_data` 中的任何东西；
* 在最后的 `close` 关闭设备。

scull 的基本形式没有硬件去关闭, 因此需要的代码是最少的:

~~~c
int scull_release(struct inode *inode, struct file *filp)
{
	return 0;
}
~~~

你可能想知道当一个设备文件关闭次数超过它被打开的次数会发生什么。毕竟, dup 和fork 系统调用不调用 open 来创建打开文件的拷贝; 每个拷贝接着在程序终止时被关闭。例如, 大部分程序不打开它们的 stdin 文件(或设备), 但是它们都以关闭它结束。当一个打开的设备文件已经真正被关闭时驱动如何知道?

**答案简单**: 不是每个 close 系统调用引起调用 release 方法。只有真正释放设备数据结构的调用会调用这个方法 -- 因此得名。 内核维持一个文件结构被使用多少次的计数。fork 和 dup 都不创建新文件(只有 open 这样); 它们只递增正存在的结构中的计数。close 系统调用仅在文件结构计数掉到 0 时执行 release 方法, 这在结构被销毁时发生。release 方法和 close 系统调用之间的这种关系保证了你的驱动一次 open 只看到一次release。

**注意, flush 方法在每次应用程序调用 close 时都被调用。但是, 很少驱动实现 flush,因为常常在 close 时没有什么要做, 除非调用 release。**

如你会想到的, 前面的讨论即便是应用程序没有明显地关闭它打开的文件也适用: 内核在进程 exit 时自动关闭了任何文件, 通过在内部使用 close 系统调用。



## 3.6 scull 的内存使用

在介绍读写操作前, 我们最好看看如何以及为什么 scull 进行内存分配。"如何"是需要全面理解代码, "为什么"演示了驱动编写者需要做的选择, 尽管 scull 明确地不是典型设备。

本节只处理 scull 中的内存分配策略, 不展示给你编写真正驱动需要的硬件管理技能。这些技能在第 9 章和第 10 章介绍。因此, 你可跳过本章, 如果你不感兴趣于理解面向内存的 scull 驱动的内部工作。

scull 使用的内存区, 也称为一个设备, 长度可变。你写的越多, 它增长越多; 通过使用一个短文件覆盖设备来进行修整。scull 驱动引入 2 个核心函数来管理 Linux 内核中的内存。这些函数, 定义在`<linux/slab.h>`, 是:

~~~c
void *kmalloc(size_t size, int flags);
void kfree(void *ptr);
~~~

对 `kmalloc` 的调用试图分配 `size` 字节的内存; 返回值是指向那个内存的指针或者如果分配失败为`NULL`。`flags` 参数用来描述内存应当如何分配; 我们在第 8 章详细查看这些标志。对于现在, 我们一直使用 `GFP_KERNEL`。分配的内存应当用 `kfree` 来释放。你应当从不传递任何不是从 `kmalloc` 获得的东西给 `kfree`。但是, 传递一个 `NULL` 指针给 `kfree`是合法的。

==`kmalloc` 不是最有效的分配大内存区的方法(见第 8 章), 所以挑选给 scull 的实现不是一个特别巧妙的。==一个巧妙的源码实现可能更难阅读, 而本节的目标是展示读和写, 不是内存管理。这是为什么代码只是使用 `kmalloc` 和 `kfree` 而不依靠整页的分配, 尽管这个方法会更有效。

在 flip 一边, 我们不想限制"设备"区的大小, 由于理论上的和实践上的理由。理论上,给在被管理的数据项施加武断的限制总是个坏想法。实践上, scull 可用来暂时地吃光你系统中的内存, 以便运行在低内存条件下的测试。运行这样的测试可能会帮助你理解系统的内部。你可以使用命令 `cp /dev/zero /dev/scull0` 来用 scull 吃掉所有的真实 RAM,并且你可以使用 dd 工具来选择贝多少数据给 scull 设备。

在 scull, 每个设备是一个指针链表, 每个都指向一个 scull_dev 结构。每个这样的结构,缺省地, 指向最多 4 兆字节, 通过一个中间指针数组。发行代码使用一个 1000 个指针的数组指向每个 4000 字节的区域。我们称每个内存区域为一个量子, 数组(或者它的长度)为一个量子集。一个 scull 设备和它的内存区如图一个 scull 设备的布局所示。

~~~
[scull_dev head] -> [scull_dev] -> [scull_dev] -> NULL
						| 				|
						| 				`-> qset array (len=1000)
						|
						`-> qset array (len=1000)

每个 scull_dev:

scull_dev {
	struct scull_qset *data; // 指向量子集指针数组（qset array）
	struct scull_dev *next; // 链表中的下一个设备
	int quantum; // 单个量子的大小（默认 4000 bytes）
	int qset; // qset 数组的长度（默认 1000）
	/* 其他成员... */
}

qset array (示意，长度 = 1000):

index: 	0 		1 		2 		... 	999
		| 		| 		| 				|
		ptr0 	ptr1 	ptr2 	... 	ptr999
		| 		| 		| 				|
		quant0 	quant1 	quant2 	... 	quant999
		(4KB) 	(4KB) 	(4KB) 	... 	(4KB)

因此：每个 scull_dev 最多 = qset * quantum = 1000 * 4000 ≈ 4,000,000 bytes (~4MB)
~~~

选定的数字是这样, 在 scull 中写单个一个字节消耗 8000 或 12,000 KB 内存: 4000 是量子, 4000 或者 8000 是量子集(根据指针在目标平台上是用 32 位还是 64 位表示)。相反, 如果你写入大量数据, 链表的开销不是太坏。每 4 MB 数据只有一个链表元素, 设备的最大尺寸受限于计算机的内存大小。

为量子和量子集选择合适的值是一个策略问题, 而不是机制, 并且优化的值依赖于设备如何使用。因此, scull 驱动不应当强制给量子和量子集使用任何特别的值。在 scull 中,用户可以掌管改变这些值, 有几个途径:编译时间通过改变 `scull.h` 中的宏`SCULL_QUANTUM` 和 `SCULL_QSET`, 在模块加载时设定整数值 `scull_quantum` 和 `scull_qset`,或者使用 `ioctl` 在运行时改变当前值和缺省值。

**使用宏定义和一个整数值来进行编译时和加载时配置, 是对于如何选择主编号的回忆。我们在驱动中任何与策略相关或专断的值上运用这个技术。**

余下的唯一问题是如果选择缺省值。在这个特殊情况下, 问题是找到最好的平衡, 由填充了一半的量子和量子集导致内存浪费, 如果量子和量子集小的情况下分配释放和指针连接引起开销。另外, `kmalloc` 的内部设计应当考虑进去。(现在我们不追求这点, 不过;`kmalloc` 的内部在第 8 章探索。) 缺省值的选择来自假设测试时可能有大量数据写进scull, 尽管设备的正常使用最可能只传送几 KB 数据。

我们已经见过内部代表我们设备的 `scull_dev` 结构。结构的 `quantum` 和 `qset` 分别代表设备的量子和量子集大小。实际数据, 但是, 是由一个不同的结构跟踪, 我们称为 `struct scull_qset`:

~~~c
/*
 * Representation of scull quantum sets.
 */
struct scull_qset {
	void **data;
	struct scull_qset *next;
};

~~~

下一个代码片段展示了实际中 `struct scull_dev` 和 `struct scull_qset` 是如何被用来持有数据的。`sucll_trim` 函数负责释放整个数据区, 由 `scull_open` 在文件为写而打开时调用。它简单地遍历列表并且释放它发现的任何量子和量子集。

~~~c
/*
 * Empty out the scull device; must be called with the device
 * semaphore held.
 */
int scull_trim(struct scull_dev *dev)
{
	struct scull_qset *next, *dptr;
	int qset = dev->qset;   /* "dev" is not-null */
	int i;

	for (dptr = dev->data; dptr; dptr = next) { /* all the list items */
		if (dptr->data) {
			for (i = 0; i < qset; i++)
				kfree(dptr->data[i]);
			kfree(dptr->data);
			dptr->data = NULL;
		}
		next = dptr->next;
		kfree(dptr);
	}
	dev->size = 0;
	dev->quantum = scull_quantum;
	dev->qset = scull_qset;
	dev->data = NULL;
	return 0;
}
~~~

scull_trim 也用在模块清理函数中, 来归还 scull 使用的内存给系统。



## 3.7 读和写

读和写方法都进行类似的任务, 就是, 从和到应用程序代码拷贝数据。因此, 它们的原型相当相似, 可以同时介绍它们:

~~~c
ssize_t read(struct file *filp, char __user *buff, size_t count, loff_t *offp);
ssize_t write(struct file *filp, const char __user *buff, size_t count, loff_t *offp);
~~~

对于 2 个方法, filp 是文件指针, `count` 是请求的传输数据大小。`buff` 参数指向持有被写入数据的缓存, 或者放入新数据的空缓存。最后, `offp` 是一个指针指向一个"`long offset type`"对象, 它指出用户正在存取的文件位置。返回值是一个"`signed size type`";它的使用在后面讨论。

让我们重复一下, `read` 和 `write` 方法的 `buff` 参数是用户空间指针。因此, 它不能被内核代码直接解引用。这个限制有几个理由:

* 依赖于你的驱动运行的体系, 以及内核被如何配置的, 用户空间指针当运行于内核模式可能根本是无效的。可能没有那个地址的映射, 或者它可能指向一些其他的随机数据。
* 就算这个指针在内核空间是同样的东西, 用户空间内存是分页的, 在做系统调用时这个内存可能没有在 RAM 中。试图直接引用用户空间内存可能产生一个页面错, 这是内核代码不允许做的事情。结果可能是一个"oops", 导致进行系统调用的进程死亡。
* 置疑中的指针由一个用户程序提供, 它可能是错误的或者恶意的。如果你的驱动盲目地解引用一个用户提供的指针, 它提供了一个打开的门路使用户空间程序存取或覆盖系统任何地方的内存。如果你不想负责你的用户的系统的安全危险, 你就不能直接解引用用户空间指针。

显然, 你的驱动必须能够存取用户空间缓存以完成它的工作。但是, 为安全起见这个存取必须使用特殊的, 内核提供的函数。我们介绍几个这样的函数(定义于` <asm/uaccess.h>`),剩下的在第一章"使用 `ioctl` 参数"一节中。它们使用一些特殊的, 依赖体系的技巧来确保内核和用户空间的数据传输安全和正确。

scull 中的读写代码需要拷贝一整段数据到或者从用户地址空间。这个能力由下列内核函数提供, 它们拷贝一个任意的字节数组, 并且位于大部分读写实现的核心中。

~~~c
unsigned long copy_to_user(void __user *to,const void *from,unsigned long count);
unsigned long copy_from_user(void *to,const void __user *from,unsigned long count);
~~~

尽管这些函数表现象正常的 `memcpy` 函数, 必须加一点小心在从内核代码中存取用户空间。寻址的用户也当前可能不在内存, 虚拟内存子系统会使进程睡眠在这个页被传送到位时。例如, 这发生在必须从交换空间获取页的时候。 对于驱动编写者来说, 最终结果是任何存取用户空间的函数必须是可重入的, 必须能够和其他驱动函数并行执行, 并且, 特别的,必须在一个它能够合法地睡眠的位置。我们在第 5 章再回到这个主题。

用户空间存取和无效用户空间指针的主题有些高级, 在第 6 章讨论。然而, 值得注意的是如果你不需要检查用户空间指针, 你可以调用 `__copy_to_user` 和 `__copy_from_user` 来代替。这是有用处的, 例如, 如果你知道你已经检查了这些参数。但是, 要小心; 事实上,如果你不检查你传递给这些函数的用户空间指针, 那么你可能造成内核崩溃和/或安全漏洞。至于实际的设备方法, `read` 方法的任务是从设备拷贝数据到用户空间(使用`copy_to_user`), 而 `write` 方法必须从用户空间拷贝数据到设备(使用 `copy_from_user`)。每个 `read` 或 `write` 系统调用请求一个特定数目字节的传送, 但是驱动可自由传送较少数据 -- 对读和写这确切的规则稍微不同, 在本章后面描述。

不管这些方法传送多少数据, 它们通常应当更新 `*offp` 中的文件位置来表示在系统调用成功完成后当前的文件位置。内核接着在适当时候传播文件位置的改变到文件结构。`pread`和 `pwrite` 系统调用有不同的语义; 它们从一个给定的文件偏移操作, 并且不改变其他的系统调用看到的文件位置。这些调用传递一个指向用户提供的位置的指针, 并且放弃你的驱动所做的改变。

`read` 和 `write` 方法都在发生错误时返回一个负值。相反, 大于或等于 `0` 的返回值告知调用程序有多少字节已经成功传送。如果一些数据成功传送接着发生错误, 返回值必须是成功传送的字节数, 错误不报告直到函数下一次调用。 实现这个传统, 当然, 要求你的驱动记住错误已经发生, 以便它们可以在以后返回错误状态。

尽管内核函数返回一个负数指示一个错误, 这个数的值指出所发生的错误类型( 如第 2 章介绍 ), 用户空间运行的程序常常看到 `-1` 作为错误返回值。它们需要存取 `errno` 变量来找出发生了什么。用户空间的行为由 POSIX 标准来规定, 但是这个标准没有规定内核内部如何操作。



### 3.7.1 read 方法

`read` 的返回值由调用的应用程序解释:

* 如果这个值等于传递给 `read` 系统调用的 `count` 参数, 请求的字节数已经被传送。这是最好的情况。
* 如果是正数, 但是小于 `count`, 只有部分数据被传送。这可能由于几个原因, 依赖于设备。常常, 应用程序重新试着读取。例如, 如果你使用 `fread` 函数来读取, 库函数重新发出系统调用直到请求的数据传送完成。
* 如果值为 `0`, 到达了文件末尾(没有读取数据)。
* 一个负值表示有一个错误。 这个值指出了什么错误, 根据 `<linux/errno.h>`。 出错的典型返回值包括 `-EINTR`( 被打断的系统调用) 或者 `-EFAULT`( 坏地址 )。

前面列表中漏掉的是这种情况"没有数据, 但是可能后来到达"。在这种情况下, `read` 系统调用应当阻塞。我们将在第 6 章涉及阻塞。

scull 代码利用了这些规则。特别地, 它利用了部分读规则。每个 `scull_read` 调用只处理单个数据量子, 不实现一个循环来收集所有的数据; 这使得代码更短更易读。如果读程序确实需要更多数据, 它重新调用。如果标准 `I/O` 库(例如, `fread`)用来读取设备, 应用程序甚至不会注意到数据传送的量子化。

如果当前读取位置大于设备大小, scull 的 `read` 方法返回 `0` 来表示没有可用的数据(换句话说, 我们在文件尾)。这个情况发生在如果进程 A 在读设备, 同时进程 B 打开它写,这样将设备截短为 `0`。进程 A 突然发现自己过了文件尾, 下一个读调用返回 `0`。

这是 `read` 的代码( 忽略对 `down_interruptible` 的调用并且现在为 `up`; 我们在下一章中讨论它们):

~~~c
/*
 * Data management: read and write
 */

ssize_t scull_read(struct file *filp, char __user *buf, size_t count,
                loff_t *f_pos)
{
	struct scull_dev *dev = filp->private_data; 
	struct scull_qset *dptr;	/* the first listitem */
	int quantum = dev->quantum, qset = dev->qset;
	int itemsize = quantum * qset; /* how many bytes in the listitem */
	int item, s_pos, q_pos, rest;
	ssize_t retval = 0;

	if (mutex_lock_interruptible(&dev->lock))
		return -ERESTARTSYS;
	if (*f_pos >= dev->size)
		goto out;
	if (*f_pos + count > dev->size)
		count = dev->size - *f_pos;

	/* find listitem, qset index, and offset in the quantum */
	item = (long)*f_pos / itemsize;
	rest = (long)*f_pos % itemsize;
	s_pos = rest / quantum; q_pos = rest % quantum;

	/* follow the list up to the right position (defined elsewhere) */
	dptr = scull_follow(dev, item);

	if (dptr == NULL || !dptr->data || ! dptr->data[s_pos])
		goto out; /* don't fill holes */

	/* read only up to the end of this quantum */
	if (count > quantum - q_pos)
		count = quantum - q_pos;

	if (copy_to_user(buf, dptr->data[s_pos] + q_pos, count)) {
		retval = -EFAULT;
		goto out;
	}
	*f_pos += count;
	retval = count;

  out:
	mutex_unlock(&dev->lock);
	return retval;
}
~~~



### 3.7.2 write 方法

`write` 可以传送少于要求的数据, 根据返回值的下列规则:

* 如果值等于 `count`, 要求的字节数已被传送。
* 如果正值, 但是小于 `count`, 只有部分数据被传送。程序最可能重试写入剩下的数据。
* 如果值为 `0`, 什么没有写。这个结果不是一个错误, 没有理由返回一个错误码。再一次, 标准库重试写调用。我们将在第 6 章查看这种情况的确切含义, 那里介绍了阻塞。
* 一个负值表示发生一个错误; 如同对于读, 有效的错误值是定义于`<linux/errno.h>`中。

不幸的是, 仍然可能有发出错误消息的不当行为程序, 它在进行了部分传送时终止。这是因为一些程序员习惯看写调用要么完全失败要么完全成功, 这实际上是大部分时间的情况,应当也被设备支持。scull 实现的这个限制可以修改, 但是我们不想使代码不必要地复杂。

`write` 的 scull 代码一次处理单个量子, 和 `read` 是类似的处理方法:

~~~c
ssize_t scull_write(struct file *filp, const char __user *buf, size_t count,
                loff_t *f_pos)
{
	struct scull_dev *dev = filp->private_data;
	struct scull_qset *dptr;
	int quantum = dev->quantum, qset = dev->qset;
	int itemsize = quantum * qset;
	int item, s_pos, q_pos, rest;
	ssize_t retval = -ENOMEM; /* value used in "goto out" statements */

	if (mutex_lock_interruptible(&dev->lock))
		return -ERESTARTSYS;

	/* find listitem, qset index and offset in the quantum */
	item = (long)*f_pos / itemsize;
	rest = (long)*f_pos % itemsize;
	s_pos = rest / quantum; q_pos = rest % quantum;

	/* follow the list up to the right position */
	dptr = scull_follow(dev, item);
	if (dptr == NULL)
		goto out;
	if (!dptr->data) {
		dptr->data = kmalloc(qset * sizeof(char *), GFP_KERNEL);
		if (!dptr->data)
			goto out;
		memset(dptr->data, 0, qset * sizeof(char *));
	}
	if (!dptr->data[s_pos]) {
		dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL);
		if (!dptr->data[s_pos])
			goto out;
	}
	/* write only up to the end of this quantum */
	if (count > quantum - q_pos)
		count = quantum - q_pos;

	if (copy_from_user(dptr->data[s_pos]+q_pos, buf, count)) {
		retval = -EFAULT;
		goto out;
	}
	*f_pos += count;
	retval = count;

        /* update the size */
	if (dev->size < *f_pos)
		dev->size = *f_pos;

  out:
	mutex_unlock(&dev->lock);
	return retval;
}

~~~



### 3.7.3 readv 和 writev

Unix 系统已经长时间支持名为 `readv` 和 `writev` 的 2 个系统调用。这些 `read` 和 `write` 的"矢量"版本使用一个结构数组, 每个包含一个缓存的指针和一个长度值。一个 `readv` 调用被期望来轮流读取指示的数量到每个缓存。相反, `writev` 要收集每个缓存的内容到一起并且作为单个写操作送出它们。

如果你的驱动不提供方法来处理矢量操作, `readv` 和 `writev` 由多次调用你的 `read` 和 `write` 方法来实现。在许多情况, 但是, 直接实现 readv 和 writev 能获得更大的效率。矢量操作的原型是:

~~~c
ssize_t (*readv) (struct file *filp, const struct iovec *iov, 
                  unsigned long count, loff_t *ppos);
ssize_t (*writev) (struct file *filp, const struct iovec *iov, 
                   unsigned long count, loff_t *ppos);
~~~

这里, `filp` 和 `ppos` 参数与 `read` 和 `write` 的相同. `iovec` 结构, 定义于`<linux/uio.h>`, 如同:

~~~c
struct iovec
{
	void __user *iov_base; 
    __kernel_size_t iov_len;
};
~~~

每个 `iovec` 描述了一块要传送的数据; 它开始于 `iov_base` (在用户空间)并且有 `iov_len`字节长。`count` 参数告诉有多少 `iovec` 结构。这些结构由应用程序创建, 但是内核在调用驱动之前拷贝它们到内核空间。

矢量操作的最简单实现是一个直接的循环, 只是传递出去每个 `iovec` 的地址和长度给驱动的 `read` 和 `write` 函数。然而, 有效的和正确的行为常常需要驱动更聪明。例如, 一个磁带驱动上的 `writev` 应当将全部 `iovec` 结构中的内容作为磁带上的单个记录。

很多驱动, 但是, 没有从自己实现这些方法中获益。因此, scull 省略它们。内核使用`read` 和 `write` 来模拟它们, 最终结果是相同的。



## 3.8 使用新设备

一旦你装备好刚刚描述的 4 个方法, 驱动可以编译并测试了; 它保留了你写给它的任何数据, 直到你用新数据覆盖它。这个设备表现如一个数据缓存器, 它的长度仅仅受限于可用的真实 RAM 的数量。你可试着使用 `cp`, `dd`, 以及 输入/输出重定向来测试这个驱动。

`free` 命令可用来看空闲内存的数量如何缩短和扩张的, 依据有多少数据写入 scull。

为对一次读写一个量子有更多信心, 你可增加一个 `printk` 在驱动的适当位置, 并且观察当应用程序读写大块数据中发生了什么。可选地, 使用 `strace` 工具来监视程序发出的系统调用以及它们的返回值。跟踪一个 `cp` 或者一个 `ls -l > /dev/scull0` 展示了量子化的读和写. 监视(以及调试)技术在第 4 章详细介绍。



## 3.9 快速参考

本章介绍了下面符号和头文件。`struct file_operations` 和 `struct file` 中的成员的列表这里不重复了。

~~~c
#include <linux/types.h>
dev_t
~~~

> dev_t 是用来在内核里代表设备号的类型。

---------------

~~~c
int MAJOR(dev_t dev);
int MINOR(dev_t dev);
~~~

> 从设备编号中抽取主次编号的宏。

------------------------

~~~c
dev_t MKDEV(unsigned int major, unsigned int minor);
~~~

> 从主次编号来建立 dev_t 数据项的宏定义。

------------------------

~~~c
#include <linux/fs.h>
~~~

> "文件系统"头文件是编写设备驱动需要的头文件。许多重要的函数和数据结构在此定义。

------------------------

~~~c
int register_chrdev_region(dev_t first, unsigned int count, char *name)
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, 
                        unsigned int count, char *name)
void unregister_chrdev_region(dev_t first, unsigned int count);
~~~

> 允许驱动分配和释放设备编号的范围的函数。register_chrdev_region 应当用在事先知道需要的主编号时; 对于动态分配, 使用 alloc_chrdev_region 代替。

------------------------

~~~c
int register_chrdev(unsigned int major, const char *name, 
                    struct file_operations *fops);
~~~

> 老的( 2.6 之前) 字符设备注册函数。它在 2.6 内核中被模拟, 但是不应当给新代码使用。如果主编号不是 0, 可以不变地用它; 否则一个动态编号被分配给这个设备。

------------------------

~~~c
int unregister_chrdev(unsigned int major, const char *name);
~~~

> 恢复一个由 register_chrdev 所作的注册的函数。major 和 name 字符串必须包含之前用来注册设备时同样的值。

------------------------

~~~c
struct file_operations;
struct file;
struct inode;
~~~

> 大部分设备驱动使用的 3 个重要数据结构。file_operations 结构持有一个字符驱动的方法; struct file 代表一个打开的文件, struct inode 代表磁盘上的一个文件。

------------------------

~~~c
#include <linux/cdev.h>
struct cdev *cdev_alloc(void);
void cdev_init(struct cdev *dev, struct file_operations *fops);
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
void cdev_del(struct cdev *dev);
~~~

> cdev 结构管理的函数, 它代表内核中的字符设备。

------------------------

~~~c
#include <linux/kernel.h>
container_of(pointer, type, field);
~~~

> 一个传统宏定义, 可用来获取一个结构指针, 从它里面包含的某个其他结构的指针。

------------------------

~~~c
#include <asm/uaccess.h>
~~~

> 这个包含文件声明内核代码使用的函数来移动数据到和从用户空间。

------------------------

~~~c
unsigned long copy_from_user (void *to, const void *from, unsigned long count);
unsigned long copy_to_user (void *to, const void *from, unsigned long count);
~~~

> 在用户空间和内核空间拷贝数据。

------------------------



## 3.10 my_scull_1 实验

下面是在学习完本章后，参考书中源码+改进实现的简单字符设备，不包含管道等用法，对文件的操作函数（`scull_fops` 中的lseek、read、write等）也是直接从源码中拷贝过来的

my_scull_1/main.c

~~~c
/* my_scull_1/main.c */
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/types.h>
#include <linux/fs.h>

#include <linux/slab.h>
#include <linux/fcntl.h>	/* O_ACCMODE */
#include <linux/seq_file.h>
#include <linux/cdev.h>

#include <linux/uaccess.h>	/* copy_*_user */

/*
 * Our parameters which can be set at load time.
 */
#define     SCULL_MAJOR     0
#define     SCULL_NR_DEVS   1
#define     SCULL_P_NR_DEVS 4
#define     SCULL_QUANTUM   4000
#define     SCULL_QSET      1000

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

int scull_major =   SCULL_MAJOR;
int scull_minor =   0;
int scull_nr_devs = SCULL_NR_DEVS;	/* number of bare scull devices */
int scull_quantum = SCULL_QUANTUM;
int scull_qset =    SCULL_QSET;
static struct class *scull_class;
/* 全局或模块级变量（若还没有） */
static dev_t scull_dev_number = 0;
static bool cdev_added = false;
static bool device_created = false;
static bool class_created = false;

module_param(scull_major, int, S_IRUGO);
module_param(scull_minor, int, S_IRUGO);
module_param(scull_nr_devs, int, S_IRUGO);
module_param(scull_quantum, int, S_IRUGO);
module_param(scull_qset, int, S_IRUGO);

struct scull_qset {
	void **data;
	struct scull_qset *next;
};

struct scull_dev {
	struct scull_qset *data;  /* Pointer to first quantum set */
	int quantum;              /* the current quantum size */
	int qset;                 /* the current array size */
	unsigned long size;       /* amount of data stored here */
	unsigned int access_key;  /* used by sculluid and scullpriv */
	struct mutex lock;     /* mutual exclusion semaphore     */
	struct cdev cdev;	  /* Char device structure		*/
};

/*
 * Empty out the scull device; must be called with the device
 * semaphore held.
 */
int scull_trim(struct scull_dev *dev)
{
	struct scull_qset *next, *dptr;
	int qset = dev->qset;   /* "dev" is not-null */
	int i;

	for (dptr = dev->data; dptr; dptr = next) { /* all the list items */
		if (dptr->data) {
			for (i = 0; i < qset; i++)
				kfree(dptr->data[i]);
			kfree(dptr->data);
			dptr->data = NULL;
		}
		next = dptr->next;
		kfree(dptr);
	}
	dev->size = 0;
	dev->quantum = scull_quantum;
	dev->qset = scull_qset;
	dev->data = NULL;
	return 0;
}

struct scull_dev *scull_device;	/* allocated in scull_init_module */

/*
 * Open and close
 */

int scull_open(struct inode *inode, struct file *filp)
{
	struct scull_dev *dev; /* device information */

	dev = container_of(inode->i_cdev, struct scull_dev, cdev);
	filp->private_data = dev; /* for other methods */

	/* now trim to 0 the length of the device if open was write-only */
	if ( (filp->f_flags & O_ACCMODE) == O_WRONLY) {
		if (mutex_lock_interruptible(&dev->lock))
			return -ERESTARTSYS;
		scull_trim(dev); /* ignore errors */
		mutex_unlock(&dev->lock);
	}
	return 0;          /* success */
}

int scull_release(struct inode *inode, struct file *filp)
{
	return 0;
}
/*
 * Follow the list
 */
struct scull_qset *scull_follow(struct scull_dev *dev, int n)
{
	struct scull_qset *qs = dev->data;

        /* Allocate first qset explicitly if need be */
	if (! qs) {
		qs = dev->data = kmalloc(sizeof(struct scull_qset), GFP_KERNEL);
		if (qs == NULL)
			return NULL;  /* Never mind */
		memset(qs, 0, sizeof(struct scull_qset));
	}

	/* Then follow the list */
	while (n--) {
		if (!qs->next) {
			qs->next = kmalloc(sizeof(struct scull_qset), GFP_KERNEL);
			if (qs->next == NULL)
				return NULL;  /* Never mind */
			memset(qs->next, 0, sizeof(struct scull_qset));
		}
		qs = qs->next;
		continue;
	}
	return qs;
}

/*
 * Data management: read and write
 */

ssize_t scull_read(struct file *filp, char __user *buf, size_t count,
                loff_t *f_pos)
{
	struct scull_dev *dev = filp->private_data; 
	struct scull_qset *dptr;	/* the first listitem */
	int quantum = dev->quantum, qset = dev->qset;
	int itemsize = quantum * qset; /* how many bytes in the listitem */
	int item, s_pos, q_pos, rest;
	ssize_t retval = 0;

	if (mutex_lock_interruptible(&dev->lock))
		return -ERESTARTSYS;
	if (*f_pos >= dev->size)
		goto out;
	if (*f_pos + count > dev->size)
		count = dev->size - *f_pos;

	/* find listitem, qset index, and offset in the quantum */
	item = (long)*f_pos / itemsize;
	rest = (long)*f_pos % itemsize;
	s_pos = rest / quantum; q_pos = rest % quantum;

	/* follow the list up to the right position (defined elsewhere) */
	dptr = scull_follow(dev, item);

	if (dptr == NULL || !dptr->data || ! dptr->data[s_pos])
		goto out; /* don't fill holes */

	/* read only up to the end of this quantum */
	if (count > quantum - q_pos)
		count = quantum - q_pos;

	if (copy_to_user(buf, dptr->data[s_pos] + q_pos, count)) {
		retval = -EFAULT;
		goto out;
	}
	*f_pos += count;
	retval = count;

  out:
	mutex_unlock(&dev->lock);
	return retval;
}

ssize_t scull_write(struct file *filp, const char __user *buf, size_t count,
                loff_t *f_pos)
{
	struct scull_dev *dev = filp->private_data;
	struct scull_qset *dptr;
	int quantum = dev->quantum, qset = dev->qset;
	int itemsize = quantum * qset;
	int item, s_pos, q_pos, rest;
	ssize_t retval = -ENOMEM; /* value used in "goto out" statements */

	if (mutex_lock_interruptible(&dev->lock))
		return -ERESTARTSYS;

	/* find listitem, qset index and offset in the quantum */
	item = (long)*f_pos / itemsize;
	rest = (long)*f_pos % itemsize;
	s_pos = rest / quantum; q_pos = rest % quantum;

	/* follow the list up to the right position */
	dptr = scull_follow(dev, item);
	if (dptr == NULL)
		goto out;
	if (!dptr->data) {
		dptr->data = kmalloc(qset * sizeof(char *), GFP_KERNEL);
		if (!dptr->data)
			goto out;
		memset(dptr->data, 0, qset * sizeof(char *));
	}
	if (!dptr->data[s_pos]) {
		dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL);
		if (!dptr->data[s_pos])
			goto out;
	}
	/* write only up to the end of this quantum */
	if (count > quantum - q_pos)
		count = quantum - q_pos;

	if (copy_from_user(dptr->data[s_pos]+q_pos, buf, count)) {
		retval = -EFAULT;
		goto out;
	}
	*f_pos += count;
	retval = count;

        /* update the size */
	if (dev->size < *f_pos)
		dev->size = *f_pos;

  out:
	mutex_unlock(&dev->lock);
	return retval;
}

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
    {
        //err = !access_ok_wrapper(VERIFY_WRITE, (void __user *)arg, _IOC_SIZE(cmd));
        err = !access_ok((void __user *)arg, _IOC_SIZE(cmd));
    }
	else if (_IOC_DIR(cmd) & _IOC_WRITE)
    {
        //err =  !access_ok_wrapper(VERIFY_READ, (void __user *)arg, _IOC_SIZE(cmd));
        err = !access_ok((void __user *)arg, _IOC_SIZE(cmd));
    }
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
#if 0
	  case SCULL_P_IOCTSIZE:
		scull_p_buffer = arg;
		break;

	  case SCULL_P_IOCQSIZE:
		return scull_p_buffer;
#endif

	  default:  /* redundant, as cmd was checked against MAXNR */
		return -ENOTTY;
	}
	return retval;

}


/*
 * The "extended" operations -- only seek
 */

loff_t scull_llseek(struct file *filp, loff_t off, int whence)
{
	struct scull_dev *dev = filp->private_data;
	loff_t newpos;

	switch(whence) {
	  case 0: /* SEEK_SET */
		newpos = off;
		break;

	  case 1: /* SEEK_CUR */
		newpos = filp->f_pos + off;
		break;

	  case 2: /* SEEK_END */
		newpos = dev->size + off;
		break;

	  default: /* can't happen */
		return -EINVAL;
	}
	if (newpos < 0) return -EINVAL;
	filp->f_pos = newpos;
	return newpos;
}
struct file_operations scull_fops = {
	.owner =    THIS_MODULE,
	.llseek =   scull_llseek,
	.read =     scull_read,
	.write =    scull_write,
	.unlocked_ioctl = scull_ioctl,
	.open =     scull_open,
	.release =  scull_release,
};


static void my_scull_1_cleanup_all(dev_t dev)
{
    /* 注意：顺序很重要 —— 先移除 device，再销毁 class，再删除 cdev/free，再注销 devnum */
    if (device_created && scull_class) {
        device_destroy(scull_class, dev);
        device_created = false;
    }

    if (class_created && scull_class) {
        class_destroy(scull_class);
        class_created = false;
    }

    if (cdev_added && scull_device) {
        cdev_del(&scull_device->cdev);
        cdev_added = false;
    }

    if (scull_device) {
        scull_trim(scull_device); /* 如果需要 */
        kfree(scull_device);
        scull_device = NULL;
    }

    if (scull_dev_number) {
        unregister_chrdev_region(scull_dev_number, scull_nr_devs);
        scull_dev_number = 0;
    }
}
static char *my_scull_devnode(struct device *dev, umode_t *mode)
{
    if (mode)
        *mode = 0666;   // 所有用户可读写
    return NULL;
}

// 模块初始化函数
static int __init my_scull_1_init(void) {
    int result;
	dev_t dev = 0;
    printk(KERN_INFO "[my_scull_1]: Module loaded.\n");
      
    /* 1. Register char device NUMMBER */
    if (scull_major) {
        /* If scull_major isn't 0. Get dev by scull_major and scull_minor */
		dev = MKDEV(scull_major, scull_minor);
        /* Register char device by dev  */
		result = register_chrdev_region(dev, scull_nr_devs, "my_scull_1");
        scull_dev_number = dev;
	} else {
        /* If scull_major is 0. Automatic registration char device */
		result = alloc_chrdev_region(&dev, scull_minor, scull_nr_devs,
				"my_scull_1");
        if (result == 0) {
            /* Get scull_major by dev */
            scull_major = MAJOR(dev);
            scull_dev_number = dev;
        }
	}

    if (result < 0) {
		printk(KERN_WARNING "[my_scull_1]: can't get major %d\n", scull_major);
		return result;
	}
    /* 2. Create class and dev/my_scull0 */
    scull_class = class_create(THIS_MODULE, "scull_class");
    if (IS_ERR(scull_class)) {
        result = PTR_ERR(scull_class);
        goto fail;
    }
    class_created = true;
    scull_class->devnode = my_scull_devnode;

    device_create(scull_class, NULL, dev , NULL, "my_scull0");
    printk(KERN_WARNING "[my_scull_1]: major = %d\n", scull_major);
    device_created = true;
    /* 3. Register char device. Apply for space, registration method */
    scull_device = kmalloc(sizeof(struct scull_dev), GFP_KERNEL);
    if (!scull_device) {
		result = -ENOMEM;
		goto fail;  /* Make this more graceful */
	}
	memset(scull_device, 0, sizeof(struct scull_dev));
    // init cdev for kernel
    scull_device->quantum = scull_quantum;
	scull_device->qset = scull_qset;
	mutex_init(&scull_device->lock);
    cdev_init(&scull_device->cdev, &scull_fops);
    scull_device->cdev.owner = THIS_MODULE;
    result = cdev_add (&scull_device->cdev, dev, 1);
    if (result)
    {
        printk(KERN_NOTICE "Error %d adding myscull0", result);
        goto fail;  /* Make this more graceful */
    }
	cdev_added = true;
    return 0;
  fail:
	my_scull_1_cleanup_all(dev);
	return result;

}

// 模块卸载函数
static void __exit my_scull_1_exit(void) {
    dev_t dev = MKDEV(scull_major, scull_minor);
    printk(KERN_INFO "[my_scull_1]: Module unloaded.\n");
    my_scull_1_cleanup_all(dev);
}

module_init(my_scull_1_init);
module_exit(my_scull_1_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GG GG");
MODULE_DESCRIPTION("my_scull_1");
MODULE_VERSION("0.1");
~~~

最后的实现效果如下：

~~~(空)
w32336@h3cw32336:~$ sudo insmod my_scull_1.ko
w32336@h3cw32336:~$ ls -l my_scull_1.ko
-rwx--x--x 1 w32336 w32336 20352 12月  7 18:15 my_scull_1.ko
w32336@h3cw32336:~$ dmesg | tail
[  449.454990] [my_scull_1]: Module loaded.
[  449.455164] [my_scull_1]: major = 240
w32336@h3cw32336:~$
w32336@h3cw32336:~$ ls -l /sys/class/scull_class/
total 0
lrwxrwxrwx 1 root root 0 12月  7 18:19 my_scull0 -> ../../devices/virtual/scull_class/my_scull0
w32336@h3cw32336:~$ ls -l /dev/my_scull0
crw-rw-rw- 1 root root 240, 0 12月  7 18:18 /dev/my_scull0
w32336@h3cw32336:~$ echo "my_scull0 test 1" > /dev/my_scull0
w32336@h3cw32336:~$ cat /dev/my_scull0
my_scull0 test 1
w32336@h3cw32336:~$ cp /dev/my_scull0 ./log.txt
w32336@h3cw32336:~$ cat log.txt
my_scull0 test 1
w32336@h3cw32336:~$ sudo rmmod my_scull_1
w32336@h3cw32336:~$ dmesg | tail
[  449.454990] [my_scull_1]: Module loaded.
[  449.455164] [my_scull_1]: major = 240
[  601.804461] [my_scull_1]: Module unloaded.
w32336@h3cw32336:~$
w32336@h3cw32336:~$ ls -l /sys/class/scull_class/
ls: cannot access '/sys/class/scull_class/': No such file or directory
w32336@h3cw32336:~$ ls -l /dev/my_scull0
ls: cannot access '/dev/my_scull0': No such file or directory
w32336@h3cw32336:~$
~~~

本次实验就为简单实验 字符驱动 的使用，后续会继续扩展优化。



# END























