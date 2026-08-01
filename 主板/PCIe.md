PCIe 最底层叫 PL（Physical Layer，物理层）
PL 层干的具体活（最底层，离"电线"最近）
- 差分对：每条 lane = TX+/TX-、RX+/RX- 两根线一对，抗干扰
- lane 聚合：x1 / x4 / x8 / x16，DMI 固定 x4
- 编码：
   - PCIe 3.0 之前 → 8b/10b（传 8bit 变 10bit，插时钟+直流平衡）
   - PCIe 3.0 起 → 128b/130b（更省开销）
- 速率（GT/s，每秒千兆次）：
   - Gen1: 2.5 GT/s
   - Gen2: 5 GT/s
   - Gen3: 8 GT/s（DMI 3.0 这档）
   - Gen4: 16 GT/s（DMI 4.0 这档）
   - Gen5: 32 GT/s
PL 层就是"这一堆 0/1 怎么变成差分电信号在铜线上跑"——纯硬件 PHY 干的事，不用管"这包是读还是写"。

DMI 的 PL 层 = 原样抄 PCIe

DMI 4.0 的 PL 层 == PCIe 4.0 x4 的 PL 层：

- 同样 16 GT/s
- 同样 128b/130b
- 同样差分对电气特性
- Intel 连 PHY IP 都没重写，直接复用 PCIe 的
最底层 DMI 跟 PCIe 是 100% 一样的，从"电线"这个角度看，分不出来哪根是 DMI 哪根是 PCIe。
---

往上垒第二层：DLL（Data Link Layer）    
DLL ← 你在 PL 上传的每一包 TLP，DLL 给它加：
     ├── 序列号（Seq Num）—— 防丢、防重
     ├── LCRC（Link CRC）—— 校验
     └── ACK/NAK 机制 —— 对方没回 ACK 就重发

--- 

最上层：TL（Transaction Layer）—— DMI 唯一动手脚的地方

TLP 格式（简化）：
┌─────────────────────────────┐
│ Header（Fmt/Type / 地址 / 长度）← ⚠️ Intel 私货就在 Type 字段里
├─────────────────────────────┤
│ Data（如果有，比如 MWr 写数据）
└─────────────────────────────┘

- 标准 PCIe 的 Type 字段：PCI-SIG  分配，CfgRd、MRd、MWr、Cpl…
- Intel DMI 私有的 Type：PCI-SIG 不管的那片保留区，Intel 自己填，用来传 PCH↔CPU 的 sideband（风扇、SMI、ME、fTPM…）
     