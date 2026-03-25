---
title: "构建"
linkTitle: "构建"
weight: 30
date: 2025-11-15
description: >
  定制用于 microvm 的极致精简内核
---

## 信息

cloud hypervisor官方优化内核的代码仓库为：

https://github.com/cloud-hypervisor/linux/

选择目前最稳定的分支版本： 6.12.8

## 构建

### 准备工作

先安装依赖包：

```bash
sudo apt install libelf-dev
sudo apt install bc
```

下载源码，切到 6.12.8 分支：

```bash
mkdir /home/sky/work/code/cloud-hypervisor
cd /home/sky/work/code/cloud-hypervisor

git clone --depth 1 https://github.com/cloud-hypervisor/linux.git -b ch-6.12.8
cd linux
```

### 默认构建

```bash
# 生成 Cloud Hypervisor 专属精简配置
make ch_defconfig

# 开始编译 (只生成 vmlinux，不浪费时间打包 bzImage)
make -j$(nproc) vmlinux
```

完成后，生成的 vmlinux 文件就放在根目录，大小为 48MB：

```bash
ls -lh vmlinux         
-rwxrwxr-x 1 sky sky 48M Feb 26 19:45 vmlinux
```

这个版本的内核还是太大了，包含了大量的调试信息和不必要的通用驱动。需要继续裁剪。

## 定制极简内核

可以通过菜单方式进行配置：

```bash
make menuconfig
```

![](images/make%20menuconfig_001.png)

但这个方式比较繁琐，而且不方便重用。因此我们采用命令行方式进行配置。

### 查找内核空间占用

首先找出来内核里最占空间的“罪魁祸首”是谁，比如这是执行 `make ch_defconfig ` 之后构建的 linux 内核的空间占用结果：

```bash
nm --size-sort -r vmlinux | head -n 20
```

输出结果为：

```bash
0000000000580000 d _printk_rb_static_infos
0000000000400000 d sme_workarea
0000000000200000 b __log_buf
0000000000180000 d _printk_rb_static_descs
0000000000044000 b preferred_node_policy
0000000000040000 D early_dynamic_pgts
0000000000019000 d change_point_list
0000000000018000 b wait_table
0000000000010130 B hstates
0000000000010000 b stack_pools
0000000000010000 b __brk_early_pgt_alloc
0000000000010000 b __brk_dmi_alloc
0000000000010000 D __apicid_to_node
000000000000fa04 d e820_table_kexec_init
000000000000fa04 d e820_table_init
000000000000fa04 d e820_table_firmware_init
000000000000fa00 d new_entries
000000000000c800 B pfn_mapped
000000000000c800 d change_point
000000000000c008 b numa_reserved_meminfo
```

此时内核文件的大小为 48 MB：

```bash
ls -lh vmlinux         
-rwxrwxr-x 1 sky sky 48M Feb 26 19:45 vmlinux
```

我们开始逐个进行裁剪，看看内核大小的变化。

### 开启尺寸优化

开启 GCC 的 -Os 优化级别，显著减少内核体积（通常能砍掉 2MB - 5MB）。

```bash
./scripts/config --enable CC_OPTIMIZE_FOR_SIZE

make clean
make olddefconfig

make -j$(nproc) vmlinux
ls -lh vmlinux
nm --size-sort -r vmlinux | head -n 20
```

但比较奇怪的是，单独开启这个优化，内核文件大小反而从 48MB 增加到了 49MB。但没关系，后面陆续裁剪之后这个优化的效果会显露出来。

```bash
ls -lh vmlinux
-rwxrwxr-x 1 sky sky 49M Feb 26 19:46 vmlinux
```

### 缩小日志缓存区

排名第一的 _printk_rb_static_infos 占用了 5.5MB， __log_buf 占用 2MB。

```bash
0000000000580000 d _printk_rb_static_infos
0000000000200000 b __log_buf
```

内核体积大，很大程度上是因为日志缓冲区（Log Buffer）和打印信息设置得太大

路径： General setup -> Kernel log buffer size，默认值是 21，修改为 14.

检查 General setup -> CPU kernel log buffer size contribution，设置为 0.

> 为什么设为 0 是合适的？
>
> 在标准的 Linux 内核中，这个参数的作用是：根据 CPU 核心数动态增加日志缓冲区的大小。
>
> 公式大致为：Total Log Buffer = Default Buffer + (Num_CPUs * 2^Contribution)。
>
> 由于我的 MicroVM 通常只分配 1 到 4 个虚拟 CPU，这种动态增长几乎没有意义，反而会静态预留内存页。
>
> 将其设为 0 意味着禁用动态增长，强制内核仅使用设定的 2^14 (16KB) 的基础缓冲区。对于只跑特定应用的 MicroVM，16KB 的启动日志已经绰绰有余。

界面操作还是比较麻烦的，尤其有些配置隐藏的比较深，可以通过命令来执行：

```bash
# 极度缩小日志缓冲区 (从 2MB+ 压低到 16KB)
./scripts/config --set-val LOG_BUF_SHIFT 14
./scripts/config --set-val LOG_CPU_MAX_BUF_SHIFT 0

# 关闭调试索引和 5 级页表 (节省静态内存)
./scripts/config --disable PRINTK_INDEX
./scripts/config --disable X86_5LEVEL

# 开启 Retpoline 支持
./scripts/config --enable RETPOLINE
```

裁剪之后重新编译：

```bash
make olddefconfig
make clean
make -j$(nproc) vmlinux
ls -lh vmlinux
nm --size-sort -r vmlinux | head -n 20
```

编译之后的 linux 内核文件大小为41MB，和之前的49MB相比减少了8MB：

```bash
ls -lh vmlinux
-rwxrwxr-x 1 sky sky 41M Feb 26 19:48 vmlinux
```

对比之前，_printk_rb_static_infos 和 __log_buf 这两个空间占用第一和第三的项目消失了：

```bash
nm --size-sort -r vmlinux | head -n 20

0000000000400000 d sme_workarea
0000000000044000 b preferred_node_policy
0000000000040000 D early_dynamic_pgts
0000000000019000 d change_point_list
```

### 关闭 SME

继续裁剪SME，Secure Memory Encryption 是 AMD 处理器的内存加密特性，通常在 MicroVM 内部不需要这个功能。

```bash
0000000000400000 d sme_workarea
```

```bash
./scripts/config --disable AMD_MEM_ENCRYPT

make olddefconfig
make clean
make -j$(nproc) vmlinux
ls -lh vmlinux
ls -l vmlinux
nm --size-sort -r vmlinux | head -n 20
```

裁剪 SME 之后，内核文件大小从41MB降低到了30MB：

```bash
ls -lh vmlinux                            
-rwxrwxr-x 1 sky sky 30M Feb 26 19:50 vmlinux
ls -l vmlinux                            
-rwxrwxr-x 1 sky sky 31133144 Feb 26 19:50 vmlinux
```

### 裁剪非必要的子系统

对于 MicroVM 来说，有些子系统是不需要的，比如图形支持，声音支持等。

```bash
./scripts/config --disable DRM
./scripts/config --disable SOUND
./scripts/config --disable ETHERNET
./scripts/config --disable HID
./scripts/config --disable INPUT
./scripts/config --disable KVM
./scripts/config --disable KVM_INTEL
./scripts/config --disable KVM_AMD
./scripts/config --disable PERF_EVENTS
./scripts/config --disable PROFILING
./scripts/config --disable BPF_SYSCALL
./scripts/config --disable AUDIT

make olddefconfig
make clean
make -j$(nproc) vmlinux
ls -lh vmlinux
ls -l vmlinux
nm --size-sort -r vmlinux | head -n 20
```

裁剪非必要的子系统之后，内核文件大小从30MB降低到了26MB：

```bash
-rwxrwxr-x 1 sky sky 26M Feb 26 20:27 vmlinux
-rwxrwxr-x 1 sky sky 26301552 Feb 26 20:27 vmlinux
```

### 精简文件系统

文件系统是 MicroVM 中非常重要的一个部分，但是有些文件系统是不需要的，比如 NTFS, CIFS, NFS 等。

```bash
# --- 1. 核心调试与辅助系统 (体积与内存大头) ---
# DEBUG_FS 是内核中非常庞大的符号和接口库
./scripts/config --disable DEBUG_FS
./scripts/config --disable BLK_DEBUG_FS
./scripts/config --disable STACKTRACE
./scripts/config --disable KALLSYMS_ALL

# --- 2. 移除现代存储高级特性 (MicroVM 通常不需要) ---
# DAX 是为了持久性内存(Optane等)设计的，虚拟化环境极少用到
./scripts/config --disable FS_DAX
./scripts/config --disable FS_DAX_PMD
# 禁用内核级文件加密逻辑
./scripts/config --disable FS_ENCRYPTION
./scripts/config --disable FS_ENCRYPTION_ALGS
./scripts/config --disable FS_VERITY

# --- 3. 辅助文件系统逻辑清理 ---
./scripts/config --disable AUTOFS_FS
./scripts/config --disable EFIVAR_FS
./scripts/config --disable CONFIGFS_FS
./scripts/config --disable FSNOTIFY

# --- 4. Ext4 深度瘦身 (只保留最基础的读写逻辑) ---
# 禁用 ACL(访问控制列表) 和 Security Labels(如 SELinux 标签)
# 如果你的 Rootfs 只是简单的权限管理，这两项可以砍掉几百个符号
./scripts/config --disable EXT4_FS_POSIX_ACL
./scripts/config --disable EXT4_FS_SECURITY
./scripts/config --disable FS_POSIX_ACL
./scripts/config --disable QUOTA

# --- 5. 虚拟化文件系统按需清理 ---
# 如果你只用 virtio-blk (磁盘镜像)，可以关掉下两项；
# 如果你用 Kata Containers 这种 virtio-fs 挂载模式，请保留它们。
./scripts/config --disable FUSE_FS
./scripts/config --disable VIRTIO_FS
./scripts/config --disable OVERLAY_FS

# --- 6. 常规文件系统裁剪清单 --
./scripts/config --disable XFS_FS
./scripts/config --disable BTRFS_FS
./scripts/config --disable F2FS_FS
./scripts/config --disable JFS_FS
./scripts/config --disable REISERFS_FS
./scripts/config --disable NTFS_FS
./scripts/config --disable NTFS3_FS
./scripts/config --disable CIFS
./scripts/config --disable SMB_SERVER
./scripts/config --disable NFS_FS
./scripts/config --disable GFS2_FS
./scripts/config --disable OCFS2_FS
./scripts/config --disable MINIX_FS
./scripts/config --disable HFS_FS
./scripts/config --disable HFSPLUS_FS
./scripts/config --disable ISO9660_FS
./scripts/config --disable UDF_FS
./scripts/config --disable FAT_FS
./scripts/config --disable VFAT_FS
./scripts/config --disable MSDOS_FS
./scripts/config --disable EROFS_FS
./scripts/config --disable SQUASHFS
./scripts/config --disable JOLIET
./scripts/config --disable ZISOFS

make olddefconfig
make clean
make -j$(nproc) vmlinux
ls -lh vmlinux
ls -l vmlinux
nm --size-sort -r vmlinux | head -n 20
```

裁剪文件系统之后，内核文件大小从26.30MB 降到 26.11MB，仅省下了约186KB。体积优化效果非常轻微：

```bash
#优化前
-rwxrwxr-x 1 sky sky 26301552 Feb 26 20:27 vmlinux

#优化后：
-rwxrwxr-x 1 sky sky 26115024 Feb 26 20:48 vmlinux
```

但是，虽然内核体积没减，但这个裁剪操作还有意义的，体现在以下几个方面：

- 攻击面（Attack Surface）减小：文件系统代码是内核漏洞的重灾区。关掉 NTFS、JFS、CIFS 等，意味着黑客无法通过构造恶意的磁盘镜像或网络包来利用这些复杂的解析逻辑。

- 启动稳定性：减少了内核启动时尝试探测不同文件系统类型的开销，虽然对 MicroVM 来说只是几毫秒，但更纯净。

- 内存脚印（Runtime Footprint）：虽然文件体积只差 28KB，但由于去掉了各种文件系统的初始化函数，内核启动后申请的 slab 缓存和全局变量会更少，运行时会更“轻”。

### 裁剪加密与压缩算法

```bash
nm --size-sort -r vmlinux | grep -i ' [tT] ' | head -n 20
```

可以看到有很多加密与压缩算法，这些算法在 MicroVM 中是不需要的：

```bash
0000000000004429 t __twofish_enc_blk8
0000000000004427 t __twofish_dec_blk8
0000000000002966 T intel_pmu_init
00000000000024c1 t sock_ops_convert_ctx_access
0000000000001ddd t ZSTD_decompressSequencesLong_default.constprop.0
0000000000001d86 t ZSTD_decompressSequencesLong_bmi2.constprop.0
0000000000001d85 T __twofish_enc_blk_3way
0000000000001d63 T twofish_dec_blk_3way
0000000000001b62 t ___bpf_prog_run
0000000000001a4c T x64_sys_call
000000000000187d T __twofish_setkey
000000000000183a t __dev_ethtool
0000000000001626 T aes_ctr_enc_256_avx_by8
0000000000001614 t HUF_decompress4X2_usingDTable_internal_default
0000000000001614 t HUF_decompress4X2_usingDTable_internal_bmi2
0000000000001519 T jbd2_journal_commit_transaction
00000000000014fa t ZSTD_decompressSequencesSplitLitBuffer_default.constprop.0
00000000000014a6 t ZSTD_decompressSequencesSplitLitBuffer_bmi2.constprop.0
0000000000001436 T aes_ctr_enc_192_avx_by8
0000000000001321 T blake2s_compress_generic
```

```bash
./scripts/config --disable CRYPTO_TWOFISH
./scripts/config --disable CRYPTO_TWOFISH_COMMON
./scripts/config --disable CRYPTO_TWOFISH_X86_64
./scripts/config --disable CRYPTO_TWOFISH_X86_64_3WAY
./scripts/config --disable CRYPTO_TWOFISH_AVX_X86_64
./scripts/config --disable CRYPTO_AES_NI_INTEL
./scripts/config --disable CRYPTO_MANAGER
./scripts/config --disable ZSTD_COMPRESS
./scripts/config --disable ZSTD_DECOMPRESS
./scripts/config --disable RD_ZSTD

make olddefconfig
make clean
make -j$(nproc) vmlinux
ls -lh vmlinux
ls -l vmlinux
nm --size-sort -r vmlinux | head -n 20
nm --size-sort -r vmlinux | grep -i ' [tT] ' | head -n 20
```

裁剪完成后体积减少了60KB：

```bash
#优化前
-rwxrwxr-x 1 sky sky 26115024 Feb 26 20:48 vmlinux
#优化后：
-rwxrwxr-x 1 sky sky 26055808 Feb 26 20:56 vmlinux
```

### 裁剪多国语言支持

NLS (Native Language Support)

在 ch_defconfig 中，内核通常为了兼容各种磁盘格式，编入了一大堆代码页（Codepages）。

```bash
grep "NLS" .config                           
CONFIG_NLS=y
CONFIG_NLS_DEFAULT="utf8"
CONFIG_NLS_CODEPAGE_437=y
CONFIG_NLS_CODEPAGE_737=y
CONFIG_NLS_CODEPAGE_775=y
CONFIG_NLS_CODEPAGE_850=y
CONFIG_NLS_CODEPAGE_852=y
CONFIG_NLS_CODEPAGE_855=y
CONFIG_NLS_CODEPAGE_857=y
CONFIG_NLS_CODEPAGE_860=y
CONFIG_NLS_CODEPAGE_861=y
CONFIG_NLS_CODEPAGE_862=y
CONFIG_NLS_CODEPAGE_863=y
CONFIG_NLS_CODEPAGE_864=y
CONFIG_NLS_CODEPAGE_865=y
CONFIG_NLS_CODEPAGE_866=y
CONFIG_NLS_CODEPAGE_869=y
CONFIG_NLS_CODEPAGE_936=y
CONFIG_NLS_CODEPAGE_950=y
CONFIG_NLS_CODEPAGE_932=y
CONFIG_NLS_CODEPAGE_949=y
CONFIG_NLS_CODEPAGE_874=y
CONFIG_NLS_ISO8859_8=y
CONFIG_NLS_CODEPAGE_1250=y
CONFIG_NLS_CODEPAGE_1251=y
CONFIG_NLS_ASCII=y
CONFIG_NLS_ISO8859_1=y
CONFIG_NLS_ISO8859_2=y
CONFIG_NLS_ISO8859_3=y
CONFIG_NLS_ISO8859_4=y
CONFIG_NLS_ISO8859_5=y
CONFIG_NLS_ISO8859_6=y
CONFIG_NLS_ISO8859_7=y
CONFIG_NLS_ISO8859_9=y
CONFIG_NLS_ISO8859_13=y
CONFIG_NLS_ISO8859_14=y
CONFIG_NLS_ISO8859_15=y
CONFIG_NLS_KOI8_R=y
CONFIG_NLS_KOI8_U=y
CONFIG_NLS_MAC_ROMAN=y
CONFIG_NLS_MAC_CELTIC=y
CONFIG_NLS_MAC_CENTEURO=y
CONFIG_NLS_MAC_CROATIAN=y
CONFIG_NLS_MAC_CYRILLIC=y
CONFIG_NLS_MAC_GAELIC=y
CONFIG_NLS_MAC_GREEK=y
CONFIG_NLS_MAC_ICELAND=y
CONFIG_NLS_MAC_INUIT=y
CONFIG_NLS_MAC_ROMANIAN=y
CONFIG_NLS_MAC_TURKISH=y
CONFIG_NLS_UTF8=y
```

对于只运行 Linux 现代文件系统（如 ext4/xfs）的 MicroVM，这些全是多余的。

```bash
# 彻底关闭 NLS 支持
# 6.1 强制关闭所有已有的 NLS 子项
grep "CONFIG_NLS_" .config | grep "=y" | cut -d'=' -f1 | while read line; do
    ./scripts/config --disable "$line"
done

# 6.2 重新开启必要的 UTF-8 和基础框架
./scripts/config --enable CONFIG_NLS
./scripts/config --enable CONFIG_NLS_UTF8
./scripts/config --enable CONFIG_NLS_ASCII
./scripts/config --enable CONFIG_NLS_ISO8859_1    # 西欧语言基石，必留
./scripts/config --enable CONFIG_NLS_CODEPAGE_936 # 简体中文支持
./scripts/config --set-str CONFIG_NLS_DEFAULT "utf8"
# 补充：确保基础的转换工具也被编译进去
./scripts/config --enable CONFIG_NLS_UCS2_UTILS

make olddefconfig
make clean
make -j$(nproc) vmlinux
ls -lh vmlinux
ls -l vmlinux
nm --size-sort -r vmlinux | head -n 20
nm --size-sort -r vmlinux | grep -i ' [tT] ' | head -n 20
```

NLS裁剪完成后体积减少了55KB：

```bash
#优化前
-rwxrwxr-x 1 sky sky 26055808 Feb 26 20:56 vmlinux
#优化后：
-rwxrwxr-x 1 sky sky 26000328 Feb 26 21:09 vmlinux
```

为了凑数，减少最后的 328 字节，我们再裁剪一下：

```bash
./scripts/config --disable UTS_NS
```

裁剪完成后体积减少了328字节：

```bash
#优化前
-rwxrwxr-x 1 sky sky 26000328 Feb 26 21:09 vmlinux
#优化后：
-rwxrwxr-x 1 sky sky 25999480 Feb 26 21:13 vmlinux
```
### 总结

把上面的裁剪汇总一下，这样以后如果需要裁剪内核，只需要执行下面的命令：

```bash
make clean
make ch_defconfig

./scripts/config --enable CC_OPTIMIZE_FOR_SIZE
./scripts/config --set-val LOG_BUF_SHIFT 14
./scripts/config --set-val LOG_CPU_MAX_BUF_SHIFT 0
./scripts/config --disable PRINTK_INDEX
./scripts/config --disable X86_5LEVEL
./scripts/config --enable RETPOLINE
./scripts/config --disable AMD_MEM_ENCRYPT

./scripts/config --disable DRM
./scripts/config --disable SOUND
./scripts/config --disable ETHERNET
./scripts/config --disable HID
./scripts/config --disable INPUT
./scripts/config --disable KVM
./scripts/config --disable KVM_INTEL
./scripts/config --disable KVM_AMD
./scripts/config --disable PERF_EVENTS
./scripts/config --disable PROFILING
./scripts/config --disable BPF_SYSCALL
./scripts/config --disable AUDIT

./scripts/config --disable DEBUG_FS
./scripts/config --disable BLK_DEBUG_FS
./scripts/config --disable STACKTRACE
./scripts/config --disable KALLSYMS_ALL
./scripts/config --disable FS_DAX
./scripts/config --disable FS_DAX_PMD
./scripts/config --disable FS_ENCRYPTION
./scripts/config --disable FS_ENCRYPTION_ALGS
./scripts/config --disable FS_VERITY
./scripts/config --disable AUTOFS_FS
./scripts/config --disable EFIVAR_FS
./scripts/config --disable CONFIGFS_FS
./scripts/config --disable FSNOTIFY
./scripts/config --disable EXT4_FS_POSIX_ACL
./scripts/config --disable EXT4_FS_SECURITY
./scripts/config --disable FS_POSIX_ACL
./scripts/config --disable QUOTA
./scripts/config --disable FUSE_FS
./scripts/config --disable VIRTIO_FS
./scripts/config --disable OVERLAY_FS
./scripts/config --disable XFS_FS
./scripts/config --disable BTRFS_FS
./scripts/config --disable F2FS_FS
./scripts/config --disable JFS_FS
./scripts/config --disable REISERFS_FS
./scripts/config --disable NTFS_FS
./scripts/config --disable NTFS3_FS
./scripts/config --disable CIFS
./scripts/config --disable SMB_SERVER
./scripts/config --disable NFS_FS
./scripts/config --disable GFS2_FS
./scripts/config --disable OCFS2_FS
./scripts/config --disable MINIX_FS
./scripts/config --disable HFS_FS
./scripts/config --disable HFSPLUS_FS
./scripts/config --disable ISO9660_FS
./scripts/config --disable UDF_FS
./scripts/config --disable FAT_FS
./scripts/config --disable VFAT_FS
./scripts/config --disable MSDOS_FS
./scripts/config --disable EROFS_FS
./scripts/config --disable SQUASHFS
./scripts/config --disable JOLIET
./scripts/config --disable ZISOFS

./scripts/config --disable CRYPTO_TWOFISH
./scripts/config --disable CRYPTO_TWOFISH_COMMON
./scripts/config --disable CRYPTO_TWOFISH_X86_64
./scripts/config --disable CRYPTO_TWOFISH_X86_64_3WAY
./scripts/config --disable CRYPTO_TWOFISH_AVX_X86_64
./scripts/config --disable CRYPTO_AES_NI_INTEL
./scripts/config --disable CRYPTO_MANAGER
./scripts/config --disable ZSTD_COMPRESS
./scripts/config --disable ZSTD_DECOMPRESS
./scripts/config --disable RD_ZSTD

grep "CONFIG_NLS_" .config | grep "=y" | cut -d'=' -f1 | while read line; do
    ./scripts/config --disable "$line"
done
./scripts/config --enable CONFIG_NLS
./scripts/config --enable CONFIG_NLS_UTF8
./scripts/config --enable CONFIG_NLS_ASCII
./scripts/config --enable CONFIG_NLS_ISO8859_1
./scripts/config --enable CONFIG_NLS_CODEPAGE_936 
./scripts/config --set-str CONFIG_NLS_DEFAULT "utf8"
./scripts/config --enable CONFIG_NLS_UCS2_UTILS
./scripts/config --disable UTS_NS

make clean
make olddefconfig
make -j$(nproc) vmlinux
ls -lh vmlinux
ls -l vmlinux
nm --size-sort -r vmlinux | head -n 20
nm --size-sort -r vmlinux | grep -i ' [tT] ' | head -n 20
```

最终内核大小为25M：

```bash
-rwxrwxr-x 1 sky sky 25M Feb 26 21:21 vmlinux
-rwxrwxr-x 1 sky sky 25999984 Feb 26 21:21 vmlinux
```

为了方便使用，创建脚本:

```bash
vi build_minimal_kernel.sh
```

内容为:

```bash
#!/bin/bash
set -e

echo "开始构建极致瘦身内核 (Target < 26MB)..."

# 1. 初始化配置
make clean
make ch_defconfig

# 2. 核心架构与编译器优化
./scripts/config --enable CC_OPTIMIZE_FOR_SIZE
./scripts/config --set-val LOG_BUF_SHIFT 14
./scripts/config --set-val LOG_CPU_MAX_BUF_SHIFT 0
./scripts/config --disable PRINTK_INDEX
./scripts/config --disable X86_5LEVEL
./scripts/config --enable RETPOLINE
./scripts/config --disable AMD_MEM_ENCRYPT
./scripts/config --disable UTS_NS

# 3. 剥离大型子系统 (显卡、声音、以太网、输入设备等)
./scripts/config --disable DRM
./scripts/config --disable SOUND
./scripts/config --disable ETHERNET
./scripts/config --disable HID
./scripts/config --disable INPUT
./scripts/config --disable KVM
./scripts/config --disable KVM_INTEL
./scripts/config --disable KVM_AMD
./scripts/config --disable PERF_EVENTS
./scripts/config --disable PROFILING
./scripts/config --disable BPF_SYSCALL
./scripts/config --disable AUDIT

# 4. 文件系统深度裁剪 (仅保留基础 Ext4 逻辑)
./scripts/config --disable DEBUG_FS
./scripts/config --disable BLK_DEBUG_FS
./scripts/config --disable STACKTRACE
./scripts/config --disable KALLSYMS_ALL
./scripts/config --disable FS_DAX
./scripts/config --disable FS_DAX_PMD
./scripts/config --disable FS_ENCRYPTION
./scripts/config --disable FS_ENCRYPTION_ALGS
./scripts/config --disable FS_VERITY
./scripts/config --disable AUTOFS_FS
./scripts/config --disable EFIVAR_FS
./scripts/config --disable CONFIGFS_FS
./scripts/config --disable FSNOTIFY
./scripts/config --disable EXT4_FS_POSIX_ACL
./scripts/config --disable EXT4_FS_SECURITY
./scripts/config --disable FS_POSIX_ACL
./scripts/config --disable QUOTA
./scripts/config --disable FUSE_FS
./scripts/config --disable VIRTIO_FS
./scripts/config --disable OVERLAY_FS

# 批量关闭其他不常用文件系统
for fs in XFS_FS BTRFS_FS F2FS_FS JFS_FS REISERFS_FS NTFS_FS NTFS3_FS CIFS SMB_SERVER \
          NFS_FS GFS2_FS OCFS2_FS MINIX_FS HFS_FS HFSPLUS_FS ISO9660_FS UDF_FS \
          FAT_FS VFAT_FS MSDOS_FS EROFS_FS SQUASHFS JOLIET ZISOFS; do
    ./scripts/config --disable $fs
done

# 5. 加密与压缩算法裁剪 (剔除汇编优化重灾区)
./scripts/config --disable CRYPTO_TWOFISH
./scripts/config --disable CRYPTO_TWOFISH_COMMON
./scripts/config --disable CRYPTO_TWOFISH_X86_64
./scripts/config --disable CRYPTO_TWOFISH_X86_64_3WAY
./scripts/config --disable CRYPTO_TWOFISH_AVX_X86_64
./scripts/config --disable CRYPTO_AES_NI_INTEL
./scripts/config --disable CRYPTO_MANAGER
./scripts/config --disable ZSTD_COMPRESS
./scripts/config --disable ZSTD_DECOMPRESS
./scripts/config --disable RD_ZSTD

# 6. 国际化 NLS 精简与简体中文/西欧补全
# 先清空所有 CodePage
grep "CONFIG_NLS_" .config | grep "=y" | cut -d'=' -f1 | while read line; do
    ./scripts/config --disable "$line"
done
# 精准开启
./scripts/config --enable CONFIG_NLS
./scripts/config --enable CONFIG_NLS_UTF8
./scripts/config --enable CONFIG_NLS_ASCII
./scripts/config --enable CONFIG_NLS_ISO8859_1
./scripts/config --enable CONFIG_NLS_CODEPAGE_936 
./scripts/config --set-str CONFIG_NLS_DEFAULT "utf8"
./scripts/config --enable CONFIG_NLS_UCS2_UTILS

# 7. 应用配置并编译
make olddefconfig
make -j$(nproc) vmlinux

# 8. 成果展示
echo "------------------------------------------"
ls -lh vmlinux
ls -l vmlinux
echo "Top 10 Functions by Size:"
nm --size-sort -r vmlinux | head -n 10
echo "------------------------------------------"
echo "构建完成！"
```

增加执行权限：

```bash
chmod +x build_minimal_kernel.sh
```

执行：

```bash
./build_minimal_kernel.sh
```

执行完成后，内核文件会生成在当前目录下，大小为25M。

## 定制agent专属内核

在上面定制的极简内核的基础上，针对 agent 的特殊使用场景，进行定制。主要的改动有：

- 启动 cloud hypervisor 时， 内核和 rootfs 都是只读方式
- 内核文件通过直接内核加载 (Direct Kernel Boot) 读入，跳过 BIOS/UEFI 阶段
- 使用 virtio-pmem 的方式挂载 rootfs，并开启 DAX 功能，以最大化利用 Host 的 Page Cache
- 宿主机开启 KSM (Kernel Samepage Merging) 功能，以最大化利用内存

为此，内核需要进行改动：

- 磁盘格式由 ext4 修改为 erofs
- 需要支持 overlayfs 功能，以支持 rootfs 和可写层
- 需要支持 dax 功能，以最大化利用 Host 的 Page Cache

### 构建脚本

为了方便使用，创建脚本:

```bash
vi build_agent_kernel.sh
```

内容为:

```bash
#!/bin/bash
set -e

echo "开始构建 agent 专属内核 (EROFS + DAX + Ext4 Writable)..."

# 1. 初始化配置
make clean
make ch_defconfig

# 2. 核心架构与编译器优化
./scripts/config --enable CC_OPTIMIZE_FOR_SIZE
./scripts/config --set-val LOG_BUF_SHIFT 14
./scripts/config --set-val LOG_CPU_MAX_BUF_SHIFT 0
./scripts/config --disable PRINTK_INDEX
./scripts/config --disable X86_5LEVEL
./scripts/config --enable RETPOLINE
./scripts/config --disable AMD_MEM_ENCRYPT
./scripts/config --disable UTS_NS

# 3. 剥离大型子系统
./scripts/config --disable DRM
./scripts/config --disable SOUND
./scripts/config --disable ETHERNET
./scripts/config --disable HID
./scripts/config --disable INPUT
./scripts/config --disable KVM
./scripts/config --disable KVM_INTEL
./scripts/config --disable KVM_AMD
./scripts/config --disable PERF_EVENTS
./scripts/config --disable PROFILING
./scripts/config --disable BPF_SYSCALL
./scripts/config --disable AUDIT

# 4. 文件系统基础裁剪 (先关掉不需要的)
./scripts/config --disable DEBUG_FS
./scripts/config --disable BLK_DEBUG_FS
./scripts/config --disable STACKTRACE
./scripts/config --disable KALLSYMS_ALL
./scripts/config --disable FS_ENCRYPTION
./scripts/config --disable FS_ENCRYPTION_ALGS
./scripts/config --disable FS_VERITY
./scripts/config --disable AUTOFS_FS
./scripts/config --disable EFIVAR_FS
./scripts/config --disable CONFIGFS_FS
./scripts/config --disable FSNOTIFY
./scripts/config --disable QUOTA
./scripts/config --disable FUSE_FS
./scripts/config --disable VIRTIO_FS

# 批量关闭不常用文件系统
for fs in XFS_FS BTRFS_FS F2FS_FS JFS_FS REISERFS_FS NTFS_FS NTFS3_FS CIFS SMB_SERVER \
          NFS_FS GFS2_FS OCFS2_FS MINIX_FS HFS_FS HFSPLUS_FS ISO9660_FS UDF_FS \
          FAT_FS VFAT_FS MSDOS_FS SQUASHFS JOLIET ZISOFS; do
    ./scripts/config --disable $fs
done

# 5. 加密与压缩算法裁剪
./scripts/config --disable CRYPTO_TWOFISH
./scripts/config --disable CRYPTO_TWOFISH_COMMON
./scripts/config --disable CRYPTO_TWOFISH_X86_64
./scripts/config --disable CRYPTO_TWOFISH_X86_64_3WAY
./scripts/config --disable CRYPTO_TWOFISH_AVX_X86_64
./scripts/config --disable CRYPTO_AES_NI_INTEL
./scripts/config --disable CRYPTO_MANAGER
./scripts/config --disable ZSTD_COMPRESS
./scripts/config --disable ZSTD_DECOMPRESS
./scripts/config --disable RD_ZSTD

# 6. NLS 保持 (UTF-8 + 中文支持)
grep "CONFIG_NLS_" .config | grep "=y" | cut -d'=' -f1 | while read line; do
    ./scripts/config --disable "$line"
done
./scripts/config --enable CONFIG_NLS
./scripts/config --enable CONFIG_NLS_UTF8
./scripts/config --enable CONFIG_NLS_ASCII
./scripts/config --enable CONFIG_NLS_ISO8859_1
./scripts/config --enable CONFIG_NLS_CODEPAGE_936 
./scripts/config --set-str CONFIG_NLS_DEFAULT "utf8"
./scripts/config --enable CONFIG_NLS_UCS2_UTILS

# 7. 开启 DAX 与内存管理 (EROFS DAX 必备)
./scripts/config --enable FS_DAX
./scripts/config --enable FS_DAX_PMD
./scripts/config --enable DAX
./scripts/config --enable ZONE_DEVICE
./scripts/config --enable MEMORY_HOTPLUG
./scripts/config --enable MEMORY_HOTREMOVE

# 8. 开启设备驱动 (virtio-pmem 用于 Rootfs, virtio-blk 用于写层)
./scripts/config --enable VIRTIO_PMEM
./scripts/config --enable LIBNVDIMM
./scripts/config --enable BLK_DEV_PMEM
./scripts/config --enable VIRTIO_BLK

# 9. 开启 EROFS (只读 Rootfs 格式)
./scripts/config --enable EROFS_FS
./scripts/config --enable EROFS_FS_XATTR
./scripts/config --enable EROFS_FS_POSIX_ACL
./scripts/config --enable EROFS_FS_ZIP

# 10. 开启 OverlayFS
./scripts/config --enable OVERLAY_FS

# 11. 优化 Ext4 (写层专用)
./scripts/config --enable EXT4_FS
./scripts/config --disable EXT4_FS_POSIX_ACL
./scripts/config --disable EXT4_FS_SECURITY
./scripts/config --disable EXT4_DEBUG
./scripts/config --disable EXT4_KUNIT_TESTS

# 12. 应用配置并编译
make olddefconfig
make -j$(nproc) vmlinux

# 13. 成果展示
echo "------------------------------------------"
ls -lh vmlinux
echo "Top 10 Functions by Size:"
nm --size-sort -r vmlinux | head -n 10
echo "------------------------------------------"
echo "agent 专属内核构建完成！"
```

增加执行权限：

```bash
chmod +x build_agent_kernel.sh
```

执行：

```bash
./build_agent_kernel.sh
```

执行完成后，内核文件会生成在当前目录下，大小为25M。

```bash
-rwxrwxr-x 1 sky sky 25M Feb 28 09:43 vmlinux
```