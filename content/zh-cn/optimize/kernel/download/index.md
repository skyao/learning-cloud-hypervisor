---
title: "下载"
linkTitle: "下载"
weight: 20
date: 2025-11-15
description: >
  cloud hypervisor官方优化内核下载
---

## 信息

cloud hypervisor官方优化内核的代码仓库为：

https://github.com/cloud-hypervisor/linux/

优化内核的二进制下载地址为：

https://github.com/cloud-hypervisor/linux/releases

## 选择

最新版本为 ch-release-v6.16.9-20251112， 这是 linux 6.16 内核。考虑到我目前用的 debian 13 的物理机的内核是 6.12 内核，因此我保持两者的一致，继续使用 linux 6.12 内核作为 guest os 的内核。

> 等以后 debian13 升级到 6.16 内核再考虑升级 guest os 的内核版本。

支持 x86-64, arm64 和 riscv， 以 x86-64 为例，有两个版本：

- bzImage-x86_64： 7.99MB，标准 Linux 发行版用的压缩内核。虽然 Cloud Hypervisor 也支持，但它在启动时需要额外的解压过程，对内存优化场景不如 vmlinux 直接。

- vmlinux-x86_64： 45.8MB， 原始的内核 ELF 文件（未压缩）。Cloud Hypervisor 可以直接解析它

推荐选择 vmlinux 版本： Cloud Hypervisor 属于“轻量级 VMM”，它支持 Direct Kernel Boot（直接内核引导）。与传统虚拟机需要 bzImage（带解压引导头的压缩镜像）不同，Cloud Hypervisor 更倾向于直接加载 ELF 格式的 vmlinux。这样启动速度最快，且内存占用最少。

## 下载

最新的 6.12 内核版本为：

https://github.com/cloud-hypervisor/linux/releases/tag/ch-release-v6.12.8-20250613

下载到特定目录，方便以后使用：

```bash
mkdir /home/sky/work/soft/cloudhypervisor/kernel/ch-6.12

cd /home/sky/work/soft/cloudhypervisor/kernel/ch-6.12

wget https://github.com/cloud-hypervisor/linux/releases/download/ch-release-v6.12.8-20250613/vmlinux-x86_64

cp vmlinux-x86_64 vmlinux

mv vmlinux-x86_64 vmlinux-x86_64-download
```