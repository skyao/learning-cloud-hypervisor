---
title: "精简 Guest Kernel"
linkTitle: "精简 Guest Kernel"
weight: 20
date: 2025-11-23
description: >
  精简 Guest Kernel
---


精简 Guest Kernel:

方案： 定制 Linux Kernel。移除所有不需要的驱动（USB, GPU, Sound, 传统的 PCI 等）。只保留 virtio 相关的驱动（virtio-net, virtio-blk, virtio-fs, virtio-console）。

效果： 减小 Kernel 体积（建议压缩后小于 5MB），减少启动时的内存申请，加快启动速度。

