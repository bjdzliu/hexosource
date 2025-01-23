---
title: Agent-Tuning
date: 2024-07-15 20:33:00
tags: LLM
categories:
- Technical Notes
---

### 背景
开源通用的LLMs在执行 agent 任务时，远不如 ChatGPT 和 GPT-4 等商业模型。在很多实际应用场景，无法使用最强的大模型 API ，需要私有化部署小模型，需要控制推理成本。   
提高agent任务的能力，要么是专注对某个LLM进行agent调优，要么是写一个专门的agent框架。  
论文里提出了Agent-Tuning，其目标是提高LLMs的通用代理能力，同时至少保持它们的一般LLM能力  


### 方法
- 构建 AgentInstruct 数据集  
- 微调大模型  

### 训练成本估算
* 训练框架： HuggingFace Trainer + DeepSpeed Zero3  

* 配置说明：max_len：4096 + Flash Attention + bf16 （batchsize=1、AdamW优化器）  

模型可以处理的最大 token : 4096  
通过减少显存访问次数和优化计算流程，显著加速 Transformer 模型中的自注意力机制  
采用 bfloat16 浮点数格式，减少了内存占用和计算开销  
多卡训练可以减少训练时间。  
| **模型** | **训练最低配置** | **训练数据规模（token数）** | **建议训练 epoch 数** | **平均训练时长** | 
| --- | --- | :---: | :---: | --- | 
| 7B （全参数）  | 4卡A100（4 * 80G） | 470M | 5 | 25h * 5 = 125h | 
| 14B （全参数） | 8卡A100（8 * 80G） | 470M | 4 | 24h * 4 = 96h | 
| 72B （全参数） | 32卡A100（32 * 80G） | 470M | 2 | 40h * 2 = 80h | 


### 构建 AgentInstruct
智谱开源  
AgentInstruct 是一个经过挑选的智能体数据集，包含 1866 个高质量交互、6 个多样化的真实场景任务  
数据集开源在 🤗[Huggingface Repo](https://huggingface.co/datasets/THUDM/AgentInstruct)  


#### 构建训练数据
开源 agent tool prompt 思路是一样的，只是prompt的描述，tool和返回action的结构有所不同。  
训练数据应该包含各种不同的prompt模板和response  
这里举一个简单的prompt的例子  
```python
instruction = f"""
You should follow the instructions to answer the question.
1. Each time you make a decision, you use only one tool, and you can use it as many times as you like.
2. Ensure that the command you invoke or the tool you use is included in the following given list of tools.

Answer the following questions as best you can. You have access to the following tools:

{tool_list}

If you find you need to find information from one of these tools, you can use the corresponding action. Tell me what action you want to take, and I will provide you with the information you need.
If you need action, tell me the action name and wait for the result I send you.

Print the content using the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of {tool_list}
Action Input: the input to the action

Begin!
"""
```

LLM 第一次返回

```
Thought: I need to retrieve information about the latest commit in the repository demo2_jenkinsfile in GitHub.

Action: github_api  
Action Input: repository=demo2_jenkinsfile, info=latest_commit  
```
在程序中，获取action，调用工具得到结果后，将结果及以上prompt 一并输入给LLM，最终得到 response  
截止到现在，已经得到了一个训练数据样例：  

| Input ( Agent Prompt ) | Output ( Response ) | 
| --- | --- |
｜You should follow the instructions to answer the question.<br />1. Each time you make a decision, you use only one tool, and you can use it as many times as you like.<br />2. Ensure that the command you invoke or the tool you use is included in the following given list of tools.<br />Answer the following questions as best you can. You have access to the following tools:<br />{tool_list}<br />If you find you need to find information from one of these tools, you can use the corresponding action. Tell me what action you want to take, and I will provide you with the information you need.<br />If you need action, tell me the action name and wait for the result I send you.<br />Print the content using the following format:<br />Question: the input question you must answer<br />Thought: you should always think about what to do<br />Action: the action to take, should be one of {tool_list}<br />Action Input: the input to the action|Thought: To find the latest commit in the repository demo2_jenkinsfile in GitHub, I should use the github_api tool. Let's use it to retrieve the information.<br />Action: github_api<br />Action Input: repository = "demo2_jenkinsfile"<br />Observation: The latest commit in the repository demo2_jenkinsfile is "cde45f67890abcde12345f67890abcde12345f6".<br />Thought: The latest commit in the repository demo2_jenkinsfile is "cde45f67890abcde12345f67890abcde12345f6".

有了训练数据，就可以进行全参微调，如果仅进行 LoRA（Low-Rank Adaptation），效果可能不理想  

#### 效果评估
根据任务需求，设计输入模板或提示词（prompt），指导 GPT-4 生成相关样本。  
对 GPT-4 生成的样本进行人工审核，检查其准确性和合理性。  
将修正后的样本与初始验证集合并，形成最终的 benchmark。  
采用综合评估方法，结合了 精确匹配（Exact Match, EM） 和 模糊匹配（如 ROUGE、BLEU） 两种评估方式，分别对工具名称和任务描述进行评分。  

$$
\text{Score} = \text{EM}(T_{\text{gt}}, T_{\text{pred}}) + \lambda \times \text{Image}(D_{\text{gt}}, D_{\text{pred}})
$$

- $T_{\text{gt}}$ ：Ground Truth 中的工具名称。 
- $T_{\text{pred}}$：待测试的 Agent Response 中的工具名称。 
- $D_{\text{gt}}$：Ground Truth 中的任务描述。 
- $D_{\text{pred}}$：待测试的 Agent Response 中的任务描述。 
- $\text{EM}(T_{\text{gt}}, T_{\text{pred}})$ ：精确匹配（Exact Match），结果为 0 或 1。
  
$$
\text{EM}(T_{\text{gt}}, T_{\text{pred}}) = 
\begin{cases} 
1 & \text{if } T_{\text{gt}} = T_{\text{pred}} \\
0 & \text{otherwise}
\end{cases}
$$
  
- $\text{Image}(D_{\text{gt}}, D_{\text{pred}})$：模糊匹配评分，可以使用 ROUGE、BLEU 等文本生成质量评估方法，分值为 0-1。  
- $\lambda$：权重系数，用于平衡工具名称和任务描述在总分中的贡献。

#### 人工评估（笔记）
评估标准示例：

---

<div font-size: 16px; padding-left:8px;">评分分为五个等级，具体定义如下：

1分：核心信息存在重大不准确性，大部分或全部内容不正确；
2分：核心信息不正确，或超过60%的内容有错误；
3分：核心信息准确，但10%到60%的补充信息不正确；
4分：小有出入，核心信息准确，补充信息中的错误不超过10%；
5分：完全正确。

得分为4分或5分的回答被认为是可用的。

遇到违规内容一票否决，直接判0分。

……

</div>

---

人工评估效率低、成本高，所以不能频繁使用，一般在大的模型版本上使用。

<br />

#### Agent 泛化性提升 （笔记）
从Query，Tools,Agent Prompt 模板三个维度提升泛性  
- 1.覆盖全部场景
在构建内部业务工具时，需要尽可能覆盖所有潜在的业务场景，特别是针对工具的查询（query）进行设计。这样可以避免遗漏某些场景，确保工具在不同场景下都能正常工作。  

- 2.困难负样本的构造  
    在样本构建时，需要确保正样本和负样本的多样性，特别是困难负样本的构造。正样本是那些明确需要调用工具的查询，而困难负样本是那些看起来和工具相关但实际上不需要调用工具的查询。这样的设计可以帮助训练模型识别工具调用的边界。  

    示例：  
  - 正样本：上海明天天气怎么样？（需要调用天气查询工具）  
  - 困难负样本：今天天气真好（虽然提到天气，但不需要调用工具）  
  - 经验值：正样本和困难负样本的比例通常建议为 4:1。  
样本复杂度的多样性  


- 3.工具描述格式多样性  
    同一个工具在不同平台或框架中可能会有不同的描述格式。  
    例如，OpenAI function、AutoGPT在描述方式上可能有所不同：  
    ```
    {"name": "web_search", "description": "Perform an internet search.", "parameters": {"type": "object", "properties": {"text": {"type": "str", "description": "Search query."}}}, "returns": {"description": "Multiple webpage links along with brief descriptions.", "type": "str"}, "required": ["text"]}
    AutoGPT：Web Search: "web_search", args: "text": ""
    ```



