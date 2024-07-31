# 前言

目前，许多开源项目都实现了Agent，如Langchain、LlamaIndex等，使用开源框架的确能够方便我们快速开发出Agent，但代价是开发者对执行流程的不可控。

开源框架的Prompt较为固定不易修改，且实现的代码也是与其Prompt高度适配的，但不同LLM的能力相差较大，完全依赖开源框架无法都取得好的效果。

因此，我们可以使用Langchain、LlamaIndex等框架，甚至Dify workflow来快速验证想法，而在生产环境下再自行实现Agent，这样也方便我们根据具体业务进行Prompt以及流程的调整。

# ReAct

实现Agent的理论基础是ReAct框架。ReAct框架出自论文[《ReAct: Synergizing Reasoning and Acting in Language Models》](https://arxiv.org/abs/2210.03629)，它要求 LLM以交错的方式生成“推理轨迹”和“任务特定操作”，允许 LLM与外部工具交互来获取额外信息，从而给出更可靠和实际的回应。

论文中给出了ReAct的思路，但并没有给出具体的实现方法，本文参照论文思路，给出实现ReAct的Prompt以及流程。

# ReAct Prompt

Prompt是实现ReAct的核心，以下Prompt参考了LlamaIndex框架。

````
## Background
{{ .Background }}

You are designed to help with a variety of tasks, from answering questions to providing summaries to other types of analyses.

## Tools

You have access to a wide variety of tools. You are responsible for using the tools in any sequence you deem appropriate to complete the task at hand.
This may require breaking the task into subtasks and using different tools to complete each subtask.

You have access to the following tools:
{{ .ToolDesc }}

## Output Format

Please answer in the same language as the question and use the following format:

```
Thought: The current language of the user is: (user's language). I need to use a tool to help me answer the question.
Action: tool name (one of {{ .ToolNames }}) if using a tool.
Action Input: the input to the tool, in a JSON format representing the kwargs ( e.g. {"input": "hello world", "num_beams": 5} )
```

Please ALWAYS start with a Thought.

Please use a valid JSON format for the Action Input.

Then, you need wait for user's respond. If this format is used, the user will respond in the following format.

```
Observation: tool response
```

Otherwise, user will remind you to correct your input in the following format.

```
Error: remind you to correct your input
```

You should keep repeating the above format (JUST include "Thought", "Action" and "Action Input") till you have enough information to answer the question without using any more tools. At that point, you MUST respond in the one of the following two formats:

```
Thought: I can answer without using any more tools. I'll use the user's language to answer
Answer: [your answer here (In the same language as the user's question)]
```

```
Thought: I cannot answer the question with the provided tools.
Answer: [your answer here (In the same language as the user's question)]
```

## Current Conversation

Below is the current conversation consisting of interleaving human and assistant messages.
````

该Prompt首先允许开发者自定义“背景信息”，提供“背景信息”有利于LLM做出更好的选择。然后，Prompt定义了可使用的工具，由开发者自定义，最后，具体描述了工作流程，该流程与论文中的相似。

根据该Prompt，期望的消息形式如下：

``` xml
<Systemt>
...省略
</Systemt>

<User>
Question: 北京的天气怎么样？
</User>

<Assistant>
Thought: I need to use WeatherTool to help me answer the question.
Action: WeatherTool
Action Input: {"position": "beijing"}
</Assistant>

<User>
Observation: 小雨，天空阴沉。
</User>

<Assistant>
Thought: I can answer without using any more tools. I'll use the user's language to answer
Answer: 今天北京的天气是小雨....
</Assistant>
```

示例展示了LLM的回答格式，我们需要在代码中实现相应的数据解析方法。

# 实现流程

## LLM响应内容获取

LLM被期望的回答格式仅有两种，即

``` yaml
Thought: I need to use WeatherTool to help me answer the question.
Action: WeatherTool
Action Input: {"position": "beijing"}
```

和

``` yaml
Thought: I can answer without using any more tools. I'll use the user's language to answer
Answer: 今天北京的天气是小雨....
```

我们需要按照格式提取相关信息，可借用正则表达式实现信息提取

- 对于第一种的正则表达式如下：

```
^\s*Thought: (.*?)\n+Action: ([a-zA-Z0-9_]+).*?\n+Action Input: .*?(\{.*\})
```

这段正则表达式用于从一段文本中提取三个部分的信息：

1. 从开头匹配“Thought: ”后面的内容，直到遇到一个或多个换行符。
2. 匹配“Action: ”后面的Action名称（由字母、数字和下划线组成），直到遇到一个或多个换行符。
3. 匹配“Action Input: ”后面花括号包围的内容。

- 第二种对应的正则表达式为：

```
\s*Thought:(.*?)\n+Answer:.*?(.*)
```

这段正则表达式用于从一段文本中提取两个部分的信息：

1. 匹配“Thought:”后面的内容，直到遇到一个或多个换行符。
2. 匹配“Answer:”后面的内容，直到字符串结束。

## 工作逻辑实现

一旦我们提取LLM的响应信息，我们就可以根据Prompt描述的工作流程实现相应的代码。

将LLM的Action Input交给对应的工具进行处理，并将Observation返回给LLM。当然，LLM可能返回的是Answer。

代码实现即判断输出中是否包含流程特征，如文字中包含Action或Answer。

``` go
switch{
case strings.Contain(output,"Action:"):
    // ... 处理函数调用
case strings.Contain(output,"Answer:"):
    // ... 处理最终回答
default:
    // ... 出现错误，给LLM反馈信息
}
```

# 常见错误及处理

大部分厂商都对它们的LLM进行了FunctionCall能力的调整，在大部分情况下都能正确地遵循指令，但并非所有模型都按照理预期进行回答。根据我个人的实践经验，我总结了以下几种错误。

## 错误1：调用不存在的工具

模型有时可能出现幻觉，会调用了不存在的工具，一般的处理的方法有

1. 返回错误，中止流程；
2. 选用默认工具来处理；

此外，还有一种选择是给LLM反馈信息，告知LLM该工具不存在且只能选择特定的工具等，接着要求LLM重新思考。LLM收到反馈后往往能回至原轨。

如何给模型反馈呢？大家可以注意到在Prompt定义了一种工作流，即

```
Error: remind you to correct your input
```

那么当发生错误时，我们直接向模型输出错误反馈，此时对话将变为以下形式：

``` xml
<Systemt>
...省略
</Systemt>

<User>
北京的天气怎么样
</User>

<Assistant>
Thought: I need to use WeatherTool to help me answer the question.
Action: PositionTool
Action Input: {"position": "beijing"}
</Assistant>

<User>
Error: 工具不存在，请选择指定的工具类型，......
</User>

<Assistant>
Thought: Something wrong happened, I need to use WeatherTool to help me answer the question.
Action: WeatherTool
Action Input: {"position": "beijing"}
</Assistant>
```

## 错误2：调用工具的入参错误

这种错误情况的处理方法与错误1相同，可以给LLM反馈以便LLM修正错误。

## 错误3：未真正调用工具便得出最终回答

从名称上看较为抽象，我这里给出这种错误的举例：

``` xml
<User>
北京的天气怎么样
</User>

<Assistant>
Thought: I need to use WeatherTool to help me answer the question.
Action: WeatherTool
Action Input: {"position": "beijing"}
Observation: 北京是晴天
Thought: xxxxx
Answer: 北京天晴
</Assistant>
```

这种错误情况会直接造成LLM没有进行任何工具调用便给出了最终答案，因为中间工具调用过程的结果均是LLM产生的幻觉。

这种错误可以通过正则表达式与代码的判断逻辑解决。当LLM响应之后，我们先判断响应中夫否存在Action，假如存在则只提取第一个Action以及Action Input，然后自行构造新的消息体，以此屏蔽掉LLM的幻觉。

## 错误4：Action Input的参数错误

一般情况下，LLM的错误往往是大小写、大小驼峰等格式不正确，而并非胡乱输入。

对于这种情况，我们如果能在代码中进行判断，就最好不要给模型以反馈，毕竟每一次反馈都有时间成本。

可以将ActionInput反序列化至map中，然后便利map的键值对，自行修正输入。如：

``` go
var m map[string]interface{}
json.Unmarshal(data,&m)
```

# 优秀实践

- 设计Agent流程时要考虑LLM的常见错误并进行兜底；
- 给LLM反馈错误让其自行修正；
- 优先在代码中修正错误，降低调用的时间成本。

# 总结

本文介绍了如何使用Go语言实现基于ReAct框架的Agent。ReAct框架通过交错生成“推理轨迹”和“任务特定操作”，允许LLM与外部工具交互，给出更可靠的回应。此外，本文还提供了优秀实践，包括设计Agent流程时考虑LLM常见错误并进行兜底、给LLM反馈错误让其自行修正、优先在代码中修正错误等。