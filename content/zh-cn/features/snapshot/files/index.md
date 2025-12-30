---
title: "快照文件信息"
linkTitle: "快照文件信息"
weight: 20
date: 2025-10-15
description: >
  cloud hypervisor 快照文件信息
---

## 文件列表

快照生成时，需要指定一个目录，然后会生成如下三个文件，以4G内存的虚拟机为例：

```bash
ls -lh
total 4.1G
-rw------- 1 root root 1.4K Dec 30 17:23 config.json
-rw------- 1 root root 4.0G Dec 30 17:23 memory-ranges
-rw------- 1 root root  42K Dec 30 17:23 state.json
```

## config.json

config.json 格式化之后的内容:

```json
{
  "cpus": {
    "boot_vcpus": 1,
    "max_vcpus": 1,
    "topology": null,
    "kvm_hyperv": false,
    "max_phys_bits": 46,
    "affinity": null,
    "features": {
      "amx": false
    },
    "nested": true
  },
  "memory": {
    "size": 4294967296,
    "mergeable": false,
    "hotplug_method": "Acpi",
    "hotplug_size": null,
    "hotplugged_size": null,
    "shared": false,
    "hugepages": false,
    "hugepage_size": null,
    "prefault": false,
    "zones": null,
    "thp": true
  },
  "payload": {
    "firmware": null,
    "kernel": "/home/sky/work/test/cloudhypervisor/vmlinux.bin",
    "cmdline": "root=/dev/vda1 console=hvc0 rw",
    "initramfs": null
  },
  "rate_limit_groups": null,
  "disks": [
    {
      "path": "/home/sky/work/test/cloudhypervisor/ubuntu-cloud-image/rootfs.raw",
      "readonly": false,
      "direct": false,
      "iommu": false,
      "num_queues": 1,
      "queue_size": 128,
      "vhost_user": false,
      "vhost_socket": null,
      "rate_limit_group": null,
      "rate_limiter_config": null,
      "id": "_disk0",
      "disable_io_uring": false,
      "disable_aio": false,
      "pci_segment": 0,
      "serial": null,
      "queue_affinity": null
    }
  ],
  "net": null,
  "rng": {
    "src": "/dev/urandom",
    "iommu": false
  },
  "balloon": null,
  "fs": null,
  "pmem": null,
  "serial": {
    "file": null,
    "mode": "Null",
    "iommu": false,
    "socket": null
  },
  "console": {
    "file": null,
    "mode": "Tty",
    "iommu": false,
    "socket": null
  },
  "debug_console": {
    "file": null,
    "mode": "Off",
    "iobase": 233
  },
  "devices": null,
  "user_devices": null,
  "vdpa": null,
  "vsock": null,
  "pvpanic": false,
  "iommu": false,
  "numa": null,
  "watchdog": false,
  "pci_segments": null,
  "platform": null,
  "tpm": null,
  "landlock_enable": false,
  "landlock_rules": null
}
```

这个 config.json 文件是 Cloud Hypervisor 在执行快照（Snapshot）时生成的，它记录了虚拟机在快照那一刻的静态配置信息。当恢复（Restore）快照时，Cloud Hypervisor 会读取这个文件来重建虚拟机的“骨架”（CPU、内存大小、磁盘路径等），然后再加载内存和设备状态。

关键配置解读:

### CPU 设置 (cpus)

- **boot_vcpus: 1**: 虚拟机启动时分配了 1 个 vCPU。
- **max_vcpus: 1**: 最大也是 1，说明这台 VM **不支持 CPU 热插拔**（如果要支持，max_vcpus 通常要大于 boot_vcpus）。
- **nested: true**: **嵌套虚拟化已启用**。这意味着您在这个 MicroVM 里面还可以运行 KVM（例如在 VM 里运行 Docker 或 QEMU）。这在测试环境中很有用。

### 内存设置 (memory)

- **size: 4294967296**: 即 4GB (4 * 1024 * 1024 * 1024)。
- **hotplug_method: "Acpi"**: 如果支持内存热插拔，将使用 ACPI 机制通知 Guest OS。
- **thp: true**: 启用了 **Transparent Huge Pages**（透明大页），这通常有助于减少 TLB miss，提升内存访问性能。

### 启动载荷 (payload)

这是 Direct Kernel Boot 的核心配置。

- **kernel**: `/home/sky/work/test/cloudhypervisor/vmlinux.bin` 

  注意：这里记录的是**绝对路径**。如果把快照目录复制到另一台机器上去恢复，必须确保那台机器的相同路径下也有这个内核文件，否则恢复会失败。

- **cmdline**: `root=/dev/vda1 console=hvc0 rw`

  - `root=/dev/vda1`: 指定根文件系统挂载点。
  - `console=hvc0`: **关键点**。Cloud Hypervisor 默认使用 Virtio Console (hvc0) 而不是传统的串口 (ttyS0)。如果您在启动时看不到输出，需要检查 Guest OS 是否启用了 hvc0 的 getty 服务。

### 磁盘设备 (disks)

- **path**: `/home/sky/work/test/cloudhypervisor/ubuntu-cloud-image/rootfs.raw`

  同样是**绝对路径**。恢复快照时，程序会去这个位置找磁盘文件。如果磁盘文件被删了或者移动了，恢复会报错。

- **readonly: false**: 磁盘是可读写的。

- **direct: false**: 没有开启 Direct I/O（O_DIRECT），意味着磁盘读写会经过宿主机的 Page Cache。

### 网络 (net)

- **null**: 您启动这个虚拟机时**没有配置网络接口**。它是一个“单机”环境，无法联网。

### 控制台与串口 (console & serial)

- **console (Virtio Console)**:
  - mode: "Tty"。这意味着虚拟机的 hvc0 输出被重定向到了宿主机运行 cloud-hypervisor 进程的当前终端（TTY）。
- **serial (Legacy Serial)**:
  - mode: "Null"。传统的串口（ttyS0）被禁用了/未连接。这就是为什么上面的 cmdline 必须用 console=hvc0 的原因。

### 随机数生成器 (rng)

- **src: "/dev/urandom"**: 

  虚拟机配置了 Virtio RNG 设备，透传宿主机的 /dev/urandom。这对于 MicroVM 来说很重要，可以加速虚拟机启动（避免因为熵不足导致启动阻塞）。

注意事项：

1. **路径依赖风险**：这个 config.json 文件对宿主机的文件路径（Kernel 和 Disk）有强依赖。如果在本地测试备份/恢复没问题。如果要在不同机器间迁移快照，需要在恢复前修改这个 json 文件里的路径，或者确保两台机器目录结构一致。

2. **嵌套虚拟化**：

   默认开启了 `nested: true`，这会增加一些 CPU 开销，如果不需要在 VM 里跑 VM，建议启动时去掉相关参数以提升性能。

   关闭的方法：在启动时配置 cpu为 nested off，例如   `--cpus boot=1,nested=off \`

## state.json

state.json 文件大小约为 42 KB。

在 Cloud Hypervisor 的快照体系中，state.json 与 config.json 扮演着完全不同但互补的角色。

如果说 config.json 是虚拟机的**“骨架”**（硬件配置），那么 state.json 就是虚拟机的**“灵魂”**（运行时的瞬间状态）。

### 存放的信息

文件记录了虚拟机在暂停那一瞬间，所有“非静态”的、动态变化的数据。具体包括：

- **vCPU 寄存器状态**：这是最核心的部分。包含：
  - **通用寄存器**：RAX, RBX, RSP (栈指针), RIP (指令指针 - 代码执行到哪一行了)。
  - **控制寄存器**：CR0, CR3 (页表地址), CR4 等。
  - **段寄存器**：CS, DS, SS 等。
  - **浮点数与向量寄存器**：用于数学计算的状态。
  - *简单说：它记录了 CPU 下一条指令要算什么，以及当前的计算结果存在哪。*
- **设备后端状态 (Virtio Devices)**：
  - **Virtio Queues (虚拟队列)**：记录了磁盘或网卡读写到哪一个位置了（Available Ring 和 Used Ring 的索引）。
  - **Feature Bits**：宿主机和虚拟机协商好了开启哪些功能（比如是否开启了巨型帧、多队列等）。
- **中断控制器状态 (vPIC / vIOAPIC / LAPIC)**：
  - 记录当前有哪些中断正在排队等待 CPU 处理，哪些中断被屏蔽了。
- **时钟与计时器**：
  - 虚拟机内部的时间偏移量，确保恢复后时间能尽量同步。


### 用处

它的唯一作用就是“现场还原”。

当执行 resume 或从文件恢复虚拟机时：

1. Cloud Hypervisor 先读取 config.json 创建出 CPU 线程和设备对象（造出机器）。
2. 然后读取 memory 文件把 RAM 数据填回去（恢复记忆）。
3. 最后读取 state.json，把每一个 CPU 寄存器的值填回去，把网卡/磁盘队列的指针拨回到原来的位置（恢复思维）。

如果没有 state.json，虚拟机虽然有内存数据，但 CPU 不知道该从内存的哪个地址开始执行指令，也不知道刚才磁盘读写到一半的数据该怎么处理，虚拟机就会立刻崩溃。

### 注意事项

1. 绝对不要手动修改

   config.json 里的磁盘路径可以手动改（比如迁移机器后路径变了），但 state.json 里的数据是极度敏感的二进制状态的序列化。

   - **风险**：改错一个标点或数字，CPU 恢复时的寄存器状态就会错乱，导致 Guest OS 报 Kernel Panic 或直接死机。
   - **建议**：视其为只读的二进制文件。

2. 版本强耦合 (Version Lock)

   Cloud Hypervisor 开发迭代很快，state.json 的内部结构（Schema）经常会变。

   - **风险**：用 v50.0.0 生成的 state.json，很大可能无法在 v51.0.0 或 v52.0.0 上恢复。
   - **建议**：生产环境中，**快照恢复必须使用与创建快照时完全相同版本的 Cloud Hypervisor**。

3. 与内存文件的原子性

   state.json 和同目录下的内存文件（通常叫 memory）是**即时对应**的。

   在 Cloud Hypervisor 的快照体系中，state（或者您看到的 state.json）与 config.json 扮演着完全不同但互补的角色。

   如果说 config.json 是虚拟机的**“骨架”**（硬件配置），那么 state.json 就是虚拟机的**“灵魂”**（运行时的瞬间状态）。

4. 包含敏感数据

   - **风险**：由于它保存了 CPU 寄存器，如果快照时某个程序正在处理密码或私钥，这些明文密钥可能正躺在 state.json 的寄存器值里。
   - **建议**：如果是在公有云或不可信环境传输快照，必须像保护磁盘文件一样保护 state.json。

5. CPU 拓扑一致性

   state.json 里的 vCPU 数据列表是按照 config.json 里的 CPU 数量顺序排列的。恢复时，宿主机不仅要提供相同数量的 CPU，最好连 CPU 的型号特性（Host CPU Model）也尽量保持一致，否则可能出现指令集不兼容（比如快照时用到了 AVX512，恢复的机器不支持）。


## memory-ranges




