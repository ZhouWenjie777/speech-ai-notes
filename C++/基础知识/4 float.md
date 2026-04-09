---
data: 2025-12-15T16:22:00
tags:
---
1. 浮点数
	- 概念：实数近似表示，由尾数×2^指数组成，32位float24位尾数8位指数
	- 常量：十进制127.22或科学计数法34E-8，默认double，加F/L指定float/long double
	- 类型：float≥32位，double≥48位且≥float，long double≥double；float约7位十进制精度，double约15位
	- 精度误差：1/3无法精确表示，放大后误差累积，输出333333.250000/333332.000000演示精度损失