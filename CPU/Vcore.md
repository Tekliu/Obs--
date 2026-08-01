 VCORE（也叫 CPU Core Voltage / 核心电压）就是主板 [[VRM]]（电压调节模块）实际输送给 CPU 核心的供电电压，单位伏特（V）。它是 CPU 能不能跑起来、跑得稳不稳、发热大不大的最核心电气参数 。
//voltage  电压
一、它到底是什么

CPU 内部有上亿个晶体管，每个晶体管开关都需要电力供应，VCORE 就是驱动这些晶体管的"血源"。它有几个关键特征：

- 动态变化：不是**固定值**，而是**随负载、温度、频率**实时调整——空闲时降到 **0.7-0.9V** 省电，满载时升到 **1.2-1.4V** 保证性能
- 由 **CPU** 通过 [[VID 协议]]向主板"要电"：CPU 根据**自身频率/温度**计算出需要的电压，通过 VID 信号告诉主板 VRM，"给我 1.25V"，VRM 再去 +12V 转出来
- 核心电压 ≤ I/O 电压：**CPU 核心部分的供电通常低于外围 I/O 部分的供电**
- 典型范围：现代 CPU 的 VCORE 一般在 0.8V 到 1.5V 之间，Intel 9 代 i9-9900K 的官方最大值 1.52V
- 健康区间：大多数现代 CPU 在 **0.9-1.3V** 之间运行是安全的，超过 **1.3V** 就在透支寿命

二、为什么它会变来变去（[[Vdroop 现象]]）

你用 [[HWiNFO]] 或 BIOS 看 VCORE，会发现数字一直在跳——这是正常的，原因是 Vdroop（电压下垂） ：

- 当 CPU 从空载突然满载，瞬间电流需求暴增，VRM 来不及响应，核心实际拿到的电压会短暂跌落
- 主板通过 **Load-Line Calibration**（负载线校准，LLC） 来补偿：负载越高，VRM 主动抬高输出电压，抵消 Vdroop
- 所以你会看到：待机 0.8V → 满载 1.25V → 瞬时尖峰 1.4V，这些都是设计内的正常波动

💡 一个反直觉的点：软件里看到的 "Vcore"、"CPU Core Voltage"、"VR Vout (SVI2 TFN)" 可能不是同一个传感器的读数，因为测量点不同、刷新延迟不同，同一时刻显示 1.20V 和 1.24V 都是正常的，不是故障
三、不同平台的典型范围

CPU 平台 空载 VCORE 满载 VCORE 备注
Intel 10-12 代 i3/i5 0.7-0.9V 1.0-1.2V 日常使用 1.0-1.2V
Intel 10-12 代 i7/i9 0.7-0.9V 1.1-1.4V 超频可到 1.35V+
Intel 13/14 代 K 系列 0.8-1.0V 1.2-1.4V 曾有电压过高导致缩缸问题
AMD Ryzen 3000-5000 0.8-1.0V 1.0-1.3V 多数可从降压中获益
AMD Ryzen 7000/9000 0.8-1.0V 1.1-1.3V 
老旧平台（Intel 4-9 代） 0.9-1.1V 1.2-1.4V 工艺老，所需电压偏高

注：以上数据综合自各厂商电气规格 。

四、VCORE 异常意味着什么

⚠️ 电压过高（>1.4V 长期运行）

- 后果：CPU 寿命缩短、硅老化加速（Intel 13/14 代曾经爆发过大规模"[[缩缸]]"事件，根源就是 VCORE 请求过高，后来靠微码更新 0x129/0x12B/0x12F 修复 ）
- 临界值：Intel 规定 VCCCORE 运行上限 1.72V，但这只是"不立刻烧"的天花板，不是推荐值
- 日常 24/7 使用建议：保持在 1.3V 以下最安全

⚠️ 电压过低（低于 VID 请求值）
- 后果：系统不稳定、蓝屏、死机、高负载重启
- 常见原因：
   - 电源功率不足或 +12V 输出异常
   - 主板 VRM 元件老化
   - BIOS 里 LLC 设置过低或手动降压过度
   - 散热不良触发保护性降压

🔧 技嘉主板报错代码 15

如果你看到主板 Debug 灯显示代码 15，那就是 "CPU Vcore Power Supply Error"——核心供电异常。排查顺序 ：

1. 检查电源功率是否足够、+12V 输出是否稳定
2. 检查主板 VRM 区域有无烧焦/电容鼓包
3. BIOS 恢复默认设置（特别是超频后）
4. 重新安装 CPU 和散热器，确保接触良好

五、怎么看自己 CPU 的 VCORE

🪟 Windows 下

- Intel XTU（Intel 官方工具，仅支持带 K/X 后缀的 CPU）：打开 → 点右下角硬件监控图标 → 勾选 Core Voltage
- HWiNFO64：运行 → Sensors Only 模式 → 找 "Vcore" 或 "CPU Core Voltage (VR Vout)"
- CPU-Z：主界面直接显示当前 VCORE

🖥️ BIOS/UEFI 里

开机按 Del / F2 / F12 进 BIOS → 找 "Monitor"、"Hardware Health"、"PC Health Status" 页面 → 看 "Vcore" 或 "CPU Core Voltage"

⚠️ 重要提醒：BIOS 里可能同时显示多个电压项，务必区分 Vcore 和 [[VCCIO]]、[[VDDIO]]、[[VCCSA]] 等其他电压——只有 Vcore 是 CPU 核心供电，其他是 I/O、内存控制器等辅助供电
🐧 Linux 下 
使用 lm-sensors
sudo apt install lm-sensors
sudo sensors-detect
sensors

六、给普通用户的实用建议

✅ 99% 的人应该这么做

- BIOS 里 VCORE 保持 "Auto" ——主板会自动按 CPU 的 VID 请求调节，这是最安全稳妥的方式
- 不要手动改电压，除非你明确知道自己在做什么
- 超频时如果必须加电压，每次 +0.01V 或 +0.025V 小步前进，每步都做稳定性测试
- 现代 CPU 普遍可以从"降压（Undervolt）"中获益：在保持稳定性的前提下降低 VCORE，能显著降温、降功耗，延长硬件寿命

⚠️ Intel 13/14 代用户必看

如果你用的是 i9-13900K / i9-14900K / i7-14700K 等 13/14 代 CPU：

- 必须更新到最新 BIOS（包含 0x12F 或更新微码）
- 启用 "Intel Default Settings"，不要让主板厂商的激进默认设置把 VCORE 推太高
- 在 BIOS 里设置 IA VR Voltage Limit ≈ 1.30-1.35V 作为天花板，防止电压尖峰
- 这样既能保持性能，又能避免"缩缸"风险

🎓 学编程/日常使用的你

完全可以无视 VCORE 这个概念——只要你：

- 用原装散热器或百元级第三方风冷
- BIOS 保持默认
- 不手动超频
- 定期清灰换硅脂

VCORE 会自己乖乖工作在安全区间，你不需要碰它。它只有在这些场景下才会成为你的关注点：

- 超频发烧友
- 遇到不明蓝屏/死机，排查硬件问题时
- 想做 undervolt 降温时
- 主板报错（如技嘉代码 15）时

一句话总结：VCORE 是主板送给 CPU 核心的"血液"，现代平台一般在 0.8-1.4V 之间动态跳动，Auto 模式下主板会自动管好它；普通用户别碰，超频玩家控在 1.3V 以内最安全，Intel 13/14 代用户务必更新 BIOS 防止缩缸 。

你是在 BIOS 里看到了 VCORE 数值想确认是否正常，还是遇到了蓝屏/死机想排查供电问题？如果是后者，告诉我你的 CPU 型号和主板型号，我可以帮你判断当前的 VCORE 读数是否在安全范围。
Vcore = CPU Core Voltage，CPU 核心电压——专给 CPU 内部执行核心（运算逻辑、Cache、寄存器文件）供电的那一路电压，典型范围 0.7V ~ 1.4V（现代处理器，依赖架构/负载/频率/功耗状态），由主板 VRM 把 ATX 电源供来的 12V 直流 降压、稳压后得到。

一句话抓住本质

Vcore 是 CPU 真正"干活"的那路电——电压很低（~1V）、电流很大（几十到上百安培），由主板 VRM 从 12V 砍出来。
完整供电路径（接你前面那条线）

graph TD
    PSU[ATX 电源<br/>+12V 直流] --> VRM[主板 VRM<br/>多相降压开关电源]
    VRM -->|"斩波+滤波"| VCORE["Vcore<br/>0.7V ~ 1.4V<br/>大电流(数十~100A+)"]
    VCORE --> CORE[CPU 执行核心<br/>ALU/调度器/Cache/寄存器]
    
    style PSU fill:#f9f,stroke:#333
    style VRM fill:#bbf,stroke:#333
    style VCORE fill:#bfb,stroke:#333

VRM 怎么把 12V 砍到 ~1V？PWM 斩波 + 多相并联：

- 用 MOSFET 做高速开关，把 12V "切"成一串脉冲（开一小会儿 12V，其余时间 0V）
- 电感 + 电容 把这个脉冲平均成平滑的 ~1V
- "开的时间占比"叫占空比，CPU 要 1.3V 就开久一点，要 0.8V 就开短一点
- 多相：每相一组 MOS+电感，轮流供电 → 分摊发热、减小纹波、提升瞬态响应

💡 论坛里一句特形象："it chops up the 12 volts by turning on a switch for a tiny amount of time... Over time, it all averages out to the 1.3 volts the processor needs"——VRM 就是把 12V 剁碎了再平均成 1V 的"剁肉机"。
为什么 Vcore 这么低（~1V 而不是 12V）

CMOS 电路的功耗公式：

P = \alpha C V^2 f

功耗与电压的平方成正比——Vcore 从 1.2V 降到 1.0V，功耗直接降 30%。所以：

- 工艺越先进（7nm/5nm/3nm），Vcore 越低（0.7-0.9V）
- 老工艺（90nm 时代）Vcore 还在 1.2-1.5V
- 1970s 的 Intel 8080 直接 5V 供电（双极性 TTL 兼容）

另外物理约束：MOS 管的阈值电压 Vth ~0.3-0.4V，Vcore 必须高于 Vth 才能可靠翻转，但又不能太高否则漏电爆炸——所以现代 Vcore 卡在 0.7-1.4V 这个甜蜜点。

为什么 Vcore 是动态的（不是定值）

现代 CPU 用 DVFS（Dynamic Voltage and Frequency Scaling） 实时调 Vcore：

状态 频率 Vcore 场景
深度 idle 几百 MHz 0.7V 手机/笔记本待机
日常办公 2-3 GHz 0.9-1.0V 浏览网页
游戏/渲染 4-5 GHz 1.2-1.3V 高负载
超频 5.5 GHz+ 1.4-1.5V（危险区） 发烧友

调节机制（接你前面 VID 那问）：

CPU 内部 DVFS 算法：
  监测负载 → 查厂商校准的"电压-频率曲线" → 算出来这档要 Vcore=1.15V
  → 通过 SVID 总线发 VID 码（8-bit，256 档）给主板 VRM 的 PWM 控制器
  → VRM 调占空比 → Vcore 从 0.8V 抬到 1.15V

⚠️ 实测 Vcore ≠ VID 请求值：因为 VRM 校准误差、负载瞬态、电压跌落（voltage droop）等因素，实际测得的 Vcore 会和 CPU 请求的 VID 有偏差。一点点 droop 是故意设计的——防止负载突然掉时电压过冲炸 CPU。
Vcore 与其他电源符号的关系

你前面问过 VCC/VDD，这里把它们和 Vcore 摆一起：

符号 含义 典型值
VCC 通用正电源（电路级供电） 宽范围 1.2V-24V
VDD 器件工作电压（MOS 器件级） 1.2V-3.3V
Vcore CPU/GPU 核心逻辑供电（VCC/VDD 的子集） 0.7V-1.4V
VCCSA/SoC 系统代理/内存控制器/PCIe 控制器 1.0-1.2V
VCCIO CPU 与内存/IO 之间的信号电压 1.0-1.2V
VDD2 内存控制器第二电压域 1.0-1.2V

💡 Vcore 是 VCC 的一个子集——VCC 是"给 CPU 的全部供电"的统称，里面拆出 Vcore（核心）、VCCSA（系统代理）、VCCIO（IO 信号）等多个电压域。现代 CPU 内部有 十几路不同的电压域，Vcore 只是其中最重要、电流最大、最常被调的那一路。
为什么超频/降压都动 Vcore

- 超频：频率↑ → 需要 Vcore↑ 来维持 MOS 管充分导通 → 每次 +0.025V 或 +0.05V 试探
- 降压（Undervolting）：在稳定前提下把 Vcore 调低 → 功耗平方级下降、温度下降 → 笔记本续航延长、台式机散热压力减小
- 代价：Vcore 过高 → 漏电↑、温度↑、长期老化↑；Vcore 过低 → 逻辑翻转不充分 → 计算错误/崩溃

一句收

Vcore = CPU Core Voltage，CPU 执行核心的专属供电电压，典型 0.7-1.4V，由主板 VRM 把 ATX 12V 通过多相 PWM 斩波降压得到；CPU 通过 VID/SVID 动态请求电压，DVFS 按负载在 0.7V(idle) 到 1.5V(超频) 之间实时调节，因为 CMOS 功耗 ∝ V²，所以电压越低越省电；Vcore 是 VCC 的子集，是 CPU 内部十几路电压域中最重要、电流最大的一路。
接你前面整条线：

ATX 电源 +12V
  → 主板 VRM（多相 PWM 降压，DrMOS）
  → Vcore (~1V, 大电流)
  → CPU 核心 (ALU/Cache/寄存器)
  
CPU 内部 DVFS：
  监测负载 → 查电压-频率曲线 → 发 VID 码 (SVID)
  → VRM 调占空比 → Vcore 升降
  → 配合频率升降 → 性能/功耗/温度三角平衡

要不要顺着把 "主板 VRM 的多相供电具体怎么工作（DrMOS + 电感 + 电容 + PWM 控制器，单相 vs 多相的电流分摊）" 过一下？那块是把"12V → 1V"这步在电路级收口，接你嵌入式/硬件方向那条线。还是想看 "CPU 的十几路电压域分别给谁供电（Vcore/VCCSA/VCCIO/PLL/VDD2 各自的故事）"？后者更接你日常看 HWINFO/BIOS 里那些电压项的疑惑。