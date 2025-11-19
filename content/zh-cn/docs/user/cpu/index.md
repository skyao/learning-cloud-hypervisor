---
title: "虚拟CPU配置"
linkTitle: "CPU"
weight: 20
date: 2025-11-15
description: >
  cloud hypervisor 用户文档概述
---

https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/cpu.md

-----

在创建虚拟 CPU 方面，Cloud Hypervisor 提供了多种选项。本文档旨在解释 Cloud Hypervisor 的功能以及如何使用它来满足不同用例的需求。

## Options  选项

CpusConfig 或从 CLI 的角度称为 `--cpus` 是设置 Cloud Hypervisor vCPU 选项的方式。

```rust
struct CpusConfig {
    boot_vcpus: u8,
    max_vcpus: u8,
    topology: Option<CpuTopology>,
    kvm_hyperv: bool,
    max_phys_bits: u8,
    affinity: Option<Vec<CpuAffinity>>,
    features: CpuFeatures,
}
```

```bash
--cpus boot=<boot_vcpus>,max=<max_vcpus>,topology=<threads_per_core>:<cores_per_die>:<dies_per_package>:<packages>,kvm_hyperv=on|off,max_phys_bits=<maximum_number_of_physical_bits>,affinity=<list_of_vcpus_with_their_associated_cpuset>,features=<list_of_features_to_enable>
```

### boot

启动时存在的 vCPU 数量。

此选项允许定义虚拟机启动时存在的特定 vCPU 数量。在使用 --cpus 参数时，此选项是必需的。如果未指定 --cpus ，此选项将采用默认值 1，以单个 vCPU 启动虚拟机。

值是一个 8 位的无符号整数。

```bash
--cpus boot=2
```

### max

最大 vCPU 数量。

此选项定义可分配给虚拟机的最大 vCPU 数量。特别是，当想用 CPU 热插拔时使用此选项，因为它可以提供有关在虚拟机运行期间可能需要多少 vCPU 的指示。例如，如果使用 2 个 vCPU 启动虚拟机，并且最大限制为 6 个 vCPU，这意味着最多可以在运行时通过调整虚拟机大小添加 4 个 vCPU。

该值必须大于或等于启动 vCPU 的数量。该值是一个 8 位的无符号整数。

默认情况下，此选项取 boot 的值，意味着不期望 vCPU 热插拔，也无法执行。

```bash
--cpus max=3
```

### topology

guest 平台的拓扑结构。

该选项为用户提供了一种描述应向 guest 暴露的确切拓扑结构的方法。它可用于向 guest 描述与主机上相同的拓扑结构，因为这允许正确使用资源，并且是提高性能的一种方式。

拓扑结构通过以下结构描述：

```rust
struct CpuTopology {
    threads_per_core: u8,
    cores_per_die: u8,
    dies_per_package: u8,
    packages: u8,
}
```

或者通过 CLI 使用以下语法：

```bash
topology=<threads_per_core>:<cores_per_die>:<dies_per_package>:<packages>
```

默认情况下，拓扑结构将是 1:1:1:1 。示例:

```bash
--cpus boot=2,topology=1:1:2:1
```

### kvm_hyperv

启用 KVM Hyper-V 模拟。

当启用此选项时，它依赖于 KVM 来模拟合成中断控制器（SynIC）以及 Windows guest 所期望的合成计时器。Windows 客机通常在 Microsoft Hyper-V 上运行，因此期望这些合成设备存在。这就是为什么 KVM 提供了一种模拟它们的方法，并避免了使用 Cloud Hypervisor 运行 Windows guest 时出现的故障。

默认情况下，此选项是关闭的。

### max_phys_bits

最大可寻址空间大小。

此选项定义了所有 vCPU 的最大物理位数，这为虚拟机的可寻址空间大小设置了限制。这主要用于调试目的。

### affinity

每个 vCPU 的亲和性。

此选项为用户提供了一种方式来指定每个 vCPU 关联的主机 CPU 集。这对于实现 CPU 固定很有用，可以确保多个虚拟机不会互相影响性能。在 NUMA 上下文中，它也可以被使用，因为它是一种确保虚拟机可以在特定主机 NUMA 节点上运行的方式。通常，此选项用于根据主机平台和虚拟机中运行的负载类型来提高虚拟机的性能。

亲和性通过以下结构描述：

```rust
struct CpuAffinity {
    vcpu: u8,
    host_cpus: Vec<u8>,
}
```

或者通过 CLI 使用以下语法：

```bash
affinity=[<vcpu_id1>@[<host_cpu_id1>, <host_cpu_id2>], <vcpu_id2>@[<host_cpu_id3>, <host_cpu_id4>]]
```

外层括号定义了 vCPU 的列表。而对于每个 vCPU，附在 @ 上的内层括号定义了该 vCPU 被允许运行的主 CPU 列表。

可以提供多个值来定义每个列表。每个值是一个 8 位的无符号整数。

例如，如果需要在从 0 到 4 的主 CPU 上运行 vCPU 0，使用 `-` 的语法将有助于定义一个连续的范围，配合 affinity=0@[0-4] 使用。同一个例子也可以用 affinity=0@[0,1,2,3,4] 来描述。

当需要描述一个包含从 0 到 99 的主 CPU 和主 CPU 255 的列表时，同时使用 `-` 和 `,` 分隔符会很有用，因为这样可以用 affinity=0@[0-99,255] 简单地描述。

一旦试图描述一组值，就必须使用 [ 和 ] 来界定这组值。

默认情况下，每个 vCPU 都在整个主机 CPU 集上运行。

```bash
--cpus boot=3,affinity=[0@[2,3],1@[0,1]]
```

在这个例子中，假设主机有 4 个 CPU，vCPU 0 将专门在主机的 CPU 2 和 3 上运行，而 vCPU 1 将专门在主机的 CPU 0 和 1 上运行。由于 vCPU 2 没有定义，它可以在任何 4 个主 CPU 上运行。

### features

要启用的 CPU 功能集。

此选项允许用户启用一组默认情况下被禁用的 CPU 特性。

当前可用的特性集是： amx

amx 功能将启用 x86 扩展，为矩阵运算（整数和浮点数点积）添加硬件单元。该扩展的目标是为这些常见操作提供性能提升。

```bash
--cpus features=amx
```

在这个示例中，将为 VMM 启用 amx CPU 功能。

