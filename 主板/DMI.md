## DMI = Direct Media Interface 
就是前面提过的那根——CPU 和 PCH 之间的那条专用窄桥，Nehalem（2008）起 Intel 消费级/主流平台的标配。

位置和形态

CPU ───DMI x4─── PCH
      (4 lane)

- 本质：**PCIe 协议包了一层 Intel 私有头**，物理层就是 PCIe，所以 DMI 2.0 ≈ PCIe 2.0 x4，DMI 3.0 ≈ PCIe 3.0 x4，DMI 4.0 ≈ PCIe 4.0 x4
- lane 数：固定 x4（4 条差分对），从 Nehalem 到现在没变过
- 带宽演进：
   - DMI 1.0（2008）= PCIe 1.0 x4 → 1 GB/s 单向
   - DMI 2.0（Ivy Bridge）= PCIe 2.0 x4 → 2 GB/s
   - DMI 3.0（Skylake）= PCIe 3.0 x4 → 8 GB/s（现在主流板还多这）
   - DMI 4.0（Alder Lake / 600 系芯片组）= PCIe 4.0 x4 → 16 GB/s 单向

它干啥用

PCH 下挂的所有东西要跟 CPU 通信，全得过这条 DMI：

PCH 下挂                        DMI x4                       CPU
────────────────────────────────────────────────────────────────
副 M.2(NVMe) ┐
网卡(2.5G/10G)├────────── DMI ──────────→ 内存 / 显卡 / 主 M.2
USB 一堆     │
SATA         │
音频 codec   │

所以瓶颈就在这：哪怕 PCH 下每个设备都没跑满，加起来超过 DMI 带宽就堵。比如你副 M.2 跑 7 GB/s + 网卡 2 GB/s + USB 一堆，DMI 3.0（8 GB/s）就顶住了，得上升级 DMI 4.0 的板。

名字来历 & 跟 QPI 的关系

- DMI = Direct Media Interface，消费级 / 主流平台 CPU↔PCH 专用
- QPI / UPI = 服务器 / 多路平台 CPU↔CPU 专用，带宽更大（x20 起跳，不是 x4）

这俩同根：都是 Nehalem 那波"替掉 FSB 的 point-to-point 串行互连"，只是：

- 服务器侧要连多颗 CPU → 做宽做贵 → QPI → UPI
- 消费侧只要连一颗 PCH，x4 够用还省 pin → DMI

一句话：DMI = "穷人版 QPI"，CPU↔PCH 那根 x4 窄桥，协议层 PCIe，物理层差分对，从 Nehalem 活到现在，是 PCH 还能存在的唯一理由——没这条桥，PCH 下挂的设备回不到 CPU。
接你前面"PCH 在退场"那条——AI 服务器砍 PCH 的根本原因就是：DMI 这条 x4 窄桥太窄了，至强6 直接 CPU 直出 88 lane PCIe5 给 GPU/NVMe/NIC，PCH 那点活扔给 BMC/DPU，DMI 这条链就被跳过了。

要顺着看 CXL 怎么又从 CPU 直出、把"CPU↔加速器/内存扩展"这条也跳出 PCIe/DMI 老框" 不？那条是现在（2024+）的新互连革命，接 QPI→UPI→CXL 这条"CPU 互连三代"刚好收口。