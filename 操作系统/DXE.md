DXE = Driver Execution Environment（驱动执行环境）——UEFI 七阶段里的第三阶段，也是整个 UEFI 的"小型操作系统内核"。PEI 把 DRAM 初始化好、打好 HOB 包交给它之后，DXE 在一套完整可用的内存环境里：**建服务表、派遣驱动、组装**   Protocol 数据库，把裸硬件组织成一个"可被 [[BDS]] 使用、可被 [[OS Loader]] 接管"的标准预启动环境。

为什么 DXE 是"UEFI 的真正起点"

PI 规范原文有个关键声明："The DXE Foundation is designed to be completely portable with no processor, chipset, or platform dependencies... the DXE Foundation does not depend on any services from a previous phase, so all the prior phases can be unloaded once the HOB list is passed"。

翻译过来：DXE Foundation 不依赖任何前一阶段的运行服务，HOB 列表一到手，SEC 和 PEI 就可以完全卸载。这意味着：

- SEC/PEI 是"一次性引导代码"——用完即扔
- DXE 才是 UEFI 真正"站稳脚跟"的起点——它只依赖 HOB 这份静态数据，之后完全自洽

💡 这就是为什么规范说"DXE 阶段不需要 PEI 阶段执行，唯一要求是有一个有效的 HOB 列表"——理论上你可以写一个完全不同的前期固件，只要最后给它一份合规的 HOB 列表，DXE 照样能跑起来。
DXE 阶段的资源状态（与前两阶段对比）

资源 SEC PEI DXE
主存（DRAM） ❌ ❌→✅ 后期 ✅ 完全可用
执行环境 纯汇编→CAR 临时栈 C + CAR 临时 RAM 完整 C 环境：堆/栈/全局变量
代码位置 SPI Flash XIP SPI Flash XIP 可加载到任意物理内存
驱动规模 无 PEIM（轻量） DXE 驱动（完整 PE/COFF，可较大）
服务 无 PEI Services Boot/Runtime/DXE Services 全套

DXE Foundation 不含硬编码地址——"can be loaded anywhere in physical memory, and it can function correctly no matter where physical memory or where Firmware Volumes are located"。这种位置无关性让 DXE 真正像个"微型内核"。

DXE 内部四大组件

graph TD
    HOB[HOB 列表<br/>PEI 传入，只读] --> FOUNDATION
    
    subgraph DXE["DXE Foundation（DXE Core / DxeCore）"]
        FOUNDATION["1. 消费 HOB<br/>重建内存视图"] 
        FOUNDATION --> SYSTEM["2. 建立 UEFI System Table<br/>+ Boot Services<br/>+ Runtime Services<br/>+ DXE Services"]
        SYSTEM --> GCD["3. 初始化 GCD<br/>Global Coherency Domain<br/>物理内存/MMIO/IO 资源地图"]
        GCD --> HANDLED["4. 初始化 Handle 数据库<br/>+ Protocol 数据库"]
        HANDLED --> DISPATCH["5. 进入 DXE Dispatcher 循环"]
    end
    
    DISPATCH -->|按 DEPEX 派遣| EARLY["Early DXE 驱动<br/>产生 Architectural Protocols"]
    DISPATCH -->|按 DEPEX 派遣| LATE["Late DXE 驱动<br/>UEFI Driver Model"]
    
    EARLY --> PROTOCOL["Protocol 数据库<br/>Handle + Protocol 模型"]
    LATE --> PROTOCOL
    
    PROTOCOL --> BDS["触发 BDS Architectural Protocol<br/>→ 进入 BDS 阶段"]
    
    style FOUNDATION fill:#bfb,stroke:#333
    style DISPATCH fill:#bbf,stroke:#333
    style PROTOCOL fill:#ffb,stroke:#333
    style BDS fill:#fbb,stroke:#333

1️⃣ 消费 HOB 列表（只读）

DXE Foundation 入口收到的第一个东西就是 HOB 链表指针。HOB 里包含：

- 启动模式（冷启动 / S3 唤醒 / Recovery）
- 系统内存描述（PEI 初始化的物理内存范围）
- 固件设备位置（FV 在物理地址空间的位置）
- I/O 与 MMIO 资源描述

⚠️ 规范强调：HOB 列表对 DXE 是只读数据结构（read-only data structure），DXE Foundation 和 DXE 驱动绝不应该修改 HOB 内容。它是"PEI 留给 DXE 的快照"，改了就破坏了跨阶段契约。
2️⃣ 建立四张服务表

DXE Foundation 必须产出的核心服务：

🔹 UEFI System Table（系统表）

所有 DXE 驱动和 BDS 的"总入口"——通过它拿到一切服务：

EFI_SYSTEM_TABLE
├── Firmware Vendor / Version / Revision
├── Console In/Out/Err（控制台 Protocol 指针）
├── Boot Services 指针    ← 内存分配/事件/Protocol 处理/镜像加载
├── Runtime Services 指针  ← 变量/时间/复位（OS 运行期仍可用）
└── Configuration Table 指针
    ├── DXE Services Table 指针
    ├── HOB List 指针
    ├── ACPI Table 指针
    └── SMBIOS Table 指针

🔹 Boot Services（启动服务）

DXE/BDS 阶段最常用的接口：

- 内存管理：
"AllocatePool" / 
"AllocatePages" / 
"FreePages"
- Protocol 处理：
"InstallProtocolInterface" / 
"LocateProtocol" / 
"HandleProtocol"
- 事件/定时器：
"CreateEvent" / 
"SetTimer"（配合 TPL 任务优先级调度）
- 镜像加载：
"LoadImage" / 
"StartImage" / 
"ExitBootServices"
- 句柄操作：
"LocateHandle" / 
"ConnectController"

🔹 Runtime Services（运行时服务）

OS 启动后仍可调用（通过虚拟地址重映射）：

- NVRAM 变量：
"GetVariable" / 
"SetVariable"
- 系统时间：
"GetTime" / 
"SetTime"
- 系统复位：
"ResetSystem"
- 虚拟地址映射：
"SetVirtualAddressMap"（ExitBootServices 前调用）

🔹 DXE Services（DXE 专有服务）

比 Boot Services 更低级，主要管 GCD（Global Coherency Domain）：

- 内存空间管理：
"AddMemorySpace" / 
"AllocateMemorySpace"（添加/分配物理内存或 MMIO 范围）
- I/O 空间管理：
"AddIoSpace" / 
"AllocateIoSpace"
- 属性调整：
"SetMemorySpaceAttribute"（改 cacheability/访问权限）

3️⃣ GCD（Global Coherency Domain）—— 资源地图

GCD 是 DXE 内部的"物理资源数据库"，从 HOB 的 Resource Descriptor 重建而来，跟踪：

GCD 内存空间地图：
  [0x00000000 - 0x0007FFFF]  → EfiGcdMemoryTypeReserved（中断向量表）
  [0x00080000 - 0x7FFFFFFF]  → EfiGcdMemoryTypeMemory（系统内存）
  [0x80000000 - 0x9FFFFFFF]  → EfiGcdMemoryTypeMemoryMappedIo（PCIe MMIO）
  [0xA0000000 - 0xA00FFFFF]  → EfiGcdMemoryTypeReserved（TOLUD 以下）
  ...

GCD I/O 空间地图：
  [0x0000 - 0x0FFF]  → 可用 IO 端口
  [0x1000 - 0x10FF]  → 分配给 AHCI 控制器
  ...

所有 DXE 驱动通过 
"gDS->AddMemorySpace()" / 
"gDS->AllocateMemorySpace()" 来声明和占用资源。这就是为什么 DXE 能避免资源冲突——所有分配都在 GCD 里记账。

4️⃣ Handle + Protocol 数据库 —— DXE 的"对象模型"

这是 DXE 最优雅的设计。UEFI 用 Handle（对象句柄）+ Protocol（接口） 模型组织整个世界：

Handle Database（句柄数据库）
┌─────────────────────────────────────────┐
│ Handle 1 (代表: 主板设备树根)           
│   ├── Protocol: 