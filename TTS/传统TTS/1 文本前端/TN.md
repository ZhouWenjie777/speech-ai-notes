---
data:
tags:
---
1. IPA vs ARPAbet vs CMUDict
	- IPA（International Phonetic Alphabet）
		- 国际语音学学会制定的通用音素符号集，语言无关（中文适用）
		- 例：英语 `"ship" → /ʃɪp/`，法语 `"chat" → /ʃa/`，都用同一个 `/ʃ/` 符号
	- ARPAbet
		- 1980 年代美国 `DARPA` 为英语语音识别设计的 ASCII--only 音素集
		- 例：`ship → SH IH P`；所有符号仅用大写字母或数字，方便旧系统键盘输入
	- CMUDict
		- 卡内基梅隆大学发布的免费英语发音词典
		- 条目格式：`word  ARPAbet-phone1  ARPAbet-phone2 ...`
		- 不含 IPA；若要用 IPA，需映射表（many-to-many，因两者音素粒度不同）
	- 主要差异
		- 符号范围：IPA 通用多语言，ARPAbet 仅英语
		- 字符集：IPA含Unicode音标（`ʃ, ð, ɜː`），ARPAbet纯ASCII（`SH, DH, ER`
		- 模型使用习惯：
			- Tacotron/FastSpeech/Glow-TTS 训练 CMUDict 分支时常用 ARPAbet
			- VITS 用 `phonemizer+espeak` 直接输出 IPA，省去 ARPAbet↔IPA 转换
	- 对于汉字中文
		- 流程：汉字 → pypinyin → 声母/韵母查表 → symbols 列表 → id 序列
		- symbols 为声母或韵母（共 60~70 个 ASCII 符号），再补 `pad/punct/ letter` 即可，长度通常 < 80
		- `embedding` 层直接用这套 id 训练，和英文走 IPA/ARPAbet 等价