---
title: "Cloud Hypervisor指南"
linkTitle: "Cloud Hypervisor指南"
weight: 10
date: 2025-10-07
description: >
  Cloud Hypervisor指南
---

Guide to Cloud Hypervisor in 2026: Modern VMM for cloud workloads

https://northflank.com/blog/guide-to-cloud-hypervisor

----

Cloud Hypervisor 是一个用 Rust 编写的开源虚拟机监控器，专为现代云工作负载而设计。它通过轻量级虚拟机提供硬件级隔离，同时支持 CPU 和内存热插拔、虚拟主机用户设备以及 Kata 容器集成。

它能在约 200 毫秒内启动虚拟机，并可在 x86-64 和 AArch64 架构的 KVM 和 Microsoft Hypervisor 上运行。该项目隶属于 Linux 基金会，通常与 Kata Containers 结合使用，用于云工作负载。

## 什么是 Cloud Hypervisor?

Cloud Hypervisor 是一种虚拟机监控程序(Virtual Machine Monitor,缩写为VMM），用于创建和管理用于云工作负载的轻量级虚拟机。

与注重灵活性和对传统硬件支持的传统虚拟机管理程序不同，Cloud Hypervisor 专注于在云环境中运行的现代操作系统。该项目最初是英特尔对 Rust 虚拟机管理程序 (VMM) 生态系统的贡献，并借鉴了 Firecracker 和 crosvm 的经验。

Firecracker 优先考虑无服务器工作负载的极简主义，而 QEMU 优先考虑满足所有可能用例的完整性，Cloud Hypervisor 则力求达到中间水平：提供足够的功能来处理生产工作负载，而不会造成不必要的复杂性。

Cloud Hypervisor 实现了云应用程序实际需要的现代虚拟化功能：通过 virtio 设备实现半虚拟化 I/O、CPU 和内存热插拔、通过 VFIO 实现设备直通以及与容器编排平台集成。

## Cloud Hypervisor 架构

Cloud Hypervisor 基于 Rust VMM 项​​目构建，与 Firecracker 和 crosvm 共享虚拟化组件。

关键的架构选择包括最小的设备仿真（现代工作负载只需要 16 个设备）、通过 virtio 实现网络和存储的半虚拟化、通过 REST 进行 API 驱动的管理，以及 Rust 的内存安全机制来防止常见的漏洞。

| Feature 特征       | Specification 规格               |
| :----------------- | :------------------------------- |
| 语言               | Rust（内存安全）                 |
| 代码大小           | 约50,000行                       |
| 启动时间           | 约200毫秒                        |
| 架构               | x86-64, AArch64 x86-64，AArch64  |
| 客户操作系统支持   | Linux, Windows 10/Server 2019    |
| 虚拟机管理程序后端 | KVM, Microsoft Hypervisor (MSHV) |

## Cloud Hypervisor 与其他虚拟机管理程序 (VMM) 相比如何？

了解 Cloud Hypervisor 相对于 Firecracker 和 QEMU 的位置，有助于明确其设计上的权衡取舍。

- **对比 Firecracker：** Cloud Hypervisor 的启动时间约为 200 毫秒，而 Firecracker 的启动时间约为 125 毫秒。多出的 75 毫秒支持 CPU 和内存热插拔、虚拟主机用户设备以及更广泛的硬件兼容性，而 Firecracker 则刻意省略了这些功能。Firecracker 针对短暂的无服务器功能进行了优化，而 Cloud Hypervisor 则面向需要运行时灵活性的长时间运行工作负载。
- **对比 QEMU：** Cloud Hypervisor 约有 5 万行 Rust 代码，而 QEMU 约有 200 万行 C 代码。QEMU 模拟了 40 多种设备，包括传统硬件；Cloud Hypervisor 仅实现了 16 种现代设备。Cloud Hypervisor 的启动速度明显更快（约 200 毫秒，而 QEMU 需要几秒），并且针对云工作负载提供了合理的默认设置，而 QEMU 则需要大量的配置。

| Factor 因素   | Cloud Hypervisor         | Firecracker           | QEMU             |
| :------------ | :----------------------- | :-------------------- | :--------------- |
| 代码大小**    | 约 5 万行代码（Rust）    | 约 5 万行代码（Rust） | 约 200 万行（C） |
| **启动时间**  | 约200毫秒                | 约125毫秒             | 几秒钟           |
| **热插拔**    | CPU、内存、设备          | 不支持                | 是的（复杂）     |
| **GPU 支持**  | 有限                     | 不支持                | 完整（VFIO）     |
| **Kata 集成** | 支持（云平台中常用选项） | Supported 支持        | 支持（默认）     |

## Cloud Hypervisor 的主要特性有哪些？

Cloud Hypervisor 提供了多种功能，使其适用于生产云工作负载。

### CPU 和内存热插拔

Cloud Hypervisor 无需重启即可向正在运行的虚拟机添加 CPU 和内存，从而使工作负载能够动态扩展资源。

CPU 热插拔的工作原理是：虚拟机管理程序创建新的虚拟 CPU（vCPU），并通过 ACPI 将其通告给客户机内核。然后，客户机操作系统将这些新 CPU 上线。内存热插拔则是在主机上分配额外的内存（以 128 MiB 的倍数为单位），并将其映射到客户机的地址空间。

### Vhost-user 设备支持

Vhost-user 将设备模拟卸载到单独的进程中，从而提高性能和安全性。

Cloud Hypervisor 不直接在虚拟机管理程序 (VMM) 进程中处理 I/O，而是将其委托给专门的守护进程。这种架构通过在专用内核上运行设备处理程序来实现更高的 I/O 吞吐量，通过将设备逻辑与 VMM 分离来实现更好的隔离，并通过标准的虚拟主机用户协议简化设备实现。

### Virtio 设备型号

Cloud Hypervisor 使用半虚拟化的 virtio 设备进行所有 I/O 操作。

这包括用于网络连接的 virtio-net、用于块存储的 virtio-blk、用于文件系统共享的 virtio-fs、用于主机-客户机通信的 virtio-vsock 以及用于持久内存的 virtio-pmem。Virtio 避免模拟真实硬件，提供客户机和主机都能理解的简洁接口，从而消除开销并实现更佳性能。

### VFIO 设备直通

Cloud Hypervisor 支持使用 VFIO 将物理设备直接传递给虚拟机。

这为需要直接硬件访问的 PCIe 设备（例如网卡或加速器）提供了接近原生性能。虽然 QEMU 对 VFIO 的支持更为成熟，但 Cloud Hypervisor 能够提供大多数云用例所需的功能。

## Cloud Hypervisor 如何与 Kubernetes 集成？

Kata Containers 支持多种虚拟机管理程序（包括 QEMU 和 Cloud Hypervisor）。许多云平台都使用 Cloud Hypervisor 运行 Kata，以支持 Kubernetes microVM工作负载。

当您使用 Kata 的 Cloud Hypervisor 运行时类部署容器时，Kata 会处理所有编排工作：配置虚拟机、启动最小化的客户机内核、挂载容器镜像、管理客户机和主机之间的网络连接以及处理虚拟机生命周期。从 Kubernetes 的角度来看，它就是一个普通的容器。但在底层，它是一个具有硬件隔离的完整虚拟机。

这种集成使得需要虚拟机级安全且无需构建自定义基础架构的团队能够轻松使用 Cloud Hypervisor。Northflank平台利用 Kata Containers 与 Cloud Hypervisor 的集成，为生产工作负载提供硬件级隔离。您只需使用标准的 Kubernetes YAML 进行部署，Kata 便会处理 VMM 的复杂性。

## Cloud Hypervisor 有哪些局限性？

与功能齐全的虚拟机管理程序相比，Cloud Hypervisor 会做出一些有意的权衡，从而造成一些限制。

- **不支持传统硬件：** cloud hypervisor不会模拟软盘驱动器、PS/2 键盘或 ISA 总线等传统设备。现代云工作负载不再需要这些设备，而且模拟它们会增加复杂性并扩大攻击面。
- **GPU 支持有限：** 虽然 Cloud Hypervisor 通过 VFIO 支持一些 GPU 直通场景，但 QEMU 的实现更加成熟，可以处理更广泛的 GPU 和配置。
- **Windows 支持正在不断发展：** cloud hypervisor支持 Windows 10 和 Windows Server 2019，但其实现成熟度不如 Linux 支持。大多数生产部署都运行 Linux 虚拟机。
- **快照稳定性：** 快照/恢复和实时迁移功能虽然存在，但无法保证在不同版本之间保持稳定。生产环境部署应在使用前对这些功能进行全面测试。

## 何时应该使用cloud hypervisor？

了解cloud hypervisor的理想用例有助于您确定它是否符合您的需求。

- **Kubernetes 中的生产微型虚拟机：** Kubernetes 中的生产微型虚拟机：当运行硬件隔离的容器工作负载时，cloud hypervisor是 Kata Containers 常用的虚拟机管理程序。
- **多租户云工作负载：** SaaS 平台、代码执行环境、AI 沙箱和客户部署都受益于cloud hypervisor在安全性和功能性方面的平衡。
- **需要运行时灵活性的工作负载：** 如果您的应用程序需要 CPU/内存热插拔、vhost-user 设备或比 Firecracker 提供的更广泛的硬件支持，Cloud Hypervisor 可以提供这些功能，而无需 QEMU 的复杂性。
- **内存安全性：** Rust 的内存安全机制可以防止影响基于 C 语言的虚拟机管理程序的一系列漏洞。与 QEMU 相比，Rust 的代码库更小，这意味着潜在的攻击途径也更少。



## VMM 之间的优缺点是什么？

不同的虚拟机管理程序会根据其目标用例做出不同的权衡。

| VMM              | 设计重点                 | 最适合                                                       |
| :--------------- | :----------------------- | :----------------------------------------------------------- |
| Cloud Hypervisor | 功能与极简主义的平衡     | 需要虚拟机隔离和运行时灵活性（热插拔、虚拟主机用户）的生产级 Kubernetes 工作负载。常与 Kata Containers 配合使用，用于云工作负载。 |
| Firecracker      | 极简主义和速度           | 无服务器函数需要尽可能快的启动速度和最小的资源占用。AWS Lambda 是临时工作负载的基础。 |
| QEMU             | 最大限度的灵活性和兼容性 | 支持全系统仿真、GPU 工作负载、传统硬件和桌面虚拟化。灵活性最高，但攻击面也最大。 |

## 关于cloud hypervisor的常见问题

### Cloud Hypervisor 和 Firecracker 的主要区别是什么？

Cloud Hypervisor 提供的功能比 Firecracker 更丰富（例如 CPU/内存热插拔、虚拟主机用户、更广泛的设备支持），同时保持了以安全为中心的设计理念。Firecracker 启动速度更快（约 125 毫秒对比约 200 毫秒），但为了保持简洁，它刻意省略了一些功能。Cloud Hypervisor 的目标应用是长时间运行的云工作负载，而 Firecracker 的目标应用是短暂的无服务器功能。

### Cloud Hypervisor 比 QEMU 更安全吗？

cloud hypervisor的代码库规模要小得多（Rust 代码约 5 万行，而 C 代码约 200 万行），并且使用内存安全语言编写，从而降低了潜在的安全漏洞。然而，QEMU 已经过数十年的实战检验，并接受了广泛的安全审计。对于现代云工作负载而言，cloud hypervisor较小的攻击面通常更具优势。

### cloud hypervisor是否支持 GPU 直通？

Cloud Hypervisor 通过 VFIO 支持部分 GPU 直通场景，但其实现不如 QEMU 成熟。对于生产环境的 GPU 工作负载，请务必使用 Cloud Hypervisor 对您的特定 GPU 和驱动程序进行全面测试，或者考虑使用 QEMU 以获得更可靠的 GPU 支持。

### cloud hypervisor可以运行 Windows 吗？

是的，Cloud Hypervisor 支持 Windows 10 和 Windows Server 2019。但是，Linux 虚拟机获得了更多的开发关注，并且在生产环境中的部署也更为广泛。在生产环境部署之前，请务必对 Windows 工作负载进行全面测试。

### Cloud Hypervisor 如何与 Kata Containers 集成？

Kata Containers 支持多种虚拟机管理程序 (VMM)，包括 QEMU 和 Cloud Hypervisor。某些平台默认将 Kata 配置为使用 Cloud Hypervisor。在 Kubernetes 中指定 kata-clh 运行时类后，Kata 会自动使用 Cloud Hypervisor 创建和管理虚拟机。此集成处理了所有运维复杂性，使用户可以通过标准的 Kubernetes API 访问 Cloud Hypervisor。

### 我可以在 Kubernetes 之外使用 Cloud Hypervisor 吗？

是的，Cloud Hypervisor 可以独立运行。但是，大多数生产部署都通过 Kata Containers 将其与 Kubernetes 集成。直接运行 Cloud Hypervisor 需要手动进行虚拟机生命周期管理、网络配置和编排。
