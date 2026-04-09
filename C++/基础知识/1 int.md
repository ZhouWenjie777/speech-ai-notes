---
data: 2025-12-15T15:29:00
tags:
---
1. 整型
	- 分类：short ≤ int ≤ long ≤ long long，各分signed/unsigned；C++11起long long至少64位。
	- 空间探测：sizeof(类型)返回字节数；climits提供CHAR_MAX、INT_MAX等符号常量。
	- 声明与初始化：short score=0; int x=y; 支持定义时直接赋常量或变量初值。
	- 字面量后缀：123U无符号，123L长整型，123LL long long，可组合123ULL。
	- 进制表示：0开头八进制042，0x开头十六进制0x2A；cout用dec/hex/oct控制输出基数
