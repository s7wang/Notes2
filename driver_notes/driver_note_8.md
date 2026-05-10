# Linux 设备驱动

# 第八章 分配内存

至此, 我们已经使用 `kmalloc` 和 `kfree` 来分配和释放内存。`Linux` 内核提供了更丰富的一套内存分配原语。但是. 在本章, 我们查看在设备驱动中使用内存的其他方法和如何优化你的系统的内存资源。我们不涉及不同的体系实际上如何管理内存。模块不牵扯在分段,分页等问题中, 因为内核提供一个统一的内存管理驱动接口。另外, 我们不会在本章描述内存管理的内部细节, 但是推迟在 15 章。

## 8.1 kmalloc 的真实故事

`kmalloc` 分配引擎是一个有力的工具并且容易学习因为它对 `malloc` 的相似性。这个函数快(除非它阻塞)并且不清零它获得的内存; 分配的区仍然持有它原来的内容。分配的区也是在物理内存中连续. 在下面几节, 我们详细讨论 `kmalloc`, 因此你能比较它和我们后来要讨论的内存分配技术。

### 8.1.1 flags 参数

记住 kmalloc 原型是:

~~~c
#include <linux/slab.h>
void *kmalloc(size_t size, int flags);
~~~

给 `kmalloc` 的第一个参数是要分配的块的大小。 第 2 个参数, 分配标志, 非常有趣, 因为它以几个方式控制 `kmalloc` 的行为。

最一般使用的标志, `GFP_KERNEL`, 意思是这个分配((内部最终通过调用`__get_free_pages` 来进行, 它是 `GFP_` 前缀的来源) 代表运行在内核空间的进程而进行的。换句话说, 这意味着调用函数是代表一个进程在执行一个系统调用。使用 `GFP_KENRL`意味着 `kmalloc` 能够使当前进程在少内存的情况下睡眠来等待一页。一个使用`GFP_KERNEL` 来分配内存的函数必须, 因此, 是可重入的并且不能在原子上下文中运行。当当前进程睡眠, 内核采取正确的动作来定位一些空闲内存, 或者通过刷新缓存到磁盘或者交换出去一个用户进程的内存。

`GFP_KERNEL` 不一直是使用的正确分配标志; 有时 `kmalloc` 从一个进程的上下文的外部调用。例如, 这类的调用可能发生在中断处理, `tasklet`, 和内核定时器中。在这个情况下,当前进程不应当被置为睡眠, 并且驱动应当使用一个`GFP_ATOMIC` 标志来代替. 内核正常地试图保持一些空闲页以便来满足原子的分配。当使用 `GFP_ATOMIC` 时, `kmalloc` 能够使用甚至最后一个空闲页. 如果这最后一个空闲页不存在, 但是, 分配失败。

其他用来代替或者增添 `GFP_KERNEL` 和 `GFP_ATOMIC` 的标志, 尽管它们 2 个涵盖大部分设备驱动的需要. 所有的标志定义在 `<linux/gfp.h>`, 并且每个标志用一个双下划线做前缀, 例如 `__GFP_DMA`。另外, 有符号代表常常使用的标志组合; 这些缺乏前缀并且有时被称为分配优先级。后者包括:

~~~c
GFP_ATOMIC
~~~

> 用来从中断处理和进程上下文之外的其他代码中分配内存. 从不睡眠.

------------------------

~~~c
GFP_KERNEL
~~~

> 内核内存的正常分配. 可能睡眠.

------------------------

~~~c
GFP_USER
~~~

> 用来为用户空间页来分配内存; 它可能睡眠.

------------------------

~~~c
GFP_HIGHUSER
~~~

> 如同 GFP_USER, 但是从高端内存分配, 如果有. 高端内存在下一个子节描述.

------------------------

~~~c
GFP_NOIO
GFP_NOFS
~~~

> 这个标志功能如同 GFP_KERNEL, 但是它们增加限制到内核能做的来满足请求. 一个 GFP_NOFS 分配不允许进行任何文件系统调用, 而 GFP_NOIO 根本不允许任何I/O 初始化. 它们主要地用在文件系统和虚拟内存代码, 那里允许一个分配睡眠,但是递归的文件系统调用会是一个坏注意.

------------------------

上面列出的这些分配标志可以是下列标志的相或来作为参数, 这些标志改变这些分配如何进行:

~~~c
__GFP_DMA
~~~

> 这个标志要求分配在能够 DMA 的内存区. 确切的含义是平台依赖的并且在下面章节来解释.

------------------------

~~~c
__GFP_HIGHMEM
~~~

> 这个标志指示分配的内存可以位于高端内存.

------------------------

~~~c
__GFP_COLD
~~~

> 正常地, 内存分配器尽力返回"缓冲热"的页 -- 可能在处理器缓冲中找到的页. 相反, 这个标志请求一个"冷"页, 它在一段时间没被使用. 它对分配页作 DMA 读是有用的, 此时在处理器缓冲中出现是无用的. 一个完整的对如何分配 DMA 缓存的讨论看"直接内存存取"一节在第 1 章.

------------------------

~~~c
__GFP_NOWARN
~~~

> 这个很少用到的标志,阻止内核来发出警告(使用 printk ), 当一个分配无法满足.

------------------------

~~~c
__GFP_HIGH
~~~

> 这个标志标识了一个高优先级请求, 它被允许来消耗甚至被内核保留给紧急状况的最后的内存页.

------------------------

~~~c
__GFP_REPEAT
__GFP_NOFAIL
__GFP_NORETRY
~~~

> 这些标志修改分配器如何动作, 当它有困难满足一个分配. `__GFP_REPEAT` 意思是"更尽力些尝试" 通过重复尝试 -- 但是分配可能仍然失败. `__GFP_NOFAIL` 标志告诉分配器不要失败; 它尽最大努力来满足要求. 使用 `__GFP_NOFAIL` 是强烈不推荐的; 可能从不会有有效的理由在一个设备驱动中使用它. 最后, `__GFP_NORETRY` 告知分配器立即放弃如果得不到请求的内存.

------------------------

在现代linux内核中，kmalloc与上述使用差异不是很大，几点比较需要注意的地方：

#### 1. API形式：基本没变，但 flag 语义更丰富

LDD3 时代（大约 Linux 2.6）：

```c
void *p = kmalloc(size, GFP_KERNEL);
```

现代 Linux（5.x/6.x）：

```c
void *p = kmalloc(size, GFP_KERNEL);
```

接口几乎一样。

------

都还是：

```c
void *kmalloc(size_t size, gfp_t flags);
```

参数：

- `size`：申请字节数
- `flags`：分配策略（最重要）

------

#### 2. 底层 allocator 完全变了（最大区别）

LDD3 时代主要是：

```
kmalloc
  → slab allocator
```

现代 Linux：

```
kmalloc
  → SLUB allocator（默认）
```

------

LDD3时代：SLAB，经典 slab：

```
Buddy Allocator
   ↓
Page
   ↓
slab cache
   ↓
object
```

特点：

- 每种对象一个 cache
- 管理 metadata 较多
- per-cpu magazine
- cache coloring
- 调试复杂

例如：

```
kmalloc-32
kmalloc-64
kmalloc-128
```

------

现代：SLUB（默认），现代内核基本：

```
Buddy Allocator
   ↓
SLUB
   ↓
kmalloc cache
```

特点：

更简单，去掉很多 slab 元数据。

------

per-cpu freelist，CPU 本地快速分配：

```
CPU0 freelist
CPU1 freelist
```

kmalloc 通常无需锁。

------

更快，典型路径：

```
kmalloc()
  → __kmalloc()
      → slab_alloc_node()
           → this_cpu_ptr()
```

很多时候只是：

```
取 freelist head
```

O(1)。

------

更强调试能力，可启用：

```
slub_debug=P
```

支持：

- redzone
- poison
- use-after-free检测
- double free检测

------

#### 3. kmalloc cache 组织变化

LDD3 中常看到：

```
kmalloc-32
kmalloc-64
kmalloc-128
```

现代还是类似，但可能更多粒度：

```
kmalloc-8
kmalloc-16
kmalloc-32
kmalloc-64
kmalloc-96
kmalloc-128
kmalloc-192
...
```

因为 SLUB 做了优化。查看：

```
cat /proc/slabinfo | grep kmalloc
```

你在树莓派上可以直接看。

------

#### 4. GFP flag变化很大

LDD3主要讲：

```
GFP_KERNEL
GFP_ATOMIC
GFP_DMA
```

现代内核增加很多：

------

##### GFP_KERNEL（仍是最常用）

可睡眠：

```
buf = kmalloc(1024, GFP_KERNEL);
```

用于：

- probe
- ioctl
- sysfs
- 普通驱动路径

------

##### GFP_ATOMIC（仍重要）

中断上下文：

```
irq_handler()
{
    kmalloc(64, GFP_ATOMIC);
}
```

不能睡眠。

------

##### GFP_NOWAIT

现代常用：

```
kmalloc(size, GFP_NOWAIT);
```

区别：

```
GFP_ATOMIC = 可用 emergency reserve
GFP_NOWAIT = 不用 reserve
```

更轻量。

------

##### __GFP_ZERO

现代常见：

```
p = kmalloc(size, GFP_KERNEL | __GFP_ZERO);
```

等价：

```
kzalloc(size, GFP_KERNEL);
```

------

##### NUMA相关 flags

现代服务器常见：

```
kmalloc_node(size, GFP_KERNEL, node);
```

LDD3里较少强调。

------

#### 5. 推荐函数变化：更多用 kzalloc / kcalloc

LDD3 常写：

```
p = kmalloc(sizeof(*p), GFP_KERNEL);
memset(p, 0, sizeof(*p));
```

现代推荐：

```
p = kzalloc(sizeof(*p), GFP_KERNEL);
```

------

数组，旧：

```
p = kmalloc(n * size, GFP_KERNEL);
```

有整数溢出风险。现代：

```
p = kcalloc(n, size, GFP_KERNEL);
```

内核会检查：

```
n * size overflow
```

更安全。

------

#### 6. 大块内存分配行为更明确

kmalloc 依赖物理连续页。例如：

```
kmalloc(1MB, GFP_KERNEL);
```

可能失败。因为需要：

```
连续物理页
```

现代更强调：

------

##### 大块用 vmalloc

```
p = vmalloc(1 << 20);
```

特点：

```
虚拟连续
物理可不连续
```

------

##### DMA用 dma_alloc_coherent

不要再：

```
kmalloc(...);
virt_to_phys(...);
```

LDD3 有些老例子这样写。现代推荐：

```
dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
```

------

#### 7. 安全强化（现代新增）

------

##### kfree_sensitive()

释放敏感数据：

```
kfree_sensitive(key);
```

自动清零。

------

##### CONFIG_HARDENED_USERCOPY

用户拷贝检查：

```
copy_to_user
copy_from_user
```

会验证 kmalloc buffer 边界。

------

##### KASAN

内核地址消毒器：

```
use-after-free
out-of-bounds
```

能检测：

```
p = kmalloc(16);
p[20] = 1;
```

LDD3时代没有。

------

#### 8. 调试方式不同

LDD3 常用：

```
slabinfo
/proc/slabinfo
```

现代还会用：

------

##### kmemleak

```
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

查泄漏。

------

##### tracepoints

```
trace-cmd record -e kmem
```

看：

```
kmalloc
kfree
```

------

##### ftrace

追踪调用路径。

### 8.1.2 内存区

`__GFP_DMA` 和 `__GFP_HIGHMEM` 都有一个平台相关的角色, 尽管对所有平台它们的使用都有效.Linux 内核知道最少 3 个内存区: DMA-能够 内存, 普通内存, 和高端内存. 尽管通常地分配都发生于普通区, 设置这些刚刚提及的位的任一个请求从不同的区来分配内存. 这个想法是, 每个必须知道特殊内存范围(不是认为所有的 RAM 等同)的计算机平台将落入这个抽象中.

DMA-能够 的内存是位于一个优先的地址范围, 外设可以在这里进行 DMA 存取. 在大部分的健全的平台, 所有的内存都在这个区. 在 x86, DMA 区用在 RAM 的前 16 MB, 这里传统的 ISA 设备可以进行 DMA; PCI 设备没有这个限制.

高端内存是一个机制用来允许在 32-位 平台存取(相对地)大量内存. 如果没有首先设置一个特殊的映射这个内存无法直接从内核存取并且通常更难使用. 如果你的驱动使用大量内存, 但是, 如果它能够使用高端内存它将在大系统中工作的更好. 高端内存如何工作以及如何使用它的详情见第 1 章的"高端和低端内存"一节.

无论何时分配一个新页来满足一个内存分配请求, 内核都建立一个能够在搜索中使用的内存区的列表. 如果 `__GFP_DMA` 指定了, 只有 DMA 区被搜索: 如果在低端没有内存可用,分配失败. 如果没有特别的标志存取, 普通和 DMA 内存都被搜索; 如果 `__GFP_HIGHMEM`设置了, 所有的 3 个区都用来搜索一个空闲的页. (注意, 但是, kmalloc 不能分配高端内存.)

情况在非统一内存存取(NUMA)系统上更加复杂. 作为一个通用的规则, 分配器试图定位进行分配的处理器的本地的内存, 尽管有几个方法来改变这个行为.内存区后面的机制在 `mm/page_alloc.c` 中实现, 而内存区的初始化在平台特定的文件中,常常在 arch 目录树的 `mm/init.c`. 我们将在第 15 章再次讨论这些主题.

#### 内存区相关分析

**kmalloc 分配的内存来自内核哪些“内存区（memory zones）”**，还是 **kmalloc 自己内部有哪些 cache/slab 区**？

---

#### 1.kmalloc 最终来自哪些物理内存区（Zone）

`kmalloc()` **不是直接“造内存”**，它最终还是从 **Buddy Allocator（伙伴系统）** 拿页。路径：

```
kmalloc()
  ↓
SLUB (对象分配器)
  ↓
Buddy allocator (按页分配)
  ↓
Zone（DMA / NORMAL / MOVABLE）
```

即：

> **kmalloc对象 → 来自slab page → slab page来自buddy → buddy从zone里拿页**

##### ZONE_DMA

老设备 DMA 用。

```
低地址物理内存
```

x86 常见：前16M

```
< 16MB
```

ARM：看平台定义。

用法

```c
buf = kmalloc(size, GFP_DMA);
```

意思：

```
请从 ZONE_DMA 分配
```

**为什么需要？** - 老设备 DMA 控制器只能访问低地址：

```
device DMA address width = 24 bit
```

只能看到：

```
0x000000 ~ 0xFFFFFF
```

##### ZONE_DMA32

现代 32bit DMA。

```
< 4GB
```

很多 PCIe 设备用。例如：

```
kmalloc(size, GFP_DMA32);
```

------

##### ZONE_NORMAL（最重要）

kmalloc 默认主要从这里来。

```
p = kmalloc(128, GFP_KERNEL);
```

通常就是：

```
ZONE_NORMAL
```

它是：内核永久映射的普通 RAM

内核可以直接访问：

```
virt = phys + PAGE_OFFSET
```

64位系统上：

```
几乎所有RAM都可能是 NORMAL
```

------

32位系统：NORMAL 通常有限。

----

##### ZONE_HIGHMEM（旧时代）

LDD3里常见。

```
高端内存
```

32位机器 RAM > 896MB 时出现。

------

特点：

```
内核不能永久映射
```

必须：

```
kmap(page);
```

------

**kmalloc 不能直接从 highmem 分配。**因为：

```
kmalloc返回的是直接可访问虚拟地址
```

highmem 不能保证。现代：

```
64位基本没有 highmem 概念
```

---

##### ZONE_MOVABLE

用于：

```
页迁移
内存热插拔
大页整理
```

------

一般：

```
kmalloc不会用它
```

因为：

```
slab对象不可随便迁移
```

**查看 zone**

```
cat /proc/zoneinfo
```

你会看到：

```
Node 0, zone DMA
Node 0, zone DMA32
Node 0, zone Normal
Node 0, zone Movable
```

#### 2.kmalloc 内部的 cache 区（slab cache）

这是另一层“内存区”。

##### kmalloc-8

```
8字节对象
```

------

##### kmalloc-16

```
16字节对象
```

------

##### kmalloc-32

```
32字节对象
```

------

直到：

```
kmalloc-8192
kmalloc-16384
...
```

------

比如：

```
p = kmalloc(20, GFP_KERNEL);
```

会进入：

```
kmalloc-32 cache
```

------

```
p = kmalloc(100, GFP_KERNEL);
```

进入：

```
kmalloc-128 cache
```

------

查看：

```
cat /proc/slabinfo | grep kmalloc
```

可能看到：

```
kmalloc-8
kmalloc-16
kmalloc-32
kmalloc-64
...
```

------

**cache 从哪来？** 例如：

```
kmalloc-64
```

需要更多对象：

```
SLUB向buddy申请一页
```

比如：

```
PAGE_SIZE = 4096
```

切成：

```
4096 / 64 = 64 objects
```

放入 freelist。

------

##### per-cpu cache

现代 SLUB：每个 CPU 有本地缓存：

```
CPU0 kmalloc-64 freelist
CPU1 kmalloc-64 freelist
```

分配时：

```
pop freelist
```

无锁。

#### 3.虚拟地址属于哪个内核区域？

kmalloc 返回的是：

```
Kernel Virtual Address
```

通常位于：

```
direct mapping area
```

即：

```
PAGE_OFFSET + phys
```

------

不是：

```
vmalloc area
```

------

区别：

kmalloc

```
虚拟连续
物理连续
```

------

vmalloc

```
虚拟连续
物理不连续
```

------

可以看：

```
virt_to_phys(ptr);
```

kmalloc 可用。vmalloc通常不行。

-------

##### DMA buffer

不要：

```
kmalloc()+virt_to_phys()
```

应该：

```
dma_alloc_coherent()
```

#### 一句话记忆

```
kmalloc对象属于 kmalloc-N slab cache
slab cache 的 page 来自 buddy allocator
buddy 从 Zone(DMA/NORMAL) 拿页
返回 direct-mapped kernel virtual address
```

```
CPU (64-bit)
   ↓
能访问 huge RAM
   ↓
设备 DMA 能力不同
   ├─ 老 ISA设备 → 24-bit → 16MB → ZONE_DMA
   ├─ 普通 PCI设备 → 32-bit → 4GB → ZONE_DMA32
   └─ 新 PCIe设备 → 64-bit → 任意地址
```

> **`ZONE_DMA` 的“前 16MB”是为了兼容“只能访问 24-bit 物理地址”的老 DMA 设备，不是因为 Linux 或 CPU 只能用 16MB。**

在你做 BSP/驱动时：

- 普通内存：`kzalloc()`
- DMA 缓冲：`dma_alloc_coherent()`
- **不要优先想着 `GFP_DMA`**（除非你明确知道设备限制）



### 8.1.3 size 参数

内核管理系统的物理内存, 这些物理内存只是以页大小的块来使用. 结果是, kmalloc 看来非常不同于一个典型的用户空间 malloc 实现. 一个简单的, 面向堆的分配技术可能很快有麻烦; 它可能在解决页边界时有困难. 因而, 内核使用一个特殊的面向页的分配技术来最好地利用系统 RAM.

Linux 处理内存分配通过创建一套固定大小的内存对象池. 分配请求被这样来处理, 进入一个持有足够大的对象的池子并且将整个内存块递交给请求者. 内存管理方案是非常复杂,并且细节通常不是全部设备驱动编写者都感兴趣的.

然而, 驱动开发者应当记住的一件事情是, 内核只能分配某些预定义的, 固定大小的字节数组. 如果你请求一个任意数量内存, 你可能得到稍微多于你请求的, 至多是 2 倍数量.同样, 程序员应当记住 kmalloc 能够处理的最小分配是 32 或者 64 字节, 依赖系统的体系所使用的页大小.

kmalloc 能够分配的内存块的大小有一个上限. 这个限制随着体系和内核配置选项而变化.如果你的代码是要完全可移植, 它不能指望可以分配任何大于 128 KB. 如果你需要多于几个 KB, 但是, 有个比 kmalloc 更好的方法来获得内存, 我们在本章后面描述.

#### 现代Linux内核修正

1. “只能分配固定大小对象”——本质仍然成立；
2. 但“最多浪费 2 倍”已经不那么对了，现代 SLUB 加入了更多中间档位，现代仍有向上取整，但浪费通常远小于“最多 2 倍”；
3. “最小分配是 32 或 64 字节”——已经不对了，现代一般支持8/16字节甚至4字节；
4. 小对象甚至可能走 per-CPU fastpath；
5. 超大对象不走普通 slab - 如果请求很大：`kmalloc(200000);` 可能不会进入：`kmalloc-256k` 而是直 接：`alloc_pages()` 走伙伴系统。阈值通常：`KMALLOC_MAX_CACHE_SIZE`



## 8.2 后备缓存









~~~c

~~~

> 1

------------------------

