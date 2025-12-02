---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-11-15
description: >
  调度优化概述
---


numa 感知： 在多 cpu NUMA 架构下，合理绑定 vCPU 与内存节点，避免跨 NUMA 访问带来的延迟。

cpu pinning： 将 vCPU 固定在物理 CPU 核心上，减少调度抖动



overcommit 超卖策略

- 在 cpu 层面适度超卖，但需要监控负载

- 内存层面依赖 KSM 和 ballooning，避免 OOM

和容器结合

- 在 microVM 内运行容器，进一步提升密度

