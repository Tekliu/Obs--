   
🎯 一句话定位

**VRM = Voltage Regulator Module**（电压调节模块），**主板 CPU 插座旁边那片电感 + MOS + 电容 + 控制芯片**的区域，负责把 ATX 送来的 +12V 降压到 Vcore（≈1.0–1.3V），还要动态跟 CPU 讨价还价调电压。

你主板上"16+1+2 相供电"广告里吹的那个"16 相"，指的就是 VRM 的 **Vcore 相数**。

⚙️ VRM 长啥样（肉眼认）
 
主板 CPU 插槽东侧/北侧那一坨：

- 电感（一个个小方块/圆柱，上面印 
"R22" 
"R68" 这种**微亨值**）—— 每相一个
- DrMOS 芯片（贴在电感旁边，有的裸露有的盖散热片）—— 每相一颗，把上下管+驱动封一起了
- 输入电容（ATX 12V 进口处那几颗胖电解）
- 输出电容（贴 CPU 一侧的 MLCC 群）—— 滤 Vcore 纹波
- PWM 控制器（藏在电感群里或背面，小一颗 QFN）—— VRM 的"大脑"

🔌 VRM 怎么干活（多相 Buck 直觉版）

1. 拓扑：多相 Buck（降压）

ATX +12V
  → 输入电容滤波
  → 每相：DrMOS（上管开→电感储能 → 下管开→电感放能给 CPU）
  → 多相轮流切（相位交错）
  → 输出电容群滤纹波
  → Vcore 喂进 CPU 的 VCC 针脚

为啥要"多相"不"单相"：

- 单相 200A 那根 MOS 直接炸 → 分摊到 16 相每相才 12A
- 纹波小：相位交错，电感电流峰错开，输出电容压力小
- 响应快：负载突变（CPU 从 idle → full boost）多相一起顶

2. 谁指挥 VRM 调电压

CPU 里有个 VRM 控制器接口（Intel 叫 SVID，AMD 叫 SVI），走串行总线跟 VRM 的 PWM 芯片对话：

CPU → SVID/SVI → VRM PWM → 调每相占空比 → Vcore 变

对话内容大致是：

- VID 码（Voltage ID）—— "我要 1.15V"
- 电流回报 —— VRM 把实时电流回喂给 CPU，CPU 算能效、算温度
- 负载线（Loadline）协商 —— 大电流时允许压降多少（Intel 叫 AC/DC Loadline，BIOS Setup 里能改）

3. 启动链里 VRM 在哪现身

回链 ATX 那轮：

按开机键 → EC 拉 PS_ON#
  → ATX 出 +12V/+5/+3.3，PWR_OK 拉高给 PCH
  → PCH 放 PLTRST# → 放 CPURST#
  → CPU 从 0xFFFFFFF0 取指 → SEC(CAR) → PEI
    → 【PEI 早期】**VRM 还没活透，但 CPU 已经在 CAR 里跑了**
    → 【PEI 中】`MemoryInit` 之前，必须先：
      1. 配 CPU 的 MSR/PCIe 寄存器和 VRM 握手（SVID 初始）
      2. VRM 出 Vcore（可能先给个保守值，比如 0.8V）
      3. VRM 再出 VDDQ/VDIMM（内存电）、VCCIO、VCCSA
    → 电压都稳 → DRAM 控制器才能训

所以 VRM 初始化是 PEI → 内存训练的前置条件，VRM 没起来，DRAM 训不了，卡 POST 的 
"55"（内存没认到) 很多时候其实是 VRM 某相 DrMOS 挂了或 SVID 线松了。

📊 VRM 那几个"相数 / DrMOS / 供电用料"到底啥意思

术语 人话
相数（16+1+2） 16 相 Vcore + 1 相 VCCGT(核显) + 2 相 VCCSA/VCCIO；相数越多能扛电流越大，但"虚标相数"也常见（并联当翻倍报）
DrMOS 上管+下管+驱动三合一芯片，Infineon / Renesas / OnSemi 三家主流；高端板用 70A/90A/110A DrMOS
PWM 控制器 指挥多相切波的，Infineon XDPE 系列最常见
输入 12V 口 ATX 24pin 里的 +12 只喂周边，CPU 主力吃 CPU 4+4pin 或 8+8pin（ATX12V/EPS 规范），这根直接连 VRM 输入
负载线（LLC） BIOS 里能改，大电流时允许 Vcore 掉多少；调太低稳但发热大，调太高省电但会蓝屏

🔗 跟前面几轮的回链

- ATX 给你 +12V 粗电 → VRM 降到 Vcore/VDIMM/VCCIO → DRAM 控制器才能训 → PEI 阶段收工 → DXE 起来
- 你之前卡 POST 听到的 "55 code" / "DRAM LED 常亮"，大半是 VRM → VDIMM 这侧（DrMOS 挂 / VDIMM 相掉 / SVID 没握手成）
- EC 那颗 8051 不管 VRM 细活，VRM 归 PCH + CPU 的 SVID 驱动 管，EC 只管"按开机键拉 PS_ON#"那一下

🛠 容易混的点

- VRM ≠ 主板供电"总称"——准确说 VRM 是"CPU 供电那块"，主板还有PCH 供电（1.05V/1.8V 那路，通常小 DrMOS 或 LDO）、内存供电（VDIMM 独立相，有的跟 Vcore 共用 DrMOS 但独立 PWM 通道）、VTT（终端电阻电，≈0.6V）
- "16+1+2 相"广告会注水：有的厂商把"每相并联两颗 DrMOS"报成"双倍相"，要看 DrMOS 电流档（70A 真·16 相比 50A 注水 16 相香）
- VRM 散热片为啥大：DrMOS 效率 90%+ 看着高，但 200A×12V=2400W 进、Vcore 1.2V×200A=240W 出，差出来的 2100W 多是损耗？不对重算——哦算错了：12V×单相电流… 实际上 12V→1.2V 降压，效率 85-90% 的话，损耗大概 12V×I_in vs 1.2V×200A，损耗在几十瓦级，全堆在 DrMOS 和电感上，所以散热片必须顶

🗺 Mermaid 思维导图

mindmap
  root((VRM))
    ATX +12V 进口
    多相 Buck
      电感
      DrMOS(上+下+驱)
      PWM 控制器
    指挥总线
      Intel SVID
      AMD SVI
    输出电
      Vcore(1.0-1.3V, 200A+)
      VDIMM(内存电)
      VCCIO/VCCSA
    启动链位置
      PEI 早期: SVID 握手
      PEI 中期: 出 Vcore → 训内存
      卡 POST 55: VRM/VDIMM 嫌疑

下跳 
"next: [[PEI_内存训练]]"——VRM 出 Vcore + VDIMM 之后，PEI 里那段 
"MemoryInit" 怎么把 DRAM 跑通、为什么"卡 55"、Cache-As-RAM 怎么移交栈到 DRAM，这一段是 SEC→PEI 最核心的活。要继续拆 PEI 那趴的 EDK2 代码（
"PeiCore" → 
"MemoryDiscoveredPpi" → 
"DxeIpl" 跳 DXE）还是先拆 Cache-As-RAM 那行 SEC 的 CAR 代码？前者偏"PEI 主线"，后者偏"SEC 那几行 MTRR 咋配的"——你挑。