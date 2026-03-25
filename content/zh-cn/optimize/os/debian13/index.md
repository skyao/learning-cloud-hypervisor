---
title: "debian13"
linkTitle: "debian13"
weight: 30
date: 2025-11-15
description: >
  debian13 rootfs 优化
---

## 背景

由于我们为 microvm 场景定制的内核是 25.9MB 的全内置 (Monolithic) 内核，因此 Rootfs 制作将遵循一个核心原则：彻底剔除所有内核模块和硬件固件。

