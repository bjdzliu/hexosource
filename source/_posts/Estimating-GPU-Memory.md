---
title: 估算GPU内存
date: 2024-07-14 19:00:00
tags: LLM
categories:
- Technical Notes
---

### 训练模型时什么消耗了所有内存？
* model weights 
* gradients 
* optimizer states 
* forward activations saved for gradient computation
* temporary buffers
* functionality-specific memory
refer to [model_memory_anatomy](https://huggingface.co/docs/transformers/model_memory_anatomy)


以 **meta-llama/Llama-2-7b** 为例，用在线工具和本地计算的方式算一算

### 利用Huggingface tool计算
[模型内存计算器](https://huggingface.co/spaces/hf-accelerate/model-memory-usage)
- 基于batchsize=1
- 结论：工具给出了优化器需要的内存，至少是推理内存x4
因为用像 Adam 这样的优化器进行训练时，会使用额外的内存来存储模型的 动量（momentum） 和 二阶矩估计（second moment estimate）。这些信息是 Adam 等优化器计算梯度时所需要的，因此会占用额外的内存。


| dtype                | Largest Layer or Residual Group | Total Size | Training using Adam (Peak vRAM) |
|----------------------|----------------------------------|------------|---------------------------------|
| float16/bfloat16     | 388.02 MB                        | 12.37 GB   | 49.48 GB                        |

 49.48 GB：这表示当使用 Adam 优化器进行训练时，对显存峰值需求。

| dtype           | Model             | Gradient calculation | Backward pass | Optimizer step |
|-----------------|-------------------|----------------------|---------------|----------------|
| float16/bfloat16 | 24.74 GB          | 37.11 GB             | 49.48 GB      | 49.48 GB       |

Backward pass 反向传播的总内存消耗就是激活值的内存消耗和梯度的内存消耗之和
Model=24.74 不太理解数值怎么计算的。7b模型内存消耗也就是14G，24G里面应该还加了其他占用。

Total=24+49+49=122G


### 本地计算
从两个角度计算：
- 模型占用的内存  
- 激活内存  
    * 指用于存储每一层的激活值（即中间计算结果）的内存  
    * 每一层的激活值是在 **前向传播（Forward Pass）时计算并存储的，这些激活值在反向传播（Backward Pass）** 中被用来计算梯度。  

#### 模型占用的内存
Model States ≈ (2+4+8)x7=98G  
- mode size 2x7=14G  
- 梯度内存 4x7=28G  
- 优化器状态内存 8x7=56G  

**梯度** 通常还是保留在 float32 精度上，因为梯度值可能会具有较大的动态范围，所以是4x7

#### 激活内存
Activations激活内存 ≈ 2.1G

Activations激活内存大小取决于许多因素，其中关键因素包括序列长度、隐藏大小和批量大小，用这些参数做一个python计算：

```python
def calculate_memory(sequence_length, hidden_size, batch_size, dtype=torch.float32, num_layers=1):
    """
    Calculate the memory usage for activations of a neural network during forward pass.
    
    Parameters:
    - sequence_length: Length of the input sequence (e.g., number of words in a sentence).
    - hidden_size: Size of the hidden layer (e.g., the dimensionality of the embeddings).
    - batch_size: Number of sequences processed in parallel (batch size).
    - dtype: Data type of the model weights (default is float32, can be changed to float16 for lower precision).
    - num_layers: Number of layers in the network (default is 1 for a simple network).
    
    Returns:
    - memory_in_GB: Memory used for activations in gigabytes (GB).
    """
    # Calculate the size of one activation (batch_size, sequence_length, hidden_size)
    num_elements_per_layer = batch_size * sequence_length * hidden_size
    
    # If there are multiple layers, the activations will be stored for each layer
    total_elements = num_elements_per_layer * num_layers
    
    # Memory usage in bytes: num_elements * size of each element (in bytes)
    element_size = torch.tensor(0, dtype=dtype).element_size()  # Get the size of each element in bytes
    
    total_memory_bytes = total_elements * element_size
    
    # Convert bytes to gigabytes (1 GB = 1e9 bytes)
    memory_in_GB = total_memory_bytes / 1e9
    
    return memory_in_GB

```

Total = Model States Activations = 大约100G

| **精度** | **字节** | 
| --- | --- | 
| float32 | 4 | 
| float16 | 2 | 







