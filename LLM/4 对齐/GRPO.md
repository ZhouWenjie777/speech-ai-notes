---
data: 2026-02-09T12:06:00
tags:
---
1. 群体相对策略优化（Group Relative Policy Optimization, GRPO）
	- [（1）通俗讲解](https://zhuanlan.zhihu.com/p/1940894587217514873) 
2. GRPO 算法流程（RL，组相对策略优化）
	- （1）输入：SFT 模型 $\pi_{ref}$（冻结），策略模型 $\pi_\theta$，问题数据集 $\mathcal{D}$，KL 超参 $\beta$，组大小 $G$，裁剪超参 $\epsilon$ 
	- （2）输出：与人类偏好对齐的模型 $\pi_\theta$ 
	- （3）算法步骤
		- 初始化：$\pi_\theta \leftarrow \pi_{ref}$（可复制参数或热启动）
		- 循环直至收敛
			- a. 从 $\mathcal{D}$ 采样一个问题 $q$ 
			- b. **组内生成**（G 个输出）
				- 用当前策略 $\pi_\theta$ 对同一问题 $q$ 采样生成 $G$ 个不同答案：$o_1, o_2, ..., o_G$ 
				- 每个输出独立采样，允许多样性
			- c. **奖励计算**（可并行）
				- 用 Reward 模型对每个输出打分：$r_1, r_2, ..., r_G$ 
				- 同时计算每个输出与参考模型的 KL 散度惩罚（可选）
			- d. **组内优势估计** 
				- 计算组内奖励均值：$\text{mean}(r) = \frac{1}{G}\sum_{i=1}^{G} r_i$ 
				- 计算组内奖励标准差：$\text{std}(r) = \sqrt{\frac{1}{G}\sum_{i=1}^{G}(r_i - \text{mean}(r))^2}$ 
				- 计算每个输出相对优势：$A_i = \frac{r_i - \text{mean}(r)}{\text{std}(r) + \epsilon}$（$\epsilon$ 为数值稳定性小常数）
				- 高于均值得正优势（奖励），低于均值得负优势（惩罚）
			- e. **策略优化（最大化目标）** 
				- 计算重要性采样比率：$\rho_i = \frac{\pi_\theta(o_i|q)}{\pi_{\theta_{old}}(o_i|q)}$
		        - 计算裁剪目标：$\rho_i^{clip} = \text{clip}(\rho_i, 1-\epsilon, 1+\epsilon)$ 
		        - **目标函数（最大化）**：
			        - $\mathcal{J}_{GRPO}(\theta) = \frac{1}{G}\sum_{i=1}^{G} \left[ \min(\rho_i A_i, \rho_i^{clip} A_i) - \beta \mathbb{D}_{KL}(\pi_\theta \| \pi_{ref}) \right]$ 
			        - 其中 KL 散度：$\mathbb{D}_{KL}(\pi_\theta \| \pi_{ref}) = \frac{\pi_{ref}(o_i|q)}{\pi_\theta(o_i|q)} - \log\frac{\pi_{ref}(o_i|q)}{\pi_\theta(o_i|q)} - 1$
			        - **或损失函数（最小化）**：$\mathcal{L}_{GRPO}(\theta) = -\mathcal{J}_{GRPO}(\theta)$ 
			- f. 反向传播更新 $\pi_\theta$（梯度上升最大化目标，或梯度下降最小化损失）
			- g. 定期更新 $\pi_{\theta_{old}} \leftarrow \pi_\theta$ 
		- 返回 $\pi_\theta$ 
	- （4）关键细节
		- GRPO 用**组内均值**替代 PPO 中昂贵的 Critic 模型作为基线（Baseline），显著减少显存占用
		- 目标函数是**最大化**形式（与 PPO 一致），若写损失函数需加负号
		- KL 惩罚项采用 Schulman 近似形式，而非标准 KL 散度
		- 组大小 $G$ 通常设为 4~16，平衡估计方差与计算开销
		- 优势归一化（除以 std）对训练稳定性至关重要
		- 保留了 RL 的探索能力（采样生成），同时摆脱了 Critic 模型的负担
3. GRPO 目标函数解析：为什么"减平均"能实现"好的更好、坏的更坏"
	- 核心公式：
		- $$\hat{A}_i = \frac{r_i - \bar{r}}{\sigma}, \quad \bar{r} = \frac{1}{G}\sum_{i=1}^{G} r_i$$
	- 关键洞察：优势符号编码相对好坏
		- | 情况 | 条件 | $\hat{A}_i$ 符号 | 含义 |
		  |:---|:---|:---|:---|
		  | 高于平均 | $r_i > \bar{r}$ | **正** | 好响应，值得鼓励 |
		  | 低于平均 | $r_i < \bar{r}$ | **负** | 坏响应，需要抑制 |
	- 最大化如何实现"提高好的、降低坏的"
		- 目标函数核心项：$\min(\rho_i \hat{A}_i, \rho_i^{clip} \hat{A}_i)$，其中 $\rho_i = \frac{\pi_\theta(o_i|q)}{\pi_{\theta_{old}}(o_i|q)}$
		- **Case 1：好响应（$\hat{A}_i > 0$）**
			- 目标 $\propto +\rho_i$，最大化 → **增大 $\rho_i$** → $\pi_\theta(o_i|q)$ 上升
			- 效果：提高高于平均分的响应生成概率
		- **Case 2：坏响应（$\hat{A}_i < 0$）**
			- 目标 $\propto -\rho_i$，最大化 → **减小 $\rho_i$** → $\pi_\theta(o_i|q)$ 下降
			- 效果：降低低于平均分的响应生成概率
		- **减平均获得相对信号，正负号自动决定梯度方向**——正优势拉概率上升，负优势拉概率下降，最大化目标自然实现"好的更好、坏的更坏"
	- 对比：PPO vs GRPO 基线
		- | 特性 | PPO | GRPO |
		  |:---|:---|:---|
		  | 基线来源 | 独立 Critic 网络 $V(s)$ | 组内平均分 $\bar{r}$ |
		  | 优势计算 | $A_i = r_i + \gamma V(s') - V(s)$ | $\hat{A}_i = \frac{r_i - \bar{r}}{\sigma}$ |
		  | 资源消耗 | 需要额外 Critic 模型 | 零开销，用组内统计即可 |
		  | 核心思想 | 比 Critic 预测好就鼓励 | 比组内平均好就鼓励 |
	- 为什么用组平均有效？
		- **相对评价**：同一问题的不同回答之间具有可比性
		- **自适应**：难问题的平均低，好答案只需相对高；易问题反之
		- **免训练**：无需学习 Critic，避免价值估计误差
