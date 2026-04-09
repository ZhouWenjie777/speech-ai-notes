---
data:
tags:
---
1. CFG（Classifier-Free Guidance）
	- 来源：`Ho & Salimans 2022（Diffusion Models 社区）`
	- 核心思想：训练时随机“扔掉”条件，让模型同时学“有条件/无条件”两种分布；推理时对两项输出做加权差值，提高保真度与鲁棒性
	- 关键词：dropout-condition、guidance-scale β、unconditional path
