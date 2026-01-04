---
title: "优化性能"
linkTitle: "优化性能"
weight: 30
date: 2025-10-15
description: >
  cloud hypervisor 快照优化性能
---

## 准备工作

### 准备目录和文件

继续使用之前的 `work/test/cloudhypervisor` 目录：

```bash
cd /home/sky/work/test/cloudhypervisor
```

### 启动

冷启动虚拟机，用于制作快照，注意修改内容大小设置：

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

根据前面设置的内存大小，制作快照，保存在不同的目录下，备用：

```bash
rm -rf /home/sky/work/test/cloudhypervisor/snapshot-4g
mkdir -p /home/sky/work/test/cloudhypervisor/snapshot-4g
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock pause
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  snapshot file:///home/sky/work/test/cloudhypervisor/snapshot-4g
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  resume
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  shutdown
sudo fuser -k -TERM /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock
```

重复以上操作，可以分别制作出 512M/1G/2G/4G 的快照文件，用于后面的恢复操作。

### 恢复虚拟机

执行恢复虚拟机的命令，注意选择快照文件的来源目录，其实就是选择不同的内存规格：

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/cloud-hypervisor \
    --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock \
    --restore source_url=file:///home/sky/work/test/cloudhypervisor/snapshot-4g
```

恢复之后，虚拟机是处于 pause 状态的，因此还要执行 resume 才能唤醒到正常运行状态：

```bash
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  resume
```

## 恢复性能测试

### 默认

使用脚本 test_restore_speed.sh 进行测试，执行前修改脚本中的这一行，指定要使用的不同内存规格的快照文件：

```bash
SNAPSHOT_URL="file:///home/sky/work/test/cloudhypervisor/snapshot-512m"
```

执行前强制清理 page cache：

```bash
./test_restore_speed.sh false
```

多次测试的结果汇总为：

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

 1. Load (Disk -> RAM)     :    533.121 ms
 2. Resume (CPU Active)    :     15.142 ms
 --------------------------------------------------
 3. Total Recovery         :    548.263 ms

====== 开始性能测试 (Cache: false Size=512M) ======

 1. Load (Disk -> RAM)     :    304.199 ms
 2. Resume (CPU Active)    :     15.110 ms
 --------------------------------------------------
 3. Total Recovery         :    319.310 ms
```

### page cache优化

类似的，不过执行前执行 page cache 优化：

```bash
./test_restore_speed.sh true
```

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

###  HugePages 优化

类似，但是在启动虚拟机时设置内存为 hugepages=on：

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

同样制作快照，为了和之前 hugepages=off 的快照区分，存放在不同的目录，以后缀 `-hugepage` 为标志：

```bash
rm -rf /home/sky/work/test/cloudhypervisor/snapshot-1g-hugepage
mkdir -p /home/sky/work/test/cloudhypervisor/snapshot-1g-hugepage
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock pause
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  snapshot file:///home/sky/work/test/cloudhypervisor/snapshot-1g-hugepage
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  resume
sudo /home/sky/work/soft/cloudhypervisor/bin/ch-remote --api-socket=/home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock  shutdown
sudo fuser -k -TERM /home/sky/work/test/cloudhypervisor/cloud-hypervisor.sock
```

继续测试，开启 hugepage + 不开启 page cache 基本没差别：

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

 1. Load (Disk -> RAM)     :    521.281 ms
 2. Resume (CPU Active)    :     14.697 ms
 --------------------------------------------------
 3. Total Recovery         :    535.978 ms

====== 开始性能测试 (Cache: false Size=512M) ======

 1. Load (Disk -> RAM)     :    300.310 ms
 2. Resume (CPU Active)    :     15.617 ms
 --------------------------------------------------
 3. Total Recovery         :    315.927 ms
```

开启 hugepage + 开启 page cache 能快一点： 

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

### 性能对比分析

将上面测试的几组数据汇总，对比一下：

| 内存规格 | 操作    | 默认（强制清理page cache） | 开启 page cache | 不开启 page cache + 开启 hugepage | 开启 page cache + 开启 hugepage |
| -------- | ------- | -------------------------- | --------------- | --------------------------------- | ------------------------------- |
| 4G       | restore | 1913.322 ms                | 623.998 ms      | 1906.534 ms                       | 526.327 ms                      |
|          | resume  | 16.584 ms                  | 14.062 ms       | 15.309 ms                         | 15.826 ms                       |
|          | total   | 1929.907 ms                | 638.060 ms      | 1921.843 ms                       | 542.153 ms                      |
| 2G       | restore | 985.878 ms                 | 333.440 ms      | 985.883 ms                        | 312.838 ms                      |
|          | resume  | 14.172 ms                  | 17.197 ms       | 13.876 ms                         | 15.331 ms                       |
|          | total   | 1000.050 ms                | 350.637 ms      | 999.759 ms                        | 328.169 ms                      |
| 1G       | restore | 533.121 ms                 | 190.534 ms      | 521.281 ms                        | 153.869 ms                      |
|          | resume  | 15.142 ms                  | 16.542 ms       | 14.697 ms                         | 16.558 ms                       |
|          | total   | 548.263 ms                 | 207.076 ms      | 535.978 ms                        | 170.427 ms                      |
| 512M     | restore | 304.199 ms                 | 105.908 ms      | 300.310 ms                        | 102.818 ms                      |
|          | resume  | 15.110 ms                  | 16.547 ms       | 15.617 ms                         | 15.310 ms                       |
|          | total   | 319.310 ms                 | 122.456 ms      | 315.927 ms                        | 118.128 ms                      |

分析：

1. resume 操作的时间非常固定，稳定在 15 毫秒左右，不受内存规格和各种优化策略的影响
2. restore 操作和内存规格影响非常大，基本呈现线性
3. 开启 page cache 能极大的加速 restore 的速度，但也只能加速到 30% 左右，不能直接到非常小，而且也是和内存规格直接相关（几乎也是线性）
4. 开启 hugepage 能再次加速 restore 的速度，但很有限。

结论：

1. restore 操作的确是在读取和操作完整的内存快照文件，并没有跳过空闲内存



