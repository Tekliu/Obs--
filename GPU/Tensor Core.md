## 张量核心
数量:每 SM 4 个
干啥：专啃 矩阵乘加 D = A × B + C
Volta(伏特)（Tensor Core 诞生）→ Ampere(安培)（TF32 普及）→ Hopper(霍普)（FP8 + Transformer Engine，大模型主力）→ Blackwell(布莱克维尔)（FP4 + 双 die 拼，万卡集群） 是数据中心那条线；
消费级对应 Turing → Ampere → Ada → Blackwell。每一代 SM 里"CUDA Core 数 / Tensor Core 代数 / RT Core 有无"都在变，但"SM 是 GPU 最小计算簇"这定位没变