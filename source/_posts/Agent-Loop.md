---
title: Agent-Loop
author: Gaesar
date: 2026-10-05 17:56:13
tags: 从0学习Agent
categories: AI-Agent
---

# Agent Loop

## 最简 Agent Loop

> ReAct模型：Reasoning + Acting，思考与行动交替

```
messages = [system 提示词, 用户任务]
for turn in range(MAX_TURNS):                  # 停止条件②：轮数上限
    reply = 调用模型(messages, tools=工具表)
    if reply 没有 tool_calls:                   # 停止条件①：模型认为说完了
        return reply.content                    # 这就是最终答案
    messages.append(reply)                      # 带 tool_calls 的 assistant 消息进历史
    for call in reply.tool_calls:
        result = 执行工具(call)                  # 真正干活的还是你的代码
        messages.append(tool 消息(result))
return "达到轮数上限，任务未完成"                 # 兜底
```

1. ReAct

| ReAct三要素 | 循环中的对应                                       |
| ----------- | -------------------------------------------------- |
| Reason      | 模型根据消息数组决定下一步是调用工具还是直接回答。 |
| Act         | 根据模型返回的工具调用请求，代码中执行工具调用。   |
| Observation | 把工具调用的结果装入消息数组，用于下一轮思考。     |

2.循环结束的条件

- 模型不再请求工具
- 轮数达到上限
- 用户打断

3.ReAct的三种失控行为：失控的共同根源是，模型在按"看起来对"生成下一步，而没有外部机制逼它对齐"实际上对"。所有治理手段（好的报错、验证要求、评审、轮数上限）本质都是往循环里注入现实反馈。

- 同一条命令反复执行，每次都失败，每次都不换思路。 解决办法：错误信息要教模型自救。
- 改动越来越多，引入的新问题越来越大。 解决办法：system prompt 里"不要做没被要求的修改"这类约束。
- 没跑验证就宣布修好了。 解决办法：system prompt 强制"改完必须验证"。

## Plan-and-Execute

> 模型先输出一个分步计划，再逐步执行

```
plan = 调用模型("为这个任务列出分步计划：" + task)
for step in plan:
    result = agent_loop(step)      # 每步内部还是小循环
    if 计划赶不上变化: plan = 重新规划(...)
```

针对重复执行、跑偏问题。每一步有参照物，模型不容易在局部细节里迷路。

适合步骤多、结构清晰的任务（大重构、多文件改动）。

但规划本身可能是错的，执行过程中需要要重新规划。

## Reflection

> 任务完成后让模型检查一遍

```
answer = agent_loop(task)
critique = 调用模型("检查这份工作的问题：" + answer)
if critique 有问题:
    answer = agent_loop(task + "注意以下问题：" + critique)
```

针对虚假完成的问题和低级错误。

但重新检查一遍存在开销。

## 是否添加Plan-and-Execute 和 Reflection

由要完成的任务决定，如果任务在ReAct上失败率高，就添加，任务越长越复杂，规划和反思的收益越明显。

## workflow VS Agent

Workflow是由代码决定下一步，Agent是由模型决定下一步。

workflow可控，Agent不可控。

一般能用workflow解决的就不用Agent，但二者也可以结合使用。

## 实验

1.写了一个打折的函数，如下所示，测试样例是按照严格的 > 号生成的，这里修改成了 >= ，并将三个测试用例写在 >= 的边界上。

> 打折函数

```python
def calculate_money(user_pay: float) -> float:
    if user_pay > 300:
        return 0.6 * user_pay
    if user_pay > 200:
        return 0.8 * user_pay
    if user_pay > 100:
        return 0.9 * user_pay
    return user_pay
```

运行agent_loop，`帮我看看./test_func.py中的代码有什么问题? test_money.py这个文件有几个测试样例过不去。修改这个bug，确保正确修改。` 观察Agent每一步的操作。

- Agent第一步列出了目录下的文件，并阅读了./test_func.py代码。

- Agent第二步阅读了./test_money.py和./agent_loop.py代码。

- Agent 第三步观察测试样例和打折函数，推理了可能的问题，并运行了测试代码。

- Agent第四步确认了问题，修改了文件，再次运行了测试代码。

- Agent第五步返回了最终的结果。

2.运行了多次，发现每次Agent处理的过程都不一样，体现了Agent的不确定性。

````
你> 我看看./test_func.py中的代码有什么问题? test_money.py这个文件有几个测试样例过不去。修改这个bug，确保正确修改。
MiniAgent observation ...
MiniAgent Reasoning > Let me start by exploring the directory and reading the files.
MiniAgent Acting >list_files的执行结果：目录 D:\00agent learn\code\4.3 下共 5 个条目：
[目录] .pytest_cache
[目录] __pycache__
[文件] agent_loop.py
[文件] test_func.py
[文件] test_money.py
MiniAgent Acting >read_file的执行结果：./test_func.py文件的内容为：
def calculate_money(user_pay: float) -> float:
    if user_pay >= 300:
        return 0.6 * user_pay
    if user_pay >= 200:
        return 0.8 * user_pay
    if user_pay >= 100:
        return 0.9 * user_pay
    return user_pay

MiniAgent observation ...
MiniAgent Reasoning > Now let's read test_money.py and agent_loop.py.
MiniAgent Acting >read_file的执行结果：./test_money.py文件的内容为：
from test_func import calculate_money
import pytest


@pytest.mark.parametrize("user_pay, expected", [
    (50, 50),
    (100, 100),
    (100.1, 90.09),
    (150, 135),
    (200, 180),
    (200.1, 160.08),
    (250, 200),
    (300, 240),
    (300.1, 180.06),
    (400, 240),
    (0, 0),
    (-100, -100),
])


def test_calculate_money(user_pay, expected):
    assert abs(calculate_money(user_pay) - expected) < 1e-6


MiniAgent Acting >read_file的执行结果：./agent_loop.py文件的内容为：
from openai import OpenAI
import os
import sys
from pathlib import Path
import json

SYSTEM_PROMPT = "你是 MiniAgent，一个能独立完成编码任务的命令行 Agent。工作方式：先观察（读文件、跑命令），再动手（改文件），" \
                "改完必须验证（重新运行测试或程序），确认无误后用一段话总结你做了什么。不要凭想象修改没读过的文件。"
# 测试系统提示词的影响
# SYSTEM_PROMPT = "使用文言文回答"

MODEL = "Qwen3.8-Flash-Next"
# 改成你对应服务商的base_url
BASE_URL = "https://api.hcnsec.cn/v1"
MAX_AGENT_LOOP = 5
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
            return path + "文件的内容为：\n" + f.read()
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
        return f"输出是：stdout:\n{result.stdout}\n错误是：stderr:\n{result.stderr}"
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

def load_api_key() -> str:
    # 优先读取环境变量
    env_key = os.environ.get("DEEPSEEK_API_KEY")
    if env_key:
        return env_key.strip()

    # 尝试读取.env（优先当前目录，备选上级目录）
    env_paths = [
        Path.cwd() / ".env",
        Path(__file__).resolve().parent / ".env",
        Path(__file__).resolve().parents[1] / ".env",
    ]
    for env_file in env_paths:
        if env_file.exists():
            try:
                lines = env_file.read_text(encoding="utf-8").splitlines()
                for line in lines:
                    line = line.strip()
                    if not line or line.startswith("#"):
                        continue
                    if line.startswith("DEEPSEEK_API_KEY="):
                        return line.split("=", 1)[1].strip()
            except Exception as e:
                print(f"读取 {env_file} 失败: {e}", file=sys.stderr)

    sys.exit("错误：没有找到 DEEPSEEK_API_KEY。请配置环境变量或在.env中填写。")


def chat_stream(client: OpenAI, messages: list):

    """流式对话，返回完整回复文本"""
    reply = ""
    try:
        # `stream` 是一个迭代器，阻塞等待服务端推送 chunk。
        stream = client.chat.completions.create(
            model=MODEL,
            messages=messages,
            stream=True,
            temperature=1.7
        )
        for chunk in stream:
            # print(f"\n[DEBUG CHUNK] {chunk}", file=sys.stderr)
            if not chunk.choices:
                continue
            delta = chunk.choices[0].delta
            if delta and delta.content:
                # end="" 参数确保不换行，flush=True 立即输出到终端
                print(delta.content, end="", flush=True)
                reply += delta.content
        return reply
    except Exception as e:
        print(f"\nAPI调用异常: {e}", file=sys.stderr)
        return ""


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


def main() -> None:
    api_key = load_api_key()
    client = OpenAI(api_key=api_key, base_url=BASE_URL)

    messages = [{"role": "system", "content": SYSTEM_PROMPT}]
    print(f"MiniAgent v0.1（{MODEL}）")
    print("输入 exit 退出，输入 clear 清空对话历史\n")

    while True:
        # 测试大模型记忆能力
        user_input = input("你> ").strip()
        if user_input == "exit":
            break
        if user_input == "clear":
            messages = [{"role": "system", "content": SYSTEM_PROMPT}]
            print("对话已清空\n")
            continue
        if not user_input:
            continue

        messages.append({"role": "user", "content": user_input})

        current_loop = 1
        reply = ""
        while current_loop <= MAX_AGENT_LOOP:
            current_loop = current_loop + 1
            print("MiniAgent observation ...")
            response = client.chat.completions.create(
                model=MODEL,
                messages=messages,
                tools=TOOL_SCHEMAS,
                tool_choice="auto"
            )
            message = response.choices[0].message
            if not message.tool_calls:  # 没想调工具，直接说了答案
                reply = message.content
                break
            # 保存工具调用信息
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
            print("MiniAgent Reasoning > " + response.choices[0].message.reasoning)
            # 执行工具调用
            for call in message.tool_calls:
                result = execute_tool_call(call)
                print("MiniAgent Acting >" + call.function.name + "的执行结果：" + result)
                messages.append({
                    "role": "tool",
                    "tool_call_id": call.id,
                    "content": result
                })
        if reply == "" or current_loop > MAX_AGENT_LOOP:
            response = client.chat.completions.create(
                model=MODEL,
                messages=messages
            )
            reply = response.choices[0].message.content
        print("MiniAgent> " + reply)
        # reply = chat_stream(client, messages)
        # print()  # 换行
        # 追加assistant回复，维持上下文记忆
        messages.append({"role": "assistant", "content": reply})



if __name__ == "__main__":
    main()

MiniAgent observation ...
MiniAgent Reasoning > Let's run the tests to see which fail.

Expected behavior from tests:
- 50 -> 50 (no discount)
- 100 -> 100 (no discount? expected 100, but code gives 0.9*100=90) → so boundary at 100: 100 should return 100, meaning condition should be `> 100` i.e., strictly greater.
- 100.1 -> 90.09 = 0.9*100.1 ✓
- 150 -> 135 = 0.9*150 ✓
- 200 -> 180 = 0.9*200 ✓ hmm interesting: 200 gives 180 which is 0.9*200, not 0.8. So boundary at 200 belongs to 0.9 tier. So condition should be `> 200`.
- 200.1 -> 160.08 = 0.8*200.1 ✓
- 250 -> 200 = 0.8*250 ✓
- 300 -> 240 = 0.8*300 ✓ (boundary belongs to 0.8 tier)
- 300.1 -> 180.06 = 0.6*300.1 ✓
- 400 -> 240 = 0.6*400 ✓
- 0 -> 0 ✓
- -100 -> -100 ✓

So the fix: use strict `>` comparisons instead of `>=`. Boundaries inclusive to lower discount tier.

So: if user_pay > 300: 0.6; elif > 200: 0.8; elif > 100: 0.9; else: user_pay.

Let me run the tests first to confirm.

Also note the parametrize decorator is missing a colon `:` at the end in test_money.py! Look: `@pytest.mark.parametrize("user_pay, expected", [ ... ])` — the content shows `])` without a colon. Wait, the read shows:

```
@pytest.mark.parametrize("user_pay, expected", [
    (50, 50),
    ...
    (-100, -100),
])
```
The last line is `])` — missing colon. Hmm, let me check actual file content carefully with cat -A or read raw. The read output shows `])` then blank line then blank line then `def test_calculate_money`. That's a syntax error → all tests fail with collection error. But user says "有几个测试样例过不去" (a few test cases fail), implying others pass. Let me check the actual file bytes.

Let me run pytest first.

MiniAgent Acting >run_bash的执行结果：输出是：stdout:
    
    def test_calculate_money(user_pay, expected):
>       assert abs(calculate_money(user_pay) - expected) < 1e-6
E       assert 20.0 < 1e-06
E        +  where 20.0 = abs((160.0 - 180))
E        +    where 160.0 = calculate_money(200)

test_money.py:22: AssertionError
________________________ test_calculate_money[300-240] ________________________

user_pay = 300, expected = 240

    @pytest.mark.parametrize("user_pay, expected", [
        (50, 50),
        (100, 100),
        (100.1, 90.09),
        (150, 135),
        (200, 180),
        (200.1, 160.08),
        (250, 200),
        (300, 240),
        (300.1, 180.06),
        (400, 240),
        (0, 0),
        (-100, -100),
    ])
    
    
    def test_calculate_money(user_pay, expected):
>       assert abs(calculate_money(user_pay) - expected) < 1e-6
E       assert 60.0 < 1e-06
E        +  where 60.0 = abs((180.0 - 240))
E        +    where 180.0 = calculate_money(300)

test_money.py:22: AssertionError
=========================== short test summary info ===========================
FAILED test_money.py::test_calculate_money[100-100] - assert 10.0 < 1e-06
FAILED test_money.py::test_calculate_money[200-180] - assert 20.0 < 1e-06
FAILED test_money.py::test_calculate_money[300-240] - assert 60.0 < 1e-06
3 failed, 9 passed in 0.07s

错误是：stderr:

MiniAgent Acting >run_bash的执行结果：输出是：stdout:
'from test_func import calculate_money\nimport pytest\n\n\n@pytest.mark.parametrize("user_pay, expected", [\n    (50, 50),\n    (100, 100),\n    (100.1, 90.09),\n    (150, 135),\n    (200, 180),\n    (200.1, 160.08),\n    (250, 200),\n    (300, 240),\n    (300.1, 180.06),\n    (400, 240),\n    (0, 0),\n    (-100, -100),\n])\n\n\ndef test_calculate_money(user_pay, expected):\n    assert abs(calculate_money(user_pay) - expected) < 1e-6\n\n'

错误是：stderr:

MiniAgent observation ...
MiniAgent Reasoning > The test file is intact (pytest output display mangled it, but the raw content is fine).

Failing tests:
- 100 -> expected 100, got 90
- 200 -> expected 180, got 160
- 300 -> expected 240, got 180

So the boundary semantics: the discount thresholds are "满 X 元" meaning strictly greater than X? Let's check:
- (100.1, 90.09) → 0.9 discount applies when > 100. So `user_pay > 100` for 0.9.
- (100, 100) → no discount at exactly 100. So condition is `> 100`.
- (200, 180) → at exactly 200, result is 180 = 0.9 * 200. So at 200, still 0.9 discount (not 0.8). So the 0.8 tier requires > 200.
- (200.1, 160.08) = 0.8 * 200.1 ✓.
- (300, 240) = 0.8 * 300 = 240. So at exactly 300, 0.8 discount, not 0.6. So 0.6 tier requires > 300.
- (300.1, 180.06) = 0.6*300.1 ✓
- (250, 200) = 0.8*250 ✓
- (400, 240) = 0.6*400 ✓
- (50,50), (0,0), (-100,-100) → no discount.

So the bug: `>=` should be `>` (i.e., 满减条件是"超过"阈值才打折)。

Fix: change >= to >.

Let me edit test_func.py.

MiniAgent Acting >write_file的执行结果：成功写入文件: ./test_func.py
MiniAgent observation ...
MiniAgent Reasoning > Now verify with pytest, plus check floating point precision issues (e.g., 0.9*100.1 = 90.09 within 1e-6? yes). Run tests.

MiniAgent Acting >run_bash的执行结果：输出是：stdout:
============================= test session starts =============================
platform win32 -- Python 3.7.4, pytest-7.4.4, pluggy-1.2.0 -- F:\PYTHON307\python.exe
cachedir: .pytest_cache
rootdir: D:\00agent learn\code\4.3
plugins: anyio-3.6.2, hydra-core-1.3.2, typeguard-4.1.2
collecting ... collected 12 items

test_money.py::test_calculate_money[50-50] PASSED                        [  8%]
test_money.py::test_calculate_money[100-100] PASSED                      [ 16%]
test_money.py::test_calculate_money[100.1-90.09] PASSED                  [ 25%]
test_money.py::test_calculate_money[150-135] PASSED                      [ 33%]
test_money.py::test_calculate_money[200-180] PASSED                      [ 41%]
test_money.py::test_calculate_money[200.1-160.08] PASSED                 [ 50%]
test_money.py::test_calculate_money[250-200] PASSED                      [ 58%]
test_money.py::test_calculate_money[300-240] PASSED                      [ 66%]
test_money.py::test_calculate_money[300.1-180.06] PASSED                 [ 75%]
test_money.py::test_calculate_money[400-240] PASSED                      [ 83%]
test_money.py::test_calculate_money[0-0] PASSED                          [ 91%]
test_money.py::test_calculate_money[-100--100] PASSED                    [100%]

============================= 12 passed in 0.03s ==============================
"=== 重新查看源码 ===" 
def calculate_money(user_pay: float) -> float:
    if user_pay > 300:
        return 0.6 * user_pay
    if user_pay > 200:
        return 0.8 * user_pay
    if user_pay > 100:
        return 0.9 * user_pay
    return user_pay

错误是：stderr:

MiniAgent> 

12 个测试全部通过（之前 3 个失败）。

**Bug 分析与修复总结：**

问题出在 `test_func.py` 的**边界判断运算符**。原代码用的是 `>=`，导致 `user_pay` 恰好等于门槛值（100/200/300）时被错误地套用更高一档折扣：`100` 被打了 9 折（实际返回 90，期望 100）、`200` 被打了 8 折（返回 160，期望 180）、`300` 被打了 6 折（返回 180，期望 240）。

从测试样例可以反推出正确的业务规则是"满 X 元**以上**才享受折扣，等于门槛值时留在原档位"：
- `(100, 100)` ✔ 不打折 ↔ `(100.1, 90.09)` ✔ 9 折 → 九折条件为 `> 100`
- `(200, 180)` = 0.9×200 ✔ 仍是九折 ↔ `(200.1, 160.08)` ✔ 8 折 → 八折条件为 `> 200`
- `(300, 240)` = 0.8×300 ✔ 仍是八折 ↔ `(300.1, 180.06)` ✔ 6 折 → 六折条件为 `> 300`

因此把三处 `>=` 全部改为 `>`（折扣逻辑本身、以及 400→240、0→0、-100→-100 等其余样例均无需改动），改完重新运行 `pytest test_money.py`，结果为 **12 passed**，确认修复无误。
````

## 代码

```python
from openai import OpenAI
import os
import sys
from pathlib import Path
import json

SYSTEM_PROMPT = "你是 MiniAgent，一个能独立完成编码任务的命令行 Agent。工作方式：先观察（读文件、跑命令），再动手（改文件），" \
                "改完必须验证（重新运行测试或程序），确认无误后用一段话总结你做了什么。不要凭想象修改没读过的文件。"
# 测试系统提示词的影响
# SYSTEM_PROMPT = "使用文言文回答"

MODEL = "Qwen3.8-Flash-Next"
# 改成你对应服务商的base_url
BASE_URL = "https://api.hcnsec.cn/v1"
MAX_AGENT_LOOP = 5
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
            return path + "文件的内容为：\n" + f.read()
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
        return f"输出是：stdout:\n{result.stdout}\n错误是：stderr:\n{result.stderr}"
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

def load_api_key() -> str:
    # 优先读取环境变量
    env_key = os.environ.get("DEEPSEEK_API_KEY")
    if env_key:
        return env_key.strip()

    # 尝试读取.env（优先当前目录，备选上级目录）
    env_paths = [
        Path.cwd() / ".env",
        Path(__file__).resolve().parent / ".env",
        Path(__file__).resolve().parents[1] / ".env",
    ]
    for env_file in env_paths:
        if env_file.exists():
            try:
                lines = env_file.read_text(encoding="utf-8").splitlines()
                for line in lines:
                    line = line.strip()
                    if not line or line.startswith("#"):
                        continue
                    if line.startswith("DEEPSEEK_API_KEY="):
                        return line.split("=", 1)[1].strip()
            except Exception as e:
                print(f"读取 {env_file} 失败: {e}", file=sys.stderr)

    sys.exit("错误：没有找到 DEEPSEEK_API_KEY。请配置环境变量或在.env中填写。")


def chat_stream(client: OpenAI, messages: list):

    """流式对话，返回完整回复文本"""
    reply = ""
    try:
        # `stream` 是一个迭代器，阻塞等待服务端推送 chunk。
        stream = client.chat.completions.create(
            model=MODEL,
            messages=messages,
            stream=True,
            temperature=1.7
        )
        for chunk in stream:
            # print(f"\n[DEBUG CHUNK] {chunk}", file=sys.stderr)
            if not chunk.choices:
                continue
            delta = chunk.choices[0].delta
            if delta and delta.content:
                # end="" 参数确保不换行，flush=True 立即输出到终端
                print(delta.content, end="", flush=True)
                reply += delta.content
        return reply
    except Exception as e:
        print(f"\nAPI调用异常: {e}", file=sys.stderr)
        return ""


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


def main() -> None:
    api_key = load_api_key()
    client = OpenAI(api_key=api_key, base_url=BASE_URL)

    messages = [{"role": "system", "content": SYSTEM_PROMPT}]
    print(f"MiniAgent v0.1（{MODEL}）")
    print("输入 exit 退出，输入 clear 清空对话历史\n")

    while True:
        # 测试大模型记忆能力
        user_input = input("你> ").strip()
        if user_input == "exit":
            break
        if user_input == "clear":
            messages = [{"role": "system", "content": SYSTEM_PROMPT}]
            print("对话已清空\n")
            continue
        if not user_input:
            continue

        messages.append({"role": "user", "content": user_input})

        current_loop = 1
        reply = ""
        while current_loop <= MAX_AGENT_LOOP:
            current_loop = current_loop + 1
            print("MiniAgent observation ...")
            response = client.chat.completions.create(
                model=MODEL,
                messages=messages,
                tools=TOOL_SCHEMAS,
                tool_choice="auto"
            )
            message = response.choices[0].message
            if not message.tool_calls:  # 没想调工具，直接说了答案
                reply = message.content
                break
            # 保存工具调用信息
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
            print("MiniAgent Reasoning > " + response.choices[0].message.reasoning)
            # 执行工具调用
            for call in message.tool_calls:
                result = execute_tool_call(call)
                print("MiniAgent Acting >" + call.function.name + "的执行结果：" + result)
                messages.append({
                    "role": "tool",
                    "tool_call_id": call.id,
                    "content": result
                })
        if reply == "" or current_loop > MAX_AGENT_LOOP:
            response = client.chat.completions.create(
                model=MODEL,
                messages=messages
            )
            reply = response.choices[0].message.content
        print("MiniAgent> " + reply)
        # reply = chat_stream(client, messages)
        # print()  # 换行
        # 追加assistant回复，维持上下文记忆
        messages.append({"role": "assistant", "content": reply})



if __name__ == "__main__":
    main()
```



