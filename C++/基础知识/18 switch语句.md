---
data: 2025-12-16T15:28:00
tags:
---
1. switch语句
	- 格式：
		- switch(整型表达式) { case 常量1: 语句1; break; ... default: 语句n+1; } 表达式与常量匹配后顺序执行到break或末尾。
	- break：
		- 跳出当前switch，若无break继续向下贯穿后续case，实现“多分支共享代码”或故意落空。
	- continue：
		- 仅用于循环，跳过本次剩余语句进入下一轮迭代；在switch中无意义。
	- 对比if：switch条件离散且为整型/枚举时更清晰；嵌套if适合区间判断或复杂逻辑