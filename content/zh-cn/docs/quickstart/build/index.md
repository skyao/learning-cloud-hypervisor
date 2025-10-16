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

- 操作系统: linux mint 22
- 内核: 6.8

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

cargo build --release
sudo setcap cap_net_admin+ep ./target/release/cloud-hypervisor
```

此时构建好的 cloud-hypervisor 二进制文件在 ./target/release/ 目录下,可以执行命令看一下版本:

```bash
$ ./target/release/cloud-hypervisor --version

cloud-hypervisor v48.0-51-g68f9e8244
```

文件大小 3.9M. 真不大:

```bash
ls -lh ./target/release/cloud-hypervisor   
-rwxrwxr-x 2 sky sky 3.9M Oct 16 11:17 ./target/release/cloud-hypervisor
```

安全起见,不用 master 分支构建,改用最新的 release tag,然后采用静态链接:

```bash
rm -rf target

git checkout v48.0
cargo build --release --target=x86_64-unknown-linux-musl --all
sudo setcap cap_net_admin+ep ./target/x86_64-unknown-linux-musl/release/cloud-hypervisor

./target/x86_64-unknown-linux-musl/release/cloud-hypervisor --version
cloud-hypervisor v48.0

ls -lh ./target/x86_64-unknown-linux-musl/release/cloud-hypervisor 
-rwxrwxr-x 2 sky sky 4.0M Oct 16 15:31 ./target/x86_64-unknown-linux-musl/release/cloud-hypervisor
```

静态链接也才4M,没大多少.









