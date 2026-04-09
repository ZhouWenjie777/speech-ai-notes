---
data: 2026-01-07T14:14:00
tags:
---
1. 文本正则化（同 tacotron，`text_to_sequence` 额外采用G2P模块）
	- （1）正则化
		- `text = _clean_text(text, cleaner_names)`
		- 同 Tacotron，得到小写无缩写的英文句子
	- （2）查词典（由 `dictionary` 即 CMUDict 给定）
		- `for w in words: arpa = dictionary.lookup(w) # ARPAbet序列 或 None`
		- `tokens += ["{"+arpa+"}"] if hit else [w]`
		- `例：hello world → "{HH AH L OW1} {W ER1 L D}"`
	- （3）ARPAbet → 音素
		- `sequence = _arpabet_to_sequence(text) # 正则捕获 {} 内内容，加前缀 @`
		- `例："{HH AH L OW1}" → ["@HH", "@AH", "@L", "@OW1"]`
		-  模型 embedding 表里同时有英文字母和 ARPAbet 两套符号，加 @ 避免id冲突，以及表示“这一帧来自词典”的标志位，方便前后端对齐与符号隔离
	- （4）映射到 id
		- `sequence = _symbols_to_sequence(sequence)`
		- 查表 symbols → id, Glow-TTS 共 148 种符号，其中 84 为音素
	- （5）返回 int 列表，直接喂入模型 embedding 层
2. Glow-TTS 核心卖点
	- 问题背景
		- FastSpeech/ParaNet 需外部自回归教师做对齐器，训练两阶段，流程笨重
		- 目标：完全摆脱教师，实现「并行 + 自对齐」TTS
	- 解决思路
		- 用**可逆生成流 (Glow)** 将 mel-spectrogram 映射到球形高斯 latent；对齐搜索在 latent 空间完成
		- 结合**动态规划 (Monotonic Alignment Search, MAS)**，一次性找出最可能单调对齐 → 无需 attention、无需教师
	- 结果
		- 对齐完全可微，端到端训练；推理并行，速度比 Tacotron 2 快**一个数量级**
		- 硬单调约束保证长句鲁棒，几乎无跳词/重复
		- 流模型天生可逆，支持语音转换、长度控制、多样性采样等扩展
	- 扩展性
		- 仅需在全局条件向量里加入 speaker embedding，即可零改动推广到**多说话人**场景
3. Glow-TTS 引言要点
	- 自回归痛点
		- 推理时长随输出长度线性增长，长句延迟大
		- attention 易出错，重复/跳词现象显著
	- 并行方案现状
		- FastSpeech 等虽快且鲁棒，但**依赖外部教师对齐器**
		- 教师质量直接决定学生上限，训练两阶段繁琐
	- Glow-TTS 突破
		- 首次把「可逆流 + 动态规划」引入 TTS
			- 流模型将 mel↔latent 双向映射，概率似然可 tractable
			- “可 tractable” = 能精确、高效、可微地算出对数似然，用来端到端训练
			- Monotonic Alignment Search (MAS) 在 latent 空间一次性搜最优硬对齐，无需任何外部模型
		- 结果：15.7× 快于 Tacotron 2，长句鲁棒性↑；利用 latent 插值即可控制语调、音高
4. 相关研究定位
	- 对齐估计
		- HMM / CTC：用前向-后向 DP 找文本-音频对应，但 CTC 判别式、HMM 需条件独立假设
		- Glow-TTS：生成式流模型 + DP，无观测独立性假设， latent 空间并行搜最优单调对齐
	- 并行 TTS
		- FastSpeech / ParaNet：仍靠外部自回归教师提取 attention 地图
		- Glow-TTS：首个**无需任何外部对齐器**的并行声学模型，端到端训练与推理
	- Flow-based 语音生成
		- WaveGlow / FloWaveNet：用流加速 vocoder，**条件仍是 mel**
		- Glow-TTS：把流用在**mel 声学模型**本身，实现文本→mel 并行且可逆，支持多样采样与 latent 控制
	- 同期工作
		- AlignTTS：并行但非流；Flowtron / Flow-TTS：亦流亦并行， yet 仍需外部对齐或专注风格迁移
		- Glow-TTS 率先将「流 + DP 自对齐」结合，完成 standalone fast acoustic model
5. 训练 & 对齐机制
	- 可逆流框架
		- 基于变量替换公式：$$
   \log p_\theta(x|c) = \log p(z|c) + \log\left|\det\frac{\partial z}{\partial x}\right|
   $$
		- 其中 $z = f_{\text{dec}}(x; c, A)$，$f_{\text{dec}}$ 为可逆流解码器，$A$ 为单调对齐函数
	- 对齐函数 $A$
		- 单调且满射：$A(i)$ 单调不减，保证无跳词/重复
		- 先验 $p(z|c)$ 为各向同性高斯，其均值 $\mu_{A(j)}$ 与方差 $\sigma_{A(j)}$ 由文本编码器给出
	- Viterbi 式迭代训练
		- (i) **固定参数 $\theta$**，用 MAS 搜最可能对齐 $A^*$：$$
   A^* = \arg\max_A \sum_{j=1}^{T_{\text{mel}}} \log\mathcal{N}\!\bigl(z_j;\mu_{A(j)},\sigma_{A(j)}\bigr)
   $$
		- (ii) **固定 $A^*$**，梯度上升最大化 $\log p_\theta(x|c, A^*)$
		- 每步交替，保证下界单调提升
	- Duration Predictor（推理用）
		- 架构同 FastSpeech： 2×(1D-CNN+LN+Dropout)+Linear
		- 目标：$\log d_i$ 与 $A^*$ 计算出的时长做 MSE；输入加 stop-gradient，避免干扰流似然
		- 推理：直接用预测时长 $\hat{d}_i$ 构造对齐，无需再跑 MAS，实现完全并行合成
6. Glow-TTS 对齐搜索 A 的本质
	- 符号对照
		- $x_j$：第 j 帧 mel-spectrogram
		- $z_j = f_{\text{dec}}(x_j)$：可逆流解码后对应的 latent 变量（球形高斯）
		- $(\mu_i, \sigma_i)$：文本编码器给音素 i 输出的高斯参数
	- 对齐函数 A
		- $A(j) = i$ 表示“第 j 帧 mel 分配给音素 i” 
		- 约束：单调且满射 → 相当于把 m 帧切成 n 段，每段对应一个音素时长
	- 为什么能搜 A
		- 假设：若分段正确，则 $z_j \sim \mathcal{N}(\mu_{A(j)}, \sigma_{A(j)})$  
		- 对任意合法单调分段，可算总似然 $\sum_j \log \mathcal{N}(z_j; \mu_{A(j)}, \sigma_{A(j)})$  
		- Monotonic Alignment Search (MAS) 用动态规划在所有切法里找**似然最大**的那一条 → 即最优 $A^*$  
	- 结果：无需外部教师，内部即可自监督得到音素时长，用于后续时长预测器训练
7. MAS 动态规划细节
	- 目标
		- 给定 latent 序列 $z_{1:T_\text{mel}}$ 与音素统计 $(\mu_{1:T_\text{text}}, \sigma_{1:T_\text{text}})$，搜最可能单调对齐 $A^*$，使总似然最大
	- 递推定义
		- $Q_{i,j}$ = 前 $i$ 个音素 ↔ 前 $j$ 帧 latent 的最大对数似然
		- 递推式（保证单调满射）：$$
Q_{i,j} = \max\bigl(Q_{i-1,j-1},\; Q_{i,j-1}\bigr) + \log\mathcal{N}(z_j;\mu_i,\sigma_i)
$$
	- 算法步骤
		- 初始化 $Q_{1,j} = \sum_{k=1}^j \log\mathcal{N}(z_k;\mu_1,\sigma_1)$
		- 按行/列填表，复杂度 $O(T_\text{text} \times T_\text{mel})$
		- 回溯：从 $A^*(T_\text{mel}) = T_\text{text}$ 开始，选使 $Q$ 更大的前驱路径，得整条对齐
	- 效率
		- CPU 单条 < 20 ms，占训练总时间 < 2 %
		- 推理无需 MAS，直接用 Duration Predictor 输出对齐，实现完全并行合成
8. Glow-TTS 模型结构速览
	- Decoder（可逆流栈）
		- 单元块：ActNorm → Invertible 1×1 Conv → Affine Coupling（去掉了 WaveGlow 的局部条件）  
		- 并行正反变换：训练时 mel→latent，推理时 latent→mel，均一次前向完成
		- 工程加速：
			- 80-ch mel 帧沿时间轴劈半 → 拼成 160-ch 特征图，减少卷积次数
			- 1×1 Conv 按 40 group 分组计算，Jacobian 行列式简化，显存↓30 %
	- Encoder（文本侧）
		- Transformer TTS 骨架，两处微调：
			- 去掉绝对位置编码，改用相对位置表示（RoPE 风格），更好处理长句
			- pre-net 加残差连接，梯度更顺畅
		- 末端线性层输出每音素的高斯统计 $(\mu_i, \sigma_i)$，供 MAS 与流解码器使用
	- Duration Predictor
		- 同 FastSpeech： 2×(1D-CNN + ReLU + LN + Dropout) + Linear
		- 目标：$\log d_i$（来自 MAS），推理时直接用预测时长构造对齐，实现并行合成
9. Affine Coupling Layer（仿射耦合层）
	- 目的
		- 在 Normalizing Flow 中提供**可逆、并行、tractable Jacobian** 的非线性变换，用于 mel↔latent 双向映射
	- 操作步骤（正向）
		- 输入 x 沿通道切半：$x = [x_A, x_B]$
		- 仅用 $x_A$ 过一个小网络（2×1D-CNN + ReLU）输出：
			- s = scale 向量（同通道数）
			- t = shift 向量（同通道数）
		- 对 $x_B$ 做仿射变换：$y_B = s ⊙ x_B + t$；$y_A = x_A$
		- 输出 $y = [y_A, y_B]$
	- 反向（无需额外参数）
		- $x_B = (y_B − t) ⊙ exp(−s)$；$x_A = y_A$
	- 特性
		- 正反向均一次前向，可并行
		- Jacobian 行列式 $det = ∏ exp(s)$，取对数即 $∑ s$，易算易微分
		- 多层堆叠可形成复杂非线性，同时保持可逆与 exact likelihood
	- 在 Glow-TTS 中的角色
		- 解码器由N个\[ActNorm → Invertible 1×1 Conv → Affine Coupling\]块组成
		- 训练：mel → latent，最大化 exact log-likelihood
		- 推理：从标准高斯采样 latent → 逆变换并行生成 mel
10. Glow-TTS 多说话人扩展要点
	- 单说话人版本
		- 文本编码器仅输出 (μ, σ) 用于高斯先验
		- 训练目标：最大化 mel 似然，无 speaker 信息
	- 拓展到多说话人（仅需以下改动）
		- 增加 speaker embedding
			- 训练集：`LibriTTS` 247 人 → 每人一个可学习向量 `e_spk`，维度 256
		- 注入方式
			- 在**全部 Affine Coupling 层**的 scale/shift 网络里，把 `e_spk` 拼接到输入特征 → 全局条件
			- 编码器仍**不接触 speaker id**，保证文本与身份解耦
		- 时长预测器
			- 同样接收 `e_spk`，使不同 speaker 可有不同语速分布
		- 训练策略
			- 与单说话人相同迭代数，仅 hidden dim 略增；无额外正则，端到端完成
	- 效果
		- MOS 3.45（T=0.667），与 Tacotron 2 持平
		- 同文本仅换 `e_spk` → 说话人时长、基频，实现 zero-shot voice conversion
		- 代码改动 < 50 行，验证“流模型 + 全局条件”即可 scale 到多说话人
