Bootloader（引导加载程序） 是计算机或嵌入式设备通电后运行的第一段代码，负责初始化硬件、加载操作系统内核，然后把控制权交给操作系统。

它在启动流程中的位置

上电 → Bootloader → 操作系统内核 → 用户空间 

CPU 复位后从固定地址开始执行，这段代码就是 bootloader。它先做最基础的硬件初始化（时钟、内存控制器、串口等），然后从存储设备（硬盘、eMMC、SD 卡等）找到操作系统镜像，加载到内存，最后跳转执行。

不同平台的典型实现

PC（x86）：BIOS/UEFI 之后通常是 [[GRUB]]、systemd-boot、LILO 等，支持多系统选择

嵌入式 Linux：U-Boot（最主流）、barebox

单片机：往往没有 OS，bootloader 可能只是个 ISP/IAP 烧录器

Android 手机：分 bootloader + recovery 两层；"解锁 bootloader" 指解除厂商对 boot 分区的签名校验限制，从而刷入第三方 recovery/ROM

Bootloader 的核心职责

硬件初始化 —— 把 CPU、RAM、时钟、存储控制器带到可用状态

加载内核 —— 从指定分区/设备读取 OS 镜像到 RAM

传参给内核 —— 通过 ATAGS 或 Device Tree Blob 告诉内核硬件信息

可选功能 —— 串口交互、网络启动（TFTP/NFS）、固件升级、防 brick 恢复

一个具体的例子：U-Boot 启动阶段

1. SPL (Secondary Program Loader) —— 初始化 DDR 2. 加载完整的 U-Boot 到 RAM 3. U-Boot 初始化外设，读取环境变量 4. 执行 bootcmd：加载 kernel + dtb 到 RAM 5. bootz 跳转到内核入口 

💡 在 Android 设备上，"OEM unlock" 解的就是 bootloader 阶段的 verify 锁——解锁后才能刷入未签名的 boot.img，这也是 root / 装 Magisk / 上第三方 ROM 的前提。

想深入哪一块？比如 U-Boot 命令调试、Android bootloader 解锁风险、还是 自己写一个裸机 bootloader？告诉我场景我能给得更具体。

