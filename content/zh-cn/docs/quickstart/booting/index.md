---
title: "启动"
linkTitle: "启动"
weight: 30
date: 2025-10-15
description: >
  启动 cloud hypervisor
---

参考:

https://www.cloudhypervisor.org/docs/prologue/quick-start/#booting-linux

在 linux 上启动.

Cloud Hypervisor 支持直接内核启动（x86-64内核需使用支持PVH构建的内核）或通过固件/firmware启动（可选Rust Hypervisor固件或名为 `CLOUDHV/CLOUDHV_EFI` 的edk2 UEFI固件）。

最新版本的Rust Hypervisor固件及edk2存储库中均提供固件文件的二进制构建版本。

固件的选择取决于您的客户操作系统选项，可能需要进行一些实验性测试。


## 固件启动

Cloud Hypervisor 支持引导包含运行云工作负载所需全部组件的磁盘镜像，即云镜像。

### 准备工作

首先安装 qemu-utils,稍后要用到其中的 qemu-img :

```bash
sudo apt install qemu-utils
```

以下示例命令将下载Ubuntu云镜像，将其转换为 Cloud Hypervisor 可用的格式，并生成引导该镜像所需的固件。

```bash
mkdir -p ~/work/soft/cloudhypervisor/fireware
cd ~/work/soft/cloudhypervisor/fireware

wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

qemu-img convert -p -f qcow2 -O raw focal-server-cloudimg-amd64.img focal-server-cloudimg-amd64.raw

wget https://github.com/cloud-hypervisor/rust-hypervisor-firmware/releases/download/0.5.0/hypervisor-fw
```

处理完成后的文件列表:

```bash
$ ls -lh    

total 2.1G
-rw-rw-r-- 1 sky sky 618M Jun 25 09:39 focal-server-cloudimg-amd64.img
-rw-r--r-- 1 sky sky 2.2G Oct 16 16:54 focal-server-cloudimg-amd64.raw
-rw-rw-r-- 1 sky sky 133K Nov 30  2024 hypervisor-fw
```

Ubuntu云镜像默认不提供密码，因此首次启动时必须使用 cloud-init 磁盘镜像进行镜像定制。本脚本可生成基础 cloud-init 镜像，为镜像预设默认用户名/密码为 `cloud/cloud123` 。仅需在首次启动时添加此磁盘镜像即可。脚本同时通过 `--net “mac=12:34:56:78:90:ab,tap=”` 选项，基于 test_data/cloud-init/ubuntu/local/network-config 配置信息分配默认IP地址。随后系统将根据 network-config 详情启用匹配MAC地址的网络接口。

准备需要的脚本, 在代码仓库根目录下的 scripts 中:

https://github.com/cloud-hypervisor/cloud-hypervisor/tree/v48.0/scripts

后面还要使用到 test_data/cloud-init/ubuntu/ 目录下的文件, 因此最好是把整个代码仓库下载下来,方便后面使用

```bash
mkdir -p ~/work/code/cloud-hypervisor/
cd ~/work/code/cloud-hypervisor/

git clone https://github.com/cloud-hypervisor/cloud-hypervisor.git
cd cloud-hypervisor
git checkout v48.0
```

执行:

```bash
~/work/code/cloud-hypervisor/cloud-hypervisor/scripts/create-cloud-init.sh
```

输出如下:

```bash
+ rm -f /tmp/ubuntu-cloudinit.img
+ mkdosfs -n CIDATA -C /tmp/ubuntu-cloudinit.img 8192
mkfs.fat 4.2 (2021-01-31)
+ mcopy -oi /tmp/ubuntu-cloudinit.img -s test_data/cloud-init/ubuntu/local/user-data ::
+ mcopy -oi /tmp/ubuntu-cloudinit.img -s test_data/cloud-init/ubuntu/local/meta-data ::
+ mcopy -oi /tmp/ubuntu-cloudinit.img -s test_data/cloud-init/ubuntu/local/network-config ::
```

这会生成一个 ubuntu-cloudinit.img 文件,大小为 8M:

```bash
$ ls -lh /tmp/ubuntu-cloudinit.img
-rw-rw-r-- 1 sky sky 8.0M Oct 16 17:12 /tmp/ubuntu-cloudinit.img
```

### 启动

注意启动命令中的路径要用绝对路径,不能用 '~/' 之类,否则会报错无法找到文件.

```bash
cd ~/work/code/cloud-hypervisor/cloud-hypervisor/

./target/x86_64-unknown-linux-musl/release/cloud-hypervisor \
	--kernel /home/sky/work/soft/cloudhypervisor/fireware/hypervisor-fw \
	--disk path=/home/sky/work/soft/cloudhypervisor/fireware/focal-server-cloudimg-amd64.raw path=/tmp/ubuntu-cloudinit.img \
	--cpus boot=4 \
	--memory size=1024M \
	--net "tap=,mac=,ip=,mask="
```

输出为:

```bash
cloud-hypervisor: 481.142000µs: <main> WARN:vmm/src/vm_config.rs:356 -- Deprecation warning: No IP address provided. A default IP address is assigned. This behavior will be deprecated soon.
cloud-hypervisor: 506.863000µs: <main> WARN:vmm/src/vm_config.rs:363 -- Deprecation warning: No network mask provided. A default network mask is assigned. This behavior will be deprecated soon.

Ubuntu 20.04.6 LTS cloud hvc0

cloud login:
```

输入前面设置的用户/密码 (cloud / cloud123), 登录进去, 如果要退出,执行 `sudo shutdown now` 命令关闭启动的 microVM 即可.

此时因为没有配置好网络, 所以登录进去 microVM 后会发现没有网络可用.

### 配置microVM的网络

> 没成功,等下继续.

首先要在主机上创建并配置 TAP 接口:

```bash
sudo ip tuntap add name vm-tap0 mode tap user $(whoami) multi_queue
sudo ip link set vm-tap0 up
sudo ip addr add 192.168.1.127/24 dev vm-tap0
```

此时在主机上看到的 vm-tap0 设备为:

```bash
$ ip addr
......

7: vm-tap0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether 3e:7b:2e:ca:9a:3e brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.127/24 scope global vm-tap0
       valid_lft forever preferred_lft forever
```

注意：上面的 ip 命令在重启后会失效。若需持久化配置，可将其写入主机的网络配置文件（如 /etc/network/interfaces 或使用 systemd-networkd/NetworkManager)。


```bash
/home/sky/work/code/cloud-hypervisor/cloud-hypervisor/target/x86_64-unknown-linux-musl/release/cloud-hypervisor \
	--kernel /home/sky/work/soft/cloudhypervisor/fireware/hypervisor-fw \
	--disk path=/home/sky/work/soft/cloudhypervisor/fireware/focal-server-cloudimg-amd64.raw path=/tmp/ubuntu-cloudinit.img \
	--cpus boot=4 \
	--memory size=1024M \
	--net "tap=vm-tap0,mac=12:34:56:78:90:22,ip=192.168.1.129,mask=255.255.255.0"
```
