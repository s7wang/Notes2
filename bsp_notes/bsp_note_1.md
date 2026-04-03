# BSP NOTE 1

## boot加载过程 (无BIOS的ARM平台)

### U-Boot 为例(无BIOS的ARM平台)

**启动流程总览**

~~~(空)
上电
 ↓
SoC内部 BootROM（固化在芯片）
 ↓
加载 First Stage Bootloader（FSBL / SPL）
 ↓
初始化 SRAM / DDR
 ↓
加载 Second Stage Bootloader（U-Boot）
 ↓
加载 OS（Linux / Windows）
 ↓
用户空间（init / systemd）
~~~

**第一阶段：SoC+BootROM（SROM）硬件起点**

SoC内部包含：

* CPU（ARM / x86）

* SRAM（片上内存）

* BootROM（固化代码）

* 外设控制器（SPI / eMMC / USB 等）

BootROM（SROM）特点：

> 1. 固化在芯片中不可修改
> 2. 上电自动执行
> 3. 整个系统的真正起点

主要用于：硬件最小初始化（时钟、引脚）、判断启动介质（eMMC、SD、SPI Flash、USB）。

BootROM很小，所以

> 1. 不能初始化DDR（太复杂）
> 2. 只能用SRAM

将**第一阶段 Bootloader**从外部存储中copy到SRAM中并转跳执行。

**第二阶段：SRAM + 第一阶段 Bootloader（SPL/FSBL）**

SRAM的作用：

> 1. SRAM是片上小内存（几十KB ~ 几百KB），SROM唯一能使用的内存；
> 2. 加载SPL（Second Program Loader），也叫FSBL（First Stage Bootloader）或BL1 / BL2（ARM Trusted Firmware 体系），核心任务：初始化DDR
>    * DDR时序配置、内存控制器初始化、电源配置
>
> ~~~(空)
> BootROM
>   ↓
> 把 SPL 从 Flash 拷到 SRAM
>   ↓
> 在 SRAM 运行 SPL
>   ↓
> 初始化 DDR
> ~~~

**第三阶段：完整Bootloader（U-Boot）**

在ARM体系主流是使用U-Boot，U-Boot的作用：

> 1. 初始化外设（网卡 / 串口 / USB）
> 2. 提供命令行（类似 shell）
> 3. 加载内核（从 flash / tftp / sd）

U-Boot 内部结构（重要）分两阶段：

~~~(空)
SPL（小）
↓
U-Boot proper（大）
~~~



## 含有ATF的boot流程

### ATF编译产物

编译 ATF 后，一般会得到：

```
build/<platform>/release/
├── bl1.bin
├── bl2.bin
├── bl31.bin
├── bl32.bin（可选，TEE）
```

#### **BL1（最早阶段）**

~~~(空)
bl1.bin
~~~

这个是BootROM之后的第一段，特点是：**有些SoC内置**，**有些需要烧录到外部存储（SPI/eMMC）**。

作用：**建立最基本的运行环境**，**加载BL2**。

#### **BL2（最关键）**

~~~(空)
bl2.bin
~~~

运行在SRAM上（类似SPL），作用（核心）：**初始化DDR（重点！！！）**，**加载BL31/BL33**。

可以简单理解：

~~~(空)
BL2 ≈ U-Boot SPL
~~~

#### **BL31（EL3 runtime）**

~~~(空)
bl31.bin
~~~

**DDR 初始化之后，一直驻留**，作用：**运行再EL3（最高特权级别）**，**提供SMC（安全调用）**。

比如：PSCI（CPU上下电），Secure Monitor，Linux也会调用。

#### BL32（可选）

~~~(空)
bl32.bin
~~~

作用：TEE（可信执行环境）。

比如：OP-TEE



### U-Boot 编译产物（重点）

编译 U-Boot 后：

```
├── u-boot.bin
├── u-boot.elf
├── u-boot.img
├── spl/u-boot-spl.bin
```

#### SPL（第一阶段 Bootloader）

~~~(空)
spl/u-boot-spl.bin
~~~

对应阶段：**和BL2类似（SRAM）**。

作用：**初始化DDR**，**加载u-boot.bin**。

#### u-boot.bin（主程序）

~~~(空)
u-boot.bin
~~~

对应阶段：**BL33（Normal World）**。

作用：**提供shell**，**加载kernel**，**网络/存储支持**。

#### u-boot.img

~~~(空)
u-boot.img
~~~

本质：**u-boot.bin + header（mkimage）**

用于：NAND / SD 卡启动



### ATF + U-Boot 组合关系（最重要）

#### 标准 ARM64 启动（推荐架构）

~~~(空)
BootROM
 ↓
BL1（ATF）
 ↓
BL2（ATF） ← DDR 初始化
 ↓
BL31（ATF） ← 常驻 EL3
 ↓
BL33（U-Boot） ← u-boot.bin
 ↓
Linux
~~~

文件对应关系：

| 阶段    | 文件       | 来源   |
| ------- | ---------- | ------ |
| BootROM | 固化       | SoC    |
| BL1     | bl1.bin    | ATF    |
| BL2     | bl2.bin    | ATF    |
| BL31    | bl31.bin   | ATF    |
| BL33    | u-boot.bin | U-Boot |

#### 真实烧写布局

以eMMC/SD卡为例：

~~~(空)
[BootROM读取位置]
 ├── BL1 / SPL
 ├── BL2
 ├── BL31
 ├── U-Boot
 ├── Kernel
 └── RootFS
~~~

**不同 SoC偏移地址完全不同**（厂商定义），**需要看BSP / TRM（技术手册）**。

#### Q & A

##### ❗1：BL2 和 SPL 是谁？

👉 本质一样：

| 名称 | 来源   |
| ---- | ------ |
| BL2  | ATF    |
| SPL  | U-Boot |

👉 **二选一，不同时用**

------

##### ❗2：BL31 为什么必须？

因为：

👉 Linux 在 ARM64 上需要：

- PSCI（电源管理）、CPU 热插拔

这些都在 BL31

------

##### ❗3：U-Boot 是第几阶段？

👉 在 ATF 架构中：

```
U-Boot = BL33
```

### MTK的boot构建

MTK的boot构建：bl2.img + fip.bin = ATF 主导启动，**U-Boot 被当成 BL33 打包进 FIP**。

也就是说：

~~~(空)
BootROM
 ↓
BL2（bl2.img）
 ↓
FIP（里面包含：BL31 + U-Boot + 可选 BL32）
~~~

####  什么是 FIP（关键）

FIP = Firmware Image Package（固件包）

来自 ARM Trusted Firmware，本质：

```
一个“容器文件”，里面打包多个镜像
```

**FIP的结构**：

~~~(空)
fip.bin
 ├── BL31（EL3 runtime）
 ├── BL33（U-Boot）
 ├── BL32（可选，TEE）
 ├── 配置数据（tb_fw_config 等）
~~~

**FIP的作用**：

> BL2 需要加载多个镜像（BL31 / U-Boot / TEE），但 Flash 读取不方便，所以：
>
> 把所有镜像打成一个包 → BL2 一次加载 → 按 ID 分发。

#### FIP的构建过程

1. 编译U-Boot：生成 u-boot.bin（BL33）；

2. 编译ATF：生成bl2.bin，bl31.bin；
3. 把U-Boot塞进ATF：在ATF编译时 `make PLAT=xxx BL33=path/to/u-boot.bin` ，ATF会把 u-boot.bin 作为 BL33 打包进 FIP；
4. 最终生成FIP：fip.bin

**最终输出**：

~~~(空)
bl2.img   ← 第一阶段（SRAM）是bl2.bin打包结果，增加信息头和校验信息。
fip.bin   ← 第二阶段（DDR）
~~~

### 启动时真实执行过程（重点）

#### Step 1：BootROM 阶段

```
BootROM
 ↓
从 Flash 读取 bl2.img
 ↓
拷贝到 SRAM
 ↓
执行 BL2
```

------

#### Step 2：BL2 阶段（关键）

BL2 做三件事：

```
1. 初始化 DDR
2. 读取 fip.bin
3. 解析 FIP
```

------

##### FIP 解析（核心机制）

FIP 内部有：

```
UUID → 镜像
```

BL2 会按类型加载：

```
BL31 → EL3
BL33 → U-Boot
BL32 → TEE（可选）
```

------

#### Step 3：跳转执行

```
BL2
 ↓
跳转 BL31（EL3）
 ↓
BL31 初始化
 ↓
启动 BL33（U-Boot）
```

------

#### Step 4：U-Boot 阶段

```
U-Boot
 ↓
加载 kernel
 ↓
启动 Linux
```

#### MTK的设计思路

##### 强制分层（安全架构）

```
BL2（初始化）
BL31（安全控制）
U-Boot（普通世界）
```

👉 避免 U-Boot 直接控制底层

##### 统一加载入口（FIP）

BL2 不需要知道：

- U-Boot 在哪、TEE 在哪

👉 只需要：解析 FIP

##### 方便升级

你可以单独替换：

- U-Boot（BL33）、TEE（BL32）

不用动 BL2

##### 存储分布

可以把 MTK 启动看成：

```
Flash:
 ├── bl2.img   ← BootROM直接加载
 └── fip.bin   ← BL2解析

RAM:
 ├── BL31（EL3）
 ├── U-Boot（BL33）
 └── BL32（TEE，可选）
```

> bl2.img = 第一阶段（SRAM执行，负责加载）
>
> fip.bin = 第二阶段容器（包含 BL31 + U-Boot + TEE）



## 高通boot流程（ipq5018）

高通方案的boot是：**高通 TrustZone + 多子系统固件体系**。

在 **IPQ5018（QCA平台）**里：

👉 **Boot 不是单一产物，而是多个固件拼装**
 最终烧录的不是 `.img`，而是：

👉 **.mbn / .bin / FIT / NAND分区镜像**

核心 boot 组成：

```
PBL (ROM)
  ↓
SBL (boot_images生成)
  ↓
TZ / ATF / QSEE (btfw_proc)
  ↓
U-Boot (apss_proc)
  ↓
Linux Kernel
```

### boot_images（最核心 Boot 组件）

👉 相当于：**高通版 BL2/SBL**

里面包含：

- SBL1 / SBL2 / SBL3
- DDR初始化
- PMIC初始化
- 安全验证（Secure Boot）
- 加载 TZ / U-Boot

编译输出：

```
sbl1.mbn
sbl1.elf
prog_emmc_firehose_*.mbn（烧录工具）
```

关键点：**这是整个系统第一个可控boot代码**

类似：

- MTK：BL2
- ARM标准：TF-A BL2

### btfw_proc（TrustZone / ATF / 安全固件）

👉 相当于：**BL31 + Secure OS**

包含：

- QSEE（Qualcomm Secure Execution Environment）
- TrustZone OS
- SCM / SMC接口
- crypto / secure storage

编译输出：

```
tz.mbn
hyp.mbn（有的芯片）
devcfg.mbn
keymaster.mbn
cmnlib.mbn
```

👉 关键点：运行在 EL3 / Secure World

提供：

- secure boot校验、smc调用、crypto

👉 对标：

- ARM：TF-A BL31 + OP-TEE

### apss_proc（应用处理器系统）

👉 相当于：**U-Boot + Linux**

包含：

- U-Boot
- Kernel
- rootfs（有些SDK一起）

编译输出：

```
u-boot.bin
appsboot.mbn   ← 高通打包后的U-Boot
Image / zImage
*.dtb
```

👉 关键点：

- appsboot.mbn = U-Boot + 签名头
- 这是你最终控制 Linux 启动的地方

### BTFW.MAPLE.xxx（预编译固件包）

👉 本质是：
 👉 **高通官方提供的闭源固件集合**

里面包含：

```
tz.mbn
hyp.mbn
rpm.mbn
wlanfw.bin
```

👉 作用：

- 不需要你编译
- 直接参与打包

### 最终 Boot 是怎么拼出来的（重点）

| 阶段   | 文件来源         |
| ------ | ---------------- |
| PBL    | SoC内部ROM       |
| SBL    | boot_images      |
| TZ     | btfw_proc / BTFW |
| U-Boot | apss_proc        |
| Kernel | apss_proc        |

### 最终 Flash 布局（典型）

NAND / eMMC：

```
0x0        → SBL1 (sbl1.mbn)
0x80000    → MIBIB
0x100000   → QSEE (tz.mbn)
0x200000   → U-Boot (appsboot.mbn)
0x400000   → Kernel
```

------

### 启动流程（完整）

```
[ROM PBL]
   ↓
加载 sbl1.mbn
   ↓
[SBL1]
   ↓ 初始化DDR
   ↓ 校验签名
   ↓ 加载 tz.mbn
   ↓
[TrustZone]
   ↓ 初始化安全环境
   ↓
加载 appsboot.mbn
   ↓
[U-Boot]
   ↓
加载 Kernel + dtb
   ↓
[Linux]
```



## boot加载过程 （BIOS/UEFI + GRUB）

**“硬件组成 → 固件 → Bootloader → 内核 → 用户态”**

### 硬件组成

一台 x86 机器启动，本质涉及这些硬件：

#### CPU（处理器）

关键点：

> * 上电后从固定地址执行 （Reset Vector）。
> * 初始是：16-bit实时模式、无分页、无保护；
> * 拥有寄存器：CS:IP（指令入口）、CR0/CR3（后续控制分页）

#### ROM / BIOS Flash

> * 存放：BIOS固件（或UEFI）
> * CPU上电后直接映射执行

#### RAM（内存）

> * 上电时：内容时随机的；
> * BIOS会初始化：DDR控制器，内存映射（e820）

#### 存储设备

> * HDD/SSD/NVMe/USB
> * 存放：MBR/GPT，GRUB，Linux内核

#### 外设 & 总线

> PCI / PCIe，SATA / NVMe 控制器，网卡（PXE）

BIOS会枚举这些设备。

### 启动过程

### 1.CPU上电 → BIOS执行（硬件→固件的交界）

1. **Reset Vector**

CPU 上电后：

```
CS:IP → 0xFFFF:0x0000
物理地址 → 0xFFFFFFF0
```

👉 这个地址映射到 BIOS ROM

------

2. **BIOS 接管控制**

BIOS 做三件关键事：

（1）硬件初始化

- DDR training（最关键）
- PCI 初始化
- 时钟 / 电源管理

👉 这是硬件 + 固件协同的第一步

------

（2）构建运行环境

BIOS 提供：

- **实模式**
- **BIOS中断**
  - int 0x13（磁盘）
  - int 0x10（显示）

👉 后续 GRUB 依赖这些接口

------

（3）构建内存信息（e820）

BIOS 会提供：

```
可用内存 / 保留内存 / MMIO
```

👉 Linux 后面会用这个表做内存管理

### 2.启动设备选择（硬件+固件协同）

BIOS 按顺序尝试：

```
硬盘 → USB → PXE
```

每一步：

- 读取设备扇区
- 检查是否有启动标志（0x55AA）

### 3.MBR → Stage1（硬件 I/O + 固件接口）

1. **BIOS 读取 MBR**

```
调用 int 0x13
```

读取：

```
LBA 0 → 0x7C00
```

------

2. **MBR 的作用**

MBR 的核心作用：

👉 **加载更复杂的 Bootloader（GRUB）**

### 4.GRUB（Bootloader 层）

GRUB 是软件 + 硬件的桥梁

------

#### 1. Stage1（极小代码）

特点：

- 只有 446 字节
- 只能：
  - 调 BIOS
  - 加载下一阶段

------

#### 2. Stage1.5 / core.img

- 支持文件系统
- 能读取 `/boot`

👉 从这里开始进入“文件系统世界”

------

#### 3. Stage2（完整 GRUB）

功能：

**（1）文件系统解析**

支持：

- ext4、xfs、btrfs

👉 直接读 `/boot`

------

**（2）加载内核**

```
linux /boot/vmlinuz
```

- 解压 bzImage
- 放入内存

------

**（3）加载 initramfs**

```
initrd /boot/initramfs.img
```

------

**（4）构建 boot 参数**

传给 Linux：

- cmdline
- 内存信息
- initrd 地址

------

**（5）切换 CPU 模式**

GRUB 会：

```
实模式 → 保护模式 → 长模式(64-bit)
```

------

（6）跳转内核

```
jmp linux_kernel_entry
```

### 5.Linux 内核启动（软件开始完全接管硬件）

#### 1. 内核入口

```
arch/x86/boot/header.S
```

------

#### 2. setup阶段（非常关键）

完成：

- 解压内核
- 处理参数
- 切换模式

------

#### 3. start_kernel()

进入 C 世界：

```
start_kernel()
```

做：

（1）内存初始化

- 解析 e820
- 建立页表

------

（2）中断初始化

- IDT
- APIC

------

（3）调度器初始化

- task_struct
- CFS

------

（4）驱动初始化

- PCI 驱动
- 网卡驱动

------

4. 挂载 rootfs

如果有 initramfs：

```
/init
```

否则：

```
/sbin/init
```

------

### 6.用户空间（最终状态）

进入：

- systemd / init

然后：

```
登录 / shell / 服务
```





















