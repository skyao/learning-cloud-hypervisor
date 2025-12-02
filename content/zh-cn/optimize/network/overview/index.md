---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-11-15
description: >
  网络优化概述
---


网络中断风暴是高密场景的隐形杀手。

Vhost-net / Vhost-user:

方案： 务必使用 vhost 后端。它将数据平面的处理下沉到 Host 内核（vhost-net），减少了用户态（Cloud Hypervisor）和内核态之间的上下文切换。

高级方案： 如果 BMS 网卡支持 SRIOV 且数量足够（通常不够高密），可以使用直通。更通用的高密方案是使用 Tap 设备 + Bridge 或者 MacVTap。

被动模式与中断合并:

在 Guest 内核中开启 NAPI，减少每个数据包的中断触发频率。




高效网络栈：

- 使用 vhost-net / vhost-user 加速网络 I/O

- 配合 DPDK 或者 SR-IOV 提升网络吞吐