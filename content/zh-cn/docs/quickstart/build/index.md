---
title: "构建"
linkTitle: "构建"
weight: 20
date: 2025-10-15
description: >
  cloud hypervisor 编译和构建
---

也可以不下载,自己从源码开始构建.

> 参考: https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/building.md

## 准备

机器信息:

- 操作系统: debian13
- 内核: 6.12

准备工作:

```bash
mkdir -p $HOME/work/soft/cloudhypervisor

export CLOUDH=$HOME/work/soft/cloudhypervisor

cd $CLOUDH
```

安装依赖包:

```bash
sudo apt-get update
sudo apt install git build-essential m4 bison flex uuid-dev qemu-utils musl-tools

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

rustup target add x86_64-unknown-linux-musl
```

## 构建

clone 代码并开始构建:

```bash
mkdir -p ~/work/code/cloud-hypervisor
cd ~/work/code/cloud-hypervisor

git clone https://github.com/cloud-hypervisor/cloud-hypervisor.git
cd cloud-hypervisor
# 最新的 release tag
git checkout v50.0
# 采用静态链接
cargo build --release --target=x86_64-unknown-linux-musl --all
sudo setcap cap_net_admin+ep ./target/x86_64-unknown-linux-musl/release/cloud-hypervisor
```

检查构建之后的版本：

```bash
./target/x86_64-unknown-linux-musl/release/cloud-hypervisor --version
cloud-hypervisor v50.0

ls -lh ./target/x86_64-unknown-linux-musl/release/cloud-hypervisor 
-rwxrwxr-x 2 sky sky 4.0M Nov 18 15:23 ./target/x86_64-unknown-linux-musl/release/cloud-hypervisor
```

静态链接也才4M,没大多少.









