TSS = Task State Segment（任务状态段），是 x86 专门为**任务切换设计的一块内存数据结构**，存在内存里，GDT 里挂一条"TSS 描述符"指着它，CPU 有个**专用寄存器 TR（Task Register）** 存"当前 TSS 的选择子+基址"。

它最初想干啥（Intel 80386 的野心）

80386 引入保护模式时，Intel 想让 "任务切换"硬件化——每个任务（进程）配一个 TSS，切任务时 CPU 硬件自动把：

- 所有通用寄存器（EAX~EDI）
- 段寄存器（CS/DS/ES/FS/GS/SS）
- IP / FLAGS
- 栈指针（[[ESP and SS]]）
- CR3（页表根，隔离进程地址空间）
- LDT 选择子
- [[IOMap]] 偏移（管 in/out 指令权限）

全塞进旧任务的 TSS，再从新任务的 TSS 把这一坨恢复出来 —— 一条 
"jmp TSS选择子" 或 
"call TSS选择子" 就完成切换，OS 不用手写调度器。

听起来美，但实际没人用——原因：

- 硬件切换不灵活（Linux 的 
"schedule()" 要跑调度算法、要选下一个 task、要记 cgroup、要记优先级，硬件那套给不了）
- 慢（要存/恢复几十个寄存器，Cache 全刷）
- 多核下 TR 每核一个，硬件 TSS 切换跟 per-CPU 冲突
- 所以 Linux/Windows 从第一天就没用硬件 TSS 切换，改用软件调度 + 只留 TSS 干两件事。

TSS 现在实际干啥（32 位 + x64）

1️⃣ 存 Ring 3 → Ring 0 的栈信息（最重要）

用户态程序执行 
"syscall" / 
"int 0x80" / 异常 进内核时，CPU 要立刻换栈（不能再用用户栈，怕用户栈溢出/恶意）：

TSS 里关键字段：
  SS0   → 内核数据段选择子（一般 0x10）
  ESP0  → 内核栈顶指针（每个任务不一样，Linux 每 sched 一次就改一次 ESP0）

流程：

- 用户态 ESP 在 3GB 附近（用户栈）
- 一进内核，CPU 自动从 TSS.SS0 / TSS.ESP0 拿内核栈 → 切到内核栈 → 再 push 用户寄存器等
- 所以 TSS.ESP0 必须实时跟当前 running task 对齐——Linux 里 
"switch_to()" 末尾会 
"mov esp0, next_task.stack"

2️⃣ 存 IOMap（I/O Permission Bitmap）

"in"/
"out" 指令访问端口（0x60 键盘、0x3F8 串口…）时，CPL ≤ IOPL 才放行；否则查 TSS 里的 IOMap——每端口 1 bit，0=允许，1=#GP。

- 一般用户态 CPL=3，IOPL=0 → 用户直接 
"in al, 0x60" 会 #GP
- root 改 TSS.IOMap 或提 IOPL 才能放

3️⃣ x64 下多了 IST（Interrupt Stack Table）

x64 下 TSS 结构扩了，加了 7 个 IST 槽（Interrupt Stack Table）：

TSS(x64):
  ...
  IST1 → 异常 #1（DB）专用栈
  IST2 → NMI 专用栈
  IST3 → 双重故障专用栈
  ...

用途：某些异常不能复用当前栈（比如 NMI 打在内核栈上可能把栈踩了，双重故障更甚），IDT 每条门描述符里可以标 
"IST=n"，CPU 进异常时不压当前 RSP，直接换 IST[n] 那根栈。

💡 Spectre/Meltdown 之后 IST 被提得更多——因为中断栈隔离变得更敏感，x64 这设计比 32 位稳。

TSS 跟 GDT/TR 的关系（接你上一问）

graph TD
    A[TR 寄存器<br/>存"当前 TSS 选择子+基址"] -->|CPU 查| B[GDT]
    B -->|GDT 里一条<br/>Type=S=0, Type=9/11| C["TSS 描述符<br/>Base/Limit/DPL"]
    C -->|Base 指| D[TSS 结构体<br/>ESP0/SS0/IOMap/IST[x64]]
    
    style A fill:#f9f,stroke:#333
    style C fill:#bbf,stroke:#333

- TR = Task Register，类似 GDTR/LDTR，但 TR 是每核一个
- 
"ltr" 指令（Ring 0 专用）把 TSS 选择子 load 进 TR，顺便把 GDT[TSS索引] 的描述符缓存进 TR 的隐藏部分（跟 CS 的"可见部分16位+隐藏部分描述符缓存"同款思路）
- Linux 启动早期 
"setup_arch()" → 
"cpu_init()" → 
"ltr(INIT_TSS_SELECTOR)" 把 boot TSS 挂 TR 上，之后TSS 内容会动态改（ESP0 每 sched 跟新 task，IST 一般不动）

32 位 vs x64 下 TSS 的差异

 32 位 x64
原设计用途 硬件任务切换载体 同上（但没人用）
实际用途 ESP0/SS0 + IOMap ESP0/SS0 + IOMap + IST×7
大小 ~104 字节 ~256+ 字节（IST 占多）
每任务一个？ Linux 早期试过，后来改每 CPU 一个 每 CPU 一个 TSS，所有任务共享，只改 ESP0
TR 每核一个 每核一个

💡 "每 CPU 一个 TSS"是关键——TSS 不再是"任务镜像"，退化成"每核的内核栈入口+IOMap+中断栈表"。硬件 TSS 切换那条路 Intel 赌输了，但 TSS 这个结构体没废，缩水接着用。

一句收

TSS = 80386 想用来做"硬件任务切换"的那块内存结构，挂 GDT 里、TR 寄存器指着；但 Linux/Windows 都没用硬件切换，现在 TSS 只干三件事：Ring3→Ring0 栈切换（ESP0/SS0）、IOMap 管 in/out 权限、x64 下加 IST 给 NMI/双重故障独立栈。每 CPU 一个，不是每任务一个。
要不要顺着把 "
"syscall" 指令从 Ring3→Ring0 那几纳秒，CS/SS/ESP0 怎么从 TSS/MSR 里捞出来" 过一下？那块是"你敲个 
"read()" 进内核"最底层路径，接你 CS:IP→GDT→TSS 这条线正好收口。还是想先看 "Linux 
"switch_to()" 怎么改 TR 隐藏缓存 + ESP0"（C + 汇编那段，调度器底层）？后者更 OS 内核向，前者更 syscall 向。