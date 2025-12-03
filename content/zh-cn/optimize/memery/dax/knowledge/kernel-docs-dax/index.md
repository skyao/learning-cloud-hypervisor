---
title: "kernel docs"
linkTitle: "kernel docs"
weight: 10
date: 2025-11-15
description: >
  linux kernel docs 中对 dax 的介绍
---

https://docs.kernel.org/filesystems/dax.html

Direct Access for files

----

## 动机

页面缓存（page cache）通常用于缓冲对文件的读写操作。它还用于通过调用 mmap 映射到用户空间的页面。

对于类内存的块设备，页缓存页面（page cache pages）将是原始存储的不必要副本。DAX 代码通过直接对存储设备执行读写操作来消除这个额外副本。对于文件映射，存储设备会直接映射到用户空间。

## 用法

如果你有一个支持 DAX 的块设备，可以像往常一样在其上创建文件系统。目前 DAX 代码仅支持块大小等于内核 PAGE_SIZE 的文件，因此在创建文件系统时可能需要指定块大小。

目前有 5 个文件系统支持 DAX：ext2、ext4、xfs、virtiofs 和 erofs。在这些文件系统上启用 DAX 的方式各不相同。

### 在 ext2 和 erofs 上启用 DAX

挂载文件系统时，在命令行上使用 -o dax 选项或在 /etc/fstab 的选项中添加 'dax'。这可以启用文件系统中所有文件的 DAX 功能。它等同于下面的 `-o dax=always` 行为。

### 在 xfs 和 ext4 上启用 DAX

#### 摘要

1. 存在一个内核文件访问模式标志 S_DAX，对应于 statx 标志 STATX_ATTR_DAX。有关此访问模式的详细信息，请参阅 statx(2) 的手册页。

2. 存在一个持久的标志 FS_XFLAG_DAX，它可以应用于常规文件和目录。这个建议标志可以在任何时候设置或清除，但这样做不会立即影响 S_DAX 状态。

3. 如果在目录上设置了持久的 FS_XFLAG_DAX 标志，那么随后在此目录中创建的所有常规文件和子目录都将继承此标志。在父目录上设置或清除此标志时已存在的文件和子目录不会受到此父目录修改的影响。

4. 存在一些 dax 挂载选项可以覆盖 FS_XFLAG_DAX 在 S_DAX 标志设置中的值。假设底层存储支持 DAX，则以下情况成立：

   - -o dax=inode 表示"遵循 FS_XFLAG_DAX"，这是默认设置。
   - -o dax=never 表示"永不设置 S_DAX，忽略 FS_XFLAG_DAX。"
   - -o dax=always 表示"始终设置 S_DAX 忽略 FS_XFLAG_DAX。"
   - -o dax 是一个传统选项，它是 dax=always 的别名。

   警告： 选项 -o dax 可能会在未来被移除，因此 -o dax=always 是指定此行为的推荐方法。

   注意： 即使文件系统以 dax 选项挂载，FS_XFLAG_DAX 的修改和继承行为也保持不变。但是，直到文件系统以 dax=inode 重新挂载并且 inode 从内核内存中驱逐之前，内存中的 inode 状态（S_DAX）将被覆盖。

5. 可以通过以下方式更改 S_DAX 策略：

   - 在创建文件前根据需要设置父目录的 FS_XFLAG_DAX 标志

   - 设置适当的 dax="foo" 挂载选项

   - 更改现有常规文件和目录上的 FS_XFLAG_DAX 标志。这具有下文 6) 中描述的运行时限制和约束。

6. 通过切换持久化的 FS_XFLAG_DAX 标志来更改 S_DAX 策略时，对现有常规文件的更改只有在所有进程关闭这些文件后才会生效。

### 详情

每个文件有 2 个 dax 标志。一个是持久的 inode 设置（FS_XFLAG_DAX），另一个是表示该功能活动状态的易失性标志（S_DAX）。

FS_XFLAG_DAX 在文件系统中被保留。这个持久性配置设置可以使用 FS_IOC_FS`[`GS]`ETXATTR ioctl（参见 ioctl_xfs_fsgetxattr(2)）或诸如 'xfs_io' 之类的工具进行设置、清除和/或查询。

新文件和目录在创建时会自动从其父目录继承 FS_XFLAG_DAX。因此，在创建目录时设置 FS_XFLAG_DAX 可以为整个子树设置默认行为。

为阐明继承关系，以下是 3 个示例：

Example A: 示例 A：

```
mkdir -p a/b/c
xfs_io -c 'chattr +x' a
mkdir a/b/c/d
mkdir a/e

------[outcome]------

dax: a,e
no dax: b,c,d
```

Example B: 示例 B：

```
mkdir a
xfs_io -c 'chattr +x' a
mkdir -p a/b/c/d

------[outcome]------

dax: a,b,c,d
no dax:
```

Example C: 示例 C：

```
mkdir -p a/b/c
xfs_io -c 'chattr +x' c
mkdir a/b/c/d

------[outcome]------

dax: c,d
no dax: a,b
```


当前启用状态(S_DAX)是在内核将文件 inode 实例化到内存时设置的。该状态基于底层媒体支持、FS_XFLAG_DAX 的值以及文件系统的 dax 挂载选项进行设置。

statx 可用于查询 S_DAX。

{{% alert title="注意" color=warning %}} 

只有常规文件才会设置 S_DAX 标志，因此 statx 绝不会指示目录上设置了 S_DAX。

 {{% /alert %}}

即使底层介质不支持 dax 和/或文件系统被挂载选项覆盖，设置 FS_XFLAG_DAX 标志（无论是直接设置还是通过继承）仍然会发生。

### 在 virtiofs 上启用 DAX

virtiofs 上 DAX 的语义基本上与 ext4 和 xfs 上的语义相同，只是当指定了'-o dax=inode'时，virtiofs 客户端通过 FUSE 协议从 virtiofs 服务器获取是否启用 DAX 的提示，而不是持久的 FS_XFLAG_DAX 标志。也就是说，是否启用 DAX 完全由 virtiofs 服务器决定，而 virtiofs 服务器本身可能部署各种算法来做出此决定，例如取决于主机上的持久 FS_XFLAG_DAX 标志。

在客户机内部设置或清除持久性 FS_XFLAG_DAX 标志仍然受支持，但不能保证相应文件会启用或禁用 DAX。客户机内的用户仍需调用 statx(2) 并检查 statx 标志 STATX_ATTR_DAX，以查看是否为此文件启用了 DAX。



## 块驱动程序编写者的实现技巧

要在块驱动程序中支持 DAX，请实现'direct_access'块设备操作。该操作用于将扇区号（以 512 字节扇区为单位）转换为标识内存物理页面的页框号(pfn)。它还会返回一个可用于访问内存的内核虚拟地址。

direct_access 方法接受一个 'size' 参数，表示请求的字节数。该函数应返回在该偏移量处可以连续访问的字节数。如果发生错误，它也可以返回一个负的 errno。

为了支持此方法，存储必须始终能够被 CPU 字节访问。如果您的设备使用分页技术通过较小的窗口暴露大量内存，那么您就无法实现 direct_access。同样，如果您的设备偶尔会使 CPU 长时间停滞，您也不应尝试实现 direct_access。

这些块设备可作为参考： - pmem：NVDIMM 持久内存驱动

## 文件系统编写者的实现技巧

文件系统支持包括：

- 通过在 i_flags 中设置 S_DAX 标志来添加将 inode 标记为 DAX 的支持
- 实现 ->read_iter 和 ->write_iter 操作，当 inode 设置了 S_DAX 标志时使用 `dax_iomap_rw()`
- 为 DAX 文件实现一个 mmap 文件操作，该操作在 VMA 上设置 VM_MIXEDMAP 和 VM_HUGEPAGE 标志，并将 vm_ops 设置为包含 fault、pmd_fault、page_mkwrite 和 pfn_mkwrite 的处理程序。这些处理程序可能应该调用 `dax_iomap_fault()` ，并传递适当的故障大小和 iomap 操作。
- 调用 `iomap_zero_range()` 为 DAX 文件传递适当的 iomap 操作，而不是 `block_truncate_page()`
- 确保在读取、写入、截断和缺页之间有足够的锁机制

用于分配块的 iomap 处理程序必须确保分配的块在返回前已清零并转换为已写入范围，以避免通过 mmap 暴露未初始化的数据。

## 处理介质错误

libnvdimm 子系统为每个 pmem 块设备（在 gendisk->badblocks 中）存储已知介质错误位置的记录。如果我们在此类位置或存在尚未发现的潜在错误的位置发生故障，应用程序可以预期会收到 SIGBUS。libnvdimm 还允许通过简单写入受影响的扇区来清除这些错误（通过 pmem 驱动，并且如果底层 NVDIMM 支持 ACPI 定义的 clear_poison DSM）。

由于 DAX IO 通常不会经过 `driver/bio` 路径，应用程序或系统管理员可以通过以下方式从先前的 `backup/inbuilt` 冗余中恢复丢失的数据：

1. 删除受影响的文件，并从备份中恢复（系统管理员方式）：这将释放文件占用的文件系统块，下次分配这些块时，它们将被先清零，这个过程通过驱动程序完成，可以清除坏扇区。
2. 截断或打孔处理文件中含有坏块的部分（至少需要打孔处理整个对齐的扇区，但不一定是整个文件系统块）。

这是两条基本路径，它们允许 DAX 文件系统在媒体错误存在的情况下继续运行。未来可以在此基础上构建更强大的错误恢复机制，例如，通过 DM 在块层提供的冗余/镜像，或者额外地在文件系统级别提供。这些机制必须依赖于上述两个基本原则：错误清除可以通过驱动程序发送 IO 来完成，或者通过清零（同样通过驱动程序）来完成。

## 不足之处

即使内核或其模块存储在支持 DAX 的文件系统上，该文件系统位于支持 DAX 的块设备上，它们仍然会被复制到 RAM 中。

DAX 代码在具有虚拟映射缓存的架构（如 ARM、MIPS 和 SPARC）上无法正常工作。

对从 DAX 文件 mmapped 的用户内存范围调用 `get_user_pages()` ，当没有 `struct page` 来描述这些页面时会失败。这个问题已经在一些设备驱动程序中得到解决，通过添加可选的 `struct page` 支持来支持驱动程序控制的页面（参见 `drivers/nvdimm` 中的 CONFIG_NVDIMM_PFN 示例，了解如何实现这一点）。在非 `struct page` 情况下，从非 DAX 文件对这些内存范围进行 O_DIRECT 读取/写入将会失败

{{% alert title="注意" color=warning %}} 

对 DAX 文件的 O_DIRECT 读取/写入是有效的，关键在于所访问的内存）。在非 `struct page` 情况下，其他无法实现的功能包括 RDMA、 `sendfile()` 和 `splice()` 。

 {{% /alert %}}







