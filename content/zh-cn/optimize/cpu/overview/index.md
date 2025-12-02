---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-11-15
description: >
  cpu 优化概述
---




高密部署意味着 CPU 必然是超分 (Overcommit) 的。

- vCPU 拓扑:

   建议 MicroVM 配置为 1 vCPU。对于微服务或函数计算场景，单核足以处理，且调度开销最小。

- Host 调度器调整:

不要在 Host 上对 MicroVM 进行严格的物理核绑定 (Pinning)，因为高密场景下这会导致碎片化。依赖 Linux CFS (Completely Fair Scheduler) 进行调度。

   Halt Polling: 调整 KVM 的 halt_poll_ns。对于高密且 I/O 频繁的场景，适当降低该值，让 vCPU 空闲时尽快释放物理 CPU 资源给其他 VM，而不是空转等待唤醒。

- VMM (Virtual Machine Monitor) 进程开销:

   Cloud Hypervisor 是多线程模型。确保其辅助线程（如 vhost-user 线程, API 线程）不会抢占过多的计算资源。




