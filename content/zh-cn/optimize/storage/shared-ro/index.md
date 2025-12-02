---
title: "共享只读文件"
linkTitle: "共享只读文件"
weight: 10
date: 2025-11-23
description: >
  Shared Read-Only Rootfs
---


Shared Read-Only Rootfs (共享只读根文件系统):

方案： 所有 MicroVM 共享同一个基础镜像作为只读层（通过 Virtio-fs DAX 挂载）。

写操作： 对于需要写的目录（如 /var, /tmp），挂载一个极小的、基于内存的 tmpfs，或者使用 OverlayFS 挂载一个空的 Host 目录。

效果： 磁盘 I/O 几乎全部转化为内存映射操作，极大降低存储带宽压力。

