---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-11-15
description: >
  裁剪概述
---



设备模型最小化：只保留必需的 paravirt 设备（virtio-net、virtio-blk/virtio-fs、vsock），禁用未用的 ACPI/PCI 设备与中断源，减少 emulation 成本。

关闭不必要的设备模拟，如 pci 设备，复杂 I/O

使用 virtio 驱动，减少模拟开销



