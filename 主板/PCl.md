## PCI = Peripheral Component Interconnect

**1992** 年 Intel 搞的总线标准，用来**把显卡、声卡、网卡、SCSI 卡这些"外设"挂到北桥/南桥上**。你

## 拓扑位置（1990s–2000s 经典款）

CPU ←FSB→ 北桥 ──PCI总线── 南桥(ICH)
                ↓              ↓
             显卡(AGP优先)    USB/SATA/音频/PCI插槽
             网卡/SCSI卡      （南桥里也桥接更多PCI设备）

- 北桥侧：有个 **PCI Root Bridge**，把 **CPU 的 FSB** 事务翻译成 PCI 事务
- 南桥侧：有个 PCI-ISA / PCI-LPC 桥，把 PCI 再往下桥接给更老的总线（ISA→LPC，接 BIOS/CMOS/老串口）
- PCI 插槽：主板上那些白色的长插槽（老主板能看到，现在新板基本绝迹）

## PCI 本身的特征（跟 FSB 是同类毛病）

|     | PCI                    | FSB                    |
| --- | ---------------------- | ---------------------- |
| 拓扑  | 广播共享总线                 | 广播共享总线                 |
| 频率  | 33 MHz / 66 MHz        | 1333/1600 MT/s         |
| 位宽  | 32-bit（也有 64-bit 服务器版） | 64-bit（Quad-pumped）    |
| 带宽  | 33 MB/s(33M×32b)       | ~12.8 GB/s（1600MT×64b） |
| 仲裁  | 多 master 抢             | REQ/GNT 同理             |


所以 PCI 跟 FSB 是同一类病：广播 + 抢线 + 频率上不去。2004 年 [[PCIe]] 出来就是专门替它的——把"广播总线"改成"**point-to-point** 串行 lane"，类似 FSB→QPI 那波革命。

## PCI → PCIe 的对应关系（帮你串）

|        老         |               新               |        改了啥        |
| :--------------: | :---------------------------: | :---------------: |
| PCI 总线（并行，多设备共享） |      PCIe（串行，每设备独占 lane）      |     **广播→交换**     |
|   33/66 MHz 并行   |  2.5GT/s 起（Gen1→Gen5 32GT/s）  |       频率解绑        |
|    北桥—PCI—南桥     | CPU 直出 PCIe Root，PCH 再出副 PCIe | 北桥被吞后 PCIe 留下来当主干 |

💡 名字里的 "e" = express，PCIe = PCI Express，意思是"PCI 的高速串行版"，不是新东西，是老 PCI 的换代。

回到你那条"北桥↔南桥"的线

- 1998–2003（BX440 / i815 那代）：北桥 ↔ 南桥 之间真的就是 PCI 总线，外加一条 Hub Link（Intel 810 起自己搞的私有窄带，比 PCI 快，专门连南北桥，后来演成 [[DMI]]）
- 2004 之后：PCI 逐渐被 PCIe 替掉，南桥（后来的 PCH）里的设备也纷纷换 PCIe 口（USB 控制器、网卡、NVMe…）
- 现在：PCH 里还留着 PCI 兼容层（ACPI 里还能看到 
"PCI0" 总线，设备号 
"00:1f.0" 就是 PCH 本体），但物理层已经是 [[PCIe]] 了，PCI 只是逻辑兼容壳

所以你那句"北桥到南桥的是 PCI"——老时代对，新时代那条物理链路叫 DMI（本质 PCIe x4），PCI 只是逻辑上还挂着个名。

要不要顺着把 Hub Link → DMI 这条"南北桥专用私连线" 单独理一下？它比 PCI 更贴近你说的"北桥南桥之间的那条"，Intel 从 i810（1999）就开始用，比 PCIe 还早 5 年，是 DMI 的真正祖宗。