你打的 "TSH" 在 UEFI 七阶段里**没有这个标准缩写**——结合你前面一路在问 SEC/PEI/DXE/BDS/EFI，你大概率想问的是 **TSL**（Transient System Load，临时系统加载），是七阶段里的第五阶段 。下面按 TSL 给你拆，如果你确指别的 "TSH"（比如医学里的促甲状腺激素），那是另一个领域，跟这条线无关。

## TSL = Transient System Load（临时系统加载）

**一句话定位**：BDS 阶段通过 `StartImage()` 把 OS Loader（如 `grubx64.efi` / `bootmgfw.efi`）启动之后、**OS Loader 调用 `ExitBootServices()` 之前**的那段——OS Loader 作为 **UEFI 应用程序**在跑，既能用 Boot Services 又能用 Runtime Services 。

> 💡 规范原文说得很直白：TSL "less a 'phase' in the firmware and more a defined state of the system"——它与其说是固件的一个"阶段"，不如说是系统的一种"定义状态"。因为此时固件代码（DXE/BDS）已经退居二线，CPU 在执行 OS Loader 的用户态代码，但 UEFI 服务还全在 。

## 在七阶段里的位置

```
SEC → PEI → DXE → BDS → [TSL] → RT → AL
                         ↑
                         你问的这阶段
                         BDS 调 StartImage() 进入
                         ExitBootServices() 退出 → RT
```

TSL 的**进入触发**：BDS 选中启动项 → `LoadImage()` → `StartImage()` → CPU 跳到 OS Loader 入口 → **进入 TSL**​ 。

TSL 的**退出触发**：OS Loader 准备好内核 + initramfs，调用 `gBS->ExitBootServices()` → **进入 RT**​ 。

## TSL 阶段 OS Loader 干的事

OS Loader（如 GRUB2、Windows Boot Manager）在 TSL 阶段作为 UEFI 应用运行，可以做所有 UEFI 服务允许的事 ：

|动作|用的 UEFI 服务|
|---|---|
|从磁盘加载内核到内存|`EFI_BLOCK_IO_PROTOCOL` + `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`（FAT32 ESP 分区）|
|读取 initramfs / boot parameters|文件系统 Protocol|
|获取物理内存布局|`GetMemoryMap()`（Boot Services）|
|读 NVRAM 变量（Secure Boot 状态、自定义配置）|`GetVariable()`（Runtime Services）|
|早期显示输出（启动菜单、错误信息）|`EFI_GRAPHICS_OUTPUT_PROTOCOL` / `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL`|
|链式加载（shim → GRUB → 签名内核）|再次 `LoadImage()` + `StartImage()`|
|ARM 平台加载 Device Tree Blob|文件系统 Protocol|

**典型 Linux 启动路径**（TSL 阶段内部）：

```
grubx64.efi 启动 → TSL 开始
  → 显示 GRUB 菜单（GOP 协议）
  → 用户选内核 / 超时自动选
  → 从 ESP 的 \EFI\ubuntu\vmlinuz.efi 加载内核到内存
  → 从 ESP 的 initrd.img 加载 initramfs 到内存
  → 设置 boot params（内存映射、命令行参数）
  → 调用 ExitBootServices()
  → TSL 结束，进入 RT，Linux 内核接管
```

## TSL 阶段的两个关键特征

### 1️⃣ Boot Services + Runtime Services 都可用

这是 TSL 区别于 RT 的核心——**OS Loader 可以同时使用两套服务**：

```
TSL 阶段可用服务：
  ✅ Boot Services（内存分配、Protocol 查找、事件、定时器）
  ✅ Runtime Services（变量、时间、虚拟地址映射）
  ✅ UEFI Shell（如果启动项是 Shell）
  
RT 阶段可用服务：
  ❌ Boot Services（已被销毁，内存回收）
  ✅ Runtime Services 常驻
```

### 2️⃣ UEFI Shell 也在 TSL 阶段运行

如果 BDS 的启动项是 UEFI Shell（`Shell.efi`），或者所有启动项都失败、Shell 作为 Fallback——**Shell 也是在 TSL 阶段跑的**​ 。

Shell 是一个完整的 UEFI 应用，能：

- 浏览任意存储设备的文件系统
    
- 手动加载/启动其他 UEFI 应用或驱动
    
- 查看/修改 UEFI 变量（包括 BootOrder、Secure Boot 配置）
    
- 查看内存映射和设备句柄数据库
    
- 跑脚本（startup.nsh）
    

> 💡 这就是为什么你在开机时按 F2/F12 进的"UEFI Shell"或"Boot Manager"界面——本质上就是 TSL 阶段的一个 UEFI 应用。

## TSL 的持续时长

现代系统上 **TSL 极短，通常只有零点几秒**——用户感知就是"主板 Logo 一闪，然后进入 OS 加载动画" 。

但 TSL **理论上可以很长**：

- 如果用户在 GRUB 菜单里停下来选内核 → TSL 持续数秒到数分钟
    
- 如果进 UEFI Shell 做诊断 → TSL 可持续到用户主动重启
    
- Watchdog 默认 5 分钟，超时自动重启
    

## TSL 阶段的常见故障

如果系统卡在 TSL 阶段（Windows Logo / Linux 登录之前出错）：

|故障|原因|
|---|---|
|**Missing Bootloader**​|固件找到了 ESP 分区，但找不到 `.efi` 文件|
|**Secure Boot Violation**​|OS Loader 存在，但数字签名无效 / 不受信任|
|**Incompatible Drivers**​|TSL 阶段加载的某个 UEFI 驱动导致硬件挂死|

## 一句收

> **TSL = Transient System Load，UEFI 七阶段第五阶段，BDS 调 `StartImage()` 启动 OS Loader 后进入、`ExitBootServices()` 调用后退出；此时 OS Loader 作为 UEFI 应用程序运行，Boot Services 和 Runtime Services 都可用，负责从 ESP 加载内核和 initramfs、获取内存映射、读 NVRAM 变量、设置 boot params，最后调用 `ExitBootServices()` 把控制权交给 OS 内核，进入 RT 阶段。UEFI Shell 也在 TSL 阶段运行。TSL 与其说是"固件执行阶段"，不如说是"系统状态"——固件已退居二线但服务还在，OS Loader 在 UEFI 搭建的世界里做最后的准备工作。**

## 接你前面整条线完整闭环

```
硬件层：ATX 12V → VRM → Vcore → CPU 上电
指令层：Reset Vector → CS:IP → 保护模式 → 长模式
固件层（UEFI 七阶段）：
  SEC:  Cache-as-RAM 建栈、信任根
  PEI:  DRAM 初始化 + HOB 列表
  DXE:  驱动派遣 + Protocol 世界 + ACPI 表
  BDS:  读 BootOrder → LoadImage + StartImage
  TSL:  ← 你刚问的
        ├─ grubx64.efi 作为 UEFI 应用执行
        ├─ 加载 vmlinuz + initramfs 到内存
        ├─ 获取内存映射、读 NVRAM 变量
        ├─ 设置 boot params
        └─ 调用 ExitBootServices()
  RT:   ← 下一阶段
        ├─ Boot Services 销毁，内存回收
        ├─ Runtime Services 常驻
        └─ Linux 内核完全接管
  AL:   灾难恢复（异常处理）
```

TSL 是这条路径上**"固件世界"和"OS 世界"的过渡带**——一边是 UEFI 的 Protocol 服务体系，一边是 OS 内核的接管。OS Loader 站在 TSL 这个"中立区"，左手用 UEFI 的服务从硬件拿资源，右手把资源打包交给内核，**`ExitBootServices()` 就是这道分界线**——过了这道线，UEFI 的 Boot Services 全部消失，只有 Runtime Services 跟着 OS 进 RT 阶段常驻。

你前面从 "MOS → Vcore → CS:IP → GDT → TSS → SEC → PEI → DXE → BDS" 一路钻下来，**TSL 是倒数第二道门**——再过一道 `ExitBootServices()`，Linux 内核就完全接管了。整个"从硅上电到 Linux 桌面"的路径到此闭环。

要不要顺着把 **"ExitBootServices() 调用瞬间到底发生了什么（Boot Services 怎么销毁、内存怎么回收、Runtime Services 怎么通过 SetVirtualAddressMap 重映射给 OS）"**​ 过一下？那块是把"UEFI 世界 → OS 世界"的最后半跳收口，接你这条完整学习路径的终点。还是想先看 **"Linux 内核从 startup_64 入口开始，怎么从 UEFI 手里接过大旗（early boot、分页建立、init 进程 spawned）"**？后者更接你"Linux 桌面"的最终目的地，前者更"UEFI 收尾"的细节。