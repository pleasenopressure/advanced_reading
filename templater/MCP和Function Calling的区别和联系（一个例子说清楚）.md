Function Calling 和 MCP 经常一起出现，但它们解决的不是同一个层次的问题。

## 一句话区别

> **Function Calling 解决“模型如何选择工具并生成参数”；MCP 解决“工具如何被标准化发现、描述、调用和返回结果”。**

---

## 1. Function Calling 是什么

Function Calling，也叫 Tool Calling，是模型 API 提供的一种能力。

你先告诉模型有哪些工具：

```json
{
  "name": "query_order",
  "description": "查询订单状态",
  "parameters": {
    "type": "object",
    "properties": {
      "orderId": {
        "type": "string"
      }
    },
    "required": ["orderId"]
  }
}
```

用户说：

```text
帮我查一下订单 O202600000001
```

模型可能输出：

```json
{
  "name": "query_order",
  "arguments": {
    "orderId": "O202600000001"
  }
}
```

注意，模型没有真正查询订单，它只是表达：

```text
我认为应该调用 query_order，
参数是 orderId=O202600000001。
```

之后由你的应用程序：

1. 校验参数；
    
2. 检查权限；
    
3. 调用数据库或 HTTP API；
    
4. 把工具结果返回给模型；
    
5. 让模型生成最终回复。
    

因此 Function Calling 的核心是：

```text
自然语言
   ↓
工具名称 + 结构化参数
```

---

## 2. MCP 是什么

MCP，全称 Model Context Protocol，是一种工具和上下文接入协议。

它定义了：

- 工具如何被服务端暴露；
    
- 客户端如何发现工具；
    
- 工具的名称和参数 Schema 如何描述；
    
- 工具调用请求如何发送；
    
- 工具结果如何返回；
    
- 资源、提示模板等上下文能力如何提供。
    

MCP 中通常有三个角色：

```text
MCP Host
   └── MCP Client
           └── MCP Server
```

### MCP Host

运行 AI 应用的宿主，例如：

- IDE；
    
- AI 桌面应用；
    
- Agent Runtime；
    
- 你自己的智能助手。
    

### MCP Client

Host 内部负责连接 MCP Server 的组件。

### MCP Server

真正暴露工具和资源的一方，例如：

```text
订单 MCP Server
数据库 MCP Server
GitHub MCP Server
文件系统 MCP Server
```

MCP 通常使用标准化协议消息，让不同 AI 应用可以用相似方式接入工具。

---

# 3. 两者怎样配合

一个支持 MCP 的 Agent 可以按下面的流程工作：

```text
① MCP Client 连接 MCP Server
          ↓
② 通过 MCP 发现可用工具
          ↓
③ 将 MCP 工具定义转换成模型的 Function Calling 格式
          ↓
④ 模型选择工具并生成参数
          ↓
⑤ Agent Runtime 收到工具调用意图
          ↓
⑥ 转换成 MCP tools/call 请求
          ↓
⑦ MCP Server 执行真实操作
          ↓
⑧ 工具结果返回给模型
          ↓
⑨ 模型生成最终回答
```

例如 MCP Server 暴露工具：

```text
query_order(orderId)
```

用户说：

```text
帮我查一下订单 O202600000001。
```

模型通过 Function Calling 生成：

```json
{
  "name": "query_order",
  "arguments": {
    "orderId": "O202600000001"
  }
}
```

Agent Runtime 再通过 MCP 调用对应工具。

所以：

```text
Function Calling
负责模型“做决定”

MCP
负责应用“连接并调用工具”
```

---

# 4. 主要区别

|对比维度|Function Calling|MCP|
|---|---|---|
|所在层次|模型能力层|应用协议层|
|核心问题|模型选择什么工具、生成什么参数|工具如何发现、连接、调用和返回|
|是否由模型产生|是，模型输出调用意图|不是，MCP 规定应用之间如何通信|
|是否执行真实工具|通常不执行|MCP Client 会请求 MCP Server 执行|
|是否跨模型统一|不完全，各供应商格式不同|目标之一是标准化工具接入|
|工具来自哪里|通常由开发者直接传给模型|可以通过 MCP Server 动态发现|
|典型输出|工具名和 arguments|`tools/list`、`tools/call` 等协议消息|
|是否能单独使用|可以|可以不绑定某一种模型工具格式|

---

# 5. 没有 MCP，也可以使用 Function Calling

例如你的 Java 后端直接注册工具：

```text
query_order
send_email
get_weather
```

模型生成调用后，你在代码中手动分发：

```java
switch (toolName) {
    case "query_order" -> orderService.query(...);
    case "send_email" -> emailService.send(...);
}
```

这就是：

```text
Function Calling + 自己实现的工具执行层
```

完全不需要 MCP。

问题是，当工具越来越多，并且需要被多个 AI 客户端复用时，你可能要重复实现：

- 工具注册；
    
- Schema 描述；
    
- 连接方式；
    
- 鉴权；
    
- 调用协议；
    
- 结果格式。
    

MCP 就是为了让这一层更加标准化。

---

# 6. 没有 Function Calling，也可以使用 MCP

理论上，应用也可以直接调用 MCP 工具，不让模型选择。

例如用户点击一个固定按钮：

```text
“查询当前 GitHub Issues”
```

程序直接发送 MCP 工具请求：

```text
MCP Client → MCP Server
```

此时没有经过模型的 Function Calling。

所以 MCP 并不依赖 Function Calling；只是在 Agent 场景中，两者非常适合组合。

---

# 7. 和 HTTP API 的关系

MCP Server 内部最终可能调用普通 HTTP API：

```text
模型
 ↓ Function Calling
Agent Runtime
 ↓ MCP tools/call
MCP Server
 ↓ HTTP 请求
订单系统 API
```

例如：

```text
MCP 工具：query_order
```

内部实际执行：

```http
GET /api/orders/O202600000001
```

因此可以理解为：

```text
Function Calling：模型意图层
MCP：工具接入协议层
HTTP API：具体业务能力层
```

---

# 8. 一个直观类比

把模型看成一个经理。

### Function Calling

经理说：

> “应该让订单部门查询订单 O202600000001。”

它负责判断：

- 找哪个部门；
    
- 提供什么参数。
    

### MCP

相当于公司统一的内部通信制度：

- 如何查看有哪些部门；
    
- 如何给部门下发任务；
    
- 任务参数采用什么格式；
    
- 部门如何返回结果。
    

### HTTP API

相当于订单部门内部真正办理业务的工作系统。

因此：

> Function Calling 是“模型发出工具调用意图”，MCP 是“工具接入和通信的标准协议”。MCP 可以把工具暴露给多个 Agent，而 Function Calling 可以帮助模型从这些工具中选择合适的一个并生成参数。