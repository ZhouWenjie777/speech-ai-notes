---
data: 2026-02-10T15:23:00
tags:
---
1. 时间线
	- | 版本 | 时间 | 核心变化 |
	  |------|------|----------|
	  | GPT-1 | 2018.06 | 奠基：Transformer Decoder + 无监督预训练 |
	  | GPT-2 | 2019.02 | 规模扩大 10 倍，Zero-Shot 涌现 |
	  | GPT-3 | 2020.05 | 175B 参数，In-Context Learning |
	  | GPT-3.5 | 2022.03 | 指令微调 + RLHF，ChatGPT 前身 |
	  | GPT-4 | 2023.03 | **多模态**，MoE 架构，SwiGLU（推测）|
	  | GPT-4o | 2024.05 | **原生端到端多模态** |
	  | GPT-4.5 | 2025.02 | 性能优化版 |
	  | **GPT-5** | **2025.08** | **统一模型**，内置推理，多模态 |
	  | GPT-5.1 | 2026.01 | 自适应思考时间 |
	  | GPT-5.2 | 2025.12 | Codex 编程版 |
	  | **GPT-5.3** | **2026.02** | **最强编程智能体**，速度+25% |

2. 核心组件（始终不变）
	- | 组件 | GPT-1/2/3 | GPT-4+ |
	  |------|-----------|--------|
	  | 归一化 | LayerNorm | **RMSNorm**（推测）|
	  | 激活函数 | **GeLU** | **SwiGLU**（门控）|
	  | 位置编码 | 可学习位置嵌入 | **RoPE**（推测）|
	  | 注意力 | 标准 MHA | **GQA/MQA**（推测）|
	  | 架构 | Dense | **MoE**（稀疏）|
	 
	- 关键公式速查：
		- **LayerNorm（早期）**：$$\bar{x} = \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} \odot \gamma + \beta$$
		- **RMSNorm（后期，推测）**：$$\bar{x} = \frac{x}{\sqrt{\frac{1}{d}\sum x_i^2 + \epsilon}} \odot \gamma$$
		- **GeLU（GPT-1/2/3）**：$$\text{GeLU}(x) = x \cdot \Phi(x) = x \cdot \frac{1}{2}\left[1 + \text{erf}\left(\frac{x}{\sqrt{2}}\right)\right]$$$$\text{FFN}(x) = \text{GeLU}(xW_1 + b_1)W_2 + b_2$$
		- **SwiGLU（GPT-4+，推测）**：$$\text{SwiGLU}(x) = (\text{Swish}(xW_1) \odot xW_2)W_3$$
		- **自注意力**：$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V$$
		- 其中M为 **因果掩码**：$$M_{ij} = \begin{cases} 0 & i \geq j \\ -\infty & i < j \end{cases}$$
			-  0：允许看（注意力正常计算）
			- -∞：禁止看（softmax后变为0，注意力权重为0）
			- 因果掩码 = 上三角遮罩，确保模型只能看左边（过去），不能看右边（未来），保证自回归生成
3. 关键改进点
	- | 版本 | 解决什么问题 |
	  |------|-------------|
	  | GPT-2 | 验证 Scaling Law |
	  | GPT-3 | In-Context Learning，无需微调 |
	  | GPT-3.5 | **RLHF + 指令微调**，对齐人类偏好 |
	  | **GPT-4** | **MoE 架构**，多模态，SwiGLU 激活 |
	  | GPT-4o | **原生端到端多模态**，降低延迟 |
	  | **GPT-5** | **统一架构**，内置推理，自动选择思考深度 |
	  | **GPT-5.3** | **代理式编程**，长时任务执行，自我开发 |

4. GPT-5 系列特点
	- | 特性 | 说明 |
	  |------|------|
	  | 统一模型 | 合并 GPT/o 系列，自动路由 |
	  | 动态思考 | 根据任务难度自动选择思考时间 |
	  | 多模态 | 文本 + 图像 + 音频 + 视频 |
	  | 持久记忆 | 跨会话记忆 |
	  | 智能体能力 | 长时任务执行，工具使用 |

5. 架构演变总结
	- | 阶段 | 激活函数 | 架构 | 特点 |
	  |------|----------|------|------|
	  | GPT-1/2/3 | **GeLU** | Dense | 标准 Transformer |
	  | GPT-4+ | **SwiGLU** | **MoE** | 门控 + 稀疏激活 |

	- 早期 GeLU + Dense，后期 SwiGLU + MoE，对齐靠 RLHF，推理靠统一架构
