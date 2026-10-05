---
title: Function Calling
date: 2026-10-05 15:03:18
categories: [AI-Agent]
author: Gaesar
tags: 
  - 从0学习Agent
---

## Function Calling的原理

向大模型发送请求的时候附带上工具的描述信息（包括：函数名、函数的功能描述，参数列表，参数的描述），如果需要调用工具，会返回要调用的函数的名称、参数。实际工具的执行是在代码中执行的。

### Function Calling的过程

```
模型读取messages -> 模型返回tool_call -> 收到回复后在代码中执行工具 -> tool结果写回messages
```

### 发送给大模型的工具描述信息示例

```json
{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询一个城市的天气情况",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"},
            },
            "required": ["city"],
        },
    },
}
```

参数解析：

1. description：要描述精确什么情况使用，避免几个工具描述相似导致模型选错工具。
2. 参数数量要少：参数复杂，出错率高。
3. 约束的信息写入描述中可以省掉大量的失败重试。
4. 中文任务用中文描述。

### 大模型返回的需要调用的工具信息示例

```
{
  "message": {
    "role": "assistant",
    "content": "我来帮你查询北京今天的天气。",
    "reasoning_content": "用户想知道北京今天的天气。我需要使用get_weather工具，传入\"北京\"作为城市名。",
    "tool_calls": [{
      "id": "call_00_YtU5AfVNIzGbaTaBWbCV3295",
      "type": "function",
      "function": {"name": "get_weather", "arguments": "{\"city\": \"北京\"}"}
    }]
  },
  "finish_reason": "tool_calls"
}

```

参数解析：

1. `"finish_reason": "tool_calls"`是代码里判断"模型想干活还是说完了"的依据。
2. `"tool_calls"` 模型一轮可以请求多个工具，每个call包含唯一的id，用于将工具执行结果与执行工具请求配对

### 代码中实际调用工具

> 收到模型返回的工具调用请求后，先保存到消息数组中

```python
    messages.append({
        "role": "assistant",
        "content": message.content,
        "tool_calls": [
            {
                "id": tc.id,
                "type": tc.type,
                "function": {
                    "name": tc.function.name,
                    "arguments": tc.function.arguments
                }
            }
            for tc in message.tool_calls
        ]
    })
```

> 然后在代码中逐个执行工具

```python
    for call in message.tool_calls:
        result = execute_tool_call(call)
        messages.append({
            "role": "tool",
            "tool_call_id": call.id,
            "content": result
        })
```

#### 执行的原理

先从回复中取出函数名和函数的参数，根据函数名从映射字典中取出代码中的函数。

将函数的参数从字符串转为json，并解析为参数列表传给函数。

将执行的结果存储到消息数组中。

最终把整个消息数组传给大模型，大模型根据工具执行结果做出回答。

> 本地实际执行工具调用的核心代码

```python
TOOL_FUNCTIONS = {
    "read_file": read_file,
    "write_file": write_file,
    "edit_file": edit_file,
    "run_bash": run_bash,
    "list_files": list_files,
}


# 工具调用执行核心逻辑
def execute_tool_call(tool_call):
    """
    根据模型返回的tool_call，找到本地函数并执行
    tool_call: model返回的tool_calls里单个元素
    """
    func_name = tool_call.function.name
    func_args = json.loads(tool_call.function.arguments)

    # 路由查找本地函数
    if func_name not in TOOL_FUNCTIONS:
        return f"错误：不存在工具 {func_name}"

    local_func = TOOL_FUNCTIONS[func_name]
    return local_func(**func_args)
```

## 实验

1. 读取同级目录下的代码，并判断出错误。

```
你> 帮我看看./test_func.py中的代码有什么问题?
MiniAgent> 函数名是 `add`（加法），但实现却是 `a - b`（减法）。应该改为 `return a + b`。
```

2. 新增列出当前目录下所有文件的工具。

```
你> 帮我看看当前目录下有哪些文件
MiniAgent>
当前目录下共有2 个文件：
| 文件名 | 类型 |
|--------|------|
| `func_call.py` | Python 文件 |
| `test_func.py` | Python 文件 |
目录路径为 `D:\00agent learn\code\3.3`，没有其他子文件夹。
```

3.打印模型回复的原始结构。

`response`

```
ChatCompletion(
    id='55b6ee84361144e3a4ca21abf16b27db',
    choices=[
        Choice(
            finish_reason='tool_calls',
            index=0,
            logprobs=None,
            message=ChatCompletionMessage(
                content='\n\n',
                refusal=None,
                role='assistant',
                annotations=None,
                audio=None,
                function_call=None,
                tool_calls=[
                    ChatCompletionMessageFunctionToolCall(
                        id='call_5db3e907a7fd43d997a3e856',
                        function=Function(
                            arguments='{"path": "."}',
                            name='list_files'
                        ),
                        type='function',
                        index=0
                    )
                ],
                reasoning='用户想查看当前目录下的文件。我应该使用 list_files 工具,参数 path 默认是当前目录,我可以传 "."。\n'
            ),
            matched_stop=None
        )
    ],
    created=1791189658,
    model='self-dploy/Qwen3.8-Flash-Next',
    object='chat.completion',
    metadata={
        'request_id': '5bktP8KVgBbR9MXNnlMpoxRZ6xQbDr90NPMyOINS',
        'requested_model': 'Qwen3.8-Flash-Next',
        'requested_provider': None,
        'used_model': 'Qwen3.8-Flash-Next',
        'used_provider': 'self-dploy',
        'underlying_used_model': 'Qwen3.8-Flash-Next',
        'routing': [
            {
                'provider': 'self-dploy',
                'model': 'Qwen3.8-Flash-Next',
                'status_code': 200,
                'error_type': 'none',
                'succeeded': True,
                'apiKeyHash': 'ce10dea265c1cb6e0219ebfdc40832cca3d0949c855b2b05409513e79fdc4d91',
                'logId': 'aytExtgMDYOJP0P8fTkh'
            }
        ],
        'log_id': 'aytExtgMDYOJP0P8fTkh',
        'organization_id': 'oneclick-org-59197',
        'project_id': 'oneclick-project-59197',
        'discount': None
    },
    moderation=None,
    service_tier=None,
    system_fingerprint=None,
    usage=CompletionUsage(
        completion_tokens=54,
        prompt_tokens=786,
        total_tokens=840,
        completion_tokens_details=CompletionTokensDetails(
            accepted_prediction_tokens=None,
            audio_tokens=0,
            reasoning_tokens=28,
            rejected_prediction_tokens=None,
            text_tokens=None,
            image_tokens=0
        ),
        prompt_tokens_details=PromptTokensDetails(
            audio_tokens=0,
            cache_write_tokens=0,
            cached_tokens=64,
            image_tokens=0,
            text_tokens=None,
            video_tokens=0
        ),
        reasoning_tokens=28,
        cost=0.00014876,
        cost_details={
            'upstream_inference_cost': 0.00014876,
            'upstream_inference_prompt_cost': 0.00011022,
            'upstream_inference_completions_cost': 3.8539999999999994e-05,
            'total_cost': 0.00014876,
            'input_cost': 0.0001083,
            'output_cost': 3.8539999999999994e-05,
            'cached_input_cost': 1.92e-06,
            'cache_write_input_cost': 0,
            'request_cost': 0,
            'web_search_cost': 0,
            'image_input_cost': None,
            'image_output_cost': None
        }
    )
)

```

`response.choices[0]`

```
Choice(
    finish_reason='tool_calls',
    index=0,
    logprobs=None,
    message=ChatCompletionMessage(
        content='\n\n',
        refusal=None,
        role='assistant',
        annotations=None,
        audio=None,
        function_call=None,
        tool_calls=[
            ChatCompletionMessageFunctionToolCall(
                id='call_5db3e907a7fd43d997a3e856',
                function=Function(
                    arguments='{"path": "."}',
                    name='list_files'
                ),
                type='function',
                index=0
            )
        ],
        reasoning='用户想查看当前目录下的文件。我应该使用 list_files 工具,参数 path 默认是当前目录,我可以传 "."。\n'
    ),
    matched_stop=None
)

```

`response.choices[0].message`

```
ChatCompletionMessage(
    content='\n\n',
    refusal=None,
    role='assistant',
    annotations=None,
    audio=None,
    function_call=None,
    tool_calls=[
        ChatCompletionMessageFunctionToolCall(
            id='call_5db3e907a7fd43d997a3e856',
            function=Function(
                arguments='{"path": "."}',
                name='list_files'
            ),
            type='function',
            index=0
        )
    ],
    reasoning='用户想查看当前目录下的文件。我应该使用 list_files 工具,参数 path 默认是当前目录,我可以传 "."。\n'
)

```

`response.choices[0].message.tool_calls`

```
[
    ChatCompletionMessageFunctionToolCall(
        id='call_5db3e907a7fd43d997a3e856',
        function=Function(
            arguments='{"path": "."}',
            name='list_files'
        ),
        type='function',
        index=0
    )
]
```

4.测试连续任务，连续任务失败，失败原因是只选择了一个工具不足以完成任务。

```
你> 帮我看看./test_func.py中的代码有什么问题? 并修改这个bug，确保正确修改。
MiniAgent>
<tool_call>
<function=write_file>
<parameter=path>
./test_func.py
</parameter>
<parameter=content>
def add(a, b):
    return a + b

</parameter>
</function>
</tool_call>
```

5.测试模糊描述，随意选择工具。

```
你> 操作./test_func.py这个文件。
MiniAgent>
<tool_call>
<function=shell>
<parameter=command>
python ./test_func.py
</parameter>
</function>
</tool_call>
```



## 代码

```python
from openai import OpenAI
import os
import sys
from pathlib import Path
import json

MODEL = "Qwen3.8-Flash-Next"
API_KEY = "sk-594IlRrDlzpPsh8TqrdcAZxx4m9FB0TC20Y1p2ep8iiVvVZ5"
BASE_URL = "https://api.hcnsec.cn/v1"
SYSTEM_PROMPT = "回答要简洁明了，不要太啰嗦。"
messages = [{"role": "system", "content": SYSTEM_PROMPT}]
client = OpenAI(api_key=API_KEY, base_url=BASE_URL)
# 工具Schema（给大模型看的说明书）
TOOL_SCHEMAS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "读取一个文本文件的完整内容",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件路径"},
                },
                "required": ["path"],
                "additionalProperties": False
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "写入内容到文本文件，覆盖原有内容",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件路径"},
                    "content": {"type": "string", "description": "要写入的文本内容"},
                },
                "required": ["path", "content"],
                "additionalProperties": False
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "edit_file",
            "description": "编辑文件，替换指定行区间的内容",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件路径"},
                    "start_line": {"type": "integer", "description": "起始行号，从0开始"},
                    "end_line": {"type": "integer", "description": "结束行号"},
                    "content": {"type": "string", "description": "替换后的文本"},
                },
                "required": ["path", "start_line", "end_line", "content"],
                "additionalProperties": False
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "run_bash",
            "description": "执行bash shell命令，返回标准输出和错误",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {"type": "string", "description": "shell命令字符串"},
                },
                "required": ["command"],
                "additionalProperties": False
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "list_files",
            "description": "列出指定目录下的所有文件和子目录，默认列出当前目录",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "目录路径，不填则默认当前工作目录",
                        "default": "."
                    },
                },
                "required": ["path"],
                "additionalProperties": False
            },
        },
    }
]


# 实际执行的工具函数
def read_file(path: str) -> str:
    """读取文本文件"""
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"read_file error: {str(e)}"


def write_file(path: str, content: str) -> str:
    """写入文本文件"""
    try:
        with open(path, "w", encoding="utf-8") as f:
            f.write(content)
        return f"成功写入文件: {path}"
    except Exception as e:
        return f"write_file error: {str(e)}"


def edit_file(path: str, start_line: int, end_line: int, content: str) -> str:
    """替换文件指定行范围内容"""
    try:
        with open(path, "r", encoding="utf-8") as f:
            lines = f.readlines()
        lines[start_line:end_line] = [content + "\n"]
        with open(path, "w", encoding="utf-8") as f:
            f.writelines(lines)
        return f"已修改 {path}，行范围 {start_line}-{end_line}"
    except Exception as e:
        return f"edit_file error: {str(e)}"


def run_bash(command: str) -> str:
    """执行shell命令"""
    import subprocess
    try:
        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=10)
        return f"stdout:\n{result.stdout}\nstderr:\n{result.stderr}"
    except Exception as e:
        return f"run_bash error: {str(e)}"


def list_files(path: str = ".") -> str:
    """列出目录下所有文件和子目录"""
    try:
        p = Path(path)
        if not p.exists():
            return f"错误：路径 {path} 不存在"
        if not p.is_dir():
            return f"错误：{path} 不是目录"
        entries = []
        for item in sorted(p.iterdir()):
            kind = "目录" if item.is_dir() else "文件"
            entries.append(f"[{kind}] {item.name}")
        return f"目录 {p.resolve()} 下共 {len(entries)} 个条目：\n" + "\n".join(entries)
    except Exception as e:
        return f"list_files error: {str(e)}"


# 工具路由表：函数名 → 本地真实函数
TOOL_FUNCTIONS = {
    "read_file": read_file,
    "write_file": write_file,
    "edit_file": edit_file,
    "run_bash": run_bash,
    "list_files": list_files,
}


# 工具调用执行核心逻辑
def execute_tool_call(tool_call):
    """
    根据模型返回的tool_call，找到本地函数并执行
    tool_call: model返回的tool_calls里单个元素
    """
    func_name = tool_call.function.name
    func_args = json.loads(tool_call.function.arguments)

    # 路由查找本地函数
    if func_name not in TOOL_FUNCTIONS:
        return f"错误：不存在工具 {func_name}"

    local_func = TOOL_FUNCTIONS[func_name]
    return local_func(**func_args)


def main():
    # 用户提问
    user_query = "帮我看看./test_func.py中的代码有什么问题?"
    # user_query = "帮我看看./test_func.py中的代码有什么问题? 并修改这个bug，确保正确修改。"
    # user_query = "操作./test_func.py这个文件。"
    # user_query = "帮我看看当前目录下有哪些文件?"
    print("你> " + user_query)
    messages.append({"role": "user", "content": user_query})

    # 第 1 步：带着工具说明书请求模型
    response = client.chat.completions.create(
        model=MODEL,
        messages=messages,
        tools=TOOL_SCHEMAS,
        tool_choice="auto"
    )

    message = response.choices[0].message
    if not message.tool_calls:  # 没想调工具，直接说了答案
        print("模型直接回答：")
        print(message.content)
        return

    # 打印回复
    # print("response")
    # print(response)
    # print("response.choices[0]")
    # print(response.choices[0])
    # print("response.choices[0].message")
    # print(response.choices[0].message)
    # print("response.choices[0].message.tool_calls")
    # print(response.choices[0].message.tool_calls)


    # 第 2 步：保存工具调用信息
    messages.append({
        "role": "assistant",
        "content": message.content,
        "tool_calls": [
            {
                "id": tc.id,
                "type": tc.type,
                "function": {
                    "name": tc.function.name,
                    "arguments": tc.function.arguments
                }
            }
            for tc in message.tool_calls
        ]
    })

    # 第 3 步：执行工具调用
    for call in message.tool_calls:
        result = execute_tool_call(call)
        messages.append({
            "role": "tool",
            "tool_call_id": call.id,
            "content": result
        })

    # 第 4 步：把工具结果发回去，拿最终回答（只执行一轮，无循环）
    final_resp = client.chat.completions.create(
        model=MODEL,
        messages=messages
    )
    print("MiniAgent>")
    print(final_resp.choices[0].message.content)


if __name__ == '__main__':
    main()

```

