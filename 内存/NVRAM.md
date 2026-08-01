## NVRAM = Non-Volatile RAM（非易失 RAM）。

在 PC 语境下，90% 指的是 SPI Flash 里 BIOS Region 下属的 
"FV_NVRAM" 区域，UEFI 规范里叫 Variable Store，靠 
"gEfiVariableWriteArchProtocol" 管着。

💡 注意："NVRAM"是个类别名不是硬件名——**任何掉电不丢又可字节级读写**的存储都能叫 NVRAM（Flash 区、FRAM、MRAM、电池供电的 SRAM 都算）。主板语境下默认指"UEFI 变量区"。
🗺 在 SPI Flash 里的位置

接上一轮 Flash 分区往下细化：

SPI Flash (16–64 MB)
├── Flash Descriptor
├── BIOS Region
│   ├── FV_MAIN (SEC/PEI/DXE/BDS 代码)
│   ├── FV_NVRAM  ← ★就是它
│   │   ├── VarStore (变量存储，EFI_VARIABLE_STORE_HEADER)
│   │   ├── VarData (实际 kv 对：Boot####, BootOrder, Setup, SecBoot...)
│   │   └── FTW (Fault Tolerant Write 日志，防断电撕裂)
│   └── FV_MICROCODE 等
├── ME Region
└── EC Region (可选)

所以"刷 BIOS 没清 NVRAM"和"刷 BIOS 顺便清了 NVRAM"是两回事——前者保留 
"BootOrder"/
"SecureBoot" 配置，后者回到出厂 Setup 默认值。

⚙️ NVRAM 里到底存了啥

分几类，都是 
"guid + name" 的 kv 对：

1. 启动相关（最重要）

- 
"Boot####" — 单个启动项（设备路径 + 描述 + 属性），#### 是 4 位 16 进制编号
- 
"BootOrder" — 
"Boot####" 的优先级列表，UEFI BDS 按它挨个试
- 
"BootNext" — 下次启动临时用一项（一次性，进系统后 
"efibootmgr -n" 改的就是它）
- 
"OsIndications" — OS 给 FW 留的意图位（比如"我要进固件设置""我要重启进 UEFI Shell"）

2. 安全

- 
"PK" / 
"KEK" / 
"db" / 
"dbx" — Secure Boot 密钥链
- 
"SetupMode" / 
"SecureBoot" — 当前模式

3. 固件配置（就是 BIOS Setup 菜单里那堆）

- 
"Setup" — 内存训练结果、CSM 开关、VT-x/VT-d、风扇曲线偏好…
- 
"Lang" — 界面语言

4. 厂商私有

- 各 OEM 自己 GUID 下的东西（戴尔 Latitude 的 
"Dell*"，联想的 
"Lnv*" 之类）

🔗 启动链里 NVRAM 的角色

串回你那条链：

CPU 复位 → SEC(CAR) → PEI(训DRAM) → DXE(扫设备 + 挂驱动)
  → 【DXE 阶段早中期】Variable Services 初始化
    → 读 SPI Flash 里 FV_NVRAM，把 VarStore 映射到内存里一份
    → 后续 DXE 驱动、BDS 都能调 gBS->GetVariable()
  → BDS 读 BootOrder → 找 ESP 里的 bootx64.efi
  → ExitBootServices() 之后
    → Boot Services 全废，但 **Runtime Services 还活着**
    → OS 还能通过 UEFI Runtime 调 GetVariable/SetVariable
    → Linux 下暴露成 /sys/firmware/efi/efivars/

关键点：NVRAM 是 UEFI 里"固件 ↔ OS"双向通信的唯一正规通道。OS 改 
"BootNext"、改 
"dbx"、设 
"OsIndications"，都是走这个。

🛠 实操里能见到的 NVRAM

Linux 下：

ls /sys/firmware/efi/efivars/          # 能看到所有变量
efibootmgr -v                          # 读 Boot#### / BootOrder
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Fedora" --loader \\EFI\\fedora\\grubx64.efi

Windows 下：

bcdedit /enum firmware                 # 看固件启动项
mountvol S: /S                         # 挂出 ESP + NVRAM（管理员）

翻车现场：

- 
"BootOrder" 乱了 → 开机直接进 grub rescue 或"找不到启动设备"
- 
"dbx" 更新把某个 bootloader hash 拉黑 → 开机 Secure Boot Violation
- NVRAM 写满 / FTW 日志坏 → DXE 阶段卡 
"EFI_NOT_FOUND"，清 NVRAM（主板扣电池 / 
"Load Defaults"）能救

⚠️ NVRAM ≠ CMOS 老概念，但也沾亲

老 BIOS 时代有 CMOS RAM（64 字节，靠纽扣电池供电的 SRAM） 存日期+软盘参数+密码，那是 NVRAM 的祖宗形态。

UEFI 把这块彻底换掉了——现在纽扣电池主要喂 RTC（实时时钟），Setup 配置不再放 CMOS RAM，全挪进 SPI Flash 的 NVRAM 区。所以：

- "扣电池清 CMOS"在现代 UEFI 主板上 = RTC 清零 + 触发 NVRAM 回到 Defaults（PCH 检测到 RTC 失效会走 
"GetVariable()" 失败 → 固件走 Default 分支），本质还是清的 Flash 里那块
- 纯 UEFI 板子 CMOS 区 64 字节基本废弃，只剩 RTC 时间和几个 legacy 兼容位

📊 NVRAM 跟前面几种存储的关系

 SRAM DRAM Flash(NOR) BIOS 区 NVRAM(FV_NVRAM) Flash(NAND/SSD)
掉电 丢 丢 不丢 不丢 不丢
可写 ✅ ✅ ❌烧录 ✅字节级 ✅页写块擦
擦写寿命 ∞ ∞ 有限 有限（FTW 兜底） 有限
典型 Cache 内存条 SEC/PEI/DXE 代码 BootOrder/SecureBoot/Setup 系统盘

🗺 Mermaid 思维导图

mindmap
  root((NVRAM))
    UEFI Variable Store
      Boot####
      BootOrder
      BootNext
      OsIndications
    SecureBoot
      PK / KEK / db / dbx
    Setup
      内存训练
      CSM / VT-x
      风扇曲线
    Runtime 可见
      /sys/firmware/efi/efivars
      GetVariable / SetVariable
    老祖宗
      CMOS RAM(64B, 纽扣电池)

下轮顺着 
"next" 走可以拆 UEFI Runtime Services 那套（
"SetVirtualAddressMap" 怎么把物理地址重映射到 OS 虚拟地址空间，为什么 ExitBootServices 之后 UEFI 还能活着调 
"GetTime"），或者跳回 EDK2 里 Variable 驱动怎么在 SPI Flash 上做 FTW（防断电撕裂）——前者偏"OS 接管后 UEFI 还剩啥"，后者偏"NVRAM 写塌了怎么办"。你偏哪边？