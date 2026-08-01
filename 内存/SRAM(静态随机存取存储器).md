接着上一轮敲定的"Obsidian 归档版"规格，给你出这份 SRAM 条目——后面你可以直接 
".md" 存 vault，
"prev/next" 已经按你这条启动链锚好了。

📄 Obsidian 归档版 · SRAM

---
title: SRAM
tag: [[firmware-boot]] [[存储层次]]
prev: [[Flash]]
next: [[Cache-As-RAM / SEC]]
---

🎯 一句话定位

SRAM = Static Random-Access Memory（静态随机存取存储器）。

"静态"是相对 DRAM 的"动态"而言——6 个晶体管靠双稳态锁存存 1 bit，只要不断电就不用刷新，速度纳秒级，但面积大、贵。

⚙️ 6T Cell 直觉

每个 SRAM bit 是 4 NMOS + 2 PMOS 搭成两个反相器交叉耦合：

Q ───[反相器1(非门)]───┐
                  │  ← 双稳态：Q=1/¬Q=0 或 Q=0/¬Q=1
     [反相器2]───┘
WL（字线）──┤  ├── BL / ¬BL（位线，读写数据）

- 两个反相器互锁 → 掉电才丢，不用刷新 = "static"
- WL 拉高 → 访问管开 → 位线走数据
- DRAM 是 1T1C（电容漏电 → ms 级必须刷新），SRAM 是 6T 锁存 → 快但占面积

📊 SRAM vs DRAM vs Flash（Obsidian 表格对齐版）

维度 SRAM DRAM NAND Flash
存 1bit 6T 锁存 1T + 1C 浮栅 / 电荷捕获
刷新 不用 要（ms 级） 不用（但擦按块）
速度 ns 级 几十 ns µs ~ ms
擦写单位 字节 字节 页写 / 块擦
密度 低（贵） 中 高（便宜）
掉电 丢 丢 不丢
典型 L1/L2/L3 内存条 SSD / U 盘

面积比直观感受：同制程下 1KB SRAM ≈ 5–6 × 1KB DRAM，所以缓存不能做大——不是不想，是 die 面积和良率不允许。

🏛 在电脑里的位置（存储层次）

graph BT
    Reg[CPU 寄存器<br/>1 cycle<br/>也是 SRAM 结构]
    L1[L1 Cache<br/>3-5 cycles]
    L2[L2 Cache<br/>10-15 cycles]
    L3[L3 Cache<br/>30-50 cycles<br/>多核共享]
    DRAM[DDR4/5 内存条<br/>100+ cycles]
    SSD[SSD / NVMe<br/>µs ~ ms]
    
    Reg --> L1 --> L2 --> L3 --> DRAM --> SSD

- L1/L2/L3 全是 SRAM，所以"缓存命中/未命中"命中的就是它
- 消费级 i9 L3 已 36MB+，服务器 EPYC 单 Die L3 能到 96MB；X3D 的 V-Cache 是把额外 64MB SRAM 3D 堆叠焊在 L3 上方

🔗 回链到你的启动链

前面讲 POST → PEI 时提过一句：

DRAM 还没起来，栈只能放 CPU L2/L3 Cache 里（Cache-As-RAM，CAR）
那个 Cache 就是 SRAM，这一段的逻辑串起来是这样的：

CPU 复位 → SEC 阶段
  → DRAM 控制器还没训 → 没 DRAM 可用
  → 把 L2/L3 SRAM 配成「Write-Back 但不回写到内存」
  → MTRR 把一段物理地址映到 Cache，当临时 RAM 用
  → 栈、局部变量全在这块 SRAM 里跑
  → PEI 训完 DRAM → 栈迁到 DRAM → CAR 撤掉 → Cache 恢复"缓存"本职

所以 SRAM 在你启动链里出现两次身份：

1. SEC/PEI 早期 → 被迫当"主存"用（CAR）
2. 进系统后 → 回归本职当 CPU Cache

🛠 单片机上也有 SRAM

STM32F103 规格里"64KB Flash + 20KB SRAM"——MCU 的"RAM"默认就是 SRAM，因为 MCU 一般不挂 DRAM（嫌 DDR PHY + 刷新麻烦），SRAM 够用。EC 那颗 8051 片上 XRAM 也是 SRAM。

🗺 Mermaid 思维导图（整条存储谱系）

mindmap
  root((存储))
    volatile(掉电丢)
      SRAM
        CPU 寄存器
        L1/L2/L3 Cache
        MCU 片上 RAM
        CAR(SEC/PEI 临时 RAM)
      DRAM
        DDR4/DDR5 内存条
    non-volatile(掉电不丢)
      NOR Flash
        SPI BIOS 芯片
        XIP 可执行
      NAND Flash
        SSD
        U 盘 / eMMC / UFS

下轮顺着 
"next" 走就是 SEC 阶段 Cache-As-RAM 那段的 EDK2 代码（
"SecEntry.S" 怎么配 MTRR 把 Cache 当 RAM 用），还是先把前面 UEFI → POST → EC/PCH → Flash → SSD → SRAM 这批补一圈 Obsidian 归档？前者偏"固件代码怎么写"，后者偏"你 vault 先铺齐"。