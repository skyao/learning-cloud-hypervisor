---
title: "Northflank介绍"
linkTitle: "介绍"
weight: 40
date: 2025-11-23
description: >
  Northflank介绍
---

https://northflank.com/

Northflank 在业界以提供极其安全、可扩展的计算沙箱而闻名，尤其是在当前的 AI Agent 时代，他们利用 Cloud Hypervisor 处理了海量的“不可信代码执行”任务。

以下是关于 Northflank 及其在 Cloud Hypervisor 上实践的详细资料梳理：

### 1. Northflank 是什么？

**Northflank** 是一个全栈的云原生 PaaS（平台即服务）平台，类似于国外的 Heroku、Vercel 或 Render。它的核心优势在于：允许开发者通过 UI、API、CLI 或 GitOps 极速部署容器、数据库、API 和后台任务。
近年来，随着大模型（LLM）的爆发，Northflank 演变成为了顶级的**“不可信代码执行沙箱（Untrusted Code Execution）”**平台。每月在生产环境中运行超过 **200 万个 MicroVM**，为许多 AI 初创公司和企业提供多租户隔离的底层基建。

### 2. Northflank 在 Cloud Hypervisor 上的深度实践

Northflank 并没有选择自己从头造轮子，也没有单一依赖 AWS 的 Firecracker，而是构建了一个极具弹性的底层调度架构：

#### A. 核心架构：Kata Containers + Cloud Hypervisor

*   **首选 VMM**：Northflank 将 **Cloud Hypervisor** 作为其微虚拟机隔离的**首选/主要 VMM（虚拟机监视器）**。

*   **结合 Kubernetes**：他们使用 **Kata Containers** 将 Cloud Hypervisor 融入到 Kubernetes 集群中。在用户看来，他们只是上传了一个标准的 Docker (OCI) 镜像；但在底层，Northflank 使用 Kata Containers 调用 Cloud Hypervisor，在 **~200毫秒**内为这个容器拉起一个拥有独立内核的微虚拟机（MicroVM）。

*   **为什么选 Cloud Hypervisor？** 根据其官方博客透露，相比于极其精简但功能受限的 Firecracker（启动 ~125ms），Cloud Hypervisor（启动 ~200ms）虽然慢了几十毫秒，但提供了企业级生产环境必须的关键特性：**CPU 和内存的热插拔（Hotplugging）、vhost-user 设备支持、以及更广泛的硬件兼容性**。同时，它同样基于 Rust 编写，具备极高的内存安全性。

#### B. 智能回退机制 (Fallback Strategy)

并非所有的云环境都支持嵌套虚拟化（Nested Virtualization）来运行硬件级虚拟机。Northflank 采用了一套智能调度策略：

*   在支持嵌套虚拟化的环境（如裸金属或特定的云主机），运行 **Kata + Cloud Hypervisor**（或 Firecracker）提供最强的硬件级隔离。

*   如果底层不支持硬件虚拟化，则自动无缝回退到使用 Google 的 **gVisor** 提供系统调用（Syscall）级别的用户态隔离。

#### C. 开源社区贡献

*   Northflank 的工程团队不仅是使用者，还是**积极的社区贡献者**。他们向上游的 Kata Containers、Cloud Hypervisor、QEMU 和 containerd 提交代码，以确保他们依赖的隔离层得到持续的维护和优化。

### 3. 主要的应用场景

*   **AI Agent 代码执行沙箱**：当 AI Agent 需要自己写代码并执行以完成任务时，Northflank 利用 Cloud Hypervisor 秒级拉起沙箱，确保 AI 生成的恶意代码或死循环绝对无法穿透沙箱影响宿主机。

*   **BYOC (自带云部署)**：Northflank 允许企业客户将其调度引擎部署在客户自己的 AWS、GCP 或裸机机房内（Bring Your Own Cloud）。这意味着客户可以在自己的私有 VPC 内，利用 Cloud Hypervisor 享受极速的 MicroVM 隔离体验，无需自己搭建复杂的虚拟化底层。

备注：这家公司的 blog 上有不少写的不错的介绍文章。

