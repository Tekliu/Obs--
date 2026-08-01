SEC = Security Phase（安全阶段）——**UEFI/PI 规范定义的开机后第一个执行的固件阶段**，CPU 从 **Reset Vector** 出来第一句代码就是它。它的使命用一句话概括："在 DRAM (DDR5)还没好的极端环境里，建立临时执行环境（Cache-as-RAM），作为信任链的根，然后把控制权交给 [[PEI]]"。

它在 UEFI 七阶段里的位置

上电/复位 → [SEC] → PEI → DXE → BDS → TSL → RT → AL
            ↑
            
            CPU 复位后执行的第一段代码

SEC 是 PI 规范里第一个 phase，所有后续阶段的根基。

SEC 的四大职责（PI 规范原文）

根据 PI 规范，SEC 必须完成这四件事：

1. 处理所有平台重启事件：冷启动、热重启、S3 唤醒、各类异常重启
2. 建立临时内存（Temporary Memory Store）：在 DRAM 未初始化前，用 CPU 内部资源（最常见是 Cache）当临时 RAM
3. 作为系统信任根（Root of Trust）：整个信任链的起点，SEC 可以验证 [[PEI]] 的完整性后再交出控制权
4. 传递交接信息给 PEI Foundation：通过 
"EFI_SEC_PEI_HAND_OFF" 结构体传参

核心矛盾：DRAM 没好，代码怎么跑？

这是 SEC 最特别的地方——它是 UEFI 七阶段里最"裸"的一个：

上电瞬间：
  CPU 复位 → 从 Reset Vector (x86: 0xFFFFFFF0) 取第一条指令
  → 此时状态：
    ✅ CPU 自己活着（寄存器、Cache 可用）
    ❌ DRAM 没初始化（内存控制器还没配置）
    ❌ 没有栈（ESP/RSP 没法用）
    ❌ 没有全局变量区
    ❌ 中断不可用（IDT 未建）
    ❌ 代码通常在 SPI Flash 里 XIP（原地执行）

没有栈 = 不能调用函数 = 不能写 C 代码——所以 SEC 早期必须是纯汇编。

解法：Cache-as-RAM（CAR）

PI 规范原文："This temporary memory store can include but is not limited to programming the processor cache to behave as a linear store of memory. This cache behavior is referred to as 'no evictions mode'" —— 把 CPU 的 Cache 配成 no-eviction（不淘汰）模式，当 [[RAM]] 用。

工作原理

正常 Cache 模式：
  读命中 → 返回 Cache 数据
  读缺失 → 去主存取，填入 Cache（淘汰一行）
  写命中 → 写 Cache
  写缺失 → 去主存取，写入 Cache

no-eviction 模式（CAR）：
  读命中 → 返回 Cache 数据
  读缺失 → 不向主存发请求！直接当"命中"处理（因为背后没 DRAM）
  写命中 → 写 Cache
  写缺失 → 不向主存发请求！直接在 Cache 里分配一行

关键：背后根本没有 DRAM 可以 evict(回避/避免) 到，所以所有访问都被强制当作"命中"，Cache 变成了一个纯 SRAM 式的临时内存。

建立 CAR 的步骤（x86）

1. 进入 SEC 汇编入口（Reset Vector 之后的早期代码）
2. 刷 Cache，确保干净状态
3. 切模式：[[实模式]] → 保护模式（32-bit flat） → （64位系统）[[长模式]]
4. 配置 [[MTRR]]（Memory Type Range Registers）
   → 把一段物理地址范围标记为 WB（Write-Back）类型
5. 通过特定序列激活 No-Eviction Mode（[[NEM]]）
   → 此时 CPU 的 L1 Cache 被"冻结"成 [[RAM]]
6. 在 CAR 区域里建立栈（设 ESP/RSP）
7. 此时 C 语言环境可用 → 跳入 SEC 入口函数

💡 这就是为什么 SEC 通常分两段：Reset Vector 阶段（纯汇编，CAR 未建好前） + SEC 功能阶段（CAR 建好后，C 代码可跑）。前者用 [[jmp]] 模拟调用（因为没有栈），后者才正常调函数。
SEC 的执行流程（Mermaid）

graph TD
    A[CPU 复位] --> B["Reset Vector<br/>(x86: 0xFFFFFFF0)"]
    B --> C[纯汇编：刷 Cache]
    C --> D[切到保护模式/长模式]
    D --> E[配置 MTRR]
    E --> F[激活 No-Eviction Mode<br/>Cache 变 CAR]
    F --> G[在 CAR 里建栈 ESP/RSP]
    G --> H["跳入 SEC 入口函数<br/>(C 代码可跑)"]
    H --> I[微码更新]
    I --> J[早期 CPU/芯片组初始化]
    J --> K[定位 BFV 里的 PEI Core]
    K --> L{验证 PEI 完整性?}
    L -->|通过| M[填充 EFI_SEC_PEI_HAND_OFF]
    L -->|失败| Z[停机/告警]
    M --> N["调用 PeiCoreEntry()<br/>交出控制权"]
    
    style F fill:#bfb,stroke:#333
    style N fill:#bbf,stroke:#333

SEC → PEI 的交接数据

SEC 最后通过 
"EFI_SEC_PEI_HAND_OFF" 结构体把这些信息传给 
"PeiCoreEntry()"：

typedef struct {
    UINT16  DataSize;                       // 结构体大小
    VOID    *BootFirmwareVolumeBase;        // BFV 基地址（PEI Core 所在）
    UINT32  BootFirmwareVolumeSize;         // BFV 大小
    VOID    *TemporaryRamBase;             // CAR 临时 RAM 基地址
    UINT32  TemporaryRamSize;              // CAR 临时 RAM 大小
    VOID    *PeiTemporaryRamBase;          // PEI 阶段可用临时 RAM
    UINT32  PeiTemporaryRamSize;
    VOID    *StackBase;                    // 栈基地址
    UINT32  StackSize;                     // 栈大小
    // ... 还有其他字段
} EFI_SEC_PEI_HAND_OFF;

PEI Foundation 入口函数签名：

void __cdecl PeiMain(
    int SecCoreData,                    // 指向 EFI_SEC_PEI_HAND_OFF
    EFI_PEI_PPI_DESCRIPTION *PpiList    // 初始 [[PPI 列表]]
);

从此 SEC 退出，PEI 接管——直到下一次系统复位才会再回来。

SEC 作为信任根（Security 的真意）

"Security" 不是指 SEC 阶段本身做加密，而是指 它是整个信任链的锚点：

硬件信任根 (HRoT)
  → 验证 SEC 代码（通常由 Boot Guard / PSP 在 CPU 内部做）
  → SEC 验证 PEI Core 完整性（Hash/签名校验）
    → PEI 验证 [[DXE]] Core 和各 DXE 驱动
      → DXE 验证 OS Loader
        → OS Loader 验证内核

在支持 [[TPM]] 的平台，SEC 阶段作为 SRTM（Static Root of Trust for Measurement）/ CRTM（Core Root of Trust for Measurement） 的起点：

- SEC 度量 PEI Core → Extend 到 PCR[0]
- PEI 度量 DXE Core → 继续 Extend 到 PCR[0]
- 形成一条"只能追加、不能篡改"的信任链日志

⚠️ Intel 平台叫 Boot Guard，用芯片组熔丝里的密钥验证 SEC 代码；AMD 平台叫 Platform Secure Boot，由 PSP（Platform Security Processor，一个嵌入式 ARM 核）在 main CPU 释放前先验证初始固件。
架构差异（x86 vs ARM）

不同架构的 SEC 实现差别很大：

 x86 ARM
入口 Reset Vector 
"0xFFFFFFF0" 架构定义的异常向量，很多 SoC 先跑片内 Boot ROM
临时内存 Cache-as-RAM（no-eviction 模式） 片上 SRAM（64-256KB，复位即可用）
初始模式 16-bit 实模式 → 保护模式 → 长模式 通常 EL3（ARMv8-A）
信任根 Boot Guard / ACM PSP / TrustZone

ARM 平台一般不用 CAR 技巧——因为大多数 ARM SoC 自带一小块 on-chip SRAM，复位即可用，直接当栈就行。x86 没这块 SRAM，所以发明了 CAR 这招。

一句收

SEC = Security Phase，UEFI 七阶段的第一阶段，CPU 从 Reset Vector 出来执行的第一段代码；它在 DRAM 未初始化的极端环境下，通过 Cache-as-RAM（no-eviction 模式）把 L1 Cache 当临时内存用，建立栈和 C 语言执行环境，作为整个信任链的根（可验证 PEI 完整性，是 SRTM/CRTM 起点），最后通过 
"EFI_SEC_PEI_HAND_OFF" 结构体把临时 RAM 地址/大小、栈地址/大小、[[BFV]] 位置