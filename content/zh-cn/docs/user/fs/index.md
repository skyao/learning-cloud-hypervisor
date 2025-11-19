---
title: "使用virtio-fs"
linkTitle: "virtio-fs"
weight: 30
date: 2025-11-15
description: >
  如何使用 virtio-fs 
---

https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/fs.md

----

在虚拟化环境中，能够从 host 共享目录到 guest 总是很方便。

virtio-fs，也称为 vhost-user-fs，是由 VIRTIO 规范定义的虚拟设备，它允许任何 VMM 执行文件系统共享。

## 前提条件

### 守护进程

这个虚拟设备依赖于 vhost-user 协议，该协议假设后端（设备模拟）由主机上运行的专用进程处理。这个守护进程称为 virtiofsd，并且需要在主机上存在。

构建 virtiofsd:

```bash
git clone https://gitlab.com/virtio-fs/virtiofsd
pushd virtiofsd
cargo build --release
sudo setcap cap_sys_admin+epi target/release/virtiofsd
```

创建共享目录:

```bash
mkdir /tmp/shared_dir
```

运行 virtiofsd:

```bash
./virtiofsd \
    --log-level debug \
    --socket-path=/tmp/virtiofs \
    --shared-dir=/tmp/shared_dir \
    --cache=never \
    --thread-pool-size=$N
```

当使用 Cloud Hypervisor 与 virtiofsd 时， cache=never 选项是默认设置。这会阻止使用主机页面缓存，从而减少对主机内存的整体占用。这增加了单个主机上可以启动的虚拟机的最大密度。

`cache=always` 选项将允许使用主机页面缓存，这可能会以增加对主机内存占用的代价，为 guest 的负载带来更好的性能。

thread-pool-size 选项控制生成多少个 IO 线程。对于非常快的存储如 NVMe，生成足够的工作线程对于与原生性能相比获得可接受的性能至关重要。

### 内核支持

现代 Linux 内核（至少 v5.10）支持 virtio-fs。使用较旧的内核，即使有额外的补丁，也不受支持。

## 如何与 cloud-hypervisor 共享目录

### 启动虚拟机

一旦守护进程运行，就需要使用 Cloud Hypervisor 的 --fs 选项。

只要云镜像包含足够新的内核，就可以使用直接内核启动和 EFI 固件来启动使用 virtio-fs 的虚拟机。

`--fs` 的正确运行需要 `--memory shared=on` 来促进进程间内存共享。

假设你的系统上已经安装了 focal-server-cloudimg-amd64.raw 和 vmlinux ，以下是需要运行的 Cloud Hypervisor 命令：

```bash
./cloud-hypervisor \
    --cpus boot=1 \
    --memory size=1G,shared=on \
    --disk path=focal-server-cloudimg-amd64.raw \
    --kernel vmlinux \
    --cmdline "console=hvc0 root=/dev/vda1 rw" \
    --fs tag=myfs,socket=/tmp/virtiofs,num_queues=1,queue_size=512
```

### 挂载共享目录

最后一步是在 guest 内部挂载共享目录，使用 virtiofs 文件系统类型。

```bash
mkdir mount_dir
mount -t virtiofs myfs mount_dir/
```

tag 需要与通过 Cloud Hypervisor 命令行提供的内容保持一致，在本例中恰好是 myfs 。


## DAX 功能

由于 DAX 功能从守护进程的角度来看还不稳定，因此它目前不可用。






