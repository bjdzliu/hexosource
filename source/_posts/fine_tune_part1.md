---
title: 总结Fine-Tune ChatGLM3的过程part-1
date: 2024-02-20 14:11:00
tags: LLM
categories:
- Technical Notes
---

了解一些训练过程中必备的知识点  
买一个公有云GPU服务器，了解训练过程


### 回顾基础知识

模型量化、计算机计算精度：  
FP32 使用32位存储一个数字  
FP16 16bit存一个数字。其中1位为符号，5位为指数，10位为尾数。  
INT8是 FP32的量化版本，其中浮点数用8位整数近似。（还是有计算过程的，不是随便舍弃，INT8可以反向推导出FP32）  


n-gram:  
统计语言模型，用于预测下一个词的概率。  
利用机率大小來判断句子合不合理  

n是切词的个数。比如  
n=2,  叫 bigram。"I love learning about deep learning."。这句话的bigrams将包括："I love"，"love learning"  
n=3，叫trigram  

SLOT和BLEU的指标:   
槽填充（Slot Filling）: 我从上海飞往北京。上海和北京是slot value。slot是离开城市和抵达城市  
BLEU：精确匹配n-grams来测量句子相似度。机器翻译的结果与一个或多个参考翻译。  

Transform模型中：  
Token Embedding+Position Embedding  
每次将 input 序列都将经过这两层Embedding  
Token Embedding：每一行都是一个word embedding，a list of number  
Position Embedding： 当前位置和其他位置 ，进行sin or cos 的结果，整个序列形成一个矩阵。  

Pre-training：通用数据训练出来的模型  

Instruction tuning:  明确的问题及明确的答案。特指利用 pairs of input-output instructions 调优模型。目的：遵循自然语言指令，显著提高其在一系列任务中的有用性和适用性。也是一种Fine-Tune。  

PEFT：仅微调少量 (注入额外的) 模型参数，同时冻结预训练 LLM 的大部分参数。目的：减少全参优化造成的资源消耗。本次实操过程属于PEFT。简要说一下在哪里注入参数： 在各个神经网络层，GPT3.5有96层，那么就在每一层，添加参数。  

PEFT：
* prompt-tuning  用新初始化的embedding，加入到数据集中，修改输入序列。在输入层修改。
* prefix-tuning  在神经单元网络中，每个隐层添加参数。
* LoRA 训练低秩矩阵(AXB),添加到W(Q)、 W(V)上。  
* QLoRA 用模型量化，减少了参数的精度。  

GPT-3 模型由两个组件组成： The Model Body and the Head.  
LLM Head ：可能是一个线性层，用 softmax 函数激活，为词汇表中的每个单词生成概率。  


### 云GPU服务器资源  

NVIDIA 显卡上的几种类型的核心：  
* CUDA 核心：最通用的核心，适用于各种计算任务。
* 张量核心：针对某些机器学习计算进行了优化
* 光线追踪 (RT) 核心：对于游戏来说比大多数机器学习更重要，这些核心专门用于模拟光的行为。  

VRAM:是显卡上的内存量  

125 teraflops  = 125万亿次浮点运算    
万亿次浮点运算是指处理器每秒计算一兆个浮点运算的能力。例如，说某个东西有“6 TFLOPS”，意味着它的处理器设置平均每秒可以处理6万亿个浮点计算。


#### 显卡介绍：
NVIDIA Tesla T4 是一款中端数据中心 GPU。它于2019年发布。
* CUDA 核心：2560
* 张量核心：320
* 显存：16GiB
规格页面；https://www.nvidia.com/en-us/data-center/tesla-t4/


A10 是比 T4 更大、更强大的 GPU，于 2021 年发布，采用 NVIDIA 的 Ampere 架构。
* CUDA 核心：9216
* 张量核心：288
* 显存：24 GiB

#### 腾讯云
选择实例： PNV4.7XLARGE116

- 自动安装或者手动安装驱动
- 手动安装 Tesla Driver + CUDA  
下载 NVIDIA Tesla驱动  
https://cloud.tencent.com/document/product/560/8048  


- 安装python3.9  
- 安装pip3.9  
```shell
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3.9 get-pip.py
```
- 安装git-lfs
```shell
apt-get install git-lfs !git lfs install
```

### 下载Model
```shell
git lfs install
GIT_LFS_SKIP_SMUDGE=1 git clone https://www.modelscope.cn/ZhipuAI/chatglm3-6b.git
cd chatglm3-6b
git lfs fetch --include="tokenizer.model"
git lfs checkout tokenizer.model
git lfs fetch --include="*.safetensors"
git lfs checkout *.safetensors
```


### 了解数据集
####  chatglm3-6b的对话格式
-<|system|> 提示器  
-<|user|> 输入  
-<|assistant|>  模型回复  
-<|observation|> 工具调用、代码执行的结果  


多轮对话的json格式表示,按照以下格式准备：训练数据集和验证数据集 train.json 和 dev.json文件  
```json
[
  {
    "tools": [
      "webTool:根据衣服查找哪个网站便宜\nParameters: {\"name":\"衣服名称\",\"price\":\"衣服售价"}\nOutput:A url string that can tell user where is the cloth is on-sale"
    ],
    "conversations": [
      {
        "role": "user",
        "content": "类型#上衣*材质#牛仔布*颜色#白色*风格#简约*图案#刺绣*衣样式#外套*衣款式#破洞"
      },
      {
        "role": "assistant",
        "content": "简约而不简单的牛仔外套，白色的衣身十分百搭。"
      },
      {
        "role": "user",
        "content": "在哪买"
      },
      {
        "role": "tool",
        "name": "search_hotels",
        "parameters": {
          "name": "牛仔外套",
          "price": 300
        },
        "observation": [
          {
            "website": "https://example.com"
          }
        ]
      }
    ]
  }
]


```
ChatGLM3-6b原生支持function call  
tools :表示function和parameters的定义  
conversations： 表示多轮对话  
user 和 assistant ： 分别是用户和AI的问答。  
角色带loss字段。没写默认对 system, user 不计算 loss，其余角色则计算 loss。  

### 下载数据集
这里参照 [github](https://github.com/THUDM/ChatGLM3/blob/main/finetune_demo/lora_finetune.ipynb) 操作即可

###  在单卡上运行
用官方的代码,花了5个小时  
```shell
cd finetune_demo
python finetune_hf.py  data/AdvertiseGen/  THUDM/chatglm3-6b  configs/lora.yaml
```

生成如下内容,使用新模型，加载最新的目录即可。大小在几百兆。我的理解是lora训练的新参数，依赖于原始的Model（12G）文件（别删了）。
```
drwxr-xr-x  3 root root 4096 Mar  1 15:33 test_lora-20240301-153316
drwxr-xr-x  3 root root 4096 Mar  1 15:37 test_lora-20240301-153719
drwxr-xr-x  3 root root 4096 Mar  1 15:49 test_lora-20240301-154939
drwxr-xr-x  3 root root 4096 Mar  1 16:09 test_lora-20240301-160948
drwxr-xr-x 13 root root 4096 Mar  1 21:47 test_lora-20240301-161128

```


