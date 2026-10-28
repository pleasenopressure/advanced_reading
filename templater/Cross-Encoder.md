这里指的是 **Cross-Encoder 作为 Reranker 的实现原理**。你上传的文档只讲到“Rerank 会重新精细排序候选 Chunk”，没有展开 Cross-Encoder 的内部结构；下面是补充原理。万字详解 RAG 基础概念 .mdMD

# 1. Cross-Encoder 的核心思想

假设用户问题是：

```
Query：
员工离职时什么时候归还电脑？
```

检索器已经找到一个候选 Chunk：

```
Document：
员工应在最后工作日归还电脑、门禁卡和充电器。
```

Cross-Encoder 会把 **Query 和 Document 同时放进同一个 Transformer 模型**：

```
Query + Document
       ↓
同一个 Transformer
       ↓
相关性分数
```

例如输出：

```
0.96
```

表示这个文档和问题非常相关。

Sentence Transformers 的官方说明也是：Cross-Encoder 同时接收一对文本并输出一个分数，因此常用于对第一阶段 Top-K 结果进行重排。[sbert.net](https://www.sbert.net/docs/cross_encoder/usage/usage.html?utm_source=chatgpt.com)

---

# 2. 输入是如何构造的？

以 BERT 类模型为例，输入通常会拼成：

```
[CLS] Query [SEP] Document [SEP]
```

例如：

```
[CLS]
员工 离职 时 什么 时候 归还 电脑 ？
[SEP]
员工 应 在 最后 工作日 归还 电脑 和 充电器 。
[SEP]
```

其中：

- `[CLS]`：用于表示整个输入对；
- `[SEP]`：用于区分 Query 和 Document；
- Query 和 Document 被放在同一个输入序列中。

模型还会使用 segment/token-type embedding 来区分两部分：

```
Query 部分     → Segment A
Document 部分  → Segment B
```

BERT passage reranking 的经典实现就是将 Query 和候选 Passage 作为一个文本对输入 BERT，然后输出相关性分数。[arXiv](https://arxiv.org/abs/1901.04085?utm_source=chatgpt.com)

---

# 3. 为什么叫 Cross-Encoder？

关键就在这个 **Cross**。

因为 Query 中的 Token 和 Document 中的 Token，可以在 Transformer 的 Self-Attention 中直接相互交互。

例如：

```
Query：
什么时候  归还  电脑
   ↘        ↓      ↙
Document：
最后工作日  归还  公司电脑
```

模型可以直接学习：

- “什么时候”对应“最后工作日”；
- “归还”对应“归还”；
- “电脑”对应“公司电脑”；
- 整句话是在讨论离职设备归还时间。

也就是说，Attention 矩阵覆盖整个拼接序列：

\[ \operatorname{Attention}(Q,K,V) = \operatorname{softmax} \left( \frac{QK^\top}{\sqrt{d_k}} \right)V \]

这里的 \(Q,K,V\) 不只是来自用户 Query，也包含 Document 中的 Token。

因此一个 Query Token 可以直接关注某个 Document Token。

这就是“交叉编码”：

> Query 和 Document 在编码过程中发生深层 Token 级交互，而不是编码完成之后才比较两个向量。

Cross-Encoder 的这一联合编码特征，是它通常比独立向量相似度更准确的重要原因。[sbert.net](https://www.sbert.net/examples/cross_encoder/applications/README.html?utm_source=chatgpt.com)

---

# 4. 模型最后怎样输出分数？

经过多层 Transformer 后，每个 Token 都会得到一个上下文化表示。

通常取 `[CLS]` 的最终隐藏向量：

\[ \mathbf h_{\text{CLS}} \]

然后接一个线性层：

\[ s=\mathbf w^\top\mathbf h_{\text{CLS}}+b \]

其中：

- \(\mathbf h_{\text{CLS}}\)：整对 Query–Document 的联合表示；
- \(\mathbf w,b\)：训练得到的参数；
- \(s\)：相关性 logit。

如果任务是二分类，可以再通过 Sigmoid：

\[ P(\text{relevant}\mid q,d)=\sigma(s) \]

得到：

```
0.96
```

也可以不做 Sigmoid，直接使用原始 logit 排序：

```
Document A：8.7
Document B：4.1
Document C：-2.3
```

只要分数越大表示越相关即可。CrossEncoder 实现通常允许输出一个或多个标签分数，具体取决于模型配置。[sbert.net](https://www.sbert.net/docs/package_reference/cross_encoder/model.html?utm_source=chatgpt.com)

---

# 5. Cross-Encoder 是怎么训练的？

需要训练数据：

```
Query + Document + 相关性标签
```

例如：

|Query|Document|标签|
|---|---|---|
|如何申请退款？|用户可以在订单页面提交退款申请。|1|
|如何申请退款？|公司电脑每四年更新一次。|0|
|如何申请退款？|商品退货后款项将在五日内到账。|1|

## Pointwise：逐对训练

每个 Query–Document 对单独判断相关或不相关。

例如：

\[ \mathcal L = -\left[ y\log p+(1-y)\log(1-p) \right] \]

其中：

- \(y=1\)：相关；
- \(y=0\)：不相关；
- \(p\)：模型预测的相关概率。

训练后，模型学会给相关文档更高分。

---

## Pairwise：成对排序训练

给模型：

```
Query
+
正文档 d⁺
+
负文档 d⁻
```

要求：

\[ s(q,d^+) > s(q,d^-) \]

例如使用：

\[ \mathcal L = -\log \sigma \left( s(q,d^+)-s(q,d^-) \right) \]

模型不只学“相关还是不相关”，还学习：

> 对于同一个问题，哪个文档应该排在前面。

---

## Listwise：列表级训练

一次给模型一组候选：

```
Query
→ [d₁,d₂,d₃,\ldots,d_n]
```

训练目标直接优化整个排序列表，例如让正确文档在 Top-1、Top-5 中排名更高。

实际工程中，Pointwise 和 Pairwise 都比较常见；BERT reranking 工作证明了将预训练语言模型微调用于 Query–Passage 相关性评分可以显著提升重排效果。[arXiv](https://arxiv.org/abs/1901.04085?utm_source=chatgpt.com)

---

# 6. 它为什么比向量相似度更准确？

需要和 **Bi-Encoder** 对比。

## Bi-Encoder：分别编码

向量检索通常是：

```
Query
  ↓ Encoder
Query Embedding

Document
  ↓ Encoder
Document Embedding

最后计算余弦相似度
```

形式上：

\[ \mathbf q=f(q) \]\[ \mathbf d=g(d) \]\[ s(q,d)=\cos(\mathbf q,\mathbf d) \]

Query 和 Document 在编码时完全没有见到对方。

它们只能先各自压缩成一个固定长度向量，再计算相似度。

---

## Cross-Encoder：联合编码

Cross-Encoder 是：

\[ s(q,d)=h([q;d]) \]

Query 和 Document 共同进入模型。

因此它能进行更细致的比较。

例如问题：

```
苹果电脑无法开机怎么办？
```

候选 A：

```
MacBook 无法启动时，可以长按电源键十秒。
```

候选 B：

```
吃苹果有助于补充膳食纤维。
```

Bi-Encoder 通常也可以区分，但它先独立压缩文本。

Cross-Encoder 可以在联合上下文中直接判断：

```
“苹果” + “电脑” → Apple/MacBook
而不是水果
```

它还能理解：

- 否定关系；
- 条件限制；
- 数字和时间；
- 实体对应；
- 复杂问句；
- Query 与文档中不同词语之间的对应关系。

Cross-Encoder 的逐 Token 交互带来更强相关性判断，但计算成本也明显更高。[sbert.net](https://www.sbert.net/docs/cross_encoder/usage/usage.html?utm_source=chatgpt.com)

---

# 7. 为什么 Cross-Encoder 不能直接搜索整个数据库？

假设知识库有一百万个 Chunk。

Cross-Encoder 必须分别计算：

```
(Query, Chunk 1)
(Query, Chunk 2)
(Query, Chunk 3)
...
(Query, Chunk 1,000,000)
```

也就是需要运行一百万次联合 Transformer 前向计算。

而 Bi-Encoder 可以提前离线计算所有文档向量：

```
Document 1 → Embedding，预先存好
Document 2 → Embedding，预先存好
...
```

查询时只需要：

1. 计算一次 Query Embedding；
2. 在向量索引里做快速近邻搜索。

因此实际系统通常采用两阶段架构：

```
第一阶段：Retriever
从 1,000,000 个 Chunk 中快速找出 Top-50

第二阶段：Cross-Encoder
精细计算这 50 个 Query–Chunk 对

最后保留 Top-5
```

官方 Sentence Transformers 文档也因此将 Cross-Encoder 的典型用途定位为对 Bi-Encoder 或稀疏检索返回的 Top-K 结果做 reranking。[sbert.net](https://www.sbert.net/docs/cross_encoder/usage/usage.html?utm_source=chatgpt.com)

---

# 8. 一个完整例子

用户问题：

```
员工离职多久后需要归还公司电脑？
```

初始 Retriever 返回：

```
Chunk A：
员工应在最后工作日归还公司电脑。
向量分数：0.83

Chunk B：
员工离职后，账户将在三日内关闭。
向量分数：0.86

Chunk C：
公司电脑通常每四年进行更新。
向量分数：0.81
```

向量检索错误地把 Chunk B 排在第一，因为“员工离职后”“三日内”在语义上与问题很接近。

Cross-Encoder 分别联合读取：

```
(Query, Chunk A) → 0.97
(Query, Chunk B) → 0.41
(Query, Chunk C) → 0.18
```

重排后：

```
1. Chunk A
2. Chunk B
3. Chunk C
```

原因是 Cross-Encoder 能更准确地看到：

```
问题问的是：
归还电脑的时间

Chunk A：
明确给出了归还电脑的时间

Chunk B：
给出的是账户关闭时间

Chunk C：
给出的是电脑更新周期
```

这正是 Reranker 的价值：

> 第一阶段结果可能整体语义接近，但 Cross-Encoder 可以进一步判断哪个候选真正回答了问题。

# 最后总结

Cross-Encoder 的实现过程可以压缩成：

```
Query 和候选 Chunk 拼接
        ↓
[CLS] Query [SEP] Chunk [SEP]
        ↓
共同进入同一个 Transformer
        ↓
Query 与 Chunk 的 Token 通过 Attention 直接交互
        ↓
取 [CLS] 的联合表示
        ↓
线性层输出相关性分数
        ↓
按照分数重新排列候选 Chunk
```

它与 Bi-Encoder 最核心的区别是：

```
Bi-Encoder：
先分别理解，再比较两个向量
速度快，适合大规模召回

Cross-Encoder：
把两段文本放在一起逐词比较
精度高，适合小规模重排
```

因此标准生产链路通常是：

```
BM25 / 向量检索快速召回
            ↓
Cross-Encoder 精细重排
            ↓
取 Top-K 放进 LLM 上下文
```