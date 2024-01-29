---
title: 自注意力机制做了什么？
date: 2023-07-01 16:22:00
tags: LLM
categories:
- Technical Notes

---
在听了很多关于 自注意力机制 的描述后，不自觉的会想到它用什么数学方式，模仿人类思考、关注重点词汇、短语呢？  
这里我打算记录一下个人理解。

在 Transformer 架构中，Encoder 和 Decoder 中的每一个小模块，都代表着一堆数学公式，Multi-Head Attention 也不例外
![Transformer ](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/ailab/attention.png)

既然是数学公式，就有输入，输出，结合注意力机制的描述，我们想测算“输入” 的内容是否重要，是否值得关注，那么就用 “输出” 进行打分，分数高的，权重就大。  

在注意力机制中，通常存在三个关键元素：查询（query）、键（key）、和值（value）。它们各自的角色和作用如下：  
查询（query）：输入的内容，比如你在google中输入的查询问题  
键（key）：输入序列中词语信息的位置向量，不是词向量。 理解成google搜索结果中的标题。  
值（value）：  输入序列中词语的内容。理解成google搜索结果中的content


两个向量的相似度可以用点积计算
![Func ](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/ailab/attention_simi.jpg)
完全不相关=垂直=0


计算公式：
![Func ](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/ailab/func_attention.jpg)

根据输入数据的不同，查询（queries）、键（keys）和值（values）可以以各种方式表示，例如向量、矩阵或张量。在自然语言处理（NLP）的背景下，query 和 key 通常被表示为词嵌入（word embeddings），而 value 则表示为上下文嵌入（contextualized embeddings）。

计算过程基本如下：  
- 三个权重矩阵（W(Q)、W(K)、W(V)），初始化是随机的
- 第一步是将每个编码器输入向量与我们在训练过程中训练的三个权重矩阵（W(Q)、W(K)、W(V)）相乘。该矩阵乘法将为每个输入向量提供三个向量：键向量、查询向量和值向量。
- K向量和Q向量点积
- 在第三步中，我们将得分除以关键向量 (d k )维度的平方根。论文中关键向量的维数是64，所以是8。背后的原因是如果点积变大，这会使得 Softmax 的输出接近于 0 或 1，而梯度较小，从而可能导致梯度消失的问题。除以(d k ),缩小点积。
- 在第四步中，我们将对查询词（这里是第一个词）计算的所有自注意力分数应用 softmax 函数。
- 在第五步中，我们将值向量乘以我们在上一步中计算的向量。
- 在最后一步中，我们对上一步中获得的加权值向量求和，这将为我们提供给定单词的自注意力输出。

以上计算，让模型考虑了所有序列的位置。  

在最后一步中，一旦softmax和v相乘，将过滤出高注意力部分，也就是说，得到了类似人脑看到的最重要的图像，文字。
  

参考：  
  [Attention Networks](https://medium.com/@geetkal67/attention-networks-a-simple-way-to-understand-self-attention-f5fb363c736d)  
  [Attention in NLP](https://www.geeksforgeeks.org/self-attention-in-nlp/)


