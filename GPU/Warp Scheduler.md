数量：每 SM 4 个
干啥：SM 里最核心的控制件——每个 scheduler 管一组 warp（32 线程），每周期选一个 ready 的 warp 发指令下去上一轮聊到 SM（Streaming Multiprocessor）是 GPU 里"多"的那个单元，这次把 SM 肚子里装的部件拆开讲——NVIDIA 这边叫 SM，AMD 那边叫 CU（Compute Unit），结构大同小异，下面以 NVIDIA 为主。

SM 内部部件清单（以 Ampere / Ada 为参照）

一颗 SM 肚子里大致分 计算单元、调度单元、存储单元、特殊功能 四类：

一、计算单元（真正动手的）

1. CUDA Core（FP32 / INT32 / FP64）

数量：每 SM 128 个（Ampere 之后，Ada/Lovelace 同）

干啥：最传统的 ALU，浮点 + 整数

FP32：顶点变换、片元光照、传统 shader 的主力

INT32：地址计算、循环计数

FP64：双精度，消费级卡被砍得很惨（4090 每 SM 只有 2 个 FP64，是 FP32 的 1/64）——科学计算才用

你前面聊的图形管线：顶点着色器里 P·V·M·pos 的乘加，片元着色器的光照 dot(N,L)，基本都是 CUDA Core 在跑

2. Tensor Core（张量核心）

数量：每 SM 4 个（Volta 起引入，Ampere 第 3 代，Hopper 第 4 代，Blackwell 第 5 代）

干啥：专啃 矩阵乘加 D = A × B + C

Volta：4×4×4

Turing/Ampere：16×16×16 的 FP16 块一拍完

Hopper：加 FP8，Transformer Engine 能自动在 FP8/FP16 间切

为啥重要：深度学习的所有矩阵乘（Transformer 的 QKᵀ、FFN 的 Wx）全靠它，AI 爆发的硬件根基

你前面聊的 PyTorch/CNN/LLM：conv2d、linear 底层最终都落到 Tensor Core，比 CUDA Core 快几十倍

3. RT Core（光线追踪核心，消费级才有）

数量：每 SM 1 个（Turing 起）

干啥：BVH（包围盒层次）遍历 + 射线-三角形相交测试

图形侧：路径追踪、软阴影、反射；你玩的赛博朋克 2077 开光追就是它扛

二、调度单元（发号施令的）

4. Warp Scheduler（束调度器）

数量：每 SM 4 个（Ampere 后）

干啥：SM 里最核心的控制件——每个 scheduler 管一组 warp（32 线程），每周期选一个 ready 的 warp 发指令下去

SIMT 的载体：32 线程锁步走同一条指令，分歧就序列化

4 个 scheduler → 每 SM 每周期最多发 4 个 warp 的指令（每个 warp 32 线程 × 4 = 128 线程/周期理论上）

5. Dispatch Unit（分发单元）

配合 scheduler，把指令拆给 CUDA Core / Tensor Core / LD-ST 这些执行单元

三、存储单元（喂数据的）

6. Register File（寄存器文件）⭐ 非常重要

大小：每 SM 256 KB（Ampere）→ Ada 涨到 320 KB+

干啥：给 warp 里 32 线程存局部变量，每个线程能分到几十个寄存器

为啥关键：GPU 能养几万线程的秘诀——线程切换零开销，因为寄存器已经绑死在线程上，切 warp 不用刷（CPU 切换线程要存上下文到内存，慢死）

代价：如果一个线程用太多寄存器（比如 shader 里写了个大 struct），SM 能同时驻留的 warp 数就降 → occupancy 掉

7. Shared Memory / L1 Cache（共享内存 + 一级缓存）

大小：每 SM 128 KB（Ampere 起，L1 + Shared 可动态分，比如 100KB Shared + 28KB L1）

Shared Mem 干啥：同一线程块（block）里的线程共享，低延迟（比走显存快一个量级）

CUDA 编程里手动用 __shared__ 声明

经典场景：tile 矩阵乘，先把 A、B 小块搬进 Shared，块内线程复用

L1 干啥：自动缓存全局显存的读取，程序员不用管

8. L2 Cache（不在 SM 内，但贴着）

所有 SM 共享，Ada 4090 是 72 MB，Blackwell Ultra 单 die 有 128 MB（两颗 die 合计更大）

显存访问的最后一道缓冲

四、特殊功能

9. SFU（Special Function Unit）

每 SM 若干，算 sin / cos / exp / log / rsqrt 这些 transcendental

图形里法线变换、光照衰减常用；SFU 跟 CUDA Core 是分开的，不占 ALU 周期

10. Load/Store Unit（LD/ST）

每 SM 若干，管线程从显存/Shared/L1 搬数据

你 shader 里每写一个 texture(...) 或 buffer[] 访问，背后都是 LD/ST 在跑

把部件跟前面你聊的东西对一下

你之前聊的对应 SM 里谁在干顶点着色器 gl_Position = P·V·M·posCUDA Core（矩阵乘加）片元着色器光照、纹理采样CUDA Core + LD/ST + Texture CacheCNN 卷积、LLM 的 LinearTensor Core（主力）+ CUDA Core 补光线追踪反射/阴影RT Corewarp 32 线程锁步Warp Scheduler几万线程同时跑不卡Register File（零开销切换）tile 矩阵乘优化Shared Memory 

一句钉死 SM 的"部件哲学"

SM 不是"一个大力核"，是一个包工头（Warp Scheduler）+ 一堆便宜小学生（CUDA Core）+ 几个矩阵特种兵（Tensor Core）+ 一个超大作业本（Register File）+ 一个班组工具箱（Shared Mem） 凑成的计算簇——靠"人多 + 切换零成本 + 矩阵硬件加速"吃饭，不靠单核 IPC。

要不要接着走 Warp + SIMT 那条线（为啥 GPU 是"单指令多线程"、跟 CPU SMT 本质不同、分支发散 if 为啥会让 GPU 掉速）？那条走完，"SM 内部怎么跑你的 shader / CUDA kernel"就全通了。还是回图形侧把 P 矩阵 z 那两项 + z-fighting 收掉？

