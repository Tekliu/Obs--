ACPI 表 = 主板固件（BIOS/UEFI）在开机早期塞进内存的一组"硬件说明书"数据结构，操作系统启动时来读，读完就知道"这台机器有哪些设备、怎么进睡眠、电源按钮按了咋办、CPU 怎么降频"——ACPI 的全称 Advanced Configuration and Power Interface（高级配置与电源接口）。

💡 关键认知：ACPI 表不是"一张表"，是一整套表——像一本手册，有目录、有正文、有附录。OS 从"目录的目录"开始翻，一路翻到具体设备的控制方法。

表体系结构（Mermaid 大图）

graph TD
    FW[BIOS/UEFI 固件<br/>开机早期把表放进内存] --> RSDP
    
    RSDP[RSDP<br/>Root System Description Pointer<br/>"RSD PTR" 魔数字符串] --> XSDT
    RSDP --> RSDT
    
    XSDT[XSDT<br/>64位系统用<br/>指向其他所有表] --> FADT
    XSDT --> MADT
    XSDT --> DSDT
    XSDT --> SSDT
    XSDT --> HPET
    XSDT --> MCFG
    XSDT --> BERT
    
    RSDT[RSDT<br/>32位兼容用<br/>指向其他所有表] --> FADT
    RSDT --> MADT
    RSDT --> DSDT
    
    FADT[FADT<br/>Fixed ACPI Description Table<br/>固定硬件寄存器地址] --> DSDT2[DSDT<br/>不同iated System Description Table<br/>主板设备拓扑 + AML 字节码]
    FADT --> FACS[FACS<br/>固件ACPI控制结构]
    FADT --> SSDT2[SSDT<br/>Secondary System Description Table<br/>DSDT的补充/覆盖]
    
    style RSDP fill:#f9f,stroke:#333
    style XSDT fill:#bbf,stroke:#333
    style FADT fill:#bfb,stroke:#333
    style DSDT2 fill:#fbb,stroke:#333

OS 的查找路径（ACPI 规范规定）：

1. 固件把 [[RSDP ]]放在内存固定区域
   - Legacy BIOS: 放在 [[EBDA]] 或 0xE0000-0xFFFFF 区间
   - UEFI: 通过 [[EFI 系统表]]找
1. OS 搜 "RSD PTR" 魔数字符串 → 找到 RSDP
2. RSDP 里取出 RSDT 或 XSDT 的物理地址（64位系统优先 XSDT）
3. 遍历 XSDT/RSDT 里的指针数组 → 拿到所有其他表的地址
4. 按签名（signature）识别每张表 → 解析

核心表清单（必会那几张）

表名 签名 干啥的
RSDP 
"RSD PTR " 入口指针，定位 RSDT/XSDT
RSDT 
"RSDT" 32 位根表，指向其他表的 32 位地址数组
XSDT 
"XSDT" 64 位根表，指向其他表的 64 位地址数组，现代系统用这个
FADT 
"FACP" 固定 ACPI 硬件寄存器地址（PM1a_EVT_BLK、PM_TMR_BLK 等）、睡眠状态控制、指向 DSDT
DSDT 
"DSDT" ACPI 的心脏——主板设备拓扑 + AML 字节码（CPU、芯片组、EC、电源按钮、LID 等）
SSDT 
"SSDT" DSDT 的补充/覆盖，OEM 用来加自定义设备（独显、特殊按钮）、CPU 频率策略
MADT 
"APIC" 多处理器 APIC 描述，SMP 中断路由（Linux 启动多核必读）
FACS 
"FACS" 固件 ACPI 控制结构，休眠/唤醒时 OS 和固件交换数据的区域
HPET 
"HPET" 高精度定时器描述
MCFG 
"MCFG" PCIe 配置空间 MMIO 基地址
BERT 
"BERT" 启动错误记录（故障排查）
TPM2 
"TPM2" 可信平台模块描述
SRAT 
"SRAT" NUMA 拓扑（哪个 CPU 靠近哪块内存）
SLIT 
"SLIT" NUMA 距离矩阵

DSDT 里装的"AML"是啥——ACPI 最骚的设计

DSDT（和 SSDT）里不是死数据，是可执行的字节码——叫 AML（ACPI Machine Language）。

AML 是个图灵完备的小虚拟机语言，有：

- 控制流：
"If/Else/While"
- 数据结构：
"Package/Buffer/Field"
- 方法调用：
"Method"
- 硬件访问：
"OperationRegion + Field"（直接读写 IO/MMIO 寄存器）
- 事件触发：
"Notify"

举例——电源按钮按下时 OS 该干啥，写在 DSDT 里：

Method(_PWR) {    // 电源按钮的 _PWR 方法
    If (LEqual(\_SST, 0)) {
        Sleep(100)        // 防抖
        Notify(\_SB.PWR, 0x80)  // 通知 OS: 电源按钮事件
    }
}

OS 的 ACPI 子系统（Linux 的 
"drivers/acpi/"、Windows 的 
"ACPI.sys"）里有个 AML 解释器，运行时去执行这些 Method。

💡 这就是 ACPI 的精髓：固件不只给"数据"，还给"行为"。主板厂商把"这台机器特有的电源逻辑、热管理逻辑"写成 AML 塞进 DSDT，OS 不用为每块主板写驱动，只要有个 AML 解释器就能跑。
ACPI 表管的事（OS 拿来干啥）

1. 设备枚举：通过 
"_HID"/
"_CID"/
"_ADR" 识别设备，不用 OS 自己探测
2. 电源状态：
   - 全局状态：S0（工作）→ S1/S2/S3（睡眠）→ S4（休眠）→ S5（软关机）
   - 设备状态：D0（全速）→ D3hot/D3cold（断电）
3. 性能调节：P-state（Performance state）通过 
"_PSS"/
"_PPC" 方法，Intel 叫 SpeedStep、AMD 叫 Cool'n'Quiet
4. 热管理：
"_TMP" 读温度、
"_CRT" 临界温度触发关机、
"_HOT" 热限触发降频
5. 事件处理：电源按钮 
"_PWRS"、合盖 
"_LID"、电池 
"_BIF"/
"_BST"
6. 多核启动：MADT 告诉 OS 有几个 CPU、APIC 怎么路由中断
7. NUMA 拓扑：SRAT/SLIT 告诉 OS 内存和 CPU 的亲近关系

为啥要有 ACPI 表——对比老 APM

 APM（传统） ACPI（现代）
控制权 固件主导（BIOS 控制电源） OS 主导（OS 按策略控制）
灵活性 固定策略 动态策略，OS 可调
设备支持 仅 CPU 等少数设备 全系统设备
事件响应 被动（按键触发） 主动（温度/电量监控）

ACPI 的本质创新：把电源管理的控制权从固件移交给 OS。固件只负责"描述硬件"（通过 ACPI 表），OS 负责"决策和控制"（通过 AML 解释器执行这些描述）。

所以 ACPI 表 = 固件给 OS 的"硬件填空题"：厂商按规范把自家硬件信息填进对应表格，OS 知道每张表的格式，读出来就能用——实现"硬件描述"与"OS 驱动"的解耦。

接你前面那条线

你前面从 "MOS → CPU 取指 → 保护模式 → GDT/TSS → 多核启动" 一路钻下来——ACPI 表就是"多核启动之后，OS 接管硬件"那一步的关键输入：

开机 → UEFI/BIOS 固件
  → 固件建 ACPI 表，塞进内存
  → 把 RSDP 放在固定位置
  → 跳入 OS 引导程序（如 Linux 的 head_64.S）
  → OS 早期启动代码搜 "RSD PTR" → 找到 RSDP
  → 遍历 XSDT → 解析 FADT/DSDT/MADT/SSDT...
  → 构建 ACPI 命名空间（AML 解释器执行 DSDT 里的 Method）
  → 枚举设备、初始化电源管理、启动 AP 核（MADT）
  → 系统进入 S0 工作状态

所以你按电源键到桌面出现这几秒里，ACPI 表一直在使用——CPU 降频、风扇调速、USB 热插拔、合盖睡眠，背后全是 OS 在查 ACPI 表 + 执行 AML 方法。

一句收：ACPI 表 = BIOS/UEFI 开机早期放进内存的一组硬件描述数据结构，RSDP 是入口，XSDT/RSDT 是指向其他表的目录，FADT 定义固定硬件寄存器，DSDT 是心脏（含 AML 字节码描述主板设备行为），SSDT/MADT/SRAT 等是补充；OS 解析这些表来枚举设备、管电源/睡眠/热/性能，本质是把电源管理控制权从固件移交给 OS 的标准化机制。

要不要顺着把 "Linux 内核怎么从 RSDP 开始解析 ACPI 表、AML 解释器怎么工作" 过一下？那块接你"内核启动 → 多核 → 设备驱动"那条线，是 OS 底层岗/驱动岗的高频考点。还是想先看 "ACPI 定义的 S0-S5 睡眠状态和 D0-D3 设备状态具体怎么切换（谁触发、硬件怎么动作）"？后者更接你日常"合盖睡眠/关机"的直觉。