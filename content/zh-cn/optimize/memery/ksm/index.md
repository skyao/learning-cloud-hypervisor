---
title: "KSM"
linkTitle: "KSM"
weight: 20
date: 2025-11-15
description: >
  Kernel Same-page Merging 内核页面合并
---


KSM (Kernel Same-page Merging):

- 方案： 在 Host (BMS) 内核开启 KSM。扫描内存页，将内容相同的页合并为只读页。

- 适用场景： 如果你的 MicroVM 运行着相同的应用程序或库，KSM 能进一步合并代码段和数据段内存。

注意： KSM 消耗 CPU 资源，需权衡扫描频率（sleep_millisecs）和 CPU 占用。



利用 ksm 合并系统内存页
