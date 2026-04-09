---
data: 2025-12-15T16:01:00
tags:
---
1. char类型
	- 本质：8位整数，存字符编码（ASCII/EBCDIC等），可当作比short更小的整型做加减比较。
	- 常量：单引号'A'、转义序列'\n'、'\t'、'\xF2'，八进制'\015'，十六进制'\xF2'
	- 输入输出：cin>>c读入字符，cout<<c输出字符；cout.put(ch)成员函数同样输出字符
2. bool类型
	- 值：true、false，机内用1字节表示，true为1，false为0。
	- 运算：可直接参与算术，true提升为1，false为0；cout输出显示1/0而非true/false