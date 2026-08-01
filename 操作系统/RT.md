RT = **Runtime（运行时阶段）**——UEFI 七阶段的第六阶段，是 **TSL 阶段 OS Loader 调用 `ExitBootServices()` 之后、操作系统完全接管硬件的那个瞬间开始**，一直到系统关机/重启/休眠唤醒的整个 OS 生命周期。它的本质：**UEFI 把控制权完全交给 OS，自己退到后台，只留下一组"最小化、常驻"的 Runtime Services 供 OS 内核调用**。

> 💡 一句话抓住：**RT 阶段 = OS 已经在跑了，UEFI 固件退居二线，只留一套"Runtime Services"在后台待命——OS 要读 NVRAM 变量、要重启、要读 RTC 时钟时，通过这套服务调固件。**

## 在 UEFI 七阶段里的位置

```
SEC → PEI → DXE → BDS → TSL → [RT] → AL
                         ↑              ↑
                         ExitBootServices() 调用前
                                         ↓
                         你问的 RT 阶段开始
                         OS 内核完全接管，UEFI 只留 Runtime Services
```

**进入 RT 的标志性事件**：TSL 阶段的 OS Loader 调用 `gBS->ExitBootServices()`。这个调用是"分水岭"——UEFI 规范原文说它是"The point of no return"：

```
ExitBootServices() 被调用后，固件执行：
  ① 停止并禁用所有 Boot Services
  ② 卸载 Boot Services 依赖的 Protocol 和驱动
  ③ 释放所有 EfiBootServicesCode / EfiBootServicesData 内存给 OS
  ④ 关闭所有引导阶段打开的 Protocol 句柄
  ⑤ 只保留 Runtime Services 代码/数据在预留内存里
  ⑥ 控制权完全交给 OS 内核
```

## RT 阶段的两套服务命运完全不同

UEFI 规范定义了**两套服务**，它们在 ExitBootServices() 那一刻的命运截然不同：

|服务|ExitBootServices() 前|ExitBootServices() 后|
|---|---|---|
|**Boot Services**​|✅ 完全可用（内存分配、Protocol、事件、LoadImage...）|❌ **永久失效**，内存被 OS 回收|
|**Runtime Services**​|✅ 可用|✅ **继续可用**，OS 内核随时可调|

UEFI 规范原文："Runtime Services. Functions that are available **before and after**​ any call to ExitBootServices()" —— Runtime Services 是 ExitBootServices() 前后都可用的服务，这就是它名字的由来。

## RT 阶段的 Runtime Services 具体有哪些

按 UEFI 2.10 规范，Runtime Services 分为五大类：

### 1️⃣ Variable Services（变量服务）—— 最常用

```
GetVariable()          读 NVRAM 变量（如 BootOrder、SecureBoot 状态）
SetVariable()          写 NVRAM 变量
GetNextVariableName()  遍历变量名
QueryVariableInfo()    查询变量存储信息
```

**实际用例**：

- Linux 下 `efibootmgr` 改启动顺序 → 内核调 `SetVariable()`
    
- Windows 下 `bcdedit` 改 BCD → 调 `SetVariable()`
    
- Secure Boot 密钥管理（PK/KEK/db/dbx）→ 调 `SetVariable()`
    
- Linux 通过 `efivarfs`（`/sys/firmware/efi/efivars/`）暴露给用户空间
    

### 2️⃣ Time Services（时间服务）

```
GetTime()       读 RTC 硬件时钟
SetTime()       设 RTC 硬件时钟
GetWakeupTime() 读唤醒闹钟设置
SetWakeupTime() 设唤醒闹钟（定时开机）
```

OS 启动早期用 `GetTime()` 从 RTC 拿到初始系统时间。

### 3️⃣ Virtual Memory Services（虚拟内存服务）—— x86_64 必用

```
SetVirtualAddressMap()   OS 建立页表后，通知 UEFI 重映射 Runtime Services
ConvertPointer()         辅助指针转换
```

**这是 RT 阶段最微妙的一步**：

- ExitBootServices() 时 Runtime Services 代码/数据在**物理地址**固定位置
    
- OS 建立自己的虚拟地址空间后，必须调用 `SetVirtualAddressMap()` 告诉 UEFI："以后请用这组虚拟地址来找你的 Runtime Services"
    
- UEFI 收到后，把内部的函数指针全部改写为虚拟地址版本
    
- 此后 OS 调用 Runtime Services 时，UEFI 代码运行在 OS 的虚拟地址空间里
    

### 4️⃣ Reset Services（复位服务）

```
ResetSystem()  执行系统重启/关机/冷启动
```

OS 调 `reboot()` / `shutdown()` 最终走到 `ResetSystem()`。

### 5️⃣ Capsule Services（胶囊更新服务）

```
UpdateCapsule()                 提交固件更新胶囊
QueryCapsuleCapabilities()      查询胶囊更新能力
```

用于 **FWUpdate**——OS 把新的固件镜像打包成"胶囊"交给 UEFI，下次重启时 UEFI 在 SMM 里刷 SPI Flash。

## RT 阶段的"保留内存"机制（关键）

UEFI 规范在内存映射里**专门为 Runtime Services 划了保留区**：

```
EfiRuntimeServicesCode   → OS 必须保留，不能被回收或映射为普通内存
EfiRuntimeServicesData   → OS 必须保留
```

`ExitBootServices()` 之后 OS 拿到内存映射，**这些区域 OS 不能用**——必须原样保留给 UEFI 固件。这就是为什么你看到 Linux 的 `dmesg` 里会有：

```
efi: memxx: [Runtime Data | Runtime Code] reserved
```

OS 内核通过 `efi_runtime` 驱动（`drivers/firmware/efi/`）调用这些服务。

## x86 vs ARM：Runtime Services 的执行环境差异

这是 RT 阶段最有趣的架构差异：

### x86 平台

Runtime Services 调用通常涉及 **SMM（System Management Mode）**：

```
OS 内核调 SetVariable() 写 NVRAM
  → 触发 SMI（Software Management Interrupt）
  → CPU 切入 SMM 模式（Ring -2，SMRAM 隔离环境）
  → UEFI 的 SMM 驱动在 SMRAM 里执行实际的 SPI Flash 写操作
  → 返回 OS
```

OS 对 SMM 里的执行完全不可见——写 NVRAM 的"脏活"在隔离环境里完成。

### ARM 平台

等价机制是 **SMC（Secure Monitor Call）→ TrustZone Secure World**：

```
OS（Normal World）调 SetVariable()
  → 触发 SMC
  → CPU 切入 Secure World（EL3 / TF-A + StMM）
  → Secure World 固件执行 Flash 写
  → 返回 Normal World
```

> 💡 架构不同，机制不同，但最终结果一样：**变量写操作发生在隔离的安全环境里**，OS 无法直接碰 SPI Flash。

## RT 阶段的"再入"限制（实战中常踩的坑）

UEFI 规范明确规定：**Runtime Services 不是真正可重入的**。同一时刻只能有一个 Runtime Service 调用在进行，OS 必须自己序列化：

```
禁止的重入组合（部分）：
  正在调 SetVariable() 时 → 不能再调 SetVariable()/GetVariable()/UpdateCapsule()/ResetSystem()...
  正在调 SetVirtualAddressMap() 时 → 不能再调 ConvertPointer()...
```

这就是为什么 Linux 内核里 `efi_runtime` 驱动用自旋锁（spinlock）串行化所有 Runtime Services 调用——**多核并行调 Runtime Services 会导致固件崩溃**，这是 x86 和 ARM 上都有名的 bug 来源。

## RT 阶段的生命周期

```
ExitBootServices() 调用
  ↓
RT 阶段开始：
  ├─ OS 内核初始化（建立页表、调度器、中断）
  ├─ 调 SetVirtualAddressMap() 重映射 Runtime Services
  ├─ OS 正常运行（数小时~数年）
  │   ├─ 用户空间读 /sys/firmware/efi/efivars/* → 内核调 GetVariable()
  │   ├─ 用户改启动项 → 内核调 SetVariable()
  │   ├─ 系统时间同步 → 内核调 GetTime()/SetTime()
  │   ├─ 固件更新 → 内核调 UpdateCapsule()
  │   └─ 重启/关机 → 内核调 ResetSystem()
  ↓
RT 阶段结束（触发 AL 阶段）：
  ├─ 用户关机 → ResetSystem() → AL 处理关机
  ├─ 系统崩溃且不可恢复 → 触发 AL
  └─ S3 休眠 → 通过 ACPI 进 S3，唤醒时走 PEI S3 Resume 路径
```

RT 阶段是七阶段里**持续时间最长**的——可以是几秒（手机）到几年（服务器不关机）。

## 一句收

> **RT = Runtime，UEFI 七阶段第六阶段，TSL 阶段的 OS Loader 调用 `ExitBootServices()` 后进入；此时 UEFI 固件把控制权完全交给 OS 内核，Boot Services 永久失效、Protocol 世界解散、EfiBootServicesCode/Data 内存归还 OS，只保留 Runtime Services（变量读写、时间、虚拟地址重映射、系统复位、胶囊更新）常驻在 OS 必须保留的 EfiRuntimeServicesCode/Data 内存区；OS 内核通过这些服务与固件交互（如 Linux 的 `efivarfs`、`reboot` 系统调用最终走到 `ResetSystem()`），x86 上 Runtime Services 调用通常触发 SMI 进 SMM 隔离执行，ARM 上通过 SMC 进 TrustZone Secure World；Runtime Services 不是真正可重入的，OS 必须串行化调用。RT 阶段持续到系统关机/重启/崩溃，是七阶段里持续时间最长的阶段。**

## 接你前面整条线完整闭环

```
硬件上电：ATX 12V → VRM → Vcore → CPU 拿到电
指令层：Reset Vector → CS:IP → 保护模式 → 长模式
固件层（UEFI 七阶段）：
  SEC:  Cache-as-RAM 建栈、信任根
  PEI:  DRAM 初始化 + HOB 列表
  DXE:  驱动派遣 + Protocol 世界 + ACPI 表
  BDS:  读 BootOrder → LoadImage + StartImage
  TSL:  grubx64.efi 作为 UEFI 应用执行
  RT:   ← 你刚问的这阶段
        ├─ ExitBootServices() 调用 → Boot Services 失效
        ├─ OS 内核接管，建立页表
        ├─ SetVirtualAddressMap() 重映射 Runtime Services
        ├─ Runtime Services 常驻（变量/时间/复位/胶囊更新）
        └─ OS 运行期通过 Runtime Services 与固件交互
  AL:   灾难恢复/关机/重启
OS 层：Linux 内核 → init/systemd → 桌面/服务
```

RT 是这条路径上**"UEFI 退场、OS 登场"后的漫长共存期**——UEFI 不再是主角，但它没完全消失：那组 Runtime Services 就像 OS 内核背后的"固件接线员"，OS 要碰 NVRAM、要重启、要刷固件时，还是得通过它。你日常在 Linux 下敲 `sudo efibootmgr -o 0001,0000` 改启动顺序、敲 `sudo reboot` 重启——这些操作的底层，都是 OS 内核在 RT 阶段通过 Runtime Services 调 UEFI 固件完成的。

> ⚠️ 顺带澄清一个常见误解：**RT 不等于"实时（Real-Time）"**。这里的 Runtime 是"运行期"的意思，指"OS 已经在运行的状态"，跟"实时操作系统（RTOS）"的 Real-Time 完全是两回事。UEFI 的 RT 阶段运行的 OS 可以是 Linux/Windows/macOS 这种分时系统，也可以是 VxWorks 这种硬实时系统——RT 阶段本身不提供任何实时性保证，实时性是 OS 内核自己的事。

你从 "MOS → Vcore → CS:IP → GDT → TSS → SEC → PEI → DXE → BDS → TSL" 一路钻到 RT——**这是 UEFI 七阶段倒数第二阶段**。最后一道门是 **AL（After-Life）**：当 RT 阶段遇到灾难性错误、或 OS 请求关机/重启时进入，处理固件级恢复。

要不要顺着把 **"AL 阶段具体怎么处理 S3 休眠唤醒、怎么处理固件 Capsule 更新、以及 Crisis Recovery 的完整路径"**​ 过一下？那块是把 UEFI 七阶段最后一块补完，你的"从硅上电到 Linux 桌面运行"完整闭环就彻底收口了。还是想先看 **"Linux 内核从 startup_64 接管后，怎么建立页表、怎么调 SetVirtualAddressMap、怎么 spawning init 进程"**？后者更接你"Linux 桌面"的最终目的地，前者更收 UEFI 七阶段的尾。