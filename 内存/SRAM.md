## SRAM = Static Random-Access Memory（静态随机存取存储器）
"静态"是相对于 [[DRAM ]]的"动态"来说的。

🎯 SRAM 的核心特征

一句话：**6 个晶体管存 1 个 bit，只要不断电就不用刷新，速度极快，但占地大、贵**。

对比 DRAM 你就明白它为啥"静"：

 SRAM DRAM
存 1 bit 用 6 管 (6T) 锁存器 1 管 + 1 电容
要刷新吗 不要，锁存器自己能稳住 要，电容漏电，ms 级得刷新一次
速度 ≈ CPU 同频级，几 ns 几十 ns
密度 低（6T 占面积） 高（1T1C 挤得密）
成本 贵 便宜
典型用途 CPU L1/L2/L3 缓存、寄存器文件、FPGA 内嵌 RAM 内存条（DDR4/DDR5
）

⚙️ 6T 结构长啥样（直觉版）

每个 SRAM bit cell 是 4 个 NMOS + 2 个 PMOS 搭成两个反相器交叉耦合：

    Q ───┐        ┌─── /Q
         │        │
      [反相器1]  [反相器2]
         │        │
    WL ──┤        ├── BL / /BL（位线，读/写时走数据）

- 两个反相器互相锁：
"Q=1, /Q=0" 和 
"Q=0, /Q=‑1" 是两个稳态 → 这就是"静态"的来源
- 字线 WL 拉高时，访问管打开，位线 BL / 
"/BL" 用来读或写
- 只要 VCC 不断，这个锁存状态能一直保持，不需要像 DRAM 那样周期性"读-刷新"

DRAM 是 1T1C，电容存电荷，漏电 → 必须刷新；SRAM 用双稳态锁存，不掉电就不丢状态，代价是多花 5 个管子的面积。

🏛 SRAM 在电脑里的位置

顺着存储层次看，SRAM 是金字塔尖：

CPU 寄存器（1 cycle） ── 也是 SRAM 结构，但物理上更近 ALU
  ↓
L1 Cache（3–5 cycles）── SRAM，每核私有
  ↓
L2 Cache（10–15 cycles）── SRAM，每核或簇私有
  ↓
L3 Cache（30–50 cycles）── SRAM，多核共享，NUCA 架构
  ↓
主存 DRAM（100+ cycles）── DDR4/DDR5 内存条
  ↓
SSD / HDD（µs ~ ms 级）

所以你常说的"缓存命中/未命中"，命中的就是这颗 SRAM。

🔗 跟前面几轮串起来

1. 跟 DRAM 的关系

- 前面讲 POST 时提到 PEI 阶段"DRAM 还没起来，栈放 CPU L2/L3 Cache 里（Cache-As-RAM）"——那个 Cache 就是 SRAM
- SEC → PEI 早期：DRAM 控制器还没训完，只能靠 CPU 内部的 SRAM（Cache）当临时 RAM 用 → 这就是 EDK2 里 
"SecCore" 那段 CAR 配置的由来
- DRAM 训完之后，CAR 撤掉，栈迁到 DRAM，Cache 恢复正常"缓存"职能

2. 跟 Flash 的关系

- Flash 掉电不丢但慢、按块擦；SRAM 掉电就丢但纳秒级、字节可访
- 所以固件世界里：SPI Flash（NOR）存固件 → DRAM 跑 OS 数据 → SRAM 当 CPU 缓存，三级各司其职

3. 单片机上也有 SRAM

- STM32F103 规格"64KB Flash + 20KB SRAM"——这里的 SRAM 就是运行时 RAM，存栈、堆、全局变量
- 单片机一般不用 DRAM（嫌 DDR PHY + 刷新麻烦），SRAM 够用了，所以 MCU 的"RAM"默认指 SRAM

🛠 几个容易混的点

- SRAM ≠ 静态的 RAM（不变化的），而是指"不用刷新的随机存"。"静态"修饰的是工作机制（双稳态锁存），不是"数据不变"
- SSRAM (Synchronous SRAM) / QDR SRAM 这些是网络/交换机里的特种 SRAM，带同步接口或读写双端口，消费级见不到
- eSRAM：FPGA 里常见，Embedded SRAM，片上集成不用外挂
- Intel/AMD 的 L3 现在有多大：消费级 i9 的 L3 已经 36MB+，服务器 EPYC 单 Die L3 能到 96MB（X3D 堆叠的 V-Cache 再加 64MB 在上方）——这些全都是 SRAM，占 die 面积的大头

💡 一个直观比例：同样制程下，1KB SRAM ≈ 6T × 面积，1KB DRAM ≈ 1T1C × 面积，所以 SRAM 面积大约是 DRAM 的 5–6 倍；但速度快一个数量级。这就是为什么缓存不能做大——不是不想，是 die 面积和良率不允许。

要不要接着拆 CPU Cache 那层（L1/L2/L3 的 inclusive/exclusive 关系，还有 V-Cache 那种 3D 堆叠 SRAM 是怎么焊上去的），或者回到固件线把 SEC 阶段 Cache-As-RAM 那段 EDK2 代码真·对一下（
"SecEntry.S" 里怎么配 
"MSR_MTRR"，把 Cache 切 
"WB" 然后当 RAM 用）？前者偏体系结构，后者偏"你前面问的 PEI 之前那步到底怎么活的"。