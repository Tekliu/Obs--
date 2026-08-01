# QPI = QuickPath Interconnect，Intel 在 Nehalem（2008）引入、用来替掉 FSB 的 point-to-point 串行互连。
 
 FSB 是广播共享总线——**多核抢一根，频率上不去，核越多越炸**。

| 时代                | CPU↔""北桥活""     | 多路 CPU↔CPU  | CPU↔""南桥""      |
|-------------------|-----------------|-------------|-----------------|
| 2008 前            | FSB → 北桥(外置IMC) | 无/少         | 北桥—PCI—南桥       |
| 2008 Nehalem 起    | IMC 进 CPU，FSB 死 | QPI         | DMI → PCH（南桥改名） |
| 2017 Skylake-SP 后 | 同上              | UPI（QPI 升级） | DMI 3.0/4.0     |
| 现在                | 同上              | UPI 3.0（至强） | DMI 4.0（消费）     |
QPI 改成 pair-wise 串行链路，每个 link 是单向 lane 成对（Tx/Rx），包交换：

以前 FSB：   CPU0 ──广播总线(FSB)── 北桥（所有核抢）
QPI 之后：   CPU0 ══QPI══ CPU1   （每对 CPU 独享一条）
            CPU0 ══QPI══ PCH     （消费端这条后来改名叫 DMI）

- 点对点：每对 CPU/设备独占一条，不广播
- 包交换：地址、数据、确认打成 flit 发，类似网络
- 频率独立：QPI 自己跑（Nehalem 时代 4.8-6.4 GT/s），不绑内存频率
- 带内带外都走：不仅传内存访问，还传核间中断、缓存一致性（MESI 改 MESIF，F 是 Forward 给 QPI 加的）

QPI 用在哪

场景 链路
多路至强（2S/4S/8S） CPU ↔ CPU 之间走 QPI
单路桌面/服务器 CPU ↔ PCH，这条后来改名叫 DMI（本质还是 QPI 协议瘦身版）
Uncore 内部 QPI 的 Home/CA/SA 代理挂在 L3 上，管 NUMA 内存路由

后来怎么演进

- 2017 左右 Skylake-SP → QPI 升级成 UPI（Ultra Path Interconnect），带宽更高、时延更低，现在至强之间还在用（UPI 2.0 / 3.0）
- 消费端那条 CPU→PCH 从 Nehalem 起就叫 DMI 2.0/3.0/4.0，本质就是 QPI 的"省钱版"——4 lane 窄桥，够用就行
- AMD 同期对应的是 HyperTransport（HT，2003 K8 起） → 后来 EPYC 改成 Infinity Fabric（GMI 片间 / xGMI 多路）

一句话串：FSB（广播共享）→ QPI（point-to-point，Intel 2008）→ UPI（至强现役）/ DMI（消费端 CPU→PCH 那条）。
所以你前面问"FSB 被啥替了"——服务器/多路侧是 QPI→UPI，消费侧是 DMI，俩是同根生的兄弟，只是 DMI 被砍成 4 lane 窄桥给 PCH 用。
要顺着这条"互连演化"再看 Infinity Fabric 跟 UPI 怎么打的吗？AMD 那边更有意思——GMI/xGMI 跟内存控制器、IO die 捆一起的玩法，跟 Intel 的 UPI 思路不太一样。