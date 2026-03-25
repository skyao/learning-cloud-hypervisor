---
title: "介绍"
linkTitle: "介绍"
weight: 10
date: 2025-10-15
description: >
  cloud hypervisor 的文件后端内存介绍
---

在 Cloud Hypervisor (CH) 中，使用文件作为内存后端（File-backed memory）主要是为了实现高性能（Hugepages）、内存共享（vhost-user）或持久化内存（PMEM）等功能。

这个配置主要通过 `--memory` 参数下的 file 选项来实现。



## 基本语法

核心参数是 --memory:

```bash
--memory size=<内存大小>,file=<文件路径>[,shared=on|off][,hugepages=on|off]
```

- **size**: 虚拟机内存大小 (例如 1G, 4G).
- **file**: 主机（Host）上用于映射内存的文件的绝对路径。
- **shared**: (可选) 是否将内存映射为共享（MAP_SHARED）。如果需要与 vhost-user 设备通信，必须设为 on。
- **hugepages**: (可选) 显式声明该文件位于 Hugepages 文件系统中。



## 常见使用场景

### 使用 Hugepages (大页内存) 

这是最常见的用途，推荐用于生产环境。

使用大页内存可以减少 TLB miss，显著提高虚拟机性能。

步骤：

1. 确保主机已挂载 hugetlbfs (通常在 /dev/hugepages)。
2. 指定文件路径

命令示例：

```bash
# 准备目录
mkdir -p /home/sky/work/test/cloudhypervisor/file-backed-memory
cd /home/sky/work/test/cloudhypervisor/file-backed-memory

sudo rm -rf /home/sky/work/test/cloudhypervisor/file-backed-memory/cloud-hypervisor.sock
sudo /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor \
    --api-socket /home/sky/work/test/cloudhypervisor/file-backed-memory/cloud-hypervisor.sock \
    --cpus boot=2 \
    --memory size=4G,file=/dev/hugepages/ch-ram-backing \
    --kernel /home/sky/work/test/cloudhypervisor/vmlinux.bin \
    --cmdline "root=/dev/vda1 console=hvc0 rw" \
    --disk path=/home/sky/work/test/cloudhypervisor/ubuntu-cloud-image/rootfs.raw


# 完成后关闭
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/file-backed-memory/cloud-hypervisor.sock shutdown

```

*注意：* Cloud Hypervisor 会在 /dev/hugepages/ 下创建名为 ch-ram-backing 的文件。如果该路径是 hugetlbfs 挂载点，CH 会自动识别并使用大页。
