---
title: "去除固件"
linkTitle: "去除固件"
weight: 40
date: 2025-12-01
description: >
  Direct Kernel Boot 去除固件
---

去除固件 (Direct Kernel Boot):

方案： Cloud Hypervisor 支持直接加载 Linux Kernel (vmlinux)，跳过 BIOS/UEFI 阶段。

效果： 既加快了启动速度，又节省了用于模拟固件的内存开销。




