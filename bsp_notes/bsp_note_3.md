# BSP NOTE 3

# 总线

## 总线分类

ARM 和 x86 本质差异在于：

- ARM → SoC集成，**片上总线为主（AMBA体系）**
- x86 → 平台化，**外设/主板总线为主（PCIe体系）**

## ARM体系总线（SoC内部为核心）

ARM基本都围绕 AMBA 架构展开。

### 1️⃣ AMBA总线体系（核心）

#### （1）AXI（高性能主干总线）

- 用于：CPU / GPU / DDR 控制器
- 特点：
  - 支持乱序、多通道（读写分离）、高吞吐

👉 场景：

```
CPU → AXI → DDR
CPU → AXI → GPU/NPU
```

------

#### （2）AHB（中速总线）

- 用于：DMA、USB、以太网等
- 特点：
  - 单通道、比AXI简单

------

#### （3）APB（低速外设总线）

- 用于：GPIO / UART / I2C / SPI
- 特点：
  - 低功耗、无突发传输

👉 典型结构：

```
AXI → AHB → APB → UART/I2C/GPIO
```

------

#### （4）ACE / CHI（一致性总线）

- 用于：多核一致性
- 支持 Cache Coherency

👉 比如：

```
CPU cluster ↔ CPU cluster
CPU ↔ GPU
```

------

### 2️⃣ ARM常见外设总线协议

这些通常挂在 APB / AHB 上：

#### （1）I2C

- 低速控制类、两线（SCL/SDA）

------

#### （2）SPI

- 高于 I2C、全双工

------

#### （3）UART

- 串口通信、调试核心

------

#### （4）CAN

- 工业/车载

------

#### （5）SDIO / eMMC

- 存储设备

------

### 3️⃣ ARM高速接口

#### （1）PCIe（ARM也支持）

- 用于 WiFi / NVMe

#### （2）USB

- USB2 / USB3

#### （3）MIPI（移动设备专用）

- CSI（摄像头）、DSI（显示）

## x86体系总线（平台化为核心）

x86是“CPU + 芯片组”的体系，核心是 PCI Express。

------

### 1️⃣ CPU直连总线

#### （1）内存总线（DDR）

- CPU → 内存控制器 → DRAM

------

#### （2）PCIe（核心总线）

- GPU / NVMe / 网卡

👉 现代结构：

```
CPU → PCIe → GPU
CPU → PCIe → SSD
```

------

### 2️⃣ 芯片组总线（PCH）

#### （1）DMI（Intel）

- CPU ↔ 芯片组通信

👉 本质就是 PCIe

------

#### （2）南桥/外设总线

* USB、SATA、Audio（HDA）、LAN



# I2C Bus

## 一、I2C 基本原理

### 1. 物理层

I2C 只需要两根线：

- **SCL（时钟线）**
- **SDA（数据线）**

特点：

- **开漏（Open-Drain）结构**
- 需要 **上拉电阻**
- 多主多从（但一般只有一个主设备）

------

### 2. 通信规则

I2C 典型时序：

```
START → 地址 + R/W → ACK → 数据 → ACK → ... → STOP
```

关键点：

- **主设备控制时钟 SCL（Master）**
- 从设备响应
- 每字节 8 bit + 1 bit ACK
- 数据只在 **SCL=低电平时变化**
- **SCL=高电平时采样**

**START**

```
SCL: 1
SDA: 1 → 0
```

**STOP**

```
SCL: 1
SDA: 0 → 1
```

------

**ACK 机制**

每发送 8bit：

- 第 9 位由从设备拉低（ACK=0）
- 不拉低 = NACK

------

### 3. 设备地址

- 7-bit 地址（常见）
- 10-bit 地址（少见）

例如：

```
0x50 → EEPROM
0x68 → RTC
```

### 4. 数据时序（关键）

```
       ┌───┐   ┌───┐
SCL:   │   │___│   │___

SDA:   ----data stable----
        ↑   ↑
      setup hold
```

### 5 通信协议

1. 起止信号：时钟线为1时，正常不允许发生数据线变动，但是时钟线为1时，数据线 1->0 表示开始信号，数据线0->1表示停止。
2. 写操作：写操作首先要指定一个设备和指令，在开始信号结束后，发送一个字节的数据 7位设备器地址+1位写标记，然后收到ack后发送寄存器地址，在收到ack后写入一个字节数据（这里可以选择连续写），在收到最后一个ack后，发出结束信号。
3. 读操作：读操作首先先要指定一个设备的一个寄存器，所以前序和写操作一样要写一个寄存器地址，**指定一个设备和指令，在开始信号结束后，发送一个字节的数据 7位设备器地址+1位写标记，然后收到ack后发送寄存器地址**，收到ack后，然后在不结束的前提下重新发出开始信号，并且发送这个设备的**一个字节的数据 7位设备器地址+1位读标记**，然后以ACK为请求读（可以连续读），以NACK表示停止读，最后发送停止信号。

**读 - 总线完整时序**

```
START
→ 0xA0 (addr<<1 | write)
→ ACK
→ 0x10 (寄存器地址)
→ ACK
→ 0xAB (数据)
→ ACK
→ STOP
```

**写 - 总线完整时序**

```
START
→ 0xA0 (write)
→ ACK
→ 0x10 (寄存器地址)
→ ACK
→ RESTART
→ 0xA1 (read)
→ ACK
→ DATA
→ NACK
→ STOP
```



## 二、硬件层：控制器（Controller）

I2C在SoC中通常由一个**硬件控制器（I2C Controller）**实现：

### 控制器主要功能：

- 产生 SCL
- 发送/接收 SDA
- 处理 ACK
- 产生中断 / DMA

在Linux中对应：

```
I2C Controller Driver（适配芯片）
```

## 三、Boot 阶段（ARM / x86）

### 1. Boot阶段的I2C作用

在 Boot（Bootloader / BIOS / UEFI）中，I2C主要用于：

- 读取 **EEPROM / SPD（内存条信息）**
- 初始化 **PMIC（电源芯片）**
- 读取 **板级配置**
- 控制 **reset / GPIO expander**
- 读取 **sensor / 校准参数**

------

### 2. ARM：U-Boot 中的 I2C

在 ARM 平台（如 RK / MTK / Qualcomm）：

#### U-Boot 提供 I2C 命令

```
i2c dev 0
i2c probe
i2c read 0x50 0x00 0x100 buffer
i2c write 0x50 0x00 0x01
```

#### 代码层：

```
drivers/i2c/
```

关键接口：

```
i2c_set_bus_num()
i2c_probe()
i2c_read()
i2c_write()
```

------

### 3. UEFI / BIOS（x86）

在 x86：

#### ① BIOS 阶段（传统）

- 很少直接用 I2C
- 主要用 SMBus（I2C 子集）

------

#### ② UEFI 阶段

UEFI中：

- I2C 通过 **I2C Protocol**
- 或 **SMBus Protocol**

接口：

```
EFI_I2C_MASTER_PROTOCOL
```

典型用途：

- 读取 SPD（内存参数）
- PMIC 控制
- 面板 / 触控控制器

------

### 4. SMBus vs I2C（x86重点）

SMBus = I2C 的子集 + 一些扩展，区别：

| 特性          | I2C  | SMBus |
| ------------- | ---- | ----- |
| 速率          | 可高 | 较低  |
| 超时          | 无   | 有    |
| 协议严格性    | 宽松 | 严格  |
| x86 BIOS/UEFI | 少见 | 常见  |

在 x86 内存条 SPD 读取：

```
SMBus → SPD EEPROM（I2C设备）
```

## 四、内核阶段

I2C 在 Linux 内核里的工作，本质上是 **“总线驱动（adapter）+ 设备驱动（client）+ 核心框架（i2c core）” 三层协作模型**。我给你按你做 BSP / 驱动开发的视角，讲清楚“启动 → 匹配 → 通信 → 驱动实现”。

### 1.I2C 在内核中的整体架构

Linux I2C 子系统分 3 层：

```
用户空间
   ↓
设备驱动 (i2c_driver)   ← 你主要写这个
   ↓
I2C Core (i2c-core.c)  ← 总线匹配 / 调度
   ↓
控制器驱动 (i2c_adapter) ← SoC厂商提供
   ↓
I2C 硬件控制器
   ↓
I2C 设备（EEPROM / 传感器等）
```

### 2.控制器驱动先初始化（adapter）

在 kernel 启动过程中（通常在 `subsys_initcall` 或 `arch_initcall`）：

👉 SoC I2C 控制器驱动会：

```c
i2c_add_adapter(&adap);
```

核心作用：

- 注册一个 I2C 总线（比如 i2c-0 / i2c-1）
- 提供底层操作函数（master_xfer）

👉 adapter 关键结构：

```c
struct i2c_adapter {
    const struct i2c_algorithm *algo;
    int nr;
};
```

👉 algo（最关键）：

```c
struct i2c_algorithm {
    int (*master_xfer)(...);   // 真正收发函数
};
```

✔ 这一步完成：
 👉 “硬件控制器具备通信能力”

### 3. 设备信息注册（client）

设备怎么来？

Device Tree（最常用）

```
i2c1 {
    status = "okay";

    eeprom@50 {
        compatible = "atmel,24c02";
        reg = <0x50>;
    };
};
```

👉 内核解析 DT 后：

```
i2c_new_client_device()
```

生成：

```
struct i2c_client {
    addr = 0x50;
}
```

✔ 这一步完成：
 👉 “总线上有哪些设备”

------

### 4. 驱动注册 + 匹配

你写的驱动：

```c
static struct i2c_driver my_driver = {
    .driver = {
        .name = "my_i2c_dev",
        .of_match_table = xxx,
    },
    .probe = my_probe,
};

module_i2c_driver(my_driver);
```

👉 内核做的事：

```
i2c_register_driver()
   ↓
bus_add_driver()
   ↓
driver 与 client 匹配
   ↓
调用 probe()
```

✔ 匹配规则：

- DT: compatible
- 或 i2c_device_id

### 5. probe 之后：驱动如何操作 I2C

probe 里你拿到：

```
struct i2c_client *client;
```

然后就可以通信了。

#### 1. 核心通信接口

（1）i2c_transfer（底层）

```
int i2c_transfer(adapter, msgs, num);
```

👉 你可以精确控制：

- start
- restart
- read/write

👉 message：

```
struct i2c_msg {
    addr
    flags  // 读写
    buf
    len
};
```

------

（2）常用封装（推荐）

写寄存器：

```
i2c_smbus_write_byte_data(client, reg, val);
```

读寄存器：

```
i2c_smbus_read_byte_data(client, reg);
```

------

#### 2. 典型“读寄存器”流程（重点）

比如读设备寄存器 0x10：

I2C 总线行为：

```
START
设备地址 + 写
寄存器地址（0x10）
RESTART
设备地址 + 读
读取数据
STOP
```

👉 对应 kernel：

```
u8 reg = 0x10;
u8 val;

struct i2c_msg msgs[2] = {
    {
        .addr = client->addr,
        .flags = 0, // write
        .buf = &reg,
        .len = 1,
    },
    {
        .addr = client->addr,
        .flags = I2C_M_RD,
        .buf = &val,
        .len = 1,
    }
};

i2c_transfer(client->adapter, msgs, 2);
```

✔ 关键点：
 👉 “写寄存器地址 + restart + 读数据”

## 五、I2C的轮询和中断

### 1. 先给你一个本质结论

```
I2C 轮询 vs 中断 = “CPU等硬件” vs “硬件通知CPU”
```

| 模式              | 核心思想           |
| ----------------- | ------------------ |
| 轮询（polling）   | CPU 一直查寄存器   |
| 中断（interrupt） | 硬件完成后通知 CPU |

### 2. I2C 在内核中的位置（关键）

你写：

```
i2c_transfer(...)
```

👇 实际调用：

```
i2c core
   ↓
adapter->algo->master_xfer   ← 控制器驱动实现
   ↓
（这里决定：轮询 or 中断）
```

👉 **轮询 / 中断完全是在“控制器驱动”里实现的**

------

### 3. I2C 轮询模式（Polling）

1. 工作方式

```
CPU:
    写寄存器启动 I2C
    ↓
    while(没完成)
        一直读状态寄存器
    ↓
    读数据
```

------

2. 伪代码

```
write_reg(START);

while (!(read_reg(STATUS) & DONE)) {
    cpu_relax();  // 空转
}

data = read_reg(DATA);
```

✅ 优点

- 实现简单（很多 early BSP 都这么写）
- 调试容易
- 不依赖中断控制器（GIC）

------

❌ 缺点

- **CPU 忙等（浪费性能）**
- 不能并发
- 延迟不稳定

------

4. 适用场景

- bootloader（比如 U-Boot）
- 简单 SoC
- 调试阶段

### 4. I2C 中断模式（Interrupt）

1. 工作方式

```
CPU:
    发起 I2C
    ↓
    睡眠 / 返回
    ↓
I2C 控制器完成
    ↓
触发中断
    ↓
ISR 处理数据
```

------

### 5. 内核结构（典型）

（1）master_xfer

```c
init_completion(&done);

write_reg(START);

wait_for_completion_timeout(&done);
```

------

（2）中断处理函数

```c
irq_handler() {
    status = read_reg(STATUS);

    if (RX_READY)
        read_fifo();

    if (TX_EMPTY)
        write_fifo();

    if (DONE)
        complete(&done);
}
```

✅ 优点

- CPU 不忙等
- 支持并发
- 更高性能
- 更符合 Linux 设计

------

❌ 缺点

- 实现复杂
- 调试难（中断 + 状态机）
- 容易出 race condition

------

适用场景

- Linux 内核正式驱动
- 高频 I2C（如 camera / sensor）
- 多设备系统

------

### 6. 核心区别（你必须掌握）

| 对比       | 轮询 | 中断 |
| ---------- | ---- | ---- |
| CPU 占用   | 高   | 低   |
| 实现复杂度 | 低   | 高   |
| 性能       | 低   | 高   |
| 是否用中断 | ❌    | ✅    |
| Linux 推荐 | ❌    | ✅    |

## 六、I2C中断模型

**接近真实内核风格的 I2C 控制器中断模式代码模板**（简化但结构完整），你可以直接拿去理解或改 BSP。

------

### 1. 整体结构（你先有全局感）

```
i2c_transfer()
   ↓
master_xfer()
   ↓
启动硬件 + 等待 completion
   ↓
IRQ 触发
   ↓
irq_handler() 分阶段处理
   ↓
complete()
   ↓
返回上层
```

------

### 2.核心数据结构

```c
struct my_i2c_dev {
    void __iomem *base;
    int irq;

    struct completion done;

    struct i2c_msg *msgs;
    int msg_num;
    int msg_idx;

    int buf_pos;

    spinlock_t lock;
};
```

------

### 3.master_xfer（入口）

```c
static int my_i2c_master_xfer(struct i2c_adapter *adap,
                             struct i2c_msg *msgs,
                             int num)
{
    struct my_i2c_dev *dev = i2c_get_adapdata(adap);
    unsigned long timeout;

    dev->msgs = msgs;
    dev->msg_num = num;
    dev->msg_idx = 0;
    dev->buf_pos = 0;

    reinit_completion(&dev->done);

    /* 1. 配置设备地址 + 方向 */
    writel(msgs[0].addr, dev->base + ADDR_REG);

    /* 2. 使能中断 */
    writel(IRQ_TX_EMPTY | IRQ_RX_READY | IRQ_DONE,
           dev->base + IRQ_ENABLE);

    /* 3. 启动传输（产生 START） */
    writel(CTRL_START, dev->base + CTRL_REG);

    /* 4. 等待完成（睡眠） */
    timeout = wait_for_completion_timeout(&dev->done,
                                          msecs_to_jiffies(1000));

    if (!timeout) {
        dev_err(adap->dev.parent, "i2c timeout\n");
        return -ETIMEDOUT;
    }

    return num;
}
```

------

### 4.中断处理函数（核心重点🔥）

```
static irqreturn_t my_i2c_irq(int irq, void *data)
{
    struct my_i2c_dev *dev = data;
    u32 status;

    status = readl(dev->base + IRQ_STATUS);

    /* 1. TX FIFO 空（需要发送数据） */
    if (status & IRQ_TX_EMPTY) {

        struct i2c_msg *msg = &dev->msgs[dev->msg_idx];

        if (!(msg->flags & I2C_M_RD)) {
            if (dev->buf_pos < msg->len) {
                writel(msg->buf[dev->buf_pos++],
                       dev->base + TX_FIFO);
            }
        }
    }

    /* 2. RX FIFO 有数据（接收数据） */
    if (status & IRQ_RX_READY) {

        struct i2c_msg *msg = &dev->msgs[dev->msg_idx];

        if (msg->flags & I2C_M_RD) {
            if (dev->buf_pos < msg->len) {
                msg->buf[dev->buf_pos++] =
                    readl(dev->base + RX_FIFO);
            }
        }
    }

    /* 3. 当前 message 完成 */
    if (status & IRQ_MSG_DONE) {

        dev->msg_idx++;
        dev->buf_pos = 0;

        if (dev->msg_idx < dev->msg_num) {
            /* 下一个 message（可能是 repeated start） */
            struct i2c_msg *msg = &dev->msgs[dev->msg_idx];

            writel(msg->addr, dev->base + ADDR_REG);

            if (msg->flags & I2C_M_RD)
                writel(CTRL_READ, dev->base + CTRL_REG);
            else
                writel(CTRL_WRITE, dev->base + CTRL_REG);

        } else {
            /* 所有传输完成 → STOP */
            writel(CTRL_STOP, dev->base + CTRL_REG);
        }
    }

    /* 4. 整个传输完成 */
    if (status & IRQ_DONE) {
        complete(&dev->done);
    }

    /* 清中断 */
    writel(status, dev->base + IRQ_CLEAR);

    return IRQ_HANDLED;
}
```

------

### 5.probe 里注册中断

```
static int my_i2c_probe(struct platform_device *pdev)
{
    struct my_i2c_dev *dev;

    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);

    init_completion(&dev->done);
    spin_lock_init(&dev->lock);

    dev->irq = platform_get_irq(pdev, 0);

    devm_request_irq(&pdev->dev, dev->irq,
                     my_i2c_irq,
                     0,
                     "my-i2c",
                     dev);

    return 0;
}
```

## 七、这段代码你要真正理解的点（面试/开发重点）

------

### 🔥 1. 为什么要 completion？

```
wait_for_completion()
```

👉 让进程睡眠
 👉 中断里 `complete()` 唤醒

✔ 这就是 **中断模式核心机制**

------

### 🔥 2. 为什么要 msg_idx？

因为：

```
i2c_transfer 可以是多段消息
```

典型：

```
write reg → read data
```

👉 就是 2 个 msg

------

### 🔥 3. 为什么要 TX_EMPTY / RX_READY？

👉 FIFO 驱动模型：

| 中断     | 含义       |
| -------- | ---------- |
| TX_EMPTY | 该写数据了 |
| RX_READY | 有数据可读 |
| DONE     | 结束       |

------

### 🔥 4. STOP 什么时候发？

```
最后一个 msg 完成后
```

## 

| 图里                | 代码        |
| ------------------- | ----------- |
| wait_for_completion | master_xfer |
| IRQ: TX_EMPTY       | 写 FIFO     |
| IRQ: RX_READY       | 读 FIFO     |
| IRQ: DONE           | complete    |

**总结一句话**

```
I2C 中断模式本质 = 状态机 + FIFO + completion + IRQ驱动
```
