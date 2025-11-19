---
title: "热拔插"
linkTitle: "热拔插"
weight: 40
date: 2025-11-15
description: >
  cloud hypervisor 热拔插
---

https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/hotplug.md

----

目前 cloud hypervisor 支持 CPU 设备（仅限 x86）、PCI 设备和内存调整的热插拔。

## 内核支持

在 cloud hypervisor 上进行热插拔需要 ACPI GED 支持。这可以通过开启 CONFIG_ACPI_REDUCED_HARDWARE_ONLY 或使用此内核补丁（在 5.5-rc1 及更高版本中可用）：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/patch/drivers/acpi/Makefile?id=ac36d37e943635fc072e9d4f47e40a48fbcdb3f0


## CPU 热插拔

额外的 vCPU 可以在运行的 cloud-hypervisor 实例中添加和移除。这由两种机制控制：

1. 指定一个大于默认（boot）vCPU 数量的最大潜在 vCPU 数。

2. 向 VMM 发出 HTTP API 请求，要求添加额外的 vCPU。

要使用 CPU 热插拔功能，请以大于启动 vCPU 数量的最大 vCPU 数启动虚拟机，例如。

```bash
$ pushd $CLOUDH
$ sudo setcap cap_net_admin+ep ./cloud-hypervisor/target/release/cloud-hypervisor
$ ./cloud-hypervisor/target/release/cloud-hypervisor \
	--kernel custom-vmlinux.bin \
	--cmdline "console=ttyS0 console=hvc0 root=/dev/vda1 rw" \
	--disk path=focal-server-cloudimg-amd64.raw \
	--cpus boot=4,max=8 \
	--memory size=1024M \
	--net "tap=,mac=,ip=,mask=" \
	--rng \
	--api-socket=/tmp/ch-socket
$ popd
```

注意 --api-socket=/tmp/ch-socket 和 max 参数在 --cpus boot=4,max=8 中的添加。

若要请求 VMM 添加额外的 vCPU，请使用 resize API：

```bash
./ch-remote --api-socket=/tmp/ch-socket resize --cpus 8
```

额外的 vCPU 线程将被创建并向运行中的内核进行通告。内核不会立即启动 CPU，相反，用户必须从虚拟机内部将它们 "online" 化：

```bash
root@ch-guest ~ # lscpu | grep list:
On-line CPU(s) list:             0-3
Off-line CPU(s) list:            4-7
root@ch-guest ~ # echo 1 | tee /sys/devices/system/cpu/cpu[4,5,6,7]/online
1
root@ch-guest ~ # lscpu | grep list:
On-line CPU(s) list:             0-7
```

重启后，添加的 CPU 将保持在线状态。

移除 CPU 的操作与减少 resize API 中"desired_vcpus"字段的数值类似。CPU 将在虚拟机内部自动下线，因此无需在虚拟机内部运行任何命令：

```bash
./ch-remote --api-socket=/tmp/ch-socket resize --cpus 2
```

关于向虚拟机添加 CPU，重启后虚拟机将以减少的 vCPU 数量运行。

## 内存热插拔

### ACPI 方法

可以从正在运行的 cloud-hypervisor 实例中添加额外内存。这由两种机制控制：

1. 为热插拔内存分配部分 guest 物理地址空间。

2. 向 VMM 发起 HTTP API 请求，要求为虚拟机分配新的内存量。在为虚拟机扩展内存的情况下，新内存将被热插拔到正在运行的虚拟机中；如果减少内存大小，则更改将在下次重启后生效。

要使用内存热插拔，启动虚拟机时在内存配置的 hotplug_size 参数中指定一些 RAM 大小。这个参数中指定的所有内存并不都会用于热插拔，因为存在间隔和对齐要求，所以建议使其大于所需的热插拔 RAM 大小。

由于 ACPI 方法为默认选项，因此无需添加额外的选项 hotplug_method=acpi 。

```bash
$ pushd $CLOUDH
$ sudo setcap cap_net_admin+ep ./cloud-hypervisor/target/release/cloud-hypervisor
$ ./cloud-hypervisor/target/release/cloud-hypervisor \
	--kernel custom-vmlinux.bin \
	--cmdline "console=ttyS0 console=hvc0 root=/dev/vda1 rw" \
	--disk path=focal-server-cloudimg-amd64.raw \
	--cpus boot=4,max=8 \
	--memory size=1024M,hotplug_size=8192M \
	--net "tap=,mac=,ip=,mask=" \
	--rng \
	--api-socket=/tmp/ch-socket
$ popd
```

在发出 API 请求之前，需要在虚拟机内部运行以下命令，以使其自动上线添加的内存：

```bash
root@ch-guest ~ # echo online | sudo tee /sys/devices/system/memory/auto_online_blocks
```

要求 VMM 扩展虚拟机的 RAM：

```bash
./ch-remote --api-socket=/tmp/ch-socket resize --memory 3G
```

新内存现在可以在虚拟机内部使用：

```bash
free -h
              total        used        free      shared  buff/cache   available
Mem:          3.0Gi        71Mi       2.8Gi       0.0Ki        47Mi       2.8Gi
Swap:          32Mi          0B        32Mi
```

由于 guest 操作系统限制，必须确保添加的内存量（当前分配的 RAM 与所需 RAM 之间的差值）是 128MiB 的倍数。

相同的 API 也可以用来减少虚拟机的所需 RAM，但更改不会在虚拟机重新启动后生效。

内存和 CPU 调整可以组合在同一个 HTTP API 请求中。

### virtio-mem 方法

额外的内存可以从正在运行的 Cloud Hypervisor 实例中添加和移除。这由两种机制控制：

1. 为热插拔内存分配部分客户物理地址空间。

2. 向 VMM 发起 HTTP API 请求，请求为虚拟机分配新的内存量。

要使用内存热插拔功能，启动虚拟机时需要在 hotplug_size 参数中指定一些内存大小，并在内存配置中使用 hotplug_method=virtio-mem 。

```bash
$ pushd $CLOUDH
$ sudo setcap cap_net_admin+ep ./cloud-hypervisor/target/release/cloud-hypervisor
$ ./cloud-hypervisor/target/release/cloud-hypervisor \
	--kernel custom-vmlinux.bin \
	--cmdline "console=ttyS0 console=hvc0 root=/dev/vda1 rw" \
	--disk path=focal-server-cloudimg-amd64.raw \
	--memory size=1024M,hotplug_size=8192M,hotplug_method=virtio-mem \
	--net "tap=,mac=,ip=,mask=" \
	--api-socket=/tmp/ch-socket
$ popd
```

要求 VMM 为虚拟机扩展 RAM（请求单位为字节）：

```bash
./ch-remote --api-socket=/tmp/ch-socket resize --memory 3G
```

新内存现在可以在虚拟机内部使用：


相同的 API 也可以用来减少虚拟机的所需 RAM。需要注意的是，减少 RAM 大小可能只能部分生效，因为 guest 可能正在使用其中的一部分。

### PCI 设备热插拔

### 添加 VFIO 设备

稍后看,目前没需要.

### 添加磁盘设备

要让 VMM 添加额外的磁盘设备，请使用 add-disk API。

```bash
./ch-remote --api-socket=/tmp/ch-socket add-disk path=/foo/bar/cloud.img
```

### 添加文件系统设备

要请求 VMM 添加额外的文件系统设备，请使用 add-fs API。

```bash
./ch-remote --api-socket=/tmp/ch-socket add-fs tag=myfs,socket=/foo/bar/virtiofs.sock
```

### 添加网络设备

要求 VMM 添加额外的网络设备，请使用 add-net API。

```bash
./ch-remote --api-socket=/tmp/ch-socket add-net tap=chtap0
```

### 添加 Pmem 设备

稍后看,目前没需要.

### 添加 Vsock 设备

若要请求 VMM 添加额外的 vsock 设备，请使用 add-vsock API。

```bash
./ch-remote --api-socket=/tmp/ch-socket add-vsock cid=3,socket=/foo/bar/vsock.sock
```

### 所有 PCI 设备的通用特性

额外的 PCI 设备将被创建并向正在运行的内核进行通告。新设备可以通过检查 PCI 设备列表来找到。

```bash
root@ch-guest ~ # lspci
00:00.0 Host bridge: Intel Corporation Device 0d57
00:01.0 Unassigned class [ffff]: Red Hat, Inc. Virtio console (rev 01)
00:02.0 Mass storage controller: Red Hat, Inc. Virtio block device (rev 01)
00:03.0 Unassigned class [ffff]: Red Hat, Inc. Virtio RNG (rev 01)
```

重启后，添加的 PCI 设备将保持存在。

### 移除 PCI 设备

移除 PCI 设备对所有类型的 PCI 设备都采用相同的方式。必须提供与设备相关的唯一标识符。此标识符可以在添加新设备时由用户提供，或者默认情况下由 Cloud Hypervisor 分配。

```bash
./ch-remote --api-socket=/tmp/ch-socket remove-device _disk0
```

在为虚拟机添加 PCI 设备后，重启后虚拟机将不再运行已移除的 PCI 设备。



