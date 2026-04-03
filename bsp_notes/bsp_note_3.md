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





