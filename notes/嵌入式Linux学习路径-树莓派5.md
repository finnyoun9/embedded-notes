# 嵌入式 Linux 学习路径 — 树莓派 5 部署

> **目标**: 在树莓派 5 上运行自编译内核 + 自写字符设备驱动，面试展示"我会 Linux 驱动"
> **当前进度**: 树莓派 5 已有，环境待搭建

---

## 为什么学嵌入式 Linux？

中级以上嵌入式岗位几乎都要求 Linux 能力。**"会 STM32 + FreeRTOS" 是入门标配，"会 Linux 驱动"才是面试拉开差距的关键。**

深圳嵌入式工程师招聘 JD 高频出现：
- "熟悉 Linux 内核、驱动开发"
- "了解设备树(Device Tree)、字符设备驱动框架"
- "有 ARM Linux 移植/BSP 开发经验优先"

---

## 三层架构（从浅到深）

```
Layer 3: Linux 应用层
  ├── Shell 脚本、文件 I/O
  ├── Socket 编程 (TCP/UDP)
  └── 多线程 (pthread)

Layer 2: Linux 系统层
  ├── 交叉编译 (aarch64-linux-gnu-gcc)
  ├── Buildroot / Yocto 构建系统
  ├── 设备树 (DTS)
  └── 内核模块编译

Layer 1: Linux 驱动层  ← 面试区别点
  ├── 字符设备驱动 (cdev)
  ├── Platform Driver 框架
  ├── GPIO / I2C / SPI 子系统
  └── 中断处理 (ISR / tasklet / workqueue)
```

---

## 树莓派 5 环境搭建

### Step 1: 烧录系统

```bash
# 下载 Ubuntu 24.04 ARM64 镜像
# https://ubuntu.com/download/raspberry-pi

# 用 Raspberry Pi Imager 烧录到 SD 卡
# 或命令行:
# sudo dd if=ubuntu-24.04-preinstalled-server-arm64+raspi.img of=/dev/diskX bs=4M
```

### Step 2: 首次启动 + SSH

```bash
# 连接显示器/键盘，或配置 headless:
# 在 SD 卡 boot 分区创建 user-data 文件

# SSH 连接（Pi 连入 wifi 后）
ssh ubuntu@<pi-ip>
# 默认密码: ubuntu，首次登录强制修改

# 安装基础工具
sudo apt update && sudo apt install -y build-essential git vim
```

### Step 3: 树莓派科学上网（已有文档）

参考 `notes/RaspberryPi科学上网配置.md` — 已配置 Mihomo 代理。

---

## 学习路线（4 周）

### Week 1: Linux 系统基础

```
├── 文件系统 (/proc, /sys, /dev 的作用)
├── 进程管理 (ps, top, kill, nice)
├── Shell 脚本 (条件判断、循环、函数)
├── 文件 I/O (open/read/write/close — 系统调用级别)
└── Makefile 编写
```

### Week 2: 内核模块入门

```c
// hello.c — 你的第一个内核模块
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello, Finn's first kernel module!\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye from kernel module!\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Finn");
```

```makefile
# Makefile
obj-m += hello.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

all:
    make -C $(KDIR) M=$(PWD) modules
clean:
    make -C $(KDIR) M=$(PWD) clean
```

```bash
# 编译 + 加载
make
sudo insmod hello.ko    # 插入模块
dmesg | tail -5         # 看打印
sudo rmmod hello        # 移除模块
```

### Week 3: 字符设备驱动

```c
// 目标: 写一个 /dev/finn-led 字符设备
// 打开 → 控制树莓派 GPIO LED
// 
// 核心 API:
//   register_chrdev()    注册设备号
//   file_operations      定义 open/read/write/ioctl
//   copy_from_user()     从用户空间拿数据
//   gpio_set_value()     控制 GPIO
```

### Week 4: 设备树 + Platform Driver

```
├── 理解设备树语法 (.dts → .dtb)
├── 在树莓派上修改设备树添加自定义外设
├── 写 Platform Driver 匹配设备树节点
└── probe/remove 生命周期
```

---

## 交叉编译方案（在 Mac 上编译给 Pi）

### 方案 A: 直接装交叉编译器（推荐）

```bash
# macOS Apple Silicon → aarch64-linux
brew tap messense/macos-cross-toolchains
brew install aarch64-unknown-linux-gnu

# 验证
aarch64-unknown-linux-gnu-gcc --version
```

### 方案 B: Docker（平台无关）

```bash
# 使用 ARM64 交叉编译容器
docker run --rm -v $(pwd):/work -w /work \
  arm64v8/ubuntu:24.04 \
  bash -c "apt update && apt install -y build-essential && make"
```

### 方案 C: 直接在 Pi 上编译（最简单）

```bash
# SSH 到 Pi，直接编译（不需要交叉工具链）
ssh ubuntu@pi "cd /path/to/src && make"
```

---

## 面试会问的 Linux 内核问题

| 问题 | 答案要点 |
|------|----------|
| **用户空间和内核空间区别？** | 内核态有特权指令、可直接访问硬件；用户态通过系统调用进入内核 |
| **字符设备和块设备区别？** | 字符设备按字节流访问(串口)，块设备按块访问(硬盘)，有缓存层 |
| **设备树的作用？** | 描述硬件拓扑，使内核代码与板级硬件解耦；dts→dtb→内核解析 |
| **中断上半部和下半部？** | 上半部(ISR)快进快出，下半部(tasklet/workqueue/threaded IRQ)处理耗时操作 |
| **copy_from_user 为什么必要？** | 安全：验证用户空间指针合法性，防止内核访问非法地址崩溃 |

---

## 下一步行动

- [ ] 树莓派 5 烧录 Ubuntu 24.04
- [ ] SSH 连接 + 基础环境配置
- [ ] 完成 hello.ko（第一个内核模块）
- [ ] 完成 /dev/finn-led 字符设备驱动
- [ ] 了解 Buildroot 构建最小 Linux 系统
