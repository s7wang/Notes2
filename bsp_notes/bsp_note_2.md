# BSP NOTE 2

# Kernel + rootfs 启动流程

## Kernel启动流程

无论 x86 还是 ARM，从 **kernel 被加载执行之后**，整体流程是完全一致的：

```
kernel入口
 → start_kernel()
 → 初始化内核子系统（内存/调度/中断/驱动）
 → 挂载 rootfs（initramfs 或 真实 root）
 → 启动第一个用户进程（/init 或 /sbin/init）
 → init(systemd / busybox / procd)
 → 启动用户空间服务（网络 / ssh / dbus 等）
```

一句话总结： **kernel 的终极目标 = 挂载 rootfs + exec 第一个用户进程**。

## 阶段1：进入内核入口

**通用入口**

```
start_kernel()
```

在此之前（架构相关）：

- x86：解压 bzImage → 进入 64 位模式
- ARM：MMU 开启 → 建页表 → 跳转 C 入口

👉 从 `start_kernel()` 开始，**x86 和 ARM 基本统一**

## 阶段2：内核核心初始化

**主调用链（极重要）**

```
start_kernel()
 ├─ setup_arch()        // 架构初始化
 ├─ mm_init()           // 内存管理
 ├─ sched_init()        // 调度器
 ├─ init_IRQ()          // 中断
 ├─ time_init()
 ├─ console_init()
 ├─ rest_init()
```

~~~(空)
start_kernel()
  → rest_init()
    → kernel_init()
      → do_basic_setup()
        → do_initcalls()   ← 就在这里执行
~~~

------

**关键点解释**

**1️⃣ setup_arch()**

- x86：解析 ACPI / e820
- ARM：解析 DTB（设备树）

👉 **这是 x86 vs ARM 最后一次明显差异点**

------

**2️⃣ initcall 机制（驱动初始化核心）**

所有驱动通过：

```
device_initcall()
fs_initcall()
subsys_initcall()
```

被统一执行：

```
do_initcalls()
```

👉 这一步完成：

- 块设备驱动（MMC / SATA / NAND）
- 文件系统（ext4 / squashfs / ubifs）
- 网络栈

⚠️ **rootfs 能不能挂载，取决于这里驱动是否 ready**

## 阶段3：创建第一个内核线程

```
rest_init()
 ├─ kernel_thread(kernel_init)
 ├─ kernel_thread(kthreadd)
```

------

## 阶段4：kernel_init（关键转折点）

```
kernel_init()
 ├─ kernel_init_freeable()
 │   ├─ do_basic_setup()
 │   │   └─ do_initcalls()   ← 所有驱动初始化完成
 │   └─ prepare_namespace()  ← 挂载 rootfs
 └─ run_init_process()
```

------

## 阶段5：rootfs 挂载过程（核心重点）

### 1️⃣ prepare_namespace()

```
prepare_namespace()
 ├─ parse_rootflags()
 ├─ mount_root()
```

------

### 2️⃣ rootfs 类型判断

内核根据启动参数决定：

```
root=xxx
init=xxx
rdinit=xxx
```

------

### 3️⃣ 三种典型路径

------

#### ✔ 路径1：initramfs（最常见）

```
内核内置 initramfs
 → unpack_to_rootfs()
 → 挂载 rootfs（ramfs/tmpfs）
 → 执行 /init
```

源码：

```
populate_rootfs()
```

------

#### ✔ 路径2：块设备 rootfs

```
root=/dev/mmcblk0p2
```

流程：

```
mount_block_root()
 ├─ open_dev()
 ├─ mount_fs(ext4/squashfs)
```

------

#### ✔ 路径3：NFS rootfs

```
root=/dev/nfs
```

流程：

```
mount_nfs_root()
```

### 4️⃣ OpenWrt 特殊（你工作相关）

```
squashfs（只读）
 + overlay（jffs2/ubifs）
```

流程：

```
mount squashfs → /rom
mount overlay → /overlay
overlayfs 合并 → /
```

## 阶段6：启动第一个用户进程

### run_init_process()

```
run_init_process("/sbin/init")
```

尝试顺序：

```
/init
/sbin/init
/etc/init
/bin/init
/bin/sh
```

失败：

```
Kernel panic: No init found
```

## 阶段7：init 进程之后（用户空间启动）

这里开始进入 **用户态世界**

------

### 1️⃣ init 类型

不同系统：

| 系统    | init    |
| ------- | ------- |
| Ubuntu  | systemd |
| OpenWrt | procd   |
| BusyBox | init    |

------

### 2️⃣ systemd 启动流程（x86常见）

```
systemd
 → 解析 unit 文件
 → 启动 target（multi-user.target）
 → 并行启动服务
```

关键服务：

- udev（设备管理）、network、dbus、ssh

------

### 3️⃣ OpenWrt（procd）

```
procd
 → 读取 /etc/inittab
 → 执行 rcS
 → 启动服务（/etc/init.d）
```

------

### 4️⃣ BusyBox init

```
/etc/inittab
 → rcS
 → shell/login
```

## 阶段8：用户空间服务启动链

以 systemd 为例：

```
init
 → basic.target
 → sysinit.target
 → multi-user.target
 → network.target
 → sshd.service
```