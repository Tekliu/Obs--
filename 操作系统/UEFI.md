1998 年，Intel 给自己安腾（Itanium，IA-64）做固件时发现：16 位 BIOS 在 64 位 CPU 上根本跑不起来，必须重搞。于是 Intel 搞了个 EFI (Extensible Firmware Interface) 1.0。

2005 年，Intel 把 EFI 交给业界，成立 UEFI Forum（成员：Intel、AMD、Microsoft、Apple、HP、Dell…），改名 UEFI (Unified EFI)，版本从 2.0 起走。

关键时间节点：

- 2006 Mac 率先用 EFI 装 x86 → 苹果带头吃螃蟹
- 2011 Win8 强制要求 Secure Boot 认证 → 倒逼主板厂全线切 UEFI
- 2013 后 新 PC 基本默认 UEFI + [[GPT]]，Legacy BIOS 只剩 CSM 兼容模式

所以 UEFI 本质是 Intel 为了 IA-64 救场，被微软借着 Win8 推广成行业标准的。