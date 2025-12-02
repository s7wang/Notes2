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

































