---
title: "DAX"
linkTitle: "DAX"
weight: 10
date: 2025-11-15
description: >
  Derect Access 直接访问
---




Virtio-fs + DAX (Direct Access): 这是高密部署的“杀手锏”。

- 原理： 传统的虚拟化中，Guest 读取文件会将其缓存在 Guest Page Cache 中，Host 也会有一份缓存，造成双重内存浪费。

- 方案： 使用 Virtio-fs 将 Host 的文件系统直接透传给 Guest。开启 DAX Window 功能，允许 Guest 直接映射 Host 的文件内容到自己的内存地址空间，绕过 Guest 的 Page Cache。

- 效果： 多个 MicroVM 共享同一个只读 Rootfs（底包），这些文件在物理内存中只有一份副本。这可以极大地节省内存。




