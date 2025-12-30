---
title: "优化性能"
linkTitle: "优化性能"
weight: 30
date: 2025-10-15
description: >
  cloud hypervisor 快照优化性能
---

## 命令

### 启动

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor \
    --api-socket /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock \
    --cpus boot=1,nested=off \
    --memory size=512M \
    --kernel /home/sky/work/test/cloudhypervisor/vmlinux.bin \
    --cmdline "root=/dev/vda1 rw quiet 8250.nr_uarts=0" \
    --disk path=/home/sky/work/test/cloudhypervisor/ubuntu-cloud-image/rootfs.raw \
    --console off \
    --serial off \
    --log-file /dev/null
```

### 制作快照

```bash
rm -rf /home/sky/work/test/cloudhypervisor/snapshot-4g
mkdir -p /home/sky/work/test/cloudhypervisor/snapshot-4g
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock pause
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  snapshot file:///home/sky/work/test/cloudhypervisor/snapshot-4g
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  resume
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  shutdown
sudo fuser -k -TERM /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock
```

### 恢复虚拟机

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor \
    --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock \
    --restore source_url=file:///home/sky/work/test/cloudhypervisor/snapshot-4g
```


```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  resume
```

## 默认

执行前强制清理 page cache：

```bash
====== 开始性能测试 (Cache: false Size=4G) ======

 1. Load (Disk -> RAM)     :   1913.322 ms
 2. Resume (CPU Active)    :     16.584 ms
 --------------------------------------------------
 3. Total Recovery         :   1929.907 ms

====== 开始性能测试 (Cache: false Size=2G) ======

 1. Load (Disk -> RAM)     :    985.878 ms
 2. Resume (CPU Active)    :     14.172 ms
 --------------------------------------------------
 3. Total Recovery         :   1000.050 ms

====== 开始性能测试 (Cache: false Size=1G) ======

 1. Load (Disk -> RAM)     :    531.535 ms
 2. Resume (CPU Active)    :     21.058 ms
 --------------------------------------------------
 3. Total Recovery         :    552.594 ms

====== 开始性能测试 (Cache: false Size=512M) ======

 1. Load (Disk -> RAM)     :    304.199 ms
 2. Resume (CPU Active)    :     15.110 ms
 --------------------------------------------------
 3. Total Recovery         :    319.310 ms
```

## page cache优化

```bash
====== 开始性能测试 (Cache: true Size=4G) ======

 1. Load (Disk -> RAM)     :    623.998 ms
 2. Resume (CPU Active)    :     14.062 ms
 --------------------------------------------------
 3. Total Recovery         :    638.060 ms

====== 开始性能测试 (Cache: true Size=2G) ======

 1. Load (Disk -> RAM)     :    333.440 ms
 2. Resume (CPU Active)    :     17.197 ms
 --------------------------------------------------
 3. Total Recovery         :    350.637 ms

====== 开始性能测试 (Cache: true Size=1G) ======

 1. Load (Disk -> RAM)     :    190.534 ms
 2. Resume (CPU Active)    :     16.542 ms
 --------------------------------------------------
 3. Total Recovery         :    207.076 ms

====== 开始性能测试 (Cache: true Size=512M ======

 1. Load (Disk -> RAM)     :    105.908 ms
 2. Resume (CPU Active)    :     16.547 ms
 --------------------------------------------------
 3. Total Recovery         :    122.456 ms
```

##  HugePages 优化

启动

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor \
    --api-socket /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock \
    --cpus boot=1,nested=off \
    --memory size=1G,hugepages=on \
    --kernel /home/sky/work/test/cloudhypervisor/vmlinux.bin \
    --cmdline "root=/dev/vda1 rw quiet 8250.nr_uarts=0" \
    --disk path=/home/sky/work/test/cloudhypervisor/ubuntu-cloud-image/rootfs.raw \
    --console off \
    --serial off \
    --log-file /dev/null
```

```bash
rm -rf /home/sky/work/test/cloudhypervisor/snapshot-1g-hugepage
mkdir -p /home/sky/work/test/cloudhypervisor/snapshot-1g-hugepage
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock pause
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  snapshot file:///home/sky/work/test/cloudhypervisor/snapshot-1g-hugepage
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  resume
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  shutdown
sudo fuser -k -TERM /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock
```

不开启 page cache 基本没差别：

```bash
====== 开始性能测试 (Cache: false Size=4G) ======

 1. Load (Disk -> RAM)     :   1906.534 ms
 2. Resume (CPU Active)    :     15.309 ms
 --------------------------------------------------
 3. Total Recovery         :   1921.843 ms

====== 开始性能测试 (Cache: false Size=2G) ======

 1. Load (Disk -> RAM)     :    985.883 ms
 2. Resume (CPU Active)    :     13.876 ms
 --------------------------------------------------
 3. Total Recovery         :    999.759 ms

====== 开始性能测试 (Cache: false Size=1G) ======

 1. Load (Disk -> RAM)     :    522.341 ms
 2. Resume (CPU Active)    :     20.296 ms
 --------------------------------------------------
 3. Total Recovery         :    542.638 ms

====== 开始性能测试 (Cache: false Size=512M) ======

 1. Load (Disk -> RAM)     :    293.190 ms
 2. Resume (CPU Active)    :     22.518 ms
 --------------------------------------------------
 3. Total Recovery         :    315.708 ms
```

开启 page cache 能快一点：

```bash
====== 开始性能测试 (Cache: true Size=4G) ======

 1. Load (Disk -> RAM)     :    526.327 ms
 2. Resume (CPU Active)    :     15.826 ms
 --------------------------------------------------
 3. Total Recovery         :    542.153 ms

====== 开始性能测试 (Cache: true Size=2G) ======

 1. Load (Disk -> RAM)     :    312.838 ms
 2. Resume (CPU Active)    :     15.331 ms
 --------------------------------------------------
 3. Total Recovery         :    328.169 ms

====== 开始性能测试 (Cache: true Size=1G) ======

 1. Load (Disk -> RAM)     :    153.869 ms
 2. Resume (CPU Active)    :     16.558 ms
 --------------------------------------------------
 3. Total Recovery         :    170.427 ms

====== 开始性能测试 (Cache: true Size=512M ======

 1. Load (Disk -> RAM)     :    102.818 ms
 2. Resume (CPU Active)    :     15.310 ms
 --------------------------------------------------
 3. Total Recovery         :    118.128 ms
```
