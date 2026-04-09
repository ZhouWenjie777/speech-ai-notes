---
data: 2026-01-29T15:56:00
tags:
---
1. 摘要要点
	- 定位：端到端中英双语语音对话机器人，实时可变声（情感、语调、语速、方言）
	- 核心技术
		- 超低码率语音 token 化
			- 在 ASR 编码器里插入向量量化瓶颈 → 单码本 175 bps，帧率 12.5 Hz
		- 模态知识迁移
			- 用文本预训练语料驱动“文本→token”模型合成海量语音-文本交错数据，无需录制语音
		- 三阶段训练
			- 继续预训练：GLM-4-9 B 文本基座 + 无监督语音 + 交错语料 + 有监督语音-文本，累计 1 T token
			- 高质量对话语音微调：提升口语问答与对话自然度
	- 结果：语音语言建模与口语问答 SOTA；对话能力与语音质量均优于现有基线
	- 开源：模型 + 代码已放 GitHub & Hugging Face
2. 引言要点
	- 背景痛点
		- 传统语音对话 = ASR → LLM → TTS，级联延迟高、误差累积、情感表现力弱
		- SpeechLM 数据稀缺：公开语音≈7 M h，vs 文本千亿级 token，预训练不充分→智商受限
		- 对齐式方案（speech encoder + TTS 插件）缺语音预训练，声音仍“机器味”
	- GLM-4-Voice 思路
		- 端到端单码本语音 token
			- 12.5 Hz帧率，175 bps，直接 ASR 编码器+VQ 瓶颈，无需单独声学模型
		- 流匹配语音解码器
			- Flow-matching 将 token → 波形，保证实时与自然度
		- 三阶段数据飞轮
			- 合成交错语料：用“文本→token”模型把 GLM-4 预训练文本自动转成语音 token，生成万亿级 speech-text 混合数据
			- 继续预训练：1 T token（无监督语音 + 合成交错 + 有监督 ASR/TTS）
			- 对话微调：高质量 conversational 数据 + “streaming thoughts”模板（文本 token 与语音 token 交替输出），实现低延迟、高表现力对话
	- 能力覆盖
		- 语音语言建模、口语问答、ASR、TTS、多情感/方言/语速控制，全在一条模型内完成
3. Related Work 要点
	- 语音 token 化
		- 神经声学 codec：目标高保真低码率，RVQ 多层 \[`Zeghidour`’21, `Defossez`’22\] → DAC \[`Kumar`’24\]  refine  
		- 语义 token：自监督提取 \[`Baevski`’21, `Chong`’22\]  
		- 统一方案：SpeechTokenizer、Mini 用不同 RVQ 层承载语义/声学，但同位多 token 需并行预测或退化为纯语义  
		- 监督语义 token: CosyVoice \[`Du`’24\] 从 ASR 提取，已用于 TTS，但未用于 SpeechLM 
	- 语音语言建模
		- 纯语义：GSLM \[`Lakhotia`’24\] 在离散语义 token 上做 AR  
		- 混合：AudioLM 语义+声学双层；TWIST \[`Borsos`’22\] 用 OPT 文本热启；Moshi 把自然语音放大到 7 M h  
		- 文本-语音交错：Spirit-LM \[`Nguyen`’24\] 用平行语料构造，但数据规模受限  
		- 本文：用“文本→token”合成海量交错语料，突破平行数据稀缺
	- 端到端语音聊天机器人
		- 语音翻译 → 对话：SpeechGPT \[`Zhang`’23\] 套 LLM+离散语音 token；Moshi 全双工对话  
		- 对齐式：Qwen-Audio、Llama-Omni、Freeze-Omni 先 Whisper 编码再 LLM+TTS，只能控内容，不控风格  
		- 简易微调：Mini-Omni 直接指令微调同时出文本+语音，无语音预训练 → 质量差（实验将对比）
4. 架构要点
	- 设计目标
		- 端到端听懂语音并语义正确回应
		- 按用户口语指令实时生成带副语言特征（情感、语调、语速、方言）的语音
	- 核心策略
		- 最小改动自回归 Transformer，保留 GLM-4-9 B 文本能力
		- 单码本监督语音 token：175 bps，12.5 Hz，避免多层 RVQ 并行重构麻烦
		- 统一语音表示：输入/输出同一套 token，直接 next-token 预测，支持大规模无监督语音预训练
		- 流匹配语音解码器：支持 chunk 级流式合成，降低对话延迟
		- Streaming-thought 模板：SFT 阶段让模型交替输出文本 token 与语音 token，实现“边想边说”低延迟交互
	- 组件来源
		- Speech tokenizer & decoder 直接沿用 `Zeng et al.` \[45\] 的流匹配方案，无需重新设计声学模型 
5. 语音 token 化细节
	- 背景
		- 声学 token 化：靠重建/对抗目标，高保真但需高采样率或多码本 RVQ，AR 生成负担大  
		- 语义 token 化：自监督提取，低码率却丢声学细节，合成质量差
		- 理想 tokenizer 应同时满足：① 单码本+低帧率 ② 与文本对齐 ③ 高保真重建
	- 目标
		- 单码本 + 低帧率（12.5 Hz）→ 适配 AR 生成
		- 与文本对齐 → 可继承 LLM 知识
		- 支持高保真合成
	- 架构 = 监督 ASR 编码器 + VQ
		- 基座：whisper-large-v3
		- 插入池化层 + 向量量化（EMA 更新，commit-loss=10.0）
		- 码本不足频度时随机重置，防塌陷 \[`Dhariwal`’20\] 
	- 流式因果改造
		- 首层卷积 → 因果卷积
		- Bi-Attention → Block-Causal Attention，实现 chunk 级流式编码
	- 训练数据
		- 有监督：`LibriSpeech、GigaSpeech、MLS-Eng、Wenet、Common Voice、AISHELL-1` + 10 k h 私有中文 ASR
		- 伪标签：700 k h 英/中无监督语音（`whisper/paraformer-large` 标注） 
		- 比例 1:3，batch 4096，lr 1e-5，2 epoch
	- 帧率-码本大小联动
		- 帧率 ↓ → 池化倍数 ↑ → 信息损失 ↑ → 码本大小 ↑（补偿信息损失）
	- 结果（表 1）
		- 12.5 Hz/175 bps 在 LibriSpeech clean/other WER 2.10/4.90，AISHELL-1 CER 3.02 综合最佳 → 被选为 GLM-4-Voice 正式 tokenizer
6. 语音解码器
	- 架构 = CosyVoice 同款三件套
		- ① 语音 token 编码器　② 条件 Flow-Matching　③ HiFi-GAN 声码器
		- 全部沿用 `Zeng et al. [45]`，无需重设计
	- 两阶段训练
		- 预训练：随机说话人+多质量 700 kh 无监督语音 → 学得通用声学分布
		- 精调：单说话人高音质数据 → 提升音色一致性与 MOS
		- 精调时注入“截断音频”：只用前 n·b 秒片段，强制模型凭前缀预测后续，模拟流式场景
	- 流式推理策略
		- 块大小 b = 0.8 s（≈10 个 token）→ 首包延迟 ≤ 0.8 s 
		- 运行时：已合成 (n-1)b 秒作为 prompt，实时预测下一块，实现“边想边说”
	- 重建质量（表 1）
		- 12.5 Hz/175 bps 在 LibriSpeech 
			- WER 8.43（内容保留）
			- MOSNet 3.39（主观听感）
		- 对比 SpeechTokenizer 1.5 kbps 层：比特率↓ 8.6×，质量仍持平 → 被选为正式解码器
7. 推理流程与延迟公式
	- 任务解耦
		- 先 Speech→Text：用户语音 $Q_s$ → 文本回答 $A_t$ 
		- 再 Speech&Text→Speech：$Q_s + A_t$ → 带语调/情感的语音 $A_s$ 
		- 保证语义准确，但需等 $A_t$ 完整生成后才能开始 $A_s$，首包延迟高
	- Streaming-Thought 模板
		- 交替输出：13 文本 token : 26 语音 token = 1 : 2 
		- 确保文本始终超前，为语音提供上下文；26 语音 token≈2.1 s 音频（12.5 Hz）
	- 端到端延迟公式
		- 总延迟 = 流式 token 化 + LLM 预填充 + LLM 解码 + 语音解码 $$
			T_{\text{total}} = f_{\text{tokenize}}(t_{\text{block}}) + f_{\text{prefill}}(f_r \cdot T_{\text{user}}) + f_{\text{decode}}(23) + f_{\text{decode}}(10)
			$$
		- 其中 $f_r = 12.5\ \text{Hz}$，首包只需 10 语音 token≈0.8 s 
8. 训练流程
	- 阶段一：联合语音-文本预训练
		- 数据配比（1 T token，seq=8192）
			- 文本-only：10 T token，占比 30 %，保文本能力
			- 交错语音-文本：455 B 语音+279 B 文本，0.90 epoch，跨模态知识迁移
			- 无监督语音：31 B token，2.10 epoch，学真实语音分布
			- 有监督语音-文本（ASR+TTS）：11 B 语音 + 3.5 B 文本，2.07 epoch，强化基础任务
		- 超参
			- 初始：GLM-4-9 B-Base，词表扩展新增语音 token 
			- 优化器：AdamW β=(0.9,0.95)，lr 线性衰减 6e-5 → 6e-6 
	- 阶段二：监督微调（SFT）
		- 数据构建
			- 多轮口语对话：文本对话过滤→删代码/数学→缩短长句→合成语音；额外录制多样口音 Qs
			- 风格可控对话：速度、情感、方言等标签，人工录制高音质多轮音频
		- 训练细节
			- 任务分离+Streaming-Thought 模板
				- 样本拆两条：
					- 只算文本输出损失（Qs→At），加速语义收敛
					- 只算语音输出损失（Qs+At→As），保副语言质量
			- 轮次差异
				- 语音输出：20 epoch；文本输出：4 epoch
			- 正则
				- lr 1e-5 → 1e-6，weight decay 0.1，dropout 0.5，grad clip 1.0
9. Chat 模型评测
	- 指标
		- General QA: AlpacaEval（排除数学）+ GPT-4o 1-10 打分 
		- Knowledge: `WebQuestions/Llama-Q/TriviaQA`100 题，GPT-4o 判对错0-10
		- 语音自然度：UTMOS 预测 MOS 
		- 音字一致性：Whisper 转写生成语音 → 与文本回答算 WER（ASR-WER）
	- 结果（表 6，英文输出限定）
		- | 模型 | General QA↑ | Knowledge↑ | UTMOS↑ | ASR-WER↓ |
		  |---|---|---|---|---|
		  | SpeechGPT | 1.40 | 2.20 | 3.86 | 66.57 |
		  | Mini-Omni | 2.44 | 1.10 | 3.17 | 25.28 |
		  | Llama-Omni | 3.50 | 3.90 | 3.92 | 9.18 |
		  | Moshi | 2.42 | 3.60 | 3.90 | 7.95 |
		  | GLM-4-Voice | 5.40 | 5.20 | 4.45 | 5.74 |
		- 所有指标均最佳，且双语模式下英文输出仍保持低 WER
	- 结论
		- 12.5 Hz 监督 token + Flow-Matching 解码 + 1 T token 预训练，实现文本-语音无缝统一
		- 在语音语言建模、ASR、TTS、口语问答、对话质量上均达 SOTA
		- 流式低延迟、可控制副语言特征
		- 模型与代码已开源，推动实用化语音 AI 研究
