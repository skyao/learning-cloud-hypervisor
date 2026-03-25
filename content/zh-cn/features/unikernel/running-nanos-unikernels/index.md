---
title: "Running Nanos Unikernels under Cloud Hypervisor"
linkTitle: "Running Nanos Unikernels"
weight: 300
date: 2025-10-15
description: >
  youtube上的介绍视频
---

Running Nanos Unikernels under Cloud Hypervisor

https://www.youtube.com/watch?v=1IteC54pcWM


-----




这是一份为您详细整理的视频内容图文解析。由于我无法直接生成视频截图文件，我将采用**“关键画面描述与终端代码块还原”**的方式来为您展示“图片”和视觉信息，结合详细的文字解析，带您完整回顾该视频。

---

### 📑 视频内容摘要 (Summary)

本视频是一个关于 **Cloud Hypervisor** 与 **Nanos Unikernel (单内核)** 结合使用的硬核技术演示。
作者首先介绍了 Cloud Hypervisor 的官方网站、核心特性（如基于 Rust 开发、毫秒级启动、极小内存开销等）以及其背后的支持大厂（包含腾讯云等）。随后，作者转入 Linux 终端实战，展示了一个简单的 Go 语言程序，将其编译为 Nanos Unikernel 镜像，并通过 Cloud Hypervisor 极速拉起该轻量级虚拟机。程序成功在虚拟机内运行并输出日志。最后，作者通过强杀进程结束了演示，并总结指出这种极速、轻量的技术栈非常适合用来构建 Serverless（无服务器）架构。

---

### 🎬 详细内容与图文解析 (Detailed Breakdown)

#### 第一阶段：介绍 Cloud Hypervisor (00:00 - 00:27)

*   **📺 画面展示：**
    浏览器停留在 Cloud Hypervisor 的官方网站主页 (`cloudhypervisor.org`)。
    > **[网页视觉呈现]**
    > **大标题**：Run Cloud Virtual Machines Securely and Efficiently (安全高效地运行云虚拟机)
    > **简介**：Cloud Hypervisor is an open source Virtual Machine Monitor (VMM) implemented in Rust...
    > **核心特性面板**：Secure (安全)、Fast (快，启动<100ms)、Slim (轻量)、Kata Containers (支持 Kata)、Live migration (热迁移) 等。
    > **支持机构**：Alibaba, AMD, ARM, ByteDance, Intel, Microsoft, **Tencent Cloud**。

*   **📝 文字解说：**
    作者欢迎大家来到视频教程系列。今天的主角是 Cloud Hypervisor。他介绍道，这是一个用 Rust 语言编写的全新虚拟机监视器（VMM）。它拥有许多酷炫的功能，并且得到了如阿里巴巴、字节跳动、英特尔、腾讯、Ampere 等众多大厂的支持。作者预告，今天的演示目标是在 Cloud Hypervisor 上启动一些 Unikernel（单内核）。

#### 第二阶段：准备 Go 语言测试程序 (00:27 - 00:50)

*   **📺 画面展示：**
    画面切换到纯黑背景的 Linux 终端。作者使用 Vim 打开了一个名为 `main.go` 的文件。
    > **[终端代码截图还原]**
    ```go
    package main
    import (
        "fmt"
        "time"
    )
    func main() {
        for i := 0; i < 10; i++ {
            fmt.Println("hello from nanos")
            time.Sleep(1 * time.Second)
        }
    }
    ```

*   **📝 文字解说：**
    作者展示了他准备好的一个非常简单的 Go 语言程序。这个程序的功能只有一个：通过一个 `for` 循环，每隔 1 秒钟在控制台打印一次 `"hello from nanos"`，总共循环 10 次。作者表示，他在此之前已经提前将这个 Go 程序编译好了。

#### 第三阶段：查看编译产物与 Nanos 内核 (00:50 - 01:36)

*   **📺 画面展示：**
    终端执行了 `ls -lh` 命令，展示当前目录下的文件列表和大小。
    > **[终端输出截图还原]**
    ```bash
    cyberg@nanos:~/...$ ls -lh
    -rwxrwxr-x 1 cyberg cyberg 4.0M May 24 08:15 cloud-hypervisor
    -rw-rw-r-- 1 cyberg cyberg 2.4M May 24 08:27 kernel.img
    -rwxrwxr-x 1 cyberg cyberg 120  May 24 08:30 run.sh
    ```

*   **📝 文字解说：**
    作者向大家展示当前工作目录中的三个关键文件：
    1.  `cloud-hypervisor`：编译好的 VMM 执行文件，体积非常小巧，大约只有 **4MB**。
    2.  `kernel.img`：这是 Nanos Unikernel 的内核镜像文件。
    3.  为了照顾想要自己从源码编译 Nanos 的开发者，作者还特意敲击路径，展示了这个 `kernel.img` 在 Nanos 源码树中的默认生成位置（通常在 `.../nanos/output/platform/pc/bin/kernel.img`）。

#### 第四阶段：一键极速启动轻量虚拟机 (01:36 - 02:18)

*   **📺 画面展示：**
    作者查看并执行了启动脚本 `run.sh`。随后，终端立刻开始疯狂输出虚拟机的启动日志和 Go 程序的运行结果。
    > **[启动脚本与运行画面还原]**
    ```bash
    # 查看 run.sh 内容
    cyberg@nanos:~/...$ cat run.sh
    ./cloud-hypervisor --kernel kernel.img --disk "path=/home/cyberg/ch/g" --console off --serial tty

    # 执行脚本
    cyberg@nanos:~/...$ sudo ./run.sh
    [sudo] password for cyberg: ******

    # 虚拟机瞬间拉起并输出
    en1: assigned FE80::2C2C:47FF:FEE6:7D07
    hello from nanos
    hello from nanos
    hello from nanos
    ...
    ```

*   **📝 文字解说：**
    作者打开了 `run.sh` 脚本，讲解了启动 Cloud Hypervisor 的核心参数：指定内核文件 (`--kernel`)、指定挂载的磁盘路径 (`--disk`) 以及将输出重定向到串口终端 (`--serial tty`)。
    当他输入 `sudo` 密码按下回车后，**虚拟机在瞬间（毫秒级）就完成了启动**。我们直接在屏幕上看到了网卡分配的日志，紧接着，刚才编写的 Go 程序开始正常工作，屏幕上稳稳地每秒打印出一句 `"hello from nanos"`。

#### 第五阶段：展示进程管理并结束演示 (02:18 - 02:44)

*   **📺 画面展示：**
    作者将终端分屏（Split Screen）。左侧继续打印日志，右侧用于查找并杀掉进程。
    > **[分屏操作画面还原]**
    ```bash
    # 右侧终端操作
    cyberg@nanos:~/...$ ps aux | grep cloud
    root     13904  0.0  0.0 2004488 64536 pts/2 ... ./cloud-hypervisor --kernel ...
    cyberg@nanos:~/...$ sudo kill -9 13904

    # 左侧终端显示：
    hello from nanos
    hello from nanos
    Killed
    cyberg@nanos:~/...$
    ```

*   **📝 文字解说：**
    作者提到，Cloud Hypervisor 其实还提供了一个非常强大的 API（REST API），可以通过 API 来控制虚拟机的生命周期，但他今天不打算演示，留到以后的视频再说。
    为了结束当前一直打印的虚拟机，他在右侧终端使用 `ps aux` 找到了 Cloud Hypervisor 的进程 PID，然后直接运行 `sudo kill -9` 强杀了该进程。左侧的虚拟机随即停止运行，显示 `Killed` 并退回了宿主机的终端提示符。
    最后，作者总结道：“这就是在 Cloud Hypervisor 下运行 Nanos Unikernel 的演示。我认为**你可以利用这项技术制作出非常酷炫的 Serverless（无服务器）风格的特性**。下次再见！”


