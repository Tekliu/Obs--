POST  **Power-On Self-Test**（上电自检） 的缩写，属于[[固件启动链]]里最早、最底层那段——芯片复位后第一条指令开始，到"能去找启动盘"之前，全算 POST 范畴。

🔌 POST 在启动链的位置

按下电源 → 主板供电稳定（PCH/EC 发 POWER_OK） → CPU 复位，CS:IP 指向 0xFFFFFFF0（就是 ROM 里固件的入口） ├── 【POST 开始】 │ ├── SEC：做最早期初始化（Cache-As-RAM、定位固件镜像） │ ├── PEI：Pre-EFI Init，DRAM 初始化、把内存跑通 │ └── DXE：Driver Exec Env，枚举 PCI/USB/SATA、ACPI 表、SMBIOS ├── 【BDS】：挑启动项，找 ESP 里的 bootx64.efi └── 交权给 bootloader → 退出 POST 

Legacy BIOS 时代整个这一段笼统叫 POST；UEFI 规范把它拆得更细（SEC / PEI / DXE），但大家口语里还是常说"进 POST""卡 POST"。

🛠 POST 具体在干啥

按时间顺序大概是这样：

1. 最早期（SEC 阶段，CPU 刚醒）

执行复位向量那条指令（存在 SPI Flash / ROM 里，映射在 4GB 顶端）

关中断、建栈——但 DRAM 还没起来，栈只能放 CPU L2/L3 Cache 里（Cache-As-RAM，CAR），这是 Intel/AMD 固件里很经典的一个 trick

定位固件镜像主体，跳进 PEI

2. 内存起来前（PEI 阶段）

初始化 DRAM 控制器——这一步最关键也最容易"卡 POST"（内存条没插好、频率不对、XMP 翻车，都是这里崩）

跑内存训练（training）：调时序、校准 DQ/DQS，让信号眼图张开

测出可用内存大小，建个最简易的 HOB（Hand-Off Block）传给下一阶段

3. 设备枚举（DXE 阶段，这部分最长）

扫 PCI/PCIe 总线，分配 IO/MEM 空间、IRQ

初始化 USB、SATA/NVMe、网口、显卡 OPROM

建 ACPI 表（给后面 OS 用的，DSDT/SSDT/FADT 都在这里拼）

建 SMBIOS 表（dmidecode 能看到的那堆"BIOS 信息"）

跑完所有 gEfiDriverBindingProtocolGuid 的驱动

4. BDS → 准备交权

按 NVRAM 里的 BootOrder 找启动项

挂 ESP 分区，加载 \EFI\...\bootx64.efi

调用 ExitBootServices() 之前，UEFI Boot Services 还能用（所以 bootloader 能靠 UEFI 读文件、画界面）

⏱ 为什么 Legacy BIOS 的 POST 那么慢

老 BIOS 是纯串行的，而且带一堆历史包袱：

每扫一个设备都要 delay() 等几毫秒（ISA 时代留下的习惯）

键盘控制器要等 5 秒"没键按下"才算过

内存检测是 逐地址写读校验，1 GB 都扫半天，更别说 32 GB

显卡 OPROM、IDE 延时……叠起来开机 30 秒起步

UEFI 的改进：

DXE 驱动模型替代 INT 10h/13h 那套，逻辑更干净

内存不做全检（Fast Boot 模式下跳过），只训一部分

部分初始化可以延迟到 Runtime（比如某些 USB 设备等 OS 驱动来接）

所以"按电源 → 进 grub" UEFI 能做到 2–3 秒（台式机开 Fast Boot 甚至 1 秒多点）

🚨 "卡 POST"一般卡在哪

民间说的"卡 POST"多半指DRAM 初始化失败或PCI 设备 OPROM 挂了，表现就是：

屏幕没信号 / 停在主板 LOGO

debug 卡跑某个固定 code（A0、55、D6 这些，各家含义不一样）

风扇转、键盘灯不亮、无限重启

常见原因：内存条没插紧、XMP 电压不够、PCIe 设备短路、BIOS 设置翻车。Clear CMOS（扣电池 / 跳线）能救回来，就是因为 NVRAM 里 Boot + SPD 配置被清空，回到最保守的 PEI 路径重训内存。

要不要接着往下拆？比如 SEC/PEI/DXE/BDS 每个阶段在 EDK2 源码里对应哪个目录（SecMain.c / PeiCore / DxeMain 那些），或者 ExitBootServices() 那一瞬间内核到底接住了什么——前者偏"怎么读 UEFI 代码"，后者偏"OS 启动链怎么接"。

