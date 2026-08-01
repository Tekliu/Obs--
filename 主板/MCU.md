## MCU = Microcontroller Unit，微控制器，"8051 内核小单片机"的正式学名。MCU = 单片机，叫法不同而已，大陆叫单片机多，datasheet/规格书里统一写 MCU。
next: [[EC_eSPI]]
🎯 一句话定位

MCU = 把"**CPU 核 + RAM + Flash + 外设**"全塞一颗芯片里的 **SoC-lite**，**焊上电+晶振**就能跑，不用外接内存条、不用外接南桥。

对比你熟的几个：

 应用处理器 /[[ MPU ]](Application Processor) MCU
所以 MPU 是"要配外围才能活"，MCU 是"自己一颗就活"——EC 那颗就是后者。

⚙️ MCU 内部长啥样（以 STM32F103 为例）

┌─────────────────────────────────┐
│  Cortex-M3 核 (72MHz)            │ ← CPU
│   ├─ NVIC (中断控制器)            │
│   ├─ SysTick                     │
│   └─ MPU (内存保护单元, F 系有)   │
├─────────────────────────────────┤
│  片上 Flash: 64/128/256 KB       │ ← 存固件 (你的 .bin 烧这)
├─────────────────────────────────┤
│  片上 SRAM: 20 KB                │ ← 运行时 RAM (栈/堆/全局)
├─────────────────────────────────┤
│  外设 (全挂 APB/AHB 总线上)       │
│   ├─ GPIO (扫按键/点 LED)         │
│   ├─ USART / UART (串口打印)      │
│   ├─ [[I2C / SPI]] (接传感器/EEPROM) │
│   ├─ ADC (读电压/温度)            │
│   ├─ TIM (PWM 调风扇/舵机)        │
│   ├─ DMA (搬运不占 CPU)           │
│   └─ USB / CAN / ETH (高端才有)   │
├─────────────────────────────────┤
│  Debug: SWD/JTAG (接 ST-Link)    │ ← 烧录+单步
└─────────────────────────────────┘

关键特征：外设全走内部总线（AHB/APB）挂到核上，寄存器映射到固定地址，C 语言直接 
"*(volatile uint32_t*)0x40010800 = 0x01" 这种就能操控 [[GPIO]]——这就是"裸机 C"的精髓，跟你写 x86 UEFI 的 MMIO 思路一模一样，只是地址和寄存器定义不同。

🔗 回链你这条线里见过的 MCU

- EC (笔记本) → 大多是 8051 内核 MCU（ITE IT8586 / Nuvoton NPCE），
"+5VSB" 喂电，走 [[eSPI/LPC ]]跟 PCH 通信，管开机键/电池/风扇/键盘
- [[STM32]] (Cortex-M3/M4/M7) → 工业/消费电子绝对主力，你以后想自己写个 toy 固件从零跑，STM32F103 是入门标配
- [[ESP32]] → RISC-V 或 Xtensa 双核，Wi-Fi+BLE 集成，物联网向
- [[Arduino]] (AVR 内核) → 教学向，封装重但好上手
- PCH 里的 ME 核 → 其实也是一颗 ARC / RISC-V 小 MCU 跑在 PCH 里独立供电，关机也活（AMT/远程 KVM 靠它），本质也是 MCU 思路

📊 MCU vs EC vs 8051 vs STM32 的称呼关系

很多人混，给你理一下：

- MCU = 大类（微控制器）
- 8051 = 一种 MCU 内核架构（Intel 1980 年出的，现在 ITE/Nuvoton 出的"8051"是换皮加料版，内核名留着）
- STM32 = 一种 MCU 产品系列（ST 公司，Cortex-M 内核）
- EC = 一种 MCU 的用途/岗位（"Embedded Controller，笔记本里管杂活的那个 MCU"），EC 芯片本身可以是 8051 也可以是 ARM

所以"EC 是 8051 MCU"这句话拆开是：EC(岗位) = 8051(内核) MCU(大类)。

🛠 一个 C++ 背景的直觉类比

把 MCU 想成 "一台只有一颗芯片的裸机电脑"：

- 
"Cortex-M3" ≈ i9（但 72MHz，没 cache 没乱序）
- 
"片上 SRAM 20KB" ≈ 内存条（但塞在同一颗硅片里）
- 
"片上 Flash 128KB" ≈ SSD（但也是同一颗硅片里，XIP 直接跑）
- 
"GPIO/UART/I2C" ≈ 主板上的南桥功能（但也是同一颗硅片里）
- 
"裸机 while(1) 或 RTOS 任务" ≈ Linux 的 /init

你写 UEFI 的 
"SecEntry.S" 是跑在 MPU（x86 核） 上的，有 DRAM、有 PCI、有 PCH；EC 那颗写的是 MCU（8051 核） 上的固件，没 DRAM、没 MMU、寄存器直接 map——所以 EC 固件里全是 
"EC_REG_BASE + 0x12 = 0x03" 这种 magic number，跟 UEFI 的 
"MmioWrite32()" 思路一样但更裸。