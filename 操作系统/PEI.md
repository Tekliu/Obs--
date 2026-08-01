PEI = **Pre-EFI Initialization（EFI 预初始化阶段）**——UEFI/PI 规范定义的**开机后第二个执行的固件阶段**，SEC 之后、DXE 之前。它的使命用一句话概括：**"在 DRAM 还没初始化好的极端受限环境下，用最薄的代码把永久内存跑起来，然后通过 HOB 把硬件状态交给 DXX"**。

> 💡 一句话抓住本质：**PEI 是 UEFI 七阶段里"最薄、最受限、最依赖处理器架构"的阶段——它要在"没有内存"的环境里把内存造出来。**

## 在 UEFI 七阶段里的位置

```
上电/复位 → [SEC] → [PEI] → DXE → BDS → TSL → RT → AL
            信任根   ↑        驱动执行   启动选择  过渡加载  运行时
                   你问的这阶段
```

PI 规范原文："The Pre-EFI Initialization (PEI) phase... is invoked quite early in the boot flow. Specifically, after some preliminary processing in the Security (SEC) phase, any machine restart event will invoke the PEI phase"。

## PEI 面临的核心矛盾：没有内存，却要初始化内存

这是 PEI 最特别的地方，也是它整个设计围绕的核心：

```
PEI 初期状态：
  ✅ CPU 活着（SEC 已切到 flat 32 / 长模式，Ring 0）
  ✅ CPU 内部 Cache 可用
  ❌ DRAM 未初始化（内存控制器还没配置）
  ❌ 没有可用的系统 RAM
  ❌ 堆栈空间极小（通常只有几 KB ~ 几十 KB 的 CAR）
  ❌ 单线程、串行执行，无堆空间
```

**这个矛盾怎么破？**​ SEC 阶段已经用 **Cache-as-RAM（CAR）**​ 把 L1/L2 Cache 配成 no-eviction 模式当临时内存，PEI 接着用这块临时内存跑。

> 💡 CAR 空间有多小？规范强调 PEI "operates with the platform in a nascent state, leveraging only on-processor resources, such as the processor cache as a call stack"——通常只有 4KB~64KB，所以 PEI **代码必须尽量薄**。

## PEI 的四大设计目标（PI 规范原文）

1. **最小化代码占用**：PEI 是"thinnest amount of code"——任何复杂算法都应推迟到 DXE
    
2. **维护信任链**：PEI Foundation 及其 [[PEIM]] 都要经过验证（防篡改 + 认证）
    
3. **提供核心 PEI 模块**：PEI Foundation 在某处理器架构上相对稳定，但支持各厂商加插件
    
4. **允许早期初始化模块独立开发**：通过 PEIM 机制解耦
    

## PEI 内部的两大组件

### 1️⃣ PEI Foundation（PEI 核心）

**单一二进制可执行体**（文件类型 `EFI_FV_FILETYPE_PEI_CORE`），由 SEC 发现并验证后跳入。它干两件事：

- **PEI Dispatcher**：按依赖顺序派遣 PEIM
    
- **提供 PEI Services 给所有 PEIM 使用**
    

PI 规范定义的 **PEI Services 六大类**：

|服务类|干什么|
|---|---|
|**PPI Services**​|管理 PPI 数据库，让 PEIM 之间互相调用|
|**Boot Mode Services**​|管理启动模式（正常/S3/S5/Recovery...）|
|**HOB Services**​|创建 HOB（Hand-Off Block），传给下一阶段|
|**Firmware Volume Services**​|在 FV 里找 PEIM 和其他固件文件|
|**PEI Memory Services**​|永久内存发现前后的内存管理服务|
|**Status Code / Reset Services**​|状态码报告（如 0x80 端口）、系统复位|

### 2️⃣ PEIM（Pre-EFI Initialization Module）

**一个个独立的功能模块**，每个负责一小块硬件的早期初始化：

```
典型 PEIM 清单：
├── CPU PEIM          微码加载、CPU 早期配置
├── Chipset PEIM      芯片组早期初始化
├── Memory Init PEIM  ★最关键★ DRAM 训练、内存控制器初始化
├── Board PEIM        主板特定初始化
├── Recovery PEIM     Crisis Recovery 支持
└── S3 Resume PEIM    S3 唤醒快速路径
```

PEIM 派遣规则（PEI Dispatcher 算法）：

```
1. 扫描 BFV（Boot Firmware Volume）和其他 FV
2. 对每个 PEIM 评估：
   - 是否已执行过？
   - 文件格式正确？
   - 通过验证（验证码/校验和）？
   - DEPEX（依赖表达式）是否满足？
3. 满足全部条件 → 派遣执行
4. 循环直到没有新 PEIM 可派遣
```

## PEIM 之间怎么通信：PPI

**PPI = PEIM-to-PEIM Interface**——PEI 阶段内部的"服务接口"机制。

为什么需要它？因为每个 PEIM 独立开发，但相互之间有依赖：

- 内存初始化的 PEIM 跑完后，发布 `MEMORY_INIT_PPI`
    
- 其他 PEIM 通过 `LocatePpi(MEMORY_INIT_PPI_GUID)` 拿到这个接口
    

PPI 两种形式：

- **Services PPI**：PEIM 提供函数/数据给其他 PEIM 用（InstallPpi）
    
- **Notification PPI**：PEIM 注册回调，当某个 PPI 被安装时触发
    

```
Memory Init PEIM:
  InstallPpi(&gMemoryInitPpiGuid, MemoryInitStruct)  // 发布服务
  
Platform PEIM:
  LocatePpi(&gMemoryInitPpiGuid, ...)               // 查找服务
  → 拿到函数指针 → 调用
```

> 💡 PPI 之于 PEI，相当于 Protocol 之于 DXE——都是"模块间解耦的服务接口"思想。但 PPI 存在临时内存里，DXE 一开始 PPI 数据库就销毁；而 Protocol 存在永久内存里，可贯穿到 OS。

## PEI 的两个子阶段（Pre-Memory → Post-Memory）

PEI 内部按"永久内存好不好"分为两段：

### Pre-Memory PEI（永久内存未发现前）

- 运行在 **CAR（Cache-as-RAM）**​ 里
    
- 堆栈极小（几 KB ~ 几十 KB）
    
- 派遣早期 PEIM（CPU、Chipset、Memory Init）
    
- **Memory Init PEIM 执行 DRAM 训练**
    

### Post-Memory PEI（永久内存可用后）

- DRAM 训练成功 → 永久内存可用
    
- **PEI Foundation 把所有结构从 CAR 迁移到永久内存**
    
- 重新派遣剩余 PEIM（依赖内存的那些）
    
- 构建 HOB 列表
    
- 通过 **DXE IPL PPI**​ 跳入 DXE
    

## PEI → DXE 的交接：HOB

**HOB = Hand-Off Block**——PEI 留给 DXE 的"遗产清单"。

HOB 里装啥：

- **PHIT HOB**：HOB 列表头（必需）
    
- **Resource Descriptor HOB**：物理内存范围描述
    
- **Firmware Volume HOB**：FV 位置（DXE 去哪找驱动）
    
- **Memory Allocation HOB**：栈/BSP 位置
    
- **GUID Extension HOB**：厂商自定义数据
    
- **END HOB**：HOB 列表尾（必需）
    

```
PEI 末期：
  构建 HOB 链表
  → 找到 DXE IPL PPI
  → 通过 DXE IPL 加载 DXE Foundation
  → 把 HOB 链表地址传给 DXE
  → 跳进 DXE 入口
  
DXE 起来后第一件事：遍历 HOB 链表，重建资源视图
```

> ⚠️ HOB 对 DXE 是**只读**的——PEI 写的快照，DXE 不能改，改了就破坏跨阶段契约。

## PEI 的额外职责：Crisis Recovery + S3 Resume

PI 规范特别强调 PEI **还负责两件事**：

1. **Crisis Recovery（灾难恢复）**：PEI 必须驻留在固件存储里"小而易损的块"，所以代码体积必须最小，且能通过"容错机制"更新——保证更新中途断电，旧 PEIM 或新 PEIM 至少有一个完整有效
    
2. **S3 Resume（从 S3 睡眠恢复）**：S3 唤醒时 PEI 要走最短路径快速恢复，速度至关重要
    

**这就是为什么 PEI 代码必须尽量小**——这两类场景都要求"代码路径最短"。

## 为什么 PEI "更像处理器的奴隶"

PI 规范原话："The implementation of the PEI phase is more dependent on the processor architecture than any other phase"。

意思是：PEI 极度依赖 CPU 架构——

- x86 上 SEC 要建立 CAR、切到保护模式、开 64-bit
    
- ARM 上 SEC 要切到 EL2/EL1、建立 MMU 雏形
    
- RISC-V 上 SEC 要配 CSR、建立 trap 向量
    

而 DXE 及以后就与架构无关了（都是 C 代码 + UEFI Protocol）。所以 **PEI 是 UEFI 里"最贴近硅"的阶段**。

## PEI 与前后阶段的关系

```
SEC 移交 PEI 的前提条件：
  1. 处理器执行模式已建立（x86: flat 32 模式，Ring 0）
  2. 能访问包含 PEI Foundation 的 Boot Firmware Volume (BFV)
  3. SEC 已通过验证 PEI Foundation 的签名（信任链延续）
  
PEI 移交 DXE 的条件：
  4. 永久内存已初始化
  5. HOB 列表已构建
  6. 通过 DXE IPL PPI 定位并加载 DXE Foundation
```

## 一句收

> **PEI = Pre-EFI Initialization，UEFI 七阶段第二阶段，SEC 用 Cache-as-RAM 建好临时栈后跳入；PEI Foundation（PEI Core）派遣一堆 PEIM 模块，在"DRAM 未初始化"的极端受限环境（CAR 临时内存只有几 KB~几十 KB、单线程串行）里，先初始化 DRAM、早期芯片组、CPU 微码，PEIM 之间通过 PPI 通信，阶段末期把硬件资源打包成 HOB 链表，通过 DXE IPL PPI 把控制权交给 DXE。PI 规范要求 PEI 代码尽量薄（"thinnest amount of code"），因为还要负责 Crisis Recovery 和快速 S3 Resume；PEI 是 UEFI 七阶段里最依赖处理器架构的阶段，复杂算法全推迟到 DXE。**

## 接你前面整条线完整闭环

```
硬件上电：
  ATX 12V → VRM → Vcore (~1V) → CPU 拿到电

指令层：
  Reset Vector (x86: 0xFFFFFFF0) → CS:IP 取第一条指令

固件层（UEFI 七阶段）：
  SEC:  Cache-as-RAM 建栈、信任根、验证 PEI Core
  PEI:  ← 你刚问的这阶段
        ├─ Pre-Memory PEI（CAR 环境）
        │   ├─ PEI Foundation 建立 PEI Services
        │   ├─ 派遣早期 PEIM（CPU/芯片组）
        │   └─ Memory Init PEIM 跑 DRAM 训练 ★关键★
        ├─ 永久内存可用 → 迁移到 DRAM
        ├─ Post-Memory PEI
        │   ├─ 派遣依赖内存的 PEIM
        │   ├─ 构建 HOB 链表
        │   └─ 通过 DXE IPL PPI 跳入 DXE
  DXE:  完整内存可用，加载各硬件驱动
  BDS:  选启动项、加载 OS Loader
  TSL:  grubx64.efi 作为 UEFI 应用执行
  RT:   ExitBootServices() → Linux 内核接管
```

PEI 是这条路径上**"无中生有"的阶段**——它从"没有内存"的真空里，把 DRAM 这个后续所有阶段的基础造出来。你前面学的 **Vcore（CPU 核心电压）**​ 是硬件上电的底座，**Cache-as-RAM**​ 是 SEC 给 PEI 的临时工作台，**PEI 的 Memory Init PEIM 跑 DRAM 训练**则是真正意义上的"让内存活过来"——这一步涉及复杂的时序校准、电平匹配、信号训练算法（Intel 叫 MRC，Memory Reference Code，是整个 UEFI 里最复杂、代码量最大的 IP 之一）。

要不要顺着把 **"DRAM 训练（Memory Training）具体在 PEI 阶段干了啥——时序校准、电平匹配、DQS 训练、写 leveling"**​ 过一下？那块是把"PEI 的 Memory Init PEIM 怎么把 DDR 跑起来"在硬件信号级收口，接你"Vcore → 内存"那条线。还是想先看 **"SEC 怎么用 Cache-as-RAM（no-eviction 模式配置）把 L1 cache 变成临时内存"**？后者更接你"CPU 复位后第一微秒"的极致底层，前者更接硬件训练方向。