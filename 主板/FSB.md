## FSB= Front Side Bus，前端总线，就是 **Nehalem（2008)** 之前，CPU 和北桥之间那根唯一的广播总线。

#拓扑位置

CPU ←──FSB──→ 北桥(MCH) ←→ 内存
                  ↓
               显卡(AGP)

所有 CPU 核要访问内存、要发请求给显卡，全得走这一根 FSB，北桥是 FSB 那头的"代理人"。

#名字为啥叫前端

CPU 内部视角：

- 后端 = ALU 算完、写回寄存器那一段（你平时听的"后端流水线"是这）
- 前端 = CPU 核往外取指令、取数据、跟外面要东西这一侧 → 所以连北桥这根总线叫 Front（前）Side Bus

#FSB的三个原罪（也是IMC为啥必须进CPU）

1. **广播 + 共享：所有核抢一根总线，核越多越挤** （4 核还能忍，服务器 8+ 核直接炸）
2. 频率上不去：FSB 1333/1600 MT/s 就顶天，DRAM 自己都 2400+ 了，瓶颈不在内存而在总线
3. 多一跳延迟：CPU → 北桥 → 内存，比"CPU 直连内存"多一跳，内存延迟多 ~30-40ns

**退场**

Nehalem（2008）把 IMC 塞进 CPU 后：

- 桌面/服务器 FSB → 被[[QPI]]（QuickPath Interconnect，CPU 之间 / CPU↔PCH） 替掉
- 到后面 QPI 又演进成 [[UPI]]（Ultra Path Interconnect），现在至强之间还在用
- 消费级那条 CPU→PCH 的窄桥改叫 [[DMI]]（本质 PCIe 包了一层）

一句话：FSB = 北桥时代 CPU 往外唯一的那根广播总线，死于"核多了挤不动 + 内存频率追不上"，被 QPI/DMI 这种 point-to-point 串行互连替掉。