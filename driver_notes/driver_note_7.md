# Linux 设备驱动

# 第七章 时间、延时和延后工作

到此, 我们知道了如何编写一个全特性字符模块的基本知识。真实世界的驱动, 然而, 需要做比实现控制一个设备的操作更多的事情; 它们不得不处理诸如定时, 内存管理, 硬件存取, 等更多. 幸运的是, 内核输出了许多设施来简化驱动编写者的任务。在下几章中,我们将描述一些你可使用的内核资源。这一章引路, 通过描述定时问题是如何阐述。处理时间问题包括下列任务, 按照复杂度上升的顺序:

* 测量时间流失和比较时间
* 知道当前时间
* 指定时间量的延时操作
* 调度异步函数在之后的时间发生



## 7.1 测量时间流失

内核通过定时器中断来跟踪时间的流动。中断在第 10 章详细描述。

定时器中断由系统定时硬件以规律地间隔产生; 这个间隔在启动时由内核根据 HZ 值来编程, HZ 是一个体系依赖的值, 在 `<linux/param.h>`中定义或者它所包含的一个子平台文件中。在发布的内核源码中的缺省值在真实硬件上从 50 到 1200 嘀哒每秒, 在软件模拟器中往下到 24。大部分平台运行在 100 或者 1000 中断每秒; 流行的 `x86` PC 缺省是1000, 尽管它在以前版本上(向上直到并且包括 2.4)常常是 100。作为一个通用的规则,即便如果你知道 HZ 的值, 在编程时你应当从不依赖这个特定值。

可能改变 HZ 的值, 对于那些要系统有一个不同的时钟中断频率的人。如果你在头文件中改变 HZ 的值, 你需要使用新的值重编译内核和所有的模块。 如果你愿意付出额外的时间中断的代价来获得你的目标, 你可能想提升 HZ 来得到你的异步任务的更细粒度的精度。实际上, 提升 HZ 到 1000 在使用 `2.4` 或 `2.2` 内核版本的 `x86` 工业系统中是相当普遍的。但是, 对于当前版本, 最好的方法是保持 HZ 的缺省值, 由于我们完全信任内核开发者, 他们肯定已经选择了最好的值。另外, 一些内部计算当前实现为只为从 12 到 1535范围的 HZ (见 `<linux/timex.h>` 和 `RFC-1589`)。

每次发生一个时钟中断, 一个内核计数器的值递增。这个计数器在系统启动时初始化为 0,因此它代表从最后一次启动以来的时钟嘀哒的数目。这个计数器是一个 64-位 变量( 即便在 32-位的体系上)并且称为 `jiffies_64`。但是, 驱动编写者正常地存取 `jiffies` 变量, 一个 `unsigned long`, 或者和 `jiffies_64` 是同一个或者它的低有效位。使用`jiffies` 常常是首选, 因为它更快, 并且再所有的体系上存取 64-位的 `jiffies_64` 值不必要是原子的。

除了低精度的内核管理的 `jiffy` 机制, 一些 CPU 平台特有一个高精度的软件可读的计数器。尽管它的实际使用有些在各个平台不同, 它有时是一个非常有力的工具。



### 7.1.1 使用 jiffies 计数器

这个计数器和来读取它的实用函数位于 `<linux/jiffies.h>`, 尽管你会常常只是包含`<linux/sched.h>`, 它会自动地将 `jiffies.h` 拉进来。不用说, `jiffies` 和 `jiffies_64` 必须当作只读的。

~~~c
#include <linux/jiffies.h>
unsigned long j, stamp_1, stamp_half, stamp_n;

j = jiffies; /* read the current value */
stamp_1 = j + HZ; /* 1 second in the future */
stamp_half = j + HZ/2; /* half a second */
stamp_n = j + n * HZ / 1000; /* n milliseconds */
~~~

这个代码对于 `jiffies` 回绕没有问题, 只要不同的值以正确的方式进行比较。尽管在HZ 是 1000 时, 计数器只是每 50 天回绕一次, 你的代码应当准备面对这个事件。为比较你的被缓存的值( 象上面的 `stamp_1` ) 和当前值, 你应当使用下面一个宏定义:

~~~c
#include <linux/jiffies.h>
int time_after(unsigned long a, unsigned long b);
int time_before(unsigned long a, unsigned long b);
int time_after_eq(unsigned long a, unsigned long b);
int time_before_eq(unsigned long a, unsigned long b);
~~~

第一个当 a, 作为一个 `jiffies` 的快照, 代表 b 之后的一个时间时, 取值为真, 第二个当 时间 a 在时间 b 之前时取值为真, 以及最后 2 个比较"之后或相同"和"之前或相同"。这个代码工作通过转换这个值为 `signed long`, 减它们, 并且比较结果。如果你需要以一种安全的方式知道 2 个 `jiffies` 实例之间的差, 你可以使用同样的技巧:

~~~c
diff = (long)t2 - (long)t1;
// 你可以转换一个 jiffies 差为毫秒, 一般地通过:
msec = diff * 1000 / HZ;
~~~

有时你需要与用户空间程序交换时间表示, 它们打算使用 `struct timeval` 和`struct timespec` 来表示时间。这 2 个结构代表一个精确的时间量, 使用 2 个成员:`seconds` 和 `microseconds` 在旧的流行的 `struct timeval` 中使用, `seconds` 和`nanoseconds` 在新的 `struct timespec` 中使用。内核输出 4 个帮助函数来转换以`jiffies` 表达的时间值, 到和从这些结构:

~~~c
#include <linux/time.h>
unsigned long timespec_to_jiffies(struct timespec *value);
void jiffies_to_timespec(unsigned long jiffies, struct timespec *value);
unsigned long timeval_to_jiffies(struct timeval *value);
void jiffies_to_timeval(unsigned long jiffies, struct timeval *value);
~~~

存取这个 64-位 `jiffy` 计数值不象存取` jiffies` 那样直接. 而在 64-位 计算机体系上,这 2 个变量实际上是一个, 存取这个值对于 32-位 处理器不是原子的。这意味着你可能读到错误的值如果这个变量的两半在你正在读取它们时被更新。极不可能你会需要读取这个 64-位 计数器, 但是万一你需要, 你会高兴地得知内核输出了一个特别地帮助函数,为你完成正确地加锁:

~~~c
#include <linux/jiffies.h>
u64 get_jiffies_64(void);
~~~

在上面的原型中, 使用了 u64 类型。这是一个定义在 <linux/types.h> 中的类型, 在11 章中讨论, 并且表示一个 unsigned 64-位 类型。

如果你在奇怪 32-位 平台如何同时更新 32-位 和 64-位 计数器, 读你的平台的连接脚本( 查找一个文件, 它的名子匹配 `valinux*.lds*`)。在那里, jiffies 符号被定义来存取这个 64-位 值的低有效字, 根据平台是小端或者大端。实际上, 同样的技巧也用在64-位 平台上, 因此这个 `unsigned long` 和 u64 变量在同一个地址被存取。

最后, 注意实际的时钟频率几乎完全对用户空间隐藏。宏 HZ 一直扩展为 100 当用户空间程序包含 `param.h`, 并且每个报告给用户空间的计数器都对应地被转换。这应用于`clock(3)`, `times(2)`, 以及任何相关的函数。对 HZ 值的用户可用的唯一证据是时钟中断多快发生, 如在 `/proc/interrupts` 所显示的。例如, 你可以获得 HZ, 通过用在`/proc/uptime` 中报告的系统 `uptime` 除这个计数值。



### 7.1.2 处理器特定的寄存器

如果你需要测量非常短时间间隔, 或者你需要非常高精度, 你可以借助平台依赖的资源,一个要精度不要移植性的选择。

在现代处理器中, 对于经验性能数字的迫切需求被大部分 CPU 设计中内在的指令定时不确定性所阻碍, 这是由于缓存内存, 指令调度, 以及分支预测引起。作为回应, CPU 制造商引入一个方法来计数时钟周期, 作为一个容易并且可靠的方法来测量时间流失。因此,大部分现代处理器包含一个计数器寄存器, 它在每个时钟周期固定地递增一次。现在, 这个时钟计数器是唯一可靠的方法来进行高精度的时间管理任务。

细节每个平台不同: 这个寄存器可以或者不可以从用户空间可读, 它可以或者不可以写,并且它可能是 64 或者 32 位宽。在后一种情况, 你必须准备处理溢出, 就象我们处理`jiffy` 计数器一样。这个寄存器甚至可能对你的平台来说不存在, 或者它可能被硬件设计外部设备实现, 如果 CPU 缺少这个特性并且你在使用一个特殊用途的计算机。无论是否寄存器可以被清零, 我们强烈不鼓励复位它, 即便当硬件允许时。毕竟, 在任何给定时间你可能不是这个计数器的唯一用户; 在一些支持 SMP 的平台上, 例如, 内核依赖这样一个计数器来在处理器之间同步。因为你可以一直测量各个值的差, 只要差没有超过溢出时间, 你可以通过修改它的当前值来做这个事情不用声明独自拥有这个寄存器。

最有名的计数器寄存器是 `TSC` ( timestamp counter), 在 `x86` 处理器中随 Pentium 引入的并且在所有从那之后的 CPU 中出现 -- 包括 `x86_64` 平台。它是一个 64-位 寄存器计数 CPU 的时钟周期; 它可从内核和用户空间读取。在包含了` <asm/msr.h>` (一个 `x86`-特定的头文件, 它的名子代表"`machine-specific registers`"), 你可使用一个这些宏:

~~~c
rdtsc(low32,high32);
rdtscl(low32);
rdtscll(var64);
~~~

第一个宏自动读取 64-位 值到 2 个 32-位 变量; 下一个("`read low half`") 读取寄存器的低半部到一个 32-位 变量, 丢弃高半部; 最后一个读 64-位 值到一个 `long long` 变量, 由此得名. 所有这些宏存储数值到它们的参数中。

对大部分的 `TSC` 应用, 读取这个计数器的的低半部足够了。 一个 `1-GHz` 的 CPU 只在每4.2 秒溢出一次, 因此你不会需要处理多寄存器变量, 如果你在使用的时间流失确定地使用更少时间。但是, 随着 CPU 频率不断上升以及定时需求的提高, 将来你会几乎可能需要常常读取 64-位 计数器。

作为一个只使用寄存器低半部的例子, 下面的代码行测量了指令自身的执行:

~~~c
unsigned long ini, end;
rdtscl(ini); rdtscl(end);
printk("time lapse: %li\n", end - ini);
~~~

一些其他的平台提供相似的功能, 并且内核头文件提供一个体系独立的功能, 你可用来代替 rdtsc。它称为 `get_cycles`, 定义在 `<asm/timex.h>`( 由 `<linux/timex.h>` 包含)。它的原型是:

~~~c
#include <linux/timex.h>
cycles_t get_cycles(void);
~~~

这个函数为每个平台定义, 并且它一直返回 0 在没有周期-计数器寄存器的平台上。`cycles_t` 类型是一个合适的 `````````````unsigned````````````` 类型来持有读到的值。

> 1. 再现代的linux系统中，**用户态**推荐使用 `clock_gettime()` 例如：
>
> ~~~c
> #include <time.h>
> #include <stdio.h>
> 
> int main() {
>     struct timespec start, end;
> 
>     clock_gettime(CLOCK_MONOTONIC_RAW, &start);
> 
>     // 要测试的代码
>     for (volatile int i = 0; i < 1000000; i++);
> 
>     clock_gettime(CLOCK_MONOTONIC_RAW, &end);
> 
>     long long diff =
>         (end.tv_sec - start.tv_sec) * 1000000000LL +
>         (end.tv_nsec - start.tv_nsec);
> 
>     printf("time: %lld ns\n", diff);
> }
> // 推荐使用：CLOCK_MONOTONIC_RAW
> /* 优点：；
>  * 不受 NTP 调整影响
>  * 单调递增
>  * 适合性能测量
>  */
> ~~~
>
> 2. 如果是想替代 rdtscl ，rdtscl 本质是读取CPU的时间戳计数器 TSC，现代的写法是：
>
> ~~~c
> #include <x86intrin.h>
> 
> unsigned long long t = __rdtsc();
> /* 或者 */
> static inline unsigned long long rdtsc(void)
> {
>     unsigned int lo, hi;
>     __asm__ __volatile__ ("rdtsc" : "=a"(lo), "=d"(hi));
>     return ((unsigned long long)hi << 32) | lo;
> }
> /* 注意：
>  * 多核之间 TSC 可能不同步（现代 CPU 基本已解决）
>  * CPU 频率变化会影响结果（旧 CPU）
>  * 虚拟机里可能不准确（尤其是在 WSL/VM）
>  */
> ~~~
>
> 3. 现代**内核里** get_cycles 怎么写：
>
> ~~~c
> #include <linux/timex.h>
> u64 t = get_cycles();
> /* 它内部会：
>  * x86 上用 rdtsc
>  * ARM 上用 arch counter
>  * 不同架构自动适配
>  */
> ~~~
>
> 4. 现代**内核开发**中更推荐 `ktime_get_ns()` 或 `ktime_get()` :
>
> ~~~c
> #include <linux/ktime.h>
> 
> u64 t = ktime_get_ns();
> /*
>  * 跨架构
>  * 单调递增
>  * 不依赖 CPU 频率
>  * 比 get_cycles 更安全
>  */
> ~~~

不论一个体系独立的函数是否可用, 我们最好利用机会来展示一个内联汇编代码的例子。为此, 我们实现一个 `rdtscl` 函数给 `MIPS` 处理器, 它与在 `x86` 上同样的方式工作。拖尾的 `nop` 指令被要求来阻止编译器在 `mfc0` 之后马上存取指令中的目标寄存器。这种内部锁在 `RISC` 处理器中是典型的, 并且编译器仍然可以在延迟时隙中调度有用的指令。在这个情况中, 我们使用 `nop` 因为内联汇编对编译器是一个黑盒并且不会进行优化。

~~~c
#define rdtscl(dest) \
__asm__ __volatile__("mfc0 %0,$9; nop" : "=r" (dest))
~~~

有这个宏在, MIPS 处理器可以执行同样的代码, 如同前面为 `x86` 展示的一样的代码。使用 `gcc` 内联汇编, 通用寄存器的分配留给编译器。刚刚展示的这个宏使用 `%0` 作为"参数 `0`"的一个占位符, 之后它被指定为"任何用作输出( `=` )的寄存器( `r` )"。这个宏还声明输出寄存器必须对应 `C` 表达式 `dest`。内联函数的语法是非常强大但是有些复杂, 特别对于那些有限制每个寄存器可以做什么的体系上(就是说, `x86` 家族)。这个用法在 `gcc`文档中描述, 常常在 `info` 文档目录树中有。

本节已展示的这个简短的 C-代码片段已在一个 K7-级 `x86` 处理器 和一个 `MIPS VR4181`( 使用刚刚描述过的宏 )上运行。前者报告了一个 11 个时钟嘀哒的时间流失而后者只是2 个时钟嘀哒。小的数字是期望的, 因为 RISC 处理器常常每个时钟周期执行一条指令。有另一个关于时戳计数器的事情值得知道: 它们在一个 SMP 系统中不必要跨处理器同步。为保证得到一个一致的值, 你应当为查询这个计数器的代码禁止抢占。



## 7.2 获知当前时间

内核代码能一直获取一个当前时间的表示, 通过查看 jifies 的值。常常地, 这个值只代表从最后一次启动以来的时间, 这个事实对驱动来说无关, 因为它的生命周期受限于系统的 uptime。如所示, 驱动可以使用 jiffies 的当前值来计算事件之间的时间间隔(例如,在输入驱动中从单击中区分双击或者计算超时)。简单地讲, 查看 jiffies 几乎一直是足
够的, 当你需要测量时间间隔。如果你需要对短时间流失的非常精确的测量, 处理器特定的寄存器来帮忙了( 尽管它们带来严重的移植性问题 )。

一个驱动会需要知道墙上时钟时间，但是这是非常不可能（以月, 天, 和小时来表达的）; 这个信息常常只对用户程序需要, 例如 cron 和 syslogd。处理真实世界的时间常常最好留给用户空间, 那里的 C 库提供了更好的支持; 另外, 这样的代码常常太策略相关以至于不属于内核。有一个内核函数转变一个墙上时钟时间到一个 jiffies 值, 但是:

~~~c
#include <linux/time.h>
unsigned long mktime (unsigned int year, unsigned int mon,
unsigned int day, unsigned int hour,
unsigned int min, unsigned int sec);
~~~

**重复:直接在驱动中处理墙上时钟时间往往是一个在实现策略的信号, 并且应当因此而被置疑。**

虽然你不会一定处理人可读的时间表示, 有时你需要甚至在内核空间中处理绝对时间。为此, `<linux/time.h>` 输出了 `do_gettimeofday` 函数。当被调用时, 它填充一个 `struct timeval` 指针 -- 和在 `gettimeofday` 系统调用中使用的相同 -- 使用类似的秒和毫秒值。`do_gettimeofday` 的原型是:

~~~c
#include <linux/time.h>
void do_gettimeofday(struct timeval *tv);
~~~

这段源代码声明 `do_gettimeofday` 有" 接近毫秒的精度", 因为它询问时间硬件当前`jiffy` 多大比例已经流失。这个精度每个体系都不同。但是, 因为它依赖实际使用中的硬件机制。例如, 一些 `m68knommu` 处理器, `Sun3` 系统, 和其他 `m68k` 系统不能提供大于`jiffy` 的精度。`Pentium` 系统, 另一方面, 提供了非常快速和精确的小于嘀哒的测量, 通过读取本章前面描述的时戳计数器。

当前时间也可用( 尽管使用 `jiffy` 的粒度 )来自 `xtime` 变量, 一个 `struct timespec` 值。不鼓励这个变量的直接使用, 因为难以原子地同时存取这 2 个字段。因此, 内核提供了实用函数 `current_kernel_time`:

~~~c
#include <linux/time.h>
struct timespec current_kernel_time(void);
~~~

用来以各种方式获取当前时间的代码, 可以从由 O' Reilly 提供的 FTP 网站上的源码文件的 `jit` ("just in time") 模块获得。 `jit` 创建了一个文件称为 `/proc/currentime`, 当读取时, 它以 ASCII 码返回下列项:（`/proc/currentime` 不是标准 Linux 内核里的文件，标准的Linux系统中默认不纯在该文件。）

* 当前的 `jiffies` 和 `jiffies_64` 值, 以 `16 进制数`的形式。
* 如同 `do_gettimeofday` 返回的相同的当前时间。
*  由 `current_kernel_time` 返回的 `timespec`。

我们选择使用一个动态的 `/proc` 文件来保持样板代码为最小 -- 它不值得创建一整个设备只是返回一点儿文本信息。这个文件连续返回文本行只要这个模块加载着; 每次 read 系统调用收集和返回一套数据,为更好阅读而组织为 2 行。无论何时你在少于一个时钟嘀哒内读多个数据集, 你将看到 `do_gettimeofday `之间的差别, 它询问硬件, 并且其他值仅在时钟嘀哒时被更新。

-----------

以上为书中内容，现代Linux内核已经优化相关函数和数据结构，在较新的 Linux 内核（5.x / 6.x）中：

~~~c
#include <linux/time.h>
unsigned long mktime(unsigned int year, unsigned int mon,
                     unsigned int day, unsigned int hour,
                     unsigned int min, unsigned int sec);

~~~

> * 仍然存在（兼容性原因）
> * 但**不推荐在新代码中使用**
> * 返回的是 `unsigned long`
> * 有 **2038 年问题风险（32bit 系统）** 

以上为书中内容，现代Linux内核已经废弃：

* 获取当前 wall-clock 时间

~~~c
/*
 * void do_gettimeofday(struct timeval *tv);
 * 已经废弃 不允许新代码使用 现代替换方法如下：
 */
time64_t
struct timespec64
struct ktime
#include <linux/timekeeping.h>

struct timespec64 ts;
ktime_get_real_ts64(&ts);
/* 如果只需要 秒 */
time64_t now = ktime_get_real_seconds();

~~~

* 获取当前内核时间

~~~c
/*
 * struct timespec current_kernel_time(void);
 * 已经废弃 不允许新代码使用 现代替换方法如下：
 */
struct timespec64 ts;
ktime_get_real_ts64(&ts);
/* 或者（monotonic 时间） */
ktime_get_ts64(&ts);

~~~

* 用法示例

~~~c
#include <linux/timekeeping.h>

static void print_time(void)
{
    struct timespec64 ts;

    ktime_get_real_ts64(&ts);

    pr_info("time: %lld.%09ld\n",
            (long long)ts.tv_sec,
            ts.tv_nsec);
}

~~~

-----------



## 7.3 延后执行

设备驱动常常需要延后一段时间执行一个特定片段的代码, 常常允许硬件完成某个任务。在这一节我们涉及许多不同的技术来获得延后。每种情况的环境决定了使用哪种技术最好;我们全都仔细检查它们, 并且指出每一个的长处和缺点。

一件要考虑的重要的事情是你需要的延时如何与时钟嘀哒比较, 考虑到 HZ 的跨各种平台的范围。那种可靠地比时钟嘀哒长并且不会受损于它的粗粒度的延时, 可以利用系统时钟。每个短延时典型地必须使用软件循环来实现。在这 2 种情况中存在一个灰色地带。在本章, 我们使用短语" long " 延时来指一个多 jiffy 延时, 在一些平台上它可以如同几个毫秒一样少, 但是在 CPU 和内核看来仍然是长的。

下面的几节讨论不同的延时, 通过采用一些长路径, 从各种直觉上不适合的方法到正确的方法。我们选择这个途径因为它允许对内核相关定时方面的更深入的讨论。如果你急于找出正确的代码, 只要快速浏览本节。

### 7.3.1 长时延

偶尔地, 一个驱动需要延后执行相对长时间 -- 多于一个时钟嘀哒。有几个方法实现这类延时; 我们从最简单的技术开始, 接着进入到高级些的技术。

#### 7.3.1.1 忙等待

==（以下内容都是基于2.x的旧内核，现代Linux内核中大多数已经弃用。）== 

如果你想延时执行多个时钟嘀哒, 允许在值中某些疏忽, 最容易的( 尽管不推荐 ) 的实现是一个监视 `jiffy` 计数器的循环。这种忙等待实现常常看来象下面的代码, 这里` j1`是 `jiffies` 的在延时超时的值:

~~~c
while (time_before(jiffies, j1))
	cpu_relax();
~~~

对 cpu_relex 的调用使用了一个特定于体系的方式来说, 你此时没有在用处理器做事情.在许多系统中它根本不做任何事; 在对称多线程(" 超线程" ) 系统中, 可能让出核心给其他线程。在如何情况下, 无论何时有可能, 这个方法应当明确地避免。我们展示它是因为偶尔你可能想运行这个代码来更好理解其他代码的内幕。

我们来看一下这个代码如何工作。这个循环被保证能工作因为 `jiffies` 被内核头文件声明做易失性的, 并且因此, 在任何时候 C 代码寻址它时都从内存中获取。尽管技术上正确( 它如同设计的一样工作 ), 这种忙等待严重地降低了系统性能。如果你不配置你的内核为抢占操作, 这个循环在延时期间完全锁住了处理器; 调度器永远不会抢占一个在内核中运行的进程, 并且计算机看起来完全死掉直到时间 ` j1` 到时。这个问题如果你运行一个可抢占的内核时会改善一点, 因为, 除非这个代码正持有一个锁, 处理器的一些时间可以被其他用途获得。但是, 忙等待在可抢占系统中仍然是昂贵的。

更坏的是, 当你进入循环时如果中断碰巧被禁止, `jiffies` 将不会被更新, 并且 `while` 条件永远保持真。运行一个抢占的内核也不会有帮助, 并且你将被迫去击打大红按钮。这个延时代码的实现可拿到, 如同下列的, 在 `jit` 模块中。模块创建的这些 `/proc/jit*` 文件每次你读取一行文本就延时一整秒, 并且这些行保证是每个 20 字节。如果你想测试忙等待代码, 你可以读取 `/proc/jitbusy`, 每当它返回一行它忙-循环一秒。为确保读, 最多, 一行( 或者几行 ) 一次从 `/proc/jitbusy`。 简化的注册 `/proc` 文件的内核机制反复调用 read 方法来填充用户请求的数据缓存。因此, 一个命令, 例如 `cat /proc/jitbusy`, 如果它一次读取 4KB, 会冻住计算机 205 秒。

推荐的读 `/proc/jitbusy` 的命令是 `dd bs=200 < /proc/jitbusy`, 可选地同时指定块数目. 文件返回的每 20-字节 的行表示 `jiffy` 计数器已有的值, 在延时之前和延时之后。

----------------

**现代Linux内核关于忙等待不在推荐使用 jiffies 轮询延时**，现在更推荐使用：

- `udelay()`   -  短延时（< 10us）
- `ndelay()`
- `mdelay()`（慎用）
- `usleep_range()`  -   微秒~毫秒级
- `fsleep()`（6.x 新接口 单位ns）

`fsleep()` 是新的统一延时接口，自动选择最佳策略。**现代内核基本不再推荐手写 jiffies busy wait。**

==**现代内核严禁长时延时忙等！！！**==

> “长时延”一般指：
>
> - ≥ 10ms
> - 可能等待 100ms ~ 秒级
> - 可能等待硬件响应
> - 可能等待用户空间
> - 可能等待网络 / DMA / 外设
>
> 这种情况 **绝对不能 busy wait**。

现代内核对长时延的核心原则：

1. **不能占 CPU，绝对禁止**：

~~~c
while (!done)
    cpu_relax();
~~~

在 6.x 内核上，这种代码在 SMP + PREEMPT 下会严重影响系统。

2. **必须可调度，线程必须进入**：

- TASK_INTERRUPTIBLE
- TASK_UNINTERRUPTIBLE
- TASK_KILLABLE

3. **必须支持超时，防止**：

- 硬件死机
- 永久挂死
- 用户无法 ctrl+c

--------------------------

**现代 Linux 6.x 推荐长时延模型**

**模型 1：wait_event_timeout（最常用）**

这是现在驱动的标准模型。

```c
ret = wait_event_interruptible_timeout(
        dev->wq,
        dev->done,
        msecs_to_jiffies(5000));
```

特点：

- 不占 CPU
- 可被唤醒
- 支持信号
- 支持超时
- 与 poll/epoll 兼容

👉 这是现代驱动首选方案。

**模型 2：completion（更语义化）**

现代内核大量使用 completion。

```c
struct completion done;

init_completion(&done);

ret = wait_for_completion_interruptible_timeout(
        &done,
        msecs_to_jiffies(5000));
```

中断里：

```c
complete(&done);
```

优点：

- 代码清晰
- 不需要自己写 condition
- 单次事件语义更强
- 几乎所有新驱动都用

👉 比 LDD3 的 wait_event 更“现代”。

**模型 3：schedule_timeout（少用）**

如下文说些的超时方法仍然存在：

```c
set_current_state(TASK_INTERRUPTIBLE);
schedule_timeout(msecs_to_jiffies(5000));
```

但：

❌ 不绑定具体事件 ❌ 不优雅 ❌ 不好维护

**现在更推荐 wait_event/completion。**

==**现代推荐“长时延最佳实践结构”**==

这是 6.x 内核驱动最标准结构：

```c
/* 1. 初始化 */
init_waitqueue_head(&dev->wq);
dev->done = false;

/* 2. 启动硬件 */
start_hw();

/* 3. 等待完成 */
ret = wait_event_interruptible_timeout(
        dev->wq,
        dev->done,
        msecs_to_jiffies(3000));

if (ret == 0)
    return -ETIMEDOUT;

/* 4. 中断处理 */
irq_handler()
{
    dev->done = true;
    wake_up(&dev->wq);
}
```

**这就是现代驱动标准模式。** 

----------

#### 7.3.1.2 让出处理器

如我们已见到的, 忙等待强加了一个重负载给系统总体; 我们乐意找出一个更好的技术。想到的第一个改变是明确地释放 CPU 当我们对其不感兴趣时。这是通过调用调度函数而实现地, 在 `<linux/sched.h>` 中声明:

~~~c
while (time_before(jiffies, j1)) {
	schedule();
}
~~~

这个循环可以通过读取 `/proc/jitsched` 如同我们上面读 `/proc/jitbusy` 一样来测试。但是, 还是不够优化。当前进程除了释放 CPU 不作任何事情, 但是它保留在运行队列中.如果它是唯一的可运行进程, 实际上它运行( 它调用调度器来选择同一个进程, 进程又调用调度器, 这样下去)。换句话说, 机器的负载( 在运行的进程的平均数 ) 最少是 1, 并且空闲任务 ( 进程号 0, 也称为对换进程, 由于历史原因) 从不运行。尽管这个问题可能看来无关, 在计算机是空闲时运行空闲任务减轻了处理器工作负载, 降低它的温度以及提高它的生命期, 同时电池的使用时间如果这个计算机是你的膝上机。更多的, 因为进程实际上在延时中执行, 它所耗费的时间都可以统计。`/proc/jitsched` 的行为实际上类似于运行 `/proc/jitbusy` 在一个抢占的内核下。

-------------

**现代Linux内核有更多让出处理器的方法**：

**一、最基础：cond_resched()（现代推荐的“轻量让步”）**

⭐⭐⭐⭐⭐ 现代内核首选（循环中）

```c
cond_resched();
```

作用：

- 如果需要调度（`TIF_NEED_RESCHED`），就调用 `schedule()`
- 如果不需要，就立即返回

适合：

- 长循环
- 扫描大链表
- 遍历大数组
- 非阻塞逻辑中偶尔让步

示例：

```c
while (node) {
    process(node);
    cond_resched();
    node = node->next;
}
```

✔ 不改变 task state ✔ 不 sleep ✔ 不影响逻辑 ✔ 非常安全

👉 这是现代替代 LDD3 里 `schedule()` 循环的方式。

-----------

**二、schedule() —— 直接让出**

```c
schedule();
```

作用：

- 无条件让出 CPU
- 进入调度器

**⚠️ 现代不推荐裸用**

因为：

- 不设置 `task state`
- 可能立刻被调度回来
- 行为不可预测

现代一般不会单独使用。

-----------

**三、schedule_timeout() —— 带超时让出**

```c
set_current_state(TASK_INTERRUPTIBLE);
schedule_timeout(timeout);
// 或：
schedule_timeout_interruptible(timeout);
```

作用：

- 设置 task state
- 进入调度
- 最多等待 timeout

适合：

- 主动 sleep
- 延时等待

**但现在更推荐：**

- wait_event_timeout()
- completion

---------------

**四、yield 相关 API**

1. **yield()**

```c
yield();
```

内部调用 `schedule()` ⚠️ 基本不推荐

因为：

- 语义模糊
- 行为依赖调度器
- 在 CFS 下几乎没意义

Linux 内核维护者长期不推荐使用 yield。

2. **cond_resched_lock()**

用于持有锁时：

```c
cond_resched_lock(&lock);
```

适合：

- 长时间持有 `spinlock`（极少情况）
- 复杂内核子系统内部

-----------------

**五、sleep 系列（本质是让出 CPU）**

**六、wait_event —— 事件驱动式让出**

这是现代驱动主流：

```c
wait_event_interruptible(wq, condition);
```

本质：

- 设置 TASK_INTERRUPTIBLE
- schedule()
- 被 wake_up() 唤醒

这是“带条件的让出调度器”。

-----------

**七、completion —— 更现代的事件等待**

```c
wait_for_completion(&done);
```

内部也是 `schedule()` 但语义更清晰。

----------------

**不同场景推荐总结**

| 场景             | 推荐方法             |
| ---------------- | -------------------- |
| 长循环中偶尔让步 | ⭐ `cond_resched()`   |
| 等待条件         | ⭐ `wait_event`       |
| 等待单次事件     | ⭐ `completion`       |
| 需要超时         | `wait_event_timeout` |
| 主动 sleep       | `usleep_range`       |
| 延迟毫秒级       | `msleep`             |
| 不推荐           | `yield()`            |
| 更不推荐         | 裸 `schedule()`      |

--------------



#### 7.3.1.3 超时

目前为止所展示的次优化延时循环通过查看 `jiffy` 计数器而不告诉任何人来工作。但是最好实现一个延时的方法，通常是请求内核来完成。有两种方法来建立一个基于 `jiffy` 的超时，依赖于是否你的驱动在等待其他的事件。

如果你的驱动使用一个等待队列来**等待某些其他事件**, 但是你也想确保它在一个确定时间段内运行, 可以使用 `wait_event_timeout` 或者 `wait_event_interruptible_timeout`:

~~~c
#include <linux/wait.h>
long wait_event_timeout(wait_queue_head_t q, condition, long timeout);
long wait_event_interruptible_timeout(wait_queue_head_t q, condition, long timeout);
~~~

这些函数在给定队列上睡眠, 但是它们在超时(以 `jiffies` 表示)到后返回。因此, 它们实现一个限定的睡眠不会一直睡下去。注意超时值表示要等待的 `jiffies` 数, 不是一个绝对时间值。这个值由一个有符号的数表示, 因为它有时是一个相减运算的结果, 尽管这些函数如果提供的超时值是负值通过一个 printk 语句抱怨。如果超时到, 这些函数返回 0; 如果这个进程被其他事件唤醒, 它返回以 `jiffies` 表示的剩余超时值。返回值从不会是负值, 甚至如果延时由于系统负载而比期望的值大。

`/proc/jitqueue` 文件展示了一个基于 wait_event_interruptible_timeout 的延时, 结果这个模块没有事件来等待, 并且使用 0 作为一个条件:

~~~c
wait_queue_head_t wait;
init_waitqueue_head (&wait);
wait_event_interruptible_timeout(wait, 0, delay);
//当读取 /proc/jitqueue 时, 观察到的行为近乎优化的, 即便在负载下:
/*
phon% dd bs=20 count=5 < /proc/jitqueue
2027024 2028024
2028025 2029025
2029026 2030026
2030027 2031027
2031028 2032028
*/
~~~

因为读进程当等待超时( 上面是 dd )不在运行队列中, 你看不到表现方面的差别, 无论代码是否运行在一个抢占内核中。

`wait_event_timeout` 和 `wait_event_interruptible_timeout` 被设计为有硬件驱动存在,这里可以用任何一种方法来恢复执行: 或者有人调用 `wake_up` 在等待队列上, 或者超时到。这不适用于 `jitqueue`, 因为没人在等待队列上调用 `wake_up` ( 毕竟, 没有其他代码知道它 ), 因此这个进程当超时到时一直唤醒。为适应这个特别的情况, 这里你**想延后执行不等待特定事件**, 内核提供了 `schedule_timeout` 函数, 因此你可以避免声明和使用一个多余的等待队列头:

~~~c
#include <linux/sched.h>
signed long schedule_timeout(signed long timeout);
~~~

这里, `timeout` 是要延时的 `jiffies` 数。返回值是 0 除非这个函数在给定的 `timeout` 流失前返回(响应一个信号)。`schedule_timeout` 请求调用者首先设置当前的进程状态,因此一个典型调用看来如此:

~~~c
set_current_state(TASK_INTERRUPTIBLE);
schedule_timeout (delay);
~~~

前面的行( 来自 `/proc/jitschedto` ) 导致进程睡眠直到经过给定的时间。因为 `ait_event_interruptible_timeout` 在内部依赖 `schedule_timeout`, 我们不会费劲显示 `jitschedto` 返回的数, 因为它们和 `jitqueue` 的相同。再一次, 不值得有一个额外的时间间隔在超时到和你的进程实际被调度来执行之间。

在刚刚展示的例子中, 第一行调用 `set_current_state` 来设定一些东西以便调度器不会再次运行当前进程, 直到超时将它置回 `TASK_RUNNING` 状态. 为获得一个不可中断的延时,使用 `TASK_UNINTERRUPTIBLE` 代替。如果你忘记改变当前进程的状态, 调用`schedule_time` 如同调用 `shcedule`( 即, `jitsched` 的行为), 建立一个不用的定时器。

如果你想使用这 4 个 `jit` 文件在不同的系统情况下或者不同的内核, 或者尝试其他的方式来延后执行, 你可能想配置延时量当加载模块时通过设定延时模块参数。

---------

**现代Linux内核中上述这类超时 API —— 仍然是现代驱动主流写法**

✔ 现在依然推荐 ✔ 内核内部已经优化 ✔ 与 epoll/poll 完美配合

示例（现代标准写法）：

```c
ret = wait_event_interruptible_timeout(
        my_wq,
        device_ready,
        msecs_to_jiffies(5000));
```

这是 **当前驱动开发的标准模式**。

-----------------



### 7.3.2 短延时

当一个设备驱动需要处理它的硬件的反应时间, 涉及到的延时常常是最多几个毫秒。在这个情况下, 依靠时钟嘀哒显然不对的。

> The kernel functions ndelay, udelay, and mdelay serve well for short delays, delaying execution for the specified number of nanoseconds, microseconds, or milliseconds respectively.* Their prototypes are: * The u in udelay represents the Greek letter mu and stands for micro.

内核函数 `ndelay`, `udelay`, 以及 `mdelay` 对于短延时好用, 分别延后执行指定的纳秒数, 微秒数或者毫秒数。它们的原型是: （udelay 中的 u 表示希腊字母 mu 并且代表 micro）。

~~~c
#include <linux/delay.h>
void ndelay(unsigned long nsecs);
void udelay(unsigned long usecs);
void mdelay(unsigned long msecs);
~~~

这些函数的实际实现在 `<asm/delay.h>`, 是体系特定的, 并且有时建立在一个外部函数上。每个体系都实现 `udelay`, 但是其他的函数可能或者不可能定义; 如果它们没有定义, `<linux/delay.h>` 提供一个缺省的基于 `udelay` 的版本. 在所有的情况中, 获得的延时至少是要求的值, 但可能更多; 实际上, 当前没有平台获得了纳秒的精度, 尽管有几个提供了次微秒的精度. 延时多于要求的值通常不是问题, 因为驱动中的短延时常常需要等待硬件, 并且这个要求是等待至少一个给定的时间流失。

`udelay` 的实现( 可能 ndelay 也是) 使用一个软件循环基于在启动时计算的处理器速度,使用整数变量 `loos_per_jiffy`。如果你想看看实际的代码, 但是, 小心 x86 实现是相当复杂的一个因为它使用的不同的时间源, 基于什么 CPU 类型在运行代码。为避免在循环计算中整数溢出, `udelay` 和 `ndelay` 强加一个上限给传递给它们的值。如果你的模块无法加载和显示一个未解决的符号, `__bad_udelay`, 这意味着你使用太大的参数调用 ```udleay```。注意, 但是, 编译时检查只对常量进行并且不是所有的平台实现它。作为一个通用的规则, 如果你试图延时几千纳秒, 你应当使用 `udelay` 而不是 `ndelay`; 类似地, 毫秒规模的延时应当使用 `mdelay` 完成而不是一个更细粒度的函数。

重要的是记住这 3 个延时函数是忙等待; 其他任务在时间流失时不能运行。因此, 它们重复, 尽管在一个不同的规模上, `jitbusy` 的做法。因此, 这些函数应当只用在没有实用的替代时。有另一个方法获得毫秒(和更长)延时而不用涉及到忙等待。文件 `<linux/delay.h>` 声明这些函数:

~~~c
void msleep(unsigned int millisecs);
unsigned long msleep_interruptible(unsigned int millisecs);
void ssleep(unsigned int seconds)
~~~

前 2 个函数使调用进程进入睡眠给定的毫秒数。一个对 `msleep` 的调用是不可中断的;你能确保进程睡眠至少给定的毫秒数。如果你的驱动位于一个等待队列并且你想唤醒来打断睡眠, 使用 `msleep_interruptible`。从 `msleep_interruptible` 的返回值正常地是 0;如果, 但是, 这个进程被提早唤醒, 返回值是在初始请求睡眠周期中剩余的毫秒数。对`ssleep` 的调用使进程进入一个不可中断的睡眠给定的秒数。通常, 如果你能够容忍比请求的更长的延时, 你应当使用 `schedule_timeout`, `msleep`, 或者 `ssleep`。

-------------------

**对于现代Linux内核： **

**一、loops_per_jiffy 描述 —— ⚠️ 已经过时** 

原文说：

> 使用 `loops_per_jiffy` 计算的软件循环

在早期内核确实如此。

**但在现代内核（特别是 x86, arm64）：** 

- 不再单纯依赖 `loops_per_jiffy`
- 会优先使用 `TSC`
- 或 `arch counter`
- 或高精度 `timer`
- 或 `calibrated delay loop`

尤其在 `x86` 上，实现远比 `LDD3` 时代复杂。

所以这句：

> `udelay` 基于 `loops_per_jiffy`

⚠️ 现在只能说“部分平台如此”，不能作为通用描述。

-----------

**二、__bad_udelay 描述 —— ⚠️ 基本过时**

你写到：

> 如果出现未解决符号 `__bad_udelay`，说明参数过大

这个机制确实存在过。

但在现代内核：

- 很多架构已经不再使用这个符号
- 编译器优化也不同
- 不一定还能看到这个错误

**所以这段属于“历史机制”。** 

----------------

**三、关于“毫秒级用 mdelay” —— ⚠️ 现代不推荐**

原文建议：

> 毫秒规模的延时应当使用 `mdelay`

⚠️ 这在现代内核中是**不推荐的**。

因为：

```c
mdelay(100);
```

会 `busy wait` 100ms —— 这是灾难。现代规则：

| 延时长度    | 推荐方式       |
| ----------- | -------------- |
| < 10us      | `udelay`       |
| 10us ~ 20ms | `usleep_range` |
| > 20ms      | `msleep`       |

`mdelay` 只适用于：

- 早期启动代码
- 原子上下文
- 非常短毫秒级

**在驱动中一般禁止使用 mdelay。**

-----------

**四、关于 msleep / msleep_interruptible —— 基本正确**

描述是正确的：

```c
void msleep(unsigned int);
unsigned long msleep_interruptible(unsigned int);
void ssleep(unsigned int);
```

但现代建议：

⚠️ msleep 精度低，内核 tick 影响较大。

现代更推荐：

```c
usleep_range(min, max);
```

原因：

- 支持 hrtimer、更省电、更精确、调度器更友好

----------------

**五、现代 6.x 新变化（原文未提及）**

**1️⃣ 新接口：fsleep()**

6.x 新统一延时接口：

```c
fsleep(u64 nsecs);
```

自动选择：

- `busy wait`、`schedule`、`hrtimer`

更智能。

------

**2️⃣ tickless 内核（NO_HZ）**

现代系统可能无 `tick`。

这会影响：

- `msleep` 精度、`schedule_timeout` 行为

LDD3 时代没有这个问题。

------

**3️⃣ PREEMPT_RT**

在 RT 内核中：

- 忙等待更危险、`mdelay` 几乎不能用

------

**六、关于“依靠时钟嘀嗒显然不对”这句话**

原文说：

> 短延时不能依赖时钟嘀嗒

在 2.6 时代是对的。

但现在：

- `hrtimer` 已经支持微秒级精度
- `tick` 不再是唯一时间源

所以现代说法应该是：

> 不应依赖 `jiffies`，但可以使用 `hrtimer` 或 `usleep_range`

------

**七、现代 6.x 修正版总结**

如果你要把这段改写为“现代内核版本”，应该这样描述：

**现代短延时原则**

1. `ndelay`/`udelay`/`mdelay` 仍然是 `busy-wait`
2. 不应在进程上下文使用` mdelay`
3. 推荐使用 `usleep_range` 替代毫秒级 `busy-wait`
4. 对于较长延时应使用 `sleep` 接口
5. 延时实现已不再单纯依赖 ```loops_per_jiffy```
6. `__bad_udelay` 已不是通用机制

------

**九、简洁判断**

| 结论                         | 是否正确   |
| ---------------------------- | ---------- |
| 这段内容符合 LDD3            | ✅          |
| 在现代 6.x 完全正确          | ❌          |
| 需要补充 `usleep_range`      | ✅          |
| `mdelay` 仍然推荐            | ❌          |
| `loops_per_jiffy `是核心机制 | ⚠️ 部分平台 |
| `__bad_udelay` 仍普遍存在    | ❌          |

--------------



## 7.4 内核定时器

无论何时你需要调度一个动作以后发生, 而不阻塞当前进程直到到时, 内核定时器是给你的工具。这些定时器用来调度一个函数在将来一个特定的时间执行, 基于时钟嘀哒, 并且可用作各类任务; 例如, 当硬件无法发出中断时, 查询一个设备通过在定期的间隔内检查它的状态。其他的内核定时器的典型应用是关闭软驱马达或者结束另一个长期终止的操作。在这种情况下, 来自 `close` 的延后返回将强加一个不必要(并且吓人的)开销在应用程序上。最后, 内核自身使用定时器在几个情况下, 包括实现 `schedule_timeout`。

一个内核定时器是一个数据结构, 它指导内核执行一个用户定义的函数使用一个用户定义的参数在一个用户定义的时间. 这个实现位于 `<linux/timer.h>` 和 `kernel/timer.c` 并且在"内核定时器"一节中详细介绍。

被调度运行的函数几乎确定不会在注册它们的进程在运行时运行。它们是, 相反, 异步运行。直到现在, 我们在我们的例子驱动中已经做的任何事情已经在执行系统调用的进程上下文中运行。当一个定时器运行时, 但是, 这个调度进程可能睡眠, 可能在不同的一个处理器上运行, 或者很可能已经一起退出。

这个异步执行类似当发生一个硬件中断时所发生的( 这在第 10 章详细讨论 )。实际上,内核定时器被作为一个"软件中断"的结果而实现。当在这种原子上下文运行时, 你的代码易受到多个限制。定时器函数必须是原子的以所有的我们在第 1 章"自旋锁和原子上下文"一节中曾讨论过的方式, 但是有几个附加的问题由于缺少一个进程上下文而引起的. 我们将介绍这些限制; 在后续章节的几个地方将再次看到它们。循环被调用因为原子上下文的规则必须认真遵守, 否则系统会发现自己陷入大麻烦中。

为能够被执行, 多个动作需要进程上下文。当你在进程上下文之外(即, 在中断上下文),你必须遵守下列规则:

* 没有允许存取用户空间。因为没有进程上下文, 没有和任何特定进程相关联的到用户空间的途径。
* 这个 current 指针在原子态没有意义, 并且不能使用因为相关的代码没有和已被中断的进程的联系。
* 不能进行睡眠或者调度。原子代码不能调用 `schedule` 或者某种 `wait_event`, 也不能调用任何其他可能睡眠的函数。例如, 调用 `kmalloc`(`...`, `GFP_KERNEL`) 是违犯规则的. 旗标也必须不能使用因为它们可能睡眠。

内核代码能够告知是否它在中断上下文中运行, 通过调用函数 `in_interrupt()`, 它不要参数并且如果处理器当前在中断上下文运行就返回非零, 要么硬件中断要么软件中断。

一个和 `in_interrupt()` 相关的函数是 `in_atomic()`。它的返回值是非零无论何时调度被禁止; 这包含硬件和软件中断上下文以及任何持有自旋锁的时候。在后一种情况,`current` 可能是有效的, 但是存取用户空间被禁止, 因为它能导致调度发生。无论何时你使用 `in_interrupt()`, 你应当真正考虑是否 `in_atomic` 是你实际想要的。2 个函数都在`<asm/hardirq.h>` 中声明。

内核定时器的另一个重要特性是一个任务可以注册它本身在后面时间重新运行。这是可能的, 因为每个 `timer_list` 结构在运行前从激活的定时器链表中去连接, 并且因此能够马上在其他地方被重新连接。尽管反复重新调度相同的任务可能表现为一个无意义的操作,有时它是有用的。例如, 它可用作实现对设备的查询。

也值得了解在一个 SMP 系统, 定时器函数被注册时相同的 CPU 来执行, 为在任何可能的时候获得更好的缓存局部特性。因此, 一个重新注册它自己的定时器一直运行在同一个CPU。

不应当被忘记的定时器的一个重要特性是, 它们是一个潜在的竞争条件的源, 即便在一个单处理器系统。这是它们与其他代码异步运行的一个直接结果。因此, 任何被定时器函数存取的数据结构应当保护避免并发存取, 要么通过原子类型( 在第 1 章的"原子变量"一节) 要么使用自旋锁( 在第 9 章讨论 )。

---------------------

**现代 Linux 5.x / 6.x** 的角度看，确实有不少重要差异和演进点。

下面是按“哪些仍然正确 / 哪些已变化 / 现代新增机制”系统对比。

------

**一、总体评价**

| 项目             | 现在是否仍然成立    |
| ---------------- | ------------------- |
| 定时器是异步执行 | ✅                   |
| 运行在原子上下文 | ✅                   |
| 不能睡眠         | ✅                   |
| 不能访问用户空间 | ✅                   |
| 需要自旋锁保护   | ✅                   |
| 基于 jiffies     | ⚠️ 部分正确          |
| 属于软中断机制   | ⚠️ 表述已变化        |
| 主要延迟机制     | ❌ 已被 hrtimer 分流 |

------

**二、最重要的现代差异**

**1️⃣ timer_list API 已改变（重大变化）**

LDD3 时代写法：

```c
init_timer(&timer);
timer.function = func;
timer.data = data;
timer.expires = jiffies + HZ;
add_timer(&timer);
```

⚠️ 现代内核（4.15+）已经废弃这种写法。

现在必须使用：

```c
timer_setup(&timer, func, 0);
mod_timer(&timer, jiffies + HZ);
```

并且回调函数签名变为：

```c
void func(struct timer_list *t);
```

而不是：

```c
void func(unsigned long data);
```

这是一个**非常重要的 API 变化**。

------

**2️⃣ 不再强调“软件中断”说法**

LDD3 说：

> 定时器作为“软件中断”实现

在现代内核中：

- timer_list 回调运行在 softirq 上下文（`TIMER_SOFTIRQ`）
- 但内核已经重构 softirq 处理方式
- 并且 `PREEMPT_RT` 下行为更复杂

所以：

✔ 本质仍然是 softirq
 ⚠️ 但实现细节比 LDD3 复杂很多

------

**3️⃣ hrtimer 的大量使用（最大结构变化）**

LDD3 时代：

- 只有 jiffies 定时器

现代内核：

- timer_list → 低精度（jiffies）
- hrtimer → 高精度（纳秒级）

如果需要：

- 微秒级、高精度周期性、精确定时

必须用：

```c
#include <linux/hrtimer.h>
```

而不是 `timer_list`。**这是现代最关键差异。**

------

**4️⃣ NO_HZ / tickless 影响**

LDD3 时代默认周期 `tick`。

现代：

- 支持 `NO_HZ`
- 支持动态 `tick`
- 支持 `full tickless`

因此：

> 定时器仍然基于 jiffies，但 tick 可能不是周期性的。

实现机制已经比书中复杂。

------

5️⃣ SMP 行为描述已经不准确

书中说：

> 定时器在注册它的 CPU 上运行

⚠️ 现代内核不再保证严格 CPU 绑定。

现在：

- 默认可能在任意 CPU 执行
- 如果需要绑定 CPU，要使用：
  - `add_timer_on()`

**但一般不建议依赖 CPU 绑定。**

------

**6️⃣ in_interrupt() / in_atomic() 仍然存在**

这部分仍然正确。

但现代建议：

- 很少直接用 `in_interrupt()`
- 更推荐根据 API 语义设计代码
- 使用 `might_sleep()` 调试

------

**三、现代新增机制（LDD3 未涵盖）**

**1️⃣ deferrable timer**

```c
timer_setup(&timer, func, TIMER_DEFERRABLE);
```

作用：

- 系统 idle 时不强制唤醒 CPU、更省电

LDD3 没有这个。

------

**2️⃣ pinned timer**

```c
add_timer_on(&timer, cpu);
```

控制 CPU 亲和性。

------

**3️⃣ timer_shutdown()**

新接口（避免 use-after-free）：

```c
timer_shutdown_sync(&timer);
```

比 del_timer_sync 更安全。

------

**4️⃣ 高精度 hrtimer 子系统**

现代驱动中：

- 网络栈、高精度调度、实时系统

大量使用 `hrtimer`。

`timer_list` 现在主要用于：

- 低精度、容忍误差、内核子系统维护

------

**四、关于“不能睡眠”——现代更严格**

在 `PREEMPT_RT` 下：

- softirq 可能运行在 `ksoftirqd` 线程中
- 但 `timer callback` 仍然不能睡眠

规则没有变，甚至更严格。

------

**五、现代推荐使用场景对比**

| 场景          | 推荐机制       |
| ------------- | -------------- |
| ≥1ms 容忍误差 | `timer_list`   |
| 微秒级精度    | `hrtimer`      |
| 等待事件      | `wait_event`   |
| 单次完成      | `completion`   |
| 延时 sleep    | `usleep_range` |

现在：

> timer_list 主要用于“内核内部维护”
>  而驱动更多使用 `wait_event` + `hrtimer`

------

**六、结构层面的核心变化**

LDD3 时代模型：

```
timer → softirq → callback
```

现代模型：

```
timer wheel（层级时间轮）
hrtimer red-black tree
NO_HZ 动态 tick
SMP per-CPU timer base
```

内部复杂度大幅增加。

------

七、竞争条件部分 —— 仍然完全正确

你最后一段：

> 定时器是潜在竞争源

✔ 现在仍然完全正确。

甚至：

- SMP 更多、抢占更多、并发更强

竞争问题比 LDD3 时代更严重。

------

**八、最核心差异总结**

| 项目       | LDD3          | 现代 Linux                      |
| ---------- | ------------- | ------------------------------- |
| API        | init_timer    | timer_setup                     |
| 回调参数   | unsigned long | struct timer_list *             |
| 精度       | jiffies       | jiffies + hrtimer               |
| CPU 绑定   | 默认同 CPU    | 不保证                          |
| tick       | 周期性        | 可 NO_HZ                        |
| 删除 API   | del_timer     | del_timer_sync / timer_shutdown |
| 高精度支持 | 无            | hrtimer                         |

------

**九、一句话结论**

> DDL3中这段关于内核定时器的描述在概念层面仍然正确，但 API 和实现细节已经发生明显演进。
>  最大变化是：hrtimer 的出现 + timer_setup API 重构 + tickless 支持。



### 7.4.1 定时器API

内核提供给驱动许多函数来声明, 注册, 以及去除内核定时器。下列的引用展示了基本的代码块:

~~~c
#include <linux/timer.h>
struct timer_list
{
/* ... */
unsigned long expires;
void (*function)(unsigned long);
unsigned long data;
};
void init_timer(struct timer_list *timer);
struct timer_list TIMER_INITIALIZER(_function, _expires, _data);
void add_timer(struct timer_list * timer);
int del_timer(struct timer_list * timer);
~~~

这个数据结构包含比曾展示过的更多的字段, 但是这 3 个是打算从定时器代码自身以外被存取的。这个 `expires` 字段表示定时器期望运行的 `jiffies` 值; 在那个时间, 这个`function 函数`被调用使用 `data` 作为一个参数。如果你需要在参数中传递多项, 你可以捆绑它们作为一个单个数据结构并且传递一个转换为 `unsiged long` 的指针, 在所有支持的体系上的一个安全做法并且在内存管理中相当普遍( 如同 15 章中讨论的 )。`expires`值不是一个 jiffies_64 项因为定时器不被期望在将来很久到时, 并且 `64位`操作在 `32位`平台上慢。

这个结构必须在使用前初始化。这个步骤保证所有的成员被正确建立, 包括那些对调用者不透明的。初始化可以通过调用 `init_timer` 或者 安排 `TIMER_INITIALIZER` 给一个静态结构, 根据你的需要。在初始化后, 你可以改变 3 个公共成员在调用 `add_timer` 前。为在到时前禁止一个已注册的定时器, 调用 `del_timer`。

jit 模块包括一个例子文件, `/proc/jitimer` ( 为 "`just in timer`"), 它返回一个头文件行以及 6 个数据行。这些数据行表示当前代码运行的环境; 第一个由读文件操作产生并且其他的由定时器。

如果你读 `/proc/jitimer` 当系统无负载时, 你会发现定时器的上下文是进程 0, 空闲任务, 它被称为"对换进程"只要由于历史原因。用来产生 	 数据的定时器是缺省每 `10 jiffies` 运行一次, 但是你可以在加载模块时改变这个值通过设置 `tdelay` ( `timer delay` ) 参数。

下面的代码引用展示了 jit 关于 `jitimer` 定时器的部分。当一个进程试图读取我们的文件, 我们建立这个定时器如下:

~~~c
unsigned long j = jiffies;
/* fill the data for our timer function */
data->prevjiffies = j;
data->buf = buf2;
data->loops = JIT_ASYNC_LOOPS;
/* register the timer */
data->timer.data = (unsigned long)data;
data->timer.function = jit_timer_fn;
data->timer.expires = j + tdelay; /* parameter */
add_timer(&data->timer);
/* wait for the buffer to fill */
wait_event_interruptible(data->wait, !data->loops);
The actual timer function looks like this:
void jit_timer_fn(unsigned long arg)
{
    struct jit_data *data = (struct jit_data *)arg;
    unsigned long j = jiffies;
    data->buf += sprintf(data->buf, "%9li %3li %i %6i %i %s\n",
    		j, j - data->prevjiffies, in_interrupt() ? 1 : 0,
    		current->pid, smp_processor_id(), current->comm);
    if (--data->loops) {
    	data->timer.expires += tdelay;
    	data->prevjiffies = j;
    	add_timer(&data->timer);
    } else {
    	wake_up_interruptible(&data->wait);
    }
}
~~~

定时器 API 包括几个比上面介绍的那些更多的功能。下面的集合是完整的核提供的函数列表:

~~~c
int mod_timer(struct timer_list *timer, unsigned long expires);
~~~

> 更新一个定时器的超时时间, 使用一个超时定时器的一个普通的任务(再一次, 关马达软驱定时器是一个典型例子)。mod_timer 也可被调用于非激活定时器, 那里你正常地使用 add_timer。

------

~~~c
int del_timer_sync(struct timer_list *timer);
~~~

> 如同 `del_timer` 一样工作, 但是还保证当它返回时, 定时器函数不在任何 CPU 上运行。`del_timer_sync` 用来避免竞争情况在 SMP 系统上, 并且在 UP 内核中和`del_timer` 相同。这个函数应当在大部分情况下比 `del_timer` 更首先使用。这个函数可能睡眠如果它被从非原子上下文调用, 但是在其他情况下会忙等待。要十分小心调用 `del_timer_sync` 当持有锁时; 如果这个定时器函数试图获得同一个锁,系统会死锁。如果定时器函数重新注册自己, 调用者必须首先确保这个重新注册不会发生; 这常常同设置一个" 关闭 "标志来实现, 这个标志被定时器函数检查。

-------

~~~c
int timer_pending(const struct timer_list * timer);
~~~

> 返回真或假来指示是否定时器当前被调度来运行, 通过调用结构的其中一个不透明的成员。

-----

**现代Linux内核对定时器API**：这里的“定时器”指的是 **struct timer_list（低精度内核定时器）**，不是 hrtimer。

**一、最大的变化：初始化 API 完全改变**

📌 LDD3 时代写法（已废弃）

```c
struct timer_list timer;

init_timer(&timer);
timer.function = my_func;
timer.data = (unsigned long)dev;
timer.expires = jiffies + HZ;
add_timer(&timer);
```

特点：

- 手动赋值 `function`、手动赋值 `data`、手动设置 `expires`、`add_timer()`

------

📌 **现代内核写法（必须这样）**

```c
struct timer_list timer;

timer_setup(&timer, my_func, 0);
mod_timer(&timer, jiffies + HZ);
```

**回调函数签名也变了：**

❌ LDD3：

```c
void my_func(unsigned long data);
```

✅ 现代：

```c
void my_func(struct timer_list *t);
```

如果需要获取宿主结构：

```c
struct my_dev *dev = from_timer(dev, t, timer);
```

⚠️ 这是最重要的 API 变化。

------

**二、timer.data 字段被移除**

LDD3 中：

```c
timer.data = (unsigned long)dev;
```

现代内核：

- 不再有 `data` 字段、必须使用 `from_timer()` 或 `container_of()`

示例：

```c
struct my_dev {
    struct timer_list timer;
};
```

回调中：

```c
struct my_dev *dev = from_timer(dev, t, timer);
```

------

**三、init_timer() 已废弃**

| LDD3          | 现代   |
| ------------- | ------ |
| init_timer()  | ❌ 删除 |
| setup_timer() | ❌ 删除 |
| timer_setup() | ✅ 使用 |

------

**四、add_timer() 仍然存在，但推荐用 mod_timer()**

**LDD3：**

```c
add_timer(&timer);
```

**现代推荐：**

```c
mod_timer(&timer, expires);
```

原因：

- mod_timer 自动处理已激活状态、更安全、可重复使用

------

**五、删除定时器 API 增强**

LDD3：

```c
del_timer(&timer);
del_timer_sync(&timer);
```

**现代新增：**

```c
timer_shutdown(&timer);
timer_shutdown_sync(&timer);
```

作用：

- 防止 use-after-free
- 禁止再次 re-arm

这是现代内核的重要增强。

------

**六、CPU 绑定变化**

**LDD3 描述：**

> 定时器在注册它的 CPU 上运行

现代内核：

- 不再保证
- 若要指定 CPU：

```c
add_timer_on(&timer, cpu);
```

但一般不推荐依赖 CPU 绑定。

------

**七、timer flags 机制变化**

LDD3 几乎没提 flags。

现代：

```c
timer_setup(&timer, func, flags);
```

可用 flags：

| flag             | 作用                  |
| ---------------- | --------------------- |
| TIMER_DEFERRABLE | 空闲时不唤醒 CPU      |
| TIMER_PINNED     | 绑定 CPU              |
| TIMER_IRQSAFE    | 在 hardirq 关闭时运行 |

这是现代新增能力。

------

**八、内部实现变化（API 未变但机制变了）**

LDD3 时代：

- 简单 timer wheel

现代内核：

- 分层时间轮（hierarchical timing wheel）
- per-CPU timer base
- 支持 NO_HZ
- 高精度定时器分流到 hrtimer

但 API 表面保持类似。

------

**九、与 hrtimer 的分工变化**

LDD3 时代：

- 所有定时器都是 jiffies

现代：

- timer_list → 低精度
- hrtimer → 纳秒级精度

**如果你现在需要 <1ms 精度，必须用 hrtimer。**

------

**十、API 变化对比表（核心总结）**

| 项目       | LDD3          | 现代 Linux                 |
| ---------- | ------------- | -------------------------- |
| 初始化     | init_timer    | timer_setup                |
| 回调参数   | unsigned long | struct timer_list *        |
| data 传参  | timer.data    | from_timer()               |
| 添加定时器 | add_timer     | mod_timer                  |
| 删除       | del_timer     | del_timer + timer_shutdown |
| flags      | 无            | 有 flags                   |
| CPU 保证   | 默认同 CPU    | 不保证                     |
| 精度       | jiffies       | jiffies + hrtimer 分流     |

------

**十一、现代标准写法完整示例**

```c
struct my_dev {
    struct timer_list timer;
};

static void my_timer_func(struct timer_list *t)
{
    struct my_dev *dev = from_timer(dev, t, timer);

    pr_info("timer fired\n");

    mod_timer(&dev->timer, jiffies + HZ);
}

static void init_my_timer(struct my_dev *dev)
{
    timer_setup(&dev->timer, my_timer_func, 0);
    mod_timer(&dev->timer, jiffies + HZ);
}

static void cleanup_my_timer(struct my_dev *dev)
{
    timer_shutdown_sync(&dev->timer);
}
```

这就是 6.x 标准模式。

------

**十二、总结一句话**

> API 层面最大的变化是：
>
> - init_timer → timer_setup
> - data 参数消失
> - 回调签名改变
> - 新增 timer_shutdown
> - 增加 flags

而底层机制变化更大（NO_HZ、分层时间轮、hrtimer 分流）。

-----------------------------

**定时器API代码示例的现代 Linux 修改后的完整代码**

**1️⃣ 定时器回调函数（新签名）**

```c
static void jit_timer_fn(struct timer_list *t)
{
    struct jit_data *data = from_timer(data, t, timer);
    unsigned long j = jiffies;

    data->buf += sprintf(data->buf, "%9lu %3lu %i %6i %i %s\n",
            j, j - data->prevjiffies,
            in_interrupt() ? 1 : 0,
            current->pid,
            smp_processor_id(),
            current->comm);

    if (--data->loops) {
        data->timer.expires = j + tdelay;
        data->prevjiffies = j;
        mod_timer(&data->timer, data->timer.expires);
    } else {
        wake_up_interruptible(&data->wait);
    }
}
```

------

**2️⃣ 定时器初始化与启动**

```c
unsigned long j = jiffies;

/* fill the data for our timer function */
data->prevjiffies = j;
data->buf = buf2;
data->loops = JIT_ASYNC_LOOPS;

/* 初始化定时器（现代API） */
timer_setup(&data->timer, jit_timer_fn, 0);

/* 设置触发时间 */
data->timer.expires = j + tdelay;

/* 启动定时器 */
mod_timer(&data->timer, data->timer.expires);

/* wait for the buffer to fill */
wait_event_interruptible(data->wait, !data->loops);
```



### 7.4.2 内核定时器的实现

**DDL3旧内核是实现方法**

为了使用它们, 尽管你不会需要知道内核定时器如何实现, 这个实现是有趣的, 并且值得看一下它们的内部。
定时器的实现被设计来符合下列要求和假设:

- 定时器管理必须尽可能简化。
- 设计应当随着激活的定时器数目上升而很好地适应。
- 大部分定时器在几秒或最多几分钟内到时, 而带有长延时的定时器是相当少见。
- 一个定时器应当在注册它的同一个 CPU 上运行。

由内核开发者想出的解决方法是基于一个`每个CPU` 数据结构。这个 `timer_list` 结构包括一个指针指向这个的数据结构在它的 `base` 成员。如果 `base` 是 `NULL`, 这个定时器没有被调用运行; 否则, 这个指针告知哪个数据结构(并且, 因此, 哪个 CPU )运行它。`每个CPU` 数据项在第 8 章的"每个CPU 变量"一节中描述。

无论何时内核代码注册一个定时器( 通过 `add_timer` 或者 `mod_timer`), 操作最终由`internal_add_timer` 进行( 在`kernel/timer.c`), 它依次添加新定时器到一个双向定时器链表在一个关联到当前 CPU 的"层叠表" 中。这个层叠表象这样工作: 如果定时器在下一个 `0` 到 `255 jiffies` 内到时, 它被添加到专供短时定时器 256 列表中的一个上, 使用 `expires` 成员的最低有效位。如果它在将来更久时间到时( 但是在 `16,384 jiffies `之前 ), 它被添加到基于 `expires` 成员的 9 - 14位的 64 个列表中一个。对于更长的定时器, 同样的技巧用在 15 - 20 位, 21 - 26 位,和 27 - 31 位。带有一个指向将来还长时间的 expires 成员的定时器( 一些只可能发生在 64-位 平台上的事情 ) 被使用一个延时值 0xffffffff 进行哈希处理, 并且带有在过去到时的定时器被调度来在下一个时钟嘀哒运行。(一个已经到时的定时器模拟有时在高负载情况下被注册, 特别的是如果你运行一个可抢占内核)。

当触发 `__run_timers`, 它为当前定时器嘀哒执行所有挂起的定时器。如果 `jiffies` 当前是 256 的倍数, 这个函数还重新哈希处理一个下一级别的定时器列表到 256 短期列表,可能地层叠一个或多个别的级别, 根据 `jiffies` 的位表示。

这个方法, 虽然第一眼看去相当复杂, 在几个和大量定时器的时候都工作得很好。用来管理每个激活定时器的时间独立于已经注册的定时器数目并且限制在几个对于它的 `expires`成员的二进制表示的逻辑操作上。关联到这个实现的唯一的开销是给 512 链表头的内存( 256 短期链表和 4 组 64 更长时间的列表) -- 即 4 KB 的容量.

函数 `__run_timers`, 如同 `/proc/jitimer` 所示, 在原子上下文运行。除了我们已经描述过的限制, 这个带来一个有趣的特性: 定时器刚好在合适的时间到时, 甚至你没有运行一个可抢占内核, 并且 CPU 在内核空间忙。你可以见到发生了什么当你在后台读`/proc/jitbusy` 时以及在前台 `/proc/jitimer`。尽管系统看来牢固地被锁住被这个忙等待系统调用, 内核定时器照样工作地不错。

但是, 记住, 一个内核定时器还远未完善, 因为它受累于` jitter` 和 其他由硬件中断引起怪物, 还有其他定时器和其他异步任务。虽然一个关联到简单数字 `I/O` 的定时器对于一个如同运行一个步进马达或者其他业余电子设备等简单任务是足够的, 它常常是不合适在工业环境中的生产系统。对于这样的任务, 你将最可能需要依赖一个实时内核扩展。

-------------------------

**现代Linux的实现方法**

现代 Linux（5.x / 6.x）内核中的**定时器实现机制**相比《Linux Device Drivers》时代已经有明显演进。LDD3 主要描述的是 **早期 timer wheel + jiffies** 的模型，而现代内核在结构、扩展性和高精度支持方面做了很多改进。

下面从 **整体架构 → 核心数据结构 → 工作流程 → 与旧实现差异** 逐步说明。

**一、现代 Linux 定时器体系结构**

现代内核的定时器系统实际上分为两类：

```
Linux Timer Subsystem
│
├─ Low resolution timer
│   └─ timer_list（传统 jiffies 定时器）
│
└─ High resolution timer
    └─ hrtimer（纳秒级高精度）
```

两者分工：

| 类型       | 精度      | 使用场景     |
| ---------- | --------- | ------------ |
| timer_list | jiffies级 | 普通内核延迟 |
| hrtimer    | ns级      | 高精度调度   |

很多驱动仍然使用 **timer_list**。

------------------------

**二、timer_list 的核心实现：分层时间轮**

现代 Linux 使用 **Hierarchical Timing Wheel（分层时间轮）**。

结构类似：

```
Level 0  256 slots
Level 1  64 slots
Level 2  64 slots
Level 3  64 slots
Level 4  64 slots
```

每个 slot 是一个链表：

```
slot[i] → timer → timer → timer
```

时间轮结构示意：

```
                Level 4
               ┌───────┐
               │ slots │
               └───────┘
                   ↑
               Level 3
                   ↑
               Level 2
                   ↑
               Level 1
                   ↑
               Level 0  (当前tick)
```

**为什么这样设计**, 目的：

- O(1) 插入定时器
- O(1) 删除定时器
- 减少扫描成本
- 支持大量定时器

这是 Linux 高性能的重要设计之一。

-------------

**三、每 CPU 定时器队列**

现代内核使用 **per-CPU timer base**。结构：

```
CPU0
 └─ timer_base
     └─ timing wheel

CPU1
 └─ timer_base
     └─ timing wheel
```

优点：

- 减少锁竞争
- 提高 SMP 性能
- 定时器通常在本 CPU 运行

核心结构（简化）：

```c
struct timer_base {
    spinlock_t lock;
    struct hlist_head vectors[LVL_DEPTH][LVL_SIZE];
    unsigned long clk;
};
```

-----------

**四、timer_list 数据结构**

现代版本（简化）：

```c
struct timer_list {
    struct hlist_node entry;
    unsigned long expires;
    void (*function)(struct timer_list *);
    u32 flags;
};
```

关键字段：

| 字段     | 作用       |
| -------- | ---------- |
| entry    | 链入时间轮 |
| expires  | 触发时间   |
| function | 回调函数   |
| flags    | 定时器属性 |

------------

**五、定时器触发流程**

整体流程如下：

```
硬件时钟中断
        │
        ▼
tick interrupt
        │
        ▼
run_local_timers()
        │
        ▼
timer softirq
        │
        ▼
__run_timers()
        │
        ▼
执行 timer callback
```

更完整流程：

```
clock interrupt
   │
   ▼
update_process_times()
   │
   ▼
run_local_timers()
   │
   ▼
raise_softirq(TIMER_SOFTIRQ)
   │
   ▼
run_timer_softirq()
   │
   ▼
__run_timers()
   │
   ▼
call_timer_fn()
```

因此：**定时器回调运行在 softirq 上下文。**

------------------------

**六、定时器插入算法**

当调用：

```
mod_timer(&timer, expires);
```

内核会：

1. 计算剩余 tick
2. 选择时间轮层级
3. 插入对应 slot 链表

伪代码：

```c
delta = expires - current_time

if delta < 256
    放入 level0
else if delta < 2^14
    放入 level1
else if ...
```

这样可以避免扫描大量定时器。

--------------

**七、时间推进机制**

每个 tick：

```
current_time++
```

然后：

```
检查 level0 slot
执行其中定时器
```

如果 level0 轮转：

```
cascade level1
```

即：

```
level1 slot → 下移到 level0
```

这就是 **时间轮级联(cascade)**。

---------------

**八、删除定时器**

删除定时器只需：

```
从链表移除
```

因此：

```c
del_timer()
```

复杂度：

```
O(1)
```

这是 timer wheel 的关键优势。

------------

**九、NO_HZ（tickless）支持**

现代 Linux 支持 **NO_HZ**：

CPU idle 时：

```
停止 periodic tick
```

定时器会：

```c
program next event
```

即：

```
下一次最近的 timer
```

这样可以：

- 节能、减少中断

LDD3 时代没有这个机制。

-----------

**十、定时器回调上下文**

回调运行在：

```
softirq context
```

因此，**不能**：

```
sleep
schedule
mutex_lock
```

允许：

```
spin_lock
atomic ops
wake_up
```

这和 LDD3 描述基本一致。

--------

**十一、现代新增特性**

现代 timer_list 增加了 flags：

| flag             | 作用          |
| ---------------- | ------------- |
| TIMER_DEFERRABLE | idle 时不唤醒 |
| TIMER_PINNED     | 绑定 CPU      |
| TIMER_IRQSAFE    | IRQ safe      |

示例：

```c
timer_setup(&timer, func, TIMER_DEFERRABLE);
```

-------------

**十二、与 LDD3 实现差异总结**

| 项目        | LDD3       | 现代 Linux  |
| ----------- | ---------- | ----------- |
| timer wheel | 有         | 改进版      |
| CPU 模型    | 全局       | per-CPU     |
| tick 模型   | periodic   | 支持 NO_HZ  |
| API         | init_timer | timer_setup |
| 高精度      | 无         | hrtimer     |
| flags       | 无         | 有          |

------

**十三、一句话总结**

现代 Linux 定时器的核心思想仍然是：

> **基于 jiffies 的分层时间轮 + per-CPU 队列 + softirq 执行回调**

但实现已经增加：

- per-CPU timer base、NO_HZ tickless、高精度 hrtimer、更安全的 API



## 7.5 Tasklets 机制

另一个有关于定时问题的内核设施是 tasklet 机制。它大部分用在中断管理(我们将在第10 章再次见到)。

`tasklet` 类似内核定时器在某些方面。它们一直在中断时间运行, 它们一直运行在调度它们的同一个 CPU 上, 并且它们接收一个 `unsigned long` 参数。不象内核定时器, 但是,你无法请求在一个指定的时间执行函数。通过调度一个 `tasklet`, 你简单地请求它在以后的一个由内核选择的时间执行。这个行为对于中断处理特别有用, 那里硬件中断必须被尽快处理, 但是大部分的时间管理可以安全地延后到以后的时间。实际上, 一个` tasklet`,就象一个内核定时器, 在一个"软中断"的上下文中执行(以原子模式), 在使能硬件中断时执行异步任务的一个内核机制。一个`tasklet` 存在为一个时间结构, 它必须在使用前被初始化。初始化能够通过调用一个特定函数或者通过使用某些宏定义声明结构:

~~~c
#include <linux/interrupt.h>
struct tasklet_struct {
/* ... */
void (*func)(unsigned long);
	unsigned long data;
};
void tasklet_init(struct tasklet_struct *t,
void (*func)(unsigned long), unsigned long data);
DECLARE_TASKLET(name, func, data);
DECLARE_TASKLET_DISABLED(name, func, data);
~~~

`tasklet` 提供了许多有趣的特色:

* 一个 `tasklet` 能够被禁止并且之后被重新使能; 它不会执行直到它被使能与被禁止相同的的次数.
* 如同定时器, 一个 `tasklet` 可以注册它自己.
* 一个 `tasklet` 能被调度来执行以正常的优先级或者高优先级. 后一组一直是首先执行.
* `tasklet` 可能立刻运行, 如果系统不在重载下, 但是从不会晚于下一个时钟嘀哒.一个 `tasklet` 可能和其他 `tasklet` 并发, 但是对它自己是严格地串行的 -- 同样的 `tasklet` 从不同时运行在超过一个处理器上. 同样, 如已经提到的, 一个 `tasklet` 常常在调度它的同一个 CPU 上运行.

`jit` 模块包括 2 个文件, `/proc/jitasklet` 和 `/proc/jitasklethi`, 它返回和在"内核定时器"一节中介绍过的 `/proc/jitimer` 同样的数据. 当你读其中一个文件时, 你取回一个`header` 和 `sixdata` 行。第一个数据行描述了调用进程的上下文, 并且其他的行描述了一个 `tasklet` 过程连续运行的上下文。这是一个在编译一个内核时的运行例子:

~~~c
void tasklet_disable(struct tasklet_struct *t);
~~~

> 这个函数禁止给定的 `tasklet`。 `tasklet` 可能仍然被 `tasklet_schedule` 调度, 但是它的执行被延后直到这个 `tasklet` 被再次使能。如果这个 `tasklet` 当前在运行,这个函数忙等待直到这个`tasklet` 退出; 因此, 在调用 `tasklet_disable` 后, 你可以确保这个 `tasklet` 在系统任何地方都不在运行。

------------------------

~~~c
void tasklet_disable_nosync(struct tasklet_struct *t);
~~~

> 禁止这个 `tasklet`, 但是没有等待任何当前运行的函数退出。当它返回, 这个`tasklet` 被禁止并且不会在以后被调度直到重新使能, 但是它可能仍然运行在另一个 CPU 当这个函数返回时。

------------------------

~~~c
void tasklet_enable(struct tasklet_struct *t);
~~~

> 使能一个之前被禁止的 `tasklet`。如果这个 `tasklet` 已经被调度, 它会很快运行。一个对 `tasklet_enable` 的调用必须匹配每个对 `tasklet_disable` 的调用, 因为内核跟踪每个 `tasklet` 的"禁止次数"。

------------------------

~~~c
void tasklet_schedule(struct tasklet_struct *t);
~~~

> 调度 `tasklet` 执行。如果一个 `tasklet` 被再次调度在它有机会运行前, 它只运行一次。但是, 如果他在运行中被调度, 它在完成后再次运行; 这保证了在其他事件被处理当中发生的事件收到应有的注意。这个做法也允许一个 `tasklet` 重新调度它自己。

------------------------

~~~c
void tasklet_hi_schedule(struct tasklet_struct *t);
~~~

> 调度 `tasklet` 在更高优先级执行。当软中断处理运行时, 它处理高优先级`tasklet` 在其他软中断之前, 包括"正常的" `tasklet`。理想地, 只有具有低响应周期要求( 例如填充音频缓冲 )应当使用这个函数, 为避免其他软件中断处理引入的附加周期。实际上, `/proc/jitasklethi` 没有显示可见的与 `/proc/jitasklet` 的区别。

------------------------

~~~c
void tasklet_kill(struct tasklet_struct *t);
~~~

> 这个函数确保了这个 tasklet 没被再次调度来运行; 它常常被调用当一个设备正被关闭或者模块卸载时. 如果这个 tasklet 被调度来运行, 这个函数等待直到它已执行. 如果这个 tasklet 重新调度它自己, 你必须阻止在调用 tasklet_kill 前它重新调度它自己, 如同使用 del_timer_sync.

------------------------

`tasklet` 在 `kernel/softirq.c` 中实现。2 个 tasklet 链表( 正常和高优先级 )被声明为每-CPU 数据结构, 使用和内核定时器相同的 CPU-亲和 机制。在 `tasklet` 管理中的数据结构是简单的链表, 因为 `tasklet` 没有内核定时器的分类请求。

----------------

现代 Linux 内核里 **tasklet** 机制其实已经逐步被“边缘化”，很多子系统正在改用 **workqueue** 或 **softirq + threaded context**。如果你对比 **DDL3**（很多厂商内部系统里也叫 DDL3 的延迟执行模型），会发现 Linux 的 tasklet 在设计理念上和早期 DDL3 的 **deferred execution** 很像，但实现和演进差别比较大。

**一、Linux Tasklet 本质**

**Tasklet = 基于 SoftIRQ 的轻量级延迟执行机制**

它解决的问题：

```
硬中断 (IRQ)
    ↓
不能做耗时工作
    ↓
延迟执行
    ↓
tasklet / softirq / workqueue
```

执行层级：

```
Hard IRQ
   ↓
SoftIRQ
   ↓
Tasklet
```

所以 **tasklet其实是softirq的一个封装层**。

内核里有两个固定的 softirq：

```c
TASKLET_SOFTIRQ
HI_SOFTIRQ
```

对应两个队列：

```c
tasklet_vec[]
tasklet_hi_vec[]
```

-------------

**二、Tasklet 内核结构**

核心结构：

```c
struct tasklet_struct {
    struct tasklet_struct *next;
    unsigned long state;
    atomic_t count;
    void (*func)(unsigned long);
    unsigned long data;
};
```

字段作用：

| 字段  | 作用        |
| ----- | ----------- |
| next  | 链表        |
| state | 运行状态    |
| count | disable计数 |
| func  | 回调        |
| data  | 参数        |

------

调度流程，驱动中使用：

```
tasklet_schedule(&t);
```

流程：

```
tasklet_schedule
      ↓
__tasklet_schedule
      ↓
加入 per-CPU tasklet_vec
      ↓
raise_softirq_irqoff(TASKLET_SOFTIRQ)
      ↓
softirq 执行
      ↓
tasklet_action()
      ↓
执行 func()
```

执行路径：

```
IRQ
 ↓
softirq
 ↓
tasklet_action
 ↓
tasklet->func()
```

------------

**三、Linux 6.x 的重要变化**

在 **Linux kernel 5.x ~ 6.x** 中，tasklet 有几个重要变化：

**1 Tasklet 已经被官方标记为 Deprecated**

Linux kernel 文档已经明确：

> **Tasklets are deprecated**

原因：

**① 难以并行**

tasklet 特点：

```
同一个tasklet不能并行执行
```

靠 state 锁实现：

```
TASKLET_STATE_RUN
```

导致：

```
SMP扩展性差
```

------

**② 不可睡眠**

tasklet运行在：

```
softirq context
```

限制：

```
不能sleep
不能阻塞
不能调用可能schedule的函数
```

这在现代驱动里很不方便。

------

**③ RT 内核不兼容**

在 **PREEMPT_RT** 中：

```
softirq 会线程化
```

tasklet语义会改变。

------

**2 新驱动推荐使用**

官方建议：

| 旧机制      | 新机制         |
| ----------- | -------------- |
| tasklet     | workqueue      |
| softirq     | threaded irq   |
| bottom half | kthread worker |

例如：

```
tasklet → workqueue
```

------------

**3 内核源码中的变化**

Linux 6.x 仍然保留：

```
kernel/softirq.c
```

关键函数：

```
tasklet_action()
tasklet_hi_action()
```

但越来越多驱动删除 tasklet。例如：

```
network drivers
wireless drivers
```

都迁移到：

```
NAPI
workqueue
```

------------

**四、和 DDL3 的区别**

如果你说的 **DDL3 延迟执行模型**（很多嵌入式系统里用），它和 Linux tasklet 很像，但有关键差异：

| 特性        | Linux tasklet     | DDL3         |
| ----------- | ----------------- | ------------ |
| 运行层级    | softirq           | 自定义调度   |
| CPU绑定     | per-CPU           | 通常全局     |
| 并发        | 同tasklet不可并发 | 通常可并发   |
| 调度来源    | IRQ               | 任务/ISR     |
| 是否能sleep | 不能              | 通常可以     |
| 现代支持    | 逐渐淘汰          | 很多系统仍用 |

------

**典型 DDL3 模型**

一般结构：

```
ISR
 ↓
DDL3 enqueue
 ↓
scheduler thread
 ↓
deferred function
```

而 Linux：

```
ISR
 ↓
raise_softirq
 ↓
tasklet_action
 ↓
callback
```

关键区别：

```
DDL3  = thread based
tasklet = softirq based
```

-------------

**五、现代 Linux Bottom Half 体系**

现代 Linux 延迟执行层级：

```
Hard IRQ
  │
  ├─ SoftIRQ
  │     ├─ NET_RX
  │     ├─ NET_TX
  │     └─ TASKLET
  │
  ├─ Tasklet  (deprecated)
  │
  ├─ Workqueue  (推荐)
  │
  └─ kthread worker
```

------

**六、为什么 Linux 不再推荐 Tasklet**

核心原因只有三个：

```
1 SMP 扩展性差
2 无法 sleep
3 RT 不友好
```

所以现代驱动基本用：

```
IRQ
 ↓
NAPI / threaded IRQ
 ↓
workqueue
```

------

**七、一个真实驱动例子**

旧写法：

```c
tasklet_schedule(&rx_tasklet);
```

新写法：

```c
queue_work(system_wq, &rx_work);
```

------



## 7.6 工作队列

工作队列是, 表面上看, 类似于 `taskets`; 它们允许内核代码来请求在将来某个时间调用一个函数。但是, 有几个显著的不同在这 2 个之间, 包括:

* `tasklet` 在软件中断上下文中运行的结果是所有的 `tasklet` 代码必须是原子的.相反, 工作队列函数在一个特殊内核进程上下文运行; 结果, 它们有更多的灵活性。特别地, 工作队列函数能够睡眠。

* `tasklet` 常常在它们最初被提交的处理器上运行。工作队列以相同地方式工作, 缺省地。

* 内核代码可以请求工作队列函数被延后一个明确的时间间隔。

两者之间关键的不同是 `tasklet` 执行的很快, 短时期, 并且在原子态, 而工作队列函数可能有高周期但是不需要是原子的. 每个机制有它适合的情形。

工作队列有一个 `struct workqueue_struct` 类型, 在 `<linux/workqueue.h>` 中定义. 一个工作队列必须明确的在使用前创建, 使用一个下列的 2 个函数:

~~~c
struct workqueue_struct *create_workqueue(const char *name);
struct workqueue_struct *create_singlethread_workqueue(const char *name);
~~~

每个工作队列有一个或多个专用的进程("内核线程"), 它运行提交给这个队列的函数。如果你使用 `create_workqueue`, 你得到一个工作队列它有一个专用的线程在系统的每个处理器上。在很多情况下, 所有这些线程是简单的过度行为; 如果一个单个工作者线程就足够, 使用 `create_singlethread_workqueue` 来代替创建工作队列提交一个任务给一个工作队列, 你需要填充一个 `work_struct` 结构。这可以在编译时完成, 如下:

~~~c
DECLARE_WORK(name, void (*function)(void *), void *data);
~~~

这里 `name` 是声明的结构名称, `function` 是从工作队列被调用的函数, 以及 `data` 是一个传递给这个函数的值。如果你需要建立 `work_struct` 结构在运行时, 使用下面 2 个宏定义:

~~~c
INIT_WORK(struct work_struct *work, void (*function)(void *), void *data);
PREPARE_WORK(struct work_struct *work, void (*function)(void *), void *data);
~~~

`INIT_WORK` 做更加全面的初始化结构的工作; 你应当在第一次建立结构时使用它。`PREPARE_WORK` 做几乎同样的工作, 但是它不初始化用来连接 `work_struct` 结构到工作队列的指针。如果有任何的可能性这个结构当前被提交给一个工作队列, 并且你需要改变这个队列, 使用 `PREPARE_WORK` 而不是 `INIT_WORK`。有 2 个函数来提交工作给一个工作队列:

~~~c
int queue_work(struct workqueue_struct *queue, struct work_struct *work);
int queue_delayed_work(struct workqueue_struct *queue, struct work_struct
			*work, unsigned long delay);
~~~

每个都添加工作到给定的队列。如果使用 `queue_delay_work`, 但是, 实际的工作没有进行直到至少 `delay jiffies` 已过去。从这些函数的返回值是 0 如果工作被成功加入到队列; 一个非零结果意味着这个 `work_struct` 结构已经在队列中等待, 并且第 2 次没有加入。

在将来的某个时间, 这个工作函数将被使用给定的 data 值来调用。这个函数将在工作者线程的上下文运行, 因此它可以睡眠如果需要 -- 尽管你应当知道这个睡眠可能怎样影响提交给同一个工作队列的其他任务。这个函数不能做的是, 但是, 是存取用户空间。因为它在一个内核线程中运行, 完全没有用户空间来存取.如果你需要取消一个挂起的工作队列入口, 你可以调用:

~~~c
int cancel_delayed_work(struct work_struct *work);
~~~

返回值是非零如果这个入口在它开始执行前被取消。内核保证给定入口的执行不会在调用`cancel_delay_work` 后被初始化。如果 `cancel_delay_work` 返回 0, 但是, 这个入口可能已经运行在一个不同的处理器, 并且可能仍然在调用 `cancel_delayed_work` 后在运行。要绝对确保工作函数没有在 `cancel_delayed_work` 返回 0 后在任何地方运行, 你必须跟随这个调用来调用:

~~~C
void flush_workqueue(struct workqueue_struct *queue);
~~~

在 `flush_workqueue` 返回后, 没有在这个调用前提交的函数在系统中任何地方运行。当你用完一个工作队列, 你可以去掉它, 使用:

~~~c
void destroy_workqueue(struct workqueue_struct *queue);
~~~

-------------

**现代 Linux kernel 的 Workqueue 和 DDL3 的“工作队列（work queue）” **。其实两者名字很像，但**设计层级完全不同**。

**一、架构层级区别（最核心）**

**Linux Workqueue 架构**，Linux 是 **内核线程池模型**：

```
kernel subsystem / driver
        │
        ▼
   queue_work()
        │
        ▼
workqueue_struct
        │
        ▼
per-CPU worker_pool
        │
        ▼
kworker thread
        │
        ▼
work callback
```

关键特点：

```
每个 CPU 有 worker pool
多个 kworker 并行执行
```

------

**DDL3 工作队列架构**，DDL3 通常是 **调度线程 + 队列模型**：

```
ISR / task
    │
    ▼
enqueue(work)
    │
    ▼
DDL3 work queue
    │
    ▼
scheduler thread
    │
    ▼
callback
```

关键特点：

```
一个或多个调度线程
从队列取任务执行
```

------

**二、CPU 并发模型**

**Linux Workqueue**，Linux 内核设计是 **SMP 优化的**：

```
CPU0 worker_pool
CPU1 worker_pool
CPU2 worker_pool
CPU3 worker_pool
```

任务执行：

```
work1 → CPU0
work2 → CPU1
work3 → CPU2
```

优点：

```
天然支持多核并行
减少锁竞争
```

------

**DDL3 工作队列**，通常是：

```
global queue
      │
      ▼
thread pool
```

并发能力取决于：

```
线程数量
队列锁
```

缺点：

```
可能出现锁竞争
CPU locality 差
```

------

**三、调度机制**

**Linux Workqueue** 调度函数：

```
queue_work()
schedule_work()
queue_delayed_work()
```

流程：

```
queue_work
   │
   ▼
pwq (per cpu workqueue)
   │
   ▼
worker_pool
   │
   ▼
wake_up_worker
```

延迟任务：

```
delayed_work
   │
timer
   │
workqueue
```

------

**DDL3 工作队列**，调度通常是：

```
enqueue()
signal()
```

执行：

```
while(true)
{
    task = dequeue()
    run(task)
}
```

延迟任务：

```
timer thread
   │
enqueue
```

------

**四、实时性和阻塞**

**Linux Workqueue**，执行在：

```
kworker thread
```

特点：

```
可以 sleep
可以阻塞
不会影响 IRQ
```

支持：

```
RT kernel
CPU affinity
priority workqueue
```

------

**DDL3 工作队列**，执行在线程中：

```
scheduler thread
```

如果：

```
task 阻塞
```

可能：

```
阻塞整个队列线程
```

需要开发者自己控制。

------

**五、资源管理**

**Linux Workqueue**，内核自动管理：

```
worker 数量
CPU 绑定
负载均衡
唤醒策略
```

关键代码在：

```
kernel/workqueue.c
```

自动扩展 worker：

```
create_worker()
destroy_worker()
```

------

**DDL3**，通常开发者要自己管理：

```
线程池
队列
同步锁
调度策略
```

灵活但复杂。

------

**六、典型结构对比**

**Linux Workqueue 数据结构**，核心结构：

```
work_struct
workqueue_struct
pool_workqueue
worker_pool
worker
```

关系：

```
work_struct
     │
     ▼
workqueue_struct
     │
     ▼
pool_workqueue
     │
     ▼
worker_pool
     │
     ▼
worker thread
```

------

**DDL3 工作队列结构**，一般比较简单：

```
task_node
queue
mutex
condition
thread_pool
```

结构：

```
task
 │
 ▼
queue
 │
 ▼
thread
```

------

**七、最本质区别（总结）**

| 特性     | Linux Workqueue     | DDL3 工作队列 |
| -------- | ------------------- | ------------- |
| 层级     | 内核调度机制        | 应用/系统框架 |
| 线程管理 | 内核自动            | 开发者管理    |
| CPU模型  | per-CPU worker pool | 通常全局队列  |
| 并发     | 高度并行            | 取决于线程池  |
| 锁竞争   | 低                  | 可能较高      |
| 实时控制 | 内核支持            | 依赖实现      |

一句话总结：

```
Linux Workqueue = SMP 优化的内核线程池
DDL3 Work Queue = 普通任务队列 + 线程池
```

------

💡 **一个很多人不知道但非常关键的区别：**

Linux workqueue 里面有一个很复杂但很核心的结构：

```
pool_workqueue (pwq)
```

它负责：

```
workqueue ↔ worker_pool 的映射
```

这个结构是 **Linux Workqueue 能做到高并发 + 低锁竞争的关键设计**。

----------------



### 7.6.1 共享队列

一个设备驱动, 在许多情况下, 不需要它自己的工作队列。如果你只偶尔提交任务给队列,简单地使用内核提供的共享的, 缺省的队列可能更有效。如果你使用这个队列, 但是, 你必须明白你将和别的在共享它。从另一个方面说, 这意味着你不应当长时间独占队列(无长睡眠), 并且可能要更长时间它们轮到处理器。

`jiq` ("`just in queue`") 模块输出 2 个文件来演示共享队列的使用。它们使用一个单个`work_struct structure`, 这个结构这样建立:

~~~c
static struct work_struct jiq_work;
/* this line is in jiq_init() */
INIT_WORK(&jiq_work, jiq_print_wq, &jiq_data);
~~~

当一个进程读 `/proc/jiqwq`, 这个模块不带延迟地初始化一系列通过共享的工作队列的路线。

~~~c
int schedule_work(struct work_struct *work);
~~~

注意, 当使用共享队列时使用了一个不同的函数; 它只要求` work_struct `结构作为一个参数。在 `jiq` 中的实际代码看来如此:

~~~c
prepare_to_wait(&jiq_wait, &wait, TASK_INTERRUPTIBLE);
schedule_work(&jiq_work);
schedule();
finish_wait(&jiq_wait, &wait);
~~~

这个实际的工作函数打印出一行就象 `jit` 模块所作的, 接着, 如果需要, 重新提交这个`work_structcture` 到工作队列中。在这是 `jiq_print_wq` 全部:

~~~c
static void jiq_print_wq(void *ptr)
{
    struct clientdata *data = (struct clientdata *) ptr;
    if (! jiq_print (ptr))
    	return;
    if (data->delay)
    	schedule_delayed_work(&jiq_work, data->delay);
    else
    	schedule_work(&jiq_work);
}
~~~

如果用户在读被延后的设备 (`/proc/jiqwqdelay`), 这个工作函数重新提交它自己在延后的模式, 使用 `schedule_delayed_work`:

~~~c
int schedule_delayed_work(struct work_struct *work, unsigned long delay);
~~~

如果你需要取消一个已提交给工作队列的工作入口, 你可以使用 `cancel_delayed_work`,如上面所述. 刷新共享队列需要一个不同的函数, 但是:

~~~c
void flush_scheduled_work(void);
~~~

因为你不知道别人谁可能使用这个队列, 你从不真正知道 `flush_schduled_work` 返回可能需要多长时间。

-----------------

**现代 Linux kernel Workqueue** 和 **DDL3 工作队列中的共享队列**，其实这是两种不同的并发设计思想。 核心区别在 **队列是否被多个执行线程共享**。

**一、DDL3 的共享队列（Shared Work Queue）**

DDL3 中常见的是 **共享任务队列**：

结构大致是：

```
            +-------------+
            |  Work Queue |
            +-------------+
               /   |   \
              /    |    \
         Thread1 Thread2 Thread3
```

执行流程：

```
task enqueue
     │
     ▼
shared queue
     │
     ▼
worker thread pool
     │
     ▼
dequeue task
     │
     ▼
execute
```

特点：

1️⃣ **所有线程共享同一个队列**
2️⃣ 线程竞争队列锁
3️⃣ 哪个线程抢到锁就执行任务

典型代码逻辑：

```
lock(queue)
task = dequeue()
unlock(queue)

execute(task)
```

优点：

- 结构简单、实现容易、任务负载自动均衡

缺点：

```
锁竞争严重
CPU cache locality 差
扩展性有限
```

当 CPU 数量多时：

```
多个线程争用同一队列锁
```

性能会下降。

------

**二、Linux Workqueue 的队列模型**

Linux 为了 **SMP 扩展性**，基本不使用单一共享队列，而是 **per-CPU 队列**。结构：

```
             workqueue
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
   CPU0 queue CPU1 queue CPU2 queue
      │         │         │
   kworker0  kworker1  kworker2
```

执行流程：

```
queue_work()
     │
     ▼
per-CPU worklist
     │
     ▼
kworker thread
     │
     ▼
execute
```

特点：

```
每个CPU自己的任务队列
减少锁竞争
提高cache命中
```

------

**三、Linux 也有“共享队列”**

虽然 Linux 默认是 per-CPU 队列，但也支持 **共享 worker pool**。例如：

```
WQ_UNBOUND workqueue
```

结构：

```
           global worker pool
                 │
         +-------+-------+
         |       |       |
       worker  worker  worker
                 │
            shared worklist
```

这种模式和 DDL3 **比较类似**：

```
共享线程池
共享任务队列
```

但是 Linux 仍然做了优化：

- worker 自动扩展、NUMA 优化、调度器控制 CPU 亲和性

------

**四、锁竞争对比**

**DDL3 共享队列**

```
enqueue → lock(queue)
dequeue → lock(queue)
```

多个线程：

```
Thread1 ┐
Thread2 ├─竞争 queue lock
Thread3 ┘
```

------

**Linux per-CPU 队列**

```
CPU0 → queue0
CPU1 → queue1
CPU2 → queue2
```

基本没有竞争：

```
CPU0 worker → queue0
CPU1 worker → queue1
```

优势：

```
锁竞争极低
cache locality好
SMP扩展好
```

------

**五、架构思想差异（本质）**

DDL3：

```
shared queue
   │
thread pool
```

Linux：

```
per-CPU queue
     │
per-CPU worker
```

设计理念：

| 系统  | 设计目标   |
| ----- | ---------- |
| DDL3  | 简单线程池 |
| Linux | SMP 可扩展 |

------

**六、什么时候需要共享队列**

Linux 仍然保留共享队列模式，因为有些场景适合：

```
长时间任务
CPU不敏感任务
IO处理
```

例如：

```
system_unbound_wq
```

不绑定 CPU。

------

**七、总结**

| 特性     | DDL3 共享队列 | Linux Workqueue  |
| -------- | ------------- | ---------------- |
| 队列     | 单共享队列    | per-CPU 队列     |
| 锁竞争   | 高            | 低               |
| CPU扩展  | 一般          | 很好             |
| 线程管理 | 线程池        | kworker线程池    |
| 共享模式 | 默认          | 可选(WQ_UNBOUND) |

一句话总结：

```
DDL3 = 共享队列 + 线程池
Linux = per-CPU队列 + worker pool
```



# END


