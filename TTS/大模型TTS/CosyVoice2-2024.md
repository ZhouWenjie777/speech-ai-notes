---
data: 2026-01-09T20:49:00
tags:
---
1. 摘要核心
	- 背景
		- CosyVoice 1 已验证：监督离散 token + LLM/Flow Matching → 高自然度、内容一致、说话人相似
		- 多模态 LLM 时代**实时因子 & 首包延迟**成为交互体验瓶颈 → 需要**流式合成** 
	- CosyVoice 2 核心优化
		- 有限标量量化（FSQ）
			- 替换原始 VQ，提高 codebook 利用率，减少码本大小与重建误差
		- 文本→语音 LLM 架构精简
			- 可直接加载**预训练大模型权重**作为 backbone，无需从头训练 LM
		- Chunk-aware Causal Flow Matching
			- 支持**块级因果生成** → 同一模型既能流式（C-by-C）也能整句非流式
		- 大规模多语训练
			- 数据↑ + 上述优化 → 流式模式下**人类级自然度**、**首包延迟≈无损音质** 
	- 结果亮点
		- 流式 MOS 与整句版**无统计差异**（人类 parity）
		- 首包 < 200 ms，RTF < 0.05，真正做到“边生成边播放”
		- 开源 demo：`https://funaudiollm.github.io/cosyvoice2`
2. 引言要点
	- 背景盘点
		- 零样本 TTS 三大路线
			- Codec LM（AR／Masked LM）
			- 特征扩散／Flow Matching（NAR）
			- 混合系统（LM 对齐 + 扩散特征）
		- 共性痛点：**非流式** → 需完整文本 → 首包延迟高 → 语音聊天体验差
	- CosyVoice 2 定位
		- 在 1 基础上，**首次实现“统一流式/非流式”** 零样本TTS，且流式音质**几乎无损**  
	- 四大升级
		- 统一框架
			- 单一模型即可切换 chunk-by-chunk 流式 or 整句非流式
			- 关键：Chunk-aware Causal Flow Matching + 统一文本-语音 LLM
		- LM 简化
			- 去掉独立 Text Encoder & speaker embedding
			- 直接加载预训练文本大模型作为 backbone，上下文理解↑，训练成本↓
		- 量化升级
			- VQ → **Finite Scalar Quantization (FSQ)**
			- 同样码本大小下利用率↑，捕获更多语音信息，重建误差↓
		- 指令 TTS 增强
			- 情感、口音、角色风格、细粒度副语言（单模型融合零-shot + instruct）
	- 结果
		- 流式 MOS **≈ 离线版**，首包延迟 < 200 ms，RTF < 0.05
		- 统一权重部署，一套代码支持“实时对话”与“高保真整句”两种场景
		- Chunk-aware Flow 可迁移到其它TTS → 为“流式扩散/流匹配”提供通用模板
	- 开源
		- 代码与预训练权重已发布：`https://github.com/FunAudioLLM/CosyVoice2`
3. 设计总览（图 1）
	- 核心哲学
		- 延续 CosyVoice 1：语义 ↔ 声学 **解耦** + **渐进语义解码** 
		- 统一框架：一套权重支持 chunk-streaming 与 whole-sentence offline
	- 五大升级模块（后续分节展开）
		- 文本 tokenizer
		- 监督语义语音 tokenizer（FSQ 替换 VQ）
		- 统一文本-语音 LLM（Streaming & Non-streaming 一体）
		- Chunk-aware Causal Flow Matching（流式/整句双模式）
		- 指令式控制增强（情感、口音、角色、副语言）
	- 数据流鸟瞰（图 1）
		- (a) 监督 tokenizer:  Whisper-Encoder + FSQ → 离散语义 token $μ$ 
		- (b) 统一 LLM：文本嵌入 → 自回归生成 μ（含 `<sos>/<eos>/<turn>`）
			- **Causal Transformer Encoder** 保证 chunk 级因果性
			- **Causal Upsampling Transformer** 把 $μ$ 上采样到帧级分辨率
		- (c) Chunk-aware OT-CFM：
			- 条件：{speaker embed $v$, $μ$, masked Mel $X$, $X_t$} → 向量场 → Mel 谱
			- 仅依赖**历史 chunk**，可随时暂停/继续 → 真正“边生成边播放”
		- (d) 后端：现成 HiFi-GAN → 24 kHz 波形
	- 结果承诺
		- 流式 MOS **≈ 离线版**（人类 parity）
		- 首包 < 200 ms，单 V100 RTF < 0.05
		- 一套权重 = **实时语音对话** + **高保真整句合成** + **富文本指令控制** 
4. 文本 Tokenizer
	- 设计目标
		- **直接吃原始文本**，彻底砍掉 G2P 前端，简化流程、端到端学发音
	- BPE 基础
		- 与主流文本 LLM 相同：SentencePiece，词汇量 40 k
	- 中文特殊掩码
		- 若一个 BPE token 编码**多个汉字** → 强制拆成**单字逐一编码** 
		- 原因：
			- 防止“一对多”映射使发音序列过长
			- 减少数据稀疏导致的 corner case（罕见多字 token 发音漂移）
	- 其它语言
		- 英/日/韩**不特殊处理**（字母或音节天然粒度较细）
	- 结果
		- 中文：字级输入，LLM 自己学多音字、变调、连读
		- 跨语：同一 BPE 词汇表，无需额外 phonemizer/forced aligner
		- 为后续“预训练 LLM 直接加载”扫清粒度障碍
5. 监督语义 Speech Tokenizer（FSQ）
	- backbone
		- 仍用 SenseVoice-Large 前 6 层（Encoder 1）+ RoPE
		- 插入 **Finite Scalar Quantization (FSQ)** 模块，取代传统 VQ
	- FSQ 流程（式 1–2）
		- 中间表示 H → 线性降维到 D 维低秩空间
		- 每维值用 **有界 round** 量化到整数区间 \[-K, K]
			- H̃ = ROUND(Proj_down(H))
		- 再线性升维回原始维度
			- H = Proj_up(H̃)
		- 直穿估计（straight-through）回传梯度
	- Token 获取
		- 把每维量化值视为$(2K+1)$进制数字→ 合成整数索引→ 得到speech token $μ_i$
		- 码本**无需显式存储**，维度 D 与边界 K 即决定 $(2K+1)^D$ 个离散状态
	- 关键优势
		- **codebook 利用率 100%**（无 dead code）
		- 同样比特率下重建误差 < VQ → 音质↑
		- 训练与 SenseVoice 其余部分**端到端**微调
	- 速率
		- 25 Hz，每秒25个token→ 与原始CosyVoice保持一致，便于流式 chunk 对齐
	- 结果
		- 相同比特率下，FSQ 版在客观重建误差与主观 MOS 均**优于 VQ 基线**
		- 为后续“大模型直接加载”提供高利用率、低冗余的离散语义表示
6. 统一文本-语音 LLM（Streaming & Non-streaming）
	- 核心思想
		- 同一 `Qwen2.5-0.5B` 权重，**两种序列构造**即可切换流式/非流式
		- 移除 speaker embedding & 独立 Text Encoder
			- 避免 utterance-level 向量泄露语言/副语言信息 → 提升韵律自然度与跨语能力
			- Qwen 自身已足够对齐文本↔token
	- 序列构造对比（图 2）
		- Non-Streaming（离线）
			- \[S, 全部 text, T, 全部 speech, E\]
			- 一次性给完整文本，LLM 自回归生成全部 $μ$
		- Streaming（流式）
			- 混合比例 $N:M = 5:15$
			- 每 5 个 text token 后插 15 个 speech token（已生成或待生成）
			- 若下一段仍是 text→ LLM 输出 **Filling Token**，表示后面还有文本待输入
			- 一旦 text 耗尽 → 插入 T → 剩余 speech 一次性生成直到 E
			- 实际推理：每生成 15 个 $μ$ 就返回一次音频 chunk，实现边输入边播放
	- 应用模式细节（图 3 文字描述）
		- ICL vs SFT
			- | 缩写 | 全称 | 阶段 | 是否更新参数 | 数据量 | 目的 |
			  |---|---|---|---|---|---|
			  | **ICL** | In-Context Learning | 推理 | ❌ | 少量参考prompt（3–10 s） | 零样本模仿新说话人/风格 |
			  | **SFT** | Supervised Fine-Tuning | 训练 | ✅ | 大量标注数据（几十分钟） | 永久专精特定任务或音色 |
			- 在 CosyVoice 2 里
				- **ICL 模式**：给出参考语音，模型即时克隆
				- **SFT 模式**：用目标说话人数据继续训练，模型**永久学会**该音色
				- ICL 现场学，SFT 永久会；一个 prompt 搞定，一个微调定终身
		- ICL Non-Streaming
			- \[S, prompt_text + target_text, T, prompt_speech, target_speech\]
			- prompt_speech 当作“已生成”固定前缀
		- ICL Streaming
			- \[S, (prompt_text+target_text)与 prompt_speech 按 N:M 交错, T, 剩余 speech\]
			- 文本长则补 Filling Token；每 M 个 $μ$ 返回一次
		- SFT Non-Streaming
			- \[S, target_text, T, target_speech\]
			- 无需 prompt，直接为目标说话人生成
		- SFT Streaming（极低延迟）
			- \[S, 首 N 字\] → 生成 M 个 $μ$ → 手动补下 N 字 → 循环直至结束
			- 可用于**语音→语音多模态 LLM**的“即时回声”场景
	- 效果
		- 同一权重，切换序列构造可在 200 ms 内播声，且流式MOS与离线无统计差异
		- 首次在开源实现一个 checkpoint，同时 serving 实时对话 + 高保真整句
7. Chunk-aware Flow Matching（流式核心）
	- 目标
		- 同一 UNet 权重，**既能 offline 整句，也能 chunk-by-chunk 流式**生成 Mel
	- 基础设置
		- Mel 帧率 50 Hz，采样率 24 kHz
		- token → Mel 先 2× 上采样 + lookahead conv（pad=P, kernel=P+1）→ 为因果模块提供“未来 1 帧”预览
	- OT-CFM 回顾（式 3–6）
		- 最优传输直线路径：$\phi_{\text{OT}}(X_0,X_1)=(1-t)X_0+tX_1$
		- UNet 预测向量场 $v_\theta(\phi,t; c)$，条件 $c=(v,\mu,X)$
	- 训练策略
		- **随机掩码 70–100 % 末尾帧** → 让模型学会“补全”
		- **时间步 cosine 调度**：$t:=1-\cos(\frac{\pi}{2}\tau)$ → 初始阶段更多步，降低起步误差  
		- **Classifier-Free Guidance**（式 10）：$\beta=0.7$，NFE=10 → 质量/速度折中  
	- 四重因果掩码（图 3）
		- | 掩码 | 可见范围 | 场景 |
		  |---|---|---|
		  | Non-causal | 全部帧 | offline，质量最佳 |
		  | Full-causal | 仅历史帧 | 极低延迟首包 |
		  | Chunk-M | 历史 + 未来 M 帧 | 首 chunk 低延迟 |
		  | Chunk-2M | 历史 + 未来 2M 帧 | 级联后续 chunk，近似 offline |
	- 训练与自蒸馏
		- mini-batch 内**均匀随机采样**四种掩码 → 同一模型兼容不同延迟需求
		- 大视野掩码充当“教师”，小视野掩码为“学生”→ **隐式自蒸馏**，无需额外损失
	- 结果
		- 流式模式：首包 < 200 ms，NFE=10，RTF < 0.05
		- 掩码切换**零额外参数**，部署仅改一行代码，真正做到“一个ckpt，全场景通用”
8. 流式延迟分析（式 11–12）
	- TTS 首包延迟 L<sub>TTS</sub>
		- 来源：token 生成 + Flow Matching + vocoder
		- 公式：
		  $$
			  L_{\text{TTS}} = M \cdot d_{\text{lm}} + M \cdot d_{\text{fm}} + M \cdot d_{\text{voc}}
			  $$
			- M = 15（首包需生成的 speech token 数）
			- d<sub>lm</sub>/d<sub>fm</sub>/d<sub>voc</sub>：分别对应 LLM、Flow Matching、HiFi-GAN 处理单 token 的 GPU 时间
	- LLM 语音聊天首包 L<sub>Chat</sub>
		- 还要加上**前端大模型**生成文本 token 的时间
		- 上限：
		  $$
			  L_{\text{Chat}} \le N \cdot d_{\text{ulm}} + L_{\text{TTS}}
			  $$
		- N：首句文本 token 数（CosyVoice 2 字级 BPE，N 更小）
		- d<sub>ulm</sub>：上游 LLM 每文本 token 耗时
	- 实测结果（后续表）
		- GPU 单 V100
			- L<sub>TTS</sub> ≈ **120 ms**（M=15, NFE=10）
			- L<sub>Chat</sub> ≈ **200 ms**（含 20 个文本 token）
		- 与人类“对话响应时间”心理阈值（≈200 ms）持平 → **GPT-4o 级交互体验** 
	- 优化手段
		- 减少 M（首包 token 数）、降低 NFE、使用 faster vocoder、并行 pipeline
		- 公式给出**理论下限**，方便后续硬件升级或算法改进时直接代入估算
9. 指令式生成（Instructed Generation）
	- 数据规模
		- 1500 h 指令数据（自然语言 + 细粒度标签）
		- 与基础训练集混合，**一次性联合训练** → 无需额外微调
	- 自然语言指令（表 1 上半）
		- 格式：`<instruction> <|endofprompt|> 正文`
		- 维度：
			- **情感**：高兴/悲伤/惊讶/愤怒/恐惧/厌恶/冷静/严肃
			- **语速**：快速/非常快速/慢速/非常慢速
			- **方言**：粤语、四川话、上海话、郑州话、长沙话、天津话
			- **角色扮演**：神秘、凶猛、好奇、优雅、孤独、机器人、小猪佩奇等
	- 细粒度指令（表 1 下半）
		- **Vocal Bursts**：`[laughter]`、`[breath]` 插入句中任意位置
		- **Vocal Features**
			- `<strong>xxx</strong>` → 重读强调
			- `<laughter>xxx</laughter>` → 边笑边说
	- 训练策略
		- 指令与正文一起进入 LLM 上下文 → 模型端到端学习“指令→声学表现”映射
		- 无额外条件模块，**零外设**实现情感、口音、角色、副语言控制
	- 效果示例（表 1）
		- 自然语言：
			- 你能用高兴的情感说吗？`<|endofprompt|>`今天真是太开心了…
		- 细粒度：
			- `[laughter]`有时候，看着小孩子们的天真行为`[laughter]`...
	- 结果
		- 同一模型，一句提示即可切换情感、方言、角色、重读、笑声
		- 主观评测：指令版 MOS ≈ 无指令版，证明控制不牺牲自然度
		- 为业界首个开源、单模型、中英双语、富文本指令 TTS 提供 baseline
10. 多说话人微调（mSFT）
	- 动机
		- 单说话人 SFT 易灾难性遗忘原有多语/多风格能力
		- 同时微调多人 → 保留泛化性 + 提升每人的音质与相似度
	- mSFT 策略
		- 数据混合
			- 多人语料同时进 batch，每人样本均衡
		- 说话人标签前缀
			- 文本开头加 `SpeakerA<|endofprompt|>` / `SpeakerB<|endofprompt|>`
			- 无标签样本用 `unknown<|endofprompt|>`
		- 超参
			- 学习率固定 $1×10^{-5}$，全程不调度 → 防止“抢说话”导致音色混淆
	- 效果
		- 同一 checkpoint 内含多个高质量说话人
		- 推理时只需切换前缀即可零秒切换音色，无需额外嵌入
		- 客观相似度与主观 MOS 均高于单人逐一轮训 baseline
	- 部署优势
		- **一套权重 = 多人库**，降低线上模型数量与内存占用
		- 为“**千人千声**”产品提供**单文件**解决方案
11. 强化学习微调（RL-for-SFT）
	- 初衷
		- 标准交叉熵只能“复制”训练分布，无法显式优化**说话人相似度**与**发音准确率**
		- 引入 RL → 让模型输出**更贴近人类偏好**（高相似、低 WER）
	- 经典 DPO 路径
		- 奖励：Speaker Similarity (SS) + ASR WER
		- 构造偏好对 $(\alpha^w, \alpha^l)$ → 提取对应 speech token $(\mu^w, \mu^l)$
		- DPO 损失：
		  $$
			  \mathcal{L}_{\text{DPO}}(\pi_\theta; \pi_{\text{ref}}) = -\log \sigma\left(\beta \log\frac{\pi_\theta(\mu^w|y)}{\pi_{\text{ref}}(\mu^w|y)} - \beta \log\frac{\pi_\theta(\mu^l|y)}{\pi_{\text{ref}}(\mu^l|y)}\right)
			  $$
		- 问题：需**反复合成波形**→ 极耗时、GPU 占用×4
	- 简化 ASR-Reward 路径（式 14–16）
		- token → 低秩表示 H（FSQ 逆操作）
		- 冻结 ASR backend → 对 H 重预测文本 → 负对数后验当作奖励
		- Gumbel-Softmax 使 token 采样可微 → 直接优化 LM 的 LASR
	- 结果
		- **无需合成波形**即可获得“ASR 奖励”→ 训练速度↑4×，GPU 节省 75 %
		- 客观指标：相似度 ↑0.8 %，WER ↓3 %（相对）
		- 主观评测：RL 版 MOS **> 标准 SFT 版**，证明“奖励对齐”有效
	- 结论
		- 首次把**ASR 后端当作可微奖励**引入 TTS-RL，避免传统 DPO 的波形循环
		- 为“大模型 + 可微奖励”提供轻量级范式，后续可扩展到情感、风格等其它偏好
12. 结论与局限
	- 核心成果
		- 统一流式/非流式框架 → **单 checkpoint** 即可
		- 流式首包 < 200 ms，RTF < 0.05，MOS 与离线**无统计差异**（人类级）
	- 关键创新
		- FSQ 量化：codebook 利用率 100 %，重建误差↓ 
		- 简化 LLM：直接加载预训练文本大模型，上下文理解↑
		- Chunk-aware CFM：四种掩码，同一UNet兼容 ultra-low-latency ↔ 高保真 
		- 富文本指令：情感、口音、角色、笑声、重读**单模型融合** 
	- 部署优势
		- 一套权重 + 一行序列构造切换 → 同时 serving
			- 实时语音对话（流式）
			- 高保真整句合成（非流式）
			- 富文本角色扮演（指令式）
	- 现存局限
		- **语言覆盖有限**
			- 主要中英；重叠字符集语言（中日韩）易互扰 → 待扩展多语数据
		- **文本指令无法控制音色**
			- 只能通过speaker emb 或参考音频换声，角色扮演仍缺“文本→音色”映射
		- **歌唱场景表现差**
			- 训练集以朗读为主，缺少音符/音高标签 → 未来需引入乐理或歌声数据
	- 展望
		- 扩展 50+ 语言、歌声分支、文本驱动音色控制、实时情绪切换
		- Chunk-aware Flow 设计已证明可迁移到任意 NAR TTS，为“流式扩散/流匹配”提供通用模板
	- 开源
		- 代码 + 预训练权重已发布：`https://github.com/FunAudioLLM/CosyVoice2`
		- 官方 demo 页可在线体验流式、指令、多说话人功能
