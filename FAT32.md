**FAT32（File Allocation Table 32）**​ 是一种老牌文件系统，1996 年随 Windows 95 OSR2 引入，至今仍活跃在 U 盘、SD 卡、EFI 系统分区（ESP）等场景中。

## 在启动链中的角色

```
磁盘 → EFI 系统分区 (ESP, FAT32) → GRUB 的 grubx64.efi → Linux Kernel
```

UEFI 规范要求 ESP **必须是 FAT32**（或较旧的 FAT16），因为 UEFI 固件自身只内置了对 FAT 家族的驱动。所以你装双系统时，`/boot/efi` 那块几百 MB 的分区几乎一定是 FAT32。

## 核心特性

- **无权限模型**​ —— 不支持 Unix 文件权限（chmod/chown 无效），无用户/组概念
    
- **无日志**​ —— 意外断电容易丢数据或产生交叉链路
    
- **跨平台**​ —— Windows / Linux / macOS / 嵌入式设备全认，是"万能交换格式"
    
- **结构简单**​ —— bootloader 级别很容易实现读写，所以 U-Boot、UEFI 都原生支持
    

## ⚠️ 关键限制：4GB 单文件上限

这是 FAT32 最广为人知的痛点：

```
最大单文件 = 2³²-1 bytes ≈ 4 GiB - 1 byte
最大卷容量  = 2 TiB（Windows 格式化工具限制在 32 GiB）
```

意味着：

- 单个大于 4GB 的 ISO 镜像无法放进 FAT32 U 盘
    
- 大型虚拟机镜像、高清视频文件会写不进去
    

> 💡 制作 Ventoy 启动盘时，Ventoy 通过特殊手段绕过了这个限制，但原生 FAT32 是无解的。

## Linux 下的常用操作

**格式化**：

```bash
sudo mkfs.vfat -F 32 /dev/sdX1     # 格式化为 FAT32
sudo fatlabel /dev/sdX1 MY_USB      # 设置卷标
```

**挂载**：

```bash
sudo mount /dev/sdX1 /mnt/usb -o uid=1000,gid=1000,umask=022
# uid/gid 让普通用户拥有读写权限，umask 控制权限掩码
```

**检查修复**：

```bash
sudo fsck.vfat /dev/sdX1
```

**查看 FAT32 分区信息**：

```bash
sudo fdisk -l /dev/sdX
sudo blkid /dev/sdX1
```

## FAT32 vs 其他文件系统

|文件系统|最大单文件|日志|跨平台|典型用途|
|---|---|---|---|---|
|**FAT32**​|4 GB|❌|全平台|U 盘、ESP、嵌入式|
|**exFAT**​|16 EB|❌|Win/Linux/macOS|大容量 U 盘、SD 卡|
|**NTFS**​|16 TB|✅|Win 原生，Linux 只读需 ntfs-3g|Windows 系统盘|
|**ext4**​|16 TB|✅|Linux 原生|Linux 根分区|

## 实战场景

**制作 EFI 系统分区**：

```bash
# 创建 ESP 分区（通常 512MB，类型 ef00）
sudo mkfs.vfat -F 32 /dev/nvme0n1p1
sudo mount /dev/nvme0n1p1 /boot/efi
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi
```

**U 盘跨平台交换数据**：

```bash
sudo mkfs.vfat -F 32 -n DATA /dev/sdb1  # 4GB 以下文件随意拷
```

> 📌 **选型建议**：U 盘 ≤ 32GB 且需要兼容老设备 → FAT32；U 盘 > 32GB 或有大文件 → exFAT；纯 Linux 环境 → ext4。

你是想在 **U 盘上做启动盘**、**配置 EFI 分区**，还是 **解决大文件拷贝受限**​ 的问题？告诉我具体场景我能给到更精确的操作步骤。