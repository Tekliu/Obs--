#PCH的定位
x86 平台里**CPU 甩出来的低速 IO 总管 + 平台管理二传手**——CPU 只留内存控制器+PCIe Root（显卡/主 M.2），剩下 USB / SATA / 副 M.2 / 声卡 / GPIO / fTPM / 远程管理(ME) 全甩给 PCH，PCH 再用一条 DMI（本质 4 lane PCIe）回 CPU。

它不计算，只做**IO 集散 + 带外管理**

历史发展（怎么来的）

1. 早期：[[南北桥]]分立，**北桥管内存+显卡，南桥管杂活**（ICH 那代）
2. 2003 AMD K8 / 2008 Intel Nehalem：内存控制器搬进 CPU → 北桥被掏空
3. 2008 起 Intel 改名叫 PCH（Platform Controller Hub），AMD 后来叫 FCH——本质就是"剩下来那半南桥"，单芯片接着干活
4. 现在：CPU 直出 PCIe 越来越多（显卡、主 M.2、USB4、雷电都直出），PCH 管的越剩越少

历史潮流里的趋势

PCH 是 x86 过渡产物，正在被两头挤死：

- 消费端：还活着，但管的越来越少（SATA 快灭、M.2 走 CPU、USB4 走 CPU）
- AI 服务器（至强6 + B200）：PCH 已被砍，CPU 直出 88 lane PCIe5，管理面扔给 BMC/DPU
- 手机 / Apple Silicon / Grace / 高通：从来就没有 PCH，SoC 一把梭
- 国产龙芯 / 申威：还保留 CPU + PCH 分立，x86 式思路没完全 SoC 化

长期看：legacy 越清完，PCH 越萎缩，最后退成"最小壳"或消失。Intel Clearwater Forest 已经在试 PCH-less 服务器板。
一句话收：PCH = 北桥死后的南桥续命版，靠 legacy 和多 SKU 刀法多活了 15 年，现在两头被 CPU 直出 + DPU 夹，在退场路上。