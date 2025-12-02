---
title: "Hugepages"
linkTitle: "Hugepages"
weight: 30
date: 2025-11-15
description: >
  Hugepages 大页内存
---

Hugepages (大页内存):

方案： 在 Host 上为 Cloud Hypervisor 进程分配 2MB 或 1GB 的 Hugepages。

效果： 减少 Host CPU 的 TLB (Translation Lookaside Buffer) Miss，降低页表本身的内存占用（Page Table Overhead）。对于高密场景，海量的 4KB 页表项会占用大量内存，使用 2MB 大页是必须的。


优化 Hugepage 使用，减少 TLB miss。



