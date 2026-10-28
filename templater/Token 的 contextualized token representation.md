假设某一层输入的 Token 表示是：

$$H =
\begin{bmatrix}
h_1\\
h_2\\
\vdots\\
h_n
\end{bmatrix}$$
其中 $h_i$是第 i 个 Token 当前的表示。

Transformer 使用固定的权重矩阵计算：
$$Q=HW_Q$$$$K=HW_K$$$$V=HW_V$$
这里：

- $W_Q,W_K,W_V$​：模型参数，推理时不变；
- $Q,K,V$：根据当前输入动态算出的激活值，每篇文档都不同。

所以需要区分：

```
W_Q、W_K、W_V：
训练得到的固定权重矩阵

Q、K、V：
当前文档经过矩阵变换后得到的动态向量
```

你可以把它理解成：

```
Q = document_hidden_states @ fixed_W_Q
K = document_hidden_states @ fixed_W_K
V = document_hidden_states @ fixed_W_V
```

`fixed_W_Q` 不变，但因为 `document_hidden_states` 不同，所以 Q、K、V 会变化。

# 三、Transformer 确实会改变 Token 的表示

假设文档中有一句话：

```
苹果发布了新款手机。它搭载了新的芯片。
```

最开始，Token“它”通过 Embedding Table 得到一个初始向量：
$$e_{\text{它}}$$​

这个初始向量只表达“它”这个 Token 本身，不知道这里的“它”指什么。

加入位置编码后：
$$h_{\text{它}}^{(0)}
=
e_{\text{它}}+p_{\text{position}}$$
经过 Self-Attention，“它”可以关注前面的：

```
苹果
发布
新款
手机
```

Self-Attention 的核心计算可以简化为：
$$\operatorname{Attention}(Q,K,V)
=
\operatorname{softmax}
\left(
\frac{QK^\top}{\sqrt{d_k}}
\right)V$$
对于“它”这个 Token，会计算它对其他 Token 的注意力权重。假设结果是：

```
对“苹果”的注意力：0.10
对“发布”的注意力：0.05
对“新款”的注意力：0.10
对“手机”的注意力：0.65
对其他词的注意力：0.10
```

然后，“它”的新表示会混入这些 Token 对应的 Value：
$$z_{\text{它}}
=
0.10v_{\text{苹果}}
+0.05v_{\text{发布}}
+0.10v_{\text{新款}}
+0.65v_{\text{手机}}
+\cdots$$
经过残差连接、LayerNorm 和前馈网络后，得到新的隐藏状态：
$$h_{\text{它}}^{(1)}$$

因此：

```
初始 Token Embedding：
主要表示“它”这个词本身

经过 Transformer 后的隐藏状态：
表示“这里的它指的是前面提到的手机”
```

经过多层 Transformer 后：
$$h_{\text{它}}^{(L)}$$
已经是一个上下文化表示。

所以不是 Embedding Table 中“它”的原始向量被永久修改了，而是：

> 当前这一次前向计算产生了一个包含上下文信息的新 Token 表示。

# 四、普通 Chunking 与 Late Chunking 的区别

假设原文是：

```
第一段：
苹果公司发布了一款名为 Orion 的新型处理器。

第二段：
它采用 3nm 工艺，主要用于下一代笔记本电脑。
```

我们把它切成两个 Chunk。

## 普通做法：先切，再分别编码

```
Chunk 1：
苹果公司发布了一款名为 Orion 的新型处理器。

Chunk 2：
它采用 3nm 工艺，主要用于下一代笔记本电脑。
```

然后分别送进 Transformer：

```
Transformer(Chunk 1) → Chunk 向量 1
Transformer(Chunk 2) → Chunk 向量 2
```

编码 Chunk 2 时，模型只能看到：

```
它采用 3nm 工艺……
```

但看不到前一个 Chunk，因此可能不知道“它”指的是：

```
Orion 处理器
```

可以表示为：
$$H_2
=
\operatorname{Transformer}
(\text{Chunk 2})$$
这里 Chunk 2 的 Token 只能利用 Chunk 2 内部的上下文。

---

## Late Chunking：先编码全文，再切隐藏状态

先把完整文本一起送入 Transformer：

```
苹果公司发布了一款名为 Orion 的新型处理器。
它采用 3nm 工艺，主要用于下一代笔记本电脑。
```

得到每个 Token 的上下文化隐藏状态：
$$H=
[h_1^{(L)},h_2^{(L)},\ldots,h_n^{(L)}]$$
此时“它”的隐藏状态可以关注前文的：

```
Orion
新型处理器
```

然后再按照预先确定的 Chunk 边界，从隐藏状态序列中切出两段：

```
Chunk 1 对应的隐藏状态：
h₁ ... hₘ

Chunk 2 对应的隐藏状态：
hₘ₊₁ ... hₙ
```

最后分别池化：
$$v_{\text{chunk1}}
=
\operatorname{Pool}
(h_1,\ldots,h_m)$$$$v_{\text{chunk2}}
=
\operatorname{Pool}
(h_{m+1},\ldots,h_n)$$
例如 Mean Pooling：
$$v_{\text{chunk2}}
=
\frac{1}{n-m}
\sum_{i=m+1}^{n} h_i^{(L)}$$
虽然池化只使用了 Chunk 2 范围内的 Token，但这些 Token 的隐藏状态在全文编码阶段已经读取了前文。
> [!NOTE]+ 💡 我的理解
> 
> **池化的目的就是把一个 Chunk 内多个 Token 的向量，压缩成一个固定维度的 Chunk 向量。** 这样它才能作为一条记录存入向量数据库，并与 Query 向量直接计算相似度。


因此 Chunk 2 的向量能够隐含表达：

```
Orion 处理器采用 3nm 工艺，
用于下一代笔记本电脑。
```

而不只是：

```
某个不明确的“它”采用 3nm 工艺。
```