---
data:
tags:
---
1. OT-CFM（Optimal-Transport Conditional Flow Matching）
	- 来源：`Lipman et al. 2023（NeurIPS）`
	- 核心思想：把“扩散”里的噪声→数据路径换成“最优传输”直线路径，训练目标从“去噪”变成“预测向量场”，梯度更简单、采样步数更少
	- 关键词：straight-flow、vector-field regression、α-scheduler
