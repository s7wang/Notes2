# Linux 设备驱动

# 第四章 调试技术

内核编程带有它自己的, 独特的调试挑战性。内核代码无法轻易地在一个调试器下运行, 也无法轻易的被跟踪, 因为它是一套没有与特定进程相关连的功能的集合。内核代码错误也特别难以重现, 它们会牵连整个系统与它们一起失效, 从而破坏了大量的能用来追踪错误的证据。

本章介绍了在如此艰难情况下能够用以监视内核代码和跟踪错误的技术。



## 4.1 内核中的调试支持

在第 2 章, 我们建议你建立并安装你自己的内核, 而不是运行来自你的发布商的现成的内核。运行你自己的内核的最充分的理由之一是内核开发者已经在内核自身中构建了多个调试特性。这些特性能产生额外的输出并降低性能, 因此发布商的产品内核中往往不会使能它们。作为一个内核开发者, 但是, 你有不同的优先权并且会乐于接收这些格外的内核调试支持带来的开销。

这里, 我们列出用来开发的内核应当激活的配置选项。 除了另外指出的, 所有的这些选项都在 "`kernel hacking`" 菜单, 不管什么样的你喜欢的内核配置工具。注意有些选项不是所有体系都支持。

### 4.1.1 内核相关调试宏

~~~c
CONFIG_DEBUG_KERNEL
~~~

> 这个选项只是使其他调试选项可用; 它应当打开, 但是它自己不激活任何的特性。

~~~c
CONFIG_DEBUG_SLAB
~~~

> 这个重要的选项打开了内核内存分配函数的几类检查; 激活这些检查, 就可能探测到一些内存覆盖和遗漏初始化的错误。被分配的每一个字节在递交给调用者之前都设成 0xa5, 随后在释放时被设成 0x6b. 你在任何时候如果见到任一个这种"坏"模式重复出现在你的驱动输出(或者常常在一个 oops 的列表), 你会确切知道去找什
> 么类型的错误。当激活调试, 内核还会在每个分配的内存对象的前后放置特别的守护值; 如果这些值曾被改动, 内核知道有人已覆盖了一个内存分配区, 它大声抱怨.各种的对更模糊的问题的检查也给激活了。
>
> 是 **Linux 内核中用于调试 SLAB 分配器的一个配置选项**，主要用于 **发现内存越界、非法释放、双重释放等 slab 相关的内存错误**。它只用于 **调试内核问题**，**不建议在生产环境开启**。



~~~c
CONFIG_DEBUG_PAGEALLOC
~~~

> 满的页在释放时被从内核地址空间去除. 这个选项会显著拖慢系统, 但是它也能快速指出某些类型的内存损坏错误。
>
> CONFIG_DEBUG_PAGEALLOC 是“页级别的内存断电保护”，一旦踩线立刻翻车。
>
> | Bug 类型               | 是否容易抓到 |
> | ---------------------- | ------------ |
> | use-after-free（页级） | ⭐⭐⭐⭐⭐        |
> | 跨页越界访问           | ⭐⭐⭐⭐⭐        |
> | 非法访问释放页         | ⭐⭐⭐⭐⭐        |
>
> 代价很大只适合短时间调试，**不建议在生产环境开启**。

------------------------

~~~c
CONFIG_DEBUG_SPINLOCK
~~~

> `CONFIG_DEBUG_SPINLOCK` 是 **Linux 内核里用于调试自旋锁（spinlock）使用错误的配置选项**，核心目的只有一个：尽早发现“锁用错了”的 bug，并给出明确的调用栈。
>
> 开启后，内核在 `spin_lock()` / `spin_unlock()` 等路径上会做**额外合法性检查**。
>
> 1. 检查是否解了一个“没加过的锁”
>
> ```
> spin_unlock(&lock);   // 从未 lock
> ```
>
> 立刻 `BUG` / `WARN`
>
> 2. 检查锁状态是否被破坏
>
>    - lock 结构体被内存踩坏
>
>    - lock 值非法
>
> 直接报错，而不是“随机死机”
>
> 3. 检测锁递归使用（在非 raw spinlock 场景）
>
> ```
> spin_lock(&lock);
> spin_lock(&lock);   // 同 CPU 递归
> ```
>
> 可能触发调试告警
>
> 4. 检查 unlock 的 CPU 是否是 lock 的 CPU
>
>    - 锁在 CPU0 获取
>
>    - 却在 CPU1 释放
>
> 这是 **严重 bug**，会被检查出来
>
> 5. 检测中断上下文违规，结合其他调试项时（如 `CONFIG_DEBUG_LOCK_ALLOC`）：
>
>    - 在 **中断上下文** 用了错误的 lock API
>
>    - `spin_lock()` / `spin_lock_irqsave()` 用错
>
> 比 `DEBUG_PAGEALLOC` **轻得多**，但仍不适合生产环境。

------------------------

~~~c
CONFIG_DEBUG_SPINLOCK_SLEEP
~~~

> `CONFIG_DEBUG_SPINLOCK_SLEEP` 是 **Linux 内核中用于调试“自旋锁 + 睡眠”违规的配置选项**，核心目的只有一个：防止在不可睡眠（atomic）上下文中调用可能睡眠的函数。拿着 spinlock、关中断、在中断上下文时 —— 绝对不能 sleep。**少量检查，几乎不印象性能**。

------------------------

~~~c
CONFIG_INIT_DEBUG
~~~

> `CONFIG_INIT_DEBUG` 是 **Linux 内核中用于调试“初始化阶段（initcall）”的配置选项**，它的作用是：
>
> 把内核和驱动在启动阶段每一个 initcall 的执行过程完整打印出来，用来定位“开机卡住 / 启动慢 / 驱动初始化失败”等问题。把内核启动时“谁在什么时候初始化”的时间线全部摊开给你看。
>
> 1. **开启后，内核会对 所有 initcall 做额外日志输出，包括**：
>
> 调用前打印：
>
> ```
> calling  my_driver_init+0x0/0x100 @ 1
> ```
>
> 调用后打印：
>
> ```
> initcall my_driver_init+0x0/0x100 returned 0 after 3 usecs
> ```
>
> 2. **确认内核启动卡住的位置**
>
> 屏幕停在：
>
> ```
> Starting kernel ...
> ```
>
> 或某个驱动后不动了，看最后一个打印的 initcall 就知道 **卡在哪个驱动**。
>
> 3. **确认启动慢的进程**
>
> 示例输出：
>
> ```
> initcall usb_init+0x0/0x300 returned 0 after 253456 usecs
> ```
>
> 一眼就知道 **USB 初始化慢**。
>
> 3. **驱动 init 返回错误**
>
> ~~~(空)
> initcall my_driver_init+0x0/0x80 returned -16
> ~~~
>
>
>  -EBUSY，驱动没起来

------------------------

~~~c
CONFIG_DEBUG_INFO
~~~

> 这个选项使得内核在建立时包含完整的调试信息. 如果你想使用 gdb 调试内核。你将需要这些信息. 如果你打算使用 gdb, 你还要激活 CONFIG_FRAME_POINTER。
>
> `CONFIG_DEBUG_INFO` 是 **Linux 内核里用于生成调试符号（DWARF）的配置选项**，它的核心作用是：让内核和模块“可被调试、可被还原源码行号和变量名”。开启后，内核在编译时会：
>
> - 给 **vmlinux** 和 **.ko 模块**加入 **DWARF 调试信息**
> - 保留：函数名、源文件名、行号、局部变量 / 结构体信息
>
> 不开它，你看到的调试信息会非常有限。
>
> | 影响         | 说明                 |
> | ------------ | -------------------- |
> | vmlinux 体积 | **明显变大**（几倍） |
> | .ko 大小     | 增大                 |
> | 运行性能     | ❌ 无影响             |
> | RAM          | ❌ 无影响             |
>
> 只影响 **磁盘 / 镜像大小**。

------------------------

~~~c
CONFIG_MAGIC_SYSRQ
~~~

> `CONFIG_MAGIC_SYSRQ` 是 **Linux 内核中用于启用“Magic SysRq 键”功能的配置选项**，它的作用可以一句话概括为：在系统已经“基本挂死”的情况下，仍然能直接向内核发指令做紧急操作。
>
> **允许内核注册并处理 SysRq 指令**即使：
>
> - 用户态挂死
>
> - 调度异常
>
> - shell 无响应
>
> 只要内核还在跑，SysRq 依然有效。
>
> 经典安全重启顺序（**REISUB**）：
>
> ```
> Alt + SysRq + r e i s u b
> ```
>
> 做 **BSP / 驱动 / OpenWrt / 内核调试**：
>
> *  **开发内核：强烈建议开启**
> * **量产系统：开启但运行时限制**

------------------------

~~~c
CONFIG_DEBUG_STACKOVERFLOW
CONFIG_DEBUG_STACK_USAGE
~~~

> 这些选项能帮助跟踪内核堆栈溢出。堆栈溢出的确证是一个 oops 输出, 但是没有任何形式的合理的回溯。第一个选项给内核增加了明确的溢出检查; 第 2 个使得内核监测堆栈使用并作一些统计, 这些统计可以用魔术 SysRq 键得到。
>
> | 选项                           | 解决什么问题 | 何时触发          |
> | ------------------------------ | ------------ | ----------------- |
> | **CONFIG_DEBUG_STACKOVERFLOW** | **栈溢出**   | 栈已经快/已经越界 |
> | **CONFIG_DEBUG_STACK_USAGE**   | **栈用太多** | 栈使用量接近上限  |
>
> 一个是 **“已经出事”**，一个是 **“快要出事”**。
>
> `CONFIG_DEBUG_STACKOVERFLOW`是检测内核栈是否发生越界（overflow）。
>
> 内核栈很小（通常）：
>
> - x86_64：**16KB**
> - ARM / ARM64：**8KB 或 16KB**
> - 中断栈 / IRQ 栈更小
>
> 一旦溢出：会覆盖 task_struct、随机 crash、极难定位
>
> - 在内核栈 **边界附近插入检查**
> - 在：系统调用、中断、异常、上下文切换 时检测栈指针是否合法
>
> 发现异常直接：
>
> ```
> BUG: stack overflow detected
> ```
>
> 并给出调用栈。**调试内核 / 驱动强烈建议开。**
>
> `CONFIG_DEBUG_STACK_USAGE`：**统计每个 task 实际使用过的最大内核栈深度。**它不是抓 crash，而是 **给你“水位线”**。
>
> * 任务创建时，把整个内核栈填充为固定模式（如 `0xAA`）
> * 运行过程中：栈被用掉的部分会被覆盖
> * 在特定时机统计：最深使用到哪里
>
> 通过 `procfs`：
>
> ```
> cat /proc/<pid>/stack
> ```
>
> 或（依架构）：
>
> ```
> cat /proc/<pid>/status | grep Stack
> ```
>
> 或者在调试日志中看到类似：
>
> ```
> stack usage: 7328 bytes
> ```
>
> **调试内核 / 驱动强烈建议开。**
>
> ### 推荐组合（开发阶段）
>
> ```
> CONFIG_DEBUG_STACKOVERFLOW=y
> CONFIG_DEBUG_STACK_USAGE=y
> ```

------------------------

~~~c
CONFIG_KALLSYMS
~~~

> `CONFIG_KALLSYMS` 是 **Linux 内核中用于保留和导出内核符号表的配置选项**，它的核心作用是：**让内核地址能被解析成人类可读的“函数名 + 偏移”。**一句话总结**CONFIG_KALLSYMS = 给内核装一份“地址 ↔ 符号名”的对照表。**
>
> 1. **Oops / Panic 调用栈解析**
>
> `dmesg`串口日志**没有它，栈基本不可读**。
>
> 2. /proc/kallsyms
>
> ~~~(空)
> cat /proc/kallsyms | head
> ~~~
>
> 内核符号地址表；perf / ftrace / crash 都会用到。
>
> 3. **perf / ftrace / bpf 符号化**：栈回溯；函数级 profiling
>
> 4. **驱动开发定位问题**：“死在哪个函数”，“偏移多少”。
>
> 5. **相关子选项CONFIG_KALLSYMS_ALL**：默认：只导出 **非 static 的符号**，开启后：**static 函数也进符号表**调试内核/驱动时非常有用。
>
> | 选项              | 作用                         |
> | ----------------- | ---------------------------- |
> | CONFIG_KALLSYMS   | **地址 → 函数名**            |
> | CONFIG_DEBUG_INFO | **函数名 → 源码行号 / 变量** |
>
> | 影响     | 说明                  |
> | -------- | --------------------- |
> | 内核体积 | 增加（几十～几百 KB） |
> | 运行性能 | ❌ 无影响              |
> | 内存     | ❌ 无影响              |
>
> 性价比极高，安全性注意（生产系统）
>
> - `/proc/kallsyms` 可能泄露符号信息
>
> - 通常结合：
>
>   ```
>   kernel.kptr_restrict = 1
>   ```
>
> 限制普通用户读取地址。

------------------------

~~~c
CONFIG_IKCONFIG
CONFIG_IKCONFIG_PROC
~~~

> 1. `CONFIG_IKCONFIG` 的作用是：
>
> - 在 **编译内核时**
> - 把当前使用的 **`.config` 文件**
> - **原样嵌入到内核镜像里（压缩保存）**
>
> 这一步 **只负责“存进去”**，不负责“给你看”。
>
> 2. `CONFIG_IKCONFIG_PROC` 的作用是：
>
> - 在运行时
>
> - 提供一个接口：
>
>   ```
>   /proc/config.gz
>   ```
>
> - 让你可以 **直接查看编译内核时的配置**
>
> 它依赖 `CONFIG_IKCONFIG`，**单独开没意义**。

------------------------

~~~c
CONFIG_ACPI_DEBUG
~~~

> `CONFIG_ACPI_DEBUG` 是 **Linux 内核中用于开启 ACPI 子系统调试能力的编译选项**，主要面向 **ACPI 代码、固件（BIOS/UEFI）以及电源/设备枚举问题的调试**。
>
> 开启后，内核会：
>
> - **编译进 ACPI 调试代码**
> - 允许在运行时：打开 ACPI Debug 日志；按模块、级别精细控制输出。
>
> **仅开启该选项 ≠ 一直刷日志** ，是否打印，由 **运行时参数** 决定。
>
>  推荐策略：
>
> - **编译期开 CONFIG_ACPI_DEBUG**
> - **运行时默认关闭**
> - 需要时再打开

------------------------

~~~c
CONFIG_DEBUG_DRIVER
~~~

> `CONFIG_DEBUG_DRIVER` 是 **Linux 内核中用于开启“驱动模型（Driver Model）调试信息”的配置选项**，它关注的不是某个具体驱动，而是 **设备 / 总线 / 驱动三者之间的绑定关系和生命周期**。`CONFIG_DEBUG_DRIVER `= 让内核把“设备和驱动是怎么绑定、怎么解绑的”都打印出来。
>
> 开启后，内核会在 **driver core（drivers/base/）** 的关键路径打印调试信息，例如：
>
> - 设备注册 / 注销
> - 驱动注册 / 注销
> - probe / remove
> - bind / unbind
> - match 失败原因
>
> 这些都是平时 **“设备为什么没被驱动到”** 的核心线索。
>
> | 影响 | 说明           |
> | ---- | -------------- |
> | 日志 | 启动时明显增多 |
> | 性能 | 轻微           |
> | 内存 | 几乎无         |
>
> 它只打印 **driver core 层**
>
> 要看驱动内部：`pr_debug`和`dynamic_debug`。

------------------------

~~~c
CONFIG_SCSI_CONSTANTS
~~~

> `CONFIG_SCSI_CONSTANTS` 是 **Linux 内核中 SCSI 子系统的一个调试/可读性选项**，主要作用是 **把原本的“数字 SCSI 常量”翻译成可读的字符串**，方便排错和看日志。`CONFIG_SCSI_CONSTANTS `= 把 SCSI 日志里的“数字代码”，换成“人能看懂的名字”。
>
> CONFIG_SCSI_CONSTANTS 实际做了：
>
> 在内核中：保留一组 **SCSI 错误码 → 字符串** 的映射表；
>
> 当打印 SCSI 错误时：同时输出 **名字 + 原始数值**；
>
> 不影响 SCSI 协议行为，不影响 I/O 路径逻辑。

------------------------

~~~c
CONFIG_INPUT_EVBUG
~~~

> `CONFIG_INPUT_EVBUG` 是 **Linux 内核 input 子系统的一个调试选项**，用于 **把所有输入事件（EV_\*）原样打印到内核日志**，主要面向 **输入驱动/设备调试**。`CONFIG_INPUT_EVBUG `= 把“键盘 / 触摸 / 按键 / 编码器”的原始输入事件全部打到 dmesg。
>
> CONFIG_INPUT_EVBUG开启后，内核会：
>
> - 编译一个调试用 input handler：`evbug`
> - 这个 handler 会：
>   - 自动 attach 到 **所有 input 设备**
>   - 在事件产生时打印：
>
> ```
> EV_KEY KEY_POWER 1
> EV_SYN SYN_REPORT 0
> ```
>
> 属于 **input 层“旁观者”调试器**
>
> ✔ **只在调试内核时开启** ✔ 问题定位完立刻关闭 ❌ 不要用于量产 / 长期运行

------------------------

~~~c
CONFIG_PROFILING
~~~

> `CONFIG_PROFILING` 是 **Linux 内核里用于启用“内核性能分析（profiling）基础设施”的总开关**，它本身不直接产生性能数据，但**为各种 profiler 提供必要的钩子和接口**。`CONFIG_PROFILING `= 给 perf / ftrace / oprofile 等性能工具“通电”的总电源。
>
> `CONFIG_PROFILING`开启后，内核会：
>
> - 启用 **profiling framework**
> - 允许内核在关键路径上：
>   - 记录执行点
>   - 采样 PC（指令指针）
> - 向用户态工具（如 `perf`）暴露接口
>
> ⚠️ **不开这个选项，很多性能工具直接不可用或功能受限。**
>
> ✔ **建议开启**（即使量产内核） ✔ 不用时几乎零成本 ✔ 需要时救命
>
> `CONFIG_PROFILING` 不是“分析工具”，而是“让分析工具能存在的基础设施”。

------------------------



## 4.2 用打印调试

最常用的调试技术是监视, 在应用程序编程当中是通过在合适的地方调用 printf 来实现。在你调试内核代码时, 你可以通过 printk 来达到这个目的。

### 4.2.1 printk

`printk`的使用方法几乎与`printf`一致，一个不同是 `printk` 允许你根据消息的严重程度对其分类, 通过附加不同的记录级别或者优先级在消息上。你常常用一个宏定义来指示记录级别。例如, `KERN_INFO`, 我们之前曾在一些打印语句的前面看到过, 是消息记录级别的一种可能值。记录宏定义扩展成一个字串, 在编译时与消息文本连接在一起; 这就是为什么下面的在优先级和格式串之间没有逗号的原因。这里有 2 个 `printk` 命令的例子, 一个调试消息, 一个紧急消息:

~~~c
printk(KERN_DEBUG "Here I am: %s:%i\n", __FILE__, __LINE__);
printk(KERN_CRIT "I'm trashed; giving up on %p\n", ptr);
~~~

有 8 种可能的记录字串, 在头文件 `<linux/kernel.h>` 里定义; 我们按照严重性递减的顺序列出它们:

~~~c
KERN_EMERG		//用于紧急消息, 常常是那些崩溃前的消息.
KERN_ALERT		//需要立刻动作的情形.
KERN_CRIT		//严重情况, 常常与严重的硬件或者软件失效有关.
KERN_ERR		//用来报告错误情况; 设备驱动常常使用 KERN_ERR 来报告硬件故障. 
KERN_WARNING	//有问题的情况的警告, 这些情况自己不会引起系统的严重问题.
KERN_NOTICE		//正常情况, 但是仍然值得注意. 在这个级别一些安全相关的情况会报告.
KERN_INFO		//信息型消息. 在这个级别, 很多驱动在启动时打印它们发现的硬件的信息.
KERN_DEBUG		//用作调试消息.
~~~

每个字串( 在宏定义扩展里 )代表一个在角括号中的整数。整数的范围从 0 到 7, 越小的数表示越大的优先级。

一条没有指定优先级的 printk 语句缺省是 `DEFAULT_MESSAGE_LOGLEVEL`, 在`kernel/printk.c` 里指定作为一个整数。在 2.6.10 内核中, `DEFAULT_MESSAGE_LOGLEVEL`是 `KERN_WARNING`, 但是在过去已知是改变的。

基于记录级别, 内核可能打印消息到当前控制台, 可能是一个文本模式终端, 串口, 或者是一台并口打印机。如果优先级小于整型值 `console_loglevel`, 消息被递交给控制台,一次一行( 除非提供一个新行结尾, 否则什么都不发送 。如果 `klogd` 和 `syslogd` 都在系统中运行, 内核消息被追加到 `/var/log/messages` (或者另外根据你的 `syslogd `配置处理), 独立于 `console_loglevel`。如果 `klogd` 没有运行, 你只有读 `/proc/kmsg` ( 用`dmsg` 命令最易做到 )将消息取到用户空间。当使用 `klogd` 时, 你应当记住, 它不会保存连续的同样的行; 它只保留第一个这样的行, 随后是, 它收到的重复行数。

变量 `console_loglevel` 初始化成 `DEFAULT_CONSOLE_LOGLEVEL`, 并且可通过 `sys_syslog`系统调用修改。一种修改它的方法是在调用 klogd 时指定 `-c` 开关, 在 `klogd` 的`manpage` 里有指定。注意要改变当前值, 你必须先杀掉 `klogd`, 接着使用 `-c` 选项重启它。另外, 你可写一个程序来改变控制台记录级别。你会发现这样一个程序的版本在由 `O'Reilly` 提供的 FTP 站点上的 `miscprogs/setlevel.c`。新的级别指定未一个整数, 在 1和 8 之前, 包含 1 和 8。如果它设为 1, 只有 0 级消息( `KERN_EMERG` )到达控制台;如果它设为 8, 所有消息, 包括调试消息,都显示。

也可以通过文本文件 `/proc/sys/kernel/printk` 读写控制台记录级别。这个文件有 4 个整型值: 当前记录级别, 适用没有明确记录级别的消息的缺省级别, 允许的最小记录级别,以及启动时缺省记录级别。写一个单个值到这个文件就改变当前记录级别成这个值; 因此,例如, 你可以使所有内核消息出现在控制台, 通过简单地输入:

~~~(空)
# echo 8 > /proc/sys/kernel/printk
~~~

现在应当清楚了为什么 `hello.c` 例子使用 `KERN_ALERT` 标志; 它们是要确保消息会出现在控制台上.



### 4.2.2 重定向控制台消息

Linux 在控制台记录策略上允许一些灵活性, 它允许你发送消息到一个指定的虚拟控制台(如果你的控制台使用的是文本屏幕)。缺省地, 这个"控制台"是当前虚拟终端。为了选择一个不同地虚拟终端来接收消息, 你可对任何控制台设备调用 ioctl(TIOCLINUX)。下面的程序, setconsole, 可以用来选择哪个控制台接收内核消息; 它必须由超级用户运行,可以从 misc-progs 目录得到.下面是全部程序。 应当使用一个参数来指定用以接收消息的控制台的编号。

~~~c
int main(int argc, char **argv)
{
    char bytes[2] = {11,0}; /* 11 is the TIOCLINUX cmd number */
    if (argc==2) 
        bytes[1] = atoi(argv[1]); /* the chosen console */
	else {
		fprintf(stderr, "%s: need a single arg\n",argv[0]); 
        exit(1); 
    } 
    
    if (ioctl(STDIN_FILENO, TIOCLINUX, bytes)<0) { /* use stdin */
		fprintf(stderr,"%s: ioctl(stdin, TIOCLINUX): %s\n",argv[0], strerror(errno));
		exit(1);
	}
	exit(0);
}
~~~

setconsole 使用特殊的 `ioctl` 命令 `TIOCLINUX`, 来实现特定于 linux 的功能。为使用`TIOCLINUX`, 你传递它一个指向字节数组的指针作为参数。数组的第一个字节是一个数,指定需要的子命令, 下面的字节是特对于子命令的。在 `setconsole` 里, 使用子命令 `11`,下一个字节(存于 `bytes[1]`)指定虚拟控制台。`TIOCLINUX` 的完整描述在内核源码的`drivers/char/tty_io.c` 里。



### 4.2.3 消息是如何记录的

`printk` 函数将消息写入一个` __LOG_BUF_LEN` 字节长的环形缓存, 长度值从 `4 KB` 到 `1MB`, 由配置内核时选择. 这个函数接着唤醒任何在等待消息的进程, 就是说, 任何在系统调用中睡眠或者在读取 `/proc/kmsg` 的进程。这 2 个日志引擎的接口几乎是等同的, 但是注意, 从 `/proc/kmsg` 中读取是从日志缓存中消费数据, 然而 `syslog` 系统调用能够选择地在返回日志数据地同时保留它给其他进程。通常, 读取 `/proc` 文件容易些并且是 `klogd` 的缺省做法。`dmesg` 命令可用来查看缓存的内容, 不会冲掉它; 实际上, 这个命令将缓存区的整个内容返回给 `stdout`, 不管它是否已经被读过。

在停止 `klogd` 后, 如果你偶尔手工读取内核消息, 你会发现 `/proc` 看起来象一个 `FIFO`,读者阻塞在里面, 等待更多数据。显然, 你无法以这种方式读消息, 如果 `klogd` 或者其他进程已经在读同样的数据, 因为你要竞争它。

如果环形缓存填满, `printk` 绕回并在缓存的开头增加新数据, 覆盖掉最老的数据。因此,这个记录过程会丢失最老的数据。这个问题相比于使用这样一个环形缓存的优点是可以忽略的。例如, 环形缓存允许系统即便没有一个日志进程也可运行, 在没有人读它的时候可以通过覆盖旧数据浪费最少的内存。Linux 对于消息的解决方法的另一个特性是, `printk`可以从任何地方调用, 甚至从一个中断处理里面, 没有限制能打印多少数据。唯一的缺点是可能丢失一些数据。

如果 `klogd` 进程在运行, 它获取内核消息并分发给 `syslogd`, `syslogd` 接着检查`/etc/syslog.conf` 来找出如何处理它们。`syslogd` 根据一个设施和一个优先级来区分消息; 这个设施和优先级的允许值在 `<sys/syslog.h>` 中定义. 内核消息由 `LOG_KERN` 设施来记录, 在一个对应于 `printk` 使用的优先级上(例如, `LOG_ERR` 用于 `KERN_ERR` 消息)。

如果 `klogd` 没有运行, 数据保留在环形缓存中直到有人读它或者缓存被覆盖。如果你要避免你的系统被来自你的驱动的监视消息击垮, 你或者给 `klogd` 指定一个 `-f`(文件) 选项来指示它保存消息到一个特定的文件, 或者定制 `/etc/syslog.conf` 来适应你的要求。但是另外一种可能性是采用粗暴的方式: 杀掉 `klogd` 和详细地打印消息在一个没有用到的虚拟终端上, 或者从一个没有用到的 `xterm` 上发出命令 `cat /proc/kmsg`。



### 4.2.4 打开和关闭消息

在驱动开发的早期, `printk` 非常有助于调试和测试新代码。当你正式发行驱动时, 换句话说, 你应当去掉, 或者至少关闭, 这些打印语句。不幸的是, 你很可能会发现, 就在你认为你不再需要这些消息并去掉它们时, 你要在驱动中实现一个新特性(或者有人发现了一个 `bug`), 你想要至少再打开一个消息。有几个方法来解决这 2 个问题, 全局性地打开或关闭你地调试消息和打开或关闭单个消息。这里我们展示一种编码 `printk` 调用的方法, 你可以单独或全局地打开或关闭它们; 这个技术依靠定义一个宏, 在你想使用它时就转变成一个 `printk` (或者 `printf`)调用。

* 每个 `printk` 语句可以打开或关闭, 通过去除或添加单个字符到宏定义的名子。
* 所有消息可以马上关闭, 通过在编译前改变 `CFLAGS` 变量的值。
* 同一个 `print` 语句可以在内核代码和用户级代码中使用, 因此对于格外的消息,驱动和测试程序能以同样的方式被管理。

下面的代码片断实现了这些特性, 直接来自头文件 `scull.h`：

~~~c
#include <linux/ioctl.h> /* needed for the _IOW etc stuff used later */

/*
 * Macros to help debugging
 */

#undef PDEBUG             /* undef it, just in case */
#ifdef SCULL_DEBUG
#  ifdef __KERNEL__
     /* This one if debugging is on, and kernel space */
#    define PDEBUG(fmt, args...) printk( KERN_DEBUG "scull: " fmt, ## args)
#  else
     /* This one for user space */
#    define PDEBUG(fmt, args...) fprintf(stderr, fmt, ## args)
#  endif
#else
#  define PDEBUG(fmt, args...) /* not debugging: nothing */
#endif

#undef PDEBUGG
#define PDEBUGG(fmt, args...) /* nothing: it's a placeholder */
~~~

符号 `PDEBUG` 定义和去定义, 取决于 `SCULL_DEBUG` 是否定义, 和以何种方式显示消息适合代码运行的环境: 当它在内核中就使用内核调用 `printk`, 在用户空间运行就使用 `libc`调用 `fprintf` 到标准错误输出。`PDEBUGG` 符号, 换句话说, 什么不作; 他可用来轻易地"注释" `print` 语句, 而不用完全去掉它们。为进一步简化过程, 添加下面的行到你的 `makfile` 里:

~~~makefile
# Comment/uncomment the following line to disable/enable debugging
DEBUG = y

# Add your debugging flag (or not) to CFLAGS
ifeq ($(DEBUG),y)
  DEBFLAGS = -O -g -DSCULL_DEBUG # "-O" is needed to expand inlines
else
  DEBFLAGS = -O2
endif
~~~

本节中出现的宏定义依赖 `gcc` 对 `ANSI C` 预处理器的扩展, 支持带可变个数参数的宏定义。这个 `gcc` 依赖不应该是个问题, 因为无论如何内核固有的非常依赖于 `gcc` 特性。另外, `makefile` 依赖 `GNU` 版本的 `make`; 再一次, 内核也依赖 `GNU make`, 所以这个依赖不是问题。

如果你熟悉 C 预处理器, 你可以扩展给定的定义来实现一个"调试级别"的概念, 定义不同的级别, 安排一个整数(或者位掩码)值给每个级别, 以便决定它应当多么详细。

但是每个驱动有它自己的特性和监视需求。好的编程技巧是在灵活性和效率之间选择最好的平衡, 我们无法告诉你什么是最好的。记住, 预处理器条件(连同代码中的常数表达式)在编译时执行, 因此你必须重新编译来打开或改变消息。一个可能的选择是使用 C 条件句, 它在运行时执行, 因而, 能允许你在出现执行时打开或改变消息机制。这是一个好的特性, 但是它在每次代码执行时需要额外的处理, 这样即便消息给关闭了也会影响效率。有时这个效率损失无法接受。本节出现的宏定义已经证明在多种情况下是有用的, 唯一的缺点是要求在任何对它的消息改变后重新编译。



### 4.2.5 速率限制

如果你不小心, 你会发现自己用 `printk` 产生了上千条消息, 压倒了控制台并且, 可能地,使系统日志文件溢出。当使用一个慢速控制台设备(例如, 一个串口), 过量的消息速率也能拖慢系统或者只是使它不反应了。非常难于着手于系统出错的地方, 当控制台不停地输出数据。因此, 你应当非常注意你打印什么, 特别在驱动的产品版本以及特别在初始化完成后。通常, 产品代码在正常操作时不应当打印任何东西; 打印的输出应当是指示需要注意的异常情况。

另一方面, 你可能想发出一个日志消息, 如果你驱动的设备停止工作。但是你应当小心不要做过了头。一个面对失败永远继续的傻瓜进程能产生每秒上千次的尝试; 如果你的驱动每次都打印"my device is broken", 它可能产生大量的输出, 如果控制台设备慢就有可能霸占 CPU -- 没有中断用来驱动控制台, 就算是一个串口或者一个行打印机。

在很多情况下, 最好的做法是设置一个标志说, "我已经抱怨过这个了", 并不打印任何后来的消息只要这个标志设置着。然而, 有几个理由偶尔发出一个"设备还是坏的"的提示。内核已经提供了一个函数帮助这个情况:

~~~c
int printk_ratelimit(void);
~~~

这个函数应当在你认为打印一个可能会常常重复的消息之前调用。如果这个函数返回非零值, 继续打印你的消息, 否则跳过它。这样, 典型的调用如这样:

~~~c
if (printk_ratelimit())
	printk(KERN_NOTICE "The printer is still on fire\n");
// printk_ratelimit 通过跟踪多少消息发向控制台而工作. 当输出级别超过一个限度,
// printk_ratelimit 开始返回 0 并使消息被扔掉.
~~~

`printk_ratelimit` 的行为可以通过修改 `/proc/sys/kern/printk_ratelimit`( 在重新使能消息前等待的秒数 ) 和 `/proc/sys/kernel/printk_ratelimit_burst`(限速前可接收的消息数)来定制。



### 4.2.6 打印设备编号

偶尔地, 当从一个驱动打印消息, 你会想打印与感兴趣的硬件相关联的设备号。打印主次编号不是特别难, 但是, 为一致性考虑, 内核提供了一些实用的宏定义( 在`<linux/kdev_t.h>` 中定义)用于这个目的:

~~~c
int print_dev_t(char *buffer, dev_t dev);
char *format_dev_t(char *buffer, dev_t dev);
~~~

两个宏定义都将设备号编码进给定的缓冲区; 唯一的区别是 `print_dev_t` 返回打印的字符数, 而 `format_dev_t` 返回缓存区; 因此, 它可以直接用作 `printk` 调用的参数, 但是必须记住 `printk` 只有提供一个结尾的新行才会刷行. 缓冲区应当足够大以存放一个设备号; 如果 64 位编号在以后的内核发行中明显可能, 这个缓冲区应当可能至少是 20 字节长。

~~~c
#include <linux/kdev_t.h>
#include <linux/printk.h>

void demo(dev_t dev)
{
    char buf[32];

    printk(KERN_INFO "device number: %s\n",
           format_dev_t(buf, dev)); // device number: 240:0
}
/********************************************************************/
void demo(dev_t dev)
{
    char buf[32];
    int len;

    len = print_dev_t(buf, dev);

    printk(KERN_INFO "device number(%d chars): %s\n",
           len, buf);
}

~~~

| 场景        | 推荐              |
| ----------- | ----------------- |
| 直接 printk | `format_dev_t()`  |
| 拼接字符串  | `print_dev_t()`   |
| 快速调试    | `MAJOR()/MINOR()` |
| 内核规范    | `format_dev_t()`  |



## 4.3 用查询来调试

大量使用 `printk` 能够显著地拖慢系统, 即便你降低 `cosole_loglevel` 来避免加载控制台设备, 因为 `syslogd` 会不停地同步它的输出文件; 因此, 要打印的每一行都引起一次磁盘操作。从 `syslogd` 的角度这是正确的实现。它试图将所有东西写到磁盘上, 防止系统刚好在打印消息后崩溃; 然而, 你不想只是为了调试信息的原因而拖慢你的系统。可以在出现于 `/etc/syslogd.conf` 中的你的日志文件名前加一个连字号来解决这个问题。改变配置文件带来的问题是, 这个改变可能在你结束调试后保留在那里, 即便在正常系统操作中你确实想尽快刷新消息到磁盘。这样永久改变的另外的选择是运行一个非 `klogd`程序( 例如 `cat /proc/kmsg`, 如之前建议的), 但是这可能不会提供一个合适的环境给正常的系统操作。

经常地, 最好的获得相关信息的方法是查询系统, 在你需要消息时, 不是连续地产生数据。实际上, 每个 `Unix` 系统提供许多工具来获取系统消息: `ps`, `netstat`, `vmstat`, 等等。有几个技术给驱动开发者来查询系统: 创建一个文件在 `/proc` 文件系统下, 使用 `ioctl` 驱动方法, 借助 sysfs 输出属性。使用 `sysfs` 需要不少关于驱动模型的背景知识. 在 14 章讨论。

这段话的核心意思是：**`printk` 本身不慢，真正把系统拖慢的是“日志被不停地同步写磁盘”**。下面我用**可运行、可观察现象的例子**一步一步说明。

### 4.3.0 说明

#### 一、问题场景：为什么大量 printk 会拖慢系统？

##### 场景设定（非常典型，驱动初学阶段常犯）

你在字符设备驱动里写了大量调试输出：

~~~c
ssize_t scull_read(struct file *filp, char __user *buf,
                   size_t count, loff_t *f_pos)
{
    printk(KERN_DEBUG "scull_read count=%zu\n", count);
    ...
}
~~~

然后在用户态：

~~~sh
while true; do
    cat /dev/scull0 > /dev/null
done
~~~

##### 表面现象

- CPU 占用不高；
- 但：系统明显卡顿，`iowait` 飙升，SSH / shell 反应变慢，嵌入式系统甚至“假死”。

#### 二、真正的罪魁祸首是谁？

##### printk 的真实路径

~~~(空)
printk
  ↓
kernel ring buffer
  ↓
klogd
  ↓
syslogd
  ↓
write() + fsync()
  ↓
磁盘

~~~

**每一条 printk，都会导致一次同步磁盘写**

##### 为什么 syslogd 要这么“傻”？

因为它的设计目标是：

> **如果系统在打印完日志后立刻崩溃，日志不能丢**

所以默认是：**同步写**，**尽快落盘**，从 **syslogd 的角度是完全正确的**。

#### 三、console_loglevel 降低 ≠ 问题解决

你可能做过这个：

```
echo 1 > /proc/sys/kernel/printk
```

或者：

```
printk(KERN_DEBUG ...)
```

##### 结果？

- 控制台不刷屏了 ✅
- **磁盘 IO 仍然很高 ❌**

原因是：

> `console_loglevel` **只控制是否输出到控制台**
>
> **不影响 syslogd 是否写磁盘**

#### 四、解决方案 1：syslogd 配置前加 `-`（重点）

##### syslogd 配置示例

原来的 `/etc/syslog.conf`（或 `/etc/syslogd.conf`）：

```
kern.*    /var/log/kern.log
```

改成：

```
kern.*   -/var/log/kern.log
```

##### 这个 `-` 是什么作用？

| 配置                 | 行为                   |
| -------------------- | ---------------------- |
| `/var/log/kern.log`  | 每条日志 **同步写盘**  |
| `-/var/log/kern.log` | **异步写盘**（有缓冲） |

##### 立竿见影的效果

- printk 再多，系统不卡了
- IO 压力明显下降
- 调试体验改善巨大

##### 代价（必须知道）

⚠️ 如果系统崩溃：**最后一部分日志可能丢失**。

------

#### 五、为什么文档说“这个改变可能被永久保留”？

问题在于：

> 你是为了 **调试** 才加 `-`，但调试结束后，**你可能忘了改回去**

在**正常生产系统**中：

- kernel panic
- OOPS
- 硬件异常

👉 **日志及时落盘非常重要**

所以：

- 调试时：`-`
- 正式系统：不加 `-`

------

#### 六、解决方案 2：不用 klogd / syslogd（临时调试）

##### 直接读取内核日志缓冲区

```
cat /proc/kmsg
```

或者：

```
dmesg -w
```

##### 特点

| 项目     | 说明         |
| -------- | ------------ |
| 磁盘 IO  | ❌ 不写       |
| 实时性   | ✅ 很好       |
| 系统影响 | 很小         |
| 适合     | **短期调试** |

##### 为什么不适合长期？

- 没有日志文件
- 系统重启全丢
- 无日志轮转
- 无安全控制

👉 **不适合正常系统运行**

------

#### 七、真实嵌入式 / 驱动调试例子（非常贴近你）

##### ❌ 错误示例

```
for (i = 0; i < count; i++) {
    printk(KERN_DEBUG "rx byte[%d]=0x%02x\n", i, data[i]);
}
```

👉 1 次 read = 几百次磁盘同步写

------

##### ✅ 正确调试方式

方式 1：限制 printk 频率

```
if (printk_ratelimit())
    printk(KERN_DEBUG "rx %zu bytes\n", count);
```

------

方式 2：条件编译

```
#ifdef DEBUG
printk(KERN_DEBUG "rx %zu bytes\n", count);
#endif
```

------

方式 3：使用动态调试（强烈推荐）

```
pr_debug("rx %zu bytes\n", count);
```

运行时开启：

```
echo 'module scull +p' > /sys/kernel/debug/dynamic_debug/control
```

------

#### 八、一句话总结（内核老兵经验）

> **拖慢系统的不是 printk，而是“你让日志同步写盘了”**

调试时：

- ✔ `pr_debug`
- ✔ `dynamic_debug`
- ✔ `dmesg -w`
- ✔ syslog 文件前加 `-`

生产系统：

- ❌ 大量 printk
- ❌ 异步内核日志
- ✔ 可靠落盘



### 4.3.1 使用 /proc 文件系统

`/proc` 文件系统是一个特殊的软件创建的文件系统, 内核用来输出消息到外界。 `/proc` 下的每个文件都绑到一个内核函数上, 当文件被读的时候即时产生文件内容。我们已经见到一些这样的文件起作用; 例如, `/proc/modules`, 常常返回当前已加载的模块列表。

`/proc` 在 Linux 系统中非常多地应用。很多现代 Linux 发布中的工具, 例如 `ps`, `top`, 以及 `uptime`, 从 `/proc` 中获取它们的信息. 一些设备驱动也通过 `/proc` 输出信息, 你的也可以这样做。`/proc` 文件系统是动态的, 因此你的模块可以在任何时候添加或去除条目。

完全特性的 `/proc` 条目可能是复杂的野兽; 另外, 它们可写也可读, 但是, 大部分时间,`/proc` 条目是只读的文件。本节只涉及简单的只读情况。那些感兴趣于实现更复杂的东西的人可以从这里获取基本知识; 接下来可参考内核源码来获知完整的信息。

在我们继续之前, 我们应当提及在 `/proc` 下添加文件是不鼓励的。`/proc `文件系统在内核开发者看作是有点无法控制的混乱, 它已经远离它的本来目的了(是提供关于系统中运行的进程的信息)。建议新代码中使信息可获取的方法是利用 `sysfs`。如同建议的, 使用`sysfs` 需要对 Linux 设备模型的理解, 然而, 我们直到 14 章才接触它。同时, `/proc`下的文件容易创建, 并且它们完全适合调试目的, 所以我们在这里包含它们。

#### 4.3.1.1 在 /proc 里实现文件

所有使用 `/proc` 的模块应当包含 `<linux/proc_fs.h>` 来定义正确的函数。要创建一个只读 `/proc` 文件，你的驱动必须实现一个函数来在文件被读时产生数据。当某个进程读文件时（使用`read`系统调用），这个请求通过这个函数到达你的模块。我们先看看这个函数并在本章后边讨论注册接口。

当一个进程读你的 `/proc` 文件, 内核分配了一页内存(就是说, `PAGE_SIZE` 字节), 驱动可以写入数据来返回给用户空间. 那个缓存区传递给你的函数, 是一个称为 `read_proc`的方法:

~~~c
int (*read_proc)(char *page, char **start, off_t offset, int count, 
                 int *eof, void *data);
~~~

`page` 指针是你写你的数据的缓存区; `start` 是这个函数用来说有关的数据写在页中哪里(下面更多关于这个); `offset` 和 `count` 对于 `read` 方法有同样的含义. `eof` 参数指向一个整数, 必须由驱动设置来指示它不再有数据返回, `data` 是驱动特定的数据指针, 你可以用做内部用途.

这个函数应当返回实际摆放于 `page` 缓存区的数据的字节数, 就象 `read` 方法对别的文件所作一样. 别的输出值是 `*eof` 和 `*start`。 `eof` 是一个简单的标志, 但是 `start` 值的使用有些复杂; 它的目的是帮助实现大的(超过一页) `/proc` 文件。

`start` 参数有些非传统的用法. 它的目的是指示哪里(哪一页)找到返回给用户的数据. 当调用你的 `proc_read` 方法, `*start` 将会是 `NULL`。如果你保持它为 `NULL`, 内核假定数据已放进 `page` 偏移是 0; 换句话说, 它假定一个头脑简单的 `proc_read` 版本, 它安放虚拟文件的整个内容到 `page`, 没有注意 `offset` 参数。如果, 相反, 你设置 `*start` 为一个 非NULL 值, 内核认为由 `*start` 指向的数据考虑了 `offset`, 并且准备好直接返回给用户。通常, 返回少量数据的简单 `proc_read` 方法只是忽略 `start`。更复杂的方法设置`*start` 为 `page` 并且只从请求的 `offset` 那里开始安放数据。

还有一段距离到 `/proc` 文件的另一个主要问题, 它也打算解答 `start`。有时内核数据结构的 `ASCII` 表示在连续的 `read` 调用中改变, 因此读进程可能发现从一个调用到下一个有不一致的数据。如果 `*start` 设成一个小的整数值, 调用者用它来递增 `filp-<f_pos`不依赖你返回的数据量, 因此使 `f_pos` 成为你的 `read_proc` 过程的一个内部记录数。如果, 例如, 如果你的 `read_proc` 函数从一个大结构数组返回信息并且第一次调用返回了5 个结构, `*start` 可设成5。下一个调用提供同一个数作为 `offset`; 驱动就知道从数组中第 6 个结构返回数据。这是被它的作者承认的一个" `hack` ", 可以在 `fs/proc/generic.c` 见到。

注意, 有更好的方法实现大的 `/proc` 文件; 它称为 `seq_file`, 我们很快会讨论它。首先,然而, 是时间举个例子了。下面是一个简单的(有点丑陋) `read_proc` 实现, 为 `scull` 设备:

~~~c
/*
 * The proc filesystem: function to read and entry
 */

int scull_read_procmem(struct seq_file *s, void *v)
{
        int i, j;
        int limit = s->size - 80; /* Don't print more than this */

        for (i = 0; i < scull_nr_devs && s->count <= limit; i++) {
                struct scull_dev *d = &scull_devices[i];
                struct scull_qset *qs = d->data;
                if (mutex_lock_interruptible(&d->lock))
                        return -ERESTARTSYS;
                seq_printf(s,"\nDevice %i: qset %i, q %i, sz %li\n",
                             i, d->qset, d->quantum, d->size);
                for (; qs && s->count <= limit; qs = qs->next) { /* scan the list */
                        seq_printf(s, "  item at %p, qset at %p\n",
                                     qs, qs->data);
                        if (qs->data && !qs->next) /* dump only the last item */
                                for (j = 0; j < d->qset; j++) {
                                        if (qs->data[j])
                                                seq_printf(s, "    % 4i: %8p\n",
                                                             j, qs->data[j]);
                                }
                }
                mutex_unlock(&scull_devices[i].lock);
        }
        return 0;
}
~~~

这是一个相当典型的 `read_proc` 实现. 它假定不会有必要产生超过一页数据并且因此忽略了 `start` 和 `offset` 值。它是, 但是, 小心地不覆盖它的缓存, 只是以防万一。

#### 4.3.1.2 老接口

如果你阅览内核源码, 你会遇到使用老接口实现 `/proc` 的代码:

~~~c
int (*get_info)(char *page, char **start, off_t offset, int count);
~~~

所有的参数的含义同 `read_proc` 的相同, 但是没有 `eof` 和 `data` 参数。这个接口仍然支持, 但是将来会消失; 新代码应当使用 `read_proc` 接口来代替。

#### 4.3.1.3 创建你的 /proc 文件

一旦你有一个定义好的 `read_proc` 函数, 你应当连接它到 `/proc` 层次中的一个入口项。使用一个 `creat_proc_read_entry` 调用:

~~~c
struct proc_dir_entry *create_proc_read_entry(const char *name,mode_t mode,
		struct proc_dir_entry *base, read_proc_t *read_proc, void *data);
~~~

这里, `name` 是要创建的文件名子, `mod` 是文件的保护掩码(缺省系统范围时可以作为 0传递), `base` 指出要创建的文件的目录( 如果 `base` 是 `NULL`, 文件在 `/proc` 根下创建 ),`read_proc` 是实现文件的 `read_proc` 函数, `data` 被内核忽略( 但是传递给 `read_proc`)。这就是 scull 使用的调用, 来使它的 `/proc` 函数可用做 `/proc/scullmem`:

~~~c
/*
 * Actually create (and remove) the /proc file(s).
 */

static void scull_create_proc(void)
{
	proc_create_data("scullmem", 0 /* default mode */,
			NULL /* parent dir */, proc_ops_wrapper(&scullmem_proc_ops, scullmem_pops),
			NULL /* client data */);
	proc_create("scullseq", 0, NULL, proc_ops_wrapper(&scullseq_proc_ops, scullseq_pops));
}

~~~

这里, 我们创建了一个名为 `scullmem` 的文件, 直接在 `/proc` 下, 带有缺省的, 全局可读的保护。

目录入口指针可用来在 `/proc` 下创建整个目录层次。但是, 注意, 一个入口放在 `/proc`的子目录下会更容易, 通过简单地给出目录名子作为这个入口名子的一部分 -- 只要这个目录自身已经存在。例如, 一个(常常被忽略)传统的是 `/proc` 中与设备驱动相连的入口应当在 `driver/` 子目录下; scull 能够安放它的入口在那里, 简单地通过指定它为名子`driver/scullmem`。

`/proc` 中的入口, 当然, 应当在模块卸载后去除. `remove_proc_entry` 是恢复`create_proc_read_entry` 所做的事情的函数:

~~~c
static void scull_remove_proc(void)
{
	/* no problem if it was not registered */
	remove_proc_entry("scullmem", NULL /* parent dir */);
	remove_proc_entry("scullseq", NULL);
}

~~~

去除入口失败会导致在不希望的时间调用, 或者, 如果你的模块已被卸载, 内核崩掉.当如展示的使用 `/proc` 文件, 你必须记住几个实现的麻烦事 -- 不要奇怪现在不鼓励使用它。

最重要的问题是关于去除 `/proc` 入口. 这样的去除很可能在文件使用时发生, 因为没有所有者关联到 `/proc` 入口, 因此使用它们不会作用到模块的引用计数。这个问题可以简单的触发, 例如通过运行 `sleep 100 < /proc/myfile`, 刚好在去除模块之前。

另外一个问题时关于用同样的名子注册两个入口. 内核信任驱动, 不会检查名子是否已经注册了, 因此如果你不小心, 你可能会使用同样的名子注册两个或多个入口。这是一个已知发生在教室中的问题, 这样的入口是不能区分的, 不但在你存取它们时, 而且在你调用`remove_proc_entry` 时。

#### 4.3.1.4 seq_file 接口

如我们上面提到的, 在 `/proc` 下的大文件的实现有点麻烦。一直以来, `/proc` 方法因为当输出数量变大时的错误实现变得声名狼藉。作为一种清理 `/proc` 代码以及使内核开发者活得轻松些的方法, 添加了 `seq_file` 接口。这个接口提供了简单的一套函数来实现大内核虚拟文件。

`set_file` 接口假定你在创建一个虚拟文件, 它涉及一系列的必须返回给用户空间的项.为使用 `seq_file`, 你必须创建一个简单的 "`iterator`" 对象, 它能在序列里建立一个位置, 向前进, 并且输出序列里的一个项。它可能听起来复杂, 但是, 实际上, 过程非常简单。我们一步步来创建 `/proc` 文件在 `scull` 驱动里, 来展示它是如何做的。第一步, 不可避免地, 是包含 `<linux/seq_file.h>`. 接着你必须创建 4 个 `iterator` 方法, 称为 `start`, `next`, `stop`, 和 `show`。

**`start` 方法一直是首先调用. 这个函数的原型是:**

~~~c
void *start(struct seq_file *sfile, loff_t *pos);
~~~

`sfile` 参数可以几乎是一直被忽略。 `pos` 是一个整型位置值, 指示应当从哪里读. 位置的解释完全取决于实现; 在结果文件里不需要是一个字节位置。因为 `seq_file` 实现典型地步进一系列感兴趣的项, `position` 常常被解释为指向序列中下一个项的指针。`scull` 驱动解释每个设备作为系列中的一项, 因此进入的 `pos` 简单地是一个 `scull_device` 数组的索引。因此, scull 使用的 `start` 方法是:

~~~~c
/*
 * Here are our sequence iteration methods.  Our "position" is
 * simply the device number.
 */
static void *scull_seq_start(struct seq_file *s, loff_t *pos)
{
	if (*pos >= scull_nr_devs)
		return NULL;   /* No more to read */
	return scull_devices + *pos;
}
~~~~

返回值, 如果非`NULL`, 是一个可以被 `iterator` 实现使用的私有值。

**`next` 函数应当移动 `iterator` 到下一个位置, 如果序列里什么都没有剩下就返回 `NULL`。这个方法的原型是:**

~~~c
void *next(struct seq_file *sfile, void *v, loff_t *pos);
~~~

这里, `v` 是从前一个对 `start` 或者 `next` 的调用返回的 `iterator`, `pos` 是文件的当前位置. `next` 应当递增有 `pos` 指向的值; 根据你的 `iterator` 是如何工作的, 你可能(尽管可能不会)需要递增 `pos` 不止是 1. 这是 scull 所做的:

~~~c
static void *scull_seq_next(struct seq_file *s, void *v, loff_t *pos)
{
	(*pos)++;
	if (*pos >= scull_nr_devs)
		return NULL;
	return scull_devices + *pos;
}

~~~

**当内核处理完 `iterator`, 它调用 `stop` 来清理:**

~~~c
void stop(struct seq_file *sfile, void *v);
static void scull_seq_stop(struct seq_file *s, void *v)
{
	/* Actually, there's nothing to do here */
}
~~~

`scull` 实现没有清理工作要做, 所以它的 `stop` 方法是空的。

设计上, 值得注意 `seq_file` 代码在调用 `start` 和 `stop` 之间不睡眠或者进行其他非原子性任务。你也肯定会看到在调用 `start` 后马上有一个 `stop` 调用。因此, 对你的`start` 方法来说请求信号量或自旋锁是安全的。只要你的其他 `seq_file` 方法是原子的,调用的整个序列是原子的。(如果这一段对你没有意义, 在你读了下一章后再回到这。)

**在这些调用中, 内核调用 `show` 方法来真正输出有用的东西给用户空间。这个方法的原型是:**

~~~c
int show(struct seq_file *sfile, void *v);
~~~

这个方法应当创建序列中由 `iterator` ` v` 指示的项的输出。不应当使用 `printk`, 但是;有一套特殊的用作 `seq_file` 输出的函数:

~~~c
int seq_printf(struct seq_file *sfile, const char *fmt, ...);
// 这是给 seq_file 实现的 printf 对等体; 它采用常用的格式串和附加值参数. 你
// 必须也将给 show 函数的 set_file 结构传递给它, 然而. 如果seq_printf 返回
// 非零值, 意思是缓存区已填充, 输出被丢弃. 大部分实现忽略了返回值, 但是.
int seq_putc(struct seq_file *sfile, char c);
int seq_puts(struct seq_file *sfile, const char *s);
// 它们是用户空间 putc 和 puts 函数的对等体.
int seq_escape(struct seq_file *m, const char *s, const char *esc);
// 这个函数是 seq_puts 的对等体, 除了 s 中的任何也在 esc 中出现的字符以八进
// 制格式打印. esc 的一个通用值是"\t\n\\", 它使内嵌的空格不会搞乱输出和可能
// 搞乱 shell 脚本.
int seq_path(struct seq_file *sfile, struct vfsmount *m, 
             struct dentry *dentry, char *esc);
// 这个函数能够用来输出和给定命令项关联的文件名子. 它在设备驱动中不可能有用;
// 我们是为了完整在此包含它.
~~~

回到我们的例子; 在 scull 使用的 `show` 方法是:

~~~c
static int scull_seq_show(struct seq_file *s, void *v)
{
	struct scull_dev *dev = (struct scull_dev *) v;
	struct scull_qset *d;
	int i;

	if (mutex_lock_interruptible(&dev->lock))
		return -ERESTARTSYS;
	seq_printf(s, "\nDevice %i: qset %i, q %i, sz %li\n",
			(int) (dev - scull_devices), dev->qset,
			dev->quantum, dev->size);
	for (d = dev->data; d; d = d->next) { /* scan the list */
		seq_printf(s, "  item at %p, qset at %p\n", d, d->data);
		if (d->data && !d->next) /* dump only the last item */
			for (i = 0; i < dev->qset; i++) {
				if (d->data[i])
					seq_printf(s, "    % 4i: %8p\n",
							i, d->data[i]);
			}
	}
	mutex_unlock(&dev->lock);
	return 0;
}
~~~

这里, 我们最终解释我们的" `iterator`" 值, 简单地是一个 `scull_dev` 结构指针。现在已有了一个完整的 `iterator` 操作的集合, scull 必须包装起它们, 并且连接它们到`/proc` 中的一个文件。 第一步是填充一个 `seq_operations` 结构:

~~~c
/*
 * Tie the sequence operators up.
 */
static struct seq_operations scull_seq_ops = {
	.start = scull_seq_start,
	.next  = scull_seq_next,
	.stop  = scull_seq_stop,
	.show  = scull_seq_show
};
~~~

有那个结构在, 我们必须创建一个内核理解的文件实现。我们不使用前面描述过的`read_proc` 方法; 在使用 `seq_file` 时, 最好在一个稍低的级别上连接到 `/proc`。那意味着创建一个 `file_operations` 结构(是的, 和字符驱动使用的同样结构) 来实现所有内核需要的操作, 来处理文件上的读和移动。幸运的是, 这个任务是简单的。第一步是创建一个 `open` 方法连接文件到 `seq_file` 操作:

~~~c
/*
 * Now to implement the /proc files we need only make an open
 * method which sets up the sequence operators.
 */
static int scullmem_proc_open(struct inode *inode, struct file *file)
{
	return single_open(file, scull_read_procmem, NULL);
}

static int scullseq_proc_open(struct inode *inode, struct file *file)
{
	return seq_open(file, &scull_seq_ops);
}
~~~

调用 `seq_open` 连接文件结构和我们上面定义的序列操作。事实证明, `open` 是我们必须自己实现的唯一文件操作, 因此我们现在可以建立我们的 `file_operations` 结构:

~~~c
/*
 * Create a set of file operations for our proc files.
 */
static struct file_operations scullmem_proc_ops = {
	.owner   = THIS_MODULE,
	.open    = scullmem_proc_open,
	.read    = seq_read,
	.llseek  = seq_lseek,
	.release = single_release
};

static struct file_operations scullseq_proc_ops = {
	.owner   = THIS_MODULE,
	.open    = scullseq_proc_open,
	.read    = seq_read,
	.llseek  = seq_lseek,
	.release = seq_release
};
	
~~~

这里我们指定我们自己的 `open` 方法, 但是使用预装好的方法 `seq_read`, `seq_lseek`, 和`seq_release` 给其他。最后的步骤是创建 `/proc` 中的实际文件:

~~~c
/*
 * Actually create (and remove) the /proc file(s).
 */

static void scull_create_proc(void)
{
	proc_create_data("scullmem", 0 /* default mode */,
			NULL /* parent dir */, proc_ops_wrapper(&scullmem_proc_ops, scullmem_pops),
			NULL /* client data */);
	proc_create("scullseq", 0, NULL, proc_ops_wrapper(&scullseq_proc_ops, scullseq_pops));
}

static void scull_remove_proc(void)
{
	/* no problem if it was not registered */
	remove_proc_entry("scullmem", NULL /* parent dir */);
	remove_proc_entry("scullseq", NULL);
}
~~~

 `create_proc_entry()` 已经被内核废弃并移除了，不能再用了；现在必须用 `proc_create()` / `proc_create_data()` + `struct proc_ops`。`remove_proc_entry` 没有弃用但是也不推荐使用了，建议`proc_remove` 替代，参考如下：

~~~c
static struct proc_dir_entry *scullmem_entry;

scullmem_entry = proc_create("scullmem", 0, NULL, &scullmem_ops);

/* 模块卸载时 */
proc_remove(scullmem_entry);

~~~

有了上面代码, scull 有一个新的 `/proc` 入口, 看来很象前面的一个。但是, 它是高级的, 因为它不管它的输出有多么大, 它正确处理移动, 并且通常它是易读和易维护的。我们建议使用 `seq_file` , 来实现包含多个非常小数目的输出行数的文件。

### 4.3.2 ioctl 方法

`ioctl`, 我们在第 1 章展示给你如何使用, 是一个系统调用, 作用于一个文件描述符; 它接收一个确定要进行的命令的数字和(可选地)另一个参数, 常常是一个指针。作为一个使用 `/proc` 文件系统的替代, 你可以实现几个用来调试用的 `ioctl` 命令。这些命令可以从驱动拷贝相关的数据结构到用户空间, 这里你可以检查它们。

这种方式使用 `ioctl` 来获取信息有些比使用 `/proc` 困难, 因为你需要另一个程序来发出`ioctl` 并且显示结果。必须编写这个程序, 编译, 并且与你在测试的模块保持同步。另一方面, 驱动侧代码可能容易过需要实现一个 `/proc` 文件的代码。

有时候 `ioctl` 是获取信息最好的方法, 因为它运行比读取 `/proc` 快。如果在数据写到屏幕之前必须做一些事情, 获取二进制形式的数据比读取一个文本文件要更有效。另外,`ioctl` 不要求划分数据为小于一页的片段。

`ioctl` 方法的另一个有趣的优点是信息获取命令可留在驱动中, 当调试被禁止时。不象对任何查看目录的人(并且太多人可能奇怪"这个怪文件是什么")都可见的 `/proc` 文件, 不记入文档的 `ioctl` 命令可能保持不为人知。另外, 如果驱动发生了怪异的事情, 它们仍将在那里。唯一的缺点是模块可能会稍微大些。



## 4.4 使用观察来调试

有时小问题可以通过观察用户空间的应用程序的行为来追踪。监视程序也有助于建立对驱动正确工作的信心。例如, 我们能够对 scull 感到有信心, 在看了它的读实现如何响应不同数量数据的读请求之后。

有几个方法来监视用户空间程序运行。你可以运行一个调试器来单步过它的函数, 增加打印语句, 或者在 `strace` 下运行程序。这里, 我们将讨论最后一个技术, 当真正目的是检查内核代码时它是最有趣的。

`strace` 命令时一个有力工具, 显示所有的用户空间程序发出的系统调用。它不仅显示调用, 还以符号形式显示调用的参数和返回值。当一个系统调用失败, 错误的符号值(例如,`ENOMEM`)和对应的字串(Out of memory) 都显示。`strace` 有很多命令行选项; 其中最有用的是 `-t` 来显示每个调用执行的时间, `-T` 来显示调用中花费的时间, `-e` 来限制被跟踪调用的类型, 以及`-o` 来重定向输出到一个文件。 缺省地, `strace` 打印调用信息到 `stderr`。

strace 从内核自身获取信息. 这意味着可以跟踪一个程序, 不管它是否带有调试支持编译(对 gcc 是 -g 选项)以及不管它是否 strip 过. 你也可以连接追踪到一个运行中的进程, 类似于一个调试器的方式连接到一个运行中的进程并控制它.跟踪信息常用来支持发给应用程序开发者的故障报告, 但是对内核程序员也是很有价值的.我们已经看到驱动代码运行如何响应系统调用; strace 允许我们检查每个调用的输入和输出数据的一致性.

例如, 下面的屏幕输出显示(大部分)运行命令 `strace ls /dev > /dev/scull0` 的最后的行:

~~~(空)
open("/dev", O_RDONLY|O_NONBLOCK|O_LARGEFILE|O_DIRECTORY) = 3
fstat64(3, {st_mode=S_IFDIR|0755, st_size=24576, ...}) = 0
fcntl64(3, F_SETFD, FD_CLOEXEC) = 0
getdents64(3, /* 141 entries */, 4096) = 4088
[...]
getdents64(3, /* 0 entries */, 4096) = 0
close(3) = 0
[...]
fstat64(1, {st_mode=S_IFCHR|0664, st_rdev=makedev(254, 0), ...}) = 0
write(1, "MAKEDEV\nadmmidi0\nadmmidi1\nadmmid"..., 4096) = 4000
write(1, "b\nptywc\nptywd\nptywe\nptywf\nptyx0\n"..., 96) = 96
write(1, "b\nptyxc\nptyxd\nptyxe\nptyxf\nptyy0\n"..., 4096) = 3904
write(1, "s17\nvcs18\nvcs19\nvcs2\nvcs20\nvcs21"..., 192) = 192
write(1, "\nvcs47\nvcs48\nvcs49\nvcs5\nvcs50\nvc"..., 673) = 673
close(1) = 0
exit_group(0) = ?
~~~

从第一个 `write` 调用看, 明显地, 在 `ls` 结束查看目标目录后, 它试图写 `4KB`. 奇怪地(对`ls`), 只有 `4000 字节`写入, 并且操作被重复。但是, 我们知道 scull 中的写实现一次写一个单个量子, 因此我们本来就期望部分写。几步之后, 所有东西清空, 程序成功退出。

作为另一个例子, 让我们读取 scull 设备`strace wc /dev/scull0`(使用 `wc` 命令):

~~~(空)
[...]
open("/dev/scull0", O_RDONLY|O_LARGEFILE) = 3
fstat64(3, {st_mode=S_IFCHR|0664, st_rdev=makedev(254, 0), ...}) = 0
read(3, "MAKEDEV\nadmmidi0\nadmmidi1\nadmmid"..., 16384) = 4000
read(3, "b\nptywc\nptywd\nptywe\nptywf\nptyx0\n"..., 16384) = 4000
read(3, "s17\nvcs18\nvcs19\nvcs2\nvcs20\nvcs21"..., 16384) = 865
read(3, "", 16384) = 0
fstat64(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 1), ...}) = 0
write(1, "8865 /dev/scull0\n", 17) = 17
close(3) = 0
exit_group(0) = ?
~~~

如同期望的, `read` 一次只能获取 `4000 字节`, 但是数据总量等同于前个例子写入的。注意在这个例子里读取是如何组织的, 同前面跟踪的相反。`wc` 为快速读被优化过, 因此绕过了标准库, 试图一个系统调用读取更多数据。你可从跟踪的读的行里看到 `wc` 是如何试图一次读取 `16 KB`。



## 4.5 调试系统故障

即便你已使用了所有的监视和调试技术, 有时故障还留在驱动里, 当驱动执行时系统出错。当发生这个时, 能够收集尽可能多的信息来解决问题是重要的。

注意"故障"不意味着"崩溃"。Linux 代码是足够健壮地优雅地响应大部分错误:一个故障常常导致当前进程的破坏而系统继续工作。系统可能崩溃, 如果一个故障发生在一个进程的上下文之外, 或者如果系统的一些至关重要的部分毁坏了。但是当是一个驱动错误导致的问题, 它常常只会导致不幸使用驱动的进程的突然死掉。当进程被销毁时唯一无法恢复的破坏是分配给进程上下文的一些内存丢失了; 例如, 驱动通过 kmalloc 分配的动态列表可能丢失。但是, 因为内核为任何一个打开的设备在进程死亡时调用关闭操作, 你的驱动可以释放由 open 方法分配的东西。

尽管一个 oops 常常都不会关闭整个系统, 你很有可能发现在发生一次后需要重启系统。一个满是错误的驱动能使硬件处于不能使用的状态, 使内核资源处于不一致的状态, 或者,最坏的情况, 在随机的地方破坏内核内存。常常你可简单地卸载你的破驱动并且在一次oops 后重试。然而, 如果你看到任何东西建议说系统作为一个整体不太好了, 你最好立刻重启。

我们已经说过, 当内核代码出错, 一个提示性的消息打印在控制台上。下一节解释如何解释并利用这样的消息。尽管它们对新手看来相当模糊, 处理器转储是很有趣的信息, 常常足够来查明一个程序错误而不需要附加的测试。

### 4.5.1 oops 消息

大部分 bug 以解引用 `NULL` 指针或者使用其他不正确指针值来表现自己的。此类 bug 通常的输出是一个 `oops` 消息。



















~~~c

~~~

> 1

------------------------

