 #Flash指的是FlashMemory（闪存）
 主板 / SSD(固态硬盘) / U 盘 / 手机里全是它，但主板上的那颗 SPI Flash 和你买 SSD 里的 Flash 不是同一种，得分开讲。

🧱 Flash 的本质

Flash =**电可擦、掉电不丢、按块擦按页写的非易失存储器**

对比一下你熟悉的几种存储，位置就清楚了：

|      | [[内存/SRAM]]     | [[DRAM]] | [[EEPROM]]                     | Flash                |
| ---- | ------------ | -------- | ------------------------------ | -------------------- |
| 掉电丢吗 | 丢            | 丢        | 不丢                             | 不丢                   |
| 速度   | 最快（CPU 缓存级）  | 快（内存条）   | 慢                              | 介于 EEPROM 和 DRAM 之间  |
| 擦写单位 | 字节           | 字节       | 字节                             | 块(Block) / 页(Page)   |
| 成本   | 贵            | 中        | 贵                              | 便宜                   |
| 典型用途 | CPU L1/L2/L3 | 内存条      | 存几个配置字节（EEPROM 现在基本被 Flash 取代） | BIOS、EC、SSD、U 盘、手机存储 |

Flash 的核心特征就两条：

1. 按"块"擦、按"页"写——不能像 RAM 那样改一个字节就改一个字节，要改得先把一整块（几 KB ~ 几百 KB）擦成 0xFF，再往里写
2. 寿命有限——每个块擦写几千~几万次（SLC > MLC > TLC > QLC 递减），所以 SSD 要做磨损均衡

名字 Flash 来自东芝当年（1984）发明人舛冈富士雄的描述——"擦除的时候像闪光(flash)一样把一整块清掉"，跟 EEPROM 逐字节擦比起来快太多了。

🔌 主板上的 Flash：SPI Flash（NOR Flash）

回到主板语境，CPU 旁边、PCH 附近有一颗 8 脚小芯片，丝印可能是 Winbond（W25Q256）、Macronix（MX25L）、Gigadevice——这就是 SPI Flash，存的东西有：

一颗 SPI Flash（常见 16MB / 32MB / 64MB）里的分区（由 Flash Descriptor 定义）：
┌─────────────────────────────┐
│ Flash Descriptor (FD)        │ ← 描述各区域边界 + 权限
├─────────────────────────────┤
│ BIOS Region                  │ ← UEFI 固件主体（SEC/PEI/DXE/BDS）
├─────────────────────────────┤
│ ME / CSME Region             │ ← Intel Management Engine 固件
├─────────────────────────────┤
│ EC Region（部分主板）        │ ← EC 8051 固件（有的 EC 自己独立 Flash）
├─────────────────────────────┤
│ GbE Region（可选）           │ ← 板载网卡的 PXE/iSCSI 配置
└─────────────────────────────┘

为什么用 NOR Flash 而不是 NAND？

- NOR：可 XIP（Execute In Place）——CPU 可以直接通过总线取指执行，不用先拷进 RAM。主板 SPI Flash 挂在 PCH 的 SPI 控制器下，地址被 MMIO 映射到 4GB 顶端（
"0xFF000000" 那块），CPU 从 
"0xFFFFFFF0" 取第一条指令，就是在这里取到的 → 这就是"复位向量指向 SPI Flash"的硬件基础
- NAND：密度高、便宜，但不能 XIP，得先 DMA 拷进 RAM 才能跑 → 所以 NAND 用在 SSD / eMMC / U 盘，不用来存 BIOS

所以主板这颗是 SPI NOR Flash，特点：

- 容量小（16–128 MB 顶天）
- 读快、写慢、擦更慢
- 位宽串行（SPI 四线：CLK / MOSI / MISO / CS，后来有 Dual/Quad SPI 提速）
- 电压 3.3V，8 脚（SOIC-8），热风枪就能吹下来

📱 SSD / U 盘 / 手机里的 Flash：NAND Flash

NAND 才是大众说的"闪存"的主力，跟主板 SPI Flash 是同门不同种：

 NOR (SPI Flash) NAND (SSD/U盘/手机)
接口 串行 SPI 并行 / ONFI / NVMe
容量 MB 级 GB ~ TB 级
XIP ✅ 可以 ❌ 不行
读速度 慢（几十 MB/s） 快（PCIe 5.0 SSD 已破 10 GB/s）
擦写粒度 块 4–64 KB 块 128KB–几 MB
位可靠性 几乎不错 会比特翻转，要 ECC + 坏块管理
典型 BIOS 芯片、EC 固件 SSD、eMMC、UFS、SD 卡

NAND 又分 SLC / MLC / TLC / QLC（每个单元存 1/2/3/4 bit），越往后越便宜越大越慢寿命越短。

⚙️ 启动链里 Flash 的角色串一遍

把前面几轮连起来，Flash 在哪：

按下电源 → VRM 稳 → PCH 收 PWR_OK → 释 PLTRST# → 释 CPURST#
  → CPU CS:IP = 0xFFFFFFF0
    → 那里是 **PCH 把 SPI Flash 映射到 4GB 顶端**的窗口
      → 取到的是 SPI Flash 里最末一条跳转指令
        → 跳去 Flash 里 FD 区之后的 BIOS Region 入口
          → 进 SEC（Cache-As-RAM）→ PEI → DXE ...

所以 Flash 是整个启动链的"第一本书"——CPU 醒来第一件事就是去它里面读指令，没它或它坏了 = 主板变砖（表现为"风扇转、黑屏、debug 卡无 code 或固定 code"，编程器重刷能救）。

🛠 顺带提几个容易混的点

- "刷 BIOS"刷的就是 SPI Flash——工具如 
"flashrom" + CH341A 编程器，或者直接 
"fwupd" / 主板 BIOS 界面里 Capsule Update（后者走 UEFI 
"UpdateCapsule()"，不关 SPI 直接写）
- Flash Descriptor 锁：很多主板 FD 设了 
"BIOS_REGION_WRITE_LOCK"，导致即使你在 OS 里 
"flashrom -p internal" 也写不进——这是防 brick 也是防恶意刷，企业机尤其严
- 云/服务器里的"Flash"：BMC（基板管理控制器，就是 iDRAC / iLO / IPMI 那套）自己也挂一颗 SPI Flash，存 BMC 固件，跟主 BIOS Flash 是两颗，互不干扰——所以服务器"主 BIOS 刷炸了"还能靠 BMC 远程救

要不要接着拆 Flash Descriptor 那张分区表长啥样（
"ifdtool -d" 能吐出来），或者 CPU 从 
"0xFFFFFFF0" 取到那条跳转之后，怎么找到 SEC 入口的 FV（Firmware Volume）头？前者偏"一颗 Flash 怎么被切成几块给 BIOS/ME/EC 分"，后者偏"SEC 阶段真·第一行 C 代码在哪"。