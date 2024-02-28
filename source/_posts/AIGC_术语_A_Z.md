---
title: AGI_Glossary术语
date: 2023-06-30 11:33:00
tags: LLM
categories:
- Technical Notes
---
想快速学习 Generative AI , 先收集一下这些术语

### A
### auto-regressive model
自回归模型
一种根据自己先前的预测推断出预测的模型。例如，自回归语言模型基于先前预测的令牌来预测下一个令牌。所有基于 Transformer 的大型语言模型都是自回归的。

相比之下，基于 GAN 的图像模型通常不会自动回归，因为它们在一个单一的前向通道中生成图像，而不是在步骤中迭代。但是，某些图像生成模型是自动回归的，因为它们按步骤生成图像。


### C
### chain-of-thought prompting
思维链对话，简写COT
一种提示工程技术，鼓励大型语言模型(LLM)逐步解释其推理过程。例如，考虑下面的提示，特别注意第二句话:  

``` 
如果一辆车在7秒内从0加速到60英里每小时，司机会受到多少重力的作用？在答案中，显示所有相关的计算结果。
```

LLM 的反应可能是: 
- 显示一系列物理公式，在适当的位置插入值0、60和7。
- 解释为什么它选择这些公式和各种变量的含义。



### D
### direct prompting 
同 zero-shot prompting.

### distillation
中文翻译 蒸馏
将一个模型(称为“teacher”)缩小为一个较小的模型(称为“student”)的过程，该模型尽可能忠实地模拟原始模型的预测。
distillation 是有用的，因为较小的模型比较大的模型(老师)有两个关键的好处:
- 更快的推理时间
- 减少能耗

### F
### few-shot prompting
Prompt Engineering （提示词工程）的一种方式。  
包含若干个提示词。看例子
| Parts of one prompt   | Notes |
|--------|-----|
| What is the official currency of the specified country?  | The question you want the LLM to answer.| 
| BobFrance: EUR	    | One example.| 
| United Kingdom: GBP  | Another example.  | 
| India:  | 查询语句  | 

Few-shot prompting比 zero-shot prompting 和 one-shot prompting的结果理想些，但是提示词需要的更长。
