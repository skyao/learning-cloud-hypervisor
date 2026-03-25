---
title: "alpine"
linkTitle: "alpine"
weight: 20
date: 2025-11-15
description: >
  alpine rootfs 优化
---

## 背景

由于我们为 microvm 场景定制的内核是 25.9MB 的全内置 (Monolithic) 内核，因此 Rootfs 制作将遵循一个核心原则：彻底剔除所有内核模块和硬件固件。

Alpine 使用 musl libc 和 BusyBox，是目前启动最快的 Linux。

## 定制脚本

```bash
mkdir -p /home/sky/work/code/cloud-hypervisor/rootfs/alphine
cd /home/sky/work/code/cloud-hypervisor/rootfs/alphine

vi build_agent_rootfs_erofs.sh
```

内容为:

```bash
#!/bin/bash
set -e

# --- 配置参数 ---
ROOTFS_NAME="alpine_rootfs.erofs"
WORKING_DIR="./mnt_alpine"
ALPINE_VERSION="3.20.3"
ALPINE_TAR="alpine-minirootfs-${ALPINE_VERSION}-x86_64.tar.gz"
ALPINE_URL="https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/${ALPINE_TAR}"

echo "🚀 开始自动化构建 EROFS + DAX 专用 Alpine Rootfs..."

# 1. 依赖检查
if ! command -v mkfs.erofs &> /dev/null; then
    echo "❌ 错误: 未安装 erofs-utils。请运行: sudo apt install erofs-utils"
    exit 1
fi

# 2. 下载原始 Rootfs 包
if [ ! -f "$ALPINE_TAR" ]; then
    echo "📥 下载 Alpine 原始包..."
    curl -L -O "$ALPINE_URL"
fi

# 3. 准备工作目录
echo "📂 准备构建目录..."
sudo rm -rf "$WORKING_DIR"
mkdir -p "$WORKING_DIR"
sudo tar -xzkf "$ALPINE_TAR" -C "$WORKING_DIR"

# 4. 进入 Chroot 进行内部定制
echo "🛠️ 开始内部定制..."
sudo cp /etc/resolv.conf "$WORKING_DIR/etc/resolv.conf"

sudo chroot "$WORKING_DIR" /bin/sh <<EOF
# 设置密码
echo "root:root" | chpasswd

# 1. 基础环境补全
mkdir -p /etc/init.d
mkdir -p /run/openrc
touch /run/openrc/softlevel

# 2. 优化终端：仅保留 hvc0
sed -i '/tty[0-9]/d' /etc/inittab
if ! grep -q "hvc0" /etc/inittab; then
    echo "hvc0::respawn:/sbin/getty -L hvc0 115200 vt100" >> /etc/inittab
fi

# 3. 换源并安装 OpenRC 
echo "https://mirrors.aliyun.com/alpine/v3.20/main" > /etc/apk/repositories
echo "https://mirrors.aliyun.com/alpine/v3.20/community" >> /etc/apk/repositories
apk add --no-cache openrc

# 4. 配置启动项
rc-update add devfs boot
rc-update add local boot
EOF

# 5. 注入 OverlayFS 极速启动逻辑 (针对 DAX 和写层优化)
echo "⚡ 注入 OverlayFS 启动逻辑..."
sudo tee "$WORKING_DIR/etc/init.d/rcS" <<EOF
#!/bin/sh
# 基础虚拟文件系统挂载
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev

# 准备写层 (假设 /dev/vda 是通过 --disk 传入的 Ext4 镜像)
mkdir -p /mnt/write_layer
if mount -t ext4 -o noatime /dev/vda /mnt/write_layer; then
    echo "✅ Writable layer (/dev/vda) detected."
    # 准备 OverlayFS 结构
    mkdir -p /mnt/write_layer/upper /mnt/write_layer/work
    # 将 /var 叠加到写层，使得日志和配置可持久化
    mount -t overlay overlay -o lowerdir=/var,upperdir=/mnt/write_layer/upper,workdir=/mnt/write_layer/work /var
else
    echo "⚠️ Warning: No writable disk detected, using tmpfs for /var."
    mount -t tmpfs tmpfs /var
fi

# 其他临时目录使用内存
mount -t tmpfs tmpfs /tmp
mount -t tmpfs tmpfs /run

echo "------------------------------------------------"
echo "  Welcome to Agent MicroVM (EROFS + DAX)       "
echo "  Status: Rootfs is Read-Only | /var is Writable"
echo "------------------------------------------------"

# 执行正常的 init 流程
exec /sbin/init
EOF

sudo chmod +x "$WORKING_DIR/etc/init.d/rcS"

# 6. 极致清理
echo "🧹 执行最后瘦身清理..."
sudo rm -rf "$WORKING_DIR/lib/modules"/*
sudo rm -rf "$WORKING_DIR/lib/firmware"/*
sudo rm -rf "$WORKING_DIR/var/cache/apk"/*
sudo rm -f "$WORKING_DIR/etc/resolv.conf"

# 7. 构建 EROFS 镜像 (核心步骤)
# -d4: 4KB对齐，DAX 必需
# --fsid=rootfs: 方便内核识别
echo "💎 构建 EROFS 镜像..."
rm -f "$ROOTFS_NAME"
sudo mkfs.erofs -d4 "$ROOTFS_NAME" "$WORKING_DIR"

# 8. 完成
sudo rm -rf "$WORKING_DIR"
echo "✅ 构建完成！镜像文件: ${ROOTFS_NAME}"
ls -lh "$ROOTFS_NAME"
```

增加执行权限：

```bash
chmod +x build_agent_rootfs_erofs.sh
```

执行：

```bash
./build_agent_rootfs_erofs.sh
```

执行完成后，rootfs 文件会生成在当前目录下，大小为约 10MB。

```bash
-rw-r--r-- 1 root root 8.9M Feb 28 09:57 alpine_rootfs.erofs
```

## 测试

### 准备工作

准备工作目录和文件：

```bash
mkdir -p /home/sky/work/test/cloudhypervisor/agent
cd /home/sky/work/test/cloudhypervisor/agent
cp /home/sky/work/code/cloud-hypervisor/linux/vmlinux .
cp /home/sky/work/code/cloud-hypervisor/rootfs/alphine/alpine_rootfs.erofs .
```

准备可写层文件：

```bash
# 1. 创建一个 64MB 的原始镜像 (大小可根据日志量调整)
# 使用 seek 可以创建一个空洞文件，逻辑大小 64M，实际初始占用 0
dd if=/dev/zero of=write_layer_template.img bs=1M count=0 seek=64

# 2. 格式化为极致精简的 Ext4
# -O ^has_journal: 关键优化！禁用日志。
#   在只读根镜像+独立写层的架构下，日志会产生双倍 IO 开销，禁用它能大幅提升小文件写入速度。
# -O ^huge_file,^flex_bg: 移除针对大文件系统的冗余特性。
# -m 0: 将保留给 root 用户的 5% 空间设为 0，充分利用 64MB。
mkfs.ext4 -O ^has_journal,^huge_file,^flex_bg -m 0 write_layer_template.img
```

### 启动 microvm

```bash
mkdir microvm-1
cp write_layer_template.img microvm-1/write_layer.img

sudo /usr/local/bin/cloud-hypervisor \
    --kernel ./vmlinux \
    --memory size=1024M \
    --pmem file=./alpine_rootfs.erofs \
    --disk path=./microvm-1/write_layer.img \
    --cpus boot=1 \
    --console tty \
    --cmdline "console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS"
```

### 进程分析

这是启动单个 microvm 的进程信息：

```bash
ps -ef | grep cloud
root     1351503 1349610  0 14:50 pts/0    00:00:00 sudo /usr/local/bin/cloud-hypervisor --kernel ./vmlinux --memory size=1024M --pmem file=./alpine_rootfs.erofs --disk path=./microvm-1/write_layer.img --cpus boot=1 --console tty --cmdline console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS
root     1351505 1351503  0 14:50 pts/1    00:00:00 sudo /usr/local/bin/cloud-hypervisor --kernel ./vmlinux --memory size=1024M --pmem file=./alpine_rootfs.erofs --disk path=./microvm-1/write_layer.img --cpus boot=1 --console tty --cmdline console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS
root     1351506 1351505  1 14:50 pts/1    00:00:00 /usr/local/bin/cloud-hypervisor --kernel ./vmlinux --memory size=1024M --pmem file=./alpine_rootfs.erofs --disk path=./microvm-1/write_layer.img --cpus boot=1 --console tty --cmdline console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS
root     1351509 1351506  0 14:50 pts/1    00:00:00 /usr/local/bin/cloud-hypervisor --kernel ./vmlinux --memory size=1024M --pmem file=./alpine_rootfs.erofs --disk path=./microvm-1/write_layer.img --cpus boot=1 --console tty --cmdline console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS
```

这是一个多层级进程结构。虽然只运行了一个启动命令，但在 Linux 的 ps 输出中出现了 4 个相关的条目。这是由 sudo 的工作机制、Cloud Hypervisor 的内部架构以及 Linux 线程的处理方式共同决定的。

按照 **PID (进程 ID)** 和 **PPID (父进程 ID)** 的链路来拆解，进程链路树状:

| PID     | PPID    | 用户 | 角色           | 状态描述                                                     |
| ------- | ------- | ---- | -------------- | ------------------------------------------------------------ |
| 1351503 | 1349610 | root | Sudo Shell     | `pts/0` 输入命令后产生的第一个 `sudo` 进程。                 |
| 1351505 | 1351503 | root | Sudo Runner    | `sudo` 内部为了安全执行目标程序而分叉（fork）出的运行环境。  |
| 1351506 | 1351505 | root | CH Main (核心) | 这是真正的 MicroVM 主进程。它负责内存管理、设备初始化。      |
| 1351509 | 1351506 | root | CH Worker      | 这是一个子线程（被 `ps` 显示为进程），通常负责异步 I/O 或 API 处理。 |

由于存在多余的 sudo 进程，因此大规模部署时不建议用 sudo。考虑到直接用 root 也不够安全，因此最好的解决方式是给运行 cloud hypervisor 的非 root 帐号 和 cloud-hypervisor 二进制文件赋予需要的权限。

以 sky 帐号为例，具体需要的操作有：

```bash
# 赋予 ch 网络管理、原始 IO 和内存锁定权限
sudo setcap 'cap_net_admin,cap_sys_rawio,cap_ipc_lock+ep' /usr/local/bin/cloud-hypervisor

# 设置设备访问权限
# Cloud Hypervisor 需要访问 /dev/kvm（虚拟化加速）和 /dev/vhost-vsock 等设备
# 将帐号加入 kvm 组
sudo usermod -aG kvm sky
# 为了确保每次开机 /dev/kvm 等设备都能被 sky 访问，创建一条规则
echo 'KERNEL=="kvm", GROUP="kvm", MODE="0660"' | sudo tee /etc/udev/rules.d/99-kvm.rules
echo 'KERNEL=="vhost-vsock", GROUP="kvm", MODE="0660"' | sudo tee -a /etc/udev/rules.d/99-kvm.rules
sudo udevadm control --reload-rules && sudo udevadm trigger

# 确保你的 vmlinux、alpine_rootfs.erofs 以及 microvm-1/ 目录下的所有文件，属主都是 sky
# sudo chown -R sky:sky /home/sky/work/test/cloudhypervisor/agent
```

去掉 sudo，用 sky 帐号启动：

```bash
/usr/local/bin/cloud-hypervisor \
    --kernel ./vmlinux \
    --memory size=1024M \
    --pmem file=./alpine_rootfs.erofs \
    --disk path=./microvm-1/write_layer.img \
    --cpus boot=1 \
    --console tty \
    --cmdline "console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS quiet"
```

ps的信息如下：

```bash
ps -ef | grep cloud

sky      1352498 1351990  0 15:10 pts/0    00:00:00 /usr/local/bin/cloud-hypervisor --kernel ./vmlinux --memory size=1024M --pmem file=./alpine_rootfs.erofs --disk path=./microvm-1/write_layer.img --cpus boot=1 --console tty --cmdline console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS

sky      1352501 1352498  0 15:10 pts/0    00:00:00 /usr/local/bin/cloud-hypervisor --kernel ./vmlinux --memory size=1024M --pmem file=./alpine_rootfs.erofs --disk path=./microvm-1/write_layer.img --cpus boot=1 --console tty --cmdline console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS
```

去掉 sudo 后，现在的 ps 输出变得非常清爽且符合预期。中间那两层中转进程彻底消失了，直接看到的是 Cloud Hypervisor 本身。

| **PID** | **PPID** | **用户** | **角色**                | **状态描述**                                                 |
| ------- | -------- | -------- | ----------------------- | ------------------------------------------------------------ |
| 1352498 | 1351990  | sky      | 主进程 (Parent)         | 这是你手动启动的那个二进制文件，负责资源协调。               |
| 1352501 | 1352498  | sky      | 工作线程 (Child/Thread) | 它是主进程派生出的一个轻量级进程（LWP），通常处理 I/O 或信号。 |

### 内存分析

```bash
sudo pmap -x 1351506
1351506:   /usr/local/bin/cloud-hypervisor --kernel ./vmlinux --memory size=1024M --pmem file=./alpine_rootfs.erofs --disk path=./microvm-1/write_layer.img --cpus boot=1 --console tty --cmdline console=hvc0 root=/dev/pmem0 ro rootfstype=erofs init=/etc/init.d/rcS
Address           Kbytes     RSS   Dirty Mode  Mapping
000055556aece000       4       0       0 -----   [ anon ]
000055556aecf000       8       8       8 rw---   [ anon ]
00007f8d66ff4000       8       0       0 -----   [ anon ]
00007f8d66ff6000    2052       4       4 rw---   [ anon ]
00007f8d671f7000       8       0       0 -----   [ anon ]
00007f8d671f9000    2052      20      20 rw---   [ anon ]
00007f8d673fa000       8       0       0 -----   [ anon ]
00007f8d673fc000    2052       4       4 rw---   [ anon ]
00007f8d675fd000       8       0       0 -----   [ anon ]
00007f8d675ff000    2052    2048    2048 rw---   [ anon ]
00007f8d67800000   10240    2752       0 rw-s- alpine_rootfs.erofs
00007f8d683fd000       8       0       0 -----   [ anon ]
00007f8d683ff000    2052    2048    2048 rw---   [ anon ]
00007f8d68600000 1048576   69632   69632 rw---   [ anon ]
00007f8da87fa000       8       0       0 -----   [ anon ]
00007f8da87fc000    2052       4       4 rw---   [ anon ]
00007f8da89fd000       8       0       0 -----   [ anon ]
00007f8da89ff000    2052    2048    2048 rw---   [ anon ]
00007f8da8c00000     368     368       0 r---- cloud-hypervisor
00007f8da8c5c000    3276    2940       0 r-x-- cloud-hypervisor
00007f8da8f8f000     576     384       0 r---- cloud-hypervisor
00007f8da901f000     600     260     260 rw--- cloud-hypervisor
00007f8da90b5000       8       8       8 rw---   [ anon ]
00007f8da910b000       4       4       4 rw---   [ anon ]
00007f8da910c000       4       0       0 -----   [ anon ]
00007f8da910d000       8       0       0 rw---   [ anon ]
00007f8da910f000       4       4       4 rw---   [ anon ]
00007f8da9110000       4       0       0 -----   [ anon ]
00007f8da9111000       8       0       0 rw---   [ anon ]
00007f8da9113000     124      56      56 rw---   [ anon ]
00007f8da9132000       8       8       8 rw-s-   [ anon ]
00007f8da9134000       8       8       4 rw-s-   [ anon ]
00007f8da9136000       4       0       0 -----   [ anon ]
00007f8da9137000       8       0       0 rw---   [ anon ]
00007f8da9139000      20      16      16 rw---   [ anon ]
00007f8da913e000       4       0       0 -----   [ anon ]
00007f8da913f000       8       0       0 rw---   [ anon ]
00007f8da9141000       4       0       0 -----   [ anon ]
00007f8da9142000       8       0       0 rw---   [ anon ]
00007f8da9144000      60      60      56 rw---   [ anon ]
00007f8da9153000      12       8       8 rw-s-   [ anon ]
00007f8da9156000       4       4       4 rw-s- zero (deleted)
00007f8da9157000      84      80      80 rw---   [ anon ]
00007f8da916c000       4       0       0 -----   [ anon ]
00007f8da916d000       8       0       0 rw---   [ anon ]
00007f8da916f000       4       4       4 rw---   [ anon ]
00007f8da9170000       4       0       0 -----   [ anon ]
00007f8da9171000       8       0       0 rw---   [ anon ]
00007f8da9173000      72      72      72 rw---   [ anon ]
00007f8da9185000       4       0       0 -----   [ anon ]
00007f8da9186000       8       0       0 rw---   [ anon ]
00007f8da9188000       4       4       4 rw---   [ anon ]
00007f8da9189000      16       0       0 r----   [ anon ]
00007f8da918d000       8       4       0 r-x--   [ anon ]
00007ffe2b7bf000     132      60      60 rw---   [ stack ]
---------------- ------- ------- ------- 
total kB         1078736   82920   76464
```

## 参考资料

和 gemini 的交互过程：

https://gemini.google.com/app/604671631189df20