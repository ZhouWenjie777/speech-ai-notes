---
data:
tags:
---
1. G2P 用  `phonemizer + espeak` 直接出 IPA，跨语言一套通吃
	- ```python
		  from phonemizer import phonemize
		  txt = "中国央行行长提醒：明天人民币对美元汇率或跌破7。"
		  # 1. 汉字→IPA
		  ipa = phonemize(txt,
                language='cmn',
                backend='espeak',
                strip=True,
                preserve_punctuation=False,
                with_stress=True)      # 带声调数字
        print(ipa)
        # 输出：ʈʂʊŋ³⁵ kuɔ³⁵ ʈʂʰaŋ⁵⁵ jɪn⁵⁵ xaŋ³⁵ xaŋ⁵⁵ ʈʂʰaŋ³⁵ tʰi³⁵ xɤŋ⁵⁵ ...
        # with_stress=False 只剩音标，没有右上角的³⁵调号，方便后续id映射
	  ```
2. 在 G2P 流程里
	- phonemizer = 跨语言“调度壳”
		- 统一接口，自动调用不同后端（espeak、festival、segments 等），把文本变音素，省得自己写脚本
	- espeak = 真正的“音素引擎”
		- 一个开源的轻量级语音合成（TTS）引擎，支持 100+ 种语言，能把任意文本直接转成音素序列或 IPA 符号
	- phonemizer 负责“一句话调用”，espeak 负责“给出符号”；单独 phonemizer 没数据，单独 espeak 接口难用
