---
title: 🎉 AI | 学习笔记
summary: AI学习笔记
date: 2025-08-29

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - AI
---
**Activation Sparsity = 很多神经元在处理某个输入时“保持沉默”，输出为 0 或接近 0 的比例很高**。

**Skewness**（偏度）是衡量数据分布对称性的一种指标，通常是指：

- **某些 neuron 激活值的偏斜程度**
- **token 出现频率分布的偏度**
- **layer 输出分布是否对称**

哪些 neuron 的激活分布“偏度很大” → 说明它不是经常激活（冷 neuron）！

discrepancy：差异



Transformer Layer
├── Self-Attention（含 Multi-Head）
└── MLP

Attention(Q, K, V) = softmax(QK^T / sqrt(d)) * V

#### 🌈 什么是多头？

> 就是并行地做 **多个不同的 self-attention**，每个头负责“看不同角度”。

#### 🌟 为什么多个头有用？

- 假设你有 8 个头：
  - 第一个头关注语法结构
  - 第二个头关注主谓关系
  - 第三个头关注情感词
  - 第四个头专门处理远距离依赖
  - …
- 多个头可以并行从不同“子空间”学习特征信息

 Neuron 

## 🧠 在神经网络里，Neuron 做了什么？

每个神经元会：

1. 接收一些输入（来自前一层的神经元）
2. 把这些输入乘上自己的“权重” + 加上一个“偏置”（bias）
3. 通过一个激活函数（比如 ReLU、GELU）处理一下
4. 输出结果，传给下一层神经元

Self-Attention
├── Multi-Head Attention
│   ├── Head 1: Attention(Q1, K1, V1)
│   ├── Head 2: Attention(Q2, K2, V2)
│   ├── ...
│   └── Head N
│   └── Concat → Linear Projection
└── 残差连接 + LayerNorm

## 🧠 Neuron 在 Transformer / LLM 中的角色？

虽然 Transformer 更偏向 Attention 结构，但 **MLP 部分就是由 neuron 构成的前馈神经网络**。

在 GPT 等 LLM 中：

- 每个 Transformer Layer 的 MLP 部分通常有：
  - **一层升维**（比如从 4096 -> 16384）（包含多个neuron）
  - **激活函数**
  - **一层降维**（再从 16384 -> 4096）（包含多个neuron）

> 所以一个 LLM 模型里，其实包含了 **数十亿甚至上万亿个 neuron**！

前文中的它把输入的序列转成 Query（Q）、Key（K）、Value（V）三个矩阵

## 🧠 举个具体例子：一句话

```
text


复制编辑
"Tom loves pizza"
```

假设输入被表示成了三个 embedding，这里的embedding是在训练过程中固定下来的（相对稳定）。（每个词变成一个向量）：

```
Tom    → x₁ = [0.1, 0.3, 0.5]
loves  → x₂ = [0.2, 0.1, 0.7]
pizza  → x₃ = [0.5, 0.4, 0.2]
```

## 🔧 怎么变成 Q、K、V？

很简单：

> 就是对每个输入向量 **乘以不同的权重矩阵**（**模型的核心参数**，数量巨大（GPT-3 有 1750 亿个参数！））！

```
Q = X @ W_q
K = X @ W_k
V = X @ W_v
```

其中：

- `X` 是输入的 embedding（形状是 `[seq_len, d_model]`）
- `W_q`, `W_k`, `W_v` 是模型学到的权重（比如 `[d_model, d_k]`）

## ✅ 总结一句话：

| 问题                   | 答案                                |
| ---------------------- | ----------------------------------- |
| Tom 会变成向量吗？     | ✅ 是的，通过 embedding              |
| 是固定的吗？           | 推理时固定，训练时可更新            |
| embedding 是怎么算的？ | 查 embedding 表，向量由模型训练得出 |
| embedding 有语义吗？   | ✅ 有，比如“近义词”会靠得近          |

## MLP

**MLP（Multi-Layer Perceptron，多层感知机）就是一段普通的“前馈神经网络”**，它在 Transformer 的每一层里都负责非线性处理、信息混合。

在 Transformer 中，MLP 通常就两层线性变换，中间加个激活函数（非线性）：

### 🧠 通俗点解释：

1. 输入向量 x（比如 Attention 输出）
2. 先通过一个线性层 `W1` → 把信息“升维”
3. 经过非线性函数 `ReLU`（加入“判断力”）
4. 再通过另一个线性层 `W2` → “降维”回来

## 🔍 举个具体例子：

假设你输入了一个维度为 512 的向量（模型中的某个 token 表示）：

1. **第一层（升维）**：
   - 输入 512 → 输出 2048（比如扩展到更大的特征空间）
2. **ReLU 非线性**：
   - 把负值截断成 0，引入“判断力”和“非线性区分能力”
3. **第二层（降维）**：
   - 输入 2048 → 输出 512（变回原来的维度）

## 🧪 为什么需要 MLP？

因为仅靠 Attention，模型只是把 token 信息在彼此之间“路由”了。

> MLP 的作用是：**每个 token 内部做更复杂的、非线性的变换。**

你可以理解为：

> Attention 是“我和别人怎么互动”，MLP 是“我自己内部做什么决定”。



## predictors

**MLP predictors** 中的 **predictors** 到底指什么 👇

## ✅ 一句话解释：

> 在“MLP predictors”这个短语中，**predictors 就是用来预测结果的 MLP（多层感知器）模块**。

它是神经网络中专门**执行预测任务**的部分。

他们为每一层 Transformer 层都设计了一个 **专属的 predictor（MLP 模型）**，这个 predictor 的大小（隐藏层维度）不是固定的，而是根据该层的激活特征**动态调整的**。

## 📌 第二段：建立初始规模

> The process begins by establishing a baseline model size based on the layer’s sparsity profile.

### ✅ 含义：

- 每层 Transformer 的 **稀疏性不同**（即激活为 0 的 neuron 比例不同）。
- 根据这个 sparsity（比如 90% neuron 都不动），先定一个初始的 predictor 大小。

## 🔁 第三段：迭代调整隐藏层

> the model size is iteratively adjusted, considering the internal activation skewness to maintain accuracy.

### ✅ 含义：

- 然后通过迭代方式调整这个 predictor 的“隐藏层维度”；
- 依据是该层 **activation skewness**（激活值是否集中于某些 neuron，偏度高说明分布不均）；
- 目的是：**保证准确率（精度）不下降。**

建立了两张神经元的表。一个放在cpu一个放在GPU。这个表 correlate each neuron to its original position in the matrix.（推理时，PowerInfer 按这个策略把权重矩阵 **分割并加载**到不同设备内存中（GPU vs CPU））

### 🧱 2. 每一层的神经元是怎么分的？

> For each layer, which may consist of multiple weight matrices, PowerInfer assigns each neuron to either the GPU or CPU based on whether the neuron is hot-activated.

✅ 含义：

- 每一层（比如 MLP 层）都有多个权重矩阵（比如 FC1、FC2）
- 每个 neuron（即每个输出维度）都可以**单独被分配到 GPU 或 CPU**
- 分配标准：它是不是高频激活的 neuron（hot neuron）

✖️🟰 5. 实际矩阵乘法怎么做？

During the process of multiplying with an input tensor, each neuron interacts with its corresponding tensor value, guided by the mappings in the neuron tables.
在实际与输入张量进行处理的时候，每一个神经元都会在neuron tables的指引下与相应的tensor value相运算

##  GPU-CPU Hybrid Execution

PowerInfer会构建一个有向无环图，每一个节点都代表一个计算的LLM inference operator。并且都会保存在CPU的内存中。每一个节点都会被tagged with its prerequisite operators.

在推理阶段，从全局的队列里面pull operators from the global queue, check dependencies, and assign them to the appropriate processing unit.

### 🧵 线程调度器（executors）

> ... two types of executors, **pthreads** created by the host OS, manage calculations on both CPU and GPU.

- 系统用 POSIX 线程（pthread）创建两个调度器：
  - 一个调度 GPU 上的任务
  - 一个调度 CPU 上的任务

### 🔄 执行器如何调度任务？

> They pull operators from the global queue, check dependencies, and assign them to the appropriate processing unit.

- 每个执行器不断从队列里拉取 operator
- 检查依赖是否满足（前置任务是否都已完成）
- 然后把任务分派给 CPU 或 GPU 对应的执行函数

### 💡 GPU 与 CPU 执行细节

> The GPU and CPU use their **neuron-aware operators**, with the GPU executor launching GPU operators using APIs like **cudaLaunchKernel**, and the CPU executor coordinating unoccupied CPU cores...

- 每个任务是“neuron-aware”的，即知道自己负责的是哪部分 neuron（之前我们讲过的热 neuron/cold neuron）
- GPU 端通过 `cudaLaunchKernel` 这样的 API 调用 GPU 运算
- CPU 端分配当前空闲的核（核多任务调度）
- 如果某个 operator 在 CPU 上执行，它的依赖（parent node）却在 GPU 上执行 → 那必须等 GPU 完成
- 所以使用 **barrier（屏障机制）** 来确保依赖完成才开始

PowerInfer 引入了neuron-aware的operator，这东西直接计算activated neurons以及their weights without the need for runtime conversion to dense format. These operators differ from traditional ones as they focus on individual row/column vectors within a matrix rather than the entire matrix.

## 🔧 Neuron-aware Operators on **GPU**

### ✅ 为什么要用 vector-vector 而不是 matrix-vector？

> ... **vector-vector calculations** being less efficient than matrix-vector calculations on GPU, neuron-aware operators based on vector-vector computation are advantageous when the **batch size is small**.

**解释：**

- **传统 LLM 用的是矩阵乘法**（比如 weight 矩阵 × 输入向量）
- 但当激活稀疏性高、batch size 小时，大量 neuron 是“空计算”
- 此时用 **逐 neuron 的 vector-vector 点乘**，可以跳过未激活 neuron，**整体效率反而更高**
- 不像一些稀疏工具需要先把稀疏矩阵转成密集格式再计算（如 CSR → dense）
- 这种设计中 **完全跳过未激活 neuron**，也不需要任何格式转换或数据填充
- **每个 GPU thread block 被分配一批 neuron**
- 它负责检查这些 neuron 有没有被激活，如果激活就计算
- 每个 block 都是“独立处理”的，所以 **无需线程间同步** → 避免了 warp divergence / lock 等性能问题

🧮 Neuron-aware Operators on **CPU**

- 相比 GPU 的几千线程，CPU 并发核少、SIMD 吞吐低
- 所以 CPU 更需要避免“无用计算”，精细控制 **谁该算、谁不该算**
- **多个 CPU 核心**并行处理 neuron 的 batch
- 每个核心只算被激活的 neuron
- 用 **AVX2 等 SIMD 指令集** 加速“逐 neuron 的点乘”操作（即一次算多个 float）

## Neuron Placement Policy

###  Communication Constraint 

GPU上加上传输时间小于在CPU上运行的时间，以layer为单位。

![image-20250421173329718](D:\研究生数据\研究整理\论文\AI agent\图片\image-20250421173329718.png)

### Memory Constraint

when allocating neurons of a layer to the GPU, it either assigns at least the minimum number of neurons specified in Inequality 4 to offset communication costs or opts not to allocate any neurons from that layer to the GPU. Specifically, the number of neurons for layer 𝑙 on the GPU must either exceed 𝐶𝑙 or be equal to zero.