---
title: "LangChain 入门：它是什么、底层在做什么、由哪些部分组成、和 LangGraph 什么关系"
date: 2026-10-28
tags: ["LangChain", "LangGraph", "LLM", "AI 学习笔记"]
author: "Q"
categories: ["学习笔记"]
series: "LangChain 学习笔记"
level: "入门"
prior: "我目前只知道 LangChain 是大模型相关的架构，可以直接调用其他函数，很方便；但我不知道它的底层逻辑，也不知道它具体包含哪些内容，更不清楚 LangChain 和 LangGraph 有什么区别。"
---

## 1. 我原来的理解

> 我目前只知道 LangChain 是大模型相关的架构，可以直接调用其他函数，很方便；但我不知道它的底层逻辑，也不知道它具体包含哪些内容，更不清楚 LangChain 和 LangGraph 有什么区别。

## 2. 它到底解决什么问题

先说清楚一件事：不用框架我照样能调通模型。`requests.post(url, json=...)` 就够了。所以框架的价值不在"能调通"，在调通之后。

有四件事一出现，手写就开始漏：

**换模型。** 每家的入参、返回结构、流式分片格式、tool_call 字段都不一样。先写了 OpenAI 再换 DeepSeek，调用点、解析、重试全得改一遍。

**多轮记忆。** `history: list[dict]` 自己 append、自己裁剪、自己按用户隔离。并发一进来就全乱。

**让模型调我自己的 Python 函数。** 真正的工作量是：写 JSON schema 告诉模型有哪些函数、解析它返回的 arguments、执行、把结果包成新消息塞回去、再调一次模型。这段跟业务逻辑毫无关系，错了却是线上炸。

**多个 prompt 串流水线。** 手写函数嵌套，每加一级改上下游签名。

框架在这四件事上省的不是行数，是**类型契约**：每段的输入输出类型固定，管道搭不起来在拼的时候就 `TypeError`，类型对不上要等第一次调用才暴露。手写 HTTP 时同样的错会以 500、JSON 解析失败、字段名拼错的形式，跑到线上才炸。

## 3. 核心概念地图

五个概念，一句话说清谁包谁、数据怎么流：

- **Runnable**——是协议，不是工具类。只有继承它的组件（或交给 `RunnableLambda` 包的普通函数、`dict`）才能进管道；自己写个只有 `invoke` 方法的类不算数。
- **`|`**——就是 `Runnable.__or__`，拼出来的 `RunnableSequence` 自己也还是 Runnable，所以整条链白拿四种调用方式。
- **Prompt 模板**——唯一"填变量"的地方，dict → `ChatPromptValue`（里面裹着 SystemMessage / HumanMessage）。
- **模型对象**——消息 → `AIMessage`。换模型 = 换这个对象。
- **`create_agent`**——包着"模型 ↔ 工具"的循环和状态，产物是 LangGraph 图（我实测类型是 `CompiledStateGraph`）。

数据流是一条直线，四个类型：

```text
dict → ChatPromptValue → AIMessage → str
      (prompt 段)       (model 段)   (parser 段)
```

看懂这条类型链，就懂了所谓"底层逻辑"——框架没做什么魔法，它只是在每段之间强制了这四个类型。

## 4. 最小可跑示例

```python
# minimal_lcel.py —— 离线可跑，不需要任何 API key
import importlib.metadata as md                                                 # 用来看真实版本号
from langchain_core.prompts import ChatPromptTemplate                            # 提示词模板：把 dict 填进消息列表
from langchain_core.language_models.fake_chat_models import FakeListChatModel   # 官方假模型：按顺序吐固定回复
from langchain_core.output_parsers import StrOutputParser                        # 输出解析器：AIMessage -> str

print(md.version("langchain"), "|", md.version("langchain-core"))                # 实测 1.4.7 | 1.7.0

prompt = ChatPromptTemplate.from_messages([                                      # 两步模板：system 定角色，human 留变量位
    ("system", "你是一个只回答一句话的助手。"),
    ("human", "{question}"),
])
model = FakeListChatModel(responses=[                                            # 假模型替代真模型，每调一次消费一条
    "LangChain 把模型、提示词、工具和流程统一成可组合的组件。",
    "统一的模型调用与流程编排层。",
])
chain = prompt | model | StrOutputParser()                                       # `|` = Runnable.__or__，拼出 RunnableSequence

print(type(chain).__name__)                                                      # 整条链自己的类型
print("[invoke]", chain.invoke({"question": "LangChain 是什么？"}))              # 单个 dict 进 -> str 出
print("[stream]", [c for c in chain.stream({"question": "再问一次"})][:5], "...")  # 生成器，逐块吐
```

我真跑的输出：

```text
1.4.7 | 1.7.0
RunnableSequence
[invoke] LangChain 把模型、提示词、工具和流程统一成可组合的组件。
[stream] ['统', '一', '的', '模', '型'] ...
```

逐段说框架替你做了什么：

`prompt` 段把 dict 填进两条消息模板——这是全链唯一拼字符串的地方，少传变量就在这一步 `KeyError`。`model` 段拿到上一步的 `ChatPromptValue`，转成某家 provider 的 HTTP 请求体，回一条 `AIMessage` 而不是字符串，这一点最容易踩（`chain.invoke` 去掉 parser 就是 AIMessage）。`StrOutputParser` 只做一件事：取 `.content`。`|` 把三段串成 `RunnableSequence`，所以 `stream` 不需要额外写代码——它顺着三段自动传下去，假模型逐字吐，于是拿到字符列表。

换成真模型只改 `model = ...` 这一行（上面再加一行 import）：

```python
from langchain.chat_models import init_chat_model
model = init_chat_model("openai:gpt-5.5")  # 需 pip install langchain-openai + 环境变量 OPENAI_API_KEY
```

其余一个字不用改，因为下游只依赖 `AIMessage` 这一个类型——这就是"标准模型接口"的全部价值。注意它在**构造时**就校验凭据，没 key 直接抛 `OpenAIError: Missing credentials`，别指望"先建对象、晚点再调"。

## 5. 用我熟悉的东西类比

**`Runnable` 的统一调用协议 ↔ sklearn estimator 统一成 `fit`/`predict`。**
作用一样：不管底下是哪个实现，外面写法固定。
失效处：sklearn 没有 `stream` / `ainvoke` 这一档——你只能一次拿到完整结果，流式在 estimator 里没有对应概念。

**`|` ↔ `torch.nn.Sequential` / `sklearn.Pipeline`。**
作用一样：串起来就是一条流水线，前一环喂后一环。
失效处：Sequential 只能串张量，Pipeline 的中间每步必须是 transformer（只有末环可以是 predictor）；`|` 的两端可以是模板、模型、`@tool`、任意 Python 函数，中间流的是消息对象和 dict，不是张量。

**checkpointer + thread_id ↔ 训练断点续跑。**
作用一样：进程死了能从上次的位置接着跑。
失效处：训练 checkpoint 存权重，恢复出来还是同一个模型；这里存的是消息列表状态，而且按 `thread_id` 隔离成多份——更像多个训练任务各自一份 checkpoint，不是全局单点。

## 6. 三个常见误区

第一条就是我开头那句话。

**误区一：「可以直接调用其他函数，很方便」。**
错在：v1 的 `langchain` 不是函数库，是被刻意收缩的命名空间（agents / messages / tools / chat_models / embeddings）。你要"调"的不是某个函数，而是先拼出一段 Runnable 再 `invoke` 它。找"输入文本输出文本的函数"这个心理模型本身就错位。
依据：实测 `from langchain.chains import LLMChain` → `ModuleNotFoundError: No module named 'langchain.chains'`；`langchain.agents` 里跟 agent 相关的入口只剩 `create_agent`（其余几个名字是 `AgentState`、`middleware`、`structured_output`、`factory`）。

**误区二：装了 `langchain` 就能用 `ChatOpenAI`。**
错在：框架和 provider 适配彻底拆开了，`langchain` 不带任何 provider SDK。
依据：实测 `langchain 1.4.7` 的依赖只有 `langchain-core`、`langgraph`、`pydantic`。用哪家装哪家，`pip install langchain langchain-openai`。

**误区三：照 2023 教程写 `ConversationBufferMemory` + `AgentExecutor`。**
错在：记忆从"一个可插拔的类"变成"checkpointer 持久化的图状态 + thread_id"；agent 换成 `create_agent`，参数是 `system_prompt=` 不是 `prompt=`。
依据：实测 `langchain.memory` 模块不存在、`langchain.agents` 没有 `AgentExecutor` 属性。

## 7. 我卡住的点 & 怎么想通的

**卡点一：为什么不直接 `requests` 发 JSON？**
想通：单轮、单 provider、无状态时确实该直接用。但一旦要"模型调我的函数"，手写的就是 schema + 解析 + 回填 + 再调一遍模型的循环，这段正确性跟业务无关却最容易错。框架把它压成了 `@tool` 加一行 `tools=[...]`。

**卡点二：LangChain 和 LangGraph 是不是"高级版 / 低级版"？**
想通：不是版本关系，是层级关系。LangGraph 是运行时（持久执行、中断、人工审批），LangChain 建在它上面。硬证据：`create_agent(...)` 返回的对象类型是 `<langgraph.graph.state.CompiledStateGraph>`——LangChain 的 agent 本身就是一个编译好的 LangGraph 图。也可以只用 LangGraph 不要 LangChain。

**卡点三（真踩到的）：我在管道里插了个 `print` 探针看类型，结果 `stream` 不逐字吐了。**
实测：干净链条 stream 出 14 块（示例那句 14 个字，一字一块）；在模型后面接一个 `RunnableLambda(lambda x: x)`，只剩 1 块。
想通：流式由**最弱的一环**决定。普通函数只能一次性返回，它必须把上游的 14 块聚成 1 个输入才能执行，下游就只看到 1 块。加探针别加在模型后面。

## 8. 现在我能做什么

- 三个月前我说不出 `|` 是什么，现在能说：它是 `Runnable.__or__`，拼出 `RunnableSequence`，而后者自己也是 Runnable——所以我跑 `type(chain).__name__` 拿到 `RunnableSequence`，且整条链照样能 `.stream()`。
- 三个月前我以为"没 key 也能先建对象再调"，现在能验证：`init_chat_model("openai:gpt-5.5")` 在构造阶段就抛 `OpenAIError: Missing credentials`。
- 三个月前我不知道"多轮记忆"存在哪，现在能写出 `create_agent(..., checkpointer=InMemorySaver())` + `{"configurable": {"thread_id": "t1"}}`，并指出 `langchain.memory` 已整个消失。

## 9. 下一步学什么

1. 用 `from langchain.tools import tool` 把示例里的假模型换掉，先把一个自己的小函数（比如查天气的 `get_weather`）写成真工具，再配真模型跑一次 agent 循环，亲眼看 `AIMessage.tool_calls` → `ToolMessage` → `AIMessage` 的往返。需要 `OPENAI_API_KEY`，或一个兼容端点：`ChatOpenAI(model=..., base_url=..., api_key=...)`。
2. 手写一张 `StateGraph`，两个节点（model / tools）配一条条件边，跑通"有 tool_calls 就回 tools 节点、没有就结束"，把 `create_agent` 内部那张图拆开看一遍。
3. 读 `docs.langchain.com/oss/python/langchain/short-term-memory`，把 `InMemorySaver` 换成落盘的 `SqliteSaver`（另装 `langgraph-checkpoint-sqlite`），验证进程重启后同一 `thread_id` 的历史还在。
