# Linux 设备驱动

# 第五章 并发和竞争情况

迄今, 我们未曾关心并发的问题 -- 就是说, 当系统试图一次做多件事时发生的情况。然而, 并发的管理是操作系统编程的核心问题之一。 并发相关的错误是一些最易出现又最难发现的问题。即便是专家级 Linux 内核程序员偶尔也会出现并发相关的错误。

早期的 Linux 内核, 较少有并发的源头。内核不支持对称多处理器(SMP)系统, 并发执行的唯一原因是硬件中断服务。那个方法提供了简单性, 但是在有越来越多处理器的系统上注重性能并且坚持系统要快速响应事件的世界中它不再可行了。为响应现代硬件和应用程序的要求, Linux 内核已经发展为很多事情在同时进行。这个进步已经产生了很大的性能和可扩展性。然而, 它也很大地使内核编程任务复杂化. 设备启动程序员现在必须从一开始就将并发作为他们设计的要素, 并且他们必须对内核提供的并发管理设施有很强的理解。

本章的目的是开始建立那种理解的过程。为此目的, 我们介绍一些设施来立刻应用到第 3章的 scull 驱动。展示的其他设施暂时还不使用。但是首先, 我们看一下我们的简单scull 驱动可能哪里出问题并且如何避免这些潜在的问题。

## 5.1 scull 中的缺陷

让我们快速看一段 scull 内存管理代码。在写逻辑的深处, scull 必须决定它请求的内存是否已经分配。处理这个任务的代码是:

~~~c
if (!dptr->data[s_pos]) {
    dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL);
    if (!dptr->data[s_pos])
        goto out;
}
~~~

假设有 2 个进程( 我们会称它们为"A"和"B" ) 独立地试图写入同一个 schull 设备的相同偏移。每个进程同时到达上面片段的第一行的 if 测试。如果被测试的指针是 `NULL`, 每个进程都会决定分配内存, 并且每个都会复制结果指针给 `dptr->datat[s_pos]`。因为 2 个进程都在赋值给同一个位置, 显然只有一个赋值可以成功。

当然, 发生的是第 2 个完成赋值的进程将"胜出"。如果进程 A 先赋值, 它的赋值将被进程 B 覆盖。在此, scull 将完全忘记 A 分配的内存; 它只有指向 B 的内存的指针。A 所分配的指针, 因此, 将被丢掉并且不再返回给系统。

事情的这个顺序是一个竞争情况的演示。竞争情况是对共享数据的无控制存取的结果。当错误的存取模式发生了, 产生了不希望的东西。对于这里讨论的竞争情况, 结果是内存泄漏。这已经足够坏了, 但是竞争情况常常导致系统崩溃和数据损坏。程序员可能被诱惑而忽视竞争情况为相当低可能性的事件。但是, 在计算机世界, 百万分之一的事件会每隔几秒发生, 并且后果会是严重的。

很快我们将去掉 scull 的竞争情况, 但是首先我们需要对并发做一个更普遍的回顾。



## 5.2 并发和它的管理

竞争情况来自对资源的共享存取的结果。当 2 个执行的线路有机会操作同一个数据结构(或者硬件资源), 混合的可能性就一直存在。因此第一个经验法则是在你设计驱动时在任何可能的时候记住避免共享的资源。如果没有并发存取, 就没有竞争情况. 因此小心编写的内核代码应当有最小的共享。这个想法的最明显应用是避免使用全局变量。如果你将一个资源放在多个执行线路能够找到它的地方, 应当有一个很强的理由这样做。

事实是, 然而, 这样的共享常常是需要的。硬件资源是, 由于它们的特性, 共享的, 软件资源也必须常常共享给多个线程。也要记住全局变量远远不是共享数据的唯一方式; 任何时候你的代码传递一个指针给内核的其他部分, 潜在地它创造了一个新的共享情形。共享是生活的事实。

这是资源共享的硬规则: 任何时候一个硬件或软件资源被超出一个单个执行线程共享, 并且可能存在一个线程看到那个资源的不一致时, 你必须明确地管理对那个资源的存取。在上面的 scull 例子, 这个情况在进程 B 看来是不一致的; 不知道进程 A 已经为( 共享的 ) 设备分配了内存, 它做它自己的分配并且覆盖了 A 的工作。在这个例子里, 我们必须控制对 scull 数据结构的存取。我们需要安排, 这样代码或者看到内存已经分配了,或者知道没有内存已经或者将要被其他人分配。存取管理的常用技术是加锁或者互斥 -- 确保在任何时间只有一个执行线程可以操作一个共享资源. 本章剩下的大部分将专门介绍加锁。

然而, 首先, 我们必须简短考虑一下另一个重要规则。当内核代码创建一个会被内核其他部分共享的对象时, 这个对象必须一直存在(并且功能正常)到它知道没有对它的外部引用存在为止。scull 使它的设备可用的瞬间, 它必须准备好处理对那些设备的请求. 并且scull 必须一直能够处理对它的设备的请求直到它知道没有对这些设备的引用(例如打开的用户空间文件)存在。2 个要求出自这个规则: 除非它处于可以正确工作的状态, 不能有对象能对内核可用, 对这样的对象的引用必须被跟踪。在大部分情况下, 你将发现内核为你处理引用计数, 但是常常有例外。

遵照上面的规则需要计划和对细节小心注意。容易被对资源的并发存取而吃惊, 你事先并没有认识到被共享。通过一些努力, 然而, 大部分竞争情况能够在它们咬到你或者你的用户前被消灭。



## 5.3 旗标和互斥体

让我们看看我们如何给 scull 加锁。我们的目标是使我们对 scull 数据结构的操作原子化, 就是在有其他执行线程的情况下这个操作一次发生。对于我们的内存泄漏例子, 我们需要保证, 如果一个线程发现必须分配一个特殊的内存块, 它有机会进行这个分配在其他线程可做测试之前。为此, 我们必须建立临界区: 在任何给定时间只有一个线程可以执行的代码。

不是所有的临界区是同样的, 因此内核提供了不同的原语适用不同的需求。在这个例子中,对 scull 数据结构的存取都发生在由一个直接用户请求所产生的进程上下文中; 没有从中断处理或者其他异步上下文中的存取。没有特别的周期(响应时间)要求; 应用程序程序员理解 I/O 请求常常不是马上就满足的。进一步讲, scull 没有持有任何其他关键系统资源, 在它存取它自己的数据结构时。所有这些意味着如果 scull 驱动在等待轮到它存取数据结构时进入睡眠, 没人介意。

"去睡眠" 在这个上下文中是一个明确定义的术语。当一个 Linux 进程到了一个它无法做进一步处理的地方时, 它去睡眠(或者 "阻塞"), 让出处理器给别人直到以后某个时间它能够再做事情。进程常常在等待 I/O 完成时睡眠。随着我们深入内核, 我们会遇到很多情况我们不能睡眠。然而 scull 中的 write 方法不是其中一个情况。因此我们可使用一个加锁机制使进程在等待存取临界区时睡眠。

正如重要地, 我们将进行一个可能会睡眠的操作( 使用 kmalloc 分配内存 ) -- 因此睡眠是一个在任何情况下的可能性。如果我们的临界区要正确工作, 我们必须使用一个加锁原语在一个拥有锁的进程睡眠时起作用。不是所有的加锁机制都能够在可能睡眠的地方使用( 我们在本章后面会看到几个不可以的 )。然而, 对我们现在的需要, 最适合的机制时一个旗标。

旗标在计算机科学中是一个被很好理解的概念。在它的核心, 一个旗标是一个单个整型值,结合有一对函数, 典型地称为 P 和 V。一个想进入临界区的进程将在相关旗标上调用 P;如果旗标的值大于零, 这个值递减 1 并且进程继续。相反, 如果旗标的值是 0 ( 或更小 ), 进程必须等待直到别人释放旗标。解锁一个旗标通过调用 V 完成; 这个函数递增旗标的值, 并且, 如果需要, 唤醒等待的进程。

当旗标用作互斥 -- 阻止多个进程同时在同一个临界区内运行 -- 它们的值将初始化为 1。这样的旗标在任何给定时间只能由一个单个进程或者线程持有。以这种模式使用的旗标有时称为一个互斥锁, 就是, 当然, "互斥"的缩写。几乎所有在 Linux 内核中发现的旗标都是用作互斥。



### 5.3.1 Linux 旗标实现

Linux 内核提供了一个遵守上面语义的旗标实现, 尽管术语有些不同。为使用旗标, 内核代码必须包含 `<asm/semaphore.h>`。相关的类型是 struct semaphore; 实际旗标可以用几种方法来声明和初始化。 一种是直接创建一个旗标, 接着使用 `sema_init` 来设定它:

~~~c
void sema_init(struct semaphore *sem, int val); //这里 val 是安排给旗标的初始值。
~~~

然而, 通常旗标以互斥锁的模式使用。为使这个通用的例子更容易些, 内核提供了一套帮助函数和宏定义。因此, 一个互斥锁可以声明和初始化, 使用下面的一种:

~~~c
DECLARE_MUTEX(name);
DECLARE_MUTEX_LOCKED(name);
~~~

这里, 结果是一个旗标变量( 称为 `name` ), 初始化为 1 ( 使用 `DECLARE_MUTEX` ) 或者0 (使用 `DECLARE_MUTEX_LOCKED` ). 在后一种情况, 互斥锁开始于上锁的状态; 在允许任何线程存取之前将不得不显式解锁它。如果互斥锁必须在运行时间初始化( 这是如果动态分配它的情况, 举例来说), 使用下列中的一个:

~~~c
void init_MUTEX(struct semaphore *sem);
void init_MUTEX_LOCKED(struct semaphore *sem);
~~~

在 Linux 世界中, `P` 函数称为 `down` -- 或者这个名子的某个变体。这里, "`down`" 指的是这样的事实, 这个函数递减旗标的值, 并且, 也许在使调用者睡眠一会儿来等待旗标变可用之后, 给予对被保护资源的存取。有 3 个版本的 `down`:

~~~c
void down(struct semaphore *sem);
int down_interruptible(struct semaphore *sem);
int down_trylock(struct semaphore *sem);
~~~

`down` 递减旗标值并且等待需要的时间。`down_interruptible` 同样, 但是操作是可中断的.这个可中断的版本几乎一直是你要的那个; 它允许一个在等待一个旗标的用户空间进程被用户中断。作为一个通用的规则, 你不想使用不可中断的操作, 除非实在是没有选择。不可中断操作是一个创建不可杀死的进程( 在 `ps` 中见到的可怕的 "D 状态" )和惹恼你的用户的好方法, 使用 `down_interruptible` 需要一些格外的小心, 但是, 如果操作是可中断的, 函数返回一个非零值, 并且调用者不持有旗标。正确的使用 `down_interruptible`需要一直检查返回值并且针对性地响应。最后的版本 ( `down_trylock` ) 从不睡眠; 如果旗标在调用时不可用, `down_trylock` 立刻返回一个非零值。

一旦一个线程已经成功调用 `down` 各个版本中的一个, 就说它持有着旗标(或者已经"取得"或者"获得"旗标)。这个线程现在有权力存取这个旗标保护的临界区。当这个需要互斥的操作完成时, 旗标必须被返回。 `V`  的 Linux 对应物是 `up`:

~~~c
void up(struct semaphore *sem); // 一旦 up 被调用, 调用者就不再拥有旗标.
~~~

如你所愿, 要求获取一个旗标的任何线程, 使用一个(且只能一个)对 up 的调用释放它。在错误路径中常常需要特别的小心; 如果在持有一个旗标时遇到一个错误, 旗标必须在返回错误状态给调用者之前释放旗标。没有释放旗标是容易犯的一个错误; 这个结果( 进程挂在看来无关的地方 )可能是难于重现和跟踪的。



### 5.3.2 在 scull 中使用旗标

#### scull 中的 semaphore

旗标机制给予 scull 一个工具, 可以在存取 `scull_dev` 数据结构时用来避免竞争情况。但是正确使用这个工具是我们的责任。正确使用加锁原语的关键是严密地指定要保护哪个资源并且确认每个对这些资源的存取都使用了正确的加锁方法。在我们的例子驱动中, 感兴趣的所有东西都包含在 `scull_dev` 结构里面, 因此它是我们的加锁体制的逻辑范围。让我们在看看这个结构:

~~~c
struct scull_dev {
    struct scull_qset *data; 	/* Pointer to first quantum set */
    int quantum; 				/* the current quantum size */
    int qset; 					/* the current array size */
    unsigned long size; 		/* amount of data stored here */
    unsigned int access_key; 	/* used by sculluid and scullpriv */
    struct semaphore sem; 		/* mutual exclusion semaphore */
    struct cdev cdev; 			/* Char device structure */
};
/* demo中的结构 */
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

到结构的底部是一个称为 sem 的成员, 当然, 它是我们的旗标。我们已经选择为每个虚拟 scull 设备使用单独的旗标。使用一个单个的全局的旗标也可能会是同样正确。通常各种 scull 设备不共享资源, 然而, 并且没有理由使一个进程等待, 而另一个进程在使用不同 scull 设备。不同设备使用单独的旗标允许并行进行对不同设备的操作, 因此,提高了性能。

**由于API的进化现在几乎不再使用 `struct semaphore sem` 后续将简单介绍 `struct semaphore sem` 和 `struct mutex lock` 的差异。**

旗标在使用前必须初始化. scull 在加载时进行这个初始化, 在这个循环中:

~~~c
for (i = 0; i < scull_nr_devs; i++) {
    scull_devices[i].quantum = scull_quantum;
    scull_devices[i].qset = scull_qset;
    init_MUTEX(&scull_devices[i].sem);
    scull_setup_cdev(&scull_devices[i], i);
}
/* demo中的初始化 */
/* Initialize each device. */
for (i = 0; i < scull_nr_devs; i++) {
    scull_devices[i].quantum = scull_quantum;
    scull_devices[i].qset = scull_qset;
    mutex_init(&scull_devices[i].lock);
    scull_setup_cdev(&scull_devices[i], i);
}
~~~

注意, 旗标必须在 scull 设备对系统其他部分可用前初始化。因此, `init_MUTEX` 在`scull_setup_cdev` 前被调用。以相反的次序进行这个操作可能产生一个竞争情况, 旗标可能在它准备好之前被存取.下一步, 我们必须浏览代码, 并且确认在没有持有旗标时没有对 `scull_dev` 数据结构的存取。因此, 例如, `scull_write` 以这个代码开始:

~~~c
if (down_interruptible(&dev->sem))
	return -ERESTARTSYS;
/* demo中的 down */
if (mutex_lock_interruptible(&dev->lock))
	return -ERESTARTSYS;
~~~

注意对 `down_interruptible` 返回值的检查; 如果它返回非零,  操作被打断了。 在这个情况下通常要做的是返回  `-ERESTARTSYS`。 看到这个返回值后,  内核的高层要么从头重启这个调用要么返回这个错误给用户。 如果你返回 `-ERESTARTSYS`, 你必须首先恢复任何用户可见的已经做了的改变, 以保证当重试系统调用时正确的事情发生。如果你不能以这个方式恢复, 你应当替之返回 `-EINTR`。

`scull_write` 必须释放旗标, 不管它是否能够成功进行它的其他任务。如果事事都顺利,执行落到这个函数的最后几行:

~~~c
out:
    up(&dev->sem);
    return retval;
/* demo中的 out */
out:
    mutex_unlock(&dev->lock);
    return retval;
~~~

这个代码释放旗标并且返回任何需要的状态。在 `scull_write` 中有几个地方可能会出错;这些地方包括内存分配失败或者在试图从用户空间拷贝数据时出错。在这些情况中, 代码进行了一个 `goto out`, 以确保进行正确的清理。

#### demo 中的 mutex

**书中用 `struct semaphore`，demo 用 `struct mutex`，原因是：书写时的内核版本不同，`mutex` 是后来专门为“互斥”引入并推荐使用的更轻量锁。** 二者功能目标是一样的，都是为了**保护 `scull_dev` 的临界区**和**防止多个进程同时读写 `data / size / qset` 等成员**，但是本质实现手段不同一个是信号量一个是互斥锁。LDD3 成书时（约 2005 年）：内核里还没有 mutex；`semaphore` 是唯一的睡眠锁；用 `sema_init(&sem, 1)` 当作互斥锁使用。现在更多的是使用互斥量因为`mutex` **故意禁止很多危险用法**：

- ❌ 不能在中断上下文使用
- ❌ 不能递归加锁
- ❌ unlock 非持有者会 WARN

 这些限制能 **更早暴露 BUG**。



### 5.3.3 读者/写者旗标

旗标为所有调用者进行互斥, 不管每个线程可能想做什么。然而, 很多任务分为 2 种清楚的类型: 只需要读取被保护的数据结构的类型, 和必须做改变的类型。允许多个并发读者常常是可能的, 只要没有人试图做任何改变。这样做能够显著提高性能; 只读的任务可以并行进行它们的工作而不必等待其他读者退出临界区。Linux 内核为这种情况提供一个特殊的旗标类型称为 `rwsem` (或者" `reader/writer semaphore`")。`rwsem` 在驱动中的使用相对较少, 但是有时它们有用。

使用 `rwsem` 的代码必须包含 `<linux/rwsem.h>`。读者写者旗标 的相关数据类型是 `struct rw_semaphore`; 一个 `rwsem` 必须在运行时显式初始化:

~~~c
void init_rwsem(struct rw_semaphore *sem);
~~~

一个新初始化的 `rwsem` 对出现的下一个任务( 读者或者写者 )是可用的. 对需要只读存取的代码的接口是:

~~~c
void down_read(struct rw_semaphore *sem);
int down_read_trylock(struct rw_semaphore *sem);
void up_read(struct rw_semaphore *sem);
~~~

对 `down_read` 的调用提供了对被保护资源的只读存取, 与其他读者可能地并发地存取。注意 `down_read` 可能将调用进程置为不可中断的睡眠。`down_read_trylock` 如果读存取是不可用时不会等待; 如果被准予存取它返回非零, 否则是 0。注意 `down_read_trylock`的惯例不同于大部分的内核函数, 返回值 0 指示成功。 一个使用 `down_read` 获取的`rwsem` 必须最终使用 `up_read` 释放.读者的接口类似:

~~~c
void down_write(struct rw_semaphore *sem);
int down_write_trylock(struct rw_semaphore *sem);
void up_write(struct rw_semaphore *sem);
void downgrade_write(struct rw_semaphore *sem);
~~~

`down_write`, `down_write_trylock`, 和 `up_write` 全部就像它们的读者对应部分。除了,当然, 它们提供写存取。如果你处于这样的情况, 需要一个写者锁来做一个快速改变, 接着一个长时间的只读存取, 你可以使用 `downgrade_write` 在一旦你已完成改变后允许其他读者进入。

一个 `rwsem` 允许一个读者或者不限数目的读者来持有旗标。写者有优先权; 当一个写者试图进入临界区, 就不会允许读者进入直到所有的写者完成了它们的工作。这个实现可能导致读者饥饿 -- 读者被长时间拒绝存取 -- 如果你有大量的写者来竞争旗标。由于这个原因, `rwsem` 最好用在很少请求写的时候, 并且写者只占用短时间。

#### rw_semaphore 的变化

**注意：虽然`struct rw_semaphore` 本身没有被废弃，API 仍然有效； 但实现与推荐用法在新内核中发生了“明显演进”， 尤其是：公平性、性能路径、以及可替代方案。** 下面我分**“有没有变” → “变了什么” → “现在怎么选” **三层讲清楚。

##### 一、`struct rw_semaphore`：**有没有被更新 / 废弃？**

首先，**结构体仍然存在**、**核心 API 仍然可用**、**大量子系统仍在用（VFS / mm / fs）**、 **没有被 mutex 取代** 且 **没有被标记 deprecated**，所以**可以放心在驱动中使用**。

~~~c
struct rw_semaphore sem;

init_rwsem(&sem);

down_read(&sem);
up_read(&sem);

down_write(&sem);
up_write(&sem);

down_read_trylock()
down_write_trylock()
~~~

----------

##### 二、虽然 **API 没怎么变**，但 **内核实现和语义细节变了很多**。

1. **内部实现的“进化”（你看不到，但很重要）**

早期内核（2.6 时代）

- 等待队列 + 自旋锁
- 偏向读者（reader bias）
- 容易 **写者饥饿**

新内核（4.x → 6.x）

- 更复杂的 state + 原子操作
- 引入 **公平性策略**
- 减少写者 starvation
- 优化 fast-path（无竞争时非常快）

📌 **API 不变 ≠ 行为完全一致**

----------------

2. **公平性策略的变化（面试常考）**

~~~(空)
R R R R R R R ...
        W（一直进不来）
~~~

新内核的策略

- 一旦有 writer 等待
- **阻止新的 reader 进入**
- 让 writer 尽快获得锁

👉 **读性能略降，但系统整体更可控**

--------

3. **抢占 & RT 内核下的变化**

在：`CONFIG_PREEMPT`、`CONFIG_PREEMPT_RT`场景下：

- rwsem 内部可能转化为可睡眠锁
- 行为更接近 mutex（但仍区分读 / 写）

--------

##### 三、API 层面有没有“新 / 旧之分”？有一点

推荐使用的“新风格”并不是新函数，而是**新选择**以前（书中 / 老代码）

~~~c
struct rw_semaphore sem;
~~~

**现在更常见的替代方案（取决于场景）** **rwsem 不再是“第一反应”**

| 场景             | 更推荐       |
| ---------------- | ------------ |
| 读多写少，数据小 | RCU          |
| 临界区很短       | seqlock      |
| 普通配置路径     | mutex        |
| 真正并发读       | rw_semaphore |

一些“看起来新”的辅助 API（但不是必须），例如：

```
downgrade_write(&sem);  // 写锁降级为读锁
```

老内核就有，但现在用得更多（减少释放/重加锁）。

-----------------

##### 四、在驱动里，现在该不该用 `rw_semaphore`？

**适合用的场景**

- **读操作频繁**
- **写操作很少**
- **临界区可能睡眠**
- **读操作可以并发**

例如：

- 设备配置表
- 路由 / 规则表（你做融合通信时很常见）
- 状态缓存

------

❌ **不适合的场景（非常重要）**

| 场景         | 原因         |
| ------------ | ------------ |
| 临界区极短   | mutex 更快   |
| 写操作频繁   | rwsem 反而慢 |
| 数据结构简单 | 过度设计     |
| 中断上下文   | ❌ 不能用     |

------------------



## 5.4 Completions 机制

内核编程的一个普通模式包括在当前线程之外初始化某个动作, 接着等待这个动作结束。这个动作可能是创建一个新内核线程或者用户空间进程, 对一个存在着的进程的请求, 或者一些基于硬件的动作。在这些情况中, 很有诱惑去使用一个旗标来同步 2 个任务, 使用这样的代码:

~~~c
struct semaphore sem;
init_MUTEX_LOCKED(&sem);
start_external_task(&sem);
down(&sem);
// up(&sem)
/* demo 中已经替换为互斥量了 */
struct mutex lock;
mutex_init(&scull_devices[i].lock);
start_external_task(dev);
if (mutex_lock_interruptible(&dev->lock))
    return -ERESTARTSYS;
// mutex_unlock(&dev->lock);
~~~

外部任务可以接着调用`up(&sem)`, 在它的工作完成时。

事实证明, 这种情况旗标不是最好的工具。正常使用中, 试图加锁一个旗标的代码发现旗标几乎在所有时间都可用; 如果对旗标有很多竞争, 性能会受损并且加锁方案需要重新审视。因此旗标已经对"可用"情况做了很多的优化。当用上面展示的方法来通知任务完成,然而, 调用 `down` 的线程将几乎是一直不得不等待; 因此性能将受损。旗标还可能易于处于一个( 困难的 ) 竞争情况, 如果它们表明为自动变量以这种方式使用时。在一些情况中, 旗标可能在调用 `up` 的进程用完它之前消失。

这些问题引起了在 `2.4.7` 内核中增加了 "`completion`" 接口。 `completion` 是任务使用的一个轻量级机制: 允许一个线程告诉另一个线程工作已经完成。为使用 `completion`, 你的代码必须包含 `<linux/completion.h>`。一个 `completion` 可被创建, 使用:

~~~c
DECLARE_COMPLETION(my_completion);
/* demo 中 */
DECLARE_COMPLETION(comp);

// 或者, 如果 completion 必须动态创建和初始化:
struct completion my_completion; /* ... */
init_completion(&my_completion);
// 等待 completion 是一个简单事来调用:
void wait_for_completion(struct completion *c);
~~~

注意这个函数进行一个不可打断的等待。如果你的代码调用 `wait_for_completion` 并且没有人完成这个任务, 结果会是一个不可杀死的进程。另一方面, 真正的 `completion` 事件可能通过调用下列之一来发出:

~~~c
void complete(struct completion *c);
void complete_all(struct completion *c);
~~~

如果多于一个线程在等待同一个 `completion` 事件, 这 2 个函数做法不同. `complete` 只唤醒一个等待的线程, 而 `complete_all` 允许它们所有都继续。在大部分情况下, 只有一个等待者, 这 2 个函数将产生一致的结果。

一个 `completion` 正常地是一个单发设备; 使用一次就放弃。然而, 如果采取正确的措施重新使用 `completion` 结构是可能的。如果没有使用 `complete_all`, 重新使用一个`completion` 结构没有任何问题, 只要对于发出什么事件没有模糊。如果你使用`complete_all`, 然而, 你必须在重新使用前重新初始化 `completion` 结构。宏定义:

~~~c
INIT_COMPLETION(struct completion c);
~~~

可用来快速进行这个初始化。作为如何使用 `completion` 的一个例子, 考虑 `complete` 模块, 它包含在例子源码里. 这个模块使用简单的语义定义一个设备: 任何试图从一个设备读的进程将等待(使用`wait_for_completion`)直到其他进程向这个设备写。实现这个行为的代码是:

~~~c
ssize_t complete_read (struct file *filp, char __user *buf, 
                       size_t count, loff_t *pos)
{
	printk(KERN_DEBUG "process %i (%s) going to sleep\n",
			current->pid, current->comm);
	wait_for_completion(&comp);
	printk(KERN_DEBUG "awoken %i (%s)\n", current->pid, current->comm);
	return 0; /* EOF */
}

ssize_t complete_write (struct file *filp, const char __user *buf, 
                        size_t count, loff_t *pos)
{
	printk(KERN_DEBUG "process %i (%s) awakening the readers...\n",
			current->pid, current->comm);
	complete(&comp);
	return count; /* succeed, to avoid retrial */
}
~~~

有多个进程同时从这个设备"读"是有可能的。每个对设备的写将确切地使一个读操作完成,但是没有办法知道会是哪个。

`completion` 机制的典型使用是在模块退出时与内核线程的终止一起。在这个原型例子里,一些驱动的内部工作是通过一个内核线程在一个 while(1) 循环中进行的。当模块准备好被清理时, exit 函数告知线程退出并且等待结束。为此目的, 内核包含一个特殊的函数给线程使用:

~~~c
void complete_and_exit(struct completion *c, long retval);
~~~

`completion` 机制使用的时候需要小心，`complete_all()` 会把 completion 变成“永久完成态”。

> **内部语义（重要）**
>
> `struct completion` 里有一个 `done` 计数：
>
> - 初始：`done = 0`
> - `complete()`：`done++`
> - `complete_all()`：`done = UINT_MAX`

也就是说：`complete_all()` 之后：`done` 永远 `> 0` 。

**正确用法**：场景：驱动初始化完成通知多个等待者

~~~c
struct completion init_done;
/* 初始化 */
INIT_COMPLETION(init_done); 
/* 多个线程等待 */
wait_for_completion(&init_done);
pr_info("init finished\n");
/* 初始化完成，唤醒所有等待者 */
complete_all(&init_done);
/********************这一轮同步结束*****************************/
/* 如果你还想“下一轮再用” */
complete_all(&init_done);

/* 准备下一轮 */
INIT_COMPLETION(init_done);

wait_for_completion(&init_done);  // 正确阻塞

~~~

| API              | done 变化         | 唤醒谁      | 是否可重复      |
| ---------------- | ----------------- | ----------- | --------------- |
| `complete()`     | `done++`          | 一个 waiter | ⚠️ 小心          |
| `complete_all()` | `done = UINT_MAX` | 所有 waiter | ❌ 必须重新 init |

**工程级正确用法模板（推荐）**

~~~c
mutex_lock(&state_lock);

INIT_COMPLETION(init_done);
start_async_work();

mutex_unlock(&state_lock);

wait_for_completion(&init_done);

~~~

异步完成路径：

```
complete_all(&init_done);
```

----------



## 5.5 自旋锁

自旋锁概念上简单。一个自旋锁是一个互斥设备, 只能有 2 个值:"上锁"和"解锁"。它常常实现为一个整数值中的一个单个位。想获取一个特殊锁的代码测试相关的位。如果锁是可用的, 这个"上锁"位被置位并且代码继续进入临界区。相反, 如果这个锁已经被别人获得, 代码进入一个紧凑的循环中反复检查这个锁, 直到它变为可用。这个循环就是自旋锁的"自旋"部分。

当然, 一个自旋锁的真实实现比上面描述的复杂一点。这个"测试并置位"操作必须以原子方式进行, 以便只有一个线程能够获得锁, 就算如果有多个进程在任何给定时间自旋。必须小心以避免在超线程处理器上死锁 -- 实现多个虚拟 CPU 以共享一个单个处理器核心和缓存的芯片。因此实际的自旋锁实现在每个 Linux 支持的体系上都不同。核心的概念在所有系统上相同。然而, 当有对自旋锁的竞争, 等待的处理器在一个紧凑循环中执行并且不作有用的工作。

它们的特性上, 自旋锁是打算用在多处理器系统上, 尽管一个运行一个抢占式内核的单处理器工作站的行为如同 SMP, 如果只考虑到并发。如果一个非抢占的单处理器系统进入一个锁上的自旋, 它将永远自旋; 没有其他的线程再能够获得 CPU 来释放这个锁。因此,自旋锁在没有打开抢占的单处理器系统上的操作被优化为什么不作, 除了改变 IRQ 屏蔽状态的那些。由于抢占, 甚至如果你从不希望你的代码在一个 SMP 系统上运行, 你仍然需要实现正确的加锁。

### 5.5.1 自旋锁 API 简介

自旋锁原语要求的包含文件是 `<linux/spinlock.h>`。一个实际的锁有类型 `spinlock_t`。象任何其他数据结构, 一个 自旋锁必须初始化。这个初始化可以在编译时完成, 如下:

~~~c
spinlock_t my_lock = SPIN_LOCK_UNLOCKED;
~~~

或者在运行时使用:

~~~c
void spin_lock_init(spinlock_t *lock);
~~~

在进入一个临界区前, 你的代码必须获得需要的 lock , 用:

~~~c
void spin_lock(spinlock_t *lock);
~~~

注意所有的自旋锁等待是, 由于它们的特性, 不可中断的。一旦你调用 `spin_lock`, 你将自旋直到锁变为可用。为释放一个你已获得的锁, 传递它给:

~~~c
void spin_unlock(spinlock_t *lock);
~~~

有很多其他的自旋锁函数, 我们将很快都看到。但是没有一个背离上面列出的函数所展示的核心概念。除了加锁和释放, 没有什么可对一个锁所作的。但是, 有几个规则关于你必须如何使用自旋锁。我们将用一点时间来看这些, 在进入完整的自旋锁接口之前。

### 5.5.2 自旋锁和原子上下文

> 想象一会儿你的驱动请求一个自旋锁并且在它的临界区里做它的事情。在中间某处, 你的驱动失去了处理器. 或许它已调用了一个函数( copy_from_user, 假设) 使进程进入睡眠。或者, 也许, 内核抢占发威, 一个更高优先级的进程将你的代码推到一边。你的代码现在持有一个锁, 在可见的将来的如何时间不会释放这个锁。如果某个别的线程想获得同一个锁, 它会, 在最好的情况下, 等待( 在处理器中自旋 )很长时间。最坏的情况, 系统可能
> 完全死锁。

大部分读者会同意这个场景最好是避免。因此, 应用到自旋锁的核心规则是任何代码必须,在持有自旋锁时, 是原子性的。**它不能睡眠**; 事实上, 它不能因为任何原因放弃处理器,除了服务中断(并且有时即便此时也不行)。

==其实这种说法并不完全正确，后续将说明“自旋锁”的现代化改动，接口没变但是实现逻辑确实发生了变化。==

内核抢占的情况由自旋锁代码自己处理。内核代码持有一个自旋锁的任何时间, 抢占在相关处理器上被禁止. 即便单处理器系统必须以这种方式禁止抢占以避免竞争情况。这就是为什么需要正确的加锁, 即便你从不期望你的代码在多处理器机器上运行。

在持有一个锁时避免睡眠是更加困难; 很多内核函数可能睡眠, 并且这个行为不是都被明确记录了。拷贝数据到或从用户空间是一个明显的例子: 请求的用户空间页可能需要在拷贝进行前从磁盘上换入, 这个操作显然需要一个睡眠。必须分配内存的任何操作都可能睡眠。kmalloc 能够决定放弃处理器, 并且等待更多内存可用除非它被明确告知不这样做。睡眠可能发生在令人惊讶的地方; 编写会在自旋锁下执行的代码需要注意你调用的每个函数。

这有另一个场景: 你的驱动在执行并且已经获取了一个锁来控制对它的设备的存取。当持有这个锁时, 设备发出一个中断, 使得你的中断处理运行。中断处理, 在存取设备之前,必须获得锁。在一个中断处理中获取一个自旋锁是一个要做的合法的事情; 这是自旋锁操作不能睡眠的其中一个理由。但是如果中断处理和起初获得锁的代码在同一个处理器上会发生什么? 当中断处理在自旋, 非中断代码不能运行来释放锁。这个处理器将永远自旋。

避免这个陷阱需要在持有自旋锁时禁止中断( 只在本地 CPU )。有各种自旋锁函数会为你禁止中断( 我们将在下一节见到它们 )。但是, 一个完整的中断讨论必须等到第 10 章了。

关于自旋锁使用的最后一个重要规则是自旋锁必须一直是尽可能短时间的持有。你持有一个锁越长, 另一个进程可能不得不自旋等待你释放它的时间越长, 它不得不完全自旋的机会越大。长时间持有锁也阻止了当前处理器调度, 意味着高优先级进程 -- 真正应当能获得 CPU 的 -- 可能不得不等待。内核开发者尽了很大努力来减少内核反应时间( 一个进程可能不得不等待调度的时间 )在 `2.5` 开发系列。一个写的很差的驱动会摧毁所有的进程, 仅仅通过持有一个锁太长时间。为避免产生这类问题, 重视使你的锁持有时间短。

#### 自旋锁的变化（底层实现）

**原来：raw spinlock 时代（老 2.6）**

- 自旋 = 关抢占 + 忙等
- SMP / UP 差异大
- 对 RT / PREEMPT 支持弱

------

**现在：spinlock 是“语义层”，底下很复杂**

现代内核中：

```
spinlock_t
 └─ raw_spinlock_t
     └─ arch_spinlock_t
```

内部已经包含：

- 抢占控制
- 内存屏障
- 锁调试（lockdep）
- RT 适配
- NUMA / cache 优化

**spinlock 不再只是“while(locked)”**

---------------

**最大的变化在 “PREEMPT_RT” 语义改变，在普通内核（非 RT）**

```
spin_lock(&lock);
```

语义是：**禁止抢占**、**忙等**、**不能睡眠**。

在 `CONFIG_PREEMPT_RT` 内核中（关键变化）：

> **`spinlock_t` 可能“退化为 mutex”**

也就是说：`spin_lock()` **可能睡眠**、`spinlock_t` ≈ `rt_mutex`

 这是为了：**实时性**、**降低最坏延迟**。

⚠️ 影响巨大的一条规则：

> **不能再假设 spin_lock 一定不会睡眠**

----------------------

如果想保证**真的不发生睡眠**，用 **`raw_spinlock_t`** ， **raw_spinlock 永远忙等，不会变** 

```
raw_spinlock_t lock;

raw_spin_lock(&lock);
raw_spin_unlock(&lock);
```

| 锁类型         | RT 内核      | 是否睡眠 |
| -------------- | ------------ | -------- |
| spinlock_t     | 可能变 mutex | ⚠️ 可能   |
| raw_spinlock_t | 不变         | ❌ 不会   |

**新规则：spinlock 里“逻辑上不能睡眠”，而不是“实现上”**

旧思维（错误）：

> spinlock = 永不睡眠

新思维（正确）：

> **spinlock = 保护“不能并发”的临界区语义**

-----------------------

现代内核在 `<linux/spinlock.h>` 里集成了：

**lockdep（死锁检测）、might_sleep() 检查、DEBUG_SPINLOCK、CONFIG_PROVE_LOCKING**

所以现在：

```
spin_lock(&lock);
msleep(10);   // 新内核直接 WARN / BUG
```

**比老内核“更严格”**，参考使用选择：

~~~(空)
中断 / 硬实时        → raw_spinlock
普通原子路径         → spinlock
进程上下文互斥       → mutex
一次性事件           → completion
读多写少             → rwsem / RCU

~~~

#### PREEMPT_RT 补充说明

`PREEMPT_RT` **是一个内核“实时化配置体系”**， **在代码里主要体现为 `CONFIG_PREEMPT_RT` 及相关条件编译**。

核心目标是：**把 Linux 从“尽力而为调度”变成“可预测、低延迟、硬实时友好”的系统**。

关键词：**最坏延迟可控**、**调度点更多**、**减少关中断 / 忙等**。

差异：

**普通内核（CONFIG_PREEMPT / VOLUNTARY）**

- 内核态 **大量不可抢占**
- spinlock = 忙等 + 关抢占
- 中断处理时间可能很长
- 最坏延迟不可预测

------

**RT 内核（CONFIG_PREEMPT_RT）**

- **几乎整个内核可抢占**
- 中断线程化（IRQ thread）
- spinlock **变为 rt_mutex**
- 关中断时间大幅缩短

 **实时性来自“可抢占 + 可睡眠”**

-------------------

**PREEMPT_RT 打开后，内核发生的变化**：

1. **中断线程化（这是最大变化）**

普通内核

```
硬中断
 └─ 直接在中断上下文执行 handler
```

RT 内核

```
硬中断（极短）
 └─ 唤醒 IRQ thread（可调度）
```

效果：

- 中断 handler 可以睡眠
- 可以被高优先级任务抢占
- 延迟可控

---------------

2. **spinlock 的语义变化（你之前问到的）**

非 RT

```
spin_lock();  // 忙等，不睡眠
```

RT

```
spin_lock();  // 可能睡眠（rt_mutex）
```

这就是为什么 **RT 内核下要区分 raw_spinlock**

--------------

3. **大量“关中断区”被消灭**

RT 会把：

- `local_irq_disable()`
- `spin_lock_irqsave()`

缩到**最小必要范围**

------

4. **优先级继承（PI）成为常态**

RT mutex 自动支持：

- Priority Inheritance
- 防止优先级反转

-----------------

### 5.5.3 自旋锁函数

我们已经看到 2 个函数, `spin_lock` 和 `spin_unlock`, 可以操作自旋锁。有其他几个函数, 然而, 有类似的名子和用途。我们现在会展示全套。这个讨论将带我们到一个我们无法在几章内适当涵盖的地方; 自旋锁 API 的完整理解需要对中断处理和相关概念的理解.实际上有 4 个函数可以加锁一个自旋锁:

~~~c
void spin_lock(spinlock_t *lock);
void spin_lock_irqsave(spinlock_t *lock, unsigned long flags);
void spin_lock_irq(spinlock_t *lock);
void spin_lock_bh(spinlock_t *lock)；
~~~

我们已经看到自旋锁如何工作。`spin_loc_irqsave` 禁止中断(只在本地处理器)在获得自旋锁之前; 之前的中断状态保存在 `flags` 里。如果你绝对确定在你的处理器上没有禁止中断的(或者, 换句话说, 你确信你应当在你释放你的自旋锁时打开中断), 你可以使用`spin_lock_irq` 代替, 并且不必保持跟踪 `flags`。最后, `spin_lock_bh` 在获取锁之前禁止软件中断, 但是硬件中断留作打开的。

如果你有一个可能被在(硬件或软件)中断上下文运行的代码获得的自旋锁, 你必须使用一种 spin_lock 形式来禁止中断。其他做法可能死锁系统, 迟早. 如果你不在硬件中断处理里存取你的锁, 但是你通过软件中断(例如, 在一个 tasklet 运行的代码, 在第 7 章涉及的主题 ), 你可以使用 spin_lock_bh 来安全地避免死锁, 而仍然允许硬件中断被服
务。

也有 4 个方法来释放一个自旋锁; 你用的那个必须对应你用来获取锁的函数。

~~~c
void spin_unlock(spinlock_t *lock);
void spin_unlock_irqrestore(spinlock_t *lock, unsigned long flags);
void spin_unlock_irq(spinlock_t *lock);
void spin_unlock_bh(spinlock_t *lock);
~~~

每个 `spin_unlock` 变体恢复由对应的 `spin_lock` 函数锁做的工作。传递给`spin_unlock_irqrestore` 的 `flags` 参数必须是传递给 `spin_lock_irqsave` 的同一个变量。你必须也调用 `spin_lock_irqsave` 和 `spin_unlock_irqrestore` 在同一个函数里。否则, 你的代码可能破坏某些体系。

还有一套非阻塞的自旋锁操作:

~~~c
int spin_trylock(spinlock_t *lock);
int spin_trylock_bh(spinlock_t *lock);
~~~

这些函数成功时返回非零( 获得了锁 ), 否则 0. 没有"try"版本来禁止中断。

#### 函数用法解释

**==补充说明：==**

1️⃣ **`spin_lock(spinlock_t *lock)`**

**做什么**：获取自旋锁、禁止内核抢占、**不关中断**

**不能防什么**：❌ 本 CPU 的硬中断、❌ softirq

**使用场景**

- 只在 **中断上下文**
- 或 **不会被中断重入的进程上下文**

**示例**

```
spin_lock(&lock);
/* 临界区 */
spin_unlock(&lock);
```

------

2️⃣ `spin_lock_irqsave(spinlock_t *lock, unsigned long flags)`

**做什么**：保存当前 CPU 中断状态到 `flags`、关闭本地中断、获取自旋锁

**特点**：防硬中断重入、可嵌套（依赖 flags）、最通用、最安全

**使用场景（最常见）**

- **进程上下文 + 中断上下文共用一把锁**

**示例**

```
unsigned long flags;

spin_lock_irqsave(&lock, flags);
/* 临界区 */
spin_unlock_irqrestore(&lock, flags);
```

------

3️⃣ `spin_lock_irq(spinlock_t *lock)`

**做什么**：直接关闭本地中断、获取自旋锁、**不保存中断状态**

**特点**：❌ 不能嵌套、❌ 不可恢复原中断状态

**使用场景**

- 确定当前上下文 **中断一定是开的**
- 内核内部少量场景

**示例**

```
spin_lock_irq(&lock);
/* 临界区 */
spin_unlock_irq(&lock);
```

------

4️⃣ `spin_lock_bh(spinlock_t *lock)`

**做什么**：禁止 **softirq / tasklet（BH）**、获取自旋锁、**不关硬中断**

**特点**：✅ 防 softirq 重入、❌ 防不了硬中断

**使用场景**

- **进程上下文 + softirq 共用锁**
- 网络子系统非常常见

**示例**

```
spin_lock_bh(&lock);
/* 临界区 */
spin_unlock_bh(&lock);
```

------

| API                 | 关硬中断 | 关 softirq | 保存状态 | 典型场景       |
| ------------------- | -------- | ---------- | -------- | -------------- |
| `spin_lock`         | ❌        | ❌          | ❌        | 纯中断上下文   |
| `spin_lock_irqsave` | ✅        | ❌          | ✅        | 进程 + 中断    |
| `spin_lock_irq`     | ✅        | ❌          | ❌        | 特殊内核路径   |
| `spin_lock_bh`      | ❌        | ✅          | ❌        | 进程 + softirq |

------

```
进程 + 中断    → spin_lock_irqsave
进程 + softirq → spin_lock_bh
只有中断      → spin_lock
不确定？       → irqsave（保命）
```

> `spin_lock` 只防并发不防中断；
>
> `spin_lock_irqsave` 防硬中断且最安全；
>
> `spin_lock_irq` 少用；
>
> `spin_lock_bh` 专防 softirq。

-----------

**`spin_trylock*()` 是“试着拿锁，拿不到就立刻返回”，不会自旋、不睡眠。**

1️⃣ `int spin_trylock(spinlock_t *lock);`

做什么：**尝试**获取自旋锁、**不自旋等待**、**不关中断、不关 softirq**

返回值

- `1`：成功拿到锁
- `0`：锁已被占用，立即返回

使用场景

- 不允许阻塞、也不想忙等
- 快速检查 / 旁路路径
- 中断或性能敏感代码

示例

```
if (spin_trylock(&lock)) {
    /* 临界区 */
    spin_unlock(&lock);
} else {
    /* 拿不到锁，直接走其他路径 */
}
```

注意：❌ 防不了中断重入、❌ 若进程和中断共用锁，仍可能死锁

------

2️⃣ `int spin_trylock_bh(spinlock_t *lock);`

做什么：**禁止 softirq（BH）**、尝试获取自旋锁、**不自旋等待**

返回值

- `1`：成功
- `0`：失败

使用场景

- **进程上下文 + softirq 共用锁**
- 网络路径常见
- 想避免在 softirq 里死锁，但又不想自旋

示例

```
if (spin_trylock_bh(&lock)) {
    /* 临界区 */
    spin_unlock_bh(&lock);
} else {
    /* 拿不到锁，直接返回 */
}
```

------

| API               | 是否自旋 | 关硬中断 | 关 softirq | 典型用途       |
| ----------------- | -------- | -------- | ---------- | -------------- |
| `spin_trylock`    | ❌        | ❌        | ❌          | 快速试探       |
| `spin_trylock_bh` | ❌        | ❌        | ✅          | 进程 + softirq |

------

✅ 适合：临界区**可选**、有备用路径、不希望增加延迟

❌ 不适合：锁必须拿到、需要强一致性、与硬中断共享锁（应使用 irqsave）

> **`spin_trylock()` 只是尝试拿锁；**
>
> **`spin_trylock_bh()` 在此基础上禁止 softirq；**
>
> **两者都不会自旋等待，拿不到锁就立即返回。**

--------------



### 5.5.4 读者/写者自旋锁

内核提供了一个自旋锁的读者/写者形式, 直接模仿我们在本章前面见到的读者/写者旗标.这些锁允许任何数目的读者同时进入临界区, 但是写者必须是排他的存取。读者写者锁有一个类型 `rwlock_t`, 在 `<linux/spinlokc.h>` 中定义。它们可以以 2 种方式被声明和被初始化:

~~~c
rwlock_t my_rwlock = RW_LOCK_UNLOCKED; /* Static way */
rwlock_t my_rwlock;
rwlock_init(&my_rwlock); /* Dynamic way */
~~~

可用函数的列表现在应当看来相当类似。对于读者, 下列函数是可用的:

~~~c
void read_lock(rwlock_t *lock);
void read_lock_irqsave(rwlock_t *lock, unsigned long flags);
void read_lock_irq(rwlock_t *lock);
void read_lock_bh(rwlock_t *lock);
void read_unlock(rwlock_t *lock);
void read_unlock_irqrestore(rwlock_t *lock, unsigned long flags);
void read_unlock_irq(rwlock_t *lock);
~~~

有趣地, 没有 `read_trylock`。对于写存取的函数是类似的:

~~~c
void write_lock(rwlock_t *lock);
void write_lock_irqsave(rwlock_t *lock, unsigned long flags);
void write_lock_irq(rwlock_t *lock);
void write_lock_bh(rwlock_t *lock);
int write_trylock(rwlock_t *lock);
void write_unlock(rwlock_t *lock);
void write_unlock_irqrestore(rwlock_t *lock, unsigned long flags);
void write_unlock_irq(rwlock_t *lock);
void write_unlock_bh(rwlock_t *lock);
~~~

读者/写者锁能够饿坏读者, 就像 `rwsem` 一样。这个行为很少是一个问题; 然而, 如果有足够的锁竞争来引起饥饿, 性能无论如何都不行。



## 5.6 锁陷阱

多年使用锁的经验 -- 早于 Linux 的经验 -- 已经表明加锁可能是非常难于正确的。管理并发是一个固有的技巧性的事情, 有很多出错的方式。在这一节, 我们快速看一下可能出错的东西。

### 5.6.1 模糊的规则

如同上面已经说过的, 一个正确的加锁机制需要清晰和明确的规则。当你创建一个可以被并发存取的资源时, 你应当定义哪个锁将控制存取。加锁应当真正在开始处进行; 事后更改会是难的事情。开始时花费的时间常常在调试时获得回报。

当你编写你的代码, 你会毫无疑问遇到几个函数需要存取通过一个特定锁保护的结构。在此, 你必须小心: 如果一个函数需要一个锁并且接着调用另一个函数也试图请求这个锁,你的代码死锁。不论旗标还是自旋锁都不允许一个持锁者第 2 次请求锁; 如果你试图这样做, 事情就简单地完了。

为使的加锁正确工作, 你不得不编写一些函数, 假定它们的调用者已经获取了相关的锁。常常地, 只有你的内部的, 静态函数能够这样编写; 从外部调用的函数必须明确处理加锁。当你编写内部函数对加锁做了假设, 方便自己(和其他使用你的代码的人)并且明确记录这些假设。在几个月后可能很难回来并记起是否你需要持有一个锁来调用一个特殊函数。

在 sucll 的例子里, 采用的设计决定是要求所有的函数直接从系统调用里调用, 来请求应用到被存取的设备结构上的旗标。所有的内部函数, 那些只是从其他 scull 函数里调用的, 可以因此假设旗标已经正确获得。



### 5.6.2 加锁顺序规则

在有大量锁的系统中(并且内核在成为这样一个系统), 一次需要持有多于一个锁, 对代码是不寻常的。如果某类计算必须使用 2 个不同的资源进行, 每个有它自己的锁, 常常没有选择只能获取 2 个锁。

获得多个锁可能是危险的, 如果你有 2 个锁, 称为 Lock1 和 Lock2, 代码需要同时都获取, 你有一个潜在的死锁。仅仅想象一个线程锁住 Lock1 而另一个同时获得Lock2。接着每个线程试图得到它没有的那个 2 个线程都会死锁。

这个问题的解决方法常常是简单的: **当多个锁必须获得时, 它们应当一直以同样顺序获得。**只要遵照这个惯例, 象上面描述的简单死锁能够避免。然而, 遵照加锁顺序规则是做比说难。非常少见这样的规则真正在任何地方被写下。常常你能做的最好的是看看别的代码如何做的。

一些经验规则能帮上忙。如果你必须获得一个对你的代码来说的本地锁(假如, 一个设备锁), 以及一个属于内核更中心部分的锁, 先获取你的。如果你有一个旗标和自旋锁的组合, 你必须,先获得旗标; 调用 down (可能睡眠) 在持有一个自旋锁时是一个严重的错误。但是最重要的, 尽力避免需要多于一个锁的情况。



### 5.6.3 细 -粗- 粒度加锁

第一个支持多处理器系统的 Linux 内核是 `2.0`; 它只含有一个自旋锁。这个大内核锁将整个内核变为一个大的临界区; 在任何时候只有一个 CPU 能够执行内核代码。这个锁足够好地解决了并发问题以允许内核开发者从事所有其他的开发 SMP 所包含的问题。但是它不是扩充地很好。甚至一个 2 个处理器的系统可能花费可观数量的时间只是等待这个大内核锁。一个 4 个处理器的系统的性能甚至不接近 4 个独立的机器的性能。

因此, 后续的内核发布已经包含了更细粒度的加锁。在 `2.2` 中, 一个自旋锁控制对块`I/O` 子系统的存取; 另一个为网络而工作, 等等. 一个现代的内核能包含几千个锁, 每个保护一个小的资源。这种细粒度的加锁可能对伸缩性是好的; 它允许每个处理器在它自己特定的任务上工作而不必竞争其他处理器使用的锁。很少人忘记大内核锁。

但是, 细粒度加锁带有开销。 在有几千个锁的内核中, 很难知道你需要那个锁 -- 以及你应当以什么顺序获取它们 -- 来进行一个特定的操作。记住加锁错误可能非常难发现; 更多的锁提供了更多的机会使真正有害的加锁 bug 钻进内核中。细粒度加锁能带来一定水平的复杂性, 长期来, 对内核的可维护性有一个大的, 不利的效果。

在一个设备驱动中加锁常常是相对直接的; 你可以用一个锁来涵盖你做的所有东西, 或者你可以给你管理的每个设备创建一个锁。作为一个通用的规则, 你应当从相对粗的加锁开始, 除非你有确实的理由相信竞争可能是一个问题。忍住怂恿去过早地优化; 真实地性能约束常常表现在想不到的地方。

如果你确实怀疑锁竞争在损坏性能, 你可能发现 lockmeter 工具有用。这个补丁(从http://oss.sgi.com/projects/lockmeter/ 可得到) 装备内核来测量在锁等待花费的时间。通过看这个报告, 你能够很快知道是否锁竞争真的是问题。



## 5.7 加锁的各种选择

Linux 内核提供了不少有力的加锁原语能够用来使内核避免被自己绊倒。但是, 如同我们已见到的, 一个加锁机制的设计和实现不是没有缺陷。常常对于旗标和自旋锁没有选择;它们可能是唯一的方法来正确地完成工作。然而, 有些情况, 可以建立原子的存取而不用完整的加锁。本节看一下做事情的其他方法。

### 5.7.1 不加锁算法

有时, 你可以重新打造你的算法来完全避免加锁的需要。许多读者/写者情况 -- 如果只有一个写者 -- 常常能够在这个方式下工作。如果写者小心使数据结构的视图, 由读者所见的, 是一直一致的, 有可能创建一个不加锁的数据结构。常常可以对无锁的生产者/消费者任务有用的数据结构是环形缓存。这个算法包含一个生产者安放数据到一个数组的尾端, 而消费者从另一端移走数据。当到达数组末端, 生产者绕回到开始。因此一个环形缓存需要一个数组和 2 个索引值来跟踪下一个新值放到哪里,以及哪个值在下一次应当从缓存中移走。

当小心地实现了, 一个环形缓存在没有多个生产者或消费者时不需要加锁。生产者是唯一允许修改写索引和它所指向的数组位置的线程。只要写者在更新写索引之前存储一个新值到缓存中, 读者将一直看到一个一致的视图。读者, 轮换地, 是唯一存取读索引和它指向的值的线程。加一点小心到确保 2 个指针不相互覆盖, 生产者和消费者可以并发存取缓存而没有竞争情况。



~~~(空)
 index:   0   1   2   3   4   5   6   7
        +---+---+---+---+---+---+---+---+
buffer: |   |   |   |   |   |   |   |   |
        +---+---+---+---+---+---+---+---+
(mask = 7)
write_seq=0 -> index 0
read_seq =0 -> index 0

buffer:
        +---+---+---+---+---+---+---+---+
index:  | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
        +---+---+---+---+---+---+---+---+
data:   |   |   |   |   |   |   |   |   |
------------------------------
写三个
buffer:
        +---+---+---+---+---+---+---+---+
index:  | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
        +---+---+---+---+---+---+---+---+
data:   | A | B | C |   |   |   |   |   |
          ^
        read_seq
读两个
buffer:
        +---+---+---+---+---+---+---+---+
index:  | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
        +---+---+---+---+---+---+---+---+
data:   | A | B | C |   |   |   |   |   |
                  ^
              read_seq
绕回到开头
buffer:
        +---+---+---+---+---+---+---+---+
index:  | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
        +---+---+---+---+---+---+---+---+
data:   | I | J | C | D | E | F | G | H |
                  ^
             read_seq (2)

------------------------------

 
 					LOGICAL SEQUENCE SPACE
      ------------------------------------------------->
      ...  6   7   8   9  10  11  12  13  ...

                   |   |   |   |   |
                   v   v   v   v   v
              +---------------------------------+
array index:  |  6 |  7 |  0 |  1 |  2 |  ...   |
              +---------------------------------+
                   ^                   ^
                read_seq            write_seq

~~~

**总结：**

>  **环形缓冲区不是“转圈的指针”**
>
> **它是“一条无限延伸的数轴”**
>
> **数组只是这条数轴的一个窗口**

在设备驱动中环形缓存出现相当多。网络适配器, 特别地, 常常使用环形缓存来与处理器交换数据(报文)。注意, 对于 2.6.10, 有一个通用的环形缓存实现在内核中可用; 如何使用它的信息看 <linux/kfifo.h>。

#### 程序实例

**==后续补充代码示例==**



### 5.7.2 原子变量

有时, 一个共享资源是一个简单的整数值。假设你的驱动维护一个共享变量 n_op, 它告知有多少设备操作目前未完成。正常地, 即便一个简单的操作例如:

~~~c
n_op++;
~~~

可能需要加锁. 某些处理器可能以原子的方式进行那种递减, 但是你不能依赖它。但是一个完整的加锁体制对于一个简单的整数值看来过分了。对于这样的情况, 内核提供了一个原子整数类型称为 atomic_t, 定义在 `<asm/atomic.h>`。

一个 `atomic_t` 持有一个 `int` 值在所有支持的体系上。但是, 因为这个类型在某些处理器上的工作方式, 整个整数范围可能不是都可用的; 因此, 你不应当指望一个 `atomic_t` 持有多于 24 位。下面的操作为这个类型定义并且保证对于一个 SMP 计算机的所有处理器来说是原子的。操作是非常快的, 因为它们在任何可能时编译成一条单个机器指令。

~~~c
typedef struct {
    int counter;
} atomic_t;
~~~

> 这个定义在 `include/linux/types.h` 和 `include/linux/atomic.h` 中声明，表示一个原子计数器。它只是一个包装了 `int counter` 的结构。

------------------------

~~~c
/*
 * atomic_read — wrapper around arch_atomic_read
 */
#define atomic_read(v) arch_atomic_read(&(v)->counter)

/*
 * atomic_set — wrapper around arch_atomic_set
 */
#define atomic_set(v, i) arch_atomic_set(&(v)->counter, (i))

int atomic_read(atomic_t *v);
void atomic_set(atomic_t *v, int i);
atomic_t v = ATOMIC_INIT(0);
~~~

> 在现代内核中，`atomic_read`/`atomic_set` 并不是直接写 asm，而是通过架构相关的底层实现（例如 arm64/x86）加上通用封装。底层的封装定义可以在 `include/asm-generic/atomic-instrumented.h` 中找到,这些 wrapper 会调用架构自己实现的 `arch_atomic_read()`、`arch_atomic_set()`。
>
> `atomic_set` 设置原子变量 v 为整数值 i。你也可在编译时使用宏定义 ATOMIC_INIT 初始化原子值。
>
> `atomic_read` 返回 v 的当前值.

-----

~~~c
void atomic_add(int i, atomic_t *v);
void atomic_sub(int i, atomic_t *v);
~~~

> `atomic_add` 由 v 指向的原子变量加 i. 返回值是 void, 因为有一个额外的开销来返回新值,并且大部分时间不需要知道它。`atomic_sub` 从 *v 减去 i。

------------------------

~~~c
void atomic_inc(atomic_t *v);
void atomic_dec(atomic_t *v);
~~~

> 递增或递减一个原子变量。

------------------------

~~~c
int atomic_inc_and_test(atomic_t *v);
int atomic_dec_and_test(atomic_t *v);
int atomic_sub_and_test(int i, atomic_t *v);
~~~

> 进行一个特定的操作并且测试结果; 如果, 在操作后, 原子值是 0, 那么返回值是真; 否则, 它是假. 注意没有 `atomic_add_and_test`。

------------------------

~~~c
int atomic_add_negative(int i, atomic_t *v);
~~~

> 加整数变量 i 到 v。如果结果是负值返回值是真, 否则为假。

------------------------

~~~c
int atomic_add_return(int i, atomic_t *v);
int atomic_sub_return(int i, atomic_t *v);
int atomic_inc_return(atomic_t *v);
int atomic_dec_return(atomic_t *v);
~~~

> 就像 `atomic_add` 和其类似函数, 除了它们返回原子变量的新值给调用者。

------------------------

如同它们说过的, `atomic_t` 数据项必须通过这些函数存取。如果你传递一个原子项给一个期望一个整数参数的函数, 你会得到一个编译错误。你还应当记住, `atomic_t` 值只在当被置疑的量真正是原子的时候才起作用。需要多个`atomic_t` 变量的操作仍然需要某种其他种类的加锁。考虑一下下面的代码:

~~~c
atomic_sub(amount, &first_atomic);
atomic_add(amount, &second_atomic);
~~~

从第一个原子值中减去 amount, 但是还没有加到第二个时, 存在一段时间。如果事情的这个状态可能产生麻烦给可能在这 2 个操作之间运行的代码, 某种加锁必须采用。



### 5.7.3 位操作

`atomic_t` 类型在进行整数算术时是不错的。但是, 它无法工作的好, 当你需要以原子方式操作单个位时。为此, 内核提供了一套函数来原子地修改或测试单个位。因为整个操作在单步内发生, 没有中断(或者其他处理器)能干扰。

原子位操作非常快, 因为它们使用单个机器指令来进行操作, 而在任何时候低层平台做的时候不用禁止中断。函数是体系依赖的并且在 `<asm/bitops.h>` 中声明。它们保证是原子的, 即便在 SMP 计算机上, 并且对于跨处理器保持一致是有用的。

不幸的是, 键入这些函数中的数据也是体系依赖的。nr 参数(描述要操作哪个位)常常定义为 int, 但是在几个体系中是 `unsigned long`。要修改的地址常常是一个 `unsigned long` 指针, 但是几个体系使用 `void *` 代替。各种位操作是:

~~~c
void set_bit(nr, void *addr);
~~~

> 设置第 nr 位在 addr 指向的数据项为1。

------------------------

~~~c
void clear_bit(nr, void *addr);
~~~

> 清除指定位在 addr 处的无符号长型数据. 它的语义与 `set_bit` 的相反。

------------------------

~~~c
void change_bit(nr, void *addr);
~~~

> 翻转这个位。

------------------------

~~~c
test_bit(nr, void *addr);
~~~

> 这个函数是唯一一个不需要是原子的位操作; 它简单地返回这个位的当前值。

------------------------

~~~c
int test_and_set_bit(nr, void *addr);
int test_and_clear_bit(nr, void *addr);
int test_and_change_bit(nr, void *addr);
~~~

> 原子地动作如同前面列出的, 除了它们还返回这个位以前的值。

------------------------

当这些函数用来存取和修改一个共享的标志, 除了调用它们不用做任何事; 它们以原子发生进行它们的操作。使用位操作来管理一个控制存取一个共享变量的锁变量, 另一方面,是有点复杂并且应该有个例子。大部分现代的代码不以这种方法来使用位操作, 但是象下面的代码仍然在内核中存在。

一段需要存取一个共享数据项的代码试图原子地请求一个锁, 使用`test_and_set_bit`或者`test_and_clear_bit`。通常的实现展示在这里; 它假定锁是在地址 addr 的 nr 位。它还假定当锁空闲是这个位是 0, 忙为 非零。

~~~c
/* try to set lock */
while (test_and_set_bit(nr, addr) != 0)
wait_for_a_while();
/* do your work */
/* release lock, and check... */
if (test_and_clear_bit(nr, addr) == 0)
something_went_wrong(); /* already released: error */
~~~

如果你通读内核源码, 你会发现象这个例子的代码。但是, 最好在新代码中使用自旋锁;自旋锁很好地调试过, 它们处理问题如同中断和内核抢占, 并且别人读你代码时不必努力理解你在做什么。

#### 补充说明

**补丁 0：整体定位（先给结论）**

> **LDD 讲的是“内核机制的思想”，不是“今天推荐的 API 全集”**

- 核心概念：✅ 仍然正确
- 具体接口：⚠️ 部分过时 / 不完整
- 并发语义：❌ 明显不足（这是最大差异）

------

**补丁 1：atomic 接口（语义补丁）**

**📖 LDD 中的写法**

```
atomic_t cnt;
atomic_set(&cnt, 0);
atomic_inc(&cnt);
if (atomic_read(&cnt) > 10) { ... }
```

**⚠️ LDD 的问题**

- 没区分 **原子性 vs 内存可见性**
- 默认读者不知道 **atomic ≠ memory barrier**

----------------

✅ **Linux 6.x 补丁说明**

1. atomic_read / atomic_set 仍然存在

✔ 不用改代码   ❗ 但**不能假设它们有顺序保证**

2. 新增推荐用法（有内存序）

```
atomic_set_release(&cnt, 1);
val = atomic_read_acquire(&cnt);
```

> LDD 的 atomic 用法 →
>  **涉及“状态发布 / 消费”的地方必须升级为 acquire/release**

------

**补丁 2：位操作（bit ops 补丁）**

**📖 LDD 中的写法**

```
set_bit(FLAG_BUSY, &dev->flags);
if (test_bit(FLAG_BUSY, &dev->flags)) { ... }
clear_bit(FLAG_BUSY, &dev->flags);
```

**⚠️ LDD 的问题**

- 没强调 `__set_bit()` 的存在
- 没讲 bit-lock（轻量锁）

------

**✅ Linux 6.x 补丁说明**

1. 原子 vs 非原子明确区分

```
set_bit();      // 并发安全
__set_bit();    // 需要外部锁
```

> 如果 LDD 示例在锁保护下
>  → **应替换为 `__set_bit()`**

------

2. 新增 bit-lock（LDD 没有）

```
if (test_and_set_bit_lock(BUSY, &flags))
    return -EBUSY;

/* critical section */

clear_bit_unlock(BUSY, &flags);
```

📌 用于：状态机、驱动轻量互斥、比 spinlock 成本低。

------

**补丁 3：链表（list）——几乎不用补**

**📖 LDD**

```
struct list_head list;
list_add(&node->list, &head);
```

------

**补丁 4：radix tree → xarray（重大补丁）**

**📖 LDD 中的写法（radix_tree）**

```
radix_tree_insert(&root, index, ptr);
ptr = radix_tree_lookup(&root, index);
```

**❌ Linux 6.x 现状**

- radix_tree **已不推荐**
- 新代码基本不用

------

**✅ Linux 6.x 补丁方案：xarray**

```
DEFINE_XARRAY(xa);

xa_store(&xa, index, ptr, GFP_KERNEL);
ptr = xa_load(&xa, index);
```

> LDD 中所有 radix_tree 示例
>  → **一律替换为 xarray**

------

**补丁 5：并发模型（LDD 最大缺失点）**

**📖 LDD 的隐含模型**

- “加锁即可”
- 很少区分 context

❌ 现代内核不够用

------

**✅ Linux 6.x 补丁说明（非常重要）**

1. 必须明确执行上下文

| 上下文  | 能否睡眠 | 能否用 mutex |
| ------- | -------- | ------------ |
| process | ✅        | ✅            |
| softirq | ❌        | ❌            |
| hardirq | ❌        | ❌            |

> LDD 示例代码
>  → **你必须补充 context 判断**

------

2. RCU（LDD 几乎没讲）

```
rcu_read_lock();
ptr = rcu_dereference(dev->ptr);
rcu_read_unlock();
```

📌 现代内核：网络、VFS、数据路径，都离不开 RCU。

------

**补丁 6：性能意识补丁（LDD 时代不敏感）**

**📖 LDD**

- 示例偏功能正确
- 很少考虑 cache / lock 竞争

**✅ Linux 6.x 必须补的意识**

| LDD 做法    | 现代补丁         |
| ----------- | ---------------- |
| 大锁        | 拆锁 / bit-lock  |
| malloc 频繁 | bitmap / slab    |
| 中断里干活  | NAPI / workqueue |

------

**总结：LDD → Linux 6.x 的“补丁表”**

| LDD 内容   | 是否还能用 | 补丁要点          |
| ---------- | ---------- | ----------------- |
| atomic     | ✅          | 加内存序          |
| bit ops    | ✅          | 区分原子 / 非原子 |
| list       | ✅          | 基本不变          |
| radix tree | ❌          | 换 xarray         |
| 锁         | ⚠️          | 加 context / RCU  |
| 示例代码   | ⚠️          | 性能 & 并发补丁   |



### 5.7.4 seqlock 锁

`2.6` 内核包含了一对新机制打算来提供快速地, 无锁地存取一个共享资源。`seqlock` 在这种情况下工作, 要保护的资源小, 简单, 并且常常被存取, 并且很少写存取但是必须要快。基本上, 它们通过允许读者释放对资源的存取, 但是要求这些读者来检查与写者的冲突而工作, 并且当发生这样的冲突时, 重试它们的存取。`seqlock` 通常不能用在保护包含指针的数据结构, 因为读者可能跟随着一个无效指针而写者在改变数据结构。

`seqlock` 定义在 `<linux/seqlock.h>`。有 2 个通常的方法来初始化一个 `seqlock`( 有`seqlock_t` 类型 ):

~~~c
seqlock_t lock1 = SEQLOCK_UNLOCKED;
seqlock_t lock2;
seqlock_init(&lock2);
~~~

读存取通过在进入临界区入口获取一个(无符号的)整数序列来工作. 在退出时, 那个序列值与当前值比较; 如果不匹配, 读存取必须重试. 结果是, 读者代码象下面的形式:

~~~c
unsigned int seq;
do {
	seq = read_seqbegin(&the_lock);
    /* Do what you need to do */
} while read_seqretry(&the_lock, seq);
~~~

这个类型的锁常常用在保护某种简单计算, 需要多个一致的值。如果这个计算最后的测试表明发生了一个并发的写, 结果被简单地丢弃并且重新计算。如果你的 seqlock 可能从一个中断处理里存取, 你应当使用 IRQ 安全的版本来代替:

~~~c
unsigned int read_seqbegin_irqsave(seqlock_t *lock, unsigned long flags);
int read_seqretry_irqrestore(seqlock_t *lock, unsigned int seq, 
                             unsigned long flags);
~~~

写者必须获取一个排他锁来进入由一个 `seqlock` 保护的临界区。为此, 调用:

~~~c
void write_seqlock(seqlock_t *lock);
~~~

写锁由一个自旋锁实现, 因此所有的通常的限制都适用。调用:

~~~c
void write_sequnlock(seqlock_t *lock);
~~~

来释放锁。因为自旋锁用来控制写存取, 所有通常的变体都可用:

~~~c
void write_seqlock_irqsave(seqlock_t *lock, unsigned long flags);
void write_seqlock_irq(seqlock_t *lock);
void write_seqlock_bh(seqlock_t *lock);
void write_sequnlock_irqrestore(seqlock_t *lock, unsigned long flags);
void write_sequnlock_irq(seqlock_t *lock);
void write_sequnlock_bh(seqlock_t *lock);
~~~

还有一个 `write_tryseqlock` 在它能够获得锁时返回非零。

#### 补充说明

**LDD 时代的隐含假设（当年合理）**

LDD 默认假设：

- 单核或弱并发
- cache coherence 问题较少
- 内存模型没被严格形式化

👉 所以：

- 没讲 acquire / release
- 没讲 CPU 重排序
- 没讲 RCU 对比

------

**seqlock 的本质机制（不论年代）**，先把**机制说清楚**，后面才能理解“补丁”。

```
sequence counter（偶数 = 稳定，奇数 = 正在写）
```

写流程：

```
seq++  → 写数据 → seq++
```

读流程：

```
读 seq → 读数据 → 再读 seq
```

只要 seq：发生变化、或为奇数 → 读者重试

-------------------

**Linux 6.x 时代的 seqlock（补丁版）**

1. **接口：看起来没变，其实语义更严格**

**API 名字几乎没变**：

```
read_seqbegin()
read_seqretry()
write_seqlock()
write_sequnlock()
```

但 **内存语义已经被“补上”了**

------

2. **最重要的变化：内存模型被正式定义**

**LDD 最大的“历史缺失”**

> LDD 没告诉你：
>  **为什么读者一定能看到“完整的一致数据”？**

**现代内核给出的答案**

- `read_seqbegin()` → **acquire 语义**
- `write_sequnlock()` → **release 语义**
- seq counter 使用 `WRITE_ONCE / READ_ONCE`

📌 这保证了：写者对数据的修改、在 seq 更新之前/之后不会乱序。

------

3. **seqlock vs seqcount（LDD 没有）**

🔹 **LDD：只有 seqlock**

```
seqlock_t
```

🔹 **现在：拆成两层**

```
seqcount_t   // 只有序号，不含锁
seqlock_t    // seqcount + spinlock
```

**现代推荐：**

- **写者已被其他锁保护 → 用 `seqcount_t`**
- **需要写者互斥 → 用 `seqlock_t`**

```
seqcount_t seq;
spinlock_t lock;
write_seqcount_begin(&seq);
...
write_seqcount_end(&seq);
```

📌 **这是 LDD 完全没有的演进**

------

4. 中断 / softirq 场景（LDD 没细讲）

现代内核区分得非常清楚：

| 场景       | 推荐            |
| ---------- | --------------- |
| 进程上下文 | seqlock         |
| 中断上下文 | seqlock_irqsave |
| 写者已上锁 | seqcount        |

```
write_seqlock_irqsave(&lock, flags);
...
write_sequnlock_irqrestore(&lock, flags);
```



### 5.7.5 读取-拷贝-更新



## 5.8 快速参考



~~~c

~~~

> 1

------------------------

