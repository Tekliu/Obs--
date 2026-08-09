**GRUB**（GRand Unified Bootloader） 是 Linux 世界**最主流的 bootloader**，工作在 BIOS/UEFI 之后、操作系统内核之前，负责把内核从磁盘加载到内存并启动。

在启动链中的位置
kernel 内核
BIOS/UEFI → GRUB → [[Linux Kernel]] → init/systemd → 用户空间

UEFI 主板会从 [[EFI]] 分区读取 
"grubx64.efi"，BIOS 主板则读取磁盘开头的 
"core.img"。GRUB 接管后，呈现启动菜单，选定项后加载 
"vmlinuz" 内核和 
"initramfs" 到内存，跳转到内核入口。

两大版本

- GRUB Legacy（0.9x）：早期版本，配置简单但功能有限，已淘汰
- GRUB 2（2.x）：现在说的"GRUB"默认指它，模块化设计，支持脚本、主题、加密、网络启动等

核心文件结构

/boot/grub/
├── grub.cfg          # 主配置文件（自动生成，勿手动改）
├── grubenv           # 环境变量（记录上次启动项等）
└── x86_64-efi/       # EFI 平台下的模块
/etc/default/grub     # 用户友好配置入口
/etc/grub.d/          # 生成 grub.cfg 的脚本片段

修改配置后必须执行：

sudo update-grub        # Debian/Ubuntu
sudo grub-mkconfig -o /boot/grub/grub.cfg   # Arch 等

关键能力

1. 多系统引导 —— 一份 GRUB 菜单可以同时启动 Linux、Windows、BSD 等
2. 内核参数编辑 —— 启动菜单按 
"e" 临时修改 
"quiet splash"、
"init=/bin/bash" 等
3. 救援模式 —— 内核崩了可以从菜单进 recovery 或使用 
"grub-rescue>"
4. 主题与美化 —— 支持自定义背景、字体、颜色
5. LUKS 解锁 —— 配合 
"cryptodisk" 模块启动加密根分区

常见实战场景

双系统重装 Windows 后 Linux 引导丢失：

# Live USB 启动后 chroot 修复
sudo mount /dev/sdaX /mnt    # 挂载根分区
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
sudo update-grub

忘记 root 密码：启动菜单按 
"e"，在 
"linux" 行末尾加 
"init=/bin/bash"，启动后 
"passwd root"。

默认启动项调整：编辑 
"/etc/default/grub"：

GRUB_DEFAULT=0            # 默认第一项
GRUB_TIMEOUT=5            # 菜单等待 5 秒
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

💡 GRUB 的 
"grub.cfg" 是自动生成的，手动改了也会被 
"update-grub" 覆盖；要持久化修改就去改 
"/etc/default/grub" 和 
"/etc/grub.d/" 下的脚本。
你是想了解 GRUB 配置美化、双系统引导修复，还是 手动写 GRUB 启动项？方向不同能给的代码差别很大。