IOMap = I/O Permission Bitmap（I/O 许可位图），躺在 TSS 末尾 那块可变长区域，是 x86 给 
"in" / 
"out" 指令（端口 I/O） 做的权限闸门。

为啥要有它

x86 有两种"跟外设打交道"的方式：

- MMIO（内存映射 I/O，把设备寄存器 map 到地址空间，用 
"mov" 访问）—— 权限走分页（页表 R/W 位控）
- Port I/O（端口 I/O，用 
"in"/
"out" 指令，0-0xFFFF 共 65536 个端口） —— 权限不走分页，走的是 CPL + IOPL + IOMap 这套

典型端口你肯定见过：

- 
"0x60/0x64" → PS/2 键盘/鼠标控制器
- 
"0x3F8/0x2F8" → COM1/COM2 串口
- 
"0x1F0/0x170" → IDE 硬盘（老）
- 
"0xCF8/0xCFC" → PCI 配置空间访问（CONFIG_ADDRESS/CONFIG_DATA）

这些端口内核能随便 in/out（CPL=0），但用户态（CPL=3）默认不能——闸门就是 IOMap。

查权限的完整逻辑（CPU 执行 
"in"/
"out" 时）

in al, 0x60   ← 用户态想读键盘端口
  ↓
CPU 看两样：
  1. CPL（当前权限，用户态=3，内核态=0）
  2. IOPL（存在 EFLAGS[12:13]，一般 Linux 设成 0）
  
if CPL ≤ IOPL:
   放行（内核态 CPL=0 ≤ IOPL=0 → 直接过）
else:
   → CPL=3 > IOPL=0 → 得查 IOMap
   → 端口号 N → 查 bitmap 第 N 位
      bit=0 → 放行
      bit=1 → #GP(0) 异常，用户进程收到 SIGSEGV/SIGBUS

💡 所以 IOPL 是"粗闸门"（CPL≤IOPL 全放行），IOMap 是"细闸门"（端口级 0/1），两层叠着用。

IOMap 在 TSS 里的位置

TSS 结构（32 位简化，x64 类似但更大）：

TSS 开头（固定 104 字节）
├── 前 28B：Link(旧 TSS 选择子)
├── SS0/ESP0（内核栈，最重要的那挂）
├── SS1/ESP1（Ring1，基本废）
├── SS2/ESP2（Ring2，基本废）
├── CR3（页表根，硬件任务切换用，Linux 不用）
├── EIP/EFLAGS/EAX..EDI/ES..GS 通用寄存器镜像（硬件切换用）
├── LDTR 选择子
├── I/O Map Base Offset  ← ⭐ 关键：指向 bitmap 起始
└── [IOMap 从这里开始，可变长，最长 65536/8 = 8192 字节]
    └── TSS 末尾还要跟 1 字节 0xFF 作"越界哨兵"

- I/O Map Base 字段（TSS 偏移 0x66 处，32 位）= bitmap 在 TSS 内的偏移
- 如果 I/O Map Base ≥ TSS Limit → 表示"没 bitmap" → CPL> IOPL 时 所有 in/out 全 #GP
- Linux 默认就是这状态——TSS 里 I/O Map Base 标成 >=Limit，所以用户态 
"in"/
"out" 直接 GP，除非进程调 
"ioperm()"/
"iopl()" 显式开

bitmap 自身怎么映射端口

1 bit = 1 个端口，0=允许，1=拒：

端口 0 → bitmap[0] 的 bit0
端口 1 → bitmap[0] 的 bit1
...
端口 7 → bitmap[0] 的 bit7
端口 8 → bitmap[1] 的 bit0

例子：想让用户态能访问 0x60（键盘）和 0x64（键盘控制），其他端口拒：

bitmap[0x60>>3] 的 bit(0x60&7)=bit0 → 0
bitmap[0x64>>3] 的 bit(0x64&7)=bit4 → 0
其他全 1

x64 下端口空间还是 64K（0-0xFFFF），所以 bitmap 最大 65536 bit = 8192 字节；TSS 末尾还要加 1 字节 0xFF 当哨兵——CPU 查 bitmap 时如果端口号超了 bitmap 长度，会去读这字节，全 1 ＝ 全拒，防止越界。

Linux 下实战：谁会碰 IOMap

普通用户进程 
"in al, 0x60" → 立刻 #GP → SIGSEGV，因为：

- CPL=3，IOPL=0 → 要查 IOMap
- Linux TSS 里 I/O Map Base 默认标"无"或 bitmap 全 1 → 全拒

两种情况会开：

1. 
"ioperm(start, len, on)" 系统调用（只 root 或 CAP_SYS_RAWIO）
   - 改当前进程 TSS 里的 bitmap 对应位，只开一段端口
   - 比如 Xorg 早期要碰 0x3B0-0x3DF（VGA 寄存器），就 
"ioperm()" 开这一段
2. 
"iopl(3)" 系统调用（root）
   - 直接把当前进程的 EFLAGS[IOPL] 从 0 改成 3 → 所有端口全放行，不用查 bitmap
   - 危险，root 才干，比如老 
"hdparm"、
"setserial" 这类工具

💡 虚拟机 / 容器里 
"iopl()" 经常被 seccomp / capabilities 拦掉，因为开了能让用户态直接摸主板寄存器（SMI、CMOS、RTC…），安全风险大。

跟 x64 那问的衔接

x64 下 TSS 还在、IOMap 机制没变——但：

- 64 位 Linux 基本没人用 
"iopl()" 了（MMIO + 
"devmem" / 
"pci_mmap_io" 替代）
- 用户态碰端口的场景只剩：老 Xorg 驱动、嵌入式板子调串口/并口、硬件 hacking 工具
- 所以 IOMap 在 x64 下是"规范还活着、实战凉了" 那挂，跟 LDT 类似

一句收：IOMap = TSS 末尾的端口许可 bitmap（1 bit/端口，0 放 1 拒），CPL> IOPL 时 CPU 查它决定 
"in"/
"out" 是否 #GP；Linux 默认全 1 拒，root 用 
"ioperm()"/
"iopl()" 开，x64 下机制还在但实战基本凉。

要不要顺着把 "
"in"/
"out" vs 
"mmap(/dev/mem)" 这两种"用户摸硬件"路径的区别" 过一下？那块接你"底层→找工作"那条线（驱动岗常撞 
"mmap" 走 MMIO，端口 I/O 基本被 MMIO 替了，除了老 VGA/串口）。还是回 
"syscall" 那条——"Ring3→Ring0 那几纳秒 CS/SS/ESP0/GS 怎么跳"？后者更接你 CS:IP→GDT→TSS→IOMap 这条刚搭完的线。