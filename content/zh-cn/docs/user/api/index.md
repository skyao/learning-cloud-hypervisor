---
title: "API"
linkTitle: "API"
weight: 10
date: 2025-11-15
description: >
  cloud hypervisor HTTP API
---

https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/api.md

------------

cloud hypervisor API 由两个不同的接口组成：

1. 外部 API / External API 

   这是面向用户的 API。用户和运维可以通过多种选项（包括 REST API、命令行界面（CLI）或基于 D-Bus 的 API）控制和管理Cloud Hypervisor，其中基于 D-Bus 的 API 默认情况下不会被编译到 Cloud Hypervisor 中。

2. 内部 API / internal API

   内部 API 基于 rust 的多生产者、单消费者（MPSC）模块。此 API 由云虚拟机线程内部使用，用于线程之间的通信。

本文档的目标是全面描述 Cloud Hypervisor API，并概述内部和外部 API 在架构上的关系。

## 外部 API

Cloud Hypervisor 管理器 REST API 触发特定于虚拟机(VM)和虚拟机管理器(VMM)的操作，因此它被设计为一系列 RPC 风格的静态方法。

该 API 符合 OpenAPI 3.0 标准。有关 API 负载和响应的更多详细信息，请参阅 Cloud Hypervisor OpenAPI 文档。

### 位置和可用性

如果启用了 REST API，则 Cloud Hypervisor 二进制文件启动后即可通过 UNIX 套接字（如 Cloud Hypervisor 选项 --api-socket path=... 中指定的）或 --api-socket fd=... 的文件描述符使用。

```bash
$ cloud-hypervisor --api-socket path=/tmp/cloud-hypervisor.sock
```

### REST API 端点

#### 虚拟机管理器（VMM）操作

| 操作                   | 端点            | 请求体 | 响应体                     | 前提条件     |
| ---------------------- | --------------- | ------ | -------------------------- | ------------ |
| 检查 REST API 的可用性 | `/vmm.ping`     | N/A    | `/schemas/VmmPingResponse` | N/A          |
| 关闭 VMM               | `/vmm.shutdown` | N/A    | N/A                        | VMM 正在运行 |



##### 虚拟机（VM）操作

| 操作                          | 端点                    | 请求体                          | 响应体                   | 前提条件                                 |
| ----------------------------- | ----------------------- | ------------------------------- | ------------------------ | ---------------------------------------- |
| 创建虚拟机                    | `/vm.create`            | `/schemas/VmConfig`             | N/A                      | 虚拟机尚未创建                           |
| 删除虚拟机                    | `/vm.delete`            | N/A                             | N/A                      | N/A                                      |
| 启动虚拟机                    | `/vm.boot`              | N/A                             | N/A                      | 虚拟机已创建但未启动                     |
| 关闭虚拟机                    | `/vm.shutdown`          | N/A                             | N/A                      | 虚拟机已启动                             |
| 重启虚拟机                    | `/vm.reboot`            | N/A                             | N/A                      | 虚拟机已启动                             |
| 触发虚拟机电源按钮            | `/vm.power-button`      | N/A                             | N/A                      | 虚拟机已启动                             |
| 暂停虚拟机                    | `/vm.pause`             | N/A                             | N/A                      | 虚拟机已启动                             |
| 恢复虚拟机                    | `/vm.resume`            | N/A                             | N/A                      | 虚拟机已暂停                             |
| 对虚拟机进行快照任务          | `/vm.snapshot`          | `/schemas/VmSnapshotConfig`     | N/A                      | 虚拟机已暂停                             |
| 对虚拟机进行 coredump *       | `/vm.coredump`          | `/schemas/VmCoredumpData`       | N/A                      | 虚拟机已暂停                             |
| 从快照恢复虚拟机              | `/vm.restore`           | `/schemas/RestoreConfig`        | N/A                      | 虚拟机已创建但未启动                     |
| 向/从虚拟机添加/移除 CPU      | `/vm.resize`            | `/schemas/VmResize`             | N/A                      | 虚拟机已启动                             |
| 向虚拟机添加/删除内存         | `/vm.resize`            | `/schemas/VmResize`             | N/A                      | 虚拟机已启动                             |
| 向 zene 添加/删除内存         | `/vm.resize-zone`       | `/schemas/VmResizeZone`         | N/A                      | 虚拟机已启动                             |
| 导出(Dump)虚拟机信息          | `/vm.info`              | N/A                             | `/schemas/VmInfo`        | 虚拟机已创建                             |
| 向虚拟机添加 VFIO PCI 设备    | `/vm.add-device`        | `/schemas/VmAddDevice`          | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 向虚拟机添加磁盘设备          | `/vm.add-disk`          | `/schemas/DiskConfig`           | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 向虚拟机添加文件系统设备      | `/vm.add-fs`            | `/schemas/FsConfig`             | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 向虚拟机添加持久内存设备      | `/vm.add-pmem`          | `/schemas/PmemConfig`           | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 向虚拟机添加网络设备          | `/vm.add-net`           | `/schemas/NetConfig`            | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 向虚拟机添加用户空间 PCI 设备 | `/vm.add-user-device`   | `/schemas/VmAddUserDevice`      | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 向虚拟机添加 vdpa 设备        | `/vm.add-vdpa`          | `/schemas/VdpaConfig`           | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 为虚拟机添加 vsock 设备       | `/vm.add-vsock`         | `/schemas/VsockConfig`          | `/schemas/PciDeviceInfo` | 虚拟机已启动                             |
| 从虚拟机中移除设备            | `/vm.remove-device`     | `/schemas/VmRemoveDevice`       | N/A                      | 虚拟机已启动                             |
| 转储(Dump)虚拟机计数器        | `/vm.counters`          | N/A                             | `/schemas/VmCounters`    | 虚拟机已启动                             |
| 注入 NMI                      | `/vm.nmi`               | N/A                             | N/A                      | 虚拟机已启动                             |
| 准备接收迁移                  | `/vm.receive-migration` | `/schemas/ReceiveMigrationData` | N/A                      | N/A                                      |
| 开始向目标发送迁移            | `/vm.send-migration`    | `/schemas/SendMigrationData`    | N/A                      | 虚拟机已启动，并启用了（共享内存或大页） |



- `vmcoredump` 操作仅适用于 `x86_64` 架构，并且只有在 `guest_debug` 功能启用时才能执行。如果没有此功能，相应的 REST API 或 D-Bus API 端点将不可用。

### REST API 示例

注意要先确保 cloud-hypervisor 有 net_admin 特权: 

```bash
# 确保 cloud-hypervisor 有 net_admin 特权
# which cloud-hypervisor
# /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor
sudo setcap cap_net_admin+ep /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor
```

不然涉及到网络操作时会报错 Operation not permitted, 例如:

```bash
$ curl --unix-socket /tmp/cloud-hypervisor.sock -i -X PUT 'http://localhost/api/v1/vm.boot'
HTTP/1.1 500 
Server: Cloud Hypervisor API
Connection: keep-alive
Content-Type: application/json
Content-Length: 226

["Error from API","The VM could not boot","Error from device manager","Cannot create virtio-net device","Failed to open taps","Open tap device failed","Unable to configure tap interface","Operation not permitted (os error 1)"]% 
```

对于以下示例集，我们假设 Cloud Hypervisor 已启动，并且 REST API  在  `/tmp/cloud-hypervisor.sock` 可用:

```bash
cloud-hypervisor --api-socket /tmp/cloud-hypervisor.sock
```

备注: 这里启动的是 cloud-hypervisor VMM 进程, 后面可以通过 /tmp/cloud-hypervisor.sock 进行 unix domain socket 通讯.

可以开启两个终端, 一个执行上面的命令启动 cloud-hypervisor VMM 进程, 稍后 vm 启动之后可以登录和操作这个新启动的 vm. 另一个终端上执行各种操作命令. 

#### 创建虚拟机

我们想要创建一个具有以下特征的虚拟机：

- 4 vCPUs 4 个 vCPU
- 1 GB 内存
- 1 个基于 virtio 的网络接口
- 从位于 `/opt/clh/kernel/vmlinux-virtio-fs-virtio-iommu` 的定制 5.6.0-rc4 Linux 内核直接内核启动
- 使用一个位于 `/opt/clh/images/focal-server-cloudimg-amd64.raw` 的 Ubuntu 镜像作为其 rootfs

在另外一个终端中执行: 

```bash
curl --unix-socket /tmp/cloud-hypervisor.sock -i \
     -X PUT 'http://localhost/api/v1/vm.create'  \
     -H 'Accept: application/json'               \
     -H 'Content-Type: application/json'         \
     -d '{
         "cpus":{"boot_vcpus": 4, "max_vcpus": 4},
         "payload":{"kernel":"/home/sky/work/code/cloud-hypervisor/linux-cloud-hypervisor/arch/x86/boot/compressed/vmlinux.bin", "cmdline":"console=ttyS0 console=hvc0 root=/dev/vda1 rw"},
         "disks":[{"path":"/home/sky/work/soft/cloudhypervisor/fireware/focal-server-cloudimg-amd64.raw"}],
         "rng":{"src":"/dev/urandom"},
         "net":[{"ip":"192.168.10.10", "mask":"255.255.255.0", "mac":"12:34:56:78:90:01"}]
         }'
```

这里重用了之前文档中构建的 linux 内核 6.12 内核和 Ubuntu image:

https://www.cloudhypervisor.org/docs/prologue/quick-start/#custom-kernel-and-disk-image

如果成功,则返回 response:

```properties
HTTP/1.1 204 
Server: Cloud Hypervisor API
Connection: keep-alive
```

#### 启动虚拟机

创建虚拟机后，我们可以启动它：

```bash
curl --unix-socket /tmp/cloud-hypervisor.sock -i -X PUT 'http://localhost/api/v1/vm.boot'
```

如果成功,则返回 response (和前面一样):

```properties
HTTP/1.1 204 
Server: Cloud Hypervisor API
Connection: keep-alive
```

此时启动 cloud hypervisor vmm 的终端会显示 vm 创建并开始启动 ubuntu 虚拟机, 出现如下内容:

```bash
cloud-hypervisor:  24.616571s: <vcpu0> WARN:devices/src/legacy/debug_port.rs:77 -- [Debug I/O port: Kernel code 0x40] 0.018741 seconds
[    0.092310] Non-volatile memory driver v1.3
[    0.092879] Hangcheck: starting hangcheck timer 0.9.1 (tick is 180 seconds, margin is 60 seconds).
[    0.094172] brd: module loaded
[    0.094994] loop: module loaded
[    0.095309] virtio_blk virtio1: 1/0/0 default/read/poll queues
[    0.095785] virtio_blk virtio1: [vda] 4612096 512-byte logical blocks (2.36 GB/2.20 GiB)
[    0.097155]  vda: vda1 vda14 vda15
[    0.097392] zram: Added device: zram0
[    0.097816] null_blk: disk nullb0 created
[    0.098028] null_blk: module loaded
[    0.098308] tun: Universal TUN/TAP device driver, 1.6
[    0.099701] VFIO - User Level meta-driver version: 0.3
[    0.100014] mousedev: PS/2 mouse device common for all mice
[    0.100549] device-mapper: ioctl: 4.48.0-ioctl (2023-03-01) initialised: dm-devel@lists.linux.dev
[    0.101084] device-mapper: multipath round-robin: version 1.2.0 loaded
[    0.101401] intel_pstate: CPU model not supported
[    0.101686] hid: raw HID events driver (C) Jiri Kosina
[    0.102023] Initializing XFRM netlink socket
[    0.102242] NET: Registered PF_PACKET protocol family
[    0.102502] NET: Registered PF_KEY protocol family
[    0.102875] NET: Registered PF_VSOCK protocol family
[    0.103105] IPI shorthand broadcast: enabled
[    0.103400] AES CTR mode by8 optimization enabled
[    0.105347] sched_clock: Marking stable (72001308, 32049962)->(118964763, -14913493)
[    0.105809] registered taskstats version 1
[    0.106657] Demotion targets for Node 0: null
[    0.106949] Key type .fscrypt registered
[    0.107137] Key type fscrypt-provisioning registered
[    0.108865] clk: Disabling unused clocks
[    0.116017] EXT4-fs (vda1): mounted filesystem 0101c7fc-3b92-405d-87d1-ab167be30b9d r/w with ordered data mode. Quota mode: disabled.
[    0.117105] VFS: Mounted root (ext4 filesystem) on device 254:1.
[    0.117843] devtmpfs: mounted
[    0.118491] Freeing unused decrypted memory: 2028K
[    0.119130] Freeing unused kernel image (initmem) memory: 2820K
[    0.119570] Write protecting the kernel read-only data: 18432k
[    0.120403] Freeing unused kernel image (rodata/data gap) memory: 2044K
cloud-hypervisor:  24.756056s: <vcpu2> WARN:devices/src/legacy/debug_port.rs:77 -- [Debug I/O port: Kernel code 0x41] 0.158220 seconds
[    0.120848] Run /sbin/init as init process
[    0.154021] systemd[1]: systemd 245.4-4ubuntu3.24 running in system mode. (+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=hybrid)
[    0.155677] systemd[1]: Detected virtualization kvm.
[    0.156071] systemd[1]: Detected architecture x86-64.

Welcome to Ubuntu 20.04.6 LTS!

[    0.202215] systemd[1]: Set hostname to <cloud>.
[    0.292433] systemd[1]: Configuration file /run/systemd/system/netplan-ovs-cleanup.service is marked world-inaccessible. This has no effect as configuration data is accessible via APIs without restrictions. Proceeding anyway.
[    0.297285] systemd[1]: /lib/systemd/system/snapd.service:23: Unknown key name 'RestartMode' in section 'Service', ignoring.
[    0.314391] systemd[1]: Created slice system-modprobe.slice.
[  OK  ] Created slice system-modprobe.slice.
[    0.314830] systemd[1]: Created slice system-serial\x2dgetty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[    0.315183] systemd[1]: Created slice system-systemd\x2dfsck.slice.
[  OK  ] Created slice system-systemd\x2dfsck.slice.
[    0.315458] systemd[1]: Created slice User and Session Slice.
[  OK  ] Created slice User and Session Slice.
[    0.315699] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[  OK  ] Started Forward Password Requests to Wall Directory Watch.
[    0.316017] systemd[1]: Set up automount Arbitrary Executable File Formats File System Automount Point.
[  OK  ] Set up automount Arbitrary Executa…le Formats File System Automount Point.
[    0.316261] systemd[1]: Reached target User and Group Name Lookups.
[  OK  ] Reached target User and Group Name Lookups.
[    0.316436] systemd[1]: Reached target Slices.
[  OK  ] Reached target Slices.
[    0.316564] systemd[1]: Reached target Mounting snaps.
[  OK  ] Reached target Mounting snaps.
[    0.316700] systemd[1]: Reached target Swap.
[  OK  ] Reached target Swap.
[    0.316858] systemd[1]: Listening on Device-mapper event daemon FIFOs.
[  OK  ] Listening on Device-mapper event daemon FIFOs.
[    0.317077] systemd[1]: Listening on LVM2 poll daemon socket.
[  OK  ] Listening on LVM2 poll daemon socket.
[    0.317244] systemd[1]: Listening on multipathd control socket.
[  OK  ] Listening on multipathd control socket.
[    0.317440] systemd[1]: Listening on Syslog Socket.
[  OK  ] Listening on Syslog Socket.
[    0.317598] systemd[1]: Listening on fsck to fsckd communication Socket.
[  OK  ] Listening on fsck to fsckd communication Socket.
[    0.317781] systemd[1]: Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[    0.318037] systemd[1]: Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Audit Socket.
[    0.318221] systemd[1]: Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket (/dev/log).
[    0.318433] systemd[1]: Listening on Journal Socket.
[  OK  ] Listening on Journal Socket.
[    0.318616] systemd[1]: Listening on Network Service Netlink Socket.
[  OK  ] Listening on Network Service Netlink Socket.
[    0.318803] systemd[1]: Listening on udev Control Socket.
[  OK  ] Listening on udev Control Socket.
[    0.318952] systemd[1]: Listening on udev Kernel Socket.
[  OK  ] Listening on udev Kernel Socket.
[    0.348268] systemd[1]: Mounting Huge Pages File System...
         Mounting Huge Pages File System...
[    0.349953] systemd[1]: Mounting POSIX Message Queue File System...
         Mounting POSIX Message Queue File System...
[    0.351493] systemd[1]: Mounting Kernel Debug File System...
         Mounting Kernel Debug File System...
[    0.352253] systemd[1]: Condition check resulted in Kernel Trace File System being skipped.
[    0.353759] systemd[1]: Starting Journal Service...
         Starting Journal Service...
[    0.355227] systemd[1]: Starting Set the console keyboard layout...
         Starting Set the console keyboard layout...
[    0.355804] systemd[1]: Condition check resulted in Create list of static device nodes for the current kernel being skipped.
[    0.356820] systemd[1]: Starting Monitoring of LVM2 mirrors, snapshots etc. using dmeventd or progress polling...
         Starting Monitoring of LVM2 mirror…. using dmeventd or progress polling...
[    0.358029] systemd[1]: Starting Load Kernel Module chromeos_pstore...
         Starting Load Kernel Module chromeos_pstore...
[    0.358902] systemd[1]: Starting Load Kernel Module drm...
         Starting Load Kernel Module drm...
[    0.359573] systemd[1]: Starting Load Kernel Module efi_pstore...
         Starting Load Kernel Module efi_pstore...
[    0.360210] systemd[1]: Starting Load Kernel Module pstore_blk...
         Starting Load Kernel Module pstore_blk...
[    0.360821] systemd[1]: Starting Load Kernel Module pstore_zone...
         Starting Load Kernel Module pstore_zone...
[    0.361651] systemd[1]: Starting Load Kernel Module ramoops...
         Starting Load Kernel Module ramoops...
[    0.361935] systemd[1]: Condition check resulted in OpenVSwitch configuration for cleanup being skipped.
[    0.362638] systemd[1]: Condition check resulted in Set Up Additional Binary Formats being skipped.
[    0.362956] systemd[1]: Condition check resulted in File System Check on Root Device being skipped.
[    0.363610] systemd[1]: Starting Load Kernel Modules...
         Starting Load Kernel Modules...
[    0.364029] systemd[1]: Starting Remount Root and Kernel File Systems...
         Starting Remount Root and Kernel File Systems...
[    0.364482] systemd[1]: Starting udev Coldplug all Devices...
         Starting udev Coldplug all Devices...
[    0.365174] systemd[1]: Starting Uncomplicated firewall...
         Starting Uncomplicated firewall...
[    0.365928] systemd[1]: Mounted Huge Pages File System.
[  OK  ] Mounted Huge Pages File System.
[    0.366163] systemd[1]: Mounted POSIX Message Queue File System.
[  OK  ] Mounted POSIX Message Queue File System.
[    0.366362] systemd[1]: Mounted Kernel Debug File System.
[  OK  ] Mounted Kernel Debug File System.
[    0.366557] systemd[1]: modprobe@chromeos_pstore.service: Succeeded.
[    0.366762] systemd[1]: Finished Load Kernel Module chromeos_pstore.
[  OK  ] Finished Load Kernel Module chromeos_pstore.
[    0.367005] systemd[1]: modprobe@drm.service: Succeeded.
[    0.367183] systemd[1]: Finished Load Kernel Module drm.
[  OK  ] Finished Load Kernel Module drm.
[    0.367415] systemd[1]: modprobe@efi_pstore.service: Succeeded.
[    0.367625] systemd[1]: Finished Load Kernel Module efi_pstore.
[  OK  ] Finished Load Kernel Module efi_pstore.
[    0.367882] systemd[1]: modprobe@pstore_blk.service: Succeeded.
[    0.368113] systemd[1]: Finished Load Kernel Module pstore_blk.
[  OK  ] Finished Load Kernel Module pstore_blk.
[    0.368360] systemd[1]: modprobe@pstore_zone.service: Succeeded.
[    0.368563] systemd[1]: Finished Load Kernel Module pstore_zone.
[  OK  ] Finished Load Kernel Module pstore_zone.
[    0.368801] systemd[1]: modprobe@ramoops.service: Succeeded.
[    0.368991] systemd[1]: Finished Load Kernel Module ramoops.
[  OK  ] Finished Load Kernel Module ramoops.
[    0.369206] systemd[1]: systemd-modules-load.service: Main process exited, code=exited, status=1/FAILURE
[    0.369449] systemd[1]: systemd-modules-load.service: Failed with result 'exit-code'.
[    0.369681] systemd[1]: Failed to start Load Kernel Modules.
[FAILED] Failed to start Load Kernel Modules.
See 'systemctl status systemd-modules-load.service' for details.
[    0.370099] systemd[1]: Finished Uncomplicated firewall.
[  OK  ] Finished Uncomplicated firewall.
[    0.370580] systemd[1]: Mounting FUSE Control File System...
         Mounting FUSE Control File System...
[    0.370969] systemd[1]: Mounting Kernel Configuration File System...
         Mounting Kernel Configuration File System...
[    0.371325] systemd[1]: Starting Apply Kernel Variables...
         Starting Apply Kernel Variables...
[    0.372044] systemd[1]: Mounted FUSE Control File System.
[  OK  ] Mounted FUSE Control File System.
[    0.372315] systemd[1]: Mounted Kernel Configuration File System.
[  OK  ] Mounted Kernel Configuration File System.
[    0.374224] systemd[1]: Started Journal Service.
[  OK  ] Started Journal Service.
[  OK  ] Finished Apply Kernel Variables.
[  OK  ] Finished udev Coldplug all Devices.
         Starting udev Wait for Complete Device Initialization...
[  OK  ] Finished Remount Root and Kernel File Systems.
         Starting Flush Journal to Persistent Storage...
         Starting Load/Save Random Seed...
         Starting Create System Users...
[  OK  ] Finished Set the console keyboard layout.
[  OK  ] Finished Create System Users.
         Starting Create Static Device Nodes in /dev...
[  OK  ] Finished Create Static Device Nodes in /dev.
         Starting udev Kernel Device Manager...
[  OK  ] Finished Load/Save Random Seed.
[  OK  ] Finished Flush Journal to Persistent Storage.
[  OK  ] Started udev Kernel Device Manager.
[  OK  ] Started Dispatch Password Requests to Console Directory Watch.
[  OK  ] Reached target Local Encrypted Volumes.
         Starting Network Service...
[  OK  ] Started Network Service.
         Starting Wait for Network to be Configured...
[  OK  ] Found device /dev/ttyS0.
[  OK  ] Found device /dev/hvc0.
         Mounting Arbitrary Executable File Formats File System...
[  OK  ] Mounted Arbitrary Executable File Formats File System.
[  OK  ] Found device /dev/disk/by-label/UEFI.
[  OK  ] Finished Wait for Network to be Configured.
[  OK  ] Finished udev Wait for Complete Device Initialization.
         Starting Device-Mapper Multipath Device Controller...
[  OK  ] Finished Monitoring of LVM… dmeventd or progress polling.
[  OK  ] Started Device-Mapper Multipath Device Controller.
[  OK  ] Reached target Local File Systems (Pre).
         Mounting Mount unit for core20, revision 2599...
         Mounting Mount unit for lxd, revision 32662...
         Mounting Mount unit for snapd, revision 24671...
         Starting File System Check on /dev/disk/by-label/UEFI...
[  OK  ] Started File System Check Daemon to report status.
[  OK  ] Finished File System Check on /dev/disk/by-label/UEFI.
         Mounting /boot/efi...
[FAILED] Failed to mount Mount unit for core20, revision 2599.
See 'systemctl status snap-core20-2599.mount' for details.
[  OK  ] Mounted /boot/efi.
[FAILED] Failed to mount Mount unit for lxd, revision 32662.
See 'systemctl status snap-lxd-32662.mount' for details.
[DEPEND] Dependency failed for Serv…snap application lxd.activate.
[DEPEND] Dependency failed for Sock…r snap application lxd.daemon.
[FAILED] Failed to mount Mount unit for snapd, revision 24671.
See 'systemctl status snap-snapd-24671.mount' for details.
[  OK  ] Reached target Mounted snaps.
[  OK  ] Reached target Local File Systems.
         Starting Set console font and keymap...
         Starting Create final runt…dir for shutdown pivot root...
         Starting Tell Plymouth To Write Out Runtime Data...
         Starting Create Volatile Files and Directories...
[  OK  ] Finished Create final runtime dir for shutdown pivot root.
[  OK  ] Finished Set console font and keymap.
[  OK  ] Finished Create Volatile Files and Directories.
         Starting Network Name Resolution...
         Starting Network Time Synchronization...
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Finished Tell Plymouth To Write Out Runtime Data.
[  OK  ] Finished Update UTMP about System Boot/Shutdown.
[  OK  ] Started Network Time Synchronization.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target Paths.
[  OK  ] Reached target System Time Set.
[  OK  ] Reached target System Time Synchronized.
[  OK  ] Started Daily apt download activities.
[  OK  ] Started Daily apt upgrade and clean activities.
[  OK  ] Started Periodic ext4 Online Metadata Check for All Filesystems.
[  OK  ] Started Discard unused blocks once a week.
[  OK  ] Started Refresh fwupd metadata regularly.
[  OK  ] Started Daily rotation of log files.
[  OK  ] Started Daily man-db regeneration.
[  OK  ] Started Message of the Day.
[  OK  ] Reached target Timers.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Listening on Open-iSCSI iscsid Socket.
         Starting Socket activation for snappy daemon.
[  OK  ] Listening on UUID daemon activation socket.
[  OK  ] Listening on Socket activation for snappy daemon.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
         Starting Accounts Service...
[  OK  ] Started D-Bus System Message Bus.
[  OK  ] Started Save initial kernel messages after boot.
         Starting Remove Stale Online ext4 Metadata Check Snapshots...
         Starting Record successful boot for GRUB...
[  OK  ] Started irqbalance daemon.
         Starting Dispatcher daemon for systemd-networkd...
         Starting Authorization Manager...
         Starting System Logging Service...
[  OK  ] Reached target Login Prompts (Pre).
         Starting Wait until snapd is fully seeded...
         Starting Snap Daemon...
         Starting Login Service...
         Starting Disk Manager...
         Starting Rotate log files...
         Starting Daily man-db regeneration...
[  OK  ] Started Network Name Resolution.
[  OK  ] Reached target Network.
[  OK  ] Reached target Network is Online.
[  OK  ] Reached target Host and Network Name Lookups.
[  OK  ] Reached target Remote File Systems (Pre).
[  OK  ] Reached target Remote File Systems.
         Starting LSB: automatic crash report generation...
         Starting Deferred execution scheduler...
         Starting Availability of block devices...
[  OK  ] Started Regular background program processing daemon.
         Starting Pollinate to seed…udo random number generator...
         Starting Permit User Sessions...
[  OK  ] Finished Remove Stale Onli…ext4 Metadata Check Snapshots.
[  OK  ] Finished Availability of block devices.
[  OK  ] Started Deferred execution scheduler.
[  OK  ] Finished Permit User Sessions.
         Starting Hold until boot process finishes up...
         Starting Terminate Plymouth Boot Screen...
[  OK  ] Finished Hold until boot process finishes up.
[  OK  ] Started Serial Getty on hvc0.
[  OK  ] Started Serial Getty on ttyS0.
         Starting Set console scheme...
[  OK  ] Started System Logging Service.
[  OK  ] Finished Set console scheme.
[  OK  ] Created slice system-getty.slice.
[  OK  ] Started Getty on tty1.
[  OK  ] Reached target Login Prompts.
[  OK  ] Finished Record successful boot for GRUB.
         Starting GRUB failed boot detection...
[  OK  ] Finished Terminate Plymouth Boot Screen.
[  OK  ] Started Login Service.
[  OK  ] Started Unattended Upgrades Shutdown.
[  OK  ] Started LSB: automatic crash report generation.
[  OK  ] Started Authorization Manager.
         Starting Modem Manager...
[  OK  ] Finished GRUB failed boot detection.
[  OK  ] Started Accounts Service.
[  OK  ] Started Disk Manager.
[  OK  ] Finished Rotate log files.
[  OK  ] Started Modem Manager.
[  OK  ] Finished Pollinate to seed the pseudo random number generator.
         Starting OpenBSD Secure Shell server...
[  OK  ] Started Dispatcher daemon for systemd-networkd.
[  OK  ] Started OpenBSD Secure Shell server.
[  OK  ] Started Snap Daemon.
         Starting Time & Date Service...
[  OK  ] Started Time & Date Service.
[  OK  ] Finished Wait until snapd is fully seeded.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Finished Update UTMP about System Runlevel Changes.
[  OK  ] Finished Daily man-db regeneration.

Ubuntu 20.04.6 LTS cloud hvc0

cloud login: 
```

用默认帐号 cloud/cloud123 登录之后, 可以看到一些系统信息:

```bash
$ uname -a
Linux cloud 6.12.8+ #1 SMP Tue Nov 18 15:50:32 CST 2025 x86_64 x86_64 x86_64 GNU/Linux
```

这是 ubuntu 20.04 跑在了 linux 6.12 内核上. 查看网络信息: 

```bash
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 12:34:56:78:90:01 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
```

网卡创建了, ens3 的 mac 地址 `12:34:56:78:90:01` 是之前创建 microvm 时指定的 mac 地址. 

但这里网络没能生效, `"ip":"192.168.10.10"` 没能设置成功, 稍后再看. 

pci 设备非常的少: 

```bash
$ lspci
00:00.0 Host bridge: Intel Corporation Device 0d57
00:01.0 Unassigned class [ffff]: Red Hat, Inc. Virtio console (rev 01)
00:02.0 Mass storage controller: Red Hat, Inc. Virtio block device (rev 01)
00:03.0 Ethernet controller: Red Hat, Inc. Virtio network device (rev 01)
00:04.0 Unassigned class [ffff]: Red Hat, Inc. Virtio RNG (rev 01)
```

一块 Virtio network device 网卡, 一个 Virtio block device 的 mass 存储.

查看内存信息:

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:          473Mi        94Mi       208Mi       0.0Ki       171Mi       357Mi
Swap:            0B          0B          0B
```

查看 cpu 信息:

```bash
lscpu
Architecture:                         x86_64
CPU op-mode(s):                       32-bit, 64-bit
Byte Order:                           Little Endian
Address sizes:                        46 bits physical, 48 bits virtual
CPU(s):                               4
On-line CPU(s) list:                  0-3
Thread(s) per core:                   1
Core(s) per socket:                   4
Socket(s):                            1
NUMA node(s):                         1
Vendor ID:                            GenuineIntel
CPU family:                           6
Model:                                186
Model name:                           Genuine Intel(R) 0000
Stepping:                             2
CPU MHz:                              2611.200
BogoMIPS:                             5222.40
Virtualization:                       VT-x
Hypervisor vendor:                    KVM
Virtualization type:                  full
L1d cache:                            96 KiB
L1i cache:                            64 KiB
L2 cache:                             1.3 MiB
L3 cache:                             24 MiB
NUMA node0 CPU(s):                    0-3
Vulnerability Gather data sampling:   Not affected
Vulnerability Itlb multihit:          Not affected
Vulnerability L1tf:                   Not affected
Vulnerability Mds:                    Not affected
Vulnerability Meltdown:               Not affected
Vulnerability Mmio stale data:        Not affected
Vulnerability Reg file data sampling: Mitigation; Clear Register File
Vulnerability Retbleed:               Not affected
Vulnerability Spec rstack overflow:   Not affected
Vulnerability Spec store bypass:      Mitigation; Speculative Store Bypass disabled via prctl
Vulnerability Spectre v1:             Mitigation; usercopy/swapgs barriers and __user pointer san
                                      itization
Vulnerability Spectre v2:             Vulnerable: eIBRS with unprivileged eBPF
Vulnerability Srbds:                  Not affected
Vulnerability Tsx async abort:        Not affected
Flags:                                fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cm
                                      ov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdp
                                      e1gb rdtscp lm constant_tsc rep_good nopl xtopology nonstop
                                      _tsc cpuid tsc_known_freq pni pclmulqdq vmx ssse3 fma cx16 
                                      sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xs
                                      ave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cp
                                      uid_fault ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow fle
                                      xpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 sme
                                      p bmi2 erms invpcid rdseed adx smap clflushopt clwb sha_ni 
                                      xsaveopt xsavec xgetbv1 xsaves avx_vnni arat vnmi umip pku 
                                      waitpkg gfni vaes vpclmulqdq rdpid movdiri movdir64b fsrm m
                                      d_clear serialize flush_l1d arch_capabilities
```

因为创建 vm 时的 cpu 参数设置的是 `"cpus":{"boot_vcpus": 4, "max_vcpus": 4},` 设置了 4 个 cpu. 目前 4 个 cpu 都是 online 状态:

```bash
CPU(s):                               4
On-line CPU(s) list:                  0-3
Thread(s) per core:                   1
Core(s) per socket:                   4
```

##### dump 虚拟机信息

我们可以获取任何虚拟机创建后的信息：

```bash
curl --unix-socket /tmp/cloud-hypervisor.sock -i \
     -X GET 'http://localhost/api/v1/vm.info' \
     -H 'Accept: application/json'
```

返回应答:

```bash
HTTP/1.1 200 
Server: Cloud Hypervisor API
Connection: keep-alive
Content-Type: application/json
Content-Length: 3329
```

http body 为 json, 将格式转为可读格式后, 内容为:

```json
{
  "config": {
    "cpus": {
      "boot_vcpus": 4,
      "max_vcpus": 4,
      "topology": null,
      "kvm_hyperv": false,
      "max_phys_bits": 46,
      "affinity": null,
      "features": {
        "amx": false
      }
    },
    "memory": {
      "size": 536870912,
      "mergeable": false,
      "hotplug_method": "Acpi",
      "hotplug_size": null,
      "hotplugged_size": null,
      "shared": false,
      "hugepages": false,
      "hugepage_size": null,
      "prefault": false,
      "zones": null,
      "thp": true
    },
    "payload": {
      "firmware": null,
      "kernel": "/home/sky/work/code/cloud-hypervisor/linux-cloud-hypervisor/arch/x86/boot/compressed/vmlinux.bin",
      "cmdline": "console=ttyS0 console=hvc0 root=/dev/vda1 rw",
      "initramfs": null
    },
    "rate_limit_groups": null,
    "disks": [
      {
        "path": "/home/sky/work/soft/cloudhypervisor/fireware/focal-server-cloudimg-amd64.raw",
        "readonly": false,
        "direct": false,
        "iommu": false,
        "num_queues": 1,
        "queue_size": 128,
        "vhost_user": false,
        "vhost_socket": null,
        "rate_limit_group": null,
        "rate_limiter_config": null,
        "id": "_disk0",
        "disable_io_uring": false,
        "disable_aio": false,
        "pci_segment": 0,
        "serial": null,
        "queue_affinity": null
      }
    ],
    "net": [
      {
        "tap": null,
        "ip": "192.168.10.10",
        "mask": "255.255.255.0",
        "mac": "12:34:56:78:90:01",
        "host_mac": "12:94:98:54:3d:aa",
        "mtu": null,
        "iommu": false,
        "num_queues": 2,
        "queue_size": 256,
        "vhost_user": false,
        "vhost_socket": null,
        "vhost_mode": "Client",
        "id": "_net1",
        "fds": null,
        "rate_limiter_config": null,
        "pci_segment": 0,
        "offload_tso": true,
        "offload_ufo": true,
        "offload_csum": true
      }
    ],
    "rng": {
      "src": "/dev/urandom",
      "iommu": false
    },
    "balloon": null,
    "fs": null,
    "pmem": null,
    "serial": {
      "file": null,
      "mode": "Null",
      "iommu": false,
      "socket": null
    },
    "console": {
      "file": null,
      "mode": "Tty",
      "iommu": false,
      "socket": null
    },
    "debug_console": {
      "file": null,
      "mode": "Off",
      "iobase": 233
    },
    "devices": null,
    "user_devices": null,
    "vdpa": null,
    "vsock": null,
    "pvpanic": false,
    "iommu": false,
    "numa": null,
    "watchdog": false,
    "pci_segments": null,
    "platform": null,
    "tpm": null,
    "landlock_enable": false,
    "landlock_rules": null
  },
  "state": "Running",
  "memory_actual_size": 536870912,
  "device_tree": {
    "_virtio-pci-_net1": {
      "id": "_virtio-pci-_net1",
      "resources": [
        {
          "PciBar": {
            "index": 0,
            "base": 70364448161792,
            "size": 524288,
            "type_": "Mmio64",
            "prefetchable": false
          }
        }
      ],
      "parent": null,
      "children": ["_net1"],
      "pci_bdf": "0000:00:03.0"
    },
    "__ioapic": {
      "id": "__ioapic",
      "resources": [],
      "parent": null,
      "children": [],
      "pci_bdf": null
    },
    "_virtio-pci-__console": {
      "id": "_virtio-pci-__console",
      "resources": [
        {
          "PciBar": {
            "index": 0,
            "base": 70364448686080,
            "size": 524288,
            "type_": "Mmio64",
            "prefetchable": false
          }
        }
      ],
      "parent": null,
      "children": ["__console"],
      "pci_bdf": "0000:00:01.0"
    },
    "_disk0": {
      "id": "_disk0",
      "resources": [],
      "parent": "_virtio-pci-_disk0",
      "children": [],
      "pci_bdf": null
    },
    "_virtio-pci-__rng": {
      "id": "_virtio-pci-__rng",
      "resources": [
        {
          "PciBar": {
            "index": 0,
            "base": 70364447637504,
            "size": 524288,
            "type_": "Mmio64",
            "prefetchable": false
          }
        }
      ],
      "parent": null,
      "children": ["__rng"],
      "pci_bdf": "0000:00:04.0"
    },
    "__console": {
      "id": "__console",
      "resources": [],
      "parent": "_virtio-pci-__console",
      "children": [],
      "pci_bdf": null
    },
    "__rng": {
      "id": "__rng",
      "resources": [],
      "parent": "_virtio-pci-__rng",
      "children": [],
      "pci_bdf": null
    },
    "_net1": {
      "id": "_net1",
      "resources": [],
      "parent": "_virtio-pci-_net1",
      "children": [],
      "pci_bdf": null
    },
    "__serial": {
      "id": "__serial",
      "resources": [],
      "parent": null,
      "children": [],
      "pci_bdf": null
    },
    "_virtio-pci-_disk0": {
      "id": "_virtio-pci-_disk0",
      "resources": [
        {
          "PciBar": {
            "index": 0,
            "base": 3891789824,
            "size": 524288,
            "type_": "Mmio32",
            "prefetchable": false
          }
        }
      ],
      "parent": null,
      "children": ["_disk0"],
      "pci_bdf": "0000:00:02.0"
    }
  }
}
```

这台虚拟机的配置：4 个 vCPU、512MB 内存、一个磁盘、一个网卡、控制台和 RNG 设备。

#### 重启虚拟机

可以重启已经启动的虚拟机：

```bash
curl --unix-socket /tmp/cloud-hypervisor.sock -i -X PUT 'http://localhost/api/v1/vm.reboot'
```

ubuntu 虚拟机非常快的就重启了, 目测 1 秒上下.

#### 关闭虚拟机

启动后，我们可以通过 REST API 关闭虚拟机：

```bash
curl --unix-socket /tmp/cloud-hypervisor.sock -i -X PUT 'http://localhost/api/v1/vm.shutdown'
```

vm 退出后, 启动 cloud hypervisor vmm 的另外一个终端, 不会显示 vm 已经退出了, 只是不再相应. ctrl + c 可以退出.

### D-Bus API

 cloud hypervisor 提供了一个 D-Bus API 作为其 REST API 的替代方案。这个 D-Bus API 完全反映了 REST API 的功能，暴露了相同的端点组。由于它也消费/产生 JSON，因此可以作为一个即插即用的替代方案。

此外，D-Bus API 还以 D-Bus 信号的形式暴露了来自 `event-monitor` 的事件，用户可以订阅这些信号。更多详细信息，请参阅 D-Bus API 接口。

稍后看.

### 命令行界面

 cloud hypervisor 命令行界面（CLI）只能用于启动  cloud hypervisor  二进制文件，即一旦 VMM 或虚拟机启动运行后，无法使用该界面控制它们。

如果您想检查 VMM 或在从 CLI 启动 cloud hypervisor 后控制虚拟机，您必须使用 REST API 或 D-Bus API。

从 CLI 中，您可以：

1. 使用 CLI 选项构建虚拟机配置来创建和启动完整的虚拟机。运行 `cloud-hypervisor --help` 获取 CLI 选项的完整列表。与 D-Bus API 不同，一旦 `cloud-hypervisor` 二进制文件启动，REST API 即可用于控制和管理虚拟机。D-Bus API 不会自动启动，需要明确配置才能运行。
2. 无需传递任何虚拟机配置选项，即可启动 REST API、D-Bus API 或两者同时启动。然后可以通过调用选择的 API 方法异步创建和启动虚拟机。需要注意的是，外部 API 并不互斥；同时运行 REST 和 D-Bus API 是可能的。

### REST API、D-Bus API 和 CLI 架构关系

REST API、D-Bus API 和 CLI 都依赖于一个通用、内部的 API。

CLI 选项由 clap crate 解析，然后转换为内部 API 命令。

REST API 由使用 Firecracker 的 `micro_http` crate 的 HTTP 线程处理。与 CLI 类似，HTTP 请求最终会转换为内部 API 命令。

D-Bus API 使用 zbus crate 实现，并在其自己的线程中运行。每当它需要调用内部 API 时，会使用 blocking crate 在 zbus 的异步上下文中执行该调用。

总而言之，REST API、D-Bus API 和 CLI 本质上都是内部 API 的前端：

```bash
                                  +------------------+
                        REST API  |                  |
                       +--------->+    micro_http    +--------+
                       |          |                  |        |
                       |          +------------------+        |
                       |                                      |      +------------------------+
                       |                                      |      |                        |
+------------+         |            +----------+              |      |                        |
|            |         |  D-Bus API |          |              |      | +--------------+       |
|    User    +---------+----------->+   zbus   +--------------+------> | Internal API |       |
|            |         |            |          |              |      | +--------------+       |
+------------+         |            +----------+              |      |                        |
                       |                                      |      |                        |
                       |                                      |      +------------------------+
                       |            +----------+              |                 VMM
                       |     CLI    |          |              |
                       +----------->+   clap   +--------------+
                                    |          |
                                    +----------+
```

## 内部 API

Cloud Hypervisor 内部 API，如其名称所示，是 Cloud Hypervisor 不同线程（VMM、HTTP、D-Bus、控制循环等）之间发送命令和响应所使用的内部接口。

它基于 rust 的 Multi-Producer, Single-Consumer (MPSC)，而单一消费者（即 API 接收者）是 Cloud Hypervisor 的控制循环。

API 生产者包括处理 REST API 的 HTTP 线程、处理 D-Bus API 的 D-Bus 线程以及最初解析 CLI 的主线程。

### 目标和设计

内部 API 用于控制、管理和检查 Cloud Hypervisor VMM 及其客户机。它是通过 REST API、D-Bus API 或 CLI 接口处理外部、用户可见请求的后端。

该 API 遵循命令-响应方案，与 REST API 紧密对应。任何命令都必须得到响应。

命令是基于 MPSC 的消息，由 VMM 控制循环接收和处理。

为了让 VMM 控制循环能够响应任何内部 API 命令，它必须能够向 MPSC 发送者发送响应。为此，所有内部 API 命令的有效载荷都包含 MPSC 通道的发送端。

因此，任何内部 API 命令的发送者负责：

1. 创建 MPSC 响应通道。
2. 将响应通道的发送端作为内部 API 命令负载的一部分传递。
3. 在响应通道的接收端等待内部 API 命令的响应。

## 端到端示例

为了进一步了解外部和内部 Cloud Hypervisor API 如何协同工作，让我们查看一个完整的虚拟机创建流程，从 REST API 调用到外部用户将收到的回复：

1. 用户或运维向 Cloud Hypervisor  REST API 发送 HTTP 请求以创建虚拟机：

   ```bash
   shell
   #!/usr/bin/env bash
   
    curl --unix-socket /tmp/cloud-hypervisor.sock -i \
    	-X PUT 'http://localhost/api/v1/vm.create'  \
    	-H 'Accept: application/json'               \
    	-H 'Content-Type: application/json'         \
    	-d '{
    		"cpus":{"boot_vcpus": 4, "max_vcpus": 4},
    		"payload":{"kernel":"/opt/clh/kernel/vmlinux-virtio-fs-virtio-iommu", "cmdline":"console=ttyS0 console=hvc0 root=/dev/vda1 rw"},
    		"disks":[{"path":"/opt/clh/images/focal-server-cloudimg-amd64.raw"}],
    		"rng":{"src":"/dev/urandom"},
    		"net":[{"ip":"192.168.10.10", "mask":"255.255.255.0", "mac":"12:34:56:78:90:01"}]
    		}'
   ```

2. Cloud Hypervisor  HTTP 线程处理请求，并将 HTTP 请求的 JSON 正文反序列化为内部 `VmConfig` 结构。

3. Cloud Hypervisor HTTP 线程为内部 API 服务器创建了一个 MPSC 通道，用于发送其响应。

4. Cloud Hypervisor  HTTP 线程准备一个用于创建虚拟机的内部 API 命令。该命令的负载由反序列化的 `VmConfig` 结构和响应通道组成：

   ```rust
   VmCreate(Arc<Mutex<VmConfig>>, Sender<ApiResponse>)
   ```

5. Cloud Hypervisor HTTP 线程发送内部 API 命令，并等待响应：

   ```rust
   // Send the VM creation request.
    api_sender
        .send(ApiRequest::VmCreate(config, response_sender))
        .map_err(ApiError::RequestSend)?;
    api_evt.write(1).map_err(ApiError::EventFdWrite)?;
   
    response_receiver.recv().map_err(ApiError::ResponseRecv)??;
   ```

6. Cloud Hypervisor  控制循环接收到该命令，因为它监听内部 API MPSC 通道：

   ```rust
   // Read from the API receiver channel
   let api_request = api_receiver.recv().map_err(Error::ApiRequestRecv)?;
   ```

7. Cloud Hypervisor 控制循环将接收到的内部 API 与 `VmCreate` 负载进行匹配，并从命令负载中提取 `VmConfig` 结构和发送者。它存储 `VmConfig` 结构，并回复给发送者（（HTTP 线程）：）

   ```rust
   match api_request {
       ApiRequest::VmCreate(config, sender) => {
    	   // We only store the passed VM config.
    	   // The VM will be created when being asked to boot it.
    	   let response = if self.vm_config.is_none() {
    		   self.vm_config = Some(config);
    		   Ok(ApiResponsePayload::Empty)
    	   } else {
    		   Err(ApiError::VmAlreadyCreated)
    	   };
   
           sender.send(response).map_err(Error::ApiResponseSend)?;
       }
   ```

8. Cloud Hypervisor HTTP 线程将内部 API 命令响应作为其 `VmCreate` HTTP 处理程序的返回值。根据控制循环内部 API 响应，它生成适当的 HTTP 响应：

   ```rust
   // Call vm_create()
   match vm_create(api_notifier, api_sender, Arc::new(Mutex::new(vm_config)))
       .map_err(HttpError::VmCreate)
   {
       Ok(_) => Response::new(Version::Http11, StatusCode::NoContent),
       Err(e) => error_response(e, StatusCode::InternalServerError),
   }
   ```

9. Cloud Hypervisor  HTTP 线程将形成的 HTTP 响应发送回用户。这由 micro_http 库抽象化。

