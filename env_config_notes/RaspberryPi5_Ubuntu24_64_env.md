# ubuntu24交叉编译环境配置 & 树莓派5内核升级

## ubuntu24交叉编译环境配置

### 环境准备

1.安装工具

~~~shell
sudo apt update
sudo apt install crossbuild-essential-arm64
sudo apt install bc bison flex libssl-dev bc libncurses-dev libelf-dev
sudo apt install libncurses5-dev make 
~~~

2.下载源码

~~~shell
git clone --depth=1 --branch=rpi-6.12.y  https://github.com/raspberrypi/linux.git
~~~

3.编译内核 - 初始化配置

~~~shell
cd linux
KERNEL=kernel_2712
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2712_defconfig
~~~

4. 编译内核 - 编译

~~~shell
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image.gz modules dtbs
~~~



## 树莓派5内核升级

1. 替换RaspberryPi5的内核

~~~shell
#首先，运行 lsblk。然后，连接启动媒体。再次运行 lsblk；新设备代表启动媒体。你应该会看到类似下面的输出：
# sdb                         8:16   1 29.7G  0 disk 
# ├─sdb1                      8:17   1  512M  0 part 
# └─sdb2                      8:18   1  5.5G  0 part 
# sr0                        11:0    1 1024M  0 rom 
# 首先，将这些分区挂载为 mnt/boot 和 mnt/root，调整分区代号以匹配启动媒体的位置：
mkdir mnt
mkdir mnt/boot
mkdir mnt/root
sudo mount /dev/sdb1 mnt/boot
sudo mount /dev/sdb2 mnt/root
~~~

2. 内核模块安装到启动媒体上：

~~~shell
sudo env PATH=$PATH make -j12 ARCH=arm64 \
CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/root modules_install
~~~

3. 安装 64 位内核：

- 运行以下命令创建当前内核的备份镜像，安装新的内核镜像、覆盖层、README，并卸载分区：

~~~shell
sudo cp mnt/boot/$KERNEL.img mnt/boot/$KERNEL-backup.img
sudo cp arch/arm64/boot/Image mnt/boot/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/boot/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/boot/overlays/
sudo cp arch/arm64/boot/dts/overlays/README mnt/boot/overlays/
sudo umount mnt/boot
sudo umount mnt/root
~~~

4. 启动后验证： 编译时间以及内核版本以及更新

~~~shell
uname -a
Linux Wangs7Raspberry5 6.12.77-v8-16k+ #1 SMP PREEMPT Wed Mar 25 03:10:18 UTC 2026 aarch64 GNU/Linux
# 镜像默认版本为6.12.47-1+rpt1
~~~



注意：从树莓派上获取kernel-heads（没有移植内核前，移植后回失效，版本信息不匹配且链接目录为空）

~~~shell
wangs7_pi@Wangs7Raspberry5:~ $ sudo apt install linux-headers-rpi-v8
wangs7_pi@Wangs7Raspberry5:~ $ 
wangs7_pi@Wangs7Raspberry5:~ $ ls -l /lib/modules/$(uname -r)/
total 2972
lrwxrwxrwx  1 root root     24 Mar 25 22:57 build -> /home/wangs7server/linux
drwxr-xr-x 10 root root   4096 Mar 25 22:57 kernel
-rw-r--r--  1 root root 703592 Mar 25 22:57 modules.alias
-rw-r--r--  1 root root 725118 Mar 25 22:57 modules.alias.bin
-rw-r--r--  1 root root  16566 Mar 25 22:57 modules.builtin
-rw-r--r--  1 root root   8219 Mar 25 22:57 modules.builtin.alias.bin
-rw-r--r--  1 root root  18491 Mar 25 22:57 modules.builtin.bin
-rw-r--r--  1 root root 110073 Mar 25 22:57 modules.builtin.modinfo
-rw-r--r--  1 root root 252741 Mar 25 22:57 modules.dep
-rw-r--r--  1 root root 337860 Mar 25 22:57 modules.dep.bin
-rw-r--r--  1 root root    384 Mar 25 22:57 modules.devname
-rw-r--r--  1 root root  75635 Mar 25 22:57 modules.order
-rw-r--r--  1 root root   1166 Mar 25 22:57 modules.softdep
-rw-r--r--  1 root root 341455 Mar 25 22:57 modules.symbols
-rw-r--r--  1 root root 413925 Mar 25 22:57 modules.symbols.bin
wangs7_pi@Wangs7Raspberry5:~ $ 
~~~

这里可以把 

~~~shell
lrwxrwxrwx  1 root root     24 Mar 25 22:57 build
~~~

这个链接指向的目录打包导出，后续操作和前面相同。



## 内核模块编译

* 内核编译环境构建 - 在编译过内核的工程中

~~~shell
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules_prepare
# 清除执行 sudo  make mrproper
~~~

* 编译主机上构建工程

~~~(空)
hello_drv/
├── hello.c
└── Makefile
~~~

* hello.c

~~~c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

/* 模块信息（必须） */
MODULE_LICENSE("GPL");
MODULE_AUTHOR("wangs7");
MODULE_DESCRIPTION("Simple Hello Kernel Module");
MODULE_VERSION("1.0");

/* 模块加载时执行 */
static int __init hello_init(void)
{
    pr_info("Hello Kernel Module Loaded!\n");
    return 0;
}

/* 模块卸载时执行 */
static void __exit hello_exit(void)
{
    pr_info("Hello Kernel Module Unloaded!\n");
}

module_init(hello_init);
module_exit(hello_exit);
~~~

* Makefile

~~~makefile
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
obj-m += hello.o

KDIR := /home/wangs7server/linux   # 你的内核源码路径

PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
~~~

* 执行编译

~~~shell
wangs7server@ubuntuserver:~/hello_drv$ make
make -C /home/wangs7server/linux    M=/home/wangs7server/hello_drv modules
make[1]: Entering directory '/home/wangs7server/linux'
  CC [M]  /home/wangs7server/hello_drv/hello.o
  MODPOST /home/wangs7server/hello_drv/Module.symvers
  CC [M]  /home/wangs7server/hello_drv/hello.mod.o
  CC [M]  /home/wangs7server/hello_drv/.module-common.o
  LD [M]  /home/wangs7server/hello_drv/hello.ko
make[1]: Leaving directory '/home/wangs7server/linux'
wangs7server@ubuntuserver:~/hello_drv$ ls
hello.c  hello.ko  hello.mod  hello.mod.c  hello.mod.o  hello.o  Makefile  modules.order  Module.symvers
wangs7server@ubuntuserver:~/hello_drv$ 
~~~

* 传输ko文件

~~~shell
scp hello.ko wangs7_pi@192.168.124.99:~/
~~~

目标主机验证

~~~shell
wangs7_pi@Wangs7Raspberry5:~ $ chmod +x hello.ko 
wangs7_pi@Wangs7Raspberry5:~ $ sudo insmod hello.ko 
wangs7_pi@Wangs7Raspberry5:~ $ dmesg | tail -n 1
[ 4072.017848] Hello Kernel Module Loaded!
wangs7_pi@Wangs7Raspberry5:~ $ sudo rmmod hello.ko 
wangs7_pi@Wangs7Raspberry5:~ $ dmesg | tail -n 2
[ 4072.017848] Hello Kernel Module Loaded!
[ 4096.584007] Hello Kernel Module Unloaded!
~~~

到此编译环境构建完成。





