---
data: 2026-01-08T12:23:00
tags:
---
1. 摘要核心
	- 痛点
		- GAN 声码器虽快、省内存，但音质仍落后于自回归/流式模型
	- 目标
		- 同时实现“高效 + 高保真”：22.05 kHz 音频，单 V100 推理 **167.9×实时**
	- 关键思路
		- 语音由多种周期正弦信号组成 → **显式建模周期模式**是提升品质的核心
		- 提出周期感知模块 + 多尺度判别器，捕捉不同周期/频率成分
	- 结果亮点
		- 单说话人 MOS **接近人类录音**，速度 167.9× 实时（V100）
		- 零-shot 泛化：未见说话人 mel 反演、端到端 TTS 均保持高自然度
		- 小 footprint 版：**CPU 13.4× 实时**，质量媲美自回归对手，适合边缘部署
	- 开源
		- 代码与预训练模型已发布，后续被广泛作为“mel → 波形”标准声码器
2. 引言要点
	- 背景
		- 二阶段流水线：文本→mel→波形，声码器决定最终音质与速度
		- 自回归 WaveNet：质量好，但样本级生成 → 极慢
		- 并行替代：
			- Flow-based: Parallel WaveNet（需蒸馏）、WaveGlow（90+层，参数量大）
			- GAN-based: MelGAN、GANT-TPS 可实现 CPU 实时，但音质仍落后 AR/Flow
	- 核心观察
		- 语音 = 多周期正弦叠加 →**显式建模周期模式**是提升感知质量的关键
	- HiFi-GAN 创新
		- 多周期子判别器（Multi-Period Discriminator）
			- 每个子判别器只抓特定周期片段 → 强迫生成器同时匹配多种周期统计
		- 生成器并行多尺度残差块→ 一次前向观察不同长度模式，配合判别器周期先验
		- 大/小/极速三档模型
			- 默认：V100 **3.7 MHz**（≈167×实时）MOS优于 WaveNet/WaveGlow
			- Tiny 版：**0.92 M 参数**，质量仍领先公开模型
			- Fast 版：CPU **13.4×实时**、V100 **1,186×实时**，音质媲美自回归
	- 结论
		- 首次让 GAN vocoder 在**音质+速度**双重指标上**同时超越 AR 与 Flow**
		- 周期感知架构成为后续多声码器（BigVGAN、Avocodo 等）通用设计
3. 生成器与 MRF 模块
	- 生成器总览
		- 全卷积结构；mel-spectrogram 为唯一输入，无额外噪声
		- 逐级上采样：转置卷积组，总倍数 $k_u$（可调变量）
		- 每级转置卷积后 → MRF 模块 → 输出与波形时序对齐
	- MRF 模块（图 1 细节）
		- 并行 $|k_r|$ 个残差块，输出相加
		- 单块结构：
			- 1×1 投影（通道压降）
			- 3×1 扩张卷积， dilation 取集合 $D_r[n]$（变量）
			- 跳跃连接回输入
		- 不同 $\{k_r, D_r\}$ 组合 → 同时捕获短/中/长时模式，增强周期建模
	- 可调变量
		- $h_u$：隐藏通道数
		- $k_u$：每级上采样率
		- $k_r$：残差块内核大小集合
		- $D_r$：对应扩张率集合
		- 作者通过网格搜索 + MOS 确定默认数值，实现速度/质量权衡
	- 效果
		- MRF 为生成器注入“多周期”先验，配合多周期判别器，使 GAN 首次在 MOS 上超越 WaveNet/WaveGlow，同时保持 100× 级实时速度
4. 判别器
	- 设计目标
		- 同时捕捉**周期模式**（MPD）与**连续长程依赖**（MSD）
	- Multi-Period Discriminator (MPD) 
		- 周期集合：p ∈ {2, 3, 5, 7, 11}（互质，避免重叠）
		- 单个子判别器流程（图 2b）
			- 将 1D 音频长度 T → 重塑为 2D 形状 (T/p, p)
			- 2D 卷积，**kernel 宽轴恒为 1** → 仅沿时间轴滑动，各周期通道独立处理
			- 堆叠步长卷积 + LeakyReLU，末端 weight norm
		- 梯度可回传至**所有时间步**，非采样，保证全局更新
	- Multi-Scale Discriminator (MSD)
		- 三级输入（图 2a）
			- raw 音频
			- ×2 平均池化
			- ×4 平均池化
		- 每级子判别器：
			- 步长/分组卷积 + LeakyReLU
			- **raw 级用 spectral norm**，其余 weight norm  
				- 权重归一化
					- 把卷积核向量 w 分解成 $w = g · v/‖v‖$，重新参数化
					- 训练时只优化标量 g 与方向 v，使梯度传播更平滑
					- 不限制 $Lipschitz$ 常数，适合中间层，计算量小
				- 谱归一化
					- 每步前向时把核矩阵 W 除以其最大奇异值 $σ_1$，使 $‖W‖₂ ≈ 1$
					- 强制满足 $1-Lipschitz$，防止判别器梯度爆炸/模式崩塌
					- 计算稍贵，对敏感的“原始音频”子判别器使用，可抑制训练震荡
				- 总结
					- raw 级（最细粒度、最易不稳定）→ 用 spectral norm 强约束
					- 池化后级（信号已平滑）→ 用 weight norm 轻量级归一化
	- 协同作用
		- MPD：抓离散周期统计 → 抑制机械谐振
		- MSD：抓平滑长程波形 → 抑制块状/喀哒 artifact
		- 联合训练 → 生成器被迫同时满足“周期正确 + 连续自然”，实现 GAN vocoder 首次 MOS 超 WaveNet
5. 训练损失
	- （1）GAN 损失（LSGAN）
		- 判别器：
		  $$
			  \mathcal{L}_{\text{adv}}(D) = \mathbb{E}_r[(D(r)-1)^2] + \mathbb{E}_s[D(G(s))^2]
			  $$
		- 生成器：
		  $$
			  \mathcal{L}_{\text{adv}}(G) = \mathbb{E}_s[(D(G(s))-1)^2]
		  $$
		  其中 $r$ 为真波形，$s$ 为输入 mel 条件
	- （2）Mel-Spectrogram 损失
		- L1 距离：
		  $$
			  \mathcal{L}_{\text{mel}}(G) = \mathbb{E}_{r,s}\left[\|\phi(r) - \phi(G(s))\|_1\right],\quad \phi=\text{mel-transform}
		  $$
		- $\phi$：transforms a waveform into the corresponding mel-spectrogram
		- 作用：早期稳定+感知对齐；权重 $\lambda_{\text{mel}}=45$
	- （3）Feature Matching 损失
		- 对每段子判别器 $D_k$ 逐层特征 L1
		  $$
			  \mathcal{L}_{\text{fm}}(G) = \sum_{k=1}^K\sum_{i=1}^{T_k} \frac{1}{N_{k,i}}\|D_k^{(i)}(r) - D_k^{(i)}(G(s))\|_1
		  $$
		  where $T$ denotes the number of layers；$D^{(i)}$, $N_i$ denote the features and the number of features in the i-th layer
		  权重 $\lambda_{\text{fm}}=2$ 
	- （4）总目标
		- 生成器：
		  $$
			\mathcal{L}_G = \sum_{k=1}^{K} \Bigl[ \mathcal{L}_{\text{adv}}(G; D_k) + \lambda_{\text{fm}} \mathcal{L}_{\text{fm}}(G; D_k) \Bigr] + \lambda_{\text{mel}} \mathcal{L}_{\text{mel}}(G)
			$$
		- 判别器：
		  $$
			  \mathcal{L}_D = \sum_{k=1}^K \mathcal{L}_{\text{adv}}(D_k)
		  $$
		- $K$ 为 MPD + MSD 子判别器总数；各损失权重固定，无需调参即可稳定收敛
6. HiFi-GAN 实验设置
	- 数据集
		- LJSpeech: 22 kHz，24 h，单说话人；
		- VCTK: 44 kHz→22 kHz，44 h，109 人，留 9 人做 unseen 测试  
		- 对比基线：MoL-WaveNet、WaveGlow、MelGAN，均用官方预训练权重
	- MOS 测试
		- 平台：Amazon Mechanical Turk，5 分制，95 % CI
		- 音量归一化，每样本仅评一次，防止记忆偏差
	- 速度测量
		- 设备：单 NVIDIA V100 & MacBook Pro i7 2.6 GHz
		- 精度：32-bit 浮点，无额外优化，保证公平
	- 生成器三档配置
		- | 版本 | $h_u$ | $k_u$ | $k_r$ | $D_r$ | 说明 |
		  |---|---|---|---|---|---|
		  | V1 | 512 | \[16,16,4,4] | \[3,7,11] | \[\[1,1],\[3,1],\[5,1]]×3 | 默认大模型 |
		  | V2 | 128 | 同 V1 | 同 V1 | 同 V1 | 窄通道，参数量↓ |
		  | V3 | 128 | 精简层数 | 自定义 | 自定义 | 保持感受野，层数最少 |
	- 训练超参
		- 输入：80-band mel，FFT 1024 / window 1024 / hop 256
		- 优化器：AdamW $(β_1=0.8, β_2=0.99, λ=0.01)$ 
		- LR：初值 $2×10^{-4}$，每 epoch ×0.999 衰减
		- 总步数：2.5 M（VCTK）/ 250 k（LJSpeech），与基线对齐
7. 结论
	- 核心成果
		- 音质：超越当时最佳公开模型（WaveNet/WaveGlow），MOS 接近人类录音
		- 速度：单 V100 最高 **1 186×实时**；CPU 小模型 **13.4×实时**，首次实现“GAN 声码器在音质与效率双重维度同时领先”  
	- 关键验证
		- 多周期判别器（MPD）是质量提升核心，消融实验去除后 MOS 显著下降
		- 零-shot 泛化：未见说话人 + noisy Tacotron2 mel → 微调后仍达人类水平
		- 小 footprint 版仅 0.92 M 参数，**CPU 实时合成**，为终端侧低延迟、低内存部署铺平道路  
	- 通用性
		- 同一套 MPD+MSD 判别器即可训练 V1/V2/V3 三种生成器，无需重复调参 → 工程上“换生成器如换灯泡”  
	- 开源与影响
		- 代码与权重已发布，后续被直接作为 FastSpeech 2、VITS、BigVGAN 等项目的默认声码器，成为“mel → 波形”事实基准
