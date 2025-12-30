---
title: "快速开始"
linkTitle: "快速开始"
weight: 10
date: 2025-10-15
description: >
  cloud hypervisor 快照快速开始
---

## 机器配置

安装好 cloud hypervisor，测试机器为 debian13 物理机器，intel x86-64 cpu。

```bash
$ cloud-hypervisor --version
cloud-hypervisor v50.0.0

$ ch-remote --version
ch-remote v50.0.0

$ uname -a
Linux skydev3 6.12.57+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.57-1 (2025-11-05) x86_64 GNU/Linux
```

## 内核

为了极致的启动速度和测试效率，推荐使用 Cloud Hypervisor 官方提供的测试资源，或者使用 Alpine Linux 这种极简发行版。

```bash
mkdir -p ~/work/test/cloudhypervisor
cd ~/work/test/cloudhypervisor
```

### ch官方内核

linux kernel 采用之前构建的 x86 内核（45MB），方便起见复制到本地：

```bash
cp ~/work/code/cloud-hypervisor/linux-cloud-hypervisor/arch/x86/boot/compressed/vmlinux.bin .
```

这个内核版本是 6.12，非常新。

记录文件地址备用:

```bash
/home/sky/work/test/cloudhypervisor/vmlinux.bin
```

## rootfs

### ubuntu cloud image

兼容性最好，但是有点大：

```bash
# 下载 Ubuntu 24.04 (Noble) 云镜像
mkdir ubuntu-cloud-image
cd ubuntu-cloud-image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Cloud Hypervisor 需要 raw 格式，且需要解压
sudo apt install -y qemu-utils
qemu-img convert -p -f qcow2 -O raw noble-server-cloudimg-amd64.img rootfs.raw

# 调整大小（可选，云镜像默认只有 2GB 左右）
qemu-img resize rootfs.raw 4G
```

操作之后得到的 rootfs 为：

```bash
ls -lh                       
total 2.4G
-rw-rw-r-- 1 sky sky 598M Dec 13 21:20 noble-server-cloudimg-amd64.img
-rw-r--r-- 1 sky sky 4.0G Dec 30 17:07 rootfs.raw
```

记录文件路径备用:

```bash
/home/sky/work/test/cloudhypervisor/ubuntu-cloud-image/rootfs.raw
```

###  Alpine Linux

TODO

## 启动

启动 microvm：

```bash
sudo rm -rf /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock
sudo /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor \
    --api-socket /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock \
    --cpus boot=1 \
    --memory size=4G \
    --kernel /home/sky/work/test/cloudhypervisor/vmlinux.bin \
    --cmdline "root=/dev/vda1 console=hvc0 rw" \
    --disk path=/home/sky/work/test/cloudhypervisor/ubuntu-cloud-image/rootfs.raw
```

在另外一个终端，先 pause：

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock pause
```

再执行快照命令，快照文件输出到 snapshot-4g 文件夹：


```bash
mkdir -p snapshot-4g
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  snapshot file:///home/sky/work/test/cloudhypervisor/snapshot-4g
```

完成后可以看到：

```bash
ls -lh ./snapshot-4g
```

文件列表为：

```bash
total 4.1G
-rw------- 1 root root 1.4K Dec 30 17:23 config.json
-rw------- 1 root root 4.0G Dec 30 17:23 memory-ranges
-rw------- 1 root root  42K Dec 30 17:23 state.json
```

执行关闭命令，关闭这个 microvm

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  resume
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  shutdown
sleep 1
sudo rm -rf /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock
```

## 快照恢复

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor \
    --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock \
    --restore source_url=file:///home/sky/work/test/cloudhypervisor/snapshot-4g
```



