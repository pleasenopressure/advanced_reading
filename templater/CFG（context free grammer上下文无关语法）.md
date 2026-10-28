你的理解没有错，但少了一半：**CFG 不只是用来“分析一个已有句子”，也可以用来“规定哪些句子允许被生成”。**

在 NLP 课程里，你看到的是 CFG 的“解析用途”：

```text
输入句子
  ↓
根据 CFG 做句法分析
  ↓
构建句法树
```

在 Structured Outputs 里，用的是同一套形式化工具的“生成约束用途”：

```text
JSON Schema
  ↓
转换为 CFG
  ↓
模型每生成一个 token 前，
用 CFG 判断哪些 token 仍能组成合法 JSON
```

它不是在分析用户输入的自然语言，而是在**实时分析模型已经生成的 JSON 前缀，并限制下一步输出**。OpenAI 对 Structured Outputs 的公开说明就是：先把 Schema 转换成 CFG，然后在每一步采样时动态计算合法 token，非法 token 的概率被置零。([OpenAI](https://openai.com/index/introducing-structured-outputs-in-the-api/?utm_source=chatgpt.com "Introducing Structured Outputs in the API"))

---

# 1. CFG 本质上是什么

CFG，全称是：

> Context-Free Grammar，上下文无关文法

它由四部分组成：

$$  
G=(N,\Sigma,P,S)  
$$

其中：

- $N$：非终结符，例如 `Sentence`、`Object`、`Value`
    
- $\Sigma$：终结符，也就是真正出现在结果里的字符或符号
    
- $P$：产生式规则
    
- $S$：开始符号
    

在自然语言中，你可能见过：

```text
S  → NP VP
NP → Det N
VP → V NP
```

例如：

```text
The cat eats fish
```

可以分析为：

```text
              S
            /   \
          NP     VP
         / \     / \
       Det  N   V   NP
```

这里 CFG 描述的是“合法英语句子的某种结构”。

而 JSON 同样是一种语言，也有自己的语法：

```text
Object → { Members }
Members → Pair
Members → Pair , Members
Pair → String : Value
Value → String | Number | Boolean | Object | Array
```

这里 CFG 描述的是“合法 JSON 文本的结构”。

所以二者本质相同：

|NLP 中的 CFG|Structured Outputs 中的 CFG|
|---|---|
|描述句子结构|描述 JSON 结构|
|`S → NP VP`|`Object → { Members }`|
|解析自然语言句子|解析或约束 JSON 前缀|
|构建句法树|构建 JSON 语法结构|

---

# 2. 为什么叫“上下文无关”

这是最容易被名字误导的地方。

“上下文无关”**不是说系统完全不考虑前文**，也不是说模型没有上下文。

它的正式含义是：

> 一个非终结符如何展开，只取决于这个非终结符本身，不需要查看它左右两边是什么。

CFG 的产生式形式是：

$$  
A\rightarrow\alpha  
$$

左边只能是一个非终结符 $A$。

例如：

```text
Priority → "LOW"
Priority → "MEDIUM"
Priority → "HIGH"
```

只要当前需要展开的是 `Priority`，就只能从这三个选项中选择。

无论它前面出现过什么，规则本身都是：

```text
Priority → LOW | MEDIUM | HIGH
```

不需要写成：

```text
当 Priority 左边是某个字段，
右边是某个括号时，
才允许展开成 HIGH
```

这就是“上下文无关”的意思。

更正式地说：

```text
xAy ⇒ xαy
```

只要有规则：

```text
A → α
```

不管 $A$ 左右的 $x$ 和 $y$ 是什么，都可以把 $A$ 替换成 $\alpha$。

---

# 3. 为什么运行时看起来又需要“上下文”

虽然文法规则是上下文无关的，但解码器必须知道：

- 当前已经生成到哪里；
    
- 当前处于哪个非终结符；
    
- 哪些括号还没有关闭；
    
- 当前期待字段名、冒号还是字段值。
    

这叫**解析状态**，不等于文法是上下文相关的。

可以类比成：

```text
文法 = 游戏规则
当前解析状态 = 游戏现在进行到哪一步
```

国际象棋规则本身是固定的，但你仍然需要看当前棋盘状态，才能判断下一步哪些棋合法。

Structured Outputs 也是一样：

- CFG 定义“所有合法输出”；
    
- 当前生成前缀决定“此刻哪些 token 合法”。
    

---

# 4. 一个 Schema 如何转换成 CFG

假设 Schema 是：

```json
{
  "type": "object",
  "properties": {
    "priority": {
      "type": "string",
      "enum": ["LOW", "MEDIUM", "HIGH"]
    }
  },
  "required": ["priority"],
  "additionalProperties": false
}
```

可以粗略转换为：

```text
Root     → "{" PriorityPair "}"
PriorityPair → "\"priority\"" ":" Priority
Priority → "\"LOW\""
Priority → "\"MEDIUM\""
Priority → "\"HIGH\""
```

也可以简写成：

```text
Root →
{
  "priority": Priority
}

Priority → "LOW" | "MEDIUM" | "HIGH"
```

这套规则定义了所有合法输出：

```json
{"priority":"LOW"}
```

```json
{"priority":"MEDIUM"}
```

```json
{"priority":"HIGH"}
```

但不包含：

```json
{"priority":"URGENT"}
```

也不包含：

```json
{"priority":123}
```

还不包含：

```json
{
  "priority": "HIGH",
  "extra": "hello"
}
```

因为它们没有办法从 `Root` 按规则推导出来。

---

# 5. CFG 在模型解码时具体做什么

假设模型正在生成：

```json
{"priority":
```

此时 CFG 的解析状态表明：

```text
当前必须生成 Priority
```

而：

```text
Priority → "LOW" | "MEDIUM" | "HIGH"
```

因此下一步允许的开头只有：

```text
"
```

开始字符串后，允许的内容只能走向：

```text
LOW
MEDIUM
HIGH
```

假设模型已经生成：

```json
{"priority":"H
```

这时候三个枚举中只有：

```text
HIGH
```

以 `H` 开头。

因此下一步只能继续生成能够组成：

```text
IGH"
```

的 token。

例如模型本来给候选 token 打分：

|token|模型概率|
|---|--:|
|`IGH`|0.55|
|`ello`|0.20|
|`elp`|0.15|
|`UR`|0.10|

CFG 检查后：

|token|CFG 是否允许|
|---|---|
|`IGH`|允许|
|`ello`|禁止|
|`elp`|禁止|
|`UR`|禁止|

非法 token 被 mask：

$$  
z'_i=  
\begin{cases}  
z_i,&\text{token 合法}\  
-\infty,&\text{token 非法}  
\end{cases}  
$$

经过 softmax 后，非法 token 的概率就变成：

$$  
P(\text{非法 token})=0  
$$

模型只能继续生成：

```json
{"priority":"HIGH"}
```

注意，真实模型的 token 不一定对应单个字符。一个 token 可能是：

```text
"priority"
```

也可能是：

```text
IGH
```

甚至包含引号和标点。因此实际系统会检查一个 token 对应的字符或字节序列，是否仍然能使当前前缀匹配这套 grammar。

---

# 6. 它和“构建句法树”是什么关系

两种场景实际上都有一棵树。

自然语言句子的树：

```text
S
├── NP
│   ├── Det
│   └── N
└── VP
    ├── V
    └── NP
```

JSON 的语法树：

```text
Root
└── Object
    └── Pair
        ├── Key: "priority"
        └── Value
            └── Priority
                └── "HIGH"
```

区别主要在执行方向。

## 传统句法分析

你已经有完整句子：

```text
The cat eats fish
```

然后问：

> 这句话能不能由 CFG 推导出来？对应的语法树是什么？

这是：

```text
字符串 → 语法树
```

## 约束生成

你先有一套 CFG，然后要求模型：

> 只能生成能够由这套 CFG 推导出来的字符串。

这是：

```text
语法规则 → 合法字符串
```

Structured Outputs 通常一边生成，一边进行**增量解析**：

```text
生成一个 token
    ↓
更新当前语法状态
    ↓
计算下一步合法 token
    ↓
屏蔽非法 token
    ↓
继续生成
```

它不一定等模型全部生成完才构建完整树，而是在生成过程中维护类似“当前推导到哪里”的状态。

---

# 7. 为什么不用简单的正则表达式

简单 JSON 可以通过正则或有限状态机约束，但 JSON 允许递归嵌套：

```json
{
  "user": {
    "orders": [
      {
        "items": [
          {
            "name": "coffee"
          }
        ]
      }
    ]
  }
}
```

其语法包含递归：

```text
Value → Object
Value → Array

Object → { Members }
Members → Pair , Members
Pair → String : Value
```

这里 `Value` 可以包含 `Object`，`Object` 里的字段值又可以是 `Value`。

CFG 很擅长表示这种递归结构：

```text
Object
  包含 Value
    又可以是 Object
      又包含 Value
        ……
```

它也能自然处理任意深度的括号匹配：

```text
{}
{{}}
{{{}}}
```

在理论上，普通[[有限状态机]]无法识别任意深度的成对括号，因为它没有无限栈来记录还需要关闭多少层；CFG 配合下推自动机则可以处理这种递归嵌套。OpenAI 也特别提到，CFG 相比有限状态机方式的优势之一，就是能够表达递归 Schema。([OpenAI](https://openai.com/index/introducing-structured-outputs-in-the-api/?utm_source=chatgpt.com "Introducing Structured Outputs in the API"))


---

# 8. CFG 能保证什么，不能保证什么

CFG 擅长保证**结构和语法**。

它可以约束：

- `{}` 和 `[]` 正确闭合；
    
- 字段名称正确；
    
- 必填字段存在；
    
- 值是字符串、数字或布尔值；
    
- 枚举只能从固定集合中选择；
    
- 对象、数组可以递归嵌套；
    
- 禁止未声明字段。
    

但 CFG 不能直接保证业务事实。

例如：

```json
{
  "orderId": "O202600000001",
  "priority": "HIGH"
}
```

即使严格符合 CFG，也不能说明：

- 订单真实存在；
    
- 订单属于当前用户；
    
- `priority` 的判断正确；
    
- 当前用户有查询权限；
    
- 订单允许退款。
    

可以这样区分：

```text
CFG 检查：
“这句话的结构是否合法？”

业务校验检查：
“这句话说的事情是否真实、合理、允许执行？”
```

例如下面这些属于上下文相关或业务相关约束：

```text
refundAmount 不能超过订单实付金额
orderId 必须属于当前 userId
startDate 必须早于 endDate
两个字段的值必须完全相同
某字段是否允许出现取决于数据库状态
```

这些通常不能仅靠 CFG 完成，需要：

- JSON Schema 的部分条件规则；
    
- 服务端程序；
    
- 数据库查询；
    
- 权限系统；
    
- 业务规则引擎。
    

这也是文件强调“Structured Outputs 之后仍需服务端校验”的原因。

---

# 9. 最容易记住的理解

你可以把 CFG 看作一个**实时语法门卫**。

模型负责提出候选 token：

```text
HIGH
URGENT
hello
123
```

CFG 根据 Schema 判断：

```text
HIGH    → 放行
URGENT  → 拦截
hello   → 拦截
123     → 拦截
```

所以这里的 CFG 不是主要用于理解自然语言含义，而是用于：

> 根据 JSON Schema 定义合法输出语言，并在模型生成过程中实时判断哪些 token 仍然可以组成这门语言中的合法字符串。

一句话总结：

> 在句法分析中，CFG 用来判断“一个已有句子如何被解析”；在结构化输出中，CFG 用来限制“模型接下来允许生成什么”。“上下文无关”指产生式只依赖当前非终结符，不依赖它左右的符号，而不是说解码过程不需要查看已经生成的前缀。