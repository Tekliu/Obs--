EC 和 PCH 都是主板上"CPU 之外"的关键**芯片**，但层级和活儿完全不一样——一个管"**慢速平台杂事**"，一个管"**高速 IO + 时序总控**  "。顺着上一轮的开机链讲你一下就对上号了。

🔧 **EC = Embedded Controller**（**嵌入式控制器**）

在哪：主要在**笔记本主板**上，台式机一般没有独立 EC（功能并入 Super I/O 或直接 PCH 管）。物理上是一颗小 [[MCU]]，厂商常见是 ENE、ITE、Nuvoton 这几家，俗称"KBC"（Keyboard Backlight Controller，名字历史遗留）。

本质：一颗独立的 8051 / ARC **内核小单片机**，有自己的固件（
"ec.bin"，跟 BIOS 放同一颗 **SPI Flash**的不同区），关机了也能跑（靠 
"+5VSB" 喂电）。

EC 管的事（全是"慢速 + 平台级杂活"）：

- 开机键就是发给 EC 的——你按的那一下，EC 收到才拉 
"PS_ON#"
- 键盘扫描（笔记本内置键盘，非 USB 那种）
- 风扇 PWM（温度 → 转速曲线，[[DPTF]] 那套的根）
- 电池管理（电量、充放电策略、过充保护，
"/sys/class/power_supply/BAT0" 的数据源头）
- 背光、指示灯、触摸板开关
- 温度传感器（CPU/GPU/主板各处，热保护阈值在这设）
- Fn 组合键解码

所以笔记本"按开机键没反应""风扇狂转""电池认不到""键盘某个键失灵"，第一怀疑对象都是 EC。刷 EC 固件翻车 = 主板变砖，比刷 BIOS 还刺激。

🏛 PCH = Platform Controller Hub（平台控制器中枢）

在哪：Intel 台式机 / 笔记本主板上，CPU 旁边那块大芯片（现在消费级通常是 PCH 自己一颗，高端 HEDT/服务器可能还在 I/O die 里）。AMD 对应物叫 FCH (Fusion Controller Hub) / ASP，思路类似。

来历：2008 年 Intel 把老架构的"北桥 + 南桥"拆了——

- 北桥里的内存控制器 → 搬进 CPU（Nehalem 开始）
- 北桥里的 PCIe 根端口（显卡那条 ×16） → 也进 CPU
- 剩下南桥的所有 IO + 原来北桥剩的一点 → 合并成 PCH

所以现在架构是：

CPU ←── QPI/UPI/环形总线 ──→ PCH ←── eSPI/LPC ──→ EC / Super I/O
        │   │                              │
        │   └─ PCIe ×16 给独显              └─ 风扇/键盘/电池
        └─ 内存控制器 + IMC

PCH 管的事（全是"IO + 平台总控"）：

- USB 2.0/3.x 根端口（主板后置那些 USB 口）
- SATA / 部分 PCIe 通道（M.2、网卡、声卡这些）
- GPIO（开机灯、复位键、机箱跳线）
- SPI 控制器 → 挂 SPI Flash（BIOS/UEFI 固件就存在这）
- eSPI / LPC 总线 → 接 EC / Super I/O
- HPET / RTC / 看门狗
- ME (Management Engine) —— Intel 那套带外管理，跑在 PCH 里独立的小核上（AMT、远程 KVM 靠它）
- 供电时序总控：收 ATX 的 
"PWR_OK" + VRM 各级 
"PGOOD" → 汇总 → 发 
"PLTRST#" → 释放 
"CPURST#"

🔗 开机链里 EC 和 PCH 怎么配合

把上一轮的链补全，角色就清楚了：

按开机键
  → 【EC】收到，先自检（电池/适配器电压够不够）
  → EC 拉 PS_ON# 给 ATX 电源
  → ATX 出 +3.3/+5/+12V → VRM 降出 Vcore/VDIMM…
  → 各级 PGOOD 汇总到【PCH】
  → 【PCH】确认都 OK，拉 PWR_OK 给自己
  → 【PCH】释放 PLTRST#（平台复位）
  → 【PCH】释放 CPURST#（CPU 复位）
  → CPU 从 0xFFFFFFF0 取指，进 SEC

分工一句话：

- EC = 笔记本的"小管家"，管按钮/电池/风扇/键盘，开机键它先接
- PCH = 平台的"大总管"，管 IO + 供电时序 + 复位，CPU 能不能醒它拍板

📌 顺带提 AMD 的对应

AMD 平台不说 PCH，说 FCH (Fusion Controller Hub)，后来 Ryzen 又改叫 Chipset（但本质还是 FCH 那摊），笔记本里也有自己的 EC（因为键盘/电池/风扇这套跟 Intel 本一样，必须有一颗 MCU 管），AMD 的 EC 同样走 eSPI 连 FCH。

要不要接着拆 
"0xFFFFFFF0" 那条复位向量到底指向 SPI Flash 里哪个文件（比如 EDK2 里 
"FvMain" / 
"SecCore" 怎么摆的），或者 SEC 阶段那个 Cache-As-RAM 到底怎么用 L2/L3 当栈？前者偏"CPU 第一次取指到哪"，后者偏"DRAM 没起来前固件怎么活"，看你往 CPU 方向还是固件方向继续挖。