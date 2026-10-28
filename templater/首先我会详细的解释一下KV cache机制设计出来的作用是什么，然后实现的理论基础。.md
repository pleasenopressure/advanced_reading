为了避免后续的疑问，我直接摆出一个结论，KV cache是用于model推理阶段的。
它的理论基础来自 **Transformer 的因果注意力（Causal Self-Attention）** 和 **自回归生成（Autoregressive Generation）**
# 一、先回顾 Attention 的计算

Transformer 一层 Attention 的计算公式是：

$$Q=XW_Q,\qquad K=XW_K,\qquad V=XW_V$$

然后计算

$$\text{Attention}(Q,K,V)
=
\text{Softmax}
\left(
\frac{QK^T}{\sqrt d}
\right)
V$$

可以看到：

Attention 分成两个阶段：

1. 计算 Q、K、V
2. 用 Q 和 K 算权重，再对 V 加权求和

# 二、为什么 Decoder 可以缓存？

关键就在于 **自回归生成**。

假设已经生成了

```text
Token1
Token2
Token3
```

现在生成

```text
Token4
```

模型输入实际上是

```text
Token1 Token2 Token3
```

预测

```text
Token4
```

然后下一步

输入变成

```text
Token1 Token2 Token3 Token4
```

预测

```text
Token5
```

注意观察：

前面的 Token 有没有变？

**没有。**

真正增加的只有

```text
Token4
```

---

# 三、为什么前面的 K、V 不会变？

来看公式：

假设

输入矩阵

[  
X=  
[x_1,x_2,\cdots,x_n]  
]

其中

[  
x_i  
]

就是第 i 个 Token 的隐藏表示。

那么

第 i 个 Token 的 Key

就是

[  
k_i=x_iW_K  
]

Value

就是

[  
v_i=x_iW_V  
]

注意：

这里只依赖于

- 当前 Token 的表示
    
- 参数矩阵 (W_K,W_V)
    

它**没有依赖未来 Token**。

例如

第 5 个 Token

计算

[  
k_5=x_5W_K  
]

不会用到

```text
Token6
Token7
Token8
```

因为它们根本还不存在。

所以：

> **过去 Token 的 Key、Value 一旦算出来，就永远不会改变。**

这就是 KV Cache 能成立的第一个理论基础。

---

# 四、为什么 Q 不能缓存？

很多人第一次学都会问：

> 为什么只缓存 KV？

因为 Query 不一样。

假设生成

Token100

它需要

$$q_{100}
=
x_{100}W_Q$$
下一步生成

Token101

新的 Query

变成

$$q_{101}
=
x_{101}W_Q$$
注意：

这是一个**新的 Token**。

新的 Token

自然产生新的 Query。

以前根本不存在。

所以：

Query 没法提前缓存。
注意这里的$$q_{101}
$$不是一个q = {q1, q2,...q101},即不是一个向量中的一个值
而是一个token对应的一个q向量，


---

# 五、真正决定 KV Cache 的，是 Causal Mask

这是最重要的一点。

普通 Self-Attention

每个 Token

都可以看到：

```text
ABCDE
```

例如

C

可以看到

```text
ABCDE
```

于是

未来 Token

可能影响过去 Token。

这种情况下

缓存就错了。

---

但是 GPT 使用的是

**Causal Mask**

例如：

```text
A

只能看 A
```

```text
AB

B只能看

AB
```

```text
ABC

C只能看

ABC
```

永远不能看未来。

所以

当已经算完

```text
ABC
```

以后

后来来了

```text
D
```

不会影响

A

不会影响

B

不会影响

C

因此

```text
K_A
K_B
K_C
```

全部保持不变。

这就是第二个理论基础：

> **因果掩码保证未来 Token 不会改变过去 Token 的表示。**

---

# 六、生成 Token1001 时发生什么？

假设已经缓存了

```text
K1
K2
...
K1000

V1
V2
...
V1000
```

现在输入

Token1001

只需要计算

```text
Q1001

K1001

V1001
```

然后

拼接：

```text
K=
[
K1
K2
...
K1000
K1001
]
```

Value 同理。

Attention 计算：

$$\text{Softmax}(Q_{1001}K^T)V$$

这里

Q1001

会和

全部历史 K

计算相似度。

然后利用

全部历史 V

加权求和。

因此：

历史信息没有丢。

只是不用重新计算了。

---

# 七、为什么结果完全一样？

这是最重要的一点。

很多优化算法

例如：

剪枝

量化

都会改变模型。

但是 KV Cache

不会。

原因：

假设第一次算

```text
Token5
```

得到
$$K_5
=
x_5W_K$$

缓存下来。

以后直接读取：

还是
$$K_5$$
完全一样。

不是近似。

不是压缩。

不是重新估计。

就是第一次计算出来的结果。

所以：

> **KV Cache 本质上只是把原本重复计算的结果保存下来，下次直接复用。**

数学表达式没有任何变化。

---
# 如果没有 KV Cache，会发生什么？

当生成第三个 Token 时，模型需要重新从输入开始计算：

```
Token1
 ↓
Q1 K1 V1   （重新计算）

Token2
 ↓
Q2 K2 V2   （重新计算）

Token3
 ↓
Q3 K3 V3
```

即使 K1、V1 与第一次完全相同，也会再次执行矩阵乘法。

而使用 KV Cache 时：

```
Token1
 ↓
直接读取 K1、V1

Token2
 ↓
直接读取 K2、V2

Token3
 ↓
只计算 Q3、K3、V3
```

这样计算结果**完全一致**，只是避免了重复计算。
# 八、为什么训练时不用 KV Cache？

因为训练时不是自回归生成，而是：

一句话：

```text
I
love
AI
very
much
```

一次性全部输入。

于是：

所有 Token

都会一起计算：

```text
Q

K

V
```

然后

一起反向传播。

没有：

> 先生成一个

再生成一个

的过程。

所以：

KV Cache

没有意义。

---

# 九、总结：KV Cache 的理论基础

KV Cache 能够成立，依赖于三个关键事实：

1. **K、V 是每个 Token 的固定线性投影。**  
    每个 Token 的 K、V 只由它自身当前层的隐藏表示和固定参数矩阵决定，一旦算出，在本次推理过程中不会改变。
    
2. **Causal Self-Attention（因果掩码）。**  
    Decoder 中过去 Token 不会受到未来 Token 的影响，因此历史 Token 的 K、V 始终有效，不需要重新计算。
    
3. **自回归生成（Autoregressive Generation）。**  
    每一步推理只新增一个 Token，因此每一步真正新增的计算只有这个 Token 的 Q、K、V，其余历史 K、V 都可以直接复用。
    

因此，**KV Cache 不是一种近似技巧，而是一种严格等价的计算重用（computation reuse）机制**。它利用了因果 Transformer 的数学性质，把原本每一步都会重复计算的历史 K、V 保存下来，从而把推理复杂度从“每步重新处理整个历史序列”降低为“每步只处理新增 Token”，在保证输出完全一致的前提下大幅提升推理效率。

# 具体计算中的样子
当然可以。**我们不用抽象公式，而是给每个 Token 一个具体的数值向量，把整个 KV Cache 的计算过程完整算一遍。**这是理解 KV Cache 最直观的方法。

为了简单，我们假设：

- Embedding 维度：**2**
    
- 每个 Attention Head 维度：**2**
    
- 一共只有 **1 层 Attention**
    
- 一次生成一个 Token
    

---

# 第一步：生成第一个 Token

假设第一个 Token 是：

```text
Token1 = "I"
```

Embedding 后得到隐藏表示（Hidden State）：

$$x_1=
\begin{bmatrix}
1\\
2
\end{bmatrix}$$

模型参数（整个推理过程中固定不变）：

$$W_Q=
\begin{bmatrix}
1&0\\
0&1
\end{bmatrix}$$
$$W_K=
\begin{bmatrix}
2&0\\
0&1
\end{bmatrix}$$
$$W_V=
\begin{bmatrix}
1&1\\
0&2
\end{bmatrix}$$

---

## 计算 Q、K、V

### Query

$$q_1=x_1W_Q$$

即

$$\begin{bmatrix}
1&2
\end{bmatrix}
\begin{bmatrix}
1&0\\
0&1
\end{bmatrix}
=
\begin{bmatrix}
1&2
\end{bmatrix}$$
所以

```text
Q1 = [1,2]
```

---

### Key

$$k_1=x_1W_K$$

得到

$$\begin{bmatrix}
1&2
\end{bmatrix}
\begin{bmatrix}
2&0\\
0&1
\end{bmatrix}
=
\begin{bmatrix}
2&2
\end{bmatrix}$$
因此

```text
K1 = [2,2]
```

---

### Value

$$v_1=x_1W_V$$

得到

$$\begin{bmatrix}
1&2
\end{bmatrix}
\begin{bmatrix}
1&1\\
0&2
\end{bmatrix}
=
\begin{bmatrix}
1&5
\end{bmatrix}$$

因此

```text
V1 = [1,5]
```

---

此时 KV Cache 中保存的是

```text
Cache

K =
[
[2,2]
]

V =
[
[1,5]
]
```

---

# 第二步：生成第二个 Token

模型预测出了

```text
Token2 = "love"
```

Embedding 后
$$x_2=
\begin{bmatrix}
3\\
1
\end{bmatrix}$$

---

计算新的 Query
$$q_2=x_2W_Q$$

得到

```text
Q2 = [3,1]
```

计算新的 Key
$$k_2=x_2W_K$$

得到

```text
K2 = [6,1]
```

计算新的 Value

$$v_2=x_2W_V$$
得到

```text
V2 = [3,5]
```

---

然后把新的 KV 放进缓存：

```text
Cache

K =
[
[2,2]
[6,1]
]

V =
[
[1,5]
[3,5]
]
```

注意：

**K1 和 V1 完全没有重新计算。**

只是：

```text
新增

K2

V2
```

---

# 第三步：计算 Token2 的 Attention

Token2 的 Query

```text
Q2=[3,1]
```

要和所有 Key 做点积：

和 K1

$$3\times2+1\times2=8$$

和 K2

$$3\times6+1\times1=19$$

得到

```text
Attention Score

[8,19]
```

Softmax 后

假设得到

```text
[0.01,0.99]
```

那么最终输出

就是

```text
0.01×V1

+

0.99×V2
```

可以看到：

虽然 **K1、V1 没重新算**，

但是

**Token2 依然能够利用 Token1 的信息。**

---

# 第四步：生成第三个 Token

假设

```text
Token3 = AI
```

Embedding
$$x_3=
\begin{bmatrix}
2\\
4
\end{bmatrix}$$
计算

```text
Q3=[2,4]

K3=[4,4]

V3=[2,10]
```

此时缓存

```text
K

[
K1
K2
K3
]
```

即

```text
[
[2,2]
[6,1]
[4,4]
]
```

Value

```text
[
[1,5]
[3,5]
[2,10]
]
```

---

然后

Query3

分别和

```text
K1

K2

K3
```

计算点积：

```text
Q3·K1

↓

2×2+4×2=12
```

```text
Q3·K2

↓

2×6+4×1=16
```

```text
Q3·K3

↓

2×4+4×4=24
```

得到

```text
Score

[12

16

24]
```

Softmax

↓

得到权重

↓

乘以

```text
V1

V2

V3
```

得到最终输出。

---

# 如果没有 KV Cache，会发生什么？

当生成第三个 Token 时，模型需要重新从输入开始计算：

```text
Token1
 ↓
Q1 K1 V1   （重新计算）

Token2
 ↓
Q2 K2 V2   （重新计算）

Token3
 ↓
Q3 K3 V3
```

即使 K1、V1 与第一次完全相同，也会再次执行矩阵乘法。

而使用 KV Cache 时：

```text
Token1
 ↓
直接读取 K1、V1

Token2
 ↓
直接读取 K2、V2

Token3
 ↓
只计算 Q3、K3、V3
```

这样计算结果**完全一致**，只是避免了重复计算。

---

## 最后用一张时间轴总结

```text
第1步生成 Token1

计算：
Q1 K1 V1

缓存：
K1 V1
```

↓

```text
第2步生成 Token2

读取：
K1 V1

新增计算：
Q2 K2 V2

缓存变成：

K1 K2
V1 V2
```

↓

```text
第3步生成 Token3

读取：
K1 K2
V1 V2

新增计算：
Q3 K3 V3

缓存变成：

K1 K2 K3
V1 V2 V3
```

你会发现一个非常重要的规律：

- **Query（Q）**：每一步都只为**当前新 Token**计算一次，用于发起这一步的注意力查询。
    
- **Key（K）和 Value（V）**：每个 Token **一生只计算一次**，之后永久保存在 KV Cache 中，被后续所有 Token 重复利用。
    

这正是 KV Cache 能够显著提升 Transformer 推理速度的根本原因。